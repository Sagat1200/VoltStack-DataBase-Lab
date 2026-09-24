# 333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md

## 1. Propósito

Este documento define la arquitectura oficial del **Database Migration Schema Compatibility System** de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseMigrationSchemaCompatibilitySystem
```

y es responsable de determinar si el **schema físico real**, el **schema declarado por el sistema origen** y el **schema esperado por VoltStack** son compatibles antes, durante y después de una migración.

Su objetivo no es únicamente comparar nombres de tablas y columnas.

Debe demostrar que la transición:

```text
Source Persistence Model
        ↓
Actual Database Schema
        ↓
VoltStack Target Model
```

preserva correctamente:

```text
structure
types
constraints
identifiers
relationships
defaults
generated values
indexes
platform semantics
data representation
```

---

## 2. Dependencias documentales

Este documento continúa directamente:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md
327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md
328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md
329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
331_DATABASE_MIGRATION_RULE_ENGINE.md
332_DATABASE_MIGRATION_CODE_TRANSFORMER.md
```

y alimentará:

```text
334_DATABASE_MIGRATION_BEHAVIOR_VERIFICATION_SYSTEM.md
335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
337_DATABASE_MIGRATION_TESTING_AND_VALIDATION_SYSTEM.md
338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md
339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 3. Principio fundamental

```text
ORM metadata is not the database.

Migration history is not the database.

The database is not necessarily the intended model.

Compatibility must compare all of them.
```

---

## 4. Problema

Un proyecto puede declarar:

```text
Doctrine:
amount DECIMAL(12,2)
```

mientras el schema real contiene:

```text
DECIMAL(18,4)
```

y una migración antigua puede indicar:

```text
DECIMAL(10,2)
```

Por tanto:

```text
Declared Metadata
!=
Migration History
!=
Actual Schema
```

en proyectos reales.

---

## 5. Triple Source of Truth

El sistema utilizará como mínimo:

```text
Source Persistence Metadata
Migration History
Actual Database Schema
```

y añadirá:

```text
VoltStack Target Schema Expectation
```

durante la migración.

---

## 6. Modelo conceptual

```text
Source Metadata ────────┐
                        │
Migration History ──────┼──► Schema Evidence Model
                        │
Actual Database ────────┘
                                │
                                ▼
                        Compatibility Engine
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
             Source Compatibility     Target Compatibility
                    │                       │
                    └───────────┬───────────┘
                                ▼
                      Schema Migration Plan
```

---

## 7. Responsabilidades

El sistema deberá:

```text
introspect actual schema
normalize schema metadata
compare source declarations
compare migration history
compare target expectations
detect drift
classify differences
evaluate compatibility
detect destructive changes
evaluate data conversion risks
produce required schema actions
attach validation requirements
provide evidence to Rule Engine
```

---

## 8. No responsabilidades

No deberá:

```text
silently alter production schema
assume ORM metadata is authoritative
automatically delete data
perform application behavior verification
perform final cutover
```

---

## 9. Inputs

Entradas principales:

```text
MigrationSchemaModel from MIM
Actual Schema Snapshot
Source Metadata Snapshot
Migration History Snapshot
Target VoltStack Schema Model
Database Platform Capabilities
Migration Policy
```

---

## 10. Outputs

Salidas:

```text
SchemaCompatibilityReport
SchemaConflictSet
SchemaMigrationRequirements
SchemaValidationRequirements
SchemaRiskAssessment
TargetSchemaFingerprint
```

---

# Parte I — Modelo de evidencia

## 11. Schema Evidence

Toda observación estructural se representará mediante:

```text
MigrationSchemaEvidence
```

---

## 12. Evidence Sources

```text
SOURCE_METADATA
MIGRATION_HISTORY
ACTUAL_DATABASE
TARGET_MODEL
MANUAL_OVERRIDE
RUNTIME_OBSERVATION
```

---

## 13. Evidence Provenance

Cada evidencia deberá conservar:

```text
source
adapter
connection
database
schema
object
file/location where applicable
snapshot ID
timestamp
confidence
```

---

## 14. Evidence Priority

No existirá una prioridad universal del tipo:

```text
actual DB always wins
```

porque la base física puede contener drift accidental.

La prioridad dependerá de la pregunta.

---

## 15. Example

Para saber:

```text
what exists now?
```

el schema físico tiene máxima relevancia.

Para saber:

```text
what application expects?
```

metadata y comportamiento observado pueden ser más relevantes.

---

## 16. Facts vs Intent

El sistema distinguirá:

```text
PHYSICAL_FACT
DECLARED_INTENT
HISTORICAL_INTENT
TARGET_INTENT
```

---

# Parte II — Schema Snapshot

## 17. Actual Schema Snapshot

El sistema producirá:

```text
MigrationDatabaseSchemaSnapshot
```

---

## 18. Snapshot Contents

```text
platform
platform version where available
database
schemas
tables
columns
primary keys
foreign keys
unique constraints
check constraints
indexes
sequences
views
materialized views
generated columns
triggers
procedures
functions
```

según capacidades del motor.

---

## 19. Snapshot Immutability

Un snapshot utilizado para planificación será inmutable.

---

## 20. Snapshot Fingerprint

Se calculará:

```text
schema_fingerprint
```

para detectar drift.

---

## 21. Canonicalization

Antes del fingerprint deberán normalizarse diferencias irrelevantes como:

```text
metadata ordering
platform-generated names when explicitly ignorable
case conventions where platform semantics allow
```

sin eliminar diferencias semánticas.

---

## 22. Schema Drift

Si:

```text
planned_schema_fingerprint
!=
current_schema_fingerprint
```

antes de una fase crítica:

```text
SCHEMA_DRIFT_DETECTED
```

---

## 23. Drift Policy

Dependiendo del cambio:

```text
INFO
WARNING
REPLAN_REQUIRED
BLOCK
```

---

# Parte III — Introspection

## 24. Schema Introspector

Componente:

```text
MigrationSchemaIntrospector
```

---

## 25. Platform-specific Introspectors

Podrán existir:

```text
MySqlMigrationSchemaIntrospector
MariaDbMigrationSchemaIntrospector
PostgreSqlMigrationSchemaIntrospector
SqliteMigrationSchemaIntrospector
SqlServerMigrationSchemaIntrospector
```

según plataformas oficialmente soportadas.

---

## 26. Capability-driven Introspection

No se asumirá que todas las plataformas pueden describir exactamente los mismos conceptos.

---

## 27. Capability Example

```text
partial indexes
expression indexes
deferrable constraints
materialized views
generated columns
sequences
```

varían por plataforma.

---

## 28. Unknown Metadata

Si una plataforma no puede introspeccionar una propiedad:

```text
UNKNOWN
```

y no:

```text
false
```

---

## 29. Read-only Introspection

La introspección será:

```text
read-only
```

por diseño.

---

## 30. Permissions

La herramienta deberá funcionar con permisos mínimos suficientes para inspección cuando la plataforma lo permita.

---

# Parte IV — Tablas

## 31. Table Compatibility

Para cada tabla se comparará:

```text
logical entity mapping
physical name
schema/catalog
columns
primary key
foreign keys
indexes
constraints
triggers
```

---

## 32. Missing Table

Clasificación:

```text
TARGET_REQUIRES_TABLE
ACTUAL_TABLE_MISSING
```

---

## 33. Extra Table

Una tabla no utilizada por el ORM no deberá considerarse automáticamente obsoleta.

Puede pertenecer a:

```text
legacy subsystem
reporting
audit
external integration
stored procedures
another application
```

---

## 34. Extra Table Policy

Por defecto:

```text
PRESERVE
```

hasta demostrar que puede eliminarse.

---

## 35. Table Rename

El sistema deberá distinguir:

```text
rename
```

de:

```text
drop + create
```

cuando exista evidencia suficiente.

---

## 36. Rename Confidence

Una inferencia de rename podrá utilizar:

```text
column similarity
migration history
explicit override
source metadata
```

pero una inferencia ambigua requerirá revisión.

---

# Parte V — Columnas

## 37. Column Compatibility

Para cada columna:

```text
name
native type
logical type
length
precision
scale
nullable
default
generated
collation
charset
unsigned where applicable
identity/sequence behavior
```

---

## 38. Column Difference Classes

```text
EXACT
COMPATIBLE
COMPATIBLE_WITH_CONVERSION
LOSSY
INCOMPATIBLE
UNKNOWN
```

---

## 39. Exact

Ejemplo:

```text
VARCHAR(255)
→ VARCHAR(255)
```

---

## 40. Compatible

Ejemplo conceptual:

```text
VARCHAR(100)
→ VARCHAR(255)
```

si no cambia semántica relevante.

---

## 41. Compatible with Conversion

Ejemplo:

```text
INT
→ BIGINT
```

normalmente puede ser seguro estructuralmente, sujeto a plataforma y aplicación.

---

## 42. Lossy

Ejemplo:

```text
DECIMAL(18,4)
→ DECIMAL(10,2)
```

puede perder rango o precisión.

---

## 43. Incompatible

Ejemplo:

```text
JSON document
→ integer
```

sin estrategia de transformación.

---

## 44. Unknown

Cuando no existe evidencia suficiente.

---

# Parte VI — Type Compatibility

## 45. Type Compatibility Matrix

Existirá:

```text
MigrationTypeCompatibilityMatrix
```

por plataforma.

---

## 46. Neutral Type Comparison

Primero:

```text
Native Source Type
     ↓
MIM Type
     ↓
Target Type
```

---

## 47. Native Type Preservation

El tipo nativo se conservará para detectar diferencias que el tipo lógico no capture.

---

## 48. String Types

Se evaluará:

```text
length
charset
collation
fixed/variable
binary/non-binary
```

---

## 49. Integer Types

Se evaluará:

```text
width/range
signedness
identity behavior
```

según plataforma.

---

## 50. Decimal

Se evaluará:

```text
precision
scale
range
rounding implications
PHP representation
```

---

## 51. Decimal Safety Rule

```text
DECIMAL
→ FLOAT
```

deberá considerarse de alto riesgo cuando importe precisión financiera.

---

## 52. Floating Point

Se distinguirá:

```text
FLOAT
DOUBLE
REAL
```

según semántica de plataforma.

---

## 53. Boolean

Se evaluarán representaciones como:

```text
BOOLEAN
TINYINT(1)
BIT
CHAR(1)
INTEGER flags
```

sin asumir equivalencia automática.

---

## 54. Date/Time

Se comparará:

```text
DATE
TIME
DATETIME
TIMESTAMP
timezone-aware types
precision
```

---

## 55. Timezone Semantics

Dos tipos físicamente similares pueden ser incompatibles si la aplicación interpreta timezone de forma distinta.

---

## 56. JSON

Se distinguirá:

```text
JSON
JSONB
TEXT containing JSON
BLOB containing JSON
```

---

## 57. JSON Conversion

`TEXT → JSON` requerirá validar que los valores existentes sean JSON válido.

---

## 58. Enum

Se distinguirá:

```text
database-native enum
check-constrained string
application enum
integer enum
```

---

## 59. Enum Evolution

Cambiar valores de enum puede afectar datos existentes y deberá analizarse explícitamente.

---

## 60. UUID

Se distinguirán representaciones:

```text
CHAR(36)
VARCHAR
BINARY(16)
native UUID
```

---

## 61. UUID Conversion

Cambiar representación física requiere estrategia de datos, no solo schema.

---

## 62. Binary

Se evaluará:

```text
length
encoding assumptions
binary/blob semantics
```

---

## 63. Custom Types

No deberán reducirse automáticamente a su storage type si contienen semántica adicional.

---

# Parte VII — Nullability y Defaults

## 64. Nullability

Cambiar:

```text
NULLABLE
→ NOT NULL
```

requiere verificar datos existentes.

---

## 65. Null Data Check

Antes de considerar segura la transición deberá poder comprobarse:

```sql
COUNT(*) WHERE column IS NULL
```

en entorno autorizado.

---

## 66. Default Values

Se distinguirá:

```text
literal default
database expression
application default
trigger-generated value
no default
unknown
```

---

## 67. Database Expression Defaults

Ejemplos:

```text
CURRENT_TIMESTAMP
uuid_generate...
nextval...
```

no deberán tratarse como strings literales.

---

## 68. Default Removal

Eliminar un default puede ser compatible con datos existentes pero incompatible con nuevos inserts.

Por tanto también requiere verificación de comportamiento.

---

# Parte VIII — Identificadores

## 69. Primary Key Compatibility

Se comparará:

```text
columns
order
type
generation
physical constraint
```

---

## 70. Composite Keys

No deberán transformarse a single surrogate key sin decisión explícita.

---

## 71. Auto Increment

Se deberá verificar:

```text
identity/autoincrement mechanism
current sequence/counter
insert behavior
```

---

## 72. Sequence

Para motores con sequences:

```text
sequence name
allocation behavior
current value where relevant
ownership
```

---

## 73. Identifier Conversion

Cambiar:

```text
integer ID
→ UUID
```

no será considerado una simple migración de ORM.

Es una transformación de datos/schema separada y de alto riesgo.

---

# Parte IX — Foreign Keys

## 74. Foreign Key Compatibility

Se comparará:

```text
local columns
target table
target columns
on delete
on update
deferrable
constraint existence
```

---

## 75. Logical vs Physical FK

Una relación ORM puede existir sin foreign key física.

El sistema deberá representar:

```text
LOGICAL_RELATIONSHIP_ONLY
```

---

## 76. Adding FK

Agregar una FK requerirá validar datos existentes.

---

## 77. Orphan Detection

Antes de agregar FK:

```text
orphan rows
```

deberán detectarse.

---

## 78. Cascade Difference

```text
ON DELETE CASCADE
```

vs:

```text
application cascade
```

son mecanismos distintos.

No deberán declararse equivalentes sin análisis de comportamiento.

---

# Parte X — Unique y Check Constraints

## 79. Unique Constraints

Se comparará:

```text
columns
expressions
partial predicates
null semantics
```

según plataforma.

---

## 80. Unique Data Validation

Antes de agregar unique:

```text
duplicate values
```

deberán comprobarse.

---

## 81. Check Constraints

Se conservará la expresión cuando sea posible.

---

## 82. Check Semantic Compatibility

Dos expresiones textualmente distintas pueden ser equivalentes; V1 podrá clasificarlas como:

```text
UNKNOWN/MANUAL
```

si no existe normalización segura.

---

# Parte XI — Indexes

## 83. Index Compatibility

Se comparará:

```text
columns
column order
sort order
unique
partial predicate
expressions
included columns
platform options
```

---

## 84. Index Name

El nombre puede ser irrelevante semánticamente en algunos contextos, pero deberá preservarse si:

```text
migrations
operational tooling
DBAs
external systems
```

dependen de él.

---

## 85. Missing Index

Puede ser:

```text
PERFORMANCE_DIFFERENCE
```

sin ser necesariamente un conflicto funcional.

---

## 86. Index Risk

El sistema distinguirá:

```text
correctness requirement
performance requirement
operational requirement
```

---

# Parte XII — Generated Columns y Triggers

## 87. Generated Columns

Se comparará:

```text
expression
stored/virtual
type
dependencies
```

---

## 88. Trigger Discovery

Triggers deberán formar parte del snapshot físico.

---

## 89. Trigger Conflict

Ejemplo:

```text
DB trigger updates updated_at
+
VoltStack automatic timestamps
```

Resultado:

```text
DUPLICATE_BEHAVIOR_RISK
```

---

## 90. Trigger Preservation

Por defecto, un trigger existente no deberá eliminarse porque el ORM origen no lo declare.

---

# Parte XIII — Views

## 91. Views

El sistema deberá descubrir:

```text
views
materialized views where supported
```

---

## 92. View Dependencies

Una migración de tabla deberá considerar views dependientes.

---

## 93. View-backed Read Models

Un read model podrá mapear directamente una view y preservarse.

---

# Parte XIV — Procedures y Functions

## 94. Stored Procedures

El schema compatibility deberá verificar:

```text
existence
signature
parameter types
result expectations where known
```

---

## 95. Functions

Funciones utilizadas por:

```text
defaults
indexes
queries
generated columns
triggers
```

formarán parte del dependency graph.

---

## 96. Opaque Procedure Body

V1 no necesitará demostrar equivalencia semántica completa de cada procedimiento para preservarlo.

Podrá marcar:

```text
PRESERVED_EXTERNAL_DATABASE_BEHAVIOR
```

---

# Parte XV — Schema Dependency Graph

## 97. Dependency Graph

Se construirá:

```text
MigrationSchemaDependencyGraph
```

---

## 98. Nodes

```text
table
column
index
constraint
sequence
view
trigger
procedure
function
```

---

## 99. Edges

Ejemplos:

```text
FK → target table
view → table
generated column → source columns
trigger → table/function
default → sequence/function
index expression → function
```

---

## 100. Purpose

El grafo permitirá:

```text
safe operation ordering
impact analysis
rollback planning
destructive change detection
```

---

# Parte XVI — Compatibility Classification

## 101. Object Compatibility

Cada objeto tendrá:

```text
EXACT
COMPATIBLE
COMPATIBLE_WITH_ACTION
INCOMPATIBLE
BLOCKED
UNKNOWN
```

---

## 102. EXACT

No requiere cambio.

---

## 103. COMPATIBLE

Diferencia no afecta el contrato target.

---

## 104. COMPATIBLE_WITH_ACTION

Requiere una acción conocida y verificable.

---

## 105. INCOMPATIBLE

Existe diferencia semántica significativa.

---

## 106. BLOCKED

No puede resolverse mientras exista una dependencia/conflicto.

---

## 107. UNKNOWN

No existe suficiente evidencia.

---

# Parte XVII — Difference Model

## 108. Schema Difference

```text
MigrationSchemaDifference
```

contendrá:

```text
object ID
property
source value
actual value
target value
classification
risk
evidence
recommended action
validation requirements
```

---

## 109. Difference Categories

```text
MISSING
EXTRA
TYPE
NULLABILITY
DEFAULT
IDENTIFIER
CONSTRAINT
INDEX
RELATIONSHIP
GENERATION
PLATFORM
BEHAVIOR
NAME
```

---

## 110. Difference Severity

```text
INFO
LOW
MEDIUM
HIGH
CRITICAL
```

---

# Parte XVIII — Destructive Change Detection

## 111. Destructive Changes

Se considerarán potencialmente destructivos:

```text
DROP TABLE
DROP COLUMN
narrow type
reduce precision
reduce length
change encoding
remove enum value
change identifier strategy
remove constraint required by behavior
```

---

## 112. Destructive Default

Regla:

```text
Destructive schema changes are never SAFE by default.
```

---

## 113. Data Evidence

Para reducir riesgo se requerirán verificaciones sobre los datos afectados.

---

## 114. Example

Cambiar:

```text
VARCHAR(255)
→ VARCHAR(100)
```

solo podrá considerarse potencialmente viable si:

```text
MAX(character_length(column)) <= 100
```

y se verifican semánticas adicionales.

---

# Parte XIX — Data Compatibility Probes

## 115. Data Probe

El sistema podrá generar:

```text
MigrationDataCompatibilityProbe
```

---

## 116. Probe Examples

```text
null count
duplicate count
orphan count
max length
numeric range
decimal precision
invalid JSON count
invalid enum values
UUID parseability
```

---

## 117. Read-only

Los probes deberán ser read-only.

---

## 118. Production Safety

En producción podrán requerir:

```text
explicit enable
timeout
row/sample policy
replica use
query cost guard
```

---

## 119. Full Scan

No deberá ejecutarse automáticamente una consulta costosa sobre una tabla enorme sin policy explícita.

---

## 120. Sampling

Sampling puede aportar evidencia, pero no deberá presentarse como prueba completa de integridad.

---

# Parte XX — Schema Action Plan

## 121. Schema Migration Requirement

Cuando sea necesario un cambio se generará:

```text
MigrationSchemaRequirement
```

---

## 122. Requirement Actions

```text
CREATE
ALTER
RENAME
ADD_CONSTRAINT
DROP_CONSTRAINT
ADD_INDEX
DROP_INDEX
PRESERVE
MANUAL
BLOCK
```

---

## 123. Requirement vs SQL

El compatibility system define:

```text
what schema action is required
```

no necesariamente el SQL final.

---

## 124. SQL Compilation

El SQL deberá pasar por la arquitectura normal de:

```text
Schema AST
Platform SQL Compiler
Migration System
```

de `Quantum/Database`.

---

## 125. Code Transformer Integration

El documento 332 podrá generar el archivo de migration correspondiente a los requirements.

---

## 126. No Direct Production Mutation

El Schema Compatibility System no ejecutará directamente:

```text
ALTER TABLE
DROP TABLE
DROP COLUMN
```

en producción.

---

# Parte XXI — Operation Ordering

## 127. Ordering

El dependency graph determinará orden seguro.

Ejemplo:

```text
create table
→ create columns
→ copy/transform data
→ create constraints
→ create indexes
```

---

## 128. Constraint Ordering

Agregar una FK antes de corregir orphans puede fallar.

El plan deberá reflejarlo.

---

## 129. Expand/Contract

Cambios complejos podrán utilizar:

```text
EXPAND
MIGRATE
VERIFY
CONTRACT
```

---

## 130. Example

Renombrar una columna en zero/low downtime:

```text
add new column
dual compatibility
backfill
switch reads
switch writes
verify
remove old column
```

cuando el plan lo requiera.

---

## 131. V1 Boundary

El sistema V1 podrá **planear** estos pasos sin pretender ser una plataforma distribuida avanzada de online schema migration.

---

# Parte XXII — Schema Compatibility Policies

## 132. Policy

```text
MigrationSchemaCompatibilityPolicy
```

---

## 133. Example Policies

```text
strict_schema_match
preserve_extra_objects
allow_safe_widening
require_data_probe_for_narrowing
block_destructive_changes
preserve_physical_names
require_fk_validation
```

---

## 134. Default Conservative Policy

La policy predeterminada deberá priorizar:

```text
Data Integrity
```

sobre automatización.

---

# Parte XXIII — Overrides

## 135. Schema Override

El usuario podrá declarar:

```text
MigrationSchemaOverride
```

---

## 136. Override Example

```text
Actual column:
legacy_code VARCHAR(64)

Decision:
preserve unchanged

Reason:
external ERP integration
```

---

## 137. Override Audit

El override deberá aparecer en:

```text
MIM
plan
report
validation requirements
```

---

## 138. No Hidden Override

Nunca deberá existir una configuración que silenciosamente ignore todos los conflictos.

---

# Parte XXIV — Platform Semantics

## 139. Platform Awareness

El sistema deberá comprender diferencias reales entre plataformas.

---

## 140. MySQL/MariaDB

Deberá considerar, según versión/capacidad:

```text
charset
collation
unsigned
auto increment
generated columns
enum
JSON representation
index length
```

---

## 141. PostgreSQL

Deberá considerar:

```text
schemas
sequences/identity
native UUID
JSON/JSONB
arrays
partial indexes
expression indexes
deferrable constraints
generated columns
```

según capacidades.

---

## 142. SQLite

Deberá considerar su modelo particular de:

```text
type affinity
ALTER TABLE capabilities
foreign key configuration
```

---

## 143. SQL Server

Deberá considerar:

```text
identity
schemas
computed columns
filtered indexes
rowversion
```

según capacidades.

---

## 144. No Lowest-common-denominator

La compatibilidad no deberá destruir features válidas únicamente porque otra plataforma no las soporte.

---

# Parte XXV — Charset y Collation

## 145. Charset

Cambios de charset pueden afectar:

```text
storage
comparison
encoding
index sizes
data validity
```

---

## 146. Collation

Cambios pueden alterar:

```text
case sensitivity
sorting
uniqueness
comparison
```

---

## 147. Unique Risk

Un cambio de collation puede convertir dos valores antes distintos en equivalentes.

Deberá verificarse antes de crear/recrear unique constraints.

---

# Parte XXVI — Data Representation

## 148. Representation Compatibility

Dos schemas pueden parecer compatibles pero almacenar semántica distinta.

Ejemplo:

```text
status = 1
```

puede significar:

```text
active
```

en un sistema y otra cosa en otro.

---

## 149. Domain Encoding

El MIM deberá conservar mappings conocidos:

```text
enum values
boolean encodings
type discriminators
soft-delete markers
tenant keys
```

---

## 150. Discriminator Safety

Cambiar un morph/discriminator map puede romper filas históricas.

Por defecto deberá preservarse.

---

# Parte XXVII — Authentication and Security-sensitive Schema

## 151. Sensitive Tables

Podrán marcarse:

```text
AUTHENTICATION_CRITICAL
AUTHORIZATION_CRITICAL
SECURITY_CRITICAL
```

---

## 152. Examples

```text
password hashes
MFA secrets
sessions
remember tokens
API credentials
roles
permissions
```

---

## 153. Conservative Transformations

Sobre estos datos se requerirá mayor evidencia y validación.

---

## 154. Hash Columns

No se deberá cambiar longitud/tipo de hashes sin verificar formatos existentes y futuros.

---

## 155. Encryption Columns

No deberá asumirse que:

```text
VARCHAR → TEXT
```

es la única preocupación; encoding y envelope format también importan.

---

# Parte XXVIII — Multitenancy

## 156. Optional Multitenancy

El sistema deberá soportar análisis de:

```text
shared table
schema per tenant
database per tenant
dynamic tenant connections
```

sin depender obligatoriamente del paquete Multitenancy.

---

## 157. Tenant Schema Consistency

En estrategias con múltiples schemas/databases podrá verificarse:

```text
schema consistency across tenants
```

si el paquete correspondiente lo solicita.

---

## 158. Tenant Drift

Un tenant con schema distinto deberá reportarse explícitamente.

---

# Parte XXIX — Persistent Runtime

## 159. Schema and Runtime

El schema compatibility no es directamente dependiente de FrankenPHP, pero ciertas diferencias pueden afectar runtime persistente.

Ejemplo:

```text
session variables
temporary tables
connection-specific schema selection
```

---

## 160. Runtime Requirements

Estos casos deberán producir:

```text
MigrationRuntimeRequirement
```

para verificación posterior.

---

# Parte XXX — Dual ORM

## 161. Dual ORM Compatibility

Antes de ejecutar:

```text
Eloquent + VoltStack
```

o:

```text
Doctrine + VoltStack
```

sobre las mismas tablas, ambos modelos deberán ser compatibles con el schema físico.

---

## 162. Shared Schema Contract

El schema físico se convierte temporalmente en:

```text
shared persistence contract
```

---

## 163. No Independent Schema Management

Durante dual runtime deberá evitarse que dos sistemas modifiquen el schema independientemente.

---

# Parte XXXI — Shadow Queries

## 164. Shadow Query Dependency

El documento 336 utilizará el schema compatibility result para interpretar diferencias.

---

## 165. Example

Si una columna target fue convertida:

```text
TINYINT → BOOLEAN
```

el result comparator deberá conocer esa normalización.

---

# Parte XXXII — Validation

## 166. Schema Validation Phases

```text
PRE_MIGRATION
POST_TRANSFORMATION
POST_SCHEMA_MIGRATION
PRE_CUTOVER
POST_CUTOVER
```

---

## 167. Pre-migration

Establece baseline.

---

## 168. Post-transformation

Verifica que el código generado espera el schema correcto.

---

## 169. Post-schema-migration

Introspecciona nuevamente el DB.

---

## 170. Pre-cutover

Confirma ausencia de drift crítico.

---

## 171. Post-cutover

Confirma que el schema permanece consistente.

---

# Parte XXXIII — Verification Levels

## 172. Verification Status

```text
NOT_CHECKED
STRUCTURALLY_VERIFIED
DATA_COMPATIBILITY_VERIFIED
FULLY_VERIFIED
FAILED
UNKNOWN
```

---

## 173. Structural Verification

No implica que los datos sean convertibles.

---

## 174. Data Compatibility Verification

No implica comportamiento completo de aplicación.

---

## 175. Behavioral Verification

Pertenece principalmente al documento 334.

---

# Parte XXXIV — Diagnostics

## 176. Diagnostic

Cada incompatibilidad deberá explicar:

```text
what differs
where
why it matters
evidence
risk
possible action
required validation
```

---

## 177. Example Diagnostic

```text
VSDB-MIG-SCHEMA-0042

Table:
payments

Column:
amount

Declared:
DECIMAL(12,2)

Actual:
DECIMAL(18,4)

Target:
DECIMAL(12,2)

Risk:
HIGH / DATA_INTEGRITY

Reason:
Target precision is narrower than existing schema.

Required:
Resolve intended precision before transformation.
```

---

## 178. Stable Diagnostic IDs

Se utilizarán IDs:

```text
VSDB-MIG-SCHEMA-xxxx
```

---

# Parte XXXV — CLI

## 179. Analyze Schema

Conceptualmente:

```bash
php volt database:migrate:schema --analyze
```

---

## 180. Compare

```bash
php volt database:migrate:schema --compare
```

---

## 181. Entity Scope

```bash
php volt database:migrate:schema \
    --entity=Payment
```

---

## 182. Connection Scope

```bash
php volt database:migrate:schema \
    --connection=legacy
```

---

## 183. Data Probes

```bash
php volt database:migrate:schema \
    --probe
```

deberá mostrar previamente qué queries pretende ejecutar cuando puedan ser costosas.

---

## 184. Export

Podrá exportar:

```text
schema snapshot
compatibility report
schema requirements
```

en formato machine-readable.

---

# Parte XXXVI — Reporting

## 185. Summary

Ejemplo:

```text
Tables                 84
Exact                  70
Compatible              6
Action required          4
Incompatible             2
Unknown                  2

Destructive risks        1
Data probes required     5
Manual reviews           2
```

---

## 186. No False Green

Si existen objetos:

```text
UNKNOWN
```

el reporte no deberá afirmar compatibilidad completa.

---

# Parte XXXVII — Integration with Rule Engine

## 187. Rule Feedback

El resultado podrá:

```text
confirm rule
block rule
raise risk
require manual review
request data probe
```

---

## 188. Replanning

Si 333 descubre una incompatibilidad que invalida una decisión de 331:

```text
plan must be regenerated/reapproved
```

---

## 189. No Transformer Override

El Code Transformer no deberá ignorar un bloqueo de schema.

---

# Parte XXXVIII — Integration with Code Transformer

## 190. Schema Requirements

El Transformer recibirá:

```text
MigrationSchemaRequirement
```

y podrá generar migrations.

---

## 191. Migration Generation

Debe utilizar el sistema normal de schema/migrations de VoltStack.

---

## 192. Generated Migration Metadata

Podrá incluir:

```text
origin plan ID
schema requirement ID
rule ID
rollback classification
```

---

# Parte XXXIX — Integration with Behavior Verification

## 193. Behavior Requirements

Diferencias como:

```text
default changed
cascade changed
trigger changed
generated value changed
```

deberán producir verificaciones para el documento 334.

---

# Parte XL — Integration with Testing

## 194. Generated Tests

El documento 337 podrá derivar:

```text
schema contract tests
data compatibility tests
constraint tests
type round-trip tests
```

---

# Parte XLI — Integration with Rollback

## 195. Rollback Classification

Cada schema requirement deberá indicar:

```text
REVERSIBLE
REVERSIBLE_WITH_DATA_BACKUP
CONDITIONALLY_REVERSIBLE
IRREVERSIBLE
UNKNOWN
```

---

## 196. Example

```text
ADD INDEX
```

normalmente es reversible estructuralmente.

```text
DROP COLUMN
```

puede ser irreversible sin backup de datos.

---

# Parte XLII — Performance and Operational Safety

## 197. Operational Risk

Una migración puede ser semánticamente correcta pero operacionalmente peligrosa.

---

## 198. Examples

```text
table rewrite
long metadata lock
index build
large backfill
constraint validation
```

---

## 199. Operational Metadata

El system podrá adjuntar:

```text
estimated impact
requires maintenance window
online capability
locking risk
```

cuando pueda determinarlo.

---

## 200. No Unfounded Estimates

Si no existe evidencia suficiente:

```text
UNKNOWN
```

en lugar de inventar duración.

---

## 201. Large Table Awareness

Tablas grandes deberán elevar precaución para:

```text
ALTER
index creation
backfill
validation scans
```

---

# Parte XLIII — Observability

## 202. Metrics

Métricas posibles:

```text
database.migration.schema.objects
database.migration.schema.conflicts
database.migration.schema.drift
database.migration.schema.destructive
database.migration.schema.probes
database.migration.schema.unknown
database.migration.schema.duration
```

---

## 203. Logging

Canal:

```text
database.migration.schema
```

---

## 204. Sensitive Data

Los resultados de probes no deberán incluir PII innecesaria.

Preferir:

```text
counts
ranges
hashes
aggregates
```

---

# Parte XLIV — Caching

## 205. Snapshot Cache

La introspección podrá cachearse por:

```text
connection fingerprint
schema fingerprint
platform
```

---

## 206. Critical Phase Refresh

Antes de cutover no deberá depender de un snapshot obsoleto.

---

# Parte XLV — Error Model

## 207. Exceptions

Conceptualmente:

```text
MigrationSchemaException
MigrationSchemaIntrospectionException
MigrationSchemaConflictException
MigrationSchemaDriftException
MigrationSchemaProbeException
MigrationSchemaUnsupportedFeatureException
MigrationSchemaDestructiveChangeException
```

---

## 208. Partial Introspection

Si solo parte del schema pudo inspeccionarse:

```text
PARTIAL_SCHEMA_SNAPSHOT
```

y el sistema deberá indicar exactamente qué falta.

---

# Parte XLVI — Testing Architecture

## 209. Unit Tests

Para:

```text
type compatibility
difference classification
risk classification
normalization
dependency graph
```

---

## 210. Platform Integration Tests

Contra databases reales soportadas.

---

## 211. Schema Fixture Tests

Fixtures deberán incluir:

```text
simple schema
composite keys
custom types
foreign keys
indexes
generated columns
triggers
views
procedures
drift
```

---

## 212. Golden Snapshot Tests

Un schema conocido deberá producir snapshot determinista.

---

## 213. Drift Tests

```text
baseline
→ external schema modification
→ re-introspection
→ drift detected
```

---

## 214. Data Probe Tests

Se verificarán:

```text
nulls
duplicates
orphans
length overflow
invalid JSON
enum mismatch
```

---

## 215. Destructive Change Tests

Cada clase destructiva deberá estar cubierta.

---

## 216. Cross-platform Semantic Tests

Una abstracción neutral deberá producir clasificación coherente aunque el tipo nativo difiera.

---

## 217. Security Tests

Se comprobará:

```text
credential redaction
PII-safe diagnostics
read-only probes
query timeout policy
```

---

## 218. Performance Tests

Schemas con miles de objetos deberán poder compararse eficientemente.

---

# Parte XLVII — Componentes principales

## 219. Component Architecture

```text
DatabaseMigrationSchemaCompatibilitySystem
│
├── MigrationSchemaSnapshotManager
├── MigrationSchemaIntrospectorRegistry
├── MigrationSchemaNormalizer
├── MigrationSchemaEvidenceCollector
├── MigrationSchemaEvidenceMerger
├── MigrationSchemaComparator
├── MigrationTypeCompatibilityMatrix
├── MigrationSchemaDifferenceClassifier
├── MigrationSchemaConflictDetector
├── MigrationSchemaDependencyGraph
├── MigrationDestructiveChangeDetector
├── MigrationDataProbePlanner
├── MigrationDataProbeExecutor
├── MigrationSchemaRiskEvaluator
├── MigrationSchemaRequirementBuilder
├── MigrationSchemaPolicyResolver
├── MigrationSchemaDriftDetector
├── MigrationSchemaValidator
└── MigrationSchemaReporter
```

---

# Parte XLVIII — Pipeline completo

## 220. Pipeline

```text
Source Metadata
      │
Migration History
      │
Actual Database
      │
Target Model
      │
      ▼
Collect Evidence
      │
      ▼
Normalize Schema
      │
      ▼
Build Snapshots
      │
      ▼
Compare
      │
      ▼
Detect Conflicts
      │
      ▼
Classify Compatibility
      │
      ▼
Detect Destructive Changes
      │
      ▼
Plan Data Probes
      │
      ▼
Evaluate Risk
      │
      ▼
Build Schema Requirements
      │
      ▼
Feed Rule Engine / Transformer
      │
      ▼
Re-introspect and Verify
```

---

# Parte XLIX — Ejemplo integral

## 221. Source

Doctrine declara:

```text
payments.amount
DECIMAL(12,2)
NOT NULL
```

---

## 222. Migration History

Una migración antigua creó:

```text
DECIMAL(10,2)
```

---

## 223. Actual Database

La base contiene:

```text
DECIMAL(18,4)
```

---

## 224. Target

La transformación propuesta espera:

```text
DECIMAL(12,2)
```

---

## 225. Compatibility Result

```text
Status:
INCOMPATIBLE

Risk:
HIGH

Category:
DATA_INTEGRITY

Conflict:
precision/scale mismatch

Required probe:
existing value range
existing decimal scale

Automatic action:
BLOCKED
```

---

## 226. Correct Behavior

VoltStack no deberá generar automáticamente:

```sql
ALTER COLUMN amount DECIMAL(12,2)
```

hasta resolver el conflicto.

---

# Parte L — Ejemplo FK

## 227. Source Metadata

```text
Order belongsTo User
```

---

## 228. Actual Schema

```text
orders.user_id
```

pero sin foreign key.

---

## 229. Target

VoltStack metadata espera la relación, pero la policy no exige agregar FK automáticamente.

Resultado posible:

```text
Logical relationship:
COMPATIBLE

Physical FK:
MISSING

Action:
PRESERVE / OPTIONAL ACTION
```

---

## 230. Important Distinction

La ausencia de FK física no deberá confundirse con ausencia de relación lógica.

---

# Parte LI — Ejemplo Nullability

## 231. Actual

```text
users.email NULL
```

Target:

```text
NOT NULL
```

---

## 232. Required Probe

```text
count rows where email IS NULL
```

---

## 233. Result

Si existen nulls:

```text
BLOCKED
```

hasta definir estrategia.

Si no existen:

```text
COMPATIBLE_WITH_ACTION
```

sujeto a behavior verification.

---

# Parte LII — Ejemplo JSON

## 234. Actual

```text
metadata TEXT
```

Target:

```text
JSON
```

---

## 235. Probe

Validar todos los valores relevantes como JSON antes de la conversión.

---

## 236. Invalid Values

Si existen:

```text
MANUAL DATA REMEDIATION REQUIRED
```

---

# Parte LIII — Ejemplo Index

## 237. Source/Target

La query crítica filtra:

```text
tenant_id + created_at
```

pero no existe índice.

---

## 238. Classification

No necesariamente:

```text
SCHEMA INCOMPATIBLE
```

pero sí:

```text
PERFORMANCE REQUIREMENT
```

que deberá pasar a pruebas de performance.

---

# Parte LIV — Arquitectura de seguridad

## 239. Safety Principle

```text
Schema automation must never outrun evidence.
```

---

## 240. Production Defaults

En producción:

```text
introspection = allowed when configured
data probes = conservative
schema mutation = disabled here
destructive operations = blocked
```

---

## 241. Credentials

El sistema recibirá conexiones mediante infraestructura segura de VoltStack.

No almacenará credenciales en snapshots.

---

# Parte LV — Decisiones arquitectónicas

## 242. Decisión 1

La compatibilidad se evaluará contra metadata, historial, schema físico y target.

## 243. Decisión 2

Ninguna de esas fuentes será autoridad universal para todas las preguntas.

## 244. Decisión 3

El schema físico se introspeccionará mediante adapters por plataforma.

## 245. Decisión 4

Los snapshots serán versionados, deterministas e inmutables.

## 246. Decisión 5

Los conflictos permanecerán explícitos hasta resolución.

## 247. Decisión 6

`UNKNOWN` será un estado válido y diferente de ausencia.

## 248. Decisión 7

Los cambios destructivos nunca serán `SAFE` por defecto.

## 249. Decisión 8

Los cambios que dependan de contenido existente podrán requerir data probes.

## 250. Decisión 9

Los probes serán read-only y operacionalmente controlados.

## 251. Decisión 10

SQL/schema mutation se ejecutará mediante el sistema normal de migrations de VoltStack, no directamente desde este componente.

## 252. Decisión 11

Los objetos extra del database se preservarán por defecto hasta demostrar que pueden eliminarse.

## 253. Decisión 12

Las diferencias de performance se distinguirán de incompatibilidades funcionales.

## 254. Decisión 13

Triggers, views, procedures, generated columns y database behavior serán parte del análisis.

## 255. Decisión 14

Los nombres físicos se preservarán durante migración salvo decisión explícita.

## 256. Decisión 15

Las diferencias platform-specific no se reducirán artificialmente al mínimo común denominador.

## 257. Decisión 16

Los datos sensibles de autenticación, autorización y cifrado recibirán políticas conservadoras.

## 258. Decisión 17

El sistema detectará schema drift antes de fases críticas.

## 259. Decisión 18

Una verificación estructural no equivale a verificación de comportamiento.

---

# Parte LVI — Criterios de finalización

## 260. Schema Compatibility Completion

Una unidad podrá considerarse schema-compatible cuando:

```text
all required objects inspected
no unresolved critical conflict
all required destructive changes explicitly approved
required data probes passed
target schema expectation resolved
physical names mapped
identifiers verified
relationships verified
critical constraints verified
schema fingerprint recorded
```

---

## 261. Full Migration Completion

Aun después de lo anterior será necesario:

```text
Behavior Verification
Query Comparison
Testing
Cutover Validation
```

---

# Parte LVII — Resultado esperado

## 262. Antes

```text
Source ORM says one thing
Migration files say another
Database may contain something else
Target assumes a fourth model
```

---

## 263. Después

```text
Schema Compatibility Model
│
├── Source Intent
├── Historical Intent
├── Physical Reality
├── Target Intent
├── Differences
├── Conflicts
├── Risk
├── Data Probes
├── Required Actions
└── Verification State
```

---

# Parte LVIII — Principio final

## 264. Regla

```text
Never migrate the schema you assume exists.

Migrate the schema you have actually verified.
```

Y:

```text
Structural similarity is not sufficient.

Data representation and database behavior are part of compatibility.
```

---

# Parte LIX — Conclusión

## 265. Arquitectura final

`DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM` establece la frontera de seguridad entre la representación lógica de persistencia y la realidad física del database.

La arquitectura de migración queda:

```text
Source Adapters
      │
      ▼
Analysis Engine
      │
      ▼
Migration Intermediate Model
      │
      ▼
Migration Rule Engine
      │
      ▼
Code Transformer
      │
      ├──────────────┐
      ▼              ▼
Target Model    Schema Compatibility
                     │
                     ▼
             Schema Requirements
                     │
                     ▼
             VoltStack Migrations
                     │
                     ▼
              Re-introspection
                     │
                     ▼
              Schema Verification
```

Con este sistema, VoltStack evita uno de los errores más peligrosos en migraciones de ORM: asumir que el modelo declarado y la base real son idénticos.

La regla arquitectónica definitiva será:

```text
Inspect reality.

Preserve evidence.

Expose conflicts.

Protect data.

Generate only verified schema changes.
```

---

**Documento:** `333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
