# 332_DATABASE_MIGRATION_CODE_TRANSFORMER.md

## 1. Propósito

Este documento define la arquitectura oficial del **Database Migration Code Transformer** de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseMigrationCodeTransformer
```

y es responsable de ejecutar sobre el proyecto las decisiones previamente producidas por:

```text
DatabaseMigrationRuleEngine
```

El Transformer no decide **qué** debe migrarse.

Su responsabilidad es determinar **cómo aplicar técnicamente** una decisión aprobada sobre:

```text
PHP source code
configuration
metadata
dependency declarations
migration artifacts
tests
supporting compatibility code
```

manteniendo seguridad, trazabilidad, idempotencia y capacidad de rollback.

---

## 2. Dependencias documentales

Este documento continúa:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md
327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md
328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md
329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
331_DATABASE_MIGRATION_RULE_ENGINE.md
```

y alimentará:

```text
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

## 3. Principio fundamental

```text
Rule Engine decides WHAT.

Code Transformer decides HOW.

Validator proves WHETHER IT WORKED.
```

El Transformer nunca deberá inventar una decisión de migración que no exista en el plan aprobado.

---

## 4. Problema

Una decisión como:

```text
REPLACE global PDO singleton
WITH VoltStack ConnectionManager
```

todavía requiere resolver:

```text
which files change?
which imports change?
which constructor changes?
which dependency injection bindings change?
which call sites change?
which tests change?
which compatibility layer is required?
```

Por tanto:

```text
Migration Decision
!=
Source Code Patch
```

---

## 5. Objetivo

Transformar:

```text
Approved MigrationTransformationPlan
```

en:

```text
Deterministic Change Set
```

que pueda:

```text
preview
validate
apply
re-apply safely
rollback
audit
```

---

## 6. Arquitectura general

```text
Approved Migration Plan
          │
          ▼
 Transformation Planner
          │
          ▼
 Source Resolution
          │
          ▼
 AST / Structured Editors
          │
          ▼
 Change Set
          │
          ▼
 Preflight Validation
          │
          ▼
 Patch Preview
          │
          ▼
 Apply Transaction
          │
          ▼
 Post-transform Validation
```

---

## 7. No responsabilidades

El Transformer no deberá:

```text
reanalyze architecture
choose migration strategy
silently resolve blockers
execute production cutover
rewrite database data arbitrarily
declare behavioral equivalence
```

---

## 8. Entradas

El Transformer recibirá:

```text
MigrationTransformationPlan
Migration Intermediate Model snapshot
source references
rule provenance
target VoltStack version
project configuration
approved overrides
```

---

## 9. Plan Preconditions

Antes de transformar deberá verificar:

```text
plan status = APPROVED/EXECUTABLE
MIM fingerprint unchanged
rule set fingerprint unchanged
relevant source files unchanged
configuration fingerprint compatible
no unresolved blocker for selected unit
```

---

## 10. Stale Plan

Si el proyecto cambió:

```text
PLAN_STALE
```

El Transformer no deberá intentar adaptar silenciosamente el plan antiguo.

---

## 11. Source Fingerprints

Cada archivo relevante podrá tener:

```text
path
content hash
expected AST fingerprint
```

para detectar modificaciones posteriores al análisis.

---

## 12. Transformation Unit

La unidad mínima será:

```text
MigrationTransformationUnit
```

Conceptualmente:

```php
final readonly class MigrationTransformationUnit
{
    public function id(): string;
    public function decisionId(): string;
    public function inputs(): array;
    public function operations(): array;
    public function dependencies(): array;
    public function validations(): array;
}
```

---

## 13. Operation Types

Las operaciones podrán ser:

```text
CREATE_FILE
MODIFY_PHP_AST
MODIFY_CONFIG
MODIFY_COMPOSER
CREATE_MIGRATION
CREATE_TEST
CREATE_ADAPTER
DELETE_FILE
MOVE_FILE
RENAME_SYMBOL
ADD_IMPORT
REMOVE_IMPORT
ADD_ATTRIBUTE
REMOVE_ATTRIBUTE
REPLACE_CALL
REPLACE_EXPRESSION
ADD_CONTAINER_BINDING
REMOVE_CONTAINER_BINDING
```

---

## 14. Structured First

Regla:

```text
Structured transformation > text replacement
```

Para PHP deberá preferirse AST.

---

## 15. No Regex Refactoring

Regex podrá utilizarse para detección auxiliar, pero no como mecanismo principal para refactorizaciones semánticas de PHP.

---

## 16. PHP AST Transformer

Componente:

```text
PhpMigrationAstTransformer
```

responsable de:

```text
parse
locate symbols
modify nodes
preserve semantics
print source
```

---

## 17. Formatting Preservation

Cuando sea posible deberá preservar:

```text
comments
docblocks
attributes
imports
namespace
code style
line endings
```

---

## 18. Minimal Diff

El Transformer deberá preferir:

```text
smallest safe change
```

sobre reescribir archivos completos.

---

## 19. Symbol Resolution

Las transformaciones deberán operar sobre símbolos resueltos.

Ejemplo:

```text
Illuminate\Database\Eloquent\Model
```

y no únicamente sobre el texto:

```text
Model
```

---

## 20. Import Management

Al añadir:

```php
use VoltStack\Quantum\Database\...;
```

deberá evitar:

```text
duplicate imports
alias conflicts
unused imports
```

---

## 21. Namespace Safety

No deberá asumir que una clase se encuentra en un namespace específico por su ruta física.

---

## 22. Comments

Los comentarios existentes deberán conservarse salvo que documenten una API eliminada y la regla haya autorizado su modificación.

---

## 23. Generated Marker

El código generado podrá incluir metadata discreta cuando sea útil:

```text
generated by VoltStack migration tooling
rule ID
```

pero no deberá contaminar permanentemente el código con comentarios innecesarios.

---

## 24. Source Mapping

Cada operación deberá conservar:

```text
source location before
target location after
rule ID
decision ID
```

---

## 25. Change Set

El resultado previo a aplicar será:

```text
MigrationChangeSet
```

---

## 26. Change Set Contents

```text
files created
files modified
files deleted
symbols changed
config changed
dependencies changed
migrations generated
tests generated
validation requirements
rollback operations
```

---

## 27. Immutable Change Set

Una vez aprobado:

```text
Change Set
```

deberá ser inmutable.

---

## 28. Change Set Fingerprint

Se generará:

```text
change_set_fingerprint
```

para auditoría.

---

## 29. Dry Run

El modo predeterminado recomendado será:

```text
preview first
```

Ejemplo:

```bash
php volt database:migrate:transform --dry-run
```

---

## 30. Preview

El preview deberá mostrar:

```text
file
operation
rule
reason
risk
diff
required validation
```

---

## 31. Patch Representation

Las modificaciones podrán representarse internamente como:

```text
MigrationPatch
```

---

## 32. Patch Types

```text
AST_PATCH
TEXT_PATCH
CONFIG_PATCH
DEPENDENCY_PATCH
FILE_OPERATION
```

---

## 33. Text Patch Restrictions

Solo deberá utilizarse cuando:

```text
format lacks structured parser
change is deterministic
context fingerprint is verified
```

---

## 34. Atomic Apply

La aplicación deberá tratar el Change Set como una operación controlada.

Si una operación crítica falla:

```text
stop
rollback applied operations where possible
mark transformation failed
```

---

## 35. Filesystem Transaction

Conceptualmente:

```text
prepare temporary outputs
validate
backup originals
swap/apply
validate
commit
```

---

## 36. Backup

Antes de modificar archivos podrá crearse un backup local de transformación.

Esto no sustituye Git.

---

## 37. Version Control Requirement

La CLI deberá recomendar fuertemente:

```text
clean working tree
dedicated migration branch
```

antes de `--apply`.

---

## 38. Dirty Working Tree

Por defecto podrá requerir confirmación o bloquear operaciones de alto impacto.

---

## 39. Git Independence

Git será útil, pero el Transformer no deberá depender obligatoriamente de Git para funcionar.

---

## 40. Idempotence

Reaplicar la misma transformación deberá:

```text
detect already applied
or
produce no semantic change
```

---

## 41. Applied Transformation Marker

El estado podrá registrarse fuera del código:

```text
.voltstack/database-migration/applied/
```

con:

```text
decision ID
rule ID
change set fingerprint
source fingerprint
```

---

## 42. Marker Is Not Source of Truth

El Transformer también deberá verificar el estado real del código.

Un marker no basta para asumir que el cambio sigue presente.

---

## 43. Partial Application Detection

Si parte del cambio ya existe:

```text
PARTIALLY_APPLIED
```

y se requerirá reconciliación.

---

## 44. Reconciliation

Podrá:

```text
complete safe missing operations
rollback
request manual review
```

según la regla.

---

## 45. Transformer Registry

Existirá:

```text
MigrationTransformerRegistry
```

---

## 46. Transformer Contract

```php
interface MigrationTransformerInterface
{
    public function supports(
        MigrationDecision $decision
    ): bool;

    public function buildChangeSet(
        MigrationTransformationContext $context,
        MigrationDecision $decision
    ): MigrationChangeSet;
}
```

---

## 47. Transformer Categories

```text
EntityTransformer
FieldTransformer
RelationshipTransformer
RepositoryTransformer
QueryTransformer
ConnectionTransformer
TransactionTransformer
LifecycleTransformer
BehaviorTransformer
ConfigTransformer
ComposerTransformer
CompatibilityAdapterTransformer
TestScaffoldTransformer
```

---

## 48. One Decision, Multiple Operations

Ejemplo:

```text
REPLACE legacy connection
```

puede generar:

```text
modify config
add container binding
change constructor
replace call sites
remove static accessor
create compatibility adapter
```

---

## 49. Dependency Ordering

Las operaciones deberán respetar el grafo del plan.

---

## 50. Example Ordering

```text
Create target type
      ↓
Create entity metadata
      ↓
Transform repository
      ↓
Transform call sites
      ↓
Remove legacy dependency
```

---

## 51. Entity Transformation

Una entidad podrá requerir:

```text
create class
modify class
add attributes/metadata
remove source ORM inheritance
replace source traits
add target contracts
```

según el modelo nativo final.

---

## 52. No Blind Inheritance Removal

Eliminar:

```php
extends Model
```

no es suficiente.

Antes deberán resolverse:

```text
query APIs
casts
events
relationships
timestamps
soft deletes
serialization
```

---

## 53. Eloquent Model Transformation

Pipeline conceptual:

```text
Eloquent Model
     │
     ▼
MIM Entity + Behaviors
     │
     ▼
Approved Rules
     │
     ▼
VoltStack Entity/Repository representation
```

---

## 54. Doctrine Entity Transformation

No deberá copiar:

```text
EntityManager assumptions
Doctrine proxies
Doctrine annotations
Doctrine-specific collections
```

si el target no los utiliza.

---

## 55. Legacy Record Transformation

Un array-based record podrá permanecer:

```text
array/DTO
```

si el plan seleccionó `PRESERVE`.

---

## 56. Field Transformation

Deberá conservar:

```text
type
nullable
default
length
precision
scale
column name
generated behavior
```

---

## 57. Rename Safety

Renombrar una propiedad PHP no implica renombrar la columna física.

Ambas decisiones deberán ser independientes.

---

## 58. Naming Freeze

Durante migración se recomienda preservar:

```text
table names
column names
join table names
foreign key names when relevant
```

salvo decisión explícita.

---

## 59. Identifier Transformation

Deberá manejar:

```text
auto increment
sequence
UUID
ULID
assigned IDs
composite IDs
custom generators
```

sin cambiar estrategia accidentalmente.

---

## 60. Relationship Transformation

Podrá modificar:

```text
metadata
properties
collection initialization
repository queries
join configuration
```

según target.

---

## 61. Relationship Safety

No deberá asumir:

```text
relationship method name
=
database relationship truth
```

sin el MIM.

---

## 62. Polymorphic Relationships

Deberán preservar:

```text
type column
ID column
morph map
stored discriminator values
```

---

## 63. Repository Transformation

Una estrategia segura podrá ser:

```text
preserve interface
replace implementation
```

antes de cambiar consumidores.

---

## 64. Repository Contract

El Transformer deberá preservar inicialmente:

```text
method names
parameters
return types
exceptions
```

si el plan así lo requiere.

---

## 65. Query Transformation

El Transformer podrá convertir:

```text
Eloquent Builder
Doctrine DQL
Doctrine QueryBuilder
DBAL
PDO
raw SQL
```

hacia:

```text
VoltStack Query Builder
VoltStack Repository
VoltStack Native SQL
Query Object
```

según decisión.

---

## 66. Query Preservation

Si la decisión es:

```text
PRESERVE
```

el Transformer solo deberá mover la ejecución a infraestructura VoltStack cuando sea necesario.

---

## 67. SQL Semantic Preservation

No deberá reescribir SQL únicamente para hacerlo “más bonito”.

---

## 68. Parameter Binding

Transformaciones de SQL inseguro deberán utilizar binding cuando sea semánticamente posible.

---

## 69. Dynamic Identifiers

No deberán convertirse en:

```text
bind parameters
```

porque los identificadores requieren validación distinta.

---

## 70. Allow-list Generation

Si una regla aprobada lo indica, podrá generarse un resolver:

```php
final class AllowedReportTable
{
    // explicit mapping
}
```

en lugar de concatenación libre.

---

## 71. Transaction Transformation

Deberá preservar:

```text
begin boundary
commit boundary
rollback
connection ownership
isolation
savepoints
retry
```

---

## 72. Transaction Wrapper

Código:

```php
$pdo->beginTransaction();

try {
    // ...
    $pdo->commit();
} catch (\Throwable $e) {
    $pdo->rollBack();
    throw $e;
}
```

podrá migrarse a una abstracción transaccional VoltStack solo si la regla demuestra equivalencia.

---

## 73. No Transaction Shrinkage

El Transformer no deberá mover accidentalmente operaciones fuera de la frontera transaccional.

---

## 74. No Transaction Expansion

Tampoco deberá introducir operaciones externas dentro de una transacción sin decisión explícita.

---

## 75. Retry Preservation

Deadlock retry y serialization retry deberán conservarse o mapearse explícitamente.

---

## 76. Lifecycle Transformation

Hooks deberán transformarse respetando:

```text
timing
ordering
transaction relationship
side effects
```

---

## 77. Event Transformation

Un callback source-specific podrá convertirse a:

```text
VoltStack database event
domain event
application service
repository behavior
```

según la decisión.

---

## 78. No Mechanical Event Rename

Ejemplo:

```text
Doctrine PostPersist
```

no deberá renombrarse automáticamente a un evento VoltStack que ocurra en un momento diferente.

---

## 79. Behavior Transformation

Behaviors comunes:

```text
soft delete
timestamps
tenant filters
audit
encryption
optimistic locking
```

deberán tener transformers especializados.

---

## 80. Soft Delete

Podrá requerir:

```text
metadata
default query filtering
restore support
force delete
tests
```

---

## 81. Timestamp Transformation

Deberá verificar si:

```text
application
database trigger
database default
```

es quien realmente mantiene timestamps.

---

## 82. Duplicate Timestamp Prevention

No deberá activar simultáneamente dos mecanismos que escriban el mismo valor sin necesidad.

---

## 83. Type Transformer

Custom types podrán requerir:

```text
new VoltStack type
codec
binding logic
hydration logic
tests
```

---

## 84. Decimal Safety

Nunca deberá introducir:

```text
decimal → float
```

automáticamente para valores donde importe precisión.

---

## 85. Date Safety

Deberá preservar:

```text
timezone
mutability
microseconds
database representation
```

cuando formen parte del contrato.

---

## 86. Encryption Safety

Una transformación que toque datos cifrados será conservadora.

No deberá:

```text
decrypt/re-encrypt production data
change algorithm
change key
change serialized format
```

sin plan explícito.

---

## 87. Connection Transformation

Podrá convertir:

```text
manual PDO creation
global connection
legacy factory
```

hacia:

```text
VoltStack ConnectionManager
```

---

## 88. Secret Preservation

Configuración:

```text
env references
secret manager references
vault references
```

deberá preservarse.

Nunca deberá insertar secretos reales en archivos generados.

---

## 89. Config Transformer

Debe soportar transformaciones estructuradas cuando el formato lo permita.

---

## 90. Environment Files

`.env` deberá tratarse con especial cuidado.

El Transformer podrá sugerir nombres de variables, pero no deberá imprimir secretos en reportes.

---

## 91. Composer Transformer

Podrá:

```text
add VoltStack migration/runtime package
remove legacy ORM package
adjust autoload
```

solo cuando el plan lo autorice.

---

## 92. Dependency Removal Gate

No deberá eliminar:

```text
laravel/database
doctrine/orm
```

mientras existan referencias runtime activas.

---

## 93. Dependency Scan

Antes de remover un paquete deberá verificar:

```text
imports
inheritance
attributes
annotations
config
container bindings
CLI scripts
tests
serialized references where relevant
```

---

## 94. Dev-only Migration Packages

Los adapters de migración podrán instalarse como dependencias de desarrollo y eliminarse después.

---

## 95. Compatibility Adapter Generation

Cuando el plan indique `ADAPT`, el Transformer podrá generar:

```text
Legacy API
    │
    ▼
Compatibility Adapter
    │
    ▼
VoltStack Database
```

---

## 96. Adapter Metadata

Cada adapter generado deberá registrar:

```text
temporary status
legacy API represented
migration decision
deprecation path
```

---

## 97. Adapter Deprecation

Podrá integrarse con:

```text
324_DATABASE_DEPRECATION_POLICY.md
```

para localizar uso restante.

---

## 98. No Permanent Compatibility Pollution

Los adapters no deberán añadirse al core nativo de `Quantum/Database`.

---

## 99. Container Transformation

Podrá generar:

```text
bindings
aliases
factories
```

siguiendo:

```text
312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md
```

---

## 100. Framework Integration

Las transformaciones deberán respetar:

```text
311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md
```

---

## 101. Configuration Integration

Deberán respetar:

```text
313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md
```

---

## 102. Cache Integration

Si el source ORM usa cache, la transformación deberá considerar:

```text
314_DATABASE_CACHE_INTEGRATION_SYSTEM.md
```

---

## 103. Event Integration

Para lifecycle/event mappings:

```text
315_DATABASE_EVENT_SYSTEM_INTEGRATION.md
```

---

## 104. Telemetry Integration

El Transformer podrá añadir instrumentación necesaria siguiendo:

```text
316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md
```

---

## 105. Validation Integration

Transformaciones que afecten validación deberán respetar:

```text
317_DATABASE_VALIDATION_INTEGRATION_SYSTEM.md
```

---

## 106. Authentication Integration

Migraciones de modelos de autenticación deberán respetar:

```text
318_DATABASE_AUTHENTICATION_INTEGRATION_SYSTEM.md
```

---

## 107. Authorization Integration

Migraciones de roles/permisos deberán respetar:

```text
319_DATABASE_AUTHORIZATION_INTEGRATION_SYSTEM.md
```

---

## 108. Queue Integration

El Transformer deberá detectar riesgos de:

```text
serialized ORM entities
legacy proxies
database handles
```

en jobs según:

```text
320_DATABASE_JOB_AND_QUEUE_INTEGRATION_SYSTEM.md
```

---

## 109. HTTP Lifecycle

Cambios deberán respetar:

```text
321_DATABASE_HTTP_REQUEST_LIFECYCLE_INTEGRATION.md
```

especialmente para persistent workers.

---

## 110. Generated Migrations

El Transformer podrá generar:

```text
VoltStack migration files
```

cuando el plan requiera cambios de schema.

---

## 111. Schema Mutation Boundary

No deberá aplicar directamente esos cambios a producción.

Generará artefactos para el sistema normal de migrations.

---

## 112. Baseline Generation

Si el plan utiliza:

```text
BASELINE_CURRENT_SCHEMA
```

podrá generar un baseline representativo.

---

## 113. Baseline Safety

No deberá interpretar baseline como:

```text
run CREATE TABLE against existing production DB
```

---

## 114. Existing Migration Conversion

Podrá convertir historial cuando la estrategia sea:

```text
FULL_CONVERSION
```

pero preservará orden y semántica.

---

## 115. Seeder Transformation

Podrá transformar:

```text
Eloquent seeders
Doctrine fixtures
legacy SQL seed scripts
```

cuando exista regla.

---

## 116. Factory Transformation

Podrá generar factories equivalentes cuando sean parte del proyecto.

---

## 117. Test Scaffold Generation

Una decisión podrá requerir generar:

```text
characterization test
repository test
transaction test
query equivalence test
```

---

## 118. Generated Tests Are Not Proof

La existencia del test generado no significa que haya pasado.

---

## 119. Test TODO Policy

Cuando no pueda generar assertions correctas, deberá marcar explícitamente:

```text
MANUAL ASSERTION REQUIRED
```

en lugar de producir un test que siempre pasa.

---

## 120. Transformation Stages

Una ejecución podrá dividirse:

```text
PREPARE
GENERATE
PREVIEW
APPLY
VERIFY_STRUCTURE
FINALIZE
```

---

## 121. PREPARE

Verifica:

```text
plan
source fingerprints
filesystem
permissions
dependencies
```

---

## 122. GENERATE

Construye:

```text
temporary files
AST patches
config patches
tests
migrations
```

sin modificar aún originales.

---

## 123. PREVIEW

Produce diff.

---

## 124. APPLY

Realiza cambios.

---

## 125. VERIFY_STRUCTURE

Comprueba:

```text
PHP parses
autoload resolves
config parses
generated artifacts exist
```

---

## 126. FINALIZE

Registra el Change Set aplicado.

No declara equivalencia funcional.

---

## 127. PHP Syntax Validation

Todo archivo PHP modificado deberá pasar:

```text
syntax parse
```

antes de commit del Change Set.

---

## 128. Static Analysis

Cuando esté configurado, podrá ejecutar:

```text
PHPStan
Psalm
VoltStack Analyzer
```

---

## 129. Autoload Validation

Después de cambios de clases/namespaces deberá validarse autoload.

---

## 130. Composer Validation

Si `composer.json` cambia deberá verificarse su estructura.

---

## 131. No Implicit Composer Update

Modificar requisitos y ejecutar:

```text
composer update
```

son operaciones diferentes.

El Transformer no deberá actualizar todo el dependency graph sin decisión explícita.

---

## 132. Lock File

La estrategia para `composer.lock` deberá ser explícita.

---

## 133. Formatting

Podrá ejecutar un formatter configurado, pero solo sobre archivos afectados o bajo policy explícita.

---

## 134. Avoid Diff Noise

No deberá reformatear todo el proyecto durante una migración.

---

## 135. File Encoding

Deberá preservar:

```text
UTF-8
line endings where practical
BOM policy
```

---

## 136. Generated File Naming

Los nombres deberán seguir:

```text
VoltStack naming conventions
```

y evitar colisiones.

---

## 137. Existing Target Symbol

Si el target ya existe:

```text
EXISTING_TARGET_CONFLICT
```

a menos que la regla soporte merge.

---

## 138. Merge Transformer

Algunos artefactos podrán integrarse:

```text
config arrays
service registration
repository implementation
```

mediante merge estructurado.

---

## 139. No Blind File Overwrite

Un archivo existente nunca deberá reemplazarse completo únicamente porque el generador conoce una plantilla.

---

## 140. User Code Preservation

El Transformer deberá distinguir:

```text
generated region
user-owned region
```

cuando genere archivos parcialmente administrados.

---

## 141. Code Generation Architecture

Debe reutilizar conceptos de:

```text
310_DATABASE_CODE_GENERATION_SYSTEM.md
```

en lugar de implementar un segundo generador independiente.

---

## 142. Template Versioning

Los templates generados deberán estar versionados.

---

## 143. Template Drift

Un cambio de template no deberá reescribir automáticamente código previamente generado y modificado por el usuario.

---

## 144. Conflict Markers

No deberá insertar silenciosamente markers estilo Git dentro de código productivo.

Los conflictos se reportarán antes de aplicar.

---

## 145. Manual Edit Instructions

Cuando un cambio no sea automatizable, podrá producir:

```text
file
symbol
reason
suggested transformation
validation required
```

---

## 146. Example Manual Instruction

```text
File:
src/Billing/MoneyType.php

Reason:
Custom encrypted decimal codec cannot be
mapped safely.

Action:
Implement VoltStack custom type preserving
existing serialization.

Required validation:
Round-trip existing fixtures.
```

---

## 147. Transformation Report

Cada ejecución producirá:

```text
MigrationTransformationReport
```

---

## 148. Report Contents

```text
plan ID
change set ID
files changed
symbols changed
rules applied
manual items
skipped items
failures
structural validation
rollback availability
```

---

## 149. Audit Log

Las operaciones deberán ser trazables sin registrar secretos.

---

## 150. Security

El Transformer trabaja con código y configuración sensible.

Deberá aplicar:

```text
path validation
safe file writes
secret redaction
symlink awareness
permission preservation
untrusted plugin restrictions
```

---

## 151. Path Traversal

Ninguna regla o plugin deberá poder escribir fuera del project root salvo permiso explícito.

---

## 152. Symlink Safety

Antes de modificar un archivo deberá determinar si:

```text
path is symlink
```

y aplicar policy segura.

---

## 153. File Permissions

Deberá preservar permisos cuando sustituya archivos.

---

## 154. Secret Redaction

Diffs y reportes deberán redactar valores sensibles cuando sea necesario.

---

## 155. Untrusted Generated SQL

El Transformer no deberá ejecutar SQL generado solo para comprobarlo.

La validación de SQL deberá usar parsers/compilers o entornos controlados.

---

## 156. Extension Security

Transformers de terceros ejecutan código.

Solo deberán cargarse desde paquetes confiables.

---

## 157. Determinism

Mismo:

```text
plan
source
transformer versions
templates
configuration
```

deberá producir el mismo Change Set.

---

## 158. Time-independent Output

Los timestamps no deberán alterar fingerprints semánticos.

---

## 159. Random IDs

No deberán usarse IDs aleatorios en nombres generados salvo que la especificación del artefacto lo requiera.

---

## 160. Concurrency

No deberán ejecutarse dos transformaciones sobre el mismo proyecto sin coordinación.

---

## 161. Project Lock

Podrá existir:

```text
MigrationTransformationLock
```

para impedir escritura concurrente.

---

## 162. Read-only Analysis Concurrency

El análisis puede ejecutarse concurrentemente; la escritura requiere exclusión.

---

## 163. Crash Recovery

Si el proceso termina durante APPLY, el estado deberá permitir determinar:

```text
which operations completed
which did not
whether rollback is possible
```

---

## 164. Transformation Journal

Se utilizará conceptualmente:

```text
MigrationTransformationJournal
```

---

## 165. Journal Entries

```text
operation ID
started
completed
failed
backup reference
result fingerprint
```

---

## 166. Journal Durability

El journal deberá actualizarse antes/después de operaciones críticas.

---

## 167. Rollback

El Transformer implementará rollback técnico de archivos.

El sistema completo de recuperación se formalizará en:

```text
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 168. File Rollback

Podrá restaurar:

```text
modified files
deleted files
moved files
created files
```

si existen backups válidos.

---

## 169. Schema Rollback

No será ejecutado directamente por el Code Transformer.

---

## 170. Data Rollback

Tampoco será responsabilidad directa del Transformer.

---

## 171. Partial Rollback

Si una operación no es reversible deberá aparecer antes de APPLY.

---

## 172. FrankenPHP

El Transformer deberá generar código compatible con el modelo de ejecución persistente.

---

## 173. Forbidden Generated Patterns

Deberá evitar generar:

```text
request-specific static mutable state
static current tenant
static EntityManager
global transaction state
global request connection state
```

---

## 174. Request Scope

Dependencias request-local deberán resolverse mediante los mecanismos de contexto/container de VoltStack.

---

## 175. Connection Cleanup

Código generado deberá respetar los contratos de lifecycle de Database.

---

## 176. RoadRunner/OpenSwoole

Las mismas reglas de aislamiento aplicarán a sus paquetes oficiales.

---

## 177. Multitenancy

Si está instalado, las transformaciones deberán preservar:

```text
tenant connection resolution
tenant schema selection
tenant filters
```

sin hacer del paquete una dependencia del transformer core.

---

## 178. SaaS

El Transformer core seguirá independiente del paquete SaaS.

---

## 179. Performance

Transformaciones masivas deberán usar:

```text
indexed source map
incremental AST parsing
cached symbol resolution
batched writes
```

cuando sea posible.

---

## 180. Memory

No será necesario mantener todos los AST del proyecto simultáneamente.

---

## 181. Changed Files Only

El Transformer deberá limitar parsing/escritura al conjunto afectado por el plan.

---

## 182. Telemetry

Métricas posibles:

```text
database.migration.transform.duration
database.migration.transform.files
database.migration.transform.operations
database.migration.transform.failures
database.migration.transform.rollbacks
database.migration.transform.manual_items
```

---

## 183. Logging

Canal:

```text
database.migration.transform
```

---

## 184. CLI

Comando conceptual:

```bash
php volt database:migrate:transform
```

---

## 185. Dry Run CLI

```bash
php volt database:migrate:transform \
    --plan=.voltstack/database-migration/plan.json \
    --dry-run
```

---

## 186. Apply

```bash
php volt database:migrate:transform \
    --plan=.voltstack/database-migration/plan.json \
    --apply
```

---

## 187. Scope

Podrá limitarse:

```bash
--module=Users
--entity=User
--decision=...
```

si el plan permite aplicar ese subconjunto de forma segura.

---

## 188. Atomic Group Enforcement

Si una decisión pertenece a un grupo atómico, no podrá aplicarse aisladamente.

---

## 189. Diff Output

La CLI deberá mostrar o exportar:

```text
unified diff
structured change summary
```

---

## 190. Interactive Approval

La UX podrá permitir revisar cambios, pero los modos CI deberán ser completamente no interactivos.

---

## 191. CI Mode

En CI normalmente:

```text
generate
validate
compare expected
```

sin aplicar sobre el branch salvo workflow explícito.

---

## 192. Exit Codes

Deberán distinguir:

```text
success
changes available
manual review
blocked
stale plan
transform failure
validation failure
```

---

## 193. Integration with 333

Después de generar artefactos de schema:

```text
Schema Compatibility System
```

deberá verificar que el resultado target corresponde al schema esperado.

---

## 194. Integration with 334

Después de aplicar cambios:

```text
Behavior Verification System
```

verificará equivalencia funcional.

---

## 195. Integration with 335

Compatibility adapters generados podrán participar en Dual Runtime.

---

## 196. Integration with 336

Queries transformadas podrán ejecutarse en Shadow Comparison.

---

## 197. Integration with 337

Los requisitos de validación se convertirán en suites ejecutables.

---

## 198. Integration with 338

La CLI completa coordinará:

```text
analyze
plan
preview
apply
validate
rollback
```

---

## 199. Integration with 339

Todos los Change Sets, conflictos y resultados alimentarán reporting.

---

## 200. Integration with 340

El journal, backups y rollback descriptors serán entradas del Recovery System.

---

## 201. Componentes principales

```text
DatabaseMigrationCodeTransformer
│
├── MigrationTransformationContext
├── MigrationPlanVerifier
├── MigrationSourceFingerprintVerifier
├── MigrationTransformerRegistry
├── MigrationTransformationPlanner
├── PhpMigrationAstTransformer
├── MigrationSymbolResolver
├── MigrationImportManager
├── MigrationConfigTransformer
├── MigrationComposerTransformer
├── MigrationCodeGeneratorAdapter
├── MigrationPatchBuilder
├── MigrationChangeSetBuilder
├── MigrationChangeSetValidator
├── MigrationDiffRenderer
├── MigrationFilesystemTransaction
├── MigrationTransformationJournal
├── MigrationBackupManager
├── MigrationStructuralValidator
└── MigrationTransformationReporter
```

---

## 202. Pipeline completo

```text
Approved Plan
      │
      ▼
Verify Plan Freshness
      │
      ▼
Resolve Source Symbols
      │
      ▼
Select Transformers
      │
      ▼
Build Operations
      │
      ▼
Build Change Set
      │
      ▼
Validate Change Set
      │
      ▼
Generate Preview
      │
      ▼
Explicit Apply
      │
      ▼
Filesystem Transaction
      │
      ▼
Structural Validation
      │
      ▼
Transformation Journal
      │
      ▼
Behavior / Schema Verification
```

---

## 203. Ejemplo: PDO a ConnectionManager

Origen:

```php
final class UserRepository
{
    public function __construct()
    {
        $this->pdo = new PDO(
            $_ENV['DB_DSN'],
            $_ENV['DB_USER'],
            $_ENV['DB_PASSWORD']
        );
    }
}
```

Decisión:

```text
REPLACE
```

Cambio conceptual:

```php
final class UserRepository
{
    public function __construct(
        private ConnectionInterface $connection
    ) {
    }
}
```

La configuración de credenciales permanece fuera del repository.

---

## 204. Ejemplo: Repository Compatibility

Origen:

```php
interface LegacyUserRepository
{
    public function findByEmail(string $email): ?array;
}
```

Si el contrato debe preservarse:

```text
ADAPT
```

el Transformer puede generar una implementación VoltStack manteniendo la interfaz temporalmente.

---

## 205. Ejemplo: Raw SQL

Origen:

```php
$pdo->prepare(
    'SELECT id, email FROM users WHERE email = :email'
);
```

Si la decisión es:

```text
PRESERVE
```

podrá convertirse a Native SQL sobre una conexión VoltStack sin alterar la consulta.

---

## 206. Ejemplo: Dynamic SQL

Origen:

```php
$sql = 'SELECT * FROM ' . $table;
```

Sin allow-list demostrada:

```text
BLOCK
```

El Transformer no deberá tocarlo.

---

## 207. Ejemplo: Soft Delete

Una regla puede requerir:

```text
remove source-specific trait
add VoltStack behavior metadata
preserve deleted_at column
generate behavior tests
```

como un único Change Set coordinado.

---

## 208. Ejemplo: Dependency Removal

Solo después de:

```text
0 runtime references
0 source attributes
0 config references
0 compatibility adapters requiring package
```

podrá generarse la operación:

```text
REMOVE_COMPOSER_DEPENDENCY
```

---

## 209. Testing Architecture

El Transformer deberá contar con:

```text
unit tests
AST transformation tests
golden source tests
idempotence tests
conflict tests
crash recovery tests
rollback tests
security tests
cross-platform filesystem tests
integration fixture tests
```

---

## 210. Golden Source Tests

Ejemplo:

```text
input.php
expected.php
```

para cada transformación.

---

## 211. AST Semantic Tests

No bastará comparar texto.

También deberá comprobarse que el AST esperado sea equivalente.

---

## 212. Idempotence Test

```text
transform(input)
→ output

transform(output)
→ no changes
```

---

## 213. Comment Preservation Test

Casos con comentarios/docblocks deberán verificarse.

---

## 214. Alias Import Tests

Se probarán:

```text
same short class names
aliased imports
fully-qualified references
```

---

## 215. Failure Injection

Las pruebas deberán simular fallo después de:

```text
1st operation
middle operation
final write
```

para verificar journal/rollback.

---

## 216. Security Tests

Se probará:

```text
path traversal
symlink targets
secret redaction
untrusted path input
malformed source
```

---

## 217. Performance Tests

Fixtures grandes medirán:

```text
files/second
memory
AST cache effectiveness
Change Set generation time
```

---

## 218. Decisiones arquitectónicas

### Decisión 1

El Code Transformer ejecutará planes; no decidirá estrategias.

### Decisión 2

Solo transformará planes aprobados y no obsoletos.

### Decisión 3

PHP se modificará preferentemente mediante AST.

### Decisión 4

Regex no será el mecanismo principal de refactoring semántico.

### Decisión 5

Las transformaciones buscarán minimal diffs.

### Decisión 6

Todo cambio conservará provenance hacia rule/decision/source.

### Decisión 7

Los Change Sets serán deterministas e inmutables una vez aprobados.

### Decisión 8

Dry-run y preview serán capacidades de primera clase.

### Decisión 9

No se sobrescribirán archivos existentes de forma ciega.

### Decisión 10

Se preservará código del usuario y se detectarán conflictos.

### Decisión 11

La aplicación de cambios utilizará journal y backups para recuperación.

### Decisión 12

El Transformer será idempotente o detectará estado parcialmente aplicado.

### Decisión 13

Los cambios de schema se generarán como migrations, no se aplicarán directamente a producción.

### Decisión 14

Los secretos nunca deberán copiarse a código, diffs o reportes.

### Decisión 15

La eliminación de dependencias legacy ocurrirá únicamente después de demostrar que no quedan referencias necesarias.

### Decisión 16

Los compatibility adapters serán temporales y permanecerán fuera del core de Database.

### Decisión 17

El código generado será compatible con persistent workers y evitará estado request-specific global.

### Decisión 18

La transformación estructural exitosa no equivale a migración verificada; la verificación pertenece a fases posteriores.

---

## 219. Resultado esperado

Entrada:

```text
MigrationTransformationPlan
├── decisions
├── dependencies
├── validators
└── rollback metadata
```

Salida:

```text
MigrationChangeSet
├── PHP AST patches
├── configuration changes
├── dependency changes
├── generated migrations
├── generated adapters
├── generated tests
├── source mappings
├── structural validation
└── rollback journal
```

---

## 220. Principio final

```text
Never rewrite what has not been decided.

Never apply what has not been previewed.

Never overwrite what has changed unexpectedly.

Never call a transformation verified until validators prove it.
```

---

## 221. Conclusión

`DATABASE_MIGRATION_CODE_TRANSFORMER` constituye la capa de ejecución técnica del sistema de migración de VoltStack Database V1.

Su arquitectura completa queda:

```text
Source Systems
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
Approved Transformation Plan
      │
      ▼
Database Migration Code Transformer
      │
      ▼
Change Set
      │
      ▼
Structural Validation
      │
      ▼
Schema / Behavior / Query Verification
```

La separación es deliberada:

```text
Analysis understands.

MIM describes.

Rules decide.

Transformer changes.

Validators prove.

Recovery reverses when necessary.
```

Esto permite que una migración desde Eloquent, Doctrine o sistemas legacy sea tratada como una operación arquitectónica reproducible y auditable, y no como una colección de reemplazos de texto sobre el proyecto.

---

**Documento:** `332_DATABASE_MIGRATION_CODE_TRANSFORMER.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
