# 97_DATABASE_SCHEMA_METADATA_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Metadata System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 97 — Database Schema Metadata System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Metadata System` define la arquitectura mediante la cual VoltStack representa, clasifica, transporta, normaliza, compara y gobierna la información adicional asociada a los objetos de esquema.

La metadata complementa estructuras como:

```text
DatabaseSchema
Table
Column
Index
Constraint
ForeignKey
Sequence
View
```

sin convertirse en una segunda representación paralela del esquema.

La separación fundamental será:

```text
Schema Structure
≠
Schema Metadata

Structural Identity
≠
Observation Metadata

Declared Metadata
≠
Observed Metadata

Semantic Metadata
≠
Informational Metadata

Physical Metadata
≠
Portable Semantics

Unknown Metadata
≠
Default Metadata
```

La metadata deberá responder preguntas como:

```text
¿De dónde proviene esta información?
¿Qué nivel de certeza tiene?
¿Fue declarada o introspectada?
¿Es portable?
¿Afecta la semántica?
¿Afecta el almacenamiento físico?
¿Debe participar en Schema Diff?
¿Está completa?
¿Pertenece a una plataforma concreta?
```

---

# 2. Principio central

> **La metadata describe contexto, evidencia, propiedades complementarias y características de los objetos del esquema; nunca debe convertirse en una segunda fuente de verdad para aquello que ya pertenece al modelo estructural.**

Por ejemplo:

```text
ColumnDefinition
├── type
├── nullable
└── default
```

no deberá duplicarse mediante:

```text
metadata["type"]
metadata["nullable"]
metadata["default"]
```

La regla será:

```text
Canonical Structure
=
single structural source of truth
```

mientras:

```text
Metadata
=
context + evidence + annotations + platform details
```

---

# 3. Posición arquitectónica

```text
Physical Database
       │
       ▼
Schema Introspection
       │
       ▼
Observed Metadata
       │
       ▼
Schema Metadata System
       │
       ├── Provenance
       ├── Certainty
       ├── Coverage
       ├── Platform Metadata
       └── Observation Metadata
       │
       ▼
Schema Snapshot
       │
       ├──────────────┐
       ▼              ▼
 Schema Diff      Diagnostics
       │
       ▼
 Migrations
```

También podrá recibir metadata declarada:

```text
Schema Builder
      │
      ▼
Definitions
      │
      ▼
Declared Metadata
```

---

# 4. Responsabilidades

El sistema será responsable de:

- definir categorías de metadata;
- definir metadata portable;
- representar metadata específica de plataforma;
- representar metadata declarada;
- representar metadata observada;
- preservar provenance;
- preservar certainty;
- preservar coverage;
- diferenciar metadata conocida y desconocida;
- controlar extensiones;
- normalizar metadata;
- comparar metadata;
- fingerprinting;
- serialización;
- propagación controlada;
- filtrado;
- seguridad;
- redacción de datos sensibles;
- diagnostics;
- integración con Schema Diff;
- compatibilidad con persistent runtimes.

---

# 5. No responsabilidades

No será responsable de:

- representar columnas como metadata;
- representar índices únicamente como metadata;
- representar foreign keys únicamente como metadata;
- ejecutar introspection;
- generar DDL;
- ejecutar DDL;
- decidir migraciones;
- resolver conexiones;
- ejecutar queries;
- gestionar ORM entities;
- decidir autorización;
- convertir metadata desconocida en valores predeterminados.

---

# 6. Ecuación conceptual

La representación de un objeto podrá expresarse como:

```text
SchemaObject
=
StructuralDefinition
+
StructuralIdentity
+
SchemaMetadata
```

Para objetos observados:

```text
ObservedSchemaObject
=
StructuralDefinition
+
StructuralIdentity
+
SchemaMetadata
+
ObservationContext
```

Sin embargo, conceptualmente:

```text
ObservationContext
```

puede formar parte del metadata envelope.

---

# 7. Clasificación principal

VoltStack deberá clasificar metadata como mínimo en:

```text
SchemaMetadata
├── Structural
├── Semantic
├── Physical
├── Informational
├── Observational
├── Operational
├── Platform
├── Extension
└── Unknown
```

Estas categorías podrán solaparse mediante traits/flags controlados, pero deberá existir una clasificación canónica.

---

# 8. Structural metadata

`Structural Metadata` representa información complementaria que afecta la representación estructural pero que no justifica crear otro objeto estructural independiente.

Ejemplos potenciales:

```text
declared column ordinal
object naming origin
logical generation name
structural annotations
```

Debe utilizarse con extrema precaución.

Si una propiedad es fundamental para determinar la estructura:

```text
column type
nullability
foreign key target
index keys
```

deberá vivir en el modelo estructural.

---

# 9. Semantic metadata

Afecta el significado observable del schema.

Ejemplos:

```text
collation semantics
constraint enforcement status
deferrability semantics
generated expression semantics
validation semantics
```

cuando dichas propiedades no estén ya modeladas directamente.

Regla:

```text
Semantic Metadata
may participate in semantic equality
```

---

# 10. Physical metadata

Describe características físicas de almacenamiento o implementación.

Ejemplos:

```text
storage engine
tablespace
row format
physical index implementation
page configuration
fill factor
storage parameters
```

Puede ser relevante para:

```text
operations
performance
platform-specific diff
```

sin necesariamente alterar la semántica portable.

---

# 11. Informational metadata

Ejemplos:

```text
comments
descriptions
developer annotations
documentation labels
source locations
display information
```

Normalmente:

```text
Informational Metadata
∉
Semantic Equality
```

aunque una política de Schema Diff puede decidir sincronizar comentarios.

---

# 12. Observational metadata

Proviene de introspection.

Puede contener:

```text
provenance
certainty
visibility
coverage
observedAt
snapshot ID
native representation
decoder information
normalization information
```

---

# 13. Operational metadata

Representa información útil para tooling u operaciones pero que no forma parte del schema semántico.

Ejemplos:

```text
migration origin
tooling hints
inspection diagnostics
synchronization state
management annotations
```

No deberá alterar automáticamente SQL ni semántica.

---

# 14. Platform metadata

Representa información específica de:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

o futuras plataformas.

Ejemplo:

```text
MySqlTableEngineMetadata
PostgreSqlStorageParameterMetadata
SqliteTableMetadata
```

---

# 15. Extension metadata

Paquetes externos podrán añadir metadata tipada mediante:

```text
SchemaMetadataExtension
```

sin modificar las clases core.

---

# 16. Unknown metadata

Cuando VoltStack reconoce que existe metadata pero no comprende su semántica:

```text
UnknownMetadata
```

o:

```text
OpaqueMetadata
```

deberá preservarse cuando la política lo permita.

Nunca:

```text
unknown
    ↓
discard silently
```

por defecto.

---

# 17. Metadata ≠ Property Bag

No se recomienda:

```php
$metadata = [
    'foo' => 'bar',
    'engine' => 'InnoDB',
    'whatever' => 123,
];
```

como representación primaria.

Preferido:

```php
$metadata = SchemaMetadataSet::of(
    new TableCommentMetadata('Customer accounts'),
    new MySqlTableEngineMetadata('InnoDB'),
);
```

---

# 18. Tipado

Contrato conceptual:

```php
interface SchemaMetadata
{
    public function key(): SchemaMetadataKey;

    public function category(): SchemaMetadataCategory;

    public function origin(): SchemaMetadataOrigin;
}
```

---

# 19. Metadata key

Se propone:

```php
final readonly class SchemaMetadataKey
{
    public function __construct(
        public string $namespace,
        public string $name,
        public int $version,
    ) {}
}
```

Ejemplo conceptual:

```text
voltstack.schema.comment@1
voltstack.mysql.table.engine@1
vendor.extension.foo@2
```

---

# 20. Namespace

La metadata deberá utilizar namespaces para evitar colisiones:

```text
voltstack.*
mysql.*
mariadb.*
postgresql.*
sqlite.*
extension.vendor.*
```

---

# 21. Metadata identity

Debe distinguirse:

```text
MetadataKey
≠
MetadataInstanceId
≠
SchemaObjectId
```

Normalmente no será necesario un ID mutable por instancia.

---

# 22. SchemaMetadataSet

Se propone una colección inmutable:

```php
final readonly class SchemaMetadataSet
{
    public function has(SchemaMetadataKey $key): bool;

    public function get(SchemaMetadataKey $key): ?SchemaMetadata;

    public function require(SchemaMetadataKey $key): SchemaMetadata;

    public function all(): iterable;
}
```

---

# 23. Inmutabilidad

Después de construido:

```text
SchemaMetadataSet
```

será inmutable.

Transformaciones:

```text
with()
without()
replace()
```

devolverán nuevas instancias.

---

# 24. Duplicados

No deberá existir:

```text
same metadata key
+
two canonical values
```

salvo que el tipo declare explícitamente multiplicidad.

---

# 25. Multiplicity

Se propone:

```text
SchemaMetadataMultiplicity
├── SINGLE
└── MULTIPLE
```

---

# 26. Metadata descriptor

Cada tipo registrado podrá declarar:

```php
final readonly class SchemaMetadataDescriptor
{
    public function __construct(
        public SchemaMetadataKey $key,
        public SchemaMetadataCategory $category,
        public SchemaMetadataMultiplicity $multiplicity,
        public SchemaMetadataPortability $portability,
        public SchemaMetadataComparisonPolicy $comparison,
    ) {}
}
```

---

# 27. Metadata origin

Debe distinguirse:

```text
SchemaMetadataOrigin
├── DECLARED
├── OBSERVED
├── INFERRED
├── GENERATED
├── NORMALIZED
├── EXTENSION
└── UNKNOWN
```

---

# 28. Declared metadata

Ejemplo:

```php
$table->comment('Customer accounts');
```

podrá generar:

```text
TableCommentMetadata
origin = DECLARED
```

---

# 29. Observed metadata

Introspection puede producir:

```text
TableCommentMetadata
origin = OBSERVED
```

con provenance adicional.

---

# 30. Declared ≠ observed

Dos valores idénticos:

```text
comment = "users"
```

pueden tener diferente origen.

La igualdad semántica podrá ignorar el origen.

La igualdad exacta podrá considerarlo.

---

# 31. Provenance

Se reutilizará el concepto del documento 96:

```text
ObservationProvenance
```

Ejemplos:

```text
INFORMATION_SCHEMA
SYSTEM_CATALOG
PRAGMA
DRIVER_METADATA
NATIVE_QUERY
PLATFORM_INFERENCE
EXTENSION
UNKNOWN
```

---

# 32. Origin ≠ provenance

Debe distinguirse:

```text
origin = OBSERVED
```

de:

```text
provenance = PG_CATALOG
```

Origin responde:

```text
¿Cómo entró esta metadata al modelo?
```

Provenance responde:

```text
¿De qué evidencia concreta provino?
```

---

# 33. Certainty

Metadata observada podrá asociarse a:

```text
ObservationCertainty
├── EXACT
├── NORMALIZED
├── INFERRED
├── APPROXIMATE
├── UNKNOWN
└── UNAVAILABLE
```

---

# 34. Metadata envelope

Se propone separar valor de contexto:

```php
final readonly class SchemaMetadataEntry
{
    public function __construct(
        public SchemaMetadata $value,
        public SchemaMetadataContext $context,
    ) {}
}
```

---

# 35. SchemaMetadataContext

Conceptualmente:

```php
final readonly class SchemaMetadataContext
{
    public function __construct(
        public SchemaMetadataOrigin $origin,
        public ?ObservationProvenance $provenance,
        public ?ObservationCertainty $certainty,
        public SchemaMetadataVisibility $visibility,
        public SchemaMetadataTrust $trust,
    ) {}
}
```

---

# 36. Value ≠ envelope

Esto permite:

```text
TableCommentMetadata("Accounts")
```

ser igual estructuralmente aunque:

```text
Entry A = DECLARED
Entry B = OBSERVED
```

---

# 37. Coverage

Coverage no deberá almacenarse como un simple booleano:

```php
$metadata['complete'] = true;
```

Se reutilizará:

```text
SchemaCoverageDescriptor
```

del sistema de introspección.

---

# 38. Object-level coverage

Podrá existir:

```text
SchemaObjectMetadataCoverage
```

para indicar:

```text
columns metadata complete
indexes metadata partial
comments unavailable
platform options complete
```

---

# 39. Granular coverage

Ejemplo:

```text
Table users
├── Columns ........ COMPLETE
├── Indexes ........ COMPLETE
├── Constraints .... COMPLETE
├── Comments ....... UNKNOWN
└── Storage ........ PARTIAL
```

---

# 40. Metadata presence

Se deberá distinguir:

```text
PRESENT
ABSENT
UNKNOWN
NOT_VISIBLE
NOT_REQUESTED
UNSUPPORTED
```

cuando la ausencia tenga significado.

---

# 41. No value ≠ default value

Ejemplo:

```text
collation metadata unavailable
```

no deberá convertirse automáticamente en:

```text
database default collation
```

---

# 42. Explicit inheritance

Cuando una propiedad hereda de un contexto superior deberá representarse explícitamente:

```text
MetadataValueState
├── EXPLICIT
├── INHERITED
├── PLATFORM_DEFAULT
├── ABSENT
└── UNKNOWN
```

---

# 43. Platform default

`PLATFORM_DEFAULT` solo deberá utilizarse cuando exista evidencia suficiente.

No como fallback genérico.

---

# 44. Inherited metadata

Ejemplo:

```text
Column collation
    ↓
INHERITED_FROM_TABLE
```

o:

```text
INHERITED_FROM_DATABASE
```

cuando pueda demostrarse.

---

# 45. Effective metadata

Puede existir:

```text
Declared Metadata
+
Inheritance
+
Platform Defaults
        ↓
Effective Metadata
```

pero deberá conservarse la distinción entre:

```text
declared
```

y:

```text
effective
```

---

# 46. EffectiveMetadataResolver

Se propone:

```php
interface EffectiveSchemaMetadataResolver
{
    public function resolve(
        SchemaObject $object,
        SchemaMetadataResolutionContext $context,
    ): EffectiveSchemaMetadataSet;
}
```

---

# 47. Resolver purity

El resolver no deberá ejecutar hidden database I/O.

Recibirá explícitamente:

```text
SchemaSnapshot
PlatformCapabilitySnapshot
Parent Metadata
Resolution Policy
```

---

# 48. Metadata hierarchy

Podrá existir herencia:

```text
Database
   ↓
Namespace
   ↓
Table
   ↓
Column
```

para propiedades compatibles.

No toda metadata será heredable.

---

# 49. Inheritance descriptor

Cada metadata podrá declarar:

```text
NONE
DATABASE_TO_NAMESPACE
NAMESPACE_TO_TABLE
TABLE_TO_COLUMN
CUSTOM
```

---

# 50. No universal inheritance

No deberá asumirse:

```text
every metadata item
inherits
```

---

# 51. Portability

Se propone:

```text
SchemaMetadataPortability
├── PORTABLE
├── PORTABLE_WITH_LOSS
├── PLATFORM_SCOPED
├── EXTENSION_SCOPED
└── UNKNOWN
```

---

# 52. Portable metadata

Ejemplo potencial:

```text
comment
```

si el modelo portable lo soporta.

---

# 53. Platform-scoped metadata

Ejemplo:

```text
MySqlTableEngineMetadata("InnoDB")
```

---

# 54. Portable-with-loss

Puede representar una propiedad que tenga equivalentes parciales.

Ejemplo conceptual:

```text
index storage tuning
```

---

# 55. Unknown portability

Nunca deberá tratarse automáticamente como portable.

---

# 56. Semantic impact

Cada descriptor podrá declarar:

```text
SchemaMetadataSemanticImpact
├── NONE
├── INFORMATIONAL
├── STRUCTURAL
├── SEMANTIC
├── PHYSICAL
└── UNKNOWN
```

---

# 57. Semantic impact ≠ diff policy

Que metadata sea:

```text
INFORMATIONAL
```

no significa que Schema Diff deba ignorarla siempre.

Puede existir un perfil:

```text
SYNC_COMMENTS
```

---

# 58. Comparison policy

Se propone:

```text
SchemaMetadataComparisonPolicy
├── IGNORE
├── EXACT
├── STRUCTURAL
├── SEMANTIC
├── PLATFORM_NORMALIZED
└── CUSTOM
```

---

# 59. Equality modes

La metadata participará en los modos generales:

```text
Exact Equality
Structural Equality
Semantic Equality
Platform-Normalized Equality
```

---

# 60. Exact equality

Puede considerar:

```text
value
type
version
origin
platform representation
```

según descriptor.

---

# 61. Structural equality

Considerará metadata que afecte estructura.

Normalmente ignorará:

```text
observedAt
diagnostic IDs
source locations
```

---

# 62. Semantic equality

Considerará únicamente aquello que afecte comportamiento semántico.

---

# 63. Platform-normalized equality

Permitirá equivalencias como:

```text
native representation A
≈
native representation B
```

cuando la plataforma las considere equivalentes.

---

# 64. Comparison context

Se propone:

```php
final readonly class SchemaMetadataComparisonContext
{
    public function __construct(
        public SchemaComparisonMode $mode,
        public DatabasePlatform $platform,
        public PlatformCapabilitySnapshot $capabilities,
        public SchemaMetadataComparisonProfile $profile,
    ) {}
}
```

---

# 65. Diff participation

Cada tipo deberá declarar cómo puede producir cambios.

Ejemplo:

```text
TableCommentMetadata
    ↓
COMMENT CHANGE

MySqlTableEngineMetadata
    ↓
PLATFORM TABLE OPTION CHANGE
```

---

# 66. Metadata diff

No deberá utilizarse:

```text
array_diff()
```

Se requiere:

```text
SchemaMetadataDiffer
```

---

# 67. Metadata diff result

Conceptualmente:

```text
SchemaMetadataDiff
├── Added
├── Removed
├── Changed
├── Unknown
├── Incomparable
└── PlatformDependent
```

---

# 68. Unknown metadata and diff

Debe cumplirse:

```text
Current = UNKNOWN
Target = X
```

no implica automáticamente:

```text
CHANGE UNKNOWN → X
```

La política de seguridad deberá intervenir.

---

# 69. Incomplete metadata

Si:

```text
current metadata coverage = PARTIAL
```

una metadata no observada no será considerada automáticamente eliminada.

---

# 70. Metadata normalization

Se propone:

```text
Raw Metadata
    │
    ▼
Decode
    │
    ▼
Typed Metadata
    │
    ▼
Canonicalize
    │
    ▼
Normalized Metadata
```

---

# 71. Normalization properties

Debe ser:

```text
deterministic
idempotent
side-effect free
platform-aware
versioned
```

---

# 72. Normalization invariant

Idealmente:

```text
Normalize(Normalize(M))
=
Normalize(M)
```

---

# 73. Normalization ≠ information destruction

No deberá convertir:

```text
platform-specific metadata
```

en:

```text
portable metadata
```

si se pierde información crítica sin registrarlo.

---

# 74. Loss reporting

Si una normalización pierde información deliberadamente:

```text
NormalizationLossDiagnostic
```

deberá poder producirse.

---

# 75. Canonical ordering

`SchemaMetadataSet` deberá serializarse en orden determinista.

Por ejemplo:

```text
namespace
name
version
```

---

# 76. Runtime insertion order

No deberá afectar:

```text
fingerprint
serialization
equality
```

salvo metadata donde el orden sea explícitamente semántico.

---

# 77. Fingerprinting

Se propone:

```text
SchemaMetadataFingerprint
```

---

# 78. Fingerprint modes

```text
EXACT
STRUCTURAL
SEMANTIC
PLATFORM_NORMALIZED
```

---

# 79. Excluded metadata

Un fingerprint estructural normalmente excluirá:

```text
observedAt
query duration
source location
diagnostic identifiers
temporary runtime IDs
```

---

# 80. Fingerprint versioning

Debe incluir:

```text
algorithm version
normalization version
metadata schema version
comparison profile
```

cuando corresponda.

---

# 81. Hash ≠ equality proof

Debe cumplirse:

```text
same hash
≠
absolute proof of equality
```

El hash será acelerador.

---

# 82. Serialization

La metadata deberá utilizar formato:

```text
versioned
deterministic
portable where possible
extension-aware
```

---

# 83. No PHP serialize contract

No utilizar:

```php
serialize($metadata);
```

como contrato persistente público.

---

# 84. Serialized envelope

Ejemplo conceptual:

```json
{
  "key": "mysql.table.engine",
  "version": 1,
  "value": {
    "engine": "InnoDB"
  },
  "context": {
    "origin": "OBSERVED",
    "certainty": "EXACT"
  }
}
```

---

# 85. Unknown metadata serialization

Cuando una extensión no esté instalada:

```text
OpaqueSerializedSchemaMetadata
```

podrá preservar:

```text
key
version
payload
origin
```

según policy.

---

# 86. Unknown metadata policy

Se propone:

```text
FAIL
PRESERVE_OPAQUE
IGNORE_WITH_DIAGNOSTIC
```

---

# 87. Default policy

Para snapshots/import/export se recomienda:

```text
PRESERVE_OPAQUE
```

cuando sea seguro.

Para operaciones que requieren interpretación semántica:

```text
FAIL
```

puede ser más apropiado.

---

# 88. Metadata registry

Se propone:

```text
SchemaMetadataRegistry
```

con descriptors tipados.

---

# 89. Frozen registry

Después de bootstrap:

```text
MutableRegistry
    ↓
freeze()
    ↓
FrozenSchemaMetadataRegistry
```

---

# 90. Duplicate registration

Nunca:

```text
last registration wins
```

silenciosamente.

Debe fallar.

---

# 91. Version registration

Podrá existir:

```text
metadata.foo@1
metadata.foo@2
```

si el sistema de versionado lo permite.

---

# 92. Metadata migration

Para metadata serializada antigua podrá existir:

```text
SchemaMetadataUpcaster
```

---

# 93. Upcaster

Ejemplo:

```text
Metadata v1
    ↓
Upcaster
    ↓
Metadata v2
```

No deberá consultar la base de datos.

---

# 94. Extension registration

Una extensión podrá registrar:

```text
descriptor
decoder
serializer
comparator
normalizer
upcaster
```

pero no podrá reemplazar metadata core arbitrariamente.

---

# 95. Trust model

Se propone:

```text
SchemaMetadataTrust
├── FRAMEWORK
├── APPLICATION
├── PLATFORM
├── EXTENSION
├── EXTERNAL
└── UNTRUSTED
```

---

# 96. Trust ≠ correctness

```text
FRAMEWORK
```

no significa que el dato sea necesariamente:

```text
EXACT
```

Trust y certainty son dimensiones diferentes.

---

# 97. Security classification

Metadata podrá marcarse:

```text
PUBLIC
INTERNAL
SENSITIVE
SECRET_PROHIBITED
```

---

# 98. Secret prohibition

El Schema Metadata System no deberá almacenar:

```text
database passwords
API secrets
access tokens
private keys
```

---

# 99. Sensitive schema metadata

Algunos elementos pueden ser sensibles:

```text
internal table names
security-related column names
view definitions
comments
extension configuration
```

y deberán poder redactarse.

---

# 100. Redaction

Se propone:

```text
SchemaMetadataRedactor
```

con perfiles:

```text
DEVELOPER
LOGGING
TELEMETRY
EXCEPTION
EXPORT
DEBUG
```

---

# 101. Redaction ≠ mutation

La redacción deberá producir:

```text
MetadataView
```

o copia transformada.

No modificar el metadata original.

---

# 102. Diagnostics

Metadata inválida deberá generar diagnostics estructurados.

Ejemplo:

```text
UNKNOWN_METADATA_VERSION
INVALID_PLATFORM_METADATA
METADATA_NORMALIZATION_LOSS
INCOMPLETE_METADATA
UNSUPPORTED_METADATA
AMBIGUOUS_METADATA
```

---

# 103. Diagnostic severity

```text
INFO
WARNING
ERROR
CRITICAL
```

---

# 104. Diagnostics ≠ metadata truth

Un diagnostic asociado no deberá modificar implícitamente el valor.

---

# 105. Source location

Metadata declarada puede conservar:

```text
file
line
column
builder operation
```

para developer experience.

---

# 106. Source location semantic rule

```text
SourceLocation
∉
StructuralEquality
```

---

# 107. Comments

Se propone:

```php
final readonly class SchemaCommentMetadata implements SchemaMetadata
{
    public function __construct(
        public string $comment,
    ) {}
}
```

---

# 108. Comment policy

Los comentarios podrán:

- serializarse;
- introspectarse;
- compararse;
- sincronizarse opcionalmente;
- excluirse de semantic fingerprint.

---

# 109. Naming origin metadata

Puede existir:

```text
SchemaObjectNamingOrigin
├── EXPLICIT
├── GENERATED
├── PLATFORM_GENERATED
├── INTROSPECTED
└── UNKNOWN
```

---

# 110. Logical name ≠ physical name

Ejemplo:

```text
Logical constraint name
    ↓
Compiler
    ↓
Physical truncated/hashed name
```

La metadata podrá preservar la relación cuando sea necesario.

---

# 111. Physical naming metadata

Se propone:

```text
PhysicalObjectNameMetadata
```

solo cuando no pueda representarse adecuadamente mediante el identificador canónico.

No deberá duplicar nombres sin necesidad.

---

# 112. Platform metadata architecture

```text
SchemaMetadata
     │
     └── PlatformSchemaMetadata
              │
              ├── MySqlSchemaMetadata
              ├── MariaDbSchemaMetadata
              ├── PostgreSqlSchemaMetadata
              └── SqliteSchemaMetadata
```

---

# 113. Platform identity

Toda metadata platform-scoped deberá declarar:

```text
DatabasePlatformId
```

o capability domain equivalente.

---

# 114. No vendor conditionals

Core consumers no deberán hacer:

```php
if ($metadata->platform() === 'mysql') {
    // ...
}
```

de forma dispersa.

Se utilizarán:

```text
typed handlers
capabilities
metadata descriptors
platform services
```

---

# 115. MySQL metadata

Podrá incluir tipos como:

```text
MySqlTableEngineMetadata
MySqlRowFormatMetadata
MySqlCharsetMetadata
MySqlCollationMetadata
MySqlIndexVisibilityMetadata
```

cuando no estén representados mejor por modelos portables.

---

# 116. MariaDB metadata

MariaDB tendrá metadata propia cuando la semántica diverja.

No será automáticamente:

```text
MySQL metadata
```

---

# 117. PostgreSQL metadata

Ejemplos:

```text
PostgreSqlTablespaceMetadata
PostgreSqlStorageParametersMetadata
PostgreSqlReplicaIdentityMetadata
```

según evolución futura.

---

# 118. SQLite metadata

Podrá representar propiedades como:

```text
WITHOUT ROWID
STRICT
```

mediante metadata/extensiones tipadas si no forman parte del modelo portable.

---

# 119. Capability requirements

Metadata podrá declarar:

```text
required capabilities
```

Ejemplo:

```text
PostgreSqlStorageParameterMetadata
    ↓
requires relevant platform capability
```

---

# 120. Capability validation

Se propone:

```text
SchemaMetadataCapabilityValidator
```

---

# 121. Validation layers

```text
Metadata Local Validation
        ↓
Metadata Set Validation
        ↓
Object Compatibility Validation
        ↓
Platform Capability Validation
        ↓
Cross-Object Validation
```

---

# 122. Local validation

Ejemplos:

```text
invalid metadata key
invalid version
empty required value
malformed extension payload
```

---

# 123. Set validation

Ejemplos:

```text
duplicate singleton metadata
conflicting metadata
incompatible metadata categories
```

---

# 124. Object compatibility

Ejemplo:

```text
TableEngineMetadata
```

no podrá adjuntarse a:

```text
Column
```

si su descriptor solo permite tablas.

---

# 125. Target kinds

Cada descriptor podrá declarar:

```text
DATABASE
NAMESPACE
TABLE
COLUMN
INDEX
CONSTRAINT
FOREIGN_KEY
SEQUENCE
VIEW
```

---

# 126. Metadata target descriptor

```php
final readonly class SchemaMetadataTargetDescriptor
{
    public function __construct(
        public SchemaObjectKindSet $allowedKinds,
    ) {}
}
```

---

# 127. Cross-object metadata

Debe evitarse metadata que cree referencias estructurales ocultas.

Ejemplo incorrecto:

```text
metadata["foreign_table"] = "users"
```

La FK pertenece al modelo estructural.

---

# 128. Derived metadata

Algunas metadata podrán calcularse:

```text
SchemaObject
    ↓
MetadataDeriver
    ↓
DerivedMetadata
```

---

# 129. Derived metadata examples

```text
estimated portability
capability requirements
management classification
normalized display information
```

---

# 130. Derived ≠ stored truth

Si puede derivarse deterministicamente de estructura:

```text
do not persist redundantly
```

salvo por caching explícito.

---

# 131. Metadata projections

Consumidores no deberán recibir necesariamente toda metadata.

Podrán solicitar:

```text
SchemaMetadataProjection
```

---

# 132. Projection examples

```text
SEMANTIC_ONLY
PORTABLE_ONLY
DIFF_RELEVANT
PLATFORM_ONLY
OBSERVATION_ONLY
TELEMETRY_SAFE
```

---

# 133. Schema Diff projection

`Schema Diff` podrá pedir:

```text
DIFF_RELEVANT
```

según un `SchemaDiffProfile`.

---

# 134. Query Semantic projection

El Semantic Engine podrá recibir:

```text
SEMANTIC_ONLY
```

sin observation timestamps ni tooling annotations.

---

# 135. Telemetry projection

Telemetry recibirá:

```text
TELEMETRY_SAFE
```

con redacción aplicada.

---

# 136. Metadata views

Se propone:

```text
SchemaMetadataView
```

como proyección inmutable.

---

# 137. No copy explosion requirement

Las views podrán implementarse eficientemente mediante:

```text
immutable references
+
filters
```

siempre que no expongan mutabilidad.

---

# 138. Metadata merge

Será necesario para:

```text
declared metadata
+
inherited metadata
+
observed metadata
```

pero no deberá utilizarse un merge genérico.

---

# 139. SchemaMetadataMerger

Se propone:

```php
interface SchemaMetadataMerger
{
    public function merge(
        SchemaMetadataSet $base,
        SchemaMetadataSet $overlay,
        SchemaMetadataMergePolicy $policy,
    ): SchemaMetadataMergeResult;
}
```

---

# 140. Merge policies

```text
STRICT
DECLARED_WINS
OBSERVED_WINS
PRESERVE_CONFLICT
CUSTOM
```

No deberá existir un `last wins` implícito.

---

# 141. Conflict

Ejemplo:

```text
Declared:
engine = InnoDB

Observed:
engine = MyISAM
```

Eso es:

```text
metadata conflict
```

o posteriormente:

```text
schema drift
```

No debe resolverse silenciosamente.

---

# 142. Metadata conflict object

```text
SchemaMetadataConflict
├── key
├── left
├── right
├── classification
└── diagnostic
```

---

# 143. Provenance preservation during merge

Si dos fuentes coinciden:

```text
Declared = X
Observed = X
```

podrá conservarse:

```text
multiple evidence records
```

sin duplicar el valor canónico.

---

# 144. Evidence model

Para casos avanzados se propone:

```text
SchemaMetadataEvidenceSet
```

---

# 145. Evidence

```php
final readonly class SchemaMetadataEvidence
{
    public function __construct(
        public SchemaMetadataOrigin $origin,
        public ?ObservationProvenance $provenance,
        public ?ObservationCertainty $certainty,
    ) {}
}
```

---

# 146. Multiple evidence

Esto permite:

```text
Value X
├── declared by application
└── observed from system catalog
```

sin crear dos valores conflictivos.

---

# 147. Metadata lifecycle

```text
Created
   ↓
Validated
   ↓
Normalized
   ↓
Attached
   ↓
Sealed
   ↓
Projected / Compared / Serialized
```

---

# 148. Metadata builder

Podrá existir internamente:

```text
SchemaMetadataSetBuilder
```

mutable y operation-scoped.

Pero el resultado será:

```text
SchemaMetadataSet
```

inmutable.

---

# 149. Builder ≠ metadata set

```text
SchemaMetadataSetBuilder
≠
SchemaMetadataSet
```

---

# 150. Persistent runtime

Shared:

```text
Frozen Metadata Registry
Descriptors
Normalizers
Serializers
Comparators
```

Operation-scoped:

```text
MetadataSetBuilder
MergeSession
NormalizationSession
Diagnostics
```

---

# 151. FrankenPHP safety

Nunca conservar:

```text
request metadata
tenant metadata
snapshot metadata
```

en servicios singleton mutables.

---

# 152. Tenant metadata

Multitenancy podrá adjuntar metadata de integración.

Pero el core no deberá introducir:

```text
currentTenant
```

en `SchemaMetadataSet`.

---

# 153. Tenant identifiers

Si son necesarios para cache/snapshot scope:

```text
TenantContext
```

deberá vivir en el contexto operativo correspondiente, no mezclarse indiscriminadamente con estructura portable.

---

# 154. Cache integration

Metadata inmutable y normalizada podrá cachearse.

Pero:

```text
Metadata System
```

no dependerá obligatoriamente del Cache System.

---

# 155. Cache identity

Si se cachea metadata observada deberá considerar:

```text
schema snapshot
platform
capabilities
normalization version
metadata registry version
```

---

# 156. Performance

Objetivos:

```text
metadata lookup     ≈ O(1)
metadata iteration  ≈ O(n)
normalization       ≈ O(n)
fingerprinting      ≈ O(n)
serialization       ≈ O(n)
```

para `n` entries.

---

# 157. Metadata key index

`SchemaMetadataSet` deberá usar internamente una estructura indexada por:

```text
SchemaMetadataKey
```

---

# 158. Deterministic iteration

La estructura interna podrá ser hash-based.

La exposición/serialización deberá tener orden determinista.

---

# 159. Lazy fingerprints

Podrán memoizarse si:

```text
SchemaMetadataSet
```

es realmente inmutable.

---

# 160. Budgets

Se propone:

```php
final readonly class SchemaMetadataBudget
{
    public function __construct(
        public int $maxEntriesPerObject,
        public int $maxTotalEntries,
        public int $maxSerializedBytes,
        public int $maxOpaquePayloadBytes,
        public int $maxEvidenceEntries,
        public int $maxExtensionEntries,
    ) {}
}
```

---

# 161. Budget failure

Nunca truncar silenciosamente.

Debe producir:

```text
SchemaMetadataBudgetExceededException
```

o diagnóstico explícito según policy.

---

# 162. Error hierarchy

```text
DatabaseSchemaMetadataException
├── InvalidSchemaMetadataException
├── InvalidSchemaMetadataKeyException
├── UnsupportedSchemaMetadataVersionException
├── DuplicateSchemaMetadataException
├── ConflictingSchemaMetadataException
├── InvalidSchemaMetadataTargetException
├── SchemaMetadataNormalizationException
├── SchemaMetadataComparisonException
├── SchemaMetadataMergeException
├── SchemaMetadataSerializationException
├── SchemaMetadataDeserializationException
├── SchemaMetadataCapabilityException
├── SchemaMetadataExtensionException
├── SchemaMetadataSecurityException
├── SchemaMetadataBudgetExceededException
└── SchemaMetadataInvariantException
```

---

# 163. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Metadata
```

---

# 164. Proposed directory structure

```text
Schema/
└── Metadata/
    ├── Contract/
    │   ├── SchemaMetadata.php
    │   ├── SchemaMetadataNormalizer.php
    │   ├── SchemaMetadataComparator.php
    │   ├── SchemaMetadataSerializer.php
    │   ├── SchemaMetadataMerger.php
    │   └── EffectiveSchemaMetadataResolver.php
    │
    ├── Core/
    │   ├── SchemaMetadataKey.php
    │   ├── SchemaMetadataEntry.php
    │   ├── SchemaMetadataSet.php
    │   ├── SchemaMetadataContext.php
    │   └── SchemaMetadataDescriptor.php
    │
    ├── Category/
    │   ├── SchemaMetadataCategory.php
    │   ├── StructuralMetadata.php
    │   ├── SemanticMetadata.php
    │   ├── PhysicalMetadata.php
    │   ├── InformationalMetadata.php
    │   ├── ObservationalMetadata.php
    │   ├── OperationalMetadata.php
    │   └── UnknownMetadata.php
    │
    ├── Origin/
    │   ├── SchemaMetadataOrigin.php
    │   ├── SchemaMetadataEvidence.php
    │   └── SchemaMetadataEvidenceSet.php
    │
    ├── Coverage/
    │   ├── SchemaObjectMetadataCoverage.php
    │   ├── MetadataValueState.php
    │   └── SchemaMetadataVisibility.php
    │
    ├── Portability/
    │   ├── SchemaMetadataPortability.php
    │   └── SchemaMetadataSemanticImpact.php
    │
    ├── Comparison/
    │   ├── SchemaMetadataComparisonPolicy.php
    │   ├── SchemaMetadataComparisonContext.php
    │   ├── SchemaMetadataComparisonProfile.php
    │   └── SchemaMetadataFingerprint.php
    │
    ├── Diff/
    │   ├── SchemaMetadataDiffer.php
    │   ├── SchemaMetadataDiff.php
    │   └── SchemaMetadataChange.php
    │
    ├── Merge/
    │   ├── SchemaMetadataMerger.php
    │   ├── SchemaMetadataMergePolicy.php
    │   ├── SchemaMetadataMergeResult.php
    │   └── SchemaMetadataConflict.php
    │
    ├── Resolution/
    │   ├── EffectiveSchemaMetadataResolver.php
    │   ├── SchemaMetadataResolutionContext.php
    │   └── SchemaMetadataInheritanceDescriptor.php
    │
    ├── Projection/
    │   ├── SchemaMetadataProjection.php
    │   └── SchemaMetadataView.php
    │
    ├── Registry/
    │   ├── SchemaMetadataRegistry.php
    │   └── FrozenSchemaMetadataRegistry.php
    │
    ├── Normalization/
    │   ├── SchemaMetadataNormalizer.php
    │   └── SchemaMetadataNormalizationResult.php
    │
    ├── Serialization/
    │   ├── SchemaMetadataSerializer.php
    │   ├── SchemaMetadataDeserializer.php
    │   ├── SchemaMetadataUpcaster.php
    │   └── OpaqueSerializedSchemaMetadata.php
    │
    ├── Security/
    │   ├── SchemaMetadataTrust.php
    │   ├── SchemaMetadataSecurityClassification.php
    │   ├── SchemaMetadataRedactor.php
    │   └── SchemaMetadataRedactionProfile.php
    │
    ├── Platform/
    │   ├── PlatformSchemaMetadata.php
    │   ├── MySQL/
    │   ├── MariaDB/
    │   ├── PostgreSQL/
    │   └── SQLite/
    │
    ├── Extension/
    │   ├── SchemaMetadataExtension.php
    │   └── ExtensionSchemaMetadata.php
    │
    ├── Budget/
    │   └── SchemaMetadataBudget.php
    │
    ├── Diagnostic/
    │   ├── SchemaMetadataDiagnostic.php
    │   └── SchemaMetadataDiagnosticSet.php
    │
    └── Exception/
        ├── DatabaseSchemaMetadataException.php
        ├── InvalidSchemaMetadataException.php
        ├── InvalidSchemaMetadataKeyException.php
        ├── UnsupportedSchemaMetadataVersionException.php
        ├── DuplicateSchemaMetadataException.php
        ├── ConflictingSchemaMetadataException.php
        ├── InvalidSchemaMetadataTargetException.php
        ├── SchemaMetadataNormalizationException.php
        ├── SchemaMetadataComparisonException.php
        ├── SchemaMetadataMergeException.php
        ├── SchemaMetadataSerializationException.php
        ├── SchemaMetadataDeserializationException.php
        ├── SchemaMetadataCapabilityException.php
        ├── SchemaMetadataExtensionException.php
        ├── SchemaMetadataSecurityException.php
        ├── SchemaMetadataBudgetExceededException.php
        └── SchemaMetadataInvariantException.php
```

---

# 165. Ejemplo de metadata de tabla

```php
$metadata = SchemaMetadataSet::of(
    SchemaMetadataEntry::declared(
        new SchemaCommentMetadata('Registered application users'),
    ),

    SchemaMetadataEntry::observed(
        value: new MySqlTableEngineMetadata('InnoDB'),
        provenance: ObservationProvenance::SYSTEM_CATALOG,
        certainty: ObservationCertainty::EXACT,
    ),
);
```

Conceptualmente:

```text
users
│
├── Structure
│   ├── columns
│   ├── constraints
│   └── indexes
│
└── Metadata
    ├── Comment
    │   └── DECLARED
    │
    └── MySQL Engine
        ├── InnoDB
        ├── OBSERVED
        ├── SYSTEM_CATALOG
        └── EXACT
```

---

# 166. Ejemplo de metadata heredada

```text
Database
└── collation = utf8mb4_unicode_ci
        │
        ▼
Table users
└── collation = INHERITED
        │
        ▼
Column email
└── collation = INHERITED
```

El modelo deberá poder distinguir:

```text
Declared value:
NONE

Effective value:
utf8mb4_unicode_ci

Value state:
INHERITED
```

---

# 167. Ejemplo de conflicto

Target:

```text
MySqlTableEngine = InnoDB
origin = DECLARED
```

Observed:

```text
MySqlTableEngine = MyISAM
origin = OBSERVED
certainty = EXACT
```

Resultado:

```text
SchemaMetadataConflict
├── key = mysql.table.engine
├── declared = InnoDB
├── observed = MyISAM
└── classification = VALUE_MISMATCH
```

Esto podrá convertirse posteriormente en:

```text
Schema Diff
    ↓
Table Option Change
```

---

# 168. Ejemplo de metadata desconocida

Snapshot serializado:

```text
extension.acme.storage@4
```

pero la extensión no está instalada.

Con política:

```text
PRESERVE_OPAQUE
```

VoltStack conserva:

```text
OpaqueSerializedSchemaMetadata
├── key
├── version
├── raw payload
└── original context
```

sin fingir comprender su semántica.

---

# 169. Ejemplo de igualdad

Objeto A:

```text
Comment = "Users"
origin = DECLARED
```

Objeto B:

```text
Comment = "Users"
origin = OBSERVED
```

Podría cumplirse:

```text
ExactEqual(A, B) = false
```

mientras:

```text
StructuralEqual(A, B) = true
```

y:

```text
SemanticEqual(A, B) = true
```

dependiendo del perfil.

---

# 170. Ejemplo de metadata parcial

```text
users
├── columns metadata .... COMPLETE
├── indexes metadata .... COMPLETE
├── comments metadata ... UNKNOWN
└── storage metadata .... PARTIAL
```

Un Schema Diff no deberá concluir:

```text
target comment exists
current comment absent
→ add comment
```

sin aplicar la política correspondiente al estado `UNKNOWN`.

---

# 171. Architectural invariants

## DB-SCHEMA-META-001
Schema Metadata será distinta de Schema Structure.

## DB-SCHEMA-META-002
Metadata no duplicará estructura canónica.

## DB-SCHEMA-META-003
Column type no existirá únicamente como metadata.

## DB-SCHEMA-META-004
Foreign key target no existirá únicamente como metadata.

## DB-SCHEMA-META-005
Index keys no existirán únicamente como metadata.

## DB-SCHEMA-META-006
Metadata será tipada.

## DB-SCHEMA-META-007
Metadata keys serán versionadas.

## DB-SCHEMA-META-008
Metadata keys tendrán namespace.

## DB-SCHEMA-META-009
SchemaMetadataSet será inmutable.

## DB-SCHEMA-META-010
Duplicate singleton metadata será inválida.

## DB-SCHEMA-META-011
Multiplicity será explícita.

## DB-SCHEMA-META-012
Metadata category será explícita.

## DB-SCHEMA-META-013
Metadata origin será explícito cuando sea relevante.

## DB-SCHEMA-META-014
Origin será distinto de provenance.

## DB-SCHEMA-META-015
Provenance será preservada para metadata observada cuando esté disponible.

## DB-SCHEMA-META-016
Certainty será preservada cuando sea necesaria.

## DB-SCHEMA-META-017
EXACT será distinto de INFERRED.

## DB-SCHEMA-META-018
UNKNOWN será distinto de ABSENT.

## DB-SCHEMA-META-019
UNKNOWN será distinto de PLATFORM_DEFAULT.

## DB-SCHEMA-META-020
UNKNOWN será distinto de INHERITED.

## DB-SCHEMA-META-021
No value será distinto de default value.

## DB-SCHEMA-META-022
Inherited value será explícito.

## DB-SCHEMA-META-023
Platform default requerirá evidencia.

## DB-SCHEMA-META-024
Declared metadata será distinta de effective metadata.

## DB-SCHEMA-META-025
Effective metadata resolution no realizará hidden DB I/O.

## DB-SCHEMA-META-026
Metadata inheritance será descriptor-driven.

## DB-SCHEMA-META-027
No toda metadata será heredable.

## DB-SCHEMA-META-028
Metadata portability será explícita.

## DB-SCHEMA-META-029
Platform metadata no será falsamente portable.

## DB-SCHEMA-META-030
Unknown portability no será asumida portable.

## DB-SCHEMA-META-031
Semantic impact será explícito.

## DB-SCHEMA-META-032
Semantic impact será distinto de diff policy.

## DB-SCHEMA-META-033
Metadata comparison será profile-driven.

## DB-SCHEMA-META-034
Exact equality será distinta de structural equality.

## DB-SCHEMA-META-035
Structural equality será distinta de semantic equality.

## DB-SCHEMA-META-036
Platform-normalized equality será explícita.

## DB-SCHEMA-META-037
Schema Metadata Diff no utilizará comparación genérica de arrays.

## DB-SCHEMA-META-038
Unknown current metadata no implicará automáticamente change.

## DB-SCHEMA-META-039
Partial coverage no implicará metadata absent.

## DB-SCHEMA-META-040
Normalization será determinista.

## DB-SCHEMA-META-041
Normalization será idempotente donde aplique.

## DB-SCHEMA-META-042
Normalization será side-effect free.

## DB-SCHEMA-META-043
Normalization no realizará DB I/O.

## DB-SCHEMA-META-044
Normalization loss será diagnosticable.

## DB-SCHEMA-META-045
Canonical serialization order será determinista.

## DB-SCHEMA-META-046
Runtime insertion order no afectará fingerprints no ordenados.

## DB-SCHEMA-META-047
Fingerprint será versionado.

## DB-SCHEMA-META-048
Fingerprint mode será explícito.

## DB-SCHEMA-META-049
Structural fingerprint excluirá observation timestamps.

## DB-SCHEMA-META-050
Hash no será prueba absoluta de igualdad.

## DB-SCHEMA-META-051
Serialization será versionada.

## DB-SCHEMA-META-052
Serialization será determinista.

## DB-SCHEMA-META-053
PHP serialize no será contrato persistente.

## DB-SCHEMA-META-054
Unknown serialized metadata seguirá policy explícita.

## DB-SCHEMA-META-055
Opaque preservation no implicará comprensión semántica.

## DB-SCHEMA-META-056
Metadata registry será frozen tras bootstrap.

## DB-SCHEMA-META-057
Duplicate registration fallará.

## DB-SCHEMA-META-058
Last registration wins estará prohibido.

## DB-SCHEMA-META-059
Metadata versions podrán upcastearse explícitamente.

## DB-SCHEMA-META-060
Upcasters no realizarán DB I/O.

## DB-SCHEMA-META-061
Extensions no reemplazarán metadata core silenciosamente.

## DB-SCHEMA-META-062
Trust será distinto de certainty.

## DB-SCHEMA-META-063
Trust será distinto de origin.

## DB-SCHEMA-META-064
Security classification será explícita cuando corresponda.

## DB-SCHEMA-META-065
Secrets estarán prohibidos.

## DB-SCHEMA-META-066
Passwords no serán Schema Metadata.

## DB-SCHEMA-META-067
Access tokens no serán Schema Metadata.

## DB-SCHEMA-META-068
Private keys no serán Schema Metadata.

## DB-SCHEMA-META-069
Sensitive metadata podrá redactarse.

## DB-SCHEMA-META-070
Redaction no mutará metadata original.

## DB-SCHEMA-META-071
Logging utilizará redaction profile.

## DB-SCHEMA-META-072
Telemetry utilizará safe projection.

## DB-SCHEMA-META-073
Diagnostics no modificarán metadata truth.

## DB-SCHEMA-META-074
Source location será informational.

## DB-SCHEMA-META-075
Source location no afectará structural equality.

## DB-SCHEMA-META-076
Comments podrán excluirse de semantic equality.

## DB-SCHEMA-META-077
Comments podrán incluirse en diff mediante policy.

## DB-SCHEMA-META-078
Logical names serán distintos de physical names.

## DB-SCHEMA-META-079
Platform metadata tendrá platform identity.

## DB-SCHEMA-META-080
MariaDB metadata podrá diferir de MySQL metadata.

## DB-SCHEMA-META-081
Vendor conditionals dispersos estarán prohibidos.

## DB-SCHEMA-META-082
Capability validation será separada de local validation.

## DB-SCHEMA-META-083
Metadata target kinds serán explícitos.

## DB-SCHEMA-META-084
Table-only metadata no podrá adjuntarse a Column.

## DB-SCHEMA-META-085
Cross-object structure no será escondida en metadata.

## DB-SCHEMA-META-086
Derived metadata será distinta de stored metadata.

## DB-SCHEMA-META-087
Deterministically derivable metadata no será duplicada sin necesidad.

## DB-SCHEMA-META-088
Metadata projections serán inmutables.

## DB-SCHEMA-META-089
Semantic consumers podrán solicitar semantic projection.

## DB-SCHEMA-META-090
Telemetry consumers podrán solicitar safe projection.

## DB-SCHEMA-META-091
Metadata merge tendrá policy explícita.

## DB-SCHEMA-META-092
Generic last-wins merge estará prohibido.

## DB-SCHEMA-META-093
Metadata conflicts serán explícitos.

## DB-SCHEMA-META-094
Equal values de múltiples fuentes podrán compartir evidence.

## DB-SCHEMA-META-095
Evidence será distinta del valor canónico.

## DB-SCHEMA-META-096
Metadata set builder será operation-scoped.

## DB-SCHEMA-META-097
Published metadata set será sealed.

## DB-SCHEMA-META-098
Shared registry será immutable.

## DB-SCHEMA-META-099
Request metadata no vivirá en singleton mutable.

## DB-SCHEMA-META-100
Tenant metadata no se filtrará entre requests.

## DB-SCHEMA-META-101
No existirá current tenant global en metadata core.

## DB-SCHEMA-META-102
Cache será integración opcional.

## DB-SCHEMA-META-103
Cached metadata preservará version identity.

## DB-SCHEMA-META-104
Metadata lookup deberá ser eficientemente indexable.

## DB-SCHEMA-META-105
Serialization deberá tener orden estable.

## DB-SCHEMA-META-106
Lazy fingerprint caching solo será permitido sobre objetos inmutables.

## DB-SCHEMA-META-107
Budgets serán explícitos.

## DB-SCHEMA-META-108
Budget exhaustion no truncará silenciosamente.

## DB-SCHEMA-META-109
Opaque payloads tendrán límites.

## DB-SCHEMA-META-110
Extension metadata tendrá límites.

## DB-SCHEMA-META-111
Schema Builder podrá producir declared metadata.

## DB-SCHEMA-META-112
Schema Introspection podrá producir observed metadata.

## DB-SCHEMA-META-113
Schema Model podrá transportar metadata sin depender de introspection.

## DB-SCHEMA-META-114
Schema Diff consumirá metadata mediante políticas explícitas.

## DB-SCHEMA-META-115
Schema Compiler no reinterpretará metadata desconocida.

## DB-SCHEMA-META-116
Schema Compiler solo consumirá metadata que comprenda explícitamente.

## DB-SCHEMA-META-117
Unsupported metadata no será silenciosamente compilada.

## DB-SCHEMA-META-118
ORM no será propietario del Metadata System.

## DB-SCHEMA-META-119
Query Engine no mutará Schema Metadata.

## DB-SCHEMA-META-120
Semantic Engine no realizará introspection mediante Metadata System.

## DB-SCHEMA-META-121
Metadata resolution será context-explicit.

## DB-SCHEMA-META-122
Platform capabilities serán suministradas explícitamente.

## DB-SCHEMA-META-123
Metadata comparisons serán deterministas.

## DB-SCHEMA-META-124
Metadata fingerprints serán deterministas.

## DB-SCHEMA-META-125
Metadata serialization round-trip preservará semántica soportada.

## DB-SCHEMA-META-126
Unknown metadata round-trip preservará opaque payload cuando policy lo exija.

## DB-SCHEMA-META-127
Invalid metadata producirá errores tipados.

## DB-SCHEMA-META-128
Metadata conflicts no serán tratados como excepciones genéricas sin contexto.

## DB-SCHEMA-META-129
Schema Metadata System no abrirá conexiones.

## DB-SCHEMA-META-130
Schema Metadata System no ejecutará SQL.

## DB-SCHEMA-META-131
Schema Metadata System no ejecutará DDL.

## DB-SCHEMA-META-132
Schema Metadata System no ejecutará migrations.

## DB-SCHEMA-META-133
Schema Metadata System no autorizará cambios.

## DB-SCHEMA-META-134
Metadata no podrá omitir safety gates del Schema Diff.

## DB-SCHEMA-META-135
Metadata no podrá omitir capability validation.

## DB-SCHEMA-META-136
Metadata extension no podrá ejecutar arbitrary SQL.

## DB-SCHEMA-META-137
Metadata extension no podrá elevar privileges.

## DB-SCHEMA-META-138
Metadata extension no podrá acceder a tenant global oculto.

## DB-SCHEMA-META-139
FrankenPHP state isolation será obligatorio.

## DB-SCHEMA-META-140
RoadRunner state isolation será obligatorio.

## DB-SCHEMA-META-141
OpenSwoole state isolation será obligatorio.

## DB-SCHEMA-META-142
Metadata categories no reemplazarán typed metadata classes.

## DB-SCHEMA-META-143
Informational metadata será distinta de semantic metadata.

## DB-SCHEMA-META-144
Physical metadata será distinta de portable semantic structure.

## DB-SCHEMA-META-145
Observed metadata será distinta de declared metadata.

## DB-SCHEMA-META-146
Unknown metadata será distinta de default metadata.

## DB-SCHEMA-META-147
Metadata nunca será una segunda fuente de verdad estructural.

## DB-SCHEMA-META-148
Toda metadata crítica deberá tener semántica documentada.

## DB-SCHEMA-META-149
Toda metadata extension deberá estar versionada.

## DB-SCHEMA-META-150
El sistema preservará incertidumbre en lugar de inventar certeza.

---

# 172. Anti-patterns

## 172.1 Metadata como array universal

Incorrecto:

```php
$table->metadata = [
    'engine' => 'InnoDB',
    'comment' => 'Users',
    'nullable' => false,
    'primary_key' => ['id'],
];
```

Mezcla:

```text
structure
metadata
platform options
semantics
```

en una sola bolsa.

---

# 173. Duplicar estructura

Incorrecto:

```text
ColumnDefinition:
nullable = false

Metadata:
nullable = false
```

Ahora existen dos fuentes de verdad.

---

# 174. Last wins

Incorrecto:

```php
$metadata[$key] = $newValue;
```

sin verificar conflicto.

---

# 175. Unknown → default

Incorrecto:

```php
$engine = $metadata->engine() ?? 'InnoDB';
```

si `null` significa:

```text
UNKNOWN
```

---

# 176. Observed → declared

Incorrecto:

```text
Database says X
therefore application declared X
```

---

# 177. Platform metadata falsa

Incorrecto:

```text
storageEngine = InnoDB
```

como propiedad portable de todas las bases de datos.

---

# 178. Hidden inheritance

Incorrecto:

```php
$column->collation();
```

que silenciosamente consulta database/table/global config sin contexto explícito.

---

# 179. Source location in structural hash

Incorrecto:

```text
same schema
different source line
=
different structural fingerprint
```

---

# 180. Secrets in metadata

Incorrecto:

```php
new DatabaseMetadata([
    'password' => 'secret',
]);
```

---

# 181. Raw extension execution

Incorrecto:

```php
$metadata->apply($connection);
```

La metadata describe.

No ejecuta.

---

# 182. Metadata-driven hidden DDL

Incorrecto:

```text
Metadata attached
    ↓
automatically executes ALTER TABLE
```

---

# 183. Testing strategy

Debe probarse:

```text
typed metadata creation
duplicate detection
target validation
normalization
idempotence
comparison profiles
merge policies
conflict preservation
inheritance
effective resolution
fingerprinting
serialization
upcasting
opaque preservation
redaction
budgets
platform metadata
extension metadata
persistent runtime isolation
```

---

# 184. Property-based tests

Especialmente útiles para:

```text
Normalize(Normalize(M)) = Normalize(M)
Serialize/Deserialize(M) ≈ M
Fingerprint(M) = Fingerprint(Clone(M))
Merge determinism
Projection determinism
```

---

# 185. Cross-platform tests

Una misma metadata portable deberá producir semántica consistente sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando la capability exista.

Metadata específica deberá permanecer scoped.

---

# 186. Snapshot round-trip

```text
Observed Metadata
      │
      ▼
SchemaSnapshot
      │
      ▼
Serialize
      │
      ▼
Deserialize
      │
      ▼
Equivalent Metadata
```

preservando:

```text
value
type
version
origin
provenance
certainty
opaque extensions
```

según el perfil.

---

# 187. Integration with Schema Diff

El siguiente sistema utilizará:

```text
Current Schema
+
Current Metadata
+
Target Schema
+
Target Metadata
+
Coverage
+
Comparison Profile
+
Platform Capabilities
```

para producir diferencias estructuradas.

La relación será:

```text
Schema Metadata
        │
        ▼
 Metadata Comparison
        │
        ▼
Schema Diff Engine
```

---

# 188. Metadata diff safety equation

Para una metadata `M`:

```text
SafeDifference(M)
=
Comparable(Current(M), Target(M))
∧
SufficientCoverage(Current(M))
∧
KnownSemantics(M)
∧
ApplicableComparisonPolicy(M)
```

Por tanto:

```text
Different bytes
≠
Safe schema difference
```

---

# 189. Metadata equality equation

```text
MetadataEqual(A, B, Profile)
=
Comparator(
    Normalize(A),
    Normalize(B),
    Profile
)
```

No:

```text
serialize(A) === serialize(B)
```

---

# 190. Metadata effective-value equation

```text
EffectiveMetadata(O)
=
Resolve(
    ExplicitMetadata(O),
    ParentMetadata(O),
    PlatformDefaults,
    Capabilities,
    ResolutionPolicy
)
```

sin hidden I/O.

---

# 191. Metadata provenance equation

Para metadata observada:

```text
ObservedMetadata
=
Value
+
Origin
+
Provenance
+
Certainty
+
Coverage
```

---

# 192. Metadata architecture formula

```text
Schema Metadata System
=
Typed Metadata
+
Classification
+
Origin
+
Provenance
+
Certainty
+
Coverage
+
Inheritance
+
Portability
+
Platform Metadata
+
Normalization
+
Comparison
+
Diff Participation
+
Merge
+
Evidence
+
Projection
+
Fingerprinting
+
Serialization
+
Versioning
+
Extension Control
+
Security
+
Budgets
+
Diagnostics
+
Persistent Runtime Isolation
```

---

# 193. Correctness formula

```text
CorrectSchemaMetadata
=
Typed
∧
Immutable
∧
Deterministic
∧
NonDuplicative
∧
ProvenanceAware
∧
UncertaintyPreserving
∧
PlatformHonest
∧
ComparisonExplicit
∧
ExtensionSafe
∧
SecurityAware
∧
RuntimeIsolated
```

---

# 194. Arquitectura final

```text
             Schema Builder
                  │
                  ▼
           Declared Metadata
                  │
                  │
                  ▼
          ┌─────────────────┐
          │ Schema Metadata │
          │     System      │
          └─────────────────┘
                  ▲
                  │
                  │
          Observed Metadata
                  ▲
                  │
          Schema Introspection
                  ▲
                  │
             Database
```

El sistema produce:

```text
SchemaMetadataSet
├── Typed Entries
├── Context
│   ├── Origin
│   ├── Provenance
│   ├── Certainty
│   └── Trust
│
├── Coverage
├── Platform Metadata
├── Extension Metadata
└── Diagnostics
```

que puede consumirse mediante:

```text
                 SchemaMetadataSet
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
 Semantic View       Diff View        Telemetry View
       │                 │                  │
       ▼                 ▼                  ▼
 Query Engine        Schema Diff        Telemetry
```

---

# 195. Regla arquitectónica final

> **La estructura dice qué es el esquema; la metadata explica propiedades, contexto, evidencia y características adicionales de esa estructura.**

Por tanto:

```text
Structure defines.
Metadata describes.
Introspection observes.
Coverage limits certainty.
Comparison interprets.
Diff decides whether differences matter.
```

Y la regla más importante:

```text
Metadata must enrich truth,
never duplicate it,
never silently replace it,
and never invent it.
```

---

# 196. Resultado

Con `Database Schema Metadata System`, VoltStack obtiene una capa capaz de preservar simultáneamente:

```text
portable schema semantics
+
platform-specific details
+
declared intent
+
observed evidence
+
uncertainty
+
coverage
+
extension information
```

sin degradar el Schema Model a una colección genérica de propiedades.

Esto permite que los siguientes subsistemas puedan distinguir correctamente entre:

```text
actual structural difference
platform representation difference
informational difference
unknown difference
observation uncertainty
metadata drift
```

antes de generar cualquier operación de modificación.

---

# 197. Siguiente documento

```text
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
```

El siguiente documento definirá el motor responsable de transformar:

```text
Current Schema Snapshot
        +
Target Schema
        +
Comparison Profile
        +
Platform Capabilities
        ↓
Schema Diff
```

y deberá formalizar especialmente:

```text
Difference
≠
Change Operation

Difference
≠
Migration

Missing
≠
Dropped

Added
≠
Created

Rename
≠
Drop + Add

Structural Difference
≠
Semantic Difference

Metadata Difference
≠
Schema Difference

Detected Difference
≠
Authorized Change
```

además de establecer la base para:

```text
99_DATABASE_SCHEMA_COMPILER_SYSTEM.md
```

y posteriormente:

```text
101_DATABASE_MIGRATION_ARCHITECTURE.md
```