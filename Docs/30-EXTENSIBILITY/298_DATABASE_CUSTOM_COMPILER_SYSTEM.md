# 298_DATABASE_CUSTOM_COMPILER_SYSTEM.md

# VoltStack Quantum Database
## Custom Compiler System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 298 — Custom Compiler System  
**Bloque:** 30 — Extensibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `297_DATABASE_CUSTOM_DIALECT_SYSTEM.md`  
**Siguiente documento:** `299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial para crear, registrar, resolver, extender y validar **compiladores personalizados** dentro de:

```text
VoltStack/Quantum/Database
```

El Custom Compiler System permitirá incorporar nuevos compiladores para:

```text
queries
DML
DDL
expressions
vendor-specific features
custom query extensions
future database paradigms
```

sin modificar el núcleo de Database ni romper las fronteras entre:

```text
Query Builder
AST
Semantic Engine
Optimizer
Planner
Compiler
Dialect
Driver
Execution Engine
```

La regla central será:

> **Un custom compiler transforma una representación semántica previamente validada y planificada en una representación ejecutable utilizando el dialecto, la plataforma, las capabilities y el contexto de compilación; no determina intención de negocio, no ejecuta consultas, no administra conexiones y no sustituye al Optimizer ni al Planner.**

Formalmente:

```text
Custom Compiler
=
Identity
+
Descriptor
+
Input Contract
+
Compilation Context
+
Compiler Pipeline
+
Node Dispatch
+
Statement Compilers
+
Expression Compilers
+
Dialect Integration
+
Capability Validation
+
Parameter Compilation
+
Output Contract
+
Diagnostics
+
Conformance
```

pero:

```text
Custom Compiler
≠
Query Builder
≠
Semantic Analyzer
≠
Optimizer
≠
Planner
≠
Dialect
≠
Driver
≠
Executor
```

---

# 2. Objetivos

El sistema deberá permitir:

1. incorporar compiladores sin modificar Database Core;
2. registrar compiladores mediante Extension Architecture;
3. distribuirlos mediante Plugin System;
4. identificar compiladores de forma estable;
5. resolver el compilador correcto;
6. compilar Query Models/AST/Plans;
7. producir representaciones ejecutables;
8. integrar dialectos;
9. integrar Platform;
10. consultar capabilities;
11. compilar parámetros;
12. compilar statements;
13. compilar expresiones;
14. soportar DML;
15. soportar DDL mediante compiladores especializados;
16. extender nodos;
17. extender operadores;
18. extender funciones;
19. mantener compilación determinista;
20. soportar compiled query caching;
21. generar diagnostics;
22. preservar seguridad;
23. funcionar en runtimes persistentes;
24. soportar conformance testing;
25. permitir futuros targets distintos de SQL cuando la arquitectura lo justifique.

---

# 3. Posición arquitectónica

La compilación se encuentra después de las decisiones semánticas y de planificación:

```text
Developer API
    ↓
Query Builder
    ↓
Query Model / AST
    ↓
Semantic Analysis
    ↓
Optimizer
    ↓
Planner
    ↓
Compilation Input
    ↓
Compiler
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

# 4. Compiler ≠ Query Builder

El Query Builder expresa intención.

Ejemplo:

```php
$query
    ->from('users')
    ->where('active', true);
```

El Compiler no ofrece esta API.

Por tanto:

```text
Query Builder
≠
Compiler
```

---

# 5. Compiler ≠ Semantic Analyzer

El Semantic Analyzer determina:

```text
symbol resolution
type compatibility
relationship meaning
expression validity
```

El Compiler deberá recibir estructuras semánticamente válidas.

---

# 6. Compiler ≠ Optimizer

El Optimizer transforma una consulta válida en una forma semánticamente equivalente potencialmente mejor.

El Compiler representa la forma seleccionada.

```text
Optimizer
→ what equivalent form should be used?

Compiler
→ how is that form represented?
```

---

# 7. Compiler ≠ Planner

El Planner decide:

```text
strategy
logical/physical plan
execution approach
emulation strategy where applicable
```

El Compiler no deberá reconstruir estas decisiones.

---

# 8. Compiler ≠ Dialect

El Compiler entiende la estructura a compilar.

El Dialect entiende reglas sintácticas.

```text
Compiler
uses
Dialect
```

No:

```text
Compiler
=
Dialect
```

---

# 9. Compiler ≠ Platform

Platform representa conocimiento del DBMS.

Compiler consume ese conocimiento.

---

# 10. Compiler ≠ Capability System

El Compiler consulta capacidades.

No inventa capacidades.

---

# 11. Compiler ≠ Driver

El Compiler genera comandos.

El Driver los transporta.

```text
Compiler
→ CompiledCommand
→ Driver
```

---

# 12. Compiler ≠ Executor

El Compiler no ejecuta.

Nunca:

```php
$compiler->compileAndExecute($query);
```

como contrato arquitectónico base.

---

# 13. Arquitectura general

```text
                    Compilation Request
                           │
                           ▼
                    Compiler Resolver
                           │
                           ▼
                    Compiler Registry
                           │
                           ▼
                    Compiler Factory
                           │
                           ▼
                    Custom Compiler
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         Statements    Expressions   Parameters
              │            │            │
              └────────────┼────────────┘
                           ▼
                        Dialect
                           │
                    Platform/Capabilities
                           │
                           ▼
                  Compilation Output
                           │
                           ▼
                    Compiled Command
                           │
                           ▼
                    Execution Engine
```

---

# 14. Compiler Identity

Todo compiler deberá poseer:

```text
CompilerId
```

estable.

Ejemplos:

```text
sql.default
sql.mysql
sql.postgresql
schema.mysql
schema.postgresql

acme.oracle.query
acme.oracle.schema
acme.analytics
```

---

# 15. CompilerId

Conceptualmente:

```php
final readonly class CompilerId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 16. CompilerId ≠ FQCN

Nunca se utilizará la clase PHP como identidad durable principal.

```text
CompilerId
≠
ImplementationClass
```

---

# 17. Compiler Kind

Podrá existir:

```text
CompilerKind
```

con valores como:

```text
QUERY
SCHEMA
EXPRESSION
COMMAND
SPECIALIZED
```

---

# 18. Query Compiler ≠ Schema Compiler

Aunque compartan infraestructura:

```text
QuerySqlCompiler
≠
SchemaSqlCompiler
```

porque sus inputs y garantías son diferentes.

---

# 19. Compiler Version

Cada compiler podrá declarar:

```text
CompilerVersion
```

---

# 20. CompilerVersion ≠ DialectVersion

Son dimensiones distintas.

---

# 21. CompilerVersion ≠ ServerVersion

También:

```text
CompilerVersion
≠
DatabaseServerVersion
```

---

# 22. Compiler Descriptor

Conceptualmente:

```php
final readonly class CompilerDescriptor
{
    public function __construct(
        public CompilerId $id,
        public CompilerVersion $version,
        public CompilerKind $kind,
        public CompilationTarget $target,
        public CompilerFeatureSet $features,
    ) {}
}
```

---

# 23. Descriptor ≠ Compiler Instance

El descriptor será metadata estructural.

No deberá contener estado de una compilación activa.

---

# 24. Compilation Target

Inicialmente:

```text
SQL
```

será el target principal.

Pero la arquitectura podrá contemplar:

```text
SQL
NATIVE_COMMAND
PROTOCOL_COMMAND
CUSTOM
```

para extensibilidad futura.

---

# 25. SQL First

La V1 deberá optimizarse para SQL sin sobre-generalizar innecesariamente.

---

# 26. Compiler Registry

Resolverá:

```text
CompilerId
→
CompilerFactory
```

---

# 27. Registry Lifecycle

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

# 28. Duplicate CompilerId

Será error:

```text
DuplicateCompilerException
```

Nunca:

```text
last compiler wins
```

---

# 29. Compiler Factory

Contrato conceptual:

```php
interface CompilerFactory
{
    public function id(): CompilerId;

    public function create(
        CompilerConstructionContext $context
    ): DatabaseCompiler;
}
```

---

# 30. Compiler Construction Context

Podrá contener:

```text
Dialect
Platform
Compiler Extensions
Renderer Registries
Static Configuration
```

pero no:

```text
current transaction
current connection
current entity
current request mutable state
```

---

# 31. Compiler Resolution

La resolución podrá considerar:

```text
CompilationTarget
CompilerKind
Platform
Dialect
Explicit Configuration
Registered Extensions
```

---

# 32. Explicit Compiler Selection

Podrá configurarse:

```php
'compiler' => 'acme.oracle.query',
```

para escenarios avanzados.

---

# 33. Automatic Compiler Resolution

Normalmente:

```text
Platform
+
Dialect
+
Compilation Kind
→
Compiler
```

---

# 34. Ambiguous Compiler

Si más de un compiler es igualmente válido:

```text
AmbiguousCompilerException
```

---

# 35. Missing Compiler

Si no existe:

```text
CompilerNotFoundException
```

---

# 36. Compiler Resolution ≠ Capability Resolution

Seleccionar un compiler no significa que todas sus features puedan ejecutarse en el endpoint.

---

# 37. Compilation Input

El input deberá ser explícito.

Conceptualmente:

```php
interface CompilationInput
{
}
```

Subtipos:

```text
QueryCompilationInput
SchemaCompilationInput
ExpressionCompilationInput
CommandCompilationInput
```

---

# 38. Raw Query Builder ≠ Compilation Input

Preferentemente el compiler no recibirá directamente el mutable Query Builder.

El flujo será:

```text
Query Builder
   ↓
Normalized Query Model / AST
   ↓
Semantic Analysis
   ↓
Plan
   ↓
Compilation Input
```

---

# 39. Compilation Input Immutability

Una vez iniciada la compilación:

```text
CompilationInput
```

deberá ser inmutable.

---

# 40. Compilation Input Identity

Podrá incluir:

```text
SemanticFingerprint
PlanFingerprint
```

para caching y diagnostics.

---

# 41. Compilation Context

Toda compilación utilizará:

```text
CompilationContext
```

---

# 42. CompilationContext

Conceptualmente:

```php
final readonly class CompilationContext
{
    public function __construct(
        public Platform $platform,
        public SqlDialect $dialect,
        public CapabilitySnapshot $capabilities,
        public ParameterCompilationContext $parameters,
        public CompilationPolicy $policy,
    ) {}
}
```

---

# 43. CompilationContext ≠ DatabaseContext

No deberán confundirse.

`DatabaseContext` puede incluir:

```text
tenant
shard
read/write intent
transaction affinity
```

El `CompilationContext` contendrá únicamente la información necesaria para producir una representación correcta.

---

# 44. Tenant Data in Compilation

Sólo deberá aparecer si afecta realmente la estructura compilada.

No se copiará todo DatabaseContext por conveniencia.

---

# 45. Compilation Context Immutability

Deberá ser:

```text
immutable
operation-scoped
```

---

# 46. Compiler Instance State

Preferentemente:

```text
Compiler
=
stateless/shared
```

y:

```text
CompilationSession
=
mutable/operation-scoped
```

---

# 47. Compilation Session

Para evitar mutable state dentro de un compiler compartido:

```php
interface CompilationSession
{
}
```

podrá contener:

```text
parameter counter
alias state
temporary fragments
node path
diagnostics
```

---

# 48. Compiler ≠ Compilation Session

Regla:

```text
Compiler
≠
CompilationSession
```

---

# 49. Persistent Runtime Safety

Esto permitirá:

```text
shared immutable Compiler
+
isolated CompilationSession
```

en FrankenPHP, RoadRunner y OpenSwoole.

---

# 50. Compilation Pipeline

Pipeline conceptual:

```text
Compilation Input
      ↓
Precondition Validation
      ↓
Capability Validation
      ↓
Compiler Dispatch
      ↓
Statement Compilation
      ↓
Expression Compilation
      ↓
Dialect Rendering
      ↓
Parameter Compilation
      ↓
Output Assembly
      ↓
Output Validation
      ↓
Compiled Command
```

---

# 51. Compilation ≠ Optimization

El pipeline no deberá reescribir arbitrariamente el Query AST buscando rendimiento.

---

# 52. Normalization Before Compilation

La normalización estructural deberá ocurrir antes cuando corresponda.

---

# 53. Compiler-local Normalization

Sólo se permitirán transformaciones de representación que:

```text
do not change semantics
```

y pertenezcan claramente a la fase de compilación.

---

# 54. Compilation Precondition Validation

Antes de compilar se verificará:

```text
input kind
plan validity
dialect availability
required capabilities
required compiler extensions
```

---

# 55. Semantic Revalidation

El Compiler no deberá repetir todo el Semantic Engine.

Puede verificar invariantes de entrada.

---

# 56. Invalid Compilation Input

Resultado:

```text
InvalidCompilationInputException
```

---

# 57. Capability Validation

Si un plan requiere:

```text
RETURNING
```

el compiler deberá comprobar el soporte efectivo requerido.

---

# 58. Capability UNKNOWN

Regla:

```text
UNKNOWN
≠
SUPPORTED
```

---

# 59. Unsupported Capability

Resultado:

```text
UnsupportedCompilationCapabilityException
```

si no existe una estrategia de emulación previamente aprobada.

---

# 60. Compiler Does Not Invent Emulation

El compiler no deberá descubrir unilateralmente una estrategia semánticamente diferente.

---

# 61. Planner-approved Emulation

Preferido:

```text
Planner
   ↓
Physical Plan
   ├── native strategy
   └── approved emulation strategy
        ↓
Compiler
```

---

# 62. AST Compiler

Podrá existir infraestructura visitor:

```text
AST Node
   ↓
Node Compiler
   ↓
Compiled Fragment
```

---

# 63. Visitor ≠ Mandatory Design

VoltStack podrá utilizar:

```text
visitor
registry dispatch
pattern matching
specialized compiler
```

según rendimiento y claridad.

La arquitectura no dependerá obligatoriamente de Visitor clásico.

---

# 64. Node Compiler

Conceptualmente:

```php
interface NodeCompiler
{
    public function compile(
        AstNode $node,
        CompilationSession $session
    ): CompiledFragment;
}
```

---

# 65. Node Compiler Identity

Cada node compiler deberá asociarse explícitamente a:

```text
AST Node Type
+
Compiler Domain
```

---

# 66. Node Dispatch

Preferido:

```text
NodeClass
→
Compiled NodeCompiler
```

No:

```text
scan every extension
for every node
```

---

# 67. Dispatch Registry

Durante bootstrap:

```text
Mutable Node Compiler Registry
        ↓
Validation
        ↓
Frozen Dispatch Table
```

---

# 68. Statement Compilers

Podrán existir:

```text
SelectCompiler
InsertCompiler
UpdateCompiler
DeleteCompiler
```

---

# 69. SELECT Compiler

Responsabilidades:

```text
projection rendering
FROM rendering
JOIN rendering
predicate placement
GROUP BY
HAVING
ORDER BY
pagination syntax integration
locking syntax integration
```

según el plan recibido.

---

# 70. INSERT Compiler

Responsabilidades:

```text
target table
column list
values representation
parameter mapping
conflict strategy representation
returning representation
```

---

# 71. UPDATE Compiler

Responsabilidades:

```text
target
assignments
source clauses where supported
predicates
returning
```

---

# 72. DELETE Compiler

Responsabilidades:

```text
target
source clauses where supported
predicates
returning
```

---

# 73. Statement Compiler ≠ Query Planner

Ejemplo:

El compiler puede representar:

```text
SELECT_IN relationship loading
```

si el plan ya lo decidió.

No deberá decidir que `SELECT_IN` es mejor que `JOIN`.

---

# 74. Expression Compiler

Responsable de convertir expresiones semánticas en fragments.

Ejemplos:

```text
column reference
parameter
literal
binary expression
predicate
function
case
subquery
aggregate
window expression
```

---

# 75. Expression Compiler ≠ Type Inference

Los tipos ya deberán haber sido resueltos cuando sean necesarios.

---

# 76. Expression Compilation Context

Podrá contener:

```text
expected type
precedence context
null semantics
dialect requirements
```

---

# 77. Operator Precedence

El compiler deberá preservar semántica mediante:

```text
parentheses
precedence rules
associativity
```

---

# 78. Pretty SQL ≠ Correct SQL

La prioridad será:

```text
semantic correctness
```

no minimizar paréntesis.

---

# 79. Function Compilation

Flujo:

```text
Semantic Function
      ↓
Function Compiler/Renderer
      ↓
Dialect Representation
```

---

# 80. Vendor Function Names

No deberán aparecer dispersos en Query Builder/ORM.

---

# 81. Parameter Compilation

El Compiler transformará parámetros semánticos en:

```text
CompiledParameter
```

---

# 82. Compiled Parameter

Conceptualmente:

```php
final readonly class CompiledParameter
{
    public function __construct(
        public ParameterId $id,
        public int $position,
        public string $placeholder,
        public DatabaseType $type,
        public BindingMode $bindingMode,
    ) {}
}
```

---

# 83. Parameter Compilation ≠ Parameter Binding

Regla crítica:

```text
Compiler
→ defines placeholder and binding metadata

Driver
→ performs actual binding
```

---

# 84. Parameter Value Separation

Siempre que sea posible:

```text
Compiled SQL Structure
```

se mantendrá separada de:

```text
Runtime Parameter Values
```

---

# 85. Cache Benefit

Esto permitirá reutilizar:

```text
Compiled Query
```

con valores diferentes.

---

# 86. Parameter Ordering

Para dialectos/drivers posicionales, el Compiler deberá producir un orden determinista.

---

# 87. Duplicate Semantic Parameter

Podrá mapearse a:

```text
one placeholder reused
```

o:

```text
multiple physical placeholders
```

según dialect/driver.

La decisión deberá estar modelada.

---

# 88. Parameter Naming

Los nombres generados deberán ser:

```text
deterministic
collision-safe
valid for target
```

---

# 89. Placeholder Strategy

Provendrá del Dialect/compilation target correspondiente.

---

# 90. Parameter Values Never Become SQL by Default

Regla:

> **La compilación normal no concatenará valores dinámicos en la representación SQL.**

---

# 91. Static Compiler Literals

Algunos elementos estructurales podrán representarse como literals controlados.

Ejemplo:

```text
NULL
TRUE
FALSE
```

según dialecto.

---

# 92. Identifier Compilation

Flujo:

```text
Typed Identifier
     ↓
Validation
     ↓
Dialect Identifier Rules
     ↓
Quoted/Unquoted Representation
```

---

# 93. Raw Identifier Input

No deberá aceptarse directamente desde input no confiable.

---

# 94. Alias Compilation

Los aliases generados deberán ser:

```text
deterministic
collision-free
dialect-valid
```

---

# 95. Alias Generator

Preferentemente estará en:

```text
CompilationSession
```

no como state global del compiler.

---

# 96. Alias Stability

Para un mismo input/configuración:

```text
same alias generation
```

deberá obtenerse cuando el determinismo sea requerido.

---

# 97. Dialect Integration

El Compiler utilizará contratos como:

```text
IdentifierQuoter
PlaceholderStrategy
OperatorRenderer
FunctionRenderer
PaginationSyntax
ReturningSyntax
LockSyntax
```

---

# 98. Compiler Does Not Duplicate Dialect Rules

Incorrecto:

```php
if ($platform === 'postgres') {
    return '"' . $name . '"';
}
```

si `IdentifierQuoter` ya representa esa responsabilidad.

---

# 99. Platform Integration

El Compiler podrá consultar:

```text
physical type mappings
semantic platform rules
feature restrictions
```

mediante contratos explícitos.

---

# 100. Platform ≠ Dialect

El Compiler deberá mantener ambas dependencias separadas.

---

# 101. Capability Integration

Antes de representar una feature:

```text
Requirement
    ↓
Capability Snapshot
    ↓
Support Decision
```

---

# 102. Capability Snapshot Scope

Podrá ser:

```text
platform
endpoint
connection
```

según la feature.

---

# 103. Compilation Against Stale Capabilities

Cuando una compiled query dependa de capabilities variables:

```text
CapabilityGeneration
```

deberá participar en su identidad/cache.

---

# 104. Compilation Output

El resultado deberá ser estructurado.

No únicamente:

```php
string $sql;
```

---

# 105. CompiledCommand

Conceptualmente:

```php
final readonly class CompiledCommand
{
    public function __construct(
        public CompiledCommandId $id,
        public string $commandText,
        public CompiledParameterSet $parameters,
        public ResultExpectation $resultExpectation,
        public CompilationMetadata $metadata,
    ) {}
}
```

---

# 106. CompiledCommand ≠ Executed Statement

Todavía no existe ejecución.

---

# 107. Compilation Metadata

Podrá contener:

```text
compiler id/version
dialect id/version
platform id
capability generation
semantic fingerprint
plan fingerprint
compiled fingerprint
required driver features
```

---

# 108. Metadata ≠ Telemetry Payload

La metadata estructural no deberá incluir valores sensibles innecesarios.

---

# 109. CompiledCommand Immutability

Deberá ser inmutable.

---

# 110. Compiled Command Fingerprint

Podrá calcularse:

```text
CompiledCommandFingerprint
=
H(
  SemanticFingerprint
  + PlanFingerprint
  + CompilerFingerprint
  + DialectFingerprint
  + RelevantCapabilities
)
```

---

# 111. Fingerprint ≠ Raw SQL Hash Only

El hash de SQL puede formar parte, pero no deberá sustituir identidad semántica cuando sea necesaria.

---

# 112. Multi-command Compilation

Algunas operaciones podrían requerir:

```text
CompiledCommandSequence
```

---

# 113. Command Sequence

Conceptualmente:

```text
CompiledCommandSequence
├── Command 1
├── Command 2
└── Command N
```

---

# 114. Multi-command ≠ Transaction

Generar múltiples comandos no implica crear una transacción.

---

# 115. Atomicity Requirements

Si un plan requiere atomicidad:

```text
Transaction Manager
```

deberá gobernarla.

---

# 116. Compiler Must Declare Execution Requirements

El output podrá declarar:

```text
requires transaction
requires same connection
requires writer
requires capability X
requires result from previous command
```

como requirements, sin ejecutar.

---

# 117. Dependent Command Sequence

Ejemplo:

```text
Command A
   ↓ generated value
Command B
```

deberá representarse explícitamente si alguna emulación requiere dependencia.

---

# 118. Compiler Does Not Orchestrate Runtime Dependency

La representación describe.

El Executor/Persistence/Transaction layers ejecutan la secuencia según contrato.

---

# 119. Query Compilation

Pipeline:

```text
Query Plan
   ↓
Query Compiler
   ↓
Statement Compiler
   ↓
Expression Compilers
   ↓
Dialect
   ↓
Parameters
   ↓
Compiled Query
```

---

# 120. Schema Compilation

Pipeline separado:

```text
Schema Operation Plan
   ↓
Schema Compiler
   ↓
Dialect/Platform
   ↓
Compiled Schema Commands
```

---

# 121. Query Compiler ≠ Schema Compiler

Podrán compartir:

```text
identifier rendering
type rendering
fragment infrastructure
diagnostics
```

pero no se fusionarán artificialmente.

---

# 122. Custom Compiler Extensions

Un plugin podrá registrar:

```text
node compilers
statement compilers
expression compilers
function compilers
custom command compilers
```

---

# 123. Extension Registration

Ejemplo conceptual:

```php
$registrar->compilers()
    ->for('sql.postgresql')
    ->node(
        VectorDistanceExpression::class,
        new VectorDistanceCompiler()
    );
```

---

# 124. Extension Registration Time

Sólo durante bootstrap/configuration.

---

# 125. Runtime Mutation

Prohibida por defecto.

---

# 126. Extension Conflict

Si dos extensiones compiten por:

```text
same CompilerId
+
same NodeType
+
same CompilationDomain
```

se producirá:

```text
CompilerExtensionConflictException
```

---

# 127. Last Registration Wins

No será política predeterminada.

---

# 128. Extension Priority

Sólo se utilizará si el extension point define explícitamente semántica de prioridad.

---

# 129. Extension Ordering

Deberá ser:

```text
deterministic
validated
explainable
```

---

# 130. Extension Dependencies

Podrán declararse:

```text
requires
before
after
conflicts
```

cuando sean necesarias.

---

# 131. Extension Graph

Durante bootstrap:

```text
Compiler Extensions
      ↓
Dependency Graph
      ↓
Validation
      ↓
Topological Resolution
      ↓
Frozen Dispatch Tables
```

---

# 132. Cyclic Extension Dependencies

Resultado:

```text
CompilerExtensionCycleException
```

---

# 133. Custom AST Node

Un Query Extension podrá introducir:

```text
CustomAstNode
```

---

# 134. Custom AST Node Requirement

No podrá compilarse hasta que exista un compiler compatible.

---

# 135. AST Extension ≠ Compiler Extension

Son partes distintas:

```text
AST Extension
→ defines semantic representation

Compiler Extension
→ defines target representation
```

---

# 136. Missing Custom Node Compiler

Resultado:

```text
UnsupportedCompilationNodeException
```

---

# 137. Compiler Plugin Integration

Arquitectura:

```text
Composer Package
      ↓
Database Plugin
      ↓
Compiler Extension
      ↓
Compiler Registry
      ↓
Compiler Factory
      ↓
Frozen Compiler
```

---

# 138. Plugin ≠ Compiler

Un plugin puede proporcionar:

```text
driver
platform
dialect
compiler
query extensions
types
```

simultáneamente.

---

# 139. Custom DBMS Package

Ejemplo:

```text
acme/voltstack-oracle
├── OracleDriver
├── OraclePlatform
├── OracleDialect
├── OracleQueryCompiler
├── OracleSchemaCompiler
├── OracleCapabilities
└── OracleConformanceSuite
```

---

# 140. Compiler Composition

Se preferirá composición:

```text
SqlQueryCompiler
├── SelectCompiler
├── InsertCompiler
├── UpdateCompiler
├── DeleteCompiler
├── ExpressionCompilerRegistry
├── ParameterCompiler
└── Dialect
```

sobre una clase monolítica.

---

# 141. God Compiler

Deberá evitarse:

```text
10,000-line compiler
```

con todos los vendors y features mezclados.

---

# 142. Compiler Inheritance

Podrá utilizarse con moderación.

Ejemplo:

```text
BaseSqlCompiler
      ↓
SpecializedCompiler
```

pero composición será preferida para diferencias independientes.

---

# 143. Vendor `if` Explosion

Deberá evitarse:

```php
if ($platform === 'mysql') {
} elseif ($platform === 'postgresql') {
} elseif ($platform === 'sqlite') {
}
```

disperso por los compilers.

---

# 144. Specialized Compiler

No obstante, una diferencia verdaderamente estructural puede justificar:

```text
PostgreSqlSelectCompiler
```

en lugar de forzar una abstracción artificial.

---

# 145. Abstraction Rule

> **Compartir código no deberá tener prioridad sobre preservar correctamente las diferencias semánticas.**

---

# 146. Deterministic Compilation

Dado:

```text
same semantic input
same physical plan
same compiler generation
same dialect generation
same relevant capability snapshot
same compilation policy
```

deberá obtenerse:

```text
equivalent compiled output
```

---

# 147. Determinism Formula

```text
Compile(I, C) = O
```

Para inputs/contextos equivalentes:

```text
I₁ ≡ I₂
∧
C₁ ≡ C₂
→
Compile(I₁, C₁) ≡ Compile(I₂, C₂)
```

---

# 148. Random Aliases

No deberán generarse mediante random global.

---

# 149. Current Time

No deberá afectar compilation salvo que el modelo semántico lo exija explícitamente.

---

# 150. Environment Variables

No deberán consultarse arbitrariamente durante compile.

---

# 151. Hidden Global State

Prohibido.

---

# 152. Compiler Fingerprint

Podrá definirse:

```text
CompilerFingerprint
=
H(
  CompilerId
  + CompilerVersion
  + CompilerConfiguration
  + ExtensionGeneration
)
```

---

# 153. Compiler Generation

Cambiar un renderer deberá cambiar:

```text
CompilerGeneration
```

o fingerprint equivalente.

---

# 154. Compiled Query Cache

La cache podrá usar:

```text
QueryFingerprint
+
PlanFingerprint
+
CompilerFingerprint
+
DialectFingerprint
+
RelevantCapabilityFingerprint
```

---

# 155. Cache Key Minimality

No deberán añadirse dimensiones que no afecten el output.

---

# 156. Cache Key Completeness

Tampoco deberán omitirse dimensiones que sí lo afecten.

---

# 157. Cached Compiled Command ≠ Cached Result

Regla:

```text
Compiled Query Cache
≠
Result Cache
```

---

# 158. Cached Command Values

Preferentemente la cache almacenará estructura parametrizada, no valores de request.

---

# 159. Tenant Isolation and Cache

Si tenant altera SQL estructuralmente:

```text
TenantCompilationDimension
```

deberá participar.

Si sólo altera parámetros:

```text
tenant value
```

no deberá contaminar innecesariamente la compiled query cache.

---

# 160. Shard Context

Misma regla.

---

# 161. Security

El Compiler será una barrera crítica contra SQL injection.

---

# 162. Parameterization

Valores dinámicos:

```text
→ parameters
```

por defecto.

---

# 163. Typed Identifiers

Identifiers:

```text
→ validated typed identifiers
→ dialect quoting
```

---

# 164. Raw Expressions

Sólo mediante escape hatch explícito.

---

# 165. RawExpression Compilation

El compiler deberá poder marcar metadata:

```text
containsRawExpression = true
```

para diagnostics/audit cuando sea útil.

---

# 166. Trusted Raw SQL

No deberá convertirse automáticamente en seguro por llamarse `trusted`.

La confianza deberá provenir del boundary/API correspondiente.

---

# 167. Sensitive Values

Nunca deberán aparecer innecesariamente en:

```text
compiler exception
compiler cache key
compiler diagnostics
telemetry
```

---

# 168. Compilation Errors

Taxonomía propuesta:

```text
CompilerException
├── CompilerNotFoundException
├── DuplicateCompilerException
├── AmbiguousCompilerException
├── InvalidCompilationInputException
├── UnsupportedCompilationNodeException
├── UnsupportedCompilationCapabilityException
├── CompilerExtensionConflictException
├── CompilerExtensionCycleException
├── ParameterCompilationException
├── IdentifierCompilationException
├── DialectCompilationException
└── CompilationInvariantViolationException
```

---

# 169. Compilation Error ≠ Execution Error

El comando todavía no ha sido ejecutado.

---

# 170. Compilation Failure Outcome

Por definición:

```text
CompilationFailed
→
DatabaseStatementNotExecutedByThisCompilation
```

---

# 171. Diagnostics

El Compiler deberá poder explicar:

```text
which compiler was selected
which dialect was selected
which capabilities were required
which node compiler handled each extension
why a feature was rejected
whether emulation was used
which output requirements were generated
```

---

# 172. Compilation Diagnostic Report

Conceptualmente:

```php
final readonly class CompilationDiagnosticReport
{
    public function __construct(
        public CompilerId $compiler,
        public DialectId $dialect,
        public array $requirements,
        public array $decisions,
        public array $warnings,
    ) {}
}
```

---

# 173. Diagnostic Levels

Podrán existir:

```text
TRACE
INFO
WARNING
ERROR
```

para herramientas de desarrollo.

---

# 174. Production Diagnostics

Deberán mantenerse mínimos y sanitizados.

---

# 175. Debug Compilation

En desarrollo podrá habilitarse:

```text
AST
→ Plan
→ Compiler decisions
→ Compiled SQL
```

sin mostrar automáticamente parameter values sensibles.

---

# 176. Explain Compilation

Ejemplo:

```text
Node:
CursorPagination

Plan:
KeysetPagination

Dialect:
PostgreSQL

Capability:
TupleComparison = SUPPORTED

Renderer:
PostgreSqlKeysetRenderer

Result:
Native tuple predicate
```

---

# 177. Compilation Telemetry

Podrá medirse:

```text
compilation latency
cache hit/miss
node count
parameter count
compiled command count
emulation count
compilation failures
```

---

# 178. Telemetry ≠ Compiler Semantics

La telemetría observa.

No modifica el output.

---

# 179. Profiling

Podrá medirse por fases:

```text
validation
dispatch
expression compilation
dialect rendering
parameter compilation
assembly
```

---

# 180. Instrumentation Overhead

Deberá ser configurable y medible.

---

# 181. Performance Requirements

La compilación se encuentra potencialmente en un hot path.

Por ello deberá evitar:

```text
reflection per node
plugin scan per node
container resolution per expression
dynamic file discovery
repeated capability probing
```

---

# 182. Precompiled Dispatch

Preferido:

```text
NodeType
→
NodeCompiler
```

en estructuras inmutables.

---

# 183. Capability Snapshot

No deberán ejecutarse probes de capabilities por cada nodo.

---

# 184. Dependency Injection

Los compiler components deberán resolverse antes del hot path cuando sea posible.

---

# 185. Memory

Una compilation session deberá liberar:

```text
AST traversal references
temporary fragments
diagnostic buffers
parameter maps
```

al finalizar.

---

# 186. Persistent Worker

No deberá existir crecimiento acumulativo por query en:

```text
static arrays
global diagnostics
compiler-local mutable caches
```

---

# 187. Local Compiler Caches

Si existen deberán ser:

```text
bounded
immutable after population
or explicitly governed
```

---

# 188. Compiler Conformance Testing

Todo custom compiler deberá ejecutar una suite común.

---

# 189. Conformance Dimensions

```text
identity
resolution
determinism
parameterization
identifier safety
statement compilation
expression compilation
dialect integration
capability enforcement
output metadata
extension dispatch
error handling
persistent runtime safety
```

---

# 190. Unit Tests

Adecuados para:

```text
AST node → fragment
parameter ordering
identifier rendering
operator precedence
function rendering
statement assembly
```

---

# 191. Golden SQL Tests

Podrán utilizarse.

Pero:

```text
Golden SQL
≠
Execution Correctness
```

---

# 192. Integration Tests

El SQL generado deberá probarse contra DBMS reales cuando la propiedad dependa de su comportamiento.

---

# 193. Semantic Round-trip

Patrón recomendado:

```text
Semantic Query
      ↓
Compiler
      ↓
Compiled SQL
      ↓
Real DBMS
      ↓
Observed Result
      ↓
Expected Semantics
```

---

# 194. Cross-dialect Testing

La misma intención semántica podrá compilarse mediante:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

y producir representaciones distintas.

---

# 195. Same SQL Not Required

Conformance exige:

```text
same required semantics
```

no:

```text
same SQL text
```

---

# 196. Compiler/Driver Boundary Test

Deberá demostrarse que:

```text
Compiler
```

no requiere una conexión para compilar operaciones puras cuando no sea necesario.

---

# 197. Capability Test

Para cada feature:

```text
SUPPORTED
→ compilation succeeds

UNSUPPORTED
→ explicit rejection or approved emulation

UNKNOWN
→ no silent native assumption
```

---

# 198. Parameter Security Test

Payload:

```text
' OR 1=1 --
```

deberá permanecer:

```text
parameter value
```

y no alterar la estructura compilada.

---

# 199. Identifier Security Test

Input arbitrario no deberá convertirse en identifier confiable mediante simple concatenación.

---

# 200. Determinism Test

Compilar repetidamente el mismo input deberá producir output equivalente.

---

# 201. Persistent Runtime Test

```text
Request A
  ↓ compile query A
  ↓ reset operation scope

Request B
  ↓ compile query B
```

No deberá existir contaminación de:

```text
aliases
parameters
tenant
diagnostics
AST state
```

---

# 202. FrankenPHP

Será runtime persistente de referencia.

---

# 203. RoadRunner

Deberá mantener las mismas invariantes.

---

# 204. OpenSwoole

CompilationSession deberá ser aislada por coroutine/operation cuando corresponda.

---

# 205. Compiler Performance Testing

Deberán existir benchmarks para:

```text
simple SELECT
complex predicates
many parameters
many joins
nested subqueries
CTEs
window functions
bulk INSERT
large AST
custom extensions
cold compile
warm compiled-cache lookup
```

---

# 206. Compiler Performance ≠ Driver Performance

La medición deberá aislar:

```text
compilation
```

de:

```text
database execution
```

---

# 207. Custom Compiler Example

```php
final class AcmeQueryCompiler implements QueryCompiler
{
    public function compile(
        QueryCompilationInput $input,
        CompilationContext $context
    ): CompiledCommand {
        $session = new AcmeCompilationSession(
            $context
        );

        return $this->statementCompiler
            ->compile($input->plan, $session);
    }
}
```

---

# 208. Custom Node Compiler Example

```php
final readonly class VectorDistanceCompiler
    implements NodeCompiler
{
    public function compile(
        AstNode $node,
        CompilationSession $session
    ): CompiledFragment {
        // Delegate dialect-specific representation.
    }
}
```

---

# 209. Correct Responsibility

El custom node compiler podrá conocer:

```text
VectorDistanceExpression
```

y cómo traducirla utilizando:

```text
Dialect
Capabilities
```

pero no deberá ejecutar la operación.

---

# 210. Incorrect Custom Compiler

```php
final class BadCompiler
{
    public function compile(Query $query): array
    {
        $pdo = new PDO(...);

        $sql = $this->buildSql($query);

        return $pdo->query($sql)->fetchAll();
    }
}
```

Esto mezcla:

```text
compilation
connection
execution
result fetching
```

y estará prohibido arquitectónicamente.

---

# 211. Query Extension Integration

El documento siguiente profundizará esta relación:

```text
Custom Query Extension
       ↓
Semantic Node
       ↓
Semantic Validation
       ↓
Planning
       ↓
Custom Compiler Extension
       ↓
Dialect Representation
```

---

# 212. ORM Integration

El ORM nunca deberá seleccionar directamente un custom compiler.

Flujo:

```text
ORM
 ↓
Query Model
 ↓
Query Engine
 ↓
Planner
 ↓
Compiler Resolver
```

---

# 213. Schema Integration

Igualmente:

```text
Schema Model
 ↓
Schema Operations
 ↓
Schema Planner
 ↓
Schema Compiler
```

---

# 214. Migration Integration

Migration System no deberá generar SQL manualmente.

```text
Migration
 ↓
Schema Operations
 ↓
Schema Planner
 ↓
Schema Compiler
```

---

# 215. Driver Integration

El Driver recibirá:

```text
CompiledCommand
```

No:

```text
Query AST
```

---

# 216. Execution Integration

El Executor será responsable de:

```text
connection acquisition
statement preparation
binding
execution
result collection
```

utilizando la metadata producida por compilation.

---

# 217. Transaction Integration

El Compiler puede declarar:

```text
transaction requirement
```

pero no iniciar transacciones.

---

# 218. Read/Write Routing

El Compiler no seleccionará:

```text
writer
replica
```

como responsabilidad principal.

---

# 219. Sharding

El Compiler no resolverá el shard.

---

# 220. Multitenancy

El Compiler no resolverá el tenant.

---

# 221. Tenant Predicate

Si Tenant Query Context ya produjo una restricción semántica:

```text
Tenant Filter AST
```

el Compiler simplemente la representará.

---

# 222. Security Policies

Misma regla:

```text
Authorization/Data Access Policy
     ↓
Semantic Query Constraint
     ↓
Compiler
```

El Compiler no decide quién tiene acceso.

---

# 223. Raw SQL Compilation

Podrá existir:

```text
RawSqlCompilationInput
```

como escape hatch.

---

# 224. Raw SQL Validation Limits

VoltStack no deberá fingir que puede demostrar la misma seguridad/semántica de un AST estructurado para SQL arbitrario.

---

# 225. Raw SQL Metadata

Podrá marcarse:

```text
structured = false
containsRawSql = true
portability = unknown
```

---

# 226. Compiler Maturity

Compilers de terceros podrán declarar:

```text
EXPERIMENTAL
PARTIAL
STABLE
DEPRECATED
```

---

# 227. Maturity ≠ Capability

Un compiler `STABLE` puede no implementar todas las features.

---

# 228. Compiler Compatibility

Un compiler deberá declarar compatibilidad con:

```text
AST generation
Plan generation
Dialect family/version
Platform family
Extension API generation
```

cuando corresponda.

---

# 229. API Compatibility

No deberá asumirse por número de package solamente.

---

# 230. Compiler Replacement

Cambiar:

```text
Compiler A
→
Compiler B
```

para la misma plataforma deberá ejecutar conformance y integration tests.

---

# 231. Equivalent Compiler Assumption

Prohibido:

```text
same platform
→
same compiler semantics
```

sin evidencia.

---

# 232. Compiler Fallback

No deberá existir fallback silencioso hacia un compiler genérico si pudiera cambiar semántica.

---

# 233. Explicit Fallback

Sólo podrá ocurrir si:

```text
compatibility proven
+
policy allows
```

---

# 234. Unknown Compilation Feature

No deberá convertirse automáticamente en RawExpression.

---

# 235. Fail Closed

Preferido:

```text
unknown semantic node
→
compilation error
```

---

# 236. Developer Diagnostics

El error deberá indicar:

```text
node type
compiler id
dialect id
required extension
capability state
```

sin secretos.

---

# 237. Compiler Discovery

La disponibilidad se resolverá durante bootstrap.

No mediante filesystem scan por query.

---

# 238. Composer Integration

Composer podrá distribuir packages.

Pero:

```text
Composer
≠
Compiler Registry
```

---

# 239. Service Container Integration

El Container podrá construir factories/components.

Pero:

```text
Container
≠
Compiler Resolver
```

---

# 240. Compiler Public Extension API

Deberá ser pequeña y estable.

Preferentemente:

```text
CompilerExtension
CompilerExtensionRegistrar
NodeCompiler
StatementCompiler
ExpressionCompiler
CompilationContext
CompiledFragment
CompiledCommand
```

---

# 241. Internal APIs

Detalles como:

```text
specific SQL buffer
internal alias map
private optimization helpers
```

no deberán convertirse accidentalmente en API pública.

---

# 242. Versioned Extension Contracts

Los extension points deberán evolucionar mediante política de versionado.

---

# 243. Compiler Testing Kit

VoltStack podrá distribuir:

```text
CompilerConformanceKit
```

para terceros.

---

# 244. Proposed Directory Structure

```text
src/Quantum/Database/Compiler/
├── Contract/
│   ├── DatabaseCompiler.php
│   ├── QueryCompiler.php
│   ├── SchemaCompiler.php
│   ├── CompilerFactory.php
│   ├── NodeCompiler.php
│   ├── StatementCompiler.php
│   └── ExpressionCompiler.php
│
├── Identity/
│   ├── CompilerId.php
│   ├── CompilerVersion.php
│   ├── CompilerGeneration.php
│   └── CompilerFingerprint.php
│
├── Descriptor/
│   ├── CompilerDescriptor.php
│   ├── CompilerKind.php
│   └── CompilationTarget.php
│
├── Registry/
│   ├── CompilerRegistry.php
│   ├── MutableCompilerRegistry.php
│   └── FrozenCompilerRegistry.php
│
├── Resolution/
│   ├── CompilerResolver.php
│   ├── CompilerSelection.php
│   └── CompilerCompatibilityResolver.php
│
├── Context/
│   ├── CompilationContext.php
│   ├── CompilerConstructionContext.php
│   ├── CompilationPolicy.php
│   └── CompilationSession.php
│
├── Input/
│   ├── CompilationInput.php
│   ├── QueryCompilationInput.php
│   ├── SchemaCompilationInput.php
│   └── RawSqlCompilationInput.php
│
├── Output/
│   ├── CompiledCommand.php
│   ├── CompiledCommandSequence.php
│   ├── CompiledFragment.php
│   ├── CompilationMetadata.php
│   └── ExecutionRequirements.php
│
├── Query/
│   ├── SelectCompiler.php
│   ├── InsertCompiler.php
│   ├── UpdateCompiler.php
│   └── DeleteCompiler.php
│
├── Expression/
│   ├── ExpressionCompilerRegistry.php
│   ├── BinaryExpressionCompiler.php
│   ├── PredicateCompiler.php
│   ├── FunctionCompiler.php
│   └── SubqueryCompiler.php
│
├── Parameter/
│   ├── ParameterCompiler.php
│   ├── CompiledParameter.php
│   └── CompiledParameterSet.php
│
├── Identifier/
│   ├── IdentifierCompiler.php
│   └── AliasGenerator.php
│
├── Extension/
│   ├── CompilerExtension.php
│   ├── CompilerExtensionRegistrar.php
│   ├── CompilerExtensionGraph.php
│   └── FrozenCompilerDispatchTable.php
│
├── Cache/
│   ├── CompilerCacheIdentity.php
│   └── CompiledCommandFingerprint.php
│
├── Diagnostics/
│   ├── CompilationDiagnosticReport.php
│   ├── CompilationTrace.php
│   └── CompilerInspector.php
│
└── Exception/
    ├── CompilerException.php
    ├── CompilerNotFoundException.php
    ├── DuplicateCompilerException.php
    ├── AmbiguousCompilerException.php
    ├── InvalidCompilationInputException.php
    ├── UnsupportedCompilationNodeException.php
    ├── UnsupportedCompilationCapabilityException.php
    ├── CompilerExtensionConflictException.php
    └── CompilationInvariantViolationException.php
```

---

# 245. Third-party Package Structure

```text
acme/voltstack-database-compiler/
├── composer.json
├── src/
│   ├── Plugin/
│   ├── Extension/
│   ├── Compiler/
│   ├── Statement/
│   ├── Expression/
│   └── Diagnostics/
└── tests/
    ├── Unit/
    ├── Conformance/
    └── Integration/
```

---

# 246. Architectural Invariants

## DB-CUSTOM-COMP-001

Compiler ≠ Query Builder.

## DB-CUSTOM-COMP-002

Compiler ≠ Semantic Analyzer.

## DB-CUSTOM-COMP-003

Compiler ≠ Optimizer.

## DB-CUSTOM-COMP-004

Compiler ≠ Planner.

## DB-CUSTOM-COMP-005

Compiler ≠ Dialect.

## DB-CUSTOM-COMP-006

Compiler ≠ Platform.

## DB-CUSTOM-COMP-007

Compiler ≠ Capability System.

## DB-CUSTOM-COMP-008

Compiler ≠ Driver.

## DB-CUSTOM-COMP-009

Compiler ≠ Executor.

## DB-CUSTOM-COMP-010

CompilerId será estable.

## DB-CUSTOM-COMP-011

CompilerId ≠ FQCN.

## DB-CUSTOM-COMP-012

CompilerVersion ≠ DialectVersion.

## DB-CUSTOM-COMP-013

CompilerVersion ≠ ServerVersion.

## DB-CUSTOM-COMP-014

Query Compiler ≠ Schema Compiler.

## DB-CUSTOM-COMP-015

Descriptor ≠ Compiler Instance.

## DB-CUSTOM-COMP-016

Duplicate CompilerId será error.

## DB-CUSTOM-COMP-017

Compiler Construction Context no contendrá request state.

## DB-CUSTOM-COMP-018

Ambiguous Compiler será error.

## DB-CUSTOM-COMP-019

Compiler Resolution ≠ Capability Resolution.

## DB-CUSTOM-COMP-020

Mutable Query Builder no será input ideal del compiler.

## DB-CUSTOM-COMP-021

Compilation Input será inmutable.

## DB-CUSTOM-COMP-022

CompilationContext ≠ DatabaseContext.

## DB-CUSTOM-COMP-023

Compiler será stateless cuando sea posible.

## DB-CUSTOM-COMP-024

Compiler ≠ CompilationSession.

## DB-CUSTOM-COMP-025

Compilation ≠ Optimization.

## DB-CUSTOM-COMP-026

Compiler no repetirá Semantic Engine.

## DB-CUSTOM-COMP-027

UNKNOWN capability ≠ SUPPORTED.

## DB-CUSTOM-COMP-028

Compiler no inventará emulation strategy.

## DB-CUSTOM-COMP-029

Planner-approved emulation precederá compilation.

## DB-CUSTOM-COMP-030

Visitor pattern no será obligatorio.

## DB-CUSTOM-COMP-031

Node dispatch será determinista.

## DB-CUSTOM-COMP-032

Statement Compiler ≠ Query Planner.

## DB-CUSTOM-COMP-033

Expression Compiler ≠ Type Inference.

## DB-CUSTOM-COMP-034

Semantic correctness tendrá prioridad sobre pretty SQL.

## DB-CUSTOM-COMP-035

Parameter Compilation ≠ Parameter Binding.

## DB-CUSTOM-COMP-036

Runtime values permanecerán separados de SQL estructural.

## DB-CUSTOM-COMP-037

Parameter ordering será determinista.

## DB-CUSTOM-COMP-038

Parameter naming será collision-safe.

## DB-CUSTOM-COMP-039

Dynamic values no serán concatenados por defecto.

## DB-CUSTOM-COMP-040

Identifiers serán tipados/validados.

## DB-CUSTOM-COMP-041

Alias state pertenecerá a CompilationSession.

## DB-CUSTOM-COMP-042

Compiler no duplicará reglas del Dialect.

## DB-CUSTOM-COMP-043

Platform ≠ Dialect.

## DB-CUSTOM-COMP-044

Capability probing no ocurrirá por AST node.

## DB-CUSTOM-COMP-045

CompiledCommand será estructurado.

## DB-CUSTOM-COMP-046

CompiledCommand ≠ Executed Statement.

## DB-CUSTOM-COMP-047

CompiledCommand será inmutable.

## DB-CUSTOM-COMP-048

Compiled fingerprint ≠ raw SQL hash únicamente.

## DB-CUSTOM-COMP-049

Multi-command ≠ Transaction.

## DB-CUSTOM-COMP-050

Compiler no iniciará transacciones.

## DB-CUSTOM-COMP-051

Execution requirements podrán declararse sin ejecutarse.

## DB-CUSTOM-COMP-052

Compiler no orquestará runtime dependencies.

## DB-CUSTOM-COMP-053

AST Extension ≠ Compiler Extension.

## DB-CUSTOM-COMP-054

Missing custom node compiler será error.

## DB-CUSTOM-COMP-055

Plugin ≠ Compiler.

## DB-CUSTOM-COMP-056

Composition será preferida a God Compiler.

## DB-CUSTOM-COMP-057

Vendor conditionals dispersos deberán evitarse.

## DB-CUSTOM-COMP-058

Code sharing no romperá semantic differences.

## DB-CUSTOM-COMP-059

Compilation será determinista.

## DB-CUSTOM-COMP-060

Random global no controlará aliases.

## DB-CUSTOM-COMP-061

Current time no afectará compile implícitamente.

## DB-CUSTOM-COMP-062

Environment variables no serán hidden compiler input.

## DB-CUSTOM-COMP-063

Compiler fingerprint reflejará extensiones relevantes.

## DB-CUSTOM-COMP-064

Compiled Query Cache ≠ Result Cache.

## DB-CUSTOM-COMP-065

Compiled cache no almacenará request values innecesariamente.

## DB-CUSTOM-COMP-066

Tenant sólo afectará cache key si afecta output estructural.

## DB-CUSTOM-COMP-067

Shard sólo afectará cache key si afecta output estructural.

## DB-CUSTOM-COMP-068

Compiler preservará parameterization.

## DB-CUSTOM-COMP-069

RawExpression será escape hatch explícito.

## DB-CUSTOM-COMP-070

Sensitive values no aparecerán en cache identity.

## DB-CUSTOM-COMP-071

Compilation Error ≠ Execution Error.

## DB-CUSTOM-COMP-072

Compilation failure no implicará statement enviado.

## DB-CUSTOM-COMP-073

Diagnostics serán sanitizados.

## DB-CUSTOM-COMP-074

Telemetry no modificará compilation semantics.

## DB-CUSTOM-COMP-075

Reflection per node deberá evitarse en hot path.

## DB-CUSTOM-COMP-076

Plugin scan per node estará prohibido.

## DB-CUSTOM-COMP-077

Capability probes per node estarán prohibidos.

## DB-CUSTOM-COMP-078

CompilationSession no filtrará estado entre requests.

## DB-CUSTOM-COMP-079

Golden SQL ≠ Execution Correctness.

## DB-CUSTOM-COMP-080

Conformance exige semántica, no mismo SQL.

## DB-CUSTOM-COMP-081

Real DBMS será necesario para probar comportamiento real.

## DB-CUSTOM-COMP-082

UNKNOWN capability no activará sintaxis nativa.

## DB-CUSTOM-COMP-083

Security payload permanecerá parameterized.

## DB-CUSTOM-COMP-084

Compiler no resolverá ORM entities.

## DB-CUSTOM-COMP-085

Compiler no resolverá tenant.

## DB-CUSTOM-COMP-086

Compiler no resolverá shard.

## DB-CUSTOM-COMP-087

Compiler no seleccionará replica.

## DB-CUSTOM-COMP-088

Compiler no administrará connection pool.

## DB-CUSTOM-COMP-089

Compiler no ejecutará statements.

## DB-CUSTOM-COMP-090

Compiler no hará hydration.

## DB-CUSTOM-COMP-091

Compiler no hará persistence.

## DB-CUSTOM-COMP-092

Compiler no determinará transaction outcome.

## DB-CUSTOM-COMP-093

Raw SQL tendrá garantías reducidas explícitas.

## DB-CUSTOM-COMP-094

Maturity ≠ Capability.

## DB-CUSTOM-COMP-095

Same Platform ≠ Equivalent Compiler.

## DB-CUSTOM-COMP-096

Compiler fallback no será silencioso.

## DB-CUSTOM-COMP-097

Unknown node no se convertirá automáticamente en raw SQL.

## DB-CUSTOM-COMP-098

Compiler discovery ocurrirá fuera del query hot path.

## DB-CUSTOM-COMP-099

Container ≠ Compiler Resolver.

## DB-CUSTOM-COMP-100

Custom Compiler preservará las invariantes globales de Database.

---

# 247. Anti-patrones

## 247.1 Compiler ejecutando SQL

Incorrecto.

---

## 247.2 Compiler abriendo conexiones

Incorrecto.

---

## 247.3 Compiler hidratando entidades

Incorrecto.

---

## 247.4 Compiler decidiendo qué relación eager-load

Incorrecto.

---

## 247.5 Compiler optimizando joins arbitrariamente

Incorrecto.

---

## 247.6 Compiler seleccionando writer/replica

Incorrecto.

---

## 247.7 Compiler resolviendo tenant

Incorrecto.

---

## 247.8 Compiler resolviendo shard

Incorrecto.

---

## 247.9 Compiler haciendo capability probe por nodo

Incorrecto.

---

## 247.10 Compiler concatenando parámetros

Incorrecto.

---

## 247.11 Compiler aceptando identifiers arbitrarios

Incorrecto.

---

## 247.12 Plugin scan por expresión

Incorrecto.

---

## 247.13 Resolver dependencias desde Container por cada AST node

Incorrecto.

---

## 247.14 Usar random para aliases

Incorrecto.

---

## 247.15 Guardar parameter counter en singleton compartido

Incorrecto.

---

## 247.16 God Compiler para todos los DBMS

Debe evitarse.

---

## 247.17 Copiar reglas de Dialect dentro del Compiler

Incorrecto.

---

## 247.18 Interpretar UNKNOWN como soportado

Incorrecto.

---

## 247.19 Convertir nodos desconocidos a raw SQL

Incorrecto.

---

## 247.20 Autoactualizar compiled cache sin generación/fingerprint

Incorrecto.

---

# 248. Modelo formal

Sea:

```text
I
```

un input semántico válido,

```text
P
```

un plan,

```text
C
```

un contexto de compilación,

y:

```text
K
```

un compiler.

Entonces:

```text
Compile(K, I, P, C)
→
CompiledCommand
```

deberá preservar:

```text
Semantics(CompiledCommand)
≡
Semantics(I, P)
```

dentro de las garantías del target.

---

# 249. Valid Compilation

```text
ValidCompilation
=
ValidSemanticInput
∧
ValidPlan
∧
CompilerCompatible
∧
DialectCompatible
∧
RequiredCapabilitiesSatisfied
∧
ExtensionsResolved
∧
OutputInvariantSatisfied
```

---

# 250. Compiler Selection

```text
ResolveCompiler(
    Kind,
    Target,
    Platform,
    Dialect,
    Configuration
)
→
ExactlyOneCompiler
```

Si no:

```text
0
→ CompilerNotFound

>1
→ AmbiguousCompiler
```

---

# 251. Deterministic Output

```text
EquivalentInputs
∧
EquivalentCompilationContext
∧
SameCompilerGeneration
→
EquivalentCompiledOutput
```

---

# 252. Cache Validity

```text
CompiledCacheReusable
=
SameSemanticFingerprint
∧
SamePlanFingerprint
∧
SameCompilerFingerprint
∧
SameDialectFingerprint
∧
SameRelevantCapabilityGeneration
```

---

# 253. Execution Separation

```text
Compile(Query)
→
Command
```

no:

```text
Compile(Query)
→
DatabaseResult
```

---

# 254. Parameter Safety

Para un valor dinámico `V`:

```text
V ∈ RuntimeValues
→
V ∉ SQLStructure
```

salvo un escape hatch explícito y controlado.

---

# 255. Arquitectura consolidada

```text
                    Query Builder / ORM
                           │
                           ▼
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
                   Compilation Input
                           │
                           ▼
                  ┌──────────────────┐
                  │ Compiler Resolver│
                  └────────┬─────────┘
                           ▼
                  ┌──────────────────┐
                  │ Custom Compiler  │
                  └────────┬─────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       Statements      Expressions     Parameters
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                     ┌───────────┐
                     │  Dialect  │
                     └─────┬─────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Platform    Capabilities   Extensions
              │            │            │
              └────────────┼────────────┘
                           ▼
                   Compiled Command
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

---

# 256. Estrategia V1

Para V1 deberán estabilizarse:

```text
CompilerId
CompilerDescriptor
CompilerFactory
CompilerRegistry
CompilerResolver
CompilationInput
CompilationContext
CompilationSession
QueryCompiler
SchemaCompiler contracts
StatementCompiler
ExpressionCompiler
ParameterCompiler
CompiledFragment
CompiledCommand
ExecutionRequirements
Dialect integration
Capability validation
CompilerFingerprint
Extension dispatch
Conformance testing
```

---

# 257. Evolución posterior

La arquitectura podrá evolucionar hacia:

```text
async compilation
parallel compilation of independent fragments
compiler IR
ahead-of-time compilation
precompiled query manifests
native protocol command compilation
specialized analytical compilers
vector database targets
distributed query targets
remote compiler services
```

sin romper las fronteras fundamentales.

---

# 258. Regla final

> **El Custom Compiler System será la frontera que transforma intención semántica ya validada y planificada en comandos ejecutables para un target concreto. Podrá extender la representación de VoltStack Database, pero nunca deberá apropiarse de las responsabilidades de construcción, análisis, optimización, planificación, transporte o ejecución.**

Por tanto:

```text
Compiler
≠
Query Builder
```

```text
Compiler
≠
Semantic Analyzer
```

```text
Compiler
≠
Optimizer
```

```text
Compiler
≠
Planner
```

```text
Compiler
≠
Dialect
```

```text
Compiler
≠
Platform
```

```text
Compiler
≠
Capability System
```

```text
Compiler
≠
Driver
```

```text
Compiler
≠
Executor
```

```text
Compiler
≠
CompilationSession
```

```text
Parameter Compilation
≠
Parameter Binding
```

```text
CompiledCommand
≠
Executed Statement
```

```text
Multi-command Compilation
≠
Transaction
```

```text
AST Extension
≠
Compiler Extension
```

```text
Compilation Error
≠
Execution Error
```

```text
Golden SQL
≠
Runtime Correctness
```

```text
Same Platform
≠
Equivalent Compiler
```

```text
UNKNOWN Capability
≠
SUPPORTED
```

y finalmente:

```text
Safe Custom Compiler
=
Stable Identity
+
Immutable Inputs
+
Explicit Compilation Context
+
Isolated Compilation Session
+
Deterministic Dispatch
+
Semantic Preservation
+
Dialect Integration
+
Platform Integration
+
Capability Validation
+
Safe Parameter Compilation
+
Typed Identifier Compilation
+
Structured Output
+
Cache-aware Identity
+
Extension Conflict Detection
+
Diagnostics
+
Persistent Runtime Isolation
+
Conformance Testing
```

---

# 259. Siguiente documento

```text
299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM.md
```

El siguiente documento deberá definir cómo VoltStack permitirá ampliar el lenguaje semántico de consultas sin recurrir a raw SQL como mecanismo principal.

La arquitectura deberá cubrir:

```text
Custom Query Extension System
│
├── Query Extension Identity
├── Extension Descriptor
├── Semantic Extension Registry
├── Custom AST Nodes
├── Custom Expressions
├── Custom Predicates
├── Custom Operators
├── Custom Functions
├── Custom Aggregates
├── Custom Window Expressions
├── Custom Clauses
├── Query Builder Integration
├── Semantic Validation
├── Type Inference Integration
├── Optimizer Integration
├── Planner Integration
├── Compiler Integration
├── Dialect Integration
├── Capability Requirements
├── Security Boundaries
├── Portability Model
├── Extension Conflicts
├── Diagnostics
├── Testing
└── Plugin Integration
```

manteniendo como principio:

> **Una Custom Query Extension deberá ampliar el modelo semántico de Query Engine mediante estructuras tipadas, validables, optimizables, planificables y compilables; raw SQL será un escape hatch, no el mecanismo principal de extensibilidad.**