# 330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md

## 1. Propósito

Este documento define el **Migration Intermediate Model (MIM)** de `VoltStack/Quantum/Database`.

El modelo intermedio constituye la representación neutral utilizada para describir una arquitectura de persistencia durante una migración, independientemente de si la fuente original utiliza:

```text
Laravel Eloquent
Doctrine ORM
Doctrine DBAL
PDO
mysqli
Raw SQL
Custom Active Record
Custom Data Mapper
DAO
Repositories
Stored Procedures
Legacy Database Layers
```

El MIM se sitúa entre:

```text
Source-specific analysis
```

y:

```text
VoltStack-native transformation
```

Su función principal es impedir que las decisiones internas de un ORM externo se filtren hacia el núcleo de `Quantum/Database`.

---

## 2. Dependencias documentales

Este documento continúa directamente:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md
327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md
328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md
329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md
```

y será utilizado por:

```text
331_DATABASE_MIGRATION_RULE_ENGINE.md
332_DATABASE_MIGRATION_CODE_TRANSFORMER.md
333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md
334_DATABASE_MIGRATION_BEHAVIOR_VERIFICATION_SYSTEM.md
335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
337_DATABASE_MIGRATION_TESTING_AND_VALIDATION_SYSTEM.md
338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md
339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 3. Problema arquitectónico

Una migración directa:

```text
Eloquent Model
      ↓
VoltStack Entity
```

o:

```text
Doctrine Entity
      ↓
VoltStack Entity
```

parece sencilla, pero crea acoplamiento conceptual.

Por ejemplo:

```text
Eloquent Scope
Doctrine Proxy
Doctrine UnitOfWork
PDO FETCH_ASSOC
Legacy DAO
```

son conceptos propios de sus respectivos sistemas.

VoltStack necesita comprender su **intención**, no copiar necesariamente su implementación.

---

## 4. Solución

Introducir:

```text
Migration Intermediate Model
```

como frontera semántica.

```text
Eloquent ─────┐
Doctrine ─────┤
PDO ──────────┤
Legacy SQL ───┼──► MIM ───► Migration Rules ───► VoltStack
Custom ORM ───┤
DBAL ─────────┘
```

---

## 5. Principio fundamental

```text
Source Syntax
     ↓
Source Semantics
     ↓
Neutral Migration Semantics
     ↓
VoltStack Semantics
     ↓
VoltStack Implementation
```

Nunca:

```text
Source API
     ↓
API name replacement
     ↓
VoltStack API
```

---

## 6. Objetivos

El MIM deberá ser:

```text
ORM-independent
framework-independent
database-aware
platform-aware
source-traceable
deterministic
serializable
versioned
extensible
validation-friendly
transformation-friendly
```

---

## 7. No objetivo

El MIM no será:

```text
runtime ORM
query execution engine
production persistence layer
domain model
replacement for VoltStack metadata
permanent compatibility API
```

Su existencia pertenece al proceso de migración.

---

## 8. Ciclo de vida

```text
Source
  │
  ▼
Analysis
  │
  ▼
Source Descriptor
  │
  ▼
Normalization
  │
  ▼
MIM
  │
  ▼
Rules
  │
  ▼
Target Model
  │
  ▼
Verification
```

---

## 9. Modelo raíz

Conceptualmente:

```php
final readonly class MigrationIntermediateModel
{
    public function metadata(): MigrationModelMetadata;

    public function connections(): array;

    public function entities(): array;

    public function valueObjects(): array;

    public function fields(): array;

    public function identifiers(): array;

    public function relationships(): array;

    public function queries(): array;

    public function repositories(): array;

    public function transactions(): array;

    public function types(): array;

    public function lifecycle(): array;

    public function schema(): MigrationSchemaModel;

    public function procedures(): array;

    public function behaviors(): array;

    public function evidence(): array;
}
```

---

## 10. Modelo versionado

Cada representación deberá declarar:

```text
mim_version
```

Ejemplo:

```json
{
  "mim_version": "1.0"
}
```

La versión del MIM será independiente de:

```text
VoltStack Framework version
Database package version
source ORM version
```

---

## 11. Razón del versionado

Permite:

```text
reproducible migrations
snapshot compatibility
rule versioning
migration report stability
future MIM evolution
```

---

## 12. Metadata raíz

```text
MigrationModelMetadata
```

contendrá:

```text
MIM version
analysis snapshot ID
target VoltStack version
source revision
created at
source adapters
database platforms
schema fingerprint
```

---

## 13. Stable IDs

Cada objeto del MIM deberá poseer un identificador estable dentro del snapshot.

Ejemplos:

```text
entity:user
field:user.email
relationship:post.user
query:user.find_by_email
connection:default
transaction:checkout.complete
```

---

## 14. Stable ID ≠ Runtime ID

Los IDs del MIM son identificadores de análisis.

No son:

```text
database primary keys
entity IDs
UUIDs de negocio
```

---

## 15. Source References

Cada elemento podrá conservar:

```text
MigrationSourceReference
```

con:

```text
adapter
source type
file
line
class
method
metadata location
schema object
```

---

## 16. Evidence References

Un elemento podrá estar respaldado por múltiples evidencias.

```text
Entity User
├── Eloquent Model
├── Migration File
└── Actual Table
```

---

## 17. Provenance

El MIM deberá permitir responder:

```text
Where did this information come from?
```

sin depender del adapter después de la normalización.

---

## 18. Confidence

Cada propiedad inferida podrá almacenar:

```text
HIGH
MEDIUM
LOW
UNKNOWN
```

cuando sea necesario.

---

## 19. Explicit vs Inferred

Se distinguirá:

```text
EXPLICIT
INFERRED
OBSERVED
OVERRIDDEN
```

Ejemplo:

```text
table name:
users

origin:
INFERRED_FROM_ELOQUENT_CONVENTION
```

frente a:

```text
origin:
EXPLICIT_MODEL_CONFIGURATION
```

---

## 20. Intermediate Connection Model

```text
MigrationConnection
```

describirá:

```text
logical name
driver
platform
database
schema
charset
collation
read/write role
topology
transaction capabilities
session behavior
credential source reference
```

Nunca incluirá secretos en claro.

---

## 21. Credential Representation

El MIM podrá almacenar:

```text
credential_source = ENV:DATABASE_PASSWORD
```

pero no:

```text
credential_value = actual_password
```

---

## 22. Connection Roles

```text
PRIMARY
READ
WRITE
REPLICA
REPORTING
AUDIT
TENANT
LEGACY
CUSTOM
```

---

## 23. Connection Topology

Ejemplo:

```text
default
├── write → primary
└── read
    ├── replica-1
    └── replica-2
```

El MIM preservará intención, no necesariamente implementación de pooling.

---

## 24. Platform Capabilities

Una conexión podrá registrar capacidades relevantes:

```text
savepoints
sequences
returning
json
generated columns
advisory locks
transactional DDL
```

---

## 25. Entity Model

```text
MigrationEntity
```

representará un objeto con identidad persistente.

Campos conceptuales:

```text
name
source class
physical table
identifier
fields
relationships
repository
lifecycle
inheritance
read/write capability
```

---

## 26. Entity vs Record

El MIM distinguirá:

```text
ENTITY
RECORD
ROW_MODEL
DTO
VALUE_OBJECT
EMBEDDABLE
READ_MODEL
```

No todo resultado SQL será tratado como entidad.

---

## 27. Row Model

Una query PDO que devuelve:

```text
associative array
```

puede representarse como:

```text
ROW_MODEL
```

sin inventar una entidad.

---

## 28. Read Model

Reportes y proyecciones podrán representarse como:

```text
READ_MODEL
```

aunque provengan de múltiples tablas.

---

## 29. Value Object

```text
MigrationValueObject
```

representará estructuras sin identidad independiente.

Ejemplos:

```text
Money
Address
Coordinates
DateRange
```

---

## 30. Embeddable

Un embeddable Doctrine podrá normalizarse como Value Object con información de persistencia embebida.

---

## 31. Physical Mapping

Cada entidad podrá mapear:

```text
catalog
schema
table
```

de forma explícita.

Esto evita depender posteriormente de convenciones del source ORM.

---

## 32. Logical vs Physical Name

Se distinguirá:

```text
logical_name = User
physical_name = users
```

---

## 33. Field Model

```text
MigrationField
```

contendrá:

```text
logical name
property name
column name
intermediate type
native database type
nullable
default
length
precision
scale
generated
readonly
serialization
conversion
```

---

## 34. Intermediate Type

Los tipos se representarán mediante conceptos neutrales.

Ejemplos:

```text
STRING
TEXT
INTEGER
BIG_INTEGER
SMALL_INTEGER
BOOLEAN
DECIMAL
FLOAT
DATE
TIME
DATETIME
DATETIME_IMMUTABLE
JSON
BINARY
BLOB
UUID
ULID
ENUM
CUSTOM
PLATFORM_NATIVE
```

---

## 35. Source Type

También se conservará:

```text
source_type
```

Ejemplos:

```text
Eloquent decimal:2
Doctrine decimal
PDO string
MySQL DECIMAL(18,4)
```

---

## 36. Database Native Type

Cuando sea relevante:

```text
native_type = NUMERIC(18,4)
```

se conservará por separado.

---

## 37. Type Conversion Model

```text
MigrationTypeConversion
```

podrá describir:

```text
database → PHP
PHP → database
serialization
normalization
binding
```

---

## 38. Custom Types

Un tipo custom no deberá colapsarse prematuramente a `string`.

Ejemplo:

```text
CUSTOM:
money
```

con referencias a sus conversiones.

---

## 39. Decimal

El MIM deberá conservar:

```text
precision
scale
rounding assumptions where known
PHP representation
```

---

## 40. Date/Time

Se conservará:

```text
database type
timezone assumptions
PHP representation
mutability
precision
```

---

## 41. Boolean Representation

Podrá representar:

```text
native boolean
0/1
Y/N
custom mapping
```

---

## 42. JSON

Se distinguirá:

```text
JSON
JSONB
TEXT_ENCODED_JSON
CUSTOM_JSON_CODEC
```

cuando sea conocido.

---

## 43. Serialized Values

Ejemplos:

```text
PHP_SERIALIZED
JSON_SERIALIZED
CSV
CUSTOM_CODEC
ENCRYPTED_PAYLOAD
```

---

## 44. Identifier Model

```text
MigrationIdentifier
```

contendrá:

```text
fields
strategy
generator
database behavior
source behavior
```

---

## 45. Identifier Strategies

```text
ASSIGNED
AUTO_INCREMENT
IDENTITY
SEQUENCE
UUID
ULID
COMPOSITE
FOREIGN
CUSTOM
UNKNOWN
```

---

## 46. Composite Identifiers

Se representarán como:

```text
Identifier
├── field A
└── field B
```

sin forzar un ID artificial.

---

## 47. Generated Values

El modelo podrá indicar:

```text
generated_by:
APPLICATION
DATABASE
SEQUENCE
TRIGGER
CUSTOM
UNKNOWN
```

---

## 48. Relationship Model

```text
MigrationRelationship
```

describirá:

```text
source
target
cardinality
ownership
foreign key
join table
nullable
cascade
fetch intent
orphan semantics
ordering
polymorphism
```

---

## 49. Cardinalities

```text
ONE_TO_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
POLYMORPHIC_ONE
POLYMORPHIC_MANY
CUSTOM
```

---

## 50. Ownership

Se representará:

```text
OWNING
INVERSE
SHARED
IMPLICIT
UNKNOWN
```

sin imponer terminología Doctrine al target.

---

## 51. Foreign Key Model

```text
MigrationForeignKey
```

podrá contener:

```text
local columns
referenced table
referenced columns
on update
on delete
deferrable
physical constraint existence
```

---

## 52. Logical Relationship without FK

El MIM permitirá:

```text
relationship exists
physical_fk = false
```

común en sistemas legacy.

---

## 53. Join Table

```text
MigrationJoinTable
```

describirá:

```text
table
source columns
target columns
extra columns
constraints
```

---

## 54. Association Entity Candidate

Si una join table contiene:

```text
quantity
role
created_at
price
```

podrá marcarse:

```text
ASSOCIATION_ENTITY_CANDIDATE
```

sin transformarla automáticamente.

---

## 55. Cascade Model

En lugar de copiar APIs del ORM:

```text
CascadeIntent
```

podrá expresar:

```text
PERSIST
REMOVE
REFRESH
DETACH
ALL
CUSTOM
```

---

## 56. Orphan Semantics

Se conservará como concepto independiente de cascade.

---

## 57. Fetch Intent

El MIM podrá describir:

```text
LAZY
EAGER
EXTRA_LAZY
BATCH
EXPLICIT
UNKNOWN
```

como intención observada.

---

## 58. Polymorphism

Relaciones polimórficas deberán representar explícitamente:

```text
type discriminator
identifier
type map
physical columns
```

---

## 59. Inheritance Model

```text
MigrationInheritance
```

podrá describir:

```text
NONE
SINGLE_TABLE
JOINED
TABLE_PER_CLASS
CUSTOM
```

según fuente y capacidad conocida.

---

## 60. Discriminator

Se conservará:

```text
column
type
map
stored values
```

para evitar cambios accidentales de datos.

---

## 61. Repository Model

```text
MigrationRepository
```

describirá:

```text
entity/read model
source class
methods
dependencies
query usage
transaction assumptions
return contracts
```

---

## 62. Repository Method

```text
MigrationRepositoryMethod
```

podrá incluir:

```text
name
parameters
return shape
queries
side effects
exceptions
transaction requirements
```

---

## 63. Query Model

Una pieza central será:

```text
MigrationQuery
```

---

## 64. Query Kinds

```text
SELECT
INSERT
UPDATE
DELETE
UPSERT
MERGE
DDL
PROCEDURE_CALL
FUNCTION_CALL
RAW
UNKNOWN
```

---

## 65. Query Representation

Cuando sea posible:

```text
MigrationQueryAST
```

contendrá:

```text
sources
joins
projection
predicates
grouping
having
ordering
pagination
locking
returning
parameters
```

---

## 66. Raw Query Preservation

Si una query no puede analizarse:

```text
representation = RAW
```

con:

```text
original SQL
platform
parameter metadata
source reference
```

El SQL deberá sanitizarse apropiadamente para outputs.

---

## 67. Query Source

Se conservará:

```text
ELOQUENT
DOCTRINE_DQL
DOCTRINE_DBAL
PDO
RAW_SQL
CUSTOM_BUILDER
STORED_PROCEDURE
```

solo como provenance, no como semántica target.

---

## 68. Query Parameters

```text
MigrationQueryParameter
```

describirá:

```text
name/position
type
nullable
binding type
source expression where known
```

No necesita almacenar valores runtime.

---

## 69. Result Contract

```text
MigrationResultContract
```

podrá representar:

```text
ENTITY
ENTITY_COLLECTION
ROW
ROW_COLLECTION
SCALAR
SCALAR_COLLECTION
DTO
STREAM
CURSOR
AFFECTED_ROWS
VOID
CUSTOM
```

---

## 70. Cardinality

Para consultas individuales:

```text
ZERO_OR_ONE
EXACTLY_ONE
ONE_OR_MORE
ZERO_OR_MORE
```

---

## 71. Ordering Semantics

El MIM deberá preservar ordering explícito.

No deberá asumir que una consulta sin `ORDER BY` posee orden estable.

---

## 72. Pagination

Se representará:

```text
OFFSET
CURSOR
KEYSET
CUSTOM
```

con su semántica conocida.

---

## 73. Locking

```text
MigrationLockIntent
```

podrá expresar:

```text
OPTIMISTIC
PESSIMISTIC_READ
PESSIMISTIC_WRITE
ADVISORY
TABLE
CUSTOM
```

---

## 74. Native SQL Extensions

Una query podrá declarar:

```text
platform_features
```

por ejemplo:

```text
POSTGRES_RETURNING
MYSQL_ON_DUPLICATE_KEY
SQLSERVER_TOP
```

---

## 75. Transaction Model

```text
MigrationTransaction
```

representará una unidad transaccional observada o declarada.

---

## 76. Transaction Properties

```text
connection
begin boundary
commit boundary
rollback paths
isolation
savepoints
retry policy
participants
nested semantics
```

---

## 77. Transaction Participants

Podrán ser:

```text
repository
query
service
procedure
external adapter
```

---

## 78. Cross-source Transaction

Ejemplo:

```text
Transaction
├── Eloquent write
└── PDO write
```

se conservará como una sola intención transaccional si la evidencia lo demuestra.

---

## 79. Unknown Transaction Semantics

Si no pueden inferirse:

```text
UNKNOWN
```

será preferible a inventar atomicidad.

---

## 80. Unit of Work Semantics

El MIM podrá representar conceptos neutrales:

```text
TRACKED_PERSISTENCE
DEFERRED_WRITE
EXPLICIT_FLUSH
IMMEDIATE_WRITE
```

sin almacenar una instancia de Doctrine UnitOfWork.

---

## 81. Persistence Context

Podrá representarse intención de:

```text
identity consistency
managed entity scope
clear/reset boundaries
```

sin copiar EntityManager.

---

## 82. Change Tracking

```text
IMPLICIT
EXPLICIT
SNAPSHOT
NOTIFICATION
NONE
CUSTOM
UNKNOWN
```

podrá utilizarse como semántica intermedia.

---

## 83. Lifecycle Model

```text
MigrationLifecycleHook
```

describirá:

```text
event intent
timing
subject
handler
side effects
transaction relation
```

---

## 84. Lifecycle Timing

El MIM deberá poder diferenciar:

```text
BEFORE_VALIDATION
BEFORE_PERSIST
BEFORE_SQL
AFTER_SQL
BEFORE_COMMIT
AFTER_COMMIT
AFTER_HYDRATE
AFTER_LOAD
BEFORE_REMOVE
AFTER_REMOVE
CUSTOM
```

---

## 85. Source Event Names

Se conservarán como provenance:

```text
Eloquent creating
Doctrine PrePersist
legacy beforeSave
```

pero el concepto intermedio será neutral.

---

## 86. Side Effect Model

Los hooks podrán declarar efectos:

```text
DATABASE_WRITE
DOMAIN_EVENT
EXTERNAL_IO
CACHE
AUDIT
MUTATION
UNKNOWN
```

---

## 87. Database Trigger Model

```text
MigrationDatabaseTrigger
```

se incorporará al modelo de comportamiento, aunque no sea PHP.

---

## 88. Procedure Model

```text
MigrationProcedure
```

contendrá:

```text
name
platform
input parameters
output parameters
result sets
tables affected
transaction behavior
side effects
```

cuando sean conocidos.

---

## 89. Function Model

Funciones de database podrán representarse separadamente cuando tengan impacto en queries o tipos.

---

## 90. Schema Model

```text
MigrationSchemaModel
```

contendrá:

```text
tables
columns
indexes
unique constraints
foreign keys
sequences
views
materialized views
triggers
generated columns
procedures
functions
```

---

## 91. Schema Evidence Layers

Cada propiedad podrá contener valores provenientes de:

```text
DECLARED_METADATA
MIGRATION_HISTORY
ACTUAL_SCHEMA
LEGACY_ASSUMPTION
```

---

## 92. Conflict Representation

El MIM no deberá eliminar conflictos.

Ejemplo:

```text
Column amount

Doctrine metadata:
decimal(12,2)

Actual schema:
decimal(18,4)
```

se representará mediante:

```text
MigrationConflict
```

---

## 93. Conflict Status

```text
UNRESOLVED
RESOLVED_BY_RULE
RESOLVED_BY_OVERRIDE
VERIFIED
```

---

## 94. Schema Baseline

Cuando el proyecto adopte baseline:

```text
MigrationSchemaBaseline
```

representará el punto de partida para futuras migrations VoltStack.

---

## 95. Index Model

Se conservarán:

```text
columns
order where supported
unique
partial predicate
expression
platform options
```

---

## 96. Constraint Model

El modelo deberá distinguir:

```text
PRIMARY_KEY
FOREIGN_KEY
UNIQUE
CHECK
EXCLUSION
CUSTOM
```

según plataforma.

---

## 97. Generated Columns

Se conservarán:

```text
expression
stored/virtual behavior
platform
```

cuando puedan introspeccionarse.

---

## 98. Default Values

El MIM distinguirá:

```text
literal default
database expression
application default
unknown
```

---

## 99. Behavior Model

No toda semántica cabe en schema/query/entity.

Para ello existirá:

```text
MigrationBehavior
```

---

## 100. Behavior Examples

```text
soft delete
automatic timestamps
tenant filtering
audit trail
optimistic locking
global scope
automatic encryption
slug generation
```

---

## 101. Soft Delete

Representación neutral:

```text
Behavior:
SOFT_DELETE

column:
deleted_at

active predicate:
deleted_at IS NULL

restore supported:
true
```

---

## 102. Automatic Timestamps

```text
Behavior:
AUTOMATIC_TIMESTAMPS

created:
created_at

updated:
updated_at
```

sin depender de `$timestamps` de Eloquent.

---

## 103. Tenant Filtering

```text
Behavior:
TENANT_ISOLATION

strategy:
shared_table

tenant_key:
tenant_id
```

cuando sea conocido.

---

## 104. Encryption

```text
Behavior:
FIELD_ENCRYPTION
```

podrá registrar intención y codec reference sin almacenar claves.

---

## 105. Validation Relation

El MIM podrá registrar constraints relevantes observadas en persistence, pero no sustituirá el sistema general de Validation.

---

## 106. Authentication Data

Campos de autenticación podrán marcarse:

```text
SENSITIVE_AUTH_DATA
```

para impedir transformaciones peligrosas de hashes/tokens/MFA secrets.

---

## 107. Authorization Data

Relaciones utilizadas por roles/permissions podrán marcarse como críticas si son detectadas.

---

## 108. Cache Behavior

El MIM podrá registrar:

```text
query result cache
entity cache
repository cache
```

como comportamiento, no como propiedad esencial de entidad.

---

## 109. Source-specific Extensions

Un adapter podrá añadir metadata adicional mediante:

```text
extensions
```

pero deberá usar namespaces.

Ejemplo:

```text
extensions.doctrine.change_tracking
extensions.eloquent.cast_source
```

---

## 110. Extension Rule

Los campos source-specific:

```text
must not be required
```

para interpretar la semántica neutral principal.

---

## 111. Unknown Extensions

El MIM deberá poder preservar información no comprendida:

```text
OpaqueMigrationExtension
```

para evitar pérdida durante análisis.

---

## 112. Migration Annotation

Los objetos podrán contener:

```text
MigrationAnnotation
```

con:

```text
compatibility
confidence
risk
manual review
rule candidates
verification requirements
```

---

## 113. Separation of Facts and Decisions

El MIM deberá diferenciar:

```text
FACT
```

de:

```text
MIGRATION_DECISION
```

Ejemplo:

```text
FACT:
table = users

DECISION:
preserve table name
```

---

## 114. Why This Matters

Sin esta separación, un análisis posterior no podría distinguir qué descubrió la herramienta y qué decidió el plan.

---

## 115. Source Descriptor vs MIM

Ejemplo:

```text
Eloquent:
$casts['amount'] = 'decimal:2'
```

Source Descriptor:

```text
EloquentCastDescriptor
```

MIM:

```text
Field Type:
DECIMAL

scale:
2

source representation:
ELOQUENT_CAST
```

---

## 116. Doctrine Example

Doctrine:

```php
#[Column(type: 'decimal', precision: 18, scale: 4)]
```

MIM:

```text
Type:
DECIMAL

precision:
18

scale:
4
```

---

## 117. PDO Example

PDO:

```php
$row['amount']
```

más schema:

```text
DECIMAL(18,4)
```

MIM podrá inferir:

```text
Type:
DECIMAL

confidence:
HIGH for DB representation
UNKNOWN/MEDIUM for PHP semantic representation
```

---

## 118. Normalization Pipeline

```text
Source Descriptor
      │
      ▼
Source Normalizer
      │
      ▼
Canonical Value
      │
      ▼
Evidence Merge
      │
      ▼
Conflict Detection
      │
      ▼
MIM Object
```

---

## 119. Normalizer Contract

```php
interface MigrationSourceNormalizerInterface
{
    public function supports(
        SourceDescriptor $descriptor
    ): bool;

    public function normalize(
        SourceDescriptor $descriptor,
        MigrationNormalizationContext $context
    ): array;
}
```

---

## 120. Normalization Context

Podrá incluir:

```text
platform
schema evidence
target version
source adapter
project overrides
```

---

## 121. Canonical Names

Enums y categorías internas deberán utilizar nombres estables.

No deberán reutilizar constantes externas de Eloquent/Doctrine.

---

## 122. Immutability

Los objetos MIM deberán ser preferentemente:

```text
immutable
```

Una transformación no deberá alterar silenciosamente el snapshot original.

---

## 123. Transformation Copies

Las reglas producirán:

```text
Target Migration Model
```

o una nueva versión derivada, manteniendo trazabilidad.

---

## 124. Serialization

El MIM deberá serializarse a un formato machine-readable.

Formato recomendado:

```text
JSON
```

---

## 125. JSON Snapshot

Ejemplo simplificado:

```json
{
  "mim_version": "1.0",
  "entities": [
    {
      "id": "entity:user",
      "name": "User",
      "table": "users"
    }
  ]
}
```

---

## 126. Canonical Serialization

Para reproducibilidad se deberán definir:

```text
stable ordering
stable enum values
normalized paths
deterministic IDs
```

---

## 127. Hashing

Un snapshot podrá producir:

```text
MIM fingerprint
```

para verificar si el modelo analizado cambió.

---

## 128. Snapshot Integrity

El fingerprint no deberá depender de:

```text
timestamps
temporary paths
random IDs
```

si se pretende comparar semántica.

---

## 129. Diff

Dos modelos podrán compararse:

```text
Entity added
Field changed
Query removed
Transaction changed
Relationship changed
Schema conflict resolved
```

---

## 130. MIM Diff Use Cases

```text
CI
migration progress
rollback
analysis regression
source evolution during migration
```

---

## 131. Validation

Antes de utilizar el MIM deberá ejecutarse:

```text
MigrationIntermediateModelValidator
```

---

## 132. Structural Validation

Comprobará:

```text
unique stable IDs
valid references
known enum values
valid field ownership
valid relationship targets
valid query references
```

---

## 133. Semantic Validation

Comprobará inconsistencias como:

```text
relationship references missing entity
composite ID missing field
query references unknown table
transaction references unknown connection
```

---

## 134. Conflict Validation

Un conflicto no resuelto podrá impedir determinadas transformaciones.

---

## 135. Validation Levels

```text
VALID
VALID_WITH_WARNINGS
INCOMPLETE
INVALID
```

---

## 136. Partial Models

El MIM podrá ser incompleto.

Ejemplo:

```text
runtime evidence unavailable
```

No deberá invalidarse todo el modelo por ello.

---

## 137. Unknown Values

El modelo deberá representar explícitamente:

```text
UNKNOWN
```

cuando sea semánticamente distinto de:

```text
NULL
false
empty
```

---

## 138. Null vs Unknown

Ejemplo:

```text
nullable = true
```

es un hecho.

```text
nullable = UNKNOWN
```

significa que aún no pudo determinarse.

---

## 139. Security

Los snapshots MIM se consideran artefactos potencialmente sensibles.

No deberán almacenar:

```text
passwords
tokens
database credentials
raw production PII
encryption keys
full sensitive query parameter values
```

---

## 140. SQL Storage

Si se conserva SQL para transformación, deberá evaluarse si contiene literales sensibles.

El sistema podrá:

```text
retain source reference
sanitize report representation
store protected local analysis artifact
```

según política.

---

## 141. Data Samples

El MIM no necesita almacenar filas de producción.

Los ejemplos de datos deberán residir en fixtures de verificación separadas.

---

## 142. Multi-tenancy

El modelo deberá poder representar:

```text
shared database/shared schema
shared database/separate schema
database per tenant
dynamic connection per tenant
```

sin hacer de Multitenancy una dependencia obligatoria del core.

---

## 143. Tenant Context

Cuando exista:

```text
MigrationTenantBehavior
```

podrá describir cómo se selecciona:

```text
connection
schema
tenant key
filter
```

---

## 144. SaaS

El MIM no dependerá del paquete SaaS.

Si SaaS está instalado, podrá aportar metadata adicional mediante extensions.

---

## 145. FrankenPHP

El modelo podrá registrar requisitos de lifecycle:

```text
RESET_CONNECTION_STATE
RESET_PERSISTENCE_CONTEXT
CLEAR_IDENTITY_MAP
ROLLBACK_OPEN_TRANSACTION
CLEAR_TENANT_CONTEXT
```

---

## 146. Runtime Requirement Model

```text
MigrationRuntimeRequirement
```

permitirá representar estas obligaciones sin acoplarlas a un servidor concreto.

---

## 147. Server Profiles

Después podrán evaluarse contra:

```text
FrankenPHP
RoadRunner
OpenSwoole
traditional PHP-FPM
```

---

## 148. Performance Metadata

El MIM podrá asociar observaciones:

```text
query count
latency
rows
memory
lazy loads
```

pero estas métricas no formarán parte de la semántica estable principal.

---

## 149. Observed Metrics

Se almacenarán como:

```text
MigrationObservation
```

con contexto de entorno y ventana de observación.

---

## 150. Rule Engine Input

`331_DATABASE_MIGRATION_RULE_ENGINE.md` recibirá objetos MIM.

Ejemplo:

```text
MigrationField
type = DECIMAL
precision = 18
scale = 4
```

y no:

```text
Doctrine\Column
```

---

## 151. Code Transformer Input

El Code Transformer recibirá:

```text
migration decisions
source references
MIM semantics
selected rules
```

para generar cambios.

---

## 152. Schema Compatibility Input

El documento 333 utilizará:

```text
MigrationSchemaModel
```

para comparar source intent, actual schema y target expectation.

---

## 153. Behavior Verification Input

El documento 334 utilizará:

```text
queries
result contracts
lifecycle
transactions
behaviors
```

para generar verificaciones.

---

## 154. Dual Runtime Input

El documento 335 utilizará:

```text
ownership
connections
entities
tables
transactions
runtime requirements
```

para definir coexistencia segura.

---

## 155. Shadow Query Input

El documento 336 utilizará:

```text
MigrationQuery
MigrationResultContract
```

para construir comparaciones equivalentes.

---

## 156. Testing Input

El documento 337 podrá derivar tests de:

```text
entities
queries
relationships
transactions
types
lifecycle
```

---

## 157. CLI Representation

El documento 338 deberá permitir inspeccionar objetos MIM.

Ejemplo conceptual:

```bash
php volt database:migrate:model entity:user
```

---

## 158. Reporting

El documento 339 podrá renderizar MIM como:

```text
tables
graphs
migration reports
diagnostics
```

sin depender de adapters source.

---

## 159. Rollback

El documento 340 podrá conservar snapshots MIM pre/post para comprender qué transformación se ejecutó.

---

## 160. MIM Repository

Conceptualmente:

```php
interface MigrationIntermediateModelRepositoryInterface
{
    public function save(
        MigrationIntermediateModel $model
    ): void;

    public function load(
        string $snapshotId
    ): MigrationIntermediateModel;

    public function diff(
        string $left,
        string $right
    ): MigrationModelDiff;
}
```

---

## 161. Storage

Ubicación local posible:

```text
.voltstack/database-migration/models/
```

---

## 162. Repository Security

Los snapshots no deberán publicarse accidentalmente dentro de:

```text
public/
web root
compiled frontend assets
```

---

## 163. Schema Evolution

La versión MIM podrá evolucionar.

Ejemplo:

```text
MIM 1.0
 ↓
MIM 1.1
 ↓
MIM 2.0
```

---

## 164. MIM Migration

Si cambia el formato, podrán existir migradores del propio modelo:

```text
MigrationIntermediateModelUpgrader
```

---

## 165. Backward Compatibility

Los cambios compatibles podrán añadir campos opcionales.

Cambios incompatibles requerirán nueva major del MIM.

---

## 166. V1 Freeze

Para Database V1 deberá definirse una versión estable:

```text
MIM 1.x
```

antes de declarar estable el sistema de migración.

---

## 167. Extension Namespace

Ejemplo JSON:

```json
{
  "extensions": {
    "doctrine": {},
    "eloquent": {},
    "project": {}
  }
}
```

---

## 168. Core Independence

Ninguna clase core del MIM deberá requerir:

```text
Illuminate\Database
Doctrine\ORM
Doctrine\DBAL
```

---

## 169. Package Boundary

Arquitectura:

```text
Database-Migration-Eloquent
          │
          ▼
Database-Migration-Core
          ▲
          │
Database-Migration-Doctrine
          ▲
          │
Database-Migration-Legacy
```

El MIM vive en:

```text
Database-Migration-Core
```

---

## 170. Suggested Namespace

Conceptualmente:

```text
VoltStack\Database\Migration\Model
```

o dentro de la convención final de paquetes Quantum.

---

## 171. Core Components

```text
MigrationIntermediateModel
MigrationModelMetadata
MigrationConnection
MigrationEntity
MigrationValueObject
MigrationField
MigrationIdentifier
MigrationRelationship
MigrationRepository
MigrationQuery
MigrationQueryAST
MigrationResultContract
MigrationTransaction
MigrationType
MigrationLifecycleHook
MigrationBehavior
MigrationSchemaModel
MigrationProcedure
MigrationRuntimeRequirement
MigrationEvidenceReference
MigrationConflict
MigrationAnnotation
```

---

## 172. Builder

Para construcción incremental podrá existir:

```text
MigrationIntermediateModelBuilder
```

---

## 173. Builder Rule

El Builder será mutable durante construcción.

El modelo final será:

```text
immutable snapshot
```

---

## 174. Merge Engine

Múltiples adapters podrán contribuir al mismo modelo mediante:

```text
MigrationModelMergeEngine
```

---

## 175. Merge Example

```text
Eloquent:
User → users

Legacy SQL:
SELECT FROM users

Schema:
users table
```

deberán converger hacia objetos relacionados, no duplicados.

---

## 176. Merge Conflict

Si dos adapters producen semánticas incompatibles:

```text
MigrationConflict
```

se conservará hasta resolución.

---

## 177. Conflict Resolution

Resoluciones permitidas:

```text
verified evidence
migration rule
explicit developer override
manual resolution
```

---

## 178. No Silent Override

La última fuente procesada nunca deberá sobrescribir silenciosamente una anterior.

---

## 179. Deterministic Merge

El resultado no dependerá del orden en que los adapters terminaron su análisis.

---

## 180. Canonical Ordering

Los objetos podrán ordenarse por:

```text
stable ID
```

para serialización y diff.

---

## 181. Migration Decision Model

Después del análisis podrá añadirse una capa:

```text
MigrationDecision
```

Ejemplos:

```text
PRESERVE
TRANSFORM
ADAPT
REPLACE
MANUAL
IGNORE_WITH_REASON
```

---

## 182. Decisions Are Not Facts

Las decisiones deberán permanecer separadas de las propiedades descubiertas.

---

## 183. Verification Requirements

Una decisión podrá declarar:

```text
SCHEMA_VERIFY
QUERY_COMPARE
TRANSACTION_TEST
BEHAVIOR_TEST
PERFORMANCE_TEST
MANUAL_REVIEW
```

---

## 184. Transformation Eligibility

Un objeto podrá ser:

```text
NOT_ANALYZED
ANALYZED
BLOCKED
ELIGIBLE
TRANSFORMED
VERIFIED
```

como estado de proceso.

---

## 185. Process State vs Semantic Model

El estado de proceso no deberá alterar la descripción semántica del objeto.

---

## 186. Example Complete Entity

```text
MigrationEntity
-------------------------------
id:
entity:user

name:
User

physical table:
users

identifier:
id

fields:
id
email
created_at
updated_at

relationships:
posts

behaviors:
automatic timestamps

repository:
UserRepository

sources:
Eloquent model
schema introspection

confidence:
HIGH
```

---

## 187. Example Query

```text
MigrationQuery
-------------------------------
id:
query:user.find_by_email

kind:
SELECT

source:
users

projection:
*

predicate:
email = :email

result:
ZERO_OR_ONE ROW

parameters:
email:string

source technologies:
Eloquent
Legacy PDO

platform-specific:
false
```

---

## 188. Example Transaction

```text
MigrationTransaction
-------------------------------
id:
transaction:checkout.complete

connection:
default

participants:
OrderRepository
InventoryDAO
PaymentRepository

atomicity:
REQUIRED

nested:
false

retry:
deadlock retry x3

confidence:
HIGH
```

---

## 189. Example Conflict

```text
MigrationConflict
-------------------------------
subject:
field:payment.amount

property:
precision/scale

Doctrine:
decimal(12,2)

Schema:
decimal(18,4)

Legacy PHP:
float conversion

status:
UNRESOLVED

impact:
DATA_INTEGRITY
```

---

## 190. Example Runtime Requirement

```text
MigrationRuntimeRequirement
-------------------------------
scope:
connection:default

requirement:
RESET_SESSION_STATE

reason:
legacy SQL uses session variables

persistent runtimes:
FrankenPHP
RoadRunner
OpenSwoole
```

---

## 191. Testing

El MIM deberá tener pruebas para:

```text
construction
validation
serialization
deserialization
stable IDs
deterministic ordering
merge
conflicts
diff
version upgrades
extension preservation
security redaction
```

---

## 192. Adapter Contract Tests

Cada adapter deberá demostrar que puede convertir sus source descriptors al MIM sin introducir clases externas dentro del core model.

---

## 193. Round-trip Serialization

```text
MIM
 ↓ serialize
JSON
 ↓ deserialize
MIM
```

deberá preservar semántica.

---

## 194. Golden Fixtures

Se mantendrán fixtures con:

```text
Eloquent
Doctrine
Legacy
Hybrid
```

que produzcan modelos intermedios esperados.

---

## 195. Merge Tests

Especial atención a:

```text
same table from multiple sources
conflicting field types
shared transaction
duplicate query
same relation inferred differently
```

---

## 196. Security Tests

Se verificará que snapshots no contengan:

```text
passwords
tokens
secret environment values
runtime PII
encryption keys
```

---

## 197. Performance

El modelo deberá soportar proyectos con:

```text
thousands of entities
tens of thousands of queries
large dependency graphs
```

sin requerir cargar artefactos innecesarios.

---

## 198. Lazy Sections

Partes voluminosas como:

```text
query AST
source snippets
runtime observations
```

podrán almacenarse por referencia cuando sea conveniente.

---

## 199. No Source Code Duplication

El snapshot no deberá copiar todo el source PHP.

Debe conservar referencias y únicamente la información necesaria para migración.

---

## 200. Architectural Boundary

La frontera final será:

```text
SOURCE WORLD
────────────────────────────
Eloquent
Doctrine
PDO
Legacy
Custom ORM
────────────────────────────
        Adapters
────────────────────────────
MIGRATION WORLD
────────────────────────────
Migration Intermediate Model
────────────────────────────
        Rules
────────────────────────────
VOLTSTACK WORLD
────────────────────────────
Quantum/Database
```

---

## 201. Decisiones arquitectónicas

### Decisión 1

Toda migración hacia VoltStack Database utilizará una representación intermedia neutral.

### Decisión 2

El MIM pertenecerá al paquete de migración, no al runtime normal de Database.

### Decisión 3

El core del MIM no dependerá de Eloquent, Doctrine ni otro ORM externo.

### Decisión 4

Los adapters convertirán conceptos source-specific a semántica neutral antes de cualquier transformación.

### Decisión 5

El modelo distinguirá hechos, inferencias, observaciones, overrides y decisiones.

### Decisión 6

Cada objeto mantendrá provenance y evidencia.

### Decisión 7

Los conflictos se representarán explícitamente y nunca se sobrescribirán silenciosamente.

### Decisión 8

El modelo será versionado, serializable, determinista e idealmente inmutable.

### Decisión 9

No todo acceso a datos deberá convertirse en entidad u ORM.

### Decisión 10

SQL nativo será una representación válida dentro del MIM.

### Decisión 11

Queries conservarán result shape, parameters, ordering, pagination y locking.

### Decisión 12

Transactions serán objetos de primera clase.

### Decisión 13

Lifecycle y side effects serán modelados explícitamente.

### Decisión 14

Schema físico e intención lógica permanecerán diferenciados.

### Decisión 15

Tipos custom conservarán su semántica hasta que exista una transformación segura.

### Decisión 16

El modelo podrá representar UNKNOWN sin convertirlo en NULL o valor por defecto.

### Decisión 17

El MIM será consciente de requisitos de persistent runtime sin acoplarse a FrankenPHP.

### Decisión 18

La versión estable de Database V1 congelará un contrato MIM 1.x.

---

## 202. Resultado esperado

El ecosistema de migración pasa de:

```text
Eloquent concepts
Doctrine concepts
PDO concepts
Legacy concepts
```

a:

```text
Connections
Entities
Records
Value Objects
Fields
Identifiers
Relationships
Queries
Repositories
Transactions
Types
Lifecycle
Behaviors
Schema
Procedures
Runtime Requirements
Evidence
Conflicts
```

y únicamente después:

```text
VoltStack Database
```

---

## 203. Principio final

```text
Do not translate APIs.

Translate semantics.
```

y:

```text
Source-specific knowledge ends at the adapter boundary.
```

---

## 204. Conclusión

`DATABASE_MIGRATION_INTERMEDIATE_MODEL` constituye una de las piezas fundamentales de la arquitectura de migración de VoltStack Database V1.

Su existencia permite que:

```text
Eloquent
Doctrine
PDO
legacy systems
future third-party adapters
```

compartan el mismo pipeline sin convertir a `Quantum/Database` en un conjunto de capas de compatibilidad.

La arquitectura definitiva será:

```text
Source
  ↓
Adapter
  ↓
Source Descriptor
  ↓
Normalizer
  ↓
Migration Intermediate Model
  ↓
Migration Rule Engine
  ↓
VoltStack Target Model
  ↓
Code Transformation
  ↓
Verification
```

La regla que deberá mantenerse durante toda la evolución del sistema es:

```text
The intermediate model describes persistence semantics.

It does not reproduce the source framework.

It does not dictate the target implementation.

It is the stable semantic bridge between both.
```

---

**Documento:** `330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
