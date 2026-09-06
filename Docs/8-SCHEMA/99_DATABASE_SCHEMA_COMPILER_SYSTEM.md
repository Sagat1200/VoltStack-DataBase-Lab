# 99_DATABASE_SCHEMA_COMPILER_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Compiler System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 99 — Database Schema Compiler System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Compiler System` define la arquitectura mediante la cual VoltStack transforma intención estructural previamente validada y planificada en representaciones DDL específicas de una plataforma de base de datos.

Su responsabilidad fundamental será transformar:

```text
Schema AST
+
Schema Change Plan
+
Database Platform
+
Dialect
+
Capability Snapshot
+
Compilation Context
        ↓
Compiled Schema Commands
```

sin ejecutar esos comandos.

La separación central será:

```text
Schema Compiler
≠
Schema Builder

Schema Compiler
≠
Schema Model

Schema Compiler
≠
Schema Diff

Schema Compiler
≠
Migration Planner

Schema Compiler
≠
Execution Engine

Compiled Schema Command
≠
Executed Statement
```

---

# 2. Principio central

> **El Schema Compiler representa una operación estructural validada en el lenguaje DDL de una plataforma concreta; no decide qué cambio debe hacerse ni ejecuta el cambio.**

Por tanto:

```text
Schema AST
      │
      ▼
Schema Planner
      │
      ▼
Planned Schema Operation
      │
      ▼
Schema Compiler
      │
      ▼
Compiled Schema Command
```

y no:

```text
Schema Diff
      │
      ▼
Schema Compiler
      │
      ▼
Database
```

---

# 3. Regla maestra

La regla fundamental será:

```text
Builder constructs intent.
AST represents intent.
Diff discovers differences.
Planner chooses structural strategy.
Compiler renders platform representation.
Executor performs execution.
```

---

# 4. Posición arquitectónica

```text
Schema Builder
      │
      ▼
Schema AST
      │
      ▼
Schema Validation
      │
      ▼
Schema Planning
      │
      ▼
Schema Execution Plan
      │
      ▼
Schema Compiler
      │
      ├── Identifier Renderer
      ├── Type Renderer
      ├── Expression Renderer
      ├── Constraint Renderer
      ├── Index Renderer
      └── Platform Extensions
      │
      ▼
Compiled Schema Command Set
      │
      ▼
Execution Engine
      │
      ▼
Driver
      │
      ▼
Database
```

---

# 5. Responsabilidades

El sistema será responsable de compilar operaciones como:

```text
CREATE DATABASE / NAMESPACE
DROP DATABASE / NAMESPACE

CREATE TABLE
ALTER TABLE
RENAME TABLE
DROP TABLE

ADD COLUMN
ALTER COLUMN
RENAME COLUMN
DROP COLUMN

CREATE INDEX
DROP INDEX

ADD CONSTRAINT
DROP CONSTRAINT
RENAME CONSTRAINT

ADD FOREIGN KEY
DROP FOREIGN KEY

CREATE SEQUENCE
ALTER SEQUENCE
DROP SEQUENCE

CREATE VIEW
ALTER / REPLACE VIEW
DROP VIEW

platform-specific schema operations
```

según capacidades disponibles.

---

# 6. No responsabilidades

El Schema Compiler no deberá:

- decidir qué diferencia existe;
- realizar Schema Diff;
- inferir renames;
- autorizar cambios;
- determinar si una migración es segura;
- consultar datos;
- ejecutar backfills;
- abrir conexiones;
- comenzar transacciones;
- ejecutar DDL;
- decidir retries;
- alterar el Schema Model;
- consultar metadata de manera oculta;
- realizar hidden introspection;
- seleccionar tenant;
- resolver el lifecycle del worker.

---

# 7. Input principal

El compilador no deberá consumir estructuras arbitrarias.

Su input deberá ser un artefacto estructurado como:

```text
PlannedSchemaOperation
```

o:

```text
SchemaExecutionPlan
```

dependiendo de la granularidad final adoptada.

---

# 8. PlannedSchemaOperation

Se propone conceptualmente:

```php
interface PlannedSchemaOperation
{
    public function id(): SchemaOperationId;

    public function kind(): SchemaOperationKind;

    public function requirements(): SchemaCapabilityRequirementSet;

    public function dependencies(): SchemaOperationDependencySet;

    public function metadata(): SchemaOperationMetadata;
}
```

Especializaciones:

```text
CreateTableOperation
AlterTableOperation
DropTableOperation
CreateIndexOperation
AddConstraintOperation
...
```

---

# 9. AST vs planned operation

Debe mantenerse:

```text
Schema AST
=
requested structural intent
```

mientras:

```text
Planned Schema Operation
=
chosen structural execution strategy
```

Ejemplo:

```text
AST:
Alter column email type
```

podría planificarse como:

```text
Plan A:
ALTER COLUMN directly
```

o:

```text
Plan B:
Create replacement table
Copy data
Swap
```

si la plataforma requiere una estrategia compleja.

El compiler no elige entre A y B.

---

# 10. Multi-command compilation

Una operación planificada puede producir:

```text
1..N
```

comandos compilados.

Formalmente:

```text
Compile(Operation)
→
CompiledSchemaCommandSet
```

No necesariamente:

```text
1 operation = 1 SQL statement
```

---

# 11. Ejemplo

Una operación planificada:

```text
RebuildTableOperation(users)
```

puede producir:

```text
CREATE TABLE __tmp_users ...
INSERT INTO __tmp_users ...
DROP TABLE users
ALTER TABLE __tmp_users RENAME TO users
```

pero solo si dicha secuencia ya fue aprobada por el planner.

---

# 12. Compiler must not hide strategy

Incorrecto:

```text
AlterColumnNode
    ↓
compiler secretly rebuilds table
```

Correcto:

```text
AlterColumnNode
    ↓
Schema Planner
    ↓
RebuildTableOperation
    ↓
Compiler
    ↓
explicit command sequence
```

---

# 13. Arquitectura principal

Se propone:

```text
SchemaCompiler
├── SchemaCompilerResolver
├── SchemaCompilationPipeline
├── SchemaOperationCompilerRegistry
├── SchemaIdentifierRenderer
├── SchemaTypeRenderer
├── SchemaExpressionRenderer
├── SchemaConstraintRenderer
├── SchemaIndexRenderer
├── SchemaOptionRenderer
├── PlatformExtensionCompiler
└── CompiledCommandAssembler
```

---

# 14. Contrato principal

```php
interface SchemaCompiler
{
    public function compile(
        SchemaExecutionPlan $plan,
        SchemaCompilationContext $context,
    ): CompiledSchemaPlan;
}
```

Alternativamente, internamente:

```php
interface SchemaOperationCompiler
{
    public function compile(
        PlannedSchemaOperation $operation,
        SchemaCompilationContext $context,
    ): CompiledSchemaCommandSet;
}
```

---

# 15. SchemaCompilationContext

Se propone:

```php
final readonly class SchemaCompilationContext
{
    public function __construct(
        public DatabasePlatform $platform,
        public DatabaseDialect $dialect,
        public PlatformCapabilitySnapshot $capabilities,
        public SchemaCompilationProfile $profile,
        public SchemaCompilationBudget $budget,
        public FrozenSchemaCompilerExtensionRegistry $extensions,
    ) {}
}
```

---

# 16. Contexto explícito

Nunca deberán obtenerse ocultamente:

```text
current platform
current database
current schema
current tenant
current connection
server version
```

desde singletons globales.

---

# 17. Platform compiler

Se propone:

```text
SchemaCompiler
│
├── MySqlSchemaCompiler
├── MariaDbSchemaCompiler
├── PostgreSqlSchemaCompiler
└── SqliteSchemaCompiler
```

Los componentes podrán compartir infraestructura interna.

---

# 18. MariaDB first-class

Debe mantenerse:

```text
MariaDB
≠
MySQL alias
```

Podrán compartir:

```text
common compiler primitives
```

pero tener:

```text
separate capability sets
separate platform rules
separate extensions
```

---

# 19. Dialect vs Platform

El compiler utilizará ambos conceptos.

```text
Dialect
=
SQL rendering conventions
```

```text
Platform
=
database semantic/capability model
```

Por tanto:

```text
Dialect
≠
Platform
```

---

# 20. Driver independence

Schema Compiler no dependerá de:

```text
PDO
PDOStatement
mysqli
pgsql driver handles
```

Su salida será declarativa.

---

# 21. CompiledSchemaCommand

Se propone:

```php
final readonly class CompiledSchemaCommand
{
    public function __construct(
        public CompiledSchemaCommandId $id,
        public SqlText $sql,
        public SchemaCommandKind $kind,
        public SchemaCommandParameterSet $parameters,
        public SchemaCommandExecutionRequirements $requirements,
        public SchemaCommandMetadata $metadata,
    ) {}
}
```

---

# 22. SQL text

El SQL compilado puede ser texto porque esta es precisamente la frontera de representación.

Antes del compiler:

```text
typed structure
```

Después:

```text
platform SQL representation
```

---

# 23. Compiled command ≠ raw untrusted SQL

Aunque contenga SQL:

```text
CompiledSchemaCommand
```

ha pasado por:

```text
typed identifiers
typed expressions
compiler rendering
capability validation
```

salvo escape hatches explícitos.

---

# 24. CompiledSchemaPlan

Se propone:

```php
final readonly class CompiledSchemaPlan
{
    public function __construct(
        public CompiledSchemaCommandSet $commands,
        public SchemaCompiledPlanRequirements $requirements,
        public SchemaCompilationDiagnosticSet $diagnostics,
        public SchemaCompilationFingerprint $fingerprint,
    ) {}
}
```

---

# 25. Command ordering

El orden de:

```text
CompiledSchemaCommandSet
```

será significativo.

Debe preservar:

```text
Schema Execution Plan order
```

salvo transformaciones locales permitidas y demostrablemente equivalentes.

---

# 26. Compiler does not reorder dependencies

El compiler no deberá decidir:

```text
Create B before A
```

porque detectó una FK.

Eso corresponde al planner.

---

# 27. Compilation pipeline

Se propone:

```text
Schema Execution Plan
        │
        ▼
Compilation Input Validation
        │
        ▼
Capability Verification
        │
        ▼
Operation Dispatch
        │
        ▼
Platform Operation Compiler
        │
        ▼
Identifier Rendering
        │
        ▼
Type / Expression Rendering
        │
        ▼
Command Assembly
        │
        ▼
Command Validation
        │
        ▼
Canonical Formatting
        │
        ▼
Fingerprinting
        │
        ▼
Compiled Schema Plan
```

---

# 28. Input validation

Antes de compilar deberá verificarse:

```text
operation is supported
required references are resolved
plan is internally valid
platform context matches
required capabilities exist
extension versions match
```

---

# 29. Capability validation

Formalmente:

```text
Compilable(Operation, Platform)
=
Valid(Operation)
∧
Capabilities(Platform) ⊇ Requirements(Operation)
```

---

# 30. Unsupported capability

Si falta una capability:

```text
fail
```

o delegar a una estrategia de emulación ya planificada.

Nunca:

```text
silently omit unsupported semantics
```

---

# 31. Example: unsupported deferrability

Target operation contiene:

```text
DEFERRABLE INITIALLY DEFERRED
```

pero la plataforma no soporta la capability.

Incorrecto:

```text
compile as normal FK
```

Correcto:

```text
UnsupportedSchemaCapabilityException
```

salvo plan de emulación explícito.

---

# 32. Emulation boundary

Debe distinguirse:

```text
Compiler-level syntactic lowering
```

de:

```text
Planner-level semantic emulation
```

---

# 33. Syntactic lowering

Ejemplo:

```text
BooleanType
```

puede renderizarse físicamente como:

```text
BOOLEAN
```

o:

```text
TINYINT(1)
```

según platform compiler.

Eso es compilación legítima.

---

# 34. Semantic emulation

Ejemplo:

```text
AlterColumn unsupported
```

requiere:

```text
rebuild table
```

Eso no debe ocultarse en compiler.

Es planificación.

---

# 35. Regla de emulación

```text
Same semantics,
different syntax
→ compiler
```

```text
Different operational strategy,
multiple structural steps
→ planner
```

---

# 36. Identifier rendering

Se propone:

```php
interface SchemaIdentifierRenderer
{
    public function render(
        SchemaIdentifier $identifier,
        SchemaIdentifierRenderingContext $context,
    ): RenderedIdentifier;
}
```

---

# 37. Identifier validation precedes rendering

Nunca:

```php
return '"' . $identifier . '"';
```

sobre input arbitrario.

---

# 38. Structured identifiers

Tipos:

```text
TableIdentifier
ColumnIdentifier
IndexIdentifier
ConstraintIdentifier
SequenceIdentifier
NamespaceIdentifier
```

serán renderizados mediante reglas del dialect.

---

# 39. Qualified identifiers

Ejemplo:

```text
catalog?
namespace?
table
```

se renderizará con:

```text
PlatformQualifiedIdentifierRenderer
```

sin concatenar `"."` indiscriminadamente.

---

# 40. Reserved words

Identifiers que coincidan con reserved words podrán ser quoted correctamente.

No deberán rechazarse necesariamente si el dialect permite quoting.

---

# 41. Case preservation

El renderer respetará:

```text
identifier declaration
platform folding rules
quoting policy
```

sin blind lowercase/uppercase.

---

# 42. Identifier rendering modes

Se podría definir:

```text
AUTO
ALWAYS_QUOTE
QUOTE_WHEN_REQUIRED
PLATFORM_DEFAULT
```

como compilation profile.

---

# 43. Deterministic quoting

La misma entrada/contexto deberá producir el mismo output.

---

# 44. Type rendering

Se propone:

```php
interface SchemaTypeRenderer
{
    public function render(
        DatabaseType $type,
        SchemaTypeRenderingContext $context,
    ): RenderedDatabaseType;
}
```

---

# 45. Logical type → physical type

Ejemplo:

```text
StringType(length=255)
    ↓
VARCHAR(255)
```

---

# 46. Cross-platform type rendering

Ejemplo:

```text
UuidType
```

puede compilarse como:

```text
PostgreSQL → UUID
MySQL     → CHAR/BINARY strategy
SQLite    → TEXT/BLOB strategy
```

dependiendo de la definición y capabilities.

---

# 47. Compiler must preserve semantics

No deberá elegir un tipo físico que cambie silenciosamente:

```text
range
precision
nullability
timezone semantics
```

sin autorización/policy explícita.

---

# 48. Decimal example

```text
DecimalType(
    precision = 18,
    scale = 4
)
```

deberá renderizarse de forma compatible con ambas propiedades.

---

# 49. Unknown types

Para:

```text
OpaquePlatformType
```

solo un compiler compatible podrá renderizarlo.

Un compiler de otra plataforma deberá fallar o aplicar una extensión explícita.

---

# 50. Type compiler registry

Podrá existir:

```text
SchemaTypeCompilerRegistry
```

frozen tras bootstrap.

---

# 51. Default value rendering

Se propone:

```text
ColumnDefaultRenderer
```

para:

```text
NoDefault
LiteralDefault
NullDefault
ExpressionDefault
GeneratedDefault
```

---

# 52. NoDefault

Debe compilarse como:

```text
absence of DEFAULT clause
```

No como:

```text
DEFAULT NULL
```

---

# 53. Literal defaults

Los valores deberán renderizarse con reglas seguras del dialect.

Ejemplo:

```text
LiteralDefault("CURRENT_TIMESTAMP")
```

deberá compilar como literal string.

No función.

---

# 54. Expression defaults

```text
SchemaExpression::currentTimestamp()
```

podrá compilar a la expresión correspondiente.

---

# 55. Literal ≠ expression

Regla crítica:

```text
"CURRENT_TIMESTAMP"
≠
CURRENT_TIMESTAMP
```

---

# 56. Schema expression compiler

Se propone:

```php
interface SchemaExpressionRenderer
{
    public function render(
        SchemaExpression $expression,
        SchemaExpressionCompilationContext $context,
    ): CompiledSchemaExpression;
}
```

---

# 57. Schema expressions

Podrán utilizarse en:

```text
defaults
generated columns
check constraints
expression indexes
partial index predicates
```

---

# 58. Schema expression ≠ query expression

El compiler debe usar el conjunto de expresiones permitido para schema.

No aceptar arbitrariamente el AST completo de Query Engine.

---

# 59. Raw schema expressions

Se representarán mediante:

```text
RawSchemaExpression
```

con trust/platform metadata.

---

# 60. Raw expression compilation

Regla:

```text
RawSchemaExpression
```

solo podrá compilarse si:

```text
trusted
+
platform-compatible
+
policy allows raw
```

---

# 61. Raw SQL escape hatch

Podrá existir:

```text
RawSchemaOperation
```

pero deberá evitar pasar por renderers que pretendan analizar su contenido.

---

# 62. Raw operation metadata

Debe incluir:

```text
platform scope
trust
origin
portability
execution requirements
```

---

# 63. Raw ≠ safe

Siempre:

```text
RAW
≠
SAFE
```

---

# 64. Create Table compilation

Input:

```text
CreateTableOperation
└── TableDefinition
```

Output conceptual:

```text
CREATE TABLE ...
```

---

# 65. Create table renderer

Debe compilar:

```text
table identifier
columns
primary key
constraints
foreign keys
table options
temporary/persistence semantics
```

según la estrategia ya planificada.

---

# 66. Inline vs separate constraints

El compiler puede decidir renderizado sintáctico:

```text
CREATE TABLE (...) CONSTRAINT ...
```

frente a otra forma equivalente.

Pero no decidir si una FK debe diferirse a operación posterior si eso cambia el plan.

---

# 67. Column compilation

Cada `ColumnDefinition` deberá convertirse mediante:

```text
identifier
type
nullability
default
generation
charset/collation
column options
```

---

# 68. Column modifier ordering

El compiler deberá definir un orden canónico por plataforma.

Ejemplo conceptual:

```text
name type generation nullability default ...
```

según gramática concreta.

---

# 69. Table options

Se compilarán mediante:

```text
SchemaTableOptionRenderer
```

No como generic string bags.

---

# 70. Unsupported table option

Debe fallar o ignorarse solo si una policy explícita permite degradación informacional.

Nunca silent drop para metadata estructural/semántica.

---

# 71. Alter Table compilation

`AlterTableOperation` deberá contener una estrategia suficientemente explícita.

Ejemplo:

```text
AlterTableOperation
├── AddColumn
├── DropColumn
├── AlterColumn
└── AddConstraint
```

si la plataforma permite combinarlas.

---

# 72. Combined ALTER

Si el planner ha permitido combinar múltiples cambios:

```text
ALTER TABLE users
    ADD ...,
    DROP ...,
    ...
```

el compiler podrá renderizarlos juntos.

---

# 73. Combined vs separate commands

Debe decidirse antes o durante compilación únicamente cuando ambas formas sean operacionalmente equivalentes bajo el plan.

Si afecta:

```text
locking
transactionality
failure isolation
online behavior
```

deberá ser decisión del planner.

---

# 74. Rename Table

Input:

```text
RenameTableOperation
```

se compilará mediante la sintaxis soportada.

---

# 75. Drop Table

Debe respetar:

```text
DropBehavior
├── REQUIRE_PRESENT
└── IGNORE_IF_MISSING
```

y:

```text
DependencyDropBehavior
├── RESTRICT
├── CASCADE
└── PLATFORM_DEFAULT
```

cuando sea soportado.

---

# 76. CASCADE safety

El compiler solo renderiza:

```text
CASCADE
```

si ya forma parte de la operación.

Nunca lo añade para "hacer que funcione".

---

# 77. Create Index compilation

Input:

```text
CreateIndexOperation
└── IndexDefinition
```

Debe compilar:

```text
uniqueness
index identifier
table
keys
expressions
ordering
included columns
predicate
method
persistent options
```

---

# 78. Concurrent index creation

Si el plan contiene:

```text
IndexCreationMode::CONCURRENT
```

el PostgreSQL compiler puede renderizarlo.

Pero esa preferencia operacional no pertenece a `IndexDefinition`.

---

# 79. Index expression rendering

Debe usar:

```text
SchemaExpressionRenderer
```

y no concatenación arbitraria.

---

# 80. Included columns

Solo se renderizarán si:

```text
capability supported
```

o el plan ya definió una emulación válida.

---

# 81. Partial index

Igualmente:

```text
predicate
```

solo se renderizará con capability adecuada.

---

# 82. Drop Index

La sintaxis puede variar por plataforma:

```text
DROP INDEX index
```

vs formas que necesitan tabla/namespace.

La diferencia pertenece al platform compiler.

---

# 83. Constraint compilation

`ConstraintDefinition` deberá despacharse por tipo:

```text
PrimaryKey
UniqueConstraint
CheckConstraint
ForeignKey
ExtensionConstraint
```

---

# 84. Primary key rendering

El compiler podrá producir:

```text
PRIMARY KEY (...)
```

con nombre cuando la plataforma lo permita/requiera.

---

# 85. Unique constraint rendering

No deberá compilarse automáticamente como:

```text
UNIQUE INDEX
```

salvo que el platform plan haya determinado una equivalencia física apropiada.

---

# 86. Check constraint rendering

Debe compilar:

```text
CHECK (
    CompiledSchemaExpression
)
```

respetando:

```text
enforcement
validation
platform capabilities
```

---

# 87. Foreign key rendering

Debe compilar:

```text
FOREIGN KEY (...)
REFERENCES ...
ON UPDATE ...
ON DELETE ...
MATCH ...
DEFERRABLE ...
```

solo con propiedades soportadas.

---

# 88. Foreign key order

Composite mapping deberá preservar:

```text
local[0] → referenced[0]
local[1] → referenced[1]
```

Nunca ordenar ambos lados independientemente.

---

# 89. NO ACTION vs RESTRICT

Si el target declara:

```text
NO_ACTION
```

el compiler no debe sustituirlo por:

```text
RESTRICT
```

salvo equivalencia explícitamente definida por la plataforma y comparison/compatibility model.

---

# 90. Constraint validation clauses

Properties como:

```text
NOT VALID
NOT ENFORCED
```

solo se renderizarán con capabilities correspondientes.

---

# 91. Sequence compilation

Debe soportar cuando aplique:

```text
CREATE SEQUENCE
ALTER SEQUENCE
DROP SEQUENCE
```

y propiedades:

```text
start
increment
min
max
cycle
cache
ownership
```

según plan/capabilities.

---

# 92. Unsupported sequences

Para plataformas sin sequences:

```text
CreateSequenceOperation
```

deberá haber sido rechazado o transformado por planning.

El compiler no deberá improvisar una tabla de secuencias.

---

# 93. View compilation

Debe soportar:

```text
CreateViewOperation
DropViewOperation
ReplaceViewOperation
```

según capabilities.

---

# 94. Structured view

Cuando exista un AST estructurado:

```text
ViewDefinition
    ↓
ViewQueryCompiler
```

podrá colaborar con la infraestructura de Query SQL Compiler cuidadosamente.

---

# 95. Avoid circular dependency

Debe evitarse:

```text
Schema Compiler
    ↓
Query Compiler
    ↓
Schema Semantic Resolver
    ↓
Schema Compiler
```

---

# 96. Shared SQL expression infrastructure

Podrán compartirse:

```text
identifier rendering
literal rendering
expression primitives
```

mediante una capa común inferior.

No mediante dependencia circular entre subsistemas de alto nivel.

---

# 97. Opaque view

Para:

```text
OpaqueViewDefinition
```

solo podrá renderizarse si:

```text
platform-compatible raw definition
+
trust policy permits
```

---

# 98. Namespace compilation

Operaciones:

```text
CreateNamespace
DropNamespace
RenameNamespace?
```

dependerán del modelo de plataforma.

---

# 99. Catalog != schema

El compiler no deberá asumir:

```text
catalog = schema
```

universalmente.

---

# 100. Temporary tables

`TablePersistence::TEMPORARY` deberá renderizarse mediante la sintaxis apropiada.

---

# 101. Temporary table lifetime

Si existe:

```text
session
transaction
connection
```

scope específico, deberá ser capability/property explícita.

No inventada por el compiler.

---

# 102. Platform extensions

Se propone:

```text
SchemaCompilerExtension
```

capaz de registrar:

```text
custom operation compilers
custom type renderers
custom option renderers
custom expression renderers
```

---

# 103. Extension compilation contract

```php
interface SchemaCompilerExtension
{
    public function supports(
        PlannedSchemaOperation $operation,
        SchemaCompilationContext $context,
    ): bool;

    public function compile(
        PlannedSchemaOperation $operation,
        SchemaCompilationContext $context,
    ): CompiledSchemaCommandSet;
}
```

---

# 104. Frozen compiler registry

Después de bootstrap:

```text
Mutable Registry
      ↓
freeze
      ↓
FrozenSchemaCompilerExtensionRegistry
```

---

# 105. No last-wins

Dos compilers que reclaman exactamente la misma operación sin prioridad/resolution contract deberán producir conflicto.

---

# 106. Compiler dispatch

Podrá resolverse mediante:

```text
OperationKind
+
Platform
+
Capability Domain
+
Extension Type
```

---

# 107. Deterministic dispatch

Mismo input/contexto:

```text
same compiler selected
```

---

# 108. Compile result purity

Idealmente:

```text
Compile(Input, Context)
→
Output
```

sin side effects.

---

# 109. No hidden DB I/O

El compiler jamás deberá:

```text
query information_schema
query pg_catalog
open connection
execute PRAGMA
```

para decidir sintaxis.

Toda información deberá venir en el contexto.

---

# 110. No server-version lookup

Tampoco:

```text
SELECT VERSION()
```

dentro del compiler.

El capability snapshot ya debe representar las capacidades.

---

# 111. Version ≠ capability

Regla:

```text
compiler asks capabilities
```

no:

```text
compiler branches on vendor version
```

salvo dentro del componente que construye el capability snapshot, fuera del compiler.

---

# 112. Command parameters

DDL suele tener restricciones respecto de placeholders.

Debe distinguirse:

```text
value parameter
identifier
schema expression
DDL literal
```

---

# 113. Prepared statements

No se asumirá que todo DDL puede ser parametrizado.

```text
CREATE TABLE ?
```

no es una estrategia portable para identifiers.

---

# 114. Identifier safety

Identifiers se protegen mediante:

```text
validation
+
typed representation
+
dialect quoting
```

No mediante value bindings.

---

# 115. DDL values

Si alguna plataforma permite o requiere bound parameters para valores en DDL, podrán representarse mediante:

```text
SchemaCommandParameterSet
```

pero la política será explícita.

---

# 116. Default strategy

Preferiblemente la mayoría del DDL será compilado de forma completa y segura mediante renderers tipados.

---

# 117. Statement boundaries

Un `CompiledSchemaCommand` deberá representar una unidad que el Execution Engine pueda ejecutar/controlar.

---

# 118. Multi-statement strings

Evitar por defecto:

```text
"CREATE ...; ALTER ...; DROP ...;"
```

dentro de un único comando.

Preferir:

```text
Command 1
Command 2
Command 3
```

---

# 119. Why separate commands

Permite:

```text
telemetry
error attribution
cancellation
transaction planning
retry analysis
diagnostics
```

---

# 120. Compound commands

Solo si un dialect requiere una unidad textual compuesta podrá existir:

```text
CompoundCompiledSchemaCommand
```

explícito.

---

# 121. Execution requirements

Cada comando podrá declarar:

```text
SchemaCommandExecutionRequirements
├── TransactionRequirement
├── LockExpectation
├── SessionRequirement
├── AutocommitRequirement
├── ConnectionAffinity
├── CannotRunInsideTransaction?
└── CapabilityRequirements
```

---

# 122. Requirement ≠ execution

Compiler declara:

```text
requires no transaction
```

Executor/transaction orchestrator decide cómo cumplirlo.

---

# 123. PostgreSQL CONCURRENTLY example

Un comando:

```text
CREATE INDEX CONCURRENTLY ...
```

puede declarar:

```text
MustRunOutsideTransactionBlock
```

---

# 124. DDL transaction behavior

No asumir:

```text
DDL always transactional
```

ni:

```text
DDL never transactional
```

---

# 125. Transactional classification

Se propone:

```text
SchemaCommandTransactionBehavior
├── ALLOWED_IN_TRANSACTION
├── REQUIRED_IN_TRANSACTION
├── FORBIDDEN_IN_TRANSACTION
├── IMPLICIT_COMMIT_POSSIBLE
├── PLATFORM_MANAGED
└── UNKNOWN
```

---

# 126. MySQL implicit commits

El platform compiler/capability context podrá reflejar que determinadas operaciones tienen comportamiento transaccional particular.

No ocultarlo.

---

# 127. SQLite rebuild plan

Si planner produce:

```text
RebuildTableOperation
```

el SQLite compiler renderiza sus pasos.

No decide espontáneamente hacer rebuild.

---

# 128. Command metadata

Puede incluir:

```text
source operation ID
schema object path
platform
compilation version
source AST node IDs
risk annotations
```

sin incluir secretos.

---

# 129. Traceability

Debe ser posible:

```text
CompiledCommand
      │
      ▼
PlannedOperation
      │
      ▼
Schema AST node
```

para diagnostics.

---

# 130. Source map

Se propone:

```text
SchemaCompilationSourceMap
```

---

# 131. Source map example

```text
command-17
→ operation-5
→ ast-node-31
→ builder source database/schema/users.php:41
```

---

# 132. Compiler diagnostics

Ejemplos:

```text
UNSUPPORTED_SCHEMA_CAPABILITY
UNRENDERABLE_DATABASE_TYPE
INVALID_IDENTIFIER
UNSUPPORTED_SCHEMA_EXPRESSION
UNSUPPORTED_TABLE_OPTION
UNSUPPORTED_CONSTRAINT_OPTION
RAW_SCHEMA_OPERATION_PLATFORM_MISMATCH
COMPILER_EXTENSION_CONFLICT
```

---

# 133. Compilation warning

Puede existir warning para:

```text
portable semantics compiled with physical compromise
```

solo cuando una policy lo permite sin alterar correctness.

---

# 134. Silent degradation prohibited

Nunca:

```text
target requests feature X
platform does not support X
    ↓
ignore X
```

para propiedades estructurales/semánticas.

---

# 135. Compilation profiles

Se propone:

```text
SchemaCompilationProfile
├── DEFAULT
├── PORTABLE_STRICT
├── PLATFORM_STRICT
├── CANONICAL
├── DEBUG
└── MIGRATION
```

---

# 136. PORTABLE_STRICT

Rechaza features fuera del portable core.

Útil para proyectos multi-platform.

---

# 137. PLATFORM_STRICT

Permite features específicas solo si la plataforma/capabilities las soportan exactamente.

---

# 138. CANONICAL

Produce SQL estable y reproducible para:

```text
tests
snapshots
fingerprints
```

---

# 139. DEBUG

Puede añadir:

```text
safe comments
source annotations
diagnostic mapping
```

si el dialect/policy lo permiten.

---

# 140. Canonical SQL formatting

Se recomienda que la salida sea determinista:

```text
same input
+
same platform
+
same capabilities
+
same compiler version
=
same SQL
```

---

# 141. Formatting ≠ semantics

El compiler podrá separar:

```text
SQL AST / token stream
```

de:

```text
SQL formatter
```

---

# 142. Intermediate DDL representation

Puede ser conveniente introducir:

```text
SchemaSqlAst
```

interno.

Pipeline:

```text
Planned Operation
      ↓
Platform DDL IR
      ↓
SQL Renderer
      ↓
SQL Text
```

---

# 143. Benefit

Permite separar:

```text
semantic lowering
```

de:

```text
string rendering
```

---

# 144. DDL IR

Podría representar:

```text
CreateTableStatement
AlterTableStatement
CreateIndexStatement
AddConstraintClause
ColumnDefinitionClause
```

solo dentro del compiler subsystem.

---

# 145. Schema AST ≠ DDL AST

Debe mantenerse:

```text
Schema AST
=
platform-neutral structural intent
```

```text
DDL IR
=
platform-oriented SQL representation
```

---

# 146. DDL IR not public core model

No deberá convertirse en una segunda API pública de schema.

Es implementation detail del compiler.

---

# 147. Token rendering

Puede existir:

```text
SqlTokenStream
```

para:

```text
keywords
identifiers
literals
operators
punctuation
```

---

# 148. Safe literal renderer

Se propone:

```text
DatabaseLiteralRenderer
```

compartido con SQL Compiler cuando sea viable.

---

# 149. Avoid duplicated quoting logic

Idealmente:

```text
Query SQL Compiler
Schema Compiler
```

reutilizan primitivas comunes:

```text
IdentifierRenderer
LiteralRenderer
DialectGrammar
```

sin compartir responsabilidades de alto nivel.

---

# 150. Shared low-level SQL infrastructure

Puede residir en:

```text
VoltStack\Quantum\Database\Sql
```

o:

```text
VoltStack\Platform\Database\Sql
```

según estructura final.

---

# 151. Compiler cache

Dado que compilación puede ser determinista:

```text
SchemaCompiledCommandCache
```

podría añadirse posteriormente.

Pero no será dependencia obligatoria.

---

# 152. Cache key

Conceptualmente:

```text
operation fingerprint
+
platform identity
+
capability fingerprint
+
compiler version
+
profile
+
extension registry fingerprint
```

---

# 153. No live objects in cache

No cachear:

```text
Connection
Driver Statement
Transaction
Request
Tenant Context
```

---

# 154. Compilation fingerprint

Se propone:

```text
SchemaCompilationFingerprint
```

---

# 155. Fingerprint formula

```text
Fingerprint
=
H(
    PlannedOperationFingerprint,
    PlatformId,
    CapabilityFingerprint,
    CompilerVersion,
    CompilationProfile,
    ExtensionRegistryFingerprint
)
```

---

# 156. SQL hash

Puede existir adicionalmente:

```text
CompiledSqlFingerprint
```

pero no reemplaza la identidad estructural.

---

# 157. Compiler versioning

Cambios en:

```text
rendering rules
canonical formatting
type mappings
DDL grammar
```

deberán poder versionarse.

---

# 158. Backward compatibility

No se deberá prometer:

```text
byte-identical SQL forever
```

sin una versión canónica explícita.

---

# 159. Error hierarchy

Se propone:

```text
DatabaseSchemaCompilerException
├── InvalidSchemaCompilationInputException
├── UnsupportedSchemaOperationException
├── UnsupportedSchemaCapabilityException
├── SchemaCompilationContextException
├── SchemaIdentifierCompilationException
├── SchemaTypeCompilationException
├── SchemaExpressionCompilationException
├── SchemaDefaultCompilationException
├── SchemaColumnCompilationException
├── SchemaTableCompilationException
├── SchemaIndexCompilationException
├── SchemaConstraintCompilationException
├── SchemaForeignKeyCompilationException
├── SchemaSequenceCompilationException
├── SchemaViewCompilationException
├── SchemaOptionCompilationException
├── RawSchemaCompilationException
├── SchemaCompilerExtensionException
├── SchemaCompilationBudgetExceededException
└── SchemaCompilerInvariantException
```

---

# 160. Error certainty

Compiler failure será:

```text
no command produced for failed unit
```

No deberá fingir éxito parcial salvo `CompiledSchemaPlan` explícitamente marcado y una policy que lo permita.

---

# 161. Atomic compilation

La compilación completa del plan debería ser:

```text
all-or-error
```

por defecto.

---

# 162. Partial compilation

Para tooling/debugging podrá existir:

```text
PartialSchemaCompilationResult
```

pero nunca deberá confundirse con un plan ejecutable completo.

---

# 163. Compilation completeness

Se propone:

```text
COMPLETE
PARTIAL
FAILED
```

---

# 164. Budget model

```php
final readonly class SchemaCompilationBudget
{
    public function __construct(
        public int $maxOperations,
        public int $maxCommands,
        public int $maxExpressionDepth,
        public int $maxSqlBytes,
        public int $maxIdentifiers,
        public int $maxExtensionCalls,
    ) {}
}
```

---

# 165. Budget overflow

Debe producir:

```text
SchemaCompilationBudgetExceededException
```

Nunca truncar SQL.

---

# 166. Cancellation

Compilaciones muy grandes pueden soportar:

```text
CancellationToken
```

aunque normalmente sea pure CPU work.

---

# 167. Cancelled compilation

No podrá marcarse:

```text
COMPLETE
```

---

# 168. Performance target

Para `n` operaciones:

```text
Compilation ≈ O(n + total expression size)
```

en ausencia de extensiones costosas.

---

# 169. No quadratic identifier scans

Utilizar:

```text
registries/maps
```

para resolver renderers y compilers.

---

# 170. Memory

Debe evitarse almacenar múltiples copias grandes de:

```text
schema
DDL IR
SQL text
```

innecesariamente.

---

# 171. Streaming assembly

Para planes enormes podría ensamblarse:

```text
command by command
```

pero el plan final publicado permanecerá inmutable.

---

# 172. Persistent runtime safety

Shared immutable:

```text
Compiler registries
Dialect grammar
Type renderers
Expression renderers
Frozen extension registries
```

Operation-scoped:

```text
CompilationSession
SourceMapBuilder
Diagnostics
CommandAssembler
Temporary token buffers
```

---

# 173. No mutable current platform

Nunca:

```php
SchemaCompiler::$platform;
```

---

# 174. FrankenPHP

Cada request/operation tendrá su propio:

```text
SchemaCompilationSession
```

---

# 175. RoadRunner/OpenSwoole

La misma regla aplica.

No mantener:

```text
last compiled schema
current tenant
current connection
```

en singletons mutables.

---

# 176. Multitenancy

Schema Compiler será tenant-agnostic.

Puede recibir:

```text
qualified identifiers
planned operations
```

ya resueltos por capas superiores.

---

# 177. No tenant resolution

Compiler no deberá hacer:

```text
TenantContext → database/schema
```

---

# 178. Tenant isolation

Compiled plan cache deberá incluir scope identity donde sea necesario para evitar colisiones entre tenants con nombres lógicos iguales pero destinos físicos distintos.

---

# 179. Security model

Compiler está en una frontera sensible porque produce SQL.

Debe tener:

```text
strict typed inputs
trusted renderers
raw escape-hatch controls
extension boundaries
output budgets
```

---

# 180. Identifier injection prevention

La seguridad de identifiers será:

```text
Typed Identifier
+
Validation
+
Dialect Rendering
```

---

# 181. Value injection prevention

Literals deberán pasar por:

```text
typed literal renderer
```

o parámetros cuando aplique.

---

# 182. Raw input prohibition

No deberá aceptar:

```php
$compiler->compile("ALTER TABLE $input");
```

como API normal.

---

# 183. Raw schema API

Si existe:

```text
RawSchemaOperation
```

deberá estar claramente separada y auditada.

---

# 184. Extension security

Un extension compiler no podrá:

- abrir conexiones;
- ejecutar SQL;
- leer secrets;
- acceder a request globals;
- bypass capability checks;
- mutate plan;
- modify another command after emission.

---

# 185. Output security metadata

Un comando podrá marcarse:

```text
containsRawSql
containsSensitiveMetadata
requiresRedaction
```

para diagnostics/telemetry.

---

# 186. Telemetry

Podrá registrar:

```text
compilation duration
operation count
command count
compiler platform
capability fingerprint
SQL byte size
diagnostic count
cache hit/miss
```

---

# 187. Telemetry privacy

Por defecto no registrar:

```text
full DDL
raw expressions
schema comments
sensitive identifiers
```

sin redaction profile.

---

# 188. Testing strategy

Debe incluir:

```text
unit compilation tests
identifier rendering tests
type rendering tests
expression rendering tests
constraint rendering tests
platform conformance
canonical formatting
determinism
capability failures
raw boundary tests
extension tests
budget tests
persistent runtime tests
```

---

# 189. Golden SQL tests

Pueden utilizarse snapshots:

```text
Input Operation
      ↓
Compiler
      ↓
Expected SQL
```

por plataforma.

---

# 190. Golden tests warning

No deberán reemplazar pruebas estructurales.

Un mismo SQL visual puede ocultar errores en:

```text
metadata
execution requirements
source mapping
command boundaries
```

---

# 191. Cross-platform tests

Mismo input lógico:

```text
CreateTable(users)
```

se compilará para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

comparando semántica esperada, no byte identity entre plataformas.

---

# 192. Round-trip integration test

Ideal:

```text
SchemaDefinition
      ↓
Schema AST
      ↓
Plan
      ↓
Compile
      ↓
Execute
      ↓
Introspect
      ↓
PlatformNormalizedCompare
```

---

# 193. Determinism property

Debe cumplirse:

```text
Compile(P, C)
=
Compile(P, C)
```

para mismos input/context/version.

---

# 194. No-I/O property

Unit tests podrán garantizar que compiler funciona sin conexión real.

---

# 195. Unsupported feature tests

Por ejemplo:

```text
Partial Index
+
Platform without PARTIAL_INDEX
```

deberá fallar explícitamente.

---

# 196. Raw expression tests

Verificar:

```text
platform mismatch
untrusted raw
oversized raw
unsupported raw policy
```

---

# 197. Command boundary tests

Verificar que:

```text
multi-step plan
```

produce comandos separados y ordenados correctamente.

---

# 198. Transaction requirement tests

Verificar que operaciones como:

```text
concurrent index creation
```

produzcan los requirements esperados sin que compiler maneje la transacción.

---

# 199. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Compiler
```

---

# 200. Proposed directory structure

```text
Schema/
└── Compiler/
    ├── Contract/
    │   ├── SchemaCompiler.php
    │   ├── SchemaOperationCompiler.php
    │   ├── SchemaIdentifierRenderer.php
    │   ├── SchemaTypeRenderer.php
    │   ├── SchemaExpressionRenderer.php
    │   └── SchemaOptionRenderer.php
    │
    ├── Core/
    │   ├── SchemaCompilationContext.php
    │   ├── SchemaCompilationProfile.php
    │   ├── SchemaCompilationSession.php
    │   └── SchemaCompilationFingerprint.php
    │
    ├── Pipeline/
    │   ├── SchemaCompilationPipeline.php
    │   ├── SchemaCompilationStage.php
    │   └── SchemaCompilationResult.php
    │
    ├── Command/
    │   ├── CompiledSchemaCommand.php
    │   ├── CompiledSchemaCommandId.php
    │   ├── CompiledSchemaCommandSet.php
    │   ├── CompiledSchemaPlan.php
    │   ├── SchemaCommandKind.php
    │   ├── SchemaCommandParameterSet.php
    │   ├── SchemaCommandMetadata.php
    │   └── SchemaCommandExecutionRequirements.php
    │
    ├── Transaction/
    │   └── SchemaCommandTransactionBehavior.php
    │
    ├── Identifier/
    │   ├── SchemaIdentifierRenderingContext.php
    │   ├── RenderedIdentifier.php
    │   └── QualifiedIdentifierRenderer.php
    │
    ├── Type/
    │   ├── SchemaTypeRenderingContext.php
    │   ├── RenderedDatabaseType.php
    │   └── SchemaTypeCompilerRegistry.php
    │
    ├── Expression/
    │   ├── SchemaExpressionCompilationContext.php
    │   ├── CompiledSchemaExpression.php
    │   ├── SchemaExpressionRenderer.php
    │   └── RawSchemaExpressionCompiler.php
    │
    ├── Default/
    │   └── ColumnDefaultRenderer.php
    │
    ├── Table/
    │   ├── CreateTableOperationCompiler.php
    │   ├── AlterTableOperationCompiler.php
    │   ├── RenameTableOperationCompiler.php
    │   └── DropTableOperationCompiler.php
    │
    ├── Column/
    │   ├── ColumnDefinitionCompiler.php
    │   ├── AddColumnOperationCompiler.php
    │   ├── AlterColumnOperationCompiler.php
    │   ├── RenameColumnOperationCompiler.php
    │   └── DropColumnOperationCompiler.php
    │
    ├── Index/
    │   ├── IndexDefinitionCompiler.php
    │   ├── CreateIndexOperationCompiler.php
    │   └── DropIndexOperationCompiler.php
    │
    ├── Constraint/
    │   ├── ConstraintDefinitionCompiler.php
    │   ├── PrimaryKeyConstraintCompiler.php
    │   ├── UniqueConstraintCompiler.php
    │   ├── CheckConstraintCompiler.php
    │   ├── AddConstraintOperationCompiler.php
    │   └── DropConstraintOperationCompiler.php
    │
    ├── ForeignKey/
    │   └── ForeignKeyDefinitionCompiler.php
    │
    ├── Sequence/
    │   ├── CreateSequenceOperationCompiler.php
    │   ├── AlterSequenceOperationCompiler.php
    │   └── DropSequenceOperationCompiler.php
    │
    ├── View/
    │   ├── CreateViewOperationCompiler.php
    │   ├── ReplaceViewOperationCompiler.php
    │   └── DropViewOperationCompiler.php
    │
    ├── Option/
    │   ├── TableOptionRenderer.php
    │   ├── ColumnOptionRenderer.php
    │   └── IndexOptionRenderer.php
    │
    ├── Ddl/
    │   ├── SchemaDdlAst.php
    │   ├── SchemaDdlRenderer.php
    │   └── SqlTokenStream.php
    │
    ├── Platform/
    │   ├── MySQL/
    │   │   ├── MySqlSchemaCompiler.php
    │   │   ├── MySqlTypeRenderer.php
    │   │   ├── MySqlExpressionRenderer.php
    │   │   └── MySqlDdlRenderer.php
    │   ├── MariaDB/
    │   │   ├── MariaDbSchemaCompiler.php
    │   │   ├── MariaDbTypeRenderer.php
    │   │   ├── MariaDbExpressionRenderer.php
    │   │   └── MariaDbDdlRenderer.php
    │   ├── PostgreSQL/
    │   │   ├── PostgreSqlSchemaCompiler.php
    │   │   ├── PostgreSqlTypeRenderer.php
    │   │   ├── PostgreSqlExpressionRenderer.php
    │   │   └── PostgreSqlDdlRenderer.php
    │   └── SQLite/
    │       ├── SqliteSchemaCompiler.php
    │       ├── SqliteTypeRenderer.php
    │       ├── SqliteExpressionRenderer.php
    │       └── SqliteDdlRenderer.php
    │
    ├── Registry/
    │   ├── SchemaOperationCompilerRegistry.php
    │   └── FrozenSchemaOperationCompilerRegistry.php
    │
    ├── Extension/
    │   ├── SchemaCompilerExtension.php
    │   └── FrozenSchemaCompilerExtensionRegistry.php
    │
    ├── SourceMap/
    │   ├── SchemaCompilationSourceMap.php
    │   └── SchemaCompilationSourceReference.php
    │
    ├── Budget/
    │   └── SchemaCompilationBudget.php
    │
    ├── Diagnostic/
    │   ├── SchemaCompilationDiagnostic.php
    │   └── SchemaCompilationDiagnosticSet.php
    │
    └── Exception/
        ├── DatabaseSchemaCompilerException.php
        ├── InvalidSchemaCompilationInputException.php
        ├── UnsupportedSchemaOperationException.php
        ├── UnsupportedSchemaCapabilityException.php
        ├── SchemaCompilationContextException.php
        ├── SchemaIdentifierCompilationException.php
        ├── SchemaTypeCompilationException.php
        ├── SchemaExpressionCompilationException.php
        ├── SchemaDefaultCompilationException.php
        ├── SchemaColumnCompilationException.php
        ├── SchemaTableCompilationException.php
        ├── SchemaIndexCompilationException.php
        ├── SchemaConstraintCompilationException.php
        ├── SchemaForeignKeyCompilationException.php
        ├── SchemaSequenceCompilationException.php
        ├── SchemaViewCompilationException.php
        ├── SchemaOptionCompilationException.php
        ├── RawSchemaCompilationException.php
        ├── SchemaCompilerExtensionException.php
        ├── SchemaCompilationBudgetExceededException.php
        └── SchemaCompilerInvariantException.php
```

---

# 201. Architectural invariants

## DB-SCHEMA-COMP-001
Schema Compiler será distinto de Schema Builder.

## DB-SCHEMA-COMP-002
Schema Compiler será distinto de Schema Model.

## DB-SCHEMA-COMP-003
Schema Compiler será distinto de Schema Diff.

## DB-SCHEMA-COMP-004
Schema Compiler será distinto de Migration Planner.

## DB-SCHEMA-COMP-005
Schema Compiler será distinto de Execution Engine.

## DB-SCHEMA-COMP-006
Schema Compiler no ejecutará SQL.

## DB-SCHEMA-COMP-007
Schema Compiler no abrirá conexiones.

## DB-SCHEMA-COMP-008
Schema Compiler no iniciará transacciones.

## DB-SCHEMA-COMP-009
Schema Compiler no realizará introspection.

## DB-SCHEMA-COMP-010
Schema Compiler no consultará row data.

## DB-SCHEMA-COMP-011
Compiler consumirá operaciones ya planificadas.

## DB-SCHEMA-COMP-012
Compiler no decidirá Schema Diff.

## DB-SCHEMA-COMP-013
Compiler no inferirá renames.

## DB-SCHEMA-COMP-014
Compiler no decidirá destructive safety.

## DB-SCHEMA-COMP-015
Compiler no decidirá authorization.

## DB-SCHEMA-COMP-016
Schema AST será distinto de DDL IR.

## DB-SCHEMA-COMP-017
DDL IR será implementation detail del compiler.

## DB-SCHEMA-COMP-018
Compiled Command será distinto de Executed Statement.

## DB-SCHEMA-COMP-019
Una operación podrá producir múltiples comandos.

## DB-SCHEMA-COMP-020
Multi-command strategy deberá provenir del plan.

## DB-SCHEMA-COMP-021
Compiler no ocultará table rebuild strategy.

## DB-SCHEMA-COMP-022
Compiler no ocultará drop/recreate strategy.

## DB-SCHEMA-COMP-023
Syntactic lowering pertenecerá al compiler.

## DB-SCHEMA-COMP-024
Semantic emulation pertenecerá al planner.

## DB-SCHEMA-COMP-025
Capability validation será explícita.

## DB-SCHEMA-COMP-026
Unsupported capability no se omitirá silenciosamente.

## DB-SCHEMA-COMP-027
Platform context será explícito.

## DB-SCHEMA-COMP-028
Dialect context será explícito.

## DB-SCHEMA-COMP-029
Platform será distinto de Dialect.

## DB-SCHEMA-COMP-030
Compiler será driver-independent.

## DB-SCHEMA-COMP-031
Compiler no contendrá PDO.

## DB-SCHEMA-COMP-032
Compiler no contendrá DriverStatement.

## DB-SCHEMA-COMP-033
Compiler no dependerá de current connection global.

## DB-SCHEMA-COMP-034
MySQL tendrá compiler explícito.

## DB-SCHEMA-COMP-035
MariaDB tendrá compiler first-class.

## DB-SCHEMA-COMP-036
PostgreSQL tendrá compiler explícito.

## DB-SCHEMA-COMP-037
SQLite tendrá compiler explícito.

## DB-SCHEMA-COMP-038
Version checks no reemplazarán capability checks.

## DB-SCHEMA-COMP-039
Identifier rendering será tipado.

## DB-SCHEMA-COMP-040
Identifier quoting pertenecerá al renderer.

## DB-SCHEMA-COMP-041
Identifiers no usarán value binding.

## DB-SCHEMA-COMP-042
Qualified identifiers serán estructurados.

## DB-SCHEMA-COMP-043
Blind lowercase/uppercase estará prohibido.

## DB-SCHEMA-COMP-044
Reserved identifiers podrán renderizarse safely.

## DB-SCHEMA-COMP-045
Type rendering utilizará DatabaseType.

## DB-SCHEMA-COMP-046
Logical type será distinto de physical type.

## DB-SCHEMA-COMP-047
Type rendering preservará semantic requirements.

## DB-SCHEMA-COMP-048
Unknown type no tendrá fallback genérico inseguro.

## DB-SCHEMA-COMP-049
NoDefault será distinto de DefaultNull.

## DB-SCHEMA-COMP-050
LiteralDefault será distinto de ExpressionDefault.

## DB-SCHEMA-COMP-051
Schema expressions serán compiladas mediante typed renderer.

## DB-SCHEMA-COMP-052
SchemaExpression será distinta de QueryExpression.

## DB-SCHEMA-COMP-053
RawSchemaExpression será explícita.

## DB-SCHEMA-COMP-054
RAW no implicará safe.

## DB-SCHEMA-COMP-055
Raw platform mismatch fallará.

## DB-SCHEMA-COMP-056
CreateTable compilation preservará TableDefinition intent.

## DB-SCHEMA-COMP-057
Constraint inline/separate rendering no alterará semántica.

## DB-SCHEMA-COMP-058
Column compilation preservará type/nullability/default/generation.

## DB-SCHEMA-COMP-059
Compiler no inventará column defaults.

## DB-SCHEMA-COMP-060
Compiler no inventará CASCADE.

## DB-SCHEMA-COMP-061
Compiler no inventará indexes.

## DB-SCHEMA-COMP-062
CreateIndex compilation preservará ordered keys.

## DB-SCHEMA-COMP-063
Included columns no se convertirán en keys.

## DB-SCHEMA-COMP-064
Partial predicates se renderizarán solo con support válido.

## DB-SCHEMA-COMP-065
Expression indexes se renderizarán solo con support válido.

## DB-SCHEMA-COMP-066
UniqueConstraint será distinta de UniqueIndex durante compilation.

## DB-SCHEMA-COMP-067
PrimaryKey será distinta de backing index.

## DB-SCHEMA-COMP-068
ForeignKey mapping order será preservado.

## DB-SCHEMA-COMP-069
NO_ACTION será distinto de RESTRICT salvo equivalencia explícita.

## DB-SCHEMA-COMP-070
Deferrability unsupported no será ignorada.

## DB-SCHEMA-COMP-071
Validation-state unsupported no será ignorada.

## DB-SCHEMA-COMP-072
Sequences no serán emuladas ocultamente.

## DB-SCHEMA-COMP-073
View compilation tendrá boundary explícito con Query Compiler.

## DB-SCHEMA-COMP-074
No habrá circular dependency con Query Compiler.

## DB-SCHEMA-COMP-075
Low-level SQL rendering podrá compartirse.

## DB-SCHEMA-COMP-076
Catalog y namespace permanecerán conceptos distintos.

## DB-SCHEMA-COMP-077
Temporary table semantics serán capability-driven.

## DB-SCHEMA-COMP-078
Extension compilers serán registrados explícitamente.

## DB-SCHEMA-COMP-079
Compiler extension registry será frozen.

## DB-SCHEMA-COMP-080
Extension collision no utilizará last-wins.

## DB-SCHEMA-COMP-081
Compiler dispatch será determinista.

## DB-SCHEMA-COMP-082
Compilation será side-effect free salvo diagnostics locales.

## DB-SCHEMA-COMP-083
Compiler no realizará hidden DB I/O.

## DB-SCHEMA-COMP-084
Compiler no ejecutará server-version query.

## DB-SCHEMA-COMP-085
DDL parameter semantics serán explícitas.

## DB-SCHEMA-COMP-086
Multi-statement SQL string no será default.

## DB-SCHEMA-COMP-087
Command boundaries serán explícitos.

## DB-SCHEMA-COMP-088
Command ordering preservará plan ordering.

## DB-SCHEMA-COMP-089
Compiler no reordenará dependency graph arbitrariamente.

## DB-SCHEMA-COMP-090
Execution requirements serán declarativos.

## DB-SCHEMA-COMP-091
Transaction requirement será distinto de transaction execution.

## DB-SCHEMA-COMP-092
Compiler no hará BEGIN.

## DB-SCHEMA-COMP-093
Compiler no hará COMMIT.

## DB-SCHEMA-COMP-094
Compiler no hará ROLLBACK.

## DB-SCHEMA-COMP-095
DDL transaction behavior será platform-aware.

## DB-SCHEMA-COMP-096
Source mapping será preservable.

## DB-SCHEMA-COMP-097
Diagnostics serán estructurados.

## DB-SCHEMA-COMP-098
Silent semantic degradation estará prohibida.

## DB-SCHEMA-COMP-099
Compilation profile será explícito.

## DB-SCHEMA-COMP-100
Canonical compilation será determinista.

## DB-SCHEMA-COMP-101
Formatting será distinta de semantics.

## DB-SCHEMA-COMP-102
DDL IR podrá existir sin volverse public schema model.

## DB-SCHEMA-COMP-103
Query y Schema Compiler podrán compartir low-level renderers.

## DB-SCHEMA-COMP-104
Shared renderers no fusionarán high-level subsystems.

## DB-SCHEMA-COMP-105
Compilation cache será opcional.

## DB-SCHEMA-COMP-106
Cache key incluirá capability fingerprint.

## DB-SCHEMA-COMP-107
Cache key incluirá compiler version.

## DB-SCHEMA-COMP-108
Cache no contendrá live connections.

## DB-SCHEMA-COMP-109
Compilation fingerprint será versionado.

## DB-SCHEMA-COMP-110
SQL hash será distinto de structural identity.

## DB-SCHEMA-COMP-111
Compiler output versioning será explícito.

## DB-SCHEMA-COMP-112
Compilation failure será typed.

## DB-SCHEMA-COMP-113
Compilation será all-or-error por defecto.

## DB-SCHEMA-COMP-114
Partial compilation será marcada explícitamente.

## DB-SCHEMA-COMP-115
Partial compilation no será executable complete plan.

## DB-SCHEMA-COMP-116
Budgets serán explícitos.

## DB-SCHEMA-COMP-117
Budget exhaustion no truncará SQL.

## DB-SCHEMA-COMP-118
Cancelled compilation no será COMPLETE.

## DB-SCHEMA-COMP-119
Compilation buscará complejidad lineal.

## DB-SCHEMA-COMP-120
Mutable compiler sessions serán operation-scoped.

## DB-SCHEMA-COMP-121
Shared compiler services serán immutable.

## DB-SCHEMA-COMP-122
No habrá mutable current platform global.

## DB-SCHEMA-COMP-123
No habrá mutable current schema global.

## DB-SCHEMA-COMP-124
No habrá mutable current tenant global.

## DB-SCHEMA-COMP-125
FrankenPHP request isolation será obligatoria.

## DB-SCHEMA-COMP-126
RoadRunner worker isolation será obligatoria.

## DB-SCHEMA-COMP-127
OpenSwoole coroutine isolation será obligatoria.

## DB-SCHEMA-COMP-128
Multitenancy será integración externa.

## DB-SCHEMA-COMP-129
Compiler no resolverá tenant.

## DB-SCHEMA-COMP-130
Tenant-aware cache identity evitará cross-tenant collisions.

## DB-SCHEMA-COMP-131
Compiler será una security-sensitive boundary.

## DB-SCHEMA-COMP-132
Identifiers siempre pasarán por typed rendering.

## DB-SCHEMA-COMP-133
Literals siempre pasarán por typed rendering.

## DB-SCHEMA-COMP-134
Raw SQL no será API normal.

## DB-SCHEMA-COMP-135
Raw operations serán auditables.

## DB-SCHEMA-COMP-136
Extensions no ejecutarán SQL.

## DB-SCHEMA-COMP-137
Extensions no abrirán conexiones.

## DB-SCHEMA-COMP-138
Extensions no podrán bypass capabilities.

## DB-SCHEMA-COMP-139
Telemetry podrá registrar compiler metrics.

## DB-SCHEMA-COMP-140
Telemetry no expondrá full DDL por defecto.

## DB-SCHEMA-COMP-141
Golden SQL tests no reemplazarán structural tests.

## DB-SCHEMA-COMP-142
Cross-platform conformance tests serán obligatorios.

## DB-SCHEMA-COMP-143
Compiler será testeable sin database real.

## DB-SCHEMA-COMP-144
Compile result será deterministic con mismo input/context/version.

## DB-SCHEMA-COMP-145
Compiled command será immutable.

## DB-SCHEMA-COMP-146
Compiled plan será immutable.

## DB-SCHEMA-COMP-147
Compiler representará, no ejecutará.

## DB-SCHEMA-COMP-148
Planner decidirá estrategia; compiler decidirá sintaxis.

## DB-SCHEMA-COMP-149
Executor decidirá ejecución; compiler no observará outcome.

## DB-SCHEMA-COMP-150
Schema Compiler será la única capa autorizada para traducir operaciones estructurales planificadas a DDL de plataforma dentro de VoltStack.

---

# 202. Anti-patterns

## 202.1 SQL en Schema Builder

Incorrecto:

```php
$table->sql(
    'CREATE TABLE users (...)'
);
```

como flujo normal.

---

## 202.2 Diff → SQL

Incorrecto:

```php
$ddl = $schemaCompiler->compile(
    $schemaDiffer->diff($current, $target)
);
```

sin planning intermedio.

---

## 202.3 Hidden table rebuild

Incorrecto:

```php
if (!$platform->supportsAlterColumn()) {
    // compiler secretly rebuilds entire table
}
```

---

## 202.4 Version conditionals dispersos

Incorrecto:

```php
if ($mysqlVersion >= '8.0.13') {
    ...
}
```

en múltiples renderers.

---

## 202.5 Identifier concatenation

Incorrecto:

```php
$sql = 'ALTER TABLE ' . $tableName;
```

con nombre arbitrario sin typed renderer.

---

## 202.6 Unique constraint como index

Incorrecto:

```php
if ($constraint->isUnique()) {
    return "CREATE UNIQUE INDEX ...";
}
```

como regla universal.

---

## 202.7 DEFAULT NULL por ausencia

Incorrecto:

```php
$default = $column->default() ?? 'NULL';
```

---

## 202.8 Silent unsupported feature

Incorrecto:

```php
if (!$supportsDeferrable) {
    // just omit it
}
```

---

## 202.9 Compiler abre conexión

Incorrecto:

```php
$version = $connection->query('SELECT VERSION()');
```

---

## 202.10 Compiler ejecuta

Incorrecto:

```php
$connection->exec(
    $compiler->compile($operation)
);
```

dentro del propio compiler.

---

## 202.11 SQL blob de múltiples operaciones

Incorrecto por defecto:

```text
CREATE ...;
ALTER ...;
DROP ...;
```

como un solo `CompiledSchemaCommand`.

---

## 202.12 Current tenant singleton

Incorrecto:

```php
$table = Tenant::current()->prefix() . $table;
```

dentro del compiler.

---

# 203. Ejemplo: CREATE TABLE portable

Input:

```text
CreateTableOperation
└── TableDefinition(users)
    ├── id
    │   ├── BigIntegerType
    │   ├── NOT_NULL
    │   └── IdentityGeneration
    │
    ├── email
    │   ├── StringType(320)
    │   └── NOT_NULL
    │
    ├── PrimaryKey(id)
    └── Unique(email)
```

PostgreSQL compiler podría producir conceptualmente:

```sql
CREATE TABLE "users" (
    "id" BIGINT GENERATED BY DEFAULT AS IDENTITY NOT NULL,
    "email" VARCHAR(320) NOT NULL,
    CONSTRAINT "pk_users" PRIMARY KEY ("id"),
    CONSTRAINT "uq_users_email" UNIQUE ("email")
)
```

MySQL podría producir otra representación equivalente según capabilities.

La semántica estructural no cambia.

---

# 204. Ejemplo: índice parcial

Input:

```text
CreateIndexOperation
├── index = idx_users_active_email
├── key = email
└── predicate = deleted_at IS NULL
```

PostgreSQL:

```sql
CREATE INDEX "idx_users_active_email"
ON "users" ("email")
WHERE "deleted_at" IS NULL
```

Si la plataforma no tiene:

```text
PARTIAL_INDEX
```

el compiler deberá fallar salvo que el plan ya represente una estrategia alternativa.

---

# 205. Ejemplo: FK

Input:

```text
AddForeignKeyOperation
└── fk_orders_customer
    ├── customer_id
    ├── customers.id
    └── ON DELETE CASCADE
```

Salida conceptual:

```sql
ALTER TABLE "orders"
ADD CONSTRAINT "fk_orders_customer"
FOREIGN KEY ("customer_id")
REFERENCES "customers" ("id")
ON DELETE CASCADE
```

El compiler no decidió:

```text
crear índice
```

ni:

```text
validar datos
```

ni:

```text
deshabilitar FK temporalmente
```

---

# 206. Ejemplo: SQLite table rebuild

Schema AST:

```text
AlterColumn(users.email)
```

Planner determina:

```text
RebuildTablePlan
├── CreateTemporaryTable
├── CopyData
├── DropOriginalTable
├── RenameTemporaryTable
└── RecreateIndexesAndConstraints
```

El SQLite compiler produce comandos para ese plan.

Por tanto:

```text
SQLite limitation
```

no convierte al compiler en planner.

---

# 207. Ejemplo: transacción

Compiled command:

```text
CreateIndexConcurrentlyCommand
├── SQL
│   └── CREATE INDEX CONCURRENTLY ...
│
└── requirements
    └── FORBIDDEN_IN_TRANSACTION
```

El Execution Engine podrá entonces validar su contexto.

---

# 208. Ejemplo: default literal vs expresión

Definitions:

```text
A:
default = LiteralDefault("CURRENT_TIMESTAMP")
```

```text
B:
default = ExpressionDefault(CurrentTimestamp)
```

Compiler debe producir conceptualmente:

```text
A → DEFAULT 'CURRENT_TIMESTAMP'
B → DEFAULT CURRENT_TIMESTAMP
```

según dialecto.

No colapsar ambos.

---

# 209. Ejemplo: type mapping

Logical:

```text
BooleanType
```

Possible rendering:

```text
PostgreSQL:
BOOLEAN
```

```text
MySQL:
BOOLEAN / appropriate physical representation
```

```text
SQLite:
INTEGER or platform-defined boolean representation
```

según las reglas oficiales de `Platform` y `Dialect`.

---

# 210. Ejemplo de error

Input:

```text
CreateIndexOperation
└── INCLUDE(name)
```

Capabilities:

```text
INCLUDED_COLUMNS = false
```

Resultado:

```text
UnsupportedSchemaCapabilityException
├── operation = CreateIndex
├── required = INCLUDED_COLUMNS
├── platform = selected platform
└── source operation ID
```

No SQL parcial.

---

# 211. Fórmula maestra

```text
Schema Compiler System
=
Planned Structural Operations
+
Platform Resolution
+
Dialect Rendering
+
Capability Verification
+
Identifier Rendering
+
Type Rendering
+
Expression Rendering
+
Constraint Rendering
+
Index Rendering
+
DDL Intermediate Representation
+
Command Assembly
+
Execution Requirements
+
Source Mapping
+
Deterministic Formatting
+
Fingerprinting
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

# 212. Fórmula de compilación

Para operación `O`, plataforma `P`, dialecto `D`, capabilities `C`:

```text
Compile(O, P, D, C)
=
CompiledCommandSet
```

si:

```text
Valid(O)
∧
C ⊇ Requirements(O)
```

---

# 213. Fórmula de corrección

```text
CorrectCompilation
=
Meaning(CompiledRepresentation)
≈
Meaning(PlannedOperation)
```

dentro de las garantías de la plataforma.

---

# 214. Boundary formula

```text
Schema AST
=
Requested Intent
```

```text
Schema Plan
=
Chosen Strategy
```

```text
Schema Compiler
=
Platform Representation
```

```text
Execution Engine
=
Runtime Execution
```

---

# 215. Regla de emulación

```text
Syntax substitution
→ Compiler
```

```text
Structural strategy substitution
→ Planner
```

Ejemplos:

```text
BooleanType → vendor physical type
=
compiler
```

```text
ALTER COLUMN → rebuild entire table
=
planner
```

---

# 216. Security formula

```text
SafeSchemaCompilation
=
TypedInputs
∧
ValidatedIdentifiers
∧
CapabilityChecks
∧
TrustedRawBoundaries
∧
DeterministicRenderers
∧
ExtensionIsolation
∧
OutputBudgets
```

---

# 217. Arquitectura final

```text
                    Schema AST
                        │
                        ▼
                  Schema Planner
                        │
                        ▼
                Schema Execution Plan
                        │
                        ▼
                Schema Compiler
                        │
        ┌───────────────┼─────────────────┐
        │               │                 │
        ▼               ▼                 ▼
 Identifier          Type /            Platform
 Rendering         Expression           Rules
                  Rendering
        │               │                 │
        └───────────────┼─────────────────┘
                        │
                        ▼
                    DDL IR
                        │
                        ▼
                  SQL Renderer
                        │
                        ▼
               Compiled Commands
                        │
                        ├── SQL
                        ├── execution requirements
                        ├── source map
                        ├── metadata
                        └── diagnostics
                        │
                        ▼
                 Execution Engine
```

---

# 218. Regla arquitectónica final

> **El Schema Compiler de VoltStack debe ser un traductor determinista entre operaciones estructurales ya planificadas y el DDL de una plataforma; puede cambiar la forma sintáctica, pero nunca debe inventar, omitir ni reinterpretar silenciosamente la intención estructural.**

La regla puede resumirse:

```text
Plan decides what strategy.
Compiler decides how that strategy is written.
Executor decides how it is run.
```

---

# 219. Resultado arquitectónico

Con este sistema VoltStack obtiene una frontera clara:

```text
Platform-neutral architecture
           │
           ▼
    Schema Compiler
           │
           ▼
Platform-specific DDL
```

capaz de soportar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin insertar condiciones de proveedor en:

```text
Schema Builder
Schema Model
Schema Diff
ORM
Query Builder
```

y preservando:

```text
correctness
determinism
portability
platform specialization
traceability
security
persistent-runtime safety
```

---

# 220. Siguiente documento

```text
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md
```

El siguiente documento formalizará cómo VoltStack determina si una definición, diferencia, operación o plan de schema puede representarse correctamente en una plataforma específica:

```text
Schema Definition / Operation
          +
Database Platform
          +
Capability Snapshot
          +
Compatibility Profile
          ↓
Compatibility Analysis
          ↓
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

Deberá diferenciar especialmente:

```text
Supported
≠
Compilable By Accident

Unsupported
≠
Invalid

Capability
≠
Database Version

Portability
≠
Feature Intersection Only

Emulation
≠
Silent Degradation

Physical Difference
≠
Semantic Incompatibility
```

y cerrará el bloque de Schema antes de entrar a:

```text
101_DATABASE_MIGRATION_ARCHITECTURE.md
```