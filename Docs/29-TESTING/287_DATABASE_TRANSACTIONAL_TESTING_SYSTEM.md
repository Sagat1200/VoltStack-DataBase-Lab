# 287_DATABASE_TRANSACTIONAL_TESTING_SYSTEM.md

# VoltStack Quantum Database
## Database Transactional Testing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 287 — Database Transactional Testing System  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `286_DATABASE_DATABASE_TEST_ENVIRONMENT_SYSTEM.md`  
**Siguiente documento:** `288_DATABASE_DATABASE_FAKE_AND_MOCK_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database Transactional Testing System** de VoltStack.

Su responsabilidad es proporcionar una arquitectura especializada para verificar de forma determinista y reproducible:

- inicio de transacciones;
- commit;
- rollback;
- savepoints;
- transacciones anidadas;
- niveles de aislamiento;
- visibilidad entre conexiones;
- locking;
- deadlocks;
- optimistic locking;
- pessimistic locking;
- retries;
- eventos transaccionales;
- interacción ORM/transacción;
- `flush()` frente a `commit()`;
- rollback frente al estado de objetos;
- resultados `UNKNOWN`;
- concurrencia;
- recuperación;
- aislamiento entre pruebas.

Regla central:

> **Una prueba transaccional deberá controlar explícitamente quién posee cada transacción, qué conexión la ejecuta, qué observadores pueden verificar sus efectos y qué nivel de certeza existe sobre su resultado.**

Formalmente:

```text
TransactionalTestEvidence
=
InitialDatabaseState
+
TransactionActors
+
ConnectionOwnership
+
TransactionSchedule
+
ObservedIntermediateStates
+
FinalDatabaseState
+
TransactionOutcomeEvidence
```

No:

```text
TransactionalTest
=
wrap test in transaction
+
rollback at end
```

---

# 2. Transactional Testing ≠ Transaction-per-Test

Una estrategia común de testing consiste en:

```text
BEGIN

run test

ROLLBACK
```

Esto es útil para aislamiento.

Pero no permite verificar correctamente todas las propiedades transaccionales.

Por ejemplo:

```text
commit()
deadlocks
isolation
multiple connections
savepoints
failover
unknown commit outcome
```

pueden requerir control directo de la transacción.

Por tanto:

```text
Transactional Test
≠
Test Isolation Transaction
```

---

# 3. Dos conceptos diferentes

VoltStack distinguirá:

```text
Test Isolation Transaction
```

de:

```text
Transaction Under Test
```

---

# 4. Test Isolation Transaction

Su objetivo es:

```text
prevent test state leakage
```

Ejemplo:

```text
BEGIN TEST TRANSACTION
    create data
    execute assertions
ROLLBACK TEST TRANSACTION
```

---

# 5. Transaction Under Test

Su objetivo es comprobar comportamiento transaccional real.

Ejemplo:

```text
BEGIN
INSERT
COMMIT

new connection
SELECT
```

Aquí el commit debe ocurrir realmente.

---

# 6. Regla crítica

> **Una Test Isolation Transaction nunca deberá impedir que la Transaction Under Test ejerza la semántica que se pretende verificar.**

---

# 7. Ejemplo incorrecto

```text
Outer Test Transaction
│
├── application begin()
├── application commit()
│
└── test rollback
```

Si la implementación convierte internamente:

```text
application commit()
```

en un savepoint o no realiza commit físico debido a la transacción externa, el test no ha demostrado un commit real.

---

# 8. Arquitectura general

```text
               Transactional Test
                       │
                       ▼
               Transaction Scenario
                       │
                       ▼
                Actor Coordinator
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Actor A       Actor B      Observer
          │            │            │
          ▼            ▼            ▼
    Connection A  Connection B  Connection C
          │            │            │
          └────────────┼────────────┘
                       ▼
                      DBMS
                       │
                       ▼
              Observable DB State
                       │
                       ▼
              Transaction Assertions
                       │
                       ▼
             Transaction Evidence
```

---

# 9. Transaction Test Scenario

Toda prueba transaccional compleja deberá poder representarse como escenario.

```text
TransactionScenario
├── InitialState
├── Actors
├── Connections
├── Transactions
├── SynchronizationPoints
├── Operations
├── ExpectedObservations
├── ExpectedOutcome
└── CleanupPolicy
```

---

# 10. Transaction Actor

Un actor representa una secuencia independiente de operaciones.

```php
final readonly class TransactionTestActor
{
    public function __construct(
        public TransactionActorId $id,
        public ConnectionRole $connectionRole,
        public TransactionScenarioSteps $steps,
    ) {}
}
```

Ejemplos:

```text
WriterA
WriterB
Reader
Observer
LockHolder
LockWaiter
```

---

# 11. Actor ≠ Thread

Un actor es una abstracción lógica.

Podrá ejecutarse mediante:

```text
process
thread
coroutine
fiber
worker
separate PHP execution
```

según runtime y escenario.

---

# 12. Actor ≠ Connection

Un actor normalmente utilizará una conexión.

Pero conceptualmente:

```text
Actor
≠
Connection
```

---

# 13. Connection Ownership

Cada conexión involucrada deberá tener propietario explícito.

```text
Connection A → Actor A
Connection B → Actor B
Connection C → Observer
```

---

# 14. No accidental connection sharing

Dos actores concurrentes no deberán compartir una conexión mutable salvo que el test esté verificando precisamente ese comportamiento.

---

# 15. Transaction Ownership

Toda transacción deberá saber quién la creó.

```text
Transaction
├── Owner
├── Connection
├── Context
├── Isolation
└── State
```

---

# 16. Transaction State

Estados conceptuales:

```text
NEW
ACTIVE
COMMITTING
COMMITTED
ROLLING_BACK
ROLLED_BACK
FAILED
UNKNOWN
```

---

# 17. Test state ≠ transaction state

El test puede fallar mientras la transacción haya sido confirmada.

Ejemplo:

```text
COMMIT succeeds
assertion fails
```

Resultado:

```text
Transaction = COMMITTED
Test = FAILED
```

---

# 18. Transaction outcome evidence

VoltStack deberá distinguir:

```text
requested action
```

de:

```text
demonstrated outcome
```

Ejemplo:

```text
commit requested
≠
commit demonstrated
```

---

# 19. Basic Commit Test

Escenario:

```text
Initial:
users = 0

Actor A:
BEGIN
INSERT user
COMMIT

Observer:
new connection
SELECT users
```

Resultado esperado:

```text
users = 1
```

---

# 20. Why observer matters

Comprobar únicamente el estado del mismo objeto PHP no demuestra commit.

---

# 21. Fresh Observer Principle

Cuando sea necesario demostrar estado persistido:

> **La verificación deberá realizarse mediante una conexión/contexto independiente capaz de observar el estado confirmado de la base de datos.**

---

# 22. Basic Rollback Test

```text
Initial:
users = 0

Actor A:
BEGIN
INSERT user
ROLLBACK

Observer:
SELECT users
```

Esperado:

```text
users = 0
```

---

# 23. Rollback assertion

No deberá limitarse a:

```text
transaction.state == ROLLED_BACK
```

También podrá comprobarse el estado real de DB.

---

# 24. Statement success ≠ commit

```text
INSERT succeeded
```

no implica:

```text
transaction committed
```

---

# 25. flush() ≠ commit()

Especialmente importante para ORM.

Escenario:

```text
BEGIN

entityManager.persist(user)
entityManager.flush()

Observer B:
SELECT user
```

Dependiendo del aislamiento:

```text
Observer B should not observe uncommitted user
```

Después:

```text
COMMIT
```

el observer deberá poder verlo.

---

# 26. ORM Flush Test

Pipeline:

```text
Entity
  ↓
persist()
  ↓
UnitOfWork
  ↓
flush()
  ↓
SQL executed
  ↓
Transaction still ACTIVE
```

---

# 27. Commit Test

Después:

```text
TransactionManager.commit()
```

se verificará:

```text
DB state committed
```

---

# 28. Rollback after flush

Escenario crítico:

```text
BEGIN
persist(entity)
flush()
ROLLBACK
```

Resultado:

```text
Database:
entity absent
```

---

# 29. PHP object state

Pero el objeto puede continuar:

```text
existing in memory
having generated ID
containing modified fields
```

Por tanto:

```text
Database Rollback
≠
Object Graph Rewind
```

---

# 30. Rollback Object State Testing

La suite deberá verificar la política del EntityManager/UoW después del rollback.

Posibles estados:

```text
TAINTED
DETACHED
REQUIRES_CLEAR
STALE
```

según la arquitectura final.

---

# 31. No fabricated clean state

VoltStack no deberá marcar automáticamente:

```text
EntityManager = CLEAN
```

si ya no puede demostrar coherencia entre objetos y DB.

---

# 32. Generated IDs after rollback

Ejemplo:

```text
INSERT → generated id 42
ROLLBACK
```

El objeto PHP puede seguir teniendo:

```text
id = 42
```

aunque:

```text
row 42 does not exist
```

La prueba deberá cubrir esta divergencia.

---

# 33. Savepoint Testing

Cuando:

```text
transaction.savepoint = SUPPORTED
```

deberán existir pruebas reales.

---

# 34. Savepoint basic scenario

```text
BEGIN

INSERT A

SAVEPOINT S1

INSERT B

ROLLBACK TO S1

COMMIT
```

Esperado:

```text
A = exists
B = absent
```

---

# 35. Savepoint release

Cuando exista soporte:

```text
SAVEPOINT S1
...
RELEASE S1
```

deberá verificarse.

---

# 36. Savepoint ≠ Nested Transaction

Un savepoint es un mecanismo dentro de una transacción física.

No es automáticamente una transacción independiente.

---

# 37. Nested Transaction Testing

VoltStack contempla políticas como:

```text
JOIN
SAVEPOINT
REJECT
REQUIRES_NEW
```

---

# 38. JOIN

```text
Outer BEGIN
    Inner BEGIN → join outer
    Inner COMMIT → no physical commit
Outer COMMIT
```

El test deberá verificar que el inner scope no confirme prematuramente la transacción física.

---

# 39. SAVEPOINT

```text
Outer BEGIN
    Inner BEGIN
       → SAVEPOINT
    Inner ROLLBACK
       → ROLLBACK TO SAVEPOINT
Outer COMMIT
```

---

# 40. REJECT

Una nested transaction deberá producir error explícito.

---

# 41. REQUIRES_NEW

Cuando pueda implementarse correctamente:

```text
Outer Transaction → Connection A

Inner REQUIRES_NEW → Connection B
```

o mecanismo equivalente real.

---

# 42. REQUIRES_NEW caveat

No deberá fingirse mediante savepoint si la semántica requerida es una transacción física independiente.

---

# 43. Nested failure propagation

Las pruebas deberán verificar:

```text
inner failure
```

y su efecto sobre:

```text
outer transaction
```

según política.

---

# 44. Rollback-only state

Algunas estrategias pueden marcar la transacción externa como:

```text
ROLLBACK_ONLY
```

después de ciertos errores internos.

Esto deberá ser probado explícitamente.

---

# 45. Isolation Level Testing

Los niveles deberán tratarse como semántica real del DBMS.

Ejemplos:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

cuando estén disponibles.

---

# 46. Requested ≠ Effective Isolation

```text
RequestedIsolation
≠
EffectiveIsolation
```

El test deberá utilizar la capability y comportamiento real.

---

# 47. Multiple connections required

Los tests de aislamiento normalmente requieren:

```text
Connection A
Connection B
```

---

# 48. Dirty Read Scenario

Conceptualmente:

```text
Tx A:
BEGIN
UPDATE balance = 500
(no commit)

Tx B:
SELECT balance
```

La observación dependerá del aislamiento efectivo.

---

# 49. Non-repeatable Read Scenario

```text
Tx A:
BEGIN
SELECT value → 10

Tx B:
BEGIN
UPDATE value → 20
COMMIT

Tx A:
SELECT value → ?
```

El resultado dependerá del isolation level.

---

# 50. Phantom Scenario

```text
Tx A:
SELECT WHERE score > 100
→ 5 rows

Tx B:
INSERT row score=200
COMMIT

Tx A:
SELECT WHERE score > 100
→ ?
```

---

# 51. Serialization failure

Bajo ciertos niveles el DBMS puede abortar una transacción en lugar de proporcionar una lectura concreta.

Esto también será un resultado válido según plataforma.

---

# 52. Behavioral assertions

Las pruebas deberán verificar:

```text
actual observable semantics
```

no sólo:

```text
SET TRANSACTION ISOLATION LEVEL succeeded
```

---

# 53. Transaction Visibility Matrix

Podrá modelarse:

| Actor | Uncommitted own writes | Other uncommitted | Committed writes |
|---|---:|---:|---:|
| Tx A | Sí | según aislamiento | Sí/según snapshot |
| Tx B | Sí | según aislamiento | según aislamiento |
| Observer | N/A | No normalmente | Sí |

La matriz real será capability/platform-aware.

---

# 54. Synchronization Architecture

La concurrencia deberá controlarse mediante barreras.

```text
Actor A                     Actor B

BEGIN
LOCK row
   │
   └──── barrier LOCKED ───────►
                               BEGIN
                               request same lock
   ◄──── barrier WAITING ──────
COMMIT
                               acquire lock
```

---

# 55. TestBarrier

Podrá existir:

```php
interface TransactionTestBarrier
{
    public function arrive(
        BarrierPoint $point,
        TransactionActorId $actor
    ): void;

    public function await(
        BarrierPoint $point,
        TestDeadline $deadline
    ): void;
}
```

---

# 56. Barrier ≠ sleep

Incorrecto:

```php
sleep(3);
```

como mecanismo principal para inferir que otro actor ya adquirió un lock.

---

# 57. Bounded synchronization

Toda barrera deberá tener:

```text
deadline
cancellation
diagnostics
```

---

# 58. Actor failure

Si un actor falla:

```text
Actor A = FAILED
```

los demás actores deberán ser desbloqueados/cancelados de forma segura.

---

# 59. Locking Testing

Deberán probarse las capabilities reales:

```text
exclusive row lock
shared lock
NOWAIT
SKIP LOCKED
lock timeout
```

---

# 60. Basic Pessimistic Lock Test

```text
Tx A:
BEGIN
SELECT row FOR UPDATE

Tx B:
BEGIN
SELECT same row FOR UPDATE
```

Tx B deberá:

```text
WAIT
FAIL NOWAIT
TIMEOUT
```

según política.

---

# 61. Lock ownership

El test deberá verificar que liberar/commit/rollback de Tx A permita progresar a Tx B cuando corresponda.

---

# 62. Lock release on commit

```text
Tx A COMMIT
```

deberá liberar locks transaccionales correspondientes.

---

# 63. Lock release on rollback

Igualmente:

```text
Tx A ROLLBACK
```

---

# 64. NOWAIT

Cuando sea soportado:

```text
Tx B
→ immediate lock conflict
```

deberá verificarse.

---

# 65. SKIP LOCKED

Cuando sea soportado:

```text
locked rows
```

deberán omitirse conforme a la semántica real.

---

# 66. Lock timeout

La prueba deberá distinguir:

```text
DB lock timeout
```

de:

```text
test runner timeout
```

---

# 67. Deadlock Testing

Escenario clásico:

```text
Tx A                     Tx B

lock row 1               lock row 2
    │                        │
    ▼                        ▼
request row 2            request row 1
        \                  /
         \                /
             DEADLOCK
```

---

# 68. Deadlock victim

No se deberá asumir qué transacción será víctima salvo garantía específica del DBMS.

---

# 69. Deadlock assertion

La propiedad portable será:

```text
deadlock detected
+
at least one participant aborted
+
error correctly classified
+
surviving state remains consistent
```

---

# 70. Deadlock cleanup

Después del escenario:

```text
no active abandoned transaction
no retained locks
connections reconciled
```

---

# 71. Deadlock error mapping

Pipeline:

```text
Native DB Error
      ↓
Driver
      ↓
Database Error Classification
      ↓
DeadlockException / classification
```

---

# 72. Deadlock retry

Si existe policy:

```text
deadlock
   ↓
transaction rollback
   ↓
whole transaction replay
```

---

# 73. Retry whole transaction

No:

```text
retry failed statement only
```

cuando la transacción completa ha sido invalidada.

---

# 74. Retry Boundary

Una prueba deberá demostrar que el callback/transacción completa se ejecutó nuevamente.

---

# 75. Retry attempt context

Cada intento deberá tener:

```text
attempt number
transaction id
connection context
failure evidence
```

---

# 76. Retry limit

Ejemplo:

```text
maxAttempts = 3
```

deberá verificarse.

---

# 77. Backoff

Las pruebas unitarias podrán verificar cálculo.

Las integraciones podrán verificar que el retry coordinator respeta la policy sin depender de tiempos exactos frágiles.

---

# 78. Idempotency

El test deberá comprobar que un retry seguro no produce:

```text
duplicate side effects
```

cuando el diseño afirma replayability.

---

# 79. External side effects

Una transacción que además envía:

```text
email
HTTP request
message
```

no se vuelve mágicamente replayable.

---

# 80. Outbox integration

Cuando exista integración:

```text
Business Mutation
+
Outbox Record
```

podrán verificarse dentro de la misma transacción DB.

---

# 81. External delivery

Queda fuera de la atomicidad DB salvo protocolo adicional.

---

# 82. Optimistic Locking Testing

Escenario:

```text
A loads User(version=5)
B loads User(version=5)

A updates
→ version=6

B updates with version=5
→ conflict
```

---

# 83. Required assertion

```text
B update affected rows = 0
```

o mecanismo equivalente deberá convertirse en conflicto optimista.

---

# 84. No silent overwrite

B no deberá sobrescribir A silenciosamente.

---

# 85. Entity state after optimistic conflict

El EntityManager/UoW deberá quedar en el estado definido por la arquitectura.

---

# 86. Conflict retry

Si existe retry de optimistic locking deberá ser explícito y volver a cargar/recalcular según policy.

---

# 87. Pessimistic Locking Testing

Requiere transacción real.

---

# 88. Lock outside transaction

Cuando el DBMS/contrato lo requiera, deberá rechazarse o definirse explícitamente.

---

# 89. Transaction Manager Testing

La integración deberá verificar:

```text
begin
commit
rollback
nested behavior
savepoints
isolation
failure mapping
context cleanup
```

---

# 90. Transaction Context

Durante una transacción deberá existir información scoped:

```text
TransactionContext
├── TransactionId
├── Connection
├── State
├── Isolation
├── Nesting
├── Savepoints
├── RetryAttempt
└── OutcomeEvidence
```

---

# 91. Context isolation

Dos transacciones concurrentes no deberán compartir TransactionContext mutable.

---

# 92. Persistent Runtime

Particularmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 93. Transaction leakage test

```text
Operation A:
BEGIN
throw exception

runtime cleanup

Operation B:
SELECT
```

B no deberá heredar la transacción de A.

---

# 94. Unfinished transaction

Al finalizar un scope:

```text
ACTIVE transaction
```

deberá:

```text
rollback
discard connection
or quarantine
```

según evidencia y policy.

---

# 95. Never return active transaction connection

Una conexión con transacción activa no deberá volver silenciosamente al pool.

---

# 96. Unknown transaction state

Si no puede demostrarse rollback:

```text
connection = unsafe
```

y normalmente deberá descartarse.

---

# 97. Transaction Event Testing

Podrán verificarse:

```text
TransactionStarted
TransactionCommitAttempted
TransactionCommitted
TransactionRollbackAttempted
TransactionRolledBack
TransactionFailed
TransactionOutcomeUnknown
```

según arquitectura.

---

# 98. Events follow evidence

Nunca:

```text
commit requested
→ TransactionCommitted
```

sin confirmación suficiente.

---

# 99. afterCommit testing

Escenario:

```text
COMMIT successful
afterCommit listener fails
```

Resultado:

```text
Database = committed
Transaction = COMMITTED
Listener = FAILED
```

---

# 100. Listener failure ≠ rollback

El test deberá impedir esa falsa interpretación.

---

# 101. afterRollback

Sólo deberá emitirse cuando rollback pueda afirmarse.

---

# 102. Transaction telemetry

Se podrán verificar:

```text
transaction duration
outcome
retry count
deadlock count
rollback count
```

sin exponer información sensible.

---

# 103. Transaction ID cardinality

Los IDs podrán aparecer en traces/logs.

No necesariamente como labels globales de métricas.

---

# 104. Transaction and Cache Testing

Una mutación dentro de transacción:

```text
BEGIN
UPDATE
```

no deberá publicar cache state como committed.

---

# 105. Commit-aware cache

```text
BEGIN
UPDATE
COMMIT
   ↓
invalidate/populate
```

según estrategia.

---

# 106. Rollback cache

```text
BEGIN
UPDATE
ROLLBACK
```

no deberá publicar el estado mutado.

---

# 107. UNKNOWN commit and cache

Si:

```text
commit outcome = UNKNOWN
```

se deberá probar la política conservadora de invalidación.

---

# 108. Transaction and IdentityMap

La IdentityMap refleja objetos managed.

No es una representación transaccional completa del DB.

---

# 109. Rollback mismatch

Después de rollback:

```text
IdentityMap
```

puede contener objetos que ya no representan el DB.

La suite deberá verificar el mecanismo de:

```text
clear
detach
taint
refresh requirement
```

definido.

---

# 110. Transaction and EntityManager

Si el outcome es `UNKNOWN`, el EntityManager normalmente deberá quedar:

```text
TAINTED
```

---

# 111. EntityManager reuse

Un manager `TAINTED` no deberá utilizarse como si fuera coherente.

---

# 112. Transaction and relationship persistence

Las operaciones sobre relaciones deberán obedecer la misma transacción física.

---

# 113. Partial flush failure

Si varias statements forman un flush y una falla:

```text
statement 1 succeeds
statement 2 succeeds
statement 3 fails
```

el test deberá verificar:

```text
transaction rollback semantics
+
ORM consistency policy
```

---

# 114. Autocommit Testing

El sistema deberá verificar diferencias entre:

```text
explicit transaction
```

y:

```text
autocommit statement
```

---

# 115. Autocommit ≠ explicit commit

Una statement fuera de transacción puede tener semántica de commit implícito del DBMS.

Debe modelarse explícitamente.

---

# 116. DDL Transaction Testing

Las plataformas difieren significativamente.

Ejemplo:

```text
BEGIN
CREATE TABLE ...
ROLLBACK
```

puede tener comportamiento distinto según DBMS.

---

# 117. Capability-driven DDL tests

Nunca asumir:

```text
transactional DDL = universal
```

---

# 118. Implicit commit

Algunas operaciones pueden provocar:

```text
implicit commit
```

según plataforma.

Esto deberá representarse mediante capabilities y pruebas específicas.

---

# 119. Transaction Safety

Una prueba deberá saber si la operación puede alterar el control transaccional.

---

# 120. Isolation Test Environment

Los escenarios transaccionales complejos deberán usar:

```text
database-per-test
schema-per-test
database-per-worker
```

u otra estrategia que no interfiera con la transacción bajo prueba.

---

# 121. Transactional fixture strategy

Para tests sencillos:

```text
BEGIN isolation transaction
load fixture
run assertions
ROLLBACK
```

puede ser válido.

---

# 122. Fixture visibility caveat

Si el fixture se carga dentro de una transacción de aislamiento:

```text
Connection B
```

puede no verlo.

---

# 123. Multi-connection scenario fixture

Preferible:

```text
prepare fixture
COMMIT

then begin transaction scenario
```

seguido de reset externo.

---

# 124. Cleanup after transactional tests

Podrá utilizar:

```text
drop database
truncate
restore snapshot
recreate schema
```

según environment.

---

# 125. Cleanup must not rely on tested transaction

Si el test ha roto deliberadamente el transaction manager, cleanup deberá poder ejecutarse mediante un contexto independiente.

---

# 126. Cleanup Connection

Podrá existir una conexión administrativa separada.

---

# 127. Cleanup connection privileges

Sólo los necesarios para reset.

---

# 128. Cleanup after deadlock

Deberá confirmar que no quedan transacciones abiertas.

---

# 129. Cleanup after timeout

Igualmente.

---

# 130. Cleanup after unknown outcome

Podrá requerir:

```text
reconcile database state
+
discard tested connection
+
full environment reset
```

---

# 131. Unknown Commit Outcome

Este es uno de los escenarios más importantes.

```text
Client
   │
COMMIT
   │
   ▼
Server
   │
commit occurs
   │
   ▼
ACK
   X network failure
```

El cliente puede no saber si el commit ocurrió.

---

# 132. UNKNOWN ≠ FAILED

No deberá transformarse automáticamente en:

```text
ROLLED_BACK
```

---

# 133. UNKNOWN ≠ COMMITTED

Tampoco podrá suponerse commit.

---

# 134. UNKNOWN outcome test requirements

Requiere capacidad para interrumpir comunicación en una ventana controlada.

---

# 135. Failure injection

Podrá utilizar:

```text
network proxy
connection killer
custom driver test adapter
DBMS process control
```

dependiendo del nivel de realismo requerido.

---

# 136. Deterministic unknown simulation

Para lógica interna podrá usarse un fake driver.

Pero la integración real deberá existir para las fronteras donde sea técnicamente posible.

---

# 137. Unknown reconciliation

El sistema podrá usar:

```text
business idempotency key
transaction marker
domain identifier
audit record
```

para determinadas reconciliaciones.

No habrá mecanismo universal.

---

# 138. Reconciliation ≠ transaction recovery

Comprobar después si un registro existe puede resolver un caso de negocio concreto.

No convierte el protocolo DB en exactamente una vez.

---

# 139. Retry after UNKNOWN

Por defecto:

```text
UNKNOWN commit
→ no blind retry
```

---

# 140. Why

Porque:

```text
first attempt may have committed
```

y repetir podría duplicar efectos.

---

# 141. Connection Failure During Transaction

Escenarios:

```text
before statement
during statement
after statement
during commit
during rollback
```

deberán diferenciarse.

---

# 142. Failure phase

El error deberá registrar:

```text
phase
transaction state
statement state
outcome certainty
```

---

# 143. Statement UNKNOWN

Incluso antes de commit, determinadas operaciones pueden tener resultado incierto desde el cliente.

---

# 144. Transaction-level uncertainty

Si la conexión se pierde:

```text
transaction state
```

puede quedar indeterminado para el cliente hasta que el servidor resuelva/desconecte.

---

# 145. Connection discard

Después de pérdida grave de protocolo:

```text
reuse = false
```

por defecto.

---

# 146. Failover During Transaction

Escenario:

```text
Tx A on Writer 1
Writer 1 fails
Writer 2 promoted
```

VoltStack no deberá continuar:

```text
Tx A
```

sobre Writer 2 como si fuera la misma transacción.

---

# 147. Required behavior

```text
transaction failed/unknown
+
connection invalid
+
new transaction required
```

según evidencia.

---

# 148. Replica transaction tests

Una read-only transaction podrá ejecutarse sobre replica sólo cuando routing/capabilities/policy lo permitan.

---

# 149. Read-only enforcement

Si el sistema declara una transacción:

```text
READ ONLY
```

deberá probarse su enforcement efectivo cuando sea posible.

---

# 150. Writer affinity

Una transacción con escrituras deberá quedar fijada al writer correspondiente.

---

# 151. Transaction Routing Stability

Dentro de una transacción:

```text
connection/topology target
```

no deberá cambiar arbitrariamente.

---

# 152. Sharded Transaction Testing

Una transacción normal deberá tener:

```text
Shard Affinity
```

---

# 153. Cross-shard transaction

No deberá exponerse como ACID global salvo implementación distribuida real.

---

# 154. Cross-shard rejection

La suite deberá verificar que una segunda operación hacia otro shard pueda ser rechazada cuando el contexto ya esté fijado.

---

# 155. Tenant Transaction Testing

Una transacción deberá conservar:

```text
Tenant Affinity
```

---

# 156. Tenant switch during transaction

Deberá rechazarse.

```text
BEGIN Tenant A
switch Tenant B
```

no deberá migrar silenciosamente la transacción.

---

# 157. Transaction Context Immutability

Propiedades como:

```text
tenant
shard
connection
```

deberán quedar fijadas cuando sean semánticamente necesarias.

---

# 158. Transaction Deadline

Podrá existir:

```text
TransactionDeadline
```

para limitar transacciones largas.

---

# 159. Test deadline ≠ transaction deadline

Ambos deberán distinguirse.

---

# 160. Transaction timeout

Puede provenir de:

```text
DBMS
driver
VoltStack policy
test runner
```

La prueba deberá identificar la fuente.

---

# 161. Long transaction tests

Podrán existir para verificar:

```text
timeout
lock retention
resource governance
```

pero no deberán bloquear indefinidamente CI.

---

# 162. Resource Governance

La suite deberá controlar:

```text
max concurrent transactions
max connections
lock wait duration
scenario duration
retry count
```

---

# 163. Transaction Scenario DSL

Podrá existir una API declarativa.

Ejemplo conceptual:

```php
$scenario
    ->actor('A')
        ->begin()
        ->update('accounts', 1)
        ->barrier('a-locked')
    ->actor('B')
        ->await('a-locked')
        ->begin()
        ->update('accounts', 1)
        ->expectLockWait()
    ->actor('A')
        ->commit()
    ->actor('B')
        ->expectProgress()
        ->commit();
```

---

# 164. DSL ≠ transaction engine

La DSL será exclusivamente infraestructura de testing.

---

# 165. Deterministic schedules

Los escenarios deberán intentar controlar:

```text
operation ordering
```

mediante barriers.

---

# 166. Real DB scheduling remains partially nondeterministic

El DBMS puede tomar decisiones internas no controlables.

Por tanto, assertions deberán evitar supuestos no garantizados.

---

# 167. Scenario Trace

Cada ejecución podrá registrar:

```text
T0 Actor A BEGIN
T1 Actor A LOCK row1
T2 Actor B BEGIN
T3 Actor B requests row1
T4 Actor A COMMIT
T5 Actor B acquires row1
```

---

# 168. Logical timestamps

El trace podrá usar secuencias lógicas además del reloj real.

---

# 169. Scenario diagnostics

Ante fallo:

```text
actor states
transaction states
barrier states
connection states
native errors
scenario trace
```

deberán estar disponibles.

---

# 170. Transaction Assertions

Podrán existir:

```text
assertTransactionActive()
assertCommitted()
assertRolledBack()
assertOutcomeUnknown()
assertRowVisibleTo()
assertRowNotVisibleTo()
assertLockConflict()
assertDeadlock()
assertRetryCount()
assertConnectionDiscarded()
```

---

# 171. Assertion semantics

No deberán inferir estado únicamente desde flags internos si la propiedad requiere evidencia externa.

---

# 172. Database State Assertion

Ejemplo:

```text
assertCommitted()
```

podrá requerir observer independiente.

---

# 173. Intermediate state assertions

Algunas propiedades requieren observar durante una transacción.

Ejemplo:

```text
A sees own write
B does not see A uncommitted write
```

---

# 174. Observer isolation

El observer deberá tener su propio contexto claramente definido.

---

# 175. Transaction Test Result

Podrá modelarse:

```php
final readonly class TransactionTestResult
{
    public function __construct(
        public TransactionScenarioId $scenario,
        public TransactionScenarioStatus $status,
        public array $actorResults,
        public array $transactionOutcomes,
        public TransactionScenarioTrace $trace,
    ) {}
}
```

---

# 176. Scenario status

```text
PASSED
FAILED
INCONCLUSIVE
ENVIRONMENT_FAILURE
CANCELLED
UNKNOWN
```

---

# 177. INCONCLUSIVE

Útil cuando la propiedad no pudo observarse.

Ejemplo:

```text
deadlock was not formed
```

por scheduling inesperado.

---

# 178. INCONCLUSIVE ≠ PASSED

No deberá ocultarse.

---

# 179. Repetition policy

Algunos escenarios de concurrencia podrán repetirse.

---

# 180. Repetition evidence

Cada intento deberá conservar:

```text
iteration
seed
schedule trace
outcome
```

---

# 181. Flaky test detection

Si:

```text
same environment
same scenario
```

produce resultados incompatibles sin una causa semánticamente permitida, deberá investigarse como posible flakiness/race.

---

# 182. Platform Matrix

Los contratos deberán ejecutarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

según capabilities.

---

# 183. SQLite caveats

Determinadas pruebas de:

```text
concurrent writes
row locking
isolation
```

pueden tener semánticas distintas.

No deberán forzarse a imitar PostgreSQL/MySQL.

---

# 184. Capability Gates

Ejemplos:

```text
transaction.savepoint
transaction.isolation.serializable
locking.nowait
locking.skip_locked
transaction.read_only
```

---

# 185. Unsupported

Una capability legítimamente ausente podrá producir:

```text
SKIPPED_NOT_SUPPORTED
```

---

# 186. UNKNOWN

No deberá convertirse automáticamente en unsupported.

---

# 187. Cross-platform contract

Ejemplo portable:

```text
BEGIN
INSERT
ROLLBACK
→ inserted row absent
```

deberá cumplirse en toda plataforma que declare transacciones apropiadas.

---

# 188. Platform-specific contract

Ejemplo:

```text
PostgreSQL serialization failure behavior
```

podrá tener suite específica.

---

# 189. Transactional DDL

Deberá estar capability-gated.

---

# 190. Autocommit platform differences

También.

---

# 191. Error taxonomy

```text
TransactionalTestingException
├── TransactionScenarioException
├── TransactionActorException
├── TransactionSynchronizationException
├── TransactionBarrierTimeoutException
├── TransactionObservationException
├── TransactionAssertionException
├── DeadlockScenarioException
├── LockScenarioException
├── IsolationScenarioException
├── TransactionFailureInjectionException
├── TransactionCleanupException
└── TransactionOutcomeIndeterminateException
```

---

# 192. Test infrastructure failure

Ejemplo:

```text
Actor B process crashed
```

antes de ejecutar el escenario.

Esto podrá ser:

```text
ENVIRONMENT_FAILURE
```

en lugar de fallo funcional del TransactionManager.

---

# 193. Expected transaction failure

Ejemplo:

```text
deadlock exception
```

puede ser el resultado esperado y por tanto el test puede pasar.

---

# 194. Exception ≠ failed test automatically

El test evaluará si la excepción corresponde a la semántica esperada.

---

# 195. Proposed namespace

```text
src/Quantum/Database/Testing/Transaction/
├── Contract/
│   ├── TransactionScenario.php
│   ├── TransactionActor.php
│   ├── TransactionBarrier.php
│   └── TransactionObserver.php
│
├── Model/
│   ├── TransactionScenarioId.php
│   ├── TransactionActorId.php
│   ├── TransactionScenarioStep.php
│   ├── TransactionScenarioResult.php
│   ├── TransactionActorResult.php
│   └── TransactionScenarioTrace.php
│
├── Scenario/
│   ├── TransactionScenarioBuilder.php
│   ├── TransactionScenarioRunner.php
│   └── TransactionScenarioCoordinator.php
│
├── Actor/
│   ├── TransactionActorRunner.php
│   └── TransactionActorContext.php
│
├── Synchronization/
│   ├── TestBarrier.php
│   ├── BarrierPoint.php
│   └── BarrierCoordinator.php
│
├── Observer/
│   ├── DatabaseTransactionObserver.php
│   └── TransactionStateObserver.php
│
├── Assertion/
│   ├── TransactionAssertions.php
│   ├── IsolationAssertions.php
│   ├── LockAssertions.php
│   └── VisibilityAssertions.php
│
├── Failure/
│   ├── TransactionFailureInjector.php
│   └── TransactionFailureScenario.php
│
├── Trace/
│   ├── TransactionTraceCollector.php
│   └── TransactionTraceEvent.php
│
└── Exception/
    └── ...
```

---

# 196. Unit vs Integration

La lógica de:

```text
retry classification
nested policy
state machine
event decision
```

deberá tener Unit Tests.

La semántica de:

```text
commit
rollback
lock
deadlock
isolation
```

deberá tener Integration Tests reales.

---

# 197. Fake transactional tests

Podrán utilizarse para explorar estados difíciles.

Pero:

```text
Fake Commit Success
```

no demuestra commit real.

---

# 198. Testing pyramid

```text
             Specialized Failure Tests
                     /\
                    /  \
             Integration Tests
                  /      \
                 /        \
            Unit Transaction Tests
```

---

# 199. Invariantes

## DB-TXTEST-001

Transactional Testing no será sinónimo de transaction-per-test.

## DB-TXTEST-002

Test Isolation Transaction será distinta de Transaction Under Test.

## DB-TXTEST-003

El mecanismo de aislamiento no alterará la propiedad evaluada.

## DB-TXTEST-004

Toda Transaction Under Test tendrá ownership explícito.

## DB-TXTEST-005

Toda transacción tendrá conexión identificable.

## DB-TXTEST-006

Actor no será Connection.

## DB-TXTEST-007

Actor no será Thread necesariamente.

## DB-TXTEST-008

Connection sharing concurrente será explícito.

## DB-TXTEST-009

Requested commit no significará committed.

## DB-TXTEST-010

Statement success no significará committed transaction.

## DB-TXTEST-011

Commit podrá verificarse con observer independiente.

## DB-TXTEST-012

Rollback podrá verificarse mediante estado real de DB.

## DB-TXTEST-013

flush no será commit.

## DB-TXTEST-014

persist no será INSERT.

## DB-TXTEST-015

Rollback no será object graph rewind.

## DB-TXTEST-016

Generated ID podrá sobrevivir en memoria a rollback.

## DB-TXTEST-017

ORM consistency post-rollback será explícita.

## DB-TXTEST-018

Savepoint no será nested physical transaction.

## DB-TXTEST-019

JOIN no realizará commit físico interno.

## DB-TXTEST-020

SAVEPOINT policy requerirá capability correspondiente.

## DB-TXTEST-021

REJECT producirá error explícito.

## DB-TXTEST-022

REQUIRES_NEW no se fingirá con savepoint si requiere independencia física.

## DB-TXTEST-023

Nested failure propagation será definida por policy.

## DB-TXTEST-024

Requested isolation no será Effective isolation.

## DB-TXTEST-025

Isolation testing utilizará múltiples conexiones cuando sea necesario.

## DB-TXTEST-026

Dirty read se comprobará por comportamiento.

## DB-TXTEST-027

Non-repeatable read se comprobará por comportamiento.

## DB-TXTEST-028

Phantom behavior se comprobará por comportamiento.

## DB-TXTEST-029

Serialization failure podrá ser resultado legítimo.

## DB-TXTEST-030

SET isolation success no demostrará aislamiento efectivo.

## DB-TXTEST-031

Concurrency se coordinará con barriers.

## DB-TXTEST-032

Sleep no será mecanismo principal de sincronización.

## DB-TXTEST-033

Toda barrera tendrá deadline.

## DB-TXTEST-034

Actor failure liberará/cancelará participantes bloqueados.

## DB-TXTEST-035

Locking tests utilizarán locks reales.

## DB-TXTEST-036

Lock timeout será distinto de test timeout.

## DB-TXTEST-037

Locks deberán liberarse después de commit según semántica DB.

## DB-TXTEST-038

Locks deberán liberarse después de rollback según semántica DB.

## DB-TXTEST-039

NOWAIT será capability-gated.

## DB-TXTEST-040

SKIP LOCKED será capability-gated.

## DB-TXTEST-041

Deadlock testing tendrá al menos dos transacciones.

## DB-TXTEST-042

Deadlock victim no será asumida arbitrariamente.

## DB-TXTEST-043

Deadlock deberá mapearse a clasificación VoltStack.

## DB-TXTEST-044

Retry de deadlock repetirá la transacción completa cuando corresponda.

## DB-TXTEST-045

Statement retry no sustituirá transaction replay.

## DB-TXTEST-046

Cada retry tendrá attempt identity.

## DB-TXTEST-047

Retry tendrá límite.

## DB-TXTEST-048

UNKNOWN commit no tendrá blind retry.

## DB-TXTEST-049

External side effects no serán transaccionales automáticamente.

## DB-TXTEST-050

Outbox record podrá participar en la transacción DB.

## DB-TXTEST-051

Optimistic locking tendrá test concurrente real.

## DB-TXTEST-052

Optimistic conflict no permitirá silent overwrite.

## DB-TXTEST-053

Pessimistic locking requerirá transaction context cuando el contrato lo exija.

## DB-TXTEST-054

TransactionContext será scoped.

## DB-TXTEST-055

TransactionContext no será global mutable.

## DB-TXTEST-056

Persistent worker no conservará transacción entre operaciones.

## DB-TXTEST-057

Active transaction connection no volverá silenciosamente al pool.

## DB-TXTEST-058

Unknown connection state impedirá reuse seguro.

## DB-TXTEST-059

Transaction events seguirán evidencia.

## DB-TXTEST-060

CommitAttempted no será Committed.

## DB-TXTEST-061

afterCommit failure no deshará commit.

## DB-TXTEST-062

afterRollback requerirá rollback demostrable.

## DB-TXTEST-063

Transaction telemetry no redefinirá outcome.

## DB-TXTEST-064

Uncommitted mutations no se publicarán como committed cache state.

## DB-TXTEST-065

Rollback no publicará cache state inexistente.

## DB-TXTEST-066

UNKNOWN commit usará cache policy conservadora.

## DB-TXTEST-067

IdentityMap no será transaction snapshot.

## DB-TXTEST-068

EntityManager podrá quedar TAINTED.

## DB-TXTEST-069

TAINTED manager no se reutilizará como limpio.

## DB-TXTEST-070

Partial flush failure tendrá consistency policy explícita.

## DB-TXTEST-071

Autocommit será distinto de explicit transaction.

## DB-TXTEST-072

DDL transactional behavior será capability-aware.

## DB-TXTEST-073

Implicit commit será platform-aware.

## DB-TXTEST-074

Transactional fixtures no ocultarán semántica de múltiples conexiones.

## DB-TXTEST-075

Fixtures multi-connection podrán prepararse y confirmarse antes del escenario.

## DB-TXTEST-076

Cleanup no dependerá necesariamente de la transacción bajo prueba.

## DB-TXTEST-077

Cleanup podrá utilizar conexión independiente.

## DB-TXTEST-078

Deadlock cleanup verificará ausencia de transacciones abandonadas.

## DB-TXTEST-079

UNKNOWN outcome podrá requerir environment reset completo.

## DB-TXTEST-080

UNKNOWN no será FAILED automáticamente.

## DB-TXTEST-081

UNKNOWN no será COMMITTED automáticamente.

## DB-TXTEST-082

UNKNOWN no será ROLLED_BACK automáticamente.

## DB-TXTEST-083

Failure phase formará parte de evidencia.

## DB-TXTEST-084

Connection loss durante commit será tratado conservadoramente.

## DB-TXTEST-085

Protocol-corrupted connection será descartada.

## DB-TXTEST-086

Failover no migrará una transacción activa.

## DB-TXTEST-087

Nueva writer requerirá nueva transacción.

## DB-TXTEST-088

Read-only transaction routing será capability/policy-aware.

## DB-TXTEST-089

Write transaction tendrá writer affinity.

## DB-TXTEST-090

Transaction routing permanecerá estable durante su scope.

## DB-TXTEST-091

Sharded transaction tendrá shard affinity.

## DB-TXTEST-092

Cross-shard ACID no se fingirá.

## DB-TXTEST-093

Tenant transaction tendrá tenant affinity.

## DB-TXTEST-094

Tenant switch durante transaction será rechazado.

## DB-TXTEST-095

Transaction deadline será distinto de test deadline.

## DB-TXTEST-096

DB timeout será distinto de runner timeout.

## DB-TXTEST-097

Long-running tests tendrán límites.

## DB-TXTEST-098

Scenario DSL no será transaction engine.

## DB-TXTEST-099

Scenario scheduling será controlado donde sea posible.

## DB-TXTEST-100

DB internal scheduling podrá permanecer parcialmente no determinista.

## DB-TXTEST-101

Scenario trace conservará orden lógico.

## DB-TXTEST-102

Failure diagnostics incluirán estados de actores.

## DB-TXTEST-103

Assertions no dependerán sólo de flags internos.

## DB-TXTEST-104

Intermediate visibility podrá ser assertion.

## DB-TXTEST-105

Observer tendrá contexto independiente.

## DB-TXTEST-106

INCONCLUSIVE no será PASSED.

## DB-TXTEST-107

Concurrency repetitions conservarán evidence.

## DB-TXTEST-108

Flakiness no se ocultará mediante retries del test runner.

## DB-TXTEST-109

MySQL tendrá suite transaccional real.

## DB-TXTEST-110

MariaDB tendrá suite transaccional real.

## DB-TXTEST-111

PostgreSQL tendrá suite transaccional real.

## DB-TXTEST-112

SQLite tendrá suite conforme a sus propias capacidades.

## DB-TXTEST-113

SQLite no imitará artificialmente PostgreSQL.

## DB-TXTEST-114

Capability-gated tests no dependerán sólo de vendor name.

## DB-TXTEST-115

Unsupported será distinto de UNKNOWN.

## DB-TXTEST-116

Portable transaction contracts se compartirán.

## DB-TXTEST-117

Platform-specific semantics tendrán tests específicos.

## DB-TXTEST-118

Expected exception no implicará test failure.

## DB-TXTEST-119

Environment failure será distinto de transaction behavior failure.

## DB-TXTEST-120

Unit tests no sustituirán commit integration.

## DB-TXTEST-121

Fake deadlock no sustituirá deadlock integration.

## DB-TXTEST-122

Fake isolation no sustituirá isolation integration.

## DB-TXTEST-123

Fresh observer será preferido para demostrar persistencia confirmada.

## DB-TXTEST-124

Observer state no deberá contaminar Transaction Under Test.

## DB-TXTEST-125

Transaction outcome será evidencia, no inferencia optimista.

## DB-TXTEST-126

Rollback attempt no significará rollback success.

## DB-TXTEST-127

Connection close no significará rollback observado desde cliente, salvo garantía/evidencia aplicable.

## DB-TXTEST-128

Transaction retries conservarán causa original.

## DB-TXTEST-129

Transaction retries generarán nueva attempt identity.

## DB-TXTEST-130

No se reutilizará un transaction context terminado.

## DB-TXTEST-131

Savepoint names serán scope-safe.

## DB-TXTEST-132

Nested transaction depth tendrá límites.

## DB-TXTEST-133

Savepoint cleanup será verificable.

## DB-TXTEST-134

Rollback-only state no podrá cometerse silenciosamente.

## DB-TXTEST-135

Aislamiento se evaluará sobre comportamiento observable.

## DB-TXTEST-136

Lock ownership será verificable mediante escenario.

## DB-TXTEST-137

Lock waiter tendrá bounded wait.

## DB-TXTEST-138

Test cancellation liberará recursos transaccionales cuando sea posible.

## DB-TXTEST-139

Cancellation uncertainty será preservada.

## DB-TXTEST-140

Resource exhaustion no se confundirá con deadlock.

## DB-TXTEST-141

Deadlock no se confundirá con lock timeout.

## DB-TXTEST-142

Serialization failure no se confundirá con deadlock.

## DB-TXTEST-143

Optimistic conflict no se confundirá con DB deadlock.

## DB-TXTEST-144

Transaction error taxonomy conservará diferencias semánticas.

## DB-TXTEST-145

Test traces no expondrán bindings sensibles.

## DB-TXTEST-146

Transaction IDs no serán métricas de alta cardinalidad por defecto.

## DB-TXTEST-147

Cleanup deberá ser posible incluso si TransactionManager bajo prueba falla.

## DB-TXTEST-148

Persistent runtime cleanup tendrá pruebas específicas.

## DB-TXTEST-149

Transaction state nunca se almacenará en singleton global mutable.

## DB-TXTEST-150

La suite transaccional deberá probar aquello que VoltStack afirma, no aquello que una plataforma no promete.

---

# 200. Anti-patrones

## 200.1 Envolver todos los tests en una transacción

Puede ocultar:

```text
real commit
multi-connection visibility
deadlocks
failover
```

---

## 200.2 `sleep()` para locks

```php
sleep(2);
```

no demuestra que el otro actor haya adquirido el lock.

---

## 200.3 Una conexión para probar aislamiento

Aislamiento entre transacciones requiere observadores independientes.

---

## 200.4 Suponer que rollback revierte objetos PHP

Incorrecto.

---

## 200.5 Reintentar sólo la statement del deadlock

Puede violar la semántica transaccional.

---

## 200.6 Reintentar UNKNOWN commit

Puede duplicar efectos.

---

## 200.7 Asumir víctima del deadlock

No portable.

---

## 200.8 Compartir TransactionContext global

Crítico en workers persistentes.

---

## 200.9 Retornar conexión activa al pool

Produce contaminación entre operaciones.

---

## 200.10 Convertir cualquier timeout en deadlock

Semánticamente incorrecto.

---

## 200.11 Fingir REQUIRES_NEW con savepoint

Cuando la API promete independencia física, es incorrecto.

---

## 200.12 Declarar éxito porque `commit()` no lanzó excepción

Cuando el escenario requiere evidencia externa adicional.

---

## 200.13 Usar el mismo EntityManager como observer

Puede responder desde IdentityMap en lugar de observar DB.

---

## 200.14 Confundir test retry con transaction retry

Reejecutar todo el test hasta que pase no valida el retry del framework.

---

## 200.15 Forzar semántica PostgreSQL en SQLite

Incorrecto.

---

# 201. Modelo formal

Sea:

```text
D0 = initial database state
A  = set of transaction actors
S  = transaction schedule
I  = effective isolation configuration
F  = injected failures
```

La ejecución produce:

```text
(Df, O, E)
=
ExecuteTransactions(D0, A, S, I, F)
```

donde:

```text
Df = final database state
O  = transaction outcomes
E  = observations/evidence
```

La prueba pasa cuando:

```text
Expected(Df, O, E) = true
```

---

# 202. Modelo de visibilidad

Para:

```text
T1 = transaction 1
T2 = transaction 2
w  = write performed by T1
```

la visibilidad:

```text
Visible(T2, w, t)
```

dependerá de:

```text
CommitState(T1)
Isolation(T2)
Snapshot(T2)
PlatformSemantics
```

No únicamente del tiempo físico.

---

# 203. Modelo de outcome

```text
Outcome(T)
∈
{
    COMMITTED,
    ROLLED_BACK,
    FAILED,
    UNKNOWN
}
```

con:

```text
UNKNOWN
```

como estado arquitectónicamente válido.

---

# 204. Regla de retry

Un retry será permitido sólo si:

```text
RetryableFailure
∧
ReplayableBoundary
∧
Outcome != UNKNOWN_COMMIT
∧
AttemptsRemaining
```

---

# 205. Regla de conexión

```text
Reusable(Connection)
=
KnownProtocolState
∧
NoActiveTransaction
∧
SessionResetSuccessful
∧
NotPoisoned
```

---

# 206. Regla ORM

Después de rollback/fallo:

```text
DatabaseReality
```

y:

```text
ORMKnowledge
```

pueden divergir.

Por tanto:

```text
DatabaseRollback
≠
ORMConsistencyProof
```

---

# 207. Arquitectura consolidada

```text
                 TRANSACTIONAL TEST
                         │
                         ▼
                 Scenario Definition
                         │
                         ▼
                Scenario Coordinator
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
        Actor A        Actor B      Observer
           │             │             │
           ▼             ▼             ▼
      Connection A  Connection B  Connection C
           │             │             │
           └─────────────┼─────────────┘
                         ▼
                        DBMS
                         │
      ┌──────────────────┼──────────────────┐
      ▼                  ▼                  ▼
 Transactions          Locks            Visibility
      │                  │                  │
      └──────────────────┼──────────────────┘
                         ▼
                  Scenario Evidence
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     Outcome         DB State          ORM State
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                    Assertions
                         │
                         ▼
                       Cleanup
```

---

# 208. Regla final

> **VoltStack no considerará demostrada una propiedad transaccional únicamente porque una API haya devuelto éxito. Las pruebas deberán observar la realidad de la base de datos, controlar las conexiones participantes y conservar explícitamente la incertidumbre cuando el resultado no pueda determinarse.**

En forma compacta:

```text
flush()
≠
commit()
```

```text
commit requested
≠
commit proven
```

```text
rollback()
≠
object graph rewind
```

```text
savepoint
≠
independent transaction
```

```text
timeout
≠
deadlock
```

```text
connection lost
≠
rollback proven
```

y:

```text
Transactional Correctness
=
Controlled Actors
+
Known Connections
+
Explicit Schedule
+
Real DB Semantics
+
Observable Evidence
+
Conservative Outcome Classification
```

---

# 209. Siguiente documento

```text
288_DATABASE_DATABASE_FAKE_AND_MOCK_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Database Fake and Mock System
│
├── Test Double Architecture
├── Fake vs Mock vs Stub vs Spy
├── Fake Driver
├── Fake Connection
├── Fake Statement
├── Fake Result
├── Fake Cursor
├── Fake Transaction
├── Fake Schema Introspector
├── Fake Capability Provider
├── Fake Clock
├── Fake Event Dispatcher
├── Fake Telemetry
├── Fake Cache Integration
├── Scripted Database Behavior
├── Deterministic Failure Injection
├── Query Recording
├── Interaction Verification
├── Contract-safe Test Doubles
├── State Machines
├── UNKNOWN Outcome Simulation
├── Persistent Runtime Isolation
├── Test Double Reset
├── Anti-overmocking Rules
└── Boundary Between Unit and Integration Tests
```

manteniendo como principio:

> **Un fake o mock de Database deberá facilitar pruebas deterministas de la lógica de VoltStack sin presentarse como evidencia del comportamiento de un DBMS real; cuanto más dependa una propiedad del protocolo, concurrencia, locking, aislamiento o semántica del motor, menor será la autoridad probatoria del test double.**