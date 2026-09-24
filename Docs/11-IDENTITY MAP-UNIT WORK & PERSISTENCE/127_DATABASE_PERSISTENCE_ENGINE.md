# 127_DATABASE_PERSISTENCE_ENGINE.md

# VoltStack Quantum Database
## Database Persistence Engine

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 127 — Database Persistence Engine  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Persistence Engine` define la arquitectura responsable de transformar el estado preparado por el ORM en operaciones estructuradas de persistencia.

Recibe información procedente de:

```text
EntityManager
UnitOfWork
Entity State
Entity Snapshots
ChangeSets
Relationship State
Entity Metadata
```

y produce:

```text
Persistence Operations
Persistence Operation Graph
Persistence Requirements
Persistence Results
Reconciliation Information
```

sin generar ni ejecutar SQL directamente.

Principio central:

> **El Persistence Engine transforma un UnitOfWork preparado en operaciones ORM de persistencia estructuradas y reconciliables; no genera SQL, no ejecuta queries y no sustituye al Persistence Planner, Query Engine, Transaction Manager ni Execution Engine.**

---

# 2. Posición arquitectónica

La arquitectura acumulada es:

```text
Entity
  │
  ▼
EntityManager
  │
  ▼
UnitOfWork
  │
  ├── Entity State
  ├── Identity Map
  ├── Snapshots
  └── ChangeSets
  │
  ▼
Persistence Engine
  │
  ├── Entity Change Graph
  ├── Persistence Operations
  └── Persistence Operation Graph
  │
  ▼
Persistence Planner
  │
  ▼
Persistence Plan
  │
  ▼
Query Model / AST
  │
  ▼
Semantic Query Engine
  │
  ▼
Optimizer
  │
  ▼
Query Planner
  │
  ▼
SQL Compiler
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
Database
```

---

# 3. Separación fundamental

```text
UnitOfWork
≠
Persistence Engine
≠
Persistence Planner
≠
Query Engine
≠
SQL Compiler
≠
Execution Engine
```

Cada componente responde una pregunta distinta.

---

# 4. UnitOfWork

Responde:

> ¿Qué entidades y relaciones presentan trabajo pendiente de persistencia?

Ejemplo:

```text
User#10    DIRTY
Order#NEW  NEW
Post#40    REMOVED
```

---

# 5. Persistence Engine

Responde:

> ¿Qué operaciones ORM representan esos cambios?

Ejemplo:

```text
UpdateEntity(User#10)
InsertEntity(Order#temporary-27)
DeleteEntity(Post#40)
```

---

# 6. Persistence Planner

Responde:

> ¿En qué orden y bajo qué dependencias deben ejecutarse esas operaciones?

---

# 7. Query Engine

Responde:

> ¿Cómo representar cada operación mediante Query Model/AST?

---

# 8. SQL Compiler

Responde:

> ¿Cómo expresar el Query Model utilizando el dialecto de la plataforma?

---

# 9. Execution Engine

Responde:

> ¿Cómo ejecutar la operación compilada y capturar su resultado?

---

# 10. Regla de dependencia

La dirección será:

```text
ORM
 ↓
Persistence Engine
 ↓
Persistence Planner
 ↓
Query Engine
 ↓
Execution Engine
 ↓
Connection
 ↓
Driver
```

Nunca:

```text
Driver
→ Persistence Engine

Compiler
→ Entity

Connection
→ UnitOfWork
```

---

# 11. Persistence Engine ≠ SQL generator

Nunca:

```php
$sql = "UPDATE users SET name = ? WHERE id = ?";
```

dentro del Persistence Engine.

Debe producir algo equivalente a:

```php
new EntityUpdateOperation(
    entity: $entityKey,
    assignments: $changes,
    predicates: $identityPredicate,
);
```

---

# 12. Persistence Engine ≠ Executor

El engine tampoco realizará:

```php
$pdo->execute(...);
```

---

# 13. Persistence Engine ≠ Transaction Manager

No deberá decidir:

```text
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
```

---

# 14. Persistence Engine ≠ UnitOfWork

El `UnitOfWork` mantiene:

```text
managed state
```

mientras el Persistence Engine deriva:

```text
persistence intent
```

---

# 15. Persistence Engine ≠ Change Tracking

Change Tracking produce:

```text
ChangeSet
```

Persistence Engine consume ChangeSets.

---

# 16. Persistence Engine ≠ Entity Snapshot

Snapshot representa:

```text
known persisted baseline
```

Persistence Engine puede utilizarlo para construir operaciones y requisitos.

---

# 17. Persistence Engine ≠ Persistence Plan

El engine produce:

```text
PersistenceOperationGraph
```

El Planner lo transforma en:

```text
PersistencePlan
```

---

# 18. Entrada principal

Se propone:

```php
final readonly class PersistenceRequest
{
    public function __construct(
        public PreparedUnitOfWork $unitOfWork,
        public PersistenceContext $context,
        public PersistenceMode $mode,
        public PersistenceOptions $options,
    ) {}
}
```

---

# 19. PreparedUnitOfWork

El Persistence Engine no debería trabajar sobre un UnitOfWork arbitrariamente mutable.

Antes:

```text
UnitOfWork
   ↓
change detection
   ↓
cascade discovery
   ↓
snapshot stabilization
   ↓
state validation
   ↓
PreparedUnitOfWork
```

---

# 20. PreparedUnitOfWork

Representará una vista estable del trabajo pendiente:

```php
final readonly class PreparedUnitOfWork
{
    public function __construct(
        public UnitOfWorkId $id,
        public UnitOfWorkGeneration $generation,
        public PreparedEntityChangeCollection $entities,
        public PreparedRelationshipChangeCollection $relationships,
        public PreparedCollectionChangeCollection $collections,
        public UnitOfWorkFingerprint $fingerprint,
    ) {}
}
```

---

# 21. Prepared ≠ immutable entities

Las entidades PHP todavía pueden ser mutables.

Lo congelado es:

```text
persistence input generation
```

---

# 22. Concurrent mutation during flush

Si una entidad cambia después de preparar el UnitOfWork:

```text
Generation N
→ persistence execution

entity mutates
→ Generation N+1
```

la nueva mutación no deberá mezclarse silenciosamente con el plan de `N`.

---

# 23. Flush generation

Cada flush podrá asociarse a:

```text
PersistenceGeneration
```

---

# 24. PersistenceGeneration

```php
final readonly class PersistenceGeneration
{
    public function __construct(
        public int $value,
    ) {}
}
```

---

# 25. Generation ≠ transaction

Una generación de persistencia no implica una transacción.

---

# 26. Generation ≠ batch

Tampoco implica:

```text
SQL batch
Migration batch
DB transaction
```

---

# 27. Arquitectura interna

```text
Prepared UnitOfWork
        │
        ▼
Persistence Input Validator
        │
        ▼
Entity Change Graph Builder
        │
        ▼
Persistence Intent Resolver
        │
        ▼
Persistence Operation Factory
        │
        ▼
Persistence Operation Graph
        │
        ▼
Operation Normalizer
        │
        ▼
Operation Validator
        │
        ▼
Persistence Engine Result
        │
        ▼
Persistence Planner
```

---

# 28. Entity Change Graph

Antes de crear operaciones físicas, VoltStack representará las relaciones lógicas entre cambios.

```text
EntityChangeGraph
```

---

# 29. Ejemplo

Supongamos:

```text
Customer NEW
Order NEW
Order.customer = Customer
```

El grafo conceptual:

```text
Customer
   │
   │ referenced by
   ▼
Order
```

---

# 30. EntityChangeGraph ≠ PersistenceOperationGraph

El primero representa:

```text
entity-level change dependencies
```

El segundo:

```text
persistence operation dependencies
```

---

# 31. EntityChangeNode

```php
final readonly class EntityChangeNode
{
    public function __construct(
        public EntityChangeNodeId $id,
        public ManagedEntityHandle $entity,
        public EntityPersistenceIntent $intent,
        public ?EntityKey $entityKey,
        public EntityType $entityType,
        public EntityChangeDescriptor $change,
    ) {}
}
```

---

# 32. EntityChangeEdge

```php
final readonly class EntityChangeEdge
{
    public function __construct(
        public EntityChangeNodeId $from,
        public EntityChangeNodeId $to,
        public EntityChangeDependencyType $type,
        public EntityChangeDependencyMetadata $metadata,
    ) {}
}
```

---

# 33. Dependency types

Se proponen:

```php
enum EntityChangeDependencyType
{
    case REQUIRES_IDENTITY;
    case REQUIRES_EXISTENCE;
    case OWNS_FOREIGN_KEY;
    case CASCADE_PERSIST;
    case CASCADE_REMOVE;
    case ORPHAN_DEPENDENCY;
    case COLLECTION_MEMBERSHIP;
    case VERSION_DEPENDENCY;
    case GENERATED_VALUE_DEPENDENCY;
    case EXTENSION_DEFINED;
}
```

---

# 34. Dependency ≠ execution order

Un edge indica una restricción semántica.

El Planner decidirá el orden físico.

---

# 35. Persistence intent

Cada cambio deberá transformarse en un intent explícito.

```php
enum EntityPersistenceIntent
{
    case INSERT;
    case UPDATE;
    case DELETE;
    case RELATIONSHIP_CHANGE;
    case COLLECTION_CHANGE;
    case NO_OPERATION;
}
```

---

# 36. Intent ≠ SQL verb

Aunque exista:

```text
INSERT
UPDATE
DELETE
```

como terminología ORM, no implica que la plataforma termine ejecutando exactamente una sentencia SQL homónima.

---

# 37. Ejemplo de emulación

Una operación conceptual:

```text
UPDATE
```

podría requerir:

```text
SELECT
UPDATE
RETURNING emulation
refresh
```

dependiendo de capabilities.

Eso se resolverá más abajo.

---

# 38. Persistence operation taxonomy

```text
PersistenceOperation
├── EntityPersistenceOperation
│   ├── InsertEntityOperation
│   ├── UpdateEntityOperation
│   └── DeleteEntityOperation
│
├── RelationshipPersistenceOperation
│   ├── AssociateRelationshipOperation
│   ├── DissociateRelationshipOperation
│   └── ReplaceRelationshipOperation
│
├── CollectionPersistenceOperation
│   ├── AddCollectionMemberOperation
│   ├── RemoveCollectionMemberOperation
│   ├── ReplaceCollectionMembershipOperation
│   └── ReorderCollectionOperation
│
├── ValidationPersistenceOperation
│
├── SynchronizationPersistenceOperation
│
└── ExtensionPersistenceOperation
```

---

# 39. Base contract

```php
interface PersistenceOperation
{
    public function id(): PersistenceOperationId;

    public function type(): PersistenceOperationType;

    public function target(): PersistenceTarget;

    public function requirements(): PersistenceRequirementSet;
}
```

---

# 40. PersistenceOperationId

Identifica una operación dentro del ciclo de persistencia.

```text
PersistenceOperationId
≠
EntityKey
≠
QueryId
≠
ExecutionId
```

---

# 41. InsertEntityOperation

```php
final readonly class InsertEntityOperation implements PersistenceOperation
{
    public function __construct(
        public PersistenceOperationId $id,
        public ManagedEntityHandle $entity,
        public EntityType $entityType,
        public PersistentValueSet $values,
        public GeneratedValueRequirementSet $generatedValues,
        public PersistenceRequirementSet $requirements,
    ) {}
}
```

---

# 42. Insert values

No serán:

```text
all entity properties
```

sino:

```text
mapped insertable persistent values
```

---

# 43. Non-insertable properties

Pueden excluirse:

```text
computed fields
database-generated values
read-only mapped values
derived properties
inverse relationships
```

---

# 44. Generated values

Una operación INSERT podrá requerir:

```text
generated primary key
sequence value
database default
timestamp
computed column
version token
```

---

# 45. GeneratedValueRequirement

```php
final readonly class GeneratedValueRequirement
{
    public function __construct(
        public PersistentPropertyId $property,
        public GeneratedValueKind $kind,
        public GeneratedValueRequirementMode $mode,
    ) {}
}
```

---

# 46. GeneratedValueRequirementMode

```php
enum GeneratedValueRequirementMode
{
    case REQUIRED;
    case OPTIONAL;
    case IGNORE;
}
```

---

# 47. UpdateEntityOperation

```php
final readonly class UpdateEntityOperation implements PersistenceOperation
{
    public function __construct(
        public PersistenceOperationId $id,
        public EntityKey $entity,
        public PersistentAssignmentSet $assignments,
        public PersistencePredicateSet $predicates,
        public GeneratedValueRequirementSet $generatedValues,
        public PersistenceRequirementSet $requirements,
    ) {}
}
```

---

# 48. Update assignments

Derivarán principalmente de:

```text
Stable ChangeSet
```

---

# 49. Empty ChangeSet

Un entity marked dirty con:

```text
effective changes = ∅
```

deberá convertirse normalmente en:

```text
NO_OPERATION
```

---

# 50. No-op suppression

Evita:

```text
UPDATE users SET ...
```

sin cambios reales.

---

# 51. DeleteEntityOperation

```php
final readonly class DeleteEntityOperation implements PersistenceOperation
{
    public function __construct(
        public PersistenceOperationId $id,
        public EntityKey $entity,
        public PersistencePredicateSet $predicates,
        public PersistenceRequirementSet $requirements,
    ) {}
}
```

---

# 52. Delete ≠ immediate SQL DELETE

Puede representar:

```text
hard delete
soft-delete extension
temporal close
custom persistence strategy
```

según capabilities/extensions posteriores.

---

# 53. Identity predicate

Una actualización/eliminación necesita identificar correctamente el target.

Conceptualmente:

```text
EntityKey
    ↓
Identifier Mapping
    ↓
Persistence Predicate
```

---

# 54. Identity ≠ string concatenation

Para composite identifiers:

```text
OrderItem(orderId=10, productId=20)
```

se conservarán componentes tipados.

---

# 55. Optimistic lock predicate

Un UPDATE puede requerir:

```text
identity predicate
+
version predicate
```

Ejemplo conceptual:

```text
id = 10
AND version = 7
```

---

# 56. PersistencePredicateSet ≠ SQL WHERE

Será una representación ORM estructurada.

---

# 57. Relationship operations

Una relación ORM no siempre equivale a modificar una FK.

Por tanto:

```text
RelationshipPersistenceOperation
```

será first-class.

---

# 58. To-one owning relationship

Ejemplo:

```text
Order.customer
Bob → Alice
```

puede traducirse a:

```text
UpdateEntityOperation(Order)
```

con cambio de FK.

---

# 59. Many-to-many

Ejemplo:

```text
User.roles
+ Admin
- Editor
```

puede producir:

```text
AddCollectionMemberOperation
RemoveCollectionMemberOperation
```

sobre una join structure.

---

# 60. Relationship operation ≠ query

La operación aún no sabe:

```text
SQL dialect
placeholder syntax
PDO binding
```

---

# 61. Collection operation

```php
final readonly class AddCollectionMemberOperation
{
    public function __construct(
        public PersistenceOperationId $id,
        public EntityKey $owner,
        public RelationshipId $relationship,
        public EntityReference $member,
        public PersistenceRequirementSet $requirements,
    ) {}
}
```

---

# 62. Collection snapshot integration

El engine podrá comparar:

```text
Collection Snapshot
vs
Current Collection State
```

pero preferentemente consumirá cambios ya estabilizados por Change Tracking/relationship tracking.

---

# 63. Persistence Engine no repite dirty checking

Regla:

> El Persistence Engine no deberá reconstruir desde cero el trabajo que corresponde al Change Tracking System.

---

# 64. Relationship ownership

Solo el lado con autoridad de persistencia deberá producir la operación física principal.

---

# 65. Inverse side

Modificar únicamente el inverse side puede:

```text
produce graph consistency diagnostics
```

sin necesariamente producir DB mutation.

---

# 66. Ownership mismatch

Ejemplo:

```text
User.orders inverse
Order.user owning
```

Si aplicación modifica solo:

```text
$user->orders->add($order)
```

sin sincronizar:

```text
$order->user
```

el ORM deberá seguir una política explícita.

---

# 67. Relationship consistency policies

```php
enum RelationshipConsistencyPolicy
{
    case STRICT;
    case SYNCHRONIZE_WHEN_UNAMBIGUOUS;
    case OWNING_SIDE_AUTHORITATIVE;
}
```

---

# 68. Recommended default

```text
OWNING_SIDE_AUTHORITATIVE
```

con diagnostics de inconsistencias.

---

# 69. Cascade integration

Cascades son reglas de propagación ORM.

```text
cascade persist
cascade remove
```

---

# 70. Cascade ≠ database cascade

```text
ORM Cascade
≠
ON DELETE CASCADE
```

---

# 71. Cascade discovery

Idealmente se completa antes de `PreparedUnitOfWork`.

---

# 72. Late cascade detection

Si el Persistence Engine encuentra una referencia NEW no registrada pero mapping permite cascade persist:

V1 debería preferir:

```text
reject unstable PreparedUnitOfWork
```

y pedir reconstrucción, antes que mutar silenciosamente la generación preparada.

---

# 73. Stable input principle

```text
PreparedUnitOfWork
```

deberá ser suficientemente cerrado para que el engine produzca resultados deterministas.

---

# 74. Orphan removal

Orphan removal puede transformar:

```text
relationship membership removal
```

en:

```text
DeleteEntityOperation
```

---

# 75. Orphan removal ≠ dissociation

Eliminar relación:

```text
Parent.children - Child
```

no siempre significa eliminar `Child`.

Solo si metadata lo especifica.

---

# 76. Orphan candidate

Se recomienda representar primero:

```text
OrphanCandidate
```

antes de producir DELETE definitivo.

---

# 77. Orphan analysis

Debe verificar:

```text
relationship ownership
orphan-removal mapping
current graph references
pending changes
```

---

# 78. Entity change graph

Ejemplo complejo:

```text
Customer NEW
   │
   ├─────┐
   ▼     ▼
OrderA  OrderB
   │
   ▼
LineItem
```

El engine preservará dependencias.

---

# 79. Persistence operation graph

Puede resultar:

```text
Insert Customer
      │
      ├───────────────┐
      ▼               ▼
Insert OrderA     Insert OrderB
      │
      ▼
Insert LineItem
```

---

# 80. Graph node

```php
final readonly class PersistenceOperationNode
{
    public function __construct(
        public PersistenceOperationId $id,
        public PersistenceOperation $operation,
    ) {}
}
```

---

# 81. Graph edge

```php
final readonly class PersistenceOperationEdge
{
    public function __construct(
        public PersistenceOperationId $from,
        public PersistenceOperationId $to,
        public PersistenceDependencyType $type,
    ) {}
}
```

---

# 82. PersistenceDependencyType

```php
enum PersistenceDependencyType
{
    case REQUIRES_BEFORE;
    case REQUIRES_AFTER;
    case REQUIRES_GENERATED_VALUE;
    case REQUIRES_EXISTENCE;
    case REQUIRES_DISSOCIATION;
    case REQUIRES_COLLECTION_CHANGE;
    case REQUIRES_VERSION_CHECK;
    case CONFLICTS_WITH;
    case EXTENSION_DEFINED;
}
```

---

# 83. Graph ≠ ordered list

No deberá reducirse prematuramente a:

```php
$operations = [A, B, C];
```

porque se perdería información de dependencias.

---

# 84. Planner ownership

El documento siguiente formalizará:

```text
Graph
→ ordered executable PersistencePlan
```

---

# 85. Cycles

Ejemplo:

```text
Entity A references B
Entity B references A
```

puede generar ciclo.

---

# 86. Cycle ≠ immediate error

Algunos ciclos pueden resolverse mediante:

```text
nullable intermediate FK
deferred constraints
generated IDs available before insert
two-phase association
platform capability
```

---

# 87. Cycle classification

```php
enum PersistenceCycleType
{
    case RESOLVABLE;
    case CONDITIONALLY_RESOLVABLE;
    case UNRESOLVABLE;
    case UNKNOWN;
}
```

---

# 88. Engine vs Planner cycle responsibility

Persistence Engine:

```text
detects/preserves semantic cycle
```

Persistence Planner:

```text
chooses resolution strategy
```

---

# 89. No hidden cycle breaking

El engine no deberá:

```text
set FK null
insert
update FK
```

silenciosamente.

---

# 90. Persistence requirements

Cada operación puede declarar requisitos.

```text
PersistenceRequirement
```

---

# 91. Requirement examples

```text
GENERATED_IDENTIFIER
RETURNING_VALUE
ACTIVE_TRANSACTION
OPTIMISTIC_VERSION_CHECK
FOREIGN_KEY_ORDERING
DEFERRED_CONSTRAINT
WRITE_CONNECTION
PRIMARY_CONNECTION
ENTITY_EXISTENCE
RELATIONSHIP_TARGET_EXISTENCE
```

---

# 92. Requirement ≠ capability

```text
Requirement
=
what operation needs

Capability
=
what platform can provide
```

---

# 93. Requirement resolution

Posteriormente:

```text
Requirement
+
Platform Capabilities
+
Planner
→ Strategy
```

---

# 94. Write intent

Persistence operations declararán:

```text
ConnectionIntent::WRITE
```

pero no resolverán una conexión.

---

# 95. Read/write routing

El Persistence Engine no deberá llamar:

```text
getPrimaryConnection()
```

---

# 96. Sharding

Tampoco:

```text
choose shard 3
```

por lógica vendor-specific.

Puede conservar:

```text
logical persistence target
```

proporcionado por `DatabaseContext`.

---

# 97. PersistenceTarget

```php
final readonly class PersistenceTarget
{
    public function __construct(
        public DatabaseContextKey $database,
        public ?PersistencePartitionKey $partition,
    ) {}
}
```

---

# 98. Target ≠ Connection

Un target es lógico.

---

# 99. Tenant context

Core no hardcodeará:

```text
tenant_id
```

El `EntityKey`/DatabaseContext ya debe preservar aislamiento.

---

# 100. Cross-target graph

Si un UnitOfWork contiene entidades de targets distintos:

```text
Target A
Target B
```

deberá detectarse.

---

# 101. No hidden distributed transaction

Nunca:

```text
two databases
→ pretend one atomic transaction
```

---

# 102. Cross-target policy

```php
enum CrossTargetPersistencePolicy
{
    case REJECT;
    case SPLIT_NON_ATOMIC;
    case EXPLICIT_COORDINATION_REQUIRED;
}
```

---

# 103. Default

Para V1:

```text
REJECT
```

dentro de un único atomic persistence scope salvo API explícita.

---

# 104. Persistence context

```php
final readonly class PersistenceContext
{
    public function __construct(
        public PersistenceContextId $id,
        public DatabaseContext $database,
        public TransactionContextView $transaction,
        public PlatformCapabilitySnapshot $capabilities,
        public PersistencePolicySet $policies,
    ) {}
}
```

---

# 105. Capability snapshot

El engine puede conocer capabilities inmutables necesarias para declarar requisitos.

No deberá consultar la DB escondidamente.

---

# 106. Version ≠ capability

Nunca:

```php
if ($mysqlVersion >= ...)
```

dentro del engine.

Preferir:

```php
$capabilities->supportsReturning();
```

---

# 107. MySQL/MariaDB separation

MariaDB continuará siendo plataforma first-class.

No deberá suponerse:

```text
MariaDB = MySQL with another version string
```

---

# 108. Operation normalization

Después de crear operaciones:

```text
raw persistence operations
```

se normalizarán.

---

# 109. Normalization examples

```text
remove duplicate collection membership operation
collapse no-op assignment
canonicalize identifier predicates
canonicalize operation metadata
merge compatible field assignments
```

---

# 110. Normalization ≠ optimization

Normalización:

```text
canonical semantics
```

Optimización:

```text
improved execution strategy
```

---

# 111. Persistence optimizer?

V1 no necesita introducir otro optimizer general.

El Query Optimizer ya existe.

Persistence-level optimizations específicas podrán vivir en:

```text
Persistence Planner
Batch Persistence
```

---

# 112. Operation deduplication

Ejemplo:

```text
add Role#1
remove Role#1
```

dentro de la misma estable generation puede resultar:

```text
NO_OPERATION
```

si no existen efectos intermedios observables requeridos.

---

# 113. Deduplication safety

No deberá eliminar:

```text
lifecycle-significant operations
version checks
required validations
extension operations
```

sin contrato.

---

# 114. Operation validation

Validaciones:

```text
valid EntityKey
valid mapping
required identifiers
valid change set
no immutable identifier mutation
valid relationship target
no impossible operation
no cross-context entity
no unsupported transient reference
```

---

# 115. Validation ≠ DB constraint validation

El engine puede detectar:

```text
structural ORM invalidity
```

pero no garantizar:

```text
UNIQUE constraint will pass
```

sin ejecución/consulta.

---

# 116. Immutable identifier mutation

Si ChangeSet contiene:

```text
id: 10 → 20
```

y mapping no permite rekey:

```text
EntityIdentifierMutationException
```

antes de ejecutar.

---

# 117. Cross-context entity

Una relación accidental:

```text
EntityManager A entity
→ EntityManager B entity
```

deberá rechazarse o convertirse explícitamente a una referencia compatible.

---

# 118. Detached entity reference

No toda referencia a detached entity es inválida.

Si tiene:

```text
known EntityKey
```

y mapping lo permite, puede utilizarse como:

```text
EntityReference
```

sin volverla managed.

---

# 119. Detached ≠ managed

Persistence Engine no deberá attach automáticamente objetos.

---

# 120. Transient relation target

Ejemplo:

```text
Order managed
Order.customer = new Customer()
```

sin cascade persist.

Debe producir:

```text
TransientRelationshipTargetException
```

---

# 121. Persistence result

El engine produce:

```php
final readonly class PersistenceEngineResult
{
    public function __construct(
        public PersistenceGeneration $generation,
        public EntityChangeGraph $entityChanges,
        public PersistenceOperationGraph $operations,
        public PersistenceRequirementSet $requirements,
        public PersistenceDiagnosticCollection $diagnostics,
        public PersistenceFingerprint $fingerprint,
    ) {}
}
```

---

# 122. Result ≠ execution result

`PersistenceEngineResult` significa:

```text
persistence work successfully derived
```

no:

```text
database changed
```

---

# 123. Persistence fingerprint

Representará de forma determinista:

```text
prepared UnitOfWork generation
mapping fingerprints
operation graph
logical target
relevant policies
```

---

# 124. Fingerprint ≠ ChangeSet fingerprint

Distintos niveles.

---

# 125. Fingerprint ≠ Query fingerprint

Tampoco.

---

# 126. Fingerprint uses

```text
stale plan detection
debugging
reconciliation correlation
testing
telemetry
```

---

# 127. Mapping drift

Si:

```text
PreparedUnitOfWork mapping fingerprint
≠
current mapping fingerprint
```

deberá bloquearse/reprepararse.

---

# 128. Entity mutation drift

Si el UnitOfWork generation cambió antes de planning:

```text
PreparedUnitOfWork stale
```

---

# 129. Stale input policy

Preferencia:

```text
REPREPARE
```

antes de ejecución.

Nunca mezclar generaciones.

---

# 130. Persistence lifecycle

```text
COLLECT
  ↓
PREPARE
  ↓
VALIDATE
  ↓
BUILD ENTITY GRAPH
  ↓
DERIVE INTENTS
  ↓
BUILD OPERATIONS
  ↓
NORMALIZE
  ↓
VALIDATE OPERATIONS
  ↓
FINGERPRINT
  ↓
SEAL
  ↓
PLAN
```

---

# 131. Engine state

El engine idealmente será:

```text
stateless service
```

o conservará solo configuración immutable.

---

# 132. Persistence session

El estado mutable de una ejecución estará en:

```php
final class PersistenceDerivationSession
{
}
```

---

# 133. Engine ≠ session

Permite reutilizar el engine de forma segura en persistent runtimes.

---

# 134. PersistenceDerivationSession

Puede contener:

```text
current generation
node registry
operation registry
diagnostics
temporary dependency graph
extension context
```

---

# 135. Session scope

```text
one persistence derivation
```

---

# 136. No cross-flush state

Después de derivar:

```text
session discarded
```

---

# 137. Query Model generation boundary

Existen dos opciones:

### Opción A

```text
Persistence Engine
→ Persistence Operations
→ Persistence Planner
→ Persistence Plan
→ Query Model Factory
```

### Opción B

```text
Persistence Engine
→ Persistence Operations containing Query Models
```

VoltStack utilizará **Opción A**.

---

# 138. Why Option A

Porque preserva:

```text
ORM semantics
```

separadas de:

```text
query representation
```

---

# 139. Persistence operation ≠ Query Model

```text
UpdateEntityOperation
≠
UpdateQuery
```

---

# 140. Example

ORM operation:

```text
Update User#10
changes:
    name Bob → Alice
expected version:
    7
generated:
    version
```

Después el Persistence Planner/Query factory puede producir:

```text
UpdateQueryModel
```

---

# 141. Query Engine remains authoritative

Una vez convertido a Query Model:

```text
semantic validation
optimization
query planning
compilation
```

siguen perteneciendo al Query Engine.

---

# 142. Persistence Planner

El siguiente documento deberá consumir:

```text
PersistenceOperationGraph
```

y producir:

```text
PersistencePlan
```

---

# 143. PersistencePlan responsibilities

Incluirá:

```text
ordered operation steps
dependency resolution
generated-value barriers
cycle resolution
batch opportunities
transaction requirements
query-generation strategy
reconciliation boundaries
```

---

# 144. Persistence Engine no ordena físicamente

Puede preservar edges, pero no deberá decidir prematuramente:

```text
step 1
step 2
step 3
```

salvo orden semántico obligatorio.

---

# 145. Generated identity dependency

Ejemplo:

```text
Customer NEW
Order NEW
Order.customer = Customer
```

Customer utiliza DB-generated ID.

Graph:

```text
InsertCustomer
      │
      │ REQUIRES_GENERATED_VALUE(customer.id)
      ▼
InsertOrder
```

---

# 146. Generated ID ≠ known before execution

El engine preserva:

```text
DeferredPersistentValue
```

---

# 147. DeferredPersistentValue

```php
final readonly class DeferredPersistentValue
{
    public function __construct(
        public PersistenceOperationId $producer,
        public PersistentPropertyId $property,
    ) {}
}
```

---

# 148. Deferred value ≠ lazy database query

Es una dependencia de resultado dentro del plan.

---

# 149. Generated value flow

```text
Insert Customer
      ↓
Execution Result
      ↓
customer.id = 100
      ↓
Generated Value Resolution
      ↓
Insert Order(customer_id=100)
```

---

# 150. Reconciliation architecture

Después de ejecutar, debe existir el camino inverso:

```text
Execution Results
      ↓
Persistence Reconciler
      ↓
Entity state
IdentityMap
Snapshots
UnitOfWork
Generated values
Version tokens
```

---

# 151. Persistence Engine vs Reconciler

El `Persistence Engine` es responsable de definir los contratos de reconciliación.

La mutación post-ejecución puede delegarse a:

```text
PersistenceReconciler
```

---

# 152. PersistenceReconciliationDescriptor

Cada operación podrá declarar:

```text
what result matters
what generated values are expected
what snapshot transition is possible
what entity state transition is expected
```

---

# 153. Example INSERT reconciliation

```text
Before:
EntityState = NEW
id = unassigned

Confirmed INSERT:
generated id = 100

After reconciliation:
id = 100
EntityKey established
IdentityMap registered
Snapshot established
EntityState = MANAGED
```

---

# 154. Example UPDATE reconciliation

```text
Before:
Snapshot generation = 4
version = 7

Confirmed UPDATE:
new version = 8

After:
Snapshot generation = 5
version = 8
EntityState = MANAGED synchronized
```

---

# 155. Example DELETE reconciliation

Confirmed DELETE may produce:

```text
EntityState = REMOVED/DETACHED
IdentityMap removal
Snapshot removal
UnitOfWork cleanup
```

La semántica exacta se coordinará con Entity State System.

---

# 156. Outcome certainty

Toda reconciliación dependerá de:

```text
PersistenceOutcomeCertainty
```

---

# 157. Certainty model

```php
enum PersistenceOutcomeCertainty
{
    case CONFIRMED;
    case CONFIRMED_NO_EFFECT;
    case PARTIALLY_CONFIRMED;
    case UNKNOWN;
}
```

---

# 158. CONFIRMED

Puede reconciliarse normalmente.

---

# 159. CONFIRMED_NO_EFFECT

No se deberá avanzar baseline como si hubiera cambio.

---

# 160. PARTIALLY_CONFIRMED

Solo operaciones con efectos conocidos podrán reconciliarse con certeza.

---

# 161. UNKNOWN

No deberá inventarse estado.

---

# 162. Unknown outcome principle

```text
UNKNOWN
≠
FAILED_WITH_NO_EFFECT
```

---

# 163. EntityManager tainting

Una persistencia UNKNOWN puede requerir:

```text
EntityManager::TAINTED
```

---

# 164. Why taint

Porque:

```text
DB state
```

y:

```text
managed object state
```

ya no pueden reconciliarse con certeza.

---

# 165. Retry

Persistence Engine no decidirá automáticamente:

```text
retry
```

---

# 166. Retryability ≠ replayability

```text
TransientFailure
```

no significa:

```text
safe to replay INSERT
```

---

# 167. Retry metadata

Una operación podrá declarar características útiles:

```text
idempotency
replayability
outcome sensitivity
```

pero la política final pertenece al Execution/Retry System.

---

# 168. INSERT retry risk

Si:

```text
INSERT committed
connection lost
```

repetir puede duplicar datos.

---

# 169. Client-generated identifiers

Pueden mejorar replayability, pero no garantizan por sí solos que toda operación sea segura de repetir.

---

# 170. Transactions

Persistence Engine puede declarar:

```text
transaction requirement
```

pero no abrir transacciones.

---

# 171. TransactionRequirement

```php
enum TransactionRequirement
{
    case NONE;
    case RECOMMENDED;
    case REQUIRED;
    case EXISTING_REQUIRED;
}
```

---

# 172. One flush ≠ one transaction

VoltStack no impondrá universalmente:

```text
flush() = BEGIN ... COMMIT
```

---

# 173. Default EntityManager policy

La capa superior podrá ofrecer:

```text
transactional flush
```

como default configurable.

Pero sigue siendo:

```text
EntityManager / Transaction coordination
```

no Persistence Engine.

---

# 174. Existing transaction

Si existe:

```text
TransactionContext
```

las operaciones se ejecutarán dentro de ella según Planner/Executor.

---

# 175. Transaction-local snapshot

La reconciliación respetará el modelo del documento 126.

---

# 176. Batch persistence

Persistence Engine podrá detectar operaciones compatibles con:

```text
batch insert
batch update
batch delete
```

pero no necesariamente agruparlas físicamente.

---

# 177. Batch eligibility

Puede anotar:

```text
BatchCompatibilityKey
```

---

# 178. BatchCompatibilityKey

Podría considerar:

```text
entity mapping
operation type
target
column set
generated-value requirements
locking requirements
result requirements
```

---

# 179. Batch hint ≠ batch plan

El documento 133 definirá:

```text
DATABASE_BATCH_PERSISTENCE_SYSTEM
```

---

# 180. Lifecycle integration

Persistence operations deberán conservar lifecycle requirements.

Ejemplo:

```text
prePersist
postPersist
```

---

# 181. Lifecycle callback ≠ operation

Los callbacks no deberán modelarse ingenuamente como SQL-like operations.

---

# 182. Lifecycle phases

El Persistence Engine puede generar:

```text
LifecycleBarrier
```

o metadata para que el plan preserve:

```text
PRE event
operation
POST event
```

---

# 183. postPersist ≠ committed

Como ya se estableció:

```text
postPersist
```

no significa necesariamente:

```text
transaction committed
```

---

# 184. Domain events

Persistence Engine no deberá asumir que:

```text
postPersist
=
publish domain event
```

---

# 185. Event dispatch after commit

Si una aplicación requiere:

```text
after commit
```

deberá integrarse con Transaction/Event System.

---

# 186. Validation integration

Entity validation no pertenece al Persistence Engine por defecto.

---

# 187. ORM structural validation

Sí pertenece verificar:

```text
operation can be represented from mapping/state
```

---

# 188. Business validation

No pertenece decidir:

```text
customer age is valid
```

---

# 189. Authorization

Persistence Engine no decide:

```text
current user may edit User#10
```

---

# 190. Security boundary

Authorization deberá ocurrir antes o mediante integración explícita.

---

# 191. Mass assignment

Tampoco pertenece al Persistence Engine.

---

# 192. SQL injection

Como Persistence Engine nunca concatena SQL:

```text
parameterized Query Model
```

será preservado downstream.

---

# 193. Raw values

Los valores seguirán tipados.

Nunca:

```text
'value' => "'Alice'"
```

como SQL fragment.

---

# 194. Raw expression

Si una operación requiere raw expression, deberá utilizar el escape hatch tipado definido en:

```text
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md
```

---

# 195. Persistence operation extensions

Extensiones posibles:

```text
Soft Delete
Temporal Persistence
Audit Columns
Custom Entity Storage
Polymorphic Persistence
Domain-specific generated values
```

---

# 196. Extension point

```php
interface PersistenceOperationContributor
{
    public function contribute(
        PreparedUnitOfWork $unitOfWork,
        PersistenceContributionContext $context,
    ): PersistenceContribution;
}
```

---

# 197. Contributor ≠ executor

Una extensión no deberá ejecutar SQL.

---

# 198. Contributor ≠ hidden UnitOfWork mutation

Debe devolver:

```text
structured contribution
```

---

# 199. Extension ordering

Será explícito mediante:

```text
priority
phase
dependency
```

---

# 200. No last-wins

Conflictos incompatibles producirán:

```text
PersistenceExtensionConflictException
```

---

# 201. Extension phases

```php
enum PersistenceExtensionPhase
{
    case BEFORE_GRAPH;
    case GRAPH_CONTRIBUTION;
    case OPERATION_CONTRIBUTION;
    case NORMALIZATION;
    case VALIDATION;
    case AFTER_DERIVATION;
}
```

---

# 202. Frozen extensions

En producción:

```text
PersistenceExtensionRegistry
```

deberá estar frozen.

---

# 203. Determinism

Mismos:

```text
PreparedUnitOfWork
metadata
capability snapshot
policies
extensions
context
```

deberán producir el mismo:

```text
PersistenceOperationGraph
```

semánticamente.

---

# 204. Runtime data

Datos operativos cambiantes no deberán consultarse escondidamente durante derivación.

---

# 205. Persistent runtime

Arquitectura:

```text
FrankenPHP Worker
│
├── Shared Immutable
│   ├── Persistence Engine
│   ├── Metadata
│   ├── Operation Factories
│   ├── Frozen Extension Registry
│   └── Policies
│
├── Request A
│   ├── EntityManager A
│   ├── UnitOfWork A
│   └── PersistenceSession A
│
└── Request B
    ├── EntityManager B
    ├── UnitOfWork B
    └── PersistenceSession B
```

---

# 206. Shared mutable state

Prohibido:

```php
private static array $pendingOperations;
```

---

# 207. Request termination

Debe liberar:

```text
PersistenceDerivationSession
temporary graph
temporary operations
temporary diagnostics
reconciliation context
```

---

# 208. No implicit persistence at worker reset

Nunca:

```text
worker cleanup
→ auto flush
```

---

# 209. RoadRunner/OpenSwoole

Misma regla de scope.

Especialmente en OpenSwoole:

```text
coroutine-local logical persistence context
```

---

# 210. Concurrency

Una instancia mutable de:

```text
PersistenceDerivationSession
```

no será coroutine-safe para dos flushes simultáneos.

---

# 211. Concurrent flush same EntityManager

V1 deberá rechazar:

```text
flush A
flush B
```

concurrentes sobre el mismo EntityManager.

---

# 212. Reentrancy

También deberá controlarse:

```text
prePersist callback
→ entityManager->flush()
```

---

# 213. Nested flush

Default:

```text
REJECT
```

---

# 214. Why

Nested flush puede invalidar:

```text
PreparedUnitOfWork generation
operation graph
snapshots
lifecycle ordering
```

---

# 215. Persistence lock

No es DB lock.

Puede existir un:

```text
EntityManagerFlushGuard
```

para impedir reentrancy.

---

# 216. Resource governance

Large UnitOfWork puede producir:

```text
large graphs
many operations
high memory usage
```

---

# 217. Limits

```php
final readonly class PersistenceResourceLimits
{
    public function __construct(
        public int $maxEntities,
        public int $maxOperations,
        public int $maxGraphEdges,
        public int $maxCascadeDepth,
        public int $maxRelationshipChanges,
        public int $maxDerivationMemoryBytes,
    ) {}
}
```

---

# 218. Limit exceeded

No truncar silenciosamente.

Debe:

```text
fail
```

o requerir estrategia explícita de batch/chunk.

---

# 219. Cascade explosion

Un pequeño `persist($root)` puede descubrir:

```text
100,000 entities
```

por cascade.

Debe ser observable y gobernable.

---

# 220. Graph depth

El engine deberá proteger contra:

```text
pathological recursive graphs
```

sin confundir ciclos válidos con recursion bugs.

---

# 221. Iterative traversal

Implementación puede preferir algoritmos iterativos para grafos grandes.

---

# 222. Memory release

Después de construir un immutable/sealed result:

```text
temporary traversal structures
```

deberán liberarse.

---

# 223. Telemetry

Métricas:

```text
orm.persistence.derivation.count
orm.persistence.derivation.duration
orm.persistence.entities
orm.persistence.operations
orm.persistence.graph.nodes
orm.persistence.graph.edges
orm.persistence.insert_operations
orm.persistence.update_operations
orm.persistence.delete_operations
orm.persistence.relationship_operations
orm.persistence.collection_operations
orm.persistence.noop_operations
orm.persistence.generated_value_dependencies
orm.persistence.cycles
orm.persistence.orphans
orm.persistence.cascade_depth
orm.persistence.cross_target_rejections
orm.persistence.validation_failures
orm.persistence.stale_generations
orm.persistence.extension_contributions
```

---

# 224. Tracing

Span principal:

```text
orm.persistence.derive
```

Subspans:

```text
orm.persistence.validate_input
orm.persistence.build_entity_graph
orm.persistence.resolve_intents
orm.persistence.build_operations
orm.persistence.normalize
orm.persistence.validate_operations
orm.persistence.fingerprint
```

---

# 225. Telemetry ≠ semantics

Desactivar telemetry no cambiará:

```text
operation graph
ordering constraints
validation
```

---

# 226. Diagnostics

Ejemplo:

```text
Persistence Generation: 14

Prepared UnitOfWork:
  Entities: 5
  Relationships: 3
  Collections: 1

Operations:
  INSERT: 2
  UPDATE: 1
  DELETE: 1
  RELATIONSHIP: 2
  COLLECTION: 2

Generated Values:
  Customer.id
  Order.id

Graph:
  nodes: 8
  edges: 11

Cycles:
  0

Target:
  default/write

Fingerprint:
  7b21...
```

---

# 227. Explainability

Debe poder explicarse:

```text
Why does this operation exist?
```

---

# 228. Operation provenance

Cada operación deberá preservar:

```text
source entity
source ChangeSet
source relationship change
mapping rule
cascade rule
extension contributor
```

---

# 229. PersistenceProvenance

```php
final readonly class PersistenceProvenance
{
    public function __construct(
        public PersistenceProvenanceSource $source,
        public ?EntityKey $entity,
        public ?ChangeSetId $changeSet,
        public ?RelationshipId $relationship,
        public ?ExtensionId $extension,
    ) {}
}
```

---

# 230. Provenance ≠ audit log

Es diagnostic/architectural provenance.

No sustituye:

```text
business audit
```

---

# 231. Failure model

Fases:

```text
INPUT
GRAPH
INTENT
OPERATION
NORMALIZATION
VALIDATION
SEALING
```

---

# 232. PersistenceDerivationFailure

```php
final readonly class PersistenceDerivationFailure
{
    public function __construct(
        public PersistenceFailurePhase $phase,
        public PersistenceFailureCode $code,
        public string $message,
        public PersistenceDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 233. Derivation failure ≠ DB failure

En este punto normalmente:

```text
no DB mutation has happened
```

---

# 234. Strong property

Como el Persistence Engine no ejecuta DB I/O:

```text
DerivationFailure
⇒
NoPersistenceEngineDatabaseSideEffect
```

---

# 235. Exception hierarchy

```text
DatabaseOrmException
└── PersistenceException
    ├── PersistenceEngineException
    ├── PersistenceInputException
    ├── InvalidPreparedUnitOfWorkException
    ├── StaleUnitOfWorkGenerationException
    ├── PersistenceGraphException
    ├── PersistenceGraphCycleException
    ├── UnresolvablePersistenceDependencyException
    ├── PersistenceIntentException
    ├── PersistenceOperationException
    ├── PersistenceOperationValidationException
    ├── PersistenceOperationConflictException
    ├── PersistenceNormalizationException
    ├── PersistenceRequirementException
    ├── GeneratedValueDependencyException
    ├── TransientRelationshipTargetException
    ├── CrossContextEntityException
    ├── CrossTargetPersistenceException
    ├── EntityIdentifierMutationException
    ├── OrphanResolutionException
    ├── RelationshipPersistenceException
    ├── CollectionPersistenceException
    ├── PersistenceExtensionException
    ├── PersistenceExtensionConflictException
    ├── PersistenceResourceLimitException
    ├── PersistenceReentrancyException
    ├── ConcurrentPersistenceException
    ├── PersistenceFingerprintException
    └── PersistenceInvariantException
```

---

# 236. Directory structure

```text
src/Quantum/Database/ORM/Persistence/
│
├── Contract/
│   ├── PersistenceEngine.php
│   ├── PersistenceOperation.php
│   ├── PersistenceOperationFactory.php
│   ├── PersistenceOperationContributor.php
│   └── PersistenceReconciler.php
│
├── Engine/
│   ├── DefaultPersistenceEngine.php
│   ├── PersistenceRequest.php
│   ├── PersistenceEngineResult.php
│   ├── PersistenceMode.php
│   └── PersistenceOptions.php
│
├── Preparation/
│   ├── PreparedUnitOfWork.php
│   ├── PreparedEntityChange.php
│   ├── PreparedRelationshipChange.php
│   └── PreparedCollectionChange.php
│
├── Generation/
│   ├── PersistenceGeneration.php
│   ├── PersistenceDerivationSession.php
│   └── PersistenceFingerprint.php
│
├── Graph/
│   ├── EntityChangeGraph.php
│   ├── EntityChangeNode.php
│   ├── EntityChangeEdge.php
│   ├── EntityChangeGraphBuilder.php
│   ├── PersistenceOperationGraph.php
│   ├── PersistenceOperationNode.php
│   ├── PersistenceOperationEdge.php
│   ├── PersistenceDependencyType.php
│   └── PersistenceCycleType.php
│
├── Intent/
│   ├── EntityPersistenceIntent.php
│   ├── PersistenceIntentResolver.php
│   └── PersistenceIntentCollection.php
│
├── Operation/
│   ├── Entity/
│   │   ├── InsertEntityOperation.php
│   │   ├── UpdateEntityOperation.php
│   │   └── DeleteEntityOperation.php
│   │
│   ├── Relationship/
│   │   ├── AssociateRelationshipOperation.php
│   │   ├── DissociateRelationshipOperation.php
│   │   └── ReplaceRelationshipOperation.php
│   │
│   ├── Collection/
│   │   ├── AddCollectionMemberOperation.php
│   │   ├── RemoveCollectionMemberOperation.php
│   │   ├── ReplaceCollectionMembershipOperation.php
│   │   └── ReorderCollectionOperation.php
│   │
│   ├── Validation/
│   ├── Synchronization/
│   └── Extension/
│
├── Value/
│   ├── PersistentValue.php
│   ├── PersistentValueSet.php
│   ├── PersistentAssignment.php
│   ├── PersistentAssignmentSet.php
│   ├── DeferredPersistentValue.php
│   └── GeneratedValueRequirement.php
│
├── Predicate/
│   ├── PersistencePredicate.php
│   ├── PersistencePredicateSet.php
│   ├── IdentityPersistencePredicate.php
│   └── VersionPersistencePredicate.php
│
├── Requirement/
│   ├── PersistenceRequirement.php
│   ├── PersistenceRequirementSet.php
│   ├── TransactionRequirement.php
│   └── GeneratedValueRequirementSet.php
│
├── Relationship/
│   ├── RelationshipPersistenceResolver.php
│   ├── RelationshipConsistencyPolicy.php
│   └── OrphanCandidate.php
│
├── Normalization/
│   ├── PersistenceOperationNormalizer.php
│   └── PersistenceNormalizationPipeline.php
│
├── Validation/
│   ├── PersistenceInputValidator.php
│   ├── PersistenceOperationValidator.php
│   └── PersistenceGraphValidator.php
│
├── Reconciliation/
│   ├── PersistenceReconciliationDescriptor.php
│   ├── PersistenceOutcomeCertainty.php
│   └── PersistenceReconciliationContext.php
│
├── Extension/
│   ├── PersistenceExtensionRegistry.php
│   ├── PersistenceExtensionPhase.php
│   ├── PersistenceContribution.php
│   └── PersistenceContributionContext.php
│
├── Runtime/
│   ├── PersistenceRuntimeScope.php
│   ├── EntityManagerFlushGuard.php
│   └── PersistenceRuntimeResetter.php
│
├── Resource/
│   ├── PersistenceResourceLimits.php
│   └── PersistenceResourceGuard.php
│
├── Telemetry/
│   ├── PersistenceTelemetry.php
│   ├── PersistenceProfiler.php
│   └── PersistenceDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 237. Public internal contract

```php
interface PersistenceEngine
{
    public function derive(
        PersistenceRequest $request,
    ): PersistenceEngineResult;
}
```

Debe ser deliberadamente pequeño.

---

# 238. Example conceptual flow

```php
$prepared = $unitOfWork->prepare();

$result = $persistenceEngine->derive(
    new PersistenceRequest(
        unitOfWork: $prepared,
        context: $persistenceContext,
        mode: PersistenceMode::FLUSH,
        options: PersistenceOptions::defaults(),
    ),
);

$plan = $persistencePlanner->plan($result);
```

No:

```php
$persistenceEngine->execute($result);
```

---

# 239. Testing strategy

Debe cubrir:

```text
unit tests
graph tests
operation derivation tests
relationship tests
cascade tests
orphan tests
generated-value tests
cycle tests
runtime tests
extension tests
determinism tests
resource tests
```

---

# 240. Test — NEW entity

```text
NEW User
→ InsertEntityOperation
```

---

# 241. Test — DIRTY entity

```text
MANAGED + ChangeSet
→ UpdateEntityOperation
```

---

# 242. Test — no-op dirty

```text
DIRTY marker + empty effective ChangeSet
→ NO_OPERATION
```

---

# 243. Test — removed entity

```text
REMOVED
→ DeleteEntityOperation
```

---

# 244. Test — assigned ID

INSERT conserva assigned identifier.

---

# 245. Test — generated ID

Produce generated-value requirement.

---

# 246. Test — parent generated ID

Child operation depende del resultado del parent.

---

# 247. Test — composite ID

Predicados preservan componentes tipados.

---

# 248. Test — identifier mutation

Se rechaza.

---

# 249. Test — optimistic version

Update contiene version requirement/predicate.

---

# 250. Test — relationship owning side

Produce operación apropiada.

---

# 251. Test — inverse side

No genera DB mutation incorrecta.

---

# 252. Test — many-to-many add

Produce membership operation.

---

# 253. Test — many-to-many remove

Produce removal operation.

---

# 254. Test — orphan removal

Produce orphan candidate/delete cuando corresponde.

---

# 255. Test — no orphan removal

No elimina entidad accidentalmente.

---

# 256. Test — cascade persist

Incluye entidades descubiertas durante preparation.

---

# 257. Test — missing cascade

Transient relationship target error.

---

# 258. Test — cascade cycle

No recursión infinita.

---

# 259. Test — graph cycle

Clasificación correcta.

---

# 260. Test — cross-context entity

Rechazo.

---

# 261. Test — detached reference

Permitir cuando identity contract sea válido.

---

# 262. Test — cross-target UnitOfWork

Default reject.

---

# 263. Test — no SQL

Ningún derivation test deberá requerir SQL string.

---

# 264. Test — no connection

Engine puede probarse sin conexión activa.

---

# 265. Test — no hidden query

Derivation no incrementa query count.

---

# 266. Test — deterministic graph

Mismos inputs → mismo graph.

---

# 267. Test — deterministic fingerprint

Mismos inputs → mismo fingerprint.

---

# 268. Test — stale generation

Rechazar.

---

# 269. Test — mapping drift

Rechazar/reprepare.

---

# 270. Test — extension contribution

Contribución estructurada.

---

# 271. Test — extension collision

No last-wins.

---

# 272. Test — resource limit

No truncar operaciones.

---

# 273. Test — nested flush

Rechazar.

---

# 274. Test — concurrent flush

Rechazar.

---

# 275. Test — request isolation

Request B no ve session A.

---

# 276. Test — tenant isolation

EntityKeys no colisionan.

---

# 277. Test — MariaDB capability

No tratarla como MySQL mediante version hack.

---

# 278. Test — unknown capability

No asumir support.

---

# 279. Anti-pattern: Entity::save() generates SQL

Incorrecto.

```text
Model API
→ EntityManager
→ UnitOfWork
→ Persistence Engine
```

---

# 280. Anti-pattern: UnitOfWork executes INSERT

Incorrecto.

---

# 281. Anti-pattern: Persistence Engine uses PDO

Prohibido.

---

# 282. Anti-pattern: Persistence Engine compiles SQL

Prohibido.

---

# 283. Anti-pattern: operation stores SQL string

Incorrecto:

```php
new InsertEntityOperation(
    sql: 'INSERT ...'
);
```

---

# 284. Anti-pattern: operation stores connection object

Persistence target será lógico.

---

# 285. Anti-pattern: graph reduced to array too early

Pierde dependencias.

---

# 286. Anti-pattern: cascade discovered during SQL execution

Demasiado tarde.

---

# 287. Anti-pattern: cascade recursion without guard

Puede explotar o ciclar infinitamente.

---

# 288. Anti-pattern: owning/inverse ambiguity ignored

Puede producir inconsistencias.

---

# 289. Anti-pattern: orphan = relationship removed

No siempre.

---

# 290. Anti-pattern: UPDATE every field

Debe utilizarse ChangeSet/policy.

---

# 291. Anti-pattern: generated ID assumed

Nunca fabricar:

```text
next ID
```

desde el ORM.

---

# 292. Anti-pattern: optimistic lock handled as ordinary field

El token tiene semántica concurrency-specific.

---

# 293. Anti-pattern: vendor conditionals

Incorrecto:

```php
if ($driver === 'pgsql') { ... }
```

Preferir capabilities downstream.

---

# 294. Anti-pattern: hidden transaction

Persistence Engine no abre transaction silenciosamente.

---

# 295. Anti-pattern: flush = commit

Incorrecto.

---

# 296. Anti-pattern: hidden retry

Prohibido.

---

# 297. Anti-pattern: failed request = no DB effect

Solo cierto antes de execution. Después, outcome puede ser UNKNOWN.

---

# 298. Anti-pattern: reconcile before confirmed result

Puede corromper IdentityMap/Snapshot.

---

# 299. Anti-pattern: static operation registry

Incompatible con persistent runtime.

---

# 300. Anti-pattern: nested flush

Default prohibido.

---

# 301. Anti-pattern: extension mutates UnitOfWork secretly

Debe usar structured contribution.

---

# 302. Anti-pattern: authorization in Persistence Engine

Separación incorrecta.

---

# 303. Anti-pattern: business validation in Persistence Engine

Separación incorrecta.

---

# 304. Anti-pattern: batch assumption

Múltiples operaciones compatibles no implican que deban ejecutarse como batch.

---

# 305. Architectural invariants

## DB-ORM-PERSISTENCE-001
Persistence Engine consumirá un UnitOfWork preparado.

## DB-ORM-PERSISTENCE-002
Persistence Engine será distinto de UnitOfWork.

## DB-ORM-PERSISTENCE-003
Persistence Engine será distinto de Change Tracking.

## DB-ORM-PERSISTENCE-004
Persistence Engine será distinto de Entity Snapshot System.

## DB-ORM-PERSISTENCE-005
Persistence Engine será distinto de Persistence Planner.

## DB-ORM-PERSISTENCE-006
Persistence Engine será distinto de Query Engine.

## DB-ORM-PERSISTENCE-007
Persistence Engine será distinto de SQL Compiler.

## DB-ORM-PERSISTENCE-008
Persistence Engine será distinto de Execution Engine.

## DB-ORM-PERSISTENCE-009
Persistence Engine será distinto de Transaction Manager.

## DB-ORM-PERSISTENCE-010
Persistence Engine no generará SQL.

## DB-ORM-PERSISTENCE-011
Persistence Engine no ejecutará SQL.

## DB-ORM-PERSISTENCE-012
Persistence Engine no utilizará PDO directamente.

## DB-ORM-PERSISTENCE-013
Persistence Engine no resolverá conexiones físicas.

## DB-ORM-PERSISTENCE-014
Persistence Engine no abrirá transacciones.

## DB-ORM-PERSISTENCE-015
Persistence Engine no realizará hidden retry.

## DB-ORM-PERSISTENCE-016
Persistence Engine no repetirá dirty checking completo.

## DB-ORM-PERSISTENCE-017
PreparedUnitOfWork representará una generación estable.

## DB-ORM-PERSISTENCE-018
Entity mutation posterior no se mezclará silenciosamente con generación preparada.

## DB-ORM-PERSISTENCE-019
PersistenceGeneration será distinta de transaction.

## DB-ORM-PERSISTENCE-020
PersistenceGeneration será distinta de batch.

## DB-ORM-PERSISTENCE-021
EntityChangeGraph será distinto de PersistenceOperationGraph.

## DB-ORM-PERSISTENCE-022
PersistenceOperationGraph será distinto de PersistencePlan.

## DB-ORM-PERSISTENCE-023
PersistenceOperation será distinta de Query Model.

## DB-ORM-PERSISTENCE-024
PersistenceOperation será distinta de SQL statement.

## DB-ORM-PERSISTENCE-025
PersistenceOperationId será distinto de EntityKey.

## DB-ORM-PERSISTENCE-026
PersistenceOperationId será distinto de QueryId.

## DB-ORM-PERSISTENCE-027
PersistenceOperationId será distinto de ExecutionId.

## DB-ORM-PERSISTENCE-028
Persistence intent será explícito.

## DB-ORM-PERSISTENCE-029
Persistence intent no implicará necesariamente un único SQL verb.

## DB-ORM-PERSISTENCE-030
INSERT utilizará únicamente valores mapped insertable.

## DB-ORM-PERSISTENCE-031
UPDATE utilizará cambios persistentes efectivos.

## DB-ORM-PERSISTENCE-032
Empty effective ChangeSet podrá producir NO_OPERATION.

## DB-ORM-PERSISTENCE-033
DELETE no implicará necesariamente hard delete cuando una extensión cambie semántica explícitamente.

## DB-ORM-PERSISTENCE-034
Identity predicates serán tipados.

## DB-ORM-PERSISTENCE-035
Composite IDs no se concatenarán como identidad interna.

## DB-ORM-PERSISTENCE-036
Optimistic lock predicate será explícito.

## DB-ORM-PERSISTENCE-037
Relationship operations serán first-class.

## DB-ORM-PERSISTENCE-038
ORM relationship será distinta de foreign key.

## DB-ORM-PERSISTENCE-039
Collection operations serán first-class.

## DB-ORM-PERSISTENCE-040
Owning side determinará autoridad de persistencia.

## DB-ORM-PERSISTENCE-041
Inverse side no generará automáticamente mutation física incorrecta.

## DB-ORM-PERSISTENCE-042
Relationship consistency policy será explícita.

## DB-ORM-PERSISTENCE-043
ORM cascade será distinto de database cascade.

## DB-ORM-PERSISTENCE-044
Cascade discovery deberá estabilizarse antes de execution.

## DB-ORM-PERSISTENCE-045
Late cascade no mutará silenciosamente PreparedUnitOfWork.

## DB-ORM-PERSISTENCE-046
Orphan removal será distinto de relationship dissociation.

## DB-ORM-PERSISTENCE-047
Orphan candidate será validado antes de DELETE.

## DB-ORM-PERSISTENCE-048
Graph edges representarán semantic dependencies.

## DB-ORM-PERSISTENCE-049
Graph dependency será distinta de final execution order.

## DB-ORM-PERSISTENCE-050
Persistence Planner poseerá physical ordering.

## DB-ORM-PERSISTENCE-051
Cycles serán first-class.

## DB-ORM-PERSISTENCE-052
Cycle no implicará automáticamente error.

## DB-ORM-PERSISTENCE-053
Cycle resolution no será escondida.

## DB-ORM-PERSISTENCE-054
Persistence requirements serán explícitos.

## DB-ORM-PERSISTENCE-055
Requirement será distinto de capability.

## DB-ORM-PERSISTENCE-056
Platform capabilities determinarán estrategias downstream.

## DB-ORM-PERSISTENCE-057
Persistence Engine no utilizará vendor conditionals como arquitectura.

## DB-ORM-PERSISTENCE-058
MariaDB será first-class.

## DB-ORM-PERSISTENCE-059
Version será distinta de capability.

## DB-ORM-PERSISTENCE-060
Write intent no resolverá conexión física.

## DB-ORM-PERSISTENCE-061
PersistenceTarget será lógico.

## DB-ORM-PERSISTENCE-062
PersistenceTarget será distinto de Connection.

## DB-ORM-PERSISTENCE-063
Core no hardcodeará tenant_id.

## DB-ORM-PERSISTENCE-064
Entity identity namespace preservará tenant/database isolation.

## DB-ORM-PERSISTENCE-065
Cross-target persistence será explícita.

## DB-ORM-PERSISTENCE-066
No existirá hidden distributed transaction.

## DB-ORM-PERSISTENCE-067
Default V1 rechazará unsupported cross-target atomic scope.

## DB-ORM-PERSISTENCE-068
Operation normalization será determinista.

## DB-ORM-PERSISTENCE-069
Normalization será distinta de optimization.

## DB-ORM-PERSISTENCE-070
No-op operations podrán eliminarse cuando sea semánticamente seguro.

## DB-ORM-PERSISTENCE-071
Lifecycle-significant effects no se eliminarán como no-op sin contrato.

## DB-ORM-PERSISTENCE-072
Persistence operation validation ocurrirá antes de planning.

## DB-ORM-PERSISTENCE-073
Validation estructural será distinta de DB constraint validation.

## DB-ORM-PERSISTENCE-074
Immutable identifier mutation será rechazada por default.

## DB-ORM-PERSISTENCE-075
Cross-context managed entities no se mezclarán silenciosamente.

## DB-ORM-PERSISTENCE-076
Detached entity no será attached automáticamente.

## DB-ORM-PERSISTENCE-077
Known detached identity podrá convertirse en referencia cuando mapping lo permita.

## DB-ORM-PERSISTENCE-078
Transient relationship target sin cascade válido será error.

## DB-ORM-PERSISTENCE-079
PersistenceEngineResult será distinto de execution result.

## DB-ORM-PERSISTENCE-080
Derivation exitosa no significará database mutation.

## DB-ORM-PERSISTENCE-081
Persistence fingerprint será determinista.

## DB-ORM-PERSISTENCE-082
Persistence fingerprint será distinto de ChangeSet fingerprint.

## DB-ORM-PERSISTENCE-083
Persistence fingerprint será distinto de Query fingerprint.

## DB-ORM-PERSISTENCE-084
Mapping drift será detectable.

## DB-ORM-PERSISTENCE-085
Stale UnitOfWork generation será detectable.

## DB-ORM-PERSISTENCE-086
Stale generations no se mezclarán.

## DB-ORM-PERSISTENCE-087
Persistence Engine será stateless o immutable-service oriented.

## DB-ORM-PERSISTENCE-088
Mutable derivation state vivirá en scoped session.

## DB-ORM-PERSISTENCE-089
Derivation session no sobrevivirá al scope.

## DB-ORM-PERSISTENCE-090
Persistence operations no contendrán Query Models prematuramente.

## DB-ORM-PERSISTENCE-091
Query Model generation ocurrirá después de persistence planning.

## DB-ORM-PERSISTENCE-092
Query Engine conservará autoridad sobre Query Model semantics.

## DB-ORM-PERSISTENCE-093
Generated identity dependencies serán explícitas.

## DB-ORM-PERSISTENCE-094
DeferredPersistentValue será distinto de hidden lazy query.

## DB-ORM-PERSISTENCE-095
Generated IDs no serán fabricados por ORM.

## DB-ORM-PERSISTENCE-096
Persistence reconciliation será outcome-aware.

## DB-ORM-PERSISTENCE-097
Reconciliation será distinta de operation derivation.

## DB-ORM-PERSISTENCE-098
Confirmed INSERT podrá establecer persistent identity.

## DB-ORM-PERSISTENCE-099
Confirmed UPDATE podrá avanzar snapshot.

## DB-ORM-PERSISTENCE-100
Confirmed DELETE podrá remover managed baseline según state policy.

## DB-ORM-PERSISTENCE-101
UNKNOWN outcome será first-class.

## DB-ORM-PERSISTENCE-102
UNKNOWN outcome no se interpretará como no-effect failure.

## DB-ORM-PERSISTENCE-103
UNKNOWN outcome no se interpretará como success.

## DB-ORM-PERSISTENCE-104
PARTIALLY_CONFIRMED será first-class.

## DB-ORM-PERSISTENCE-105
Unknown persistence podrá taint EntityManager.

## DB-ORM-PERSISTENCE-106
Retryability será distinta de replayability.

## DB-ORM-PERSISTENCE-107
Persistence Engine no decidirá retries.

## DB-ORM-PERSISTENCE-108
Client-generated identifier no garantizará universalmente safe replay.

## DB-ORM-PERSISTENCE-109
Transaction requirement podrá declararse sin abrir transaction.

## DB-ORM-PERSISTENCE-110
One flush será distinto de one transaction.

## DB-ORM-PERSISTENCE-111
Existing TransactionContext será respetado.

## DB-ORM-PERSISTENCE-112
Transaction-local snapshot semantics serán preservadas.

## DB-ORM-PERSISTENCE-113
Batch eligibility será distinta de batch plan.

## DB-ORM-PERSISTENCE-114
Batch persistence no será asumida automáticamente.

## DB-ORM-PERSISTENCE-115
Lifecycle requirements serán preservados.

## DB-ORM-PERSISTENCE-116
Lifecycle callback será distinto de PersistenceOperation.

## DB-ORM-PERSISTENCE-117
postPersist será distinto de transaction committed.

## DB-ORM-PERSISTENCE-118
Domain event será distinto de lifecycle callback.

## DB-ORM-PERSISTENCE-119
Business validation no será responsabilidad del Persistence Engine.

## DB-ORM-PERSISTENCE-120
Authorization no será responsabilidad del Persistence Engine.

## DB-ORM-PERSISTENCE-121
Mass assignment no será responsabilidad del Persistence Engine.

## DB-ORM-PERSISTENCE-122
Raw SQL strings no serán persistent values.

## DB-ORM-PERSISTENCE-123
Raw expressions utilizarán escape hatch explícito.

## DB-ORM-PERSISTENCE-124
Extensions contribuirán operaciones estructuradas.

## DB-ORM-PERSISTENCE-125
Extensions no ejecutarán SQL.

## DB-ORM-PERSISTENCE-126
Extensions no mutarán UnitOfWork ocultamente.

## DB-ORM-PERSISTENCE-127
Extension ordering será explícito.

## DB-ORM-PERSISTENCE-128
Extension collisions no usarán last-wins.

## DB-ORM-PERSISTENCE-129
Extension registry será frozen en runtime normal.

## DB-ORM-PERSISTENCE-130
Derivation será determinista para mismos inputs.

## DB-ORM-PERSISTENCE-131
Derivation no realizará hidden DB I/O.

## DB-ORM-PERSISTENCE-132
Mutable persistence state será request/operation-scoped.

## DB-ORM-PERSISTENCE-133
FrankenPHP requests no compartirán derivation state.

## DB-ORM-PERSISTENCE-134
RoadRunner requests no compartirán derivation state.

## DB-ORM-PERSISTENCE-135
OpenSwoole logical scopes no compartirán derivation state accidentalmente.

## DB-ORM-PERSISTENCE-136
Process-global pending operation registry estará prohibido.

## DB-ORM-PERSISTENCE-137
Worker reset no ejecutará implicit flush.

## DB-ORM-PERSISTENCE-138
Same EntityManager no soportará concurrent flushes en V1.

## DB-ORM-PERSISTENCE-139
Nested flush será rechazado por default.

## DB-ORM-PERSISTENCE-140
Flush reentrancy será detectable.

## DB-ORM-PERSISTENCE-141
Resource limits serán explícitos.

## DB-ORM-PERSISTENCE-142
Resource exhaustion no truncará operations silenciosamente.

## DB-ORM-PERSISTENCE-143
Cascade explosion será observable.

## DB-ORM-PERSISTENCE-144
Graph traversal protegerá contra recursion patológica.

## DB-ORM-PERSISTENCE-145
Temporary derivation structures deberán liberarse.

## DB-ORM-PERSISTENCE-146
Telemetry no cambiará semantics.

## DB-ORM-PERSISTENCE-147
Diagnostics preservarán operation provenance.

## DB-ORM-PERSISTENCE-148
Operation provenance será distinta de business audit.

## DB-ORM-PERSISTENCE-149
Derivation failure será distinta de execution failure.

## DB-ORM-PERSISTENCE-150
Persistence Engine derivation failure no deberá haber producido DB side effects.

## DB-ORM-PERSISTENCE-151
EntityChangeGraph conservará relaciones semánticas entre cambios.

## DB-ORM-PERSISTENCE-152
PersistenceOperationGraph conservará dependencias entre operaciones.

## DB-ORM-PERSISTENCE-153
Graph structure no se reducirá prematuramente a una lista ordenada.

## DB-ORM-PERSISTENCE-154
Generated-value barriers serán preservadas para Planner.

## DB-ORM-PERSISTENCE-155
Operation requirements serán inspeccionables.

## DB-ORM-PERSISTENCE-156
Persistence Engine será testeable sin DB activa.

## DB-ORM-PERSISTENCE-157
Persistence Engine será testeable sin SQL compiler.

## DB-ORM-PERSISTENCE-158
Persistence Engine será testeable sin driver.

## DB-ORM-PERSISTENCE-159
Persistence Engine no duplicará Query Engine.

## DB-ORM-PERSISTENCE-160
Persistence Engine transformará estado ORM preparado en un grafo tipado, determinista, validado y reconciliable de operaciones de persistencia.

---

# 306. Fórmula fundamental

```text
PersistenceEngine
=
PreparedUnitOfWork
→
PersistenceOperationGraph
```

---

# 307. Entity transformation

```text
EntityPersistenceIntent(E)
=
f(
    EntityState(E),
    ChangeSet(E),
    Snapshot(E),
    EntityMetadata(E),
    RelationshipChanges(E)
)
```

---

# 308. Persistence operation derivation

```text
PersistenceOperations
=
Normalize(
    Validate(
        Derive(
            EntityChangeGraph(
                PreparedUnitOfWork
            )
        )
    )
)
```

---

# 309. Insert formula

```text
InsertOperation(E)
=
InsertableMappedValues(E)
+
GeneratedValueRequirements(E)
+
RelationshipDependencies(E)
+
PersistenceRequirements(E)
```

---

# 310. Update formula

```text
UpdateOperation(E)
=
EffectivePersistentChangeSet(E)
+
IdentityPredicate(E)
+
ConcurrencyPredicate(E)
+
GeneratedValueRequirements(E)
```

---

# 311. Delete formula

```text
DeleteOperation(E)
=
EntityIdentity(E)
+
ConcurrencyRequirements(E)
+
RelationshipDependencies(E)
+
DeletionStrategy(E)
```

---

# 312. Graph formula

```text
PersistenceOperationGraph
=
Operations
+
SemanticDependencies
+
GeneratedValueDependencies
+
RelationshipDependencies
+
ConflictEdges
```

---

# 313. Generated identity formula

```text
ChildNeedsParentId
∧
ParentIdGeneratedByPersistence
⇒
ParentInsert
    → GeneratedValueBarrier
    → ChildPersistence
```

---

# 314. Stable generation formula

```text
SafeDerivation
=
PreparedUnitOfWorkGeneration
=
ExpectedPersistenceGeneration
```

Si no:

```text
REPREPARE
```

---

# 315. Reconciliation formula

```text
PersistenceReconciliation
=
OldManagedState
+
PersistencePlan
+
ExecutionOutcome
+
GeneratedValues
+
TransactionState
→
NewManagedState
```

---

# 316. Certainty formula

```text
AdvanceManagedBaseline
⇔
RequiredPersistenceEffectsConfirmed
∧
RequiredGeneratedValuesResolved
∧
TransactionSemanticsSatisfied
```

---

# 317. Unknown outcome formula

```text
UnknownOutcome
⇒
¬ AssumeSuccess
∧
¬ AssumeNoEffect
∧
PreserveUncertainty
```

---

# 318. Safe persistence derivation

```text
SafePersistenceDerivation
=
StableUnitOfWork
∧
ValidEntityStates
∧
ValidMappings
∧
ValidIdentities
∧
ConsistentRelationshipSemantics
∧
DeterministicGraph
∧
ExplicitRequirements
∧
NoHiddenIO
∧
NoHiddenExecution
```

---

# 319. Persistent runtime formula

```text
SafePersistenceRuntime
=
ImmutableSharedEngine
∧
FrozenSharedExtensions
∧
ScopedDerivationSession
∧
ScopedUnitOfWork
∧
ScopedEntityManager
∧
NoConcurrentFlushPerManager
∧
NoNestedFlushByDefault
∧
DeterministicCleanup
```

---

# 320. Architectural pipeline

```text
Entity Mutations
      ↓
Change Tracking
      ↓
ChangeSets
      ↓
UnitOfWork
      ↓
PreparedUnitOfWork
      ↓
Entity Change Graph
      ↓
Persistence Engine
      ↓
Persistence Operation Graph
      ↓
Persistence Planner
      ↓
Persistence Plan
      ↓
Query Model Generation
      ↓
Query Engine
      ↓
SQL Compiler
      ↓
Execution Engine
      ↓
Database
      ↓
Execution Results
      ↓
Persistence Reconciliation
      ↓
Entity State + Snapshots + IdentityMap
```

---

# 321. Master Formula

```text
Database Persistence Engine
=
Prepared UnitOfWork Consumption
+
Stable Persistence Generations
+
Entity Change Graph
+
Persistence Intent Resolution
+
Insert Operation Derivation
+
Update Operation Derivation
+
Delete Operation Derivation
+
Relationship Operations
+
Collection Operations
+
Cascade Integration
+
Orphan Analysis
+
Generated Value Dependencies
+
Optimistic Lock Requirements
+
Persistence Predicates
+
Persistence Requirements
+
Logical Persistence Targets
+
Operation Graph Construction
+
Cycle Preservation
+
Operation Normalization
+
Operation Validation
+
Deterministic Fingerprinting
+
Persistence Provenance
+
Outcome Reconciliation Contracts
+
Transaction Awareness
+
Batch Eligibility Metadata
+
Lifecycle Integration
+
Capability Awareness
+
Extension Governance
+
Runtime Isolation
+
Resource Governance
+
Telemetry
+
Diagnostics
+
Failure Modeling
```

---

# 322. Master Rule

> **El Persistence Engine de VoltStack transforma una generación estable del UnitOfWork en un grafo tipado de operaciones ORM que describe qué debe persistirse y qué dependencias existen entre esas operaciones. No determina todavía la estrategia física final, no genera SQL, no ejecuta consultas, no abre transacciones y no supone el resultado de la persistencia.**

---

# 323. Resultado arquitectónico

Con los documentos:

```text
123 Identity Map
124 Unit of Work Architecture
125 Change Tracking
126 Entity Snapshot
127 Persistence Engine
```

VoltStack ya dispone del núcleo:

```text
               Managed Entity
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     Identity Map          Entity State
          │                     │
          └──────────┬──────────┘
                     ▼
                UnitOfWork
                     │
             ┌───────┴───────┐
             ▼               ▼
         Snapshot         ChangeSet
             │               │
             └───────┬───────┘
                     ▼
             PreparedUnitOfWork
                     │
                     ▼
             Persistence Engine
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
 Entity Change Graph    Persistence Operations
          │                     │
          └──────────┬──────────┘
                     ▼
          PersistenceOperationGraph
```

Sin haber introducido todavía:

```text
SQL
driver execution
physical connection logic
```

---

# 324. Próxima frontera

Ahora debe resolverse la pregunta:

> Tenemos un grafo válido de operaciones de persistencia. ¿Cómo lo convertimos en un plan determinista y ejecutable respetando dependencias, ciclos, IDs generados, relaciones, locking, batching, capabilities y transaction boundaries?

Esa responsabilidad pertenece al:

```text
Persistence Planner
```

---

# 325. Siguiente documento

```text
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
```

Este documento deberá formalizar:

```text
Persistence Planner architecture

Persistence Engine Result
        ↓
Persistence Operation Graph
        ↓
Dependency Analysis
        ↓
Cycle Analysis
        ↓
Strategy Resolution
        ↓
Operation Ordering
        ↓
Generated Value Barriers
        ↓
Transaction Boundaries
        ↓
Batch Opportunities
        ↓
Query Generation Steps
        ↓
Reconciliation Boundaries
        ↓
Persistence Plan
```

Incluyendo:

```text
PersistencePlanner
PersistencePlan
PersistencePlanId
PersistencePlanFingerprint
PersistencePlanStep
PersistencePlanPhase
PersistencePlanGraph
PersistenceDependencyResolver
Topological ordering
Cycle resolution
Deferred identifier resolution
Generated value barriers
Relationship ordering
Insert ordering
Update ordering
Delete ordering
Collection operation ordering
Constraint-aware planning
Optimistic locking planning
Transaction requirements
Savepoint considerations
Batch eligibility
Batch grouping boundaries
Query Model generation boundary
Execution barriers
Lifecycle barriers
Reconciliation barriers
Platform capability strategy
Target validation
Plan determinism
Stale plan detection
Plan explainability
Plan validation
Plan resource budgets
Persistent runtime isolation
Extension planning
Telemetry
Diagnostics
Testing
Failure model
```

con la regla central:

> **El Persistence Planner decide cómo transformar un grafo válido de operaciones ORM en un plan de persistencia ordenado y ejecutable; planifica dependencias y estrategias, pero no ejecuta operaciones, no genera SQL y no altera el UnitOfWork.**