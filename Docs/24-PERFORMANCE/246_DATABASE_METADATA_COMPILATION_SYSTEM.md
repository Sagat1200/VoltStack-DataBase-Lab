# 246_DATABASE_METADATA_COMPILATION_SYSTEM.md

# VoltStack Quantum Database
## Database Metadata Compilation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 246 — Database Metadata Compilation System  
**Bloque:** 24 — Performance  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md`  
**Siguiente documento:** `247_DATABASE_QUERY_COMPILATION_OPTIMIZATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Database Metadata Compilation System** de VoltStack.

Su responsabilidad es transformar metadata declarativa, flexible y relativamente costosa de interpretar en estructuras:

```text
validated
normalized
resolved
indexed
immutable
compiled
cacheable
runtime-efficient
```

listas para ser consumidas por los hot paths del Database Engine.

La transformación conceptual será:

```text
PHP Classes
Attributes
Configuration
Mapping Definitions
Custom Types
Relationships
Identifiers
Lifecycle Definitions
Inheritance
Repositories
Property Access
        │
        ▼
Metadata Discovery
        │
        ▼
Metadata Normalization
        │
        ▼
Metadata Validation
        │
        ▼
Symbol Resolution
        │
        ▼
Dependency Resolution
        │
        ▼
Metadata Compilation
        │
        ▼
Compiled Metadata Graph
        │
        ▼
Runtime Metadata Registry
```

Regla central:

> **VoltStack deberá pagar el costo de descubrir, interpretar, validar y resolver metadata principalmente durante bootstrap, warmup o compilación, y no repetidamente dentro de los hot paths de query building, hydration, dirty checking, relationship loading, flush y persistence.**

---

# 2. Problema que resuelve

Una arquitectura ORM declarativa puede depender de información como:

```php
#[Entity]
#[Table('users')]
class User
{
    #[Id]
    #[Column(type: 'uuid')]
    private UserId $id;

    #[Column]
    private string $name;

    #[ManyToOne(target: Company::class)]
    private Company $company;
}
```

Interpretar esta definición puede requerir:

```text
ReflectionClass
ReflectionProperty
attribute discovery
attribute instantiation
type inspection
mapping normalization
identifier discovery
relationship resolution
target entity resolution
table resolution
column resolution
type registry lookup
accessor creation
validation
dependency graph construction
```

Si estas operaciones se repiten por cada fila:

```text
10,000 rows
×
metadata interpretation
```

el diseño sería innecesariamente costoso.

VoltStack deberá convertirlo en algo conceptualmente similar a:

```text
CompiledEntityMetadata<User>

EntityTypeId: user
Table: users
Identifier: [id]
Fields:
    id   → users.id   → uuid → accessor #1
    name → users.name → string → accessor #2

Relationship:
    company
        target: company
        kind: MANY_TO_ONE
        FK: company_id
```

y reutilizar esa representación.

---

# 3. Metadata declarativa ≠ metadata compilada

Debe distinguirse:

```text
Source Metadata
≠
Normalized Metadata
≠
Validated Metadata
≠
Compiled Metadata
≠
Runtime State
```

---

# 4. Source Metadata

Representa la información obtenida directamente desde:

```text
PHP attributes
configuration
mapping files
extensions
conventions
explicit programmatic mapping
```

Puede ser incompleta.

Puede contener referencias simbólicas.

Puede necesitar interpretación.

---

# 5. Normalized Metadata

Convierte diferentes formas declarativas a un modelo canónico.

Ejemplo:

```text
#[Column(type: 'string')]
```

y una futura configuración equivalente:

```text
fields:
    name:
        type: string
```

deberán converger en una representación interna común.

---

# 6. Validated Metadata

Es metadata cuya estructura ha sido comprobada.

Ejemplo:

```text
Entity has identifier
Column names are valid
Relationship target exists
Mapped property exists
Type is registered
Inheritance configuration is coherent
```

---

# 7. Compiled Metadata

Es una representación optimizada para runtime.

Puede contener:

```text
resolved TypeIds
resolved EntityTypeIds
integer field indexes
compiled property readers
compiled property writers
identifier extractors
relationship descriptors
precomputed masks
dependency indexes
lifecycle callback dispatch tables
precomputed persistence instructions
```

---

# 8. Runtime State

Nunca deberá confundirse con metadata.

Ejemplos:

```text
Entity instance
IdentityMap
UnitOfWork
ChangeSet
Transaction
Connection
Tenant
ResultCursor
HydrationSession
```

no son metadata compilada.

---

# 9. Principio de compilación anticipada

VoltStack favorecerá:

```text
discover once
normalize once
validate once
resolve once
compile once
reuse many times
```

frente a:

```text
discover repeatedly
reflect repeatedly
resolve repeatedly
validate repeatedly
```

---

# 10. Objetivos

El sistema deberá:

- reducir Reflection en runtime;
- reducir parsing de Attributes;
- eliminar resolución repetitiva de mapping;
- pre-resolver relaciones;
- pre-resolver tipos;
- pre-resolver identificadores;
- precompilar accessors;
- generar índices eficientes;
- soportar metadata cache;
- soportar invalidación generacional;
- detectar metadata inválida tempranamente;
- soportar desarrollo y producción;
- funcionar correctamente en FrankenPHP;
- preparar compatibilidad con RoadRunner/OpenSwoole;
- permitir extensiones;
- preservar aislamiento entre requests;
- permitir diagnósticos;
- permitir warmup;
- permitir benchmarking.

---

# 11. No objetivos

El sistema no será responsable de:

```text
executing SQL
building SQL
executing migrations
hydrating entities
tracking changes
opening transactions
managing connections
query optimization
database introspection at every request
```

---

# 12. Arquitectura general

```text
                SOURCE CODE
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
      Attributes  Config   Extensions
          │         │         │
          └─────────┼─────────┘
                    ▼
             Metadata Discovery
                    │
                    ▼
             Source Metadata
                    │
                    ▼
             Normalization
                    │
                    ▼
           Canonical Metadata
                    │
                    ▼
              Validation
                    │
                    ▼
          Symbol Resolution
                    │
                    ▼
        Dependency Resolution
                    │
                    ▼
             Compilation
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    Accessors     Indexes      Masks
        │           │           │
        └───────────┼───────────┘
                    ▼
         Compiled Metadata Graph
                    │
                    ▼
         Runtime Metadata Registry
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
     Query        Hydration   Persistence
```

---

# 13. MetadataCompiler

Contrato principal:

```php
interface MetadataCompiler
{
    public function compile(
        MetadataCompilationInput $input,
        MetadataCompilationContext $context,
    ): CompiledMetadataSet;
}
```

---

# 14. MetadataCompilationInput

```php
final readonly class MetadataCompilationInput
{
    /**
     * @param list<EntitySourceDefinition> $entities
     */
    public function __construct(
        public array $entities,
        public MetadataSourceGeneration $generation,
    ) {}
}
```

---

# 15. Compilation Context

```php
final readonly class MetadataCompilationContext
{
    public function __construct(
        public TypeRegistry $types,
        public MetadataCompilationPolicy $policy,
        public MetadataExtensionRegistry $extensions,
        public PlatformCapabilitySet $capabilities,
    ) {}
}
```

---

# 16. Context ≠ request context

`MetadataCompilationContext` contendrá información estructural.

No:

```text
current request
current user
current transaction
current entity
current tenant mutable state
```

---

# 17. Pipeline

El pipeline oficial será:

```text
DISCOVER
   ↓
NORMALIZE
   ↓
VALIDATE_LOCAL
   ↓
REGISTER_SYMBOLS
   ↓
RESOLVE_REFERENCES
   ↓
VALIDATE_GRAPH
   ↓
OPTIMIZE
   ↓
COMPILE
   ↓
FREEZE
   ↓
PUBLISH
```

---

# 18. Discover

Obtiene metadata fuente.

Ejemplo:

```text
User.php
    ↓
ReflectionClass
    ↓
Attributes
    ↓
EntitySourceDefinition
```

---

# 19. Normalize

Convierte distintas formas de declaración a:

```text
CanonicalEntityDefinition
```

---

# 20. Local validation

Valida propiedades que no requieren conocer otras entidades.

Ejemplo:

```text
duplicate fields
invalid column name
missing type
invalid identifier declaration
```

---

# 21. Symbol registration

Antes de resolver relaciones se registrarán símbolos como:

```text
EntityTypeId
EmbeddableTypeId
TypeId
RelationshipId
```

---

# 22. Reference resolution

Después:

```text
User.company
    target Company::class
```

se convierte en:

```text
EntityTypeId(company)
```

---

# 23. Graph validation

Una vez resueltas referencias pueden comprobarse:

```text
relationship consistency
mappedBy/inversedBy
inheritance graphs
embedded cycles
cascade graphs
identifier dependencies
```

---

# 24. Optimization

Podrán prepararse estructuras runtime eficientes.

---

# 25. Compile

Produce metadata final.

---

# 26. Freeze

Después de compilada:

```text
CompiledMetadata
```

será inmutable.

---

# 27. Publish

Solo metadata completamente compilada y válida deberá publicarse al runtime registry.

---

# 28. No partial publication

Nunca:

```text
compile Entity A
publish A
compile Entity B
fail
```

dejando registry parcialmente actualizado.

---

# 29. Atomic publication

Conceptualmente:

```text
OldGeneration
      │
      ▼
Compile NewGeneration
      │
      ├── FAIL → discard
      │
      └── SUCCESS
              ↓
       publish atomically
              ↓
       NewGeneration
```

---

# 30. Metadata generations

Cada conjunto compilado tendrá:

```php
final readonly class MetadataGeneration
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 31. Generation purpose

Permitirá saber si:

```text
HydrationPlan
QueryPlan
PersistencePlan
MetadataCache
CompiledAccessor
```

fue generado contra metadata compatible.

---

# 32. Generation ≠ application version

```text
MetadataGeneration
≠
ApplicationVersion
```

aunque puedan estar correlacionadas.

---

# 33. EntityTypeId

No se recomienda usar FQCN como identidad interna universal.

Se utilizará:

```php
final readonly class EntityTypeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 34. Stable IDs

Ejemplo:

```text
user
company
invoice
invoice_line
```

---

# 35. FQCN mapping

Registry:

```text
App\Domain\User
        ↕
EntityTypeId(user)
```

---

# 36. Why

Esto desacopla:

```text
persistent metadata identity
```

de:

```text
PHP implementation name
```

cuando corresponda.

---

# 37. FieldId

Cada campo podrá recibir:

```php
final readonly class FieldId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 38. Field index

Además podrá recibir un índice compacto:

```text
id         → 0
name       → 1
email      → 2
createdAt  → 3
```

---

# 39. Index usage

Permitirá:

```text
bit masks
snapshot arrays
dirty masks
loaded masks
compiled assignments
```

---

# 40. FieldId ≠ FieldIndex

`FieldId` será identidad lógica.

`FieldIndex` será optimización interna.

---

# 41. Stable logical identity

El índice podría cambiar entre generaciones.

Por tanto no deberá persistirse como identidad durable.

---

# 42. CompiledEntityMetadata

```php
final readonly class CompiledEntityMetadata
{
    /**
     * @param array<int, CompiledFieldMetadata> $fields
     * @param array<string, int> $fieldNameIndex
     * @param array<int, CompiledRelationshipMetadata> $relationships
     */
    public function __construct(
        public EntityTypeId $entityType,
        public string $className,
        public CompiledTableMetadata $table,
        public CompiledIdentifierMetadata $identifier,
        public array $fields,
        public array $fieldNameIndex,
        public array $relationships,
        public CompiledLifecycleMetadata $lifecycle,
        public MetadataGeneration $generation,
    ) {}
}
```

---

# 43. Immutable metadata

Después de construcción:

```text
CompiledEntityMetadata
```

no podrá mutarse.

---

# 44. Why immutable

Permite:

```text
safe sharing
predictable caches
persistent runtime reuse
lock-free reads
stable plans
```

---

# 45. Field metadata

```php
final readonly class CompiledFieldMetadata
{
    public function __construct(
        public FieldId $id,
        public int $index,
        public string $property,
        public string $column,
        public TypeId $type,
        public bool $nullable,
        public CompiledPropertyReader $reader,
        public CompiledPropertyWriter $writer,
    ) {}
}
```

---

# 46. Runtime field lookup

No:

```text
foreach metadata fields:
    if name === requested
```

Preferido:

```text
fieldNameIndex[$name]
```

---

# 47. Expected lookup

Promedio:

```text
O(1)
```

---

# 48. Property readers

Contrato:

```php
interface CompiledPropertyReader
{
    public function read(object $entity): mixed;
}
```

---

# 49. Property writers

```php
interface CompiledPropertyWriter
{
    public function write(
        object $entity,
        mixed $value,
    ): void;
}
```

---

# 50. Accessor strategies

Podrán existir:

```text
DIRECT
CLOSURE
REFLECTION
GENERATED
CUSTOM
```

---

# 51. Reflection fallback

Reflection seguirá siendo válida como fallback.

Pero deberá prepararse una sola vez cuando sea posible.

---

# 52. ReflectionProperty reuse

No:

```php
new ReflectionProperty(...)
```

por cada acceso.

Puede preconstruirse.

---

# 53. Closure binding

Podrá evaluarse para acceso rápido a propiedades privadas.

---

# 54. Generated accessors

Podrán introducirse posteriormente.

Ejemplo conceptual:

```php
static function (User $entity): string {
    return $entity->name;
}
```

---

# 55. Generated code security

Si se genera código:

```text
metadata input
→
code generation
```

deberá existir validación estricta.

Nunca se interpolarán valores no confiables como PHP arbitrario.

---

# 56. eval()

El uso indiscriminado de:

```php
eval()
```

no será arquitectura base.

---

# 57. Generated files

Una estrategia futura podría generar:

```text
var/cache/voltstack/database/metadata/
```

con clases PHP compiladas.

---

# 58. Production optimization

Ejemplo:

```text
metadata/
├── Entity_User_Metadata.php
├── Entity_Company_Metadata.php
└── metadata_manifest.php
```

---

# 59. OPcache

Los archivos PHP compilados podrán beneficiarse de:

```text
PHP OPcache
```

sin convertir OPcache en dependencia conceptual del sistema.

---

# 60. Attribute discovery

Attributes serán una fuente importante de metadata.

Ejemplo:

```php
#[Entity]
#[Table('users')]
final class User
{
}
```

---

# 61. Attribute instantiation

No deberá repetirse dentro de hot paths.

---

# 62. Discovery cache

El resultado de discovery podrá almacenarse temporalmente durante compilación.

---

# 63. Attribute constructors

Attributes de mapping deberán ser:

```text
declarative
side-effect-free
deterministic
```

---

# 64. Anti-pattern

Un attribute no debería:

```text
connect to database
call HTTP API
read current request
resolve current user
```

---

# 65. Metadata determinism

Dadas las mismas:

```text
source definitions
configuration
extension versions
```

la metadata compilada deberá ser determinista.

---

# 66. Deterministic ordering

Entidades y campos deberán ordenarse canónicamente antes de producir fingerprints.

---

# 67. Metadata fingerprint

```php
final readonly class MetadataFingerprint
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 68. Fingerprint input

Puede incluir:

```text
entity definitions
field mappings
relationships
types
inheritance
lifecycle callbacks
extension contributions
```

---

# 69. Fingerprint ≠ source file mtime

`mtime` puede ayudar en development invalidation.

No será identidad semántica suficiente.

---

# 70. Source fingerprint

Podrá existir separadamente:

```text
SourceFingerprint
```

---

# 71. Semantic fingerprint

Representará el significado normalizado.

---

# 72. Cache architecture

```text
Source
  ↓
SourceFingerprint
  ↓
Metadata Cache Lookup
  ├── HIT
  │    ↓
  │ CompiledMetadataSet
  │
  └── MISS
       ↓
    Compile
       ↓
    Store
```

---

# 73. Metadata Cache

Ya definido como categoría independiente:

```text
Metadata Cache
≠
Entity Cache
≠
Result Cache
≠
Query Cache
≠
IdentityMap
```

---

# 74. Cache contents

Podrá almacenar:

```text
normalized metadata
compiled metadata
accessor descriptors
dependency graph
fingerprints
```

---

# 75. Cache safety

No almacenará:

```text
entity instances
connections
transactions
IdentityMaps
UnitOfWork
request context
```

---

# 76. Serialization

Compiled metadata deberá diseñarse pensando en serialización/cache cuando sea razonable.

---

# 77. Closures complication

Closures no siempre son fácilmente serializables.

Por tanto podrá separarse:

```text
SerializableCompiledMetadata
+
RuntimeAccessorBinding
```

---

# 78. Two-phase compilation

Una estrategia posible:

```text
Structural Compilation
        ↓
Serializable Metadata
        ↓
Runtime Binding
        ↓
Bound Compiled Metadata
```

---

# 79. Structural compilation

Resuelve:

```text
entities
fields
types
relationships
identifiers
indexes
masks
```

---

# 80. Runtime binding

Prepara:

```text
ReflectionProperty
closures
generated callables
runtime handlers
```

---

# 81. Runtime binding ≠ rediscovery

No deberá volver a interpretar attributes.

---

# 82. Type Registry integration

Cada field:

```text
type: "uuid"
```

se resolverá a:

```text
TypeId(uuid)
```

y después a:

```text
TypeHandler
```

cuando corresponda.

---

# 83. String lookup reduction

Hot paths no deberían hacer repetidamente:

```php
$typeRegistry->get('uuid');
```

---

# 84. Pre-bound type handler

Metadata compilada podrá contener:

```text
CompiledTypeBinding
```

---

# 85. Platform-specific type behavior

Debe evitarse mezclar metadata ORM lógica con SQL físico innecesariamente.

---

# 86. Logical metadata

Debe describir:

```text
UUID
string
datetime
json
money
```

---

# 87. Physical mapping

La resolución:

```text
UUID → PostgreSQL UUID
UUID → MySQL CHAR/BINARY
```

pertenece a Platform/Schema/Compiler según contexto.

---

# 88. Metadata compilation ≠ SQL compilation

Regla:

```text
Metadata Compilation
≠
SQL Compilation
```

---

# 89. Identifier metadata

```php
final readonly class CompiledIdentifierMetadata
{
    /**
     * @param non-empty-list<int> $fieldIndexes
     */
    public function __construct(
        public array $fieldIndexes,
        public bool $composite,
        public IdentifierGenerationStrategy $generationStrategy,
    ) {}
}
```

---

# 90. Identifier extractor

Podrá precompilarse:

```php
interface CompiledIdentifierExtractor
{
    public function fromEntity(object $entity): EntityIdentifier;

    public function fromRow(RowView $row): EntityIdentifier;
}
```

---

# 91. Composite IDs

Ejemplo:

```text
tenant_id
+
invoice_number
```

podrá tener un extractor especializado.

---

# 92. Identifier canonicalization

La compilación podrá pre-resolver:

```text
field converters
ordering
canonical representation
```

---

# 93. IdentityMap integration

IdentityMap podrá recibir directamente:

```text
EntityTypeId
EntityIdentifier
```

sin reinterpretar metadata.

---

# 94. Relationship metadata

```php
final readonly class CompiledRelationshipMetadata
{
    public function __construct(
        public RelationshipId $id,
        public RelationshipKind $kind,
        public EntityTypeId $owner,
        public EntityTypeId $target,
        public bool $owningSide,
        public CompiledJoinMapping $join,
        public RelationshipFetchPolicy $fetch,
        public CascadePolicy $cascade,
    ) {}
}
```

---

# 95. Relationship resolution

Antes de runtime:

```text
User.company
target: Company::class
```

deberá estar convertido a:

```text
EntityTypeId(company)
```

---

# 96. mappedBy

También:

```text
Company.users
mappedBy: company
```

se resolverá a un `RelationshipId` válido.

---

# 97. No string navigation hot path

Evitar:

```text
find target entity by class string
find relationship by property string
find inverse relation by scanning
```

durante hydration/persistence.

---

# 98. Relationship indexes

Podrán existir:

```text
relationshipByProperty
relationshipById
relationshipsByTarget
owningRelationships
inverseRelationships
```

---

# 99. Cascade graph

Cascade metadata podrá compilarse a estructuras eficientes.

---

# 100. Cascade ≠ execution

La metadata describe:

```text
cascade persist
cascade remove
```

pero no ejecuta persist/remove.

---

# 101. Relationship dependency graph

Persistence Planner podrá consumir un grafo precomputado.

---

# 102. Dependency edges

Ejemplo:

```text
Order
  └── customer → Customer
```

puede representar dependencia para ciertas operaciones.

---

# 103. Static dependency ≠ runtime graph

Metadata graph:

```text
EntityType → EntityType
```

Runtime graph:

```text
EntityInstance → EntityInstance
```

son distintos.

---

# 104. Embedded metadata

Value Objects embebidos podrán compilarse.

Ejemplo:

```php
#[Embedded]
private Address $address;
```

---

# 105. Flattened field indexes

Podría producir:

```text
address.street → field 4
address.city   → field 5
address.zip    → field 6
```

---

# 106. Embedded accessor plan

El compilador puede preparar:

```text
read Address
read street
```

o un accessor especializado.

---

# 107. Nested object creation

Hydration metadata podrá conocer cómo construir:

```text
Address
```

sin redescubrir su mapping.

---

# 108. Value Object metadata

Deberá distinguirse:

```text
EntityMetadata
≠
ValueObjectMetadata
```

---

# 109. Inheritance metadata

Si VoltStack soporta herencia ORM:

```text
SINGLE_TABLE
JOINED
TABLE_PER_CLASS
```

o subconjunto explícito, deberá compilarse.

---

# 110. Discriminator metadata

```php
final readonly class CompiledDiscriminatorMap
{
    /**
     * @param array<string, EntityTypeId> $map
     */
    public function __construct(
        public array $map,
    ) {}
}
```

---

# 111. Safe discriminator

Valores persistidos:

```text
user
admin
customer
```

no deberán interpretarse como FQCN arbitrario.

---

# 112. Reverse discriminator map

También podrá existir:

```text
EntityTypeId
→
discriminator
```

para persistence.

---

# 113. Lifecycle metadata

Callbacks:

```text
prePersist
postPersist
preUpdate
postUpdate
preRemove
postRemove
postLoad
```

podrán compilarse.

---

# 114. Callback dispatch

No deberá hacerse:

```text
scan all methods
for every lifecycle event
```

---

# 115. Dispatch table

Preferido:

```text
POST_LOAD
    → callback #1
    → callback #2
```

---

# 116. Callback binding

Puede prepararse:

```text
method
callable
priority
extension source
```

---

# 117. Lifecycle callback ≠ Database Event

Debe mantenerse:

```text
Entity Lifecycle Callback
≠
Database Event System Event
```

aunque puedan integrarse.

---

# 118. Repository metadata

Una entidad podrá declarar:

```php
#[Entity(repository: UserRepository::class)]
```

---

# 119. Repository resolution

Se validará:

```text
repository exists
repository contract compatible
entity type compatible
```

antes de runtime cuando sea posible.

---

# 120. Repository instance ≠ metadata

Metadata puede contener:

```text
repository class descriptor
```

No necesariamente una instancia mutable global.

---

# 121. Model API metadata

Para Active Record-style API podrán precompilarse:

```text
entity type
identifier
default table
fillable/mapped fields where applicable
casts
relationships
```

sin crear un segundo ORM.

---

# 122. Dual API

Continúa:

```text
Model API
      │
      ▼
Compiled ORM Metadata
      ▲
      │
Repository / EntityManager
```

---

# 123. Single metadata engine

No existirán:

```text
ModelMetadataEngine
```

y:

```text
EntityManagerMetadataEngine
```

incompatibles.

---

# 124. Active Record metadata

Será una vista/fachada sobre metadata canónica.

---

# 125. Casting metadata

Casts declarativos podrán pre-resolverse.

---

# 126. Cast ≠ Type

Continúa vigente:

```text
Cast
≠
ORM Type
```

---

# 127. Compiled cast chain

Ejemplo:

```text
DB raw
 ↓
Type Converter
 ↓
Canonical Persistent Value
 ↓
Cast
 ↓
Application Representation
```

---

# 128. Cast handlers

Podrán pre-bindearse.

---

# 129. Query metadata

Query Engine puede necesitar:

```text
property → column
entity → table
relationship → join mapping
field → logical type
```

---

# 130. Query Builder hot path

En vez de:

```text
reflect User
find field
find column
resolve type
```

hará:

```text
metadata.field("email")
```

sobre índice compilado.

---

# 131. Semantic Query Engine

Podrá consumir metadata para:

```text
symbol resolution
type inference
relationship resolution
constraint analysis
```

---

# 132. Metadata compilation advantage

Mucho de ese conocimiento ya estará:

```text
validated
indexed
resolved
```

---

# 133. Hydration integration

HydrationPlan podrá construirse desde:

```text
CompiledEntityMetadata
```

---

# 134. Property writer reuse

No será necesario recrear property writers para cada query.

---

# 135. Persistence integration

Persistence Engine podrá obtener:

```text
insertable fields
updatable fields
identifier fields
generated fields
version field
relationship ownership
```

precalculados.

---

# 136. Insert field set

Metadata podrá tener:

```text
insertFieldIndexes
```

---

# 137. Update field set

También:

```text
updateFieldIndexes
```

---

# 138. Dirty checking integration

Snapshot arrays podrán usar `FieldIndex`.

Ejemplo:

```text
0 → id
1 → name
2 → email
3 → status
```

---

# 139. Dirty mask

```text
0010
```

podría indicar:

```text
email dirty
```

dependiendo del mapping.

---

# 140. Loaded mask

La misma indexación estructural puede servir a:

```text
LoadedFieldMask
```

---

# 141. Shared indexing contract

Debe existir una fuente única para `FieldIndex`.

---

# 142. Field index generation

Se asignará durante metadata compilation.

---

# 143. Deterministic indexes

Dadas las mismas definiciones canónicas, los índices deberán ser deterministas dentro de una generación.

---

# 144. Index persistence caution

No se usarán como identificadores durables externos.

---

# 145. Schema integration

ORM metadata podrá contribuir a generar:

```text
Schema Model
```

si VoltStack soporta schema derivation.

---

# 146. Mapping ≠ Schema

Continúa:

```text
ORM Mapping
≠
Database Schema
```

---

# 147. Why

Una columna ORM puede no describir todos los detalles físicos:

```text
indexes
partitioning
storage parameters
database-specific constraints
```

---

# 148. Schema derivation

Será una transformación explícita:

```text
ORM Metadata
    ↓
Schema Derivation
    ↓
Schema Model
```

no equivalencia.

---

# 149. Validation levels

Podrán existir:

```php
enum MetadataValidationLevel
{
    case BASIC;
    case STANDARD;
    case STRICT;
}
```

---

# 150. BASIC

Comprueba estructura esencial.

---

# 151. STANDARD

Comprueba relaciones, tipos y consistencia normal.

---

# 152. STRICT

Puede detectar:

```text
suspicious mappings
unused inverse definitions
ambiguous mappings
unsafe nullable assumptions
unsupported platform semantics
```

---

# 153. Validation ≠ DB introspection

Validar metadata no requerirá necesariamente conectarse a DB.

---

# 154. Mapping/schema validation

Comparar metadata contra base real será operación distinta.

---

# 155. Offline compilation

Debe ser posible:

```text
composer install
       ↓
volt database:metadata:compile
```

sin conexión DB cuando el mapping no la requiera.

---

# 156. CLI

Comandos potenciales:

```text
volt database:metadata:compile
volt database:metadata:warm
volt database:metadata:clear
volt database:metadata:validate
volt database:metadata:inspect
volt database:metadata:graph
```

---

# 157. Compile command

```text
$ volt database:metadata:compile
```

podría mostrar:

```text
Discovering entities........ 142
Normalizing metadata........ done
Validating metadata......... done
Resolving relationships..... done
Compiling accessors......... done
Writing cache............... done

Generation:
    md_9f8a2c...

Compiled entities:
    142

Relationships:
    387

Fields:
    2,614

Duration:
    218 ms
```

---

# 158. Validate command

Podrá detectar:

```text
User.company references unknown Company entity.
```

antes de atender tráfico.

---

# 159. Inspect command

```text
$ volt database:metadata:inspect User
```

---

# 160. Example output

```text
Entity
    user

Class
    App\Domain\User

Table
    users

Identifier
    id : uuid

Fields
    id
    name
    email
    createdAt

Relationships
    company    MANY_TO_ONE
    orders     ONE_TO_MANY

Lifecycle
    POST_LOAD  1 callback

Generation
    md_9f8a2c...
```

---

# 161. Development mode

En desarrollo puede favorecerse:

```text
automatic invalidation
rapid recompilation
rich diagnostics
```

---

# 162. Production mode

En producción:

```text
precompiled metadata
immutable registry
no filesystem scanning per request
minimal reflection
```

---

# 163. Dev ≠ production behavior

Las optimizaciones no deberán cambiar semántica.

Solo estrategia de obtención/compilación.

---

# 164. Auto-reload

Development podrá detectar cambios.

---

# 165. Change detection

Puede utilizar:

```text
file mtimes
manifest
source hashes
composer classmap changes
config fingerprints
```

---

# 166. Production auto-reload

No será necesario por defecto.

Deploy deberá generar nueva metadata.

---

# 167. Cache manifest

```php
final readonly class MetadataCacheManifest
{
    public function __construct(
        public MetadataGeneration $generation,
        public MetadataFingerprint $fingerprint,
        public string $frameworkVersion,
        public string $phpVersion,
        public array $extensions,
    ) {}
}
```

---

# 168. PHP compatibility

Metadata compilada puede depender de:

```text
PHP major/minor
framework version
metadata compiler version
```

---

# 169. Compiler version

Deberá existir:

```text
MetadataCompilerVersion
```

---

# 170. Cache compatibility

Un cache será válido solo si sus requisitos son compatibles.

---

# 171. Cache invalidation

Debe ocurrir ante cambios relevantes como:

```text
entity mapping
type definitions
relationship mapping
compiler version
extension contributions
relevant config
```

---

# 172. Cache invalidation ≠ FLUSHALL

Nunca:

```text
Redis FLUSHALL
```

como mecanismo normal.

---

# 173. Namespace cache

Las claves deberán estar namespaced.

---

# 174. Metadata cache key

Conceptualmente:

```text
voltstack
database
metadata
compiler-v3
fingerprint-X
```

---

# 175. Cache corruption

Si cache está corrupto:

```text
detect
reject
recompile if policy allows
```

---

# 176. Corrupt cache ≠ valid empty metadata

Nunca se tratará corrupción como:

```text
0 entities
```

---

# 177. Compilation errors

Jerarquía:

```text
DatabaseMetadataCompilationException
├── MetadataDiscoveryException
├── MetadataNormalizationException
├── MetadataValidationException
├── MetadataSymbolResolutionException
├── MetadataRelationshipResolutionException
├── MetadataTypeResolutionException
├── MetadataAccessorCompilationException
├── MetadataCacheException
└── MetadataPublicationException
```

---

# 178. Error location

Un error deberá identificar:

```text
entity
field/relationship
source class
source file when safe
mapping origin
reason
```

---

# 179. Example diagnostic

```text
Metadata compilation failed.

Entity:
    App\Domain\Order

Relationship:
    customer

Target:
    App\Domain\Customer

Problem:
    Target class exists but is not registered as an Entity.

Source:
    Order::$customer
```

---

# 180. Suggestions

Podrá incluir:

```text
Did you forget #[Entity] on App\Domain\Customer?
```

cuando exista evidencia suficiente.

---

# 181. No speculative fixes

No deberá inventar soluciones sin evidencia.

---

# 182. Duplicate EntityTypeId

Dos clases no podrán registrar:

```text
EntityTypeId(user)
```

simultáneamente salvo mecanismo explícito de override/extensión.

---

# 183. Duplicate table mapping

Dos entidades podrían mapear la misma tabla en casos especiales.

No deberá rechazarse universalmente sin analizar estrategia.

---

# 184. Duplicate column

Dentro de una entidad, dos campos que escriben la misma columna pueden ser ambiguos.

Debe requerir semántica explícita.

---

# 185. Unknown type

```text
type: money
```

sin `TypeId(money)` registrado:

```text
MetadataTypeResolutionException
```

---

# 186. Relationship target resolution

Target desconocido:

```text
MetadataRelationshipResolutionException
```

---

# 187. mappedBy validation

Si:

```text
Company.users mappedBy company
```

pero `User.company` no existe:

```text
compile error
```

---

# 188. Relationship kind consistency

Si ambos lados declaran ownership incompatible:

```text
compile error
```

---

# 189. Cascade validation

Cascades incompatibles podrán producir warning/error según policy.

---

# 190. Identifier validation

Una managed entity normalmente requerirá identificador válido.

---

# 191. Keyless result model

Si VoltStack soporta read models sin ID:

```text
KeylessProjection
```

deberá ser tipo distinto, no managed entity normal.

---

# 192. Version field

Optimistic locking metadata deberá pre-resolverse.

---

# 193. CompiledVersionMetadata

```php
final readonly class CompiledVersionMetadata
{
    public function __construct(
        public int $fieldIndex,
        public TypeId $type,
        public VersionStrategy $strategy,
    ) {}
}
```

---

# 194. Soft delete metadata

Si el sistema Soft Delete está instalado:

```text
deletedAt
```

podrá contribuir metadata mediante extensión.

---

# 195. Extension contribution

El core compiler no deberá conocer todos los módulos opcionales.

---

# 196. Metadata extensions

```php
interface MetadataCompilerExtension
{
    public function contribute(
        MetadataCompilationGraph $graph,
        MetadataCompilationContext $context,
    ): void;
}
```

---

# 197. Extension phases

Una extensión podrá declarar fase:

```text
AFTER_DISCOVERY
AFTER_NORMALIZATION
AFTER_SYMBOL_REGISTRATION
AFTER_RESOLUTION
BEFORE_FREEZE
```

---

# 198. Extension ordering

Debe ser determinista.

---

# 199. Extension dependencies

Ejemplo:

```text
SoftDeleteMetadataExtension
requires
ORMCoreMetadataExtension
```

---

# 200. Circular extension dependency

Deberá detectarse.

---

# 201. Extension mutation

Después de `FREEZE`:

```text
no mutation
```

---

# 202. Extension isolation

Una extensión no deberá modificar metadata de forma arbitraria sin contratos.

---

# 203. Compilation graph

Durante compilación puede existir un grafo mutable controlado:

```text
MetadataCompilationGraph
```

---

# 204. Compilation graph ≠ runtime registry

El primero:

```text
mutable during compile
```

El segundo:

```text
immutable after publish
```

---

# 205. RuntimeMetadataRegistry

```php
interface RuntimeMetadataRegistry
{
    public function entity(
        EntityTypeId $id,
    ): CompiledEntityMetadata;

    public function entityForClass(
        string $class,
    ): CompiledEntityMetadata;

    public function generation(): MetadataGeneration;
}
```

---

# 206. Registry lookup

Deberá ser rápido.

---

# 207. Class lookup

Puede mantener:

```text
FQCN → EntityTypeId
```

---

# 208. Table lookup

Cuando sea necesario:

```text
LogicalTableId → EntityTypeId(s)
```

---

# 209. Relationship lookup

```text
RelationshipId → CompiledRelationshipMetadata
```

---

# 210. Runtime registry immutability

No:

```php
$registry->addEntity(...)
```

durante request normal.

---

# 211. Dynamic runtime metadata

No será estrategia default.

---

# 212. Why

Metadata dinámica durante requests dificulta:

```text
cache validity
query plan validity
hydration plan validity
persistent runtime safety
concurrency
debugging
```

---

# 213. Dynamic modules

Si aplicaciones necesitan módulos dinámicos, deberán generar una nueva:

```text
MetadataGeneration
```

---

# 214. Atomic registry swap

Persistent runtime podría hacer:

```text
Generation A
    ↓
compile B separately
    ↓
atomic publish B
```

solo en mecanismos explícitos.

---

# 215. In-flight request

Un request iniciado con:

```text
Generation A
```

deberá continuar coherentemente con A.

---

# 216. Generation pinning

Contextos podrán capturar:

```text
MetadataGeneration
```

al iniciar operación.

---

# 217. No mixed generation

Nunca:

```text
QueryPlan from A
HydrationPlan from B
PersistenceMetadata from C
```

dentro de una operación.

---

# 218. Generation mismatch

Debe producir:

```text
MetadataGenerationMismatchException
```

o invalidar/recompilar plan antes de ejecutar.

---

# 219. Persistent runtimes

Especial importancia para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 220. Shared immutable metadata

Podrá mantenerse:

```text
worker lifetime
```

---

# 221. Request-local metadata state

No deberá existir salvo wrappers/contextos de generación.

---

# 222. FrankenPHP

Arquitectura esperada:

```text
Worker Start
    │
    ▼
Load Compiled Metadata
    │
    ▼
Freeze Registry
    │
    ├── Request 1 → read-only use
    ├── Request 2 → read-only use
    ├── Request 3 → read-only use
    └── ...
```

---

# 223. Benefits

Reduce:

```text
filesystem scanning
reflection
attribute parsing
mapping resolution
relationship validation
```

por request.

---

# 224. Worker leak prevention

Registry no deberá almacenar referencias a:

```text
EntityManager
request
response
current tenant
current user
transaction
entity instances
```

---

# 225. Tenant metadata

Multitenancy puede tener varios modelos.

---

# 226. Shared-schema tenancy

Si todos comparten mapping:

```text
one compiled metadata generation
```

puede reutilizarse.

---

# 227. Schema-per-tenant

Si estructura lógica es igual:

```text
Logical Metadata
```

puede compartirse mientras:

```text
physical schema resolution
```

permanezca contextual.

---

# 228. Avoid tenant explosion

No deberá compilarse automáticamente una copia completa por tenant si solo cambia:

```text
schema name
database name
```

---

# 229. Tenant-specific mappings

Si realmente cambia estructura:

```text
MetadataVariantId
```

podrá representar variante.

---

# 230. Variant explosion protection

Debe existir gobernanza para impedir:

```text
100,000 tenants
×
full metadata graph
```

en memoria.

---

# 231. MetadataVariant

```php
final readonly class MetadataVariantId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 232. Variant ≠ TenantId

Una variante puede ser compartida por múltiples tenants.

---

# 233. Sharding

Shard físico tampoco deberá generar metadata ORM distinta si la estructura lógica es igual.

---

# 234. Metadata and capabilities

Ciertas features pueden depender de capacidades.

Pero:

```text
Entity Mapping
```

no deberá contaminarse con vendor checks.

---

# 235. Capability binding

Puede existir una etapa posterior:

```text
Compiled Logical Metadata
        +
Platform Capabilities
        ↓
Platform Binding
```

cuando sea necesario.

---

# 236. MySQL ≠ MariaDB

Si existen diferencias relevantes:

```text
Platform Capability System
```

las expresará.

No:

```php
if ($db === 'mysql') { ... }
```

disperso en metadata.

---

# 237. Security

Metadata puede revelar:

```text
table names
column names
class names
relationships
sensitive field classifications
```

---

# 238. Metadata debug access

En producción deberá controlarse.

---

# 239. Sensitive metadata

Campos podrán marcarse:

```text
SENSITIVE
SECRET
PII
CREDENTIAL
```

mediante integración con Sensitive Data Protection.

---

# 240. Classification compilation

Clasificaciones podrán precompilarse a máscaras.

---

# 241. Sensitive field mask

Ejemplo:

```text
FieldIndex:
0 id
1 email
2 passwordHash
3 apiToken
```

Mask:

```text
0011
```

según convención interna.

---

# 242. Redaction integration

Telemetry/Debug/Audit podrá consultar metadata compilada para redaction.

---

# 243. Security classification ≠ authorization

Un campo marcado sensitive no define por sí mismo quién puede leerlo.

---

# 244. Authorization integration

Data Access Security podrá usar metadata como evidencia estructural.

---

# 245. Metadata provenance

Cada definición podrá conservar:

```text
origin
```

para debugging.

---

# 246. Provenance examples

```text
PHP_ATTRIBUTE
CONFIG_FILE
EXTENSION
CONVENTION
GENERATED
```

---

# 247. Provenance in hot path

No es necesario mantener información pesada en estructuras críticas si existe un descriptor debug separado.

---

# 248. Split runtime/debug metadata

Podrá existir:

```text
CompiledRuntimeMetadata
```

y:

```text
MetadataDiagnosticMap
```

---

# 249. Production trimming

Información de diagnóstico muy pesada podrá omitirse en builds optimizados.

---

# 250. Correctness diagnostics

La información necesaria para errores seguros deberá mantenerse.

---

# 251. Memory model

Metadata compilada consumirá memoria aproximadamente según:

```text
Mmetadata
=
Mentities
+
Mfields
+
Mrelationships
+
Mindexes
+
Maccessors
+
Mtypes
+
Mlifecycle
+
Mdiagnostics
```

---

# 252. Avoid duplication

No almacenar repetidamente:

```text
same class string
same Type object
same relationship descriptor
same immutable handler
```

si puede compartirse de forma segura.

---

# 253. String interning

VoltStack no implementará necesariamente un string interning propio en V1.

Primero se medirán costos reales.

---

# 254. Compact indexes

Los índices enteros serán útiles especialmente para:

```text
fields
relationships
callbacks
```

---

# 255. Array tradeoffs

En PHP:

```text
arrays
```

pueden ser costosos.

Se evaluarán estructuras compactas donde aporten beneficio.

---

# 256. SplFixedArray

No se asumirá automáticamente superior.

Deberá benchmarkearse.

---

# 257. Objects vs arrays

Tampoco se asumirá:

```text
array faster than object
```

o viceversa universalmente.

---

# 258. Performance requirement

La representación se elegirá mediante:

```text
benchmarks
memory measurements
DX
maintainability
```

---

# 259. Hot vs cold metadata

Podrá separarse:

```text
HotMetadata
ColdMetadata
```

---

# 260. Hot metadata

Incluye:

```text
field indexes
accessors
types
identifier
table
relationships
```

---

# 261. Cold metadata

Puede incluir:

```text
documentation
source locations
verbose diagnostics
annotations not used at runtime
```

---

# 262. Benefit

Los hot paths pueden trabajar con estructuras más pequeñas.

---

# 263. Metadata compilation metrics

Métricas posibles:

```text
database.metadata.compile.duration
database.metadata.entities
database.metadata.fields
database.metadata.relationships
database.metadata.cache.hit
database.metadata.cache.miss
database.metadata.validation.errors
database.metadata.memory.bytes
```

---

# 264. Cardinality

No usar:

```text
entity FQCN
```

como label ilimitado en producción si la aplicación puede registrar miles dinámicamente.

---

# 265. Compilation tracing

Podrán existir spans:

```text
database.metadata.discover
database.metadata.normalize
database.metadata.validate
database.metadata.resolve
database.metadata.compile
database.metadata.publish
```

solo durante bootstrap/warmup, no por query.

---

# 266. Performance report

Ejemplo:

```text
Metadata Compilation

Entities                312
Fields                 5,842
Relationships          1,106
Lifecycle Callbacks      127
Custom Types              18

Discovery               74 ms
Normalization           18 ms
Validation              31 ms
Resolution              27 ms
Compilation             42 ms
Cache Write             11 ms

Total                  203 ms

Compiled Size          8.4 MB
```

---

# 267. Warm load

Después:

```text
Metadata Cache Load

Entities               312
Load Time              13 ms
Runtime Binding         8 ms
Total                  21 ms
```

---

# 268. Goal

Mover costo de:

```text
every query
every row
every flush
```

hacia:

```text
bootstrap / deployment / worker startup
```

---

# 269. Cold start

No deberá ignorarse.

Serverless/future environments podrían valorar:

```text
metadata load latency
```

---

# 270. Lazy metadata loading

Podría explorarse.

Pero tiene tradeoffs:

```text
lower startup
vs
runtime first-hit latency
```

---

# 271. Default V1

Para aplicaciones normales:

```text
compile/load complete registry
```

será más simple y predecible.

---

# 272. Large applications

Para miles de entidades podría estudiarse:

```text
segmented metadata registry
```

---

# 273. Segmentation

Ejemplo:

```text
Sales
Billing
Inventory
CRM
```

---

# 274. Segment dependencies

Deberán compilarse explícitamente.

---

# 275. Partial registry

No deberá permitir referencias no resueltas accidentalmente.

---

# 276. Incremental compilation

Development podría recompilar únicamente segmentos afectados.

---

# 277. Dependency graph

Para ello:

```text
Entity A changed
   ↓
Relationships referencing A
   ↓
Affected Metadata Set
```

---

# 278. Reverse dependency index

Puede mantener:

```text
EntityTypeId
→
dependent metadata nodes
```

---

# 279. Production compilation

Preferirá build completo determinista.

---

# 280. Concurrency

Dos procesos podrían intentar generar cache simultáneamente.

---

# 281. Cache write

Debe ser:

```text
atomic
```

cuando sea posible.

---

# 282. Temporary file pattern

Ejemplo:

```text
metadata.tmp
    ↓
write complete
    ↓
fsync if required
    ↓
atomic rename
```

según filesystem/capability.

---

# 283. Partial cache file

Nunca deberá publicarse como válido.

---

# 284. Locking

Podrá existir:

```text
MetadataCompilationLock
```

para evitar stampede.

---

# 285. Lock failure

No deberá bloquear indefinidamente el worker.

---

# 286. Distributed cache

Si metadata cache vive en proveedor distribuido, deberán existir:

```text
generation
integrity
version
namespace
```

---

# 287. Local compiled cache

Para PHP runtime probablemente será preferible un cache local de build para muchas instalaciones.

La arquitectura no lo impondrá universalmente.

---

# 288. Integrity

Artefactos compilados podrán incluir:

```text
checksum
```

---

# 289. Checksum ≠ signature

Checksum detecta corrupción accidental.

No autentica origen.

---

# 290. Signed artifacts

Podrían soportarse para entornos de supply-chain más estrictos.

No será requisito base.

---

# 291. Build provenance

Futuro:

```text
compiler version
build ID
source fingerprint
timestamp
```

---

# 292. Reproducible compilation

Cuando entradas sean idénticas, la salida semántica deberá ser reproducible.

---

# 293. Timestamp

No deberá formar parte del fingerprint semántico si cambia sin alterar mapping.

---

# 294. Ordering

Maps/sets deberán canonicalizarse antes del fingerprint.

---

# 295. Extension versioning

Contribuciones de plugins deberán aportar:

```text
extension ID
extension metadata version
```

---

# 296. Plugin upgrade

Puede invalidar metadata compilada.

---

# 297. Custom driver

Normalmente no debería invalidar ORM metadata lógica.

---

# 298. Custom type

Sí puede invalidarla si cambia:

```text
TypeId semantics
conversion contract
mapping contribution
```

---

# 299. Testing

El sistema deberá probar:

```text
attribute discovery
normalization
local validation
symbol resolution
relationship resolution
type resolution
composite IDs
inheritance
embeddables
lifecycle callbacks
accessor compilation
cache hit
cache miss
cache corruption
generation mismatch
extension ordering
extension cycles
persistent runtime reuse
atomic publication
```

---

# 300. Determinism tests

Compilar dos veces:

```text
same input
```

deberá producir:

```text
same semantic fingerprint
```

---

# 301. Mutation tests

Modificar:

```text
field type
relationship
identifier
column
```

deberá cambiar fingerprint cuando sea semánticamente relevante.

---

# 302. Cache compatibility tests

Caches de compiler version anterior deberán rechazarse correctamente.

---

# 303. Persistent runtime tests

Después de múltiples requests:

```text
metadata registry
```

debe permanecer válido mientras:

```text
request state
```

no quede retenido.

---

# 304. Leak tests

Se comprobará que metadata no mantenga accidentalmente referencias a:

```text
EntityManager
Connection
Transaction
Request
User
TenantContext
Entity
```

---

# 305. Benchmark suite

Escenarios:

```text
10 entities
100 entities
1,000 entities
5,000 entities
```

con distintas densidades de relaciones.

---

# 306. Benchmark cold compilation

Medirá:

```text
discovery
reflection
attribute parsing
normalization
validation
resolution
compilation
```

---

# 307. Benchmark warm load

Medirá:

```text
cache read
deserialization/load
runtime binding
registry construction
```

---

# 308. Runtime lookup benchmark

Medirá:

```text
entity lookup
field lookup
relationship lookup
identifier extraction
property read
property write
```

---

# 309. Performance target philosophy

No se fijarán números universales todavía.

Se establecerán mediante:

```text
250_DATABASE_PERFORMANCE_BENCHMARK_SYSTEM.md
```

---

# 310. Directory structure

```text
src/Quantum/Database/Metadata/
│
├── Contract/
│   ├── MetadataCompiler.php
│   ├── MetadataSource.php
│   ├── MetadataNormalizer.php
│   ├── MetadataValidator.php
│   ├── MetadataResolver.php
│   ├── RuntimeMetadataRegistry.php
│   └── MetadataCompilerExtension.php
│
├── Source/
│   ├── AttributeMetadataSource.php
│   ├── ConfigurationMetadataSource.php
│   ├── ConventionMetadataSource.php
│   └── CompositeMetadataSource.php
│
├── Definition/
│   ├── EntitySourceDefinition.php
│   ├── FieldSourceDefinition.php
│   ├── RelationshipSourceDefinition.php
│   ├── IdentifierSourceDefinition.php
│   ├── LifecycleSourceDefinition.php
│   └── ValueObjectSourceDefinition.php
│
├── Normalization/
│   ├── DefaultMetadataNormalizer.php
│   ├── CanonicalEntityDefinition.php
│   ├── CanonicalFieldDefinition.php
│   └── CanonicalRelationshipDefinition.php
│
├── Validation/
│   ├── DefaultMetadataValidator.php
│   ├── EntityMetadataValidator.php
│   ├── FieldMetadataValidator.php
│   ├── IdentifierMetadataValidator.php
│   ├── RelationshipMetadataValidator.php
│   └── LifecycleMetadataValidator.php
│
├── Resolution/
│   ├── MetadataSymbolTable.php
│   ├── MetadataSymbolResolver.php
│   ├── RelationshipResolver.php
│   ├── TypeResolver.php
│   └── RepositoryResolver.php
│
├── Compilation/
│   ├── DefaultMetadataCompiler.php
│   ├── MetadataCompilationPipeline.php
│   ├── MetadataCompilationGraph.php
│   ├── MetadataCompilationInput.php
│   ├── MetadataCompilationContext.php
│   └── MetadataCompilationPolicy.php
│
├── Compiled/
│   ├── CompiledMetadataSet.php
│   ├── CompiledEntityMetadata.php
│   ├── CompiledFieldMetadata.php
│   ├── CompiledIdentifierMetadata.php
│   ├── CompiledRelationshipMetadata.php
│   ├── CompiledLifecycleMetadata.php
│   ├── CompiledDiscriminatorMap.php
│   ├── CompiledVersionMetadata.php
│   └── CompiledTypeBinding.php
│
├── Accessor/
│   ├── CompiledPropertyReader.php
│   ├── CompiledPropertyWriter.php
│   ├── ReflectionPropertyReader.php
│   ├── ReflectionPropertyWriter.php
│   ├── ClosurePropertyReader.php
│   ├── ClosurePropertyWriter.php
│   └── AccessorCompiler.php
│
├── Identity/
│   ├── EntityTypeId.php
│   ├── FieldId.php
│   ├── RelationshipId.php
│   ├── MetadataGeneration.php
│   ├── MetadataFingerprint.php
│   └── MetadataVariantId.php
│
├── Registry/
│   ├── DefaultRuntimeMetadataRegistry.php
│   ├── EntityMetadataIndex.php
│   ├── FieldMetadataIndex.php
│   └── RelationshipMetadataIndex.php
│
├── Cache/
│   ├── MetadataCache.php
│   ├── MetadataCacheKey.php
│   ├── MetadataCacheManifest.php
│   ├── MetadataCacheLoader.php
│   └── MetadataCacheWriter.php
│
├── Extension/
│   ├── MetadataExtensionRegistry.php
│   ├── MetadataExtensionGraph.php
│   └── MetadataExtensionPhase.php
│
├── Diagnostic/
│   ├── MetadataDiagnosticMap.php
│   ├── MetadataProvenance.php
│   └── MetadataCompilationReport.php
│
├── Telemetry/
│   └── MetadataCompilationTelemetryBridge.php
│
├── Testing/
│   ├── FakeMetadataSource.php
│   ├── MetadataCompilationAssertions.php
│   └── MetadataFixtureBuilder.php
│
└── Exception/
    ├── DatabaseMetadataCompilationException.php
    ├── MetadataDiscoveryException.php
    ├── MetadataNormalizationException.php
    ├── MetadataValidationException.php
    ├── MetadataSymbolResolutionException.php
    ├── MetadataRelationshipResolutionException.php
    ├── MetadataTypeResolutionException.php
    ├── MetadataAccessorCompilationException.php
    ├── MetadataGenerationMismatchException.php
    ├── MetadataCacheException.php
    └── MetadataPublicationException.php
```

---

# 311. Dependency rules

Permitido:

```text
ORM
 ↓
RuntimeMetadataRegistry
```

```text
Hydration
 ↓
CompiledEntityMetadata
```

```text
Persistence
 ↓
CompiledEntityMetadata
```

```text
Query Semantic Engine
 ↓
CompiledEntityMetadata
```

---

# 312. Prohibido

```text
Metadata
 ↓
EntityManager runtime state
```

```text
Metadata
 ↓
UnitOfWork mutable state
```

```text
Metadata
 ↓
current Connection
```

```text
Metadata
 ↓
current Transaction
```

---

# 313. Compiler dependencies

El compiler sí puede depender estructuralmente de:

```text
Type Registry
Platform Capability contracts
Extension Registry
Configuration
Reflection abstraction
```

---

# 314. Metadata and Container

Container podrá construir:

```text
MetadataCompiler
RuntimeMetadataRegistry
```

---

# 315. Compiled metadata ≠ service locator

Metadata no deberá guardar:

```text
Container
```

para resolver servicios arbitrariamente.

---

# 316. Custom handlers

Si necesita handler:

```text
HandlerDescriptor
```

podrá resolverse por un runtime binder.

---

# 317. Why

Evita:

```text
CompiledMetadata
→
Container
→
everything
```

---

# 318. Architectural invariants

## DB-METACOMP-001
Metadata Compilation no ejecutará queries de aplicación.

## DB-METACOMP-002
Metadata Compilation no generará SQL de queries.

## DB-METACOMP-003
Source Metadata será distinta de Compiled Metadata.

## DB-METACOMP-004
Compiled Metadata será distinta de runtime state.

## DB-METACOMP-005
Metadata deberá normalizarse antes de compilación final.

## DB-METACOMP-006
Metadata deberá validarse antes de publicación.

## DB-METACOMP-007
Symbol resolution ocurrirá antes del freeze.

## DB-METACOMP-008
Relationship references deberán resolverse antes del freeze.

## DB-METACOMP-009
Compiled Metadata será inmutable.

## DB-METACOMP-010
Publication será atómica a nivel de generación.

## DB-METACOMP-011
Una compilación fallida no publicará metadata parcial.

## DB-METACOMP-012
Cada conjunto tendrá MetadataGeneration.

## DB-METACOMP-013
MetadataGeneration no será ApplicationVersion.

## DB-METACOMP-014
EntityTypeId será identidad lógica estable.

## DB-METACOMP-015
FQCN no será necesariamente identidad persistente.

## DB-METACOMP-016
FieldId será distinto de FieldIndex.

## DB-METACOMP-017
FieldIndex será optimización interna.

## DB-METACOMP-018
FieldIndex no será identificador durable externo.

## DB-METACOMP-019
Field indexes serán deterministas dentro de una generación.

## DB-METACOMP-020
Field lookup evitará scans lineales en hot paths.

## DB-METACOMP-021
Relationship lookup evitará scans lineales.

## DB-METACOMP-022
Type resolution no se repetirá innecesariamente.

## DB-METACOMP-023
Reflection discovery no ocurrirá por fila.

## DB-METACOMP-024
Attribute parsing no ocurrirá por fila.

## DB-METACOMP-025
Attribute parsing no ocurrirá por entity flush.

## DB-METACOMP-026
Property readers podrán precompilarse.

## DB-METACOMP-027
Property writers podrán precompilarse.

## DB-METACOMP-028
Reflection podrá existir como fallback.

## DB-METACOMP-029
ReflectionProperty será reutilizable cuando sea seguro.

## DB-METACOMP-030
Generated code será opcional.

## DB-METACOMP-031
Generated code requerirá validación de seguridad.

## DB-METACOMP-032
eval indiscriminado no será arquitectura base.

## DB-METACOMP-033
Generated metadata podrá aprovechar OPcache.

## DB-METACOMP-034
OPcache no será dependencia conceptual.

## DB-METACOMP-035
Metadata compilation será determinista.

## DB-METACOMP-036
Semantic fingerprint usará representación canónica.

## DB-METACOMP-037
mtime no será semantic fingerprint.

## DB-METACOMP-038
Metadata Cache será distinta de Entity Cache.

## DB-METACOMP-039
Metadata Cache será distinta de Result Cache.

## DB-METACOMP-040
Metadata Cache será distinta de Query Cache.

## DB-METACOMP-041
Metadata Cache será distinta de IdentityMap.

## DB-METACOMP-042
Metadata Cache no almacenará entity instances.

## DB-METACOMP-043
Metadata Cache no almacenará Connections.

## DB-METACOMP-044
Metadata Cache no almacenará Transactions.

## DB-METACOMP-045
Metadata Cache no almacenará request state.

## DB-METACOMP-046
Runtime binding no repetirá source discovery.

## DB-METACOMP-047
Type handlers podrán pre-bindearse.

## DB-METACOMP-048
Logical metadata no dependerá innecesariamente de SQL físico.

## DB-METACOMP-049
Metadata Compilation será distinta de SQL Compilation.

## DB-METACOMP-050
Identifier metadata será pre-resuelta.

## DB-METACOMP-051
Composite identifiers serán soportados.

## DB-METACOMP-052
Identifier canonicalization será consistente.

## DB-METACOMP-053
Relationship targets serán pre-resueltos.

## DB-METACOMP-054
mappedBy/inversedBy serán validados.

## DB-METACOMP-055
Relationship ownership será explícito.

## DB-METACOMP-056
Cascade metadata no ejecutará cascades.

## DB-METACOMP-057
Static dependency graph será distinto de runtime object graph.

## DB-METACOMP-058
Embeddables podrán precompilarse.

## DB-METACOMP-059
ValueObjectMetadata será distinta de EntityMetadata.

## DB-METACOMP-060
Inheritance metadata será validada.

## DB-METACOMP-061
Discriminator maps serán seguros.

## DB-METACOMP-062
DB discriminator no podrá resolver FQCN arbitrario.

## DB-METACOMP-063
Lifecycle callbacks se pre-resolverán.

## DB-METACOMP-064
Lifecycle methods no se escanearán por evento.

## DB-METACOMP-065
Lifecycle Callback será distinto de Database Event.

## DB-METACOMP-066
Repository metadata no será repository mutable global.

## DB-METACOMP-067
Model API y EntityManager compartirán metadata canónica.

## DB-METACOMP-068
No existirán dos ORM metadata engines.

## DB-METACOMP-069
Cast será distinto de Type.

## DB-METACOMP-070
Cast handlers podrán pre-bindearse.

## DB-METACOMP-071
Query Engine podrá consumir metadata compilada.

## DB-METACOMP-072
Hydration podrá consumir metadata compilada.

## DB-METACOMP-073
Persistence podrá consumir metadata compilada.

## DB-METACOMP-074
Dirty checking podrá utilizar FieldIndex.

## DB-METACOMP-075
LoadedFieldMask podrá utilizar FieldIndex.

## DB-METACOMP-076
FieldIndex tendrá una fuente estructural única.

## DB-METACOMP-077
ORM Mapping será distinto de Schema.

## DB-METACOMP-078
Schema derivation será explícita.

## DB-METACOMP-079
Metadata validation no requerirá DB por defecto.

## DB-METACOMP-080
Schema validation será operación distinta.

## DB-METACOMP-081
Offline compilation será soportada cuando sea posible.

## DB-METACOMP-082
Development podrá recompilar automáticamente.

## DB-METACOMP-083
Production favorecerá metadata precompilada.

## DB-METACOMP-084
Development y production preservarán misma semántica.

## DB-METACOMP-085
Cache manifest tendrá versión de compiler.

## DB-METACOMP-086
Cache incompatible será rechazado.

## DB-METACOMP-087
Cache corruption no significará empty metadata.

## DB-METACOMP-088
Cache invalidation no usará FLUSHALL.

## DB-METACOMP-089
Compilation errors identificarán origen.

## DB-METACOMP-090
Duplicate EntityTypeId será detectado.

## DB-METACOMP-091
Unknown TypeId será error.

## DB-METACOMP-092
Unknown relationship target será error.

## DB-METACOMP-093
Invalid mappedBy será error.

## DB-METACOMP-094
Identifier mapping será validado.

## DB-METACOMP-095
Keyless projection no será managed entity.

## DB-METACOMP-096
Optimistic lock version field será pre-resuelto.

## DB-METACOMP-097
Optional modules contribuirán mediante extensions.

## DB-METACOMP-098
Core compiler no conocerá todos los optional modules.

## DB-METACOMP-099
Extension ordering será determinista.

## DB-METACOMP-100
Extension dependency cycles serán detectados.

## DB-METACOMP-101
Extensions no mutarán metadata frozen.

## DB-METACOMP-102
CompilationGraph será distinto de RuntimeRegistry.

## DB-METACOMP-103
CompilationGraph podrá ser mutable durante compilación.

## DB-METACOMP-104
RuntimeRegistry será inmutable.

## DB-METACOMP-105
RuntimeRegistry lookup será indexado.

## DB-METACOMP-106
RuntimeRegistry no se mutará durante request normal.

## DB-METACOMP-107
Dynamic metadata generará nueva generation.

## DB-METACOMP-108
In-flight operation permanecerá en una generation coherente.

## DB-METACOMP-109
No se mezclarán planes de generations incompatibles.

## DB-METACOMP-110
Generation mismatch será detectable.

## DB-METACOMP-111
Immutable metadata podrá sobrevivir persistent runtime requests.

## DB-METACOMP-112
Metadata no retendrá EntityManager.

## DB-METACOMP-113
Metadata no retendrá Connection.

## DB-METACOMP-114
Metadata no retendrá Transaction.

## DB-METACOMP-115
Metadata no retendrá Request.

## DB-METACOMP-116
Metadata no retendrá current User.

## DB-METACOMP-117
Metadata no retendrá mutable TenantContext.

## DB-METACOMP-118
Metadata no retendrá entity instances.

## DB-METACOMP-119
Shared-schema tenancy reutilizará metadata cuando sea compatible.

## DB-METACOMP-120
Schema name variable no obligará a duplicar metadata lógica.

## DB-METACOMP-121
Tenant-specific structure podrá usar MetadataVariant.

## DB-METACOMP-122
MetadataVariant será distinto de TenantId.

## DB-METACOMP-123
Variant explosion será gobernable.

## DB-METACOMP-124
Shard físico no duplicará metadata lógica sin necesidad.

## DB-METACOMP-125
Platform differences usarán capabilities.

## DB-METACOMP-126
Vendor conditionals no se dispersarán por metadata.

## DB-METACOMP-127
Sensitive classifications podrán precompilarse.

## DB-METACOMP-128
Sensitive classification será distinta de authorization.

## DB-METACOMP-129
Debug metadata podrá ser restringida.

## DB-METACOMP-130
Metadata provenance podrá conservarse.

## DB-METACOMP-131
Runtime metadata y diagnostic metadata podrán separarse.

## DB-METACOMP-132
Production podrá reducir cold diagnostic metadata.

## DB-METACOMP-133
Required error information permanecerá disponible.

## DB-METACOMP-134
Metadata memory será medible.

## DB-METACOMP-135
Duplicación estructural deberá minimizarse.

## DB-METACOMP-136
Compact representations requerirán benchmarks.

## DB-METACOMP-137
SplFixedArray no se asumirá superior.

## DB-METACOMP-138
Array no se asumirá superior a object.

## DB-METACOMP-139
HotMetadata podrá separarse de ColdMetadata.

## DB-METACOMP-140
Compilation telemetry ocurrirá principalmente en bootstrap.

## DB-METACOMP-141
Compilation telemetry no ocurrirá por query.

## DB-METACOMP-142
Cold compilation será medible.

## DB-METACOMP-143
Warm load será medible.

## DB-METACOMP-144
Runtime lookup será benchmarkeable.

## DB-METACOMP-145
Metadata cost se moverá fuera de hot paths.

## DB-METACOMP-146
Cold start seguirá siendo una consideración.

## DB-METACOMP-147
Lazy metadata loading será opcional.

## DB-METACOMP-148
V1 favorecerá registry completo salvo necesidad demostrada.

## DB-METACOMP-149
Segmented registry podrá existir para aplicaciones enormes.

## DB-METACOMP-150
Partial registry no dejará referencias accidentalmente unresolved.

## DB-METACOMP-151
Incremental compilation podrá existir en development.

## DB-METACOMP-152
Production compilation favorecerá determinismo.

## DB-METACOMP-153
Concurrent cache generation será segura.

## DB-METACOMP-154
Cache publication evitará archivos parciales.

## DB-METACOMP-155
Compilation lock no bloqueará indefinidamente.

## DB-METACOMP-156
Distributed metadata artifacts tendrán generation.

## DB-METACOMP-157
Cache integrity podrá verificarse.

## DB-METACOMP-158
Checksum será distinto de signature.

## DB-METACOMP-159
Reproducible compilation será objetivo.

## DB-METACOMP-160
Timestamp no cambiará semantic fingerprint por sí solo.

## DB-METACOMP-161
Extension versions podrán invalidar metadata.

## DB-METACOMP-162
Custom driver no invalidará metadata lógica sin razón.

## DB-METACOMP-163
Custom type semantics sí podrán invalidarla.

## DB-METACOMP-164
Determinism tendrá tests.

## DB-METACOMP-165
Cache compatibility tendrá tests.

## DB-METACOMP-166
Persistent runtime safety tendrá tests.

## DB-METACOMP-167
Reference leaks tendrán tests.

## DB-METACOMP-168
Compilation performance tendrá benchmarks.

## DB-METACOMP-169
Warm load tendrá benchmarks.

## DB-METACOMP-170
Runtime lookup tendrá benchmarks.

## DB-METACOMP-171
Metadata no será Service Locator.

## DB-METACOMP-172
Compiled metadata no almacenará Container arbitrariamente.

## DB-METACOMP-173
Runtime handlers podrán resolverse mediante binder explícito.

## DB-METACOMP-174
Correctness tendrá prioridad sobre compilation speed.

## DB-METACOMP-175
Runtime efficiency no debilitará mapping semantics.

## DB-METACOMP-176
Invalid metadata deberá fallar temprano.

## DB-METACOMP-177
UNKNOWN mapping no se convertirá silenciosamente en default.

## DB-METACOMP-178
Metadata compilation será observable.

## DB-METACOMP-179
Metadata generation será observable.

## DB-METACOMP-180
Compiled metadata será la representación canónica de runtime.

---

# 319. Modelo formal

Sea:

```text
S
```

el conjunto de fuentes declarativas.

La normalización produce:

```text
N = Normalize(S)
```

La validación:

```text
Validate(N) → Valid | Error
```

La resolución:

```text
R = Resolve(N, Symbols, Types, Extensions)
```

La compilación:

```text
C = Compile(R)
```

y el freeze:

```text
F = Freeze(C)
```

Finalmente:

```text
Publish(F, Generation)
```

---

# 320. Condición de publicación

Solo si:

```text
Discovery = SUCCESS
∧
Normalization = SUCCESS
∧
Validation = SUCCESS
∧
Resolution = SUCCESS
∧
Compilation = SUCCESS
∧
Freeze = SUCCESS
```

se permite:

```text
Publish
```

---

# 321. Atomicidad conceptual

```text
Registry(Gn)
    │
    │ compile
    ▼
Candidate(Gn+1)
    │
    ├── invalid → discard
    │
    └── valid
          ↓
      atomic swap
          ↓
Registry(Gn+1)
```

---

# 322. Runtime lookup model

Después de compilación:

```text
LookupEntity(EntityTypeId)
≈
O(1)

LookupField(EntityTypeId, FieldId)
≈
O(1)

LookupRelationship(RelationshipId)
≈
O(1)
```

promedio con estructuras hash/indexadas apropiadas.

---

# 323. Hot-path transformation

Antes:

```text
Query
 ↓
Reflection
 ↓
Attributes
 ↓
Type lookup
 ↓
Relationship lookup
 ↓
Property lookup
 ↓
Execution
```

Después:

```text
Query
 ↓
Compiled Metadata Lookup
 ↓
Execution
```

---

# 324. Hydration transformation

Antes:

```text
for each row
    discover mapping
    discover type
    discover property
    reflect property
    assign
```

Después:

```text
HydrationPlan
    ↓
Compiled Field Metadata
    ↓
Pre-bound Converter
    ↓
Compiled Writer
```

---

# 325. Persistence transformation

Antes:

```text
flush entity
 ↓
inspect class
 ↓
discover fields
 ↓
discover ID
 ↓
discover relationships
 ↓
build persistence work
```

Después:

```text
flush entity
 ↓
CompiledEntityMetadata
 ↓
Field indexes
 ↓
Identifier extractor
 ↓
Relationship descriptors
 ↓
Persistence Planner
```

---

# 326. Runtime cost philosophy

El costo deseado será:

```text
Application Bootstrap
    =
    Higher Structural Cost

Application Hot Path
    =
    Lower Repeated Structural Cost
```

---

# 327. Persistent runtime model

Con FrankenPHP:

```text
Process / Worker
│
├── Load Metadata Generation G42
│
├── Freeze Registry
│
├── Bind Accessors
│
│
├── Request A
│     └── read G42
│
├── Request B
│     └── read G42
│
├── Request C
│     └── read G42
│
└── Worker Shutdown
```

No:

```text
Request A mutates metadata
      ↓
Request B observes mutation
```

---

# 328. Compilation philosophy

La metadata declarativa debe optimizar la experiencia del desarrollador:

```php
#[Entity]
final class User
{
    #[Id]
    private UserId $id;

    #[Column]
    private string $name;
}
```

mientras la metadata compilada debe optimizar el runtime:

```text
EntityTypeId       #14
IdentifierFields   [0]
FieldCount         2
Field[0]            UUID handler #3
Field[1]            String handler #1
Reader[0]           accessor #42
Writer[0]           accessor #43
RelationshipMask    0000
```

El desarrollador no deberá trabajar directamente con esta representación compacta.

---

# 329. Developer API ≠ runtime representation

Regla:

> **VoltStack podrá ofrecer una metadata declarativa expresiva para humanos y simultáneamente una metadata compilada compacta para la máquina; ninguna de las dos necesidades deberá obligar a sacrificar la otra.**

---

# 330. Anti-patterns

Quedan prohibidos o desaconsejados:

```text
Reflection per row

Reflection per field read

Reflection per field write

Attribute parsing per query

Attribute parsing per hydration

Attribute parsing per flush

Scanning all entity fields to find one field

Scanning all relationships to find one relationship

Resolving TypeRegistry strings per hydrated value

Resolving target classes repeatedly

Using FQCN from database values

Publishing partially compiled metadata

Mutating compiled metadata during requests

Storing EntityManager inside metadata

Storing Connection inside metadata

Storing Transaction inside metadata

Storing Request inside metadata

Storing current Tenant inside metadata

Storing entity instances inside metadata

Treating metadata cache as entity cache

Using FLUSHALL for metadata invalidation

Treating corrupted cache as empty metadata

Using file mtime as semantic identity

Mixing generations in one operation

Generating code from untrusted strings

Using eval() as default compiler strategy

Duplicating metadata per tenant unnecessarily

Duplicating metadata per shard unnecessarily

Making metadata a service locator

Keeping Container inside every metadata object

Running DB queries from attributes

Running HTTP calls from attributes

Using non-deterministic attribute behavior

Depending on current user during compilation

Depending on request-specific state during compilation

Assuming array is always fastest

Assuming generated code is always fastest

Assuming Reflection is always too slow

Optimizing without benchmarks

Persisting FieldIndex as stable external identity

Treating ORM Mapping as complete physical Schema

Performing schema introspection per request

Allowing extensions to mutate frozen metadata

Ignoring extension dependency cycles

Ignoring cache/compiler version compatibility
```

---

# 331. Integración global

```text
                        SOURCE CODE
                            │
                            ▼
                  METADATA COMPILATION
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Entities        Fields      Relationships
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                    COMPILED METADATA
                            │
       ┌────────────────────┼─────────────────────┐
       ▼                    ▼                     ▼
 QUERY ENGINE          HYDRATION ENGINE      ORM ENGINE
       │                    │                     │
       ▼                    ▼                     ▼
Symbol Resolution     Field Mapping        Identity
Type Inference        Type Conversion      Dirty Checking
Join Resolution       Accessors            UnitOfWork
       │              Identity Mapping     Relationships
       │                    │                     │
       └────────────────────┼─────────────────────┘
                            ▼
                    PERSISTENCE ENGINE
                            │
                            ▼
                       QUERY ENGINE
```

---

# 332. Beneficio para VoltStack

Esta arquitectura permite combinar:

```text
Laravel-like DX
```

con una estrategia interna más cercana a:

```text
compiled metadata
explicit runtime state
immutable structural descriptors
pre-resolved mappings
```

sin exponer complejidad innecesaria al desarrollador.

---

# 333. Ejemplo completo

Código de aplicación:

```php
#[Entity]
#[Table('users')]
final class User
{
    #[Id]
    #[Column(type: 'uuid')]
    private UserId $id;

    #[Column(type: 'string')]
    private string $name;

    #[Column(type: 'string')]
    private string $email;

    #[ManyToOne(target: Company::class)]
    private Company $company;
}
```

Discovery:

```text
Reflection
Attributes
```

produce:

```text
EntitySourceDefinition(User)
```

Normalization:

```text
CanonicalEntityDefinition
```

Resolution:

```text
User
    ↓
EntityTypeId(user)

Company
    ↓
EntityTypeId(company)

uuid
    ↓
TypeId(uuid)
```

Compilation:

```text
CompiledEntityMetadata

EntityType:
    user

Table:
    users

Identifier:
    field #0

Fields:
    #0 id
    #1 name
    #2 email

Relationships:
    #0 company
        target → EntityTypeId(company)
        kind   → MANY_TO_ONE
```

Runtime:

```text
User query
   ↓
MetadataRegistry[user]
   ↓
field lookup
   ↓
Query AST
   ↓
Execution
   ↓
HydrationPlan
   ↓
compiled writers
   ↓
User entities
```

Sin volver a descubrir la clase en cada fila.

---

# 334. Resultado arquitectónico

El sistema deberá conseguir que:

```text
Expensive Structural Interpretation
```

ocurra principalmente en:

```text
Build
Bootstrap
Warmup
Worker Startup
```

mientras:

```text
Request
Query
Hydration
Dirty Checking
Flush
Relationship Loading
```

consuman:

```text
Compiled Structural Knowledge
```

---

# 335. Regla arquitectónica final

```text
Source Metadata
≠
Compiled Metadata

Metadata
≠
Runtime State

Metadata Compilation
≠
SQL Compilation

ORM Mapping
≠
Physical Schema

FieldId
≠
FieldIndex

EntityTypeId
≠
FQCN

MetadataGeneration
≠
ApplicationVersion

Metadata Cache
≠
Entity Cache

Compiled Metadata
≠
Service Locator

Static Metadata Graph
≠
Runtime Object Graph

Development Reload
≠
Runtime Mutation

Shared Metadata
≠
Shared ORM State

Optimization
≠
Loss of Semantics
```

---

# 336. Principio final

> **La metadata declarativa existe para hacer VoltStack expresivo para el desarrollador; la metadata compilada existe para hacer VoltStack eficiente para el runtime.**

El sistema deberá permitir escribir:

```php
#[ManyToOne(target: Company::class)]
private Company $company;
```

pero ejecutar internamente con algo conceptualmente más cercano a:

```text
RelationshipIndex: 3
TargetEntityType: 7
OwnerField: 5
JoinField: 2
TypeBinding: 4
Reader: 17
Writer: 18
```

sin redescubrir esa información repetidamente.

---

# 337. Relación con el Bloque 24

```text
242 DATABASE PERFORMANCE ARCHITECTURE
             │
             ├── 243 QUERY PERFORMANCE
             ├── 244 ORM PERFORMANCE
             ├── 245 HYDRATION PERFORMANCE
             ├── 246 METADATA COMPILATION
             │       │
             │       ├── Discovery
             │       ├── Normalization
             │       ├── Validation
             │       ├── Symbol Resolution
             │       ├── Relationship Resolution
             │       ├── Type Binding
             │       ├── Accessor Compilation
             │       ├── Index Generation
             │       ├── Cache
             │       ├── Freeze
             │       └── Runtime Registry
             │
             ├── 247 QUERY COMPILATION OPTIMIZATION
             ├── 248 MEMORY MANAGEMENT
             ├── 249 RESOURCE GOVERNANCE
             └── 250 PERFORMANCE BENCHMARK
```

---

# 338. Estado del Bloque 24

```text
BLOCK 24 — PERFORMANCE

✓ 242_DATABASE_PERFORMANCE_ARCHITECTURE.md
✓ 243_DATABASE_QUERY_PERFORMANCE_SYSTEM.md
✓ 244_DATABASE_ORM_PERFORMANCE_SYSTEM.md
✓ 245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md
✓ 246_DATABASE_METADATA_COMPILATION_SYSTEM.md
│
├── 247_DATABASE_QUERY_COMPILATION_OPTIMIZATION_SYSTEM.md
├── 248_DATABASE_MEMORY_MANAGEMENT_SYSTEM.md
├── 249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
└── 250_DATABASE_PERFORMANCE_BENCHMARK_SYSTEM.md
```

---

# 339. Siguiente documento

```text
247_DATABASE_QUERY_COMPILATION_OPTIMIZATION_SYSTEM.md
```

El siguiente documento deberá definir cómo VoltStack optimizará el camino:

```text
Query Model / AST
        ↓
Semantic Query
        ↓
Logical Plan
        ↓
Physical Plan
        ↓
SQL Compiler
        ↓
Compiled Query
        ↓
Prepared Statement
```

reduciendo:

```text
repeated AST traversal
repeated semantic analysis
repeated normalization
repeated compiler dispatch
repeated SQL generation
repeated parameter layout calculation
```

mediante:

```text
canonical query fingerprints
compiled query reuse
prepared compilation plans
dialect-aware specialization
parameter layout caching
immutable compiler artifacts
safe persistent-runtime reuse
```

bajo la regla:

> **VoltStack deberá reutilizar trabajo de compilación únicamente cuando pueda demostrar equivalencia semántica y compatibilidad de contexto; una coincidencia superficial de SQL, strings o estructura nunca será suficiente para comprometer correctness, seguridad, tenant isolation o platform semantics.**