# 172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md

# VoltStack Quantum Database
## Database Optimistic Locking System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 172 — Database Optimistic Locking System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md`  
**Siguiente documento:** `173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md`

---

# 1. Propósito

`Database Optimistic Locking System` define cómo VoltStack detectará que una entidad o recurso persistente fue modificado por otra operación concurrente desde el momento en que fue leído hasta el momento en que se intenta actualizar o eliminar.

El principio base será:

```text id="2nngls"
Read Version V1
    ↓
Application modifies entity
    ↓
UPDATE ... WHERE id = ? AND version = V1
    ↓
Affected rows = 1
        → success

Affected rows = 0
        → concurrent conflict or missing row
```

La arquitectura deberá distinguir:

```text id="gweofu"
Optimistic Lock Conflict
≠
Deadlock
≠
Lock Timeout
≠
Serialization Failure
≠
Constraint Violation
```

Regla central:

> **VoltStack implementará optimistic locking como una condición explícita de compare-and-swap sobre un token de versión previamente observado; una operación solo podrá considerarse persistida si la versión esperada sigue siendo válida en el momento de la escritura y el resultado de la operación confirma que el recurso correcto fue modificado.**

---

# 2. Objetivos

El sistema deberá soportar:

1. version fields;
2. version metadata;
3. integer versions;
4. timestamp versions;
5. opaque version tokens;
6. expected version;
7. next version;
8. compare-and-swap;
9. optimistic update;
10. optimistic delete;
11. stale entity detection;
12. affected-row verification;
13. UnitOfWork integration;
14. ChangeSet integration;
15. Entity Snapshot integration;
16. Persistence Planner integration;
17. Query Model integration;
18. SQL Compiler integration;
19. hydration integration;
20. detached entity handling;
21. relationship conflict awareness;
22. bulk operation restrictions;
23. transaction interaction;
24. retry interaction;
25. conflict diagnostics;
26. conflict resolution boundaries;
27. HTTP/API integration boundaries;
28. telemetry;
29. testing;
30. persistent-runtime safety.

---

# 3. No objetivos

Optimistic Locking System no deberá:

- adquirir row locks;
- implementar `SELECT ... FOR UPDATE`;
- sustituir transaction isolation;
- sustituir constraints;
- resolver deadlocks;
- reintentar automáticamente business operations;
- realizar merge automático de estados incompatibles;
- implementar CRDTs;
- implementar distributed consensus;
- asumir que timestamp siempre es seguro;
- basarse únicamente en `updated_at` por defecto;
- modificar entidades sin pasar por Persistence Engine.

---

# 4. Distinciones fundamentales

```text id="0r1rdn"
Optimistic Locking
≠
Pessimistic Locking
≠
Transaction Isolation
≠
Dirty Checking
≠
Entity Snapshot
≠
Database Constraint
≠
HTTP ETag
```

---

# 5. Problema de lost update

Ejemplo:

```text id="i24arp"
Initial:
    balance = 100
    version = 7
```

Dos procesos leen:

```text id="znmosb"
Process A:
    balance = 100
    version = 7

Process B:
    balance = 100
    version = 7
```

A actualiza:

```text id="ewh2la"
balance = 120
version = 8
```

B actualiza posteriormente:

```text id="ayt8n1"
balance = 90
```

Sin control de versión, B puede sobrescribir silenciosamente el cambio de A.

---

# 6. Compare-and-swap

VoltStack evitará esto generando una condición equivalente a:

```sql id="w6ghn2"
UPDATE accounts
SET
    balance = ?,
    version = ?
WHERE
    id = ?
    AND version = ?;
```

Con:

```text id="8t8boc"
next version = 8
expected version = 7
```

---

# 7. Resultado esperado

Si:

```text id="xbarsl"
affected rows = 1
```

la actualización ganó la condición competitiva.

Si:

```text id="8gi8sz"
affected rows = 0
```

VoltStack deberá investigar/clasificar:

```text id="w7cgra"
STALE_VERSION
ROW_NOT_FOUND
UNKNOWN
```

según la evidencia disponible.

---

# 8. OptimisticLockMetadata

Cada entidad optimistically locked tendrá metadata explícita.

```php id="mkfeyn"
final readonly class OptimisticLockMetadata
{
    public function __construct(
        public PropertyMetadata $property,
        public OptimisticVersionStrategy $strategy,
        public VersionGenerator $generator,
        public OptimisticLockPolicy $policy,
    ) {}
}
```

---

# 9. Mapping example

Mediante attribute:

```php id="qqvsou"
#[Entity]
class Order
{
    #[Id]
    private int $id;

    #[Version]
    private int $version;
}
```

---

# 10. Explicit metadata only

VoltStack no deberá inferir automáticamente:

```text id="fiudzc"
property named "version"
```

como optimistic locking sin mapping explícito.

Esto evita cambios semánticos sorpresivos.

---

# 11. Version token

Concepto canónico:

```text id="oqkuub"
OptimisticVersionToken
```

representa la versión observada.

```php id="kr4gnl"
interface OptimisticVersionToken
{
    public function equals(
        OptimisticVersionToken $other,
    ): bool;
}
```

---

# 12. Version token ≠ field value only

Aunque muchas veces:

```text id="ph9ca0"
token = integer
```

la arquitectura no deberá asumirlo universalmente.

Podrá existir:

```text id="s43n54"
integer token
timestamp token
UUID-like token
database-generated token
opaque binary token
```

---

# 13. OptimisticVersionStrategy

```php id="3x6ih9"
enum OptimisticVersionStrategy
{
    case INTEGER;
    case TIMESTAMP;
    case OPAQUE;
    case DATABASE_GENERATED;
    case CUSTOM;
}
```

---

# 14. INTEGER

Modelo recomendado y portable:

```text id="ec8cwl"
1
2
3
4
...
```

---

# 15. Integer progression

Normalmente:

```text id="4n94r5"
NextVersion
=
CurrentVersion + 1
```

---

# 16. Overflow

La implementación deberá verificar:

```text id="3tjq9s"
version overflow
```

y nunca realizar wrap silencioso.

---

# 17. Version type size

El Schema/Type System deberá permitir un entero suficientemente amplio para la vida esperada del recurso.

---

# 18. Timestamp strategy

Podrá utilizarse un valor temporal como versión.

Pero:

```text id="we87lf"
TimestampVersion
```

tiene riesgos:

- precisión insuficiente;
- múltiples writes dentro del mismo tick;
- timezone confusion;
- clock source ambiguity;
- DB/app clock mismatch.

---

# 19. Timestamp safety rule

VoltStack no deberá declarar timestamp-based locking completamente seguro a menos que:

```text id="fheevn"
resolution
+
source
+
platform semantics
```

sean suficientes.

---

# 20. updated_at ≠ safe version automatically

Regla:

```text id="7bk4n7"
updated_at
≠
optimistic version
```

por default.

---

# 21. Opaque token

Ejemplo:

```text id="w0pypq"
version token:
    b6f0c8...
```

La aplicación no necesita conocer semántica incremental.

Solo:

```text id="in9j83"
ExpectedToken == CurrentStoredToken?
```

---

# 22. Database-generated version

Algunos motores/platform features pueden producir automáticamente un token de versión.

VoltStack podrá soportarlo mediante capabilities.

---

# 23. Capability-driven generation

Nunca:

```php id="4g1ez1"
if ($database === 'vendor-x') {
    // ...
}
```

en ORM core.

Usar:

```text id="w7xczj"
Platform Optimistic Version Capabilities
```

---

# 24. VersionGenerator

```php id="3okmsn"
interface VersionGenerator
{
    public function next(
        OptimisticVersionToken $current,
        VersionGenerationContext $context,
    ): OptimisticVersionToken;
}
```

---

# 25. Generator determinism

Para integer strategy:

```text id="4le7uh"
next(current)
```

será determinista.

Para otros tipos deberá documentarse si:

```text id="d2u8ga"
database-generated
application-generated
```

---

# 26. Version state

El ORM deberá distinguir:

```text id="hd8ybe"
ObservedVersion
CurrentEntityVersion
ExpectedDatabaseVersion
NextPersistentVersion
```

---

# 27. ObservedVersion

Es la versión obtenida durante:

```text id="ysgj80"
hydration
reload
refresh
```

y pertenece al baseline/snapshot persistente.

---

# 28. CurrentEntityVersion

Es el valor que actualmente contiene el objeto.

No deberá asumirse automáticamente como expected version si el usuario lo modificó manualmente.

---

# 29. Version field immutability

Por default, la propiedad version será:

```text id="ehuy3c"
ORM-managed
```

y la aplicación no deberá modificarla directamente.

---

# 30. User mutation

Si se detecta:

```php id="1xlwzh"
$order->version = 999;
```

sobre un campo gestionado, deberá:

```text id="1c6uqt"
fail
```

o ignorarse según política estricta, nunca incorporarse silenciosamente como versión legítima.

---

# 31. Version snapshot

`EntitySnapshot` deberá conservar:

```text id="zchx3h"
version token observed
```

junto al resto del baseline.

---

# 32. Snapshot rule

```text id="qkv9eu"
ExpectedVersion
=
Snapshot.Version
```

para entidades managed normales.

---

# 33. Hydration integration

Durante hydration:

```text id="q07ijb"
row
    ↓
Value Conversion
    ↓
Version Token Conversion
    ↓
Entity
    ↓
EntitySnapshot
```

---

# 34. Full identifier requirement

Una entidad optimistically locked deberá tener:

```text id="rzwoqe"
known identity
+
known version
```

antes de optimistic update/delete.

---

# 35. Partial hydration hazard

Si la entidad fue cargada parcialmente sin version field:

```text id="5vo8jy"
ExpectedVersion = UNKNOWN
```

No deberá permitirse un optimistic update normal.

---

# 36. Required version field

Cuando una entidad optimistically locked se hidrata como entity managed destinada a persistencia:

```text id="kmwl9r"
version field
```

deberá formar parte del hydration plan obligatorio.

---

# 37. Projection exception

Una DTO/projection no necesita version field si no será utilizada como managed entity.

---

# 38. UnitOfWork integration

Durante change tracking:

```text id="2qizrq"
Managed Entity
    ↓
Snapshot
    ↓
ChangeSet
    ↓
Optimistic Version Metadata
```

---

# 39. ChangeSet

La versión gestionada no deberá aparecer como un cambio ordinario del usuario.

Puede existir:

```text id="58gora"
SystemGeneratedChange
```

para:

```text id="b2nwsk"
version old → version new
```

---

# 40. ChangeSet structure

Conceptualmente:

```php id="j6geij"
final readonly class OptimisticVersionChange
{
    public function __construct(
        public OptimisticVersionToken $expected,
        public OptimisticVersionToken $next,
    ) {}
}
```

---

# 41. Persistence Planner

El planner deberá convertir:

```text id="gyu51i"
EntityUpdate
+
OptimisticVersionChange
```

en una operación semántica:

```text id="kdfyak"
OptimisticUpdateOperation
```

---

# 42. OptimisticUpdateOperation

```php id="253yxn"
final readonly class OptimisticUpdateOperation
{
    public function __construct(
        public EntityKey $entity,
        public ChangeSet $changes,
        public OptimisticVersionToken $expectedVersion,
        public OptimisticVersionToken $nextVersion,
    ) {}
}
```

---

# 43. Semantic operation ≠ SQL

`OptimisticUpdateOperation` no contendrá necesariamente:

```text id="bp5640"
UPDATE string
```

El Query Engine construirá un Query Model.

---

# 44. Query Model

Conceptualmente:

```text id="cof8fa"
UPDATE entity_table
SET
    changed fields
    version = next
WHERE
    identity predicate
    AND version = expected
```

---

# 45. SQL Compiler

Solo el compiler producirá SQL físico.

Regla previa continúa:

```text id="3xu9dp"
ORM
≠
SQL Compiler
```

---

# 46. Affected-row verification

Después de ejecutar la operación deberá verificarse:

```text id="g65u90"
AffectedRows
```

---

# 47. Exact expected count

Para una actualización por identidad única:

```text id="2ccjj8"
ExpectedAffectedRows = 1
```

---

# 48. AffectedRows = 1

Significa que:

```text id="k3bvn2"
identity matched
+
expected version matched
+
write accepted
```

conforme a las semantics conocidas de plataforma.

---

# 49. AffectedRows = 0

Podrá significar:

```text id="56sa8n"
entity deleted
version changed
row missing
platform affected-row semantic issue
```

---

# 50. Need conflict classification

No deberá asumirse inmediatamente:

```text id="7ia52s"
0 rows
→ stale version
```

si la plataforma no garantiza esa interpretación.

---

# 51. Update count semantics

El Platform deberá exponer:

```php id="plgo7h"
interface AffectedRowSemantics
{
    public function supportsReliableMatchedRowCount(): bool;

    public function supportsReliableChangedRowCount(): bool;
}
```

---

# 52. Important no-op update issue

Algunas APIs/motores pueden reportar affected rows según:

```text id="q7otfw"
rows changed
```

y no:

```text id="co01ag"
rows matched
```

Esto es crítico si la actualización no cambia realmente valores.

---

# 53. Version increment solves many no-op cases

Si el sistema siempre actualiza:

```text id="a6drsj"
version
```

entonces existe un cambio físico incluso cuando los demás campos mantienen el mismo valor.

Esto ayuda a obtener una señal de éxito fiable.

---

# 54. No-change flush

Sin cambios de usuario:

```text id="ll4cda"
flush()
```

no deberá incrementar versión por default.

---

# 55. Meaningful update only

La versión deberá avanzar cuando exista una operación persistente que conceptualmente modifica el recurso.

---

# 56. Touch semantics

Si la aplicación desea incrementar versión sin otro cambio podrá existir una operación explícita:

```php id="dys2wm"
$entityManager->touch($entity);
```

o equivalente futuro.

---

# 57. Conflict exception

```php id="b54bsx"
final class OptimisticLockException
    extends DatabaseConcurrencyException
{
    public function entityKey(): EntityKey;

    public function expectedVersion(): OptimisticVersionToken;

    public function actualVersion(): ?OptimisticVersionToken;

    public function conflictKind(): OptimisticConflictKind;
}
```

---

# 58. Conflict kinds

```php id="10cwzc"
enum OptimisticConflictKind
{
    case STALE_VERSION;
    case ENTITY_DELETED;
    case ENTITY_MISSING;
    case VERSION_UNKNOWN;
    case RESULT_AMBIGUOUS;
}
```

---

# 59. Actual version

Obtener:

```text id="clxd91"
actual version
```

puede requerir otra query.

VoltStack no deberá hacerla necesariamente en hot path.

---

# 60. Conflict diagnostic enrichment

Policy:

```text id="0h56h0"
NONE
ON_DEMAND
DEBUG
ALWAYS
```

podrá controlar si se realiza un follow-up SELECT.

---

# 61. Follow-up query race

Incluso si se consulta después:

```text id="afmp4i"
UPDATE affects 0
    ↓
SELECT current version
```

otro proceso podría modificar nuevamente entre ambos.

Por tanto:

```text id="6ltnkq"
actualVersion
```

será información diagnóstica observada posteriormente, no una prueba atómica del valor que causó exactamente el conflicto.

---

# 62. Conflict truth

La verdad atómica es:

```text id="7pthpy"
expected version predicate did not successfully update target
```

El motivo exacto puede requerir más evidencia.

---

# 63. Optimistic delete

Delete deberá incluir la versión:

```sql id="70fcb4"
DELETE FROM orders
WHERE id = ?
  AND version = ?;
```

---

# 64. Delete success

```text id="7t1vde"
affected rows = 1
```

→ delete accepted.

---

# 65. Delete conflict

```text id="v9uzba"
affected rows = 0
```

→ stale/deleted/missing/ambiguous según classification.

---

# 66. Delete without version

Para una entidad optimistically locked, `remove()` deberá usar el token esperado.

No deberá degradarse silenciosamente a:

```text id="n6cdhm"
DELETE WHERE id = ?
```

---

# 67. Soft delete

Si soft delete se implementa como UPDATE:

```text id="gbix94"
deleted_at = ...
version = version + 1
```

deberá respetar optimistic locking.

---

# 68. Relationship modifications

Cambios relacionales pueden requerir consideración especial.

Ejemplo:

```text id="h4uyn4"
Order
    version = 7

addLineItem()
```

¿Debe incrementar versión de `Order`?

La respuesta deberá ser mapping/policy-driven.

---

# 69. Relationship version policy

```php id="gr91dc"
enum RelationshipVersionPolicy
{
    case NONE;
    case OWNER_CHANGES_ONLY;
    case MEMBERSHIP_CHANGES;
    case CUSTOM;
}
```

---

# 70. Many-to-many membership

Un cambio en:

```text id="jt5o2j"
join table
```

no toca necesariamente la fila del owner.

Si se desea proteger aggregate-level consistency, podrá requerirse:

```text id="9qfn0n"
owner version bump
```

explícito.

---

# 71. Aggregate concurrency

VoltStack no deberá asumir que:

```text id="f06v3s"
Entity Version
```

protege automáticamente todo:

```text id="m687ij"
aggregate graph
```

---

# 72. Aggregate version policy

Una futura capa Domain/Persistence podrá declarar:

```text id="n769j4"
AggregateVersionRoot
```

pero permanecerá separada del mecanismo de version CAS base.

---

# 73. One-to-many changes

Añadir un child puede modificar solamente:

```text id="sqgfcd"
child foreign key
```

y no la versión del parent.

La política deberá ser explícita.

---

# 74. Detached entities

Caso:

```text id="vd7g26"
Request A:
    loads Order version 4
    serializes DTO

later:
    detached object returned for update
```

El sistema necesita conocer:

```text id="48glor"
expected version = 4
```

---

# 75. Detached version token

El expected version podrá provenir de:

- DTO;
- command;
- request;
- entity snapshot;
- explicit lock token.

Pero deberá validarse.

---

# 76. Detached update API

Conceptualmente:

```php id="73ldqo"
$repository->update(
    id: $id,
    expectedVersion: $version,
    changes: $changes,
);
```

Esto permite optimistic locking sin rehidratar necesariamente una entidad managed.

---

# 77. Direct optimistic update

El Query/Repository layer podrá producir una operación CAS explícita.

Ejemplo:

```text id="9b0ibf"
UpdateWhereIdentityAndVersion
```

sin convertirla en Active Record magic.

---

# 78. Merge semantics

VoltStack no deberá implementar automáticamente:

```text id="o13xar"
detached entity
+
current entity
→ magically merge all fields
```

porque puede sobrescribir cambios concurrentes.

---

# 79. Safe merge

Una estrategia de merge deberá conocer:

```text id="5u3hpq"
original baseline
client changes
current database state
```

y resolver conflictos campo por campo.

Esto pertenece a un sistema superior.

---

# 80. Optimistic conflict ≠ automatic merge

Regla:

```text id="m2wbp8"
OptimisticLockException
≠
MergeInstruction
```

---

# 81. Retry interaction

Un optimistic conflict puede ser técnicamente reintentable.

Pero:

```text id="abosz4"
retry same callback blindly
```

puede no ser semánticamente correcto.

---

# 82. Example business conflict

```text id="ma2diq"
A:
    changes shipping address

B:
    cancels order
```

A recibe optimistic conflict.

Reejecutar automáticamente A sobre el nuevo estado podría:

```text id="fku7tp"
modify cancelled order
```

incorrectamente.

---

# 83. Default retry recommendation

Para:

```text id="6p9t2v"
OptimisticLockConflict
```

default:

```text id="ta8xdq"
RECONCILE / APPLICATION DECISION
```

no:

```text id="6t62h5"
AUTOMATIC TRANSACTION RETRY
```

---

# 84. Retry policy distinction

Deadlock:

```text id="vp041z"
same business intention
+
transaction aborted due scheduling
```

frecuentemente puede reintentarse.

Optimistic conflict:

```text id="uobc35"
business state changed
```

puede requerir reevaluar intención.

---

# 85. Explicit optimistic retry

VoltStack podrá permitir una política:

```php id="s84tm2"
OptimisticRetryPolicy::reloadAndReexecute(...)
```

pero será opt-in y deberá ejecutar lógica de negocio nuevamente sobre estado fresco.

---

# 86. Reload requirement

Para retry correcto:

```text id="zjwwcz"
fresh EntityManager
+
reload entity
+
re-run business logic
```

será normalmente preferido.

---

# 87. Blind UPDATE retry prohibited

No:

```text id="jzo2nm"
version conflict
→ use actual version
→ issue same UPDATE
```

automáticamente.

Eso anularía la protección.

---

# 88. Transaction isolation interaction

Optimistic locking funciona:

```text id="e2zvr3"
in addition to
```

transaction isolation.

---

# 89. READ COMMITTED

Con `READ_COMMITTED`, optimistic locking puede proteger contra lost updates de entidades versionadas.

---

# 90. REPEATABLE READ

También puede proporcionar detección explícita de version conflicts, aunque el motor ya ofrezca otras garantías.

---

# 91. SERIALIZABLE

Incluso bajo `SERIALIZABLE`, un version token puede servir para:

- application-level stale detection;
- detached update protection;
- API concurrency control.

No obstante, puede ser redundante para ciertos conflictos físicos.

---

# 92. Isolation does not make version useless

Regla:

```text id="nactne"
Serializable Isolation
≠
Automatic Detached Entity Freshness
```

---

# 93. Transaction requirement

Optimistic locking no requiere conceptualmente mantener un lock desde la lectura.

Pero la write operation ocurre dentro de la semántica normal de transacción/autocommit del DB.

---

# 94. Multi-operation business changes

Si se modifican varias entidades versionadas:

```text id="a7w6or"
Order version
Customer version
Inventory version
```

deberán ejecutarse dentro de una transaction boundary cuando requieran atomicidad conjunta.

---

# 95. Partial success

Sin transacción:

```text id="1d8brc"
Order succeeds
Customer conflict
```

puede dejar cambios parciales.

Por ello:

```text id="6l38ow"
OptimisticLocking
≠
TransactionAtomicity
```

---

# 96. Transaction rollback on conflict

Si un optimistic conflict ocurre dentro de una transacción con otras escrituras:

```text id="a73e5o"
transaction
→ rollback-only / rollback
```

será normalmente necesario.

---

# 97. Query Executor result

Execution layer deberá devolver:

```text id="nc94mi"
affected rows
returning values
generated values
```

sin interpretar entidades.

---

# 98. Persistence layer interprets count

`Persistence Engine` sabe:

```text id="cc2z5i"
operation expected exactly one versioned entity
```

y puede transformar:

```text id="jh2ykj"
0 affected rows
```

en un conflicto optimista.

---

# 99. Executor ≠ optimistic lock authority

Regla:

```text id="e72hpf"
QueryExecutor
does not know
Entity optimistic semantics
```

---

# 100. RETURNING support

Cuando una plataforma soporte:

```text id="zqe5kn"
RETURNING
```

podrá recuperarse el nuevo token/version en la misma operación.

---

# 101. No RETURNING

Sin esa capability:

- next version puede ser conocida por aplicación;
- podrá requerirse select posterior para database-generated token;
- o la estrategia podrá no estar soportada eficientemente.

---

# 102. Database-generated version problem

Si el DB genera el token y no existe forma fiable de recuperarlo:

```text id="g21zzb"
Optimistic version strategy
```

podrá ser:

```text id="8lzgom"
UNSUPPORTED
```

para managed continuation.

---

# 103. Version write ordering

La versión nueva deberá formar parte de la misma operación atómica que los cambios protegidos.

No:

```text id="pggbmb"
UPDATE entity fields
then
UPDATE version
```

como dos statements independientes.

---

# 104. Atomic CAS invariant

```text id="xdv0gr"
Check(ExpectedVersion)
+
WriteChanges
+
AdvanceVersion
```

deberán ocurrir como una operación atómica lógica.

---

# 105. Integer SQL expression

Podrá compilarse:

```sql id="tj8wpf"
version = version + 1
```

o:

```sql id="ctsj2z"
version = :next_version
```

según strategy/platform.

---

# 106. Expected and next values

Si se usa:

```text id="e4br9r"
version = version + 1
```

el Query Model seguirá conociendo:

```text id="fa0ntc"
expected version
```

y la Persistence Layer deberá conocer el nuevo valor cuando sea necesario.

---

# 107. Overflow with DB-side increment

Platform/type capability deberá garantizar detección o impedir la operación antes cuando sea posible.

---

# 108. Version initialization

Para nuevas entidades:

```text id="zrpwfu"
InitialVersion
```

será definida por strategy.

Ejemplo integer:

```text id="0j7g1h"
1
```

---

# 109. InitialVersion policy

```php id="841965"
interface VersionInitializationPolicy
{
    public function initial(
        VersionInitializationContext $context,
    ): OptimisticVersionToken;
}
```

---

# 110. Insert and version

INSERT de una entidad versionada deberá establecer o recuperar:

```text id="1vud1a"
initial version
```

como parte de persistence reconciliation.

---

# 111. Insert is not optimistic conflict

Una entidad NEW no posee una versión observada previa para CAS normal.

Su insert puede fallar por:

- unique constraint;
- FK;
- other errors;

pero eso no es optimistic lock conflict.

---

# 112. Version after successful insert

Después de insert confirmado:

```text id="8d3ko8"
Entity.version
=
PersistedInitialVersion
```

y snapshot también.

---

# 113. Version after successful update

Después de statement success, pero antes de commit:

```text id="s8tyhu"
TransactionalEntityVersion
=
NextVersion
```

puede actualizarse internamente.

Pero debe distinguirse:

```text id="f62ao1"
statement succeeded
```

de:

```text id="nwjjvl"
transaction committed
```

---

# 114. Transaction rollback implication

Si posteriormente:

```text id="mdut4s"
ROLLBACK
```

la DB conserva la versión anterior, mientras el objeto puede tener la nueva.

Regla nuevamente:

```text id="7y7jam"
Rollback
≠
Object Graph Rewind
```

---

# 115. Reconciliation after rollback

Persistence Consistency System deberá manejar:

```text id="0eapql"
version property
snapshot
entity state
```

sin que Optimistic Locking System finja una restauración automática.

---

# 116. Commit UNKNOWN

Si CAS update fue ejecutado y:

```text id="hjsh2b"
COMMIT
→ UNKNOWN
```

no puede saberse si la nueva versión quedó persistida.

Resultado:

```text id="l3p457"
PersistenceContext
→ TAINTED
```

según reglas anteriores.

---

# 117. Bulk DML

Ejemplo:

```php id="xv4r54"
Order::where('status', 'pending')
    ->update(['status' => 'expired']);
```

No existe necesariamente:

```text id="sc92qq"
one expected version per entity
```

---

# 118. Bulk operations ≠ entity optimistic locking

Regla:

```text id="upb6up"
Bulk Update
≠
Managed Entity Optimistic Update
```

---

# 119. Bulk restrictions

Una bulk update sobre entity type versionado deberá:

- ser explícitamente bulk;
- documentar que bypassa entity-level version checks;
- o incorporar una estrategia especial si el usuario suministra predicates/version rules.

Nunca fingir que protege cada entidad individual.

---

# 120. Bulk version increment

Podrá permitirse:

```text id="t6y7zs"
SET version = version + 1
```

pero esto:

```text id="e08xel"
≠
per-entity expected version check
```

---

# 121. EntityQuery update shortcut

APIs tipo:

```php id="ni0rz9"
User::where(...)->update(...)
```

deberán diferenciarse de:

```php id="5tyotq"
$user->save()
```

respecto a optimistic locking.

---

# 122. Developer warning

En modo strict/testing podrá advertirse:

```text id="31lniq"
Bulk DML bypasses optimistic locking for versioned entity User.
```

---

# 123. Upsert

`UPSERT` presenta semánticas distintas.

No deberá utilizarse automáticamente como optimistic update de una entidad managed.

---

# 124. Upsert hazard

```text id="y0gh9n"
INSERT ... ON CONFLICT UPDATE
```

puede sobrescribir estado sin expected version predicate.

---

# 125. Custom optimistic upsert

Solo mediante operación semántica explícita que incorpore:

```text id="c32ye7"
expected version
```

y cuyas semantics sean soportadas por Platform.

---

# 126. Second-level cache

Entity Cache deberá almacenar:

```text id="mznm8n"
version token
```

si se utiliza para invalidation/consistency.

---

# 127. Cache entry identity

Ejemplo:

```text id="bp0mvq"
EntityCacheEntry
├── EntityKey
├── VersionToken
└── Data
```

---

# 128. Cache stale detection

Una entidad cargada desde cache con token antiguo deberá seguir fallando correctamente al escribir gracias al predicate CAS.

---

# 129. Cache is not lock authority

```text id="v1nmom"
EntityCache
≠
Optimistic Lock Source of Truth
```

La base de datos sigue siendo la autoridad final.

---

# 130. API/HTTP integration

Optimistic version tokens pueden mapearse conceptualmente a:

```text id="qr9cix"
ETag
If-Match
```

en APIs HTTP.

---

# 131. Separation boundary

Sin embargo:

```text id="wt2fvk"
Database Version Token
≠
HTTP ETag
```

necesariamente.

---

# 132. HTTP ETag adapter

Una capa HTTP podrá construir:

```text id="7x4lgf"
ETag
```

desde un version token, pero esto pertenece a:

```text id="xvm8ld"
HTTP/API integration
```

no al Database Optimistic Locking core.

---

# 133. If-Match

Ejemplo:

```text id="wfm16k"
Client:
    If-Match: "order-v7"

Application:
    expected version 7

Database:
    CAS update
```

---

# 134. Security of tokens

Un version token no deberá considerarse:

```text id="pl6dmv"
authorization credential
```

Conocer la versión no autoriza modificar el recurso.

---

# 135. Authorization still applies

Debe cumplirse:

```text id="zefz7y"
Authorized
∧
ExpectedVersionMatches
```

para una actualización protegida.

---

# 136. Validation order

Dependiendo de la arquitectura:

```text id="nbokjv"
authentication
authorization
business validation
optimistic persistence check
```

seguirán siendo responsabilidades independientes.

---

# 137. Conflict disclosure

Una API no deberá exponer automáticamente:

```text id="m84fyb"
actual database version
```

si eso viola una política de seguridad/information disclosure.

---

# 138. Diagnostics vs public errors

Internamente:

```text id="nhx3df"
expected=7
actual=9
```

puede ser útil.

Externamente podrá responderse solo:

```text id="6bb8nj"
resource changed concurrently
```

---

# 139. Telemetry

Métricas propuestas:

```text id="dyoizk"
db.optimistic_lock.conflict
db.optimistic_lock.update_success
db.optimistic_lock.delete_conflict
db.optimistic_lock.entity_type
```

---

# 140. Cardinality control

No usar:

```text id="fa62on"
entity id
expected version
actual version
user id
```

como metric labels.

---

# 141. Safe labels

Podrán utilizarse:

```text id="7za314"
entity_type
operation
strategy
platform
```

si el número de entity types está controlado.

---

# 142. Trace event

Ejemplo:

```text id="oxw9zc"
event:
    db.optimistic_lock.conflict

attributes:
    entity.type=Order
    operation=update
    strategy=integer
```

---

# 143. Structured logging

```text id="xbqdtn"
Optimistic lock conflict

Entity:
    Order

Key:
    [redacted/hash or debug-only]

Expected Version:
    7

Operation:
    UPDATE

Transaction:
    tx-42
```

La exposición del key/version dependerá de logging policy.

---

# 144. Conflict diagnostics

API conceptual:

```php id="fbjii1"
DB::concurrency()
    ->optimistic()
    ->explain($exception);
```

---

# 145. Diagnostic example

```text id="u40fr4"
OPTIMISTIC LOCK CONFLICT

Entity:
    Order

Operation:
    UPDATE

Strategy:
    INTEGER

Expected Version:
    7

Affected Rows:
    0

Classification:
    STALE_OR_MISSING

Transaction:
    ROLLBACK REQUIRED

Recommendation:
    Reload current state and re-evaluate the business operation.
```

---

# 146. Enriched diagnostic

Cuando actual version sea consultada:

```text id="1hexsq"
Expected:
    7

Observed After Conflict:
    9

Observation Confidence:
    POST_CONFLICT_READ

Note:
    This value was observed after the failed CAS and may not represent
    the exact intermediate value at the instant of conflict.
```

---

# 147. Conflict resolver boundary

VoltStack podrá definir un contrato superior:

```php id="tqq5lr"
interface OptimisticConflictResolver
{
    public function resolve(
        OptimisticConflictContext $context,
    ): ConflictResolution;
}
```

pero no habilitará merges automáticos por default.

---

# 148. Conflict resolutions

Conceptualmente:

```php id="iwpv7n"
enum ConflictResolution
{
    case ABORT;
    case RELOAD;
    case RETRY_BUSINESS_OPERATION;
    case USER_INTERVENTION;
    case CUSTOM;
}
```

---

# 149. Core default

```text id="et1b51"
ABORT / REPORT CONFLICT
```

será el default seguro.

---

# 150. Conflict resolution ≠ transaction retry

Un conflict resolver puede decidir:

```text id="ma99ei"
reload and re-run business logic
```

Eso es conceptualmente distinto de:

```text id="q4a5hn"
retry same physical transaction callback due deadlock
```

---

# 151. Exception hierarchy

```text id="887sdk"
DatabaseConcurrencyException
│
├── OptimisticLockException
│   ├── StaleEntityVersionException
│   ├── OptimisticDeleteConflictException
│   ├── OptimisticVersionMissingException
│   ├── OptimisticVersionAmbiguousException
│   └── OptimisticVersionMappingException
│
└── OptimisticLockInvariantViolationException
```

---

# 152. StaleEntityVersionException

Se utilizará cuando exista evidencia suficiente de que:

```text id="lf4pua"
database version
≠
expected version
```

---

# 153. Version missing exception

Ejemplo:

```text id="fh6f7a"
managed partial entity
+
optimistic update
+
version not loaded
```

---

# 154. Mapping exception

Ejemplos:

```text id="ybitbr"
multiple #[Version] fields
unsupported type
mutable custom version without comparator
invalid generator
```

---

# 155. Invariant exception

Se usará para estados internos imposibles, como:

```text id="d6s7y1"
optimistic update planned
without expected token
```

---

# 156. Metadata validation

En bootstrap deberá validarse:

```text id="5dgbrv"
at most one version property
type supports equality
generator compatible
field writable by persistence layer
field included in mapping
strategy supported
```

---

# 157. One version field per entity

Regla base:

```text id="3s3scv"
EntityType
→
0 or 1 optimistic version definition
```

---

# 158. Composite version

Si se desea un token compuesto, deberá modelarse como:

```text id="i0t25m"
single logical OptimisticVersionToken
```

aunque físicamente pueda derivarse de múltiples columnas mediante custom extension.

---

# 159. Version type requirements

Un version token deberá tener:

```text id="p36q50"
stable canonical equality
```

---

# 160. Floating point prohibited

Por default:

```text id="gwo8qn"
FLOAT/DOUBLE
```

no serán apropiados como version tokens.

---

# 161. JSON version prohibited by default

Un JSON arbitrario tampoco será un version token apropiado salvo custom strategy explícita.

---

# 162. Mutable object token

Si el PHP representation es mutable, deberá existir snapshot/comparator seguro.

Preferencia:

```text id="adg958"
immutable token value object
```

---

# 163. Persistent runtime safety

Metadata compartible:

```text id="r44e7u"
OptimisticLockMetadata
VersionStrategy definitions
Compiled accessors
Version generators stateless
```

podrá ser application-scoped.

---

# 164. Mutable runtime state

No compartible:

```text id="lbbapv"
ExpectedVersion per entity
Entity snapshots
Conflict context
Affected-row results
```

---

# 165. FrankenPHP

Request A:

```text id="qmd9f7"
Order#1 expected v7
```

Request B no deberá heredar:

```text id="aj0w65"
expected v7
```

de A.

Esto se garantiza porque la versión esperada vive en:

```text id="08agee"
EntitySnapshot / scoped PersistenceContext
```

---

# 166. RoadRunner

Cada operation deberá comenzar con:

```text id="zajfr9"
fresh PersistenceContext state
```

según lifecycle del ORM.

---

# 167. OpenSwoole

Dos coroutines no compartirán:

```text id="26ii16"
EntitySnapshot
ExpectedVersion
ConflictContext
```

aunque utilicen metadata compilada común.

---

# 168. No static expected versions

Prohibido:

```php id="koaq8f"
static array $versions = [];
```

en un componente compartido.

---

# 169. Performance characteristics

Costo principal de optimistic locking integer-based:

```text id="srui79"
one additional predicate
+
one version assignment
+
affected-row verification
```

sin requerir un SELECT extra en el camino exitoso.

---

# 170. No automatic pre-update SELECT

Prohibido como default:

```text id="zjelwj"
SELECT current version
UPDATE
```

porque introduce:

- round trip adicional;
- race entre SELECT y UPDATE;
- menor eficiencia.

---

# 171. Correct CAS

Preferir:

```text id="gy8h52"
single conditional UPDATE
```

---

# 172. Post-conflict SELECT

Solo para diagnostics/reconciliation opcional.

---

# 173. Index requirements

La condición:

```text id="n2ye6i"
WHERE primary_key = ?
AND version = ?
```

normalmente aprovechará el índice/PK sobre identidad.

No es necesariamente necesario indexar `version` por separado.

---

# 174. Composite keys

Para identidad compuesta:

```text id="b5w16f"
WHERE key_a = ?
AND key_b = ?
AND version = ?
```

---

# 175. Tenant context

En multitenancy, el predicate real puede incluir además:

```text id="jf5ubc"
tenant constraint
```

o estar implícitamente aislado por conexión/schema.

---

# 176. Tenant safety

El optimistic predicate no sustituirá:

```text id="8pz0g8"
tenant isolation
```

---

# 177. Query correctness

La operación debe asegurar que:

```text id="kpyfbr"
Identity Predicate
∧
Execution Domain Predicate
∧
Expected Version Predicate
```

identifican exactamente el recurso esperado.

---

# 178. Soft-delete filters

Si el ORM aplica soft-delete conditions, deberá definirse si estas forman parte de optimistic update/delete.

Ejemplo:

```text id="2drxz1"
WHERE id = ?
AND version = ?
AND deleted_at IS NULL
```

---

# 179. Multiple failure causes

Affected rows = 0 podría significar:

- stale version;
- already soft-deleted;
- tenant mismatch;
- row missing.

La clasificación deberá ser conservadora.

---

# 180. Avoid information leakage

No realizar automáticamente múltiples queries para distinguir causas cuando eso pueda revelar recursos fuera del execution domain.

---

# 181. Testing architecture

La suite deberá incluir:

```text id="u2h7ji"
OptimisticLockMetadataTests
OptimisticIntegerVersionTests
OptimisticTimestampVersionTests
OptimisticOpaqueVersionTests
OptimisticHydrationTests
OptimisticSnapshotTests
OptimisticUpdateTests
OptimisticDeleteTests
OptimisticConflictTests
OptimisticRelationshipTests
OptimisticBulkOperationTests
OptimisticTransactionTests
OptimisticRetryTests
OptimisticCacheTests
OptimisticPersistentRuntimeTests
OptimisticPlatformConformanceTests
```

---

# 182. Basic concurrent update test

```text id="1vrvj6"
Row:
    version 1

EM-A loads v1
EM-B loads v1

A updates:
    WHERE version=1
    SET version=2
    success

B updates:
    WHERE version=1
    SET version=2
    affects 0
```

Esperado:

```text id="w94po9"
OptimisticLockException
```

---

# 183. Delete conflict test

A carga v4.

B modifica a v5.

A remove:

```text id="h3d9pt"
DELETE WHERE id=? AND version=4
```

Esperado:

```text id="srb3yh"
OptimisticDeleteConflictException
```

---

# 184. Partial hydration test

Managed entity sin version loaded intenta flush.

Esperado:

```text id="1os9xw"
OptimisticVersionMissingException
```

antes de ejecutar UPDATE inseguro.

---

# 185. Version mutation test

Application modifica manualmente un ORM-managed version field.

Esperado:

```text id="xnerm6"
mapping/runtime policy violation
```

---

# 186. No-change test

Entidad sin cambios:

```text id="5dim6s"
flush
```

no deberá incrementar versión.

---

# 187. Integer increment test

```text id="mxgc3z"
v10 → update → v11
```

confirmar entity + snapshot reconciliation.

---

# 188. Rollback test

```text id="ohudvf"
v10
UPDATE succeeds to v11
ROLLBACK
```

DB vuelve a v10.

Verificar que el framework no afirme automáticamente que todos los objetos PHP regresaron a v10.

---

# 189. Commit unknown test

```text id="qgziwr"
CAS update success
COMMIT sent
connection lost
```

Esperado:

```text id="3xv1y7"
PersistenceContext TAINTED
version outcome UNKNOWN
no blind retry
```

---

# 190. Concurrent delete test

A carga entity.

B deletes entity.

A updates.

Esperado:

```text id="jhztj3"
0 affected rows
```

clasificado como:

```text id="uzqac4"
STALE_OR_MISSING
```

si no existe evidence adicional.

---

# 191. Conflict enrichment test

Con diagnostics enrichment habilitado:

```text id="sclnde"
post-conflict read
```

debe marcar actual version como:

```text id="cae330"
observed after conflict
```

no como causa atómica garantizada.

---

# 192. Bulk update test

Sobre versioned entity:

```text id="0j64nr"
bulk update
```

deberá:

- bypass explícito;
- warning en strict mode;
- no emitir falsa garantía optimistic.

---

# 193. Relationship test

Configurar:

```text id="yo75bi"
MembershipChanges → bump owner version
```

y verificar conflicto concurrente.

---

# 194. Aggregate policy test

Sin aggregate version policy, cambiar child no deberá incrementar parent version automáticamente.

---

# 195. Retry test

Optimistic conflict no deberá activar el mismo automatic retry policy usado para deadlocks salvo opt-in explícito.

---

# 196. HTTP mapping test

El Database subsystem deberá funcionar sin:

```text id="yb4q8j"
ETag
HTTP Request
```

demostrando separación arquitectónica.

---

# 197. Persistent worker test

Secuencia:

```text id="xxut8f"
Request A:
    expected v5
    conflict

Request B:
    load same entity at v6
```

B debe operar con:

```text id="in692j"
v6
```

sin leakage de conflict context.

---

# 198. Coroutine test

Dos PersistenceContexts simultáneos:

```text id="td10vg"
Coroutine A → expected v8
Coroutine B → expected v9
```

deben permanecer independientes.

---

# 199. Platform conformance

Cada plataforma deberá verificar:

```text id="7hiz41"
affected row semantics
conditional update semantics
conditional delete semantics
RETURNING capability
database-generated version retrieval
integer overflow behavior
timestamp precision
```

---

# 200. Proposed directory structure

```text id="5s7331"
src/Quantum/Database/Concurrency/
│
├── Optimistic/
│   ├── OptimisticLockMetadata.php
│   ├── OptimisticVersionStrategy.php
│   ├── OptimisticVersionToken.php
│   ├── OptimisticConflictKind.php
│   │
│   ├── Version/
│   │   ├── IntegerVersionToken.php
│   │   ├── TimestampVersionToken.php
│   │   ├── OpaqueVersionToken.php
│   │   ├── VersionGenerator.php
│   │   ├── VersionInitializationPolicy.php
│   │   └── VersionComparator.php
│   │
│   ├── Persistence/
│   │   ├── OptimisticVersionChange.php
│   │   ├── OptimisticUpdateOperation.php
│   │   ├── OptimisticDeleteOperation.php
│   │   └── OptimisticPersistenceVerifier.php
│   │
│   ├── Relationship/
│   │   ├── RelationshipVersionPolicy.php
│   │   └── AggregateVersionPolicy.php
│   │
│   ├── Conflict/
│   │   ├── OptimisticConflictContext.php
│   │   ├── OptimisticConflictResolver.php
│   │   └── ConflictResolution.php
│   │
│   ├── Platform/
│   │   ├── OptimisticLockCapabilities.php
│   │   └── AffectedRowSemantics.php
│   │
│   ├── Diagnostics/
│   │   ├── OptimisticLockInspector.php
│   │   └── OptimisticConflictDiagnostic.php
│   │
│   ├── Telemetry/
│   │   └── OptimisticLockTelemetry.php
│   │
│   └── Exception/
│       ├── OptimisticLockException.php
│       ├── StaleEntityVersionException.php
│       ├── OptimisticDeleteConflictException.php
│       ├── OptimisticVersionMissingException.php
│       ├── OptimisticVersionMappingException.php
│       └── OptimisticLockInvariantViolationException.php
```

---

# 201. Dependency model

Permitido:

```text id="gx1ciz"
Optimistic Locking
    ↓
ORM Metadata Contracts
Entity Snapshot Contracts
UnitOfWork Changes
Persistence Planner
Query Model Contracts
Execution Result Contracts
Platform Capabilities
Telemetry/Diagnostics
```

No permitido:

```text id="6ha461"
Optimistic Locking
    ↓
HTTP globals
Entity-specific business logic
PDO directly
static expected versions
automatic merge engine
automatic generic retry
```

---

# 202. Architectural invariants

## DB-OL-001
Optimistic locking utilizará un expected version token.

## DB-OL-002
Expected version representará una versión previamente observada.

## DB-OL-003
Expected version no se obtendrá silenciosamente del valor mutable actual si el baseline existe.

## DB-OL-004
Optimistic locking utilizará compare-and-swap semantics.

## DB-OL-005
Identity y expected version formarán parte de la condición de escritura.

## DB-OL-006
Version advancement formará parte de la misma operación lógica protegida.

## DB-OL-007
Optimistic update no se dividirá en check y write no atómicos.

## DB-OL-008
Pre-update SELECT no será el mecanismo principal.

## DB-OL-009
Affected-row verification será obligatoria.

## DB-OL-010
Expected affected rows para entity update singular será uno.

## DB-OL-011
Zero affected rows será conflicto/missing/ambiguous hasta clasificación.

## DB-OL-012
Zero affected rows no se interpretará ciegamente como stale version.

## DB-OL-013
Platform affected-row semantics serán capability-driven.

## DB-OL-014
Version increment ayudará a diferenciar matched no-op updates.

## DB-OL-015
Entidad sin cambios no incrementará versión por default.

## DB-OL-016
Version field será explícitamente mapeado.

## DB-OL-017
Field llamado version no activará locking automáticamente.

## DB-OL-018
Cada EntityType tendrá como máximo una definición optimistic principal.

## DB-OL-019
Version field gestionado no será mutable libremente por application code.

## DB-OL-020
Version token tendrá canonical equality.

## DB-OL-021
Integer version será strategy portable principal.

## DB-OL-022
Integer version verificará overflow.

## DB-OL-023
Timestamp version no se asumirá segura automáticamente.

## DB-OL-024
updated_at no equivaldrá automáticamente a optimistic version.

## DB-OL-025
Opaque version será soportable.

## DB-OL-026
Database-generated token será capability-driven.

## DB-OL-027
Custom token será explícitamente registrado.

## DB-OL-028
Float no será version type por default.

## DB-OL-029
Mutable custom token requerirá comparator/snapshot correcto.

## DB-OL-030
Immutable version value será preferido.

## DB-OL-031
Hydration cargará version para managed versioned entities.

## DB-OL-032
Managed optimistic entity requerirá full identifier.

## DB-OL-033
Managed optimistic update requerirá known version.

## DB-OL-034
Partial entity sin version no podrá realizar update protegido.

## DB-OL-035
DTO/projection no requerirá version si no se persiste como entity.

## DB-OL-036
EntitySnapshot almacenará observed version.

## DB-OL-037
Snapshot version será expected version normal.

## DB-OL-038
Version change será system-generated persistence change.

## DB-OL-039
Application ChangeSet no tratará version bump como business mutation ordinaria.

## DB-OL-040
Persistence Planner conocerá optimistic operation semantics.

## DB-OL-041
ORM no generará SQL directly.

## DB-OL-042
Query Model representará expected version predicate.

## DB-OL-043
SQL Compiler producirá dialect SQL.

## DB-OL-044
Query Executor no interpretará entity version conflicts.

## DB-OL-045
Persistence layer interpretará execution result.

## DB-OL-046
Successful update deberá reconciliar entity version.

## DB-OL-047
Successful update deberá reconciliar snapshot version.

## DB-OL-048
Statement success no equivaldrá a commit success.

## DB-OL-049
Rollback no rebobinará automáticamente entity version.

## DB-OL-050
Rollback no rebobinará automáticamente snapshot/object graph.

## DB-OL-051
Commit UNKNOWN producirá persistence uncertainty.

## DB-OL-052
Commit UNKNOWN no habilitará blind optimistic retry.

## DB-OL-053
Optimistic delete incluirá expected version.

## DB-OL-054
Versioned remove no degradará silenciosamente a delete-by-id.

## DB-OL-055
Soft delete podrá participar en versioning.

## DB-OL-056
Soft delete version policy será explícita.

## DB-OL-057
Relationship modifications no incrementarán owner version universalmente.

## DB-OL-058
Relationship versioning será policy-driven.

## DB-OL-059
Many-to-many membership no se asumirá protegido por owner version.

## DB-OL-060
Aggregate-level versioning será distinto de row-level entity versioning.

## DB-OL-061
Aggregate version policy será explícita.

## DB-OL-062
Detached updates podrán usar explicit expected version.

## DB-OL-063
Detached version token será validado.

## DB-OL-064
Detached entity merge no será automático.

## DB-OL-065
Optimistic conflict no implicará automatic merge.

## DB-OL-066
Conflict resolution pertenecerá a capa superior.

## DB-OL-067
Default conflict resolution será abort/report.

## DB-OL-068
Optimistic conflict no será deadlock.

## DB-OL-069
Optimistic conflict no será lock timeout.

## DB-OL-070
Optimistic conflict no será serialization failure.

## DB-OL-071
Optimistic conflict será concurrency failure separado.

## DB-OL-072
Automatic deadlock retry policy no se aplicará automáticamente a optimistic conflict.

## DB-OL-073
Blind retry usando nueva actual version estará prohibido.

## DB-OL-074
Retry de optimistic conflict requerirá reevaluación de business logic.

## DB-OL-075
Fresh entity reload será preferido para conflict retry.

## DB-OL-076
Fresh PersistenceContext será preferido para retry.

## DB-OL-077
Optimistic locking será complementario a transaction isolation.

## DB-OL-078
Optimistic locking no sustituirá transaction atomicity.

## DB-OL-079
Optimistic locking no sustituirá constraints.

## DB-OL-080
Optimistic locking no sustituirá authorization.

## DB-OL-081
Optimistic locking no será pessimistic locking.

## DB-OL-082
Optimistic locking no mantendrá row lock desde lectura por definición.

## DB-OL-083
Multi-entity atomic changes requerirán transaction boundary apropiado.

## DB-OL-084
Conflict dentro de transaction podrá requerir rollback.

## DB-OL-085
Bulk update será semánticamente distinto de managed optimistic update.

## DB-OL-086
Bulk DML no fingirá per-entity optimistic checks.

## DB-OL-087
Bulk version increment no equivaldrá a expected-version CAS.

## DB-OL-088
Strict mode podrá advertir bulk bypass.

## DB-OL-089
Upsert no será optimistic entity update por default.

## DB-OL-090
Optimistic upsert requerirá semántica explícita.

## DB-OL-091
Second-level cache podrá conservar version token.

## DB-OL-092
Cache no será optimistic lock authority.

## DB-OL-093
Database será autoridad final del CAS.

## DB-OL-094
HTTP ETag será integración separada.

## DB-OL-095
Database version token no será necesariamente HTTP ETag.

## DB-OL-096
Version token no será authorization credential.

## DB-OL-097
Public conflict response no expondrá actual version automáticamente.

## DB-OL-098
Diagnostics internos podrán enriquecer conflicto bajo policy.

## DB-OL-099
Post-conflict actual-version query no será evidencia atómica exacta del instante del conflicto.

## DB-OL-100
Conflict kind conservará ambigüedad cuando corresponda.

## DB-OL-101
STALE_VERSION solo se declarará con evidencia suficiente.

## DB-OL-102
ENTITY_MISSING será distinto de stale version cuando pueda demostrarse.

## DB-OL-103
RESULT_AMBIGUOUS será first-class.

## DB-OL-104
Mapping validation ocurrirá en bootstrap.

## DB-OL-105
Multiple version mappings serán inválidos.

## DB-OL-106
Version generator incompatible fallará temprano.

## DB-OL-107
Platform capability incompatible fallará temprano.

## DB-OL-108
Initial version strategy será explícita.

## DB-OL-109
NEW entity no tendrá ordinary expected-version conflict.

## DB-OL-110
Successful insert establecerá initial persisted version.

## DB-OL-111
Generated version será reconciliada después de insert.

## DB-OL-112
Database-generated token requerirá retrieval capability apropiada.

## DB-OL-113
Version token retrieval no realizará SQL arbitrario desde ORM.

## DB-OL-114
RETURNING se utilizará capability-driven.

## DB-OL-115
No RETURNING tendrá estrategia explícita.

## DB-OL-116
Version update permanecerá atómica con protected changes.

## DB-OL-117
Integer CAS será portable cuando platform semantics sean compatibles.

## DB-OL-118
Composite identity será compatible.

## DB-OL-119
Tenant execution domain será parte de query safety.

## DB-OL-120
Optimistic version no sustituirá tenant isolation.

## DB-OL-121
Soft-delete predicates podrán afectar conflict classification.

## DB-OL-122
Tenant mismatch no se expondrá como resource existence leak.

## DB-OL-123
Optimistic metadata será immutable y worker-safe.

## DB-OL-124
Expected versions serán PersistenceContext-local.

## DB-OL-125
Conflict context será operation-local.

## DB-OL-126
Static expected-version maps estarán prohibidos.

## DB-OL-127
FrankenPHP requests no compartirán expected version state.

## DB-OL-128
RoadRunner operations no compartirán expected version state.

## DB-OL-129
OpenSwoole coroutines no compartirán entity snapshots.

## DB-OL-130
Compiled accessors podrán compartirse si son inmutables.

## DB-OL-131
Hot-path optimistic update no requerirá pre-read adicional.

## DB-OL-132
Conflict enrichment SELECT será opcional.

## DB-OL-133
Conflict telemetry será observational.

## DB-OL-134
Entity IDs no serán metric labels.

## DB-OL-135
Version values no serán metric labels.

## DB-OL-136
Raw SQL no será metric label.

## DB-OL-137
Entity type podrá ser low-cardinality telemetry dimension bajo governance.

## DB-OL-138
OptimisticLockException preservará EntityKey internamente cuando sea seguro.

## DB-OL-139
Original execution failure será preservado cuando exista.

## DB-OL-140
Optimistic conflict será explicable en diagnostics.

## DB-OL-141
Testing incluirá concurrent writers reales.

## DB-OL-142
Testing incluirá rollback semantics.

## DB-OL-143
Testing incluirá partial hydration.

## DB-OL-144
Testing incluirá detached update.

## DB-OL-145
Testing incluirá bulk bypass.

## DB-OL-146
Testing incluirá persistent runtimes.

## DB-OL-147
Custom version strategies requerirán conformance tests.

## DB-OL-148
Affected-row semantics requerirán platform conformance.

## DB-OL-149
Cuando el motivo exacto de cero filas afectadas no pueda demostrarse, VoltStack conservará la ambigüedad.

## DB-OL-150
VoltStack nunca eliminará silenciosamente el expected-version predicate para lograr que una escritura conflictiva tenga éxito.

---

# 203. Anti-patterns

## 203.1 Check then update

```text id="vjn74x"
SELECT version
    ↓
if version matches
    ↓
UPDATE without version predicate
```

**Incorrecto.**

Existe race entre ambos statements.

---

## 203.2 Blind overwrite after conflict

```text id="o20qhe"
conflict
→ read latest version
→ repeat same update with latest version
```

**Prohibido como default.**

---

## 203.3 updated_at automático

```text id="7o6czr"
has updated_at
→ optimistic lock enabled
```

**Rechazado.**

---

## 203.4 Mutable version by user

```php id="pr4j03"
$order->setVersion(999);
$order->save();
```

**Rechazado para ORM-managed version field.**

---

## 203.5 Bulk update pretending to be protected

```text id="ljlysc"
UPDATE all pending orders
→ claim every entity was optimistic-locked
```

**Incorrecto.**

---

## 203.6 Version increment separate statement

```text id="u93bin"
UPDATE fields
UPDATE version
```

**Rechazado.**

---

## 203.7 Retry conflict as deadlock

```text id="rlorim"
OptimisticLockException
→ automatic same-callback retry
```

**Rechazado por default.**

---

## 203.8 Entity version as authorization

```text id="nexq9u"
client knows version
→ allow update
```

**Prohibido.**

---

## 203.9 ORM-generated SQL directly

```php id="v8j22i"
$unitOfWork->executeSql(
    "UPDATE ... version ..."
);
```

**Rechazado.**

---

## 203.10 Assume rollback rewinds version

```text id="1eo3ad"
DB rollback
→ PHP entity version automatically restored
```

**Incorrecto.**

---

# 204. Master formulas

## Compare-and-swap

```text id="iv00vn"
CAS(Entity)
=
Update(
    Identity = ExpectedIdentity
    ∧
    Version = ExpectedVersion
)
```

---

## Update success

```text id="yc0k6u"
OptimisticUpdateSuccess
⇔
AffectedRows = 1
```

bajo affected-row semantics compatibles.

---

## Version progression

Para integer strategy:

```text id="nsf1ki"
NextVersion
=
ExpectedVersion + 1
```

---

## Expected version

Para entity managed:

```text id="859er8"
ExpectedVersion
=
EntitySnapshot.Version
```

---

## Conflict

```text id="ku1u8b"
AffectedRows = 0
→
OptimisticConflictCandidate
```

No necesariamente:

```text id="p4ig21"
AffectedRows = 0
→
Definitely StaleVersion
```

---

## Isolation relationship

```text id="pokb4k"
ConcurrencyProtection
=
TransactionIsolation
+
OptimisticVersionCAS
+
Constraints
+
DomainRules
```

---

## Rollback

```text id="pffljz"
DatabaseRollback
≠
EntityVersionObjectRewind
```

---

# 205. Modelo final

```text id="959fqy"
                 Entity Hydration
                       │
                       ▼
                 Version Observed
                       │
                       ▼
                  Entity Snapshot
                       │
                       ▼
                    UnitOfWork
                       │
                Change Detection
                       │
                       ▼
             Persistence Planner
                       │
                       ▼
         OptimisticUpdateOperation
                       │
          ┌────────────┴─────────────┐
          ▼                          ▼
    ExpectedVersion              NextVersion
          │                          │
          └────────────┬─────────────┘
                       ▼
                   Query Model
                       │
                       ▼
                  SQL Compiler
                       │
                       ▼
                  Query Executor
                       │
                       ▼
                    Database
                       │
                       ▼
                 Affected Rows
                  /          \
                 1            0
                 │            │
                 ▼            ▼
             SUCCESS      CONFLICT
                 │            │
                 ▼            ▼
         Reconcile Entity  OptimisticLockException
         + Snapshot              │
                                 ▼
                        Application Re-evaluation
```

---

# 206. Decisiones arquitectónicas finales

VoltStack implementará optimistic locking mediante un modelo de:

```text id="1pqul1"
Expected Version
+
Atomic Conditional Write
+
Version Advancement
+
Affected-Row Verification
```

El mecanismo principal será un:

```text id="9zkhio"
Compare-And-Swap
```

sobre identidad y versión.

La estrategia portable recomendada será:

```text id="mwcvna"
INTEGER VERSION
```

aunque la arquitectura soportará:

```text id="9frucu"
TIMESTAMP
OPAQUE
DATABASE_GENERATED
CUSTOM
```

cuando las capabilities lo permitan.

La versión observada se conservará en:

```text id="i1d4tc"
EntitySnapshot
```

y será esa versión, no un valor arbitrariamente mutable del objeto, la que forme el:

```text id="b8tt66"
ExpectedVersion
```

de una entidad managed.

La operación protegida deberá realizar atómicamente:

```text id="17hjqf"
verify expected version
+
apply changes
+
advance version
```

Nunca mediante:

```text id="5l54on"
SELECT
then
unconditional UPDATE
```

VoltStack mantendrá:

```text id="5b3ppq"
Optimistic Lock Conflict
≠
Deadlock
≠
Serialization Failure
```

y, a diferencia de muchos deadlocks, un optimistic conflict no activará automatic retry por default, porque suele indicar que:

```text id="lk70yx"
business state changed
```

y la intención de la aplicación debe reevaluarse.

Asimismo:

```text id="b1us5a"
Optimistic Locking
≠
Transaction Atomicity
```

por lo que cambios en múltiples entidades seguirán requiriendo una transacción cuando necesiten persistirse atómicamente.

Bulk DML permanecerá explícitamente separado de las garantías de una entidad managed:

```text id="ee3zxm"
Bulk Update
≠
Per-Entity Optimistic CAS
```

La integración HTTP podrá adaptar version tokens a mecanismos como ETags, pero el Database subsystem no dependerá de HTTP y los tokens de versión nunca se considerarán credenciales de autorización.

Finalmente, en caso de rollback o commit incierto:

```text id="rymu2v"
Database State
≠
PHP Object State
```

seguirá siendo una invariante fundamental, por lo que la reconciliación será responsabilidad del Persistence Consistency System y no una restauración ficticia ejecutada por Optimistic Locking.

La regla final será:

> **VoltStack nunca resolverá un conflicto optimista eliminando o relajando silenciosamente la condición de versión que lo detectó; un conflicto significa que el supuesto de frescura de la aplicación dejó de ser válido y deberá ser reevaluado explícitamente.**

---

# 207. Relación con Transaction & Concurrency

```text id="ppjca7"
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

# 208. Siguiente documento

```text id="1l4cho"
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md
```

El siguiente documento deberá definir en profundidad:

- pessimistic locking architecture;
- lock intents;
- shared locks;
- exclusive locks;
- `FOR UPDATE`;
- `FOR SHARE`;
- platform-specific lock modes;
- lock wait policy;
- `NOWAIT`;
- `SKIP LOCKED`;
- lock timeout;
- lock scope;
- row locking;
- range/predicate locking considerations;
- lock acquisition ordering;
- transaction requirement;
- connection affinity;
- isolation interaction;
- Query AST representation;
- Query Builder API;
- SQL Compiler integration;
- ORM integration;
- EntityManager lock API;
- relationship locking;
- batch workers and queues;
- deadlock interaction;
- retry interaction;
- lock diagnostics;
- telemetry;
- persistent-runtime safety;
- testing;
- architectural invariants.