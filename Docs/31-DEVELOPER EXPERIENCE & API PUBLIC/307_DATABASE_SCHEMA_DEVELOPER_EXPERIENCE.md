# 307_DATABASE_SCHEMA_DEVELOPER_EXPERIENCE.md

# VoltStack Quantum Database
## Database Schema Developer Experience

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 307 — Database Schema Developer Experience  
**Bloque:** 31 — Developer Experience / Public API  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `306_DATABASE_QUERY_DEVELOPER_EXPERIENCE.md`  
**Siguiente documento:** `308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md`

---

# 1. Propósito

Este documento define la experiencia oficial de desarrollo para describir, crear, inspeccionar, comparar y evolucionar estructuras de base de datos mediante `VoltStack/Quantum/Database`.

La experiencia de Schema deberá proporcionar una API sencilla para operaciones comunes:

```php
Schema::create('users', function (TableBlueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamps();
});
```

sin convertir el Schema Builder en:

- un generador de strings SQL;
- un executor;
- un migration runner;
- un introspector;
- un driver;
- un ORM;
- un sistema de detección de capabilities improvisado.

La regla central será:

> **Schema Builder expresa estructura e intención de transformación; no genera SQL directamente, no ejecuta DDL y no decide por sí mismo cómo representar físicamente una operación en cada DBMS.**

---

# 2. Objetivo de Developer Experience

La experiencia deberá combinar:

```text
Laravel-like ergonomics
        +
Typed Schema Model
        +
Schema AST
        +
Platform Capabilities
        +
Schema Diff
        +
Migration Safety
        +
VoltStack Diagnostics
```

El desarrollador deberá poder escribir:

```php
Schema::table('users', function (TableBlueprint $table) {
    $table->string('phone')->nullable();
});
```

mientras internamente ocurre:

```text
Developer Intent
      ↓
Schema Builder
      ↓
Schema Definition / Schema AST
      ↓
Validation
      ↓
Capability Analysis
      ↓
Schema Planning
      ↓
Schema Compilation
      ↓
Execution
```

---

# 3. Principios fundamentales

Debe mantenerse:

```text
Schema Builder
≠
SQL Builder
```

```text
Schema Model
≠
Schema AST
```

```text
Schema Definition
≠
Migration
```

```text
Schema Diff
≠
Migration
```

```text
Schema Compiler
≠
Schema Executor
```

```text
Schema Introspection
≠
Schema Definition
```

```text
Database Constraint
≠
Database Index
```

```text
Unique Constraint
≠
Unique Index
```

```text
Primary Key
≠
Index
```

y:

```text
Platform Version
≠
Platform Capability
```

---

# 4. Arquitectura general

```text
Application / Migration
          │
          ▼
      Schema API
          │
          ▼
    Schema Builder
          │
          ├──────────────┐
          ▼              ▼
Schema Definition     Schema AST
          │              │
          └──────┬───────┘
                 ▼
           Validation
                 │
                 ▼
       Capability Analysis
                 │
                 ▼
          Schema Planner
                 │
                 ▼
         Schema Compiler
                 │
                 ▼
      Compiled Commands
                 │
                 ▼
        Execution Engine
                 │
                 ▼
           Connection
                 │
                 ▼
             Driver
                 │
                 ▼
              DBMS
```

La ruta inversa para observación será:

```text
DBMS
 ↓
Driver
 ↓
Schema Introspector
 ↓
Normalization
 ↓
Schema Model
```

Y la comparación:

```text
Current Schema Model
          +
Target Schema Model
          ↓
      Schema Diff
          ↓
Typed Differences
          ↓
Migration Planning
```

---

# 5. Superficies públicas

La experiencia podrá ofrecer:

```php
Schema::create(...);
Schema::table(...);
Schema::drop(...);
Schema::inspect(...);
Schema::hasTable(...);
Schema::hasColumn(...);
```

además de APIs inyectables:

```php
$schema->create(...);
```

y servicios de bajo nivel:

```php
$schemaManager->inspect(...);
$schemaDiffer->diff(...);
$schemaPlanner->plan(...);
```

---

# 6. Facade

La facade podrá permitir:

```php
Schema::create('users', ...);
```

pero:

```text
Schema Facade
≠
Static Schema State
```

La facade resolverá el contexto efectivo desde el container/scope.

---

# 7. API inyectable

Componentes internos y paquetes deberán poder utilizar:

```php
final class Installer
{
    public function __construct(
        private SchemaManager $schema,
    ) {}
}
```

sin depender obligatoriamente de la facade.

---

# 8. Helper opcional

Podrá existir:

```php
schema()->table(...);
```

si resulta coherente con el Helper System.

La existencia del helper no creará una segunda implementación.

---

# 9. Same Engine Rule

Estas superficies:

```text
Schema::
schema()
SchemaManager
Migration Schema API
Package Installer
```

deberán converger en la misma infraestructura Schema.

---

# 10. Creación de tablas

API principal:

```php
Schema::create('users', function (TableBlueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email');
});
```

Conceptualmente:

```text
CreateTableOperation
└── TableDefinition
    ├── id
    ├── name
    └── email
```

No:

```text
CREATE TABLE users (...)
```

durante construcción.

---

# 11. Table Blueprint

`TableBlueprint` será una superficie ergonómica para construir una definición estructurada.

Ejemplo:

```php
Schema::create('products', function (TableBlueprint $table) {
    $table->uuid('id')->primary();
    $table->string('sku', 100)->unique();
    $table->decimal('price', 12, 2);
    $table->boolean('active')->default(true);
    $table->timestamps();
});
```

---

# 12. Blueprint ≠ Schema Model

El Blueprint representa construcción/intención.

El Schema Model representa estructura normalizada.

Por tanto:

```text
Blueprint
≠
Schema Model
```

---

# 13. Blueprint ≠ AST

Aunque puedan estar relacionados:

```text
Blueprint
≠
Schema AST
```

El Builder podrá producir AST/definitions internamente.

---

# 14. Column API

VoltStack deberá proporcionar tipos comunes:

```php
$table->bigInteger('id');
$table->integer('age');
$table->string('name');
$table->text('description');
$table->boolean('active');
$table->decimal('amount', 12, 2);
$table->date('birth_date');
$table->dateTime('created_at');
$table->json('metadata');
$table->binary('payload');
$table->uuid('id');
```

---

# 15. Logical types

Estas APIs deberán representar:

```text
Database Logical Type
```

y no necesariamente el nombre físico exacto del DBMS.

---

# 16. Logical Type ≠ Physical Type

Debe mantenerse:

```text
VoltStack Logical Type
≠
MySQL Type
≠
PostgreSQL Type
≠
SQLite Storage Class
```

El compiler/platform mapping decidirá la representación física.

---

# 17. Column Definition

Una columna podrá contener:

```text
ColumnDefinition
├── name
├── logical type
├── nullability
├── default
├── generated semantics
├── collation
├── length
├── precision
├── scale
├── identity strategy
├── comments
└── platform extensions
```

según aplique.

---

# 18. Fluent modifiers

Ejemplo:

```php
$table->string('email', 320)
    ->nullable()
    ->comment('Primary contact email');
```

---

# 19. Modifier validation

No todas las combinaciones deberán considerarse válidas.

Por ejemplo:

```text
autoIncrement + arbitrary string
```

deberá rechazarse si no existe una semántica soportada.

---

# 20. Convenience columns

Podrán existir:

```php
$table->id();
$table->uuidId();
$table->timestamps();
$table->softDeletes();
$table->rememberToken();
```

cuando tengan una definición precisa.

---

# 21. Convenience ≠ hidden architecture

Por ejemplo:

```php
$table->timestamps();
```

será azúcar sintáctica para una definición conocida.

No una operación especial en el Driver.

---

# 22. Identifier API

Nombres de:

```text
tables
columns
indexes
constraints
schemas
```

deberán tratarse como identifiers.

---

# 23. Identifier validation

Debe mantenerse:

```text
Identifier
≠
SQL fragment
```

---

# 24. User input

No deberá utilizarse directamente:

```php
Schema::create($request->input('table'), ...);
```

sin validación explícita.

---

# 25. Primary Key

Ejemplo:

```php
$table->id();
```

podrá expandirse conceptualmente a:

```text
ColumnDefinition
+
PrimaryKeyConstraint
+
IdentityStrategy
```

según configuración.

---

# 26. Explicit Primary Key

También deberá permitirse:

```php
$table->uuid('id');
$table->primary('id');
```

o:

```php
$table->primary(['tenant_id', 'id']);
```

---

# 27. Composite keys

La arquitectura deberá representar claves compuestas sin tratarlas como caso accidental.

---

# 28. Primary Key ≠ Index

Aunque un DBMS pueda crear estructuras físicas asociadas:

```text
PrimaryKeyConstraint
≠
IndexDefinition
```

---

# 29. Unique constraints

Ejemplo:

```php
$table->unique('email');
```

o:

```php
$table->unique(
    ['tenant_id', 'email'],
    'uq_users_tenant_email'
);
```

---

# 30. Unique Constraint ≠ Unique Index

La API podrá ofrecer ambos conceptos si la plataforma los distingue.

VoltStack no deberá colapsarlos arquitectónicamente sólo porque algunos DBMS los implementen de forma similar.

---

# 31. Indexes

Ejemplo:

```php
$table->index('email');
```

```php
$table->index([
    'tenant_id',
    'created_at',
]);
```

---

# 32. Index Definition

Podrá incluir:

```text
name
columns
expressions
ordering
uniqueness
method
predicate
included columns
platform options
```

cuando las capabilities lo permitan.

---

# 33. Advanced index features

Features como:

```text
partial indexes
expression indexes
covering indexes
GIN/GiST
FULLTEXT
SPATIAL
```

deberán modelarse mediante capabilities/extensions apropiadas.

---

# 34. No vendor branching in user code

Idealmente no:

```php
if (DB::driver() === 'pgsql') {
    // schema...
}
```

para operaciones que puedan expresarse semánticamente.

---

# 35. Foreign Keys

Ejemplo:

```php
$table->foreign('customer_id')
    ->references('id')
    ->on('customers');
```

---

# 36. Foreign Key Definition

Representación:

```text
ForeignKeyConstraint
├── local columns
├── referenced table
├── referenced columns
├── update action
├── delete action
├── deferrability
└── name
```

según capabilities.

---

# 37. Convenience foreign ID

Podrá ofrecerse:

```php
$table->foreignId('customer_id')
    ->constrained('customers');
```

---

# 38. Convenience ≠ inference without evidence

Si el sistema infiere:

```text
customer_id → customers.id
```

la convención deberá estar documentada.

Cuando exista ambigüedad deberá requerirse información explícita.

---

# 39. Referential actions

API:

```php
->onDelete(ForeignKeyAction::CASCADE)
->onUpdate(ForeignKeyAction::RESTRICT)
```

preferiblemente mediante enums/value objects.

---

# 40. Strings convenience

Podrán aceptarse strings ergonómicos:

```php
->onDelete('cascade');
```

pero deberán normalizarse a tipos conocidos.

---

# 41. Check Constraints

Ejemplo:

```php
$table->check(
    DB::column('balance')->greaterThanOrEqual(0)
);
```

o una expresión schema específica equivalente.

---

# 42. Check ≠ arbitrary SQL

Debe favorecerse una representación estructurada.

Raw CHECK deberá ser escape hatch.

---

# 43. Defaults

Ejemplo:

```php
$table->boolean('active')
    ->default(true);
```

El valor:

```text
true
```

será un default literal tipado.

---

# 44. Default Expression

Debe diferenciarse:

```text
literal default
```

de:

```text
database expression default
```

Ejemplo:

```php
$table->dateTime('created_at')
    ->defaultExpression(
        SchemaExpression::currentTimestamp()
    );
```

---

# 45. Default string ≠ SQL expression

Debe mantenerse:

```text
"default text"
≠
CURRENT_TIMESTAMP
```

---

# 46. Generated Columns

Podrá existir:

```php
$table->decimal('total', 12, 2)
    ->generatedAs(
        SchemaExpression::multiply(
            'quantity',
            'unit_price'
        )
    );
```

si la plataforma lo soporta.

---

# 47. Generated column capabilities

Deberán distinguirse:

```text
virtual
stored
unsupported
emulated
unknown
```

según capabilities.

---

# 48. Comments

Podrán definirse:

```php
$table->string('email')
    ->comment('Primary contact email');
```

pero soporte/representación dependerá de plataforma.

---

# 49. Collation

Podrá utilizarse:

```php
$table->string('name')
    ->collation('...');
```

pero una collation es platform-sensitive.

---

# 50. Charset

Igualmente:

```text
charset
```

no deberá tratarse como propiedad universal idéntica.

---

# 51. Table options

Opciones específicas de plataforma deberán aislarse.

Por ejemplo:

```php
$table->platformOption(...);
```

o extensiones tipadas.

---

# 52. Platform options ≠ core semantics

Una opción específica de MySQL no deberá contaminar el modelo universal si no representa una semántica portable.

---

# 53. Alter table

API:

```php
Schema::table('users', function (TableBlueprint $table) {
    $table->string('phone')->nullable();
});
```

deberá producir una intención:

```text
AlterTable
└── AddColumn(phone)
```

---

# 54. Add Column

Ejemplo:

```php
$table->string('phone');
```

en contexto `Schema::table()` podrá convertirse en `AddColumnOperation`.

---

# 55. Explicit APIs

Para eliminar ambigüedad también podrán existir:

```php
$table->addColumn(...);
$table->dropColumn(...);
$table->renameColumn(...);
$table->alterColumn(...);
```

---

# 56. Drop Column

```php
$table->dropColumn('legacy_code');
```

deberá ser identificada como operación potencialmente destructiva.

---

# 57. Destructive operation

Debe mantenerse:

```text
Valid DDL
≠
Safe DDL
```

---

# 58. Schema Safety

La operación podrá ser:

```text
SAFE
CONDITIONALLY_SAFE
DESTRUCTIVE
REQUIRES_REWRITE
REQUIRES_LOCK
UNKNOWN
```

según análisis.

---

# 59. Unknown Safety

Critical:

```text
UNKNOWN
≠
SAFE
```

---

# 60. Rename Column

Ejemplo:

```php
$table->renameColumn(
    'fullname',
    'full_name'
);
```

deberá expresar explícitamente un rename.

---

# 61. Rename inference

Schema Diff no deberá inferir automáticamente un rename destructivo sólo porque:

```text
column A disappeared
column B appeared
```

---

# 62. Explicit rename preferred

Cuando el desarrollador conoce la intención, deberá poder declararla explícitamente.

---

# 63. Alter Column

Ejemplo:

```php
$table->string('name', 500)
    ->change();
```

podrá mantenerse como convenience.

Sin embargo, internamente deberá convertirse a:

```text
AlterColumnOperation
```

con estado objetivo explícito.

---

# 64. Partial alteration ambiguity

La arquitectura deberá evitar que:

```php
$table->string('name', 500)->change();
```

borre accidentalmente propiedades no mencionadas.

---

# 65. Alteration model

Se deberá definir claramente si `change()` expresa:

```text
complete target column definition
```

o:

```text
patch
```

VoltStack deberá favorecer una semántica explícita.

---

# 66. Recommended approach

Internamente:

```text
Current Column
+
Requested Changes
↓
Resolved Target Column
↓
Validated Alter Operation
```

cuando exista introspection confiable.

---

# 67. Introspection dependency

Si la operación necesita conocer estado actual y éste no está disponible:

```text
UNKNOWN
```

deberá preservarse.

---

# 68. Drop Table

Podrá existir:

```php
Schema::drop('legacy_users');
```

---

# 69. `dropIfExists`

Podrá existir:

```php
Schema::dropIfExists('legacy_users');
```

pero:

```text
IF EXISTS
≠
safe operation
```

Sólo evita un tipo de error.

---

# 70. Explicit destructive intent

En tooling crítico podrá requerirse:

```text
DestructiveOperationAcknowledgement
```

o policy equivalente.

---

# 71. Rename Table

```php
Schema::rename('users', 'accounts');
```

deberá representarse como operación específica.

---

# 72. Table existence

API:

```php
Schema::hasTable('users');
```

requiere introspection real.

Por tanto:

```text
hasTable()
```

sí puede producir I/O.

---

# 73. Construction vs Observation

Debe distinguirse:

```text
Schema::create(...)
```

como definición/operación,

de:

```text
Schema::hasTable(...)
```

como observación.

---

# 74. Clear API semantics

La documentación deberá indicar qué métodos:

```text
build
inspect
execute
```

para evitar I/O oculto.

---

# 75. `hasColumn`

```php
Schema::hasColumn(
    'users',
    'email'
);
```

consultará introspection.

---

# 76. Schema Inspection

Podrá existir:

```php
$table = Schema::inspect('users');
```

retornando:

```text
TableModel
```

normalizado.

---

# 77. Inspect database

API avanzada:

```php
$database = Schema::inspectDatabase();
```

podrá retornar:

```text
SchemaModel
```

---

# 78. Introspection result

La introspection deberá incluir coverage.

Ejemplo:

```text
TableModel
Coverage:
  columns: COMPLETE
  indexes: COMPLETE
  constraints: PARTIAL
  generated expressions: UNKNOWN
```

---

# 79. Critical observation rule

Debe mantenerse:

```text
Not Observed
≠
Absent
```

---

# 80. Introspection Coverage

Podrán utilizarse:

```text
COMPLETE
PARTIAL
UNKNOWN
UNSUPPORTED
```

según diseño definitivo.

---

# 81. `has*` under incomplete evidence

Si una operación no puede determinar existencia con certeza, no deberá convertir automáticamente:

```text
UNKNOWN
```

en:

```text
false
```

---

# 82. Tri-state APIs

Infraestructura avanzada podrá utilizar:

```text
YES
NO
UNKNOWN
```

aunque helpers ergonómicos puedan ofrecer políticas explícitas.

---

# 83. Schema Model

El Schema Model deberá ser:

```text
normalized
typed
immutable
platform-aware where necessary
```

pero no un volcado de SQL vendor-specific.

---

# 84. Model hierarchy

Conceptualmente:

```text
DatabaseSchemaModel
├── Namespace/Schema
│   ├── Table
│   │   ├── Columns
│   │   ├── Constraints
│   │   ├── Indexes
│   │   └── Options
│   ├── Views
│   └── Sequences
└── Metadata
```

según capacidades V1/futuras.

---

# 85. Schema AST

El AST describirá transformaciones estructurales.

Ejemplo:

```text
CreateTableNode
AddColumnNode
DropColumnNode
RenameColumnNode
AlterColumnNode
AddIndexNode
DropIndexNode
AddConstraintNode
DropConstraintNode
```

---

# 86. AST ≠ migration file

Critical:

```text
Schema AST
≠
Migration
```

Una migration puede producir múltiples operaciones AST.

---

# 87. Schema Validation

Antes de compilación podrán validarse:

- identifiers;
- duplicate columns;
- duplicate constraints;
- invalid type parameters;
- invalid defaults;
- incompatible modifiers;
- invalid FK references cuando exista metadata;
- unsupported operations;
- dependency errors;
- unsafe transformations.

---

# 88. Duplicate column example

```php
Schema::create('users', function ($table) {
    $table->string('email');
    $table->string('email');
});
```

deberá fallar antes de llegar al DBMS.

---

# 89. Type validation

Ejemplo inválido:

```php
$table->decimal(
    'amount',
    precision: 2,
    scale: 5
);
```

deberá detectarse.

---

# 90. Capability System

El Schema System consultará:

```text
Database Capability System
```

para conocer si una operación puede representarse.

---

# 91. Version is evidence

Debe mantenerse:

```text
DBMS Version
→ capability evidence
```

no:

```text
DBMS Version
= capability
```

---

# 92. Capability statuses

Podrán utilizarse:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EXTENSION
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 93. Schema support mode

Además:

```text
NATIVE
EMULATED
DEGRADED
NONE
```

cuando aplique.

---

# 94. Example

Una operación:

```text
DROP COLUMN
```

podría ser:

```text
Platform A:
NATIVE

Platform B:
EMULATED via table rebuild

Platform C:
UNKNOWN
```

---

# 95. Emulation transparency

Si una modificación requiere reconstruir una tabla, el developer tooling deberá mostrarlo.

---

# 96. Validity vs Cost

Debe distinguirse:

```text
Operation can be represented
```

de:

```text
Operation is cheap/safe to execute
```

---

# 97. Schema Planner

El Planner deberá ordenar operaciones respetando dependencias.

Ejemplo:

```text
Create referenced table
↓
Create referencing table
↓
Add foreign key
```

si la plataforma/estrategia lo requiere.

---

# 98. Dependency Graph

Podrá construirse:

```text
SchemaOperationGraph
```

para detectar:

```text
dependencies
cycles
ordering
barriers
```

---

# 99. Planner ≠ Compiler

Debe mantenerse:

```text
Planner
→ decides operation strategy/order

Compiler
→ creates platform representation
```

---

# 100. Schema Compiler

Recibirá operaciones ya:

```text
validated
planned
capability-resolved
```

y generará comandos específicos.

---

# 101. Compiler output

Podrá ser:

```text
CompiledSchemaCommand
```

con:

```text
SQL
bindings if applicable
execution flags
transaction characteristics
expected effects
diagnostic metadata
```

---

# 102. Compiler ≠ executor

El Compiler nunca deberá llamar al Driver para ejecutar el DDL.

---

# 103. Platform Compilers

Existirán compiladores para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 104. MySQL ≠ MariaDB

También para Schema:

```text
MySQL Schema Compiler
≠
MariaDB Schema Compiler
```

aunque compartan componentes.

---

# 105. SQLite

SQLite podrá requerir estrategias especiales como table rebuild para ciertas transformaciones.

Estas estrategias deberán pertenecer al planner/compiler correspondiente, no al Blueprint.

---

# 106. Schema Diff

VoltStack deberá permitir:

```php
$diff = Schema::diff(
    current: $current,
    target: $target,
);
```

o mediante servicio especializado.

---

# 107. Diff direction

Debe ser explícito:

```text
Current
→
Target
```

---

# 108. Difference types

Podrán incluir:

```text
TableAdded
TableRemoved
ColumnAdded
ColumnRemoved
ColumnChanged
IndexAdded
IndexRemoved
ConstraintAdded
ConstraintRemoved
```

etc.

---

# 109. Difference ≠ Migration

Debe mantenerse:

```text
Difference
≠
Migration Operation
```

El Migration Planner decidirá cómo realizar la transición.

---

# 110. Rename ambiguity

Por ejemplo:

```text
old_name removed
new_name added
```

podría significar:

```text
rename
```

o:

```text
drop + add
```

VoltStack no deberá adivinar destructivamente.

---

# 111. Diff Confidence

Una diferencia podrá incluir:

```text
confidence
evidence
coverage
```

---

# 112. Unknown current state

Si introspection fue parcial:

```text
UNKNOWN
```

deberá propagarse al Diff cuando afecte la comparación.

---

# 113. Schema Diff visualization

Developer tooling podrá mostrar:

```text
users
 ├─ + phone:string nullable
 ├─ ~ email:string(255) → string(320)
 └─ - legacy_code:string
```

con indicadores de riesgo.

---

# 114. Schema declaration

VoltStack podrá soportar en el futuro un modelo declarativo completo:

```php
$target = SchemaDefinition::database()
    ->table('users', ...)
    ->table('orders', ...);
```

que pueda compararse contra el estado actual.

---

# 115. Declarative schema

Flujo:

```text
Desired Schema
      +
Observed Schema
      ↓
Schema Diff
      ↓
Migration Plan
```

---

# 116. Migration Integration

Las migrations podrán usar la misma Schema API:

```php
final class AddPhoneToUsers extends Migration
{
    public function up(): void
    {
        Schema::table('users', function ($table) {
            $table->string('phone')->nullable();
        });
    }

    public function down(): void
    {
        Schema::table('users', function ($table) {
            $table->dropColumn('phone');
        });
    }
}
```

---

# 117. Migration owns evolution context

La migration aporta:

```text
identity
version/order
batch
execution history
rollback intent
safety context
```

El Schema Builder no.

---

# 118. Schema Builder ≠ Migration System

Esto deberá permanecer como frontera arquitectónica.

---

# 119. Migration Planner

Antes de ejecutar podrá transformar:

```text
Schema Operations
```

en:

```text
Migration Execution Plan
```

---

# 120. Migration Safety

El plan podrá clasificarse por:

```text
data loss
table rewrite
locking
duration risk
backfill
compatibility
rollback complexity
zero-downtime compatibility
```

---

# 121. Destructive operations

Ejemplos:

```text
DROP TABLE
DROP COLUMN
narrowing type
NOT NULL on existing data
```

deberán ser analizados.

---

# 122. Destructive ≠ forbidden

Una operación destructiva puede ser intencional.

El objetivo es:

```text
make risk explicit
```

no impedir toda operación destructiva.

---

# 123. Safe migration API

Tooling podrá requerir:

```php
Migration::allowDataLoss(...)
```

o acknowledgement/policy equivalente.

La forma final deberá evitar bypasses triviales.

---

# 124. `--force`

Un flag genérico:

```text
--force
```

no deberá desactivar protecciones contra un entorno identificado como producción cuando la política lo prohíba.

---

# 125. Zero Downtime

Schema DX deberá poder indicar si una operación es incompatible con una política zero-downtime.

---

# 126. Zero Downtime ≠ DDL property

Debe mantenerse:

```text
Zero Downtime
=
Application
+
Schema
+
Data
+
Traffic
+
Deployment sequence
```

---

# 127. Expand/Contract

Para cambios complejos podrán recomendarse fases:

```text
EXPAND
↓
MIGRATE DATA
↓
DUAL COMPATIBILITY
↓
SWITCH
↓
CONTRACT
```

---

# 128. Example rename

En lugar de un rename inmediato crítico:

```text
add new column
↓
dual write
↓
backfill
↓
switch reads
↓
remove old column
```

cuando la política operacional lo requiera.

---

# 129. Schema Safety Diagnostics

Ejemplo:

```text
Migration risk detected

Operation:
ALTER users.email VARCHAR(320) → VARCHAR(100)

Risk:
Potential data truncation

Evidence:
Existing maximum observed length: 247

Classification:
DESTRUCTIVE

Recommended action:
Validate and transform existing data before narrowing the column.
```

---

# 130. Evidence ≠ Guarantee

Una consulta previa que observe:

```text
max length = 80
```

no necesariamente garantiza que el dato no cambie antes del DDL.

La safety policy deberá considerar concurrencia.

---

# 131. Transactional DDL

VoltStack no deberá asumir:

```text
DDL
=
transactional
```

universalmente.

---

# 132. Platform DDL semantics

El Capability System deberá describir:

```text
transactional DDL
implicit commit
online DDL
concurrent index creation
lock characteristics
```

cuando sea posible.

---

# 133. Rollback

Migration rollback no deberá asumirse automáticamente posible.

---

# 134. Reversible operation

Una operación podrá clasificarse:

```text
REVERSIBLE
CONDITIONALLY_REVERSIBLE
IRREVERSIBLE
UNKNOWN
```

---

# 135. Drop data

Después de:

```text
DROP COLUMN
```

recrear la columna no restaura sus datos.

Por tanto:

```text
structural reversal
≠
data restoration
```

---

# 136. Schema inspection tooling

CLI:

```text
php voltstack database:schema
```

podrá mostrar:

```text
Database
Tables
Columns
Indexes
Constraints
Foreign Keys
Capabilities
Coverage
```

---

# 137. Inspect table

Ejemplo:

```text
php voltstack database:schema users
```

---

# 138. Schema diff CLI

Podrá existir:

```text
php voltstack database:schema:diff
```

con salida legible.

---

# 139. Dry Run

Migration/schema tooling deberá soportar:

```text
--dry-run
```

cuando sea viable.

---

# 140. Dry Run semantics

Debe mantenerse:

```text
Dry Run
≠
Guaranteed execution outcome
```

Un dry run demuestra el plan generado, no el futuro estado del servidor.

---

# 141. SQL preview

Podrá mostrarse:

```text
planned operations
compiled commands
capability decisions
risk classifications
```

antes de ejecución.

---

# 142. Raw Schema SQL

Deberá existir una escape hatch limitada para casos no modelados.

Por ejemplo:

```php
Schema::raw(...);
```

pero será considerada:

```text
platform-specific
reduced analyzability
potentially unsafe
```

---

# 143. Raw schema rule

Debe mantenerse:

```text
Raw DDL
≠
Normal Schema API
```

---

# 144. Raw DDL and safety

El Safety System no podrá inferir completamente los efectos de SQL arbitrario.

Por ello podrá clasificarlo:

```text
UNKNOWN RISK
```

por defecto.

---

# 145. Custom Schema Extensions

Plugins podrán añadir:

```text
column types
constraints
indexes
schema nodes
compiler handlers
capability providers
introspection handlers
```

mediante APIs formales.

---

# 146. Extension ≠ runtime monkey patch

Los registries deberán configurarse durante bootstrap y congelarse.

---

# 147. Custom logical types

Ejemplo:

```php
$table->vector(
    'embedding',
    dimensions: 1536
);
```

podrá provenir de una extensión.

---

# 148. Extension capability

La extensión deberá declarar requisitos como:

```text
VECTOR_TYPE
VECTOR_INDEX
```

y no asumir vendor sólo por nombre.

---

# 149. Schema API and ORM

El ORM podrá consumir Schema Metadata.

Pero:

```text
ORM Mapping
≠
Database Schema
```

---

# 150. Mapping does not create schema automatically

Definir:

```php
#[Column(type: 'string')]
public string $name;
```

no deberá ejecutar automáticamente DDL.

---

# 151. ORM Schema Generation

Tooling podrá convertir metadata ORM en un:

```text
Target Schema Model
```

pero deberá ser una operación explícita.

---

# 152. ORM Metadata → Schema

Flujo posible:

```text
Entity Metadata
      ↓
Schema Projection
      ↓
Target Schema Model
      ↓
Schema Diff
      ↓
Migration Proposal
```

---

# 153. Proposal ≠ execution

Generar una migration desde ORM metadata no deberá ejecutarla automáticamente.

---

# 154. Model API and migrations

Los Models no deberán contener responsabilidades de Schema mutation.

---

# 155. Schema caching

Metadata introspectada podrá cachearse.

Pero:

```text
Cached Schema Metadata
≠
Database Truth
```

---

# 156. Cache validity

El cache deberá asociarse a:

```text
database
schema
platform
metadata generation
migration generation
```

cuando corresponda.

---

# 157. Migration invalidation

Después de un cambio estructural confirmado deberán invalidarse snapshots/cache relacionados.

---

# 158. UNKNOWN migration outcome

Si el resultado DDL es incierto:

```text
schema cache
```

deberá invalidarse conservadoramente.

---

# 159. Persistent Runtime

Bajo FrankenPHP:

```text
Schema mutable operation state
```

no deberá permanecer entre requests.

---

# 160. Shareable state

Podrán compartirse:

```text
immutable schema definitions
compiled metadata
frozen registries
platform definitions
```

cuando sea seguro.

---

# 161. Non-shareable state

No deberán compartirse globalmente:

```text
current migration
current schema operation
current connection
current tenant
temporary diff
execution plan state
```

---

# 162. Multitenancy

El paquete opcional Multitenancy podrá definir:

```text
database-per-tenant
schema-per-tenant
shared-schema
```

---

# 163. Core independence

Debe mantenerse:

```text
Database Schema Core
```

sin dependencia obligatoria de:

```text
VoltStack Multitenancy
```

---

# 164. Tenant-aware schema

Cuando Multitenancy esté instalado, podrá aportar:

```text
TenantSchemaContext
TenantMigrationCoordinator
TenantSchemaResolver
```

---

# 165. Tenant safety

Una operación schema no deberá aplicarse accidentalmente a:

```text
all tenants
```

si el contexto sólo autorizaba uno.

---

# 166. Tenant migrations

Operaciones masivas sobre muchos tenants requerirán orchestration específica.

---

# 167. Schema and Sharding

De forma similar, un esquema distribuido podrá tener:

```text
ShardSchemaGeneration
```

o estado por shard.

---

# 168. Schema drift

VoltStack podrá detectar:

```text
expected schema
≠
observed schema
```

entre:

```text
replicas
shards
tenants
environments
```

cuando sea relevante.

---

# 169. Drift ≠ migration automatically

Detectar drift no deberá aplicar cambios automáticamente.

---

# 170. Schema health

Health tooling podrá indicar:

```text
EXPECTED
DRIFTED
UNKNOWN
INCOMPLETE_EVIDENCE
```

---

# 171. Testing DX

Schema deberá integrarse con:

```text
290_DATABASE_SCHEMA_TESTING_SYSTEM.md
```

---

# 172. Semantic assertions

Ejemplo:

```php
assertSchema('users')
    ->hasColumn('email')
    ->column('email')
        ->isString()
        ->isNotNullable();
```

---

# 173. Structural assertion ≠ SQL assertion

Debe favorecerse:

```text
assert semantic schema
```

sobre:

```text
assert CREATE TABLE string
```

para tests portables.

---

# 174. Platform compiler tests

El SQL exacto sí deberá probarse en tests específicos del compiler.

---

# 175. Real DB testing

La afirmación:

```text
PostgreSQL creates this constraint correctly
```

requiere integración real.

---

# 176. Roundtrip testing

Patrón:

```text
Schema Definition
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

# 177. Shared bug risk

Ese roundtrip puede ocultar un bug si compiler e introspector comparten la misma interpretación incorrecta.

---

# 178. Independent schema fixtures

También deberán probarse schemas creados externamente mediante SQL nativo conocido.

---

# 179. SQLite

SQLite no deberá utilizarse como prueba universal de comportamiento de:

```text
MySQL
MariaDB
PostgreSQL
```

---

# 180. Schema Performance

Deberán poder medirse por separado:

```text
definition construction
normalization
diff
planning
compilation
introspection
DDL execution
```

---

# 181. DDL execution benchmark

No deberá mezclarse automáticamente con:

```text
Schema Builder performance
```

---

# 182. Large schema

VoltStack deberá probar schemas con:

```text
hundreds/thousands of tables
columns
indexes
constraints
```

para evitar algoritmos accidentalmente cuadráticos donde no sean necesarios.

---

# 183. Introspection performance

La introspection deberá minimizar round trips innecesarios.

---

# 184. Lazy inspection

Podrá soportarse introspection selectiva:

```text
table only
columns only
indexes only
constraints only
```

cuando sea apropiado.

---

# 185. Partial inspection

Pero deberá conservar coverage explícito.

---

# 186. Diagnostics

Schema DX deberá explicar:

```text
what
where
why
risk
capability
possible action
```

---

# 187. Unknown type error

Ejemplo:

```text
Unknown schema type `moneyz`.

Column:
orders.total

Available logical types:
- decimal
- integer
- string
...

Did you mean:
money
```

sólo si existe un tipo registrado apropiado.

---

# 188. Unsupported capability error

Ejemplo:

```text
Cannot create index `idx_articles_search`.

Requested feature:
FULL_TEXT_INDEX

Platform:
SQLite

Capability:
UNSUPPORTED

No safe emulation strategy is registered.
```

---

# 189. Partial capability

Ejemplo:

```text
Feature supported with limitations.

Feature:
GENERATED_COLUMN

Requested mode:
STORED

Effective platform support:
VIRTUAL_ONLY
```

---

# 190. Unsafe migration diagnostic

Ejemplo:

```text
Potentially destructive schema operation.

Table:
users

Operation:
DROP COLUMN legacy_code

Data impact:
All values stored in users.legacy_code will be lost.

Safety:
DESTRUCTIVE
```

---

# 191. Lock risk

Podrá informarse:

```text
Operation may require an exclusive table lock.
```

sólo cuando exista evidencia/capability suficiente.

---

# 192. No fabricated guarantees

Si no puede conocerse:

```text
Lock impact: UNKNOWN
```

será preferible a una afirmación falsa.

---

# 193. Schema explain

Podrá existir:

```php
Schema::plan(...)->explain();
```

---

# 194. Example explain

```text
Schema Plan

Target:
users

Operation:
AddColumn(phone)

Validation:
PASS

Capability:
ADD_COLUMN = SUPPORTED

Execution strategy:
NATIVE

Risk:
LOW

Compiler:
PostgreSQLSchemaCompiler

Commands:
1
```

---

# 195. Migration explain

Para cambios complejos:

```text
Migration Plan
├── Add nullable column
├── Backfill required
├── Validate values
├── Add constraint
└── Remove compatibility state
```

---

# 196. Schema command naming

Las operaciones deberán tener nombres estables para:

```text
logs
telemetry
audit
testing
diagnostics
```

---

# 197. Audit

Operaciones administrativas/destructivas podrán producir audit events.

---

# 198. Audit ≠ telemetry

Debe mantenerse:

```text
Audit
≠
Telemetry
```

---

# 199. Telemetry

Podrá capturarse:

```text
operation type
duration
platform
database
schema
result
risk class
command count
```

sin exponer secretos.

---

# 200. DDL content

SQL completo podrá ser sensible.

Su captura deberá obedecer política.

---

# 201. Developer source location

En modo desarrollo podrá registrarse:

```text
migration
file
line
operation
```

para facilitar diagnóstico.

---

# 202. IDE Experience

Las APIs deberán priorizar:

```text
typed methods
enums
value objects
generic-friendly collections
documented return types
```

---

# 203. Stringly typed API reduction

Preferible:

```php
$table->foreign('user_id')
    ->onDelete(ForeignKeyAction::CASCADE);
```

a depender exclusivamente de:

```php
->onDelete('cascade');
```

aunque pueda existir convenience.

---

# 204. Type discoverability

El IDE deberá poder descubrir:

```text
column methods
modifiers
constraint methods
index methods
```

sin documentación externa para operaciones comunes.

---

# 205. Static analysis

La API deberá diseñarse para herramientas como:

```text
PHPStan
Psalm
IDE language servers
```

sin hacerlas dependencias obligatorias del runtime.

---

# 206. Named arguments

APIs complejas podrán beneficiarse de PHP named arguments:

```php
$table->decimal(
    name: 'amount',
    precision: 12,
    scale: 2,
);
```

---

# 207. Error locality

Errores de definición deberán aparecer lo más cerca posible de la construcción original.

---

# 208. No silent normalization of dangerous mistakes

Una definición claramente contradictoria no deberá "arreglarse" silenciosamente.

---

# 209. Naming conventions

Cuando no se especifique nombre para:

```text
indexes
constraints
foreign keys
```

VoltStack podrá generar nombres deterministas.

---

# 210. Deterministic naming

La misma definición bajo la misma política deberá producir el mismo nombre lógico.

---

# 211. Name length limits

El compiler/planner deberá considerar límites de plataforma.

---

# 212. Hash suffix

Podrá utilizarse una estrategia determinista para evitar colisiones/truncamiento.

---

# 213. Generated name ≠ semantic identity

La identidad lógica no deberá depender exclusivamente del string físico generado.

---

# 214. Reserved words

Identifiers que colisionen con palabras reservadas deberán manejarse mediante el sistema de identifiers/compiler.

---

# 215. Quoting

El usuario no deberá escribir manualmente:

```text
`users`
"users"
[users]
```

en la API normal.

---

# 216. Quoting belongs to compiler

Debe mantenerse:

```text
Identifier quoting
→ platform compiler
```

---

# 217. Database namespaces

PostgreSQL-style schemas u otros namespaces deberán representarse estructuralmente.

---

# 218. Qualified identifiers

Ejemplo conceptual:

```php
Schema::table(
    QualifiedTable::of(
        schema: 'billing',
        table: 'invoices'
    ),
    ...
);
```

con conveniences más simples cuando sea apropiado.

---

# 219. String parsing

Podrá aceptarse:

```text
billing.invoices
```

como convenience si el parser es seguro y no ambiguo.

---

# 220. Database selection

El desarrollador podrá indicar conexión/base lógica:

```php
Schema::connection('analytics')
    ->create(...);
```

---

# 221. Logical connection

Debe significar:

```text
logical database/connection identity
```

no necesariamente un socket físico inmediato.

---

# 222. Connection acquisition

La construcción de Schema AST no deberá adquirir conexión salvo que requiera:

```text
introspection
capability probe
execution
```

---

# 223. Late resource acquisition

La ejecución deberá adquirir recursos tan tarde como sea razonable.

---

# 224. Schema Context

Podrá existir:

```text
SchemaContext
├── logical database
├── namespace
├── tenant
├── migration
├── platform
├── capabilities
├── safety policy
├── execution policy
└── telemetry
```

---

# 225. Context ≠ Schema AST

El AST expresa la operación.

El contexto expresa dónde/bajo qué políticas será evaluada.

---

# 226. Schema Plan immutability

Una vez validado/finalizado, el plan deberá ser immutable.

---

# 227. Plan identity

Podrá tener:

```text
SchemaPlanId
SchemaPlanFingerprint
```

para diagnóstico/reproducibilidad.

---

# 228. Schema fingerprint

Un Schema Model normalizado podrá producir un fingerprint.

---

# 229. Fingerprint uses

Podrá utilizarse para:

```text
drift detection
cache validation
testing
migration evidence
diagnostics
```

---

# 230. Fingerprint ≠ complete backup

Obviamente:

```text
Schema Fingerprint
≠
Schema Backup
```

---

# 231. Views

La arquitectura deberá poder evolucionar hacia:

```php
Schema::createView(...);
```

aunque no necesariamente forme parte del V1 mínimo.

---

# 232. Materialized Views

Deberán modelarse separadamente cuando sean soportadas.

---

# 233. Sequences

Plataformas con sequences deberán poder representarlas mediante capabilities.

---

# 234. Identity columns

Identity/autoincrement deberá expresarse semánticamente y compilarse por plataforma.

---

# 235. Enum schema types

Enums nativos de plataforma no deberán confundirse con el Enum Mapping del ORM.

---

# 236. Domain types

PostgreSQL domains u otros tipos avanzados deberán pertenecer a extensiones/capabilities cuando corresponda.

---

# 237. Triggers

Triggers son objetos schema avanzados.

Si se soportan, deberán tener un modelo explícito.

No simplemente raw SQL obligatorio como arquitectura permanente.

---

# 238. Stored procedures

Igualmente deberán tratarse como extensión avanzada y no contaminar el V1 básico.

---

# 239. Database portability

La API deberá diferenciar:

```text
portable core
```

de:

```text
platform-specific extension
```

---

# 240. Portability report

Tooling podrá indicar:

```text
Portable: 94%

Platform-specific:
- PostgreSQL partial index
- PostgreSQL generated expression
```

si puede calcularse con evidencia suficiente.

---

# 241. Portability ≠ lowest common denominator

VoltStack no deberá eliminar funciones avanzadas sólo para mantener una API idéntica.

La estrategia será:

```text
portable semantic core
+
explicit capabilities/extensions
```

---

# 242. Environment awareness

Schema operations deberán conocer el entorno efectivo cuando se ejecuten.

---

# 243. Production safeguards

Operaciones destructivas en producción podrán requerir políticas más estrictas.

---

# 244. Environment name alone

Debe mantenerse:

```text
APP_ENV=production
```

como evidencia útil, pero no como única prueba de identidad de una base crítica cuando se requiera mayor seguridad.

---

# 245. Database identity

Tooling administrativo podrá verificar:

```text
server
database
cluster
environment fingerprint
```

antes de acciones destructivas.

---

# 246. Schema Backup Coordination

Para cambios críticos, policies podrán exigir:

```text
verified backup
```

antes de ejecutar.

---

# 247. Backup Created ≠ Verified

Debe mantenerse:

```text
Backup Created
≠
Backup Verified
≠
Backup Restorable
```

---

# 248. Schema API simplicity

A pesar de toda esta infraestructura, el caso común deberá permanecer:

```php
Schema::create('users', function ($table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamps();
});
```

---

# 249. Complexity hiding

La complejidad deberá residir en:

```text
Schema Engine
```

no en cada aplicación.

---

# 250. Progressive Disclosure

La experiencia será:

```text
Level 1
Schema::create()

Level 2
Typed Blueprint

Level 3
Schema Definitions

Level 4
Schema AST

Level 5
Schema Diff / Planner

Level 6
Compiler / Platform Extensions
```

---

# 251. Basic developer

No deberá necesitar entender:

```text
Schema AST
Capability Resolver
Schema Planner
```

para crear una tabla sencilla.

---

# 252. Advanced developer

Sí deberá poder acceder a esas capas para:

```text
framework packages
database tooling
migration generators
platform plugins
administration
testing
```

---

# 253. Proposed namespace

```text
src/Quantum/Database/
└── Schema/
    ├── Builder/
    │   ├── SchemaBuilder.php
    │   ├── TableBlueprint.php
    │   ├── ColumnBlueprint.php
    │   └── ConstraintBlueprint.php
    │
    ├── Definition/
    │   ├── DatabaseDefinition.php
    │   ├── TableDefinition.php
    │   ├── ColumnDefinition.php
    │   ├── IndexDefinition.php
    │   └── ConstraintDefinition.php
    │
    ├── AST/
    │   ├── SchemaNode.php
    │   ├── CreateTableNode.php
    │   ├── AlterTableNode.php
    │   ├── AddColumnNode.php
    │   └── DropColumnNode.php
    │
    ├── Model/
    ├── Introspection/
    ├── Normalization/
    ├── Validation/
    ├── Diff/
    ├── Planning/
    ├── Compilation/
    ├── Capability/
    ├── Safety/
    ├── Diagnostics/
    └── Extension/
```

---

# 254. Public API namespace

Las interfaces públicas comunes deberán ser pocas y estables.

Ejemplo:

```text
Schema
SchemaManager
TableBlueprint
ColumnDefinition
SchemaModel
SchemaDiff
SchemaPlan
```

---

# 255. Internal API

Tipos internos como:

```text
platform compiler nodes
introspection adapters
planner internals
```

no deberán convertirse accidentalmente en API pública estable.

---

# 256. Backward compatibility

Cambios a métodos públicos como:

```php
$table->string(...)
```

deberán respetar la política general de compatibilidad de VoltStack.

---

# 257. Extension compatibility

Custom schema extensions deberán declarar versiones/contracts compatibles.

---

# 258. Error hierarchy

Podrá incluir:

```text
SchemaException
├── SchemaDefinitionException
├── SchemaValidationException
├── SchemaIntrospectionException
├── SchemaDiffException
├── SchemaPlanningException
├── SchemaCompilationException
├── SchemaExecutionException
├── SchemaCapabilityException
└── UnsafeSchemaOperationException
```

---

# 259. Error phase

Cada error deberá indicar su fase cuando sea relevante:

```text
DEFINE
INTROSPECT
NORMALIZE
DIFF
VALIDATE
PLAN
COMPILE
EXECUTE
VERIFY
```

---

# 260. Error context

Deberá incluir de forma segura:

```text
database
schema
table
column
operation
platform
capability
migration
```

según aplique.

---

# 261. Exception ≠ diagnostic

Una exception podrá contener un identificador hacia un objeto diagnóstico más rico.

---

# 262. Machine-readable diagnostics

Tooling deberá poder obtener:

```text
code
severity
phase
component
operation
evidence
suggestions
```

sin parsear strings humanos.

---

# 263. Human-readable diagnostics

Al mismo tiempo, CLI y developer tools deberán mostrar mensajes útiles.

---

# 264. Schema Developer Experience Formula

Conceptualmente:

```text
SchemaDX
=
Ergonomics
+
StructuralCorrectness
+
Portability
+
CapabilityAwareness
+
Safety
+
Explainability
+
IDE Discoverability
```

---

# 265. Valid Schema Operation

```text
ValidSchemaOperation
=
StructurallyValid
∧
SemanticallyValid
∧
CapabilityCompatible
```

---

# 266. Executable Schema Operation

```text
Executable
=
ValidSchemaOperation
∧
ResolvedContext
∧
ResolvedPlatform
∧
ResolvedCapabilities
∧
SafetyPolicySatisfied
```

---

# 267. Safe Operation

```text
Safe
≠
Executable
```

Una operación puede ser ejecutable pero insegura bajo una política concreta.

---

# 268. Diff validity

```text
ReliableDiff
=
ComparableModels
∧
SufficientCoverage
∧
KnownDirection
```

---

# 269. Introspection confidence

```text
ObservedAbsence
```

sólo será afirmable cuando:

```text
Coverage = COMPLETE
```

para la dimensión correspondiente.

---

# 270. Schema execution outcome

Después de DDL:

```text
SUCCESS
FAILED
UNKNOWN
```

deberá preservarse según evidencia.

---

# 271. Unknown outcome

Si la conexión se pierde durante DDL y no puede determinarse el resultado:

```text
UNKNOWN
```

no deberá convertirse automáticamente en:

```text
FAILED
```

---

# 272. Reconciliation

Después de outcome incierto podrá requerirse:

```text
reconnect
↓
introspect
↓
compare expected vs observed
↓
reconcile
```

---

# 273. Retry

Schema DDL no deberá reintentarse ciegamente.

---

# 274. Idempotency

Una operación con:

```text
IF NOT EXISTS
```

no es automáticamente completamente idempotente en toda semántica.

---

# 275. Migration repository

El estado de migration deberá actualizarse sólo cuando exista evidencia suficiente del resultado.

---

# 276. Anti-pattern: direct SQL construction

```php
$sql = "CREATE TABLE {$name} (...)";
```

como implementación del Blueprint.

Prohibido.

---

# 277. Anti-pattern: Builder executing DDL

```text
TableBlueprint
→ PDO::exec()
```

Prohibido.

---

# 278. Anti-pattern: compiler executing

Prohibido.

---

# 279. Anti-pattern: ORM as schema truth

```text
Entity metadata
=
database schema
```

Prohibido.

---

# 280. Anti-pattern: index equals constraint

Prohibido.

---

# 281. Anti-pattern: version checks everywhere

```php
if ($mysqlVersion >= ...)
```

disperso en Schema Builder.

Evitar.

---

# 282. Anti-pattern: vendor conditionals in migrations

Evitar cuando una semantic API/capability pueda representar la intención.

---

# 283. Anti-pattern: assuming MySQL/MariaDB equivalence

Prohibido.

---

# 284. Anti-pattern: SQLite as universal schema test

Prohibido.

---

# 285. Anti-pattern: unknown = absent

Prohibido.

---

# 286. Anti-pattern: rename guessing

No inferir renames destructivos sin evidencia suficiente.

---

# 287. Anti-pattern: drop + recreate hidden as alter

Una emulación destructiva deberá ser visible.

---

# 288. Anti-pattern: migration success after uncertain DDL

Prohibido.

---

# 289. Anti-pattern: global schema state

Prohibido bajo persistent runtimes.

---

# 290. Anti-pattern: tenant leakage

Prohibido.

---

# 291. Anti-pattern: raw DDL as normal API

Evitar.

---

# 292. Anti-pattern: unsafe production bypass

Prohibido.

---

# 293. Anti-pattern: generated SQL as architecture

El Schema subsystem deberá modelar intención antes de SQL.

---

# 294. Anti-pattern: hidden introspection

No realizar introspection costosa durante cada llamada Builder sin necesidad.

---

# 295. Anti-pattern: silently changing requested semantics

Si una plataforma no puede representar una definición:

```text
fail / explain / explicit emulation
```

en lugar de cambiar silenciosamente el significado.

---

# 296. Invariantes — Schema API

## DB-SDX-001

Schema Builder ≠ SQL Builder.

## DB-SDX-002

Schema Builder ≠ Executor.

## DB-SDX-003

Schema Builder ≠ Migration Runner.

## DB-SDX-004

Schema Builder ≠ Introspector.

## DB-SDX-005

Schema Builder ≠ Driver.

## DB-SDX-006

Schema Model ≠ Schema AST.

## DB-SDX-007

Schema Definition ≠ Schema Model.

## DB-SDX-008

Schema AST ≠ Migration.

## DB-SDX-009

Schema Diff ≠ Migration.

## DB-SDX-010

Schema Compiler ≠ Executor.

---

# 297. Invariantes — Types

## DB-SDX-011

Logical Type ≠ Physical Type.

## DB-SDX-012

Physical type mapping pertenecerá a plataforma/compiler.

## DB-SDX-013

Unknown type no será convertido silenciosamente a string.

## DB-SDX-014

Precision y scale serán validados.

## DB-SDX-015

Nullability será explícita.

## DB-SDX-016

Literal Default ≠ Default Expression.

## DB-SDX-017

Generated Expression ≠ Literal Default.

## DB-SDX-018

Identity strategy será semántica.

## DB-SDX-019

Vendor type será extensión cuando no pertenezca al core portable.

## DB-SDX-020

Type conversion no pertenecerá al Schema Builder.

---

# 298. Invariantes — Constraints

## DB-SDX-021

Constraint ≠ Index.

## DB-SDX-022

Primary Key ≠ Index.

## DB-SDX-023

Unique Constraint ≠ Unique Index.

## DB-SDX-024

Foreign Key será una constraint explícita.

## DB-SDX-025

Composite key será first-class.

## DB-SDX-026

Referential actions serán tipadas.

## DB-SDX-027

Check constraint será semántica cuando sea posible.

## DB-SDX-028

Constraint names podrán generarse determinísticamente.

## DB-SDX-029

Physical implementation no redefinirá identidad lógica.

## DB-SDX-030

Constraint support será capability-aware.

---

# 299. Invariantes — Introspection

## DB-SDX-031

Introspection observa; no define.

## DB-SDX-032

Not Observed ≠ Absent.

## DB-SDX-033

Coverage será explícita.

## DB-SDX-034

PARTIAL ≠ COMPLETE.

## DB-SDX-035

UNKNOWN ≠ NO.

## DB-SDX-036

Introspection errors no se convertirán silenciosamente en schema vacío.

## DB-SDX-037

Introspection result será normalizado.

## DB-SDX-038

Vendor metadata no contaminará innecesariamente el modelo portable.

## DB-SDX-039

Selective introspection preservará coverage.

## DB-SDX-040

Cached introspection ≠ database truth.

---

# 300. Invariantes — Diff

## DB-SDX-041

Schema Diff será direccional.

## DB-SDX-042

Current → Target será explícito.

## DB-SDX-043

Difference ≠ Migration.

## DB-SDX-044

Rename no será inferido destructivamente sin evidencia.

## DB-SDX-045

Unknown source state propagará incertidumbre.

## DB-SDX-046

Diff no generará SQL.

## DB-SDX-047

Diff será determinista bajo inputs equivalentes.

## DB-SDX-048

Diff preservará semantic identity.

## DB-SDX-049

Diff podrá producir evidencia diagnóstica.

## DB-SDX-050

Diff no ejecutará operaciones.

---

# 301. Invariantes — Capabilities

## DB-SDX-051

Version ≠ Capability.

## DB-SDX-052

UNKNOWN ≠ UNSUPPORTED.

## DB-SDX-053

Capability evidence ≠ capability decision.

## DB-SDX-054

Builder no hará vendor branching innecesario.

## DB-SDX-055

Platform resolverá representación física.

## DB-SDX-056

Emulation será explícita.

## DB-SDX-057

Degraded semantics serán explícitas.

## DB-SDX-058

Unsupported semantics no se compilarán silenciosamente.

## DB-SDX-059

MySQL ≠ MariaDB.

## DB-SDX-060

Capability snapshot tendrá scope apropiado.

---

# 302. Invariantes — Planning/Compilation

## DB-SDX-061

Planner ≠ Compiler.

## DB-SDX-062

Planner resolverá dependencias.

## DB-SDX-063

Compiler generará representación física.

## DB-SDX-064

Compiler no ejecutará.

## DB-SDX-065

Compiler no decidirá seguridad operacional global.

## DB-SDX-066

Compiled Command ≠ Executed Command.

## DB-SDX-067

Operation ordering será determinista cuando sea posible.

## DB-SDX-068

Table rebuild será visible.

## DB-SDX-069

DDL transactional semantics serán capability-aware.

## DB-SDX-070

Unknown DDL behavior no se tratará como seguro.

---

# 303. Invariantes — Safety

## DB-SDX-071

Valid ≠ Safe.

## DB-SDX-072

Executable ≠ Safe.

## DB-SDX-073

UNKNOWN safety ≠ SAFE.

## DB-SDX-074

Destructive operation será identificable.

## DB-SDX-075

Data loss risk será explícito.

## DB-SDX-076

Lock risk será explícito cuando pueda conocerse.

## DB-SDX-077

Rollback structural ≠ data restoration.

## DB-SDX-078

`IF EXISTS` ≠ safe.

## DB-SDX-079

`--force` no invalidará hard safety invariants.

## DB-SDX-080

Raw DDL tendrá safety confidence reducida.

---

# 304. Invariantes — Migrations

## DB-SDX-081

Migration ≠ Schema Builder.

## DB-SDX-082

Migration Repository ≠ Schema Model.

## DB-SDX-083

Migration Plan ≠ Schema Diff.

## DB-SDX-084

Migration batch será explícito.

## DB-SDX-085

Migration outcome UNKNOWN será preservado.

## DB-SDX-086

DDL no será retried ciegamente.

## DB-SDX-087

Zero downtime no será asumido por DDL válido.

## DB-SDX-088

Expand/contract podrá modelarse.

## DB-SDX-089

Rollback podrá ser irreversible.

## DB-SDX-090

Migration history sólo se actualizará con evidencia suficiente.

---

# 305. Invariantes — Runtime

## DB-SDX-091

Schema mutable state será scope-local.

## DB-SDX-092

Current migration no será static global.

## DB-SDX-093

Current tenant no será static global.

## DB-SDX-094

Current schema plan no será static global.

## DB-SDX-095

Immutable registries podrán compartirse.

## DB-SDX-096

FrankenPHP deberá aislar requests.

## DB-SDX-097

RoadRunner podrá reutilizar la misma arquitectura.

## DB-SDX-098

OpenSwoole requerirá coroutine isolation.

## DB-SDX-099

Reset failure no será ignorado.

## DB-SDX-100

Cross-request schema state leakage será error crítico.

---

# 306. Invariantes — Developer Experience

## DB-SDX-101

Common schema operations serán simples.

## DB-SDX-102

Advanced semantics estarán disponibles sin raw SQL cuando estén modeladas.

## DB-SDX-103

Raw DDL será escape hatch.

## DB-SDX-104

IDE discoverability será prioritaria.

## DB-SDX-105

Enums/value objects serán preferidos para opciones cerradas.

## DB-SDX-106

Error messages identificarán la operación.

## DB-SDX-107

Diagnostics distinguirán fact/evidence/unknown.

## DB-SDX-108

Schema plans podrán inspeccionarse.

## DB-SDX-109

Compiled SQL podrá previsualizarse.

## DB-SDX-110

Preview ≠ execution guarantee.

---

# 307. Invariantes — Testing

## DB-SDX-111

Schema semantics podrán probarse sin DB cuando corresponda.

## DB-SDX-112

Compiler behavior se probará por plataforma.

## DB-SDX-113

Real DDL behavior requerirá integración real.

## DB-SDX-114

SQLite no probará MySQL/PostgreSQL.

## DB-SDX-115

Roundtrip testing no será la única evidencia.

## DB-SDX-116

Externally-created schemas deberán probar introspection.

## DB-SDX-117

Schema assertions serán semánticas.

## DB-SDX-118

Safety policies tendrán tests.

## DB-SDX-119

Unknown coverage tendrá tests.

## DB-SDX-120

Persistent runtime leakage tendrá tests.

---

# 308. Invariantes — Extensibility

## DB-SDX-121

Schema extensions serán explícitamente registradas.

## DB-SDX-122

Registries serán frozen después de bootstrap.

## DB-SDX-123

Custom type podrá declarar capabilities.

## DB-SDX-124

Custom compiler handler será platform-aware.

## DB-SDX-125

Custom introspection tendrá coverage explícita.

## DB-SDX-126

Unknown extension node no será ignorado.

## DB-SDX-127

Extension no podrá alterar arbitrariamente core semantics.

## DB-SDX-128

Extension compatibility será versionada.

## DB-SDX-129

Plugin failure será diagnosticable.

## DB-SDX-130

Extension state mutable no será global.

---

# 309. Invariantes — Portability

## DB-SDX-131

Portable Core ≠ Lowest Common Denominator.

## DB-SDX-132

Platform-specific feature será explícita.

## DB-SDX-133

Vendor-specific option no redefinirá core semantics.

## DB-SDX-134

Identifier quoting pertenecerá al compiler.

## DB-SDX-135

Reserved words serán manejados por plataforma.

## DB-SDX-136

Physical type names no serán core logical types.

## DB-SDX-137

Platform limitations serán diagnosticables.

## DB-SDX-138

Emulation no será silenciosa.

## DB-SDX-139

Degradation no será silenciosa.

## DB-SDX-140

Portability podrá analizarse antes de ejecución cuando exista evidencia.

---

# 310. Invariantes arquitectónicas finales

## DB-SDX-141

Developer API → Builder.

## DB-SDX-142

Builder → Definition/AST.

## DB-SDX-143

Definition/AST → Validation.

## DB-SDX-144

Validation → Capability Analysis.

## DB-SDX-145

Capability Analysis → Planner.

## DB-SDX-146

Planner → Compiler.

## DB-SDX-147

Compiler → Execution Engine.

## DB-SDX-148

Execution Engine → Connection.

## DB-SDX-149

Connection → Driver.

## DB-SDX-150

Las dependencias no se invertirán arbitrariamente.

---

# 311. V1 Recommended API

El núcleo V1 debería cubrir como mínimo:

```php
Schema::create(...)
Schema::table(...)
Schema::drop(...)
Schema::dropIfExists(...)
Schema::rename(...)
Schema::hasTable(...)
Schema::hasColumn(...)
Schema::inspect(...)
```

---

# 312. V1 Table API

Como mínimo:

```text
id
uuid
integer
bigInteger
decimal
boolean
string
text
binary
date
time
dateTime
timestamp
json
```

con el conjunto final alineado al Type System.

---

# 313. V1 Modifiers

Como mínimo:

```text
nullable
default
primary
unique
index
comment
generated identity/autoincrement
```

donde exista semántica portable suficiente.

---

# 314. V1 Constraints

Como mínimo:

```text
Primary Key
Unique
Foreign Key
Check
```

sujeto a capabilities.

---

# 315. V1 Structural Operations

Como mínimo:

```text
CreateTable
DropTable
RenameTable
AddColumn
DropColumn
RenameColumn
AlterColumn
AddIndex
DropIndex
AddConstraint
DropConstraint
```

---

# 316. V1 Platforms

Deberán considerarse first-class:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 317. V1 inspection

Deberá poder observar al menos:

```text
tables
columns
primary keys
indexes
foreign keys
constraints
```

según coverage/capabilities de cada plataforma.

---

# 318. V1 Safety

Deberá identificar como mínimo:

```text
DROP TABLE
DROP COLUMN
type narrowing
NOT NULL transitions
table rebuild
unknown raw DDL
```

como operaciones que requieren análisis adicional.

---

# 319. V1 migration integration

Schema deberá integrarse completamente con:

```text
Migration Discovery
Migration Repository
Migration Planner
Migration Execution
Migration Rollback
Migration Batch
Migration Safety
```

---

# 320. Acceptance Criteria

`DATABASE_SCHEMA_DEVELOPER_EXPERIENCE` estará preparado para V1 cuando pueda demostrarse que:

1. `Schema::create()` construye estructura semántica;
2. Schema Builder no concatena SQL;
3. Schema Builder no ejecuta DDL;
4. TableBlueprint está tipado;
5. ColumnDefinition está tipada;
6. logical types están separados de physical types;
7. constraints están separadas de indexes;
8. PK está separada conceptualmente de index;
9. UniqueConstraint está separada de UniqueIndex;
10. foreign keys son first-class;
11. composite keys son representables;
12. defaults literales y expressions están separados;
13. generated columns son capability-aware;
14. alter operations son explícitas;
15. destructive operations son detectables;
16. UNKNOWN safety no se considera SAFE;
17. introspection produce Schema Model;
18. introspection conserva coverage;
19. NotObserved no se considera Absent;
20. Schema Diff es direccional;
21. Schema Diff no se considera Migration;
22. rename inference es conservadora;
23. Planner resuelve dependencias;
24. Compiler es platform-specific;
25. Compiler no ejecuta DDL;
26. MySQL y MariaDB tienen tratamiento independiente;
27. SQLite puede usar estrategias específicas explícitas;
28. capabilities no dependen únicamente de version;
29. UNKNOWN capability no equivale a UNSUPPORTED;
30. emulation es visible;
31. table rebuild es visible;
32. DDL transactionality es capability-aware;
33. migrations utilizan la misma Schema API;
34. Schema Builder no administra migration history;
35. migration outcome incierto se preserva;
36. rollback no promete restaurar datos eliminados;
37. zero-downtime se analiza fuera del simple SQL;
38. schema plans pueden inspeccionarse;
39. dry-run no se presenta como garantía;
40. raw DDL es una escape hatch explícita;
41. ORM metadata no se considera schema truth;
42. ORM schema generation produce una propuesta/modelo;
43. persistent runtime state está aislado;
44. FrankenPHP está soportado desde V1;
45. Multitenancy continúa siendo opcional;
46. tenant schema context no se filtra;
47. semantic schema assertions están disponibles;
48. real DB behavior se prueba con integración;
49. extensions pueden añadir tipos/nodos/compilers formalmente;
50. la API común continúa siendo simple.

---

# 321. Experiencia objetivo

Para el desarrollador común:

```php
Schema::create('users', function (TableBlueprint $table) {
    $table->id();

    $table->string('name', 150);

    $table->string('email', 320)
        ->unique();

    $table->boolean('active')
        ->default(true);

    $table->timestamps();
});
```

deberá ser suficiente.

El desarrollador no deberá necesitar conocer:

```text
Schema AST
Schema Planner
Capability Resolver
Platform Compiler
```

para este caso.

---

# 322. Pero internamente

VoltStack deberá realizar:

```text
Schema::create()
       ↓
TableBlueprint
       ↓
TableDefinition
       ↓
CreateTable AST
       ↓
Structural Validation
       ↓
Semantic Validation
       ↓
Capability Resolution
       ↓
Safety Analysis
       ↓
Dependency Planning
       ↓
Platform Compilation
       ↓
Compiled Commands
       ↓
Execution Engine
       ↓
Connection Manager
       ↓
Driver
       ↓
DBMS
```

---

# 323. Filosofía final

La experiencia de Schema de VoltStack deberá perseguir simultáneamente dos objetivos que normalmente parecen opuestos:

```text
Simple Developer API
+
Rigorous Database Architecture
```

El desarrollador deberá poder pensar:

```text
Create users table
```

mientras VoltStack piensa:

```text
What structural intent was requested?

Is the definition valid?

What is the effective platform?

Which capabilities are proven?

Can the requested semantics be represented?

Will emulation be required?

What dependencies exist?

Is the operation destructive?

Could it rewrite the table?

What is the migration context?

What commands represent the operation?

What execution guarantees actually exist?

How will the resulting state be verified?
```

La complejidad deberá pertenecer al framework.

No al código cotidiano de la aplicación.

---

# 324. Regla arquitectónica definitiva

> **El Schema API de VoltStack expresará estructura e intención; el Schema Engine será responsable de validar esa intención, compararla con el estado observado cuando corresponda, resolver capabilities, evaluar seguridad, planificar dependencias y producir una representación física específica de plataforma.**

Por tanto:

```text
Developer Intent
      ↓
Schema Definition
      ↓
Validated Structure
      ↓
Capability-Aware Plan
      ↓
Platform Representation
      ↓
Controlled Execution
      ↓
Observed Result
```

y nunca:

```text
Schema::create()
      ↓
String concatenation
      ↓
PDO::exec()
```

---

# 325. Relación con la arquitectura Database

La posición definitiva será:

```text
Application / Package / Migration
              │
              ▼
         Public Schema API
              │
              ▼
         Schema Builder
              │
              ▼
      Definition / Schema AST
              │
              ▼
          Validation
              │
              ▼
       Capability System
              │
              ▼
          Safety System
              │
              ▼
        Schema Planner
              │
              ▼
       Schema Compiler
              │
              ▼
       Execution Engine
              │
              ▼
      Connection Manager
              │
              ▼
            Driver
              │
              ▼
             DBMS
```

con la ruta observacional:

```text
DBMS
 ↓
Schema Introspection
 ↓
Normalized Schema Model
 ↓
Schema Diff
 ↓
Migration Planning
```

Esto mantiene intacta la arquitectura definida en los documentos anteriores y permite una experiencia similar en ergonomía a frameworks modernos sin sacrificar la separación interna requerida por VoltStack.

---

# 326. Siguiente documento

```text
308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md
```

El siguiente documento definirá la experiencia transversal de errores y diagnósticos de `VoltStack/Quantum/Database`.

Deberá unificar errores provenientes de:

```text
Configuration
Connection
Driver
Query
Semantic Analysis
Planner
Compiler
Execution
Schema
Migration
ORM
Hydration
Relationships
Transactions
Concurrency
Cache
Routing
Sharding
Multitenancy
Security
Resilience
Persistent Runtimes
Backup/Restore
Extensions
```

manteniendo como principio:

> **Un error técnico no deberá limitarse a informar que algo falló; VoltStack deberá indicar qué operación falló, en qué fase, bajo qué contexto, qué evidencia existe, qué parte del resultado es conocida o incierta y, cuando sea seguro hacerlo, qué puede hacer el desarrollador para diagnosticar o corregir el problema.**

Arquitectura esperada:

```text
Failure / Warning / Anomaly
            ↓
     Error Classification
            ↓
      Diagnostic Context
            ↓
       Evidence Model
            ↓
      Redaction/Security
            ↓
      Diagnostic Engine
        ┌───┼────┐
        ▼   ▼    ▼
 Exception CLI  Debug Tools
        │
        ├──────────────┐
        ▼              ▼
   Developer        Telemetry
   Experience
```