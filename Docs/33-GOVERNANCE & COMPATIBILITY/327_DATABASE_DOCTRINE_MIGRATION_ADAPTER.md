# 327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md

## 1. Propósito

Este documento define la arquitectura del adaptador oficial de migración desde **Doctrine ORM y Doctrine DBAL** hacia **VoltStack/Quantum/Database**.

El componente se denomina conceptualmente:

```text
DoctrineMigrationAdapter
```

y forma parte del sistema definido en:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
```

Su responsabilidad es comprender una aplicación basada en Doctrine, extraer su modelo de persistencia, normalizar entidades, metadata, repositories, tipos, consultas y semánticas del `EntityManager` hacia el modelo intermedio de migración de VoltStack.

El adaptador no pretende ejecutar Doctrine indefinidamente dentro de VoltStack ni convertir `Quantum/Database` en una capa de compatibilidad con Doctrine.

---

## 2. Objetivo principal

Permitir una transición controlada desde:

```text
Application
    │
    ▼
Doctrine ORM / DBAL
    │
    ▼
Database
```

hacia:

```text
Application
    │
    ▼
VoltStack Database
    │
    ▼
Database
```

preservando, cuando sea técnicamente posible:

```text
same schema
same data
same identifiers
same relationships
same transaction semantics
same persistence behavior
```

---

## 3. Principio arquitectónico

El flujo será:

```text
Doctrine Source
      │
      ▼
Discovery
      │
      ▼
Metadata Extraction
      │
      ▼
Static Analysis
      │
      ├── Optional Runtime Analysis
      ▼
Doctrine Semantic Model
      │
      ▼
Normalization
      │
      ▼
Intermediate Database Model
      │
      ▼
VoltStack Migration Pipeline
```

---

## 4. Regla de aislamiento

La dependencia permitida será:

```text
DoctrineMigrationAdapter
        │
        ▼
Migration Core
        │
        ▼
VoltStack Database
```

Nunca:

```text
VoltStack Database
        │
        ▼
Doctrine ORM / DBAL
```

Por tanto, el adaptador deberá poder eliminarse completamente después de la migración.

---

## 5. Distribución

Se recomienda un paquete separado:

```text
VoltStack/Database-Migration-Doctrine
```

instalado preferentemente como dependencia de desarrollo.

Conceptualmente:

```bash
composer require --dev voltstack/database-migration-doctrine
```

Al finalizar:

```bash
composer remove voltstack/database-migration-doctrine
```

---

## 6. Alcance

El adapter deberá analizar:

```text
Entities
Mapped Superclasses
Embeddables
Entity Metadata
Attributes
Legacy Annotations
XML Mapping
YAML Mapping
Repositories
EntityManager
UnitOfWork
Identity Map
Proxy / Lazy Loading
Associations
Join Tables
Cascade Operations
Orphan Removal
Lifecycle Callbacks
Entity Listeners
Event Subscribers
Custom Types
Value Generators
Inheritance Mapping
Discriminator Maps
DQL
QueryBuilder
Native SQL
DBAL
Transactions
Locking
Schema Metadata
Migrations
Fixtures
Filters
Naming Strategies
Second-Level Cache
```

---

## 7. Doctrine ORM y DBAL

El sistema distinguirá:

```text
Doctrine ORM
    │
    └── entity persistence

Doctrine DBAL
    │
    └── database abstraction
```

Una aplicación puede utilizar:

```text
ORM only
DBAL only
ORM + DBAL
```

El plan de migración deberá reflejarlo.

---

## 8. Versiones soportadas

El adapter deberá declarar explícitamente las versiones de:

```text
doctrine/orm
doctrine/dbal
doctrine/persistence
doctrine/migrations
```

que hayan sido verificadas.

No deberá asumir compatibilidad automática entre major versions.

---

## 9. Doctrine Version Profile

Conceptualmente:

```php
final class DoctrineVersionProfile
{
    public function ormVersion(): ?string;

    public function dbalVersion(): ?string;

    public function capabilities(): array;

    public function migrationRules(): array;
}
```

---

## 10. Version Detector

El detector analizará:

```text
composer.json
composer.lock
installed packages
Doctrine configuration
```

y seleccionará el perfil adecuado.

---

## 11. Discovery Pipeline

```text
composer metadata
      │
      ▼
Doctrine Version Detector
      │
      ▼
Configuration Scanner
      │
      ▼
Metadata Source Detector
      │
      ▼
Entity / Repository Scanner
      │
      ▼
Query / DBAL Scanner
      │
      ▼
Doctrine Source Model
```

---

## 12. Static-first

Al igual que el adapter Eloquent, Doctrine utilizará un enfoque:

```text
STATIC FIRST
RUNTIME WHEN NECESSARY
```

El análisis inicial no deberá requerir:

```text
production connection
application boot
entity hydration
query execution
```

---

## 13. Metadata Sources

Doctrine puede definir mappings mediante:

```text
PHP Attributes
Annotations
XML
YAML
programmatic metadata
```

El adapter deberá detectar qué mecanismo utiliza cada entidad.

---

## 14. Metadata Normalization

Independientemente de la fuente:

```text
Attributes ─┐
Annotations ├──► Doctrine Metadata Descriptor
XML ────────┤
YAML ───────┘
```

Posteriormente:

```text
Doctrine Metadata Descriptor
          │
          ▼
Intermediate Database Model
```

---

## 15. Entity Discovery

Ejemplo:

```php
#[Entity]
#[Table(name: 'users')]
final class User
{
}
```

deberá producir:

```text
Entity:
User

Table:
users
```

---

## 16. Entity Descriptor

Cada entidad se representará inicialmente mediante:

```text
DoctrineEntityDescriptor
```

con:

```text
class
table
schema
fields
identifiers
associations
embedded objects
inheritance
repository
callbacks
listeners
cache
change tracking
```

---

## 17. Mapped Superclasses

El adapter deberá detectar:

```php
#[MappedSuperclass]
abstract class BaseEntity
{
}
```

y resolver campos, identifiers y callbacks heredados.

---

## 18. Embeddables

Ejemplo:

```php
#[Embeddable]
final class Address
{
}
```

deberá normalizarse como:

```text
Embedded Value Object
```

manteniendo:

```text
column prefix
field mappings
nullability
nested embedding
```

---

## 19. Entity vs Value Object

El migrador deberá distinguir:

```text
Entity
→ identity

Embeddable
→ value semantics
```

y evitar convertir automáticamente un embeddable en entidad independiente.

---

## 20. Table Mapping

El adapter deberá preservar:

```text
table name
schema
catalog where relevant
indexes
unique constraints
options
```

Los nombres inferidos deberán congelarse explícitamente.

---

## 21. Naming Freeze

Si Doctrine utiliza una `NamingStrategy`:

```text
ClassName
      │
      ▼
Naming Strategy
      │
      ▼
physical_table
```

el resultado físico deberá registrarse explícitamente en la metadata VoltStack.

---

## 22. Custom Naming Strategies

Una naming strategy personalizada deberá analizarse.

Si no puede reproducirse de forma segura:

```text
resolve actual physical names
+
freeze mapping
```

será preferible a portar la estrategia completa.

---

## 23. Fields

Cada field deberá conservar:

```text
property
column
Doctrine type
database type
length
precision
scale
nullable
unique
options
generated behavior
```

---

## 24. Identifiers

El adapter deberá analizar:

```text
single ID
composite ID
assigned ID
generated ID
foreign ID
```

Doctrine posee soporte más amplio para composite identifiers que Eloquent, por lo que esta área requiere especial atención.

---

## 25. Generated Values

Se deberán detectar estrategias:

```text
AUTO
IDENTITY
SEQUENCE
NONE
CUSTOM
UUID-like custom generators
```

y convertirlas hacia la estrategia de generación de VoltStack.

---

## 26. Sequence Generators

Para plataformas que utilicen sequences deberán preservarse:

```text
sequence name
allocation size
initial value
```

cuando sean relevantes.

---

## 27. Custom ID Generators

Un generador personalizado deberá clasificarse:

```text
TRANSFORMABLE
ADAPTABLE
MANUAL
```

según pueda reproducirse su semántica.

---

## 28. Associations

El adapter deberá comprender:

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
```

además de:

```text
owning side
inverse side
mappedBy
inversedBy
join columns
join tables
cascade
fetch
orphanRemoval
orderBy
indexBy
```

---

## 29. Owning Side

Doctrine utiliza explícitamente el concepto de lado propietario.

Esta información no deberá perderse.

Ejemplo:

```text
User
   │ inverse
   ▼
Posts
   ▲
   │ owning
Post.user
```

---

## 30. ManyToOne

Ejemplo:

```php
#[ManyToOne(targetEntity: User::class)]
#[JoinColumn(name: 'user_id')]
private User $user;
```

normalización:

```text
MANY_TO_ONE
source: Post
target: User
foreign_key: user_id
```

---

## 31. OneToMany

Ejemplo:

```php
#[OneToMany(
    mappedBy: 'user',
    targetEntity: Post::class
)]
private Collection $posts;
```

deberá vincularse con el owning side correspondiente.

---

## 32. OneToOne

El adapter deberá conservar:

```text
owning side
join column
unique constraint
cascade
orphan removal
```

---

## 33. ManyToMany

Deberá extraer:

```text
join table
join columns
inverse join columns
owning side
inverse side
cascade
```

---

## 34. Association Entity

Si una join table contiene información de dominio adicional, el migrador podrá recomendar modelarla como entidad asociativa.

No deberá convertirla automáticamente si la semántica no es evidente.

---

## 35. Cascade

Doctrine puede configurar:

```text
persist
remove
detach
refresh
all
```

La migración deberá mapear explícitamente cada cascade.

---

## 36. Cascade Risk

Una diferencia en cascade puede provocar:

```text
missing writes
unexpected deletes
detached entities
data loss
```

Por ello, los cascades forman parte de la verificación crítica.

---

## 37. Orphan Removal

```text
orphanRemoval=true
```

no equivale simplemente a cascade delete.

La semántica deberá preservarse y probarse.

---

## 38. Fetch Strategy

Doctrine puede utilizar:

```text
LAZY
EAGER
EXTRA_LAZY
```

La migración deberá registrar esta intención.

---

## 39. EXTRA_LAZY

`EXTRA_LAZY` puede cambiar el comportamiento de:

```text
count
contains
slice
```

sin cargar toda la colección.

El equivalente VoltStack deberá preservar las características relevantes o generar una advertencia.

---

## 40. Collections

Doctrine utiliza colecciones para asociaciones.

El adapter deberá distinguir:

```text
collection semantics
domain collection behavior
Doctrine Collection API dependencies
```

Código de dominio acoplado a `Doctrine\Common\Collections\Collection` deberá aparecer en el reporte.

---

## 41. Collection API Leakage

Ejemplo:

```php
public function posts(): Collection
```

puede exponer Doctrine al dominio.

El migrador deberá identificar estos contratos para decidir si:

```text
preserve temporarily
adapt
replace with domain collection
```

---

## 42. Criteria API

Uso de:

```text
Doctrine Criteria
```

deberá transformarse hacia:

```text
Intermediate Query Model
```

cuando sea posible.

---

## 43. EntityManager

El adapter deberá localizar dependencias sobre:

```text
EntityManagerInterface
EntityManager
```

y operaciones:

```text
persist
remove
flush
clear
detach
refresh
merge where applicable
contains
find
getReference
```

---

## 44. Persist Semantics

Doctrine:

```php
$em->persist($entity);
```

no significa necesariamente SQL inmediato.

La migración deberá preservar la semántica:

```text
schedule persistence
      │
      ▼
Unit of Work
      │
      ▼
flush
```

---

## 45. Flush Semantics

`flush()` es una frontera crítica.

El analyzer deberá descubrir dónde se ejecuta:

```text
controllers
services
repositories
commands
jobs
listeners
```

y preservar los límites de persistencia.

---

## 46. Partial Flush / Specialized Patterns

Cualquier patrón dependiente de flush parcial, orden específico o extensiones deberá marcarse para revisión según la versión Doctrine.

---

## 47. Unit of Work

El adapter deberá comprender:

```text
NEW
MANAGED
DETACHED
REMOVED
```

y conceptos como:

```text
change sets
scheduled insertions
scheduled updates
scheduled deletions
association changes
```

---

## 48. Unit of Work Mapping

La migración será:

```text
Doctrine UnitOfWork
       │
       ▼
Semantic Persistence Operations
       │
       ▼
VoltStack UnitOfWork
```

No se reutilizarán clases internas Doctrine.

---

## 49. Identity Map

Doctrine garantiza identidad dentro del contexto gestionado.

Conceptualmente:

```php
$a = $em->find(User::class, 1);
$b = $em->find(User::class, 1);
```

puede devolver la misma instancia gestionada.

La migración deberá verificar que el Persistence Context de VoltStack preserve las garantías definidas para V1.

---

## 50. Entity States

El adapter deberá identificar código que dependa de:

```text
managed
detached
removed
```

especialmente cuando utilice APIs internas o avanzadas.

---

## 51. Change Tracking

Doctrine soporta estrategias de tracking.

El migrador deberá detectar configuraciones como:

```text
DEFERRED_IMPLICIT
DEFERRED_EXPLICIT
NOTIFY where applicable historically
```

y mapear la intención hacia el mecanismo de dirty tracking de VoltStack.

---

## 52. Dirty Checking

La verificación deberá comprobar:

```text
scalar changes
embedded changes
association changes
collection changes
```

para evitar actualizaciones perdidas.

---

## 53. Proxies

Doctrine puede utilizar proxies para lazy loading.

El adapter deberá localizar dependencias explícitas o implícitas sobre proxies.

VoltStack no deberá imitar la implementación de proxies si posee otra estrategia de lazy loading.

---

## 54. Proxy Leakage

Código como:

```text
instanceof Doctrine Proxy
proxy initialization checks
```

deberá marcarse como acoplamiento de infraestructura.

---

## 55. Lazy Loading

El runtime analyzer podrá registrar:

```text
proxy initialization
lazy collection initialization
query count
source location
```

para detectar diferencias durante la migración.

---

## 56. Repositories

El adapter deberá descubrir:

```text
EntityRepository
ServiceEntityRepository
custom repository classes
repository factory behavior
```

---

## 57. Repository Descriptor

Cada repository podrá describirse como:

```text
entity
base class
custom methods
DQL usage
QueryBuilder usage
native SQL
dependencies
```

---

## 58. Repository Migration

Flujo:

```text
Doctrine Repository
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

---

## 59. Repository Methods

Los métodos que contienen únicamente lógica de dominio independiente de Doctrine podrán preservarse casi directamente.

Los métodos acoplados a:

```text
EntityManager
QueryBuilder
DQL
Doctrine Result
```

deberán transformarse.

---

## 60. DQL

El analyzer deberá reconocer:

```text
SELECT
JOIN
WHERE
GROUP BY
HAVING
ORDER BY
subqueries where supported
constructor expressions
partial objects
```

y convertir consultas analizables al Intermediate Query Model.

---

## 61. DQL Parser

Arquitectura conceptual:

```text
DQL
 │
 ▼
DQL Parser
 │
 ▼
Doctrine Query Descriptor
 │
 ▼
Intermediate Query AST
 │
 ▼
VoltStack Query AST
```

---

## 62. QueryBuilder

Ejemplo:

```php
$qb->select('u')
   ->from(User::class, 'u')
   ->where('u.active = :active')
   ->setParameter('active', true);
```

deberá normalizarse semánticamente y no por reemplazo de métodos.

---

## 63. Expressions

El adapter deberá reconocer:

```text
andX / orX patterns
comparisons
IN
LIKE
IS NULL
EXISTS
functions
parameters
```

según la versión de Doctrine.

---

## 64. Parameters

Se deberán preservar:

```text
named parameters
positional parameters
explicit parameter types
array parameters
```

---

## 65. Custom DQL Functions

Funciones DQL personalizadas deberán inventariarse.

Ejemplo:

```text
DATE_DIFF
JSON_EXTRACT
custom spatial function
```

Clasificación:

```text
DIRECT
PLATFORM_SPECIFIC
CUSTOM_EXTENSION
MANUAL
```

---

## 66. Native SQL

Doctrine permite SQL nativo.

Cuando sea más seguro:

```text
preserve native SQL
```

en lugar de intentar transformarlo.

Siempre deberán preservarse bindings y tipos.

---

## 67. ResultSetMapping

Uso de `ResultSetMapping` u otras estrategias avanzadas de native hydration deberá marcarse para análisis especial.

---

## 68. Hydration Modes

El adapter deberá detectar dependencias sobre:

```text
object hydration
array hydration
scalar hydration
single scalar
custom hydrators
```

La migración deberá conservar la forma contractual del resultado cuando sea relevante.

---

## 69. Partial Objects

Las consultas que produzcan entidades parcialmente hidratadas requieren especial cuidado.

No deberán transformarse automáticamente a entidades completas si cambia el comportamiento o coste.

---

## 70. DBAL Connection

Para aplicaciones DBAL:

```text
Doctrine\DBAL\Connection
```

se analizarán:

```text
executeQuery
executeStatement
fetch*
insert
update
delete
transactions
platform access
```

---

## 71. DBAL QueryBuilder

El DBAL QueryBuilder se mapeará directamente hacia el modelo intermedio de consultas cuando sea posible.

---

## 72. DBAL Types

Los tipos Doctrine deberán normalizarse:

```text
string
integer
smallint
bigint
boolean
decimal
float
date
datetime
datetime_immutable
json
binary
blob
guid
custom types
```

antes de seleccionar tipos VoltStack.

---

## 73. Custom Types

Los custom Doctrine Types son un área crítica.

El analyzer deberá determinar:

```text
PHP representation
SQL declaration
database representation
conversion to database
conversion to PHP
binding type
platform dependencies
```

---

## 74. Type Migration Adapter

Cuando sea viable podrá generarse:

```text
Doctrine Custom Type
       │
       ▼
Temporary Type Adapter
       │
       ▼
VoltStack Database Type
```

hasta implementar el tipo nativo.

---

## 75. Platform-specific Types

Tipos dependientes de:

```text
PostgreSQL
MySQL
SQL Server
SQLite
Oracle where applicable
```

deberán preservar su naturaleza específica.

La migración de ORM no implica migración de motor.

---

## 76. Enum Types

Se deberán distinguir:

```text
PHP Enum
Doctrine custom enum mapping
database native enum
string/integer representation
```

---

## 77. Schema Manager

Uso directo del Schema Manager de DBAL deberá detectarse.

El código que introspecciona dinámicamente la base puede necesitar migración hacia las APIs de schema de VoltStack.

---

## 78. Doctrine Migrations

El adapter deberá detectar:

```text
doctrine/migrations
migration configuration
migration classes
version table
executed migrations
```

---

## 79. Migration History Strategy

Estrategias:

```text
KEEP_HISTORY
BASELINE_CURRENT_SCHEMA
CONVERT_SELECTED
FULL_CONVERSION
```

Para sistemas maduros se recomienda normalmente:

```text
KEEP HISTORY
+
VOLTSTACK BASELINE
```

---

## 80. Migration Metadata Table

La tabla utilizada por Doctrine Migrations para registrar versiones no deberá modificarse accidentalmente.

El baseline de VoltStack deberá utilizar su propia metadata salvo estrategia explícita de importación.

---

## 81. Fixtures

Doctrine Fixtures podrán analizarse como una capa independiente.

Dependencias sobre:

```text
ObjectManager
EntityManager
references
fixture ordering
```

deberán mapearse.

---

## 82. Lifecycle Callbacks

El adapter deberá detectar:

```text
PrePersist
PostPersist
PreUpdate
PostUpdate
PreRemove
PostRemove
PostLoad
PreFlush
PostFlush
OnFlush
```

según el mecanismo y versión aplicables.

---

## 83. Lifecycle Timing

No basta con mapear nombres.

Debe preservarse:

```text
before UnitOfWork computation
after change-set computation
before SQL
after SQL
before flush completion
after flush
after hydration
```

cuando corresponda.

---

## 84. Entity Listeners

Los listeners específicos de entidades deberán inventariarse con:

```text
target entity
event
service dependency
method
side effects
```

---

## 85. Event Subscribers

Los subscribers globales deberán analizarse porque pueden modificar múltiples entidades.

---

## 86. onFlush

Código dependiente de `onFlush` y del UnitOfWork interno deberá considerarse de alto riesgo.

Puede inspeccionar o modificar:

```text
scheduled insertions
updates
deletions
collection changes
change sets
```

y probablemente requerirá una migración especializada.

---

## 87. postFlush

Los efectos posteriores al flush deberán analizarse para determinar si requieren:

```text
after persistence
after transaction
domain event
integration event
```

No se asumirá equivalencia automática.

---

## 88. Filters

Doctrine Filters deberán detectarse.

Ejemplos:

```text
tenant filter
soft delete filter
visibility filter
security filter
```

Son conceptualmente similares a global constraints y deben tratarse como comportamiento crítico.

---

## 89. Filter Parameters

Los filtros pueden recibir parámetros runtime.

La migración deberá identificar:

```text
parameter source
lifetime
request isolation
tenant isolation
```

---

## 90. Inheritance Mapping

El adapter deberá reconocer:

```text
SINGLE_TABLE
JOINED
MappedSuperclass
discriminator maps
```

y cualquier estrategia soportada por la versión analizada.

---

## 91. Single Table Inheritance

Se deberán preservar:

```text
single physical table
discriminator column
discriminator values
subclass fields
```

---

## 92. Joined Inheritance

Se deberán preservar:

```text
base table
subclass tables
primary key joins
polymorphic queries
```

Esta estrategia requerirá verificación exhaustiva.

---

## 93. Discriminator Map

Los valores persistidos:

```text
user
admin
customer
```

no deberán cambiar accidentalmente.

Renombrar clases no deberá implicar cambiar discriminadores almacenados.

---

## 94. Inheritance Risk

Si VoltStack V1 no puede garantizar una estrategia equivalente:

```text
MANUAL
```

será preferible a una transformación incompleta.

---

## 95. Optimistic Locking

El adapter deberá detectar:

```text
#[Version]
version columns
optimistic lock APIs
```

y preservar:

```text
version increment
conflict detection
exception semantics
```

---

## 96. Pessimistic Locking

Se analizarán:

```text
PESSIMISTIC_READ
PESSIMISTIC_WRITE
```

y sus requisitos transaccionales.

---

## 97. Lock Mode Mapping

```text
Doctrine LockMode
       │
       ▼
Intermediate Lock Intent
       │
       ▼
VoltStack Lock API
```

---

## 98. Transactions

El adapter deberá analizar:

```text
Connection::beginTransaction
commit
rollBack
transactional/wrapInTransaction patterns
EntityManager transaction APIs
savepoints
```

---

## 99. Flush vs Transaction

Doctrine `flush()` y database `commit()` son conceptos diferentes.

El migrador deberá conservar esta distinción:

```text
flush
→ synchronize UnitOfWork

commit
→ finalize database transaction
```

---

## 100. Nested Transactions

Se deberá comprobar:

```text
savepoint support
nesting behavior
rollback semantics
```

por plataforma.

---

## 101. EntityManager Clear

`clear()` afecta el Persistence Context.

Código dependiente de esta operación deberá mapearse hacia la capacidad equivalente de VoltStack.

---

## 102. Detach

`detach()` o comportamientos equivalentes deberán identificarse si la aplicación depende de entidades no gestionadas.

---

## 103. Refresh

`refresh()` deberá preservar la intención:

```text
discard in-memory state
reload from database
```

---

## 104. getReference

Uso de referencias/proxies sin cargar la entidad deberá mapearse a un mecanismo equivalente o a una referencia de identidad explícita.

---

## 105. Batch Processing

Doctrine suele recomendar patrones:

```text
persist
flush every N
clear
```

El migrador deberá detectar estos loops para preservar consumo de memoria.

---

## 106. Bulk DQL

DQL `UPDATE` y `DELETE` masivos pueden evitar lifecycle y sincronización normal del UnitOfWork.

La migración deberá conservar esa semántica.

---

## 107. Second-Level Cache

Si está habilitado:

```text
entity cache
collection cache
query cache
regions
```

deberá inventariarse.

No se deberá activar automáticamente un cache equivalente sin validar coherencia.

---

## 108. Cache Isolation

Durante Dual ORM Mode:

```text
doctrine.*
voltstack.*
```

deberán permanecer aislados salvo integración explícitamente verificada.

---

## 109. Metadata Cache

El cache de metadata no representa necesariamente datos de negocio, pero su configuración puede afectar startup/performance y deberá documentarse.

---

## 110. Query Cache

Se deberá diferenciar:

```text
query compilation cache
result cache
entity cache
```

porque no son equivalentes.

---

## 111. Symfony Integration

Aunque Doctrine suele utilizarse con Symfony, el adapter no deberá depender de Symfony.

Podrá detectar integraciones como:

```text
DoctrineBundle
service repositories
configuration
container aliases
```

mediante plugins o scanners especializados.

---

## 112. DoctrineBundle

Si está presente, el analyzer podrá usar su configuración como fuente de información, pero el core del adapter deberá seguir siendo independiente.

---

## 113. ServiceEntityRepository

Los repositories basados en Symfony/Doctrine Bundle deberán resolverse hasta:

```text
managed entity
manager registry dependency
custom methods
```

---

## 114. ManagerRegistry

Dependencias sobre:

```text
ManagerRegistry
```

deberán analizarse para conocer:

```text
multiple entity managers
multiple connections
dynamic manager selection
```

---

## 115. Multiple Entity Managers

Una aplicación puede utilizar:

```text
EntityManager A
EntityManager B
```

El migrador deberá preservar sus fronteras y mappings.

---

## 116. Multiple Connections

También deberá conservar:

```text
connection name
platform
credentials source
transaction boundary
```

sin mezclar contextos.

---

## 117. Sharding / Legacy Advanced Features

Cualquier extensión Doctrine externa para sharding u otras capacidades avanzadas deberá clasificarse como extensión específica y no asumirse soportada automáticamente por Database V1.

---

## 118. Custom Persisters

Dependencias sobre persisters internos o personalizados se considerarán de alto riesgo por acoplamiento a internals.

---

## 119. Internal API Usage

El scanner deberá identificar imports o llamadas hacia namespaces/clases internas de Doctrine.

Clasificación:

```text
PUBLIC API
DEPRECATED API
INTERNAL API
UNKNOWN
```

---

## 120. Deprecations

Las APIs Doctrine ya deprecadas deberán marcarse especialmente.

No tiene sentido migrar primero hacia una API legacy intermedia.

El destino deberá ser la API nativa actual de VoltStack.

---

## 121. Error Mapping

Se deberán localizar catches de excepciones como:

```text
ORMException families
DBALException families
UniqueConstraintViolationException
OptimisticLockException
NoResultException
NonUniqueResultException
```

según versión.

---

## 122. Exception Semantics

La migración deberá preservar la intención:

```text
not found
non unique
constraint violation
deadlock
connection failure
optimistic conflict
```

aunque las clases concretas cambien.

---

## 123. Query Result Semantics

Se analizarán APIs como:

```text
getResult
getArrayResult
getScalarResult
getSingleResult
getOneOrNullResult
getSingleScalarResult
iterate/toIterable
```

según versión.

---

## 124. Cardinality Contracts

`getSingleResult()` no equivale simplemente a obtener el primer elemento.

Debe preservarse su comportamiento ante:

```text
0 rows
1 row
>1 rows
```

---

## 125. Streaming

`toIterable()` y patrones similares deberán mapearse a streaming/lazy iteration sin cargar el dataset completo.

---

## 126. Hydration

La migración deberá separar:

```text
query execution
result hydration
entity management
```

para mapear cada fase correctamente hacia VoltStack.

---

## 127. Custom Hydrators

Hydrators personalizados deberán analizarse individualmente.

Pueden contener:

```text
DTO mapping
aggregation
normalization
specialized object construction
```

---

## 128. Constructor Result / DTO Queries

Consultas DQL que construyan DTOs deberán conservar su contrato de resultado sin convertirlos necesariamente en entidades.

---

## 129. Read-only Entities

Configuraciones read-only deberán preservarse cuando sean relevantes para tracking y rendimiento.

---

## 130. Immutable Domain Objects

El adapter deberá identificar patrones de entidades/embeddables inmutables y evitar imponer mutabilidad innecesaria.

---

## 131. Reflection-based Hydration

Si la aplicación depende de comportamientos específicos de Doctrine al hidratar propiedades privadas sin setters, el migrador deberá verificar el modelo de hydration de VoltStack.

---

## 132. Constructors

Doctrine puede hidratar entidades sin utilizar el constructor como una creación normal de dominio.

La migración deberá comprobar diferencias de lifecycle y construcción.

---

## 133. PostLoad

`PostLoad` puede ser crítico cuando la entidad requiere inicialización posterior a hydration.

El equivalente deberá ejecutarse en el momento correcto.

---

## 134. Serialization of Entities

Entidades Doctrine/proxies serializadas en:

```text
sessions
queues
cache
messages
```

deberán detectarse como riesgo.

Se recomienda migrar hacia:

```text
IDs
DTOs
Value Objects
```

---

## 135. Jobs y Workers

Una migración gradual puede dejar mensajes pendientes que contienen estructuras Doctrine.

El deployment plan deberá considerar compatibilidad temporal.

---

## 136. Runtime State

Con FrankenPHP, RoadRunner u OpenSwoole se deberá auditar:

```text
EntityManager lifetime
UnitOfWork lifetime
Identity Map reset
transaction reset
filters
listeners
tenant state
```

---

## 137. EntityManager Reset

Una aplicación Symfony tradicional puede depender de mecanismos de reset del container.

VoltStack deberá garantizar un reset explícito de su Persistence Context por request/job según su runtime.

---

## 138. Memory Leaks

El adapter/runtime analyzer deberá ayudar a detectar:

```text
entities retained across requests
unbounded UnitOfWork
collections retained
listeners holding references
```

---

## 139. Schema Introspection

El adapter combinará metadata Doctrine con introspección real:

```text
Doctrine Metadata
      +
Doctrine Migration History
      +
Actual Database Schema
```

---

## 140. Schema Conflict

Ejemplo:

```text
Doctrine metadata:
length 255

Migration:
length 191

Production:
length 128
```

Resultado:

```text
SCHEMA_CONFLICT
```

No se deberá elegir automáticamente una fuente.

---

## 141. Metadata Validation

Antes de transformar, se comprobará:

```text
entity mappings valid
association targets resolvable
join columns valid
identifier mappings complete
inheritance consistent
```

---

## 142. Doctrine SchemaTool

Si una aplicación utiliza SchemaTool, el adapter podrá usar sus conceptos como evidencia, pero no deberá ejecutar operaciones destructivas durante análisis.

---

## 143. No Production Mutation

El adapter no modificará directamente una base de producción.

Cualquier cambio físico deberá pasar por el sistema de migrations de VoltStack.

---

## 144. Migration Rule IDs

Las reglas utilizarán IDs:

```text
VSDB-MIG-DOC-0001
VSDB-MIG-DOC-0002
...
```

Ejemplos:

```text
VSDB-MIG-DOC-0010
Entity attribute mapping

VSDB-MIG-DOC-0020
ManyToOne mapping

VSDB-MIG-DOC-0030
Embeddable mapping

VSDB-MIG-DOC-0040
Custom type migration

VSDB-MIG-DOC-0050
Lifecycle callback migration
```

---

## 145. Clasificación

Cada elemento se marcará:

```text
DIRECT
TRANSFORMABLE
ADAPTABLE
MANUAL
UNSUPPORTED
RISKY
```

---

## 146. Ejemplo de entidad

Origen:

```php
#[Entity(repositoryClass: UserRepository::class)]
#[Table(name: 'users')]
final class User
{
    #[Id]
    #[GeneratedValue]
    #[Column(type: 'integer')]
    private int $id;

    #[Column(length: 255)]
    private string $email;
}
```

Resultado conceptual:

```text
Entity:
User

Table:
users

Identifier:
id
type: integer
generated: true

Field:
email
type: string
length: 255

Repository:
UserRepository
```

---

## 147. Ejemplo de relación

Origen:

```php
#[ManyToOne(targetEntity: User::class)]
#[JoinColumn(
    name: 'user_id',
    referencedColumnName: 'id',
    nullable: false
)]
private User $user;
```

Resultado:

```text
Relationship:
MANY_TO_ONE

Source:
Post

Target:
User

Join:
user_id → users.id

Nullable:
false
```

---

## 148. Ejemplo de Embeddable

Origen:

```php
#[Embedded(class: Address::class)]
private Address $address;
```

Resultado:

```text
Embedded Value Object:
Address

Owner:
User

Column mapping:
resolved from metadata
```

---

## 149. CLI

El adapter se integrará con:

```bash
php volt database:migrate-orm \
    --from=doctrine \
    --analyze
```

---

## 150. Entity Scope

```bash
php volt database:migrate-orm \
    --from=doctrine \
    --entity=App\\Entity\\User \
    --analyze
```

---

## 151. DBAL-only Mode

```bash
php volt database:migrate-orm \
    --from=doctrine-dbal \
    --analyze
```

podrá analizar aplicaciones sin ORM.

---

## 152. Dry Run

```bash
php volt database:migrate-orm \
    --from=doctrine \
    --dry-run
```

no deberá modificar código ni schema.

---

## 153. Diagnostics

Ejemplo:

```text
[VSDB-MIG-DOC-0087]

Entity:
App\Entity\Order

Feature:
Custom Doctrine Type

Type:
money

Problem:
Database and PHP conversion semantics cannot
be inferred safely.

Action:
Register a VoltStack migration type mapping
or implement a custom type adapter.
```

---

## 154. Configuración del adapter

Conceptualmente:

```php
return [
    'doctrine' => [
        'entity_paths' => [
            'src/Entity',
        ],

        'mapping_sources' => [
            'attributes',
        ],

        'runtime_analysis' => false,

        'freeze_physical_names' => true,
    ],
];
```

---

## 155. Overrides

El usuario podrá proporcionar información explícita para:

```text
custom types
discriminator maps
naming mappings
connections
entity managers
unsupported extensions
```

sin modificar el source original durante análisis.

---

## 156. Custom Rule Extensions

Los proyectos podrán registrar:

```php
final class MoneyTypeMigrationRule
    implements DoctrineMigrationRuleInterface
{
}
```

---

## 157. Doctrine Extensions

El scanner deberá detectar paquetes que modifican Doctrine.

Ejemplos conceptuales:

```text
behavior extensions
soft delete extensions
translatable extensions
tree extensions
spatial extensions
custom types
```

No se asumirá que una extensión desconocida es compatible.

---

## 158. Extension Registry

```text
Doctrine Adapter
      │
      ▼
Extension Registry
      │
      ├── official migration plugin
      ├── third-party plugin
      └── project-specific plugin
```

---

## 159. Dual ORM Mode

Durante transición podrá existir:

```text
Module A → Doctrine
Module B → VoltStack
```

pero deberá definirse ownership claro de entidades.

---

## 160. Shared Database

El escenario típico será:

```text
Doctrine ──┐
           ├── Same Physical Database
VoltStack ─┘
```

sin duplicar datos.

---

## 161. Same Entity Risk

Gestionar el mismo registro simultáneamente mediante dos Identity Maps puede causar:

```text
stale state
lost updates
double flush
cache inconsistency
conflicting lifecycle events
```

por lo que deberá evitarse salvo coordinación explícita.

---

## 162. Transaction Interop

Si Doctrine y VoltStack participan en la misma operación, el sistema deberá verificar si pueden compartir:

```text
physical connection
transaction
isolation level
savepoints
```

La atomicidad nunca deberá asumirse.

---

## 163. Shadow Reads

Podrá ejecutarse:

```text
Doctrine Query ───► Result A
                       │
VoltStack Query ──► Result B
                       │
                       ▼
                  Comparator
```

para validar comportamiento.

---

## 164. Result Normalization

Antes de comparar se normalizarán diferencias de representación como:

```text
Doctrine Collections
VoltStack Collections
proxy classes
date object implementations
```

sin ocultar diferencias de negocio.

---

## 165. Identity Verification

Para entidades gestionadas deberá verificarse no solo contenido, sino:

```text
identity
association consistency
change tracking
```

cuando sea relevante.

---

## 166. Characterization Tests

Antes de migrar componentes complejos podrán generarse tests para:

```text
UnitOfWork
flush boundaries
associations
cascade
orphan removal
lifecycle
custom types
queries
locking
```

---

## 167. Performance Baseline

Se compararán:

```text
query count
SQL latency
hydration time
UnitOfWork cost
memory
lazy loads
flush duration
```

---

## 168. Security

El adapter deberá sanitizar:

```text
DSNs
passwords
query parameters
PII
encrypted values
production snapshots
```

---

## 169. Application Scanner

No bastará con analizar entidades.

También deberán buscarse dependencias Doctrine en:

```text
Controllers
Services
Commands
Jobs
Listeners
Subscribers
Domain
Repositories
Tests
Modules
Packages
```

---

## 170. Doctrine Leakage

El reporte podrá mostrar:

```text
Doctrine API Leakage

Domain              14
Services            43
Controllers          7
Jobs                 9
Tests              184
```

Esto ayuda a medir el acoplamiento real.

---

## 171. Type Hint Leakage

Ejemplos:

```php
EntityManagerInterface
ManagerRegistry
Doctrine Collection
QueryBuilder
Connection
```

fuera de infraestructura deberán inventariarse.

---

## 172. Migration Gateway

Para desacoplar progresivamente:

```text
Application
    │
    ▼
Persistence Contract
    │
    ├── Doctrine Adapter
    └── VoltStack Adapter
```

puede utilizarse temporalmente.

---

## 173. Dependency Graph

El sistema podrá construir un grafo:

```text
Entities
Repositories
Custom Types
Listeners
Subscribers
Queries
Services
```

para determinar orden de migración.

---

## 174. Strongly Connected Components

Entidades con asociaciones circulares podrán agruparse para evitar migraciones parciales inseguras.

---

## 175. Suggested Migration Order

Una estrategia típica:

```text
1. DBAL connections
2. Simple entities
3. Types
4. Repositories
5. Associations
6. UnitOfWork-dependent services
7. Lifecycle listeners
8. Inheritance
9. Advanced extensions
10. Remove Doctrine
```

El orden real dependerá del análisis.

---

## 176. Doctrine vs VoltStack

La migración no será:

```text
Doctrine EntityManager
      =
VoltStack EntityManager
```

aunque existan conceptos similares.

Será:

```text
Doctrine Semantics
       │
       ▼
Intermediate Model
       │
       ▼
VoltStack Native Semantics
```

---

## 177. Ventaja del modelo intermedio

Esto evita que decisiones internas de Doctrine definan permanentemente el diseño de VoltStack.

Por ejemplo:

```text
Doctrine Proxy
```

puede mapearse a:

```text
Lazy Loading Intent
```

sin exigir que VoltStack implemente proxies de la misma manera.

---

## 178. UnitOfWork Mapping Principle

Lo que se conserva es:

```text
persistence semantics
```

no:

```text
Doctrine UnitOfWork implementation
```

---

## 179. Identity Map Mapping Principle

Se conserva:

```text
entity identity guarantee
```

no:

```text
Doctrine IdentityMap internals
```

---

## 180. Metadata Mapping Principle

Se conserva:

```text
entity/database mapping
```

no:

```text
Doctrine attribute/annotation classes
```

---

## 181. Query Mapping Principle

Se conserva:

```text
query semantics
```

no:

```text
DQL syntax
```

---

## 182. Integration with 329

La coordinación del análisis pertenece a:

```text
329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md
```

El Doctrine Adapter será un proveedor especializado.

---

## 183. Integration with 330

La representación neutral se formalizará en:

```text
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
```

---

## 184. Integration with 331

Las transformaciones se ejecutarán mediante:

```text
331_DATABASE_MIGRATION_RULE_ENGINE.md
```

---

## 185. Integration with 332

La modificación de source code pertenece a:

```text
332_DATABASE_MIGRATION_CODE_TRANSFORMER.md
```

El adapter no deberá mezclar análisis con escritura de código.

---

## 186. Integration with 333

La equivalencia de schema pertenece a:

```text
333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md
```

---

## 187. Integration with 334

La equivalencia de lifecycle, UnitOfWork y comportamiento pertenece a:

```text
334_DATABASE_MIGRATION_BEHAVIOR_VERIFICATION_SYSTEM.md
```

---

## 188. Integration with 335

La coexistencia Doctrine/VoltStack se especificará en:

```text
335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md
```

---

## 189. Integration with 336

Shadow queries y comparación se especificarán en:

```text
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
```

---

## 190. Integration with 337–340

```text
337 → Migration Testing and Validation
338 → Migration CLI and Developer Experience
339 → Migration Reporting and Diagnostics
340 → Migration Rollback and Recovery
```

---

## 191. Arquitectura completa

```text
┌─────────────────────────────────────────────────────┐
│                 Doctrine Application                │
├─────────────────────────────────────────────────────┤
│ Entities │ ORM │ DBAL │ DQL │ Repositories │ Events │
└────┬────────┬──────┬─────┬────────────┬────────┬────┘
     │        │      │     │            │        │
     └────────┴──────┴─────┴────────────┴────────┘
                         │
                         ▼
              DoctrineMigrationAdapter
                         │
     ┌───────────────────┼─────────────────────┐
     ▼                   ▼                     ▼
 Metadata Scanner    Query Analyzer      Runtime Analyzer
     │                   │                     │
     ├─────────┬─────────┴─────────┬───────────┤
     ▼         ▼                   ▼           ▼
 Associations Types             UoW       Lifecycle
     │         │                   │           │
     └─────────┴──────────┬────────┴───────────┘
                          ▼
                Doctrine Source Model
                          │
                          ▼
                     Normalizer
                          │
                          ▼
              Intermediate Database Model
                          │
                          ▼
                 Migration Rule Engine
                          │
                          ▼
                   VoltStack Model
```

---

## 192. Flujo recomendado

```text
1. Detect Doctrine versions
          ↓
2. Detect mapping sources
          ↓
3. Discover entities/embeddables
          ↓
4. Resolve physical schema mappings
          ↓
5. Analyze identifiers/types
          ↓
6. Analyze associations
          ↓
7. Analyze repositories/queries
          ↓
8. Analyze EntityManager/UnitOfWork usage
          ↓
9. Analyze lifecycle/events/filters
          ↓
10. Analyze inheritance/extensions
          ↓
11. Inspect actual schema
          ↓
12. Detect conflicts
          ↓
13. Normalize
          ↓
14. Generate migration findings
```

---

## 193. Decisiones arquitectónicas

### Decisión 1

Doctrine tendrá un adapter oficial independiente del core de Database.

### Decisión 2

El adapter cubrirá tanto ORM como DBAL, pero mantendrá sus responsabilidades diferenciadas.

### Decisión 3

Attributes, annotations, XML y YAML convergerán a un descriptor neutral antes de migrarse.

### Decisión 4

Las naming strategies se resolverán hacia nombres físicos explícitos por defecto.

### Decisión 5

La semántica del UnitOfWork se preservará, pero no sus implementaciones internas.

### Decisión 6

Las garantías de Identity Map deberán verificarse cuando la aplicación dependa de ellas.

### Decisión 7

Owning/inverse side, cascade y orphan removal serán metadata crítica.

### Decisión 8

Custom Types no se convertirán automáticamente sin comprender sus conversiones PHP/DB.

### Decisión 9

DQL se transformará semánticamente mediante un modelo intermedio y no por sustitución textual.

### Decisión 10

SQL nativo podrá conservarse cuando sea la alternativa más segura.

### Decisión 11

Lifecycle callbacks, listeners, subscribers y filters deberán verificarse antes del cutover.

### Decisión 12

Inheritance mappings deberán conservar discriminadores y representación física existente.

### Decisión 13

La migración de ORM no implicará rediseñar el schema.

### Decisión 14

El adapter analizará acoplamiento Doctrine en toda la aplicación.

### Decisión 15

Doctrine y VoltStack podrán coexistir temporalmente mediante Dual ORM Mode.

### Decisión 16

El adapter será removible y no formará parte del runtime final de Database.

---

## 194. Criterios de finalización

Una migración Doctrine podrá declararse completa cuando:

```text
0 Doctrine EntityManager dependencies at runtime
0 Doctrine repository dependencies at runtime
0 Doctrine QueryBuilder/DQL dependencies at runtime
0 Doctrine DBAL dependencies requiring legacy runtime
0 unresolved custom types
0 unresolved lifecycle callbacks
0 unresolved filters
0 unresolved inheritance mappings
0 unresolved UnitOfWork dependencies
0 compatibility adapters required
schema verification passes
behavior verification passes
transaction verification passes
test suite passes
```

Las migraciones históricas o documentación archivada pueden conservar referencias sin considerarse dependencias runtime.

---

## 195. Resultado esperado

La aplicación pasará de:

```text
Application
    │
    ▼
Doctrine ORM
    │
    ├── EntityManager
    ├── UnitOfWork
    ├── Identity Map
    ├── DQL
    └── DBAL
          │
          ▼
       Database
```

hacia:

```text
Application
    │
    ▼
VoltStack Persistence Contracts
    │
    ▼
Quantum/Database
    │
    ├── Entity Metadata
    ├── Repository
    ├── Persistence Context
    ├── Unit of Work
    ├── Identity Map
    ├── Query AST
    ├── SQL Compiler
    ├── Transactions
    └── Drivers
          │
          ▼
       Database
```

---

## 196. Principio final

```text
Understand Doctrine
        ↓
Extract Metadata
        ↓
Extract Persistence Semantics
        ↓
Normalize
        ↓
Transform Safely
        ↓
Verify UnitOfWork / Identity / Queries
        ↓
Remove Doctrine
```

VoltStack no deberá copiar la implementación interna de Doctrine.

Deberá preservar únicamente aquellas garantías y comportamientos que formen parte del contrato real de la aplicación.

---

## 197. Conclusión

`DATABASE_DOCTRINE_MIGRATION_ADAPTER` proporciona la capa especializada necesaria para migrar aplicaciones construidas sobre Doctrine ORM y DBAL hacia la arquitectura nativa de VoltStack Database.

Doctrine aporta conceptos particularmente relevantes para VoltStack:

```text
Data Mapper
Entity Metadata
EntityManager
Unit of Work
Identity Map
Repositories
Explicit Associations
Lifecycle
DBAL
```

pero estos conceptos deberán pasar por el modelo intermedio antes de convertirse a implementaciones VoltStack.

La regla arquitectónica final será:

```text
Doctrine is understood at the migration boundary.

Doctrine does not enter the VoltStack Database core.

Persistence semantics are preserved.

Implementation dependencies disappear.
```

---

**Documento:** `327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**Estado:** Architectural Specification
