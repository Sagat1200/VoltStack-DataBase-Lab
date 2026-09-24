# 206_DATABASE_IMPORT_SYSTEM.md

# VoltStack Quantum Database
## Database Import System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 206 — Database Import System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `205_DATABASE_BULK_DELETE_SYSTEM.md`  
**Siguiente documento:** `207_DATABASE_EXPORT_SYSTEM.md`

---

# 1. Propósito

`Database Import System` define la arquitectura mediante la cual VoltStack podrá importar grandes volúmenes de datos desde fuentes externas o internas hacia el sistema de base de datos mediante un pipeline explícito, observable, reanudable y gobernado.

Ejemplo conceptual:

```php
$result = DB::import()
    ->fromCsv('/data/customers.csv')
    ->into('customers')
    ->map([
        'email_address' => 'email',
        'full_name' => 'name',
    ])
    ->validate($rules)
    ->batchSize(1000)
    ->run();
```

También deberá soportar fuentes:

```text
CSV
JSON
NDJSON
iterables
streams
files
uploaded files
external adapters
custom sources
```

La regla central será:

> **Importar datos en VoltStack no será equivalente a ejecutar Bulk Insert: el Import System será un pipeline de ingestión responsable de leer, decodificar, mapear, normalizar, validar, transformar, enrutar y entregar datos a los mecanismos de persistencia apropiados.**

Formalmente:

```text
Import
=
Source
+
Decoding
+
Mapping
+
Normalization
+
Validation
+
Transformation
+
Routing
+
Persistence Strategy
+
Progress
+
Failure Handling
+
Recovery
```

y nunca:

```text
Import
=
BulkInsert
```

---

# 2. Posición dentro del Bloque 19

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

Los documentos 203–205 proporcionan las primitivas de escritura masiva.

Import construirá sobre ellas:

```text
External Data
      ↓
Import Pipeline
      ↓
Bulk Insert / Bulk Update / Upsert
      ↓
Database
```

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Import
≠
Bulk Insert
≠
Seeder
≠
Fixture
≠
Migration
≠
ETL completo
≠
File Upload
≠
Database Restore
≠
Replication
```

---

# 4. Import vs Bulk Insert

Bulk Insert recibe:

```text
already normalized row values
```

Import recibe:

```text
raw external data
```

Ejemplo:

```text
"25/12/2026"
"MXN 1,250.50"
" YES "
"null"
```

antes de convertirse en:

```text
LocalDate
Money
bool
null
```

---

# 5. Import vs Seeder

Seeder define datos intencionalmente creados para:

```text
development
testing
initial reference data
```

Import procesa datos provenientes de una fuente.

---

# 6. Import vs Migration

Migration cambia:

```text
database structure / controlled data evolution
```

Import ingiere:

```text
business data
external datasets
operational records
```

---

# 7. Import vs Restore

Restore intenta reconstruir un estado previamente respaldado.

Import interpreta y procesa datos como dataset lógico.

Por tanto:

```text
Restore
≠
Import
```

---

# 8. Arquitectura general

```text
Import API
    │
    ▼
ImportRequest
    │
    ▼
ImportPlanner
    │
    ├── Source
    ├── Decoder
    ├── Mapping
    ├── Normalization
    ├── Validation
    ├── Transformation
    ├── Persistence
    ├── Routing
    ├── Transaction
    ├── Recovery
    └── Resources
    │
    ▼
ImportPlan
    │
    ▼
ImportRunner
    │
    ▼
Source Reader
    │
    ▼
Decoder
    │
    ▼
Record Mapper
    │
    ▼
Normalizer
    │
    ▼
Validator
    │
    ▼
Transformer
    │
    ▼
Persistence Adapter
    │
    ├── Bulk Insert
    ├── Bulk Update
    ├── Upsert
    └── Custom
    │
    ▼
ImportResult
```

---

# 9. Pipeline fundamental

```text
Bytes / Records
      ↓
Decode
      ↓
Raw Record
      ↓
Map
      ↓
Mapped Record
      ↓
Normalize
      ↓
Normalized Record
      ↓
Validate
      ↓
Valid Record
      ↓
Transform
      ↓
Persistence Record
      ↓
Route
      ↓
Batch
      ↓
Persist
```

---

# 10. Source abstraction

Propuesta:

```php
interface ImportSource
{
    public function open(
        ImportSourceContext $context
    ): ImportSourceReader;
}
```

---

# 11. Source Reader

```php
interface ImportSourceReader
{
    public function records(): iterable;

    public function checkpoint(): ?ImportSourceCheckpoint;

    public function close(): void;
}
```

---

# 12. Source ≠ Decoder

Una fuente responde:

> ¿De dónde vienen los datos?

Un Decoder responde:

> ¿Cómo se interpretan los datos?

---

# 13. Fuente CSV

```php
DB::import()
    ->fromCsv('/data/users.csv');
```

---

# 14. Fuente JSON

```php
DB::import()
    ->fromJson('/data/users.json');
```

---

# 15. NDJSON

Para datasets grandes:

```text
{"id":1,"name":"A"}
{"id":2,"name":"B"}
{"id":3,"name":"C"}
```

NDJSON será particularmente útil porque permite:

```text
streaming record-by-record
```

---

# 16. Iterable source

```php
DB::import()
    ->fromIterable($records)
    ->into('users');
```

---

# 17. Stream source

```php
DB::import()
    ->fromStream($stream);
```

---

# 18. External adapters

Podrán existir:

```text
S3ImportSource
HttpImportSource
GoogleCloudStorageImportSource
SftpImportSource
DatabaseImportSource
MessageStreamImportSource
```

como extensiones.

---

# 19. Core independence

El core no deberá depender obligatoriamente de:

```text
S3 SDK
Google Cloud SDK
HTTP client specific
```

---

# 20. Source capabilities

Cada fuente podrá declarar:

```text
STREAMABLE
SEEKABLE
REPLAYABLE
CHECKPOINTABLE
SIZED
COMPRESSED
REMOTE
```

---

# 21. SourceCapability

```php
enum ImportSourceCapability
{
    case STREAMABLE;
    case SEEKABLE;
    case REPLAYABLE;
    case CHECKPOINTABLE;
    case SIZE_KNOWN;
}
```

---

# 22. Replayable source

Una fuente replayable puede reiniciarse desde el inicio.

---

# 23. Seekable source

Una fuente seekable permite moverse a:

```text
byte offset
record offset
logical checkpoint
```

---

# 24. Checkpointable source

Permite reanudar un import sin empezar desde cero.

---

# 25. Source replayability ≠ persistence replay safety

Aunque podamos volver a leer los mismos registros:

```text
source replayable
```

eso no implica:

```text
database persistence replay safe
```

---

# 26. Decoder abstraction

```php
interface ImportDecoder
{
    public function decode(
        ImportInputChunk $input,
        ImportDecodeContext $context,
    ): iterable;
}
```

---

# 27. Decoder output

Producirá:

```text
RawImportRecord
```

---

# 28. RawImportRecord

```php
final readonly class RawImportRecord
{
    public function __construct(
        public ImportRecordPosition $position,
        public array $fields,
        public ?string $rawFragment = null,
    ) {}
}
```

---

# 29. Raw fragment

Debe ser opcional porque conservar el contenido crudo de millones de registros puede:

```text
consume memory
expose PII
increase logging risk
```

---

# 30. Position

Cada registro deberá poder identificarse por:

```text
record number
line number
byte offset
source-specific location
```

para diagnósticos.

---

# 31. CSV Decoder

Deberá manejar:

```text
delimiter
enclosure
escape
headers
encoding
BOM
empty lines
multiline fields
```

---

# 32. CSV Header

Policies posibles:

```text
REQUIRED
OPTIONAL
NONE
CUSTOM
```

---

# 33. Duplicate headers

Ejemplo:

```text
id,name,name,email
```

deberá producir error o estrategia explícita.

No sobrescribir silenciosamente.

---

# 34. JSON modes

Se podrán distinguir:

```text
JSON_ARRAY
NDJSON
OBJECT_STREAM
CUSTOM
```

---

# 35. Huge JSON array

Un archivo:

```json
[
  {...},
  {...},
  ...
]
```

no deberá forzosamente cargarse completo.

El decoder podrá utilizar parsing incremental cuando la implementación lo soporte.

---

# 36. Encoding

Import deberá poder declarar:

```text
UTF-8
ISO-8859-1
Windows-1252
custom
```

pero internamente deberá normalizar a una representación consistente.

---

# 37. Encoding error policies

```text
FAIL
REPLACE
SKIP_RECORD
CUSTOM
```

---

# 38. Mapping

El Import Mapping System convierte nombres/posiciones externos en campos internos.

Ejemplo:

```php
->map([
    'customer_email' => 'email',
    'customer_name'  => 'name',
])
```

---

# 39. Mapping ≠ Type Conversion

```text
Mapping
=
which field maps where

Conversion
=
how value becomes target type
```

---

# 40. ImportMapping

```php
final readonly class ImportMapping
{
    public function __construct(
        public array $fieldMappings,
    ) {}
}
```

---

# 41. Positional mapping

Para CSV sin header:

```php
->mapPositions([
    0 => 'id',
    1 => 'name',
    2 => 'email',
]);
```

---

# 42. Constant values

El mapping podrá incluir:

```text
tenant_id = current tenant
source = "legacy-system"
```

---

# 43. Derived fields

Ejemplo:

```text
full_name
→
first_name
+
last_name
```

pertenece mejor a:

```text
Transformation
```

que Mapping simple.

---

# 44. Missing source fields

Policies:

```text
ERROR
USE_DEFAULT
USE_NULL
IGNORE_TARGET
CUSTOM
```

---

# 45. Missing ≠ null

Mantener:

```text
source field absent
≠
source field explicitly null
```

---

# 46. Normalization

Normalización transforma formas externas a valores canónicos.

Ejemplo:

```text
" YES "
→ true
```

```text
"  Alice  "
→ "Alice"
```

```text
"25/12/2026"
→ LocalDate
```

---

# 47. Type System integration

Debe reutilizar:

```text
155_DATABASE_TYPE_SYSTEM.md
157_DATABASE_VALUE_CONVERSION_SYSTEM.md
159_DATABASE_ENUM_MAPPING_SYSTEM.md
160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md
161_DATABASE_JSON_TYPE_SYSTEM.md
162_DATABASE_DATE_TIME_TYPE_SYSTEM.md
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md
```

---

# 48. External parser vs database converter

Ejemplo:

```text
"25/12/2026"
```

requiere primero entender:

```text
dd/mm/yyyy
```

Esto pertenece al Import Normalizer.

Después:

```text
LocalDate
→ database representation
```

pertenece al Type System.

---

# 49. Regla

```text
External Text Parsing
≠
Database Type Conversion
```

---

# 50. Locale

Import deberá soportar contexto:

```text
locale
decimal separator
thousands separator
date format
timezone
boolean vocabulary
```

---

# 51. Money example

```text
"1.234,56"
```

puede significar:

```text
1234.56
```

según locale.

Nunca inferir locale global silenciosamente.

---

# 52. ImportNormalizationContext

```php
final readonly class ImportNormalizationContext
{
    public function __construct(
        public string $locale,
        public ?DateTimeZone $timezone,
        public ImportNullPolicy $nullPolicy,
    ) {}
}
```

---

# 53. Null policies

Fuentes externas pueden expresar null como:

```text
NULL
null
N/A
-
empty
\N
```

---

# 54. Configuración explícita

```php
->nullValues([
    '',
    'NULL',
    '\N',
])
```

---

# 55. Empty string ≠ null

Default recomendado:

```text
"" remains ""
```

salvo configuración explícita.

---

# 56. Validation

Después de normalizar, cada registro podrá validarse.

---

# 57. Import Validator

```php
interface ImportRecordValidator
{
    public function validate(
        NormalizedImportRecord $record,
        ImportValidationContext $context,
    ): ImportValidationResult;
}
```

---

# 58. Validation System integration

Podrá integrarse con:

```text
VoltStack Validation System
```

pero la capa Database Import no deberá duplicar sus reglas.

---

# 59. Validation layers

Podrán existir:

```text
STRUCTURAL
TYPE
FIELD
RECORD
CROSS_RECORD
DATABASE
```

---

# 60. Structural validation

Ejemplo:

```text
required column missing
invalid number of CSV fields
malformed JSON object
```

---

# 61. Type validation

Ejemplo:

```text
"abc"
→ integer target
```

---

# 62. Record validation

Ejemplo:

```text
start_date <= end_date
```

---

# 63. Cross-record validation

Ejemplo:

```text
duplicate external_id in same import
```

puede requerir estado adicional.

---

# 64. Database validation

Ejemplo:

```text
foreign key must exist
email must be unique
```

---

# 65. DB constraints remain authority

Aunque se pre-valide:

```text
Database Constraint
```

continúa siendo autoridad final por concurrencia.

---

# 66. Cross-record validation memory

No deberá requerir almacenar todo el dataset si existe una estrategia bounded.

---

# 67. Uniqueness within import

Policies posibles:

```text
NONE
PER_BATCH
GLOBAL_EXTERNAL_STORE
GLOBAL_MEMORY_BOUNDED
DATABASE_ENFORCED
```

---

# 68. Validation result

```php
final readonly class ImportValidationResult
{
    public function __construct(
        public bool $valid,
        public array $violations,
    ) {}
}
```

---

# 69. Error policies

Una importación puede elegir:

```php
enum ImportInvalidRecordPolicy
{
    case FAIL_OPERATION;
    case FAIL_BATCH;
    case SKIP_RECORD;
    case COLLECT_AND_CONTINUE;
    case REDIRECT_TO_ERROR_SINK;
}
```

---

# 70. Default recomendado

Para herramientas empresariales:

```text
FAIL_OPERATION
```

o:

```text
REDIRECT_TO_ERROR_SINK
```

según modo.

Nunca ignorar registros inválidos silenciosamente.

---

# 71. Error sink

```php
interface ImportErrorSink
{
    public function report(
        ImportRecordError $error
    ): void;
}
```

---

# 72. Implementaciones

Podrán existir:

```text
MemoryErrorSink
FileErrorSink
DatabaseErrorSink
LoggerErrorSink
CallbackErrorSink
```

---

# 73. Memory sink

Deberá estar bounded.

No almacenar:

```text
10,000,000 errors
```

en memoria sin límites.

---

# 74. Error payload security

Errores no deberán copiar automáticamente:

```text
passwords
tokens
sensitive PII
full raw records
```

---

# 75. Transformation

Después de mapping/normalization/validation se podrán aplicar transformaciones.

---

# 76. Transformer

```php
interface ImportTransformer
{
    public function transform(
        NormalizedImportRecord $record,
        ImportTransformContext $context,
    ): ImportTransformResult;
}
```

---

# 77. Transformaciones

Ejemplos:

```text
split full name
calculate checksum
normalize email
map legacy status
derive tenant
derive shard key
calculate slug
```

---

# 78. Transformation ≠ Validation

Una transformación cambia datos.

Una validación decide si son aceptables.

---

# 79. Transformation order

Default:

```text
Decode
→ Map
→ Normalize
→ Validate
→ Transform
→ Persistence validation
```

pero deberá ser configurable cuando una transformación sea necesaria antes de validación.

---

# 80. Pipeline phases

Una arquitectura flexible podrá soportar:

```text
PRE_NORMALIZATION_TRANSFORM
NORMALIZATION
POST_NORMALIZATION_TRANSFORM
VALIDATION
PRE_PERSISTENCE_TRANSFORM
```

---

# 81. Determinism

Transformaciones reanudables/reintentables deberían ser deterministas.

---

# 82. Non-deterministic transform

Ejemplo:

```text
created_at = now()
```

puede producir valores distintos tras retry.

---

# 83. Deterministic clock

El `ImportContext` deberá poder proveer:

```text
ImportClock
```

estable por operación.

---

# 84. Randomness

Si se requiere:

```text
UUID
random synthetic value
```

debe existir policy de determinismo/replay.

---

# 85. Persistence strategies

Después del pipeline, Import deberá delegar en una estrategia de persistencia.

```php
enum ImportPersistenceStrategy
{
    case INSERT;
    case UPDATE_BY_KEY;
    case UPSERT;
    case INSERT_IGNORE_CONFLICT;
    case CUSTOM;
}
```

---

# 86. INSERT

Delegará a:

```text
203_DATABASE_BULK_INSERT_SYSTEM.md
```

---

# 87. UPDATE_BY_KEY

Delegará a:

```text
204_DATABASE_BULK_UPDATE_SYSTEM.md
```

---

# 88. DELETE

No será una estrategia normal de import.

Un workflow de synchronization podría usar Bulk Delete, pero será un nivel superior.

---

# 89. UPSERT

Deberá tratarse como semántica específica.

No será una combinación improvisada de:

```text
SELECT
then INSERT/UPDATE
```

si la concurrencia puede romperla.

---

# 90. Import Upsert

Podrá requerir:

```text
ConflictTarget
InsertValues
UpdateValues
```

y deberá ser capability-driven.

---

# 91. Upsert fallback

Si no existe soporte nativo, cualquier emulación deberá declarar claramente sus garantías de concurrencia.

---

# 92. No select-then-insert assumption

```text
SELECT missing
↓
concurrent insert
↓
INSERT
```

puede competir.

La DB continúa siendo autoridad.

---

# 93. Persistence adapter

```php
interface ImportPersistenceAdapter
{
    public function persist(
        ImportRecordBatch $batch,
        ImportPersistenceContext $context,
    ): ImportPersistenceResult;
}
```

---

# 94. Bulk integration

El adapter no deberá duplicar:

```text
batch sizing
type conversion
SQL compilation
transactions
routing
```

cuando puedan ser delegados a Bulk subsystems.

---

# 95. Import batch ≠ Bulk batch

Distinción importante:

```text
Import Batch
=
records processed together in pipeline

Bulk Batch
=
records grouped for DB operation
```

Pueden coincidir, pero no es obligatorio.

---

# 96. Ejemplo

```text
Import batch = 5,000 records

Bulk insert effective batch = 800
because parameter limit
```

---

# 97. Bounded pipeline

Import debe permitir:

```text
Source → 5,000
Normalizer → 5,000
Validator → valid 4,950
Bulk → 7 DB batches
```

sin cargar el archivo completo.

---

# 98. Import buffering

Cada fase deberá declarar:

```text
streaming
bounded buffering
full materialization
```

---

# 99. Full materialization

Debe evitarse en core para grandes datasets.

---

# 100. Progress

Import necesita un modelo de progreso explícito.

---

# 101. Progress model

```php
final readonly class ImportProgress
{
    public function __construct(
        public int $recordsRead,
        public int $recordsDecoded,
        public int $recordsValid,
        public int $recordsRejected,
        public int $recordsPersisted,
        public int $batchesCompleted,
        public ?int $totalRecords,
    ) {}
}
```

---

# 102. Total records

Puede ser:

```text
KNOWN
ESTIMATED
UNKNOWN
```

---

# 103. Progress percentage

Solo podrá calcularse con confianza cuando exista una base válida.

Nunca:

```text
unknown total
→ 73%
```

inventado.

---

# 104. Byte progress

Para archivos seekable se podrá usar:

```text
bytesConsumed / totalBytes
```

pero esto mide bytes, no records.

---

# 105. Progress unit

Debe distinguir:

```text
RECORDS
BYTES
BATCHES
UNKNOWN
```

---

# 106. Checkpoints

Import deberá soportar resumability cuando la fuente y persistence semantics lo permitan.

---

# 107. Import checkpoint

Podrá incluir:

```text
version
import id
source fingerprint
source checkpoint
pipeline fingerprint
mapping fingerprint
schema target
persistence strategy
tenant
shard/domain
records read
records persisted
last confirmed batch
```

---

# 108. Checkpoint ≠ transaction log

No registra necesariamente todos los efectos DB.

---

# 109. Checkpoint ≠ exactly once

Siempre:

```text
Checkpoint
≠
Exactly Once
```

---

# 110. Checkpoint timing

Default:

```text
persist batch
↓
required transaction outcome
↓
save import checkpoint
```

---

# 111. Checkpoint before persistence

Sería peligroso:

```text
checkpoint says batch completed
↓
DB write fails
```

provocando pérdida.

---

# 112. Checkpoint failure after persistence

Caso:

```text
DB commit confirmed
↓
checkpoint store failure
```

reanudación podría repetir records.

---

# 113. Replay safety

Persistence strategy deberá declarar:

```text
SAFE
IDEMPOTENT_BY_KEY
CONSTRAINT_PROTECTED
UNSAFE
UNKNOWN
```

---

# 114. Import insert replay

INSERT puro puede duplicar datos si se repite.

---

# 115. Upsert replay

Puede ser más replay-safe, pero:

```text
triggers
timestamps
increments
side effects
```

pueden seguir haciéndolo no idempotente.

---

# 116. Durable import identity

Una importación reanudable tendrá:

```text
ImportId
```

estable.

---

# 117. ImportId ≠ random execution id

Podrán existir ambos:

```text
ImportId
ExecutionAttemptId
```

---

# 118. Source fingerprint

Para detectar archivo cambiado:

```text
size
mtime
checksum
etag
version id
custom identity
```

según fuente.

---

# 119. Source fingerprint mismatch

Al reanudar:

```text
checkpoint source fingerprint
≠
current source fingerprint
```

deberá fallar por default.

---

# 120. Pipeline fingerprint

Cambiar:

```text
mapping
validation
transformation
persistence strategy
```

puede invalidar un checkpoint.

---

# 121. Checkpoint versioning

Formato del checkpoint deberá versionarse.

---

# 122. Import state machine

```text
CREATED
PLANNED
RUNNING
PAUSED
COMPLETED
PARTIAL
FAILED
CANCELLED
UNKNOWN
```

---

# 123. Pause

A diferencia de STOP definitivo, un import checkpointable puede:

```text
PAUSE
```

y reanudarse.

---

# 124. Pause requirements

Requiere:

```text
checkpointable source
durable checkpoint
safe persistence boundary
```

---

# 125. Cancellation

Cancellation podrá detener la operación sin prometer resumability.

---

# 126. Pause ≠ cancel

```text
PAUSE
≠
CANCEL
```

---

# 127. Transaction policies

Propuesta:

```php
enum ImportTransactionPolicy
{
    case NONE;
    case USE_EXISTING;
    case PER_IMPORT_BATCH;
    case PER_PERSISTENCE_BATCH;
    case WHOLE_IMPORT;
    case CUSTOM;
}
```

---

# 128. WHOLE_IMPORT

Para millones de registros suele ser peligroso:

```text
long transaction
large locks
MVCC growth
massive rollback
connection pinning
```

---

# 129. Default recomendado

Para grandes imports:

```text
PER_PERSISTENCE_BATCH
```

o:

```text
PER_IMPORT_BATCH
```

según estrategia.

---

# 130. Partial import

Con batches committed:

```text
Batch 1 ✓
Batch 2 ✓
Batch 3 ✗
```

resultado global puede ser:

```text
PARTIAL
```

---

# 131. Import atomicity

Nunca prometer:

```text
entire import atomic
```

salvo que la infraestructura realmente lo garantice.

---

# 132. External transaction

Si utiliza `USE_EXISTING`, Import no deberá hacer commit de la transacción del caller.

---

# 133. UNKNOWN commit

Si un batch finaliza con:

```text
COMMIT UNKNOWN
```

el import deberá preservar:

```text
UNKNOWN
```

para ese boundary.

---

# 134. No blind resume

No avanzar checkpoint ni reintentar ciegamente después de outcome desconocido.

---

# 135. Sharding

Import deberá determinar routing por registro cuando aplique.

---

# 136. Routing pipeline

```text
Normalized Record
↓
Tenant Resolution
↓
Shard Key Resolution
↓
Partition Routing
↓
Persistence Batch
```

---

# 137. Routing before DB batching

Para datos mezclados:

```text
Record A → Shard 1
Record B → Shard 2
Record C → Shard 1
```

deberá agruparlos según policy.

---

# 138. Cross-shard import

No implica una transacción global.

---

# 139. Result

Podrá indicar:

```text
per-shard progress
per-shard failures
per-shard outcomes
```

---

# 140. Routing key missing

Para una escritura que requiere shard key:

```text
missing key
→ validation/routing failure
```

No broadcast.

---

# 141. Tenant isolation

Todo registro tenant-aware deberá resolverse a un tenant autorizado.

---

# 142. Tenant source strategies

Posibles:

```text
FIXED_CONTEXT
FIELD_MAPPED
CUSTOM_RESOLVER
ADMINISTRATIVE_MULTI_TENANT
```

---

# 143. FIXED_CONTEXT

Todos los registros pertenecen al:

```text
current TenantContext
```

---

# 144. FIELD_MAPPED

El input contiene:

```text
tenant_external_id
```

y un resolver autorizado lo mapea.

---

# 145. Cross-tenant import

Solo mediante API administrativa explícita.

---

# 146. No tenant from untrusted field blindly

Nunca confiar:

```text
tenant_id from CSV
```

sin resolver/autorizar.

---

# 147. Authorization

Import deberá aplicar:

```text
who can import
into what target
for which tenant/domain
which fields
which persistence mode
```

---

# 148. Field-level restrictions

Podrá prohibirse importar directamente:

```text
role
is_admin
password_hash
billing_status
internal_version
```

sin permisos específicos.

---

# 149. Import mapping security

Mapping user-configurable deberá validar target fields.

---

# 150. Raw target identifiers

No deberán concatenarse directamente a SQL.

---

# 151. Secret data

Import Error Sink, telemetry y diagnostics deberán redactar:

```text
password
token
API key
sensitive PII
```

---

# 152. File security

El Import core Database no será responsable de upload security completa.

Pero un File Import Adapter deberá recibir un recurso ya autorizado/controlado.

---

# 153. Decompression

Podrán existir adapters:

```text
gzip
zip
bz2
```

pero deben controlar:

```text
decompression bombs
size ratios
max expanded bytes
```

---

# 154. Resource governance

Import deberá tener límites explícitos.

---

# 155. ImportResourceBudget

```php
final readonly class ImportResourceBudget
{
    public function __construct(
        public ?int $maxRecords,
        public ?int $maxSourceBytes,
        public ?int $maxExpandedBytes,
        public ?int $maxBatchRecords,
        public ?int $maxErrors,
        public ?int $maxDurationMs,
        public ?int $maxMemoryBytes,
        public ?int $maxTransformationDepth,
    ) {}
}
```

---

# 156. Maximum errors

Una policy útil:

```text
stop after 1,000 invalid records
```

para evitar procesar un archivo fundamentalmente incorrecto durante horas.

---

# 157. Error rate policy

También:

```text
stop if invalid rate > 25%
after minimum 1,000 records
```

---

# 158. ImportQualityPolicy

```php
final readonly class ImportQualityPolicy
{
    public function __construct(
        public ?int $maxErrors,
        public ?float $maxErrorRate,
        public int $minimumSample,
    ) {}
}
```

---

# 159. Memory

El pipeline deberá aspirar:

```text
O(batch size)
```

para fuentes streamable.

---

# 160. Pero

Puede crecer por:

```text
cross-record validation
global deduplication
error accumulation
transform caches
ORM state
```

---

# 161. Import should prefer rows, not managed entities

Para grandes imports, el camino principal será:

```text
record
→ row data
→ Bulk persistence
```

no:

```text
record
→ Entity
→ EntityManager
```

---

# 162. ORM import mode

Podrá existir una estrategia:

```text
ORM_PERSIST
```

cuando se requieran:

```text
domain invariants
lifecycle
relationships
entity events
```

pero será más costosa.

---

# 163. Import mode distinction

```text
ROW_BULK
≠
ORM_ENTITY
```

---

# 164. ORM Import

Ejemplo:

```php
->persistenceMode(
    ImportPersistenceMode::ORM_ENTITY
)
```

podrá construir entidades mediante:

```text
Factory/Mapper
→ EntityManager
→ UnitOfWork
```

---

# 165. No fake lifecycle

ROW_BULK no deberá disparar lifecycle events ORM como si hubieran existido entidades.

---

# 166. Relationship imports

Datos relacionales pueden requerir varias fases.

Ejemplo:

```text
customers.csv
orders.csv
order_items.csv
```

---

# 167. Import dependencies

Podrá modelarse:

```text
Customer Import
   ↓
Order Import
   ↓
OrderItem Import
```

pero esto será orquestación de imports, no responsabilidad de un simple record importer.

---

# 168. Generated IDs

Si input usa legacy IDs:

```text
legacy_customer_id
```

y DB genera nuevos IDs, puede requerirse un:

```text
ImportReferenceMap
```

---

# 169. ImportReferenceMap

```php
interface ImportReferenceMap
{
    public function put(
        ImportReferenceKey $external,
        DatabaseIdentifier $internal,
    ): void;

    public function get(
        ImportReferenceKey $external,
    ): ?DatabaseIdentifier;
}
```

---

# 170. Reference map storage

Para millones de registros no deberá ser necesariamente memoria.

Podrá usar:

```text
temporary DB table
KV store
file/index
custom provider
```

---

# 171. Reference map ≠ IdentityMap

```text
ImportReferenceMap
≠
ORM IdentityMap
```

---

# 172. Duplicate handling

Import puede encontrar duplicados:

```text
within source
against database
```

---

# 173. Duplicate policies

```php
enum ImportDuplicatePolicy
{
    case ERROR;
    case SKIP;
    case INSERT;
    case UPDATE_EXISTING;
    case UPSERT;
    case CUSTOM;
}
```

---

# 174. Duplicate detection

Puede depender de:

```text
source key
target unique key
database constraint
dedup state
```

---

# 175. Pre-check duplicates

No deberá utilizarse como garantía de concurrencia.

---

# 176. Database constraint final

Unique constraints continúan siendo autoridad final.

---

# 177. Batch error isolation

Cuando un Bulk Insert batch de 1000 falla porque una fila es inválida según DB constraint, pueden existir estrategias:

```text
FAIL_BATCH
BINARY_SPLIT
ROW_FALLBACK
CUSTOM
```

---

# 178. Binary split

Conceptualmente:

```text
1000 fail
↓
500 / 500
↓
identify failing subset
```

puede aislar registros malos.

---

# 179. Tradeoff

Aumenta:

```text
queries
latency
transaction complexity
```

---

# 180. Not default

No debería ser default universal.

---

# 181. Row fallback

Ejecutar una por una después de batch failure puede identificar errores, pero reduce rendimiento.

Debe ser opt-in.

---

# 182. Error attribution

Import debe intentar asociar errores a:

```text
source position
record index
field
target column
batch
```

sin inventar precisión.

---

# 183. Database batch errors

Algunos drivers no indican qué fila del multi-row insert causó un error.

Entonces:

```text
RecordCause = UNKNOWN
```

---

# 184. No false precision

Nunca atribuir arbitrariamente el error a:

```text
first row in batch
```

sin evidencia.

---

# 185. Dry run

Import debería soportar:

```php
->dryRun()
```

para:

```text
decode
map
normalize
validate
plan
```

sin persistir.

---

# 186. Dry run limitations

No puede garantizar que la ejecución real:

```text
will succeed
```

porque:

```text
database state changes
constraints
concurrency
routing
permissions
```

pueden variar.

---

# 187. Dry run ≠ transaction rollback

No se recomienda implementar dry-run como:

```text
execute everything
then rollback
```

por default.

---

# 188. Preview

Podrá existir:

```php
->preview(100);
```

que procese los primeros registros sin persistir.

---

# 189. Diagnostics

Ejemplo conceptual:

```php
DB::import()->explain($request);
```

---

# 190. Explain output

```text
DATABASE IMPORT PLAN

Source:
    CSV

Source:
    streamable
    seekable
    checkpointable

Target:
    customers

Mapping:
    customer_email → email
    customer_name  → name

Normalization:
    locale: es-MX
    timezone: America/Monterrey
    null tokens: ["NULL", "\\N"]

Validation:
    enabled

Persistence:
    BULK INSERT

Import Batch:
    5,000

Effective Bulk Batch:
    800

Transaction:
    PER PERSISTENCE BATCH

Resume:
    ENABLED

Checkpoint:
    AFTER CONFIRMED PERSISTENCE

Tenant:
    FIXED CONTEXT

Routing:
    SINGLE SHARD

Error Policy:
    REDIRECT TO ERROR SINK

Maximum Errors:
    1000

Estimated Memory:
    BOUNDED

ORM Lifecycle:
    BYPASSED
```

---

# 191. Import Plan

```php
final readonly class ImportPlan
{
    public function __construct(
        public ImportSourcePlan $source,
        public ImportDecodePlan $decode,
        public ImportMappingPlan $mapping,
        public ImportNormalizationPlan $normalization,
        public ImportValidationPlan $validation,
        public ImportTransformationPlan $transform,
        public ImportPersistencePlan $persistence,
        public ImportRoutingPlan $routing,
        public ImportTransactionPlan $transaction,
        public ImportRecoveryPlan $recovery,
        public ImportResourcePlan $resources,
    ) {}
}
```

---

# 192. Planner ≠ Runner

```text
ImportPlanner
```

no lee el archivo completo ni persiste registros.

Produce:

```text
ImportPlan
```

---

# 193. Runner

`ImportRunner` coordina las fases.

---

# 194. Runner ≠ Bulk System

Runner delega persistencia.

No deberá reimplementar:

```text
Bulk Insert
Bulk Update
Transaction Engine
Routing Engine
Compiler
Executor
```

---

# 195. Pause and resume

API conceptual:

```php
$handle = DB::import()->start($request);

$handle->pause();

$handle->resume();
```

La implementación concreta de control de procesos podrá vivir en una capa superior, pero Database deberá modelar:

```text
checkpoint
resume token
progress
state
```

---

# 196. Resume token

No deberá contener datos sensibles completos.

---

# 197. Stateful checkpoints

Para importaciones grandes será preferible un:

```text
ImportCheckpointStore
```

---

# 198. Checkpoint Store

```php
interface ImportCheckpointStore
{
    public function load(ImportId $id): ?ImportCheckpoint;

    public function save(
        ImportId $id,
        ImportCheckpoint $checkpoint,
    ): void;

    public function delete(ImportId $id): void;
}
```

---

# 199. Storage providers

Podrán implementarse sobre:

```text
database
cache
filesystem
external state store
```

pero core será agnóstico.

---

# 200. Persistent runtime safety

En FrankenPHP:

```text
Import A
Import B
```

no deberán compartir:

```text
current record
source reader
checkpoint
transaction
tenant
shard
error buffer
progress
```

---

# 201. Long-running imports

No deberán depender del request HTTP.

---

# 202. Request lifecycle

Una importación larga debería poder ejecutarse desde:

```text
CLI
Job/Queue
Worker
Dedicated process
```

---

# 203. Database layer independence

Import System no deberá depender directamente del Queue System.

Será integrable posteriormente.

---

# 204. Worker reuse

Después de terminar:

```text
reader closed
buffers released
transaction closed/returned
temporary resources cleaned
context reset
```

---

# 205. Temporary files

Si un adapter crea temporales:

```text
decompressed file
staging file
temporary relation
```

deberá existir ownership explícito.

---

# 206. Cleanup

Cleanup deberá ejecutarse en:

```text
SUCCESS
FAILURE
CANCEL
```

cuando sea seguro.

---

# 207. Cleanup failure

No deberá ocultar el error principal.

---

# 208. Import result

```php
final readonly class ImportResult
{
    public function __construct(
        public ImportStatus $status,
        public ImportProgress $progress,
        public ImportPersistenceSummary $persistence,
        public ImportErrorSummary $errors,
        public ?ImportCheckpoint $checkpoint,
        public ImportOutcomeEvidence $evidence,
    ) {}
}
```

---

# 209. Status

```php
enum ImportStatus
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

# 210. Error summary

Podrá incluir:

```text
records read
valid
invalid
persisted
skipped
duplicate
failed
unknown
```

---

# 211. UNKNOWN

Si una persistence batch tiene commit desconocido:

```text
records persisted
```

para ese batch puede ser:

```text
UNKNOWN
```

---

# 212. No fake counts

Nunca sumar:

```text
attempted rows
```

como:

```text
persisted rows
```

sin evidencia.

---

# 213. Telemetry

Eventos conceptuales:

```text
ImportPlanned
ImportStarted
ImportSourceOpened
ImportBatchRead
ImportRecordRejected
ImportBatchValidated
ImportPersistenceStarted
ImportPersistenceCompleted
ImportCheckpointSaved
ImportPaused
ImportResumed
ImportCompleted
ImportPartial
ImportCancelled
ImportFailed
ImportOutcomeUnknown
ImportSourceClosed
```

---

# 214. Métricas

```text
db.import.operations
db.import.duration
db.import.records_read
db.import.records_valid
db.import.records_invalid
db.import.records_persisted
db.import.batches
db.import.errors
db.import.checkpoints
db.import.resume_count
db.import.partial
db.import.unknown
```

---

# 215. Cardinality

No usar:

```text
source path
customer email
record ID
tenant ID
raw field values
raw error payload
```

como metric labels.

---

# 216. Security logging

Los diagnostics podrán mostrar:

```text
column names
phase
record index
error category
```

sin exponer contenido sensible.

---

# 217. Directory structure

```text
src/Quantum/Database/Import/
│
├── ImportRequest.php
├── ImportOptions.php
├── ImportPlanner.php
├── ImportPlan.php
├── ImportRunner.php
├── ImportResult.php
├── ImportStatus.php
├── ImportId.php
│
├── Source/
│   ├── ImportSource.php
│   ├── ImportSourceReader.php
│   ├── ImportSourceCapability.php
│   ├── ImportSourceContext.php
│   ├── ImportSourceCheckpoint.php
│   ├── CsvImportSource.php
│   ├── JsonImportSource.php
│   ├── NdjsonImportSource.php
│   ├── IterableImportSource.php
│   └── StreamImportSource.php
│
├── Decode/
│   ├── ImportDecoder.php
│   ├── CsvImportDecoder.php
│   ├── JsonImportDecoder.php
│   ├── NdjsonImportDecoder.php
│   ├── ImportDecodeContext.php
│   └── RawImportRecord.php
│
├── Mapping/
│   ├── ImportMapping.php
│   ├── ImportMapper.php
│   ├── ImportMappingPlan.php
│   └── ImportFieldMapping.php
│
├── Normalization/
│   ├── ImportNormalizer.php
│   ├── ImportNormalizationContext.php
│   ├── ImportNullPolicy.php
│   ├── NormalizedImportRecord.php
│   └── ImportLocalePolicy.php
│
├── Validation/
│   ├── ImportRecordValidator.php
│   ├── ImportValidationContext.php
│   ├── ImportValidationResult.php
│   ├── ImportInvalidRecordPolicy.php
│   └── ImportQualityPolicy.php
│
├── Transformation/
│   ├── ImportTransformer.php
│   ├── ImportTransformContext.php
│   ├── ImportTransformResult.php
│   └── ImportTransformationPipeline.php
│
├── Persistence/
│   ├── ImportPersistenceAdapter.php
│   ├── ImportPersistenceStrategy.php
│   ├── ImportPersistencePlan.php
│   ├── BulkInsertImportAdapter.php
│   ├── BulkUpdateImportAdapter.php
│   ├── UpsertImportAdapter.php
│   └── OrmImportAdapter.php
│
├── Duplicate/
│   ├── ImportDuplicatePolicy.php
│   └── ImportDuplicateResolver.php
│
├── Reference/
│   ├── ImportReferenceMap.php
│   ├── ImportReferenceKey.php
│   └── DatabaseImportReferenceMap.php
│
├── Batch/
│   ├── ImportRecordBatch.php
│   ├── ImportBatchPolicy.php
│   └── ImportBatchPlanner.php
│
├── Routing/
│   ├── ImportRoutingPlan.php
│   ├── ImportTenantResolver.php
│   └── ImportShardResolver.php
│
├── Transaction/
│   ├── ImportTransactionPolicy.php
│   └── ImportTransactionPlan.php
│
├── Recovery/
│   ├── ImportCheckpoint.php
│   ├── ImportCheckpointStore.php
│   ├── ImportRecoveryPlan.php
│   ├── ImportSourceFingerprint.php
│   └── ImportPipelineFingerprint.php
│
├── Progress/
│   ├── ImportProgress.php
│   ├── ImportProgressUnit.php
│   └── ImportProgressReporter.php
│
├── Error/
│   ├── ImportErrorSink.php
│   ├── ImportRecordError.php
│   ├── MemoryImportErrorSink.php
│   └── NullImportErrorSink.php
│
├── Resource/
│   ├── ImportResourceBudget.php
│   └── ImportResourcePlan.php
│
├── Diagnostics/
│   ├── ImportInspector.php
│   └── ImportExplainer.php
│
├── Telemetry/
│   └── ImportTelemetry.php
│
└── Exception/
    └── ...
```

---

# 218. Error hierarchy

```text
DatabaseException
└── ImportException
    ├── ImportSourceException
    ├── ImportDecodeException
    ├── ImportEncodingException
    ├── ImportMappingException
    ├── ImportNormalizationException
    ├── ImportValidationException
    ├── ImportTransformationException
    ├── ImportDuplicateException
    ├── ImportRoutingException
    ├── ImportPersistenceException
    ├── ImportCheckpointException
    │   ├── ImportCheckpointMismatchException
    │   └── ImportCheckpointPersistenceException
    ├── ImportReplayUnsafeException
    ├── ImportResourceException
    ├── ImportCancellationException
    └── ImportOutcomeUnknownException
```

---

# 219. Testing matrix

| Área | Caso |
|---|---|
| CSV | simple header |
| CSV | no header |
| CSV | duplicate headers |
| CSV | quoted multiline |
| JSON | array |
| JSON | invalid JSON |
| NDJSON | large stream |
| Encoding | UTF-8 |
| Encoding | legacy encoding |
| Mapping | rename fields |
| Mapping | missing field |
| Mapping | positional |
| Null | empty vs null |
| Locale | decimal parsing |
| Date | timezone |
| Validation | valid |
| Validation | invalid record |
| Validation | max errors |
| Transformation | deterministic |
| Transformation | exception |
| Input | generator |
| Input | stream |
| Persistence | insert |
| Persistence | update |
| Persistence | upsert |
| Persistence | ORM mode |
| Duplicate | source duplicate |
| Duplicate | DB conflict |
| Batch | effective size |
| Failure | batch failure |
| Failure | source failure |
| Checkpoint | save |
| Checkpoint | resume |
| Checkpoint | source changed |
| Checkpoint | pipeline changed |
| Retry | replay safe |
| Retry | unsafe |
| Tenant | fixed tenant |
| Tenant | mapped tenant |
| Tenant | unauthorized tenant |
| Sharding | one shard |
| Sharding | multiple shards |
| Transaction | per batch |
| Transaction | whole import |
| Outcome | partial |
| Outcome | unknown commit |
| Cancellation | graceful |
| Pause | checkpointed |
| Resource | memory bounded |
| Runtime | worker reset |

---

# 220. Architectural invariants

## DB-IMPORT-001
Import será distinto de Bulk Insert.

## DB-IMPORT-002
Import será distinto de Seeder.

## DB-IMPORT-003
Import será distinto de Fixture.

## DB-IMPORT-004
Import será distinto de Migration.

## DB-IMPORT-005
Import será distinto de Restore.

## DB-IMPORT-006
Import será un pipeline.

## DB-IMPORT-007
Source será distinto de Decoder.

## DB-IMPORT-008
Decoder será distinto de Mapper.

## DB-IMPORT-009
Mapping será distinto de Normalization.

## DB-IMPORT-010
Normalization será distinta de Validation.

## DB-IMPORT-011
Validation será distinta de Transformation.

## DB-IMPORT-012
Transformation será distinta de Persistence.

## DB-IMPORT-013
Persistence será delegada a subsystems especializados.

## DB-IMPORT-014
Import no generará SQL directamente.

## DB-IMPORT-015
Import no ejecutará Drivers directamente.

## DB-IMPORT-016
Bulk Insert seguirá siendo autoridad para insert masivo.

## DB-IMPORT-017
Bulk Update seguirá siendo autoridad para update masivo.

## DB-IMPORT-018
Query Engine seguirá siendo autoridad de queries.

## DB-IMPORT-019
Compiler seguirá siendo autoridad SQL.

## DB-IMPORT-020
Source podrá ser streamable.

## DB-IMPORT-021
Source podrá ser replayable.

## DB-IMPORT-022
Source replayability será distinta de persistence replay safety.

## DB-IMPORT-023
Source checkpoint será distinto de DB transaction state.

## DB-IMPORT-024
Checkpoint será distinto de exactly-once.

## DB-IMPORT-025
CSV será soportado.

## DB-IMPORT-026
JSON será soportado.

## DB-IMPORT-027
NDJSON será soportado.

## DB-IMPORT-028
Iterable source será soportado.

## DB-IMPORT-029
Stream source será soportado.

## DB-IMPORT-030
External adapters serán extensibles.

## DB-IMPORT-031
Core no dependerá de cloud SDKs concretos.

## DB-IMPORT-032
Decoder deberá preservar source position.

## DB-IMPORT-033
Raw payload retention será opcional.

## DB-IMPORT-034
Duplicate headers no serán ignorados silenciosamente.

## DB-IMPORT-035
Huge JSON no requerirá materialización completa cuando exista parser incremental.

## DB-IMPORT-036
Encoding será explícito o detectado bajo policy segura.

## DB-IMPORT-037
Encoding errors tendrán policy.

## DB-IMPORT-038
Mapping no realizará type conversion DB.

## DB-IMPORT-039
Missing field será distinto de null.

## DB-IMPORT-040
Normalization utilizará contexto de locale.

## DB-IMPORT-041
Date parsing utilizará timezone/contexto explícito.

## DB-IMPORT-042
Empty string será distinto de null por default.

## DB-IMPORT-043
External text parsing será distinto de Database Type Conversion.

## DB-IMPORT-044
Type System seguirá siendo autoridad de valores persistentes.

## DB-IMPORT-045
Validation podrá ser estructural.

## DB-IMPORT-046
Validation podrá ser por field.

## DB-IMPORT-047
Validation podrá ser por record.

## DB-IMPORT-048
Cross-record validation será explícita.

## DB-IMPORT-049
Database validation no sustituirá constraints.

## DB-IMPORT-050
DB constraints seguirán siendo autoridad final.

## DB-IMPORT-051
Invalid record policy será explícita.

## DB-IMPORT-052
Invalid records no serán ignorados silenciosamente.

## DB-IMPORT-053
Error sink será bounded o externalizable.

## DB-IMPORT-054
Sensitive record payloads no se loggearán por default.

## DB-IMPORT-055
Transformation será distinta de validation.

## DB-IMPORT-056
Transformation pipeline podrá tener fases.

## DB-IMPORT-057
Replayable imports deberán preferir transformations deterministas.

## DB-IMPORT-058
Clock podrá fijarse por operación.

## DB-IMPORT-059
Randomness podrá ser determinista bajo policy.

## DB-IMPORT-060
INSERT persistence delegará en Bulk Insert.

## DB-IMPORT-061
UPDATE persistence delegará en Bulk Update.

## DB-IMPORT-062
UPSERT tendrá semántica explícita.

## DB-IMPORT-063
Select-then-insert no se considerará atomic upsert.

## DB-IMPORT-064
Import batch será distinto de Bulk batch.

## DB-IMPORT-065
Bulk effective batch podrá ser menor al import batch.

## DB-IMPORT-066
Pipeline deberá ser bounded para sources streamable.

## DB-IMPORT-067
Full materialization no será default.

## DB-IMPORT-068
Progress tendrá unidades explícitas.

## DB-IMPORT-069
Unknown total permanecerá UNKNOWN.

## DB-IMPORT-070
Percentage no se inventará sin denominator confiable.

## DB-IMPORT-071
Checkpoints serán opcionales.

## DB-IMPORT-072
Checkpoint avanzará después del required persistence outcome.

## DB-IMPORT-073
Checkpoint failure después de commit podrá producir replay.

## DB-IMPORT-074
Checkpoint no implicará exactly-once.

## DB-IMPORT-075
Source fingerprint será validable.

## DB-IMPORT-076
Pipeline fingerprint será validable.

## DB-IMPORT-077
Source changed invalidará resume por default.

## DB-IMPORT-078
Mapping changed podrá invalidar resume.

## DB-IMPORT-079
Validation changed podrá invalidar resume.

## DB-IMPORT-080
Transformation changed podrá invalidar resume.

## DB-IMPORT-081
Persistence strategy changed podrá invalidar resume.

## DB-IMPORT-082
ImportId podrá persistir entre attempts.

## DB-IMPORT-083
AttemptId será distinto de ImportId.

## DB-IMPORT-084
PAUSED será distinto de CANCELLED.

## DB-IMPORT-085
Resume requerirá source/recovery capability suficiente.

## DB-IMPORT-086
Transaction policy será explícita.

## DB-IMPORT-087
WHOLE_IMPORT no será default para datasets grandes.

## DB-IMPORT-088
USE_EXISTING no committeará transacción ajena.

## DB-IMPORT-089
PER_BATCH podrá producir PARTIAL.

## DB-IMPORT-090
UNKNOWN commit permanecerá UNKNOWN.

## DB-IMPORT-091
No habrá blind resume después de unknown outcome.

## DB-IMPORT-092
Routing ocurrirá antes de persistencia.

## DB-IMPORT-093
Missing shard key no producirá broadcast.

## DB-IMPORT-094
Cross-shard import no implicará global ACID.

## DB-IMPORT-095
Tenant context será explícito.

## DB-IMPORT-096
Tenant IDs del input no serán confiados automáticamente.

## DB-IMPORT-097
Cross-tenant import requerirá autorización administrativa.

## DB-IMPORT-098
Field-level import restrictions serán soportables.

## DB-IMPORT-099
User-configurable target fields serán validados.

## DB-IMPORT-100
Raw identifiers no se concatenarán en SQL.

## DB-IMPORT-101
Compressed input tendrá expansion limits.

## DB-IMPORT-102
Resource governance limitará records.

## DB-IMPORT-103
Resource governance limitará bytes.

## DB-IMPORT-104
Resource governance limitará errores.

## DB-IMPORT-105
Error-rate cutoff será soportable.

## DB-IMPORT-106
Import preferirá row persistence para grandes datasets.

## DB-IMPORT-107
ORM import mode será explícito.

## DB-IMPORT-108
ROW_BULK no fingirá ORM lifecycle.

## DB-IMPORT-109
ORM mode podrá ser más costoso.

## DB-IMPORT-110
ImportReferenceMap será distinto de IdentityMap.

## DB-IMPORT-111
ReferenceMap podrá usar external storage.

## DB-IMPORT-112
Duplicate policy será explícita.

## DB-IMPORT-113
Pre-check de duplicates no sustituirá unique constraint.

## DB-IMPORT-114
Batch failure no se atribuirá a un record específico sin evidencia.

## DB-IMPORT-115
Binary split será opt-in.

## DB-IMPORT-116
Row fallback será opt-in.

## DB-IMPORT-117
Dry run no persistirá.

## DB-IMPORT-118
Dry run no garantizará éxito futuro.

## DB-IMPORT-119
Dry run no será rollback-based por default.

## DB-IMPORT-120
Preview será bounded.

## DB-IMPORT-121
Diagnostics no ejecutarán import completo.

## DB-IMPORT-122
Planner será distinto de Runner.

## DB-IMPORT-123
Runner será distinto de Bulk subsystems.

## DB-IMPORT-124
Import no duplicará Transaction System.

## DB-IMPORT-125
Import no duplicará Routing System.

## DB-IMPORT-126
Import no duplicará Validation System.

## DB-IMPORT-127
Pause/resume no dependerán del request HTTP.

## DB-IMPORT-128
Long-running import podrá vivir en CLI/job/worker.

## DB-IMPORT-129
Database Import no dependerá directamente del Queue System.

## DB-IMPORT-130
Temporary resources tendrán ownership explícito.

## DB-IMPORT-131
Cleanup ocurrirá en success/failure/cancel cuando sea seguro.

## DB-IMPORT-132
Cleanup failure no ocultará error principal.

## DB-IMPORT-133
Result separará progress, persistence y error summary.

## DB-IMPORT-134
Persisted counts no serán inventados.

## DB-IMPORT-135
Telemetry tendrá bounded cardinality.

## DB-IMPORT-136
Source paths sensibles no serán metric labels.

## DB-IMPORT-137
Raw records no serán metric labels.

## DB-IMPORT-138
Persistent runtime state será operation-scoped.

## DB-IMPORT-139
Source reader no se compartirá entre imports.

## DB-IMPORT-140
Current batch no será static state.

## DB-IMPORT-141
Checkpoint mutable no será global.

## DB-IMPORT-142
Tenant state no se filtrará entre imports.

## DB-IMPORT-143
Shard state no se filtrará entre imports.

## DB-IMPORT-144
Transaction state no se filtrará entre imports.

## DB-IMPORT-145
FrankenPHP worker reuse será seguro.

## DB-IMPORT-146
RoadRunner worker reuse será seguro.

## DB-IMPORT-147
OpenSwoole worker reuse será seguro.

## DB-IMPORT-148
Coroutine imports estarán aislados.

## DB-IMPORT-149
Immutable plans podrán compartirse solo cuando no contengan context state.

## DB-IMPORT-150
Import será explainable.

## DB-IMPORT-151
Source capabilities serán diagnosticables.

## DB-IMPORT-152
Mapping será diagnosticable.

## DB-IMPORT-153
Validation policy será diagnosticable.

## DB-IMPORT-154
Persistence strategy será diagnosticable.

## DB-IMPORT-155
Batching será diagnosticable.

## DB-IMPORT-156
Transaction scope será diagnosticable.

## DB-IMPORT-157
Resume safety será diagnosticable.

## DB-IMPORT-158
Replay safety será diagnosticable.

## DB-IMPORT-159
Routing será diagnosticable.

## DB-IMPORT-160
Resource limits serán diagnosticables.

## DB-IMPORT-161
Import no convertirá incertidumbre en certeza.

## DB-IMPORT-162
UNKNOWN permanecerá UNKNOWN.

## DB-IMPORT-163
Correctness tendrá prioridad sobre throughput.

## DB-IMPORT-164
Security tendrá prioridad sobre convenience mapping.

## DB-IMPORT-165
Database continuará siendo autoridad final del estado persistido.

---

# 221. Modelo formal

Sea:

```text
S
=
source sequence
```

con:

```text
S = {s1, s2, ..., sn}
```

Cada registro atraviesa funciones:

```text
D = Decode
M = Map
N = Normalize
V = Validate
T = Transform
P = Persist
```

Entonces:

```text
ri
=
T(
  V(
    N(
      M(
        D(si)
      )
    )
  )
)
```

si el registro supera todas las fases.

---

# 222. Registros inválidos

Si:

```text
V(ri) = invalid
```

su comportamiento depende de:

```text
ImportInvalidRecordPolicy
```

y puede:

```text
STOP
SKIP
REPORT
REDIRECT
```

---

# 223. Persistencia por batches

Los records válidos se agrupan:

```text
R_valid
→
I1, I2, ..., Ik
```

como import batches.

Cada `Ii` podrá convertirse a:

```text
B1, B2, ..., Bm
```

persistence batches según límites del subsistema Bulk.

---

# 224. Recovery model

Para checkpoint `Ci`:

```text
Read
↓
Transform
↓
Persist
↓
Required Outcome
↓
Save Ci
```

La reanudación inicia desde:

```text
SourceResume(Ci)
```

si el source y pipeline fingerprints siguen siendo compatibles.

---

# 225. Memory model

Para pipeline streaming:

```text
M_import
≈
M(source buffer)
+
M(import batch)
+
M(persistence batch)
+
M(error buffer)
+
M(validation state)
+
M(reference state)
```

Idealmente bounded.

---

# 226. Arquitectura final

```text
                         Import API
                             │
                             ▼
                       ImportRequest
                             │
                             ▼
                        ImportPlanner
                             │
                             ▼
                         ImportPlan
                             │
                             ▼
                        ImportRunner
                             │
                             ▼
                       ImportSource
                             │
                             ▼
                          Decoder
                             │
                             ▼
                         Raw Record
                             │
                             ▼
                          Mapper
                             │
                             ▼
                        Normalizer
                             │
                             ▼
                         Validator
                             │
                     ┌───────┴────────┐
                     ▼                ▼
                   Valid           Invalid
                     │                │
                     ▼                ▼
                Transformer       Error Policy
                     │
                     ▼
                Routing Resolver
                     │
                     ▼
                 Import Batch
                     │
                     ▼
             Persistence Adapter
          ┌──────────┼───────────┐
          ▼          ▼           ▼
      Bulk Insert Bulk Update   Upsert
          │          │           │
          └──────────┼───────────┘
                     ▼
               Database Outcome
                     │
              ┌──────┴──────┐
              ▼             ▼
          Progress       Checkpoint
              │             │
              └──────┬──────┘
                     ▼
                ImportResult
```

---

# 227. Regla maestra final

> **Database Import en VoltStack será una arquitectura de ingestión, no una simple llamada de inserción masiva: cada dato deberá atravesar fases explícitas de lectura, interpretación, mapping, normalización, validación, transformación, routing y persistencia antes de convertirse en estado persistente.**

Siempre:

```text
Import
≠
Bulk Insert
```

```text
Import Batch
≠
Bulk Batch
```

```text
Source Replayable
≠
Persistence Replay Safe
```

```text
Checkpoint
≠
Exactly Once
```

```text
Dry Run
≠
Guaranteed Future Success
```

```text
Pre-validation
≠
Database Constraint
```

```text
Mapping
≠
Type Conversion
```

```text
Empty String
≠
NULL
```

```text
Tenant Field
≠
Authorized Tenant
```

```text
Upsert
≠
SELECT + INSERT/UPDATE
```

```text
Progress
≠
Known Completion Percentage
```

y:

```text
UNKNOWN Persistence Outcome
≠
Safe Resume
```

---

# 228. Resultado arquitectónico

VoltStack podrá soportar imports como:

```php
DB::import()
    ->fromCsv('/imports/customers.csv')
    ->into('customers')
    ->map([
        'Customer ID'    => 'external_id',
        'Customer Name'  => 'name',
        'Email Address'  => 'email',
        'Created Date'   => 'created_at',
    ])
    ->locale('es-MX')
    ->timezone('America/Monterrey')
    ->validate([
        'external_id' => ['required'],
        'email'       => ['required', 'email'],
    ])
    ->onInvalid(
        ImportInvalidRecordPolicy::REDIRECT_TO_ERROR_SINK
    )
    ->persistUsing(
        ImportPersistenceStrategy::UPSERT
    )
    ->batchSize(5000)
    ->checkpointEveryBatch()
    ->run();
```

sin exigir cargar todo el archivo en memoria y reutilizando:

```text
Type System
Validation System
Bulk Insert
Bulk Update
Transactions
Routing
Sharding
Multitenancy
Telemetry
Security
```

en lugar de implementar versiones paralelas dentro de Import.

---

# 229. Bloque 19 hasta ahora

```text
✓ 199_DATABASE_PAGINATION_SYSTEM.md
✓ 200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
✓ 201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
✓ 202_DATABASE_LAZY_COLLECTION_SYSTEM.md
✓ 203_DATABASE_BULK_INSERT_SYSTEM.md
✓ 204_DATABASE_BULK_UPDATE_SYSTEM.md
✓ 205_DATABASE_BULK_DELETE_SYSTEM.md
✓ 206_DATABASE_IMPORT_SYSTEM.md

→ 207_DATABASE_EXPORT_SYSTEM.md
→ 208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md
```

La arquitectura ahora cubre:

```text
Large Reads
+
Bulk Mutations
+
External Data Ingestion
```

El siguiente sistema deberá resolver la dirección opuesta:

```text
Database
↓
Large Result Traversal
↓
Transformation
↓
Encoding
↓
External Destination
```

---

# 230. Siguiente documento

```text
207_DATABASE_EXPORT_SYSTEM.md
```

El siguiente documento definirá:

```text
Database Export System
├── export sources
├── Query Builder integration
├── ORM projection export
├── CSV
├── JSON
├── NDJSON
├── streams
├── files
├── remote destinations
├── chunk/cursor/lazy traversal
├── type serialization
├── formatting
├── column mapping
├── masking/redaction
├── authorization
├── tenant isolation
├── shard-aware export
├── checkpoints
├── resumability
├── compression
├── resource governance
├── cancellation
├── progress
├── telemetry
└── persistent-runtime safety
```

bajo la regla central:

> **Exportar datos en VoltStack no será equivalente a ejecutar una query y llamar `json_encode()`: el Export System será un pipeline de extracción, recorrido, transformación, serialización y entrega diseñado para datasets potencialmente masivos sin romper memoria, seguridad ni consistencia.**