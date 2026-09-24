# 117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md

# VoltStack Quantum Database
## Database Attribute Mapping System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 117 — Database Attribute Mapping System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Attribute Mapping System` define la fuente oficial de mapping basada en **PHP Attributes** para el ORM de VoltStack.

Su función es transformar declaraciones como:

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

    #[ManyToOne(target: Organization::class)]
    #[JoinColumn(name: 'organization_id')]
    private Organization $organization;
}
```

en declaraciones tipadas consumibles por:

```text
116_DATABASE_ENTITY_MAPPING_SYSTEM
```

El pipeline será:

```text
PHP Source
    ↓
Reflection / Compiled Manifest
    ↓
Attribute Discovery
    ↓
Attribute Reading
    ↓
Attribute Syntax Validation
    ↓
Attribute Semantic Translation
    ↓
MappingDeclarationSet
    ↓
Entity Mapping System
    ↓
EntityMetadata
```

Principio central:

> **PHP Attributes son una sintaxis declarativa para expresar mapping ORM; no son Entity Metadata, no son Schema y no ejecutan comportamiento de persistencia.**

---

# 2. Attribute Mapping ≠ ORM

Debe mantenerse:

```text
PHP Attribute
≠
Entity Mapping
≠
Entity Metadata
≠
ORM Runtime
```

Por ejemplo:

```php
#[Column(length: 255)]
private string $email;
```

es solamente una declaración.

No:

- crea una columna;
- consulta una columna;
- valida el schema físico;
- hidrata `$email`;
- detecta cambios;
- genera SQL;
- ejecuta SQL.

---

# 3. Attribute Mapping ≠ Reflection

Reflection es uno de los mecanismos posibles para descubrir attributes.

Por tanto:

```text
Attribute Mapping System
≠
PHP Reflection
```

En desarrollo podrá existir:

```text
PHP Reflection
      ↓
Attribute Reader
```

En producción:

```text
Compiled Attribute Manifest
      ↓
Attribute Reader
```

Ambos deberán producir la misma semántica.

---

# 4. Objetivos

El sistema deberá proporcionar:

- catálogo oficial de attributes ORM;
- discovery tipado;
- lectura segura;
- validación estructural;
- validación semántica local;
- traducción a `EntityMappingDeclaration`;
- provenance;
- aliases controlados;
- repeatable attributes;
- custom attributes;
- extension attributes;
- compiled manifests;
- caching;
- fingerprints;
- diagnostics;
- soporte para persistent runtimes;
- compatibilidad con entidades POPO;
- integración con Model API;
- coste mínimo en producción.

---

# 5. No objetivos

Este sistema no deberá:

- construir directamente EntityMetadata;
- resolver completamente relaciones entre entidades;
- ejecutar queries;
- ejecutar DDL;
- abrir conexiones;
- modificar schema;
- ejecutar migrations;
- administrar EntityManager;
- administrar UnitOfWork;
- hidratar entidades;
- persistir entidades;
- implementar lazy loading;
- resolver tenants;
- ejecutar lifecycle callbacks.

---

# 6. Posición arquitectónica

```text
PHP Classes
    │
    ├── Class Attributes
    ├── Property Attributes
    ├── Method Attributes
    └── Parameter Attributes
             │
             ▼
      Attribute Discovery
             │
             ▼
       Attribute Reader
             │
             ▼
      Attribute Validator
             │
             ▼
      Attribute Translator
             │
             ▼
 MappingDeclarationSet
             │
             ▼
116 Entity Mapping System
             │
             ▼
115 Entity Metadata System
             │
             ▼
         ORM Runtime
```

---

# 7. Arquitectura principal

```text
                    PHP SOURCE
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
        Reflection Path     Compiled Manifest
              │                   │
              └─────────┬─────────┘
                        ▼
                AttributeReader
                        │
                        ▼
              AttributeDescriptor
                        │
                        ▼
             AttributeValidator
                        │
                        ▼
             AttributeTranslator
                        │
                        ▼
           MappingDeclarationSet
                        │
                        ▼
             EntityMappingSystem
```

---

# 8. Capas

El sistema se dividirá conceptualmente en:

```text
Discovery
Reading
Descriptor Model
Validation
Translation
Registry
Compilation
Cache
Extension
Diagnostics
```

---

# 9. Contratos principales

Se proponen:

```text
AttributeMappingSource
AttributeReader
AttributeDescriptor
AttributeTarget
AttributeRegistry
AttributeDefinition
AttributeValidator
AttributeTranslator
AttributeTranslationContext
AttributeValidationContext

CompiledAttributeManifest
AttributeManifestCompiler
AttributeManifestLoader

AttributeAliasRegistry
AttributeExtensionRegistry

AttributeMappingFingerprint
```

---

# 10. AttributeMappingSource

Implementará el contrato definido en el documento 116:

```php
final class AttributeMappingSource implements EntityMappingSource
{
    public function id(): EntityMappingSourceId
    {
        return new EntityMappingSourceId(
            'voltstack.orm.attributes'
        );
    }

    public function declarations(
        EntityMappingCandidate $candidate,
        EntityMappingContext $context,
    ): iterable {
        // Attribute pipeline.
    }
}
```

---

# 11. Identidad de la fuente

La fuente oficial será conceptualmente:

```text
voltstack.orm.attributes
```

Su identidad deberá ser:

- estable;
- namespaced;
- versionable;
- independiente del orden de bootstrap.

---

# 12. Attribute targets

VoltStack deberá distinguir:

```text
CLASS
PROPERTY
METHOD
PARAMETER
```

y, si se requieren posteriormente:

```text
CLASS_CONSTANT
```

---

# 13. AttributeTarget

```php
enum AttributeTarget
{
    case CLASS_;
    case PROPERTY;
    case METHOD;
    case PARAMETER;
}
```

El modelo interno podrá utilizar nombres diferentes para evitar conflictos sintácticos.

---

# 14. Attribute Descriptor

La representación descubierta no deberá depender permanentemente de `ReflectionAttribute`.

Se propone:

```php
final readonly class AttributeDescriptor
{
    public function __construct(
        public AttributeName $name,
        public AttributeTarget $target,
        public AttributeArguments $arguments,
        public AttributeLocation $location,
        public AttributeProvenance $provenance,
    ) {}
}
```

---

# 15. Descriptor ≠ Attribute Instance

Distinción importante:

```text
AttributeDescriptor
≠
instancia PHP del Attribute
```

VoltStack deberá poder analizar atributos sin necesariamente ejecutar constructores arbitrarios.

---

# 16. Razón

Usar directamente:

```php
$reflectionAttribute->newInstance();
```

puede introducir:

- ejecución de código;
- errores tardíos;
- dependencias innecesarias;
- mayor coste;
- dificultad para compilación estática.

La arquitectura deberá permitir una representación intermedia segura.

---

# 17. Attribute Reader

Contrato conceptual:

```php
interface AttributeReader
{
    public function readClass(
        ClassReference $class,
    ): AttributeDescriptorCollection;

    public function readProperty(
        PropertyReference $property,
    ): AttributeDescriptorCollection;

    public function readMethod(
        MethodReference $method,
    ): AttributeDescriptorCollection;
}
```

---

# 18. Implementaciones

Se prevén:

```text
ReflectionAttributeReader
CompiledAttributeReader
CompositeAttributeReader
```

---

# 19. ReflectionAttributeReader

Será útil principalmente para:

```text
development
testing
dynamic discovery
```

---

# 20. CompiledAttributeReader

Será preferido en:

```text
production
persistent workers
preloaded environments
```

---

# 21. Equivalencia semántica

Regla:

```text
ReflectionAttributeReader(Source)
```

y:

```text
CompiledAttributeReader(Compile(Source))
```

deberán producir declaraciones semánticamente equivalentes.

---

# 22. Catálogo base

VoltStack podrá proporcionar inicialmente:

```text
Entity
Table

Id
Column
GeneratedValue

OneToOne
OneToMany
ManyToOne
ManyToMany

JoinColumn
JoinColumns
JoinTable

Embeddable
Embedded

MappedSuperclass

Inheritance
DiscriminatorColumn
DiscriminatorMap

Version

ChangeTracking

PrePersist
PostPersist
PreUpdate
PostUpdate
PreRemove
PostRemove
PostLoad
```

Posteriormente podrán añadirse attributes especializados.

---

# 23. Organización del catálogo

Se recomienda:

```text
Attribute/
├── Entity/
├── Field/
├── Identifier/
├── Relationship/
├── Embedded/
├── Inheritance/
├── Version/
├── Lifecycle/
└── Extension/
```

---

# 24. `#[Entity]`

Declara una clase como entidad ORM.

Ejemplo:

```php
#[Entity]
final class User
{
}
```

Semánticamente produce:

```text
entity.kind = ENTITY
entity.class = App\Entity\User
```

---

# 25. Entity attribute

Posible diseño:

```php
#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Entity
{
    public function __construct(
        public ?string $name = null,
        public ?string $repository = null,
    ) {}
}
```

Los argumentos finales deberán mantenerse mínimos.

---

# 26. Entity name

Si se soporta un nombre lógico:

```php
#[Entity(name: 'user')]
```

deberá distinguirse de:

```text
PHP class name
table name
EntityTypeId
```

---

# 27. Repository reference

Si:

```php
#[Entity(repository: UserRepository::class)]
```

se permite, será una referencia declarativa.

No instancia el Repository.

---

# 28. `#[Table]`

Ejemplo:

```php
#[Entity]
#[Table('users')]
final class User
{
}
```

Produce:

```text
entity.storage.table = users
```

---

# 29. Table options

Podría soportar:

```php
#[Table(
    name: 'users',
    schema: 'public',
)]
```

Los argumentos vendor-specific deberán evitarse en el attribute base.

---

# 30. Table ≠ physical target

```text
#[Table('users')]
```

no selecciona:

```text
database server
tenant
replica
connection instance
```

---

# 31. Logical connection

Si se desea:

```php
#[Entity]
#[Connection('customers')]
```

deberá significar:

```text
LogicalConnectionName(customers)
```

no una conexión viva.

---

# 32. `#[Id]`

Declara un field como parte del identificador.

```php
#[Id]
#[Column]
private int $id;
```

---

# 33. Composite ID

Múltiples:

```php
#[Id]
private int $orderId;

#[Id]
private int $productId;
```

producen un identificador compuesto si la entidad/estrategia lo permite.

---

# 34. Orden de composite IDs

El orden deberá ser determinista.

No deberá depender accidentalmente de Reflection.

Podrá derivarse de:

```text
explicit position
canonical property order
compiled declaration order
```

según contrato definido.

---

# 35. `#[GeneratedValue]`

Ejemplo:

```php
#[Id]
#[GeneratedValue('ulid')]
private UserId $id;
```

---

# 36. Estrategias

Podrá normalizar:

```text
AUTO
IDENTITY
SEQUENCE
UUID
ULID
CUSTOM
NONE
```

---

# 37. Custom generator

Ejemplo:

```php
#[GeneratedValue(
    strategy: 'custom',
    generator: SnowflakeGenerator::class,
)]
```

se traducirá a una referencia.

No deberá instanciarse durante mapping.

---

# 38. `#[Column]`

Será uno de los attributes centrales.

Ejemplo:

```php
#[Column(
    name: 'email_address',
    type: 'email',
    length: 254,
    nullable: false,
)]
private Email $email;
```

---

# 39. Column arguments

El modelo podrá soportar:

```text
name
type
length
precision
scale
nullable
unique hint
insertable
updatable
generated
options
```

con cautela sobre opciones vendor-specific.

---

# 40. Column declaration

Debe traducirse a múltiples paths:

```text
fields.email.column.name
fields.email.type
fields.email.length
fields.email.nullable
```

en vez de convertirse en un blob opaco.

---

# 41. Type inference

Ejemplo:

```php
#[Column]
private string $name;
```

puede permitir:

```text
PHP string
    ↓
ORM string
```

por el resolver del Mapping System.

---

# 42. Explicit type

```php
#[Column(type: 'json')]
private array $preferences;
```

produce:

```text
TypeReference(json)
```

---

# 43. Type attribute argument ≠ SQL type string

Evitar que:

```php
#[Column(type: 'VARCHAR(255)')]
```

sea el modelo principal.

Preferir tipos semánticos.

---

# 44. Nullable inference

```php
#[Column]
private ?string $nickname;
```

podrá inferir:

```text
nullable = true
```

si la política lo habilita.

---

# 45. Explicit nullability

```php
#[Column(nullable: false)]
private ?string $nickname;
```

deberá conservar la contradicción para validación.

No deberá corregirla silenciosamente.

---

# 46. Length

```php
#[Column(length: 150)]
```

representa una restricción de mapping.

La traducción física corresponde posteriormente al Schema/Platform System.

---

# 47. Precision y scale

Ejemplo:

```php
#[Column(
    type: 'decimal',
    precision: 19,
    scale: 4,
)]
private MoneyAmount $amount;
```

---

# 48. Generated field

Ejemplo conceptual:

```php
#[Column(
    generated: GeneratedField::DATABASE,
    insertable: false,
    updatable: false,
)]
private string $searchKey;
```

---

# 49. Column uniqueness

Si se permite:

```php
#[Column(unique: true)]
```

deberá interpretarse como shorthand declarativo.

Internamente:

```text
unique field hint
    ↓
constraint/schema projection
```

No confundir con `UniqueIndex`.

---

# 50. Dedicated constraints

Para configuraciones complejas será preferible un attribute específico:

```php
#[UniqueConstraint(...)]
```

en lugar de sobrecargar `Column`.

---

# 51. Relationship attributes

Catálogo:

```text
#[OneToOne]
#[OneToMany]
#[ManyToOne]
#[ManyToMany]
```

---

# 52. `#[ManyToOne]`

Ejemplo:

```php
#[ManyToOne(target: Organization::class)]
#[JoinColumn(name: 'organization_id')]
private Organization $organization;
```

---

# 53. Target inference

Podrá permitirse:

```php
#[ManyToOne]
private Organization $organization;
```

si el PHP type permite inferir inequívocamente el target.

---

# 54. Target inference limitations

No inferir silenciosamente cuando exista:

```text
union type
intersection type
interface with multiple entities
untyped property
mixed
```

---

# 55. `#[OneToOne]`

Ejemplo:

```php
#[OneToOne(
    target: Profile::class,
    cascade: ['persist'],
)]
private Profile $profile;
```

---

# 56. `#[OneToMany]`

Ejemplo:

```php
#[OneToMany(
    target: Post::class,
    mappedBy: 'author',
)]
private Collection $posts;
```

---

# 57. Collection target

El target no deberá inferirse de:

```php
private Collection $posts;
```

sin información adicional.

Podrá inferirse mediante:

```text
attribute target
generic metadata
static analysis
compiled type information
```

cuando exista evidencia suficiente.

---

# 58. `#[ManyToMany]`

Ejemplo:

```php
#[ManyToMany(
    target: Role::class,
)]
#[JoinTable(name: 'user_roles')]
private Collection $roles;
```

---

# 59. Relationship arguments

Podrán incluir:

```text
target
mappedBy
inversedBy
cascade
fetch
orphanRemoval
```

según tipo.

---

# 60. Cascade enum

Preferir:

```php
cascade: [
    Cascade::PERSIST,
    Cascade::REMOVE,
]
```

sobre strings arbitrarios cuando la ergonomía lo permita.

---

# 61. Fetch strategy

Preferir enum:

```php
fetch: FetchStrategy::LAZY
```

con opciones:

```text
LAZY
EAGER
EXPLICIT
BATCH
```

---

# 62. Relationship attribute ≠ runtime loader

`FetchStrategy::LAZY` es metadata.

No crea proxy ni ejecuta query.

---

# 63. `#[JoinColumn]`

Ejemplo:

```php
#[JoinColumn(
    name: 'author_id',
    referencedColumn: 'id',
    nullable: false,
)]
```

---

# 64. Preferencia semántica

Cuando sea posible se preferirá:

```text
referencedField = id
```

sobre:

```text
referencedColumn = id
```

para evitar acoplamiento físico innecesario.

---

# 65. Legacy schemas

Aun así deberá poder declararse:

```php
#[JoinColumn(
    name: 'COD_CLI',
    referencedColumn: 'ID_CLIENTE',
)]
```

para schemas legacy.

---

# 66. Repeatable JoinColumn

Para composite joins:

```php
#[JoinColumn(name: 'order_id', referencedColumn: 'order_id')]
#[JoinColumn(name: 'line_id', referencedColumn: 'line_id')]
```

podrá utilizarse un attribute repeatable.

---

# 67. Repeatability

PHP:

```php
#[Attribute(
    Attribute::TARGET_PROPERTY |
    Attribute::IS_REPEATABLE
)]
```

cuando corresponda.

---

# 68. `#[JoinColumns]`

Opcionalmente podrá existir un contenedor:

```php
#[JoinColumns([
    new JoinColumn(...),
    new JoinColumn(...),
])]
```

pero no deberá ser necesario si repeatable attributes ofrecen mejor API.

---

# 69. `#[JoinTable]`

Ejemplo:

```php
#[JoinTable(
    name: 'user_roles',
)]
```

---

# 70. Join table details

Podrá declarar:

```text
name
join columns
inverse join columns
schema
```

sin convertirse en SchemaDefinition.

---

# 71. `#[Embeddable]`

Ejemplo:

```php
#[Embeddable]
final readonly class Address
{
    #[Column]
    public string $street;

    #[Column]
    public string $city;
}
```

---

# 72. `#[Embedded]`

Ejemplo:

```php
#[Embedded(
    class: Address::class,
    prefix: 'address_',
)]
private Address $address;
```

---

# 73. Embedded class inference

Podrá inferirse de:

```php
private Address $address;
```

cuando sea inequívoco.

---

# 74. Prefix semantics

Distinguir:

```text
prefix unspecified
prefix explicitly null
prefix empty
prefix explicit string
```

si esas variantes poseen semántica diferente.

---

# 75. Embeddable ≠ Entity

Una clase:

```php
#[Embeddable]
final class Address
```

no entra automáticamente al IdentityMap como entidad independiente.

---

# 76. `#[MappedSuperclass]`

Ejemplo:

```php
#[MappedSuperclass]
abstract class TimestampedEntity
{
    #[Column]
    protected DateTimeImmutable $createdAt;
}
```

---

# 77. Entity + MappedSuperclass conflict

Una clase no deberá ser simultáneamente:

```text
ENTITY
+
MAPPED_SUPERCLASS
```

salvo que una futura semántica lo defina explícitamente.

Por defecto será error.

---

# 78. Inheritance

Catálogo:

```text
#[Inheritance]
#[DiscriminatorColumn]
#[DiscriminatorMap]
```

---

# 79. `#[Inheritance]`

Ejemplo:

```php
#[Entity]
#[Inheritance(
    strategy: InheritanceStrategy::SINGLE_TABLE
)]
abstract class Payment
{
}
```

---

# 80. DiscriminatorColumn

```php
#[DiscriminatorColumn(
    name: 'payment_type',
    type: 'string',
)]
```

---

# 81. DiscriminatorMap

```php
#[DiscriminatorMap([
    'card' => CardPayment::class,
    'cash' => CashPayment::class,
])]
```

---

# 82. Discriminator validation

Deberá verificar:

- target classes válidas;
- keys únicas;
- inheritance graph válido;
- class membership;
- no mappings contradictorios.

---

# 83. Version

```php
#[Version]
#[Column]
private int $version;
```

---

# 84. Version attribute

`#[Version]` solo declara:

```text
entity.version.field = version
```

El optimistic locking se implementará posteriormente.

---

# 85. ChangeTracking

Ejemplo:

```php
#[Entity]
#[ChangeTracking(
    ChangeTrackingPolicy::SNAPSHOT
)]
final class User
{
}
```

---

# 86. ChangeTracking ≠ tracking

El attribute no crea snapshots ni compara propiedades.

---

# 87. Lifecycle attributes

Catálogo inicial:

```text
#[PrePersist]
#[PostPersist]
#[PreUpdate]
#[PostUpdate]
#[PreRemove]
#[PostRemove]
#[PostLoad]
```

---

# 88. Ejemplo

```php
#[PrePersist]
public function initializeTimestamps(): void
{
}
```

---

# 89. Translation

Produce:

```text
entity.lifecycle.prePersist
    += MethodReference(
        User::initializeTimestamps
    )
```

---

# 90. No execution during mapping

El reader jamás deberá ejecutar:

```text
initializeTimestamps()
```

---

# 91. Lifecycle method validation

Podrá validar:

- visibility;
- static/non-static policy;
- allowed parameters;
- return type;
- duplicate registration;
- incompatible lifecycle event.

---

# 92. Lifecycle ordering

Cuando existan varios callbacks, el orden deberá ser determinista.

Podrá definirse mediante:

```text
priority
declaration order
stable method identity
```

pero nunca orden accidental de Reflection.

---

# 93. Attribute definitions

Cada attribute oficial deberá estar descrito por:

```text
AttributeDefinition
```

---

# 94. AttributeDefinition model

Conceptualmente:

```php
final readonly class AttributeDefinition
{
    public function __construct(
        public AttributeName $name,
        public AttributeTargetSet $targets,
        public bool $repeatable,
        public AttributeArgumentSchema $arguments,
        public AttributeTranslatorId $translator,
    ) {}
}
```

---

# 95. Registry

```text
AttributeRegistry
```

contendrá definitions conocidas.

---

# 96. Registry rules

Deberá ser:

```text
deterministic
collision-aware
namespaced
frozen after bootstrap
```

---

# 97. Attribute identity

La identidad no deberá depender únicamente del short name.

Correcto:

```text
VoltStack\Quantum\Database\ORM\Mapping\Attribute\Entity
```

No:

```text
Entity
```

como identidad interna global.

---

# 98. Aliases

Podrá existir:

```text
AttributeAliasRegistry
```

para compatibilidad controlada.

---

# 99. Alias example

Una futura migración de namespace podría permitir:

```text
Old\Column
    ↓ alias
VoltStack\Column
```

durante un periodo de compatibilidad.

---

# 100. Alias ≠ duplicate implementation

El alias deberá traducirse al mismo:

```text
AttributeDefinitionId
```

---

# 101. Alias collision

Dos aliases resolviendo ambiguamente deberán fallar.

---

# 102. Deprecated attribute

La registry podrá almacenar:

```text
ACTIVE
DEPRECATED
REMOVED
```

---

# 103. Deprecated diagnostics

Ejemplo:

```text
DB-ATTRIBUTE-DEPRECATED-001

Attribute:
OldColumn

Replacement:
Column

Removal:
VoltStack 3.0
```

---

# 104. Attribute arguments

Se introduce:

```text
AttributeArguments
```

como representación tipada.

---

# 105. Raw arguments ≠ normalized arguments

Antes:

```text
[
    "type" => "string",
    "length" => 150
]
```

Después:

```text
ColumnAttributeArguments
├── TypeReference(STRING)
└── Length(150)
```

---

# 106. Argument schema

Cada definition deberá declarar:

```text
required arguments
optional arguments
types
enums
defaults
constraints
```

---

# 107. Unknown argument

Deberá ser error.

Ejemplo:

```php
#[Column(lenght: 150)]
```

no deberá ignorarse por typo.

---

# 108. Missing argument

Ejemplo:

```php
#[JoinColumn()]
```

puede ser válido si convention puede resolver el nombre.

Pero:

```php
#[DiscriminatorMap()]
```

podría requerir argumentos.

---

# 109. Attribute defaults

Distinguir:

```text
argument omitted
```

de:

```text
argument explicitly set to null
```

cuando tenga significado.

---

# 110. Attribute Validator

Se propone:

```php
interface AttributeValidator
{
    public function validate(
        AttributeDescriptor $attribute,
        AttributeValidationContext $context,
    ): AttributeValidationResult;
}
```

---

# 111. Validation levels

```text
SYNTAX
TARGET
LOCAL_SEMANTIC
CROSS_ATTRIBUTE
```

La resolución cross-entity completa permanece en el Mapping System.

---

# 112. Syntax validation

Verifica:

```text
argument shape
argument types
enum values
required values
ranges
```

---

# 113. Target validation

Ejemplo:

```php
#[Entity]
private string $name;
```

deberá fallar porque `Entity` requiere class target.

---

# 114. Local semantic validation

Ejemplo:

```php
#[Column(length: -1)]
private string $name;
```

es inválido.

---

# 115. Cross-attribute validation

Ejemplo:

```php
#[Id]
#[OneToMany]
private Collection $children;
```

deberá rechazarse bajo las reglas normales.

---

# 116. Otro conflicto

```php
#[Column]
#[ManyToOne]
private User $owner;
```

deberá considerarse conflicto salvo que una extensión defina una semántica especial.

---

# 117. Validation result

```text
PASS
FAIL
UNKNOWN
```

Podrá incluir warnings.

---

# 118. UNKNOWN

Si una regla depende de información aún no resuelta:

```text
target entity metadata unavailable
```

puede diferirse a Entity Mapping Resolution.

---

# 119. Attribute Translator

Cada attribute deberá traducirse a declaraciones del documento 116.

---

# 120. Contrato

```php
interface AttributeTranslator
{
    public function supports(
        AttributeDescriptor $attribute,
    ): bool;

    public function translate(
        AttributeDescriptor $attribute,
        AttributeTranslationContext $context,
    ): iterable;
}
```

---

# 121. Translator output

Salida:

```text
EntityMappingDeclaration[]
```

No:

```text
EntityMetadata
```

---

# 122. Example: Entity

Entrada:

```php
#[Entity]
final class User {}
```

Salida conceptual:

```text
Declaration
├── path: entity.kind
├── value: ENTITY
├── source: voltstack.orm.attributes
└── provenance: User.php:10
```

---

# 123. Example: Table

```php
#[Table('users')]
```

produce:

```text
entity.storage.table = TableReference(users)
```

---

# 124. Example: Column

```php
#[Column(
    name: 'email_address',
    length: 254,
)]
```

puede producir:

```text
fields.email.column.name = email_address
fields.email.length = 254
```

---

# 125. Example: Relationship

```php
#[ManyToOne(
    target: User::class,
    inversedBy: 'posts',
)]
```

produce:

```text
relationships.author.kind = MANY_TO_ONE
relationships.author.target = User
relationships.author.inversedBy = posts
```

---

# 126. Provenance preservation

Cada declaración deberá conservar:

```text
attribute class
source class
property/method
file
line
arguments
source ID
```

cuando esté disponible.

---

# 127. Provenance after compilation

Un compiled manifest deberá conservar suficiente provenance para diagnostics.

No deberá perder toda la ubicación original.

---

# 128. Source line stability

La línea de código podrá cambiar sin que necesariamente cambie el fingerprint semántico.

Por tanto:

```text
diagnostic provenance
```

y:

```text
semantic fingerprint inputs
```

serán distintos.

---

# 129. Fingerprint

Se introduce:

```text
AttributeMappingFingerprint
```

---

# 130. Fingerprint inputs

Podrá incluir:

```text
attribute semantic identity
normalized arguments
target identity
mapping format version
translator version
extension semantic version
```

---

# 131. Fingerprint exclusions

Excluir:

```text
absolute file path
line number
build timestamp
request ID
object identity
```

salvo que exista una razón explícita.

---

# 132. Attribute Compilation

El sistema deberá soportar precompilación.

Pipeline:

```text
PHP Sources
    ↓
Attribute Scanner
    ↓
Attribute Descriptors
    ↓
Validation
    ↓
Translation
    ↓
Canonical Declarations
    ↓
Compiled Attribute Manifest
```

---

# 133. Compiled manifest

Se propone:

```text
CompiledAttributeManifest
```

---

# 134. Manifest contents

Puede contener:

```text
format version
framework version
source fingerprint
entity descriptors
attribute descriptors
mapping declarations
translator fingerprint
extension fingerprint
provenance index
```

---

# 135. Manifest ≠ Metadata cache

Distinguir:

```text
CompiledAttributeManifest
≠
CompiledEntityMapping
≠
EntityMetadataCache
```

---

# 136. Compilation levels

VoltStack podrá permitir:

```text
LEVEL 0
raw reflection

LEVEL 1
compiled attribute descriptors

LEVEL 2
compiled mapping declarations

LEVEL 3
compiled canonical EntityMapping

LEVEL 4
compiled EntityMetadata
```

La configuración de producción podrá elegir el nivel apropiado.

---

# 137. Production recommendation

Preferencia:

```text
Source Code
    ↓ build/deploy
Compiled Mapping/Metadata
    ↓ runtime
ORM
```

evitando reflection repetitiva por request.

---

# 138. Development recommendation

```text
Source Code
    ↓
Reflection
    ↓
automatic invalidation
    ↓
Mapping
```

para mejorar DX.

---

# 139. Reflection cost

El sistema deberá medir por separado:

```text
class discovery
reflection creation
attribute enumeration
argument normalization
translation
validation
```

para identificar costes reales.

---

# 140. Reflection once ≠ Reflection per query

Regla:

> Incluso en desarrollo, Reflection ORM nunca deberá ejecutarse por cada query.

---

# 141. Runtime cache

Como mínimo:

```text
class → AttributeDescriptorCollection
```

podrá cachearse durante una generación de mapping.

---

# 142. Worker cache

En FrankenPHP:

```text
Worker
├── Frozen Attribute Registry
├── Compiled Attribute Manifest
├── Compiled Entity Mapping
└── request scopes
```

---

# 143. Hot reload

En desarrollo deberá existir invalidación cuando cambie:

```text
entity source
attribute class
mapping configuration
translator
extension
```

---

# 144. Generation model

Puede utilizarse:

```text
MappingGenerationId
```

---

# 145. Generation swap

En persistent runtime:

```text
Generation N
     ↓
compile N+1
     ↓
validate N+1
     ↓
atomic registry swap
```

sin mutar mappings ya utilizados por requests activos.

---

# 146. Request isolation

Request A podrá terminar con:

```text
Generation N
```

mientras Request B nuevo utiliza:

```text
Generation N+1
```

si el runtime soporta este modelo.

---

# 147. No in-place mutation

Nunca:

```text
modify global EntityMapping while requests use it
```

---

# 148. Custom Attributes

VoltStack deberá permitir atributos personalizados.

Ejemplo:

```php
#[Encrypted]
#[Column]
private string $taxId;
```

---

# 149. Extension flow

```text
#[Encrypted]
      ↓
Custom AttributeDefinition
      ↓
Custom AttributeTranslator
      ↓
Extension Mapping Declaration
      ↓
Entity Mapping
```

---

# 150. Extension registration

```text
AttributeExtensionRegistry
```

permitirá registrar:

```text
attribute definition
validator
translator
optional diagnostics
```

---

# 151. Extension namespace

Los custom attributes deberán tener identidad namespaced.

Ejemplo:

```text
acme.orm.encrypted
```

---

# 152. Extension collisions

Dos extensions reclamando la misma identidad serán error.

---

# 153. Custom attribute ≠ arbitrary execution

Un custom attribute no deberá ejecutar código de persistencia durante discovery.

---

# 154. Safe descriptors

Preferir extraer:

```text
class name
literal/scalar arguments
class references
enum references
arrays
```

a ejecutar constructores con efectos secundarios.

---

# 155. Constructor purity

Los attributes oficiales deberán tener constructores:

```text
pure
side-effect free
I/O free
service-free
```

---

# 156. Prohibido en constructors

No:

```php
public function __construct()
{
    Database::query(...);
}
```

ni:

```php
$this->tenant = Tenant::current();
```

---

# 157. Attribute object immutability

Los attributes oficiales deberían ser:

```php
final readonly class Column
```

cuando sea práctico.

---

# 158. Attribute class inheritance

Preferir attributes finales para reducir semántica implícita.

Extensibilidad deberá realizarse mediante registry/translator, no herencia arbitraria del attribute base.

---

# 159. Meta-attributes

VoltStack podría soportar en el futuro:

```text
attributes that describe mapping attributes
```

pero no deberán ser necesarios para V1.

---

# 160. Attribute groups

Podrá existir un mecanismo para agrupar semántica repetitiva.

Ejemplo conceptual:

```php
#[Timestamps]
final class User
{
}
```

que una extensión traduzca a declaraciones específicas.

---

# 161. Macro attributes

Un attribute podrá producir múltiples declaraciones.

```text
#[Timestamps]
    ↓
createdAt mapping
updatedAt mapping
lifecycle hints
```

siempre que la expansión sea:

```text
deterministic
visible
diagnosable
```

---

# 162. Macro ≠ hidden magic

`mapping:explain` deberá mostrar exactamente qué declaraciones produjo.

---

# 163. Attribute composition

Múltiples attributes sobre un mismo target deberán poder colaborar.

Ejemplo:

```php
#[Id]
#[GeneratedValue('ulid')]
#[Column(type: 'ulid')]
private UserId $id;
```

---

# 164. Composition responsibility

El Attribute Mapping System traduce cada declaración.

La composición definitiva pertenece a:

```text
EntityMappingComposer
```

del documento 116.

---

# 165. Attribute local conflict detection

Aun así podrá detectar conflictos obvios tempranos.

Ejemplo:

```php
#[ManyToOne]
#[OneToMany]
private mixed $relation;
```

---

# 166. Repeatable attributes

Un attribute repeatable deberá declarar explícitamente su estrategia:

```text
APPEND
MERGE
MULTI_VALUE
```

---

# 167. Duplicate non-repeatable

Ejemplo inválido:

```php
#[Column]
#[Column]
private string $name;
```

si `Column` no es repeatable.

---

# 168. Attribute order

La semántica no deberá depender normalmente de:

```php
#[Id]
#[Column]
```

vs:

```php
#[Column]
#[Id]
```

---

# 169. Explicit order

Cuando el orden sí importe deberá modelarse como dato:

```php
#[PrePersist(priority: 100)]
```

no mediante posición textual accidental.

---

# 170. PHP promoted properties

Debe soportarse:

```php
public function __construct(
    #[Id]
    #[Column]
    private UserId $id,

    #[Column]
    private string $name,
) {}
```

---

# 171. Promoted property identity

El sistema deberá evitar duplicar:

```text
constructor parameter attribute
+
promoted property attribute
```

cuando PHP exponga ambos aspectos.

---

# 172. Parameter attributes

Si un attribute es válido sobre promoted parameters, el reader deberá normalizar el target semántico correctamente.

---

# 173. Readonly properties

Ejemplo:

```php
#[Column]
public readonly string $email;
```

deberá conservar:

```text
PHP readonly semantics
```

para que Hydration/Metadata validen una estrategia compatible.

---

# 174. Readonly class

Debe soportarse:

```php
#[Entity]
readonly class Currency
{
}
```

solo cuando el modelo ORM correspondiente sea válido.

No deberá asumirse automáticamente que toda readonly class es ValueObject.

---

# 175. Constructor mapping

Attributes futuros podrían permitir:

```php
#[HydrationConstructor]
```

o:

```php
#[HydrateFrom('email')]
```

pero deben integrarse al mismo sistema de declarations.

---

# 176. Property visibility

Attributes deberán funcionar con:

```text
public
protected
private
```

La accesibilidad runtime corresponde al Hydration/Persistence System.

---

# 177. Static properties

Por defecto:

```php
#[Column]
private static string $foo;
```

deberá rechazarse.

---

# 178. Traits

Los attributes provenientes de traits deberán manejarse explícitamente.

Ejemplo:

```php
trait HasTimestamps
{
    #[Column]
    private DateTimeImmutable $createdAt;
}
```

---

# 179. Trait provenance

Deberá conservarse:

```text
declaring trait
consuming entity
effective property
```

---

# 180. Trait collisions

Si dos traits aportan mappings incompatibles al mismo path, deberá reportarse conflicto.

---

# 181. Parent classes

Attributes heredados deberán seguir reglas explícitas.

No asumir que todo class attribute PHP es semánticamente heredable.

---

# 182. Attribute inheritance policy

Cada `AttributeDefinition` podrá declarar:

```text
NOT_INHERITED
INHERITED
COMPOSED
OVERRIDABLE
```

a nivel ORM.

---

# 183. PHP inheritance ≠ mapping inheritance

La ausencia de inheritance automática de PHP Attributes no deberá obligar al ORM a carecer de inheritance semántica.

VoltStack deberá resolverla en su propio modelo.

---

# 184. Interfaces

Por defecto, mapping persistente definido en interfaces deberá manejarse con extrema cautela.

Una interface no representa automáticamente una Entity.

---

# 185. Abstract classes

Podrán ser:

```text
abstract Entity
MappedSuperclass
inheritance root
```

según attributes.

---

# 186. Enums

PHP enums podrán aparecer como:

```text
field type
discriminator value
attribute argument
```

sin convertirse automáticamente en entidades.

---

# 187. Attribute argument enums

Preferencia:

```php
#[GeneratedValue(
    strategy: GenerationStrategy::ULID
)]
```

sobre strings cuando no afecte excesivamente la ergonomía.

---

# 188. Class-string arguments

Ejemplo:

```php
#[ManyToOne(target: User::class)]
```

deberá normalizarse a:

```text
ClassReference(User)
```

---

# 189. No class instantiation

Resolver `User::class` no deberá instanciar `User`.

---

# 190. String references

Podrán aceptarse aliases lógicos cuando exista registry explícita.

Pero:

```text
User::class
```

será preferido para referencias PHP directas.

---

# 191. Naming conventions

Attributes incompletos podrán delegar defaults al NamingStrategy.

Ejemplo:

```php
#[Entity]
final class UserProfile
```

sin `#[Table]`.

Entonces:

```text
UserProfile
    ↓
NamingStrategy
    ↓
user_profiles
```

---

# 192. Explicit declaration wins

Si:

```php
#[Table('profiles')]
```

la naming strategy no deberá reemplazarlo.

---

# 193. Convention source vs attribute source

El Mapping System deberá preservar que:

```text
profiles
```

provino del attribute y no de convention.

---

# 194. Model API coexistence

Ejemplo:

```php
#[Entity]
final class User extends Model
{
    protected string $table = 'users';
}
```

puede producir declaraciones desde:

```text
Attribute source
Model convention source
```

El documento 116 resolverá precedencia/conflictos.

---

# 195. No special attribute privilege

Attributes son una fuente importante, pero no deberán bypassar:

```text
composition
validation
normalization
conflict detection
```

---

# 196. Diagnostics

Cada error deberá mostrar contexto útil.

Ejemplo:

```text
DB-ATTRIBUTE-REL-004

Entity:
App\Entity\Post

Property:
$author

Attribute:
#[ManyToOne]

Problem:
Target entity cannot be inferred.

PHP Type:
UserInterface

Suggested fix:
Specify target explicitly.

Example:
#[ManyToOne(target: User::class)]
```

---

# 197. Error location

Cuando sea posible:

```text
src/Entity/Post.php:42
```

---

# 198. Error snippets

El CLI podrá mostrar fragmentos pequeños de código en development.

No será requisito del core domain model.

---

# 199. Error codes

Los diagnostics deberán utilizar IDs estables:

```text
DB-ATTRIBUTE-ENTITY-*
DB-ATTRIBUTE-FIELD-*
DB-ATTRIBUTE-ID-*
DB-ATTRIBUTE-REL-*
DB-ATTRIBUTE-EMBED-*
DB-ATTRIBUTE-INHERIT-*
DB-ATTRIBUTE-LIFECYCLE-*
DB-ATTRIBUTE-EXT-*
```

---

# 200. Error hierarchy

```text
DatabaseOrmException
└── AttributeMappingException
    ├── AttributeDiscoveryException
    ├── AttributeReadException
    ├── AttributeDescriptorException
    ├── AttributeDefinitionException
    ├── AttributeRegistryException
    ├── AttributeRegistryCollisionException
    ├── AttributeTargetException
    ├── AttributeArgumentException
    ├── AttributeValidationException
    ├── AttributeConflictException
    ├── AttributeTranslationException
    ├── AttributeAliasException
    ├── AttributeAliasCollisionException
    ├── AttributeCompilationException
    ├── AttributeManifestException
    ├── AttributeManifestVersionException
    ├── AttributeManifestCorruptionException
    ├── AttributeFingerprintException
    ├── AttributeExtensionException
    └── AttributeMappingInvariantException
```

---

# 201. CLI

Se proponen comandos futuros:

```text
volt database:mapping:attributes
volt database:mapping:validate
volt database:mapping:compile
volt database:mapping:explain
```

---

# 202. Inspect attributes

Ejemplo:

```text
volt database:mapping:attributes App\Entity\User
```

Salida conceptual:

```text
Entity: App\Entity\User

Class:
  #[Entity]
  #[Table("users")]

Properties:

$id
  #[Id]
  #[GeneratedValue(ULID)]
  #[Column(type: ulid)]

$email
  #[Column(type: email, length: 254)]
```

---

# 203. Explain translation

```text
volt database:mapping:explain App\Entity\User.email
```

podrá mostrar:

```text
Source:
PHP Attribute

Attribute:
#[Column(type: "email", length: 254)]

Translated declarations:
fields.email.type   = email
fields.email.length = 254

Normalized:
EntityFieldMapping(...)
```

---

# 204. Compile command

```text
volt database:mapping:compile
```

deberá poder:

1. discover;
2. read;
3. validate;
4. translate;
5. compose;
6. resolve;
7. validate mapping;
8. compile;
9. fingerprint;
10. write artifacts.

---

# 205. Strict compilation

Production build deberá poder usar:

```text
--strict
```

para convertir warnings seleccionados en build failures.

---

# 206. CI integration

El pipeline CI podrá ejecutar:

```text
database:mapping:validate
database:mapping:compile --strict
```

sin abrir una base de datos cuando solo se realiza validación estática.

---

# 207. Schema-aware validation

Una fase opcional posterior podrá comparar mappings con Schema Metadata.

Eso no deberá contaminar la fase básica de attribute reading.

---

# 208. No hidden DB I/O

Regla absoluta:

```text
Reading PHP Attributes
```

no deberá causar:

```text
database connection
schema introspection
query
```

---

# 209. Attribute cache

Se podrán distinguir:

```text
DescriptorCache
DeclarationCache
CompiledManifestCache
```

---

# 210. Cache invalidation inputs

Como mínimo:

```text
source fingerprint
attribute registry fingerprint
translator registry fingerprint
mapping format version
extension fingerprint
```

---

# 211. Registry fingerprint

La colección frozen de definitions deberá poseer:

```text
AttributeRegistryFingerprint
```

---

# 212. Translator registry

Se recomienda:

```text
AttributeTranslatorRegistry
```

---

# 213. Translator collision

Dos translators reclamando de manera exclusiva el mismo attribute deberán producir error.

---

# 214. Translator composition

Algunos attributes podrán permitir múltiples translators únicamente si su contrato lo declara.

---

# 215. No last-wins translator

Nunca:

```text
translator registered last wins
```

---

# 216. Deterministic translation

Mismo:

```text
descriptor
registry
translator versions
context
```

deberá producir las mismas declaraciones.

---

# 217. Translation context

Puede contener:

```text
entity candidate
property/method reference
mapping profile
naming policy reference
mapping format version
```

---

# 218. Translation context prohibitions

No:

```text
current HTTP request
current User
current live EntityManager
current UnitOfWork
current Transaction
current Connection
```

---

# 219. Persistent runtime

El diseño deberá ser seguro para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 220. Shared immutable state

Puede compartirse:

```text
AttributeRegistry
AttributeTranslatorRegistry
CompiledAttributeManifest
AttributeAliasRegistry
```

una vez frozen.

---

# 221. Operation-scoped state

Debe permanecer aislado:

```text
discovery session
build diagnostics
temporary translation buffers
mapping generation
```

cuando sean mutables.

---

# 222. Coroutine safety

No deberá existir:

```php
static ?ClassReference $currentEntity;
```

en translators.

---

# 223. Tenant isolation

Los attributes no deberán contener current tenant state.

Incorrecto:

```php
#[Table(Tenant::current()->tableName())]
```

---

# 224. Tenant-neutral mapping

Preferir:

```php
#[Table('users')]
```

y después:

```text
TenantContext
    ↓
physical target resolution
```

---

# 225. Security

Attribute discovery deberá tratar classes/extensions como código confiable de aplicación/paquete.

No deberá permitir que input HTTP seleccione arbitrariamente:

```text
attribute class
entity class
translator
mapping provider
```

---

# 226. Sensitive arguments

Los attributes ORM no deberán almacenar:

```text
passwords
database credentials
private keys
encryption keys
tokens
```

---

# 227. Security references

Permitido:

```php
#[Encrypted(policy: 'pii')]
```

si `pii` es:

```text
EncryptionPolicyId
```

y no la clave real.

---

# 228. Serialization

Compiled manifests deberán utilizar un formato:

```text
versioned
deterministic
validated
```

---

# 229. PHP serialize

No deberá considerarse contrato arquitectónico permanente:

```php
serialize($reflectionObjects)
```

---

# 230. Portable representation

Preferir estructuras compiladas de:

```text
scalars
arrays
stable IDs
class references
enum references
typed DTOs
```

---

# 231. Manifest integrity

El loader deberá validar:

```text
format version
fingerprint
registry compatibility
translator compatibility
source compatibility
```

---

# 232. Corrupt manifest

Nunca deberá cargarse parcialmente como si fuese válido.

---

# 233. Fallback

En development:

```text
invalid manifest
    ↓
rebuild
```

puede permitirse.

En production:

```text
invalid manifest
    ↓
fail-fast / controlled rebuild policy
```

según configuración.

---

# 234. No silent fallback production

Una configuración estricta deberá evitar que un deployment accidentalmente vuelva a Reflection masiva sin advertencia.

---

# 235. Preloading

El sistema podrá aprovechar PHP preloading cuando esté disponible.

Pero:

```text
preloading
≠
required architecture
```

---

# 236. OPcache

Los attributes PHP se benefician del código compilado por PHP, pero VoltStack seguirá evitando interpretación ORM repetitiva.

---

# 237. Performance target

El hot path ORM ideal será:

```text
EntityMetadata lookup
```

no:

```text
Reflection
→ Attribute reading
→ Translation
→ Mapping
→ Metadata
```

---

# 238. Cold path

```text
Reflection
→ Attributes
→ Mapping
→ Metadata
```

pertenece al:

```text
build/bootstrap/cold path
```

---

# 239. Performance formula

```text
RuntimeAttributeCost
≈
ManifestLookup
```

en producción compilada.

Idealmente:

```text
RuntimeReflectionCost
≈ 0
```

para mappings ya compilados.

---

# 240. Telemetry

Métricas:

```text
orm.attribute.discovery.duration
orm.attribute.read.duration
orm.attribute.translate.duration
orm.attribute.validation.errors
orm.attribute.manifest.hit
orm.attribute.manifest.miss
orm.attribute.manifest.rebuild
orm.attribute.registry.size
```

---

# 241. Build tracing

```text
Attribute Mapping Build
├── Discover Classes
├── Read Attributes
├── Normalize Descriptors
├── Validate
├── Translate
├── Fingerprint
└── Compile Manifest
```

---

# 242. No sensitive telemetry

No registrar indiscriminadamente:

```text
attribute arguments
```

si extensions pudieran contener información sensible.

---

# 243. Testing strategy

El sistema deberá probar tanto Reflection como compiled manifests.

---

# 244. Entity tests

```text
valid Entity
invalid target
duplicate Entity
Entity + MappedSuperclass conflict
abstract Entity
```

---

# 245. Field tests

```text
Column
inferred type
explicit type
nullable
length
precision
scale
generated
readonly
static property rejection
```

---

# 246. ID tests

```text
single ID
composite ID
generated ID
assigned ID
custom generator
invalid generated relationship
```

---

# 247. Relationship tests

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
target inference
target ambiguity
mappedBy
inversedBy
join column
composite join
join table
cascade
fetch
orphan removal
```

---

# 248. Embedded tests

```text
Embeddable
Embedded
prefix
nested embedding
cycle
invalid entity/embeddable combination
```

---

# 249. Inheritance tests

```text
MappedSuperclass
SingleTable
Joined
TablePerClass
DiscriminatorColumn
DiscriminatorMap
invalid hierarchy
```

---

# 250. Lifecycle tests

```text
all lifecycle attributes
invalid target
invalid signature
deterministic ordering
priority
duplicate callbacks
```

---

# 251. Repeatable tests

Verificar:

```text
repeatable allowed
non-repeatable duplicate rejected
order independence
canonical ordering
```

---

# 252. Trait tests

```text
trait property mapping
trait provenance
multiple traits
trait collision
override
```

---

# 253. Promoted property tests

```text
constructor promoted property
parameter attributes
property attributes
duplicate normalization
readonly promoted property
```

---

# 254. Extension tests

```text
custom attribute
custom translator
collision
malformed definition
non-deterministic translator detection where possible
registry freeze
```

---

# 255. Manifest tests

```text
compile
load
fingerprint
corruption
wrong version
stale source
changed translator
changed extension
development rebuild
production strict failure
```

---

# 256. Equivalence tests

Debe comprobarse:

```text
Reflection Pipeline
```

vs:

```text
Compiled Manifest Pipeline
```

produciendo:

```text
same canonical mapping declarations
```

---

# 257. Persistent worker tests

Ejecutar múltiples requests simulados y comprobar:

```text
no mapping state leakage
no current entity leakage
no tenant leakage
no mutable registry leakage
```

---

# 258. Invariantes arquitectónicas

## DB-ATTRIBUTE-MAPPING-001
PHP Attributes serán una sintaxis de mapping.

## DB-ATTRIBUTE-MAPPING-002
PHP Attributes no serán EntityMetadata.

## DB-ATTRIBUTE-MAPPING-003
PHP Attributes no serán SchemaModel.

## DB-ATTRIBUTE-MAPPING-004
PHP Attributes no serán migrations.

## DB-ATTRIBUTE-MAPPING-005
Attribute Mapping no ejecutará SQL.

## DB-ATTRIBUTE-MAPPING-006
Attribute Mapping no generará SQL.

## DB-ATTRIBUTE-MAPPING-007
Attribute Mapping no abrirá conexiones.

## DB-ATTRIBUTE-MAPPING-008
Attribute Mapping no ejecutará DDL.

## DB-ATTRIBUTE-MAPPING-009
Attribute Mapping no administrará EntityManager.

## DB-ATTRIBUTE-MAPPING-010
Attribute Mapping no administrará UnitOfWork.

## DB-ATTRIBUTE-MAPPING-011
Attribute Mapping no hidratará entidades.

## DB-ATTRIBUTE-MAPPING-012
Attribute Mapping no persistirá entidades.

## DB-ATTRIBUTE-MAPPING-013
Attribute Mapping no ejecutará lifecycle callbacks.

## DB-ATTRIBUTE-MAPPING-014
Reflection será un mecanismo de discovery, no la arquitectura.

## DB-ATTRIBUTE-MAPPING-015
Compiled manifests podrán sustituir Reflection en runtime.

## DB-ATTRIBUTE-MAPPING-016
Reflection y compiled manifests producirán semántica equivalente.

## DB-ATTRIBUTE-MAPPING-017
AttributeDescriptor será distinto de instancia del attribute.

## DB-ATTRIBUTE-MAPPING-018
Attributes oficiales tendrán constructores sin I/O.

## DB-ATTRIBUTE-MAPPING-019
Attributes oficiales no accederán a servicios runtime.

## DB-ATTRIBUTE-MAPPING-020
Attributes oficiales no dependerán del request actual.

## DB-ATTRIBUTE-MAPPING-021
Attribute source tendrá ID estable.

## DB-ATTRIBUTE-MAPPING-022
Attribute identity será namespaced.

## DB-ATTRIBUTE-MAPPING-023
Short attribute names no serán identidad global suficiente.

## DB-ATTRIBUTE-MAPPING-024
AttributeRegistry será collision-aware.

## DB-ATTRIBUTE-MAPPING-025
AttributeRegistry será frozen después del bootstrap.

## DB-ATTRIBUTE-MAPPING-026
Attribute definitions declararán targets válidos.

## DB-ATTRIBUTE-MAPPING-027
Attribute definitions declararán repeatability.

## DB-ATTRIBUTE-MAPPING-028
Attribute definitions declararán argument schema.

## DB-ATTRIBUTE-MAPPING-029
Unknown arguments no serán ignorados.

## DB-ATTRIBUTE-MAPPING-030
Invalid targets serán errores.

## DB-ATTRIBUTE-MAPPING-031
Argument defaults distinguirán omitted cuando sea semánticamente necesario.

## DB-ATTRIBUTE-MAPPING-032
Attributes traducirán a MappingDeclarations.

## DB-ATTRIBUTE-MAPPING-033
Attributes no construirán EntityMetadata directamente.

## DB-ATTRIBUTE-MAPPING-034
Attributes no bypassarán EntityMappingComposer.

## DB-ATTRIBUTE-MAPPING-035
Attributes no bypassarán EntityMappingValidator.

## DB-ATTRIBUTE-MAPPING-036
Attribute translation será determinista.

## DB-ATTRIBUTE-MAPPING-037
Attribute translation preservará provenance.

## DB-ATTRIBUTE-MAPPING-038
Diagnostic provenance no alterará necesariamente fingerprint semántico.

## DB-ATTRIBUTE-MAPPING-039
`#[Entity]` declarará identidad ORM, no table por sí solo.

## DB-ATTRIBUTE-MAPPING-040
`#[Table]` declarará storage lógico.

## DB-ATTRIBUTE-MAPPING-041
`#[Table]` no resolverá servidor físico.

## DB-ATTRIBUTE-MAPPING-042
`#[Id]` declarará identifier membership.

## DB-ATTRIBUTE-MAPPING-043
Composite identifiers tendrán orden determinista.

## DB-ATTRIBUTE-MAPPING-044
`#[GeneratedValue]` no generará el ID durante mapping.

## DB-ATTRIBUTE-MAPPING-045
Custom generator será una referencia declarativa.

## DB-ATTRIBUTE-MAPPING-046
`#[Column]` será distinto de Schema ColumnDefinition.

## DB-ATTRIBUTE-MAPPING-047
Column type será semántico antes que SQL vendor-specific.

## DB-ATTRIBUTE-MAPPING-048
PHP type inference no elegirá SQL físico prematuramente.

## DB-ATTRIBUTE-MAPPING-049
Nullability contradictions serán preservadas para validación.

## DB-ATTRIBUTE-MAPPING-050
PHP default será distinto de database default.

## DB-ATTRIBUTE-MAPPING-051
Column unique shorthand no será confundido con UniqueIndex.

## DB-ATTRIBUTE-MAPPING-052
Relationship attributes no cargarán relaciones.

## DB-ATTRIBUTE-MAPPING-053
Relationship target inference requerirá evidencia suficiente.

## DB-ATTRIBUTE-MAPPING-054
Target ambiguo será UNKNOWN/error, no guess silencioso.

## DB-ATTRIBUTE-MAPPING-055
`mappedBy` será una referencia declarativa antes de resolution.

## DB-ATTRIBUTE-MAPPING-056
`inversedBy` será una referencia declarativa antes de resolution.

## DB-ATTRIBUTE-MAPPING-057
JoinColumn será distinto de ForeignKeyDefinition.

## DB-ATTRIBUTE-MAPPING-058
Composite joins serán representables.

## DB-ATTRIBUTE-MAPPING-059
Repeatable JoinColumn tendrá semántica explícita.

## DB-ATTRIBUTE-MAPPING-060
JoinTable será distinto de TableDefinition.

## DB-ATTRIBUTE-MAPPING-061
Embeddable será distinto de Entity.

## DB-ATTRIBUTE-MAPPING-062
Embedded mapping no creará IdentityMap entry independiente.

## DB-ATTRIBUTE-MAPPING-063
Embedded target inference deberá ser inequívoca.

## DB-ATTRIBUTE-MAPPING-064
MappedSuperclass será distinto de Entity.

## DB-ATTRIBUTE-MAPPING-065
Entity + MappedSuperclass incompatible será error.

## DB-ATTRIBUTE-MAPPING-066
Inheritance strategy será explícita.

## DB-ATTRIBUTE-MAPPING-067
Discriminator mappings serán validados posteriormente contra el hierarchy graph.

## DB-ATTRIBUTE-MAPPING-068
Version attribute no ejecutará optimistic locking.

## DB-ATTRIBUTE-MAPPING-069
ChangeTracking attribute no realizará change tracking.

## DB-ATTRIBUTE-MAPPING-070
Lifecycle attributes no ejecutarán callbacks.

## DB-ATTRIBUTE-MAPPING-071
Lifecycle callback order será determinista.

## DB-ATTRIBUTE-MAPPING-072
Attribute textual order no controlará semántica salvo contrato explícito.

## DB-ATTRIBUTE-MAPPING-073
Explicit priority será dato, no efecto accidental de Reflection.

## DB-ATTRIBUTE-MAPPING-074
Non-repeatable duplicate será error.

## DB-ATTRIBUTE-MAPPING-075
Repeatable attributes declararán estrategia de composición.

## DB-ATTRIBUTE-MAPPING-076
Promoted properties no producirán mapping duplicado.

## DB-ATTRIBUTE-MAPPING-077
Readonly semantics serán preservadas.

## DB-ATTRIBUTE-MAPPING-078
Static persistent properties serán rechazadas por defecto.

## DB-ATTRIBUTE-MAPPING-079
Trait provenance será preservada.

## DB-ATTRIBUTE-MAPPING-080
Trait mapping conflicts serán visibles.

## DB-ATTRIBUTE-MAPPING-081
PHP attribute inheritance será distinto de ORM mapping inheritance.

## DB-ATTRIBUTE-MAPPING-082
Mapping inheritance tendrá política propia.

## DB-ATTRIBUTE-MAPPING-083
Interfaces no serán entities automáticamente.

## DB-ATTRIBUTE-MAPPING-084
PHP enums no serán entities automáticamente.

## DB-ATTRIBUTE-MAPPING-085
Class-string arguments se normalizarán a ClassReference.

## DB-ATTRIBUTE-MAPPING-086
Class references no instanciarán clases.

## DB-ATTRIBUTE-MAPPING-087
NamingStrategy completará defaults, no reemplazará declaraciones explícitas.

## DB-ATTRIBUTE-MAPPING-088
Attributes no tendrán privilegio para saltarse source conflict resolution.

## DB-ATTRIBUTE-MAPPING-089
Model API y Attributes podrán coexistir como fuentes distintas.

## DB-ATTRIBUTE-MAPPING-090
Source precedence será gobernada por Entity Mapping System.

## DB-ATTRIBUTE-MAPPING-091
Aliases tendrán resolución determinista.

## DB-ATTRIBUTE-MAPPING-092
Alias collisions serán errores.

## DB-ATTRIBUTE-MAPPING-093
Deprecated attributes producirán diagnostics.

## DB-ATTRIBUTE-MAPPING-094
Removed attributes no serán aceptados silenciosamente.

## DB-ATTRIBUTE-MAPPING-095
Custom attributes requerirán registration explícita.

## DB-ATTRIBUTE-MAPPING-096
Custom attributes tendrán IDs namespaced.

## DB-ATTRIBUTE-MAPPING-097
Custom translators no ejecutarán queries.

## DB-ATTRIBUTE-MAPPING-098
Custom translators no ejecutarán migrations.

## DB-ATTRIBUTE-MAPPING-099
Custom translators no abrirán conexiones ocultamente.

## DB-ATTRIBUTE-MAPPING-100
Macro attributes deberán ser expandibles y explicables.

## DB-ATTRIBUTE-MAPPING-101
Macro attributes no ocultarán declaraciones generadas.

## DB-ATTRIBUTE-MAPPING-102
Translator collisions no utilizarán last-wins.

## DB-ATTRIBUTE-MAPPING-103
TranslatorRegistry será frozen.

## DB-ATTRIBUTE-MAPPING-104
TranslationContext no contendrá current User.

## DB-ATTRIBUTE-MAPPING-105
TranslationContext no contendrá live EntityManager.

## DB-ATTRIBUTE-MAPPING-106
TranslationContext no contendrá live UnitOfWork.

## DB-ATTRIBUTE-MAPPING-107
TranslationContext no contendrá live Transaction.

## DB-ATTRIBUTE-MAPPING-108
TranslationContext no contendrá live Connection.

## DB-ATTRIBUTE-MAPPING-109
Attribute fingerprint será determinista.

## DB-ATTRIBUTE-MAPPING-110
Fingerprint no dependerá de line number salvo política explícita.

## DB-ATTRIBUTE-MAPPING-111
Fingerprint no dependerá de build timestamp.

## DB-ATTRIBUTE-MAPPING-112
Compiled manifest tendrá format version.

## DB-ATTRIBUTE-MAPPING-113
Compiled manifest tendrá integrity validation.

## DB-ATTRIBUTE-MAPPING-114
Corrupt manifest no será parcialmente aceptado.

## DB-ATTRIBUTE-MAPPING-115
Manifest compatibility será validada.

## DB-ATTRIBUTE-MAPPING-116
Production podrá operar sin Reflection repetitiva.

## DB-ATTRIBUTE-MAPPING-117
Reflection no ocurrirá por query.

## DB-ATTRIBUTE-MAPPING-118
Reflection no ocurrirá por entity hydration.

## DB-ATTRIBUTE-MAPPING-119
Reflection no ocurrirá por flush para mappings compilados.

## DB-ATTRIBUTE-MAPPING-120
Hot runtime path utilizará EntityMetadata.

## DB-ATTRIBUTE-MAPPING-121
Attribute processing pertenecerá al cold/build path.

## DB-ATTRIBUTE-MAPPING-122
Cache invalidation incluirá source fingerprint.

## DB-ATTRIBUTE-MAPPING-123
Cache invalidation incluirá registry fingerprint.

## DB-ATTRIBUTE-MAPPING-124
Cache invalidation incluirá translator fingerprint.

## DB-ATTRIBUTE-MAPPING-125
Cache invalidation incluirá extension fingerprint.

## DB-ATTRIBUTE-MAPPING-126
Mapping generation no será mutada in-place mientras esté activa.

## DB-ATTRIBUTE-MAPPING-127
Persistent workers compartirán solo registries/manifests immutable.

## DB-ATTRIBUTE-MAPPING-128
Mutable build state será operation-scoped.

## DB-ATTRIBUTE-MAPPING-129
OpenSwoole translators no usarán static current entity state.

## DB-ATTRIBUTE-MAPPING-130
Tenant state no será capturado en attributes.

## DB-ATTRIBUTE-MAPPING-131
Tenant physical target resolution ocurrirá fuera del Attribute Mapping System.

## DB-ATTRIBUTE-MAPPING-132
Attributes ORM no almacenarán credentials.

## DB-ATTRIBUTE-MAPPING-133
Attributes ORM no almacenarán encryption keys.

## DB-ATTRIBUTE-MAPPING-134
Security policy references serán IDs, no secrets.

## DB-ATTRIBUTE-MAPPING-135
Untrusted HTTP input no registrará attributes.

## DB-ATTRIBUTE-MAPPING-136
Untrusted HTTP input no seleccionará translators arbitrarios.

## DB-ATTRIBUTE-MAPPING-137
Compiled manifest serialization será versionada.

## DB-ATTRIBUTE-MAPPING-138
PHP serialize no será contrato permanente del sistema.

## DB-ATTRIBUTE-MAPPING-139
Production strict mode podrá impedir fallback silencioso a Reflection.

## DB-ATTRIBUTE-MAPPING-140
Attribute telemetry no cambiará semántica.

## DB-ATTRIBUTE-MAPPING-141
Telemetry no expondrá secrets de custom attributes.

## DB-ATTRIBUTE-MAPPING-142
Reflection y compiled paths tendrán conformance tests.

## DB-ATTRIBUTE-MAPPING-143
Mismo source semántico producirá mismas MappingDeclarations.

## DB-ATTRIBUTE-MAPPING-144
Mismas MappingDeclarations participarán en el mismo pipeline del documento 116.

## DB-ATTRIBUTE-MAPPING-145
EntityManager nunca deberá interpretar attributes directamente.

## DB-ATTRIBUTE-MAPPING-146
UnitOfWork nunca deberá interpretar attributes directamente.

## DB-ATTRIBUTE-MAPPING-147
Hydrator nunca deberá interpretar attributes directamente.

## DB-ATTRIBUTE-MAPPING-148
Persistence Engine nunca deberá interpretar attributes directamente.

## DB-ATTRIBUTE-MAPPING-149
Query Engine nunca deberá interpretar attributes directamente.

## DB-ATTRIBUTE-MAPPING-150
Todo attribute ORM deberá terminar convertido en semántica canónica antes de participar en el runtime de persistencia.

---

# 259. Anti-patterns

## 259.1 Reflection dentro de EntityManager

Incorrecto:

```php
public function persist(object $entity): void
{
    $reflection = new ReflectionClass($entity);

    foreach ($reflection->getAttributes() as $attribute) {
        // Interpret mapping.
    }
}
```

Correcto:

```text
Build/Bootstrap
    ↓
Attributes
    ↓
Mapping
    ↓
Metadata

Runtime
    ↓
Metadata Lookup
```

---

# 260. Attributes con comportamiento

Incorrecto:

```php
#[Attribute]
final class Column
{
    public function save(object $entity): void
    {
        Database::insert(...);
    }
}
```

El attribute describe.

No ejecuta.

---

# 261. Attribute conectado a DB

Incorrecto:

```php
public function __construct(string $table)
{
    if (!Database::tableExists($table)) {
        throw new Exception();
    }
}
```

La validación física pertenece a una fase explícita posterior.

---

# 262. Vendor SQL

Evitar:

```php
#[Column(type: 'VARCHAR(255) CHARACTER SET utf8mb4')]
```

como API principal.

Preferir:

```php
#[Column(
    type: 'string',
    length: 255,
)]
```

y delegar representación física.

---

# 263. Magic relation inference

Incorrecto:

```php
#[OneToMany]
private Collection $items;
```

y asumir arbitrariamente:

```text
OrderItem
```

sin evidencia.

---

# 264. Attributes por request

Incorrecto conceptualmente:

```text
Request
    ↓
Reflect every entity
    ↓
Read attributes
    ↓
Build mapping
```

cada vez.

---

# 265. Global mutable registry

Incorrecto:

```php
AttributeRegistry::$currentEntity = $entity;
```

---

# 266. Attribute aliases silenciosos

No aceptar nombres desconocidos por similitud.

Ejemplo:

```text
Colum
```

no deberá autocorregirse a:

```text
Column
```

sin una definición de alias explícita.

---

# 267. Arquitectura de clases propuesta

```text
src/Quantum/Database/ORM/Mapping/Attribute/
│
├── Contract/
│   ├── AttributeReader.php
│   ├── AttributeValidator.php
│   ├── AttributeTranslator.php
│   └── AttributeManifestLoader.php
│
├── Definition/
│   ├── AttributeDefinition.php
│   ├── AttributeDefinitionId.php
│   ├── AttributeArgumentSchema.php
│   └── AttributeInheritancePolicy.php
│
├── Descriptor/
│   ├── AttributeDescriptor.php
│   ├── AttributeDescriptorCollection.php
│   ├── AttributeArguments.php
│   ├── AttributeLocation.php
│   ├── AttributeTarget.php
│   └── AttributeProvenance.php
│
├── Registry/
│   ├── AttributeRegistry.php
│   ├── AttributeTranslatorRegistry.php
│   ├── AttributeAliasRegistry.php
│   └── AttributeExtensionRegistry.php
│
├── Reader/
│   ├── ReflectionAttributeReader.php
│   ├── CompiledAttributeReader.php
│   └── CompositeAttributeReader.php
│
├── Validation/
│   ├── AttributeValidationContext.php
│   ├── AttributeValidationResult.php
│   └── AttributeValidationIssue.php
│
├── Translation/
│   ├── AttributeTranslationContext.php
│   └── DefaultAttributeTranslator.php
│
├── Manifest/
│   ├── CompiledAttributeManifest.php
│   ├── AttributeManifestCompiler.php
│   ├── AttributeManifestLoader.php
│   └── AttributeManifestVersion.php
│
├── Fingerprint/
│   ├── AttributeMappingFingerprint.php
│   └── AttributeRegistryFingerprint.php
│
├── Entity/
│   ├── Entity.php
│   ├── Table.php
│   └── MappedSuperclass.php
│
├── Identifier/
│   ├── Id.php
│   └── GeneratedValue.php
│
├── Field/
│   └── Column.php
│
├── Relationship/
│   ├── OneToOne.php
│   ├── OneToMany.php
│   ├── ManyToOne.php
│   ├── ManyToMany.php
│   ├── JoinColumn.php
│   └── JoinTable.php
│
├── Embedded/
│   ├── Embeddable.php
│   └── Embedded.php
│
├── Inheritance/
│   ├── Inheritance.php
│   ├── DiscriminatorColumn.php
│   └── DiscriminatorMap.php
│
├── Version/
│   └── Version.php
│
├── ChangeTracking/
│   └── ChangeTracking.php
│
├── Lifecycle/
│   ├── PrePersist.php
│   ├── PostPersist.php
│   ├── PreUpdate.php
│   ├── PostUpdate.php
│   ├── PreRemove.php
│   ├── PostRemove.php
│   └── PostLoad.php
│
├── Extension/
│   └── ...
│
└── Exception/
    └── ...
```

---

# 268. Namespace público de attributes

La API deberá favorecer imports claros.

Una posibilidad:

```php
use VoltStack\Quantum\Database\ORM\Mapping\Attribute\Entity\Entity;
use VoltStack\Quantum\Database\ORM\Mapping\Attribute\Entity\Table;
use VoltStack\Quantum\Database\ORM\Mapping\Attribute\Field\Column;
```

Sin embargo, esto puede resultar demasiado verboso.

---

# 269. Namespace ergonómico recomendado

Se recomienda evaluar una fachada namespace:

```text
VoltStack\ORM\Attributes
```

o:

```text
VoltStack\Database\ORM\Attributes
```

permitiendo:

```php
use VoltStack\ORM\Attributes as ORM;

#[ORM\Entity]
#[ORM\Table('users')]
final class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue('ulid')]
    #[ORM\Column(type: 'ulid')]
    private UserId $id;
}
```

---

# 270. Separación API/implementación

Podrá existir:

```text
Public:
VoltStack\ORM\Attributes\Entity

Internal:
VoltStack\Quantum\Database\ORM\Mapping\Attribute\...
```

si el framework utiliza una capa pública de aliases/clases delgadas.

---

# 271. Ergonomía objetivo

VoltStack deberá buscar una experiencia aproximadamente:

```php
use VoltStack\ORM\Attributes as ORM;

#[ORM\Entity]
#[ORM\Table('users')]
final class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue(ORM\Generation::ULID)]
    #[ORM\Column]
    private UserId $id;

    #[ORM\Column(length: 150)]
    private string $name;

    #[ORM\Column(type: 'email')]
    private Email $email;

    #[ORM\ManyToOne(
        target: Organization::class,
        inversedBy: 'users',
    )]
    #[ORM\JoinColumn(nullable: false)]
    private Organization $organization;
}
```

---

# 272. Minimal mapping

VoltStack también deberá permitir:

```php
use VoltStack\ORM\Attributes as ORM;

#[ORM\Entity]
final class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    private int $id;

    #[ORM\Column]
    private string $name;
}
```

donde conventions completen:

```text
table = users
column(id) = id
column(name) = name
type(id) = integer
type(name) = string
```

---

# 273. Explicit mapping

Para legacy:

```php
#[ORM\Entity]
#[ORM\Table('TB_USR_001')]
final class User
{
    #[ORM\Id]
    #[ORM\Column(
        name: 'COD_USR',
        type: 'integer',
    )]
    private int $id;

    #[ORM\Column(
        name: 'NM_USR',
        type: 'string',
        length: 80,
    )]
    private string $name;
}
```

La misma arquitectura soportará ambos extremos.

---

# 274. Filosofía de API

```text
Simple by default
Explicit when needed
Strict when ambiguous
Compiled in production
Explainable always
```

---

# 275. Flujo completo

```text
                  PHP ENTITY SOURCE
                         │
                         ▼
               Attribute Discovery
                         │
            ┌────────────┴────────────┐
            ▼                         ▼
      Reflection Reader        Compiled Reader
            │                         │
            └────────────┬────────────┘
                         ▼
               AttributeDescriptor
                         │
                         ▼
                Registry Lookup
                         │
                         ▼
              Argument Validation
                         │
                         ▼
                Target Validation
                         │
                         ▼
               Local Semantics
                         │
                         ▼
              AttributeTranslator
                         │
                         ▼
             MappingDeclarations
                         │
                         ▼
           116 Entity Mapping System
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Compose      Resolve     Normalize
             └───────────┼───────────┘
                         ▼
                     Validate
                         ▼
                Canonical Mapping
                         ▼
            115 Entity Metadata
                         ▼
                    ORM Runtime
```

---

# 276. Fórmula conceptual

```text
AttributeMapping
=
Discover
+
Read
+
Describe
+
Validate
+
Translate
```

No:

```text
AttributeMapping
=
Persist
```

---

# 277. Fórmula de traducción

Para un attribute `a`:

```text
Declarations(a)
=
Translator(
    NormalizeArguments(a),
    Target(a),
    Provenance(a),
    TranslationContext
)
```

---

# 278. Fórmula de equivalencia

```text
Translate(
    ReflectionRead(Source)
)
≈
Translate(
    CompiledRead(
        Compile(Source)
    )
)
```

---

# 279. Fórmula del hot path

```text
Production ORM Hot Path
=
EntityType
→ MetadataRegistry
→ EntityMetadata
```

No:

```text
EntityType
→ Reflection
→ Attributes
→ Mapping
→ Metadata
```

---

# 280. Fórmula de seguridad de persistent runtime

```text
PersistentRuntimeSafe
=
FrozenRegistries
∧
ImmutableManifests
∧
ImmutableMappings
∧
NoRequestStateInAttributes
∧
NoMutableGlobalTranslationContext
```

---

# 281. Master Formula

```text
Database Attribute Mapping System
=
Typed PHP Attribute Catalog
+
Attribute Discovery
+
Safe Attribute Descriptors
+
Reflection Reader
+
Compiled Reader
+
Target Validation
+
Argument Validation
+
Semantic Translation
+
Entity Mapping Declarations
+
Source Provenance
+
Repeatable Attribute Semantics
+
Alias Governance
+
Deprecation Governance
+
Custom Attribute Extensions
+
Deterministic Translator Registry
+
Trait/Inheritance Awareness
+
Promoted Property Support
+
Readonly Awareness
+
Compiled Attribute Manifests
+
Deterministic Fingerprinting
+
Cache Invalidation
+
Production Reflection Elimination
+
Persistent Runtime Isolation
+
Diagnostics
+
Telemetry
+
Entity Mapping Integration
```

---

# 282. Master Rule

> **VoltStack trata los PHP Attributes como una sintaxis declarativa compilable. Los attributes describen intención ORM, pero toda esa intención debe convertirse primero en declaraciones tipadas, pasar por el Entity Mapping System y terminar en Entity Metadata canónica antes de que cualquier componente del runtime ORM pueda utilizarla.**

---

# 283. Resultado arquitectónico

La arquitectura final queda:

```text
Developer
   │
   ▼
PHP Attributes
   │
   ▼
Attribute Mapping System
   │
   ▼
Typed Mapping Declarations
   │
   ▼
Entity Mapping System
   │
   ▼
Canonical Entity Mapping
   │
   ▼
Entity Metadata System
   │
   ├──────────────┬───────────────┐
   ▼              ▼               ▼
EntityManager   Hydrator      UnitOfWork
                                  │
                                  ▼
                         Persistence Engine
                                  │
                                  ▼
                            Query Engine
```

Esto evita que Reflection y Attributes se filtren por todo el ORM.

---

# 284. Bloque ORM hasta este punto

```text
112_DATABASE_ORM_ARCHITECTURE
        ↓
113_DATABASE_ENTITY_MODEL
        ↓
114_DATABASE_MODEL_API_SYSTEM
        ↓
115_DATABASE_ENTITY_METADATA_SYSTEM
        ↓
116_DATABASE_ENTITY_MAPPING_SYSTEM
        ↓
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM
```

Las capas quedan claramente separadas:

```text
Entity Model
     ↓
Developer API
     ↓
Mapping declarations
     ↓
Canonical metadata
     ↓
Runtime ORM
```

---

# 285. Siguiente documento

```text
118_DATABASE_ENTITY_MANAGER_SYSTEM.md
```

Este documento deberá introducir el componente central de coordinación runtime del ORM:

```text
EntityManager
```

y definir cómo coordina:

```text
Entity Metadata
IdentityMap
UnitOfWork
Repositories
Entity Queries
Hydration
Persistence
Transactions
Lifecycle
```

sin convertirse en un objeto monolítico que implemente internamente todos esos subsistemas.

La arquitectura deberá avanzar hacia:

```text
Application
    ↓
EntityManager
    ├── MetadataRegistry
    ├── RepositoryRegistry
    ├── IdentityMap
    ├── UnitOfWork
    ├── EntityQuery
    └── Persistence Coordinator
            ↓
       Query Engine
            ↓
     Execution Engine
```

manteniendo una de las reglas centrales del ORM de VoltStack:

> **EntityManager coordina el ciclo de vida de persistencia de las entidades; no genera SQL, no implementa Query Builder, no reemplaza UnitOfWork y no debe convertirse en un Service Locator universal.**