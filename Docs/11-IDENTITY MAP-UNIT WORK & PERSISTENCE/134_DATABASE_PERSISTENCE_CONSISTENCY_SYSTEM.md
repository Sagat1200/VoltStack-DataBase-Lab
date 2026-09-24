# 134_DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM.md

# VoltStack Quantum Database
## Database Persistence Consistency System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 134 — Database Persistence Consistency System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Persistence Consistency System` define cómo VoltStack determina, preserva, verifica, degrada y recupera la consistencia entre las diferentes representaciones de estado que participan en la persistencia ORM.

El sistema coordina conceptualmente:

```text
Entity
EntityState
IdentityMap
UnitOfWork
ChangeSet
EntitySnapshot
PersistencePlan
PersistenceOperation
ExecutionOutcome
GeneratedValues
Relationships
Transaction
Lifecycle
DatabaseKnowledge
```

El objetivo no es garantizar que la memoria siempre sea idéntica al estado físico de la base de datos.

Eso sería imposible frente a:

- concurrencia;
- otras conexiones;
- otros procesos;
- réplicas;
- triggers;
- raw SQL;
- bulk operations;
- errores de red;
- timeouts;
- transacciones;
- rollback;
- resultados de ejecución desconocidos.

La garantía correcta será:

> **VoltStack deberá conocer y representar explícitamente qué nivel de consistencia puede afirmar en cada momento.**

---

# 2. Regla fundamental

VoltStack distinguirá:

```text
Internal ORM Consistency
Database Observational Consistency
Transaction Consistency
Durable Persistence Certainty
```

Por tanto:

```text
ORM Internally Consistent
≠
Database Definitely Contains Same State
```

y:

```text
SQL Execution Succeeded
≠
Transaction Committed
```

y:

```text
Transaction Commit Requested
≠
Commit Outcome Known
```

---

# 3. Problema fundamental

Supongamos:

```php
$user->setName('Alice');

$entityManager->flush();
```

Después de `flush()` pueden existir varios escenarios.

### Escenario A

```text
UPDATE succeeded
no explicit transaction pending
outcome certain
```

La persistencia puede considerarse aplicada según las garantías de la conexión.

### Escenario B

```text
UPDATE succeeded
transaction still ACTIVE
```

El ORM sabe que el statement funcionó, pero:

```text
durability = not yet established
```

### Escenario C

```text
UPDATE sent
connection lost
```

No sabemos si:

```text
UPDATE executed
```

### Escenario D

```text
UPDATE succeeded
COMMIT sent
connection lost before acknowledgment
```

La situación es todavía más delicada:

```text
TransactionOutcome = UNKNOWN
```

El ORM no puede afirmar:

```text
COMMITTED
```

ni:

```text
ROLLED_BACK
```

---

# 4. Principio epistemológico

La arquitectura necesita diferenciar:

```text
actual database state
```

de:

```text
what VoltStack knows about database state
```

Formalmente:

```text
DatabaseReality ≠ ORMKnowledge
```

El Persistence Consistency System trabaja principalmente con:

```text
ORMKnowledge
```

basado en evidencia.

---

# 5. Arquitectura general

```text
                     Domain Entity
                          │
                          ▼
                     EntityState
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        IdentityMap    Snapshot     ChangeSet
             │            │            │
             └────────────┼────────────┘
                          ▼
                     UnitOfWork
                          │
                          ▼
                  PersistencePlan
                          │
                          ▼
                   Physical Plan
                          │
                          ▼
                  Execution Engine
                          │
                          ▼
                      Database
                          │
                          ▼
                 Execution Evidence
                          │
                          ▼
                  Transaction State
                          │
                          ▼
             Persistence Reconciliation
                          │
                          ▼
              ┌──────────────────────┐
              │ Consistency System   │
              └──────────┬───────────┘
                         │
           ┌─────────────┼──────────────┐
           ▼             ▼              ▼
      CONSISTENT      UNCERTAIN       TAINTED
```

---

# 6. No objetivos

Este sistema no será responsable de:

```text
generating SQL
executing SQL
detecting entity changes
managing database transactions
creating snapshots
building PersistencePlans
implementing IdentityMap
performing hydration
```

Coordina y evalúa las garantías producidas por esos sistemas.

---

# 7. Distinciones fundamentales

```text
Consistency ≠ Equality

Consistency ≠ Freshness

Consistency ≠ Durability

Consistency ≠ Transaction Commit

Consistency ≠ Snapshot

Consistency ≠ Change Tracking

Consistency ≠ Cache Coherence

Consistency ≠ Isolation Level

Consistency ≠ Replication Consistency

Consistency ≠ Database Integrity Constraints

Consistency ≠ Eventual Consistency
```

---

# 8. Consistency Domains

VoltStack modelará la consistencia por dominios.

Propuesta:

```php
enum PersistenceConsistencyDomain
{
    case IDENTITY;
    case ENTITY_STATE;
    case SNAPSHOT;
    case CHANGE_SET;
    case UNIT_OF_WORK;
    case RELATIONSHIP;
    case PERSISTENCE_PLAN;
    case EXECUTION;
    case GENERATED_VALUES;
    case TRANSACTION;
    case DATABASE_KNOWLEDGE;
}
```

---

# 9. ¿Por qué múltiples dominios?

Porque puede ocurrir:

```text
IdentityMap       = CONSISTENT
EntityState       = CONSISTENT
Snapshot          = CONSISTENT
Transaction       = UNKNOWN
DatabaseKnowledge = UNCERTAIN
```

No debe colapsarse todo inmediatamente a un único booleano.

---

# 10. Consistency Status

Propuesta:

```php
enum PersistenceConsistencyStatus
{
    case CONSISTENT;
    case STALE;
    case UNCERTAIN;
    case INCONSISTENT;
    case TAINTED;
    case UNKNOWN;
}
```

---

# 11. Significado de CONSISTENT

`CONSISTENT` significa:

> Según la evidencia y garantías conocidas por VoltStack, el dominio satisface actualmente sus invariantes.

No significa:

```text
the database cannot have changed externally
```

---

# 12. STALE

`STALE` significa:

```text
representation is internally valid
but known to potentially represent an older database observation
```

Ejemplo:

```text
Entity loaded
   ↓
external process updates row
   ↓
managed entity remains unchanged
```

La entidad puede estar:

```text
internally consistent
but stale
```

---

# 13. UNCERTAIN

Significa:

```text
insufficient evidence exists
to determine the actual persistence outcome
```

Ejemplo:

```text
UPDATE sent
connection lost
```

---

# 14. INCONSISTENT

Significa que existe evidencia concreta de violación de invariantes.

Ejemplo:

```text
IdentityMap:
User#42 → Object A

UnitOfWork:
User#42 → Object B
```

---

# 15. TAINTED

`TAINTED` será un estado operacional fuerte.

Significa:

> El contexto de persistencia ya no puede continuar realizando operaciones normales con garantías suficientes.

Ejemplo:

```text
UNKNOWN execution outcome
```

puede convertir:

```text
EntityManager
→ TAINTED
```

---

# 16. UNKNOWN

Se utiliza cuando un dominio todavía no ha sido evaluado o la evidencia disponible no permite clasificarlo con mayor precisión.

---

# 17. Consistency ≠ Freshness

Supongamos:

```php
$user = $entityManager->find(User::class, 10);
```

Después otro proceso modifica el registro.

El IdentityMap sigue correctamente garantizando:

```text
User#10
→ same canonical object
```

Por tanto:

```text
Identity Consistency = CONSISTENT
```

aunque:

```text
Database Freshness = STALE
```

---

# 18. Identity Consistency

La regla:

```text
EntityKey → one canonical managed object
```

debe mantenerse.

Formalmente:

```text
∀ key:
Managed(a,key) ∧ Managed(b,key)
⇒
a === b
```

---

# 19. Identity conflict

Si:

```text
User#42 → Object A
User#42 → Object B
```

dentro del mismo `PersistenceContext`:

```text
IdentityConsistency = INCONSISTENT
```

---

# 20. Identity namespace

La consistencia deberá considerar:

```text
EntityIdentityNamespace
```

por lo que:

```text
Tenant A / User#42
```

y:

```text
Tenant B / User#42
```

son identidades diferentes.

---

# 21. Entity State Consistency

`EntityState` deberá concordar con:

```text
IdentityMap
UnitOfWork
Persistence registration
```

Ejemplo:

```text
MANAGED entity
```

normalmente deberá estar registrada en:

```text
IdentityMap
UnitOfWork
```

cuando posea identidad establecida.

---

# 22. Invalid combination

Ejemplo:

```text
EntityState = MANAGED
IdentityMap = absent
UnitOfWork = absent
```

sin una política especial:

```text
EntityStateConsistency = INCONSISTENT
```

---

# 23. NEW entity

Una entidad `NEW` con ID generado por DB todavía puede no existir en IdentityMap bajo una identidad persistente.

Eso no constituye inconsistencia.

---

# 24. REMOVED entity

`REMOVED` significa:

```text
scheduled for removal
```

No:

```text
database row definitely absent
```

---

# 25. DETACHED entity

Una entidad detached:

```text
may continue existing as PHP object
```

pero ya no forma parte de las garantías del PersistenceContext.

---

# 26. Snapshot Consistency

Un snapshot representa:

```text
ORM baseline knowledge
```

no necesariamente el estado actual de la DB.

---

# 27. Snapshot invariant

Para una entidad managed limpia:

```text
CurrentPersistentState
=
SnapshotPersistentState
```

según la estrategia de change tracking.

---

# 28. Dirty entity

Para entidad dirty:

```text
CurrentPersistentState
≠
SnapshotPersistentState
```

es esperado.

No es inconsistencia.

---

# 29. Snapshot after successful update

Después de una actualización suficientemente cierta:

```text
Snapshot
←
PersistedEntityState
```

considerando valores generados.

---

# 30. Snapshot after unknown update

No deberá hacerse:

```text
Snapshot ← current entity
```

como si la actualización hubiera sido confirmada.

---

# 31. Rule

```text
UnknownPersistenceOutcome
→
DoNotFabricateNewBaseline
```

---

# 32. ChangeSet Consistency

Un ChangeSet deberá representar:

```text
difference(
    baseline snapshot,
    current persistent state
)
```

para estrategias snapshot-based.

---

# 33. Lifecycle mutation

Después de `preUpdate`:

```text
ChangeSet
```

debe recalcularse antes de congelar el plan.

---

# 34. Frozen ChangeSet

Una vez congelado el PersistencePlan:

```text
ChangeSet used by plan
```

debe permanecer estable durante ejecución.

---

# 35. PostUpdate mutation

Si `postUpdate` modifica una propiedad persistente:

```text
entity becomes dirty again
```

para un futuro flush.

No deberá modificar retroactivamente el plan ya ejecutado.

---

# 36. UnitOfWork Consistency

El UoW deberá mantener coherencia entre:

```text
registered entities
entity states
snapshots
change sets
scheduled operations
identity map
```

---

# 37. UoW consistency equation

Conceptualmente:

```text
UoWConsistency
=
StateRegistryConsistent
∧
IdentityRegistryConsistent
∧
SnapshotRegistryConsistent
∧
ChangeTrackingConsistent
∧
SchedulingConsistent
```

---

# 38. PersistencePlan Consistency

El plan debe corresponder al UoW estabilizado.

```text
PersistencePlan
=
Plan(
    StabilizedUnitOfWork
)
```

---

# 39. Plan stale

Si el UoW cambia después de generar el plan y antes de congelarlo:

```text
plan must be regenerated
```

o estabilizado nuevamente.

---

# 40. Frozen plan

Después de:

```text
PersistencePlan.freeze()
```

no se permitirá mutación arbitraria.

---

# 41. Plan fingerprint

Puede existir:

```text
PersistencePlanFingerprint
```

para diagnóstico y detección de mutaciones inesperadas.

---

# 42. Execution Consistency

La ejecución deberá conservar una relación:

```text
PersistenceOperationId
↔
ExecutionOutcome
```

---

# 43. ExecutionOutcome

Estados generales:

```php
enum PersistenceExecutionStatus
{
    case SUCCEEDED;
    case FAILED;
    case CONFLICT;
    case NOT_EXECUTED;
    case UNKNOWN;
}
```

---

# 44. UNKNOWN es first-class

Nunca:

```text
UNKNOWN → FAILED
```

automáticamente.

Tampoco:

```text
UNKNOWN → SUCCEEDED
```

---

# 45. Why

Una pérdida de conexión puede ocurrir:

```text
before execution
during execution
after execution
after server commit
before client acknowledgment
```

---

# 46. Evidence model

El sistema deberá conservar evidencia.

```php
final readonly class PersistenceEvidence
{
    public function __construct(
        public PersistenceOperationId $operationId,
        public PersistenceExecutionStatus $status,
        public OutcomeCertainty $certainty,
        public EvidenceCollection $evidence,
    ) {}
}
```

---

# 47. Outcome Certainty

```php
enum OutcomeCertainty
{
    case CERTAIN;
    case PARTIAL;
    case UNCERTAIN;
    case UNKNOWN;
}
```

---

# 48. Evidence examples

```text
driver acknowledgment
affected rows
RETURNING result
generated identifier
transaction state
rollback acknowledgment
commit acknowledgment
connection failure
server error
```

---

# 49. Database Knowledge Model

VoltStack no modelará únicamente:

```text
database state
```

sino:

```text
knowledge about database state
```

---

# 50. DatabaseKnowledgeStatus

```php
enum DatabaseKnowledgeStatus
{
    case OBSERVED;
    case INFERRED;
    case STALE;
    case UNCERTAIN;
    case UNKNOWN;
}
```

---

# 51. OBSERVED

Resultado directamente observado mediante una operación suficientemente confiable.

---

# 52. INFERRED

Derivado de garantías conocidas.

Ejemplo:

```text
UPDATE acknowledged
affectedRows = 1
```

puede permitir inferir cierto estado.

---

# 53. STALE

Observación válida en el pasado pero cuya frescura no está garantizada.

---

# 54. UNCERTAIN

Existe evidencia contradictoria o incompleta.

---

# 55. UNKNOWN

No existe conocimiento suficiente.

---

# 56. Database Knowledge ≠ Database Reality

Regla permanente:

```text
Knowledge(state)
≠
AbsoluteTruth(state)
```

---

# 57. Generated Identity Consistency

Después de insert:

```text
Entity
↔
GeneratedIdentifier
↔
EntityKey
↔
IdentityMap
↔
Snapshot
↔
UnitOfWork
```

deberán reconciliarse.

---

# 58. Safe generated identity reconciliation

```text
GeneratedIdentityReconciliationAllowed
=
InsertOutcomeSufficientlyCertain
∧
GeneratedIdentifierValid
∧
NoIdentityConflict
```

---

# 59. UNKNOWN insert

No deberá inventarse:

```text
generated ID
```

---

# 60. ID received before later rollback

Un caso diferente:

```text
INSERT returned ID 101
transaction later rolls back
```

El objeto puede continuar teniendo:

```text
id = 101
```

pero:

```text
ID Assigned
≠
Row Durable
```

---

# 61. Identifier rollback policy

VoltStack no deberá asumir universalmente que un ID generado puede "desasignarse".

Algunos generadores:

```text
sequences
auto increment
UUID
ULID
```

tienen semánticas diferentes.

---

# 62. GeneratedIdentityState

Puede modelarse:

```php
enum GeneratedIdentityState
{
    case UNASSIGNED;
    case ASSIGNED;
    case PERSISTENCE_CONFIRMED;
    case TRANSACTION_PENDING;
    case ROLLED_BACK;
    case OUTCOME_UNKNOWN;
}
```

Esto describe conocimiento de persistencia, no sustituye `EntityState`.

---

# 63. Relationship Consistency

Las relaciones ORM requieren coherencia entre:

```text
owning side
inverse side
foreign-key persistence
join-table persistence
entity state
```

---

# 64. Object graph consistency

El ORM podrá detectar:

```text
$user->orders contains Order#5
```

pero:

```text
$order->user !== $user
```

si la mapping policy exige bidirectional consistency.

---

# 65. Object consistency ≠ DB consistency

Aunque ambos lados estén sincronizados en memoria:

```text
relationship not flushed
```

todavía puede no existir en DB.

---

# 66. Relationship persistence consistency

Después de flush exitoso:

```text
relationship snapshot
```

deberá reflejar los cambios confirmados.

---

# 67. Partial relationship failure

Si:

```text
entity insert succeeded
join-table insert failed
```

la consistencia dependerá de la transacción.

---

# 68. With rollback certain

Si todo estaba dentro de una transacción y rollback fue confirmado:

```text
database effects reverted
```

pero el estado de objetos deberá reconciliarse.

---

# 69. Without transaction

Puede existir:

```text
partial durable state
```

---

# 70. Partial Persistence Consistency

Será un concepto first-class.

```php
enum PersistenceCompletion
{
    case NONE;
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 71. Partial execution

Ejemplo:

```text
Insert A = SUCCEEDED
Insert B = SUCCEEDED
Insert C = FAILED
```

sin transacción.

Resultado:

```text
PersistenceCompletion = PARTIAL
```

---

# 72. Partial does not mean total inconsistency

Puede conocerse exactamente:

```text
A persisted
B persisted
C not persisted
```

---

# 73. Selective reconciliation

VoltStack podrá reconciliar:

```text
A
B
```

y conservar:

```text
C
```

como pendiente/fallido según policy.

---

# 74. But

Si el EntityManager no puede garantizar una continuación segura:

```text
EntityManager → TAINTED
```

aunque algunos outcomes sean conocidos.

---

# 75. Transaction consistency

La persistencia deberá distinguir:

```text
statement outcome
```

de:

```text
transaction outcome
```

---

# 76. Transaction states relevant to persistence

Conceptualmente:

```text
NONE
ACTIVE
COMMITTING
COMMITTED
ROLLING_BACK
ROLLED_BACK
FAILED
UNKNOWN
```

---

# 77. Flush inside transaction

```php
$transaction->begin();

$entityManager->flush();
```

Después:

```text
PersistenceOperations = SUCCEEDED
Transaction = ACTIVE
```

No:

```text
Durable = true
```

---

# 78. Pre-commit consistency

El ORM puede alcanzar:

```text
transaction-local consistency
```

sin durabilidad.

---

# 79. Post-commit consistency

Después de:

```text
COMMIT acknowledged
```

puede afirmarse un nivel mayor de certeza.

---

# 80. Commit failure

No todos los errores de commit significan rollback.

---

# 81. Commit UNKNOWN

Ejemplo:

```text
COMMIT sent
connection lost
```

Resultado:

```text
TransactionOutcome = UNKNOWN
```

---

# 82. Critical rule

```text
CommitOutcomeUnknown
→
PersistenceContextTainted
```

por defecto.

---

# 83. Why

No puede saberse si la DB:

```text
committed
```

o:

```text
rolled back
```

---

# 84. Rollback acknowledged

Cuando rollback es cierto:

```text
DatabaseTransactionEffects = reverted
```

según las garantías de la DB.

Pero el objeto PHP sigue conteniendo mutaciones.

---

# 85. Rollback ≠ rewind objects

Regla:

```text
DatabaseRollback
≠
AutomaticObjectGraphRewind
```

---

# 86. Why

Restaurar objetos automáticamente puede:

- destruir cambios posteriores;
- ejecutar setters;
- romper invariantes de dominio;
- perder referencias;
- revertir efectos no persistentes;
- no poder revertir side effects.

---

# 87. Rollback reconciliation policy

Opciones:

```php
enum RollbackReconciliationPolicy
{
    case MARK_DIRTY;
    case DETACH_AFFECTED;
    case CLEAR_CONTEXT;
    case REQUIRE_REFRESH;
    case CUSTOM;
}
```

---

# 88. Default recommendation

Para V1:

```text
rollback after flushed changes
→
affected entities remain objects
→
persistence baselines invalidated/reconciled
→
context may require clear/refresh depending scenario
```

No hacer rewind mágico.

---

# 89. Snapshot after rollback

Si snapshot fue avanzado provisionalmente después del statement:

```text
rollback
```

requiere restaurar el baseline de persistencia anterior o invalidarlo.

---

# 90. Transaction-aware snapshots

Por ello puede ser útil distinguir:

```text
CommittedBaseline
TransactionLocalBaseline
```

---

# 91. Snapshot layering

Conceptualmente:

```text
CommittedSnapshot
       │
       ▼
TransactionLocalSnapshot
       │
       ▼
CurrentEntityState
```

---

# 92. V1 simplification

No necesariamente se necesitan dos objetos físicos.

Puede conservarse:

```text
pre-transaction snapshot evidence
```

en el transaction synchronization layer.

---

# 93. Flush success inside transaction

No debe destruir la información necesaria para reconciliar un futuro rollback.

---

# 94. Transaction synchronization

Persistence Consistency deberá integrarse con:

```text
TransactionSynchronization
```

para recibir:

```text
afterCommit
afterRollback
afterCompletion
unknownCompletion
```

---

# 95. Flush ≠ transaction synchronization

El Flush System no será propietario de la transacción.

---

# 96. Consistency checkpoint

Se podrá representar:

```php
final readonly class PersistenceConsistencyCheckpoint
{
    public function __construct(
        public PersistenceConsistencyCheckpointId $id,
        public UnitOfWorkVersion $unitOfWorkVersion,
        public PersistencePlanId $planId,
        public TransactionContextId $transactionId,
        public ConsistencyVector $consistency,
    ) {}
}
```

---

# 97. Purpose

Permite saber:

```text
what the ORM believed
at a specific persistence boundary
```

---

# 98. Consistency Vector

En lugar de un booleano:

```php
final readonly class ConsistencyVector
{
    public function __construct(
        public PersistenceConsistencyStatus $identity,
        public PersistenceConsistencyStatus $entityState,
        public PersistenceConsistencyStatus $snapshot,
        public PersistenceConsistencyStatus $changeSet,
        public PersistenceConsistencyStatus $unitOfWork,
        public PersistenceConsistencyStatus $relationships,
        public PersistenceConsistencyStatus $execution,
        public PersistenceConsistencyStatus $transaction,
        public PersistenceConsistencyStatus $databaseKnowledge,
    ) {}
}
```

---

# 99. Global consistency

Puede derivarse:

```text
GlobalConsistency
=
WorstRelevantDomainStatus(
    ConsistencyVector
)
```

pero sin perder el vector original.

---

# 100. Severity ordering

No necesariamente debe ser un simple ordinal, pero conceptualmente:

```text
CONSISTENT
   ↓
STALE
   ↓
UNCERTAIN
   ↓
INCONSISTENT
   ↓
TAINTED
```

`UNKNOWN` se maneja según contexto.

---

# 101. Tainted PersistenceContext

Un contexto tainted deberá rechazar operaciones peligrosas.

Ejemplo:

```php
$entityManager->flush();
```

podría producir:

```text
TaintedPersistenceContextException
```

---

# 102. Allowed operations when tainted

Podrían permitirse:

```text
diagnostics
close
clear
detach
consistency inspection
recovery procedure
```

según causa.

---

# 103. find() when tainted

Por defecto deberá evitarse continuar usando el contexto como si nada hubiera ocurrido.

---

# 104. Taint reason

```php
enum PersistenceContextTaintReason
{
    case UNKNOWN_EXECUTION_OUTCOME;
    case UNKNOWN_TRANSACTION_OUTCOME;
    case IDENTITY_CONFLICT;
    case RECONCILIATION_FAILURE;
    case PARTIAL_PERSISTENCE;
    case SNAPSHOT_CORRUPTION;
    case INTERNAL_INVARIANT_FAILURE;
    case CROSS_CONTEXT_CONTAMINATION;
}
```

---

# 105. Taint evidence

Siempre deberá conservarse:

```text
reason
operation
transaction
timestamp
diagnostic context
```

sin almacenar datos sensibles innecesarios.

---

# 106. Recovery

No todo taint será recuperable.

---

# 107. Recovery strategies

```php
enum PersistenceRecoveryStrategy
{
    case CLEAR_CONTEXT;
    case RELOAD_AFFECTED;
    case REBUILD_CONTEXT;
    case CLOSE_ENTITY_MANAGER;
    case APPLICATION_RECONCILIATION_REQUIRED;
}
```

---

# 108. Unknown transaction outcome

Recomendación:

```text
close current EntityManager
```

y resolver el estado mediante un nuevo contexto.

---

# 109. Why new context

El viejo contexto contiene suposiciones potencialmente inválidas sobre:

```text
snapshots
entity states
generated IDs
relationships
database effects
```

---

# 110. Recovery by reread

Cuando sea seguro:

```text
New PersistenceContext
        ↓
Read authoritative database state
        ↓
Application reconciliation
```

---

# 111. But

Una reread tampoco siempre resuelve:

```text
was external side effect executed?
```

Ejemplo trigger que envía trabajo externo.

Eso queda fuera de las garantías ORM.

---

# 112. External database mutations

El ORM puede quedar stale debido a:

```text
other application instance
SQL console
stored procedure
trigger
raw SQL
bulk query
background process
```

---

# 113. IdentityMap behavior

No se invalidará automáticamente ante cambios externos desconocidos.

---

# 114. Why

Detectarlos universalmente requeriría:

```text
continuous database change tracking
```

que no pertenece al IdentityMap.

---

# 115. Explicit freshness mechanisms

VoltStack deberá ofrecer:

```text
refresh(entity)
clear()
detach()
reload()
query with refresh policy
```

---

# 116. Refresh

```text
refresh(entity)
```

deberá actualizar la instancia canónica existente.

No crear otra entidad con el mismo `EntityKey`.

---

# 117. Refresh consistency

Después de refresh exitoso:

```text
entity
snapshot
change tracking baseline
```

deben reconciliarse.

---

# 118. Refresh with dirty entity

Deberá existir policy explícita.

Ejemplos:

```php
enum DirtyRefreshPolicy
{
    case REJECT;
    case DISCARD_LOCAL_CHANGES;
    case MERGE;
}
```

---

# 119. Default

`REJECT` o requerir explicit discard será la opción más segura.

---

# 120. Raw SQL

Ejemplo:

```php
$connection->execute(
    'UPDATE users SET status = ? WHERE id = ?',
    [...]
);
```

El ORM no sabe automáticamente qué entidades fueron afectadas.

---

# 121. Raw SQL invalidation

Podrá marcar:

```text
PersistenceContext freshness = potentially stale
```

cuando raw SQL se ejecuta mediante una conexión asociada al contexto.

---

# 122. Granularity

Dependiendo de metadata disponible:

```text
GLOBAL
ENTITY_TYPE
ENTITY_KEY
UNKNOWN_SCOPE
```

---

# 123. Invalidation Scope

```php
enum PersistenceInvalidationScope
{
    case ENTITY;
    case ENTITY_TYPE;
    case TABLE;
    case PERSISTENCE_CONTEXT;
    case UNKNOWN;
}
```

---

# 124. Query Builder bulk operations

Igualmente:

```text
bulk UPDATE
bulk DELETE
```

pueden invalidar managed entities.

---

# 125. Bulk policy

Propuesta:

```php
enum BulkPersistenceConsistencyPolicy
{
    case MARK_STALE;
    case DETACH_AFFECTED;
    case CLEAR_AFFECTED_TYPE;
    case CLEAR_CONTEXT;
    case REJECT_WHEN_MANAGED_ENTITIES_CONFLICT;
}
```

---

# 126. Default V1

Preferir:

```text
mark affected managed representations stale
```

cuando el scope pueda conocerse.

Si no:

```text
mark context potentially stale
```

---

# 127. Bulk operation ≠ lifecycle

Como ya se estableció:

```text
bulk query
```

no dispara lifecycle por cada entidad.

---

# 128. Bulk operation ≠ snapshot synchronization

No actualizar snapshots intentando reconstruir cambios sin hidratar evidencia suficiente.

---

# 129. External transaction changes

Si otra conexión cambia los datos:

```text
VoltStack may not know
```

hasta:

```text
refresh
new query bypassing IdentityMap
transaction isolation boundary
```

según el caso.

---

# 130. Isolation level

El Consistency System consume información del Transaction System.

No redefine:

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

---

# 131. ORM consistency ≠ transaction isolation

Son dimensiones diferentes.

---

# 132. Replica consistency

También:

```text
primary state
≠
replica immediately visible state
```

---

# 133. Persistence consistency and replicas

Las escrituras se consideran respecto al target de escritura.

La visibilidad posterior en réplica pertenece a:

```text
Replica Lag Awareness
Read/Write Routing
Sticky Connection
```

documentos posteriores.

---

# 134. Batch Persistence Consistency

Batching no cambia el modelo.

```text
BatchExecutionOutcome
        ↓
IndividualOperationOutcome[]
        ↓
Consistency Reconciliation
```

---

# 135. Batch partial failure

Debe conservar:

```text
A = SUCCEEDED
B = SUCCEEDED
C = FAILED
D = NOT_EXECUTED
```

si existe evidencia.

---

# 136. Batch unknown

Si:

```text
batch outcome = UNKNOWN
```

y no hay correlación individual:

```text
all affected operations
→ UNKNOWN
```

---

# 137. No batch optimism

Nunca:

```text
99 expected successes
1 failure
→ assume first 99 succeeded
```

sin evidencia.

---

# 138. Lifecycle Consistency

`postPersist`, `postUpdate`, `postRemove` solo ocurren cuando la operación correspondiente alcanzó la certeza semántica definida.

---

# 139. But

```text
postUpdate
≠
afterCommit
```

---

# 140. External side effects

Para:

```text
email
webhook
broker message
external API
```

utilizar:

```text
after-commit integration
transactional outbox
jobs after commit
```

---

# 141. Domain Events

Un domain event registrado durante cambios de entidad:

```text
Recorded
```

no necesariamente:

```text
Published
```

---

# 142. Consistency boundary for external effects

```text
ExternalDurableEffectAllowed
=
TransactionCommitCertain
∧
EffectPolicySatisfied
```

salvo arquitecturas explícitamente idempotentes/outbox.

---

# 143. Consistency Assertions

El sistema deberá poder ejecutar verificaciones internas.

```php
interface PersistenceConsistencyChecker
{
    public function check(
        PersistenceContext $context
    ): PersistenceConsistencyReport;
}
```

---

# 144. Check levels

```php
enum ConsistencyCheckLevel
{
    case LIGHT;
    case STANDARD;
    case DEEP;
}
```

---

# 145. LIGHT

Hot-path-safe:

```text
basic IdentityMap/UoW indexes
state registration
taint status
```

---

# 146. STANDARD

Puede verificar:

```text
snapshots
ChangeSets
scheduled operations
identity consistency
```

---

# 147. DEEP

Development/testing:

```text
full graph consistency
metadata consistency
relationship checks
plan correlation
reverse indexes
```

---

# 148. No hidden DB queries

Por defecto:

```text
consistency checker
```

no consulta la DB.

---

# 149. Database verification

Si se desea comparar contra DB será una operación explícita diferente:

```text
verifyAgainstDatabase()
```

---

# 150. Why

Una consulta DB:

- cuesta;
- puede observar otro instante;
- depende de isolation;
- puede alterar locks;
- puede no ser autoritativa en réplica.

---

# 151. Consistency Report

```php
final readonly class PersistenceConsistencyReport
{
    public function __construct(
        public ConsistencyVector $vector,
        public array $violations,
        public array $warnings,
        public array $uncertainties,
        public array $recoverySuggestions,
    ) {}
}
```

---

# 152. Violation

```php
final readonly class PersistenceConsistencyViolation
{
    public function __construct(
        public PersistenceConsistencyDomain $domain,
        public string $code,
        public string $message,
        public ConsistencySeverity $severity,
    ) {}
}
```

---

# 153. Invariant checking in production

Solo checks baratos en hot path.

Checks profundos:

```text
development
testing
diagnostic mode
```

---

# 154. Persistence generation

El UoW puede mantener:

```text
PersistenceGeneration
```

incrementada por flush/reconciliation.

---

# 155. Why

Permite detectar:

```text
snapshot from generation N
plan from generation N-1
```

---

# 156. UnitOfWorkVersion

Cada mutación estructural puede incrementar:

```text
UnitOfWorkVersion
```

---

# 157. Plan source version

`PersistencePlan` guarda:

```text
sourceUnitOfWorkVersion
```

---

# 158. Before execution

Se verifica:

```text
CurrentUoWVersion
=
PlanSourceVersion
```

salvo cambios explícitamente absorbidos por stabilization.

---

# 159. Stale plan

Si no:

```text
StalePersistencePlanException
```

---

# 160. Snapshot generation

Cada snapshot puede asociarse a:

```text
SnapshotGeneration
```

para diagnóstico.

---

# 161. Consistency token

Podría existir:

```php
final readonly class PersistenceConsistencyToken
{
    public function __construct(
        public PersistenceContextId $context,
        public PersistenceGeneration $generation,
        public UnitOfWorkVersion $unitOfWorkVersion,
    ) {}
}
```

---

# 162. Internal atomic reconciliation

Cuando una operación se confirma, la actualización de:

```text
IdentityMap
EntityState
Snapshot
UnitOfWork
```

debe comportarse como una transición lógica atómica.

---

# 163. Why

No queremos:

```text
snapshot updated
↓
exception
↓
EntityState still DIRTY
```

dejando estructuras internas contradictorias.

---

# 164. Reconciliation Transaction

No es una DB transaction.

Es una transición interna:

```text
InMemoryReconciliationTransaction
```

---

# 165. In-memory rollback

Si falla antes de completar:

```text
restore internal registries
```

cuando sea posible.

---

# 166. If internal rollback fails

```text
PersistenceContext → TAINTED
```

---

# 167. Reconciliation ordering

Propuesta conceptual:

```text
validate outcome
    ↓
validate generated values
    ↓
validate identity conflicts
    ↓
prepare reconciliation mutations
    ↓
apply internal transition
    ↓
verify invariants
    ↓
publish semantic post lifecycle
```

---

# 168. Lifecycle failure after reconciliation

Como se definió anteriormente:

```text
PersistenceOperationOutcome = SUCCEEDED
LifecycleOutcome = FAILED
```

son dimensiones diferentes.

---

# 169. Do not undo known DB success

Si `postUpdate` falla después de UPDATE confirmado:

```text
do not pretend UPDATE failed
```

---

# 170. Context may still fail flush

Sí.

```text
FlushOutcome = FAILED
```

puede coexistir con:

```text
PersistenceOperationOutcome = SUCCEEDED
```

---

# 171. Transaction may rollback later

Si hay transaction:

```text
known statement success
```

puede revertirse posteriormente.

---

# 172. Consistency Timeline

Ejemplo:

```text
T0 Entity clean
   ORM: CONSISTENT

T1 Entity modified
   Change tracking: DIRTY
   ORM: CONSISTENT

T2 Flush plan created
   Plan: CONSISTENT

T3 UPDATE succeeded
   Execution: SUCCEEDED

T4 Snapshot reconciled
   ORM: CONSISTENT
   Transaction: ACTIVE

T5 COMMIT succeeded
   Transaction: COMMITTED
   Durability knowledge: CONFIRMED
```

---

# 173. Unknown timeline

```text
T0 Entity clean

T1 Entity modified

T2 UPDATE sent

T3 connection lost

T4 outcome unknown

Execution: UNKNOWN
DatabaseKnowledge: UNCERTAIN
PersistenceContext: TAINTED
```

---

# 174. Rollback timeline

```text
T0 baseline A

T1 entity becomes B

T2 UPDATE B succeeds

T3 transaction rollback succeeds

Database returns to A
Entity object remains B
```

El ORM deberá representar:

```text
database baseline = A
object state = B
```

como un cambio pendiente/reconciliation case.

---

# 175. This is not inconsistency by definition

Puede convertirse correctamente en:

```text
Entity = DIRTY
Snapshot = A
Current = B
```

permitiendo un nuevo flush si policy/context lo permite.

---

# 176. But generated inserts are harder

```text
NEW entity
INSERT
generated id 42
ROLLBACK
```

El objeto:

```text
id = 42
```

pero row no existe.

---

# 177. Entity state after rolled-back insert

Debe ser gobernado por policy.

Opciones conceptuales:

```text
NEW_WITH_ASSIGNED_ID
DETACHED
REQUIRES_REPERSIST
```

No introducir necesariamente nuevos estados públicos; puede expresarse mediante metadata de reconciliación.

---

# 178. Recommended rule

```text
Rollback does not erase generated identity blindly.
```

La UoW policy decidirá si la entidad puede reinsertarse con ese ID o requiere regeneración.

---

# 179. Rolled-back delete

Ejemplo:

```text
Entity MANAGED
DELETE succeeds
state reconciled
ROLLBACK
```

DB conserva row.

El contexto deberá poder restaurar una representación administrada coherente o requerir clear/reload.

---

# 180. Complexity boundary

No intentar hacer un "time machine ORM".

---

# 181. Recommended V1 rollback strategy

Para transacciones con flush previo y rollback:

```text
simple updates:
    restore persistence baseline where reliable

complex inserts/deletes/generated values:
    mark affected context for reconciliation

unknown:
    taint/close
```

---

# 182. Consistency Policy

```php
final readonly class PersistenceConsistencyPolicy
{
    public function __construct(
        public UnknownOutcomePolicy $unknownOutcome,
        public PartialFailurePolicy $partialFailure,
        public RollbackReconciliationPolicy $rollback,
        public BulkPersistenceConsistencyPolicy $bulkOperations,
        public DirtyRefreshPolicy $dirtyRefresh,
    ) {}
}
```

---

# 183. Strict mode

VoltStack podrá ofrecer:

```text
STRICT
BALANCED
PERFORMANCE
```

pero nunca permitir que `PERFORMANCE` convierta UNKNOWN en success.

---

# 184. Strict

Puede:

```text
taint on partial execution
reject dirty refresh
clear after bulk mutation
perform additional assertions
```

---

# 185. Balanced

Defaults razonables para producción.

---

# 186. Performance

Reduce verificaciones profundas, pero conserva invariantes fundamentales.

---

# 187. Safety floor

Ningún mode podrá desactivar:

```text
identity uniqueness
tenant isolation
unknown outcome preservation
transaction outcome separation
```

---

# 188. Persistent Runtime

VoltStack está diseñado para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 189. Shared immutable state

Puede compartirse:

```text
consistency rules
compiled metadata
immutable policies
checker definitions
diagnostic code definitions
```

---

# 190. Scoped mutable state

Siempre:

```text
ConsistencyVector
PersistenceContext taint
UnitOfWorkVersion
PersistenceGeneration
ConsistencyCheckpoint
transaction correlation
entity references
snapshots
uncertainties
```

---

# 191. Forbidden

```php
static PersistenceConsistencyStatus $currentStatus;
```

---

# 192. Request boundary

Al finalizar request:

```text
PersistenceContext
EntityManager
UnitOfWork
IdentityMap
ConsistencyState
```

deberán limpiarse.

---

# 193. No implicit recovery

Request reset no debe ocultar:

```text
unknown transaction outcome
```

antes de que sea diagnosticado correctamente.

---

# 194. Logging before reset

Errores críticos de consistencia deberán emitirse a telemetry antes del cleanup.

---

# 195. FrankenPHP

El worker puede sobrevivir.

El persistence context no.

---

# 196. RoadRunner

Misma regla.

---

# 197. OpenSwoole

Consistency state será coroutine/logical-context scoped.

---

# 198. Concurrent access

Un mismo PersistenceContext no será thread/coroutine-safe por defecto.

---

# 199. Concurrent mutation

Si se detecta:

```text
ConcurrentPersistenceContextMutation
```

se deberá rechazar.

---

# 200. Why

Dos operaciones simultáneas podrían romper:

```text
UoWVersion
Snapshots
IdentityMap
PersistencePlan
Reconciliation
```

---

# 201. Telemetry

Métricas:

```text
orm.persistence.consistency.checks
orm.persistence.consistency.violations
orm.persistence.consistency.stale
orm.persistence.consistency.uncertain
orm.persistence.consistency.tainted

orm.persistence.reconciliation.total
orm.persistence.reconciliation.failed

orm.persistence.outcome.unknown
orm.persistence.outcome.partial

orm.persistence.rollback.reconciliation

orm.persistence.bulk.invalidation
orm.persistence.raw_sql.invalidation

orm.persistence.recovery.attempt
orm.persistence.recovery.success
orm.persistence.recovery.failure
```

---

# 202. Taint reasons telemetry

Cardinalidad limitada:

```text
unknown_execution
unknown_transaction
identity_conflict
partial_execution
reconciliation_failure
invariant_failure
```

---

# 203. No sensitive labels

Nunca:

```text
entity_id
email
raw SQL parameters
tenant secret
```

como metric labels.

---

# 204. Tracing

```text
orm.flush
 ├── consistency.precheck
 ├── persistence.plan
 ├── persistence.execute
 ├── consistency.reconcile
 ├── consistency.verify
 └── transaction.sync
```

---

# 205. Debug Toolbar

Ejemplo:

```text
ORM Persistence Consistency

Context:
  Status: CONSISTENT
  Generation: 18
  UoW Version: 47

Identity:
  CONSISTENT

Snapshots:
  CONSISTENT

Change Sets:
  CONSISTENT

Database Knowledge:
  STALE

Transaction:
  NONE

Managed entities: 32
Dirty entities:    3
Stale entities:    2

Unknown outcomes:  0
Taint reasons:     none
```

---

# 206. Tainted diagnostic

```text
ORM Persistence Context

STATUS: TAINTED

Reason:
  UNKNOWN_TRANSACTION_OUTCOME

Transaction:
  tx-8F21

Last operation:
  COMMIT

Evidence:
  connection lost after commit request

Safe continuation:
  NO

Recommended action:
  close current EntityManager
  create new PersistenceContext
  reconcile application state
```

---

# 207. Exceptions

Jerarquía propuesta:

```text
DatabasePersistenceConsistencyException
├── PersistenceConsistencyViolationException
├── PersistenceConsistencyUnknownException
├── PersistenceContextTaintedException
├── PersistenceContextInconsistentException
├── PersistenceIdentityConsistencyException
├── PersistenceEntityStateConsistencyException
├── PersistenceSnapshotConsistencyException
├── PersistenceChangeSetConsistencyException
├── PersistenceUnitOfWorkConsistencyException
├── PersistenceRelationshipConsistencyException
├── PersistencePlanConsistencyException
├── StalePersistencePlanException
├── PersistenceExecutionConsistencyException
├── PersistenceOutcomeUnknownException
├── PersistencePartialOutcomeException
├── PersistenceGeneratedIdentityConsistencyException
├── PersistenceTransactionConsistencyException
├── PersistenceTransactionOutcomeUnknownException
├── PersistenceRollbackReconciliationException
├── PersistenceBulkInvalidationException
├── PersistenceRawSqlInvalidationException
├── PersistenceRefreshConsistencyException
├── PersistenceReconciliationException
├── PersistenceReconciliationRollbackException
├── PersistenceRecoveryException
├── PersistenceConcurrentAccessException
├── PersistenceRuntimeIsolationException
└── PersistenceConsistencyInvariantException
```

---

# 208. Directory Structure

```text
src/Quantum/Database/ORM/Persistence/Consistency/
│
├── Contract/
│   ├── PersistenceConsistencyChecker.php
│   ├── PersistenceConsistencyManager.php
│   ├── PersistenceReconciler.php
│   └── PersistenceRecoveryManager.php
│
├── Model/
│   ├── PersistenceConsistencyStatus.php
│   ├── PersistenceConsistencyDomain.php
│   ├── ConsistencyVector.php
│   ├── PersistenceCompletion.php
│   ├── PersistenceGeneration.php
│   ├── PersistenceConsistencyToken.php
│   ├── PersistenceConsistencyCheckpoint.php
│   └── PersistenceConsistencyCheckpointId.php
│
├── Knowledge/
│   ├── DatabaseKnowledgeStatus.php
│   ├── DatabaseKnowledge.php
│   ├── OutcomeCertainty.php
│   ├── PersistenceEvidence.php
│   └── EvidenceCollection.php
│
├── Check/
│   ├── DefaultPersistenceConsistencyChecker.php
│   ├── ConsistencyCheckLevel.php
│   ├── PersistenceConsistencyReport.php
│   ├── PersistenceConsistencyViolation.php
│   └── ConsistencySeverity.php
│
├── Taint/
│   ├── PersistenceContextTaint.php
│   ├── PersistenceContextTaintReason.php
│   └── PersistenceContextTaintRegistry.php
│
├── Reconciliation/
│   ├── DefaultPersistenceReconciler.php
│   ├── ReconciliationPlan.php
│   ├── ReconciliationOperation.php
│   ├── ReconciliationResult.php
│   └── InMemoryReconciliationTransaction.php
│
├── Transaction/
│   ├── PersistenceTransactionSynchronizer.php
│   ├── TransactionPersistenceCheckpoint.php
│   ├── RollbackReconciliationPolicy.php
│   └── TransactionOutcomeReconciler.php
│
├── Invalidation/
│   ├── PersistenceInvalidationScope.php
│   ├── PersistenceInvalidation.php
│   ├── BulkPersistenceConsistencyPolicy.php
│   ├── RawSqlPersistenceInvalidator.php
│   └── BulkPersistenceInvalidator.php
│
├── Refresh/
│   ├── DirtyRefreshPolicy.php
│   └── RefreshConsistencyCoordinator.php
│
├── Recovery/
│   ├── PersistenceRecoveryStrategy.php
│   ├── PersistenceRecoveryPlan.php
│   └── DefaultPersistenceRecoveryManager.php
│
├── Policy/
│   ├── PersistenceConsistencyPolicy.php
│   ├── UnknownOutcomePolicy.php
│   └── PartialFailurePolicy.php
│
├── Runtime/
│   ├── PersistenceConsistencyRuntimeContext.php
│   └── PersistenceConsistencyRuntimeResetter.php
│
├── Telemetry/
│   ├── PersistenceConsistencyTelemetry.php
│   └── PersistenceConsistencyDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 209. Testing Strategy

La consistencia deberá probarse mediante invariantes, no solo mediante happy paths.

---

# 210. Identity consistency test

Dos resoluciones del mismo `EntityKey`:

```text
must return same object
```

---

# 211. Cross-tenant identity

Mismo ID en tenants diferentes:

```text
must not collide
```

---

# 212. Managed state consistency

Entidad MANAGED aparece correctamente en registros asociados.

---

# 213. NEW entity consistency

Entidad NEW sin generated ID no produce falso identity conflict.

---

# 214. Dirty snapshot

Current != snapshot se interpreta como dirty, no inconsistent.

---

# 215. Clean snapshot

Current == snapshot después de reconciliación exitosa.

---

# 216. preUpdate mutation

ChangeSet se recalcula.

---

# 217. postUpdate mutation

Entidad queda dirty para siguiente flush.

---

# 218. Plan version

Plan construido con UoW version antigua se rechaza.

---

# 219. Successful insert

Reconcilia:

```text
ID
IdentityMap
EntityState
Snapshot
UoW
```

---

# 220. Unknown insert

No fabrica generated ID.

---

# 221. Successful update

Snapshot avanza correctamente.

---

# 222. Unknown update

Snapshot no avanza falsamente.

---

# 223. Successful delete

Reconciliación coherente.

---

# 224. Unknown delete

No afirma ausencia del row.

---

# 225. Partial batch

Solo items confirmados se reconcilian.

---

# 226. Unknown batch

Propaga incertidumbre.

---

# 227. Flush inside transaction

No afirma durability.

---

# 228. Commit success

Actualiza transaction knowledge.

---

# 229. Commit unknown

Taints PersistenceContext.

---

# 230. Rollback update

No hace magical object rewind.

---

# 231. Rollback insert

Generated identity se maneja según policy.

---

# 232. Rollback delete

Contexto se reconcilia o marca para recuperación.

---

# 233. Raw SQL

Managed representations afectadas quedan stale según scope.

---

# 234. Bulk UPDATE

No actualiza snapshots inventando valores.

---

# 235. Bulk DELETE

No elimina silenciosamente entidades del IdentityMap sin policy.

---

# 236. Refresh clean entity

Actualiza objeto y snapshot.

---

# 237. Refresh dirty entity

Respeta DirtyRefreshPolicy.

---

# 238. External mutation

IdentityMap sigue canónico aunque freshness sea stale.

---

# 239. Tainted context

Nuevo flush se rechaza.

---

# 240. Clear tainted context

Libera estado de forma determinista según recovery policy.

---

# 241. Runtime isolation

Request A no afecta consistency state de B.

---

# 242. Coroutine isolation

Contexts independientes.

---

# 243. Deep checker

Detecta inconsistencia entre forward/reverse registries.

---

# 244. No hidden query

Consistency checker estándar no consulta DB.

---

# 245. Reconciliation failure

Contexto queda tainted.

---

# 246. Telemetry

No expone datos sensibles.

---

# 247. Architectural Invariants

## DB-ORM-CONSISTENCY-001

Persistence consistency no será un booleano simplista.

## DB-ORM-CONSISTENCY-002

VoltStack distinguirá consistencia interna de certeza sobre DB.

## DB-ORM-CONSISTENCY-003

ORM consistency no implicará database freshness.

## DB-ORM-CONSISTENCY-004

ORM consistency no implicará durability.

## DB-ORM-CONSISTENCY-005

Statement success no implicará commit.

## DB-ORM-CONSISTENCY-006

Flush success no implicará commit.

## DB-ORM-CONSISTENCY-007

Commit requested no implicará commit success.

## DB-ORM-CONSISTENCY-008

DatabaseReality será distinta de ORMKnowledge.

## DB-ORM-CONSISTENCY-009

Persistence consistency será evaluable por dominios.

## DB-ORM-CONSISTENCY-010

Identity consistency será first-class.

## DB-ORM-CONSISTENCY-011

Snapshot consistency será first-class.

## DB-ORM-CONSISTENCY-012

ChangeSet consistency será first-class.

## DB-ORM-CONSISTENCY-013

UnitOfWork consistency será first-class.

## DB-ORM-CONSISTENCY-014

Relationship consistency será first-class.

## DB-ORM-CONSISTENCY-015

Execution consistency será first-class.

## DB-ORM-CONSISTENCY-016

Transaction consistency será first-class.

## DB-ORM-CONSISTENCY-017

Database knowledge será first-class.

## DB-ORM-CONSISTENCY-018

CONSISTENT no significará eternamente fresh.

## DB-ORM-CONSISTENCY-019

STALE será distinto de INCONSISTENT.

## DB-ORM-CONSISTENCY-020

UNCERTAIN será distinto de FAILED.

## DB-ORM-CONSISTENCY-021

UNKNOWN será preservado.

## DB-ORM-CONSISTENCY-022

TAINTED representará contexto no seguro para continuación normal.

## DB-ORM-CONSISTENCY-023

IdentityMap mantendrá una instancia canónica por EntityKey.

## DB-ORM-CONSISTENCY-024

Identity consistency incluirá namespace.

## DB-ORM-CONSISTENCY-025

EntityState deberá concordar con registros ORM aplicables.

## DB-ORM-CONSISTENCY-026

NEW sin generated ID no será inconsistente por ausencia del IdentityMap.

## DB-ORM-CONSISTENCY-027

REMOVED no significará row definitivamente eliminado.

## DB-ORM-CONSISTENCY-028

DETACHED quedará fuera de garantías managed.

## DB-ORM-CONSISTENCY-029

Snapshot representará baseline ORM, no verdad absoluta de DB.

## DB-ORM-CONSISTENCY-030

Dirty entity podrá diferir de snapshot sin inconsistencia.

## DB-ORM-CONSISTENCY-031

Unknown update no avanzará snapshot falsamente.

## DB-ORM-CONSISTENCY-032

ChangeSet corresponderá a la estrategia de tracking configurada.

## DB-ORM-CONSISTENCY-033

preUpdate mutation obligará a recomputar ChangeSet.

## DB-ORM-CONSISTENCY-034

postUpdate mutation no modificará retroactivamente el plan ejecutado.

## DB-ORM-CONSISTENCY-035

UoW mantendrá coherencia entre sus registros.

## DB-ORM-CONSISTENCY-036

PersistencePlan procederá de UoW estabilizado.

## DB-ORM-CONSISTENCY-037

Frozen PersistencePlan será inmutable.

## DB-ORM-CONSISTENCY-038

Stale PersistencePlan será rechazado.

## DB-ORM-CONSISTENCY-039

PersistenceOperation conservará correlación con ExecutionOutcome.

## DB-ORM-CONSISTENCY-040

UNKNOWN execution outcome no será convertido a success.

## DB-ORM-CONSISTENCY-041

UNKNOWN execution outcome no será convertido a failure sin evidencia.

## DB-ORM-CONSISTENCY-042

Execution evidence será preservada.

## DB-ORM-CONSISTENCY-043

Database knowledge podrá ser OBSERVED.

## DB-ORM-CONSISTENCY-044

Database knowledge podrá ser INFERRED.

## DB-ORM-CONSISTENCY-045

Database knowledge podrá ser STALE.

## DB-ORM-CONSISTENCY-046

Database knowledge podrá ser UNCERTAIN.

## DB-ORM-CONSISTENCY-047

Database knowledge podrá ser UNKNOWN.

## DB-ORM-CONSISTENCY-048

Generated identity se reconciliará solo con evidencia suficiente.

## DB-ORM-CONSISTENCY-049

UNKNOWN insert no fabricará generated identity.

## DB-ORM-CONSISTENCY-050

Identifier assigned no implicará row durable.

## DB-ORM-CONSISTENCY-051

Rollback no eliminará generated IDs arbitrariamente.

## DB-ORM-CONSISTENCY-052

Relationship object consistency será distinta de DB consistency.

## DB-ORM-CONSISTENCY-053

Relationship snapshots se actualizarán según outcomes confirmados.

## DB-ORM-CONSISTENCY-054

Partial persistence será first-class.

## DB-ORM-CONSISTENCY-055

Partial persistence no se reducirá automáticamente a total failure.

## DB-ORM-CONSISTENCY-056

Selective reconciliation requerirá outcomes ciertos.

## DB-ORM-CONSISTENCY-057

Contexto podrá taintarse aun con outcomes parciales conocidos.

## DB-ORM-CONSISTENCY-058

Statement outcome será distinto de transaction outcome.

## DB-ORM-CONSISTENCY-059

Flush dentro de transaction no afirmará durability.

## DB-ORM-CONSISTENCY-060

Pre-commit consistency será distinta de post-commit certainty.

## DB-ORM-CONSISTENCY-061

Unknown commit outcome será first-class.

## DB-ORM-CONSISTENCY-062

Unknown commit outcome taintará el contexto por defecto.

## DB-ORM-CONSISTENCY-063

Database rollback no implicará object graph rewind.

## DB-ORM-CONSISTENCY-064

Rollback reconciliation será explícita.

## DB-ORM-CONSISTENCY-065

Flush dentro de transaction preservará información necesaria para rollback reconciliation.

## DB-ORM-CONSISTENCY-066

Persistence consistency se integrará con transaction synchronization.

## DB-ORM-CONSISTENCY-067

Flush no será propietario del commit.

## DB-ORM-CONSISTENCY-068

Consistency checkpoints serán opcionalmente observables.

## DB-ORM-CONSISTENCY-069

Consistency vector no será destruido al calcular global status.

## DB-ORM-CONSISTENCY-070

Tainted context rechazará operaciones inseguras.

## DB-ORM-CONSISTENCY-071

Taint conservará reason.

## DB-ORM-CONSISTENCY-072

Taint conservará evidencia diagnóstica.

## DB-ORM-CONSISTENCY-073

Recovery será explícito.

## DB-ORM-CONSISTENCY-074

No todo taint será recuperable.

## DB-ORM-CONSISTENCY-075

Unknown transaction outcome preferirá nuevo PersistenceContext.

## DB-ORM-CONSISTENCY-076

External DB mutation podrá volver stale al ORM.

## DB-ORM-CONSISTENCY-077

IdentityMap no detectará automáticamente cambios externos.

## DB-ORM-CONSISTENCY-078

Refresh será explícito.

## DB-ORM-CONSISTENCY-079

Refresh conservará la instancia canónica.

## DB-ORM-CONSISTENCY-080

Dirty refresh tendrá policy explícita.

## DB-ORM-CONSISTENCY-081

Raw SQL podrá invalidar conocimiento ORM.

## DB-ORM-CONSISTENCY-082

Raw SQL no actualizará snapshots mágicamente.

## DB-ORM-CONSISTENCY-083

Bulk operations podrán invalidar managed representations.

## DB-ORM-CONSISTENCY-084

Bulk operations no dispararán lifecycle per entity.

## DB-ORM-CONSISTENCY-085

Bulk operations no reconstruirán snapshots sin evidencia.

## DB-ORM-CONSISTENCY-086

Isolation level será distinto de ORM consistency.

## DB-ORM-CONSISTENCY-087

Replica consistency será distinta de persistence consistency.

## DB-ORM-CONSISTENCY-088

Batching no cambiará el consistency model.

## DB-ORM-CONSISTENCY-089

Batch outcomes se reconciliarán por operación cuando exista evidencia.

## DB-ORM-CONSISTENCY-090

Unknown batch outcome propagará incertidumbre.

## DB-ORM-CONSISTENCY-091

Lifecycle post* no implicará commit.

## DB-ORM-CONSISTENCY-092

External durable effects deberán coordinarse con commit/outbox.

## DB-ORM-CONSISTENCY-093

Recorded domain event no implicará published event.

## DB-ORM-CONSISTENCY-094

Consistency checker no hará hidden DB I/O por defecto.

## DB-ORM-CONSISTENCY-095

Database verification será operación explícita.

## DB-ORM-CONSISTENCY-096

Hot-path checks deberán ser bounded.

## DB-ORM-CONSISTENCY-097

Deep checks podrán reservarse para development/testing.

## DB-ORM-CONSISTENCY-098

UoWVersion permitirá detectar stale plans.

## DB-ORM-CONSISTENCY-099

PersistenceGeneration será scoped.

## DB-ORM-CONSISTENCY-100

Snapshot generation podrá utilizarse para diagnóstico.

## DB-ORM-CONSISTENCY-101

Internal reconciliation deberá ser lógicamente atómica.

## DB-ORM-CONSISTENCY-102

Internal reconciliation no será una DB transaction.

## DB-ORM-CONSISTENCY-103

Reconciliation failure taintará contexto cuando no pueda revertirse internamente.

## DB-ORM-CONSISTENCY-104

Known DB success no se convertirá a failure por post lifecycle failure.

## DB-ORM-CONSISTENCY-105

Flush failure podrá coexistir con persistence operation success.

## DB-ORM-CONSISTENCY-106

Transaction rollback podrá revertir operaciones previamente exitosas.

## DB-ORM-CONSISTENCY-107

Object state podrá diferir del DB state después de rollback sin ocultarlo.

## DB-ORM-CONSISTENCY-108

VoltStack no implementará magical time-travel object rollback.

## DB-ORM-CONSISTENCY-109

Rollback de generated insert requerirá policy específica.

## DB-ORM-CONSISTENCY-110

Rollback de delete requerirá reconciliation.

## DB-ORM-CONSISTENCY-111

Consistency policy será explícita.

## DB-ORM-CONSISTENCY-112

Performance mode no podrá convertir UNKNOWN en success.

## DB-ORM-CONSISTENCY-113

Ninguna policy podrá desactivar identity uniqueness.

## DB-ORM-CONSISTENCY-114

Ninguna policy podrá desactivar tenant isolation.

## DB-ORM-CONSISTENCY-115

Ninguna policy podrá ocultar unknown transaction outcome.

## DB-ORM-CONSISTENCY-116

Immutable consistency definitions podrán compartirse entre requests.

## DB-ORM-CONSISTENCY-117

Mutable consistency state será request/operation scoped.

## DB-ORM-CONSISTENCY-118

No existirá global mutable consistency state.

## DB-ORM-CONSISTENCY-119

FrankenPHP no reutilizará PersistenceContext entre requests.

## DB-ORM-CONSISTENCY-120

RoadRunner no reutilizará PersistenceContext entre jobs/requests.

## DB-ORM-CONSISTENCY-121

OpenSwoole aislará consistency state por contexto lógico.

## DB-ORM-CONSISTENCY-122

PersistenceContext no será concurrent-mutation-safe por defecto.

## DB-ORM-CONSISTENCY-123

Concurrent unsupported mutation será rechazada.

## DB-ORM-CONSISTENCY-124

Telemetry registrará consistency violations.

## DB-ORM-CONSISTENCY-125

Telemetry registrará uncertain outcomes.

## DB-ORM-CONSISTENCY-126

Telemetry registrará taint.

## DB-ORM-CONSISTENCY-127

Telemetry registrará reconciliation failures.

## DB-ORM-CONSISTENCY-128

Telemetry no expondrá entity IDs como labels.

## DB-ORM-CONSISTENCY-129

Telemetry no expondrá sensitive values por defecto.

## DB-ORM-CONSISTENCY-130

Diagnostics deberán explicar la causa del taint.

## DB-ORM-CONSISTENCY-131

Diagnostics deberán distinguir execution outcome de transaction outcome.

## DB-ORM-CONSISTENCY-132

Diagnostics deberán distinguir stale de uncertain.

## DB-ORM-CONSISTENCY-133

Consistency status nunca sustituirá evidencia detallada.

## DB-ORM-CONSISTENCY-134

Evidence tendrá precedencia sobre inferencias optimistas.

## DB-ORM-CONSISTENCY-135

Absence of evidence no será evidence of rollback.

## DB-ORM-CONSISTENCY-136

Absence of evidence no será evidence of success.

## DB-ORM-CONSISTENCY-137

Connection loss no implicará automáticamente operation failure.

## DB-ORM-CONSISTENCY-138

Timeout no implicará automáticamente operation failure.

## DB-ORM-CONSISTENCY-139

Commit connection loss no implicará automáticamente rollback.

## DB-ORM-CONSISTENCY-140

Consistency recovery no utilizará el mismo contexto cuando sus premisas fundamentales sean desconocidas.

## DB-ORM-CONSISTENCY-141

Clear no significará database rollback.

## DB-ORM-CONSISTENCY-142

Detach no significará database rollback.

## DB-ORM-CONSISTENCY-143

Refresh no significará transaction synchronization.

## DB-ORM-CONSISTENCY-144

IdentityMap hit no garantizará freshness.

## DB-ORM-CONSISTENCY-145

IdentityMap miss no implicará row absence.

## DB-ORM-CONSISTENCY-146

Snapshot equality no probará database equality.

## DB-ORM-CONSISTENCY-147

ChangeSet empty no probará database equality.

## DB-ORM-CONSISTENCY-148

Managed state no probará row existence.

## DB-ORM-CONSISTENCY-149

Generated ID no probará row existence.

## DB-ORM-CONSISTENCY-150

PersistencePlan no probará execution.

## DB-ORM-CONSISTENCY-151

Execution success no probará transaction durability.

## DB-ORM-CONSISTENCY-152

Transaction commit certainty será necesaria para efectos after-commit.

## DB-ORM-CONSISTENCY-153

Consistency assertions no modificarán entidades.

## DB-ORM-CONSISTENCY-154

Consistency checker será side-effect free por defecto.

## DB-ORM-CONSISTENCY-155

Consistency recovery sí podrá requerir operaciones explícitas.

## DB-ORM-CONSISTENCY-156

Recovery operations serán auditables/diagnosticables.

## DB-ORM-CONSISTENCY-157

Consistency state pertenecerá al PersistenceContext, no a la Connection global.

## DB-ORM-CONSISTENCY-158

Connection lifetime será distinta de PersistenceContext lifetime.

## DB-ORM-CONSISTENCY-159

Transaction lifetime será distinta de IdentityMap lifetime.

## DB-ORM-CONSISTENCY-160

VoltStack nunca fabricará una visión aparentemente consistente cuando la evidencia real indique incertidumbre sobre persistencia, ejecución o transacción.

---

# 248. Fórmulas fundamentales

## 248.1 ORM Consistency

```text
ORMConsistency
=
IdentityConsistency
∧
EntityStateConsistency
∧
SnapshotConsistency
∧
ChangeSetConsistency
∧
UnitOfWorkConsistency
∧
RelationshipConsistency
∧
PlanConsistency
```

---

# 249. Persistence Knowledge

```text
PersistenceKnowledge
=
ExecutionEvidence
+
TransactionEvidence
+
GeneratedValueEvidence
+
ReconciliationEvidence
```

---

# 250. Durable certainty

```text
DurablePersistenceCertain
=
PersistenceOperationSucceeded
∧
(
    NoExternalTransactionPending
    ∨
    TransactionCommitCertain
)
```

sujeto a las garantías reales de la plataforma.

---

# 251. Unknown persistence

```text
PersistenceOutcomeUnknown
=
ExecutionOutcomeUnknown
∨
TransactionOutcomeUnknown
```

cuando la transacción es relevante para la operación.

---

# 252. Safe snapshot advancement

```text
SafeSnapshotAdvance
=
OperationOutcomeSufficientlyCertain
∧
GeneratedValuesReconciled
∧
EntityStateTransitionValid
```

---

# 253. Safe identity reconciliation

```text
SafeIdentityReconciliation
=
IdentifierValid
∧
IdentityNamespaceValid
∧
NoIdentityConflict
∧
PersistenceOutcomeSufficientlyCertain
```

---

# 254. Safe continuation

```text
SafePersistenceContinuation
=
ContextNotTainted
∧
CoreInvariantsSatisfied
∧
NoUnresolvedCriticalOutcome
```

---

# 255. Taint condition

```text
TaintRequired
=
UnknownCriticalOutcome
∨
InternalInvariantViolation
∨
UnrecoverableReconciliationFailure
∨
UnsafePartialPersistence
∨
CrossContextContamination
```

---

# 256. Rollback consistency

```text
RollbackReconciliation
=
DatabaseRollbackEvidence
+
PreTransactionPersistenceBaseline
+
CurrentObjectState
+
GeneratedValueState
+
ConfiguredRollbackPolicy
```

---

# 257. Freshness

```text
Freshness
≠
Consistency
```

y:

```text
CanonicalIdentity
≠
FreshDatabaseObservation
```

---

# 258. Global consistency vector

```text
ConsistencyVector
=
(
    Identity,
    EntityState,
    Snapshot,
    ChangeSet,
    UnitOfWork,
    Relationships,
    PersistencePlan,
    Execution,
    GeneratedValues,
    Transaction,
    DatabaseKnowledge
)
```

---

# 259. Master Formula

```text
Database Persistence Consistency System
=
Consistency Domains
+
Consistency Status Model
+
Identity Consistency
+
Entity State Consistency
+
Snapshot Consistency
+
ChangeSet Consistency
+
UnitOfWork Consistency
+
Relationship Consistency
+
PersistencePlan Consistency
+
Execution Outcome Consistency
+
Evidence Model
+
Outcome Certainty
+
Database Knowledge Model
+
Generated Identity Consistency
+
Partial Persistence Modeling
+
Transaction-Aware Consistency
+
Pre-Commit State
+
Post-Commit State
+
Unknown Commit Handling
+
Rollback Reconciliation
+
Persistence Checkpoints
+
Consistency Vectors
+
Tainted Persistence Context
+
Recovery Strategies
+
External Mutation Awareness
+
Raw SQL Invalidation
+
Bulk Operation Invalidation
+
Refresh Policies
+
Batch Reconciliation
+
Lifecycle Consistency
+
After-Commit Boundaries
+
Consistency Assertions
+
UnitOfWork Versioning
+
Persistence Generations
+
Atomic In-Memory Reconciliation
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 260. Master Rule

> **En VoltStack, la consistencia de persistencia no significará que el ORM pretenda conocer permanentemente la realidad absoluta de la base de datos. Significará que cada afirmación sobre identidad, estado de entidad, snapshots, cambios, operaciones, resultados y transacciones estará respaldada por evidencia suficiente y por invariantes verificables. Cuando la base de datos, la conexión o la transacción dejen un resultado incierto, VoltStack preservará esa incertidumbre explícitamente y, cuando sea necesario, marcará el `PersistenceContext` como `TAINTED` en lugar de inventar un estado limpio, exitoso o revertido.**

---

# 261. Cierre del Bloque 11

Con este documento queda completo:

```text
BLOCK 11
IDENTITY MAP, UNIT OF WORK & PERSISTENCE

123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md
131_DATABASE_DELETE_PERSISTENCE_SYSTEM.md
132_DATABASE_FLUSH_SYSTEM.md
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
134_DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM.md
```

La arquitectura resultante queda:

```text
                        Entity
                           │
                           ▼
                     EntityManager
                           │
                           ▼
                       UnitOfWork
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         IdentityMap   Snapshots    ChangeTracking
              │            │            │
              └────────────┼────────────┘
                           ▼
                         Flush
                           │
                           ▼
                  Persistence Planner
                           │
                           ▼
                 Logical PersistencePlan
                           │
                           ▼
                  Batch Optimization
                           │
                           ▼
                Physical PersistencePlan
                           │
                           ▼
                  Persistence Engine
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
                  Execution Evidence
                           │
                           ▼
                     Reconciliation
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         IdentityMap   Snapshots    EntityState
              │            │            │
              └────────────┼────────────┘
                           ▼
               Persistence Consistency
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
           CONSISTENT     STALE      UNCERTAIN
                                       │
                                       ▼
                                    TAINTED
```

Con esto, VoltStack ya dispone conceptualmente de una infraestructura ORM donde:

```text
Entity identity
+
Unit of Work
+
Change tracking
+
Snapshots
+
Persistence planning
+
Insert/Update/Delete
+
Flush
+
Batch persistence
+
Consistency
```

forman un único modelo coherente sin convertir al ORM en generador directo de SQL ni confundir ejecución con transacción o durabilidad.

---

# 262. Siguiente bloque

El siguiente bloque es:

```text
BLOCK 12
HYDRATION
```

compuesto por:

```text
135_DATABASE_HYDRATION_ARCHITECTURE.md
136_DATABASE_ENTITY_HYDRATOR_SYSTEM.md
137_DATABASE_RESULT_HYDRATION_SYSTEM.md
138_DATABASE_SCALAR_HYDRATION_SYSTEM.md
139_DATABASE_PARTIAL_ENTITY_HYDRATION_SYSTEM.md
140_DATABASE_HYDRATION_PLAN_SYSTEM.md
141_DATABASE_HYDRATION_CACHE_SYSTEM.md
```

---

# 263. Siguiente documento

```text
135_DATABASE_HYDRATION_ARCHITECTURE.md
```

Este documento deberá establecer la arquitectura general del camino inverso a Persistence:

```text
Database Result
      ↓
Result Metadata
      ↓
Hydration Plan
      ↓
Identity Resolution
      ↓
IdentityMap
      ↓
Entity Construction / Reuse
      ↓
Field Conversion
      ↓
Relationship Assembly
      ↓
Snapshot Creation
      ↓
EntityState MANAGED
      ↓
Lifecycle postLoad
      ↓
Application
```

manteniendo la separación:

```text
Hydration
≠
Entity Construction

Hydration
≠
Serialization

Hydration
≠
Mapping

Hydration
≠
Persistence

Hydration
≠
Query Execution

Hydration
≠
IdentityMap
```

y resolviendo especialmente:

```text
duplicate rows from JOINs
identity reuse
circular object graphs
two-phase hydration
generated proxies
partial entities
scalar/projection hydration
type conversion
readonly properties
constructor bypass policies
snapshot creation
dirty-state prevention during hydration
streaming
large datasets
hydration cache
persistent-runtime isolation
```