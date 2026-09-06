# 87_DATABASE_SCHEMA_ARCHITECTURE.md

# VoltStack Quantum Database
## Schema Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 87 — Database Schema Architecture  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Architecture` define la arquitectura general mediante la cual VoltStack representa, construye, inspecciona, compara, valida y transforma estructuras de bases de datos.

El sistema deberá proporcionar una representación independiente del motor para conceptos como:

```text
Database
Schema / Namespace
Table
Column
Index
Foreign Key
Primary Key
Unique Constraint
Check Constraint
Default
Sequence
Generated Column
Identity
View
```

sin convertir el modelo interno de VoltStack en una copia directa del catálogo de:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

La arquitectura deberá separar rigurosamente:

```text
Schema Model
≠
Schema AST
≠
Schema Builder
≠
Schema Introspection
≠
Schema Metadata
≠
Schema Diff
≠
Schema Compiler
≠
Migration
≠
Execution
```

Principio fundamental:

```text
Schema describes database structure.

Schema does not execute database structure changes.
```

---

# 2. Objetivo arquitectónico

VoltStack deberá permitir flujos como:

```php
$schema->create('users', function (Table $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamps();
});
```

manteniendo internamente una arquitectura mucho más rigurosa:

```text
Developer API
    ↓
Schema Builder
    ↓
Schema Operation Model
    ↓
Schema AST
    ↓
Schema Validation
    ↓
Schema Planning
    ↓
Schema Compiler
    ↓
Compiled Database Command
    ↓
Execution Engine
    ↓
Driver
```

El API sencillo no deberá sacrificar:

```text
portability
type safety
determinism
capability awareness
migration safety
schema introspection
schema diff
persistent-runtime safety
```

---

# 3. Posición dentro de Quantum Database

La arquitectura general pasa a quedar:

```text
VoltStack/Quantum/Database
│
├── Platform
│   ├── Driver
│   ├── Connection
│   ├── Dialect
│   └── Capabilities
│
├── Query
│   ├── Model
│   ├── AST
│   ├── Semantic
│   ├── Builder
│   ├── Optimizer
│   ├── Planner
│   └── Compiler
│
├── Execution
│   ├── QueryExecutor
│   ├── Statement
│   ├── Binding
│   ├── Result
│   ├── Cursor
│   ├── Streaming
│   ├── Cancellation
│   ├── Error
│   └── Retry
│
└── Schema
    ├── Model
    ├── AST
    ├── Builder
    ├── Definition
    ├── Validation
    ├── Introspection
    ├── Metadata
    ├── Diff
    ├── Planning
    ├── Compiler
    └── Capability
```

---

# 4. Frontera fundamental

La arquitectura deberá establecer:

```text
Schema Model
    ↓
Schema AST
    ↓
Schema Compiler
    ↓
Execution Engine
```

Nunca:

```text
Schema Model
    ↓
PDO
```

ni:

```text
Schema Builder
    ↓
SQL strings
```

ni:

```text
Migration
    ↓
PDO
```

---

# 5. Principio maestro

```text
Schema Builder builds structure.

Schema Model represents structure.

Schema AST represents schema operations.

Schema Diff discovers structural differences.

Schema Compiler generates database-specific representation.

Execution Engine performs the operation.
```

---

# 6. Schema Model

`Schema Model` representa una estructura de base de datos.

Ejemplo conceptual:

```text
DatabaseSchema
│
├── Table users
│   ├── Column id
│   ├── Column name
│   ├── Column email
│   ├── PrimaryKey
│   └── UniqueIndex email
│
└── Table posts
    ├── Column id
    ├── Column user_id
    ├── Column title
    └── ForeignKey user_id → users.id
```

No representa necesariamente:

```text
CREATE TABLE ...
```

Representa:

```text
what the database structure is
```

---

# 7. Schema Model ≠ Schema AST

Distinción fundamental:

```text
Schema Model
=
state
```

mientras:

```text
Schema AST
=
operations / structural transformation intent
```

Ejemplo:

```text
Schema Model

users
├── id
├── name
└── email
```

frente a:

```text
Schema AST

CreateTable(users)
AddColumn(name)
AddColumn(email)
AddUniqueConstraint(email)
```

---

# 8. Schema state vs schema operation

Formalmente:

```text
SchemaState
≠
SchemaOperation
```

Un estado puede representar:

```text
S₀
```

y una operación:

```text
O
```

produce:

```text
S₁ = Apply(O, S₀)
```

---

# 9. Schema Builder

`Schema Builder` será el API orientado al desarrollador.

Ejemplo:

```php
Schema::create('users', function (TableDefinition $table) {
    $table->id();
    $table->string('name', 150);
    $table->string('email')->unique();
});
```

Pero internamente:

```text
Schema Facade
    ↓
SchemaBuilder
    ↓
TableDefinitionBuilder
    ↓
SchemaOperation
    ↓
Schema AST
```

---

# 10. Builder does not generate SQL

Invariante:

```text
SchemaBuilder
→ Schema AST
```

Nunca:

```text
SchemaBuilder
→ SQL
```

Esto evita:

```php
$sql = 'CREATE TABLE ' . $name . ' (...)';
```

dentro del Builder.

---

# 11. Schema AST

El `Schema AST` representará operaciones estructurales.

Ejemplo:

```text
CreateTableNode
├── TableIdentifier(users)
├── Columns
│   ├── id
│   ├── name
│   └── email
├── PrimaryKey
└── UniqueConstraint
```

---

# 12. Schema AST node families

Inicialmente:

```text
SchemaNode
├── CreateSchemaNode
├── DropSchemaNode
├── CreateTableNode
├── AlterTableNode
├── RenameTableNode
├── DropTableNode
├── AddColumnNode
├── AlterColumnNode
├── RenameColumnNode
├── DropColumnNode
├── AddIndexNode
├── DropIndexNode
├── AddForeignKeyNode
├── DropForeignKeyNode
├── AddConstraintNode
├── DropConstraintNode
└── ExtensionSchemaNode
```

La lista podrá crecer mediante extensiones tipadas.

---

# 13. Schema AST ≠ Query AST

No deberán compartir nodos accidentalmente.

```text
Query AST
≠
Schema AST
```

Aunque puedan compartir primitivas de bajo nivel como:

```text
Identifier
Type
Expression
CapabilityRequirement
SourceLocation
```

cuando su semántica sea realmente común.

---

# 14. DDL vs DML

La arquitectura deberá distinguir:

```text
DDL
=
structure manipulation
```

de:

```text
DML
=
data manipulation
```

Ejemplos:

```text
CREATE TABLE
ALTER TABLE
DROP INDEX
```

pertenecen al sistema Schema.

Mientras:

```text
INSERT
UPDATE
DELETE
SELECT
```

pertenecen al Query Engine.

---

# 15. Shared compiler infrastructure

Esto no significa que Schema necesite un compilador SQL completamente aislado.

Podrá reutilizar infraestructura común:

```text
SQL Writer
Identifier Renderer
Dialect
Platform Capability Snapshot
Source Map
CompiledDatabaseCommand
```

pero:

```text
SchemaCompiler
≠
QueryCompiler
```

---

# 16. Schema compiler boundary

Flujo:

```text
Schema AST
    ↓
Schema Validation
    ↓
Schema Compilation
    ↓
CompiledDatabaseCommand
```

No:

```text
Schema AST
    ↓
execute()
```

---

# 17. Execution boundary

Una operación DDL compilada deberá entrar al mismo sistema general:

```text
CompiledDatabaseCommand
        ↓
Execution Engine
        ↓
Statement Execution
        ↓
Prepared Statement / Direct DDL Strategy
        ↓
Driver
```

dependiendo de las capacidades reales del driver/plataforma.

---

# 18. Schema operation

Contrato conceptual:

```php
interface SchemaOperation
{
    public function id(): SchemaOperationId;

    public function kind(): SchemaOperationKind;

    public function requirements(): SchemaCapabilityRequirementSet;
}
```

---

# 19. SchemaOperationKind

```php
enum SchemaOperationKind: string
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

    case ADD_FOREIGN_KEY = 'add_foreign_key';
    case DROP_FOREIGN_KEY = 'drop_foreign_key';

    case ADD_CONSTRAINT = 'add_constraint';
    case DROP_CONSTRAINT = 'drop_constraint';

    case EXTENSION = 'extension';
}
```

---

# 20. Schema definition system

El sistema de definición deberá proporcionar objetos tipados.

Ejemplo:

```php
final readonly class TableDefinition
{
    public function __construct(
        public TableIdentifier $name,
        public array $columns,
        public array $indexes,
        public array $foreignKeys,
        public array $constraints,
        public TableOptionSet $options,
    ) {}
}
```

---

# 21. Table definition

Una tabla deberá modelarse mediante estructura.

No mediante:

```php
array $table = [
    'name' => 'users',
    'sql' => '...',
];
```

---

# 22. Column definition

Ejemplo:

```php
final readonly class ColumnDefinition
{
    public function __construct(
        public ColumnIdentifier $name,
        public DatabaseType $type,
        public Nullability $nullability,
        public ?DefaultValue $default,
        public ColumnOptionSet $options,
    ) {}
}
```

---

# 23. Logical type vs physical type

Distinción esencial:

```text
VoltStack Logical Database Type
≠
Platform Physical SQL Type
```

Ejemplo:

```text
STRING(255)
```

puede compilarse como:

```text
VARCHAR(255)
```

pero la decisión concreta pertenece al:

```text
Schema Compiler
+
Platform
+
Dialect
+
Capabilities
```

---

# 24. Schema types

El Schema System deberá integrarse con:

```text
DATABASE_TYPE_SYSTEM
```

posteriormente documentado en el bloque 14.

Hasta entonces deberá mantener una abstracción compatible con:

```text
StringType
IntegerType
BigIntegerType
DecimalType
BooleanType
DateType
TimeType
DateTimeType
BinaryType
JsonType
UuidType
EnumType
CustomType
```

---

# 25. Platform-specific types

Tipos específicos podrán existir como extensión.

Ejemplo conceptual:

```text
PostgreSQL INET
PostgreSQL CIDR
PostgreSQL JSONB
MySQL SET
SQLite affinity-specific extension
```

pero no deberán contaminar el núcleo portable.

---

# 26. Portable schema

VoltStack deberá distinguir:

```text
PortableSchemaDefinition
```

de:

```text
PlatformSpecificSchemaDefinition
```

---

# 27. Portability level

Puede modelarse:

```php
enum SchemaPortabilityLevel
{
    case PORTABLE;
    case PORTABLE_WITH_DEGRADATION;
    case PLATFORM_SPECIFIC;
    case EXTENSION_SPECIFIC;
}
```

---

# 28. No silent degradation

Si una característica no puede representarse exactamente:

```text
requested semantics
≠
platform capability
```

VoltStack no deberá degradarla silenciosamente.

---

# 29. Capability system

Schema deberá consumir:

```text
PlatformCapabilitySnapshot
```

y producir requisitos tipados.

---

# 30. Schema capabilities

Ejemplos:

```text
CREATE_SCHEMA
ALTER_TABLE
DROP_COLUMN
RENAME_COLUMN
FOREIGN_KEYS
CHECK_CONSTRAINTS
GENERATED_COLUMNS
IDENTITY_COLUMNS
SEQUENCES
PARTIAL_INDEXES
EXPRESSION_INDEXES
CONCURRENT_INDEX_CREATION
DEFERRABLE_CONSTRAINTS
CASCADE_OPERATIONS
TRANSACTIONAL_DDL
RETURNING_DDL_METADATA
STRICT_TABLES
```

---

# 31. Capability ≠ vendor check

Nunca:

```php
if ($driver === 'pgsql') {
}
```

en componentes genéricos.

Preferible:

```php
if ($capabilities->supports(
    SchemaCapability::DEFERRABLE_CONSTRAINTS
)) {
}
```

---

# 32. Version ≠ capability

Una versión:

```text
PostgreSQL X
MySQL Y
SQLite Z
```

no deberá utilizarse directamente como sustituto universal de capacidades.

---

# 33. Capability snapshot

La compilación deberá recibir un snapshot inmutable:

```php
final readonly class SchemaCompilationContext
{
    public function __construct(
        public DatabasePlatform $platform,
        public SchemaCapabilitySnapshot $capabilities,
        public SchemaCompilationConfiguration $configuration,
        public SchemaExtensionSet $extensions,
    ) {}
}
```

---

# 34. No live capability discovery during compilation

Nunca:

```text
SchemaCompiler
→ query database version
→ inspect PRAGMA
→ inspect information_schema
```

durante compile.

La información deberá haberse resuelto previamente.

---

# 35. Schema introspection

`Schema Introspection` representa el flujo inverso:

```text
Existing Database
        ↓
Driver Catalog Access
        ↓
Platform Introspector
        ↓
Normalized Schema Metadata
        ↓
Schema Model
```

---

# 36. Introspection ≠ compilation

```text
SchemaCompiler
=
model → database representation
```

```text
SchemaIntrospector
=
database catalog → model
```

Son direcciones distintas.

---

# 37. Introspection boundary

La introspección sí requiere acceso real a la base de datos.

Pero deberá hacerlo mediante:

```text
Connection System
+
Driver Catalog Adapter
```

No mediante SQL arbitrario disperso por componentes.

---

# 38. Platform introspector

Cada plataforma podrá proporcionar:

```text
MySqlSchemaIntrospector
MariaDbSchemaIntrospector
PostgreSqlSchemaIntrospector
SqliteSchemaIntrospector
```

---

# 39. MariaDB first-class

MariaDB no deberá tratarse simplemente como:

```text
MySQL alias
```

Podrá reutilizar componentes comunes, pero tendrá:

```text
own capability profile
own introspection behavior
own compilation behavior
```

cuando corresponda.

---

# 40. SQLite first-class

SQLite requiere tratamiento explícito de:

```text
type affinity
ROWID
WITHOUT ROWID
STRICT
PRAGMA metadata
limited ALTER behavior
generated columns
foreign-key behavior
```

sin convertir el core Schema en código SQLite-specific.

---

# 41. PostgreSQL first-class

PostgreSQL deberá poder modelar:

```text
schemas/namespaces
sequences
identity
deferrable constraints
partial indexes
expression indexes
native enums
generated columns
```

mediante capacidades y extensiones.

---

# 42. MySQL first-class

MySQL deberá modelar correctamente:

```text
storage options
generated columns
index characteristics
charset/collation
auto increment
foreign keys
```

sin asumir que esas características existen igual en todas las plataformas.

---

# 43. Schema metadata

`Schema Metadata` representará información estructural conocida.

Ejemplo:

```text
TableMetadata
ColumnMetadata
IndexMetadata
ConstraintMetadata
ForeignKeyMetadata
```

---

# 44. Definition ≠ Metadata

Distinción:

```text
Definition
=
desired structure
```

```text
Metadata
=
observed / known structure
```

Aunque puedan transformarse entre sí.

---

# 45. Metadata certainty

La introspección puede producir diferentes niveles de certeza:

```php
enum SchemaMetadataCertainty
{
    case EXACT;
    case DRIVER_REPORTED;
    case INFERRED;
    case APPROXIMATE;
    case UNKNOWN;
}
```

---

# 46. Unknown metadata

`UNKNOWN` no deberá convertirse automáticamente en:

```text
false
```

Ejemplo:

```text
unknown whether index is clustered
```

no significa:

```text
index is not clustered
```

---

# 47. Schema snapshot

La estructura observada podrá encapsularse en:

```php
final readonly class SchemaSnapshot
{
    public function __construct(
        public SchemaSnapshotId $id,
        public DatabaseSchema $schema,
        public SchemaMetadataSet $metadata,
        public PlatformIdentity $platform,
        public SchemaCapabilitySnapshot $capabilities,
        public Instant $capturedAt,
    ) {}
}
```

---

# 48. Snapshot immutability

Un snapshot deberá ser inmutable.

```text
SchemaSnapshot(t₁)
```

no deberá mutar para convertirse en:

```text
SchemaSnapshot(t₂)
```

Se crea otro snapshot.

---

# 49. Schema diff

`Schema Diff` deberá comparar:

```text
Current Schema
```

contra:

```text
Target Schema
```

para obtener:

```text
SchemaDiff
```

---

# 50. Formalización

```text
Diff(Current, Target)
=
StructuralChanges
```

---

# 51. SchemaDiff ≠ Migration

Un diff describe diferencias.

Una migration describe una transición administrada.

```text
SchemaDiff
≠
Migration
```

---

# 52. Example diff

```text
Current
users
├── id
└── name

Target
users
├── id
├── name
└── email
```

produce:

```text
SchemaDiff
└── AddColumn(users.email)
```

---

# 53. Rename ambiguity

El Diff System no deberá asumir automáticamente:

```text
drop old column
+
add similar column
=
rename
```

Los renames pueden requerir:

```text
explicit hint
confidence model
developer confirmation
migration metadata
```

---

# 54. Destructive changes

Cambios como:

```text
DROP TABLE
DROP COLUMN
type narrowing
constraint tightening
```

deberán clasificarse como potencialmente destructivos.

---

# 55. Safety classification

```php
enum SchemaChangeSafety
{
    case SAFE;
    case CONDITIONALLY_SAFE;
    case POTENTIALLY_DESTRUCTIVE;
    case DESTRUCTIVE;
    case UNKNOWN;
}
```

---

# 56. Safety classification ≠ execution authorization

Que una operación sea:

```text
SAFE
```

no significa automáticamente:

```text
execute now
```

La autorización pertenece a capas superiores, especialmente Migrations.

---

# 57. Schema planning

Entre diff y compiler podrá existir una capa:

```text
Schema Change Planner
```

responsable de convertir cambios estructurales en operaciones ordenadas.

---

# 58. Why planning is necessary

Ejemplo:

```text
create table users
create table posts
add FK posts.user_id → users.id
```

requiere dependencias.

Otro ejemplo:

```text
drop foreign key
drop column
```

requiere orden correcto.

---

# 59. Schema plan

```php
final readonly class SchemaChangePlan
{
    public function __construct(
        public SchemaPlanId $id,
        public SchemaOperationGraph $operations,
        public SchemaPlanDependencySet $dependencies,
        public SchemaSafetySummary $safety,
    ) {}
}
```

---

# 60. Schema operation graph

```text
Create users
     │
     ▼
Create posts
     │
     ▼
Add FK posts.user_id → users.id
```

---

# 61. Graph edges

Podrán existir:

```text
STRUCTURAL_DEPENDENCY
OBJECT_EXISTENCE
REFERENCE_DEPENDENCY
ORDERING
SAFETY
TRANSACTION_BOUNDARY
CAPABILITY
EXTENSION
```

---

# 62. Schema planning ≠ Query planning

No deberán confundirse.

```text
Query Planner
=
plan data access/execution
```

```text
Schema Planner
=
plan structural transformation
```

---

# 63. Schema compilation

El compiler transforma:

```text
SchemaOperation
+
Platform
+
Dialect
+
Capabilities
```

en:

```text
CompiledDatabaseCommand
```

o una secuencia explícita de comandos cuando la operación necesariamente lo requiera.

---

# 64. One operation may require multiple commands

Algunas plataformas pueden requerir una estrategia estructural más compleja.

Ejemplo conceptual:

```text
AlterTable
```

podría necesitar:

```text
create replacement table
copy data
drop old table
rename replacement
recreate indexes
```

---

# 65. Compiler must not invent migrations

Sin embargo:

```text
complex schema transformation
```

no deberá aparecer como magia oculta del renderer.

Debe existir previamente como:

```text
SchemaChangePlan
```

o estrategia estructural explícita.

---

# 66. No hidden destructive emulation

Nunca:

```text
ALTER unsupported
↓
compiler silently rebuilds entire table
```

sin que esa estrategia sea visible y validada.

---

# 67. Schema compiler architecture

```text
SchemaChangePlan
        ↓
SchemaCompilerCoordinator
        ↓
SchemaOperationCompilerRegistry
        ↓
Platform Schema Compiler
        ↓
Schema SQL Generation
        ↓
CompiledDatabaseCommandSet
```

---

# 68. Compiler specializations

Podrán existir:

```text
CreateTableCompiler
AlterTableCompiler
DropTableCompiler
ColumnCompiler
IndexCompiler
ForeignKeyCompiler
ConstraintCompiler
```

---

# 69. No God SchemaCompiler

Debe evitarse:

```php
class SchemaCompiler
{
    public function compileEverything(...) {}
}
```

con miles de condicionales.

---

# 70. Platform compiler specialization

Arquitectura:

```text
Common Schema Compiler
        │
        ├── MySQL
        ├── MariaDB
        ├── PostgreSQL
        └── SQLite
```

Compartiendo únicamente lo realmente común.

---

# 71. Schema identifier model

Los nombres deberán representarse estructuralmente.

```php
final readonly class TableIdentifier
{
    public function __construct(
        public Identifier $table,
        public ?Identifier $schema = null,
        public ?Identifier $catalog = null,
    ) {}
}
```

---

# 72. Identifier ≠ raw SQL

Nunca:

```php
$table = 'public."users"';
```

como representación interna principal.

Preferible:

```text
catalog = null
schema = public
table = users
```

---

# 73. Identifier quoting

El quoting pertenece a:

```text
Dialect / Compiler
```

No al:

```text
Schema Model
```

---

# 74. Identifier normalization

Las plataformas difieren en:

```text
case folding
case sensitivity
quoted identifiers
maximum lengths
reserved words
```

Estas diferencias deberán ser modeladas por Platform/Dialect.

---

# 75. Object identity

Debe distinguirse:

```text
TableIdentifier
≠
TableId
```

El primero identifica una estructura DB.

El segundo puede ser una identidad interna del modelo/AST.

---

# 76. Column identity

Igualmente:

```text
ColumnIdentifier
≠
ColumnNodeId
≠
ResultColumnId
≠
ORM PropertyId
```

---

# 77. Defaults

Los defaults deberán modelarse explícitamente.

Ejemplos:

```text
literal default
NULL default
CURRENT_TIMESTAMP
database expression
sequence/identity generated
platform-specific default
```

---

# 78. DefaultValue

```php
interface DefaultValue
{
}
```

Posibles implementaciones:

```text
LiteralDefault
NullDefault
CurrentTimestampDefault
ExpressionDefault
PlatformDefault
ExtensionDefault
```

---

# 79. Runtime values ≠ schema defaults

Un default estructural:

```sql
DEFAULT CURRENT_TIMESTAMP
```

no es un runtime query parameter.

---

# 80. Schema expressions

Algunas estructuras necesitan expresiones:

```text
CHECK
generated column
expression index
default expression
partial index predicate
```

---

# 81. SchemaExpression

Deberá existir una representación tipada.

Podrá reutilizar conceptos comunes de Expression System sólo si la semántica coincide.

---

# 82. Query expression ≠ schema expression automatically

Ejemplo:

```text
WHERE expression
```

y:

```text
CHECK constraint expression
```

pueden compartir nodos básicos, pero tienen:

```text
different legal contexts
different capability requirements
different volatility restrictions
```

---

# 83. Constraint architecture

Una tabla puede contener:

```text
PrimaryKeyConstraint
UniqueConstraint
CheckConstraint
ForeignKeyConstraint
ExtensionConstraint
```

---

# 84. Index ≠ constraint

Distinción esencial:

```text
UniqueIndex
≠
UniqueConstraint
```

aunque una plataforma pueda implementar uno mediante el otro.

El modelo deberá preservar la intención semántica.

---

# 85. Primary key ≠ index

Igualmente:

```text
PrimaryKeyConstraint
≠
Index
```

aunque físicamente exista un índice asociado.

---

# 86. Foreign key

Una FK deberá representar:

```text
source columns
target table
target columns
on update
on delete
deferrability
match semantics
name
```

cuando sean aplicables.

---

# 87. Referential action

```php
enum ReferentialAction
{
    case NO_ACTION;
    case RESTRICT;
    case CASCADE;
    case SET_NULL;
    case SET_DEFAULT;
}
```

---

# 88. Referential action capability

No todas las plataformas ofrecen exactamente la misma semántica.

El compiler deberá validar capacidades.

---

# 89. Index model

Un índice podrá contener:

```text
columns
expressions
ordering
prefix lengths
predicate
include columns
uniqueness
method
options
```

pero cada feature tendrá capability requirements.

---

# 90. Common denominator problem

VoltStack no deberá limitar todo Schema al mínimo común denominador.

En lugar de:

```text
only features supported everywhere
```

usará:

```text
portable core
+
capability-aware advanced features
+
typed extensions
```

---

# 91. Progressive portability

La estrategia será:

```text
Level 1
Portable core

Level 2
Capability-gated standard features

Level 3
Platform-specific typed features

Level 4
Extension-defined features

Level 5
Explicit raw schema escape hatch
```

---

# 92. Raw schema escape hatch

Podrá existir un mecanismo avanzado:

```php
$schema->raw(...);
```

pero deberá ser:

```text
explicit
trusted
platform-scoped
non-portable
security-sensitive
diagnostically visible
```

---

# 93. Raw schema operation

Nunca deberá confundirse con:

```text
normal Schema AST
```

---

# 94. Raw DDL and migrations

Las migrations podrán usar raw DDL cuando sea imprescindible.

Pero VoltStack deberá marcar:

```text
portability unknown
rollback unknown
diff visibility limited
safety unknown
```

cuando corresponda.

---

# 95. Schema validation

Antes de compilar deberá validarse:

```text
identifier validity
duplicate columns
duplicate constraints
duplicate indexes
missing referenced columns
foreign-key arity
type compatibility
default compatibility
nullability rules
capability requirements
dependency validity
extension validity
```

---

# 96. Validation layers

Propuesta:

```text
Structural Validation
        ↓
Semantic Schema Validation
        ↓
Cross-Object Validation
        ↓
Capability Validation
        ↓
Platform Validation
        ↓
Compilation Validation
```

---

# 97. Structural validation

Ejemplos:

```text
table must have name
column must have type
foreign key must reference columns
```

---

# 98. Semantic validation

Ejemplo:

```text
SET NULL
```

sobre una columna:

```text
NOT NULL
```

puede representar una incompatibilidad semántica.

---

# 99. Cross-object validation

Ejemplo:

```text
FK posts.user_id
→ users.id
```

requiere resolver:

```text
posts.user_id
users.id
```

---

# 100. Capability validation

Ejemplo:

```text
DEFERRABLE
```

requiere:

```text
SchemaCapability::DEFERRABLE_CONSTRAINTS
```

---

# 101. Platform validation

Puede verificar restricciones como:

```text
maximum identifier length
legal type combinations
platform-specific index restrictions
```

---

# 102. Validation ≠ introspection

El validator no deberá abrir conexiones para descubrir estructuras.

Cuando requiera estado existente deberá recibir:

```text
SchemaSnapshot
```

explícitamente.

---

# 103. Determinism

Dados:

```text
same Schema Model
same Schema Operation
same Platform
same Capability Snapshot
same Compiler Configuration
same Extension Set
```

deberá producirse:

```text
same canonical compiled representation
```

---

# 104. Schema fingerprint

Los artefactos podrán tener:

```text
SchemaFingerprint
```

---

# 105. Fingerprint inputs

No deberá ser sólo:

```text
hash(DDL string)
```

Deberá considerar:

```text
schema model version
operation
platform
dialect
capability snapshot
compiler version
extension set
relevant options
```

---

# 106. Schema identity vs fingerprint

```text
SchemaObjectId
≠
SchemaFingerprint
```

Uno identifica.

El otro representa contenido/configuración estructural.

---

# 107. Canonicalization

Para comparar schemas será necesaria una representación canónica.

Ejemplo:

```text
index option ordering
constraint ordering
metadata ordering
```

no deberá producir falsos diffs.

---

# 108. Canonicalization must preserve semantics

Nunca:

```text
canonicalize
→ remove meaningful ordering
```

si la plataforma considera ese orden semánticamente relevante.

---

# 109. Schema comparison

Deberá distinguir:

```text
exact equality
structural equality
semantic equivalence
platform equivalence
portable equivalence
```

---

# 110. Equality levels

Ejemplo:

```php
enum SchemaComparisonMode
{
    case EXACT;
    case STRUCTURAL;
    case SEMANTIC;
    case PLATFORM_NORMALIZED;
}
```

---

# 111. Metadata noise

La introspección puede producir metadata no relevante para migration diff.

Ejemplo:

```text
physical storage metadata
statistics
internal IDs
```

Debe poder excluirse de comparaciones estructurales.

---

# 112. Schema dependencies

Objetos estructurales forman un grafo.

```text
Table
├── Columns
├── Indexes
├── Constraints
└── Foreign Keys
```

y entre tablas:

```text
posts
   │
   └── FK
        ↓
      users
```

---

# 113. Dependency graph

```php
final readonly class SchemaDependencyGraph
{
    // immutable structural dependency graph
}
```

---

# 114. Dependency use cases

Servirá para:

```text
creation ordering
drop ordering
migration planning
diff planning
cycle detection
diagnostics
```

---

# 115. Cyclic foreign keys

Ejemplo:

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

El planner deberá resolverlo explícitamente.

---

# 116. No recursive magic

El compiler no deberá descubrir este orden mientras genera strings SQL.

---

# 117. Transactional DDL

Algunas plataformas soportan distintos niveles de:

```text
transactional DDL
```

Esto deberá expresarse mediante capabilities.

---

# 118. DDLTransactionCapability

Conceptualmente:

```php
enum DdlTransactionCapability
{
    case FULL;
    case PARTIAL;
    case IMPLICIT_COMMIT;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 119. Schema compiler does not manage transactions

El compiler sólo declara:

```text
transaction requirements
```

La ejecución pertenece a:

```text
Transaction Manager
+
Execution Engine
```

---

# 120. Schema plan transaction metadata

```text
SchemaOperation
    ↓
TransactionRequirement
```

puede indicar:

```text
TRANSACTION_ALLOWED
TRANSACTION_REQUIRED
TRANSACTION_FORBIDDEN
IMPLICIT_COMMIT_POSSIBLE
UNKNOWN
```

---

# 121. Schema execution result

Las operaciones estructurales podrán producir:

```text
NoResult
AffectedMetadata
PlatformSpecificResult
```

pero no deberán inventar row results si la plataforma no los ofrece.

---

# 122. Schema result ≠ introspection

Después de:

```text
CREATE TABLE
```

VoltStack no deberá ejecutar automáticamente:

```text
introspect table
```

salvo que una capa superior lo solicite.

---

# 123. Schema cache

Podrán cachearse:

```text
immutable schema metadata
schema snapshots
compiled schema artifacts
```

con invalidación adecuada.

Pero no será responsabilidad principal de este documento.

---

# 124. Introspection cache

La introspección cacheada deberá distinguir:

```text
observed at t₁
```

de:

```text
current database state
```

Nunca deberá presentarse un snapshot viejo como certeza actual.

---

# 125. Persistent runtime model

VoltStack operará principalmente bajo:

```text
FrankenPHP
```

con futuros adaptadores:

```text
RoadRunner
OpenSwoole
```

Por tanto Schema deberá ser persistent-worker safe.

---

# 126. Shared state allowed

Podrán compartirse:

```text
immutable Schema Compiler Registry
immutable capability definitions
frozen type registry
frozen extension registry
stateless validators
immutable configuration
```

---

# 127. Operation-scoped state

Deberán permanecer aislados:

```text
SchemaBuilder session
SchemaIntrospection session
SchemaDiff session
SchemaCompilation session
SchemaChangePlan
connection lease
transaction context
tenant context
diagnostics
temporary buffers
```

---

# 128. Forbidden persistent state

Nunca:

```php
static $currentSchema;
static $currentConnection;
static $currentTenant;
static $currentMigration;
```

---

# 129. Tenant isolation

Aunque Multitenancy sea un paquete opcional, Schema deberá permitir recibir explícitamente:

```text
SchemaExecutionContext
```

con información suficiente para que la integración futura pueda seleccionar:

```text
tenant database
tenant schema
tenant connection
```

sin introducir dependencia obligatoria hacia Multitenancy.

---

# 130. Core does not depend on Multitenancy

```text
Database Schema
        X
VoltStack/Quantum/Multitenancy
```

No habrá dependencia core obligatoria.

La integración posterior será:

```text
Multitenancy package
        ↓
Database extension/integration contracts
```

---

# 131. Tenant schema operations

Una migration tenant-aware será responsabilidad de:

```text
Multitenancy Integration
+
Migration System
```

No del Schema Model base.

---

# 132. Security model

Schema operations son especialmente sensibles porque pueden:

```text
destroy data
alter constraints
remove security boundaries
expose columns
change database behavior
```

---

# 133. Security principles

El sistema deberá aplicar:

```text
structured identifiers
no runtime string concatenation
explicit raw DDL
capability validation
operation classification
destructive-change metadata
sensitive metadata redaction
auditable operations
```

---

# 134. Identifier injection prevention

Nunca:

```php
$sql = "DROP TABLE {$userInput}";
```

El identificador deberá pasar por:

```text
Identifier Model
↓
Validation
↓
Dialect Quoting
```

---

# 135. Values vs identifiers

Igual que en Query:

```text
runtime value
≠
database identifier
```

Un nombre de tabla no se trata como un binding parameter normal.

---

# 136. Dynamic identifiers

Cuando una aplicación necesite nombres dinámicos:

```text
tenant_123
archive_2026
```

deberán pasar por políticas explícitas de identifier validation.

---

# 137. Destructive operation policy

El Schema System deberá etiquetar:

```text
DropTable
DropColumn
NarrowColumn
DropConstraint
```

con metadata de riesgo.

Pero la decisión final:

```text
allow in production?
```

pertenecerá al Migration Safety System.

---

# 138. Sensitive schema metadata

Nombres de tablas/columnas pueden revelar información.

Telemetry deberá evitar exportar indiscriminadamente:

```text
full schema dumps
DDL
comments
default values
```

---

# 139. Telemetry

Eventos sugeridos:

```text
SchemaBuildStarted
SchemaBuildCompleted
SchemaValidationFailed
SchemaIntrospectionStarted
SchemaIntrospectionCompleted
SchemaDiffComputed
SchemaPlanCreated
SchemaCompilationStarted
SchemaCompilationCompleted
SchemaCompilationFailed
```

---

# 140. Metrics

Ejemplos:

```text
db.schema.introspection.duration
db.schema.introspection.objects
db.schema.diff.operations
db.schema.compile.duration
db.schema.compile.operations
db.schema.validation.failures
```

---

# 141. Cardinality control

No utilizar como labels:

```text
table name
column name
SQL
schema name
tenant id
```

por defecto.

---

# 142. Debug diagnostics

Los diagnósticos sí podrán incluir nombres estructurales cuando la política lo permita:

```text
Foreign key "posts_user_id_fk" references missing column "users.idd".
```

pero deberán respetar:

```text
security
sensitivity
debug profile
```

---

# 143. Error architecture

Propuesta:

```text
DatabaseSchemaException
├── SchemaDefinitionException
├── SchemaValidationException
├── SchemaCapabilityException
├── SchemaIntrospectionException
├── SchemaMetadataException
├── SchemaDiffException
├── SchemaPlanningException
├── SchemaCompilationException
├── SchemaDependencyException
├── SchemaPortabilityException
├── SchemaSecurityException
├── SchemaExtensionException
└── SchemaInvariantException
```

---

# 144. Definition errors

Ejemplo:

```text
duplicate column "email"
```

---

# 145. Capability errors

Ejemplo:

```text
Deferrable foreign keys are not supported by the selected platform capability profile.
```

---

# 146. Diff errors

Ejemplo:

```text
Potential rename ambiguity:
old column "username"
new column "user_name"
```

---

# 147. Compilation errors

Ejemplo:

```text
Schema operation cannot be represented exactly for current platform capabilities.
```

---

# 148. Introspection errors

Deberán preservar:

```text
native driver cause
catalog operation
platform
resource impact
```

sin filtrar credenciales.

---

# 149. Extension architecture

Schema deberá ser extensible.

Posibles contribuciones:

```text
custom schema object
custom type
custom constraint
custom index option
custom schema expression
custom introspector
custom compiler
custom validator
custom metadata provider
```

---

# 150. Typed extensions

No:

```php
$schema->macro('whatever', ...);
```

como mecanismo arquitectónico principal.

Preferible:

```text
SchemaExtensionDescriptor
+
typed extension IDs
+
declared capabilities
+
deterministic registry
```

---

# 151. Extension lifecycle

```text
discover
↓
validate descriptor
↓
resolve dependencies
↓
validate platform compatibility
↓
detect conflicts
↓
deterministic ordering
↓
compose registries
↓
freeze
```

---

# 152. No last-wins

Dos extensiones que registren el mismo:

```text
SchemaCompilerId
```

deberán producir:

```text
conflict
```

No:

```text
last registered wins
```

---

# 153. Extension safety

Una extensión no deberá:

```text
open arbitrary connections during compilation
execute DDL during validation
mutate global registries
silently replace core compiler
bypass identifier security
bypass capability validation
hide destructive operations
```

---

# 154. Schema versioning

Los modelos internos deberán tener versiones explícitas.

Ejemplo:

```text
SchemaModelVersion
SchemaAstVersion
SchemaCompilerVersion
SchemaSnapshotVersion
```

---

# 155. Serialization

Artefactos inmutables podrán serializarse para:

```text
cache
testing
diagnostics
migration planning
schema snapshots
```

siempre mediante formato versionado.

---

# 156. Serialized snapshot ≠ current database

Un snapshot serializado representa:

```text
database structure observed at time T
```

No prueba el estado actual.

---

# 157. Concurrency

Dos procesos pueden modificar schema simultáneamente.

Por tanto:

```text
SchemaSnapshot
→ Plan
→ Execute
```

puede sufrir:

```text
schema drift
```

---

# 158. Schema drift

Definición:

```text
ObservedSchema(t₁)
≠
ActualSchema(t₂)
```

antes de ejecutar el plan.

---

# 159. Drift detection

El Migration/Schema execution layer podrá utilizar:

```text
expected schema fingerprint
```

para detectar cambios concurrentes.

---

# 160. Optimistic schema validation

Conceptualmente:

```text
ExpectedCurrentFingerprint
=
ActualCurrentFingerprint
```

antes de ejecutar operaciones críticas.

---

# 161. No universal locking assumption

VoltStack no asumirá que existe:

```text
portable global schema lock
```

entre todas las plataformas.

---

# 162. Concurrency policy

Será capability-driven y posteriormente integrada con:

```text
Migration System
Transaction System
Locking System
```

---

# 163. Schema lifecycle

```text
Definition
    ↓
Validation
    ↓
Planning
    ↓
Compilation
    ↓
Execution
    ↓
Optional Introspection
    ↓
New Snapshot
```

---

# 164. Introspection lifecycle

```text
Connection
    ↓
Catalog Adapter
    ↓
Native Metadata
    ↓
Normalization
    ↓
Validation
    ↓
Schema Metadata
    ↓
Schema Model
    ↓
Schema Snapshot
```

---

# 165. Diff lifecycle

```text
Current Schema Snapshot
        +
Target Schema Definition
        ↓
Normalization
        ↓
Comparison
        ↓
Difference Detection
        ↓
Rename Analysis
        ↓
Safety Classification
        ↓
SchemaDiff
```

---

# 166. Planning lifecycle

```text
SchemaDiff
    ↓
Change Expansion
    ↓
Dependency Graph
    ↓
Capability Analysis
    ↓
Safety Analysis
    ↓
Ordering
    ↓
Transaction Requirements
    ↓
SchemaChangePlan
```

---

# 167. Compilation lifecycle

```text
SchemaChangePlan
    ↓
Operation Validation
    ↓
Compiler Resolution
    ↓
Platform Adaptation
    ↓
DDL Representation
    ↓
Command Generation
    ↓
Dependency/Fingerprint Metadata
    ↓
CompiledDatabaseCommandSet
```

---

# 168. Execution lifecycle

```text
CompiledDatabaseCommandSet
        ↓
Execution Engine
        ↓
Connection Manager
        ↓
Transaction Coordination
        ↓
Statement Execution
        ↓
Driver
```

---

# 169. Schema facade

La futura API podrá proporcionar:

```php
Schema::create(...);
Schema::table(...);
Schema::drop(...);
Schema::hasTable(...);
Schema::hasColumn(...);
```

pero la Facade será sólo:

```text
developer convenience layer
```

---

# 170. Facade ≠ architecture

Nunca:

```text
Schema Facade
→ platform SQL
```

La ruta deberá seguir:

```text
Facade
→ Builder/Introspection API
→ Schema subsystems
```

---

# 171. hasTable()

Una consulta:

```php
Schema::hasTable('users');
```

pertenece conceptualmente a:

```text
Schema Introspection
```

No a Schema Builder.

---

# 172. hasColumn()

Igualmente:

```php
Schema::hasColumn('users', 'email');
```

deberá consultar metadata/introspection.

---

# 173. Blueprint naming

VoltStack puede utilizar una API similar a:

```text
Blueprint
```

pero arquitectónicamente es preferible distinguir:

```text
TableDefinitionBuilder
```

de:

```text
TableDefinition
```

---

# 174. Mutable builder vs immutable artifact

Regla recomendada:

```text
Builder
=
mutable operation-local convenience object
```

```text
Definition
=
immutable structural artifact
```

---

# 175. Freeze boundary

```text
Mutable Builder
      ↓
freeze()
      ↓
Immutable Definition
```

---

# 176. Builder lifetime

Un builder nunca deberá almacenarse como estado global persistente.

---

# 177. Builder reuse

Después de finalizar:

```text
freeze
```

la instancia deberá:

```text
reject mutation
```

o dejar de exponerse.

---

# 178. Fluent API

Podrá existir:

```php
$table
    ->string('email')
    ->length(255)
    ->nullable(false);
```

pero internamente deberá producir metadata tipada.

---

# 179. Convenience methods

Ejemplos:

```php
$table->id();
$table->timestamps();
$table->softDeletes();
$table->foreignId('user_id');
```

deberán expandirse a primitivas estructurales conocidas.

---

# 180. Convenience macro ≠ primitive

Ejemplo:

```text
timestamps()
```

puede expandirse a:

```text
created_at
updated_at
```

pero no debe convertirse en una primitiva del Schema AST si no aporta semántica estructural independiente.

---

# 181. Expansion phase

Convenience DSL:

```text
Developer shorthand
↓
Definition expansion
↓
Canonical Schema Definition
```

antes de planificación/compilación.

---

# 182. Soft deletes

`softDeletes()` no pertenece realmente al motor DDL como concepto universal.

Es:

```text
convenience schema convention
```

que crea una columna.

El comportamiento de soft delete pertenece posteriormente a:

```text
269_DATABASE_SOFT_DELETE_SYSTEM.md
```

---

# 183. Timestamps

Igualmente:

```text
timestamps()
```

es una convención de schema, no una capacidad especial del DB core.

---

# 184. Foreign ID

```php
$table->foreignId('user_id');
```

puede producir:

```text
column definition
```

pero:

```php
->constrained()
```

produce adicionalmente:

```text
foreign key definition
```

Deben seguir siendo objetos separados internamente.

---

# 185. Naming conventions

Nombres automáticos como:

```text
users_email_unique
posts_user_id_foreign
```

deberán generarse mediante un:

```text
SchemaNamingStrategy
```

---

# 186. Naming strategy

```php
interface SchemaNamingStrategy
{
    public function indexName(...): IndexIdentifier;

    public function foreignKeyName(...): ConstraintIdentifier;

    public function uniqueConstraintName(...): ConstraintIdentifier;
}
```

---

# 187. Deterministic naming

La misma definición deberá producir el mismo nombre automático.

---

# 188. Identifier limits

Si una plataforma limita nombres:

```text
63 chars
64 chars
etc.
```

la estrategia deberá generar nombres:

```text
deterministic
collision-resistant
platform-compatible
```

---

# 189. Truncation

Nunca:

```text
truncate blindly
```

si puede causar colisiones.

Preferible:

```text
stable prefix
+
short deterministic fingerprint
```

---

# 190. Comments

Schema puede modelar:

```text
table comments
column comments
```

como metadata estructural cuando la plataforma lo soporte.

---

# 191. Comment portability

No todas las plataformas representan comentarios de igual forma.

Por tanto:

```text
comment
→ capability requirement
```

---

# 192. Charset and collation

Deberán modelarse como opciones tipadas.

No como strings SQL arbitrarios.

---

# 193. Collation semantics

Debe distinguirse:

```text
logical collation request
```

de:

```text
platform collation identifier
```

cuando sea necesario.

---

# 194. Storage engine

Conceptos como:

```text
InnoDB
```

son platform-specific.

No deberán aparecer como propiedad universal obligatoria de `TableDefinition`.

---

# 195. TableOptionSet

La arquitectura podrá usar:

```php
final readonly class TableOptionSet
{
    // typed portable and platform-specific options
}
```

---

# 196. Option namespaces

Ejemplo:

```text
portable.*
mysql.*
mariadb.*
postgresql.*
sqlite.*
extension.*
```

conceptualmente, aunque la implementación final puede usar objetos tipados en lugar de strings.

---

# 197. No option bag of mixed strings

Evitar:

```php
$options = [
    'engine' => 'InnoDB',
    'whatever' => 'foo',
];
```

como arquitectura central.

---

# 198. Generated columns

Deberán modelarse mediante:

```text
expression
generation kind
storage behavior
capability requirement
```

---

# 199. Identity/autoincrement

Debe distinguirse:

```text
logical generated identifier
```

de mecanismos específicos:

```text
AUTO_INCREMENT
IDENTITY
SEQUENCE
ROWID
```

---

# 200. Auto increment ≠ universal primitive

El Schema Model deberá evitar hacer de:

```text
AUTO_INCREMENT
```

la semántica universal.

Preferible:

```text
ValueGenerationStrategy
```

---

# 201. ValueGenerationStrategy

```php
enum ValueGenerationStrategy
{
    case NONE;
    case IDENTITY;
    case SEQUENCE;
    case PLATFORM_NATIVE;
    case GENERATED_EXPRESSION;
    case EXTENSION;
}
```

---

# 202. Sequence model

Las plataformas con secuencias podrán representarlas explícitamente:

```text
SequenceDefinition
```

sin obligar a SQLite/MySQL a fingir la misma capacidad.

---

# 203. Views

Aunque el bloque inicial se centra en tablas, la arquitectura deberá permitir evolución hacia:

```text
ViewDefinition
MaterializedViewDefinition
```

como capacidades/extensiones.

---

# 204. Triggers

Los triggers no deberán introducirse improvisadamente dentro de `TableDefinition`.

Podrán ser:

```text
advanced schema objects
```

en una extensión o futura especificación.

---

# 205. Procedures/functions

Igualmente, no son parte necesaria del Schema V1 portable.

---

# 206. Schema object model

Arquitectura extensible:

```text
SchemaObject
├── Table
├── Sequence
├── View
├── MaterializedView
├── NativeType
└── ExtensionObject
```

---

# 207. Schema namespace

En PostgreSQL:

```text
schema
```

es un namespace real.

En otros motores su significado puede variar.

VoltStack no deberá asumir:

```text
schema = database
```

universalmente.

---

# 208. Catalog vs schema vs database

Deben modelarse separadamente:

```text
Catalog
Schema/Namespace
Database Connection Target
```

cuando la plataforma los distinga.

---

# 209. Qualified object names

Representación:

```text
catalog.schema.table
```

deberá construirse estructuralmente.

---

# 210. Current schema

No deberá existir:

```php
static $currentSchema;
```

La resolución de namespace actual deberá provenir del:

```text
Connection/Schema Context
```

---

# 211. SchemaContext

```php
final readonly class SchemaContext
{
    public function __construct(
        public ConnectionReference $connection,
        public ?CatalogIdentifier $catalog,
        public ?SchemaIdentifier $namespace,
        public SchemaCapabilitySnapshot $capabilities,
    ) {}
}
```

---

# 212. Context ≠ service locator

`SchemaContext` no contendrá:

```text
container
HTTP request
ORM
cache service
event dispatcher
```

como bolsa arbitraria.

---

# 213. Resource governance

Schema operations pueden ser costosas.

Ejemplos:

```text
introspecting thousands of tables
rebuilding large tables
creating indexes
large schema diff
```

---

# 214. Schema budgets

Podrán existir:

```text
max introspected objects
max metadata bytes
max diff operations
max dependency nodes
max compiler operations
max generated commands
max diagnostics
max planning time
```

---

# 215. Budget exhaustion

Nunca deberá producir:

```text
partial schema diff presented as complete
```

sin marcarlo explícitamente.

---

# 216. Completeness

La introspección deberá poder representar:

```php
enum SchemaSnapshotCompleteness
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 217. Partial introspection

Un snapshot parcial no podrá utilizarse automáticamente como base segura para:

```text
destructive diff
```

---

# 218. Safety formula

```text
DestructiveDiffAllowed
⇒
CurrentSnapshotCompleteness = COMPLETE
```

salvo override explícito de una capa superior.

---

# 219. Cancellation

Introspection, diff, planning y compilation deberán poder cooperar con:

```text
CancellationToken
```

cuando la operación pueda ser larga.

---

# 220. Timeout

La introspección real podrá estar sujeta a:

```text
Deadline
```

mediante el Execution/Connection layer.

---

# 221. Compiler timeout

La compilación es CPU-local y deberá usar:

```text
CompilationBudget
```

más que DB timeout.

---

# 222. Failure atomicity

Los artefactos:

```text
SchemaDiff
SchemaChangePlan
CompiledSchemaCommandSet
```

deberán publicarse sólo cuando estén válidos.

---

# 223. No half-valid artifact

Nunca:

```text
compiled first 7 operations
operation 8 fails
→ return first 7 as successful plan
```

salvo API explícita de diagnóstico parcial.

---

# 224. Execution atomicity

Eso no significa que todas las plataformas puedan ejecutar DDL atómicamente.

La atomicidad de ejecución es otra preocupación.

---

# 225. Compilation atomicity ≠ execution atomicity

```text
CompilationAtomicity
≠
DatabaseTransactionAtomicity
```

---

# 226. Testing strategy

El Schema Architecture deberá probarse en tres niveles:

```text
portable model tests
platform compiler/introspection tests
cross-driver conformance tests
```

---

# 227. Portable model tests

Sin base de datos real:

```text
definitions
validation
AST
diff
dependency graph
planning
canonicalization
fingerprints
```

---

# 228. Platform tests

Con motores reales:

```text
compile
execute
introspect
compare
```

---

# 229. Round-trip test

Una prueba fundamental:

```text
Definition
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

deberá producir equivalencia estructural dentro de las capacidades soportadas.

---

# 230. Round-trip formula

```text
Normalize(
    Introspect(
        Execute(
            Compile(D)
        )
    )
)
≈
Normalize(D)
```

donde:

```text
≈
```

representa equivalencia estructural compatible con la plataforma.

---

# 231. Cross-platform tests

La misma definición portable deberá probarse sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 232. Capability conformance tests

Si una plataforma declara:

```text
supports feature F
```

debe existir una prueba que verifique:

```text
compile F
execute F
introspect F
```

cuando sea viable.

---

# 233. Unsupported capability tests

También deberá probarse:

```text
unsupported F
→ explicit SchemaCapabilityException
```

No SQL inválido enviado al servidor.

---

# 234. Diff conformance

Debe verificarse:

```text
Current
→ Target
→ Diff
→ Plan
→ Execute
→ Introspect
→ TargetEquivalent
```

---

# 235. Persistent-runtime tests

En FrankenPHP-style worker:

```text
Request A
→ Schema operation X

Request B
→ Schema operation Y
```

Y deberá cumplirse:

```text
MutableState(A)
∩
MutableState(B)
=
∅
```

---

# 236. Concurrent compilation tests

Dos schemas compilándose simultáneamente no deberán compartir:

```text
aliases
temporary buffers
operation lists
diagnostics
platform state
tenant state
```

---

# 237. Introspection isolation tests

Dos conexiones/platforms simultáneas deberán mantener metadata separada.

---

# 238. Error injection tests

Deberán cubrir:

```text
catalog query failure
connection loss
unsupported feature
invalid metadata
partial introspection
compiler extension failure
dependency cycle
budget exhaustion
cancellation
```

---

# 239. Performance model

El Schema Model no estará normalmente en el hot path de cada request.

Aun así debe ser eficiente para:

```text
large enterprise schemas
migration generation
developer tooling
CI
schema diff
```

---

# 240. Complexity targets

Para `n` objetos estructurales:

```text
basic traversal ≈ O(n)
```

Dependency planning:

```text
O(V + E)
```

para operaciones topológicas normales.

Diff dependerá de índices estructurales adecuados para evitar:

```text
O(n²)
```

innecesario.

---

# 241. Schema indexes

Internamente:

```text
TableIdentifier → TableDefinition
ColumnIdentifier → ColumnDefinition
ConstraintIdentifier → ConstraintDefinition
```

podrán utilizar mapas inmutables.

---

# 242. Large schemas

La arquitectura deberá soportar:

```text
thousands of tables
tens of thousands of columns
large FK graphs
```

sin depender de búsquedas lineales repetidas.

---

# 243. Lazy metadata

La introspección podrá permitir carga selectiva cuando la API lo solicite.

Pero:

```text
lazy
```

no deberá ocultar consultas inesperadas dentro de operaciones que prometen ser puras.

---

# 244. Explicit introspection scope

Ejemplo:

```php
$scope = SchemaIntrospectionScope::tables([
    'users',
    'posts',
]);
```

---

# 245. Full vs scoped introspection

```text
FullSchemaIntrospection
≠
ScopedSchemaIntrospection
```

---

# 246. Scoped snapshot

Un snapshot scoped deberá marcarse:

```text
PARTIAL
```

respecto al database completo.

---

# 247. Schema API levels

La arquitectura podrá ofrecer:

```text
Level 1 — Facade / Fluent Builder
Level 2 — Typed Definitions
Level 3 — Schema AST
Level 4 — Schema Diff / Planner
Level 5 — Platform Extensions
```

---

# 248. Public API principle

El 90% de desarrolladores debería poder usar:

```php
Schema::create(...)
```

sin conocer:

```text
Schema AST
Dependency Graph
Compiler Registry
Capability Snapshot
```

---

# 249. Internal rigor

La simplicidad externa deberá descansar sobre:

```text
strong internal boundaries
```

no sobre lógica implícita.

---

# 250. Proposed namespace

```text
VoltStack\Quantum\Database\Schema
```

---

# 251. Proposed directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Schema/
            ├── Contract/
            │   ├── SchemaOperation.php
            │   ├── SchemaCompiler.php
            │   ├── SchemaIntrospector.php
            │   ├── SchemaValidator.php
            │   └── SchemaExtension.php
            │
            ├── Model/
            │   ├── DatabaseSchema.php
            │   ├── SchemaObject.php
            │   ├── Table/
            │   ├── Column/
            │   ├── Index/
            │   ├── Constraint/
            │   ├── ForeignKey/
            │   ├── Sequence/
            │   └── Identifier/
            │
            ├── AST/
            │   ├── SchemaNode.php
            │   ├── Table/
            │   ├── Column/
            │   ├── Index/
            │   ├── Constraint/
            │   └── Extension/
            │
            ├── Builder/
            │   ├── SchemaBuilder.php
            │   ├── TableDefinitionBuilder.php
            │   ├── ColumnDefinitionBuilder.php
            │   ├── IndexDefinitionBuilder.php
            │   └── ConstraintDefinitionBuilder.php
            │
            ├── Definition/
            │   ├── TableDefinition.php
            │   ├── ColumnDefinition.php
            │   ├── IndexDefinition.php
            │   ├── ForeignKeyDefinition.php
            │   └── ConstraintDefinition.php
            │
            ├── Validation/
            │   ├── SchemaValidationPipeline.php
            │   ├── StructuralSchemaValidator.php
            │   ├── SemanticSchemaValidator.php
            │   ├── CapabilitySchemaValidator.php
            │   └── PlatformSchemaValidator.php
            │
            ├── Capability/
            │   ├── SchemaCapability.php
            │   ├── SchemaCapabilitySet.php
            │   ├── SchemaCapabilitySnapshot.php
            │   └── SchemaCapabilityRequirement.php
            │
            ├── Introspection/
            │   ├── SchemaIntrospectionManager.php
            │   ├── SchemaIntrospectionScope.php
            │   ├── SchemaSnapshot.php
            │   ├── MySQL/
            │   ├── MariaDB/
            │   ├── PostgreSQL/
            │   └── SQLite/
            │
            ├── Metadata/
            │   ├── SchemaMetadata.php
            │   ├── TableMetadata.php
            │   ├── ColumnMetadata.php
            │   ├── IndexMetadata.php
            │   └── ConstraintMetadata.php
            │
            ├── Diff/
            │   ├── SchemaDiffer.php
            │   ├── SchemaDiff.php
            │   ├── SchemaChange.php
            │   ├── SchemaChangeSafety.php
            │   └── RenameAnalyzer.php
            │
            ├── Planning/
            │   ├── SchemaChangePlanner.php
            │   ├── SchemaChangePlan.php
            │   ├── SchemaOperationGraph.php
            │   └── SchemaDependencyGraph.php
            │
            ├── Compiler/
            │   ├── SchemaCompilerCoordinator.php
            │   ├── SchemaCompilationContext.php
            │   ├── SchemaOperationCompilerRegistry.php
            │   ├── Common/
            │   ├── MySQL/
            │   ├── MariaDB/
            │   ├── PostgreSQL/
            │   └── SQLite/
            │
            ├── Naming/
            │   ├── SchemaNamingStrategy.php
            │   └── DefaultSchemaNamingStrategy.php
            │
            ├── Comparison/
            │   ├── SchemaComparator.php
            │   ├── SchemaComparisonMode.php
            │   └── SchemaCanonicalizer.php
            │
            ├── Context/
            │   └── SchemaContext.php
            │
            ├── Extension/
            │   ├── SchemaExtensionDescriptor.php
            │   └── SchemaExtensionRegistry.php
            │
            ├── Telemetry/
            │   ├── SchemaTelemetry.php
            │   └── SchemaMetrics.php
            │
            └── Exception/
                ├── DatabaseSchemaException.php
                ├── SchemaDefinitionException.php
                ├── SchemaValidationException.php
                ├── SchemaCapabilityException.php
                ├── SchemaIntrospectionException.php
                ├── SchemaMetadataException.php
                ├── SchemaDiffException.php
                ├── SchemaPlanningException.php
                ├── SchemaCompilationException.php
                ├── SchemaDependencyException.php
                ├── SchemaPortabilityException.php
                ├── SchemaSecurityException.php
                ├── SchemaExtensionException.php
                └── SchemaInvariantException.php
```

---

# 252. Dependency direction

```text
Schema Builder
      ↓
Schema Definition
      ↓
Schema Model / AST
      ↓
Schema Validation
      ↓
Schema Planning
      ↓
Schema Compiler
      ↓
Compiled Command
      ↓
Execution Engine
      ↓
Connection
      ↓
Driver
```

---

# 253. Introspection dependency direction

```text
Driver Catalog
      ↓
Platform Introspector
      ↓
Schema Metadata
      ↓
Schema Model
      ↓
Schema Snapshot
```

---

# 254. Diff dependency direction

```text
Current Schema Model
          +
Target Schema Model
          ↓
      Schema Diff
          ↓
   Schema Planner
```

---

# 255. Forbidden dependency direction

Nunca:

```text
Driver
→ Schema Builder
```

ni:

```text
Connection
→ Schema Diff
```

ni:

```text
Schema Model
→ Execution Engine
```

ni:

```text
Schema Compiler
→ ORM
```

---

# 256. ORM relationship

Posteriormente ORM podrá consumir Schema metadata.

```text
Schema Metadata
        ↓
ORM Mapping Validation
```

Pero Schema no dependerá de ORM.

---

# 257. Query relationship

Query Semantic Engine podrá consumir Schema metadata:

```text
Schema Metadata
        ↓
Schema-aware Query Resolution
```

Esto ya se conecta conceptualmente con:

```text
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
```

---

# 258. Circular dependency prevention

No deberá ocurrir:

```text
Schema
→ Query Semantic Engine
→ Schema
```

Para introspection queries internas, deberán utilizarse contratos inferiores controlados o driver catalog adapters.

---

# 259. Catalog queries

La introspección no deberá depender del Query Builder de alto nivel si esto crea ciclos arquitectónicos.

Preferible:

```text
Driver Catalog API
```

o:

```text
Internal Catalog Command
```

tipado y aislado.

---

# 260. Schema introspection trust boundary

Metadata reportada por el servidor deberá tratarse como:

```text
external structured input
```

y validarse.

---

# 261. Malformed metadata

Un driver/plugin defectuoso podría devolver:

```text
duplicate IDs
invalid column metadata
unknown constraint type
invalid identifier encoding
```

El Schema System deberá detectarlo.

---

# 262. Metadata normalization

Pipeline:

```text
Native Catalog Metadata
        ↓
Driver Adaptation
        ↓
Platform Normalization
        ↓
Structural Validation
        ↓
Schema Metadata
```

---

# 263. Normalization ≠ information destruction

Información platform-specific útil deberá poder conservarse mediante:

```text
extension metadata
```

aunque no forme parte del portable core.

---

# 264. Round-trip metadata

Idealmente:

```text
introspect
→ modify
→ compile
```

no deberá perder características desconocidas silenciosamente.

---

# 265. Unknown platform features

Cuando una característica no pueda representarse:

```text
UnsupportedNativeSchemaFeature
```

deberá quedar registrada.

---

# 266. Preservation mode

Podría existir:

```php
enum UnknownSchemaFeaturePolicy
{
    case FAIL;
    case PRESERVE_OPAQUE;
    case IGNORE_WITH_DIAGNOSTIC;
}
```

---

# 267. Default policy

Para herramientas de migration/diff:

```text
FAIL
```

o:

```text
PRESERVE_OPAQUE
```

será preferible a ignorar silenciosamente.

---

# 268. Schema evolution

La arquitectura deberá permitir que nuevos objetos sean añadidos sin romper:

```text
existing Schema Model
existing migrations
serialized snapshots
extension API
```

---

# 269. Backward compatibility

Los AST/model artifacts versionados deberán tener estrategias de:

```text
migration
compatibility
deprecation
```

que se formalizarán posteriormente.

---

# 270. Core architectural invariants

## DB-SCHEMA-001

Schema Model no ejecutará SQL.

## DB-SCHEMA-002

Schema Builder no generará SQL.

## DB-SCHEMA-003

Schema Compiler no ejecutará SQL.

## DB-SCHEMA-004

Execution Engine no interpretará Schema Model.

## DB-SCHEMA-005

Schema Model será distinto de Schema AST.

## DB-SCHEMA-006

Schema AST será distinto de Query AST.

## DB-SCHEMA-007

Schema Diff será distinto de Migration.

## DB-SCHEMA-008

Schema Introspection será distinta de Schema Compilation.

## DB-SCHEMA-009

Schema Metadata será distinta de Schema Definition.

## DB-SCHEMA-010

Schema Plan será distinto de ExecutionPlan de queries.

## DB-SCHEMA-011

Builder será mutable sólo dentro de su operación.

## DB-SCHEMA-012

Definitions publicadas serán inmutables.

## DB-SCHEMA-013

Schema snapshots serán inmutables.

## DB-SCHEMA-014

Snapshot antiguo no será presentado como estado actual.

## DB-SCHEMA-015

Schema compiler no realizará live DB I/O.

## DB-SCHEMA-016

Schema validation no realizará hidden introspection.

## DB-SCHEMA-017

Capability discovery será separada de compilation.

## DB-SCHEMA-018

Version no será equivalente a capability.

## DB-SCHEMA-019

Platform-specific behavior estará capability-driven.

## DB-SCHEMA-020

MySQL será first-class.

## DB-SCHEMA-021

MariaDB será first-class.

## DB-SCHEMA-022

PostgreSQL será first-class.

## DB-SCHEMA-023

SQLite será first-class.

## DB-SCHEMA-024

MariaDB no será simplemente un alias de MySQL.

## DB-SCHEMA-025

SQLite limitations no serán ocultadas.

## DB-SCHEMA-026

Unsupported features fallarán explícitamente.

## DB-SCHEMA-027

No habrá silent semantic degradation.

## DB-SCHEMA-028

Schema identifiers serán estructurados.

## DB-SCHEMA-029

Identifier quoting pertenecerá al compiler/dialect.

## DB-SCHEMA-030

Runtime values serán distintos de identifiers.

## DB-SCHEMA-031

Dynamic identifiers requerirán validación.

## DB-SCHEMA-032

Logical database type será distinto de physical SQL type.

## DB-SCHEMA-033

Index será distinto de constraint.

## DB-SCHEMA-034

UniqueIndex será distinto de UniqueConstraint.

## DB-SCHEMA-035

PrimaryKey será distinto de Index.

## DB-SCHEMA-036

ForeignKey tendrá semántica estructural propia.

## DB-SCHEMA-037

ValueGenerationStrategy será distinta de AUTO_INCREMENT.

## DB-SCHEMA-038

Schema defaults serán distintos de runtime bindings.

## DB-SCHEMA-039

Schema expressions tendrán contexto explícito.

## DB-SCHEMA-040

Raw DDL será escape hatch explícito.

## DB-SCHEMA-041

Raw DDL será marcado non-portable cuando corresponda.

## DB-SCHEMA-042

Raw DDL no bypassará security policy silenciosamente.

## DB-SCHEMA-043

Schema validation será layered.

## DB-SCHEMA-044

Cross-object references serán validadas.

## DB-SCHEMA-045

Capability requirements serán explícitos.

## DB-SCHEMA-046

Schema operations tendrán identidad tipada.

## DB-SCHEMA-047

Schema object identity será distinta de fingerprint.

## DB-SCHEMA-048

Fingerprint no será sólo hash del DDL.

## DB-SCHEMA-049

Canonicalization preservará semántica.

## DB-SCHEMA-050

Diff evitará falsos positivos por metadata irrelevante.

## DB-SCHEMA-051

Rename no será inferido ciegamente.

## DB-SCHEMA-052

Destructive changes serán clasificados.

## DB-SCHEMA-053

Safety classification no implicará execution authorization.

## DB-SCHEMA-054

Schema dependency graph será explícito.

## DB-SCHEMA-055

Dependency cycles serán tratados explícitamente.

## DB-SCHEMA-056

Compiler no resolverá ciclos mediante string-generation magic.

## DB-SCHEMA-057

Complex platform emulation deberá estar planificada.

## DB-SCHEMA-058

Compiler no ocultará table rebuilds destructivos.

## DB-SCHEMA-059

DDL transaction capability será explícita.

## DB-SCHEMA-060

Schema compiler no controlará transacciones.

## DB-SCHEMA-061

Execution Engine ejecutará compiled schema commands.

## DB-SCHEMA-062

Schema operation result no será ORM hydration.

## DB-SCHEMA-063

Schema introspection usará Connection/Driver boundaries.

## DB-SCHEMA-064

Introspection no utilizará SQL arbitrario disperso.

## DB-SCHEMA-065

Native metadata será validada.

## DB-SCHEMA-066

Unknown metadata no significará false.

## DB-SCHEMA-067

Partial snapshots serán marcados.

## DB-SCHEMA-068

Partial snapshot no será base segura para destructive diff por defecto.

## DB-SCHEMA-069

Schema diff será determinista para inputs equivalentes.

## DB-SCHEMA-070

Schema compilation será determinista para inputs equivalentes.

## DB-SCHEMA-071

Schema naming automático será determinista.

## DB-SCHEMA-072

Identifier truncation evitará colisiones.

## DB-SCHEMA-073

Platform options serán tipadas.

## DB-SCHEMA-074

No habrá arbitrary mixed option bag como core API.

## DB-SCHEMA-075

Portable core no limitará advanced platform features.

## DB-SCHEMA-076

Advanced features serán capability-gated.

## DB-SCHEMA-077

Platform extensions serán tipadas.

## DB-SCHEMA-078

Schema extension registry será frozen.

## DB-SCHEMA-079

No habrá last-wins extension resolution.

## DB-SCHEMA-080

Extensions no ejecutarán DDL durante compilation.

## DB-SCHEMA-081

Extensions no abrirán conexiones durante compilation.

## DB-SCHEMA-082

Extensions no ocultarán destructive operations.

## DB-SCHEMA-083

Shared runtime state será immutable/frozen/stateless.

## DB-SCHEMA-084

Mutable SchemaBuilder no será global.

## DB-SCHEMA-085

Current schema no será global static state.

## DB-SCHEMA-086

Current tenant no será global static state.

## DB-SCHEMA-087

Concurrent requests tendrán schema contexts aislados.

## DB-SCHEMA-088

Concurrent compilations tendrán sessions aisladas.

## DB-SCHEMA-089

FrankenPHP worker reuse no filtrará schema state.

## DB-SCHEMA-090

RoadRunner integration futura conservará aislamiento.

## DB-SCHEMA-091

OpenSwoole integration futura conservará coroutine isolation.

## DB-SCHEMA-092

Database core no dependerá obligatoriamente de Multitenancy.

## DB-SCHEMA-093

Multitenancy se integrará mediante contratos explícitos.

## DB-SCHEMA-094

Schema telemetry no expondrá schema dumps por defecto.

## DB-SCHEMA-095

Telemetry labels evitarán identifiers high-cardinality.

## DB-SCHEMA-096

Schema errors preservarán causas útiles sin credenciales.

## DB-SCHEMA-097

Compilation artifacts se publicarán atómicamente.

## DB-SCHEMA-098

Compilation atomicity será distinta de DDL transaction atomicity.

## DB-SCHEMA-099

Budget exhaustion no producirá artifact falsamente completo.

## DB-SCHEMA-100

Schema introspection será budget-aware.

## DB-SCHEMA-101

Schema diff será budget-aware.

## DB-SCHEMA-102

Schema planning será budget-aware.

## DB-SCHEMA-103

Schema compilation será budget-aware.

## DB-SCHEMA-104

Long-running introspection será cancellation-aware.

## DB-SCHEMA-105

Schema drift será representable.

## DB-SCHEMA-106

No se asumirá global schema lock portable.

## DB-SCHEMA-107

Schema snapshot podrá incluir expected fingerprint.

## DB-SCHEMA-108

Schema facade será convenience API.

## DB-SCHEMA-109

Facade no contendrá platform SQL.

## DB-SCHEMA-110

hasTable pertenecerá a introspection.

## DB-SCHEMA-111

hasColumn pertenecerá a introspection.

## DB-SCHEMA-112

Convenience DSL se expandirá antes de compilation.

## DB-SCHEMA-113

timestamps() será convenience convention.

## DB-SCHEMA-114

softDeletes() será convenience convention.

## DB-SCHEMA-115

foreignId() no fusionará column y FK identities.

## DB-SCHEMA-116

Schema Model no dependerá de ORM.

## DB-SCHEMA-117

ORM podrá consumir Schema Metadata.

## DB-SCHEMA-118

Query Semantic Engine podrá consumir Schema Metadata.

## DB-SCHEMA-119

Schema introspection evitará circular dependency con Query Engine.

## DB-SCHEMA-120

Catalog access tendrá boundary explícito.

## DB-SCHEMA-121

Unknown native features no se perderán silenciosamente.

## DB-SCHEMA-122

Opaque native metadata podrá preservarse explícitamente.

## DB-SCHEMA-123

Schema artifacts tendrán versiones.

## DB-SCHEMA-124

Serialized snapshots no implicarán current state.

## DB-SCHEMA-125

Round-trip conformance será objetivo de drivers oficiales.

## DB-SCHEMA-126

Capability claims deberán probarse.

## DB-SCHEMA-127

Unsupported capabilities deberán fallar antes de enviar DDL inválido cuando sea detectable.

## DB-SCHEMA-128

Creation ordering se derivará del dependency graph.

## DB-SCHEMA-129

Drop ordering respetará dependencias.

## DB-SCHEMA-130

Schema Planner no cambiará la intención estructural.

## DB-SCHEMA-131

Schema Compiler podrá cambiar representación, no significado.

## DB-SCHEMA-132

Schema Introspector normalizará representación, no inventará estructura.

## DB-SCHEMA-133

Schema Diff detectará diferencias, no ejecutará cambios.

## DB-SCHEMA-134

Migration System administrará evolución, no redefinirá Schema Model.

## DB-SCHEMA-135

Execution Engine ejecutará decisiones, no reinterpretará schema.

## DB-SCHEMA-136

No habrá hidden DB I/O en operaciones declaradas puras.

## DB-SCHEMA-137

No habrá hidden schema mutation durante introspection.

## DB-SCHEMA-138

No habrá hidden introspection después de DDL salvo solicitud explícita.

## DB-SCHEMA-139

Security classification acompañará operaciones destructivas.

## DB-SCHEMA-140

Schema Architecture será portable-by-default y capability-aware-by-design.

---

# 271. Fórmula arquitectónica

La arquitectura general puede expresarse:

```text
Database Schema System
=
Schema Model
+
Schema AST
+
Schema Builder
+
Schema Definitions
+
Schema Validation
+
Schema Capabilities
+
Schema Introspection
+
Schema Metadata
+
Schema Comparison
+
Schema Diff
+
Schema Dependency Graph
+
Schema Planning
+
Schema Compilation
+
Schema Extensions
+
Schema Security
+
Schema Telemetry
+
Persistent Runtime Isolation
```

---

# 272. Fórmula de transformación

```text
TargetSchema
+
CurrentSchema
        ↓
      Diff
        ↓
SchemaChangeSet
        ↓
      Plan
        ↓
SchemaChangePlan
        ↓
     Compile
        ↓
CompiledDatabaseCommands
        ↓
     Execute
        ↓
ActualDatabaseSchema
```

---

# 273. Fórmula de introspección

```text
ActualDatabaseSchema
        ↓
Driver Catalog Representation
        ↓
Platform Normalization
        ↓
Schema Metadata
        ↓
DatabaseSchema
        ↓
SchemaSnapshot
```

---

# 274. Fórmula de round-trip

Para una definición portable `D`:

```text
Normalize(
    Introspect(
        Execute(
            Compile(D)
        )
    )
)
≈
Normalize(D)
```

dentro de las capacidades declaradas de la plataforma.

---

# 275. Fórmula de seguridad

```text
SafeSchemaChange
=
ValidStructure
∧
ValidDependencies
∧
SupportedCapabilities
∧
KnownChangeSemantics
∧
AcceptableDestructiveRisk
∧
ValidExecutionContext
```

La autorización final será responsabilidad de Migration/Security layers.

---

# 276. Fórmula de portabilidad

```text
PortableSchemaOperation
=
SemanticIntent
∩
SupportedPlatformCapabilities
```

pero:

```text
Unsupported
```

deberá producir:

```text
explicit capability failure
```

en lugar de degradación silenciosa.

---

# 277. Fórmula de responsabilidad

```text
Schema Builder
=
Intent Construction

Schema Model
=
Structural Representation

Schema Introspection
=
Structural Observation

Schema Diff
=
Structural Comparison

Schema Planner
=
Structural Transformation Planning

Schema Compiler
=
Platform Representation Transformation

Execution Engine
=
Runtime Execution
```

---

# 278. Principio final

La regla central del Schema System será:

> **El Schema System modela y transforma la estructura de la base de datos mediante artefactos tipados; nunca debe convertir una API cómoda en SQL improvisado ni confundir representación, planificación y ejecución.**

En forma compacta:

```text
Schema describes.
Builder constructs.
Introspector observes.
Diff compares.
Planner orders.
Compiler represents.
Executor executes.
Migration governs evolution.
```

---

# 279. Integración con la arquitectura completa

Con este nuevo bloque, VoltStack obtiene:

```text
Application
    │
    ├──────────── Query ──────────────┐
    │                                │
    │   Query Builder                │
    │       ↓                        │
    │   Query AST                    │
    │       ↓                        │
    │   Semantic Engine              │
    │       ↓                        │
    │   Optimizer                    │
    │       ↓                        │
    │   Planner                      │
    │       ↓                        │
    │   SQL Compiler                 │
    │                                │
    ├──────────── Schema ─────────────┤
    │                                │
    │   Schema Builder               │
    │       ↓                        │
    │   Schema Model / AST           │
    │       ↓                        │
    │   Schema Validation            │
    │       ↓                        │
    │   Schema Planner               │
    │       ↓                        │
    │   Schema Compiler              │
    │                                │
    └────────────────┬───────────────┘
                     ↓
             Compiled Commands
                     ↓
              Execution Engine
                     ↓
             Connection Manager
                     ↓
                   Driver
                     ↓
                 Database
```

Mientras la dirección inversa para estructura será:

```text
Database
    ↓
Driver Catalog Adapter
    ↓
Schema Introspection
    ↓
Schema Metadata
    ↓
Schema Model / Snapshot
    ↓
Query Semantic Engine
Migration System
Developer Tooling
ORM Metadata Validation
```

---

# 280. Resultado arquitectónico

Con `87_DATABASE_SCHEMA_ARCHITECTURE.md` queda establecida la frontera principal del **Bloque 8 — Schema**.

Los siguientes documentos profundizarán cada componente:

```text
87_DATABASE_SCHEMA_ARCHITECTURE.md
        │
        ├── 88_DATABASE_SCHEMA_MODEL.md
        │
        ├── 89_DATABASE_SCHEMA_AST_SYSTEM.md
        │
        ├── 90_DATABASE_SCHEMA_BUILDER_SYSTEM.md
        │
        ├── 91_DATABASE_TABLE_DEFINITION_SYSTEM.md
        │
        ├── 92_DATABASE_COLUMN_DEFINITION_SYSTEM.md
        │
        ├── 93_DATABASE_INDEX_SYSTEM.md
        │
        ├── 94_DATABASE_FOREIGN_KEY_SYSTEM.md
        │
        ├── 95_DATABASE_CONSTRAINT_SYSTEM.md
        │
        ├── 96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md
        │
        ├── 97_DATABASE_SCHEMA_METADATA_SYSTEM.md
        │
        ├── 98_DATABASE_SCHEMA_DIFF_SYSTEM.md
        │
        ├── 99_DATABASE_SCHEMA_COMPILER_SYSTEM.md
        │
        └── 100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md
```

La arquitectura mantiene intacta la regla global de VoltStack:

```text
High-Level API
      ↓
Typed Model
      ↓
Validated Architecture
      ↓
Planning
      ↓
Compilation
      ↓
Execution
      ↓
Driver
```

Nunca:

```text
High-Level API
      ↓
SQL improvisado
      ↓
PDO
```

---

# 281. Siguiente documento

```text
88_DATABASE_SCHEMA_MODEL.md
```

Este documento deberá formalizar en profundidad el modelo estructural canónico de VoltStack:

```text
DatabaseSchema
├── Catalog
├── Namespace / Schema
├── Table
│   ├── Column
│   ├── PrimaryKey
│   ├── Index
│   ├── UniqueConstraint
│   ├── CheckConstraint
│   └── ForeignKey
├── Sequence
├── View
└── ExtensionObject
```

y definir especialmente:

```text
Schema Object Identity
Schema Object Naming
Structural Equality
Semantic Equality
Object Ownership
Object Dependencies
Immutable Collections
Schema Object Registry
Schema Canonicalization
Schema Fingerprinting
Platform-Neutral Metadata
Extension Metadata
```

manteniendo la invariante:

```text
Schema Model
=
immutable structural knowledge

Schema Model
≠
DDL
≠
migration
≠
execution
```