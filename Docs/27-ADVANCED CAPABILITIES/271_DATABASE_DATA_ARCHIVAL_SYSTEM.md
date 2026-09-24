# 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md

# VoltStack Quantum Database
## Data Archival System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 271 — Data Archival System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `270_DATABASE_DATA_RETENTION_SYSTEM.md`  
**Siguiente documento:** `272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Data Archival System** de VoltStack Database.

El sistema será responsable de mover, copiar o transformar datos que ya no requieren permanecer en el almacenamiento operacional primario hacia una capa de almacenamiento de archivo, preservando explícitamente las garantías necesarias de:

- identidad;
- integridad;
- trazabilidad;
- seguridad;
- clasificación;
- tenant;
- relaciones;
- historial;
- esquema;
- tipos;
- política de retención;
- recuperación;
- verificación.

La regla central será:

> **Archival mueve datos fuera de su almacenamiento operacional primario preservando las garantías necesarias para su conservación y recuperación; no es equivalente a Backup, Export, Soft Delete, Purge, History ni Retention.**

---

# 2. Objetivos

El Data Archival System deberá proporcionar:

- políticas declarativas de archival;
- selección segura de candidatos;
- integración con Data Retention;
- múltiples niveles de almacenamiento;
- almacenamiento local o remoto;
- soporte para Object Storage;
- paquetes de archivo;
- manifests;
- checksums;
- verificación criptográfica;
- cifrado;
- compresión;
- versionado de formato;
- procesamiento incremental;
- procesamiento por chunks;
- archivado por tenant;
- archivado por shard;
- archivado de relaciones;
- archivado de históricos;
- restauración;
- rehidratación;
- búsqueda mediante metadata;
- auditoría;
- telemetría;
- resiliencia;
- ejecución resumible;
- dry-run;
- seguridad en runtimes persistentes.

---

# 3. Distinciones fundamentales

VoltStack distinguirá:

```text
Archive
≠ Backup
≠ Export
≠ Retention
≠ Purge
≠ Soft Delete
≠ History
≠ Snapshot
≠ Cache
≠ Replica
```

---

# 4. Archive ≠ Backup

Un backup está orientado principalmente a:

```text
disaster recovery
database recovery
point-in-time recovery
operational restoration
```

Un archive está orientado a:

```text
long-term preservation
operational data reduction
compliance retention
historical access
cold storage
```

---

# 5. Ejemplo

Backup:

```text
PostgreSQL cluster
        ↓
full backup
        ↓
backup-2026-09-19
```

Archive:

```text
Invoices 2020
      ↓
Archive Policy
      ↓
Archive Package
      ↓
Cold Object Storage
```

---

# 6. Archive ≠ Export

Export responde:

> ¿Cómo representamos datos fuera de VoltStack?

Archive responde:

> ¿Cómo preservamos datos fuera del almacenamiento operacional manteniendo su lifecycle, integridad y recuperabilidad?

Un CSV exportado manualmente no constituye automáticamente un archive válido.

---

# 7. Archive ≠ Retention

Retention determina:

```text
cuánto tiempo conservar
```

Archive determina:

```text
dónde y cómo conservar
```

---

# 8. Archive ≠ Purge

Archivar no implica necesariamente eliminar el original.

Podrá existir:

```text
COPY_ONLY
```

o:

```text
ARCHIVE_THEN_PURGE
```

---

# 9. Archive ≠ Soft Delete

Una entidad soft-deleted continúa normalmente en la base operacional.

Una entidad archived puede dejar de existir físicamente en ella.

---

# 10. Archive ≠ History

History conserva cambios/versiones.

Archive conserva datasets fuera de su ubicación operacional primaria.

El History System podrá ser una fuente de datos para Archival.

---

# 11. Principio arquitectónico

```text
Operational Data
       │
       ▼
Archive Candidate
       │
       ▼
Archive Policy
       │
       ▼
Archive Planner
       │
       ▼
Archive Package
       │
       ▼
Archive Storage
       │
       ▼
Verification
       │
       ▼
Archive Commit
       │
       ├── KEEP SOURCE
       │
       └── PURGE SOURCE
```

---

# 12. Archive lifecycle

El ciclo conceptual será:

```text
ACTIVE
   │
   ▼
ARCHIVE_ELIGIBLE
   │
   ▼
ARCHIVE_PLANNED
   │
   ▼
ARCHIVING
   │
   ▼
ARCHIVE_WRITTEN
   │
   ▼
VERIFYING
   │
   ▼
ARCHIVED
   │
   ├── source retained
   │
   └── source purge eligible
```

---

# 13. Archive states

```php
enum ArchiveState
{
    case ACTIVE;
    case ELIGIBLE;
    case PLANNED;
    case ARCHIVING;
    case WRITTEN;
    case VERIFYING;
    case VERIFIED;
    case ARCHIVED;
    case FAILED;
    case PARTIAL;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 14. WRITTEN ≠ ARCHIVED

Que bytes hayan sido escritos en un destino no demuestra que el archive sea válido.

---

# 15. VERIFIED

Un archive podrá considerarse válido cuando:

```text
payload exists
+
manifest exists
+
integrity validated
+
required metadata persisted
+
required encryption validated
+
storage acknowledgement known
```

---

# 16. Archive Subject

La unidad lógica será:

```text
ArchiveSubject
```

Podrá representar:

```text
Entity
Entity Graph
Historical Versions
Dataset
Partition
Tenant Dataset
Shard Dataset
Audit Dataset
Custom Domain Dataset
```

---

# 17. Archive Subject ≠ ORM Entity

Un archive puede representar millones de registros.

Por tanto:

```text
ArchiveSubject
≠
Entity
```

---

# 18. Archive Policy

La política principal será:

```text
ArchivePolicy
```

---

# 19. Modelo conceptual

```php
final readonly class ArchivePolicy
{
    public function __construct(
        public ArchivePolicyId $id,
        public ArchivePolicyVersion $version,
        public ArchiveScope $scope,
        public ArchiveEligibilityPolicy $eligibility,
        public ArchiveTargetPolicy $target,
        public ArchiveFormatPolicy $format,
        public ArchiveVerificationPolicy $verification,
        public ArchiveSourceDisposition $sourceDisposition,
    ) {}
}
```

---

# 20. Stable policy ID

Ejemplos:

```text
financial-invoices-archive
customer-history-archive
security-audit-cold-storage
tenant-offboarding-archive
```

---

# 21. Policy version

```text
financial-invoices-archive:v1
financial-invoices-archive:v2
```

---

# 22. Archive scope

Podrá aplicar a:

```text
entity type
dataset
classification
tenant
historical versions
audit records
partition
domain
custom query
```

---

# 23. Archive eligibility

La elegibilidad deberá ser explícita.

Ejemplo:

```text
Invoices older than 24 months
AND status = CLOSED
AND retention allows archive
```

---

# 24. Candidate ≠ Eligible

Al igual que Retention:

```text
ArchiveCandidate
≠
ArchiveEligible
```

---

# 25. Archive Eligibility Evaluator

```php
interface ArchiveEligibilityEvaluator
{
    public function evaluate(
        ArchiveSubject $subject,
        ArchiveContext $context,
    ): ArchiveEligibilityResult;
}
```

---

# 26. Eligibility result

```php
enum ArchiveEligibilityDecision
{
    case ELIGIBLE;
    case NOT_ELIGIBLE;
    case BLOCKED;
    case UNKNOWN;
}
```

---

# 27. UNKNOWN

Puede resultar de:

```text
retention unknown
policy unavailable
tenant unavailable
classification unknown
relationship state incomplete
source consistency unknown
```

---

# 28. UNKNOWN ≠ ELIGIBLE

Para archival destructivo:

```text
UNKNOWN → BLOCK
```

por default.

---

# 29. Archive eligibility ≠ purge eligibility

Un dato puede ser:

```text
ARCHIVE_ELIGIBLE
```

pero:

```text
NOT_PURGE_ELIGIBLE
```

---

# 30. Example

```text
Invoice:
  age: 3 years

Archive after:
  2 years

Purge after:
  5 years
```

Entonces entre año 2 y 5:

```text
ARCHIVE_ELIGIBLE
RETAIN_REQUIRED
```

---

# 31. Storage tiers

VoltStack reconocerá conceptualmente:

```text
HOT
WARM
COLD
DEEP_ARCHIVE
```

---

# 32. HOT

Datos operacionales frecuentes.

Ejemplo:

```text
primary database
```

---

# 33. WARM

Datos menos frecuentes pero todavía accesibles con latencia relativamente baja.

---

# 34. COLD

Datos históricos con acceso ocasional.

---

# 35. DEEP_ARCHIVE

Datos con acceso muy infrecuente y recuperación potencialmente lenta.

---

# 36. Tier ≠ provider

```text
COLD
```

es una intención de almacenamiento.

No significa automáticamente:

```text
Amazon S3 Glacier
```

---

# 37. Provider abstraction

VoltStack definirá:

```php
interface ArchiveStorageProvider
{
    public function store(
        ArchiveObject $object,
        ArchiveStorageContext $context,
    ): ArchiveStorageReceipt;
}
```

---

# 38. Providers posibles

Podrán existir adaptadores para:

```text
Filesystem
S3-compatible Object Storage
AWS S3
DigitalOcean Spaces
MinIO
Google Cloud Storage
Azure Blob Storage
Custom Storage
```

El núcleo no dependerá directamente de ninguno.

---

# 39. ArchiveStorageProvider ≠ Filesystem abstraction

Aunque pueda reutilizar contratos del Filesystem System, Archival necesita semánticas adicionales:

```text
immutability
verification
manifest
retention
object identity
storage class
archive receipt
```

---

# 40. Archive Target

Una policy podrá resolver:

```text
provider
bucket/container
logical namespace
storage tier
region
encryption profile
retention profile
```

---

# 41. No credentials in policy

Una ArchivePolicy no deberá almacenar secretos directamente.

---

# 42. Credential resolution

Se utilizarán referencias como:

```text
archive-storage-primary
```

resueltas mediante el sistema seguro de configuración/credenciales.

---

# 43. Archive object

Unidad física almacenada:

```text
ArchiveObject
```

---

# 44. Archive package

Un archive lógico podrá contener múltiples objetos:

```text
ArchivePackage
├── manifest
├── data-00001
├── data-00002
├── data-00003
└── indexes
```

---

# 45. Package ≠ file

Un package puede contener:

```text
1
10
1000
```

objetos físicos.

---

# 46. Archive Package ID

Cada package tendrá:

```text
ArchivePackageId
```

globalmente estable dentro del dominio correspondiente.

---

# 47. Ejemplo

```text
arc_pkg_01J9X4...
```

---

# 48. Archive Manifest

Todo package deberá poder poseer:

```text
ArchiveManifest
```

---

# 49. Manifest purpose

Describe:

```text
what was archived
how
when
from where
under which policy
with which schema
with which integrity evidence
```

---

# 50. Manifest conceptual

```text
ArchiveManifest
├── formatVersion
├── packageId
├── createdAt
├── archivePolicy
├── source
├── tenant
├── shard
├── schema
├── objects
├── recordCount
├── integrity
├── encryption
├── compression
└── retention
```

---

# 51. Manifest ≠ payload

El manifest no deberá duplicar todo el contenido archivado.

---

# 52. Manifest sensitivity

Puede contener metadata sensible.

Por tanto deberá estar sujeto a:

```text
access control
encryption
retention
audit
```

---

# 53. Format Version

El formato de archive deberá ser versionado.

Ejemplo:

```text
voltstack-archive/1
```

---

# 54. Format version ≠ policy version

```text
ArchiveFormatVersion
≠
ArchivePolicyVersion
≠
SchemaVersion
```

---

# 55. Schema preservation

Un archive deberá conservar suficiente información para interpretar los datos posteriormente.

---

# 56. Schema descriptor

Podrá almacenar:

```text
entity metadata generation
logical field names
logical types
relationships
schema fingerprint
serialization format
```

---

# 57. Schema snapshot

Dependiendo de la policy podrá conservar:

```text
full schema snapshot
```

o:

```text
schema reference + fingerprint
```

---

# 58. Reference risk

Una referencia a metadata externa puede dejar de ser resoluble dentro de 10 años.

Para archival de largo plazo puede ser preferible una representación autosuficiente.

---

# 59. Self-describing archives

VoltStack deberá permitir archives suficientemente self-describing cuando la policy lo requiera.

---

# 60. Type preservation

El archive deberá preservar:

```text
VoltStack logical type
```

cuando sea necesario.

No únicamente el tipo físico del driver.

---

# 61. Example

```text
Money
Instant
UUID
Enum
JSON
ValueObject
```

deben conservar semántica suficiente.

---

# 62. Serialization format

El sistema podrá soportar distintos formatos mediante adapters.

Ejemplos conceptuales:

```text
JSON Lines
CSV
Parquet
binary canonical format
custom domain format
```

---

# 63. Core neutrality

El core no asumirá que:

```text
Archive = ZIP + JSON
```

---

# 64. Archive Encoder

```php
interface ArchiveEncoder
{
    public function encode(
        ArchiveRecordStream $records,
        ArchiveEncodingContext $context,
    ): ArchiveEncodedStream;
}
```

---

# 65. Encoder ≠ Query Engine

El encoder recibe datos ya obtenidos.

No construye consultas SQL.

---

# 66. Encoder ≠ Storage Provider

El encoder representa.

El provider almacena.

---

# 67. Streaming encoding

Para grandes datasets deberá evitarse:

```text
load 100 GB into memory
```

---

# 68. Pipeline

```text
Query
 ↓
Chunk/Stream
 ↓
Canonical Archive Record
 ↓
Encoder
 ↓
Compression
 ↓
Encryption
 ↓
Checksum
 ↓
Storage
```

---

# 69. Canonical Archive Record

Entre hydration/query y encoder podrá existir:

```text
CanonicalArchiveRecord
```

---

# 70. Why canonical layer

Evita acoplar:

```text
ORM Entity
```

directamente a:

```text
Parquet/JSON/CSV/etc.
```

---

# 71. Record identity

Cada record podrá conservar:

```text
EntityType
Identifier
Tenant
Version
Temporal coordinates
Classification
```

según policy.

---

# 72. Relationship archival

Archivar un entity graph requiere distinguir:

```text
ownership
references
independent lifecycle
retention
```

---

# 73. No blind cascade archival

No:

```text
archive Customer
→ recursively archive everything reachable
```

sin policy.

---

# 74. Relationship policies

Podrán existir:

```text
INCLUDE
REFERENCE
SEPARATE_ARCHIVE
EXCLUDE
BLOCK
CUSTOM
```

---

# 75. INCLUDE

La relación se incorpora al mismo package.

---

# 76. REFERENCE

Sólo se conserva una referencia estable.

---

# 77. SEPARATE_ARCHIVE

El target tiene su propio archive lifecycle.

---

# 78. EXCLUDE

No forma parte del archive.

---

# 79. BLOCK

El archive no puede completarse mientras la relación esté en ese estado.

---

# 80. Cycles

El graph planner deberá manejar ciclos.

Ejemplo:

```text
A → B → C → A
```

---

# 81. Archive Graph

Podrá existir:

```text
ArchiveGraph
```

distinto de:

```text
ORM Object Graph
Persistence Graph
```

---

# 82. Archive Graph Planner

Responsable de determinar:

```text
subjects
dependencies
ordering
references
package boundaries
```

---

# 83. Historical data

El sistema podrá archivar:

```text
current entity
history only
current + history
selected versions
```

---

# 84. History archive policy

Ejemplo:

```text
Current:
  operational DB

Versions older than 2 years:
  archive

Versions older than 7 years:
  purge
```

---

# 85. Temporal archives

Podrán conservar:

```text
valid_from
valid_to
system_from
system_to
```

---

# 86. Temporal coordinates ≠ archive timestamps

---

# 87. Source selection

Archive candidate discovery utilizará:

```text
Query Engine
```

---

# 88. No direct SQL

Archival no deberá generar:

```php
$sql = 'SELECT * FROM ...';
```

internamente como arquitectura principal.

---

# 89. Archive Query Plan

Podrá existir una especialización:

```text
ArchiveSourcePlan
```

que referencie Query Models normales.

---

# 90. Candidate query

Ejemplo conceptual:

```php
Invoice::query()
    ->where('status', InvoiceStatus::CLOSED)
    ->whereBefore('closedAt', $archiveThreshold);
```

---

# 91. Selection ≠ final eligibility

El query descubre candidatos.

El evaluator determina elegibilidad final.

---

# 92. Chunk Processing integration

Para grandes datasets:

```text
Archive Source Query
        ↓
Chunk Processing
        ↓
Canonical Records
        ↓
Archive Encoder
        ↓
Archive Storage
```

---

# 93. Chunk size

Será configurable mediante Resource Governance.

---

# 94. Chunk ≠ archive package

Un package puede contener varios chunks.

Un chunk puede contribuir a uno o más archive objects.

---

# 95. Lazy Collection

También podrá utilizarse para:

```text
lazy transformation pipeline
```

cuando las garantías sean compatibles.

---

# 96. Streaming Result

Podrá utilizarse cuando el driver/plataforma permita streaming real.

---

# 97. Streaming ≠ constant memory guarantee

Encoder, compression, encryption o callbacks podrían acumular memoria.

---

# 98. Archive Planner

La planificación será responsabilidad de:

```text
ArchivePlanner
```

---

# 99. Input

```text
Archive Policy
Archive Subject
Retention Context
Tenant Context
Distribution Context
Resource Budget
Storage Capabilities
```

---

# 100. Output

```text
ArchiveExecutionPlan
```

---

# 101. ArchiveExecutionPlan

Conceptualmente:

```php
final readonly class ArchiveExecutionPlan
{
    public function __construct(
        public ArchiveExecutionId $executionId,
        public ArchivePolicyId $policy,
        public ArchiveSourcePlan $source,
        public ArchiveTargetPlan $target,
        public ArchiveFormatPlan $format,
        public ArchiveVerificationPlan $verification,
        public ArchiveDispositionPlan $disposition,
    ) {}
}
```

---

# 102. Planner ≠ Executor

El planner no:

```text
queries DB
uploads files
deletes source
```

---

# 103. Archive Runner

El:

```text
ArchiveRunner
```

orquestará la ejecución.

---

# 104. Execution lifecycle

```text
PLANNED
   ↓
READING
   ↓
ENCODING
   ↓
WRITING
   ↓
VERIFYING
   ↓
COMMITTING_ARCHIVE
   ↓
SOURCE_DISPOSITION
   ↓
COMPLETED
```

---

# 105. Failure states

```text
FAILED
PARTIAL
CANCELLED
UNKNOWN
```

---

# 106. Archive transaction problem

No existe necesariamente una transacción ACID que abarque:

```text
database
+
object storage
```

---

# 107. Therefore

```text
upload successful
+
DB rollback
```

es posible.

También:

```text
archive successful
+
connection lost before source disposition result known
```

---

# 108. No fake distributed ACID

VoltStack no declarará atomicidad inexistente.

---

# 109. Archive Commit

El concepto:

```text
ArchiveCommit
```

representará que el archive alcanzó las garantías exigidas por la policy.

---

# 110. Archive commit ≠ DB commit

---

# 111. Archive Receipt

El storage provider retornará:

```text
ArchiveStorageReceipt
```

---

# 112. Receipt contents

Podrá contener:

```text
object identity
provider version
etag/checksum
size
timestamp
storage tier
provider metadata
```

---

# 113. Provider ETag ≠ cryptographic integrity

Un ETag no deberá asumirse universalmente como SHA-256.

---

# 114. Integrity System

VoltStack calculará su propia evidencia cuando la policy lo requiera.

---

# 115. Archive checksum

Podrá utilizar:

```text
SHA-256
SHA-512
```

u otros algoritmos permitidos por el Security System.

---

# 116. Algorithm agility

El algoritmo no deberá estar hard-coded en el formato de manera imposible de migrar.

---

# 117. Integrity descriptor

```text
algorithm
digest
scope
encoding
```

---

# 118. Per-object checksum

Cada object podrá tener checksum.

---

# 119. Package checksum

El manifest podrá incluir una raíz de integridad para el package.

---

# 120. Merkle structures

Para packages grandes podrá soportarse opcionalmente:

```text
Merkle Tree
```

para verificación parcial.

No será requisito del core inicial.

---

# 121. Verification levels

```php
enum ArchiveVerificationLevel
{
    case NONE;
    case EXISTENCE;
    case SIZE;
    case CHECKSUM;
    case FULL_READBACK;
    case CUSTOM;
}
```

---

# 122. NONE

Sólo aceptable cuando policy lo permita.

---

# 123. EXISTENCE

Comprueba existencia del object.

---

# 124. SIZE

Comprueba existencia + tamaño esperado.

---

# 125. CHECKSUM

Comprueba contenido mediante digest.

---

# 126. FULL_READBACK

Lee y valida completamente el archive.

Puede ser costoso.

---

# 127. Verification ≠ trust provider response blindly

---

# 128. Archive verification evidence

```text
ArchiveVerificationEvidence
```

deberá ser persistible cuando sea necesaria para Retention.

---

# 129. Integration with Retention

`270_DATABASE_DATA_RETENTION_SYSTEM.md` podrá exigir:

```text
ArchiveRequirement::VERIFIED_REQUIRED
```

---

# 130. Therefore

```text
archive uploaded
```

no es suficiente.

Debe existir:

```text
ArchiveState = VERIFIED
```

cuando la policy lo requiera.

---

# 131. Source Disposition

Después del archive:

```php
enum ArchiveSourceDisposition
{
    case KEEP;
    case SOFT_DELETE;
    case PURGE_IF_ELIGIBLE;
    case PURGE_REQUIRED;
    case CUSTOM;
}
```

---

# 132. KEEP

El source permanece operacional.

---

# 133. SOFT_DELETE

Se conserva pero deja de formar parte del dataset activo.

---

# 134. PURGE_IF_ELIGIBLE

Retention System deberá reevaluar:

```text
purge eligibility
```

---

# 135. PURGE_REQUIRED

Sólo será ejecutable si las policies aplicables lo permiten.

---

# 136. Archive Policy cannot override Retention

Regla crítica:

> Una ArchivePolicy no podrá convertir un dato protegido por Retention en purgeable.

---

# 137. Revalidation

Antes de source purge:

```text
Archive verified
       ↓
Retention re-evaluation
       ↓
Legal Hold re-evaluation
       ↓
Authorization
       ↓
Purge
```

---

# 138. Race condition

Entre archive y purge puede aparecer:

```text
Legal Hold
```

Por ello el archive puede quedar válido mientras el source permanece.

---

# 139. Valid outcome

```text
ARCHIVED
SOURCE_RETAINED
```

es un resultado perfectamente válido.

---

# 140. Archive Registry

VoltStack necesitará registrar archives existentes.

---

# 141. Archive Catalog

Conceptualmente:

```text
ArchiveCatalog
```

---

# 142. Catalog responsibilities

```text
package discovery
metadata lookup
status
location
policy
schema
tenant
time range
verification state
```

---

# 143. Catalog ≠ payload storage

El catálogo no deberá almacenar todo el archive.

---

# 144. Catalog entry

```text
ArchiveCatalogEntry
├── packageId
├── policy
├── state
├── target
├── createdAt
├── verifiedAt
├── recordCount
├── tenant
├── shard
├── timeRange
├── schemaFingerprint
└── manifestLocation
```

---

# 145. Catalog durability

La pérdida del catálogo no debería necesariamente hacer irrecuperables archives self-describing.

---

# 146. Self-describing package

Idealmente:

```text
Storage
└── Package
    ├── manifest
    └── data
```

puede reconstruir parte del catálogo.

---

# 147. Catalog rebuild

Podrá existir:

```text
archive:catalog:rebuild
```

---

# 148. Archive discovery

Un provider podrá implementar:

```text
list packages
read manifest
verify package
register catalog entry
```

---

# 149. Object naming

Los nombres físicos deberán evitar depender únicamente de datos mutables.

---

# 150. Example

```text
archives/
  financial/
    2026/
      arc_pkg_01J...
```

---

# 151. Sensitive names

No:

```text
archives/customer-john-doe@email.com/
```

---

# 152. Tenant partitioning

Podrá existir:

```text
tenant/<opaque-tenant-id>/
```

según policy.

---

# 153. Tenant Archive

Cada tenant deberá permanecer aislado.

---

# 154. Archive tenant context

Manifest deberá incluir una identidad tenant estable cuando sea relevante.

---

# 155. Tenant isolation

Un tenant no podrá:

```text
discover
read
restore
delete
```

archives de otro tenant.

---

# 156. Cross-tenant archives

Estarán deshabilitados por default.

---

# 157. Administrative aggregate archives

Si existen:

```text
platform-wide archives
```

serán explícitos y privilegiados.

---

# 158. Tenant offboarding

Workflow posible:

```text
Tenant Disabled
      ↓
Final Operational Snapshot
      ↓
Archive
      ↓
Verify
      ↓
Retention Period
      ↓
Purge Operational Data
      ↓
Archive Retention
      ↓
Final Archive Purge
```

---

# 159. Archive encryption

Archives sensibles deberán soportar cifrado.

---

# 160. Encryption layers

Podrán existir:

```text
provider-side encryption
application-side encryption
envelope encryption
```

---

# 161. Provider encryption ≠ application encryption

Deben modelarse por separado.

---

# 162. Encryption profile

```text
ArchiveEncryptionProfile
```

podrá definir:

```text
algorithm
key reference
key version
envelope strategy
rotation policy
```

---

# 163. Keys not in manifest

El manifest no contendrá claves privadas/decryption secrets.

---

# 164. Key reference

Podrá almacenar:

```text
kms-key-alias
key version
encrypted data key
```

según estrategia.

---

# 165. Key rotation

Archive encryption deberá considerar archivos que vivirán muchos años.

---

# 166. Rotation ≠ rewriting always

Dependiendo de envelope encryption puede rotarse:

```text
key-encryption-key
```

sin reescribir todo el payload.

---

# 167. Crypto erasure

La destrucción de una clave puede ser parte de una estrategia de purge.

Sólo si la arquitectura demuestra que el dato se vuelve inaccesible.

---

# 168. Compression

Archive podrá aplicar:

```text
NONE
GZIP
ZSTD
CUSTOM
```

según adapters disponibles.

---

# 169. Compression order

Pipeline recomendado:

```text
serialize
↓
compress
↓
encrypt
```

---

# 170. Why

Los datos cifrados normalmente no son compresibles eficientemente.

---

# 171. Compression metadata

El manifest deberá identificar:

```text
algorithm
version/options if necessary
```

---

# 172. Compression bomb protection

Restore deberá protegerse contra:

```text
decompression bombs
```

mediante Resource Governance.

---

# 173. Archive size budgets

```text
max object size
max package size
max uncompressed size
max records
```

---

# 174. Object splitting

Packages grandes podrán dividirse:

```text
part-000001
part-000002
part-000003
```

---

# 175. Multipart upload

Providers podrán soportar multipart upload mediante capability.

---

# 176. Capability-driven architecture

No:

```php
if ($provider === 's3') {
}
```

---

# 177. Storage capabilities

Ejemplos:

```text
supportsMultipartUpload()
supportsObjectVersioning()
supportsObjectLock()
supportsChecksum()
supportsStorageClasses()
supportsServerSideEncryption()
supportsAtomicCreate()
supportsConditionalWrite()
```

---

# 178. Provider capability ≠ policy guarantee

Que un provider soporte Object Lock no significa que toda ArchivePolicy lo utilice.

---

# 179. Immutability

Archives regulatorios pueden requerir:

```text
WORM
```

Write Once Read Many.

---

# 180. Archive immutability policy

```php
enum ArchiveImmutabilityMode
{
    case NONE;
    case APPLICATION_ENFORCED;
    case PROVIDER_ENFORCED;
    case WORM;
}
```

---

# 181. Application-enforced ≠ provider-enforced

Una aplicación puede prohibir delete, pero un administrador del storage podría borrarlo.

---

# 182. Evidence strength

El sistema deberá poder describir la fuerza de la garantía.

---

# 183. Archive lock

Podrá existir:

```text
ArchiveRetentionLock
```

independiente de Data Retention policy.

---

# 184. Storage retention ≠ application retention

Deben coordinarse.

---

# 185. Dangerous mismatch

```text
Application retention:
  5 years

Storage immutable lock:
  20 years
```

puede impedir una obligación de eliminación anterior.

---

# 186. Policy validation

VoltStack deberá detectar configuraciones incompatibles cuando pueda demostrarlas.

---

# 187. Archive deletion

Archives también tienen lifecycle.

---

# 188. Archive retention

Un archive podrá tener:

```text
ArchiveRetentionPolicy
```

---

# 189. Example

```text
Operational:
  2 years

Archive:
  years 2–7

Final purge:
  year 7
```

---

# 190. Archive expiration ≠ automatic deletion

---

# 191. Archive purge

Deberá seguir:

```text
Archive Retention Evaluation
       ↓
Legal Hold
       ↓
Security
       ↓
Storage Lock
       ↓
Purge Plan
       ↓
Delete Archive
       ↓
Verify Deletion
```

---

# 192. Delete request ≠ deleted

Provider puede aceptar:

```text
DELETE
```

pero mantener:

```text
version
replica
retention lock
delayed deletion
```

---

# 193. ArchiveDeletionEvidence

Deberá registrar la garantía realmente conocida.

---

# 194. Storage versioning

Con object versioning:

```text
delete marker
```

no equivale necesariamente a destrucción de versiones anteriores.

---

# 195. Purge all versions

Cuando policy requiera destrucción:

```text
all retained object versions
```

deberán evaluarse.

---

# 196. Archive retrieval

VoltStack deberá soportar recuperación.

---

# 197. Retrieval ≠ Restore

Retrieval:

```text
obtener/leer archive
```

Restore:

```text
reintegrarlo a un sistema operacional
```

---

# 198. Archive Reader

```php
interface ArchiveReader
{
    public function open(
        ArchivePackageId $package,
        ArchiveReadContext $context,
    ): ArchiveRecordStream;
}
```

---

# 199. Read pipeline

```text
Storage
 ↓
Integrity Check
 ↓
Decrypt
 ↓
Decompress
 ↓
Decode
 ↓
Canonical Records
```

---

# 200. Verification before trust

Datos recuperados no deberán confiarse antes de verificar las garantías exigidas.

---

# 201. Archive Rehydration

Rehydration convierte:

```text
CanonicalArchiveRecord
```

a una representación utilizable.

---

# 202. Archive Rehydration ≠ ORM Hydration

El archive puede haber sido generado con una versión antigua del modelo.

---

# 203. Rehydration target

Podrá ser:

```text
DTO
array
historical entity representation
current entity
migration representation
```

---

# 204. Direct current entity hydration risk

Un archive de 2020 puede no coincidir con:

```text
Customer 2026
```

---

# 205. Archive Migration

Podrá existir:

```text
ArchiveSchemaMigration
```

para transformar:

```text
Archive Format/Schema N
→
Readable Representation N+K
```

---

# 206. Archive migration ≠ DB migration

---

# 207. Preserve original

Una transformación para lectura no deberá destruir automáticamente el archive original.

---

# 208. Restore

Restore será una operación explícita.

---

# 209. Restore modes

```php
enum ArchiveRestoreMode
{
    case PREVIEW;
    case TEMPORARY;
    case REIMPORT;
    case RECONCILE;
    case CUSTOM;
}
```

---

# 210. PREVIEW

Lee datos sin insertarlos en DB.

---

# 211. TEMPORARY

Restaura a un entorno/almacenamiento temporal.

---

# 212. REIMPORT

Inserta datos nuevamente.

---

# 213. RECONCILE

Compara archive con estado operacional y decide acciones.

---

# 214. Restore ≠ blind INSERT

Debe considerar:

```text
current schema
existing identities
tenant
relationships
retention
security
history
conflicts
```

---

# 215. Identity conflict

Puede existir:

```text
Archived Customer #42
Current Customer #42
```

---

# 216. Conflict strategies

```text
FAIL
REMAP
MERGE
SKIP
CUSTOM
```

Nunca un overwrite silencioso por default.

---

# 217. Restore + IdentityMap

Si se restauran entities dentro de ORM:

```text
IdentityMap
```

deberá preservar identidad canónica.

---

# 218. Restore + UoW

La restauración utilizará Persistence Engine normal cuando opere a nivel ORM.

---

# 219. Large restore

Podrá usar:

```text
Chunk
Bulk Insert
Batch Persistence
```

según garantías requeridas.

---

# 220. Restore transaction

No deberá asumir una única transaction gigantesca.

---

# 221. Restore checkpoint

Podrá existir:

```text
ArchiveRestoreCheckpoint
```

---

# 222. Resume

Debe validar:

```text
package identity
manifest fingerprint
schema transformation
tenant
target DB
checkpoint version
```

---

# 223. Search metadata

No siempre será viable buscar directamente dentro de terabytes de archive.

---

# 224. Archive Search Index

Podrá existir un índice separado con metadata como:

```text
package
tenant
entity type
time range
classification
business key hash
```

---

# 225. Search index ≠ archive truth

El archive package continúa siendo la fuente archivada.

---

# 226. Search index rebuild

Debe ser posible reconstruirlo cuando la policy/format lo permita.

---

# 227. Sensitive search metadata

No deberán indexarse PII/secretos sin justificación.

---

# 228. Full-text search

La integración con:

```text
272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
```

podrá permitir búsqueda sobre metadata o contenido permitido.

---

# 229. Full-text index ≠ archive

---

# 230. Incremental archival

VoltStack deberá soportar archival incremental.

---

# 231. Incremental ≠ differential backup

Archival incremental significa procesar nuevos candidatos desde una ejecución anterior.

---

# 232. Checkpoint

```text
ArchiveCheckpoint
```

podrá contener:

```text
policy
query fingerprint
ordering
continuation boundary
tenant
shard
schema generation
policy generation
processed records
package state
```

---

# 233. Checkpoint ≠ exactly once

---

# 234. Crash scenario

```text
upload object
↓
crash
↓
checkpoint not updated
```

Al reiniciar podría intentarse nuevamente.

---

# 235. Idempotency

Archive object creation deberá diseñarse para soportar:

```text
retries
deduplication
reconciliation
```

---

# 236. Deterministic object identity

Cuando sea apropiado podrá derivarse de:

```text
execution
package
part number
```

---

# 237. Conditional create

Si provider soporta:

```text
create-if-absent
```

podrá utilizarse.

---

# 238. Retry

Retry deberá distinguir:

```text
safe retry
unsafe retry
unknown outcome
```

---

# 239. UNKNOWN upload

Si:

```text
upload sent
connection lost
```

el sistema deberá reconciliar existencia/integridad antes de volver a escribir cuando sea posible.

---

# 240. UNKNOWN ≠ failed

---

# 241. Reconciliation

Podrá ejecutar:

```text
lookup expected object
↓
verify identity
↓
verify checksum
↓
reuse or rewrite
```

---

# 242. Partial package

Un package puede quedar:

```text
PARTIAL
```

---

# 243. PARTIAL package ≠ valid archive

No deberá habilitar source purge.

---

# 244. Package finalization

Sólo después de completar:

```text
all required objects
manifest
verification
```

podrá transicionar a:

```text
VERIFIED
```

---

# 245. Manifest finalization

Puede utilizarse estrategia:

```text
write parts
↓
verify parts
↓
write final manifest
↓
verify manifest
↓
mark catalog VERIFIED
```

---

# 246. Final manifest as commit marker

Puede funcionar conceptualmente como señal de package completo.

Pero:

```text
manifest exists
```

por sí solo no sustituye verification cuando ésta es requerida.

---

# 247. Distributed archival

En sharding:

```text
Shard A
Shard B
Shard C
```

podrán archivarse independientemente.

---

# 248. Per-shard package

Recomendado cuando no existe necesidad de archive global.

---

# 249. Global package

Podrá requerir:

```text
distributed merge
global ordering
cross-shard manifest
```

---

# 250. No fake global consistency

Un global archive deberá declarar su consistency model.

---

# 251. Shard map generation

Manifest/checkpoint podrá incluir:

```text
ShardMapGeneration
```

---

# 252. Topology changes

Un resume deberá validar si:

```text
resharding
migration
failover
```

afecta el dataset.

---

# 253. Archive consistency

Podrán existir:

```php
enum ArchiveConsistency
{
    case BEST_EFFORT;
    case STABLE_HORIZON;
    case SNAPSHOT;
    case TRANSACTIONAL_SOURCE;
    case CUSTOM;
}
```

---

# 254. BEST_EFFORT

Cada chunk puede observar cambios.

---

# 255. STABLE_HORIZON

Se fija un límite lógico:

```text
archive all records <= boundary T
```

---

# 256. SNAPSHOT

Requiere fuente capaz de proporcionar snapshot consistente.

---

# 257. Snapshot cost

Puede ser caro o imposible para datasets enormes.

---

# 258. No automatic huge snapshot

VoltStack no abrirá automáticamente una transaction de horas.

---

# 259. Stable horizon

Será frecuentemente una alternativa práctica.

Ejemplo:

```text
archive closed invoices
where closed_at <= 2025-12-31
```

---

# 260. Horizon ≠ snapshot

Los registros dentro del horizonte todavía podrían modificarse si el dominio lo permite.

---

# 261. Immutable source

Cuando el dominio garantice inmutabilidad después de cierre:

```text
STABLE_HORIZON
```

puede proporcionar garantías fuertes.

---

# 262. Mutation detection

Opcionalmente podrá utilizarse:

```text
version
updated_at
checksum
```

para detectar cambios entre lectura y finalización.

---

# 263. Mutation detection ≠ transaction

---

# 264. Source mutation policy

```text
FAIL
RETRY_SUBJECT
REARCHIVE
MARK_STALE
CUSTOM
```

---

# 265. Archive supersession

Si un dato archivado cambia legítimamente:

```text
Archive Package V1
```

podrá ser reemplazado lógicamente por:

```text
Archive Package V2
```

sin destruir inmediatamente V1.

---

# 266. Supersedes relationship

Manifest podrá indicar:

```text
supersedesPackage
```

---

# 267. Immutable archives

Preferentemente los archives ya finalizados no serán modificados in-place.

---

# 268. New version instead of mutation

```text
Archive V1
      ↓ superseded by
Archive V2
```

---

# 269. Cache integration

Result Cache no deberá utilizarse por default como fuente para archival.

---

# 270. Why

Archive requiere evidencia suficientemente fuerte del source.

---

# 271. Metadata Cache

Sí podrá reutilizar:

```text
compiled metadata
schema metadata
type metadata
policy metadata
```

---

# 272. Archive Catalog Cache

Podrá existir con invalidación apropiada.

---

# 273. Cache ≠ archive truth

---

# 274. Security architecture

Archival deberá integrarse con:

```text
226_DATABASE_SECURITY_ARCHITECTURE.md
```

---

# 275. Permissions

Ejemplos:

```text
archive.view
archive.plan
archive.execute
archive.verify
archive.retrieve
archive.restore
archive.delete
archive.manage_policy
archive.manage_provider
```

---

# 276. Separation of duties

Podrá configurarse:

```text
archive operator
≠
archive deletion operator
```

---

# 277. Archive delete

Será una de las operaciones más privilegiadas.

---

# 278. Archive restore

También podrá ser altamente sensible porque devuelve datos antiguos al sistema operacional.

---

# 279. Authorization ≠ eligibility

Tener permiso para archivar no convierte un dato en elegible.

---

# 280. Eligibility ≠ authorization

Que un dato sea elegible tampoco concede permiso.

---

# 281. Credential security

Storage credentials deberán resolverse mediante:

```text
229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
```

o infraestructura equivalente.

---

# 282. No secret logging

Nunca registrar:

```text
access keys
secret keys
signed credentials
encryption keys
```

---

# 283. Signed URLs

Si se utilizan:

```text
pre-signed URLs
```

deberán tratarse como secretos temporales.

---

# 284. Archive download

Acceso deberá estar:

```text
authorized
audited
time-bounded
```

cuando aplique.

---

# 285. Sensitive Data Protection

Manifest, catalog y payload estarán sujetos a:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
```

---

# 286. Audit

Operaciones críticas:

```text
archive created
archive verified
archive retrieved
archive restored
archive deleted
policy changed
provider changed
verification failed
```

deberán poder auditarse.

---

# 287. Audit ≠ telemetry

---

# 288. Audit minimization

El audit no deberá copiar todo el payload.

---

# 289. Audit record

Podrá contener:

```text
package ID
policy ID/version
operation
actor
tenant
timestamp
result
verification level
```

---

# 290. Query audit

Candidate queries podrán integrarse con Query Audit.

---

# 291. Telemetry

Métricas posibles:

```text
database_archive_executions_total
database_archive_records_total
database_archive_bytes_total
database_archive_packages_total
database_archive_failures_total
database_archive_verification_failures_total
database_archive_restore_total
database_archive_duration
```

---

# 292. Bounded labels

Ejemplos:

```text
policy category
storage tier
format
result
verification level
```

---

# 293. Forbidden high-cardinality labels

No:

```text
package ID
tenant ID
customer ID
object key
```

por default.

---

# 294. Tracing

Spans:

```text
database.archive.plan
database.archive.read
database.archive.encode
database.archive.write
database.archive.verify
database.archive.finalize
database.archive.source_disposition
database.archive.restore
```

---

# 295. Per-record spans

No serán default.

---

# 296. Archive profiler

Podrá medir:

```text
DB read throughput
encoding throughput
compression ratio
encryption throughput
upload throughput
verification throughput
```

---

# 297. Slow archive detection

Podrá identificar:

```text
slow source query
slow object storage
compression bottleneck
encryption bottleneck
verification bottleneck
```

---

# 298. Resource Governance

Integración obligatoria con:

```text
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
```

---

# 299. Budgets

```text
max execution duration
max records
max bytes
max memory
max package size
max object size
max concurrent uploads
max DB concurrency
max retry count
max restore size
```

---

# 300. Backpressure

Cuando storage sea más lento que DB:

```text
DB producer
    ↓
bounded buffer
    ↓
storage consumer
```

---

# 301. No unbounded queue

No:

```text
read 10M rows
→ queue in memory
→ upload later
```

---

# 302. Bounded pipeline

Cada stage deberá respetar capacidad limitada.

---

# 303. Concurrency

Podrá existir paralelismo en:

```text
encoding
compression
encryption
upload
verification
```

cuando sea seguro.

---

# 304. Ordering

Si el formato exige ordering:

```text
parallelism
```

no deberá romperlo.

---

# 305. Package ordering

Puede ser:

```text
ORDERED
UNORDERED
PARTITION_ORDERED
```

---

# 306. Determinism

Cuando se requiera archive reproducible:

```text
canonical ordering
canonical encoding
stable metadata
```

serán necesarios.

---

# 307. Deterministic archive ≠ same encrypted bytes

Encryption puede utilizar IV/nonces aleatorios legítimos.

---

# 308. Logical fingerprint

Podrá calcularse antes de encryption para identificar contenido lógico.

---

# 309. Physical fingerprint

Podrá calcularse después de encoding/compression/encryption.

---

# 310. Logical ≠ physical fingerprint

Ambos tienen propósitos distintos.

---

# 311. Failure model

Fallos posibles:

```text
source query failure
hydration/conversion failure
encoding failure
compression failure
encryption failure
storage failure
verification failure
catalog failure
checkpoint failure
source purge failure
unknown external outcome
```

---

# 312. Failure classification

```text
RETRYABLE
NON_RETRYABLE
REQUIRES_RECONCILIATION
UNKNOWN
```

---

# 313. Storage timeout

No siempre significa:

```text
object not stored
```

---

# 314. Reconciliation required

Ante resultado incierto:

```text
inspect provider
verify expected object
compare integrity
```

---

# 315. Circuit Breaker

Integración con:

```text
239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
```

podrá proteger storage providers.

---

# 316. Provider circuit

Un storage provider fallando repetidamente puede pasar a:

```text
OPEN
```

---

# 317. Circuit breaker ≠ archive failure semantics

Sólo evita llamadas adicionales temporalmente.

---

# 318. Failover

Archive targets podrán tener:

```text
primary
secondary
```

si la policy lo permite.

---

# 319. Storage failover danger

Cambiar target puede afectar:

```text
jurisdiction
residency
encryption
immutability
cost
retention guarantees
```

---

# 320. No blind failover

---

# 321. Target eligibility

El Archive Planner deberá comprobar que el target alternativo cumple las capacidades requeridas.

---

# 322. Data residency

Una ArchivePolicy podrá restringir:

```text
region
country
provider class
```

---

# 323. Residency ≠ performance preference

Puede ser requisito de governance.

---

# 324. Unknown target location

Si residency es obligatoria:

```text
UNKNOWN
```

deberá bloquear archival.

---

# 325. Archive portability

Los formatos deberán minimizar lock-in cuando sea un objetivo de policy.

---

# 326. Provider-specific metadata

Podrá almacenarse adicionalmente, pero el archive lógico no deberá depender innecesariamente de ella.

---

# 327. Archive migration between providers

Podrá realizar:

```text
Provider A
   ↓
verify source
   ↓
copy
   ↓
verify destination
   ↓
register destination
   ↓
policy check
   ↓
delete old archive if permitted
```

---

# 328. Copy ≠ migration completed

La migración termina después de verificar el nuevo target y actualizar authoritative metadata.

---

# 329. Testing

El sistema requerirá pruebas exhaustivas.

---

# 330. Unit tests

```text
policy resolution
eligibility
manifest generation
format versioning
checksums
package state
source disposition
```

---

# 331. Provider contract tests

Todo provider deberá pasar:

```text
write
read
existence
failure
timeout
retry
checksum
delete
versioning where supported
```

---

# 332. Encoder tests

```text
canonical representation
nulls
enums
timestamps
JSON
binary
value objects
large records
```

---

# 333. Integrity tests

Modificar un byte deberá producir fallo cuando checksum sea requerido.

---

# 334. Encryption tests

```text
encrypt
decrypt
wrong key
rotated key
missing key
corrupted ciphertext
```

---

# 335. Compression tests

```text
compress
decompress
corruption
size limits
bomb protection
```

---

# 336. Partial upload tests

```text
part 1 ✓
part 2 ✓
part 3 ✗
```

no deberá producir:

```text
VERIFIED
```

---

# 337. Crash recovery

Probar crash después de:

```text
upload
manifest write
verification
catalog write
source purge
```

---

# 338. UNKNOWN outcomes

Deberán permanecer explícitos.

---

# 339. Retention integration tests

```text
archive verified
+
purge eligible
→ purge allowed
```

pero:

```text
archive verified
+
legal hold
→ source retained
```

---

# 340. Tenant tests

A y B deberán permanecer aislados.

---

# 341. Shard tests

```text
per-shard package
partial shard failure
resume
resharding
```

---

# 342. Restore tests

```text
old schema
current schema
identity conflict
tenant mismatch
relationship mismatch
```

---

# 343. Persistent runtime tests

Request A:

```text
Tenant A
Archive Target A
```

Request B:

```text
Tenant B
Archive Target B
```

No deberá existir contaminación.

---

# 344. Persistent Runtime

Archival deberá ser compatible con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 345. Shared immutable state

Podrá compartirse:

```text
compiled policies
format descriptors
immutable provider metadata
encoder definitions
```

---

# 346. Scoped mutable state

Será request/job/execution scoped:

```text
tenant
execution
package builder
checkpoint
credentials
current provider session
buffer
source iterator
```

---

# 347. Forbidden globals

No:

```php
static ?ArchivePackage $currentPackage;
static ?TenantId $tenant;
static array $uploadBuffer;
```

---

# 348. Worker cleanup

Al terminar ejecución deberán liberarse:

```text
DB cursors
streams
temporary files
encryption contexts
compression contexts
multipart sessions
buffers
provider handles
```

---

# 349. Cancellation

Archive execution será cancelable.

---

# 350. Graceful cancellation

Debe:

```text
stop source reading
stop new uploads
finish/cancel active operation safely
persist checkpoint if possible
close resources
mark package appropriately
```

---

# 351. Cancellation ≠ failure

Podrá resultar:

```text
CANCELLED
```

---

# 352. Cancellation ≠ valid archive

Un package incompleto seguirá:

```text
PARTIAL
```

o será descartado.

---

# 353. Temporary objects

El sistema podrá utilizar:

```text
temporary/
staging/
```

namespace antes de finalización.

---

# 354. Promotion

Cuando provider lo permita:

```text
staging
→ verified
→ final
```

---

# 355. Rename assumptions

Object stores no siempre soportan rename atómico.

No deberá diseñarse arquitectura suponiendo semántica de filesystem POSIX.

---

# 356. Copy + delete

Algunos providers implementan move como:

```text
copy
+
delete
```

Esto deberá reconocerse.

---

# 357. Temporary cleanup

Stale multipart uploads/staging objects requerirán maintenance.

---

# 358. Cleanup ≠ archive purge

Son residuos operacionales, no archives finalizados.

---

# 359. Maintenance

Podrán existir comandos:

```text
archive:cleanup-staging
archive:abort-stale-uploads
archive:verify
archive:reconcile
archive:catalog:rebuild
```

---

# 360. CLI conceptual

```bash
volt database:archive:plan invoices
volt database:archive:run invoices
volt database:archive:verify arc_pkg_...
volt database:archive:inspect arc_pkg_...
volt database:archive:restore arc_pkg_... --preview
volt database:archive:reconcile
```

---

# 361. Dry-run

```bash
volt database:archive:run invoices --dry-run
```

---

# 362. Dry-run output

```text
Archive Policy
  financial-invoices-archive:v3

Candidates
  1,284,100

Estimated Data
  186 GB

Eligible
  1,280,402

Blocked
  3,698

Target
  cold-storage-primary

Format
  parquet-v1

Compression
  zstd

Encryption
  application-envelope

Verification
  checksum

Source Disposition
  purge-if-eligible
```

---

# 363. Dry-run ≠ guarantee

El estado debe revalidarse durante ejecución.

---

# 364. Diagnostics

```text
Archive Diagnostics
────────────────────────────────

Package:
  arc_pkg_01J...

State:
  VERIFIED

Policy:
  financial-invoices-archive:v3

Records:
  2,500,000

Objects:
  25

Logical Size:
  42.1 GB

Stored Size:
  8.7 GB

Compression:
  ZSTD

Encryption:
  envelope-v2

Verification:
  SHA-256

Tenant:
  tenant-scoped

Source:
  shard-03

Disposition:
  source-purged
```

---

# 365. Explainability

Debe poder responder:

```text
¿Por qué se archivó este dataset?
¿Con qué policy?
¿Dónde está?
¿Con qué formato?
¿Está verificado?
¿Se eliminó el source?
¿Puede restaurarse?
```

---

# 366. Directory proposal

```text
src/Quantum/Database/Archive/
│
├── Contract/
│   ├── ArchivePlanner.php
│   ├── ArchiveRunner.php
│   ├── ArchiveEligibilityEvaluator.php
│   ├── ArchiveStorageProvider.php
│   ├── ArchiveEncoder.php
│   ├── ArchiveDecoder.php
│   ├── ArchiveVerifier.php
│   ├── ArchiveReader.php
│   └── ArchiveCatalog.php
│
├── Policy/
│   ├── ArchivePolicy.php
│   ├── ArchivePolicyId.php
│   ├── ArchivePolicyVersion.php
│   ├── ArchivePolicyRegistry.php
│   ├── ArchiveScope.php
│   ├── ArchiveTargetPolicy.php
│   ├── ArchiveFormatPolicy.php
│   ├── ArchiveVerificationPolicy.php
│   ├── ArchiveEncryptionProfile.php
│   └── ArchiveSourceDisposition.php
│
├── Eligibility/
│   ├── ArchiveCandidate.php
│   ├── ArchiveEligibilityDecision.php
│   ├── ArchiveEligibilityResult.php
│   └── DefaultArchiveEligibilityEvaluator.php
│
├── Planning/
│   ├── ArchiveExecutionPlan.php
│   ├── ArchiveSourcePlan.php
│   ├── ArchiveTargetPlan.php
│   ├── ArchiveFormatPlan.php
│   ├── ArchiveVerificationPlan.php
│   ├── ArchiveDispositionPlan.php
│   └── DefaultArchivePlanner.php
│
├── Package/
│   ├── ArchivePackage.php
│   ├── ArchivePackageId.php
│   ├── ArchiveManifest.php
│   ├── ArchiveManifestVersion.php
│   ├── ArchiveObject.php
│   ├── ArchiveObjectId.php
│   ├── ArchiveRecord.php
│   └── ArchiveState.php
│
├── Graph/
│   ├── ArchiveGraph.php
│   ├── ArchiveGraphNode.php
│   ├── ArchiveGraphEdge.php
│   ├── ArchiveRelationshipPolicy.php
│   └── ArchiveGraphPlanner.php
│
├── Storage/
│   ├── ArchiveStorageContext.php
│   ├── ArchiveStorageReceipt.php
│   ├── ArchiveStorageCapabilities.php
│   ├── ArchiveStorageTier.php
│   ├── ArchiveImmutabilityMode.php
│   └── Provider/
│       ├── FilesystemArchiveProvider.php
│       └── S3CompatibleArchiveProvider.php
│
├── Format/
│   ├── ArchiveFormat.php
│   ├── ArchiveFormatVersion.php
│   ├── ArchiveEncodingContext.php
│   ├── ArchiveDecodingContext.php
│   └── CanonicalArchiveRecord.php
│
├── Compression/
│   ├── ArchiveCompressor.php
│   ├── ArchiveDecompressor.php
│   └── ArchiveCompressionProfile.php
│
├── Encryption/
│   ├── ArchiveEncryptor.php
│   ├── ArchiveDecryptor.php
│   ├── ArchiveEncryptionContext.php
│   └── ArchiveKeyReference.php
│
├── Integrity/
│   ├── ArchiveChecksum.php
│   ├── ArchiveIntegrityDescriptor.php
│   ├── ArchiveVerificationEvidence.php
│   └── DefaultArchiveVerifier.php
│
├── Catalog/
│   ├── ArchiveCatalogEntry.php
│   ├── ArchiveCatalogRepository.php
│   └── ArchiveCatalogRebuilder.php
│
├── Execution/
│   ├── ArchiveExecutionId.php
│   ├── ArchiveExecutionContext.php
│   ├── ArchiveCheckpoint.php
│   ├── ArchiveExecutionResult.php
│   └── DefaultArchiveRunner.php
│
├── Restore/
│   ├── ArchiveRestorePlan.php
│   ├── ArchiveRestoreMode.php
│   ├── ArchiveRestoreCheckpoint.php
│   ├── ArchiveRehydrator.php
│   └── ArchiveRestoreRunner.php
│
├── Search/
│   ├── ArchiveSearchIndex.php
│   ├── ArchiveSearchDocument.php
│   └── ArchiveSearchQuery.php
│
├── Diagnostics/
│   ├── ArchiveDiagnostics.php
│   ├── ArchiveExplain.php
│   └── ArchiveReconciliationReport.php
│
├── Telemetry/
│   └── ArchiveTelemetry.php
│
└── Exception/
    ├── ArchiveException.php
    ├── ArchivePolicyException.php
    ├── ArchiveEligibilityException.php
    ├── ArchiveStorageException.php
    ├── ArchiveEncodingException.php
    ├── ArchiveCompressionException.php
    ├── ArchiveEncryptionException.php
    ├── ArchiveVerificationException.php
    ├── ArchiveIntegrityException.php
    ├── ArchiveRestoreException.php
    ├── ArchivePartialException.php
    └── ArchiveOutcomeUnknownException.php
```

---

# 367. Exception hierarchy

```text
DatabaseException
└── ArchiveException
    ├── ArchivePolicyException
    ├── ArchiveEligibilityException
    ├── ArchiveStorageException
    ├── ArchiveEncodingException
    ├── ArchiveCompressionException
    ├── ArchiveEncryptionException
    ├── ArchiveVerificationException
    │   └── ArchiveIntegrityException
    ├── ArchiveRestoreException
    ├── ArchivePartialException
    └── ArchiveOutcomeUnknownException
```

---

# 368. Architectural invariants

## DB-ARCHIVE-001

Archive será distinto de Backup.

## DB-ARCHIVE-002

Archive será distinto de Export.

## DB-ARCHIVE-003

Archive será distinto de Retention.

## DB-ARCHIVE-004

Archive será distinto de Purge.

## DB-ARCHIVE-005

Archive será distinto de Soft Delete.

## DB-ARCHIVE-006

Archive será distinto de History.

## DB-ARCHIVE-007

Archive candidate será distinto de archive eligible.

## DB-ARCHIVE-008

Archive eligible será distinto de purge eligible.

## DB-ARCHIVE-009

UNKNOWN no será considerado eligible para operaciones destructivas.

## DB-ARCHIVE-010

Archive Policy tendrá ID estable.

## DB-ARCHIVE-011

Archive Policy será versionable.

## DB-ARCHIVE-012

Policy version será distinta de format version.

## DB-ARCHIVE-013

Format version será distinta de schema version.

## DB-ARCHIVE-014

Storage tier será distinto de provider.

## DB-ARCHIVE-015

Provider será abstraído por capability.

## DB-ARCHIVE-016

El core no dependerá de S3/AWS/MinIO específicos.

## DB-ARCHIVE-017

Credenciales no residirán directamente en ArchivePolicy.

## DB-ARCHIVE-018

Archive Package será distinto de file.

## DB-ARCHIVE-019

Archive Manifest será distinto del payload.

## DB-ARCHIVE-020

Manifest podrá ser sensible.

## DB-ARCHIVE-021

Archive deberá conservar información suficiente para interpretar los datos.

## DB-ARCHIVE-022

Logical types podrán preservarse.

## DB-ARCHIVE-023

Encoder no ejecutará queries.

## DB-ARCHIVE-024

Storage Provider no decidirá queries.

## DB-ARCHIVE-025

Archival no generará SQL manualmente.

## DB-ARCHIVE-026

Candidate discovery utilizará Query Engine.

## DB-ARCHIVE-027

Selection no será final eligibility.

## DB-ARCHIVE-028

Archival de grandes datasets será bounded.

## DB-ARCHIVE-029

Chunk será distinto de Archive Package.

## DB-ARCHIVE-030

Streaming no garantizará memoria constante por sí mismo.

## DB-ARCHIVE-031

Archive Planner no ejecutará I/O.

## DB-ARCHIVE-032

Archive Runner ejecutará planes ya validados.

## DB-ARCHIVE-033

Archive commit será distinto de DB commit.

## DB-ARCHIVE-034

VoltStack no fingirá distributed ACID entre DB y storage.

## DB-ARCHIVE-035

Storage receipt no será prueba universal de integridad.

## DB-ARCHIVE-036

Provider ETag no se asumirá criptográficamente seguro.

## DB-ARCHIVE-037

Integrity algorithm será versionable/agile.

## DB-ARCHIVE-038

Verification level será explícito.

## DB-ARCHIVE-039

WRITTEN será distinto de VERIFIED.

## DB-ARCHIVE-040

PARTIAL será distinto de VERIFIED.

## DB-ARCHIVE-041

Archive required by Retention deberá alcanzar la verification exigida.

## DB-ARCHIVE-042

ArchivePolicy no podrá debilitar Retention.

## DB-ARCHIVE-043

Source purge requerirá revalidación de Retention.

## DB-ARCHIVE-044

Legal Hold podrá impedir source purge después de archive.

## DB-ARCHIVE-045

ARCHIVED + SOURCE_RETAINED será estado válido.

## DB-ARCHIVE-046

Archive Catalog será distinto de payload storage.

## DB-ARCHIVE-047

Archives self-describing podrán facilitar catalog rebuild.

## DB-ARCHIVE-048

Object names evitarán PII innecesaria.

## DB-ARCHIVE-049

Tenant archives permanecerán aislados.

## DB-ARCHIVE-050

Cross-tenant archival estará deshabilitado por default.

## DB-ARCHIVE-051

Encryption profile no contendrá secretos.

## DB-ARCHIVE-052

Encryption keys no estarán en manifest.

## DB-ARCHIVE-053

Provider encryption será distinta de application encryption.

## DB-ARCHIVE-054

Compression ocurrirá antes de encryption por default.

## DB-ARCHIVE-055

Restore protegerá contra decompression bombs.

## DB-ARCHIVE-056

Archive pipeline tendrá resource budgets.

## DB-ARCHIVE-057

Multipart upload será capability-driven.

## DB-ARCHIVE-058

Provider capability será distinta de policy guarantee.

## DB-ARCHIVE-059

Application immutability será distinta de provider immutability.

## DB-ARCHIVE-060

WORM será una garantía explícita.

## DB-ARCHIVE-061

Storage retention será distinta de application retention.

## DB-ARCHIVE-062

Conflicting storage/application retention deberá detectarse cuando sea posible.

## DB-ARCHIVE-063

Archive expiration será distinta de archive deletion.

## DB-ARCHIVE-064

Delete request será distinto de verified deletion.

## DB-ARCHIVE-065

Delete marker será distinto de physical purge.

## DB-ARCHIVE-066

Object versions deberán considerarse en purge.

## DB-ARCHIVE-067

Retrieval será distinto de Restore.

## DB-ARCHIVE-068

Archive Rehydration será distinta de ORM Hydration.

## DB-ARCHIVE-069

Archive Schema Migration será distinta de DB Migration.

## DB-ARCHIVE-070

Restore no hará blind INSERT.

## DB-ARCHIVE-071

Identity conflicts serán explícitos.

## DB-ARCHIVE-072

Restore no hará overwrite silencioso por default.

## DB-ARCHIVE-073

Large restore será bounded.

## DB-ARCHIVE-074

Archive Search Index será distinto del archive.

## DB-ARCHIVE-075

Search index podrá ser reconstruible.

## DB-ARCHIVE-076

Sensitive search metadata se minimizará.

## DB-ARCHIVE-077

Incremental archival será distinto de incremental backup.

## DB-ARCHIVE-078

Checkpoint no prometerá exactly-once.

## DB-ARCHIVE-079

Archive execution deberá ser idempotency-aware.

## DB-ARCHIVE-080

UNKNOWN upload será distinto de failed upload.

## DB-ARCHIVE-081

UNKNOWN outcomes requerirán reconciliation.

## DB-ARCHIVE-082

Partial package no habilitará source purge.

## DB-ARCHIVE-083

Package finalization requerirá todos los objetos obligatorios.

## DB-ARCHIVE-084

Manifest existence no sustituirá verification requerida.

## DB-ARCHIVE-085

Sharded archival podrá operar per-shard.

## DB-ARCHIVE-086

Global archive deberá declarar consistency model.

## DB-ARCHIVE-087

No se fingirá global snapshot.

## DB-ARCHIVE-088

Shard map generation podrá persistirse.

## DB-ARCHIVE-089

Resume validará topology generation.

## DB-ARCHIVE-090

Stable horizon será distinto de snapshot.

## DB-ARCHIVE-091

No se abrirá automáticamente una huge transaction.

## DB-ARCHIVE-092

Source mutation policy será explícita.

## DB-ARCHIVE-093

Finalized archive será immutable por default.

## DB-ARCHIVE-094

Archive update producirá nueva versión cuando corresponda.

## DB-ARCHIVE-095

Result Cache no será source authoritative por default.

## DB-ARCHIVE-096

Metadata Cache podrá reutilizarse.

## DB-ARCHIVE-097

Archive Catalog Cache no será archive truth.

## DB-ARCHIVE-098

Authorization será distinta de eligibility.

## DB-ARCHIVE-099

Eligibility será distinta de authorization.

## DB-ARCHIVE-100

Archive deletion será privileged.

## DB-ARCHIVE-101

Archive restore será privileged.

## DB-ARCHIVE-102

Storage credentials nunca aparecerán en logs.

## DB-ARCHIVE-103

Signed URLs serán tratadas como secretos temporales.

## DB-ARCHIVE-104

Audit será distinto de telemetry.

## DB-ARCHIVE-105

Audit minimizará payload sensible.

## DB-ARCHIVE-106

Telemetry labels serán bounded.

## DB-ARCHIVE-107

Per-record tracing no será default.

## DB-ARCHIVE-108

Archive execution tendrá budgets.

## DB-ARCHIVE-109

Pipeline buffers serán bounded.

## DB-ARCHIVE-110

No existirán unbounded queues.

## DB-ARCHIVE-111

Parallelism respetará ordering semantics.

## DB-ARCHIVE-112

Logical fingerprint será distinto de physical fingerprint.

## DB-ARCHIVE-113

Failure outcomes serán clasificados.

## DB-ARCHIVE-114

Storage timeout no implicará object absence.

## DB-ARCHIVE-115

Circuit breaker no redefinirá archive semantics.

## DB-ARCHIVE-116

Storage failover no será ciego.

## DB-ARCHIVE-117

Alternate target deberá cumplir required capabilities.

## DB-ARCHIVE-118

Data residency podrá ser policy requirement.

## DB-ARCHIVE-119

Unknown mandatory residency bloqueará archival.

## DB-ARCHIVE-120

Provider migration requerirá destination verification.

## DB-ARCHIVE-121

Copy será distinta de completed migration.

## DB-ARCHIVE-122

Provider implementations pasarán contract tests.

## DB-ARCHIVE-123

Corruption será detectable cuando verification lo exija.

## DB-ARCHIVE-124

Partial upload no será valid archive.

## DB-ARCHIVE-125

Crash recovery será testeable.

## DB-ARCHIVE-126

Retention integration será testeable.

## DB-ARCHIVE-127

Tenant isolation será testeable.

## DB-ARCHIVE-128

Persistent worker isolation será testeable.

## DB-ARCHIVE-129

Compiled archive policies podrán compartirse si son immutable.

## DB-ARCHIVE-130

Mutable execution state será scoped.

## DB-ARCHIVE-131

No habrá global current archive package.

## DB-ARCHIVE-132

Worker cleanup liberará streams y handles.

## DB-ARCHIVE-133

Cancellation será soportada.

## DB-ARCHIVE-134

Cancellation no convertirá package parcial en valid archive.

## DB-ARCHIVE-135

Staging objects serán distintos de finalized archives.

## DB-ARCHIVE-136

No se asumirá atomic rename en Object Storage.

## DB-ARCHIVE-137

Stale staging tendrá maintenance lifecycle.

## DB-ARCHIVE-138

Staging cleanup será distinto de archive purge.

## DB-ARCHIVE-139

Dry-run será first-class.

## DB-ARCHIVE-140

Dry-run no garantizará eligibility futura.

## DB-ARCHIVE-141

Archive decisions serán explainable.

## DB-ARCHIVE-142

Archive metadata será versionable.

## DB-ARCHIVE-143

Restore deberá validar package integrity.

## DB-ARCHIVE-144

Restore deberá validar tenant context.

## DB-ARCHIVE-145

Restore deberá validar schema compatibility.

## DB-ARCHIVE-146

Restore deberá respetar current authorization.

## DB-ARCHIVE-147

Archive source disposition será explícita.

## DB-ARCHIVE-148

KEEP será válido.

## DB-ARCHIVE-149

Purge-after-archive no será implícito.

## DB-ARCHIVE-150

Archive verification evidence será persistible.

## DB-ARCHIVE-151

Archive manifest podrá incluir policy fingerprint.

## DB-ARCHIVE-152

Archive manifest podrá incluir schema fingerprint.

## DB-ARCHIVE-153

Archive manifest podrá incluir topology context.

## DB-ARCHIVE-154

Archive manifest no contendrá encryption secrets.

## DB-ARCHIVE-155

Archive graph será distinto de ORM graph.

## DB-ARCHIVE-156

Relationship archival no será recursive cascade implícito.

## DB-ARCHIVE-157

Cyclic graphs serán planificados explícitamente.

## DB-ARCHIVE-158

History podrá archivarse independientemente.

## DB-ARCHIVE-159

Temporal coordinates se preservarán cuando sean requeridas.

## DB-ARCHIVE-160

Archive timestamp será distinto de temporal validity.

## DB-ARCHIVE-161

Source data mutation podrá invalidar archive assumptions.

## DB-ARCHIVE-162

Archive supersession será explícita.

## DB-ARCHIVE-163

Old package no se destruirá automáticamente al crear nueva versión.

## DB-ARCHIVE-164

Archive deletion tendrá su propia retention evaluation.

## DB-ARCHIVE-165

Archive lifecycle continuará después de abandonar operational DB.

## DB-ARCHIVE-166

Archive system no será un segundo filesystem.

## DB-ARCHIVE-167

Archive system no será un segundo query engine.

## DB-ARCHIVE-168

Archive system no será un segundo transaction engine.

## DB-ARCHIVE-169

Archive system no será un segundo security system.

## DB-ARCHIVE-170

Archive system no será un segundo telemetry system.

## DB-ARCHIVE-171

Archive system reutilizará infraestructura existente mediante contratos.

## DB-ARCHIVE-172

Driver no conocerá archive policies.

## DB-ARCHIVE-173

Connection no conocerá archive formats.

## DB-ARCHIVE-174

SQL Compiler no conocerá storage providers.

## DB-ARCHIVE-175

ORM no almacenará archivos directamente.

## DB-ARCHIVE-176

Query Engine no realizará uploads.

## DB-ARCHIVE-177

Storage Provider no interpretará ORM entities.

## DB-ARCHIVE-178

Archive Encoder no resolverá tenant authorization.

## DB-ARCHIVE-179

Archive Runner no modificará global runtime state.

## DB-ARCHIVE-180

El archive sólo se declarará válido cuando la evidencia requerida por su policy sea conocida.

---

# 369. Modelo formal

Sea:

```text
D
```

un dataset operacional.

Sea:

```text
P
```

una ArchivePolicy.

Sea:

```text
E(D,P,t)
```

la evaluación de elegibilidad en instante `t`.

El dataset podrá archivarse cuando:

```text
E(D,P,t) = ELIGIBLE
```

y:

```text
Authorization(D,P) = ALLOWED
```

---

# 370. Construcción del archive

Sea:

```text
R(D)
```

el stream canónico de records.

Sea:

```text
Encode(R)
```

la representación serializada.

Sea:

```text
Compress(...)
```

la compresión.

Sea:

```text
Encrypt(...)
```

el cifrado.

Entonces:

```text
Payload =
Encrypt(
    Compress(
        Encode(
            R(D)
        )
    )
)
```

cuando dichas etapas estén habilitadas.

---

# 371. Integridad

Sea:

```text
H(Payload)
```

el digest.

El manifest almacena:

```text
algorithm
digest
scope
```

---

# 372. Verification

Conceptualmente:

```text
Verified(A)
=
Exists(A)
∧ ManifestValid(A)
∧ IntegrityValid(A)
∧ RequiredObjectsPresent(A)
∧ RequiredSecurityValid(A)
```

según policy.

---

# 373. Source purge

El source sólo podrá eliminarse cuando:

```text
Verified(A)
∧ RetentionPurgeEligible(D)
∧ ¬LegalHold(D)
∧ Authorized(Purge)
```

---

# 374. Critical implication

```text
ArchiveSuccess
≠
PurgeAuthorization
```

---

# 375. Arquitectura conceptual final

```text
                     Operational Database
                              │
                              ▼
                     Candidate Discovery
                              │
                              ▼
                    Eligibility Evaluation
                              │
                              ▼
                       Archive Planner
                              │
                              ▼
                       Source Traversal
                              │
                  ┌───────────┴───────────┐
                  │                       │
                  ▼                       ▼
              Query Engine          Chunk / Stream
                  │                       │
                  └───────────┬───────────┘
                              ▼
                    Canonical Records
                              │
                              ▼
                         Encoder
                              │
                              ▼
                        Compression
                              │
                              ▼
                         Encryption
                              │
                              ▼
                         Integrity
                              │
                              ▼
                    Archive Storage
                              │
                              ▼
                         Verification
                              │
                              ▼
                       Archive Manifest
                              │
                              ▼
                       Archive Catalog
                              │
                   ┌──────────┴───────────┐
                   │                      │
                   ▼                      ▼
             Source Retained       Retention Recheck
                                          │
                                          ▼
                                      Source Purge
```

---

# 376. Recuperación conceptual

```text
Archive Catalog
      │
      ▼
Archive Package
      │
      ▼
Storage Provider
      │
      ▼
Integrity Verification
      │
      ▼
Decrypt
      │
      ▼
Decompress
      │
      ▼
Decode
      │
      ▼
Canonical Archive Records
      │
      ├─────────────► Preview
      │
      ├─────────────► Search
      │
      └─────────────► Restore Planner
                            │
                            ▼
                      Schema Adaptation
                            │
                            ▼
                     Persistence Engine
                            │
                            ▼
                    Operational Database
```

---

# 377. Integración con VoltStack Database

```text
Data Archival System
        │
        ├── Query Engine
        ├── Type System
        ├── Metadata System
        ├── ORM
        ├── Hydration
        ├── Relationships
        ├── Chunk Processing
        ├── Lazy Collections
        ├── Bulk Operations
        ├── Transaction System
        ├── Retention System
        ├── History System
        ├── Temporal Data System
        ├── Soft Delete System
        ├── Security
        ├── Audit
        ├── Telemetry
        ├── Resilience
        ├── Resource Governance
        ├── Multitenancy
        ├── Sharding
        └── Filesystem / Object Storage adapters
```

---

# 378. Dependencias prohibidas

No deberá existir:

```text
Driver
  ↓
Archive Policy
```

ni:

```text
SQL Compiler
  ↓
S3 Provider
```

ni:

```text
Archive Storage
  ↓
ORM Entity Manager
```

---

# 379. Dirección correcta

```text
Archive
   ↓
Query / Persistence abstractions
   ↓
Execution
   ↓
Connection
   ↓
Driver
```

Y para storage:

```text
Archive
   ↓
Archive Storage Contract
   ↓
Provider Adapter
   ↓
External Storage
```

---

# 380. Decisión arquitectónica final

VoltStack implementará Data Archival como un subsistema explícito de lifecycle para trasladar información desde almacenamiento operacional hacia almacenamiento histórico de largo plazo sin perder las garantías necesarias para comprender, verificar y recuperar esa información posteriormente.

El flujo general será:

```text
Operational Data
      ↓
Archive Candidate
      ↓
Eligibility
      ↓
Archive Policy
      ↓
Archive Plan
      ↓
Bounded Extraction
      ↓
Canonical Representation
      ↓
Encoding
      ↓
Compression
      ↓
Encryption
      ↓
Integrity
      ↓
Archive Storage
      ↓
Verification
      ↓
Manifest
      ↓
Catalog
      ↓
Archive Finalization
      ↓
Retention Re-evaluation
      ↓
Optional Source Purge
```

El principio rector será:

> **Un dato no se considerará archivado simplemente porque haya sido copiado fuera de la base de datos. VoltStack sólo considerará válido un archive cuando el contenido, su identidad, metadata, integridad, seguridad y requisitos de verificación exigidos por la política hayan sido satisfechos explícitamente.**

Complementariamente:

> **Archivar y destruir serán siempre decisiones independientes: un archive válido puede coexistir con el dato operacional, y la existencia de un archive nunca otorgará por sí misma permiso para eliminar el original.**

Finalmente:

> **La arquitectura deberá permitir que un archive creado hoy siga siendo interpretable, verificable y recuperable años después, incluso cuando el esquema, las entidades, los drivers, el proveedor de almacenamiento o la propia versión de VoltStack hayan evolucionado.**

---

# 381. Bloque 27 — progreso

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES

✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
✓ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
✓ 269_DATABASE_SOFT_DELETE_SYSTEM.md
✓ 270_DATABASE_DATA_RETENTION_SYSTEM.md
✓ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
□ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
□ 273_DATABASE_JSON_QUERY_SYSTEM.md
□ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
□ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 382. Siguiente documento

```text
272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
```

El siguiente documento definirá la arquitectura de búsqueda textual avanzada de VoltStack Database, incluyendo:

```text
Full-Text Search Architecture
Search Documents
Search Fields
Search Expressions
Search Query AST
Search Predicates
Tokenization
Normalization
Language Configuration
Stemming
Stop Words
Phrase Search
Boolean Search
Prefix Search
Ranking
Relevance Scores
Highlighting
Search Metadata
Search Indexes
Database-Native Full-Text Search
MySQL / MariaDB FULLTEXT
PostgreSQL tsvector / tsquery
SQLite FTS
Capability Detection
Portable Search Semantics
Platform-Specific Extensions
Query Builder Integration
ORM Integration
Pagination
Cursor Pagination
Security
Multitenancy
Sharding
Telemetry
Performance
Testing
```

bajo el principio:

> **Full-Text Search será una extensión semántica del Query Engine y no un conjunto de fragmentos SQL específicos del proveedor expuestos directamente al ORM o al desarrollador.**