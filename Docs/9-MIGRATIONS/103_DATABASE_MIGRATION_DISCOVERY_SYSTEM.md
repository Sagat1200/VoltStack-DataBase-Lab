# 103_DATABASE_MIGRATION_DISCOVERY_SYSTEM.md

# VoltStack Quantum Database
## Database Migration Discovery System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 103 — Database Migration Discovery System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration Discovery System` define la arquitectura mediante la cual VoltStack:

- localiza fuentes de migraciones;
- descubre candidatos;
- identifica sus descriptores;
- normaliza sus identidades;
- detecta colisiones;
- clasifica su procedencia;
- aplica filtros de descubrimiento;
- construye un catálogo determinista de migraciones disponibles;
- permite descubrimiento desde aplicaciones, módulos, paquetes y extensiones;
- mantiene el proceso independiente del estado histórico de ejecución.

El sistema se sitúa inmediatamente después de:

```text
101_DATABASE_MIGRATION_ARCHITECTURE.md
102_DATABASE_MIGRATION_SYSTEM.md
```

y proporciona la entrada estructurada para:

```text
104_DATABASE_MIGRATION_REPOSITORY_SYSTEM.md
105_DATABASE_MIGRATION_PLANNER_SYSTEM.md
```

La pregunta fundamental de este documento es:

> **¿Cómo sabe VoltStack qué migraciones existen sin confundir su descubrimiento con cargarlas completamente, ejecutarlas o consultar cuáles están pendientes?**

---

# 2. Principio central

> **Discovery descubre disponibilidad; no determina estado de ejecución.**

Formalmente:

```text
Discovery
=
Available Migration Sources
→
Migration Descriptors
→
Migration Catalog
```

y nunca:

```text
Discovery
=
Database State
→
Pending Migrations
```

Por tanto:

```text
Discovery
≠
Repository

Discovery
≠
Pending Calculation

Discovery
≠
Planning

Discovery
≠
Execution
```

---

# 3. Separación arquitectónica fundamental

VoltStack distinguirá:

```text
Migration Source
Migration Candidate
Migration Descriptor
Migration Definition
Migration Repository Record
Migration Plan
Migration Execution
```

como conceptos independientes.

Pipeline:

```text
Migration Sources
      │
      ▼
Discovery
      │
      ▼
Candidates
      │
      ▼
Descriptor Extraction
      │
      ▼
Identity Validation
      │
      ▼
Catalog Assembly
      │
      ▼
Migration Catalog
```

Posteriormente:

```text
Migration Catalog
        +
Migration Repository
        ↓
Migration Planner
```

---

# 4. Responsabilidades

El Discovery System será responsable de:

```text
Source registration
Source enumeration
Candidate discovery
Descriptor extraction
Identity normalization
Source metadata collection
Duplicate detection
Namespace resolution
Discovery filtering
Deterministic catalog assembly
Discovery diagnostics
Discovery caching
Extension integration
Resource governance
```

No será responsable de:

```text
SQL execution
Schema introspection
Migration execution
Rollback
Transaction management
Database locking
Batch assignment
Pending calculation
Migration planning
Repository persistence
```

---

# 5. Modelo conceptual

```text
MigrationDiscoverySystem
│
├── MigrationSourceRegistry
│
├── MigrationSource[]
│
├── MigrationCandidateScanner
│
├── MigrationDescriptorExtractor
│
├── MigrationIdentityResolver
│
├── MigrationDiscoveryFilterPipeline
│
├── MigrationCollisionDetector
│
├── MigrationCatalogBuilder
│
└── MigrationDiscoveryDiagnostics
```

Resultado:

```text
MigrationCatalog
```

---

# 6. Migration Source

Una `MigrationSource` representa un origen lógico desde el cual pueden descubrirse migraciones.

Contrato:

```php
interface MigrationSource
{
    public function id(): MigrationSourceId;

    public function kind(): MigrationSourceKind;

    public function namespace(): MigrationNamespace;

    public function discover(
        MigrationDiscoveryContext $context
    ): iterable;
}
```

El iterable deberá producir:

```text
MigrationCandidate
```

o directamente descriptors cuando la fuente pueda proporcionarlos eficientemente.

---

# 7. MigrationSourceId

```php
final readonly class MigrationSourceId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
app.database.migrations
voltstack.framework.database
package.acme.billing
plugin.audit.migrations
testing.fixture.migrations
```

---

# 8. Source ID ≠ namespace

Debe mantenerse:

```text
MigrationSourceId
≠
MigrationNamespace
```

Una fuente describe:

```text
where migrations come from
```

mientras namespace describe:

```text
migration identity domain
```

---

# 9. MigrationSourceKind

Se propone:

```php
enum MigrationSourceKind
{
    case APPLICATION;
    case FRAMEWORK;
    case PACKAGE;
    case MODULE;
    case PLUGIN;
    case GENERATED;
    case TESTING;
    case EXTERNAL;
}
```

---

# 10. Fuente de aplicación

Ejemplo:

```text
database/migrations/
```

podrá registrarse como:

```text
SourceId:
app.database.migrations

Namespace:
app
```

---

# 11. Fuente del framework

VoltStack podrá tener migraciones internas:

```text
VoltStack/Quantum/Database/Migrations
```

pero deberán estar aisladas mediante namespace:

```text
voltstack.database
```

---

# 12. Migraciones de paquetes

Un paquete podrá registrar:

```php
$migrations->registerSource(
    PackageMigrationSource::fromPath(
        package: 'acme/billing',
        path: __DIR__.'/../database/migrations',
        namespace: 'package.acme.billing',
    )
);
```

---

# 13. Migraciones de módulos

VoltStack podrá soportar módulos:

```text
Modules/
├── Billing/
│   └── Database/Migrations/
├── Inventory/
│   └── Database/Migrations/
└── CRM/
    └── Database/Migrations/
```

cada uno con namespace propio.

---

# 14. Plugin migrations

Las extensiones podrán registrar fuentes mediante contratos oficiales.

No podrán modificar directamente el catálogo interno.

---

# 15. Source Registry

Se propone:

```php
interface MigrationSourceRegistry
{
    public function register(MigrationSource $source): void;

    public function freeze(): FrozenMigrationSourceRegistry;
}
```

Después del bootstrap:

```text
Mutable Registry
      ↓ freeze
FrozenMigrationSourceRegistry
```

---

# 16. Registry lifecycle

```text
Framework Bootstrap
      ↓
Register framework sources
      ↓
Register application sources
      ↓
Register package sources
      ↓
Register plugin sources
      ↓
Validate
      ↓
Freeze
```

---

# 17. Frozen registry

Después de `freeze()`:

```text
register()
remove()
replace()
```

deberán rechazarse.

Esto es especialmente importante bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 18. Duplicate Source ID

Dos fuentes con:

```text
sourceId = package.acme.billing
```

deberán producir:

```text
DuplicateMigrationSourceException
```

No:

```text
last registration wins
```

---

# 19. Source priority

No se recomienda utilizar prioridad para resolver colisiones de identidad.

Incorrecto:

```text
application migration overrides package migration
```

Si dos migraciones tienen la misma identidad:

```text
collision
```

debe reportarse.

---

# 20. Source ordering

El registry podrá mantener orden determinista para discovery.

Pero:

```text
Source Registration Order
≠
Migration Execution Order
```

---

# 21. MigrationCandidate

Un candidate representa un artefacto encontrado antes de tener un descriptor validado.

```php
final readonly class MigrationCandidate
{
    public function __construct(
        public MigrationSourceId $source,
        public MigrationCandidateReference $reference,
        public MigrationCandidateMetadata $metadata,
    ) {}
}
```

---

# 22. Candidate reference

Puede representar:

```text
PHP file
class name
package resource
generated artifact
manifest entry
extension-defined reference
```

---

# 23. Candidate ≠ Descriptor

```text
Candidate
=
something that may be a migration
```

```text
Descriptor
=
validated lightweight migration description
```

---

# 24. Candidate ≠ Definition

El candidate no necesita cargar:

```text
MigrationDefinition
operations
rollback
full dependencies
```

---

# 25. File candidate

Ejemplo:

```text
database/migrations/
2026_09_06_010000_create_users.php
```

produce:

```text
MigrationCandidate
├── source = app.database.migrations
├── reference = file://...create_users.php
└── metadata
```

---

# 26. Descriptor extraction

Después:

```text
Candidate
    ↓
Descriptor Extractor
    ↓
MigrationDescriptor
```

Contrato:

```php
interface MigrationDescriptorExtractor
{
    public function supports(
        MigrationCandidate $candidate
    ): bool;

    public function extract(
        MigrationCandidate $candidate,
        MigrationDiscoveryContext $context
    ): MigrationDescriptor;
}
```

---

# 27. Descriptor extraction no ejecuta migration

El extractor no deberá invocar:

```text
up()
down()
MigrationDefinition construction
```

si la identidad puede obtenerse sin hacerlo.

---

# 28. Lightweight descriptor

Idealmente:

```text
MigrationDescriptor
├── identity
├── source
├── sourceKind
├── discovery metadata
├── definition loader reference
└── optional lightweight metadata
```

---

# 29. MigrationDescriptor

Tomando la abstracción del documento 102:

```php
final readonly class MigrationDescriptor
{
    public function __construct(
        public MigrationIdentity $identity,
        public MigrationSourceReference $source,
        public MigrationDescriptorMetadata $metadata,
        public MigrationDefinitionReference $definition,
    ) {}
}
```

---

# 30. DefinitionReference

Permite:

```text
Descriptor
   │
   └── definitionReference
              ↓
        Definition Loader
              ↓
      MigrationDefinition
```

sin cargarla durante discovery.

---

# 31. Descriptor metadata

Podrá contener:

```text
source kind
source ID
relative source path
display name
discovery timestamp? runtime-only
source checksum
package identity
module identity
extension identity
trust level
```

El timestamp de discovery:

```text
must not participate in semantic identity
```

---

# 32. Filename discovery

VoltStack podrá utilizar convención:

```text
YYYY_MM_DD_HHMMSS_description.php
```

como estrategia predeterminada.

Ejemplo:

```text
2026_09_06_010000_create_users.php
```

---

# 33. Filename ≠ canonical identity

Regla:

```text
Filename
≠
MigrationIdentity
```

aunque una estrategia pueda derivar provisionalmente el ID desde el filename.

---

# 34. Identity extraction

Se propone:

```text
Candidate
↓
Descriptor Extractor
↓
Declared / Derived Identity
↓
MigrationIdentityResolver
↓
Canonical MigrationIdentity
```

---

# 35. MigrationIdentityResolver

```php
interface MigrationIdentityResolver
{
    public function resolve(
        MigrationCandidate $candidate,
        MigrationDescriptorData $data,
        MigrationDiscoveryContext $context
    ): MigrationIdentity;
}
```

---

# 36. Identity resolution priority

Preferencia:

```text
1. Explicit declared MigrationIdentity
2. Manifest identity
3. Source-specific deterministic identity
4. Filename convention
```

Nunca:

```text
random identity
```

---

# 37. Qualified identity

Canonical:

```text
namespace:migration-id
```

Ejemplos:

```text
app:2026_09_06_010000_create_users

package.acme.billing:
2026_09_06_020000_create_invoices
```

---

# 38. Namespace assignment

La fuente puede proporcionar un namespace predeterminado.

Ejemplo:

```text
MigrationSource
namespace = package.acme.billing
```

Un descriptor sin namespace explícito hereda ese namespace bajo reglas controladas.

---

# 39. Namespace override

Una migration no deberá poder escapar arbitrariamente del namespace de su fuente.

Ejemplo peligroso:

```text
source:
package.evil

migration declares:
voltstack.core
```

Debe ser validado mediante policy.

---

# 40. Namespace policy

Se propone:

```php
interface MigrationNamespacePolicy
{
    public function validate(
        MigrationSource $source,
        MigrationIdentity $identity
    ): MigrationNamespaceDecision;
}
```

---

# 41. Identity normalization

Normalización puede incluir:

```text
trim
valid character validation
canonical namespace separator
Unicode policy
length limits
reserved namespaces
```

pero no deberá cambiar identidad semántica silenciosamente.

---

# 42. Case sensitivity

Debe definirse explícitamente.

Recomendación:

```text
Migration IDs
=
case-sensitive canonical strings
```

pero namespaces pueden utilizar una convención estricta:

```text
lowercase dot-separated
```

Ejemplo:

```text
package.acme.billing
```

---

# 43. Duplicate identity detection

Después de resolver descriptors:

```text
MigrationDescriptor[]
        ↓
Identity Index
        ↓
Collision Detection
```

---

# 44. Collision definition

Existe colisión cuando:

```text
Identity(A) = Identity(B)
∧
SourceReference(A) ≠ SourceReference(B)
```

---

# 45. Collision behavior

VoltStack deberá fallar.

Nunca:

```text
first wins
last wins
newest wins
application wins
package wins
```

---

# 46. MigrationCollision

Se propone:

```php
final readonly class MigrationCollision
{
    public function __construct(
        public MigrationIdentity $identity,
        public array $descriptors,
    ) {}
}
```

---

# 47. Collision diagnostic

Ejemplo:

```text
Duplicate migration identity:

package.acme.billing:
2026_09_06_020000_create_invoices

Sources:
- vendor/acme/billing/database/migrations/...
- modules/Billing/database/migrations/...
```

---

# 48. Duplicate source artifact

Un mismo archivo puede aparecer mediante dos paths simbólicos o registries duplicados.

Discovery deberá poder detectar:

```text
Same physical/logical source artifact
```

cuando sea posible.

---

# 49. Artifact identity

Puede existir:

```text
MigrationSourceArtifactId
```

separado de:

```text
MigrationIdentity
```

para deduplicar el mismo recurso descubierto por múltiples rutas.

---

# 50. Symlink handling

El comportamiento deberá ser configurable:

```text
FOLLOW
IGNORE
FAIL
```

con límites para evitar:

```text
recursive symlink loops
```

---

# 51. Directory traversal

El scanner deberá aplicar:

```text
depth limit
file count limit
path normalization
extension filters
symlink policy
cancellation
```

---

# 52. MigrationCandidateScanner

```php
interface MigrationCandidateScanner
{
    /**
     * @return iterable<MigrationCandidate>
     */
    public function scan(
        MigrationSource $source,
        MigrationDiscoveryContext $context
    ): iterable;
}
```

---

# 53. Filesystem scanner

Para fuentes filesystem:

```text
FilesystemMigrationScanner
```

deberá limitarse a descubrir candidatos.

No deberá:

```text
require every PHP file immediately
instantiate migration
execute code
```

---

# 54. Manifest-based discovery

Para paquetes grandes puede utilizarse:

```text
migrations.manifest.php
```

o un artefacto compilado equivalente.

Ejemplo conceptual:

```php
return [
    [
        'id' => '2026_09_06_010000_create_users',
        'file' => '...',
    ],
];
```

---

# 55. Manifest advantages

Permite evitar:

```text
recursive filesystem scanning
```

en producción.

Pipeline:

```text
Package
  ↓
Migration Manifest
  ↓
Descriptors
```

---

# 56. Manifest ≠ source of truth necesariamente

Durante desarrollo:

```text
filesystem
```

puede ser fuente primaria.

Durante producción:

```text
compiled manifest
```

puede ser optimización.

El sistema deberá detectar manifest obsoleto cuando corresponda.

---

# 57. Compiled migration catalog

VoltStack podrá generar:

```text
bootstrap/cache/database-migrations.php
```

conceptualmente.

Pero:

```text
Compiled Catalog
≠
Repository
```

---

# 58. Discovery cache

El cache puede almacenar:

```text
descriptors
source fingerprints
catalog fingerprint
```

Nunca:

```text
live Migration objects
connections
repository state
execution state
```

---

# 59. Cache key

Conceptualmente:

```text
DiscoveryCacheKey
=
H(
    SourceRegistryFingerprint,
    DiscoveryProfile,
    ExtensionRegistryFingerprint,
    DiscoveryAlgorithmVersion
)
```

---

# 60. Source fingerprint

Una fuente podrá proporcionar:

```php
interface FingerprintableMigrationSource
{
    public function fingerprint(): MigrationSourceFingerprint;
}
```

---

# 61. Filesystem source fingerprint

Puede derivarse de:

```text
canonical source path
relevant file metadata
manifest checksum
source configuration
```

según profile.

No depender únicamente de `mtime` si se requiere alta confiabilidad.

---

# 62. Catalog fingerprint

```text
MigrationCatalogFingerprint
=
H(
    ordered descriptor identities,
    descriptor source fingerprints,
    discovery version
)
```

---

# 63. Catalog

Resultado principal:

```php
final readonly class MigrationCatalog
{
    public function __construct(
        public MigrationDescriptorSet $descriptors,
        public MigrationCatalogMetadata $metadata,
        public MigrationCatalogFingerprint $fingerprint,
    ) {}
}
```

---

# 64. Catalog properties

El catálogo deberá ser:

```text
immutable
deterministic
indexed
collision-free
source-traceable
namespace-aware
```

---

# 65. Catalog indexes

Internamente:

```text
by qualified identity
by namespace
by source ID
by source kind
possibly by package/module
```

---

# 66. Catalog order

Debe existir un orden canónico para:

```text
serialization
diagnostics
fingerprinting
testing
```

pero:

```text
Catalog Order
≠
Execution Order
```

---

# 67. Recommended canonical ordering

Por ejemplo:

```text
namespace ASC
migration ID ASC
source ID ASC
```

únicamente como orden determinista de representación.

---

# 68. Timestamp ordering

Si IDs utilizan timestamps:

```text
2026_09_06_010000_...
2026_09_06_020000_...
```

el planner podrá aprovechar ese orden como fallback.

Pero:

```text
Timestamp Order
≠
Dependency Truth
```

---

# 69. Explicit dependencies dominate

Si:

```text
M3 depends on M1
M2 independent
```

el planner construirá el orden real.

Discovery solo conserva la información disponible.

---

# 70. Discovery profile

Se propone:

```php
enum MigrationDiscoveryProfile
{
    case DEVELOPMENT;
    case PRODUCTION;
    case TESTING;
    case COMPILED;
}
```

---

# 71. Development profile

Puede favorecer:

```text
filesystem scanning
rich diagnostics
source maps
manifest validation
```

---

# 72. Production profile

Puede favorecer:

```text
compiled manifests
cached catalogs
strict source validation
minimal filesystem traversal
```

---

# 73. Testing profile

Puede permitir:

```text
in-memory sources
synthetic descriptors
fixture migrations
```

---

# 74. Compiled profile

Puede exigir:

```text
no filesystem scan
```

y cargar únicamente un catálogo previamente generado.

---

# 75. DiscoveryContext

```php
final readonly class MigrationDiscoveryContext
{
    public function __construct(
        public MigrationDiscoveryProfile $profile,
        public FrozenMigrationSourceRegistry $sources,
        public FrozenMigrationDiscoveryExtensionRegistry $extensions,
        public MigrationDiscoveryBudget $budget,
        public CancellationToken $cancellation,
    ) {}
}
```

---

# 76. Context exclusions

No deberá contener:

```text
Connection
PDO
Transaction
MigrationRepository mutable handle
current tenant global
HTTP request
```

---

# 77. Cancellation

Discovery deberá respetar:

```text
CancellationToken
```

durante:

```text
filesystem traversal
manifest parsing
descriptor extraction
catalog construction
```

---

# 78. Discovery budget

Se propone:

```php
final readonly class MigrationDiscoveryBudget
{
    public function __construct(
        public int $maxSources,
        public int $maxCandidates,
        public int $maxDescriptors,
        public int $maxDirectories,
        public int $maxDepth,
        public int $maxManifestBytes,
        public int $maxPathLength,
        public int $maxDiagnostics,
    ) {}
}
```

---

# 79. Budget overflow

Debe producir:

```text
MigrationDiscoveryBudgetExceededException
```

Nunca:

```text
return partial catalog as complete
```

---

# 80. Discovery completeness

Se propone:

```php
enum MigrationDiscoveryCompleteness
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 81. Complete catalog

Solo puede declararse:

```text
COMPLETE
```

si todas las fuentes requeridas fueron procesadas correctamente.

---

# 82. Partial discovery

Ejemplo:

```text
Application source → success
Package A → success
Package B → inaccessible
```

resultado:

```text
PARTIAL
```

No deberá tratarse automáticamente como catálogo completo.

---

# 83. Unknown discovery

Si no puede determinarse el alcance esperado:

```text
UNKNOWN
```

---

# 84. Pending calculation safety

Regla crítica:

```text
Incomplete Catalog
+
Repository
≠
Safe Pending Migration Set
```

El Planner deberá considerar completeness.

---

# 85. Discovery diagnostics

Tipos:

```text
INFO
WARNING
ERROR
FATAL
```

Ejemplos:

```text
ignored non-migration file
invalid filename convention
duplicate source
duplicate migration identity
manifest stale
source inaccessible
unsupported descriptor
namespace violation
budget approaching limit
```

---

# 86. Diagnostics ≠ exceptions

Problemas recuperables pueden producir diagnostic.

Problemas que invalidan corrección deben producir exception/failure.

---

# 87. Strictness policy

Se propone:

```text
STRICT
NORMAL
LENIENT
```

pero `LENIENT` nunca deberá ocultar:

```text
identity collision
catalog corruption
security violation
```

---

# 88. Invalid migration file

Ejemplo:

```text
2026_create_users.php
```

si no cumple convención.

Según policy:

```text
IGNORE_WITH_DIAGNOSTIC
FAIL
```

pero nunca inventar identidad no determinista.

---

# 89. PHP source loading security

Los archivos PHP de migraciones son código.

Por ello:

```text
Discovery
```

debe minimizar su ejecución.

Preferencia:

```text
filename/manifest/metadata extraction
```

antes de:

```text
require file
```

---

# 90. Static descriptor metadata

Cuando sea posible, VoltStack puede utilizar:

```text
manifest
generated descriptor
PHP attribute metadata
compiled registry
```

para evitar ejecutar código durante discovery.

---

# 91. Attributes

Ejemplo conceptual:

```php
#[DatabaseMigration(
    id: '2026_09_06_010000_create_users',
    namespace: 'app'
)]
final class CreateUsersMigration extends Migration
{
}
```

Sin embargo, descubrir atributos de una clase puede requerir cargarla.

Por ello un manifest compilado sigue siendo preferible en producción.

---

# 92. Migration manifests generation

Una herramienta CLI futura podrá hacer:

```text
voltstack database:migrations:compile
```

conceptualmente.

Resultado:

```text
source scanning
     ↓
descriptor validation
     ↓
catalog generation
     ↓
compiled manifest
```

---

# 93. CLI ownership

El comando pertenece al futuro:

```text
309_DATABASE_CLI_SYSTEM.md
```

Discovery solo expone los contratos necesarios.

---

# 94. Source abstraction

Discovery no deberá asumir que toda migration está en filesystem.

Fuentes futuras:

```text
Composer package manifest
PHAR
generated module
precompiled deployment artifact
testing memory source
framework registry
```

---

# 95. InMemoryMigrationSource

Para testing:

```php
$source = new InMemoryMigrationSource([
    $descriptorA,
    $descriptorB,
]);
```

sin filesystem.

---

# 96. Package source integration

Composer podrá utilizarse durante bootstrap para localizar paquetes registrados.

Pero:

```text
Composer Discovery
≠
Migration Discovery Core
```

Debe existir un adapter.

---

# 97. Framework package adapter

Conceptualmente:

```text
ComposerPackage
      ↓
VoltStack Package Metadata
      ↓
MigrationSource Registration
```

---

# 98. Automatic package discovery

Podrá existir:

```text
package manifest
```

con:

```text
database.migrations.path
database.migrations.namespace
```

---

# 99. Explicit registration remains valid

Paquetes avanzados podrán registrar:

```php
$database->migrations()->sources()->register(...);
```

durante bootstrap.

---

# 100. No runtime source registration

Después del bootstrap:

```text
FrozenMigrationSourceRegistry
```

evita que una request añada una fuente global.

---

# 101. Persistent runtime model

Compartible:

```text
FrozenMigrationSourceRegistry
FrozenMigrationDiscoveryExtensionRegistry
Discovery configuration
Compiled MigrationCatalog
```

Operation-scoped:

```text
MigrationDiscoverySession
candidate buffers
diagnostics
filesystem iterators
temporary indexes
```

---

# 102. FrankenPHP

En FrankenPHP:

```text
Worker
├── immutable registry
├── immutable configuration
└── request/command
      └── fresh DiscoverySession
```

---

# 103. RoadRunner

Mismo principio:

```text
Worker state
≠
mutable discovery session
```

---

# 104. OpenSwoole

Especial cuidado:

```text
Coroutine A
DiscoverySession A

Coroutine B
DiscoverySession B
```

Nunca compartir:

```text
candidate cursor
diagnostic buffer
mutable catalog builder
```

---

# 105. DiscoverySession

```php
final class MigrationDiscoverySession
{
    // operation-scoped mutable state
}
```

Puede contener:

```text
visited sources
candidate count
descriptor index
diagnostics
collision candidates
budget counters
```

---

# 106. CatalogBuilder

```php
final class MigrationCatalogBuilder
{
    public function add(
        MigrationDescriptor $descriptor
    ): void;

    public function seal(): MigrationCatalog;
}
```

Es mutable únicamente durante discovery.

---

# 107. Catalog sealing

Lifecycle:

```text
CREATED
   ↓
COLLECTING
   ↓
VALIDATING
   ↓
CANONICALIZING
   ↓
SEALED
```

Después:

```text
MigrationCatalog immutable
```

---

# 108. Catalog validation

Antes de sealing:

```text
identity uniqueness
namespace validity
source validity
descriptor validity
definition reference validity
catalog completeness
```

---

# 109. Descriptor set

```php
final readonly class MigrationDescriptorSet
{
    // immutable deterministic collection
}
```

---

# 110. Definition loading boundary

Después de discovery:

```text
MigrationCatalog
       ↓
descriptor lookup
       ↓
MigrationDefinitionLoader
       ↓
MigrationDefinition
```

Esto puede ocurrir durante:

```text
planning
inspection
validation
CLI tooling
```

---

# 111. Discovery does not eagerly instantiate

Regla:

> El sistema no deberá instanciar todas las migraciones únicamente para saber que existen si dispone de información suficiente para construir sus descriptors.

---

# 112. Definition loading cache

Podrá existir posteriormente un cache de:

```text
immutable MigrationDefinition
```

pero no pertenece necesariamente a Discovery.

---

# 113. Discovery and Repository

Separación:

```text
Discovery:
"What migrations exist?"
```

```text
Repository:
"What migrations were recorded as applied?"
```

---

# 114. Pending calculation

Solo puede surgir al combinar:

```text
Available Migration Catalog
+
Repository Snapshot
+
Planning Rules
```

---

# 115. Formula

```text
Pending
≠
Catalog - Repository
```

como simple diferencia de arrays.

Debe considerar:

```text
checksums
dependencies
repository consistency
missing definitions
catalog completeness
target
namespace
```

---

# 116. Missing applied migration

Supongamos Repository:

```text
M1
M2
M3
```

pero Catalog:

```text
M1
M3
```

Discovery reporta lo que existe.

No decide si:

```text
M2 was deleted
source missing
catalog partial
package disabled
```

---

# 117. Repository orphan

El Planner/Repository reconciliation podrá clasificar:

```text
APPLIED_BUT_DEFINITION_MISSING
```

pero Discovery no inventa explicación.

---

# 118. New migration

Catalog:

```text
M1 M2 M3 M4
```

Repository:

```text
M1 M2 M3
```

M4 puede ser candidata pendiente.

La decisión final pertenece al Planner.

---

# 119. Discovery and checksum

Discovery puede proporcionar:

```text
source checksum
descriptor checksum
```

pero el semantic migration checksum requiere:

```text
MigrationDefinition
```

si depende de las operaciones.

---

# 120. Lazy semantic checksum

Por tanto:

```text
Discovery Descriptor
      ↓
source fingerprint
```

mientras:

```text
Definition Loading
      ↓
semantic checksum
```

---

# 121. Descriptor source checksum

Sirve para:

```text
cache invalidation
source change detection
manifest validation
```

No sustituye al repository semantic checksum.

---

# 122. Source change after discovery

Si el descriptor se descubrió y el archivo cambia antes de definition loading:

```text
TOCTOU
```

VoltStack deberá poder detectarlo mediante source fingerprint/checksum cuando el profile lo requiera.

---

# 123. TOCTOU mitigation

Proceso:

```text
Discover
   ↓
SourceFingerprint A
   ↓
Load Definition
   ↓
SourceFingerprint B
   ↓
A == B ?
```

Si no:

```text
MigrationSourceChangedException
```

en modo estricto.

---

# 124. Production compiled artifacts

La estrategia más fuerte:

```text
Build/Deploy
   ↓
Compile migration catalog
   ↓
Freeze deployment artifact
   ↓
Runtime uses immutable catalog
```

reduce problemas TOCTOU.

---

# 125. Security model

Discovery deberá proteger contra:

```text
path traversal
symlink loops
unexpected executable files
namespace spoofing
source collision
manifest tampering
unbounded directory traversal
untrusted extension scanners
diagnostic secret leakage
```

---

# 126. Path traversal

Una source root:

```text
/project/database/migrations
```

no deberá permitir candidate paths que escapen:

```text
../../secrets.php
```

salvo source explícitamente autorizada.

---

# 127. Canonical paths

Filesystem adapter deberá comparar:

```text
canonical root
canonical candidate
```

bajo policy adecuada.

---

# 128. Hidden files

Default recomendado:

```text
ignore hidden files
```

salvo configuración explícita.

---

# 129. File extensions

Default:

```text
.php
```

para migraciones PHP.

No cargar arbitrariamente:

```text
.tmp
.bak
.txt
```

---

# 130. Backup files

Ejemplo:

```text
2026_..._create_users.php.bak
```

debe ignorarse.

---

# 131. Duplicate filename ≠ duplicate identity necesariamente

En namespaces distintos:

```text
app:create_users
package.foo:create_users
```

pueden coexistir.

---

# 132. Same filename within same namespace

No implica automáticamente colisión si IDs explícitos son diferentes.

La identidad canónica manda.

---

# 133. Filename order

Nunca usar:

```text
filesystem iterator order
```

como orden semántico.

---

# 134. Filesystem nondeterminism

Los filesystems pueden devolver:

```text
A B C
```

o:

```text
C A B
```

Por tanto el resultado debe canonicalizarse antes del sealing.

---

# 135. Determinism equation

Debe cumplirse:

```text
Discover(Sources, Context)
=
EquivalentCatalog
```

independientemente del orden físico de enumeración.

---

# 136. Catalog determinism

Para mismas:

```text
sources
source contents
discovery profile
extensions
algorithm version
```

debe obtenerse:

```text
same descriptors
same canonical order
same fingerprint
```

---

# 137. Parallel discovery

Fuentes independientes podrán descubrirse concurrentemente.

```text
Source A ─┐
Source B ─┼→ Catalog Builder
Source C ─┘
```

pero la concurrencia no podrá afectar:

```text
catalog order
collision results
fingerprint
diagnostics semantics
```

---

# 138. Concurrent source scanning

Podrá utilizarse en:

```text
CLI
build process
large modular applications
```

si el runtime lo soporta.

---

# 139. Concurrency safety

Los resultados deberán combinarse mediante:

```text
deterministic merge
```

no mediante orden de finalización.

---

# 140. Extension architecture

Se propone:

```text
MigrationDiscoveryExtension
```

para:

```text
custom source kinds
custom candidate scanners
custom descriptor extractors
custom manifests
custom metadata
```

---

# 141. Frozen extension registry

```text
MigrationDiscoveryExtensionRegistry
      ↓ freeze
FrozenMigrationDiscoveryExtensionRegistry
```

---

# 142. Extension IDs

Cada extensión deberá tener:

```text
vendor namespace
extension ID
version
```

---

# 143. Extension collision

Dos extensions intentando manejar el mismo candidate con prioridad ambigua deberán fallar o resolverse mediante reglas explícitas.

Nunca:

```text
last wins
```

---

# 144. Extension restrictions

Una discovery extension no podrá:

- ejecutar migraciones;
- actualizar repository;
- abrir una transaction;
- alterar el schema;
- crear hidden migration identities;
- ocultar identity collisions;
- desactivar budgets;
- registrar sources después del freeze.

---

# 145. Discovery event integration

El sistema podrá emitir eventos conceptuales:

```text
MigrationDiscoveryStarted
MigrationSourceDiscoveryStarted
MigrationSourceDiscovered
MigrationDescriptorDiscovered
MigrationDiscoveryCompleted
MigrationDiscoveryFailed
```

pero mediante integración futura con Event System.

---

# 146. Events ≠ mutable hooks

Los listeners no deberán poder modificar silenciosamente el catálogo canónico.

Extensibilidad estructural debe pasar por registries oficiales.

---

# 147. Telemetry

Métricas posibles:

```text
migration.discovery.duration
migration.discovery.sources
migration.discovery.candidates
migration.discovery.descriptors
migration.discovery.cache_hits
migration.discovery.cache_misses
migration.discovery.collisions
migration.discovery.failures
```

---

# 148. Telemetry safety

No registrar por defecto:

```text
absolute private paths
source code
migration contents
credentials
raw configuration secrets
```

---

# 149. Path redaction

Preferir:

```text
database/migrations/...
```

sobre:

```text
/home/customer/private/project/...
```

en telemetry externa.

---

# 150. Performance model

Para:

```text
S = number of sources
C = number of candidates
D = number of descriptors
```

objetivo aproximado:

```text
Discovery ≈ O(S + C + D)
```

más costos específicos de filesystem/manifest.

---

# 151. Duplicate detection complexity

Con hash index:

```text
O(D)
```

promedio.

No:

```text
O(D²)
```

comparando cada migration contra todas.

---

# 152. Namespace indexing

```text
Map<MigrationNamespace, DescriptorSet>
```

permite consultas eficientes.

---

# 153. Identity index

```text
Map<MigrationIdentity, MigrationDescriptor>
```

será el índice principal del catálogo.

---

# 154. Streaming discovery

Candidates pueden procesarse incrementalmente:

```text
Scanner
  ↓ candidate
Extractor
  ↓ descriptor
Validator
  ↓
CatalogBuilder
```

sin mantener todos los candidates en memoria.

---

# 155. Catalog memory

Solo los descriptors finales necesitan permanecer en memoria.

Esto favorece proyectos con muchas migraciones.

---

# 156. Diagnostics budget

Los diagnostics también deben limitarse.

Si existen 100,000 archivos inválidos:

```text
do not create 100,000 large diagnostic objects
```

Se puede:

```text
retain first N
+
summary counts
```

sin ocultar que hubo más.

---

# 157. Testing architecture

Pruebas mínimas:

```text
source registration
source freeze
duplicate source detection
filesystem scanning
manifest scanning
candidate extraction
descriptor extraction
identity resolution
namespace validation
collision detection
catalog canonicalization
catalog fingerprinting
partial discovery
budget enforcement
cancellation
symlink loops
path traversal
cache invalidation
TOCTOU
parallel discovery determinism
persistent runtime isolation
extension conflicts
```

---

# 158. Unit test — collision

```php
$this->expectException(
    DuplicateMigrationIdentityException::class
);
```

para:

```text
Source A → app:M1
Source B → app:M1
```

---

# 159. Unit test — namespace isolation

Debe aceptar:

```text
app:M1
package.foo:M1
```

---

# 160. Unit test — nondeterministic filesystem

Input:

```text
Run 1:
C A B

Run 2:
B C A
```

Output:

```text
A B C
```

canónico en ambos casos.

---

# 161. Unit test — partial discovery

```text
Source A success
Source B failure
```

debe producir:

```text
PARTIAL
```

o error según policy.

Nunca `COMPLETE`.

---

# 162. Unit test — cache

Mismas source fingerprints:

```text
cache hit
```

Cambio en una source:

```text
cache invalidated
```

---

# 163. Unit test — source changed

```text
Discover source checksum A
Modify file
Load definition
```

en strict mode:

```text
MigrationSourceChangedException
```

---

# 164. Property-based testing

Generar:

```text
random source ordering
random namespace combinations
random migration IDs
duplicate patterns
invalid paths
deep directory trees
```

y verificar invariantes.

---

# 165. Persistent runtime testing

Ejecutar discovery concurrentemente:

```text
Session A
Session B
Session C
```

y comprobar:

```text
no candidate leakage
no diagnostic leakage
no catalog mutation
no namespace leakage
```

---

# 166. Error hierarchy

Se propone:

```text
DatabaseMigrationDiscoveryException
├── MigrationSourceException
│   ├── InvalidMigrationSourceException
│   ├── DuplicateMigrationSourceException
│   ├── MigrationSourceUnavailableException
│   └── MigrationSourceChangedException
│
├── MigrationCandidateException
│   ├── InvalidMigrationCandidateException
│   └── UnsupportedMigrationCandidateException
│
├── MigrationDescriptorException
│   ├── MigrationDescriptorExtractionException
│   └── InvalidMigrationDescriptorException
│
├── MigrationIdentityDiscoveryException
│   ├── MigrationIdentityResolutionException
│   ├── DuplicateMigrationIdentityException
│   └── MigrationNamespaceViolationException
│
├── MigrationManifestException
│   ├── InvalidMigrationManifestException
│   └── StaleMigrationManifestException
│
├── MigrationCatalogException
│   ├── InvalidMigrationCatalogException
│   └── IncompleteMigrationCatalogException
│
├── MigrationDiscoveryCacheException
├── MigrationDiscoveryExtensionException
├── MigrationDiscoveryBudgetExceededException
├── MigrationDiscoveryCancelledException
└── MigrationDiscoveryInvariantException
```

---

# 167. Namespace propuesto

```text
VoltStack\Quantum\Database\Migration\Discovery
```

---

# 168. Estructura de directorios propuesta

```text
Migration/
└── Discovery/
    ├── Contract/
    │   ├── MigrationDiscoverySystem.php
    │   ├── MigrationSource.php
    │   ├── MigrationCandidateScanner.php
    │   ├── MigrationDescriptorExtractor.php
    │   └── MigrationIdentityResolver.php
    │
    ├── Core/
    │   ├── DefaultMigrationDiscoverySystem.php
    │   ├── MigrationDiscoveryContext.php
    │   ├── MigrationDiscoverySession.php
    │   ├── MigrationDiscoveryProfile.php
    │   └── MigrationDiscoveryCompleteness.php
    │
    ├── Source/
    │   ├── MigrationSourceId.php
    │   ├── MigrationSourceKind.php
    │   ├── MigrationSourceRegistry.php
    │   ├── FrozenMigrationSourceRegistry.php
    │   ├── ApplicationMigrationSource.php
    │   ├── FrameworkMigrationSource.php
    │   ├── PackageMigrationSource.php
    │   ├── ModuleMigrationSource.php
    │   ├── PluginMigrationSource.php
    │   └── InMemoryMigrationSource.php
    │
    ├── Candidate/
    │   ├── MigrationCandidate.php
    │   ├── MigrationCandidateReference.php
    │   ├── MigrationCandidateMetadata.php
    │   └── MigrationSourceArtifactId.php
    │
    ├── Scanner/
    │   ├── FilesystemMigrationScanner.php
    │   ├── ManifestMigrationScanner.php
    │   └── InMemoryMigrationScanner.php
    │
    ├── Descriptor/
    │   ├── MigrationDescriptorExtractorRegistry.php
    │   ├── PhpMigrationDescriptorExtractor.php
    │   ├── ManifestMigrationDescriptorExtractor.php
    │   └── MigrationDefinitionReference.php
    │
    ├── Identity/
    │   ├── DefaultMigrationIdentityResolver.php
    │   ├── MigrationNamespacePolicy.php
    │   ├── MigrationNamespaceDecision.php
    │   └── MigrationIdentityIndex.php
    │
    ├── Collision/
    │   ├── MigrationCollisionDetector.php
    │   └── MigrationCollision.php
    │
    ├── Catalog/
    │   ├── MigrationCatalog.php
    │   ├── MigrationCatalogBuilder.php
    │   ├── MigrationDescriptorSet.php
    │   ├── MigrationCatalogMetadata.php
    │   └── MigrationCatalogFingerprint.php
    │
    ├── Manifest/
    │   ├── MigrationManifest.php
    │   ├── MigrationManifestLoader.php
    │   ├── MigrationManifestValidator.php
    │   └── MigrationManifestFingerprint.php
    │
    ├── Cache/
    │   ├── MigrationDiscoveryCache.php
    │   ├── MigrationDiscoveryCacheKey.php
    │   └── MigrationSourceFingerprint.php
    │
    ├── Filter/
    │   ├── MigrationDiscoveryFilter.php
    │   └── MigrationDiscoveryFilterPipeline.php
    │
    ├── Budget/
    │   ├── MigrationDiscoveryBudget.php
    │   └── MigrationDiscoveryBudgetTracker.php
    │
    ├── Diagnostic/
    │   ├── MigrationDiscoveryDiagnostic.php
    │   ├── MigrationDiscoveryDiagnosticBag.php
    │   └── MigrationDiscoveryDiagnosticSeverity.php
    │
    ├── Extension/
    │   ├── MigrationDiscoveryExtension.php
    │   ├── MigrationDiscoveryExtensionRegistry.php
    │   └── FrozenMigrationDiscoveryExtensionRegistry.php
    │
    └── Exception/
        ├── DatabaseMigrationDiscoveryException.php
        ├── InvalidMigrationSourceException.php
        ├── DuplicateMigrationSourceException.php
        ├── MigrationSourceUnavailableException.php
        ├── MigrationSourceChangedException.php
        ├── InvalidMigrationCandidateException.php
        ├── MigrationDescriptorExtractionException.php
        ├── InvalidMigrationDescriptorException.php
        ├── MigrationIdentityResolutionException.php
        ├── DuplicateMigrationIdentityException.php
        ├── MigrationNamespaceViolationException.php
        ├── InvalidMigrationManifestException.php
        ├── StaleMigrationManifestException.php
        ├── InvalidMigrationCatalogException.php
        ├── IncompleteMigrationCatalogException.php
        ├── MigrationDiscoveryExtensionException.php
        ├── MigrationDiscoveryBudgetExceededException.php
        ├── MigrationDiscoveryCancelledException.php
        └── MigrationDiscoveryInvariantException.php
```

---

# 169. Flujo completo

```text
                     Framework Bootstrap
                            │
                            ▼
                Migration Source Registry
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
     Application         Packages          Modules
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                   Freeze Source Registry
                            │
                            ▼
                  MigrationDiscoverySystem
                            │
                            ▼
                     Source Enumeration
                            │
                            ▼
                     Candidate Scanning
                            │
                            ▼
                   Descriptor Extraction
                            │
                            ▼
                    Identity Resolution
                            │
                            ▼
                   Namespace Validation
                            │
                            ▼
                    Collision Detection
                            │
                            ▼
                  Descriptor Validation
                            │
                            ▼
                  Canonical Catalog Build
                            │
                            ▼
                     Catalog Sealing
                            │
                            ▼
                    MigrationCatalog
```

---

# 170. Flujo posterior

Discovery termina aquí:

```text
MigrationCatalog
```

Después:

```text
                    MigrationCatalog
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
     Definition Loader           Migration Repository
             │                           │
             ▼                           ▼
    MigrationDefinitions       Applied Migration State
             │                           │
             └─────────────┬─────────────┘
                           ▼
                  Migration Planner
                           │
                           ▼
                    Migration Plan
```

---

# 171. Relación con Repository

El próximo sistema deberá poder contestar:

```text
Which migration identities have been applied?
```

Discovery responde únicamente:

```text
Which migration identities are currently available?
```

Formalmente:

```text
Available = Discover(Sources)

Applied = RepositorySnapshot(Database)
```

Solo después:

```text
Plan = MigrationPlanner(
    Available,
    Applied,
    Definitions,
    Policies
)
```

---

# 172. Invariantes del sistema

## DB-MIGRATION-DISCOVERY-001
Discovery descubrirá migraciones disponibles.

## DB-MIGRATION-DISCOVERY-002
Discovery no determinará por sí mismo migraciones pendientes.

## DB-MIGRATION-DISCOVERY-003
Discovery no actualizará Migration Repository.

## DB-MIGRATION-DISCOVERY-004
Discovery no ejecutará migraciones.

## DB-MIGRATION-DISCOVERY-005
Discovery no realizará rollback.

## DB-MIGRATION-DISCOVERY-006
Discovery no abrirá transacciones.

## DB-MIGRATION-DISCOVERY-007
Discovery no adquirirá migration locks de ejecución.

## DB-MIGRATION-DISCOVERY-008
Discovery no compilará Schema AST a DDL.

## DB-MIGRATION-DISCOVERY-009
Discovery no realizará schema introspection.

## DB-MIGRATION-DISCOVERY-010
MigrationSource será first-class.

## DB-MIGRATION-DISCOVERY-011
MigrationSourceId será distinto de MigrationNamespace.

## DB-MIGRATION-DISCOVERY-012
Source registration ocurrirá durante bootstrap.

## DB-MIGRATION-DISCOVERY-013
Source registry será frozen después de bootstrap.

## DB-MIGRATION-DISCOVERY-014
Duplicate Source IDs serán errores.

## DB-MIGRATION-DISCOVERY-015
No existirá last-wins source registration.

## DB-MIGRATION-DISCOVERY-016
Source priority no resolverá identity collisions.

## DB-MIGRATION-DISCOVERY-017
Source order será distinto de execution order.

## DB-MIGRATION-DISCOVERY-018
Candidate será distinto de Descriptor.

## DB-MIGRATION-DISCOVERY-019
Descriptor será distinto de Definition.

## DB-MIGRATION-DISCOVERY-020
Candidate podrá ser lightweight.

## DB-MIGRATION-DISCOVERY-021
Descriptor extraction evitará definition execution cuando sea posible.

## DB-MIGRATION-DISCOVERY-022
MigrationDescriptor será immutable.

## DB-MIGRATION-DISCOVERY-023
Descriptor conservará source traceability.

## DB-MIGRATION-DISCOVERY-024
Descriptor podrá referenciar lazy definition loading.

## DB-MIGRATION-DISCOVERY-025
Filename será distinto de canonical migration identity.

## DB-MIGRATION-DISCOVERY-026
Identity resolution será determinista.

## DB-MIGRATION-DISCOVERY-027
Random migration identities estarán prohibidas.

## DB-MIGRATION-DISCOVERY-028
Explicit identity tendrá prioridad sobre filename-derived identity.

## DB-MIGRATION-DISCOVERY-029
Migration identity será namespace-aware.

## DB-MIGRATION-DISCOVERY-030
Namespace assignment será validada.

## DB-MIGRATION-DISCOVERY-031
Package migrations no escaparán silenciosamente de su namespace.

## DB-MIGRATION-DISCOVERY-032
Identity normalization no alterará semántica silenciosamente.

## DB-MIGRATION-DISCOVERY-033
Duplicate qualified identities serán errores.

## DB-MIGRATION-DISCOVERY-034
Duplicate identity no utilizará first-wins.

## DB-MIGRATION-DISCOVERY-035
Duplicate identity no utilizará last-wins.

## DB-MIGRATION-DISCOVERY-036
Application migrations no sobrescribirán package migrations.

## DB-MIGRATION-DISCOVERY-037
Artifact identity será distinta de migration identity.

## DB-MIGRATION-DISCOVERY-038
Filesystem traversal será bounded.

## DB-MIGRATION-DISCOVERY-039
Symlink traversal será policy-driven.

## DB-MIGRATION-DISCOVERY-040
Symlink loops serán detectados o bounded.

## DB-MIGRATION-DISCOVERY-041
Candidate scanning no ejecutará migraciones.

## DB-MIGRATION-DISCOVERY-042
Filesystem scanner no requerirá todos los PHP files innecesariamente.

## DB-MIGRATION-DISCOVERY-043
Manifest discovery será soportable.

## DB-MIGRATION-DISCOVERY-044
Compiled catalog será distinto de Migration Repository.

## DB-MIGRATION-DISCOVERY-045
Discovery cache no almacenará live connections.

## DB-MIGRATION-DISCOVERY-046
Discovery cache no almacenará transaction state.

## DB-MIGRATION-DISCOVERY-047
Discovery cache no almacenará execution state.

## DB-MIGRATION-DISCOVERY-048
Cache keys serán versionadas.

## DB-MIGRATION-DISCOVERY-049
Source fingerprint será distinto de migration semantic checksum.

## DB-MIGRATION-DISCOVERY-050
MigrationCatalog será immutable.

## DB-MIGRATION-DISCOVERY-051
MigrationCatalog será deterministic.

## DB-MIGRATION-DISCOVERY-052
MigrationCatalog será collision-free.

## DB-MIGRATION-DISCOVERY-053
MigrationCatalog será namespace-aware.

## DB-MIGRATION-DISCOVERY-054
MigrationCatalog tendrá canonical ordering.

## DB-MIGRATION-DISCOVERY-055
Catalog ordering será distinto de migration execution ordering.

## DB-MIGRATION-DISCOVERY-056
Timestamp ordering será distinto de dependency truth.

## DB-MIGRATION-DISCOVERY-057
Discovery profiles serán explícitos.

## DB-MIGRATION-DISCOVERY-058
Production podrá utilizar compiled manifests.

## DB-MIGRATION-DISCOVERY-059
Testing podrá utilizar in-memory sources.

## DB-MIGRATION-DISCOVERY-060
DiscoveryContext no contendrá PDO.

## DB-MIGRATION-DISCOVERY-061
DiscoveryContext no contendrá live Connection.

## DB-MIGRATION-DISCOVERY-062
DiscoveryContext no contendrá Transaction.

## DB-MIGRATION-DISCOVERY-063
DiscoveryContext no dependerá de current tenant global.

## DB-MIGRATION-DISCOVERY-064
Discovery soportará cancellation.

## DB-MIGRATION-DISCOVERY-065
Discovery tendrá budgets.

## DB-MIGRATION-DISCOVERY-066
Budget exhaustion no producirá catálogo silenciosamente truncado.

## DB-MIGRATION-DISCOVERY-067
Discovery completeness será first-class.

## DB-MIGRATION-DISCOVERY-068
PARTIAL será distinto de COMPLETE.

## DB-MIGRATION-DISCOVERY-069
UNKNOWN será distinto de COMPLETE.

## DB-MIGRATION-DISCOVERY-070
Failed required source impedirá declarar COMPLETE.

## DB-MIGRATION-DISCOVERY-071
Incomplete catalog no deberá tratarse como pending set completo.

## DB-MIGRATION-DISCOVERY-072
Diagnostics serán bounded.

## DB-MIGRATION-DISCOVERY-073
Recoverable diagnostics serán distintos de fatal correctness errors.

## DB-MIGRATION-DISCOVERY-074
Lenient mode no ocultará identity collisions.

## DB-MIGRATION-DISCOVERY-075
Lenient mode no ocultará security violations.

## DB-MIGRATION-DISCOVERY-076
Discovery minimizará ejecución de PHP source.

## DB-MIGRATION-DISCOVERY-077
Static/manifest metadata será preferible cuando esté disponible.

## DB-MIGRATION-DISCOVERY-078
Discovery no asumirá filesystem como única source.

## DB-MIGRATION-DISCOVERY-079
Composer integration será adapter, no core assumption.

## DB-MIGRATION-DISCOVERY-080
Package auto-discovery será desacoplado del core.

## DB-MIGRATION-DISCOVERY-081
Runtime source registration después de freeze será rechazada.

## DB-MIGRATION-DISCOVERY-082
Shared source registry será immutable.

## DB-MIGRATION-DISCOVERY-083
MigrationDiscoverySession será operation-scoped.

## DB-MIGRATION-DISCOVERY-084
Candidate buffers no se compartirán entre sessions.

## DB-MIGRATION-DISCOVERY-085
Diagnostic buffers no se compartirán entre sessions.

## DB-MIGRATION-DISCOVERY-086
CatalogBuilder será mutable solo antes de sealing.

## DB-MIGRATION-DISCOVERY-087
Sealed catalog no podrá modificarse.

## DB-MIGRATION-DISCOVERY-088
Definition loading será separable de discovery.

## DB-MIGRATION-DISCOVERY-089
Discovery no cargará todas las definitions sin necesidad.

## DB-MIGRATION-DISCOVERY-090
Discovery será distinto de Repository reconciliation.

## DB-MIGRATION-DISCOVERY-091
Pending no será calculado como simple array subtraction.

## DB-MIGRATION-DISCOVERY-092
Missing applied definition no será explicado automáticamente por Discovery.

## DB-MIGRATION-DISCOVERY-093
Descriptor checksum será distinto de semantic migration checksum.

## DB-MIGRATION-DISCOVERY-094
Source fingerprints podrán utilizarse para cache invalidation.

## DB-MIGRATION-DISCOVERY-095
Source changes entre discovery y loading podrán detectarse.

## DB-MIGRATION-DISCOVERY-096
TOCTOU policy será explícita.

## DB-MIGRATION-DISCOVERY-097
Compiled deployment artifacts podrán reducir TOCTOU.

## DB-MIGRATION-DISCOVERY-098
Path traversal será prevenido.

## DB-MIGRATION-DISCOVERY-099
Candidate paths deberán respetar source root policy.

## DB-MIGRATION-DISCOVERY-100
Hidden file behavior será explícito.

## DB-MIGRATION-DISCOVERY-101
Allowed file extensions serán explícitas.

## DB-MIGRATION-DISCOVERY-102
Backup files no serán cargados como migraciones por default.

## DB-MIGRATION-DISCOVERY-103
Same filename en namespaces diferentes podrá ser válido.

## DB-MIGRATION-DISCOVERY-104
Filesystem enumeration order no será semántico.

## DB-MIGRATION-DISCOVERY-105
Discovery canonicalizará resultados.

## DB-MIGRATION-DISCOVERY-106
Equivalent source sets producirán equivalent catalogs.

## DB-MIGRATION-DISCOVERY-107
Parallel discovery no cambiará resultado.

## DB-MIGRATION-DISCOVERY-108
Parallel discovery utilizará deterministic merge.

## DB-MIGRATION-DISCOVERY-109
Extension registry será frozen.

## DB-MIGRATION-DISCOVERY-110
Extension handlers tendrán IDs versionados.

## DB-MIGRATION-DISCOVERY-111
Ambiguous extension handlers no usarán last-wins.

## DB-MIGRATION-DISCOVERY-112
Extensions no ejecutarán migrations.

## DB-MIGRATION-DISCOVERY-113
Extensions no actualizarán repository.

## DB-MIGRATION-DISCOVERY-114
Extensions no alterarán schema.

## DB-MIGRATION-DISCOVERY-115
Extensions no ocultarán collisions.

## DB-MIGRATION-DISCOVERY-116
Extensions no desactivarán budgets.

## DB-MIGRATION-DISCOVERY-117
Events no actuarán como hidden mutation hooks.

## DB-MIGRATION-DISCOVERY-118
Telemetry no expondrá private paths por default.

## DB-MIGRATION-DISCOVERY-119
Telemetry no expondrá migration source code.

## DB-MIGRATION-DISCOVERY-120
Identity indexing buscará complejidad aproximadamente O(D).

## DB-MIGRATION-DISCOVERY-121
Collision detection evitará comparación O(D²) innecesaria.

## DB-MIGRATION-DISCOVERY-122
Candidate processing podrá ser streaming.

## DB-MIGRATION-DISCOVERY-123
Discovery no requerirá retener todos los candidates.

## DB-MIGRATION-DISCOVERY-124
Catalog fingerprint será deterministic.

## DB-MIGRATION-DISCOVERY-125
Catalog fingerprint incluirá algorithm/version context relevante.

## DB-MIGRATION-DISCOVERY-126
Catalog fingerprint no incluirá runtime object IDs.

## DB-MIGRATION-DISCOVERY-127
Discovery errors serán typed.

## DB-MIGRATION-DISCOVERY-128
Source failures serán distinguibles de descriptor failures.

## DB-MIGRATION-DISCOVERY-129
Descriptor failures serán distinguibles de identity collisions.

## DB-MIGRATION-DISCOVERY-130
Cancellation será distinguible de failure.

## DB-MIGRATION-DISCOVERY-131
Budget exhaustion será distinguible de cancellation.

## DB-MIGRATION-DISCOVERY-132
MigrationSourceChanged será distinguible de invalid source.

## DB-MIGRATION-DISCOVERY-133
Catalog validation ocurrirá antes de sealing.

## DB-MIGRATION-DISCOVERY-134
Catalog completeness será conservada en metadata.

## DB-MIGRATION-DISCOVERY-135
Catalog consumers podrán conocer si el catálogo es parcial.

## DB-MIGRATION-DISCOVERY-136
Discovery no inventará descriptors para fuentes fallidas.

## DB-MIGRATION-DISCOVERY-137
Discovery no inventará migration identity cuando no pueda resolverla.

## DB-MIGRATION-DISCOVERY-138
Discovery no inferirá applied state.

## DB-MIGRATION-DISCOVERY-139
Discovery no inferirá rollback state.

## DB-MIGRATION-DISCOVERY-140
Discovery no inferirá batch state.

## DB-MIGRATION-DISCOVERY-141
Discovery no inferirá migration safety.

## DB-MIGRATION-DISCOVERY-142
Discovery no inferirá platform compatibility final.

## DB-MIGRATION-DISCOVERY-143
Discovery no resolverá physical DB connection.

## DB-MIGRATION-DISCOVERY-144
Discovery no dependerá de schema state.

## DB-MIGRATION-DISCOVERY-145
FrankenPHP sessions estarán aisladas.

## DB-MIGRATION-DISCOVERY-146
RoadRunner sessions estarán aisladas.

## DB-MIGRATION-DISCOVERY-147
OpenSwoole coroutine sessions estarán aisladas.

## DB-MIGRATION-DISCOVERY-148
Discovery deberá poder probarse sin base de datos.

## DB-MIGRATION-DISCOVERY-149
Discovery deberá poder operar con fuentes totalmente in-memory.

## DB-MIGRATION-DISCOVERY-150
VoltStack tratará discovery como construcción determinista de un catálogo de migraciones disponibles, nunca como ejecución ni como historial.

---

# 173. Anti-patterns

## 173.1 Discovery consultando repository

Incorrecto:

```php
foreach ($files as $file) {
    if (!$repository->has($file)) {
        $pending[] = $file;
    }
}
```

Esto mezcla:

```text
Discovery + Repository + Pending Calculation
```

---

## 173.2 Ejecutar `up()` para descubrir ID

Incorrecto:

```php
$migration = require $file;
$migration->up();
```

---

## 173.3 Last migration wins

Incorrecto:

```php
$catalog[$id] = $descriptor;
```

sin verificar colisiones.

---

## 173.4 Filename como identidad global

Incorrecto:

```text
create_users.php
=
global migration identity
```

---

## 173.5 Filesystem order como execution order

Incorrecto:

```php
foreach (new DirectoryIterator($path) as $file) {
    execute($file);
}
```

---

## 173.6 Source priority para ocultar collision

Incorrecto:

```text
Application > Package
```

por lo tanto ignorar la package migration.

---

## 173.7 Global mutable registry

Incorrecto:

```php
MigrationSources::$sources[] = $source;
```

desde cualquier request.

---

## 173.8 Unbounded recursive scanning

Incorrecto:

```php
scanRecursively('/');
```

sin límites.

---

## 173.9 Cache con Migration objects mutables

Incorrecto:

```text
Discovery Cache
└── instantiated mutable migrations
```

---

## 173.10 Partial catalog marked complete

Incorrecto:

```text
Package source failed
→
return complete catalog anyway
```

---

# 174. Fórmula principal

```text
MigrationCatalog
=
Discover(
    FrozenMigrationSources,
    DiscoveryProfile,
    FrozenDiscoveryExtensions,
    DiscoveryBudget
)
```

---

# 175. Fórmula expandida

```text
MigrationCatalog
=
Seal(
    Canonicalize(
        Validate(
            DetectCollisions(
                ResolveIdentities(
                    ExtractDescriptors(
                        ScanCandidates(
                            Sources
                        )
                    )
                )
            )
        )
    )
)
```

---

# 176. Propiedad de determinismo

Para:

```text
S = same logical source set
C = same source contents
P = same discovery profile
E = same frozen extensions
V = same algorithm version
```

debe cumplirse:

```text
Discover(S, C, P, E, V)
=
EquivalentCatalog
```

---

# 177. Propiedad de completitud

```text
Catalog.Completeness = COMPLETE
```

solo si:

```text
∀ required source S:
Discovery(S) = SUCCESS
```

---

# 178. Propiedad de unicidad

Para catálogo válido:

```text
∀ A,B ∈ Catalog:

A ≠ B
⇒
Identity(A) ≠ Identity(B)
```

---

# 179. Propiedad de aislamiento

```text
DiscoverySession(A)
∩ mutable state
DiscoverySession(B)
=
∅
```

---

# 180. Relación conceptual final

```text
Sources
   │
   ▼
Discovery
   │
   ▼
MigrationCatalog
   │
   ├───────────────┐
   │               │
   ▼               ▼
Definitions     Repository
   │               │
   └───────┬───────┘
           ▼
        Planner
           │
           ▼
     Execution Plan
           │
           ▼
        Executor
```

---

# 181. Regla maestra

> **El Migration Discovery System responde qué migraciones están disponibles y de dónde provienen; nunca responde cuáles deben ejecutarse.**

En forma compacta:

```text
Discovery finds.
Definition describes.
Repository remembers.
Planner decides.
Executor executes.
```

---

# 182. Resultado arquitectónico

Con este diseño, VoltStack puede descubrir migraciones provenientes de:

```text
Application
Framework
Packages
Modules
Plugins
Generated artifacts
Testing sources
Future extensions
```

sin acoplar el núcleo a:

```text
filesystem
Composer
PDO
Migration Repository
specific database platform
execution runtime
```

La separación:

```text
MigrationSource
        ↓
MigrationCandidate
        ↓
MigrationDescriptor
        ↓
MigrationCatalog
```

permite además que producción utilice:

```text
Compiled Migration Catalog
```

mientras desarrollo puede utilizar:

```text
Dynamic Filesystem Discovery
```

sin modificar la arquitectura superior.

---

# 183. Siguiente documento

```text
104_DATABASE_MIGRATION_REPOSITORY_SYSTEM.md
```

El siguiente documento deberá definir el subsistema que conserva el **historial persistente de migraciones aplicadas**:

```text
Migration Catalog
        │
        │
        │         Database
        │            │
        │            ▼
        │    Migration Repository
        │            │
        │            ▼
        │     Repository Snapshot
        │            │
        └──────┬─────┘
               ▼
         Migration Planner
```

y establecerá distinciones críticas como:

```text
Migration Repository
≠
Migration Discovery

Repository Record
≠
Migration Definition

Applied Migration
≠
Available Migration

Migration Batch
≠
Transaction

Repository Checksum
≠
Source Checksum

Repository State
≠
Execution Attempt State

Missing Definition
≠
Rolled Back Migration
```

También deberá diseñar:

```text
migration repository schema
qualified migration identities
semantic checksums
batch references
application timestamps
execution metadata
repository snapshots
repository initialization
repository versioning
consistency validation
drift detection
missing definitions
concurrent repository access
persistent runtime safety
repository corruption handling
```