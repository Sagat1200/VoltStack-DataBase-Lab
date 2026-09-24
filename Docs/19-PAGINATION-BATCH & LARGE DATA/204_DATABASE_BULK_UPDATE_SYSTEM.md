# 204_DATABASE_BULK_UPDATE_SYSTEM.md

# VoltStack Quantum Database
## Database Bulk Update System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 204 — Database Bulk Update System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `203_DATABASE_BULK_INSERT_SYSTEM.md`  
**Siguiente documento:** `205_DATABASE_BULK_DELETE_SYSTEM.md`

---

# 1. Propósito

`Database Bulk Update System` define la arquitectura mediante la cual VoltStack podrá modificar grandes conjuntos de registros de forma directa, tipada, planificada y eficiente, sin requerir la carga individual de todas las entidades ni simular que el `UnitOfWork` observó cambios que nunca atravesaron el ciclo normal del ORM.

Ejemplo:

```php
$result = DB::table('users')
    ->where('status', 'pending')
    ->updateBulk([
        'status' => 'active',
        'activated_at' => $now,
    ]);
```

También deberá soportar actualizaciones heterogéneas:

```php
$result = DB::table('products')->updateBulkByKey(
    rows: [
        [
            'id' => 100,
            'price' => 199.90,
            'stock' => 20,
        ],
        [
            'id' => 101,
            'price' => 249.90,
            'stock' => 35,
        ],
    ],
    key: 'id',
);
```

y datasets incrementales:

```php
$result = DB::table('products')->updateBulkByKey(
    rows: $updates,
    key: 'id',
    batchSize: 1000,
);
```

La regla central será:

> **Bulk Update en VoltStack modificará conjuntos de registros directamente mediante operaciones semánticas y tipadas; no representará un ciclo abreviado de `EntityManager`, `UnitOfWork` o `Entity::save()`, y cualquier estado ORM potencialmente afectado deberá tratarse explícitamente como posiblemente stale.**

Formalmente:

```text
BulkUpdate
=
TargetSelection
+
MutationDefinition
+
TypeResolution
+
Routing
+
BatchPlanning
+
Compilation
+
Execution
+
OutcomeAggregation
+
ORMCoherenceHandling
```

Nunca:

```text
BulkUpdate
=
foreach ($entities as $entity) {
    mutate($entity);
    $entityManager->flush();
}
```

---

# 2. Posición dentro del Bloque 19

```text
199 Pagination
200 Cursor Pagination
201 Chunk Processing
202 Lazy Collection

203 Bulk Insert
204 Bulk Update
205 Bulk Delete

206 Import
207 Export
208 Large Dataset Processing
```

La secuencia de mutación masiva será:

```text
INSERT
    crea filas

UPDATE
    modifica filas existentes

DELETE
    elimina filas existentes
```

Los tres deberán compartir infraestructura común sin confundirse semánticamente.

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Bulk Update
≠
Entity Mutation Loop
≠
UnitOfWork Change Tracking
≠
Batch Persistence
≠
Query Builder
≠
Upsert
≠
Bulk Insert
≠
Bulk Delete
≠
Data Migration
≠
Import
```

---

# 4. Bulk Update vs ORM Update

ORM:

```php
$user = $repository->find($id);

$user->activate();

$entityManager->flush();
```

puede involucrar:

```text
Entity
↓
EntityState
↓
Change Tracking
↓
ChangeSet
↓
UnitOfWork
↓
Persistence Planner
↓
Update Query
```

Bulk Update:

```text
Predicate / Keys
↓
Mutation Definition
↓
Bulk Update Planner
↓
Query Engine
↓
Execution
```

No requiere entidades managed.

---

# 5. Consecuencia principal

Después de:

```php
DB::table('users')
    ->where('status', 'pending')
    ->updateBulk([
        'status' => 'active',
    ]);
```

una entidad ya cargada:

```php
$user = $entityManager->find(User::class, 10);
```

puede continuar mostrando:

```text
status = pending
```

en memoria.

Por tanto:

```text
Database State
≠
Current Managed Object State
```

después de una mutación bulk externa al `UnitOfWork`.

---

# 6. Bulk Update vs Batch Persistence

`133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md` agrupa trabajo ORM.

Bulk Update modifica directamente registros.

```text
Batch Persistence
=
many ORM changes
+
canonical ORM semantics

Bulk Update
=
set-oriented database mutation
```

---

# 7. Bulk Update vs Upsert

Upsert combina:

```text
insert if absent
+
update if conflict
```

Bulk Update solo opera sobre registros que satisfacen:

```text
selection criteria
```

No deberá crearse una fila ausente por default.

---

# 8. Bulk Update vs Data Migration

Una Migration puede utilizar Bulk Update para transformar datos.

Pero:

```text
Migration
=
evolution workflow

Bulk Update
=
mutation primitive
```

---

# 9. Dos familias principales

VoltStack deberá soportar al menos dos modelos conceptuales.

## 9.1 Set-based update

Misma mutación para todas las filas seleccionadas.

```php
DB::table('subscriptions')
    ->where('expires_at', '<', $now)
    ->updateBulk([
        'status' => 'expired',
    ]);
```

Conceptualmente:

```text
Target Predicate
+
Shared Mutation
```

## 9.2 Keyed heterogeneous update

Cada fila puede recibir valores distintos.

```php
DB::table('products')->updateBulkByKey(
    rows: [
        ['id' => 1, 'price' => 100],
        ['id' => 2, 'price' => 200],
    ],
    key: 'id',
);
```

Conceptualmente:

```text
Key → Mutation A
Key → Mutation B
Key → Mutation C
```

---

# 10. No mezclar ambas semánticas

```text
SetBasedBulkUpdate
≠
KeyedBulkUpdate
```

Aunque ambos compartan:

```text
types
routing
transactions
batching
telemetry
results
```

---

# 11. Arquitectura general

```text
Bulk Update API
      │
      ▼
BulkUpdateRequest
      │
      ▼
BulkUpdateNormalizer
      │
      ▼
BulkUpdatePlanner
      │
      ├── Selection Analysis
      ├── Mutation Analysis
      ├── Type Resolution
      ├── Routing
      ├── Batch Strategy
      ├── Locking/Versioning
      ├── Transaction Policy
      ├── Returning Policy
      ├── ORM Coherence
      └── Resource Policy
      │
      ▼
BulkUpdatePlan
      │
      ▼
BulkUpdateRunner
      │
      ▼
UpdateQueryModel(s)
      │
      ▼
Query Engine
      │
      ▼
Compiler
      │
      ▼
Execution Engine
      │
      ▼
BulkUpdateResult
```

---

# 12. API set-based

```php
$result = DB::table('users')
    ->where('active', false)
    ->updateBulk([
        'active' => true,
        'updated_at' => $now,
    ]);
```

---

# 13. Query Builder integration

El `where()` previo deberá permanecer en el Query Model.

Bulk Update no deberá reconstruirlo desde SQL strings.

---

# 14. Mutation Model

Propuesta:

```php
final readonly class BulkUpdateMutation
{
    /**
     * @param array<ColumnId, BulkUpdateValue> $assignments
     */
    public function __construct(
        public array $assignments,
    ) {}
}
```

---

# 15. Update values

Un valor podrá ser:

```text
LiteralValue
BoundValue
ExpressionValue
IncrementValue
DecrementValue
DefaultValue
NullValue
```

---

# 16. Ejemplo incremento

```php
DB::table('products')
    ->where('category_id', 50)
    ->updateBulk([
        'stock' => DB::increment(10),
    ]);
```

deberá convertirse a una expresión semántica:

```text
stock = stock + 10
```

no a SQL raw concatenado.

---

# 17. Expression system

Deberá reutilizar:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM
```

---

# 18. Bulk Update no será Expression Engine

Nunca:

```text
BulkUpdatePlanner
→ parse arbitrary SQL expression strings
```

---

# 19. Type resolution

Para:

```php
[
    'status' => OrderStatus::PAID,
]
```

deberá utilizarse:

```text
Type Registry
Value Conversion
Enum Mapping
```

---

# 20. Tipos

Se deberán reutilizar:

```text
155 Type System
156 Type Registry
157 Value Conversion
158 Casting
159 Enum Mapping
160 Value Object Mapping
161 JSON
162 DateTime
163 Custom Types
```

---

# 21. Mutation type validation

Si:

```text
column = integer
value = incompatible object
```

deberá rechazarse antes de ejecución cuando exista suficiente metadata.

---

# 22. NULL

Siempre:

```text
SET column = NULL
```

será distinto de:

```text
do not update column
```

---

# 23. Missing field

En keyed heterogeneous updates:

```php
[
    ['id' => 1, 'price' => 100],
    ['id' => 2, 'stock' => 50],
]
```

la ausencia de `price` en la segunda fila significa:

```text
leave price unchanged
```

no:

```text
price = NULL
```

---

# 24. Mutation shape

Por tanto cada fila podrá tener:

```text
Key Shape
+
Mutation Shape
```

---

# 25. BulkUpdateShape

```php
final readonly class BulkUpdateShape
{
    public function __construct(
        public array $keyColumns,
        public array $mutationColumns,
    ) {}
}
```

---

# 26. Shape grouping

Filas heterogéneas podrán agruparse:

```text
[id, price]
[id, price]
[id, stock]
[id, price, stock]
```

en grupos compatibles.

---

# 27. Reordering

Igual que Bulk Insert, cualquier reordenamiento deberá preservar:

```text
InputRowIndex
```

si el resultado requiere correlación.

---

# 28. Keyed Bulk Update

Propuesta:

```php
$result = DB::table('products')
    ->updateBulkByKey(
        rows: $rows,
        key: ['id'],
    );
```

---

# 29. Composite key

También:

```php
key: [
    'tenant_id',
    'invoice_number',
]
```

---

# 30. Key ≠ Primary Key necesariamente

Puede utilizarse una unique key explícita si:

```text
semantic uniqueness
+
routing
+
platform behavior
```

son compatibles.

---

# 31. Non-unique key

Si la key no es única:

```text
one input row
→ potentially many DB rows
```

Eso deberá ser explícitamente soportado o rechazado.

Default recomendado:

```text
REQUIRE_UNIQUE_KEY
```

---

# 32. Key validation

Metadata podrá probar:

```text
PRIMARY KEY
UNIQUE CONSTRAINT
```

pero deberá considerar:

```text
tenant scope
NULL semantics
partial indexes
platform semantics
```

---

# 33. Set-based update model

Conceptualmente:

```text
UPDATE Target
SET Mutation
WHERE Predicate
```

pertenece al `UpdateQueryModel`.

---

# 34. Keyed heterogeneous update strategies

El planner podrá seleccionar estrategias como:

```text
MULTIPLE_UPDATE_STATEMENTS
CASE_EXPRESSION
VALUES_JOIN
TEMPORARY_RELATION
NATIVE_BULK_UPDATE
CUSTOM
```

---

# 35. CASE strategy

Ejemplo físico posible:

```sql
UPDATE products
SET price = CASE id
    WHEN ? THEN ?
    WHEN ? THEN ?
END
WHERE id IN (?, ?);
```

Pero esta es una decisión del Compiler/Planner específico de plataforma.

---

# 36. CASE limitations

Con múltiples columnas y shapes puede crecer:

```text
statement size
parameters
compiler complexity
```

rápidamente.

---

# 37. VALUES join

Algunas plataformas permiten representar:

```text
input rows
as
derived relation
```

y hacer join/update.

---

# 38. Temporary relation

Otra estrategia:

```text
load update dataset
→ temporary/staging relation
→ set-based UPDATE JOIN
```

puede ser apropiada para volúmenes muy grandes.

---

# 39. Temporary table ≠ core requirement

No todas las plataformas/contexts la soportan.

Además introduce:

```text
lifecycle
cleanup
transaction visibility
connection pinning
permissions
```

---

# 40. Native bulk update

Podrán existir extensiones específicas si algún driver/plataforma ofrece capacidades especializadas.

---

# 41. Strategy resolver

```php
enum BulkUpdateExecutionStrategy
{
    case SET_BASED;
    case MULTI_STATEMENT;
    case CASE_EXPRESSION;
    case VALUES_RELATION;
    case TEMPORARY_RELATION;
    case NATIVE;
    case CUSTOM;
}
```

---

# 42. Strategy ≠ semantics

Distintas estrategias físicas deberán producir la misma semántica lógica requerida.

---

# 43. Planner

La selección deberá considerar:

```text
platform capabilities
row count
column count
shape count
parameter limits
statement size
transaction policy
returning requirements
routing
locking/versioning
resource budget
```

---

# 44. Batching

Para keyed updates:

```text
Rows
↓
Route
↓
Shape
↓
Batch
↓
Physical strategy
```

---

# 45. Effective batch size

Conceptualmente:

```text
EffectiveBatchSize
=
min(
    RequestedBatchSize,
    ParameterCapacity,
    StatementCapacity,
    PlatformCapacity,
    ResourceBudget
)
```

---

# 46. Parameter complexity

Para CASE updates, la cantidad de parámetros puede crecer aproximadamente como:

```text
Rows × (
    KeyComponents
    +
    UpdatedColumns
    +
    PredicateComponents
)
```

dependiendo de la estrategia.

---

# 47. No límite fijo universal

Los límites deberán provenir de:

```text
Platform Capability System
```

---

# 48. Input sources

Keyed updates deberán soportar:

```text
array
Iterator
Generator
LazyCollection
BulkUpdateRowSource
```

---

# 49. Streaming input

Ejemplo:

```php
DB::table('products')->updateBulkByKey(
    rows: productUpdates(),
    key: 'id',
    batchSize: 1000,
);
```

no deberá materializar toda la fuente.

---

# 50. Backpressure

Pipeline:

```text
Read input batch
↓
Plan/normalize batch
↓
Execute
↓
Release
↓
Read next batch
```

---

# 51. Infinite source

Al igual que Bulk Insert, deberá estar gobernada por:

```text
maxRows
deadline
cancellation
```

cuando corresponda.

---

# 52. Empty updates

Una fila:

```php
[
    'id' => 10,
]
```

sin columnas a modificar puede:

```text
SKIP
ERROR
```

según policy.

Default recomendado:

```text
SKIP_NOOP
```

con telemetry/debugging apropiado.

---

# 53. No-op update

También una asignación:

```text
price = current price
```

puede ser lógicamente no-op, pero VoltStack no deberá realizar pre-read obligatorio para comprobarlo.

---

# 54. Affected rows semantics

Una dificultad importante:

```text
affected rows
```

puede tener distinta semántica por plataforma/driver.

Puede significar:

```text
matched rows
changed rows
reported rows
```

---

# 55. Distinciones

VoltStack deberá separar:

```text
RowsMatched
RowsChanged
RowsReported
RowsKnown
```

cuando exista evidencia.

---

# 56. No asumir

Nunca:

```text
driverAffectedRows
=
rows actually changed
```

sin capability/contract.

---

# 57. BulkUpdateCountResult

```php
final readonly class BulkUpdateCountResult
{
    public function __construct(
        public ?int $matched,
        public ?int $changed,
        public ?int $reported,
        public BulkCountKnowledge $knowledge,
    ) {}
}
```

---

# 58. UNKNOWN counts

Si solo se conoce:

```text
reported = 500
```

no inventar:

```text
matched = 500
changed = 500
```

---

# 59. RETURNING

Cuando la plataforma soporte:

```text
UPDATE ... RETURNING
```

podrá solicitarse:

```php
returning: [
    'id',
    'version',
    'updated_at',
]
```

---

# 60. Returning use cases

Útil para:

```text
new optimistic version
computed columns
trigger-generated values
updated timestamps
```

---

# 61. Returning ≠ ORM synchronization automática

Aunque VoltStack conozca nuevos valores DB:

```text
BulkUpdateReturningResult
```

no deberá aplicar cambios ciegamente a objetos managed.

---

# 62. Razón

El objeto puede tener:

```text
dirty unflushed changes
different lifecycle state
partial loaded state
relationship changes
```

---

# 63. Optimistic locking

Bulk Update deberá soportar versiones cuando sea solicitado.

---

# 64. Single-version set update

Ejemplo:

```text
UPDATE users
SET status = 'active',
    version = version + 1
WHERE status = 'pending'
  AND version = ?
```

puede ser válido para un conjunto específico.

---

# 65. Per-row optimistic versions

Keyed heterogeneous update puede incluir:

```php
[
    [
        'id' => 100,
        'expected_version' => 5,
        'price' => 100,
    ],
    [
        'id' => 101,
        'expected_version' => 12,
        'price' => 200,
    ],
]
```

---

# 66. Optimistic lock result

Para cada input lógico podrá existir:

```text
UPDATED
CONFLICT
NOT_FOUND
UNKNOWN
```

si la estrategia permite distinguirlos.

---

# 67. Zero affected rows

Para una operación versionada:

```text
affected = 0
```

puede significar:

```text
row missing
or
version conflict
```

si no existe evidencia adicional.

---

# 68. No inventar conflicto

No deberá afirmar:

```text
OptimisticLockConflict
```

cuando no puede distinguirlo de `NOT_FOUND`.

---

# 69. Pre-read tradeoff

Una lectura previa podría distinguir casos, pero introduce:

```text
race window
additional I/O
different snapshot
```

y no equivale necesariamente al estado del UPDATE.

---

# 70. Returning/CTE strategy

Una plataforma podría permitir estrategias más precisas.

Deberán ser capability-driven.

---

# 71. Optimistic locking reuse

Se integrará con:

```text
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
```

No se creará un sistema paralelo.

---

# 72. Pessimistic locking

Bulk Update podrá ejecutarse dentro de un workflow que previamente bloqueó filas.

Pero Bulk Update no deberá adquirir locks ORM implícitos salvo policy explícita.

---

# 73. Update with predicate safety

Una operación:

```php
DB::table('users')->updateBulk([
    'active' => false,
]);
```

sin `WHERE` podría actualizar toda la tabla.

---

# 74. Safety policy

Debe existir:

```php
enum BulkUpdateScopeSafety
{
    case REQUIRE_PREDICATE;
    case ALLOW_ALL_EXPLICIT;
    case ALLOW_ALL;
}
```

---

# 75. Default recomendado

```text
REQUIRE_PREDICATE
```

para APIs orientadas a aplicación.

Para actualizar todo deberá usarse algo explícito:

```php
DB::table('users')
    ->allRows()
    ->updateBulk([...]);
```

o equivalente.

---

# 76. allRows()

Debe representar intención explícita:

```text
FullTableMutationAcknowledged
```

---

# 77. WHERE true ≠ allRows acknowledgement

No deberá considerarse suficiente:

```text
WHERE 1 = 1
```

como señal de seguridad.

---

# 78. Estimated impact

Si metadata/optimizer puede estimar cardinalidad:

```text
EstimatedRows = 25,000,000
```

Resource/Safety Policy podrá:

```text
WARN
REQUIRE_CONFIRMATION
REJECT
```

en herramientas interactivas.

---

# 79. Core no prompt

El Database core no deberá mostrar preguntas de terminal.

Solo producirá:

```text
SafetyDecision / Diagnostic
```

CLI decidirá UX.

---

# 80. Bulk update + joins

Una actualización podrá depender de joins/subqueries si Query Model lo soporta.

---

# 81. Portabilidad

No todas las plataformas permiten el mismo estilo físico de:

```text
UPDATE JOIN
UPDATE FROM
```

---

# 82. Logical model

VoltStack deberá expresar:

```text
Update target
+
selection relation
+
predicate
+
mutation
```

y permitir al Compiler decidir representación.

---

# 83. Unsupported update form

Si una semántica no puede representarse de forma correcta:

```text
UNSUPPORTED
```

deberá ser preferible a una transformación incorrecta.

---

# 84. CTE

Bulk Update podrá utilizar CTEs cuando el Query Model y plataforma lo permitan.

---

# 85. Subqueries

Las assignments podrían contener subqueries válidas según Type/Semantic Analyzer.

---

# 86. Query validation

El Semantic Query Engine deberá verificar:

```text
target symbols
column types
scope
subquery cardinality
assignment compatibility
```

---

# 87. Raw expressions

Continúan siendo escape hatch explícito.

---

# 88. ORM coherence problem

Después de Bulk Update:

```text
Database
    updated

IdentityMap
    potentially stale
```

---

# 89. Coherence policies

Propuesta:

```php
enum BulkOrmCoherencePolicy
{
    case IGNORE;
    case MARK_POTENTIALLY_STALE;
    case EVICT_MATCHED_WHEN_IDENTIFIABLE;
    case REFRESH_RETURNED;
    case REJECT_IF_MANAGED_TARGETS_PRESENT;
    case CUSTOM;
}
```

---

# 90. Default recomendado

```text
MARK_POTENTIALLY_STALE
```

para contextos ORM activos.

---

# 91. IGNORE

Adecuado cuando:

```text
no active EntityManager
or
caller explicitly accepts stale state
```

---

# 92. Mark stale

El EntityManager/UoW podrá registrar:

```text
ExternalBulkMutationMarker
```

asociado a:

```text
entity type
field set
predicate/domain
```

cuando pueda representarse.

---

# 93. Predicate complexity

Una mutation:

```text
WHERE score > complex_subquery()
```

puede hacer imposible identificar objetos exactos en IdentityMap.

Entonces:

```text
EntityTypePotentiallyStale
```

puede ser más seguro.

---

# 94. EVICT_MATCHED

Solo es viable cuando las keys afectadas son conocidas con suficiente precisión.

Ejemplo:

```text
id IN (10, 20, 30)
```

---

# 95. Eviction ≠ object deletion

Sacar una entidad del IdentityMap:

```text
DETACHED
```

no elimina su objeto PHP ni revierte referencias externas.

---

# 96. REFRESH_RETURNED

Si `RETURNING` proporciona IDs/valores, una policy avanzada podría refrescar entidades managed.

---

# 97. Riesgo de refresh

Nunca sobrescribir automáticamente:

```text
dirty unflushed local changes
```

---

# 98. Dirty managed entity

Si una entidad afectada está dirty:

```text
Bulk DB Update
+
Managed Dirty Object
```

existe un conflicto de autoridad.

---

# 99. Default

No hacer merge silencioso.

Puede:

```text
mark stale/conflicted
reject refresh
require caller resolution
```

---

# 100. Flush after Bulk Update

Caso:

```text
Bulk Update:
status = active

Managed entity:
status = pending

Later UoW flush
```

puede sobrescribir el bulk update.

---

# 101. Important warning

Bulk Update dentro de un EntityManager activo debe ser considerado una operación capaz de invalidar assumptions del UoW.

---

# 102. UoW notification

El sistema podrá notificar:

```text
BulkExternalMutation
```

para que `UnitOfWork` decida:

```text
STALE
CONFLICTED
REQUIRES_REFRESH
```

según sus políticas.

---

# 103. Bulk Update no reescribe snapshots automáticamente

No deberá modificar snapshots de entidades que no fueron realmente reconciliadas.

---

# 104. IdentityMap ≠ DB mirror

Regla ya establecida:

```text
IdentityMap
≠
Database Cache
```

---

# 105. Relationship implications

Actualizar foreign keys directamente puede afectar:

```text
to-one relationship
inverse collections
relationship coverage
```

---

# 106. Example

```text
orders.customer_id
10 → 20
```

Bulk Update directo puede dejar:

```text
Customer(10).orders
Customer(20).orders
Order.customer
```

stale en memoria.

---

# 107. Relationship metadata integration

Si el campo afectado participa en una relación, la coherence policy deberá poder escalar:

```text
entity stale
relationship stale
collection stale
```

---

# 108. No relationship fixup automático

No deberá intentar recorrer todo el IdentityMap y rearmar grafos complejos por default.

---

# 109. Many-to-many

Bulk Update normalmente no aplica directamente a membership tables mediante ORM Relationship semantics.

Modificar una join table directamente será una low-level mutation y deberá invalidar relación correspondiente.

---

# 110. Soft Delete boundary

Si una entidad utiliza:

```text
Soft Delete
```

actualizar:

```text
deleted_at
```

mediante Bulk Update puede equivaler semánticamente a soft deletion.

---

# 111. Pero

La arquitectura formal de soft delete pertenecerá a:

```text
269_DATABASE_SOFT_DELETE_SYSTEM.md
```

---

# 112. Bulk Update no redefinirá soft delete

El sistema avanzado podrá reconocer metadata de soft delete y emitir:

```text
BulkSoftDeleteMutation
```

pero no duplicará el subsystem.

---

# 113. Audit/version fields

Campos como:

```text
updated_at
updated_by
version
```

no deberán modificarse mágicamente salvo metadata/policy explícita.

---

# 114. Automatic timestamps

Una Model API convenience podría solicitar:

```text
touchUpdatedAt = true
```

pero debe ser una opción visible.

---

# 115. ORM callbacks

Bulk Update no disparará automáticamente:

```text
preUpdate
postUpdate
entity observers
domain methods
```

por fila.

---

# 116. Razón

No existen objetos Entity individuales necesariamente.

---

# 117. Bulk lifecycle events

En su lugar:

```text
BulkUpdatePlanned
BulkUpdateStarted
BulkUpdateBatchStarted
BulkUpdateBatchCompleted
BulkUpdateCommitted
BulkUpdateCompleted
BulkUpdateFailed
```

---

# 118. Domain invariants

Bulk Update es API avanzada.

Puede saltarse lógica como:

```php
$order->cancel();
```

que quizá verifica:

```text
payment state
permissions
stock
audit
domain events
```

---

# 119. Recommendation

Utilizar Bulk Update para:

```text
maintenance
derived flags
status reconciliation
ETL
administrative transformations
mass data corrections
```

cuando sus invariantes estén claramente comprendidas.

---

# 120. Security scopes

Bulk Update deberá respetar cualquier:

```text
tenant predicate
authorization scope
resource scope
row-level policy
```

que forme parte del Query Model.

---

# 121. Authorization ≠ post-filter

Nunca:

```text
UPDATE all rows
then decide which were authorized
```

---

# 122. Full-table mutation authorization

`allRows()` no elimina necesidad de autorización.

---

# 123. Sensitive assignments

Telemetry/logs no deberán registrar:

```text
password hashes
tokens
PII
secret values
full JSON payloads
```

por default.

---

# 124. Credential fields

Si se actualizan credenciales o hashes, la responsabilidad de producir valores seguros pertenece al subsystem correspondiente.

Bulk Update solo persiste valores tipados.

---

# 125. Routing

Bulk Update es una operación de escritura.

Siempre:

```text
ReadWriteIntent = WRITE
```

---

# 126. Replica prohibition

Nunca enviar Bulk Update a:

```text
read replica
```

---

# 127. Sharded set-based update

Una query puede resolverse a:

```text
ONE_SHARD
SHARD_SET
ALL_SHARDS
UNKNOWN
```

---

# 128. Ordinary write

Cuando la semántica exige un único shard:

```text
UNKNOWN
```

debe fallar.

---

# 129. Multi-shard administrative update

Puede existir explícitamente:

```text
all shards
```

pero no implica una transacción global.

---

# 130. Keyed row routing

Para updates por key, cada input puede resolverse:

```text
Row A → Shard 1
Row B → Shard 2
```

y agruparse por shard.

---

# 131. Routing key mutation

Caso difícil:

```text
current shard key = A
new shard key = B
```

Eso puede implicar mover ownership entre shards.

---

# 132. Rule

Un `UPDATE` ordinario no deberá cambiar un shard key de modo que la fila pertenezca a otro shard sin un protocolo de relocation explícito.

---

# 133. Cross-shard move

Será:

```text
Distributed Row Relocation
```

no simple Bulk Update.

---

# 134. Reject default

Si se detecta cambio de partition/shard ownership:

```text
BulkUpdateShardMoveRequiredException
```

---

# 135. Tenant key mutation

Igualmente:

```text
tenant_id A → tenant_id B
```

no deberá convertirse en movimiento cross-tenant normal.

---

# 136. Tenant isolation

Debe considerarse una operación altamente sensible y normalmente prohibida.

---

# 137. Transaction policies

Se reutilizará la infraestructura común:

```php
enum BulkTransactionPolicy
{
    case NONE;
    case USE_EXISTING;
    case WHOLE_OPERATION;
    case PER_BATCH;
    case CUSTOM;
}
```

---

# 138. Set-based single statement

Una operación set-based puede ser:

```text
one statement
```

y aun así no estar dentro de una transacción explícita.

El motor DB puede garantizar atomicidad del statement, pero eso es distinto de una operación multi-batch.

---

# 139. Statement atomicity

No deberá confundirse:

```text
Atomic Statement
```

con:

```text
Atomic Multi-Batch Operation
```

---

# 140. Heterogeneous updates

Para múltiples batches:

```text
PER_BATCH
```

puede producir:

```text
PARTIAL
```

---

# 141. WHOLE_OPERATION

Solo cuando todos los batches pertenecen al mismo transactional domain.

---

# 142. Existing transaction

Nunca committear transacción ajena.

---

# 143. Savepoints

Podrán utilizarse mediante Transaction System cuando una policy lo requiera.

No son transacciones independientes.

---

# 144. Failure model

Distinguir:

```text
VALIDATION_FAILURE
PLANNING_FAILURE
INPUT_FAILURE
ROUTING_FAILURE
COMPILATION_FAILURE
EXECUTION_FAILURE
OPTIMISTIC_CONFLICT
TRANSACTION_FAILURE
COMMIT_UNKNOWN
RETURNING_FAILURE
ORM_COHERENCE_FAILURE
CANCELLATION
```

---

# 145. ORM coherence failure after DB commit

Caso importante:

```text
DB commit confirmed
↓
attempt ORM refresh fails
```

La DB ya cambió.

Resultado DB:

```text
SUCCEEDED
```

pero coherence action:

```text
FAILED
```

---

# 146. No rollback fiction

No deberá reportarse:

```text
Bulk Update failed and rolled back
```

si la falla ocurrió después de commit.

---

# 147. Result dimensions

El resultado podrá separar:

```text
DatabaseOutcome
OrmCoherenceOutcome
CacheOutcome
EventPublicationOutcome
```

---

# 148. BulkUpdateResult

```php
final readonly class BulkUpdateResult
{
    public function __construct(
        public BulkUpdateStatus $status,
        public BulkUpdateCountResult $counts,
        public array $batchResults,
        public BulkReturningResult $returning,
        public BulkOrmCoherenceResult $ormCoherence,
        public BulkUpdateOutcomeEvidence $evidence,
    ) {}
}
```

---

# 149. BulkUpdateStatus

```php
enum BulkUpdateStatus
{
    case SUCCEEDED;
    case PARTIAL;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 150. Unknown commit

Siempre:

```text
COMMIT UNKNOWN
→
operation UNKNOWN
```

si el commit define el outcome final.

---

# 151. Retry

Una mutación update puede no ser idempotente.

Ejemplo:

```text
counter = counter + 1
```

---

# 152. Retry danger

Reintentar después de outcome desconocido podría producir:

```text
+2
```

en lugar de:

```text
+1
```

---

# 153. Assignment replay classes

Podrán clasificarse:

```text
SET_CONSTANT
SET_DETERMINISTIC_VALUE
INCREMENT
DECREMENT
NONDETERMINISTIC_EXPRESSION
CUSTOM
```

---

# 154. Replay safety

Ejemplos:

```text
SET status = active
```

puede ser más replay-safe que:

```text
SET attempts = attempts + 1
```

pero aún depende de predicates, triggers y side effects.

---

# 155. No global assumption

```text
SET_CONSTANT
```

no implica automáticamente idempotencia de toda la operación.

---

# 156. Retry policy

```php
enum BulkUpdateRetryPolicy
{
    case NEVER;
    case TRANSACTION_SAFE;
    case IDEMPOTENT_ONLY;
    case CUSTOM;
}
```

---

# 157. Deadlocks

Se reutilizará:

```text
171_DATABASE_DEADLOCK_HANDLING_SYSTEM
```

---

# 158. Whole transaction retry

Cuando sea replay-safe, el Transaction System podrá reintentar la unidad transaccional completa.

---

# 159. No statement retry in aborted transaction

Regla existente permanece.

---

# 160. Concurrency

Bulk Update deberá coexistir con:

```text
optimistic locking
pessimistic locking
concurrent writes
replica readers
external writers
```

---

# 161. Lost updates

Un set-based update:

```text
SET price = 100
```

puede sobrescribir un cambio concurrente.

Si esto no es aceptable, deberá usarse:

```text
version predicate
locking
more precise condition
```

---

# 162. Compare-and-set

Podrá expresarse:

```text
WHERE status = pending
SET status = processing
```

como una transición condicional.

---

# 163. Claimed rows

El número de rows affected puede servir como evidencia de claim, sujeto a affected-row semantics del motor.

---

# 164. Work claiming

Para workflows de workers puede ser más apropiado:

```text
locking / SKIP LOCKED
```

que Bulk Update simple.

---

# 165. Cache invalidation

Toda mutación Bulk Update deberá participar en:

```text
191_DATABASE_CACHE_INVALIDATION_SYSTEM
```

---

# 166. Semantic Change

Podrá emitirse:

```text
EntityFieldsBulkChanged
TableRowsChanged
RelationshipKeysChanged
SoftDeleteStateChanged
```

dependiendo de metadata disponible.

---

# 167. Invalidation precision

Para:

```text
id IN [10,20,30]
```

puede haber precisión:

```text
ENTITY_EXACT / KEY_SET
```

---

# 168. Predicate update

Para:

```text
WHERE updated_at < X
```

puede requerirse:

```text
PREDICATE_BOUNDED
TABLE_WIDE
```

---

# 169. Unknown impact

Si no puede determinarse con suficiente precisión:

```text
UNKNOWN
→ broader invalidation
```

---

# 170. No invalidation per row obligatoria

Para millones de filas se permitirá coalescing.

---

# 171. Result Cache

Consultas cuyos resultados dependan de campos modificados deberán invalidarse según dependency metadata.

---

# 172. Entity Cache

Entradas de entidades afectadas deberán invalidarse/actualizarse solo tras commit confirmado.

---

# 173. Relationship Cache

Si cambia una FK o membership representation, deberán invalidarse regiones de relación correspondientes.

---

# 174. UNKNOWN commit

Aplicará invalidación conservadora.

---

# 175. Cache failure after commit

No puede deshacer el Bulk Update.

---

# 176. Event ordering

Eventos pre-execution no deberán presentarse como outcome final.

---

# 177. afterCommit

Side effects externos derivados de la actualización deberán preferir:

```text
afterCommit
outbox
queue
```

---

# 178. Query cache

Un Bulk Update no requiere invalidar estructuras de Query Cache salvo que cambie metadata/schema, lo cual normalmente no ocurre.

---

# 179. Metadata cache

Tampoco se invalida por cambios de datos normales.

---

# 180. Cancellation

`BulkUpdateRunner` deberá aceptar `CancellationToken`.

---

# 181. Cancellation between batches

Con `PER_BATCH`, batches anteriores pueden estar committed.

---

# 182. Cancellation within single set update

La capacidad de cancelar statement depende del Execution Engine/driver.

---

# 183. Cancellation ≠ rollback

Siempre:

```text
Cancel Requested
≠
Database Rolled Back
```

---

# 184. Resource governance

Deberá controlar:

```text
input rows
batch size
parameter count
statement size
duration
memory
concurrency
affected-row risk
returning result size
```

---

# 185. Returning explosion

Solicitar:

```text
RETURNING *
```

para 50 millones de filas puede destruir la ventaja de una operación bulk.

---

# 186. Returning policy

Podrá limitar:

```text
maxReturningRows
maxReturningBytes
allowFullRows
```

---

# 187. Result sink

Para grandes `RETURNING`, podrá utilizarse:

```text
BulkReturningSink
```

en vez de materializar todo.

---

# 188. Resource budget

```php
final readonly class BulkUpdateResourceBudget
{
    public function __construct(
        public ?int $maxInputRows,
        public ?int $maxAffectedRows,
        public ?int $maxBatchRows,
        public ?int $maxParameters,
        public ?int $maxReturningRows,
        public ?int $maxDurationMs,
        public ?int $maxMemoryBytes,
    ) {}
}
```

---

# 189. maxAffectedRows

Para set-based updates puede utilizarse como safety guard.

---

# 190. Hard affected-row guard

No siempre puede comprobarse sin ejecutar.

Una estrategia puede:

```text
pre-count
```

pero introduce race.

---

# 191. Pre-count ≠ execution guarantee

Entre:

```text
COUNT
```

y:

```text
UPDATE
```

el conjunto puede cambiar.

---

# 192. Strong guard

Para garantizar límite exacto puede requerirse una estrategia distinta:

```text
key selection
locking/snapshot
then keyed update
```

---

# 193. Safety planner

VoltStack deberá distinguir:

```text
ESTIMATED_GUARD
BEST_EFFORT_GUARD
ENFORCED_GUARD
```

---

# 194. Persistent runtimes

En FrankenPHP:

```text
Bulk Update A
Bulk Update B
```

deberán tener separados:

```text
current input batch
transaction
tenant
shard
returning collector
ORM coherence state
cancellation
```

---

# 195. No static mutation state

Nunca:

```php
BulkUpdateRunner::$currentKeys
```

---

# 196. RoadRunner/OpenSwoole

Misma arquitectura.

---

# 197. Concurrent coroutines

Cada operación tendrá su propio:

```text
BulkOperationContext
```

---

# 198. Shared immutable infrastructure

Podrá compartirse:

```text
metadata
platform capabilities
stateless planners
compiled mapping
connection pools
```

---

# 199. Telemetry

Eventos conceptuales:

```text
BulkUpdatePlanned
BulkUpdateStarted
BulkUpdateBatchPlanned
BulkUpdateBatchStarted
BulkUpdateBatchCompleted
BulkUpdateConflictDetected
BulkUpdateCommitted
BulkUpdateOrmStateMarkedStale
BulkUpdateCacheInvalidationScheduled
BulkUpdateCompleted
BulkUpdatePartial
BulkUpdateCancelled
BulkUpdateOutcomeUnknown
BulkUpdateFailed
```

---

# 200. Métricas

```text
db.bulk_update.operations
db.bulk_update.duration
db.bulk_update.batches
db.bulk_update.input_rows
db.bulk_update.rows_reported
db.bulk_update.conflicts
db.bulk_update.partial
db.bulk_update.unknown
db.bulk_update.orm_stale
```

---

# 201. Cardinalidad

No usar como labels:

```text
primary keys
tenant IDs
raw predicates
raw SQL
assignment values
cursor/checkpoint state
```

---

# 202. Diagnostics

API conceptual:

```php
DB::bulk()->explainUpdate(
    query: DB::table('users')
        ->where('status', 'pending'),
    assignments: [
        'status' => 'active',
    ],
);
```

---

# 203. Explain output

```text
BULK UPDATE PLAN

Mode:
    SET_BASED

Target:
    users

Predicate:
    status = :status

Assignments:
    status = active

Estimated Impact:
    2,450,000 rows

Execution:
    SINGLE SET-BASED UPDATE

Transaction:
    USE EXISTING / STATEMENT ATOMICITY

Optimistic Locking:
    NONE

Returning:
    NONE

Routing:
    WRITER
    SINGLE SHARD

ORM Coherence:
    MARK ENTITY TYPE POTENTIALLY STALE

Relationship Impact:
    none detected

Cache Invalidation:
    PREDICATE_BOUNDED → TABLE REGION

Retry Safety:
    CONDITIONAL

Safety:
    HIGH IMPACT WARNING
```

---

# 204. Explain keyed update

```text
BULK UPDATE PLAN

Mode:
    KEYED_HETEROGENEOUS

Input:
    STREAMING

Key:
    id

Key Uniqueness:
    VERIFIED

Rows:
    estimated 1,000,000

Mutation Shapes:
    3

Execution Strategy:
    VALUES_RELATION

Batch Size Requested:
    2000

Effective Batch Size:
    750

Limiting Factor:
    PARAMETER CAPACITY

Transaction:
    PER_BATCH

Returning:
    id, version

Optimistic Locking:
    PER ROW

Routing:
    GROUP BY SHARD

Global Atomicity:
    NOT PROVIDED

ORM:
    RETURNED MANAGED ENTITIES WILL NOT BE AUTO-OVERWRITTEN

Cache:
    KEY_SET INVALIDATION

Outcome Detail:
    PER_BATCH
```

---

# 205. BulkUpdatePlan

```php
final readonly class BulkUpdatePlan
{
    public function __construct(
        public BulkUpdateMode $mode,
        public UpdateTarget $target,
        public BulkSelectionPlan $selection,
        public BulkMutationPlan $mutation,
        public BulkUpdateExecutionPlan $execution,
        public BulkBatchPlan $batch,
        public BulkRoutingPlan $routing,
        public BulkTransactionPlan $transaction,
        public BulkOptimisticLockPlan $locking,
        public BulkReturningPlan $returning,
        public BulkOrmCoherencePlan $orm,
        public BulkResourcePlan $resources,
    ) {}
}
```

---

# 206. Directory structure

```text
src/Quantum/Database/Bulk/
│
├── Common/
│   ├── BulkOperationContext.php
│   ├── BulkTransactionPolicy.php
│   ├── BulkRoutingPlan.php
│   ├── BulkBatchPlan.php
│   └── BulkOperationStatus.php
│
└── Update/
    ├── BulkUpdateRequest.php
    ├── BulkUpdateOptions.php
    ├── BulkUpdateMode.php
    ├── BulkUpdatePlanner.php
    ├── BulkUpdatePlan.php
    ├── BulkUpdateRunner.php
    ├── BulkUpdateResult.php
    ├── BulkUpdateStatus.php
    │
    ├── Selection/
    │   ├── BulkSelectionPlan.php
    │   ├── BulkUpdateScopeSafety.php
    │   └── FullTableMutationIntent.php
    │
    ├── Mutation/
    │   ├── BulkUpdateMutation.php
    │   ├── BulkUpdateValue.php
    │   ├── BulkMutationPlan.php
    │   └── BulkMutationShape.php
    │
    ├── Keyed/
    │   ├── BulkUpdateRow.php
    │   ├── BulkUpdateRowSource.php
    │   ├── BulkUpdateKey.php
    │   ├── BulkUpdateShape.php
    │   ├── BulkUpdateShapeResolver.php
    │   └── BulkUpdateInputNormalizer.php
    │
    ├── Strategy/
    │   ├── BulkUpdateExecutionStrategy.php
    │   ├── BulkUpdateStrategyResolver.php
    │   ├── SetBasedUpdateStrategy.php
    │   ├── CaseUpdateStrategy.php
    │   ├── ValuesRelationUpdateStrategy.php
    │   └── TemporaryRelationUpdateStrategy.php
    │
    ├── Locking/
    │   ├── BulkOptimisticLockPlan.php
    │   ├── BulkOptimisticConflict.php
    │   └── BulkConcurrencyPolicy.php
    │
    ├── Count/
    │   ├── BulkUpdateCountResult.php
    │   └── BulkCountKnowledge.php
    │
    ├── Returning/
    │   ├── BulkReturningPlan.php
    │   ├── BulkReturningResult.php
    │   └── BulkReturningSink.php
    │
    ├── ORM/
    │   ├── BulkOrmCoherencePolicy.php
    │   ├── BulkOrmCoherencePlan.php
    │   ├── BulkOrmCoherenceResult.php
    │   └── BulkExternalMutationNotifier.php
    │
    ├── Transaction/
    │   └── BulkUpdateTransactionPlan.php
    │
    ├── Routing/
    │   ├── BulkUpdateRoutingPlanner.php
    │   └── BulkUpdateShardMoveDetector.php
    │
    ├── Retry/
    │   ├── BulkUpdateRetryPolicy.php
    │   └── BulkUpdateReplaySafety.php
    │
    ├── Resource/
    │   ├── BulkUpdateResourceBudget.php
    │   └── BulkUpdateResourcePlan.php
    │
    ├── Diagnostics/
    │   ├── BulkUpdateInspector.php
    │   └── BulkUpdateExplainer.php
    │
    ├── Telemetry/
    │   └── BulkUpdateTelemetry.php
    │
    └── Exception/
        └── ...
```

---

# 207. Error hierarchy

```text
DatabaseException
└── BulkOperationException
    └── BulkUpdateException
        ├── BulkUpdateInputException
        ├── BulkUpdateValidationException
        ├── BulkUpdatePlanningException
        ├── BulkUpdateScopeException
        ├── BulkUpdateShapeException
        ├── BulkUpdateKeyException
        ├── BulkUpdateExecutionException
        ├── BulkUpdateUnsupportedCapabilityException
        ├── BulkUpdateOptimisticConflictException
        ├── BulkUpdateReturningException
        ├── BulkUpdateRoutingException
        ├── BulkUpdateShardMoveRequiredException
        ├── BulkUpdateTransactionException
        ├── BulkUpdateRetryUnsafeException
        ├── BulkUpdateOrmCoherenceException
        ├── BulkUpdateResourceException
        └── BulkUpdateOutcomeUnknownException
```

---

# 208. Testing matrix

| Área | Caso |
|---|---|
| Set-based | simple predicate |
| Set-based | full table blocked |
| Set-based | explicit all rows |
| Keyed | single key |
| Keyed | composite key |
| Keyed | non-unique key |
| Input | array |
| Input | generator |
| Input | LazyCollection |
| Shape | uniform |
| Shape | heterogeneous |
| Values | NULL |
| Values | increment |
| Values | enum |
| Values | JSON |
| Values | datetime |
| Strategy | set based |
| Strategy | CASE |
| Strategy | VALUES relation |
| Strategy | temp relation |
| Batch | parameter limit |
| Returning | supported |
| Returning | unsupported |
| Count | matched vs changed |
| Lock | optimistic success |
| Lock | conflict |
| Lock | missing vs conflict unknown |
| ORM | managed stale |
| ORM | dirty managed entity |
| ORM | known-key eviction |
| Relationship | FK update |
| Soft delete | deleted_at update |
| Transaction | none |
| Transaction | existing |
| Transaction | whole |
| Transaction | per batch |
| Retry | deterministic SET |
| Retry | increment unsafe |
| Failure | middle batch |
| Failure | unknown commit |
| Routing | writer |
| Sharding | one shard |
| Sharding | multi-shard |
| Sharding | shard-key mutation |
| Tenant | isolation |
| Cache | key invalidation |
| Cache | broad invalidation |
| Runtime | worker reuse |
| Cancellation | between batches |
| Resource | returning explosion |

---

# 209. Architectural invariants

## DB-BUPD-001
Bulk Update será distinto de Entity save loop.

## DB-BUPD-002
Bulk Update será distinto de UnitOfWork change tracking.

## DB-BUPD-003
Bulk Update será distinto de Batch Persistence.

## DB-BUPD-004
Bulk Update será distinto de Upsert.

## DB-BUPD-005
Bulk Update será distinto de Bulk Insert.

## DB-BUPD-006
Bulk Update será distinto de Bulk Delete.

## DB-BUPD-007
Bulk Update será distinto de Migration.

## DB-BUPD-008
Bulk Update modificará registros directamente.

## DB-BUPD-009
Bulk Update no requerirá cargar entidades.

## DB-BUPD-010
Bulk Update no generará SQL directamente.

## DB-BUPD-011
Query Engine continuará siendo autoridad semántica de queries.

## DB-BUPD-012
Compiler continuará siendo autoridad SQL.

## DB-BUPD-013
Executor continuará siendo autoridad de ejecución.

## DB-BUPD-014
Driver continuará siendo autoridad protocol-level.

## DB-BUPD-015
Set-based y keyed update serán modelos distintos.

## DB-BUPD-016
Set-based update compartirá una mutation común.

## DB-BUPD-017
Keyed update permitirá mutaciones heterogéneas.

## DB-BUPD-018
Mutation Model será tipado.

## DB-BUPD-019
Update values usarán Query Expression System.

## DB-BUPD-020
Bulk Update no parseará SQL arbitrario como expresión normal.

## DB-BUPD-021
Type System resolverá conversiones.

## DB-BUPD-022
Enum mapping será respetado.

## DB-BUPD-023
JSON mapping será respetado.

## DB-BUPD-024
Temporal mapping será respetado.

## DB-BUPD-025
Value Object mapping será respetado.

## DB-BUPD-026
Custom types serán respetados.

## DB-BUPD-027
Missing mutation field será distinto de NULL.

## DB-BUPD-028
Missing field significará no cambiar por default.

## DB-BUPD-029
Mutation shapes serán explícitas.

## DB-BUPD-030
Shape grouping será permitido solo bajo policy segura.

## DB-BUPD-031
Input order/correlation será preservable.

## DB-BUPD-032
Composite keys serán soportables.

## DB-BUPD-033
Update key no tendrá que ser PK necesariamente.

## DB-BUPD-034
Non-unique update key será rechazado por default.

## DB-BUPD-035
Key uniqueness se validará con metadata cuando sea posible.

## DB-BUPD-036
Physical update strategy será capability-driven.

## DB-BUPD-037
CASE update será estrategia física, no semántica.

## DB-BUPD-038
VALUES relation podrá ser estrategia física.

## DB-BUPD-039
Temporary relation será estrategia opcional.

## DB-BUPD-040
Native bulk update será extensión especializada.

## DB-BUPD-041
Strategy resolver no contendrá vendor conditionals como arquitectura central.

## DB-BUPD-042
Version será distinta de Capability.

## DB-BUPD-043
Batch size respetará parameter limits.

## DB-BUPD-044
Batch size respetará statement limits.

## DB-BUPD-045
Batch size respetará resource budgets.

## DB-BUPD-046
Input podrá ser streaming.

## DB-BUPD-047
Bulk Update no requerirá materializar todo el input.

## DB-BUPD-048
Backpressure será preservable.

## DB-BUPD-049
Infinite input será gobernable por budgets/deadline/cancellation.

## DB-BUPD-050
Empty row mutation podrá tratarse como no-op según policy.

## DB-BUPD-051
Bulk Update no realizará pre-read obligatorio para detectar no-op.

## DB-BUPD-052
Rows reported serán distintas de rows matched.

## DB-BUPD-053
Rows reported serán distintas de rows changed.

## DB-BUPD-054
Counts no serán inventados.

## DB-BUPD-055
UNKNOWN count permanecerá UNKNOWN.

## DB-BUPD-056
RETURNING será capability-driven.

## DB-BUPD-057
RETURNING no implicará ORM synchronization.

## DB-BUPD-058
Generated/computed values podrán recuperarse cuando plataforma lo permita.

## DB-BUPD-059
Optimistic locking reutilizará subsystem existente.

## DB-BUPD-060
Per-row expected versions serán soportables.

## DB-BUPD-061
Zero affected no significará automáticamente version conflict.

## DB-BUPD-062
NOT_FOUND y CONFLICT permanecerán distintos cuando exista evidencia.

## DB-BUPD-063
Pre-read no se considerará equivalente a atomic conflict detection.

## DB-BUPD-064
Pessimistic locking será integración, no default implícito.

## DB-BUPD-065
Bulk Update sin predicate podrá ser bloqueado por policy.

## DB-BUPD-066
Full-table mutation requerirá intención explícita bajo default seguro.

## DB-BUPD-067
WHERE TRUE no sustituirá full-table acknowledgement.

## DB-BUPD-068
Estimated affected rows podrán alimentar safety policy.

## DB-BUPD-069
Core Database no mostrará prompts interactivos.

## DB-BUPD-070
Join update semantics serán platform-neutral a nivel lógico.

## DB-BUPD-071
Platform incapaz deberá reportar UNSUPPORTED.

## DB-BUPD-072
Subqueries serán validadas por Query Semantic System.

## DB-BUPD-073
Bulk Update podrá volver stale el IdentityMap.

## DB-BUPD-074
Database state no será igual a managed state después de external bulk mutation.

## DB-BUPD-075
ORM coherence policy será explícita.

## DB-BUPD-076
Default ORM policy podrá marcar tipos afectados como potencialmente stale.

## DB-BUPD-077
Bulk Update no hará EntityManager::clear() silenciosamente.

## DB-BUPD-078
Bulk Update no sobrescribirá managed dirty objects automáticamente.

## DB-BUPD-079
Bulk Update no reescribirá UoW snapshots silenciosamente.

## DB-BUPD-080
Known-key eviction será distinta de object deletion.

## DB-BUPD-081
Relationship fields afectados producirán invalidación de relationship assumptions.

## DB-BUPD-082
Bulk Update no realizará relationship graph fixup completo por default.

## DB-BUPD-083
Soft-delete semantics pertenecerán a Soft Delete System.

## DB-BUPD-084
Bulk Update no disparará ORM preUpdate/postUpdate por fila.

## DB-BUPD-085
Bulk-specific events serán distintos de entity lifecycle events.

## DB-BUPD-086
Domain invariants no serán ejecutados automáticamente.

## DB-BUPD-087
Bulk Update será API avanzada.

## DB-BUPD-088
Authorization scope será aplicado antes de mutación.

## DB-BUPD-089
Full-table acknowledgment no sustituirá authorization.

## DB-BUPD-090
Sensitive assignments no serán loggeados por default.

## DB-BUPD-091
Bulk Update será WRITE intent.

## DB-BUPD-092
Bulk Update no será enviado a read replica.

## DB-BUPD-093
Shard routing ocurrirá antes de ejecución.

## DB-BUPD-094
UNKNOWN shard no significará broadcast write.

## DB-BUPD-095
Multi-shard update será explícito.

## DB-BUPD-096
Multi-shard update no fingirá global ACID.

## DB-BUPD-097
Keyed input podrá agruparse por shard.

## DB-BUPD-098
Shard-key mutation cross-owner será rechazada por default.

## DB-BUPD-099
Cross-shard row movement será operación distinta.

## DB-BUPD-100
Tenant-key mutation cross-tenant será prohibida por default.

## DB-BUPD-101
Tenant isolation será obligatoria.

## DB-BUPD-102
Transaction policy será explícita.

## DB-BUPD-103
NONE no abrirá transaction.

## DB-BUPD-104
USE_EXISTING no committeará transaction ajena.

## DB-BUPD-105
WHOLE_OPERATION solo cerrará transaction propia.

## DB-BUPD-106
PER_BATCH podrá producir PARTIAL.

## DB-BUPD-107
Atomic statement será distinto de atomic multi-batch operation.

## DB-BUPD-108
Savepoint no será transaction independiente.

## DB-BUPD-109
Failure model distinguirá planning/execution/transaction/coherence.

## DB-BUPD-110
Post-commit ORM coherence failure no revertirá DB outcome.

## DB-BUPD-111
Result podrá separar DatabaseOutcome y OrmCoherenceOutcome.

## DB-BUPD-112
UNKNOWN commit permanecerá UNKNOWN.

## DB-BUPD-113
Increment mutation será potencialmente unsafe para replay.

## DB-BUPD-114
Retry safety se evaluará por operación completa.

## DB-BUPD-115
SET constant no implicará idempotencia universal.

## DB-BUPD-116
Deadlock retry utilizará Transaction System.

## DB-BUPD-117
Statement retry no ocurrirá dentro de transaction abortada.

## DB-BUPD-118
Lost update protection requerirá locking/versioning/predicate apropiado.

## DB-BUPD-119
Compare-and-set será expresable.

## DB-BUPD-120
Work claiming no será responsabilidad base de Bulk Update.

## DB-BUPD-121
Cache invalidation será semántica.

## DB-BUPD-122
Known key updates podrán invalidar key sets.

## DB-BUPD-123
Predicate updates podrán requerir invalidación más amplia.

## DB-BUPD-124
UNKNOWN impact producirá invalidación conservadora.

## DB-BUPD-125
No se requerirá evento de invalidación por fila.

## DB-BUPD-126
Entity Cache invalidation ocurrirá tras commit confirmado.

## DB-BUPD-127
Relationship cache invalidation será considerada cuando cambien FKs.

## DB-BUPD-128
UNKNOWN commit podrá requerir invalidación conservadora.

## DB-BUPD-129
Cache failure post-commit no revertirá DB.

## DB-BUPD-130
Bulk Update no invalidará Query/Metadata Cache por cambios de datos ordinarios.

## DB-BUPD-131
External side effects deberán preferir afterCommit/outbox.

## DB-BUPD-132
Cancellation será soportada.

## DB-BUPD-133
Cancellation no implicará rollback confirmado.

## DB-BUPD-134
Committed batches permanecerán committed tras cancellation.

## DB-BUPD-135
Resource Governance limitará batch size.

## DB-BUPD-136
Resource Governance limitará returning.

## DB-BUPD-137
RETURNING grande podrá usar sink streaming.

## DB-BUPD-138
maxAffectedRows tendrá semántica de evidencia explícita.

## DB-BUPD-139
Pre-count será best-effort salvo locking/snapshot adicional.

## DB-BUPD-140
Persistent runtime state será operation-scoped.

## DB-BUPD-141
No habrá current update batch static mutable.

## DB-BUPD-142
TransactionContext no se compartirá entre operaciones.

## DB-BUPD-143
TenantContext no se filtrará entre workers.

## DB-BUPD-144
Shard routing state no se filtrará entre workers.

## DB-BUPD-145
FrankenPHP worker reuse será seguro.

## DB-BUPD-146
RoadRunner worker reuse será seguro.

## DB-BUPD-147
OpenSwoole worker reuse será seguro.

## DB-BUPD-148
Concurrent coroutine operations estarán aisladas.

## DB-BUPD-149
Immutable metadata podrá compartirse.

## DB-BUPD-150
BulkUpdatePlan será immutable.

## DB-BUPD-151
Planner será distinto de Runner.

## DB-BUPD-152
Runner será distinto de Query Executor.

## DB-BUPD-153
Bulk Update no conocerá PDO.

## DB-BUPD-154
Bulk Update no conocerá runtime concreto.

## DB-BUPD-155
Bulk Update no creará un segundo ORM.

## DB-BUPD-156
Bulk Update no creará un segundo UnitOfWork.

## DB-BUPD-157
Bulk Update no creará un segundo Transaction System.

## DB-BUPD-158
Bulk Update no creará un segundo Cache System.

## DB-BUPD-159
Bulk Update no creará un segundo Routing System.

## DB-BUPD-160
Bulk Update será explainable.

## DB-BUPD-161
Affected-row semantics serán diagnosticables.

## DB-BUPD-162
ORM coherence consequences serán diagnosticables.

## DB-BUPD-163
Retry safety será diagnosticable.

## DB-BUPD-164
Transaction atomicity scope será diagnosticable.

## DB-BUPD-165
Shard movement requirement será diagnosticable.

## DB-BUPD-166
Cache invalidation precision será diagnosticable.

## DB-BUPD-167
Telemetry tendrá bounded cardinality.

## DB-BUPD-168
Raw predicates no serán metric labels.

## DB-BUPD-169
Sensitive assignment values no serán metric labels.

## DB-BUPD-170
Database continuará siendo autoridad del estado persistente.

---

# 210. Modelo formal — Set-Based Update

Sea:

```text
Q
=
target selection
```

y:

```text
M
=
mutation function
```

Entonces:

```text
BulkUpdate(Q,M)
```

intenta transformar cada fila seleccionada:

```text
r
→
M(r)
```

según las reglas del motor DB y transaction context.

---

# 211. Modelo formal — Keyed Update

Sea una secuencia:

```text
U =
[
    (K1, M1),
    (K2, M2),
    ...
    (Kn, Mn)
]
```

donde:

```text
Ki
=
lookup key

Mi
=
mutation for key Ki
```

El planner produce:

```text
Route
→ Shape Groups
→ Batches
→ Physical Update Plans
```

---

# 212. Shape partition

```text
U
→
G1 ∪ G2 ∪ ... ∪ Gm
```

donde cada grupo comparte:

```text
key shape
+
mutation column shape
+
routing domain
```

---

# 213. Outcome aggregation

Sea:

```text
O(Bi)
```

el outcome de cada batch.

Entonces:

```text
O(Operation)
=
Aggregate(
    O(B1),
    ...,
    O(Bn),
    TransactionPolicy
)
```

---

# 214. Transaction model

Si:

```text
WHOLE_OPERATION
+
single transactional domain
+
confirmed commit
```

podrá afirmarse atomicidad global de la operación dentro de ese dominio.

Si:

```text
PER_BATCH
```

entonces:

```text
AtomicityScope
=
Batch
```

no operación completa.

---

# 215. Optimistic concurrency model

Para una fila con versión:

```text
Update succeeds
iff
CurrentVersion
=
ExpectedVersion
```

y la actualización puede producir:

```text
NewVersion
=
VersionTransition(CurrentVersion)
```

según metadata.

---

# 216. ORM coherence model

Después de Bulk Update:

```text
DBState'
=
BulkUpdate(DBState)
```

pero:

```text
IdentityMapState'
```

no cambia automáticamente.

Por tanto:

```text
DBState'
≠
ManagedState
```

es una condición posible y explícita.

---

# 217. Cache model

Después de commit confirmado:

```text
SemanticChangeSet
→
InvalidationPlanner
→
Cache Invalidation
```

Nunca:

```text
BulkUpdate
→
direct FLUSHALL
```

---

# 218. Arquitectura final

```text
                    Bulk Update API
                           │
                           ▼
                   BulkUpdateRequest
                           │
                           ▼
                 Selection / Input
                    ┌──────┴──────┐
                    ▼             ▼
               Predicate      Keyed Rows
                    │             │
                    └──────┬──────┘
                           ▼
                 Mutation Normalizer
                           │
                           ▼
                  Type Resolution
                           │
                           ▼
                  BulkUpdatePlanner
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
       Strategy         Routing         Concurrency
          │                │                 │
          ├────────────────┼─────────────────┤
          ▼                ▼                 ▼
        Batch         Transaction        Returning
          │                │                 │
          └────────────────┼─────────────────┘
                           ▼
                    BulkUpdatePlan
                           │
                           ▼
                    BulkUpdateRunner
                           │
                           ▼
                   UpdateQueryModel
                           │
                           ▼
                      Query Engine
                           │
                           ▼
                        Compiler
                           │
                           ▼
                        Executor
                           │
                           ▼
                  Transaction Outcome
                           │
              ┌────────────┼─────────────┐
              ▼            ▼             ▼
          ORM Stale     Cache        Bulk Events
           Handling   Invalidation
              │            │             │
              └────────────┼─────────────┘
                           ▼
                   BulkUpdateResult
```

---

# 219. Regla maestra final

> **Bulk Update en VoltStack será una operación directa de mutación masiva sobre el estado persistente; optimizará conjuntos grandes sin afirmar que entidades, snapshots, relaciones o eventos ORM individuales fueron actualizados cuando el `UnitOfWork` nunca observó esas modificaciones.**

Siempre:

```text
Bulk Update
≠
Entity::save() × N
```

```text
Bulk Update
≠
UnitOfWork Change Tracking
```

```text
Database Updated
≠
IdentityMap Updated
```

```text
Rows Reported
≠
Rows Changed
```

```text
Zero Rows Updated
≠
Optimistic Conflict
```

```text
RETURNING
≠
Safe Managed-Object Overwrite
```

```text
Set-Based Statement Atomicity
≠
Multi-Batch Atomicity
```

```text
Cross-Shard Update
≠
Global ACID
```

```text
Shard-Key Update
≠
Row Relocation
```

y:

```text
UNKNOWN Commit
≠
Safe Retry
```

---

# 220. Resultado arquitectónico

VoltStack podrá ejecutar operaciones como:

```php
DB::table('notifications')
    ->where('read', false)
    ->where('created_at', '<', $threshold)
    ->updateBulk([
        'archived' => true,
    ]);
```

o:

```php
DB::table('products')->updateBulkByKey(
    rows: $productChanges,
    key: 'id',
    options: new BulkUpdateOptions(
        batchSize: 1000,
        transactionPolicy: BulkTransactionPolicy::PER_BATCH,
    ),
);
```

sobre:

```text
100 rows
100,000 rows
10,000,000 rows
```

sin cargar todas las entidades y preservando contratos explícitos sobre:

```text
types
selection
routing
batching
transactions
optimistic locking
returning
cache invalidation
ORM coherence
resource limits
outcomes
```

---

# 221. Relación con Bulk Delete

Con 203 y 204 tenemos:

```text
Bulk Insert
    creates persistent rows

Bulk Update
    mutates persistent rows
```

El siguiente sistema deberá resolver:

```text
Bulk Delete
    removes persistent rows
```

pero con riesgos adicionales relacionados con:

```text
foreign keys
cascades
soft deletes
orphan semantics
managed entities
relationship collections
cache tombstones
data retention
audit
```

---

# 222. Siguiente documento

```text
205_DATABASE_BULK_DELETE_SYSTEM.md
```

El siguiente documento definirá:

```text
Bulk Delete System
├── predicate deletes
├── keyed deletes
├── batch deletes
├── hard delete
├── soft-delete boundary
├── cascade semantics
├── FK constraints
├── returning deleted identifiers
├── optimistic version checks
├── ORM stale/removed state
├── IdentityMap implications
├── relationship invalidation
├── transaction policies
├── retries
├── sharding
├── tenant isolation
├── cache invalidation
├── tombstones
├── audit
├── resource governance
└── telemetry
```

estableciendo especialmente:

```text
Bulk Delete
≠
foreach EntityManager::remove()
```

```text
Bulk Delete
≠
Soft Delete
```

```text
Database Row Deleted
≠
PHP Object Destroyed
```

con la regla central:

> **Bulk Delete eliminará conjuntos de registros directamente bajo una intención de borrado explícita y gobernada, sin simular cascadas ORM, lifecycle events o transiciones `REMOVED` que el `UnitOfWork` nunca ejecutó.**