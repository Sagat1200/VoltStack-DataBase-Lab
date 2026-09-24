# 300_DATABASE_CUSTOM_ORM_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## Custom ORM Extension System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 300 — Custom ORM Extension System  
**Bloque:** 30 — Extensibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM.md`  
**Siguiente documento:** `301_DATABASE_DATABASE_CAPABILITY_DISCOVERY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial mediante la cual VoltStack permitirá extender el ORM de:

```text
VoltStack/Quantum/Database
```

sin crear motores de persistencia paralelos ni romper las invariantes centrales de:

```text
EntityManager
Repository
Model API
Entity Metadata
Mapping
UnitOfWork
IdentityMap
Change Tracking
Persistence Engine
Hydration
Relationships
Type System
Transactions
Query Engine
Cache
Events
Security
Persistent Runtime
```

El sistema deberá permitir que paquetes oficiales, plugins y extensiones de terceros agreguen comportamiento al ORM mediante **extension points explícitos, tipados, deterministas y gobernados**.

La regla central será:

> **Una ORM Extension podrá ampliar metadata, mapping, lifecycle, query ergonomics, hydration, persistence policies y otros puntos formalmente soportados, pero no deberá crear un segundo UnitOfWork, una segunda IdentityMap, un segundo EntityManager autoritativo ni un motor de persistencia paralelo que compita con el ORM central de VoltStack.**

Formalmente:

```text
Custom ORM Extension
=
Stable Identity
+
Descriptor
+
Explicit Extension Points
+
Metadata Integration
+
Mapping Integration
+
Lifecycle Integration
+
Query Integration
+
Persistence Integration
+
Hydration Integration
+
Relationship Integration
+
Security Boundaries
+
Transaction Awareness
+
Cache Awareness
+
Conflict Detection
+
Diagnostics
+
Conformance Testing
```

---

# 2. Objetivos

El sistema deberá permitir:

1. extender el ORM sin modificar Core;
2. registrar extensiones mediante Plugin System;
3. extender metadata;
4. extender mapping;
5. añadir atributos de mapping;
6. extender repositories;
7. extender Model API;
8. integrar Query Extensions;
9. añadir políticas de persistencia;
10. ampliar lifecycle;
11. ampliar change tracking mediante contratos controlados;
12. integrar custom value objects;
13. extender hydration mediante estrategias explícitas;
14. añadir relationship semantics soportadas;
15. integrar soft deletes;
16. integrar temporal/versioned entities;
17. integrar auditing;
18. integrar multitenancy;
19. integrar security policies;
20. preservar UnitOfWork;
21. preservar IdentityMap;
22. preservar transaction boundaries;
23. preservar ORM consistency;
24. detectar conflictos;
25. mantener determinismo;
26. soportar runtimes persistentes;
27. permitir testing/conformance;
28. mantener compatibilidad futura.

---

# 3. Principio arquitectónico fundamental

VoltStack tendrá:

```text
ONE ORM ENGINE
```

aunque permita múltiples APIs.

```text
Model API ──────────────┐
                       │
Repository API ─────────┼──→ EntityManager
                       │
Custom ORM Extension ───┘
                              │
                              ▼
                         UnitOfWork
                              │
                         IdentityMap
                              │
                    Persistence Engine
                              │
                         Query Engine
```

Nunca:

```text
Model API
   ↓
Persistence Engine A

Repository API
   ↓
Persistence Engine B

Plugin
   ↓
Persistence Engine C
```

---

# 4. ORM Extension ≠ ORM Replacement

Una extensión opera dentro de los contratos del ORM.

Un ORM replacement sería otro sistema.

```text
Extension
≠
Replacement
```

---

# 5. ORM Extension ≠ Second EntityManager

No deberá existir:

```text
PluginEntityManager
```

manteniendo una realidad distinta de entidades administradas.

---

# 6. ORM Extension ≠ Second UnitOfWork

Regla crítica:

```text
One persistence scope
→
One authoritative UnitOfWork
```

---

# 7. ORM Extension ≠ Second IdentityMap

Igualmente:

```text
One ORM scope
→
One canonical IdentityMap
```

---

# 8. ORM Extension ≠ Persistence Engine

Una extensión podrá intervenir en puntos explícitos.

No podrá crear silenciosamente otro pipeline de INSERT/UPDATE/DELETE.

---

# 9. ORM Extension ≠ Query Extension

El documento anterior define extensiones semánticas de consultas.

```text
ORM Extension
→ entity-oriented behavior

Query Extension
→ query-language semantics
```

Una ORM Extension podrá consumir Query Extensions.

---

# 10. ORM Extension ≠ Event Listener

Un listener es un mecanismo específico.

Una ORM Extension puede registrar múltiples componentes:

```text
metadata contributors
mapping resolvers
listeners
query extensions
hydration policies
persistence policies
```

---

# 11. ORM Extension ≠ Plugin

Plugin:

```text
distribution/composition mechanism
```

ORM Extension:

```text
ORM capability
```

---

# 12. Posición arquitectónica

```text
                    Application
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
      Model API                    Repository API
          │                             │
          └──────────────┬──────────────┘
                         ▼
                   EntityManager
                         │
              ┌──────────┼───────────┐
              ▼          ▼           ▼
          Metadata   IdentityMap   UnitOfWork
              │                      │
              ▼                      ▼
         ORM Extensions       Persistence Engine
              │                      │
              └──────────┬───────────┘
                         ▼
                    Query Engine
                         │
                         ▼
                  Execution Engine
```

---

# 13. Extension Identity

Toda ORM Extension tendrá:

```text
OrmExtensionId
```

estable.

Ejemplos:

```text
voltstack.orm.soft_delete
voltstack.orm.temporal
voltstack.orm.auditing

acme.orm.encryption
acme.orm.slug
acme.orm.domain_mapping
```

---

# 14. OrmExtensionId

Conceptualmente:

```php
final readonly class OrmExtensionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 15. OrmExtensionId ≠ FQCN

Los refactors PHP no deberán cambiar la identidad de la extensión.

---

# 16. OrmExtensionId ≠ Package Name

Un package podrá proporcionar varias extensiones ORM.

---

# 17. Extension Version

Podrá existir:

```text
OrmExtensionVersion
```

para versionar contratos semánticos.

---

# 18. ExtensionVersion ≠ PackageVersion

Son dimensiones distintas.

---

# 19. Extension Descriptor

Conceptualmente:

```php
final readonly class OrmExtensionDescriptor
{
    public function __construct(
        public OrmExtensionId $id,
        public OrmExtensionVersion $version,
        public OrmExtensionPointSet $extensionPoints,
        public array $dependencies,
        public array $conflicts,
    ) {}
}
```

---

# 20. Extension Points

La V1 podrá reconocer:

```text
METADATA
MAPPING
ENTITY_LIFECYCLE
REPOSITORY
MODEL_API
QUERY
CHANGE_TRACKING
PERSISTENCE
HYDRATION
RELATIONSHIP
TYPE
EVENT
SECURITY
CACHE
```

---

# 21. Extension Point ≠ Arbitrary Hook

No se expondrá:

```php
beforeAnything(callable $callback);
```

como mecanismo universal.

Cada extension point deberá tener un contrato definido.

---

# 22. ORM Extension Registry

Durante bootstrap:

```text
Plugin Discovery
      ↓
ORM Extension Providers
      ↓
Mutable ORM Extension Registry
      ↓
Dependency Validation
      ↓
Conflict Detection
      ↓
Extension Compilation
      ↓
Frozen ORM Extension Registry
```

---

# 23. Registry Freeze

Después del bootstrap:

```text
register()
remove()
replace()
```

estarán prohibidos por defecto.

---

# 24. Why Freeze

Permite:

```text
deterministic metadata
safe caches
predictable lifecycle
persistent runtime safety
faster dispatch
```

---

# 25. Duplicate ID

Resultado:

```text
DuplicateOrmExtensionException
```

---

# 26. Last Registration Wins

No será política predeterminada.

---

# 27. Extension Provider

Contrato conceptual:

```php
interface OrmExtensionProvider
{
    public function register(
        OrmExtensionRegistrar $registrar
    ): void;
}
```

---

# 28. Registrar

Podrá ofrecer APIs específicas:

```php
$registrar->metadata(...);
$registrar->mapping(...);
$registrar->lifecycle(...);
$registrar->repository(...);
$registrar->model(...);
$registrar->persistence(...);
$registrar->hydration(...);
```

---

# 29. Provider ≠ Runtime Service

El Provider registra componentes durante bootstrap.

No deberá convertirse en state container por request.

---

# 30. Metadata Extension

Una extensión podrá aportar metadata adicional a entidades.

Ejemplo:

```text
SoftDeleteMetadata
TemporalMetadata
EncryptionMetadata
AuditMetadata
```

---

# 31. Entity Metadata Remains Canonical

La metadata extendida deberá converger en:

```text
EntityMetadata
```

o estructuras asociadas gobernadas por el Metadata System.

No existirá:

```text
CoreMetadata
PluginMetadataReality
```

como realidades independientes.

---

# 32. Metadata Contributor

Conceptualmente:

```php
interface EntityMetadataContributor
{
    public function contribute(
        EntityMetadataBuilder $metadata,
        MetadataCompilationContext $context
    ): void;
}
```

---

# 33. Metadata Compilation

Las contribuciones deberán resolverse durante:

```text
metadata discovery/compilation
```

cuando sea posible.

No por cada query.

---

# 34. Metadata Immutability

Después de compilation:

```text
CompiledEntityMetadata
```

será inmutable.

---

# 35. Metadata Extension ≠ Runtime Mutation

Incorrecto:

```php
$metadata->addField(...)
```

durante un request normal.

---

# 36. Metadata Conflict

Si dos extensiones intentan definir semánticas incompatibles para la misma metadata:

```text
OrmMetadataConflictException
```

---

# 37. Mapping Extension

Permitirá ampliar:

```text
PHP property
↔
ORM semantic mapping
```

---

# 38. Custom Mapping Attribute

Ejemplo:

```php
#[Encrypted]
private string $taxId;
```

o:

```php
#[Slug(source: 'title')]
private string $slug;
```

---

# 39. Attribute ≠ Behavior

El atributo describe intención.

Un resolver interpreta esa intención.

---

# 40. Mapping Attribute Resolver

Conceptualmente:

```php
interface OrmMappingAttributeResolver
{
    public function resolve(
        ReflectionAttribute $attribute,
        MappingContext $context
    ): MappingContribution;
}
```

---

# 41. Reflection Hot Path

La reflexión deberá ocurrir principalmente durante metadata compilation.

No durante cada hydration/flush.

---

# 42. Mapping ≠ Schema

Una extensión ORM no deberá asumir que añadir metadata modifica automáticamente la base de datos.

```text
ORM Mapping
≠
Schema Definition
```

---

# 43. Schema Integration

Si una extensión necesita schema support:

```text
ORM Extension
       ↓
Schema Extension/Metadata Bridge
       ↓
Schema Model
```

de forma explícita.

---

# 44. Mapping ≠ Type

Una mapping extension podrá usar un custom type.

Pero:

```text
Mapping
≠
Type System
```

---

# 45. Model API Extension

VoltStack permitirá ampliar la API Laravel-like sin crear otro ORM.

Ejemplo conceptual:

```php
User::query()->withTrashed();
```

---

# 46. Model API Extension → Query Semantics

`withTrashed()` deberá convertirse en una modificación explícita del Query Model/policy.

No ejecutar consultas directamente.

---

# 47. Static API ≠ Static ORM State

Regla ya establecida:

```text
static syntax
≠
static EntityManager
```

---

# 48. Model Extension Resolution

Las extensiones Model deberán resolver el contexto mediante:

```text
ModelContextResolver
```

o infraestructura equivalente.

---

# 49. Model Extension State

No deberá almacenarse en:

```php
static $currentTenant;
static $withTrashed;
static $manager;
```

---

# 50. Repository Extension

Podrán proporcionarse repositories especializados.

Ejemplo:

```php
final class UserRepository extends Repository
{
    public function activeAdministrators(): array
    {
        // semantic query
    }
}
```

---

# 51. Repository ≠ Persistence Engine

Un custom repository deberá usar:

```text
EntityManager
Query Engine
UnitOfWork
```

existentes.

---

# 52. Repository Factory Extension

Podrá existir:

```text
RepositoryFactory
```

para seleccionar repositories especializados por metadata.

---

# 53. Repository Identity

La resolución deberá ser determinista.

---

# 54. Repository Scope

Un Repository no deberá capturar permanentemente un EntityManager de otro request.

---

# 55. Persistent Runtime Rule

```text
Repository Definition
→ reusable

Repository Instance bound to scoped EntityManager
→ operation/request scoped
```

cuando contenga contexto mutable.

---

# 56. Query Integration

Una ORM Extension podrá registrar:

```text
query scopes
entity filters
custom query methods
query extensions
```

---

# 57. ORM Query Scope

Ejemplo:

```text
SoftDeleteScope
```

podrá añadir:

```text
deleted_at IS NULL
```

semánticamente.

---

# 58. Scope ≠ Raw SQL

Preferido:

```text
Scope
→ Query AST transformation
```

No:

```text
Scope
→ append "AND deleted_at IS NULL"
```

---

# 59. Query Scope Composition

Múltiples scopes deberán componerse de forma determinista.

---

# 60. Scope Identity

Cada scope tendrá:

```text
OrmQueryScopeId
```

---

# 61. Scope Ordering

Sólo deberá importar cuando exista dependencia semántica.

---

# 62. Scope Conflict

Conflictos deberán detectarse explícitamente.

---

# 63. Scope Disable

Deshabilitar un scope deberá requerir una API explícita.

Ejemplo:

```php
$query->withoutScope(SoftDeleteScope::class);
```

---

# 64. Security Scope

Los scopes relacionados con autorización/tenant no deberán ser deshabilitables mediante APIs ordinarias.

---

# 65. Scope Classification

Podrá existir:

```text
CONVENIENCE
DOMAIN
SECURITY
TENANCY
SYSTEM
```

---

# 66. Security Scope ≠ Convenience Scope

Sus políticas de bypass serán diferentes.

---

# 67. UnitOfWork Extension

Éste será uno de los puntos más restringidos.

---

# 68. One Authoritative UnitOfWork

Nunca:

```text
CoreUnitOfWork
+
PluginUnitOfWork
```

administrando entidades simultáneamente.

---

# 69. UnitOfWork Extension Contract

Una extensión podrá contribuir:

```text
change interpretation
persistence metadata
additional validation
dependency information
lifecycle scheduling
```

mediante contratos controlados.

---

# 70. UnitOfWork Extension ≠ UoW Replacement

No podrá redefinir:

```text
entity state authority
change-set authority
flush ownership
identity semantics
```

arbitrariamente.

---

# 71. Change Tracking Extension

Podrá aportar estrategias para casos específicos.

Ejemplo:

```text
custom immutable value object comparison
encrypted value comparison
domain collection comparison
```

---

# 72. Change Tracking Strategy

Conceptualmente:

```php
interface ChangeTrackingExtension
{
    public function compare(
        mixed $original,
        mixed $current,
        ChangeTrackingContext $context
    ): ChangeResult;
}
```

---

# 73. Change Tracking ≠ Persistence

Detectar un cambio no significa ejecutar UPDATE.

---

# 74. Snapshot Semantics

Una extensión deberá definir qué forma canónica se almacena en snapshots.

---

# 75. Mutable Value Objects

Deberán tratarse con especial cuidado.

El sistema no deberá depender únicamente de referencia de objeto.

---

# 76. Change Result

Podrá distinguir:

```text
UNCHANGED
CHANGED
UNKNOWN
```

---

# 77. UNKNOWN Change

No deberá convertirse silenciosamente en `UNCHANGED`.

---

# 78. Persistence Extension

Permitirá participar en la preparación semántica de persistence operations.

---

# 79. Persistence Extension ≠ SQL Generator

La extensión deberá producir/modificar:

```text
Persistence Operation
Query Model
Persistence Metadata
```

según el extension point.

No SQL.

---

# 80. Persistence Pipeline

```text
Entity State
     ↓
UnitOfWork
     ↓
ChangeSet
     ↓
Persistence Extension Points
     ↓
Persistence Planner
     ↓
Query Model
     ↓
Query Engine
```

---

# 81. Pre-persistence Extension

Podrá realizar validaciones o transformaciones permitidas antes de planificar.

---

# 82. Persistence Mutation Risk

Modificar una entidad durante `flush()` puede crear nuevos changesets.

Por ello deberá existir política explícita.

---

# 83. Flush Stabilization

Si las extensiones pueden alterar state:

```text
Detect Changes
    ↓
Apply Controlled Extensions
    ↓
Recompute Affected ChangeSets
    ↓
Validate Stable Persistence Graph
```

---

# 84. Infinite Flush Mutation

El sistema deberá detectar:

```text
extension modifies entity
→ recompute
→ extension modifies again
→ ...
```

y abortar.

---

# 85. Flush Iteration Budget

Podrá existir un límite defensivo:

```text
MaxChangeSetStabilizationIterations
```

---

# 86. Persistence Extension Ordering

Deberá ser determinista.

---

# 87. Generated Values

Una extensión podrá participar en valores generados:

```text
slug
audit timestamp
domain identifier
```

si su semántica está declarada.

---

# 88. Generated Value Timing

Deberá indicar:

```text
ON_PERSIST
ON_FLUSH
BEFORE_INSERT
AFTER_INSERT
ON_UPDATE
```

según el contrato.

---

# 89. Generated Value ≠ Database Generated Value

Distinción:

```text
Application Generated
≠
Database Generated
```

---

# 90. Database-generated Values

Deberán regresar mediante:

```text
Persistence Engine
→ Result
→ Entity State synchronization
```

no mediante queries ocultas arbitrarias del plugin.

---

# 91. Persistence Veto

Algunos extension points podrán rechazar una operación.

Ejemplo:

```text
invalid encrypted mapping
immutable entity violation
```

---

# 92. Persistence Veto ≠ Authorization

La autorización pertenece a su sistema específico.

---

# 93. Hydration Extension

Permitirá extender cómo ciertos valores/objetos se reconstruyen.

---

# 94. Hydration Extension ≠ Second Hydrator Pipeline

Toda extensión deberá integrarse con:

```text
Hydration Plan
Entity Hydrator
IdentityMap
```

existentes.

---

# 95. Identity Before Extension

Cuando se hidrata una entidad administrada:

```text
IdentityMap
```

seguirá siendo autoridad de identidad.

---

# 96. Extension Cannot Create Duplicate Managed Entity

Regla crítica:

> **Ninguna extensión de hydration podrá materializar una segunda instancia administrada para la misma EntityType + Identifier + Database/Tenant Context dentro del mismo scope.**

---

# 97. Custom Property Hydrator

Podrá existir para:

```text
value objects
encrypted properties
special collections
custom immutable structures
```

---

# 98. Hydration Assignment ≠ Dirty Mutation

Las asignaciones realizadas durante hydration deberán mantener la semántica del Hydration System.

---

# 99. Hydration Lifecycle

```text
Row
 ↓
Value Conversion
 ↓
Extension Hydration
 ↓
Entity Assignment
 ↓
Snapshot
 ↓
Managed State
 ↓
postLoad
```

según el plan aplicable.

---

# 100. postLoad Mutation

Si una extensión modifica una entidad después del baseline:

```text
change tracking
```

deberá observar la modificación normalmente.

---

# 101. Partial Hydration

Una extensión deberá respetar:

```text
LoadedFieldMask
```

---

# 102. Missing Field ≠ NULL

Nunca deberá convertir automáticamente:

```text
not selected
```

en:

```text
NULL
```

---

# 103. Relationship Extension

Permitirá añadir metadata/comportamiento a relaciones existentes.

---

# 104. Relationship Extension ≠ New Persistence Universe

Las relaciones deberán continuar integrándose con:

```text
Relationship Metadata
Relationship Loading
Relationship Persistence
UnitOfWork
IdentityMap
```

---

# 105. Custom Relationship Type

Será una API avanzada.

Podría ser útil para:

```text
specialized association semantics
external reference semantics
temporal relationship semantics
```

---

# 106. Custom Relationship Requirements

Deberá definir:

```text
identity semantics
ownership
cardinality
loading
persistence
cascade behavior
orphan semantics
query representation
```

---

# 107. Relationship Ownership

Toda relationship persistible deberá mantener una autoridad clara.

---

# 108. Two Owning Sides

No deberán existir accidentalmente.

---

# 109. Cascade Extensions

Podrán ampliar políticas, pero:

```text
cascade
```

deberá ser explícito.

---

# 110. Cascade Remove

Continuará siendo conservador por defecto.

---

# 111. Cross-tenant Relationships

No serán habilitadas simplemente porque una extensión pueda representarlas.

---

# 112. Type System Integration

Una ORM Extension podrá registrar:

```text
Custom Database Type
Value Object Mapping
Casting Policy
```

mediante los sistemas ya definidos.

---

# 113. ORM Type ≠ Cast

La extensión deberá preservar:

```text
Type
≠
Value Conversion
≠
Cast
≠
Value Object
```

---

# 114. Value Object Mapping

Ejemplo:

```php
final readonly class Money
{
    public function __construct(
        public string $currency,
        public string $amount,
    ) {}
}
```

Una extensión podrá mapearlo a:

```text
single column
multiple columns
JSON
```

si la metadata lo declara.

---

# 115. Value Object ≠ Entity

No tendrá IdentityMap identity propia salvo que realmente sea una Entity.

---

# 116. Encryption Extension Example

Una extensión podría proporcionar:

```php
#[Encrypted]
private string $ssn;
```

---

# 117. Encryption Architecture

```text
PHP Value
   ↓
Encryption Extension
   ↓
Canonical Persistent Value
   ↓
Type Conversion
   ↓
Database
```

y reversa durante hydration.

---

# 118. Encryption ≠ Database Type

Puede utilizar tipos existentes como binary/string.

---

# 119. Encryption Query Limitations

La extensión deberá declarar si el campo soporta:

```text
equality
ordering
LIKE
full-text
range
```

No deberá permitir operaciones semánticamente inválidas.

---

# 120. Lifecycle Extensions

Podrán participar en:

```text
prePersist
postPersist
preUpdate
postUpdate
preRemove
postRemove
postLoad
preFlush
postFlush
```

según los eventos formalmente soportados.

---

# 121. Lifecycle Hook ≠ Domain Event

Deberán mantenerse distintos.

---

# 122. Lifecycle Hook ≠ Transaction Event

También:

```text
postPersist
≠
afterCommit
```

---

# 123. postPersist ≠ Commit Success

Una entidad puede haberse insertado y luego la transacción hacer rollback.

---

# 124. External Side Effects

No deberán ejecutarse como si `postPersist` significara commit durable.

---

# 125. Outbox

Para side effects externos confiables:

```text
Outbox Pattern
```

será preferido cuando aplique.

---

# 126. Lifecycle Failure Policy

Cada extension point deberá definir si un error:

```text
aborts operation
marks EntityManager tainted
is diagnostic only
```

---

# 127. No Swallowed Extension Errors

Errores relevantes no deberán desaparecer silenciosamente.

---

# 128. Transaction Boundaries

Una ORM Extension deberá respetar:

```text
flush ≠ commit
```

---

# 129. Extension Cannot Commit Hidden Transaction

Prohibido:

```php
$connection->commit();
```

dentro de un ORM extension hook ordinario.

---

# 130. Extension Cannot Begin Hidden Transaction

Igualmente prohibido salvo un extension point especializado explícitamente diseñado para ello.

---

# 131. Savepoints

Una extensión no deberá crear savepoints invisibles para TransactionManager.

---

# 132. Rollback

```text
Database Rollback
≠
Automatic Object Graph Rewind
```

continúa siendo invariante.

---

# 133. Unknown Transaction Outcome

Las extensiones deberán preservar:

```text
UNKNOWN
```

y nunca convertirlo en success/rollback supuesto.

---

# 134. EntityManager State

Si una extensión provoca una falla que vuelve inseguro continuar:

```text
EntityManager
→ TAINTED
```

según política.

---

# 135. Persistence Consistency

Una extensión no podrá marcar:

```text
CONSISTENT
```

sin evidencia suficiente.

---

# 136. Cache Integration

Una ORM Extension podrá participar en:

```text
Entity Cache
Metadata Cache
Query Cache
Result Cache
```

mediante contratos explícitos.

---

# 137. Extension Cache ≠ IdentityMap

Una cache de plugin nunca deberá sustituir la canonical identity del scope.

---

# 138. Entity Cache Payload

Deberá almacenar:

```text
canonical entity state
```

no:

```text
live managed object
```

---

# 139. Metadata Cache

Las metadata extensions podrán formar parte de:

```text
MetadataFingerprint
```

---

# 140. Extension Generation

Modificar metadata extension deberá invalidar metadata compilada incompatible.

---

# 141. Cache Fingerprint

Podrá incluir:

```text
OrmExtensionId
OrmExtensionVersion
MetadataGeneration
MappingGeneration
```

cuando afecten la representación cacheada.

---

# 142. After-commit Cache Actions

Invalidaciones/publicaciones que dependan de commit deberán respetar Transaction Events.

---

# 143. UNKNOWN Commit

Deberá provocar estrategia conservadora.

---

# 144. Security Integration

Una ORM Extension no deberá crear bypasses de:

```text
authorization
tenant isolation
data access policy
sensitive data protection
```

---

# 145. Data Access Extension

Una extensión podrá añadir políticas.

Pero la decisión de acceso deberá integrarse con:

```text
Authorization/Data Access Security System
```

---

# 146. Hidden Query Bypass

Incorrecto:

```text
extension uses direct Driver query
```

para evitar scopes/policies.

---

# 147. Query Engine Required

Las queries generadas por ORM Extensions deberán pasar por Query Engine salvo APIs internas explícitas y justificadas.

---

# 148. Tenant Context

La extensión deberá usar el:

```text
DatabaseContext
```

scoped.

Nunca resolver tenant mediante static global.

---

# 149. Tenant Metadata

Si metadata depende del tenant, deberá existir una arquitectura explícita.

No deberá mutarse metadata global compartida por request.

---

# 150. Preferred Tenant-specific Model

```text
Base Compiled Metadata
+
Scoped Tenant Mapping Overlay
```

sólo cuando realmente sea necesario.

---

# 151. Overlay ≠ Global Mutation

Crítico para runtimes persistentes.

---

# 152. Sharding Integration

Las extensiones no deberán ocultar shard keys al Query/Partition Routing System.

---

# 153. Read/Write Routing

Una extensión no seleccionará manualmente replica/writer salvo un componente formal de routing.

---

# 154. Events

Una ORM Extension podrá registrar lifecycle/event subscribers.

---

# 155. Event Ordering

Será determinista cuando importe.

---

# 156. Event Priority

No deberá utilizarse para ocultar conflictos semánticos.

---

# 157. Async Events

No podrán depender de live managed entities.

---

# 158. Event Payload

Deberá ser serializable/canonical cuando cruce el proceso.

---

# 159. Extension Dependencies

Una ORM Extension podrá declarar:

```text
requires
optional
conflicts
before
after
```

---

# 160. Example

```text
acme.orm.encrypted_search
requires:
    acme.orm.encryption
    acme.query.encrypted_search
```

---

# 161. Dependency Resolution

```text
ORM Extensions
     ↓
Dependency Graph
     ↓
Cycle Detection
     ↓
Compatibility Validation
     ↓
Topological Resolution
     ↓
Frozen Registry
```

---

# 162. Cycles

Resultado:

```text
OrmExtensionDependencyCycleException
```

---

# 163. Conflict Detection

Conflictos podrán ocurrir sobre:

```text
metadata key
mapping attribute
query scope
repository resolver
hydration strategy
persistence policy
relationship extension
```

---

# 164. Conflict Resolution

Por defecto:

```text
explicit error
```

No:

```text
last wins
```

---

# 165. Overrides

Cuando un extension point permita override:

```text
target
expected version
replacement contract
explicit permission
```

deberán declararse.

---

# 166. Extension Ordering

El orden deberá provenir de:

```text
dependencies
explicit ordering constraints
extension-point semantics
```

no del orden accidental de Composer.

---

# 167. Plugin Integration

Un Database Plugin podrá registrar:

```text
Custom Driver
Custom Dialect
Custom Compiler
Query Extensions
ORM Extensions
Types
Capabilities
```

---

# 168. Example Plugin

```text
AcmeEncryptedOrmPlugin
│
├── EncryptionType
├── EncryptedAttribute
├── MetadataContributor
├── ChangeTrackingStrategy
├── HydrationExtension
├── PersistenceExtension
└── QueryRestrictions
```

---

# 169. Plugin Activation

Instalación del package no deberá implicar necesariamente activación automática si la extensión requiere configuración explícita.

---

# 170. Discovery ≠ Activation

Regla:

```text
Discovered
≠
Enabled
```

---

# 171. Enabled ≠ Applicable

Una extensión habilitada puede aplicar sólo a entidades con metadata específica.

---

# 172. Applicability

Podrá resolverse por:

```text
entity metadata
mapping attribute
interface
explicit configuration
```

---

# 173. Interface-based Mapping

Ejemplo:

```php
interface SoftDeletable
{
}
```

podrá contribuir metadata.

Pero deberá evitarse magic behavior excesivo.

---

# 174. Explicit Metadata Preferred

Para comportamientos con consecuencias importantes:

```text
explicit mapping
```

será preferible.

---

# 175. Extension Context

Podrá existir:

```text
OrmExtensionContext
```

scoped por operación.

---

# 176. Context May Contain

```text
DatabaseContext
EntityManager reference
operation metadata
transaction context reference
diagnostics
```

según el extension point.

---

# 177. Context Shall Not Become Service Locator

No deberá permitir acceso indiscriminado a todo el Container.

---

# 178. Scoped Context

No podrá conservarse después de finalizar el scope.

---

# 179. Persistent Runtime Architecture

Clasificación:

```text
SHAREABLE
├── frozen extension descriptors
├── compiled metadata
├── immutable mapping definitions
├── stateless validators
└── immutable dispatch tables

SCOPED
├── EntityManager
├── UnitOfWork
├── IdentityMap
├── DatabaseContext
├── TransactionContext
├── hydration session
└── ORM extension operation context
```

---

# 180. FrankenPHP

Será el runtime persistente principal para pruebas de aislamiento.

---

# 181. RoadRunner

Deberá preservar las mismas invariantes.

---

# 182. OpenSwoole

El estado mutable deberá aislarse también entre coroutines.

---

# 183. Extension Reset

Las extensiones con state scoped deberán implementar contratos de cleanup/reset cuando corresponda.

---

# 184. Reset Failure

Si un extension state no puede limpiarse con seguridad:

```text
scope/resource
→ quarantine/discard
```

según el recurso afectado.

---

# 185. Static Mutable State

Estará prohibido para:

```text
current entity
current tenant
current EntityManager
current UnitOfWork
current transaction
current hydration
```

---

# 186. Diagnostics

El sistema deberá poder responder:

```text
which ORM extensions are active?
which extension modified this metadata?
why is this field mapped this way?
which scopes apply to this entity?
which extension changed this ChangeSet?
which extension participated in hydration?
which persistence policies are active?
```

---

# 187. ORM Extension Inspector

Conceptualmente:

```php
interface OrmExtensionInspector
{
    public function inspect(
        OrmExtensionId $id
    ): OrmExtensionDiagnosticReport;
}
```

---

# 188. Entity Metadata Diagnostics

Ejemplo:

```text
Entity:
App\Entity\User

Extensions:
- voltstack.orm.soft_delete
- acme.orm.encryption

Field:
email

Mapping:
string

Contributors:
CoreMapping
AcmeEncryptionExtension

Query Scopes:
SoftDeleteScope

Change Tracking:
DeferredImplicit
```

---

# 189. Extension Trace

En desarrollo podrá registrarse:

```text
Metadata Build
  → Core
  → Extension A
  → Extension B

Flush
  → Change Detection
  → Extension A
  → Persistence Planning
```

---

# 190. Diagnostics Security

No deberán mostrar automáticamente:

```text
decrypted values
credentials
tokens
sensitive entity fields
```

---

# 191. Telemetry

Podrá medir:

```text
extension metadata compilation time
extension hook duration
scope application count
hydration extension duration
persistence extension duration
extension errors
```

---

# 192. Telemetry Cardinality

`OrmExtensionId` puede ser una dimensión acotada.

Entity IDs no deberán usarse como metric labels.

---

# 193. Extension Hook Performance

Los hooks del hot path deberán ser eficientes.

---

# 194. Precompiled Dispatch

Preferido:

```text
EntityType
→ applicable extension set
```

en lugar de escanear todas las extensiones.

---

# 195. Metadata-driven Dispatch

Durante metadata compilation podrá calcularse:

```text
CompiledOrmExtensionPlan
```

por EntityType.

---

# 196. Compiled ORM Extension Plan

Conceptualmente:

```php
final readonly class CompiledOrmExtensionPlan
{
    public function __construct(
        public EntityType $entityType,
        public array $lifecycleHandlers,
        public array $changeTrackers,
        public array $persistenceHandlers,
        public array $hydrationHandlers,
        public array $queryScopes,
    ) {}
}
```

---

# 197. Plan Immutability

Será inmutable y cacheable.

---

# 198. Plan ≠ Runtime State

No almacenará:

```text
current entity
current ChangeSet
current transaction
```

---

# 199. Performance Principle

Preferido:

```text
compile extension decisions once
execute compact plan many times
```

---

# 200. Reflection Principle

Preferido:

```text
reflection at metadata compilation
```

no:

```text
reflection per entity hydration
```

---

# 201. ORM Extension Testing

Toda extensión deberá poder probarse mediante una suite formal.

---

# 202. Unit Tests

Adecuados para:

```text
metadata contribution
mapping resolution
change comparison
scope generation
extension dependency resolution
conflict detection
fingerprints
```

---

# 203. Integration Tests

Necesarios para:

```text
persistence
hydration
relationships
transactions
database-generated values
query scopes
real type round-trip
```

---

# 204. IdentityMap Conformance Test

Toda extensión de hydration deberá demostrar:

```php
$a = $repository->find(10);
$b = $repository->find(10);

assert($a === $b);
```

dentro del mismo scope/contexto.

---

# 205. UnitOfWork Conformance Test

Una extensión no deberá crear cambios invisibles al UoW.

---

# 206. Persistence Conformance Test

```text
Entity
 ↓
Extension
 ↓
UoW
 ↓
Persistence Engine
 ↓
DB
 ↓
Reload
 ↓
Expected State
```

---

# 207. Rollback Test

Deberá comprobarse:

```text
flush
→ rollback
```

sin asumir rewind automático del object graph.

---

# 208. Unknown Commit Test

Si la extensión participa en operaciones durante:

```text
commit sent
→ connection lost
```

deberá preservar `UNKNOWN`.

---

# 209. Transaction Event Test

`postPersist` no deberá tratarse como `afterCommit`.

---

# 210. Query Scope Test

Deberá comprobarse:

```text
scope enabled
scope explicitly disabled where allowed
scope non-bypassable where security critical
```

---

# 211. Tenant Isolation Test

Una extensión no deberá filtrar entidades entre tenants.

---

# 212. Persistent Runtime Test

Secuencia:

```text
Request A
Tenant A
Extension state A
    ↓
reset
    ↓
Request B
Tenant B
```

y demostrar:

```text
no state leakage
```

---

# 213. Metadata Cache Test

Cambiar extension generation deberá invalidar metadata incompatible.

---

# 214. Performance Test

Deberán medirse:

```text
metadata compilation overhead
per-hydration overhead
per-entity change tracking overhead
flush overhead
query scope overhead
memory overhead
```

---

# 215. Conformance Kit

VoltStack podrá proporcionar:

```text
OrmExtensionConformanceKit
```

---

# 216. Conformance Categories

```text
IDENTITY
METADATA
MAPPING
QUERY
UNIT_OF_WORK
IDENTITY_MAP
CHANGE_TRACKING
PERSISTENCE
HYDRATION
RELATIONSHIP
TRANSACTION
SECURITY
CACHE
RUNTIME
PERFORMANCE
```

---

# 217. Extension Maturity

Podrá declararse:

```text
EXPERIMENTAL
STABLE
DEPRECATED
```

---

# 218. Maturity ≠ Correctness

`STABLE` no sustituye pruebas.

---

# 219. Third-party Certification

En el futuro podrá existir:

```text
SELF_TESTED
COMMUNITY_VERIFIED
VOLTSTACK_VERIFIED
```

sin convertirlo en requisito técnico del Core.

---

# 220. Example: Soft Delete Extension

Metadata:

```php
#[SoftDelete(column: 'deleted_at')]
final class User
{
}
```

---

# 221. Soft Delete Query Behavior

Default:

```text
User Query
 ↓
SoftDeleteScope
 ↓
deleted_at IS NULL
```

---

# 222. withTrashed()

```php
User::query()->withTrashed();
```

no elimina SQL manualmente.

Modifica la política semántica del query.

---

# 223. Soft Delete Persistence

```php
$user->delete();
```

podrá convertirse semánticamente en:

```text
UPDATE deleted_at
```

mediante un persistence policy formal.

---

# 224. Soft Delete ≠ Core DELETE Rewrite Everywhere

La extensión deberá intervenir sólo para entidades aplicables.

---

# 225. forceDelete()

Podrá expresar:

```text
physical removal intent
```

de forma explícita.

---

# 226. Security

`forceDelete()` podrá requerir authorization adicional fuera del ORM.

---

# 227. Example: Auditing Extension

Podrá contribuir:

```text
created_at
updated_at
created_by
updated_by
```

según configuración.

---

# 228. Audit Extension ≠ Database Audit System

Timestamps de entidad no sustituyen:

```text
DATABASE_QUERY_AUDIT_SYSTEM
```

---

# 229. created_by

La extensión no deberá leer globales arbitrarios.

Deberá recibir un:

```text
ActorContext
```

mediante un contrato scoped.

---

# 230. Missing Actor

La política deberá definir:

```text
NULL
SYSTEM
REJECT
```

según contexto.

---

# 231. Example: Encryption Extension

```php
#[Encrypted(
    algorithm: EncryptionProfile::SENSITIVE_TEXT
)]
private string $nationalId;
```

---

# 232. Metadata

La metadata compilada podría contener:

```text
EncryptedFieldMetadata
├── profile
├── searchable
├── deterministic
└── key reference
```

nunca la clave secreta.

---

# 233. Key Material

No deberá almacenarse en metadata cache.

---

# 234. Key Provider

La resolución de claves pertenecerá al security/secret boundary correspondiente.

---

# 235. Change Tracking with Encryption

No deberá comparar ciphertext aleatorio para determinar cambio lógico.

---

# 236. Correct Comparison

Preferentemente:

```text
logical canonical value
```

o estrategia específica segura.

---

# 237. Query Restrictions

Un campo con cifrado no determinista podrá rechazar:

```text
WHERE encrypted_field = ?
```

si no existe una estrategia semánticamente válida.

---

# 238. Example: Temporal ORM Extension

Podrá añadir metadata:

```text
valid_from
valid_to
system_from
system_to
```

y query APIs como:

```php
$repository->asOf($instant);
```

---

# 239. Temporal ORM Extension

Deberá apoyarse en:

```text
DATABASE_TEMPORAL_DATA_SYSTEM
```

y Query Extensions apropiadas.

No reinventarlo dentro del ORM.

---

# 240. Example: Domain Slug Extension

```php
#[Slug(source: 'title')]
private string $slug;
```

podrá generar valor antes de persistencia.

---

# 241. Slug Collision

No deberá asumir unicidad sólo porque generó un string.

La garantía deberá provenir de:

```text
schema constraint
+
conflict handling policy
```

cuando sea necesaria.

---

# 242. Race Conditions

Un:

```text
SELECT slug exists
→ INSERT
```

no constituye garantía de unicidad.

---

# 243. Database Constraints Remain Authoritative

Cuando una invariante depende de concurrencia:

```text
DB constraint
```

será necesaria.

---

# 244. Custom Entity State

Una extensión no deberá introducir estados incompatibles paralelos a:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

sin extender formalmente el state model.

---

# 245. Extension-specific State

Podrá mantener metadata adicional:

```text
encrypted-field-dirty
soft-delete-intent
temporal-version-intent
```

pero deberá mapearse al state/persistence model central.

---

# 246. Entity Lifecycle Authority

El EntityManager seguirá gobernando:

```text
manage
detach
clear
persist
remove
flush
```

---

# 247. Extension Cannot Reattach Arbitrarily

La identidad deberá validarse mediante EntityManager/IdentityMap.

---

# 248. Lazy Loading

Una ORM Extension podrá participar en relationship loading.

Pero deberá respetar:

```text
ALLOW
WARN
FORBID
```

y context availability.

---

# 249. Detached Entity

Una extensión no deberá recuperar un EntityManager global para hacer lazy load.

---

# 250. Serialization

Una extensión no deberá disparar lazy loading automáticamente durante serialization salvo política explícita.

---

# 251. N+1 Detection

Custom relationship/query extensions deberán emitir suficiente metadata semántica para que:

```text
N+1 Detection System
```

pueda correlacionar cargas cuando sea posible.

---

# 252. Batch Loading

Custom relationships deberán integrarse con Batch Loading si declaran soporte.

---

# 253. Eager Loading

Una extensión no deberá asumir:

```text
eager
=
JOIN
```

El planner continuará seleccionando estrategia.

---

# 254. ORM Extension API Stability

Los extension points públicos deberán ser pequeños.

Preferentemente:

```text
OrmExtension
OrmExtensionProvider
OrmExtensionRegistrar

EntityMetadataContributor
OrmMappingAttributeResolver

OrmQueryScope
RepositoryFactoryExtension
ModelApiExtension

ChangeTrackingExtension
PersistenceExtension
HydrationExtension
RelationshipExtension
```

---

# 255. Internal APIs

No deberán hacerse públicas accidentalmente:

```text
internal UoW arrays
private hydration buffers
internal persistence queues
IdentityMap storage implementation
```

---

# 256. UnitOfWork Encapsulation

Especialmente importante:

> **Las extensiones no recibirán acceso mutable indiscriminado a las estructuras internas del UnitOfWork.**

---

# 257. Controlled UoW API

Cuando una extensión necesite interacción:

```text
UnitOfWorkExtensionContext
```

expondrá operaciones limitadas.

---

# 258. Example

En lugar de:

```php
$unitOfWork->dirtyEntities[] = $entity;
```

podría existir:

```php
$context->requestRecompute($entity);
```

---

# 259. Hydration Encapsulation

Igualmente, una extensión no deberá manipular directamente internals de IdentityMap.

---

# 260. Persistence Encapsulation

No deberá insertar operaciones directamente en arrays privados del Persistence Engine.

---

# 261. Architecture Tests

VoltStack deberá incluir pruebas que impidan dependencias prohibidas.

Ejemplo:

```text
ORM Extension package
must not depend directly on
PDO
```

salvo componentes explícitamente autorizados fuera del ORM extension layer.

---

# 262. Dependency Direction

```text
Custom ORM Extension
        ↓
ORM Public Extension Contracts
        ↓
ORM Core
        ↓
Query Engine
        ↓
Execution
        ↓
Connection
        ↓
Driver
```

Nunca:

```text
Driver
→ ORM Extension
```

---

# 263. Proposed Directory Structure

```text
src/Quantum/Database/Orm/Extension/
├── Contract/
│   ├── OrmExtension.php
│   ├── OrmExtensionProvider.php
│   ├── OrmExtensionRegistrar.php
│   └── OrmExtensionPoint.php
│
├── Identity/
│   ├── OrmExtensionId.php
│   ├── OrmExtensionVersion.php
│   └── OrmExtensionFingerprint.php
│
├── Descriptor/
│   ├── OrmExtensionDescriptor.php
│   ├── OrmExtensionDependency.php
│   └── OrmExtensionConflict.php
│
├── Registry/
│   ├── OrmExtensionRegistry.php
│   ├── MutableOrmExtensionRegistry.php
│   ├── FrozenOrmExtensionRegistry.php
│   └── OrmExtensionDispatchTable.php
│
├── Metadata/
│   ├── EntityMetadataContributor.php
│   ├── ExtensionMetadataBag.php
│   └── MetadataExtensionCompiler.php
│
├── Mapping/
│   ├── OrmMappingAttributeResolver.php
│   ├── MappingContribution.php
│   └── MappingExtensionRegistry.php
│
├── Model/
│   ├── ModelApiExtension.php
│   └── ModelExtensionResolver.php
│
├── Repository/
│   ├── RepositoryFactoryExtension.php
│   └── RepositoryExtensionResolver.php
│
├── Query/
│   ├── OrmQueryScope.php
│   ├── OrmQueryScopeId.php
│   ├── QueryScopeRegistry.php
│   └── QueryScopePolicy.php
│
├── ChangeTracking/
│   ├── ChangeTrackingExtension.php
│   ├── ChangeTrackingContext.php
│   └── ChangeResult.php
│
├── UnitOfWork/
│   ├── UnitOfWorkExtension.php
│   └── UnitOfWorkExtensionContext.php
│
├── Persistence/
│   ├── PersistenceExtension.php
│   ├── PersistenceExtensionContext.php
│   └── PersistenceExtensionPlan.php
│
├── Hydration/
│   ├── HydrationExtension.php
│   ├── PropertyHydrationExtension.php
│   └── HydrationExtensionContext.php
│
├── Relationship/
│   ├── RelationshipExtension.php
│   └── RelationshipExtensionMetadata.php
│
├── Lifecycle/
│   ├── OrmLifecycleExtension.php
│   └── LifecycleExtensionPolicy.php
│
├── Runtime/
│   ├── OrmExtensionContext.php
│   ├── CompiledOrmExtensionPlan.php
│   └── OrmExtensionResetter.php
│
├── Diagnostics/
│   ├── OrmExtensionInspector.php
│   ├── OrmExtensionDiagnosticReport.php
│   └── OrmExtensionTrace.php
│
└── Exception/
    ├── OrmExtensionException.php
    ├── DuplicateOrmExtensionException.php
    ├── OrmExtensionConflictException.php
    ├── OrmExtensionDependencyException.php
    ├── OrmExtensionDependencyCycleException.php
    ├── OrmMetadataConflictException.php
    ├── OrmExtensionRuntimeException.php
    └── OrmExtensionInvariantViolationException.php
```

---

# 264. Third-party Package Structure

```text
acme/voltstack-orm-extension/
├── composer.json
├── src/
│   ├── Plugin/
│   ├── Extension/
│   ├── Metadata/
│   ├── Mapping/
│   ├── Query/
│   ├── Persistence/
│   ├── Hydration/
│   └── Diagnostics/
└── tests/
    ├── Unit/
    ├── Integration/
    ├── Conformance/
    ├── Security/
    └── Runtime/
```

---

# 265. Architectural Invariants

## DB-CUSTOM-ORM-001

ORM Extension ≠ ORM Replacement.

## DB-CUSTOM-ORM-002

ORM Extension ≠ Second EntityManager.

## DB-CUSTOM-ORM-003

ORM Extension ≠ Second UnitOfWork.

## DB-CUSTOM-ORM-004

ORM Extension ≠ Second IdentityMap.

## DB-CUSTOM-ORM-005

ORM Extension ≠ Persistence Engine.

## DB-CUSTOM-ORM-006

ORM Extension ≠ Query Extension.

## DB-CUSTOM-ORM-007

ORM Extension ≠ Event Listener.

## DB-CUSTOM-ORM-008

ORM Extension ≠ Plugin.

## DB-CUSTOM-ORM-009

VoltStack mantendrá un único ORM Engine.

## DB-CUSTOM-ORM-010

Model API y Repository API convergerán al mismo EntityManager.

## DB-CUSTOM-ORM-011

OrmExtensionId será estable.

## DB-CUSTOM-ORM-012

OrmExtensionId ≠ FQCN.

## DB-CUSTOM-ORM-013

ExtensionVersion ≠ PackageVersion.

## DB-CUSTOM-ORM-014

Extension Point ≠ Arbitrary Hook.

## DB-CUSTOM-ORM-015

Registry será frozen después de bootstrap.

## DB-CUSTOM-ORM-016

Duplicate extension ID será error.

## DB-CUSTOM-ORM-017

Last registration wins estará prohibido por defecto.

## DB-CUSTOM-ORM-018

Provider ≠ Runtime State Container.

## DB-CUSTOM-ORM-019

Entity Metadata permanecerá canónica.

## DB-CUSTOM-ORM-020

Compiled metadata será inmutable.

## DB-CUSTOM-ORM-021

Metadata no se mutará por request.

## DB-CUSTOM-ORM-022

Mapping ≠ Schema.

## DB-CUSTOM-ORM-023

Mapping ≠ Type.

## DB-CUSTOM-ORM-024

Static Model API ≠ Static ORM State.

## DB-CUSTOM-ORM-025

Repository ≠ Persistence Engine.

## DB-CUSTOM-ORM-026

Repository no capturará EntityManager de otro scope.

## DB-CUSTOM-ORM-027

ORM scopes producirán Query AST.

## DB-CUSTOM-ORM-028

ORM scope ≠ Raw SQL.

## DB-CUSTOM-ORM-029

Security scopes no serán bypassables por API ordinaria.

## DB-CUSTOM-ORM-030

One persistence scope → one authoritative UnitOfWork.

## DB-CUSTOM-ORM-031

One ORM scope → one canonical IdentityMap.

## DB-CUSTOM-ORM-032

UnitOfWork Extension ≠ UoW Replacement.

## DB-CUSTOM-ORM-033

Change Tracking ≠ Persistence.

## DB-CUSTOM-ORM-034

UNKNOWN change ≠ UNCHANGED.

## DB-CUSTOM-ORM-035

Persistence Extension ≠ SQL Generator.

## DB-CUSTOM-ORM-036

Persistence extensions producirán semántica, no SQL.

## DB-CUSTOM-ORM-037

Flush mutation deberá estabilizarse.

## DB-CUSTOM-ORM-038

Infinite flush mutation será detectada.

## DB-CUSTOM-ORM-039

Application-generated value ≠ DB-generated value.

## DB-CUSTOM-ORM-040

Database-generated values regresarán por Persistence Engine.

## DB-CUSTOM-ORM-041

Hydration Extension ≠ Second Hydration Pipeline.

## DB-CUSTOM-ORM-042

IdentityMap conservará autoridad durante hydration.

## DB-CUSTOM-ORM-043

Una extensión no creará duplicate managed identity.

## DB-CUSTOM-ORM-044

Hydration assignment ≠ Dirty Mutation.

## DB-CUSTOM-ORM-045

Partial hydration respetará LoadedFieldMask.

## DB-CUSTOM-ORM-046

Missing Field ≠ NULL.

## DB-CUSTOM-ORM-047

Relationship Extension ≠ Parallel Relationship Engine.

## DB-CUSTOM-ORM-048

Relationship ownership permanecerá explícito.

## DB-CUSTOM-ORM-049

Cascade remove será conservador por defecto.

## DB-CUSTOM-ORM-050

Type ≠ Value Conversion ≠ Cast ≠ Value Object.

## DB-CUSTOM-ORM-051

Value Object ≠ Entity.

## DB-CUSTOM-ORM-052

Lifecycle Hook ≠ Domain Event.

## DB-CUSTOM-ORM-053

Lifecycle Hook ≠ Transaction Event.

## DB-CUSTOM-ORM-054

postPersist ≠ afterCommit.

## DB-CUSTOM-ORM-055

postPersist ≠ durable commit.

## DB-CUSTOM-ORM-056

External side effects no asumirán commit en postPersist.

## DB-CUSTOM-ORM-057

flush ≠ commit.

## DB-CUSTOM-ORM-058

ORM Extension no hará hidden commit.

## DB-CUSTOM-ORM-059

ORM Extension no hará hidden transaction.

## DB-CUSTOM-ORM-060

ORM Extension no hará hidden savepoint.

## DB-CUSTOM-ORM-061

Rollback ≠ Object Graph Rewind.

## DB-CUSTOM-ORM-062

UNKNOWN transaction outcome permanecerá UNKNOWN.

## DB-CUSTOM-ORM-063

EntityManager podrá quedar TAINTED ante fallos inseguros.

## DB-CUSTOM-ORM-064

Extension Cache ≠ IdentityMap.

## DB-CUSTOM-ORM-065

Entity Cache no almacenará live managed objects.

## DB-CUSTOM-ORM-066

Metadata extensions participarán en metadata fingerprints.

## DB-CUSTOM-ORM-067

Extension generations invalidarán caches incompatibles.

## DB-CUSTOM-ORM-068

After-commit cache actions respetarán transaction outcome.

## DB-CUSTOM-ORM-069

UNKNOWN commit tendrá invalidación conservadora.

## DB-CUSTOM-ORM-070

ORM Extension no bypassará authorization.

## DB-CUSTOM-ORM-071

ORM Extension no bypassará tenant isolation.

## DB-CUSTOM-ORM-072

ORM Extension no bypassará Data Access Security.

## DB-CUSTOM-ORM-073

ORM Extension no hará direct Driver query como bypass.

## DB-CUSTOM-ORM-074

Tenant state no será static/global.

## DB-CUSTOM-ORM-075

Tenant-specific metadata no mutará metadata global.

## DB-CUSTOM-ORM-076

Extension no ocultará shard keys relevantes.

## DB-CUSTOM-ORM-077

Extension no seleccionará replica directamente.

## DB-CUSTOM-ORM-078

Async events no transportarán live managed entities.

## DB-CUSTOM-ORM-079

Extension dependency cycles serán error.

## DB-CUSTOM-ORM-080

Extension conflicts serán explícitos.

## DB-CUSTOM-ORM-081

Ordering no dependerá del orden accidental de Composer.

## DB-CUSTOM-ORM-082

Discovery ≠ Activation.

## DB-CUSTOM-ORM-083

Enabled ≠ Applicable.

## DB-CUSTOM-ORM-084

OrmExtensionContext no será Service Locator.

## DB-CUSTOM-ORM-085

Scoped contexts no sobrevivirán al scope.

## DB-CUSTOM-ORM-086

Immutable extension definitions podrán compartirse.

## DB-CUSTOM-ORM-087

Mutable ORM state será scoped.

## DB-CUSTOM-ORM-088

Static mutable ORM extension state estará prohibido.

## DB-CUSTOM-ORM-089

Diagnostics no expondrán sensitive values.

## DB-CUSTOM-ORM-090

Hot path dispatch será precompilable.

## DB-CUSTOM-ORM-091

CompiledOrmExtensionPlan será inmutable.

## DB-CUSTOM-ORM-092

CompiledOrmExtensionPlan ≠ Runtime State.

## DB-CUSTOM-ORM-093

Reflection deberá concentrarse en metadata compilation.

## DB-CUSTOM-ORM-094

IdentityMap conformance será obligatoria para hydration extensions.

## DB-CUSTOM-ORM-095

UoW extensions no crearán invisible changes.

## DB-CUSTOM-ORM-096

Soft Delete ≠ Physical Delete.

## DB-CUSTOM-ORM-097

Audit timestamps ≠ Database Audit System.

## DB-CUSTOM-ORM-098

Encryption keys no pertenecerán a metadata cache.

## DB-CUSTOM-ORM-099

Encrypted fields declararán query limitations.

## DB-CUSTOM-ORM-100

Temporal ORM extensions reutilizarán Temporal Data System.

## DB-CUSTOM-ORM-101

Application uniqueness checks ≠ Database uniqueness guarantee.

## DB-CUSTOM-ORM-102

Database constraints conservarán autoridad bajo concurrencia.

## DB-CUSTOM-ORM-103

Extension-specific entity state se integrará al state model central.

## DB-CUSTOM-ORM-104

EntityManager conservará lifecycle authority.

## DB-CUSTOM-ORM-105

Detached entity no recuperará global EntityManager.

## DB-CUSTOM-ORM-106

Serialization no disparará lazy loading por defecto.

## DB-CUSTOM-ORM-107

Custom relationships deberán integrarse con N+1 detection cuando sea posible.

## DB-CUSTOM-ORM-108

Eager Loading ≠ JOIN.

## DB-CUSTOM-ORM-109

Extensions no tendrán mutable unrestricted access al UoW.

## DB-CUSTOM-ORM-110

Extensions no manipularán internals del IdentityMap.

## DB-CUSTOM-ORM-111

Extensions no manipularán internals del Persistence Engine.

## DB-CUSTOM-ORM-112

Driver no dependerá de ORM Extension.

## DB-CUSTOM-ORM-113

Custom ORM Extension preservará las invariantes globales del ORM.

---

# 266. Anti-patrones

## 266.1 Segundo EntityManager

```text
PluginEntityManager
```

Incorrecto.

---

## 266.2 Segundo UnitOfWork

Incorrecto.

---

## 266.3 Segunda IdentityMap

Incorrecto.

---

## 266.4 Plugin haciendo INSERT directamente

Incorrecto.

---

## 266.5 Lifecycle listener haciendo commit

Incorrecto.

---

## 266.6 Metadata mutable por request

Incorrecto.

---

## 266.7 Repository singleton con EntityManager scoped

Incorrecto.

---

## 266.8 Scope mediante concatenación SQL

Incorrecto.

---

## 266.9 Security scope deshabilitable con API genérica

Incorrecto.

---

## 266.10 Extension accediendo a arrays internos del UoW

Incorrecto.

---

## 266.11 Hydrator extension creando duplicate entity

Incorrecto.

---

## 266.12 Tratar field no cargado como NULL

Incorrecto.

---

## 266.13 postPersist enviando irreversible external message

Incorrecto si depende de commit durable.

---

## 266.14 Cache de objetos managed entre requests

Incorrecto.

---

## 266.15 Tenant en variable static

Incorrecto.

---

## 266.16 Modificar metadata global para cada tenant

Incorrecto.

---

## 266.17 Query directa al Driver para evitar scopes

Incorrecto.

---

## 266.18 Resolver services desde Container por entidad

Incorrecto.

---

## 266.19 Reflection por propiedad en cada hydration

Debe evitarse.

---

## 266.20 Extension order dependiente de Composer

Incorrecto.

---

## 266.21 Last registration wins

Incorrecto.

---

## 266.22 Autoactivar package peligroso al instalarse

Debe evitarse.

---

## 266.23 Cifrado no determinista comparado por ciphertext para change tracking

Incorrecto.

---

## 266.24 Comprobar unicidad sólo con SELECT previo

No constituye garantía concurrente.

---

## 266.25 Extension convirtiéndose en un ORM paralelo

Prohibido arquitectónicamente.

---

# 267. Modelo formal

Sea:

```text
E
```

una entidad administrada,

```text
M
```

el EntityManager,

```text
U
```

el UnitOfWork,

```text
I
```

la IdentityMap,

y:

```text
X
```

una ORM Extension.

Entonces:

```text
X(E)
```

deberá operar dentro de:

```text
M
→ U
→ I
→ Persistence Engine
```

y nunca crear autoridades paralelas:

```text
M'
U'
I'
PersistenceEngine'
```

para la misma unidad de persistencia.

---

# 268. Canonical Identity

Para:

```text
EntityType = T
Identifier = ID
DatabaseContext = C
```

dentro del mismo scope:

```text
load(T, ID, C)₁
===
load(T, ID, C)₂
```

deberá mantenerse incluso cuando intervengan ORM Extensions.

---

# 269. Extension Validity

```text
ValidOrmExtension
=
StableIdentity
∧
RegisteredExtensionPoints
∧
DependencyGraphValid
∧
NoConflicts
∧
MetadataValid
∧
MappingValid
∧
PersistenceInvariantsPreserved
∧
IdentityMapInvariantPreserved
∧
UnitOfWorkInvariantPreserved
∧
TransactionBoundariesPreserved
∧
SecurityBoundariesPreserved
```

---

# 270. Flush Extension Validity

Si:

```text
S₀
```

es el ChangeSet inicial y las extensiones producen:

```text
S₁ ... Sn
```

el pipeline deberá alcanzar:

```text
Sₙ = Stable
```

dentro de un número acotado de iteraciones.

Si:

```text
¬Stable
```

entonces:

```text
Flush
→ Abort
```

en lugar de ejecutar un persistence plan indeterminado.

---

# 271. Metadata Identity

La metadata efectiva podrá modelarse como:

```text
EffectiveMetadata(E)
=
CoreMetadata(E)
+
ValidatedExtensionContributions(E)
```

pero el resultado deberá convertirse en una única:

```text
CompiledEntityMetadata
```

canónica.

---

# 272. Extension Applicability

```text
Applicable(X, Entity)
=
Enabled(X)
∧
MatchesMetadata(X, Entity)
∧
DependenciesSatisfied(X)
∧
ContextPermits(X)
```

Por tanto:

```text
Installed
≠
Enabled
≠
Applicable
```

---

# 273. Cache Validity

Para metadata extendida:

```text
MetadataCacheValid
=
SameEntityMetadataFingerprint
∧
SameMappingGeneration
∧
SameRelevantOrmExtensionGeneration
```

---

# 274. Transaction Safety

Una extensión no podrá inferir:

```text
StatementSuccess
→ CommitSuccess
```

ni:

```text
postPersist
→ DurablePersistence
```

Formalmente:

```text
StatementSuccess
≠
TransactionCommit
```

---

# 275. Persistence Safety

```text
OrmExtension
→ Persistence Semantic Contribution
→ UoW/Persistence Planner
→ Query Model
→ Query Engine
```

Nunca:

```text
OrmExtension
→ Raw SQL
→ Driver
```

como flujo ordinario.

---

# 276. Arquitectura consolidada

```text
                        Application
                             │
             ┌───────────────┴────────────────┐
             ▼                                ▼
          Model API                      Repository API
             │                                │
             └───────────────┬────────────────┘
                             ▼
                       EntityManager
                             │
             ┌───────────────┼─────────────────┐
             ▼               ▼                 ▼
         Metadata        IdentityMap       UnitOfWork
             │                                 │
             │                         Change Tracking
             │                                 │
             └───────────────┬─────────────────┘
                             ▼
                    ORM Extension Plan
             ┌───────────────┼──────────────────┐
             ▼               ▼                  ▼
          Query           Hydration         Persistence
        Extensions       Extensions         Extensions
             │               │                  │
             └───────────────┼──────────────────┘
                             ▼
                    Persistence Engine
                             │
                             ▼
                       Query Engine
                             │
                             ▼
                        Compiler
                             │
                             ▼
                    Execution Engine
                             │
                             ▼
                         Connection
                             │
                             ▼
                           Driver
                             │
                             ▼
                            DBMS
```

Cross-cutting:

```text
Transactions
Cache
Events
Security
Telemetry
Multitenancy
Runtime
Testing
```

---

# 277. Estrategia V1

La V1 deberá estabilizar prioritariamente:

```text
OrmExtensionId
OrmExtensionDescriptor
OrmExtensionProvider
OrmExtensionRegistrar
OrmExtensionRegistry

EntityMetadataContributor
OrmMappingAttributeResolver

OrmQueryScope
RepositoryFactoryExtension
ModelApiExtension

ChangeTrackingExtension
UnitOfWorkExtensionContext

PersistenceExtension
HydrationExtension

CompiledOrmExtensionPlan

Dependency Resolution
Conflict Detection
Extension Fingerprints

Diagnostics
Persistent Runtime Isolation
Conformance Testing
```

Los puntos más invasivos, como:

```text
entirely new relationship semantics
custom entity states
complex flush mutation
```

deberán permanecer restringidos hasta que los contratos centrales estén suficientemente estabilizados.

---

# 278. Evolución posterior

La arquitectura podrá soportar posteriormente extensiones oficiales o de terceros para:

```text
Soft Deletes
Temporal Entities
Entity Versioning
Field Encryption
Automatic Auditing
Slugs
Immutable Entities
Domain Value Objects
Localized Fields
Custom Collections
Materialized Paths
Tree Models
Graph Relationships
Spatial Entities
Vector Embeddings
Event Sourcing Bridges
Search Index Synchronization
External Persistence Projections
```

sin convertir cada feature en una modificación directa del ORM Core.

---

# 279. Regla sobre Event Sourcing

Una integración futura con Event Sourcing deberá ser particularmente cuidadosa.

```text
Event Store
≠
VoltStack ORM UnitOfWork
```

Si se integran:

```text
explicit bridge
```

será obligatorio.

No se fingirá que ambos persistence models son el mismo.

---

# 280. Regla sobre External Search Indexes

Una extensión que sincronice:

```text
Elasticsearch
OpenSearch
Meilisearch
```

u otro sistema externo no deberá considerar `postPersist` como garantía suficiente de commit.

Preferido:

```text
Transaction
 ↓
Outbox
 ↓
Commit
 ↓
Async Consumer
 ↓
Search Index
```

---

# 281. Regla sobre External Storage

Misma consideración para:

```text
object storage
external API
message broker
search engine
analytics store
```

Los side effects externos deberán respetar transaction outcome.

---

# 282. Relación con la arquitectura de extensibilidad

La secuencia completa hasta este documento queda:

```text
294_DATABASE_EXTENSION_ARCHITECTURE
            │
            ▼
295_DATABASE_PLUGIN_SYSTEM
            │
            ├─────────────────────────────┐
            ▼                             │
296_DATABASE_CUSTOM_DRIVER_SYSTEM         │
            │                             │
            ▼                             │
297_DATABASE_CUSTOM_DIALECT_SYSTEM        │
            │                             │
            ▼                             │
298_DATABASE_CUSTOM_COMPILER_SYSTEM       │
            │                             │
            ▼                             │
299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM│
            │                             │
            ▼                             │
300_DATABASE_CUSTOM_ORM_EXTENSION_SYSTEM ◄┘
```

Las responsabilidades permanecen separadas:

```text
Extension Architecture
→ cómo extender Database

Plugin System
→ cómo distribuir/componer extensiones

Custom Driver
→ cómo hablar con otro backend

Custom Dialect
→ cómo representar su lenguaje SQL

Custom Compiler
→ cómo compilar semántica a comandos

Custom Query Extension
→ cómo ampliar Query Language

Custom ORM Extension
→ cómo ampliar comportamiento ORM
```

---

# 283. Regla final

> **La extensibilidad del ORM de VoltStack se basará en colaboración controlada con el ORM central, no en reemplazar silenciosamente sus autoridades internas. EntityManager, UnitOfWork, IdentityMap, Persistence Engine, Query Engine y Transaction System continuarán siendo las fuentes canónicas de sus respectivas responsabilidades.**

Por tanto:

```text
ORM Extension
≠
ORM Replacement
```

```text
ORM Extension
≠
Second EntityManager
```

```text
ORM Extension
≠
Second UnitOfWork
```

```text
ORM Extension
≠
Second IdentityMap
```

```text
ORM Extension
≠
Parallel Persistence Engine
```

```text
ORM Extension
≠
Query Extension
```

```text
ORM Extension
≠
Plugin
```

```text
Repository
≠
Persistence Engine
```

```text
Mapping
≠
Schema
```

```text
Mapping
≠
Type
```

```text
Change Tracking
≠
Persistence
```

```text
Hydration Extension
≠
Second Hydration Pipeline
```

```text
Lifecycle Hook
≠
Domain Event
```

```text
Lifecycle Hook
≠
Transaction Event
```

```text
postPersist
≠
afterCommit
```

```text
flush
≠
commit
```

```text
Database Rollback
≠
Object Graph Rewind
```

```text
Extension Cache
≠
IdentityMap
```

```text
Discovered
≠
Enabled
```

```text
Enabled
≠
Applicable
```

y finalmente:

```text
Safe Custom ORM Extension
=
Stable Identity
+
Explicit Extension Points
+
Canonical Metadata
+
Single EntityManager
+
Single UnitOfWork
+
Single IdentityMap
+
Controlled Change Tracking
+
Semantic Persistence Integration
+
Identity-safe Hydration
+
Relationship Invariants
+
Query Engine Integration
+
Transaction Awareness
+
Security Preservation
+
Cache Coherence
+
Deterministic Ordering
+
Conflict Detection
+
Persistent Runtime Isolation
+
Diagnostics
+
Conformance Testing
```

---

# 284. Siguiente documento

```text
301_DATABASE_DATABASE_CAPABILITY_DISCOVERY_SYSTEM.md
```

El siguiente documento cerrará el bloque de extensibilidad definiendo cómo VoltStack descubrirá, verificará, agregará y expondrá las capacidades efectivamente disponibles en un entorno Database.

Deberá cubrir:

```text
Database Capability Discovery System
│
├── Capability Discovery Architecture
├── Capability Identity
├── Discovery Sources
├── Static Platform Evidence
├── Server Version Evidence
├── Driver Evidence
├── Connection Evidence
├── Runtime Probes
├── Server Configuration Evidence
├── Installed Extension Detection
├── Schema-dependent Capabilities
├── Endpoint-specific Discovery
├── Capability Evidence
├── Evidence Confidence
├── Evidence Provenance
├── Capability Resolver
├── Conflicting Evidence
├── UNKNOWN Handling
├── Probe Safety
├── Probe Permissions
├── Probe Timeouts
├── Probe Caching
├── Capability Snapshot
├── Capability Fingerprinting
├── Topology Awareness
├── Replica Differences
├── Shard Differences
├── Persistent Runtime Behavior
├── Diagnostics
├── Telemetry
├── Security
├── Testing
└── Plugin Integration
```

manteniendo como regla fundamental:

> **VoltStack no inferirá una capacidad únicamente por el nombre o versión declarada del DBMS cuando exista evidencia más precisa disponible. El Discovery System recopilará evidencia; el Capability System decidirá soporte.**

Y preservando:

```text
Discovery
≠
Capability Decision
```

```text
Evidence
≠
Support
```

```text
Vendor + Version
≠
Capability
```

```text
Probe Failure
≠
Unsupported
```

```text
UNKNOWN
≠
UNSUPPORTED
```

```text
Endpoint A Capability
≠
Endpoint B Capability
```