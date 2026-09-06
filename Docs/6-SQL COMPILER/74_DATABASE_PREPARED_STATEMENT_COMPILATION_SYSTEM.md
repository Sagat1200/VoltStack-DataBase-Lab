# 74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md

# VoltStack Quantum Database
## Prepared Statement Compilation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 74 — Prepared Statement Compilation System  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Prepared Statement Compilation System` define la arquitectura mediante la cual VoltStack transforma un:

```text
CompiledDatabaseCommand
```

en una representación preparada, reusable y todavía independiente de una conexión viva:

```text
PreparedStatementBlueprint
```

El objetivo es separar rigurosamente:

```text
SQL Compilation
Prepared Statement Compilation
Live Statement Preparation
Runtime Parameter Binding
Statement Execution
Result Fetching
```

Principio central:

```text
Prepared Statement Compilation
≠
Database Statement Preparation
```

La primera es una transformación estructural.

La segunda requiere una conexión real.

---

# 2. Fórmula fundamental

```text
CompilePreparedStatement(
    CompiledDatabaseCommand,
    PreparedStatementCompilationContext
)
→ PreparedStatementBlueprint
```

mientras que posteriormente:

```text
Prepare(
    PreparedStatementBlueprint,
    Connection
)
→ PreparedStatement
```

y:

```text
Bind(
    PreparedStatement,
    RuntimeBindings
)
→ BoundPreparedStatement
```

seguido por:

```text
Execute(
    BoundPreparedStatement
)
→ ExecutionResult
```

---

# 3. Objetivo arquitectónico

VoltStack deberá poder ejecutar:

```text
Query
 ↓
SQL Compiler
 ↓
CompiledDatabaseCommand
 ↓
Prepared Statement Compiler
 ↓
PreparedStatementBlueprint
 ↓
Connection
 ↓
Driver Preparation
 ↓
Live PreparedStatement
 ↓
Runtime Binding
 ↓
Execution
```

sin permitir que las responsabilidades de estas capas se mezclen.

---

# 4. Posición dentro del pipeline

```text
Query AST
   ↓
Semantic Engine
   ↓
Optimizer
   ↓
Logical Plan
   ↓
Physical Plan
   ↓
Execution Plan
   ↓
SQL Compiler
   ↓
CompiledDatabaseCommand
   ↓
┌───────────────────────────────────────────┐
│ Prepared Statement Compilation System     │
└───────────────────────────────────────────┘
   ↓
PreparedStatementBlueprint
   ↓
Execution Engine
   ↓
Connection Resolution
   ↓
Driver
   ↓
Live Statement Preparation
   ↓
Runtime Binding
   ↓
Execution
```

---

# 5. Separación conceptual

VoltStack distinguirá al menos:

```text
CompiledDatabaseCommand
PreparedStatementBlueprint
PreparedStatement
BoundPreparedStatement
StatementExecution
Result
```

Estas entidades no son equivalentes.

---

# 6. CompiledDatabaseCommand

Representa el resultado del SQL Compiler.

Conceptualmente:

```php
final readonly class CompiledDatabaseCommand
{
    public function __construct(
        public CompiledCommandId $id,
        public CompiledCommandKind $kind,
        public RenderedSql $sql,
        public CompiledBindingLayout $bindings,
        public CompiledResultContract $result,
        public CompilationDependencySet $dependencies,
        public CompiledQueryFingerprint $fingerprint,
        public CompiledCommandMetadata $metadata,
    ) {}
}
```

Contiene SQL y metadata de compilación.

No contiene:

```text
PDOStatement
connection
transaction
cursor
runtime values
execution state
```

---

# 7. PreparedStatementBlueprint

El resultado principal del sistema será:

```php
final readonly class PreparedStatementBlueprint
{
    public function __construct(
        public PreparedStatementBlueprintId $id,
        public PreparedSqlDescriptor $sql,
        public PreparedParameterLayout $parameters,
        public PreparedResultDescriptor $result,
        public StatementPreparationRequirements $requirements,
        public StatementDependencySet $dependencies,
        public StatementCacheabilityDescriptor $cacheability,
        public PreparedStatementFingerprint $fingerprint,
        public PreparedStatementMetadata $metadata,
    ) {}
}
```

---

# 8. Blueprint ≠ live statement

```text
PreparedStatementBlueprint
≠
PDOStatement
```

y:

```text
PreparedStatementBlueprint
≠
mysqli_stmt
```

y:

```text
PreparedStatementBlueprint
≠
PostgreSQL server prepared statement
```

El blueprint es framework-level.

---

# 9. Blueprint properties

Un blueprint deberá ser:

```text
immutable
serializable where possible
deterministic
connection-independent where possible
runtime-value-free
persistent-worker-safe
cacheable when valid
```

---

# 10. Prepared statement compilation

La compilación preparada deberá resolver todo aquello que pueda resolverse sin abrir una conexión.

Por ejemplo:

```text
parameter slots
binding order
driver type intent
conversion descriptors
result expectations
statement requirements
cacheability
dependency metadata
preparation strategy requirements
```

---

# 11. Lo que NO hará

No deberá:

```text
open connection
prepare statement on server
allocate PDOStatement
bind runtime values
execute SQL
fetch rows
begin transaction
commit transaction
rollback transaction
select replica
hydrate entities
```

---

# 12. Compiler boundary

```text
Prepared Statement Compiler
        │
        ├── understands compiled SQL structure
        ├── understands binding descriptors
        ├── understands target preparation capabilities
        └── produces preparation blueprint
```

pero:

```text
Prepared Statement Compiler
        ✕
        database connection
```

---

# 13. Preparation boundary

Posteriormente:

```text
Execution Engine
        ↓
Connection
        ↓
Driver Statement Preparer
        ↓
PreparedStatementBlueprint
        ↓
Live PreparedStatement
```

---

# 14. Core distinction

```text
Compilation determines:
"What must be prepared?"

Runtime preparation determines:
"Prepare it on this connection."
```

---

# 15. Prepared statement architecture

```text
CompiledDatabaseCommand
        │
        ▼
Prepared Statement Compiler
        │
        ├── Command Validation
        ├── Preparation Capability Validation
        ├── Parameter Layout Compilation
        ├── Binding Conversion Planning
        ├── Result Descriptor Compilation
        ├── Preparation Requirement Resolution
        ├── Affinity Analysis
        ├── Cacheability Analysis
        ├── Dependency Collection
        └── Fingerprinting
        │
        ▼
PreparedStatementBlueprint
```

---

# 16. Formal compilation pipeline

```text
PS0  Input Validation
PS1  Preparation Capability Resolution
PS2  SQL Preparation Analysis
PS3  Parameter Slot Compilation
PS4  Binding Conversion Planning
PS5  Result Descriptor Compilation
PS6  Preparation Requirement Compilation
PS7  Connection/Session Affinity Analysis
PS8  Statement Reuse Analysis
PS9  Cacheability Classification
PS10 Dependency Compilation
PS11 Fingerprint Construction
PS12 Blueprint Validation
PS13 Freeze
```

---

# 17. PS0 — Input validation

Debe validar:

```text
CompiledDatabaseCommand is complete
RenderedSql is valid
BindingLayout is internally consistent
ResultContract exists
dependencies are known
compiler fingerprint exists
```

---

# 18. No repair during PS0

Si el SQL Compiler produjo un artifact inválido:

```text
→ fail
```

No:

```text
→ silently repair
```

---

# 19. PS1 — Preparation capability resolution

El sistema deberá analizar las capacidades necesarias para preparar el command.

Ejemplos:

```text
supportsPreparedStatements
supportsNamedParameters
supportsPositionalParameters
supportsServerSidePrepare
supportsClientSidePrepare
supportsPreparedStatementReuse
supportsBinaryParameterBinding
supportsStreamingParameters
supportsPreparedReturning
```

---

# 20. Capability snapshot

```php
final readonly class StatementPreparationCapabilitySnapshot
{
    public function __construct(
        public CapabilitySet $capabilities,
        public CapabilityFingerprint $fingerprint,
    ) {}
}
```

---

# 21. Capability snapshot ≠ connection

El snapshot podrá haberse construido antes.

No implica poseer una conexión viva.

---

# 22. Version ≠ capability

Nunca asumir:

```text
database version X
→ automatically behavior Y
```

cuando exista una capability explícita.

---

# 23. PS2 — SQL preparation analysis

Esta fase analiza:

```text
statement kind
placeholder representation
statement count
parameter count
result behavior
preparation restrictions
target requirements
```

---

# 24. Single statement invariant

Por defecto:

```text
1 CompiledDatabaseCommand
=
1 SQL Statement
=
1 PreparedStatementBlueprint
```

---

# 25. Hidden multi-statement SQL forbidden

Nunca:

```sql
UPDATE users SET ...;
SELECT ...;
```

oculto dentro de un único blueprint para emular una feature.

---

# 26. Explicit command batches

Si en el futuro VoltStack soporta batches:

```text
CompiledCommandBatch
        ↓
multiple PreparedStatementBlueprint
```

deberá modelarse explícitamente.

---

# 27. Statement kind

```php
enum PreparedStatementKind
{
    case SELECT;
    case INSERT;
    case UPDATE;
    case DELETE;
    case DDL;
    case UTILITY;
    case EXPLAIN;
    case EXTENSION;
}
```

---

# 28. SQL descriptor

```php
final readonly class PreparedSqlDescriptor
{
    public function __construct(
        public string $sql,
        public PreparedStatementKind $kind,
        public PlaceholderStyle $placeholderStyle,
        public int $parameterOccurrenceCount,
        public SqlTextFingerprint $fingerprint,
    ) {}
}
```

---

# 29. SQL text remains immutable

Prepared Statement Compilation no deberá reescribir arbitrariamente SQL.

---

# 30. Driver preparation adaptation

Si un driver necesita una forma diferente de SQL para preparación, deberá existir una transformación formal:

```text
Compiled SQL
        ↓
Driver Preparation Adapter
        ↓
Prepared SQL Representation
```

---

# 31. Adaptation restrictions

La adaptación deberá preservar:

```text
semantics
parameter identity
parameter occurrence semantics
result shape
security predicates
locking behavior
mutation behavior
```

---

# 32. Prepared SQL ≠ semantic rewrite

No deberá utilizarse para:

```text
optimizer rewrites
join reorder
predicate removal
pagination changes
RETURNING emulation
```

---

# 33. PS3 — Parameter slot compilation

Ésta es una de las responsabilidades principales.

Entrada:

```text
CompiledBindingLayout
```

Salida:

```text
PreparedParameterLayout
```

---

# 34. Parameter identities

VoltStack distinguirá:

```text
Query ParameterId
ExecutionParameterSlotId
SqlPlaceholderId
PreparedParameterSlotId
DriverBindingPosition
RuntimeValue
```

---

# 35. No identity collapse

Nunca asumir:

```text
ParameterId == DriverBindingPosition
```

---

# 36. Parameter occurrence

Un mismo parameter puede aparecer varias veces:

```sql
WHERE a = ? OR b = ?
```

y ambos placeholders podrían provenir del mismo:

```text
ParameterId(P1)
```

---

# 37. Prepared parameter layout

```php
final readonly class PreparedParameterLayout
{
    /**
     * @param list<PreparedParameterSlot> $slots
     */
    public function __construct(
        public array $slots,
        public PreparedParameterLayoutFingerprint $fingerprint,
    ) {}
}
```

---

# 38. Prepared parameter slot

```php
final readonly class PreparedParameterSlot
{
    public function __construct(
        public PreparedParameterSlotId $id,
        public ExecutionParameterSlotId $executionSlot,
        public SqlPlaceholderId $placeholder,
        public DriverBindingDescriptor $driverBinding,
        public QueryType $semanticType,
        public ParameterNullability $nullability,
        public ParameterSensitivity $sensitivity,
        public BindingConversionPlan $conversion,
    ) {}
}
```

---

# 39. Slot ≠ runtime value

```text
PreparedParameterSlot
≠
RuntimeBinding
```

El slot describe dónde y cómo.

El runtime binding contiene el valor.

---

# 40. Driver binding descriptor

```php
final readonly class DriverBindingDescriptor
{
    public function __construct(
        public DriverBindingMode $mode,
        public ?int $position,
        public ?string $name,
        public DriverParameterType $type,
    ) {}
}
```

---

# 41. Binding modes

```php
enum DriverBindingMode
{
    case POSITIONAL;
    case NAMED;
    case NUMBERED;
    case DRIVER_NATIVE;
}
```

---

# 42. Position normalization

La arquitectura deberá evitar errores de:

```text
0-based
vs
1-based
```

mediante value objects explícitos.

---

# 43. DriverBindingPosition

```php
final readonly class DriverBindingPosition
{
    private function __construct(
        public int $value,
    ) {}
}
```

La validación pertenece al adapter correspondiente.

---

# 44. Named parameter identity

```text
Semantic Parameter Name
≠
Rendered Placeholder Name
≠
Driver Parameter Name
```

---

# 45. Placeholder duplication

Un driver puede permitir reutilizar un named placeholder.

Otro puede requerir occurrences separados.

El blueprint deberá representar la estrategia real.

---

# 46. Parameter occurrence map

```text
Parameter P1
   ├── occurrence 1 → slot S1
   ├── occurrence 2 → slot S2
   └── occurrence 3 → slot S3
```

---

# 47. Binding cardinality invariant

Todo placeholder runtime deberá tener un binding slot.

---

# 48. Reverse invariant

Todo required prepared binding slot deberá corresponder a una placeholder occurrence válida.

---

# 49. Structural literals

No forman parte del runtime parameter layout:

```text
NULL
TRUE
FALSE
DEFAULT
compile-time structural literal
```

---

# 50. Runtime values forbidden

El blueprint nunca contendrá:

```text
actual password
actual email
actual tenant id
actual user input
```

---

# 51. Sensitive parameters

Deberán conservar:

```text
ParameterSensitivity
```

para:

```text
logging
telemetry
diagnostics
error reporting
```

---

# 52. PS4 — Binding conversion planning

El sistema deberá describir cómo convertir:

```text
runtime semantic value
        ↓
driver-bindable value
```

---

# 53. Conversion plan

```php
final readonly class BindingConversionPlan
{
    public function __construct(
        public QueryType $sourceType,
        public DriverParameterType $targetType,
        public ConversionStrategyId $strategy,
        public ConversionRequirementSet $requirements,
    ) {}
}
```

---

# 54. Conversion plan ≠ conversion execution

La compilación sólo crea el plan.

La conversión real ocurre cuando existe el runtime value.

---

# 55. Examples

```text
UUID object
→ string/binary binding representation

DateTimeImmutable
→ target datetime representation

Enum
→ scalar database representation

JSON structure
→ encoded driver representation
```

---

# 56. Value Object boundary

Value Object mapping puede participar mediante descriptors.

Pero Prepared Statement Compilation no deberá inspeccionar entities.

---

# 57. Conversion failure

Si runtime conversion falla:

```text
BindingConversionException
```

durante binding.

No durante compilation si todavía no existe el valor.

---

# 58. Compile-time conversion validation

Sí puede comprobarse:

```text
conversion strategy exists
target driver type supported
required capability exists
```

---

# 59. Null handling

El parameter slot deberá conocer:

```text
nullable
non-nullable
unknown
```

---

# 60. Null conversion

`NULL` runtime deberá utilizar driver null binding semantics.

No:

```text
string "NULL"
```

---

# 61. Binary parameters

El blueprint deberá poder describir:

```text
binary
blob
stream
large object
```

sin contener el contenido.

---

# 62. Streaming parameter

```php
final readonly class StreamingBindingRequirement
{
    public function __construct(
        public StreamingBindingMode $mode,
        public ?int $knownLength,
    ) {}
}
```

---

# 63. Streaming parameter ≠ stream instance

El blueprint no almacena handles abiertos.

---

# 64. PS5 — Result descriptor compilation

Entrada:

```text
CompiledResultContract
```

Salida:

```text
PreparedResultDescriptor
```

---

# 65. Prepared result descriptor

```php
final readonly class PreparedResultDescriptor
{
    /**
     * @param list<PreparedResultColumn> $columns
     */
    public function __construct(
        public PreparedResultKind $kind,
        public array $columns,
        public ResultConversionPlan $conversion,
    ) {}
}
```

---

# 66. Result kinds

```php
enum PreparedResultKind
{
    case ROWS;
    case SCALAR;
    case AFFECTED_ROWS;
    case RETURNING_ROWS;
    case NO_RESULT;
    case DRIVER_SPECIFIC;
}
```

---

# 67. Result column

```php
final readonly class PreparedResultColumn
{
    public function __construct(
        public ResultColumnOrdinal $ordinal,
        public InternalSqlAlias $internalAlias,
        public ?DriverColumnLabel $driverLabel,
        public QueryType $semanticType,
        public ResultConversionDescriptor $conversion,
    ) {}
}
```

---

# 68. Result descriptor ≠ hydration

No contiene:

```text
EntityClass
Repository
UnitOfWork
IdentityMap
Model
RelationshipLoader
```

---

# 69. Driver result conversion

Puede describir:

```text
driver string → integer
driver string → decimal object
driver string → datetime
driver JSON → semantic JSON value
```

según Type System.

---

# 70. Hydration occurs later

```text
Driver Result
        ↓
Result Conversion
        ↓
Database Result
        ↓
Hydration System
        ↓
Entity
```

---

# 71. PS6 — Preparation requirements

El blueprint deberá describir los requisitos necesarios para preparar el statement.

---

# 72. StatementPreparationRequirements

```php
final readonly class StatementPreparationRequirements
{
    public function __construct(
        public ConnectionRoleRequirement $connectionRole,
        public TransactionRequirement $transaction,
        public SessionRequirementSet $session,
        public CapabilityRequirementSet $capabilities,
        public StatementPreparationMode $mode,
    ) {}
}
```

---

# 73. Connection role

Ejemplos:

```text
READ
WRITE
PRIMARY_REQUIRED
REPLICA_ALLOWED
ANY_COMPATIBLE
```

---

# 74. Role ≠ connection instance

El blueprint podrá requerir:

```text
WRITE
```

pero no contener:

```text
Connection #42
```

---

# 75. Transaction requirement

Ejemplos:

```php
enum StatementTransactionRequirement
{
    case NONE;
    case OPTIONAL;
    case REQUIRED;
    case EXISTING_REQUIRED;
    case FORBIDDEN;
}
```

---

# 76. Transaction descriptor ≠ transaction object

No se almacena una transacción viva.

---

# 77. Session requirements

Podrán incluir:

```text
required schema/search path
required SQL mode
required timezone semantics
required collation
required extension availability
required temporary object
```

cuando sean realmente necesarios.

---

# 78. Session state sensitivity

Algunos prepared statements pueden depender de session state.

Esto afecta reutilización.

---

# 79. Example

Un statement cuya resolución depende de:

```text
search_path
```

puede no ser reusable en otra session con diferente search path.

---

# 80. SessionRequirementSet

```php
final readonly class SessionRequirementSet
{
    /** @param list<SessionRequirement> $requirements */
    public function __construct(
        public array $requirements,
    ) {}
}
```

---

# 81. Preparation modes

```php
enum StatementPreparationMode
{
    case DRIVER_DEFAULT;
    case SERVER_SIDE;
    case CLIENT_SIDE;
    case EMULATED;
    case NATIVE;
}
```

---

# 82. Mode selection

No deberá basarse únicamente en preferencias.

Debe considerar:

```text
driver capability
platform capability
statement compatibility
security
parameter behavior
reuse policy
```

---

# 83. Emulated prepare

Si un driver ofrece emulated prepared statements, VoltStack deberá distinguirlo de native/server-side prepare.

---

# 84. Security invariant

Emulated preparation nunca deberá degradar parameter safety.

---

# 85. PS7 — Affinity analysis

Prepared statement reuse puede tener diferentes grados de afinidad.

---

# 86. Affinity dimensions

```text
platform
driver
connection family
specific connection
session
transaction
schema
tenant
runtime worker
```

---

# 87. PreparedStatementAffinity

```php
final readonly class PreparedStatementAffinity
{
    public function __construct(
        public PlatformAffinity $platform,
        public DriverAffinity $driver,
        public ConnectionAffinity $connection,
        public SessionAffinity $session,
        public TransactionAffinity $transaction,
        public TenantAffinity $tenant,
    ) {}
}
```

---

# 88. Blueprint affinity ≠ live statement affinity

El blueprint normalmente podrá ser más reusable que el live statement.

---

# 89. Example

```text
PreparedStatementBlueprint
→ reusable across compatible connections

Live PostgreSQL prepared statement
→ tied to one physical connection/session
```

---

# 90. Connection affinity levels

```php
enum ConnectionAffinityLevel
{
    case NONE;
    case COMPATIBLE_CONNECTION_CLASS;
    case CONNECTION_POOL;
    case PHYSICAL_CONNECTION;
}
```

---

# 91. Physical connection affinity

Un blueprint no deberá tenerla salvo que una característica target-specific lo requiera explícitamente.

Un live prepared statement sí puede tenerla.

---

# 92. Transaction affinity

Un statement preparado antes de una transacción puede seguir siendo reusable dependiendo del driver.

Esto será capability-driven.

---

# 93. Transaction-bound statement

Si un target requiere preparación dentro de una transacción específica:

```text
transaction affinity
=
required
```

y el live statement no podrá escapar de ella.

---

# 94. Tenant affinity

En multitenancy:

```text
same SQL
```

no significa necesariamente:

```text
same statement environment
```

---

# 95. Database-per-tenant

Un blueprint puede ser portable entre tenants si:

```text
same schema
same platform
same compiler capabilities
same statement dependencies
```

pero live statements nunca deberán compartirse entre conexiones físicas de tenants diferentes.

---

# 96. Schema-per-tenant

Si SQL usa qualified schema names:

```text
tenant-specific SQL
→ tenant-specific blueprint fingerprint
```

---

# 97. Search-path tenant routing

Si depende de session search path:

```text
tenant/session affinity
```

deberá declararse.

---

# 98. No hidden tenant state

Nunca:

```text
PreparedStatementCompiler
→ global current tenant
```

---

# 99. PS8 — Reuse analysis

El sistema deberá decidir qué nivel de reutilización es seguro.

---

# 100. Reuse classifications

```php
enum PreparedStatementReuseScope
{
    case NONE;
    case EXECUTION;
    case TRANSACTION;
    case CONNECTION;
    case CONNECTION_POOL;
    case APPLICATION;
}
```

---

# 101. Blueprint reuse

Normalmente:

```text
PreparedStatementBlueprint
```

podrá reutilizarse más ampliamente.

---

# 102. Live statement reuse

Normalmente:

```text
LivePreparedStatement
```

estará limitado a:

```text
physical connection
```

o:

```text
connection session
```

---

# 103. Application reuse

Sólo aplica al blueprint, no al statement nativo vivo.

---

# 104. Reuse conditions

```text
same prepared SQL
same placeholder model
same driver preparation profile
compatible capabilities
compatible session requirements
valid dependencies
compatible result contract
```

---

# 105. Reuse ≠ concurrent execution

Un statement reusable no es necesariamente:

```text
concurrently executable
```

---

# 106. Concurrent reuse capability

Debe modelarse separadamente.

```php
enum StatementConcurrentReuse
{
    case NOT_SUPPORTED;
    case SERIAL_ONLY;
    case DRIVER_DEFINED;
}
```

---

# 107. Live statement mutable state

Puede contener:

```text
current bindings
cursor
execution state
driver buffers
row position
error state
```

por lo que no debe asumirse thread-safe.

---

# 108. PS9 — Cacheability analysis

El blueprint deberá clasificarse.

---

# 109. Cacheability model

```php
enum PreparedStatementBlueprintCacheability
{
    case CACHEABLE;
    case CONTEXT_BOUND;
    case SPECIALIZED;
    case NON_CACHEABLE;
}
```

---

# 110. Cacheability descriptor

```php
final readonly class StatementCacheabilityDescriptor
{
    public function __construct(
        public PreparedStatementBlueprintCacheability $classification,
        public StatementCacheReasonSet $reasons,
        public CacheScope $scope,
    ) {}
}
```

---

# 111. CACHEABLE

Puede reutilizarse cuando sus dependencies siguen válidas.

---

# 112. CONTEXT_BOUND

Depende de un contexto específico como:

```text
platform profile
session profile
tenant schema profile
```

---

# 113. SPECIALIZED

Fue compilado para una representación especializada.

Ejemplo:

```text
specific parameter type specialization
```

---

# 114. NON_CACHEABLE

No deberá almacenarse en compiled blueprint cache.

---

# 115. Non-cacheable reasons

Ejemplos:

```text
unstable extension
ephemeral schema object
temporary relation
session-specific construct
non-deterministic preparation dependency
```

---

# 116. Cacheability ≠ prepared statement cache

Distinguir:

```text
Blueprint Cache
```

de:

```text
Live Prepared Statement Cache
```

---

# 117. Blueprint cache

Puede vivir a nivel:

```text
application
worker
compiled artifact cache
```

si es seguro.

---

# 118. Live statement cache

Deberá vivir normalmente dentro de:

```text
physical connection lifecycle
```

---

# 119. Statement cache architecture

```text
Application
    │
    └── Blueprint Cache
            │
            ▼
       Blueprint
            │
            ▼
Connection Pool
    │
    ├── Connection A
    │      └── Live Statement Cache A
    │
    └── Connection B
           └── Live Statement Cache B
```

---

# 120. Never share live statement across connections

```text
Statement(Connection A)
≠
Statement(Connection B)
```

aunque el SQL sea idéntico.

---

# 121. PS10 — Dependency compilation

El blueprint deberá registrar las dependencies que determinan su validez.

---

# 122. StatementDependencySet

```php
final readonly class StatementDependencySet
{
    /** @param list<StatementDependency> $dependencies */
    public function __construct(
        public array $dependencies,
    ) {}
}
```

---

# 123. Dependency categories

Podrán incluir:

```text
compiled SQL dependency
platform
dialect
driver preparation profile
placeholder behavior
parameter type mapping
result type mapping
schema objects
extensions
collations
session semantics
preparation capability snapshot
```

---

# 124. Schema dependencies

Sólo las realmente relevantes.

No:

```text
entire database schema
```

por defecto.

---

# 125. Driver dependency

Un blueprint preparado para:

```text
PDO PostgreSQL profile
```

no deberá asumirse compatible con cualquier driver PostgreSQL.

---

# 126. Preparation profile

```php
final readonly class DriverPreparationProfile
{
    public function __construct(
        public DriverId $driver,
        public PreparationCapabilitySnapshot $capabilities,
        public PlaceholderProfile $placeholders,
        public BindingProfile $bindings,
        public PreparationProfileFingerprint $fingerprint,
    ) {}
}
```

---

# 127. Driver ≠ platform

```text
PostgreSQL Platform
```

puede tener varios drivers/adapters.

---

# 128. Dependency classification

Podrán distinguirse:

```text
HARD
CONTEXT
SOFT_DIAGNOSTIC
```

---

# 129. Hard dependency

Cambio implica:

```text
blueprint invalid
```

---

# 130. Context dependency

El blueprint sigue siendo válido sólo en contextos compatibles.

---

# 131. Soft dependency

Útil para diagnostics sin invalidación automática.

---

# 132. PS11 — Fingerprint

Cada blueprint deberá tener fingerprint determinista.

---

# 133. Formula

```text
PreparedStatementFingerprint
=
Hash(
    CompiledQueryFingerprint
    +
    PreparedSqlFingerprint
    +
    ParameterLayoutFingerprint
    +
    ResultDescriptorFingerprint
    +
    PreparationRequirementFingerprint
    +
    DriverPreparationProfileFingerprint
    +
    RelevantDependencyFingerprint
    +
    PreparedStatementCompilerVersion
)
```

---

# 134. Runtime values excluded

No incluir:

```text
actual parameter values
current connection id
transaction id
request id
cursor id
```

---

# 135. Tenant identity

Sólo se incluirá cuando la representación o requirements sean realmente tenant-specific.

---

# 136. Fingerprint hierarchy

```text
SemanticFingerprint
        ↓
LogicalPlanFingerprint
        ↓
PhysicalPlanFingerprint
        ↓
ExecutionPlanFingerprint
        ↓
CompiledQueryFingerprint
        ↓
PreparedStatementFingerprint
```

Cada uno representa una capa diferente.

---

# 137. SQL hash ≠ prepared statement fingerprint

Mismo SQL puede requerir diferentes:

```text
parameter mappings
driver preparation modes
result conversion
session requirements
```

---

# 138. PS12 — Blueprint validation

Antes de publicar deberá comprobarse:

```text
SQL exists
single statement invariant
all placeholders mapped
all required slots mapped
driver binding types supported
conversion strategies exist
result descriptor matches command
requirements satisfiable
dependencies complete
cacheability valid
fingerprint complete
no runtime values embedded
```

---

# 139. Validation result

```php
final readonly class PreparedStatementValidationResult
{
    public function __construct(
        public bool $valid,
        public PreparedStatementDiagnosticSet $diagnostics,
    ) {}
}
```

---

# 140. Validation does not repair

```text
invalid blueprint
→ exception
```

---

# 141. PS13 — Freeze

Sólo después de validación:

```text
Mutable Builder
        ↓
Validation
        ↓
Freeze
        ↓
PreparedStatementBlueprint
```

---

# 142. No partially published blueprint

Un error en cualquier fase invalida el intento completo.

---

# 143. Atomic compilation

```text
PreparedStatementCompilation
=
all-or-nothing
```

---

# 144. Runtime preparation

El Execution Engine utilizará posteriormente:

```php
interface DriverStatementPreparer
{
    public function prepare(
        DriverConnection $connection,
        PreparedStatementBlueprint $blueprint,
    ): DriverPreparedStatement;
}
```

---

# 145. DriverPreparedStatement

```php
interface DriverPreparedStatement
{
    public function identity(): DriverStatementId;

    public function blueprint():
        PreparedStatementBlueprint;

    public function state():
        DriverPreparedStatementState;
}
```

---

# 146. Driver statement lifecycle

```text
CREATED
   ↓
PREPARED
   ↓
READY
   ↓
BOUND
   ↓
EXECUTING
   ↓
RESULT_ACTIVE
   ↓
RESETTABLE
   ↓
CLOSED
```

Driver capabilities pueden variar.

---

# 147. Blueprint lifecycle

Mucho más simple:

```text
BUILT
 ↓
VALIDATED
 ↓
FROZEN
```

No tiene execution state.

---

# 148. Server-side prepared statements

Algunas plataformas/drivers permiten statements registrados en el servidor.

Conceptualmente:

```text
Blueprint
    ↓
Connection
    ↓
PREPARE statement_name AS ...
    ↓
Server Statement Handle
```

---

# 149. Server statement identity

Será connection/session scoped.

---

# 150. Statement naming

Si se requieren nombres:

```text
VoltStackStatementName
```

deberá generarse deterministicamente dentro del scope apropiado o mediante allocator connection-local.

---

# 151. Statement name ≠ blueprint fingerprint

El fingerprint puede contribuir al nombre, pero ambos conceptos no son equivalentes.

---

# 152. Collision handling

Debe existir estrategia explícita.

No depender de truncamiento ingenuo.

---

# 153. Client-side preparation

En drivers con client-side prepare:

```text
Blueprint
    ↓
Driver Parser/Adapter
    ↓
Client Prepared Representation
```

sin cambiar el contrato público.

---

# 154. Emulated preparation

Debe quedar marcada en metadata para:

```text
diagnostics
security
telemetry
performance analysis
```

---

# 155. Native preparation preference

No deberá existir una regla universal:

```text
native always better
```

La política podrá considerar comportamiento real del driver.

---

# 156. Preparation policy

```php
interface StatementPreparationPolicy
{
    public function choose(
        PreparedStatementCompilationInput $input,
    ): StatementPreparationMode;
}
```

---

# 157. Policy constraints

La policy sólo puede elegir entre modos:

```text
valid
supported
security-compliant
```

---

# 158. Preference cannot create capability

```text
prefer SERVER_SIDE
```

no implica que exista.

---

# 159. Parameter binding runtime

Posteriormente:

```php
interface RuntimeParameterBinder
{
    public function bind(
        DriverPreparedStatement $statement,
        PreparedParameterLayout $layout,
        RuntimeBindingSet $values,
    ): BoundDriverStatement;
}
```

---

# 160. RuntimeBindingSet

```php
final readonly class RuntimeBindingSet
{
    /** @param array<string, mixed> $values */
    public function __construct(
        private array $values,
    ) {}
}
```

Esta estructura será operation-scoped.

---

# 161. Binding completeness

Antes de execution:

```text
all required runtime values present
```

deberá comprobarse.

---

# 162. Extra bindings

Por defecto deberán rechazarse o diagnosticarse explícitamente.

---

# 163. Missing bindings

Deberán fallar antes de enviar SQL al database cuando sea posible.

---

# 164. Sensitive values

Nunca deberán incluirse automáticamente en:

```text
statement fingerprint
cache key
logs
telemetry
exceptions
debug toolbar
```

---

# 165. Binding conversion pipeline

```text
Runtime Value
      ↓
Runtime Type Validation
      ↓
Binding Conversion Plan
      ↓
Driver Value
      ↓
Driver Binding
```

---

# 166. Parameter binding ≠ string interpolation

Siempre:

```text
value
→ driver binding
```

No:

```text
value
→ concatenate into SQL
```

---

# 167. Identifier parameters forbidden

SQL identifiers no son runtime value parameters.

Incorrecto:

```sql
SELECT * FROM ?
```

para parametrizar un table name.

---

# 168. Identifier resolution

Identifiers deberán haberse resuelto durante SQL Compilation.

---

# 169. List parameters

Un parámetro conceptual como:

```text
ids = [1,2,3]
```

no necesariamente corresponde a un único SQL placeholder.

---

# 170. Expansion belongs earlier

Si el target requiere:

```sql
IN (?, ?, ?)
```

la cardinalidad estructural deberá resolverse antes de Prepared Statement Compilation mediante un mecanismo explícito de parameter expansion/specialization.

---

# 171. Blueprint parameter cardinality

Una vez creado el blueprint, su placeholder cardinality deberá ser estable.

---

# 172. Array-native binding

Si una plataforma soporta array binding nativo:

```text
one semantic array parameter
→ one prepared slot
```

podrá utilizarse capability-aware.

---

# 173. Bulk parameters

Bulk operations deberán tener modelos específicos.

No deberán generar accidentalmente miles de placeholders sin budget accounting.

---

# 174. Parameter budget

El compiler deberá respetar:

```text
platform max parameters
driver max parameters
framework configured max
```

---

# 175. Parameter limit validation

Debe ocurrir antes de live prepare cuando sea posible.

---

# 176. Statement size

Igualmente:

```text
max SQL bytes
max statement complexity
```

podrán formar parte del preparation profile.

---

# 177. Error architecture

```text
PreparedStatementCompilationException
├── InvalidCompiledCommandException
├── UnsupportedPreparationCapabilityException
├── PreparedParameterLayoutException
├── UnsupportedDriverBindingException
├── BindingConversionPlanException
├── PreparedResultDescriptorException
├── StatementPreparationRequirementException
├── StatementAffinityException
├── StatementCacheabilityException
├── StatementDependencyException
├── PreparedStatementFingerprintException
├── PreparedStatementBudgetException
└── PreparedStatementInvariantException
```

---

# 178. Runtime errors are separate

Por ejemplo:

```text
DriverPrepareException
ParameterBindingException
StatementExecutionException
ResultFetchException
```

no son Prepared Statement Compilation errors.

---

# 179. Error translation

El driver layer podrá traducir errores nativos a exceptions estructuradas.

---

# 180. Preparation failure

Ejemplo:

```text
database rejects PREPARE
```

es runtime preparation failure.

No compilation failure salvo que pudiera conocerse por capabilities.

---

# 181. Diagnostics

Prepared Statement Compilation deberá poder mostrar:

```text
statement kind
placeholder style
parameter count
parameter semantic types
driver binding types
preparation mode
reuse scope
cacheability
affinity
dependencies
```

sin mostrar valores.

---

# 182. Explain example

```text
Prepared Statement Blueprint
────────────────────────────────────────

Kind:
    SELECT

Preparation:
    SERVER_SIDE

Placeholder Style:
    NUMBERED

Parameters:
    3

Reuse Scope:
    CONNECTION_POOL

Blueprint Cacheability:
    CACHEABLE

Live Statement Scope:
    PHYSICAL_CONNECTION

Transaction:
    OPTIONAL
```

---

# 183. Parameter diagnostics

```text
Slot S1
  Source: P1
  Placeholder: $1
  Type: UUID
  Driver Type: STRING
  Nullable: false
  Sensitive: false
```

No:

```text
Value: "550e8400..."
```

por defecto.

---

# 184. Source map integration

El blueprint deberá conservar la relación:

```text
Prepared Slot
        ↓
SQL Placeholder
        ↓
Rendered SQL Range
        ↓
Source Map
        ↓
Query/Semantic Source
```

---

# 185. Error location

Si el driver reporta error cerca de una posición SQL, VoltStack podrá relacionarlo con el source map original.

---

# 186. Statement cache key

Para live statements:

```text
LiveStatementCacheKey
=
PreparedStatementFingerprint
+
PhysicalConnectionPreparationProfile
+
RelevantSessionFingerprint
```

---

# 187. Physical connection identity

Dependiendo del driver, el cache puede estar implícitamente asociado a la conexión y no necesitar incluir su ID en la key.

---

# 188. Cache ownership

El live statement cache deberá pertenecer a:

```text
Connection
```

o a un componente con exactamente el mismo lifecycle.

---

# 189. Connection reset

Cuando una conexión se resetee:

```text
statement cache
```

deberá invalidarse si el driver invalida sus prepared statements.

---

# 190. Reconnect

Después de reconnect:

```text
old live prepared statements
=
invalid
```

---

# 191. Blueprint after reconnect

El blueprint puede continuar válido si:

```text
capabilities
schema dependencies
driver profile
```

siguen siendo compatibles.

---

# 192. Session reset

Debe existir una política explícita sobre qué session changes invalidan live statements.

---

# 193. Schema changes

Algunos databases invalidan automáticamente prepared statements.

Otros pueden mantenerlos con behavior diferente.

VoltStack no deberá asumir comportamiento universal.

---

# 194. Schema dependency validation

```text
Schema change
        ↓
Dependency Validator
        ↓
Blueprint/Live Statement Validity Decision
```

---

# 195. Reprepare strategy

Si un live statement deja de ser válido:

```text
discard live statement
        ↓
reuse blueprint if still valid
        ↓
prepare again
```

---

# 196. Recompile strategy

Si también el blueprint es inválido:

```text
discard blueprint
        ↓
return to SQL compilation/prepared compilation
```

---

# 197. Retry boundary

Reprepare no es lo mismo que query execution retry.

---

# 198. Automatic reprepare

Sólo deberá ocurrir cuando:

```text
safe
capability-aware
bounded
diagnosable
```

---

# 199. Infinite reprepare forbidden

Debe existir:

```text
reprepare budget
```

en execution.

---

# 200. Persistent runtime architecture

En FrankenPHP:

```text
Worker
├── Immutable PreparedStatementBlueprint Cache
│
├── Request A
│   └── RuntimeBindings A
│
└── Request B
    └── RuntimeBindings B
```

---

# 201. Runtime values must never enter shared blueprint

Éste es un invariante crítico.

---

# 202. Connection pool

```text
Worker
    │
    ├── Connection A
    │      └── Live Statement Cache A
    │
    └── Connection B
           └── Live Statement Cache B
```

---

# 203. Request isolation

```text
Request A bindings
```

nunca deberán permanecer en un statement reusable que posteriormente use:

```text
Request B
```

sin reset explícito.

---

# 204. Statement reset

Después de execution deberá existir un lifecycle de reset/close según driver.

---

# 205. Reset contract

```php
interface DriverPreparedStatementResetter
{
    public function reset(
        DriverPreparedStatement $statement,
    ): StatementResetResult;
}
```

---

# 206. Reset failure

Si no puede garantizarse un estado limpio:

```text
discard statement
```

---

# 207. Reuse safety rule

```text
Unable to prove clean reusable state
→ do not reuse
```

---

# 208. Cursor interaction

Un statement con cursor activo no podrá regresar al statement cache.

---

# 209. Streaming result

```text
PreparedStatement
    ↓
execute
    ↓
StreamingResult
```

puede transferir ownership temporal del statement/cursor al result object.

---

# 210. Cache return

Sólo cuando:

```text
stream closed
cursor closed
statement reset
```

podrá considerarse reusable.

---

# 211. Resource ownership

```text
Execution
    owns
        ↓
Prepared Statement Lease
```

hasta que el result lifecycle permita devolverlo.

---

# 212. Statement lease

```php
interface PreparedStatementLease
{
    public function statement(): DriverPreparedStatement;

    public function release(): void;

    public function discard(): void;
}
```

---

# 213. Lease ≠ blueprint

El blueprint no necesita lease.

---

# 214. Concurrency

Nunca compartir el mismo live statement mutable entre dos executions simultáneas salvo capability explícita extremadamente clara.

Default:

```text
one live statement
→ one active execution
```

---

# 215. Blueprint concurrency

Un immutable blueprint sí podrá consumirse concurrentemente.

---

# 216. Worker safety

```text
Immutable Blueprint
→ safe shared read

Live Prepared Statement
→ connection-scoped mutable resource
```

---

# 217. FrankenPHP

Default runtime:

```text
FrankenPHP
```

deberá beneficiarse de:

```text
compiled blueprint reuse
connection reuse
connection-local statement caches
```

sin state leakage.

---

# 218. RoadRunner

El mismo architecture deberá funcionar con RoadRunner.

---

# 219. OpenSwoole

El mismo architecture deberá soportar OpenSwoole sin depender de coroutine-local globals dentro del compiler.

---

# 220. Runtime abstraction

Prepared Statement Compilation no conocerá:

```text
FrankenPHP worker
RoadRunner worker
OpenSwoole coroutine
```

---

# 221. Runtime resource profile

Si un runtime afecta políticas de cache/reuse:

```text
RuntimeResourceProfile
```

podrá proporcionarse explícitamente a capas operativas posteriores.

No deberá alterar SQL semantics.

---

# 222. Security architecture

Prepared statements son una pieza fundamental contra SQL injection.

Pero:

```text
Prepared Statement
≠
Complete SQL Injection Protection
```

---

# 223. Values vs identifiers

Prepared statements protegen runtime values.

No solucionan identifiers dinámicos inseguros.

---

# 224. Compiler responsibility

Identifiers deberán haber sido estructurados y validados antes.

---

# 225. Raw SQL

Raw SQL con parámetros deberá producir:

```text
Raw SQL Structure
+
Explicit Parameter Descriptors
```

y pasar por el mismo Prepared Statement Compilation System.

---

# 226. No interpolation fallback

Si un driver no soporta cierto parameter style:

```text
adapt structurally
```

o:

```text
fail
```

Nunca:

```text
interpolate untrusted values
```

---

# 227. Emulated prepared statements

Si el driver emula prepare mediante escaping/interpolation internamente, esa responsabilidad permanece encapsulada en un driver considerado seguro y explícitamente capability-described.

El framework no deberá construir manualmente esa interpolación.

---

# 228. Sensitive statement diagnostics

No deberán contener:

```text
password
token
secret
PII parameter value
```

---

# 229. Statement fingerprint security

Fingerprints tampoco deberán derivarse directamente de secretos.

---

# 230. Hashing runtime values forbidden

Incluso:

```text
hash(secret)
```

puede filtrar información mediante correlación.

Por tanto los runtime values se excluyen.

---

# 231. Parameter type specialization

En algunos casos la estrategia de preparación puede depender del tipo runtime conocido antes de execution.

Esto deberá modelarse como:

```text
Explicit Prepared Statement Specialization
```

---

# 232. Specialization artifact

```php
final readonly class PreparedStatementSpecialization
{
    public function __construct(
        public PreparedStatementBlueprint $base,
        public ParameterTypeSpecializationSet $types,
        public PreparedStatementFingerprint $fingerprint,
    ) {}
}
```

---

# 233. Specialization ≠ value embedding

Especializar:

```text
P1 type = UUID
```

puede ser válido.

Especializar:

```text
P1 value = secret
```

no debe ocurrir ordinariamente.

---

# 234. Parameter-sensitive SQL

Si el SQL estructural cambia según cardinalidad/tipo:

```text
return to SQL Compilation specialization
```

No modificar SQL silenciosamente dentro del live binder.

---

# 235. Statement preparation extension system

Los compiler extensions podrán contribuir:

```text
driver binding descriptors
custom type preparation
custom statement requirements
custom result descriptors
```

mediante contracts controlados.

---

# 236. Prepared statement extension contract

```php
interface PreparedStatementCompilationExtension
{
    public function descriptor():
        PreparedStatementExtensionDescriptor;
}
```

---

# 237. Extension boundary

Una extensión no podrá:

```text
open connection
prepare live statement
bind runtime values
execute
```

durante blueprint compilation.

---

# 238. Custom driver preparation

Drivers podrán implementar:

```php
interface DriverPreparedStatementAdapter
{
    public function supports(
        PreparedStatementBlueprint $blueprint,
        DriverConnection $connection,
    ): bool;

    public function prepare(
        PreparedStatementBlueprint $blueprint,
        DriverConnection $connection,
    ): DriverPreparedStatement;
}
```

---

# 239. Adapter selection

Debe ser determinista y capability-aware.

---

# 240. No driver fallback through raw interpolation

Nunca.

---

# 241. Observability

Prepared statement telemetry podrá registrar:

```text
blueprint cache hit/miss
live statement cache hit/miss
prepare duration
reprepare count
statement reuse
statement discard
binding conversion duration
```

sin valores sensibles.

---

# 242. Telemetry separation

```text
Prepared Statement Compilation Telemetry
≠
Runtime Statement Telemetry
```

---

# 243. Compilation telemetry

Ejemplos:

```text
parameter_count
binding_strategy
preparation_mode
cacheability
reuse_scope
```

---

# 244. Runtime telemetry

Ejemplos:

```text
prepare_time
bind_time
execute_time
fetch_time
reuse_count
```

---

# 245. Debug toolbar

Podrá mostrar:

```text
SQL
parameter types
placeholder count
prepared mode
cache hits
statement reuse
```

pero no valores sensibles por defecto.

---

# 246. Performance architecture

Prepared statement compilation debe ser considerablemente más barata que:

```text
Semantic Analysis
Optimizer
Planner
SQL Compilation
```

---

# 247. Blueprint cache

Permitirá evitar repetir incluso esta pequeña fase.

---

# 248. Fast path

```text
CompiledDatabaseCommand
        ↓
PreparedStatementFingerprint Lookup
        ↓
Blueprint Cache Hit
        ↓
PreparedStatementBlueprint
```

---

# 249. Live statement fast path

```text
Blueprint
        ↓
Connection Statement Cache
        ↓
Live Statement Cache Hit
        ↓
Reset/Lease
        ↓
Bind
        ↓
Execute
```

---

# 250. Cache miss

```text
Blueprint
        ↓
Driver prepare
        ↓
Live statement
        ↓
Cache according to policy
```

---

# 251. Cache eviction

Live statement caches deberán tener límites:

```text
max statements
memory budget
idle policy
connection lifecycle
```

---

# 252. Eviction closes resources

Evict:

```text
statement
→ close/deallocate
```

cuando el driver lo requiera.

---

# 253. Server-side deallocation

Para server prepared statements:

```text
DEALLOCATE
```

o equivalente será responsabilidad del driver adapter.

---

# 254. Blueprint eviction

No requiere DB deallocation.

---

# 255. Resource governance

Live prepared statements consumen recursos:

```text
client memory
server memory
statement handles
server plan cache entries
```

por lo que el cache debe ser bounded.

---

# 256. No unbounded statement cache

Especialmente importante en persistent workers.

---

# 257. Statement cache pollution

Queries altamente especializadas pueden producir demasiados fingerprints.

La policy deberá poder decidir:

```text
do not cache
```

---

# 258. Cache admission policy

```php
interface PreparedStatementCacheAdmissionPolicy
{
    public function admit(
        PreparedStatementBlueprint $blueprint,
        StatementExecutionMetadata $metadata,
    ): bool;
}
```

---

# 259. Compilation vs runtime policy

La compilación podrá declarar:

```text
eligible
```

y runtime policy decidir:

```text
actually cache
```

---

# 260. Cacheable ≠ must cache

Principio importante:

```text
CACHEABLE
≠
CACHE REQUIRED
```

---

# 261. Server plan caching

El database puede tener su propio plan cache.

Eso es independiente de:

```text
VoltStack Blueprint Cache
VoltStack Live Statement Cache
```

---

# 262. Cache layers

```text
VoltStack Compiled Query Cache
        ↓
VoltStack Blueprint Cache
        ↓
VoltStack Connection Statement Cache
        ↓
Driver Cache
        ↓
Database Server Plan Cache
```

Cada nivel tiene semántica distinta.

---

# 263. No cache conflation

Nunca tratar todos esos caches como uno solo.

---

# 264. MySQL considerations

El adapter MySQL deberá describir explícitamente:

```text
placeholder behavior
native/emulated prepare capabilities
binding types
statement reuse
cursor behavior
server resource behavior
```

---

# 265. MariaDB considerations

MariaDB deberá tener perfil propio cuando diverja de MySQL.

---

# 266. PostgreSQL considerations

El adapter PostgreSQL podrá modelar:

```text
numbered placeholders
server-side statement naming
prepared plan lifecycle
session affinity
parameter type inference
```

según el driver concreto.

---

# 267. SQLite considerations

SQLite preparation será local a la conexión SQLite y deberá respetar:

```text
statement handle lifecycle
schema invalidation
binding semantics
```

sin asumir server-side preparation.

---

# 268. Platform abstraction

No deberán dispersarse:

```php
if ($database === 'mysql') { ... }
elseif ($database === 'postgres') { ... }
```

---

# 269. Preparation profile abstraction

Preferir:

```text
DriverPreparationProfile
+
Capabilities
+
Adapters
```

---

# 270. PDO abstraction

Si VoltStack utiliza PDO en determinados drivers:

```text
PDO
```

será detalle del Driver layer.

---

# 271. PreparedStatementBlueprint must not depend on PDOStatement

Esto mantiene la arquitectura portable.

---

# 272. Driver-native APIs

VoltStack podrá soportar drivers no basados en PDO sin rediseñar el compiler.

---

# 273. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Compiler\PreparedStatement
```

---

# 274. Estructura propuesta

```text
PreparedStatement/
├── Contract/
│   ├── PreparedStatementCompiler.php
│   ├── StatementPreparationPolicy.php
│   └── PreparedStatementCompilationExtension.php
│
├── Compilation/
│   ├── PreparedStatementCompilationContext.php
│   ├── PreparedStatementCompilationSession.php
│   ├── PreparedStatementCompilationPipeline.php
│   └── PreparedStatementCompilationResult.php
│
├── Blueprint/
│   ├── PreparedStatementBlueprint.php
│   ├── PreparedStatementBlueprintId.php
│   ├── PreparedStatementMetadata.php
│   └── PreparedStatementFingerprint.php
│
├── Sql/
│   ├── PreparedSqlDescriptor.php
│   ├── PreparedStatementKind.php
│   └── SqlTextFingerprint.php
│
├── Parameter/
│   ├── PreparedParameterLayout.php
│   ├── PreparedParameterSlot.php
│   ├── PreparedParameterSlotId.php
│   ├── DriverBindingDescriptor.php
│   ├── DriverBindingPosition.php
│   ├── DriverBindingMode.php
│   └── ParameterOccurrenceMap.php
│
├── Conversion/
│   ├── BindingConversionPlan.php
│   ├── BindingConversionStrategy.php
│   ├── ResultConversionPlan.php
│   └── ResultConversionDescriptor.php
│
├── Result/
│   ├── PreparedResultDescriptor.php
│   ├── PreparedResultKind.php
│   ├── PreparedResultColumn.php
│   └── ResultColumnOrdinal.php
│
├── Requirement/
│   ├── StatementPreparationRequirements.php
│   ├── StatementTransactionRequirement.php
│   ├── ConnectionRoleRequirement.php
│   ├── SessionRequirement.php
│   └── SessionRequirementSet.php
│
├── Capability/
│   ├── StatementPreparationCapabilitySnapshot.php
│   ├── StatementPreparationMode.php
│   └── DriverPreparationProfile.php
│
├── Affinity/
│   ├── PreparedStatementAffinity.php
│   ├── PlatformAffinity.php
│   ├── DriverAffinity.php
│   ├── ConnectionAffinity.php
│   ├── SessionAffinity.php
│   ├── TransactionAffinity.php
│   └── TenantAffinity.php
│
├── Reuse/
│   ├── PreparedStatementReuseScope.php
│   ├── StatementConcurrentReuse.php
│   └── PreparedStatementReuseAnalyzer.php
│
├── Cache/
│   ├── StatementCacheabilityDescriptor.php
│   ├── PreparedStatementBlueprintCacheability.php
│   ├── PreparedStatementBlueprintCache.php
│   └── PreparedStatementCacheAdmissionPolicy.php
│
├── Dependency/
│   ├── StatementDependency.php
│   ├── StatementDependencySet.php
│   ├── StatementDependencyType.php
│   └── StatementDependencyValidator.php
│
├── Specialization/
│   ├── PreparedStatementSpecialization.php
│   └── ParameterTypeSpecializationSet.php
│
├── Validation/
│   ├── PreparedStatementBlueprintValidator.php
│   └── PreparedStatementValidationResult.php
│
├── Diagnostic/
│   ├── PreparedStatementDiagnostic.php
│   ├── PreparedStatementDiagnosticSet.php
│   └── PreparedStatementCompilationTrace.php
│
└── Exception/
    ├── PreparedStatementCompilationException.php
    ├── InvalidCompiledCommandException.php
    ├── UnsupportedPreparationCapabilityException.php
    ├── PreparedParameterLayoutException.php
    ├── UnsupportedDriverBindingException.php
    ├── BindingConversionPlanException.php
    ├── PreparedResultDescriptorException.php
    ├── StatementPreparationRequirementException.php
    ├── StatementAffinityException.php
    ├── StatementCacheabilityException.php
    ├── StatementDependencyException.php
    ├── PreparedStatementFingerprintException.php
    ├── PreparedStatementBudgetException.php
    └── PreparedStatementInvariantException.php
```

---

# 275. Runtime namespace

La preparación viva deberá residir fuera del compiler, por ejemplo:

```text
VoltStack\Quantum\Database\Execution\Statement
```

con:

```text
DriverStatementPreparer
DriverPreparedStatement
RuntimeParameterBinder
PreparedStatementLease
LivePreparedStatementCache
```

---

# 276. Dependency direction

```text
Prepared Statement Compiler
        ↓
Prepared Statement Contracts
```

y:

```text
Execution Engine
        ↓
Prepared Statement Blueprint
        ↓
Driver
```

Nunca:

```text
Prepared Statement Compiler
        ↓
Execution Engine
```

---

# 277. ORM independence

El sistema no deberá conocer:

```text
Entity
Model
Repository
EntityManager
UnitOfWork
IdentityMap
```

---

# 278. Transaction independence

La compilación puede describir transaction requirements.

No puede controlar la transacción.

---

# 279. Connection independence

La compilación puede describir connection requirements.

No puede resolver una conexión viva.

---

# 280. Security independence

La compilación preserva security metadata.

No vuelve a decidir authorization.

---

# 281. Invariantes arquitectónicos

## DB-PSTMT-001

Prepared Statement Compilation será distinta de live statement preparation.

## DB-PSTMT-002

`CompiledDatabaseCommand` será distinto de `PreparedStatementBlueprint`.

## DB-PSTMT-003

`PreparedStatementBlueprint` será distinto de `DriverPreparedStatement`.

## DB-PSTMT-004

`DriverPreparedStatement` será distinto de `BoundDriverStatement`.

## DB-PSTMT-005

Prepared Statement Compiler no abrirá conexiones.

## DB-PSTMT-006

Prepared Statement Compiler no ejecutará SQL.

## DB-PSTMT-007

Prepared Statement Compiler no bindeará runtime values.

## DB-PSTMT-008

Prepared Statement Compiler no hará fetch de results.

## DB-PSTMT-009

Prepared Statement Compiler no controlará transactions.

## DB-PSTMT-010

Blueprint será immutable.

## DB-PSTMT-011

Blueprint no contendrá runtime values.

## DB-PSTMT-012

Blueprint no contendrá live connection.

## DB-PSTMT-013

Blueprint no contendrá live transaction.

## DB-PSTMT-014

Blueprint no contendrá cursor.

## DB-PSTMT-015

Blueprint no contendrá driver statement handle.

## DB-PSTMT-016

Un command producirá un statement blueprint por defecto.

## DB-PSTMT-017

Hidden multi-statement compilation estará prohibida.

## DB-PSTMT-018

Multi-command execution deberá modelarse explícitamente.

## DB-PSTMT-019

Prepared SQL no podrá cambiar query semantics.

## DB-PSTMT-020

Prepared SQL no podrá cambiar mutation semantics.

## DB-PSTMT-021

Prepared SQL no podrá eliminar security predicates.

## DB-PSTMT-022

Prepared SQL no podrá cambiar locking semantics.

## DB-PSTMT-023

`ParameterId` será distinto de `PreparedParameterSlotId`.

## DB-PSTMT-024

`PreparedParameterSlotId` será distinto de `DriverBindingPosition`.

## DB-PSTMT-025

`SqlPlaceholderId` será distinto de `DriverBindingPosition`.

## DB-PSTMT-026

Repeated semantic parameters podrán tener múltiples occurrences.

## DB-PSTMT-027

Cada runtime placeholder tendrá binding descriptor.

## DB-PSTMT-028

Cada required binding slot tendrá source válido.

## DB-PSTMT-029

Structural literals no serán runtime bindings.

## DB-PSTMT-030

Sensitive values no formarán parte del blueprint.

## DB-PSTMT-031

Binding conversion plan será distinto de runtime conversion.

## DB-PSTMT-032

Null binding usará driver null semantics.

## DB-PSTMT-033

Streaming binding descriptor no contendrá live stream.

## DB-PSTMT-034

Result descriptor será distinto de hydration plan.

## DB-PSTMT-035

Prepared Statement Compiler no hidratará entities.

## DB-PSTMT-036

Connection role requirement será distinto de connection instance.

## DB-PSTMT-037

Transaction requirement será distinto de transaction instance.

## DB-PSTMT-038

Session requirements serán explícitos.

## DB-PSTMT-039

Preparation mode será capability-aware.

## DB-PSTMT-040

Preference no creará capability.

## DB-PSTMT-041

Emulated prepare no podrá degradar parameter safety.

## DB-PSTMT-042

Blueprint affinity será explícita.

## DB-PSTMT-043

Blueprint affinity será distinta de live statement affinity.

## DB-PSTMT-044

Live statement será connection-scoped cuando el driver lo requiera.

## DB-PSTMT-045

Live statement nunca se compartirá entre physical connections.

## DB-PSTMT-046

Tenant state no será global.

## DB-PSTMT-047

Tenant-specific representation deberá afectar fingerprint.

## DB-PSTMT-048

Reuse scope será explícito.

## DB-PSTMT-049

Reusable será distinto de concurrently reusable.

## DB-PSTMT-050

Live mutable statement no se asumirá thread-safe.

## DB-PSTMT-051

Blueprint cache será distinto de live statement cache.

## DB-PSTMT-052

Live statement cache será connection-lifecycle scoped.

## DB-PSTMT-053

Blueprint cache podrá tener scope más amplio.

## DB-PSTMT-054

Cacheable será distinto de must-cache.

## DB-PSTMT-055

Statement cache será bounded.

## DB-PSTMT-056

Connection reconnect invalidará live statements anteriores.

## DB-PSTMT-057

Blueprint podrá sobrevivir reconnect si dependencies siguen válidas.

## DB-PSTMT-058

Connection reset invalidará live statements cuando sea necesario.

## DB-PSTMT-059

Session-sensitive statements declararán session dependencies.

## DB-PSTMT-060

Schema-sensitive statements declararán schema dependencies.

## DB-PSTMT-061

Driver preparation profile será explícito.

## DB-PSTMT-062

Driver será distinto de platform.

## DB-PSTMT-063

Statement dependencies serán estructuradas.

## DB-PSTMT-064

Dependency tracking no se inferirá parseando SQL.

## DB-PSTMT-065

PreparedStatementFingerprint será determinista.

## DB-PSTMT-066

Runtime values estarán excluidos del fingerprint.

## DB-PSTMT-067

Connection IDs estarán excluidos del blueprint fingerprint salvo arquitectura explícita que lo requiera.

## DB-PSTMT-068

Request IDs estarán excluidos del fingerprint.

## DB-PSTMT-069

Transaction IDs estarán excluidos del fingerprint.

## DB-PSTMT-070

SQL hash será distinto de PreparedStatementFingerprint.

## DB-PSTMT-071

Blueprint validation ocurrirá antes de freeze.

## DB-PSTMT-072

Validation no reparará silenciosamente artifacts.

## DB-PSTMT-073

Prepared statement compilation será atomic.

## DB-PSTMT-074

No se publicará blueprint parcial.

## DB-PSTMT-075

Live preparation pertenecerá al Execution/Driver layer.

## DB-PSTMT-076

DriverPreparedStatement no será compartido globalmente.

## DB-PSTMT-077

Server-side statement identity será connection/session scoped.

## DB-PSTMT-078

Statement name será distinto del blueprint fingerprint.

## DB-PSTMT-079

Statement naming tendrá collision handling.

## DB-PSTMT-080

Client-side preparation mantendrá el mismo framework contract.

## DB-PSTMT-081

Native prepare no será universalmente obligatorio.

## DB-PSTMT-082

Preparation policy sólo elegirá modos válidos.

## DB-PSTMT-083

Runtime bindings serán operation-scoped.

## DB-PSTMT-084

Missing bindings fallarán antes de execution cuando sea posible.

## DB-PSTMT-085

Sensitive values no se registrarán automáticamente.

## DB-PSTMT-086

Binding nunca será SQL string interpolation.

## DB-PSTMT-087

Identifiers no serán value parameters.

## DB-PSTMT-088

List expansion será estructural y explícita.

## DB-PSTMT-089

Blueprint placeholder cardinality será estable.

## DB-PSTMT-090

Array-native binding requerirá capability.

## DB-PSTMT-091

Parameter limits se validarán antes de live prepare cuando sea posible.

## DB-PSTMT-092

Statement size limits serán respetados.

## DB-PSTMT-093

Compilation errors serán distintos de runtime prepare errors.

## DB-PSTMT-094

Runtime prepare errors serán distintos de binding errors.

## DB-PSTMT-095

Binding errors serán distintos de execution errors.

## DB-PSTMT-096

Execution errors serán distintos de fetch errors.

## DB-PSTMT-097

Diagnostics no mostrarán sensitive values.

## DB-PSTMT-098

Prepared slots conservarán source-map provenance.

## DB-PSTMT-099

Live statement cache key incluirá preparation compatibility.

## DB-PSTMT-100

Live statement cache ownership seguirá connection lifecycle.

## DB-PSTMT-101

Live statement eviction liberará recursos.

## DB-PSTMT-102

Server-side statement eviction realizará deallocation cuando corresponda.

## DB-PSTMT-103

Blueprint eviction no requerirá database deallocation.

## DB-PSTMT-104

Live statement cache será resource-governed.

## DB-PSTMT-105

Statement cache pollution podrá evitarse mediante admission policy.

## DB-PSTMT-106

Database server plan cache será distinto de VoltStack caches.

## DB-PSTMT-107

MySQL preparation tendrá profile propio.

## DB-PSTMT-108

MariaDB preparation podrá divergir de MySQL.

## DB-PSTMT-109

PostgreSQL preparation tendrá profile propio.

## DB-PSTMT-110

SQLite preparation no asumirá server-side statements.

## DB-PSTMT-111

Platform-specific behavior se encapsulará en profiles/adapters.

## DB-PSTMT-112

Blueprint no dependerá de PDOStatement.

## DB-PSTMT-113

Driver-native APIs serán soportables.

## DB-PSTMT-114

Prepared Statement Compiler no conocerá ORM.

## DB-PSTMT-115

Prepared Statement Compiler no conocerá HTTP.

## DB-PSTMT-116

Prepared Statement Compiler no conocerá current request.

## DB-PSTMT-117

Prepared Statement Compiler no conocerá current worker.

## DB-PSTMT-118

Shared blueprint será persistent-worker-safe.

## DB-PSTMT-119

Operation bindings no sobrevivirán request boundaries.

## DB-PSTMT-120

Connection statement caches no mezclarán physical connections.

## DB-PSTMT-121

Statement con active cursor no volverá al cache.

## DB-PSTMT-122

Streaming result conservará resource ownership.

## DB-PSTMT-123

Statement sólo volverá al cache después de reset válido.

## DB-PSTMT-124

Reset failure implicará discard.

## DB-PSTMT-125

Unclean statement no será reusable.

## DB-PSTMT-126

Live statement no tendrá dos active executions por defecto.

## DB-PSTMT-127

Immutable blueprint podrá consumirse concurrentemente.

## DB-PSTMT-128

FrankenPHP reuse no podrá causar state leakage.

## DB-PSTMT-129

RoadRunner reuse deberá respetar las mismas fronteras.

## DB-PSTMT-130

OpenSwoole reuse deberá respetar las mismas fronteras.

## DB-PSTMT-131

Runtime-specific behavior no contaminará compiler semantics.

## DB-PSTMT-132

Prepared statements no sustituirán identifier security.

## DB-PSTMT-133

Raw SQL utilizará el mismo binding architecture.

## DB-PSTMT-134

No existirá unsafe interpolation fallback.

## DB-PSTMT-135

Sensitive values no participarán en fingerprints.

## DB-PSTMT-136

Sensitive value hashes tampoco se incluirán ordinariamente.

## DB-PSTMT-137

Parameter type specialization será explícita.

## DB-PSTMT-138

Specialization no implicará embedding de runtime values.

## DB-PSTMT-139

Structural SQL specialization volverá al SQL compilation layer.

## DB-PSTMT-140

Prepared statement extensions no abrirán connections.

## DB-PSTMT-141

Prepared statement extensions no ejecutarán SQL.

## DB-PSTMT-142

Prepared statement extensions no bindearán values.

## DB-PSTMT-143

Driver preparation adapters serán capability-aware.

## DB-PSTMT-144

Driver adapters no utilizarán unsafe interpolation fallback.

## DB-PSTMT-145

Compilation telemetry será distinta de runtime telemetry.

## DB-PSTMT-146

Telemetry no expondrá sensitive bindings.

## DB-PSTMT-147

Blueprint cache hit no implicará live statement cache hit.

## DB-PSTMT-148

Live statement cache miss no requerirá necesariamente SQL recompilation.

## DB-PSTMT-149

Invalid live statement podrá reprepare desde blueprint.

## DB-PSTMT-150

Invalid blueprint requerirá recompilation upstream.

## DB-PSTMT-151

Reprepare será distinto de execution retry.

## DB-PSTMT-152

Automatic reprepare será bounded.

## DB-PSTMT-153

Resource ownership será explícito.

## DB-PSTMT-154

PreparedStatementLease será distinto del statement blueprint.

## DB-PSTMT-155

Connection/session affinity será respetada.

## DB-PSTMT-156

Security requirements del compiled command serán preservados.

## DB-PSTMT-157

Result shape del compiled command será preservado.

## DB-PSTMT-158

Parameter semantics del compiled command serán preservadas.

## DB-PSTMT-159

Prepared statement compilation no cambiará observable query semantics.

## DB-PSTMT-160

La preparación real será siempre responsabilidad de una conexión/driver compatible.

---

# 282. Invariante maestro

```text
Semantics(
    PreparedStatementBlueprint
)
=
Semantics(
    CompiledDatabaseCommand
)
```

---

# 283. Invariante de seguridad

```text
Runtime Value
        ↓
Runtime Binding
        ↓
Binding Conversion
        ↓
Driver Binding
```

Nunca:

```text
Runtime Value
        ↓
SQL Concatenation
```

---

# 284. Invariante de lifecycle

```text
Blueprint Lifecycle
≠
Live Statement Lifecycle
```

---

# 285. Invariante de caché

```text
Blueprint Cache
≠
Connection Statement Cache
≠
Driver Statement Cache
≠
Database Plan Cache
```

---

# 286. Invariante de conexión

```text
LivePreparedStatement
belongs to
PhysicalConnection / Session
```

según las capabilities del driver.

---

# 287. Invariante de persistent runtime

```text
Shared Across Requests:
    immutable blueprint

Never Shared As Request State:
    runtime values
    active statement bindings
    active cursor
    transaction state
```

---

# 288. Invariante de preparación

```text
PreparedStatementCompiler
describes preparation.

DriverStatementPreparer
performs preparation.
```

---

# 289. Flujo completo

```text
CompiledDatabaseCommand
        │
        ▼
PreparedStatementCompiler
        │
        ├── validate command
        ├── resolve preparation capabilities
        ├── compile parameter slots
        ├── compile conversion plans
        ├── compile result descriptor
        ├── derive requirements
        ├── derive affinity
        ├── derive reuse scope
        ├── classify cacheability
        ├── collect dependencies
        └── fingerprint
        │
        ▼
PreparedStatementBlueprint
        │
        │
        ├──────────── Blueprint Cache
        │
        ▼
Execution Engine
        │
        ▼
Connection Resolver
        │
        ▼
Physical Connection
        │
        ▼
Connection Statement Cache
        │
   ┌────┴────┐
   │         │
  HIT       MISS
   │         │
   │         ▼
   │    Driver Prepare
   │         │
   └────┬────┘
        ▼
DriverPreparedStatement
        │
        ▼
PreparedStatementLease
        │
        ▼
Runtime Binding Set
        │
        ▼
Binding Conversion
        │
        ▼
Driver Binding
        │
        ▼
Execute
        │
        ▼
Result / Cursor
        │
        ▼
Close / Drain
        │
        ▼
Reset
        │
   ┌────┴────┐
   │         │
 reusable   unsafe
   │         │
   ▼         ▼
 cache     discard
```

---

# 290. Fórmula arquitectónica

```text
Prepared Statement Compilation System
=
Compiled Command Validation
+
Preparation Capability Resolution
+
Parameter Slot Compilation
+
Binding Conversion Planning
+
Result Descriptor Compilation
+
Preparation Requirements
+
Affinity Analysis
+
Reuse Analysis
+
Cacheability Classification
+
Dependency Compilation
+
Deterministic Fingerprinting
+
Blueprint Validation
+
Immutable Blueprint Publication
```

---

# 291. Principio final

VoltStack deberá poder reutilizar agresivamente:

```text
knowledge
```

sin reutilizar incorrectamente:

```text
state
```

Por tanto:

```text
PreparedStatementBlueprint
=
Reusable Knowledge
```

mientras:

```text
DriverPreparedStatement
=
Connection-Bound Runtime Resource
```

y:

```text
RuntimeBindings
=
Operation-Bound State
```

---

# 292. Regla maestra

```text
Compile once when safe.
Prepare per compatible connection.
Bind per execution.
Never leak runtime state.
```

---

# 293. Resultado arquitectónico

Con este sistema, VoltStack obtiene una separación completa:

```text
SQL Generation
      ↓
CompiledDatabaseCommand
      ↓
Prepared Statement Compilation
      ↓
PreparedStatementBlueprint
      ↓
Runtime Preparation
      ↓
DriverPreparedStatement
      ↓
Runtime Binding
      ↓
Execution
```

permitiendo simultáneamente:

```text
security
prepared statement reuse
compiled artifact caching
connection-local statement caching
driver portability
persistent-worker safety
clear resource ownership
precise invalidation
high-performance execution
```

---

# 294. Relación con el siguiente sistema

El `PreparedStatementBlueprint` es uno de los artifacts candidatos a reutilización.

Pero VoltStack también necesita evitar repetir:

```text
Query AST
    ↓
Semantic Analysis
    ↓
Optimization
    ↓
Planning
    ↓
SQL Compilation
    ↓
Prepared Statement Compilation
```

cuando la estructura de una query ya fue compilada previamente.

Esto conduce al último componente del bloque SQL Compiler:

```text
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

---

# 295. Siguiente documento

```text
75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

deberá definir:

```text
Compiled Query Cache
├── cache identity
├── cache key architecture
├── cache entries
├── compiled artifact hierarchy
├── dependency tracking
├── invalidation
├── schema-aware invalidation
├── capability-aware invalidation
├── extension-aware invalidation
├── compiler-version invalidation
├── logical/physical/execution/SQL cache boundaries
├── prepared blueprint caching
├── parameter-sensitive specialization
├── tenant isolation
├── runtime isolation
├── cache admission
├── eviction
├── memory governance
├── persistent-worker reuse
├── distributed cache considerations
├── observability
└── correctness invariants
```

manteniendo siempre:

```text
Compiled Query Cache
≠
Query Result Cache
≠
ORM Entity Cache
≠
Database Server Plan Cache
≠
Live Prepared Statement Cache
```

---

# 296. Cierre del documento

La separación introducida en este documento establece una frontera fundamental para VoltStack:

```text
Compilation artifacts may be shared.

Connection resources must be owned.

Runtime values must remain isolated.
```

Ésta será una de las bases para que `VoltStack/Quantum/Database` pueda aprovechar FrankenPHP y otros runtimes persistentes sin convertir la reutilización de recursos en una fuente de contaminación entre requests.