# 109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Diff Migration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 109 — Database Schema Diff Migration System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Diff Migration System` define la arquitectura mediante la cual VoltStack transforma diferencias estructurales detectadas entre dos representaciones de esquema en **candidatos de migración tipados, revisables, deterministas y verificables**.

El sistema conecta:

```text
Schema Architecture
        │
        ▼
Schema Diff
        │
        ▼
Migration Architecture
```

sin romper la separación entre ambos subsistemas.

Su responsabilidad fundamental es responder:

> **Dado un esquema actual y un esquema objetivo, ¿qué transición de migración podría llevar de uno al otro preservando intención, dependencias, compatibilidad, seguridad y evidencia suficiente?**

La palabra importante es:

```text
could
```

y no:

```text
must
```

porque:

```text
Schema Difference
≠
Automatically Executable Migration
```

---

# 2. Principio central

> **Schema Diff descubre diferencias; Schema Diff Migration interpreta esas diferencias como candidatos de evolución; Migration Planner decide cómo gobernarlas; Execution Engine realiza los efectos.**

Por tanto:

```text
Schema Diff
≠
Migration
```

y:

```text
Schema Diff
≠
DDL Generator
```

---

# 3. Separaciones fundamentales

VoltStack deberá preservar:

```text
Schema Snapshot
≠
Schema Diff

Schema Diff
≠
Migration Candidate

Migration Candidate
≠
Migration Definition

Migration Definition
≠
Migration Plan

Difference
≠
Migration Operation

Rename Candidate
≠
Proven Rename

Structural Change
≠
Data Transformation

Generated Migration
≠
Safe Migration

Generated Migration
≠
Approved Migration

Generated Migration
≠
Executed Migration

Schema Compatibility
≠
Migration Safety

Schema Equality
≠
Data Equality

Target Schema
≠
Desired Business State

Diff-to-Migration
≠
Direct DDL Execution
```

---

# 4. Relación con el Schema Diff System

El documento:

```text
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
```

estableció:

```text
Diff(Current, Target)
```

y produce diferencias estructurales tipadas.

Ejemplo:

```text
Current
users.email varchar(255)

Target
users.email varchar(320)
```

Schema Diff puede producir:

```text
ColumnModifiedDifference
```

El presente sistema deberá interpretar:

```text
ColumnModifiedDifference
        ↓
Potential AlterColumn Migration Operation
```

sin asumir todavía que dicha operación:

```text
is safe
is directly supported
is zero-downtime
preserves all data
```

---

# 5. Posición arquitectónica

```text
Current Schema
      │
      │
      ├───────────────┐
      │               │
      ▼               ▼
Schema Snapshot    Target Schema
      │               │
      └───────┬───────┘
              ▼
         Schema Diff
              │
              ▼
      SchemaDifferenceSet
              │
              ▼
┌───────────────────────────────┐
│ Schema Diff Migration System  │
│                               │
│ Difference Interpretation     │
│ Change Classification         │
│ Rename Resolution             │
│ Dependency Synthesis          │
│ Operation Synthesis           │
│ Data Requirement Detection    │
│ Compatibility Analysis        │
│ Candidate Validation          │
└──────────────┬────────────────┘
               ▼
      MigrationCandidate
               │
       ┌───────┴─────────┐
       ▼                 ▼
 Human Review        Policy Review
       │                 │
       └────────┬────────┘
                ▼
       MigrationDefinition
                │
                ▼
        Migration Planner
                │
                ▼
         Safety Analysis
                │
                ▼
        Migration Execution
```

---

# 6. Flujo maestro

```text
Current Schema
      +
Target Schema
      ↓
Schema Diff
      ↓
Difference Normalization
      ↓
Difference Interpretation
      ↓
Rename Evidence Analysis
      ↓
Migration Change Graph
      ↓
Operation Synthesis
      ↓
Data Requirement Analysis
      ↓
Compatibility Analysis
      ↓
Candidate Safety Classification
      ↓
Migration Candidate
      ↓
Review / Acceptance
      ↓
Migration Definition
```

---

# 7. Entrada principal

El sistema deberá consumir:

```php
final readonly class SchemaDiffMigrationRequest
{
    public function __construct(
        public SchemaSnapshot $current,
        public SchemaSnapshot $target,
        public SchemaDiff $diff,
        public DatabasePlatformContext $platform,
        public SchemaDiffMigrationPolicy $policy,
    ) {}
}
```

---

# 8. No hidden introspection

El sistema no deberá hacer:

```text
getCurrentConnection()
→ introspect database
```

ocultamente.

Toda evidencia deberá proporcionarse explícitamente.

---

# 9. Snapshot coverage

Especial atención a:

```text
COMPLETE
PARTIAL
UNKNOWN
```

del Schema System.

Regla:

```text
MissingFromPartialSnapshot
≠
RemovedObject
```

---

# 10. Destructive inference

Si:

```text
CurrentSnapshot.coverage = PARTIAL
```

y un objeto no aparece en `Target`:

VoltStack no deberá generar automáticamente:

```text
DROP object
```

sin evidencia suficiente.

---

# 11. Resultado principal

Se propone:

```php
final readonly class SchemaDiffMigrationAnalysis
{
    public function __construct(
        public SchemaDiffMigrationCandidateSet $candidates,
        public SchemaDiffMigrationDiagnosticSet $diagnostics,
        public SchemaDiffMigrationCompleteness $completeness,
        public SchemaDiffMigrationFingerprint $fingerprint,
    ) {}
}
```

---

# 12. Completeness

Estados:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 13. COMPLETE

Significa:

> Todas las diferencias relevantes pudieron interpretarse suficientemente.

No significa:

```text
migration is safe
```

---

# 14. PARTIAL

Ejemplo:

```text
12 differences
10 translated
2 require manual interpretation
```

---

# 15. UNKNOWN

Cuando evidencia crítica es insuficiente para determinar una transición confiable.

---

# 16. Migration Candidate

El artefacto central será:

```text
SchemaDiffMigrationCandidate
```

---

# 17. Definición

Un `SchemaDiffMigrationCandidate` representa:

> Una propuesta inmutable de transición de migración derivada de un conjunto específico de diferencias de esquema y de evidencia explícita.

---

# 18. Candidate ≠ MigrationDefinition

El candidato puede contener incertidumbre:

```text
possible rename
data migration required
manual review required
platform emulation required
unknown destructive impact
```

Una `MigrationDefinition` aceptada no debería esconder esas incertidumbres.

---

# 19. Modelo

```php
final readonly class SchemaDiffMigrationCandidate
{
    public function __construct(
        public SchemaDiffMigrationCandidateId $id,
        public SchemaDiffSourceDescriptor $source,
        public SchemaChangeGraph $changes,
        public MigrationOperationProposalSet $operations,
        public MigrationDataRequirementSet $dataRequirements,
        public SchemaDiffMigrationCompatibilityReport $compatibility,
        public SchemaDiffMigrationReviewRequirementSet $reviewRequirements,
        public SchemaDiffMigrationCandidateFingerprint $fingerprint,
    ) {}
}
```

---

# 20. Candidate ID

```text
SchemaDiffMigrationCandidateId
```

es distinto de:

```text
MigrationId
MigrationExecutionId
SchemaDiffId
SchemaObjectId
```

---

# 21. Candidate acceptance

Al aceptar un candidato:

```text
Candidate
   ↓
MigrationDefinitionFactory
   ↓
MigrationDefinition
```

Se asigna:

```text
MigrationId
```

explícito.

---

# 22. Difference taxonomy

El sistema deberá interpretar diferencias como:

```text
OBJECT_ADDED
OBJECT_REMOVED
OBJECT_MODIFIED
OBJECT_RENAMED
OBJECT_MOVED
UNKNOWN
INCOMPARABLE
PLATFORM_DEPENDENT
```

---

# 23. Difference ≠ operation

Ejemplo:

```text
ColumnRemovedDifference
```

puede traducirse en:

```text
DropColumnProposal
```

pero también requerir:

```text
BackupRequirement
DataArchiveRequirement
ApplicationBarrier
ManualReview
```

---

# 24. Difference interpreter

Se propone:

```php
interface SchemaDifferenceMigrationInterpreter
{
    public function supports(SchemaDifference $difference): bool;

    public function interpret(
        SchemaDifference $difference,
        SchemaDiffMigrationContext $context
    ): SchemaChangeInterpretation;
}
```

---

# 25. Interpreter registry

```text
SchemaDifferenceMigrationInterpreterRegistry
```

deberá ser:

```text
typed
deterministic
frozen
ordered by explicit priority/dependency
```

---

# 26. No last-wins

Dos interpreters para el mismo semantic domain no deberán sobrescribirse silenciosamente.

---

# 27. Change interpretation

Se propone:

```php
final readonly class SchemaChangeInterpretation
{
    public function __construct(
        public SchemaDifferenceId $difference,
        public SchemaChangeIntent $intent,
        public SchemaChangeImpact $impact,
        public MigrationOperationProposalSet $operations,
        public MigrationRequirementSet $requirements,
        public SchemaChangeEvidenceSet $evidence,
    ) {}
}
```

---

# 28. Change intent

Puede ser:

```text
CREATE
REMOVE
ALTER
RENAME
MOVE
REBUILD
REPLACE
VALIDATE
UNKNOWN
```

---

# 29. Change impact

Debe distinguir:

```text
STRUCTURAL
REPRESENTATIONAL
DATA
INTEGRITY
PERFORMANCE
APPLICATION_CONTRACT
OPERATIONAL
UNKNOWN
```

---

# 30. Change severity

Separada de impact:

```text
INFO
LOW
MEDIUM
HIGH
CRITICAL
UNKNOWN
```

---

# 31. Rename problem

Uno de los problemas más peligrosos es:

```text
Current:
users.name

Target:
users.full_name
```

Podría ser:

```text
RENAME name → full_name
```

o:

```text
DROP name
ADD full_name
```

---

# 32. Semántica diferente

```text
Rename
```

puede preservar datos.

Mientras:

```text
Drop + Add
```

puede destruirlos.

Por tanto:

```text
Rename Detection
```

es una decisión de alta importancia.

---

# 33. Rename evidence

Se propone:

```text
SchemaRenameEvidence
```

---

# 34. Evidence levels

```text
EXPLICIT
STRONG
PROBABLE
WEAK
NONE
CONFLICTING
```

---

# 35. Evidence sources

En orden conceptual:

```text
explicit developer mapping
stable schema identity
migration metadata
structural identity
semantic metadata
name similarity
position similarity
heuristics
```

---

# 36. Explicit rename

Ideal:

```php
$diff->mapRename(
    from: 'users.name',
    to: 'users.full_name'
);
```

o mediante metadata declarativa.

---

# 37. Heuristic rename

Ejemplo:

```text
name
→
full_name
```

con mismo:

```text
type
nullability
default
position
constraints
```

puede producir:

```text
RenameCandidate
```

pero no:

```text
ProvenRename
```

---

# 38. Rename confidence

Se propone:

```text
CONFIRMED
HIGH_CONFIDENCE
AMBIGUOUS
LOW_CONFIDENCE
REJECTED
```

---

# 39. Política recomendada

Solo:

```text
CONFIRMED
```

podrá transformarse automáticamente en `RenameOperationProposal` bajo modo estricto.

---

# 40. High confidence

Puede generar:

```text
RenameReviewRequired
```

---

# 41. Ambiguous rename

Debe permanecer:

```text
UNRESOLVED
```

hasta intervención explícita.

---

# 42. No arbitrary threshold

No debe existir:

```php
if ($similarity > 0.70) {
    rename();
}
```

como regla suficiente.

---

# 43. Rename map

Se propone:

```text
SchemaRenameResolutionMap
```

con:

```text
old path
new path
evidence
confidence
resolution source
```

---

# 44. Table rename

Mismas reglas:

```text
users_archive
→
archived_users
```

no es rename probado por nombre parecido.

---

# 45. Constraint rename

Los nombres pueden ser generados por plataforma.

Por tanto:

```text
constraint physical name changed
```

no siempre implica semantic change.

---

# 46. Index rename

Igualmente:

```text
IndexIdentity
```

debe analizar estructura y metadata.

---

# 47. Schema Change Graph

Después de interpretar diferencias:

```text
SchemaChangeGraph
```

representará relaciones entre cambios.

---

# 48. Ejemplo

```text
CreateTable orders
      │
      ├── AddColumn customer_id
      │
      ├── AddPrimaryKey
      │
      └── AddForeignKey customer_id → customers.id
```

---

# 49. Graph node

```php
final readonly class SchemaChangeNode
{
    public function __construct(
        public SchemaChangeNodeId $id,
        public SchemaChangeIntent $intent,
        public SchemaDifferenceSet $sourceDifferences,
        public SchemaChangeRequirementSet $requirements,
    ) {}
}
```

---

# 50. Graph edge

Puede expresar:

```text
REQUIRES
MUST_PRECEDE
MUST_FOLLOW
CONFLICTS_WITH
VALIDATES
DATA_DEPENDS_ON
```

---

# 51. Difference dependency ≠ migration dependency

El change graph representa dependencias internas de una transición.

No necesariamente:

```text
Migration A depends on Migration B
```

---

# 52. Operation synthesis

El sistema convertirá:

```text
SchemaChangeGraph
```

en:

```text
MigrationOperationProposalSet
```

---

# 53. Proposal ≠ operation accepted

```text
MigrationOperationProposal
```

puede incluir:

```text
confidence
requirements
alternatives
risks
source differences
```

---

# 54. Operation proposal taxonomy

```text
SchemaOperationProposal
DataOperationProposal
ValidationOperationProposal
BarrierOperationProposal
BackupOperationProposal
ManualOperationProposal
ExtensionOperationProposal
```

---

# 55. Schema proposal

Ejemplos:

```text
CreateTableProposal
DropTableProposal
RenameTableProposal
AddColumnProposal
AlterColumnProposal
RenameColumnProposal
DropColumnProposal
CreateIndexProposal
DropIndexProposal
AddConstraintProposal
DropConstraintProposal
```

---

# 56. Data operation proposal

Schema Diff no conoce necesariamente los datos.

Pero puede detectar:

```text
DataMigrationRequirement
```

---

# 57. Ejemplo NOT NULL

Current:

```text
users.phone
does not exist
```

Target:

```text
users.phone VARCHAR NOT NULL
```

La diferencia estructural es:

```text
ADD NOT NULL COLUMN
```

Pero para una tabla con filas existentes puede requerirse:

```text
backfill
```

---

# 58. Correct candidate

No asumir:

```text
ALTER TABLE users ADD phone VARCHAR NOT NULL
```

como transición completa.

Puede proponer:

```text
1 ADD phone nullable
2 BACKFILL REQUIRED
3 VALIDATE phone IS NOT NULL
4 SET NOT NULL
```

---

# 59. Data requirement ≠ data migration implementation

El sistema puede detectar:

```text
BackfillRequired
```

sin saber:

```text
how to derive phone
```

---

# 60. Manual data transformation

Entonces:

```text
MigrationCandidateCompleteness = PARTIAL
```

hasta que el desarrollador suministre transformación.

---

# 61. Type conversion

Current:

```text
VARCHAR
```

Target:

```text
INTEGER
```

requiere analizar:

```text
physical representability
data conversion
lossiness
validation
platform syntax
```

---

# 62. Type conversion classes

Se propone:

```text
LOSSLESS
CONDITIONALLY_LOSSLESS
LOSSY
UNSUPPORTED
UNKNOWN
```

---

# 63. Lossless structural conversion

Ejemplo posible:

```text
VARCHAR(100)
→
VARCHAR(200)
```

pero aun deberá analizar:

```text
platform behavior
locking
index limits
```

---

# 64. Narrowing conversion

```text
VARCHAR(500)
→
VARCHAR(100)
```

debe producir:

```text
DataValidationRequirement
```

---

# 65. Numeric narrowing

```text
BIGINT
→
INT
```

requiere range validation.

---

# 66. Nullability change

```text
NULLABLE
→
NOT NULL
```

requiere comprobar:

```text
no null rows
```

o realizar backfill.

---

# 67. Default change

Cambiar default afecta principalmente:

```text
future writes
```

No asumir modificación de filas existentes.

---

# 68. Constraint addition

Agregar:

```text
UNIQUE(email)
```

requiere potencialmente:

```text
duplicate validation
```

---

# 69. Foreign key addition

Puede requerir:

```text
orphan validation
type compatibility
referenced key validation
index/platform requirements
```

---

# 70. CHECK addition

Puede requerir:

```text
existing row validation
```

según plataforma y enforcement semantics.

---

# 71. Destructive changes

Se consideran al menos:

```text
DROP TABLE
DROP COLUMN
DROP CONSTRAINT with integrity impact
type narrowing
destructive generated-column change
identity strategy replacement
data-losing table rebuild
```

---

# 72. Destructive classification

```text
NON_DESTRUCTIVE
POTENTIALLY_DESTRUCTIVE
DESTRUCTIVE
UNKNOWN
```

---

# 73. Destructive candidate

Debe contener:

```text
explicit destructive marker
affected object
reason
potential data impact
backup recommendation/requirement
review requirement
```

---

# 74. No hidden destructive operation

VoltStack nunca deberá convertir:

```text
RemovedColumnDifference
```

en `DropColumn` ejecutable sin conservar la clasificación destructiva.

---

# 75. Human review

Se propone:

```text
MigrationReviewRequirement
```

---

# 76. Review kinds

```text
RENAME_CONFIRMATION
DATA_TRANSFORMATION_REQUIRED
DESTRUCTIVE_CHANGE_APPROVAL
PLATFORM_EMULATION_REVIEW
ZERO_DOWNTIME_REVIEW
UNKNOWN_METADATA_REVIEW
RAW_EXPRESSION_REVIEW
MANUAL_OPERATION_REQUIRED
```

---

# 77. Review state

```text
NOT_REQUIRED
REQUIRED
SATISFIED
REJECTED
UNKNOWN
```

---

# 78. Candidate acceptance gate

Formalmente:

```text
AcceptableCandidate
=
NoBlockingUnknowns
∧
RequiredReviewsSatisfied
∧
RequiredTransformationsProvided
∧
CompatibilityAcceptable
```

Safety completa será analizada después por el Migration Safety System.

---

# 79. Compatibility integration

Debe utilizar:

```text
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md
```

---

# 80. Final state compatibility

Pregunta:

> ¿Puede el target schema existir en la plataforma?

---

# 81. Transition compatibility

Pregunta distinta:

> ¿Puede realizarse esta transición en la plataforma?

---

# 82. Ejemplo SQLite

Target:

```text
column removed
```

puede ser representable.

Pero cierta versión/capability podría requerir:

```text
table rebuild
```

---

# 83. Emulation candidate

Schema Compatibility puede producir:

```text
REQUIRES_EMULATION
```

pero:

```text
Compatibility Emulation Candidate
≠
Selected Migration Strategy
```

---

# 84. Strategy ownership

El Diff Migration System puede elevar:

```text
MigrationStrategyRequirement
```

El Migration Planner seleccionará estrategia final.

---

# 85. Platform-specific migration candidate

Se puede producir:

```text
PlatformSpecificRequirement
```

cuando una transición no sea portable.

---

# 86. Portable candidate

No significa:

```text
same SQL everywhere
```

Significa:

```text
same required semantics can be represented
```

---

# 87. MariaDB

Debe analizarse como plataforma first-class.

Nunca:

```text
MariaDB = MySQL alias
```

---

# 88. Capability snapshot

Toda compatibilidad debe utilizar:

```text
immutable explicit capability snapshot
```

---

# 89. Version ≠ capability

No:

```php
if ($version >= 'x') {
    assumeFeature();
}
```

como contrato central.

---

# 90. Platform drift

Si el candidate fue generado para:

```text
CapabilityFingerprint C1
```

pero después:

```text
C2
```

deberá revalidarse antes de aceptación/planificación.

---

# 91. Schema Diff Migration Policy

Se propone:

```php
final readonly class SchemaDiffMigrationPolicy
{
    public function __construct(
        public RenameInferencePolicy $renamePolicy,
        public DestructiveChangePolicy $destructivePolicy,
        public PortabilityProfile $portability,
        public ManualReviewPolicy $review,
        public CandidateGenerationMode $mode,
    ) {}
}
```

---

# 92. Generation modes

```text
STRICT
ASSISTED
EXPLORATORY
```

---

# 93. STRICT

Solo genera operaciones cuando la evidencia es suficientemente fuerte.

---

# 94. ASSISTED

Puede generar proposals con:

```text
review required
```

---

# 95. EXPLORATORY

Puede mostrar alternativas sin permitir aceptación directa.

---

# 96. Example rename modes

```text
STRICT:
ambiguous rename → unresolved

ASSISTED:
ambiguous rename → rename proposal + review

EXPLORATORY:
show rename and drop/add alternatives
```

---

# 97. Alternative strategies

Un cambio puede tener varias propuestas.

Ejemplo:

```text
Rename column
```

alternativas:

```text
A native rename
B add-copy-validate-drop
C dual-column expand/contract
```

---

# 98. Candidate alternatives

Se propone:

```text
MigrationStrategyAlternativeSet
```

---

# 99. Diff system no elige siempre

Cuando alternativas tienen implicaciones operacionales importantes:

```text
Migration Planner
```

o:

```text
Zero-Downtime Migration System
```

deberá decidir.

---

# 100. Dependency synthesis

El sistema debe derivar dependencias internas.

Ejemplo:

```text
Create customers
↓
Create orders
↓
Add orders.customer_id FK
```

---

# 101. Foreign key cycle

```text
A → B
B → A
```

puede requerir:

```text
create A
create B
add FK A→B
add FK B→A
```

---

# 102. No arbitrary ordering

Nunca depender únicamente de:

```text
diff iteration order
```

---

# 103. Deterministic ordering

Para cambios independientes:

```text
canonical identifier order
+
stable change kind order
```

puede servir como tie-breaker.

---

# 104. Operation identity

Cada proposal tendrá:

```text
MigrationOperationProposalId
```

estable dentro del candidate.

---

# 105. Proposal source mapping

Debe poder rastrearse:

```text
MigrationOperationProposal
        ↓
SchemaChangeNode
        ↓
SchemaDifference
        ↓
Current/Target Schema Objects
```

---

# 106. Explainability

Ejemplo:

```text
Operation proposal:
RenameColumn users.name → users.full_name

Reason:
Explicit rename mapping supplied.

Source differences:
- users.name removed
- users.full_name added

Confidence:
CONFIRMED

Data impact:
NONE EXPECTED

Review:
NOT REQUIRED
```

---

# 107. Ambiguous diagnostic

```text
Potential rename detected:

users.name
→ users.full_name

Evidence:
- same VARCHAR(255)
- same nullability
- same ordinal position

Missing evidence:
- explicit rename mapping

Result:
MANUAL RENAME CONFIRMATION REQUIRED
```

---

# 108. Data requirement model

Se propone:

```php
interface MigrationDataRequirement
{
    public function id(): MigrationDataRequirementId;

    public function kind(): MigrationDataRequirementKind;
}
```

---

# 109. Data requirement kinds

```text
BACKFILL
VALIDATE_RANGE
VALIDATE_NULLABILITY
VALIDATE_UNIQUENESS
VALIDATE_REFERENTIAL_INTEGRITY
TRANSFORM_DATA
ARCHIVE_DATA
BACKUP_DATA
COPY_DATA
MANUAL_DATA_REVIEW
```

---

# 110. Requirement resolution

Puede estar:

```text
UNRESOLVED
PROVIDED
VALIDATED
NOT_REQUIRED
```

---

# 111. Backfill specification

Si el desarrollador completa el candidate:

```php
$candidate->resolve(
    $requirement,
    BackfillDefinition::using(
        // typed Query/Expression definition
    )
);
```

---

# 112. No arbitrary callback runtime state

La resolución deberá producir una definición serializable/determinista cuando sea posible.

---

# 113. Data query boundary

Data validation y backfills usarán:

```text
Query Engine
```

no SQL concatenado desde Schema Diff.

---

# 114. Row data not in Schema Diff

Schema Diff Migration System no debe leer todas las filas durante análisis puro.

Puede generar:

```text
RuntimeValidationRequirement
```

para preflight/execution.

---

# 115. Analysis purity

Ideal:

```text
Analyze(diff, snapshots, capabilities, policy)
```

es pure/deterministic.

---

# 116. Runtime validation

Separado:

```text
Migration Candidate
      ↓
Validation Plan
      ↓
Execution-time/read-only Query Checks
```

---

# 117. Schema-only confidence

Debe poder decir:

```text
structural candidate complete
data safety unknown until runtime validation
```

---

# 118. Schema Diff Migration Candidate completeness

Puede ser:

```text
COMPLETE
COMPLETE_WITH_RUNTIME_VALIDATION
PARTIAL
MANUAL_INTERVENTION_REQUIRED
UNKNOWN
```

---

# 119. Generated migration

Una vez resueltos los requirements:

```text
SchemaDiffMigrationCandidate
        ↓
MigrationDefinitionGenerator
        ↓
MigrationDefinition
```

---

# 120. Generated definition

Debe utilizar exactamente los mismos contratos del documento:

```text
102_DATABASE_MIGRATION_SYSTEM.md
```

No crear un segundo migration format.

---

# 121. Generated migrations are normal migrations

Después de generada:

```text
GeneratedMigrationDefinition
```

debe comportarse como cualquier:

```text
MigrationDefinition
```

---

# 122. Provenance

Sin embargo puede conservar metadata:

```text
generatedFromSchemaDiff
schemaDiffFingerprint
currentSchemaFingerprint
targetSchemaFingerprint
generatorVersion
policyFingerprint
```

---

# 123. Generated source

Si se materializa como archivo:

```text
migration PHP source
```

eso es una representación del definition.

No su identidad semántica.

---

# 124. Deterministic generation

Misma entrada:

```text
Current
Target
Diff
Capabilities
Policy
GeneratorVersion
```

debe producir el mismo canonical candidate.

---

# 125. Candidate fingerprint

Conceptualmente:

```text
F =
Hash(
    CurrentSchemaFingerprint
    + TargetSchemaFingerprint
    + SchemaDiffFingerprint
    + CapabilityFingerprint
    + PolicyFingerprint
    + GeneratorVersion
    + ExtensionRegistryFingerprint
)
```

---

# 126. Fingerprint ≠ identity proof

Hash collision sigue siendo conceptualmente posible.

Comparación estructural puede ser requerida.

---

# 127. Generated Migration fingerprint

Después de aceptar:

```text
MigrationDefinitionFingerprint
```

será calculado por Migration System.

No reutilizar ciegamente candidate fingerprint.

---

# 128. Round-trip verification

Una propiedad clave:

```text
Current
  ↓
Generated Migration
  ↓
Interpret Schema Operations
  ↓
Predicted Result
  ↓
Compare Target
```

---

# 129. Fórmula

Ideal:

```text
Interpret(Current, GeneratedSchemaOperations)
≈
Target
```

según comparison profile.

---

# 130. Round-trip failure

Si no se cumple:

```text
candidate invalid
```

---

# 131. SchemaAstInterpreter

Puede utilizarse:

```text
SchemaAstInterpreter
```

para simular cambios sin DB I/O.

---

# 132. Structural simulation

Pipeline:

```text
Current Schema Model
      ↓
Candidate Schema Operations
      ↓
Schema AST
      ↓
SchemaAstInterpreter
      ↓
Predicted Schema
      ↓
Schema Diff(Predicted, Target)
```

---

# 133. Residual diff

Debe ser:

```text
empty
```

o contener únicamente diferencias explícitamente aceptadas por el profile.

---

# 134. Data operations in round-trip

No pueden verificarse completamente mediante Schema AST.

Deben conservar:

```text
DataPostcondition
```

separadas.

---

# 135. Schema equality ≠ migration completeness

Ejemplo:

```text
Predicted Schema = Target
```

pero un backfill necesario no está definido.

Candidate:

```text
PARTIAL
```

---

# 136. Drift handling

Existen dos momentos:

```text
generation drift
execution drift
```

---

# 137. Generation drift

Current snapshot ya no representa la DB cuando se genera candidate.

---

# 138. Execution drift

Candidate fue generado correctamente, pero DB cambió antes de execution.

---

# 139. Fingerprint precondition

Candidate puede declarar:

```text
ExpectedCurrentSchemaFingerprint
```

---

# 140. Before planning/execution

Debe verificarse cuando policy lo requiera.

---

# 141. Fingerprint mismatch

Produce:

```text
SchemaDiffMigrationSourceDriftException
```

o re-diff/replan.

---

# 142. No stale destructive migration

Especialmente:

```text
DROP
ALTER destructive
```

no deberán continuar con source drift desconocido.

---

# 143. Migration preconditions

Candidate puede generar:

```text
SchemaPrecondition
```

Ejemplo:

```text
users.legacy_email exists
and has expected type
```

antes de alterarlo.

---

# 144. Postconditions

Ejemplo:

```text
users.email exists
users.email NOT NULL
unique constraint present
```

---

# 145. Preconditions ≠ drift repair

Si fallan:

```text
stop/replan
```

por default.

---

# 146. Destructive source validation

Antes de:

```text
DROP COLUMN
```

deberá verificarse que el objeto observado coincide con el esperado cuando sea posible.

---

# 147. Schema Diff candidate splitting

Un diff grande puede producir:

```text
one migration
```

o:

```text
multiple migrations
```

---

# 148. Split strategy

Puede considerar:

```text
dependency boundaries
risk boundaries
zero-downtime phases
package/module ownership
manual data transformations
transaction compatibility
```

---

# 149. Candidate group

Se propone:

```text
SchemaDiffMigrationCandidateGroup
```

---

# 150. Candidate group ≠ batch

Muy importante:

```text
Migration Candidate Group
≠
Migration Batch
```

---

# 151. Candidate group

Representa:

```text
multiple migration definitions proposed from one diff
```

Batch representa:

```text
migrations actually grouped during application
```

---

# 152. Candidate dependency graph

Si se generan:

```text
M1 expand
M2 backfill
M3 contract
```

el candidate group puede declarar:

```text
M1 → M2 → M3
```

---

# 153. Zero-downtime delegation

El documento:

```text
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md
```

profundizará esta estrategia.

---

# 154. Safety delegation

El presente sistema puede clasificar riesgos iniciales.

Pero:

```text
MigrationSafetyDecision
```

pertenece al documento:

```text
111_DATABASE_MIGRATION_SAFETY_SYSTEM.md
```

---

# 155. Preliminary risk

Se propone:

```text
SchemaDiffMigrationRiskHint
```

no:

```text
FinalSafetyDecision
```

---

# 156. Risk hints

```text
DESTRUCTIVE
LOCKING_RISK
DATA_TRANSFORMATION
APPLICATION_BREAKING
PLATFORM_EMULATION
UNKNOWN
```

---

# 157. Security

La generación desde schema diff deberá tratar como trusted boundary:

```text
raw schema expressions
raw platform metadata
extension payloads
manual SQL proposals
```

---

# 158. Raw expression

Nunca asumir que raw expression observada es portable.

---

# 159. Raw SQL proposal

Default:

```text
disabled
```

salvo escape hatch explícito.

---

# 160. Identifier safety

Los nombres de objetos seguirán siendo:

```text
StructuredIdentifier
```

No strings interpolados directamente en SQL.

---

# 161. Sensitive metadata

Diagnostics no deberán exponer:

```text
credentials
sensitive data samples
secret defaults
```

---

# 162. Persistent runtime

Shared:

```text
immutable interpreters
frozen rule registry
policy definitions
```

Operation-scoped:

```text
SchemaDiffMigrationSession
candidate builders
temporary graphs
diagnostics
rename analysis state
```

---

# 163. No current schema singleton

Prohibido:

```php
SchemaDiffMigration::$currentSchema;
```

---

# 164. FrankenPHP

No deberá sobrevivir entre requests:

```text
current diff
candidate
rename decisions
temporary diagnostics
target schema
```

---

# 165. RoadRunner/OpenSwoole

Misma regla para:

```text
jobs
workers
coroutines
```

---

# 166. Extension model

Se propone:

```text
SchemaDiffMigrationExtension
```

---

# 167. Extension points

Podrán agregar:

```text
difference interpreters
custom schema object support
custom migration operation proposals
custom rename evidence
custom data requirements
custom compatibility rules
candidate metadata
```

---

# 168. Extension restrictions

No podrán:

- ejecutar SQL;
- abrir conexiones ocultas;
- convertir UNKNOWN en confirmed sin evidencia;
- ocultar destructive changes;
- aprobar safety;
- modificar snapshots;
- alterar candidate después de seal;
- sobrescribir silenciosamente core interpreters.

---

# 169. Frozen registry

Toda extensión deberá registrarse antes del análisis y congelarse.

---

# 170. Extension fingerprint

Debe participar en:

```text
CandidateFingerprint
```

cuando afecte semántica.

---

# 171. Budget model

Un diff enorme puede contener miles de objetos.

Se propone:

```text
SchemaDiffMigrationBudget
```

---

# 172. Budgets

```text
max differences
max candidate operations
max rename comparisons
max graph nodes
max graph edges
max alternatives
max analysis duration
```

---

# 173. Budget exhaustion

No deberá producir:

```text
COMPLETE
```

si no se terminó el análisis.

Resultado:

```text
PARTIAL
```

o:

```text
UNKNOWN
```

---

# 174. Rename complexity

Comparar cada removed object contra cada added object puede ser:

```text
O(R × A)
```

---

# 175. Candidate indexing

Puede utilizarse:

```text
type signature
namespace
structural fingerprint
normalized identifier
```

para reducir candidatos.

---

# 176. Optimization ≠ semantic shortcut

Indexar candidatos no permite omitir evidencia.

---

# 177. Testing architecture

Debe probar:

- empty diff;
- create-only diff;
- destructive diff;
- rename explicit;
- rename ambiguous;
- table rename;
- index rename;
- constraint normalization;
- partial snapshots;
- unknown metadata;
- type widening;
- type narrowing;
- nullable → not null;
- unique constraint addition;
- FK addition;
- cyclic dependencies;
- data backfill requirement;
- platform emulation;
- unsupported feature;
- deterministic generation;
- candidate fingerprint;
- round-trip simulation;
- residual diff;
- drift;
- extension conflicts;
- budget exhaustion;
- persistent runtime isolation.

---

# 178. Empty diff

```text
Diff(Current, Target) = ∅
```

Resultado recomendado:

```text
NoMigrationRequired
```

No generar archivo vacío por default.

---

# 179. Create-only test

```text
Target = Current + Table X
```

Debe producir:

```text
CreateTable proposal
```

con dependencies correctas.

---

# 180. Partial snapshot test

Current coverage:

```text
PARTIAL
```

Objeto no observado.

No generar automáticamente:

```text
DROP
```

---

# 181. Rename ambiguity test

```text
removed:
first_name
last_name

added:
name
display_name
```

No realizar matching arbitrario.

---

# 182. Type narrowing test

```text
BIGINT
→
SMALLINT
```

Debe generar:

```text
DataValidationRequirement
```

---

# 183. NOT NULL test

```text
nullable
→
not null
```

Debe generar validation/backfill requirements.

---

# 184. Unique test

```text
add UNIQUE(email)
```

Debe considerar:

```text
duplicate validation
```

---

# 185. FK test

Debe considerar:

```text
orphans
referenced uniqueness
type compatibility
platform capability
```

---

# 186. Round-trip property test

Para candidatos puramente estructurales:

```text
Diff(
    Interpret(Current, CandidateOperations),
    Target
)
=
∅
```

bajo profile declarado.

---

# 187. Determinism test

```text
Generate(X) == Generate(X)
```

estructuralmente y por fingerprint.

---

# 188. Error hierarchy

```text
DatabaseSchemaDiffMigrationException
├── SchemaDiffMigrationAnalysisException
├── InvalidSchemaDiffMigrationRequestException
├── IncompleteSchemaDiffMigrationSourceException
├── SchemaDiffMigrationInterpretationException
├── UnsupportedSchemaDifferenceException
├── SchemaDiffMigrationRenameException
├── AmbiguousSchemaRenameException
├── SchemaChangeGraphException
│   └── SchemaChangeDependencyCycleException
├── MigrationOperationSynthesisException
├── MigrationDataRequirementException
├── UnresolvedMigrationDataRequirementException
├── SchemaDiffMigrationCompatibilityException
├── SchemaDiffMigrationReviewRequiredException
├── SchemaDiffMigrationCandidateException
├── SchemaDiffMigrationCandidateIncompleteException
├── SchemaDiffMigrationCandidateIntegrityException
├── SchemaDiffMigrationSourceDriftException
├── SchemaDiffMigrationRoundTripException
├── SchemaDiffMigrationBudgetExceededException
├── SchemaDiffMigrationExtensionException
└── SchemaDiffMigrationInvariantException
```

---

# 189. Namespace

Se propone:

```text
VoltStack\Quantum\Database\Migration\SchemaDiff
```

---

# 190. Estructura propuesta

```text
Migration/
└── SchemaDiff/
    ├── Contract/
    │   ├── SchemaDiffMigrationAnalyzer.php
    │   ├── SchemaDifferenceMigrationInterpreter.php
    │   ├── MigrationOperationSynthesizer.php
    │   └── MigrationDefinitionGenerator.php
    │
    ├── Analysis/
    │   ├── DefaultSchemaDiffMigrationAnalyzer.php
    │   ├── SchemaDiffMigrationRequest.php
    │   ├── SchemaDiffMigrationAnalysis.php
    │   ├── SchemaDiffMigrationContext.php
    │   └── SchemaDiffMigrationCompleteness.php
    │
    ├── Candidate/
    │   ├── SchemaDiffMigrationCandidate.php
    │   ├── SchemaDiffMigrationCandidateId.php
    │   ├── SchemaDiffMigrationCandidateSet.php
    │   ├── SchemaDiffMigrationCandidateGroup.php
    │   ├── SchemaDiffMigrationCandidateFingerprint.php
    │   └── SchemaDiffMigrationCandidateCompleteness.php
    │
    ├── Difference/
    │   ├── SchemaDifferenceMigrationInterpreterRegistry.php
    │   ├── SchemaChangeInterpretation.php
    │   ├── SchemaChangeIntent.php
    │   ├── SchemaChangeImpact.php
    │   └── SchemaChangeEvidenceSet.php
    │
    ├── Rename/
    │   ├── SchemaRenameAnalyzer.php
    │   ├── SchemaRenameEvidence.php
    │   ├── SchemaRenameEvidenceSet.php
    │   ├── SchemaRenameConfidence.php
    │   ├── SchemaRenameResolution.php
    │   └── SchemaRenameResolutionMap.php
    │
    ├── Graph/
    │   ├── SchemaChangeGraph.php
    │   ├── SchemaChangeNode.php
    │   ├── SchemaChangeNodeId.php
    │   ├── SchemaChangeEdge.php
    │   └── SchemaChangeGraphBuilder.php
    │
    ├── Operation/
    │   ├── MigrationOperationProposal.php
    │   ├── MigrationOperationProposalId.php
    │   ├── MigrationOperationProposalSet.php
    │   ├── SchemaOperationProposal.php
    │   ├── DataOperationProposal.php
    │   ├── ValidationOperationProposal.php
    │   ├── BarrierOperationProposal.php
    │   └── ManualOperationProposal.php
    │
    ├── Data/
    │   ├── MigrationDataRequirement.php
    │   ├── MigrationDataRequirementId.php
    │   ├── MigrationDataRequirementKind.php
    │   ├── MigrationDataRequirementSet.php
    │   ├── BackfillRequirement.php
    │   ├── DataValidationRequirement.php
    │   ├── DataTransformationRequirement.php
    │   └── BackupRequirement.php
    │
    ├── Compatibility/
    │   ├── SchemaDiffMigrationCompatibilityAnalyzer.php
    │   └── SchemaDiffMigrationCompatibilityReport.php
    │
    ├── Review/
    │   ├── MigrationReviewRequirement.php
    │   ├── MigrationReviewRequirementSet.php
    │   ├── MigrationReviewKind.php
    │   └── MigrationReviewState.php
    │
    ├── Strategy/
    │   ├── MigrationStrategyAlternative.php
    │   └── MigrationStrategyAlternativeSet.php
    │
    ├── Validation/
    │   ├── SchemaDiffMigrationCandidateValidator.php
    │   ├── SchemaDiffMigrationRoundTripValidator.php
    │   ├── SchemaPrecondition.php
    │   └── SchemaPostcondition.php
    │
    ├── Generation/
    │   ├── DefaultMigrationDefinitionGenerator.php
    │   ├── GeneratedMigrationMetadata.php
    │   └── GeneratedMigrationProvenance.php
    │
    ├── Policy/
    │   ├── SchemaDiffMigrationPolicy.php
    │   ├── RenameInferencePolicy.php
    │   ├── DestructiveChangePolicy.php
    │   └── CandidateGenerationMode.php
    │
    ├── Budget/
    │   └── SchemaDiffMigrationBudget.php
    │
    ├── Extension/
    │   ├── SchemaDiffMigrationExtension.php
    │   └── SchemaDiffMigrationExtensionRegistry.php
    │
    ├── Diagnostic/
    │   ├── SchemaDiffMigrationDiagnostic.php
    │   └── SchemaDiffMigrationDiagnosticSet.php
    │
    └── Exception/
        └── ...
```

---

# 191. Invariantes

## DB-MIGRATION-DIFF-001
Schema Diff será distinto de Migration.

## DB-MIGRATION-DIFF-002
Difference será distinta de Migration Operation.

## DB-MIGRATION-DIFF-003
Migration Candidate será distinto de Migration Definition.

## DB-MIGRATION-DIFF-004
Migration Definition será distinta de Migration Plan.

## DB-MIGRATION-DIFF-005
Generated Migration será distinta de Executed Migration.

## DB-MIGRATION-DIFF-006
Generated Migration no implicará Safe Migration.

## DB-MIGRATION-DIFF-007
Generated Migration no implicará Approved Migration.

## DB-MIGRATION-DIFF-008
Schema equality será distinta de data equality.

## DB-MIGRATION-DIFF-009
Target schema será distinto de business state.

## DB-MIGRATION-DIFF-010
Diff-to-Migration no ejecutará DDL.

## DB-MIGRATION-DIFF-011
Analysis no abrirá conexiones ocultas.

## DB-MIGRATION-DIFF-012
Current snapshot será input explícito.

## DB-MIGRATION-DIFF-013
Target snapshot/model será input explícito.

## DB-MIGRATION-DIFF-014
Capability snapshot será explícito.

## DB-MIGRATION-DIFF-015
MissingFromPartialSnapshot no implicará removed.

## DB-MIGRATION-DIFF-016
Partial snapshot no generará destructive absence automáticamente.

## DB-MIGRATION-DIFF-017
Candidate completeness será first-class.

## DB-MIGRATION-DIFF-018
COMPLETE no implicará SAFE.

## DB-MIGRATION-DIFF-019
PARTIAL será first-class.

## DB-MIGRATION-DIFF-020
UNKNOWN será first-class.

## DB-MIGRATION-DIFF-021
Candidate ID será distinto de Migration ID.

## DB-MIGRATION-DIFF-022
Candidate ID será distinto de Schema Diff ID.

## DB-MIGRATION-DIFF-023
Candidate acceptance asignará Migration ID explícito.

## DB-MIGRATION-DIFF-024
Difference interpreter será typed.

## DB-MIGRATION-DIFF-025
Interpreter registry será frozen.

## DB-MIGRATION-DIFF-026
Interpreter resolution será deterministic.

## DB-MIGRATION-DIFF-027
No habrá silent last-wins interpreter override.

## DB-MIGRATION-DIFF-028
Change intent será explícito.

## DB-MIGRATION-DIFF-029
Change impact será distinto de severity.

## DB-MIGRATION-DIFF-030
Rename candidate será distinto de proven rename.

## DB-MIGRATION-DIFF-031
Rename inference requerirá evidencia.

## DB-MIGRATION-DIFF-032
Name similarity no probará rename.

## DB-MIGRATION-DIFF-033
Structural similarity no probará identity.

## DB-MIGRATION-DIFF-034
Explicit rename mapping tendrá mayor evidencia que heuristics.

## DB-MIGRATION-DIFF-035
Ambiguous rename permanecerá unresolved por default.

## DB-MIGRATION-DIFF-036
No habrá arbitrary similarity threshold como prueba suficiente.

## DB-MIGRATION-DIFF-037
Rename evidence será preservada.

## DB-MIGRATION-DIFF-038
Table rename seguirá reglas de evidencia.

## DB-MIGRATION-DIFF-039
Index rename no dependerá solo del physical name.

## DB-MIGRATION-DIFF-040
Constraint rename no dependerá solo del physical name.

## DB-MIGRATION-DIFF-041
Schema Change Graph será distinto del Migration Dependency Graph.

## DB-MIGRATION-DIFF-042
Change dependencies serán first-class.

## DB-MIGRATION-DIFF-043
Diff iteration order no será execution order.

## DB-MIGRATION-DIFF-044
Independent change ordering será deterministic.

## DB-MIGRATION-DIFF-045
Operation proposal será distinta de accepted operation.

## DB-MIGRATION-DIFF-046
Proposal conservará source mapping.

## DB-MIGRATION-DIFF-047
Schema operation proposal no generará SQL.

## DB-MIGRATION-DIFF-048
Data requirement será first-class.

## DB-MIGRATION-DIFF-049
Data requirement será distinto de implementation.

## DB-MIGRATION-DIFF-050
Backfill requirement no inventará transformation.

## DB-MIGRATION-DIFF-051
Unknown data transformation producirá incomplete candidate.

## DB-MIGRATION-DIFF-052
Type conversion será analizada por lossiness.

## DB-MIGRATION-DIFF-053
Narrowing type conversion requerirá validation.

## DB-MIGRATION-DIFF-054
Nullable-to-not-null requerirá data consideration.

## DB-MIGRATION-DIFF-055
Default change no implicará historical row rewrite.

## DB-MIGRATION-DIFF-056
Unique addition considerará existing duplicates.

## DB-MIGRATION-DIFF-057
Foreign key addition considerará existing orphans.

## DB-MIGRATION-DIFF-058
Foreign key addition considerará type compatibility.

## DB-MIGRATION-DIFF-059
CHECK addition considerará existing rows cuando aplique.

## DB-MIGRATION-DIFF-060
Destructive classification será first-class.

## DB-MIGRATION-DIFF-061
Removed object no ocultará destructive semantics.

## DB-MIGRATION-DIFF-062
Destructive candidate conservará data-loss risk.

## DB-MIGRATION-DIFF-063
Human review requirement será first-class.

## DB-MIGRATION-DIFF-064
Required review no será warning decorativo.

## DB-MIGRATION-DIFF-065
Compatibility será distinta de safety.

## DB-MIGRATION-DIFF-066
Final-state compatibility será distinta de transition compatibility.

## DB-MIGRATION-DIFF-067
Emulation candidate será distinta de selected strategy.

## DB-MIGRATION-DIFF-068
Diff Migration System no seleccionará hidden emulation.

## DB-MIGRATION-DIFF-069
MariaDB será first-class.

## DB-MIGRATION-DIFF-070
Version será distinta de capability.

## DB-MIGRATION-DIFF-071
Capability drift requerirá revalidation.

## DB-MIGRATION-DIFF-072
Generation mode será explícito.

## DB-MIGRATION-DIFF-073
STRICT no aceptará ambiguous rename.

## DB-MIGRATION-DIFF-074
ASSISTED conservará review requirements.

## DB-MIGRATION-DIFF-075
EXPLORATORY no convertirá alternativas en executable definitions.

## DB-MIGRATION-DIFF-076
Alternative strategy será first-class cuando aplique.

## DB-MIGRATION-DIFF-077
Strategy selection podrá delegarse al Migration Planner.

## DB-MIGRATION-DIFF-078
Zero-downtime strategy podrá delegarse al documento 110.

## DB-MIGRATION-DIFF-079
Dependency synthesis será deterministic.

## DB-MIGRATION-DIFF-080
FK cycles podrán requerir phased operations.

## DB-MIGRATION-DIFF-081
Operation proposal ID será estable dentro del candidate.

## DB-MIGRATION-DIFF-082
Proposal tendrá traceability hasta source difference.

## DB-MIGRATION-DIFF-083
Data validation utilizará Query Engine.

## DB-MIGRATION-DIFF-084
Schema Diff analysis puro no leerá row data ocultamente.

## DB-MIGRATION-DIFF-085
Runtime validation será explícita.

## DB-MIGRATION-DIFF-086
Structural completeness será distinta de data completeness.

## DB-MIGRATION-DIFF-087
Generated definition usará canonical MigrationDefinition.

## DB-MIGRATION-DIFF-088
No existirá segundo generated-migration format.

## DB-MIGRATION-DIFF-089
Generated migration conservará provenance.

## DB-MIGRATION-DIFF-090
Source filename no será semantic migration identity.

## DB-MIGRATION-DIFF-091
Candidate generation será deterministic.

## DB-MIGRATION-DIFF-092
Candidate fingerprint será versioned.

## DB-MIGRATION-DIFF-093
Candidate fingerprint será distinto de MigrationDefinition fingerprint.

## DB-MIGRATION-DIFF-094
Fingerprint no será identity proof absoluto.

## DB-MIGRATION-DIFF-095
Structural round-trip será verificado cuando sea posible.

## DB-MIGRATION-DIFF-096
Round-trip failure invalidará structural candidate.

## DB-MIGRATION-DIFF-097
Schema AST Interpreter podrá utilizarse sin DB I/O.

## DB-MIGRATION-DIFF-098
Residual diff deberá explicarse.

## DB-MIGRATION-DIFF-099
Schema equality no probará data migration completeness.

## DB-MIGRATION-DIFF-100
Data postconditions serán separadas de schema postconditions.

## DB-MIGRATION-DIFF-101
Generation drift será distinto de execution drift.

## DB-MIGRATION-DIFF-102
Expected source fingerprint podrá ser precondition.

## DB-MIGRATION-DIFF-103
Stale destructive candidate será bloqueado por default.

## DB-MIGRATION-DIFF-104
Precondition failure no reparará drift automáticamente.

## DB-MIGRATION-DIFF-105
Drop proposal validará expected source cuando sea posible.

## DB-MIGRATION-DIFF-106
Large diff podrá dividirse en múltiples candidates.

## DB-MIGRATION-DIFF-107
Candidate Group será distinto de Migration Batch.

## DB-MIGRATION-DIFF-108
Candidate Group podrá tener migration dependencies.

## DB-MIGRATION-DIFF-109
Safety hints serán distintos de final safety decision.

## DB-MIGRATION-DIFF-110
Diff Migration System no aprobará migration safety.

## DB-MIGRATION-DIFF-111
Raw expression será explicit trust boundary.

## DB-MIGRATION-DIFF-112
Raw expression no será portable por default.

## DB-MIGRATION-DIFF-113
Raw SQL generation estará deshabilitada por default.

## DB-MIGRATION-DIFF-114
Identifiers permanecerán estructurados.

## DB-MIGRATION-DIFF-115
Sensitive metadata no se expondrá en diagnostics.

## DB-MIGRATION-DIFF-116
Runtime analysis state será operation-scoped.

## DB-MIGRATION-DIFF-117
No habrá global mutable current schema.

## DB-MIGRATION-DIFF-118
No habrá global mutable current diff.

## DB-MIGRATION-DIFF-119
No habrá global mutable current candidate.

## DB-MIGRATION-DIFF-120
FrankenPHP state será aislado por operación.

## DB-MIGRATION-DIFF-121
RoadRunner state será aislado por job.

## DB-MIGRATION-DIFF-122
OpenSwoole state será aislado por coroutine.

## DB-MIGRATION-DIFF-123
Extension registry será frozen.

## DB-MIGRATION-DIFF-124
Extensions no ejecutarán SQL.

## DB-MIGRATION-DIFF-125
Extensions no abrirán hidden connections.

## DB-MIGRATION-DIFF-126
Extensions no ocultarán destructive changes.

## DB-MIGRATION-DIFF-127
Extensions no aprobarán safety.

## DB-MIGRATION-DIFF-128
Extensions no mutarán source snapshots.

## DB-MIGRATION-DIFF-129
Extensions no mutarán sealed candidates.

## DB-MIGRATION-DIFF-130
Semantic extensions participarán en fingerprint.

## DB-MIGRATION-DIFF-131
Analysis budgets serán explícitos.

## DB-MIGRATION-DIFF-132
Budget exhaustion no devolverá false COMPLETE.

## DB-MIGRATION-DIFF-133
Rename candidate search podrá optimizarse.

## DB-MIGRATION-DIFF-134
Optimization no reducirá evidence requirements.

## DB-MIGRATION-DIFF-135
Empty diff no generará empty migration por default.

## DB-MIGRATION-DIFF-136
Partial snapshots serán testeados.

## DB-MIGRATION-DIFF-137
Ambiguous renames serán testeados.

## DB-MIGRATION-DIFF-138
Type narrowing será testeado.

## DB-MIGRATION-DIFF-139
Nullability changes serán testeados.

## DB-MIGRATION-DIFF-140
Constraint validation requirements serán testeados.

## DB-MIGRATION-DIFF-141
Cyclic schema dependencies serán testeadas.

## DB-MIGRATION-DIFF-142
Platform emulation será testeada.

## DB-MIGRATION-DIFF-143
Determinism será testeado.

## DB-MIGRATION-DIFF-144
Round-trip será testeado.

## DB-MIGRATION-DIFF-145
Drift será testeado.

## DB-MIGRATION-DIFF-146
Extension collision será testeado.

## DB-MIGRATION-DIFF-147
Budget exhaustion será testeado.

## DB-MIGRATION-DIFF-148
Candidate diagnostics serán explainable.

## DB-MIGRATION-DIFF-149
No blocking unknown será convertido silenciosamente en operation.

## DB-MIGRATION-DIFF-150
VoltStack nunca ejecutará directamente una diferencia de esquema; primero deberá convertirla en una transición de migración explícita, validable, revisable y gobernada.

---

# 192. Anti-patterns

## 192.1 Diff → SQL

Incorrecto:

```text
SchemaDiff
↓
SQL
↓
Execute
```

---

## 192.2 Removed + Added = Rename

Incorrecto:

```php
if ($removed && $added) {
    return rename($removed, $added);
}
```

---

## 192.3 Rename por Levenshtein

Incorrecto:

```php
if (levenshtein($old, $new) < 5) {
    rename();
}
```

---

## 192.4 Ignore data

Incorrecto:

```text
nullable → not null
=
ALTER COLUMN NOT NULL
```

sin validar filas existentes.

---

## 192.5 Generated means safe

Incorrecto:

```text
framework generated it
⇒
safe to run
```

---

## 192.6 Partial snapshot destructive diff

Incorrecto:

```text
not observed
⇒
drop
```

---

## 192.7 Hidden SQLite rebuild

Incorrecto:

```text
DropColumn
↓
Compiler silently rebuilds entire table
```

La estrategia debe ser visible antes.

---

## 192.8 One giant generated migration

Incorrecto cuando:

```text
expand
backfill
contract
```

requieren lifecycle independiente.

---

## 192.9 Raw SQL as universal fallback

Incorrecto:

```text
unsupported typed change
→ raw SQL
```

automáticamente.

---

## 192.10 Diff candidate = batch

Incorrecto:

```text
generated together
⇒
executed in same batch
```

---

# 193. Ejemplo completo — creación simple

Current:

```text
users
```

Target:

```text
users
profiles
```

Diff:

```text
TableAdded profiles
```

Interpretation:

```text
CREATE profiles
```

Proposal:

```text
CreateTableProposal
```

Round-trip:

```text
Interpret(Current, CreateProfiles)
≈
Target
```

Candidate:

```text
COMPLETE
```

---

# 194. Ejemplo — rename confirmado

Current:

```text
users.name VARCHAR(255)
```

Target:

```text
users.full_name VARCHAR(255)
```

Metadata:

```text
ExplicitRename:
users.name → users.full_name
```

Resultado:

```text
RenameConfidence = CONFIRMED
```

Candidate:

```text
RenameColumnProposal
```

No:

```text
Drop name
Add full_name
```

---

# 195. Ejemplo — rename ambiguo

Current:

```text
first_name
last_name
```

Target:

```text
name
display_name
```

Resultado:

```text
AmbiguousRenameSet
```

Candidate:

```text
MANUAL_INTERVENTION_REQUIRED
```

VoltStack no inventa correspondencias.

---

# 196. Ejemplo — NOT NULL

Current:

```text
users
```

Target:

```text
users.phone VARCHAR(50) NOT NULL
```

Candidate:

```text
AddColumn phone nullable
        ↓
BackfillRequirement phone
        ↓
ValidateNotNull phone
        ↓
AlterColumn phone NOT NULL
```

El desarrollador debe resolver:

```text
how phone is populated
```

---

# 197. Ejemplo — type narrowing

Current:

```text
orders.reference VARCHAR(255)
```

Target:

```text
orders.reference VARCHAR(64)
```

Candidate:

```text
ValidateLength(reference <= 64)
        ↓
AlterColumn VARCHAR(64)
```

Classification:

```text
POTENTIALLY_DESTRUCTIVE
```

---

# 198. Ejemplo — unique constraint

Current:

```text
users.email
```

Target:

```text
UNIQUE(users.email)
```

Candidate:

```text
ValidateNoDuplicateEmails
        ↓
AddUniqueConstraint
```

---

# 199. Ejemplo — foreign key

Current:

```text
orders.customer_id BIGINT
```

Target:

```text
orders.customer_id
FK → customers.id
```

Candidate:

```text
ValidateCompatibleTypes
        ↓
ValidateReferencedCandidateKey
        ↓
ValidateNoOrphans
        ↓
AddForeignKey
```

---

# 200. Ejemplo — destructive removal

Current:

```text
users.legacy_payload JSON
```

Target:

```text
column absent
```

Candidate:

```text
DropColumnProposal
```

con:

```text
Destructive = TRUE
DataLossRisk = CERTAIN/POTENTIAL
Review = REQUIRED
BackupRequirement = policy-dependent
```

Nunca simplemente:

```text
DROP COLUMN legacy_payload
```

---

# 201. Ejemplo — generated candidate group

Target requiere migrar:

```text
users.email
→
users.normalized_email
```

sin downtime.

El Diff Migration System puede producir:

```text
Candidate Group
│
├── Candidate A
│   ADD normalized_email
│
├── Candidate B
│   BACKFILL REQUIRED
│   VALIDATE
│
└── Candidate C
    REMOVE old representation
```

Dependencias:

```text
A → B → C
```

El documento 110 definirá cómo esto se convierte en una estrategia formal de zero downtime.

---

# 202. Fórmula de generación

```text
MigrationCandidate
=
Interpret(
    Diff(Current, Target),
    Capabilities,
    Policy,
    Evidence
)
```

---

# 203. Fórmula de candidate completeness

```text
CandidateComplete
=
AllRelevantDifferencesInterpreted
∧
AllBlockingRenamesResolved
∧
AllRequiredDataTransformationsSpecified
∧
AllRequiredOperationsRepresentable
∧
NoBlockingUnknowns
```

---

# 204. Fórmula de rename

```text
ConfirmedRename(A, B)
=
ExplicitIdentityEvidence(A, B)
∨
ExplicitDeveloperMapping(A, B)
∨
OtherPolicyAcceptedProof(A, B)
```

No:

```text
ConfirmedRename
=
NameSimilarity > Threshold
```

---

# 205. Fórmula de round-trip

Para la parte estructural:

```text
Predicted
=
Interpret(
    Current,
    Candidate.SchemaOperations
)
```

Ideal:

```text
Compare(Predicted, Target, Profile)
=
EQUAL
```

---

# 206. Fórmula de stale candidate

```text
StaleCandidate
=
CurrentSchemaFingerprint(now)
≠
ExpectedCurrentSchemaFingerprint(candidate)
```

cuando el comparison contract exige igualdad.

---

# 207. Fórmula de aceptación

```text
CandidateAcceptable
=
CandidateComplete
∧
RequiredReviewsSatisfied
∧
CompatibilityAcceptable
∧
SourceStateStillValid
∧
NoBlockingUnknowns
```

Esto aún no implica:

```text
MigrationSafeToExecute
```

porque Safety System mantiene autoridad posterior.

---

# 208. Fórmula maestra

```text
Schema Diff Migration System
=
Schema Difference Interpretation
+
Rename Evidence Resolution
+
Change Classification
+
Change Dependency Graph
+
Typed Operation Proposals
+
Data Migration Requirements
+
Compatibility Analysis
+
Destructive Change Visibility
+
Human Review Requirements
+
Deterministic Candidate Generation
+
Migration Definition Generation
+
Structural Round-Trip Verification
+
Source Drift Detection
+
Extension Governance
+
Persistent Runtime Isolation
```

---

# 209. Regla arquitectónica maestra

> **VoltStack nunca tratará una diferencia estructural como una instrucción directa de ejecución. Una diferencia debe convertirse primero en una propuesta explícita de evolución cuya identidad, evidencia, dependencias, requisitos de datos, compatibilidad e incertidumbres sean visibles y revisables.**

La relación correcta será:

```text
Current Schema
      +
Target Schema
      ↓
Schema Diff
      ↓
Interpretation
      ↓
Evidence
      ↓
Migration Candidate
      ↓
Review
      ↓
Migration Definition
      ↓
Migration Planner
      ↓
Migration Safety
      ↓
Execution
```

---

# 210. Resultado arquitectónico

Con este sistema VoltStack podrá soportar:

```text
schema-driven migration generation
developer-reviewed migration generation
safe rename detection
partial-schema awareness
data migration requirement detection
platform-aware migration proposals
destructive-change detection
multi-step generated migrations
zero-downtime candidate generation
deterministic migration generation
schema round-trip validation
migration provenance
schema drift protection
```

sin adoptar el peligroso modelo:

```text
schema changed
↓
generate SQL
↓
run SQL
```

El sistema preservará especialmente:

```text
Evidence
Intent
Safety Boundaries
Traceability
Determinism
Reviewability
```

---

# 211. Relación con documentos anteriores

Este documento depende conceptualmente de:

```text
87_DATABASE_SCHEMA_ARCHITECTURE.md
88_DATABASE_SCHEMA_MODEL.md
89_DATABASE_SCHEMA_AST_SYSTEM.md
90_DATABASE_SCHEMA_BUILDER_SYSTEM.md
96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md
97_DATABASE_SCHEMA_METADATA_SYSTEM.md
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
99_DATABASE_SCHEMA_COMPILER_SYSTEM.md
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md

101_DATABASE_MIGRATION_ARCHITECTURE.md
102_DATABASE_MIGRATION_SYSTEM.md
103_DATABASE_MIGRATION_DISCOVERY_SYSTEM.md
104_DATABASE_MIGRATION_REPOSITORY_SYSTEM.md
105_DATABASE_MIGRATION_PLANNER_SYSTEM.md
106_DATABASE_MIGRATION_EXECUTION_SYSTEM.md
107_DATABASE_MIGRATION_ROLLBACK_SYSTEM.md
108_DATABASE_MIGRATION_BATCH_SYSTEM.md
```

---

# 212. Siguiente documento

```text
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md
```

El siguiente documento deberá formalizar una de las capacidades más importantes del sistema de migraciones de VoltStack:

```text
Expand
   ↓
Migrate / Backfill
   ↓
Dual Compatibility
   ↓
Validate
   ↓
Switch
   ↓
Contract
```

y distinguir:

```text
Zero Downtime
≠
No Locks

Zero Downtime
≠
Instant Migration

Zero Downtime
≠
One Migration

Zero Downtime
≠
One Transaction

Zero Downtime
≠
No Operational Risk

Schema Compatibility
≠
Application Compatibility

Backward-Compatible Schema
≠
Forward-Compatible Schema

Deployment
≠
Migration

Online DDL
≠
Zero-Downtime Migration
```

incluyendo formalmente:

```text
expand/contract architecture
compatibility windows
old/new application coexistence
dual-read/dual-write strategies
backfills
shadow columns/tables
online index creation
constraint validation phases
application cutover barriers
rollback windows
contract safety
large-table migration strategies
replica considerations
persistent runtimes
migration phases
checkpoints
resume
verification
telemetry
safety integration
```