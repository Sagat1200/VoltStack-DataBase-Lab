# 100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Platform Compatibility System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 100 — Database Schema Platform Compatibility System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Platform Compatibility System` define la arquitectura mediante la cual VoltStack determina si una definición, característica, operación o plan estructural puede representarse correctamente sobre una plataforma de base de datos determinada.

El sistema deberá responder preguntas como:

```text
¿Puede esta plataforma representar este tipo?

¿Puede crear este índice?

¿Puede preservar esta constraint?

¿Puede realizar esta operación directamente?

¿Requiere una estrategia alternativa?

¿Existe pérdida semántica?

¿La compatibilidad es conocida o incierta?
```

La función general será:

```text
Schema Artifact
+
Database Platform
+
Capability Snapshot
+
Compatibility Profile
+
Explicit Platform Metadata
        ↓
Schema Compatibility Analysis
        ↓
Compatibility Report
```

---

# 2. Principio central

> **Compatibilidad significa preservar la semántica estructural requerida, no simplemente encontrar alguna sintaxis SQL que la plataforma acepte.**

Por tanto:

```text
SQL Accepted
≠
Schema Compatible
```

y:

```text
Compilable
≠
Semantically Compatible
```

---

# 3. Distinciones fundamentales

VoltStack deberá mantener:

```text
Platform Compatibility
≠
Platform Capability

Platform Compatibility
≠
SQL Compilation

Platform Compatibility
≠
Schema Validation

Platform Compatibility
≠
Migration Safety

Platform Compatibility
≠
Operational Feasibility

Platform Compatibility
≠
Database Version Check
```

Cada concepto responde una pregunta distinta.

---

# 4. Preguntas por subsistema

```text
Schema Validation
    ↓
¿La estructura solicitada es internamente válida?

Capability System
    ↓
¿Qué puede hacer esta plataforma?

Compatibility System
    ↓
¿Puede esta estructura preservar su significado
sobre esta plataforma?

Planner
    ↓
¿Qué estrategia debe utilizarse?

Compiler
    ↓
¿Cómo se representa la estrategia en SQL?

Execution Engine
    ↓
¿Cómo se ejecuta?
```

---

# 5. Posición arquitectónica

```text
                 Schema Definition
                        │
                        ▼
                Schema Validation
                        │
                        ▼
           Platform Compatibility System
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
      Platform      Capability     Compatibility
       Model         Snapshot        Profile
          │             │             │
          └─────────────┼─────────────┘
                        │
                        ▼
              Compatibility Report
                        │
                        ▼
                   Schema Diff
                        │
                        ▼
                 Schema Planner
                        │
                        ▼
                Schema Compiler
```

El Compatibility System también podrá ser consultado directamente por:

```text
Schema Builder diagnostics
Schema Diff
Migration Planner
Schema Compiler validation
Developer tooling
CLI
Testing
Portability analysis
```

---

# 6. Regla maestra

```text
Compatibility
=
Semantic Preservation
+
Capability Satisfaction
+
Representation Feasibility
+
Platform Constraints
+
Known Metadata
```

No:

```text
Compatibility
=
vendor == mysql
```

---

# 7. Artefactos analizables

El sistema deberá poder analizar:

```text
DatabaseSchema
TableDefinition
ColumnDefinition
DatabaseType
IndexDefinition
ConstraintDefinition
ForeignKeyDefinition
SequenceDefinition
ViewDefinition
SchemaExpression
Schema AST Node
Schema Diff Change
PlannedSchemaOperation
SchemaExecutionPlan
```

---

# 8. CompatibilitySubject

Se propone un contrato conceptual:

```php
interface SchemaCompatibilitySubject
{
    public function compatibilityKind(): SchemaCompatibilitySubjectKind;
}
```

No necesariamente todos los modelos deberán implementarlo directamente.

Podrán utilizarse adapters especializados para evitar contaminar los modelos estructurales.

---

# 9. Resultado principal

No deberá utilizarse simplemente:

```php
bool $compatible;
```

porque:

```text
true / false
```

no representa adecuadamente:

- compatibilidad parcial;
- limitaciones;
- emulación;
- incertidumbre;
- pérdida semántica;
- diferencias físicas;
- requisitos adicionales.

---

# 10. CompatibilityStatus

Se propone:

```php
enum SchemaCompatibilityStatus
{
    case SUPPORTED;

    case SUPPORTED_WITH_LIMITATIONS;

    case REQUIRES_EMULATION;

    case UNSUPPORTED;

    case UNKNOWN;
}
```

---

# 11. SUPPORTED

Significa:

```text
la plataforma puede representar
la semántica requerida
sin degradación relevante
```

No significa necesariamente que use la misma representación física.

---

# 12. SUPPORTED_WITH_LIMITATIONS

Significa:

```text
la semántica principal puede preservarse,
pero existen restricciones explícitas
```

Ejemplo conceptual:

```text
VARCHAR length supported,
but maximum requested size depends on
platform encoding/index limitations.
```

---

# 13. REQUIRES_EMULATION

Significa:

```text
la plataforma no posee representación
directa suficiente,
pero existe una estrategia estructural
conocida que puede preservar la semántica
bajo condiciones explícitas.
```

---

# 14. UNSUPPORTED

Significa:

```text
la semántica requerida no puede preservarse
con las capacidades y políticas disponibles.
```

---

# 15. UNKNOWN

Significa:

```text
VoltStack no posee suficiente información
para demostrar compatibilidad
o incompatibilidad.
```

Regla crítica:

```text
UNKNOWN
≠
UNSUPPORTED
```

---

# 16. Estado vs severidad

`CompatibilityStatus` no deberá confundirse con severidad diagnóstica.

Ejemplo:

```text
SUPPORTED_WITH_LIMITATIONS
```

podría producir:

```text
INFO
WARNING
ERROR
```

dependiendo del profile.

---

# 17. CompatibilityLevel

Opcionalmente:

```php
enum SchemaCompatibilitySeverity
{
    case INFO;
    case WARNING;
    case ERROR;
    case FATAL;
}
```

---

# 18. CompatibilityReport

Se propone:

```php
final readonly class SchemaCompatibilityReport
{
    public function __construct(
        public SchemaCompatibilityStatus $status,
        public SchemaCompatibilityIssueSet $issues,
        public SchemaCapabilityRequirementSet $requirements,
        public SchemaCompatibilityLimitationSet $limitations,
        public SchemaEmulationCandidateSet $emulations,
        public SchemaCompatibilityEvidenceSet $evidence,
        public SchemaCompatibilityMetadata $metadata,
    ) {}
}
```

---

# 19. Report estructurado

El resultado deberá responder:

```text
What is compatible?
What is incompatible?
Why?
Which capability is missing?
What semantics are affected?
Can it be emulated?
Under which conditions?
How certain is the conclusion?
```

---

# 20. CompatibilityIssue

```php
final readonly class SchemaCompatibilityIssue
{
    public function __construct(
        public SchemaCompatibilityIssueCode $code,
        public SchemaCompatibilitySeverity $severity,
        public SchemaObjectPath $subject,
        public string $message,
        public SchemaSemanticImpact $impact,
        public SchemaCompatibilityEvidenceSet $evidence,
    ) {}
}
```

---

# 21. Evidence-driven compatibility

Una decisión deberá basarse en evidencia estructurada.

Ejemplo:

```text
Requirement:
PARTIAL_INDEX

Capability Snapshot:
PARTIAL_INDEX = unsupported

Conclusion:
UNSUPPORTED
```

o:

```text
Requirement:
DIRECT_ALTER_COLUMN

Capability Snapshot:
DIRECT_ALTER_COLUMN = unsupported

Known strategy:
TABLE_REBUILD

Conclusion:
REQUIRES_EMULATION
```

---

# 22. Compatibility evidence

Se propone:

```text
SchemaCompatibilityEvidence
├── CapabilityEvidence
├── TypeMappingEvidence
├── PlatformRuleEvidence
├── ExtensionEvidence
├── MetadataEvidence
├── EmulationEvidence
└── UnknownEvidence
```

---

# 23. Compatibility certainty

Puede modelarse:

```php
enum SchemaCompatibilityCertainty
{
    case CERTAIN;
    case HIGH;
    case CONDITIONAL;
    case LOW;
    case UNKNOWN;
}
```

---

# 24. Capability ≠ compatibility

Una capability responde:

```text
Does PostgreSQL support feature X?
```

Compatibilidad responde:

```text
Can definition D preserve its semantics
given PostgreSQL capabilities C?
```

Formalmente:

```text
Capability = property of platform context
Compatibility = relation between artifact and platform context
```

---

# 25. Fórmula básica

Para artefacto `S` y plataforma `P`:

```text
Compatible(S, P)
=
SemanticRequirements(S)
⊆
RepresentableSemantics(P)
```

con condiciones adicionales.

---

# 26. Fórmula extendida

```text
Compatibility(S, P, C, Profile)
=
Evaluate(
    SemanticRequirements(S),
    StructuralRequirements(S),
    OperationalRequirements(S),
    C,
    PlatformRules(P),
    Profile
)
```

---

# 27. Version ≠ capability

Nunca:

```php
if ($mysqlVersion >= 8) {
    $compatible = true;
}
```

como arquitectura principal.

Preferir:

```php
$capabilities->supports(
    SchemaCapability::CHECK_CONSTRAINT_ENFORCEMENT
);
```

---

# 28. DatabasePlatform

`DatabasePlatform` proporciona semántica de plataforma.

Conceptualmente:

```php
interface DatabasePlatform
{
    public function id(): PlatformId;

    public function capabilities(): PlatformCapabilitySnapshot;

    public function schemaCompatibilityRules(): SchemaCompatibilityRuleSet;
}
```

---

# 29. Capability Snapshot

Debe ser:

```text
immutable
explicit
versioned/fingerprintable
operation-safe
```

y no consultar la base durante análisis.

---

# 30. No hidden I/O

Compatibility System jamás deberá ejecutar:

```text
SELECT VERSION()
SHOW VARIABLES
PRAGMA ...
information_schema queries
pg_catalog queries
```

durante el análisis.

---

# 31. Capability discovery

La obtención de capabilities pertenece a:

```text
Driver / Platform initialization
Capability Discovery
Connection bootstrap
```

no al Compatibility Analyzer.

---

# 32. Compatibility Profile

Se propone:

```php
enum SchemaCompatibilityProfile
{
    case NATIVE;
    case PORTABLE;
    case STRICT_PORTABLE;
    case MIGRATION;
    case INTROSPECTION_ROUND_TRIP;
}
```

---

# 33. NATIVE profile

Pregunta:

```text
¿Puede la plataforma objetivo representar
correctamente esta estructura?
```

Permite características específicas de esa plataforma.

---

# 34. PORTABLE profile

Pregunta:

```text
¿Puede esta estructura mantenerse razonablemente
portable dentro del conjunto objetivo?
```

---

# 35. STRICT_PORTABLE

Exige:

```text
semantic support across all declared target platforms
```

sin depender de extensiones vendor-specific.

---

# 36. MIGRATION profile

Considera además:

```text
operation representation
required strategy
transaction behavior
known emulation
```

pero no decide seguridad operacional.

---

# 37. INTROSPECTION_ROUND_TRIP

Evalúa:

```text
Definition
    ↓
Compile
    ↓
Database
    ↓
Introspect
```

y si puede recuperarse una estructura semánticamente equivalente.

---

# 38. Portability ≠ common denominator blindly

No debe diseñarse VoltStack como:

```text
features =
intersection(MySQL, MariaDB, PostgreSQL, SQLite)
```

Eso destruiría capacidades avanzadas.

---

# 39. Portable core + capabilities

La estrategia será:

```text
Portable Core
+
Capability-Gated Features
+
Typed Platform Extensions
```

---

# 40. Compatibility matrix

Conceptualmente:

| Feature | MySQL | MariaDB | PostgreSQL | SQLite |
|---|---|---|---|---|
| Basic tables | Capability-driven | Capability-driven | Capability-driven | Capability-driven |
| Foreign keys | Capability-driven | Capability-driven | Capability-driven | Capability-driven |
| Partial indexes | Capability-driven | Capability-driven | Capability-driven | Capability-driven |
| Expression indexes | Capability-driven | Capability-driven | Capability-driven | Capability-driven |
| Deferrable constraints | Capability-driven | Capability-driven | Capability-driven | Capability-driven |
| Sequences | Capability-driven | Capability-driven | Capability-driven | Capability-driven |
| Generated columns | Capability-driven | Capability-driven | Capability-driven | Capability-driven |

La tabla real no deberá codificarse como documentación estática dentro del motor.

El motor usará capabilities.

---

# 41. MySQL y MariaDB

Debe mantenerse:

```text
MySQL Compatibility Rules
≠
MariaDB Compatibility Rules
```

aunque compartan gran cantidad de infraestructura.

---

# 42. SchemaCompatibilityAnalyzer

Contrato principal:

```php
interface SchemaCompatibilityAnalyzer
{
    public function analyze(
        SchemaCompatibilitySubject $subject,
        SchemaCompatibilityContext $context,
    ): SchemaCompatibilityReport;
}
```

---

# 43. CompatibilityContext

```php
final readonly class SchemaCompatibilityContext
{
    public function __construct(
        public DatabasePlatform $platform,
        public PlatformCapabilitySnapshot $capabilities,
        public SchemaCompatibilityProfile $profile,
        public FrozenSchemaCompatibilityRuleRegistry $rules,
        public SchemaCompatibilityBudget $budget,
    ) {}
}
```

---

# 44. Analyzer especializados

```text
SchemaCompatibilityAnalyzer
├── DatabaseSchemaCompatibilityAnalyzer
├── TableCompatibilityAnalyzer
├── ColumnCompatibilityAnalyzer
├── TypeCompatibilityAnalyzer
├── IndexCompatibilityAnalyzer
├── ConstraintCompatibilityAnalyzer
├── ForeignKeyCompatibilityAnalyzer
├── SequenceCompatibilityAnalyzer
├── ViewCompatibilityAnalyzer
├── ExpressionCompatibilityAnalyzer
├── AstCompatibilityAnalyzer
├── DiffCompatibilityAnalyzer
└── PlanCompatibilityAnalyzer
```

---

# 45. Composite analysis

Un `TableDefinition` podrá producir:

```text
Table Report
├── Column Reports
├── Index Reports
├── Constraint Reports
├── FK Reports
├── Option Reports
└── Extension Reports
```

---

# 46. Status aggregation

Debe existir una regla determinista.

Ejemplo:

```text
if any child = UNSUPPORTED
    → UNSUPPORTED

else if any child = UNKNOWN
    → UNKNOWN or policy-defined conservative result

else if any child = REQUIRES_EMULATION
    → REQUIRES_EMULATION

else if any child = SUPPORTED_WITH_LIMITATIONS
    → SUPPORTED_WITH_LIMITATIONS

else
    → SUPPORTED
```

Pero `UNKNOWN` deberá manejarse según profile, no mediante simple orden numérico universal.

---

# 47. No lossy aggregation

El reporte padre conservará issues de los hijos.

No deberá reducir:

```text
50 compatibility findings
```

a un simple enum.

---

# 48. Column compatibility

Debe analizar:

```text
type
length
precision
scale
nullability
default
generation
collation
charset
generated expression
platform options
```

---

# 49. Type compatibility

Debe distinguir:

```text
ExactPhysicalSupport
SemanticEquivalentSupport
LossySupport
Unsupported
Unknown
```

---

# 50. Physical type ≠ logical type

Ejemplo conceptual:

```text
BooleanType
```

puede tener representaciones físicas distintas.

Eso no implica incompatibilidad si:

```text
boolean semantics
```

son preservadas según el contrato de VoltStack.

---

# 51. Lossy type mapping

Ejemplo:

```text
Decimal(65, 30)
```

sobre una plataforma cuya representación disponible no conserva precisión suficiente.

Resultado:

```text
UNSUPPORTED
```

o:

```text
SUPPORTED_WITH_LIMITATIONS
```

solo si el profile permite explícitamente esa pérdida.

---

# 52. Silent precision loss

Prohibido:

```text
Decimal(65,30)
    ↓
Decimal(18,2)
```

sin diagnóstico.

---

# 53. String length compatibility

Debe considerar:

```text
logical length
physical limits
encoding
index participation
platform-specific constraints
```

cuando sean relevantes.

---

# 54. Character semantics

Compatibilidad puede depender de:

```text
charset
collation
case sensitivity
normalization behavior
```

No únicamente de `VARCHAR`.

---

# 55. Date/time compatibility

Debe analizar:

```text
precision
timezone semantics
range
default expressions
generation
```

---

# 56. JSON compatibility

Debe distinguir:

```text
native JSON semantics
text-backed JSON
JSON validation
JSON query capability
```

No todo almacenamiento de texto representa un `JsonType` equivalente.

---

# 57. UUID compatibility

Debe permitir:

```text
native UUID
binary representation
canonical textual representation
```

solo cuando el mapping contract preserve las propiedades requeridas.

---

# 58. Enum compatibility

Debe distinguir:

```text
native enum
check-based emulation
string storage
```

La elección de emulación corresponde al planner/type strategy, no al Compatibility Analyzer.

El analyzer solo reporta posibilidades.

---

# 59. Generated columns

Analizar:

```text
generated column support
stored/virtual semantics
expression compatibility
determinism requirements
type restrictions
```

---

# 60. Identity compatibility

Debe separar:

```text
LogicalIdentityGeneration
```

de:

```text
AUTO_INCREMENT
IDENTITY
SEQUENCE
ROWID
```

---

# 61. Identity example

Una columna:

```text
GeneratedIdentifier
```

podrá ser compatible con varias estrategias físicas.

El Compatibility System reportará las estrategias posibles, sin seleccionarlas.

---

# 62. Default compatibility

Analizar:

```text
literal default
expression default
generated default
null default
platform restrictions
```

---

# 63. Expression compatibility

Debe analizar el árbol de:

```text
SchemaExpression
```

recursivamente.

---

# 64. Expression capabilities

Ejemplos:

```text
CURRENT_TIMESTAMP
function call
cast
binary operation
column reference
JSON expression
platform extension
```

pueden requerir capabilities distintas.

---

# 65. Raw expressions

Para:

```text
RawSchemaExpression
```

compatibilidad normalmente será:

```text
platform-scoped
```

o:

```text
UNKNOWN
```

si no puede analizarse.

---

# 66. Raw ≠ portable

Regla:

```text
Raw SQL
→ portability cannot be assumed
```

---

# 67. Table compatibility

Debe analizar:

```text
columns
constraints
indexes
persistence
temporary semantics
table options
namespace
extensions
```

---

# 68. Table options

Debe distinguir:

```text
PortableTableOption
PlatformTableOption
OpaqueTableOption
```

---

# 69. Platform-specific option

Ejemplo conceptual:

```text
MySQL engine option
```

puede ser:

```text
SUPPORTED
```

en MySQL,

pero:

```text
UNSUPPORTED
```

en PostgreSQL.

No significa que toda la tabla sea conceptualmente inválida.

---

# 70. Index compatibility

Debe analizar:

```text
key count
column keys
expression keys
ordering
null ordering
partial predicate
included columns
method
uniqueness
visibility
platform options
```

---

# 71. Index semantics

Compatibilidad de un índice puede ser:

```text
physical
```

más que lógica.

Aun así, si el developer declaró explícitamente una propiedad física requerida, no deberá descartarse silenciosamente.

---

# 72. Index method

Ejemplo:

```text
GIN
GiST
HASH
BTREE
```

será capability/platform-specific.

---

# 73. Partial indexes

```text
PartialIndex
```

requiere:

```text
PARTIAL_INDEX
```

o estrategia explícita de emulación conocida.

---

# 74. Expression indexes

Requieren:

```text
EXPRESSION_INDEX
```

más compatibilidad de la expresión.

---

# 75. Included columns

Requieren capability específica.

No deberán convertirse automáticamente en key columns.

---

# 76. Unique index vs constraint

Compatibility System mantendrá:

```text
UniqueIndex
≠
UniqueConstraint
```

incluso si una plataforma utiliza estructuras físicas similares.

---

# 77. Constraint compatibility

Analizará:

```text
kind
scope
columns
expression
enforcement
validation state
deferrability
platform options
```

---

# 78. Primary key

Debe comprobar:

```text
column count
supported types
nullable restrictions
generated-column restrictions
platform limits
```

sin convertir automáticamente PK en index model.

---

# 79. Unique constraint

Debe comprobar si la plataforma puede preservar:

```text
uniqueness semantics
NULL semantics
deferrability
validation/enforcement state
```

cuando dichas propiedades estén especificadas.

---

# 80. NULL uniqueness semantics

Este punto es especialmente importante.

Diferentes motores pueden manejar:

```text
NULL + UNIQUE
```

de formas relevantes para la semántica solicitada.

VoltStack no deberá asumir equivalencia universal.

---

# 81. Check constraint

Debe analizar:

```text
CHECK support
expression support
enforcement
validation state
```

---

# 82. CHECK ≠ application validation

Que una plataforma no pueda representar un CHECK no significa que:

```text
PHP validator
```

sea automáticamente una emulación equivalente.

---

# 83. Foreign key compatibility

Debe analizar:

```text
local columns
referenced columns
type compatibility
candidate key eligibility
referential actions
match semantics
deferrability
validation state
enforcement
cross-namespace references
```

---

# 84. Foreign key type compatibility

No requerirá necesariamente:

```text
typeA === typeB
```

sino:

```text
ForeignKeyCompatible(typeA, typeB, platform)
```

---

# 85. Composite FK

Debe preservar:

```text
arity
mapping order
candidate-key semantics
```

---

# 86. Referential actions

Analizar independientemente:

```text
ON DELETE
ON UPDATE
```

---

# 87. NO ACTION ≠ RESTRICT

Compatibility Analyzer no deberá asumir equivalencia universal.

---

# 88. SET NULL

Debe considerar:

```text
action support
+
local column nullability
```

---

# 89. SET DEFAULT

Debe considerar:

```text
action support
+
default availability
+
default compatibility
```

---

# 90. Deferrability

Debe analizar:

```text
NOT DEFERRABLE
DEFERRABLE INITIALLY IMMEDIATE
DEFERRABLE INITIALLY DEFERRED
```

cuando sea semánticamente requerida.

---

# 91. Constraint validation state

Debe distinguir:

```text
VALIDATED
NOT_VALIDATED
UNKNOWN
PLATFORM_DEFAULT
```

No convertir `UNKNOWN` en `VALIDATED`.

---

# 92. Enforcement state

Debe distinguir:

```text
ENFORCED
NOT_ENFORCED
UNKNOWN
PLATFORM_DEFAULT
```

---

# 93. Constraint proof safety

Una constraint compatible físicamente no implica automáticamente que pueda usarse como prueba semántica por Query Optimizer.

Se requiere además:

```text
enforced
+
validated/trusted
+
complete metadata
```

---

# 94. Sequence compatibility

Debe analizar:

```text
sequence existence
increment
start
min/max
cycle
cache
ownership
data type
```

---

# 95. Sequence emulation

Una plataforma sin sequences podrá tener una estrategia alternativa.

El Compatibility System podrá reportar:

```text
REQUIRES_EMULATION
```

pero no creará dicha estrategia.

---

# 96. View compatibility

Debe analizar:

```text
query representation
replace semantics
materialization if applicable
security options
check options
platform extensions
```

---

# 97. Query compatibility inside views

Si `ViewDefinition` contiene Query AST, podrá delegarse el análisis a una interfaz de compatibilidad del Query subsystem.

Debe evitarse dependencia circular.

---

# 98. Schema namespace compatibility

Debe modelar diferencias entre:

```text
database
catalog
schema
namespace
```

sin fingir que todas las plataformas tienen la misma jerarquía.

---

# 99. Qualified object compatibility

Una referencia:

```text
catalog.schema.table
```

puede no ser representable exactamente en todas las plataformas.

Debe producir diagnóstico.

---

# 100. Schema AST compatibility

El analyzer podrá evaluar operaciones antes de planning.

Ejemplo:

```text
RenameColumnNode
```

puede ser:

```text
SUPPORTED
```

o:

```text
REQUIRES_EMULATION
```

según capabilities.

---

# 101. Definition compatibility vs operation compatibility

Debe mantenerse:

```text
Can platform represent final state?
```

distinto de:

```text
Can platform transition to final state directly?
```

---

# 102. Ejemplo crítico

Una plataforma puede soportar perfectamente:

```text
Column VARCHAR(500)
```

como estado final.

Pero no soportar:

```text
ALTER COLUMN TYPE
```

directamente.

Entonces:

```text
Definition Compatibility = SUPPORTED
Operation Compatibility = REQUIRES_EMULATION
```

---

# 103. Esto evita un error arquitectónico

No deberá concluirse:

```text
column type unsupported
```

cuando el problema real es:

```text
alter operation unsupported
```

---

# 104. Schema Diff integration

Schema Diff podrá utilizar compatibility para clasificar:

```text
DirectlyApplicableChange
EmulatableChange
UnsupportedChange
UnknownChange
```

pero Diff no deberá decidir la estrategia física.

---

# 105. Planner integration

Planner recibe:

```text
Schema Change
+
Compatibility Report
+
Capabilities
```

y elige estrategia.

---

# 106. EmulationCandidate

Se propone:

```php
final readonly class SchemaEmulationCandidate
{
    public function __construct(
        public SchemaEmulationKind $kind,
        public SchemaCapabilityRequirementSet $requirements,
        public SchemaSemanticGuarantee $guarantee,
        public SchemaEmulationConstraintSet $constraints,
        public SchemaEmulationMetadata $metadata,
    ) {}
}
```

---

# 107. Candidate ≠ selected strategy

```text
Compatibility Analyzer
    ↓
possible emulation candidates
```

```text
Planner
    ↓
selected strategy
```

---

# 108. Emulation examples

```text
ALTER COLUMN
→ table rebuild candidate

ENUM
→ check/string representation candidate

IDENTITY
→ sequence candidate

platform-specific boolean
→ physical type mapping candidate
```

siempre que preserve el contrato requerido.

---

# 109. Emulation ≠ degradation

Una emulación válida deberá buscar:

```text
SemanticEquivalent(
    NativeBehavior,
    EmulatedBehavior
)
```

dentro del contrato definido.

---

# 110. Lossy emulation

Si no preserva semántica:

```text
LossyTransformation
```

no deberá clasificarse simplemente como `REQUIRES_EMULATION`.

Debe producir:

```text
SUPPORTED_WITH_LIMITATIONS
```

si la policy permite la pérdida,

o:

```text
UNSUPPORTED
```

en modo estricto.

---

# 111. Semantic impact

Se propone:

```php
enum SchemaSemanticImpact
{
    case NONE;
    case PHYSICAL_ONLY;
    case PERFORMANCE;
    case REPRESENTATION;
    case BEHAVIORAL;
    case INTEGRITY;
    case DATA_LOSS_RISK;
    case UNKNOWN;
}
```

---

# 112. Physical difference

Ejemplo:

```text
logical boolean
```

representado físicamente de forma distinta puede ser:

```text
PHYSICAL_ONLY
```

si la semántica se conserva.

---

# 113. Behavioral difference

Ejemplo:

```text
case-insensitive collation
```

convertida en:

```text
case-sensitive collation
```

es:

```text
BEHAVIORAL
```

y no debe ocultarse.

---

# 114. Integrity difference

Ejemplo:

```text
CHECK constraint
```

omitida porque el motor no la soporta sería:

```text
INTEGRITY
```

y generalmente `UNSUPPORTED`.

---

# 115. Operational compatibility

Debe diferenciarse de compatibilidad estructural.

Ejemplo:

```text
CREATE INDEX
```

puede ser estructuralmente compatible.

Pero:

```text
CREATE INDEX CONCURRENTLY
```

puede no estar disponible.

---

# 116. Operational requirement

Puede modelarse:

```text
SchemaOperationalCompatibilityReport
```

o como dimensión separada del reporte.

---

# 117. Compatibility dimensions

Se recomienda:

```text
SchemaCompatibilityDimensions
├── Structural
├── Semantic
├── Representational
├── Operational
├── Portability
└── RoundTrip
```

---

# 118. Structural compatibility

Pregunta:

```text
¿Puede existir esta estructura?
```

---

# 119. Semantic compatibility

Pregunta:

```text
¿Preserva el significado requerido?
```

---

# 120. Representational compatibility

Pregunta:

```text
¿Existe una representación física/DDL adecuada?
```

---

# 121. Operational compatibility

Pregunta:

```text
¿Puede realizarse esta operación
bajo las condiciones solicitadas?
```

---

# 122. Portability compatibility

Pregunta:

```text
¿Puede mantenerse el contrato
entre las plataformas objetivo?
```

---

# 123. Round-trip compatibility

Pregunta:

```text
¿Puede compilarse, observarse e introspectarse
sin perder información estructural relevante?
```

---

# 124. Round-trip equation

Idealmente:

```text
Normalize(
    Introspect(
        Execute(
            Compile(S)
        )
    )
)
≈
Normalize(S)
```

---

# 125. Round-trip limitation

No siempre será posible preservar:

```text
generated names
comments
physical implementation details
opaque vendor metadata
```

exactamente.

El profile deberá definir qué equivalencia exige.

---

# 126. CompatibilityRule

Se propone:

```php
interface SchemaCompatibilityRule
{
    public function supports(
        SchemaCompatibilitySubject $subject,
        SchemaCompatibilityContext $context,
    ): bool;

    public function evaluate(
        SchemaCompatibilitySubject $subject,
        SchemaCompatibilityContext $context,
    ): SchemaCompatibilityFindingSet;
}
```

---

# 127. Rule categories

```text
TypeCompatibilityRule
ColumnCompatibilityRule
IndexCompatibilityRule
ConstraintCompatibilityRule
ForeignKeyCompatibilityRule
ExpressionCompatibilityRule
OperationCompatibilityRule
PlatformExtensionCompatibilityRule
```

---

# 128. Rule registry

```text
SchemaCompatibilityRuleRegistry
      ↓ freeze
FrozenSchemaCompatibilityRuleRegistry
```

---

# 129. No last-wins

Dos reglas conflictivas deberán resolverse mediante:

```text
explicit priority
rule domain
composition contract
```

o producir error de configuración.

---

# 130. Rule purity

Una rule no deberá:

- ejecutar SQL;
- modificar schema;
- abrir conexiones;
- mutar capabilities;
- modificar el subject;
- resolver tenant globalmente.

---

# 131. Rule result

Las reglas producirán:

```text
findings
requirements
limitations
evidence
emulation candidates
```

no side effects.

---

# 132. Rule determinism

```text
Rule(S, Context)
```

deberá ser determinista.

---

# 133. Rule dependencies

Si una regla depende de otra deberá declararlo.

Ejemplo:

```text
ForeignKeyCompatibilityRule
    ↓
ColumnTypeCompatibilityRule
```

---

# 134. Dependency graph

Las reglas podrán organizarse mediante:

```text
CompatibilityRuleDependencyGraph
```

para ejecución determinista.

---

# 135. Cycle detection

Ciclos inválidos en dependencias de reglas deberán producir:

```text
SchemaCompatibilityRuleCycleException
```

---

# 136. Platform rules

Podrán existir:

```text
MySqlSchemaCompatibilityRules
MariaDbSchemaCompatibilityRules
PostgreSqlSchemaCompatibilityRules
SqliteSchemaCompatibilityRules
```

sobre contratos comunes.

---

# 137. No vendor conditionals scattered

Incorrecto:

```php
if ($platform === 'mysql') {
    ...
}

if ($platform === 'postgres') {
    ...
}
```

por todo el core.

---

# 138. Capability-first design

Preferir:

```php
if (!$context->capabilities->supports(
    SchemaCapability::PARTIAL_INDEX
)) {
    ...
}
```

---

# 139. Platform rule when capability is insufficient

Algunas diferencias son semánticas y no binarias.

Entonces:

```text
capability
+
platform semantic rule
```

serán necesarias.

---

# 140. Platform extension compatibility

Un artefacto:

```text
PostgreSqlIndexMethod(GIN)
```

deberá declarar:

```text
platform scope = PostgreSQL
```

---

# 141. Cross-platform extension

Una extensión podrá implementar mappings para varias plataformas.

Pero deberá declararlos explícitamente.

---

# 142. Unknown extensions

Políticas posibles:

```text
FAIL
PRESERVE_AS_UNKNOWN
REPORT_UNSUPPORTED
```

según contexto.

---

# 143. Compatibility and introspection

Metadata introspectada puede tener:

```text
COMPLETE
PARTIAL
UNKNOWN
```

coverage.

---

# 144. Partial metadata

Nunca:

```text
not observed
=
not supported
```

---

# 145. Unknown observed feature

Si introspection encuentra una feature nativa que VoltStack no comprende:

```text
Compatibility = UNKNOWN
```

o typed opaque preservation.

No asumir incompatibilidad.

---

# 146. Provenance

Compatibility findings deberán poder indicar:

```text
DECLARED
INTROSPECTED
INFERRED
EXTENSION
OPAQUE
```

como fuente.

---

# 147. Compatibility cache

El análisis puede cachearse.

Key conceptual:

```text
SubjectFingerprint
+
PlatformId
+
CapabilityFingerprint
+
CompatibilityProfile
+
RuleRegistryFingerprint
+
CompatibilityEngineVersion
```

---

# 148. Cache output

Solo:

```text
immutable compatibility reports
```

---

# 149. No live state in cache

Nunca:

```text
Connection
Transaction
TenantContext
Request
DriverStatement
```

---

# 150. Compatibility fingerprint

Se propone:

```text
SchemaCompatibilityFingerprint
```

para reproducibilidad y diagnostics.

---

# 151. Fingerprint ≠ proof

Un hash igual ayuda a identificar el mismo contexto.

No reemplaza la validación semántica.

---

# 152. Security

El Compatibility System no debe evaluar SQL arbitrario como mecanismo normal.

---

# 153. Raw SQL compatibility

Para raw DDL:

```text
compatibility = declared platform scope
```

más metadata disponible.

Si no existe suficiente metadata:

```text
UNKNOWN
```

---

# 154. No speculative SQL parser requirement

VoltStack no necesitará intentar entender cualquier DDL raw de terceros para declarar compatibilidad.

---

# 155. Security of diagnostics

Los reports no deberán exponer automáticamente:

```text
credentials
connection strings
raw secrets
sensitive defaults
```

---

# 156. Persistent runtime

Compartible:

```text
Frozen compatibility rules
Platform definitions
Capability definitions
Profiles
```

Operation-scoped:

```text
CompatibilitySession
Diagnostics
Temporary graphs
Evaluation stacks
```

---

# 157. No current platform singleton

Prohibido:

```php
SchemaCompatibility::$currentPlatform;
```

---

# 158. No current tenant singleton

El Compatibility System será tenant-agnostic.

---

# 159. Multitenancy

El paquete Multitenancy podrá proporcionar:

```text
tenant-resolved platform context
```

pero Database core no dependerá de él.

---

# 160. Cross-tenant compatibility

No deberá inferirse:

```text
tenant A schema
=
tenant B schema
```

aunque utilicen el mismo platform ID.

Capabilities/configuración pueden variar.

---

# 161. Compatibility budget

Se propone:

```php
final readonly class SchemaCompatibilityBudget
{
    public function __construct(
        public int $maxObjects,
        public int $maxRules,
        public int $maxExpressionDepth,
        public int $maxDependencies,
        public int $maxIssues,
        public int $maxExtensionEvaluations,
    ) {}
}
```

---

# 162. Budget exhaustion

Debe producir:

```text
SchemaCompatibilityBudgetExceededException
```

o un reporte explícitamente incompleto.

Nunca:

```text
SUPPORTED
```

por haber dejado de analizar.

---

# 163. Incomplete analysis

Puede modelarse:

```text
SchemaCompatibilityCompleteness
├── COMPLETE
├── PARTIAL
└── UNKNOWN
```

---

# 164. Partial analysis rule

```text
PARTIAL
```

no puede producir una garantía global de compatibilidad estricta salvo prueba suficiente.

---

# 165. Performance

Para `n` objetos y `r` reglas aplicables:

```text
O(n × applicableRules)
```

será objetivo razonable.

La indexación por kind/capability deberá reducir reglas irrelevantes.

---

# 166. Rule indexing

```text
SubjectKind
      ↓
Applicable Rule Set
```

evitando probar todas las reglas contra todos los objetos.

---

# 167. Recursive analysis

Debe protegerse contra:

```text
deep expression trees
recursive views
extension recursion
dependency cycles
```

mediante budgets y cycle detection.

---

# 168. Diagnostics DX

Ejemplo:

```text
Schema compatibility error

Object:
users.email

Feature:
Generated column expression

Target:
SQLite

Status:
UNSUPPORTED

Reason:
Required schema expression capability is unavailable.

Impact:
Generated value semantics cannot be preserved.

Possible action:
Use an application-managed value or target a platform
with the required capability.
```

---

# 169. No vague errors

Evitar:

```text
Database feature unsupported.
```

Preferir:

```text
what
where
why
required capability
semantic impact
possible strategy
```

---

# 170. Compatibility CLI

En el futuro:

```bash
php volt database:schema:compatibility
```

podrá producir:

```text
Target: PostgreSQL

Tables        24
Supported     22
Limited        1
Emulation      1
Unsupported    0
Unknown        0
```

---

# 171. Multi-platform CLI

Conceptualmente:

```bash
php volt database:schema:compatibility \
    --platform=mysql \
    --platform=postgresql \
    --platform=sqlite
```

---

# 172. Portability report

Podría producir:

```text
users
├── PostgreSQL    SUPPORTED
├── MySQL         SUPPORTED
├── MariaDB       SUPPORTED
└── SQLite        REQUIRES_EMULATION
```

---

# 173. Developer tooling

IDE/debug tooling podrá mostrar:

```text
✓ Portable
⚠ Platform-specific
↻ Requires emulation
✗ Unsupported
? Unknown
```

sin convertir estas etiquetas visuales en contratos internos.

---

# 174. Testing strategy

Debe incluir:

```text
unit tests
property tests
cross-platform tests
capability tests
type compatibility tests
constraint compatibility tests
operation compatibility tests
emulation candidate tests
unknown-state tests
round-trip tests
extension tests
budget tests
persistent-runtime tests
```

---

# 175. Type matrix tests

Cada `DatabaseType` deberá probarse contra cada plataforma soportada.

---

# 176. Constraint matrix tests

Probar:

```text
PrimaryKey
Unique
Check
ForeignKey
```

con distintas capabilities.

---

# 177. Foreign key matrix

Incluir:

```text
single-column
composite
CASCADE
SET NULL
SET DEFAULT
NO ACTION
RESTRICT
deferrability
validation state
```

---

# 178. Index matrix

Incluir:

```text
simple
unique
composite
expression
partial
included columns
custom method
```

---

# 179. Unknown-state tests

Es obligatorio verificar que:

```text
UNKNOWN
```

no se transforme accidentalmente en:

```text
SUPPORTED
```

---

# 180. Determinism test

```text
Analyze(S, C)
=
Analyze(S, C)
```

para mismo input/context/version.

---

# 181. No-I/O test

Todo el sistema deberá poder probarse sin database real.

---

# 182. Round-trip integration testing

Para cada plataforma:

```text
Definition
    ↓
Compatibility
    ↓
Plan
    ↓
Compile
    ↓
Execute
    ↓
Introspect
    ↓
Normalize
    ↓
Compare
```

---

# 183. Compatibility invariant from round-trip

Cuando se declara:

```text
SUPPORTED
```

el round-trip debería preservar la equivalencia definida por el profile, salvo propiedades explícitamente no round-trippable.

---

# 184. Error hierarchy

Se propone:

```text
DatabaseSchemaCompatibilityException
├── InvalidSchemaCompatibilitySubjectException
├── InvalidSchemaCompatibilityContextException
├── UnknownSchemaPlatformException
├── SchemaCapabilityResolutionException
├── SchemaCompatibilityRuleException
├── SchemaCompatibilityRuleConflictException
├── SchemaCompatibilityRuleCycleException
├── SchemaTypeCompatibilityException
├── SchemaColumnCompatibilityException
├── SchemaIndexCompatibilityException
├── SchemaConstraintCompatibilityException
├── SchemaForeignKeyCompatibilityException
├── SchemaExpressionCompatibilityException
├── SchemaOperationCompatibilityException
├── SchemaExtensionCompatibilityException
├── SchemaCompatibilityBudgetExceededException
└── SchemaCompatibilityInvariantException
```

---

# 185. Namespace propuesto

```text
VoltStack\Quantum\Database\Schema\Compatibility
```

---

# 186. Estructura propuesta

```text
Schema/
└── Compatibility/
    ├── Contract/
    │   ├── SchemaCompatibilityAnalyzer.php
    │   ├── SchemaCompatibilityRule.php
    │   └── SchemaCompatibilitySubjectAdapter.php
    │
    ├── Core/
    │   ├── SchemaCompatibilityContext.php
    │   ├── SchemaCompatibilityStatus.php
    │   ├── SchemaCompatibilityProfile.php
    │   ├── SchemaCompatibilityCertainty.php
    │   ├── SchemaCompatibilityCompleteness.php
    │   └── SchemaSemanticImpact.php
    │
    ├── Report/
    │   ├── SchemaCompatibilityReport.php
    │   ├── SchemaCompatibilityIssue.php
    │   ├── SchemaCompatibilityIssueSet.php
    │   ├── SchemaCompatibilityLimitation.php
    │   └── SchemaCompatibilityEvidence.php
    │
    ├── Analyzer/
    │   ├── DatabaseSchemaCompatibilityAnalyzer.php
    │   ├── TableCompatibilityAnalyzer.php
    │   ├── ColumnCompatibilityAnalyzer.php
    │   ├── TypeCompatibilityAnalyzer.php
    │   ├── IndexCompatibilityAnalyzer.php
    │   ├── ConstraintCompatibilityAnalyzer.php
    │   ├── ForeignKeyCompatibilityAnalyzer.php
    │   ├── SequenceCompatibilityAnalyzer.php
    │   ├── ViewCompatibilityAnalyzer.php
    │   ├── ExpressionCompatibilityAnalyzer.php
    │   ├── AstCompatibilityAnalyzer.php
    │   ├── DiffCompatibilityAnalyzer.php
    │   └── PlanCompatibilityAnalyzer.php
    │
    ├── Rule/
    │   ├── Type/
    │   ├── Column/
    │   ├── Index/
    │   ├── Constraint/
    │   ├── ForeignKey/
    │   ├── Expression/
    │   ├── Operation/
    │   └── Extension/
    │
    ├── Platform/
    │   ├── MySQL/
    │   ├── MariaDB/
    │   ├── PostgreSQL/
    │   └── SQLite/
    │
    ├── Capability/
    │   ├── SchemaCapabilityRequirement.php
    │   └── SchemaCapabilityRequirementSet.php
    │
    ├── Emulation/
    │   ├── SchemaEmulationCandidate.php
    │   ├── SchemaEmulationCandidateSet.php
    │   ├── SchemaEmulationKind.php
    │   └── SchemaSemanticGuarantee.php
    │
    ├── Registry/
    │   ├── SchemaCompatibilityRuleRegistry.php
    │   └── FrozenSchemaCompatibilityRuleRegistry.php
    │
    ├── Dependency/
    │   └── CompatibilityRuleDependencyGraph.php
    │
    ├── Fingerprint/
    │   └── SchemaCompatibilityFingerprint.php
    │
    ├── Cache/
    │   └── SchemaCompatibilityCache.php
    │
    ├── Budget/
    │   └── SchemaCompatibilityBudget.php
    │
    ├── Extension/
    │   └── SchemaCompatibilityExtension.php
    │
    └── Exception/
        ├── DatabaseSchemaCompatibilityException.php
        ├── InvalidSchemaCompatibilitySubjectException.php
        ├── InvalidSchemaCompatibilityContextException.php
        ├── UnknownSchemaPlatformException.php
        ├── SchemaCapabilityResolutionException.php
        ├── SchemaCompatibilityRuleException.php
        ├── SchemaCompatibilityRuleConflictException.php
        ├── SchemaCompatibilityRuleCycleException.php
        ├── SchemaTypeCompatibilityException.php
        ├── SchemaColumnCompatibilityException.php
        ├── SchemaIndexCompatibilityException.php
        ├── SchemaConstraintCompatibilityException.php
        ├── SchemaForeignKeyCompatibilityException.php
        ├── SchemaExpressionCompatibilityException.php
        ├── SchemaOperationCompatibilityException.php
        ├── SchemaExtensionCompatibilityException.php
        ├── SchemaCompatibilityBudgetExceededException.php
        └── SchemaCompatibilityInvariantException.php
```

---

# 187. Architectural invariants

## DB-SCHEMA-COMPAT-001
Platform Compatibility será distinta de Platform Capability.

## DB-SCHEMA-COMPAT-002
Platform Compatibility será distinta de Schema Validation.

## DB-SCHEMA-COMPAT-003
Platform Compatibility será distinta de Schema Compilation.

## DB-SCHEMA-COMPAT-004
Platform Compatibility será distinta de Migration Safety.

## DB-SCHEMA-COMPAT-005
Platform Compatibility será distinta de Execution.

## DB-SCHEMA-COMPAT-006
Compatibilidad se definirá por preservación semántica.

## DB-SCHEMA-COMPAT-007
SQL aceptado no implicará compatibilidad.

## DB-SCHEMA-COMPAT-008
Compilabilidad no implicará compatibilidad semántica.

## DB-SCHEMA-COMPAT-009
Compatibility result no será un boolean simple.

## DB-SCHEMA-COMPAT-010
SUPPORTED será distinto de SUPPORTED_WITH_LIMITATIONS.

## DB-SCHEMA-COMPAT-011
SUPPORTED_WITH_LIMITATIONS será distinto de REQUIRES_EMULATION.

## DB-SCHEMA-COMPAT-012
REQUIRES_EMULATION será distinto de UNSUPPORTED.

## DB-SCHEMA-COMPAT-013
UNKNOWN será distinto de UNSUPPORTED.

## DB-SCHEMA-COMPAT-014
UNKNOWN será distinto de SUPPORTED.

## DB-SCHEMA-COMPAT-015
Compatibility status será distinto de severity.

## DB-SCHEMA-COMPAT-016
Reports preservarán issues estructurados.

## DB-SCHEMA-COMPAT-017
Reports preservarán evidence.

## DB-SCHEMA-COMPAT-018
Reports preservarán limitations.

## DB-SCHEMA-COMPAT-019
Reports preservarán emulation candidates.

## DB-SCHEMA-COMPAT-020
Compatibility decisions serán evidence-driven.

## DB-SCHEMA-COMPAT-021
Capability snapshot será immutable.

## DB-SCHEMA-COMPAT-022
Compatibility analysis no realizará hidden DB I/O.

## DB-SCHEMA-COMPAT-023
Compatibility analysis no ejecutará SQL.

## DB-SCHEMA-COMPAT-024
Compatibility analysis no abrirá conexiones.

## DB-SCHEMA-COMPAT-025
Compatibility analysis no consultará server version.

## DB-SCHEMA-COMPAT-026
Version será distinta de capability.

## DB-SCHEMA-COMPAT-027
DatabasePlatform será explícita.

## DB-SCHEMA-COMPAT-028
CompatibilityProfile será explícito.

## DB-SCHEMA-COMPAT-029
NATIVE será distinto de PORTABLE.

## DB-SCHEMA-COMPAT-030
PORTABLE será distinto de STRICT_PORTABLE.

## DB-SCHEMA-COMPAT-031
Portability no será simple feature intersection.

## DB-SCHEMA-COMPAT-032
Portable Core podrá coexistir con platform extensions.

## DB-SCHEMA-COMPAT-033
MySQL tendrá reglas first-class.

## DB-SCHEMA-COMPAT-034
MariaDB tendrá reglas first-class.

## DB-SCHEMA-COMPAT-035
PostgreSQL tendrá reglas first-class.

## DB-SCHEMA-COMPAT-036
SQLite tendrá reglas first-class.

## DB-SCHEMA-COMPAT-037
MariaDB no será alias arquitectónico de MySQL.

## DB-SCHEMA-COMPAT-038
Composite analysis preservará child reports.

## DB-SCHEMA-COMPAT-039
Status aggregation será determinista.

## DB-SCHEMA-COMPAT-040
Aggregation no descartará findings.

## DB-SCHEMA-COMPAT-041
Column compatibility analizará DatabaseType.

## DB-SCHEMA-COMPAT-042
Column compatibility analizará nullability.

## DB-SCHEMA-COMPAT-043
Column compatibility analizará defaults.

## DB-SCHEMA-COMPAT-044
Column compatibility analizará generation.

## DB-SCHEMA-COMPAT-045
Logical type será distinto de physical type.

## DB-SCHEMA-COMPAT-046
Physical difference no implicará incompatibilidad.

## DB-SCHEMA-COMPAT-047
Lossy mapping nunca será silencioso.

## DB-SCHEMA-COMPAT-048
Precision loss deberá diagnosticarse.

## DB-SCHEMA-COMPAT-049
Scale loss deberá diagnosticarse.

## DB-SCHEMA-COMPAT-050
Charset semantics serán consideradas cuando sean relevantes.

## DB-SCHEMA-COMPAT-051
Collation semantics serán consideradas cuando sean relevantes.

## DB-SCHEMA-COMPAT-052
JSON storage no implicará automáticamente JSON semantic compatibility.

## DB-SCHEMA-COMPAT-053
UUID podrá tener múltiples representaciones compatibles.

## DB-SCHEMA-COMPAT-054
Enum emulation no será seleccionada por analyzer.

## DB-SCHEMA-COMPAT-055
Generated columns requerirán expression compatibility.

## DB-SCHEMA-COMPAT-056
Logical identity será distinta de AUTO_INCREMENT.

## DB-SCHEMA-COMPAT-057
Logical identity será distinta de sequence.

## DB-SCHEMA-COMPAT-058
Default literal será distinto de default expression.

## DB-SCHEMA-COMPAT-059
Expression compatibility será recursiva.

## DB-SCHEMA-COMPAT-060
Raw expression no será portable por defecto.

## DB-SCHEMA-COMPAT-061
Table compatibility será agregada desde componentes.

## DB-SCHEMA-COMPAT-062
Platform table options serán explícitas.

## DB-SCHEMA-COMPAT-063
Opaque options no se considerarán compatibles por defecto.

## DB-SCHEMA-COMPAT-064
Index compatibility preservará key ordering.

## DB-SCHEMA-COMPAT-065
Included columns serán distintas de key columns.

## DB-SCHEMA-COMPAT-066
Partial indexes requerirán capability.

## DB-SCHEMA-COMPAT-067
Expression indexes requerirán capability y compatible expression.

## DB-SCHEMA-COMPAT-068
Index method será capability/platform-aware.

## DB-SCHEMA-COMPAT-069
UniqueIndex será distinto de UniqueConstraint.

## DB-SCHEMA-COMPAT-070
PrimaryKey será distinto de Index.

## DB-SCHEMA-COMPAT-071
Constraint compatibility analizará enforcement.

## DB-SCHEMA-COMPAT-072
Constraint compatibility analizará validation state.

## DB-SCHEMA-COMPAT-073
Constraint compatibility analizará deferrability cuando aplique.

## DB-SCHEMA-COMPAT-074
NULL uniqueness semantics no se asumirán universales.

## DB-SCHEMA-COMPAT-075
CHECK no se emulará automáticamente mediante application validation.

## DB-SCHEMA-COMPAT-076
ForeignKey compatibility preservará mapping order.

## DB-SCHEMA-COMPAT-077
ForeignKey compatibility preservará arity.

## DB-SCHEMA-COMPAT-078
FK type compatibility será platform-aware.

## DB-SCHEMA-COMPAT-079
NO_ACTION será distinto de RESTRICT.

## DB-SCHEMA-COMPAT-080
ON DELETE será analizado independientemente de ON UPDATE.

## DB-SCHEMA-COMPAT-081
SET_NULL requerirá nullable-compatible local columns.

## DB-SCHEMA-COMPAT-082
SET_DEFAULT requerirá default-compatible columns.

## DB-SCHEMA-COMPAT-083
Deferrability unsupported no se ignorará.

## DB-SCHEMA-COMPAT-084
UNKNOWN validation state no será VALIDATED.

## DB-SCHEMA-COMPAT-085
UNKNOWN enforcement state no será ENFORCED.

## DB-SCHEMA-COMPAT-086
Constraint compatibility no implicará optimizer trust.

## DB-SCHEMA-COMPAT-087
Sequence compatibility será capability-driven.

## DB-SCHEMA-COMPAT-088
Sequence emulation candidate no será selected strategy.

## DB-SCHEMA-COMPAT-089
View compatibility no creará circular dependency con Query Engine.

## DB-SCHEMA-COMPAT-090
Catalog será distinto de schema/namespace.

## DB-SCHEMA-COMPAT-091
Definition compatibility será distinta de operation compatibility.

## DB-SCHEMA-COMPAT-092
Final-state support no implicará direct-transition support.

## DB-SCHEMA-COMPAT-093
Schema Diff podrá consumir compatibility reports.

## DB-SCHEMA-COMPAT-094
Schema Diff no seleccionará emulation strategy.

## DB-SCHEMA-COMPAT-095
Planner seleccionará estrategia.

## DB-SCHEMA-COMPAT-096
Compatibility Analyzer solo propondrá candidates.

## DB-SCHEMA-COMPAT-097
Emulation será distinta de degradation.

## DB-SCHEMA-COMPAT-098
Lossy transformation será explícita.

## DB-SCHEMA-COMPAT-099
Semantic impact será estructurado.

## DB-SCHEMA-COMPAT-100
Physical-only impact será distinto de behavioral impact.

## DB-SCHEMA-COMPAT-101
Behavioral impact será distinto de integrity impact.

## DB-SCHEMA-COMPAT-102
Structural compatibility será una dimensión explícita.

## DB-SCHEMA-COMPAT-103
Semantic compatibility será una dimensión explícita.

## DB-SCHEMA-COMPAT-104
Representational compatibility será una dimensión explícita.

## DB-SCHEMA-COMPAT-105
Operational compatibility será una dimensión explícita.

## DB-SCHEMA-COMPAT-106
Portability será una dimensión explícita.

## DB-SCHEMA-COMPAT-107
Round-trip compatibility será una dimensión explícita.

## DB-SCHEMA-COMPAT-108
Round-trip equivalence utilizará normalization/comparison.

## DB-SCHEMA-COMPAT-109
Generated physical names podrán excluirse de ciertas equivalencias.

## DB-SCHEMA-COMPAT-110
Compatibility rules serán puras.

## DB-SCHEMA-COMPAT-111
Compatibility rules no ejecutarán SQL.

## DB-SCHEMA-COMPAT-112
Compatibility rules no abrirán conexiones.

## DB-SCHEMA-COMPAT-113
Compatibility rules no mutarán schema.

## DB-SCHEMA-COMPAT-114
Compatibility rules no mutarán capabilities.

## DB-SCHEMA-COMPAT-115
Compatibility rule registry será frozen.

## DB-SCHEMA-COMPAT-116
Rule conflicts no usarán last-wins.

## DB-SCHEMA-COMPAT-117
Rule dependencies serán explícitas.

## DB-SCHEMA-COMPAT-118
Rule dependency cycles serán detectados.

## DB-SCHEMA-COMPAT-119
Vendor conditionals dispersos serán evitados.

## DB-SCHEMA-COMPAT-120
Capability-first design será preferido.

## DB-SCHEMA-COMPAT-121
Platform-specific semantic rules podrán complementar capabilities.

## DB-SCHEMA-COMPAT-122
Platform extensions declararán scope.

## DB-SCHEMA-COMPAT-123
Unknown extensions no se declararán compatibles arbitrariamente.

## DB-SCHEMA-COMPAT-124
Partial introspection metadata no implicará ausencia.

## DB-SCHEMA-COMPAT-125
Unknown native features preservarán incertidumbre.

## DB-SCHEMA-COMPAT-126
Compatibility provenance será preservable.

## DB-SCHEMA-COMPAT-127
Compatibility cache será opcional.

## DB-SCHEMA-COMPAT-128
Cache key incluirá subject fingerprint.

## DB-SCHEMA-COMPAT-129
Cache key incluirá capability fingerprint.

## DB-SCHEMA-COMPAT-130
Cache key incluirá profile.

## DB-SCHEMA-COMPAT-131
Cache no almacenará live connections.

## DB-SCHEMA-COMPAT-132
Compatibility fingerprint será versionado.

## DB-SCHEMA-COMPAT-133
Fingerprint no reemplazará semantic proof.

## DB-SCHEMA-COMPAT-134
Raw SQL compatibility no se inferirá universalmente.

## DB-SCHEMA-COMPAT-135
Raw DDL sin metadata suficiente producirá UNKNOWN.

## DB-SCHEMA-COMPAT-136
Compatibility diagnostics protegerán información sensible.

## DB-SCHEMA-COMPAT-137
Shared compatibility services serán immutable.

## DB-SCHEMA-COMPAT-138
Mutable analysis state será operation-scoped.

## DB-SCHEMA-COMPAT-139
No existirá mutable current platform global.

## DB-SCHEMA-COMPAT-140
No existirá mutable current tenant global.

## DB-SCHEMA-COMPAT-141
Multitenancy será integración externa.

## DB-SCHEMA-COMPAT-142
Capabilities podrán variar entre tenant contexts.

## DB-SCHEMA-COMPAT-143
Budgets serán explícitos.

## DB-SCHEMA-COMPAT-144
Budget exhaustion no producirá false SUPPORTED.

## DB-SCHEMA-COMPAT-145
Incomplete analysis será explícito.

## DB-SCHEMA-COMPAT-146
Rule lookup deberá ser indexable por subject kind.

## DB-SCHEMA-COMPAT-147
Recursive analysis tendrá depth budgets.

## DB-SCHEMA-COMPAT-148
Diagnostics explicarán semantic impact.

## DB-SCHEMA-COMPAT-149
Compatibility System será testeable sin database real.

## DB-SCHEMA-COMPAT-150
Compatibility analysis será determinista para mismo input/context/version.

## DB-SCHEMA-COMPAT-151
Cross-platform conformance tests serán obligatorios.

## DB-SCHEMA-COMPAT-152
Unknown-state tests serán obligatorios.

## DB-SCHEMA-COMPAT-153
Round-trip tests serán parte de integration testing.

## DB-SCHEMA-COMPAT-154
SUPPORTED deberá significar semantic preservation según profile.

## DB-SCHEMA-COMPAT-155
REQUIRES_EMULATION no autorizará automáticamente la emulación.

## DB-SCHEMA-COMPAT-156
SUPPORTED_WITH_LIMITATIONS no autorizará pérdida silenciosa.

## DB-SCHEMA-COMPAT-157
Compiler consumirá decisiones posteriores al compatibility analysis.

## DB-SCHEMA-COMPAT-158
Executor no será responsabilidad del Compatibility System.

## DB-SCHEMA-COMPAT-159
Compatibility System describirá posibilidades y límites, no ejecutará estrategias.

## DB-SCHEMA-COMPAT-160
VoltStack nunca sacrificará semántica estructural silenciosamente en nombre de portabilidad.

---

# 188. Anti-patterns

## 188.1 Compatibilidad por nombre del vendor

Incorrecto:

```php
if ($platform->name() === 'postgresql') {
    return true;
}
```

---

## 188.2 Compatibilidad por versión

Incorrecto:

```php
if ($mysqlVersion >= 8.0) {
    return true;
}
```

---

## 188.3 Boolean compatibility

Insuficiente:

```php
bool supportsSchema($schema);
```

---

## 188.4 Unsupported = unknown

Incorrecto:

```text
No sé si está soportado
        ↓
UNSUPPORTED
```

---

## 188.5 Unknown = supported

Aún más peligroso:

```text
No pude comprobarlo
        ↓
SUPPORTED
```

---

## 188.6 Compilable = compatible

Incorrecto:

```text
Puedo producir SQL
        ↓
Compatible
```

---

## 188.7 App validation como CHECK

Incorrecto:

```text
CHECK unsupported
      ↓
PHP validator
      ↓
equivalent
```

No es equivalente a integridad impuesta por la base.

---

## 188.8 Unique index = unique constraint

Incorrecto como regla universal.

---

## 188.9 Platform extension silently dropped

Incorrecto:

```text
Unknown PostgreSQL option
        ↓
ignore
```

---

## 188.10 Analyzer selecciona migration strategy

Incorrecto:

```text
Analyzer
   ↓
rebuild table immediately
```

Debe ser:

```text
Analyzer
   ↓
REQUIRES_EMULATION
   ↓
Planner
   ↓
RebuildTableStrategy
```

---

## 188.11 Hidden database access

Incorrecto:

```php
$analyzer->checkServerVersion($connection);
```

---

## 188.12 Lowest-common-denominator framework

Incorrecto:

```text
SQLite does not support feature X
        ↓
VoltStack forbids X everywhere
```

---

# 189. Ejemplo: definición soportada, operación no directa

Estado deseado:

```text
users.email VARCHAR(500)
```

Plataforma:

```text
supports VARCHAR(500)
```

Resultado:

```text
Definition:
SUPPORTED
```

Pero operación:

```text
ALTER users.email VARCHAR(255) → VARCHAR(500)
```

si no existe alteración directa:

```text
Operation:
REQUIRES_EMULATION
```

Esta distinción es esencial.

---

# 190. Ejemplo: índice parcial

```text
IndexDefinition
├── key: email
└── predicate: deleted_at IS NULL
```

Requirements:

```text
PARTIAL_INDEX
+
compatible predicate expression
```

Si:

```text
PARTIAL_INDEX = supported
```

y expresión compatible:

```text
SUPPORTED
```

Si no:

```text
REQUIRES_EMULATION
```

solo cuando exista una estrategia semánticamente válida conocida.

En otro caso:

```text
UNSUPPORTED
```

---

# 191. Ejemplo: tipo decimal

```text
DecimalType(
    precision = 30,
    scale = 10
)
```

Platform capability:

```text
max precision = 38
max scale = 38
```

Resultado:

```text
SUPPORTED
```

Si target solo preserva:

```text
precision <= 18
```

resultado estricto:

```text
UNSUPPORTED
```

No:

```text
automatically use DECIMAL(18,10)
```

---

# 192. Ejemplo: CHECK

```text
CHECK (price >= 0)
```

Si la plataforma:

```text
parses CHECK
but does not enforce it
```

entonces:

```text
Structural Representation:
SUPPORTED

Integrity Semantics:
UNSUPPORTED
```

Resultado global estricto:

```text
UNSUPPORTED
```

porque aceptar sintaxis no equivale a preservar integridad.

---

# 193. Ejemplo: identity

Definition:

```text
id
└── GeneratedIdentifier
```

Compatibility analysis puede producir:

```text
PostgreSQL
├── status: SUPPORTED
└── representation candidate: IDENTITY
```

```text
MySQL
├── status: SUPPORTED
└── representation candidate: platform identity strategy
```

```text
SQLite
├── status: SUPPORTED_WITH_LIMITATIONS
└── representation candidate: platform-specific row identity
```

según el contrato concreto y capability snapshot.

El analyzer no compila ninguna de ellas.

---

# 194. Ejemplo: FK SET NULL

Definition:

```text
FOREIGN KEY customer_id
REFERENCES customers.id
ON DELETE SET NULL
```

Column:

```text
customer_id NOT NULL
```

Aunque la plataforma soporte:

```text
ON DELETE SET NULL
```

la combinación estructural es incompatible.

Resultado:

```text
UNSUPPORTED
```

por contradicción semántica.

---

# 195. Ejemplo: deferrable constraint

```text
DEFERRABLE INITIALLY DEFERRED
```

Si capability:

```text
DEFERRABLE_CONSTRAINT = false
```

no deberá compilarse simplemente como constraint normal.

Resultado:

```text
UNSUPPORTED
```

o:

```text
REQUIRES_EMULATION
```

solo si existe una emulación que preserve realmente el contrato.

---

# 196. Ejemplo: portability report

Schema:

```text
users
├── id UUID
├── email VARCHAR(320)
├── profile JSON
└── index(lower(email))
```

Resultado conceptual:

```text
MySQL
├── UUID       SUPPORTED_WITH_LIMITATIONS
├── VARCHAR    SUPPORTED
├── JSON       SUPPORTED
└── Expression Index
    └── capability-dependent

MariaDB
├── UUID       platform-rule dependent
├── VARCHAR    SUPPORTED
├── JSON       platform-rule dependent
└── Expression Index
    └── capability-dependent

PostgreSQL
├── UUID       SUPPORTED
├── VARCHAR    SUPPORTED
├── JSON       SUPPORTED
└── Expression Index
    └── SUPPORTED

SQLite
├── UUID       representation-dependent
├── VARCHAR    SUPPORTED_WITH_LIMITATIONS
├── JSON       capability/contract-dependent
└── Expression Index
    └── capability-dependent
```

El reporte real se generará a partir de capability snapshots, no de esta tabla estática.

---

# 197. Integración con Schema Compiler

El flujo será:

```text
Planned Operation
       │
       ▼
Compatibility Verification
       │
       ├── SUPPORTED
       │      ↓
       │   Compiler
       │
       ├── REQUIRES_EMULATION
       │      ↓
       │   back to Planner / invalid plan
       │
       ├── UNSUPPORTED
       │      ↓
       │   fail
       │
       └── UNKNOWN
              ↓
          profile policy
```

---

# 198. Compiler defensive verification

Aunque Planner haya utilizado Compatibility System, Compiler podrá verificar:

```text
required capabilities
```

como defensa de invariantes.

Pero no deberá repetir toda la lógica de compatibilidad.

---

# 199. Integración con Schema Diff

```text
Current Schema
      │
      ├────────────┐
      │            │
      ▼            ▼
Target Schema   Target Platform
      │            │
      └──────┬─────┘
             ▼
         Schema Diff
             │
             ▼
       Schema Changes
             │
             ▼
Compatibility Analysis
             │
             ▼
       Migration Planner
```

---

# 200. Integración con Migration System

El bloque de migraciones podrá utilizar:

```text
CompatibilityReport
```

para determinar:

```text
direct migration
online strategy candidate
table rebuild
temporary object
multi-phase migration
unsupported migration
```

sin convertir Compatibility System en Migration Engine.

---

# 201. Fórmula de compatibilidad

```text
SchemaCompatibility
=
StructuralRepresentability
∧
SemanticPreservation
∧
CapabilitySatisfaction
∧
PlatformRuleSatisfaction
```

con:

```text
KnownInformation
```

suficiente para sostener la conclusión.

---

# 202. Fórmula de compatibilidad estricta

```text
StrictCompatible(S, P)
=
CompleteAnalysis(S, P)
∧
NoSemanticLoss(S, P)
∧
AllRequiredCapabilitiesAvailable(S, P)
∧
NoUnsupportedExtensions(S, P)
```

---

# 203. Fórmula de emulación

```text
RequiresEmulation(S, P)
=
¬DirectlyRepresentable(S, P)
∧
∃ E :
    SemanticEquivalent(E(S), S)
    ∧
    Executable(E, P)
```

El analyzer puede demostrar existencia de `E`.

El planner selecciona `E`.

---

# 204. Fórmula de degradación

```text
Degradation
=
Meaning(TargetRepresentation)
≠
Meaning(SourceDefinition)
```

Si existe degradación:

```text
SUPPORTED
```

no será un resultado válido bajo profile estricto.

---

# 205. Fórmula de incertidumbre

```text
UnknownCompatibility
=
InsufficientCapabilityKnowledge
∨
IncompleteMetadata
∨
UnknownExtension
∨
OpaqueSemantics
∨
IncompleteAnalysis
```

---

# 206. Fórmula de portabilidad

Para plataformas objetivo:

```text
P = {P1, P2, ..., Pn}
```

compatibilidad portable estricta:

```text
Portable(S, P)
=
∀ Pi ∈ P :
    StrictCompatible(S, Pi)
```

Pero esto no limita la API general de VoltStack a dicha intersección.

---

# 207. Arquitectura final

```text
                    Schema Artifact
                          │
                          ▼
                Compatibility Analyzer
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
     Capability       Platform         Compatibility
      Snapshot         Rules              Profile
          │               │                │
          └───────────────┼────────────────┘
                          │
                          ▼
                   Rule Evaluation
                          │
              ┌───────────┼───────────┐
              │           │           │
              ▼           ▼           ▼
           Types       Features    Extensions
              │           │           │
              └───────────┼───────────┘
                          │
                          ▼
                Compatibility Report
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
      Status           Issues          Evidence
        │                 │                 │
        ├── limitations   ├── impact        │
        ├── certainty     ├── location      │
        └── completeness  └── requirements  │
                          │
                          ▼
                Emulation Candidates
                          │
                          ▼
                     Schema Planner
                          │
                          ▼
                    Schema Compiler
```

---

# 208. Master formula

```text
Database Schema Platform Compatibility System
=
Semantic Requirement Extraction
+
Platform Capability Evaluation
+
Platform Semantic Rules
+
Type Compatibility
+
Column Compatibility
+
Index Compatibility
+
Constraint Compatibility
+
Foreign Key Compatibility
+
Expression Compatibility
+
Operation Compatibility
+
Portability Analysis
+
Round-Trip Analysis
+
Structured Evidence
+
Uncertainty Preservation
+
Emulation Discovery
+
Extension Control
+
Deterministic Rules
+
Caching
+
Budgets
+
Persistent Runtime Isolation
```

---

# 209. Regla arquitectónica final

> **VoltStack nunca debe considerar compatible una estructura únicamente porque puede producir SQL para ella. La compatibilidad existe cuando la plataforma puede preservar el significado estructural requerido dentro del contrato declarado.**

Por tanto:

```text
Syntax
<
Representation
<
Semantics
```

y la decisión deberá priorizar:

```text
Semantic Correctness
>
Convenience
```

---

# 210. Cierre del bloque Schema

Con este documento queda definido el bloque principal:

```text
87_DATABASE_SCHEMA_ARCHITECTURE
        │
88_DATABASE_SCHEMA_MODEL
        │
89_DATABASE_SCHEMA_AST_SYSTEM
        │
90_DATABASE_SCHEMA_BUILDER_SYSTEM
        │
91_DATABASE_TABLE_DEFINITION_SYSTEM
        │
92_DATABASE_COLUMN_DEFINITION_SYSTEM
        │
93_DATABASE_INDEX_SYSTEM
        │
94_DATABASE_FOREIGN_KEY_SYSTEM
        │
95_DATABASE_CONSTRAINT_SYSTEM
        │
96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM
        │
97_DATABASE_SCHEMA_METADATA_SYSTEM
        │
98_DATABASE_SCHEMA_DIFF_SYSTEM
        │
99_DATABASE_SCHEMA_COMPILER_SYSTEM
        │
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM
```

El bloque completo establece:

```text
Schema
=
Structural State
+
Structural Intent
+
Developer DSL
+
Definitions
+
Indexes
+
Constraints
+
Referential Integrity
+
Observation
+
Metadata
+
Difference Detection
+
Platform Compilation
+
Compatibility Analysis
```

---

# 211. Transición al sistema de migraciones

A partir del siguiente documento VoltStack pasa de:

```text
¿Qué estructura existe?

¿Qué estructura quiero?

¿Qué diferencias existen?

¿Puede la plataforma representarlas?
```

a:

```text
¿Cómo evoluciona de forma controlada
el esquema de una aplicación
a través del tiempo?
```

Esto introduce un nuevo nivel arquitectónico:

```text
Schema
      ↓
Structural Changes
      ↓
Migration
      ↓
Versioned Evolution
```

---

# 212. Siguiente documento

```text
101_DATABASE_MIGRATION_ARCHITECTURE.md
```

El siguiente documento deberá establecer la arquitectura general del sistema de migraciones de VoltStack:

```text
Migration Definition
        │
        ▼
Migration Discovery
        │
        ▼
Migration Repository
        │
        ▼
Migration State
        │
        ▼
Migration Planner
        │
        ▼
Schema / Data Operations
        │
        ▼
Compatibility + Safety Analysis
        │
        ▼
Migration Execution Plan
        │
        ▼
Schema Compiler
        │
        ▼
Execution Engine
        │
        ▼
Migration Repository Update
```

manteniendo especialmente:

```text
Migration
≠
Schema AST

Migration
≠
Schema Diff

Migration
≠
DDL

Migration
≠
Transaction

Migration
≠
Execution Plan

Migration File Order
≠
Arbitrary Execution Order

Rollback
≠
Automatic Inverse

Schema Evolution
≠
Only CREATE/ALTER/DROP
```

`101_DATABASE_MIGRATION_ARCHITECTURE.md` abrirá el **Bloque 9 — Migrations**, que comprenderá los documentos `101–111`.