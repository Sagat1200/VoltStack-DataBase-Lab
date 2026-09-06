# 16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md

# VoltStack Quantum Database
## Connection State and Reset System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 16 — Database Connection State and Reset System  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial para:

- representar el estado mutable de una conexión física;
- detectar contaminación de sesión;
- registrar mutaciones conocidas;
- clasificar estado desconocido;
- determinar si una conexión puede reutilizarse;
- construir planes de reset;
- ejecutar estrategias de reset;
- verificar el resultado;
- descartar conexiones cuyo estado no pueda garantizarse;
- proteger workers persistentes contra contaminación entre execution scopes.

El sistema pertenece a:

```text
VoltStack/Quantum/Database
```

y complementa directamente:

```text
14_DATABASE_CONNECTION_POOLING_SYSTEM.md
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM.md
```

---

# 2. Problema fundamental

Una conexión física no es solamente:

```text
socket + credentials
```

Una conexión mantiene estado.

Por ejemplo:

```sql
BEGIN;

SET ROLE administrator;

SET search_path TO tenant_42;

SET TIME ZONE 'UTC';

CREATE TEMPORARY TABLE temporary_report (...);
```

Después de ejecutar estas operaciones, la conexión ya no se encuentra necesariamente en el mismo estado en que fue adquirida.

En un worker persistente:

```text
Request A
   │
   ▼
Physical Connection P1
   │
   ▼
mutates session
   │
   ▼
returns P1
   │
   ▼
Request B
```

si `P1` se entrega directamente a Request B:

```text
Request B
inherits state from Request A
```

pueden aparecer:

- errores funcionales;
- contaminación entre tenants;
- privilegios incorrectos;
- transacciones huérfanas;
- problemas de seguridad;
- comportamiento no determinista;
- bugs extremadamente difíciles de reproducir.

---

# 3. Principio arquitectónico

VoltStack adoptará:

> Una conexión física sólo puede reutilizarse cuando su estado efectivo sea compatible con un baseline conocido.

Formalmente:

```text
Reusable(Connection)
=
Healthy(Connection)
AND
Resettable(Connection)
AND
VerifiedBaseline(Connection)
```

No será suficiente comprobar:

```text
connection is alive
```

También deberá comprobarse conceptualmente:

```text
connection is safe
```

---

# 4. Health ≠ State

Ésta será una separación fundamental.

```text
Health
=
¿la conexión funciona?

State
=
¿en qué condición lógica/session se encuentra?
```

Ejemplo:

```text
SELECT 1
→ success
```

puede demostrar:

```text
connection alive
```

pero no demuestra:

```text
no active transaction
default role
correct schema
clean session variables
no open cursor
no temporary objects
```

Por tanto:

```text
Health Check
≠
State Reset
```

---

# 5. State Reset ≠ Connection Reconnect

También deberán distinguirse:

```text
RESET
```

y:

```text
RECONNECT
```

Reset intenta recuperar la misma conexión física.

Reconnect:

```text
close P1
create P2
```

produce una nueva conexión física.

---

# 6. Arquitectura general

```text
PhysicalConnection
       │
       ▼
ConnectionStateTracker
       │
       ▼
ConnectionStateSnapshot
       │
       ▼
ConnectionResetPlanner
       │
       ▼
ConnectionResetPlan
       │
       ▼
ConnectionResetExecutor
       │
       ├── Transaction Reset
       ├── Cursor Reset
       ├── Statement Reset
       ├── Session Reset
       ├── Platform Reset
       └── Driver Reset
       │
       ▼
ConnectionResetVerifier
       │
       ├── CLEAN
       ├── CLEAN_WITH_WARNINGS
       └── UNSAFE
```

Si:

```text
UNSAFE
```

entonces:

```text
PhysicalConnection
→ TAINTED
→ RETIRE
→ CLOSE
```

---

# 7. Objetivos

El sistema deberá garantizar:

```text
deterministic reset
known baseline
tenant isolation
transaction isolation
request isolation
worker safety
driver portability
platform specialization
low hot-path overhead
explicit unsafe-state handling
```

---

# 8. No objetivos

Este subsystem no deberá decidir:

```text
query retries
transaction retries
replica failover
query planning
ORM state
EntityManager cleanup
tenant selection
connection routing
```

Puede colaborar con esos sistemas, pero no reemplazarlos.

---

# 9. Modelo de estado

VoltStack distinguirá al menos:

```text
Physical Lifecycle State
Connection Health State
Session State
Transaction State
Cursor State
Statement State
Native Access State
Reset State
```

No deberán comprimirse todos en un único enum.

---

# 10. Physical Lifecycle State

Definido principalmente por el documento 15:

```text
NEW
CONNECTING
INITIALIZING
READY
IDLE
LEASED
RESETTING
RETIRING
BROKEN
TAINTED
CLOSED
```

Este documento no sustituye esa máquina de estados.

---

# 11. Session State

El estado de sesión describe modificaciones realizadas dentro de la sesión física.

Estados conceptuales:

```text
BASELINE
DIRTY_KNOWN
DIRTY_UNKNOWN
RESETTING
VERIFICATION_REQUIRED
```

---

# 12. BASELINE

Significa:

> VoltStack considera que la sesión cumple el perfil esperado para ser entregada a un nuevo consumidor.

No significa necesariamente:

```text
database vendor defaults
```

El baseline es el baseline configurado por VoltStack.

---

# 13. DIRTY_KNOWN

VoltStack conoce qué partes del estado fueron modificadas.

Ejemplo:

```text
role changed
timezone changed
schema changed
```

Por tanto puede construir un reset específico.

---

# 14. DIRTY_UNKNOWN

VoltStack sabe o sospecha que existe mutación, pero no puede determinar completamente cuál.

Ejemplo:

```text
mutable native connection access
```

---

# 15. RESETTING

El estado está siendo restaurado.

Durante este estado la conexión:

```text
must not be leased
```

---

# 16. VERIFICATION_REQUIRED

El reset fue ejecutado, pero la política exige validación antes de declarar `BASELINE`.

---

# 17. ConnectionSessionProfile

El baseline esperado se representará mediante un objeto estructurado.

Ejemplo conceptual:

```php
final readonly class ConnectionSessionProfile
{
    public function __construct(
        public ?string $database,
        public ?string $schema,
        public ?string $role,
        public ?string $timezone,
        public ?string $encoding,
        public ?string $isolationLevel,
        public bool $autocommit,
        public array $sessionVariables,
    ) {}
}
```

La estructura exacta podrá evolucionar.

---

# 18. Baseline configurable

El baseline no deberá asumir que:

```text
"default vendor state"
```

es siempre correcto.

Ejemplo:

```text
timezone = UTC
encoding = utf8mb4
schema = public
role = application
```

puede ser el baseline real de una aplicación.

---

# 19. Baseline source

Podrá derivarse de:

```text
CompiledDatabaseConfiguration
Platform defaults
Connection definition
Runtime policy
Security policy
```

---

# 20. Baseline compilation

Preferencia:

```text
Configuration
      │
      ▼
SessionProfileCompiler
      │
      ▼
CompiledSessionProfile
```

para evitar recalcularlo en cada release.

---

# 21. Immutable baseline

Una vez compilado:

```text
CompiledConnectionSessionProfile
```

deberá ser inmutable.

---

# 22. Baseline identity

Podrá existir:

```text
SessionProfileFingerprint
```

para detectar incompatibilidades entre una conexión existente y una nueva configuración.

---

# 23. Baseline generation

Una conexión física podrá registrar:

```text
sessionProfileGeneration
```

para saber bajo qué baseline fue inicializada.

---

# 24. State ownership

El estado físico pertenece a:

```text
PhysicalConnection
```

no a:

```text
LogicalConnection
EntityManager
QueryBuilder
Repository
Model
```

---

# 25. ConnectionStateTracker

Componente conceptual:

```php
interface ConnectionStateTrackerInterface
{
    public function snapshot(
        PhysicalConnectionInterface $connection
    ): ConnectionStateSnapshot;

    public function record(
        PhysicalConnectionInterface $connection,
        ConnectionStateMutation $mutation
    ): void;
}
```

---

# 26. Objetivo del tracker

El tracker permite evitar que cada release necesite consultar al servidor para descubrir todo el estado.

Flujo:

```text
operation
   │
   ▼
known mutation
   │
   ▼
StateTracker
   │
   ▼
dirty flags
```

---

# 27. State tracking strategy

VoltStack utilizará una estrategia híbrida:

```text
Tracked State
+
Platform Knowledge
+
Driver Knowledge
+
Conservative Unknown-State Handling
```

---

# 28. No full server introspection

No se intentará reconstruir absolutamente todo el estado de una sesión mediante queries de introspección en cada release.

Eso sería demasiado costoso.

---

# 29. Dirty flags

El tracker podrá mantener flags conceptuales como:

```text
TRANSACTION_DIRTY
ROLE_DIRTY
SCHEMA_DIRTY
DATABASE_DIRTY
TIMEZONE_DIRTY
ENCODING_DIRTY
ISOLATION_DIRTY
AUTOCOMMIT_DIRTY
SESSION_VARIABLES_DIRTY
STATEMENTS_DIRTY
CURSORS_DIRTY
TEMPORARY_OBJECTS_DIRTY
ADVISORY_LOCKS_DIRTY
NATIVE_STATE_UNKNOWN
PLATFORM_STATE_DIRTY
```

---

# 30. Dirty mask

Internamente podrá utilizarse un bitmask eficiente:

```text
ConnectionDirtyMask
```

para el hot path.

---

# 31. Dirty mask advantage

Permite:

```text
if clean
→ fast release

if dirty
→ targeted reset
```

sin recorrer estructuras complejas.

---

# 32. StateMutation

Las mutaciones conocidas podrán representarse como:

```text
ConnectionStateMutation
```

con categorías especializadas.

---

# 33. Mutation examples

```text
TransactionStarted
RoleChanged
SchemaChanged
TimezoneChanged
IsolationChanged
SessionVariableChanged
CursorOpened
CursorClosed
TemporaryObjectCreated
NativeMutableAccessGranted
```

---

# 34. State journal

Opcionalmente, desarrollo/debug podrá mantener:

```text
ConnectionStateJournal
```

con una secuencia de mutaciones.

---

# 35. Production state journal

No deberá mantenerse un historial costoso por defecto.

Preferencia:

```text
dirty mask
+
minimal metadata
```

---

# 36. StateSnapshot

Ejemplo conceptual:

```php
final readonly class ConnectionStateSnapshot
{
    public function __construct(
        public ConnectionDirtyMask $dirtyMask,
        public TransactionState $transactionState,
        public CursorStateSummary $cursors,
        public StatementStateSummary $statements,
        public NativeAccessState $nativeAccess,
        public SessionState $sessionState,
    ) {}
}
```

---

# 37. Snapshot semantics

El snapshot representa:

```text
what VoltStack currently knows
```

No necesariamente:

```text
complete server reality
```

Esta diferencia deberá ser explícita.

---

# 38. Knowledge level

Podrá existir:

```text
KNOWN
PARTIALLY_KNOWN
UNKNOWN
```

como clasificación adicional.

---

# 39. Unknown-state principle

Cuando el framework no puede demostrar seguridad:

```text
unknown
≠
clean
```

---

# 40. Conservative default

Preferencia:

```text
unknown
→ strict reset

strict reset cannot guarantee baseline
→ discard
```

---

# 41. Transaction state

Se distinguirán estados como:

```text
NONE
ACTIVE
ROLLBACK_ONLY
ABORTED
COMMIT_UNKNOWN
ROLLBACK_UNKNOWN
UNKNOWN
```

---

# 42. Transaction state ownership

`TransactionManager` mantiene semántica de alto nivel.

`Connection State System` mantiene información suficiente para determinar:

```text
can this physical resource be reused?
```

---

# 43. Active transaction on release

```text
ACTIVE
→ rollback required
```

antes de reuse.

---

# 44. ABORTED transaction

Ejemplo común:

```text
PostgreSQL transaction aborted
```

requiere:

```text
ROLLBACK
```

antes de continuar.

---

# 45. COMMIT_UNKNOWN

Si existe incertidumbre sobre el resultado de commit:

```text
connection state unsafe
```

La conexión deberá normalmente retirarse.

---

# 46. Transaction cleanup

El reset system puede ejecutar rollback técnico para limpiar el recurso.

Pero no decide la semántica de retry de la transacción.

---

# 47. Cursor state

El tracker podrá conocer:

```text
openCursorCount
streamingCursorCount
unknownNativeCursorState
```

---

# 48. Open cursor

Antes del reset:

```text
open cursor
→ close cursor
```

---

# 49. Cursor close failure

```text
close fails
→ protocol/session may be unsafe
→ taint/discard
```

---

# 50. Statement state

El sistema deberá distinguir:

```text
ephemeral statements
cached prepared statements
active statements
native statements
```

---

# 51. Prepared statement cache

Si en el futuro existe cache de prepared statements:

```text
Statement Cache
```

deberá definir qué estado es reusable y qué estado debe limpiarse.

---

# 52. Statement cache ≠ dirty state

Un prepared statement correctamente reseteado puede permanecer reusable.

Un statement con cursor/result activo no.

---

# 53. Temporary objects

Objetos temporales pueden incluir:

```text
temporary tables
temporary views
temporary sequences
vendor-specific temporary structures
```

---

# 54. Temporary object problem

Algunas bases mantienen objetos temporales durante toda la sesión.

Por tanto:

```text
request A creates temp table
connection returns to pool
request B receives connection
```

puede causar contaminación.

---

# 55. Temporary object strategy

Las plataformas deberán declarar capabilities como:

```text
supportsSessionTemporaryObjects
supportsTemporaryObjectEnumeration
supportsTemporaryObjectReset
```

---

# 56. Temporary object policy

Opciones:

```text
TRACK_AND_DROP
NATIVE_RESET
DISCARD_CONNECTION
UNSUPPORTED
```

---

# 57. Conservative temporary object policy

Si un escape hatch puede crear objetos temporales no rastreados:

```text
DIRTY_UNKNOWN
```

---

# 58. Advisory locks

Los advisory locks también pueden ser session-scoped.

Ejemplos conceptuales:

```text
GET_LOCK
pg_advisory_lock
```

---

# 59. Advisory lock tracking

El sistema deberá considerar:

```text
advisory lock acquired
→ state dirty
```

---

# 60. Lock release

Antes de reuse:

```text
release known locks
```

o utilizar:

```text
platform reset primitive
```

si existe.

---

# 61. Unknown advisory locks

Si no pueden enumerarse o liberarse con seguridad:

```text
discard
```

puede ser la estrategia correcta.

---

# 62. Session variables

Ejemplos:

```text
sql_mode
search_path
timezone
application_name
statement_timeout
lock_timeout
work_mem
foreign_key_checks
```

según plataforma.

---

# 63. SessionVariableRegistry

Podrá existir una clasificación:

```text
portable
platform-specific
security-sensitive
resettable
non-resettable
```

---

# 64. Known session variables

Cuando VoltStack modifica una variable mediante API propia:

```text
StateTracker
```

deberá registrarlo.

---

# 65. Raw SET statements

Una query raw:

```sql
SET something = ...
```

puede ser difícil de interpretar universalmente.

---

# 66. Raw SQL policy

VoltStack no deberá intentar construir un parser completo de cada dialecto sólo para detectar todas las mutaciones.

---

# 67. Raw mutation declaration

Podrá existir una API avanzada:

```php
DB::rawSession(
    sql: 'SET ...',
    effects: [...]
);
```

para declarar efectos conocidos.

---

# 68. Unsafe raw SQL

Raw SQL que pueda alterar sesión sin metadata podrá marcar:

```text
session state potentially unknown
```

según política.

---

# 69. Native access state

Se distinguirán al menos:

```text
NONE
READ_ONLY
MUTABLE_TRACKED
MUTABLE_UNTRACKED
```

---

# 70. Native READ_ONLY

Acceso nativo estrictamente observacional no deberá ensuciar automáticamente la conexión.

---

# 71. MUTABLE_TRACKED

El consumidor declara las mutaciones realizadas.

---

# 72. MUTABLE_UNTRACKED

VoltStack ya no puede garantizar conocer la sesión.

Resultado:

```text
DIRTY_UNKNOWN
```

---

# 73. Unsafe native escape hatch

Ejemplo conceptual:

```php
DB::connection()->unsafeNative(function ($native) {
    // arbitrary native operations
});
```

deberá comunicar explícitamente su no-portabilidad y sus implicaciones de lifecycle.

---

# 74. Native handle retention

El native handle no deberá escapar de su lease.

Prohibido conceptualmente:

```php
$pdo = DB::native();

$GLOBALS['pdo'] = $pdo;
```

---

# 75. Native handle validity

Su validez estará limitada a:

```text
lease lifetime
```

---

# 76. Tenant state

El tenant puede afectar:

```text
database
schema
search_path
role
session variables
```

---

# 77. Tenant isolation principle

```text
Tenant A session state
```

nunca deberá aparecer en:

```text
Tenant B
```

---

# 78. Tenant state ownership

`Multitenancy` decide:

```text
which tenant
```

Connection State decide:

```text
whether physical session is clean
```

---

# 79. Tenant switching

Cambiar tenant dentro de la misma conexión física deberá pasar por una estrategia explícita.

---

# 80. Prefer baseline reset

Preferencia:

```text
Tenant A
→ baseline
→ Tenant B
```

en lugar de:

```text
Tenant A
→ directly mutate into Tenant B
```

cuando esto simplifique seguridad.

---

# 81. Security-sensitive state

Se clasificará especialmente:

```text
role
authorization context
schema
database/catalog
tenant session variable
row-level-security context
```

---

# 82. Security reset failure

Cualquier fallo limpiando estado security-sensitive deberá producir:

```text
DISCARD
```

por defecto.

---

# 83. Reset modes

VoltStack definirá conceptualmente:

```text
FAST
TARGETED
STRICT
NATIVE
DISCARD
```

---

# 84. FAST reset

Se utiliza cuando:

```text
dirtyMask = CLEAN
```

y no existe política que requiera validación adicional.

Puede consistir prácticamente en:

```text
release accounting
→ idle
```

---

# 85. TARGETED reset

Se conocen las mutaciones.

Ejemplo:

```text
ROLE_DIRTY
TIMEZONE_DIRTY
```

Plan:

```text
reset role
reset timezone
verify required invariants
```

---

# 86. STRICT reset

Se utiliza cuando:

```text
state partially known
native mutable access occurred
high security policy enabled
```

Puede ejecutar una secuencia más amplia.

---

# 87. NATIVE reset

Algunos drivers/plataformas pueden ofrecer una operación de reset de sesión.

Ejemplo conceptual:

```text
reset connection/session
```

sin cerrar completamente el transporte.

---

# 88. Native reset capability

Se modelará mediante capabilities.

No mediante:

```php
if ($driver === 'mysql') {
}
```

---

# 89. DISCARD

Cuando no existe estrategia segura:

```text
close connection
```

es el reset definitivo.

---

# 90. Reset strategy selection

```text
ConnectionStateSnapshot
          │
          ▼
ResetPolicy
          │
          ▼
ResetStrategySelector
          │
          ├── FAST
          ├── TARGETED
          ├── STRICT
          ├── NATIVE
          └── DISCARD
```

---

# 91. ResetPolicy

Podrá considerar:

```text
dirty state
platform capabilities
driver capabilities
security policy
native access
tenant state
transaction state
connection age
reset cost
configuration
```

---

# 92. Security over performance

La selección deberá cumplir:

```text
safety first
optimization second
```

---

# 93. ResetPlan

Objeto inmutable conceptual:

```php
final readonly class ConnectionResetPlan
{
    public function __construct(
        public ConnectionResetMode $mode,
        public array $steps,
        public bool $verificationRequired,
        public ConnectionReuseDecision $successDecision,
    ) {}
}
```

---

# 94. Reset steps

Ejemplos:

```text
CloseOpenCursors
CloseActiveStatements
RollbackTransaction
ReleaseAdvisoryLocks
DropTemporaryObjects
ResetRole
ResetSchema
ResetDatabase
ResetTimezone
ResetIsolation
ResetAutocommit
ResetSessionVariables
DriverNativeReset
PlatformReset
VerifyBaseline
```

---

# 95. Reset plan determinism

Mismo:

```text
snapshot
+
configuration
+
capabilities
+
policy
```

deberá producir el mismo plan.

---

# 96. Reset ordering

El orden deberá ser explícito.

Ejemplo:

```text
1 close cursors
2 resolve transaction
3 release session locks
4 remove temporary state
5 reset authorization state
6 reset schema/catalog
7 reset session variables
8 verify
```

---

# 97. Dependency-aware reset

Algunos pasos dependen de otros.

Ejemplo:

```text
ROLLBACK
```

puede ser necesario antes de:

```text
RESET ROLE
```

dependiendo de plataforma.

---

# 98. ResetStep

Contrato conceptual:

```php
interface ConnectionResetStepInterface
{
    public function supports(
        ConnectionResetContext $context
    ): bool;

    public function execute(
        ConnectionResetContext $context
    ): ConnectionResetStepResult;
}
```

---

# 99. Reset steps are not arbitrary middleware

No deberán convertirse en una lista abierta donde cualquier paquete ejecute código sin restricciones.

---

# 100. Reset step categories

Podrán existir:

```text
CORE
DRIVER
PLATFORM
SECURITY
EXTENSION
```

con reglas claras.

---

# 101. Core reset steps

Incluyen:

```text
cursor cleanup
statement cleanup
transaction cleanup
state tracker cleanup
```

---

# 102. Driver reset steps

Manejan primitivas nativas.

---

# 103. Platform reset steps

Manejan semántica del servidor.

---

# 104. Security reset steps

Restauran:

```text
role
authorization context
security session variables
```

---

# 105. Extension reset steps

Sólo podrán limpiar estado introducido por extensiones declaradas.

---

# 106. Extension state declaration

Una extensión que introduce session state deberá declarar:

```text
how it becomes dirty
how it resets
whether reset is verifiable
failure behavior
```

---

# 107. ResetPlanner

Responsabilidad:

```text
state
→ reset plan
```

No ejecuta SQL directamente.

---

# 108. ResetExecutor

Responsabilidad:

```text
reset plan
→ execute steps
```

---

# 109. ResetVerifier

Responsabilidad:

```text
did we restore the required baseline?
```

---

# 110. Planner ≠ Executor ≠ Verifier

Esta separación permite:

```text
testing
diagnostics
dry-run
telemetry
platform specialization
```

---

# 111. Reset verification levels

Podrán existir:

```text
NONE
TRACKED
TARGETED
STRICT
```

---

# 112. NONE

Sólo para casos donde:

```text
state never became dirty
```

y la política lo permite.

---

# 113. TRACKED

Confía en:

```text
successful reset steps
+
state tracker
```

---

# 114. TARGETED

Consulta o verifica determinados invariantes críticos.

---

# 115. STRICT

Realiza las verificaciones máximas razonables soportadas por la plataforma.

---

# 116. Verification is capability-based

No todas las bases permiten consultar el mismo estado.

---

# 117. Verification result

```text
VERIFIED_CLEAN
ASSUMED_CLEAN
CLEAN_WITH_WARNINGS
UNVERIFIED
FAILED
```

---

# 118. Production policy

Una política podrá permitir:

```text
ASSUMED_CLEAN
```

para operaciones conocidas y seguras.

---

# 119. Security-critical policy

Podrá exigir:

```text
VERIFIED_CLEAN
```

para ciertos estados.

---

# 120. UNVERIFIED

Si la política exige verificación y no puede obtenerse:

```text
discard
```

---

# 121. ResetResult

Ejemplo:

```php
final readonly class ConnectionResetResult
{
    public function __construct(
        public ConnectionResetStatus $status,
        public ConnectionReuseDecision $reuseDecision,
        public ConnectionStateSnapshot $finalState,
        public array $stepResults,
    ) {}
}
```

---

# 122. Reuse decisions

```text
REUSE
REUSE_WITH_WARNING
RETIRE
DISCARD
```

---

# 123. REUSE

La conexión puede volver al Pool.

---

# 124. REUSE_WITH_WARNING

Sólo para condiciones explícitamente no críticas.

No deberá utilizarse para fallos de aislamiento o seguridad.

---

# 125. RETIRE

La conexión está limpia pero la policy indica que no debe reutilizarse.

Ejemplo:

```text
max lifetime reached
```

---

# 126. DISCARD

La conexión no es segura o está dañada.

---

# 127. Reset failure

Principio:

```text
reset failure
→ never silently return idle
```

---

# 128. Critical reset failure

Ejemplos:

```text
rollback failed
role reset failed
schema reset failed
native reset failed
unknown cursor state
```

normalmente:

```text
DISCARD
```

---

# 129. Non-critical reset failure

Ejemplo:

```text
optional diagnostic cleanup failed
```

puede permitir reuse si no afecta estado funcional.

---

# 130. Reset exception hierarchy

Podrá existir:

```text
ConnectionResetException
├── TransactionResetException
├── CursorResetException
├── StatementResetException
├── SessionResetException
├── SecurityStateResetException
├── NativeResetException
└── ResetVerificationException
```

---

# 131. Failure aggregation

El reset executor podrá registrar múltiples fallos cuando continuar cleanup sea seguro.

---

# 132. First failure does not always stop cleanup

Ejemplo:

```text
close statement fails
→ still attempt rollback
→ still attempt native close
```

---

# 133. But no unsafe continuation

Si un paso deja el protocolo en estado donde más comandos no son seguros:

```text
stop SQL reset
→ close native resource
```

---

# 134. Reset timeout

Reset tendrá su propio:

```text
reset deadline
```

o presupuesto derivado del lifecycle.

---

# 135. Reset must not hang worker

Un reset bloqueado indefinidamente es incompatible con persistent workers.

---

# 136. Reset timeout result

```text
timeout
→ tainted
→ discard
```

---

# 137. Cancellation during reset

El reset crítico no deberá cancelarse arbitrariamente si hacerlo puede devolver una conexión insegura.

---

# 138. Lifecycle cancellation vs cleanup

Aunque el request esté cancelado:

```text
database cleanup still required
```

---

# 139. Cleanup budget

El runtime podrá proporcionar un presupuesto específico para cleanup.

---

# 140. Driver-level reset

Driver podrá exponer:

```php
interface NativeConnectionResetInterface
{
    public function resetNativeConnection(
        NativeConnectionInterface $connection
    ): NativeResetResult;
}
```

si su tecnología lo soporta.

---

# 141. Native reset semantics

El contrato deberá documentar exactamente qué limpia.

Nunca asumir:

```text
native reset
=
everything
```

sin garantía.

---

# 142. Platform reset semantics

Platform deberá describir:

```text
transaction effects
session variables
roles
schemas
temporary objects
locks
prepared statements
```

que una operación nativa realmente resetea.

---

# 143. Capability metadata

Una capability estructurada podría expresar:

```text
sessionReset:
    supported: true
    clearsTransactions: true
    clearsRoles: true
    clearsTemporaryObjects: false
    clearsPreparedStatements: true
```

---

# 144. Structured capabilities

Para reset, un booleano:

```text
supportsReset = true
```

puede ser insuficiente.

---

# 145. ResetCapability

Podrá modelarse mediante:

```text
ConnectionResetCapability
```

con semántica estructurada.

---

# 146. Driver vs Platform reset

```text
Driver
→ exposes native reset mechanism

Platform
→ explains database/session semantics
```

---

# 147. MySQL/MariaDB considerations

MySQL y MariaDB pueden compartir mecanismos de transporte, pero no deberán asumirse idénticas todas sus semánticas.

Por tanto:

```text
shared Driver possibility
≠
identical Platform reset model
```

---

# 148. PostgreSQL considerations

Especial atención a:

```text
aborted transactions
search_path
SET ROLE
session settings
advisory locks
temporary objects
prepared statements
```

---

# 149. SQLite considerations

SQLite tiene un modelo diferente porque muchas configuraciones pertenecen a:

```text
connection
database file
PRAGMA state
transaction
```

y una conexión `:memory:` puede representar también la vida de la propia base de datos.

---

# 150. SQLite in-memory special case

Cerrar una conexión:

```text
:memory:
```

puede destruir los datos asociados.

Por tanto:

```text
discard connection
```

tiene consecuencias distintas.

---

# 151. SQLite capability specialization

El Platform deberá exponer estas diferencias sin contaminar el core con:

```php
if ($driver === 'sqlite')
```

---

# 152. Connection state API

La API pública ordinaria no deberá exponer libremente mutación directa del tracker.

---

# 153. Internal mutation recording

Las capas autorizadas podrán utilizar:

```text
ConnectionStateMutationRecorder
```

---

# 154. Mutation recorder

Ejemplo conceptual:

```php
$state->record(
    ConnectionStateMutation::roleChanged(
        from: $oldRole,
        to: $newRole,
    )
);
```

---

# 155. Semantic operations preferred

En vez de permitir:

```php
$state->markDirty('x');
```

preferir mutaciones semánticas tipadas.

---

# 156. Generic dirty fallback

Aun así deberá existir:

```text
markUnknownDirty()
```

para escape hatches.

---

# 157. State transition safety

El tracker deberá impedir inconsistencias obvias.

Ejemplo:

```text
CursorClosed
```

sin cursor registrado podrá producir diagnóstico en desarrollo.

---

# 158. Tracker is not authoritative server

No deberá convertirse en un segundo motor de base de datos.

Su objetivo es lifecycle safety.

---

# 159. State reset and pooling

Integración:

```text
Lease Release
     │
     ▼
Pool
     │
     ▼
State Snapshot
     │
     ▼
Reset Planner
     │
     ▼
Reset Executor
     │
     ▼
Verifier
     │
     ├── reusable → Pool Idle
     └── unsafe   → Close
```

---

# 160. Pool invariant

El Pool sólo deberá almacenar como `IDLE` conexiones cuyo reset haya sido aprobado.

---

# 161. Pool does not implement reset semantics

El Pool coordina:

```text
when
```

El State/Reset System determina:

```text
what/how
```

---

# 162. Connection lifecycle integration

Documento 15 responde:

```text
WHEN should reset happen?
```

Documento 16 responde:

```text
WHAT is dirty?
HOW should it be reset?
HOW do we know reuse is safe?
```

---

# 163. ConnectionManager integration

`ConnectionManager` no deberá ejecutar resets.

Su responsabilidad sigue siendo:

```text
connection selection/resolution/access
```

---

# 164. Driver integration

Driver aporta primitivas.

No decide la política global.

---

# 165. Platform integration

Platform aporta semántica y capabilities.

---

# 166. Transaction integration

TransactionManager notifica cambios como:

```text
transaction started
transaction committed
transaction rolled back
transaction aborted
```

---

# 167. Execution integration

Executor puede notificar:

```text
cursor opened
cursor closed
statement active
statement completed
connection-fatal error
```

---

# 168. Schema integration

Schema operations que alteren session context deberán registrar su efecto.

---

# 169. ORM independence

El State/Reset System no deberá conocer:

```text
Entity
EntityManager
UnitOfWork
IdentityMap
Repository
Relationship
```

---

# 170. ORM cleanup separate

```text
ORM state cleanup
```

y:

```text
physical connection reset
```

son operaciones diferentes.

---

# 171. Persistent runtime safety

Éste es uno de los objetivos principales.

```text
Request A
    │
    ▼
P1
    │
    ▼
dirty
    │
    ▼
reset
    │
    ▼
baseline
    │
    ▼
Request B
```

---

# 172. Forbidden persistent flow

```text
Request A
→ P1 dirty
→ Pool
→ Request B
```

sin reset.

---

# 173. FrankenPHP

FrankenPHP será el runtime de referencia inicial.

Después de cada execution scope:

```text
physical resources
→ baseline or retired
```

---

# 174. RoadRunner

Deberá utilizar exactamente el mismo contrato conceptual.

---

# 175. OpenSwoole

Cada coroutine tendrá:

```text
isolated lease ownership
```

y el tracker de cada conexión deberá estar protegido contra uso concurrente incompatible.

---

# 176. StateTracker scope

El tracker de una conexión física vive con:

```text
PhysicalConnection
```

no con el request.

Esto es necesario porque registra estado del recurso persistente.

---

# 177. Request-specific mutation metadata

Sin embargo, información diagnóstica del owner deberá eliminarse al release.

---

# 178. No tenant retention in tracker

Después de reset:

```text
current tenant
```

no deberá permanecer como contexto activo.

Podrá existir sólo información histórica no sensible y controlada para diagnostics.

---

# 179. Reset performance

El sistema deberá optimizar el caso más común:

```text
query executes
no session mutation
no transaction remains
no streaming cursor remains
```

---

# 180. Fast clean path

```text
Lease Release
     │
     ▼
DirtyMask == CLEAN?
     │
     ├── yes → lightweight checks → IDLE
     └── no  → Reset Plan
```

---

# 181. Avoid reset query per request

No se ejecutará necesariamente:

```sql
RESET ...
```

después de cada query o request si no existe estado que limpiar.

---

# 182. Tracking enables optimization

```text
state tracking
→ fewer unnecessary reset round trips
```

---

# 183. Strict mode

VoltStack podrá ofrecer:

```text
database.connections.reset.mode = strict
```

conceptualmente.

---

# 184. Balanced/default mode

Default recomendado:

```text
tracked + conservative fallback
```

---

# 185. Performance mode

Podrá existir una política más optimizada, pero nunca deberá permitir violar invariantes de seguridad declarados.

---

# 186. Development mode

Puede habilitar:

```text
mutation journal
strict verification
leak diagnostics
reset plan tracing
```

---

# 187. Production mode

Preferirá:

```text
dirty masks
compiled reset strategies
minimal allocations
sampled telemetry
```

---

# 188. Compiled reset strategy

Para una combinación estable:

```text
Driver
Platform
SessionProfile
SecurityPolicy
Capabilities
```

podrá precompilarse:

```text
CompiledConnectionResetStrategy
```

---

# 189. Compiled strategy advantage

Evita:

```text
registry lookups
capability recomputation
strategy discovery
reflection
```

en cada release.

---

# 190. Dynamic state remains runtime

El plan concreto todavía depende de:

```text
dirty mask
transaction state
cursor state
native access state
```

---

# 191. ResetPlan cache

Planes frecuentes podrían reutilizarse si son inmutables y están identificados por:

```text
DirtyMask
+
StrategyFingerprint
```

---

# 192. Do not cache unsafe dynamic data

Nunca almacenar en un plan compartido:

```text
tenant ID
credential
connection object
native handle
transaction object
```

---

# 193. Reset telemetry

Métricas sugeridas:

```text
database.connection.reset.total
database.connection.reset.duration
database.connection.reset.fast
database.connection.reset.targeted
database.connection.reset.strict
database.connection.reset.native
database.connection.reset.discard
database.connection.reset.failure
database.connection.reset.verification_failure
database.connection.state.unknown
database.connection.state.tainted
```

---

# 194. Reset reason telemetry

Razones con cardinalidad limitada:

```text
transaction
session
cursor
native_access
security
tenant
platform
unknown
```

---

# 195. Reset plan diagnostics

Development podrá mostrar:

```text
Connection P17

Dirty:
- ROLE
- SCHEMA
- TIMEZONE

Selected strategy:
TARGETED

Steps:
1. ResetRole
2. ResetSchema
3. ResetTimezone
4. VerifySecurityBaseline

Result:
VERIFIED_CLEAN
```

---

# 196. Debug Toolbar integration

Podrá mostrar:

```text
physical connection reuse
dirty state detected
reset strategy
reset duration
discard reason
```

---

# 197. Slow reset detection

Un reset excesivamente lento deberá ser observable.

---

# 198. Reset failure logging

Deberá incluir contexto técnico seguro:

```text
connection logical name
driver
platform
physical resource ID
dirty categories
failed reset step
```

sin secretos.

---

# 199. No SQL secrets

Queries de reset que incluyan información sensible deberán redactarse.

---

# 200. State fingerprint

Podrá existir:

```text
ConnectionStateFingerprint
```

para diagnostics y tests.

---

# 201. Fingerprint limitations

No deberá incluir:

```text
passwords
tokens
private keys
high-cardinality sensitive data
```

---

# 202. Reset hooks

Se permitirán hooks limitados:

```text
beforeReset
afterReset
resetFailed
```

principalmente observacionales.

---

# 203. Correctness cannot depend on event listener

Nunca:

```text
Event listener
→ responsible for rollback
```

como arquitectura principal.

---

# 204. Extension registration

Extensiones que introduzcan estado deberán registrarse durante bootstrap.

---

# 205. Frozen reset registry

Después de bootstrap:

```text
ResetExtensionRegistry
→ frozen
```

---

# 206. No runtime mutation of reset pipeline

No deberá cambiar arbitrariamente entre requests.

---

# 207. Reset extension descriptor

Podrá declarar:

```text
extension ID
dirty state category
reset step
dependencies
ordering
criticality
capability requirements
verification support
```

---

# 208. Extension ordering

Se resolverá determinísticamente mediante:

```text
dependency graph
```

no por orden accidental de Composer.

---

# 209. Circular reset dependency

Deberá fallar durante bootstrap.

---

# 210. Reset state machine

Modelo conceptual:

```text
CLEAN
  │
  ▼
DIRTY
  │
  ▼
RESET_PLANNED
  │
  ▼
RESETTING
  │
  ├──► VERIFYING
  │       │
  │       ├──► CLEAN
  │       └──► UNSAFE
  │
  ├──► CLEAN
  │
  └──► UNSAFE
          │
          ▼
       DISCARD
```

---

# 211. No DIRTY → IDLE

Una conexión dirty no podrá saltar directamente a idle.

---

# 212. No UNSAFE → IDLE

Sin recuperación explícita.

---

# 213. Tainted state

`TAINTED` significa:

> El framework no puede demostrar que la conexión mantenga un estado reutilizable seguro.

---

# 214. Tainted is not necessarily broken

La conexión puede responder correctamente a queries.

Aun así:

```text
do not reuse
```

---

# 215. Broken state

`BROKEN` indica:

```text
transport/protocol/native resource unusable
```

---

# 216. Dirty vs Tainted

```text
DIRTY
=
known reset required

TAINTED
=
safe reset cannot currently be guaranteed
```

---

# 217. Dirty can become clean

```text
DIRTY
→ reset
→ CLEAN
```

---

# 218. Tainted default

```text
TAINTED
→ discard
```

---

# 219. Recovery from tainted

Sólo si existe una operación de reset explícitamente garantizada por Driver/Platform.

---

# 220. No optimistic untaint

Nunca:

```text
ping succeeded
→ untaint
```

---

# 221. Reset and connection age

Incluso una conexión perfectamente limpia puede retirarse por:

```text
max lifetime
max uses
credential generation
configuration generation
maintenance
```

---

# 222. Clean ≠ reusable forever

Por tanto:

```text
Clean
+
RetirementPolicy
=
ReuseDecision
```

---

# 223. Reset and pool retirement

```text
reset success
     │
     ▼
retirement policy
     │
     ├── keep → IDLE
     └── retire → CLOSED
```

---

# 224. State and connection role

Una conexión física podrá estar asociada a:

```text
PRIMARY
REPLICA
```

pero ese role de topology es distinto de:

```text
SQL session role
```

---

# 225. Naming rule

Evitar ambigüedad usando términos como:

```text
ConnectionTopologyRole
SessionAuthorizationRole
```

---

# 226. Read/write routing

State reset no decidirá:

```text
primary vs replica
```

---

# 227. Replica session reset

Las mismas garantías de aislamiento aplican a replicas.

---

# 228. Replica failure

Una conexión replica dañada se descarta; el routing/failover system decide qué hacer después.

---

# 229. Reset and read-only state

Algunas plataformas permiten session-level:

```text
read only
read write
```

Ese estado deberá ser rastreable cuando sea relevante.

---

# 230. Reset and isolation level

Una transaction puede cambiar:

```text
isolation level
```

por transaction o por session.

El Platform deberá indicar su alcance.

---

# 231. Scope-aware state semantics

El mismo comando puede tener semántica distinta:

```text
transaction-scoped
session-scoped
connection-scoped
```

según plataforma.

---

# 232. PlatformStateSemantics

Podrá existir metadata estructurada para describir estas diferencias.

---

# 233. State mutation descriptor

Conceptualmente:

```text
Mutation:
    category: ISOLATION
    scope: SESSION
    resetStrategy: EXPLICIT
    securityCritical: false
```

---

# 234. Security mutation descriptor

Ejemplo:

```text
Mutation:
    category: AUTHORIZATION_ROLE
    scope: SESSION
    resetStrategy: EXPLICIT
    securityCritical: true
```

---

# 235. Session mutation API

Podrá existir una capa interna:

```text
ConnectionSessionController
```

para operaciones de sesión soportadas.

---

# 236. Why SessionController?

En vez de dispersar:

```sql
SET ROLE
SET TIME ZONE
SET search_path
```

por todo el framework.

---

# 237. SessionController responsibilities

```text
apply mutation
record mutation
use platform compiler
preserve baseline metadata
```

---

# 238. SessionController does not own lease

Trabaja sobre un recurso ya adquirido.

---

# 239. Session operations and Compiler

Cuando una operación requiere SQL:

```text
SessionOperation
→ Platform Session Compiler
→ native execution
```

---

# 240. Avoid raw vendor SQL in core

No:

```php
if ($postgres) {
    $sql = 'RESET ROLE';
}
```

en `ConnectionStateTracker`.

---

# 241. Platform SessionResetCompiler

Podrá existir:

```text
Platform
└── Session
    ├── SessionOperationCompiler
    └── SessionResetCompiler
```

---

# 242. Driver native operations

Cuando exista primitive nativa:

```text
DriverResetter
```

podrá evitar SQL.

---

# 243. Strategy selection priority

Ejemplo conceptual:

```text
1. native reset with sufficient guarantees
2. targeted platform reset
3. strict platform reset
4. discard
```

No necesariamente será el orden universal.

---

# 244. Cost model

Reset strategy podrá considerar:

```text
round trips
number of statements
reconnect cost
authentication cost
TLS cost
server load
```

pero sólo después de satisfacer seguridad.

---

# 245. Reset vs reconnect cost

A veces:

```text
close + reconnect
```

puede ser más barato o seguro que decenas de resets.

---

# 246. Policy decision

VoltStack podrá decidir:

```text
dirty complexity threshold exceeded
→ discard/reconnect
```

---

# 247. Reset complexity score

Opcionalmente podrá existir:

```text
ResetCostEstimate
```

---

# 248. Do not overengineer first implementation

La V1 podrá comenzar con:

```text
clean fast path
targeted common reset
strict fallback
discard
```

---

# 249. Recommended V1 scope

Inicialmente cubrir:

```text
transactions
open cursors
active statements
role
schema/search_path
database/catalog where applicable
timezone
isolation
autocommit
known session variables
native mutable access
```

---

# 250. V2 capabilities

Posteriormente:

```text
temporary object tracking
advisory lock tracking
compiled reset plans
advanced verification
statement cache integration
reset cost optimization
```

---

# 251. Testing architecture

Este subsystem requerirá tests específicos de:

```text
state tracking
reset planning
reset execution
verification
platform conformance
persistent-runtime isolation
```

---

# 252. Tracker unit tests

Ejemplo:

```text
initial state
→ CLEAN

RoleChanged
→ ROLE_DIRTY

ResetRole success
→ role clean
```

---

# 253. Dirty mask tests

Cada mutación deberá activar únicamente categorías esperadas.

---

# 254. Unknown state test

```text
unsafe native access
→ DIRTY_UNKNOWN
```

---

# 255. Reset planner test

Input:

```text
ROLE_DIRTY
TIMEZONE_DIRTY
```

Output:

```text
TARGETED
ResetRole
ResetTimezone
```

---

# 256. Strict reset planner test

Input:

```text
NATIVE_STATE_UNKNOWN
```

deberá producir:

```text
STRICT
```

o:

```text
DISCARD
```

según capabilities/policy.

---

# 257. Security failure test

```text
role reset fails
```

assert:

```text
connection not reusable
```

---

# 258. Transaction cleanup test

```text
BEGIN
release lease
```

deberá producir rollback antes de reuse.

---

# 259. Aborted transaction test

Simular:

```text
transaction aborted
```

y comprobar recuperación específica de plataforma.

---

# 260. Cursor cleanup test

```text
open streaming cursor
→ reset
```

deberá cerrar cursor antes de devolver recurso.

---

# 261. Cursor failure test

Si cierre deja estado incierto:

```text
discard
```

---

# 262. Session variable test

```text
baseline timezone = UTC
change timezone
release
reacquire
```

assert:

```text
timezone = UTC
```

---

# 263. Role isolation test

```text
Request A:
SET ROLE privileged

Request B:
same physical resource

assert:
privileged role absent
```

---

# 264. Tenant isolation test

```text
Tenant A:
search_path tenant_a

reset

Tenant B:
acquire same physical connection

assert:
no tenant_a state
```

---

# 265. Native access test

```text
unsafeNative()
```

sin declaración de efectos deberá activar política conservadora.

---

# 266. Native reset capability test

Driver que declare native reset deberá demostrar mediante conformance tests qué estado limpia.

---

# 267. False capability test

Un driver no podrá declarar:

```text
clearsTemporaryObjects = true
```

si sus tests demuestran lo contrario.

---

# 268. Configuration generation test

Cambiar SessionProfile deberá provocar:

```text
old resource reset/reinitialize/retire
```

según política.

---

# 269. Persistent request test

```text
for i in 1..10000:
    begin scope
    mutate connection state
    end scope
```

Comprobar:

```text
no cross-scope contamination
```

---

# 270. FrankenPHP test

Mismo worker:

```text
Request A
Request B
Request C
```

reutilizando conexiones físicas.

Cada request deberá observar baseline correcto.

---

# 271. Queue worker test

```text
Job A mutates session
Job A throws

Job B executes
```

Job B deberá estar aislado.

---

# 272. OpenSwoole concurrency test

```text
Coroutine A
Coroutine B
```

no deberán mezclar:

```text
leases
state trackers
session context
```

---

# 273. Reset timeout test

Simular reset bloqueado.

Resultado:

```text
connection discarded
worker continues
```

---

# 274. Double reset test

Reset deberá tener semántica controlada/idempotente cuando sea posible.

---

# 275. Reset-after-close test

No deberá intentar operaciones nativas sobre recurso cerrado.

---

# 276. State architecture tests

Prohibir dependencias:

```text
ConnectionState → ORM
ConnectionState → QueryBuilder
ConnectionState → Controller
ConnectionState → HTTP Request
ConnectionState → Multitenancy implementation
```

---

# 277. Vendor-condition tests

Detectar patrones prohibidos en core:

```php
if ($driver === 'pgsql')
if ($driver === 'mysql')
if ($driver === 'sqlite')
```

---

# 278. Capability-driven tests

Platform-specific behavior deberá resolverse mediante:

```text
Platform
Dialect
Driver
Capabilities
```

---

# 279. Suggested namespaces

```text
VoltStack\Quantum\Database\Connection\State
│
├── Contract
│   ├── ConnectionStateTrackerInterface.php
│   ├── ConnectionStateMutationRecorderInterface.php
│   ├── ConnectionResetPlannerInterface.php
│   ├── ConnectionResetExecutorInterface.php
│   ├── ConnectionResetVerifierInterface.php
│   └── ConnectionResetStrategyInterface.php
│
├── Model
│   ├── ConnectionStateSnapshot.php
│   ├── ConnectionDirtyMask.php
│   ├── ConnectionStateMutation.php
│   ├── ConnectionSessionProfile.php
│   ├── CompiledConnectionSessionProfile.php
│   ├── SessionProfileFingerprint.php
│   └── ConnectionStateFingerprint.php
│
├── Tracking
│   ├── ConnectionStateTracker.php
│   ├── ConnectionStateMutationRecorder.php
│   ├── ConnectionStateJournal.php
│   └── ConnectionStateKnowledge.php
│
├── Session
│   ├── ConnectionSessionController.php
│   ├── SessionState.php
│   ├── SessionVariableRegistry.php
│   ├── SessionMutation.php
│   └── SessionStateSemantics.php
│
├── Reset
│   ├── ConnectionResetPlanner.php
│   ├── ConnectionResetPlan.php
│   ├── ConnectionResetExecutor.php
│   ├── ConnectionResetContext.php
│   ├── ConnectionResetResult.php
│   ├── ConnectionResetMode.php
│   ├── ConnectionResetStatus.php
│   └── ConnectionReuseDecision.php
│
├── Reset/Step
│   ├── CloseCursorResetStep.php
│   ├── CloseStatementResetStep.php
│   ├── RollbackTransactionResetStep.php
│   ├── ReleaseAdvisoryLockResetStep.php
│   ├── TemporaryObjectResetStep.php
│   ├── RoleResetStep.php
│   ├── SchemaResetStep.php
│   ├── TimezoneResetStep.php
│   ├── IsolationResetStep.php
│   ├── SessionVariableResetStep.php
│   └── NativeConnectionResetStep.php
│
├── Verification
│   ├── ConnectionResetVerifier.php
│   ├── ConnectionResetVerificationLevel.php
│   └── ConnectionResetVerificationResult.php
│
├── Capability
│   ├── ConnectionResetCapability.php
│   ├── SessionStateCapability.php
│   └── NativeResetCapability.php
│
├── Extension
│   ├── ConnectionStateExtensionInterface.php
│   ├── ConnectionResetExtensionInterface.php
│   ├── ConnectionResetExtensionDescriptor.php
│   └── ConnectionResetExtensionRegistry.php
│
├── Diagnostics
│   ├── ConnectionStateDiagnostics.php
│   ├── ConnectionResetDiagnostics.php
│   └── ConnectionResetPlanFormatter.php
│
└── Exception
    ├── ConnectionStateException.php
    ├── ConnectionStateUnknownException.php
    ├── ConnectionResetException.php
    ├── ConnectionResetPlanningException.php
    ├── ConnectionResetExecutionException.php
    ├── ConnectionResetVerificationException.php
    └── UnsafeConnectionStateException.php
```

---

# 280. Dependency model

```text
Connection Lifecycle
        │
        ▼
Connection State/Reset
        │
        ├────────► Driver primitives
        │
        ├────────► Platform semantics
        │
        └────────► Capability model
```

Nunca:

```text
Driver
→ Connection State System
```

como dependencia inversa de alto nivel.

---

# 281. Detailed dependency flow

```text
Lease Release
     │
     ▼
ConnectionLifecycle
     │
     ▼
ConnectionStateTracker
     │
     ▼
ConnectionResetPlanner
     │
     ├── Platform
     ├── Capabilities
     └── ResetPolicy
     │
     ▼
ConnectionResetPlan
     │
     ▼
ConnectionResetExecutor
     │
     ├── Driver primitives
     └── Platform operations
     │
     ▼
ConnectionResetVerifier
     │
     ▼
ReuseDecision
     │
     ├── REUSE ───► Pool
     └── DISCARD ─► Close
```

---

# 282. Public API exposure

La mayoría de este subsystem deberá ser:

```text
INTERNAL
```

o:

```text
EXTENSION
```

---

# 283. Public developer API

El desarrollador ordinario no debería necesitar escribir:

```php
$connection->resetState();
```

---

# 284. Lifecycle-owned reset

Reset será normalmente automático:

```text
lease release
→ lifecycle
→ reset
```

---

# 285. Administrative API

Podrá existir una API operacional:

```text
DB::connections()->retire(...)
DB::connections()->drain(...)
```

pero no deberá saltarse invariantes.

---

# 286. Diagnostic API

Podrá exponerse:

```text
DB::diagnostics()->connectionState(...)
```

con información segura.

---

# 287. Manual reset API

Si se ofrece:

```text
Connection::reset()
```

deberá ser una operación avanzada y controlada.

No un sustituto de lifecycle automático.

---

# 288. Architectural invariants

## DB-CONN-STATE-001

Toda conexión física reusable tendrá un baseline definido.

## DB-CONN-STATE-002

Health y session state serán conceptos independientes.

## DB-CONN-STATE-003

Una conexión viva no será considerada automáticamente limpia.

## DB-CONN-STATE-004

Una conexión dirty no regresará directamente a idle.

## DB-CONN-STATE-005

Toda mutación conocida deberá ser rastreable cuando pase por APIs controladas.

## DB-CONN-STATE-006

Estado desconocido nunca equivaldrá a estado limpio.

## DB-CONN-STATE-007

Mutable native access no rastreado activará una política conservadora.

## DB-CONN-STATE-008

Un reset fallido impedirá reuse cuando afecte seguridad o consistencia.

## DB-CONN-STATE-009

Una transaction activa deberá resolverse antes de reuse.

## DB-CONN-STATE-010

Una transaction con outcome incierto no será tratada como limpia.

## DB-CONN-STATE-011

Cursores dependientes de la conexión deberán cerrarse antes de reuse.

## DB-CONN-STATE-012

Statements activos deberán cerrarse o resetearse antes de reuse.

## DB-CONN-STATE-013

Estado de autorización deberá restaurarse antes de reuse.

## DB-CONN-STATE-014

Estado de tenant deberá eliminarse antes de reutilizar una conexión.

## DB-CONN-STATE-015

El Pool sólo recibirá recursos aprobados por el reset system.

## DB-CONN-STATE-016

El Pool no implementará semántica vendor-specific de reset.

## DB-CONN-STATE-017

Driver expondrá primitivas; Platform describirá semántica.

## DB-CONN-STATE-018

El core no utilizará driver-name conditionals para reset.

## DB-CONN-STATE-019

Las capabilities de reset podrán ser estructuradas.

## DB-CONN-STATE-020

Una capability de native reset deberá especificar qué estado realmente limpia.

## DB-CONN-STATE-021

Reset Planner no ejecutará operaciones nativas.

## DB-CONN-STATE-022

Reset Executor no decidirá por sí mismo política de reutilización global.

## DB-CONN-STATE-023

Reset Verifier será independiente de planificación.

## DB-CONN-STATE-024

Reset será determinista para iguales entradas.

## DB-CONN-STATE-025

El fast path evitará round trips innecesarios.

## DB-CONN-STATE-026

Optimización nunca invalidará aislamiento.

## DB-CONN-STATE-027

Una conexión tainted se descartará por defecto.

## DB-CONN-STATE-028

Un ping exitoso no eliminará estado tainted.

## DB-CONN-STATE-029

Una conexión closed no podrá resetearse.

## DB-CONN-STATE-030

El state tracker pertenecerá al recurso físico.

## DB-CONN-STATE-031

Contexto de request no sobrevivirá dentro del tracker después del reset.

## DB-CONN-STATE-032

Credenciales no formarán parte de snapshots de estado.

## DB-CONN-STATE-033

Reset telemetry no expondrá secretos.

## DB-CONN-STATE-034

Reset crítico continuará durante cleanup aunque el request haya sido cancelado.

## DB-CONN-STATE-035

Un reset timeout provocará descarte seguro.

## DB-CONN-STATE-036

Las extensiones que introduzcan session state deberán proporcionar semántica de reset.

## DB-CONN-STATE-037

La corrección del reset no dependerá de listeners observacionales.

## DB-CONN-STATE-038

FrankenPHP podrá reutilizar conexiones físicas sólo después de reset seguro.

## DB-CONN-STATE-039

RoadRunner utilizará el mismo modelo.

## DB-CONN-STATE-040

OpenSwoole deberá preservar aislamiento entre coroutines.

## DB-CONN-STATE-041

El State/Reset System será independiente de ORM.

## DB-CONN-STATE-042

El State/Reset System será independiente de HTTP.

## DB-CONN-STATE-043

El State/Reset System no decidirá query retry.

## DB-CONN-STATE-044

El State/Reset System no decidirá failover.

## DB-CONN-STATE-045

Reset y retirement serán decisiones relacionadas pero distintas.

## DB-CONN-STATE-046

Una conexión limpia todavía podrá retirarse por lifecycle policy.

## DB-CONN-STATE-047

SessionProfile será inmutable después de compilación.

## DB-CONN-STATE-048

Cambios de SessionProfile deberán detectarse mediante generation/fingerprint.

## DB-CONN-STATE-049

El native handle no deberá escapar del lifetime de su lease.

## DB-CONN-STATE-050

La seguridad del siguiente execution scope tendrá prioridad sobre conservar una conexión física.

---

# 289. Anti-pattern — Ping as reset

```sql
SELECT 1;
```

seguido de:

```text
connection = clean
```

**Prohibido.**

---

# 290. Anti-pattern — Assume rollback cleans everything

```text
ROLLBACK
→ connection completely reset
```

**Incorrecto.**

Rollback puede no limpiar:

```text
role
timezone
search_path
session variables
temporary objects
advisory locks
```

---

# 291. Anti-pattern — Always reconnect

```text
every request
→ close
→ reconnect
```

sería seguro en ciertos aspectos, pero eliminaría gran parte del beneficio del pooling.

VoltStack debe poder reutilizar conexiones cuando sea demostrablemente seguro.

---

# 292. Anti-pattern — Always run full reset

```text
every lease release
→ 10 reset queries
```

también es indeseable.

El tracking permite un fast path.

---

# 293. Anti-pattern — Trust raw SQL

```text
raw SQL
→ assume no session mutation
```

sin política explícita.

---

# 294. Anti-pattern — Inspect every SQL string

No deberá construirse una arquitectura dependiente de:

```text
regex over SQL
```

para determinar todo el estado.

---

# 295. Anti-pattern — Driver name switches

```php
switch ($driver) {
    case 'mysql':
    case 'pgsql':
}
```

en el core.

---

# 296. Anti-pattern — ORM resets connection

```text
EntityManager
→ reset PDO session
```

**Prohibido.**

---

# 297. Anti-pattern — Pool knows SQL

```text
ConnectionPool
→ execute RESET ROLE
```

**Prohibido.**

---

# 298. Anti-pattern — State tracker owns native resource

El tracker observa/representa estado.

No deberá convertirse en owner del native handle.

---

# 299. Anti-pattern — Unknown means probably safe

```text
unknown state
→ reuse anyway
```

**Prohibido.**

---

# 300. Anti-pattern — Security reset warning

```text
RESET ROLE failed
→ log warning
→ return connection to pool
```

**Prohibido.**

---

# 301. Anti-pattern — Tenant overwrite

```text
Tenant A
→ directly overwrite session to Tenant B
```

sin baseline/reset semantics.

---

# 302. Anti-pattern — Global dirty state

```php
static bool $dirty;
```

**Prohibido.**

Cada recurso físico mantiene su propio estado.

---

# 303. Anti-pattern — State stored in logical connection

Una logical connection puede utilizar distintos recursos físicos.

Por tanto:

```text
Physical session state
```

no pertenece a ella.

---

# 304. Anti-pattern — Hidden reset

Un Driver no deberá ejecutar resets implícitos no documentados que cambien semántica sin informar capabilities.

---

# 305. Anti-pattern — Optional cleanup for persistent workers

Connection reset no será una optimización opcional bajo pooling persistente.

Es parte de correctness.

---

# 306. Flujo de caso limpio

```text
Request A
   │
   ▼
Acquire P1
   │
   ▼
SELECT
   │
   ▼
Materialize Result
   │
   ▼
Release
   │
   ▼
DirtyMask == CLEAN
   │
   ▼
FAST
   │
   ▼
P1 IDLE
```

---

# 307. Flujo con transaction

```text
Request A
   │
   ▼
Acquire P1
   │
   ▼
BEGIN
   │
   ▼
TRANSACTION_DIRTY
   │
   ▼
Request fails
   │
   ▼
Release/Cleanup
   │
   ▼
ROLLBACK
   │
   ▼
Verify transaction NONE
   │
   ▼
P1 IDLE
```

---

# 308. Flujo con tenant

```text
Tenant A
   │
   ▼
Acquire P1
   │
   ▼
SET search_path = tenant_a
   │
   ▼
SCHEMA_DIRTY
   │
   ▼
Release
   │
   ▼
Reset search_path
   │
   ▼
Baseline
   │
   ▼
P1 IDLE
   │
   ▼
Tenant B may acquire P1 safely
```

---

# 309. Flujo con native access

```text
Request A
   │
   ▼
Acquire P1
   │
   ▼
unsafeNative()
   │
   ▼
MUTABLE_UNTRACKED
   │
   ▼
DIRTY_UNKNOWN
   │
   ▼
Strict reset available?
   │
   ├── yes → strict reset → verify → reuse
   │
   └── no  → discard P1
```

---

# 310. Flujo con reset fallido

```text
P1
 │
 ▼
ROLE_DIRTY
 │
 ▼
ResetRole
 │
 ▼
FAIL
 │
 ▼
TAINTED
 │
 ▼
RETIRE
 │
 ▼
CLOSE
```

Nunca:

```text
FAIL
→ IDLE
```

---

# 311. Arquitectura final

```text
                 PHYSICAL CONNECTION
                         │
                         ▼
                ConnectionStateTracker
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
     Transaction      Session        Resources
       State           State       Cursor/Statement
          │              │               │
          └──────────────┼───────────────┘
                         ▼
                ConnectionStateSnapshot
                         │
                         ▼
                  ResetPolicy
                         │
                         ▼
                ConnectionResetPlanner
                         │
                         ▼
                 ConnectionResetPlan
                         │
                         ▼
                ConnectionResetExecutor
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
      Core            Driver          Platform
   Reset Steps      Primitives       Semantics
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                ConnectionResetVerifier
                         │
               ┌─────────┴─────────┐
               ▼                   ▼
         VERIFIED CLEAN          UNSAFE
               │                   │
               ▼                   ▼
       Retirement Policy        TAINTED
               │                   │
        ┌──────┴──────┐            ▼
        ▼             ▼          DISCARD
      REUSE          RETIRE         │
        │             │             ▼
        ▼             └──────────► CLOSE
      IDLE
        │
        ▼
      POOL
```

---

# 312. Ecuación de seguridad

```text
Safe Physical Connection Reuse
=
Known Baseline
+
Tracked Mutations
+
Deterministic Reset
+
Required Verification
+
Conservative Unknown-State Policy
```

---

# 313. Ecuación de rendimiento

```text
Efficient Reset
=
State Tracking
+
Fast Clean Path
+
Targeted Reset
+
Compiled Strategies
-
Unnecessary Round Trips
```

---

# 314. Ecuación de aislamiento

```text
Execution Scope Isolation
=
No Active Transaction
+
No Open Cursor
+
No Request Session State
+
No Previous Tenant State
+
No Previous Security State
+
No Unknown Native State
```

---

# 315. Regla principal

La regla fundamental será:

> VoltStack nunca reutilizará una conexión física únicamente porque siga abierta; la reutilizará porque puede demostrar que su estado es seguro.

---

# 316. Decisión sobre rendimiento vs seguridad

Cuando exista conflicto entre:

```text
reuse physical connection
```

y:

```text
guarantee isolation
```

VoltStack elegirá:

```text
guarantee isolation
```

y descartará el recurso si es necesario.

---

# 317. Relación Pooling / Lifecycle / State

Los documentos `14`, `15` y `16` forman una unidad arquitectónica:

```text
14_DATABASE_CONNECTION_POOLING_SYSTEM
│
│  ¿Dónde se conservan y cómo se asignan
│  los recursos físicos reutilizables?
│
▼
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM
│
│  ¿Cuándo nacen, se adquieren, liberan,
│  retiran y destruyen?
│
▼
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM
   │
   │  ¿Qué estado contienen y cómo se
   │  garantiza que vuelvan limpios?
   │
   ▼
Safe Physical Connection Reuse
```

---

# 318. Resultado arquitectónico

Con este sistema VoltStack podrá aprovechar:

```text
persistent workers
+
connection pooling
+
physical connection reuse
```

sin aceptar:

```text
request state reuse
tenant contamination
authorization contamination
transaction leakage
cursor leakage
session leakage
```

Esto permite mantener una API de alto nivel sencilla:

```php
DB::table('users')->get();
```

mientras internamente:

```text
Logical Connection
        │
        ▼
Acquire Lease
        │
        ▼
Physical Connection
        │
        ▼
Execute
        │
        ▼
State Tracking
        │
        ▼
Release
        │
        ▼
Reset
        │
        ▼
Verify
        │
        ├── Pool
        └── Discard
```

---

# 319. Criterios de aceptación

El subsystem estará correctamente diseñado cuando pueda garantizar:

```text
clean physical connections have a defined baseline

health and state are independently modeled

known mutations are tracked

unknown mutations are treated conservatively

transactions cannot leak between scopes

cursors cannot leak between scopes

statements cannot retain unsafe active state

roles cannot leak between scopes

schemas/search paths cannot leak between tenants

session variables can be restored

native mutable access cannot silently bypass lifecycle safety

reset strategy is capability-driven

reset strategy is deterministic

reset failures never silently return resources to the pool

security reset failures force discard

fast clean path avoids unnecessary database round trips

strict reset exists for uncertain states

native reset can be used only with declared semantics

platform-specific behavior remains outside core

pooling does not implement reset SQL

ORM does not implement physical reset

FrankenPHP request reuse remains isolated

RoadRunner can use the same reset architecture

OpenSwoole concurrent scopes remain isolated

configuration generations can invalidate old baselines

credential/session changes can trigger retirement

reset operations are observable

reset operations are testable

unsafe resources are discarded rather than optimistically reused
```

---

# 320. Conclusión

`Connection State and Reset System` establece la frontera que hace posible combinar:

```text
high performance
+
persistent runtime
+
connection pooling
+
strict execution isolation
```

El principio no será:

```text
"return the connection to the pool"
```

sino:

```text
"prove the connection is safe,
then return it to the pool"
```

El flujo definitivo será:

```text
Acquire
   │
   ▼
Use
   │
   ▼
Track Mutations
   │
   ▼
Release Requested
   │
   ▼
Snapshot State
   │
   ▼
Plan Reset
   │
   ▼
Execute Reset
   │
   ▼
Verify
   │
   ├── SAFE
   │     │
   │     ▼
   │   REUSE
   │     │
   │     ▼
   │    POOL
   │
   └── UNSAFE
         │
         ▼
       DISCARD
```

Esta arquitectura será una pieza crítica para que FrankenPHP pueda ser tratado como runtime predeterminado de VoltStack desde el diseño inicial, sin heredar supuestos del modelo clásico de PHP donde el proceso termina después de cada request.

---

# 321. Siguiente documento

El siguiente documento será:

```text
17_DATABASE_DIALECT_SYSTEM.md
```

Este documento iniciará la separación formal de la semántica SQL respecto del Driver.

La regla será:

```text
Driver
=
How VoltStack communicates with the database

Dialect
=
How SQL is expressed for a database family

Platform
=
What the database engine means and supports
```

Por tanto:

```text
Driver ≠ Dialect ≠ Platform
```

`17_DATABASE_DIALECT_SYSTEM.md` deberá definir, entre otros:

```text
Dialect architecture
Dialect contracts
Dialect descriptors
Dialect registry
Dialect resolution
SQL lexical conventions
identifier quoting
parameter placeholder conventions
reserved words
SQL syntax families
expression syntax
function syntax
operator syntax
LIMIT/OFFSET syntax
RETURNING syntax
UPSERT syntax
CTE syntax
window syntax
locking syntax
DDL syntax boundaries
dialect capabilities
dialect versioning
dialect extensions
MySQL dialect
MariaDB dialect
PostgreSQL dialect
SQLite dialect
dialect/compiler relationship
dialect/platform separation
dialect/driver separation
portable semantic intent
vendor-specific syntax isolation
dialect conformance testing
```

La transición arquitectónica queda así:

```text
10 Driver Architecture
        │
        ▼
11–16 Connection Infrastructure
        │
        ▼
17 Dialect System
        │
        ▼
18 Platform Capability System
        │
        ▼
19–21 Concrete Platforms
```

Con `16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md` queda cerrado el bloque central de infraestructura de conexiones antes de entrar en la capa de dialectos y capacidades de plataforma.