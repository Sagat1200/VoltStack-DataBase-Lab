# 338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md

## 1. Propósito

Este documento define la arquitectura oficial de **CLI y Developer Experience (DX)** para el sistema de migración de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseMigrationCliAndDeveloperExperienceSystem
```

y proporciona una interfaz coherente para:

```text
discover
analyze
inspect
plan
transform
verify
shadow
validate
cut over
diagnose
recover
```

migraciones desde sistemas externos de persistencia hacia VoltStack.

El objetivo no es crear una colección aislada de comandos.

El objetivo es construir una experiencia operativa completa alrededor del pipeline definido entre los documentos 325–340.

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
332_DATABASE_MIGRATION_CODE_TRANSFORMER.md
333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md
334_DATABASE_MIGRATION_BEHAVIOR_VERIFICATION_SYSTEM.md
335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
337_DATABASE_MIGRATION_TESTING_AND_VALIDATION_SYSTEM.md
```

y alimentará:

```text
339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 3. Principio fundamental

```text
Migration automation must remain explainable,
inspectable, previewable and reversible where promised.
```

---

# Parte I — Objetivos de DX

## 4. Objetivos

La experiencia deberá permitir que un desarrollador pueda responder fácilmente:

```text
What was discovered?
What is compatible?
What is risky?
What will change?
Why will it change?
What has already changed?
What remains?
What blocks the next state?
Can I safely retry?
Can I roll back?
```

---

## 5. Diseño orientado a workflow

Los comandos deberán corresponder al proceso real:

```text
Discover
   ↓
Analyze
   ↓
Inspect
   ↓
Plan
   ↓
Transform
   ↓
Verify
   ↓
Validate
   ↓
Shadow
   ↓
Cutover
   ↓
Observe
   ↓
Remove Legacy
```

---

# Parte II — Command Namespace

## 6. Namespace principal

La CLI utilizará:

```bash
php volt database:migrate:*
```

---

## 7. Familias

```text
database:migrate:status
database:migrate:analyze
database:migrate:model
database:migrate:plan
database:migrate:rules
database:migrate:transform
database:migrate:schema
database:migrate:behavior
database:migrate:runtime
database:migrate:shadow
database:migrate:validate
database:migrate:report
database:migrate:rollback
database:migrate:doctor
```

---

## 8. Consistencia

Todos los comandos deberán compartir convenciones para:

```text
--unit
--source
--profile
--format
--output
--dry-run
--explain
--no-interaction
--verbose
```

cuando sean aplicables.

---

# Parte III — Entry Point

## 9. Migration Console

Componente:

```text
MigrationConsoleApplication
```

---

## 10. Integración

Se integrará con el CLI general de VoltStack, sin convertir `Quantum/Database` en dependiente de una implementación concreta de terminal.

---

## 11. Command Bus

Conceptualmente:

```text
CLI Input
   ↓
Migration Command
   ↓
Application Service
   ↓
Migration Core
   ↓
Structured Result
   ↓
Console Renderer
```

---

## 12. Rule

La lógica de migración no deberá residir en clases de comando.

---

# Parte IV — Status Command

## 13. Command

```bash
php volt database:migrate:status
```

---

## 14. Purpose

Mostrar el estado global de migración.

---

## 15. Example

```text
VoltStack Database Migration

Source systems:
  Eloquent       detected
  Legacy PDO     detected

Migration units:
  Users          VOLTSTACK_PRIMARY
  Catalog        SHADOW_READ
  Billing        PREPARED
  Reports        LEGACY

Schema:
  compatible with warnings

Validation:
  2 units ready
  1 unit blocked
  1 unit not started

Legacy dependencies:
  37 remaining

Rollback:
  available for Users
```

---

## 16. No False Green

Un estado global no deberá ocultar blockers de unidades individuales.

---

# Parte V — Analyze Command

## 17. Command

```bash
php volt database:migrate:analyze
```

---

## 18. Options

```bash
--source=eloquent
--source=doctrine
--source=legacy
--profile=quick
--profile=deep
--with-schema
--with-runtime
--snapshot
```

---

## 19. Output

```text
sources
models/entities
repositories
queries
transactions
schema conflicts
legacy dependencies
unknowns
blockers
```

---

## 20. Explain Finding

```bash
php volt database:migrate:analyze \
    --explain=VSDB-MIG-CORE-0042
```

---

# Parte VI — Intermediate Model CLI

## 21. Inspect MIM

```bash
php volt database:migrate:model
```

---

## 22. Filter

```bash
php volt database:migrate:model \
    --entity=User
```

---

## 23. Query

```bash
php volt database:migrate:model \
    --query=billing.invoice.list
```

---

## 24. Unknowns

```bash
php volt database:migrate:model \
    --unknowns
```

---

## 25. Conflicts

```bash
php volt database:migrate:model \
    --conflicts
```

---

## 26. Export

```bash
php volt database:migrate:model \
    --format=json \
    --output=.voltstack/migration/model.json
```

---

# Parte VII — Plan Command

## 27. Command

```bash
php volt database:migrate:plan
```

---

## 28. Purpose

Construir el plan de migración sin modificar código.

---

## 29. Example

```text
Migration Plan: Billing

1. Introduce repository boundary
2. Transform Invoice persistence mapping
3. Preserve native payment SQL
4. Convert transaction ownership to VoltStack
5. Keep database trigger audit_invoice
6. Generate behavior contracts
7. Enable shadow reads
8. Validate
9. Switch read ownership
10. Switch write ownership
```

---

## 30. Dependency Ordering

El plan deberá respetar:

```text
dependency graph
SCCs
schema dependencies
transaction boundaries
migration barriers
```

---

# Parte VIII — Rules CLI

## 31. List Rules

```bash
php volt database:migrate:rules
```

---

## 32. Inspect Rule

```bash
php volt database:migrate:rules \
    --show=VSDB-MIG-ELOQ-0012
```

---

## 33. Output

```text
rule ID
source pattern
target semantic
confidence requirements
preconditions
transformer
validator
risk
```

---

## 34. Overrides

```bash
php volt database:migrate:rules \
    --overrides
```

---

# Parte IX — Transform Command

## 35. Command

```bash
php volt database:migrate:transform
```

---

## 36. Safe Default

El comando deberá favorecer inicialmente:

```text
preview
```

antes de aplicar cambios.

---

## 37. Dry Run

```bash
php volt database:migrate:transform \
    --unit=Billing \
    --dry-run
```

---

## 38. Diff

Debe mostrar cambios propuestos:

```diff
- use Illuminate\Database\Eloquent\Model;
+ use VoltStack\Quantum\Database\Persistence\Entity;
```

cuando corresponda.

---

## 39. Apply

La aplicación real requerirá acción explícita:

```bash
php volt database:migrate:transform \
    --unit=Billing \
    --apply
```

---

## 40. Journal

Toda transformación aplicada generará:

```text
MigrationTransformationJournal
```

---

# Parte X — Explain Transformation

## 41. Explain

```bash
php volt database:migrate:transform \
    --unit=Billing \
    --explain
```

---

## 42. Required Explanation

Por cambio:

```text
source construct
detected semantic
rule applied
target construct
confidence
validation required
```

---

## 43. No Opaque Rewrite

No deberá existir:

```text
"142 files changed successfully"
```

sin posibilidad de inspeccionar por qué.

---

# Parte XI — Interactive Review

## 44. Interactive Mode

En terminal humana podrá ofrecer:

```text
accept
reject
skip
inspect
edit override
show evidence
```

---

## 45. Non-interactive

CI deberá utilizar:

```bash
--no-interaction
```

---

## 46. Determinism

La misma configuración no interactiva deberá producir resultados reproducibles.

---

# Parte XII — Conflict Resolution

## 47. Conflict

Cuando exista ambigüedad:

```text
MIGRATION_CONFLICT
```

---

## 48. CLI

```bash
php volt database:migrate:plan \
    --conflicts
```

---

## 49. Example

```text
Conflict:
User.deleted_at

Eloquent metadata:
soft delete

Actual schema:
nullable timestamp

Legacy SQL:
includes deleted rows

Decision required:
Which behavior is authoritative?
```

---

## 50. Resolution

El desarrollador podrá crear:

```text
MigrationOverride
```

---

## 51. Auditability

La resolución deberá guardar:

```text
reason
scope
evidence
author/decision metadata where available
timestamp
```

---

# Parte XIII — Schema CLI

## 52. Analyze

```bash
php volt database:migrate:schema \
    --analyze
```

---

## 53. Compare

```bash
php volt database:migrate:schema \
    --compare
```

---

## 54. Output

```text
expected
actual
source assumptions
conflicts
risk
```

---

## 55. Plan

```bash
php volt database:migrate:schema \
    --plan
```

---

## 56. No Implicit Mutation

Un análisis de schema nunca deberá modificar la base.

---

# Parte XIV — Behavior CLI

## 57. Command

```bash
php volt database:migrate:behavior
```

---

## 58. List Contracts

```bash
php volt database:migrate:behavior \
    --contracts
```

---

## 59. Verify

```bash
php volt database:migrate:behavior \
    --verify \
    --unit=Billing
```

---

## 60. Explain Failure

```bash
php volt database:migrate:behavior \
    --explain=VSDB-MIG-BEHAVIOR-0021
```

---

# Parte XV — Dual Runtime CLI

## 61. Status

```bash
php volt database:migrate:runtime \
    --status
```

---

## 62. Unit

```bash
php volt database:migrate:runtime \
    --unit=Catalog
```

---

## 63. Transition

```bash
php volt database:migrate:runtime \
    --unit=Catalog \
    --state=VOLTSTACK_READ
```

---

## 64. Preflight

Antes de aplicar:

```text
schema gate
behavior gate
validation gate
rollback readiness
deployment compatibility
```

---

## 65. Preview Transition

```bash
php volt database:migrate:runtime \
    --unit=Catalog \
    --state=VOLTSTACK_READ \
    --dry-run
```

---

# Parte XVI — Shadow CLI

## 66. Status

```bash
php volt database:migrate:shadow \
    --status
```

---

## 67. Eligibility

```bash
php volt database:migrate:shadow \
    --analyze-eligibility
```

---

## 68. Enable

```bash
php volt database:migrate:shadow \
    --unit=Catalog \
    --enable
```

---

## 69. Sampling

```bash
php volt database:migrate:shadow \
    --unit=Catalog \
    --sample=10%
```

---

## 70. Differences

```bash
php volt database:migrate:shadow \
    --differences
```

---

# Parte XVII — Validation CLI

## 71. Command

```bash
php volt database:migrate:validate
```

---

## 72. Profiles

```bash
--profile=quick
--profile=standard
--profile=deep
--profile=ci
--profile=pre-cutover
--profile=post-cutover
```

---

## 73. Unit

```bash
php volt database:migrate:validate \
    --unit=Billing
```

---

## 74. Dimension

```bash
php volt database:migrate:validate \
    --only=schema,behavior,transactions
```

---

## 75. Explain Gate

```bash
php volt database:migrate:validate \
    --explain-gate
```

---

## 76. Example

```text
Billing → VOLTSTACK_WRITE

BLOCKED

✓ Static
✓ Schema
✓ Data
✓ Read behavior
✗ Transaction atomicity
✓ Security
✓ FrankenPHP isolation
✓ Recovery plan

Blocking evidence:
VSDB-MIG-VALIDATION-0178
```

---

# Parte XVIII — Doctor Command

## 77. Command

```bash
php volt database:migrate:doctor
```

---

## 78. Purpose

Validar la propia infraestructura de migración.

---

## 79. Checks

```text
migration packages installed
source adapters available
DB connectivity
schema introspection
required PHP extensions
cache directory
permissions
runtime profile
telemetry integration
test environment safety
manifest consistency
```

---

## 80. Example

```text
Migration Doctor

[OK] Eloquent adapter
[OK] VoltStack database connection
[OK] Schema introspection
[OK] FrankenPHP runtime profile
[WARN] Runtime evidence disabled
[FAIL] Test database points to protected environment
```

---

# Parte XIX — Safety Confirmation

## 81. Destructive Commands

Operaciones que puedan modificar:

```text
source code
database schema
runtime ownership
migration state
```

deberán ser explícitas.

---

## 82. Interactive Confirmation

Ejemplo:

```text
This operation will modify 17 files.

Proceed? [y/N]
```

---

## 83. CI

En CI se utilizarán flags explícitos.

---

## 84. No Dangerous Guessing

La CLI no deberá inferir consentimiento a partir de:

```text
TTY presence
environment name alone
previous command
```

---

# Parte XX — Production Guard

## 85. Production Environment

Operaciones destructivas deberán pasar:

```text
MigrationEnvironmentGuard
```

---

## 86. Protected Environment

Ejemplos:

```text
production
staging with production clone
regulated environment
```

---

## 87. Stronger Confirmation

Podrá exigir:

```text
explicit flag
deployment token
approved plan fingerprint
```

según integración.

---

## 88. No Secret in CLI History

No se deberán pasar credenciales directamente en argumentos recomendados.

---

# Parte XXI — Dry Run

## 89. Universal Principle

Toda operación modificadora deberá soportar, cuando sea técnicamente viable:

```text
dry-run
```

---

## 90. Dry Run Output

Debe indicar:

```text
what would change
what would not change
preconditions
risks
generated artifacts
required follow-up validation
```

---

## 91. Fidelity

Dry-run deberá utilizar la misma lógica de planificación que apply.

---

# Parte XXII — Idempotence

## 92. Retry

Comandos deberán diseñarse para reejecución segura cuando sea posible.

---

## 93. Applied Transformation

Si una transformación ya fue aplicada:

```text
ALREADY_APPLIED
```

en lugar de duplicarla.

---

## 94. Partial Run

Una ejecución interrumpida deberá poder identificar:

```text
completed
pending
failed
unknown
```

---

# Parte XXIII — Checkpoints

## 95. Checkpoint

Antes de pasos relevantes podrá generarse:

```text
MigrationCheckpoint
```

---

## 96. Contents

```text
source revision
schema fingerprint
runtime manifest
transformation journal
validation references
```

---

## 97. Recovery

340 utilizará checkpoints para rollback/recovery.

---

# Parte XXIV — Workspace

## 98. Directory

Por defecto:

```text
.voltstack/database-migration/
```

---

## 99. Structure

```text
.voltstack/database-migration/
├── analysis/
├── model/
├── plans/
├── transformations/
├── overrides/
├── validation/
├── shadow/
├── reports/
├── checkpoints/
└── cache/
```

---

## 100. Git Policy

El proyecto podrá decidir qué artifacts versionar.

---

## 101. Recommended Versioned

Normalmente:

```text
approved migration plan
explicit overrides
behavior contracts
migration configuration
```

---

## 102. Usually Not Versioned

```text
temporary cache
large runtime evidence
ephemeral reports
```

---

# Parte XXV — Progress UX

## 103. Long Operations

Para análisis grandes:

```text
Indexing project       100%
Analyzing Eloquent     100%
Analyzing legacy SQL    74%
Building dependency graph...
```

---

## 104. Non-TTY

En CI no deberá imprimir animaciones incompatibles con logs.

---

## 105. Structured Progress

Internamente:

```text
MigrationProgressEvent
```

---

# Parte XXVI — Output Levels

## 106. Modes

```text
quiet
normal
verbose
debug
```

---

## 107. Quiet

Solo:

```text
critical output
final status
```

---

## 108. Debug

Puede incluir detalles técnicos, respetando redacción.

---

# Parte XXVII — Output Formats

## 109. Human

Default:

```text
console
```

---

## 110. Machine

```text
json
```

---

## 111. CI

Cuando aplique:

```text
JUnit
SARIF
```

mediante exporters.

---

## 112. Contract

El output JSON deberá ser versionado.

---

# Parte XXVIII — Exit Codes

## 113. Stable Exit Codes

Conceptualmente:

```text
0  success
1  validation/migration failure
2  blocked
3  invalid configuration
4  environment protection
5  incomplete/inconclusive
6  internal migration tool failure
```

---

## 114. No Parsing Text

CI deberá depender de:

```text
exit codes + structured output
```

no de regex sobre texto humano.

---

# Parte XXIX — Error UX

## 115. Error Message

Un error deberá contener:

```text
what happened
where
why known
risk
evidence ID
next action
```

---

## 116. Example

```text
[BLOCKED] Cannot switch Billing to VOLTSTACK_WRITE.

Reason:
Transaction behavior has not been verified.

Evidence:
VSDB-MIG-VALIDATION-0178

Required:
Run:
php volt database:migrate:validate \
    --unit=Billing \
    --only=transactions
```

---

## 117. No Generic Failure

Evitar:

```text
Migration failed.
```

sin contexto.

---

# Parte XXX — Stable Diagnostic IDs

## 118. IDs

Todos los errores relevantes deberán utilizar IDs como:

```text
VSDB-MIG-CLI-xxxx
```

o IDs del subsistema origen.

---

## 119. Searchability

El mismo ID deberá aparecer en:

```text
CLI
logs
reports
CI
telemetry
documentation
```

---

# Parte XXXI — Explain System

## 120. Universal Explain

Los subsistemas deberán exponer datos suficientes para:

```bash
--explain
```

---

## 121. Explain Tree

Ejemplo:

```text
Why is Billing blocked?

└── VOLTSTACK_WRITE requires transaction validation
    └── invoice.create contract is unresolved
        └── source uses legacy transaction
            └── target write uses separate connection
```

---

## 122. Evidence Chain

La explicación deberá conservar referencias a evidencia.

---

# Parte XXXII — Suggested Next Action

## 123. Guidance

La CLI podrá sugerir el siguiente comando técnicamente apropiado.

---

## 124. Example

```text
Next:
database:migrate:behavior --verify --unit=Billing
```

---

## 125. No Automatic Mutation

Sugerir no significa ejecutar.

---

# Parte XXXIII — Interactive Wizard

## 126. Optional Wizard

Podrá existir:

```bash
php volt database:migrate
```

---

## 127. Purpose

Guiar a usuarios nuevos:

```text
detect source
analyze
create plan
review blockers
```

---

## 128. Architecture

El wizard deberá reutilizar los mismos application services que los comandos individuales.

---

## 129. Expert Mode

La CLI detallada seguirá disponible.

---

# Parte XXXIV — IDE Integration

## 130. Machine Services

La lógica de CLI deberá poder consumirse desde IDE.

---

## 131. Potential UX

```text
migration diagnostics
code actions
rule explanations
MIM inspection
transformation preview
validation status
```

---

## 132. No CLI Parsing

IDE no deberá ejecutar CLI y parsear texto si existe API estructurada.

---

# Parte XXXV — API Layer

## 133. Application API

Conceptualmente:

```text
MigrationAnalysisService
MigrationPlanningService
MigrationTransformationService
MigrationValidationService
MigrationRuntimeService
MigrationReportingService
MigrationRecoveryService
```

---

## 134. Frontends

Sobre estos servicios podrán existir:

```text
CLI
IDE
CI
future web UI
```

---

# Parte XXXVI — Source-specific DX

## 135. Eloquent

La CLI podrá mostrar:

```text
models
casts
relations
scopes
observers
facade queries
```

---

## 136. Doctrine

Podrá mostrar:

```text
entities
metadata drivers
DQL
custom types
listeners
UnitOfWork risks
```

---

## 137. Legacy

Podrá mostrar:

```text
PDO sites
raw SQL
wrappers
global connections
dynamic SQL
procedures
```

---

## 138. Unified Vocabulary

A pesar de esto, la salida principal deberá usar conceptos neutrales del MIM.

---

# Parte XXXVII — Search and Filter

## 139. Filters

```bash
--unit
--entity
--table
--query
--connection
--risk
--severity
--status
--source
```

---

## 140. Composition

Filtros podrán combinarse de manera determinista.

---

# Parte XXXVIII — Diff UX

## 141. Diff Types

La CLI podrá mostrar:

```text
source code diff
schema diff
MIM diff
runtime manifest diff
validation diff
```

---

## 142. Semantic Diff

Cuando sea posible deberá priorizar:

```text
semantic change
```

sobre ruido textual.

---

## 143. Example

```text
Entity User

- identifier strategy: auto_increment
+ identifier strategy: unchanged

Relationship orders:
- lazy
+ explicit lazy equivalent

No behavioral change detected.
```

---

# Parte XXXIX — Snapshot Comparison

## 144. Analysis Diff

```bash
php volt database:migrate:analyze \
    --diff=<snapshot>
```

---

## 145. Use

Mostrar:

```text
new findings
resolved findings
changed confidence
schema changes
new legacy dependencies
```

---

# Parte XL — Baseline UX

## 146. Baseline

Podrá crearse:

```bash
php volt database:migrate:validate \
    --create-baseline
```

---

## 147. Purpose

Congelar deuda conocida para impedir nueva deuda.

---

## 148. Warning

Baseline no convierte fallos existentes en éxito.

---

# Parte XLI — Suppressions

## 149. Suppression

Una supresión deberá incluir:

```text
finding ID
scope
reason
expiry optional
```

---

## 150. CLI

```bash
php volt database:migrate:status \
    --suppressed
```

---

## 151. Visibility

Los findings suprimidos seguirán visibles en reportes auditables.

---

# Parte XLII — Configuration

## 152. Configuration File

Ejemplo conceptual:

```php
return [
    'migration' => [
        'sources' => [
            'eloquent',
            'legacy',
        ],

        'workspace' => '.voltstack/database-migration',

        'profiles' => [
            'default' => 'standard',
        ],

        'safety' => [
            'protect_production' => true,
        ],
    ],
];
```

---

## 153. Environment Overrides

Variables de entorno podrán cambiar opciones operacionales, pero no deberán almacenar reglas complejas difíciles de auditar.

---

# Parte XLIII — Configuration Validation

## 154. Before Execution

La CLI deberá validar configuración.

---

## 155. Unknown Option

Fallará de forma clara.

---

## 156. Deprecated Option

Se integrará con:

```text
324_DATABASE_DEPRECATION_POLICY.md
```

---

# Parte XLIV — Secrets

## 157. Redaction

Todo renderer deberá usar:

```text
MigrationSecretRedactor
```

---

## 158. Protected Values

```text
password
token
DSN credentials
private keys
sensitive query parameters
```

---

## 159. Debug

`--debug` no deberá desactivar automáticamente redacción.

---

# Parte XLV — Audit Trail

## 160. Mutating Actions

Acciones como:

```text
apply transformation
change runtime state
approve override
create checkpoint
rollback
```

deberán generar eventos auditables.

---

## 161. Audit Entry

```text
action
unit
before state
after state
plan fingerprint
timestamp
actor metadata where available
```

---

# Parte XLVI — Telemetry

## 162. Metrics

```text
database.migration.cli.commands
database.migration.cli.failures
database.migration.cli.duration
database.migration.cli.blocked
database.migration.cli.dry_runs
```

---

## 163. Privacy

No incluir:

```text
source code
SQL text
PII
secrets
```

como labels.

---

# Parte XLVII — Events

## 164. Events

```text
MigrationCommandStarted
MigrationCommandProgress
MigrationCommandCompleted
MigrationCommandFailed
MigrationCommandBlocked
MigrationCommandMutationApplied
```

---

# Parte XLVIII — FrankenPHP

## 165. CLI Separation

La CLI normalmente corre como proceso independiente.

Sin embargo, sus diagnósticos deberán entender el runtime de producción.

---

## 166. Runtime Profile

```bash
php volt database:migrate:doctor \
    --runtime=frankenphp
```

---

## 167. Checks

```text
request reset hooks
persistent services
connection lifecycle
migration context isolation
```

---

# Parte XLIX — CI/CD Workflow

## 168. Example

```text
checkout
   ↓
install
   ↓
migration:doctor
   ↓
migration:analyze
   ↓
migration:validate --profile=ci
   ↓
publish reports
   ↓
gate deployment
```

---

## 169. Pre-cutover Pipeline

```text
migration:status
migration:validate --profile=pre-cutover
migration:runtime --dry-run --state=...
deployment barrier
runtime transition
post-cutover validation
```

---

# Parte L — Team Workflow

## 170. Developer

```text
analyze
plan
dry-run
transform
focused validate
```

---

## 171. Reviewer

```text
inspect diff
inspect rules
inspect overrides
inspect validation
```

---

## 172. CI

```text
reproduce
validate
gate
publish evidence
```

---

## 173. Operator

```text
preflight
cutover
observe
rollback if required
```

---

# Parte LI — Resume Workflow

## 174. Resume

Una migración larga deberá poder reanudarse.

---

## 175. Command

Conceptualmente:

```bash
php volt database:migrate:status \
    --resume-info
```

---

## 176. State

Mostrará:

```text
last completed phase
current checkpoint
pending operations
stale evidence
required next action
```

---

# Parte LII — Migration Session

## 177. Session

Podrá existir:

```text
MigrationSession
```

---

## 178. Purpose

Correlacionar:

```text
analysis
plan
transformation
validation
cutover
```

---

## 179. ID

```text
migration_session_id
```

será útil para reportes, no como estado global mutable.

---

# Parte LIII — Concurrency

## 180. Concurrent Operators

Dos procesos no deberán modificar simultáneamente el mismo migration unit sin coordinación.

---

## 181. Lock

Podrá existir:

```text
MigrationOperationLock
```

---

## 182. Scope

```text
unit
schema
runtime manifest
```

---

## 183. Read-only Commands

Podrán ejecutarse concurrentemente cuando sea seguro.

---

# Parte LIV — Stale Plan Protection

## 184. Plan Fingerprint

Antes de aplicar:

```text
current state
```

deberá compararse con el estado usado para construir el plan.

---

## 185. Mismatch

Si cambió:

```text
STALE_MIGRATION_PLAN
```

---

## 186. Rule

No aplicar silenciosamente un plan calculado sobre código/schema anterior.

---

# Parte LV — Source Control Integration

## 187. Git Awareness

Cuando Git esté disponible, registrar:

```text
commit
branch optional
dirty state
```

---

## 188. Dirty Tree

No necesariamente bloqueará todos los comandos.

---

## 189. Mutation Policy

Para transformaciones masivas podrá recomendar o exigir clean working tree según policy.

---

## 190. No Git Dependency

VoltStack Database no dependerá obligatoriamente de Git.

---

# Parte LVI — Backup Awareness

## 191. Before Destructive Schema/Data Step

La CLI deberá consultar:

```text
recovery requirements
```

---

## 192. Required Backup

Si la policy exige backup y no existe evidencia:

```text
BLOCK
```

---

## 193. No Fake Backup

Un string de configuración no será suficiente evidencia si el recovery contract exige verificación.

---

# Parte LVII — Migration Barriers

## 194. Barrier

La CLI deberá presentar claramente puntos como:

```text
legacy fallback ends after this operation
```

---

## 195. Example

```text
WARNING

Applying this contraction removes column legacy_status.

After this point:
Legacy Billing runtime cannot be restored without data recovery.

Required:
- checkpoint
- backup evidence
- recovery validation
```

---

# Parte LVIII — Help System

## 196. Help

Cada comando tendrá:

```bash
--help
```

---

## 197. Examples

El help deberá incluir ejemplos seguros.

---

## 198. No Secret Examples

No incluir passwords reales en ejemplos.

---

# Parte LIX — Documentation Links

## 199. Diagnostic Documentation

Un finding podrá indicar:

```text
documentation key
```

para que herramientas futuras enlacen documentación.

---

## 200. Offline Usability

La CLI deberá seguir siendo útil sin acceso a Internet.

---

# Parte LX — Localization

## 201. Internal IDs

Los IDs serán independientes del idioma.

---

## 202. Messages

La arquitectura podrá permitir localization futura.

---

## 203. Machine Output

JSON utilizará keys estables, no traducidas.

---

# Parte LXI — Accessibility

## 204. Color

La información crítica no dependerá únicamente de color.

---

## 205. NO_COLOR

Se respetará entorno compatible con:

```text
NO_COLOR
```

cuando el renderer lo soporte.

---

## 206. Plain Output

Debe existir salida legible en terminal simple/CI.

---

# Parte LXII — Performance

## 207. Startup

Comandos simples como:

```text
status
```

no deberán ejecutar automáticamente análisis profundo.

---

## 208. Lazy Loading

Los subsistemas costosos se inicializarán cuando sean necesarios.

---

## 209. Cache

Se reutilizarán:

```text
analysis cache
MIM snapshots
validation evidence
```

si sus fingerprints siguen vigentes.

---

# Parte LXIII — Extension System

## 210. Extension

Paquetes podrán registrar:

```text
commands
renderers
exporters
diagnostic providers
source-specific inspectors
```

---

## 211. Namespace

Extensiones no deberán colisionar con comandos core.

---

## 212. Core Safety

Una extensión no podrá saltarse los guards centrales mediante registro normal.

---

# Parte LXIV — Command Contract

## 213. Interface

Conceptualmente:

```php
interface MigrationCommandHandlerInterface
{
    public function handle(
        MigrationCommandInput $input
    ): MigrationCommandResult;
}
```

---

## 214. Renderer

```php
interface MigrationCommandRendererInterface
{
    public function render(
        MigrationCommandResult $result
    ): void;
}
```

---

## 215. Separation

Esto permite:

```text
same command logic
different output renderer
```

---

# Parte LXV — Result Contract

## 216. Result

```text
MigrationCommandResult
```

---

## 217. Fields

```text
command
status
summary
findings
artifacts
next actions
evidence references
exit code
```

---

# Parte LXVI — Artifact Registry

## 218. Generated Artifacts

La CLI registrará:

```text
analysis snapshots
MIM files
plans
diffs
journals
validation manifests
reports
checkpoints
```

---

## 219. Registry

```text
MigrationArtifactRegistry
```

---

## 220. Purpose

Permitir saber:

```text
which artifact belongs to which migration session/unit/version
```

---

# Parte LXVII — Example Full Workflow

## 221. Step 1

```bash
php volt database:migrate:doctor
```

---

## 222. Step 2

```bash
php volt database:migrate:analyze \
    --profile=deep \
    --snapshot
```

---

## 223. Step 3

```bash
php volt database:migrate:plan \
    --unit=Billing
```

---

## 224. Step 4

```bash
php volt database:migrate:transform \
    --unit=Billing \
    --dry-run
```

---

## 225. Step 5

Revisar diff.

---

## 226. Step 6

```bash
php volt database:migrate:transform \
    --unit=Billing \
    --apply
```

---

## 227. Step 7

```bash
php volt database:migrate:validate \
    --unit=Billing \
    --profile=deep
```

---

## 228. Step 8

```bash
php volt database:migrate:runtime \
    --unit=Billing \
    --state=SHADOW_READ \
    --dry-run
```

---

## 229. Step 9

Activar shadow.

---

## 230. Step 10

```bash
php volt database:migrate:shadow \
    --unit=Billing \
    --differences
```

---

## 231. Step 11

```bash
php volt database:migrate:validate \
    --unit=Billing \
    --profile=pre-cutover
```

---

## 232. Step 12

```bash
php volt database:migrate:runtime \
    --unit=Billing \
    --state=VOLTSTACK_READ
```

---

## 233. Continue

El mismo patrón avanza hasta:

```text
LEGACY_REMOVED
```

---

# Parte LXVIII — Example Blocked Transformation

## 234. Scenario

```text
Doctrine custom type MoneyType
```

sin mapping target confirmado.

---

## 235. Output

```text
[BLOCKED] Invoice.amount

Source:
Doctrine custom type MoneyType

Target:
No approved VoltStack type mapping

Confidence:
UNKNOWN

Required:
Define explicit migration rule or custom type adapter.

No code was modified.
```

---

# Parte LXIX — Example Stale Evidence

## 236. Scenario

Validation pasó ayer.

Hoy cambió repository target.

---

## 237. Output

```text
Validation evidence is stale.

Changed dependency:
BillingInvoiceRepository.php

Affected:
behavior
queries
transactions

Required:
rerun validation for Billing.
```

---

# Parte LXX — Example Production Guard

## 238. Command

```text
schema mutation
```

contra DB marcada como producción.

---

## 239. Output

```text
[BLOCKED]

Protected environment detected.

Operation:
destructive schema migration

Required recovery evidence:
missing

No database changes were made.
```

---

# Parte LXXI — Example Migration Barrier

## 240. Output

```text
Migration Barrier

Unit:
Users

Operation:
Remove legacy discriminator column

Effect:
Legacy ORM will no longer be compatible.

Rollback after this point requires:
database restore or forward recovery.

Proceed only after recovery validation.
```

---

# Parte LXXII — V1 Scope

## 241. Included

Database V1 incluye:

```text
migration CLI namespace
status/analyze/model/plan/rules
transform/schema/behavior
runtime/shadow/validate
doctor/report/rollback integration
dry-run
explain
structured output
stable exit codes
workspace
artifacts
safety guards
CI integration
IDE-ready application services
```

---

## 242. Excluded

No pertenece a V1:

```text
full graphical migration IDE
autonomous AI migration operator
remote fleet migration SaaS
cloud control plane
automatic production cutover without policy
```

Estos podrán evolucionar en V2 o productos superiores.

---

# Parte LXXIII — Component Architecture

## 243. Components

```text
DatabaseMigrationCliAndDeveloperExperienceSystem
│
├── MigrationConsoleApplication
├── MigrationCommandRegistry
├── MigrationCommandDispatcher
├── MigrationStatusService
├── MigrationAnalysisService
├── MigrationModelInspectionService
├── MigrationPlanningService
├── MigrationRuleInspectionService
├── MigrationTransformationService
├── MigrationSchemaService
├── MigrationBehaviorService
├── MigrationRuntimeService
├── MigrationShadowService
├── MigrationValidationService
├── MigrationReportingService
├── MigrationRecoveryService
├── MigrationDoctorService
├── MigrationEnvironmentGuard
├── MigrationOperationLock
├── MigrationArtifactRegistry
├── MigrationProgressReporter
├── MigrationSecretRedactor
├── MigrationExplainService
├── MigrationConsoleRenderer
├── MigrationJsonRenderer
└── MigrationExitCodeMapper
```

---

# Parte LXXIV — Architectural Pipeline

## 244. Pipeline

```text
Developer / CI / Operator
          │
          ▼
   Migration Frontend
          │
          ├── CLI
          ├── IDE
          └── CI
          │
          ▼
 Migration Application API
          │
          ├── Analysis
          ├── Planning
          ├── Transformation
          ├── Validation
          ├── Runtime
          ├── Reporting
          └── Recovery
          │
          ▼
   Migration Core Systems
          │
          ▼
 Structured Result / Evidence
          │
          ▼
 Renderer / Artifact / Exit Code
```

---

# Parte LXXV — Decisiones arquitectónicas

## 245. Decisión 1

La CLI será una capa de presentación, no el lugar de la lógica de migración.

## 246. Decisión 2

Los comandos seguirán el workflow real de migración.

## 247. Decisión 3

Toda mutación relevante será explícita.

## 248. Decisión 4

Dry-run será una capacidad de primera clase.

## 249. Decisión 5

Toda transformación deberá ser explicable.

## 250. Decisión 6

Los resultados serán estructurados antes de renderizarse.

## 251. Decisión 7

CI no dependerá de parsear texto humano.

## 252. Decisión 8

Los planes y evidencia usarán fingerprints contra staleness.

## 253. Decisión 9

Los comandos deberán ser reintentables cuando sea técnicamente seguro.

## 254. Decisión 10

Operaciones concurrentes sobre el mismo migration unit serán coordinadas.

## 255. Decisión 11

La producción tendrá guards reforzados.

## 256. Decisión 12

`--debug` nunca implicará desactivar redacción de secretos.

## 257. Decisión 13

El sistema deberá indicar por qué una transición está bloqueada.

## 258. Decisión 14

El CLI expondrá el MIM y evidencia, no solo resultados finales.

## 259. Decisión 15

El workspace de migración tendrá estructura estable.

## 260. Decisión 16

La arquitectura será reutilizable por IDE y futuras interfaces.

## 261. Decisión 17

El CLI entenderá persistent-worker readiness de FrankenPHP.

## 262. Decisión 18

Los migration barriers irreversibles deberán mostrarse explícitamente antes de ejecutarse.

---

# Parte LXXVI — UX Principles

## 263. Principle 1

```text
Preview before mutation.
```

---

## 264. Principle 2

```text
Explain before forcing.
```

---

## 265. Principle 3

```text
Show evidence, not confidence theater.
```

---

## 266. Principle 4

```text
A blocked migration should explain the path to unblock it.
```

---

## 267. Principle 5

```text
Safe defaults must remain safe in CI and production.
```

---

# Parte LXXVII — Completion Criteria

## 268. CLI/DX Complete

La arquitectura V1 de CLI/DX estará completa cuando permita:

```text
discover migration state
analyze source systems
inspect MIM
inspect migration rules
build plans
preview transformations
apply controlled transformations
inspect schema compatibility
verify behavior
manage dual runtime
operate shadow comparison
execute validation profiles
explain blockers
produce machine-readable evidence
protect destructive operations
prepare reporting
invoke recovery workflows
```

---

# Parte LXXVIII — Integración con 339

## 269. Reporting

El siguiente documento formalizará:

```text
migration reports
diagnostic aggregation
trend analysis
risk summaries
coverage reports
cutover reports
executive/technical views
machine-readable exports
```

---

# Parte LXXIX — Integración con 340

## 270. Recovery

El CLI definido aquí expondrá posteriormente:

```bash
php volt database:migrate:rollback
```

pero la semántica, condiciones, estrategias y límites del rollback pertenecen a:

```text
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

# Parte LXXX — Resultado esperado

## 271. Antes

Una migración compleja suele depender de:

```text
manual scripts
tribal knowledge
ad-hoc commands
untracked decisions
implicit state
```

---

## 272. Después

```text
VoltStack Migration CLI
│
├── Status
├── Analysis
├── Intermediate Model
├── Plan
├── Rules
├── Transformation
├── Schema
├── Behavior
├── Runtime
├── Shadow
├── Validation
├── Reporting
└── Recovery
```

con:

```text
dry-run
explainability
evidence
auditability
safety
repeatability
```

---

# Parte LXXXI — Principio final

## 273. Regla

```text
A migration tool should never make the developer
guess what it discovered,
guess what it will change,
guess why it is blocked,
or guess whether the evidence is still valid.
```

---

# Parte LXXXII — Conclusión

## 274. Arquitectura final

`DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE` convierte los subsistemas de migración de VoltStack en un workflow utilizable, inspeccionable y automatizable.

La experiencia completa queda:

```text
                    Developer
                        │
                        ▼
              database:migrate:*
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
    Inspect           Preview          Execute
       │                │                │
       └────────────────┼────────────────┘
                        ▼
               Migration Services
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
    Evidence          Guards           Journals
       │                │                │
       └────────────────┼────────────────┘
                        ▼
                Validation Gates
                        │
                 ┌──────┴──────┐
                 ▼             ▼
              Advance        Block
                 │             │
                 ▼             ▼
             Observe       Explain Why
```

La relación final con el pipeline de migración es:

```text
325–329 understand the source.

330 creates the neutral semantic model.

331 decides migration rules.

332 transforms code.

333 validates schema compatibility.

334 verifies behavior.

335 controls dual runtime.

336 compares real shadow reads.

337 validates the migration globally.

338 makes the entire system operable by developers, CI and operators.

339 will consolidate reporting and diagnostics.

340 will close V1 with rollback and recovery.
```

La regla arquitectónica definitiva será:

```text
Make every migration step visible.

Make every mutation intentional.

Make every blocker explainable.

Make every important decision traceable.
```

---

**Documento:** `338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
