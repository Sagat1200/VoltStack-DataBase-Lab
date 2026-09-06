# 90_DATABASE_SCHEMA_BUILDER_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Builder System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 90 — Database Schema Builder System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Builder System` define la API declarativa y ergonómica utilizada por aplicaciones, paquetes y herramientas de VoltStack para expresar operaciones estructurales de base de datos sin escribir DDL manual.

Ejemplo:

```php
Schema::create('users', function (TableBlueprint $table): void {
    $table->id();
    $table->string('email', 320);
    $table->string('name', 150);
    $table->timestamps();

    $table->unique('email');
});
```

El Builder no ejecutará:

```sql
CREATE TABLE ...
```

directamente.

Su responsabilidad termina al producir una representación estructurada que pueda convertirse al `Schema AST`.

La cadena arquitectónica será:

```text
Developer
   │
   ▼
Schema DSL
   │
   ▼
Schema Builder
   │
   ▼
Blueprints
   │
   ▼
Typed Definitions
   │
   ▼
Schema AST
   │
   ▼
Validation
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

Principio fundamental:

> **El Schema Builder optimiza la experiencia del desarrollador; el Schema AST preserva la precisión arquitectónica.**

---

# 2. Objetivos

El sistema deberá ofrecer simultáneamente:

```text
Developer Ergonomics
+
Strong Typing
+
Database Portability
+
Explicit Escape Hatches
+
Capability Awareness
+
Deterministic AST Generation
+
Extension Support
+
Persistent Runtime Safety
```

La API deberá sentirse familiar para desarrolladores PHP provenientes de Laravel, pero sin trasladar al Builder responsabilidades que pertenecen a:

```text
Schema Model
Schema AST
Schema Planner
Schema Compiler
Connection
Driver
Execution Engine
Migration System
```

---

# 3. No objetivos

Schema Builder no deberá:

- ejecutar SQL;
- abrir conexiones;
- realizar introspección oculta;
- detectar automáticamente el motor conectado;
- decidir estrategias físicas de ALTER TABLE;
- gestionar migrations;
- administrar transactions;
- realizar rollback;
- consultar datos;
- realizar backfills;
- generar ORM entities;
- hidratar objetos;
- administrar tenants;
- convertirse en un abstraction layer de SQL arbitrario.

Formalmente:

```text
SchemaBuilder
≠
SchemaManager
≠
SchemaInspector
≠
SchemaCompiler
≠
SchemaExecutor
≠
MigrationRunner
```

---

# 4. Filosofía

La API pública debe ser sencilla:

```php
Schema::create('users', ...);
```

pero internamente deberá producir estructuras precisas.

```text
Simple API
     │
     ▼
Typed Internal Model
     │
     ▼
Architectural Precision
```

La comodidad pública no deberá contaminar el núcleo con ambigüedad.

---

# 5. Principio de lowering

Cada operación del Builder deberá poder reducirse a conceptos tipados.

Ejemplo:

```php
$table->string('email', 320)
      ->nullable(false)
      ->unique();
```

no será almacenado como una cadena.

Conceptualmente:

```text
ColumnBlueprint
├── identifier = email
├── type
│   └── StringType
│       └── length = 320
├── nullability = NOT_NULL
└── modifiers
    └── UNIQUE
```

Posteriormente:

```text
ColumnBlueprint
↓
ColumnDefinition
↓
Schema AST
```

---

# 6. Builder ≠ AST

Esta separación es obligatoria.

```text
Schema Builder
=
Mutable Construction API
```

```text
Schema AST
=
Immutable Structural Intent
```

Por tanto:

```text
Builder may be mutable.

Published AST must be immutable.
```

---

# 7. Builder ≠ Schema Model

Ejemplo:

```php
$table->string('email');
```

expresa:

```text
desired structural definition
```

No significa:

```text
the database currently has email
```

Por tanto:

```text
Builder
≠
Schema introspection
```

---

# 8. Builder ≠ Migration

Una migration puede usar Schema Builder:

```php
public function up(): void
{
    Schema::create(...);
}
```

pero:

```text
Migration
├── identity
├── version/history
├── batch
├── lifecycle
├── execution
├── rollback
└── Schema Builder usage
```

El Builder no conocerá:

```text
migration batch
migration repository
migration status
migration rollback
```

---

# 9. Arquitectura general

```text
┌────────────────────────────┐
│       Public Schema API    │
│ Schema / SchemaManager DSL │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐
│       Schema Builder       │
│                            │
│ create / alter / drop ...  │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐
│         Blueprints         │
│                            │
│ TableBlueprint             │
│ ColumnBlueprint            │
│ IndexBlueprint             │
│ ForeignKeyBlueprint        │
│ ConstraintBlueprint        │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐
│     Definition Factory     │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐
│     Typed Definitions      │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐
│       Schema AST           │
└────────────────────────────┘
```

---

# 10. API principal

Se propone el contrato:

```php
interface SchemaBuilder
{
    public function create(
        string|TableIdentifier $table,
        callable $definition,
    ): SchemaOperation;

    public function table(
        string|TableIdentifier $table,
        callable $alteration,
    ): SchemaOperation;

    public function rename(
        string|TableIdentifier $from,
        string|TableIdentifier $to,
    ): SchemaOperation;

    public function drop(
        string|TableIdentifier $table,
    ): SchemaOperation;

    public function dropIfExists(
        string|TableIdentifier $table,
    ): SchemaOperation;
}
```

La API concreta podrá ofrecer ergonomía adicional.

---

# 11. Schema facade

VoltStack podrá exponer:

```php
use VoltStack\Facades\Schema;

Schema::create(...);
```

pero:

```text
Facade
≠
Builder implementation
```

La facade sólo resolverá la API correspondiente mediante Container.

---

# 12. Facade flow

```text
Schema Facade
      │
      ▼
SchemaManager
      │
      ▼
SchemaBuilderFactory
      │
      ▼
SchemaBuilder
```

La facade no deberá contener lógica estructural.

---

# 13. SchemaManager

Se recomienda introducir:

```php
final class SchemaManager
{
}
```

como API de coordinación pública.

Podrá proporcionar:

```text
create
alter
rename
drop
compile/preview through explicit services
inspection through explicit Schema Inspector
```

sin convertirlo en un God Object.

---

# 14. Builder scope

Cada operación tendrá su propio builder.

Ejemplo:

```text
SchemaManager
      │
      ▼
CreateTableBuilder
      │
      ▼
TableBlueprint
```

Esto evita mantener un builder mutable global.

---

# 15. No global mutable builder

Prohibido:

```php
SchemaBuilder::$currentTable = $table;
```

especialmente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 16. TableBlueprint

La principal DSL estructural será:

```php
final class TableBlueprint
{
}
```

Ejemplo:

```php
Schema::create('users', function (TableBlueprint $table): void {
    $table->id();
    $table->string('email', 320);
    $table->timestamps();
});
```

---

# 17. TableBlueprint responsibility

`TableBlueprint` recopila declarativamente:

```text
columns
indexes
constraints
foreign keys
table options
alterations
```

durante una única operación de construcción.

---

# 18. TableBlueprint lifecycle

```text
CREATED
   ↓
CONFIGURING
   ↓
SEALED
   ↓
LOWERED
```

Después de:

```text
SEALED
```

no deberán aceptarse nuevas mutaciones.

---

# 19. Blueprint sealing

Ejemplo conceptual:

```php
$blueprint->seal();
```

Después:

```php
$blueprint->string('foo');
```

deberá producir:

```text
SealedBlueprintMutationException
```

---

# 20. Mutable builder, immutable output

Regla:

```text
Mutable Construction State
        ↓ seal()
Immutable Definition
        ↓
Immutable Schema AST
```

---

# 21. Column API

Se propone soporte inicial para:

```php
$table->bigInteger('id');
$table->integer('age');
$table->smallInteger('status');
$table->string('name');
$table->text('description');
$table->boolean('active');
$table->decimal('amount', 18, 2);
$table->float('score');
$table->date('birthday');
$table->time('starts_at');
$table->dateTime('published_at');
$table->timestamp('created_at');
$table->json('metadata');
$table->binary('payload');
$table->uuid('uuid');
$table->enum('status', Status::class);
```

---

# 22. Type API ≠ vendor SQL types

La API no deberá exigir:

```php
$table->column('name', 'VARCHAR(255)');
```

como camino normal.

Preferible:

```php
$table->string('name', 255);
```

que produce:

```text
DatabaseType
└── StringType(length=255)
```

---

# 23. Database Type System integration

Builder deberá utilizar el `Database Type System`.

```text
Schema Builder
      │
      ▼
Database Type Definitions
```

No deberá crear un segundo sistema de tipos incompatible.

---

# 24. ColumnDefinition

Un `ColumnBlueprint` deberá convertirse finalmente en:

```php
final readonly class ColumnDefinition
{
    public function __construct(
        public ColumnIdentifier $name,
        public DatabaseType $type,
        public ColumnNullability $nullability,
        public ?SchemaExpression $default,
        public ColumnGeneration $generation,
        public ColumnOptionSet $options,
    ) {}
}
```

---

# 25. ColumnBlueprint

Durante construcción:

```php
final class ColumnBlueprint
{
}
```

podrá ofrecer fluent modifiers.

---

# 26. Fluent API

Ejemplo:

```php
$table
    ->string('email', 320)
    ->notNull()
    ->unique();
```

o:

```php
$table
    ->string('nickname')
    ->nullable();
```

---

# 27. Default nullability

VoltStack deberá establecer una política consistente.

Recomendación:

```text
Columns are NOT NULL by default.
```

Esto favorece esquemas explícitos.

El desarrollador deberá indicar:

```php
->nullable()
```

cuando corresponda.

---

# 28. Nullable

```php
$table->string('nickname')->nullable();
```

produce:

```text
nullability = NULLABLE
```

---

# 29. notNull()

Puede existir explícitamente:

```php
$table->string('email')->notNull();
```

aunque sea el default, para claridad.

---

# 30. default()

Ejemplo:

```php
$table->boolean('active')->default(true);
```

No deberá almacenarse como:

```text
DEFAULT true
```

string.

Debe convertirse a:

```text
LiteralSchemaExpression(Boolean(true))
```

---

# 31. SQL NULL default

Debe poder distinguirse:

```php
->default(null)
```

de:

```text
no default declared
```

Por tanto:

```text
NoDefault
≠
DefaultNull
```

---

# 32. Explicit default model

Se recomienda:

```text
ColumnDefault
├── NoDefault
├── LiteralDefault
├── ExpressionDefault
└── GeneratedDefault
```

---

# 33. defaultExpression()

Para casos estructurados:

```php
$table
    ->timestamp('created_at')
    ->defaultExpression(
        SchemaExpression::currentTimestamp()
    );
```

---

# 34. Raw default

Escape hatch:

```php
$table
    ->timestamp('created_at')
    ->rawDefault('CURRENT_TIMESTAMP');
```

deberá ser explícito y tratado como no portable.

---

# 35. timestamps()

Helper:

```php
$table->timestamps();
```

podrá producir:

```text
created_at
updated_at
```

con tipos y defaults definidos por una política VoltStack.

---

# 36. timestampsTz()

Puede existir:

```php
$table->timestampsTz();
```

si el Type System distingue timestamps con timezone.

---

# 37. softDeletes()

Puede ofrecerse:

```php
$table->softDeletes();
```

pero debe entenderse como:

```text
Schema DSL convenience
```

No como implementación del futuro `Soft Delete System`.

Produce simplemente una definición de columna.

---

# 38. softDeletesTz()

Podrá existir:

```php
$table->softDeletesTz();
```

---

# 39. rememberToken()

No se recomienda introducir helpers excesivamente ligados a un sistema de autenticación concreto dentro del core Schema Builder.

Si se desea:

```php
$table->rememberToken();
```

debería pertenecer a una integración de Authentication.

---

# 40. Principio de helpers

Un helper pertenece al core cuando representa:

```text
generic database schema convention
```

No cuando representa:

```text
business/application subsystem convention
```

---

# 41. id()

Se propone:

```php
$table->id();
```

como shorthand portable.

Conceptualmente:

```text
id()
↓
ColumnDefinition
├── name = id
├── integer identity-compatible type
├── NOT NULL
└── primary key
```

---

# 42. Configurable IDs

VoltStack podrá soportar:

```php
$table->id();
$table->bigId();
$table->uuidId();
$table->ulidId();
```

dependiendo del Type System.

---

# 43. ID strategy

No deberá asumirse:

```text
AUTO_INCREMENT
```

como semántica universal.

Preferible:

```text
IdentifierGenerationStrategy
```

---

# 44. Identifier generation

Ejemplo:

```text
NONE
IDENTITY
SEQUENCE
APPLICATION
UUID
ULID
PLATFORM_NATIVE
```

---

# 45. foreignId()

API:

```php
$table->foreignId('user_id');
```

debe crear inicialmente una columna compatible con IDs.

Pero:

```text
foreignId()
≠
foreign key automatically
```

a menos que se utilice una operación explícita como:

```php
->constrained()
```

---

# 46. constrained()

Ejemplo:

```php
$table
    ->foreignId('user_id')
    ->constrained('users');
```

podrá generar:

```text
ColumnDefinition(user_id)
+
ForeignKeyDefinition
```

---

# 47. Explicit foreign key API

También:

```php
$table
    ->foreign('user_id')
    ->references('id')
    ->on('users');
```

---

# 48. Typed foreign references

Internamente:

```text
ForeignKeyBlueprint
├── localColumns
├── referencedTable
├── referencedColumns
├── onDelete
├── onUpdate
├── deferrability
└── options
```

---

# 49. Referential actions

API:

```php
->onDelete(ReferentialAction::CASCADE)
->onUpdate(ReferentialAction::RESTRICT)
```

Además de convenience methods:

```php
->cascadeOnDelete();
->restrictOnDelete();
->nullOnDelete();
```

---

# 50. No magic cascade

Nunca deberá inferirse:

```text
foreignId()
→ ON DELETE CASCADE
```

sin petición explícita.

---

# 51. Foreign key shorthand

Podrá existir:

```php
$table
    ->foreignId('user_id')
    ->constrained()
    ->cascadeOnDelete();
```

donde `constrained()` infiere convencionalmente:

```text
user_id
→ users.id
```

---

# 52. Convention inference

La inferencia deberá ser realizada por un servicio explícito:

```text
SchemaNamingConventionResolver
```

No mediante lógica dispersa.

---

# 53. Convention vs fact

Inferir:

```text
user_id → users
```

es una convención.

No es una verdad estructural.

---

# 54. Convention overrides

Debe poder hacerse:

```php
$table
    ->foreignId('owner_id')
    ->constrained(
        table: 'users',
        column: 'id'
    );
```

---

# 55. Primary key

```php
$table->primary('id');
```

o compuesto:

```php
$table->primary(['tenant_id', 'id']);
```

---

# 56. Unique constraint

```php
$table->unique('email');
```

o:

```php
$table->unique(['tenant_id', 'email']);
```

---

# 57. Index

```php
$table->index('email');
```

y:

```php
$table->index(['status', 'created_at']);
```

---

# 58. Named index

```php
$table->index(
    columns: ['status', 'created_at'],
    name: 'users_status_created_idx',
);
```

---

# 59. Generated names

Si no se especifica nombre:

```text
SchemaNamingStrategy
```

deberá producirlo determinísticamente.

---

# 60. NamingStrategy

Ejemplo:

```text
table_columns_kind
```

podría producir:

```text
users_email_unique
users_status_created_at_index
```

---

# 61. Identifier length

NamingStrategy deberá considerar:

```text
platform identifier length capability
```

durante una fase adecuada.

No deberá truncar nombres arbitrariamente en el Builder sin contexto.

---

# 62. Logical names

Puede distinguirse:

```text
LogicalConstraintName
```

de:

```text
PhysicalConstraintName
```

cuando sea necesario.

---

# 63. Check constraint

API:

```php
$table->check(
    SchemaExpression::greaterThan(
        SchemaExpression::column('age'),
        SchemaExpression::literal(0),
    )
);
```

---

# 64. Convenience check

Puede existir:

```php
$table->check('age > 0');
```

sólo como raw/trusted escape hatch, no como representación estructurada preferida.

---

# 65. Generated columns

API posible:

```php
$table
    ->decimal('total', 18, 2)
    ->generatedAs(
        SchemaExpression::multiply(
            SchemaExpression::column('quantity'),
            SchemaExpression::column('price'),
        )
    );
```

---

# 66. Stored vs virtual

Cuando aplique:

```php
->stored();
```

o:

```php
->virtual();
```

deberá producir una intención semántica capability-aware.

---

# 67. Platform capability

Si una plataforma no soporta:

```text
generated columns
```

el Builder no deberá ocultarlo.

La incompatibilidad se detectará posteriormente.

---

# 68. Comments

API:

```php
$table
    ->string('email')
    ->comment('Primary account email');
```

Los comentarios son metadata estructural cuando la plataforma los soporta.

---

# 69. Collation

```php
$table
    ->string('name')
    ->collation('...');
```

debe utilizar un tipo estructurado:

```text
CollationIdentifier
```

cuando sea posible.

---

# 70. Charset

Opciones como:

```php
$table->charset(...);
```

deberán modelarse como table options/capabilities y no como SQL arbitrario.

---

# 71. Table options

```text
TableOptionSet
```

podrá incluir:

```text
charset
collation
storage characteristics
comment
temporary
platform extensions
```

---

# 72. Portable vs platform options

Debe distinguirse:

```text
PortableTableOption
```

de:

```text
PlatformTableOption
```

---

# 73. Platform-specific API

Puede permitirse:

```php
$table->platformOption(...);
```

pero debe quedar explícitamente marcado como:

```text
non-portable
```

---

# 74. MySQL engine example

Evitar convertir:

```php
$table->engine('InnoDB');
```

en concepto universal.

Mejor:

```php
$table->platformOption(
    MySqlTableOption::engine('InnoDB')
);
```

---

# 75. MariaDB first-class

No se asumirá que toda opción MySQL es automáticamente equivalente en MariaDB.

---

# 76. PostgreSQL first-class

Opciones específicas como:

```text
tablespace
storage parameters
```

deberán pertenecer a extensiones/capabilities PostgreSQL.

---

# 77. SQLite first-class

El Builder deberá respetar conceptos propios como:

```text
WITHOUT ROWID
STRICT
```

mediante opciones tipadas/capability-aware cuando sean soportadas.

---

# 78. Create table

API:

```php
Schema::create('users', function (TableBlueprint $table): void {
});
```

produce:

```text
CreateTableBlueprint
      ↓
TableDefinition
      ↓
CreateTableNode
```

---

# 79. createIfNotExists()

Podrá existir:

```php
Schema::createIfNotExists('users', ...);
```

produciendo:

```text
CreationBehavior::IGNORE_IF_EXISTS
```

---

# 80. Idempotency warning

`createIfNotExists()` no garantiza que una definición existente sea compatible con la solicitada.

Por tanto:

```text
IF NOT EXISTS
≠
Schema Synchronization
```

---

# 81. Alter table

```php
Schema::table('users', function (TableBlueprint $table): void {
    $table->string('nickname')->nullable();
});
```

debe producir:

```text
AlterTable structural intent
```

---

# 82. Create blueprint ≠ alter blueprint

Aunque puedan compartir APIs, internamente conviene distinguir:

```text
CreateTableBlueprint
AlterTableBlueprint
```

porque las operaciones válidas son diferentes.

---

# 83. CreateTableBlueprint

Puede permitir:

```text
define columns
define indexes
define constraints
table options
```

pero no:

```text
rename existing column
drop existing column
```

---

# 84. AlterTableBlueprint

Puede permitir:

```text
add column
alter column
rename column
drop column
add/drop index
add/drop constraint
```

---

# 85. Context-specific DSL

Esto evita APIs imposibles como:

```php
Schema::create('users', function ($table) {
    $table->dropColumn('foo');
});
```

---

# 86. Add column

Dentro de alter:

```php
$table->string('nickname');
```

se interpreta como:

```text
AddColumn
```

---

# 87. change()

Para modificar una columna puede ofrecerse:

```php
$table
    ->string('email', 500)
    ->change();
```

pero esta API presenta ambigüedades.

---

# 88. Preferred alter API

Se recomienda ofrecer una API más explícita:

```php
$table->alterColumn('email', function (ColumnAlteration $column): void {
    $column->type(
        DatabaseType::string(500)
    );
});
```

---

# 89. Convenience change()

Podrá conservarse como convenience layer:

```php
$table
    ->string('email', 500)
    ->change();
```

pero deberá lowerse a:

```text
ColumnChangeSet
```

con estados explícitos.

---

# 90. Unchanged semantics

El Builder deberá distinguir:

```text
not specified
```

de:

```text
explicitly changed
```

---

# 91. ColumnAlteration

```php
final class ColumnAlteration
{
    public function type(DatabaseType $type): self;

    public function nullable(): self;

    public function notNull(): self;

    public function default(mixed $value): self;

    public function dropDefault(): self;
}
```

---

# 92. Rename column

```php
$table->renameColumn('name', 'full_name');
```

produce:

```text
RenameColumnNode
```

No:

```text
Drop + Add
```

---

# 93. Drop column

```php
$table->dropColumn('legacy_code');
```

---

# 94. Multiple drops

Convenience:

```php
$table->dropColumn([
    'legacy_code',
    'legacy_status',
]);
```

debe producir operaciones individuales tipadas.

---

# 95. dropColumns()

Alternativamente:

```php
$table->dropColumns(
    'legacy_code',
    'legacy_status',
);
```

---

# 96. Drop timestamps

Helper:

```php
$table->dropTimestamps();
```

produce drops explícitos de:

```text
created_at
updated_at
```

---

# 97. Drop soft deletes

```php
$table->dropSoftDeletes();
```

será sólo convenience DSL.

---

# 98. Rename table

```php
Schema::rename('users', 'accounts');
```

produce:

```text
RenameTableNode
```

---

# 99. Drop table

```php
Schema::drop('users');
```

produce:

```text
DropTableNode
DropBehavior::REQUIRE_PRESENT
```

---

# 100. Drop if exists

```php
Schema::dropIfExists('users');
```

produce:

```text
DropBehavior::IGNORE_IF_MISSING
```

---

# 101. Cascade

No se recomienda:

```php
Schema::drop('users', cascade: true);
```

como boolean.

Preferible:

```php
Schema::drop(
    'users',
    dependencyBehavior: DependencyDropBehavior::CASCADE
);
```

---

# 102. Destructive API visibility

Operaciones destructivas deberán ser claramente identificables.

Podrá existir:

```php
Schema::drop(...);
$table->dropColumn(...);
$table->dropForeign(...);
```

sin esconderlas detrás de métodos genéricos.

---

# 103. hasTable()

Una API como:

```php
Schema::hasTable('users');
```

es útil, pero conceptualmente pertenece a:

```text
Schema Introspection System
```

no al Builder.

---

# 104. API aggregation

La facade pública puede exponer:

```php
Schema::hasTable(...)
```

pero internamente deberá delegar a:

```text
SchemaInspector
```

---

# 105. hasColumn()

Igualmente:

```php
Schema::hasColumn('users', 'email');
```

delega a introspection.

---

# 106. Separation

```text
Schema Facade
├── Builder operations
│   ├── create
│   ├── table
│   ├── rename
│   └── drop
│
└── Inspector operations
    ├── hasTable
    ├── hasColumn
    ├── tables
    └── columns
```

La API pública puede ser unificada.

La arquitectura interna no.

---

# 107. Connection selection

El Builder podrá aceptar:

```php
Schema::connection('analytics')
    ->create(...);
```

pero la conexión no deberá abrirse durante construcción.

---

# 108. Connection target

Debe producir:

```text
SchemaExecutionTarget
```

o metadata equivalente.

---

# 109. Connection name ≠ Connection object

```text
"analytics"
≠
Connection
```

El Builder conserva una referencia lógica.

---

# 110. Tenant context

El Builder core no deberá consultar:

```text
global current tenant
```

---

# 111. Multitenancy integration

El paquete Multitenancy podrá proporcionar:

```text
TenantSchemaContext
```

de manera explícita.

---

# 112. Core independence

```text
Quantum/Database
```

no deberá depender de:

```text
Quantum/Multitenancy
```

---

# 113. Prefixes

Evitar sistemas globales ambiguos como:

```text
Schema::setTablePrefix(...)
```

Preferir resolución explícita de identifiers/namespace.

---

# 114. Schema namespace

Podrá soportarse:

```php
Schema::namespace('analytics')
    ->create('events', ...);
```

que produce una referencia estructurada:

```text
analytics.events
```

---

# 115. Qualified identifiers

Internamente:

```text
QualifiedTableIdentifier
├── catalog?
├── schema?
└── table
```

No:

```text
"analytics.events"
```

como única representación.

---

# 116. Temporary tables

Puede existir:

```php
Schema::createTemporary('temp_users', ...);
```

pero deberá expresarse como:

```text
TablePersistence::TEMPORARY
```

no mediante SQL raw.

---

# 117. Temporary semantics

La vida real de una temporary table depende de:

```text
connection
session
transaction
platform
```

El Builder sólo expresa la intención.

---

# 118. Views

Podrá ofrecerse:

```php
Schema::createView(...);
Schema::dropView(...);
```

pero el query que define la vista deberá ser estructurado.

---

# 119. View query

Idealmente:

```php
Schema::createView(
    'active_users',
    Query::table('users')
        ->select(...)
        ->where(...)
);
```

requiere una frontera formal entre Query Model y Schema.

---

# 120. No premature Query dependency

Hasta que esa integración esté formalizada, el Builder puede usar:

```text
ViewDefinition
```

como contrato independiente.

---

# 121. Raw view

Puede existir:

```php
Schema::createRawView(...);
```

como escape hatch explícito.

---

# 122. Sequences

API posible:

```php
Schema::createSequence('invoice_number');
```

con:

```php
Schema::sequence('invoice_number')
    ->start(1)
    ->incrementBy(1);
```

---

# 123. Sequence builder

```text
SequenceBlueprint
↓
SequenceDefinition
↓
CreateSequenceNode
```

---

# 124. Sequence portability

El Builder permite expresar la intención.

No promete que todas las plataformas la soporten.

---

# 125. Index Builder

Para casos avanzados:

```php
$table->index(function (IndexBlueprint $index): void {
    $index->column('status');
    $index->column('created_at')->descending();
});
```

---

# 126. Expression indexes

```php
$table->index(function (IndexBlueprint $index): void {
    $index->expression(
        SchemaExpression::lower(
            SchemaExpression::column('email')
        )
    );
});
```

---

# 127. Partial indexes

```php
$index->where(
    SchemaExpression::equals(
        SchemaExpression::column('active'),
        SchemaExpression::literal(true)
    )
);
```

---

# 128. Included columns

Cuando aplique:

```php
$index->include(['name', 'email']);
```

deberá quedar capability-aware.

---

# 129. Index method

Ejemplo:

```php
$index->method(IndexMethod::BTREE);
```

pero métodos específicos podrán ser platform extension values.

---

# 130. Full-text index

No deberá asumirse que:

```text
FULLTEXT
```

tiene idéntica semántica en MySQL, MariaDB y PostgreSQL.

Se requiere una abstracción/capability adecuada.

---

# 131. Spatial indexes

Misma regla:

```text
spatial intent
≠
universal SQL syntax
```

---

# 132. Foreign key builder

Ejemplo completo:

```php
$table->foreign(['tenant_id', 'user_id'])
    ->references(['tenant_id', 'id'])
    ->on('users')
    ->cascadeOnUpdate()
    ->restrictOnDelete();
```

---

# 133. Foreign key arity

Debe validarse:

```text
count(local columns)
=
count(referenced columns)
```

---

# 134. Foreign key self-reference

Permitido:

```php
$table
    ->foreignId('parent_id')
    ->nullable()
    ->constrained('categories');
```

---

# 135. Self-reference is explicit

No deberá tratarse como ciclo inválido automáticamente.

---

# 136. Deferrable constraints

API avanzada:

```php
$foreign
    ->deferrable()
    ->initiallyDeferred();
```

capability-aware.

---

# 137. Validation options

PostgreSQL-like concepts como:

```text
NOT VALID
```

deberán modelarse mediante opciones tipadas, no strings universales.

---

# 138. Column positioning

Convenience:

```php
$table->string('slug')->after('name');
```

o:

```php
$table->string('slug')->first();
```

pero:

```text
column placement
```

será una preferencia/capability específica.

---

# 139. No hidden table rebuild

Si SQLite requiere reconstruir una tabla para cierta alteración:

```text
Builder
```

no deberá decidirlo.

Eso pertenece a:

```text
Schema Planner
```

---

# 140. No hidden platform strategy

Ejemplo:

```php
$table->dropColumn('foo');
```

significa:

```text
desired structural operation
```

no:

```text
execute ALTER TABLE DROP COLUMN
```

---

# 141. Builder validation

El Builder deberá detectar errores locales tempranos.

Ejemplo:

```php
$table->string('');
```

deberá fallar antes del AST.

---

# 142. Local validation examples

```text
empty identifier
negative length
precision < scale
duplicate blueprint local ID
empty foreign key
empty index
invalid enum definition
contradictory modifiers
```

---

# 143. Builder validation ≠ schema validation

Builder puede saber:

```text
string length cannot be negative
```

pero no necesariamente:

```text
column already exists in database
```

---

# 144. No hidden introspection

Nunca:

```php
$table->string('email');
```

deberá consultar la base para decidir si ya existe.

---

# 145. Schema-aware builder mode

Si alguna herramienta necesita contexto actual:

```text
SchemaBuildContext
```

podrá recibir un `DatabaseSchema` snapshot explícito.

---

# 146. Explicit context

```php
$builder = $factory->forSchema($schemaSnapshot);
```

No:

```text
builder secretly introspects database
```

---

# 147. Builder diagnostics

Errores deberán apuntar a la DSL original.

Ejemplo:

```text
Invalid decimal definition for column "amount":
scale 6 cannot exceed precision 4.
```

---

# 148. Source mapping

Builder podrá registrar:

```text
BlueprintSourceLocation
```

que posteriormente se transfiera a:

```text
SchemaNodeMetadata
```

---

# 149. Source mapping purpose

Esto permite que un error del planner/compiler pueda indicar:

```text
database/migrations/...
line ...
```

cuando exista metadata suficiente.

---

# 150. Naming convention resolver

Se propone:

```php
interface SchemaNamingConventionResolver
{
    public function inferReferencedTable(
        ColumnIdentifier $foreignColumn
    ): TableIdentifier;
}
```

---

# 151. Naming strategy vs convention resolver

```text
NamingConventionResolver
=
infers semantic names from conventions
```

```text
SchemaNamingStrategy
=
generates object names
```

Ejemplo:

```text
user_id → users
```

es convention resolution.

```text
users_email_unique
```

es name generation.

---

# 152. Pluralization

No debe acoplarse directamente a una implementación global de pluralización.

Puede utilizar:

```text
InflectorContract
```

inyectado durante bootstrap.

---

# 153. Deterministic conventions

Misma configuración:

```text
same builder input
→
same definitions
```

---

# 154. Environment-independent output

No deberá ocurrir:

```text
production → different AST
development → different AST
```

salvo que configuración semántica explícita lo solicite.

---

# 155. Clock independence

Builder no deberá usar:

```php
new DateTimeImmutable();
```

para generar defaults accidentalmente.

---

# 156. Default current time

Incorrecto:

```php
$table->timestamp('created_at')->default(new DateTime());
```

si se pretende current DB timestamp.

Correcto:

```php
->defaultExpression(
    SchemaExpression::currentTimestamp()
);
```

---

# 157. UUID defaults

Debe distinguirse:

```text
database-generated UUID
application-generated UUID
literal UUID default
```

---

# 158. Enum API

Podrá soportarse:

```php
$table->enum('status', [
    'pending',
    'active',
    'disabled',
]);
```

y:

```php
$table->enum('status', UserStatus::class);
```

---

# 159. Native enum ≠ semantic enum

El Type System decidirá cómo representar:

```text
semantic enum
```

según plataforma/strategy.

Builder no deberá asumir:

```text
CREATE TYPE
```

o:

```text
ENUM(...)
```

directamente.

---

# 160. JSON

```php
$table->json('metadata');
```

expresa tipo semántico JSON.

No:

```text
MySQL JSON specifically
```

---

# 161. JSON binary variants

Si se requiere algo como PostgreSQL `jsonb`, deberá modelarse mediante:

```text
semantic capability/type option
```

o extensión específica.

---

# 162. Decimal

```php
$table->decimal('amount', 18, 2);
```

deberá preservar:

```text
precision = 18
scale = 2
```

sin convertirlo a float.

---

# 163. Monetary helper

Evitar introducir:

```php
$table->money(...)
```

como tipo universal sin semántica claramente definida.

Podría pertenecer a un custom type.

---

# 164. Custom types

API:

```php
$table->column(
    'vector',
    CustomDatabaseType::of('vector', ...)
);
```

---

# 165. Type extension

Custom type deberá registrarse mediante:

```text
Database Type Registry
```

no directamente dentro del Schema Builder.

---

# 166. Builder extensions

Se requiere:

```text
SchemaBuilderExtensionSystem
```

o integración con el futuro `Database Extension Architecture`.

---

# 167. Extension examples

Un paquete PostgreSQL podría agregar:

```php
$table->uuidNative(...);
$table->inet(...);
$table->tsvector(...);
```

si dichas APIs se registran formalmente.

---

# 168. Extension rules

Una extensión no podrá:

```text
execute queries
open connections
mutate global request state
override core methods silently
bypass identifier validation
inject raw SQL implicitly
```

---

# 169. Extension registry

```text
SchemaBuilderExtensionRegistry
```

deberá congelarse después del bootstrap.

---

# 170. No last-wins

Dos extensiones que intenten registrar:

```text
same DSL method / same extension ID
```

deberán provocar conflicto explícito.

---

# 171. Macro system

Un macro system dinámico tipo:

```php
Schema::macro(...)
```

puede ser cómodo, pero introduce problemas de:

```text
global mutation
persistent runtime leakage
IDE discoverability
type safety
conflicts
```

---

# 172. Recommended extension strategy

Preferir:

```text
typed extension objects
+
frozen registry
+
bootstrap registration
```

sobre macros globales mutables.

---

# 173. Application-level convenience

La aplicación sí puede construir helpers:

```php
function addAuditColumns(TableBlueprint $table): void
{
    $table->timestamps();
    $table->uuid('created_by')->nullable();
}
```

sin necesidad de alterar globalmente el Builder.

---

# 174. Reusable schema components

VoltStack puede ofrecer:

```text
SchemaFragment
```

para definiciones reutilizables.

---

# 175. SchemaFragment

```php
interface SchemaFragment
{
    public function apply(TableBlueprint $table): void;
}
```

---

# 176. Example fragment

```php
final class AuditColumns implements SchemaFragment
{
    public function apply(TableBlueprint $table): void
    {
        $table->timestamps();
        $table->uuid('created_by')->nullable();
        $table->uuid('updated_by')->nullable();
    }
}
```

---

# 177. Fragment ≠ inheritance

Preferir composición:

```text
TableBlueprint
+
SchemaFragment
```

en lugar de jerarquías complejas de Blueprints.

---

# 178. Fragment determinism

Fragments deberán seguir las mismas reglas:

```text
no DB I/O
no hidden tenant
no random schema mutation
no execution
```

---

# 179. Conditional schema construction

PHP permite:

```php
if ($feature) {
    $table->string('foo');
}
```

Esto es válido.

Pero el valor:

```text
$feature
```

forma parte de la entrada externa al Builder.

---

# 180. Determinism boundary

Formalmente:

```text
BuildResult
=
f(
    BuilderInput,
    ExplicitConfiguration,
    RegisteredExtensions
)
```

No:

```text
f(hidden runtime state)
```

---

# 181. Blueprint IDs

Cada elemento mutable podrá tener:

```text
BlueprintId
```

para diagnostics.

Debe distinguirse:

```text
BlueprintId
≠
SchemaNodeId
≠
SchemaObjectId
```

---

# 182. Lowering mapping

Podrá conservarse:

```text
BlueprintId
→ SchemaNodeId
```

para diagnostics/source mapping.

---

# 183. Blueprint metadata

```php
final class BlueprintMetadata
{
}
```

puede contener:

```text
source location
origin
developer annotations
extension provenance
```

---

# 184. Builder result

Se recomienda:

```php
final readonly class SchemaBuildResult
{
    public function __construct(
        public SchemaAst $ast,
        public SchemaBuildDiagnostics $diagnostics,
        public SchemaBuildMetadata $metadata,
    ) {}
}
```

---

# 185. Public convenience

La facade puede ocultar este detalle:

```php
Schema::create(...)
```

mientras herramientas avanzadas pueden usar:

```php
$result = $schemaBuilder->build(...);
```

---

# 186. Build ≠ execute

Debe quedar formalmente separado:

```php
$result = $builder->build(...);
```

de:

```php
$executor->execute(...);
```

---

# 187. Migration convenience execution

Una migration podría parecer:

```php
Schema::create(...);
```

y ejecutarse dentro del Migration Runner.

Arquitectónicamente:

```text
Migration Runner
      ↓
Schema API
      ↓
Build AST
      ↓
Plan
      ↓
Compile
      ↓
Execute
```

No porque Builder ejecute directamente.

---

# 188. Deferred operation collection

El Migration Runner puede utilizar:

```text
SchemaOperationCollector
```

para recopilar múltiples operaciones antes de planificarlas.

---

# 189. Operation collector

Ejemplo:

```text
Migration
├── Create users
├── Create posts
└── Add FK posts → users
```

puede convertirse en:

```text
SchemaAst
├── CreateTable(users)
├── CreateTable(posts)
└── AddForeignKey(...)
```

permitiendo análisis global.

---

# 190. Why collection matters

Esto permite:

```text
dependency planning
cycle handling
capability analysis
migration safety analysis
optimization of DDL strategy
```

que sería imposible si cada llamada ejecutara SQL inmediatamente.

---

# 191. Immediate API illusion

El API puede parecer imperativo:

```php
Schema::create(...);
Schema::create(...);
```

pero dentro de migration context puede operar declarativamente.

---

# 192. Execution mode

Se podrá distinguir:

```text
COLLECT
IMMEDIATE_ORCHESTRATED
PREVIEW
VALIDATE_ONLY
```

pero el Builder sigue sin ejecutar.

---

# 193. SchemaOperationSink

La facade puede enviar operaciones a:

```php
interface SchemaOperationSink
{
    public function accept(SchemaAst $operation): void;
}
```

---

# 194. Context-specific sinks

Ejemplos:

```text
MigrationOperationCollector
ImmediateSchemaOrchestrator
PreviewCollector
TestingCollector
```

---

# 195. Builder architecture

```text
Schema API
   │
   ▼
Builder
   │
   ▼
Schema AST
   │
   ▼
Operation Sink
   │
   ├── Migration Collector
   ├── Preview
   ├── Test Collector
   └── Execution Orchestrator
```

Esto preserva la separación de responsabilidades.

---

# 196. Preview

VoltStack podrá permitir:

```php
Schema::preview(function (): void {
    Schema::create(...);
});
```

y devolver:

```text
Schema AST
+
Plan
+
potential SQL
+
diagnostics
```

mediante servicios posteriores.

---

# 197. Preview ≠ Builder responsibility

Builder sólo produce AST.

El preview coordinator llama:

```text
Builder
Planner
Compiler
```

---

# 198. Dry run

Similarmente:

```text
dry-run
```

pertenece al orchestration/migration tooling.

---

# 199. Table Blueprint modes

```php
enum TableBlueprintMode
{
    case CREATE;
    case ALTER;
}
```

puede ayudar a validar operaciones.

---

# 200. Context restrictions

Ejemplo:

```text
CREATE mode
```

permite:

```text
add definitions
```

mientras:

```text
ALTER mode
```

permite:

```text
add
change
rename
drop
```

---

# 201. Blueprint child objects

```text
TableBlueprint
├── ColumnBlueprint
├── ColumnAlteration
├── IndexBlueprint
├── ConstraintBlueprint
├── ForeignKeyBlueprint
├── SequenceReference?
└── TableOptionBlueprint
```

---

# 202. Fluent ownership

Cada child blueprint deberá conocer a su owner mediante una referencia controlada durante construcción.

Pero dicha referencia no debe sobrevivir al lowering.

---

# 203. No cyclic immutable output

El AST final deberá evitar grafos de objetos PHP cíclicos innecesarios.

Usará:

```text
IDs
references
immutable value objects
```

---

# 204. Column modifier order

Estas dos expresiones:

```php
$table->string('email')->nullable()->unique();
```

y:

```php
$table->string('email')->unique()->nullable();
```

deberán producir el mismo resultado cuando los modifiers sean semánticamente conmutativos.

---

# 205. Modifier conflicts

Ejemplo:

```php
$table->string('email')
    ->nullable()
    ->notNull();
```

La política recomendada:

```text
fail on contradictory declarations
```

en lugar de:

```text
last call wins
```

---

# 206. No last-call-wins ambiguity

Especialmente para:

```text
nullable/notNull
stored/virtual
unique/nonUnique
identity/application generated
```

---

# 207. Duplicate columns

Esto:

```php
$table->string('email');
$table->integer('email');
```

deberá fallar localmente.

---

# 208. Duplicate indexes

Dos índices con identidad lógica conflictiva deberán detectarse.

---

# 209. Constraint collision

Igualmente:

```text
same logical constraint name
+
different definition
```

debe fallar.

---

# 210. Same semantic declaration

La política sobre declaraciones exactamente duplicadas debe ser explícita.

Recomendación:

```text
duplicate declaration = error
```

para evitar bugs silenciosos.

---

# 211. Identifier validation

Deberá existir:

```text
SchemaIdentifierValidator
```

compartido con el Schema Model/Compiler donde corresponda.

---

# 212. Identifier rules

Debe distinguir:

```text
logical validity
```

de:

```text
platform physical limits
```

---

# 213. Portable identifiers

La configuración puede exigir:

```text
portable identifier profile
```

para detectar nombres problemáticos antes.

---

# 214. Reserved words

No necesariamente deberán prohibirse.

El compiler puede quote identifiers.

Pero herramientas pueden advertir.

---

# 215. Identifier quoting

Builder nunca deberá hacer:

```php
$table->string('`email`');
```

como mecanismo normal.

---

# 216. Qualified column names

Dentro de una tabla:

```php
$table->string('users.email');
```

deberá rechazarse como nombre de columna local.

---

# 217. Constraint expression references

Las expresiones utilizarán:

```text
ColumnIdentifier
```

tipado.

---

# 218. Builder error hierarchy

Se propone:

```text
DatabaseSchemaBuilderException
├── InvalidSchemaBuilderStateException
├── InvalidBlueprintException
├── SealedBlueprintMutationException
├── DuplicateColumnDefinitionException
├── DuplicateIndexDefinitionException
├── DuplicateConstraintDefinitionException
├── InvalidColumnDefinitionException
├── InvalidIndexDefinitionException
├── InvalidForeignKeyDefinitionException
├── InvalidTableOptionException
├── InvalidSchemaIdentifierException
├── ConflictingColumnModifierException
├── UnsupportedBuilderOperationException
├── SchemaBuilderExtensionException
├── SchemaBuilderBudgetExceededException
├── SchemaBuilderSecurityException
└── SchemaBuilderInvariantException
```

---

# 219. Diagnostic codes

Ejemplos:

```text
DB_SCHEMA_BUILDER_DUPLICATE_COLUMN
DB_SCHEMA_BUILDER_INVALID_IDENTIFIER
DB_SCHEMA_BUILDER_CONFLICTING_MODIFIER
DB_SCHEMA_BUILDER_INVALID_FOREIGN_KEY
DB_SCHEMA_BUILDER_SEALED_BLUEPRINT
DB_SCHEMA_BUILDER_UNSUPPORTED_OPERATION
```

---

# 220. Builder budgets

Aunque el Builder no ejecuta DB I/O, debe limitar entradas patológicas.

```php
final readonly class SchemaBuilderBudget
{
    public function __construct(
        public int $maxTables,
        public int $maxColumnsPerTable,
        public int $maxIndexesPerTable,
        public int $maxConstraintsPerTable,
        public int $maxExpressionDepth,
        public int $maxAnnotations,
    ) {}
}
```

---

# 221. Budget failure

Nunca truncar:

```text
1001 columns
→ silently keep first 1000
```

Debe fallar explícitamente.

---

# 222. Security

La seguridad del Builder se apoya en:

```text
typed identifiers
typed expressions
typed options
explicit raw boundaries
frozen extension registry
input validation
```

---

# 223. Runtime values

Builder no deberá aceptar valores de usuario final como estructura sin validación.

Ejemplo peligroso:

```php
$tableName = $_GET['table'];

Schema::drop($tableName);
```

El framework puede validar sintaxis, pero la autorización de dicha operación pertenece a la aplicación/policy.

---

# 224. DDL is privileged

El hecho de que un identifier sea sintácticamente seguro no significa:

```text
authorized to modify that object
```

---

# 225. Raw SQL

Escape hatch explícito:

```php
Schema::raw(
    TrustedSchemaSql::fromTrustedString(...)
);
```

No:

```php
Schema::raw($_POST['sql']);
```

---

# 226. Raw boundary

Debe propagarse hasta AST:

```text
RawSchemaOperationNode
```

para que:

```text
planner
security
migration safety
telemetry
diagnostics
```

conozcan la pérdida de analizabilidad.

---

# 227. Persistent runtime

Builder mutable:

```text
operation-scoped only
```

Registry:

```text
frozen/shared allowed
```

Configuration:

```text
immutable snapshot/shared allowed
```

---

# 228. FrankenPHP lifecycle

```text
Request A
├── SchemaBuilder A
└── Blueprints A

Request B
├── SchemaBuilder B
└── Blueprints B
```

Nunca:

```text
Blueprint A
→ Request B
```

---

# 229. Reset

Idealmente no se deberá necesitar resetear un Builder:

```text
create new operation-scoped builder
```

es más seguro.

---

# 230. RoadRunner/OpenSwoole

La misma arquitectura permite adaptarse a:

```text
RoadRunner
OpenSwoole
```

sin modificar la semántica del Builder.

---

# 231. Concurrency

Un `TableBlueprint` no deberá modificarse concurrentemente.

Regla V1:

```text
one blueprint
=
one active builder context
```

---

# 232. Async

No existe beneficio en convertir la construcción del AST en async.

```text
Schema building
=
CPU/local-memory operation
```

---

# 233. Telemetry

Eventos posibles:

```text
SchemaBuildStarted
SchemaTableBlueprintCreated
SchemaColumnDeclared
SchemaConstraintDeclared
SchemaBuildCompleted
SchemaBuildFailed
SchemaBlueprintSealed
SchemaBuilderBudgetExceeded
SchemaBuilderExtensionUsed
```

---

# 234. Hot path telemetry

No debe instrumentarse cada fluent call de forma costosa por defecto.

Preferir:

```text
aggregate build telemetry
```

---

# 235. Sensitive metadata

No registrar automáticamente:

```text
raw SQL
comments
default values
application-specific identifiers
```

cuando puedan contener información sensible.

---

# 236. Performance objective

Builder debe ser esencialmente:

```text
O(n)
```

respecto al número de definiciones, salvo validaciones que requieran estructuras adicionales.

---

# 237. Lookup structures

Para detectar duplicados:

```text
ColumnIdentifier → ColumnBlueprint
ConstraintIdentifier → ConstraintBlueprint
IndexIdentifier → IndexBlueprint
```

pueden mantenerse mapas operation-scoped.

---

# 238. Memory

Aproximadamente:

```text
Memory(Build)
=
O(
    Columns
    +
    Indexes
    +
    Constraints
    +
    Expressions
)
```

---

# 239. Lowering performance

```text
Blueprints
→ Definitions
→ AST
```

deberá evitar reflection en hot paths cuando sea posible.

---

# 240. Testing strategy

El sistema deberá poder probarse sin base de datos.

---

# 241. Unit test example

```php
$ast = $builder->buildCreate(
    'users',
    function (TableBlueprint $table): void {
        $table->id();
        $table->string('email')->unique();
    }
);

self::assertInstanceOf(
    CreateTableNode::class,
    $ast->nodes()->first()
);
```

---

# 242. No SQL assertion in Builder tests

Incorrecto:

```php
self::assertSame(
    'CREATE TABLE ...',
    $builder->build(...)
);
```

Eso pertenece a compiler tests.

---

# 243. Builder tests should assert

```text
node kinds
typed definitions
identifiers
types
defaults
constraints
foreign keys
modifiers
metadata
diagnostics
lowering
```

---

# 244. Cross-platform tests

Builder output debería ser mayormente idéntico para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando la intención semántica es la misma.

---

# 245. Platform-specific extensions

Sólo los elementos explícitamente específicos deberán variar.

---

# 246. Snapshot testing

Puede utilizarse serialization estable del AST para snapshot tests.

---

# 247. Property-based testing

Útil para:

```text
identifier generation
modifier combinations
decimal precision/scale
foreign key arity
normalization
determinism
```

---

# 248. Conformance suite

Todo Builder extension deberá pasar pruebas de:

```text
determinism
sealing
validation
serialization compatibility
AST lowering
persistent runtime isolation
```

---

# 249. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Builder
```

---

# 250. Proposed directory structure

```text
Schema/
└── Builder/
    ├── Contract/
    │   ├── SchemaBuilder.php
    │   ├── SchemaBuilderFactory.php
    │   ├── SchemaOperationSink.php
    │   └── SchemaFragment.php
    │
    ├── Manager/
    │   └── SchemaManager.php
    │
    ├── Blueprint/
    │   ├── Table/
    │   │   ├── TableBlueprint.php
    │   │   ├── CreateTableBlueprint.php
    │   │   ├── AlterTableBlueprint.php
    │   │   └── TableBlueprintMode.php
    │   │
    │   ├── Column/
    │   │   ├── ColumnBlueprint.php
    │   │   ├── ColumnAlteration.php
    │   │   └── ColumnModifierSet.php
    │   │
    │   ├── Index/
    │   │   ├── IndexBlueprint.php
    │   │   └── IndexKeyBlueprint.php
    │   │
    │   ├── Constraint/
    │   │   ├── ConstraintBlueprint.php
    │   │   ├── ForeignKeyBlueprint.php
    │   │   ├── PrimaryKeyBlueprint.php
    │   │   ├── UniqueConstraintBlueprint.php
    │   │   └── CheckConstraintBlueprint.php
    │   │
    │   ├── Sequence/
    │   │   └── SequenceBlueprint.php
    │   │
    │   └── View/
    │       └── ViewBlueprint.php
    │
    ├── Column/
    │   ├── ColumnFactory.php
    │   ├── ColumnTypeFactory.php
    │   ├── ColumnDefaultFactory.php
    │   └── IdentifierGenerationStrategy.php
    │
    ├── Definition/
    │   ├── BlueprintDefinitionFactory.php
    │   ├── TableDefinitionFactory.php
    │   ├── ColumnDefinitionFactory.php
    │   ├── IndexDefinitionFactory.php
    │   └── ConstraintDefinitionFactory.php
    │
    ├── Lowering/
    │   ├── SchemaBlueprintLowerer.php
    │   ├── CreateTableLowerer.php
    │   ├── AlterTableLowerer.php
    │   ├── ColumnLowerer.php
    │   ├── IndexLowerer.php
    │   ├── ConstraintLowerer.php
    │   └── SchemaLoweringResult.php
    │
    ├── Naming/
    │   ├── SchemaNamingStrategy.php
    │   ├── SchemaNamingConventionResolver.php
    │   ├── DefaultSchemaNamingStrategy.php
    │   └── DefaultSchemaNamingConventionResolver.php
    │
    ├── Fragment/
    │   ├── SchemaFragmentRegistry.php
    │   └── CompositeSchemaFragment.php
    │
    ├── Context/
    │   ├── SchemaBuildContext.php
    │   ├── SchemaBuildConfiguration.php
    │   └── SchemaExecutionTarget.php
    │
    ├── Metadata/
    │   ├── BlueprintId.php
    │   ├── BlueprintMetadata.php
    │   └── BlueprintSourceLocation.php
    │
    ├── Validation/
    │   ├── SchemaBuilderValidator.php
    │   ├── TableBlueprintValidator.php
    │   ├── ColumnBlueprintValidator.php
    │   ├── IndexBlueprintValidator.php
    │   └── ForeignKeyBlueprintValidator.php
    │
    ├── Extension/
    │   ├── SchemaBuilderExtension.php
    │   ├── SchemaBuilderExtensionRegistry.php
    │   ├── SchemaBuilderExtensionDescriptor.php
    │   └── SchemaBuilderExtensionResolver.php
    │
    ├── Sink/
    │   ├── MigrationOperationCollector.php
    │   ├── PreviewOperationCollector.php
    │   ├── TestingOperationCollector.php
    │   └── ImmediateSchemaOrchestrator.php
    │
    ├── Budget/
    │   └── SchemaBuilderBudget.php
    │
    ├── Diagnostic/
    │   ├── SchemaBuildDiagnostics.php
    │   ├── SchemaBuilderDiagnostic.php
    │   └── SchemaBuilderDiagnosticCode.php
    │
    ├── Result/
    │   ├── SchemaBuildResult.php
    │   └── SchemaBuildMetadata.php
    │
    └── Exception/
        ├── DatabaseSchemaBuilderException.php
        ├── InvalidSchemaBuilderStateException.php
        ├── InvalidBlueprintException.php
        ├── SealedBlueprintMutationException.php
        ├── DuplicateColumnDefinitionException.php
        ├── DuplicateIndexDefinitionException.php
        ├── DuplicateConstraintDefinitionException.php
        ├── InvalidColumnDefinitionException.php
        ├── InvalidIndexDefinitionException.php
        ├── InvalidForeignKeyDefinitionException.php
        ├── InvalidTableOptionException.php
        ├── InvalidSchemaIdentifierException.php
        ├── ConflictingColumnModifierException.php
        ├── UnsupportedBuilderOperationException.php
        ├── SchemaBuilderExtensionException.php
        ├── SchemaBuilderBudgetExceededException.php
        ├── SchemaBuilderSecurityException.php
        └── SchemaBuilderInvariantException.php
```

---

# 251. Dependency rules

Permitido:

```text
Schema Builder
→ Schema AST
→ Database Type System
→ Schema Expressions
→ Identifier abstractions
→ immutable configuration
→ immutable extension registry
```

No permitido:

```text
Schema Builder
→ PDO
Schema Builder
→ Driver execution
Schema Builder
→ QueryExecutor
Schema Builder
→ ORM EntityManager
Schema Builder
→ Migration Repository
Schema Builder
→ global TenantContext
Schema Builder
→ HTTP Request
```

---

# 252. Architectural invariants

## DB-SCHEMA-BUILDER-001

Schema Builder será una DSL de construcción estructural.

## DB-SCHEMA-BUILDER-002

Schema Builder no ejecutará SQL.

## DB-SCHEMA-BUILDER-003

Schema Builder no abrirá conexiones.

## DB-SCHEMA-BUILDER-004

Schema Builder no realizará hidden introspection.

## DB-SCHEMA-BUILDER-005

Schema Builder no generará SQL directamente.

## DB-SCHEMA-BUILDER-006

Schema Builder producirá typed structural definitions.

## DB-SCHEMA-BUILDER-007

Typed definitions podrán convertirse en Schema AST.

## DB-SCHEMA-BUILDER-008

Builder será distinto de AST.

## DB-SCHEMA-BUILDER-009

Builder será distinto de Schema Model.

## DB-SCHEMA-BUILDER-010

Builder será distinto de Migration.

## DB-SCHEMA-BUILDER-011

Builder será distinto de Schema Compiler.

## DB-SCHEMA-BUILDER-012

Builder será distinto de Execution Engine.

## DB-SCHEMA-BUILDER-013

Blueprints podrán ser mutables durante construcción.

## DB-SCHEMA-BUILDER-014

Blueprints serán operation-scoped.

## DB-SCHEMA-BUILDER-015

Blueprints podrán sellarse.

## DB-SCHEMA-BUILDER-016

Blueprint sellado no aceptará mutaciones.

## DB-SCHEMA-BUILDER-017

AST producido será inmutable.

## DB-SCHEMA-BUILDER-018

No existirá global current blueprint.

## DB-SCHEMA-BUILDER-019

No existirá global current table.

## DB-SCHEMA-BUILDER-020

No existirá global mutable builder.

## DB-SCHEMA-BUILDER-021

Column types serán tipados.

## DB-SCHEMA-BUILDER-022

Column types no serán SQL strings por defecto.

## DB-SCHEMA-BUILDER-023

Schema Builder reutilizará Database Type System.

## DB-SCHEMA-BUILDER-024

Builder no creará un segundo type system.

## DB-SCHEMA-BUILDER-025

NoDefault será distinto de DefaultNull.

## DB-SCHEMA-BUILDER-026

Default expression será estructurada.

## DB-SCHEMA-BUILDER-027

Raw default será explícito.

## DB-SCHEMA-BUILDER-028

Raw expression no será fallback automático.

## DB-SCHEMA-BUILDER-029

Identifier será distinto de runtime value.

## DB-SCHEMA-BUILDER-030

Identifiers serán estructurados.

## DB-SCHEMA-BUILDER-031

Builder no realizará identifier quoting.

## DB-SCHEMA-BUILDER-032

Builder no aceptará qualified column strings como column identifier local.

## DB-SCHEMA-BUILDER-033

CreateTableBlueprint y AlterTableBlueprint tendrán contextos diferenciados.

## DB-SCHEMA-BUILDER-034

Create context no permitirá drop de objetos inexistentes.

## DB-SCHEMA-BUILDER-035

Alter context podrá expresar cambios estructurales.

## DB-SCHEMA-BUILDER-036

RenameColumn preservará rename intent.

## DB-SCHEMA-BUILDER-037

RenameTable preservará rename intent.

## DB-SCHEMA-BUILDER-038

DropColumn será operación explícita.

## DB-SCHEMA-BUILDER-039

DropTable será operación explícita.

## DB-SCHEMA-BUILDER-040

Cascade será explícito.

## DB-SCHEMA-BUILDER-041

Cascade no será inferido.

## DB-SCHEMA-BUILDER-042

foreignId no implicará automáticamente FK salvo DSL explícita.

## DB-SCHEMA-BUILDER-043

constrained() podrá usar convenciones explícitas.

## DB-SCHEMA-BUILDER-044

Convention inference será determinista.

## DB-SCHEMA-BUILDER-045

Convention inference será overrideable.

## DB-SCHEMA-BUILDER-046

Naming strategy será distinta de naming convention resolver.

## DB-SCHEMA-BUILDER-047

Generated object names serán deterministas.

## DB-SCHEMA-BUILDER-048

Generated names no dependerán de request state.

## DB-SCHEMA-BUILDER-049

Generated names no dependerán del clock.

## DB-SCHEMA-BUILDER-050

Duplicate columns fallarán.

## DB-SCHEMA-BUILDER-051

Duplicate indexes conflictivos fallarán.

## DB-SCHEMA-BUILDER-052

Duplicate constraints conflictivos fallarán.

## DB-SCHEMA-BUILDER-053

Contradictory modifiers fallarán.

## DB-SCHEMA-BUILDER-054

No se usará last-call-wins para contradicciones.

## DB-SCHEMA-BUILDER-055

Precision y scale serán validadas.

## DB-SCHEMA-BUILDER-056

Foreign key arity será validada.

## DB-SCHEMA-BUILDER-057

Self-referencing FK será representable.

## DB-SCHEMA-BUILDER-058

Circular FK no será automáticamente inválida.

## DB-SCHEMA-BUILDER-059

Index expressions serán estructuradas.

## DB-SCHEMA-BUILDER-060

Partial index predicates serán estructurados.

## DB-SCHEMA-BUILDER-061

Index options serán capability-aware.

## DB-SCHEMA-BUILDER-062

Table options serán capability-aware.

## DB-SCHEMA-BUILDER-063

Platform-specific options serán explícitas.

## DB-SCHEMA-BUILDER-064

MySQL-specific options no serán universales.

## DB-SCHEMA-BUILDER-065

MariaDB-specific behavior será first-class.

## DB-SCHEMA-BUILDER-066

PostgreSQL-specific behavior será first-class.

## DB-SCHEMA-BUILDER-067

SQLite-specific behavior será first-class.

## DB-SCHEMA-BUILDER-068

Builder no usará scattered vendor conditionals.

## DB-SCHEMA-BUILDER-069

Platform capabilities se resolverán downstream.

## DB-SCHEMA-BUILDER-070

Unsupported operation no se emulará dentro del Builder.

## DB-SCHEMA-BUILDER-071

SQLite table rebuild no pertenecerá al Builder.

## DB-SCHEMA-BUILDER-072

Concurrent index strategy no pertenecerá al core structural definition.

## DB-SCHEMA-BUILDER-073

DDL transaction strategy no pertenecerá al Builder.

## DB-SCHEMA-BUILDER-074

Builder no iniciará transaction.

## DB-SCHEMA-BUILDER-075

Builder no realizará rollback.

## DB-SCHEMA-BUILDER-076

Builder no realizará data backfill.

## DB-SCHEMA-BUILDER-077

Builder no ejecutará UPDATE.

## DB-SCHEMA-BUILDER-078

Schema Builder podrá coexistir con data migration orchestration.

## DB-SCHEMA-BUILDER-079

Schema Facade podrá agregar Builder e Inspector APIs.

## DB-SCHEMA-BUILDER-080

Builder e Inspector seguirán siendo servicios internos distintos.

## DB-SCHEMA-BUILDER-081

hasTable no será responsabilidad interna del Builder.

## DB-SCHEMA-BUILDER-082

hasColumn no será responsabilidad interna del Builder.

## DB-SCHEMA-BUILDER-083

Connection name será distinto de live Connection.

## DB-SCHEMA-BUILDER-084

Builder podrá conservar execution target lógico.

## DB-SCHEMA-BUILDER-085

Builder no adquirirá connection lease.

## DB-SCHEMA-BUILDER-086

Builder no dependerá de global tenant.

## DB-SCHEMA-BUILDER-087

Multitenancy integration será opcional.

## DB-SCHEMA-BUILDER-088

Core Database no dependerá de Multitenancy.

## DB-SCHEMA-BUILDER-089

Builder output será determinista bajo mismas entradas.

## DB-SCHEMA-BUILDER-090

Builder output no dependerá de hidden environment state.

## DB-SCHEMA-BUILDER-091

Builder output no dependerá de current time implícito.

## DB-SCHEMA-BUILDER-092

Current timestamp será SchemaExpression explícita.

## DB-SCHEMA-BUILDER-093

Semantic enum será distinto de native platform enum.

## DB-SCHEMA-BUILDER-094

JSON será semantic database type.

## DB-SCHEMA-BUILDER-095

Decimal no se convertirá a float.

## DB-SCHEMA-BUILDER-096

Custom types usarán Type Registry.

## DB-SCHEMA-BUILDER-097

Builder extensions serán tipadas.

## DB-SCHEMA-BUILDER-098

Builder extension registry será frozen.

## DB-SCHEMA-BUILDER-099

Builder extensions no usarán last-wins.

## DB-SCHEMA-BUILDER-100

Extensions no ejecutarán queries.

## DB-SCHEMA-BUILDER-101

Extensions no abrirán conexiones.

## DB-SCHEMA-BUILDER-102

Extensions no introducirán raw SQL implícitamente.

## DB-SCHEMA-BUILDER-103

Mutable macros globales no serán mecanismo principal de extensión.

## DB-SCHEMA-BUILDER-104

SchemaFragment favorecerá composición.

## DB-SCHEMA-BUILDER-105

SchemaFragment respetará determinismo.

## DB-SCHEMA-BUILDER-106

SchemaFragment no ejecutará DB I/O.

## DB-SCHEMA-BUILDER-107

BlueprintId será distinto de SchemaNodeId.

## DB-SCHEMA-BUILDER-108

BlueprintId podrá mapearse a SchemaNodeId.

## DB-SCHEMA-BUILDER-109

Source location podrá propagarse al AST.

## DB-SCHEMA-BUILDER-110

Build será distinto de execute.

## DB-SCHEMA-BUILDER-111

SchemaBuildResult podrá contener AST y diagnostics.

## DB-SCHEMA-BUILDER-112

Migration Runner podrá recopilar múltiples schema operations.

## DB-SCHEMA-BUILDER-113

Operation collection permitirá global dependency planning.

## DB-SCHEMA-BUILDER-114

Immediate-looking API no implicará immediate low-level SQL execution.

## DB-SCHEMA-BUILDER-115

Operation Sink será explícito.

## DB-SCHEMA-BUILDER-116

Preview no será responsabilidad del Builder.

## DB-SCHEMA-BUILDER-117

Dry-run no será responsabilidad del Builder.

## DB-SCHEMA-BUILDER-118

Compilation no será responsabilidad del Builder.

## DB-SCHEMA-BUILDER-119

Execution no será responsabilidad del Builder.

## DB-SCHEMA-BUILDER-120

Builder validation será local y estructural.

## DB-SCHEMA-BUILDER-121

Schema-aware validation requerirá snapshot explícito.

## DB-SCHEMA-BUILDER-122

Builder nunca hará hidden introspection.

## DB-SCHEMA-BUILDER-123

Blueprint budgets serán explícitos.

## DB-SCHEMA-BUILDER-124

Budget overflow fallará.

## DB-SCHEMA-BUILDER-125

Budget overflow no truncará definiciones.

## DB-SCHEMA-BUILDER-126

Raw schema SQL será trust boundary explícita.

## DB-SCHEMA-BUILDER-127

Raw SQL deberá propagarse como raw AST node.

## DB-SCHEMA-BUILDER-128

Raw SQL no se volverá portable automáticamente.

## DB-SCHEMA-BUILDER-129

DDL syntax safety será distinta de authorization.

## DB-SCHEMA-BUILDER-130

Builder no decidirá authorization.

## DB-SCHEMA-BUILDER-131

Builder mutable será request/operation scoped.

## DB-SCHEMA-BUILDER-132

Frozen registries podrán compartirse en persistent runtimes.

## DB-SCHEMA-BUILDER-133

FrankenPHP no compartirá mutable blueprints entre requests.

## DB-SCHEMA-BUILDER-134

RoadRunner no compartirá mutable blueprints entre requests.

## DB-SCHEMA-BUILDER-135

OpenSwoole no compartirá mutable blueprints entre coroutines.

## DB-SCHEMA-BUILDER-136

Blueprint concurrent mutation no será soportada en V1.

## DB-SCHEMA-BUILDER-137

Schema building no requerirá async.

## DB-SCHEMA-BUILDER-138

Telemetry no alterará builder semantics.

## DB-SCHEMA-BUILDER-139

Telemetry no registrará raw schema content por defecto.

## DB-SCHEMA-BUILDER-140

Builder será testeable sin DB.

## DB-SCHEMA-BUILDER-141

Builder tests no dependerán de SQL output.

## DB-SCHEMA-BUILDER-142

Cross-platform semantic builder output será estable cuando aplique.

## DB-SCHEMA-BUILDER-143

Lowering será determinista.

## DB-SCHEMA-BUILDER-144

Lowering preservará structural intent.

## DB-SCHEMA-BUILDER-145

Modifier order no cambiará semántica cuando los modifiers sean conmutativos.

## DB-SCHEMA-BUILDER-146

Builder no perderá rename intent.

## DB-SCHEMA-BUILDER-147

Builder no perderá destructive operation intent.

## DB-SCHEMA-BUILDER-148

Builder no inventará schema facts.

## DB-SCHEMA-BUILDER-149

Builder no inventará platform capabilities.

## DB-SCHEMA-BUILDER-150

Schema Builder será la API declarativa principal para construir Schema AST desde código PHP.

---

# 253. Anti-patterns

## 253.1 SQL generation inside Builder

Incorrecto:

```php
public function string(string $name): string
{
    return "{$name} VARCHAR(255)";
}
```

Correcto:

```text
string()
↓
ColumnBlueprint
↓
ColumnDefinition
```

---

## 253.2 Execute from Schema::create()

Arquitectónicamente incorrecto:

```text
Schema::create()
↓
PDO::exec()
```

Correcto:

```text
Schema::create()
↓
Builder
↓
AST
↓
Orchestration
↓
Planner
↓
Compiler
↓
Execution
```

---

## 253.3 Global current table

Incorrecto:

```php
static $currentTable;
```

---

## 253.4 Vendor-specific SQL types

Incorrecto:

```php
$table->column('data', 'JSONB');
```

como API portable.

---

## 253.5 Hidden introspection

Incorrecto:

```php
if (!$connection->hasColumn(...)) {
    $table->string(...);
}
```

dentro del Builder.

---

## 253.6 Last modifier wins

Incorrecto:

```php
$table->string('name')
    ->nullable()
    ->notNull();
```

y aceptar silenciosamente `notNull`.

Debe producir conflicto.

---

## 253.7 Implicit raw SQL

Incorrecto:

```php
$table->default('CURRENT_TIMESTAMP');
```

y asumir que la cadena es expresión SQL.

Correcto:

```php
->default('CURRENT_TIMESTAMP')
```

significa literal string.

Mientras:

```php
->defaultExpression(
    SchemaExpression::currentTimestamp()
)
```

significa expresión.

---

## 253.8 Magic foreign key

Incorrecto:

```php
$table->integer('user_id');
```

y crear FK automáticamente.

---

## 253.9 Platform emulation inside Builder

Incorrecto:

```php
if ($sqlite) {
    rebuildTable();
}
```

---

## 253.10 Mutable extension macros in workers

Incorrecto:

```php
Schema::macro('foo', ...);
```

durante requests persistentes.

---

# 254. Example — Create users

```php
Schema::create('users', function (TableBlueprint $table): void {
    $table->id();

    $table
        ->string('email', 320)
        ->unique();

    $table
        ->string('name', 150);

    $table
        ->boolean('active')
        ->default(true);

    $table->timestamps();
});
```

Blueprint:

```text
CreateTableBlueprint(users)
├── id
├── email
├── name
├── active
├── created_at
├── updated_at
└── unique(email)
```

Lowering:

```text
CreateTableNode
└── TableDefinition(users)
    ├── ColumnDefinition(id)
    ├── ColumnDefinition(email)
    ├── ColumnDefinition(name)
    ├── ColumnDefinition(active)
    ├── ColumnDefinition(created_at)
    ├── ColumnDefinition(updated_at)
    ├── PrimaryKey(id)
    └── Unique(email)
```

---

# 255. Example — Relationships

```php
Schema::create('posts', function (TableBlueprint $table): void {
    $table->id();

    $table
        ->foreignId('user_id')
        ->constrained('users')
        ->cascadeOnDelete();

    $table->string('title', 255);
    $table->text('content');
    $table->timestamps();
});
```

Produces:

```text
TableDefinition(posts)
├── id
├── user_id
├── title
├── content
├── created_at
├── updated_at
├── PK(id)
└── FK
    ├── posts.user_id
    ├── users.id
    └── ON DELETE CASCADE
```

---

# 256. Example — Composite key

```php
Schema::create('tenant_users', function (TableBlueprint $table): void {
    $table->uuid('tenant_id');
    $table->uuid('user_id');

    $table->primary([
        'tenant_id',
        'user_id',
    ]);
});
```

---

# 257. Example — Alter

```php
Schema::table('users', function (AlterTableBlueprint $table): void {
    $table
        ->string('nickname', 100)
        ->nullable();

    $table->renameColumn(
        'name',
        'full_name'
    );

    $table->dropColumn('legacy_code');
});
```

AST:

```text
AlterTableNode(users)
├── AddColumn(nickname)
├── RenameColumn(name → full_name)
└── DropColumn(legacy_code)
```

---

# 258. Example — Explicit alteration

```php
Schema::table('users', function (AlterTableBlueprint $table): void {
    $table->alterColumn(
        'email',
        function (ColumnAlteration $column): void {
            $column->type(
                DatabaseType::string(500)
            );

            $column->notNull();
        }
    );
});
```

Result:

```text
AlterColumn
├── email
└── changes
    ├── type
    │   └── String(500)
    ├── nullability
    │   └── NOT_NULL
    └── default
        └── UNCHANGED
```

---

# 259. Example — Index

```php
Schema::table('users', function (AlterTableBlueprint $table): void {
    $table->index(
        ['status', 'created_at'],
        'users_status_created_idx'
    );
});
```

---

# 260. Example — Advanced index

```php
Schema::table('users', function (AlterTableBlueprint $table): void {
    $table->index(function (IndexBlueprint $index): void {
        $index->column('status');

        $index
            ->column('created_at')
            ->descending();

        $index->where(
            SchemaExpression::equals(
                SchemaExpression::column('active'),
                SchemaExpression::literal(true)
            )
        );
    });
});
```

---

# 261. Example — Check constraint

```php
Schema::create('products', function (TableBlueprint $table): void {
    $table->id();

    $table->decimal(
        'price',
        precision: 18,
        scale: 2
    );

    $table->check(
        SchemaExpression::greaterThanOrEqual(
            SchemaExpression::column('price'),
            SchemaExpression::literal(0)
        )
    );
});
```

---

# 262. Example — Schema fragment

```php
final class AuditColumns implements SchemaFragment
{
    public function apply(TableBlueprint $table): void
    {
        $table->timestamps();

        $table
            ->uuid('created_by')
            ->nullable();

        $table
            ->uuid('updated_by')
            ->nullable();
    }
}
```

Uso:

```php
Schema::create('orders', function (TableBlueprint $table): void {
    $table->id();
    $table->string('number');

    $table->use(
        new AuditColumns()
    );
});
```

---

# 263. Example — Operation collection

Migration:

```php
public function up(): void
{
    Schema::create('users', ...);

    Schema::create('posts', ...);

    Schema::table('posts', function ($table): void {
        $table
            ->foreign('user_id')
            ->references('id')
            ->on('users');
    });
}
```

Internamente:

```text
MigrationOperationCollector
        │
        ▼
SchemaAst
├── CreateTable(users)
├── CreateTable(posts)
└── AddForeignKey(posts.user_id → users.id)
        │
        ▼
Dependency Analysis
        │
        ▼
Schema Planner
```

Esto permite que VoltStack comprenda la migration como una unidad estructural completa.

---

# 264. Master formula

```text
Schema Builder System
=
Developer DSL
+
Blueprint Construction
+
Typed Column Definitions
+
Typed Index Definitions
+
Typed Constraint Definitions
+
Relationship DSL
+
Naming Conventions
+
Schema Fragments
+
Local Validation
+
Deterministic Lowering
+
Schema AST Generation
+
Extension Control
+
Security Boundaries
+
Budgets
+
Diagnostics
+
Persistent Runtime Isolation
```

---

# 265. Correctness formula

```text
CorrectSchemaBuilder
=
Ergonomic
∧
Typed
∧
Deterministic
∧
IntentPreserving
∧
PlatformNeutral
∧
CapabilityHonest
∧
ExtensionSafe
∧
SecurityAware
∧
RuntimeIsolated
```

---

# 266. Lowering invariant

Para una entrada `B`:

```text
AST = Lower(B)
```

deberá cumplirse:

```text
Intent(AST)
=
Intent(B)
```

---

# 267. Determinism invariant

```text
Build(
    Input,
    Configuration,
    Extensions
)
```

con las mismas entradas deberá producir:

```text
Equivalent Schema AST
```

---

# 268. Responsibility formula

```text
Schema Builder
=
How developer expresses the change

Schema AST
=
What structural change was requested

Schema Planner
=
How that change can safely be organized

Schema Compiler
=
How that plan is represented for the target database

Execution Engine
=
How the compiled operation is executed
```

---

# 269. Architectural result

VoltStack obtiene una API familiar:

```php
Schema::create('users', function ($table) {
    $table->id();
    $table->string('email')->unique();
    $table->timestamps();
});
```

sin sacrificar una arquitectura interna formal:

```text
Developer API
      ↓
Blueprint
      ↓
Typed Definition
      ↓
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
Schema Compiler
      ↓
Execution Engine
```

Esto permite combinar:

```text
Laravel-like Developer Experience
+
Doctrine/Symfony-like Architectural Separation
+
VoltStack Query/Compiler/Execution Architecture
+
Persistent Runtime Safety
```

sin copiar directamente ninguno de esos modelos.

---

# 270. Decisión arquitectónica

VoltStack adoptará un:

```text
Typed Fluent Schema Builder
```

como API principal para definición estructural desde PHP.

El Builder será:

```text
ergonomic
fluent
strongly typed internally
deterministic
operation-scoped
extension-aware
capability-aware
AST-producing
persistent-runtime safe
```

y explícitamente no será:

```text
SQL generator
database executor
schema inspector
migration repository
transaction manager
ORM
```

---

# 271. Regla maestra

> **El desarrollador describe el esquema mediante una DSL cómoda; VoltStack transforma esa descripción en intención estructural tipada antes de decidir cualquier detalle SQL o de ejecución.**

En forma compacta:

```text
Developer Convenience
        +
Architectural Precision
        =
VoltStack Schema Builder
```

---

# 272. Siguiente documento

```text
91_DATABASE_TABLE_DEFINITION_SYSTEM.md
```

El siguiente documento deberá formalizar `TableDefinition` como representación tipada de una tabla deseada o declarada, incluyendo:

```text
TableDefinition
├── TableIdentifier
├── ColumnDefinitionSet
├── PrimaryKeyDefinition
├── IndexDefinitionSet
├── ConstraintDefinitionSet
├── ForeignKeyDefinitionSet
├── TableOptionSet
├── TableMetadata
└── ExtensionMetadata
```

y deberá establecer claramente:

```text
TableDefinition
≠
TableBlueprint
≠
Schema Model Table
≠
CreateTableNode
≠
SQL CREATE TABLE
```

Su principio central será:

> **TableBlueprint construye; TableDefinition describe; Schema AST expresa la operación; Schema Compiler representa la operación para una plataforma concreta.**