# 91_DATABASE_TABLE_DEFINITION_SYSTEM.md

# VoltStack Quantum Database
## Database Table Definition System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 91 — Database Table Definition System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Table Definition System` define la representación estructural, tipada, validada e inmutable de una tabla dentro de la arquitectura de Schema de VoltStack.

Su responsabilidad principal es responder:

> **¿Cómo se representa una tabla como estructura de base de datos, independientemente de cómo fue construida, de si ya existe físicamente y de cómo será compilada a SQL?**

Ejemplo conceptual:

```text
TableDefinition(users)
├── Identifier
│   └── users
├── Columns
│   ├── id
│   ├── email
│   ├── name
│   ├── active
│   ├── created_at
│   └── updated_at
├── PrimaryKey
│   └── id
├── UniqueConstraints
│   └── email
├── Indexes
├── ForeignKeys
├── Checks
├── Options
└── Metadata
```

`TableDefinition` deberá ser una representación estructural reusable por:

```text
Schema AST
Schema Validation
Schema Introspection
Schema Diff
Schema Planner
Schema Compiler
Migration System
Testing
Diagnostics
```

sin depender de ninguno de ellos.

---

# 2. Principio central

La separación fundamental será:

```text
TableBlueprint
=
Mutable construction API
```

```text
TableDefinition
=
Immutable structural definition
```

```text
Schema Model Table
=
Observed/current database structure
```

```text
CreateTableNode
=
Schema operation
```

```text
CREATE TABLE ...
=
Platform-specific SQL representation
```

Por tanto:

```text
TableBlueprint
≠
TableDefinition
≠
ObservedTable
≠
CreateTableNode
≠
SQL
```

---

# 3. Posición arquitectónica

```text
Developer DSL
     │
     ▼
TableBlueprint
     │
     │ lowering
     ▼
TableDefinition
     │
     ▼
Schema AST
     │
     ▼
Schema Validation
     │
     ▼
Schema Planner
     │
     ▼
Schema Compiler
     │
     ▼
Execution Engine
```

También podrá existir:

```text
Database
   │
   ▼
Schema Introspection
   │
   ▼
ObservedTableDefinition
   │
   ▼
Schema Model
```

Ambos caminos comparten conceptos estructurales, pero no necesariamente el mismo nivel de certeza.

---

# 4. Objetivos

El sistema deberá proporcionar:

- representación tipada de tablas;
- identidad estructural explícita;
- columnas ordenadas;
- primary keys;
- unique constraints;
- indexes;
- foreign keys;
- check constraints;
- table options;
- metadata;
- extension metadata;
- validación de invariantes locales;
- fingerprints estructurales;
- comparabilidad;
- serialización determinista;
- compatibilidad con Schema Diff;
- independencia del driver;
- independencia del SQL;
- seguridad para persistent runtimes.

---

# 5. No objetivos

`TableDefinition` no deberá:

- ejecutar SQL;
- generar SQL;
- abrir conexiones;
- introspectar bases de datos;
- decidir migrations;
- realizar ALTER TABLE;
- decidir estrategias de rebuild;
- gestionar transactions;
- conocer PDO;
- hidratar ORM entities;
- administrar UnitOfWork;
- resolver tenants globalmente;
- decidir capabilities físicas;
- contener runtime connection state.

---

# 6. Modelo conceptual

Se propone:

```text
TableDefinition
├── TableIdentifier
├── TableKind
├── ColumnDefinitionSet
├── PrimaryKeyDefinition?
├── UniqueConstraintDefinitionSet
├── IndexDefinitionSet
├── ForeignKeyDefinitionSet
├── CheckConstraintDefinitionSet
├── TableOptionSet
├── TableMetadata
├── ExtensionMetadataSet
└── StructuralFingerprint
```

---

# 7. Contrato principal

Una posible implementación:

```php
final readonly class TableDefinition
{
    public function __construct(
        public TableIdentifier $identifier,
        public TableKind $kind,
        public ColumnDefinitionSet $columns,
        public ?PrimaryKeyDefinition $primaryKey,
        public UniqueConstraintDefinitionSet $uniqueConstraints,
        public IndexDefinitionSet $indexes,
        public ForeignKeyDefinitionSet $foreignKeys,
        public CheckConstraintDefinitionSet $checks,
        public TableOptionSet $options,
        public TableMetadata $metadata,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

La clase concreta puede variar, pero las responsabilidades deberán mantenerse.

---

# 8. Inmutabilidad

Una vez creada:

```php
$table = new TableDefinition(...);
```

no deberá permitirse:

```php
$table->columns[] = ...;
```

ni:

```php
$table->identifier = ...;
```

Las modificaciones estructurales deberán producir nuevas definiciones.

Formalmente:

```text
TableDefinition(t₀)
      │
      │ transformation
      ▼
TableDefinition(t₁)
```

No:

```text
mutable TableDefinition
```

---

# 9. Razón de la inmutabilidad

La inmutabilidad permite:

```text
safe caching
deterministic fingerprints
schema diff
parallel analysis
persistent runtime reuse
stable diagnostics
safe planning
reproducible compilation
```

---

# 10. TableDefinition ≠ TableBlueprint

Ejemplo:

```php
$table->string('email')->unique();
```

es una acción de construcción.

Después del lowering:

```text
TableDefinition
├── ColumnDefinition(email)
└── UniqueConstraint(email)
```

La fluent API desaparece.

---

# 11. TableDefinition ≠ Schema AST operation

Una definición responde:

```text
What is this table structurally?
```

Una operación AST responde:

```text
What schema change is requested?
```

Por ejemplo:

```text
CreateTableNode
└── TableDefinition(users)
```

---

# 12. TableDefinition ≠ current database state

Una definición puede representar:

```text
desired table
observed table
expected table
fixture table
migration target table
```

Por ello, el origen debe ser metadata/contexto, no una suposición implícita.

---

# 13. TableDefinition ≠ SQL

Esto:

```text
TableDefinition(users)
```

no conoce:

```sql
CREATE TABLE `users` (...)
```

ni:

```sql
CREATE TABLE "users" (...)
```

El quoting pertenece al compiler.

---

# 14. TableIdentifier

Toda tabla tendrá:

```text
TableIdentifier
```

tipado.

Ejemplo:

```php
$table->identifier();
```

podría devolver:

```php
TableIdentifier::from('users');
```

---

# 15. QualifiedTableIdentifier

Para bases que soporten namespaces:

```text
QualifiedTableIdentifier
├── CatalogIdentifier?
├── SchemaIdentifier?
└── TableIdentifier
```

Ejemplo:

```text
analytics.public.events
```

no deberá almacenarse únicamente como string concatenado.

---

# 16. Identidad lógica vs física

Debe poder distinguirse:

```text
LogicalTableIdentity
```

de:

```text
PhysicalTableIdentifier
```

cuando naming strategies, tenancy o platform resolution lo requieran.

---

# 17. TableDefinitionId

Durante procesamiento interno puede existir:

```php
final readonly class TableDefinitionId
{
}
```

pero:

```text
TableDefinitionId
≠
TableIdentifier
```

---

# 18. TableDefinitionId purpose

Puede servir para:

- graph references;
- diagnostics;
- source mapping;
- AST mapping;
- diff correlation;
- internal identity.

No deberá convertirse automáticamente en nombre SQL.

---

# 19. Table kind

Se propone:

```php
enum TableKind
{
    case REGULAR;
    case TEMPORARY;
    case UNLOGGED;
    case EXTENSION;
}
```

No todas las plataformas soportarán todos los tipos.

---

# 20. Table kind ≠ capability

Una definición puede solicitar:

```text
TEMPORARY
```

aunque una plataforma concreta no pueda satisfacer cierta variante.

La incompatibilidad se detectará downstream.

---

# 21. ColumnDefinitionSet

Las columnas deberán almacenarse en:

```text
ColumnDefinitionSet
```

no en un array genérico sin invariantes.

---

# 22. Column order

El orden de columnas deberá preservarse.

Formalmente:

```text
Columns =
[c₁, c₂, ..., cₙ]
```

y:

```text
position(cᵢ) < position(cᵢ₊₁)
```

---

# 23. Por qué preservar orden

Aunque muchas operaciones relacionales no dependen del orden físico, éste puede ser relevante para:

- introspection;
- schema diff;
- diagnostics;
- `SELECT *` compatibility;
- dump tooling;
- legacy databases;
- migration analysis;
- human-readable output.

---

# 24. Column identity

Dentro de una tabla:

```text
ColumnIdentifier
```

deberá ser único según las reglas lógicas aplicables.

---

# 25. Duplicate columns

Esto será inválido:

```text
users
├── email STRING
└── email INTEGER
```

---

# 26. Case sensitivity

No deberá asumirse universalmente:

```text
email == EMAIL
```

ni:

```text
email != EMAIL
```

La comparación física puede depender de plataforma.

El modelo deberá distinguir:

```text
logical identifier representation
```

de:

```text
platform identifier comparison rules
```

---

# 27. Portable profile

VoltStack podrá ofrecer una política portable que rechace nombres que sólo se diferencien por casing:

```text
email
EMAIL
Email
```

para evitar problemas cross-platform.

---

# 28. Column lookup

`ColumnDefinitionSet` podrá proporcionar:

```php
public function get(ColumnIdentifier $id): ColumnDefinition;

public function has(ColumnIdentifier $id): bool;

public function ordered(): iterable;
```

---

# 29. No mutable arrays

Evitar:

```php
public array $columns;
```

como API pública principal.

Preferir:

```text
immutable typed collection
```

---

# 30. ColumnDefinition dependency

`TableDefinition System` utilizará el futuro:

```text
92_DATABASE_COLUMN_DEFINITION_SYSTEM
```

para formalizar cada columna.

Por ahora:

```text
TableDefinition
→ ColumnDefinition
```

sin duplicar sus responsabilidades.

---

# 31. Primary key

Una tabla podrá contener:

```text
0..1 PrimaryKeyDefinition
```

---

# 32. Primary key cardinality

Formalmente:

```text
|PrimaryKey(Table)| ≤ 1
```

Una primary key puede contener múltiples columnas.

---

# 33. Composite primary key

Ejemplo:

```text
PrimaryKey
├── tenant_id
└── user_id
```

es válido.

---

# 34. Primary key reference integrity

Toda columna referenciada por la primary key deberá existir.

Formalmente:

```text
PKColumns(Table)
⊆
Columns(Table)
```

---

# 35. Primary key order

Para una PK compuesta:

```text
(tenant_id, user_id)
```

el orden deberá preservarse.

No deberá normalizarse a:

```text
(user_id, tenant_id)
```

---

# 36. Primary key name

La definición podrá contener:

```text
LogicalConstraintName?
PhysicalConstraintName?
```

según el nivel arquitectónico.

Preferentemente, `TableDefinition` conserva el nombre declarado/lógico.

---

# 37. Unique constraints

Se modelarán mediante:

```text
UniqueConstraintDefinitionSet
```

---

# 38. Unique constraint ≠ unique index

Esta distinción es importante.

```text
UniqueConstraint
≠
UniqueIndex
```

Aunque una base pueda implementar una constraint mediante un índice.

El modelo semántico no deberá colapsarlos.

---

# 39. Why distinction matters

Porque pueden variar:

- metadata;
- naming;
- referential eligibility;
- deferrability;
- platform behavior;
- introspection;
- schema diff;
- DDL syntax.

---

# 40. Unique column helper

Una DSL como:

```php
$table->string('email')->unique();
```

deberá lowerse a:

```text
ColumnDefinition(email)
+
UniqueConstraintDefinition(email)
```

o a la política semántica acordada.

No deberá simplemente establecer:

```text
column.unique = true
```

si ello destruye la identidad de la constraint.

---

# 41. Index definitions

Los índices pertenecerán a:

```text
IndexDefinitionSet
```

y serán desarrollados formalmente en:

```text
93_DATABASE_INDEX_SYSTEM.md
```

---

# 42. Index membership

Todo column-based index deberá referenciar columnas existentes.

```text
IndexColumns(Table)
⊆
Columns(Table)
```

salvo expression indexes donde el modelo será diferente.

---

# 43. Expression indexes

Una definición podrá contener:

```text
IndexKey
├── ColumnIndexKey
└── ExpressionIndexKey
```

por lo que:

```text
index keys
```

no deberán reducirse a una simple lista de strings.

---

# 44. Foreign keys

Las foreign keys se almacenarán en:

```text
ForeignKeyDefinitionSet
```

---

# 45. Local FK integrity

Para:

```text
posts.user_id → users.id
```

`posts.user_id` deberá existir en la `TableDefinition(posts)`.

---

# 46. Referenced table availability

La tabla referenciada:

```text
users
```

no necesariamente estará dentro del mismo `TableDefinition`.

Por tanto, la validación completa de FK requiere:

```text
Schema Model / Schema AST context
```

---

# 47. Local vs global validation

`TableDefinition` puede validar:

```text
local columns exist
```

pero no necesariamente:

```text
referenced external table exists
```

sin contexto.

---

# 48. Self-referencing FK

Esto es válido:

```text
categories.parent_id
    ↓
categories.id
```

La definición deberá representarlo naturalmente.

---

# 49. Composite FK

Debe preservarse:

```text
(local_tenant_id, local_user_id)
        ↓
(tenant_id, id)
```

incluyendo orden.

---

# 50. FK arity

Debe cumplirse:

```text
count(local columns)
=
count(referenced columns)
```

---

# 51. Check constraints

Las check constraints se almacenarán mediante:

```text
CheckConstraintDefinitionSet
```

---

# 52. Check expressions

No deberán representarse sólo como strings.

Preferir:

```text
SchemaExpression
```

tipada.

---

# 53. Check dependencies

Una expresión:

```text
price >= 0
```

puede registrar dependencia:

```text
ColumnIdentifier(price)
```

para validación y schema diff.

---

# 54. Raw check

Cuando se use raw SQL:

```text
RawSchemaExpression
```

deberá quedar marcado explícitamente.

---

# 55. TableOptionSet

Las opciones estructurales estarán en:

```text
TableOptionSet
```

---

# 56. Opciones portables

Ejemplos conceptuales:

```text
comment
temporary characteristics
logical collation
```

cuando exista semántica portable.

---

# 57. Opciones específicas

Ejemplos:

```text
MySQL engine
PostgreSQL storage parameters
SQLite STRICT
SQLite WITHOUT ROWID
```

deberán representarse como opciones tipadas específicas.

---

# 58. No generic option bag

Evitar:

```php
$options = [
    'engine' => 'InnoDB',
    'foo' => 'bar',
];
```

como representación interna principal.

Preferir:

```text
TableOption
├── PortableTableOption
├── MySqlTableOption
├── MariaDbTableOption
├── PostgreSqlTableOption
├── SqliteTableOption
└── ExtensionTableOption
```

---

# 59. Unknown options

Una opción desconocida deberá:

```text
fail validation
```

o ser una:

```text
explicit ExtensionTableOption
```

No deberá ignorarse silenciosamente.

---

# 60. TableMetadata

Se propone:

```php
final readonly class TableMetadata
{
    public function __construct(
        public ?SchemaSourceLocation $source,
        public SchemaObjectOrigin $origin,
        public AnnotationSet $annotations,
        public SensitivityMetadata $sensitivity,
    ) {}
}
```

---

# 61. Metadata ≠ structural semantics

Debe distinguirse:

```text
Structural Table Data
```

de:

```text
Diagnostic/Operational Metadata
```

---

# 62. Metadata must not accidentally affect equality

Por ejemplo:

```text
source file line 42
```

no debería hacer que dos tablas estructuralmente idénticas sean distintas.

---

# 63. Equality models

VoltStack deberá distinguir al menos:

```text
Object Identity
Structural Equality
Normalized Structural Equality
Physical Compatibility
Migration Equivalence
```

---

# 64. Object identity

Dos objetos PHP diferentes:

```php
$a !== $b;
```

pueden representar la misma estructura.

---

# 65. Structural equality

Formalmente:

```text
StructuralEqual(A, B)
```

si poseen las mismas definiciones estructurales relevantes.

---

# 66. Normalized structural equality

Puede ignorar diferencias equivalentes después de normalización.

Ejemplo:

```text
implicit default option
vs
explicit default option
```

cuando la política permita considerarlas equivalentes.

---

# 67. Physical compatibility

Dos definiciones semánticas podrían ser compatibles en una plataforma aunque no sean estructuralmente idénticas.

Eso pertenece a:

```text
Schema Platform Compatibility
```

no a igualdad básica.

---

# 68. Migration equivalence

Tampoco debe confundirse:

```text
same final schema
```

con:

```text
same migration operation
```

Ejemplo:

```text
RenameColumn
```

y:

```text
DropColumn + AddColumn
```

pueden producir formas similares en ciertos casos, pero no son la misma operación.

---

# 69. Structural fingerprint

`TableDefinition` podrá generar o asociarse a:

```text
TableStructuralFingerprint
```

---

# 70. Fingerprint inputs

Puede considerar:

```text
identifier
table kind
columns
column order
primary key
unique constraints
indexes
foreign keys
checks
structural options
extension structural data
normalization version
```

---

# 71. Fingerprint exclusions

Normalmente excluir:

```text
source line
runtime object ID
timestamps of object creation
request ID
telemetry metadata
```

---

# 72. Fingerprint ≠ SQL hash

Nunca:

```text
fingerprint = hash(CREATE TABLE SQL)
```

como definición arquitectónica principal.

---

# 73. Fingerprint version

Debe incluirse:

```text
FingerprintAlgorithmVersion
```

para evolución futura.

---

# 74. Deterministic serialization

Una `TableDefinition` deberá poder serializarse de manera determinista.

Ejemplo conceptual:

```json
{
  "name": "users",
  "columns": [],
  "primaryKey": {},
  "indexes": [],
  "foreignKeys": []
}
```

El formato real deberá ser versionado.

---

# 75. Serialization ≠ PHP serialize()

No depender exclusivamente de:

```php
serialize($table);
```

porque acopla el formato a clases PHP concretas.

---

# 76. Schema Definition Format

Puede existir:

```text
SchemaDefinitionSerializer
```

con:

```text
formatVersion
type IDs
stable field ordering
extension IDs
```

---

# 77. Why serialization matters

Permitirá:

```text
schema cache
schema snapshots
migration planning
schema diff
testing snapshots
offline analysis
tooling
IDE integrations
```

---

# 78. Origin

Una tabla puede tener:

```php
enum SchemaObjectOrigin
{
    case DECLARED;
    case INTROSPECTED;
    case GENERATED;
    case MIGRATION;
    case EXTENSION;
    case TEST;
}
```

---

# 79. Origin ≠ certainty

Una tabla introspectada puede contener campos con distintos niveles de certeza.

Por tanto:

```text
Origin
≠
MetadataCertainty
```

---

# 80. Certainty model

Puede existir:

```text
EXACT
DECLARED
DRIVER_REPORTED
INFERRED
UNKNOWN
```

para metadata introspectada.

---

# 81. Desired vs observed table

Se recomienda no sobrecargar un único boolean:

```php
$isExisting;
```

Preferir contexto explícito.

Por ejemplo:

```text
DesiredTableDefinition
ObservedTableDefinition
```

o wrappers equivalentes.

---

# 82. Shared structural core

Ambos pueden compartir:

```text
TableDefinition
```

como núcleo estructural.

Ejemplo:

```text
DesiredTable
└── TableDefinition

ObservedTable
├── TableDefinition
└── ObservationMetadata
```

---

# 83. Avoid polluting TableDefinition

No agregar directamente:

```text
databaseServerVersion
connectionId
introspectionTimestamp
driverHandle
```

a `TableDefinition`.

Eso pertenece al snapshot/observation context.

---

# 84. Table comments

Si los comentarios son parte del schema objetivo, deberán modelarse estructuralmente.

Pero su tratamiento en fingerprints/diffs deberá ser configurable según propósito.

---

# 85. Semantic profiles

Puede existir:

```text
SchemaComparisonProfile
```

con perfiles como:

```text
FULL
STRUCTURAL
PORTABLE
MIGRATION
PERFORMANCE
```

---

# 86. FULL profile

Compara prácticamente toda la definición estructural.

---

# 87. PORTABLE profile

Puede ignorar opciones específicas no relevantes para una comparación portable.

---

# 88. MIGRATION profile

Evalúa diferencias que requieren acciones migratorias.

---

# 89. Performance profile

Podrá enfatizar:

```text
indexes
storage options
partition metadata
```

pero no debe reemplazar la igualdad estructural.

---

# 90. TableDefinitionFactory

Se propone:

```php
interface TableDefinitionFactory
{
    public function fromBlueprint(
        TableBlueprint $blueprint,
        TableDefinitionContext $context,
    ): TableDefinition;
}
```

---

# 91. Factory responsibility

La factory:

- sella/consume blueprint;
- valida invariantes;
- normaliza definiciones;
- crea typed collections;
- produce metadata;
- construye `TableDefinition`.

No:

- compila SQL;
- consulta DB;
- ejecuta DDL.

---

# 92. TableDefinitionBuilder?

No se recomienda introducir otro mutable builder público llamado:

```text
TableDefinitionBuilder
```

si `TableBlueprint` ya cumple esa función.

Evitar:

```text
Builder → Builder → Definition
```

sin necesidad.

---

# 93. Internal assembler

Sí puede existir:

```text
TableDefinitionAssembler
```

como servicio interno puro.

---

# 94. Validation pipeline

```text
Blueprint
   ↓
Local Blueprint Validation
   ↓
Lowering
   ↓
TableDefinition
   ↓
Definition Validation
   ↓
Validated TableDefinition
```

---

# 95. Definition validation

Deberá comprobar:

```text
unique local column identifiers
valid PK references
valid local FK references
valid index references
valid constraint references
option consistency
extension consistency
identifier consistency
```

---

# 96. Validation without database

Toda validación local deberá poder realizarse:

```text
without DB connection
```

---

# 97. Cross-table validation

Requiere:

```text
DatabaseSchema
```

o:

```text
SchemaValidationContext
```

---

# 98. Example

Tabla:

```text
posts
└── FK user_id → users.id
```

`TableDefinition(posts)` puede comprobar:

```text
user_id exists locally
```

pero necesita contexto para comprobar:

```text
users.id exists
```

---

# 99. Referential type compatibility

Comprobar:

```text
posts.user_id type
compatible with
users.id type
```

también requiere conocimiento de ambas tablas.

---

# 100. Validation layers

```text
Level 1
Table-local invariants

Level 2
Schema-global invariants

Level 3
Platform capability compatibility

Level 4
Migration safety

Level 5
Execution feasibility
```

`TableDefinition System` se centra principalmente en Level 1.

---

# 101. No capability resolution

`TableDefinition` no deberá preguntar:

```php
$platform->supportsGeneratedColumns();
```

---

# 102. Capability requirements

Sin embargo, puede declarar:

```text
SchemaCapabilityRequirementSet
```

derivado de sus características.

---

# 103. Example capability requirements

Una tabla con:

```text
generated column
deferrable FK
expression index
```

puede implicar:

```text
GENERATED_COLUMNS
DEFERRABLE_CONSTRAINTS
EXPRESSION_INDEXES
```

---

# 104. Requirement ≠ support

```text
Table requires X
```

no significa:

```text
Platform supports X
```

La comparación ocurre downstream.

---

# 105. Dependency extraction

Una `TableDefinition` deberá permitir derivar:

```text
SchemaDependencySet
```

---

# 106. Dependency types

Ejemplos:

```text
ForeignTableDependency
SequenceDependency
TypeDependency
CollationDependency
ExtensionDependency
FunctionDependency
```

---

# 107. Why dependencies matter

Permiten al planner determinar:

```text
creation order
drop order
cycles
required extension objects
migration sequencing
```

---

# 108. Dependency extraction ≠ planning

`TableDefinition` puede declarar:

```text
I reference users
```

pero no decidir:

```text
create users before posts
```

Eso pertenece al planner.

---

# 109. Table-level constraints

Conceptualmente:

```text
TableConstraintDefinition
├── PrimaryKeyDefinition
├── UniqueConstraintDefinition
├── ForeignKeyDefinition
├── CheckConstraintDefinition
└── ExtensionConstraintDefinition
```

---

# 110. Index ≠ constraint

Índices deberán permanecer separados:

```text
TableDefinition
├── Constraints
└── Indexes
```

aunque algunos motores implementen constraints usando índices.

---

# 111. Column-level syntax ≠ column-level semantics

SQL permite escribir:

```sql
email VARCHAR(255) UNIQUE
```

pero el modelo semántico puede normalizarlo como:

```text
Column(email)
+
UniqueConstraint(email)
```

---

# 112. Normalization

La definición deberá preferir una forma canónica.

Ejemplo:

```text
inline UNIQUE
```

y:

```text
table-level UNIQUE(email)
```

podrán normalizarse al mismo concepto cuando semánticamente sean equivalentes.

---

# 113. Canonical table representation

Objetivo:

```text
Many equivalent DSL forms
        ↓
One canonical TableDefinition
```

---

# 114. Canonicalization benefits

```text
stable diff
stable fingerprint
simpler compiler
simpler planner
better testing
better caching
```

---

# 115. Canonicalization must preserve semantics

Nunca convertir:

```text
UniqueConstraint
```

a:

```text
Index
```

sólo porque una plataforma lo haga internamente.

---

# 116. Table normalization pipeline

```text
Raw Definitions
      ↓
Identifier normalization
      ↓
Column normalization
      ↓
Constraint normalization
      ↓
Index normalization
      ↓
Option normalization
      ↓
Metadata normalization
      ↓
Canonical TableDefinition
```

---

# 117. Normalization ≠ platform compilation

No debe convertir:

```text
BooleanType
→ TINYINT(1)
```

en este nivel.

Eso es platform-specific.

---

# 118. Table annotations

Podrán existir:

```text
SchemaAnnotationSet
```

para tooling/framework metadata.

---

# 119. Annotation namespace

Cada annotation deberá tener un ID/namespaced key.

Ejemplo:

```text
voltstack.orm.entity
extension.vector.index
```

---

# 120. Structural annotations

Algunas annotations pueden afectar schema semantics.

Otras serán sólo metadata.

Esto debe declararse explícitamente.

---

# 121. Fingerprint impact declaration

Una extensión deberá indicar si su metadata:

```text
affectsStructuralFingerprint = true|false
```

---

# 122. No arbitrary metadata mutation

Después de crear `TableDefinition`, las annotations tampoco deberán modificarse.

---

# 123. Extension metadata

Se propone:

```text
ExtensionMetadataSet
├── ExtensionId
├── Version
├── Payload
└── StructuralImpact
```

---

# 124. Typed extension payload

Evitar:

```php
['anything' => mixed]
```

sin descriptor.

---

# 125. Unknown extension

Una definición que requiera una extensión desconocida deberá poder deserializarse de forma segura o fallar de forma explícita según contexto.

Nunca:

```text
silently discard unknown extension metadata
```

---

# 126. Extension compatibility

Schema cache/snapshots deberán considerar:

```text
extension ID
extension version
serialization contract
```

cuando afecten estructura.

---

# 127. Table options conflicts

Ejemplo conceptual:

```text
TEMPORARY
+
UNLOGGED
```

podría ser inválido según modelo.

La validación deberá detectar combinaciones contradictorias antes del compiler cuando sean lógicamente incompatibles.

---

# 128. Platform-only conflicts

Si la incompatibilidad depende de plataforma, se pospone a:

```text
Schema Platform Compatibility System
```

---

# 129. Empty table

VoltStack deberá decidir si:

```text
CREATE TABLE empty ()
```

es representable semánticamente.

Recomendación:

```text
TableDefinition may represent zero columns
```

si el modelo lo permite, mientras el platform validator decide compatibilidad.

---

# 130. Why allow representability

El modelo semántico no debe imponer restricciones de una plataforma concreta salvo invariantes universales.

---

# 131. Column position metadata

La posición puede derivarse directamente de:

```text
ColumnDefinitionSet ordering
```

en vez de duplicarse como:

```text
column.position
```

cuando no sea necesario.

---

# 132. Avoid duplicated truth

No mantener simultáneamente:

```text
columns ordered array
+
column.position
```

si ambos pueden divergir.

---

# 133. Table-level default collation

Puede representarse en:

```text
TableOptionSet
```

---

# 134. Column override

Una columna podrá especificar su propia collation.

La resolución conceptual:

```text
Column explicit
>
Table default
>
Schema/database default
```

puede ser evaluada downstream.

---

# 135. Do not materialize inherited defaults too early

Si una columna no especifica collation:

```text
column.collation = INHERIT
```

puede ser más correcto que copiar inmediatamente la table collation.

---

# 136. Explicit vs inherited

Debe distinguirse:

```text
EXPLICIT
INHERITED
UNSPECIFIED
PLATFORM_DEFAULT
```

cuando sea relevante.

---

# 137. Why explicitness matters

Schema Diff necesita distinguir:

```text
developer explicitly requested X
```

de:

```text
database happened to report X
```

---

# 138. Default value provenance

La misma idea aplica a:

```text
defaults
collations
charsets
generated names
platform defaults
```

---

# 139. Constraint naming provenance

Un nombre puede ser:

```text
EXPLICIT
GENERATED
PLATFORM_GENERATED
INTROSPECTED
UNKNOWN
```

---

# 140. Schema Diff implication

Esto evita migrations innecesarias cuando una base asigna nombres automáticamente.

---

# 141. Table comments provenance

Igualmente:

```text
declared comment
observed comment
unknown/unavailable
```

no deberán confundirse.

---

# 142. Unknown ≠ absent

Principio crítico:

```text
UNKNOWN
≠
ABSENT
```

Especialmente para introspection parcial.

---

# 143. Definition completeness

Podrá existir:

```php
enum DefinitionCompleteness
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

para wrappers de observación.

---

# 144. Desired definitions

Una definición declarada por Builder debería normalmente ser:

```text
COMPLETE
```

respecto a los campos expresados por el modelo.

---

# 145. Observed definitions

Un driver podría no reportar:

```text
check constraints
comments
expression details
```

y producir información parcial.

---

# 146. Do not fake certainty

El Schema Model deberá conservar:

```text
UNKNOWN
```

cuando el driver no pueda proporcionar metadata fiable.

---

# 147. TableDefinition purity

Idealmente:

```text
TableDefinition
```

será un value object estructural puro.

Información de observación se coloca alrededor:

```text
ObservedTable
├── definition
├── completeness
├── certainty
└── provenance
```

---

# 148. Structural transformations

Podrá existir:

```php
interface TableDefinitionTransformer
{
    public function transform(
        TableDefinition $table
    ): TableDefinition;
}
```

---

# 149. Transformer restrictions

Un transformer estructural:

- no ejecuta queries;
- no abre conexiones;
- no modifica input;
- es determinista;
- declara extension/capability requirements;
- produce nueva definición.

---

# 150. Example transformation

```text
TableDefinition(users)
        │
        │ add audit columns
        ▼
TableDefinition(users')
```

Pero en migration semantics, esta transformación deberá convertirse posteriormente en operaciones explícitas.

---

# 151. Definition transformation ≠ migration

Comparar dos estados:

```text
A → B
```

no es lo mismo que declarar cómo migrar:

```text
Migration(A, B)
```

---

# 152. Schema Diff responsibility

El futuro:

```text
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
```

determinará diferencias entre definiciones.

---

# 153. TableDefinition must be diff-friendly

Por tanto debe ofrecer:

```text
stable identities
canonical collections
typed options
typed constraints
stable fingerprints
explicit unknowns
```

---

# 154. Column rename problem

Dado:

```text
A:
name

B:
full_name
```

`TableDefinition` por sí sola no necesariamente puede saber que hubo rename.

Puede parecer:

```text
drop name
add full_name
```

---

# 155. Rename intent belongs to operation history

Por eso:

```text
TableDefinition
```

no reemplaza:

```text
RenameColumnNode
```

en Schema AST.

---

# 156. State model vs operation model

```text
TableDefinition
=
state
```

```text
Schema AST mutation nodes
=
operations
```

Ambos son necesarios.

---

# 157. Create table AST

```text
CreateTableNode
└── TableDefinition
```

es natural porque la operación crea un estado completo.

---

# 158. Alter table AST

Para alteraciones:

```text
AlterTableNode
├── AddColumnNode
├── RenameColumnNode
├── AlterColumnNode
└── DropColumnNode
```

es preferible a almacenar sólo:

```text
before TableDefinition
after TableDefinition
```

cuando se necesita preservar intención.

---

# 159. Desired final state

Schema Diff sí podrá generar un conjunto de operaciones a partir de:

```text
ObservedTableDefinition
vs
DesiredTableDefinition
```

---

# 160. TableDefinition and ORM

El ORM podrá leer schema metadata en ciertas herramientas.

Pero:

```text
TableDefinition
≠
EntityMetadata
```

---

# 161. Table vs entity

Una tabla:

```text
users
```

puede mapear a:

```text
User entity
```

pero también:

- no mapear a ninguna entity;
- mapear a varias projections;
- ser pivot table;
- ser audit table;
- ser infrastructure table.

---

# 162. No ORM concepts in TableDefinition

No incluir:

```text
entity class
repository class
UnitOfWork state
dirty tracking
lazy loading
```

en el núcleo de TableDefinition.

---

# 163. ORM integration metadata

Si ORM necesita vincular una tabla:

```text
ORM metadata
→ references TableIdentifier
```

No al revés.

---

# 164. TableDefinition and Query Engine

Query semantic resolution puede consultar Schema Model basado en definiciones.

Pero:

```text
TableDefinition
```

no depende de Query AST.

---

# 165. Dependency direction

Preferible:

```text
Query Semantic Engine
        ↓
Schema Model
        ↓
TableDefinition
```

Nunca:

```text
TableDefinition
→ Query Builder
```

---

# 166. TableDefinition and Compiler

Compiler consume información estructural.

Pero TableDefinition no conoce compiler.

```text
Schema Compiler
      ↓
TableDefinition
```

---

# 167. TableDefinition and Driver

No dependencia:

```text
TableDefinition
→ PDO
→ mysqli
→ pgsql handle
→ SQLite connection
```

---

# 168. TableDefinition and Platform

El core definition model puede usar IDs/capability requirement concepts compartidos.

No deberá contener lógica:

```php
if ($platform === 'mysql') { ... }
```

---

# 169. TableDefinition cache

Por ser inmutable, podrá almacenarse en:

```text
Schema Metadata Cache
```

más adelante.

---

# 170. Cache key

No deberá depender únicamente del nombre:

```text
users
```

porque podrían existir:

```text
connection A.users
connection B.users
schema X.users
schema Y.users
tenant-specific contexts
```

---

# 171. Context identity

El cache layer deberá añadir:

```text
database identity
schema namespace
snapshot version
platform context
tenant scope if applicable
```

fuera de la definición cuando corresponda.

---

# 172. No tenant data in shared definition

Una definición compartida sólo podrá reutilizarse entre tenants cuando se haya probado:

```text
structural equivalence
```

y no contenga metadata tenant-specific.

---

# 173. Persistent runtime safety

Permitido compartir:

```text
immutable TableDefinition
frozen registries
immutable type descriptors
immutable normalized metadata
```

No permitido compartir accidentalmente:

```text
mutable builders
current schema context
connection handles
tenant context
request state
```

---

# 174. Thread/coroutine safety

Una `TableDefinition` completamente inmutable será naturalmente segura para lectura concurrente.

---

# 175. No lazy mutable caches inside definition

Evitar:

```php
private ?string $fingerprint = null;
```

si introduce mutación insegura.

Preferir:

- cálculo externo;
- memoization immutable-safe;
- precomputed fingerprint;
- runtime cache separado.

---

# 176. Fingerprint service

Puede existir:

```php
interface TableFingerprintGenerator
{
    public function generate(
        TableDefinition $table,
        TableFingerprintProfile $profile,
    ): TableStructuralFingerprint;
}
```

---

# 177. Why external fingerprint service

Permite diferentes fingerprints:

```text
FULL
PORTABLE
MIGRATION
CACHE
```

sin contaminar el value object.

---

# 178. Comparison service

Igualmente:

```php
interface TableDefinitionComparator
{
    public function compare(
        TableDefinition $left,
        TableDefinition $right,
        SchemaComparisonProfile $profile,
    ): TableComparisonResult;
}
```

---

# 179. Comparator ≠ Schema Diff

Comparator puede determinar diferencias estructurales.

Schema Diff deberá convertirlas en:

```text
schema changes
dependencies
migration operations
safety classification
```

---

# 180. Table validation result

Puede utilizarse:

```php
final readonly class TableValidationResult
{
    public function __construct(
        public bool $valid,
        public SchemaDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 181. Fail-fast vs collect diagnostics

Se recomienda soportar:

```text
FAIL_FAST
COLLECT_ALL
```

para diferentes contextos.

---

# 182. Builder mode

Developer runtime puede preferir:

```text
FAIL_FAST
```

---

# 183. Tooling mode

CLI/schema analyzer puede preferir:

```text
COLLECT_ALL
```

---

# 184. Diagnostics

Ejemplos:

```text
Duplicate column "email".
Primary key references missing column "id".
Foreign key references missing local column "user_id".
Index "users_email_idx" references unknown column "email".
Constraint name collision.
```

---

# 185. Diagnostic source

Cuando exista source mapping:

```text
database/migrations/2026_09_05_create_users.php:18
```

puede propagarse.

---

# 186. Error hierarchy

Se propone:

```text
DatabaseTableDefinitionException
├── InvalidTableDefinitionException
├── InvalidTableIdentifierException
├── DuplicateColumnException
├── MissingPrimaryKeyColumnException
├── InvalidPrimaryKeyDefinitionException
├── InvalidUniqueConstraintException
├── InvalidIndexReferenceException
├── InvalidForeignKeyReferenceException
├── InvalidCheckConstraintException
├── ConflictingTableOptionException
├── UnknownTableOptionException
├── TableDefinitionSerializationException
├── TableDefinitionExtensionException
├── TableDefinitionBudgetExceededException
└── TableDefinitionInvariantException
```

---

# 187. Definition budgets

Para proteger tooling y inputs generados:

```php
final readonly class TableDefinitionBudget
{
    public function __construct(
        public int $maxColumns,
        public int $maxIndexes,
        public int $maxForeignKeys,
        public int $maxConstraints,
        public int $maxExpressionDepth,
        public int $maxMetadataEntries,
        public int $maxExtensionPayloadBytes,
    ) {}
}
```

---

# 188. Budget semantics

Si se excede:

```text
fail explicitly
```

No:

```text
truncate definition
```

---

# 189. Large legacy schemas

Los límites deberán ser configurables para soportar:

```text
legacy enterprise databases
generated schemas
analytics tables
wide tables
```

sin eliminar protección.

---

# 190. Table structural complexity

Puede estimarse:

```text
Complexity(Table)
=
Columns
+
Indexes
+
Constraints
+
ExpressionNodes
+
ExtensionWeight
```

para resource governance.

---

# 191. Security

`TableDefinition` debe tratar nombres, comments y metadata como datos estructurales potencialmente sensibles.

---

# 192. No automatic logging

No registrar automáticamente:

```text
table comments
raw expressions
extension payloads
```

en telemetry.

---

# 193. Sensitive table metadata

Puede existir:

```text
SensitivityMetadata
```

para clasificar:

```text
PUBLIC
INTERNAL
SENSITIVE
SECRET
```

---

# 194. Sensitivity ≠ authorization

Una tabla marcada `SECRET` no implementa acceso.

La autorización pertenece a:

```text
Security / Authorization layers
```

---

# 195. Raw structural content

Raw expressions/options deberán conservar un marcador:

```text
TrustLevel
```

o equivalente.

---

# 196. Trust does not disappear after lowering

Si el Builder produjo:

```text
RawSchemaExpression
```

`TableDefinition` deberá conservar ese hecho.

---

# 197. Compiler safety

Así el compiler puede aplicar políticas como:

```text
raw schema expressions disabled
```

sin perder provenance.

---

# 198. TableDefinition extension model

Una extensión podrá aportar:

```text
custom table option
custom constraint
custom metadata
custom dependency
custom capability requirement
```

---

# 199. Extension cannot redefine core semantics silently

Ejemplo prohibido:

```text
extension changes meaning of PRIMARY KEY
```

sin una extensión privilegiada explícita del framework.

---

# 200. Frozen extension descriptors

Toda interpretación de extension metadata deberá depender de:

```text
frozen extension registry
```

---

# 201. No last-wins extension behavior

Conflictos deberán fallar.

---

# 202. Unknown extension during comparison

Debe producir:

```text
comparison uncertainty
```

o error explícito según policy.

Nunca asumir igualdad.

---

# 203. Unknown extension during compilation

Si afecta estructura:

```text
compilation must fail
```

si el compiler correspondiente no existe.

---

# 204. Testing

`TableDefinition System` deberá tener pruebas independientes de DB.

---

# 205. Minimal table test

```php
$table = new TableDefinition(
    identifier: TableIdentifier::from('users'),
    kind: TableKind::REGULAR,
    columns: ColumnDefinitionSet::of(
        ColumnDefinition::integer('id')
    ),
    primaryKey: PrimaryKeyDefinition::of('id'),
    uniqueConstraints: UniqueConstraintDefinitionSet::empty(),
    indexes: IndexDefinitionSet::empty(),
    foreignKeys: ForeignKeyDefinitionSet::empty(),
    checks: CheckConstraintDefinitionSet::empty(),
    options: TableOptionSet::empty(),
    metadata: TableMetadata::empty(),
    extensions: ExtensionMetadataSet::empty(),
);
```

---

# 206. Structural equality test

```php
self::assertTrue(
    $comparator->structurallyEquals(
        $tableA,
        $tableB
    )
);
```

---

# 207. Deterministic fingerprint test

```text
Fingerprint(A)
=
Fingerprint(B)
```

cuando:

```text
StructuralEqual(A, B)
```

bajo el mismo fingerprint profile/version.

---

# 208. Different order test

Para columnas:

```text
[id, email]
```

vs:

```text
[email, id]
```

la política estructural deberá decidir explícitamente si el orden forma parte del fingerprint.

Recomendación:

```text
yes
```

en `FULL`.

---

# 209. Constraint order

El orden de declaración de constraints normalmente no debería alterar igualdad semántica.

Por tanto pueden canonicalizarse por identidad estable.

---

# 210. Important distinction

```text
Column order
=
potentially structural/observable
```

```text
Constraint declaration order
=
usually non-semantic
```

---

# 211. Index key order

Sí es semántico.

```text
INDEX(a, b)
≠
INDEX(b, a)
```

---

# 212. FK column order

También:

```text
FK(a, b) → T(x, y)
```

no puede reordenarse arbitrariamente.

---

# 213. PK column order

Igualmente se preserva.

---

# 214. Check expression normalization

Podrá utilizar canonicalización de `SchemaExpression`, pero sólo si preserva semántica.

---

# 215. No unsafe expression algebra

No asumir:

```text
a AND b
=
b AND a
```

para fingerprint si las expresiones incluyen elementos con semántica especial, volatility o extensions no conocidas.

---

# 216. Schema expression safety

La canonicalización deberá apoyarse en metadata semántica, no sólo en strings.

---

# 217. Cross-driver introspection tests

Cuando se implemente introspection:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

deberán mapear sus resultados al modelo común sin perder diferencias importantes.

---

# 218. Round-trip tests

Cuando sea posible:

```text
TableDefinition
   ↓ compile/create
Database
   ↓ introspect
ObservedTableDefinition
```

deberá producir una definición compatible.

Formalmente:

```text
Compatible(
    Desired,
    Introspect(Execute(Compile(Desired)))
)
```

---

# 219. Round-trip ≠ byte equality

No se exige que:

```text
DesiredDefinition
==
ObservedDefinition
```

byte por byte, porque la plataforma puede introducir:

```text
generated names
normalized defaults
physical types
implicit indexes
```

---

# 220. Compatibility layer

La comparación deberá pasar por:

```text
Schema Platform Compatibility System
```

más adelante.

---

# 221. Serialization tests

Probar:

```text
serialize
deserialize
compare
```

---

# 222. Serialization invariant

```text
StructuralEqual(
    T,
    Deserialize(Serialize(T))
)
```

deberá ser verdadero para versiones compatibles.

---

# 223. Persistent runtime tests

Crear definición en operación A.

Leerla concurrentemente desde contextos permitidos.

Verificar:

```text
no mutable state leakage
```

---

# 224. Extension tests

Verificar:

- extension metadata preserved;
- fingerprint impact correct;
- unknown extensions handled safely;
- conflicts rejected;
- serialization version respected.

---

# 225. Fuzz testing

Útil para:

```text
large column sets
identifier edge cases
deep check expressions
composite keys
duplicate names
extension payloads
```

---

# 226. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Definition\Table
```

---

# 227. Proposed directory structure

```text
Schema/
└── Definition/
    ├── Table/
    │   ├── TableDefinition.php
    │   ├── TableDefinitionId.php
    │   ├── TableKind.php
    │   ├── TableDefinitionFactory.php
    │   ├── TableDefinitionAssembler.php
    │   │
    │   ├── Collection/
    │   │   ├── ColumnDefinitionSet.php
    │   │   ├── IndexDefinitionSet.php
    │   │   ├── UniqueConstraintDefinitionSet.php
    │   │   ├── ForeignKeyDefinitionSet.php
    │   │   └── CheckConstraintDefinitionSet.php
    │   │
    │   ├── Identifier/
    │   │   ├── TableIdentifier.php
    │   │   ├── QualifiedTableIdentifier.php
    │   │   ├── LogicalTableIdentity.php
    │   │   └── PhysicalTableIdentifier.php
    │   │
    │   ├── Constraint/
    │   │   ├── TableConstraintDefinition.php
    │   │   ├── PrimaryKeyDefinition.php
    │   │   ├── UniqueConstraintDefinition.php
    │   │   ├── ForeignKeyDefinition.php
    │   │   └── CheckConstraintDefinition.php
    │   │
    │   ├── Option/
    │   │   ├── TableOption.php
    │   │   ├── TableOptionSet.php
    │   │   ├── PortableTableOption.php
    │   │   └── ExtensionTableOption.php
    │   │
    │   ├── Metadata/
    │   │   ├── TableMetadata.php
    │   │   ├── SchemaObjectOrigin.php
    │   │   ├── SchemaObjectProvenance.php
    │   │   ├── DefinitionCompleteness.php
    │   │   ├── MetadataCertainty.php
    │   │   └── SensitivityMetadata.php
    │   │
    │   ├── Dependency/
    │   │   ├── SchemaDependency.php
    │   │   ├── SchemaDependencySet.php
    │   │   ├── ForeignTableDependency.php
    │   │   ├── SequenceDependency.php
    │   │   ├── TypeDependency.php
    │   │   └── ExtensionDependency.php
    │   │
    │   ├── Capability/
    │   │   └── SchemaCapabilityRequirementSet.php
    │   │
    │   ├── Comparison/
    │   │   ├── TableDefinitionComparator.php
    │   │   ├── TableComparisonResult.php
    │   │   └── SchemaComparisonProfile.php
    │   │
    │   ├── Fingerprint/
    │   │   ├── TableFingerprintGenerator.php
    │   │   ├── TableStructuralFingerprint.php
    │   │   ├── TableFingerprintProfile.php
    │   │   └── FingerprintAlgorithmVersion.php
    │   │
    │   ├── Serialization/
    │   │   ├── TableDefinitionSerializer.php
    │   │   ├── TableDefinitionDeserializer.php
    │   │   └── SchemaDefinitionFormatVersion.php
    │   │
    │   ├── Validation/
    │   │   ├── TableDefinitionValidator.php
    │   │   ├── TableValidationContext.php
    │   │   └── TableValidationResult.php
    │   │
    │   ├── Transformation/
    │   │   └── TableDefinitionTransformer.php
    │   │
    │   ├── Extension/
    │   │   ├── TableDefinitionExtension.php
    │   │   ├── ExtensionMetadataSet.php
    │   │   └── ExtensionStructuralImpact.php
    │   │
    │   ├── Budget/
    │   │   └── TableDefinitionBudget.php
    │   │
    │   └── Exception/
    │       ├── DatabaseTableDefinitionException.php
    │       ├── InvalidTableDefinitionException.php
    │       ├── InvalidTableIdentifierException.php
    │       ├── DuplicateColumnException.php
    │       ├── MissingPrimaryKeyColumnException.php
    │       ├── InvalidPrimaryKeyDefinitionException.php
    │       ├── InvalidUniqueConstraintException.php
    │       ├── InvalidIndexReferenceException.php
    │       ├── InvalidForeignKeyReferenceException.php
    │       ├── InvalidCheckConstraintException.php
    │       ├── ConflictingTableOptionException.php
    │       ├── UnknownTableOptionException.php
    │       ├── TableDefinitionSerializationException.php
    │       ├── TableDefinitionExtensionException.php
    │       ├── TableDefinitionBudgetExceededException.php
    │       └── TableDefinitionInvariantException.php
```

---

# 228. Dependency rules

Permitido:

```text
TableDefinition
→ ColumnDefinition
→ IndexDefinition
→ ConstraintDefinition
→ SchemaExpression
→ Database Type System
→ Schema Identifier abstractions
→ immutable metadata
→ immutable extension descriptors
```

No permitido:

```text
TableDefinition
→ PDO
TableDefinition
→ Connection
TableDefinition
→ DriverExecution
TableDefinition
→ QueryExecutor
TableDefinition
→ EntityManager
TableDefinition
→ MigrationRunner
TableDefinition
→ HTTP Request
TableDefinition
→ mutable TenantContext
```

---

# 229. Architectural invariants

## DB-TABLE-001

`TableDefinition` será una representación estructural de tabla.

## DB-TABLE-002

`TableDefinition` será inmutable.

## DB-TABLE-003

`TableDefinition` será distinta de `TableBlueprint`.

## DB-TABLE-004

`TableDefinition` será distinta de `CreateTableNode`.

## DB-TABLE-005

`TableDefinition` será distinta de una tabla introspectada con metadata operacional.

## DB-TABLE-006

`TableDefinition` será distinta de SQL.

## DB-TABLE-007

`TableDefinition` no generará SQL.

## DB-TABLE-008

`TableDefinition` no ejecutará SQL.

## DB-TABLE-009

`TableDefinition` no abrirá conexiones.

## DB-TABLE-010

`TableDefinition` no introspectará la base de datos.

## DB-TABLE-011

Toda tabla tendrá identifier tipado.

## DB-TABLE-012

Qualified identifiers serán estructurados.

## DB-TABLE-013

`TableIdentifier` será distinto de `TableDefinitionId`.

## DB-TABLE-014

Column definitions usarán colección tipada.

## DB-TABLE-015

Column order será preservado.

## DB-TABLE-016

Column identifiers locales serán únicos.

## DB-TABLE-017

No se asumirá casing universal.

## DB-TABLE-018

Portable identifier policy podrá ser más restrictiva.

## DB-TABLE-019

Una tabla tendrá como máximo una primary key.

## DB-TABLE-020

Primary key podrá ser compuesta.

## DB-TABLE-021

Primary key column order será preservado.

## DB-TABLE-022

Toda PK column deberá existir localmente.

## DB-TABLE-023

Unique constraint será distinta de unique index.

## DB-TABLE-024

Inline unique syntax podrá canonicalizarse a constraint.

## DB-TABLE-025

Indexes serán distintos de constraints.

## DB-TABLE-026

Index key order será preservado.

## DB-TABLE-027

Expression indexes serán representables.

## DB-TABLE-028

Foreign keys serán definiciones tipadas.

## DB-TABLE-029

Toda FK local column deberá existir.

## DB-TABLE-030

FK referenced table validation podrá requerir schema context.

## DB-TABLE-031

Composite FK arity deberá coincidir.

## DB-TABLE-032

Composite FK order será preservado.

## DB-TABLE-033

Self-referencing FK será válida estructuralmente.

## DB-TABLE-034

Check constraints serán tipadas.

## DB-TABLE-035

Check expressions preferirán SchemaExpression.

## DB-TABLE-036

Raw check será explícita.

## DB-TABLE-037

Table options usarán typed option set.

## DB-TABLE-038

No existirá generic mixed option bag como modelo principal.

## DB-TABLE-039

Platform options serán explícitas.

## DB-TABLE-040

Unknown options no se ignorarán.

## DB-TABLE-041

Metadata será distinta de structural semantics.

## DB-TABLE-042

Source location no alterará structural equality por defecto.

## DB-TABLE-043

Object identity será distinta de structural equality.

## DB-TABLE-044

Structural equality será distinta de physical compatibility.

## DB-TABLE-045

Physical compatibility será distinta de migration equivalence.

## DB-TABLE-046

Structural fingerprint no será SQL hash.

## DB-TABLE-047

Fingerprint será determinista.

## DB-TABLE-048

Fingerprint tendrá algorithm version.

## DB-TABLE-049

Fingerprint excluirá runtime IDs.

## DB-TABLE-050

Fingerprint excluirá timestamps de construcción.

## DB-TABLE-051

Serialization será determinista.

## DB-TABLE-052

Serialization tendrá format version.

## DB-TABLE-053

Serialization no dependerá exclusivamente de PHP serialize.

## DB-TABLE-054

Definition origin será explícito cuando sea necesario.

## DB-TABLE-055

Origin será distinto de certainty.

## DB-TABLE-056

UNKNOWN será distinto de ABSENT.

## DB-TABLE-057

Observed metadata no deberá fingir certeza.

## DB-TABLE-058

Operational observation metadata no contaminará TableDefinition core.

## DB-TABLE-059

TableDefinition será reusable para desired schema.

## DB-TABLE-060

TableDefinition será reusable dentro de observed schema wrappers.

## DB-TABLE-061

Canonicalization preservará semántica.

## DB-TABLE-062

Canonicalization no será platform compilation.

## DB-TABLE-063

Equivalent DSL forms podrán producir canonical definition común.

## DB-TABLE-064

Canonicalization no colapsará constraint e index incorrectamente.

## DB-TABLE-065

Column order no se canonicalizará arbitrariamente.

## DB-TABLE-066

Constraint declaration order podrá normalizarse cuando no sea semántico.

## DB-TABLE-067

Index key order será semántico.

## DB-TABLE-068

PK column order será semántico.

## DB-TABLE-069

FK column order será semántico.

## DB-TABLE-070

Raw structural provenance será preservado.

## DB-TABLE-071

Raw structural content no se volverá trusted implícitamente.

## DB-TABLE-072

TableDefinition no resolverá platform capabilities.

## DB-TABLE-073

TableDefinition podrá declarar capability requirements.

## DB-TABLE-074

Capability requirement será distinto de capability support.

## DB-TABLE-075

TableDefinition podrá declarar dependencies.

## DB-TABLE-076

Dependency declaration será distinta de dependency planning.

## DB-TABLE-077

TableDefinition no decidirá creation order.

## DB-TABLE-078

TableDefinition no decidirá drop order.

## DB-TABLE-079

TableDefinition no resolverá cycles.

## DB-TABLE-080

TableDefinition no realizará migration planning.

## DB-TABLE-081

TableDefinition representará state, no operation history.

## DB-TABLE-082

Rename intent no deberá inferirse únicamente de dos TableDefinitions.

## DB-TABLE-083

Rename operation pertenecerá a Schema AST.

## DB-TABLE-084

Schema Diff podrá comparar TableDefinitions.

## DB-TABLE-085

Schema Diff será distinto de TableDefinition.

## DB-TABLE-086

TableDefinition será diff-friendly.

## DB-TABLE-087

TableDefinition tendrá stable typed identities.

## DB-TABLE-088

TableDefinition será ORM-independent.

## DB-TABLE-089

TableDefinition no contendrá EntityManager state.

## DB-TABLE-090

TableDefinition no contendrá UnitOfWork state.

## DB-TABLE-091

TableDefinition no contendrá repository metadata core.

## DB-TABLE-092

ORM podrá referenciar TableDefinition/identifier downstream.

## DB-TABLE-093

TableDefinition será Query Builder-independent.

## DB-TABLE-094

Query Semantic Engine podrá consumir Schema Model basado en definitions.

## DB-TABLE-095

TableDefinition no dependerá de Query AST.

## DB-TABLE-096

TableDefinition será Compiler-independent.

## DB-TABLE-097

Compiler podrá consumir TableDefinition.

## DB-TABLE-098

TableDefinition será Driver-independent.

## DB-TABLE-099

TableDefinition no contendrá PDO handles.

## DB-TABLE-100

TableDefinition no contendrá connection handles.

## DB-TABLE-101

TableDefinition no contendrá transaction state.

## DB-TABLE-102

TableDefinition no contendrá request state.

## DB-TABLE-103

TableDefinition no contendrá mutable tenant context.

## DB-TABLE-104

Immutable definitions podrán compartirse en persistent runtime.

## DB-TABLE-105

Mutable builders no se almacenarán dentro de definitions.

## DB-TABLE-106

Definition local validation no requerirá DB.

## DB-TABLE-107

Cross-table validation requerirá explicit schema context.

## DB-TABLE-108

Platform validation será downstream.

## DB-TABLE-109

Migration safety validation será downstream.

## DB-TABLE-110

Execution feasibility será downstream.

## DB-TABLE-111

Extensions serán tipadas.

## DB-TABLE-112

Extension structural impact será declarado.

## DB-TABLE-113

Unknown structural extensions no se descartarán silenciosamente.

## DB-TABLE-114

Extension registry será frozen.

## DB-TABLE-115

No habrá last-wins extension semantics.

## DB-TABLE-116

Extension version podrá afectar serialization/fingerprint.

## DB-TABLE-117

Definition budgets serán explícitos.

## DB-TABLE-118

Budget overflow fallará.

## DB-TABLE-119

Budget overflow no truncará schema.

## DB-TABLE-120

Telemetry no alterará TableDefinition.

## DB-TABLE-121

Telemetry no deberá exponer raw metadata por defecto.

## DB-TABLE-122

Sensitivity metadata será distinta de authorization.

## DB-TABLE-123

Definition comparison será profile-aware.

## DB-TABLE-124

FULL comparison podrá considerar column order.

## DB-TABLE-125

PORTABLE comparison podrá abstraer platform-only differences.

## DB-TABLE-126

MIGRATION comparison será distinta de basic equality.

## DB-TABLE-127

Fingerprint profiles podrán ser distintos.

## DB-TABLE-128

Fingerprint generator podrá ser externo al value object.

## DB-TABLE-129

TableDefinition será serializable sin live resources.

## DB-TABLE-130

Serialized definitions no contendrán request IDs.

## DB-TABLE-131

Serialized definitions no contendrán connection handles.

## DB-TABLE-132

Serialized definitions no contendrán runtime tenant objects.

## DB-TABLE-133

Structural transformers producirán nuevas definitions.

## DB-TABLE-134

Transformers no mutarán input.

## DB-TABLE-135

Transformers no ejecutarán queries.

## DB-TABLE-136

Transformers serán deterministas.

## DB-TABLE-137

Definition transformation será distinta de migration.

## DB-TABLE-138

Round-trip compatibility será verificable.

## DB-TABLE-139

Round-trip no requerirá byte equality.

## DB-TABLE-140

Platform-generated metadata podrá normalizarse.

## DB-TABLE-141

Platform-generated names tendrán provenance.

## DB-TABLE-142

Inherited values serán distinguibles de explicit values cuando sea necesario.

## DB-TABLE-143

Unspecified será distinto de platform default cuando sea relevante.

## DB-TABLE-144

TableDefinition evitará duplicated sources of truth.

## DB-TABLE-145

Column position preferirá collection ordering cuando sea suficiente.

## DB-TABLE-146

Definition diagnostics podrán conservar source mapping.

## DB-TABLE-147

TableDefinition será testeable sin DB.

## DB-TABLE-148

Structural equality será testeable determinísticamente.

## DB-TABLE-149

Serialization round-trip preservará structural semantics.

## DB-TABLE-150

`TableDefinition` será la unidad estructural canónica de tabla dentro del Schema Model de VoltStack.

---

# 230. Anti-patterns

## 230.1 Mutable TableDefinition

Incorrecto:

```php
$table->addColumn($column);
```

después de publicar la definición.

Correcto:

```text
TableBlueprint
→ TableDefinition
```

---

## 230.2 SQL inside definition

Incorrecto:

```php
$table->createSql = 'CREATE TABLE ...';
```

---

## 230.3 PDO inside definition

Incorrecto:

```php
$table->connection = $pdo;
```

---

## 230.4 Mixing entity metadata

Incorrecto:

```php
$table->entityClass = User::class;
$table->repository = UserRepository::class;
```

en el núcleo estructural.

---

## 230.5 Unique as boolean

Demasiado pobre:

```php
$column->unique = true;
```

cuando realmente existe una:

```text
UniqueConstraintDefinition
```

---

## 230.6 Foreign keys as strings

Incorrecto:

```php
'posts.user_id -> users.id'
```

Correcto:

```text
ForeignKeyDefinition
```

---

## 230.7 Platform SQL types

Incorrecto:

```text
column type = "BIGINT UNSIGNED"
```

como tipo semántico principal.

---

## 230.8 Unknown = absent

Incorrecto:

```text
driver did not report check constraints
→ table has no check constraints
```

---

## 230.9 Constraint = index

Incorrecto:

```text
UniqueConstraint
=
UniqueIndex
```

---

## 230.10 State = operation

Incorrecto:

```text
TableDefinition before/after
```

como reemplazo universal de:

```text
RenameColumnNode
```

---

# 231. Ejemplo completo

```php
$table = new TableDefinition(
    identifier: QualifiedTableIdentifier::table('users'),

    kind: TableKind::REGULAR,

    columns: ColumnDefinitionSet::of(
        ColumnDefinition::bigInteger('id')
            ->identity(),

        ColumnDefinition::string(
            'email',
            length: 320
        ),

        ColumnDefinition::string(
            'name',
            length: 150
        ),

        ColumnDefinition::boolean('active')
            ->default(true),

        ColumnDefinition::timestamp('created_at'),

        ColumnDefinition::timestamp('updated_at'),
    ),

    primaryKey: PrimaryKeyDefinition::columns([
        'id',
    ]),

    uniqueConstraints: UniqueConstraintDefinitionSet::of(
        UniqueConstraintDefinition::columns([
            'email',
        ])
    ),

    indexes: IndexDefinitionSet::empty(),

    foreignKeys: ForeignKeyDefinitionSet::empty(),

    checks: CheckConstraintDefinitionSet::empty(),

    options: TableOptionSet::empty(),

    metadata: TableMetadata::declared(),

    extensions: ExtensionMetadataSet::empty(),
);
```

Conceptualmente:

```text
TableDefinition(users)
│
├── kind: REGULAR
│
├── columns
│   ├── id
│   │   ├── BIG_INTEGER
│   │   ├── NOT NULL
│   │   └── IDENTITY
│   │
│   ├── email
│   │   ├── STRING(320)
│   │   └── NOT NULL
│   │
│   ├── name
│   │   ├── STRING(150)
│   │   └── NOT NULL
│   │
│   ├── active
│   │   ├── BOOLEAN
│   │   ├── NOT NULL
│   │   └── DEFAULT true
│   │
│   ├── created_at
│   └── updated_at
│
├── primary key
│   └── id
│
├── unique
│   └── email
│
├── indexes
│   └── ∅
│
├── foreign keys
│   └── ∅
│
├── checks
│   └── ∅
│
└── options
    └── ∅
```

---

# 232. Ejemplo con relaciones

```text
TableDefinition(posts)
│
├── Columns
│   ├── id
│   ├── user_id
│   ├── title
│   └── body
│
├── PrimaryKey
│   └── id
│
└── ForeignKeys
    └── posts_user_fk
        ├── local
        │   └── user_id
        ├── target table
        │   └── users
        ├── target columns
        │   └── id
        └── delete action
            └── CASCADE
```

Dependencia derivada:

```text
posts
  │
  └──────────────► users
```

La definición declara esa dependencia.

No decide el orden de ejecución.

---

# 233. Ejemplo con clave compuesta

```text
TableDefinition(tenant_users)
│
├── Columns
│   ├── tenant_id
│   ├── user_id
│   └── role
│
└── PrimaryKey
    ├── tenant_id
    └── user_id
```

Debe preservarse:

```text
PK(tenant_id, user_id)
```

exactamente en ese orden.

---

# 234. Ejemplo de estado vs operación

Estado inicial:

```text
TableDefinition(users)
└── name
```

Estado final:

```text
TableDefinition(users)
└── full_name
```

No es posible concluir únicamente de esos estados:

```text
Rename(name → full_name)
```

Podría haber sido:

```text
Drop(name)
+
Add(full_name)
```

Por eso:

```text
TableDefinition
=
state representation
```

mientras:

```text
Schema AST
=
operation representation
```

---

# 235. Ejemplo de Desired vs Observed

```text
DesiredTable
│
└── TableDefinition(users)
    └── email STRING(320)
```

Base de datos:

```text
ObservedTable
├── TableDefinition(users)
│   └── email VARCHAR(320)
│
├── Platform
│   └── PostgreSQL
│
├── Certainty
│   └── DRIVER_REPORTED
│
└── Observation Metadata
    └── ...
```

La compatibilidad se determina posteriormente:

```text
Desired
    │
    ▼
Schema Compatibility
    ▲
    │
Observed
```

---

# 236. Master formula

```text
Table Definition System
=
Table Identity
+
Ordered Column Definitions
+
Primary Key Definition
+
Unique Constraint Definitions
+
Index Definitions
+
Foreign Key Definitions
+
Check Constraint Definitions
+
Typed Table Options
+
Structural Metadata
+
Extension Metadata
+
Dependency Information
+
Capability Requirements
+
Canonicalization
+
Validation
+
Comparison
+
Fingerprinting
+
Deterministic Serialization
+
Persistent Runtime Safety
```

---

# 237. Correctness formula

```text
CorrectTableDefinition
=
Immutable
∧
Typed
∧
Canonical
∧
Deterministic
∧
LocallyValid
∧
IntentPreserving
∧
PlatformIndependent
∧
DriverIndependent
∧
DiffFriendly
∧
Serializable
∧
ExtensionSafe
∧
RuntimeSafe
```

---

# 238. Structural validity formula

Para una tabla `T`:

```text
Valid(T)
=
UniqueColumnIdentifiers(T)
∧
PKColumns(T) ⊆ Columns(T)
∧
LocalFKColumns(T) ⊆ Columns(T)
∧
IndexColumnReferences(T) ⊆ Columns(T)
∧
CheckLocalReferences(T) ⊆ Columns(T)
∧
ConsistentOptions(T)
∧
ValidExtensions(T)
```

La validación global añade posteriormente:

```text
ReferencedTablesExist
∧
ReferencedColumnsExist
∧
ReferenceTypesCompatible
∧
PlatformCapabilitiesSatisfied
```

---

# 239. State invariant

```text
TableDefinition
=
Structural State
```

No:

```text
TableDefinition
=
Migration History
```

---

# 240. Final architectural rule

> **`TableDefinition` describe qué estructura tiene o debe tener una tabla; nunca cómo fue construida, cómo será migrada ni cómo será representada en SQL.**

En forma compacta:

```text
TableBlueprint
     │
     │ constructs
     ▼
TableDefinition
     │
     │ participates in
     ▼
Schema Model / Schema AST
     │
     │ analyzed by
     ▼
Planner
     │
     │ represented by
     ▼
Compiler
     │
     │ executed by
     ▼
Execution Engine
```

---

# 241. Resultado arquitectónico

Con este sistema VoltStack obtiene una unidad estructural estable que podrá convertirse en el punto común entre:

```text
Schema Builder
      │
      ▼
TableDefinition
      ▲
      │
Schema Introspection
```

y posteriormente:

```text
Observed Schema
       │
       ▼
TableDefinition
       │
       │ compare
       ▼
TableDefinition
       ▲
       │
Desired Schema
```

permitiendo construir sobre ella:

```text
Schema Diff
Migration Planning
Schema Validation
Schema Compilation
Query Semantic Resolution
Testing
Diagnostics
Metadata Caching
```

sin introducir dependencias inversas.

---

# 242. Siguiente documento

```text
92_DATABASE_COLUMN_DEFINITION_SYSTEM.md
```

El siguiente documento deberá formalizar `ColumnDefinition`, incluyendo:

```text
ColumnDefinition
├── ColumnIdentifier
├── DatabaseType
├── Nullability
├── DefaultDefinition
├── GenerationStrategy
├── GeneratedExpression
├── Collation
├── CharacterSet
├── ColumnOptions
├── Metadata
├── CapabilityRequirements
└── ExtensionMetadata
```

y establecerá especialmente las diferencias:

```text
ColumnDefinition
≠
ColumnBlueprint
≠
ColumnAlteration
≠
ObservedColumn
≠
ORM Property
≠
SQL Column Fragment
```

Principio siguiente:

> **Una columna es una definición estructural tipada; no una cadena de SQL ni una propiedad ORM.**