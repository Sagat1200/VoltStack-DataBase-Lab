# 207_DATABASE_EXPORT_SYSTEM.md

# VoltStack Quantum Database
## Database Export System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 207 — Database Export System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `206_DATABASE_IMPORT_SYSTEM.md`  
**Siguiente documento:** `208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md`

---

# 1. Propósito

`Database Export System` define la arquitectura mediante la cual VoltStack podrá extraer datasets potencialmente masivos desde el sistema de base de datos y convertirlos en representaciones externas mediante un pipeline explícito, seguro, observable, cancelable, extensible y consciente de recursos.

Ejemplo:

```php
$result = DB::export()
    ->from(
        DB::table('customers')
            ->where('active', true)
            ->orderBy('id')
    )
    ->toCsv('/exports/customers.csv')
    ->columns([
        'id',
        'name',
        'email',
        'created_at',
    ])
    ->batchSize(5000)
    ->run();
```

Desde Model API:

```php
User::query()
    ->where('active', true)
    ->export()
    ->toCsv('/exports/users.csv')
    ->run();
```

Desde Repository:

```php
$repository
    ->queryActiveUsers()
    ->export()
    ->toNdjson($stream)
    ->run();
```

La regla central será:

> **Exportar datos en VoltStack no será equivalente a ejecutar una consulta y serializar todo su resultado; Export será un pipeline de extracción, recorrido, proyección, transformación, serialización y entrega capaz de operar sobre datasets potencialmente masivos sin exigir su materialización completa.**

Formalmente:

```text
Export
=
Source Query
+
Traversal
+
Projection
+
Hydration
+
Transformation
+
Serialization
+
Encoding
+
Destination
+
Progress
+
Security
+
Resource Governance
```

Nunca:

```text
Export
=
query()->get()->toArray()
```

para el caso general.

---

# 2. Posición arquitectónica

```text
199 Pagination
200 Cursor Pagination
201 Chunk Processing
202 Lazy Collection

203 Bulk Insert
204 Bulk Update
205 Bulk Delete

206 Import
207 Export
208 Large Dataset Processing
```

La dirección de Import es:

```text
External Source
      ↓
Database
```

Export implementa:

```text
Database
      ↓
External Destination
```

Ambos comparten algunos conceptos:

```text
streaming
batching
progress
checkpoints
resource governance
mapping
transformation
telemetry
```

pero no son el mismo sistema.

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Export
≠
Query
≠
Pagination
≠
Chunk Processing
≠
Lazy Collection
≠
Result Cursor
≠
Streaming Result
≠
Serialization
≠
Backup
≠
Database Dump
≠
Replication
```

---

# 4. Export vs Query

Query responde:

> ¿Qué datos deben obtenerse?

Export responde:

> ¿Cómo recorrer, transformar, representar y entregar esos datos?

Por tanto:

```text
Query
→ logical dataset

Export
→ external representation of that dataset
```

---

# 5. Export vs Backup

Backup busca preservar un estado restaurable de la base de datos.

Export produce una representación de datos para:

```text
integration
analytics
reporting
migration
user downloads
interchange
archival workflows
```

Por tanto:

```text
Export
≠
Backup
```

---

# 6. Export vs Database Dump

Un database dump puede incluir:

```text
DDL
indexes
constraints
sequences
database-specific metadata
SQL statements
```

Export trabaja principalmente con:

```text
logical application datasets
```

---

# 7. Export vs Serialization

Serialization es únicamente una fase:

```text
Export
    ↓
Serialization
```

No controla:

```text
query traversal
resource ownership
authorization
routing
checkpointing
progress
distributed execution
```

---

# 8. Arquitectura general

```text
Export API
    │
    ▼
ExportRequest
    │
    ▼
ExportPlanner
    │
    ├── Source Query
    ├── Projection
    ├── Traversal
    ├── Consistency
    ├── Routing
    ├── Transformation
    ├── Serialization
    ├── Destination
    ├── Security
    ├── Recovery
    └── Resources
    │
    ▼
ExportPlan
    │
    ▼
ExportRunner
    │
    ▼
Database Source
    │
    ▼
Traversal Engine
    │
    ├── Chunk
    ├── Keyset
    ├── Lazy
    ├── Streaming
    └── Custom
    │
    ▼
Rows / Projections / Entities
    │
    ▼
Export Mapper
    │
    ▼
Transformer
    │
    ▼
Serializer
    │
    ▼
Encoder
    │
    ▼
Destination Writer
    │
    ▼
ExportResult
```

---

# 9. Pipeline fundamental

```text
Query
↓
Plan
↓
Execute incrementally
↓
Read record
↓
Project
↓
Transform
↓
Mask / Redact
↓
Serialize
↓
Encode
↓
Write
↓
Checkpoint / Progress
```

---

# 10. ExportRequest

Modelo conceptual:

```php
final readonly class ExportRequest
{
    public function __construct(
        public ExportSource $source,
        public ExportDestination $destination,
        public ExportFormat $format,
        public ExportOptions $options,
    ) {}
}
```

---

# 11. Export source

La fuente normal será una definición lógica de consulta.

```php
interface ExportSource
{
    public function createQuery(
        ExportSourceContext $context,
    ): QueryModel;
}
```

---

# 12. Query Model como frontera

Export no deberá recibir SQL como representación interna principal.

Preferirá:

```text
Query Builder
↓
Query Model / AST
↓
Export Planner
```

Esto conserva:

```text
portability
security
routing
semantic analysis
optimizer
compiler separation
```

---

# 13. Raw SQL

Podrá existir como escape hatch bajo las reglas generales del Query Engine.

Pero Raw SQL puede reducir la capacidad del Export Planner para inferir:

```text
stable ordering
identifier
tenant
shard
projection metadata
resume boundary
```

---

# 14. Sources soportadas

Conceptualmente:

```text
QueryExportSource
EntityQueryExportSource
RepositoryExportSource
TableExportSource
CustomExportSource
```

---

# 15. Table export

```php
DB::export()
    ->fromTable('customers')
    ->toCsv($destination)
    ->run();
```

será convenience API.

Internamente:

```text
fromTable()
↓
Query Model
↓
normal Export pipeline
```

---

# 16. Model API

```php
User::query()
    ->where('status', 'active')
    ->export()
    ->toCsv($path)
    ->run();
```

No crea un segundo export engine.

---

# 17. Repository API

```php
$users
    ->queryForExport($criteria)
    ->export()
    ->toJson($stream)
    ->run();
```

converge en el mismo:

```text
ExportRequest
→ ExportPlanner
→ ExportRunner
```

---

# 18. Source Query ≠ Active Query Execution

Crear:

```php
$export = User::query()->export();
```

no deberá ejecutar inmediatamente la consulta.

---

# 19. Deferred execution

La ejecución ocurrirá en:

```php
$export->run();
```

o mediante un mecanismo explícito equivalente.

---

# 20. Query snapshot

La definición utilizada por el export deberá ser inmutable o capturada de manera estable.

Evitar:

```php
$query = User::query();

$export = $query->export();

$query->where('admin', true);

$export->run();
```

con semántica ambigua.

---

# 21. Export Projection

Export deberá poder seleccionar explícitamente qué información sale del sistema.

```php
->columns([
    'id',
    'name',
    'email',
])
```

---

# 22. Projection ≠ Physical Query Projection

El Query Planner puede necesitar columnas internas adicionales:

```text
identifier
ordering keys
shard key
cursor boundary
version token
```

sin exportarlas.

Por tanto:

```text
Logical Export Projection
≠
Physical Execution Projection
```

---

# 23. Hidden execution columns

Ejemplo:

```text
Export:
    name
    email

Internal traversal:
    name
    email
    id
```

`id` podrá usarse como continuation key sin aparecer en el archivo.

---

# 24. Export field model

```php
final readonly class ExportField
{
    public function __construct(
        public string $source,
        public string $output,
        public ?ExportValueFormatter $formatter = null,
    ) {}
}
```

---

# 25. Field aliases

```php
->map([
    'customer_id' => 'ID',
    'name'        => 'Customer Name',
    'email'       => 'Email Address',
])
```

---

# 26. Mapping direction

En Export:

```text
Internal Field
→
External Field
```

En Import:

```text
External Field
→
Internal Field
```

---

# 27. Traversal architecture

Export deberá reutilizar sistemas ya definidos.

```text
Export
↓
Traversal Strategy
├── Chunk Processing
├── Cursor/Keyset
├── Lazy Collection
├── Streaming Result
└── Custom
```

---

# 28. No duplicate traversal engine

Export no deberá implementar nuevamente:

```text
offset traversal
keyset predicates
cursor semantics
chunk lifecycle
streaming cursor
```

---

# 29. ExportTraversalStrategy

```php
enum ExportTraversalStrategy
{
    case AUTO;
    case CHUNK;
    case KEYSET;
    case LAZY;
    case STREAM;
    case CUSTOM;
}
```

---

# 30. AUTO

`AUTO` permitirá al planner elegir una estrategia compatible.

La elección deberá ser:

```text
deterministic
explainable
capability-driven
```

---

# 31. AUTO ≠ hidden semantic change

El planner podrá elegir una implementación distinta solo cuando preserve la semántica requerida.

---

# 32. Chunk strategy

```text
Query
↓
Chunk 1
↓
Serialize/write
↓
release
↓
Chunk 2
```

Será una opción general robusta.

---

# 33. Keyset strategy

Preferible cuando:

```text
large dataset
stable ordering
suitable key
resume needed
mutable dataset
high offset would be expensive
```

---

# 34. Lazy strategy

Puede reutilizar:

```text
202_DATABASE_LAZY_COLLECTION_SYSTEM.md
```

como abstracción de consumo.

---

# 35. Streaming strategy

Puede reutilizar:

```text
082_DATABASE_RESULT_CURSOR_SYSTEM.md
083_DATABASE_STREAMING_RESULT_SYSTEM.md
```

---

# 36. Streaming tradeoff

Streaming puede mantener:

```text
connection
statement
server cursor
transaction
```

abiertos durante mucho tiempo.

Por ello no será automáticamente la estrategia universal.

---

# 37. Chunk vs Stream

```text
Chunk
=
bounded discrete queries

Stream
=
potentially long-lived result resource
```

---

# 38. Resume support

Keyset/chunk-backed export suele ofrecer mejor base para resumability.

Streaming no será resumable por default.

---

# 39. Stable ordering

Un export reanudable necesita normalmente:

```text
stable deterministic ordering
```

---

# 40. Ordering inference

Deberá reutilizar las reglas de:

```text
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
```

---

# 41. Tie-breakers

Si:

```text
ORDER BY created_at
```

no es único, podrá inferirse:

```text
ORDER BY created_at, id
```

cuando metadata permita demostrarlo.

---

# 42. No heuristic `"id"`

No asumir que toda tabla tiene:

```text
id
```

como identificador.

---

# 43. Composite identifiers

Soportar:

```text
tenant_id
+
invoice_number
```

o cualquier identidad compuesta reconocida.

---

# 44. Export consistency

Export deberá modelar qué consistencia necesita.

```php
enum ExportConsistency
{
    case BEST_EFFORT;
    case READ_YOUR_WRITES;
    case MONOTONIC;
    case SNAPSHOT;
    case CUSTOM;
}
```

---

# 45. BEST_EFFORT

Los diferentes chunks pueden observar diferentes momentos de la base.

---

# 46. SNAPSHOT

Busca una visión consistente durante todo el export cuando la plataforma y estrategia lo permitan.

---

# 47. Snapshot tradeoffs

Puede implicar:

```text
long transaction
MVCC retention
connection pinning
replica constraints
resource pressure
```

---

# 48. No implicit snapshot

Export no abrirá una transacción larga automáticamente solo porque el dataset sea grande.

---

# 49. Same query ≠ same dataset over time

Siempre:

```text
Same Export Definition
≠
Same Export Result Forever
```

---

# 50. Mutation during export

Durante un export pueden ocurrir:

```text
INSERT
UPDATE
DELETE
```

con efectos diferentes según traversal/consistency.

---

# 51. Export horizon

Podrá existir:

```php
enum ExportHorizonPolicy
{
    case LIVE;
    case CAPTURE_UPPER_BOUND;
    case SNAPSHOT;
    case CUSTOM;
}
```

---

# 52. LIVE

Puede incluir datos agregados después de iniciar si aparecen más allá de la continuación actual.

---

# 53. CAPTURE_UPPER_BOUND

Captura un límite lógico inicial cuando sea posible.

Ejemplo:

```text
max(id) at export start
```

pero solo cuando esa estrategia sea semánticamente válida.

---

# 54. Upper bound ≠ snapshot

Aunque el límite máximo quede fijo:

```text
existing rows can still change
existing rows can still disappear
```

---

# 55. Hydration modes

Export podrá trabajar con:

```text
RAW_ROW
TYPED_ROW
SCALAR
TUPLE
PROJECTION
DTO
ENTITY
```

---

# 56. Default para grandes exports

Preferir:

```text
TYPED_ROW
```

o:

```text
PROJECTION
```

antes que entidades completas.

---

# 57. Entity hydration cost

Entity mode implica potencialmente:

```text
IdentityMap
UnitOfWork
lifecycle
relationships
more memory
```

---

# 58. Export ≠ ORM serialization

No deberá hacer:

```text
Entity
→ serialize all properties recursively
```

por default.

---

# 59. Relationship recursion

Podría causar:

```text
N+1
cycles
massive graphs
lazy loading
security leaks
```

---

# 60. Relationship export

Las relaciones deberán ser explícitas.

Ejemplo:

```php
->fields([
    'id',
    'name',
    'company.name',
])
```

y el planner decidirá cómo obtenerlas.

---

# 61. N+1 protection

El N+1 Detection System seguirá activo cuando Export utilice ORM relationships.

---

# 62. Lazy loading

No deberá activarse accidentalmente por serializers genéricos.

---

# 63. Serialization architecture

```text
Export Record
↓
Serializer
↓
Encoded Representation
↓
Destination Writer
```

---

# 64. Serializer

```php
interface ExportSerializer
{
    public function serialize(
        ExportRecord $record,
        ExportSerializationContext $context,
    ): ExportSerializedRecord;
}
```

---

# 65. Serializer ≠ Destination

Serializer decide:

```text
representation
```

Destination decide:

```text
where bytes go
```

---

# 66. Supported formats

Core inicial:

```text
CSV
JSON
NDJSON
```

Extensiones futuras:

```text
XML
Parquet
Arrow
Excel
Avro
custom
```

sin convertirlas en dependencias obligatorias del core.

---

# 67. CSV

```php
->toCsv($path)
```

---

# 68. CSV configuration

```text
delimiter
enclosure
escape
newline
header
encoding
BOM
null representation
```

---

# 69. CSV injection

Export a CSV deberá considerar spreadsheet formula injection.

Valores que comienzan con:

```text
=
+
-
@
```

pueden ser interpretados como fórmulas por algunas aplicaciones.

---

# 70. CSV security policy

Podrá existir:

```php
enum CsvFormulaPolicy
{
    case PRESERVE;
    case ESCAPE;
    case REJECT;
}
```

---

# 71. Default

Para exports destinados a hojas de cálculo:

```text
ESCAPE
```

podrá ser secure default.

La policy deberá ser explícita/configurable.

---

# 72. JSON export

Podrán existir dos formas principales:

```text
JSON_ARRAY
NDJSON
```

---

# 73. JSON array

Produce:

```json
[
  {"id":1,"name":"A"},
  {"id":2,"name":"B"}
]
```

---

# 74. JSON streaming

El writer deberá poder producir:

```text
[
record,
record,
record
]
```

incrementalmente sin construir el array PHP completo.

---

# 75. NDJSON

Produce:

```text
{"id":1,"name":"A"}
{"id":2,"name":"B"}
```

Particularmente adecuado para:

```text
large datasets
stream processing
pipelines
machine ingestion
```

---

# 76. Serialization failure

Un error en un record deberá preservar:

```text
record position
export sequence
field if known
phase
```

---

# 77. Type serialization

Deberá integrarse con:

```text
155_DATABASE_TYPE_SYSTEM.md
157_DATABASE_VALUE_CONVERSION_SYSTEM.md
```

pero:

```text
Database Conversion
≠
External Serialization
```

---

# 78. Date/time export

Policies explícitas:

```text
ISO-8601
custom format
timezone conversion
UTC normalization
```

---

# 79. Temporal semantics

Nunca perder silenciosamente distinciones entre:

```text
Instant
LocalDate
LocalTime
LocalDateTime
ZonedDateTime
```

---

# 80. Decimal export

Evitar convertir automáticamente decimals de alta precisión a:

```text
binary float
```

si puede perderse precisión.

---

# 81. Large integers

JSON puede ser consumido por plataformas con límites numéricos distintos.

Podrá existir:

```text
NUMBER
STRING_IF_UNSAFE
STRING
```

como policy.

---

# 82. Enum export

Podrá exportarse:

```text
backing value
logical name
custom external value
```

según formatter.

---

# 83. Value Object

Los Value Objects requerirán formatter/export serializer registrado.

---

# 84. NULL

Debe distinguirse de:

```text
empty string
0
false
missing field
```

---

# 85. CSV null representation

Configurable:

```text
empty
NULL
\N
custom
```

---

# 86. JSON null

Deberá producir:

```json
null
```

cuando el valor lógico sea null.

---

# 87. Transformation pipeline

Export podrá transformar registros antes de serializarlos.

```php
->transform(function (ExportRecord $record) {
    // ...
})
```

---

# 88. Transformation ≠ Query Projection

Si puede expresarse correctamente en Query AST, puede ser pushdown.

Si es código PHP arbitrario:

```text
execute DB query
↓
transform application-side
```

---

# 89. Pushdown

Ejemplo:

```text
LOWER(email)
```

podría realizarse en DB si existe representación semántica compatible.

Pero el Export System no analizará callbacks arbitrarios intentando convertirlos en SQL.

---

# 90. Transformation determinism

Importante para resumable export.

Si:

```text
exported_at = now()
```

un resume puede generar valores distintos.

---

# 91. Stable Export Clock

Podrá existir:

```text
ExportClock
```

fijado al inicio de la operación.

---

# 92. Security architecture

Export es especialmente sensible porque extrae información fuera del sistema.

La autorización deberá ocurrir:

```text
before export execution
```

---

# 93. Authorization layers

```text
Export Permission
↓
Dataset Scope
↓
Tenant Scope
↓
Row Scope
↓
Field Scope
↓
Transformation/Masking
↓
Destination Permission
```

---

# 94. Authorization before traversal

Nunca:

```text
export all
↓
remove unauthorized rows
```

---

# 95. Correct approach

```text
Authorization Scope
↓
Query
↓
Traversal
```

---

# 96. Field authorization

Un usuario puede poder exportar:

```text
name
email
```

pero no:

```text
password_hash
api_secret
internal_risk_score
```

---

# 97. Sensitive field policy

Campos podrán clasificarse:

```text
PUBLIC
INTERNAL
CONFIDENTIAL
SENSITIVE
SECRET
```

como integración futura con metadata/security.

---

# 98. Redaction

```php
->redact([
    'email' => Redaction::EMAIL,
])
```

---

# 99. Masking

Ejemplo:

```text
john@example.com
→
j***@example.com
```

---

# 100. Redaction ≠ Authorization

La autorización decide si el dato puede ser usado.

Redaction decide cómo puede salir.

---

# 101. Encryption

Export podrá integrarse con destination encryption.

Pero:

```text
Encryption
≠
Authorization
```

---

# 102. Audit

Export de datos sensibles deberá poder generar eventos auditables:

```text
who
what dataset
when
tenant
field categories
destination category
record count
result
```

sin registrar el dataset completo.

---

# 103. Destination architecture

```php
interface ExportDestination
{
    public function open(
        ExportDestinationContext $context,
    ): ExportDestinationWriter;
}
```

---

# 104. Destination Writer

```php
interface ExportDestinationWriter
{
    public function write(
        ExportEncodedChunk $chunk,
    ): void;

    public function checkpoint(): ?ExportDestinationCheckpoint;

    public function close(): void;
}
```

---

# 105. Destination types

Core:

```text
FileDestination
StreamDestination
CallbackDestination
```

Extensiones:

```text
S3Destination
R2Destination
AzureBlobDestination
GoogleCloudStorageDestination
SftpDestination
HttpDestination
```

---

# 106. Core independence

`Quantum/Database` no deberá depender obligatoriamente de SDKs cloud concretos.

---

# 107. Destination capabilities

```php
enum ExportDestinationCapability
{
    case STREAMABLE;
    case APPENDABLE;
    case SEEKABLE;
    case RESUMABLE;
    case ATOMIC_FINALIZE;
    case MULTIPART;
}
```

---

# 108. Appendable destination

Puede continuar escribiendo después de un checkpoint.

Pero:

```text
Appendable
≠
Safely Resumable
```

---

# 109. Atomic finalize

Algunas destinations podrán:

```text
write temporary object
↓
complete
↓
publish final object
```

---

# 110. Partial output

Si falla un export:

```text
50 GB expected
17 GB written
failure
```

el sistema deberá saber si el destino contiene:

```text
PARTIAL
ABORTED
FINALIZED
UNKNOWN
```

---

# 111. Destination state

```php
enum ExportDestinationState
{
    case NEW;
    case OPEN;
    case WRITING;
    case FINALIZED;
    case ABORTED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 112. Temporary file pattern

Para filesystem:

```text
customers.csv.part
↓
write
↓
fsync/close according policy
↓
rename
↓
customers.csv
```

cuando la plataforma lo permita.

---

# 113. Rename ≠ universally atomic

Las garantías dependen de:

```text
filesystem
mount
storage provider
operation
```

No asumir atomicidad universal.

---

# 114. Existing destination policy

```php
enum ExistingDestinationPolicy
{
    case FAIL;
    case OVERWRITE;
    case APPEND;
    case VERSION;
    case CUSTOM;
}
```

---

# 115. Default seguro

```text
FAIL
```

para evitar sobrescritura accidental.

---

# 116. Append

No será válido para todos los formatos.

Por ejemplo, append directo a un JSON array finalizado puede romper su estructura.

---

# 117. Format capability

El planner deberá validar compatibilidad:

```text
Format
×
Destination
×
Resume Strategy
```

---

# 118. Compression

Podrá aplicarse:

```text
Serialization
↓
Encoding
↓
Compression
↓
Destination
```

---

# 119. Compression algorithms

Extensiones:

```text
gzip
zstd
bzip2
zip
custom
```

---

# 120. Compression ≠ Export Format

```text
CSV + gzip
JSON + gzip
NDJSON + zstd
```

---

# 121. Resume + compression

Puede ser complejo.

Un stream gzip monolítico no necesariamente permite continuar limpiamente desde un byte arbitrario.

---

# 122. Planner validation

Si:

```text
resume required
+
non-resumable compression
```

deberá:

```text
reject
```

o elegir una estrategia compatible explícita.

---

# 123. Multipart output

Para datasets enormes podrá existir:

```text
users-part-00001.csv
users-part-00002.csv
...
```

---

# 124. Export partitioning

```php
->rotateEveryRecords(1_000_000)
```

o:

```php
->rotateEveryBytes(512 * 1024 * 1024)
```

---

# 125. File rotation

Permite:

```text
bounded files
parallel upload
better resume
operational manageability
```

---

# 126. Manifest

Un export multipart podrá producir:

```text
ExportManifest
```

con:

```text
export id
format
parts
record ranges
checksums
schema
status
```

---

# 127. Manifest ≠ checkpoint

Manifest describe outputs.

Checkpoint describe continuation/recovery.

---

# 128. Checksums

Cada parte podrá calcular:

```text
SHA-256
```

u otro algoritmo registrado.

---

# 129. Checksum purpose

Sirve para:

```text
integrity verification
transfer verification
manifest validation
```

No equivale a firma/autenticidad.

---

# 130. Checkpoint architecture

Un export resumable podrá almacenar:

```text
version
ExportId
source/query fingerprint
ordering fingerprint
continuation boundary
destination identity
destination checkpoint
format
serializer version
transform fingerprint
security scope fingerprint
tenant
shard topology generation
records exported
bytes written
parts completed
```

---

# 131. Checkpoint ≠ exact snapshot

Un checkpoint indica:

```text
where traversal stopped
```

no:

```text
database snapshot state
```

---

# 132. Query fingerprint

Deberá ser semántico.

No:

```text
hash(compiled SQL string)
```

como identidad principal.

---

# 133. Security scope fingerprint

Si cambian permisos entre pause y resume:

```text
resume may no longer be valid
```

---

# 134. Resume authorization

Todo resume deberá volver a comprobar autorización.

Nunca asumir:

```text
authorized yesterday
→ authorized forever
```

---

# 135. Destination fingerprint

Debe detectar cuando el output esperado ya no coincide.

---

# 136. Resume safety

```php
enum ExportResumeSafety
{
    case SAFE;
    case REQUIRES_DESTINATION_VERIFICATION;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 137. Resume boundary

Para keyset:

```text
last confirmed exported ordering tuple
```

podrá servir como continuación.

---

# 138. Confirmed exported

No basta con:

```text
record fetched
```

Debe haberse alcanzado el durability boundary requerido del destino.

---

# 139. Checkpoint order

Conceptualmente:

```text
Read records
↓
Serialize
↓
Write destination
↓
Confirm required destination state
↓
Persist checkpoint
```

---

# 140. Checkpoint failure after output write

Puede provocar replay.

Por ello destination/resume strategy deberá saber manejar duplicación o truncación/verification.

---

# 141. Exactly once

Siempre:

```text
Checkpoint
+
Resume
≠
Exactly Once
```

---

# 142. ExportId

Toda operación durable podrá tener:

```text
ExportId
```

---

# 143. AttemptId

Separar:

```text
ExportId
ExecutionAttemptId
```

---

# 144. Progress

Modelo:

```php
final readonly class ExportProgress
{
    public function __construct(
        public int $recordsRead,
        public int $recordsExported,
        public int $partsCompleted,
        public int $bytesWritten,
        public ?int $estimatedTotalRecords,
        public ?int $estimatedTotalBytes,
    ) {}
}
```

---

# 145. Unknown totals

Un export no necesita ejecutar:

```text
COUNT(*)
```

por default.

---

# 146. Count cost

Para consultas complejas:

```text
exact count
```

puede ser tan caro como una parte significativa del propio export.

---

# 147. Progress policy

```php
enum ExportProgressEstimation
{
    case NONE;
    case EXACT_COUNT;
    case ESTIMATED_COUNT;
    case CUSTOM;
}
```

---

# 148. Exact count ≠ same snapshot

Si count y export no comparten snapshot:

```text
count = 1,000,000
```

no garantiza que exactamente ese número sea exportado.

---

# 149. Progress percentage

Deberá expresar confidence.

Ejemplo:

```text
Records: 450,000
Estimated Total: ~1,000,000
Progress: ~45%
```

no como certeza absoluta.

---

# 150. Cancellation

Export deberá integrar:

```text
CancellationToken
Deadline
```

---

# 151. Cancellation lifecycle

```text
RUNNING
↓
Cancellation requested
↓
finish safe boundary
↓
close source
↓
finalize/abort destination according policy
↓
CANCELLED
```

---

# 152. Immediate cancellation

En algunas fuentes/Drivers podrá intentarse:

```text
query cancellation
```

utilizando:

```text
084_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
```

---

# 153. Cancellation ≠ rollback

Los bytes ya entregados a un destino externo no se deshacen mediante DB rollback.

---

# 154. Destination abort

Si es soportado:

```text
multipart upload
→ abort
```

podrá limpiar partes temporales.

---

# 155. Abort failure

Puede dejar recursos externos huérfanos.

El resultado deberá reportarlo.

---

# 156. Export state machine

```text
CREATED
   ↓
PLANNED
   ↓
RUNNING
   ├──→ PAUSED
   ├──→ CANCELLED
   ├──→ FAILED
   ├──→ UNKNOWN
   └──→ FINALIZING
            ↓
         COMPLETED
```

---

# 157. PAUSED

Solo deberá producirse si existe:

```text
valid checkpoint
+
safe source continuation
+
compatible destination continuation
```

---

# 158. PARTIAL

Podrá existir cuando:

```text
some output finalized
+
remaining export failed
```

especialmente en multipart.

---

# 159. UNKNOWN

Si no puede saberse si una destination confirmó una operación:

```text
UNKNOWN
```

deberá preservarse.

---

# 160. Result model

```php
final readonly class ExportResult
{
    public function __construct(
        public ExportStatus $status,
        public ExportProgress $progress,
        public ExportOutputSummary $output,
        public ?ExportCheckpoint $checkpoint,
        public ExportOutcomeEvidence $evidence,
    ) {}
}
```

---

# 161. Status

```php
enum ExportStatus
{
    case COMPLETED;
    case PARTIAL;
    case PAUSED;
    case CANCELLED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 162. Resource governance

Export deberá tener límites explícitos.

```php
final readonly class ExportResourceBudget
{
    public function __construct(
        public ?int $maxRecords,
        public ?int $maxBytes,
        public ?int $maxDurationMs,
        public ?int $maxMemoryBytes,
        public ?int $maxParts,
        public ?int $maxOpenResources,
    ) {}
}
```

---

# 163. Memory target

Para traversal incremental:

```text
M_export
≈
M(source batch)
+
M(transformation buffer)
+
M(serialization buffer)
+
M(destination buffer)
```

Idealmente:

```text
O(batch size)
```

---

# 164. Bounded memory ≠ guaranteed constant memory

Puede crecer por:

```text
ORM IdentityMap
callbacks retaining references
relationship graphs
serializer buffering
destination SDK buffering
dedup state
compression buffers
```

---

# 165. ORM memory policy

Si se exportan entidades:

```php
enum ExportOrmMemoryPolicy
{
    case KEEP_MANAGED;
    case DETACH_EXPORTED;
    case CLEAR_ENTITY_TYPE;
    case DEDICATED_ENTITY_MANAGER;
    case CALLER_MANAGED;
}
```

---

# 166. No silent clear

Export no deberá hacer:

```php
$entityManager->clear();
```

sobre un EntityManager ajeno sin policy explícita.

---

# 167. Dedicated EntityManager

Para exports largos podrá ser una estrategia recomendable.

---

# 168. Result Cache

Default recomendado para grandes exports:

```text
BYPASS
```

---

# 169. Razón

Un export masivo no debería:

```text
fill result cache
```

con datasets enormes que probablemente se consumen una sola vez.

---

# 170. Metadata caches

Sí podrán utilizarse normalmente:

```text
metadata cache
compiled query cache
schema metadata cache
```

---

# 171. Read routing

Export es normalmente read-only.

Pero:

```text
read-only
≠
replica-safe
```

---

# 172. Replica policy

Podrá utilizar:

```text
ANY_ELIGIBLE
PIN_ENDPOINT
MINIMUM_POSITION
WRITER_ONLY
```

---

# 173. Endpoint pinning

Para un export largo:

```text
PIN_ENDPOINT
```

puede reducir inconsistencias entre chunks.

---

# 174. Pinning ≠ snapshot

Aunque todos los chunks vayan a la misma replica:

```text
data may continue changing
```

---

# 175. Replica lag

Si el export requiere:

```text
READ_YOUR_WRITES
```

deberán respetarse:

```text
sticky connection
minimum replication position
writer routing
```

según capacidades.

---

# 176. Failover

Un failover durante export no deberá cambiar de endpoint silenciosamente si eso rompe las garantías solicitadas.

---

# 177. Resume after failover

Podrá requerir:

```text
authority epoch
topology generation
minimum observed position
```

---

# 178. Sharding

Export deberá soportar:

```text
single shard
multi-shard
global ordered export
per-shard export
```

---

# 179. Single shard

Caso más simple:

```text
Query
→ shard
→ traversal
→ output
```

---

# 180. Per-shard export

Puede producir:

```text
shard-01.ndjson
shard-02.ndjson
shard-03.ndjson
```

---

# 181. Global ordered export

Requiere:

```text
global ordering
+
per-shard traversal
+
merge
```

---

# 182. K-way merge

Conceptualmente:

```text
Shard A ─┐
Shard B ─┼→ Ordered Merge → Serializer
Shard C ─┘
```

---

# 183. Global ordering key

Debe ser comparable entre shards.

Un `local_id` aislado puede no bastar.

---

# 184. Ejemplo

```text
(created_at, shard_id, local_id)
```

puede formar una clave global estable.

---

# 185. Distributed export failure

Si:

```text
Shard A ✓
Shard B ✓
Shard C ✗
```

no deberá reportarse:

```text
COMPLETED
```

---

# 186. Cross-shard snapshot

No asumir snapshot global si no existe protocolo que lo garantice.

---

# 187. Multi-tenant export

Por default:

```text
one tenant context
```

---

# 188. Cross-tenant export

Requerirá:

```text
explicit administrative mode
+
authorization
+
auditing
```

---

# 189. Tenant context drift

No deberá cambiar a mitad de export.

---

# 190. Checkpoint tenant binding

El checkpoint deberá incluir el tenant/domain relevante.

---

# 191. Security context drift

Si el actor pierde permisos durante una operación pausada, resume deberá reevaluarlos.

---

# 192. Destination authorization

No basta con autorizar los datos.

También puede ser necesario autorizar:

```text
where they are exported
```

---

# 193. Example

Un usuario podría poder descargar un CSV pero no enviar datos sensibles directamente a:

```text
arbitrary external HTTP endpoint
```

---

# 194. Destination policy

Podrá clasificar destinos:

```text
LOCAL_CONTROLLED
APPROVED_STORAGE
APPROVED_EXTERNAL
ARBITRARY_EXTERNAL
```

---

# 195. SSRF

HTTP destinations configurables no deberán convertirse en mecanismo SSRF.

La integración HTTP deberá aplicar las políticas de seguridad correspondientes.

---

# 196. Path traversal

File destinations deberán impedir:

```text
../../etc/passwd
```

cuando el path provenga de input no confiable.

---

# 197. Symlink considerations

Filesystem adapter deberá considerar policies sobre:

```text
symlinks
allowed roots
overwrite
ownership
permissions
```

---

# 198. Export filename

El Database Export System no deberá confiar ciegamente en nombres proporcionados por usuarios.

---

# 199. Encryption at destination

Podrá integrarse con:

```text
filesystem encryption
object-storage encryption
application-level encryption
```

sin acoplar el core a un proveedor.

---

# 200. Compression bomb

Es principalmente un riesgo de Import.

En Export, el riesgo equivalente será:

```text
unexpected compression memory/CPU amplification
```

que deberá gobernarse mediante resource budgets.

---

# 201. Backpressure

Una destination lenta no deberá provocar crecimiento ilimitado de memoria.

---

# 202. Pipeline backpressure

```text
Database Source
      ↓
 bounded buffer
      ↓
 Serializer
      ↓
 bounded buffer
      ↓
 Slow Destination
```

deberá ralentizar upstream.

---

# 203. No unbounded queue

Nunca:

```text
read database as fast as possible
↓
store millions of pending records in RAM
↓
slow destination
```

---

# 204. Async export

Una implementación futura podrá utilizar concurrencia.

Pero:

```text
Concurrency
≠
Unbounded Parallelism
```

---

# 205. Pipeline parallelism

Posible:

```text
Reader
  ↓
Transform Workers
  ↓
Ordered Serializer
  ↓
Writer
```

cuando se preserve ordering.

---

# 206. Ordering with parallel transforms

Cada record podrá recibir:

```text
sequence number
```

para restaurar orden antes de escribir.

---

# 207. Out-of-order mode

Podrá permitirse solo si:

```text
format
consumer contract
export semantics
```

lo permiten explícitamente.

---

# 208. Default

Preservar orden lógico.

---

# 209. Parallel shard export

Puede ser especialmente útil en:

```text
PER_SHARD
```

sin requerir merge global.

---

# 210. Database concurrency pressure

Más workers no implica mejor rendimiento.

Podría saturar:

```text
connection pool
replicas
disk I/O
network
destination
```

---

# 211. Adaptive throughput

Una futura extensión podrá ajustar:

```text
batch size
parallelism
buffer size
```

según telemetry/resource pressure.

No deberá cambiar semántica.

---

# 212. Export schema

El output puede tener un schema lógico.

```php
final readonly class ExportSchema
{
    public function __construct(
        public array $fields,
        public ?string $version,
    ) {}
}
```

---

# 213. Schema ≠ Database Schema

```text
Export Schema
≠
Database Schema Model
```

---

# 214. Export schema purpose

Define:

```text
field names
logical types
nullability
external representation
order
version
```

---

# 215. Schema versioning

Útil para integraciones externas:

```text
customers-export/v1
customers-export/v2
```

---

# 216. Contract export

Una aplicación podrá definir:

```php
final class CustomerExportDefinition
{
    // stable export contract
}
```

---

# 217. Export definition

```php
interface ExportDefinition
{
    public function source(): ExportSource;

    public function schema(): ExportSchema;

    public function configure(
        ExportBuilder $builder,
    ): void;
}
```

---

# 218. Named exports

Ejemplo:

```php
DB::exports()
    ->run('customers.active.v1', $destination);
```

---

# 219. Named export security

Evita permitir al cliente construir consultas arbitrarias para datos sensibles.

---

# 220. Ad-hoc export

Seguirá disponible para código autorizado.

---

# 221. Metadata

Export metadata podrá incluir:

```text
ExportId
startedAt
completedAt
schemaVersion
format
recordCount
byteCount
partCount
checksums
consistency
source fingerprint
```

---

# 222. Metadata privacy

No incluir automáticamente:

```text
SQL
credentials
raw bindings
PII samples
```

---

# 223. Export Manifest

Ejemplo conceptual:

```json
{
  "export": "customers.active",
  "version": "1",
  "format": "ndjson",
  "records": 2500000,
  "parts": [
    {
      "name": "part-00001.ndjson",
      "records": 500000,
      "checksum": "..."
    }
  ]
}
```

---

# 224. Manifest finalization

El manifest final deberá marcar si:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 225. Manifest as evidence

Puede servir como evidencia operacional, pero no sustituye las garantías de la base o destination provider.

---

# 226. Telemetry architecture

Eventos conceptuales:

```text
ExportPlanned
ExportStarted
ExportSourceOpened
ExportBatchFetched
ExportBatchTransformed
ExportPartStarted
ExportBytesWritten
ExportPartCompleted
ExportCheckpointSaved
ExportPaused
ExportResumed
ExportFinalizing
ExportCompleted
ExportPartial
ExportCancelled
ExportFailed
ExportOutcomeUnknown
ExportSourceClosed
ExportDestinationClosed
```

---

# 227. Per-record events

No deberán emitirse por default para cada registro.

---

# 228. Razón

Un export de:

```text
100,000,000 records
```

no debe producir:

```text
100,000,000 telemetry spans
```

---

# 229. Sampling

Eventos por record podrán habilitarse mediante:

```text
sampling
debug mode
diagnostic policy
```

---

# 230. Metrics

```text
db.export.operations
db.export.duration
db.export.records
db.export.bytes
db.export.parts
db.export.failures
db.export.cancellations
db.export.resumes
db.export.checkpoints
db.export.unknown
```

---

# 231. Cardinality

No usar como labels:

```text
raw SQL
full path
record ID
email
tenant ID
cursor token
ExportId
```

si producen cardinalidad no controlada.

---

# 232. Slow destination telemetry

Podrán medirse:

```text
source read time
transform time
serialization time
destination write time
backpressure wait time
```

---

# 233. Bottleneck attribution

Esto permitirá distinguir:

```text
database bottleneck
CPU serialization bottleneck
network bottleneck
destination bottleneck
```

---

# 234. Diagnostics

API conceptual:

```php
DB::export()->explain($request);
```

---

# 235. Explain example

```text
DATABASE EXPORT PLAN

Source:
    EntityQuery<User>

Projection:
    id
    name
    email
    created_at

Hidden Execution Fields:
    id

Traversal:
    KEYSET

Ordering:
    created_at ASC
    id ASC

Batch Size:
    5000

Hydration:
    TYPED_ROW

Consistency:
    BEST_EFFORT

Read Routing:
    PIN_ENDPOINT

Destination:
    FILE

Format:
    CSV

Encoding:
    UTF-8

Compression:
    GZIP

Resume:
    ENABLED

Checkpoint:
    AFTER DESTINATION CONFIRMATION

Tenant:
    tenant-BOUND

Sharding:
    SINGLE SHARD

Sensitive Fields:
    email → MASKED

Result Cache:
    BYPASS

Estimated Memory:
    BOUNDED

ORM IdentityMap:
    NOT USED

Warnings:
    BEST_EFFORT does not guarantee snapshot consistency.
```

---

# 236. Planner

```php
final class ExportPlanner
{
    public function plan(
        ExportRequest $request,
        ExportPlanningContext $context,
    ): ExportPlan;
}
```

---

# 237. Planner responsibilities

Validará:

```text
source
projection
ordering
traversal eligibility
consistency
routing
tenant
shard
serializer
destination
resume compatibility
compression
resource budgets
security
```

---

# 238. Planner must not execute

`ExportPlanner` no deberá:

```text
run query
open DB cursor
write destination
create remote object
```

---

# 239. ExportPlan

```php
final readonly class ExportPlan
{
    public function __construct(
        public ExportSourcePlan $source,
        public ExportProjectionPlan $projection,
        public ExportTraversalPlan $traversal,
        public ExportConsistencyPlan $consistency,
        public ExportTransformationPlan $transformation,
        public ExportSerializationPlan $serialization,
        public ExportDestinationPlan $destination,
        public ExportSecurityPlan $security,
        public ExportRecoveryPlan $recovery,
        public ExportResourcePlan $resources,
    ) {}
}
```

---

# 240. Runner

`ExportRunner` ejecutará el plan.

---

# 241. Runner lifecycle

```text
validate runtime context
↓
open destination
↓
open source traversal
↓
iterate
↓
transform
↓
serialize
↓
write
↓
checkpoint/progress
↓
finalize destination
↓
close resources
↓
produce result
```

---

# 242. Cleanup

Todo recurso adquirido deberá tener ownership.

---

# 243. Source cleanup

```text
cursor
statement
connection lease
temporary state
```

---

# 244. Destination cleanup

```text
file handle
multipart upload
temporary object
compression stream
network stream
```

---

# 245. try/finally semantics

La implementación deberá garantizar conceptualmente:

```php
try {
    // export
} finally {
    // release owned resources
}
```

---

# 246. Destructor

Destructor podrá ser fallback.

Nunca la garantía principal de correctness.

---

# 247. Cleanup failure

No ocultará el error primario.

---

# 248. Multiple failures

Podrá existir:

```text
PrimaryFailure
+
CleanupFailures[]
```

en diagnostics.

---

# 249. Persistent runtime architecture

Especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 250. Scope-local state

Nunca compartir entre operaciones:

```text
current ExportId
current source iterator
current batch
current destination
current file handle
current cursor
current tenant
current shard
current transaction
current checkpoint
current progress
```

---

# 251. Immutable shared components

Podrán compartirse:

```text
serializer registry
format metadata
immutable planners
capability metadata
compiled mapping metadata
```

cuando no contengan estado request-specific.

---

# 252. Static current export

Prohibido:

```php
ExportManager::$currentExport
```

---

# 253. Coroutine safety

En OpenSwoole:

```text
Coroutine A export
Coroutine B export
```

deberán tener contextos completamente aislados.

---

# 254. Worker reset

Al finalizar:

```text
source closed
destination closed
buffers released
connection returned
transaction scope resolved
tenant context released
checkpoint references released
```

---

# 255. HTTP independence

Database Export no deberá asumir:

```text
HTTP Response
Content-Disposition
browser download
```

---

# 256. HTTP integration

Una capa superior podrá hacer:

```text
ExportDestination
→ HTTP response stream
```

---

# 257. Queue independence

Database Export tampoco dependerá obligatoriamente del Queue System.

---

# 258. Background export

Queue/Job podrá ejecutar:

```text
ExportRequest
```

o preferentemente una:

```text
serializable ExportDefinition + parameters
```

---

# 259. Active runtime objects

No serializar:

```text
PDO
Connection
ResultCursor
Generator
open stream
active ExportRunner
```

para enviarlos a un job.

---

# 260. Serializable specification

Para handoff:

```text
ExportDefinitionId
parameters
destination specification
security context reference
```

---

# 261. Destination credentials

No deberán incrustarse sin protección en job payloads/checkpoints.

---

# 262. Error hierarchy

```text
DatabaseException
└── ExportException
    ├── ExportPlanningException
    ├── ExportSourceException
    ├── ExportTraversalException
    ├── ExportProjectionException
    ├── ExportTransformationException
    ├── ExportSerializationException
    ├── ExportEncodingException
    ├── ExportDestinationException
    ├── ExportFinalizationException
    ├── ExportCheckpointException
    │   ├── ExportCheckpointMismatchException
    │   └── ExportCheckpointPersistenceException
    ├── ExportResumeException
    ├── ExportConsistencyException
    ├── ExportAuthorizationException
    ├── ExportResourceException
    ├── ExportCancellationException
    └── ExportOutcomeUnknownException
```

---

# 263. Error phase

Cada error deberá poder indicar:

```text
PLANNING
SOURCE_OPEN
QUERY
HYDRATION
TRANSFORMATION
SERIALIZATION
ENCODING
DESTINATION_WRITE
CHECKPOINT
FINALIZATION
CLEANUP
```

---

# 264. Error evidence

Cuando sea posible:

```text
batch index
record sequence
part number
source continuation
bytes written
destination state
```

---

# 265. No raw record by default

Los errores no incluirán automáticamente el record completo.

---

# 266. Testing matrix

| Área | Caso |
|---|---|
| Source | table |
| Source | Query Builder |
| Source | Model API |
| Source | Repository |
| Projection | simple |
| Projection | alias |
| Projection | hidden traversal fields |
| Traversal | chunk |
| Traversal | keyset |
| Traversal | lazy |
| Traversal | stream |
| Ordering | unique |
| Ordering | inferred tie-breaker |
| Ordering | composite |
| Consistency | best effort |
| Consistency | snapshot |
| Horizon | live |
| Horizon | upper bound |
| CSV | header |
| CSV | escaping |
| CSV | formula injection |
| JSON | array |
| JSON | streaming array |
| NDJSON | large dataset |
| Type | decimal |
| Type | datetime |
| Type | enum |
| Type | null |
| Transform | deterministic |
| Transform | failure |
| Security | row scope |
| Security | field scope |
| Security | masking |
| Security | unauthorized |
| Destination | file |
| Destination | stream |
| Destination | callback |
| Destination | overwrite |
| Destination | partial |
| Compression | gzip |
| Multipart | rotation |
| Manifest | complete |
| Manifest | partial |
| Checkpoint | save |
| Checkpoint | resume |
| Checkpoint | changed query |
| Checkpoint | changed permissions |
| Cancellation | graceful |
| Failure | DB |
| Failure | serializer |
| Failure | destination |
| Failure | checkpoint |
| Failure | finalization |
| Outcome | unknown |
| Replica | pinned |
| Replica | lag |
| Shard | single |
| Shard | per-shard |
| Shard | global merge |
| Tenant | isolated |
| Tenant | cross-tenant rejected |
| Memory | bounded |
| ORM | IdentityMap growth |
| Backpressure | slow destination |
| Runtime | worker reuse |
| Runtime | coroutine isolation |

---

# 267. Directory structure

```text
src/Quantum/Database/Export/
│
├── ExportRequest.php
├── ExportOptions.php
├── ExportId.php
├── ExportPlanner.php
├── ExportPlan.php
├── ExportRunner.php
├── ExportResult.php
├── ExportStatus.php
│
├── Source/
│   ├── ExportSource.php
│   ├── ExportSourceContext.php
│   ├── QueryExportSource.php
│   ├── TableExportSource.php
│   ├── EntityQueryExportSource.php
│   └── RepositoryExportSource.php
│
├── Projection/
│   ├── ExportProjection.php
│   ├── ExportProjectionPlan.php
│   ├── ExportField.php
│   ├── ExportFieldMap.php
│   └── ExportSchema.php
│
├── Traversal/
│   ├── ExportTraversalStrategy.php
│   ├── ExportTraversalPlan.php
│   ├── ExportTraversal.php
│   ├── ChunkExportTraversal.php
│   ├── KeysetExportTraversal.php
│   ├── LazyExportTraversal.php
│   └── StreamingExportTraversal.php
│
├── Consistency/
│   ├── ExportConsistency.php
│   ├── ExportConsistencyPlan.php
│   └── ExportHorizonPolicy.php
│
├── Transformation/
│   ├── ExportTransformer.php
│   ├── ExportTransformationPipeline.php
│   ├── ExportTransformationContext.php
│   └── ExportClock.php
│
├── Serialization/
│   ├── ExportSerializer.php
│   ├── ExportSerializationContext.php
│   ├── ExportSerializedRecord.php
│   ├── CsvExportSerializer.php
│   ├── JsonExportSerializer.php
│   ├── NdjsonExportSerializer.php
│   └── ExportValueFormatter.php
│
├── Encoding/
│   ├── ExportEncoder.php
│   ├── ExportEncoding.php
│   └── ExportEncodingContext.php
│
├── Destination/
│   ├── ExportDestination.php
│   ├── ExportDestinationWriter.php
│   ├── ExportDestinationContext.php
│   ├── ExportDestinationCapability.php
│   ├── ExportDestinationState.php
│   ├── FileExportDestination.php
│   ├── StreamExportDestination.php
│   └── CallbackExportDestination.php
│
├── Compression/
│   ├── ExportCompressor.php
│   ├── ExportCompression.php
│   └── ExportCompressionContext.php
│
├── Multipart/
│   ├── ExportPart.php
│   ├── ExportPartPolicy.php
│   ├── ExportPartWriter.php
│   └── ExportManifest.php
│
├── Security/
│   ├── ExportSecurityPlan.php
│   ├── ExportFieldPolicy.php
│   ├── ExportRedactor.php
│   ├── ExportMasker.php
│   └── CsvFormulaPolicy.php
│
├── Recovery/
│   ├── ExportCheckpoint.php
│   ├── ExportCheckpointStore.php
│   ├── ExportResumeSafety.php
│   ├── ExportRecoveryPlan.php
│   ├── ExportQueryFingerprint.php
│   └── ExportDestinationFingerprint.php
│
├── Progress/
│   ├── ExportProgress.php
│   ├── ExportProgressReporter.php
│   └── ExportProgressEstimation.php
│
├── Resource/
│   ├── ExportResourceBudget.php
│   └── ExportResourcePlan.php
│
├── Diagnostics/
│   ├── ExportInspector.php
│   └── ExportExplainer.php
│
├── Telemetry/
│   └── ExportTelemetry.php
│
└── Exception/
    └── ...
```

---

# 268. Architectural invariants

## DB-EXPORT-001
Export será distinto de Query.

## DB-EXPORT-002
Export será distinto de Pagination.

## DB-EXPORT-003
Export será distinto de Chunk Processing.

## DB-EXPORT-004
Export será distinto de Lazy Collection.

## DB-EXPORT-005
Export será distinto de Streaming Result.

## DB-EXPORT-006
Export será distinto de Backup.

## DB-EXPORT-007
Export será distinto de Database Dump.

## DB-EXPORT-008
Export será distinto de Serialization.

## DB-EXPORT-009
Export utilizará Query Engine para definir datasets.

## DB-EXPORT-010
Export no generará SQL directamente.

## DB-EXPORT-011
Export no ejecutará Driver directamente.

## DB-EXPORT-012
Export no duplicará Query Executor.

## DB-EXPORT-013
Export no duplicará Chunk Engine.

## DB-EXPORT-014
Export no duplicará Cursor Pagination.

## DB-EXPORT-015
Export no duplicará Lazy Collection.

## DB-EXPORT-016
Export no duplicará Streaming Result.

## DB-EXPORT-017
Source Query será distinta de active execution.

## DB-EXPORT-018
Crear un Export no ejecutará la consulta por default.

## DB-EXPORT-019
Query definition será estable durante la ejecución.

## DB-EXPORT-020
Logical Projection será distinta de Physical Projection.

## DB-EXPORT-021
Hidden traversal fields no serán exportados.

## DB-EXPORT-022
Field mapping será explícito.

## DB-EXPORT-023
Traversal strategy será explícita o planificada.

## DB-EXPORT-024
AUTO será explainable.

## DB-EXPORT-025
AUTO no cambiará semántica silenciosamente.

## DB-EXPORT-026
Chunk traversal reutilizará Chunk System.

## DB-EXPORT-027
Keyset reutilizará ordering/continuation infrastructure.

## DB-EXPORT-028
Lazy traversal reutilizará Lazy Collection.

## DB-EXPORT-029
Streaming reutilizará Result Cursor/Streaming Result.

## DB-EXPORT-030
Streaming no será default universal.

## DB-EXPORT-031
Streaming resource ownership será explícito.

## DB-EXPORT-032
Resume requerirá estrategia compatible.

## DB-EXPORT-033
Stable ordering será requerido cuando continuation lo necesite.

## DB-EXPORT-034
Tie-breaker podrá inferirse solo con evidencia.

## DB-EXPORT-035
No se asumirá identificador llamado `id`.

## DB-EXPORT-036
Composite identifiers serán soportados.

## DB-EXPORT-037
Consistency policy será explícita.

## DB-EXPORT-038
BEST_EFFORT no implicará snapshot.

## DB-EXPORT-039
PIN_ENDPOINT no implicará snapshot.

## DB-EXPORT-040
CAPTURE_UPPER_BOUND no implicará snapshot.

## DB-EXPORT-041
Snapshot no se abrirá automáticamente para todo export.

## DB-EXPORT-042
Same Export Definition no implicará Same Results.

## DB-EXPORT-043
Hydration mode será explícito.

## DB-EXPORT-044
Typed rows/projections serán preferidos para grandes exports.

## DB-EXPORT-045
Entity export no serializará graphs arbitrarios.

## DB-EXPORT-046
Lazy relationships no se cargarán accidentalmente.

## DB-EXPORT-047
N+1 detection permanecerá activo.

## DB-EXPORT-048
Serializer será distinto de Destination.

## DB-EXPORT-049
Serializer será distinto de Database Type Conversion.

## DB-EXPORT-050
CSV será soportado.

## DB-EXPORT-051
JSON será soportado.

## DB-EXPORT-052
NDJSON será soportado.

## DB-EXPORT-053
JSON array podrá escribirse incrementalmente.

## DB-EXPORT-054
CSV formula injection tendrá policy.

## DB-EXPORT-055
NULL será distinto de empty string.

## DB-EXPORT-056
Decimal precision deberá preservarse.

## DB-EXPORT-057
Temporal semantics deberán preservarse.

## DB-EXPORT-058
Enum serialization será configurable.

## DB-EXPORT-059
Value Objects requerirán formatter conocido.

## DB-EXPORT-060
Transform será distinto de Projection.

## DB-EXPORT-061
Callbacks arbitrarios no serán convertidos a SQL heurísticamente.

## DB-EXPORT-062
Transform determinism será relevante para resume.

## DB-EXPORT-063
Export Clock podrá ser estable.

## DB-EXPORT-064
Authorization ocurrirá antes de traversal.

## DB-EXPORT-065
Unauthorized rows no se filtrarán después del export.

## DB-EXPORT-066
Field authorization será soportada.

## DB-EXPORT-067
Redaction será distinta de Authorization.

## DB-EXPORT-068
Encryption será distinta de Authorization.

## DB-EXPORT-069
Sensitive export podrá auditarse.

## DB-EXPORT-070
Raw dataset no se almacenará en audit logs.

## DB-EXPORT-071
Destination será una abstracción.

## DB-EXPORT-072
Core no dependerá de cloud SDK concreto.

## DB-EXPORT-073
Destination capabilities serán explícitas.

## DB-EXPORT-074
Appendable será distinto de Resumable.

## DB-EXPORT-075
Destination state será explícito.

## DB-EXPORT-076
Partial output será representable.

## DB-EXPORT-077
Unknown destination state permanecerá UNKNOWN.

## DB-EXPORT-078
Existing destination policy será explícita.

## DB-EXPORT-079
Default de overwrite será seguro.

## DB-EXPORT-080
Append será validado contra format.

## DB-EXPORT-081
Compression será distinta de format.

## DB-EXPORT-082
Resume será validado contra compression.

## DB-EXPORT-083
Multipart será soportable.

## DB-EXPORT-084
Manifest será distinto de checkpoint.

## DB-EXPORT-085
Checksum será distinto de signature.

## DB-EXPORT-086
Checkpoint será versionado.

## DB-EXPORT-087
Checkpoint incluirá query fingerprint.

## DB-EXPORT-088
Checkpoint incluirá ordering fingerprint cuando aplique.

## DB-EXPORT-089
Checkpoint incluirá destination identity.

## DB-EXPORT-090
Checkpoint incluirá security scope relevante.

## DB-EXPORT-091
Checkpoint será distinto de snapshot.

## DB-EXPORT-092
Query fingerprint será semántico.

## DB-EXPORT-093
Resume revalidará autorización.

## DB-EXPORT-094
Changed security scope podrá invalidar resume.

## DB-EXPORT-095
Changed query podrá invalidar resume.

## DB-EXPORT-096
Changed ordering podrá invalidar resume.

## DB-EXPORT-097
Changed destination podrá invalidar resume.

## DB-EXPORT-098
Changed serializer contract podrá invalidar resume.

## DB-EXPORT-099
Checkpoint avanzará tras destination confirmation requerida.

## DB-EXPORT-100
Record fetched no significará record durably exported.

## DB-EXPORT-101
Checkpoint + Resume no implicará exactly-once.

## DB-EXPORT-102
ExportId será distinto de AttemptId.

## DB-EXPORT-103
Progress total podrá ser UNKNOWN.

## DB-EXPORT-104
Export no ejecutará COUNT automáticamente por default.

## DB-EXPORT-105
Exact Count será distinto de Same Snapshot.

## DB-EXPORT-106
Estimated progress será identificado como estimado.

## DB-EXPORT-107
Cancellation será soportada.

## DB-EXPORT-108
Cancellation será distinta de rollback.

## DB-EXPORT-109
Destination abort será capability-driven.

## DB-EXPORT-110
Abort failure será reportable.

## DB-EXPORT-111
PAUSED requerirá checkpoint seguro.

## DB-EXPORT-112
PARTIAL será distinto de FAILED.

## DB-EXPORT-113
UNKNOWN será distinto de FAILED.

## DB-EXPORT-114
Resource budgets serán explícitos.

## DB-EXPORT-115
Memory deberá ser bounded cuando la pipeline lo permita.

## DB-EXPORT-116
Bounded batch no garantizará constant memory.

## DB-EXPORT-117
ORM IdentityMap growth será considerado.

## DB-EXPORT-118
Export no hará `EntityManager::clear()` silenciosamente.

## DB-EXPORT-119
Dedicated EntityManager será soportable.

## DB-EXPORT-120
Large export bypassará Result Cache por default.

## DB-EXPORT-121
Metadata caches podrán seguir usándose.

## DB-EXPORT-122
Read-only no implicará replica-safe.

## DB-EXPORT-123
Replica policy será explícita.

## DB-EXPORT-124
Endpoint pinning será soportable.

## DB-EXPORT-125
Failover respetará consistency contract.

## DB-EXPORT-126
Single-shard export será soportado.

## DB-EXPORT-127
Per-shard export será soportable.

## DB-EXPORT-128
Global ordered export requerirá global ordering.

## DB-EXPORT-129
Global ordered export podrá requerir k-way merge.

## DB-EXPORT-130
Local ID no se asumirá globalmente único.

## DB-EXPORT-131
Shard failure no producirá fake complete result.

## DB-EXPORT-132
Cross-shard snapshot no será asumido.

## DB-EXPORT-133
Tenant isolation será aplicada antes de export.

## DB-EXPORT-134
Cross-tenant export será administrativo y explícito.

## DB-EXPORT-135
Tenant context no cambiará mid-export.

## DB-EXPORT-136
Checkpoint estará tenant-bound cuando aplique.

## DB-EXPORT-137
Destination authorization será soportable.

## DB-EXPORT-138
Arbitrary HTTP destination no será confiable por default.

## DB-EXPORT-139
Filesystem destination protegerá contra path traversal.

## DB-EXPORT-140
Backpressure será bounded.

## DB-EXPORT-141
Slow destination no producirá unbounded buffering.

## DB-EXPORT-142
Concurrency será bounded.

## DB-EXPORT-143
Parallel transformation preservará ordering cuando sea requerido.

## DB-EXPORT-144
Out-of-order export será explícito.

## DB-EXPORT-145
More workers no implicará better performance.

## DB-EXPORT-146
Export Schema será distinto de Database Schema.

## DB-EXPORT-147
Export schema podrá versionarse.

## DB-EXPORT-148
Named exports serán soportables.

## DB-EXPORT-149
Named export no bypassará authorization.

## DB-EXPORT-150
Manifest no incluirá secretos por default.

## DB-EXPORT-151
Telemetry tendrá bounded cardinality.

## DB-EXPORT-152
Per-record telemetry estará deshabilitada por default.

## DB-EXPORT-153
Export podrá medir backpressure.

## DB-EXPORT-154
Export será explainable.

## DB-EXPORT-155
Planner será distinto de Runner.

## DB-EXPORT-156
Planner no abrirá destination.

## DB-EXPORT-157
Planner no ejecutará query.

## DB-EXPORT-158
Runner seguirá el plan.

## DB-EXPORT-159
Resource ownership será explícito.

## DB-EXPORT-160
Cleanup ocurrirá en success/failure/cancel.

## DB-EXPORT-161
Destructor no será correctness mechanism principal.

## DB-EXPORT-162
Cleanup failure no ocultará primary failure.

## DB-EXPORT-163
Mutable runtime state será scope-local.

## DB-EXPORT-164
Current export no será static global state.

## DB-EXPORT-165
FrankenPHP worker reuse será seguro.

## DB-EXPORT-166
RoadRunner worker reuse será seguro.

## DB-EXPORT-167
OpenSwoole coroutine isolation será segura.

## DB-EXPORT-168
HTTP no será dependencia del Database Export System.

## DB-EXPORT-169
Queue no será dependencia del Database Export System.

## DB-EXPORT-170
Active DB resources no serán serializables como job payload.

## DB-EXPORT-171
Credentials no se almacenarán inseguramente en checkpoints.

## DB-EXPORT-172
Error phase será identificable.

## DB-EXPORT-173
Raw record no aparecerá en errors por default.

## DB-EXPORT-174
Outcome evidence será preservada.

## DB-EXPORT-175
UNKNOWN nunca se convertirá artificialmente en COMPLETED.

## DB-EXPORT-176
Correctness tendrá prioridad sobre throughput.

## DB-EXPORT-177
Authorization tendrá prioridad sobre convenience.

## DB-EXPORT-178
Resource safety tendrá prioridad sobre eager materialization.

## DB-EXPORT-179
Database seguirá siendo autoridad del dataset observado.

## DB-EXPORT-180
Destination seguirá siendo autoridad sobre durability externa que pueda demostrar.

---

# 269. Modelo formal

Sea una consulta lógica:

```text
Q
```

que produce un conjunto ordenado:

```text
R(Q, O)
=
{r1, r2, ..., rn}
```

donde:

```text
O
=
effective deterministic ordering
```

El pipeline de export puede representarse:

```text
E(ri)
=
Encode(
  Serialize(
    Secure(
      Transform(
        Project(ri)
      )
    )
  )
)
```

El destino recibe:

```text
D
=
E(r1) ⊕ E(r2) ⊕ ... ⊕ E(rn)
```

donde `⊕` representa la operación de composición apropiada al formato:

```text
CSV row append
NDJSON line append
JSON array element composition
multipart part composition
custom format composition
```

---

# 270. Modelo incremental

En lugar de:

```text
R = Execute(Q)
Materialize(R)
Serialize(R)
Write(R)
```

VoltStack deberá preferir:

```text
while traversal.hasNext():
    batch = traversal.next()

    for record in batch:
        output = pipeline(record)
        destination.write(output)

    release(batch)
```

---

# 271. Modelo de memoria

Idealmente:

```text
Memory
=
O(B + S + W)
```

donde:

```text
B = traversal batch buffer
S = serialization/transformation buffer
W = destination write buffer
```

y no:

```text
O(total dataset)
```

---

# 272. Modelo de continuación

Sea:

```text
K(r)
```

la clave ordenada de un registro.

Después de exportar durablemente:

```text
ri
```

la frontera:

```text
Ci = K(ri)
```

puede utilizarse para construir:

```text
Q(i+1)
=
Q
∧
After(Ci)
```

cuando la estrategia sea keyset-compatible.

---

# 273. Condición crítica

La frontera deberá actualizarse solo cuando:

```text
record output
```

haya alcanzado el durability boundary exigido por la destination policy.

---

# 274. Modelo de consistencia

Para `BEST_EFFORT`:

```text
R1 may observe T1
R2 may observe T2
...
Rn may observe Tn
```

Para `SNAPSHOT`, cuando sea realmente soportado:

```text
R1 ... Rn
observe logical snapshot S
```

No deberá declararse:

```text
SNAPSHOT
```

sin evidencia de que la infraestructura lo garantiza.

---

# 275. Arquitectura final

```text
                         Export API
                             │
                             ▼
                       ExportRequest
                             │
                             ▼
                       ExportPlanner
                             │
                             ▼
                         ExportPlan
                             │
           ┌─────────────────┼──────────────────┐
           │                 │                  │
           ▼                 ▼                  ▼
       Security          Traversal          Destination
           │                 │                  │
           └─────────────────┼──────────────────┘
                             ▼
                        ExportRunner
                             │
                             ▼
                       Query Engine
                             │
                             ▼
                      Execution Engine
                             │
                             ▼
                   Incremental Results
                             │
                             ▼
                         Projection
                             │
                             ▼
                      Transformation
                             │
                             ▼
                    Masking / Redaction
                             │
                             ▼
                       Serialization
                             │
                             ▼
                          Encoding
                             │
                             ▼
                        Compression
                             │
                             ▼
                    Destination Writer
                             │
                ┌────────────┼────────────┐
                ▼            ▼            ▼
             Progress     Checkpoint    Manifest
                │            │            │
                └────────────┼────────────┘
                             ▼
                        ExportResult
```

---

# 276. Regla maestra final

> **Database Export en VoltStack será una arquitectura de extracción incremental y gobernada, no una operación de materialización masiva: el sistema deberá mantener separadas la definición del dataset, su recorrido, su representación externa y su destino, preservando seguridad, consistencia, ownership de recursos y evidencia operacional durante todo el proceso.**

Siempre:

```text
Export
≠
Query
```

```text
Export
≠
Backup
```

```text
Export
≠
Serialization
```

```text
Logical Projection
≠
Physical Execution Projection
```

```text
Streaming
≠
Chunking
```

```text
Pinned Replica
≠
Snapshot
```

```text
Upper Bound
≠
Snapshot
```

```text
Record Read
≠
Record Durably Exported
```

```text
Checkpoint
≠
Exactly Once
```

```text
Manifest
≠
Checkpoint
```

```text
Appendable Destination
≠
Resumable Destination
```

```text
Redaction
≠
Authorization
```

```text
Encryption
≠
Authorization
```

```text
Read Only
≠
Replica Safe
```

```text
Batch Size
≠
Constant Memory
```

```text
Same Export Definition
≠
Same Export Result
```

y:

```text
UNKNOWN
≠
COMPLETED
```

---

# 277. Ejemplo de API completa

```php
$result = User::query()
    ->where('status', 'active')
    ->orderBy('created_at')
    ->export()
    ->fields([
        'id',
        'name',
        'email',
        'created_at',
    ])
    ->rename([
        'id'         => 'User ID',
        'name'       => 'Name',
        'email'      => 'Email',
        'created_at' => 'Created At',
    ])
    ->mask([
        'email' => ExportMask::EMAIL,
    ])
    ->format(ExportFormat::CSV)
    ->encoding('UTF-8')
    ->compression(ExportCompression::GZIP)
    ->traversal(ExportTraversalStrategy::KEYSET)
    ->batchSize(5000)
    ->consistency(ExportConsistency::BEST_EFFORT)
    ->checkpointEveryBatch()
    ->toFile('/exports/users.csv.gz')
    ->run();
```

Internamente:

```text
Model Query
↓
Query Model
↓
Security Scope
↓
Export Planner
↓
Keyset Traversal
↓
Query Engine
↓
Execution Engine
↓
Typed Rows
↓
Projection
↓
Masking
↓
CSV Serializer
↓
UTF-8 Encoder
↓
GZIP Compressor
↓
File Destination
↓
Checkpoint
↓
Export Result
```

---

# 278. API para datasets extremadamente grandes

```php
$result = DB::export()
    ->from(
        DB::table('events')
            ->where('created_at', '>=', $from)
            ->orderBy('created_at')
            ->orderBy('event_id')
    )
    ->toNdjson($destination)
    ->traversal(ExportTraversalStrategy::KEYSET)
    ->batchSize(10_000)
    ->rotateEveryRecords(1_000_000)
    ->manifest()
    ->resume($checkpoint)
    ->run();
```

Esto permitirá trabajar con:

```text
millions
tens of millions
hundreds of millions
```

de registros sin convertir:

```text
RAM
```

en el límite arquitectónico principal.

---

# 279. Integración con los sistemas anteriores

```text
                    Database Export
                           │
         ┌─────────────────┼───────────────────┐
         │                 │                   │
         ▼                 ▼                   ▼
    Query Engine      Chunk Processing    Lazy Collection
         │                 │                   │
         ├─────────────────┼───────────────────┤
         │                 │                   │
         ▼                 ▼                   ▼
 Cursor Pagination   Streaming Result      Type System
         │                 │                   │
         ├─────────────────┼───────────────────┤
         │                 │                   │
         ▼                 ▼                   ▼
 Read Routing          Sharding            Security
         │                 │                   │
         └─────────────────┼───────────────────┘
                           ▼
                        Export
```

Export será, por tanto, un **orquestador especializado de lectura y entrega**, no un nuevo motor paralelo.

---

# 280. Bloque 19 hasta ahora

```text
✓ 199_DATABASE_PAGINATION_SYSTEM.md
✓ 200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
✓ 201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
✓ 202_DATABASE_LAZY_COLLECTION_SYSTEM.md
✓ 203_DATABASE_BULK_INSERT_SYSTEM.md
✓ 204_DATABASE_BULK_UPDATE_SYSTEM.md
✓ 205_DATABASE_BULK_DELETE_SYSTEM.md
✓ 206_DATABASE_IMPORT_SYSTEM.md
✓ 207_DATABASE_EXPORT_SYSTEM.md

→ 208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md
```

Con Import y Export, la arquitectura ya cubre:

```text
External Data
      ↓
    Import
      ↓
   Database
      ↓
    Export
      ↓
External Data
```

mientras los sistemas anteriores proporcionan:

```text
Pagination
Cursor Traversal
Chunking
Lazy Consumption
Bulk Insert
Bulk Update
Bulk Delete
```

---

# 281. Siguiente documento

```text
208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md
```

Este documento cerrará el **Bloque 19 — Pagination, Batch & Large Data** definiendo la arquitectura superior que coordinará el procesamiento de datasets extremadamente grandes mediante:

```text
Large Dataset Processing
├── bounded-memory execution
├── dataset traversal
├── partitions
├── key ranges
├── shard-aware processing
├── parallel workers
├── work distribution
├── checkpoints
├── resumability
├── progress
├── backpressure
├── adaptive batching
├── resource budgets
├── cancellation
├── deadlines
├── failure isolation
├── retry boundaries
├── idempotency
├── exactly-once limitations
├── transaction boundaries
├── consistency models
├── ORM memory governance
├── bulk operations
├── import/export coordination
├── persistent-runtime safety
├── telemetry
└── diagnostics
```

bajo una regla fundamental:

> **Procesar un dataset grande en VoltStack no significará ejecutar la misma operación en un loop gigante; será una arquitectura de ejecución incremental, particionada, recuperable y gobernada por recursos que coordinará Chunk, Lazy, Bulk, Import y Export sin duplicar sus responsabilidades.**