# 297_DATABASE_CUSTOM_DIALECT_SYSTEM.md

# VoltStack Quantum Database
## Custom Dialect System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 297 — Custom Dialect System  
**Bloque:** 30 — Extensibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `296_DATABASE_CUSTOM_DRIVER_SYSTEM.md`  
**Siguiente documento:** `298_DATABASE_CUSTOM_COMPILER_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial para implementar, registrar, resolver y validar **dialectos SQL personalizados** dentro de:

```text
VoltStack/Quantum/Database
```

El Custom Dialect System permitirá que VoltStack represente SQL compatible con nuevas familias, variantes o versiones de lenguajes SQL sin introducir condicionales dispersos como:

```php
if ($database === 'postgresql') {
    // ...
}

if ($database === 'mysql') {
    // ...
}
```

a través de:

```text
Query Builder
ORM
Schema
Execution Engine
Driver
```

La regla central será:

> **Un dialecto personalizado describirá reglas sintácticas y convenciones de representación de una familia SQL; no ejecutará consultas, no abrirá conexiones, no realizará planificación semántica, no determinará por sí mismo la disponibilidad efectiva de capacidades y no sustituirá al SQL Compiler.**

Formalmente:

```text
Custom Dialect
=
Identity
+
Descriptor
+
Syntax Rules
+
Identifier Rules
+
Quoting Rules
+
Placeholder Rules
+
Expression Syntax
+
Statement Syntax
+
Feature Syntax
+
Compiler-facing Contracts
```

pero:

```text
Custom Dialect
≠
SQL Compiler
```

---

# 2. Objetivos

El sistema deberá permitir:

1. incorporar nuevos dialectos;
2. representar variantes SQL;
3. registrar dialectos mediante extensiones;
4. distribuir dialectos mediante plugins;
5. identificar dialectos de forma estable;
6. separar dialecto de plataforma;
7. separar dialecto de driver;
8. separar dialecto de compiler;
9. definir reglas de identifiers;
10. definir quoting;
11. definir placeholders;
12. definir operadores;
13. definir funciones;
14. definir sintaxis DML;
15. colaborar con sintaxis DDL;
16. definir pagination syntax;
17. definir locking syntax;
18. definir `RETURNING`;
19. definir CTE syntax;
20. definir window syntax;
21. manejar variantes por versión;
22. integrarse con capabilities;
23. validar representabilidad;
24. permitir extensión controlada;
25. mantener determinismo;
26. soportar caching;
27. soportar persistent runtimes;
28. proporcionar conformance testing.

---

# 3. Posición arquitectónica

El dialecto se encuentra entre el conocimiento semántico y la generación concreta de SQL:

```text
Query AST
   ↓
Semantic Analysis
   ↓
Optimizer
   ↓
Planner
   ↓
SQL Compiler
   ↓
Dialect
   ↓
SQL Representation
   ↓
Driver
   ↓
Database
```

Sin embargo, esta relación no significa que el dialecto compile el AST completo.

---

# 4. Dialect ≠ Compiler

Esta será una de las separaciones fundamentales.

El Compiler conoce:

```text
AST
Query Plan
Compilation Context
Statement structure
```

El Dialect conoce:

```text
syntax conventions
quoting
keywords
operator spelling
function forms
feature-specific syntax
```

Por tanto:

```text
Compiler
uses
Dialect
```

y no:

```text
Dialect
replaces
Compiler
```

---

# 5. Dialect ≠ Driver

El Dialect define:

```text
How SQL is written
```

El Driver define:

```text
How commands reach the database
```

Por tanto:

```text
Dialect
≠
Driver
```

---

# 6. Dialect ≠ Platform

El Platform System describe características semánticas y capacidades de una familia de DBMS.

El Dialect describe representación sintáctica.

```text
Dialect
≠
Platform
```

Ejemplo:

```text
PostgreSQL Platform
```

puede indicar que una determinada capability existe.

El dialecto PostgreSQL puede indicar:

```text
cómo se representa sintácticamente
```

esa capability.

---

# 7. Dialect ≠ Capability System

Un dialecto podrá conocer que posee una representación para una característica.

Pero:

```text
Syntax Exists
≠
Feature Available
```

---

# 8. Dialect ≠ Query AST

El AST deberá continuar siendo independiente del dialecto siempre que sea posible.

```text
Query AST
```

representa intención semántica.

```text
Dialect
```

representa sintaxis.

---

# 9. Dialect ≠ Query Builder

El Query Builder no deberá producir sintaxis específica del dialecto.

Incorrecto:

```php
$query->whereRawPostgres(...);
```

como diseño general.

Preferido:

```text
Query Builder
   ↓
Semantic AST
   ↓
Compiler + Dialect
```

---

# 10. Dialect ≠ ORM

El dialecto no deberá conocer:

```text
Entity
Repository
UnitOfWork
IdentityMap
Relationship
```

---

# 11. Arquitectura general

```text
Custom Dialect Package
        │
        ▼
Database Plugin
        │
        ▼
Dialect Extension
        │
        ▼
Dialect Descriptor
        │
        ▼
Dialect Registry
        │
        ▼
Dialect Resolver
        │
        ▼
Dialect Factory
        │
        ▼
SQL Dialect
        │
        ├── Identifier Rules
        ├── Quoting Rules
        ├── Placeholder Rules
        ├── Operator Syntax
        ├── Function Syntax
        ├── DML Syntax
        ├── Pagination Syntax
        ├── Lock Syntax
        └── Feature Syntax
                 │
                 ▼
             Compiler
                 │
                 ▼
          Compiled Statement
```

---

# 12. Dialect Identity

Todo dialecto tendrá una identidad estable:

```text
DialectId
```

Ejemplos:

```text
mysql
mariadb
postgresql
sqlite

acme.oracle
acme.customsql
```

---

# 13. DialectId

Conceptualmente:

```php
final readonly class DialectId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Deberá ser:

```text
stable
unique
normalized
case-safe
```

---

# 14. DialectId ≠ Class Name

No se utilizará el FQCN como identidad durable.

```text
DialectId
≠
ImplementationClass
```

---

# 15. Dialect Family

Podrá existir:

```text
DialectFamily
```

para representar familias relacionadas.

Ejemplo:

```text
mysql-family
```

con variantes:

```text
mysql
mariadb
```

Pero esto no deberá eliminar sus diferencias.

---

# 16. MySQL ≠ MariaDB

Regla:

```text
MySQL Dialect
≠
MariaDB Dialect
```

aunque compartan gran parte de la sintaxis.

---

# 17. Dialect Version

Podrá existir:

```text
DialectVersion
```

para versionar el contrato sintáctico del dialecto.

---

# 18. DialectVersion ≠ ServerVersion

Un mismo dialect implementation puede soportar múltiples versiones de servidor.

```text
DialectVersion
≠
DatabaseServerVersion
```

---

# 19. Dialect Descriptor

Conceptualmente:

```php
final readonly class DialectDescriptor
{
    public function __construct(
        public DialectId $id,
        public DialectVersion $version,
        public DialectFamily $family,
        public DialectFeatureSet $features,
    ) {}
}
```

---

# 20. Descriptor ≠ Dialect Instance

El descriptor deberá poder inspeccionarse durante bootstrap sin requerir conexión.

---

# 21. Dialect Registry

Resolverá:

```text
DialectId
→
DialectFactory
```

---

# 22. Registry Lifecycle

```text
BUILDING
   ↓
VALIDATING
   ↓
COMPILED
   ↓
FROZEN
```

---

# 23. Duplicate DialectId

Será error explícito:

```text
DuplicateDialectException
```

No:

```text
last registration wins
```

---

# 24. Dialect Factory

Contrato conceptual:

```php
interface DialectFactory
{
    public function id(): DialectId;

    public function create(
        DialectConfiguration $configuration
    ): SqlDialect;
}
```

---

# 25. Dialect Resolution

Podrá resolverse a partir de:

```text
explicit configuration
platform mapping
plugin contribution
server evidence
```

---

# 26. Explicit Resolution

Ejemplo:

```php
'dialect' => 'postgresql',
```

---

# 27. Automatic Resolution

Podrá existir cuando:

```text
PlatformId
→
DialectId
```

sea inequívoco.

---

# 28. Ambiguous Resolution

Si existen varias opciones válidas:

```text
AmbiguousDialectException
```

---

# 29. Driver Does Not Choose Dialect

Un Driver podrá proporcionar hints.

Pero:

```text
Driver
≠
Dialect Authority
```

Ejemplo:

```text
pdo.mysql
```

no deberá implicar automáticamente toda la semántica de una versión concreta de MySQL.

---

# 30. Platform-Dialect Mapping

Podrá existir:

```text
PlatformId
+
PlatformVersion
+
Capabilities
→
DialectSelection
```

---

# 31. Dialect Context

El dialecto podrá recibir un contexto inmutable:

```php
final readonly class DialectContext
{
    public function __construct(
        public PlatformId $platform,
        public PlatformVersion $version,
        public CapabilitySnapshot $capabilities,
    ) {}
}
```

---

# 32. Dialect Context ≠ Runtime Connection

No deberá contener:

```text
active transaction
current connection
current entity manager
current tenant mutable state
```

---

# 33. Identifier Rules

El dialecto deberá definir reglas para representar identifiers.

Ejemplos:

```text
table
column
schema
alias
index
constraint
```

---

# 34. Identifier ≠ Raw String

VoltStack deberá preferir:

```text
Identifier object
```

sobre strings arbitrarios.

---

# 35. Identifier Quoting

Ejemplos:

PostgreSQL:

```sql
"users"
```

MySQL:

```sql
`users`
```

---

# 36. Quoting Contract

Conceptualmente:

```php
interface IdentifierQuoter
{
    public function quote(
        SqlIdentifier $identifier
    ): string;
}
```

---

# 37. Quote ≠ Validate

Una cadena quoted no se vuelve automáticamente un identifier válido.

```text
Quoting
≠
Validation
```

---

# 38. Identifier Validation

Deberá ocurrir antes o durante compilation mediante contratos adecuados.

---

# 39. Qualified Identifiers

Ejemplo:

```text
schema.table.column
```

no deberá tratarse necesariamente como una sola cadena.

Preferido:

```text
QualifiedIdentifier
├── schema
├── table
└── column
```

---

# 40. Identifier Case Rules

El dialecto podrá describir:

```text
case folding
case sensitivity
quoted identifier behavior
```

cuando sea necesario.

---

# 41. Keyword Registry

Podrá proporcionar:

```text
reserved keywords
contextual keywords
```

para quoting/validation.

---

# 42. Keyword List ≠ Parser

El dialecto no necesita convertirse en un parser SQL completo.

---

# 43. Literal Representation

En casos donde un literal deba representarse directamente, el dialecto podrá definir reglas.

Sin embargo:

> **Los valores dinámicos deberán continuar parametrizándose por defecto.**

---

# 44. Value Parameters First

Preferido:

```sql
WHERE email = ?
```

o equivalente.

No:

```sql
WHERE email = 'concatenated-value'
```

---

# 45. Literal Encoder

Sólo deberá utilizarse para categorías donde la compilación requiera una representación literal segura.

Ejemplos potenciales:

```text
boolean keywords
certain static SQL tokens
compiler-generated constants
```

---

# 46. Raw User Values

Nunca deberán convertirse en SQL mediante un literal encoder como alternativa general al parameter binding.

---

# 47. Boolean Representation

Podrá variar:

```text
TRUE / FALSE
1 / 0
```

según contexto y plataforma.

---

# 48. NULL

El dialecto deberá representar:

```sql
NULL
```

según gramática correspondiente.

Pero:

```text
NULL literal
≠
missing parameter
```

---

# 49. Parameter Placeholder Strategy

El dialecto podrá colaborar en la representación de placeholders.

Ejemplos:

```text
?
:name
$1
$2
```

---

# 50. Placeholder Strategy ≠ Parameter Binding

El Dialect/Compiler decide representación.

El Driver realiza binding.

```text
Placeholder Syntax
≠
Binding Operation
```

---

# 51. Placeholder Contract

Conceptualmente:

```php
interface ParameterPlaceholderStrategy
{
    public function placeholder(
        ParameterPosition $position,
        ParameterName $name
    ): string;
}
```

---

# 52. Positional Parameters

Ejemplo:

```sql
WHERE id = ?
```

---

# 53. Numbered Parameters

Ejemplo:

```sql
WHERE id = $1
```

---

# 54. Named Parameters

Ejemplo:

```sql
WHERE id = :id
```

---

# 55. Placeholder Compatibility

La estrategia efectiva puede depender de:

```text
Dialect
+
Driver
```

porque el DBMS puede aceptar una sintaxis y el driver otra.

---

# 56. Dialect Syntax vs Driver Placeholder

Esto constituye una frontera especial.

Podrá existir una fase:

```text
Dialect SQL
      ↓
Driver Placeholder Adaptation
```

si la tecnología lo requiere.

Pero deberá ser explícita.

---

# 57. Operator Syntax

El dialecto podrá definir representación de operadores.

Ejemplos:

```text
=
<>
LIKE
ILIKE
REGEXP
JSON operators
array operators
```

---

# 58. Semantic Operator ≠ SQL Token

El AST deberá preferir:

```text
SemanticOperator
```

por ejemplo:

```text
CASE_INSENSITIVE_MATCH
```

El dialecto decide si puede representarlo como:

```sql
ILIKE
```

o mediante otra estrategia.

---

# 59. Operator Registry

Podrá existir:

```text
SemanticOperator
→
DialectOperatorRenderer
```

---

# 60. Unknown Operator

Si no existe representación válida:

```text
UnsupportedDialectFeatureException
```

o una estrategia de emulación previamente aprobada.

---

# 61. Silent Semantic Degradation

Estará prohibida.

Ejemplo:

```text
case-insensitive match
```

no deberá convertirse silenciosamente en:

```text
case-sensitive match
```

---

# 62. Function Syntax

El dialecto podrá representar funciones SQL.

Ejemplos:

```text
string length
substring
date extraction
JSON extraction
aggregation
```

---

# 63. Semantic Function ≠ Vendor Function Name

El Query AST podrá expresar:

```text
STRING_LENGTH(expression)
```

y el dialecto producir:

```text
LENGTH(...)
CHAR_LENGTH(...)
```

según corresponda.

---

# 64. Function Renderer

Conceptualmente:

```php
interface DialectFunctionRenderer
{
    public function render(
        FunctionExpression $expression,
        SqlCompilationContext $context
    ): SqlFragment;
}
```

---

# 65. Function Extension

Plugins podrán registrar funciones adicionales mediante extension points controlados.

---

# 66. Function Conflict

Dos extensiones no podrán sobrescribir silenciosamente el mismo identificador semántico.

---

# 67. DML Syntax

El dialecto podrá aportar reglas para:

```text
SELECT
INSERT
UPDATE
DELETE
MERGE-like constructs
UPSERT-like constructs
```

---

# 68. Dialect Does Not Build DML Semantics

El Compiler determina la estructura del statement.

El Dialect ayuda a representar las partes específicas.

---

# 69. SELECT

La estructura base puede ser compartida.

Las variantes podrán incluir:

```text
pagination
locking
distinct variants
vendor clauses
```

---

# 70. INSERT

Variaciones:

```text
DEFAULT VALUES
multi-row insert
returning
conflict handling
ignore syntax
```

---

# 71. UPDATE

Variaciones:

```text
UPDATE ... FROM
JOIN UPDATE
RETURNING
LIMIT
```

---

# 72. DELETE

Variaciones:

```text
DELETE ... USING
JOIN DELETE
RETURNING
LIMIT
```

---

# 73. UPSERT

El AST deberá expresar la semántica.

El dialecto podrá representar:

PostgreSQL:

```sql
ON CONFLICT ...
```

MySQL/MariaDB:

```sql
ON DUPLICATE KEY UPDATE ...
```

cuando la equivalencia semántica haya sido validada.

---

# 74. Syntax Similarity ≠ Semantic Equivalence

Dos construcciones parecidas no deberán considerarse equivalentes automáticamente.

---

# 75. RETURNING

El dialecto podrá proporcionar:

```text
ReturningSyntax
```

---

# 76. Returning Capability

La existencia de un renderer no prueba que el servidor concreto soporte la operación.

Se requerirá:

```text
CapabilitySnapshot
```

---

# 77. RETURNING Emulation

Si se permite emulación deberá declararse explícitamente.

```text
NATIVE
EMULATED
UNSUPPORTED
UNKNOWN
```

---

# 78. Emulation ≠ Native Support

Regla:

```text
EMULATED
≠
SUPPORTED_NATIVE
```

---

# 79. Pagination Syntax

El dialecto podrá representar:

```text
LIMIT/OFFSET
OFFSET/FETCH
TOP
ROW_NUMBER strategy
```

según DBMS.

---

# 80. Pagination Semantics

El Query Planner deberá determinar la estrategia semántica.

El dialecto sólo representa una estrategia válida.

---

# 81. Cursor Pagination

No pertenece al dialecto como algoritmo.

El dialecto sólo representa los predicates/order necesarios generados por Compiler.

---

# 82. Lock Syntax

Ejemplos:

```sql
FOR UPDATE
FOR SHARE
NOWAIT
SKIP LOCKED
```

---

# 83. Lock Semantics

El dialecto no deberá decidir si un lock es legal en el contexto transaccional.

---

# 84. Lock Capability

Deberá consultarse Capability System.

---

# 85. Lock Modifier

Un modifier como:

```text
NOWAIT
```

no deberá emitirse únicamente porque el dialecto conoce la palabra.

---

# 86. CTE Syntax

El dialecto podrá representar:

```text
WITH
WITH RECURSIVE
materialization modifiers
```

cuando estén disponibles.

---

# 87. Recursive CTE

Requerirá:

```text
AST semantics
+
Capability support
+
Dialect representation
```

---

# 88. Window Function Syntax

El dialecto podrá definir reglas para:

```text
OVER
PARTITION BY
ORDER BY
ROWS
RANGE
GROUPS
frame exclusions
```

---

# 89. Window Feature Granularity

No deberá existir únicamente:

```text
supportsWindowFunctions()
```

si el DBMS presenta soporte parcial significativo.

Podrán existir capabilities más específicas.

---

# 90. Set Operations

El dialecto podrá representar:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

según capabilities.

---

# 91. NULL Ordering

El dialecto podrá representar:

```text
NULLS FIRST
NULLS LAST
```

cuando exista sintaxis nativa.

---

# 92. NULL Ordering Emulation

Podrá requerir expresión adicional.

Esto deberá ser una estrategia explícita del Compiler/Dialect.

---

# 93. DDL Boundary

DDL es más complejo que DML.

El Dialect podrá aportar primitivas sintácticas para:

```text
CREATE
ALTER
DROP
RENAME
```

pero el:

```text
Schema Compiler
```

seguirá gobernando la compilación estructural.

---

# 94. Dialect ≠ Schema Compiler

Regla:

```text
Dialect
≠
SchemaCompiler
```

---

# 95. Type Declaration Syntax

El dialecto podrá colaborar con:

```text
Platform Type Mapping
```

para representar:

```text
VARCHAR(255)
NUMERIC(20, 6)
TIMESTAMP
JSONB
```

---

# 96. Logical Type ≠ SQL Declaration

La arquitectura existente se mantiene:

```text
PHP Type
≠
ORM Type
≠
Database Logical Type
≠
Platform Physical Type
≠
SQL Type Declaration
```

---

# 97. Dialect Type Renderer

Podrá recibir un tipo físico ya resuelto.

No deberá decidir por sí solo el tipo lógico de una propiedad ORM.

---

# 98. Schema Feature Syntax

Ejemplos:

```text
generated columns
partial indexes
include columns
check constraints
identity syntax
sequence syntax
```

---

# 99. Capability Before Syntax

El Compiler deberá verificar:

```text
Feature is semantically supported
```

antes de pedir al dialecto que la represente.

---

# 100. Syntax Availability

El dialecto podrá responder:

```text
I know how to render X
```

pero esto no equivale a:

```text
X is valid on this endpoint
```

---

# 101. Dialect Feature Descriptor

Conceptualmente:

```php
final readonly class DialectFeatureDescriptor
{
    public function __construct(
        public DialectFeatureId $id,
        public DialectFeatureMode $mode,
    ) {}
}
```

---

# 102. Dialect Feature Modes

Podrán incluir:

```text
NATIVE
EMULATABLE
UNAVAILABLE
```

pero la decisión efectiva continuará combinándose con capabilities.

---

# 103. Dialect Capability Relationship

Formalmente:

```text
CanRender(feature)
≠
CanExecute(feature)
```

---

# 104. Effective Support

Podrá expresarse como:

```text
EffectiveSupport(F)
=
SemanticCapability(F)
∧
DialectCanRepresent(F)
∧
DriverCanTransport(F)
```

cuando las tres dimensiones sean relevantes.

---

# 105. Compiler Integration

El Compiler utilizará interfaces pequeñas.

Ejemplo:

```text
SqlCompiler
 ├── IdentifierQuoter
 ├── PlaceholderStrategy
 ├── OperatorRenderer
 ├── FunctionRenderer
 ├── PaginationRenderer
 ├── LockRenderer
 └── ReturningRenderer
```

---

# 106. Avoid God Dialect

No deberá crearse una clase con cientos de métodos como única extensión.

Incorrecto:

```php
class GiantDialect
{
    public function everything(): mixed {}
}
```

---

# 107. Dialect Composition

Preferido:

```text
SqlDialect
├── IdentifierRules
├── PlaceholderStrategy
├── ExpressionSyntax
├── FunctionRegistry
├── DmlSyntax
├── PaginationSyntax
├── LockSyntax
└── SchemaSyntax
```

---

# 108. Immutable Dialect

Una instancia compilada deberá ser:

```text
immutable
stateless
shareable
```

siempre que sea posible.

---

# 109. No Current Connection State

El dialecto no deberá almacenar:

```text
current connection
current transaction
current query
current tenant
```

---

# 110. Persistent Runtime

Esto permitirá compartir el dialecto de forma segura entre requests en FrankenPHP.

---

# 111. Dialect Generation

Se podrá definir:

```text
DialectGeneration
```

para invalidar caches cuando cambie su configuración estructural.

---

# 112. Dialect Fingerprint

Conceptualmente:

```text
DialectFingerprint
=
H(
  DialectId
  + DialectVersion
  + StructuralConfiguration
  + RegisteredSyntaxExtensions
)
```

---

# 113. Fingerprint ≠ Server Fingerprint

No deberá incluir información dinámica del servidor salvo que forme parte de una variante compilada explícita.

---

# 114. Compiled Query Cache

Si SQL depende del dialecto:

```text
DialectFingerprint
```

deberá participar en el cache key.

---

# 115. Platform Capability Generation

También podrá participar:

```text
CapabilitySnapshotGeneration
```

si la forma compilada depende de capabilities efectivas.

---

# 116. Dialect Extensions

Plugins podrán extender:

```text
functions
operators
statement clauses
type declarations
feature renderers
```

mediante extension points públicos.

---

# 117. Extension ≠ Monkey Patch

No se permitirá como API soportada:

```text
reflection
method replacement
private property mutation
```

---

# 118. Extension Identity

Cada contribución tendrá identidad estable.

Ejemplo:

```text
dialect.function.vector_distance
```

---

# 119. Duplicate Extension

Será:

```text
conflict
```

salvo que el extension point permita composición explícita.

---

# 120. Extension Ordering

No dependerá del orden de Composer.

---

# 121. Extension Dependencies

Podrán declarar:

```text
requires
before
after
conflicts
```

sólo cuando el extension point soporte dicha semántica.

---

# 122. Plugin Integration

Ejemplo:

```text
acme/database-oracle
       │
       ▼
OracleDatabasePlugin
       │
       ▼
OracleDialectExtension
       │
       ▼
Dialect Registry
```

---

# 123. Custom DBMS Package

Un package completo podría proporcionar:

```text
Oracle Plugin
├── Oracle Driver
├── Oracle Platform
├── Oracle Dialect
├── Oracle Compiler Extensions
├── Oracle Schema Compiler
└── Oracle Capability Providers
```

---

# 124. Shared Dialect Base

Podrán existir implementaciones reutilizables.

Ejemplo:

```text
AbstractSqlDialect
      │
      ├── MySQLDialect
      └── MariaDBDialect
```

pero deberán evitar herencias que oculten diferencias importantes.

---

# 125. Composition Preferred

Cuando las diferencias sean complejas:

```text
composition
```

será preferida sobre jerarquías profundas.

---

# 126. Version-specific Syntax

Ejemplo conceptual:

```text
PostgreSQLDialect
      │
      ├── base syntax
      └── capability/version-conditioned renderers
```

---

# 127. Version Checks

Se evitarán checks dispersos:

```php
if ($version >= 150000) {
}
```

por todo el Compiler.

Preferido:

```text
Capability Snapshot
+
Dialect Feature Resolution
```

---

# 128. Version ≠ Capability

Regla heredada:

```text
Version
≠
Capability
```

---

# 129. Server Configuration

Incluso una versión compatible puede tener:

```text
extension missing
feature disabled
compatibility mode
```

Por eso la versión sola no basta.

---

# 130. Compatibility Modes

Algunos DBMS poseen modos de compatibilidad.

El dialecto podrá tener:

```text
DialectVariant
```

si el modo altera realmente la sintaxis.

---

# 131. DialectVariant

Ejemplo conceptual:

```text
DialectId: acme.database
Variant: oracle_compat
```

---

# 132. Variant ≠ New Platform Automatically

Una variante sintáctica no implica necesariamente una plataforma semántica distinta.

---

# 133. Dialect Configuration

La configuración deberá ser estructural y tipada.

Ejemplo:

```php
new DialectConfiguration(
    identifierQuoteMode: IdentifierQuoteMode::AUTO,
);
```

---

# 134. Runtime User Preference

No se permitirá cambiar arbitrariamente dialecto por query salvo API avanzada explícita.

---

# 135. One Connection Context

Una conexión deberá tener una resolución coherente de:

```text
Driver
Platform
Dialect
Capabilities
```

---

# 136. Connection Database Profile

Podrá consolidarse como:

```text
DatabaseEndpointProfile
├── Driver
├── Platform
├── Dialect
└── CapabilitySnapshot
```

---

# 137. Dialect Resolution Before Compilation

El Compiler deberá conocer el dialecto efectivo antes de producir SQL.

---

# 138. Dialect Switching Mid-query

Prohibido.

---

# 139. Dialect Validation

Durante bootstrap se validará:

```text
identity
descriptor
required syntax components
extension conflicts
configuration
compatibility
```

---

# 140. Runtime Validation

Durante compilation se validará:

```text
requested feature
capability
representability
```

---

# 141. Representability

Una operación semánticamente válida puede no ser representable en un dialecto.

```text
SemanticValidity
≠
DialectRepresentability
```

---

# 142. Unsupported Representation

Deberá producir un error explícito.

---

# 143. Emulation

Algunas operaciones podrán emularse.

Pero sólo si:

```text
semantic equivalence is proven
```

dentro del contrato requerido.

---

# 144. Emulation Cost

La estrategia podrá aportar:

```text
cost hint
complexity hint
performance warning
```

al Planner.

---

# 145. Dialect Does Not Choose Expensive Emulation Alone

El Planner deberá poder decidir si una emulación es aceptable.

---

# 146. Semantic Preservation

Toda representación deberá preservar:

```text
filter semantics
ordering semantics
null semantics
locking semantics
transaction assumptions
result shape
```

según aplique.

---

# 147. NULL Semantics

El dialecto deberá respetar diferencias entre:

```text
IS NULL
= NULL
```

y otras reglas SQL relevantes.

---

# 148. Boolean Semantics

No deberá asumirse que todos los DBMS poseen un tipo boolean nativo equivalente.

---

# 149. String Concatenation

Podrá variar:

```text
||
CONCAT(...)
```

---

# 150. Date Arithmetic

Podrá variar ampliamente.

Deberá representarse mediante operaciones semánticas y renderers especializados.

---

# 151. JSON Syntax

Ejemplos:

```text
-> 
->>
JSON_EXTRACT(...)
json_extract(...)
```

deberán permanecer encapsulados en el dialecto/compiler extension correspondiente.

---

# 152. JSON Semantics

La representación deberá distinguir:

```text
SQL NULL
JSON null
missing path
```

según el modelo definido en el Type/JSON Query System.

---

# 153. Full-text Search

La sintaxis podrá variar radicalmente.

No deberá introducirse como raw SQL en Query Builder.

---

# 154. Geographic Syntax

Igualmente:

```text
spatial functions/operators
```

deberán integrarse mediante extensiones semánticas.

---

# 155. Custom Operator Example

AST:

```text
VectorDistance(left, right)
```

PostgreSQL extension:

```sql
left <-> right
```

Otro DBMS:

```sql
VECTOR_DISTANCE(left, right)
```

---

# 156. Raw Expression Escape Hatch

El sistema conservará:

```text
RawExpression
```

como escape hatch controlado.

---

# 157. Raw Expression ≠ Dialect Extension

Si una operación es recurrente y estructuralmente soportada:

```text
custom semantic extension
```

será preferible a raw SQL repetido.

---

# 158. Raw SQL Portability

No se garantizará portabilidad de raw SQL.

---

# 159. Dialect Security

El dialecto deberá asumir que todos los identifiers/values provienen de estructuras previamente validadas.

Aun así deberá evitar APIs que incentiven concatenación insegura.

---

# 160. Identifier Injection

Un dialecto no deberá aceptar:

```php
quoteIdentifier($userInput)
```

como sustituto de un modelo de identifier validado.

---

# 161. Parameter Injection

Los valores continuarán utilizando Parameter Binding.

---

# 162. Function Name Injection

Nombres dinámicos de funciones deberán resolverse mediante:

```text
registry
allowlist
typed identifier
```

no concatenación arbitraria.

---

# 163. Operator Injection

Misma regla.

---

# 164. Dialect Diagnostics

El sistema deberá poder responder:

```text
Which dialect is active?
Which version?
Which family?
Which extensions?
Which renderer handles feature X?
Which capabilities are required?
Why was feature X rejected?
```

---

# 165. Dialect Inspector

Conceptualmente:

```php
interface DialectInspector
{
    public function inspect(
        SqlDialect $dialect
    ): DialectDiagnosticReport;
}
```

---

# 166. Explain Compilation

Podrá existir una herramienta de desarrollo:

```text
Query AST
  ↓
Compilation explanation
  ↓
Dialect decisions
```

---

# 167. Example Diagnostic

```text
Feature: CASE_INSENSITIVE_MATCH

Semantic support: yes
Platform capability: yes
Dialect renderer: PostgreSQLIlikeRenderer
Representation: ILIKE
Emulation: no
```

---

# 168. Diagnostics ≠ Raw Secrets

Los diagnostics no deberán exponer valores sensibles.

---

# 169. Dialect Telemetry

Generalmente el dialecto será una capa pura y no requerirá telemetría operacional abundante.

Podrán medirse:

```text
compile duration
emulation usage
unsupported feature count
```

en el Compiler.

---

# 170. Dialect Events

No será necesario emitir eventos por cada token generado.

---

# 171. Event Overhead

La extensibilidad no deberá convertir SQL generation en un pipeline de eventos dinámicos costoso.

---

# 172. Compile-time Dispatch

Los renderers deberán resolverse durante:

```text
bootstrap
graph compilation
compiler construction
```

cuando sea posible.

---

# 173. No Global Plugin Scan

Nunca:

```php
foreach ($plugins as $plugin) {
    if ($plugin->canRender($node)) {
        // ...
    }
}
```

por cada AST node.

---

# 174. Dispatch Table

Preferido:

```text
AST Node Type
+
Semantic Operation
→
Compiled Renderer
```

---

# 175. Renderer Registry

Conceptualmente:

```php
final class DialectRendererRegistry
{
    public function rendererFor(
        DialectFeatureId $feature
    ): DialectRenderer;
}
```

---

# 176. Frozen Renderer Registry

Después de bootstrap:

```text
RendererRegistry
→
FROZEN
```

---

# 177. Renderer State

Los renderers deberán ser:

```text
stateless
immutable
```

siempre que sea posible.

---

# 178. Request Isolation

No deberán guardar:

```text
last query
last parameter
current tenant
current connection
```

como propiedades compartidas.

---

# 179. Thread/Coroutine Safety

Un dialecto inmutable facilita compatibilidad futura con:

```text
FrankenPHP workers
RoadRunner
OpenSwoole coroutines
parallel compilation
```

---

# 180. Dialect Testing Architecture

Todo dialecto deberá tener:

```text
unit tests
golden compilation tests
semantic equivalence tests
capability-conditioned tests
integration tests
conformance tests
security tests
```

---

# 181. Unit Tests

Adecuados para:

```text
identifier quoting
placeholder generation
operator rendering
function rendering
pagination fragments
locking fragments
```

---

# 182. Golden Tests

Podrán validar:

```text
AST
→
expected SQL
```

pero deberán utilizarse cuidadosamente.

---

# 183. Golden SQL ≠ Runtime Correctness

Una cadena SQL esperada no demuestra que el DBMS la acepte ni que tenga la semántica esperada.

---

# 184. Integration Testing

Las características importantes deberán ejecutarse contra el DBMS real.

---

# 185. Dialect Conformance

La suite podrá validar:

```text
identifier behavior
parameter placeholders
basic SELECT
INSERT
UPDATE
DELETE
pagination
locking
CTE
window
returning
schema syntax
```

según capabilities.

---

# 186. Capability-conditioned Tests

Si:

```text
RETURNING = UNSUPPORTED
```

no se deberá exigir el test nativo.

Pero deberá probarse que:

```text
compiler rejects/emulates correctly
```

según política.

---

# 187. Conformance ≠ Same SQL

Distintos dialectos pueden generar SQL distinto y cumplir la misma semántica.

---

# 188. Semantic Test

Preferido:

```text
same Query Model
   ↓
different dialect SQL
   ↓
real DBMS
   ↓
equivalent expected result
```

cuando la característica deba ser portable.

---

# 189. MySQL and MariaDB Testing

Deberán probarse por separado.

---

# 190. PostgreSQL Testing

Deberá probarse contra versiones soportadas cuando una sintaxis dependa de capabilities/versiones.

---

# 191. SQLite Testing

Deberá validar su dialecto real, no utilizarse como sustituto universal.

---

# 192. Custom Dialect Test Package

VoltStack podrá proporcionar:

```text
Testing/Dialect/
├── DialectContractTest
├── IdentifierDialectTest
├── PlaceholderDialectTest
├── DmlDialectTest
├── PaginationDialectTest
├── LockDialectTest
├── ReturningDialectTest
└── SchemaDialectTest
```

---

# 193. Example Custom Dialect

```php
final class AcmeSqlDialect implements SqlDialect
{
    public function id(): DialectId
    {
        return new DialectId('acme.customsql');
    }

    public function identifierRules(): IdentifierRules
    {
        return new AcmeIdentifierRules();
    }

    public function placeholders(): ParameterPlaceholderStrategy
    {
        return new AcmePlaceholderStrategy();
    }
}
```

---

# 194. Example Function Extension

```php
final class VectorDistanceDialectExtension
{
    public function register(
        DialectExtensionRegistrar $registrar
    ): void {
        $registrar->function(
            SemanticFunction::VECTOR_DISTANCE,
            new AcmeVectorDistanceRenderer()
        );
    }
}
```

---

# 195. Example Renderer

```php
final readonly class AcmeVectorDistanceRenderer
    implements DialectFunctionRenderer
{
    public function render(
        FunctionExpression $expression,
        SqlCompilationContext $context
    ): SqlFragment {
        // Render using the dialect syntax.
    }
}
```

---

# 196. What a Dialect Must Not Do

Incorrecto:

```php
final class BadDialect
{
    public function connect(): PDO {}
    public function execute(string $sql): array {}
    public function persist(object $entity): void {}
    public function optimize(QueryAst $ast): QueryAst {}
}
```

Estas responsabilidades pertenecen a otras capas.

---

# 197. Dialect Package Structure

Propuesta para terceros:

```text
src/
├── Plugin/
│   └── AcmeDatabasePlugin.php
├── Extension/
│   └── AcmeDialectExtension.php
├── Dialect/
│   ├── AcmeDialect.php
│   ├── AcmeDialectDescriptor.php
│   ├── AcmeDialectFactory.php
│   ├── Identifier/
│   ├── Expression/
│   ├── Function/
│   ├── Dml/
│   ├── Schema/
│   └── Renderer/
└── Testing/
```

---

# 198. Core Directory Structure

Propuesta:

```text
src/Quantum/Database/Dialect/
├── Contract/
│   ├── SqlDialect.php
│   ├── DialectFactory.php
│   ├── DialectRenderer.php
│   └── DialectExtension.php
│
├── Identity/
│   ├── DialectId.php
│   ├── DialectVersion.php
│   ├── DialectFamily.php
│   ├── DialectGeneration.php
│   └── DialectFingerprint.php
│
├── Descriptor/
│   ├── DialectDescriptor.php
│   ├── DialectFeatureDescriptor.php
│   └── DialectVariant.php
│
├── Registry/
│   ├── DialectRegistry.php
│   ├── MutableDialectRegistry.php
│   └── FrozenDialectRegistry.php
│
├── Resolution/
│   ├── DialectResolver.php
│   ├── DialectSelection.php
│   └── DialectCompatibilityResolver.php
│
├── Identifier/
│   ├── IdentifierRules.php
│   ├── IdentifierQuoter.php
│   ├── QualifiedIdentifierRenderer.php
│   └── KeywordRegistry.php
│
├── Parameter/
│   ├── ParameterPlaceholderStrategy.php
│   ├── PositionalPlaceholderStrategy.php
│   ├── NumberedPlaceholderStrategy.php
│   └── NamedPlaceholderStrategy.php
│
├── Expression/
│   ├── OperatorRenderer.php
│   ├── LiteralRenderer.php
│   └── ExpressionRendererRegistry.php
│
├── Function/
│   ├── DialectFunctionRenderer.php
│   └── DialectFunctionRegistry.php
│
├── Dml/
│   ├── SelectSyntax.php
│   ├── InsertSyntax.php
│   ├── UpdateSyntax.php
│   ├── DeleteSyntax.php
│   └── UpsertSyntax.php
│
├── Feature/
│   ├── PaginationSyntax.php
│   ├── ReturningSyntax.php
│   ├── LockSyntax.php
│   ├── CteSyntax.php
│   ├── WindowSyntax.php
│   └── SetOperationSyntax.php
│
├── Schema/
│   ├── SchemaSyntax.php
│   ├── TypeDeclarationRenderer.php
│   └── ConstraintSyntax.php
│
├── Extension/
│   ├── DialectExtensionRegistrar.php
│   ├── DialectRendererRegistry.php
│   └── DialectExtensionCompiler.php
│
├── Diagnostics/
│   ├── DialectInspector.php
│   └── DialectDiagnosticReport.php
│
└── Exception/
    ├── DialectException.php
    ├── DuplicateDialectException.php
    ├── AmbiguousDialectException.php
    ├── UnsupportedDialectFeatureException.php
    ├── DialectConfigurationException.php
    └── DialectExtensionConflictException.php
```

---

# 199. Architectural Invariants

## DB-CUSTOM-DIALECT-001

Dialect ≠ Compiler.

## DB-CUSTOM-DIALECT-002

Dialect ≠ Driver.

## DB-CUSTOM-DIALECT-003

Dialect ≠ Platform.

## DB-CUSTOM-DIALECT-004

Dialect ≠ Capability System.

## DB-CUSTOM-DIALECT-005

Dialect ≠ Query AST.

## DB-CUSTOM-DIALECT-006

Dialect ≠ Query Builder.

## DB-CUSTOM-DIALECT-007

Dialect ≠ ORM.

## DB-CUSTOM-DIALECT-008

Dialect ≠ Schema Compiler.

## DB-CUSTOM-DIALECT-009

DialectId será estable.

## DB-CUSTOM-DIALECT-010

DialectId ≠ FQCN.

## DB-CUSTOM-DIALECT-011

MySQL Dialect ≠ MariaDB Dialect.

## DB-CUSTOM-DIALECT-012

DialectVersion ≠ ServerVersion.

## DB-CUSTOM-DIALECT-013

Descriptor ≠ Dialect Instance.

## DB-CUSTOM-DIALECT-014

Duplicate DialectId será error.

## DB-CUSTOM-DIALECT-015

Driver no será autoridad del dialecto.

## DB-CUSTOM-DIALECT-016

Dialect Context no contendrá state mutable de request.

## DB-CUSTOM-DIALECT-017

Identifier ≠ Raw String.

## DB-CUSTOM-DIALECT-018

Quoting ≠ Validation.

## DB-CUSTOM-DIALECT-019

Qualified identifier será estructurado.

## DB-CUSTOM-DIALECT-020

Dynamic values serán parameterized por defecto.

## DB-CUSTOM-DIALECT-021

Placeholder syntax ≠ Parameter binding.

## DB-CUSTOM-DIALECT-022

Semantic operator ≠ SQL token.

## DB-CUSTOM-DIALECT-023

Semantic function ≠ Vendor function name.

## DB-CUSTOM-DIALECT-024

Silent semantic degradation estará prohibida.

## DB-CUSTOM-DIALECT-025

Dialect no construirá semántica DML.

## DB-CUSTOM-DIALECT-026

Syntax similarity ≠ Semantic equivalence.

## DB-CUSTOM-DIALECT-027

Returning renderer ≠ Returning capability.

## DB-CUSTOM-DIALECT-028

Emulated ≠ Native.

## DB-CUSTOM-DIALECT-029

Pagination algorithm ≠ Pagination syntax.

## DB-CUSTOM-DIALECT-030

Lock syntax ≠ Lock legality.

## DB-CUSTOM-DIALECT-031

Recursive CTE requiere capability + representation.

## DB-CUSTOM-DIALECT-032

Window support podrá ser granular.

## DB-CUSTOM-DIALECT-033

Logical Type ≠ SQL Type Declaration.

## DB-CUSTOM-DIALECT-034

Capability será comprobada antes de feature syntax cuando corresponda.

## DB-CUSTOM-DIALECT-035

CanRender ≠ CanExecute.

## DB-CUSTOM-DIALECT-036

Compiler usará Dialect; Dialect no sustituirá Compiler.

## DB-CUSTOM-DIALECT-037

God Dialect deberá evitarse.

## DB-CUSTOM-DIALECT-038

Dialect composition será preferida para capacidades complejas.

## DB-CUSTOM-DIALECT-039

Dialect runtime será inmutable cuando sea posible.

## DB-CUSTOM-DIALECT-040

Dialect no almacenará current connection.

## DB-CUSTOM-DIALECT-041

Dialect no almacenará current transaction.

## DB-CUSTOM-DIALECT-042

Dialect no almacenará current tenant.

## DB-CUSTOM-DIALECT-043

Dialect fingerprint será determinista.

## DB-CUSTOM-DIALECT-044

Compiled query cache será dialect-aware cuando corresponda.

## DB-CUSTOM-DIALECT-045

Dialect extension ≠ Monkey Patch.

## DB-CUSTOM-DIALECT-046

Duplicate renderer no será last-one-wins.

## DB-CUSTOM-DIALECT-047

Plugin order no definirá renderer precedence.

## DB-CUSTOM-DIALECT-048

Version ≠ Capability.

## DB-CUSTOM-DIALECT-049

Compatibility mode podrá alterar dialect variant.

## DB-CUSTOM-DIALECT-050

Dialect switching mid-query estará prohibido.

## DB-CUSTOM-DIALECT-051

Semantic validity ≠ Dialect representability.

## DB-CUSTOM-DIALECT-052

Unsupported representation será explícita.

## DB-CUSTOM-DIALECT-053

Emulation requerirá equivalencia semántica aceptable.

## DB-CUSTOM-DIALECT-054

Dialect no decidirá unilateralmente emulación costosa.

## DB-CUSTOM-DIALECT-055

NULL semantics deberán preservarse.

## DB-CUSTOM-DIALECT-056

Boolean semantics no se asumirán universales.

## DB-CUSTOM-DIALECT-057

JSON semantics preservarán SQL NULL/JSON null/missing.

## DB-CUSTOM-DIALECT-058

RawExpression ≠ Dialect Extension.

## DB-CUSTOM-DIALECT-059

Raw SQL no tendrá portabilidad garantizada.

## DB-CUSTOM-DIALECT-060

Identifier quoting no sustituirá identifier validation.

## DB-CUSTOM-DIALECT-061

Function names dinámicos deberán validarse.

## DB-CUSTOM-DIALECT-062

Operator names dinámicos deberán validarse.

## DB-CUSTOM-DIALECT-063

Diagnostics no expondrán datos sensibles.

## DB-CUSTOM-DIALECT-064

No se emitirán eventos por token.

## DB-CUSTOM-DIALECT-065

Renderer dispatch se compilará cuando sea posible.

## DB-CUSTOM-DIALECT-066

No habrá plugin scan por AST node.

## DB-CUSTOM-DIALECT-067

Renderer Registry será congelable.

## DB-CUSTOM-DIALECT-068

Renderer shared state será inmutable.

## DB-CUSTOM-DIALECT-069

Golden SQL ≠ Runtime Correctness.

## DB-CUSTOM-DIALECT-070

Conformance ≠ Same SQL.

## DB-CUSTOM-DIALECT-071

Real DBMS será necesario para evidencia de ejecución.

## DB-CUSTOM-DIALECT-072

Capability-conditioned tests validarán soporte y rechazo.

## DB-CUSTOM-DIALECT-073

MySQL y MariaDB se probarán por separado.

## DB-CUSTOM-DIALECT-074

SQLite no será prueba universal.

## DB-CUSTOM-DIALECT-075

Dialect no abrirá conexiones.

## DB-CUSTOM-DIALECT-076

Dialect no ejecutará statements.

## DB-CUSTOM-DIALECT-077

Dialect no hidratará entidades.

## DB-CUSTOM-DIALECT-078

Dialect no persistirá entidades.

## DB-CUSTOM-DIALECT-079

Dialect no optimizará Query AST como responsabilidad principal.

## DB-CUSTOM-DIALECT-080

Dialect no determinará transaction outcome.

## DB-CUSTOM-DIALECT-081

Dialect no administrará connection pool.

## DB-CUSTOM-DIALECT-082

Dialect no resolverá tenant.

## DB-CUSTOM-DIALECT-083

Dialect no decidirá shard.

## DB-CUSTOM-DIALECT-084

Dialect no será authority de server capabilities.

## DB-CUSTOM-DIALECT-085

Dialect configuration será estructural.

## DB-CUSTOM-DIALECT-086

Dialect extensions tendrán identidad.

## DB-CUSTOM-DIALECT-087

Dialect extension conflicts serán visibles.

## DB-CUSTOM-DIALECT-088

Plugin System podrá distribuir dialectos sin definir sus contratos.

## DB-CUSTOM-DIALECT-089

Extension Architecture gobernará registros.

## DB-CUSTOM-DIALECT-090

Custom dialect preservará Database dependency direction.

## DB-CUSTOM-DIALECT-091

Driver placeholder requirements se modelarán explícitamente.

## DB-CUSTOM-DIALECT-092

Dialect-specific SQL no aparecerá disperso en ORM.

## DB-CUSTOM-DIALECT-093

Dialect-specific SQL no aparecerá disperso en Query Builder.

## DB-CUSTOM-DIALECT-094

Dialect-specific SQL no aparecerá disperso en EntityManager.

## DB-CUSTOM-DIALECT-095

Dialect-specific SQL no aparecerá disperso en Connection Manager.

## DB-CUSTOM-DIALECT-096

Type declaration renderer recibirá tipos ya resueltos.

## DB-CUSTOM-DIALECT-097

Dialect podrá aportar syntax knowledge, no database truth.

## DB-CUSTOM-DIALECT-098

Persistent runtime no mutará Dialect Registry.

## DB-CUSTOM-DIALECT-099

Custom Dialect preservará semantic correctness.

## DB-CUSTOM-DIALECT-100

Custom Dialect System deberá ser determinista.

---

# 200. Anti-patrones

## 200.1 Dialect ejecutando SQL

Incorrecto.

---

## 200.2 Dialect abriendo PDO

Incorrecto.

---

## 200.3 Dialect construyendo Query AST

Incorrecto.

---

## 200.4 Dialect hidratando entidades

Incorrecto.

---

## 200.5 Dialect decidiendo retries

Incorrecto.

---

## 200.6 Dialect asumiendo capability por versión

Incorrecto.

---

## 200.7 `if ($vendor === ...)` disperso por todo Database

Incorrecto.

---

## 200.8 Query Builder generando SQL vendor-specific

Incorrecto.

---

## 200.9 ORM concatenando sintaxis específica

Incorrecto.

---

## 200.10 Quoting de user input como protección universal

Incorrecto.

---

## 200.11 Concatenar valores dinámicos

Incorrecto.

---

## 200.12 Registrar funciones durante una query

Incorrecto.

---

## 200.13 Mutar Dialect Registry en worker activo

Incorrecto.

---

## 200.14 Last renderer wins

Incorrecto.

---

## 200.15 Una clase gigante para toda la semántica del DBMS

Debe evitarse.

---

## 200.16 Usar raw SQL para toda diferencia entre DBMS

Incorrecto.

---

## 200.17 Confundir placeholder con binding

Incorrecto.

---

## 200.18 Confundir dialecto con plataforma

Incorrecto.

---

## 200.19 Confundir dialecto con compiler

Incorrecto.

---

## 200.20 Probar sólo la cadena SQL

Insuficiente para demostrar semántica real.

---

# 201. Modelo formal

Sea:

```text
D
```

un dialecto.

Su responsabilidad será:

```text
Responsibilities(D)
⊆
{
  IdentifierRepresentation,
  PlaceholderRepresentation,
  OperatorRepresentation,
  FunctionRepresentation,
  StatementSyntax,
  FeatureSyntax,
  TypeDeclarationSyntax
}
```

y deberá cumplirse:

```text
Responsibilities(D)
∩
{
  ConnectionManagement,
  QueryExecution,
  ORM,
  QueryOptimization,
  TransactionManagement,
  CapabilityAuthority
}
=
∅
```

---

# 202. Representability

Para una operación semántica `O`:

```text
Representable(O, D)
=
HasRenderer(D, O)
∧
RendererSemanticsCompatible(D, O)
```

Pero ejecutabilidad requiere más:

```text
Executable(O)
=
Representable(O, D)
∧
CapabilitiesSatisfied(O)
∧
DriverRequirementsSatisfied(O)
```

---

# 203. Dialect Resolution

Formalmente:

```text
ResolveDialect(
    Platform,
    Configuration,
    AvailableDialects
)
→
ExactlyOneDialect
```

Si:

```text
0 candidates
```

resultado:

```text
UnsupportedDialect
```

Si:

```text
>1 candidates
```

resultado:

```text
AmbiguousDialect
```

---

# 204. Cache Identity

Para SQL compilado:

```text
CompiledSqlCacheKey
=
H(
    SemanticQueryFingerprint
    + CompilerGeneration
    + DialectFingerprint
    + RelevantCapabilityGeneration
)
```

cuando esas dimensiones afecten el resultado.

---

# 205. Emulation Rule

Sea `F` una feature.

```text
UseEmulation(F)
```

sólo podrá ser verdadero si:

```text
NativeSupport(F) = false
∧
EmulationAvailable(F)
∧
SemanticEquivalenceAcceptable(F)
∧
PlannerPolicyAllows(F)
```

---

# 206. Arquitectura consolidada

```text
                     Query Model / AST
                            │
                            ▼
                     Semantic Engine
                            │
                            ▼
                        Optimizer
                            │
                            ▼
                         Planner
                            │
                            ▼
                      SQL Compiler
                            │
             ┌──────────────┼───────────────┐
             ▼              ▼               ▼
        Platform       Capabilities      Dialect
             │              │               │
             │              │       ┌───────┼────────┐
             │              │       ▼       ▼        ▼
             │              │  Identifiers Functions Operators
             │              │       │       │        │
             │              │       ├───────┼────────┤
             │              │       ▼       ▼        ▼
             │              │  Pagination Locking Returning
             │              │       │       │        │
             └──────────────┼───────┴───────┴────────┘
                            ▼
                    Compiled Statement
                            │
                            ▼
                          Driver
                            │
                            ▼
                       DB Protocol
                            │
                            ▼
                         DBMS
```

---

# 207. Estrategia V1

La V1 deberá estabilizar primero:

```text
DialectId
DialectDescriptor
DialectRegistry
DialectResolver
DialectFactory
IdentifierRules
IdentifierQuoter
PlaceholderStrategy
OperatorRenderer
FunctionRenderer
PaginationSyntax
LockSyntax
ReturningSyntax
Basic DML Syntax
Schema Syntax Contracts
DialectFingerprint
Conformance Tests
```

---

# 208. Evolución posterior

Podrán añadirse:

```text
advanced dialect variants
vendor compatibility modes
dynamic extension packages
advanced JSON operators
vector syntax
advanced geographic syntax
temporal SQL extensions
graph-query extensions
advanced MERGE
vendor optimizer hints
```

sin romper el modelo semántico central.

---

# 209. Regla final

> **El Custom Dialect System será la capa responsable de expresar cómo una operación ya modelada semánticamente puede representarse en una variante concreta de SQL. El dialecto conocerá sintaxis, pero no será dueño de la consulta, la conexión, la ejecución ni la verdad operacional del DBMS.**

Por tanto:

```text
Dialect
≠
Compiler
```

```text
Dialect
≠
Driver
```

```text
Dialect
≠
Platform
```

```text
Dialect
≠
Capability System
```

```text
Dialect
≠
Query AST
```

```text
Dialect
≠
Query Builder
```

```text
Dialect
≠
ORM
```

```text
Dialect
≠
Schema Compiler
```

```text
Identifier
≠
Raw String
```

```text
Quoting
≠
Validation
```

```text
Placeholder Syntax
≠
Parameter Binding
```

```text
Semantic Operator
≠
SQL Token
```

```text
Semantic Function
≠
Vendor Function Name
```

```text
Syntax Similarity
≠
Semantic Equivalence
```

```text
CanRender
≠
CanExecute
```

```text
Emulated
≠
Native
```

```text
Version
≠
Capability
```

```text
Semantic Validity
≠
Dialect Representability
```

```text
Golden SQL
≠
Runtime Correctness
```

```text
RawExpression
≠
Dialect Extension
```

y finalmente:

```text
Safe Custom Dialect
=
Stable Identity
+
Explicit Syntax Contracts
+
Identifier Safety
+
Parameter Placeholder Strategy
+
Semantic Operator Mapping
+
Semantic Function Mapping
+
DML Representation
+
Feature-specific Syntax
+
Schema Syntax Boundaries
+
Capability Awareness
+
Compiler Integration
+
Immutable Runtime
+
Deterministic Extension Resolution
+
Conformance Testing
```

---

# 210. Siguiente documento

```text
298_DATABASE_CUSTOM_COMPILER_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura mediante la cual VoltStack permitirá añadir o reemplazar **compiladores especializados** para transformar estructuras semánticas y planes validados en representaciones ejecutables sin convertir el Compiler en Query Builder, Optimizer, Planner, Driver o Execution Engine.

La arquitectura deberá cubrir:

```text
Custom Compiler System
│
├── Compiler Identity
├── Compiler Descriptor
├── Compiler Registry
├── Compiler Factory
├── Compiler Resolution
├── Compilation Context
├── Compilation Input
├── Compilation Output
├── AST/Plan Visitors
├── Statement Compilers
├── Expression Compilers
├── Renderer Dispatch
├── Dialect Integration
├── Platform Integration
├── Capability Validation
├── Parameter Compilation
├── Compilation Diagnostics
├── Compiler Extensions
├── Compiler Cache Identity
├── Deterministic Compilation
├── Conformance Testing
└── Plugin Integration
```

manteniendo como principio central:

> **Un custom compiler transformará estructuras semánticas previamente validadas en representaciones ejecutables utilizando Platform, Capabilities y Dialect; no deberá decidir intención de negocio, ejecutar consultas, administrar conexiones ni absorber las responsabilidades del Optimizer o Query Planner.**