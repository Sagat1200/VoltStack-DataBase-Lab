# 174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md

# VoltStack Quantum Database
## Database Concurrency Control System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 174 — Database Concurrency Control System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md`  
**Siguiente documento:** `175_DATABASE_TRANSACTION_EVENT_SYSTEM.md`

---

# 1. Propósito

`Database Concurrency Control System` define la arquitectura global mediante la cual VoltStack coordinará operaciones concurrentes sobre datos compartidos.

Los documentos anteriores definieron mecanismos individuales:

```text
Transaction Isolation
Nested Transactions
Savepoints
Transaction Retry
Deadlock Handling
Optimistic Locking
Pessimistic Locking
```

Este documento establece cómo dichos mecanismos cooperarán como un único modelo coherente de concurrencia.

La regla central será:

> **VoltStack tratará la concurrencia como una propiedad explícita de la operación, la transacción, el execution domain y los recursos afectados; ningún mecanismo individual —isolation, locking, retries, UnitOfWork o IdentityMap— será considerado por sí solo una garantía universal de consistencia concurrente.**

Formalmente:

```text
ConcurrencySafety
=
TransactionSemantics
+
IsolationSemantics
+
ConflictDetection
+
LockingStrategy
+
RetryPolicy
+
DomainInvariants
+
DatabaseConstraints
```

No:

```text
ConcurrencySafety
=
SERIALIZABLE
```

ni:

```text
ConcurrencySafety
=
FOR UPDATE
```

ni:

```text
ConcurrencySafety
=
version column
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. modelo canónico de concurrencia;
2. clasificación uniforme de conflictos;
3. estrategias optimistic y pessimistic;
4. integración con transaction isolation;
5. coordinación con Transaction Manager;
6. integración con UnitOfWork;
7. protección contra lost updates;
8. detección de stale writes;
9. tratamiento de serialization failures;
10. tratamiento de deadlocks;
11. lock timeout handling;
12. estrategia de retry;
13. aggregate concurrency;
14. relationship concurrency;
15. resource contention management;
16. concurrency budgets;
17. backpressure;
18. starvation diagnostics;
19. livelock diagnostics;
20. fairness policies cuando sean representables;
21. persistent-runtime isolation;
22. telemetry;
23. diagnostics;
24. extensibilidad;
25. testing multi-connection.

---

# 3. No objetivos

Este sistema no deberá:

- sustituir el Transaction Manager;
- implementar el lock manager del DBMS;
- implementar un distributed consensus protocol;
- implementar distributed transactions;
- convertir Redis locks en database row locks;
- eliminar todos los race conditions automáticamente;
- inferir invariantes de negocio desconocidas;
- serializar globalmente todas las requests;
- hacer retry ilimitado;
- ocultar conflictos concurrentes;
- mantener estado concurrente global mutable entre requests;
- sustituir database constraints.

---

# 4. Problema fundamental

Considérese:

```text
Balance = 100
```

Dos operaciones concurrentes:

```text
Transaction A               Transaction B

READ Balance = 100          READ Balance = 100

+ 50                        - 30

WRITE 150                   WRITE 70
```

Resultado:

```text
70
```

o:

```text
150
```

cuando el resultado correcto esperado pudiera ser:

```text
120
```

Este problema es un:

```text
Lost Update
```

Pero no todos los problemas concurrentes tienen esta forma.

VoltStack deberá modelar explícitamente diferentes anomalías.

---

# 5. Modelo general

```text
Application Operation
        │
        ▼
Concurrency Requirements
        │
        ▼
Concurrency Strategy Resolver
        │
        ├───────────────┐
        ▼               ▼
Transaction         Domain Rules
        │               │
        ▼               │
Isolation              │
        │               │
        ├───────────────┤
        ▼               ▼
Optimistic / Pessimistic
        │
        ▼
Query / ORM
        │
        ▼
Database
        │
        ▼
Outcome Classification
        │
   ┌────┼─────┐
   ▼    ▼     ▼
Success Conflict Unknown
        │
        ▼
Retry / Surface / Abort
```

---

# 6. Distinciones fundamentales

```text
Concurrency Control
≠
Transaction Management
≠
Transaction Isolation
≠
Optimistic Locking
≠
Pessimistic Locking
≠
Retry
≠
Deadlock Handling
≠
Database Constraint
≠
IdentityMap
≠
UnitOfWork
```

---

# 7. Concurrency control como coordinación

`ConcurrencyControlSystem` no deberá convertirse en un God Object.

Su función será coordinar contratos especializados.

```text
ConcurrencyControlSystem
├── Strategy Resolver
├── Conflict Classifier
├── Conflict Policy
├── Optimistic Coordinator
├── Pessimistic Coordinator
├── Isolation Integration
├── Retry Integration
├── Resource Governance
├── Diagnostics
└── Telemetry
```

---

# 8. Canonical Concurrency Context

Cada operación concurrentemente relevante podrá disponer de:

```php
final readonly class ConcurrencyContext
{
    public function __construct(
        public ConcurrencyOperationId $operationId,
        public ExecutionDomain $domain,
        public ConcurrencyRequirements $requirements,
        public ?TransactionId $transactionId,
        public ConcurrencyBudget $budget,
    ) {}
}
```

---

# 9. ConcurrencyContext ≠ TransactionContext

`TransactionContext` describe:

```text
physical/logical transaction lifecycle
```

`ConcurrencyContext` describe:

```text
concurrency requirements and observations
```

Una operación puede poseer ambos.

---

# 10. Scope

`ConcurrencyContext` deberá ser:

```text
operation scoped
```

Nunca:

```text
static/global mutable state
```

---

# 11. ConcurrencyOperationId

Una operación lógica deberá mantener identidad estable a través de retries:

```text
Operation
   │
   ├── Attempt 1 / Transaction A
   ├── Attempt 2 / Transaction B
   └── Attempt 3 / Transaction C
```

Por tanto:

```text
ConcurrencyOperationId
≠
TransactionId
```

---

# 12. Concurrency requirements

Modelo conceptual:

```php
final readonly class ConcurrencyRequirements
{
    public function __construct(
        public ConcurrencyStrategy $strategy,
        public ConflictPolicy $conflictPolicy,
        public ConsistencyRequirement $consistency,
        public ?LockIntent $lockIntent,
        public ?OptimisticRequirement $optimistic,
    ) {}
}
```

---

# 13. ConcurrencyStrategy

```php
enum ConcurrencyStrategy
{
    case PLATFORM_DEFAULT;
    case OPTIMISTIC;
    case PESSIMISTIC;
    case SERIALIZABLE;
    case HYBRID;
    case CUSTOM;
}
```

Estos nombres expresan estrategia de alto nivel.

No sustituyen configuraciones detalladas.

---

# 14. PLATFORM_DEFAULT

Significa:

```text
no additional framework concurrency strategy
beyond configured transaction/database semantics
```

No significa:

```text
safe for every workload
```

---

# 15. OPTIMISTIC

Utiliza:

```text
read
→ work
→ conditional write
→ detect conflict
```

normalmente mediante:

```text
version token
```

---

# 16. PESSIMISTIC

Utiliza:

```text
acquire DB protection
→ work
→ write
```

dentro de una transacción.

---

# 17. SERIALIZABLE

Solicita aislamiento serializable cuando sea soportado.

Pero:

```text
SERIALIZABLE
≠
no retries
```

De hecho puede producir:

```text
serialization failures
```

que requieran replay.

---

# 18. HYBRID

Permitirá combinar:

```text
Isolation
+
Optimistic Lock
+
Selective Pessimistic Lock
```

cuando el dominio lo requiera.

---

# 19. CUSTOM

Extensiones avanzadas podrán definir estrategias especializadas.

No podrán romper invariantes core.

---

# 20. Strategy selection

La estrategia dependerá de:

```text
conflict frequency
conflict cost
operation duration
resource hotness
retry safety
latency budget
database capabilities
domain invariants
```

---

# 21. No universal default strategy

VoltStack no declarará:

```text
optimistic locking is always best
```

ni:

```text
pessimistic locking is always safer
```

---

# 22. Optimistic suitability

Frecuentemente apropiado cuando:

```text
conflicts rare
+
transactions/work may be relatively long
+
retry/user conflict resolution acceptable
```

---

# 23. Pessimistic suitability

Frecuentemente apropiado cuando:

```text
conflicts expected
+
critical resource must be serialized
+
transaction can remain short
```

---

# 24. Serializable suitability

Puede ser apropiado cuando:

```text
cross-row invariant
+
platform semantics sufficient
+
retryable transaction design
```

---

# 25. Hybrid example

Transferencia financiera conceptual:

```text
Transaction
    ↓
lock accounts in deterministic order
    ↓
validate invariant
    ↓
update balances
    ↓
optimistic version update
    ↓
commit
```

Esto puede parecer redundante, pero diferentes mecanismos pueden proteger distintas propiedades.

---

# 26. Canonical conflict taxonomy

VoltStack deberá normalizar conflictos en una jerarquía semántica.

```text
ConcurrencyConflict
├── OptimisticConflict
├── LockConflict
│   ├── LockNotAvailable
│   └── LockTimeout
├── Deadlock
├── SerializationFailure
├── StaleWrite
├── LostUpdateDetected
├── ConstraintRace
├── ConcurrentDelete
├── ConcurrentInsert
├── ConcurrentRelationshipChange
└── UnknownConcurrencyOutcome
```

---

# 27. Conflict ≠ error

Un conflicto concurrente puede ser:

```text
expected domain outcome
```

Ejemplo:

```text
two users attempt to reserve same seat
```

Uno debe perder.

No necesariamente representa:

```text
system defect
```

---

# 28. ConcurrencyConflictKind

```php
enum ConcurrencyConflictKind
{
    case OPTIMISTIC_VERSION;
    case LOCK_NOT_AVAILABLE;
    case LOCK_TIMEOUT;
    case DEADLOCK;
    case SERIALIZATION_FAILURE;
    case STALE_WRITE;
    case CONCURRENT_DELETE;
    case CONSTRAINT_RACE;
    case RELATIONSHIP_CONFLICT;
    case UNKNOWN;
}
```

---

# 29. Conflict classification

Pipeline:

```text
Driver Error
    ↓
Platform Error Mapper
    ↓
Execution Error
    ↓
Transaction Outcome
    ↓
Concurrency Conflict Classifier
    ↓
Canonical Conflict
```

---

# 30. SQLSTATE alone may be insufficient

Clasificación podrá utilizar:

```text
SQLSTATE
vendor code
driver exception
transaction state
operation type
lock intent
isolation
platform
```

---

# 31. Unknown remains unknown

Si VoltStack no puede clasificar con seguridad:

```text
UNKNOWN
```

No:

```text
probably deadlock
```

---

# 32. Anomalías concurrentes

El sistema deberá razonar al menos sobre:

```text
Dirty Read
Non-Repeatable Read
Phantom
Lost Update
Write Skew
Serialization Anomaly
Stale Write
Concurrent Delete
Constraint Race
```

---

# 33. Dirty read

```text
A writes X
A not committed
B reads X
A rolls back
```

B observó un valor nunca comprometido.

---

# 34. Non-repeatable read

```text
A reads X=10
B updates X=20 and commits
A reads X=20
```

---

# 35. Phantom

```text
A queries predicate P → 5 rows
B inserts matching row
A queries P → 6 rows
```

---

# 36. Lost update

```text
A reads V
B reads V
A writes V1
B writes V2
```

B puede sobrescribir A.

---

# 37. Write skew

Dos transacciones leen un conjunto compartido y modifican filas distintas, produciendo una violación de una invariante global.

Ejemplo:

```text
Doctor A on-call = true
Doctor B on-call = true

Invariant:
at least one doctor must remain on call
```

Concurrentemente:

```text
TX A sees B=true → A=false
TX B sees A=true → B=false
```

Resultado:

```text
A=false
B=false
```

---

# 38. Optimistic version may not stop write skew

Si cada transacción modifica una fila distinta:

```text
per-row versioning
```

puede no detectar el conflicto.

---

# 39. Cross-row invariants

Pueden requerir:

- SERIALIZABLE;
- explicit aggregate/root lock;
- constraint;
- advisory/domain coordination;
- redesign del modelo.

---

# 40. Database constraint remains essential

Ejemplo:

```text
UNIQUE(email)
```

sigue siendo la autoridad final contra:

```text
check email absent
→ concurrent inserts
```

---

# 41. Check-then-act race

Incorrecto:

```text
SELECT WHERE email = ?
if not exists:
    INSERT
```

Dos transactions pueden observar ausencia.

---

# 42. Correct architecture

Usar:

```text
database constraint
+
canonical constraint-race classification
```

y opcionalmente estrategias adicionales.

---

# 43. Isolation integration

El sistema consumirá:

```text
EffectiveTransactionIsolation
```

del documento 167.

No reinterpretará los niveles de aislamiento.

---

# 44. Isolation profile

Concurrency Resolver podrá consultar:

```text
IsolationSemantics
```

para conocer garantías disponibles.

---

# 45. Isolation does not replace domain rules

Incluso bajo aislamiento fuerte:

```text
business invariant
```

debe estar explícitamente modelada.

---

# 46. Optimistic integration

Del documento 172:

```text
Entity
+
Version Baseline
+
Conditional Write
```

produce detección de stale write.

---

# 47. Optimistic conflict

Ejemplo:

```sql
UPDATE orders
SET status = ?, version = 8
WHERE id = ?
AND version = 7
```

Affected rows:

```text
0
```

puede indicar conflicto.

Pero el sistema deberá distinguir, cuando sea necesario:

```text
version changed
row deleted
predicate changed
```

---

# 48. Pessimistic integration

Del documento 173:

```text
LockIntent
→ EffectiveLockPlan
→ DB Lock
```

---

# 49. Locking does not eliminate stale application state

Ejemplo:

```text
entity loaded yesterday
↓
transaction starts today
↓
lock row
```

El objeto PHP puede seguir conteniendo datos antiguos.

---

# 50. Lock + refresh

Cuando sea necesario:

```text
acquire lock
→ refresh authoritative state
→ validate invariant
→ mutate
```

---

# 51. IdentityMap interaction

IdentityMap garantiza:

```text
same EntityKey
→ same managed object
```

dentro del scope.

No garantiza:

```text
object always reflects latest committed DB state
```

---

# 52. IdentityMap ≠ isolation

Esta distinción será obligatoria:

```text
IdentityMap Consistency
≠
Database Transaction Isolation
```

---

# 53. Refresh semantics

`refresh()` deberá:

- usar transaction context;
- respetar lock strategy;
- respetar dirty state policy;
- no sobrescribir cambios locales silenciosamente.

---

# 54. UnitOfWork interaction

UnitOfWork conoce:

```text
NEW
MANAGED
DIRTY
REMOVED
```

y ChangeSets.

Concurrency Control conoce:

```text
whether those changes can safely synchronize
```

---

# 55. UnitOfWork ≠ concurrency manager

No deberá decidir directamente:

```text
retry transaction
acquire platform lock
change isolation
```

---

# 56. Persistence integration

Pipeline:

```text
UnitOfWork
    ↓
ChangeSet
    ↓
Persistence Planner
    ↓
Concurrency Decorator/Policy
    ↓
Conditional/Lock-aware Query Model
    ↓
Query Engine
```

---

# 57. Version predicates

Optimistic predicates deberán añadirse antes de compilation:

```text
semantic Query Model
```

No mediante SQL string rewriting.

---

# 58. Lock clauses

Pessimistic clauses pertenecerán al Query AST.

---

# 59. Concurrency annotations/metadata

ORM podrá declarar:

```php
#[Version]
private int $version;
```

y potencialmente policies:

```php
#[Concurrency(ConcurrencyStrategy::OPTIMISTIC)]
```

---

# 60. Metadata ≠ runtime state

Compiled metadata podrá compartirse.

Version baseline y lock state serán scoped.

---

# 61. Aggregate concurrency

Una entidad individual no siempre coincide con el boundary de consistencia.

Ejemplo:

```text
Order
├── OrderItem
├── Payment
└── Shipment
```

La invariante puede pertenecer al:

```text
Order Aggregate
```

---

# 62. AggregateConcurrencyPolicy

```php
interface AggregateConcurrencyPolicy
{
    public function resolve(
        AggregateDescriptor $aggregate,
        ConcurrencyContext $context,
    ): AggregateConcurrencyPlan;
}
```

---

# 63. Aggregate root lock

Una estrategia podrá utilizar:

```text
lock aggregate root
```

para serializar cambios sobre el aggregate.

---

# 64. Aggregate optimistic version

Otra estrategia:

```text
any aggregate mutation
→ increment root version
```

---

# 65. Aggregate version ≠ child version

Podrán coexistir:

```text
Order.version
OrderItem.version
```

si el modelo lo requiere.

---

# 66. Relationship concurrency

Las relaciones pueden sufrir:

```text
concurrent add
concurrent remove
concurrent clear
concurrent reorder
duplicate membership
orphan races
```

---

# 67. Many-to-many

Dos transactions:

```text
A adds User#1 → Role#5
B adds User#1 → Role#5
```

La protección final deberá incluir:

```text
unique membership constraint
```

aunque exista coordinación ORM.

---

# 68. Collection baseline

Una colección cargada:

```text
PARTIAL
```

no puede utilizarse para inferir con seguridad todos los cambios concurrentes.

---

# 69. Relationship clear

```text
collection.clear()
```

con una colección stale puede eliminar relaciones agregadas concurrentemente si se implementa ingenuamente.

---

# 70. Clear policy

Deberá distinguir:

```text
CLEAR_CURRENT_DATABASE_MEMBERSHIP
```

de:

```text
REMOVE_ONLY_MEMBERS_KNOWN_IN_BASELINE
```

cuando el API necesite ambas semánticas.

---

# 71. Relationship versioning

Podrá utilizarse:

- aggregate version;
- association entity version;
- parent version;
- DB constraints;
- locking.

No habrá una única estrategia universal.

---

# 72. Concurrent delete

Caso:

```text
A loads Order#10
B deletes Order#10
A updates Order#10
```

Resultado no deberá convertirse automáticamente en:

```text
successful update
```

---

# 73. Delete vs version conflict

Si affected rows = 0:

```text
row missing
```

y:

```text
version mismatch
```

pueden requerir clasificación adicional.

---

# 74. Conflict resolution

Una vez detectado un conflicto, las opciones serán conceptualmente:

```text
RETRY
ABORT
SURFACE
MERGE
IGNORE
CUSTOM
```

---

# 75. ConflictResolution

```php
enum ConflictResolution
{
    case RETRY;
    case ABORT;
    case SURFACE;
    case CUSTOM;
}
```

`MERGE` no será automático core por default.

---

# 76. Why no automatic merge

VoltStack no conoce si:

```text
name change
+
email change
```

son semánticamente combinables.

---

# 77. Last-write-wins

No será el default ORM para conflictos detectables.

```text
Last Write Wins
```

deberá ser una policy explícita cuando se quiera.

---

# 78. Retry integration

El documento 170 mantiene autoridad sobre replay.

Concurrency Control solamente podrá producir:

```text
RetryRecommendation
```

---

# 79. Retryable conflicts

Potencialmente:

```text
DEADLOCK
SERIALIZATION_FAILURE
LOCK_TIMEOUT
```

según policy.

---

# 80. Normally non-transparent conflicts

Frecuentemente:

```text
OPTIMISTIC_VERSION_CONFLICT
```

deberá llegar a aplicación si replay automático pudiera sobrescribir intención humana.

---

# 81. Example

Usuario abre formulario:

```text
version 7
```

otro usuario modifica:

```text
version 8
```

primer usuario envía formulario.

Hacer retry transparente contra versión 8 podría transformar:

```text
conflict detection
```

en:

```text
last-write-wins
```

---

# 82. Retry safety

Retry automático requiere:

```text
ReplaySafe
+
WithinBudget
+
RetryableFailure
+
TransactionBoundaryAvailable
```

---

# 83. External side effects

```text
send email
charge card
call external API
```

dentro de retryable transaction pueden duplicarse.

---

# 84. Side-effect architecture

Preferir:

```text
transaction
    ↓
persist business state
persist outbox event
    ↓
commit
    ↓
external processing
```

---

# 85. Unknown transaction outcome

Si commit outcome es:

```text
UNKNOWN
```

Concurrency Control no deberá:

```text
retry blindly
```

---

# 86. Why

La primera transacción puede haber sido committed.

Repetir podría duplicar la operación.

---

# 87. Unknown dominates retry

Regla:

```text
UNKNOWN_COMMIT
→
NO_AUTOMATIC_REPLAY
```

salvo protocolo explícitamente idempotente capaz de reconciliar.

---

# 88. Deadlock integration

Deadlock Handling produce:

```text
DeadlockDetected
```

y metadata diagnóstica.

Concurrency Control podrá considerar:

```text
retry recommendation
```

pero no reimplementar detector/retry.

---

# 89. Serialization failures

Serán conflictos distintos de deadlocks.

```text
SerializationFailure
≠
Deadlock
```

aunque ambos puedan ser retryable.

---

# 90. Lock timeout

```text
LockTimeout
≠
Deadlock
```

---

# 91. Lock not available

```text
NOWAIT failure
≠
LockTimeout
```

---

# 92. ConflictPolicy

```php
final readonly class ConflictPolicy
{
    public function __construct(
        public ConflictAction $optimistic,
        public ConflictAction $deadlock,
        public ConflictAction $serialization,
        public ConflictAction $lockTimeout,
        public ConflictAction $lockNotAvailable,
    ) {}
}
```

---

# 93. Explicit per-kind policy

Evitar:

```text
retry every concurrency error
```

---

# 94. Concurrency budgets

VoltStack deberá controlar cuánto recurso puede consumir una operación en conflictos.

```php
final readonly class ConcurrencyBudget
{
    public function __construct(
        public int $maxAttempts,
        public Duration $maxTotalDuration,
        public Duration $maxLockWait,
        public ?int $maxLockedTargets,
    ) {}
}
```

---

# 95. Budget dimensions

Podrán incluir:

```text
attempt budget
time budget
lock wait budget
lock target budget
backoff budget
connection occupancy budget
```

---

# 96. Retry budget

```text
AttemptNumber
≤
MaxAttempts
```

---

# 97. Time budget

```text
ElapsedOperationTime
≤
MaxTotalDuration
```

---

# 98. Lock budget

```text
RequestedWait
≤
RemainingConcurrencyBudget
```

cuando sea representable.

---

# 99. Budget exhaustion

Resultado:

```text
ConcurrencyBudgetExhaustedException
```

o outcome equivalente.

---

# 100. Budget exhaustion ≠ original conflict

Diagnostics deberán preservar:

```text
last conflict
+
budget exhaustion
```

---

# 101. Backoff

Retry podrá usar:

```text
fixed
linear
exponential
exponential+jitter
custom
```

según documento 170.

---

# 102. Jitter

Ayuda a evitar:

```text
synchronized retries
```

de múltiples workers.

---

# 103. Thundering herd

Caso:

```text
100 workers
↓
same hot row
↓
all fail
↓
all retry simultaneously
```

puede amplificar contention.

---

# 104. Backpressure

Concurrency Control deberá poder recomendar/requerir:

```text
slow down
queue
reject
defer
```

cuando el sistema alcance límites.

---

# 105. Database backpressure

No será implementado mediante:

```text
unbounded connection queue
```

---

# 106. Concurrency admission

Futuras policies podrán limitar:

```text
max concurrent expensive DB operations
max concurrent lock-heavy transactions
```

por execution domain.

---

# 107. Admission control ≠ database locking

Es resource governance.

---

# 108. Hot resource

VoltStack podrá identificar conceptualmente:

```text
HotResource
```

cuando múltiples conflictos correlacionen sobre el mismo fingerprint lógico.

---

# 109. HotResourceKey

No deberá exponer necesariamente IDs sensibles.

Podrá utilizar:

```text
hashed/bounded semantic resource fingerprint
```

---

# 110. Hot row diagnostics

Ejemplo:

```text
Resource:
    orders:{hashed-key}

Conflicts:
    147/min

Primary:
    lock timeout

Average attempts:
    2.8
```

---

# 111. Telemetry cardinality

Hot resource fingerprints deberán tener políticas estrictas de cardinalidad.

---

# 112. Starvation

Starvation ocurre cuando una operación:

```text
repeatedly fails to obtain progress
```

mientras otras continúan.

---

# 113. Starvation ≠ deadlock

En starvation:

```text
system continues making progress
```

pero una operación específica no.

---

# 114. Starvation detection

Puede inferirse mediante:

```text
same operation
+
repeated lock/retry conflicts
+
budget nearing exhaustion
```

---

# 115. Starvation confidence

Estados:

```text
SUSPECTED
LIKELY
OBSERVED
UNKNOWN
```

No se declarará certeza sin evidencia.

---

# 116. Livelock

Livelock:

```text
multiple operations actively retry/change behavior
but no useful progress
```

---

# 117. Example

```text
A conflicts → retry
B conflicts → retry
A conflicts → retry
B conflicts → retry
...
```

---

# 118. Livelock mitigation

Podrá incluir:

```text
jitter
larger backoff
attempt budget
admission control
strategy escalation
```

---

# 119. Strategy escalation

Una extensión podrá permitir:

```text
optimistic
↓ repeated conflicts
pessimistic
```

pero no será comportamiento core automático por default.

---

# 120. Why not automatic

Cambiar estrategia puede alterar:

- blocking;
- fairness;
- latency;
- deadlock probability;
- resource usage.

---

# 121. Explicit adaptive policy

Podrá existir:

```php
interface AdaptiveConcurrencyPolicy
{
    public function decide(
        ConcurrencyHistory $history,
        ConcurrencyContext $context,
    ): ConcurrencyStrategyDecision;
}
```

---

# 122. Fairness

VoltStack no prometerá:

```text
FIFO database lock acquisition
```

si el DBMS no lo garantiza.

---

# 123. Framework fairness

Sí podrá aplicar fairness en:

```text
local admission queues
retry scheduling
```

cuando corresponda.

---

# 124. DB fairness ≠ application fairness

Obligatoria distinción:

```text
Database Lock Scheduling
≠
VoltStack Retry Scheduling
```

---

# 125. Long transactions

Incrementan:

```text
lock duration
conflict window
connection occupancy
deadlock opportunities
```

---

# 126. Transaction duration diagnostics

Concurrency subsystem deberá integrarse con transaction telemetry para detectar:

```text
long transaction
+
high contention
```

---

# 127. Transaction size

Gran cantidad de statements también puede incrementar conflicto.

---

# 128. Concurrency-friendly transaction principle

Preferir:

```text
short
bounded
deterministic
database-focused
```

transactions.

---

# 129. No network calls while locked

Developer diagnostics podrán advertir:

```text
DB lock held
↓
external HTTP call
```

cuando sea observable.

---

# 130. Query ordering

Deterministic resource ordering reduce deadlock risk.

Ejemplo:

```text
sort account IDs
↓
lock in ascending order
```

---

# 131. Ordering coordinator

Podrá existir:

```php
interface ConcurrencyOrderingPolicy
{
    public function order(
        ConcurrencyResourceSet $resources,
    ): ConcurrencyResourceSet;
}
```

---

# 132. Resource identity

```php
final readonly class ConcurrencyResourceKey
{
    public function __construct(
        public ResourceNamespace $namespace,
        public string $fingerprint,
        public ExecutionDomain $domain,
    ) {}
}
```

---

# 133. ResourceKey ≠ EntityKey

Un concurrency resource puede representar:

```text
entity
aggregate
relationship
predicate
business slot
```

---

# 134. Entity resource

```text
entity:Order:100
```

---

# 135. Aggregate resource

```text
aggregate:Order:100
```

puede incluir múltiples persistence resources.

---

# 136. Predicate resource

Conceptualmente:

```text
available-room:hotel=10,date=...
```

aunque no necesariamente exista como row única.

---

# 137. Predicate locking portability

VoltStack no fingirá que puede transformar cualquier logical predicate resource en un DB lock portable.

---

# 138. Resource mapping

```text
ConcurrencyResource
    ↓
Strategy Resolver
    ↓
DB representation
```

puede resultar:

```text
row lock
version check
constraint
serializable predicate semantics
unsupported
```

---

# 139. Unsupported strategy

Si una garantía requerida no puede representarse:

```text
ConcurrencyRequirementUnsupportedException
```

---

# 140. No silent weakening

Regla:

```text
Required Guarantee
>
Available Guarantee
→
FAIL
```

No:

```text
continue anyway
```

---

# 141. Stronger strategy

Tampoco deberá aplicarse automáticamente una estrategia “más fuerte” cuando pueda alterar comportamiento significativamente.

---

# 142. Strategy compatibility

Resultado:

```php
enum ConcurrencyCompatibility
{
    case EXACT;
    case STRONGER_WITH_CONSENT;
    case WITH_LIMITATIONS;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 143. Unknown capabilities

```text
UNKNOWN
≠
SUPPORTED
```

---

# 144. Read/write routing

Dentro de transaction:

```text
Concurrency-sensitive operation
→ pinned authoritative connection
```

según Transaction Context.

---

# 145. Replica limitations

Una read replica puede:

- tener lag;
- no compartir transaction snapshot;
- no soportar locking;
- observar estado diferente.

Por tanto no se utilizará para resolver arbitrariamente decisiones de concurrencia.

---

# 146. Read-your-writes

Block 16 definirá routing detallado.

Concurrency Control únicamente expresará requirements.

---

# 147. Query cache interaction

Una concurrency-sensitive read puede requerir:

```text
bypass result cache
```

---

# 148. Why

Un cache hit no adquiere:

```text
DB lock
```

ni necesariamente representa:

```text
current transaction snapshot
```

---

# 149. Cache ≠ concurrency authority

```text
Query Cache
≠
Database Isolation
≠
Database Lock
```

---

# 150. Entity cache

Una entity obtenida desde second-level cache tampoco deberá considerarse automáticamente adecuada para una decisión crítica de concurrencia.

---

# 151. Refresh for critical decision

Policies podrán exigir:

```text
authoritative database read
```

antes de validar ciertas invariantes.

---

# 152. Validation interaction

Application validation:

```text
balance >= amount
```

puede quedar obsoleta inmediatamente bajo concurrencia.

---

# 153. Validation ≠ concurrency guarantee

La validación de negocio deberá ejecutarse bajo el boundary adecuado cuando dependa de estado concurrente.

---

# 154. Authorization interaction

Authorization puede depender de DB state.

Si ese estado puede cambiar concurrentemente:

```text
authorize
→ later mutate
```

puede requerir revalidación dentro de la transacción.

---

# 155. TOCTOU

VoltStack deberá reconocer:

```text
Time Of Check
≠
Time Of Use
```

como riesgo arquitectónico.

---

# 156. Security-sensitive concurrency

Operaciones como:

```text
spend credit
consume token
redeem coupon
change ownership
```

deberán permitir policies de concurrency explícitas.

---

# 157. Idempotency

Idempotency puede complementar retry.

```text
Retry Safety
+
Idempotency Key
```

puede reducir duplicados.

---

# 158. Idempotency ≠ transaction isolation

Son mecanismos diferentes.

---

# 159. Idempotency record

Cuando se implemente a nivel aplicación:

```text
IdempotencyKey
→ unique database constraint
→ stored outcome
```

puede convertirse en una primitive de concurrencia.

---

# 160. Distributed systems boundary

Una transacción VoltStack Database normalmente cubre:

```text
one physical database transaction/resource
```

---

# 161. Cross-service concurrency

No será resuelta mediante row locks locales.

Requerirá patrones como:

```text
Outbox
Saga
Idempotency
Consensus service
Distributed lock
```

fuera del core transaction model.

---

# 162. Distributed lock

Si en el futuro existe:

```text
VoltStack Distributed Lock System
```

deberá permanecer separado.

---

# 163. Database advisory locks

También serán una capability/extensión separada.

No deberán confundirse con:

```text
row-level pessimistic locks
```

---

# 164. Transaction-local state

Mutable:

```text
conflict history
attempt information
known lock acquisitions
optimistic baselines
budget usage
```

deberá ser scoped.

---

# 165. Persistent runtime

Compartible entre workers:

```text
compiled policies
immutable metadata
capability profiles
strategy definitions
classifiers
```

---

# 166. Not shareable

```text
current transaction
current locks
current attempts
current entities
current conflicts
current budget counters
```

---

# 167. FrankenPHP

Cada request deberá obtener:

```text
new ConcurrencyContext
```

cuando corresponda.

---

# 168. RoadRunner

Cada operation/job deberá limpiar:

```text
ConcurrencyContext
ConflictHistory
LockRegistry
RetryState
```

---

# 169. OpenSwoole

Todo estado mutable deberá ser:

```text
coroutine/fiber scoped
```

---

# 170. Static current concurrency context

Prohibido:

```php
Concurrency::$current
```

---

# 171. Context resolver

Preferir:

```php
interface ConcurrencyContextResolver
{
    public function current(): ?ConcurrencyContext;
}
```

respaldado por runtime-scoped storage.

---

# 172. Context storage

Deberá integrarse con:

```text
Runtime Execution Scope
```

igual que Transaction Context.

---

# 173. Parallelism inside one transaction

Por default:

```text
single execution lane
```

---

# 174. Why

Una única conexión puede no soportar operaciones paralelas y el orden puede afectar:

```text
locks
deadlocks
result ordering
transaction state
```

---

# 175. Explicit parallel transaction operations

Solo cuando:

```text
Driver
+
Protocol
+
Connection
+
Transaction
+
Concurrency Policy
```

lo soporten explícitamente.

---

# 176. Diagnostics

API conceptual:

```php
DB::concurrency()->explain($operation);
```

---

# 177. Diagnostic output

```text
CONCURRENCY PLAN

Operation:
    update-order

Strategy:
    HYBRID

Isolation:
    READ_COMMITTED

Optimistic:
    Order.version

Pessimistic:
    Order root / EXCLUSIVE

Lock Wait:
    NO_WAIT

Retry:
    deadlock: yes
    serialization: yes
    optimistic conflict: no

Budget:
    attempts: 3
    duration: 2s

Execution Domain:
    primary / writer

Capability:
    EXACT
```

---

# 178. Conflict diagnostics

```text
CONCURRENCY CONFLICT

Kind:
    OPTIMISTIC_VERSION

Entity:
    Order

Expected Version:
    7

Observed:
    conflict

Transaction:
    rolled back

Retry Recommendation:
    SURFACE

Certainty:
    HIGH
```

Valores sensibles deberán sanitizarse.

---

# 179. Deadlock diagnostic integration

Podrá mostrar:

```text
Deadlock:
    detected by platform

Attempt:
    2/3

Backoff:
    38ms

Next Action:
    RETRY
```

sin duplicar toda la lógica del documento 171.

---

# 180. Telemetry architecture

Eventos conceptuales:

```text
db.concurrency.operation
db.concurrency.conflict
db.concurrency.retry
db.concurrency.budget_exhausted
db.concurrency.starvation
db.concurrency.strategy
```

---

# 181. Metrics

Ejemplos:

```text
db.concurrency.conflicts
db.concurrency.retries
db.concurrency.attempts
db.concurrency.budget_exhaustions
db.concurrency.lock_wait
```

---

# 182. Labels

Permitidos:

```text
strategy
conflict_kind
platform
outcome
retry_decision
```

con cardinalidad controlada.

---

# 183. Forbidden high-cardinality labels

No usar:

```text
entity ID
transaction ID
tenant ID
raw SQL
parameter values
email
user identifier
```

como labels de métricas.

---

# 184. Tracing

Span attributes:

```text
db.concurrency.strategy
db.concurrency.attempt
db.concurrency.conflict
db.concurrency.retry
```

---

# 185. Conflict events

Telemetry deberá distinguir:

```text
conflict detected
```

de:

```text
operation failed
```

porque un conflicto puede ser manejado exitosamente mediante retry.

---

# 186. Final operation outcome

Ejemplo:

```text
attempt 1 → deadlock
attempt 2 → success
```

Debe producir:

```text
ConflictCount = 1
RetryCount = 1
FinalOutcome = SUCCESS
```

---

# 187. Testing architecture

Suite propuesta:

```text
ConcurrencyStrategyTests
ConcurrencyConflictClassifierTests
ConcurrencyIsolationIntegrationTests
ConcurrencyOptimisticIntegrationTests
ConcurrencyPessimisticIntegrationTests
ConcurrencyRetryIntegrationTests
ConcurrencyDeadlockIntegrationTests
ConcurrencySerializationTests
ConcurrencyLostUpdateTests
ConcurrencyWriteSkewTests
ConcurrencyConstraintRaceTests
ConcurrencyAggregateTests
ConcurrencyRelationshipTests
ConcurrencyBudgetTests
ConcurrencyStarvationTests
ConcurrencyLivelockTests
ConcurrencyBackpressureTests
ConcurrencyCacheInteractionTests
ConcurrencyPersistentRuntimeTests
ConcurrencyCoroutineIsolationTests
ConcurrencyPlatformConformanceTests
```

---

# 188. Lost update test

Dos conexiones:

```text
A read
B read
A update
B update
```

Verificar comportamiento bajo:

```text
no strategy
optimistic
pessimistic
serializable
```

---

# 189. Optimistic test

```text
A version 10
B version 10

A writes version 11
B conditional write version 10
```

Esperado:

```text
B → OptimisticConflict
```

---

# 190. Pessimistic test

A adquiere exclusive lock.

B solicita lock incompatible.

Verificar:

```text
WAIT
NOWAIT
TIMEOUT
SKIP_LOCKED
```

según capability.

---

# 191. Serializable test

Ejecutar workload susceptible a anomaly.

Verificar:

```text
success
or canonical serialization failure
```

sin asumir comportamiento idéntico entre plataformas.

---

# 192. Write skew test

Utilizar dos conexiones y una invariante multi-row.

Comprobar estrategias que sí/no previenen el problema.

---

# 193. Constraint race test

Dos transactions intentan insertar mismo unique key.

Esperado:

```text
one success
one canonical constraint conflict
```

---

# 194. Concurrent delete test

A carga entity.

B elimina.

A intenta modificar.

Debe producir outcome explícito.

---

# 195. Retry test

```text
attempt 1 → retryable conflict
attempt 2 → success
```

Verificar:

- OperationId estable;
- TransactionId diferente;
- nuevo TransactionContext;
- nuevo ConcurrencyContext attempt state;
- cleanup correcto.

---

# 196. Unknown commit test

Simular:

```text
COMMIT sent
connection lost
```

Verificar:

```text
UNKNOWN
+
no blind retry
```

---

# 197. Budget test

Forzar conflictos hasta:

```text
MaxAttempts
```

Esperado:

```text
ConcurrencyBudgetExhausted
```

con último conflicto preservado.

---

# 198. Starvation test

Una operación pierde repetidamente mientras otras progresan.

Verificar diagnostic:

```text
SUSPECTED/LIKELY
```

según evidencia.

---

# 199. Livelock test

Varias operaciones reintentan sin progreso útil.

Verificar:

- bounded retries;
- jitter;
- eventual budget exhaustion;
- telemetry.

---

# 200. Persistent worker test

Request A:

```text
conflict history populated
```

Request B:

```text
must begin empty
```

---

# 201. Coroutine test

Dos coroutines:

```text
ConcurrencyContext(A)
∩ mutable state
ConcurrencyContext(B)
=
∅
```

---

# 202. Platform conformance

Cada plataforma deberá probar:

```text
isolation semantics
lock capabilities
error classification
serialization failure mapping
deadlock mapping
lock timeout mapping
constraint race mapping
transaction-abort behavior
```

---

# 203. Proposed directory structure

```text
src/Quantum/Database/Concurrency/
│
├── ConcurrencyControlSystem.php
├── ConcurrencyContext.php
├── ConcurrencyContextResolver.php
├── ConcurrencyContextStorage.php
├── ConcurrencyOperationId.php
│
├── Strategy/
│   ├── ConcurrencyStrategy.php
│   ├── ConcurrencyStrategyResolver.php
│   ├── ConcurrencyStrategyDecision.php
│   ├── ConcurrencyRequirements.php
│   ├── ConcurrencyCompatibility.php
│   ├── AdaptiveConcurrencyPolicy.php
│   └── CustomConcurrencyStrategy.php
│
├── Conflict/
│   ├── ConcurrencyConflict.php
│   ├── ConcurrencyConflictKind.php
│   ├── ConcurrencyConflictClassifier.php
│   ├── ConflictPolicy.php
│   ├── ConflictAction.php
│   ├── ConflictResolution.php
│   └── RetryRecommendation.php
│
├── Resource/
│   ├── ConcurrencyResource.php
│   ├── ConcurrencyResourceKey.php
│   ├── ConcurrencyResourceSet.php
│   ├── ConcurrencyOrderingPolicy.php
│   ├── AggregateConcurrencyPolicy.php
│   └── AggregateConcurrencyPlan.php
│
├── Budget/
│   ├── ConcurrencyBudget.php
│   ├── ConcurrencyBudgetState.php
│   ├── ConcurrencyBudgetPolicy.php
│   └── ConcurrencyBudgetGuard.php
│
├── Governance/
│   ├── ConcurrencyAdmissionPolicy.php
│   ├── ConcurrencyBackpressurePolicy.php
│   ├── StarvationDetector.php
│   ├── LivelockDetector.php
│   └── HotResourceDetector.php
│
├── Optimistic/
│   └── ...
│
├── Pessimistic/
│   └── ...
│
├── Diagnostics/
│   ├── ConcurrencyInspector.php
│   ├── ConcurrencyPlanReport.php
│   ├── ConcurrencyConflictReport.php
│   └── ConcurrencyHealthReport.php
│
├── Telemetry/
│   ├── ConcurrencyTelemetry.php
│   └── ConcurrencyTelemetrySanitizer.php
│
└── Exception/
    ├── DatabaseConcurrencyException.php
    ├── ConcurrencyConflictException.php
    ├── ConcurrencyRequirementUnsupportedException.php
    ├── ConcurrencyBudgetExhaustedException.php
    ├── ConcurrencyContextException.php
    └── ConcurrencyInvariantViolationException.php
```

---

# 204. Dependency architecture

```text
                   ORM
                    │
                    ▼
               UnitOfWork
                    │
                    ▼
             Persistence Engine
                    │
                    ▼
         Concurrency Control System
          ┌─────────┼──────────┐
          ▼         ▼          ▼
     Optimistic  Isolation  Pessimistic
          │         │          │
          └─────────┼──────────┘
                    ▼
                Query Engine
                    │
                    ▼
             Execution Engine
                    │
                    ▼
           Transaction Manager
                    │
                    ▼
          Connection / Platform
                    │
                    ▼
                 Driver
```

Esto representa colaboración conceptual.

La dirección real de dependencias deberá evitar ciclos mediante contratos.

---

# 205. Dependency rule

Concurrency Control podrá consumir contratos de:

```text
Transaction
Platform Capabilities
Query Model
ORM Metadata
Persistence
Telemetry
Runtime Scope
```

pero no deberá depender de:

```text
HTTP implementation
Queue implementation
Redis
business application classes
PDO directly
global runtime state
```

---

# 206. Architectural invariants

## DB-CC-001
Concurrency Control será una arquitectura coordinadora, no un mecanismo único.

## DB-CC-002
Transaction isolation no será equivalente a concurrency control completo.

## DB-CC-003
Optimistic locking no será equivalente a concurrency control completo.

## DB-CC-004
Pessimistic locking no será equivalente a concurrency control completo.

## DB-CC-005
Retry no será equivalente a concurrency control completo.

## DB-CC-006
Database constraints seguirán siendo necesarias para invariantes representables por DB.

## DB-CC-007
ConcurrencyContext será operation-scoped.

## DB-CC-008
ConcurrencyContext no será global.

## DB-CC-009
ConcurrencyOperationId será distinto de TransactionId.

## DB-CC-010
Retries conservarán logical OperationId.

## DB-CC-011
Cada retry utilizará nueva transaction física/lógica cuando corresponda.

## DB-CC-012
Concurrency strategy será explícita.

## DB-CC-013
PLATFORM_DEFAULT no implicará universal safety.

## DB-CC-014
OPTIMISTIC utilizará conflict detection.

## DB-CC-015
PESSIMISTIC utilizará explicit DB protection cuando sea soportada.

## DB-CC-016
SERIALIZABLE podrá producir serialization failures.

## DB-CC-017
HYBRID podrá combinar mecanismos.

## DB-CC-018
CUSTOM deberá respetar core invariants.

## DB-CC-019
No existirá una estrategia universalmente óptima.

## DB-CC-020
Conflict taxonomy será canónica.

## DB-CC-021
Deadlock será distinto de serialization failure.

## DB-CC-022
Deadlock será distinto de lock timeout.

## DB-CC-023
Lock timeout será distinto de lock-not-available.

## DB-CC-024
Optimistic conflict será distinto de deadlock.

## DB-CC-025
Unknown concurrency outcome permanecerá UNKNOWN.

## DB-CC-026
Un conflicto concurrente no será necesariamente un system defect.

## DB-CC-027
Driver errors serán normalizados antes de policy decisions.

## DB-CC-028
SQLSTATE no será siempre evidencia suficiente.

## DB-CC-029
Dirty read será modelado separadamente.

## DB-CC-030
Non-repeatable read será modelado separadamente.

## DB-CC-031
Phantom será modelado separadamente.

## DB-CC-032
Lost update será modelado separadamente.

## DB-CC-033
Write skew será modelado separadamente.

## DB-CC-034
Stale write será modelado separadamente.

## DB-CC-035
Per-row optimistic locking no garantizará prevención de write skew.

## DB-CC-036
Cross-row invariants requerirán estrategia explícita.

## DB-CC-037
Check-then-act no sustituirá DB constraint.

## DB-CC-038
Unique constraint será autoridad final para uniqueness.

## DB-CC-039
Constraint races serán outcomes concurrentes clasificables.

## DB-CC-040
Effective isolation será consumido, no reinterpretado.

## DB-CC-041
Concurrency Control no modificará isolation de una transaction ACTIVE.

## DB-CC-042
Optimistic version predicates pertenecerán al semantic query model.

## DB-CC-043
Version predicates no se añadirán mediante raw SQL rewriting.

## DB-CC-044
Pessimistic LockIntent pertenecerá al Query AST.

## DB-CC-045
Locking no garantizará fresh PHP object state.

## DB-CC-046
Lock + refresh podrá ser requerido.

## DB-CC-047
IdentityMap no será database isolation.

## DB-CC-048
IdentityMap no garantizará latest committed state.

## DB-CC-049
Refresh no sobrescribirá dirty state silenciosamente.

## DB-CC-050
UnitOfWork no será Concurrency Manager.

## DB-CC-051
UnitOfWork no decidirá retries.

## DB-CC-052
UnitOfWork no cambiará isolation.

## DB-CC-053
UnitOfWork no adquirirá locks mediante Driver directamente.

## DB-CC-054
Persistence Planner podrá incorporar concurrency requirements.

## DB-CC-055
Compiled concurrency metadata podrá compartirse entre workers.

## DB-CC-056
Runtime concurrency state no podrá compartirse entre requests.

## DB-CC-057
Aggregate boundary podrá diferir de entity boundary.

## DB-CC-058
Aggregate root locking será explicit policy.

## DB-CC-059
Aggregate versioning será explicit policy.

## DB-CC-060
Child version no será automáticamente aggregate version.

## DB-CC-061
Relationship changes tendrán concurrency semantics explícitas.

## DB-CC-062
Many-to-many duplicate prevention requerirá DB constraint cuando aplique.

## DB-CC-063
PARTIAL collection no será authoritative complete baseline.

## DB-CC-064
Relationship clear no asumirá ausencia de concurrent additions.

## DB-CC-065
Concurrent delete será outcome explícito.

## DB-CC-066
Affected rows zero no significará automáticamente version conflict.

## DB-CC-067
Conflict resolution será explícita.

## DB-CC-068
Automatic merge no será core default.

## DB-CC-069
Last-write-wins no será default para conflictos detectables.

## DB-CC-070
Retry System mantendrá autoridad sobre replay.

## DB-CC-071
Concurrency Control podrá emitir RetryRecommendation.

## DB-CC-072
Retryable no significará retry obligatorio.

## DB-CC-073
Optimistic user-edit conflict no será transparent retry por default.

## DB-CC-074
Retry requerirá replay safety.

## DB-CC-075
Retry requerirá budget disponible.

## DB-CC-076
Retry requerirá transaction boundary reproducible.

## DB-CC-077
External side effects deberán considerarse antes de retry.

## DB-CC-078
Outbox será preferible para side effects post-commit.

## DB-CC-079
UNKNOWN commit outcome no tendrá blind automatic retry.

## DB-CC-080
Unknown transaction outcome prevalecerá sobre convenience retry.

## DB-CC-081
Deadlock Handling mantendrá autoridad de clasificación especializada.

## DB-CC-082
Serialization failure tendrá clasificación independiente.

## DB-CC-083
Conflict policies podrán variar por conflict kind.

## DB-CC-084
No existirá “retry every concurrency exception” como default.

## DB-CC-085
ConcurrencyBudget será bounded.

## DB-CC-086
Retry attempts estarán limitados.

## DB-CC-087
Total retry duration estará limitada.

## DB-CC-088
Lock wait podrá estar limitado.

## DB-CC-089
Lock target count podrá estar limitado.

## DB-CC-090
Budget exhaustion conservará last conflict.

## DB-CC-091
Backoff podrá utilizar jitter.

## DB-CC-092
Retries sincronizados serán considerados thundering-herd risk.

## DB-CC-093
Backpressure no utilizará unbounded queues.

## DB-CC-094
Admission control será distinto de DB locking.

## DB-CC-095
Hot-resource detection será observational.

## DB-CC-096
Hot-resource telemetry tendrá bounded cardinality.

## DB-CC-097
Starvation será distinta de deadlock.

## DB-CC-098
Starvation diagnostics expresarán confidence.

## DB-CC-099
Livelock será distinto de deadlock.

## DB-CC-100
Livelock será limitado mediante budgets.

## DB-CC-101
Adaptive strategy escalation no será default automático.

## DB-CC-102
Adaptive policy deberá ser explícita.

## DB-CC-103
VoltStack no prometerá FIFO DB locks.

## DB-CC-104
Database fairness será distinta de framework retry fairness.

## DB-CC-105
Long transactions serán contention risk.

## DB-CC-106
Long transaction diagnostics podrán integrarse con concurrency telemetry.

## DB-CC-107
External I/O while holding locks podrá producir warning.

## DB-CC-108
Deterministic resource ordering será recomendado.

## DB-CC-109
Resource ordering no podrá alterar business semantics.

## DB-CC-110
ConcurrencyResourceKey será distinto de EntityKey.

## DB-CC-111
Concurrency resource podrá representar aggregate.

## DB-CC-112
Concurrency resource podrá representar relationship.

## DB-CC-113
Concurrency resource podrá representar predicate.

## DB-CC-114
Predicate resource no implicará portable predicate lock.

## DB-CC-115
Strategy Resolver mapeará requirements a capabilities reales.

## DB-CC-116
Unsupported required guarantee deberá fallar.

## DB-CC-117
Required guarantee no será debilitada silenciosamente.

## DB-CC-118
UNKNOWN capability no será SUPPORTED.

## DB-CC-119
Stronger strategy podrá requerir consentimiento.

## DB-CC-120
Concurrency-sensitive transaction operation utilizará pinned connection.

## DB-CC-121
Replica no será concurrency authority por default.

## DB-CC-122
Concurrency-sensitive read podrá bypass cache.

## DB-CC-123
Query Cache no será DB lock.

## DB-CC-124
Entity Cache no será DB isolation.

## DB-CC-125
Critical validation podrá requerir authoritative read.

## DB-CC-126
Application validation no será concurrency guarantee.

## DB-CC-127
Authorization checks podrán requerir transactional revalidation.

## DB-CC-128
TOCTOU será tratado como riesgo explícito.

## DB-CC-129
Idempotency será distinta de isolation.

## DB-CC-130
Idempotency podrá complementar retry.

## DB-CC-131
Single DB transaction no será distributed transaction.

## DB-CC-132
Cross-service concurrency permanecerá fuera del row-lock model.

## DB-CC-133
Distributed locks permanecerán subsistema separado.

## DB-CC-134
Advisory locks permanecerán capability separada.

## DB-CC-135
Mutable conflict history será scoped.

## DB-CC-136
Mutable retry state será scoped.

## DB-CC-137
Mutable lock registry será scoped.

## DB-CC-138
Mutable budget counters serán scoped.

## DB-CC-139
Static current concurrency state estará prohibido.

## DB-CC-140
FrankenPHP requests estarán aisladas.

## DB-CC-141
RoadRunner operations estarán aisladas.

## DB-CC-142
OpenSwoole coroutines estarán aisladas.

## DB-CC-143
Runtime cleanup eliminará concurrency state.

## DB-CC-144
Parallel transaction DB operations estarán deshabilitadas por default.

## DB-CC-145
Parallel operations requerirán capabilities explícitas.

## DB-CC-146
Diagnostics serán observational.

## DB-CC-147
Diagnostics no modificarán strategy.

## DB-CC-148
Telemetry no decidirá retries.

## DB-CC-149
Telemetry distinguirá conflict outcome de final operation outcome.

## DB-CC-150
Metrics evitarán high-cardinality identifiers.

## DB-CC-151
Raw SQL no será metric label.

## DB-CC-152
Bound parameters no serán metric labels.

## DB-CC-153
Testing utilizará múltiples conexiones físicas cuando sea necesario.

## DB-CC-154
Testing incluirá lost update.

## DB-CC-155
Testing incluirá write skew.

## DB-CC-156
Testing incluirá deadlock.

## DB-CC-157
Testing incluirá serialization failure.

## DB-CC-158
Testing incluirá lock timeout.

## DB-CC-159
Testing incluirá optimistic conflict.

## DB-CC-160
Testing incluirá constraint race.

## DB-CC-161
Testing incluirá concurrent delete.

## DB-CC-162
Testing incluirá retry budget exhaustion.

## DB-CC-163
Testing incluirá unknown commit outcome.

## DB-CC-164
Testing incluirá starvation diagnostics.

## DB-CC-165
Testing incluirá persistent workers.

## DB-CC-166
Testing incluirá coroutine isolation.

## DB-CC-167
Cada Platform deberá pasar concurrency conformance tests.

## DB-CC-168
Platform-specific behavior no se filtrará como universal semantics.

## DB-CC-169
Version-specific capabilities deberán ser explícitas.

## DB-CC-170
Conflict certainty deberá preservarse.

## DB-CC-171
UNKNOWN nunca será convertido en SUCCESS.

## DB-CC-172
UNKNOWN nunca será convertido en ROLLED_BACK sin evidencia.

## DB-CC-173
UNKNOWN nunca será convertido en COMMITTED sin evidencia.

## DB-CC-174
DatabaseRollback no implicará PHP object graph rewind.

## DB-CC-175
Retry deberá comenzar con state apropiadamente reconciliado/reset.

## DB-CC-176
Stale entities no serán reutilizadas ciegamente entre retry attempts.

## DB-CC-177
IdentityMap deberá ser reconciliado/cleared según retry policy.

## DB-CC-178
Transaction-local lock state no sobrevivirá al attempt.

## DB-CC-179
Attempt-local snapshots no sobrevivirán incorrectamente al replay.

## DB-CC-180
Concurrency Control nunca fabricará una garantía que la plataforma no pueda demostrar.

---

# 207. Retry and ORM state

Un punto crítico será el estado ORM entre attempts.

Ejemplo:

```text
Attempt 1
    ↓
EntityManager
IdentityMap
Snapshots
ChangeSets
    ↓
deadlock
    ↓
rollback
```

No deberá hacerse simplemente:

```text
retry same flush with same mutable state
```

sin reconciliación.

---

# 208. Retry state policy

Estrategias posibles:

```text
CLEAR_AND_RELOAD
RECREATE_ENTITY_MANAGER
APPLICATION_REPLAY
CUSTOM
```

---

# 209. Preferred safe model

Para transaction callback retry:

```text
Operation Callback
    ↓
Attempt 1
    ├── scoped EntityManager
    ├── scoped UoW
    └── scoped Transaction
        ↓
      rollback
        ↓
dispose attempt state
        ↓
Attempt 2
    ├── new Transaction
    └── fresh/reconciled persistence state
```

---

# 210. Object graph rewind

Continúa la regla:

```text
DatabaseRollback
≠
AutomaticObjectGraphRewind
```

---

# 211. Stale object hazard

Después de rollback:

```text
PHP object
```

puede contener cambios que nunca fueron committed.

Por ello no deberá reutilizarse ciegamente.

---

# 212. Retry callback design

Preferir:

```php
DB::transaction(function () use ($orderId) {
    $order = Order::findOrFail($orderId);

    // perform operation
});
```

frente a capturar una entidad mutable externa:

```php
$order = Order::findOrFail($id);

DB::transaction(function () use ($order) {
    // retrying this same object may be unsafe
});
```

---

# 213. Retry capture diagnostics

Developer tooling podrá advertir cuando una retryable transaction closure capture managed mutable entities de un scope externo.

---

# 214. Concurrency health

Podrá existir:

```php
enum ConcurrencyHealth
{
    case HEALTHY;
    case CONTENDED;
    case DEGRADED;
    case STARVING;
    case UNCERTAIN;
}
```

---

# 215. Health ≠ transaction state

```text
TransactionState = ACTIVE
ConcurrencyHealth = CONTENDED
```

es perfectamente válido.

---

# 216. Health model

Conceptualmente:

```text
HEALTHY
   ↓ conflicts
CONTENDED
   ↓ repeated conflicts
DEGRADED
   ↓ no progress
STARVING
```

Mientras incertidumbre:

```text
UNCERTAIN
```

será una dimensión especial.

---

# 217. Health is diagnostic

No deberá convertirse automáticamente en strategy mutation.

---

# 218. Concurrency plan

Resultado final de resolución:

```php
final readonly class ConcurrencyPlan
{
    public function __construct(
        public ConcurrencyStrategy $strategy,
        public EffectiveTransactionIsolation $isolation,
        public ?OptimisticConcurrencyPlan $optimistic,
        public ?EffectiveLockPlan $pessimistic,
        public ConflictPolicy $conflicts,
        public ConcurrencyBudget $budget,
    ) {}
}
```

---

# 219. Plan immutability

Una vez iniciada la parte relevante de una transaction:

```text
effective concurrency guarantees
```

no deberán mutarse arbitrariamente.

---

# 220. Plan fingerprint

Podrá calcularse:

```text
ConcurrencyPlanFingerprint
=
hash(
    strategy
    + isolation
    + lock semantics
    + optimistic semantics
    + conflict policy generation
)
```

para diagnostics/cache de planes seguros.

---

# 221. Plan cache

Solo podrá almacenar:

```text
immutable compiled concurrency plans
```

No:

```text
TransactionContext
LockHandle
Entity
Version baseline
ConflictHistory
```

---

# 222. Failure model

```text
Concurrency Operation
        │
        ▼
      Attempt
        │
        ├── SUCCESS
        │
        ├── CONFLICT
        │      │
        │      ├── RETRY
        │      ├── SURFACE
        │      └── ABORT
        │
        ├── FAILURE
        │
        └── UNKNOWN
```

---

# 223. Success

Solo cuando el operation boundary relevante pueda considerarse completado con suficiente certeza.

---

# 224. Conflict

Existe evidencia de competencia concurrente.

---

# 225. Failure

Error no necesariamente relacionado con concurrencia.

---

# 226. Unknown

No existe evidencia suficiente para afirmar outcome final.

---

# 227. Conflict certainty

```php
enum ConflictCertainty
{
    case CERTAIN;
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

Policy automática deberá utilizar únicamente niveles apropiados.

---

# 228. No speculative retry

Un error de red clasificado vagamente como:

```text
maybe deadlock
```

no deberá activar retry de deadlock.

---

# 229. Final concurrency formula

Para una operación `O`:

```text
ConcurrencyPlan(O)
=
Resolve(
    Requirements(O),
    DomainInvariants(O),
    PlatformCapabilities,
    TransactionDefinition,
    IsolationSemantics,
    ResourceBudget
)
```

---

# 230. Conflict handling formula

```text
Decision
=
ConflictPolicy(
    ConflictKind,
    ConflictCertainty,
    TransactionOutcome,
    ReplaySafety,
    RemainingBudget
)
```

---

# 231. Automatic retry formula

```text
AutoRetryAllowed
⇔
RetryableConflict
∧ ReplaySafe
∧ OutcomeKnownRollback
∧ BudgetAvailable
∧ BoundaryReplayable
```

---

# 232. Unknown formula

```text
TransactionOutcome = UNKNOWN
⇒
BlindAutomaticRetry = FORBIDDEN
```

---

# 233. Optimistic formula

```text
OptimisticSuccess
⇔
ExpectedVersion
=
PersistedVersionAtConditionalWrite
```

según la estrategia de versión utilizada.

---

# 234. Pessimistic formula

```text
PessimisticProtection
⇔
DatabaseConfirmedLock
∧ CorrectTransaction
∧ CorrectConnection
∧ LockStillWithinTransactionLifetime
```

---

# 235. Aggregate formula

```text
AggregateConcurrencySafety
≠
Σ(EntityConcurrencySafety)
```

porque invariantes pueden cruzar múltiples entidades.

---

# 236. Relationship formula

```text
RelationshipBaseline
+
ConcurrentMutation
```

requiere strategy explícita; una colección PHP no representa automáticamente la verdad actual de DB.

---

# 237. Architecture overview

```text
                     Application
                         │
                         ▼
              Concurrency Requirements
                         │
                         ▼
             Concurrency Strategy Resolver
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
 Transaction         ORM/Persistence    Domain Policy
 Definition              │                  │
       │                 │                  │
       ▼                 ▼                  │
 Isolation          Optimistic Plan         │
       │                 │                  │
       └───────────┬─────┴───────────────┬──┘
                   │                     │
                   ▼                     ▼
           Pessimistic Plan       Constraints
                   │                     │
                   └─────────┬───────────┘
                             ▼
                     Concurrency Plan
                             │
                             ▼
                       Query Engine
                             │
                             ▼
                     Execution Engine
                             │
                             ▼
                        Database
                             │
                             ▼
                   Outcome / Exception
                             │
                             ▼
                   Conflict Classifier
                             │
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
          SUCCESS         CONFLICT          UNKNOWN
                             │
                             ▼
                      Conflict Policy
                             │
                ┌────────────┼───────────┐
                ▼            ▼           ▼
              RETRY       SURFACE       ABORT
                │
                ▼
        Transaction Retry System
```

---

# 238. Final architectural decision

VoltStack no tendrá un único mecanismo denominado informalmente:

```text
database concurrency
```

que mezcle transactions, locks y retries.

Se establecerá una arquitectura por capas:

```text
Transaction Architecture
        │
        ├── Isolation
        ├── Nested Transactions
        ├── Savepoints
        └── Transaction Context
                │
                ▼
Concurrency Control
        ├── Optimistic Locking
        ├── Pessimistic Locking
        ├── Conflict Classification
        ├── Aggregate Policies
        ├── Resource Governance
        └── Conflict Resolution
                │
                ▼
Failure Handling
        ├── Deadlock Handling
        └── Transaction Retry
```

La arquitectura mantendrá una separación esencial:

```text
Prevent Conflict
≠
Detect Conflict
≠
Classify Conflict
≠
Resolve Conflict
≠
Retry Operation
```

`Transaction Isolation` ayudará a prevenir determinadas anomalías.

`Pessimistic Locking` permitirá serializar explícitamente determinados recursos.

`Optimistic Locking` detectará modificaciones concurrentes mediante versiones u otros tokens.

`Database Constraints` protegerá invariantes que puedan expresarse estructuralmente.

`Deadlock Handling` normalizará y analizará deadlocks.

`Transaction Retry` será la única autoridad para reproducir una transacción.

`Concurrency Control System` coordinará estas capacidades sin absorber sus responsabilidades internas.

---

# 239. Regla maestra

> **VoltStack nunca considerará una operación concurrentemente segura únicamente porque esté dentro de una transacción. La seguridad concurrente será el resultado explícito de combinar las garantías reales de aislamiento de la plataforma, locking, detección de conflictos, constraints, políticas de dominio y estrategias de recuperación.**

En forma compacta:

```text
Transaction
≠
Concurrency Safety
```

y:

```text
Concurrency Safety
=
Known Guarantees
+
Explicit Strategy
+
Verified Outcome
```

---

# 240. Relación completa del bloque

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

# 241. Siguiente documento

```text
175_DATABASE_TRANSACTION_EVENT_SYSTEM.md
```

El siguiente documento definirá el sistema de eventos asociado al ciclo de vida transaccional, incluyendo:

```text
TransactionEvent
TransactionStarted
TransactionActive
TransactionBeforeCommit
TransactionCommitted
TransactionBeforeRollback
TransactionRolledBack
TransactionFailed
TransactionOutcomeUnknown
TransactionRetryScheduled
TransactionRetryStarted
SavepointCreated
SavepointReleased
SavepointRolledBack
TransactionRollbackOnlyMarked
TransactionEventDispatcher
TransactionEventContext
TransactionEventOrdering
TransactionEventPriority
TransactionEventListener
TransactionEventSubscriber
afterCommit
afterRollback
afterCompletion
event failure semantics
event reentrancy
event buffering
transaction-aware events
outbox integration boundaries
telemetry integration
persistent-runtime isolation
```

y establecerá una distinción fundamental:

```text
Transaction Event
≠
Domain Event
≠
ORM Lifecycle Event
≠
Database Query Event
≠
Outbox Message
```