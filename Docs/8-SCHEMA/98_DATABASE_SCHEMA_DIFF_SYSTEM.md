# 98_DATABASE_SCHEMA_DIFF_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Diff System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 98 — Database Schema Diff System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Diff System` define el motor responsable de comparar dos representaciones estructurales de un esquema y producir una descripción tipada, determinista, explicable y segura de sus diferencias.

Su función principal será transformar:

```text
Current Schema
      +
Target Schema
      +
Coverage
      +
Comparison Profile
      +
Platform Capabilities
      ↓
Schema Diff
```

El sistema responderá:

```text
¿Qué diferencias existen?
```

pero no:

```text
¿Cómo se ejecutarán?
```

ni:

```text
¿Deben aplicarse?
```

ni:

```text
¿Qué SQL debe generarse?
```

La separación fundamental será:

```text
Difference
≠
Change Operation

Difference
≠
Migration

Difference
≠
DDL

Difference
≠
Authorization

Difference
≠
Execution
```

---

# 2. Principio central

> **Schema Diff descubre diferencias estructurales entre dos estados; no convierte automáticamente esas diferencias en operaciones destructivas ni en SQL.**

Formalmente:

```text
Diff(Current, Target)
=
Structural Differences
```

No:

```text
Diff(Current, Target)
=
Executable Migration
```

---

# 3. Regla de seguridad fundamental

Debe cumplirse:

```text
NotObserved
≠
Absent
```

y por tanto:

```text
MissingFromPartialSnapshot
≠
DropCandidate
```

Un objeto solo podrá clasificarse como ausente cuando exista evidencia suficiente:

```text
SafelyAbsent(O)
=
NotObserved(O)
∧
RelevantCoverage(O) = COMPLETE
∧
Visibility(O) = SUFFICIENT
∧
ObservationSucceeded
```

---

# 4. Posición arquitectónica

```text
Physical Database
      │
      ▼
Schema Introspection
      │
      ▼
Current Schema Snapshot
      │
      │
      ├───────────────────────┐
      │                       │
      ▼                       ▼
Schema Metadata          Target Schema
      │                       │
      └───────────┬───────────┘
                  ▼
          Schema Diff System
                  │
                  ▼
             SchemaDiff
                  │
                  ▼
         Schema Change Planner
                  │
                  ▼
              Schema AST
                  │
                  ▼
          Schema Compiler
                  │
                  ▼
              Commands
```

---

# 5. Responsabilidades

El sistema será responsable de:

- comparar schemas;
- comparar namespaces;
- comparar tablas;
- comparar columnas;
- comparar índices;
- comparar constraints;
- comparar foreign keys;
- comparar sequences;
- comparar views;
- comparar metadata;
- identificar objetos agregados;
- identificar objetos ausentes;
- identificar objetos modificados;
- detectar candidatos a rename;
- preservar incertidumbre;
- clasificar diferencias;
- calcular confidence;
- producir diagnostics;
- construir dependency information;
- generar fingerprints;
- soportar perfiles de comparación;
- soportar extensiones;
- mantener determinismo.

---

# 6. No responsabilidades

Schema Diff no deberá:

- ejecutar SQL;
- generar SQL;
- ejecutar DDL;
- iniciar transacciones;
- ejecutar migrations;
- decidir rollback;
- autorizar cambios;
- modificar el schema;
- modificar snapshots;
- abrir conexiones;
- introspectar ocultamente;
- convertir automáticamente rename en drop/create;
- decidir estrategias zero-downtime;
- realizar backfills;
- bloquear tablas.

---

# 7. Diferencia entre estado e intención

Debe distinguirse:

```text
Schema State
```

de:

```text
Schema Change Intent
```

Ejemplo:

```text
Current:
users.email VARCHAR(255)

Target:
users.email VARCHAR(320)
```

Schema Diff detecta:

```text
Column Difference
length:
255 → 320
```

No genera todavía:

```text
ALTER TABLE users ...
```

---

# 8. Difference ≠ Change Operation

Ejemplo:

```text
Difference:
column email changed type
```

Una capa posterior puede decidir:

```text
AlterColumn
```

o:

```text
Add temporary column
Backfill
Swap columns
Drop old column
```

dependiendo de:

```text
platform
safety
migration policy
zero-downtime strategy
```

Por tanto:

```text
Detected Difference
≠
Physical Change Strategy
```

---

# 9. Modelo general

Se propone:

```text
SchemaDiff
├── DatabaseDifferences
├── NamespaceDifferences
├── TableDifferences
├── SequenceDifferences
├── ViewDifferences
├── MetadataDifferences
├── UnresolvedDifferences
├── RenameCandidates
├── Diagnostics
└── DiffMetadata
```

---

# 10. SchemaDiffer

Contrato conceptual:

```php
interface SchemaDiffer
{
    public function diff(
        DatabaseSchema $current,
        DatabaseSchema $target,
        SchemaDiffContext $context,
    ): SchemaDiff;
}
```

Cuando `current` provenga de introspection:

```text
SchemaDiffContext
```

deberá recibir coverage y observation metadata.

---

# 11. SchemaDiffContext

```php
final readonly class SchemaDiffContext
{
    public function __construct(
        public SchemaComparisonProfile $profile,
        public PlatformCapabilitySnapshot $capabilities,
        public SchemaCoverageDescriptor $currentCoverage,
        public SchemaDiffPolicy $policy,
        public SchemaDiffBudget $budget,
    ) {}
}
```

---

# 12. No hidden context

No deberá existir:

```text
current platform global
current tenant global
current schema global
current connection global
```

dentro del differ.

Todo contexto relevante será explícito.

---

# 13. Inputs

Los inputs podrán ser:

```text
Observed → Declared
Declared → Declared
Observed → Observed
Snapshot → Target Definition
Snapshot → Snapshot
```

pero el contexto deberá indicar qué clase de comparación se está realizando.

---

# 14. Comparison direction

Schema Diff será direccional.

```text
Diff(A, B)
```

significa:

```text
differences required to understand
A → B
```

Por tanto:

```text
Diff(A, B)
≠
Diff(B, A)
```

aunque puedan estar relacionados.

---

# 15. Current y target

Convención:

```text
Current
=
estado inicial

Target
=
estado deseado
```

---

# 16. Difference kinds

Se propone:

```text
SchemaDifferenceKind
├── ADDED
├── REMOVED
├── MODIFIED
├── RENAMED
├── MOVED
├── UNCHANGED
├── UNKNOWN
├── INCOMPARABLE
└── PLATFORM_DEPENDENT
```

`UNCHANGED` normalmente podrá omitirse del resultado compacto.

---

# 17. Added

```text
Current:
no users table

Target:
users table
```

si existe coverage suficiente:

```text
TableDifference
kind = ADDED
```

---

# 18. Removed

```text
Current:
legacy_users

Target:
no legacy_users
```

solo podrá clasificarse:

```text
REMOVED
```

si el target representa de forma completa el espacio relevante y las políticas permiten esa interpretación.

---

# 19. Missing ≠ removed

Debe mantenerse:

```text
MissingFromRepresentation
≠
Removed
```

porque una representación puede ser parcial.

---

# 20. Modified

Ejemplo:

```text
Current:
email VARCHAR(255)

Target:
email VARCHAR(320)
```

produce:

```text
ColumnDifference
kind = MODIFIED
```

---

# 21. Renamed

Ejemplo:

```text
Current:
users.full_name

Target:
users.name
```

puede ser:

```text
RenameCandidate
```

pero no necesariamente:

```text
RENAMED
```

---

# 22. Rename problem

Desde dos estados aislados:

```text
A has full_name
B has name
```

no puede demostrarse automáticamente que:

```text
full_name → name
```

pues también puede significar:

```text
drop full_name
+
add name
```

---

# 23. Rename confidence

Se propone:

```text
RenameConfidence
├── EXPLICIT
├── VERY_HIGH
├── HIGH
├── MEDIUM
├── LOW
└── NONE
```

---

# 24. Explicit rename

La mejor evidencia proviene de intención explícita:

```text
Schema AST:
RenameColumn(full_name, name)
```

Pero un diff puro entre estados normalmente no dispone de ella.

Si existe metadata de evolución:

```text
SchemaObjectEvolutionMetadata
```

podrá utilizarse.

---

# 25. Inferred rename

Podrá analizar:

```text
type
nullability
default
position
constraints
indexes
foreign keys
metadata
neighbor columns
structural fingerprints
```

para producir:

```text
RenameCandidate
```

---

# 26. Rename candidate ≠ rename fact

Regla:

```text
Similarity
≠
Identity
```

---

# 27. Default rename policy

Se recomienda:

```text
automatic rename inference
=
conservative
```

y para cambios destructivos:

```text
explicit confirmation / policy
```

en capas posteriores.

---

# 28. Rename ambiguity

Ejemplo:

```text
Current:
first_name VARCHAR
last_name VARCHAR

Target:
name VARCHAR
display_name VARCHAR
```

No deberá elegir arbitrariamente.

Resultado:

```text
AMBIGUOUS_RENAME
```

---

# 29. Object matching

Antes de comparar propiedades:

```text
Current Objects
       │
       ▼
Object Matcher
       │
       ▼
Matched Pairs
+
Unmatched Current
+
Unmatched Target
```

---

# 30. SchemaObjectMatcher

Se propone:

```php
interface SchemaObjectMatcher
{
    public function match(
        SchemaObjectSet $current,
        SchemaObjectSet $target,
        SchemaDiffContext $context,
    ): SchemaObjectMatchResult;
}
```

---

# 31. Matching phases

```text
1. Explicit identity/evolution mapping
2. Exact qualified identifier
3. Platform-normalized identifier
4. Stable structural identity where available
5. Rename candidate analysis
6. Unmatched classification
```

---

# 32. Matching by hash alone

Prohibido:

```text
same fingerprint
=
same object
```

El fingerprint puede acelerar candidate lookup.

No reemplaza verificación estructural.

---

# 33. Table matching

La tabla deberá considerar:

```text
qualified path
namespace
identifier semantics
evolution metadata
structural similarity
```

---

# 34. Column matching

Dentro de tabla:

```text
exact identifier
    ↓
normalized identifier
    ↓
evolution mapping
    ↓
rename candidate analysis
```

---

# 35. Index matching

Los índices presentan una particularidad:

```text
physical/generated name
```

puede cambiar aunque la semántica sea igual.

Por ello se deberá soportar:

```text
name identity
```

y:

```text
semantic index identity
```

según perfil.

---

# 36. Constraint matching

Debe distinguirse:

```text
Constraint Name Equality
```

de:

```text
Constraint Semantic Equality
```

Ejemplo:

```text
uq_users_email
```

y:

```text
users_email_unique
```

pueden representar la misma restricción semántica.

---

# 37. Foreign key matching

Puede utilizar:

```text
local columns
referenced object
referenced columns
update action
delete action
match
deferrability
```

además del nombre.

---

# 38. Primary key matching

Primary Key será singular por tabla.

Su comparación se realizará estructuralmente.

No mediante matching de índice físico.

---

# 39. Comparison hierarchy

```text
SchemaDiffer
    │
    ├── NamespaceDiffer
    ├── TableDiffer
    │    ├── ColumnDiffer
    │    ├── IndexDiffer
    │    ├── ConstraintDiffer
    │    ├── ForeignKeyDiffer
    │    └── TableMetadataDiffer
    │
    ├── SequenceDiffer
    ├── ViewDiffer
    └── SchemaMetadataDiffer
```

---

# 40. TableDifference

Se propone:

```php
final readonly class TableDifference
{
    public function __construct(
        public SchemaDifferenceKind $kind,
        public ?Table $current,
        public ?TableDefinition $target,
        public ColumnDifferenceSet $columns,
        public IndexDifferenceSet $indexes,
        public ConstraintDifferenceSet $constraints,
        public ForeignKeyDifferenceSet $foreignKeys,
        public SchemaMetadataDiff $metadata,
        public SchemaDifferenceEvidence $evidence,
    ) {}
}
```

---

# 41. Added table

Podrá contener:

```text
current = null
target = TableDefinition
kind = ADDED
```

---

# 42. Removed table

Podrá contener:

```text
current = Table
target = null
kind = REMOVED
```

pero solo si el contexto permite afirmar ausencia.

---

# 43. Modified table

```text
current != null
target != null
kind = MODIFIED
```

con diferencias internas.

---

# 44. Table rename candidate

Preferiblemente será objeto independiente:

```text
TableRenameCandidate
```

hasta alcanzar evidencia suficiente.

---

# 45. ColumnDifference

```php
final readonly class ColumnDifference
{
    public function __construct(
        public SchemaDifferenceKind $kind,
        public ?ColumnDefinition $current,
        public ?ColumnDefinition $target,
        public ColumnPropertyDifferenceSet $properties,
        public SchemaDifferenceEvidence $evidence,
    ) {}
}
```

---

# 46. Column property differences

Pueden incluir:

```text
TYPE
LENGTH
PRECISION
SCALE
UNSIGNED
NULLABILITY
DEFAULT
GENERATION
COLLATION
CHARSET
COMMENT
POSITION
PLATFORM_OPTIONS
```

según el modelo definido.

---

# 47. Property change model

Se propone:

```php
final readonly class PropertyDifference
{
    public function __construct(
        public PropertyPath $property,
        public mixed $current,
        public mixed $target,
        public PropertyDifferenceKind $kind,
    ) {}
}
```

En implementación real deberán preferirse tipos especializados para propiedades importantes.

---

# 48. Property difference kinds

```text
ADDED
REMOVED
CHANGED
UNKNOWN
INCOMPARABLE
EQUIVALENT_AFTER_NORMALIZATION
```

---

# 49. Null semantics

No deberá confundirse:

```text
property absent
```

con:

```text
property value = null
```

Ejemplo crítico:

```text
NoDefault
≠
DefaultNull
```

---

# 50. Type comparison

Debe utilizar:

```text
Database Type System
+
Comparison Profile
+
Platform Normalization
```

No strings SQL.

---

# 51. Native type string comparison

Incorrecto:

```php
$current->nativeType === $target->nativeType;
```

como única estrategia.

---

# 52. Type equality

Se podrá distinguir:

```text
ExactTypeEquality
LogicalTypeEquality
SemanticTypeEquality
PlatformNormalizedTypeEquality
```

---

# 53. Example

```text
INT
INTEGER
```

pueden ser físicamente equivalentes en cierta plataforma.

Mientras:

```text
VARCHAR(255)
VARCHAR(320)
```

normalmente no.

---

# 54. Nullability difference

```text
nullable=true
→
nullable=false
```

será una diferencia estructural/semántica.

Pero Schema Diff no decide si puede aplicarse sin backfill.

---

# 55. Default comparison

Se compararán tipos:

```text
NoDefault
LiteralDefault
ExpressionDefault
GeneratedDefault
UnknownDefault
```

---

# 56. Expression normalization

Defaults:

```text
CURRENT_TIMESTAMP
current_timestamp()
```

podrían ser equivalentes según plataforma.

La equivalencia será delegada a:

```text
SchemaExpressionComparator
```

---

# 57. Unknown expression

Si una expresión actual es opaca:

```text
RawPlatformExpression
```

y no puede compararse con seguridad:

```text
INCOMPARABLE
```

o:

```text
UNKNOWN
```

No asumir diferencia.

---

# 58. Column position

El perfil decidirá si:

```text
column order
```

participa en diff.

---

# 59. Portable profile

Puede ignorar orden cuando la plataforma o el proyecto no lo considere relevante.

---

# 60. Exact profile

Puede detectar:

```text
ordinal changed
```

como diferencia.

---

# 61. Index diff

`IndexDiffer` comparará:

```text
name
uniqueness
keys
key order
expressions
sort order
predicate
included columns
method
options
metadata
```

---

# 62. Index name policy

Se propone:

```text
IndexNameComparison
├── STRICT
├── SEMANTIC
└── IGNORE_GENERATED_NAMES
```

---

# 63. Generated names

Si ambos índices tienen nombres generados:

```text
idx_users_email_a1
idx_users_email_b7
```

pero semántica idéntica, el perfil puede considerarlos equivalentes.

---

# 64. Explicit names

Un cambio de nombre explícito puede considerarse diferencia si el perfil sincroniza nombres.

---

# 65. Index key order

Siempre debe preservarse semánticamente:

```text
INDEX(a, b)
≠
INDEX(b, a)
```

en términos generales.

---

# 66. Included column order

Si una plataforma considera el orden no semántico, podrá canonicalizarse según capability/profile.

---

# 67. Partial index predicate

Se comparará mediante:

```text
SchemaExpressionComparator
```

---

# 68. Index method

Ejemplo:

```text
BTREE
HASH
GIN
GIST
```

podrá ser:

```text
physical difference
```

o más relevante según la plataforma.

---

# 69. Constraint diff

Se compararán separadamente:

```text
Primary Key
Unique Constraint
Check Constraint
Foreign Key
```

---

# 70. PK difference

Ejemplo:

```text
Current PK:
(id)

Target PK:
(tenant_id, id)
```

produce diferencia explícita.

No:

```text
index difference only
```

---

# 71. Unique constraint vs unique index

Nunca se compensarán automáticamente:

```text
UniqueConstraint
```

y:

```text
UniqueIndex
```

como idénticos salvo que un platform normalization profile demuestre equivalencia apropiada.

---

# 72. Check constraint diff

Se comparará:

```text
expression
enforcement
validation
name
metadata
```

según capabilities.

---

# 73. Expression semantic equality

Dos AST distintos pueden ser semánticamente equivalentes.

Ejemplo:

```text
age >= 0
```

vs una forma normalizada equivalente.

No obstante, el comparador deberá ser conservador.

---

# 74. No theorem prover

Schema Diff no intentará demostrar equivalencia matemática arbitraria.

Solo reglas de normalización seguras y acotadas.

---

# 75. Foreign key diff

Comparará:

```text
local columns
target path
target columns
ON DELETE
ON UPDATE
MATCH
deferrability
validation
metadata
```

---

# 76. FK reference resolution

Si current FK apunta a:

```text
UNRESOLVED
```

no deberá compararse como si su target fuese conocido.

---

# 77. External reference

Una referencia externa al snapshot puede compararse por:

```text
qualified reference
```

si existe información suficiente.

---

# 78. FK rename

Un cambio solo en nombre de constraint puede ser:

```text
metadata/name difference
```

sin cambio de relación semántica.

---

# 79. Sequence diff

Comparará:

```text
start
increment
min
max
cycle
cache
ownership
metadata
```

según disponibilidad.

---

# 80. Sequence unknowns

Si la plataforma no expone una propiedad:

```text
UNKNOWN
```

no se comparará contra target como valor real.

---

# 81. View diff

Las views representan un problema especial.

Puede existir:

```text
StructuredViewDefinition
```

o:

```text
OpaqueViewDefinition
```

---

# 82. Structured view comparison

Si existe AST normalizado:

```text
ViewExpressionComparator
```

podrá comparar semánticamente dentro de límites definidos.

---

# 83. Raw view comparison

Para SQL opaco:

```text
normalized raw comparison
```

podrá utilizarse únicamente como aproximación explícita.

---

# 84. Raw text inequality

```text
raw SQL A != raw SQL B
```

no prueba necesariamente diferencia semántica.

---

# 85. Metadata diff integration

Se reutilizará:

```text
SchemaMetadataDiffer
```

del documento 97.

---

# 86. Metadata difference ≠ structural difference

Ejemplo:

```text
comment changed
```

puede ser:

```text
INFORMATIONAL
```

mientras:

```text
collation changed
```

puede ser:

```text
SEMANTIC
```

---

# 87. Difference classification

Se propone:

```text
SchemaDifferenceImpact
├── INFORMATIONAL
├── STRUCTURAL
├── SEMANTIC
├── PHYSICAL
├── DATA_AFFECTING
├── POTENTIALLY_DESTRUCTIVE
├── DESTRUCTIVE
├── PLATFORM_DEPENDENT
└── UNKNOWN
```

---

# 88. Impact ≠ authorization

```text
DESTRUCTIVE
```

solo describe impacto potencial.

No significa:

```text
authorized
```

---

# 89. Impact ≠ migration strategy

Un cambio:

```text
nullable → non-nullable
```

puede ser:

```text
DATA_AFFECTING
```

pero el diff no decide:

```text
backfill strategy
```

---

# 90. Safety classification

Puede existir:

```text
SchemaDifferenceSafety
├── SAFE
├── REQUIRES_VALIDATION
├── POTENTIALLY_DESTRUCTIVE
├── DESTRUCTIVE
├── UNKNOWN
└── INCOMPARABLE
```

---

# 91. Safety classification conservadora

Cuando falte información:

```text
UNKNOWN
```

preferible a:

```text
SAFE
```

---

# 92. Difference evidence

Cada diferencia importante podrá incluir:

```php
final readonly class SchemaDifferenceEvidence
{
    public function __construct(
        public DifferenceConfidence $confidence,
        public SchemaEvidenceSet $evidence,
        public SchemaDiagnosticSet $diagnostics,
    ) {}
}
```

---

# 93. Difference confidence

```text
DifferenceConfidence
├── CERTAIN
├── HIGH
├── MEDIUM
├── LOW
└── UNKNOWN
```

---

# 94. Certain difference

Ejemplo:

```text
Current column:
VARCHAR(255)

Target:
VARCHAR(320)

Current metadata certainty:
EXACT
```

podrá ser:

```text
CERTAIN
```

---

# 95. Low confidence difference

Ejemplo:

```text
Current:
OpaquePlatformType("foo")

Target:
CustomType("foo")
```

sin decoder compatible.

Resultado:

```text
INCOMPARABLE
```

o confidence baja.

---

# 96. Unknown difference

Debe ser first-class.

No deberá forzarse:

```text
equal
```

o:

```text
different
```

cuando la evidencia no permite decidir.

---

# 97. Three/four-valued comparison

Conceptualmente:

```text
EQUAL
DIFFERENT
UNKNOWN
INCOMPARABLE
```

---

# 98. Comparison result

```php
enum SchemaComparisonResult
{
    case EQUAL;
    case DIFFERENT;
    case UNKNOWN;
    case INCOMPARABLE;
}
```

---

# 99. Why boolean is insufficient

Una API:

```php
bool $different
```

no puede distinguir:

```text
not different
```

de:

```text
cannot determine
```

---

# 100. Comparison profiles

Se propone:

```text
SchemaComparisonProfile
├── EXACT
├── STRUCTURAL
├── SEMANTIC
├── PORTABLE
├── PLATFORM_NORMALIZED
└── MIGRATION
```

---

# 101. EXACT

Compara prácticamente toda propiedad estructural soportada:

```text
names
order
platform options
metadata selected
```

---

# 102. STRUCTURAL

Compara estructura relevante sin metadata puramente informacional.

---

# 103. SEMANTIC

Compara aquello que afecta comportamiento lógico.

Puede ignorar:

```text
generated physical names
comments
some physical options
```

---

# 104. PORTABLE

Solo considera propiedades representables dentro del núcleo portable.

---

# 105. PLATFORM_NORMALIZED

Considera equivalencias propias de una plataforma concreta.

---

# 106. MIGRATION

Perfil recomendado para generar diferencias que potencialmente alimentarán Migration Planning.

Será conservador con:

```text
unknown
partial coverage
rename inference
destructive changes
```

---

# 107. Comparison profile ≠ migration policy

Aunque exista perfil `MIGRATION`, este solo define comparación.

No autoriza ejecutar cambios.

---

# 108. Normalization before diff

Pipeline:

```text
Current
   │
   ▼
Normalize(Current)
   │
   ├───────────────┐
   │               │
   ▼               ▼
Compare        Normalize(Target)
   ▲               │
   └───────────────┘
```

---

# 109. Same normalization context

Current y Target deberán normalizarse bajo contexto compatible.

No comparar:

```text
Current normalized for MySQL
```

contra:

```text
Target normalized for PostgreSQL
```

sin una política de portabilidad explícita.

---

# 110. Normalization does not erase intent

No deberá convertir:

```text
explicit target name
```

en:

```text
generated name
```

si esa diferencia importa al perfil.

---

# 111. Platform normalization

Ejemplo:

```text
INTEGER
INT
```

podrán canonicalizarse.

Pero:

```text
VARCHAR(255)
TEXT
```

no deberán considerarse iguales solo porque ambos acepten strings.

---

# 112. Diff phases

Se propone:

```text
1. Input validation
2. Coverage validation
3. Canonicalization
4. Object indexing
5. Exact matching
6. Semantic matching
7. Rename candidate discovery
8. Property comparison
9. Metadata comparison
10. Cross-object analysis
11. Impact classification
12. Dependency extraction
13. Safety analysis
14. Diagnostics
15. Canonical ordering
16. Seal SchemaDiff
```

---

# 113. Input validation

Verificará:

```text
schema compatibility
comparison profile
platform context
coverage
extension registry compatibility
normalization version
```

---

# 114. Coverage validation

Antes de `REMOVED`:

```text
relevant target/current completeness
```

deberá evaluarse.

---

# 115. Object indexing

Se construirán índices como:

```text
QualifiedPath → Object
NormalizedIdentifier → CandidateSet
Fingerprint → CandidateSet
ObjectKind → Objects
```

---

# 116. Cross-object analysis

Después de diferencias locales se analizarán dependencias.

Ejemplo:

```text
Column users.id removed
```

puede afectar:

```text
Primary Key
Foreign Keys
Indexes
Generated expressions
```

---

# 117. No duplicate derived differences

Si una PK desaparece porque desaparece la tabla completa:

```text
Table REMOVED
```

no es necesario emitir cada subobjeto como top-level removal salvo que el perfil lo solicite.

---

# 118. Difference collapsing

Se propone:

```text
SchemaDifferenceAggregationPolicy
```

para decidir cómo representar diferencias contenidas.

---

# 119. Compact mode

```text
Table REMOVED
```

puede ser suficiente.

---

# 120. Detailed mode

Puede incluir:

```text
Table REMOVED
├── Columns removed
├── Indexes removed
├── Constraints removed
└── FKs removed
```

para diagnostics/tooling.

---

# 121. Dependency graph

Schema Diff podrá derivar:

```text
SchemaDifferenceDependencyGraph
```

---

# 122. Dependency examples

```text
Drop FK
    before
Drop referenced column
```

o:

```text
Create table
    before
Create FK
```

Pero esto todavía no constituye un execution plan.

---

# 123. Diff dependency graph ≠ migration plan

```text
DifferenceDependencyGraph
```

solo expresa relaciones lógicas entre diferencias.

El planner posterior decide operaciones.

---

# 124. Rename dependencies

Si se confirma:

```text
users.id → users.user_id
```

las referencias dependientes deberán poder vincularse al mismo cambio de identidad.

---

# 125. Object evolution map

Se propone:

```text
SchemaObjectEvolutionMap
```

capaz de representar:

```text
OldPath → NewPath
```

con evidencia.

---

# 126. Evolution map sources

Podrán ser:

```text
explicit schema history
migration metadata
developer hints
rename inference
extension evidence
```

---

# 127. Explicit evidence priority

Prioridad:

```text
Explicit evolution mapping
>
stable identity evidence
>
high-confidence structural inference
>
heuristic similarity
```

---

# 128. Rename heuristic limits

No deberá inferirse rename si:

```text
multiple equally plausible candidates
```

---

# 129. Similarity scoring

Podrá existir:

```text
RenameSimilarityScore
```

pero será:

```text
supporting evidence
```

no verdad.

---

# 130. Example score

Conceptualmente:

```text
score =
typeSimilarity      * w1 +
constraintSimilarity* w2 +
positionSimilarity  * w3 +
metadataSimilarity  * w4
```

sin convertir automáticamente cualquier score alto en rename.

---

# 131. Determinism

El mismo input + contexto deberá producir:

```text
same SchemaDiff
```

---

# 132. Deterministic candidate resolution

Empates no se resolverán mediante:

```text
first iteration order
```

Se marcarán ambiguos.

---

# 133. Canonical ordering

El diff final deberá tener orden estable:

```text
object kind
qualified path
difference kind
property path
```

o esquema equivalente documentado.

---

# 134. SchemaDiff immutability

El resultado final será:

```text
immutable
sealed
```

---

# 135. SchemaDiff identity

Podrá existir:

```text
SchemaDiffFingerprint
```

---

# 136. Fingerprint inputs

Podrá considerar:

```text
current structural fingerprint
target structural fingerprint
comparison profile
platform capability fingerprint
normalization version
diff algorithm version
```

---

# 137. Runtime IDs

No deberán afectar el fingerprint semántico.

---

# 138. Diff serialization

SchemaDiff podrá serializarse para:

```text
preview
CI
developer tooling
migration review
audit
```

---

# 139. Serialization requirements

```text
versioned
deterministic
typed
extension-aware
```

---

# 140. No executable serialization

Serializar un diff no deberá convertirlo en:

```text
executable migration
```

---

# 141. Schema drift

Schema Diff podrá utilizarse para detectar:

```text
Schema Drift
```

---

# 142. Drift model

```text
Expected Schema
      │
      ▼
Schema Diff
      ▲
      │
Observed Schema
```

Resultado:

```text
SchemaDriftReport
```

---

# 143. Drift ≠ migration

Detectar drift no implica corregirlo automáticamente.

---

# 144. Drift classifications

```text
EXPECTED_DIFFERENCE
UNEXPECTED_DIFFERENCE
INFORMATIONAL
PLATFORM_NORMALIZATION
UNKNOWN
CRITICAL
```

según policy superior.

---

# 145. Diff and migration generation

Pipeline correcto:

```text
SchemaDiff
    │
    ▼
SchemaChangePlanner
    │
    ▼
Schema Change Strategy
    │
    ▼
Schema AST
    │
    ▼
Schema Compiler
```

No:

```text
SchemaDiff
    ↓
SQL
```

---

# 146. Change planner

Aunque el documento específico se desarrolla posteriormente dentro de Migrations, conceptualmente deberá existir una capa que traduzca:

```text
difference
```

en:

```text
intentional operations
```

---

# 147. Example: added column

Diff:

```text
Column ADDED
users.email_verified_at
```

Planner puede generar:

```text
AddColumnNode
```

---

# 148. Example: rename candidate

Diff:

```text
Possible Rename
users.full_name
→
users.name
confidence = HIGH
```

Planner no deberá generar automáticamente:

```text
RenameColumnNode
```

si policy requiere explicit rename evidence.

---

# 149. Example: type narrowing

Current:

```text
VARCHAR(500)
```

Target:

```text
VARCHAR(100)
```

Diff:

```text
Column Type/Length Difference
impact = POTENTIALLY_DESTRUCTIVE
```

Planner decide estrategia.

---

# 150. Example: nullable tightening

```text
NULL
→
NOT NULL
```

Diff:

```text
NullabilityDifference
impact = DATA_AFFECTING
safety = REQUIRES_VALIDATION
```

No intenta consultar:

```text
SELECT COUNT(*) WHERE column IS NULL
```

porque eso sería data inspection fuera de Schema Diff.

---

# 151. Data awareness boundary

Schema Diff trabaja con:

```text
schema structure
schema metadata
```

No con:

```text
row data
```

---

# 152. Data validation requirements

Puede producir:

```text
DataValidationRequirement
```

como hint para capas posteriores.

Ejemplo:

```text
Before setting NOT NULL:
verify no NULL values
```

---

# 153. Requirement ≠ execution

Schema Diff puede declarar:

```text
requires data validation
```

pero no ejecutarla.

---

# 154. Safety requirements

Se propone:

```text
SchemaDifferenceRequirement
├── DATA_VALIDATION
├── EXCLUSIVE_CAPABILITY
├── PLATFORM_SUPPORT
├── EXPLICIT_RENAME_CONFIRMATION
├── COMPLETE_COVERAGE
├── REFERENCE_RESOLUTION
└── MANUAL_REVIEW
```

---

# 155. Platform capabilities

Schema Diff podrá usar capabilities para interpretar equivalencias.

Ejemplo:

```text
supports expression indexes
supports included columns
supports deferrable constraints
```

---

# 156. Capabilities ≠ SQL generation

El differ puede decir:

```text
property not meaningful on this platform
```

pero no genera sintaxis.

---

# 157. Unsupported target feature

Si Target contiene feature no soportada:

```text
SchemaDifference
```

puede marcar:

```text
PLATFORM_DEPENDENT
```

o producir diagnostic.

Capability validation completa puede pertenecer a fases posteriores.

---

# 158. Cross-platform diff

Podrá existir para tooling.

Pero deberá ser explícito:

```text
CrossPlatformSchemaDiffProfile
```

---

# 159. Cross-platform limitation

Comparar:

```text
MySQL observed schema
```

con:

```text
PostgreSQL target schema
```

no deberá utilizar automáticamente equivalencia física.

Se utilizará:

```text
portable semantic projection
```

cuando corresponda.

---

# 160. Extension model

Se propone:

```text
SchemaDiffExtension
```

para:

```text
custom object differ
custom metadata differ
custom comparison rules
custom normalization
custom impact classifier
```

---

# 161. Frozen registry

```text
SchemaDiffExtensionRegistry
```

será frozen tras bootstrap.

---

# 162. Extension restrictions

Una extensión no podrá:

- ejecutar SQL;
- abrir conexiones;
- introspectar;
- autorizar cambios;
- esconder diferencias críticas;
- transformar UNKNOWN en EQUAL sin evidencia;
- modificar snapshots;
- utilizar mutable global state.

---

# 163. Rule registration

Las reglas deberán tener:

```text
RuleId
Priority
Dependencies
SupportedObjectKinds
ComparisonProfiles
```

---

# 164. No last wins

Dos reglas incompatibles deberán producir conflicto de configuración.

No:

```text
last registered rule wins
```

---

# 165. Diff rule engine

Podrá existir:

```text
SchemaDiffRuleEngine
```

pero no deberá convertirse en sistema de reescritura arbitraria.

---

# 166. Rule purity

Idealmente:

```text
Rule(Input, Context)
→
ComparisonResult
```

sin side effects.

---

# 167. Budgets

Se propone:

```php
final readonly class SchemaDiffBudget
{
    public function __construct(
        public int $maxObjects,
        public int $maxDifferences,
        public int $maxRenameCandidates,
        public int $maxComparisonOperations,
        public int $maxExpressionComparisons,
        public int $maxDependencyEdges,
        public int $maxDiagnostics,
    ) {}
}
```

---

# 168. Budget exhaustion

Nunca:

```text
silently return partial diff as complete
```

Debe:

```text
fail
```

o producir:

```text
PARTIAL_DIFF
+
diagnostics
```

según policy.

---

# 169. Diff completeness

Se propone:

```text
SchemaDiffCompleteness
├── COMPLETE
├── PARTIAL
└── UNKNOWN
```

---

# 170. Partial diff

Un diff parcial no podrá utilizarse como fuente segura para cambios destructivos sin una capa superior que lo autorice explícitamente.

---

# 171. Cancellation

Para schemas grandes podrá soportarse:

```text
CancellationToken
Deadline
```

---

# 172. Cancellation result

Si se devuelve resultado parcial:

```text
completeness = PARTIAL
```

obligatoriamente.

---

# 173. Performance model

Para:

```text
n = current objects
m = target objects
```

se buscará evitar:

```text
O(n × m)
```

como estrategia general.

---

# 174. Indexed matching

Con índices:

```text
QualifiedPath → Object
Fingerprint → CandidateSet
ObjectKind → ObjectSet
```

matching normal deberá aproximarse a:

```text
O(n + m)
```

más costos de rename analysis.

---

# 175. Rename analysis complexity

La comparación de todos contra todos puede ser costosa.

Se utilizarán candidate filters:

```text
same object kind
compatible type family
same namespace/table
similar structural signature
compatible arity
```

antes del scoring.

---

# 176. Fingerprint acceleration

Ejemplo:

```text
Column structural signature
=
type family
+
nullability
+
generation category
+
constraint participation
```

para reducir candidatos.

---

# 177. Memory

No duplicar objetos completos dentro de cada difference.

Preferir:

```text
immutable references
object paths
compact snapshots
```

cuando sea seguro.

---

# 178. Persistent runtime safety

Shared:

```text
Frozen diff registry
Comparison rules
Normalizers
Comparators
```

Operation-scoped:

```text
SchemaDiffSession
Object indexes
Rename candidates
Diagnostics
Temporary graphs
```

---

# 179. FrankenPHP

Cada operación:

```text
Request A → DiffSession A → dispose
Request B → DiffSession B
```

---

# 180. RoadRunner/OpenSwoole

Misma regla:

```text
no cross-request mutable diff state
```

---

# 181. Multitenancy

Schema Diff no necesita depender del paquete Multitenancy.

Puede recibir:

```text
Tenant-specific Current Snapshot
+
Tenant-specific Target Schema
```

desde la integración correspondiente.

---

# 182. Tenant isolation

Nunca comparar accidentalmente:

```text
Tenant A current
```

contra:

```text
Tenant B target
```

El contexto superior deberá mantener identidad/scope.

Opcionalmente podrá existir un guard:

```text
SchemaScopeIdentity
```

---

# 183. Scope identity

Se propone:

```text
SchemaScopeIdentity
```

para representar:

```text
connection logical identity
database
namespace
tenant integration scope
```

sin acoplar core a Multitenancy.

---

# 184. Security

Aunque Schema Diff no ejecute SQL, puede procesar:

```text
sensitive identifiers
view definitions
check expressions
comments
platform metadata
```

---

# 185. Safe diagnostics

Diagnostics deberán soportar redaction.

---

# 186. Telemetry

Podrán medirse:

```text
diff duration
objects compared
matched objects
added objects
removed objects
modified objects
rename candidates
unknown comparisons
incomparable comparisons
diagnostic count
```

---

# 187. Telemetry privacy

No registrar por defecto:

```text
full schema
full view SQL
raw expressions
sensitive comments
```

---

# 188. Error hierarchy

Se propone:

```text
DatabaseSchemaDiffException
├── InvalidSchemaDiffInputException
├── IncompatibleSchemaDiffContextException
├── SchemaDiffCoverageException
├── SchemaDiffNormalizationException
├── SchemaObjectMatchingException
├── AmbiguousSchemaObjectMatchException
├── SchemaComparisonException
├── SchemaRenameAnalysisException
├── SchemaDifferenceClassificationException
├── SchemaDiffDependencyException
├── SchemaDiffExtensionException
├── SchemaDiffSerializationException
├── SchemaDiffBudgetExceededException
├── SchemaDiffCancelledException
└── SchemaDiffInvariantException
```

---

# 189. Diagnostics

Ejemplos:

```text
SCHEMA_DIFF_PARTIAL_COVERAGE
SCHEMA_DIFF_UNKNOWN_CURRENT_VALUE
SCHEMA_DIFF_INCOMPARABLE_TYPE
SCHEMA_DIFF_AMBIGUOUS_RENAME
SCHEMA_DIFF_UNRESOLVED_REFERENCE
SCHEMA_DIFF_PLATFORM_DEPENDENT
SCHEMA_DIFF_OPAQUE_EXPRESSION
SCHEMA_DIFF_UNSUPPORTED_EXTENSION
SCHEMA_DIFF_BUDGET_EXCEEDED
```

---

# 190. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Diff
```

---

# 191. Proposed directory structure

```text
Schema/
└── Diff/
    ├── Contract/
    │   ├── SchemaDiffer.php
    │   ├── SchemaObjectMatcher.php
    │   ├── SchemaObjectDiffer.php
    │   └── SchemaDifferenceClassifier.php
    │
    ├── Core/
    │   ├── SchemaDiff.php
    │   ├── SchemaDiffContext.php
    │   ├── SchemaDiffPolicy.php
    │   ├── SchemaDiffCompleteness.php
    │   └── SchemaDifferenceKind.php
    │
    ├── Comparison/
    │   ├── SchemaComparisonResult.php
    │   ├── SchemaComparisonProfile.php
    │   ├── SchemaComparisonContext.php
    │   └── SchemaDifferenceEvidence.php
    │
    ├── Matching/
    │   ├── SchemaObjectMatcher.php
    │   ├── SchemaObjectMatch.php
    │   ├── SchemaObjectMatchResult.php
    │   ├── SchemaObjectMatchIndex.php
    │   └── SchemaObjectEvolutionMap.php
    │
    ├── Rename/
    │   ├── RenameCandidate.php
    │   ├── TableRenameCandidate.php
    │   ├── ColumnRenameCandidate.php
    │   ├── RenameConfidence.php
    │   ├── RenameSimilarityScore.php
    │   └── SchemaRenameAnalyzer.php
    │
    ├── Table/
    │   ├── TableDiffer.php
    │   ├── TableDifference.php
    │   └── TableDifferenceSet.php
    │
    ├── Column/
    │   ├── ColumnDiffer.php
    │   ├── ColumnDifference.php
    │   ├── ColumnDifferenceSet.php
    │   └── ColumnPropertyDifference.php
    │
    ├── Index/
    │   ├── IndexDiffer.php
    │   ├── IndexDifference.php
    │   ├── IndexDifferenceSet.php
    │   └── IndexNameComparison.php
    │
    ├── Constraint/
    │   ├── ConstraintDiffer.php
    │   ├── ConstraintDifference.php
    │   └── ConstraintDifferenceSet.php
    │
    ├── ForeignKey/
    │   ├── ForeignKeyDiffer.php
    │   ├── ForeignKeyDifference.php
    │   └── ForeignKeyDifferenceSet.php
    │
    ├── Sequence/
    │   ├── SequenceDiffer.php
    │   └── SequenceDifference.php
    │
    ├── View/
    │   ├── ViewDiffer.php
    │   └── ViewDifference.php
    │
    ├── Property/
    │   ├── PropertyDifference.php
    │   ├── PropertyDifferenceKind.php
    │   └── PropertyPath.php
    │
    ├── Impact/
    │   ├── SchemaDifferenceImpact.php
    │   ├── SchemaDifferenceSafety.php
    │   ├── SchemaDifferenceRequirement.php
    │   └── SchemaDifferenceClassifier.php
    │
    ├── Dependency/
    │   ├── SchemaDifferenceDependencyGraph.php
    │   ├── SchemaDifferenceDependencyEdge.php
    │   └── SchemaDifferenceDependencyAnalyzer.php
    │
    ├── Normalization/
    │   ├── SchemaDiffNormalizer.php
    │   └── SchemaDiffNormalizationContext.php
    │
    ├── Rule/
    │   ├── SchemaDiffRule.php
    │   ├── SchemaDiffRuleId.php
    │   ├── SchemaDiffRuleEngine.php
    │   └── SchemaDiffRuleRegistry.php
    │
    ├── Extension/
    │   ├── SchemaDiffExtension.php
    │   └── FrozenSchemaDiffExtensionRegistry.php
    │
    ├── Fingerprint/
    │   └── SchemaDiffFingerprint.php
    │
    ├── Serialization/
    │   ├── SchemaDiffSerializer.php
    │   └── SchemaDiffDeserializer.php
    │
    ├── Budget/
    │   └── SchemaDiffBudget.php
    │
    ├── Diagnostic/
    │   ├── SchemaDiffDiagnostic.php
    │   └── SchemaDiffDiagnosticSet.php
    │
    └── Exception/
        ├── DatabaseSchemaDiffException.php
        ├── InvalidSchemaDiffInputException.php
        ├── IncompatibleSchemaDiffContextException.php
        ├── SchemaDiffCoverageException.php
        ├── SchemaDiffNormalizationException.php
        ├── SchemaObjectMatchingException.php
        ├── AmbiguousSchemaObjectMatchException.php
        ├── SchemaComparisonException.php
        ├── SchemaRenameAnalysisException.php
        ├── SchemaDifferenceClassificationException.php
        ├── SchemaDiffDependencyException.php
        ├── SchemaDiffExtensionException.php
        ├── SchemaDiffSerializationException.php
        ├── SchemaDiffBudgetExceededException.php
        ├── SchemaDiffCancelledException.php
        └── SchemaDiffInvariantException.php
```

---

# 192. Example: simple table diff

Current:

```text
users
├── id BIGINT NOT NULL
├── email VARCHAR(255) NOT NULL
└── PK(id)
```

Target:

```text
users
├── id BIGINT NOT NULL
├── email VARCHAR(320) NOT NULL
├── active BOOLEAN NOT NULL DEFAULT true
└── PK(id)
```

Result:

```text
SchemaDiff
└── TableDifference(users)
    ├── kind = MODIFIED
    │
    ├── ColumnDifference(email)
    │   ├── kind = MODIFIED
    │   └── length
    │       ├── current = 255
    │       └── target = 320
    │
    └── ColumnDifference(active)
        ├── kind = ADDED
        └── target
            ├── BOOLEAN
            ├── NOT NULL
            └── DEFAULT true
```

No SQL ha sido generado.

---

# 193. Example: destructive possibility

Current:

```text
users.bio TEXT
```

Target:

```text
users.bio VARCHAR(100)
```

Resultado:

```text
ColumnDifference
├── type = MODIFIED
├── current = TEXT
├── target = VARCHAR(100)
├── impact = POTENTIALLY_DESTRUCTIVE
├── safety = REQUIRES_VALIDATION
└── requirements
    └── DATA_VALIDATION
```

No ejecuta:

```text
SELECT MAX(LENGTH(bio))
```

ni:

```text
ALTER TABLE
```

---

# 194. Example: partial coverage

Snapshot:

```text
scope:
users table only

coverage:
COMPLETE within users
```

Target:

```text
users
orders
```

El diff puede detectar diferencias dentro de:

```text
users
```

pero no afirmar:

```text
orders = ADDED
```

si el current scope no cubría `orders`.

Resultado:

```text
orders
└── comparison = UNKNOWN
    reason = OUTSIDE_CURRENT_COVERAGE
```

---

# 195. Example: rename ambiguity

Current:

```text
users
├── first_name VARCHAR(100)
└── last_name VARCHAR(100)
```

Target:

```text
users
├── display_name VARCHAR(100)
└── legal_name VARCHAR(100)
```

Resultado:

```text
Rename Analysis
├── first_name
│   ├── display_name = candidate
│   └── legal_name = candidate
│
└── last_name
    ├── display_name = candidate
    └── legal_name = candidate

status = AMBIGUOUS
```

No:

```text
first_name → display_name
last_name → legal_name
```

por orden arbitrario.

---

# 196. Example: FK change

Current:

```text
orders.customer_id
    ↓
customers.id
ON DELETE RESTRICT
```

Target:

```text
orders.customer_id
    ↓
customers.id
ON DELETE CASCADE
```

Resultado:

```text
ForeignKeyDifference
├── kind = MODIFIED
└── deleteAction
    ├── current = RESTRICT
    └── target = CASCADE
```

El cambio es semánticamente importante aunque columnas y target sean iguales.

---

# 197. Example: metadata-only change

Current:

```text
users comment:
"Application users"
```

Target:

```text
users comment:
"Registered users"
```

Resultado bajo `STRUCTURAL`:

```text
possibly ignored
```

Resultado bajo perfil que sincroniza comments:

```text
MetadataDifference
├── kind = MODIFIED
├── category = INFORMATIONAL
└── key = schema.comment
```

---

# 198. Example: generated index names

Current:

```text
idx_users_email_91f2
ON users(email)
```

Target:

```text
users_email_index
ON users(email)
```

Bajo:

```text
EXACT
```

puede existir diferencia de nombre.

Bajo:

```text
SEMANTIC
+
IGNORE_GENERATED_NAMES
```

pueden ser equivalentes.

---

# 199. Example: unknown current metadata

Current:

```text
users.email collation = UNKNOWN
```

Target:

```text
users.email collation = utf8mb4_unicode_ci
```

Resultado:

```text
SchemaComparisonResult::UNKNOWN
```

No:

```text
CHANGED
```

automáticamente.

---

# 200. Schema Diff invariants

## DB-SCHEMA-DIFF-001
Schema Diff será distinto de Schema AST.

## DB-SCHEMA-DIFF-002
Schema Diff será distinto de Migration.

## DB-SCHEMA-DIFF-003
Schema Diff será distinto de SQL.

## DB-SCHEMA-DIFF-004
Schema Diff será distinto de Execution.

## DB-SCHEMA-DIFF-005
Schema Diff descubrirá diferencias, no ejecutará cambios.

## DB-SCHEMA-DIFF-006
Difference será distinta de Change Operation.

## DB-SCHEMA-DIFF-007
Difference será distinta de authorization.

## DB-SCHEMA-DIFF-008
Diff será direccional.

## DB-SCHEMA-DIFF-009
Current será distinto de Target.

## DB-SCHEMA-DIFF-010
Missing será distinto de Removed.

## DB-SCHEMA-DIFF-011
NotObserved será distinto de Absent.

## DB-SCHEMA-DIFF-012
Partial coverage no probará ausencia.

## DB-SCHEMA-DIFF-013
Removed requerirá evidencia suficiente.

## DB-SCHEMA-DIFF-014
Unknown será first-class.

## DB-SCHEMA-DIFF-015
Incomparable será first-class.

## DB-SCHEMA-DIFF-016
Comparison no será únicamente booleana.

## DB-SCHEMA-DIFF-017
EQUAL será distinto de UNKNOWN.

## DB-SCHEMA-DIFF-018
DIFFERENT será distinto de INCOMPARABLE.

## DB-SCHEMA-DIFF-019
Rename será distinto de Drop + Add.

## DB-SCHEMA-DIFF-020
Rename candidate será distinto de confirmed rename.

## DB-SCHEMA-DIFF-021
Similarity será distinta de identity.

## DB-SCHEMA-DIFF-022
Rename inference será conservadora.

## DB-SCHEMA-DIFF-023
Ambiguous rename no se resolverá arbitrariamente.

## DB-SCHEMA-DIFF-024
Explicit rename evidence tendrá prioridad.

## DB-SCHEMA-DIFF-025
Fingerprint no probará identidad por sí solo.

## DB-SCHEMA-DIFF-026
Object matching será determinista.

## DB-SCHEMA-DIFF-027
Qualified identifiers serán estructurados.

## DB-SCHEMA-DIFF-028
Identifier comparison será platform-aware.

## DB-SCHEMA-DIFF-029
Blind lowercase matching estará prohibido.

## DB-SCHEMA-DIFF-030
Table matching será distinto de column matching.

## DB-SCHEMA-DIFF-031
Index name identity será distinta de semantic index identity.

## DB-SCHEMA-DIFF-032
Constraint name equality será distinta de semantic equality.

## DB-SCHEMA-DIFF-033
Primary Key no será comparada únicamente como index.

## DB-SCHEMA-DIFF-034
Foreign Key semantic comparison incluirá actions.

## DB-SCHEMA-DIFF-035
Unresolved FK reference no será tratada como resolved.

## DB-SCHEMA-DIFF-036
External snapshot reference no será automáticamente inválida.

## DB-SCHEMA-DIFF-037
Column type comparison utilizará Database Type System.

## DB-SCHEMA-DIFF-038
Native SQL type strings no serán la única base de comparación.

## DB-SCHEMA-DIFF-039
NoDefault será distinto de DefaultNull.

## DB-SCHEMA-DIFF-040
Unknown default será distinto de NoDefault.

## DB-SCHEMA-DIFF-041
Schema expressions tendrán comparador estructurado.

## DB-SCHEMA-DIFF-042
Opaque expressions podrán resultar INCOMPARABLE.

## DB-SCHEMA-DIFF-043
Diff no será theorem prover general.

## DB-SCHEMA-DIFF-044
Column order policy será explícita.

## DB-SCHEMA-DIFF-045
Index key order será semánticamente preservado.

## DB-SCHEMA-DIFF-046
UniqueConstraint será distinta de UniqueIndex.

## DB-SCHEMA-DIFF-047
Metadata Difference será distinta de Structural Difference.

## DB-SCHEMA-DIFF-048
Informational Difference será distinta de Semantic Difference.

## DB-SCHEMA-DIFF-049
Physical Difference será distinta de Semantic Difference.

## DB-SCHEMA-DIFF-050
Impact classification será distinta de authorization.

## DB-SCHEMA-DIFF-051
Impact classification será distinta de migration strategy.

## DB-SCHEMA-DIFF-052
Unknown impact no será SAFE automáticamente.

## DB-SCHEMA-DIFF-053
Difference confidence será explícita cuando sea necesario.

## DB-SCHEMA-DIFF-054
Low confidence no será promoted silenciosamente.

## DB-SCHEMA-DIFF-055
Comparison profile será explícito.

## DB-SCHEMA-DIFF-056
Exact profile será distinto de Structural profile.

## DB-SCHEMA-DIFF-057
Structural profile será distinto de Semantic profile.

## DB-SCHEMA-DIFF-058
Portable profile será distinto de Platform-Normalized profile.

## DB-SCHEMA-DIFF-059
Migration comparison profile no autorizará migration.

## DB-SCHEMA-DIFF-060
Normalization precederá comparaciones que la requieran.

## DB-SCHEMA-DIFF-061
Normalization será determinista.

## DB-SCHEMA-DIFF-062
Normalization no realizará hidden DB I/O.

## DB-SCHEMA-DIFF-063
Current y Target usarán contextos de normalización compatibles.

## DB-SCHEMA-DIFF-064
Normalization no borrará explicit intent relevante.

## DB-SCHEMA-DIFF-065
Diff input validation será explícita.

## DB-SCHEMA-DIFF-066
Coverage validation precederá destructive absence classification.

## DB-SCHEMA-DIFF-067
Object indexes serán operation-scoped.

## DB-SCHEMA-DIFF-068
Cross-object analysis será posterior a local comparison cuando corresponda.

## DB-SCHEMA-DIFF-069
Contained differences no deberán duplicarse sin policy.

## DB-SCHEMA-DIFF-070
Difference aggregation policy será explícita.

## DB-SCHEMA-DIFF-071
Dependency graph será derivado.

## DB-SCHEMA-DIFF-072
Difference dependency graph será distinto de migration plan.

## DB-SCHEMA-DIFF-073
Evolution map conservará evidence.

## DB-SCHEMA-DIFF-074
Rename similarity score será supporting evidence.

## DB-SCHEMA-DIFF-075
Candidate tie será ambiguous.

## DB-SCHEMA-DIFF-076
SchemaDiff será immutable.

## DB-SCHEMA-DIFF-077
SchemaDiff será sealed.

## DB-SCHEMA-DIFF-078
SchemaDiff fingerprint será versionado.

## DB-SCHEMA-DIFF-079
Runtime IDs no afectarán semantic fingerprint.

## DB-SCHEMA-DIFF-080
SchemaDiff serialization será determinista.

## DB-SCHEMA-DIFF-081
Serialized diff no será executable migration.

## DB-SCHEMA-DIFF-082
Schema Drift será distinto de Migration.

## DB-SCHEMA-DIFF-083
Drift detection no corregirá automáticamente.

## DB-SCHEMA-DIFF-084
SchemaDiff no generará SQL.

## DB-SCHEMA-DIFF-085
SchemaDiff no ejecutará SQL.

## DB-SCHEMA-DIFF-086
SchemaDiff no abrirá conexiones.

## DB-SCHEMA-DIFF-087
SchemaDiff no introspectará ocultamente.

## DB-SCHEMA-DIFF-088
SchemaDiff no iniciará transacciones.

## DB-SCHEMA-DIFF-089
SchemaDiff no consultará row data.

## DB-SCHEMA-DIFF-090
Data validation requirements serán declarativos.

## DB-SCHEMA-DIFF-091
Data validation requirements no serán ejecutados por Diff.

## DB-SCHEMA-DIFF-092
Capability analysis será explícito.

## DB-SCHEMA-DIFF-093
Capabilities no generarán SQL dentro del differ.

## DB-SCHEMA-DIFF-094
Cross-platform diff requerirá perfil explícito.

## DB-SCHEMA-DIFF-095
Cross-platform comparison utilizará portable projection cuando corresponda.

## DB-SCHEMA-DIFF-096
Extensions serán registradas explícitamente.

## DB-SCHEMA-DIFF-097
Extension registry será frozen.

## DB-SCHEMA-DIFF-098
Extensions no ejecutarán SQL.

## DB-SCHEMA-DIFF-099
Extensions no abrirán conexiones.

## DB-SCHEMA-DIFF-100
Extensions no ocultarán critical differences.

## DB-SCHEMA-DIFF-101
Extensions no convertirán UNKNOWN en EQUAL sin evidencia.

## DB-SCHEMA-DIFF-102
Diff rules serán pure cuando sea posible.

## DB-SCHEMA-DIFF-103
Rule conflicts no usarán last-wins.

## DB-SCHEMA-DIFF-104
Budgets serán explícitos.

## DB-SCHEMA-DIFF-105
Budget exhaustion no producirá silent complete diff.

## DB-SCHEMA-DIFF-106
Diff completeness será explícita.

## DB-SCHEMA-DIFF-107
Partial diff será distinto de complete diff.

## DB-SCHEMA-DIFF-108
Cancelled diff no será marcado complete.

## DB-SCHEMA-DIFF-109
Matching evitará O(n×m) como estrategia general.

## DB-SCHEMA-DIFF-110
Rename candidate search será filtrado.

## DB-SCHEMA-DIFF-111
Fingerprints podrán acelerar matching.

## DB-SCHEMA-DIFF-112
Fingerprints no sustituirán equality verification.

## DB-SCHEMA-DIFF-113
Shared diff services serán immutable.

## DB-SCHEMA-DIFF-114
Mutable DiffSession será operation-scoped.

## DB-SCHEMA-DIFF-115
No existirá global current diff.

## DB-SCHEMA-DIFF-116
No existirá global current schema.

## DB-SCHEMA-DIFF-117
No existirá global current tenant.

## DB-SCHEMA-DIFF-118
FrankenPHP request isolation será obligatoria.

## DB-SCHEMA-DIFF-119
RoadRunner worker isolation será obligatoria.

## DB-SCHEMA-DIFF-120
OpenSwoole coroutine isolation será obligatoria.

## DB-SCHEMA-DIFF-121
Multitenancy será integración opcional.

## DB-SCHEMA-DIFF-122
Tenant scopes no se mezclarán.

## DB-SCHEMA-DIFF-123
Schema scope identity podrá proteger comparaciones.

## DB-SCHEMA-DIFF-124
Sensitive identifiers podrán redactarse.

## DB-SCHEMA-DIFF-125
Telemetry no registrará full schema por defecto.

## DB-SCHEMA-DIFF-126
Diagnostics serán estructurados.

## DB-SCHEMA-DIFF-127
Errors serán tipados.

## DB-SCHEMA-DIFF-128
Unknown metadata no será descartada silenciosamente.

## DB-SCHEMA-DIFF-129
Unknown platform feature no será false-equivalent.

## DB-SCHEMA-DIFF-130
Unsupported comparison será explícita.

## DB-SCHEMA-DIFF-131
Table removal podrá subsumir child removals según aggregation policy.

## DB-SCHEMA-DIFF-132
Detailed mode podrá preservar child differences.

## DB-SCHEMA-DIFF-133
Difference ordering será determinista.

## DB-SCHEMA-DIFF-134
Property comparison será typed para propiedades críticas.

## DB-SCHEMA-DIFF-135
Generic array diff no será arquitectura principal.

## DB-SCHEMA-DIFF-136
Schema metadata comparison reutilizará Metadata System.

## DB-SCHEMA-DIFF-137
Database type comparison reutilizará Type System.

## DB-SCHEMA-DIFF-138
Schema expression comparison reutilizará Schema Expression System.

## DB-SCHEMA-DIFF-139
Schema object references permanecerán estructuradas.

## DB-SCHEMA-DIFF-140
Diff no inventará object identity.

## DB-SCHEMA-DIFF-141
Diff no inventará rename identity.

## DB-SCHEMA-DIFF-142
Diff no inventará certainty.

## DB-SCHEMA-DIFF-143
Diff no inventará absence.

## DB-SCHEMA-DIFF-144
Diff no inventará platform equivalence.

## DB-SCHEMA-DIFF-145
Diff no convertirá uncertainty en destructive intent.

## DB-SCHEMA-DIFF-146
Detected Difference será distinta de Authorized Change.

## DB-SCHEMA-DIFF-147
Authorized Change será distinta de Executed Change.

## DB-SCHEMA-DIFF-148
Schema Diff preservará evidencia suficiente para explicar decisiones.

## DB-SCHEMA-DIFF-149
Toda destructive classification deberá ser trazable.

## DB-SCHEMA-DIFF-150
Schema Diff describirá diferencias sin decidir cómo ejecutarlas físicamente.

---

# 201. Anti-patterns

## 201.1 Diff directo a SQL

Incorrecto:

```php
$sql = $schemaDiffer->diff($current, $target);
```

si devuelve:

```sql
ALTER TABLE ...
```

Correcto:

```text
Diff
→
Change Planning
→
AST
→
Compiler
→
SQL
```

---

# 202. Missing = drop

Incorrecto:

```php
if (!$target->hasTable($name)) {
    $diff->dropTable($name);
}
```

sin comprobar scope/completeness.

---

# 203. Automatic rename by similarity

Incorrecto:

```php
if ($similarity > 0.8) {
    return new RenameColumn(...);
}
```

La similitud solo es evidencia.

---

# 204. Array comparison

Incorrecto:

```php
array_diff_assoc(
    $current->toArray(),
    $target->toArray()
);
```

No entiende:

```text
semantics
unknowns
platform normalization
coverage
references
```

---

# 205. Raw SQL type comparison

Incorrecto:

```php
if ($currentTypeSql !== $targetTypeSql) {
    $changed = true;
}
```

---

# 206. Unknown = different

Incorrecto:

```php
if ($current === null && $target !== null) {
    return DIFFERENT;
}
```

si `null` representa metadata desconocida.

---

# 207. Unique index = constraint

Incorrecto:

```text
UNIQUE INDEX(email)
=
UNIQUE CONSTRAINT(email)
```

como regla universal.

---

# 208. Data queries in Diff

Incorrecto:

```php
if ($columnBecomesNotNull) {
    $count = $connection->query(
        'SELECT COUNT(*) ...'
    );
}
```

Schema Diff puede declarar requisito.

No ejecutarlo.

---

# 209. Platform if/else explosion

Incorrecto:

```php
if ($platform === 'mysql') { ... }
elseif ($platform === 'pgsql') { ... }
elseif ($platform === 'sqlite') { ... }
```

disperso por todo el differ.

Preferir:

```text
Capabilities
Comparators
Platform normalization services
```

---

# 210. Hidden state

Incorrecto:

```php
SchemaDiffer::$currentPlatform
SchemaDiffer::$currentTenant
SchemaDiffer::$currentSchema
```

---

# 211. Testing architecture

Debe existir:

```text
Schema Diff Unit Tests
Object Matching Tests
Rename Analysis Tests
Coverage Tests
Column Diff Tests
Index Diff Tests
Constraint Diff Tests
FK Diff Tests
Metadata Diff Tests
Platform Normalization Tests
Property-Based Tests
Integration Tests
Performance Tests
Persistent Runtime Tests
Security Tests
```

---

# 212. Fundamental tests

Casos mínimos:

```text
empty → empty
empty → table
table → empty
same table → same table
column added
column removed
column type changed
nullability changed
default changed
column renamed explicit
column rename ambiguous
index added
index removed
index changed
constraint added
PK changed
FK action changed
metadata changed
partial coverage
unknown type
opaque expression
generated names
cross-platform normalized equality
```

---

# 213. Property-based symmetry relation

Aunque el diff sea direccional:

```text
Diff(A, B)
```

y:

```text
Diff(B, A)
```

deberán tener relaciones coherentes.

Ejemplo:

```text
ADDED ↔ REMOVED
```

cuando ambos schemas sean completos.

---

# 214. Identity property

Para schema `S` completo:

```text
Diff(S, S)
=
EmptyDiff
```

bajo el mismo perfil/contexto.

---

# 215. Determinism property

```text
Diff(A, B, C)
=
Diff(A, B, C)
```

en ejecuciones repetidas con los mismos inputs.

---

# 216. Normalization property

Si:

```text
Normalize(A) = Normalize(B)
```

bajo el perfil aplicable:

```text
Diff(A, B)
=
EmptyDiff
```

salvo metadata deliberadamente exacta fuera de dicha normalización.

---

# 217. Serialization property

```text
Deserialize(Serialize(Diff))
≈
Diff
```

bajo la misma versión.

---

# 218. Performance tests

Schemas sintéticos:

```text
100 tables
1,000 tables
10,000 tables
large index sets
large FK graphs
many rename candidates
```

para detectar degradación accidental hacia:

```text
O(n²)
```

---

# 219. Master equations

## Diff

```text
SchemaDiff
=
Compare(
    Normalize(Current),
    Normalize(Target),
    Coverage,
    ComparisonProfile,
    Capabilities
)
```

---

## Safe absence

```text
SafeAbsence(O)
=
NotObserved(O)
∧
CompleteRelevantCoverage(O)
∧
SufficientVisibility(O)
∧
SuccessfulObservation
```

---

## Rename

```text
ConfirmedRename
≠
Similarity
```

Más correctamente:

```text
RenameCandidate
=
Similarity
+
Context
+
Evidence
```

mientras:

```text
ConfirmedRename
=
SufficientIdentityEvidence
```

---

## Difference safety

```text
SafeDifference
=
Comparable
∧
SufficientEvidence
∧
SufficientCoverage
∧
KnownSemantics
```

---

## Migration boundary

```text
SchemaDiff
≠
MigrationPlan
```

y:

```text
SchemaDiff
    ↓
Change Strategy
    ↓
Schema AST
    ↓
Schema Compiler
    ↓
DDL
```

---

# 220. Correctness formula

```text
CorrectSchemaDiff
=
Typed
∧
Directional
∧
Deterministic
∧
CoverageAware
∧
UncertaintyPreserving
∧
RenameConservative
∧
PlatformAware
∧
CapabilityDriven
∧
NonExecuting
∧
Explainable
∧
ExtensionSafe
∧
RuntimeIsolated
```

---

# 221. Arquitectura final

```text
                Current Schema Snapshot
                         │
                         │
                         ▼
                  Coverage Analysis
                         │
                         ▼
                  Canonicalization
                         │
                         ▼
                    Object Index
                         │
                         ▼
                    Exact Match
                         │
                         ▼
                 Semantic Matching
                         │
                         ▼
                 Rename Analysis
                         │
                         ▼
               Property Comparison
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
    Columns           Indexes         Constraints
        │                │                │
        └────────────────┼────────────────┘
                         │
                         ▼
                 Metadata Comparison
                         │
                         ▼
                 Cross-Object Analysis
                         │
                         ▼
                 Impact Classification
                         │
                         ▼
                 Dependency Analysis
                         │
                         ▼
                  Safety Analysis
                         │
                         ▼
                     SchemaDiff
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Preview    Drift      Migration
                      Report      Planner
```

---

# 222. Regla arquitectónica final

> **Schema Diff debe describir fielmente la diferencia entre dos estados sin confundir ausencia con desconocimiento, similitud con identidad, diferencia con intención ni intención con ejecución.**

La regla maestra será:

```text
Compare what is known.
Preserve what is unknown.
Infer cautiously.
Never invent identity.
Never invent absence.
Never execute the difference.
```

---

# 223. Resultado arquitectónico

Con este sistema VoltStack obtiene la cadena:

```text
Physical Database
       ↓
Introspection
       ↓
Current Schema Snapshot
       │
       │
       ├──────────── Target Schema
       │                 │
       └────────┬────────┘
                ▼
            Schema Diff
                │
                ▼
       Typed Differences
                │
                ├── Added
                ├── Removed
                ├── Modified
                ├── Rename Candidates
                ├── Unknown
                └── Incomparable
```

sin permitir que una comparación se transforme prematuramente en una operación destructiva.

Esto proporciona la base necesaria para construir posteriormente:

```text
Schema Planning
Migration Planning
Zero-Downtime Migrations
Drift Detection
Schema Verification
Developer Tooling
CI Schema Checks
```

---

# 224. Siguiente documento

```text
99_DATABASE_SCHEMA_COMPILER_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack transforma:

```text
Schema AST
+
Schema Execution/Planning Context
+
Database Platform
+
Dialect
+
Capabilities
        ↓
Compiled Schema Commands
```

manteniendo:

```text
Schema Compiler
≠
Schema Diff

Schema Compiler
≠
Migration

Schema Compiler
≠
Execution

Schema AST
≠
DDL String

Compiled Schema Command
≠
Executed Statement
```

y deberá resolver especialmente:

```text
CREATE TABLE
ALTER TABLE
DROP TABLE
ADD/DROP/ALTER COLUMN
INDEX DDL
CONSTRAINT DDL
FOREIGN KEY DDL
SEQUENCE DDL
VIEW DDL
platform-specific DDL
multi-command transformations
capability-driven emulation
identifier rendering
type rendering
default/expression rendering
deterministic compilation
prepared-vs-non-prepared DDL boundaries
transaction requirements
```

antes de abordar:

```text
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md
```