# 331_DATABASE_MIGRATION_RULE_ENGINE.md

## 1. Propósito

Este documento define la arquitectura oficial del **Database Migration Rule Engine** de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseMigrationRuleEngine
```

y es responsable de convertir el conocimiento obtenido durante el análisis de migración en **decisiones de transformación explícitas, deterministas, trazables y verificables**.

El Rule Engine opera principalmente sobre el:

```text
Migration Intermediate Model (MIM)
```

definido en:

```text
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
```

y nunca deberá depender directamente de APIs de:

```text
Laravel Eloquent
Doctrine ORM
Doctrine DBAL
PDO
mysqli
legacy ORMs
```

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
```

y alimentará directamente:

```text
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

## 3. Problema

Después del análisis, VoltStack puede saber que existe:

```text
Entity User
Table users
Field email
Relationship User → Posts
Soft Delete
Custom Money Type
Transaction Checkout
Raw SQL Query
```

pero aún falta decidir:

```text
How should each concept migrate?
```

Una transformación no puede depender de una colección desordenada de:

```text
if source == eloquent
if type == doctrine
if legacy...
```

dispersa por todo el sistema.

---

## 4. Solución

Centralizar las decisiones de transformación mediante:

```text
Migration Rules
```

evaluadas por un motor común:

```text
MIM
 │
 ▼
Rule Matching
 │
 ▼
Eligibility
 │
 ▼
Conflict Resolution
 │
 ▼
Migration Decision
 │
 ▼
Transformation Plan
```

---

## 5. Principio fundamental

```text
Analysis discovers facts.

Rules decide transformations.

Transformers execute changes.

Validators prove results.
```

Estas responsabilidades deberán permanecer separadas.

---

## 6. Responsabilidades

El Rule Engine deberá:

```text
register rules
discover applicable rules
evaluate preconditions
match MIM objects
evaluate target capabilities
evaluate source/target versions
consider migration policy
detect rule conflicts
resolve deterministic precedence
produce migration decisions
attach required validators
attach rollback metadata
record provenance
explain every selected rule
```

---

## 7. No responsabilidades

El Rule Engine no deberá:

```text
rewrite PHP source
modify files
modify schema
execute SQL migrations
perform cutover
silently resolve semantic ambiguity
```

---

## 8. Arquitectura general

```text
Migration Intermediate Model
            │
            ▼
      Rule Candidate Finder
            │
            ▼
      Preconditions Engine
            │
            ▼
      Capability Evaluator
            │
            ▼
       Conflict Detector
            │
            ▼
      Rule Selection Engine
            │
            ▼
      Migration Decisions
            │
            ▼
     Transformation Plan
            │
            ▼
       Code Transformer
```

---

## 9. Regla de migración

Conceptualmente:

```php
interface MigrationRuleInterface
{
    public function id(): MigrationRuleId;

    public function supports(
        MigrationRuleContext $context,
        MigrationModelNode $node
    ): bool;

    public function evaluate(
        MigrationRuleContext $context,
        MigrationModelNode $node
    ): MigrationRuleEvaluation;

    public function plan(
        MigrationRuleContext $context,
        MigrationModelNode $node
    ): MigrationRuleDecision;
}
```

---

## 10. Stable Rule IDs

Toda regla oficial tendrá ID estable.

Ejemplos:

```text
VSDB-MIG-CORE-0001
VSDB-MIG-CORE-0002

VSDB-MIG-ELOQ-0001
VSDB-MIG-DOC-0001
VSDB-MIG-LEGACY-0001
```

---

## 11. Regla neutral vs regla source-specific

Las reglas source-specific podrán existir en adapters.

Ejemplo:

```text
Eloquent $timestamps
```

se normaliza primero a:

```text
AUTOMATIC_TIMESTAMPS
```

Después, la regla core trabaja con:

```text
AUTOMATIC_TIMESTAMPS
```

y no con `$timestamps`.

---

## 12. Boundary Rule

El núcleo del Rule Engine nunca deberá importar:

```text
Illuminate\Database\*
Doctrine\ORM\*
Doctrine\DBAL\*
```

---

## 13. Rule Metadata

Cada regla deberá declarar:

```text
id
name
description
category
version
introduced_in
deprecated_in
removed_in
risk
automation level
supported target versions
required evidence
required validators
```

---

## 14. Categorías

Las reglas podrán clasificarse como:

```text
CONNECTION
ENTITY
FIELD
TYPE
IDENTIFIER
RELATIONSHIP
QUERY
REPOSITORY
TRANSACTION
LIFECYCLE
BEHAVIOR
SCHEMA
PROCEDURE
RUNTIME
SECURITY
COMPATIBILITY
```

---

## 15. Rule Context

```text
MigrationRuleContext
```

contendrá:

```text
MIM version
target VoltStack version
database platform
migration policy
analysis snapshot
schema capabilities
runtime profile
project overrides
selected migration unit
```

---

## 16. Rule Candidate Finder

Primero se seleccionarán reglas potencialmente aplicables según:

```text
node type
category
target version
platform
capabilities
```

antes de evaluar lógica costosa.

---

## 17. Rule Index

El registry deberá mantener índices por:

```text
node class
category
platform
target version
feature
```

para evitar recorrer todas las reglas para cada nodo.

---

## 18. Preconditions

Una regla podrá exigir:

```text
known source semantics
known target capability
minimum confidence
schema evidence
resolved conflict
specific platform
validator availability
```

---

## 19. Preconditions Example

```text
Rule:
Convert DECIMAL field

Requires:
type = DECIMAL
precision known
scale known
target supports decimal
schema conflict = none
```

---

## 20. Failed Preconditions

Una regla que no cumpla precondiciones no deberá ejecutarse.

Podrá producir:

```text
NOT_APPLICABLE
INELIGIBLE
BLOCKED
```

según causa.

---

## 21. Rule Evaluation Result

```text
MigrationRuleEvaluation
```

podrá ser:

```text
APPLICABLE
NOT_APPLICABLE
BLOCKED
REQUIRES_REVIEW
CONFLICT
```

---

## 22. Automation Levels

Cada regla declarará:

```text
NONE
ASSISTED
SAFE
CONDITIONAL
```

---

## 23. NONE

La herramienta solo explica el caso.

No genera transformación.

---

## 24. ASSISTED

Puede generar:

```text
suggestion
template
patch preview
```

pero requiere aprobación/revisión.

---

## 25. SAFE

Puede automatizarse cuando todas sus precondiciones y validators se cumplen.

---

## 26. CONDITIONAL

Puede automatizarse únicamente bajo condiciones adicionales.

Ejemplo:

```text
schema matches
tests exist
no custom accessor
```

---

## 27. Migration Decision

El resultado principal será:

```text
MigrationDecision
```

---

## 28. Decision Types

```text
PRESERVE
TRANSFORM
ADAPT
REPLACE
DEFER
MANUAL
BLOCK
IGNORE_WITH_REASON
```

---

## 29. PRESERVE

Ejemplo:

```text
Raw PostgreSQL reporting query
```

puede conservarse como Native SQL bajo VoltStack.

---

## 30. TRANSFORM

Ejemplo:

```text
simple entity field mapping
```

se convierte a metadata nativa.

---

## 31. ADAPT

Ejemplo:

```text
legacy repository interface
```

puede mantenerse temporalmente sobre implementación VoltStack.

---

## 32. REPLACE

Ejemplo:

```text
global PDO singleton
```

debe sustituirse por `ConnectionManager`.

---

## 33. DEFER

Se pospone una transformación hasta que se resuelva una dependencia.

---

## 34. MANUAL

No existe transformación suficientemente segura.

---

## 35. BLOCK

El elemento impide continuar una fase.

---

## 36. IGNORE_WITH_REASON

Un componente puede quedar fuera del alcance actual, pero debe existir justificación explícita.

---

## 37. Decision Model

Conceptualmente:

```php
final readonly class MigrationDecision
{
    public function nodeId(): string;

    public function ruleId(): string;

    public function type(): MigrationDecisionType;

    public function rationale(): string;

    public function risk(): MigrationRisk;

    public function requiredValidations(): array;

    public function dependencies(): array;

    public function rollbackStrategy(): ?MigrationRollbackDescriptor;
}
```

---

## 38. Explainability

Toda decisión deberá responder:

```text
What rule selected this?
Why?
What evidence was used?
What assumptions exist?
What risks remain?
How will it be verified?
```

---

## 39. No Opaque Rules

No se aceptarán decisiones del tipo:

```text
"Auto migrated because engine decided so."
```

---

## 40. Rule Provenance

La decisión deberá conservar:

```text
rule ID
rule version
MIM node ID
analysis snapshot ID
target version
configuration fingerprint
```

---

## 41. Rule Registry

Componente:

```text
MigrationRuleRegistry
```

---

## 42. Registry Responsibilities

```text
register
validate uniqueness
index
filter by version
filter by platform
resolve deprecated rules
provide metadata
```

---

## 43. Rule Registration

Podrá realizarse mediante:

```text
package manifests
service provider
container registration
compiled registry
```

según la arquitectura final de VoltStack.

---

## 44. Compile-time Registry

Para rendimiento, el registry podrá compilarse a una estructura optimizada.

---

## 45. Duplicate Rule IDs

Dos reglas con el mismo ID deberán provocar error de configuración.

---

## 46. Rule Version

Una regla podrá evolucionar:

```text
VSDB-MIG-CORE-0042 v1
VSDB-MIG-CORE-0042 v2
```

sin cambiar necesariamente su identidad conceptual.

---

## 47. Semantic Rule Changes

Si cambia radicalmente la intención, deberá utilizarse un nuevo Rule ID.

---

## 48. Rule Set

Una ejecución utilizará un:

```text
MigrationRuleSet
```

que representa el conjunto exacto de reglas habilitadas.

---

## 49. Rule Set Fingerprint

Se generará:

```text
rule_set_fingerprint
```

para reproducibilidad.

---

## 50. Reproducibility

Mismo:

```text
MIM snapshot
target version
rule set
configuration
platform capabilities
```

deberá producir el mismo plan.

---

## 51. Rule Ordering

Las reglas no deberán depender accidentalmente del orden de registro.

---

## 52. Priority

Cuando sea necesario, una regla podrá declarar prioridad explícita.

Ejemplo:

```text
SECURITY_OVERRIDE
PROJECT_OVERRIDE
SPECIFIC
GENERAL
FALLBACK
```

---

## 53. Suggested Precedence

```text
1. Safety constraints
2. Explicit verified project decision
3. Platform-specific rule
4. Feature-specific rule
5. General core rule
6. Fallback/manual rule
```

---

## 54. Priority Is Not Silent Override

Si dos reglas producen decisiones incompatibles:

```text
Conflict Detector
```

deberá evaluarlo aunque una tenga mayor prioridad.

---

## 55. Rule Conflict

Ejemplo:

```text
Rule A:
TRANSFORM relationship

Rule B:
MANUAL because polymorphic discriminator is dynamic
```

El resultado no deberá ser decidido únicamente por orden.

---

## 56. Conflict Types

```text
DECISION_CONFLICT
TARGET_CONFLICT
SCHEMA_CONFLICT
VALIDATION_CONFLICT
DEPENDENCY_CONFLICT
POLICY_CONFLICT
```

---

## 57. Conflict Resolution

Podrá resolverse mediante:

```text
more specific verified rule
explicit developer override
additional evidence
manual resolution
```

---

## 58. No Last-write-wins

Regla estricta:

```text
Migration rules do not use last-write-wins semantics.
```

---

## 59. Rule Dependencies

Una regla podrá depender de otra.

Ejemplo:

```text
Relationship migration
depends on
Entity migration
depends on
Identifier migration
```

---

## 60. Dependency Graph

El motor construirá:

```text
MigrationRuleDependencyGraph
```

---

## 61. Topological Planning

Las reglas podrán ordenarse por dependencias:

```text
Connection
  ↓
Type
  ↓
Entity
  ↓
Identifier
  ↓
Relationship
  ↓
Repository
```

---

## 62. Circular Dependencies

Un ciclo deberá detectarse antes de transformación.

---

## 63. Rule Cycle Example

```text
Rule A depends on B
Rule B depends on C
Rule C depends on A
```

Resultado:

```text
RULE_DEPENDENCY_CYCLE
```

---

## 64. Target Capability Model

El Rule Engine deberá consultar:

```text
MigrationTargetCapabilities
```

---

## 65. Capability Examples

```text
supports composite IDs
supports native enums
supports JSON
supports generated columns
supports savepoints
supports optimistic locking
supports custom types
supports native SQL
```

---

## 66. Platform Capability

También deberá considerar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
SQL Server
```

cuando la transformación dependa de la plataforma.

---

## 67. Framework Capability

La capacidad target estará vinculada a:

```text
VoltStack Database V1
```

y no a features futuras.

---

## 68. V1 Scope Guard

Una regla no podrá seleccionar:

```text
future V2 capability
```

para declarar una migración V1 automática.

---

## 69. Unsupported-but-Preservable

Si VoltStack V1 no abstrae una feature pero puede preservarla mediante Native SQL:

```text
PRESERVE
```

puede ser correcto.

---

## 70. Policy Engine

El Rule Engine consultará:

```text
MigrationPolicy
```

---

## 71. Policy Examples

```text
prefer_native_sql
prefer_query_builder
prefer_repository
require_schema_match
strict_transactions
strict_security
allow_compatibility_adapters
allow_manual_items
```

---

## 72. Policy Does Not Rewrite Facts

Una policy puede cambiar una decisión, pero no el MIM.

---

## 73. Strict Mode

Ejemplo:

```text
minimum confidence = HIGH
```

para reglas automáticas críticas.

---

## 74. Assisted Mode

Puede aceptar:

```text
MEDIUM confidence
```

para generar propuesta no aplicada.

---

## 75. Confidence Thresholds

Cada categoría podrá tener umbral distinto.

Ejemplo:

```text
field rename       MEDIUM
identifier         HIGH
transaction        HIGH
encryption         HIGH
```

---

## 76. Risk Model

Cada regla deberá declarar riesgo base.

---

## 77. Risk Categories

```text
DATA_INTEGRITY
TRANSACTION
SECURITY
BEHAVIOR
PERFORMANCE
RUNTIME
PORTABILITY
MAINTAINABILITY
```

---

## 78. Risk Levels

```text
LOW
MEDIUM
HIGH
CRITICAL
```

---

## 79. Effective Risk

El riesgo efectivo podrá aumentar por:

```text
low confidence
missing tests
schema conflict
shared writes
dynamic behavior
persistent runtime issues
```

---

## 80. Effective Risk Formula

Conceptualmente:

```text
base rule risk
+
context modifiers
=
effective migration risk
```

No deberá reducirse a una fórmula numérica opaca.

---

## 81. Rule Conditions

Las reglas deberán usar condiciones declarativas cuando sea práctico.

Ejemplo conceptual:

```text
node.type == FIELD
field.type == DECIMAL
field.precision != UNKNOWN
field.scale != UNKNOWN
```

---

## 82. Declarative Rule Metadata

La selección simple podrá indexarse sin instanciar lógica compleja.

---

## 83. Imperative Evaluation

Casos complejos podrán usar PHP:

```php
public function evaluate(...): MigrationRuleEvaluation
{
    // semantic evaluation
}
```

---

## 84. Separation

```text
Declarative metadata
```

sirve para selección rápida.

```text
Imperative evaluator
```

sirve para semántica compleja.

---

## 85. Example: Simple String Field

MIM:

```text
Field:
name

Type:
STRING

Length:
255

Nullable:
false
```

Rule:

```text
VSDB-MIG-CORE-FIELD-STRING
```

Decision:

```text
TRANSFORM
```

Target:

```text
VoltStack string field metadata
```

---

## 86. Example: Decimal

MIM:

```text
amount
DECIMAL(18,4)
```

Rule:

```text
VSDB-MIG-CORE-FIELD-DECIMAL
```

Preconditions:

```text
precision known
scale known
no unresolved schema conflict
```

Validation:

```text
schema verify
round-trip value tests
```

---

## 87. Example: Unknown Decimal

```text
Doctrine:
12,2

Schema:
18,4
```

Decision:

```text
BLOCK
```

until conflict resolution.

---

## 88. Example: Soft Delete

MIM:

```text
Behavior:
SOFT_DELETE

column:
deleted_at
```

Decision:

```text
TRANSFORM
```

hacia el mecanismo nativo de soft delete de VoltStack, si existe y conserva semántica.

---

## 89. Soft Delete Validation

Deberá verificar:

```text
default filtering
with deleted behavior
restore
force delete
relationships
```

---

## 90. Example: Eloquent Scope

El adapter Eloquent normaliza:

```text
scopeActive()
```

a una intención de query/repository.

El Rule Engine decide si:

```text
TRANSFORM → repository method
PRESERVE → query helper
MANUAL → contains domain behavior
```

---

## 91. Example: Doctrine Custom Type

MIM:

```text
CUSTOM_TYPE:
money
```

El Rule Engine evalúa:

```text
native VoltStack type exists?
custom type adapter possible?
conversion understood?
tests available?
```

---

## 92. Custom Type Decisions

Posibles:

```text
TRANSFORM
ADAPT
MANUAL
BLOCK
```

---

## 93. Example: Raw SQL

MIM:

```text
RAW SQL
platform:
PostgreSQL

feature:
RETURNING
```

Si VoltStack Native SQL puede ejecutarlo sin alterar semántica:

```text
PRESERVE
```

---

## 94. Example: Global PDO

MIM:

```text
Behavior:
GLOBAL_CONNECTION_STATE
```

Decision:

```text
REPLACE
```

con:

```text
VoltStack ConnectionManager
```

---

## 95. Runtime Validation

La regla anterior deberá requerir:

```text
FrankenPHP request isolation test
transaction cleanup test
connection state reset test
```

---

## 96. Example: Composite ID

Si V1 soporta la semántica necesaria:

```text
TRANSFORM
```

Si el source usa un generador custom no comprendido:

```text
MANUAL/BLOCK
```

---

## 97. Example: Stored Procedure

MIM:

```text
Procedure:
calculate_invoice_totals
```

Por defecto:

```text
PRESERVE
```

y migrar el caller.

---

## 98. Stored Procedure Replacement

`REPLACE` solo deberá elegirse si existe una decisión explícita y pruebas suficientes.

---

## 99. Example: Database Trigger

Un trigger que actualiza `updated_at` puede producir conflicto con automatic timestamps.

El Rule Engine deberá detectar:

```text
DUPLICATE_BEHAVIOR_RISK
```

antes de habilitar ambos.

---

## 100. Behavior Conflict

```text
Target automatic timestamp
+
existing database trigger
```

puede resultar en:

```text
PRESERVE trigger + disable target behavior
```

o:

```text
remove trigger + enable target behavior
```

pero nunca ambos silenciosamente.

---

## 101. Rule Composition

Una transformación compleja podrá componerse de varias reglas.

Ejemplo:

```text
Entity Rule
├── Field Rules
├── Identifier Rule
├── Relationship Rules
└── Behavior Rules
```

---

## 102. Atomic Decision Group

Ciertas decisiones deberán agruparse.

Ejemplo:

```text
inheritance hierarchy
```

no debe migrarse clase por clase si la estrategia requiere consistencia global.

---

## 103. Migration Decision Group

```text
MigrationDecisionGroup
```

podrá representar:

```text
aggregate
inheritance hierarchy
transaction group
shared table group
```

---

## 104. All-or-nothing Group

Un grupo podrá declarar:

```text
ATOMIC_TRANSFORMATION
```

cuando aplicar solo parte sea inválido.

---

## 105. Migration Plan

El resultado global será:

```text
MigrationTransformationPlan
```

---

## 106. Plan Contents

```text
analysis snapshot
MIM fingerprint
rule set fingerprint
target version
decisions
decision groups
dependencies
manual items
blockers
validators
rollback descriptors
```

---

## 107. Plan Status

```text
DRAFT
READY_FOR_REVIEW
APPROVED
BLOCKED
EXECUTABLE
EXECUTED
VERIFIED
```

---

## 108. Plan Immutability

Un plan aprobado deberá ser inmutable.

Cambios de source o rules deberán generar un nuevo plan.

---

## 109. Stale Plan Detection

Antes de ejecución:

```text
current MIM fingerprint
==
plan MIM fingerprint
```

deberá verificarse.

---

## 110. Rule Set Drift

También:

```text
current rule set fingerprint
==
plan rule set fingerprint
```

---

## 111. Configuration Drift

La configuración relevante deberá fingerprintarse.

---

## 112. Stale Plan

Si existe drift:

```text
PLAN_STALE
```

y se requerirá reanálisis o replanificación.

---

## 113. Dry Run

El Rule Engine soportará:

```text
evaluate without transformation
```

por diseño.

---

## 114. Explain Mode

CLI conceptual:

```bash
php volt database:migrate:rules \
    --explain=field:payment.amount
```

---

## 115. Explain Output

```text
Node:
field:payment.amount

Selected rule:
VSDB-MIG-CORE-DECIMAL-001

Decision:
TRANSFORM

Evidence:
Doctrine metadata
Actual schema

Confidence:
HIGH

Required validation:
Schema compatibility
Decimal round-trip
```

---

## 116. Rule Trace

Para diagnóstico podrá mostrarse:

```text
candidate rules
rejected rules
selected rule
conflicts
```

---

## 117. Rejected Rule Reason

Ejemplo:

```text
Rule X rejected:
requires scale to be known.
```

---

## 118. CLI Rule Listing

```bash
php volt database:migrate:rules
```

podrá listar reglas disponibles.

---

## 119. Filter

```bash
php volt database:migrate:rules \
    --category=transaction
```

---

## 120. Rule Inspection

```bash
php volt database:migrate:rule \
    VSDB-MIG-CORE-0042
```

---

## 121. Plan Generation

Conceptualmente:

```bash
php volt database:migrate:plan
```

utilizará Analysis + MIM + Rule Engine.

---

## 122. Plan Review

El plan deberá mostrar:

```text
automatic
assisted
manual
blocked
preserved
```

por separado.

---

## 123. No Automatic Apply

Generar un plan no equivale a ejecutarlo.

---

## 124. Approval Boundary

La ejecución automática podrá requerir:

```text
explicit --apply
```

o equivalente en la DX final.

---

## 125. Project Rules

Un proyecto podrá registrar reglas propias.

---

## 126. Project Rule Namespace

IDs recomendados:

```text
PROJECT-MIG-...
```

para distinguirlas de reglas oficiales.

---

## 127. Project Rule Restrictions

No podrán:

```text
override safety silently
change facts in MIM
remove evidence
hide blockers
```

---

## 128. Explicit Overrides

Para resolver un caso conocido podrá existir:

```text
MigrationOverride
```

con:

```text
target node
decision
reason
author/source
validation requirements
```

---

## 129. Override Audit

Todo override deberá aparecer en:

```text
plan
reports
diagnostics
snapshot metadata
```

---

## 130. Security Overrides

Una policy podrá impedir overrides de:

```text
critical SQL injection
credential exposure
data corruption risk
```

sin un mecanismo explícito de aceptación de riesgo.

---

## 131. Rule Deprecation

Las reglas siguen:

```text
324_DATABASE_DEPRECATION_POLICY.md
```

cuando formen parte de una API/tooling estable.

---

## 132. Deprecated Rule

Una regla antigua podrá:

```text
remain for reproducibility
not be selected for new plans
```

durante una ventana definida.

---

## 133. Rule Replacement

Metadata:

```text
deprecated:
true

replacement:
VSDB-MIG-CORE-0091
```

---

## 134. Historical Reproducibility

Si un snapshot antiguo necesita una regla antigua, el sistema podrá cargar un rule set compatible.

---

## 135. Rule Packages

Arquitectura posible:

```text
Database-Migration-Core
Database-Migration-Eloquent
Database-Migration-Doctrine
Database-Migration-Legacy
```

---

## 136. Adapter Rules

Los paquetes source-specific podrán incluir reglas de **normalización**, pero las transformaciones target deberán preferir reglas core sobre MIM.

---

## 137. Third-party Rule Packs

Podrán existir:

```text
Migration Rule Packs
```

para ORMs o plataformas adicionales.

---

## 138. Rule Pack Manifest

Deberá declarar:

```text
package
version
MIM compatibility
VoltStack compatibility
rules
dependencies
```

---

## 139. Rule Pack Trust

VoltStack no deberá asumir que reglas de terceros son seguras.

Podrán requerir:

```text
explicit enable
signature/package trust policy
review
```

según la distribución final.

---

## 140. Rule Sandbox

Las reglas PHP ejecutan código dentro del proceso de tooling.

Por tanto, solo deberán cargarse desde paquetes confiables.

---

## 141. No Dynamic Remote Rules

V1 no deberá descargar y ejecutar reglas arbitrarias desde Internet durante una migración.

---

## 142. Deterministic Inputs

Una regla no deberá depender directamente de:

```text
current time
randomness
network calls
mutable global state
```

para decidir transformaciones.

---

## 143. External Data

Si una decisión requiere información externa, esta deberá incorporarse previamente como evidencia/snapshot.

---

## 144. Idempotence

Evaluar una regla varias veces sobre el mismo contexto deberá producir la misma decisión.

---

## 145. Transformation Idempotence

El transformer asociado deberá intentar ser idempotente o detectar que el cambio ya existe.

---

## 146. Rollback Descriptor

Cada transformación automática deberá declarar, cuando sea posible:

```text
how to revert source changes
how to restore config
whether schema rollback is required
whether data rollback is required
```

---

## 147. Non-reversible Rule

Una regla no reversible deberá declararlo explícitamente.

---

## 148. Destructive Rule

Las transformaciones destructivas deberán:

```text
never be SAFE by default
```

---

## 149. Data-changing Rule

Cualquier regla que cambie datos deberá requerir validación y estrategia de rollback reforzada.

---

## 150. Schema-changing Rule

El Rule Engine podrá planearla, pero la ejecución deberá pasar por el sistema normal de migrations/schema de VoltStack.

---

## 151. Code-only Rule

Ejemplo:

```text
replace PDO factory with ConnectionManager injection
```

puede no requerir schema change.

---

## 152. Config Rule

Ejemplo:

```text
legacy DSN config
→
VoltStack connection config
```

deberá preservar secret source.

---

## 153. Repository Rule

Podrá decidir:

```text
preserve repository contract
replace implementation
```

para minimizar cambios de aplicación.

---

## 154. Query Rule

Podrá elegir entre:

```text
Native SQL
Query Builder
Repository method
Query Object
```

según semántica y policy.

---

## 155. No ORM Bias

El Rule Engine no deberá considerar:

```text
ORM conversion
```

como objetivo universal.

---

## 156. SQL-first Workloads

Reporting, analytics, ETL y bulk operations podrán permanecer SQL-first.

---

## 157. Transaction Rule

Las reglas de transacción deberán considerar:

```text
connection ownership
isolation
savepoints
retry
nested semantics
cross-source participants
```

---

## 158. Transaction Safety

Una regla no podrá declarar `SAFE` si la atomicidad target no está demostrada.

---

## 159. Lifecycle Rule

Deberá considerar:

```text
timing
transaction relation
side effects
ordering
```

---

## 160. Event Timing

`after save` no es suficiente como equivalencia.

Debe diferenciarse:

```text
after SQL
after flush
before commit
after commit
```

---

## 161. Persistent Runtime Rule

Podrá generar decisiones como:

```text
REPLACE static connection
ADD reset hook
CLEAR persistence context
ROLLBACK unfinished transaction
```

---

## 162. FrankenPHP

El target runtime predeterminado deberá activar validaciones de:

```text
request isolation
state reset
connection cleanup
tenant cleanup
```

---

## 163. RoadRunner/OpenSwoole

Los mismos conceptos se aplicarán mediante perfiles oficiales.

---

## 164. Multitenancy

Si el paquete opcional está instalado, podrá aportar:

```text
tenant-aware rules
```

sin convertir Multitenancy en dependencia del core.

---

## 165. SaaS

El paquete SaaS podrá aportar reglas adicionales, pero seguirá desacoplado de Migration Core.

---

## 166. Security Rule Example

MIM detecta:

```text
dynamic SQL identifier
untrusted source
```

Rule:

```text
BLOCK
```

hasta introducir allow-list o resolver seguro.

---

## 167. Binding Rule

Una query con valores concatenados podrá recibir:

```text
REPLACE
```

hacia parameter binding cuando la semántica sea inequívoca.

---

## 168. Identifier Binding

Los identificadores SQL:

```text
table
column
order direction
```

no deberán tratarse como parámetros normales.

Requerirán validación/allow-list.

---

## 169. Authentication-sensitive Rule

Migraciones sobre:

```text
password hashes
remember tokens
MFA secrets
session IDs
```

deberán tener políticas conservadoras.

---

## 170. Encryption Rule

Nunca deberá cambiarse automáticamente el formato de cifrado sin:

```text
explicit migration plan
data validation
rollback
key availability
```

---

## 171. Testing Requirements

Cada regla oficial deberá incluir:

```text
positive fixtures
negative fixtures
boundary cases
conflict cases
idempotence tests
determinism tests
```

---

## 172. Rule Contract Tests

Toda regla deberá demostrar:

```text
stable ID
valid metadata
deterministic evaluation
declared validators
valid decision type
```

---

## 173. Cross-platform Tests

Las reglas platform-aware deberán probarse en plataformas soportadas relevantes.

---

## 174. MIM Version Tests

Una regla deberá declarar qué versiones MIM comprende.

---

## 175. Target Version Tests

También deberá declarar versiones target compatibles.

---

## 176. Conflict Tests

Se probarán casos donde dos reglas compitan para asegurar que no exista selección silenciosa.

---

## 177. Performance

Grandes proyectos pueden contener:

```text
100k+ MIM nodes
```

El Rule Engine deberá evitar:

```text
all-rules × all-nodes
```

cuando sea posible.

---

## 178. Candidate Indexing

La selección deberá aproximarse a:

```text
node
→ category/type index
→ small candidate set
→ evaluation
```

---

## 179. Parallel Evaluation

Nodos independientes podrán evaluarse concurrentemente si:

```text
rules are pure/deterministic
results merge deterministically
no global mutable state
```

---

## 180. Shared Decision Groups

Los grupos dependientes deberán evaluarse coordinadamente.

---

## 181. Evaluation Cache

Podrá cachearse:

```text
node fingerprint
rule version
context fingerprint
→ evaluation result
```

---

## 182. Cache Invalidation

Cambios en:

```text
MIM node
rule
policy
target version
platform capability
```

invalidarán el resultado.

---

## 183. Telemetry

Métricas posibles:

```text
database.migration.rules.evaluated
database.migration.rules.selected
database.migration.rules.conflicts
database.migration.rules.blocked
database.migration.rules.manual
database.migration.rules.cache_hits
database.migration.plan.duration
```

---

## 184. Logging

Canal:

```text
database.migration.rules
```

sin datos sensibles.

---

## 185. Rule Diagnostics

Un error interno de regla deberá reportar:

```text
rule ID
rule version
node ID
safe context
stack trace in development
```

---

## 186. Failure Isolation

Si una regla falla inesperadamente:

```text
RULE_EXECUTION_ERROR
```

deberá bloquear ese nodo/grupo, no producir una transformación parcial silenciosa.

---

## 187. Error Classes

Conceptualmente:

```text
MigrationRuleException
MigrationRuleConflictException
MigrationRuleDependencyException
MigrationRuleConfigurationException
MigrationRuleExecutionException
MigrationPlanStaleException
```

---

## 188. Validation Attachment

Cada decisión podrá adjuntar:

```text
MigrationValidationRequirement
```

---

## 189. Validation Types

```text
STATIC
SCHEMA
QUERY_EQUIVALENCE
BEHAVIOR
TRANSACTION
DATA
PERFORMANCE
RUNTIME_ISOLATION
SECURITY
MANUAL
```

---

## 190. Validation Gate

Una decisión transformada no podrá considerarse:

```text
VERIFIED
```

hasta satisfacer sus validaciones requeridas.

---

## 191. Code Transformer Contract

El documento 332 recibirá un plan como:

```text
MigrationTransformationPlan
```

y no volverá a decidir qué debe migrarse.

---

## 192. Important Boundary

```text
Rule Engine:
WHAT should happen.

Code Transformer:
HOW source files are changed.
```

---

## 193. Schema Compatibility Boundary

El documento 333 puede devolver:

```text
schema verified
schema conflict
schema change required
```

y el Rule Engine podrá replanificar.

---

## 194. Behavior Verification Boundary

El documento 334 verifica que la decisión conserve semántica observable.

---

## 195. Dual Runtime Boundary

El documento 335 ejecutará decisiones `ADAPT/PRESERVE` durante coexistencia.

---

## 196. Shadow Query Boundary

El documento 336 validará queries críticas seleccionadas por reglas.

---

## 197. Testing Boundary

El documento 337 implementará suites derivadas de los requisitos de validación.

---

## 198. CLI Boundary

El documento 338 permitirá:

```text
inspect rules
explain decisions
approve plan
view blockers
```

---

## 199. Reporting Boundary

El documento 339 mostrará:

```text
rule coverage
decision distribution
manual items
conflicts
validation status
```

---

## 200. Rollback Boundary

El documento 340 utilizará los rollback descriptors del plan.

---

## 201. Componentes principales

```text
DatabaseMigrationRuleEngine
│
├── MigrationRuleRegistry
├── MigrationRuleSetResolver
├── MigrationRuleCandidateFinder
├── MigrationPreconditionEvaluator
├── MigrationTargetCapabilityResolver
├── MigrationPolicyResolver
├── MigrationRiskEvaluator
├── MigrationRuleConflictDetector
├── MigrationRuleDependencyGraph
├── MigrationRuleSelector
├── MigrationDecisionFactory
├── MigrationDecisionGroupBuilder
├── MigrationValidationRequirementBuilder
├── MigrationRollbackDescriptorBuilder
├── MigrationPlanBuilder
├── MigrationRuleTrace
└── MigrationRuleEvaluationCache
```

---

## 202. Flujo completo

```text
MIM Snapshot
     │
     ▼
Select Migration Unit
     │
     ▼
Find Candidate Rules
     │
     ▼
Evaluate Preconditions
     │
     ▼
Evaluate Capabilities
     │
     ▼
Evaluate Policy
     │
     ▼
Evaluate Risk
     │
     ▼
Detect Conflicts
     │
     ▼
Resolve Dependencies
     │
     ▼
Select Decisions
     │
     ▼
Attach Validators
     │
     ▼
Attach Rollback Metadata
     │
     ▼
Build Transformation Plan
```

---

## 203. Ejemplo de plan

```text
Migration Plan: Users

Entity User
  TRANSFORM
  VSDB-MIG-CORE-ENTITY-001

Field User.id
  TRANSFORM
  VSDB-MIG-CORE-ID-002

Field User.email
  TRANSFORM
  VSDB-MIG-CORE-FIELD-STRING

Behavior SoftDelete
  TRANSFORM
  VSDB-MIG-CORE-BEHAVIOR-004

Legacy UserRepository
  ADAPT
  VSDB-MIG-CORE-REPOSITORY-012

Custom encrypted_token
  MANUAL
  VSDB-MIG-CORE-SECURITY-019

Validation
  Schema
  CRUD behavior
  Soft delete behavior
  Runtime isolation
```

---

## 204. Plan Summary

La herramienta deberá diferenciar:

```text
Automatic transformations
Assisted transformations
Preserved components
Compatibility adapters
Manual items
Blocked items
```

---

## 205. No Misleading Success

Un plan con:

```text
2 blocked
```

no deberá reportarse como:

```text
Migration ready
```

sin aclarar el alcance bloqueado.

---

## 206. Architectural Decisions

### Decisión 1

Toda transformación Database V1 será seleccionada mediante reglas explícitas.

### Decisión 2

El Rule Engine operará sobre el MIM, no directamente sobre APIs externas.

### Decisión 3

Analysis, decision, transformation y verification serán fases independientes.

### Decisión 4

Todas las reglas oficiales tendrán IDs estables y metadata versionada.

### Decisión 5

El conjunto exacto de reglas será fingerprinted para reproducibilidad.

### Decisión 6

Las reglas serán deterministas y no dependerán de red, tiempo o randomness.

### Decisión 7

Los conflictos nunca utilizarán last-write-wins.

### Decisión 8

Compatibilidad, confianza, riesgo y automation level serán dimensiones distintas.

### Decisión 9

Una regla automática requerirá ruta de validación.

### Decisión 10

Las transformaciones destructivas no serán `SAFE` por defecto.

### Decisión 11

La atomicidad transaccional tendrá prioridad sobre conveniencia de transformación.

### Decisión 12

SQL nativo podrá preservarse cuando sea la representación adecuada.

### Decisión 13

El motor no tendrá sesgo obligatorio hacia ORM.

### Decisión 14

Los overrides serán explícitos, trazables y auditables.

### Decisión 15

Los planes aprobados serán inmutables y detectarán drift.

### Decisión 16

Las reglas podrán organizarse en paquetes, manteniendo el core independiente de Eloquent/Doctrine.

### Decisión 17

Database V1 solo planeará usando capacidades realmente disponibles en V1.

### Decisión 18

Cada transformación automática deberá declarar validación y, cuando corresponda, rollback.

---

## 207. Resultado esperado

Antes del Rule Engine:

```text
MIM
├── facts
├── evidence
├── conflicts
├── queries
├── entities
├── transactions
└── behaviors
```

Después:

```text
Migration Transformation Plan
├── PRESERVE
├── TRANSFORM
├── ADAPT
├── REPLACE
├── DEFER
├── MANUAL
├── BLOCK
│
├── Dependencies
├── Validation Requirements
├── Risk
├── Rule Provenance
└── Rollback Metadata
```

---

## 208. Principio final

```text
No transformation is implicit.

Every transformation is a decision.

Every decision comes from a rule.

Every rule is traceable.

Every automatic rule is verifiable.
```

---

## 209. Conclusión

`DATABASE_MIGRATION_RULE_ENGINE` convierte el conocimiento neutral del Migration Intermediate Model en un plan concreto de migración hacia VoltStack.

La arquitectura completa queda:

```text
Source Systems
      │
      ▼
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
Transformation Plan
      │
      ▼
Code Transformer
      │
      ▼
Verification
```

Con esta separación, VoltStack evita que la lógica de migración termine distribuida entre adapters, generadores de código y comandos CLI.

La regla arquitectónica definitiva será:

```text
Adapters understand the source.

The MIM describes the semantics.

Rules decide the migration.

Transformers execute the plan.

Validators prove the result.
```

---

**Documento:** `331_DATABASE_MIGRATION_RULE_ENGINE.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
