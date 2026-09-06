# 89_DATABASE_SCHEMA_AST_SYSTEM.md

# VoltStack Quantum Database
## Database Schema AST System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 89 — Database Schema AST System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema AST System` define la representación estructurada, tipada e inmutable de las operaciones que expresan intención de cambio sobre el esquema de una base de datos.

El sistema deberá representar operaciones como:

```text
CREATE TABLE
ALTER TABLE
DROP TABLE
ADD COLUMN
ALTER COLUMN
DROP COLUMN
CREATE INDEX
DROP INDEX
ADD CONSTRAINT
DROP CONSTRAINT
ADD FOREIGN KEY
DROP FOREIGN KEY
RENAME TABLE
RENAME COLUMN
CREATE SEQUENCE
DROP SEQUENCE
```

sin convertirlas todavía en SQL ni ejecutarlas.

La regla fundamental será:

```text
Schema Model
=
Structural State

Schema AST
=
Structural Transformation Intent
```

Por tanto:

```text
Schema AST
≠ SQL
≠ DDL String
≠ Migration
≠ Schema Diff
≠ Execution Plan
≠ Driver Command
```

---

# 2. Posición arquitectónica

El Schema AST ocupa la frontera entre las APIs que expresan cambios estructurales y los subsistemas que validan, planifican y compilan dichos cambios.

```text
Schema Builder
      │
      ▼
Schema Definitions
      │
      ▼
Schema AST
      │
      ├── Validation
      ├── Normalization
      ├── Dependency Analysis
      └── Capability Analysis
      │
      ▼
Schema Change Planning
      │
      ▼
Schema Compiler
      │
      ▼
Compiled Database Commands
      │
      ▼
Execution Engine
```

También podrá ser producido por:

```text
SchemaDiff
    ↓
Schema AST / Change Intent
```

cuando posteriormente se implemente el sistema de diferencias.

---

# 3. Responsabilidad

Schema AST responde:

> ¿Qué transformación estructural se solicita?

No responde:

> ¿Qué SQL exacto debe ejecutarse?

Tampoco responde:

> ¿En qué orden físico final deben ejecutarse todas las operaciones?

Ni:

> ¿La operación debe ejecutarse dentro de una migration?

Ni:

> ¿Cómo debe realizarse el rollback?

---

# 4. Ejemplo conceptual

Una API de usuario podría expresar:

```php
Schema::create('users', function (Table $table) {
    $table->id();
    $table->string('email')->unique();
});
```

El resultado conceptual no deberá ser inmediatamente:

```sql
CREATE TABLE users (...);
```

sino:

```text
CreateTableNode
└── TableDefinition
    ├── id
    └── email
```

Posteriormente:

```text
Schema AST
↓
Validation
↓
Planning
↓
Compilation
↓
SQL
```

---

# 5. AST ≠ SQL AST

VoltStack ya distingue representaciones en diferentes niveles.

Por tanto:

```text
Schema AST
≠ SQL Emission AST
```

El Schema AST expresa:

```text
create table users
```

semánticamente.

El SQL emission model representa:

```text
CREATE TABLE "users" (...)
```

según una plataforma concreta.

---

# 6. AST ≠ Schema Model

Ejemplo de Schema Model:

```text
Table users exists
├── id
├── email
└── PK(id)
```

Ejemplo de Schema AST:

```text
CreateTable(users)
```

Formalmente:

```text
SchemaModel = State
SchemaAST   = TransformationIntent
```

---

# 7. AST ≠ Schema Diff

`SchemaDiff` representa diferencias detectadas entre estados.

```text
CurrentSchema
      +
TargetSchema
      ↓
SchemaDiff
```

Schema AST representa operaciones estructurales.

```text
SchemaDiff
      ↓
Change Translation
      ↓
Schema AST
```

Por tanto:

```text
SchemaDiff
≠
SchemaAST
```

---

# 8. AST ≠ Migration

Migration agrega conceptos como:

```text
version
batch
history
up/down
rollback
execution status
deployment lifecycle
```

Schema AST no deberá conocerlos.

---

# 9. AST ≠ Change Plan

Una lista declarada:

```text
AddColumn
AddForeignKey
CreateIndex
```

no necesariamente representa el orden físico correcto.

El planner podrá producir:

```text
AddColumn
CreateIndex
AddForeignKey
```

si existen dependencias.

Por tanto:

```text
Declared Operation Order
≠
Physical Execution Order
```

---

# 10. Principio de intención

El AST deberá conservar la intención estructural con suficiente fidelidad para que etapas posteriores puedan decidir cómo implementarla.

Ejemplo:

```text
RenameColumn(users.name → users.full_name)
```

no deberá convertirse prematuramente en:

```text
DROP name
ADD full_name
```

porque eso pierde la intención de rename.

---

# 11. Semantic intent preservation

Regla:

```text
AST must preserve user structural intent
until a planning stage intentionally lowers it.
```

Esto es crítico para:

```text
data preservation
zero-downtime migrations
diagnostics
rollback analysis
platform compatibility
schema diff
telemetry
```

---

# 12. Root abstraction

Se propone:

```php
interface SchemaNode
{
    public function id(): SchemaNodeId;

    public function kind(): SchemaNodeKind;

    public function metadata(): SchemaNodeMetadata;
}
```

---

# 13. SchemaNodeId

Debe distinguirse:

```text
SchemaNodeId
≠
SchemaObjectId
≠
MigrationId
≠
ExecutionUnitId
≠
CompiledCommandId
```

`SchemaNodeId` identifica un nodo dentro del artefacto AST.

---

# 14. SchemaNodeKind

Ejemplo inicial:

```php
enum SchemaNodeKind: string
{
    case CREATE_SCHEMA = 'create_schema';
    case DROP_SCHEMA = 'drop_schema';

    case CREATE_TABLE = 'create_table';
    case ALTER_TABLE = 'alter_table';
    case RENAME_TABLE = 'rename_table';
    case DROP_TABLE = 'drop_table';

    case ADD_COLUMN = 'add_column';
    case ALTER_COLUMN = 'alter_column';
    case RENAME_COLUMN = 'rename_column';
    case DROP_COLUMN = 'drop_column';

    case CREATE_INDEX = 'create_index';
    case DROP_INDEX = 'drop_index';

    case ADD_CONSTRAINT = 'add_constraint';
    case DROP_CONSTRAINT = 'drop_constraint';

    case ADD_FOREIGN_KEY = 'add_foreign_key';
    case DROP_FOREIGN_KEY = 'drop_foreign_key';

    case CREATE_SEQUENCE = 'create_sequence';
    case ALTER_SEQUENCE = 'alter_sequence';
    case DROP_SEQUENCE = 'drop_sequence';

    case CREATE_VIEW = 'create_view';
    case DROP_VIEW = 'drop_view';

    case EXTENSION = 'extension';
}
```

---

# 15. SchemaAst

El artefacto raíz podrá ser:

```php
final readonly class SchemaAst
{
    public function __construct(
        public SchemaAstId $id,
        public SchemaNodeCollection $nodes,
        public SchemaAstMetadata $metadata,
        public SchemaAstVersion $version,
    ) {}
}
```

---

# 16. AST artifact identity

Debe distinguirse:

```text
SchemaAstId
≠
SchemaFingerprint
```

Dos AST diferentes pueden expresar una transformación estructural equivalente.

---

# 17. Immutability

Una vez publicado:

```text
SchemaAst
```

será inmutable.

No:

```php
$ast->nodes[] = $node;
```

Preferible:

```text
AST₀
↓
Transformation
↓
AST₁
```

---

# 18. SchemaNodeCollection

```php
final readonly class SchemaNodeCollection implements IteratorAggregate, Countable
{
}
```

La colección deberá:

```text
preserve declared ordering
support deterministic traversal
support lookup by SchemaNodeId
reject duplicate node IDs
```

---

# 19. Declared order

El AST puede conservar el orden declarado.

Pero:

```text
DeclaredOrder
≠
GuaranteedExecutionOrder
```

El planner podrá reorganizar operaciones cuando sea legal.

---

# 20. Ordering barriers

Algunas operaciones podrán declarar:

```text
ordering constraints
```

que el planner deberá respetar.

Ejemplo:

```text
CreateTable(users)
before
AddForeignKey(posts.user_id → users.id)
```

---

# 21. Structural references

Los nodos deberán usar referencias estructuradas.

No:

```php
'public.users.email'
```

como representación principal.

Preferible:

```php
new ColumnSchemaReference(
    table: new TableSchemaReference(...),
    column: new ColumnIdentifier('email'),
);
```

---

# 22. SchemaNode target

Muchos nodos tendrán:

```text
target
```

Ejemplo:

```text
DropTableNode
→ TableReference
```

---

# 23. CreateTableNode

```php
final readonly class CreateTableNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableDefinition $table,
        public CreationBehavior $behavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 24. TableDefinition

El `CreateTableNode` podrá reutilizar conceptos estructurales del Schema Model.

Pero debe evitarse mezclar:

```text
existing schema object identity
```

con:

```text
new object definition
```

Por ello puede existir:

```text
TableDefinition
```

como value object declarativo.

---

# 25. Definition ≠ existing object

```text
TableDefinition
≠
Table
```

aunque compartan muchos componentes.

`Table` representa un objeto estructural perteneciente a un Schema Model.

`TableDefinition` representa la definición deseada de uno.

---

# 26. Shared structural values

Sí podrán compartirse:

```text
Identifier
DatabaseType
Nullability
DefaultValue
IndexDefinition
ConstraintDefinition
ColumnDefinition
```

cuando no introduzcan ciclos arquitectónicos.

---

# 27. Create behavior

No deberá expresarse mediante booleanos ambiguos como:

```php
$ifNotExists = true;
```

si el modelo necesita crecer.

Se propone:

```php
enum CreationBehavior
{
    case REQUIRE_ABSENT;
    case IGNORE_IF_EXISTS;
}
```

---

# 28. REQUIRE_ABSENT

Significa:

```text
existing target
→ structural conflict
```

---

# 29. IGNORE_IF_EXISTS

Representa intención explícita equivalente a:

```text
IF NOT EXISTS
```

cuando la plataforma/capability pueda soportarla o planificarla correctamente.

---

# 30. IGNORE_IF_EXISTS warning

No deberá usarse como sustituto universal de idempotencia de migrations.

---

# 31. DropTableNode

```php
final readonly class DropTableNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableSchemaReference $table,
        public DropBehavior $behavior,
        public DependencyDropBehavior $dependencyBehavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 32. DropBehavior

```php
enum DropBehavior
{
    case REQUIRE_PRESENT;
    case IGNORE_IF_MISSING;
}
```

---

# 33. DependencyDropBehavior

Debe representar conceptos como:

```text
RESTRICT
CASCADE
PLATFORM_DEFAULT
```

sin introducir SQL directamente.

---

# 34. CASCADE is dangerous

Una operación:

```text
DROP TABLE ... CASCADE
```

puede afectar objetos no declarados explícitamente.

Por tanto deberá conservarse como intención peligrosa visible.

---

# 35. No implicit cascade

Nunca:

```text
dependency exists
→ automatically cascade
```

sin intención/política explícita.

---

# 36. RenameTableNode

```php
final readonly class RenameTableNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableSchemaReference $from,
        public TableIdentifier $to,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 37. Rename semantics

```text
RenameTable(A, B)
≠
DropTable(A) + CreateTable(B)
```

---

# 38. Identity continuity

Un rename expresa potencial continuidad de identidad estructural.

Esto será importante para:

```text
data preservation
dependency updates
migration safety
diff inference
zero downtime
```

---

# 39. AlterTableNode

Puede funcionar como contenedor de alteraciones relacionadas:

```php
final readonly class AlterTableNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableSchemaReference $table,
        public TableAlterationCollection $alterations,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 40. AlterTableNode vs individual nodes

VoltStack podrá soportar ambos niveles:

```text
AlterTableNode
└── TableAlteration
```

y nodos top-level normalizados:

```text
AddColumnNode
AlterColumnNode
DropColumnNode
```

---

# 41. Canonical policy

La representación interna canónica deberá elegir una forma predominante.

Recomendación:

```text
Schema AST
├── top-level structural operations
└── typed child alterations where atomic table grouping matters
```

---

# 42. TableAlteration

```php
interface TableAlteration
{
    public function id(): TableAlterationId;

    public function kind(): TableAlterationKind;
}
```

---

# 43. Alteration kinds

```text
ADD_COLUMN
ALTER_COLUMN
RENAME_COLUMN
DROP_COLUMN
ADD_INDEX
DROP_INDEX
ADD_CONSTRAINT
DROP_CONSTRAINT
ADD_FOREIGN_KEY
DROP_FOREIGN_KEY
TABLE_OPTION_CHANGE
```

---

# 44. AddColumnNode

```php
final readonly class AddColumnNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableSchemaReference $table,
        public ColumnDefinition $column,
        public ?ColumnPlacement $placement,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 45. Column placement

Algunas plataformas soportan:

```text
FIRST
AFTER column
```

Pero:

```text
ColumnPlacement
```

deberá ser capability-aware.

---

# 46. Portable default

El portable core no deberá asumir que el posicionamiento físico de columnas siempre puede modificarse.

---

# 47. AlterColumnNode

Alterar una columna requiere especial cuidado.

No deberá utilizarse:

```php
new AlterColumnNode(
    type: 'VARCHAR(500)',
    nullable: false,
    default: null,
);
```

porque `null` puede significar demasiadas cosas.

---

# 48. ColumnChangeSet

Se propone:

```php
final readonly class ColumnChangeSet
{
    public function __construct(
        public OptionalChange $type,
        public OptionalChange $nullability,
        public OptionalChange $default,
        public OptionalChange $generation,
        public OptionalChange $collation,
        public OptionalChange $options,
    ) {}
}
```

---

# 49. Unchanged ≠ set null

Debe distinguirse:

```text
UNCHANGED
SET(NULL)
REMOVE
```

---

# 50. Change<T>

Puede modelarse:

```text
Change<T>
├── Unchanged
├── SetValue<T>
└── RemoveValue
```

---

# 51. Example

Para eliminar default:

```text
default = RemoveValue
```

Para dejarlo intacto:

```text
default = Unchanged
```

Para establecer:

```text
DEFAULT NULL
```

se utilizará el value object apropiado.

---

# 52. RenameColumnNode

```php
final readonly class RenameColumnNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public ColumnSchemaReference $from,
        public ColumnIdentifier $to,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 53. Rename column semantics

```text
RenameColumn
≠
DropColumn + AddColumn
```

---

# 54. DropColumnNode

```php
final readonly class DropColumnNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public ColumnSchemaReference $column,
        public DropBehavior $behavior,
        public DependencyDropBehavior $dependencyBehavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 55. Destructive operation classification

`DropColumnNode` deberá ser identificable como:

```text
potentially destructive
```

sin que el AST tenga que decidir si la operación está autorizada.

---

# 56. Safety metadata

Puede adjuntarse:

```text
SchemaOperationRiskHint
```

pero:

```text
risk hint
≠
safety decision
```

La seguridad de migrations será tratada posteriormente.

---

# 57. CreateIndexNode

```php
final readonly class CreateIndexNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableSchemaReference $table,
        public IndexDefinition $index,
        public CreationBehavior $behavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 58. IndexDefinition

Debe preservar:

```text
name
keys
expressions
order
uniqueness
predicate
included columns
method
options
```

sin SQL.

---

# 59. Concurrent index creation

PostgreSQL puede ofrecer conceptos como:

```text
CREATE INDEX CONCURRENTLY
```

Esto no deberá convertirse en un boolean universal de `IndexDefinition`.

Puede expresarse mediante:

```text
SchemaExecutionPreference
```

o una extensión/capability específica.

---

# 60. Structural intent vs operational preference

Distinción:

```text
CreateIndex
=
structural intent
```

```text
Concurrent creation
=
operational strategy/preference
```

---

# 61. DropIndexNode

```php
final readonly class DropIndexNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public IndexSchemaReference $index,
        public DropBehavior $behavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 62. AddConstraintNode

```php
final readonly class AddConstraintNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableSchemaReference $table,
        public ConstraintDefinition $constraint,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 63. ConstraintDefinition

Puede incluir:

```text
PrimaryKeyDefinition
UniqueConstraintDefinition
CheckConstraintDefinition
ForeignKeyDefinition
ExtensionConstraintDefinition
```

---

# 64. AddForeignKeyNode

Aunque Foreign Key sea una constraint, puede tener un nodo especializado:

```php
final readonly class AddForeignKeyNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TableSchemaReference $table,
        public ForeignKeyDefinition $foreignKey,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 65. Why specialized FK node

Porque las FKs poseen semántica adicional:

```text
cross-table dependency
referential actions
deferrability
validation
creation ordering
cycle handling
```

---

# 66. DropForeignKeyNode

```php
final readonly class DropForeignKeyNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public ForeignKeySchemaReference $foreignKey,
        public DropBehavior $behavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 67. Constraint identity

No todas las plataformas manejan nombres de constraints igual.

Por tanto una referencia podrá incluir:

```text
logical identifier
table scope
constraint kind
native metadata
```

cuando sea necesario.

---

# 68. CreateSchemaNode

```php
final readonly class CreateSchemaNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public SchemaIdentifier $schema,
        public CreationBehavior $behavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 69. Schema namespace support

`CREATE SCHEMA` será capability-aware.

No todas las plataformas poseen la misma semántica de schema namespace.

---

# 70. DropSchemaNode

```php
final readonly class DropSchemaNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public SchemaSchemaReference $schema,
        public DropBehavior $behavior,
        public DependencyDropBehavior $dependencyBehavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 71. Sequence nodes

Se propone:

```text
CreateSequenceNode
AlterSequenceNode
DropSequenceNode
```

---

# 72. CreateSequenceNode

```php
final readonly class CreateSequenceNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public SequenceDefinition $sequence,
        public CreationBehavior $behavior,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 73. Sequence capability

Una plataforma sin secuencias nativas no deberá recibir una emulación silenciosa desde el AST.

---

# 74. View nodes

Se dejará espacio para:

```text
CreateViewNode
ReplaceViewNode
DropViewNode
CreateMaterializedViewNode
RefreshMaterializedViewNode
DropMaterializedViewNode
```

según las capacidades que se adopten.

---

# 75. Refresh materialized view

Debe observarse que:

```text
REFRESH MATERIALIZED VIEW
```

no cambia necesariamente la estructura del schema de la misma forma que `CREATE`.

Puede pertenecer a una categoría operacional especializada.

---

# 76. AST classification

Será útil clasificar nodos.

```php
enum SchemaOperationCategory
{
    case CREATE;
    case ALTER;
    case RENAME;
    case DROP;
    case VALIDATE;
    case EXTENSION;
}
```

---

# 77. Risk classification

Separadamente:

```php
enum SchemaOperationRiskClass
{
    case NON_DESTRUCTIVE;
    case POTENTIALLY_DESTRUCTIVE;
    case DESTRUCTIVE;
    case PLATFORM_DEPENDENT;
    case UNKNOWN;
}
```

---

# 78. Risk is metadata

La clasificación de riesgo:

```text
does not authorize execution
```

Sólo informa a subsistemas posteriores.

---

# 79. SchemaNodeMetadata

```php
final readonly class SchemaNodeMetadata
{
    public function __construct(
        public ?SourceLocation $sourceLocation,
        public SchemaNodeAnnotationSet $annotations,
        public SchemaOperationRiskClass $riskClass,
        public SchemaExtensionMetadataSet $extensions,
    ) {}
}
```

---

# 80. Source location

Ejemplo:

```text
database/migrations/2026_09_05_create_users.php:42
```

puede conservarse para diagnóstico.

---

# 81. Source location does not affect semantics

```text
SameOperation
from different source files
```

puede seguir siendo estructuralmente equivalente.

---

# 82. Annotations

Annotations podrán almacenar información tipada como:

```text
developer label
generated origin
diff origin
diagnostic correlation
migration source
```

sin convertirse en lógica de ejecución.

---

# 83. AST provenance

Un nodo puede provenir de:

```text
BUILDER
DIFF
MIGRATION
CODE_GENERATOR
EXTENSION
INTERNAL_NORMALIZATION
```

---

# 84. SchemaNodeOrigin

```php
enum SchemaNodeOrigin
{
    case BUILDER;
    case DIFF;
    case MIGRATION;
    case GENERATED;
    case EXTENSION;
    case INTERNAL;
}
```

---

# 85. Origin ≠ semantics

Dos operaciones equivalentes no dejan de ser equivalentes por tener distinto origin.

---

# 86. Schema expressions

Algunas operaciones requieren expresiones:

```text
DEFAULT expression
CHECK predicate
generated column expression
partial index predicate
expression index
```

---

# 87. SchemaExpression

Debe existir un modelo estructurado:

```php
interface SchemaExpression
{
    public function kind(): SchemaExpressionKind;
}
```

---

# 88. Expression nodes

Ejemplo:

```text
SchemaExpression
├── LiteralExpression
├── ColumnReferenceExpression
├── FunctionExpression
├── UnaryExpression
├── BinaryExpression
├── CastExpression
├── CurrentTimestampExpression
├── SequenceValueExpression
└── ExtensionExpression
```

---

# 89. SchemaExpression ≠ QueryExpression

Aunque puedan compartir conceptos:

```text
SchemaExpression
≠
QueryExpression
```

por diferencias de contexto, capabilities y ciclo de vida.

---

# 90. Shared expression primitives

Podrán extraerse primitives comunes si la arquitectura demuestra que son realmente compartibles.

No se deberá forzar reutilización artificial.

---

# 91. Raw schema expression

Se permitirá un escape hatch explícito:

```php
final readonly class RawSchemaExpression implements SchemaExpression
{
}
```

---

# 92. Raw expression boundary

`RawSchemaExpression` deberá ser:

```text
explicit
trusted
diagnostic-visible
non-portable by default
security-sensitive
```

---

# 93. No automatic raw fallback

Nunca:

```text
unknown expression
→ stringify
→ RawSchemaExpression
```

---

# 94. Identifier safety

Identifiers nunca deberán mezclarse con runtime values.

```text
Identifier
≠
Parameter
```

---

# 95. Schema AST parameters

Normalmente DDL estructural utiliza definiciones conocidas en compile time.

Sin embargo, cualquier sistema futuro de parameters deberá distinguir:

```text
schema structural value
runtime execution value
```

---

# 96. No runtime parameter interpolation

Nunca:

```text
table name = runtime SQL binding parameter
```

Los identificadores son estructura SQL, no value bindings.

---

# 97. AST validation layers

La validación deberá dividirse:

```text
Local Node Validation
        ↓
AST Structural Validation
        ↓
Reference Validation
        ↓
Semantic Validation
        ↓
Capability Validation
        ↓
Planning Validation
```

---

# 98. Local validation

Ejemplos:

```text
CreateTable must have name
AddColumn must have column definition
RenameColumn old != new
Index must have at least one key
Foreign key arity must match
```

---

# 99. AST structural validation

Ejemplos:

```text
duplicate SchemaNodeId
invalid parent-child relation
malformed reference
unsupported extension node registration
```

---

# 100. Reference validation

Ejemplo:

```text
AddForeignKey
→ referenced table
```

debe tener una referencia válida.

---

# 101. Semantic validation

Puede detectar:

```text
duplicate column addition
drop nonexistent column
rename collision
incompatible constraint definition
```

cuando exista un Schema Model base.

---

# 102. Base schema

La validación puede recibir:

```text
CurrentSchema
```

como contexto explícito.

Nunca deberá consultarlo ocultamente desde una conexión.

---

# 103. Validation context

```php
final readonly class SchemaAstValidationContext
{
    public function __construct(
        public ?DatabaseSchema $baseSchema,
        public PlatformCapabilitySnapshot $capabilities,
        public SchemaValidationProfile $profile,
        public SchemaExtensionRegistrySnapshot $extensions,
    ) {}
}
```

---

# 104. No hidden DB I/O

Validation:

```text
MUST NOT
```

abrir una conexión para averiguar si una tabla existe.

---

# 105. Capability validation

Ejemplo:

```text
CreateSequenceNode
+
SQLite capability snapshot without sequences
```

deberá producir un resultado explícito.

---

# 106. Unsupported ≠ automatically emulated

El validator/compiler no deberá convertir automáticamente una operación no soportada en otra estrategia.

Eso corresponde al planning cuando exista una equivalencia correcta.

---

# 107. AST normalization

Podrá existir:

```php
interface SchemaAstNormalizer
{
    public function normalize(
        SchemaAst $ast,
        SchemaAstNormalizationContext $context,
    ): NormalizedSchemaAst;
}
```

---

# 108. Normalization purpose

Puede:

```text
canonicalize equivalent node forms
normalize identifiers
normalize option ordering
expand syntactic sugar
normalize annotations
```

sin cambiar intención.

---

# 109. Normalization must preserve intent

```text
Intent(Normalize(AST))
=
Intent(AST)
```

---

# 110. Syntactic sugar

Ejemplo de Builder:

```php
$table->id();
```

puede convertirse en:

```text
ColumnDefinition
├── type = integer/bigint identity
├── not null
└── primary key semantics
```

antes o durante normalización.

---

# 111. Sugar does not belong to compiler

El compiler no debería interpretar:

```text
id()
timestamps()
softDeletes()
```

Estos son conceptos de developer API/Builder.

---

# 112. NormalizedSchemaAst

Puede existir como artefacto separado:

```php
final readonly class NormalizedSchemaAst
{
}
```

---

# 113. SchemaAst ≠ NormalizedSchemaAst

Esto permite:

```text
Original intent artifact
+
Canonical downstream artifact
```

---

# 114. Dependency analysis

Los nodos pueden tener dependencias.

Ejemplo:

```text
CreateTable(users)
CreateTable(posts)
AddForeignKey(posts.user_id → users.id)
```

Grafo:

```text
CreateTable(users) ───────────┐
                              ▼
CreateTable(posts) ──▶ AddForeignKey
```

---

# 115. SchemaNodeDependencyGraph

```php
final readonly class SchemaNodeDependencyGraph
{
}
```

---

# 116. Dependency kinds

```php
enum SchemaNodeDependencyKind
{
    case STRUCTURAL;
    case REFERENCE;
    case ORDERING;
    case EXISTENCE;
    case REMOVAL;
    case RENAME;
    case CAPABILITY;
    case EXTENSION;
}
```

---

# 117. Dependency graph is derived

Como en Schema Model:

```text
AST nodes
=
source of intent truth
```

El dependency graph será una representación derivada.

---

# 118. Planner ownership

Dependency analysis puede detectar relaciones.

Pero:

```text
Dependency Analysis
≠
Final Planning
```

El planner decide el orden ejecutable.

---

# 119. Cycles

Ejemplo:

```text
Table A FK → B
Table B FK → A
```

No significa necesariamente AST inválido.

Puede requerir:

```text
Create A
Create B
Add FK A→B
Add FK B→A
```

---

# 120. Cycle handling

El planner deberá poder romper ciclos mediante estrategias semánticamente válidas.

El AST conservará la intención original.

---

# 121. Rename dependency

Ejemplo:

```text
RenameTable(users → accounts)
AddColumn(accounts.status)
```

La segunda operación depende de la identidad resultante del rename.

---

# 122. Symbol evolution

Para cambios complejos podrá necesitarse:

```text
SchemaSymbolEvolutionMap
```

que rastree:

```text
old object identity
→ new object identity
```

---

# 123. Symbol evolution ≠ mutation

No modifica el AST.

Es metadata derivada para análisis/planning.

---

# 124. AST application

Debe ser posible modelar conceptualmente:

```text
Apply(AST, Schema₀)
→ Schema₁
```

sin ejecutar una base de datos.

---

# 125. Pure schema simulation

Un:

```text
SchemaAstInterpreter
```

podría aplicar operaciones al Schema Model en memoria.

---

# 126. Interpreter purpose

Servirá para:

```text
validation
planning
diff verification
testing
migration preview
schema evolution simulation
```

---

# 127. Interpreter ≠ executor

Nunca:

```text
SchemaAstInterpreter
→ database connection
```

---

# 128. Formal transformation

```text
Interpret(S₀, AST)
=
S₁
```

donde:

```text
S₀ = initial schema state
S₁ = predicted structural state
```

---

# 129. Semantic invariant

Si la ejecución física tiene éxito correctamente:

```text
ObservedSchemaAfterExecution
≈
Interpret(CurrentSchema, PlannedAST)
```

bajo las normalizaciones propias de la plataforma.

---

# 130. AST fingerprinting

El AST podrá tener fingerprints para:

```text
cache
comparison
diagnostics
planning reuse
testing
```

---

# 131. Fingerprint content

Conceptualmente:

```text
SchemaAstFingerprint
=
H(
    ASTVersion,
    NormalizationVersion,
    CanonicalNodes,
    SemanticAnnotations,
    ExtensionVersions
)
```

---

# 132. Fingerprint excludes runtime state

No incluir:

```text
connection
PDO
request ID
worker ID
timestamp unless semantically relevant
memory address
```

---

# 133. SourceLocation exclusion

Source location no deberá participar en semantic fingerprint.

---

# 134. Multiple fingerprints

Puede haber:

```text
StructuralAstFingerprint
SemanticAstFingerprint
ExactAstFingerprint
```

---

# 135. AST equality

Debe distinguirse:

```text
identity
exact equality
structural equality
semantic equivalence
```

---

# 136. Example equivalence

Dos AST podrían ser:

```text
AST A:
AddColumn x
AddColumn y
```

```text
AST B:
AddColumn y
AddColumn x
```

Podrían ser semánticamente equivalentes sólo si:

```text
no ordering dependency
no placement semantics
no dependent operations
```

Esto no deberá asumirse universalmente.

---

# 137. AST serialization

Schema AST deberá ser serializable.

Casos:

```text
migration planning
CI
schema preview
cached plans
developer tools
diagnostics
remote deployment tooling
```

---

# 138. Versioned serialization

El formato deberá incluir:

```text
AST format version
node kind version
extension version
normalization version
```

cuando sea necesario.

---

# 139. Unknown node type

Al deserializar:

```text
unknown node
```

no deberá descartarse silenciosamente.

---

# 140. Unknown node policy

```php
enum UnknownSchemaNodePolicy
{
    case FAIL;
    case PRESERVE_OPAQUE;
    case DIAGNOSTIC_ONLY;
}
```

---

# 141. Extension nodes

```php
interface ExtensionSchemaNode extends SchemaNode
{
    public function extensionId(): SchemaExtensionId;

    public function payload(): ExtensionSchemaNodePayload;
}
```

---

# 142. Extension payload

Debe ser:

```text
typed
versioned
validated
serializable
deterministic
```

---

# 143. Extension boundaries

Una extensión de Schema AST podrá agregar nuevas intenciones estructurales.

No podrá:

```text
open DB connections
execute SQL
change transactions
mutate global registries
bypass security
inject arbitrary unvalidated SQL
```

---

# 144. Extension registration

Se propone:

```text
SchemaAstExtensionRegistry
```

congelado después del bootstrap.

---

# 145. No last-wins

Si dos extensiones registran el mismo:

```text
SchemaNodeKindId
```

sin resolución explícita:

```text
startup failure
```

---

# 146. Extension capability declaration

Una extensión deberá declarar:

```text
supported platforms
required capabilities
node versions
compiler contributions
validation contributions
serialization contract
fingerprint contract
```

---

# 147. Raw DDL escape hatch

Podrá existir:

```text
RawSchemaOperationNode
```

pero deberá considerarse una frontera de confianza.

---

# 148. Raw DDL node

Conceptualmente:

```php
final readonly class RawSchemaOperationNode implements SchemaNode
{
    public function __construct(
        public SchemaNodeId $id,
        public TrustedSchemaSql $sql,
        public RawSchemaOperationContract $contract,
        public SchemaNodeMetadata $metadata,
    ) {}
}
```

---

# 149. Raw operation contract

Debe declarar al menos:

```text
expected structural effect
platform scope
transaction characteristics
result expectations
safety classification
```

cuando sea posible.

---

# 150. Raw SQL does not become portable

```text
Raw DDL
≠
Portable Schema AST
```

---

# 151. Raw DDL cannot be semantically inspected reliably

Por ello funcionalidades como:

```text
schema diff
dependency analysis
rollback inference
zero-downtime safety
portability
```

podrán degradarse.

---

# 152. Diagnostics must expose raw boundary

Herramientas deberán poder mostrar:

```text
Raw schema operation prevents complete structural analysis.
```

---

# 153. Transaction characteristics

Algunas operaciones DDL pueden tener restricciones transaccionales según plataforma.

El AST podrá declarar requisitos conocidos:

```text
TransactionRequirement
```

pero no iniciar transacciones.

---

# 154. TransactionRequirement

Ejemplo:

```php
enum SchemaTransactionRequirement
{
    case ANY;
    case REQUIRES_TRANSACTION;
    case REQUIRES_NO_TRANSACTION;
    case PLATFORM_DEPENDENT;
    case UNKNOWN;
}
```

---

# 155. Requirement ≠ transaction execution

El Transaction Manager seguirá siendo dueño de:

```text
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
```

---

# 156. Lock impact hints

Operaciones pueden llevar:

```text
SchemaLockImpactHint
```

como metadata.

Ejemplo:

```text
LOW
MODERATE
HIGH
PLATFORM_DEPENDENT
UNKNOWN
```

---

# 157. Lock hint ≠ guarantee

El AST no deberá prometer:

```text
this operation will not lock
```

si depende del motor/version/data size.

---

# 158. Data impact

Algunas transformaciones pueden afectar datos.

Ejemplo:

```text
DROP COLUMN
ALTER TYPE
SET NOT NULL
```

Podrá existir:

```text
SchemaDataImpactHint
```

---

# 159. Data impact categories

```text
NONE
POTENTIAL_REWRITE
POTENTIAL_DATA_LOSS
VALIDATION_REQUIRED
PLATFORM_DEPENDENT
UNKNOWN
```

---

# 160. Data migration ≠ Schema AST

Transformaciones de datos arbitrarias:

```text
UPDATE users SET ...
```

no pertenecen al Schema AST.

---

# 161. Schema change with data dependency

Una migration podrá combinar:

```text
Schema AST
Data operation
Schema AST
```

pero son subsistemas distintos.

---

# 162. Example

```text
Add nullable column
↓
Backfill data
↓
Set NOT NULL
```

El AST modela:

```text
AddColumn
AlterColumn
```

El backfill pertenece al Query/Execution/Migration orchestration.

---

# 163. Zero-downtime relevance

Preservar operaciones de alto nivel permite detectar patrones como:

```text
rename column
drop column
type narrowing
not-null introduction
unique constraint creation
index creation
```

para que `Zero Downtime Migration System` pueda analizarlos.

---

# 164. AST should not prematurely lower

Incorrecto:

```text
RenameColumn
↓ immediately
DropColumn + AddColumn
```

Correcto:

```text
RenameColumn
↓
Planner
↓
platform-compatible strategy
```

---

# 165. Platform capabilities

El AST será mayormente platform-neutral.

La compatibilidad se evaluará con:

```text
PlatformCapabilitySnapshot
```

---

# 166. Version ≠ capability

Nunca:

```php
if ($mysqlVersion >= ...)
```

dentro de los nodos.

---

# 167. Capability examples

```text
supportsRenameColumn
supportsDropColumn
supportsAlterColumnType
supportsSequences
supportsCheckConstraints
supportsDeferrableConstraints
supportsPartialIndexes
supportsExpressionIndexes
supportsTransactionalDDL
```

---

# 168. AST must not contain live Platform object

Puede conservar:

```text
capability requirements
```

pero no:

```text
live platform resolver
connection
driver
```

---

# 169. Capability requirements

```php
final readonly class SchemaCapabilityRequirementSet
{
}
```

puede derivarse de un AST.

---

# 170. Requirement derivation

Ejemplo:

```text
CreateSequenceNode
↓
requires SEQUENCES
```

---

# 171. Capability graph

Para AST complejo:

```text
AST
↓
CapabilityRequirementAnalyzer
↓
SchemaCapabilityRequirementSet
```

---

# 172. Unsupported capability response

El resultado puede ser:

```text
SUPPORTED
SUPPORTED_WITH_PLANNING
UNSUPPORTED
UNKNOWN
```

---

# 173. Supported with planning

Ejemplo:

Una operación lógica puede no existir como una sola instrucción SQL, pero ser realizable mediante una secuencia segura de operaciones.

La decisión pertenece al planner.

---

# 174. Compiler boundary

El compiler deberá recibir:

```text
planned schema operation
```

no reinterpretar AST arbitrariamente.

---

# 175. Planning chain

```text
Schema AST
↓
Validation
↓
Dependency Analysis
↓
Capability Analysis
↓
Schema Planner
↓
Planned Schema Operations
↓
Schema Compiler
```

---

# 176. Compiler invariant

```text
Compiler may change representation.
Compiler must not change structural intent.
```

---

# 177. Schema planner invariant

```text
Planner may choose implementation strategy.
Planner must preserve requested structural outcome.
```

---

# 178. Desired state invariant

Si:

```text
S₁ = Interpret(S₀, AST)
```

entonces un plan válido `P` debe satisfacer:

```text
Semantics(Apply(P, S₀))
=
Semantics(S₁)
```

dentro de las reglas de normalización de la plataforma.

---

# 179. AST builder

Podrá existir un API interno:

```php
final class SchemaAstBuilder
{
    public function createTable(TableDefinition $table): self;

    public function addColumn(
        TableSchemaReference $table,
        ColumnDefinition $column,
    ): self;

    public function build(): SchemaAst;
}
```

---

# 180. AST builder mutable scope

El builder puede ser mutable mientras se construye.

Pero:

```text
SchemaAstBuilder
=
operation-scoped
```

y:

```text
build()
→ immutable SchemaAst
```

---

# 181. Public Schema Builder ≠ AST Builder

Debe distinguirse:

```text
Developer Schema Builder
```

de:

```text
Internal SchemaAstBuilder
```

---

# 182. Developer Builder

Optimizado para ergonomía:

```php
$table->string('email');
```

---

# 183. AST Builder

Optimizado para:

```text
precise typed representation
canonical nodes
internal framework usage
```

---

# 184. No fluent API leakage

El AST no deberá depender de estados mutables propios del fluent builder.

---

# 185. Node visitors

Podrá existir:

```php
interface SchemaNodeVisitor
{
}
```

para:

```text
validation
normalization
dependency analysis
serialization
diagnostics
fingerprinting
```

---

# 186. Visitor ≠ generic switch dumping ground

No deberá convertirse en:

```php
switch ($node::class) {
    // 300 cases
}
```

disperso por todo el framework.

---

# 187. Typed dispatch

Preferible:

```text
NodeKind
→ registered typed handler
```

con registry congelado.

---

# 188. Visitor purity

Visitors de análisis deberán ser:

```text
side-effect free
```

salvo acumuladores operation-scoped explícitos.

---

# 189. AST traversal

El traversal deberá ser determinista.

---

# 190. AST recursion

Schema AST normalmente será poco profundo.

Pero expresiones y operaciones agrupadas pueden formar árboles arbitrarios.

Se requerirá:

```text
max AST depth
max node count
max expression depth
```

como budgets.

---

# 191. SchemaAstBudget

```php
final readonly class SchemaAstBudget
{
    public function __construct(
        public int $maxNodes,
        public int $maxDepth,
        public int $maxExpressions,
        public int $maxMetadataEntries,
        public int $maxSerializedBytes,
    ) {}
}
```

---

# 192. Budget exhaustion

Debe fallar explícitamente:

```text
SchemaAstBudgetExceededException
```

No truncar silenciosamente el AST.

---

# 193. Security model

Schema AST puede influir en:

```text
DDL
destructive operations
identifiers
raw expressions
extensions
```

por lo que es una frontera sensible.

---

# 194. AST trust levels

Puede existir:

```php
enum SchemaAstTrustLevel
{
    case FRAMEWORK_GENERATED;
    case APPLICATION_GENERATED;
    case EXTENSION_GENERATED;
    case EXTERNAL_UNTRUSTED;
}
```

---

# 195. Trust does not skip validation

Incluso:

```text
FRAMEWORK_GENERATED
```

deberá mantener invariantes.

---

# 196. External AST

AST recibido desde:

```text
CLI input
remote deployment system
serialized artifact
plugin
```

deberá validarse antes de planificación.

---

# 197. Dangerous operation policy

El AST puede identificar:

```text
DROP TABLE
DROP COLUMN
CASCADE
type narrowing
```

pero autorización/policy pertenece a capas posteriores.

---

# 198. SQL injection

Mientras el AST mantenga:

```text
Identifier
Expression
Literal
RawSql
```

como categorías separadas, se reduce el riesgo de mezclar datos con gramática.

---

# 199. Raw operation security

`RawSchemaOperationNode` deberá exigir una API explícita.

No deberá surgir accidentalmente desde un string.

---

# 200. Persistent runtime safety

AST publicado será:

```text
immutable
```

por lo que puede reutilizarse cuando su scope lo permita.

---

# 201. No global current AST

Nunca:

```php
SchemaAst::$current
```

---

# 202. FrankenPHP

Bajo FrankenPHP:

```text
Request A AST
≠
Request B AST
```

salvo artefactos explícitamente compartidos e inmutables.

---

# 203. RoadRunner/OpenSwoole

La misma regla aplicará a futuros runtimes persistentes.

---

# 204. Extension registry

Puede ser compartido entre requests sólo si está:

```text
frozen
immutable
```

---

# 205. Mutable analysis session

Cualquier estado temporal deberá vivir en:

```text
SchemaAstAnalysisSession
```

operation-scoped.

---

# 206. Analysis session

Podrá contener:

```text
visited nodes
diagnostics
symbol maps
dependency graph builder
budget counters
extension state
```

---

# 207. Analysis session not serialized

El session state no forma parte del AST.

---

# 208. Telemetry

Eventos potenciales:

```text
SchemaAstCreated
SchemaAstNormalized
SchemaAstValidated
SchemaAstValidationFailed
SchemaAstDependencyAnalyzed
SchemaAstCapabilityAnalyzed
SchemaAstSerialized
SchemaAstDeserialized
SchemaAstBudgetExceeded
SchemaAstExtensionResolved
```

---

# 209. Telemetry privacy

No deberá registrar automáticamente:

```text
raw DDL
full schema names
comments
default secrets
extension payloads
```

---

# 210. Metrics

Podrán registrarse:

```text
AST node count
normalization duration
validation duration
dependency count
extension node count
destructive operation count
unknown capability count
```

---

# 211. No high-cardinality identifiers

Evitar labels como:

```text
table_name
column_name
migration_name
```

en métricas globales.

---

# 212. Error hierarchy

Se propone:

```text
DatabaseSchemaAstException
├── InvalidSchemaAstException
├── InvalidSchemaNodeException
├── DuplicateSchemaNodeException
├── InvalidSchemaNodeReferenceException
├── SchemaAstNormalizationException
├── SchemaAstValidationException
├── SchemaAstCapabilityException
├── SchemaAstDependencyException
├── SchemaAstCycleException
├── SchemaAstSerializationException
├── SchemaAstDeserializationException
├── SchemaAstFingerprintException
├── SchemaAstBudgetExceededException
├── SchemaAstSecurityException
├── SchemaAstExtensionException
├── UnsupportedSchemaNodeException
├── RawSchemaOperationException
└── SchemaAstInvariantException
```

---

# 213. Diagnostic structure

Cada error deberá poder incluir:

```text
node ID
node kind
object path
source location
capability
platform context
diagnostic code
safe message
cause
```

cuando aplique.

---

# 214. Stable diagnostic codes

Ejemplo:

```text
DB_SCHEMA_AST_DUPLICATE_NODE
DB_SCHEMA_AST_INVALID_REFERENCE
DB_SCHEMA_AST_UNSUPPORTED_CAPABILITY
DB_SCHEMA_AST_DESTRUCTIVE_OPERATION
```

---

# 215. Exception message safety

No incluir automáticamente:

```text
credentials
row data
secret defaults
raw external SQL
```

---

# 216. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Ast
```

---

# 217. Proposed directory structure

```text
Schema/
└── Ast/
    ├── SchemaAst.php
    ├── SchemaAstId.php
    ├── SchemaAstVersion.php
    │
    ├── Node/
    │   ├── SchemaNode.php
    │   ├── SchemaNodeId.php
    │   ├── SchemaNodeKind.php
    │   ├── SchemaNodeCollection.php
    │   │
    │   ├── Namespace/
    │   │   ├── CreateSchemaNode.php
    │   │   └── DropSchemaNode.php
    │   │
    │   ├── Table/
    │   │   ├── CreateTableNode.php
    │   │   ├── AlterTableNode.php
    │   │   ├── RenameTableNode.php
    │   │   └── DropTableNode.php
    │   │
    │   ├── Column/
    │   │   ├── AddColumnNode.php
    │   │   ├── AlterColumnNode.php
    │   │   ├── RenameColumnNode.php
    │   │   └── DropColumnNode.php
    │   │
    │   ├── Index/
    │   │   ├── CreateIndexNode.php
    │   │   └── DropIndexNode.php
    │   │
    │   ├── Constraint/
    │   │   ├── AddConstraintNode.php
    │   │   ├── DropConstraintNode.php
    │   │   ├── AddForeignKeyNode.php
    │   │   └── DropForeignKeyNode.php
    │   │
    │   ├── Sequence/
    │   │   ├── CreateSequenceNode.php
    │   │   ├── AlterSequenceNode.php
    │   │   └── DropSequenceNode.php
    │   │
    │   ├── View/
    │   │   ├── CreateViewNode.php
    │   │   ├── ReplaceViewNode.php
    │   │   └── DropViewNode.php
    │   │
    │   ├── Raw/
    │   │   └── RawSchemaOperationNode.php
    │   │
    │   └── Extension/
    │       └── ExtensionSchemaNode.php
    │
    ├── Definition/
    │   ├── TableDefinition.php
    │   ├── ColumnDefinition.php
    │   ├── IndexDefinition.php
    │   ├── ConstraintDefinition.php
    │   ├── ForeignKeyDefinition.php
    │   └── SequenceDefinition.php
    │
    ├── Change/
    │   ├── Change.php
    │   ├── Unchanged.php
    │   ├── SetValue.php
    │   ├── RemoveValue.php
    │   └── ColumnChangeSet.php
    │
    ├── Reference/
    │   ├── SchemaAstReference.php
    │   ├── TableSchemaReference.php
    │   ├── ColumnSchemaReference.php
    │   ├── IndexSchemaReference.php
    │   ├── ConstraintSchemaReference.php
    │   └── ForeignKeySchemaReference.php
    │
    ├── Expression/
    │   ├── SchemaExpression.php
    │   ├── SchemaExpressionKind.php
    │   ├── LiteralExpression.php
    │   ├── ColumnReferenceExpression.php
    │   ├── FunctionExpression.php
    │   ├── UnaryExpression.php
    │   ├── BinaryExpression.php
    │   ├── CastExpression.php
    │   └── RawSchemaExpression.php
    │
    ├── Behavior/
    │   ├── CreationBehavior.php
    │   ├── DropBehavior.php
    │   ├── DependencyDropBehavior.php
    │   └── SchemaTransactionRequirement.php
    │
    ├── Metadata/
    │   ├── SchemaNodeMetadata.php
    │   ├── SchemaNodeOrigin.php
    │   ├── SchemaOperationCategory.php
    │   ├── SchemaOperationRiskClass.php
    │   ├── SchemaLockImpactHint.php
    │   └── SchemaDataImpactHint.php
    │
    ├── Normalize/
    │   ├── SchemaAstNormalizer.php
    │   ├── SchemaAstNormalizationContext.php
    │   └── NormalizedSchemaAst.php
    │
    ├── Validation/
    │   ├── SchemaAstValidator.php
    │   ├── SchemaAstValidationContext.php
    │   └── SchemaAstValidationResult.php
    │
    ├── Dependency/
    │   ├── SchemaNodeDependency.php
    │   ├── SchemaNodeDependencyKind.php
    │   ├── SchemaNodeDependencyGraph.php
    │   └── SchemaDependencyAnalyzer.php
    │
    ├── Capability/
    │   ├── SchemaCapabilityRequirement.php
    │   ├── SchemaCapabilityRequirementSet.php
    │   └── SchemaCapabilityAnalyzer.php
    │
    ├── Interpret/
    │   ├── SchemaAstInterpreter.php
    │   ├── SchemaInterpretationContext.php
    │   └── SchemaInterpretationResult.php
    │
    ├── Fingerprint/
    │   ├── SchemaAstFingerprint.php
    │   ├── StructuralAstFingerprint.php
    │   ├── SemanticAstFingerprint.php
    │   └── ExactAstFingerprint.php
    │
    ├── Serialization/
    │   ├── SchemaAstSerializer.php
    │   ├── SchemaAstDeserializer.php
    │   └── SchemaAstSerializationVersion.php
    │
    ├── Extension/
    │   ├── SchemaAstExtension.php
    │   ├── SchemaAstExtensionRegistry.php
    │   ├── SchemaExtensionNodePayload.php
    │   └── UnknownSchemaNodePolicy.php
    │
    ├── Session/
    │   └── SchemaAstAnalysisSession.php
    │
    ├── Budget/
    │   └── SchemaAstBudget.php
    │
    └── Exception/
        ├── DatabaseSchemaAstException.php
        ├── InvalidSchemaAstException.php
        ├── InvalidSchemaNodeException.php
        ├── DuplicateSchemaNodeException.php
        ├── InvalidSchemaNodeReferenceException.php
        ├── SchemaAstNormalizationException.php
        ├── SchemaAstValidationException.php
        ├── SchemaAstCapabilityException.php
        ├── SchemaAstDependencyException.php
        ├── SchemaAstCycleException.php
        ├── SchemaAstSerializationException.php
        ├── SchemaAstFingerprintException.php
        ├── SchemaAstBudgetExceededException.php
        ├── SchemaAstSecurityException.php
        ├── SchemaAstExtensionException.php
        ├── UnsupportedSchemaNodeException.php
        ├── RawSchemaOperationException.php
        └── SchemaAstInvariantException.php
```

---

# 218. Dependency rules

Permitido:

```text
Schema AST
→ Schema shared structural values

Schema AST
→ Identifier system

Schema AST
→ Database type system

Schema AST
→ immutable capability contracts

Schema AST
→ immutable extension contracts
```

No permitido:

```text
Schema AST
→ PDO
Schema AST
→ live Connection
Schema AST
→ Driver execution
Schema AST
→ QueryExecutor
Schema AST
→ ORM EntityManager
Schema AST
→ Migration Repository
Schema AST
→ HTTP Request
```

---

# 219. Architectural invariants

## DB-SCHEMA-AST-001

Schema AST representará intención estructural.

## DB-SCHEMA-AST-002

Schema AST no representará estado estructural actual.

## DB-SCHEMA-AST-003

Schema AST será distinto de Schema Model.

## DB-SCHEMA-AST-004

Schema AST será distinto de Schema Diff.

## DB-SCHEMA-AST-005

Schema AST será distinto de Migration.

## DB-SCHEMA-AST-006

Schema AST será distinto de Schema Change Plan.

## DB-SCHEMA-AST-007

Schema AST será distinto de SQL AST.

## DB-SCHEMA-AST-008

Schema AST no generará SQL.

## DB-SCHEMA-AST-009

Schema AST no ejecutará SQL.

## DB-SCHEMA-AST-010

Schema AST no abrirá conexiones.

## DB-SCHEMA-AST-011

Schema AST publicado será inmutable.

## DB-SCHEMA-AST-012

SchemaNode publicado será inmutable.

## DB-SCHEMA-AST-013

SchemaNodeId será distinto de SchemaObjectId.

## DB-SCHEMA-AST-014

SchemaNodeId será distinto de ExecutionUnitId.

## DB-SCHEMA-AST-015

Declared order será distinto de execution order.

## DB-SCHEMA-AST-016

AST conservará el declared order.

## DB-SCHEMA-AST-017

Planner podrá reorganizar sólo cuando preserve semántica.

## DB-SCHEMA-AST-018

RenameTable será distinto de Drop+Create.

## DB-SCHEMA-AST-019

RenameColumn será distinto de Drop+Add.

## DB-SCHEMA-AST-020

AlterColumn conservará diferencias entre unchanged/set/remove.

## DB-SCHEMA-AST-021

No Default será distinto de DEFAULT NULL.

## DB-SCHEMA-AST-022

Structural identifiers serán tipados.

## DB-SCHEMA-AST-023

Identifier será distinto de runtime parameter.

## DB-SCHEMA-AST-024

Identifier quoting no pertenecerá al AST.

## DB-SCHEMA-AST-025

Raw SQL no será fallback automático.

## DB-SCHEMA-AST-026

RawSchemaOperation será explícita.

## DB-SCHEMA-AST-027

RawSchemaOperation será diagnostic-visible.

## DB-SCHEMA-AST-028

RawSchemaOperation será non-portable por defecto.

## DB-SCHEMA-AST-029

Raw operations no bypassarán security.

## DB-SCHEMA-AST-030

Schema expressions serán estructuradas por defecto.

## DB-SCHEMA-AST-031

SchemaExpression será distinta de QueryExpression.

## DB-SCHEMA-AST-032

AST normalization preservará intención.

## DB-SCHEMA-AST-033

AST normalization será determinista.

## DB-SCHEMA-AST-034

AST normalization será idempotente.

## DB-SCHEMA-AST-035

AST validation no realizará hidden DB I/O.

## DB-SCHEMA-AST-036

Capability analysis no realizará hidden DB I/O.

## DB-SCHEMA-AST-037

Dependency analysis no realizará hidden DB I/O.

## DB-SCHEMA-AST-038

AST interpreter no ejecutará DB I/O.

## DB-SCHEMA-AST-039

AST interpreter será una transformación in-memory.

## DB-SCHEMA-AST-040

AST dependencies serán explícitamente modelables.

## DB-SCHEMA-AST-041

Dependency graph será derivado.

## DB-SCHEMA-AST-042

Dependency graph no será segunda source of truth.

## DB-SCHEMA-AST-043

Cycles no implicarán automáticamente AST inválido.

## DB-SCHEMA-AST-044

Cycle resolution pertenecerá al planner.

## DB-SCHEMA-AST-045

Cross-table dependencies serán preservadas.

## DB-SCHEMA-AST-046

Foreign key dependencies serán explícitas.

## DB-SCHEMA-AST-047

Capability requirements serán explícitos o derivables.

## DB-SCHEMA-AST-048

Version será distinta de capability.

## DB-SCHEMA-AST-049

Unsupported capability no implicará hidden emulation.

## DB-SCHEMA-AST-050

Planning podrá elegir emulación sólo si preserva semántica.

## DB-SCHEMA-AST-051

Compiler no decidirá semántica estructural.

## DB-SCHEMA-AST-052

Compiler no reinterpretará rename como drop/create arbitrariamente.

## DB-SCHEMA-AST-053

Schema planner preservará desired structural outcome.

## DB-SCHEMA-AST-054

AST no iniciará transactions.

## DB-SCHEMA-AST-055

AST podrá declarar transaction requirements.

## DB-SCHEMA-AST-056

Transaction requirements no equivaldrán a execution.

## DB-SCHEMA-AST-057

Risk metadata no autorizará operaciones.

## DB-SCHEMA-AST-058

Lock impact hints no serán garantías.

## DB-SCHEMA-AST-059

Data impact hints no serán garantías.

## DB-SCHEMA-AST-060

Data migration será distinta de Schema AST.

## DB-SCHEMA-AST-061

Schema AST no representará arbitrary UPDATE backfills.

## DB-SCHEMA-AST-062

AST podrá coexistir con data operations en migrations.

## DB-SCHEMA-AST-063

AST fingerprint será determinista.

## DB-SCHEMA-AST-064

AST fingerprint será versionado.

## DB-SCHEMA-AST-065

Runtime state no participará en semantic fingerprint.

## DB-SCHEMA-AST-066

SourceLocation no participará en semantic fingerprint.

## DB-SCHEMA-AST-067

AST serialization será versionada.

## DB-SCHEMA-AST-068

Unknown nodes no serán descartados silenciosamente.

## DB-SCHEMA-AST-069

Extensions serán tipadas.

## DB-SCHEMA-AST-070

Extensions serán versionadas.

## DB-SCHEMA-AST-071

Extensions declararán capabilities.

## DB-SCHEMA-AST-072

Extensions declararán serialization semantics.

## DB-SCHEMA-AST-073

Extensions declararán fingerprint semantics.

## DB-SCHEMA-AST-074

Extension registry será frozen después del bootstrap.

## DB-SCHEMA-AST-075

No existirá last-wins extension resolution.

## DB-SCHEMA-AST-076

AST builder mutable será operation-scoped.

## DB-SCHEMA-AST-077

AST analysis state será operation-scoped.

## DB-SCHEMA-AST-078

No existirá global current AST.

## DB-SCHEMA-AST-079

AST será persistent-runtime safe.

## DB-SCHEMA-AST-080

FrankenPHP request state no se filtrará entre AST operations.

## DB-SCHEMA-AST-081

RoadRunner futuro conservará AST isolation.

## DB-SCHEMA-AST-082

OpenSwoole futuro conservará AST isolation.

## DB-SCHEMA-AST-083

AST budgets serán explícitos.

## DB-SCHEMA-AST-084

Budget exhaustion fallará explícitamente.

## DB-SCHEMA-AST-085

AST no será truncado silenciosamente.

## DB-SCHEMA-AST-086

AST traversal será determinista.

## DB-SCHEMA-AST-087

AST validation preservará source diagnostics.

## DB-SCHEMA-AST-088

Diagnostic output respetará sensitivity.

## DB-SCHEMA-AST-089

Telemetry no registrará full AST por defecto.

## DB-SCHEMA-AST-090

Telemetry no registrará raw DDL por defecto.

## DB-SCHEMA-AST-091

AST trust level no eliminará validation.

## DB-SCHEMA-AST-092

External AST será tratado como untrusted structured input.

## DB-SCHEMA-AST-093

Dangerous operations serán identificables.

## DB-SCHEMA-AST-094

Danger classification será distinta de authorization.

## DB-SCHEMA-AST-095

Drop CASCADE no será inferido automáticamente.

## DB-SCHEMA-AST-096

Create IF NOT EXISTS será intención explícita.

## DB-SCHEMA-AST-097

Drop IF EXISTS será intención explícita.

## DB-SCHEMA-AST-098

Idempotency de migrations no dependerá únicamente de IF EXISTS.

## DB-SCHEMA-AST-099

TableDefinition será distinta de existing Table identity.

## DB-SCHEMA-AST-100

ColumnDefinition será distinta de existing Column identity.

## DB-SCHEMA-AST-101

AST podrá reutilizar immutable structural value objects.

## DB-SCHEMA-AST-102

Shared values no introducirán dependencia al runtime.

## DB-SCHEMA-AST-103

AST no dependerá de ORM.

## DB-SCHEMA-AST-104

AST no dependerá de EntityManager.

## DB-SCHEMA-AST-105

AST no dependerá de QueryExecutor.

## DB-SCHEMA-AST-106

AST no dependerá de Migration Repository.

## DB-SCHEMA-AST-107

AST no dependerá de HTTP.

## DB-SCHEMA-AST-108

AST no dependerá de PDO.

## DB-SCHEMA-AST-109

AST no contendrá live driver handles.

## DB-SCHEMA-AST-110

AST no contendrá credentials.

## DB-SCHEMA-AST-111

AST no contendrá row data.

## DB-SCHEMA-AST-112

AST no será service locator.

## DB-SCHEMA-AST-113

AST operations tendrán typed targets.

## DB-SCHEMA-AST-114

AST references no serán raw qualified strings por defecto.

## DB-SCHEMA-AST-115

AST deberá soportar partial schema validation.

## DB-SCHEMA-AST-116

Unknown base schema facts no se convertirán en false.

## DB-SCHEMA-AST-117

AST interpreter deberá respetar partial schema semantics.

## DB-SCHEMA-AST-118

AST application no implicará physical DB execution.

## DB-SCHEMA-AST-119

AST simulation deberá producir un nuevo immutable Schema Model.

## DB-SCHEMA-AST-120

Schema Model no será mutado durante simulation.

## DB-SCHEMA-AST-121

Operation origin no cambiará structural intent.

## DB-SCHEMA-AST-122

Node metadata no se confundirá con structural semantics.

## DB-SCHEMA-AST-123

Semantic metadata sí podrá participar en semantic fingerprint.

## DB-SCHEMA-AST-124

Operational preferences serán distintas de structural intent.

## DB-SCHEMA-AST-125

Concurrent index preference no contaminará portable IndexDefinition.

## DB-SCHEMA-AST-126

Platform-specific strategy pertenecerá al planning/compiler correspondiente.

## DB-SCHEMA-AST-127

AST mantendrá MariaDB como plataforma first-class.

## DB-SCHEMA-AST-128

AST mantendrá PostgreSQL como plataforma first-class.

## DB-SCHEMA-AST-129

AST mantendrá MySQL como plataforma first-class.

## DB-SCHEMA-AST-130

AST mantendrá SQLite como plataforma first-class.

## DB-SCHEMA-AST-131

AST no usará vendor conditionals dispersos.

## DB-SCHEMA-AST-132

Capabilities dirigirán compatibilidad.

## DB-SCHEMA-AST-133

AST no inventará capacidades ausentes.

## DB-SCHEMA-AST-134

Planner no podrá cambiar desired schema outcome silenciosamente.

## DB-SCHEMA-AST-135

Compiler no podrá eliminar nodos unsupported silenciosamente.

## DB-SCHEMA-AST-136

Unknown extension operations fallarán o se preservarán según política explícita.

## DB-SCHEMA-AST-137

Schema AST tendrá una única fuente de intención estructural.

## DB-SCHEMA-AST-138

Derived analysis artifacts podrán reconstruirse.

## DB-SCHEMA-AST-139

AST deberá ser usable para testing sin una base de datos.

## DB-SCHEMA-AST-140

AST será la representación canónica de transformación estructural previa al planning.

---

# 220. Anti-patterns

## 220.1 SQL strings as AST

Incorrecto:

```php
$ast->add(
    'ALTER TABLE users ADD COLUMN age INT'
);
```

Correcto:

```text
AddColumnNode
├── table = users
└── column
    ├── name = age
    └── type = IntegerType
```

---

## 220.2 Executing from nodes

Incorrecto:

```php
$node->execute($pdo);
```

Correcto:

```text
SchemaNode
↓
Planner
↓
Compiler
↓
Execution Engine
```

---

## 220.3 PDO inside AST

Incorrecto:

```php
final class CreateTableNode
{
    public PDO $pdo;
}
```

---

## 220.4 Vendor branching inside node

Incorrecto:

```php
if ($platform === 'mysql') {
    ...
}
```

dentro de `AddColumnNode`.

---

## 220.5 Rename lowering too early

Incorrecto:

```text
RenameColumn
↓
DropColumn
AddColumn
```

antes del planning.

---

## 220.6 Null means unchanged

Incorrecto:

```php
new AlterColumnNode(
    default: null
);
```

sin saber si significa:

```text
unchanged
remove default
DEFAULT NULL
```

---

## 220.7 Global mutable AST

Incorrecto:

```php
SchemaAstRegistry::$currentAst = $ast;
```

---

## 220.8 Hidden schema lookup

Incorrecto:

```php
$node->validate()
{
    return DB::tableExists(...);
}
```

---

## 220.9 Silent unsupported operation

Incorrecto:

```text
Platform does not support operation
↓
skip node
↓
continue
```

---

## 220.10 Raw fallback

Incorrecto:

```php
if (!$compiler->supports($node)) {
    return (string) $node;
}
```

---

# 221. Example: create users table

Developer API:

```php
Schema::create('users', function (TableBlueprint $table) {
    $table->id();
    $table->string('email', 320);
    $table->string('name', 150);
    $table->unique('email');
});
```

Conceptual AST:

```text
SchemaAst
└── CreateTableNode
    └── TableDefinition(users)
        ├── ColumnDefinition(id)
        │   ├── BigInteger
        │   ├── NOT NULL
        │   └── Identity
        │
        ├── ColumnDefinition(email)
        │   ├── String(320)
        │   └── NOT NULL
        │
        ├── ColumnDefinition(name)
        │   ├── String(150)
        │   └── NOT NULL
        │
        ├── PrimaryKeyDefinition(id)
        └── UniqueConstraintDefinition(email)
```

No SQL ha sido generado.

---

# 222. Example: complex alteration

```php
Schema::table('users', function (TableBlueprint $table) {
    $table->renameColumn('name', 'full_name');
    $table->string('status', 30)->default('active');
    $table->dropColumn('legacy_code');
});
```

AST:

```text
AlterTableNode(users)
├── RenameColumn
│   ├── from = name
│   └── to   = full_name
│
├── AddColumn
│   └── status
│       ├── String(30)
│       └── Default("active")
│
└── DropColumn
    └── legacy_code
```

El AST conserva:

```text
rename
add
drop
```

como intenciones diferentes.

---

# 223. Example: circular foreign keys

Intención:

```text
A.b_id → B.id
B.a_id → A.id
```

AST:

```text
CreateTable(A)
CreateTable(B)
AddForeignKey(A → B)
AddForeignKey(B → A)
```

Dependency analysis:

```text
Create A ───────┐
                ├──▶ FK A→B
Create B ───────┘

Create A ───────┐
                ├──▶ FK B→A
Create B ───────┘
```

No es necesario crear una dependencia circular entre:

```text
Create A
Create B
```

El planner puede ejecutar:

```text
1. Create A
2. Create B
3. Add FK A→B
4. Add FK B→A
```

---

# 224. Example: schema simulation

Estado:

```text
S₀

users
├── id
└── name
```

AST:

```text
RenameColumn(users.name → full_name)
AddColumn(users.email)
```

Interpreter:

```text
Interpret(S₀, AST)
```

produce:

```text
S₁

users
├── id
├── full_name
└── email
```

sin ejecutar SQL.

---

# 225. Example: unsafe operation

AST:

```text
DropColumn(users.email)
```

Metadata:

```text
risk = DESTRUCTIVE
dataImpact = POTENTIAL_DATA_LOSS
```

Esto no significa:

```text
operation denied
```

ni:

```text
operation approved
```

Significa que subsistemas posteriores tienen información suficiente para aplicar políticas.

---

# 226. Schema AST pipeline

```text
Developer API
      │
      ▼
Schema Builder
      │
      ▼
SchemaAstBuilder
      │
      ▼
SchemaAst
      │
      ▼
Normalization
      │
      ▼
NormalizedSchemaAst
      │
      ├───────────────┐
      │               │
      ▼               ▼
Validation      Dependency Analysis
      │               │
      └───────┬───────┘
              ▼
      Capability Analysis
              │
              ▼
       Schema Planning
              │
              ▼
    Planned Schema Operations
              │
              ▼
       Schema Compiler
              │
              ▼
Compiled Database Commands
              │
              ▼
       Execution Engine
```

---

# 227. Relationship with Schema Model

```text
                Current DatabaseSchema
                         │
                         ▼
SchemaAst ───────▶ SchemaAstInterpreter
                         │
                         ▼
                 Predicted DatabaseSchema
```

---

# 228. Relationship with Schema Diff

```text
CurrentSchema
      +
TargetSchema
      │
      ▼
 SchemaDiff
      │
      ▼
Schema Change Intent
      │
      ▼
  SchemaAst
```

---

# 229. Relationship with Migration

```text
Migration
├── metadata
├── lifecycle
├── ordering
├── history
├── rollback
│
└── schema operations
        │
        ▼
     SchemaAst
```

Migration owns deployment history.

Schema AST owns structural intent.

---

# 230. Relationship with compiler

```text
SchemaAst
   │
   ▼
SchemaPlanner
   │
   ▼
PlannedSchemaOperation
   │
   ▼
SchemaCompiler
   │
   ▼
Platform-specific DDL representation
```

---

# 231. Relationship with execution

Schema AST never performs:

```text
prepare
bind
execute
commit
rollback
fetch
```

Execution remains downstream.

---

# 232. Master formula

```text
Schema AST System
=
Typed Structural Intent
+
Immutable Nodes
+
Structured References
+
Typed Definitions
+
Explicit Change Semantics
+
Expression Modeling
+
Dependency Modeling
+
Normalization
+
Validation
+
Capability Requirements
+
In-Memory Interpretation
+
Versioning
+
Fingerprinting
+
Serialization
+
Extension Control
+
Security
+
Budgets
+
Persistent Runtime Isolation
```

---

# 233. Correctness formula

```text
CorrectSchemaAst
=
StructurallyValid
∧
IntentPreserving
∧
ReferenceSafe
∧
Deterministic
∧
Immutable
∧
CapabilityHonest
∧
ExtensionSafe
∧
SecurityPreserving
∧
BudgetBounded
∧
RuntimeIndependent
```

---

# 234. Normalization formula

```text
Intent(
    Normalize(AST)
)
=
Intent(AST)
```

---

# 235. Interpretation formula

```text
S₁
=
Interpret(S₀, AST)
```

sin database I/O.

---

# 236. Planning correctness

Para:

```text
P = Plan(AST, S₀, Capabilities)
```

deberá cumplirse:

```text
StructuralOutcome(P, S₀)
=
Interpret(S₀, AST)
```

dentro de la equivalencia de plataforma aplicable.

---

# 237. Compilation correctness

Para un plan `P`:

```text
Semantics(
    Compile(P)
)
=
Semantics(P)
```

---

# 238. End-to-end structural invariant

Idealmente:

```text
ObservedSchema(
    Execute(
        Compile(
            Plan(AST)
        )
    )
)
≈
Interpret(CurrentSchema, AST)
```

donde:

```text
≈
```

representa equivalencia estructural/semántica bajo normalización de plataforma.

---

# 239. Intent preservation principle

La arquitectura completa deberá preservar:

```text
Developer Intent
      ↓
Schema Builder
      ↓
Schema AST
      ↓
Planner
      ↓
Compiler
      ↓
Execution
      ↓
Database Structure
```

sin reinterpretaciones silenciosas.

---

# 240. Regla maestra

> **El Schema AST describe el cambio estructural solicitado; no decide cómo ejecutarlo físicamente.**

En forma compacta:

```text
Schema Model
=
What exists

Schema AST
=
What should change

Schema Planner
=
How the change should be organized

Schema Compiler
=
How that plan is represented for the target database

Execution Engine
=
How the compiled operation is executed
```

---

# 241. Resultado arquitectónico

Con `Database Schema AST System`, VoltStack obtiene una frontera clara entre:

```text
Developer-facing schema APIs
```

y:

```text
database-specific DDL execution
```

La arquitectura resultante queda:

```text
┌─────────────────────────┐
│      Schema Builder     │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│       Schema AST        │
│                         │
│ structural intent only  │
└────────────┬────────────┘
             │
      ┌──────┼──────────┐
      │      │          │
      ▼      ▼          ▼
 Validation Dependency Capability
            Analysis   Analysis
      │      │          │
      └──────┴────┬─────┘
                  ▼
        ┌─────────────────┐
        │ Schema Planner  │
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │ Schema Compiler │
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │ Execution Engine│
        └─────────────────┘
```

---

# 242. Decisión arquitectónica

VoltStack adoptará:

```text
Typed Immutable Schema AST
```

como representación canónica de intención de transformación estructural.

Será:

```text
typed
immutable
deterministic
platform-neutral at the semantic level
capability-aware
dependency-aware
versioned
serializable
fingerprintable
extension-safe
persistent-runtime safe
```

y explícitamente no será:

```text
SQL
migration history
schema state
execution plan
driver command
ORM metadata
```

---

# 243. Siguiente documento

```text
90_DATABASE_SCHEMA_BUILDER_SYSTEM.md
```

El siguiente documento deberá definir la API ergonómica utilizada por desarrolladores para construir operaciones de schema:

```text
Schema
├── create()
├── table()
├── rename()
├── drop()
├── dropIfExists()
├── hasTable()
└── ...

TableBlueprint
├── id()
├── string()
├── integer()
├── decimal()
├── boolean()
├── json()
├── timestamp()
├── foreignId()
├── primary()
├── unique()
├── index()
├── foreign()
└── ...
```

manteniendo la separación:

```text
Developer Schema DSL
        ↓
Typed Definitions
        ↓
Schema AST
        ↓
Validation / Planning
        ↓
Schema Compiler
        ↓
Execution
```

El principio del siguiente sistema deberá ser:

> **El Schema Builder optimiza la experiencia del desarrollador; el Schema AST preserva la precisión arquitectónica.**