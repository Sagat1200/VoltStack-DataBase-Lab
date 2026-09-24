# 173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md

# VoltStack Quantum Database
## Database Pessimistic Locking System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 173 — Database Pessimistic Locking System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md`  
**Siguiente documento:** `174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md`

---

# 1. Propósito

`Database Pessimistic Locking System` define cómo VoltStack representará, solicitará, validará, compilará y observará mecanismos explícitos de bloqueo proporcionados por los motores de base de datos.

Su objetivo es permitir que una operación declare:

> “Necesito adquirir una protección de concurrencia sobre estos recursos antes de continuar y conservarla dentro del boundary transaccional correspondiente”.

Modelo conceptual:

```text
Transaction A
    ↓
SELECT Resource
FOR UPDATE
    ↓
exclusive row lock acquired
    ↓
business operation
    ↓
UPDATE
    ↓
COMMIT
    ↓
lock released
```

Mientras tanto:

```text
Transaction B
    ↓
requests incompatible lock
    ↓
WAIT / FAIL / SKIP
```

dependiendo de la política solicitada y las capabilities de la plataforma.

La regla central será:

> **VoltStack modelará pessimistic locking como una intención semántica explícita asociada a una transacción y compilada según las capacidades reales de la plataforma; nunca asumirá que una sintaxis SQL concreta, un isolation level o una consulta de lectura proporcionan por sí solos las mismas garantías de locking en todos los motores.**

---

# 2. Objetivos

El sistema deberá proporcionar:

1. modelo canónico de lock intent;
2. shared locking;
3. exclusive locking;
4. update-intent locking cuando sea representable;
5. lock wait policies;
6. WAIT;
7. NOWAIT;
8. SKIP LOCKED;
9. timeout;
10. lock scope;
11. lock targets;
12. row locking;
13. relationship locking;
14. Query AST integration;
15. Query Builder integration;
16. ORM integration;
17. EntityManager lock API;
18. Platform capability negotiation;
19. SQL Compiler integration;
20. transaction validation;
21. connection affinity;
22. isolation interaction;
23. deadlock integration;
24. retry integration;
25. queue/worker coordination;
26. diagnostics;
27. telemetry;
28. resource governance;
29. persistent-runtime safety;
30. testing de conformidad.

---

# 3. No objetivos

El sistema no deberá:

- implementar el lock manager interno del DBMS;
- mantener una tabla global de row locks en PHP;
- sustituir transaction isolation;
- sustituir optimistic locking;
- sustituir constraints;
- implementar distributed locks;
- usar Redis locks como equivalente a row locks;
- garantizar ausencia de deadlocks;
- inferir garantías no documentadas por la plataforma;
- mantener locks después de finalizar la transacción;
- adquirir locks mediante SQL raw desde ORM;
- transformar cualquier `SELECT` en locking read;
- hacer locking automático de todo un aggregate.

---

# 4. Distinciones fundamentales

```text
Pessimistic Locking
≠
Optimistic Locking
≠
Transaction Isolation
≠
Deadlock Prevention
≠
Distributed Lock
≠
Advisory Lock
≠
Application Mutex
≠
Database Constraint
```

---

# 5. Pessimistic vs optimistic

Optimistic:

```text
read V7
   ↓
work
   ↓
UPDATE ... WHERE version = 7
   ↓
detect conflict at write time
```

Pessimistic:

```text
BEGIN
   ↓
acquire incompatible lock
   ↓
work while lock is held
   ↓
write
   ↓
COMMIT / ROLLBACK
```

---

# 6. Filosofía

Optimistic locking supone:

```text
conflicts are sufficiently uncommon
```

Pessimistic locking supone:

```text
conflict should be prevented/serialized
before protected work continues
```

Ninguno será universalmente superior.

---

# 7. Canonical Lock Intent

VoltStack deberá representar el lock solicitado antes de traducirlo a SQL.

```php
final readonly class LockIntent
{
    public function __construct(
        public LockMode $mode,
        public LockWaitPolicy $wait,
        public LockScope $scope,
        public LockTargetSet $targets,
    ) {}
}
```

---

# 8. LockMode

Modelo canónico inicial:

```php
enum LockMode
{
    case SHARED;
    case EXCLUSIVE;
}
```

Podrán añadirse modos semánticos especializados mediante capabilities/extensiones.

---

# 9. SHARED

Conceptualmente:

```text
multiple compatible readers
```

pueden mantener protección compartida, mientras ciertas escrituras incompatibles deberán esperar/fallar.

Pero:

```text
SHARED
```

no deberá definirse mediante una cadena SQL concreta.

---

# 10. EXCLUSIVE

Conceptualmente expresa:

```text
exclusive modification intent
```

sobre los recursos seleccionados.

Una traducción frecuente podrá ser equivalente a:

```sql
FOR UPDATE
```

pero el modelo canónico seguirá siendo:

```text
LockMode::EXCLUSIVE
```

---

# 11. SQL syntax ≠ semantic model

Nunca:

```php
$query->lock('FOR UPDATE');
```

como API principal.

Preferir:

```php
$query->lock(LockMode::EXCLUSIVE);
```

---

# 12. Escape hatch

Podrá existir raw locking como escape hatch avanzado:

```php
$query->rawLock(...);
```

pero:

- será explícito;
- no portable;
- no recibirá garantías semánticas completas;
- podrá deshabilitar optimizaciones.

---

# 13. Lock wait policy

```php
enum LockWaitMode
{
    case WAIT;
    case NO_WAIT;
    case SKIP_LOCKED;
    case TIMEOUT;
}
```

---

# 14. WAIT

Comportamiento conceptual:

```text
lock unavailable
    ↓
wait according to DB/transaction/session policy
```

---

# 15. NO_WAIT

Conceptualmente:

```text
lock unavailable
    ↓
fail immediately
```

---

# 16. SKIP_LOCKED

Conceptualmente:

```text
candidate row locked incompatibly
    ↓
exclude it from result
```

No significa:

```text
wait less
```

Significa que el conjunto observable puede cambiar.

---

# 17. TIMEOUT

Conceptualmente:

```text
wait up to bounded duration
```

si la plataforma puede expresar la política con garantías suficientes.

---

# 18. LockWaitPolicy

```php
final readonly class LockWaitPolicy
{
    private function __construct(
        public LockWaitMode $mode,
        public ?Duration $timeout,
    ) {}
}
```

---

# 19. Timeout semantics

Debe distinguirse:

```text
Query Timeout
Lock Wait Timeout
Transaction Timeout
Application Deadline
```

No son equivalentes.

---

# 20. Lock wait timeout

Mide cuánto tiempo una adquisición de lock puede esperar.

---

# 21. Query timeout

Puede limitar la ejecución completa de una query.

---

# 22. Transaction timeout

Limita el boundary transaccional.

---

# 23. Deadline

Puede representar el presupuesto temporal global de la operación.

---

# 24. Effective wait

La espera real deberá respetar, cuando sea posible:

```text
EffectiveLockWait
≤
RemainingOperationDeadline
```

sin fingir capabilities que el motor no tenga.

---

# 25. Lock scope

VoltStack necesita modelar qué pretende protegerse.

```php
enum LockScope
{
    case SELECTED_ROWS;
    case ROOT_ENTITY;
    case RELATIONSHIP;
    case QUERY_RESULT;
    case PLATFORM_DEFINED;
}
```

---

# 26. Selected rows

El scope más directo será:

```text
rows selected by the locking query
```

---

# 27. Query result ≠ exact physical lock set

Importante:

> El conjunto lógico de filas devuelto por una consulta no necesariamente describe exhaustivamente todos los locks físicos adquiridos por el motor.

Dependiendo de:

- índices;
- isolation;
- execution plan;
- range locking;
- predicate locking;
- MVCC;
- engine internals;

el DBMS puede adquirir protecciones adicionales o diferentes.

---

# 28. VoltStack does not fake physical lock graph

El framework podrá saber:

```text
requested logical target
```

pero no necesariamente:

```text
every physical lock held by server
```

---

# 29. LockTarget

```php
interface LockTarget
{
}
```

Implementaciones conceptuales:

```text
QueryResultLockTarget
EntityLockTarget
RelationshipLockTarget
TableAliasLockTarget
PlatformLockTarget
```

---

# 30. Lock target ≠ SQL table string

El Query AST deberá resolver targets contra símbolos/aliases semánticos.

---

# 31. Query AST integration

Una locking query deberá representarse como:

```text
SelectQuery
├── Projection
├── Source
├── Predicate
├── Ordering
├── Limit
└── LockClause
```

---

# 32. LockClause AST

```php
final readonly class LockClause
{
    public function __construct(
        public LockMode $mode,
        public LockWaitPolicy $waitPolicy,
        public LockTargetSet $targets,
    ) {}
}
```

---

# 33. AST remains vendor-neutral

No deberá contener directamente:

```text
FOR UPDATE NOWAIT
```

como representación canónica.

---

# 34. Query Builder API

Ejemplos conceptuales:

```php
$order = DB::table('orders')
    ->where('id', $id)
    ->lockForUpdate()
    ->first();
```

o API tipada:

```php
$query
    ->lock(
        mode: LockMode::EXCLUSIVE,
        wait: LockWaitPolicy::wait(),
    );
```

---

# 35. Shared API

Podrá existir ergonomía:

```php
$query->lockForShare();
```

que produzca:

```text
LockMode::SHARED
```

---

# 36. NOWAIT API

```php
$query->lockForUpdate(
    wait: LockWaitPolicy::noWait()
);
```

---

# 37. SKIP LOCKED API

```php
$query->lockForUpdate(
    wait: LockWaitPolicy::skipLocked()
);
```

---

# 38. Explicit semantics

La API ergonómica siempre deberá converger en:

```text
canonical LockIntent / LockClause
```

---

# 39. Transaction requirement

Por default, un pessimistic lock deberá requerir:

```text
ACTIVE transaction
```

cuando su utilidad depende de mantener el lock más allá del statement.

---

# 40. Why

Ejemplo incorrecto:

```text
SELECT ... FOR UPDATE
autocommit statement ends
lock released
application modifies object
UPDATE later
```

El lock ya no protege el intervalo de negocio esperado.

---

# 41. Safe default

Si se intenta:

```php
Order::query()
    ->lockForUpdate()
    ->find(10);
```

fuera de un boundary válido, VoltStack deberá por default lanzar:

```text
PessimisticLockRequiresTransactionException
```

---

# 42. No hidden transaction

VoltStack no abrirá automáticamente una transacción alrededor únicamente del SELECT si después devolverá la entidad al usuario.

Eso crearía una falsa garantía.

---

# 43. EntityManager API

Ejemplo:

```php
$entityManager->lock(
    $order,
    LockMode::EXCLUSIVE,
);
```

---

# 44. Entity lock flow

```text
Managed Entity
    ↓
EntityKey
    ↓
Entity Metadata
    ↓
Lock Query Model
    ↓
Query Engine
    ↓
Compiler
    ↓
Executor
    ↓
Database
```

---

# 45. EntityManager does not emit SQL

Se conserva:

```text
EntityManager
≠
SQL Compiler
```

---

# 46. Lock managed entity

La entidad deberá:

- tener identidad establecida;
- pertenecer al execution domain correcto;
- estar asociada al contexto apropiado;
- usar la conexión fijada por la transacción.

---

# 47. Detached entity

Por default:

```text
Detached Entity
+
EntityManager::lock()
```

será rechazado o requerirá una operación explícita por identidad.

---

# 48. Lock by identity

Podrá existir:

```php
$repository->find(
    $id,
    lock: LockMode::EXCLUSIVE,
);
```

---

# 49. Lock existing managed entity

Si la entidad ya fue leída sin lock:

```text
read entity
    ↓
later request lock
```

deberá emitirse una operación real de lock/re-read cuando sea necesario.

No basta con marcar:

```text
entity.locked = true
```

en memoria.

---

# 50. Lock acquisition confirmation

El framework solo considerará una solicitud de lock exitosa después de que:

```text
database operation
```

la haya confirmado suficientemente.

---

# 51. Local lock state

VoltStack podrá registrar:

```text
LockAcquisitionRecord
```

para diagnostics.

Pero:

```text
LockAcquisitionRecord
≠
authoritative database lock table
```

---

# 52. Lock lifetime

Conceptualmente:

```text
acquire
    ↓
ACTIVE TRANSACTION
    ↓
COMMIT / ROLLBACK
    ↓
released by database
```

---

# 53. Framework cannot unlock row arbitrarily

Los row locks transaccionales normalmente se gobiernan por el DBMS.

Por tanto no se diseñará una API universal:

```php
$entityManager->unlock($entity);
```

que finja liberar el lock manteniendo la misma transacción.

---

# 54. Savepoints

Regla conservadora:

```text
RollbackToSavepoint
≠
GuaranteedReleaseOfAllLocksAcquiredAfterSavepoint
```

a nivel portable.

---

# 55. Why

Las semánticas de lock/savepoint pueden variar.

Por tanto VoltStack no prometerá:

```text
savepoint rollback
→ all later locks gone
```

sin capability explícita.

---

# 56. Nested transactions

`REQUIRED`:

```text
inner and outer
share physical transaction
```

por tanto comparten el dominio físico de locks.

---

# 57. NESTED

Un nested savepoint:

```text
does not create independent lock lifetime
```

por default.

---

# 58. REQUIRES_NEW

Una nueva transacción física podrá adquirir su propio conjunto de locks.

Pero puede bloquearse incluso contra el outer suspendido si ambos intentan recursos incompatibles.

---

# 59. Self-contention hazard

Ejemplo:

```text
Outer TX:
    locks Order#1

Outer suspended

Inner REQUIRES_NEW:
    requests Order#1
```

El inner puede esperar al outer, mientras el outer no puede continuar hasta que termine el inner.

Resultado potencial:

```text
self-induced blocking / timeout
```

---

# 60. Diagnostic rule

VoltStack deberá poder advertir cuando:

```text
REQUIRES_NEW
+
known outer lock target
+
same logical target
```

forme un riesgo evidente.

No afirmará deadlock sin evidencia del DBMS.

---

# 61. Connection affinity

Todo lock perteneciente a una transacción deberá ejecutarse sobre:

```text
PinnedConnection(TransactionContext)
```

---

# 62. No replica locking

Una locking read no deberá enviarse a una read replica normal por default.

---

# 63. Writer routing

```text
PessimisticLockIntent
→
WRITE/AUTHORITATIVE connection intent
```

aunque la query sea sintácticamente `SELECT`.

---

# 64. Important distinction

```text
SQL SELECT
≠
Read-only routing
```

cuando contiene locking semantics.

---

# 65. Sticky routing

Después de locking/write activity, cualquier routing posterior deberá respetar el transaction context y la conexión fijada.

---

# 66. Isolation interaction

Pessimistic locking y isolation son capas relacionadas pero distintas.

```text
IsolationLevel
≠
ExplicitLockMode
```

---

# 67. Same syntax, different effects

La semántica efectiva de:

```text
locking read
```

puede depender de:

- isolation level;
- platform;
- index;
- predicate;
- transaction mode;
- query shape.

---

# 68. Isolation compatibility

El sistema deberá validar:

```text
RequestedLockIntent
+
EffectiveIsolation
+
PlatformCapabilities
```

antes de prometer una garantía.

---

# 69. Serializable

`SERIALIZABLE` no implica que una API:

```php
lockForUpdate()
```

sea inútil.

Puede expresar:

- intención;
- wait policy;
- queue claiming;
- explicit resource serialization.

---

# 70. Read-only transaction

Una transacción marcada:

```text
READ_ONLY
```

podrá ser incompatible con:

```text
EXCLUSIVE lock intent
```

dependiendo de la semántica efectiva.

VoltStack deberá validarlo.

---

# 71. Shared lock semantics

`SHARED` no deberá asumirse idéntico entre motores.

El platform adapter declarará:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
EMULATED
UNSUPPORTED
UNKNOWN
```

---

# 72. PessimisticLockCapability

```php
final readonly class PessimisticLockCapability
{
    public function __construct(
        public CapabilityStatus $shared,
        public CapabilityStatus $exclusive,
        public CapabilityStatus $nowait,
        public CapabilityStatus $skipLocked,
        public CapabilityStatus $timeout,
        public CapabilityStatus $targetedLocks,
    ) {}
}
```

---

# 73. Capability negotiation

Pipeline:

```text
LockIntent
    ↓
Platform Capability Resolver
    ↓
Compatibility Analysis
    ↓
Effective Lock Plan
    ↓
SQL Compiler
```

---

# 74. No silent weakening

Si el usuario solicita:

```text
EXCLUSIVE + NOWAIT
```

y la plataforma solo soporta:

```text
EXCLUSIVE + WAIT
```

VoltStack no deberá compilar silenciosamente el segundo.

---

# 75. Why

Eso cambia:

```text
blocking behavior
latency
failure behavior
business semantics
```

---

# 76. Compatibility result

```php
enum LockCompatibilityStatus
{
    case EXACT;
    case WITH_LIMITATIONS;
    case EMULATED;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 77. Emulation

La emulación solo será aceptable si preserva la garantía semántica solicitada.

No basta con:

```text
roughly similar behavior
```

---

# 78. SQL Compiler integration

Compiler recibe:

```text
validated LockClause
+
EffectiveLockPlan
```

y produce la sintaxis correspondiente.

---

# 79. Compiler responsibilities

El compiler podrá decidir:

- ubicación de cláusula;
- sintaxis dialectal;
- target aliases;
- wait modifiers;
- platform-specific fragments.

---

# 80. Compiler does not decide policy

No decidirá:

```text
should we use NOWAIT?
```

Eso ya estará resuelto.

---

# 81. PostgreSQL/MySQL/MariaDB/SQLite

Cada platform adapter tendrá sus propias capabilities y compiladores.

Nunca:

```text
all databases support FOR UPDATE identically
```

---

# 82. SQLite

Si una plataforma no ofrece un equivalente real a row-level locking intent:

```text
UNSUPPORTED
```

será preferible a una falsa traducción.

---

# 83. Platform version

Capabilities podrán depender de:

```text
server version
driver version
transaction mode
```

---

# 84. Lock target capabilities

Algunos motores permiten seleccionar targets específicos en joins; otros pueden no hacerlo.

Por tanto:

```text
LockTargetSet
```

también deberá negociarse.

---

# 85. Joined query

Ejemplo:

```text
Order
JOIN Customer
```

La aplicación puede querer bloquear:

```text
Order only
```

y no:

```text
Customer
```

---

# 86. Targeted locking

API conceptual:

```php
$query->lock(
    LockMode::EXCLUSIVE,
    targets: ['orders']
);
```

internamente resuelta contra aliases semánticos.

---

# 87. Unsupported targeted locking

Si la plataforma solo puede bloquear un scope más amplio:

```text
requested target
≠
effective target
```

No deberá ampliarse silenciosamente si el cambio puede afectar concurrencia significativamente.

---

# 88. Lock scope expansion

Podrá requerir:

```text
ALLOW_STRONGER_SCOPE
```

como policy explícita.

---

# 89. Stronger is not always harmless

Un lock más amplio puede:

- reducir concurrencia;
- producir nuevos deadlocks;
- aumentar latencia;
- afectar throughput.

Por tanto:

```text
stronger lock
≠
automatically acceptable
```

---

# 90. Range and predicate locking

VoltStack deberá reconocer que ciertas consultas:

```text
WHERE price BETWEEN 100 AND 200
```

pueden implicar mecanismos de:

```text
range locking
predicate locking
gap locking
```

dependiendo de plataforma/isolation/plan.

---

# 91. No universal RangeLock API initially

El core inicial no deberá prometer:

```php
$query->lockRange(...)
```

como abstracción portable si no puede garantizar equivalencia.

---

# 92. Diagnostic exposure

Sí podrá exponer:

```text
Platform may acquire range/predicate protections
```

como capability/diagnostic information.

---

# 93. Index interaction

Una locking query sin índice apropiado puede:

- examinar más filas;
- adquirir locks más amplios;
- mantener locks más tiempo;
- aumentar contention.

---

# 94. Performance diagnostics

El sistema podrá correlacionar:

```text
locking query
+
slow execution
+
large scanned set
```

y recomendar revisar indexación.

---

# 95. No optimizer override

Pessimistic Locking System no deberá cambiar arbitrariamente índices/query plans para “mejorar” locking.

---

# 96. Ordering

Cuando varias filas deban bloquearse:

```text
consistent acquisition order
```

reduce el riesgo de deadlocks.

---

# 97. Example

Preferir conceptualmente:

```text
lock Account#10
lock Account#20
```

en todas las operaciones.

No:

```text
Flow A: 10 → 20
Flow B: 20 → 10
```

---

# 98. Canonical lock order

Podrá existir:

```php
interface LockOrderingPolicy
{
    public function order(
        LockTargetCollection $targets,
    ): LockTargetCollection;
}
```

---

# 99. Ordering must preserve semantics

No se reordenarán operaciones cuando:

```text
order itself
```

sea semánticamente significativo.

---

# 100. Lock acquisition plan

Para múltiples targets:

```text
LockRequest
    ↓
LockOrderingPolicy
    ↓
LockAcquisitionPlan
```

---

# 101. LockAcquisitionPlan

```php
final readonly class LockAcquisitionPlan
{
    public function __construct(
        public array $steps,
        public LockOrderingGuarantee $ordering,
    ) {}
}
```

---

# 102. Single query preferred where possible

Si varios recursos pueden bloquearse correctamente mediante una sola query:

```text
single locking query
```

puede reducir round trips.

Pero el planner deberá preservar:

- identity semantics;
- deterministic ordering;
- result correctness.

---

# 103. LIMIT interaction

`LIMIT` + locking requiere especial cuidado.

Ejemplo:

```text
SELECT jobs
WHERE status = 'pending'
ORDER BY id
LIMIT 10
FOR UPDATE
SKIP LOCKED
```

es útil para workers concurrentes.

---

# 104. Queue claiming pattern

Conceptualmente:

```text
Worker A
    ↓
select available rows
exclusive lock
skip locked
    ↓
claim jobs
```

Worker B:

```text
same query
    ↓
skips A's locked jobs
```

---

# 105. SKIP LOCKED semantics

`SKIP_LOCKED` significa que el resultado:

```text
may omit otherwise matching rows
```

debido a concurrencia.

---

# 106. Result completeness

Por tanto:

```text
SKIP_LOCKED Result
≠
Complete Snapshot Of Matching Rows
```

---

# 107. ORM collection hazard

Una relación cargada con:

```text
SKIP_LOCKED
```

no deberá marcarse:

```text
COMPLETE
```

si filas elegibles pudieron omitirse por locks.

---

# 108. Coverage model

La infraestructura de Relationship Loading deberá poder marcar:

```text
PARTIAL / CONCURRENCY_FILTERED
```

o metadata equivalente.

---

# 109. Pagination interaction

Locking + pagination puede producir resultados variables bajo concurrencia.

No se prometerá:

```text
stable global pagination
```

sin garantías adicionales.

---

# 110. Cursor pagination

Cursor ordering puede ayudar a procesar work queues, pero:

```text
cursor
≠
lock
```

---

# 111. Job claiming

El Database subsystem podrá proporcionar primitivas para construir job claiming, pero:

```text
Database Pessimistic Locking
≠
VoltStack Queue System
```

---

# 112. Lock timeout failure

Cuando una espera excede el límite:

```text
LockTimeoutException
```

deberá utilizar la clasificación canónica del documento 171.

---

# 113. NOWAIT failure

Si lock no disponible:

```text
LockNotAvailableException
```

podrá representar el fallo canónico.

---

# 114. SKIP LOCKED no es excepción

Normalmente:

```text
locked row
→ skipped
```

no:

```text
throw LockNotAvailableException
```

---

# 115. Deadlock interaction

Pessimistic locking puede incrementar la probabilidad de deadlocks si los recursos se adquieren en órdenes incompatibles.

---

# 116. Deadlock classification

Si el DBMS detecta un ciclo:

```text
Pessimistic Locking System
    ↓
does not retry
```

El error pasa a:

```text
Deadlock Handling System
```

---

# 117. Retry interaction

Un lock timeout o deadlock puede ser candidato a retry.

Pero:

```text
Lock Acquisition
≠
Retry Decision
```

---

# 118. NOWAIT and retry

Un `NOWAIT` failure puede utilizarse deliberadamente para:

```text
fail fast
```

Por tanto un automatic retry inmediato podría contradecir la intención.

---

# 119. Retry policy must respect wait intent

Ejemplo:

```text
NO_WAIT
+
LockNotAvailable
```

no deberá convertirse automáticamente en:

```text
wait via repeated retries forever
```

---

# 120. SKIP LOCKED vs retry

Si se eligió:

```text
SKIP_LOCKED
```

la política normalmente debe continuar con otros recursos, no reintentar inmediatamente el recurso omitido.

---

# 121. Lock timeout retry

Puede ser retryable según:

- boundary;
- deadline;
- idempotency;
- retry budget;
- business semantics.

---

# 122. Transaction Retry System remains authority

```text
Lock Failure
    ↓
Canonical Classification
    ↓
Retry Recommendation
    ↓
Transaction Retry System
```

---

# 123. Lock acquisition state

Podrá modelarse:

```php
enum LockAcquisitionStatus
{
    case REQUESTED;
    case ACQUIRED;
    case SKIPPED;
    case NOT_AVAILABLE;
    case TIMED_OUT;
    case DEADLOCKED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 124. UNKNOWN

Si la conexión falla mientras se intenta adquirir lock:

```text
lock acquisition outcome
```

puede ser desconocido.

---

# 125. But connection loss changes usefulness

Aunque el DB pudiera haber adquirido el lock momentáneamente, si la conexión desaparece:

```text
framework no longer controls that transaction reliably
```

El Transaction Context deberá manejar el fallo.

---

# 126. Lock ownership

VoltStack no deberá transferir un lock entre:

```text
connections
transactions
requests
coroutines
```

---

# 127. Transaction identity

Un lock acquisition record deberá asociarse a:

```text
TransactionId
ConnectionLeaseId
ExecutionDomain
```

---

# 128. Connection mismatch

Si se intenta usar un lock handle/record bajo otra conexión:

```text
PessimisticLockConnectionMismatchException
```

---

# 129. Lock handle

Si se expone:

```php
$lock = $entityManager->lock(...);
```

ese objeto será principalmente:

```text
diagnostic/scoped token
```

no una autoridad independiente capaz de liberar el DB lock.

---

# 130. Stale handles

Al finalizar transaction:

```text
LockHandle
→ TERMINAL
```

---

# 131. Use after completion

Debe fallar:

```text
LockHandle belongs to completed transaction
```

---

# 132. LockRegistry

Podrá existir un registry transaccional:

```text
TransactionLockRegistry
```

para:

- diagnostics;
- duplicate acquisition optimization;
- known ordering;
- lifecycle checks.

---

# 133. Registry ≠ lock manager

Regla:

```text
TransactionLockRegistry
≠
Database Lock Manager
```

---

# 134. Duplicate lock acquisition

Si VoltStack sabe que:

```text
same transaction
already requested equal/stronger lock
same exact target
```

podrá evitar una query redundante únicamente cuando sea semánticamente demostrable.

---

# 135. Lock strength lattice

Conceptualmente:

```text
NONE
  ↓
SHARED
  ↓
EXCLUSIVE
```

para ciertos targets.

Pero esto es una abstracción lógica limitada.

---

# 136. Lock upgrade

```text
SHARED
→
EXCLUSIVE
```

puede requerir una operación DB y puede bloquear/deadlock.

Nunca se actualizará solo el registry local.

---

# 137. Lock downgrade

```text
EXCLUSIVE
→
SHARED
```

no será una operación portable general.

No se ofrecerá como garantía core inicial.

---

# 138. Already exclusive

Si una transaction ya tiene lock exclusivo conocido sobre un target exacto, pedir shared podrá satisfacerse localmente solo si la plataforma garantiza que la información local es suficiente.

Default conservador:

```text
avoid over-optimizing unknown physical locks
```

---

# 139. Relationship locking

Ejemplo:

```text
Order
    └── Items
```

`lock(Order)` no implica:

```text
all current/future items locked
```

---

# 140. Explicit relation lock

Podrá existir:

```php
$entityManager->lockRelationship(
    $order,
    'items',
    LockMode::EXCLUSIVE,
);
```

que construya una locking query específica.

---

# 141. Empty relationship problem

Si actualmente no existen children:

```text
lock existing rows
```

no necesariamente impide:

```text
concurrent insert of new child
```

---

# 142. Phantom protection

Evitar nuevos rows que satisfagan un predicate puede requerir:

- isolation guarantees;
- range locks;
- predicate locks;
- parent row coordination;
- explicit domain strategy.

Por tanto:

```text
Lock Current Relationship Rows
≠
Prevent Future Relationship Inserts
```

---

# 143. Aggregate lock strategy

Para ciertos aggregates puede preferirse:

```text
lock aggregate root row
```

como punto de serialización de cambios.

Esto será una policy de dominio/persistencia, no comportamiento universal.

---

# 144. Polymorphic relationships

Locking de una polymorphic relation deberá resolver:

```text
type
+
identity
+
execution domain
```

antes de construir targets.

---

# 145. Cross-database relation

No se ofrecerá un lock transaccional único sobre:

```text
DB-A resource
+
DB-B resource
```

sin un sistema distribuido explícito.

---

# 146. Sharding

Un transaction-local pessimistic lock estará asociado al shard de la conexión.

```text
LockOnShardA
≠
LockOnShardB
```

---

# 147. Multitenancy

Lock target deberá preservar:

```text
TenantExecutionDomain
```

cuando exista integración multitenant.

---

# 148. No tenant bypass

Un lock query no podrá eliminar filtros de tenant para “encontrar” la fila.

---

# 149. Security

Pessimistic locking no concede permisos.

Debe cumplirse:

```text
Authorized Operation
+
Valid Lock Intent
```

---

# 150. Lock denial

Authorization deberá ocurrir antes de adquirir locks costosos siempre que sea posible.

---

# 151. User-controlled lock parameters

No se aceptarán directamente strings del usuario como:

```text
lock clause
lock target SQL
timeout expression
```

---

# 152. Resource governance

Locks consumen recursos compartidos.

VoltStack deberá permitir políticas sobre:

- máximo tiempo transaccional;
- máximo lock wait;
- máximo número lógico de targets;
- batch size;
- lock query complexity;
- retry count.

---

# 153. Long-held locks

Diagnostics deberán detectar:

```text
lock acquired
+
transaction still active
+
duration exceeds threshold
```

cuando la información local lo permita.

---

# 154. Local duration ≠ server lock duration

El framework podrá medir:

```text
time since successful lock request
```

pero no necesariamente el tiempo físico exacto de cada lock interno.

---

# 155. External I/O warning

Patrón:

```text
lock
↓
HTTP call
↓
slow external operation
↓
update
```

deberá poder producir advertencias en developer tooling.

---

# 156. User interaction inside transaction

Especialmente peligroso:

```text
lock row
↓
wait for human action
```

No deberá recomendarse.

---

# 157. Lock diagnostics

API conceptual:

```php
DB::concurrency()
    ->pessimistic()
    ->explain($query);
```

---

# 158. Explain output

```text
PESSIMISTIC LOCK PLAN

Mode:
    EXCLUSIVE

Wait:
    NO_WAIT

Scope:
    SELECTED_ROWS

Transaction:
    ACTIVE

Connection:
    WRITER / PINNED

Platform Capability:
    EXACT

Isolation:
    READ_COMMITTED

Targets:
    orders

Compiler Strategy:
    platform-specific locking clause
```

---

# 159. Runtime diagnostic

```text
LOCK ACQUISITION

Transaction:
    tx-41

Mode:
    EXCLUSIVE

Result:
    ACQUIRED

Wait Duration:
    3.8 ms

Query Fingerprint:
    qf-18...

Nested Depth:
    0
```

---

# 160. Telemetry

Métricas propuestas:

```text
db.lock.request
db.lock.acquired
db.lock.wait.duration
db.lock.timeout
db.lock.not_available
db.lock.skipped
```

---

# 161. Metric labels

Permitidos con cardinalidad controlada:

```text
mode
wait_policy
platform
outcome
```

---

# 162. Forbidden labels

No:

```text
entity id
transaction id
raw SQL
tenant id
bound parameters
```

como metric labels.

---

# 163. Tracing

Evento:

```text
db.lock.acquire
```

atributos:

```text
db.lock.mode=exclusive
db.lock.wait_policy=nowait
db.lock.outcome=acquired
```

---

# 164. Wait duration

Podrá medirse:

```text
query dispatch
→
locking query completion
```

pero eso incluye potencialmente:

```text
execution time + lock wait
```

si el driver no permite separar ambos.

---

# 165. Measurement confidence

Telemetry deberá distinguir:

```text
LOCK_WAIT_EXACT
QUERY_DURATION_APPROXIMATION
UNKNOWN
```

---

# 166. No fake precision

No reportar:

```text
lock_wait=142ms
```

como exacto si solo conocemos duración total de query.

---

# 167. Query profiler integration

Query Profiler podrá marcar:

```text
locking_query=true
```

y asociar:

```text
LockIntent
```

sanitizado.

---

# 168. Slow query diagnostics

Una locking query lenta puede deberse a:

- lock wait;
- execution cost;
- I/O;
- server load.

No se clasificará automáticamente como lock contention.

---

# 169. Platform introspection

Una herramienta administrativa opcional podrá enriquecer diagnostics con lock wait information del servidor.

Pero no será requisito del runtime.

---

# 170. Privileges

El usuario normal de aplicación no deberá necesitar permisos administrativos para utilizar el API core de locking.

---

# 171. Persistent runtime

Compartible:

```text
immutable lock capability profiles
compiled metadata
compiler strategies
lock policy definitions
```

---

# 172. Scoped mutable state

```text
LockAcquisitionRecord
TransactionLockRegistry
LockHandle
wait measurements
```

será transaction/request/operation scoped.

---

# 173. No global registry

Prohibido:

```php
static array $locks = [];
```

para representar locks actuales de todas las requests.

---

# 174. FrankenPHP

Al terminar request:

```text
TransactionContext
TransactionLockRegistry
LockHandles
```

deberán quedar completados/eliminados.

---

# 175. Leak detection

Si termina un scope con:

```text
ACTIVE transaction
+
known lock acquisitions
```

el problema principal será:

```text
transaction leak
```

y deberá activarse el cleanup seguro definido por Transaction Context.

---

# 176. No manual unlock cleanup

Cleanup deberá:

```text
rollback/close/quarantine transaction connection
```

según estado.

No ejecutar pseudo-unlocks inventados.

---

# 177. RoadRunner

Cada operation deberá recibir registry nuevo.

---

# 178. OpenSwoole

Registry deberá ser:

```text
coroutine/fiber scoped
```

cuando exista concurrencia dentro del worker.

---

# 179. Parallel statements

Por default, una transacción con locks no habilitará ejecución paralela sobre la misma conexión.

---

# 180. Why

Aunque PHP pueda crear fibers/coroutines:

```text
one physical DB connection
```

puede no soportar múltiples operaciones concurrentes seguras.

---

# 181. Parallel lock requests

Solo se permitirán cuando:

```text
driver capability
+
connection capability
+
protocol capability
+
transaction policy
```

lo garanticen.

---

# 182. Cancellation

Si una operación esperando lock es cancelada:

```text
CancellationRequested
```

no implica automáticamente:

```text
DB statement cancelled
```

---

# 183. Driver cancellation

Query Timeout/Cancellation System deberá intentar cancelar cuando exista capability.

---

# 184. Cancellation outcome

Después de cancelar deberá conocerse:

```text
statement outcome
transaction impact
connection health
```

antes de continuar.

---

# 185. Unknown cancellation

Si no puede determinarse:

```text
TransactionHealth
→ UNCERTAIN
```

o estado equivalente.

---

# 186. Exception hierarchy

```text
DatabaseConcurrencyException
│
├── PessimisticLockException
│   ├── PessimisticLockRequiresTransactionException
│   ├── PessimisticLockNotAvailableException
│   ├── PessimisticLockTimeoutException
│   ├── PessimisticLockUnsupportedException
│   ├── PessimisticLockTargetException
│   ├── PessimisticLockConnectionMismatchException
│   ├── PessimisticLockReadOnlyTransactionException
│   └── PessimisticLockStateException
│
└── PessimisticLockInvariantViolationException
```

---

# 187. Canonical shared exceptions

Cuando corresponda, podrá reutilizarse la jerarquía canónica:

```text
LockTimeoutException
LockNotAvailableException
DeadlockDetectedException
```

del sistema de concurrencia.

Las excepciones específicas de Pessimistic Locking podrán enriquecer contexto sin duplicar clasificación.

---

# 188. Unsupported lock

Ejemplo:

```text
Requested:
    SHARED + SKIP_LOCKED

Platform:
    unsupported
```

Resultado:

```text
PessimisticLockUnsupportedException
```

antes de ejecutar una traducción más débil.

---

# 189. Transaction missing

```text
lockForUpdate()
+
no active transaction
```

Resultado default:

```text
PessimisticLockRequiresTransactionException
```

---

# 190. Read-only conflict

```text
READ_ONLY transaction
+
EXCLUSIVE lock
```

Resultado:

```text
PessimisticLockReadOnlyTransactionException
```

cuando incompatible.

---

# 191. Testing architecture

La suite deberá incluir:

```text
PessimisticLockIntentTests
PessimisticLockAstTests
PessimisticLockBuilderTests
PessimisticLockCompilerTests
PessimisticLockCapabilityTests
PessimisticLockTransactionTests
PessimisticLockEntityManagerTests
PessimisticSharedLockTests
PessimisticExclusiveLockTests
PessimisticNowaitTests
PessimisticSkipLockedTests
PessimisticTimeoutTests
PessimisticDeadlockTests
PessimisticNestedTransactionTests
PessimisticRelationshipTests
PessimisticQueueClaimTests
PessimisticTelemetryTests
PessimisticPersistentRuntimeTests
PessimisticPlatformConformanceTests
```

---

# 192. Transaction requirement test

Ejecutar:

```text
lockForUpdate()
```

fuera de transaction.

Esperado:

```text
PessimisticLockRequiresTransactionException
```

---

# 193. Exclusive lock test

Connection A:

```text
BEGIN
lock row 1 exclusive
```

Connection B:

```text
request incompatible lock row 1
```

Verificar que B:

```text
waits/fails
```

según policy.

---

# 194. NOWAIT test

A mantiene lock.

B solicita:

```text
EXCLUSIVE + NO_WAIT
```

Esperado:

```text
LockNotAvailableException
```

sin espera significativa, dentro de las capabilities de plataforma.

---

# 195. SKIP LOCKED test

A bloquea row 1.

B consulta:

```text
rows 1,2,3
SKIP LOCKED
```

Esperado conceptualmente:

```text
2,3
```

si la plataforma ofrece esa semántica.

---

# 196. Coverage test

Resultado obtenido mediante `SKIP_LOCKED` no deberá marcar una relación como completa si filas pudieron omitirse.

---

# 197. Timeout test

A mantiene lock.

B solicita WAIT con timeout.

Esperado:

```text
LockTimeoutException
```

y transaction impact correctamente clasificado.

---

# 198. Deadlock test

A:

```text
lock 1
request 2
```

B:

```text
lock 2
request 1
```

Verificar integración con:

```text
Deadlock Handling System
```

---

# 199. Lock ordering test

El planner deberá producir orden determinista cuando la policy y semántica lo permitan.

---

# 200. REQUIRES_NEW self-block test

Outer bloquea row.

Inner `REQUIRES_NEW` intenta misma row.

Verificar:

- no se finge adquisición;
- timeout/cancellation funciona;
- diagnostic puede señalar riesgo.

---

# 201. Savepoint test

No asumir que rollback to savepoint libera todos los locks.

La suite deberá comprobar únicamente las guarantees declaradas por Platform.

---

# 202. Writer routing test

Una locking SELECT deberá utilizar:

```text
transaction pinned writer connection
```

y nunca una replica normal.

---

# 203. Read-only transaction test

Verificar rejection/capability handling para lock incompatible.

---

# 204. Targeted lock test

En join:

```text
Order JOIN Customer
```

solicitar lock solo sobre Order.

Validar compilation/capability.

---

# 205. Unsupported target test

La plataforma no puede representar target exacto.

Esperado:

```text
UNSUPPORTED / explicit limitation
```

no silent broadening.

---

# 206. Queue claim test

Varios workers concurrentes:

```text
LIMIT N
+
EXCLUSIVE
+
SKIP_LOCKED
```

deberán poder obtener conjuntos no solapados bajo guarantees de plataforma.

---

# 207. Worker cleanup test

Al finalizar transaction:

```text
TransactionLockRegistry
```

debe vaciarse.

---

# 208. Coroutine isolation test

Coroutine A y B deberán tener registries separados.

---

# 209. Capability conformance matrix

Cada platform adapter deberá declarar y probar:

| Capability | Requirement |
|---|---|
| Shared locking | Explicit status |
| Exclusive locking | Explicit status |
| NOWAIT | Explicit status |
| SKIP LOCKED | Explicit status |
| Lock wait timeout | Explicit status |
| Target-specific locking | Explicit status |
| Read-only compatibility | Explicit status |
| Savepoint lock behavior | Explicit/Unknown |
| Range/predicate implications | Diagnostic capability |
| Cancellation impact | Explicit/Unknown |

---

# 210. Proposed directory structure

```text
src/Quantum/Database/Concurrency/
│
├── Pessimistic/
│   ├── LockIntent.php
│   ├── LockMode.php
│   ├── LockScope.php
│   ├── LockTarget.php
│   ├── LockTargetSet.php
│   ├── LockWaitMode.php
│   ├── LockWaitPolicy.php
│   │
│   ├── Ast/
│   │   └── LockClause.php
│   │
│   ├── Planning/
│   │   ├── EffectiveLockPlan.php
│   │   ├── LockAcquisitionPlan.php
│   │   ├── LockAcquisitionStep.php
│   │   ├── LockOrderingPolicy.php
│   │   └── LockOrderingGuarantee.php
│   │
│   ├── Capability/
│   │   ├── PessimisticLockCapability.php
│   │   ├── LockCompatibilityStatus.php
│   │   ├── LockCapabilityResolver.php
│   │   └── LockCompatibilityAnalyzer.php
│   │
│   ├── Runtime/
│   │   ├── LockAcquisitionRecord.php
│   │   ├── LockAcquisitionStatus.php
│   │   ├── LockHandle.php
│   │   └── TransactionLockRegistry.php
│   │
│   ├── ORM/
│   │   ├── EntityLockCoordinator.php
│   │   ├── EntityLockTargetResolver.php
│   │   └── RelationshipLockCoordinator.php
│   │
│   ├── Diagnostics/
│   │   ├── PessimisticLockInspector.php
│   │   ├── LockDiagnosticReport.php
│   │   └── LockMeasurementConfidence.php
│   │
│   ├── Telemetry/
│   │   └── PessimisticLockTelemetry.php
│   │
│   └── Exception/
│       ├── PessimisticLockException.php
│       ├── PessimisticLockRequiresTransactionException.php
│       ├── PessimisticLockUnsupportedException.php
│       ├── PessimisticLockTargetException.php
│       ├── PessimisticLockConnectionMismatchException.php
│       ├── PessimisticLockReadOnlyTransactionException.php
│       ├── PessimisticLockStateException.php
│       └── PessimisticLockInvariantViolationException.php
```

---

# 211. Dependency model

Permitido:

```text
Pessimistic Locking
        ↓
Query AST
Query Builder Contracts
Transaction Context
Connection Manager
Platform Capabilities
SQL Compiler
Execution Engine
ORM Metadata
Telemetry
Diagnostics
```

No permitido:

```text
Pessimistic Locking
        ↓
PDO directly
HTTP globals
Queue implementation
Redis distributed lock
static current transaction
static lock registry
business-specific aggregate rules
```

---

# 212. Architectural invariants

## DB-PL-001
Pessimistic locking será una intención semántica explícita.

## DB-PL-002
Lock intent no será una cadena SQL.

## DB-PL-003
SQL locking syntax pertenecerá al compiler.

## DB-PL-004
Core no asumirá `FOR UPDATE` universal.

## DB-PL-005
Shared y exclusive serán modos canónicos.

## DB-PL-006
Platform podrá ampliar modos mediante capabilities.

## DB-PL-007
Lock mode será distinto de isolation level.

## DB-PL-008
Lock mode será distinto de optimistic versioning.

## DB-PL-009
Lock mode será distinto de distributed lock.

## DB-PL-010
Lock mode será distinto de advisory lock.

## DB-PL-011
Lock wait policy será explícita.

## DB-PL-012
WAIT será distinto de NO_WAIT.

## DB-PL-013
NO_WAIT será distinto de TIMEOUT.

## DB-PL-014
SKIP_LOCKED será distinto de WAIT.

## DB-PL-015
SKIP_LOCKED podrá modificar el conjunto observable.

## DB-PL-016
SKIP_LOCKED result no será asumido completo.

## DB-PL-017
Lock wait timeout será distinto de query timeout.

## DB-PL-018
Query timeout será distinto de transaction timeout.

## DB-PL-019
Transaction timeout será distinto de operation deadline.

## DB-PL-020
Lock scope será explícito.

## DB-PL-021
Logical lock scope no fingirá describir todos los physical locks.

## DB-PL-022
VoltStack no mantendrá un physical lock graph ficticio.

## DB-PL-023
Lock target será semántico.

## DB-PL-024
Lock target no será raw table string por default.

## DB-PL-025
LockClause formará parte del Query AST.

## DB-PL-026
Query AST permanecerá vendor-neutral.

## DB-PL-027
Builder convenience APIs convergerán en LockClause.

## DB-PL-028
Raw locking será escape hatch explícito.

## DB-PL-029
Raw locking no recibirá portability guarantees completas.

## DB-PL-030
Pessimistic row locking requerirá active transaction por default.

## DB-PL-031
VoltStack no abrirá hidden transaction que termine antes del protected business interval.

## DB-PL-032
Lock outside transaction fallará por default.

## DB-PL-033
EntityManager lock no generará SQL directamente.

## DB-PL-034
Entity lock requerirá identity.

## DB-PL-035
Entity lock preservará execution domain.

## DB-PL-036
Entity lock utilizará transaction pinned connection.

## DB-PL-037
Detached entity locking requerirá explicit semantics.

## DB-PL-038
Marcar un objeto como locked en memoria no equivaldrá a DB lock.

## DB-PL-039
Lock acquisition requerirá DB confirmation.

## DB-PL-040
Lock acquisition record no será DB lock authority.

## DB-PL-041
Lock lifetime estará ligado a physical transaction semantics.

## DB-PL-042
No existirá portable arbitrary row unlock API.

## DB-PL-043
Savepoint rollback no garantizará universalmente release de locks.

## DB-PL-044
Nested REQUIRED compartirá physical lock domain.

## DB-PL-045
NESTED no creará lock lifetime independiente.

## DB-PL-046
REQUIRES_NEW tendrá physical transaction distinta.

## DB-PL-047
REQUIRES_NEW podrá bloquearse contra outer suspendido.

## DB-PL-048
Known self-contention podrá generar diagnostic.

## DB-PL-049
Diagnostic no declarará deadlock sin evidencia.

## DB-PL-050
Locking query permanecerá en pinned writer.

## DB-PL-051
Locking SELECT no será tratado como ordinary read routing.

## DB-PL-052
Locking query no será enviada a replica normal.

## DB-PL-053
Transaction connection affinity tendrá prioridad sobre query syntax.

## DB-PL-054
Isolation y explicit locking permanecerán separados.

## DB-PL-055
Effective isolation participará en compatibility analysis.

## DB-PL-056
Read-only transaction podrá rechazar incompatible exclusive locking.

## DB-PL-057
Platform capabilities serán explícitas.

## DB-PL-058
MySQL tendrá capabilities propias.

## DB-PL-059
MariaDB tendrá capabilities propias.

## DB-PL-060
PostgreSQL tendrá capabilities propias.

## DB-PL-061
SQLite tendrá capabilities propias.

## DB-PL-062
Platform version podrá cambiar capabilities.

## DB-PL-063
Unsupported lock intent no será degradado silenciosamente.

## DB-PL-064
Unsupported NOWAIT no se convertirá silenciosamente a WAIT.

## DB-PL-065
Unsupported SKIP_LOCKED no se convertirá silenciosamente a WAIT.

## DB-PL-066
Lock scope no se ampliará silenciosamente cuando cambie significativamente concurrencia.

## DB-PL-067
Stronger lock no será asumido automáticamente aceptable.

## DB-PL-068
Emulation deberá preservar semantics.

## DB-PL-069
Approximate behavior no será suficiente para declarar exact support.

## DB-PL-070
Compiler no decidirá locking policy.

## DB-PL-071
Compiler solo traducirá effective lock plan validado.

## DB-PL-072
Targeted locking será capability-driven.

## DB-PL-073
Join locking no asumirá que todos los tables deben bloquearse.

## DB-PL-074
Unsupported exact target producirá limitation/error explícito.

## DB-PL-075
Range/predicate locking no será fingido como portable abstraction.

## DB-PL-076
Physical lock footprint podrá depender del execution plan.

## DB-PL-077
Physical lock footprint podrá depender de índices.

## DB-PL-078
Physical lock footprint podrá depender de isolation.

## DB-PL-079
Performance diagnostics podrán señalar index risk.

## DB-PL-080
Pessimistic system no cambiará índices automáticamente.

## DB-PL-081
Consistent lock ordering será recomendado.

## DB-PL-082
Lock ordering no cambiará business semantics.

## DB-PL-083
Lock acquisition planner podrá ordenar targets seguros.

## DB-PL-084
Single locking query podrá preferirse cuando preserve semantics.

## DB-PL-085
LIMIT + locking tendrá semantics explícitas.

## DB-PL-086
SKIP_LOCKED será válido para queue-claim patterns cuando Platform lo soporte.

## DB-PL-087
Database locking no será Queue System.

## DB-PL-088
SKIP_LOCKED relationship hydration podrá producir partial coverage.

## DB-PL-089
Locking pagination no garantizará stable global snapshot automáticamente.

## DB-PL-090
Cursor pagination no será lock.

## DB-PL-091
Lock timeout utilizará canonical concurrency classification.

## DB-PL-092
NOWAIT failure utilizará canonical lock-not-available classification.

## DB-PL-093
SKIP_LOCKED omission no será excepción normal.

## DB-PL-094
Pessimistic locking no impedirá todos los deadlocks.

## DB-PL-095
Deadlock Handling System clasificará deadlocks.

## DB-PL-096
Pessimistic Locking System no ejecutará deadlock retry.

## DB-PL-097
Transaction Retry System decidirá retry.

## DB-PL-098
NO_WAIT semantics deberán preservarse frente a retry policy.

## DB-PL-099
Retry no deberá convertir NO_WAIT en wait infinito.

## DB-PL-100
SKIP_LOCKED no deberá causar immediate retry del row omitido por default.

## DB-PL-101
Lock timeout podrá ser retry candidate.

## DB-PL-102
Retry candidacy no equivaldrá a retry decision.

## DB-PL-103
Lock acquisition status será explícito.

## DB-PL-104
UNKNOWN acquisition permanecerá UNKNOWN.

## DB-PL-105
Connection loss deberá evaluarse mediante Transaction Context.

## DB-PL-106
Locks no se transferirán entre conexiones.

## DB-PL-107
Locks no se transferirán entre transactions.

## DB-PL-108
Locks no se transferirán entre requests.

## DB-PL-109
Locks no se transferirán entre coroutines.

## DB-PL-110
LockHandle será scoped.

## DB-PL-111
LockHandle no será DB lock authority.

## DB-PL-112
LockHandle será terminal tras transaction completion.

## DB-PL-113
Stale LockHandle no será reutilizable.

## DB-PL-114
TransactionLockRegistry será scoped.

## DB-PL-115
TransactionLockRegistry no será Database Lock Manager.

## DB-PL-116
Local registry no garantizará conocimiento exhaustivo de locks físicos.

## DB-PL-117
Lock upgrade podrá requerir DB operation.

## DB-PL-118
Lock upgrade podrá bloquear.

## DB-PL-119
Lock upgrade podrá deadlockear.

## DB-PL-120
Registry local no fingirá lock upgrade.

## DB-PL-121
Portable lock downgrade no será asumido.

## DB-PL-122
Relationship lock no implicará aggregate lock.

## DB-PL-123
Locking existing child rows no impedirá necesariamente nuevos children.

## DB-PL-124
Phantom protection dependerá de isolation/platform strategy.

## DB-PL-125
Aggregate root locking será explicit policy.

## DB-PL-126
Polymorphic lock preservará type + identity.

## DB-PL-127
Cross-database lock no será single transaction lock.

## DB-PL-128
Shard lock estará asociado a su shard.

## DB-PL-129
Tenant isolation permanecerá obligatoria.

## DB-PL-130
Lock query no bypassará tenant boundaries.

## DB-PL-131
Pessimistic lock no concederá authorization.

## DB-PL-132
User input no controlará raw lock SQL.

## DB-PL-133
Resource governance limitará lock waits.

## DB-PL-134
Resource governance podrá limitar batch sizes.

## DB-PL-135
Resource governance podrá limitar transaction duration.

## DB-PL-136
Long-held lock diagnostics usarán measurement confidence.

## DB-PL-137
External I/O while holding locks podrá advertirse.

## DB-PL-138
Human interaction dentro de lock transaction no será patrón recomendado.

## DB-PL-139
Telemetry será observational.

## DB-PL-140
Telemetry no modificará locking semantics.

## DB-PL-141
Metric labels tendrán cardinalidad limitada.

## DB-PL-142
Entity IDs no serán metric labels.

## DB-PL-143
Transaction IDs no serán metric labels.

## DB-PL-144
Raw SQL no será metric label.

## DB-PL-145
Bound values no serán metric labels.

## DB-PL-146
Lock wait measurement no fingirá precisión inexistente.

## DB-PL-147
Slow locking query no será automáticamente lock contention.

## DB-PL-148
Administrative lock introspection será opcional.

## DB-PL-149
Application DB user no requerirá admin privileges para core locking.

## DB-PL-150
Immutable capability definitions podrán compartirse entre workers.

## DB-PL-151
Mutable lock state será execution-scoped.

## DB-PL-152
Static global lock registry estará prohibido.

## DB-PL-153
FrankenPHP requests no compartirán lock registry.

## DB-PL-154
RoadRunner operations no compartirán lock registry.

## DB-PL-155
OpenSwoole coroutines no compartirán lock registry.

## DB-PL-156
Scope cleanup no intentará fake unlock.

## DB-PL-157
Leaked transaction será tratado por Transaction Lifecycle.

## DB-PL-158
Parallel operations sobre misma connection estarán prohibidas por default.

## DB-PL-159
Parallel locking requerirá explicit capabilities.

## DB-PL-160
Cancellation request no equivaldrá a successful DB cancellation.

## DB-PL-161
Cancellation outcome deberá reconciliar transaction state.

## DB-PL-162
Unknown cancellation podrá taintar transaction context.

## DB-PL-163
Testing incluirá competing physical connections.

## DB-PL-164
Testing incluirá WAIT.

## DB-PL-165
Testing incluirá NOWAIT.

## DB-PL-166
Testing incluirá SKIP_LOCKED.

## DB-PL-167
Testing incluirá timeout.

## DB-PL-168
Testing incluirá deadlocks.

## DB-PL-169
Testing incluirá nested transactions.

## DB-PL-170
Testing incluirá writer routing.

## DB-PL-171
Testing incluirá targeted locks.

## DB-PL-172
Testing incluirá relationship coverage.

## DB-PL-173
Testing incluirá queue claiming.

## DB-PL-174
Testing incluirá persistent workers.

## DB-PL-175
Cada nuevo Platform adapter deberá declarar locking capabilities.

## DB-PL-176
UNKNOWN capability no será tratado como SUPPORTED.

## DB-PL-177
Unsupported capability deberá fallar antes de emitir SQL cuando sea posible.

## DB-PL-178
VoltStack nunca afirmará que posee un lock únicamente porque la aplicación lo solicitó.

## DB-PL-179
VoltStack nunca afirmará que un lock continúa activo después de que su transaction boundary haya finalizado.

## DB-PL-180
VoltStack nunca debilitará silenciosamente una intención de locking para hacer que una query pueda ejecutarse.

---

# 213. Anti-patterns

## 213.1 SQL strings como modelo

```php
$query->lock('FOR UPDATE NOWAIT');
```

como única abstracción.

**Rechazado.**

---

## 213.2 Lock fuera de transacción

```text
locking SELECT
↓
autocommit ends
↓
business logic
```

**Falsa garantía.**

---

## 213.3 Hidden transaction

```text
lockForUpdate()
↓
framework opens tx
↓
SELECT
↓
framework commits
↓
returns entity
```

**Prohibido.**

---

## 213.4 Read replica locking

```text
SELECT FOR UPDATE
→ replica
```

**Prohibido por default.**

---

## 213.5 Local PHP lock registry como autoridad

```text
$locks[$id] = true;
```

**No equivale a DB lock.**

---

## 213.6 Unlock manual ficticio

```php
$entityManager->unlock($entity);
```

sin finalizar/modificar realmente el boundary físico.

**No portable.**

---

## 213.7 Silent NOWAIT downgrade

```text
NOWAIT unsupported
→ WAIT
```

**Prohibido.**

---

## 213.8 Silent target broadening

```text
lock Order only
→ platform locks entire joined result
→ framework says exact
```

**Prohibido.**

---

## 213.9 SKIP LOCKED como resultado completo

```text
rows skipped due locks
→ collection marked COMPLETE
```

**Incorrecto.**

---

## 213.10 Lock all relationships automatically

```text
lock entity
→ recursively lock entire object graph
```

**Rechazado.**

---

## 213.11 Retry NOWAIT forever

```text
NO_WAIT
→ fail
→ immediate retry
→ fail
→ immediate retry...
```

**Contradice la intención.**

---

## 213.12 Assume savepoint releases locks

```text
ROLLBACK TO SAVEPOINT
→ all later locks definitely gone
```

**No portable.**

---

# 214. Master formulas

## Effective lock

```text
EffectiveLockPlan
=
Resolve(
    RequestedLockIntent,
    PlatformCapabilities,
    EffectiveIsolation,
    TransactionDefinition,
    ExecutionDomain
)
```

---

## Lock acquisition

```text
LockAcquired
⇔
DatabaseConfirms(
    EffectiveLockPlan,
    PinnedConnection
)
```

No:

```text
LockAcquired
⇔
ApplicationRequestedLock
```

---

## Transaction affinity

Para toda locking operation `L` dentro de `T`:

```text
Connection(L)
=
PinnedConnection(T)
```

---

## Wait budget

Conceptualmente:

```text
EffectiveWaitBudget
=
min(
    RequestedLockWait,
    TransactionRemainingTime,
    OperationRemainingDeadline
)
```

cuando la plataforma pueda representar dicha limitación.

---

## Lock result completeness

```text
SKIP_LOCKED
→
ResultCoverage ≠ GuaranteedComplete
```

---

## Relationship protection

```text
Lock(CurrentChildren)
≠
Prevent(NewChildren)
```

sin garantías adicionales.

---

## Deadlock relationship

```text
PessimisticLocking
+
InconsistentAcquisitionOrder
→
HigherDeadlockRisk
```

pero no:

```text
PessimisticLocking
→
DeadlockGuaranteed
```

---

# 215. Modelo final

```text
                 Application / ORM
                        │
                        ▼
                    LockIntent
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       LockMode      WaitPolicy     Targets
          │             │             │
          └─────────────┼─────────────┘
                        ▼
              Transaction Validation
                        │
                        ▼
               Capability Resolver
                        │
                        ▼
                EffectiveLockPlan
                        │
                        ▼
                    Query AST
                        │
                        ▼
                   SQL Compiler
                        │
                        ▼
                  Query Executor
                        │
                        ▼
              Pinned DB Connection
                        │
                        ▼
                    Database
                        │
             ┌──────────┼────────────┐
             ▼          ▼            ▼
         ACQUIRED     WAIT       SKIPPED
                         │
                ┌────────┼────────┐
                ▼        ▼        ▼
             ACQUIRED TIMEOUT  DEADLOCK
                         │        │
                         ▼        ▼
                   Concurrency Failure
                         │
                         ▼
                Canonical Classifier
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
         Diagnostics  Telemetry  Retry System
```

---

# 216. Decisiones arquitectónicas finales

VoltStack modelará pessimistic locking mediante:

```text
LockIntent
+
TransactionContext
+
PlatformCapabilities
+
EffectiveLockPlan
```

y no mediante strings SQL dispersos.

Las APIs ergonómicas:

```php
lockForUpdate()
lockForShare()
```

serán únicamente fachadas sobre el modelo semántico.

El Query AST representará:

```text
LockMode
LockWaitPolicy
LockTargetSet
```

mientras que únicamente el SQL Compiler conocerá la sintaxis concreta del motor.

La adquisición de locks estará vinculada a:

```text
ACTIVE TRANSACTION
+
PINNED CONNECTION
```

y VoltStack no creará hidden transactions que terminen antes de que la aplicación utilice el recurso protegido.

Una locking read será considerada una operación con intención autoritativa:

```text
Locking SELECT
→
Writer/Pinned Connection
```

aunque sintácticamente sea un `SELECT`.

Las diferencias entre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

se resolverán mediante capabilities y no mediante la suposición de que `FOR UPDATE`, `FOR SHARE`, `NOWAIT` o `SKIP LOCKED` tienen equivalencia universal.

VoltStack mantendrá:

```text
Isolation
≠
Pessimistic Lock
```

y también:

```text
Pessimistic Lock
≠
Optimistic Lock
```

Los tres mecanismos podrán cooperar dentro de una estrategia global de concurrencia, pero conservarán responsabilidades independientes.

`SKIP_LOCKED` será tratado especialmente porque:

```text
matching row
+
currently incompatible lock
→
may be omitted
```

por lo que el resultado no deberá etiquetarse automáticamente como completo. Esto será particularmente importante en:

- workers;
- queues;
- batch processors;
- relationship hydration;
- pagination concurrente.

Los failures de locking serán enviados al sistema canónico de concurrencia:

```text
LockNotAvailable
LockTimeout
Deadlock
```

y únicamente `TransactionRetrySystem` decidirá si corresponde un retry.

No se debilitarán silenciosamente intenciones como:

```text
NOWAIT → WAIT
SKIP_LOCKED → WAIT
exact target → broader target
```

porque aun una garantía aparentemente “más fuerte” puede cambiar significativamente:

- latencia;
- throughput;
- deadlock risk;
- fairness;
- business behavior.

Finalmente:

```text
TransactionLockRegistry
```

será únicamente una representación local y acotada de las adquisiciones conocidas por VoltStack.

Nunca será considerado:

```text
Database Lock Manager
```

ni se utilizará para fingir que un lock existe, fue liberado o fue actualizado sin confirmación del DBMS.

La regla final será:

> **VoltStack solo considerará adquirido un pessimistic lock cuando la base de datos lo haya confirmado dentro de la transacción y conexión correctas; la intención solicitada nunca será equivalente por sí misma a la garantía obtenida.**

---

# 217. Relación con Transaction & Concurrency

```text
164_DATABASE_TRANSACTION_ARCHITECTURE
                ↓
165_DATABASE_TRANSACTION_MANAGER_SYSTEM
                ↓
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM
                ↓
167_DATABASE_TRANSACTION_ISOLATION_SYSTEM
                ↓
168_DATABASE_NESTED_TRANSACTION_SYSTEM
                ↓
169_DATABASE_SAVEPOINT_SYSTEM
                ↓
170_DATABASE_TRANSACTION_RETRY_SYSTEM
                ↓
171_DATABASE_DEADLOCK_HANDLING_SYSTEM
                ↓
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM
                ↓
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM
                ↓
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM
                ↓
175_DATABASE_TRANSACTION_EVENT_SYSTEM
```

---

# 218. Siguiente documento

```text
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md
```

El siguiente documento consolidará los mecanismos anteriores en la arquitectura general de concurrencia de VoltStack y deberá definir:

- canonical concurrency model;
- concurrency control coordinator;
- transaction isolation;
- optimistic locking;
- pessimistic locking;
- deadlocks;
- lock timeouts;
- serialization failures;
- lost updates;
- write skew;
- stale entities;
- concurrency conflict taxonomy;
- concurrency strategy selection;
- isolation + optimistic locking composition;
- isolation + pessimistic locking composition;
- aggregate concurrency;
- relationship concurrency;
- cross-request concurrency;
- retry coordination;
- transaction context integration;
- UnitOfWork integration;
- Query Engine integration;
- resource contention;
- fairness;
- starvation;
- livelock;
- backpressure;
- concurrency budgets;
- diagnostics;
- telemetry;
- testing;
- persistent-runtime safety;
- final architectural invariants.