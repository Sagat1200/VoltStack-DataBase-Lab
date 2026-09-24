# 116_DATABASE_ENTITY_MAPPING_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Mapping System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 116 — Database Entity Mapping System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Mapping System` define cómo VoltStack transforma la descripción del dominio PHP en una representación persistente comprensible por el ORM.

Su responsabilidad central es establecer correspondencias explícitas entre:

```text
PHP Domain Model
        ↕
ORM Semantic Model
        ↕
Relational Persistence Model
```

Ejemplos:

```text
User                    → Entity
User::$id               → Identifier
User::$email            → Persistent Field
User::$organization     → Relationship
User::$createdAt        → Column
Address                  → Embeddable / Value Object
User::$version           → Optimistic Lock Version
```

El sistema deberá aceptar múltiples fuentes declarativas sin permitir que cada una cree su propia interpretación del ORM.

Principio central:

> **Mapping declara cómo el dominio se relaciona con la persistencia; Metadata es la representación canónica resultante.**

Formalmente:

```text
MappingSource
      ↓
MappingDeclaration
      ↓
Mapping Resolution
      ↓
Normalized Mapping
      ↓
EntityMetadata
```

---

# 2. Mapping no es Metadata

La distinción fundamental es:

```text
Entity Mapping
≠
Entity Metadata
```

Mapping representa:

```text
declaración / intención
```

Metadata representa:

```text
interpretación ORM canónica, resuelta y validada
```

Ejemplo:

```php
#[Column(type: 'string', length: 150)]
private string $name;
```

es mapping declarativo.

Después del procesamiento puede producir:

```text
EntityFieldMetadata
├── field: name
├── property: User::$name
├── column: name
├── database type: STRING
├── PHP type: string
├── length: 150
├── nullable: false
├── insertable: true
└── updatable: true
```

---

# 3. Mapping no es Schema

Otra frontera crítica:

```text
Entity Mapping
≠
Schema Definition
```

Ejemplo:

```php
#[Column(length: 150)]
private string $name;
```

describe la persistencia de:

```text
User::$name
```

No constituye directamente:

```text
ALTER TABLE users ...
```

La cadena correcta es:

```text
Entity Mapping
      ↓
Entity Metadata
      ↓
Desired Schema Projection
      ↓
Schema Model
      ↓
Schema Diff
      ↓
Migration Candidate
      ↓
Migration Planner
```

---

# 4. Mapping no ejecuta

El sistema no deberá:

- abrir conexiones;
- ejecutar SQL;
- generar SQL;
- ejecutar migraciones;
- consultar tablas;
- hidratar entidades;
- persistir entidades;
- administrar EntityManager;
- mantener IdentityMap;
- detectar cambios;
- ejecutar lifecycle callbacks.

---

# 5. Posición arquitectónica

```text
PHP Domain
│
├── Attributes
├── Conventions
├── Programmatic Mapping
├── Package Mapping
└── Extensions
        │
        ▼
Entity Mapping System
        │
        ├── Discovery
        ├── Source Registry
        ├── Declaration Collection
        ├── Composition
        ├── Conflict Detection
        ├── Resolution
        ├── Normalization
        └── Validation
        │
        ▼
Entity Metadata Builder
        │
        ▼
Entity Metadata System
```

---

# 6. Objetivos

El sistema deberá proporcionar:

- modelo de mapping tipado;
- múltiples fuentes de mapping;
- composición determinista;
- precedencia explícita;
- detección de conflictos;
- entity mapping;
- identifier mapping;
- field mapping;
- column mapping;
- relationship mapping;
- embedded mapping;
- inheritance mapping;
- generation mapping;
- version mapping;
- lifecycle mapping;
- persistence mapping;
- hydration mapping;
- change-tracking mapping;
- mapping profiles;
- extensibilidad;
- provenance;
- fingerprints;
- caching;
- validación;
- compatibilidad con persistent runtimes.

---

# 7. Principios

## 7.1 Declarativo

Mapping describe intención.

No realiza trabajo de persistencia.

## 7.2 Determinista

Mismas fuentes canónicas deberán producir el mismo mapping.

## 7.3 Componible

Distintas fuentes podrán aportar partes compatibles.

## 7.4 Conflict-aware

Las contradicciones no deberán resolverse silenciosamente.

## 7.5 Source-aware

VoltStack deberá poder explicar de dónde provino cada declaración.

## 7.6 Metadata-oriented

El resultado final del mapping será consumido por el Metadata System.

---

# 8. Arquitectura general

```text
                 Mapping Sources
                       │
      ┌────────────────┼─────────────────┐
      │                │                 │
      ▼                ▼                 ▼
 PHP Attributes    Conventions      Programmatic
      │                │                 │
      └────────────────┼─────────────────┘
                       │
                       ▼
              MappingSourceRegistry
                       │
                       ▼
             Mapping Discovery
                       │
                       ▼
           MappingDeclarationSet
                       │
                       ▼
            Mapping Composition
                       │
                       ▼
           Conflict Resolution
                       │
                       ▼
            Mapping Resolution
                       │
                       ▼
              Normalization
                       │
                       ▼
               Validation
                       │
                       ▼
          Canonical Entity Mapping
                       │
                       ▼
           EntityMetadataBuilder
```

---

# 9. Core abstractions

Se proponen:

```text
EntityMappingSource
EntityMappingSourceId
EntityMappingSourceRegistry

EntityMappingDeclaration
EntityMappingDeclarationSet

EntityMapping
EntityMappingBuilder

EntityMappingComposer
EntityMappingResolver
EntityMappingNormalizer
EntityMappingValidator
EntityMappingCompiler

EntityMappingProfile
EntityMappingContext

EntityMappingFingerprint
EntityMappingProvenance
```

---

# 10. Mapping Source

Contrato conceptual:

```php
interface EntityMappingSource
{
    public function id(): EntityMappingSourceId;

    public function supports(
        EntityMappingCandidate $candidate,
        EntityMappingContext $context,
    ): bool;

    public function declarations(
        EntityMappingCandidate $candidate,
        EntityMappingContext $context,
    ): iterable;
}
```

---

# 11. MappingSource ≠ MetadataContributor

Ambos pueden colaborar, pero conceptualmente:

```text
MappingSource
      ↓
MappingDeclaration

MetadataContributor
      ↓
Metadata
```

El primero pertenece a la fase declarativa.

El segundo extiende la metadata resultante.

---

# 12. Mapping source taxonomy

VoltStack podrá soportar:

```text
ATTRIBUTE
CONVENTION
PROGRAMMATIC
PACKAGE
GENERATED
COMPILED
EXTENSION
```

---

# 13. PHP Attributes

Ejemplo:

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
}
```

El Attribute Mapping System será desarrollado específicamente en:

```text
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
```

---

# 14. Convention Mapping

VoltStack podrá reducir configuración mediante convenciones.

Ejemplo:

```php
final class User extends Model
{
}
```

podría inferir:

```text
Entity:
User

Table:
users

Identifier:
id
```

si la política correspondiente está habilitada.

---

# 15. Convention ≠ Guessing

Una convención deberá ser:

```text
documented
deterministic
bounded
overrideable
diagnosable
```

Nunca:

```text
heuristic magic without provenance
```

---

# 16. Programmatic Mapping

También deberá ser posible:

```php
$mapping->entity(User::class)
    ->table('users')
    ->id('id')
    ->field('name')
        ->column('name')
        ->string(150);
```

La API concreta podrá evolucionar, pero deberá producir las mismas declaraciones semánticas.

---

# 17. Programmatic mapping use cases

Útil para:

- paquetes;
- código generado;
- entidades de terceros;
- legacy systems;
- tests;
- mappings dinámicamente ensamblados en build;
- casos donde attributes no sean deseables.

---

# 18. Package Mapping

Un paquete VoltStack podrá registrar mappings:

```text
Package
   ↓
MappingProvider
   ↓
MappingSourceRegistry
```

Ejemplo:

```text
voltstack/audit
    ↓
AuditEntry entity mapping
```

---

# 19. Package mapping isolation

Un paquete no deberá modificar silenciosamente mappings de otra entidad.

Toda contribución cross-package deberá ser explícita.

---

# 20. Mapping Provider

```php
interface EntityMappingProvider
{
    public function registerMappings(
        EntityMappingSourceRegistry $registry,
    ): void;
}
```

La implementación real deberá respetar el bootstrap general de VoltStack.

---

# 21. Mapping source identity

Cada fuente deberá poseer identidad estable:

```text
EntityMappingSourceId
```

Ejemplos:

```text
voltstack.orm.attributes
voltstack.orm.conventions
app.orm.explicit
acme.billing.mapping
```

---

# 22. Source ID ≠ Priority

Identidad y precedencia serán conceptos separados.

---

# 23. Mapping precedence

Cuando varias fuentes describan una entidad, deberá existir política explícita.

Ejemplo conceptual:

```text
Convention
    ↓
Attribute
    ↓
Explicit Programmatic Mapping
```

Pero esta precedencia deberá configurarse/formalizarse.

---

# 24. Precedence ≠ Silent Override

Que una fuente tenga mayor precedencia no significa que toda contradicción pueda ignorarse.

Podrán existir:

```text
OVERRIDABLE
NON_OVERRIDABLE
MERGEABLE
EXCLUSIVE
```

para distintas propiedades de mapping.

---

# 25. Ejemplo

Convención:

```text
User → users
```

Attribute:

```php
#[Table('accounts')]
```

Puede ser un override legítimo:

```text
users
   ↓ explicit override
accounts
```

---

# 26. Conflicto real

Dos fuentes explícitas:

```text
Source A:
User → customers

Source B:
User → accounts
```

sin relación de precedencia válida deberán producir:

```text
EntityMappingConflictException
```

---

# 27. Mapping declaration

La unidad declarativa básica será:

```text
EntityMappingDeclaration
```

---

# 28. Declaration model

Conceptualmente:

```php
interface EntityMappingDeclaration
{
    public function entity(): EntityReference;

    public function path(): MappingPath;

    public function source(): EntityMappingSourceId;

    public function provenance(): MappingProvenance;
}
```

---

# 29. MappingPath

Ejemplos:

```text
entity.table

entity.identifier.id

entity.fields.email.column

entity.fields.email.type

entity.relationships.posts.target

entity.lifecycle.prePersist
```

---

# 30. Mapping path purpose

Permite:

- merge preciso;
- detección de conflictos;
- provenance;
- diagnostics;
- override granular;
- fingerprints;
- debugging.

---

# 31. Declaration granularity

Una fuente no deberá necesariamente reemplazar una entidad completa.

Puede contribuir:

```text
User.email.type
```

sin redefinir:

```text
User.table
User.id
User.posts
```

---

# 32. Declaration set

```text
EntityMappingDeclarationSet
```

representará todas las declaraciones recopiladas para una entidad.

---

# 33. Collection phase

```text
Entity
   ↓
Sources
   ↓
Declaration Collector
   ↓
DeclarationSet
```

Todavía no implica que las declaraciones sean compatibles.

---

# 34. Mapping provenance

Cada declaración deberá poder conservar:

```text
source ID
source type
file
line
attribute
package
provider
priority
mapping profile
```

cuando aplique.

---

# 35. Provenance example

```text
Entity:
App\Entity\User

Path:
fields.email.column

Value:
email_address

Source:
voltstack.orm.attributes

File:
src/Entity/User.php

Line:
41
```

---

# 36. Entity Mapping

Después de composición y normalización se producirá:

```text
EntityMapping
```

---

# 37. EntityMapping model

Conceptualmente:

```php
final readonly class EntityMapping
{
    public function __construct(
        public EntityReference $entity,
        public EntityStorageMapping $storage,
        public EntityIdentifierMapping $identifier,
        public EntityFieldMappingCollection $fields,
        public EntityRelationshipMappingCollection $relationships,
        public EntityMappingOptions $options,
        public EntityMappingFingerprint $fingerprint,
    ) {}
}
```

---

# 38. Mapping lifecycle

```text
DISCOVERED
    ↓
COLLECTED
    ↓
COMPOSED
    ↓
RESOLVED
    ↓
NORMALIZED
    ↓
VALIDATED
    ↓
COMPILED
    ↓
FROZEN
```

---

# 39. Mapping Draft

Durante procesamiento podrá existir:

```text
EntityMappingDraft
```

mutable.

---

# 40. Draft ≠ Canonical Mapping

```text
EntityMappingDraft
≠
EntityMapping
```

---

# 41. Entity mapping

La raíz deberá declarar al menos:

```text
entity class/type
persistence eligibility
storage mapping
identifier strategy
```

según el tipo de entidad.

---

# 42. Entity classification

Podrán existir:

```text
ENTITY
MODEL
EMBEDDABLE
MAPPED_SUPERCLASS
VALUE_OBJECT
PROJECTION
```

No todos tendrán las mismas reglas de persistencia.

---

# 43. Entity-to-table mapping

Ejemplo:

```text
App\Entity\User
      ↓
users
```

representado mediante:

```text
EntityStorageMapping
```

---

# 44. Storage mapping

Puede contener:

```text
logical table
logical schema
logical catalog
logical connection
storage options
```

---

# 45. Logical names

Preferir:

```text
TableReference
LogicalConnectionName
SchemaReference
```

sobre strings libres internamente.

---

# 46. Mapping ≠ physical connection resolution

La declaración:

```text
connection = customers
```

no significa:

```text
open Connection(customers)
```

durante mapping.

---

# 47. Identifier mapping

```text
EntityIdentifierMapping
```

deberá declarar:

```text
field(s)
identifier kind
generation strategy
generator reference
```

---

# 48. Single identifier

```php
#[Id]
private UserId $id;
```

---

# 49. Composite identifier

```text
OrderItem
├── orderId
└── productId
```

deberá representarse explícitamente.

---

# 50. Identifier inference

Las convenciones podrán inferir:

```text
$id
```

como identificador.

Pero una declaración explícita deberá poder reemplazar esa convención.

---

# 51. Generated value mapping

```text
IdentifierGenerationMapping
```

podrá representar:

```text
ASSIGNED
AUTO_INCREMENT
SEQUENCE
UUID
ULID
DATABASE_GENERATED
CUSTOM
```

---

# 52. Generator reference

Un custom generator deberá referenciar:

```text
IdentifierGeneratorId
```

no necesariamente una instancia viva.

---

# 53. Field mapping

Cada field persistente tendrá:

```text
EntityFieldMapping
```

---

# 54. Field mapping example

```text
PHP:
User::$createdAt

ORM:
createdAt

Database:
created_at

Type:
datetime_immutable
```

---

# 55. Field mapping model

```php
final readonly class EntityFieldMapping
{
    public function __construct(
        public FieldName $field,
        public PropertyReference $property,
        public ColumnMapping $column,
        public TypeMapping $type,
        public FieldMappingOptions $options,
    ) {}
}
```

---

# 56. Property mapping

No se deberá asumir:

```text
field name = property name
```

aunque sea la convención default.

---

# 57. Column mapping

```text
ColumnMapping
```

puede declarar:

```text
column name
length
precision
scale
nullability
default semantics
generated semantics
insertability
updatability
```

---

# 58. ColumnMapping ≠ ColumnDefinition

Reiteramos:

```text
ORM ColumnMapping
≠
Schema ColumnDefinition
```

---

# 59. Type mapping

Deberá relacionar:

```text
PHP representation
       ↕
ORM type
       ↕
Database representation
```

---

# 60. Example

```text
Email object
    ↓
email type
    ↓
VARCHAR
```

---

# 61. Type inference

Cuando PHP declare:

```php
private int $age;
```

VoltStack podrá inferir un tipo ORM compatible.

---

# 62. Type inference ≠ final platform type

Inferir:

```text
PHP int → INTEGER semantic type
```

no significa seleccionar inmediatamente:

```text
INT
BIGINT
INTEGER
```

específico de un vendor.

---

# 63. Explicit type wins where permitted

```php
#[Column(type: 'unsigned_big_integer')]
private int $counter;
```

podrá aportar semántica más específica.

---

# 64. Nullability mapping

Podrán intervenir:

```text
PHP type
Attribute declaration
Database mapping
Default policy
```

---

# 65. Nullability conflict

Ejemplo:

```php
private string $name;

#[Column(nullable: true)]
```

deberá producir una decisión explícita:

```text
valid with warning
invalid
requires hydration policy
```

según reglas configuradas.

Nunca ignorarse.

---

# 66. Default values

Se deberán distinguir:

```text
PHP PROPERTY DEFAULT
APPLICATION DEFAULT
DATABASE DEFAULT
GENERATED DEFAULT
```

---

# 67. Default ≠ same semantics

Ejemplo:

```php
private bool $active = true;
```

no implica necesariamente:

```sql
DEFAULT TRUE
```

en el schema.

---

# 68. Generated column mapping

Podrá declarar:

```text
DATABASE_GENERATED
COMPUTED
IDENTITY
TRIGGER_GENERATED
```

sin ejecutar ni interpretar el mecanismo físico durante mapping.

---

# 69. Insert/update behavior

Mapping deberá poder declarar:

```text
insertable
updatable
immutable after insert
generated after insert
generated after update
```

---

# 70. Relationship mapping

Se introduce:

```text
EntityRelationshipMapping
```

---

# 71. Relationship types

```text
ONE_TO_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
POLYMORPHIC
CUSTOM
```

---

# 72. Relationship mapping model

Conceptualmente:

```php
interface EntityRelationshipMapping
{
    public function name(): RelationshipName;

    public function source(): EntityReference;

    public function target(): EntityReference;

    public function kind(): RelationshipKind;

    public function ownership(): RelationshipOwnership;
}
```

---

# 73. Relationship property

Ejemplo:

```php
private Organization $organization;
```

podrá mapear:

```text
User.organization
       ↓
ManyToOne
       ↓
Organization
```

---

# 74. Join mapping

```text
Relationship
    ↓
JoinMapping
```

puede declarar:

```text
local column
referenced field/column
nullable
uniqueness
```

---

# 75. Join mapping semantic references

Preferir:

```text
target identifier field
```

cuando sea posible, en vez de duplicar nombres físicos manualmente.

---

# 76. Example

```text
Post.author
    ↓
target: User
    ↓
target identifier: User.id
```

El resolver puede derivar:

```text
author_id → users.id
```

según mapping.

---

# 77. Explicit join override

El desarrollador podrá declarar:

```text
writer_uuid → accounts.user_uuid
```

cuando el schema legacy lo requiera.

---

# 78. Owning side

Mapping deberá establecer claramente qué lado gobierna la asociación persistente.

---

# 79. Inverse side

Ejemplo:

```text
User.posts
mappedBy:
Post.author
```

---

# 80. mappedBy / inversedBy

Estos conceptos deberán convertirse a referencias tipadas:

```text
RelationshipReference
```

y no permanecer únicamente como strings sin validar.

---

# 81. Cross-entity resolution

Antes:

```text
mappedBy = "author"
```

Después:

```text
RelationshipReference(
    entity = Post,
    relationship = author
)
```

---

# 82. Many-to-many mapping

Deberá soportar:

```text
join table
source join
target join
ownership
loading
cascade
```

---

# 83. Join table mapping

```text
User.roles
      ↓
user_roles
├── user_id
└── role_id
```

---

# 84. Cascade mapping

Mapping podrá declarar ORM cascades:

```text
PERSIST
REMOVE
DETACH
REFRESH
```

según operaciones finalmente soportadas.

---

# 85. Cascade mapping ≠ referential action

```text
cascade REMOVE
≠
ON DELETE CASCADE
```

---

# 86. Orphan removal

Debe ser declaración separada:

```text
orphanRemoval = true
```

---

# 87. Loading mapping

Relaciones podrán declarar preferencias:

```text
LAZY
EAGER
EXPLICIT
BATCH
```

---

# 88. Mapping loading policy ≠ runtime state

```text
mapping says LAZY
```

no significa:

```text
relation is currently unloaded
```

---

# 89. Embedded mapping

VoltStack deberá soportar:

```text
EmbeddedMapping
```

---

# 90. Example

```php
final class Customer
{
    private Address $address;
}
```

con:

```text
Address.street
Address.city
Address.postalCode
```

---

# 91. Flattened embedding

Puede mapear:

```text
address.street     → address_street
address.city       → address_city
address.postalCode → address_postal_code
```

---

# 92. Prefix mapping

Podrá existir:

```text
prefix = "address_"
```

---

# 93. Nested embedding

Debe poder modelarse:

```text
Customer
└── Address
    └── GeoPoint
```

si el Type/Embedding System lo soporta.

---

# 94. Embedded cycle

Los cycles de embeddables deberán detectarse.

Ejemplo inválido:

```text
A embeds B
B embeds A
```

sin una semántica explícita que lo permita.

---

# 95. Value object mapping

Un value object podrá mapearse como:

```text
single column
embedded columns
JSON
custom database type
```

---

# 96. Value object mapping ≠ relationship

```text
Email
Money
Address
```

no deberán convertirse automáticamente en entities relacionadas.

---

# 97. Inheritance mapping

Se introduce:

```text
EntityInheritanceMapping
```

---

# 98. Strategies

```text
NONE
SINGLE_TABLE
JOINED
TABLE_PER_CLASS
CUSTOM
```

---

# 99. Discriminator mapping

Para estrategias que lo requieran:

```text
DiscriminatorMapping
├── column
├── type
└── entity map
```

---

# 100. Mapped superclass

Un mapped superclass podrá aportar:

```text
fields
relationships where valid
lifecycle mappings
version mappings
```

sin convertirse en entity independiente.

---

# 101. Mapping inheritance composition

```text
Base Mapping
     ↓
Inheritance Composer
     ↓
Child Mapping
```

deberá resolver explícitamente:

- inherited fields;
- overrides;
- identifiers;
- lifecycle;
- table semantics;
- conflicts.

---

# 102. Field override

Podrá permitirse:

```text
Inherited field
      ↓
AttributeOverride
      ↓
Different column
```

si la estrategia lo soporta.

---

# 103. Relationship override

Igualmente podrá existir:

```text
AssociationOverride
```

para casos soportados.

---

# 104. Override ≠ mutation of parent mapping

El child mapping recibe una representación compuesta.

El parent mapping permanece inmutable.

---

# 105. Version mapping

Para optimistic locking:

```text
EntityVersionMapping
```

---

# 106. Example

```php
#[Version]
#[Column]
private int $version;
```

---

# 107. Version mapping validation

Deberá validar:

```text
single version field
compatible type
updatable semantics
generated semantics
```

---

# 108. Change tracking mapping

La entidad podrá declarar:

```text
SNAPSHOT
NOTIFY
EXPLICIT
DEFERRED_EXPLICIT
CUSTOM
```

---

# 109. Mapping does not track

La declaración:

```text
changeTracking = SNAPSHOT
```

no crea snapshots.

---

# 110. Hydration mapping

Podrá declarar:

```text
constructor strategy
property access strategy
factory
constructor parameter mapping
readonly handling
```

---

# 111. Constructor parameter mapping

Ejemplo:

```php
public function __construct(
    UserId $id,
    Email $email,
) {}
```

puede producir:

```text
constructor.id    ← field id
constructor.email ← field email
```

---

# 112. Persistence mapping

Podrá declarar:

```text
insert behavior
update behavior
generated fields
immutable fields
version behavior
soft delete integration
```

---

# 113. Soft delete mapping

Si está soportado:

```text
softDeleteField = deletedAt
```

deberá ser metadata declarativa.

La implementación detallada corresponde posteriormente a:

```text
269_DATABASE_SOFT_DELETE_SYSTEM.md
```

---

# 114. Timestamp mapping

Model API podrá aportar convenciones:

```text
created_at
updated_at
```

pero deberán convertirse a mappings ORM explícitos.

---

# 115. Lifecycle mapping

Podrán declararse callbacks:

```text
PRE_PERSIST
POST_PERSIST
PRE_UPDATE
POST_UPDATE
PRE_REMOVE
POST_REMOVE
POST_LOAD
```

---

# 116. Lifecycle declaration

Ejemplo:

```php
#[PrePersist]
public function initializeCreatedAt(): void
{
}
```

produce un:

```text
LifecycleMappingDeclaration
```

No ejecuta el método.

---

# 117. Mapping Profiles

VoltStack podrá soportar:

```text
EntityMappingProfile
```

---

# 118. Purpose

Un profile permite seleccionar un conjunto coherente de políticas/mappings.

Ejemplos:

```text
default
legacy
testing
analytics
```

---

# 119. Profile ≠ Tenant

No utilizar profiles como sustituto informal de Multitenancy.

---

# 120. Profile identity

```text
EntityMappingProfileId
```

deberá ser estable.

---

# 121. Profile resolution

```text
Application Configuration
       ↓
MappingProfileResolver
       ↓
MappingProfile
       ↓
Mapping Build
```

---

# 122. Profile selection timing

Idealmente se resuelve en:

```text
bootstrap/build
```

no en cada query.

---

# 123. Runtime profile switching

Deberá evitarse en persistent runtimes salvo que exista aislamiento explícito por metadata generation/registry.

---

# 124. Mapping Context

Se introduce:

```text
EntityMappingContext
```

---

# 125. Context may contain

```text
mapping profile
build mode
framework version
extension registry snapshot
configuration snapshot
target capability profile when explicitly required
```

---

# 126. Context must not contain

```text
current user
current HTTP request
current entity
live UnitOfWork
live transaction
mutable tenant state
```

---

# 127. Mapping discovery

Se introduce:

```text
EntityMappingDiscovery
```

---

# 128. Discovery responsibilities

Encontrar candidatos provenientes de:

```text
application
modules
packages
configured paths
generated sources
explicit providers
```

---

# 129. Discovery ≠ Reflection-only scanning

Podrá utilizar:

```text
Composer class maps
precomputed manifests
package manifests
explicit registration
reflection
```

según entorno.

---

# 130. Production discovery

Deberá favorecer:

```text
compiled mapping manifest
```

para evitar scans costosos.

---

# 131. Discovery manifest

Ejemplo:

```text
App\Entity\User
App\Entity\Post
App\Entity\Order
Acme\Billing\Entity\Invoice
```

---

# 132. Duplicate entity discovery

La misma entidad descubierta por múltiples fuentes no necesariamente es error.

Pero deberá converger a una única sesión de composición.

---

# 133. Duplicate identity

Dos clases reclamando el mismo:

```text
EntityTypeId
```

sin herencia/alias explícito deberán fallar.

---

# 134. Mapping Composer

```text
EntityMappingComposer
```

combina declaraciones compatibles.

---

# 135. Composition

```text
Convention declarations
        +
Attribute declarations
        +
Programmatic declarations
        +
Package declarations
        ↓
MappingComposer
        ↓
EntityMappingDraft
```

---

# 136. Composition semantics

Cada propiedad de mapping deberá definir una estrategia:

```text
SINGLE
MERGE
APPEND
OVERRIDE
EXCLUSIVE
```

---

# 137. SINGLE

Solo puede existir una declaración efectiva.

Ejemplo:

```text
entity identifier strategy
```

---

# 138. MERGE

Permite combinar subpropiedades.

Ejemplo:

```text
field mapping
├── column name
├── type
└── length
```

podría recibir contribuciones compatibles.

---

# 139. APPEND

Útil para:

```text
lifecycle callbacks
indexes projected from ORM hints
extension metadata
```

---

# 140. OVERRIDE

Permite reemplazo solo bajo política explícita.

---

# 141. EXCLUSIVE

Una fuente reclama control exclusivo de una sección.

Deberá utilizarse con mucha cautela.

---

# 142. Conflict Detector

Se introduce:

```text
EntityMappingConflictDetector
```

---

# 143. Conflict types

```text
VALUE_CONFLICT
TYPE_CONFLICT
SOURCE_CONFLICT
OWNERSHIP_CONFLICT
IDENTITY_CONFLICT
RELATIONSHIP_CONFLICT
INHERITANCE_CONFLICT
CAPABILITY_CONFLICT
EXTENSION_CONFLICT
```

---

# 144. Conflict report

Debe incluir:

```text
mapping path
candidate values
sources
precedence
override policy
provenance
resolution possibility
```

---

# 145. No accidental last-wins

Regla absoluta:

> El orden accidental de carga de archivos nunca deberá decidir el mapping final.

---

# 146. Mapping Resolver

Después de composición:

```text
EntityMappingResolver
```

resuelve referencias semánticas.

---

# 147. Resolution examples

```text
target class → EntityType
mappedBy string → RelationshipReference
type name → DatabaseTypeReference
generator name → IdentifierGeneratorId
connection name → LogicalConnectionName
```

---

# 148. Resolution ≠ runtime service resolution

Resolver:

```text
generator = "ulid"
```

a:

```text
IdentifierGeneratorId(ulid)
```

no significa instanciar el generador todavía.

---

# 149. Cross-entity mapping resolver

```text
CrossEntityMappingResolver
```

resolverá asociaciones entre mappings.

---

# 150. Two-pass resolution

Recomendado:

```text
Pass 1:
resolve entity identities

Pass 2:
resolve relationships/inheritance
```

---

# 151. Resolution graph

```text
User
├── posts → Post
└── organization → Organization

Post
└── author → User
```

forma un graph.

Los cycles son normales.

---

# 152. Mapping Normalizer

```text
EntityMappingNormalizer
```

convierte distintas representaciones válidas a una forma canónica.

---

# 153. Normalization examples

```text
"int"
"integer"
IntegerType::class
```

podrían normalizarse a:

```text
DatabaseTypeReference(INTEGER)
```

cuando sean semánticamente equivalentes.

---

# 154. Naming normalization

Podrá aplicar:

```text
UserProfile → user_profiles
createdAt   → created_at
```

solo si la naming strategy está habilitada.

---

# 155. NamingStrategy

Se introduce:

```text
EntityNamingStrategy
```

---

# 156. Naming strategy responsibilities

Puede resolver defaults para:

```text
entity → table
field → column
relationship → join column
join table names
```

---

# 157. NamingStrategy ≠ Schema Compiler

No deberá generar SQL ni aplicar quoting específico del dialecto.

---

# 158. Naming determinism

Mismo input + misma strategy deberá producir mismo nombre lógico.

---

# 159. Identifier length limits

Los límites físicos de una plataforma deberán manejarse posteriormente mediante compatibilidad/schema compilation, no mediante truncamiento silencioso en mapping genérico.

---

# 160. Mapping Validator

```text
EntityMappingValidator
```

validará la representación resuelta.

---

# 161. Validation stages

```text
Local Validation
      ↓
Cross-Entity Validation
      ↓
Semantic Validation
      ↓
Optional Capability Validation
```

---

# 162. Local validation

Verifica:

```text
entity structure
field declarations
identifier declarations
duplicate names
invalid combinations
```

---

# 163. Cross-entity validation

Verifica:

```text
relationship targets
mappedBy
inversedBy
inheritance
target identifiers
join references
```

---

# 164. Semantic validation

Ejemplos:

```text
generated + assigned contradiction
readonly + mutable mapping contradiction
non-null field without hydration path
multiple version fields
```

---

# 165. Capability validation

Cuando exista target platform conocido:

```text
Mapping
+
CapabilitySnapshot
→
MappingCompatibilityReport
```

---

# 166. Capability validation ≠ vendor branching

Debe preguntar:

```text
supports sequences?
supports native JSON?
supports generated columns?
```

no:

```text
is PostgreSQL?
```

salvo adaptadores especializados.

---

# 167. Validation result

Se propone:

```text
EntityMappingValidationReport
```

---

# 168. Issue severity

```text
ERROR
WARNING
INFO
```

---

# 169. Unknown

Cuando la validación depende de información no disponible deberá poder expresar:

```text
UNKNOWN
```

en vez de asumir compatibilidad.

---

# 170. Validation diagnostics

Ejemplo:

```text
DB-MAPPING-REL-007

Entity:
App\Entity\Post

Mapping:
relationships.author.mappedBy

Value:
posts

Target:
App\Entity\User

Problem:
User.posts does not target Post.

Source:
Post.php:61
```

---

# 171. Mapping Compiler

Después de validación podrá existir:

```text
EntityMappingCompiler
```

---

# 172. Compiler purpose

Convertir:

```text
canonical mapping
```

en una representación optimizada para construir:

```text
EntityMetadata
```

---

# 173. Mapping compiler ≠ SQL compiler

No deberá producir:

```text
SELECT
INSERT
UPDATE
DELETE
CREATE TABLE
ALTER TABLE
```

---

# 174. Mapping-to-Metadata compiler

Conceptualmente:

```text
EntityMapping
      ↓
EntityMappingCompiler
      ↓
Metadata Build Instructions
      ↓
EntityMetadataBuilder
```

La implementación podrá fusionar algunas fases, manteniendo las fronteras conceptuales.

---

# 175. Mapping Fingerprint

Cada mapping canónico deberá tener:

```text
EntityMappingFingerprint
```

---

# 176. Fingerprint inputs

Podrá incluir:

```text
entity identity
mapping declarations
resolved references
mapping profile
naming strategy version
relevant source versions
extension mappings
format version
```

---

# 177. Fingerprint exclusions

No incluir:

```text
build timestamp
object identity
random IDs
request ID
current user
current tenant
live connection
```

---

# 178. Determinism formula

```text
CanonicalSources(A)
=
CanonicalSources(B)
```

implica:

```text
MappingFingerprint(A)
=
MappingFingerprint(B)
```

---

# 179. Mapping fingerprint ≠ Metadata fingerprint

Ambos se relacionan, pero:

```text
MappingFingerprint
≠
EntityMetadataFingerprint
```

porque metadata puede incluir información derivada/compilada adicional.

---

# 180. Mapping cache

Podrá almacenarse:

```text
CompiledEntityMappingCache
```

---

# 181. Cache key

Conceptualmente:

```text
EntityType
+
MappingProfile
+
MappingFormatVersion
+
SourceFingerprint
+
ExtensionFingerprint
```

---

# 182. Cache content

Debe ser:

```text
immutable
versioned
portable enough for deployment strategy
free of request state
```

---

# 183. Cache invalidation

Cambios en:

```text
attributes
mapping configuration
conventions
naming strategy
mapping provider
extension
entity signature
mapping profile
```

deberán invalidar lo necesario.

---

# 184. Mapping dependency graph

Se introduce:

```text
EntityMappingDependencyGraph
```

---

# 185. Dependency examples

```text
Post.author
    ↓
User identifier

Admin
    ↓
User inheritance mapping

Order.total
    ↓
Money value-object mapping
```

---

# 186. Incremental compilation

El graph podrá permitir:

```text
change User
    ↓
recompile User
    +
affected dependents
```

sin recompilar todo el universo ORM.

---

# 187. Development mode

Pipeline:

```text
Discover
→ Collect
→ Compose
→ Resolve
→ Normalize
→ Validate
→ Compile
```

---

# 188. Production mode

Idealmente:

```text
Load Compiled Mapping
→ Verify Fingerprint
→ Build/Load Metadata
→ Freeze
```

---

# 189. Build command

Futuro CLI:

```text
volt database:mapping:compile
```

---

# 190. Validation command

```text
volt database:mapping:validate
```

---

# 191. Explain command

```text
volt database:mapping:explain App\\Entity\\User
```

---

# 192. Explain field

```text
volt database:mapping:explain App\\Entity\\User.email
```

podría mostrar:

```text
Property:
$email

Field:
email

Column:
email

Type:
email

Source:
PHP Attribute

Naming Strategy:
not used because explicit column name

Overrides:
none
```

---

# 193. Mapping diff

Tooling podrá comparar:

```text
Mapping Generation A
vs
Mapping Generation B
```

---

# 194. MappingDiff ≠ SchemaDiff

Puede detectar:

```text
field mapping changed
relationship changed
type changed
storage mapping changed
```

pero no necesariamente implica un cambio físico de schema.

---

# 195. Example

Cambiar:

```text
LAZY → EAGER
```

es un MappingDiff.

Probablemente no es un SchemaDiff.

---

# 196. Otro ejemplo

Cambiar:

```text
column: username
→
column: login
```

puede implicar SchemaDiff.

---

# 197. Mapping schema projection

Después de obtener metadata:

```text
EntityMetadata
      ↓
EntitySchemaProjector
      ↓
SchemaModel
```

No deberá hacerse directamente desde declaraciones incompletas.

---

# 198. Legacy database mapping

El sistema deberá permitir mapear schemas no convencionales.

Ejemplo:

```text
Entity:
Customer

Legacy table:
TB_CLI_001

Property:
name

Legacy column:
NM_CLI
```

---

# 199. Legacy identifiers

Debe soportar, cuando sea viable:

```text
composite IDs
non-standard PK names
assigned IDs
legacy sequences
```

---

# 200. Legacy relationship mapping

Ejemplo:

```text
ORD_CLI
→
TB_CLI_001.COD_CLI
```

sin obligar a renombrar físicamente el schema.

---

# 201. Portability

Mapping debe distinguir entre:

```text
portable semantic mapping
```

y:

```text
platform-specific mapping extension
```

---

# 202. Platform-specific mapping

Cuando sea imprescindible deberá estar encapsulado:

```text
PlatformMappingExtension
```

y declarar capabilities/restricciones.

---

# 203. No vendor leakage

No llenar entidades con lógica como:

```php
if (Database::driver() === 'mysql') {
}
```

---

# 204. Mapping extensions

Se introduce:

```text
EntityMappingExtension
```

---

# 205. Extension examples

Podrían añadir:

```text
encrypted fields
audit metadata
search metadata
temporal metadata
soft-delete mapping
custom relationship kinds
custom types
```

---

# 206. Extension contract

Conceptualmente:

```php
interface EntityMappingExtension
{
    public function id(): MappingExtensionId;

    public function contribute(
        EntityMappingDraft $mapping,
        EntityMappingExtensionContext $context,
    ): void;
}
```

---

# 207. Extension registry

```text
EntityMappingExtensionRegistry
```

deberá ser:

```text
deterministic
frozen after bootstrap
collision-aware
version-aware
```

---

# 208. Extension ordering

Basado en:

```text
dependencies
priority
stable extension ID
```

Nunca en orden accidental de filesystem.

---

# 209. Extension safety

Una extensión no podrá:

- ejecutar queries;
- ejecutar DDL;
- ejecutar migraciones;
- mutar Schema;
- abrir conexiones ocultamente;
- leer current user;
- leer request state;
- ocultar mapping conflicts;
- modificar frozen mappings.

---

# 210. Raw mapping escape hatch

Si se permite metadata arbitraria:

```text
RawMappingExtension
```

deberá quedar namespaced y aislada.

---

# 211. Raw mapping ≠ bypass

No deberá permitir violar invariantes fundamentales del ORM.

---

# 212. Model API integration

`114_DATABASE_MODEL_API_SYSTEM.md` utilizará este sistema para convertir convenciones como:

```php
protected string $table = 'users';
```

o equivalentes futuros en:

```text
EntityStorageMapping
```

---

# 213. Single ORM mapping

La clase `Model` no tendrá un segundo mapping engine.

```text
Model conventions
       ↓
Entity Mapping System
       ↓
Entity Metadata
       ↓
same ORM
```

---

# 214. Active Record mapping

```text
Active Record API
       ↓
Model Mapping Adapter
       ↓
Entity Mapping
```

---

# 215. Data Mapper mapping

```text
Plain Entity
       ↓
Attributes / Explicit Mapping
       ↓
Entity Mapping
```

Ambos convergen.

---

# 216. Unified architecture

```text
Model
   ─────────────┐
                │
Plain Entity    ├──→ Entity Mapping
                │          ↓
Package Entity ─┘     Entity Metadata
                           ↓
                      ORM Runtime
```

---

# 217. Persistent runtime requirements

Mapping canónico/compilado deberá ser compartible entre requests.

---

# 218. Shared worker state

Permitido:

```text
Frozen MappingSourceRegistry
Frozen MappingExtensionRegistry
Compiled EntityMappings
Compiled Mapping Manifest
```

---

# 219. Request-scoped state

No permitido dentro del mapping:

```text
current EntityManager
current UnitOfWork
current Connection
current Transaction
current TenantContext
current request
current authenticated user
```

---

# 220. FrankenPHP

Modelo:

```text
FrankenPHP Worker
│
├── Compiled Mapping Registry
│
├── Request A
│   └── ORM Scope A
│
├── Request B
│   └── ORM Scope B
│
└── Request C
    └── ORM Scope C
```

---

# 221. RoadRunner

Misma separación:

```text
worker-static mapping
+
request-scoped ORM state
```

---

# 222. OpenSwoole

Mappings compartidos deberán ser:

```text
immutable
coroutine-safe
request-neutral
```

---

# 223. Tenant mapping

La arquitectura preferida será:

```text
Entity Mapping
    ↓
logical storage
    ↓
Tenant Integration
    ↓
physical target
```

---

# 224. Example

Mapping:

```text
User → users
```

Tenant resolver:

```text
Tenant A
→ database_a.users

Tenant B
→ database_b.users
```

---

# 225. No tenant mutation

Nunca:

```text
$mapping->table =
    Tenant::current()->id . '_users';
```

sobre mapping global.

---

# 226. Tenant-specific variants

Si existen diferencias reales:

```text
MappingVariantKey
```

deberá gobernarlas explícitamente.

---

# 227. Variant identity

Podrá incluir:

```text
profile
schema generation
tenant class
feature set
```

pero deberá controlarse la cardinalidad.

---

# 228. Schema evolution compatibility

Mapping nuevo puede desplegarse mientras existe schema anterior.

Por ello:

```text
Mapping valid
```

no implica automáticamente:

```text
Mapping compatible with current deployment state
```

---

# 229. Deployment compatibility

Podrá analizarse:

```text
Entity Mapping
+
Current Schema
+
Compatibility Window
→
MappingDeploymentCompatibility
```

---

# 230. Expand/contract example

Release A:

```text
User.name → name
```

Expand:

```text
name
display_name
```

Transición:

```text
old app reads name
new app may read display_name
```

Contract posterior elimina `name`.

Mapping y migrations deberán coordinarse sin fusionar responsabilidades.

---

# 231. Mapping does not orchestrate deployment

El Mapping System describe.

El Zero-Downtime Migration System gobierna transición.

---

# 232. Mapping and safety

Un mapping puede ser perfectamente válido pero inducir un cambio peligroso.

Ejemplo:

```text
VARCHAR(255)
→
VARCHAR(20)
```

La seguridad de la migración pertenece a:

```text
111_DATABASE_MIGRATION_SAFETY_SYSTEM.md
```

---

# 233. Mapping security

El sistema deberá evitar mappings controlados por input de usuario runtime.

---

# 234. Static/build-time preference

Mapping deberá definirse preferentemente en:

```text
source code
trusted package manifests
trusted configuration
compiled artifacts
```

---

# 235. Untrusted mapping

No permitir que datos de request creen arbitrariamente:

```text
table names
column names
entity classes
generator classes
relationship targets
```

---

# 236. Secrets

Mapping no deberá contener:

```text
database passwords
credentials
encryption keys
API secrets
```

---

# 237. Security metadata references

Puede contener:

```text
EncryptionPolicyId
```

pero no la clave.

---

# 238. Mapping telemetry

Métricas posibles:

```text
mapping.discovery.duration
mapping.entities.discovered
mapping.sources.count
mapping.conflicts
mapping.validation.errors
mapping.compile.duration
mapping.cache.hit
mapping.cache.miss
```

---

# 239. Tracing

Build trace:

```text
Mapping Build
├── Discovery
├── Collection
├── Composition
├── Resolution
├── Normalization
├── Validation
└── Compilation
```

---

# 240. Provenance in telemetry

Evitar alta cardinalidad excesiva.

Los detalles completos deberán ir preferentemente a diagnostics/traces de desarrollo.

---

# 241. Error hierarchy

```text
DatabaseOrmException
└── EntityMappingException
    ├── EntityMappingDiscoveryException
    ├── EntityMappingSourceException
    ├── EntityMappingSourceCollisionException
    ├── EntityMappingDeclarationException
    ├── EntityMappingCompositionException
    ├── EntityMappingConflictException
    ├── EntityMappingResolutionException
    ├── EntityMappingNormalizationException
    ├── EntityMappingValidationException
    ├── EntityMappingCompilationException
    ├── EntityMappingFingerprintException
    ├── EntityMappingCacheException
    ├── EntityMappingProfileException
    ├── EntityMappingInheritanceException
    ├── EntityFieldMappingException
    ├── EntityIdentifierMappingException
    ├── EntityRelationshipMappingException
    ├── EntityEmbeddedMappingException
    ├── EntityLifecycleMappingException
    ├── EntityMappingExtensionException
    └── EntityMappingInvariantException
```

---

# 242. Namespace propuesto

```text
VoltStack\Quantum\Database\ORM\Mapping
```

---

# 243. Estructura propuesta

```text
src/Quantum/Database/ORM/Mapping/
│
├── Contract/
│   ├── EntityMappingSource.php
│   ├── EntityMappingProvider.php
│   └── EntityMappingExtension.php
│
├── Source/
│   ├── EntityMappingSourceId.php
│   ├── EntityMappingSourceRegistry.php
│   ├── MappingSourceType.php
│   └── MappingSourcePriority.php
│
├── Declaration/
│   ├── EntityMappingDeclaration.php
│   ├── EntityMappingDeclarationSet.php
│   ├── MappingPath.php
│   └── MappingProvenance.php
│
├── Entity/
│   ├── EntityMapping.php
│   ├── EntityMappingDraft.php
│   ├── EntityMappingBuilder.php
│   └── EntityStorageMapping.php
│
├── Identifier/
│   ├── EntityIdentifierMapping.php
│   └── IdentifierGenerationMapping.php
│
├── Field/
│   ├── EntityFieldMapping.php
│   ├── EntityFieldMappingCollection.php
│   └── ColumnMapping.php
│
├── Type/
│   └── TypeMapping.php
│
├── Relationship/
│   ├── EntityRelationshipMapping.php
│   ├── RelationshipMappingCollection.php
│   ├── JoinMapping.php
│   └── JoinTableMapping.php
│
├── Embedded/
│   ├── EmbeddedMapping.php
│   └── ValueObjectMapping.php
│
├── Inheritance/
│   ├── EntityInheritanceMapping.php
│   ├── DiscriminatorMapping.php
│   └── MappingOverride.php
│
├── Version/
│   └── EntityVersionMapping.php
│
├── Hydration/
│   └── EntityHydrationMapping.php
│
├── Persistence/
│   └── EntityPersistenceMapping.php
│
├── ChangeTracking/
│   └── EntityChangeTrackingMapping.php
│
├── Lifecycle/
│   └── EntityLifecycleMapping.php
│
├── Profile/
│   ├── EntityMappingProfile.php
│   └── EntityMappingProfileResolver.php
│
├── Discovery/
│   ├── EntityMappingDiscovery.php
│   └── EntityMappingManifest.php
│
├── Composition/
│   ├── EntityMappingComposer.php
│   └── MappingCompositionPolicy.php
│
├── Conflict/
│   ├── EntityMappingConflictDetector.php
│   └── EntityMappingConflict.php
│
├── Resolve/
│   ├── EntityMappingResolver.php
│   └── CrossEntityMappingResolver.php
│
├── Normalize/
│   ├── EntityMappingNormalizer.php
│   └── EntityNamingStrategy.php
│
├── Validation/
│   ├── EntityMappingValidator.php
│   ├── EntityMappingValidationReport.php
│   └── EntityMappingValidationIssue.php
│
├── Compile/
│   └── EntityMappingCompiler.php
│
├── Fingerprint/
│   └── EntityMappingFingerprint.php
│
├── Cache/
│   └── CompiledEntityMappingCache.php
│
├── Dependency/
│   └── EntityMappingDependencyGraph.php
│
├── Extension/
│   ├── EntityMappingExtensionRegistry.php
│   └── MappingExtensionId.php
│
└── Exception/
    └── ...
```

---

# 244. Dependencias permitidas

```text
Entity Mapping
    ↓
ORM contracts
Type references
Schema references
Capability contracts
Reflection abstraction
Support/value objects
Configuration snapshots
```

---

# 245. Dependencias prohibidas

El Mapping Core no deberá depender directamente de:

```text
PDO
QueryExecutor
EntityManager state
UnitOfWork state
IdentityMap
Transaction
HTTP Request
current authenticated User
current mutable TenantContext
```

---

# 246. Dependency direction

```text
Attribute Mapping
        ↓
Entity Mapping
        ↓
Entity Metadata
        ↓
ORM Runtime
```

---

# 247. No inverse dependency

No deberá existir:

```text
Entity Mapping
      ↓
EntityManager
```

ni:

```text
Entity Mapping
      ↓
Persistence Engine
```

---

# 248. Testing architecture

El sistema deberá disponer de tests para cada fase.

---

# 249. Source tests

Cubrir:

```text
attributes
conventions
programmatic
packages
extensions
compiled mappings
```

---

# 250. Composition tests

Cubrir:

```text
compatible merge
explicit override
illegal override
duplicate declaration
source collision
deterministic precedence
```

---

# 251. Entity mapping tests

Cubrir:

```text
entity
model
embeddable
mapped superclass
value object
```

---

# 252. Field tests

Cubrir:

```text
property naming
column naming
types
length
precision
scale
nullability
defaults
generated fields
insertability
updatability
```

---

# 253. Identifier tests

Cubrir:

```text
single
composite
assigned
auto increment
sequence
UUID
ULID
custom
```

---

# 254. Relationship tests

Cubrir:

```text
one-to-one
one-to-many
many-to-one
many-to-many
unidirectional
bidirectional
mappedBy
inversedBy
custom join
join table
cascade
orphan removal
```

---

# 255. Embedded tests

Cubrir:

```text
single-level
nested
prefix
custom column override
cycle detection
```

---

# 256. Inheritance tests

Cubrir:

```text
single table
joined
table per class
discriminator
mapped superclass
field override
relationship override
```

---

# 257. Resolution tests

Cubrir:

```text
unknown target
unknown field
unknown relationship
unknown type
unknown generator
cyclic relationship graph
```

---

# 258. Normalization tests

Múltiples representaciones equivalentes deberán converger al mismo mapping canónico.

---

# 259. Determinism tests

Cambiar el orden de discovery no deberá cambiar el resultado semántico.

---

# 260. Fingerprint tests

Mismo mapping canónico:

```text
→ same fingerprint
```

Cambio semántico:

```text
→ different fingerprint
```

---

# 261. Cache tests

Cubrir:

```text
valid
stale
corrupt
wrong format
changed source
changed extension
changed profile
changed naming strategy
```

---

# 262. Persistent runtime tests

Verificar que mappings compartidos no absorban estado de requests sucesivos.

---

# 263. Concurrency tests

Múltiples coroutines deberán poder consultar mappings frozen concurrentemente.

---

# 264. Security tests

Verificar:

```text
no runtime user-controlled table injection
no secrets
no hidden DB I/O
no extension bypass
```

---

# 265. Invariantes arquitectónicas

## DB-ENTITY-MAPPING-001
Mapping será distinto de Metadata.

## DB-ENTITY-MAPPING-002
Mapping será distinto de SchemaModel.

## DB-ENTITY-MAPPING-003
Mapping será distinto de Migration.

## DB-ENTITY-MAPPING-004
Mapping no ejecutará SQL.

## DB-ENTITY-MAPPING-005
Mapping no generará SQL.

## DB-ENTITY-MAPPING-006
Mapping no abrirá conexiones.

## DB-ENTITY-MAPPING-007
Mapping no administrará EntityManager.

## DB-ENTITY-MAPPING-008
Mapping no administrará UnitOfWork.

## DB-ENTITY-MAPPING-009
Mapping no mantendrá IdentityMap.

## DB-ENTITY-MAPPING-010
Mapping no ejecutará lifecycle callbacks.

## DB-ENTITY-MAPPING-011
Mapping sources tendrán identidad estable.

## DB-ENTITY-MAPPING-012
Source identity será distinta de source priority.

## DB-ENTITY-MAPPING-013
Source precedence será explícita.

## DB-ENTITY-MAPPING-014
Orden accidental de discovery no decidirá mapping.

## DB-ENTITY-MAPPING-015
Conflictos no utilizarán last-wins silencioso.

## DB-ENTITY-MAPPING-016
Overrides deberán estar permitidos explícitamente.

## DB-ENTITY-MAPPING-017
Mapping declarations preservarán provenance.

## DB-ENTITY-MAPPING-018
MappingPath identificará granularmente declaraciones.

## DB-ENTITY-MAPPING-019
Una fuente podrá contribuir parcialmente.

## DB-ENTITY-MAPPING-020
EntityMappingDraft será distinto de EntityMapping frozen.

## DB-ENTITY-MAPPING-021
Canonical mapping será immutable.

## DB-ENTITY-MAPPING-022
Entity mapping tendrá identidad no ambigua.

## DB-ENTITY-MAPPING-023
Entity mapping será distinto de table mapping.

## DB-ENTITY-MAPPING-024
Logical connection mapping no abrirá una Connection.

## DB-ENTITY-MAPPING-025
Identifier mapping será explícito.

## DB-ENTITY-MAPPING-026
Composite identifiers serán explícitos.

## DB-ENTITY-MAPPING-027
Generated identifier strategy será declarativa.

## DB-ENTITY-MAPPING-028
Custom generators se referenciarán por identidad estable.

## DB-ENTITY-MAPPING-029
Field será distinto de property.

## DB-ENTITY-MAPPING-030
Field será distinto de column.

## DB-ENTITY-MAPPING-031
ColumnMapping será distinto de Schema ColumnDefinition.

## DB-ENTITY-MAPPING-032
PHP type será distinto de ORM type.

## DB-ENTITY-MAPPING-033
ORM type será distinto de physical platform type.

## DB-ENTITY-MAPPING-034
Type inference no seleccionará vendor SQL prematuramente.

## DB-ENTITY-MAPPING-035
Nullability conflicts serán diagnosticados.

## DB-ENTITY-MAPPING-036
PHP default será distinto de database default.

## DB-ENTITY-MAPPING-037
Generated field semantics serán explícitas.

## DB-ENTITY-MAPPING-038
Insertability será representable.

## DB-ENTITY-MAPPING-039
Updatability será representable.

## DB-ENTITY-MAPPING-040
Relationship mapping será distinto de scalar field mapping.

## DB-ENTITY-MAPPING-041
Relationship targets deberán resolverse.

## DB-ENTITY-MAPPING-042
mappedBy deberá resolverse a relación válida.

## DB-ENTITY-MAPPING-043
inversedBy deberá resolverse a relación válida.

## DB-ENTITY-MAPPING-044
Owning side será explícito.

## DB-ENTITY-MAPPING-045
ORM cascade será distinto de database cascade.

## DB-ENTITY-MAPPING-046
Orphan removal será explícito.

## DB-ENTITY-MAPPING-047
Loading policy será distinta de runtime loaded state.

## DB-ENTITY-MAPPING-048
JoinMapping será distinto de Schema ForeignKeyDefinition.

## DB-ENTITY-MAPPING-049
Many-to-many join mapping será explícito.

## DB-ENTITY-MAPPING-050
Embeddable será distinto de Entity.

## DB-ENTITY-MAPPING-051
Value object será distinto de Relationship.

## DB-ENTITY-MAPPING-052
Embedded cycles inválidos serán detectados.

## DB-ENTITY-MAPPING-053
Inheritance mapping será distinto de PHP inheritance.

## DB-ENTITY-MAPPING-054
Discriminator mappings serán validados.

## DB-ENTITY-MAPPING-055
Parent mappings no serán mutados por child overrides.

## DB-ENTITY-MAPPING-056
Version field será único cuando la estrategia lo requiera.

## DB-ENTITY-MAPPING-057
Change-tracking mapping no realizará change tracking.

## DB-ENTITY-MAPPING-058
Hydration mapping no realizará hydration.

## DB-ENTITY-MAPPING-059
Persistence mapping no realizará persistence.

## DB-ENTITY-MAPPING-060
Lifecycle mapping no ejecutará callbacks.

## DB-ENTITY-MAPPING-061
Model conventions producirán mappings ORM normales.

## DB-ENTITY-MAPPING-062
Active Record no tendrá un mapping engine separado.

## DB-ENTITY-MAPPING-063
Plain Entity y Model convergerán a EntityMetadata.

## DB-ENTITY-MAPPING-064
Mapping profiles tendrán identidad estable.

## DB-ENTITY-MAPPING-065
Mapping profile será distinto de tenant.

## DB-ENTITY-MAPPING-066
Runtime profile switching no mutará mappings globales.

## DB-ENTITY-MAPPING-067
Mapping context no contendrá current user.

## DB-ENTITY-MAPPING-068
Mapping context no contendrá UnitOfWork vivo.

## DB-ENTITY-MAPPING-069
Mapping context no contendrá Transaction viva.

## DB-ENTITY-MAPPING-070
Discovery será distinto de mapping resolution.

## DB-ENTITY-MAPPING-071
Discovery podrá usar manifests compilados.

## DB-ENTITY-MAPPING-072
Duplicate discovery no implicará automáticamente duplicate mapping.

## DB-ENTITY-MAPPING-073
Duplicate EntityTypeId incompatible será error.

## DB-ENTITY-MAPPING-074
Composition será determinista.

## DB-ENTITY-MAPPING-075
Composition semantics serán explícitas por mapping path.

## DB-ENTITY-MAPPING-076
Conflict detection ocurrirá antes del freeze.

## DB-ENTITY-MAPPING-077
Conflict reports preservarán las fuentes contendientes.

## DB-ENTITY-MAPPING-078
Resolver no instanciará runtime services innecesariamente.

## DB-ENTITY-MAPPING-079
Cross-entity resolution soportará ciclos válidos.

## DB-ENTITY-MAPPING-080
Normalization no inventará certeza.

## DB-ENTITY-MAPPING-081
NamingStrategy será determinista.

## DB-ENTITY-MAPPING-082
NamingStrategy no generará SQL.

## DB-ENTITY-MAPPING-083
Mapping genérico no truncará silenciosamente nombres por límites vendor.

## DB-ENTITY-MAPPING-084
Validation ocurrirá antes del runtime caliente cuando sea posible.

## DB-ENTITY-MAPPING-085
UNKNOWN no se convertirá automáticamente en compatible.

## DB-ENTITY-MAPPING-086
Capability validation utilizará capabilities.

## DB-ENTITY-MAPPING-087
Version string no sustituirá capability semantics.

## DB-ENTITY-MAPPING-088
Mapping compiler será distinto de SQL compiler.

## DB-ENTITY-MAPPING-089
Mapping compiler no producirá DML.

## DB-ENTITY-MAPPING-090
Mapping compiler no producirá DDL.

## DB-ENTITY-MAPPING-091
Mapping fingerprint será determinista.

## DB-ENTITY-MAPPING-092
Mapping fingerprint será distinto de metadata fingerprint.

## DB-ENTITY-MAPPING-093
Fingerprint no dependerá del build timestamp.

## DB-ENTITY-MAPPING-094
Fingerprint no dependerá del request actual.

## DB-ENTITY-MAPPING-095
Mapping cache será versionado.

## DB-ENTITY-MAPPING-096
Stale mapping cache será detectable.

## DB-ENTITY-MAPPING-097
Mapping dependency graph será explícito.

## DB-ENTITY-MAPPING-098
Incremental recompilation respetará dependencias.

## DB-ENTITY-MAPPING-099
MappingDiff será distinto de SchemaDiff.

## DB-ENTITY-MAPPING-100
Mapping change no implicará automáticamente schema change.

## DB-ENTITY-MAPPING-101
Schema change no será ejecutado por Mapping System.

## DB-ENTITY-MAPPING-102
Legacy schemas podrán mapearse explícitamente.

## DB-ENTITY-MAPPING-103
Legacy naming no deberá contaminar nombres del dominio.

## DB-ENTITY-MAPPING-104
Platform-specific mapping estará aislado.

## DB-ENTITY-MAPPING-105
Platform-specific mapping declarará requirements.

## DB-ENTITY-MAPPING-106
Vendor branching no contaminará entities.

## DB-ENTITY-MAPPING-107
Extensions tendrán IDs namespaced.

## DB-ENTITY-MAPPING-108
Extension registry será frozen en runtime.

## DB-ENTITY-MAPPING-109
Extension collisions serán error.

## DB-ENTITY-MAPPING-110
Extensions no ejecutarán SQL durante mapping.

## DB-ENTITY-MAPPING-111
Extensions no ejecutarán migrations.

## DB-ENTITY-MAPPING-112
Extensions no abrirán conexiones ocultamente.

## DB-ENTITY-MAPPING-113
Extensions no ocultarán conflictos.

## DB-ENTITY-MAPPING-114
Raw mapping extensions no bypassarán invariantes.

## DB-ENTITY-MAPPING-115
Model API utilizará el mismo Entity Mapping System.

## DB-ENTITY-MAPPING-116
Active Record utilizará el mismo Entity Mapping System.

## DB-ENTITY-MAPPING-117
Repository/Data Mapper utilizarán la misma metadata resultante.

## DB-ENTITY-MAPPING-118
Compiled mappings podrán compartirse entre requests.

## DB-ENTITY-MAPPING-119
Shared mappings serán immutable.

## DB-ENTITY-MAPPING-120
Mappings no almacenarán EntityManager request-scoped.

## DB-ENTITY-MAPPING-121
Mappings no almacenarán UnitOfWork request-scoped.

## DB-ENTITY-MAPPING-122
Mappings no almacenarán live Connection.

## DB-ENTITY-MAPPING-123
Mappings no almacenarán live Transaction.

## DB-ENTITY-MAPPING-124
Mappings no almacenarán current TenantContext mutable.

## DB-ENTITY-MAPPING-125
FrankenPHP podrá compartir compiled mappings.

## DB-ENTITY-MAPPING-126
RoadRunner podrá compartir compiled mappings.

## DB-ENTITY-MAPPING-127
OpenSwoole podrá compartir immutable mappings.

## DB-ENTITY-MAPPING-128
Tenant resolution ocurrirá fuera del mapping global cuando sea posible.

## DB-ENTITY-MAPPING-129
Tenant-specific mapping no mutará mapping base.

## DB-ENTITY-MAPPING-130
Mapping variants tendrán identidad explícita.

## DB-ENTITY-MAPPING-131
Variant cardinality será gobernable.

## DB-ENTITY-MAPPING-132
Valid mapping no implicará deployment compatibility.

## DB-ENTITY-MAPPING-133
Valid mapping no implicará migration safety.

## DB-ENTITY-MAPPING-134
Zero-downtime orchestration no pertenecerá al Mapping System.

## DB-ENTITY-MAPPING-135
Mapping no aceptará arbitrariamente identifiers físicos desde input no confiable.

## DB-ENTITY-MAPPING-136
Mapping no contendrá database credentials.

## DB-ENTITY-MAPPING-137
Mapping no contendrá encryption keys.

## DB-ENTITY-MAPPING-138
Security policy podrá referenciarse por ID sin contener secrets.

## DB-ENTITY-MAPPING-139
Mapping telemetry no alterará mapping semantics.

## DB-ENTITY-MAPPING-140
Mapping provenance será auditable.

## DB-ENTITY-MAPPING-141
Mismas entradas canónicas producirán mismo mapping canónico.

## DB-ENTITY-MAPPING-142
Mismo mapping canónico producirá mismo fingerprint bajo la misma versión.

## DB-ENTITY-MAPPING-143
Entity Metadata será construida desde mapping resuelto y validado.

## DB-ENTITY-MAPPING-144
EntityManager no interpretará atributos directamente.

## DB-ENTITY-MAPPING-145
UnitOfWork no interpretará atributos directamente.

## DB-ENTITY-MAPPING-146
Hydrator no interpretará atributos directamente.

## DB-ENTITY-MAPPING-147
Persistence Engine no interpretará atributos directamente.

## DB-ENTITY-MAPPING-148
Query Engine no interpretará atributos directamente.

## DB-ENTITY-MAPPING-149
Mapping será la frontera declarativa entre dominio y metadata ORM.

## DB-ENTITY-MAPPING-150
Toda fuente de mapping soportada deberá converger al mismo modelo canónico antes de participar en el runtime ORM.

---

# 266. Anti-patterns

## 266.1 Atributos interpretados por EntityManager

Incorrecto:

```text
EntityManager
    ↓
Reflection
    ↓
#[Column]
```

Correcto:

```text
Attributes
    ↓
Entity Mapping
    ↓
Entity Metadata
    ↓
EntityManager
```

---

## 266.2 Mapping dependiente del request

Incorrecto:

```php
#[Table(Tenant::current()->table())]
```

---

## 266.3 SQL dentro de mapping

Incorrecto:

```php
$mapping->field('email')
    ->rawSqlType('VARCHAR(255) COLLATE ...');
```

como mecanismo principal portable.

---

## 266.4 Last-wins

Incorrecto:

```text
package mapping
application mapping
attribute mapping

whichever loads last wins
```

---

## 266.5 Inferencia vendor prematura

Incorrecto:

```text
PHP int
→ MYSQL INT
```

durante mapping genérico.

Preferir:

```text
PHP int
→ ORM INTEGER
→ platform mapping later
```

---

## 266.6 Segundo mapping para Model

Incorrecto:

```text
Entity Mapping Engine
+
Active Record Mapping Engine
```

Correcto:

```text
multiple declaration APIs
→ one mapping model
```

---

## 266.7 Mapping como migration

Incorrecto:

```text
change #[Column]
→ automatically ALTER TABLE production
```

---

# 267. Flujo completo

```text
                 APPLICATION DOMAIN
                        │
                        ▼
               Mapping Candidates
                        │
       ┌────────────────┼─────────────────┐
       │                │                 │
       ▼                ▼                 ▼
 Attributes        Conventions       Programmatic
       │                │                 │
       └────────────────┼─────────────────┘
                        ▼
               Source Registry
                        ▼
                Declaration Set
                        ▼
                  Composition
                        ▼
              Conflict Detection
                        ▼
                  Resolution
                        ▼
                Normalization
                        ▼
                  Validation
                        ▼
              Canonical Mapping
                        ▼
                  Compilation
                        ▼
                Entity Metadata
                        ▼
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
    EntityManager   EntityQuery     Hydration
          │
          ▼
      UnitOfWork
          │
          ▼
 Persistence Engine
```

---

# 268. Fórmula de mapping

```text
EntityMapping
=
EntityDeclaration
+
StorageMapping
+
IdentifierMapping
+
FieldMappings
+
RelationshipMappings
+
EmbeddedMappings
+
InheritanceMapping
+
VersionMapping
+
HydrationMapping
+
PersistenceMapping
+
ChangeTrackingMapping
+
LifecycleMapping
+
ExtensionMappings
```

---

# 269. Fórmula de resolución

```text
CanonicalMapping
=
Validate(
    Normalize(
        Resolve(
            Compose(
                Collect(
                    Discover(
                        MappingSources
                    )
                )
            )
        )
    )
)
```

---

# 270. Fórmula de composición

Para una propiedad `p`:

```text
EffectiveMapping(p)
=
CompositionPolicy(
    Declarations(p),
    SourcePrecedence,
    OverrideRules,
    ConflictRules
)
```

---

# 271. Fórmula de determinismo

```text
SameCanonicalSources
∧
SameProfile
∧
SameNamingStrategy
∧
SameExtensionSet
∧
SameMappingFormatVersion
```

deberá implicar:

```text
SameCanonicalMapping
∧
SameMappingFingerprint
```

---

# 272. Fórmula de separación

```text
Mapping
=
Declarative Persistence Intent
```

mientras:

```text
Metadata
=
Canonical ORM Description
```

y:

```text
Persistence
=
Runtime Database Effects
```

Por tanto:

```text
Mapping
≠
Metadata
≠
Persistence
```

---

# 273. Fórmula Active Record/Data Mapper

```text
ActiveRecordDeclarations
        │
        ├───────────┐
        ▼           │
                 EntityMapping
        ▲           │
        ├───────────┘
PlainEntityDeclarations
        │
        ▼
EntityMetadata
        ▼
Shared ORM Engine
```

---

# 274. Regla maestra

> **VoltStack permitirá múltiples formas de declarar cómo una entidad se relaciona con la persistencia, pero ninguna de esas formas será el ORM por sí misma. Todas deberán converger, de manera determinista, validada y conflict-aware, al mismo Entity Mapping y posteriormente a la misma Entity Metadata canónica.**

---

# 275. Resultado arquitectónico

Con este sistema se obtiene:

```text
Developer API flexibility
          +
Single semantic mapping model
          +
Single metadata model
          +
Single ORM runtime
```

evitando:

```text
Attribute ORM
Convention ORM
Model ORM
Programmatic ORM
Package ORM
```

como implementaciones paralelas.

En su lugar:

```text
Attributes ───────┐
Conventions ──────┤
Programmatic ─────┤
Packages ─────────┼──→ Entity Mapping
Extensions ───────┘          │
                             ▼
                       Entity Metadata
                             │
                             ▼
                         ORM Engine
```

---

# 276. Relación con documentos 112–115

```text
112_DATABASE_ORM_ARCHITECTURE
        ↓
define la arquitectura ORM

113_DATABASE_ENTITY_MODEL
        ↓
define qué es una Entity

114_DATABASE_MODEL_API_SYSTEM
        ↓
define la API ergonómica tipo Model

115_DATABASE_ENTITY_METADATA_SYSTEM
        ↓
define la representación canónica

116_DATABASE_ENTITY_MAPPING_SYSTEM
        ↓
define cómo se construye esa representación
```

---

# 277. Próxima especialización

Este documento establece el Mapping System general.

La primera fuente oficial especializada será:

```text
PHP Attributes
```

formalizada en:

```text
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
```

---

# 278. Relación 116 → 117

```text
116 Entity Mapping System
        │
        │ defines contracts
        ▼
117 Attribute Mapping System
        │
        │ implements one MappingSource
        ▼
AttributeMappingSource
        │
        ▼
MappingDeclarations
        │
        ▼
116 Entity Mapping Pipeline
        │
        ▼
115 Entity Metadata System
```

---

# 279. Restricción fundamental para 117

El Attribute Mapping System no deberá convertirse en un segundo Mapping Engine.

Deberá limitarse a:

```text
PHP Reflection
      ↓
Attribute Discovery
      ↓
Attribute Validation
      ↓
Mapping Declarations
```

y delegar:

```text
composition
cross-source conflict handling
normalization
cross-entity resolution
canonical mapping
metadata compilation
```

al sistema definido aquí.

---

# 280. Master Formula

```text
Database Entity Mapping System
=
Mapping Source Architecture
+
Typed Mapping Declarations
+
Mapping Provenance
+
Explicit Source Precedence
+
Deterministic Composition
+
Conflict Detection
+
Entity-to-Storage Mapping
+
Identifier Mapping
+
Field-to-Column Mapping
+
Type Mapping
+
Relationship Mapping
+
Embedded and Value Object Mapping
+
Inheritance Mapping
+
Version Mapping
+
Hydration Mapping
+
Persistence Mapping
+
Change Tracking Mapping
+
Lifecycle Mapping
+
Mapping Profiles
+
Cross-Entity Resolution
+
Canonical Normalization
+
Semantic Validation
+
Capability Awareness
+
Mapping Compilation
+
Deterministic Fingerprinting
+
Dependency-Aware Caching
+
Extension Governance
+
Model API Convergence
+
Persistent Runtime Isolation
```

---

# 281. Master Rule

> **Mapping in VoltStack is a deterministic declarative translation layer between the PHP domain model and the ORM metadata model. It may discover, compose, normalize, resolve and validate persistence intent, but it never becomes runtime entity state, never performs persistence and never generates or executes SQL.**

---

# 282. Siguiente documento

```text
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
```

Este documento deberá especializar la primera fuente de mapping oficial de VoltStack:

```text
PHP 8+ Attributes
      ↓
Attribute Discovery
      ↓
Attribute Reader
      ↓
Attribute Semantic Validation
      ↓
Typed Mapping Declarations
      ↓
Entity Mapping System
```

incluyendo, entre otros:

```text
#[Entity]
#[Table]
#[Id]
#[Column]
#[GeneratedValue]

#[OneToOne]
#[OneToMany]
#[ManyToOne]
#[ManyToMany]

#[JoinColumn]
#[JoinTable]

#[Embedded]
#[Embeddable]

#[Inheritance]
#[DiscriminatorColumn]
#[DiscriminatorMap]

#[Version]

#[PrePersist]
#[PostPersist]
#[PreUpdate]
#[PostUpdate]
#[PreRemove]
#[PostRemove]
#[PostLoad]

#[ChangeTracking]

custom mapping attributes
attribute aliases
repeatable attributes
attribute validation
reflection strategy
compiled attribute manifests
attribute caching
source provenance
extension attributes
persistent-runtime optimization
```

manteniendo la regla:

> **PHP Attributes are a mapping syntax, not the ORM, not Entity Metadata and not runtime persistence behavior.**