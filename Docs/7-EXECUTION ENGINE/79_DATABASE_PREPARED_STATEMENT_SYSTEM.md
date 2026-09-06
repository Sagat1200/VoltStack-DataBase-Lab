# 79_DATABASE_PREPARED_STATEMENT_SYSTEM.md

# VoltStack Quantum Database
## Prepared Statement System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 79 — Prepared Statement System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Prepared Statement System` define la infraestructura responsable de convertir un:

```text
CompiledDatabaseCommand
```

en un recurso ejecutable preparado y asociado a una conexión concreta:

```text
CompiledDatabaseCommand
        ↓
Prepared Statement System
        ↓
PreparedStatementLease
        ↓
DriverPreparedStatement
```

Formalmente:

```text
Prepare(
    CompiledDatabaseCommand,
    ConnectionLease,
    PreparedStatementContext
)
    →
PreparedStatementLease
```

El sistema administra:

- preparación de statements;
- identidad del statement;
- afinidad con conexión;
- afinidad con sesión;
- compatibilidad con driver;
- lifecycle;
- leases;
- ownership;
- reutilización;
- caching;
- reset;
- invalidación;
- cierre;
- result draining;
- cursor cleanup;
- health;
- concurrency;
- seguridad;
- telemetría;
- aislamiento bajo runtimes persistentes.

---

# 2. Principio fundamental

```text
Prepared Statement
=
Connection-Bound Executable Database Resource
```

No:

```text
Prepared Statement
=
Compiled Query
```

Tampoco:

```text
Prepared Statement
=
Execution Instance
```

Por tanto:

```text
CompiledDatabaseCommand
≠
PreparedStatement
≠
StatementExecutionInstance
```

---

# 3. Separación arquitectónica

```text
Compilation Time
──────────────────────────────────────

CompilableDatabaseOperation
        ↓
SQL Compiler
        ↓
CompiledDatabaseCommand


Runtime
──────────────────────────────────────

CompiledDatabaseCommand
        ↓
Prepared Statement System
        ↓
PreparedStatement
        ↓
Parameter Binding
        ↓
Statement Execution
```

---

# 4. Compiled command vs prepared statement

Un `CompiledDatabaseCommand` puede ser:

```text
immutable
reusable
shareable
connection-independent
```

Un `PreparedStatement` normalmente es:

```text
mutable runtime resource
driver-bound
connection-bound
session-sensitive
non-portable between connections
```

---

# 5. Relación con Statement Execution

El documento 78 definió:

```text
StatementExecutor
        ↓
PreparedStatementSystem
        ↓
PreparedStatementLease
```

`StatementExecutor` coordina la ejecución.

`PreparedStatementSystem` administra el recurso preparado.

---

# 6. Responsabilidades

El sistema deberá responder:

1. ¿Debe prepararse este command?
2. ¿Puede utilizarse preparación nativa?
3. ¿Existe un statement reusable compatible?
4. ¿A qué conexión pertenece?
5. ¿Está actualmente leased?
6. ¿Puede reutilizarse?
7. ¿Debe resetearse?
8. ¿Tiene cursor/result pendiente?
9. ¿Sigue siendo válido?
10. ¿Debe cerrarse?
11. ¿Debe invalidarse?
12. ¿Puede almacenarse en cache?
13. ¿Puede sobrevivir entre requests?
14. ¿Puede ejecutarse concurrentemente?
15. ¿Qué ocurre si la conexión muere?

---

# 7. No responsabilidades

No pertenece aquí:

```text
Query AST interpretation
Semantic Analysis
Optimization
Physical Planning
SQL Generation
Runtime value resolution
ORM hydration
Transaction policy
Retry policy
Entity lifecycle
```

---

# 8. Contrato principal

```php
namespace VoltStack\Quantum\Database\Execution\Prepared\Contract;

interface PreparedStatementManager
{
    public function acquire(
        PreparedStatementRequest $request,
    ): PreparedStatementLease;
}
```

---

# 9. PreparedStatementRequest

```php
final readonly class PreparedStatementRequest
{
    public function __construct(
        public CompiledDatabaseCommand $command,
        public ConnectionLease $connection,
        public PreparedStatementContext $context,
        public PreparedStatementPolicy $policy,
    ) {}
}
```

---

# 10. PreparedStatementContext

Puede contener:

```php
final readonly class PreparedStatementContext
{
    public function __construct(
        public DriverIdentity $driver,
        public PlatformIdentity $platform,
        public SessionIdentity $session,
        public DriverCapabilitySnapshot $capabilities,
        public PreparedStatementExecutionOptions $options,
    ) {}
}
```

---

# 11. Contexto explícito

No deberán resolverse mediante globals:

```text
current connection
current driver
current transaction
current tenant
current session
current request
```

---

# 12. DriverPreparedStatement

El recurso de bajo nivel será abstraído:

```php
interface DriverPreparedStatement
{
    public function identity(): DriverStatementIdentity;

    public function state(): DriverStatementState;

    public function close(): void;
}
```

---

# 13. No PDOStatement en core

El core no deberá depender directamente de:

```php
PDOStatement
```

Podrá existir:

```text
PdoPreparedStatementAdapter
```

en el driver correspondiente.

---

# 14. PreparedStatement

VoltStack podrá envolver el recurso del driver:

```php
final class PreparedStatement
{
    public function __construct(
        private readonly PreparedStatementIdentity $identity,
        private readonly PreparedStatementDescriptor $descriptor,
        private readonly DriverPreparedStatement $driverStatement,
        private PreparedStatementState $state,
    ) {}
}
```

---

# 15. PreparedStatementDescriptor

Describe propiedades invariantes:

```php
final readonly class PreparedStatementDescriptor
{
    public function __construct(
        public PreparedStatementIdentity $identity,
        public CompiledCommandFingerprint $command,
        public ConnectionIdentity $connection,
        public SessionCompatibilityKey $session,
        public DriverIdentity $driver,
        public PlaceholderContract $placeholders,
        public ResultContractFingerprint $result,
        public StatementOptionFingerprint $options,
    ) {}
}
```

---

# 16. Descriptor ≠ resource

```text
PreparedStatementDescriptor
=
metadata
```

```text
PreparedStatement
=
runtime resource
```

---

# 17. PreparedStatementIdentity

No deberá ser:

```text
hash(SQL)
```

solamente.

---

# 18. Identity model

Conceptualmente:

```text
PreparedStatementIdentity
=
CommandFingerprint
+
ConnectionIdentity
+
SessionCompatibility
+
DriverPreparationContract
+
PreparationOptions
```

---

# 19. SQL text insufficient

Dos commands con SQL idéntico pueden diferir en:

```text
parameter types
result contract
driver mode
placeholder contract
statement options
extension behavior
session requirements
```

---

# 20. Connection affinity

Un prepared statement normalmente pertenece a:

```text
Connection X
```

No puede utilizarse sobre:

```text
Connection Y
```

sin nueva preparación.

---

# 21. Fundamental invariant

```text
PreparedStatement(Connection A)
≠
PreparedStatement(Connection B)
```

aunque ambos provengan del mismo command.

---

# 22. Example

```text
CompiledCommand C1
       │
       ├── Connection A
       │      ↓
       │   PreparedStatement P1
       │
       └── Connection B
              ↓
           PreparedStatement P2
```

---

# 23. ConnectionIdentity

Deberá ser estable durante la vida del recurso físico.

No debe confundirse con:

```text
connection configuration name
```

Ejemplo:

```text
"default"
```

no identifica necesariamente una conexión física.

---

# 24. Physical connection identity

Puede existir:

```text
ConnectionInstanceId
```

asignado cuando se crea la conexión.

---

# 25. Reconnection

Si:

```text
ConnectionInstance A
```

se desconecta y se reconstruye:

```text
ConnectionInstance B
```

sus prepared statements anteriores quedan inválidos.

---

# 26. Session affinity

Algunos prepared statements pueden depender de:

```text
session settings
temporary schema
search path
SQL mode
charset
collation
extension registration
temporary objects
session-defined functions
```

---

# 27. SessionCompatibilityKey

Por ello puede existir:

```php
final readonly class SessionCompatibilityKey
{
    public function __construct(
        public string $fingerprint,
    ) {}
}
```

---

# 28. Session identity ≠ request identity

Un statement no deberá invalidarse sólo porque cambió el request si la sesión continúa siendo compatible.

---

# 29. Tenant boundary

Sin embargo, si el prepared statement depende de:

```text
tenant-specific schema
search_path
attached database
temporary table
session function
```

el cambio de tenant puede volverlo incompatible.

---

# 30. No hidden tenant reuse

Nunca:

```text
Tenant A prepared statement
→ Tenant B
```

si existe cualquier dependencia tenant-scoped incompatible.

---

# 31. Preparation capability

El sistema deberá distinguir:

```text
NATIVE_PREPARED
EMULATED_PREPARED
DIRECT_EXECUTION
UNSUPPORTED
```

---

# 32. Native preparation

```text
SQL
 ↓
database prepare
 ↓
server/client prepared resource
```

---

# 33. Emulated preparation

Algunos drivers pueden ofrecer:

```text
emulated prepared statements
```

Esto deberá ser una capability/configuration explícita.

---

# 34. Emulated ≠ native

No deberán tratarse como equivalentes para:

```text
security assumptions
type behavior
placeholder handling
resource lifetime
statement reuse
server resource consumption
```

---

# 35. Direct execution

Algunos commands pueden no requerir preparation.

En ese caso deberá existir una estrategia explícita.

No deberá fingirse un prepared statement nativo inexistente.

---

# 36. Preparation strategy

```php
interface StatementPreparationStrategy
{
    public function prepare(
        CompiledDatabaseCommand $command,
        ConnectionLease $connection,
        PreparedStatementContext $context,
    ): PreparedStatement;
}
```

---

# 37. Strategy resolution

```text
Command
+
DriverCapabilities
+
PlatformCapabilities
+
Policy
        ↓
PreparationStrategyResolver
```

---

# 38. No SQL inspection

La estrategia no deberá determinarse mediante:

```php
if (str_contains($sql, 'RETURNING')) {
    // ...
}
```

Debe utilizar metadata estructurada.

---

# 39. Preparation pipeline

```text
PreparedStatementRequest
        ↓
Request Validation
        ↓
Compatibility Validation
        ↓
Preparation Strategy Resolution
        ↓
Prepared Statement Identity
        ↓
Cache Lookup
        ↓
┌─────────────────────────────┐
│ reusable statement exists?  │
└─────────────────────────────┘
       ↓ yes            ↓ no
Validate Health       Driver Prepare
       ↓                 ↓
Reset if Needed      Wrap Resource
       ↓                 ↓
Acquire Lease        Register Resource
       └───────┬─────────┘
               ↓
      PreparedStatementLease
```

---

# 40. Phases

Propuesta:

```text
P0 Request Acceptance
P1 Compatibility Validation
P2 Strategy Resolution
P3 Identity Construction
P4 Cache Lookup
P5 Candidate Validation
P6 Candidate Reset
P7 Driver Preparation
P8 Resource Registration
P9 Lease Acquisition
P10 Outcome Finalization
```

---

# 41. P0 — Request Acceptance

Verificar:

```text
command
connection lease
context
policy
```

---

# 42. P1 — Compatibility

Validar:

```text
command platform
connection platform
driver compilation contract
placeholder style
statement options
session requirements
preparation capability
```

---

# 43. P2 — Strategy Resolution

Posibles estrategias:

```text
NativePreparedStatementStrategy
EmulatedPreparedStatementStrategy
DirectStatementStrategy
ExtensionPreparationStrategy
```

---

# 44. P3 — Identity

Construir:

```text
PreparedStatementCacheKey
```

sin runtime binding values.

---

# 45. Runtime values excluded

Nunca:

```text
cache key = SQL + parameter values
```

Los valores cambian entre ejecuciones.

---

# 46. P4 — Cache Lookup

```text
PreparedStatementCache
        ↓
candidate?
```

---

# 47. No global cache by SQL

Prohibido:

```php
static $cache[$sql];
```

---

# 48. Cache scope

Normalmente:

```text
per physical connection
```

---

# 49. Why per connection

Porque el prepared statement está asociado al recurso físico de conexión.

---

# 50. Possible hierarchy

```text
Connection
└── PreparedStatementPool
    ├── P1
    ├── P2
    └── P3
```

---

# 51. Cache vs pool

Conviene distinguir:

```text
PreparedStatementCache
=
lookup/reuse policy
```

```text
PreparedStatementPool
=
collection of reusable prepared resources
```

---

# 52. Statement cache key

Puede incluir:

```text
CompiledCommandFingerprint
DriverPreparationContractId
PlaceholderLayoutFingerprint
ResultContractFingerprint
PreparationOptionsFingerprint
SessionCompatibilityKey
ExtensionSetFingerprint
```

---

# 53. Connection not necessarily inside key

Si el cache ya está físicamente encapsulado por conexión:

```text
ConnectionPreparedStatementCache
```

no necesita repetir `ConnectionIdentity` dentro de cada key.

Pero la identidad completa del resource sí la conserva.

---

# 54. Cache lookup result

```text
MISS
HIT_REUSABLE
HIT_BUSY
HIT_INVALID
HIT_RESET_REQUIRED
```

---

# 55. HIT_BUSY

Un statement compatible puede estar:

```text
leased
executing
streaming
```

---

# 56. Busy statement behavior

Opciones:

```text
prepare another instance
wait
fail
```

según policy/capability.

---

# 57. Default recommendation

Para evitar contention:

```text
busy cached statement
→ prepare another statement
```

cuando los límites de recursos lo permitan.

---

# 58. Prepared statement multiplicity

Por tanto:

```text
CacheKey
→ zero or more PreparedStatement instances
```

---

# 59. Cache entry ≠ single object

Puede ser:

```text
PreparedStatementBucket
```

---

# 60. Example

```text
Command C1
Connection A

CacheKey K1
   ├── P1 BUSY
   ├── P2 IDLE
   └── P3 INVALID
```

Seleccionar:

```text
P2
```

---

# 61. P5 — Candidate Validation

Antes del reuse:

```text
state
connection identity
session compatibility
driver health
cursor state
pending results
binding state
statement options
expiration/invalidation
```

---

# 62. Statement states

Propuesta:

```text
NEW
PREPARING
IDLE
LEASED
BINDING
READY
EXECUTING
RESULT_ACTIVE
RESETTING
INVALID
CLOSING
CLOSED
BROKEN
```

---

# 63. State machine

```text
NEW
 ↓
PREPARING
 ↓
IDLE
 ↓
LEASED
 ↓
BINDING
 ↓
READY
 ↓
EXECUTING
 ↓
RESULT_ACTIVE
 ↓
RESETTING
 ↓
IDLE
```

Terminal:

```text
INVALID
BROKEN
CLOSED
```

---

# 64. Direct IDLE transition

Para statements sin result:

```text
EXECUTING
   ↓
RESETTING
   ↓
IDLE
```

---

# 65. Active cursor

Mientras exista cursor:

```text
RESULT_ACTIVE
```

---

# 66. No reuse during RESULT_ACTIVE

Salvo que el driver declare explícitamente multiplexing compatible.

Default:

```text
RESULT_ACTIVE
→ not reusable
```

---

# 67. P6 — Candidate Reset

Antes del reuse puede requerirse:

```text
close cursor
drain result
clear bindings
reset driver statement
restore options
clear generated-key state
clear output parameters
```

---

# 68. Reset contract

```php
interface PreparedStatementResetter
{
    public function reset(
        PreparedStatement $statement,
        PreparedStatementResetContext $context,
    ): PreparedStatementResetOutcome;
}
```

---

# 69. Reset strategy

Depende del driver.

---

# 70. Reset outcome

```text
RESET_SUCCESS
RESET_NOT_REQUIRED
RESET_UNSUPPORTED
RESET_FAILED
STATEMENT_INVALIDATED
CONNECTION_INVALIDATED
```

---

# 71. Reset failure

Nunca regresar al cache como reusable un statement cuyo reset falló.

---

# 72. Reset safety

```text
ResetFailure
→ invalidate statement
```

y posiblemente:

```text
→ invalidate connection
```

según driver health classification.

---

# 73. Result draining

Algunos drivers requieren consumir resultados pendientes.

---

# 74. Drain strategy

```text
CLOSE_CURSOR
DRAIN_CURRENT_RESULT
DRAIN_ALL_RESULTS
DRIVER_RESET
DISCARD_STATEMENT
DISCARD_CONNECTION
```

---

# 75. Drain ≠ fetch for application

El draining es cleanup técnico.

No debe producir resultados de aplicación.

---

# 76. Result draining budget

Debe ser bounded.

Nunca consumir indefinidamente resultados arbitrarios sólo para salvar una conexión.

---

# 77. Budget exceeded during drain

Podrá provocar:

```text
statement invalidation
connection discard
```

---

# 78. Clear bindings

Un statement reusable no deberá conservar referencias a valores de una ejecución anterior.

Especialmente:

```text
passwords
tokens
binary payloads
LOB streams
PII
```

---

# 79. Memory retention security

Después del reset:

```text
old runtime binding references
=
released
```

cuando sea posible.

---

# 80. P7 — Driver Preparation

Si no existe candidato reusable:

```text
DriverStatementPreparer
```

crea uno.

---

# 81. Driver contract

```php
interface DriverStatementPreparer
{
    public function prepare(
        DriverConnection $connection,
        DriverPreparationRequest $request,
    ): DriverPreparedStatement;
}
```

---

# 82. DriverPreparationRequest

```php
final readonly class DriverPreparationRequest
{
    public function __construct(
        public RenderedSql $sql,
        public DriverPlaceholderContract $placeholders,
        public DriverStatementOptions $options,
        public CompiledResultContract $resultContract,
    ) {}
}
```

---

# 83. No runtime values at preparation

Normalmente:

```text
Prepare
=
SQL structure
```

mientras:

```text
Bind
=
runtime values
```

---

# 84. Exception

Si un driver concreto necesita información adicional para prepare, deberá expresarse en su contract.

No mediante hacks genéricos.

---

# 85. Preparation error

Ejemplos:

```text
invalid statement
unsupported placeholder mode
server prepare limit
schema object missing
permission failure
connection failure
```

---

# 86. Error normalization

La causa deberá clasificarse sin destruir la excepción nativa.

---

# 87. Preparation failure certainty

Si falla `prepare()` antes de `execute()`:

```text
statement side effects
=
none
```

---

# 88. Prepare may interact with server

Aunque no ejecute la query, native prepare puede:

```text
send SQL
parse
validate
allocate server resources
```

---

# 89. Preparation ≠ pure computation

A diferencia del SQL Compiler:

```text
Prepared Statement System
```

sí puede realizar I/O mediante el driver.

---

# 90. Compiler distinction

```text
SQL Compiler
=
pure-ish representation transformation
```

```text
Prepared Statement System
=
runtime resource acquisition
```

---

# 91. P8 — Resource Registration

Todo prepared statement deberá registrarse con ownership.

---

# 92. Resource owner

Inicialmente:

```text
PreparedStatementManager
```

o:

```text
ConnectionPreparedStatementPool
```

según lifecycle.

---

# 93. P9 — Lease Acquisition

El caller no recibe directamente el resource reusable.

Recibe:

```text
PreparedStatementLease
```

---

# 94. PreparedStatementLease

```php
interface PreparedStatementLease
{
    public function statement(): PreparedStatementHandle;

    public function release(): void;

    public function invalidate(
        PreparedStatementInvalidationReason $reason,
    ): void;
}
```

---

# 95. Why lease

Permite separar:

```text
resource lifetime
```

de:

```text
temporary execution ownership
```

---

# 96. Lease guarantees

Mientras un lease exclusivo esté activo:

```text
another execution
```

no podrá usar ese statement salvo capability explícita.

---

# 97. Lease state

```text
ACTIVE
TRANSFERRED
RELEASED
INVALIDATED
```

---

# 98. Double release

Debe ser:

```text
idempotent
```

o producir diagnóstico controlado.

Nunca corrupción.

---

# 99. Lease ownership transfer

Streaming puede transferir:

```text
PreparedStatementLease
```

del StatementExecutor al ResultCursor.

---

# 100. Example

```text
StatementExecutor
      │
      └── PreparedStatementLease
                  │
                  └── transfer
                        ↓
                   ResultCursor
```

---

# 101. Cursor closes

Entonces:

```text
ResultCursor
     ↓
release PreparedStatementLease
```

---

# 102. Lease release

No significa necesariamente:

```text
close prepared statement
```

Puede significar:

```text
reset
→ return to cache
```

---

# 103. Release pipeline

```text
Lease Release
    ↓
State Inspection
    ↓
Result Cleanup
    ↓
Statement Reset
    ↓
Health Validation
    ↓
Reusable?
  ↙       ↘
yes       no
 ↓         ↓
cache     close
```

---

# 104. Reusability decision

Formalmente:

```text
Reusable(P)
=
Healthy(P)
∧
CompatibleSession(P)
∧
NoActiveResult(P)
∧
Resettable(P)
∧
CacheAllowed(P)
∧
ConnectionAlive(P)
```

---

# 105. Cache policy

```php
final readonly class PreparedStatementPolicy
{
    public function __construct(
        public PreparedStatementReuseMode $reuse,
        public PreparedStatementCachePolicy $cache,
        public PreparedStatementResetPolicy $reset,
        public PreparedStatementConcurrencyPolicy $concurrency,
    ) {}
}
```

---

# 106. Reuse modes

```text
NEVER
WHEN_SAFE
PREFER_REUSE
REQUIRE_REUSE
```

`REQUIRE_REUSE` deberá usarse con extrema cautela.

---

# 107. Default

```text
WHEN_SAFE
```

---

# 108. Cache capacity

El cache deberá ser bounded.

---

# 109. Why bounded

Prepared statements pueden consumir:

```text
client memory
server memory
server statement slots
metadata
handles
```

---

# 110. Cache dimensions

Podrán limitarse por:

```text
max statements per connection
max total estimated memory
max statements per command
max idle age
max server-side handles
```

---

# 111. Eviction

Políticas posibles:

```text
LRU
CLOCK
cost-aware
frequency-aware
driver-aware
```

La política no altera semantics.

---

# 112. Eviction requirement

Eviction debe cerrar correctamente el recurso.

---

# 113. Eviction of busy statement

Prohibido.

---

# 114. Busy entries

No deberán ser desalojadas físicamente hasta liberar el lease.

Podrán marcarse:

```text
EVICT_ON_RELEASE
```

---

# 115. Cache entry state

```text
ACTIVE
IDLE
EVICT_PENDING
INVALID
CLOSED
```

---

# 116. Cache invalidation

Puede producirse por:

```text
connection close
connection reconnect
session incompatibility
schema change
driver reset
server invalidation
statement error
extension change
explicit flush
resource pressure
```

---

# 117. Schema changes

Algunos DBMS invalidan prepared statements automáticamente.

Otros pueden conservarlos con diferentes comportamientos.

VoltStack no deberá asumir una regla universal.

---

# 118. Schema epoch

Opcionalmente podrá existir:

```text
SchemaCompatibilityEpoch
```

cuando la plataforma/integración pueda proporcionarlo correctamente.

---

# 119. No hidden schema polling

Prepared Statement System no deberá consultar schema metadata antes de cada ejecución.

---

# 120. Server invalidation

Si el driver informa:

```text
statement invalid
```

deberá normalizarse como:

```text
PreparedStatementInvalidated
```

---

# 121. Automatic reprepare

Puede ser posible en ciertos errores.

Pero deberá estar estrictamente controlado.

---

# 122. Reprepare ≠ retry query

Distinción:

```text
prepare failed because cached statement invalid
→ reprepare before dispatch
```

puede ser seguro.

Mientras:

```text
execute failed after dispatch
→ reprepare and execute again
```

es un retry y pertenece al documento 86.

---

# 123. Safe reprepare boundary

Si se confirma:

```text
statement was not dispatched
```

puede adquirirse otro prepared statement.

---

# 124. Never hide execution retry

Prepared Statement System no deberá reejecutar una operación.

---

# 125. Connection invalidation

Cuando una conexión muere:

```text
all prepared statements bound to that physical connection
→ INVALID
```

---

# 126. Bulk invalidation

Debe existir una operación eficiente:

```php
interface PreparedStatementInvalidator
{
    public function invalidateConnection(
        ConnectionIdentity $connection,
        PreparedStatementInvalidationReason $reason,
    ): void;
}
```

---

# 127. Invalidation reasons

```text
CONNECTION_CLOSED
CONNECTION_BROKEN
SESSION_CHANGED
SCHEMA_CHANGED
DRIVER_INVALIDATED
SERVER_INVALIDATED
RESET_FAILED
RESULT_DRAIN_FAILED
SECURITY_CONTEXT_CHANGED
CACHE_EVICTION
MANUAL
EXTENSION
```

---

# 128. Invalidation ≠ immediate destruction always

Si el statement está leased:

```text
mark invalid
→ close on release
```

---

# 129. No resource use after invalidation

Una vez invalidado:

```text
new lease
```

no podrá adquirirse.

---

# 130. Statement close

```php
interface PreparedStatementCloser
{
    public function close(
        PreparedStatement $statement,
    ): void;
}
```

---

# 131. Close semantics

`close()` deberá:

```text
release client handle
release server resource when applicable
remove from cache
clear binding references
clear result references
transition CLOSED
```

---

# 132. Close idempotence

Preferiblemente:

```text
close(CLOSED)
→ no-op
```

con diagnóstico opcional.

---

# 133. Close failure

No deberá reinsertar el statement en cache.

---

# 134. Driver close failure

Puede afectar:

```text
connection health
```

y deberá reportarse.

---

# 135. Connection close order

Idealmente:

```text
invalidate statements
↓
close statements when necessary
↓
close physical connection
```

aunque drivers que invaliden automáticamente handles pueden especializar el proceso.

---

# 136. Ownership graph

```text
Connection
   │
   └── PreparedStatementPool
          │
          ├── Statement P1
          ├── Statement P2
          └── Statement P3
```

Los leases son ownership temporal:

```text
Execution
   ↓
PreparedStatementLease
   ↓
Statement P2
```

---

# 137. Resource hierarchy

```text
Physical Connection
      ↓
Prepared Statement
      ↓
Active Result/Cursor
```

Generalmente:

```text
child lifetime ≤ parent lifetime
```

---

# 138. Parent invalidation

Si connection se invalida:

```text
statement invalid
cursor invalid
```

---

# 139. Child release

Cerrar cursor no necesariamente cierra statement.

---

# 140. Statement release

Liberar statement lease no necesariamente cierra connection.

---

# 141. Connection lease interaction

Un prepared statement cached puede existir mientras la physical connection permanezca viva, incluso si no existe un application-level connection lease activo.

Esto depende del Connection Manager.

---

# 142. Important distinction

```text
Connection resource lifetime
≠
Connection lease lifetime
```

---

# 143. Pool integration

En un connection pool:

```text
Physical Connection
        ↓
prepared statement cache
```

puede sobrevivir a múltiples requests.

---

# 144. Persistent runtime safety

Esto es válido sólo si:

```text
session state reset
tenant state reset
bindings cleared
active results closed
statement state reset
security context compatible
```

---

# 145. FrankenPHP

Con workers persistentes:

```text
Request A
  ↓
Connection C1
  ↓
Prepared P1
  ↓
release/reset

Request B
  ↓
Connection C1
  ↓
reuse P1
```

sólo si todas las invariantes se cumplen.

---

# 146. Dangerous persistent state

Nunca:

```text
P1 old password binding
P1 old cursor
P1 old tenant schema
P1 old timeout
P1 old result
```

deberán sobrevivir accidentalmente.

---

# 147. RoadRunner/OpenSwoole

Las mismas reglas aplicarán.

OpenSwoole añade especial importancia a:

```text
concurrent coroutine isolation
```

---

# 148. Concurrency model

Prepared statements serán:

```text
EXCLUSIVE
SHAREABLE
MULTIPLEXABLE
```

según capability.

---

# 149. Default

```text
EXCLUSIVE
```

---

# 150. Exclusive statement

Un lease activo impide otro lease simultáneo.

---

# 151. Shareable descriptor ≠ shareable resource

Los descriptors pueden compartirse.

El driver resource no necesariamente.

---

# 152. Locking strategy

El cache/pool deberá usar una primitiva adecuada al runtime.

No deberá hard-codearse:

```text
pthread mutex
```

en el core.

---

# 153. Runtime synchronization abstraction

```php
interface PreparedStatementSynchronization
{
    public function acquire(
        PreparedStatementIdentity $statement,
    ): PreparedStatementGuard;
}
```

---

# 154. Async/coroutine runtime

No debe bloquear innecesariamente el worker completo.

---

# 155. Wait policy

Si un statement está ocupado:

```text
WAIT
PREPARE_NEW
FAIL_FAST
```

---

# 156. Default recommendation

```text
PREPARE_NEW
```

hasta alcanzar el budget.

Después:

```text
WAIT or FAIL_FAST
```

según runtime/policy.

---

# 157. Resource budget

```text
PreparedStatementBudget
```

deberá ser explícito.

---

# 158. Budget dimensions

```text
max prepared statements per connection
max duplicate statements per cache key
max total statement handles
max estimated memory
max preparation rate
max reset work
max drain work
```

---

# 159. Server limit awareness

Si el DBMS impone límites:

```text
max prepared statements
```

podrán reflejarse en capabilities/config.

---

# 160. Unknown limit

```text
unknown
≠
unlimited
```

---

# 161. Resource pressure

Bajo presión:

```text
evict idle statements
```

antes de afectar statements leased.

---

# 162. Cache admission

No todo statement preparado necesita entrar al cache.

---

# 163. Admission policy

Puede considerar:

```text
reuse probability
statement complexity
preparation cost
resource cost
execution frequency
session stability
command classification
```

---

# 164. One-shot statement

Un command marcado:

```text
NO_REUSE
```

deberá cerrarse después de ejecución.

---

# 165. Security-sensitive statement

Puede configurarse:

```text
DO_NOT_CACHE
```

si existe razón de seguridad/driver.

---

# 166. Cache decision ≠ query semantics

No altera el resultado de la query.

---

# 167. Prepared statement fingerprint

Se recomienda distinguir:

```text
CompiledCommandFingerprint
```

de:

```text
PreparedStatementFingerprint
```

---

# 168. Formula

```text
PreparedStatementFingerprint
=
Hash(
    CompiledCommandFingerprint
    + DriverPreparationContract
    + PreparationOptions
    + SessionCompatibilityKey
    + PreparedStatementSystemVersion
)
```

---

# 169. Physical identity

Además:

```text
PreparedStatementInstanceId
```

identifica una instancia runtime concreta.

---

# 170. Fingerprint ≠ instance ID

```text
fingerprint
=
compatibility/reuse identity
```

```text
instance ID
=
runtime resource identity
```

---

# 171. Multiple instances same fingerprint

Perfectamente válido:

```text
Fingerprint F1
   ├── P100
   ├── P101
   └── P102
```

---

# 172. Statement options

Deben ser estructuradas.

Ejemplos:

```text
cursor mode
scrollability
buffering
generated keys
result mode
driver preparation mode
server prepare mode
```

---

# 173. Options compatibility

Dos statements con diferentes options pueden no ser reusable-equivalent.

---

# 174. No arbitrary options bag

Preferir tipos:

```text
CursorMode
BufferingMode
GeneratedKeyMode
PreparationMode
```

sobre:

```php
array $options;
```

---

# 175. Parameter metadata

Prepared statement puede almacenar:

```text
parameter count
driver parameter metadata
placeholder mapping
```

---

# 176. But binding ownership elsewhere

El valor runtime sigue perteneciendo al:

```text
Parameter Binding System
```

---

# 177. Parameter metadata cache

Puede formar parte del statement descriptor/runtime metadata si el driver lo proporciona.

---

# 178. Result metadata

Algunos drivers conocen metadata al preparar.

Otros sólo después de ejecutar.

---

# 179. No universal assumption

Por tanto:

```text
PreparedResultMetadata
```

debe ser opcional/parcial.

---

# 180. Statement metadata confidence

Puede modelarse:

```text
KNOWN_AT_COMPILE
KNOWN_AT_PREPARE
KNOWN_AT_EXECUTION
UNKNOWN
```

---

# 181. Security model

Prepared statements son parte central de la defensa contra SQL injection, pero:

```text
prepared statement
≠
complete SQL injection prevention
```

---

# 182. Why

Los parameters protegen valores.

No protegen automáticamente:

```text
identifiers
operators
ORDER BY direction
SQL keywords
raw fragments
function names
collations
```

Eso ya debe estar estructurado desde compilation.

---

# 183. Security invariant

Runtime values deberán llegar como:

```text
bindings
```

no como SQL concatenado.

---

# 184. Emulated prepares

Si el driver emula preparation, VoltStack deberá conocer sus garantías reales.

---

# 185. Security policy

Podrá existir:

```text
allow_emulated_prepares
```

pero como configuración explícita.

---

# 186. Raw SQL

Prepared Statement System no deberá “sanitizar” raw SQL.

Raw SQL ya debe haber pasado las fronteras de seguridad anteriores.

---

# 187. Sensitive bindings cleanup

Al liberar statement:

```text
references to sensitive values
```

deberán eliminarse tan pronto como sea posible.

---

# 188. Diagnostics

Nunca incluir:

```text
password
token
secret
full PII
binary payload
```

por defecto.

---

# 189. Prepared statement names

PostgreSQL u otros drivers pueden utilizar nombres server-side.

Dichos nombres deberán generarse internamente.

---

# 190. Name generation

Debe evitar:

```text
tenant name
user email
query contents
secret values
```

---

# 191. Deterministic vs unique names

Puede requerirse un:

```text
stable prefix
+
connection-local unique sequence
```

---

# 192. Runtime sequence

El sequence pertenece a la conexión/session runtime.

No al compiled command.

---

# 193. MySQL considerations

El sistema deberá poder representar diferencias como:

```text
native server prepares
emulated prepares
statement handle limits
result buffering behavior
cursor capabilities
metadata behavior
```

sin introducir lógica MySQL en el core.

---

# 194. MariaDB considerations

MariaDB deberá tener capabilities propias.

No heredar ciegamente las de MySQL.

---

# 195. PostgreSQL considerations

Debe poder soportarse:

```text
numbered placeholders
server-side prepared statements
statement names
plan lifecycle
search_path sensitivity
result metadata
```

según driver utilizado.

---

# 196. SQLite considerations

SQLite prepared statements son especialmente connection-bound.

Además pueden depender de:

```text
schema changes
attached databases
registered functions
registered collations
```

---

# 197. SQLite invalidation

Cambios de schema pueden requerir:

```text
reprepare
```

según comportamiento del driver/engine.

Debe manejarse mediante capability/error classification.

---

# 198. Extension-provided functions

Si una conexión registra una función custom:

```text
my_function(...)
```

un statement dependiente de ella deberá declarar dependency/session compatibility.

---

# 199. Extension-provided collations

Misma regla.

---

# 200. Dependency tracking

`PreparedStatementDescriptor` puede incluir:

```text
StatementDependencySet
```

---

# 201. Dependencies

Ejemplos:

```text
relation schema
function capability
collation capability
extension
session setting
temporary object
driver feature
```

---

# 202. Dependency invalidation

Cuando una dependency cambia:

```text
affected prepared statements
→ invalidate
```

si el sistema posee conocimiento fiable del cambio.

---

# 203. No false precision

Si VoltStack no puede saber con seguridad qué statements son afectados:

```text
invalidate broader scope
```

es preferible a reutilización insegura.

---

# 204. Statement health

Propuesta:

```text
HEALTHY
RESET_REQUIRED
SUSPECT
INVALID
BROKEN
CLOSED
```

---

# 205. Health ≠ state

State indica lifecycle.

Health indica reutilizabilidad/confiabilidad.

---

# 206. Example

```text
state  = IDLE
health = SUSPECT
```

No deberá reutilizarse automáticamente.

---

# 207. Health evaluator

```php
interface PreparedStatementHealthEvaluator
{
    public function evaluate(
        PreparedStatement $statement,
    ): PreparedStatementHealth;
}
```

---

# 208. Statement failure classification

Un error puede afectar:

```text
only current execution
statement
connection
transaction
session
```

---

# 209. Failure impact

Debe producirse:

```text
FailureImpact
```

estructurado.

---

# 210. Example

```text
syntax error
```

puede indicar command inválido.

Mientras:

```text
connection lost
```

invalida statement y connection.

---

# 211. Transaction abort

PostgreSQL, por ejemplo, puede dejar una transacción en estado abortado tras ciertos errores.

Eso pertenece al Transaction System, pero el prepared statement subsystem debe conservar la señal.

---

# 212. No transaction repair

Prepared Statement System no ejecutará:

```text
ROLLBACK
```

por sí mismo.

---

# 213. Telemetry

Eventos:

```text
PreparedStatementLookupStarted
PreparedStatementCacheHit
PreparedStatementCacheMiss
PreparedStatementCandidateRejected
PreparedStatementPreparationStarted
PreparedStatementPrepared
PreparedStatementLeaseAcquired
PreparedStatementLeaseReleased
PreparedStatementResetStarted
PreparedStatementResetCompleted
PreparedStatementInvalidated
PreparedStatementEvicted
PreparedStatementClosed
PreparedStatementPreparationFailed
```

---

# 214. Telemetry metadata

Seguro registrar:

```text
statement fingerprint
command fingerprint
driver
platform
cache hit/miss
prepare duration
reuse count
reset duration
statement age
health
state
```

---

# 215. SQL text telemetry

Debe seguir la política general de Database Telemetry.

---

# 216. Binding values

No pertenecen a la telemetría del prepared statement.

---

# 217. Metrics

Ejemplos:

```text
db.prepared.cache.hit
db.prepared.cache.miss
db.prepared.created
db.prepared.reused
db.prepared.closed
db.prepared.invalidated
db.prepared.reset.failed
db.prepared.active
db.prepared.idle
db.prepared.busy
```

---

# 218. Hit ratio

```text
PreparedStatementCacheHitRatio
=
CacheHits
/
(CacheHits + CacheMisses)
```

---

# 219. Reuse ratio

```text
ReuseRatio
=
ReusedExecutions
/
TotalPreparedStatementAcquisitions
```

---

# 220. But metrics do not define policy

Una alta hit ratio no justifica conservar statements indefinidamente.

---

# 221. Diagnostics

Debug output puede mostrar:

```text
Fingerprint
Connection
State
Health
Lease status
Reuse count
Cache status
Preparation strategy
Driver
Platform
```

sin valores de parámetros.

---

# 222. Extension architecture

Prepared Statement System podrá extenderse mediante:

```text
PreparationStrategy
PreparedStatementPolicyProvider
PreparedStatementResetStrategy
PreparedStatementHealthEvaluator
PreparedStatementCachePolicy
DriverStatementAdapter
```

---

# 223. Typed extensions only

No:

```php
onPrepare(function ($sql, $connection) {
    return whatever();
});
```

como mecanismo universal.

---

# 224. Registry

```text
PreparedStatementExtensionRegistry
```

deberá congelarse después de bootstrap.

---

# 225. No last-wins

Dos estrategias con ownership ambiguo:

```text
bootstrap failure
```

---

# 226. Platform scoping

Contributions podrán ser:

```text
generic
MySQL-specific
MariaDB-specific
PostgreSQL-specific
SQLite-specific
driver-specific
capability-specific
```

---

# 227. No vendor conditionals everywhere

Evitar:

```php
if ($platform === 'mysql') { ... }
elseif ($platform === 'pgsql') { ... }
```

en el manager central.

---

# 228. Capability resolution

Preferir:

```text
PreparedStatementCapabilityResolver
```

---

# 229. Error hierarchy

Propuesta:

```text
PreparedStatementException
├── PreparedStatementValidationException
├── PreparedStatementCompatibilityException
├── PreparedStatementUnsupportedException
├── PreparedStatementPreparationException
├── PreparedStatementLeaseException
├── PreparedStatementBusyException
├── PreparedStatementResetException
├── PreparedStatementDrainException
├── PreparedStatementInvalidationException
├── PreparedStatementCloseException
├── PreparedStatementCacheException
├── PreparedStatementBudgetException
├── PreparedStatementSecurityException
├── PreparedStatementConcurrencyException
└── PreparedStatementInvariantException
```

---

# 230. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Prepared/
                ├── Contract/
                │   ├── PreparedStatementManager.php
                │   ├── StatementPreparationStrategy.php
                │   ├── DriverStatementPreparer.php
                │   ├── PreparedStatementResetter.php
                │   ├── PreparedStatementCloser.php
                │   ├── PreparedStatementInvalidator.php
                │   └── PreparedStatementHealthEvaluator.php
                │
                ├── Manager/
                │   └── DefaultPreparedStatementManager.php
                │
                ├── Statement/
                │   ├── PreparedStatement.php
                │   ├── PreparedStatementHandle.php
                │   ├── PreparedStatementIdentity.php
                │   ├── PreparedStatementInstanceId.php
                │   ├── PreparedStatementDescriptor.php
                │   ├── PreparedStatementState.php
                │   └── PreparedStatementHealth.php
                │
                ├── Request/
                │   ├── PreparedStatementRequest.php
                │   └── DriverPreparationRequest.php
                │
                ├── Context/
                │   ├── PreparedStatementContext.php
                │   └── PreparedStatementPolicy.php
                │
                ├── Strategy/
                │   ├── NativePreparedStatementStrategy.php
                │   ├── EmulatedPreparedStatementStrategy.php
                │   ├── DirectStatementStrategy.php
                │   └── PreparationStrategyResolver.php
                │
                ├── Lease/
                │   ├── PreparedStatementLease.php
                │   ├── DefaultPreparedStatementLease.php
                │   ├── PreparedStatementLeaseState.php
                │   └── PreparedStatementLeaseManager.php
                │
                ├── Cache/
                │   ├── PreparedStatementCache.php
                │   ├── ConnectionPreparedStatementCache.php
                │   ├── PreparedStatementCacheKey.php
                │   ├── PreparedStatementBucket.php
                │   ├── PreparedStatementCacheEntry.php
                │   ├── PreparedStatementCachePolicy.php
                │   ├── PreparedStatementAdmissionPolicy.php
                │   └── PreparedStatementEvictionPolicy.php
                │
                ├── Reset/
                │   ├── PreparedStatementResetContext.php
                │   ├── PreparedStatementResetOutcome.php
                │   ├── DefaultPreparedStatementResetter.php
                │   └── StatementDrainStrategy.php
                │
                ├── Invalidation/
                │   ├── PreparedStatementInvalidationReason.php
                │   ├── PreparedStatementInvalidationRegistry.php
                │   └── DefaultPreparedStatementInvalidator.php
                │
                ├── Dependency/
                │   ├── StatementDependencySet.php
                │   ├── StatementDependency.php
                │   └── SessionCompatibilityKey.php
                │
                ├── Resource/
                │   ├── PreparedStatementResourceOwner.php
                │   └── PreparedStatementBudget.php
                │
                ├── Concurrency/
                │   ├── PreparedStatementConcurrencyMode.php
                │   ├── PreparedStatementSynchronization.php
                │   └── PreparedStatementGuard.php
                │
                ├── Health/
                │   ├── DefaultPreparedStatementHealthEvaluator.php
                │   └── PreparedStatementFailureImpact.php
                │
                ├── Telemetry/
                │   ├── PreparedStatementObserver.php
                │   ├── PreparedStatementTelemetry.php
                │   └── PreparedStatementMetrics.php
                │
                ├── Extension/
                │   └── PreparedStatementExtensionRegistry.php
                │
                └── Exception/
                    ├── PreparedStatementException.php
                    ├── PreparedStatementValidationException.php
                    ├── PreparedStatementCompatibilityException.php
                    ├── PreparedStatementUnsupportedException.php
                    ├── PreparedStatementPreparationException.php
                    ├── PreparedStatementLeaseException.php
                    ├── PreparedStatementBusyException.php
                    ├── PreparedStatementResetException.php
                    ├── PreparedStatementDrainException.php
                    ├── PreparedStatementInvalidationException.php
                    ├── PreparedStatementCloseException.php
                    ├── PreparedStatementCacheException.php
                    ├── PreparedStatementBudgetException.php
                    ├── PreparedStatementSecurityException.php
                    ├── PreparedStatementConcurrencyException.php
                    └── PreparedStatementInvariantException.php
```

---

# 231. Dependency direction

```text
StatementExecutor
       ↓
Prepared Statement System
       ↓
Driver Statement Preparation
       ↓
Connection
       ↓
Driver
```

Después:

```text
PreparedStatementLease
       ↓
Parameter Binding System
```

---

# 232. Forbidden dependencies

Prepared Statement System no dependerá de:

```text
ORM
EntityManager
UnitOfWork
IdentityMap
Repository
Query Optimizer
Physical Planner
Hydrator
```

---

# 233. Architectural invariants

## DB-PREP-001

PreparedStatement será distinto de CompiledDatabaseCommand.

## DB-PREP-002

PreparedStatement será distinto de StatementExecutionInstance.

## DB-PREP-003

PreparedStatement será runtime resource.

## DB-PREP-004

CompiledDatabaseCommand podrá reutilizarse entre ejecuciones.

## DB-PREP-005

PreparedStatement estará asociado a una conexión concreta cuando el driver lo requiera.

## DB-PREP-006

Un statement de Connection A no se reutilizará en Connection B.

## DB-PREP-007

Connection configuration name no será physical connection identity.

## DB-PREP-008

Reconnect invalidará statements de la conexión anterior.

## DB-PREP-009

Session compatibility será explícita.

## DB-PREP-010

Tenant-sensitive statements no cruzarán tenant boundaries incompatibles.

## DB-PREP-011

Native prepare será distinto de emulated prepare.

## DB-PREP-012

Emulated prepare será capability/config explícita.

## DB-PREP-013

Direct execution será strategy explícita.

## DB-PREP-014

Preparation strategy será capability-driven.

## DB-PREP-015

Strategy resolution no inspeccionará SQL mediante string matching.

## DB-PREP-016

Prepared statement identity no será solamente hash(SQL).

## DB-PREP-017

Runtime binding values no participarán en cache identity.

## DB-PREP-018

Prepared statement cache será normalmente connection-scoped.

## DB-PREP-019

No existirá cache global por SQL text.

## DB-PREP-020

Un cache key podrá corresponder a múltiples instances.

## DB-PREP-021

Busy statement no será reutilizado concurrentemente por defecto.

## DB-PREP-022

Reusable candidate será validado antes del lease.

## DB-PREP-023

RESULT_ACTIVE no será reusable por defecto.

## DB-PREP-024

Reset será explícito.

## DB-PREP-025

Reset failure invalidará el statement.

## DB-PREP-026

Reset failure podrá invalidar connection cuando corresponda.

## DB-PREP-027

Pending results deberán cerrarse o drenarse según driver contract.

## DB-PREP-028

Result drain será bounded.

## DB-PREP-029

Old bindings deberán liberarse antes del reuse.

## DB-PREP-030

Sensitive binding references no sobrevivirán innecesariamente.

## DB-PREP-031

Driver preparation ocurrirá mediante driver contract.

## DB-PREP-032

Core no dependerá directamente de PDOStatement.

## DB-PREP-033

Preparation puede realizar I/O.

## DB-PREP-034

Preparation será distinta de SQL compilation.

## DB-PREP-035

Prepared resources tendrán ownership explícito.

## DB-PREP-036

Caller recibirá lease, no ownership físico irrestricto.

## DB-PREP-037

Lease exclusivo impedirá concurrent reuse.

## DB-PREP-038

Lease release no implicará necesariamente close.

## DB-PREP-039

Streaming podrá transferir statement lease.

## DB-PREP-040

Resource transfer será explícito.

## DB-PREP-041

Released lease no podrá volver a usarse.

## DB-PREP-042

Reusable statement deberá estar healthy.

## DB-PREP-043

Reusable statement deberá tener session compatible.

## DB-PREP-044

Reusable statement no tendrá active result incompatible.

## DB-PREP-045

Reusable statement deberá poder resetearse.

## DB-PREP-046

Cache será bounded.

## DB-PREP-047

Busy statements no serán evicted físicamente.

## DB-PREP-048

Busy statement podrá marcarse EVICT_ON_RELEASE.

## DB-PREP-049

Eviction cerrará correctamente el statement.

## DB-PREP-050

Connection close invalidará sus statements.

## DB-PREP-051

Schema invalidation será capability-aware.

## DB-PREP-052

Prepared Statement System no realizará schema polling por ejecución.

## DB-PREP-053

Server invalidation será normalizada.

## DB-PREP-054

Reprepare será distinto de retry.

## DB-PREP-055

Prepared Statement System nunca reejecutará una query automáticamente.

## DB-PREP-056

Pre-dispatch reprepare podrá permitirse cuando sea seguro.

## DB-PREP-057

Invalidated statement no aceptará nuevos leases.

## DB-PREP-058

Leased invalidated statement se cerrará al liberarse.

## DB-PREP-059

Close liberará recursos asociados.

## DB-PREP-060

Close será idempotente cuando sea viable.

## DB-PREP-061

Close failure no devolverá statement al cache.

## DB-PREP-062

Connection health podrá verse afectada por close/reset failure.

## DB-PREP-063

Prepared statement lifetime será hijo de physical connection lifetime.

## DB-PREP-064

Cursor lifetime no excederá el parent statement/connection requerido.

## DB-PREP-065

Cerrar cursor no implicará necesariamente cerrar statement.

## DB-PREP-066

Liberar statement no implicará necesariamente cerrar connection.

## DB-PREP-067

Physical connection lifetime será distinto de connection lease lifetime.

## DB-PREP-068

Prepared statement cache podrá sobrevivir requests sólo bajo reset seguro.

## DB-PREP-069

Request state no quedará almacenado en cached statement.

## DB-PREP-070

Tenant state no quedará almacenado accidentalmente.

## DB-PREP-071

Timeout state no quedará almacenado accidentalmente.

## DB-PREP-072

Result state no quedará almacenado accidentalmente.

## DB-PREP-073

FrankenPHP reuse requerirá isolation/reset.

## DB-PREP-074

RoadRunner reuse requerirá isolation/reset.

## DB-PREP-075

OpenSwoole reuse será concurrency-safe.

## DB-PREP-076

Prepared statement concurrency mode será explícito.

## DB-PREP-077

Default concurrency mode será EXCLUSIVE.

## DB-PREP-078

Shareable descriptor no implicará shareable driver resource.

## DB-PREP-079

Synchronization será runtime-neutral.

## DB-PREP-080

Statement wait policy será explícita.

## DB-PREP-081

Prepared statement budget será bounded.

## DB-PREP-082

Unknown server limit no significará unlimited.

## DB-PREP-083

Resource pressure desalojará idle resources antes de leased resources.

## DB-PREP-084

No todo prepared statement deberá entrar al cache.

## DB-PREP-085

Cache admission será policy-driven.

## DB-PREP-086

Cache policy no cambiará query semantics.

## DB-PREP-087

PreparedStatementFingerprint será distinto de instance ID.

## DB-PREP-088

Múltiples instances podrán compartir fingerprint.

## DB-PREP-089

Statement options serán estructuradas.

## DB-PREP-090

Options incompatibles producirán diferentes reuse identities.

## DB-PREP-091

Binding values pertenecerán al Parameter Binding System.

## DB-PREP-092

Prepared result metadata podrá ser parcial.

## DB-PREP-093

Result metadata timing será capability-driven.

## DB-PREP-094

Prepared statements no constituirán por sí solos seguridad SQL completa.

## DB-PREP-095

Runtime values se enviarán como bindings.

## DB-PREP-096

Identifier safety será responsabilidad de compilation.

## DB-PREP-097

Emulated prepare security será explícitamente modelada.

## DB-PREP-098

Raw SQL no será sanitizado aquí.

## DB-PREP-099

Sensitive values no aparecerán en diagnostics.

## DB-PREP-100

Server statement names no contendrán secretos.

## DB-PREP-101

Platform differences estarán encapsuladas en adapters/capabilities.

## DB-PREP-102

MariaDB no heredará ciegamente MySQL behavior.

## DB-PREP-103

PostgreSQL session sensitivity será modelable.

## DB-PREP-104

SQLite connection affinity será respetada.

## DB-PREP-105

Extension-provided functions podrán participar en session compatibility.

## DB-PREP-106

Extension-provided collations podrán participar en session compatibility.

## DB-PREP-107

Statement dependencies serán estructuradas.

## DB-PREP-108

Known dependency changes podrán invalidar statements afectados.

## DB-PREP-109

Cuando la precisión de invalidation sea insuficiente se preferirá invalidación conservadora.

## DB-PREP-110

Statement state será distinto de health.

## DB-PREP-111

IDLE + SUSPECT no será automáticamente reusable.

## DB-PREP-112

Failure impact será estructurado.

## DB-PREP-113

Prepared Statement System no reparará transacciones.

## DB-PREP-114

Telemetry no incluirá bindings sensibles.

## DB-PREP-115

Cache hit/miss será observable.

## DB-PREP-116

Preparation duration será observable.

## DB-PREP-117

Reuse count será observable.

## DB-PREP-118

Reset failures serán observables.

## DB-PREP-119

Metrics no determinarán automáticamente cache policy.

## DB-PREP-120

Extensions serán tipadas.

## DB-PREP-121

Extension registries estarán frozen tras bootstrap.

## DB-PREP-122

Registry ambiguity no usará last-wins.

## DB-PREP-123

Platform scoping será explícito.

## DB-PREP-124

Core manager evitará vendor conditionals dispersos.

## DB-PREP-125

Prepared Statement System no dependerá de ORM.

## DB-PREP-126

Prepared Statement System no dependerá de Optimizer.

## DB-PREP-127

Prepared Statement System no dependerá de Planner.

## DB-PREP-128

Prepared Statement System no dependerá de EntityManager.

## DB-PREP-129

Prepared Statement System no dependerá de UnitOfWork.

## DB-PREP-130

Prepared Statement System no dependerá de IdentityMap.

## DB-PREP-131

Prepared Statement System no generará SQL.

## DB-PREP-132

Prepared Statement System no reinterpretará semantics.

## DB-PREP-133

Prepared Statement System no resolverá application authorization.

## DB-PREP-134

Prepared Statement System no decidirá transaction retry.

## DB-PREP-135

Prepared Statement System no decidirá query retry.

## DB-PREP-136

Prepared Statement System no hidratará entities.

## DB-PREP-137

Prepared statement cache deberá limpiar todos sus recursos al cerrar connection.

## DB-PREP-138

No existirá resource resurrection después de CLOSED.

## DB-PREP-139

Invalidation será monotónica salvo creación de una nueva instance.

## DB-PREP-140

Un INVALID statement no regresará a HEALTHY mediante reset ordinario.

## DB-PREP-141

Reprepare creará una nueva prepared statement instance.

## DB-PREP-142

PreparedStatementInstanceId nunca se reutilizará dentro del scope requerido.

## DB-PREP-143

Cache key collision no deberá permitir reuse incompatible.

## DB-PREP-144

Fingerprint collision deberá validarse mediante descriptor compatibility cuando sea necesario.

## DB-PREP-145

Connection identity deberá verificarse antes del reuse.

## DB-PREP-146

Session compatibility deberá verificarse antes del reuse.

## DB-PREP-147

Driver preparation contract deberá verificarse antes del reuse.

## DB-PREP-148

Placeholder contract deberá verificarse antes del reuse.

## DB-PREP-149

Result options deberán verificarse antes del reuse.

## DB-PREP-150

Active statement resources tendrán ownership único y explícito.

## DB-PREP-151

No habrá concurrent mutation de statement state sin sincronización.

## DB-PREP-152

Cache eviction será concurrency-safe.

## DB-PREP-153

Connection invalidation será concurrency-safe.

## DB-PREP-154

Lease release será concurrency-safe.

## DB-PREP-155

Statement reset será concurrency-safe respecto a nuevos leases.

## DB-PREP-156

No se adquirirá lease durante RESETTING.

## DB-PREP-157

No se adquirirá lease durante CLOSING.

## DB-PREP-158

No se adquirirá lease sobre BROKEN.

## DB-PREP-159

Prepared statement reuse nunca cambiará el significado del command.

## DB-PREP-160

Prepared Statement System será una frontera de recursos, no una segunda capa de compilación.

---

# 234. Invariante maestro de compatibilidad

Un prepared statement `P` podrá ejecutar un command `C` sólo si:

```text
Compatible(P, C)
=
SameCompiledCommandCompatibility
∧
SamePhysicalConnection
∧
CompatibleSession
∧
CompatibleDriverContract
∧
CompatiblePreparationOptions
∧
Healthy(P)
```

---

# 235. Invariante maestro de reuse

```text
Reusable(P)
=
State(P) = IDLE
∧
Health(P) = HEALTHY
∧
NoActiveCursor(P)
∧
NoPendingResult(P)
∧
NoStaleBindings(P)
∧
SessionCompatible(P)
∧
ConnectionAlive(P)
```

---

# 236. Invariante maestro de lease

```text
ExclusiveLease(P)
⇒
ActiveExecutionCount(P) ≤ 1
```

por defecto.

---

# 237. Invariante maestro de conexión

```text
PreparedStatement P
belongs-to
PhysicalConnection C
```

implica:

```text
Dead(C)
⇒
Invalid(P)
```

---

# 238. Invariante maestro de invalidación

```text
Invalid(P)
⇒
¬Acquirable(P)
```

---

# 239. Invariante maestro de reset

```text
ReusableAfterExecution(P)
⇒
ResetSuccessful(P)
```

cuando el driver requiere reset.

---

# 240. Invariante maestro de persistent runtime

Antes de devolver un statement reusable al pool:

```text
NoRequestSpecificBindings
∧
NoActiveCursor
∧
NoPendingResults
∧
NoTemporaryOptions
∧
NoSensitiveReferences
∧
CompatibleSessionState
```

---

# 241. Ejemplo — cache hit

```text
Command C1
   ↓
Connection C7
   ↓
CacheKey K1
   ↓
P12 IDLE + HEALTHY
   ↓
reset not required
   ↓
lease P12
   ↓
return PreparedStatementLease
```

No ocurre `prepare()`.

---

# 242. Ejemplo — cache miss

```text
Command C1
   ↓
Connection C7
   ↓
CacheKey K1
   ↓
MISS
   ↓
Driver prepare()
   ↓
PreparedStatement P13
   ↓
register
   ↓
lease
```

---

# 243. Ejemplo — busy cache hit

```text
K1
├── P12 RESULT_ACTIVE
└── P13 LEASED
```

Policy:

```text
PREPARE_NEW
```

Resultado:

```text
P14
```

---

# 244. Ejemplo — invalid session

```text
Prepared P1
session fingerprint = S1

Current connection
session fingerprint = S2
```

Si:

```text
S1 !~ S2
```

entonces:

```text
P1
→ INVALID
```

---

# 245. Ejemplo — connection reconnect

```text
Physical Connection C1
   ├── P1
   ├── P2
   └── P3

network failure
   ↓
C1 BROKEN
   ↓
P1 INVALID
P2 INVALID
P3 INVALID

new physical connection C2
   ↓
new prepared statements
```

---

# 246. Ejemplo — streaming result

```text
Prepared P1
   ↓
Lease L1
   ↓
execute
   ↓
ResultCursor R1
```

Mientras:

```text
R1 open
```

entonces:

```text
P1 = RESULT_ACTIVE
L1 = owned by R1
```

Al cerrar:

```text
R1.close()
   ↓
drain/close
   ↓
reset P1
   ↓
P1 IDLE
   ↓
release L1
```

---

# 247. Ejemplo — reset failure

```text
P1 RESULT_ACTIVE
   ↓
cursor close
   ↓
driver reset
   ↓
FAIL
```

Resultado:

```text
P1 → INVALID
```

y si el driver indica session corruption:

```text
Connection → SUSPECT/BROKEN
```

---

# 248. Ejemplo — invalid cached statement before dispatch

```text
cache hit P1
   ↓
driver reports stale prepared handle
   ↓
invalidate P1
   ↓
prepare P2
   ↓
return P2 lease
```

Esto puede ser válido porque:

```text
query execution never started
```

---

# 249. Ejemplo — invalidation after dispatch

```text
P1 execute()
   ↓
connection error
```

Prepared Statement System no hará:

```text
prepare P2
execute again
```

porque eso sería retry.

---

# 250. Anti-patterns

## Anti-pattern 1 — global SQL cache

```php
static array $statements = [];

$statements[$sql] ??= $pdo->prepare($sql);
```

Incorrecto.

---

# 251. Anti-pattern 2 — connection-agnostic statement

```text
Prepared P1
→ Connection A
→ Connection B
```

Prohibido salvo driver contract extraordinario y explícito.

---

# 252. Anti-pattern 3 — cached bindings

```php
$statement->lastBindings = $bindings;
```

sobre resource persistent sin cleanup.

---

# 253. Anti-pattern 4 — busy reuse

```text
P1 streaming rows
+
Execution B uses P1
```

sin multiplexing capability.

---

# 254. Anti-pattern 5 — infinite drain

```php
while ($statement->nextResult()) {
    // forever
}
```

sin budget.

---

# 255. Anti-pattern 6 — retry hidden in prepare system

```php
catch (DriverException $e) {
    $statement = $this->prepareAgain();
    return $statement->execute();
}
```

Prohibido.

---

# 256. Anti-pattern 7 — last-wins strategy

```php
$strategies[$driver] = $newStrategy;
```

silenciosamente.

---

# 257. Anti-pattern 8 — cache forever

```text
no size limit
no age limit
no invalidation
```

---

# 258. Anti-pattern 9 — statement state in singleton

```php
$this->currentStatement = $statement;
```

en manager compartido.

---

# 259. Anti-pattern 10 — treating emulated as native

```text
emulated prepare
=
native prepare
```

sin capability distinction.

---

# 260. Fórmula arquitectónica

```text
Prepared Statement System
=
Preparation Strategy Resolution
+
Connection Affinity
+
Session Compatibility
+
Prepared Statement Identity
+
Driver Preparation
+
Lease Management
+
Reuse
+
Bounded Cache
+
Reset
+
Result Draining
+
Health Evaluation
+
Invalidation
+
Resource Ownership
+
Concurrency Control
+
Persistent Runtime Isolation
+
Security
+
Telemetry
```

---

# 261. Fórmula de adquisición

```text
AcquirePreparedStatement
=
Validate
+
ResolveStrategy
+
BuildIdentity
+
LookupCache
+
ValidateCandidate
+
ResetCandidate
+
PrepareOnMiss
+
AcquireLease
```

---

# 262. Fórmula de liberación

```text
ReleasePreparedStatement
=
ClosePendingResults
+
ClearBindings
+
ResetDriverState
+
ValidateHealth
+
ValidateSessionCompatibility
+
ReturnToCacheOrClose
```

---

# 263. Principio final

El sistema deberá cumplir:

> **A compiled command may be reusable everywhere its compilation contract permits; a prepared statement is reusable only where its runtime resource contract permits.**

En forma resumida:

```text
Compiled Command
=
Reusable Representation
```

```text
Prepared Statement
=
Managed Runtime Resource
```

y:

```text
Reuse
=
Compatibility
+
Clean State
+
Healthy Resource
+
Correct Ownership
```

Nunca:

```text
same SQL
⇒
safe reuse
```

---

# 264. Relación con el siguiente documento

Una vez adquirido:

```text
PreparedStatementLease
```

es necesario transformar:

```text
RuntimeBindingSet
```

en los valores y tipos concretos que espera el driver.

La siguiente frontera es:

```text
PreparedStatementLease
        +
CompiledBindingLayout
        +
RuntimeBindingSet
        ↓
Parameter Binding System
        ↓
DriverBindingSet
        ↓
Statement Execution
```

---

# 265. Siguiente documento

```text
80_DATABASE_PARAMETER_BINDING_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Parameter Binding System
├── Binding Architecture
├── Parameter Identity
├── Parameter Slot Model
├── Runtime Binding Set
├── Compiled Binding Layout
├── Placeholder Mapping
├── Binding Resolution
├── Value Conversion
├── Driver Type Resolution
├── Null Binding
├── Boolean Binding
├── Numeric Binding
├── String Binding
├── Binary Binding
├── Date/Time Binding
├── Enum Binding
├── JSON Binding
├── UUID Binding
├── LOB Binding
├── Stream Binding
├── Array Binding
├── Custom Type Binding
├── Input/Output Parameters
├── Binding Validation
├── Binding Security
├── Sensitive Values
├── Binding Lifecycle
├── Resource Ownership
├── Driver Binding Contract
├── Binding Diagnostics
├── Binding Telemetry
├── Extension System
├── Persistent Runtime Safety
└── Architectural Invariants
```

manteniendo siempre:

```text
ParameterId
≠
ExecutionParameterSlotId
≠
SQLPlaceholder
≠
DriverBindingPosition
```