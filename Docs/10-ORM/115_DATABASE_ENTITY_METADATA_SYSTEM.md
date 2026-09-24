# 115_DATABASE_ENTITY_METADATA_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Metadata System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 115 — Database Entity Metadata System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Metadata System` define el modelo canónico mediante el cual el ORM de VoltStack describe, valida, compila, registra y consulta la estructura persistente de una entidad.

La metadata deberá responder preguntas como:

```text
¿Qué clase PHP representa esta entidad?
¿Cuál es su identidad persistente?
¿Qué campos están mapeados?
¿Qué tipos poseen?
¿Qué columnas representan?
¿Qué relaciones existen?
¿Cómo se instancia?
¿Cómo se hidrata?
¿Cómo se detectan cambios?
¿Qué estrategia de identificador utiliza?
¿Qué lifecycle hooks posee?
¿Qué comportamiento de persistencia requiere?
```

sin necesitar una instancia viva de la entidad.

Principio central:

> **Entity Metadata describe cómo VoltStack entiende una entidad; no representa la entidad viva, no contiene su estado runtime y no ejecuta persistencia.**

Formalmente:

```text
EntityMetadata
≠
Entity
≠
EntityState
≠
UnitOfWork
≠
Query
≠
PersistenceOperation
```

---

# 2. Posición arquitectónica

```text
Application Entities
        │
        ▼
Mapping Sources
├── PHP Attributes
├── Model conventions
├── Explicit configuration
└── Extensions
        │
        ▼
Metadata Discovery
        │
        ▼
Metadata Builder
        │
        ▼
Metadata Normalization
        │
        ▼
Metadata Validation
        │
        ▼
Metadata Compiler
        │
        ▼
EntityMetadata
        │
        ▼
EntityMetadataRegistry
        │
        ├───────────────┐
        ▼               ▼
 EntityManager      EntityQuery
        │               │
        ▼               ▼
   UnitOfWork       Query Engine
        │
        ▼
Persistence Engine
```

La metadata se encuentra en una posición crítica:

```text
Mapping
   ↓
Metadata
   ↓
ORM Runtime
```

---

# 3. Objetivos

El sistema deberá proporcionar:

- representación tipada de metadata;
- metadata inmutable;
- identificación estable de tipos de entidad;
- metadata de identificadores;
- metadata de campos;
- metadata de columnas;
- metadata de relaciones;
- metadata de tipos;
- metadata de generación de identificadores;
- metadata de versionado;
- metadata de lifecycle;
- metadata de change tracking;
- metadata de instanciación;
- metadata de hidratación;
- metadata de persistencia;
- metadata de herencia;
- metadata de Model API;
- validación temprana;
- compilación;
- caching;
- fingerprints;
- extensibilidad;
- introspección;
- compatibilidad con persistent runtimes.

---

# 4. No objetivos

El sistema no deberá:

- mantener entidades vivas;
- almacenar valores actuales de propiedades;
- mantener snapshots de entidades;
- detectar cambios runtime;
- ejecutar queries;
- generar SQL;
- ejecutar SQL;
- abrir conexiones;
- administrar transacciones;
- implementar IdentityMap;
- implementar UnitOfWork;
- hidratar directamente;
- persistir directamente.

---

# 5. Regla fundamental

```text
Metadata describes.
Runtime acts.
```

O:

```text
EntityMetadata
    ↓ describes
Entity

EntityManager
    ↓ manages
Entity

UnitOfWork
    ↓ tracks
Entity

PersistenceEngine
    ↓ persists
Entity
```

---

# 6. EntityMetadata

Contrato conceptual:

```php
interface EntityMetadata
{
    public function entityType(): EntityType;

    public function className(): string;

    public function identifier(): EntityIdentifierMetadata;

    public function fields(): EntityFieldMetadataCollection;

    public function relationships(): RelationshipMetadataCollection;

    public function lifecycle(): EntityLifecycleMetadata;

    public function persistence(): EntityPersistenceMetadata;

    public function hydration(): EntityHydrationMetadata;

    public function instantiation(): EntityInstantiationMetadata;

    public function changeTracking(): ChangeTrackingMetadata;

    public function fingerprint(): EntityMetadataFingerprint;
}
```

La interfaz exacta podrá dividirse en contratos más pequeños.

---

# 7. Metadata como agregado

`EntityMetadata` será un agregado compuesto:

```text
EntityMetadata
│
├── EntityTypeMetadata
├── EntityIdentifierMetadata
├── EntityFieldMetadata[]
├── RelationshipMetadata[]
├── EntityInstantiationMetadata
├── EntityHydrationMetadata
├── EntityPersistenceMetadata
├── ChangeTrackingMetadata
├── EntityLifecycleMetadata
├── EntityInheritanceMetadata
├── EntityVersionMetadata
└── ModelApiMetadataRef
```

---

# 8. Metadata ≠ reflection

Reflection puede ser una fuente para construir metadata:

```text
PHP Reflection
      ↓
Attribute Reader
      ↓
Metadata Builder
```

pero:

```text
EntityMetadata
≠
ReflectionClass
```

---

# 9. Metadata ≠ mapping source

También:

```text
#[Column(...)]
```

es una declaración de mapping.

No es la metadata runtime final.

```text
Attribute
    ↓
Mapping Declaration
    ↓
Normalization
    ↓
Validation
    ↓
Compiled Metadata
```

---

# 10. Pipeline general

```text
Entity Class
     │
     ▼
Mapping Source Discovery
     │
     ▼
Raw Mapping Declaration
     │
     ▼
Metadata Builder
     │
     ▼
Mutable Metadata Draft
     │
     ▼
Normalization
     │
     ▼
Cross-Entity Resolution
     │
     ▼
Validation
     │
     ▼
Compilation
     │
     ▼
Frozen EntityMetadata
     │
     ▼
Registry
```

---

# 11. Estados de metadata

La metadata podrá atravesar:

```text
DISCOVERED
    ↓
BUILDING
    ↓
NORMALIZED
    ↓
RESOLVED
    ↓
VALIDATED
    ↓
COMPILED
    ↓
FROZEN
```

Solo:

```text
FROZEN
```

deberá utilizarse normalmente en runtime de producción.

---

# 12. Metadata Draft ≠ Metadata

Durante construcción podrá existir:

```text
EntityMetadataDraft
```

mutable.

Pero:

```text
EntityMetadataDraft
≠
EntityMetadata
```

---

# 13. EntityType

Se introduce:

```php
final readonly class EntityType
{
    public function __construct(
        public string $name,
    ) {}
}
```

No deberá confundirse con:

```text
PHP class name
database table
entity identifier
```

---

# 14. EntityType ≠ class-string

Normalmente:

```text
EntityType(User)
↔
App\Domain\User
```

pero la arquitectura no deberá asumir que ambos conceptos son idénticos.

---

# 15. EntityTypeId

Podrá existir una identidad interna estable:

```text
EntityTypeId
```

Ejemplo conceptual:

```text
app.user
```

Esto facilita:

- metadata cache;
- proxies;
- inheritance;
- serialization;
- telemetry;
- compiled metadata;
- package boundaries.

---

# 16. Class metadata

```php
final readonly class EntityClassMetadata
{
    public function __construct(
        public string $className,
        public bool $final,
        public bool $readonly,
        public bool $abstract,
    ) {}
}
```

---

# 17. Entity table mapping

La metadata podrá contener una referencia lógica:

```text
EntityStorageMetadata
```

Ejemplo:

```php
final readonly class EntityStorageMetadata
{
    public function __construct(
        public TableReference $table,
        public ?LogicalConnectionName $connection,
    ) {}
}
```

---

# 18. TableReference ≠ SQL identifier string

Preferir:

```text
TableReference
```

sobre:

```php
'users'
```

internamente.

---

# 19. Logical storage ≠ live database

La metadata puede declarar:

```text
connection = "default"
table = "users"
```

pero no contiene:

```text
PDO
Connection
Driver
Tenant connection
```

---

# 20. Identifier metadata

Toda entidad persistible deberá definir identidad.

```text
EntityIdentifierMetadata
```

---

# 21. Identifier metadata model

```php
final readonly class EntityIdentifierMetadata
{
    public function __construct(
        public EntityIdentifierKind $kind,
        public array $fields,
        public IdentifierGenerationStrategy $generation,
    ) {}
}
```

---

# 22. Identifier kinds

Podrán existir:

```text
SINGLE
COMPOSITE
EMBEDDED
CUSTOM
```

---

# 23. Single identifier

Ejemplo:

```php
#[Id]
#[Column]
private int $id;
```

produce:

```text
Identifier
└── field: id
```

---

# 24. Composite identifier

Ejemplo conceptual:

```text
OrderItem
├── order_id
└── product_id
```

produce:

```text
CompositeIdentifier
├── orderId
└── productId
```

---

# 25. Identifier metadata ≠ identifier value

Importante:

```text
EntityIdentifierMetadata
```

describe:

```text
qué campos forman identidad
```

mientras:

```text
EntityIdentifier
```

representa:

```text
el valor concreto de identidad de una entidad
```

---

# 26. Ejemplo

Metadata:

```text
User identifier:
field = id
type = int
generation = AUTO_INCREMENT
```

Runtime:

```text
User#42
```

Son conceptos diferentes.

---

# 27. Identifier generation

La metadata podrá declarar:

```text
NONE
ASSIGNED
AUTO_INCREMENT
SEQUENCE
UUID
ULID
DATABASE_GENERATED
CUSTOM
```

---

# 28. Strategy ≠ generator instance

Metadata deberá almacenar:

```text
IdentifierGenerationStrategyDescriptor
```

no necesariamente un generador mutable vivo.

---

# 29. Field metadata

Cada campo persistente tendrá:

```text
EntityFieldMetadata
```

---

# 30. Modelo conceptual

```php
final readonly class EntityFieldMetadata
{
    public function __construct(
        public FieldName $name,
        public PropertyReference $property,
        public DatabaseTypeReference $type,
        public ColumnMetadata $column,
        public Nullability $nullability,
        public FieldMutability $mutability,
    ) {}
}
```

---

# 31. Field ≠ property ≠ column

Regla crítica:

```text
Entity Field
≠
PHP Property
≠
Database Column
```

Aunque frecuentemente exista:

```text
field name     = email
property       = $email
column         = email
```

no deben fusionarse conceptualmente.

---

# 32. Ejemplo distinto

```text
Entity field:
createdAt

PHP property:
$createdAt

Database column:
created_at
```

---

# 33. FieldName

Se recomienda value object:

```php
final readonly class FieldName
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 34. PropertyReference

```text
PropertyReference
├── declaring class
├── property name
├── visibility strategy
└── access strategy
```

---

# 35. Property access strategies

Podrán existir:

```text
DIRECT
REFLECTION
COMPILED_ACCESSOR
GETTER_SETTER
CONSTRUCTOR
CUSTOM
```

---

# 36. Production preference

En hot paths:

```text
COMPILED_ACCESSOR
```

deberá favorecerse cuando sea viable.

---

# 37. Column metadata

```text
ColumnMetadata
```

podrá describir:

```text
name
database type
length
precision
scale
nullable
default semantics
generated
insertable
updatable
```

---

# 38. ORM column metadata ≠ Schema ColumnDefinition

Existe una frontera importante:

```text
ORM ColumnMetadata
≠
Schema ColumnDefinition
```

La primera describe mapping ORM.

La segunda describe estructura física del schema.

---

# 39. Relación

```text
EntityFieldMetadata
      ↓ maps to
ColumnReference

SchemaModel
      ↓ describes
ColumnDefinition
```

---

# 40. Metadata y Schema

El ORM podrá validar mappings contra Schema Metadata cuando esté disponible.

Pero no deberá depender obligatoriamente de introspección runtime.

---

# 41. Database type reference

El campo deberá referenciar:

```text
DatabaseTypeReference
```

que posteriormente será resuelto por:

```text
DATABASE_TYPE_SYSTEM.md
```

---

# 42. PHP type metadata

También puede existir:

```text
PhpTypeMetadata
```

Ejemplo:

```text
PHP:
Email

Database:
VARCHAR
```

---

# 43. Conversion metadata

La relación:

```text
VARCHAR
↔
Email
```

deberá describirse mediante:

```text
Type/Value Conversion System
```

no mediante callbacks arbitrarios embebidos en metadata.

---

# 44. Nullability

Debe distinguirse:

```text
PHP_NULLABLE
DATABASE_NULLABLE
SEMANTIC_NULLABLE
```

cuando sea necesario.

---

# 45. Nullability mismatch

Ejemplo:

```php
private string $name;
```

contra:

```text
database column nullable
```

puede producir warning/error de mapping según política.

---

# 46. Generated fields

Metadata deberá distinguir:

```text
APPLICATION_GENERATED
DATABASE_GENERATED
COMPUTED
IDENTITY
SEQUENCE
TRIGGER_GENERATED
```

---

# 47. Insertability

Un campo podrá ser:

```text
INSERTABLE
NON_INSERTABLE
CONDITIONAL
```

---

# 48. Updatability

Igualmente:

```text
UPDATABLE
IMMUTABLE_AFTER_INSERT
NON_UPDATABLE
CONDITIONAL
```

---

# 49. Version metadata

Para optimistic locking:

```text
EntityVersionMetadata
```

Ejemplo:

```php
#[Version]
private int $version;
```

---

# 50. Version metadata ≠ entity version value

Metadata:

```text
field = version
strategy = INTEGER_INCREMENT
```

Runtime:

```text
version = 17
```

---

# 51. Version strategies

Podrán existir:

```text
INTEGER_INCREMENT
TIMESTAMP
TOKEN
CUSTOM
```

---

# 52. Relationship metadata

Cada relación deberá producir metadata tipada.

```text
RelationshipMetadata
```

---

# 53. Relationship kinds

```text
ONE_TO_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
POLYMORPHIC
CUSTOM
```

---

# 54. Relationship metadata model

Conceptualmente:

```php
interface RelationshipMetadata
{
    public function name(): RelationshipName;

    public function source(): EntityType;

    public function target(): EntityType;

    public function kind(): RelationshipKind;

    public function ownership(): RelationshipOwnership;

    public function loading(): RelationshipLoadingPolicy;
}
```

---

# 55. Relationship ownership

Deberá distinguir:

```text
OWNING
INVERSE
BIDIRECTIONAL
UNIDIRECTIONAL
```

según el tipo de mapping.

---

# 56. Relationship field ≠ scalar field

Nunca deberá tratarse:

```text
$user->organization
```

como si fuese simplemente:

```text
organization_id scalar property
```

aunque ambos estén relacionados.

---

# 57. Foreign key metadata

Relationship metadata podrá referenciar:

```text
JoinMetadata
```

con:

```text
local columns
referenced columns
nullable
cascade semantics
```

---

# 58. Join metadata ≠ Schema ForeignKey

Nuevamente:

```text
ORM JoinMetadata
≠
Schema ForeignKeyDefinition
```

aunque puedan validarse mutuamente.

---

# 59. Many-to-many

Podrá requerir:

```text
JoinTableMetadata
```

---

# 60. JoinTableMetadata

```text
JoinTableMetadata
├── table
├── source joins
├── target joins
└── uniqueness semantics
```

---

# 61. Relationship loading metadata

Puede declarar:

```text
LAZY
EAGER
EXPLICIT
BATCH
```

como política/default.

---

# 62. Loading metadata ≠ loading state

Metadata:

```text
posts → LAZY
```

Runtime:

```text
posts currently loaded = false
```

son diferentes.

---

# 63. Cascade metadata

Podrá existir:

```text
PERSIST
REMOVE
DETACH
REFRESH
MERGE
```

si la arquitectura adopta estas operaciones.

---

# 64. Cascade ≠ database cascade

Regla crítica:

```text
ORM Cascade
≠
ON DELETE CASCADE
```

---

# 65. Orphan removal

Se modelará explícitamente:

```text
OrphanRemovalPolicy
```

No se inferirá automáticamente de cascade remove.

---

# 66. Relationship inverses

Metadata deberá validar:

```text
User.posts
↔
Post.author
```

cuando se declare bidireccional.

---

# 67. Cross-entity resolution

Por ello, metadata no puede validarse siempre entidad por entidad de forma aislada.

Se necesita:

```text
CrossEntityMetadataResolver
```

---

# 68. Resolution pipeline

```text
User metadata draft
Post metadata draft
       │
       ▼
CrossEntityResolver
       │
       ├── resolve target
       ├── resolve inverse
       ├── validate ownership
       ├── validate join mapping
       └── detect conflicts
```

---

# 69. Cycles

Relaciones cíclicas:

```text
User
 ↓
Organization
 ↓
Owner
 ↓
User
```

son válidas.

El metadata builder deberá soportarlas sin depender de construcción recursiva infinita.

---

# 70. Two-phase metadata resolution

Se recomienda:

```text
Phase 1
Register entity shells

Phase 2
Resolve cross-entity references
```

---

# 71. EntityMetadataShell

Durante bootstrap puede existir:

```text
EntityMetadataShell
```

que contiene identidad básica antes de resolver relaciones.

---

# 72. Inheritance metadata

El ORM podrá soportar:

```text
EntityInheritanceMetadata
```

---

# 73. Inheritance strategies

Posibles:

```text
NONE
SINGLE_TABLE
JOINED
TABLE_PER_CLASS
CUSTOM
```

La implementación concreta podrá introducirse gradualmente.

---

# 74. Inheritance ≠ PHP inheritance

Una clase puede heredar en PHP sin necesariamente utilizar herencia ORM.

---

# 75. Discriminator metadata

Para algunas estrategias:

```text
DiscriminatorMetadata
├── column
├── type
└── value → EntityType map
```

---

# 76. Inheritance validation

Deberá detectar:

- discriminadores duplicados;
- tipos desconocidos;
- campos incompatibles;
- identificadores incompatibles;
- mappings ambiguos;
- cycles inválidos.

---

# 77. Instantiation metadata

El Hydration System necesita saber cómo crear entidades.

Se introduce:

```text
EntityInstantiationMetadata
```

---

# 78. Instantiation strategies

```text
CONSTRUCTOR
CONSTRUCTOR_BYPASS
FACTORY
STATIC_FACTORY
COMPILED_INSTANTIATOR
CUSTOM
```

---

# 79. Constructor semantics

Una entidad puede tener:

```php
public function __construct(
    Email $email,
    string $name,
) {}
```

pero hydration puede requerir una estrategia distinta.

---

# 80. Constructor bypass

Si se soporta:

```text
CONSTRUCTOR_BYPASS
```

deberá ser explícito y controlado.

No una suposición universal.

---

# 81. Entity factory

Metadata podrá referenciar:

```text
EntityInstantiatorId
```

en lugar de contener una closure viva.

---

# 82. No closures in compiled metadata

Preferencia fuerte:

> La metadata compilada no deberá depender de closures no serializables.

---

# 83. Hydration metadata

```text
EntityHydrationMetadata
```

describe cómo aplicar datos a una instancia.

---

# 84. Hydration plan inputs

Puede incluir:

```text
field accessors
constructor mapping
generated fields
relationship slots
readonly constraints
type conversion descriptors
```

---

# 85. Hydration metadata ≠ hydration plan runtime

Metadata describe posibilidades.

Un:

```text
HydrationPlan
```

puede ser específico de una consulta.

---

# 86. Ejemplo

Metadata:

```text
User:
id → id
name → name
email → email
```

Query:

```text
SELECT id, name
```

puede producir:

```text
PartialHydrationPlan
```

---

# 87. Persistence metadata

```text
EntityPersistenceMetadata
```

podrá describir:

```text
insertable fields
updatable fields
generated fields
version field
soft-delete capability
timestamp behavior
write restrictions
```

---

# 88. Persistence metadata ≠ persistence operation

Metadata:

```text
name is updatable
```

Runtime:

```text
UPDATE user SET name = ?
```

pertenece al Persistence Engine.

---

# 89. Change tracking metadata

Se introduce:

```text
ChangeTrackingMetadata
```

---

# 90. Strategies

Podrán existir:

```text
SNAPSHOT
NOTIFY
EXPLICIT
DEFERRED_EXPLICIT
CUSTOM
```

---

# 91. Metadata only

La metadata declara:

```text
strategy = SNAPSHOT
```

pero el snapshot real pertenece a:

```text
Entity Snapshot System
```

---

# 92. Lifecycle metadata

```text
EntityLifecycleMetadata
```

describe lifecycle callbacks/listeners.

---

# 93. Lifecycle phases

Podrán declararse eventos como:

```text
PRE_PERSIST
POST_PERSIST
PRE_UPDATE
POST_UPDATE
PRE_REMOVE
POST_REMOVE
POST_LOAD
PRE_FLUSH
POST_FLUSH
```

La semántica exacta será formalizada posteriormente.

---

# 94. Callback descriptor

Preferir:

```text
LifecycleCallbackDescriptor
```

sobre closures arbitrarias.

Ejemplo:

```text
event: PRE_PERSIST
method: initializeCreationTime
```

---

# 95. Lifecycle metadata ≠ Event Bus

Metadata únicamente describe handlers.

El Event/Lifecycle System decide ejecución.

---

# 96. Model API metadata

Para clases:

```php
class User extends Model
```

podrá existir:

```text
ModelApiMetadata
```

---

# 97. ModelApiMetadata

Puede incluir:

```text
fillable
guarded
hidden
visible
casts
scopes
timestamps
soft-delete behavior
serialization
model observers
```

---

# 98. Model metadata composition

```text
EntityMetadata
        │
        └── optional ModelApiMetadata
```

Una plain entity simplemente no tendrá esa sección.

---

# 99. Model API no redefine mapping

No deberá existir:

```text
Entity mapping A
+
Model mapping B
```

para la misma entidad.

Debe converger a una sola metadata ORM.

---

# 100. Metadata sources

VoltStack podrá soportar:

```text
PHP Attributes
Model Conventions
Programmatic Mapping
Package Metadata Providers
Compiled Metadata
Custom Extensions
```

---

# 101. Source precedence

La precedencia deberá ser:

```text
explicit
deterministic
documented
```

Nunca:

```text
last discovered wins
```

---

# 102. Metadata source contribution

Se propone:

```php
interface EntityMetadataContributor
{
    public function contribute(
        EntityMetadataDraft $metadata,
        MetadataBuildContext $context,
    ): void;
}
```

---

# 103. Contributor identity

Cada contributor deberá tener:

```text
ContributorId
priority/order
dependencies
version
```

---

# 104. Conflicting metadata

Ejemplo:

```text
Attribute:
table = users

Programmatic config:
table = accounts
```

sin regla explícita deberá producir:

```text
MetadataConflictException
```

---

# 105. No silent override

Regla:

```text
Conflict
≠
LastWins
```

---

# 106. Metadata builder

Se introduce:

```text
EntityMetadataBuilder
```

---

# 107. Builder responsibility

```text
Mapping declarations
      ↓
EntityMetadataBuilder
      ↓
EntityMetadataDraft
```

No valida necesariamente todas las relaciones cross-entity todavía.

---

# 108. Metadata normalizer

```text
EntityMetadataNormalizer
```

convierte distintas formas equivalentes a una representación canónica.

---

# 109. Ejemplo de normalización

```text
"int"
"integer"
PHP int
DatabaseType::INTEGER
```

pueden converger, cuando sea semánticamente válido, a:

```text
DatabaseTypeReference(INTEGER)
```

---

# 110. Normalization ≠ guessing

No deberá inferirse silenciosamente información peligrosa cuando sea ambigua.

---

# 111. Metadata resolver

```text
EntityMetadataResolver
```

resuelve:

- referencias entre entidades;
- inverses;
- identifier references;
- inherited mappings;
- type references;
- storage references;
- extension descriptors.

---

# 112. Metadata validator

```text
EntityMetadataValidator
```

deberá detectar errores antes del runtime normal.

---

# 113. Validation categories

```text
STRUCTURAL
IDENTITY
FIELD
TYPE
RELATIONSHIP
INHERITANCE
HYDRATION
PERSISTENCE
LIFECYCLE
MODEL_API
PLATFORM_COMPATIBILITY
```

---

# 114. Structural validation

Ejemplos:

- entidad sin identidad;
- field duplicado;
- relationship duplicada;
- property desconocida;
- column mapping inválido.

---

# 115. Identifier validation

Ejemplos:

- múltiples IDs incompatibles;
- composite ID incompleto;
- generated composite ID no soportado;
- nullable ID inválido según estrategia.

---

# 116. Field validation

Ejemplos:

```text
duplicate column mapping
invalid property
unknown type
invalid nullability
invalid generated strategy
```

---

# 117. Relationship validation

Ejemplos:

```text
unknown target entity
invalid inverse
invalid owning side
missing join information
incompatible identifiers
```

---

# 118. Hydration validation

Ejemplos:

```text
readonly property cannot be hydrated by configured strategy
constructor parameter unresolved
required field has no hydration path
```

---

# 119. Persistence validation

Ejemplo:

```text
field is required for INSERT
but marked non-insertable
and has no generation/default strategy
```

deberá fallar.

---

# 120. Model API validation

Ejemplos:

```text
fillable field does not exist
cast references unknown field
hidden relationship ambiguity
scope collision
```

---

# 121. Validation severity

Podrán existir:

```text
ERROR
WARNING
INFO
```

Pero invariantes estructurales no deberán degradarse a warning arbitrariamente.

---

# 122. Metadata diagnostics

Cada problema deberá incluir:

```text
code
entity
metadata path
source
message
reason
suggested remediation
```

---

# 123. Ejemplo

```text
DB-META-REL-004

Entity:
App\Entity\Post

Path:
relationships.author.inverse

Problem:
Inverse relationship "posts" does not exist on App\Entity\User.

Declared at:
Post.php:74
```

---

# 124. Source locations

La metadata de desarrollo podrá preservar:

```text
MetadataSourceLocation
```

con:

```text
file
line
attribute
contributor
package
```

---

# 125. Production metadata

Source locations detalladas podrán eliminarse/reducirse en builds optimizados si se desea ahorrar memoria.

---

# 126. Metadata compiler

Después de validación:

```text
EntityMetadataCompiler
```

transformará metadata normalizada en una representación optimizada.

---

# 127. Compilation goals

La compilación puede:

- resolver nombres;
- precomputar lookup maps;
- generar accessors;
- generar instantiators;
- generar hydration descriptors;
- generar persistence field sets;
- generar relationship lookup maps;
- calcular fingerprints;
- eliminar estructuras temporales.

---

# 128. Compiled metadata

```text
CompiledEntityMetadata
```

podrá ser la implementación runtime de `EntityMetadata`.

---

# 129. Lookup complexity

Operaciones frecuentes:

```text
fieldByName()
relationshipByName()
fieldByColumn()
identifierFields()
```

deberán favorecer:

```text
O(1)
```

mediante índices precomputados cuando sea razonable.

---

# 130. Metadata registry

Se introduce:

```text
EntityMetadataRegistry
```

---

# 131. Registry responsibility

Mapear:

```text
EntityType
→ EntityMetadata

class-string
→ EntityMetadata
```

---

# 132. Contrato

```php
interface EntityMetadataRegistry
{
    public function for(string $entityClass): EntityMetadata;

    public function has(string $entityClass): bool;

    public function all(): iterable;
}
```

---

# 133. Registry ≠ Service Container

El registry almacena metadata.

No servicios ORM arbitrarios.

---

# 134. Registry freezing

Después de bootstrap:

```text
BUILDING
   ↓
FROZEN
```

---

# 135. Frozen registry

En runtime:

```text
add metadata
remove metadata
replace metadata
```

deberán estar prohibidos salvo mecanismos explícitos de recompilación.

---

# 136. Duplicate registration

```text
same EntityType
+
different metadata
```

deberá producir error.

---

# 137. Aliases

Si se permiten aliases:

```text
UserInterface
→ User
```

deberán ser explícitos y no crear identidad ORM ambigua.

---

# 138. Metadata lookup

```php
$metadata = $registry->for(User::class);
```

deberá ser una operación barata.

---

# 139. Unknown entity

Debe lanzar:

```text
UnknownEntityMetadataException
```

---

# 140. Metadata fingerprint

Cada metadata compilada deberá tener:

```text
EntityMetadataFingerprint
```

---

# 141. Fingerprint purpose

Puede utilizarse para:

- cache invalidation;
- compiled metadata validation;
- telemetry;
- development hot reload;
- deployment consistency;
- query-plan dependencies;
- hydration-plan dependencies.

---

# 142. Fingerprint inputs

Conceptualmente:

```text
Fingerprint(
    EntityType,
    identifier,
    fields,
    relationships,
    inheritance,
    lifecycle,
    persistence,
    hydration,
    changeTracking,
    relevant extensions,
    metadata format version
)
```

---

# 143. Fingerprint exclusions

No deberá incluir:

```text
object addresses
timestamps of build
random values
current request
current tenant
current connection object
```

---

# 144. Determinism

Regla:

```text
same canonical metadata
+
same metadata format version
=
same fingerprint
```

---

# 145. Registry fingerprint

También podrá calcularse:

```text
EntityMetadataRegistryFingerprint
```

sobre el conjunto completo.

---

# 146. Registry fingerprint use

Útil para validar:

```text
compiled cache
query cache
worker consistency
deployment nodes
```

---

# 147. Metadata serialization

Compiled metadata deberá tener una representación serializable versionada.

---

# 148. No PHP serialize contract

No se recomienda depender de:

```php
serialize($metadata);
```

como formato estable de cache.

---

# 149. Versioned format

Preferir:

```text
MetadataCacheFormat v1
MetadataCacheFormat v2
...
```

---

# 150. Cache compatibility

El cache deberá incluir:

```text
framework version
metadata format version
registry fingerprint
environment/build identity
extension fingerprints
```

cuando sean relevantes.

---

# 151. Metadata cache

Pipeline de producción:

```text
Application Source
      ↓ build
Compiled Metadata Cache
      ↓ startup
Validation
      ↓
Frozen Registry
      ↓
Runtime
```

---

# 152. Development mode

Podrá utilizar:

```text
discover
build
validate
compile
```

dinámicamente.

---

# 153. Production mode

Deberá favorecer:

```text
load compiled metadata
verify
freeze
```

---

# 154. Cache invalidation

Cambios en:

```text
entity mapping
mapping attributes
metadata contributors
relevant config
extension versions
```

deberán invalidar metadata correspondiente.

---

# 155. Partial invalidation

Idealmente:

```text
changed User metadata
```

no obligará siempre a recompilar todas las entidades.

Pero dependientes cross-entity podrán requerir invalidación.

---

# 156. Dependency graph

Se podrá mantener:

```text
MetadataDependencyGraph
```

Ejemplo:

```text
Post.author
   ↓ depends on
User metadata
```

---

# 157. Dependency-aware invalidation

Si cambia:

```text
User identifier
```

puede invalidar metadata compilada de:

```text
Post.author
Comment.author
Organization.owner
```

---

# 158. Metadata dependency graph ≠ relationship graph

El dependency graph puede incluir:

```text
inheritance
types
extensions
embeddables
relationships
custom mappings
```

---

# 159. Platform compatibility

Algunas mappings pueden depender de capabilities.

Ejemplo:

```text
JSON type
generated columns
native enums
sequences
```

---

# 160. Platform-neutral metadata

La metadata ORM deberá ser tan platform-neutral como sea razonable.

---

# 161. Platform requirements

Puede declarar:

```text
EntityCapabilityRequirement
```

sin incrustar vendor checks.

---

# 162. Incorrecto

```php
if ($database === 'postgres') {
    // mapping
}
```

en metadata runtime.

---

# 163. Correcto

```text
field
requires capability:
NATIVE_JSON
```

o una representación portable.

---

# 164. Capability validation

Posteriormente:

```text
Metadata
    ↓
Capability Analyzer
    ↓
Platform Compatibility
```

---

# 165. Metadata ≠ schema portability guarantee

Que una entidad sea válida conceptualmente no significa que todo mapping sea soportado idénticamente en todos los motores.

---

# 166. MySQL / MariaDB

No deberán tratarse como una plataforma idéntica cuando sus capabilities difieran.

---

# 167. Schema validation

Opcionalmente:

```text
EntityMetadata
+
SchemaMetadata
→ MappingSchemaValidationReport
```

---

# 168. Mapping-schema validation

Puede detectar:

```text
missing table
missing column
type mismatch
nullable mismatch
missing FK
unexpected uniqueness
```

---

# 169. Mapping schema validation ≠ migrations

El sistema podrá reportar diferencias.

No deberá generar/ejecutar automáticamente una migración.

---

# 170. Integration with Schema Diff

Flujo posible:

```text
Entity Metadata
      ↓
Desired Schema Projection
      ↓
Schema Model
      ↓
Schema Diff
      ↓
Migration Candidate System
```

---

# 171. Desired schema projection

Podrá existir:

```text
EntitySchemaProjector
```

que proyecta metadata ORM a `SchemaModel`.

---

# 172. Projection ≠ metadata

```text
EntityMetadata
```

contiene semántica ORM.

```text
SchemaModel
```

contiene estructura DB.

No son intercambiables.

---

# 173. Example

Entity metadata:

```text
User.posts:
ONE_TO_MANY
inverse of Post.author
lazy
cascade persist
```

Schema projection quizá solo necesite:

```text
posts.author_id
FK → users.id
```

Parte de la semántica ORM no existe físicamente en el schema.

---

# 174. Embeddables

El sistema podrá soportar:

```text
EmbeddableMetadata
```

---

# 175. Ejemplo

```php
final class Address
{
    private string $street;
    private string $city;
}
```

embebido en:

```php
final class Customer
{
    private Address $address;
}
```

---

# 176. Embeddable ≠ Entity

Un embeddable normalmente:

```text
has no independent EntityKey
has no independent IdentityMap entry
has no independent repository
```

---

# 177. Embedded mapping

Puede proyectarse:

```text
address.street → address_street
address.city   → address_city
```

---

# 178. Value Object metadata

No todo value object deberá ser embeddable.

Un value object puede mapearse mediante:

```text
single-column custom type
multi-column embeddable
JSON
custom converter
```

---

# 179. Metadata category separation

Se recomienda distinguir:

```text
EntityMetadata
EmbeddableMetadata
MappedSuperclassMetadata
ValueObjectMappingMetadata
```

---

# 180. Mapped superclass

Puede proporcionar:

```text
fields
lifecycle
behaviors
```

sin ser entidad consultable independiente.

---

# 181. Repository metadata

La entidad podrá declarar:

```text
RepositoryClass
```

o:

```text
RepositoryServiceId
```

---

# 182. Repository metadata ≠ repository instance

No almacenar:

```text
new UserRepository(...)
```

dentro de metadata.

---

# 183. Entity query metadata

Podrá existir metadata necesaria para:

```text
field resolution
relationship path resolution
identifier shortcut resolution
default ordering
scope integration
```

---

# 184. Query aliases

Los aliases runtime:

```text
u
p
o
```

no pertenecen a EntityMetadata.

---

# 185. Metadata and Semantic Query Engine

```text
EntityQuery
    ↓
EntityMetadata
    ↓
Semantic field resolution
    ↓
Query AST
```

---

# 186. Field resolution example

```php
User::query()
    ->where('email', $email);
```

deberá resolver:

```text
"email"
   ↓
EntityFieldMetadata
   ↓
ColumnReference
   ↓
Database type
```

---

# 187. Relationship path resolution

```text
posts.author.email
```

puede resolverse:

```text
User.posts
    ↓
Post.author
    ↓
User.email
```

mediante metadata graph.

---

# 188. Unknown field

Debe fallar antes de generar SQL:

```text
UnknownEntityFieldException
```

---

# 189. Metadata performance

La metadata se utilizará en hot paths.

Por ello deberá ser:

```text
immutable
indexed
compact
pre-resolved
cache-friendly
```

---

# 190. No repeated reflection

Objetivo:

```text
Runtime Reflection Cost ≈ 0
```

en producción optimizada.

---

# 191. Precomputed maps

Ejemplo:

```text
fieldsByName
fieldsByColumn
relationshipsByName
identifierFields
generatedFields
insertableFields
updatableFields
```

---

# 192. Memory considerations

Con miles de entidades, duplicar strings/arrays innecesariamente puede ser costoso.

---

# 193. Interning

Podrá considerarse internar:

```text
type references
capability IDs
common descriptors
logical connection names
```

si benchmarks lo justifican.

---

# 194. Optimization rule

No sacrificar claridad arquitectónica por micro-optimizaciones no medidas.

---

# 195. Metadata inspection API

Desarrolladores y tooling podrán consultar:

```php
$metadata = Database::metadata(User::class);
```

---

# 196. Inspection result

Podrá exponer:

```text
table
identifier
fields
relations
types
casts
generation strategies
lifecycle
```

---

# 197. Inspection read-only

Tooling nunca deberá poder mutar metadata frozen mediante referencias retornadas.

---

# 198. CLI

Futuro CLI:

```text
volt database:metadata User
```

podrá mostrar:

```text
Entity: App\Entity\User
Table: users
Identifier: id
Fields: 12
Relations: 4
Change tracking: snapshot
```

---

# 199. Metadata explain

Podrá existir:

```text
volt database:metadata:explain User.email
```

para mostrar:

```text
PHP property
ORM field
DB column
type
conversion
source
```

---

# 200. Debugging

Metadata diagnostics deberá integrarse con:

```text
Database Debug Information System
Developer Debug Toolbar
Telemetry
```

posteriormente.

---

# 201. Telemetry

La metadata puede emitir métricas de bootstrap:

```text
entity metadata count
build duration
validation duration
compile duration
cache hits
cache misses
invalid entities
```

---

# 202. No high-cardinality entity leakage

Telemetry deberá controlar cardinalidad cuando existan miles de tipos generados/dinámicos.

---

# 203. Persistent runtime

EntityMetadata es excelente candidata para estado compartido entre requests porque será:

```text
immutable
validated
compiled
tenant-neutral where possible
```

---

# 204. Shared state

```text
Worker
│
├── Frozen EntityMetadataRegistry
├── Compiled accessors
├── Compiled instantiators
└── Metadata indexes
```

pueden sobrevivir múltiples requests.

---

# 205. Request-scoped state

Nunca deberán vivir dentro de metadata:

```text
Entity instance
IdentityMap
UnitOfWork
Transaction
Connection
TenantContext
QueryContext
current user
request
```

---

# 206. Tenant-neutral metadata

Idealmente:

```text
User → logical table users
```

y el Tenant System resuelve:

```text
tenant
→ database/schema/connection
```

fuera de metadata compartida.

---

# 207. Tenant-specific mappings

Si existen mappings realmente distintos por tenant, deberán utilizar:

```text
MetadataVariant
```

o un mecanismo explícito.

No mutar metadata compartida.

---

# 208. MetadataVariantKey

Podrá representar:

```text
mapping profile
schema version
tenant class
feature set
```

pero deberá evitar cardinalidad ilimitada.

---

# 209. Runtime isolation

Regla:

```text
Frozen Metadata
can be shared.

Mutable ORM State
cannot be shared.
```

---

# 210. FrankenPHP

```text
Worker
├── MetadataRegistry ──────────────┐
│                                 │ shared immutable
├── Request A                     │
│   ├── EntityManager A           │
│   └── UnitOfWork A              │
│                                 │
└── Request B                     │
    ├── EntityManager B           │
    └── UnitOfWork B              │
                                  │
MetadataRegistry ◄────────────────┘
```

---

# 211. RoadRunner

Misma regla:

```text
worker shared metadata
+
request scoped entity state
```

---

# 212. OpenSwoole

Metadata immutable puede compartirse entre coroutines.

No deberá contener objetos mutable-request-aware.

---

# 213. Thread/process considerations

Compiled metadata deberá evitar dependencias en recursos no serializables cuando se pretenda preloading/cache cross-process.

---

# 214. OPcache/preloading

La arquitectura deberá ser compatible con:

```text
PHP OPcache
preloading
immutable compiled PHP metadata
```

cuando resulte beneficioso.

---

# 215. Compiled PHP representation

Podría generarse:

```php
return new CompiledEntityMetadata(
    // precomputed immutable descriptors
);
```

o una representación array optimizada que posteriormente sea materializada.

---

# 216. Generated accessors

El compiler podrá producir accessors equivalentes a:

```php
static fn (User $entity) => $entity->email;
```

pero preferentemente mediante código compilable/cacheable.

---

# 217. Private properties

El sistema deberá soportar propiedades privadas sin exigir hacerlas públicas únicamente por necesidades del ORM.

---

# 218. Accessor safety

Los accessors compilados deberán respetar:

```text
type
visibility strategy
readonly semantics
initialization state
```

---

# 219. Readonly entities

PHP readonly introduce restricciones especiales.

Metadata deberá detectar si la estrategia de hydration/persistence es compatible.

---

# 220. Uninitialized properties

Metadata/Hydration deberán distinguir:

```text
null
```

de:

```text
uninitialized
```

en propiedades tipadas PHP.

---

# 221. Uninitialized ≠ null

Regla:

```text
UninitializedProperty
≠
PropertyWithNullValue
```

---

# 222. Partial hydration

Esto será especialmente importante para partial entities.

La metadata deberá proporcionar suficiente información para no escribir accidentalmente campos nunca cargados.

---

# 223. Field loadability

Podrá existir:

```text
FieldHydrationPolicy
```

con:

```text
REQUIRED
OPTIONAL
DEFERRED
GENERATED
```

---

# 224. Partial state belongs elsewhere

Metadata describe si un campo puede cargarse parcialmente.

El estado:

```text
field X is loaded
```

pertenece a la entidad managed/runtime state.

---

# 225. Metadata extensions

Se introduce:

```text
EntityMetadataExtension
```

---

# 226. Extension capabilities

Una extensión podrá contribuir:

```text
custom field metadata
custom relationship metadata
custom type descriptors
custom lifecycle descriptors
custom behavior metadata
```

---

# 227. Extension namespace

Custom metadata deberá utilizar IDs namespaced.

Ejemplo:

```text
acme.search.index
voltstack.soft_delete
voltstack.audit
```

---

# 228. Extension bag

Puede existir:

```text
MetadataExtensionBag
```

inmutable.

---

# 229. Extension restrictions

Una extensión no podrá:

- ejecutar SQL durante metadata build;
- abrir Connection;
- leer current request;
- leer current user;
- mutar frozen metadata;
- ocultar conflictos;
- reemplazar metadata silenciosamente.

---

# 230. Deterministic extension order

El orden deberá depender de:

```text
explicit dependencies
priority
stable extension ID
```

---

# 231. Extension collision

Dos extensiones con el mismo ID incompatible deberán fallar.

---

# 232. Metadata source provenance

Cada fragmento podrá preservar:

```text
MetadataProvenance
```

---

# 233. Provenance example

```text
Entity:
User

Field:
email

Declared by:
PHP Attribute

Source:
src/Entity/User.php:34

Normalized by:
CoreFieldMetadataNormalizer

Extended by:
EmailValueObjectExtension
```

---

# 234. Provenance benefits

Facilita:

- debugging;
- conflict detection;
- IDE tooling;
- explainability;
- migration generation;
- diagnostics.

---

# 235. Metadata validation modes

Podrán existir:

```text
FAST
FULL
BUILD
RUNTIME_GUARD
```

---

# 236. BUILD

Realiza validación profunda durante deployment/build.

---

# 237. FAST

Puede utilizar metadata cache previamente verificada.

---

# 238. RUNTIME_GUARD

No sustituye build validation.

Solo protege invariantes críticas si el cache está corrupto/incompatible.

---

# 239. Fail-fast

Errores de metadata deberán detectarse idealmente:

```text
deployment/build/startup
```

no durante la primera petición de un usuario.

---

# 240. Development hot reload

En desarrollo podrá permitirse reconstruir metadata cuando cambien archivos.

---

# 241. Hot reload ≠ in-place mutation

Correcto:

```text
old Frozen Registry
       ↓
build new Registry
       ↓
validate
       ↓
atomic replace development context
```

Incorrecto:

```text
modify old metadata objects in place
```

---

# 242. Metadata generations

Puede existir:

```text
MetadataGenerationId
```

para identificar generaciones durante development hot reload.

---

# 243. Existing EntityManagers

Un EntityManager creado bajo generación A no deberá mezclarse arbitrariamente con metadata generación B.

---

# 244. Metadata generation consistency

```text
EntityManager
→ MetadataRegistryGeneration
```

deberá permanecer estable durante su scope.

---

# 245. Metadata and migrations

Cuando metadata cambia:

```text
Metadata A
→ Metadata B
```

eso no significa automáticamente:

```text
Database migrated
```

---

# 246. Deployment invariant

Debe mantenerse conscientemente:

```text
Application Metadata
↔
Database Schema
```

dentro de una ventana de compatibilidad.

---

# 247. Zero-downtime implications

Durante despliegues expand/migrate/contract:

```text
new application metadata
```

puede necesitar ser compatible temporalmente con:

```text
old schema
intermediate schema
new schema
```

---

# 248. Compatibility profiles

Podrá existir:

```text
MetadataSchemaCompatibilityProfile
```

para tooling de despliegue.

---

# 249. Metadata does not own deployment

El Metadata System únicamente describe y valida.

`Zero Downtime Migration System` gobierna estrategia operacional.

---

# 250. Security

Metadata puede contener información sensible de arquitectura.

Debug endpoints no deberán exponerla indiscriminadamente.

---

# 251. Sensitive metadata

Ejemplos:

```text
table names
column names
connection aliases
encryption configuration
hidden fields
security mappings
```

---

# 252. Production diagnostics

Deberán aplicar redacción/políticas de exposición.

---

# 253. Metadata and encryption

Un campo cifrado puede declarar:

```text
EncryptionMappingDescriptor
```

pero claves criptográficas nunca deberán almacenarse en metadata.

---

# 254. Secret-free metadata

Regla:

```text
Metadata
must not contain credentials or secrets.
```

---

# 255. Testing architecture

El sistema deberá tener una suite específica de metadata.

---

# 256. Basic metadata tests

Cubrir:

```text
single entity
single identifier
multiple fields
custom columns
types
nullability
generated fields
```

---

# 257. Identifier tests

Cubrir:

```text
assigned ID
auto increment
UUID
ULID
sequence
composite ID
invalid ID
```

---

# 258. Relationship tests

Cubrir:

```text
one-to-one
one-to-many
many-to-one
many-to-many
bidirectional
unidirectional
invalid inverse
unknown target
cycles
```

---

# 259. Inheritance tests

Cubrir:

```text
single table
joined
discriminator
duplicate discriminator
invalid inherited ID
```

---

# 260. Hydration metadata tests

Cubrir:

```text
constructor
constructor bypass
private fields
readonly fields
uninitialized properties
custom instantiator
```

---

# 261. Persistence metadata tests

Cubrir:

```text
insertable
non-insertable
updatable
generated
versioned
immutable field
```

---

# 262. Change tracking tests

Cubrir todas las estrategias declaradas.

---

# 263. Model API metadata tests

Cubrir:

```text
fillable
guarded
casts
hidden
visible
scopes
timestamps
soft deletes
```

---

# 264. Compilation tests

Verificar:

```text
Draft
→ Normalize
→ Resolve
→ Validate
→ Compile
→ Freeze
```

---

# 265. Determinism tests

Misma entrada canónica deberá producir:

```text
same compiled semantics
same fingerprint
```

---

# 266. Source-order tests

Cambiar el orden físico de descubrimiento de fuentes equivalentes no deberá alterar metadata final cuando la semántica sea la misma.

---

# 267. Conflict tests

Conflictos deberán fallar determinísticamente.

---

# 268. Cache tests

Cubrir:

```text
valid cache
stale cache
wrong framework version
wrong format version
changed entity
changed extension
corrupt cache
```

---

# 269. Cross-entity invalidation tests

Modificar identifier de entidad target deberá invalidar metadata dependiente cuando corresponda.

---

# 270. Persistent runtime tests

Simular múltiples requests usando el mismo registry:

```text
Request A
Request B
Request C
```

y verificar que metadata permanezca immutable.

---

# 271. Concurrency tests

Múltiples coroutines podrán leer metadata simultáneamente sin locks de mutación.

---

# 272. No-I/O tests

Metadata build/lookup no deberá abrir conexiones salvo una fase explícita de schema compatibility validation solicitada por el caller.

---

# 273. Property-based tests

Generar combinaciones válidas e inválidas de metadata para comprobar invariantes.

---

# 274. Round-trip metadata tests

Cuando exista formato compilado:

```text
Canonical Metadata
→ Serialize
→ Load
→ Metadata
```

deberá preservar semántica y fingerprint.

---

# 275. Error hierarchy

```text
DatabaseOrmException
└── EntityMetadataException
    ├── EntityMetadataBuildException
    ├── EntityMetadataNormalizationException
    ├── EntityMetadataResolutionException
    ├── EntityMetadataValidationException
    ├── EntityMetadataCompilationException
    ├── EntityMetadataCacheException
    ├── EntityMetadataFingerprintException
    ├── UnknownEntityMetadataException
    ├── DuplicateEntityMetadataException
    ├── EntityMetadataConflictException
    ├── EntityIdentifierMetadataException
    ├── EntityFieldMetadataException
    ├── EntityRelationshipMetadataException
    ├── EntityInheritanceMetadataException
    ├── EntityHydrationMetadataException
    ├── EntityPersistenceMetadataException
    ├── EntityLifecycleMetadataException
    ├── EntityChangeTrackingMetadataException
    ├── EntityMetadataExtensionException
    ├── EntityMetadataGenerationException
    └── EntityMetadataInvariantException
```

---

# 276. Namespace propuesto

```text
VoltStack\Quantum\Database\ORM\Metadata
```

---

# 277. Estructura propuesta

```text
src/Quantum/Database/ORM/Metadata/
│
├── Contract/
│   ├── EntityMetadata.php
│   ├── EntityMetadataRegistry.php
│   ├── EntityMetadataContributor.php
│   └── EntityMetadataExtension.php
│
├── Entity/
│   ├── EntityType.php
│   ├── EntityTypeId.php
│   ├── EntityClassMetadata.php
│   └── EntityStorageMetadata.php
│
├── Identifier/
│   ├── EntityIdentifierMetadata.php
│   ├── EntityIdentifierKind.php
│   └── IdentifierGenerationStrategy.php
│
├── Field/
│   ├── EntityFieldMetadata.php
│   ├── EntityFieldMetadataCollection.php
│   ├── FieldName.php
│   ├── FieldMutability.php
│   └── PropertyReference.php
│
├── Column/
│   └── ColumnMetadata.php
│
├── Relationship/
│   ├── RelationshipMetadata.php
│   ├── RelationshipMetadataCollection.php
│   ├── RelationshipKind.php
│   ├── RelationshipOwnership.php
│   ├── JoinMetadata.php
│   └── JoinTableMetadata.php
│
├── Inheritance/
│   ├── EntityInheritanceMetadata.php
│   ├── InheritanceStrategy.php
│   └── DiscriminatorMetadata.php
│
├── Instantiation/
│   ├── EntityInstantiationMetadata.php
│   └── EntityInstantiationStrategy.php
│
├── Hydration/
│   └── EntityHydrationMetadata.php
│
├── Persistence/
│   └── EntityPersistenceMetadata.php
│
├── ChangeTracking/
│   └── ChangeTrackingMetadata.php
│
├── Lifecycle/
│   ├── EntityLifecycleMetadata.php
│   └── LifecycleCallbackDescriptor.php
│
├── Version/
│   └── EntityVersionMetadata.php
│
├── Model/
│   └── ModelApiMetadata.php
│
├── Embeddable/
│   └── EmbeddableMetadata.php
│
├── Build/
│   ├── EntityMetadataBuilder.php
│   ├── EntityMetadataDraft.php
│   └── MetadataBuildContext.php
│
├── Normalize/
│   └── EntityMetadataNormalizer.php
│
├── Resolve/
│   ├── EntityMetadataResolver.php
│   └── CrossEntityMetadataResolver.php
│
├── Validation/
│   ├── EntityMetadataValidator.php
│   ├── MetadataValidationReport.php
│   └── MetadataValidationIssue.php
│
├── Compile/
│   ├── EntityMetadataCompiler.php
│   └── CompiledEntityMetadata.php
│
├── Registry/
│   ├── DefaultEntityMetadataRegistry.php
│   └── EntityMetadataRegistryFingerprint.php
│
├── Fingerprint/
│   └── EntityMetadataFingerprint.php
│
├── Cache/
│   ├── EntityMetadataCache.php
│   └── MetadataCacheFormat.php
│
├── Dependency/
│   └── MetadataDependencyGraph.php
│
├── Source/
│   ├── MetadataSource.php
│   ├── MetadataSourceLocation.php
│   └── MetadataProvenance.php
│
├── Extension/
│   ├── MetadataExtensionRegistry.php
│   └── MetadataExtensionBag.php
│
└── Exception/
    └── ...
```

---

# 278. Dependencias permitidas

```text
Entity Metadata
    ↓
Type contracts
Schema references
Platform capability contracts
Support/value objects
```

---

# 279. Dependencias prohibidas

El Metadata Core no deberá depender de:

```text
EntityManager runtime state
UnitOfWork runtime state
IdentityMap
Query Executor
PDO
live Connection
HTTP Request
current Tenant
current User
```

---

# 280. Dependency direction

```text
EntityManager
      ↓
EntityMetadata

UnitOfWork
      ↓
EntityMetadata

Hydrator
      ↓
EntityMetadata

EntityQuery
      ↓
EntityMetadata

PersistenceEngine
      ↓
EntityMetadata
```

Nunca:

```text
EntityMetadata
      ↓
EntityManager
```

---

# 281. Architectural invariant

La metadata se encuentra:

```text
below ORM runtime behavior
```

como descripción.

No encima como coordinador.

---

# 282. Invariantes

## DB-ENTITY-META-001
Toda entidad persistible deberá poseer metadata válida.

## DB-ENTITY-META-002
EntityMetadata será distinta de una instancia Entity.

## DB-ENTITY-META-003
EntityMetadata no almacenará estado de una entidad viva.

## DB-ENTITY-META-004
EntityMetadata no almacenará snapshots runtime.

## DB-ENTITY-META-005
EntityMetadata no implementará UnitOfWork.

## DB-ENTITY-META-006
EntityMetadata no implementará IdentityMap.

## DB-ENTITY-META-007
EntityMetadata no ejecutará queries.

## DB-ENTITY-META-008
EntityMetadata no generará SQL.

## DB-ENTITY-META-009
EntityMetadata no ejecutará SQL.

## DB-ENTITY-META-010
EntityMetadata no abrirá conexiones.

## DB-ENTITY-META-011
Compiled metadata será immutable.

## DB-ENTITY-META-012
Frozen registry será immutable.

## DB-ENTITY-META-013
Metadata Draft será distinto de frozen metadata.

## DB-ENTITY-META-014
Mapping declaration será distinta de compiled metadata.

## DB-ENTITY-META-015
Reflection será una fuente, no la representación runtime obligatoria.

## DB-ENTITY-META-016
EntityType será distinto de EntityIdentifier.

## DB-ENTITY-META-017
EntityType será conceptualmente distinto de PHP class-string.

## DB-ENTITY-META-018
EntityType será distinto de table name.

## DB-ENTITY-META-019
Identifier metadata será distinta de identifier value.

## DB-ENTITY-META-020
Single y composite identifiers serán explícitos.

## DB-ENTITY-META-021
Identifier generation strategy será metadata declarativa.

## DB-ENTITY-META-022
Field será distinto de PHP property.

## DB-ENTITY-META-023
Field será distinto de database column.

## DB-ENTITY-META-024
Property será distinta de database column.

## DB-ENTITY-META-025
Field names deberán ser únicos dentro de su entidad.

## DB-ENTITY-META-026
Relationship names deberán ser únicos dentro de su entidad.

## DB-ENTITY-META-027
Ambigüedad field/relationship deberá detectarse.

## DB-ENTITY-META-028
Unknown types deberán fallar durante validation cuando sea posible.

## DB-ENTITY-META-029
Nullability incompatible deberá diagnosticarse.

## DB-ENTITY-META-030
Generated field strategy será explícita.

## DB-ENTITY-META-031
Insertability será explícitamente representable.

## DB-ENTITY-META-032
Updatability será explícitamente representable.

## DB-ENTITY-META-033
Version metadata será distinta del valor version runtime.

## DB-ENTITY-META-034
Relationship metadata será distinta de relationship loading state.

## DB-ENTITY-META-035
Relationship target deberá resolverse a EntityType conocido.

## DB-ENTITY-META-036
Bidirectional relationships deberán validar sus inverses.

## DB-ENTITY-META-037
ORM cascade será distinto de database cascade.

## DB-ENTITY-META-038
Orphan removal será explícito.

## DB-ENTITY-META-039
Join metadata será distinta de Schema ForeignKeyDefinition.

## DB-ENTITY-META-040
ORM ColumnMetadata será distinta de Schema ColumnDefinition.

## DB-ENTITY-META-041
Cross-entity cycles válidos no deberán causar construcción recursiva infinita.

## DB-ENTITY-META-042
Cross-entity resolution podrá utilizar múltiples fases.

## DB-ENTITY-META-043
Inheritance ORM será distinta de PHP inheritance.

## DB-ENTITY-META-044
Discriminator values deberán ser únicos dentro de su jerarquía.

## DB-ENTITY-META-045
Instantiation strategy será explícita.

## DB-ENTITY-META-046
Constructor bypass no será implícito universal.

## DB-ENTITY-META-047
Compiled metadata no dependerá de closures arbitrarias cuando pueda evitarse.

## DB-ENTITY-META-048
Hydration metadata será distinta de query-specific HydrationPlan.

## DB-ENTITY-META-049
Persistence metadata será distinta de PersistenceOperation.

## DB-ENTITY-META-050
ChangeTracking metadata será distinta del snapshot real.

## DB-ENTITY-META-051
Lifecycle metadata será distinta del Event Bus.

## DB-ENTITY-META-052
ModelApiMetadata será opcional.

## DB-ENTITY-META-053
Plain entities no requerirán ModelApiMetadata.

## DB-ENTITY-META-054
Model API y Entity mapping convergerán a una sola metadata ORM.

## DB-ENTITY-META-055
Metadata source precedence será determinista.

## DB-ENTITY-META-056
Conflictos no usarán last-wins silencioso.

## DB-ENTITY-META-057
Metadata contributors tendrán identidad estable.

## DB-ENTITY-META-058
Metadata normalization no convertirá ambigüedad en certeza.

## DB-ENTITY-META-059
Metadata validation deberá ocurrir antes de hot runtime cuando sea posible.

## DB-ENTITY-META-060
Errores estructurales de metadata no deberán ocultarse como warnings arbitrarios.

## DB-ENTITY-META-061
Metadata diagnostics preservarán suficiente provenance.

## DB-ENTITY-META-062
Metadata compiler no cambiará semántica válida.

## DB-ENTITY-META-063
Compiled metadata podrá precomputar índices.

## DB-ENTITY-META-064
Field lookup deberá ser eficiente.

## DB-ENTITY-META-065
Relationship lookup deberá ser eficiente.

## DB-ENTITY-META-066
Registry mapeará class/entity type de forma no ambigua.

## DB-ENTITY-META-067
Unknown entity metadata producirá error explícito.

## DB-ENTITY-META-068
Duplicate entity metadata producirá error explícito.

## DB-ENTITY-META-069
Registry podrá congelarse.

## DB-ENTITY-META-070
Frozen registry no aceptará mutaciones runtime arbitrarias.

## DB-ENTITY-META-071
Metadata fingerprints serán deterministas.

## DB-ENTITY-META-072
Fingerprint no dependerá de tiempo de compilación.

## DB-ENTITY-META-073
Fingerprint no dependerá de object identity.

## DB-ENTITY-META-074
Fingerprint no dependerá del request actual.

## DB-ENTITY-META-075
Fingerprint no dependerá del tenant actual salvo variant explícita.

## DB-ENTITY-META-076
Metadata cache tendrá formato versionado.

## DB-ENTITY-META-077
Metadata cache no dependerá de PHP serialize como contrato estable obligatorio.

## DB-ENTITY-META-078
Cache incompatible deberá invalidarse.

## DB-ENTITY-META-079
Metadata dependency graph permitirá invalidación dependiente.

## DB-ENTITY-META-080
Metadata dependency graph será distinto del relationship graph.

## DB-ENTITY-META-081
Metadata será platform-neutral cuando sea razonable.

## DB-ENTITY-META-082
Platform-specific requirements utilizarán capability semantics.

## DB-ENTITY-META-083
Version string no sustituirá capability detection.

## DB-ENTITY-META-084
MySQL y MariaDB podrán poseer capabilities diferentes.

## DB-ENTITY-META-085
Mapping-schema validation será distinta de migration execution.

## DB-ENTITY-META-086
EntityMetadata será distinta de SchemaModel.

## DB-ENTITY-META-087
EntitySchemaProjector no destruirá semántica ORM original.

## DB-ENTITY-META-088
Embeddable será distinto de Entity.

## DB-ENTITY-META-089
Embeddable no tendrá EntityKey independiente por default.

## DB-ENTITY-META-090
ValueObject mapping será distinto de Entity identity.

## DB-ENTITY-META-091
Repository metadata no almacenará repository instance.

## DB-ENTITY-META-092
Query aliases runtime no pertenecerán a EntityMetadata.

## DB-ENTITY-META-093
Unknown entity fields fallarán antes de SQL generation cuando sea posible.

## DB-ENTITY-META-094
Metadata será optimizable para hot paths.

## DB-ENTITY-META-095
Production podrá operar sin reflection repetitiva.

## DB-ENTITY-META-096
Metadata inspection será read-only.

## DB-ENTITY-META-097
Metadata diagnostics respetarán políticas de exposición.

## DB-ENTITY-META-098
Metadata no contendrá credenciales.

## DB-ENTITY-META-099
Metadata no contendrá encryption keys.

## DB-ENTITY-META-100
Immutable metadata podrá compartirse entre requests.

## DB-ENTITY-META-101
Entity instances no se almacenarán en shared metadata.

## DB-ENTITY-META-102
IdentityMap no se almacenará en metadata.

## DB-ENTITY-META-103
UnitOfWork no se almacenará en metadata.

## DB-ENTITY-META-104
Transaction state no se almacenará en metadata.

## DB-ENTITY-META-105
Connection state no se almacenará en metadata.

## DB-ENTITY-META-106
TenantContext no se almacenará en metadata compartida.

## DB-ENTITY-META-107
QueryContext no se almacenará en metadata.

## DB-ENTITY-META-108
Metadata variants no mutarán la variante base.

## DB-ENTITY-META-109
Metadata variants tendrán identidad explícita.

## DB-ENTITY-META-110
Metadata variant cardinality deberá ser gobernable.

## DB-ENTITY-META-111
FrankenPHP podrá reutilizar frozen metadata entre requests.

## DB-ENTITY-META-112
RoadRunner podrá reutilizar frozen metadata entre requests.

## DB-ENTITY-META-113
OpenSwoole podrá reutilizar frozen metadata entre coroutines si es immutable.

## DB-ENTITY-META-114
Metadata no mantendrá referencias request-aware.

## DB-ENTITY-META-115
Compiled metadata podrá ser compatible con OPcache/preloading.

## DB-ENTITY-META-116
Private properties podrán mapearse.

## DB-ENTITY-META-117
Readonly semantics deberán validarse.

## DB-ENTITY-META-118
Uninitialized property será distinta de null.

## DB-ENTITY-META-119
Partial hydration state no pertenecerá a static metadata.

## DB-ENTITY-META-120
Metadata extensions estarán namespaced.

## DB-ENTITY-META-121
Extension registry será determinista.

## DB-ENTITY-META-122
Extension collisions producirán error.

## DB-ENTITY-META-123
Extensions no podrán ejecutar SQL durante metadata build.

## DB-ENTITY-META-124
Extensions no podrán abrir conexiones durante metadata build ordinario.

## DB-ENTITY-META-125
Extensions no podrán leer current user como mapping source.

## DB-ENTITY-META-126
Extensions no podrán ocultar UNKNOWN/conflicts.

## DB-ENTITY-META-127
Metadata provenance será preservable.

## DB-ENTITY-META-128
Build validation podrá ser más profunda que runtime guards.

## DB-ENTITY-META-129
Hot reload reemplazará generaciones, no mutará metadata frozen in-place.

## DB-ENTITY-META-130
EntityManager utilizará una generación consistente de metadata durante su scope.

## DB-ENTITY-META-131
Cambiar metadata no significa que DB schema haya cambiado.

## DB-ENTITY-META-132
Cambiar metadata no ejecutará migrations automáticamente.

## DB-ENTITY-META-133
Metadata deberá poder coexistir con zero-downtime schema windows.

## DB-ENTITY-META-134
Metadata schema compatibility será verificable.

## DB-ENTITY-META-135
Metadata no será responsable de deployment orchestration.

## DB-ENTITY-META-136
Metadata no será responsable de transaction orchestration.

## DB-ENTITY-META-137
Metadata no será responsable de entity lifecycle execution.

## DB-ENTITY-META-138
Metadata no será responsable de hydration execution.

## DB-ENTITY-META-139
Metadata no será responsable de persistence execution.

## DB-ENTITY-META-140
Metadata no será responsable de change detection execution.

## DB-ENTITY-META-141
Metadata será la fuente canónica de descripción ORM de una entidad.

## DB-ENTITY-META-142
EntityManager consultará metadata en vez de inferir mapping ad hoc.

## DB-ENTITY-META-143
UnitOfWork consultará metadata en vez de inferir identidad ad hoc.

## DB-ENTITY-META-144
Hydrator consultará metadata en vez de descubrir propiedades repetidamente.

## DB-ENTITY-META-145
Persistence Engine consultará metadata para fields persistibles.

## DB-ENTITY-META-146
EntityQuery utilizará metadata para resolución semántica.

## DB-ENTITY-META-147
Model API utilizará la misma EntityMetadata que plain entities.

## DB-ENTITY-META-148
Metadata compilada deberá preservar semántica de metadata validada.

## DB-ENTITY-META-149
Metadata cache corruption deberá detectarse y no aceptarse silenciosamente.

## DB-ENTITY-META-150
Entity Metadata describirá el modelo persistente sin convertirse en estado, comportamiento runtime o motor de persistencia.

---

# 283. Anti-patterns

## 283.1 Metadata con entidad viva

Incorrecto:

```php
final class EntityMetadata
{
    public ?object $currentEntity = null;
}
```

---

## 283.2 Metadata con Connection

Incorrecto:

```php
final class EntityMetadata
{
    public Connection $connection;
}
```

---

## 283.3 Metadata con tenant actual

Incorrecto:

```php
$metadata->table = 'tenant_' . Tenant::current()->id . '_users';
```

---

## 283.4 Reflection en cada query

Incorrecto:

```text
User::query()
    ↓
ReflectionClass(User)
    ↓
scan attributes
    ↓
build mapping
```

por cada consulta.

---

## 283.5 Last-wins mapping

Incorrecto:

```text
Attribute says "users"
Config says "accounts"

Result:
accounts because loaded last
```

Debe existir conflicto o precedencia explícita.

---

## 283.6 Relationship como scalar FK

Incorrecto:

```text
Post.author
=
Post.author_id
```

Son conceptos relacionados, no idénticos.

---

## 283.7 ORM cascade como DB cascade

Incorrecto:

```text
cascade REMOVE
=
ON DELETE CASCADE
```

---

## 283.8 Metadata como SchemaModel

Incorrecto:

```text
EntityMetadata == TableDefinition
```

---

## 283.9 Closures arbitrarias en metadata cache

Evitar:

```php
$metadata->hydrator = function (...) {};
```

si impide compilación, serialización o preloading.

---

## 283.10 Mutable frozen registry

Incorrecto:

```php
$registry->replace(
    User::class,
    $newMetadata,
);
```

durante una petición ordinaria.

---

## 283.11 Runtime vendor checks

Incorrecto:

```php
if ($connection->driverName() === 'mysql') {
    $metadata->type = ...;
}
```

---

## 283.12 Secrets en metadata

Nunca:

```text
database passwords
API secrets
encryption keys
```

---

# 284. Ejemplo de entidad

```php
#[Entity]
#[Table('users')]
final class User
{
    #[Id]
    #[GeneratedValue('ulid')]
    #[Column(type: 'ulid')]
    private UserId $id;

    #[Column(length: 150)]
    private string $name;

    #[Column(type: 'email')]
    private Email $email;

    #[Version]
    #[Column]
    private int $version;

    #[OneToMany(
        target: Post::class,
        mappedBy: 'author',
    )]
    private Collection $posts;
}
```

---

# 285. Metadata resultante

Conceptualmente:

```text
EntityMetadata<User>
│
├── type
│   └── app.user
│
├── class
│   └── App\Entity\User
│
├── storage
│   └── users
│
├── identifier
│   ├── id
│   └── ULID
│
├── fields
│   ├── id
│   │   ├── property: id
│   │   ├── column: id
│   │   └── type: ulid
│   │
│   ├── name
│   │   ├── column: name
│   │   └── length: 150
│   │
│   ├── email
│   │   ├── column: email
│   │   └── type: email
│   │
│   └── version
│       ├── column: version
│       └── optimistic version
│
└── relationships
    └── posts
        ├── ONE_TO_MANY
        ├── target: Post
        └── inverse: author
```

---

# 286. Uso por EntityManager

```text
EntityManager::find(User::class, $id)
        │
        ▼
EntityMetadataRegistry
        │
        ▼
EntityMetadata<User>
        │
        ├── identifier metadata
        ├── storage metadata
        └── hydration metadata
```

---

# 287. Uso por UnitOfWork

```text
UnitOfWork
    │
    ▼
EntityMetadata<User>
    │
    ├── identity fields
    ├── change tracking strategy
    ├── version field
    └── persistence fields
```

---

# 288. Uso por EntityQuery

```text
where('email', $email)
        │
        ▼
EntityMetadata<User>
        │
        ▼
FieldMetadata(email)
        │
        ├── column: email
        └── type: EmailType
```

---

# 289. Uso por Hydrator

```text
Database Row
    │
    ▼
HydrationPlan
    │
    ▼
EntityMetadata
    │
    ├── instantiation
    ├── field access
    └── type conversion
    │
    ▼
User
```

---

# 290. Uso por Persistence Engine

```text
Managed User
    │
    ▼
UnitOfWork ChangeSet
    │
    ▼
EntityMetadata
    │
    ├── updatable fields
    ├── columns
    ├── types
    ├── generated fields
    └── version metadata
    │
    ▼
Persistence Plan
```

---

# 291. Fórmula de metadata

```text
EntityMetadata
=
IdentityDescription
+
FieldDescription
+
RelationshipDescription
+
TypeMapping
+
InstantiationDescription
+
HydrationDescription
+
PersistenceDescription
+
ChangeTrackingDescription
+
LifecycleDescription
+
OptionalModelApiDescription
+
ExtensionMetadata
```

---

# 292. Fórmula de construcción

```text
EntityMetadata
=
Freeze(
    Compile(
        Validate(
            Resolve(
                Normalize(
                    Build(
                        MappingSources
                    )
                )
            )
        )
    )
)
```

---

# 293. Fórmula de determinismo

Para entradas equivalentes:

```text
CanonicalSources₁ = CanonicalSources₂
```

deberá cumplirse:

```text
Metadata₁ ≡ Metadata₂
```

y:

```text
Fingerprint₁ = Fingerprint₂
```

---

# 294. Fórmula de runtime

```text
ORM Runtime
=
FrozenMetadata
+
ScopedEntityState
+
ScopedEntityManager
+
ScopedIdentityMap
+
ScopedUnitOfWork
```

donde:

```text
FrozenMetadata
```

puede compartirse y:

```text
ScopedEntityState
```

no.

---

# 295. Fórmula de seguridad de persistent runtime

```text
SafeMetadataSharing
=
ImmutableMetadata
∧
NoEntityInstances
∧
NoRequestState
∧
NoTenantState
∧
NoConnectionState
∧
NoTransactionState
∧
NoMutableGlobalExtensions
```

---

# 296. Fórmula de compilación

```text
CompiledMetadata
=
CanonicalMetadata
+
ResolvedReferences
+
PrecomputedIndexes
+
CompiledAccessStrategies
+
ValidatedInvariants
+
DeterministicFingerprint
```

---

# 297. Fórmula de coherencia ORM

```text
EntityManager
        │
UnitOfWork
        │
Hydrator
        ├──→ same EntityMetadata
Persistence
        │
EntityQuery
        │
Model API
```

Todos deberán interpretar la entidad mediante la misma fuente canónica.

---

# 298. Regla maestra

> **VoltStack tendrá una única representación canónica de cómo el ORM entiende cada entidad. EntityManager, UnitOfWork, Hydration, Persistence, Relationships, EntityQuery y Model API deberán consumir esa misma metadata en lugar de reconstruir o reinterpretar el mapping independientemente.**

---

# 299. Resultado arquitectónico

Con este sistema:

```text
PHP Entity
   ↓
Mapping
   ↓
EntityMetadata
```

se convierte en la frontera estable entre:

```text
declaración del dominio
```

y:

```text
infraestructura ORM
```

permitiendo que:

```text
EntityManager
UnitOfWork
IdentityMap
Hydration
Persistence
EntityQuery
Relationships
Model API
```

compartan una interpretación única de cada entidad.

---

# 300. Integración con documentos anteriores

```text
112_DATABASE_ORM_ARCHITECTURE
        │
        ▼
defines ORM boundaries

113_DATABASE_ENTITY_MODEL
        │
        ▼
defines what an Entity is

114_DATABASE_MODEL_API_SYSTEM
        │
        ▼
defines ergonomic Model API

115_DATABASE_ENTITY_METADATA_SYSTEM
        │
        ▼
defines how ORM understands entities
```

---

# 301. Integración con documentos siguientes

La metadata aquí definida será consumida directamente por:

```text
116_DATABASE_ENTITY_MAPPING_SYSTEM
        │
        ▼
mapping sources → metadata

117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM
        │
        ▼
PHP attributes → mapping declarations

118_DATABASE_ENTITY_MANAGER_SYSTEM
        │
        ▼
runtime entity coordination

119_DATABASE_REPOSITORY_SYSTEM
        │
        ▼
entity collection access

120_DATABASE_ENTITY_QUERY_SYSTEM
        │
        ▼
metadata-aware queries

121_DATABASE_ENTITY_STATE_SYSTEM
        │
        ▼
runtime state classification

122_DATABASE_ENTITY_LIFECYCLE_SYSTEM
        │
        ▼
entity state transitions
```

y posteriormente:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE
125_DATABASE_CHANGE_TRACKING_SYSTEM
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM
127_DATABASE_PERSISTENCE_ENGINE
```

---

# 302. Frontera definitiva

```text
                     IMMUTABLE DESCRIPTION
                             │
                             ▼
                    EntityMetadata
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
    EntityManager         Hydrator          EntityQuery
          │                  │                  │
          ▼                  ▼                  ▼
      UnitOfWork          Entities          Query Engine
          │
          ▼
 Persistence Engine
```

La dirección es:

```text
Runtime Systems
      ↓ consume
EntityMetadata
```

Nunca:

```text
EntityMetadata
      ↓ controls
Runtime Systems
```

---

# 303. Master Formula

```text
Database Entity Metadata System
=
Canonical Entity Description
+
Stable Entity Type Identity
+
Identifier Metadata
+
Field and Column Mapping Metadata
+
Relationship Metadata
+
Inheritance Metadata
+
Instantiation Metadata
+
Hydration Metadata
+
Persistence Metadata
+
Change Tracking Metadata
+
Lifecycle Metadata
+
Model API Metadata
+
Multi-Source Mapping Resolution
+
Normalization
+
Cross-Entity Resolution
+
Validation
+
Compilation
+
Deterministic Fingerprinting
+
Dependency-Aware Caching
+
Frozen Registry
+
Extension Governance
+
Persistent Runtime Safety
```

---

# 304. Master Rule

> **Entity Metadata tells VoltStack what an entity means to the ORM. It may be discovered, normalized, resolved, validated, compiled, cached and shared, but it never becomes the entity itself, never owns runtime entity state and never performs database work.**

---

# 305. Siguiente documento

```text
116_DATABASE_ENTITY_MAPPING_SYSTEM.md
```

El siguiente documento deberá formalizar cómo VoltStack transforma distintas fuentes declarativas en la metadata canónica definida aquí, incluyendo:

```text
Entity Mapping Architecture
Mapping Sources
Mapping Declarations
Mapping Source Registry
Mapping Profiles
Mapping Resolution
Mapping Precedence
Mapping Composition
Mapping Overrides
Mapping Conflicts
Entity-to-Table Mapping
Field-to-Column Mapping
Identifier Mapping
Relationship Mapping
Embedded Mapping
Inheritance Mapping
Generated Value Mapping
Version Mapping
Lifecycle Mapping
Persistence Mapping
Hydration Mapping
Change Tracking Mapping
Mapping Validation
Mapping Normalization
Mapping Compilation
Mapping Fingerprints
Mapping Cache
Mapping Extensions
Package Mapping
Persistent Runtime Safety
```

manteniendo la regla:

> **Mapping declara cómo el dominio se relaciona con la persistencia; Metadata es la representación canónica resultante.**