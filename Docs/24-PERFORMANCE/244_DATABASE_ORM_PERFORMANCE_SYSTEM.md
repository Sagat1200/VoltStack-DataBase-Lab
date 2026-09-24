# 244_DATABASE_ORM_PERFORMANCE_SYSTEM.md

# VoltStack Quantum Database
## Database ORM Performance System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 244 — Database ORM Performance System  
**Bloque:** 24 — Performance  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `243_DATABASE_QUERY_PERFORMANCE_SYSTEM.md`  
**Siguiente documento:** `245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **ORM Performance System** de VoltStack Database.

Su responsabilidad será medir, analizar, controlar y optimizar el costo introducido por la capa ORM:

```text
Application
    ↓
Model API / Repository
    ↓
EntityManager
    ↓
IdentityMap
    ↓
UnitOfWork
    ↓
Change Tracking
    ↓
Relationship Management
    ↓
Persistence Engine
    ↓
Query Engine
```

El objetivo no será simplemente hacer que el ORM "ejecute menos SQL".

VoltStack deberá observar también:

- entidades administradas;
- tamaño del IdentityMap;
- snapshots;
- ChangeSets;
- dirty checking;
- traversal de grafos;
- cascades;
- relaciones;
- lazy loading;
- eager loading;
- batch loading;
- operaciones de `flush()`;
- memoria;
- tiempo de CPU;
- queries generadas indirectamente;
- amplification;
- persistencia masiva;
- lifecycle callbacks;
- proxies/lazy references;
- persistent workers.

Regla central:

> **El rendimiento del ORM de VoltStack se optimizará reduciendo trabajo innecesario en identidad, seguimiento de cambios, relaciones, hidratación y persistencia sin debilitar las garantías semánticas del ORM.**

---

# 2. Principio fundamental

Una optimización ORM nunca deberá romper:

```text
Canonical Entity Identity
Entity State
UnitOfWork
ChangeSet Correctness
Relationship Semantics
Persistence Ordering
Transaction Semantics
Tenant Isolation
Shard Ownership
Lifecycle Contracts
```

Por tanto:

```text
Faster ORM
≠
Weaker ORM Correctness
```

---

# 3. Qué significa rendimiento ORM

El costo ORM puede expresarse conceptualmente como:

```text
Corm
=
Cmetadata
+
Cidentity
+
Chydration
+
Ctracking
+
Crelationships
+
Cflush
+
Cpersistence
+
Clifecycle
+
Cmemory
```

donde:

```text
Cmetadata      = metadata resolution
Cidentity      = IdentityMap operations
Chydration     = entity materialization
Ctracking      = change detection
Crelationships = relationship management/loading
Cflush         = UnitOfWork synchronization
Cpersistence   = persistence planning
Clifecycle     = lifecycle callbacks/events
Cmemory        = retained ORM state
```

---

# 4. ORM Performance ≠ Query Performance

El documento 243 cubre:

```text
Query Builder
AST
Semantic Analysis
Optimizer
Planner
Compiler
Executor
```

Este documento cubre:

```text
EntityManager
IdentityMap
UnitOfWork
Entity State
Change Tracking
Relationships
Persistence
```

Por tanto:

```text
Query Performance
≠
ORM Performance
```

aunque ambos sistemas estarán correlacionados.

---

# 5. ORM Performance ≠ Hydration Performance

La hidratación tendrá arquitectura específica en:

```text
245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md
```

Este documento únicamente define su interacción con el ORM.

---

# 6. Arquitectura general

```text
                    ORM API
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
      Model API                 Repository
          │                         │
          └────────────┬────────────┘
                       ▼
                 EntityManager
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     IdentityMap    UnitOfWork   Relationships
          │            │            │
          │            ▼            │
          │      Change Tracking    │
          │            │            │
          └────────────┼────────────┘
                       ▼
              Persistence Engine
                       │
                       ▼
                  Query Engine
                       │
                       ▼
                   Database
```

ORM Performance observará todas estas fases.

---

# 7. Responsabilidades

El sistema deberá poder responder preguntas como:

```text
¿Cuántas entidades están managed?

¿Cuánto ocupa el IdentityMap?

¿Cuántos snapshots existen?

¿Cuántas entidades se inspeccionaron durante flush?

¿Cuántas realmente cambiaron?

¿Cuántos ChangeSets se generaron?

¿Cuántas relaciones fueron recorridas?

¿Cuántos lazy loads ocurrieron?

¿Cuántos eager loads se ejecutaron?

¿Cuántas queries produjo una operación ORM?

¿Cuántos INSERT/UPDATE/DELETE fueron planificados?

¿Cuánto tardó flush?

¿Cuánta memoria retuvo EntityManager?

¿Cuántas entidades permanecieron managed después del batch?
```

---

# 8. ORMPerformanceContext

```php
final readonly class ORMPerformanceContext
{
    public function __construct(
        public ORMOperationId $operationId,
        public DatabaseContext $database,
        public ORMPerformancePolicy $policy,
        public ORMPerformanceBudget $budget,
        public ORMPerformanceMeasurementMode $measurementMode,
    ) {}
}
```

---

# 9. Scope

El contexto será:

```text
request scoped
operation scoped
job scoped
command scoped
```

según runtime.

Nunca:

```php
ORMPerformance::$current;
```

---

# 10. ORMOperationId

Cada operación ORM relevante tendrá identidad.

Ejemplo:

```text
orm:01H...
```

Esto permitirá correlacionar:

```text
ORM Operation
    ↓
Query Operations
    ↓
Connection Operations
    ↓
Transaction
```

---

# 11. ORM operation

Una operación lógica podría ser:

```php
$user = $repository->find(10);
```

o:

```php
$entityManager->flush();
```

o:

```php
$user->orders()->load();
```

---

# 12. Logical ORM operation ≠ physical query

Ejemplo:

```text
flush()
```

puede producir:

```text
5 INSERT
3 UPDATE
2 DELETE
```

Entonces:

```text
ORM Operations = 1
Physical Queries = 10
```

---

# 13. ORMPerformanceBudget

```php
final readonly class ORMPerformanceBudget
{
    public function __construct(
        public ?Duration $maxOperationDuration = null,
        public ?Duration $maxFlushDuration = null,
        public ?int $maxManagedEntities = null,
        public ?int $maxSnapshots = null,
        public ?int $maxChangeSets = null,
        public ?int $maxRelationshipLoads = null,
        public ?int $maxPhysicalQueries = null,
        public ?int $maxIdentityMapEntries = null,
        public ?int $maxMemoryBytes = null,
    ) {}
}
```

---

# 14. Budget ≠ semantic limit

Si:

```text
maxManagedEntities = 10,000
```

y existen:

```text
10,001
```

no significa necesariamente que la operación deba abortar.

Dependiendo de policy:

```text
WARN
PROFILE
REJECT
```

---

# 15. Measurement modes

```php
enum ORMPerformanceMeasurementMode
{
    case OFF;
    case MINIMAL;
    case STANDARD;
    case DETAILED;
    case PROFILE;
}
```

---

# 16. OFF

Debe tener overhead mínimo.

---

# 17. MINIMAL

Puede medir:

```text
operation
duration
entity count
query count
outcome
```

---

# 18. STANDARD

Puede agregar:

```text
IdentityMap size
UnitOfWork size
ChangeSets
relationship loads
flush statistics
```

---

# 19. DETAILED

Puede agregar:

```text
entity-type breakdown
relationship breakdown
snapshot counts
cascade traversal
memory
persistence plan statistics
```

---

# 20. PROFILE

Puede habilitar instrumentación costosa para desarrollo y benchmarking.

No será production default.

---

# 21. EntityManager performance

`EntityManager` es el coordinador principal del ORM.

Pero:

```text
EntityManager
≠
Performance Analyzer
```

Solo emitirá observaciones.

---

# 22. EntityManager lifecycle

Estados ya definidos:

```text
OPEN
TAINTED
CLOSED
```

Performance no cambiará esos estados.

---

# 23. EntityManager scope size

Uno de los indicadores fundamentales será:

```text
Managed Entity Count
```

---

# 24. Managed entities

Para un EntityManager:

```text
M = |IdentityMap|
```

Este número puede crecer durante:

```text
imports
exports
batch processing
long-running jobs
lazy iteration
large queries
```

---

# 25. Managed entity growth

Ejemplo:

```text
Chunk 1 → 1,000 managed
Chunk 2 → 2,000 managed
Chunk 3 → 3,000 managed
...
Chunk 100 → 100,000 managed
```

aunque cada chunk individual contenga solo 1,000 registros.

---

# 26. Bounded chunk ≠ bounded ORM memory

Regla:

```text
Chunk Size = 1,000
```

no implica:

```text
ORM Memory = O(1,000)
```

si el IdentityMap sigue reteniendo entidades.

---

# 27. EntityManager growth detector

Podrá existir:

```php
final class EntityManagerGrowthDetector
{
    public function analyze(
        ORMPerformanceMeasurement $measurement
    ): iterable;
}
```

---

# 28. Finding

Ejemplo:

```text
UNBOUNDED_ENTITY_MANAGER_GROWTH
```

---

# 29. IdentityMap performance

IdentityMap garantiza:

```text
(EntityType + Identifier + Effective Context)
→
Canonical Managed Instance
```

---

# 30. IdentityMap lookup

La operación ideal será aproximadamente:

```text
O(1)
```

promedio mediante estructura hash apropiada.

Pero esto no será una garantía formal universal.

---

# 31. Identity key

La clave deberá ser canónica.

Ejemplo conceptual:

```text
EntityType
+
IdentifierTuple
+
Tenant
+
DatabaseDomain
+
Shard
```

según contexto.

---

# 32. Composite identifiers

Para:

```text
(country_id, customer_id)
```

la canonicalización deberá evitar allocations innecesarias cuando sea posible.

---

# 33. IdentityMap metrics

Podrán incluir:

```text
lookups
hits
misses
registrations
removals
peak entries
current entries
```

---

# 34. IdentityMap hit ratio

Conceptualmente:

```text
HitRatio
=
Hits / Lookups
```

pero su interpretación dependerá del workload.

---

# 35. Low hit ratio ≠ bad

En una carga de entidades completamente nuevas:

```text
IdentityMap hit ratio
```

puede ser bajo y ser correcto.

---

# 36. IdentityMap memory

El costo no será únicamente:

```text
entity object
```

También:

```text
identity key
metadata references
snapshots
relationship collections
UnitOfWork state
```

---

# 37. IdentityMap ≠ Entity Cache

Continúa vigente:

```text
IdentityMap
≠
Second-Level Entity Cache
```

No deberá sustituirse IdentityMap por cache compartido para ahorrar memoria.

---

# 38. IdentityMap cannot be globally shared

Especialmente en:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

cada scope necesita identidad propia.

---

# 39. Entity registration

Registrar una entidad deberá ser suficientemente eficiente para cargas masivas.

---

# 40. Duplicate registration

Si la identidad ya existe:

```text
existing managed instance
```

deberá reutilizarse o aplicarse la policy correspondiente.

Nunca deberán coexistir silenciosamente dos managed instances de la misma identidad.

---

# 41. Identity collision

Performance no justificará omitir validaciones de identidad.

---

# 42. UnitOfWork performance

UnitOfWork mantiene:

```text
new entities
managed entities
dirty entities
removed entities
relationship changes
snapshots
```

---

# 43. UoW size

Definimos:

```text
U
=
Nnew
+
Ndirty
+
Nremoved
+
NrelationshipChanges
```

como una aproximación operativa.

---

# 44. UoW ≠ IdentityMap

Una entidad puede estar:

```text
MANAGED
```

sin tener cambios pendientes.

---

# 45. UnitOfWork metrics

```text
managed entities inspected
new entities
dirty entities
removed entities
relationship changes
change sets
cascade edges
persistence operations
```

---

# 46. Flush performance

`flush()` será uno de los puntos más importantes.

Pipeline conceptual:

```text
flush()
   ↓
discover pending state
   ↓
change detection
   ↓
relationship synchronization
   ↓
dependency graph
   ↓
persistence planning
   ↓
query generation
   ↓
execution
   ↓
state reconciliation
```

---

# 47. Flush time

Conceptualmente:

```text
Tflush
=
Tdetect
+
Trelations
+
Tgraph
+
Tplan
+
Tqueries
+
Treconcile
```

---

# 48. Flush ≠ commit

Continúa vigente:

```text
flush()
≠
transaction commit
```

Performance metrics deberán separarlos.

---

# 49. Flush measurement

```php
final readonly class FlushPerformanceStatistics
{
    public function __construct(
        public int $managedEntitiesInspected,
        public int $changeSetsGenerated,
        public int $inserts,
        public int $updates,
        public int $deletes,
        public int $relationshipOperations,
        public int $physicalQueries,
        public Duration $duration,
    ) {}
}
```

---

# 50. Flush efficiency

Una señal útil:

```text
FlushScanRatio
=
ManagedEntitiesInspected
/
ChangedEntities
```

---

# 51. Example

```text
Managed inspected: 100,000
Changed: 2
```

puede indicar un costo significativo de dirty checking.

---

# 52. Ratio ≠ universal problem

Puede ser resultado legítimo del tracking strategy.

El finding deberá considerar contexto.

---

# 53. Change Tracking performance

VoltStack soportará estrategias explícitas de seguimiento.

Conceptualmente:

```text
SNAPSHOT
EXPLICIT
NOTIFY
READ_ONLY
CUSTOM
```

según evolución del ORM.

---

# 54. Snapshot tracking

Modelo:

```text
Hydrated State
      ↓
Snapshot
      ↓
Current State
      ↓
Comparison
      ↓
ChangeSet
```

---

# 55. Snapshot cost

Puede consumir:

```text
memory
copying
comparison CPU
type conversion
```

---

# 56. Snapshot width

Una entidad con:

```text
80 mapped fields
```

puede ser más costosa que una con:

```text
5 mapped fields
```

---

# 57. Snapshot ≠ entity clone

No deberá requerirse necesariamente clonar el objeto completo.

---

# 58. Compact snapshots

Podrán almacenarse:

```text
canonical persistent values
```

en una estructura compacta.

---

# 59. Snapshot memory

Conceptualmente:

```text
SnapshotMemory
≈
Σ snapshot(entity_i)
```

---

# 60. Snapshot sharing

Valores inmutables podrán reutilizar referencias cuando sea seguro.

---

# 61. Mutable values

No deberán compartirse de forma que una mutación cambie simultáneamente:

```text
current state
+
snapshot state
```

porque invalidaría dirty checking.

---

# 62. Dirty checking

En snapshot strategy, costo aproximado:

```text
O(E × F)
```

donde:

```text
E = entities inspected
F = mapped fields inspected
```

---

# 63. Dirty checking optimization

Podrán emplearse:

```text
dirty candidate sets
field masks
change notifications
read-only entities
compiled accessors
compiled metadata
```

---

# 64. Dirty candidate ≠ dirty entity

Una entidad marcada como posible candidata todavía debe verificarse cuando la estrategia lo requiera.

---

# 65. Explicit tracking

Una estrategia explícita podría reducir scanning.

Pero cambia el contrato del programador.

No será activada silenciosamente.

---

# 66. Read-only entities

VoltStack podrá soportar:

```text
READ_ONLY
```

para consultas donde no se requiere persistencia posterior.

---

# 67. Read-only benefit

Puede evitar:

```text
snapshots
dirty checking
UnitOfWork bookkeeping
```

según diseño.

---

# 68. Read-only ≠ immutable domain object

Significa:

```text
ORM will not track persistence changes
```

no necesariamente que PHP impida modificar el objeto.

---

# 69. Read-only mutation

Si el desarrollador modifica una entidad read-only:

```text
no automatic persistence
```

deberá ser comportamiento explícito.

---

# 70. Read-only query

API conceptual:

```php
$users = User::query()
    ->readOnly()
    ->get();
```

---

# 71. Projection optimization

Cuando solo se necesitan:

```text
id
name
email
```

una projection/DTO puede ser preferible a managed entities completas.

---

# 72. Projection ≠ partial managed entity

Regla:

```text
Projection
≠
Partial Managed Entity
```

---

# 73. Partial entities

El ORM deberá evitar utilizar partial managed entities como optimización genérica.

---

# 74. Why

Porque generan complejidad en:

```text
dirty checking
snapshots
flush
lazy fields
identity
state correctness
```

---

# 75. Entity state performance

Estados:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

deberán almacenarse eficientemente.

---

# 76. Entity state lookup

Idealmente:

```text
object identity
→
state
```

sin recorrer colecciones completas.

---

# 77. Object identity maps

PHP puede utilizar estructuras adecuadas como:

```text
WeakMap
SplObjectStorage
```

cuando su semántica sea apropiada.

La implementación concreta deberá benchmarkearse.

---

# 78. Weak references caution

No deberán utilizarse si permiten que una entidad managed desaparezca antes de lo que el contrato ORM permite.

---

# 79. UnitOfWork indexing

El UoW podrá mantener índices internos separados:

```text
new
dirty
removed
```

para evitar scanning innecesario.

---

# 80. Duplicate state structures

Pero cada índice adicional consume memoria.

La decisión deberá benchmarkearse.

---

# 81. ChangeSet performance

`ChangeSet` representa:

```text
field
old value
new value
```

---

# 82. Empty ChangeSet

No deberá producir:

```text
UPDATE table SET ...
```

sin cambios, salvo semántica explícita.

---

# 83. ChangeSet compaction

Podrá almacenar únicamente campos modificados.

---

# 84. Value comparison

Debe usar:

```text
ORM Type semantics
```

no simple comparación PHP universal.

---

# 85. Expensive value comparison

Tipos como:

```text
JSON
large strings
binary
complex value objects
```

pueden tener costo mayor.

---

# 86. Type-aware optimization

Custom Type podrá proporcionar:

```php
interface ChangeComparisonStrategy
{
    public function equivalent(
        mixed $old,
        mixed $new,
    ): bool;
}
```

sin romper Type System.

---

# 87. Hash-assisted comparison

Para valores grandes e inmutables podría evaluarse fingerprint/hash.

Pero:

```text
hash equality
```

solo será utilizada según garantías adecuadas.

---

# 88. Relationship performance

Las relaciones pueden ser uno de los costos ORM más importantes.

---

# 89. Relationship operations

Incluyen:

```text
loading
collection initialization
collection diff
association synchronization
cascade persist
cascade remove
orphan detection
batch loading
eager loading
lazy loading
```

---

# 90. Relationship collection state

Una colección puede estar:

```text
UNINITIALIZED
PARTIAL
COMPLETE
```

---

# 91. Partial ≠ complete

Performance nunca deberá convertir:

```text
PARTIAL
```

en:

```text
COMPLETE
```

para evitar otra query.

---

# 92. Collection snapshots

Many-to-Many y One-to-Many pueden requerir snapshots de membresía.

---

# 93. Collection snapshot cost

Para una colección de tamaño `N`:

```text
memory ≈ O(N)
```

según estrategia.

---

# 94. Collection diff

Una implementación ingenua:

```text
O(N²)
```

deberá evitarse.

---

# 95. Identifier-based collection diff

Cuando sea semánticamente válido:

```text
Old IDs
New IDs
```

pueden compararse mediante sets/maps.

Aproximadamente:

```text
O(N)
```

promedio.

---

# 96. New entities in collection

No todas las entidades tienen identificador generado aún.

La estrategia deberá soportar:

```text
object identity
+
persistent identity when available
```

---

# 97. Relationship synchronization

Owning side seguirá siendo autoridad de persistencia.

Performance no cambiará esta regla.

---

# 98. Inverse-side optimization

No se generarán escrituras desde inverse side solo para "ahorrar traversal".

---

# 99. Lazy loading performance

Lazy loading permite evitar trabajo no utilizado.

Pero puede generar:

```text
N+1
```

---

# 100. Lazy loading metrics

```text
lazy loads
lazy loads by relationship
queries triggered
rows loaded
duration
```

---

# 101. Lazy load ≠ N+1

Un lazy load aislado puede ser correcto.

---

# 102. Repeated lazy loading

Patrones correlacionados serán analizados por:

```text
223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
```

---

# 103. Lazy loading policy

Continúan:

```text
ALLOW
WARN
FORBID
```

---

# 104. Performance system integration

Podrá aumentar severity cuando:

```text
WARN
+
high repeated lazy load count
```

---

# 105. Eager loading performance

Eager loading reduce round trips, pero puede incrementar:

```text
rows
network
memory
hydration
join amplification
```

---

# 106. Eager loading ≠ JOIN

Continúa vigente:

```text
Eager Requirement
≠
JOIN Strategy
```

---

# 107. Eager strategies

Planner puede elegir:

```text
JOIN
SELECT_IN
BATCH
```

según capacidades y contexto.

---

# 108. Strategy performance

Performance System observará:

```text
physical queries
rows transferred
root entities
related entities
join amplification
memory
```

---

# 109. Fetch amplification

Ejemplo:

```text
100 users
×
20 orders/user
×
5 items/order
```

puede producir:

```text
10,000 physical rows
```

aunque existan:

```text
100 root entities
```

---

# 110. ORM row amplification

Conceptualmente:

```text
ORMRowAmplification
=
PhysicalRowsHydrated
/
LogicalRootEntities
```

---

# 111. High amplification finding

```text
HIGH_ORM_ROW_AMPLIFICATION
```

---

# 112. Alternative

Podrá sugerirse:

```text
Evaluate SELECT_IN or batch loading.
```

No se cambiará automáticamente la estrategia sin planner policy.

---

# 113. Batch relationship loading

Permite agrupar:

```text
owners
→
single relationship query
```

---

# 114. Batch size

Debe estar acotado por:

```text
parameter limits
query size
memory
database behavior
```

---

# 115. Batch size ≠ universal constant

No existirá necesariamente:

```text
BATCH_SIZE = 1000
```

para todos los DBMS y workloads.

---

# 116. Relationship loading performance metrics

```text
owners
relationships requested
queries
rows
entities hydrated
batch count
average batch size
```

---

# 117. Cascade performance

Cascade puede recorrer grafos grandes.

---

# 118. Cascade graph

Ejemplo:

```text
Order
 ├── Items
 │    ├── Product
 │    └── Taxes
 ├── Payments
 └── Shipment
```

---

# 119. Cascade traversal

Debe detectar ciclos.

---

# 120. Visited set

Un conjunto de objetos visitados evitará traversal infinito.

---

# 121. Cascade complexity

Aproximadamente:

```text
O(V + E)
```

si:

```text
V = entities
E = relationship edges
```

y se utilizan estructuras adecuadas.

---

# 122. Cascade metrics

```text
entities visited
relationship edges traversed
cascade persist count
cascade remove count
maximum depth
```

---

# 123. Deep graph finding

```text
LARGE_PERSISTENCE_GRAPH
```

---

# 124. Cascade remove

Será especialmente sensible.

Una eliminación raíz podría producir miles de operaciones.

---

# 125. Database cascade

Cuando una FK utiliza:

```text
ON DELETE CASCADE
```

podrá reducir operaciones ORM.

Pero:

```text
Database Cascade
≠
ORM Cascade
```

---

# 126. Lifecycle callbacks

Callbacks pueden ejecutar lógica arbitraria.

Ejemplos:

```text
prePersist
postPersist
preUpdate
postUpdate
preRemove
postLoad
```

---

# 127. Callback performance

Deberá poder medirse en PROFILE mode.

---

# 128. Callback I/O

Lifecycle callback que ejecuta queries puede producir amplification.

---

# 129. Callback query correlation

QueryOperation deberá poder correlacionarse con:

```text
LifecycleCallbackId
```

cuando profiling esté activo.

---

# 130. Callback finding

```text
QUERY_DURING_LIFECYCLE_CALLBACK
```

podrá ser diagnóstico informativo.

---

# 131. Callback ≠ forbidden query

No se prohibirá universalmente.

Pero deberá ser visible.

---

# 132. Entity listeners

Se medirán bajo la misma arquitectura.

---

# 133. ORM event listeners

No deberán introducir estado global mutable.

---

# 134. Persistence planning

Persistence Engine convierte:

```text
ChangeSets
+
relationship changes
+
entity states
```

en:

```text
Persistence Plan
```

---

# 135. Persistence Plan performance

Se analizará:

```text
operation count
dependency graph
ordering
batch opportunities
generated-ID barriers
```

---

# 136. Generated identifiers

Pueden obligar a:

```text
INSERT parent
↓
obtain ID
↓
INSERT child
```

limitando batching.

---

# 137. Generated ID barrier

Deberá ser visible en diagnostics.

---

# 138. Persistence ordering

No deberá alterarse solo para maximizar batching si viola:

```text
FK dependencies
relationship semantics
lifecycle semantics
```

---

# 139. Topological ordering

Para dependency graph:

```text
G = (V, E)
```

persistence planner podrá utilizar ordenamiento topológico.

---

# 140. Cycles

Los ciclos podrán requerir:

```text
deferred FK
nullable intermediate state
multiple phases
platform-specific strategy
```

según capabilities.

---

# 141. Performance never invents unsafe cycle breaking

No se desactivarán FKs arbitrariamente.

---

# 142. Insert batching

Persistence Engine podrá agrupar INSERT compatibles.

---

# 143. Compatible inserts

Requieren al menos:

```text
same persistence target
compatible column shape
compatible generated-ID semantics
same tenant/shard
same transaction context
```

---

# 144. Update batching

Más difícil porque:

```text
different changed columns
different optimistic lock tokens
different predicates
```

pueden impedir batching.

---

# 145. Delete batching

Puede ser posible bajo ciertas condiciones.

Pero debe respetar:

```text
callbacks
cascades
optimistic locking
authorization
```

---

# 146. ORM batching ≠ Bulk API

Muy importante:

```text
ORM Batch Persistence
≠
Bulk Insert/Update/Delete
```

Bulk APIs pueden omitir algunas semánticas entity-by-entity explícitamente.

---

# 147. Flush batching

`flush()` podrá aprovechar batch persistence internamente solo cuando preserve contratos.

---

# 148. Query count optimization

Menos queries puede ser beneficioso.

Pero:

```text
1 giant query
```

puede ser peor que:

```text
5 bounded queries
```

---

# 149. ORM query count

Será una métrica, no objetivo absoluto.

---

# 150. Entity load amplification

Una operación:

```php
$repository->find(10);
```

idealmente produce:

```text
0 queries
```

si la entidad ya está en IdentityMap.

---

# 151. IdentityMap short-circuit

```text
Repository find
     ↓
IdentityMap lookup
     ├── HIT → entity
     └── MISS → query
```

---

# 152. IdentityMap hit

No deberá consultar DB solo para confirmar existencia si el contrato del scope garantiza la identidad managed correspondiente.

---

# 153. Refresh

`refresh()` será operación explícita para volver a consultar DB.

---

# 154. Find ≠ refresh

Regla:

```text
find()
≠
refresh()
```

---

# 155. Repository performance

Repository deberá delegar:

```text
IdentityMap
Query Engine
Hydration
```

sin mantener caches privados inconsistentes.

---

# 156. Repository cache anti-pattern

No:

```php
private array $loadedUsers = [];
```

como segunda IdentityMap.

---

# 157. Active Record performance

Model API deberá converger al mismo ORM.

Ejemplo:

```php
User::find(10);
```

y:

```php
$repository->find(10);
```

deben utilizar la misma identidad canónica.

---

# 158. Static API ≠ static ORM state

No se almacenará:

```php
User::$identityMap;
```

---

# 159. ModelContextResolver

Resolverá el EntityManager scoped.

---

# 160. Persistent worker safety

En FrankenPHP:

```text
Request A
    ↓
EntityManager A
    ↓
clear/close/reset

Request B
    ↓
EntityManager B
```

---

# 161. Forbidden

```text
Request A Entity
       ↓
Request B IdentityMap
```

---

# 162. Worker memory leak detection

ORM Performance deberá poder detectar tendencias como:

```text
request 1  → 40 MB
request 100 → 120 MB
request 500 → 400 MB
```

si instrumentation del runtime lo permite.

---

# 163. Leak ≠ high temporary memory

Debe diferenciarse:

```text
peak memory
```

de:

```text
retained memory after scope reset
```

---

# 164. ORM retained memory

Métrica importante:

```text
ORMRetainedMemoryAfterReset
```

cuando sea observable.

---

# 165. EntityManager clear

```php
$entityManager->clear();
```

elimina managed state.

---

# 166. Clear ≠ performance hack automático

El ORM no deberá ejecutar:

```php
$entityManager->clear();
```

silenciosamente durante operaciones genéricas.

---

# 167. Why

Porque podría detachar entidades que el caller todavía necesita.

---

# 168. Explicit memory policies

Procesamientos masivos podrán declarar:

```text
KEEP_MANAGED
DETACH_PROCESSED
CLEAR_ENTITY_TYPE
CALLER_MANAGED
DEDICATED_ENTITY_MANAGER
```

---

# 169. Dedicated EntityManager

Para workloads grandes puede ser una estrategia segura:

```text
Application EntityManager
       +
Dedicated Processing EntityManager
```

---

# 170. Scope isolation

Ambos tendrán IdentityMaps independientes.

---

# 171. Same entity across managers

Puede existir:

```text
User#10 in EM-A
User#10 in EM-B
```

como objetos distintos.

La garantía de identidad es por EntityManager scope, no global.

---

# 172. EntityManager crossing

Una entidad managed por EM-A no deberá persistirse silenciosamente mediante EM-B.

---

# 173. Performance does not relax ownership

Nunca.

---

# 174. Detach performance

`detach()` puede reducir memoria.

Pero debe actualizar correctamente:

```text
IdentityMap
UnitOfWork
relationship tracking
entity state
```

---

# 175. Detach cost

Detach graph cascade puede ser costoso.

---

# 176. Clear by type

Opcionalmente:

```php
$entityManager->clear(User::class);
```

podría soportarse.

Pero las relaciones hacia entidades detachadas deben conservar semántica definida.

---

# 177. Clear-by-type caution

No deberá introducir referencias managed → invalid state sin control.

---

# 178. Large read workloads

Para:

```text
millions of rows
```

managed entities completas normalmente no serán la estrategia más eficiente.

---

# 179. Preferred alternatives

Según caso:

```text
scalar hydration
tuple hydration
DTO projection
read-only entity hydration
lazy/chunk processing
streaming
```

---

# 180. ORM convenience ≠ mandatory managed entities

El desarrollador podrá seguir usando Query/Repository APIs sin convertir todo en managed entity.

---

# 181. Large write workloads

Para millones de escrituras:

```text
Entity-per-row persistence
```

puede ser demasiado costosa.

---

# 182. Alternatives

```text
Bulk Insert
Bulk Update
Bulk Delete
Import System
```

ya definidos en documentos 203–206.

---

# 183. Semantic trade-off

Bulk APIs deberán declarar qué garantías ORM no aplican entity-by-entity.

---

# 184. ORM Performance suggestion

Podrá indicar:

```text
Consider Bulk Insert for this workload.
```

No migrará automáticamente.

---

# 185. N+1 performance

N+1 tendrá integración con:

```text
223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
```

---

# 186. ORM-level N+1 context

Además del fingerprint de query, podrá conocer:

```text
entity type
relationship
owner count
loading strategy
```

---

# 187. N+1 finding

Ejemplo:

```text
Relationship:
    User.orders

Owners:
    500

Queries:
    501

Strategy:
    LAZY

Suggested evaluation:
    SELECT_IN / BATCH
```

---

# 188. N+1 ≠ always bug

Pequeños conjuntos pueden hacer lazy loading aceptable.

Severity dependerá de contexto.

---

# 189. Entity graph explosion

El problema inverso será:

```text
too much eager loading
```

---

# 190. Over-eager loading

Finding:

```text
OVER_EAGER_ENTITY_GRAPH
```

cuando:

```text
large related graph
+
low application consumption
```

pueda observarse.

---

# 191. Loaded vs consumed

Si instrumentation lo permite:

```text
relationships loaded
relationships accessed
```

puede revelar over-fetching.

---

# 192. Access instrumentation overhead

No será production default porque interceptar cada property access puede ser costoso.

---

# 193. PROFILE only

Tracking detallado de entity/relationship consumption será:

```text
PROFILE
```

por defecto.

---

# 194. ORM memory model

Aproximación:

```text
Morm
=
Mentities
+
Midentity
+
Msnapshots
+
Muow
+
Mrelations
+
Mmetadata_refs
+
Mtemporary
```

---

# 195. Metadata sharing

Metadata compilada e inmutable podrá compartirse entre requests/workers cuando sea seguro.

---

# 196. Entity state sharing

Nunca.

---

# 197. Snapshot sharing

Nunca entre scopes.

---

# 198. UnitOfWork sharing

Nunca entre scopes.

---

# 199. Relationship collection sharing

Nunca entre scopes salvo estructuras puramente metadata.

---

# 200. ORMPerformanceMeasurement

```php
final readonly class ORMPerformanceMeasurement
{
    public function __construct(
        public ORMOperationId $operationId,
        public ORMOperationKind $operation,
        public Duration $duration,
        public EntityManagerStatistics $entityManager,
        public IdentityMapStatistics $identityMap,
        public UnitOfWorkStatistics $unitOfWork,
        public RelationshipPerformanceStatistics $relationships,
        public PersistencePerformanceStatistics $persistence,
        public ORMMemoryStatistics $memory,
        public ORMOperationOutcome $outcome,
    ) {}
}
```

---

# 201. ORMOperationKind

```php
enum ORMOperationKind
{
    case FIND;
    case QUERY;
    case PERSIST;
    case REMOVE;
    case FLUSH;
    case REFRESH;
    case LOAD_RELATIONSHIP;
    case CLEAR;
    case DETACH;
    case MERGE_GRAPH;
    case CUSTOM;
}
```

---

# 202. EntityManagerStatistics

```php
final readonly class EntityManagerStatistics
{
    public function __construct(
        public int $managedAtStart,
        public int $managedAtEnd,
        public int $peakManaged,
        public int $attached,
        public int $detached,
    ) {}
}
```

---

# 203. IdentityMapStatistics

```php
final readonly class IdentityMapStatistics
{
    public function __construct(
        public int $lookups,
        public int $hits,
        public int $misses,
        public int $registrations,
        public int $removals,
        public int $peakEntries,
    ) {}
}
```

---

# 204. UnitOfWorkStatistics

```php
final readonly class UnitOfWorkStatistics
{
    public function __construct(
        public int $entitiesInspected,
        public int $newEntities,
        public int $dirtyEntities,
        public int $removedEntities,
        public int $changeSets,
        public int $relationshipChanges,
        public int $snapshots,
    ) {}
}
```

---

# 205. RelationshipPerformanceStatistics

```php
final readonly class RelationshipPerformanceStatistics
{
    public function __construct(
        public int $lazyLoads,
        public int $eagerLoads,
        public int $batchLoads,
        public int $collectionsInitialized,
        public int $cascadeEdgesVisited,
        public int $relationshipQueries,
    ) {}
}
```

---

# 206. PersistencePerformanceStatistics

```php
final readonly class PersistencePerformanceStatistics
{
    public function __construct(
        public int $inserts,
        public int $updates,
        public int $deletes,
        public int $relationshipOperations,
        public int $physicalQueries,
        public int $batches,
    ) {}
}
```

---

# 207. ORMMemoryStatistics

```php
final readonly class ORMMemoryStatistics
{
    public function __construct(
        public ?int $startBytes,
        public ?int $peakBytes,
        public ?int $endBytes,
        public ?int $estimatedManagedStateBytes,
    ) {}
}
```

---

# 208. Estimated memory

Si es estimado:

```text
estimatedManagedStateBytes
```

deberá marcarse como estimación.

---

# 209. Exact object memory

PHP no proporciona necesariamente atribución exacta de memoria por objeto.

No se fingirá precisión inexistente.

---

# 210. ORM findings

```php
enum ORMPerformanceFindingCode
{
    case LARGE_IDENTITY_MAP;
    case UNBOUNDED_ENTITY_MANAGER_GROWTH;
    case HIGH_SNAPSHOT_MEMORY;
    case EXPENSIVE_DIRTY_CHECKING;
    case LOW_CHANGESET_DENSITY;
    case LARGE_FLUSH;
    case HIGH_FLUSH_QUERY_COUNT;
    case LARGE_PERSISTENCE_GRAPH;
    case DEEP_CASCADE_GRAPH;
    case HIGH_RELATIONSHIP_QUERY_COUNT;
    case HIGH_LAZY_LOAD_COUNT;
    case HIGH_ORM_ROW_AMPLIFICATION;
    case OVER_EAGER_ENTITY_GRAPH;
    case LARGE_COLLECTION_DIFF;
    case GENERATED_ID_BATCH_BARRIER;
    case QUERY_DURING_LIFECYCLE_CALLBACK;
    case LARGE_MANAGED_READ_RESULT;
    case ORM_MEMORY_PRESSURE;
}
```

---

# 211. Low ChangeSet density

Conceptualmente:

```text
ChangeSetDensity
=
ChangedEntities
/
InspectedEntities
```

---

# 212. Example

```text
Inspected: 50,000
Changed: 3

Density:
0.006%
```

Puede justificar evaluar otra tracking strategy.

---

# 213. But no automatic strategy change

Porque:

```text
SNAPSHOT
→
EXPLICIT
```

cambia contratos.

---

# 214. ORMPerformanceFinding

```php
final readonly class ORMPerformanceFinding
{
    public function __construct(
        public ORMPerformanceFindingCode $code,
        public PerformanceSeverity $severity,
        public PerformanceEvidence $evidence,
        public ?PerformanceSuggestion $suggestion,
    ) {}
}
```

---

# 215. Evidence example

```text
Finding:
    LARGE_IDENTITY_MAP

Current managed entities:
    82,431

Peak:
    93,005

Operation:
    IMPORT

Duration:
    41.2 s

Estimated ORM state:
    612 MB

Confidence:
    HIGH
```

---

# 216. Suggestion example

```text
Evaluate a dedicated EntityManager with periodic explicit
detach/clear policy or a projection/bulk processing strategy.
```

---

# 217. ORM profiler

VoltStack podrá disponer de:

```php
$entityManager->profile(
    fn () => $service->process()
);
```

como API conceptual de desarrollo/testing.

---

# 218. Profile result

```text
ORM Profile
│
├── EntityManager
├── IdentityMap
├── UnitOfWork
├── ChangeTracking
├── Relationships
├── Persistence
├── Queries
├── Hydration
└── Memory
```

---

# 219. ORM profile example

```text
Operation
    OrderImport

Duration
    2.41 s

Entities
    managed start      120
    managed end      8,944
    peak             9,101

UnitOfWork
    inspected        8,944
    new              8,201
    dirty               41
    removed               0

Queries
    logical            104
    physical           117

Relationships
    lazy loads           83
    batch loads           4

Memory
    start             42 MB
    peak             188 MB
    end              176 MB

Findings
    LARGE_IDENTITY_MAP
    HIGH_LAZY_LOAD_COUNT
    ORM_MEMORY_PRESSURE
```

---

# 220. ORMPerformanceAnalyzer

```php
interface ORMPerformanceAnalyzer
{
    public function analyze(
        ORMPerformanceMeasurement $measurement,
        ORMPerformanceContext $context,
    ): ORMPerformanceReport;
}
```

---

# 221. Analyzer does not mutate ORM

El Analyzer nunca deberá:

```text
clear EntityManager
detach entities
change loading strategy
flush
commit
```

---

# 222. Performance recommendation ≠ action

Regla:

```text
Analyze
→
Recommend

not

Analyze
→
Mutate runtime behavior silently
```

---

# 223. Adaptive ORM optimization

Podría explorarse posteriormente.

No será parte del V1.

---

# 224. Why

Cambiar dinámicamente:

```text
lazy
→
eager
```

o:

```text
snapshot
→
explicit tracking
```

puede alterar comportamiento observable.

---

# 225. ORM telemetry

Métricas:

```text
database.orm.operation.duration
database.orm.entity_manager.managed
database.orm.identity_map.entries
database.orm.identity_map.lookups
database.orm.identity_map.hits
database.orm.uow.entities
database.orm.uow.changesets
database.orm.flush.duration
database.orm.flush.queries
database.orm.relationship.lazy_loads
database.orm.relationship.batch_loads
database.orm.persistence.inserts
database.orm.persistence.updates
database.orm.persistence.deletes
database.orm.memory.estimated
```

---

# 226. Metric cardinality

No se utilizará:

```text
entity_id
user_id
email
tenant_name
```

como label.

---

# 227. Entity type label

Podrá utilizarse:

```text
entity_type
```

solo si el número de tipos es bounded y conocido.

---

# 228. Relationship label

Podrá utilizarse de forma controlada.

Ejemplo:

```text
User.orders
```

si proviene de metadata registrada.

---

# 229. Dynamic relationship names

No se aceptarán strings arbitrarios como metric labels.

---

# 230. Sampling

Detailed ORM profiling podrá ser sampled.

---

# 231. Flush sampling

Las operaciones de flush excepcionalmente costosas podrán escalar a mayor diagnóstico en futuras operaciones.

---

# 232. Slow flush detector

Podrá existir:

```text
SLOW_FLUSH
```

como finding ORM.

---

# 233. Slow flush causes

Puede deberse a:

```text
large UoW
dirty checking
large relationship graph
many writes
lock waits
database latency
callbacks
```

---

# 234. Attribution

El reporte deberá intentar separar:

```text
ORM CPU
Query execution
Connection wait
Transaction wait
```

cuando exista información.

---

# 235. Cross-system correlation

```text
ORMOperationId
      │
      ├── QueryOperationId
      ├── TransactionOperationId
      ├── HydrationOperationId
      └── ConnectionOperationId
```

---

# 236. No duplicated accounting

Si:

```text
flush total = 200ms
```

y queries internas:

```text
150ms
```

no deberá reportarse:

```text
ORM total = 350ms
```

por doble conteo.

---

# 237. Inclusive/exclusive timing

Se distinguirá:

```text
inclusive time
exclusive time
```

---

# 238. ORM exclusive CPU

Conceptualmente:

```text
TormExclusive
=
TormTotal
-
Tquery
-
Thydration
-
Texternal
```

cuando sea medible de forma fiable.

---

# 239. Negative timing protection

Errores de clock/nesting no deberán producir métricas negativas.

---

# 240. Monotonic clock

Las duraciones deberán utilizar reloj monotónico apropiado.

---

# 241. ORM debugging

Debug Information System podrá mostrar:

```text
managed entities
UoW
IdentityMap
pending operations
relationships
flush statistics
```

con redacción.

---

# 242. Debug Toolbar

Podrá tener panel:

```text
ORM
```

separado del panel:

```text
Queries
```

---

# 243. Example toolbar

```text
ORM

Managed Entities       142
IdentityMap Hits         89
IdentityMap Misses       14

UnitOfWork
    NEW                   2
    DIRTY                 3
    REMOVED               0

Flush
    4.2 ms
    5 operations

Relationships
    Lazy Loads            7
    Batch Loads           1

Warnings
    Possible N+1
```

---

# 244. Security

Performance diagnostics nunca deberán exponer:

```text
entity field values
password hashes
tokens
credentials
PII
```

por defecto.

---

# 245. Entity debug identity

Podrá mostrarse:

```text
User#<redacted>
```

o identificador cuando la security policy lo permita.

---

# 246. Sensitive entity types

Podrán configurarse como:

```text
never inspect values
```

---

# 247. Lifecycle callback diagnostics

No deberán registrar argumentos sensibles.

---

# 248. Multitenancy

ORM performance context incluirá tenant cuando corresponda internamente.

Pero telemetry deberá evitar alta cardinalidad.

---

# 249. Cross-tenant EntityManager

No será permitido por defecto.

---

# 250. Tenant isolation performance

Nunca se debilitará para reutilizar:

```text
IdentityMap
metadata state
relationship state
```

entre tenants.

---

# 251. Metadata reuse

Metadata estructural sí puede compartirse si no contiene tenant-specific mutable state.

---

# 252. Sharding

Una entidad managed deberá conservar:

```text
Shard Ownership
```

cuando aplique.

---

# 253. Cross-shard relationship

No será creada automáticamente para optimizar carga.

---

# 254. Distributed ORM

El ORM podrá coordinar consultas distribuidas, pero:

```text
EntityManager
≠
Distributed Transaction Manager
```

---

# 255. Cross-shard flush

No se fingirá atomicidad global.

---

# 256. Performance cannot hide distributed boundaries

Si un flush produce:

```text
Shard A writes
Shard B writes
```

deberá ser explícito.

---

# 257. ORM and replicas

Managed entity reads pueden provenir de replica según Read/Write Routing.

Pero después de write:

```text
sticky/read-your-writes
```

debe respetarse.

---

# 258. IdentityMap and replica freshness

Si una entidad ya está managed:

```text
find()
```

puede devolverla sin nueva lectura.

Esto forma parte de la semántica del ORM scope.

---

# 259. Freshness-sensitive read

Debe usar operación explícita como:

```text
refresh()
```

o query con semántica apropiada.

---

# 260. ORM performance testing

Deberán existir pruebas para:

```text
IdentityMap hit behavior
IdentityMap growth
UnitOfWork scaling
dirty checking scaling
snapshot memory
flush scaling
collection diff
cascade traversal
lazy loading
batch loading
eager loading amplification
detach
clear
persistent runtime reset
```

---

# 261. Scaling tests

Ejemplo:

```text
100 entities
1,000 entities
10,000 entities
100,000 entities
```

---

# 262. Complexity regressions

Se deberá detectar si una operación pasa accidentalmente de:

```text
O(N)
```

a:

```text
O(N²)
```

---

# 263. Collection diff benchmark

Casos:

```text
100 members
1,000 members
10,000 members
```

con:

```text
0 changes
1 change
10% changes
100% changes
```

---

# 264. Flush benchmark

Casos:

```text
10,000 managed / 0 dirty
10,000 managed / 1 dirty
10,000 managed / 1,000 dirty
10,000 new
10,000 removed
```

---

# 265. IdentityMap benchmark

Medir:

```text
lookup hit
lookup miss
registration
composite identifier
tenant-aware key
shard-aware key
```

---

# 266. Relationship benchmark

Medir:

```text
lazy
JOIN eager
SELECT_IN
batch
large collection diff
```

---

# 267. Persistent runtime benchmark

Especialmente:

```text
10,000 sequential requests
```

para detectar crecimiento retenido.

---

# 268. FrankenPHP benchmark

Será obligatorio en la suite de performance del runtime principal.

---

# 269. RoadRunner/OpenSwoole

Se añadirán conforme maduren sus adaptadores.

---

# 270. ORM performance assertions

Testing API conceptual:

```php
$this->assertManagedEntityCountBelow(100);

$this->assertFlushQueryCountBelow(10);

$this->assertNoORMFinding(
    ORMPerformanceFindingCode::HIGH_LAZY_LOAD_COUNT
);
```

---

# 271. Query count assertion

```php
$this->assertORMQueryCount(2);
```

podrá correlacionar queries disparadas por operación ORM.

---

# 272. IdentityMap assertion

```php
$this->assertSame(
    $repository->find(10),
    $repository->find(10),
);
```

sigue siendo una prueba de correctness, no solo performance.

---

# 273. ORM performance policies

```php
interface ORMPerformancePolicy
{
    public function budgetFor(
        ORMOperationKind $operation,
        ORMPerformanceContext $context,
    ): ORMPerformanceBudget;
}
```

---

# 274. Policy examples

```text
HTTP request
    max managed entities = 5,000

CLI import
    max managed entities = 50,000

Dedicated batch processor
    explicit detach policy
```

---

# 275. Policy inheritance

Podrá resolverse:

```text
Framework Default
      ↓
Environment
      ↓
Application
      ↓
Operation
```

---

# 276. Runtime policy mutation

No deberá modificarse globalmente durante una request.

---

# 277. ORM optimization hints

Podrán existir hints como:

```text
READ_ONLY
PREFER_BATCH_RELATION_LOADING
DEDICATED_ENTITY_MANAGER
DETACH_AFTER_PROCESSING
```

pero deberán ser explícitos.

---

# 278. Hint ≠ hidden behavior

Un hint deberá tener semántica documentada.

---

# 279. Read-only hint

Puede permitir optimizaciones reales porque cambia explícitamente el contrato ORM.

---

# 280. Detach-after-processing

Solo deberá usarse en APIs diseñadas para procesamiento.

No en queries normales.

---

# 281. Lazy Collection integration

Documento 202 definió lazy iteration.

ORM Performance deberá recordar:

```text
Lazy Collection
≠
Constant ORM Memory
```

---

# 282. Chunk integration

Documento 201 definió chunk processing.

Regla:

```text
Bounded Chunk
≠
Bounded IdentityMap
```

---

# 283. Large Dataset integration

Documento 208 podrá utilizar:

```text
read-only hydration
dedicated EntityManager
projection
batch loading
explicit clearing
```

como estrategias.

---

# 284. Import integration

Import System podrá preferir:

```text
Bulk APIs
```

sobre entidad por entidad cuando el contrato lo permita.

---

# 285. Export integration

Export normalmente debería evitar managed entities completas si solo necesita datos serializados.

---

# 286. Query Performance integration

Documento 243 proveerá:

```text
physical queries
rows
connection wait
execution duration
fan-out
```

---

# 287. Hydration Performance integration

Documento 245 proveerá:

```text
row decoding
entity allocation
property assignment
IdentityMap reconciliation
relationship assembly
```

---

# 288. Metadata Compilation integration

Documento 246 reducirá costo de:

```text
reflection
attribute parsing
mapping interpretation
accessor generation
```

---

# 289. Query Compilation Optimization integration

Documento 247 reducirá costos de compilación repetitiva.

---

# 290. Memory Management integration

Documento 248 formalizará:

```text
memory budgets
retention
pressure
release
```

---

# 291. Resource Governance integration

Documento 249 establecerá límites globales.

---

# 292. Benchmark integration

Documento 250 validará cuantitativamente todas estas decisiones.

---

# 293. Estructura de directorios

```text
src/Quantum/Database/Performance/ORM/
│
├── Contract/
│   ├── ORMPerformanceManager.php
│   ├── ORMPerformanceAnalyzer.php
│   ├── ORMPerformancePolicy.php
│   ├── ORMPerformanceRecorder.php
│   └── ORMPerformanceFindingDetector.php
│
├── Context/
│   └── ORMPerformanceContext.php
│
├── Budget/
│   └── ORMPerformanceBudget.php
│
├── Model/
│   ├── ORMOperationId.php
│   ├── ORMOperationKind.php
│   ├── ORMOperationOutcome.php
│   ├── ORMPerformanceMeasurement.php
│   ├── ORMPerformanceMeasurementMode.php
│   ├── EntityManagerStatistics.php
│   ├── IdentityMapStatistics.php
│   ├── UnitOfWorkStatistics.php
│   ├── FlushPerformanceStatistics.php
│   ├── RelationshipPerformanceStatistics.php
│   ├── PersistencePerformanceStatistics.php
│   └── ORMMemoryStatistics.php
│
├── Measurement/
│   ├── DefaultORMPerformanceManager.php
│   ├── ORMPerformanceSession.php
│   ├── EntityManagerPerformanceRecorder.php
│   ├── IdentityMapPerformanceRecorder.php
│   ├── UnitOfWorkPerformanceRecorder.php
│   ├── RelationshipPerformanceRecorder.php
│   └── PersistencePerformanceRecorder.php
│
├── Analysis/
│   ├── DefaultORMPerformanceAnalyzer.php
│   ├── EntityManagerGrowthAnalyzer.php
│   ├── IdentityMapAnalyzer.php
│   ├── UnitOfWorkAnalyzer.php
│   ├── DirtyCheckingAnalyzer.php
│   ├── FlushPerformanceAnalyzer.php
│   ├── RelationshipPerformanceAnalyzer.php
│   ├── CascadePerformanceAnalyzer.php
│   └── PersistencePerformanceAnalyzer.php
│
├── Finding/
│   ├── ORMPerformanceFinding.php
│   ├── ORMPerformanceFindingCode.php
│   ├── LargeIdentityMapDetector.php
│   ├── EntityManagerGrowthDetector.php
│   ├── ExpensiveDirtyCheckingDetector.php
│   ├── LargeFlushDetector.php
│   ├── HighLazyLoadDetector.php
│   ├── ORMRowAmplificationDetector.php
│   ├── LargeCollectionDiffDetector.php
│   ├── LargePersistenceGraphDetector.php
│   ├── LifecycleQueryDetector.php
│   └── ORMMemoryPressureDetector.php
│
├── Report/
│   ├── ORMPerformanceReport.php
│   ├── ORMPerformanceReportBuilder.php
│   └── ORMPerformanceReportFormatter.php
│
├── Telemetry/
│   └── ORMPerformanceTelemetryBridge.php
│
├── Testing/
│   ├── FakeORMPerformanceRecorder.php
│   ├── FakeORMPerformancePolicy.php
│   ├── ORMPerformanceAssertions.php
│   └── ORMPerformanceTestClock.php
│
└── Exception/
    ├── DatabaseORMPerformanceException.php
    ├── ORMPerformanceMeasurementException.php
    ├── ORMPerformanceAnalysisException.php
    ├── ORMPerformanceBudgetException.php
    └── ORMPerformanceInstrumentationException.php
```

---

# 294. Dependencias

Permitido:

```text
ORM Performance
      ↓
ORM metadata
EntityManager observations
IdentityMap observations
UnitOfWork observations
Relationship observations
Persistence observations
Query Performance summaries
Hydration summaries
Telemetry contracts
```

No permitido:

```text
ORM Performance
      ↓
direct SQL execution
```

---

# 295. Instrumentación

La instrumentación deberá colocarse en puntos semánticos.

Ejemplo:

```text
EntityManager::flush
      ↓
ORMPerformanceSession
      ↓
UnitOfWork
      ↓
Persistence
```

No mediante profiling arbitrario disperso.

---

# 296. Hot path awareness

Operaciones como:

```text
IdentityMap lookup
property access
entity state lookup
```

son hot paths.

La instrumentación detallada deberá evitar overhead excesivo.

---

# 297. Counters vs events

En hot paths será preferible:

```text
counter++
```

local al scope antes que emitir un evento completo por cada lookup.

---

# 298. Aggregated emission

Al finalizar la operación:

```text
local counters
      ↓
summary
      ↓
telemetry
```

---

# 299. No event storm

No se emitirá por defecto:

```text
IdentityMapLookupEvent
```

millones de veces hacia Event System.

---

# 300. Performance telemetry ≠ domain events

Regla:

```text
High-frequency instrumentation
≠
Application Event Pipeline
```

---

# 301. Sampling lifecycle callbacks

Callbacks sí podrán medirse individualmente en PROFILE porque su frecuencia suele ser menor y su costo puede ser arbitrario.

---

# 302. Entity-level timing

Medir cada entidad individualmente será:

```text
PROFILE-only
```

por defecto.

---

# 303. Aggregate entity metrics

Production preferirá:

```text
entity count
duration histogram
operation count
```

---

# 304. Failure behavior

Si ORM operation falla:

```text
FAILED
```

se conservarán métricas parciales.

---

# 305. Unknown persistence outcome

Si existe:

```text
UNKNOWN
```

por pérdida de conexión durante escritura:

```text
ORM Performance
```

no deberá reinterpretarlo.

---

# 306. Tainted EntityManager

Si EntityManager queda:

```text
TAINTED
```

la medición lo registrará.

---

# 307. Performance finding

Podrá existir:

```text
ENTITY_MANAGER_TAINTED
```

como diagnóstico operacional, aunque no sea estrictamente performance.

Preferiblemente quedará en resiliencia/debug y solo se correlacionará aquí.

---

# 308. Failure does not erase measurement

Incluso una operación fallida puede ser la más importante para profiling.

---

# 309. Cancellation

Si operación es cancelada:

```text
CANCELLED
```

y deberá liberar instrumentation state.

---

# 310. Deadline

Performance System consumirá deadline pero no será su propietario.

---

# 311. Resource governance

Si Resource Governor decide detener una operación:

```text
RESOURCE_LIMIT_EXCEEDED
```

deberá distinguirse de query/ORM bug.

---

# 312. Garbage collection

PHP GC puede afectar benchmarks.

Los benchmarks deberán registrar/configurar su comportamiento cuando sea relevante.

---

# 313. GC ≠ ORM cleanup

Que PHP recolecte objetos no sustituye:

```text
EntityManager clear
scope reset
transaction cleanup
```

---

# 314. WeakMap considerations

Si se utilizan WeakMaps internamente, su efecto deberá benchmarkearse con workloads reales.

---

# 315. Reflection

Reflection repetitiva en hot path estará prohibida cuando metadata pueda precompilarse.

---

# 316. Attribute parsing

No deberá ocurrir por cada entidad hidratada.

---

# 317. Metadata compilation

Será responsabilidad de:

```text
246_DATABASE_METADATA_COMPILATION_SYSTEM.md
```

---

# 318. Property access

Podrán utilizarse:

```text
compiled accessors
cached closures
generated hydrators
```

según benchmarks.

---

# 319. Generated code

No se utilizará únicamente por asumir que siempre es más rápido.

Deberá probarse.

---

# 320. JIT assumptions

La arquitectura no dependerá obligatoriamente de PHP JIT.

---

# 321. OPcache

VoltStack podrá beneficiarse de OPcache, pero correctness no dependerá de él.

---

# 322. Persistent runtime advantage

FrankenPHP puede permitir reutilizar:

```text
compiled metadata
immutable mapping structures
compiled accessors
```

entre requests.

---

# 323. Persistent runtime risk

También aumenta riesgo de:

```text
state leakage
memory retention
stale metadata
cross-request contamination
```

---

# 324. Shared immutable / scoped mutable rule

Regla central:

```text
Immutable Structural State
→ shareable

Mutable ORM Runtime State
→ scoped
```

---

# 325. Shared

Ejemplos:

```text
EntityMetadata
RelationshipMetadata
TypeMetadata
CompiledAccessors
PerformancePolicies
```

cuando sean inmutables.

---

# 326. Scoped

```text
EntityManager
IdentityMap
UnitOfWork
Snapshots
ChangeSets
LazyLoad state
Transaction
Tenant
```

---

# 327. ORM performance anti-patterns

Quedan desaconsejados:

```text
Use one global EntityManager forever

Use IdentityMap as second-level cache

Clear EntityManager silently

Detach arbitrary entities automatically

Disable snapshots globally for speed

Disable dirty checking without explicit tracking contract

Treat partial entities as full managed entities

Fetch-join every relationship

Lazy-load every relationship

Fix N+1 by loading the entire graph

Use DISTINCT to hide join amplification

Use SELECT * for every entity regardless of workload

Keep millions of entities managed

Flush after every persist()

Commit after every flush()

Use entity-per-row processing for every bulk workload

Execute queries inside lifecycle callbacks invisibly

Use O(N²) collection diff

Clone entire entities for snapshots

Parse attributes on every hydration

Use reflection repeatedly in hot paths

Share UnitOfWork across requests

Share IdentityMap across tenants

Use static Active Record state

Ignore generated-ID batch barriers

Assume fewer queries always means faster ORM

Assume lazy means low memory

Assume chunk means bounded IdentityMap

Assume read-only entity means immutable object

Assume EntityManager clear is free

Assume GC equals ORM lifecycle cleanup
```

---

# 328. Invariantes arquitectónicas

## DB-ORMPERF-001
ORM Performance no será Query Performance.

## DB-ORMPERF-002
ORM Performance no será Hydration Performance.

## DB-ORMPERF-003
Optimización ORM preservará identidad canónica.

## DB-ORMPERF-004
Optimización ORM preservará EntityState.

## DB-ORMPERF-005
Optimización ORM preservará ChangeSets.

## DB-ORMPERF-006
Optimización ORM preservará UnitOfWork.

## DB-ORMPERF-007
Optimización ORM preservará relaciones.

## DB-ORMPERF-008
Optimización ORM preservará transacciones.

## DB-ORMPERF-009
Optimización ORM preservará tenant isolation.

## DB-ORMPERF-010
Optimización ORM preservará shard ownership.

## DB-ORMPERF-011
EntityManager será scoped.

## DB-ORMPERF-012
IdentityMap será scoped.

## DB-ORMPERF-013
UnitOfWork será scoped.

## DB-ORMPERF-014
Snapshots serán scoped.

## DB-ORMPERF-015
ChangeSets serán scoped.

## DB-ORMPERF-016
Static Model API no implicará static ORM state.

## DB-ORMPERF-017
IdentityMap no será Entity Cache.

## DB-ORMPERF-018
IdentityMap no será compartido entre requests.

## DB-ORMPERF-019
IdentityMap no será compartido entre tenants.

## DB-ORMPERF-020
IdentityMap no será compartido entre EntityManagers.

## DB-ORMPERF-021
Canonical identity será mantenida dentro del scope.

## DB-ORMPERF-022
Duplicate managed identity no será aceptada silenciosamente.

## DB-ORMPERF-023
Identity validation no será eliminada por performance.

## DB-ORMPERF-024
Bounded chunk no implicará bounded IdentityMap.

## DB-ORMPERF-025
Lazy iteration no implicará bounded ORM memory.

## DB-ORMPERF-026
EntityManager growth será observable.

## DB-ORMPERF-027
Managed entity count será observable.

## DB-ORMPERF-028
IdentityMap peak será observable.

## DB-ORMPERF-029
UoW size será observable.

## DB-ORMPERF-030
Flush duration será observable.

## DB-ORMPERF-031
Flush no será commit.

## DB-ORMPERF-032
Persist no será INSERT.

## DB-ORMPERF-033
Remove no será DELETE inmediato.

## DB-ORMPERF-034
ChangeSet no será SQL.

## DB-ORMPERF-035
Persistence Plan no será SQL.

## DB-ORMPERF-036
Snapshot no requerirá clonar entidad completa.

## DB-ORMPERF-037
Snapshot deberá preservar comparación correcta.

## DB-ORMPERF-038
Mutable snapshot values no compartirán estado inseguro.

## DB-ORMPERF-039
Dirty checking será type-aware.

## DB-ORMPERF-040
Dirty candidate no será dirty entity.

## DB-ORMPERF-041
Tracking strategy no cambiará silenciosamente.

## DB-ORMPERF-042
Read-only será contrato explícito.

## DB-ORMPERF-043
Read-only no significará PHP immutable.

## DB-ORMPERF-044
Projection no será partial managed entity.

## DB-ORMPERF-045
Partial managed entity no será optimización genérica.

## DB-ORMPERF-046
EntityState lookup evitará scans completos.

## DB-ORMPERF-047
Weak references no romperán managed lifecycle.

## DB-ORMPERF-048
Empty ChangeSet no generará UPDATE innecesario por defecto.

## DB-ORMPERF-049
Value comparison respetará Type System.

## DB-ORMPERF-050
Relationship PARTIAL no será COMPLETE.

## DB-ORMPERF-051
Collection diff evitará O(N²) cuando sea posible.

## DB-ORMPERF-052
Collection diff soportará entidades sin ID generado.

## DB-ORMPERF-053
Owning side seguirá siendo persistence authority.

## DB-ORMPERF-054
Inverse side no ganará autoridad por performance.

## DB-ORMPERF-055
Lazy loading no será N+1 por definición.

## DB-ORMPERF-056
N+1 será detectable.

## DB-ORMPERF-057
Eager loading no será JOIN por definición.

## DB-ORMPERF-058
JOIN eager no será siempre preferido.

## DB-ORMPERF-059
SELECT_IN será estrategia posible.

## DB-ORMPERF-060
Batch loading será estrategia posible.

## DB-ORMPERF-061
Fetch amplification será observable.

## DB-ORMPERF-062
N+1 no se corregirá creando cartesian explosion.

## DB-ORMPERF-063
Relationship batch size respetará platform limits.

## DB-ORMPERF-064
Cascade traversal detectará ciclos.

## DB-ORMPERF-065
Cascade graph será bounded por resource policy cuando aplique.

## DB-ORMPERF-066
Database Cascade no será ORM Cascade.

## DB-ORMPERF-067
Lifecycle callbacks podrán ser perfilados.

## DB-ORMPERF-068
Queries en lifecycle callbacks serán correlacionables.

## DB-ORMPERF-069
Callback query no será prohibida universalmente.

## DB-ORMPERF-070
Persistence ordering preservará FK semantics.

## DB-ORMPERF-071
Persistence ordering preservará lifecycle semantics.

## DB-ORMPERF-072
Generated IDs podrán crear batch barriers.

## DB-ORMPERF-073
Batch barrier será observable.

## DB-ORMPERF-074
FKs no serán desactivadas arbitrariamente.

## DB-ORMPERF-075
ORM batching no será Bulk API.

## DB-ORMPERF-076
Menor query count no será objetivo absoluto.

## DB-ORMPERF-077
IdentityMap HIT podrá evitar query.

## DB-ORMPERF-078
find no será refresh.

## DB-ORMPERF-079
Repository no mantendrá segunda IdentityMap.

## DB-ORMPERF-080
Active Record y Repository compartirán canonical ORM.

## DB-ORMPERF-081
Persistent workers resetearán mutable ORM state.

## DB-ORMPERF-082
Request A state no llegará a Request B.

## DB-ORMPERF-083
Retained memory será distinguible de peak memory cuando sea posible.

## DB-ORMPERF-084
EntityManager clear no ocurrirá silenciosamente.

## DB-ORMPERF-085
Detach no ocurrirá silenciosamente.

## DB-ORMPERF-086
Dedicated EntityManager será estrategia explícita.

## DB-ORMPERF-087
Identity guarantee será por EntityManager scope.

## DB-ORMPERF-088
Entidad de EM-A no será silently managed por EM-B.

## DB-ORMPERF-089
Detach actualizará todas las estructuras relevantes.

## DB-ORMPERF-090
Clear-by-type preservará invariantes de relaciones.

## DB-ORMPERF-091
Large reads no requerirán managed entities.

## DB-ORMPERF-092
Large writes podrán usar Bulk APIs.

## DB-ORMPERF-093
Bulk API trade-offs serán explícitos.

## DB-ORMPERF-094
N+1 severity dependerá de contexto.

## DB-ORMPERF-095
Over-eager loading será diagnosticable.

## DB-ORMPERF-096
Per-property access instrumentation no será production default.

## DB-ORMPERF-097
ORM memory incluirá más que entity objects.

## DB-ORMPERF-098
Immutable metadata podrá compartirse.

## DB-ORMPERF-099
Mutable entity state no podrá compartirse.

## DB-ORMPERF-100
Snapshots no podrán compartirse entre scopes.

## DB-ORMPERF-101
UnitOfWork no podrá compartirse entre scopes.

## DB-ORMPERF-102
Relationship runtime state no podrá compartirse entre scopes.

## DB-ORMPERF-103
ORM metrics tendrán cardinalidad bounded.

## DB-ORMPERF-104
PII no será metric label.

## DB-ORMPERF-105
Entity IDs no serán metric labels.

## DB-ORMPERF-106
ORM profiling detallado podrá ser sampled.

## DB-ORMPERF-107
Slow flush tendrá attribution cuando sea posible.

## DB-ORMPERF-108
ORM/query timings evitarán double counting.

## DB-ORMPERF-109
Inclusive y exclusive timing serán distinguibles.

## DB-ORMPERF-110
Duraciones usarán monotonic clock.

## DB-ORMPERF-111
Debug diagnostics serán redacted.

## DB-ORMPERF-112
Sensitive entity values no serán expuestos.

## DB-ORMPERF-113
Tenant isolation no será debilitado por performance.

## DB-ORMPERF-114
Cross-tenant IdentityMap estará prohibido.

## DB-ORMPERF-115
Shard ownership no será ignorado por performance.

## DB-ORMPERF-116
Cross-shard ACID no será fingido.

## DB-ORMPERF-117
Replica freshness seguirá Read/Write policy.

## DB-ORMPERF-118
IdentityMap semantics podrán devolver managed entity sin DB query.

## DB-ORMPERF-119
Freshness explícita usará refresh/semántica apropiada.

## DB-ORMPERF-120
ORM scaling será benchmarkeado.

## DB-ORMPERF-121
O(N²) regressions deberán detectarse.

## DB-ORMPERF-122
Collection diff tendrá benchmarks.

## DB-ORMPERF-123
Flush tendrá benchmarks por workload.

## DB-ORMPERF-124
IdentityMap tendrá benchmarks.

## DB-ORMPERF-125
Relationship strategies tendrán benchmarks.

## DB-ORMPERF-126
Persistent runtime tendrá soak tests.

## DB-ORMPERF-127
FrankenPHP será runtime benchmark principal.

## DB-ORMPERF-128
Performance assertions serán soportadas.

## DB-ORMPERF-129
ORMPerformancePolicy será configurable.

## DB-ORMPERF-130
Policies podrán variar por workload.

## DB-ORMPERF-131
Policy mutation no será global por request.

## DB-ORMPERF-132
Optimization hints serán explícitos.

## DB-ORMPERF-133
Hint no será hidden behavior.

## DB-ORMPERF-134
Chunk Processing cooperará con ORM memory policies.

## DB-ORMPERF-135
Lazy Collection cooperará con ORM memory policies.

## DB-ORMPERF-136
Large Dataset Processing podrá utilizar dedicated EM.

## DB-ORMPERF-137
Import podrá utilizar Bulk API.

## DB-ORMPERF-138
Export podrá evitar managed entities.

## DB-ORMPERF-139
Query Performance será sistema separado.

## DB-ORMPERF-140
Hydration Performance será sistema separado.

## DB-ORMPERF-141
Metadata Compilation será sistema separado.

## DB-ORMPERF-142
Memory Management será sistema separado.

## DB-ORMPERF-143
Resource Governance será sistema separado.

## DB-ORMPERF-144
Analyzer no mutará EntityManager.

## DB-ORMPERF-145
Analyzer no hará flush.

## DB-ORMPERF-146
Analyzer no hará commit.

## DB-ORMPERF-147
Analyzer no hará detach.

## DB-ORMPERF-148
Analyzer no hará clear.

## DB-ORMPERF-149
Adaptive ORM optimization no será V1.

## DB-ORMPERF-150
High-frequency instrumentation evitará event storms.

## DB-ORMPERF-151
IdentityMap lookup no emitirá application event por defecto.

## DB-ORMPERF-152
Hot-path metrics serán agregadas localmente.

## DB-ORMPERF-153
Per-entity timing será PROFILE-only por defecto.

## DB-ORMPERF-154
Failed operations conservarán métricas parciales.

## DB-ORMPERF-155
UNKNOWN persistence outcome permanecerá UNKNOWN.

## DB-ORMPERF-156
TAINTED EntityManager permanecerá TAINTED.

## DB-ORMPERF-157
Cancellation liberará instrumentation state.

## DB-ORMPERF-158
Resource Governor será autoridad sobre límites.

## DB-ORMPERF-159
GC no sustituirá ORM cleanup.

## DB-ORMPERF-160
Reflection repetitiva será evitada en hot paths.

## DB-ORMPERF-161
Attribute parsing no ocurrirá por entidad.

## DB-ORMPERF-162
Generated accessors deberán benchmarkearse.

## DB-ORMPERF-163
Arquitectura no dependerá de JIT.

## DB-ORMPERF-164
OPcache será optimización, no requisito de correctness.

## DB-ORMPERF-165
Persistent runtimes reutilizarán solo estado seguro.

## DB-ORMPERF-166
Immutable structural state podrá compartirse.

## DB-ORMPERF-167
Mutable ORM runtime state será scoped.

## DB-ORMPERF-168
Performance recorder failure no romperá ORM normalmente.

## DB-ORMPERF-169
Telemetry failure no será persistence failure.

## DB-ORMPERF-170
Performance findings requerirán evidencia.

## DB-ORMPERF-171
Suggestion no será automatic mutation.

## DB-ORMPERF-172
Estimated memory será marcado como estimado.

## DB-ORMPERF-173
No se fingirá precisión de memoria inexistente.

## DB-ORMPERF-174
ORM query amplification será observable.

## DB-ORMPERF-175
Relationship query amplification será observable.

## DB-ORMPERF-176
Flush query amplification será observable.

## DB-ORMPERF-177
Entity graph amplification será observable.

## DB-ORMPERF-178
Lifecycle amplification será observable.

## DB-ORMPERF-179
Performance no redefinirá persistence semantics.

## DB-ORMPERF-180
Performance no redefinirá transaction semantics.

## DB-ORMPERF-181
Performance no redefinirá relationship ownership.

## DB-ORMPERF-182
Performance no redefinirá entity identity.

## DB-ORMPERF-183
Performance no redefinirá change tracking silenciosamente.

## DB-ORMPERF-184
Performance no redefinirá lazy/eager semantics silenciosamente.

## DB-ORMPERF-185
Performance no saltará Query Engine.

## DB-ORMPERF-186
Performance no accederá directamente a PDO.

## DB-ORMPERF-187
Performance no accederá directamente al Driver para ejecutar.

## DB-ORMPERF-188
ORM observations serán correlacionables con Query Telemetry.

## DB-ORMPERF-189
ORM observations serán correlacionables con Transaction Telemetry.

## DB-ORMPERF-190
ORM observations serán correlacionables con Hydration Telemetry.

## DB-ORMPERF-191
Production defaults priorizarán bajo overhead.

## DB-ORMPERF-192
Development podrá habilitar profiling profundo.

## DB-ORMPERF-193
Testing podrá usar strict performance budgets.

## DB-ORMPERF-194
Benchmarks validarán optimizaciones antes de adoptarlas.

## DB-ORMPERF-195
Optimización ORM no será aceptada únicamente por microbenchmark.

## DB-ORMPERF-196
Workloads reales representativos serán requeridos.

## DB-ORMPERF-197
Memory y latency se evaluarán conjuntamente.

## DB-ORMPERF-198
Menor memoria no justificará más queries ilimitadas.

## DB-ORMPERF-199
Menos queries no justificará memoria ilimitada.

## DB-ORMPERF-200
El ORM optimizará trabajo sin sacrificar sus invariantes.

---

# 329. Modelo formal del IdentityMap

Sea:

```text
I
```

el IdentityMap.

Para una entidad:

```text
E(type, id, context)
```

debe cumplirse:

```text
I(type, id, context) = e
```

y durante el mismo scope:

```text
find(type, id, context)
→
e
```

si `e` ya está managed y la operación no solicita explícitamente refresh.

---

# 330. Canonical identity invariant

Para dos resoluciones:

```text
e1 = find(User, 10)
e2 = find(User, 10)
```

en el mismo EntityManager y contexto:

```text
e1 === e2
```

---

# 331. UnitOfWork cost model

Sea:

```text
M = managed entities
D = dirty candidates
C = actual changed entities
R = relationship changes
```

Una estrategia ingenua de snapshot scan puede aproximarse a:

```text
Costflush
≈
O(M × F)
```

donde `F` es el número medio de campos.

Una estrategia con candidatos puede aproximarse a:

```text
Costflush
≈
O(D × F + R)
```

si mantiene correctamente las garantías.

---

# 332. Collection diff model

Para old/new sets:

```text
O = old members
N = new members
```

con hashing apropiado:

```text
Removed = O - N
Added   = N - O
```

con costo esperado aproximadamente:

```text
O(|O| + |N|)
```

en lugar de comparación anidada:

```text
O(|O| × |N|)
```

---

# 333. Persistence graph

Sea:

```text
G = (V, E)
```

donde:

```text
V = persistence operations
E = dependency constraints
```

Persistence Planner deberá encontrar un orden:

```text
P = topologicalOrder(G)
```

cuando el grafo sea acíclico.

---

# 334. ORM amplification

Puede modelarse:

```text
Aorm
=
PhysicalDatabaseOperations
/
LogicalORMOperations
```

pero deberá interpretarse según operación.

---

# 335. Relationship amplification

```text
Arel
=
RelationshipQueries
/
RelationshipRequests
```

---

# 336. Flush density

```text
Dflush
=
ChangedEntities
/
InspectedEntities
```

---

# 337. Memory growth

Para chunks `C1 ... Cn`:

```text
Managed(n)
=
Managed(n-1)
+
NewlyManaged(n)
-
Detached(n)
```

Si:

```text
Detached(n) = 0
```

y cada chunk añade entidades:

```text
Managed(n)
```

crecerá continuamente.

---

# 338. Objetivo para procesamiento acotado

Con política explícita:

```text
NewlyManaged(n)
≈
Detached(n)
```

podría mantenerse:

```text
Managed(n)
≈
bounded
```

sin que el ORM lo haga silenciosamente.

---

# 339. Arquitectura de medición

```text
ORM API
   │
   ▼
ORMPerformanceSession
   │
   ├── EntityManager counters
   ├── IdentityMap counters
   ├── UnitOfWork counters
   ├── Relationship counters
   ├── Persistence counters
   ├── Query correlation
   └── Memory observations
   │
   ▼
ORMPerformanceMeasurement
   │
   ▼
ORMPerformanceAnalyzer
   │
   ├── Growth detectors
   ├── Dirty-check detectors
   ├── Relationship detectors
   ├── Flush detectors
   └── Memory detectors
   │
   ▼
ORMPerformanceReport
```

---

# 340. Filosofía de optimización

VoltStack no buscará:

```text
ORM convenience
at any cost
```

ni:

```text
maximum performance
by bypassing ORM semantics
```

La arquitectura será:

```text
ORM Developer Experience
          +
Canonical Identity
          +
UnitOfWork Correctness
          +
Efficient Change Tracking
          +
Controlled Relationship Loading
          +
Efficient Persistence Planning
          +
Observable Resource Cost
```

---

# 341. Estrategia para workloads

## Aplicaciones CRUD normales

```text
Managed entities
IdentityMap
Snapshot tracking
Lazy/eager relationships
Normal flush
```

---

## Lecturas grandes

Preferir evaluar:

```text
Projection
Scalar/Tuple hydration
Read-only entities
Lazy processing
Streaming
```

---

## Procesamiento masivo ORM

```text
Chunk/Keyset
+
Dedicated EntityManager
+
Explicit memory policy
+
Batch relationship loading
+
Periodic flush
+
Explicit detach/clear
```

---

## Escritura masiva

Evaluar:

```text
Bulk APIs
Import System
```

si no se necesitan todas las garantías entity-by-entity.

---

# 342. Regla arquitectónica final

> **VoltStack no tratará el ORM como una abstracción gratuita: cada entidad administrada, snapshot, relación, ChangeSet y operación de persistencia tiene un costo que deberá poder medirse. Sin embargo, ese costo nunca se reducirá destruyendo silenciosamente las garantías de identidad, estado, relaciones o persistencia que definen al ORM.**

En forma resumida:

```text
Managed Entity
≠
Free Object

IdentityMap
≠
Cache

UnitOfWork
≠
Transaction

Flush
≠
Commit

Persist
≠
INSERT

Remove
≠
DELETE

Snapshot
≠
Entity Clone

Dirty Candidate
≠
Dirty Entity

Projection
≠
Partial Managed Entity

Read-Only
≠
Immutable

Lazy
≠
Low Memory

Chunk
≠
Bounded IdentityMap

Eager
≠
JOIN

Fewer Queries
≠
Faster ORM

More JOINs
≠
Better ORM

Clear
≠
Harmless Optimization

Bulk
≠
ORM Batch Persistence

GC
≠
ORM Cleanup

Performance
≠
Weaker Semantics
```

---

# 343. Resultado arquitectónico

Una operación:

```php
$orders = Order::query()
    ->with(['customer', 'items.product'])
    ->where('status', OrderStatus::PENDING)
    ->get();

foreach ($orders as $order) {
    $order->markAsProcessing();
}

$entityManager->flush();
```

podrá producir un reporte:

```text
ORM PERFORMANCE

Operation
    Order Processing

Total
    184.7 ms

EntityManager
    managed start       12
    managed end      4,824
    peak             4,824

IdentityMap
    lookups           9,442
    hits              4,618
    misses            4,824

Hydration
    root entities       500
    related entities  4,312

Relationships
    eager loads           3
    lazy loads            0

UnitOfWork
    inspected         4,824
    dirty               500
    change sets          500

Flush
    ORM preparation     8.1 ms
    persistence plan    2.7 ms
    database           34.2 ms

Queries
    logical               5
    physical              5

Memory
    start              38 MB
    peak              117 MB
    end               112 MB

Findings
    LARGE_IDENTITY_MAP
    LOW_CHANGESET_DENSITY

Suggestions
    Evaluate read-only loading for relationships that are
    not modified.

    For larger batches, evaluate a dedicated processing
    EntityManager with explicit memory management.
```

sin convertir ORM Performance en:

```text
EntityManager
UnitOfWork
Hydrator
Query Engine
Persistence Engine
Resource Governor
```

---

# 344. Relación con el Bloque 24

```text
242 DATABASE PERFORMANCE ARCHITECTURE
             │
             ├── 243 QUERY PERFORMANCE
             │
             ├── 244 ORM PERFORMANCE
             │       │
             │       ├── EntityManager
             │       ├── IdentityMap
             │       ├── UnitOfWork
             │       ├── Change Tracking
             │       ├── Relationships
             │       ├── Persistence
             │       └── ORM Memory
             │
             ├── 245 HYDRATION PERFORMANCE
             ├── 246 METADATA COMPILATION
             ├── 247 QUERY COMPILATION OPTIMIZATION
             ├── 248 MEMORY MANAGEMENT
             ├── 249 RESOURCE GOVERNANCE
             └── 250 PERFORMANCE BENCHMARK
```

---

# 345. Estado del Bloque 24

```text
BLOCK 24 — PERFORMANCE

✓ 242_DATABASE_PERFORMANCE_ARCHITECTURE.md
✓ 243_DATABASE_QUERY_PERFORMANCE_SYSTEM.md
✓ 244_DATABASE_ORM_PERFORMANCE_SYSTEM.md
│
├── 245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md
├── 246_DATABASE_METADATA_COMPILATION_SYSTEM.md
├── 247_DATABASE_QUERY_COMPILATION_OPTIMIZATION_SYSTEM.md
├── 248_DATABASE_MEMORY_MANAGEMENT_SYSTEM.md
├── 249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
└── 250_DATABASE_PERFORMANCE_BENCHMARK_SYSTEM.md
```

---

# 346. Siguiente documento

```text
245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md
```

El siguiente documento deberá profundizar específicamente en:

```text
Database Result
      ↓
Result Cursor
      ↓
Hydration Plan
      ↓
Type Conversion
      ↓
Identity Resolution
      ↓
Entity Allocation
      ↓
Property Assignment
      ↓
Relationship Assembly
      ↓
Snapshot Creation
      ↓
UnitOfWork Registration
```

y deberá definir, entre otros:

```text
hydration throughput
rows/second
entity allocation cost
reflection avoidance
compiled hydrators
property accessor optimization
type conversion cost
IdentityMap reconciliation cost
row deduplication
JOIN amplification
partial projection cost
tuple/scalar hydration
DTO hydration
entity hydration
relationship assembly
snapshot creation
hydration plan caching
streaming hydration
batch hydration
memory pressure
large result sets
persistent runtime safety
hydration profiling
```

bajo la regla:

> **La hidratación de VoltStack deberá minimizar el costo de transformar resultados físicos en representaciones de aplicación sin sacrificar identidad canónica, tipos, loaded-field knowledge, snapshots, relaciones ni las garantías del UnitOfWork.**