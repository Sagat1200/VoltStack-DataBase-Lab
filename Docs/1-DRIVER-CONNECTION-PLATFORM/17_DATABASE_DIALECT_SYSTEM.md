# 17_DATABASE_DIALECT_SYSTEM.md

# VoltStack Quantum Database
## Database Dialect System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 17 — Database Dialect System  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure / SQL Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del sistema de dialectos SQL de:

```text
VoltStack/Quantum/Database
```

`Database Dialect System` será responsable de representar las diferencias de **expresión sintáctica SQL** entre motores y familias de bases de datos sin contaminar:

- Query Builder;
- Query AST;
- Semantic Query Engine;
- ORM;
- Schema Model;
- Migration Model;
- Connection;
- Driver;
- Application code;

con condicionales específicos de proveedor.

La separación fundamental será:

```text
Driver
=
cómo VoltStack se comunica con el cliente nativo

Dialect
=
cómo una operación semántica se expresa en SQL

Platform
=
qué significa y qué soporta el motor de base de datos
```

Por tanto:

```text
Driver ≠ Dialect ≠ Platform ≠ Connection
```

---

# 2. Problema arquitectónico

Los motores SQL comparten gran parte del lenguaje relacional, pero difieren en múltiples áreas.

Ejemplo conceptual:

```text
LIMIT
RETURNING
UPSERT
LOCKING
JSON
IDENTIFIER QUOTING
DDL
FUNCTIONS
OPERATORS
GENERATED COLUMNS
```

Una arquitectura ingenua termina generando código como:

```php
if ($driver === 'mysql') {
    // ...
} elseif ($driver === 'pgsql') {
    // ...
} elseif ($driver === 'sqlite') {
    // ...
}
```

distribuido por:

```text
Query Builder
Schema Builder
Migration
ORM
Repositories
Persistence
```

Esto produce:

```text
vendor coupling
+
duplicated SQL logic
+
hard-to-test behavior
+
poor extensibility
```

VoltStack evitará este diseño.

---

# 3. Principio fundamental

Las capas superiores expresarán:

```text
WHAT
```

y el Dialect ayudará al Compiler a decidir:

```text
HOW TO EXPRESS IT IN SQL
```

Ejemplo:

```text
Semantic Intent

Insert row
on conflict:
    update selected columns
```

no deberá convertirse prematuramente en:

```text
ON DUPLICATE KEY UPDATE
```

ni:

```text
ON CONFLICT (...) DO UPDATE
```

El AST conserva intención semántica.

La traducción específica ocurre posteriormente.

---

# 4. Dialect no es un SQL Compiler

Ésta será una distinción importante.

```text
Dialect
≠
SQL Compiler
```

El Dialect proporciona:

```text
syntax rules
lexical conventions
quoting rules
placeholder conventions
operator definitions
function mappings
grammar fragments
vendor syntax strategies
```

El Compiler:

```text
AST / Plan
      │
      ▼
SQL Compiler
      │
      ├── Dialect
      ├── Platform
      └── Capabilities
      │
      ▼
CompiledQuery
```

---

# 5. Responsabilidades del Dialect

El Dialect podrá definir:

```text
identifier quoting
identifier escaping
parameter placeholder syntax
reserved keywords
literal syntax
operator syntax
function syntax
pagination syntax
locking syntax
CTE syntax
window syntax
RETURNING syntax
UPSERT syntax
JSON syntax
DDL syntax fragments
vendor-specific SQL tokens
```

---

# 6. No responsabilidades

El Dialect no deberá:

```text
open connections
execute queries
manage transactions
manage connection pools
hydrate entities
track ORM state
manage IdentityMap
manage UnitOfWork
route read/write connections
perform retries
select tenants
manage application authorization
```

---

# 7. Arquitectura general

```text
Query / Schema Semantic Model
            │
            ▼
         Compiler
            │
      ┌─────┼──────────┐
      ▼     ▼          ▼
   Dialect Platform Capabilities
      │
      ▼
SQL Syntax
      │
      ▼
Compiled SQL
```

---

# 8. Posición dentro de VoltStack Database

```text
Application
    │
    ▼
Query Builder / ORM / Schema
    │
    ▼
Semantic Model
    │
    ▼
Optimizer
    │
    ▼
Planner
    │
    ▼
Compiler
    │
    ├────────► Dialect
    │
    ├────────► Platform
    │
    └────────► Capabilities
    │
    ▼
CompiledQuery
    │
    ▼
Executor
    │
    ▼
Connection
    │
    ▼
Driver
```

---

# 9. Core rule

El Dialect estará:

```text
above Driver
below Compiler policy
```

conceptualmente.

No será responsable del transporte.

---

# 10. Dialect Interface

Contrato conceptual:

```php
interface DialectInterface
{
    public function descriptor(): DialectDescriptor;

    public function identifiers(): IdentifierDialectInterface;

    public function parameters(): ParameterDialectInterface;

    public function operators(): OperatorDialectInterface;

    public function functions(): FunctionDialectInterface;

    public function syntax(): SyntaxDialectInterface;
}
```

La interfaz final podrá dividirse aún más.

---

# 11. Evitar God Dialect

No se recomienda:

```php
interface DialectInterface
{
    public function compileSelect(...);
    public function compileInsert(...);
    public function compileUpdate(...);
    public function compileDelete(...);
    public function compileSchema(...);
    public function compileJson(...);
    public function compileEverything(...);
}
```

Esto convertiría Dialect en un segundo Compiler.

---

# 12. Dialect compuesto

Preferencia:

```text
Dialect
│
├── IdentifierDialect
├── ParameterDialect
├── LiteralDialect
├── OperatorDialect
├── FunctionDialect
├── PaginationDialect
├── LockingDialect
├── ReturningDialect
├── UpsertDialect
├── JsonDialect
├── DdlDialect
└── VendorSyntaxExtensions
```

---

# 13. DialectDescriptor

Cada dialecto tendrá un descriptor inmutable.

Ejemplo:

```php
final readonly class DialectDescriptor
{
    public function __construct(
        public DialectId $id,
        public string $name,
        public DialectFamily $family,
        public DialectVersionRange $versions,
    ) {}
}
```

---

# 14. DialectId

IDs conceptuales:

```text
mysql
mariadb
postgresql
sqlite
```

Podrán existir dialectos adicionales:

```text
cockroachdb
tidb
yugabytedb
```

sin alterar el Query AST central.

---

# 15. Dialect family

Podrá distinguirse:

```text
SQL_STANDARD
MYSQL_FAMILY
POSTGRESQL_FAMILY
SQLITE_FAMILY
```

pero no deberá utilizarse como sustituto indiscriminado del Capability System.

---

# 16. Dialect version

La sintaxis disponible puede depender de la versión.

Ejemplo conceptual:

```text
Dialect
+
Server Version
=
Effective Dialect Syntax
```

---

# 17. Version no es Driver Version

Deben distinguirse:

```text
VoltStack Driver Version
Native Client Version
Database Server Version
Dialect Version
Platform Version
```

---

# 18. Dialect Registry

Existirá un registro especializado:

```text
DialectRegistry
```

responsable de:

```text
registration
duplicate detection
lookup
extension integration
freeze
```

---

# 19. No universal registry

No deberá existir:

```text
DatabaseRegistry
```

conteniendo drivers, dialectos, tipos, compiladores, plataformas y extensiones en un único objeto mutable.

---

# 20. Dialect Registry lifecycle

```text
Bootstrap
   │
   ▼
Register official dialects
   │
   ▼
Register extension dialects
   │
   ▼
Validate
   │
   ▼
Freeze
   │
   ▼
Runtime immutable lookup
```

---

# 21. DialectResolver

Componente:

```text
DialectResolver
```

resolverá el dialecto efectivo a partir de información estructurada.

---

# 22. Resolution inputs

Podrán incluir:

```text
ConnectionDefinition
DriverDescriptor
PlatformDescriptor
ServerVersion
Configuration
Extension metadata
```

---

# 23. No eager connection requirement

Resolver:

```text
DB::connection()
```

no deberá necesariamente abrir una conexión para conocer el dialecto básico.

---

# 24. Two-phase resolution

Podrá utilizarse:

```text
Configured Dialect
       │
       ▼
Preliminary Dialect
       │
       ▼
Physical Connection
       │
       ▼
Server Metadata
       │
       ▼
Resolved Dialect
```

---

# 25. Dialect refinement

El dialecto preliminar podrá refinarse cuando se descubra:

```text
server version
compatibility mode
vendor variant
```

---

# 26. Dialect mismatch

Si configuración y servidor son incompatibles:

```text
DialectResolutionException
```

deberá fallar claramente.

---

# 27. Driver/Dialect independence

Ejemplo:

```text
PDO MySQL Driver
```

puede transportar:

```text
MySQL
MariaDB
```

pero:

```text
MySqlDialect
≠
MariaDbDialect
```

cuando sus sintaxis diverjan.

---

# 28. Dialect/Platform independence

También:

```text
PostgreSqlDialect
```

describe sintaxis.

Mientras:

```text
PostgreSqlPlatform
```

describe:

```text
server semantics
features
types
transaction behavior
schema behavior
capabilities
```

---

# 29. Identifier system

El Dialect será responsable de expresar identificadores correctamente.

Ejemplo:

```text
users
user_accounts
order
schema.users
```

---

# 30. Identifier object

Los identificadores deberán modelarse estructuralmente.

Ejemplo:

```php
final readonly class Identifier
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 31. QualifiedIdentifier

Ejemplo:

```text
schema.table.column
```

podrá representarse mediante:

```text
QualifiedIdentifier
```

en vez de concatenación manual.

---

# 32. Identifier quoting

Ejemplos:

MySQL/MariaDB:

```sql
`users`
```

PostgreSQL/SQLite:

```sql
"users"
```

---

# 33. Quoting rule

Nunca:

```php
$sql = '"' . $name . '"';
```

en capas superiores.

---

# 34. IdentifierDialect

Contrato conceptual:

```php
interface IdentifierDialectInterface
{
    public function quote(Identifier $identifier): string;

    public function quoteQualified(
        QualifiedIdentifier $identifier
    ): string;
}
```

---

# 35. Identifier quoting ≠ parameter binding

Muy importante:

```text
values
→ parameters

identifiers
→ validated + quoted identifiers
```

No puede hacerse:

```sql
SELECT * FROM ?
```

para parametrizar normalmente un nombre de tabla.

---

# 36. Identifier validation

El sistema deberá validar identificadores antes de quoting.

---

# 37. Reserved words

Cada Dialect podrá proporcionar:

```text
ReservedWordSet
```

---

# 38. Reserved word policy

El Compiler podrá decidir quoting obligatorio cuando:

```text
identifier is reserved
```

---

# 39. Quote policy

Podrán existir políticas:

```text
AUTO
ALWAYS
MINIMAL
```

---

# 40. Default quote policy

Se recomienda:

```text
AUTO
```

con comportamiento determinista.

---

# 41. Case sensitivity

El Dialect deberá describir reglas sintácticas relacionadas con:

```text
identifier case folding
quoted identifier preservation
```

mientras la semántica efectiva podrá requerir colaboración del Platform.

---

# 42. Parameter placeholders

El Dialect podrá describir:

```text
?
:name
$1
```

según Compiler/Driver strategy.

---

# 43. Placeholder responsibility

La arquitectura deberá evitar confundir:

```text
native client placeholder
```

con:

```text
semantic query parameter
```

---

# 44. Parameter pipeline

```text
QueryParameter
      │
      ▼
Compiler Parameter Allocator
      │
      ▼
Dialect Placeholder Strategy
      │
      ▼
Compiled Parameter
      │
      ▼
Driver Binding
```

---

# 45. ParameterDialect

Ejemplo conceptual:

```php
interface ParameterDialectInterface
{
    public function placeholder(
        ParameterPosition $position
    ): string;
}
```

---

# 46. Named vs positional parameters

El Query Model podrá usar identificadores semánticos independientemente de cómo el SQL final los represente.

---

# 47. Literal syntax

Los valores dinámicos deberán usar binding.

Sin embargo, ciertos literales estructurales pueden requerir sintaxis del dialecto.

Ejemplos:

```text
TRUE
FALSE
NULL
CURRENT_TIMESTAMP
INTERVAL
```

---

# 48. LiteralDialect

Podrá definir:

```text
boolean literal
null literal
special temporal literals
binary literal syntax
```

---

# 49. No arbitrary value interpolation

Dialect nunca será excusa para:

```php
$sql .= "'" . $userInput . "'";
```

---

# 50. Operator system

Los operadores podrán modelarse mediante IDs semánticos.

Ejemplos:

```text
EQUAL
NOT_EQUAL
LIKE
ILIKE
REGEX
JSON_CONTAINS
ARRAY_CONTAINS
```

---

# 51. Operator mapping

```text
Semantic Operator
      │
      ▼
Dialect Operator Resolver
      │
      ▼
SQL Operator
```

---

# 52. Standard operators

Operadores universales deberán permanecer simples.

Ejemplo:

```text
EQUAL → =
GREATER_THAN → >
```

---

# 53. Vendor-specific operators

Ejemplo conceptual:

```text
CASE_INSENSITIVE_LIKE
```

podrá resolverse a una operación nativa o emulada.

---

# 54. Operator support

El Dialect describe sintaxis.

El Capability System determina si la operación:

```text
NATIVE
EMULATED
PARTIAL
UNSUPPORTED
```

---

# 55. Function system

Las funciones SQL tampoco deberán escribirse indiscriminadamente como strings.

---

# 56. Semantic functions

Podrán existir IDs como:

```text
CURRENT_DATE
CURRENT_TIMESTAMP
LOWER
UPPER
LENGTH
CONCAT
COALESCE
JSON_EXTRACT
DATE_DIFF
```

---

# 57. FunctionDialect

Contrato conceptual:

```php
interface FunctionDialectInterface
{
    public function resolve(
        FunctionId $function
    ): FunctionSyntax;
}
```

---

# 58. FunctionSyntax

Podrá representar:

```text
name
argument order
special grammar
variadic behavior
keyword/function form
```

---

# 59. Function is not always NAME(...)

Ejemplos conceptuales:

```text
CURRENT_TIMESTAMP
CAST(x AS type)
EXTRACT(part FROM x)
```

Por eso un simple:

```text
array<string,string>
```

no será suficiente para todas las funciones.

---

# 60. Function compiler strategy

Funciones complejas podrán usar:

```text
FunctionCompilerInterface
```

registrado por Dialect.

---

# 61. No arbitrary function execution

El registro de funciones deberá controlar qué funciones pueden construirse desde APIs estructuradas.

---

# 62. Pagination

El Query AST representará:

```text
limit
offset
```

semánticamente.

---

# 63. PaginationDialect

Será responsable de proporcionar la expresión SQL apropiada cuando existan diferencias.

---

# 64. Query AST portability

Nunca:

```text
LimitNode('LIMIT 10')
```

Preferir:

```text
LimitNode(10)
```

---

# 65. Locking syntax

El AST podrá representar:

```text
FOR_UPDATE
FOR_SHARE
NOWAIT
SKIP_LOCKED
```

como intención.

---

# 66. LockingDialect

Mapeará la intención a sintaxis.

---

# 67. Locking semantics

La semántica real y disponibilidad dependerán del Platform/Capabilities.

---

# 68. RETURNING

El Query Model podrá representar:

```text
return generated values
```

sin asumir sintaxis.

---

# 69. ReturningDialect

Podrá producir:

```sql
RETURNING id
```

cuando sea apropiado.

---

# 70. Returning capability

Antes de compilar:

```text
supportsReturning()
```

deberá consultarse mediante Capability System.

---

# 71. Emulated returning

Si una plataforma no soporta `RETURNING`, el Planner podrá seleccionar otra estrategia.

Ejemplo:

```text
INSERT
+
generated key retrieval
```

---

# 72. Dialect does not choose semantic fallback

Esto pertenece principalmente a:

```text
Planner
+
Capabilities
+
Platform
```

El Dialect expresa la estrategia elegida.

---

# 73. UPSERT

El Query AST representará algo como:

```text
Insert
OnConflict
ConflictTarget
UpdateAssignments
```

---

# 74. MySQL-style syntax

Podría compilarse conceptualmente como:

```sql
INSERT ...
ON DUPLICATE KEY UPDATE ...
```

---

# 75. PostgreSQL-style syntax

Podría compilarse como:

```sql
INSERT ...
ON CONFLICT (...) DO UPDATE ...
```

---

# 76. SQLite syntax

Dependiendo de versión/capabilities:

```sql
INSERT ...
ON CONFLICT (...) DO UPDATE ...
```

---

# 77. Semantic difference warning

Sintaxis parecida no implica semántica idéntica.

Por tanto:

```text
Dialect
+
Platform
+
Capabilities
```

deberán colaborar.

---

# 78. CTE syntax

El Query Model podrá representar:

```text
WITH
WITH RECURSIVE
```

sin guardar SQL específico.

---

# 79. CTEDialect

Responsable de sintaxis.

---

# 80. CTE capabilities

Platform indicará:

```text
supportsCTE
supportsRecursiveCTE
supportsDataModifyingCTE
```

según aplique.

---

# 81. Window functions

AST:

```text
WindowDefinition
PartitionBy
OrderBy
Frame
```

---

# 82. WindowDialect

Expresará:

```text
OVER (...)
PARTITION BY
ROWS
RANGE
GROUPS
```

según soporte.

---

# 83. Window capability

No toda versión de cada motor soportará las mismas funciones o frames.

---

# 84. JSON

JSON será una de las áreas con mayor divergencia.

---

# 85. Semantic JSON operations

El AST podrá representar:

```text
JSON_EXTRACT
JSON_CONTAINS
JSON_EXISTS
JSON_SET
JSON_REMOVE
JSON_PATH
```

---

# 86. JSONDialect

Traducirá operaciones soportadas a sintaxis concreta.

---

# 87. JSON portability

VoltStack no prometerá que toda operación JSON avanzada sea portable entre todos los motores.

---

# 88. Capability-first JSON

El usuario podrá consultar:

```php
$connection->capabilities()
    ->supports(DatabaseCapability::JSON_PATH);
```

conceptualmente.

---

# 89. String concatenation

Ejemplo clásico de diferencia:

```text
CONCAT(a, b)
```

vs:

```text
a || b
```

según motor/semántica.

---

# 90. Semantic concat

AST:

```text
ConcatExpression
```

no:

```text
RawExpression("a || b")
```

para la API portable.

---

# 91. Boolean expressions

Dialect deberá conocer representación sintáctica de:

```text
true
false
```

cuando sea necesario.

---

# 92. Null ordering

El AST podrá expresar:

```text
NULLS_FIRST
NULLS_LAST
DEFAULT
```

---

# 93. Null ordering capability

Algunas plataformas pueden soportar sintaxis directa; otras requerir emulación.

---

# 94. Emulation responsibility

```text
Planner
→ chooses emulation

Dialect
→ expresses selected SQL
```

---

# 95. Random ordering

Operación semántica:

```text
RANDOM
```

puede mapearse a diferentes funciones.

---

# 96. Date/time syntax

Debe existir una abstracción suficiente para:

```text
date arithmetic
intervals
extract
truncation
current date/time
```

sin intentar ocultar todas las diferencias semánticas.

---

# 97. Regex syntax

Podrá variar entre:

```text
operators
functions
different regex engines
```

Por tanto requiere Dialect + Platform.

---

# 98. Full-text search

No deberá forzarse como una única sintaxis SQL universal.

Podrá implementarse mediante:

```text
extension capability
+
semantic operation
+
dialect compiler
```

---

# 99. DDL dialect

El Schema Compiler utilizará una sección especializada:

```text
DdlDialect
```

---

# 100. DDL responsibilities

Podrá expresar:

```text
CREATE TABLE
ALTER TABLE
DROP TABLE
CREATE INDEX
DROP INDEX
ADD CONSTRAINT
DROP CONSTRAINT
```

---

# 101. DDL semantics

La posibilidad real de ejecutar ciertas operaciones pertenece al Platform.

Ejemplo:

```text
supportsDropColumn
```

---

# 102. DDL syntax ≠ schema capability

Por tanto:

```text
DdlDialect
≠
SchemaCapabilitySet
```

---

# 103. Auto increment

El Schema Model representará:

```text
generated identity
```

semánticamente.

No directamente:

```text
AUTO_INCREMENT
SERIAL
AUTOINCREMENT
```

---

# 104. Identity compilation

```text
Schema Model
     │
     ▼
Schema Planner
     │
     ▼
Identity Strategy
     │
     ▼
Dialect
     │
     ▼
Vendor SQL
```

---

# 105. Generated columns

Igual principio:

```text
GeneratedColumnDefinition
```

en Schema Model.

El Dialect sólo expresa la sintaxis elegida.

---

# 106. Constraint syntax

Dialect deberá ayudar a expresar:

```text
PRIMARY KEY
UNIQUE
CHECK
FOREIGN KEY
```

---

# 107. Constraint semantics

Platform deberá declarar:

```text
support
enforcement behavior
deferrability
validation semantics
```

---

# 108. Deferrable constraints

No se asumirán universales.

---

# 109. Index syntax

Puede variar en:

```text
partial indexes
expression indexes
included columns
index methods
operator classes
full-text indexes
```

---

# 110. Index semantic model

El Schema Model deberá conservar intención estructural.

---

# 111. Vendor extensions

Las características no portables podrán existir mediante extensiones explícitas.

Ejemplo:

```text
PostgreSqlIndexMethod
PostgreSqlOperatorClass
MySqlIndexPrefix
```

---

# 112. Portable core + explicit extension

Regla:

```text
portable semantics
+
explicit vendor extensions
```

en lugar de:

```text
lowest common denominator only
```

---

# 113. Dialect extension system

Un dialecto podrá registrar extensiones como:

```text
operators
functions
expressions
DDL fragments
query clauses
syntax compilers
```

---

# 114. Extension contract

Ejemplo conceptual:

```php
interface DialectExtensionInterface
{
    public function descriptor(): DialectExtensionDescriptor;

    public function register(
        DialectExtensionRegistry $registry
    ): void;
}
```

---

# 115. Registry freeze

Las extensiones se registrarán durante bootstrap.

Después:

```text
DialectExtensionRegistry
→ frozen
```

---

# 116. No runtime mutation

No se permitirá que una petición agregue funciones SQL globales al dialecto.

---

# 117. Extension collision

Si dos extensiones registran el mismo:

```text
FunctionId
OperatorId
SyntaxId
```

deberá existir:

```text
explicit override
```

o error.

---

# 118. No last wins

Prohibido:

```text
last registered silently wins
```

---

# 119. Extension ordering

Se utilizará:

```text
dependency graph
```

cuando sea necesario.

---

# 120. Compiler extension vs Dialect extension

No toda extensión del Compiler pertenece al Dialect.

Ejemplo:

```text
new AST node
```

puede necesitar:

```text
AST extension
Semantic extension
Planner extension
Compiler extension
Dialect syntax extension
```

---

# 121. Dialect cannot invent AST semantics

Un Dialect no deberá reinterpretar arbitrariamente el significado de nodos estándar.

---

# 122. Compiler context

El Compiler podrá recibir:

```php
final readonly class SqlCompilationContext
{
    public function __construct(
        public DialectInterface $dialect,
        public DatabasePlatformInterface $platform,
        public CapabilitySet $capabilities,
        public ParameterAllocator $parameters,
    ) {}
}
```

conceptualmente.

---

# 123. No container lookup

Dialect no deberá hacer:

```php
app(DatabaseManager::class)
```

---

# 124. No global config

Dialect no deberá leer:

```php
config(...)
getenv(...)
$_ENV
```

---

# 125. Immutable dialect

Preferencia:

```php
final readonly class PostgreSqlDialect
```

cuando sea posible.

---

# 126. Lifetime

Dialectos compilados/validados podrán vivir:

```text
application scope
```

porque deberán ser:

```text
immutable
stateless
reentrant
concurrency-safe
```

---

# 127. No current connection state

Un Dialect nunca deberá almacenar:

```text
current connection
current transaction
current tenant
current query
current request
current schema
```

---

# 128. Persistent runtime safety

Por ser inmutable/stateless:

```text
MySqlDialect
PostgreSqlDialect
SQLiteDialect
```

podrán reutilizarse entre miles de requests.

---

# 129. FrankenPHP

Modelo:

```text
Worker
│
├── MySqlDialect singleton-safe
├── PostgreSqlDialect singleton-safe
└── SQLiteDialect singleton-safe

Request A
Request B
Request C
```

sin estado mutable compartido.

---

# 130. RoadRunner/OpenSwoole

Misma regla.

---

# 131. Concurrency

Dialect deberá poder ser utilizado simultáneamente desde múltiples:

```text
fibers
coroutines
requests
jobs
```

---

# 132. Dialect cache

Podrán cachearse:

```text
reserved word sets
function descriptors
operator descriptors
compiled syntax descriptors
```

si son inmutables.

---

# 133. No query-specific cache inside Dialect

No deberá almacenar:

```text
last query
last identifier
current parameter position
```

---

# 134. ParameterAllocator ownership

El contador de parámetros pertenece a:

```text
CompilationContext
```

no al Dialect singleton.

---

# 135. Dialect capability interaction

Ejemplo:

```text
Dialect knows:
RETURNING syntax = "RETURNING ..."

Capability knows:
RETURNING supported = yes/no/partial
```

---

# 136. Platform interaction

Ejemplo:

```text
Dialect:
syntax for lock clause

Platform:
locking behavior and supported lock modes
```

---

# 137. Driver interaction

Idealmente Dialect no necesitará Driver directamente.

---

# 138. Driver-specific placeholder caveat

Si la estrategia de placeholders depende del cliente nativo, deberá modelarse mediante un descriptor neutral proporcionado al contexto de compilación.

No mediante dependencia:

```text
Dialect → PDO
```

---

# 139. Dialect + binding profile

Podrá existir:

```text
SqlBindingProfile
```

resuelto por:

```text
Dialect
+
Driver capabilities
```

antes de compilar.

---

# 140. Native placeholder strategy

Ejemplo:

```text
QUESTION_MARK
NAMED
NUMBERED
```

---

# 141. CompiledQuery portability

`CompiledQuery` ya es específico de:

```text
Dialect
Platform
Capability Snapshot
Binding Profile
```

---

# 142. Compiled query cache key

Deberá incluir fingerprints relevantes.

Ejemplo:

```text
QueryFingerprint
+
DialectFingerprint
+
PlatformCapabilityFingerprint
+
CompilerVersion
```

---

# 143. DialectFingerprint

Podrá incluir:

```text
dialect ID
dialect version
registered syntax extensions
configuration affecting compilation
```

---

# 144. No server credentials in fingerprint

Nunca incluir:

```text
password
token
private key
```

---

# 145. SQL normalization

El Dialect podrá colaborar con un SQL formatter/emitter determinista.

---

# 146. Deterministic SQL

Mismo:

```text
AST
+
Plan
+
Dialect
+
Capabilities
```

deberá generar el mismo SQL.

---

# 147. Whitespace policy

El SQL Compiler podrá tener políticas:

```text
COMPACT
DEBUG
```

sin cambiar semántica.

---

# 148. Production SQL

Preferencia:

```text
compact deterministic SQL
```

---

# 149. Debug SQL

Podrá producir formatting más legible para diagnostics.

---

# 150. Parameter values not interpolated for debug

El formatter de debug no deberá crear SQL ejecutable concatenando secretos.

---

# 151. SQL comments

Comentarios automáticos podrán utilizarse para:

```text
trace IDs
query tags
operation names
```

sólo mediante políticas seguras.

---

# 152. Query tagging

Si una plataforma permite comentarios SQL:

```sql
/* voltstack:users.list */
SELECT ...
```

el Dialect puede describir sintaxis.

---

# 153. Tag sanitization

Los tags deberán sanitizarse estrictamente.

---

# 154. No user raw comments

No permitir inyección mediante metadata de observabilidad.

---

# 155. SQL keywords

Dialect podrá exponer:

```text
KeywordSet
```

---

# 156. Keyword casing

La salida oficial podrá usar:

```text
UPPERCASE SQL keywords
```

para consistencia.

---

# 157. Keyword casing not semantic

Será una decisión de emitter/formatting.

---

# 158. Escape hatch

VoltStack conservará:

```text
RawExpression
RawSql
```

como escape hatch.

---

# 159. Raw SQL semantics

Raw SQL evita parte de:

```text
AST portability
Dialect compilation
Semantic analysis
```

pero no deberá evitar automáticamente:

```text
parameter binding
execution lifecycle
telemetry
transaction context
connection management
```

---

# 160. RawExpression

Deberá distinguirse:

```text
trusted framework/developer SQL fragment
```

de:

```text
user input
```

---

# 161. Raw identifier prohibited by default

No deberá convertirse entrada de usuario directamente en:

```text
RawIdentifier
```

---

# 162. Dialect-aware raw fragments

Podrá existir:

```text
DialectSpecificExpression
```

para código intencionalmente no portable.

---

# 163. Explicit portability loss

Ejemplo conceptual:

```php
PostgreSql::expression('...')
```

deja claro:

```text
this query is PostgreSQL-specific
```

---

# 164. Portable API priority

La API principal deberá favorecer:

```text
semantic structured expressions
```

sobre raw SQL.

---

# 165. MySQL Dialect

Se implementará:

```text
MySqlDialect
```

---

# 166. MySQL Dialect areas

Incluirá sintaxis específica para:

```text
identifier quoting
limit
locking
upsert
json
functions
operators
DDL
generated values
index syntax
```

según versión/capabilities.

---

# 167. MariaDB Dialect

Se implementará:

```text
MariaDbDialect
```

separado conceptualmente de MySQL.

---

# 168. Why MariaDB separate

Aunque exista gran compatibilidad:

```text
MySQL
≠
MariaDB
```

en todas las versiones y features.

---

# 169. Shared MySQL-family components

Podrán compartir componentes:

```text
MySqlFamilyIdentifierDialect
MySqlFamilyParameterDialect
MySqlFamilyBaseSyntax
```

cuando la semántica realmente sea común.

---

# 170. Composition over inheritance

Preferencia:

```text
shared syntax components
```

sobre una jerarquía profunda:

```text
SqlDialect
  └── MySqlDialect
       └── MariaDbDialect
            └── ...
```

---

# 171. PostgreSQL Dialect

Se implementará:

```text
PostgreSqlDialect
```

con soporte para su sintaxis particular.

---

# 172. PostgreSQL extension richness

PostgreSQL posee numerosas extensiones sintácticas.

No todas deberán entrar al core.

---

# 173. PostgreSQL optional extensions

Ejemplos:

```text
arrays
jsonb
range types
operator classes
full-text search
special index methods
```

podrán estar en módulos especializados.

---

# 174. SQLite Dialect

Se implementará:

```text
SQLiteDialect
```

---

# 175. SQLite special characteristics

SQLite requiere atención a:

```text
limited ALTER behavior
PRAGMA
dynamic typing characteristics
RETURNING version support
UPSERT version support
JSON extension availability
```

---

# 176. PRAGMA

`PRAGMA` no deberá tratarse automáticamente como SQL portable.

---

# 177. Platform operation

Muchas operaciones `PRAGMA` pertenecen más naturalmente a:

```text
SQLitePlatform
```

o componentes especializados.

---

# 178. Dialect inheritance strategy

Preferencia general:

```text
Dialect composition
```

---

# 179. Shared syntax services

Ejemplo:

```text
AnsiIdentifierDialect
StandardExpressionDialect
StandardCteDialect
StandardWindowDialect
```

podrán componerse.

---

# 180. Standard SQL baseline

VoltStack podrá definir componentes internos basados en SQL estándar.

---

# 181. SQL standard is not a runtime target

No se asumirá que:

```text
SQL standard syntax
```

funciona igual en todos los motores.

---

# 182. Dialect fallback

Cuando un dialecto no sobrescriba una sintaxis estándar, podrá utilizar una implementación compartida si su Platform declara compatibilidad.

---

# 183. Capability gating

Nunca usar un fallback sólo porque existe una implementación sintáctica.

Debe cumplirse:

```text
syntax available
AND
platform supports semantics
```

---

# 184. Dialect syntax IDs

Podrán existir IDs estables:

```text
query.returning
query.upsert
query.lock
query.limit
expression.json.extract
schema.identity
schema.drop_column
```

---

# 185. Syntax registry

Para extensibilidad avanzada podrá existir:

```text
DialectSyntaxRegistry
```

---

# 186. Avoid stringly typed internals

Preferir:

```php
SyntaxFeature::Returning
```

o value objects a strings dispersos.

---

# 187. Compiler fragments

Cada fragment compiler deberá ser pequeño.

Ejemplo:

```text
ReturningClauseCompiler
LockClauseCompiler
PaginationCompiler
UpsertClauseCompiler
```

---

# 188. Fragment compiler ownership

Algunos fragment compilers pueden pertenecer al SQL Compiler y consumir Dialect services.

No es obligatorio que todos vivan dentro del Dialect package.

---

# 189. Dialect exposes syntax strategy

Regla:

```text
Dialect provides syntax strategy
Compiler orchestrates compilation
```

---

# 190. Query Compiler example

```text
SelectNode
    │
    ▼
SelectCompiler
    │
    ├── ExpressionCompiler
    ├── IdentifierDialect
    ├── PaginationDialect
    └── LockingDialect
    │
    ▼
SQL
```

---

# 191. Insert Compiler example

```text
InsertNode
    │
    ▼
InsertCompiler
    │
    ├── IdentifierDialect
    ├── ParameterDialect
    ├── UpsertDialect
    └── ReturningDialect
    │
    ▼
SQL
```

---

# 192. Schema Compiler example

```text
CreateTablePlan
       │
       ▼
SchemaCompiler
       │
       ├── DdlDialect
       ├── TypeCompiler
       └── Platform Capabilities
       │
       ▼
DDL
```

---

# 193. Type system relation

`Database Type System` y `Dialect` deberán mantenerse separados.

---

# 194. Semantic type vs SQL declaration

Ejemplo:

```text
StringType(length: 255)
```

podría expresarse como:

```text
VARCHAR(255)
```

pero el Type/Platform system decide compatibilidad y semántica.

---

# 195. Type compiler

Podrá existir:

```text
PlatformTypeCompiler
```

o estrategia equivalente.

No deberá convertir Dialect en Type Registry.

---

# 196. Cast syntax

El Dialect podrá definir:

```text
CAST
vendor shorthand casts
```

pero el Type System determina conversiones válidas.

---

# 197. Collation syntax

Dialect expresa sintaxis.

Platform determina disponibilidad/semántica.

---

# 198. Charset syntax

Mismo principio.

---

# 199. Schema-qualified names

Dialect deberá manejar:

```text
catalog
schema
table
column
```

sin asumir que todos los motores utilizan exactamente la misma jerarquía.

---

# 200. Namespace semantics

Platform será responsable de describir:

```text
catalog/schema/database semantics
```

---

# 201. Dialect context

Podrá existir:

```php
final readonly class DialectContext
{
    public function __construct(
        public DialectDescriptor $descriptor,
        public ServerVersion $serverVersion,
        public CapabilitySet $capabilities,
    ) {}
}
```

si se necesita para estrategias versionadas.

---

# 202. Prefer resolved dialect

En hot paths se preferirá:

```text
ResolvedDialect
```

para evitar reevaluar versión/capabilities constantemente.

---

# 203. ResolvedDialect

Podrá contener:

```text
descriptor
syntax components
reserved words
function registry
operator registry
fingerprint
```

---

# 204. ResolvedDialect immutability

Será inmutable.

---

# 205. Dialect compilation

Durante bootstrap/config compilation podrán preconstruirse dialectos conocidos.

---

# 206. Server-version-dependent refinement

Partes dependientes de server version podrán resolverse después del primer metadata handshake y cachearse por target compatible.

---

# 207. No cross-server false cache

No reutilizar:

```text
PostgreSQL 17 resolved dialect
```

como si necesariamente fuera idéntico a:

```text
PostgreSQL 14
```

---

# 208. Compatibility key

Podrá existir:

```text
DialectCompatibilityKey
```

---

# 209. Capability fingerprint

La compilación de SQL dependiente de features deberá incluir:

```text
CapabilityFingerprint
```

en sus caches.

---

# 210. Dialect version negotiation

Flujo:

```text
Configured Family
      │
      ▼
Connect
      │
      ▼
Discover Server
      │
      ▼
Resolve Platform
      │
      ▼
Resolve Effective Capabilities
      │
      ▼
Resolve Effective Dialect
```

---

# 211. Offline compilation

VoltStack deberá permitir cuando sea posible:

```text
compile migrations
generate SQL
analyze schema
```

sin conexión activa.

---

# 212. Offline target

Para ello podrá declararse:

```text
target platform
target version
```

en configuración.

---

# 213. Offline dialect

Ejemplo conceptual:

```text
postgresql@17
```

permitirá generar SQL para ese target.

---

# 214. Online validation

Posteriormente podrá verificarse que el servidor real coincida con el target.

---

# 215. Mismatch policy

Opciones:

```text
FAIL
WARN
ALLOW_COMPATIBLE
```

según configuración y criticidad.

---

# 216. Migration determinism

Generar una migration SQL no deberá cambiar accidentalmente según la máquina del desarrollador.

---

# 217. Explicit target recommended

Para build/deploy reproducible:

```text
database.target.platform
database.target.version
```

podrán fijarse.

---

# 218. Dialect serialization

Los descriptors/fingerprints podrán serializarse.

El Dialect runtime object completo no necesariamente.

---

# 219. No native resources

Un Dialect serializado nunca contendrá:

```text
PDO
socket
connection
statement
result
```

---

# 220. Dialect errors

Jerarquía sugerida:

```text
DialectException
├── DialectNotFoundException
├── DialectResolutionException
├── DialectVersionException
├── DialectSyntaxException
├── UnsupportedDialectSyntaxException
├── InvalidIdentifierException
├── DialectExtensionException
└── DialectConfigurationException
```

---

# 221. Unsupported syntax

Si el Planner llega al Compiler con una estrategia no soportada:

```text
UnsupportedDialectSyntaxException
```

podrá indicar un error arquitectónico o capability mismatch.

---

# 222. Prefer earlier failure

Idealmente una feature no soportada deberá detectarse antes:

```text
Semantic Validation
or
Planner
```

---

# 223. Compiler defensive validation

Aun así Compiler/Dialect deberán defender sus invariantes.

---

# 224. Diagnostics

Un error deberá poder explicar:

```text
semantic feature
dialect
platform
server version
required capability
available alternatives
```

---

# 225. Example diagnostic

```text
Cannot compile RETURNING for target:

Dialect: mysql
Platform: mysql
Server: 8.x

Requested semantic feature:
query.returning.rows

Capability:
UNSUPPORTED

Suggested strategy:
generated-key retrieval
```

---

# 226. No misleading vendor errors

Evitar:

```text
syntax error near ...
```

cuando VoltStack puede detectar incompatibilidad antes de enviar SQL.

---

# 227. Telemetry

El Dialect System podrá exponer metadata como:

```text
dialect ID
dialect version
compiler strategy
emulation strategy
```

---

# 228. Low cardinality

No emitir cada SQL fragment como atributo de alta cardinalidad.

---

# 229. Compilation telemetry

Eventos conceptuales:

```text
DialectResolved
DialectResolutionFailed
DialectExtensionLoaded
DialectFallbackSelected
DialectCompatibilityWarning
```

---

# 230. Events optional

No se requerirá EventSystem para que Dialect funcione.

---

# 231. Telemetry optional

No se requerirá Telemetry para compilar SQL.

---

# 232. Dialect testing

Cada dialecto oficial tendrá:

```text
DialectConformanceSuite
```

---

# 233. Conformance areas

```text
identifiers
parameters
literals
operators
functions
pagination
locking
returning
upsert
CTE
windows
JSON
DDL
version behavior
extension behavior
determinism
```

---

# 234. Golden SQL tests

Podrán existir:

```text
AST
→ expected SQL
```

---

# 235. Golden tests limitation

No deberán ser la única forma de testing.

También se necesitan:

```text
semantic tests
property tests
integration tests
server tests
```

---

# 236. Identifier property tests

Ejemplo:

```text
quote
→ escape embedded quote correctly
```

---

# 237. Injection tests

Identificadores maliciosos deberán:

```text
reject
or
safely quote
```

según API.

---

# 238. Parameter tests

Asegurar que valores dinámicos:

```text
never become SQL syntax accidentally
```

---

# 239. Function tests

Verificar:

```text
argument order
special grammar
capability gating
```

---

# 240. Version tests

Ejemplo:

```text
SQLite version A
→ feature unsupported

SQLite version B
→ feature supported
```

---

# 241. MySQL/MariaDB divergence tests

Toda divergencia conocida deberá tener tests independientes.

---

# 242. Offline compilation tests

Deberá ser posible compilar para:

```text
target version
```

sin servidor.

---

# 243. Online compatibility tests

Con servidor real:

```text
compiled SQL
→ execute
→ expected semantics
```

---

# 244. Persistent runtime tests

Múltiples requests concurrentes deberán poder usar el mismo Dialect inmutable.

---

# 245. No mutable compiler state test

Ejecutar compilaciones paralelas y verificar ausencia de contaminación.

---

# 246. Extension collision tests

Duplicados no autorizados deberán fallar.

---

# 247. Registry freeze tests

Después de bootstrap:

```text
register new dialect
→ exception
```

---

# 248. Architecture tests

Prohibir:

```text
Dialect → ORM
Dialect → EntityManager
Dialect → UnitOfWork
Dialect → Repository
Dialect → HTTP
Dialect → Controller
Dialect → ConnectionPool
```

---

# 249. Driver dependency test

Dialect no deberá depender directamente de:

```text
PDO
mysqli
pgsql native resource
```

---

# 250. Vendor switch test

Core Compiler deberá evitar:

```php
switch ($connection->driverName()) {
}
```

---

# 251. Suggested directory structure

```text
VoltStack/Quantum/Database/Dialect
│
├── Contract
│   ├── DialectInterface.php
│   ├── IdentifierDialectInterface.php
│   ├── ParameterDialectInterface.php
│   ├── LiteralDialectInterface.php
│   ├── OperatorDialectInterface.php
│   ├── FunctionDialectInterface.php
│   ├── PaginationDialectInterface.php
│   ├── LockingDialectInterface.php
│   ├── ReturningDialectInterface.php
│   ├── UpsertDialectInterface.php
│   ├── JsonDialectInterface.php
│   └── DdlDialectInterface.php
│
├── Descriptor
│   ├── DialectDescriptor.php
│   ├── DialectId.php
│   ├── DialectFamily.php
│   ├── DialectVersion.php
│   └── DialectVersionRange.php
│
├── Registry
│   ├── DialectRegistry.php
│   ├── FunctionDialectRegistry.php
│   ├── OperatorDialectRegistry.php
│   └── DialectSyntaxRegistry.php
│
├── Resolution
│   ├── DialectResolver.php
│   ├── ResolvedDialect.php
│   ├── DialectContext.php
│   ├── DialectFingerprint.php
│   └── DialectCompatibilityKey.php
│
├── Identifier
│   ├── Identifier.php
│   ├── QualifiedIdentifier.php
│   ├── IdentifierQuoter.php
│   └── ReservedWordSet.php
│
├── Parameter
│   ├── ParameterPlaceholderStrategy.php
│   ├── ParameterPosition.php
│   └── SqlBindingProfile.php
│
├── Expression
│   ├── OperatorId.php
│   ├── FunctionId.php
│   ├── FunctionSyntax.php
│   └── ExpressionSyntaxRegistry.php
│
├── Query
│   ├── PaginationDialect.php
│   ├── LockingDialect.php
│   ├── ReturningDialect.php
│   ├── UpsertDialect.php
│   ├── CteDialect.php
│   └── WindowDialect.php
│
├── Schema
│   ├── DdlDialect.php
│   ├── ConstraintDialect.php
│   ├── IndexDialect.php
│   └── IdentityDialect.php
│
├── Extension
│   ├── DialectExtensionInterface.php
│   ├── DialectExtensionDescriptor.php
│   └── DialectExtensionRegistry.php
│
├── Standard
│   ├── StandardIdentifierDialect.php
│   ├── StandardExpressionDialect.php
│   ├── StandardCteDialect.php
│   └── StandardWindowDialect.php
│
├── MySql
│   ├── MySqlDialect.php
│   └── ...
│
├── MariaDb
│   ├── MariaDbDialect.php
│   └── ...
│
├── PostgreSql
│   ├── PostgreSqlDialect.php
│   └── ...
│
├── SQLite
│   ├── SQLiteDialect.php
│   └── ...
│
└── Exception
    ├── DialectException.php
    ├── DialectNotFoundException.php
    ├── DialectResolutionException.php
    ├── DialectVersionException.php
    ├── UnsupportedDialectSyntaxException.php
    └── InvalidIdentifierException.php
```

---

# 252. API classification

## Public

Muy pequeña.

Principalmente indirecta mediante:

```text
Connection
Query Builder
Schema Builder
DB Facade
```

## Extension

Incluye contratos seguros para:

```text
custom dialect
custom function
custom operator
custom syntax compiler
```

## Internal

Incluye:

```text
resolution machinery
compiler fragments
syntax registries
compatibility machinery
```

---

# 253. Custom dialect

Un paquete podrá implementar:

```text
CustomDialect
```

sin modificar VoltStack core.

---

# 254. Custom dialect requirements

Deberá proporcionar:

```text
descriptor
syntax components
compatibility information
capability integration
conformance tests
```

---

# 255. Custom dialect alone may not be enough

Agregar soporte completo para una nueva base puede requerir:

```text
Driver
Dialect
Platform
Schema support
Type mappings
Compiler support
Capability provider
```

---

# 256. Dialect-only extension

En otros casos basta con añadir:

```text
function
operator
syntax feature
```

a una plataforma existente.

---

# 257. Architectural invariants

## DB-DIALECT-001

Driver, Dialect, Platform y Connection serán abstracciones distintas.

## DB-DIALECT-002

Dialect no abrirá conexiones.

## DB-DIALECT-003

Dialect no ejecutará SQL.

## DB-DIALECT-004

Dialect no administrará transacciones.

## DB-DIALECT-005

Dialect no conocerá ORM.

## DB-DIALECT-006

Dialect no conocerá UnitOfWork.

## DB-DIALECT-007

Dialect no conocerá IdentityMap.

## DB-DIALECT-008

Dialect no administrará pooling.

## DB-DIALECT-009

Dialect no decidirá read/write routing.

## DB-DIALECT-010

Dialect expresará sintaxis, no política de negocio.

## DB-DIALECT-011

Query Builder no generará SQL específico de proveedor.

## DB-DIALECT-012

Query AST conservará intención semántica portable cuando sea posible.

## DB-DIALECT-013

Dialect no será un segundo SQL Compiler.

## DB-DIALECT-014

Compiler orquestará; Dialect proporcionará estrategias sintácticas.

## DB-DIALECT-015

Platform describirá semántica y capacidades del motor.

## DB-DIALECT-016

Capabilities determinarán disponibilidad de features.

## DB-DIALECT-017

Dialect no asumirá que una sintaxis disponible implica semántica soportada.

## DB-DIALECT-018

Identificadores y parámetros tendrán pipelines diferentes.

## DB-DIALECT-019

Valores dinámicos utilizarán binding por defecto.

## DB-DIALECT-020

El quoting de identificadores será responsabilidad especializada.

## DB-DIALECT-021

No se interpolará user input para construir SQL.

## DB-DIALECT-022

Las diferencias vendor-specific estarán aisladas.

## DB-DIALECT-023

MySQL y MariaDB podrán compartir componentes pero mantendrán dialectos conceptualmente distintos.

## DB-DIALECT-024

Dialect será inmutable/stateless cuando sea posible.

## DB-DIALECT-025

Dialect no almacenará estado de request.

## DB-DIALECT-026

Dialect no almacenará estado de tenant.

## DB-DIALECT-027

Dialect no almacenará estado de transaction.

## DB-DIALECT-028

Dialect deberá ser seguro para persistent runtimes.

## DB-DIALECT-029

Dialect deberá ser concurrency-safe.

## DB-DIALECT-030

Los registries se congelarán después de bootstrap.

## DB-DIALECT-031

No existirá silent last-registration-wins.

## DB-DIALECT-032

Extensiones usarán contratos públicos de extensión.

## DB-DIALECT-033

Las features no portables serán explícitas.

## DB-DIALECT-034

Raw SQL será un escape hatch, no la arquitectura principal.

## DB-DIALECT-035

Dialect resolution no requerirá conexión eager cuando pueda evitarse.

## DB-DIALECT-036

Server-version-dependent syntax será versionada explícitamente.

## DB-DIALECT-037

Offline compilation será soportada cuando sea posible.

## DB-DIALECT-038

Compiled SQL será determinista para iguales inputs.

## DB-DIALECT-039

Los caches de compilación incluirán fingerprint del dialecto cuando corresponda.

## DB-DIALECT-040

Credentials nunca formarán parte del Dialect fingerprint.

## DB-DIALECT-041

Los errores de capability deberán detectarse lo antes posible.

## DB-DIALECT-042

Compiler mantendrá validación defensiva.

## DB-DIALECT-043

Cada dialecto oficial tendrá conformance tests.

## DB-DIALECT-044

No habrá dependencias directas del Dialect hacia PDO u otros clientes nativos.

## DB-DIALECT-045

Vendor conditionals estarán prohibidos en capas superiores.

---

# 258. Anti-pattern — Driver equals Dialect

```text
PdoMySqlDriver
    ├── connect
    ├── quote identifiers
    ├── compile SELECT
    ├── compile schema
    ├── determine capabilities
    └── execute
```

**Prohibido.**

---

# 259. Anti-pattern — Dialect equals Compiler

```text
MySqlDialect::compileWholeQuery(AST)
```

como única arquitectura.

Esto mezcla:

```text
compiler orchestration
+
vendor syntax
```

---

# 260. Anti-pattern — Vendor checks in Query Builder

```php
if (DB::driver() === 'pgsql') {
    // build PostgreSQL query
}
```

**Prohibido para la API portable.**

---

# 261. Anti-pattern — SQL strings in AST

```php
new LimitNode('LIMIT 20');
```

**Prohibido.**

Preferir:

```php
new LimitNode(20);
```

---

# 262. Anti-pattern — Raw vendor syntax as semantic model

```php
new UpsertNode(
    'ON DUPLICATE KEY UPDATE ...'
);
```

**Prohibido.**

---

# 263. Anti-pattern — Platform features inside Dialect

```php
$dialect->supportsTransactions();
```

no será la ubicación principal.

Eso pertenece a:

```text
Platform Capability System
```

---

# 264. Anti-pattern — Connection-aware Dialect

```php
$dialect->setCurrentConnection($connection);
```

**Prohibido.**

---

# 265. Anti-pattern — Mutable parameter counter

```php
$dialect->nextParameter++;
```

**Prohibido** en dialectos compartidos.

---

# 266. Anti-pattern — Global custom function mutation

```php
Dialect::registerFunction(...)
```

durante una petición.

**Prohibido.**

---

# 267. Anti-pattern — Assume SQL Standard solves portability

```text
Use ANSI SQL everywhere
→ all databases behave the same
```

**Incorrecto.**

---

# 268. Anti-pattern — Lowest common denominator

VoltStack tampoco limitará toda su API a las capacidades de SQLite más antiguas sólo para mantener falsa portabilidad.

---

# 269. Portability levels

Podrán definirse:

```text
PORTABLE
PORTABLE_WITH_EMULATION
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
```

---

# 270. Query portability metadata

El Semantic/Planner system podrá marcar el nivel de portabilidad de una query.

---

# 271. Example portable query

```php
DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->limit(20)
    ->get();
```

Conceptualmente:

```text
Builder
  │
  ▼
Portable AST
  │
  ▼
Planner
  │
  ▼
Compiler
  │
  ├── MySQL Dialect
  ├── PostgreSQL Dialect
  └── SQLite Dialect
```

---

# 272. Example platform-specific query

Una característica avanzada podría declararse explícitamente:

```text
PostgreSQL-specific operation
```

sin contaminar el AST portable central.

---

# 273. Example UPSERT flow

```text
Application
   │
   ▼
upsert(...)
   │
   ▼
Semantic Upsert Model
   │
   ▼
Capability Analysis
   │
   ▼
Planner
   │
   ├── Native Strategy
   ├── Emulated Strategy
   └── Unsupported
   │
   ▼
Compiler
   │
   ▼
Dialect
   │
   ▼
Vendor SQL
```

---

# 274. Example identifier flow

```text
User Model Metadata
      │
      ▼
Identifier("users")
      │
      ▼
Compiler
      │
      ▼
IdentifierDialect
      │
      ├── MySQL      → `users`
      └── PostgreSQL → "users"
```

---

# 275. Example parameter flow

```text
where email = user value
          │
          ▼
QueryParameter
          │
          ▼
ParameterAllocator
          │
          ▼
Placeholder Strategy
          │
          ▼
SQL placeholder
          │
          ▼
Compiled bindings
          │
          ▼
Driver
```

El valor nunca necesita formar parte de la sintaxis SQL.

---

# 276. Example migration flow

```text
Schema Model
    │
    ▼
Schema Diff
    │
    ▼
Migration Planner
    │
    ▼
Schema Operations
    │
    ▼
Schema Compiler
    │
    ├── Platform capabilities
    └── DDL Dialect
    │
    ▼
Vendor DDL
```

---

# 277. Dialect and Semantic Engine

El Semantic Engine no deberá preguntar:

```text
what SQL string should I generate?
```

Deberá preguntar:

```text
is this semantic operation valid?
what does this expression mean?
what types result?
```

---

# 278. Dialect and Optimizer

Optimizer no deberá producir vendor SQL.

Puede utilizar capabilities para seleccionar transformaciones válidas.

---

# 279. Dialect and Planner

Planner selecciona:

```text
execution/compilation strategy
```

considerando capabilities.

---

# 280. Dialect and Compiler

Aquí se encuentra la integración principal.

```text
Plan
+
Dialect
+
Platform
+
Capabilities
=
Compiled SQL
```

---

# 281. Dialect and Executor

Executor no debería necesitar comprender Dialect.

Recibe:

```text
CompiledQuery
```

---

# 282. Dialect and Driver

Driver tampoco necesita comprender AST.

Recibe:

```text
SQL
bindings
binding metadata
```

---

# 283. Full separation

```text
Semantic Intent
      │
      ▼
     AST
      │
      ▼
Semantic Analysis
      │
      ▼
Optimizer
      │
      ▼
Planner
      │
      ▼
Compilation Strategy
      │
      ▼
SQL Compiler
      │
      ├──────── Dialect
      ├──────── Platform
      └──────── Capabilities
      │
      ▼
CompiledQuery
      │
      ▼
Executor
      │
      ▼
Connection
      │
      ▼
Driver
      │
      ▼
Native Client
      │
      ▼
Database
```

---

# 284. Architectural equation

```text
Portable Database API
=
Semantic Model
+
Capability Awareness
+
Dialect Isolation
```

---

# 285. SQL generation equation

```text
Compiled SQL
=
Semantic Plan
+
Dialect Syntax
+
Platform Semantics
+
Effective Capabilities
+
Binding Profile
```

---

# 286. Extensibility equation

```text
New SQL Feature
=
Semantic Definition
+
Capability Requirement
+
Planner Strategy
+
Compiler Support
+
Dialect Syntax
```

cuando todas esas capas sean necesarias.

---

# 287. New database equation

El soporte completo para un nuevo motor será aproximadamente:

```text
Database Support
=
Driver
+
Dialect
+
Platform
+
Capabilities
+
Types
+
Schema Integration
+
Compiler Integration
+
Conformance Tests
```

---

# 288. V1 implementation priorities

Para V1 deberán priorizarse:

```text
identifier quoting
parameter placeholders
basic expressions
basic functions
SELECT syntax
INSERT syntax
UPDATE syntax
DELETE syntax
JOIN syntax
LIMIT/OFFSET
ORDER BY
GROUP BY
HAVING
CTE
basic locking
RETURNING
UPSERT
basic JSON
basic DDL
```

---

# 289. V2+

Posteriormente:

```text
advanced JSON
full-text
advanced indexes
vendor operators
array syntax
range syntax
advanced window frames
advanced DDL
vendor extension packs
```

---

# 290. Final architecture

```text
                   QUERY / SCHEMA MODEL
                           │
                           ▼
                     Semantic Layer
                           │
                           ▼
                       Planner
                           │
                           ▼
                       Compiler
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
          Dialect       Platform     Capabilities
             │             │             │
     ┌───────┼───────┐     │             │
     ▼       ▼       ▼     │             │
Identifiers Ops   Functions│             │
     │       │       │     │             │
     ├───────┼───────┤     │             │
     ▼       ▼       ▼     │             │
 Pagination Locking JSON   │             │
     │       │       │     │             │
     ├───────┼───────┤     │             │
     ▼       ▼       ▼     │             │
 Returning Upsert   DDL    │             │
     │       │       │     │             │
     └───────┴───────┴─────┴─────────────┘
                           │
                           ▼
                     SQL Emitter
                           │
                           ▼
                    CompiledQuery
                           │
                           ▼
                       Executor
                           │
                           ▼
                      Connection
                           │
                           ▼
                        Driver
```

---

# 291. Regla maestra

> El Dialect sabe cómo expresar SQL para una familia de bases de datos, pero no decide qué quiere hacer la aplicación, qué estrategia semántica debe utilizar el Planner ni cómo se comunica VoltStack con el servidor.

En forma resumida:

```text
Application
=
intent

AST
=
structure

Semantic Engine
=
meaning

Optimizer
=
improvement

Planner
=
strategy

Dialect
=
SQL expression rules

Platform
=
database semantics and capabilities

Compiler
=
translation

Executor
=
execution orchestration

Connection
=
logical database access

Driver
=
native communication
```

---

# 292. Resultado arquitectónico

Con este diseño, VoltStack podrá ofrecer una API:

```php
DB::table('users')
    ->where('active', true)
    ->limit(20)
    ->get();
```

sin que el desarrollador necesite conocer:

```text
MySQL syntax
PostgreSQL syntax
MariaDB syntax
SQLite syntax
```

y sin que internamente exista una arquitectura basada en:

```php
if ($driver === 'mysql') {
    ...
}
```

distribuida por todo el framework.

La especialización quedará encapsulada:

```text
Query Semantic Model
        │
        ▼
     Compiler
        │
        ▼
Resolved Dialect
        │
        ├── MySQL
        ├── MariaDB
        ├── PostgreSQL
        └── SQLite
```

---

# 293. Criterios de aceptación

`Database Dialect System` estará correctamente diseñado cuando:

```text
Driver and Dialect are independent

Platform and Dialect are independent

Query Builder produces no vendor SQL

AST stores semantic structures rather than SQL fragments

identifiers are modeled structurally

identifier quoting is dialect-aware

parameter values remain bound

operators can be semantically represented

functions can be dialect-resolved

pagination remains semantic

locking remains semantic

RETURNING remains semantic

UPSERT remains semantic

CTEs remain semantic

window expressions remain semantic

JSON operations can be capability-aware

DDL uses structured Schema Model

vendor-specific extensions remain isolated

MySQL and MariaDB can diverge safely

PostgreSQL extensions can be modular

SQLite differences remain isolated

dialects are immutable and concurrency-safe

persistent workers share no mutable dialect state

dialect registries freeze after bootstrap

custom dialects can be registered through extension contracts

offline compilation is possible

version-specific syntax can be resolved

compiled query caches account for dialect compatibility

raw SQL remains available as explicit escape hatch

security never depends on manual SQL escaping

conformance suites validate every official dialect
```

---

# 294. Relación con los documentos anteriores

```text
10_DATABASE_DRIVER_ARCHITECTURE
          │
          ▼
11_DATABASE_CONNECTION_SYSTEM
          │
          ▼
12_DATABASE_CONNECTION_MANAGER
          │
          ▼
13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION
          │
          ▼
14_DATABASE_CONNECTION_POOLING_SYSTEM
          │
          ▼
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM
          │
          ▼
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM
          │
          ▼
17_DATABASE_DIALECT_SYSTEM
```

Con esto quedan formalmente separados:

```text
transport
connection abstraction
connection resolution
pooling
lifecycle
state/reset
SQL syntax
```

---

# 295. Siguiente documento

El siguiente documento será:

```text
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md
```

Su responsabilidad será responder una pregunta diferente:

```text
Dialect:
"¿Cómo se escribe?"

Platform Capability System:
"¿Puede hacerlo este motor, con esta versión,
configuración y contexto?"
```

Ejemplos:

```text
supportsReturning()
supportsNativeUpsert()
supportsRecursiveCte()
supportsSavepoints()
supportsTransactionalDdl()
supportsDeferrableConstraints()
supportsPartialIndexes()
supportsExpressionIndexes()
supportsJson()
supportsWindowFunctions()
supportsSkipLocked()
supportsGeneratedColumns()
supportsNativeSessionReset()
```

La arquitectura continuará:

```text
Driver
   │
   ▼
Connection
   │
   ▼
Dialect
   │
   ▼
Platform + Capabilities
   │
   ▼
Concrete Platforms
```

y posteriormente:

```text
19_DATABASE_MYSQL_AND_MARIADB_PLATFORM.md
20_DATABASE_POSTGRESQL_PLATFORM.md
21_DATABASE_SQLITE_PLATFORM.md
22_DATABASE_DRIVER_EXTENSION_SYSTEM.md
```

---

# 296. Conclusión

`17_DATABASE_DIALECT_SYSTEM.md` establece que VoltStack no construirá portabilidad mediante condicionales de proveedor dispersos.

La portabilidad se obtendrá mediante:

```text
Semantic Intent
      +
Structured AST
      +
Capability Analysis
      +
Planning
      +
Dialect Isolation
      +
Platform Semantics
```

La separación definitiva queda:

```text
┌─────────────────────────────────────────────┐
│                 APPLICATION                 │
└─────────────────────┬───────────────────────┘
                      ▼
               Semantic Intent
                      │
                      ▼
                     AST
                      │
                      ▼
              Semantic Analysis
                      │
                      ▼
                  Optimizer
                      │
                      ▼
                   Planner
                      │
                      ▼
                  Compiler
                ┌─────┴─────┐
                ▼           ▼
             Dialect     Platform
                │           │
                │      Capabilities
                │           │
                └─────┬─────┘
                      ▼
                Compiled SQL
                      │
                      ▼
                   Executor
                      │
                      ▼
                  Connection
                      │
                      ▼
                    Driver
                      │
                      ▼
                Native Client
                      │
                      ▼
                 DATABASE
```

La regla que gobernará todo el subsystem será:

> **El modelo expresa intención; el Planner elige estrategia; el Dialect expresa sintaxis; el Platform define semántica y capacidades; el Driver realiza la comunicación.**