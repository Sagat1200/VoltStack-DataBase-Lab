# 129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md

# VoltStack Quantum Database
## Database Insert Persistence System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 129 — Database Insert Persistence System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Insert Persistence System` define la arquitectura responsable de transformar una entidad ORM en estado `NEW` en una operación de inserción semánticamente válida, ejecutarla a través del pipeline canónico de VoltStack y reconciliar posteriormente identidad, valores generados, `IdentityMap`, snapshots y estado del `UnitOfWork`.

El sistema debe resolver correctamente:

```text
NEW Entity
    ↓
Insert Analysis
    ↓
Identifier Strategy
    ↓
Insertable State
    ↓
Relationship Dependencies
    ↓
InsertEntityOperation
    ↓
Persistence Planner
    ↓
Insert Persistence Step
    ↓
Query Model
    ↓
Compiler / Executor
    ↓
Execution Outcome
    ↓
Generated Values
    ↓
Identity Reconciliation
    ↓
IdentityMap
    ↓
Snapshot
    ↓
MANAGED
```

Principio central:

> **Una entidad `NEW` no se convierte en `MANAGED` simplemente porque exista una intención de `INSERT`; VoltStack solo establecerá su baseline persistente cuando la inserción alcance un resultado suficientemente cierto y toda identidad o valor generado necesario haya sido reconciliado correctamente.**

---

# 2. Responsabilidad

El sistema responde principalmente:

> ¿Cómo convertir de forma segura el estado persistente inicial de una entidad `NEW` en una inserción ORM ejecutable y reconciliable?

No responde:

```text
¿Cómo generar SQL?
¿Cómo abrir una transacción?
¿Cómo compilar un INSERT?
¿Cómo ejecutar PDO?
¿Cómo detectar todos los cambios del UnitOfWork?
¿Cómo hacer commit?
```

Estas responsabilidades pertenecen a otros subsistemas.

---

# 3. Posición arquitectónica

```text
EntityManager::persist($entity)
          ↓
      UnitOfWork
          ↓
      State = NEW
          ↓
   Change Tracking
          ↓
PreparedUnitOfWork
          ↓
Persistence Engine
          ↓
InsertEntityOperation
          ↓
Persistence Planner
          ↓
InsertPersistenceStep
          ↓
Insert Persistence System
          ↓
Persistence Query Translator
          ↓
Insert Query Model
          ↓
Semantic Query Engine
          ↓
Optimizer / Query Planner
          ↓
SQL Compiler
          ↓
Execution Engine
          ↓
Database
          ↓
Execution Outcome
          ↓
Insert Reconciliation
          ↓
IdentityMap / Snapshot / UoW
```

---

# 4. Separaciones fundamentales

```text
Insert Persistence
≠
SQL INSERT

Insert Persistence
≠
Query Builder

Insert Persistence
≠
SQL Compiler

Insert Persistence
≠
Statement Execution

Insert Persistence
≠
Transaction Commit

Insert Persistence
≠
EntityManager::persist()

Insert Persistence
≠
Entity Constructor
```

---

# 5. `persist()` ≠ `INSERT`

Una de las reglas fundamentales del ORM:

```php
$entityManager->persist($user);
```

no significa:

```sql
INSERT INTO users ...
```

Conceptualmente:

```text
persist(entity)
    ↓
register persistence intent
    ↓
EntityState = NEW
```

La inserción real ocurre posteriormente durante:

```text
flush()
```

---

# 6. NEW ≠ inserted

```text
NEW
```

significa que la entidad participa en el `PersistenceContext` como candidata a creación persistente.

No significa:

```text
row exists
```

---

# 7. Identifier assigned ≠ inserted

Por ejemplo:

```php
$user = new User(
    id: UserId::fromString($uuid),
);
```

puede tener identidad asignada antes del `INSERT`.

Aun así:

```text
IdentifierAssigned
≠
DatabaseRowExists
```

---

# 8. Arquitectura interna

```text
InsertEntityOperation
        │
        ▼
Insert Metadata Resolution
        │
        ▼
Identifier Strategy Resolution
        │
        ▼
Insertable Property Resolution
        │
        ▼
Value Classification
        │
        ▼
Relationship FK Resolution
        │
        ▼
Generated Value Requirement Analysis
        │
        ▼
Insert Requirement Validation
        │
        ▼
Insert Query Translation
        │
        ▼
Query Model
        │
        ▼
Execution
        │
        ▼
Insert Outcome Classification
        │
        ▼
Generated Value Reconciliation
        │
        ▼
Identity Reconciliation
        │
        ▼
IdentityMap Registration
        │
        ▼
Snapshot Establishment
        │
        ▼
UoW State Reconciliation
```

---

# 9. Contrato principal

```php
interface InsertPersistenceSystem
{
    public function prepare(
        InsertPersistenceRequest $request,
    ): PreparedInsertPersistence;

    public function reconcile(
        PreparedInsertPersistence $insert,
        InsertExecutionOutcome $outcome,
    ): InsertReconciliationResult;
}
```

---

# 10. Preparación y reconciliación

El sistema separa dos momentos:

```text
prepare()
```

y:

```text
reconcile()
```

porque entre ambos ocurre ejecución externa.

---

# 11. InsertPersistenceRequest

```php
final readonly class InsertPersistenceRequest
{
    public function __construct(
        public InsertEntityOperation $operation,
        public EntityMetadata $metadata,
        public PersistencePlanningContext $planningContext,
        public PersistenceExecutionContext $executionContext,
    ) {}
}
```

---

# 12. PreparedInsertPersistence

```php
final readonly class PreparedInsertPersistence
{
    public function __construct(
        public InsertPersistenceId $id,
        public PersistenceOperationId $operationId,
        public EntityKeyCandidate $identity,
        public InsertValueSet $values,
        public GeneratedValueRequirementSet $generatedValues,
        public InsertDependencySet $dependencies,
        public InsertExecutionRequirementSet $requirements,
        public InsertPersistenceFingerprint $fingerprint,
    ) {}
}
```

---

# 13. Prepared insert ≠ executed insert

`PreparedInsertPersistence` solo representa:

```text
validated insert intent
```

No prueba que la DB haya sido modificada.

---

# 14. Insert metadata

La preparación consume `EntityMetadata` compilado.

Ejemplo:

```text
User
├── table: users
├── id
│   └── strategy: UUID_ASSIGNED
├── name
├── email
├── status
├── createdAt
└── version
```

---

# 15. No reflection en hot path

El sistema no deberá descubrir mappings mediante Reflection durante cada `flush()`.

Debe utilizar:

```text
Compiled EntityMetadata
```

---

# 16. Insertable properties

No toda propiedad persistente tiene que participar en `INSERT`.

Se requiere:

```php
enum PropertyInsertability
{
    case INSERTABLE;
    case DATABASE_GENERATED;
    case DATABASE_DEFAULT;
    case COMPUTED;
    case READ_ONLY;
    case RELATIONSHIP_DERIVED;
    case NEVER;
}
```

---

# 17. Persistent ≠ insertable

```text
PersistentProperty
⇏
InsertableProperty
```

---

# 18. Example

Una columna:

```sql
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
```

puede mapearse como:

```text
persistent
database-generated/default
not explicitly inserted
```

---

# 19. InsertValueSet

```php
final readonly class InsertValueSet
{
    public function __construct(
        public array $values,
    ) {}
}
```

Cada valor deberá conservar semántica, no solo un valor PHP bruto.

---

# 20. InsertValue

```php
final readonly class InsertValue
{
    public function __construct(
        public PersistentPropertyId $property,
        public InsertValueKind $kind,
        public mixed $value,
        public DatabaseType $type,
    ) {}
}
```

---

# 21. Value kinds

```php
enum InsertValueKind
{
    case EXPLICIT_VALUE;
    case EXPLICIT_NULL;
    case OMITTED;
    case DATABASE_DEFAULT;
    case GENERATED;
    case RELATIONSHIP_IDENTIFIER;
}
```

---

# 22. Omitted ≠ NULL

Regla crítica:

```text
OMITTED
≠
EXPLICIT_NULL
```

---

# 23. Ejemplo

Supóngase:

```sql
status VARCHAR(20) DEFAULT 'pending'
```

Esto:

```text
OMITTED
```

permite usar:

```text
pending
```

mientras:

```text
EXPLICIT_NULL
```

puede:

```text
insert NULL
```

o violar `NOT NULL`.

---

# 24. PHP null ≠ always SQL NULL

El mapping y el estado de propiedad deberán distinguir:

```text
unset/uninitialized
explicit null
database default
generated value
```

cuando la semántica lo requiera.

---

# 25. Uninitialized properties

Las propiedades tipadas PHP no inicializadas no deberán convertirse automáticamente en:

```text
NULL
```

---

# 26. Uninitialized resolution

Podrán significar:

```text
OMITTED
DATABASE_DEFAULT
INVALID
```

dependiendo del metadata.

---

# 27. Required property

Si una propiedad:

```text
required
insertable
no default
not generated
```

no tiene valor:

```text
InsertPersistenceValidationException
```

antes de ejecutar SQL.

---

# 28. Identifier architecture

El sistema debe soportar diferentes estrategias.

```php
enum EntityIdentifierGenerationStrategy
{
    case ASSIGNED;
    case UUID;
    case ULID;
    case DATABASE_IDENTITY;
    case DATABASE_SEQUENCE;
    case DATABASE_GENERATED;
    case COMPOSITE;
    case CUSTOM;
}
```

---

# 29. ASSIGNED

Ejemplo:

```php
$user = new User(
    id: new UserId(42),
);
```

El ID existe antes del insert.

---

# 30. UUID

VoltStack puede generar:

```text
UUID
```

antes del insert.

---

# 31. ULID

Igualmente:

```text
ULID
```

puede ser application-generated.

---

# 32. Database identity

Ejemplo conceptual:

```text
AUTO_INCREMENT
IDENTITY
```

El identificador aparece como consecuencia de la inserción.

---

# 33. Sequence

Una plataforma puede permitir:

```text
sequence preallocation
```

antes del insert.

Esto permite conocer identidad previamente.

---

# 34. Sequence strategy ≠ PostgreSQL hardcode

El ORM deberá consumir capability/strategy abstracta.

No:

```php
if ($database === 'pgsql') {
}
```

---

# 35. Composite identifiers

Ejemplo:

```text
OrderLine
├── order_id
└── line_number
```

Ambos pueden constituir:

```text
CanonicalIdentifier
```

---

# 36. Composite completeness

No podrá establecerse una identidad persistente completa si falta una parte requerida.

---

# 37. Partial composite identifier

```text
(order_id = 10, line_number = unknown)
```

no deberá tratarse como `EntityKey` final si ambas partes forman la identidad.

---

# 38. Identifier validation

Antes del insert:

```text
strategy valid
type valid
mapping valid
assignment state valid
namespace valid
composite completeness valid
```

---

# 39. No fake identifiers

Para DB-generated IDs, VoltStack no deberá usar:

```text
-1
-2
-3
```

como identidades persistentes ficticias.

---

# 40. Temporary object identity

Antes del ID persistente, el UoW puede utilizar:

```text
ObjectToken
```

basado en identidad de objeto interna.

Pero:

```text
ObjectToken
≠
EntityKey
```

---

# 41. Generated value architecture

No solo IDs pueden ser generados.

Ejemplos:

```text
id
created_at
updated_at
version
computed column
database token
default status
```

---

# 42. GeneratedValueRequirement

```php
final readonly class GeneratedValueRequirement
{
    public function __construct(
        public PersistentPropertyId $property,
        public GeneratedValueKind $kind,
        public GeneratedValueNecessity $necessity,
    ) {}
}
```

---

# 43. GeneratedValueKind

```php
enum GeneratedValueKind
{
    case IDENTIFIER;
    case DEFAULT_VALUE;
    case VERSION;
    case TIMESTAMP;
    case COMPUTED;
    case DATABASE_EXPRESSION;
    case CUSTOM;
}
```

---

# 44. GeneratedValueNecessity

```php
enum GeneratedValueNecessity
{
    case REQUIRED_FOR_IDENTITY;
    case REQUIRED_FOR_DEPENDENCY;
    case REQUIRED_FOR_SNAPSHOT;
    case REQUIRED_FOR_DOMAIN_STATE;
    case OPTIONAL_REFRESH;
}
```

---

# 45. Required generated value

Si un valor es:

```text
REQUIRED_FOR_IDENTITY
```

el insert no puede reconciliarse completamente sin él.

---

# 46. Generated value retrieval

Posibles estrategias conceptuales:

```text
INSERT RETURNING
generated-key API
sequence preallocation
follow-up select
driver generated-id facility
platform-specific capability adapter
```

---

# 47. Strategy selection

La estrategia se decide mediante:

```text
Capabilities
+
Metadata
+
Persistence Requirements
+
Transaction Context
```

---

# 48. SQL representation remains elsewhere

Aunque la estrategia sea:

```text
INSERT_RETURNING
```

el Insert Persistence System no produce:

```sql
RETURNING id
```

Eso corresponde al compiler.

---

# 49. InsertGeneratedValueStrategy

```php
enum InsertGeneratedValueStrategy
{
    case PREALLOCATED;
    case INSERT_RETURNING;
    case DRIVER_GENERATED_KEY;
    case FOLLOW_UP_QUERY;
    case NO_RETRIEVAL_REQUIRED;
    case EXTENSION_DEFINED;
}
```

---

# 50. Capability-aware strategy

Ejemplo:

```php
$strategy = $resolver->resolve(
    $metadata,
    $capabilities,
    $requirements,
);
```

---

# 51. UNKNOWN capability

Si una estrategia requerida tiene:

```text
UNKNOWN
```

no se asumirá soporte.

---

# 52. Follow-up query

Una estrategia:

```text
INSERT
↓
SELECT generated value
```

debe analizar:

```text
transaction safety
identity correlation
concurrent inserts
connection affinity
```

---

# 53. Unsafe follow-up

No deberá usarse una consulta ambigua como:

```text
SELECT MAX(id)
```

para descubrir el ID insertado.

---

# 54. Generated-key correlation

VoltStack debe poder demostrar que:

```text
GeneratedValue
```

corresponde al insert específico.

---

# 55. Relationship foreign keys

Una relación:

```text
Order → Customer
```

puede requerir:

```text
customer_id
```

como valor insertable.

---

# 56. Object reference ≠ DB value

El ORM no insertará:

```php
$customerObject
```

como parámetro.

Debe resolver:

```text
Customer Entity
↓
Canonical Identifier
↓
Relationship DB Value
```

---

# 57. Relationship identifier dependency

Si Customer todavía no tiene ID generado:

```text
Insert Customer
↓
Generated ID
↓
Insert Order
```

El Planner introduce la dependencia.

---

# 58. Insert system respects planner

El Insert Persistence System no reordena arbitrariamente operaciones.

Consume el plan ya establecido.

---

# 59. Relationship existence

Incluso con ID conocido:

```text
Customer.id = UUID
```

puede existir:

```text
REQUIRES_EXISTENCE
```

por FK inmediata.

---

# 60. Cascade persist

Supóngase:

```php
$order->setCustomer($newCustomer);
$entityManager->persist($order);
```

con `cascade persist`.

El UoW descubre:

```text
Customer NEW
Order NEW
```

y Persistence Engine produce operaciones separadas.

---

# 61. Cascade ≠ recursive insert

No:

```text
Insert Order System
→ recursively insert Customer
```

Debe ser:

```text
UoW graph
→ Persistence Engine
→ Planner
```

---

# 62. Foreign-key nullability

Si la relación es nullable:

```text
NULL
```

puede ser semánticamente válida.

Pero nullable no implica que VoltStack deba romper ciclos usando NULL automáticamente.

---

# 63. Deferred association

Una estrategia explícita puede producir:

```text
Insert A without relationship
↓
Insert B
↓
Update A relationship
```

si mapping/schema/policy lo permiten.

---

# 64. Insert query translation

Una vez preparado el step:

```text
InsertPersistenceStep
↓
InsertPersistenceQueryTranslator
↓
InsertQueryModel
```

---

# 65. Translator

```php
interface InsertPersistenceQueryTranslator
{
    public function translate(
        PreparedInsertPersistence $insert,
        InsertPersistenceTranslationContext $context,
    ): InsertQueryModel;
}
```

---

# 66. Query Model

Ejemplo conceptual:

```text
InsertQueryModel
├── target
├── columns
├── parameters
├── generated-value requirements
└── execution metadata
```

---

# 67. ORM field ≠ column string

El translator resuelve mediante metadata.

---

# 68. Parameterization

Todos los valores ordinarios deberán pasar por:

```text
Database Type System
↓
Parameter Binding
```

No concatenación SQL.

---

# 69. Type conversion

```text
PHP/domain value
↓
ORM mapping
↓
Database type conversion
↓
Query parameter
```

---

# 70. Value Object

Ejemplo:

```php
EmailAddress
```

puede convertirse a:

```text
VARCHAR
```

mediante el Type System.

---

# 71. Enum

Igualmente:

```text
Status::ACTIVE
```

puede mapearse a:

```text
active
```

sin que la entidad conozca SQL.

---

# 72. InsertExecutionRequirementSet

```php
final readonly class InsertExecutionRequirementSet
{
    public function __construct(
        public bool $requiresGeneratedKeys,
        public bool $requiresReturning,
        public bool $requiresSameConnection,
        public bool $requiresTransaction,
        public bool $requiresAffectedRowCount,
    ) {}
}
```

---

# 73. Same-connection requirement

Algunas generated-value strategies pueden exigir:

```text
INSERT
and
generated key retrieval
```

sobre la misma conexión física/lógica.

Debe declararse.

---

# 74. Connection routing

El ORM no selecciona directamente una replica.

Una inserción declara:

```text
WRITE intent
```

y el Connection Routing System elige el target apropiado.

---

# 75. INSERT never silently routed to replica

Salvo arquitectura explícita futura con writable replicas.

---

# 76. Execution

El Execution Engine recibe el Query Model compilado.

---

# 77. InsertExecutionOutcome

```php
final readonly class InsertExecutionOutcome
{
    public function __construct(
        public PersistenceExecutionCertainty $certainty,
        public InsertExecutionStatus $status,
        public ?int $affectedRows,
        public GeneratedValueResultSet $generatedValues,
        public ExecutionDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 78. Certainty

```php
enum PersistenceExecutionCertainty
{
    case CERTAIN;
    case UNKNOWN;
}
```

---

# 79. Status

```php
enum InsertExecutionStatus
{
    case SUCCEEDED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 80. UNKNOWN is first-class

Ejemplo:

```text
statement sent
↓
connection lost
↓
server outcome unknown
```

No sabemos si:

```text
row inserted
```

o no.

---

# 81. UNKNOWN ≠ FAILED

Regla:

```text
UNKNOWN
≠
FAILED
```

---

# 82. UNKNOWN ≠ SUCCEEDED

Igualmente:

```text
UNKNOWN
≠
SUCCEEDED
```

---

# 83. Unknown insert danger

Reintentar ciegamente puede causar:

```text
duplicate row
duplicate business operation
duplicate generated identity
unique constraint violation
```

---

# 84. Retry safety

El Insert Persistence System debe exponer información para determinar:

```text
ReplaySafety
```

---

# 85. ReplaySafety

```php
enum InsertReplaySafety
{
    case IDEMPOTENT;
    case CONDITIONALLY_SAFE;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 86. Application-generated unique ID

Un insert con UUID único puede ser más reconciliable tras outcome desconocido.

Pero:

```text
UUID
≠
automatic idempotency
```

---

# 87. Business side effects

Triggers, sequences y otros efectos pueden hacer que un replay no sea trivial.

---

# 88. Retry ownership

El Insert Persistence System no implementa política global de retry.

Eso corresponde a:

```text
Execution Retry
Transaction Retry
Persistence Retry Policy
```

según contexto.

---

# 89. Duplicate-key failure

Debe preservarse como error estructurado.

---

# 90. Duplicate key ≠ identity-map conflict

Son capas distintas:

```text
DB unique conflict
≠
IdentityMap canonicality conflict
```

---

# 91. Constraint classification

Cuando sea posible:

```text
UniqueConstraintViolation
ForeignKeyViolation
NotNullViolation
CheckConstraintViolation
```

deberán normalizarse por el Execution/Driver error system.

---

# 92. Insert reconciliation

Solo después de ejecución se realiza:

```text
InsertReconciliation
```

---

# 93. Reconciliation order

Un flujo seguro:

```text
Validate execution outcome
↓
Validate affected-row semantics
↓
Validate generated values
↓
Convert generated values
↓
Assign generated entity values
↓
Establish canonical identity
↓
Register/promote IdentityMap identity
↓
Create persisted snapshot
↓
Update UnitOfWork state
↓
Complete lifecycle postPersist
```

---

# 94. Reconciliation must be coordinated

No deberá ocurrir:

```text
entity ID assigned
IdentityMap failed
snapshot absent
state MANAGED
```

como estado silenciosamente aceptado.

---

# 95. In-memory reconciliation transaction

VoltStack debe tratar las modificaciones internas como una transición coordinada.

Conceptualmente:

```text
begin ORM reconciliation
    assign generated values
    establish identity
    register IdentityMap
    create snapshot
    transition state
complete reconciliation
```

---

# 96. Reconciliation ≠ DB transaction

Es una coordinación del estado ORM en memoria.

---

# 97. Generated identifier assignment

Supóngase:

```php
$user->id === null;
```

antes del insert.

Después de resultado cierto:

```php
$user->id = new UserId(512);
```

mediante access strategy controlada.

---

# 98. Domain setter not mandatory

El ORM puede utilizar:

```text
compiled property accessor
```

sin exigir:

```php
setId()
```

público.

---

# 99. Generated identity validation

Antes de asignar:

```text
value present
type valid
not invalid sentinel
canonicalizable
mapping-compatible
```

---

# 100. Generated identity conflict

Antes de registrar:

```text
EntityKey
↓
IdentityMap lookup
```

Si ya existe otra instancia:

```text
InsertGeneratedIdentityConflictException
```

---

# 101. Never replace canonical entity silently

Prohibido:

```text
$newEntity overwrites existing IdentityMap entry
```

---

# 102. Assigned-ID registration

Una entidad con ID asignado puede haber reservado identidad previamente en el `IdentityMap`.

Después del insert:

```text
reservation
→ active canonical managed identity
```

---

# 103. Database-generated identity promotion

Antes:

```text
ObjectToken
```

Después:

```text
Canonical EntityKey
```

---

# 104. Promotion

```text
NEW object
+
generated ID
+
IdentityNamespace
+
EntityType
=
EntityKey
```

---

# 105. UNKNOWN outcome and identity

Si el resultado es `UNKNOWN`:

```text
do not fabricate generated identifier
```

---

# 106. Driver returned ID but operation uncertain

Debe preservarse la diferencia:

```text
GeneratedIdentifierObserved
```

y:

```text
InsertDurabilityCertain
```

No son lo mismo.

---

# 107. Identity assignment under uncertainty

La política debe ser explícita.

Default conservador:

```text
UNKNOWN persistence outcome
→ EntityManager TAINTED
→ no normal MANAGED reconciliation
```

---

# 108. TAINTED context

Cuando no puede garantizarse coherencia:

```text
EntityManager
→ TAINTED
```

---

# 109. TAINTED ≠ CLOSED

Puede conservar diagnostics para permitir:

```text
rollback
clear
discard
reconciliation workflow
```

según arquitectura transaccional futura.

---

# 110. Successful insert

Para considerar una operación semánticamente exitosa:

```text
ExecutionStatus = SUCCEEDED
∧
Certainty sufficient
∧
AffectedRow semantics valid
∧
Required generated values resolved
```

---

# 111. Affected rows

Para un insert individual normal:

```text
expected logical inserted entity count = 1
```

Pero el sistema no debe asumir que todos los drivers reportan `affectedRows` idénticamente.

---

# 112. Capability-aware row count

Puede existir:

```text
AffectedRowCountCapability
```

---

# 113. Missing row count ≠ failure

Si la plataforma no proporciona semántica fiable y no es requerida:

```text
UNKNOWN row count
```

puede ser aceptable.

---

# 114. Contradictory result

Si:

```text
status = SUCCEEDED
affectedRows = 0
```

donde el contrato exige 1:

```text
InsertAffectedRowCountException
```

---

# 115. Snapshot establishment

Tras reconciliación exitosa:

```text
Entity Snapshot
```

debe representar el nuevo baseline persistente conocido.

---

# 116. Snapshot includes generated values

Si DB generó:

```text
id
created_at
version
```

el snapshot debe usar los valores reconciliados.

---

# 117. Snapshot timing

No crear snapshot definitivo:

```text
before required generated values
```

---

# 118. Baseline

Después:

```text
CurrentPersistentState
=
SnapshotBaseline
```

salvo mutaciones posteriores de lifecycle.

---

# 119. State transition

Flujo normal:

```text
NEW
↓
INSERT_PENDING
↓
INSERT_EXECUTED
↓
RECONCILING
↓
MANAGED
```

Los estados intermedios pueden ser internos.

---

# 120. Fundamental ORM state remains

Públicamente pueden seguir existiendo:

```text
NEW
MANAGED
REMOVED
DETACHED
```

Los estados técnicos no tienen que contaminar la API pública.

---

# 121. postPersist timing

`postPersist` ocurre después de:

```text
successful semantic insert
+
generated identity reconciliation
+
managed-state reconciliation
```

---

# 122. postPersist ≠ commit

Regla absoluta:

```text
postPersist
⇏
transaction committed
```

---

# 123. postPersist generated ID

`postPersist` sí puede observar:

```text
generated identifier
```

si fue reconciliado.

---

# 124. postPersist mutation

Si modifica una propiedad persistente:

```text
entity becomes dirty for future persistence
```

No debe producir:

```text
hidden recursive UPDATE
```

---

# 125. prePersist

`prePersist` ocurre antes de sellar la operación final de inserción.

---

# 126. prePersist mutation

Puede modificar propiedades insertables.

Después:

```text
ChangeSet / insert state
```

debe recalcularse.

---

# 127. Lifecycle stabilization

```text
collect NEW entity
↓
prePersist
↓
detect mutation
↓
recompute
↓
stabilize
↓
Persistence Engine
↓
Planner
```

---

# 128. Recursive flush

Prohibido desde lifecycle handler.

---

# 129. Version initialization

Para optimistic locking:

```text
version
```

puede inicializarse durante insert.

---

# 130. Application-managed version

Ejemplo:

```text
version = 1
```

antes del insert.

---

# 131. Database-generated version

Alternativamente:

```text
version generated by DB
```

y deberá recuperarse si es requerida para futuros updates.

---

# 132. Version snapshot

El valor inicial debe formar parte del snapshot.

---

# 133. Created timestamps

VoltStack debe diferenciar:

```text
application-generated timestamp
database-generated timestamp
```

---

# 134. Database time ≠ application time

No deben tratarse como equivalentes implícitamente.

---

# 135. Database defaults

Una propiedad con default DB puede ser:

```text
OMITTED
```

para permitir la generación.

---

# 136. Need-to-know generated default

No todos los defaults deben recuperarse inmediatamente.

---

# 137. Example

Si:

```text
status DEFAULT 'pending'
```

pero la aplicación necesita `status` inmediatamente para snapshot/domain state, debe solicitar:

```text
REQUIRED_FOR_SNAPSHOT
```

o equivalente.

---

# 138. Optional refresh

Si no se necesita:

```text
OPTIONAL_REFRESH
```

puede diferirse.

---

# 139. Partial knowledge

VoltStack deberá representar que algunos valores persistidos pueden ser:

```text
database-known
ORM-not-yet-loaded
```

cuando el mapping lo permita.

---

# 140. Unknown property baseline

No debe inventarse un valor.

---

# 141. Snapshot integration

El Snapshot System deberá soportar:

```text
KNOWN
UNLOADED
UNKNOWN
```

según el modelo establecido.

---

# 142. INSERT DEFAULT VALUES

Una entidad puede no tener columnas explícitamente insertables.

Conceptualmente:

```text
INSERT default values
```

debe representarse en Query Model de manera abstracta.

---

# 143. Compiler responsibility

El compiler decide la sintaxis de:

```text
default-only insert
```

para cada plataforma.

---

# 144. Empty insert ≠ invalid automatically

Depende de capabilities.

---

# 145. Insert into generated table

Si todas las columnas son generadas:

```text
InsertQueryModel
```

puede seguir siendo válido.

---

# 146. Batch inserts

Múltiples `NEW` entities pueden ser candidatas a batching.

---

# 147. InsertBatchCandidate

```text
User A
User B
User C
```

pueden agruparse si:

```text
same target
same mapping
compatible columns
compatible generated-value strategy
compatible lifecycle
no dependency barriers
```

---

# 148. Different omitted columns

Dos inserts pueden tener diferentes shapes:

```text
A: name,email
B: name,email,status
```

Esto puede impedir batching simple.

---

# 149. Shape normalization

Una estrategia puede normalizar usando:

```text
DEFAULT
```

solo si la plataforma y semántica lo soportan.

---

# 150. Never replace omitted with NULL for batching

Prohibido.

---

# 151. Generated IDs and batching

El batch deberá preservar correlación:

```text
entity A ↔ generated id A
entity B ↔ generated id B
```

---

# 152. Order-based correlation

Solo será válida si la plataforma garantiza semánticamente el orden.

---

# 153. No guessed ID ranges

Prohibido inferir:

```text
firstId = 100
therefore IDs = 100,101,102
```

salvo capability contractual explícita que garantice exactamente esa semántica.

---

# 154. Sequence gaps

Los IDs no deberán asumirse consecutivos.

---

# 155. Triggers

Triggers pueden:

```text
consume sequences
change values
insert other rows
```

Por ello:

```text
generated ID arithmetic
```

es peligrosa.

---

# 156. Insert batching belongs to doc 133

Este documento define las restricciones semánticas.

La arquitectura completa de batch persistence se formalizará en:

```text
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
```

---

# 157. Soft delete

No afecta normalmente a insert salvo valores iniciales como:

```text
deleted_at = NULL
```

---

# 158. Soft-delete default

El mapping puede decidir:

```text
explicit null
```

o:

```text
database default
```

No deberá asumirse universalmente.

---

# 159. Tenant fields

Core Database no depende del paquete Multitenancy.

Pero un `DatabaseContext`/extensión puede contribuir:

```text
tenant identity namespace
tenant discriminator value
target database
schema
```

---

# 160. Tenant discriminator insertion

Si se usa shared-schema multitenancy:

```text
tenant_id
```

puede ser añadido mediante integración tipada.

---

# 161. No hidden global tenant

Prohibido:

```php
Tenant::$current
```

en el Persistence System.

---

# 162. Tenant value provenance

Debe conocerse si el valor proviene de:

```text
entity
database context
mapping policy
multitenancy extension
```

---

# 163. Security

Insert Persistence no sustituye Authorization.

---

# 164. Authorization ≠ persistence

Que una entidad pueda persistirse técnicamente no significa que el usuario esté autorizado.

---

# 165. Mass assignment

Tampoco corresponde a este sistema.

```text
Mass Assignment
≠
Insert Persistence
```

---

# 166. Parameter security

Valores se convierten a parámetros.

Nunca:

```text
SQL string concatenation
```

---

# 167. Raw expressions

Si alguna propiedad permite expresión DB explícita:

```text
RawExpression
```

deberá pasar por el sistema de escape hatch tipado.

No por strings arbitrarios.

---

# 168. Persistent runtime

El sistema deberá ser seguro con FrankenPHP.

---

# 169. Shared state

Puede compartirse:

```text
compiled metadata
insert strategy definitions
immutable capability definitions
frozen extension registry
```

---

# 170. Scoped state

Nunca compartir:

```text
current entity
current insert operation
generated IDs
reconciliation state
EntityManager
UnitOfWork
IdentityMap
tenant context
transaction context
```

---

# 171. FrankenPHP model

```text
Worker
├── Request A
│   ├── EntityManager A
│   ├── UoW A
│   └── InsertContext A
│
└── Request B
    ├── EntityManager B
    ├── UoW B
    └── InsertContext B
```

---

# 172. RoadRunner/OpenSwoole

Misma regla de aislamiento.

---

# 173. No static generated ID

Prohibido:

```php
InsertPersistence::$lastGeneratedId
```

---

# 174. Connection lifetime ≠ insert context lifetime

El insert context debe terminar aunque la conexión permanezca reutilizable.

---

# 175. Cancellation

Antes de enviar la query:

```text
safe cancellation
```

es simple.

Después de enviar:

```text
cancellation
```

puede producir outcome incierto.

---

# 176. Cancellation ≠ rollback certainty

Una cancelación no prueba que el DB no ejecutó el insert.

---

# 177. Timeout

Igualmente:

```text
timeout
```

puede generar:

```text
UNKNOWN
```

---

# 178. Network failure

Debe clasificarse según evidencia disponible.

---

# 179. Failure phases

```php
enum InsertFailurePhase
{
    case PREPARATION;
    case TRANSLATION;
    case COMPILATION;
    case PRE_EXECUTION;
    case EXECUTION;
    case GENERATED_VALUE_CAPTURE;
    case RECONCILIATION;
    case LIFECYCLE;
}
```

---

# 180. Preparation failure

No hubo DB side effect.

---

# 181. Compilation failure

Tampoco hubo DB side effect.

---

# 182. Pre-execution failure

No hubo DB side effect si se sabe que no se envió statement.

---

# 183. Execution failure

Puede ser:

```text
certain failure
```

o:

```text
unknown outcome
```

---

# 184. Reconciliation failure

Puede ocurrir después de que la DB haya insertado correctamente.

---

# 185. Important outcome dimensions

Se deben preservar separadamente:

```text
DatabaseOperationOutcome
ORMReconciliationOutcome
LifecycleOutcome
TransactionOutcome
```

---

# 186. Example

```text
DatabaseOperationOutcome = SUCCEEDED
ORMReconciliationOutcome = SUCCEEDED
LifecycleOutcome = FAILED
TransactionOutcome = PENDING
```

es un estado posible.

---

# 187. Another example

```text
DatabaseOperationOutcome = SUCCEEDED
ORMReconciliationOutcome = FAILED
TransactionOutcome = PENDING
```

requiere tratamiento crítico.

---

# 188. Reconciliation failure

El EntityManager puede quedar:

```text
TAINTED
```

porque DB y memoria pueden divergir.

---

# 189. No fake rollback

El ORM no debe fingir que la inserción nunca ocurrió solo porque la reconciliación falló.

---

# 190. Transaction rollback later

Si una transacción posterior hace rollback:

```text
row may disappear
```

pero el objeto pudo haber recibido:

```text
generated ID
```

---

# 191. Generated ID after rollback

VoltStack no debe asumir automáticamente:

```text
rollback
→ set ID back to null
```

Esto puede romper identidad/domain semantics.

---

# 192. Rollback reconciliation

Será formalizado con mayor profundidad en el sistema de transacciones.

---

# 193. Identifier consumption

Una secuencia/identity puede consumir un valor aunque haya rollback.

Por tanto:

```text
rollback
⇏
identifier reusable
```

---

# 194. Insert durability

Antes del commit:

```text
InsertSucceeded
```

significa que la operación fue exitosa dentro del contexto actual.

No necesariamente durable externamente.

---

# 195. Durability formula

```text
DurableInsert
=
InsertSucceeded
∧
TransactionCommitted
```

cuando existe una transacción que controla la operación.

---

# 196. Autocommit

En autocommit, la relación puede ser diferente.

Pero deberá venir del Transaction/Connection System, no inferirse localmente.

---

# 197. Events

El sistema puede producir eventos internos como:

```text
InsertPrepared
InsertExecutionSucceeded
InsertReconciled
InsertFailed
InsertOutcomeUnknown
```

---

# 198. ORM events ≠ domain events

No confundirlos.

---

# 199. InsertReconciled ≠ transaction committed

Tampoco debe usarse para enviar:

```text
email
webhook
broker message
```

que requiera durabilidad.

---

# 200. After-commit

Los side effects durables deberán usar:

```text
Transaction Synchronization
Transactional Outbox
Job After Commit
```

---

# 201. Telemetry

Métricas propuestas:

```text
orm.persistence.insert.prepared
orm.persistence.insert.executed
orm.persistence.insert.succeeded
orm.persistence.insert.failed
orm.persistence.insert.unknown
orm.persistence.insert.reconciled
orm.persistence.insert.generated_values
orm.persistence.insert.generated_identity
orm.persistence.insert.duration
orm.persistence.insert.reconciliation_duration
orm.persistence.insert.batch_candidate
orm.persistence.insert.identity_conflict
```

---

# 202. High-cardinality control

No usar IDs de entidad como labels de métricas.

---

# 203. Entity type metric

`EntityType` puede usarse solo bajo política de cardinalidad controlada.

---

# 204. Sensitive values

Nunca registrar por defecto:

```text
password
token
secret
full inserted row
personal data
```

---

# 205. Query telemetry

El Query Engine registra la query.

Insert Persistence registra:

```text
ORM semantic operation
```

---

# 206. Correlation

```text
PersistenceOperationId
↓
PersistencePlanStepId
↓
QueryId
↓
ExecutionId
```

permitirá correlación.

---

# 207. Diagnostics

Ejemplo:

```text
Insert Persistence
────────────────────────────────

Entity: App\Domain\User
State: NEW

Identifier:
  Strategy: DATABASE_IDENTITY
  Before Insert: UNASSIGNED
  Required After Insert: yes

Insert Values:
  name       EXPLICIT_VALUE
  email      EXPLICIT_VALUE
  status     OMITTED
  created_at DATABASE_DEFAULT

Generated Values:
  id         REQUIRED_FOR_IDENTITY
  created_at REQUIRED_FOR_SNAPSHOT

Strategy:
  INSERT_RETURNING

Dependencies:
  none

Execution:
  SUCCEEDED
  certainty: CERTAIN

Generated:
  id         received
  created_at received

Reconciliation:
  IdentityMap    OK
  Snapshot       OK
  UoW State      MANAGED
```

---

# 208. Extension architecture

```php
interface InsertPersistenceExtension
{
    public function contribute(
        InsertPersistenceExtensionContext $context,
    ): InsertPersistenceContribution;
}
```

---

# 209. Extension uses

Podrán implementar:

```text
custom generated IDs
custom database defaults
tenant discriminators
audit columns
custom version initialization
custom insert value policies
custom generated-value retrieval
```

---

# 210. Extension restrictions

No deberán:

```text
execute arbitrary SQL
commit
rollback
replace IdentityMap
silently remove required property
silently override another extension
```

---

# 211. Extension ordering

Determinista.

---

# 212. Extension conflicts

Dos extensiones que asignen valores incompatibles a la misma propiedad:

```text
explicit conflict
```

No:

```text
last wins
```

---

# 213. Insert contribution provenance

Cada valor contribuido deberá poder indicar origen.

```text
entity
mapping
default policy
tenant extension
audit extension
identifier generator
```

---

# 214. Directory structure

```text
src/Quantum/Database/ORM/Persistence/Insert/
│
├── Contract/
│   ├── InsertPersistenceSystem.php
│   ├── InsertPersistencePreparer.php
│   ├── InsertPersistenceReconciler.php
│   └── InsertPersistenceQueryTranslator.php
│
├── Operation/
│   ├── InsertEntityOperation.php
│   └── InsertPersistenceId.php
│
├── Preparation/
│   ├── DefaultInsertPersistenceSystem.php
│   ├── InsertPersistenceRequest.php
│   ├── PreparedInsertPersistence.php
│   ├── InsertPersistenceFingerprint.php
│   └── InsertPersistenceValidator.php
│
├── Value/
│   ├── InsertValue.php
│   ├── InsertValueKind.php
│   ├── InsertValueSet.php
│   ├── InsertValueResolver.php
│   └── PropertyInsertability.php
│
├── Identifier/
│   ├── EntityIdentifierGenerationStrategy.php
│   ├── InsertIdentifierStrategyResolver.php
│   ├── AssignedIdentifierStrategy.php
│   ├── ApplicationGeneratedIdentifierStrategy.php
│   ├── DatabaseGeneratedIdentifierStrategy.php
│   └── CompositeIdentifierInsertStrategy.php
│
├── Generated/
│   ├── GeneratedValueRequirement.php
│   ├── GeneratedValueRequirementSet.php
│   ├── GeneratedValueKind.php
│   ├── GeneratedValueNecessity.php
│   ├── InsertGeneratedValueStrategy.php
│   ├── InsertGeneratedValueStrategyResolver.php
│   └── GeneratedValueResultSet.php
│
├── Relationship/
│   ├── InsertRelationshipValueResolver.php
│   ├── InsertDependency.php
│   └── InsertDependencySet.php
│
├── Query/
│   ├── DefaultInsertPersistenceQueryTranslator.php
│   ├── InsertPersistenceTranslationContext.php
│   └── InsertQueryMetadata.php
│
├── Execution/
│   ├── InsertExecutionOutcome.php
│   ├── InsertExecutionStatus.php
│   ├── InsertExecutionRequirementSet.php
│   ├── InsertReplaySafety.php
│   └── InsertFailurePhase.php
│
├── Reconciliation/
│   ├── InsertPersistenceReconciler.php
│   ├── InsertReconciliationContext.php
│   ├── InsertReconciliationResult.php
│   ├── GeneratedValueReconciler.php
│   ├── GeneratedIdentityReconciler.php
│   ├── IdentityMapInsertReconciler.php
│   ├── SnapshotInsertReconciler.php
│   └── UnitOfWorkInsertReconciler.php
│
├── Extension/
│   ├── InsertPersistenceExtension.php
│   ├── InsertPersistenceExtensionRegistry.php
│   ├── InsertPersistenceExtensionContext.php
│   └── InsertPersistenceContribution.php
│
├── Telemetry/
│   ├── InsertPersistenceTelemetry.php
│   └── InsertPersistenceDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 215. Error hierarchy

```text
DatabaseOrmException
└── PersistenceException
    └── InsertPersistenceException
        ├── InsertPersistencePreparationException
        ├── InsertPersistenceValidationException
        ├── MissingRequiredInsertValueException
        ├── InvalidInsertValueException
        ├── InsertValueConversionException
        ├── InsertIdentifierException
        ├── InvalidAssignedIdentifierException
        ├── MissingAssignedIdentifierException
        ├── GeneratedIdentifierException
        ├── MissingGeneratedIdentifierException
        ├── CompositeIdentifierException
        ├── IncompleteCompositeIdentifierException
        ├── InsertGeneratedValueException
        ├── MissingGeneratedValueException
        ├── GeneratedValueCorrelationException
        ├── InsertRelationshipException
        ├── UnresolvedInsertRelationshipException
        ├── InsertDependencyException
        ├── InsertCapabilityException
        ├── InsertTranslationException
        ├── InsertExecutionException
        ├── InsertAffectedRowCountException
        ├── InsertOutcomeUnknownException
        ├── InsertReplaySafetyException
        ├── InsertGeneratedIdentityConflictException
        ├── InsertReconciliationException
        ├── InsertIdentityReconciliationException
        ├── InsertSnapshotReconciliationException
        ├── InsertUnitOfWorkReconciliationException
        ├── InsertLifecycleException
        ├── InsertExtensionException
        ├── InsertExtensionConflictException
        ├── InsertRuntimeIsolationException
        └── InsertPersistenceInvariantException
```

---

# 216. Testing strategy

Debe cubrir como mínimo:

```text
assigned identifiers
UUID/ULID
database-generated IDs
sequences
composite IDs
defaults
NULL semantics
generated columns
relationships
cascades
lifecycle
batch compatibility
execution outcomes
reconciliation
rollback implications
persistent runtime
extensions
```

---

# 217. Test — assigned ID

```text
NEW User(id=10)
```

genera insert con ID explícito.

---

# 218. Test — assigned ID does not prove existence

Antes del insert:

```text
EntityState = NEW
```

---

# 219. Test — UUID generation

UUID se genera antes de Query Model cuando strategy lo indica.

---

# 220. Test — ULID generation

Misma semántica.

---

# 221. Test — DB identity

ID permanece sin establecer hasta generated-value reconciliation.

---

# 222. Test — no fake ID

Nunca se usa ID negativo temporal.

---

# 223. Test — sequence preallocation

Identidad puede establecerse antes del insert sin marcar entidad `MANAGED`.

---

# 224. Test — composite complete

Inserción válida.

---

# 225. Test — composite incomplete

Falla antes de DB execution.

---

# 226. Test — explicit NULL

Produce parámetro NULL cuando mapping lo permite.

---

# 227. Test — omitted value

No se convierte en NULL.

---

# 228. Test — database default

La columna puede omitirse.

---

# 229. Test — required missing property

Falla en preparación.

---

# 230. Test — uninitialized PHP property

Respeta mapping.

---

# 231. Test — generated timestamp

Se reconcilia si requerido.

---

# 232. Test — optional generated default

No obliga roundtrip adicional.

---

# 233. Test — required snapshot default

Debe recuperarse.

---

# 234. Test — relationship assigned ID

FK se deriva correctamente.

---

# 235. Test — relationship generated ID

Respeta generated-value barrier.

---

# 236. Test — cascade persist

Genera operaciones separadas.

---

# 237. Test — no recursive insert

Insert subsystem no llama recursivamente al insert de related entity.

---

# 238. Test — deferred relationship

Solo permitido con estrategia válida.

---

# 239. Test — enum conversion

Correcto Type System.

---

# 240. Test — value object conversion

Correcto Type System.

---

# 241. Test — no SQL concatenation

Todos los valores ordinarios parameterized.

---

# 242. Test — successful insert

Reconciliación completa.

---

# 243. Test — generated identity conflict

IdentityMap protege canonicality.

---

# 244. Test — same entity reservation

Assigned-ID reservation se promueve correctamente.

---

# 245. Test — affected rows zero

Falla cuando contrato exige uno.

---

# 246. Test — row count unsupported

No falla si no es requerido.

---

# 247. Test — unknown execution outcome

No convierte entidad a MANAGED normalmente.

---

# 248. Test — connection loss after send

Context queda TAINTED cuando resultado no puede determinarse.

---

# 249. Test — retry unsafe

No reintenta ciegamente.

---

# 250. Test — duplicate key

Error DB normalizado, no IdentityMap conflict.

---

# 251. Test — generated value missing

Falla reconciliation.

---

# 252. Test — snapshot after generated values

Baseline contiene valores correctos.

---

# 253. Test — postPersist sees ID

Sí.

---

# 254. Test — postPersist is not commit

No se emite semántica after-commit.

---

# 255. Test — postPersist mutation

Produce dirty future state.

---

# 256. Test — prePersist mutation

Se incluye en insert.

---

# 257. Test — version initialization

Snapshot contiene version inicial.

---

# 258. Test — batch shape mismatch

No se agrupa incorrectamente.

---

# 259. Test — omitted vs NULL batching

No degrada semántica para crear batch.

---

# 260. Test — generated IDs batch correlation

Cada ID vuelve a la entidad correcta.

---

# 261. Test — no guessed consecutive IDs

Garantizado.

---

# 262. Test — tenant contribution

Se aplica mediante scoped integration.

---

# 263. Test — tenant isolation

Request A/B no intercambian tenant value.

---

# 264. Test — extension conflict

Error explícito.

---

# 265. Test — FrankenPHP

No hay generated IDs residuales entre requests.

---

# 266. Test — RoadRunner

Mismo aislamiento.

---

# 267. Test — OpenSwoole

Coroutine scope seguro.

---

# 268. Test — timeout after send

No se clasifica automáticamente como failed.

---

# 269. Test — cancellation after send

Preserva incertidumbre.

---

# 270. Test — reconciliation failure after DB success

EntityManager queda protegido/tainted.

---

# 271. Test — no implicit rollback

Insert system no ejecuta rollback.

---

# 272. Test — no implicit commit

Insert system no ejecuta commit.

---

# 273. Test — no compiler dependency in preparation

Preparación testeable sin SQL compiler.

---

# 274. Test — no driver dependency in semantic analysis

Análisis de insert testeable sin driver.

---

# 275. Anti-patterns

## 275.1 `persist()` ejecuta INSERT

Prohibido.

## 275.2 Entidad NEW marcada MANAGED antes de insert

Prohibido.

## 275.3 ID asignado significa row existente

Incorrecto.

## 275.4 ID generado ficticio

Prohibido.

## 275.5 `NULL` usado para representar omitted

Incorrecto.

## 275.6 Uninitialized PHP property convertido siempre en NULL

Incorrecto.

## 275.7 Todos los persistent fields se insertan

Incorrecto.

## 275.8 ORM genera SQL

Prohibido.

## 275.9 Vendor conditionals como arquitectura

Evitar.

## 275.10 `SELECT MAX(id)` para generated ID

Prohibido.

## 275.11 Adivinar IDs consecutivos en batch

Prohibido.

## 275.12 Recursive cascade inserts

Prohibido.

## 275.13 Insert system hace commit

Prohibido.

## 275.14 Insert system hace rollback

Prohibido.

## 275.15 Retry automático tras timeout

Peligroso y prohibido sin policy.

## 275.16 UNKNOWN tratado como FAILED

Incorrecto.

## 275.17 UNKNOWN tratado como SUCCESS

Incorrecto.

## 275.18 postPersist tratado como after-commit

Incorrecto.

## 275.19 DB-generated default inventado localmente

Incorrecto.

## 275.20 IdentityMap overwrite ante conflicto

Prohibido.

---

# 276. Architectural invariants

## DB-ORM-INSERT-001
`persist()` no ejecutará INSERT directamente.

## DB-ORM-INSERT-002
Una entidad `NEW` no implicará existencia de row.

## DB-ORM-INSERT-003
Una identidad asignada no implicará existencia de row.

## DB-ORM-INSERT-004
Insert Persistence será distinto de Query Builder.

## DB-ORM-INSERT-005
Insert Persistence será distinto de SQL Compiler.

## DB-ORM-INSERT-006
Insert Persistence será distinto de Execution Engine.

## DB-ORM-INSERT-007
Insert Persistence será distinto de Transaction Manager.

## DB-ORM-INSERT-008
Insert Persistence no generará SQL.

## DB-ORM-INSERT-009
Insert Persistence no utilizará PDO directamente.

## DB-ORM-INSERT-010
Insert Persistence no realizará commit.

## DB-ORM-INSERT-011
Insert Persistence no realizará rollback.

## DB-ORM-INSERT-012
Prepared insert no implicará executed insert.

## DB-ORM-INSERT-013
Persistent property será distinta de insertable property.

## DB-ORM-INSERT-014
Insertability será metadata compilado.

## DB-ORM-INSERT-015
Reflection no será requerida en hot path.

## DB-ORM-INSERT-016
OMITTED será distinto de EXPLICIT_NULL.

## DB-ORM-INSERT-017
DATABASE_DEFAULT será distinto de EXPLICIT_NULL.

## DB-ORM-INSERT-018
GENERATED será distinto de OMITTED.

## DB-ORM-INSERT-019
Uninitialized PHP property no será automáticamente NULL.

## DB-ORM-INSERT-020
Missing required property fallará antes de execution.

## DB-ORM-INSERT-021
Identifier strategies serán tipadas.

## DB-ORM-INSERT-022
Assigned identifier será soportado.

## DB-ORM-INSERT-023
Application-generated UUID será soportable.

## DB-ORM-INSERT-024
Application-generated ULID será soportable.

## DB-ORM-INSERT-025
Database-generated identity será soportable.

## DB-ORM-INSERT-026
Sequence/preallocation será capability-driven.

## DB-ORM-INSERT-027
Composite identifier será soportable.

## DB-ORM-INSERT-028
Composite identity incompleta no formará EntityKey final.

## DB-ORM-INSERT-029
No se crearán fake persistent identifiers.

## DB-ORM-INSERT-030
ObjectToken será distinto de EntityKey.

## DB-ORM-INSERT-031
Generated values no estarán limitados a identifiers.

## DB-ORM-INSERT-032
Required generated values serán declarados explícitamente.

## DB-ORM-INSERT-033
Generated-value retrieval será capability-driven.

## DB-ORM-INSERT-034
Generated-value strategy no generará SQL directamente.

## DB-ORM-INSERT-035
UNKNOWN capability no será asumida como supported.

## DB-ORM-INSERT-036
Follow-up generated-value query deberá preservar correlación.

## DB-ORM-INSERT-037
`SELECT MAX(id)` no será estrategia válida de identidad.

## DB-ORM-INSERT-038
Relationship object será distinto de FK database value.

## DB-ORM-INSERT-039
Relationship identifier será resuelto mediante metadata.

## DB-ORM-INSERT-040
Generated relationship identity producirá dependency/barrier.

## DB-ORM-INSERT-041
Known relationship ID no eliminará existence dependency automáticamente.

## DB-ORM-INSERT-042
Cascade persist será coordinado por UnitOfWork.

## DB-ORM-INSERT-043
Cascade persist no será recursive SQL insertion.

## DB-ORM-INSERT-044
Nullable relationship no implicará automatic cycle breaking.

## DB-ORM-INSERT-045
Deferred association requerirá estrategia explícita.

## DB-ORM-INSERT-046
Insert query será un Query Model.

## DB-ORM-INSERT-047
Query Model será traducido después de persistence planning.

## DB-ORM-INSERT-048
ORM field será distinto de raw SQL column string.

## DB-ORM-INSERT-049
Ordinary insert values serán parameterized.

## DB-ORM-INSERT-050
Type conversion utilizará Database Type System.

## DB-ORM-INSERT-051
Value Objects serán convertidos mediante mapping/type system.

## DB-ORM-INSERT-052
Enums serán convertidos mediante mapping/type system.

## DB-ORM-INSERT-053
Generated-key requirements serán explícitos.

## DB-ORM-INSERT-054
Same-connection requirements serán explícitos.

## DB-ORM-INSERT-055
Insert tendrá write intent.

## DB-ORM-INSERT-056
Connection routing permanecerá separado del ORM.

## DB-ORM-INSERT-057
Execution outcome será estructurado.

## DB-ORM-INSERT-058
UNKNOWN será first-class.

## DB-ORM-INSERT-059
UNKNOWN será distinto de FAILED.

## DB-ORM-INSERT-060
UNKNOWN será distinto de SUCCEEDED.

## DB-ORM-INSERT-061
Unknown insert no será reintentado ciegamente.

## DB-ORM-INSERT-062
Replay safety será explícita.

## DB-ORM-INSERT-063
UUID no implicará idempotency automática.

## DB-ORM-INSERT-064
Retry policy no pertenecerá al Insert Persistence System.

## DB-ORM-INSERT-065
Duplicate DB key será distinto de IdentityMap conflict.

## DB-ORM-INSERT-066
Constraint errors serán normalizables.

## DB-ORM-INSERT-067
Reconciliation ocurrirá después de execution outcome.

## DB-ORM-INSERT-068
Required generated values se validarán antes de MANAGED.

## DB-ORM-INSERT-069
Generated values serán convertidos mediante Type System.

## DB-ORM-INSERT-070
Generated identity será validada antes de assignment.

## DB-ORM-INSERT-071
Generated identity será canonicalizable.

## DB-ORM-INSERT-072
IdentityMap canonicality será preservada.

## DB-ORM-INSERT-073
IdentityMap entry existente no será reemplazada silenciosamente.

## DB-ORM-INSERT-074
Assigned-ID reservation podrá promoverse a managed identity.

## DB-ORM-INSERT-075
DB-generated identity podrá promover ObjectToken a EntityKey.

## DB-ORM-INSERT-076
UNKNOWN outcome no fabricará generated identity.

## DB-ORM-INSERT-077
Observed generated ID será distinto de durable insert.

## DB-ORM-INSERT-078
Unknown outcome podrá taint EntityManager.

## DB-ORM-INSERT-079
TAINTED será distinto de CLOSED.

## DB-ORM-INSERT-080
Successful semantic insert requerirá outcome suficientemente cierto.

## DB-ORM-INSERT-081
Affected-row semantics serán capability-aware.

## DB-ORM-INSERT-082
Missing row-count support no será automáticamente failure.

## DB-ORM-INSERT-083
Contradictory affected-row result será error.

## DB-ORM-INSERT-084
Snapshot será creado después de required generated values.

## DB-ORM-INSERT-085
Snapshot contendrá generated values conocidos.

## DB-ORM-INSERT-086
Snapshot no inventará DB-generated defaults.

## DB-ORM-INSERT-087
State transition a MANAGED será coordinada.

## DB-ORM-INSERT-088
Technical reconciliation states no tendrán que formar parte de public API.

## DB-ORM-INSERT-089
postPersist ocurrirá después de semantic insert reconciliation.

## DB-ORM-INSERT-090
postPersist será distinto de transaction commit.

## DB-ORM-INSERT-091
postPersist podrá observar generated identity reconciliada.

## DB-ORM-INSERT-092
postPersist mutation será future dirty state.

## DB-ORM-INSERT-093
postPersist no generará hidden recursive update.

## DB-ORM-INSERT-094
prePersist podrá alterar insertable state.

## DB-ORM-INSERT-095
prePersist mutation requerirá recomputation.

## DB-ORM-INSERT-096
Lifecycle stabilization ocurrirá antes del plan final.

## DB-ORM-INSERT-097
Recursive flush desde lifecycle será rechazado.

## DB-ORM-INSERT-098
Optimistic version podrá inicializarse en insert.

## DB-ORM-INSERT-099
Generated version será reconciliada si requerida.

## DB-ORM-INSERT-100
Initial version formará parte del snapshot.

## DB-ORM-INSERT-101
Application timestamp será distinto de DB timestamp.

## DB-ORM-INSERT-102
Database default podrá representarse mediante omission.

## DB-ORM-INSERT-103
No todos los DB defaults deberán recuperarse inmediatamente.

## DB-ORM-INSERT-104
Required snapshot defaults deberán recuperarse.

## DB-ORM-INSERT-105
Unknown persisted values no serán inventados.

## DB-ORM-INSERT-106
Default-only insert será representable.

## DB-ORM-INSERT-107
Compiler decidirá sintaxis de default-only insert.

## DB-ORM-INSERT-108
Empty explicit column set no será automáticamente inválido.

## DB-ORM-INSERT-109
Batch compatibility será semántica.

## DB-ORM-INSERT-110
Batching no cambiará OMITTED por NULL.

## DB-ORM-INSERT-111
Batch generated-value correlation será obligatoria.

## DB-ORM-INSERT-112
Generated IDs no serán inferidos por rangos sin capability contractual.

## DB-ORM-INSERT-113
Sequence gaps serán permitidos.

## DB-ORM-INSERT-114
Triggers impedirán suposiciones aritméticas sobre IDs.

## DB-ORM-INSERT-115
Batching tendrá menor prioridad que correctness.

## DB-ORM-INSERT-116
Multitenancy no será dependencia obligatoria del core.

## DB-ORM-INSERT-117
Tenant context será scoped.

## DB-ORM-INSERT-118
No existirá global mutable current tenant.

## DB-ORM-INSERT-119
Tenant discriminator contributions serán tipadas.

## DB-ORM-INSERT-120
Insert Persistence no sustituirá Authorization.

## DB-ORM-INSERT-121
Mass assignment será distinto de persistence.

## DB-ORM-INSERT-122
Raw expressions requerirán escape hatch tipado.

## DB-ORM-INSERT-123
Compiled metadata podrá compartirse entre requests.

## DB-ORM-INSERT-124
Insert operation state será request/operation-scoped.

## DB-ORM-INSERT-125
Generated IDs no serán process-global.

## DB-ORM-INSERT-126
EntityManager no será compartido entre requests.

## DB-ORM-INSERT-127
UnitOfWork no será compartido entre requests.

## DB-ORM-INSERT-128
IdentityMap no será compartido entre requests.

## DB-ORM-INSERT-129
Connection lifetime será distinta de insert-context lifetime.

## DB-ORM-INSERT-130
Cancellation before send podrá ser side-effect free.

## DB-ORM-INSERT-131
Cancellation after send podrá producir UNKNOWN.

## DB-ORM-INSERT-132
Timeout after send podrá producir UNKNOWN.

## DB-ORM-INSERT-133
Network failure no será automáticamente certain failure.

## DB-ORM-INSERT-134
Failure phase será preservada.

## DB-ORM-INSERT-135
Preparation failure no implicará DB mutation.

## DB-ORM-INSERT-136
Compilation failure no implicará DB mutation.

## DB-ORM-INSERT-137
Execution failure podrá tener outcome incierto.

## DB-ORM-INSERT-138
Reconciliation failure podrá ocurrir después de DB success.

## DB-ORM-INSERT-139
DB operation outcome será distinto de ORM reconciliation outcome.

## DB-ORM-INSERT-140
Lifecycle outcome será distinto de DB operation outcome.

## DB-ORM-INSERT-141
Transaction outcome será distinto de insert operation outcome.

## DB-ORM-INSERT-142
Reconciliation failure podrá taint EntityManager.

## DB-ORM-INSERT-143
ORM no fingirá rollback tras reconciliation failure.

## DB-ORM-INSERT-144
Transaction rollback no implicará limpiar generated ID automáticamente.

## DB-ORM-INSERT-145
Rollback no implicará reutilización de sequence ID.

## DB-ORM-INSERT-146
Insert success será distinto de durability.

## DB-ORM-INSERT-147
Durability dependerá del transaction outcome cuando aplique.

## DB-ORM-INSERT-148
Autocommit semantics vendrán del Transaction/Connection System.

## DB-ORM-INSERT-149
ORM insert events serán distintos de domain events.

## DB-ORM-INSERT-150
InsertReconciled será distinto de after-commit.

## DB-ORM-INSERT-151
External durable effects usarán transaction-aware integration.

## DB-ORM-INSERT-152
Telemetry no modificará insert semantics.

## DB-ORM-INSERT-153
Sensitive values no serán expuestos por defecto.

## DB-ORM-INSERT-154
PersistenceOperationId permitirá telemetry correlation.

## DB-ORM-INSERT-155
Extensions serán deterministas.

## DB-ORM-INSERT-156
Extensions no ejecutarán arbitrary SQL.

## DB-ORM-INSERT-157
Extensions no harán commit/rollback.

## DB-ORM-INSERT-158
Extension conflicts serán errores explícitos.

## DB-ORM-INSERT-159
Extension contribution provenance será preservado.

## DB-ORM-INSERT-160
Una entidad NEW solo alcanzará un baseline MANAGED cuando la inserción, la identidad requerida, los valores generados, el IdentityMap, el snapshot y el UnitOfWork hayan sido reconciliados de forma coherente bajo un resultado de ejecución suficientemente cierto.

---

# 277. Fórmulas fundamentales

## 277.1 Insert candidate

```text
InsertCandidate(E)
=
State(E) = NEW
∧
MappingValid(E)
∧
InsertRequirementsResolvable(E)
```

---

# 278. Insertable state

```text
InsertState(E)
=
ExplicitValues
+
ExplicitNulls
+
RelationshipIdentifiers
+
ApplicationGeneratedValues
+
DatabaseDefaultOmissions
```

---

# 279. Omission

```text
OMITTED
≠
NULL
≠
GENERATED
≠
DEFAULT_VALUE_KNOWN
```

---

# 280. Assigned identity

```text
AssignedIdentifier(E)
⇒
IdentityMayBeKnown(E)
```

pero:

```text
AssignedIdentifier(E)
⇏
RowExists(E)
```

---

# 281. Database-generated identity

```text
CanonicalIdentity(E)
=
Canonicalize(
    IdentityNamespace,
    EntityType,
    GeneratedIdentifier
)
```

solo después de obtener un identificador válido.

---

# 282. Generated-value safety

```text
SafeGeneratedValueReconciliation
=
ExecutionOutcomeSufficientlyCertain
∧
RequiredValuesPresent
∧
TypesValid
∧
CorrelationProven
∧
MappingCompatible
```

---

# 283. Relationship insert safety

```text
SafeRelationshipInsert(R)
=
RelationshipIdentifierResolvable
∧
RequiredExistenceDependencySatisfied
∧
TargetCompatible
∧
ConstraintSemanticsSatisfied
```

---

# 284. Successful insert

```text
SuccessfulInsert(E)
=
ExecutionSucceeded
∧
OutcomeCertaintySufficient
∧
AffectedRowSemanticsValid
∧
RequiredGeneratedValuesResolved
```

---

# 285. Managed transition

```text
NEW → MANAGED
```

solo si:

```text
SuccessfulInsert
∧
IdentityReconciled
∧
IdentityMapConsistent
∧
SnapshotEstablished
∧
UnitOfWorkReconciled
```

---

# 286. Insert durability

```text
InsertSucceeded
⇏
TransactionCommitted
```

y:

```text
DurableInsert
=
InsertSucceeded
∧
RequiredTransactionOutcome = COMMITTED
```

---

# 287. Unknown outcome

```text
UNKNOWN
⇒
¬ AssumeInserted
∧
¬ AssumeNotInserted
```

---

# 288. Replay

```text
SafeReplay
=
KnownReplaySemantics
∧
IdentityCorrelationSafe
∧
SideEffectsSafe
∧
TransactionStateCompatible
```

---

# 289. Snapshot baseline

```text
InitialSnapshot(E)
=
KnownPersistedState(E)
+
RequiredGeneratedValues(E)
```

Nunca:

```text
InitialSnapshot
=
GuessedDatabaseState
```

---

# 290. Batch compatibility

```text
InsertBatchable(A, B)
=
SameTarget
∧
CompatibleMapping
∧
CompatibleInsertShape
∧
CompatibleGeneratedValueStrategy
∧
NoDependencyBarrier
∧
LifecycleCompatible
∧
ResultCorrelationPreserved
```

---

# 291. Reconciliation consistency

```text
InsertReconciliationConsistent
=
EntityValuesConsistent
∧
EntityIdentityConsistent
∧
IdentityMapConsistent
∧
SnapshotConsistent
∧
UnitOfWorkConsistent
```

---

# 292. Master architecture

```text
                  EntityManager::persist()
                           │
                           ▼
                        UnitOfWork
                           │
                     Entity = NEW
                           │
                           ▼
                    Change Tracking
                           │
                           ▼
                  PreparedUnitOfWork
                           │
                           ▼
                  Persistence Engine
                           │
                           ▼
                InsertEntityOperation
                           │
                           ▼
                 Persistence Planner
                           │
                           ▼
                InsertPersistenceStep
                           │
                           ▼
             ┌─────────────────────────┐
             │ Insert Persistence      │
             │ Preparation             │
             └─────────────────────────┘
                           │
              ┌────────────┼─────────────┐
              ▼            ▼             ▼
          Identifier     Values      Relationships
           Strategy     Resolution    Resolution
              │            │             │
              └────────────┼─────────────┘
                           ▼
                Generated Requirements
                           │
                           ▼
                    Insert Query Model
                           │
                           ▼
                    Query Pipeline
                           │
                           ▼
                     SQL Compiler
                           │
                           ▼
                   Execution Engine
                           │
                           ▼
                       Database
                           │
                           ▼
                  Execution Outcome
                           │
                           ▼
             ┌─────────────────────────┐
             │ Insert Reconciliation   │
             └─────────────────────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
      Generated Values   Identity       Snapshot
            │              │              │
            │              ▼              │
            │         IdentityMap          │
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                        UnitOfWork
                           │
                           ▼
                         MANAGED
                           │
                           ▼
                      postPersist
                           │
                           ▼
               Transaction still separate
```

---

# 293. Master Formula

```text
Database Insert Persistence System
=
NEW Entity Persistence Intent
+
Compiled Entity Metadata
+
Insertable Property Resolution
+
Explicit Value Semantics
+
Omitted vs NULL Semantics
+
Database Default Semantics
+
Identifier Strategy Resolution
+
Assigned Identifier Support
+
UUID/ULID Generation
+
Database Identity Support
+
Sequence/Preallocation Support
+
Composite Identifier Validation
+
Generated Value Requirements
+
Generated Value Retrieval Strategies
+
Relationship Identifier Resolution
+
Insert Dependency Integration
+
Cascade Persist Coordination
+
Persistence Planner Integration
+
Insert Query Model Translation
+
Database Type Conversion
+
Parameter Binding
+
Write Intent
+
Execution Requirements
+
Outcome Certainty Modeling
+
Affected Row Validation
+
Replay Safety Modeling
+
Generated Value Capture
+
Generated Identity Reconciliation
+
IdentityMap Canonicality
+
Snapshot Establishment
+
UnitOfWork Reconciliation
+
Lifecycle Integration
+
Optimistic Version Initialization
+
Database Default Reconciliation
+
Batch Compatibility Semantics
+
Transaction Boundary Separation
+
Rollback Awareness
+
Persistent Runtime Isolation
+
Extension Governance
+
Telemetry
+
Diagnostics
+
Failure Modeling
```

---

# 294. Master Rule

> **En VoltStack, una inserción ORM es una transición coordinada entre intención, operación, ejecución y reconciliación. `persist()` registra la intención; el Persistence Engine describe qué debe insertarse; el Persistence Planner determina cuándo y bajo qué dependencias; el Query Engine representa la operación; el Execution Engine la ejecuta; y únicamente después de un resultado suficientemente cierto el ORM puede reconciliar valores generados, establecer identidad canónica, actualizar el IdentityMap, construir el snapshot inicial y promover coherentemente la entidad de `NEW` a `MANAGED`. Ninguna de estas fases debe confundirse con el commit de la transacción.**

---

# 295. Estado del bloque 11

Con este documento:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
```

la arquitectura posee ahora:

```text
Entity
  ↓
IdentityMap
  ↓
UnitOfWork
  ├── State
  ├── Snapshot
  └── ChangeSet
  ↓
Persistence Engine
  ↓
PersistenceOperationGraph
  ↓
Persistence Planner
  ↓
PersistencePlan
  ↓
Insert Persistence
  ↓
Query Engine
  ↓
Database
  ↓
Insert Reconciliation
```

Quedan por formalizar las otras operaciones fundamentales:

```text
UPDATE
DELETE
FLUSH
BATCH
CONSISTENCY
```

---

# 296. Siguiente documento

```text
130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
MANAGED Entity
      ↓
Change Tracking
      ↓
Snapshot Comparison
      ↓
ChangeSet
      ↓
UpdateEntityOperation
      ↓
Update Persistence Analysis
      ↓
Updatable Property Resolution
      ↓
Dirty Field Projection
      ↓
Relationship FK Changes
      ↓
Optimistic Lock Predicate
      ↓
Version Transition
      ↓
Persistence Planner
      ↓
Update Query Model
      ↓
Execution
      ↓
Affected Row Validation
      ↓
Generated/Updated Value Capture
      ↓
Snapshot Reconciliation
      ↓
UnitOfWork Reconciliation
      ↓
MANAGED / CLEAN
```

incluyendo especialmente:

```text
dirty-field updates
full-row vs partial updates
snapshot-based changes
notification-based changes
explicit dirty marking
updatable vs non-updatable properties
identifier immutability
relationship updates
nullable FK changes
optimistic locking
version predicates
version increments
zero affected rows
concurrent modification detection
database-generated update values
updated_at
computed columns
UPDATE RETURNING
no-op updates
bulk update distinction
lifecycle preUpdate/postUpdate
ChangeSet recalculation
outcome certainty
unknown update outcomes
retry safety
snapshot replacement
postUpdate future dirty state
transaction separation
persistent runtime
telemetry
diagnostics
extensions
testing
```

Regla central:

> **Un `UPDATE` en VoltStack no representa “guardar de nuevo la entidad completa”; representa sincronizar un ChangeSet persistente válido contra el baseline conocido, preservando concurrencia, versionado, identidad, resultado de ejecución y reconciliación del snapshot sin convertir `flush()` en `commit()`.**