# 199_DATABASE_PAGINATION_SYSTEM.md

# VoltStack Quantum Database
## Database Pagination System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 199 — Database Pagination System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `198_DATABASE_TEST_DATA_GENERATION_SYSTEM.md`  
**Siguiente documento:** `200_DATABASE_CURSOR_PAGINATION_SYSTEM.md`

---

# 1. Propósito

`Database Pagination System` define la arquitectura general mediante la cual VoltStack dividirá conjuntos de resultados potencialmente grandes en unidades navegables, manteniendo semántica de consulta, ordenamiento, consistencia, seguridad, distribución y rendimiento explícitos.

La paginación no será tratada simplemente como:

```sql
LIMIT 20 OFFSET 40
```

sino como una operación semántica del Query Engine.

Ejemplo de API:

```php
$users = User::query()
    ->where('status', UserStatus::ACTIVE)
    ->orderBy('created_at', 'desc')
    ->paginate(
        perPage: 25,
        page: 3,
    );
```

o:

```php
$result = DB::table('users')
    ->where('status', 'active')
    ->orderBy('created_at', 'desc')
    ->paginate(25);
```

El resultado podrá proporcionar:

```php
$result->items();
$result->page();
$result->perPage();
$result->hasNextPage();
$result->hasPreviousPage();
$result->total();
$result->lastPage();
```

cuando la estrategia utilizada permita conocer esa información.

La regla central será:

> **Pagination transforma una consulta ordenada en una ventana navegable de resultados mediante un contrato explícito de posición, tamaño, orden y metadata; no modifica la semántica lógica del conjunto consultado y no debe asumir que OFFSET/LIMIT es la única estrategia posible.**

Formalmente:

```text
Pagination
=
Query Window
+
Ordering Contract
+
Navigation State
+
Pagination Metadata
+
Consistency Semantics
```

y nunca:

```text
Pagination
=
LIMIT/OFFSET
```

---

# 2. Inicio del Bloque 19

Con este documento comienza:

```text
Block 19 — Pagination, Batch & Large Data

199 DATABASE_PAGINATION_SYSTEM
200 DATABASE_CURSOR_PAGINATION_SYSTEM
201 DATABASE_CHUNK_PROCESSING_SYSTEM
202 DATABASE_LAZY_COLLECTION_SYSTEM
203 DATABASE_BULK_INSERT_SYSTEM
204 DATABASE_BULK_UPDATE_SYSTEM
205 DATABASE_BULK_DELETE_SYSTEM
206 DATABASE_IMPORT_SYSTEM
207 DATABASE_EXPORT_SYSTEM
208 DATABASE_LARGE_DATASET_PROCESSING_SYSTEM
```

La separación conceptual será:

```text
Pagination
    navegación

Cursor Pagination
    navegación basada en posición lógica

Chunk Processing
    procesamiento incremental

Lazy Collection
    consumo diferido

Bulk Operations
    mutaciones masivas

Import / Export
    transferencia de datasets

Large Dataset Processing
    coordinación de workloads masivos
```

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Pagination
≠
Cursor Pagination
≠
Chunk Processing
≠
Streaming Result
≠
Lazy Collection
≠
Bulk Processing
≠
Database Cursor
```

Aunque puedan compartir infraestructura.

---

# 4. Pagination vs Cursor Pagination

Pagination general puede utilizar:

```text
page number
+
offset
```

mientras Cursor Pagination utiliza:

```text
ordered boundary values
+
continuation token
```

Ejemplo:

```text
Offset Pagination

page=5
perPage=20

OFFSET = 80
```

frente a:

```text
Cursor Pagination

after:
    created_at = 2026-08-01
    id = 782
```

El documento `200_DATABASE_CURSOR_PAGINATION_SYSTEM.md` especializará este segundo modelo.

---

# 5. Pagination vs Chunk Processing

Pagination responde:

> ¿Qué segmento de resultados debe mostrarse al consumidor?

Chunk Processing responde:

> ¿Cómo proceso incrementalmente un dataset completo?

Por tanto:

```text
Pagination
=
navigation

Chunking
=
processing
```

---

# 6. Pagination vs Streaming

Streaming:

```text
Query
 ↓
Cursor
 ↓
Row
 ↓
Consumer
```

Pagination:

```text
Query
 ↓
Window
 ↓
Finite result page
 ↓
Consumer
```

Una página normalmente es una colección finita materializada.

---

# 7. Pagination vs Lazy Collection

Lazy Collection representa evaluación/consumo diferido.

Pagination representa una frontera lógica de resultados.

Podrá existir:

```text
LazyCollection<Page>
```

o:

```text
Page<LazyItem>
```

pero son conceptos independientes.

---

# 8. Objetivos

El sistema deberá soportar:

1. page-number pagination;
2. offset pagination;
3. metadata de navegación;
4. total counts opcionales;
5. estrategias de count;
6. orden estable;
7. orden determinista;
8. Query Builder;
9. ORM;
10. Entity Query;
11. projections;
12. aggregates;
13. joins;
14. CTE;
15. distinct;
16. grouping;
17. Result Cache;
18. replicas;
19. sharding;
20. multitenancy;
21. seguridad;
22. resource governance;
23. diagnostics;
24. telemetry;
25. extensibilidad.

---

# 9. Arquitectura general

```text
Developer API
      │
      ▼
PaginationRequest
      │
      ▼
Pagination Planner
      │
      ├── Query Semantics
      ├── Ordering
      ├── Count Policy
      ├── Platform Capabilities
      ├── Distribution
      └── Resource Policy
      │
      ▼
PaginationPlan
      │
      ├── Data Query
      └── Count Query?
      │
      ▼
Query Engine
      │
      ▼
Optimizer
      │
      ▼
Compiler
      │
      ▼
Executor
      │
      ▼
PaginationAssembler
      │
      ▼
PageResult
```

---

# 10. Pagination como Query Semantics

Pagination deberá existir en el Query Model/AST.

Conceptualmente:

```text
SelectQuery
├── Projection
├── Source
├── Predicate
├── Grouping
├── Ordering
└── Pagination
```

No deberá añadirse mediante concatenación SQL tardía.

---

# 11. Pagination AST

Podrá existir:

```php
final readonly class PaginationNode
{
    public function __construct(
        public PaginationLimit $limit,
        public PaginationOffset $offset,
    ) {}
}
```

El Compiler correspondiente traducirá esta intención al dialecto de plataforma.

---

# 12. Query Builder no genera SQL

Ejemplo:

```php
$query
    ->limit(25)
    ->offset(50);
```

produce:

```text
Query Model / AST
```

no:

```text
"LIMIT 25 OFFSET 50"
```

La regla global permanece:

> Query Builder expresa intención; Compiler genera SQL.

---

# 13. API `paginate()`

`paginate()` será una operación de nivel superior.

Ejemplo:

```php
$page = User::query()
    ->orderBy('id')
    ->paginate(
        perPage: 20,
        page: 4,
    );
```

Conceptualmente:

```text
EntityQuery
 ↓
PaginationRequest
 ↓
PaginationPlan
 ↓
Data Query
 +
optional Count Query
 ↓
PageResult<User>
```

---

# 14. Page Number

```text
page >= 1
```

por defecto.

VoltStack no deberá mezclar silenciosamente índices:

```text
page 0
```

con:

```text
page 1
```

---

# 15. Offset

Para page-number pagination:

```text
offset
=
(page - 1) × perPage
```

---

# 16. Overflow

El cálculo deberá detectar overflow numérico.

No deberá permitirse que:

```text
(page - 1) × perPage
```

desborde silenciosamente.

---

# 17. Invalid page

Ejemplos:

```text
page = 0
page = -5
```

deberán rechazarse.

---

# 18. Invalid page size

También:

```text
perPage = 0
perPage = -1
```

serán inválidos.

---

# 19. Maximum page size

La configuración podrá establecer:

```text
defaultPerPage = 20
maxPerPage = 100
```

---

# 20. Resource governance

Una petición:

```text
perPage = 10,000,000
```

no deberá ejecutarse simplemente porque el cliente la solicitó.

---

# 21. PaginationPolicy

Propuesta:

```php
final readonly class PaginationPolicy
{
    public function __construct(
        public int $defaultPageSize,
        public int $maxPageSize,
        public PaginationCountPolicy $countPolicy,
        public PaginationOrderingPolicy $orderingPolicy,
    ) {}
}
```

---

# 22. PageResult

Contrato conceptual:

```php
/**
 * @template T
 */
final readonly class PageResult
{
    public function __construct(
        public array $items,
        public int $page,
        public int $perPage,
        public ?int $total,
        public ?int $lastPage,
        public bool $hasNextPage,
        public bool $hasPreviousPage,
        public PaginationMetadata $metadata,
    ) {}
}
```

---

# 23. `total` nullable

Muy importante:

```text
total
```

no siempre será conocido.

Por tanto:

```text
?int
```

es semánticamente correcto.

---

# 24. Unknown total

Nunca:

```text
UNKNOWN TOTAL
→
0
```

porque:

```text
Unknown
≠
Zero
```

---

# 25. Count strategies

VoltStack deberá distinguir varias estrategias.

```php
enum PaginationCountStrategy
{
    case EXACT;
    case ESTIMATED;
    case NONE;
    case DEFERRED;
    case CUSTOM;
}
```

---

# 26. EXACT

Ejecuta una consulta capaz de obtener el cardinal exacto del conjunto lógico.

---

# 27. ESTIMATED

Puede utilizar estadísticas o mecanismos de plataforma.

El resultado deberá marcarse como estimado.

---

# 28. NONE

No calcula total.

Resultado:

```text
total = unknown
lastPage = unknown
```

pero aún puede conocerse:

```text
hasNextPage
```

mediante técnicas como `limit + 1`.

---

# 29. DEFERRED

El count podrá calcularse solo si el consumidor lo solicita.

Conceptualmente:

```php
$page->total();
```

podría activar una operación diferida si el contrato de resultado lo permite.

---

# 30. Deferred I/O explícito

No deberá ocultarse I/O inesperado en una propiedad aparentemente trivial.

La API deberá comunicar que:

```text
total resolution
```

puede requerir consulta adicional.

---

# 31. Count Query

Una consulta:

```sql
SELECT users.*
FROM users
LEFT JOIN ...
WHERE ...
ORDER BY ...
```

no deberá transformarse ingenuamente en:

```sql
SELECT COUNT(*)
FROM users
LEFT JOIN ...
WHERE ...
ORDER BY ...
```

sin análisis semántico.

---

# 32. Count Planner

Deberá existir:

```text
PaginationCountPlanner
```

capaz de producir un count equivalente al conjunto lógico.

---

# 33. Count semantics

Debe considerar:

```text
DISTINCT
GROUP BY
HAVING
UNION
CTE
JOIN multiplicity
subqueries
projections
```

---

# 34. JOIN multiplicity

Ejemplo:

```text
User
LEFT JOIN Orders
```

puede producir:

```text
1 User
→
5 SQL rows
```

Por tanto:

```text
COUNT(rows)
≠
COUNT(users)
```

si el resultado paginado es entidad `User`.

---

# 35. ORM-aware count

Para Entity Query:

```text
Root Entity Cardinality
```

debe preservarse.

---

# 36. Count Query ≠ Data Query

Aunque deriven de la misma semántica:

```text
PaginationDataPlan
≠
PaginationCountPlan
```

---

# 37. Count ordering

El `ORDER BY` puede ser innecesario para un count.

El optimizer podrá eliminarlo cuando sea semánticamente seguro.

---

# 38. Count cache

Un total podrá integrarse con Result Cache si su consistencia y dependencias lo permiten.

Pero:

```text
Cached Count
≠
Current Database Cardinality
```

sin evidencia suficiente.

---

# 39. Exact total consistency

Incluso si el count es exacto al ejecutarse:

```text
COUNT query
```

y:

```text
data query
```

pueden observar estados diferentes bajo concurrencia.

---

# 40. Ejemplo

```text
T1:
COUNT = 100

T2:
INSERT row

T1:
SELECT page
→ sees 101-row state
```

dependiendo de transaction/isolation.

---

# 41. Snapshot consistency

Si el usuario requiere:

```text
count
+
page data
```

del mismo snapshot lógico, deberá utilizarse una estrategia transaccional/isolation compatible.

---

# 42. Pagination consistency policy

Propuesta:

```php
enum PaginationConsistency
{
    case BEST_EFFORT;
    case SNAPSHOT;
    case READ_YOUR_WRITES;
    case CUSTOM;
}
```

---

# 43. BEST_EFFORT

Permite count y data query bajo semántica normal de lectura.

Puede existir drift.

---

# 44. SNAPSHOT

Requiere que ambas observaciones pertenezcan al mismo snapshot cuando la plataforma y transacción lo soporten.

---

# 45. READ_YOUR_WRITES

Debe respetar:

```text
177 Read/Write Routing
180 Sticky Connection
192 Cache Consistency
```

---

# 46. Pagination no crea transaction automáticamente

No deberá ejecutar:

```text
BEGIN
```

solo porque el usuario llamó `paginate()` salvo policy explícita.

---

# 47. Ordenamiento

La paginación requiere especial atención al orden.

Una consulta:

```php
User::query()->paginate();
```

sin `ORDER BY` puede producir páginas no deterministas.

---

# 48. Ordering policy

VoltStack deberá poder exigir:

```text
EXPLICIT
INFER_STABLE
ALLOW_UNORDERED
```

---

# 49. Default recomendado

Para APIs de paginación:

```text
INFER_STABLE
```

o `EXPLICIT`, dependiendo del nivel API.

---

# 50. Stable ordering

Un orden:

```text
ORDER BY created_at
```

no necesariamente es único.

Ejemplo:

```text
User 10 → 10:00
User 11 → 10:00
User 12 → 10:00
```

---

# 51. Deterministic tie-breaker

Podrá transformarse semánticamente en:

```text
ORDER BY created_at, id
```

cuando metadata pruebe que `id` es un tie-breaker válido.

---

# 52. No modificar intención silenciosamente sin policy

Si VoltStack añade un tie-breaker, deberá ser parte de una política documentada/explainable.

---

# 53. Stable ≠ immutable

Incluso:

```text
ORDER BY id
```

es estable respecto al orden, pero el dataset puede cambiar por inserts/deletes.

---

# 54. Offset pagination drift

Supongamos:

```text
Page 1:
1 2 3 4 5

INSERT 0

Page 2 OFFSET 5:
5 6 7 8 9
```

`5` aparece dos veces.

---

# 55. Delete drift

```text
Page 1:
1 2 3 4 5

DELETE 2

Page 2 OFFSET 5:
7 8 9 10 11
```

`6` puede omitirse.

---

# 56. Esto no es bug del Compiler

Es una propiedad de offset pagination sobre datasets mutables.

---

# 57. Cursor Pagination

Cuando se requiera navegación robusta frente a cambios:

```text
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

proporcionará una alternativa.

---

# 58. Page metadata

Propuesta:

```php
final readonly class PaginationMetadata
{
    public function __construct(
        public PaginationStrategy $strategy,
        public PaginationCountMetadata $count,
        public PaginationOrderingMetadata $ordering,
        public PaginationConsistencyMetadata $consistency,
        public PaginationExecutionMetadata $execution,
    ) {}
}
```

---

# 59. Count metadata

Puede indicar:

```text
EXACT
ESTIMATED
UNKNOWN
DEFERRED
```

---

# 60. Last page

Solo puede calcularse cuando:

```text
total known
```

mediante:

```text
lastPage
=
ceil(total / perPage)
```

---

# 61. Empty dataset

Para:

```text
total = 0
```

VoltStack deberá definir claramente si:

```text
lastPage = 1
```

o:

```text
lastPage = 0
```

---

# 62. Convención recomendada

Para API page-based:

```text
page numbering starts at 1
```

y un resultado vacío solicitado en page 1 puede representar:

```text
page = 1
lastPage = 1
items = []
```

si se conoce el total.

---

# 63. Out-of-range page

Si:

```text
lastPage = 5
requestedPage = 20
```

el comportamiento deberá estar definido.

---

# 64. PageOverflowPolicy

```php
enum PageOverflowPolicy
{
    case RETURN_EMPTY;
    case THROW;
    case CLAMP_TO_LAST;
}
```

---

# 65. Default recomendado

```text
RETURN_EMPTY
```

para Database API.

`CLAMP_TO_LAST` puede producir navegación sorprendente y no debería ser default.

---

# 66. Has next

Con total conocido:

```text
hasNext
=
page < lastPage
```

---

# 67. Has next sin total

Puede solicitarse:

```text
perPage + 1
```

registros.

Ejemplo:

```text
perPage = 20
physical fetch = 21
```

Si llega el registro 21:

```text
hasNext = true
```

y se devuelve solo 20.

---

# 68. Lookahead

Esta estrategia será:

```text
PaginationLookahead
```

---

# 69. Lookahead ≠ total

Saber:

```text
hasNext = true
```

no implica saber:

```text
total
```

---

# 70. Previous page

En page-number pagination:

```text
hasPrevious
=
page > 1
```

aunque una página fuera de rango pueda requerir metadata adicional.

---

# 71. Pagination result shapes

Deberá funcionar con:

```text
Entity
Entity Collection
Scalar
Tuple
Projection
DTO
Associative Array
```

según Query/Hydration systems.

---

# 72. Pagination no hidrata

La arquitectura será:

```text
Pagination
 ↓
Query Engine
 ↓
Execution
 ↓
Hydration
 ↓
Page Assembly
```

Pagination no sustituye Hydrator.

---

# 73. ORM integration

Ejemplo:

```php
$page = $entityManager
    ->repository(User::class)
    ->query()
    ->where(...)
    ->paginate(25);
```

---

# 74. Model API

```php
$page = User::query()
    ->where('active', true)
    ->paginate(25);
```

Ambos deberán converger en el mismo Pagination Engine.

---

# 75. No dos motores

Nunca:

```text
ModelPaginator
→ SQL

EntityPaginator
→ different SQL
```

La arquitectura será:

```text
Model API ─────┐
               ▼
          Entity Query
               │
Repository ────┘
               │
               ▼
      Pagination Engine
               │
               ▼
          Query Engine
```

---

# 76. Relationship pagination

Caso:

```text
User.orders
```

podrá paginarse.

Pero:

```text
paginated relationship
≠
fully loaded relationship
```

---

# 77. Relationship coverage

Si se cargan 20 de 500 Orders:

```text
RelationshipCoverage = PARTIAL
```

no:

```text
COMPLETE
```

---

# 78. Partial collection

Nunca:

```text
20 loaded orders
→
collection complete
```

---

# 79. Eager loading

Paginar root entities con eager relations requiere preservar root cardinality.

---

# 80. Problema JOIN

```text
Users
JOIN Orders
LIMIT 20
```

puede limitar SQL rows, no Users.

---

# 81. Planner

Para:

```text
paginate Users
+
eager load Orders
```

podrá elegir:

```text
1. paginate User IDs
2. fetch Users
3. eager-load Orders
```

o estrategia equivalente.

---

# 82. Regla

> **Pagination boundary must apply to the logical root result, not accidentally to join-expanded physical rows.**

---

# 83. DISTINCT

Una consulta `DISTINCT` requiere que pagination/count preserve su semántica.

---

# 84. GROUP BY

Una consulta:

```text
GROUP BY customer_id
```

pagina grupos, no rows fuente.

---

# 85. Count de GROUP BY

El total esperado es:

```text
number of logical groups
```

cuando ése sea el result shape.

---

# 86. UNION

La paginación sobre:

```text
Query A
UNION
Query B
```

deberá aplicarse al set resultante.

---

# 87. CTE

CTE no cambia por sí sola las reglas.

La semántica del query final decide la paginación.

---

# 88. Window functions

Podrán utilizarse internamente para ciertas estrategias, pero:

```text
Pagination
≠
Window Function
```

---

# 89. Optimizer

El Optimizer podrá:

```text
remove irrelevant ordering from count
push safe limits
simplify count plan
select lookahead
```

si preserva semántica.

---

# 90. Limit pushdown

En consultas complejas/distribuidas puede ser útil.

Pero:

```text
PushDown(LIMIT)
```

solo será válido si no cambia el resultado lógico.

---

# 91. SQL Compiler

El Compiler traducirá:

```text
PaginationNode
```

a la sintaxis de:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 92. Platform capability

Nunca:

```php
if ($driver === 'mysql') {
    // pagination logic
}
```

en el Pagination Engine.

Preferir:

```text
PlatformCapabilities
+
Dialect Compiler
```

---

# 93. Prepared parameters

Cuando sea soportado/adecuado:

```text
limit
offset
```

podrán formar parte de parámetros compilados.

---

# 94. Limit validation

Incluso si el motor DB acepta valores grandes, Resource Governance podrá rechazarlos antes.

---

# 95. Offset limits

Un offset enorme:

```text
OFFSET 50,000,000
```

puede ser técnicamente válido y operacionalmente costoso.

---

# 96. OffsetCostPolicy

VoltStack podrá establecer:

```text
WARN
REJECT
ALLOW
SUGGEST_CURSOR
```

según umbral.

---

# 97. Ejemplo

```text
Requested:
page 500,001
perPage 100

Offset:
50,000,000

Diagnostic:
High offset pagination detected.
Consider cursor pagination.
```

---

# 98. No cambio automático a cursor

VoltStack no deberá convertir silenciosamente:

```text
page=500001
```

en cursor pagination porque cambia el contrato de navegación.

---

# 99. Query timeout

Pagination deberá respetar:

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM
```

---

# 100. Count timeout

Data query y count query podrán tener budgets distintos.

---

# 101. Count failure

Caso:

```text
Data query succeeds
Count query fails
```

no siempre debe destruir la página.

---

# 102. CountFailurePolicy

```php
enum CountFailurePolicy
{
    case FAIL_PAGE;
    case RETURN_WITH_UNKNOWN_TOTAL;
}
```

---

# 103. Default según API

Para UI convencional podría preferirse:

```text
RETURN_WITH_UNKNOWN_TOTAL
```

si el total no es contractual.

Para APIs estrictas:

```text
FAIL_PAGE
```

podrá configurarse.

---

# 104. Result status

Puede ser útil distinguir:

```text
COMPLETE
PARTIAL_METADATA
FAILED
UNKNOWN
```

---

# 105. `PARTIAL_METADATA`

Significa:

```text
items valid
total unavailable
```

No significa que los items estén parcialmente hidratados.

---

# 106. Cache integration

Pagination puede utilizar:

```text
188_DATABASE_RESULT_CACHE_SYSTEM
```

---

# 107. Cache key

Deberá incluir semánticamente:

```text
query identity
parameters
ordering
page/offset
page size
result shape
database domain
tenant
shard
consistency policy
generation information
```

---

# 108. Page 1 ≠ Page 2

Obviamente:

```text
CacheKey(Page1)
≠
CacheKey(Page2)
```

---

# 109. Cached page + count

Podrán ser:

```text
one combined artifact
```

o:

```text
separate data/count artifacts
```

según diseño.

---

# 110. Consistency

No deberá ocurrir:

```text
Page items from generation X
Total from incompatible generation Y
```

bajo políticas estrictas.

---

# 111. Cache invalidation

Mutaciones relevantes deberán invalidar:

```text
affected paginated result dependencies
```

mediante `191_DATABASE_CACHE_INVALIDATION_SYSTEM.md`.

---

# 112. Pagination cache explosion

Una consulta puede producir miles de page keys.

Por ello se favorecerán:

```text
dependency generations
query regions
bounded caching
```

sobre mantener enormes reverse indexes por cada página cuando no sea necesario.

---

# 113. Page cache policy

Podrá privilegiar:

```text
first N pages
```

porque suelen tener mayor reuse.

Esto es política de cache, no semántica de Pagination.

---

# 114. Read/write routing

Una página es una lectura.

Puede dirigirse a replica si:

```text
ReadIntent
ConsistencyRequirement
TransactionContext
StickyState
ReplicaFreshness
```

lo permiten.

---

# 115. Count y data query routing

Idealmente deberán utilizar dominios compatibles.

---

# 116. Problema

```text
Count → Replica A
Data → Replica B
```

con diferente lag puede producir metadata incoherente.

---

# 117. Routing policy

Para paginación con count exacto podrá requerirse:

```text
same replication group
same endpoint
same snapshot
```

dependiendo del consistency profile.

---

# 118. `Exact Count` ≠ `Snapshot Consistent`

Un count puede ser exacto respecto al estado visto por Replica A y aun no coincidir con Data Query en Replica B.

---

# 119. Sticky reads

Después de write:

```text
User creates Order
 ↓
redirect to paginated Orders
```

READ_YOUR_WRITES debe evitar que la nueva Order desaparezca temporalmente por leer una replica atrasada.

---

# 120. Sharding

Pagination distribuida introduce problemas adicionales.

---

# 121. Single-shard pagination

Si Partition Routing determina un solo shard:

```text
Pagination
→
Shard A
```

se comporta como consulta normal sobre ese shard.

---

# 122. Multi-shard pagination

Caso:

```text
ORDER BY created_at DESC
LIMIT 20 OFFSET 40
```

sobre:

```text
Shard A
Shard B
Shard C
```

no puede ejecutarse correctamente aplicando:

```text
OFFSET 40 LIMIT 20
```

independientemente a cada shard y concatenando resultados.

---

# 123. Global ordering

Se requiere:

```text
Shard A ─┐
Shard B ─┼─► Merge Coordinator ─► Global Page
Shard C ─┘
```

---

# 124. Distributed Pagination Planner

Podrá producir:

```text
ShardSubPlans
+
MergePlan
+
GlobalPaginationBoundary
```

---

# 125. Over-fetch

Para obtener una página global puede ser necesario recuperar más resultados de cada shard.

---

# 126. Offset amplification

Para:

```text
OFFSET 1,000,000
```

cross-shard pagination puede ser extremadamente costosa.

---

# 127. Cursor preferred

En distribución, Cursor Pagination será normalmente más escalable.

El documento 200 lo formalizará.

---

# 128. Global total

Un total exacto cross-shard puede requerir:

```text
COUNT shard A
+
COUNT shard B
+
COUNT shard C
```

si los conjuntos son disjuntos y la query permite esa composición.

---

# 129. Aggregate count

No todas las queries permiten sumar counts locales ingenuamente.

Ejemplo:

```text
DISTINCT email
```

cross-shard.

---

# 130. Distributed count planner

Deberá analizar si:

```text
local counts
→
composable global count
```

es válido.

---

# 131. UNKNOWN ≠ SUM

Si no puede probarse composabilidad:

```text
UNKNOWN
```

no deberá transformarse en suma de counts.

---

# 132. Shard failure

Si:

```text
Shard A success
Shard B success
Shard C failure
```

una página global exacta normalmente no puede afirmarse completa.

---

# 133. Partial distributed pages

Solo podrán devolverse bajo una policy explícita.

Nunca como página normal sin metadata.

---

# 134. Multitenancy

Tenant context deberá formar parte de:

```text
query context
pagination context
cache identity
routing
```

---

# 135. Cross-tenant pagination

Por defecto:

```text
FORBIDDEN
```

salvo una operación administrativa explícitamente autorizada.

---

# 136. Tenant ≠ page partition

No utilizar tenant como mecanismo de paginación.

---

# 137. Security

Los parámetros externos:

```text
page
perPage
sort
direction
```

deben validarse.

---

# 138. Sort field injection

Nunca:

```php
$orderBy = $_GET['sort'];

$query->rawOrderBy($orderBy);
```

---

# 139. Sort whitelist

Preferir:

```php
$pagination->allowSorts([
    'name',
    'created_at',
    'email',
]);
```

---

# 140. Sort alias

La API podrá mapear:

```text
created
→
users.created_at
```

sin exponer estructura DB.

---

# 141. Direction

Solo:

```text
ASC
DESC
```

u opciones tipadas soportadas.

---

# 142. Raw expressions

Ordenamiento raw deberá seguir las reglas de:

```text
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM
```

---

# 143. Authorization

Paginar no deberá alterar el conjunto autorizado.

Arquitectura:

```text
Authorization / Query Scope
            ↓
      Logical Query
            ↓
        Pagination
```

No:

```text
paginate all
 ↓
filter unauthorized rows in PHP
```

---

# 144. Razón

Filtrar después de paginar puede:

```text
leak total counts
produce sparse pages
expose existence
break navigation
```

---

# 145. Sensitive totals

Incluso `COUNT` puede revelar información.

Ejemplo:

```text
"Hay 1 usuario con ese email"
```

puede convertirse en un side channel.

---

# 146. Count authorization

La política de seguridad podrá impedir totals incluso si permite listar resultados.

---

# 147. Pagination metadata security

No asumir que:

```text
total
lastPage
```

siempre puede exponerse al cliente.

---

# 148. HTTP integration

Database Pagination no dependerá de HTTP.

---

# 149. Framework HTTP layer

Podrá mapear:

```text
?page=3&per_page=25
```

a:

```text
PaginationRequest
```

pero Database no leerá `$_GET`.

---

# 150. URL generation

Generar:

```text
next URL
previous URL
```

pertenece a HTTP/UI integration.

Database devuelve metadata semántica.

---

# 151. API serialization

Podrá producirse:

```json
{
  "data": [],
  "meta": {
    "page": 3,
    "perPage": 25,
    "total": 240,
    "lastPage": 10
  }
}
```

pero serialization pertenece a capas superiores.

---

# 152. SPA integration

VoltStack podrá integrar PageResult con su runtime SPA sin acoplar Database al frontend.

---

# 153. PageResult immutable

Una vez construido:

```text
PageResult
```

deberá considerarse immutable.

---

# 154. Page items

Los objetos Entity contenidos pueden ser mutables según modelo ORM.

La inmutabilidad del PageResult no convierte entidades en immutable.

---

# 155. PaginationRequest

Propuesta:

```php
final readonly class PaginationRequest
{
    public function __construct(
        public int $page,
        public int $perPage,
        public PaginationCountStrategy $countStrategy,
        public PaginationConsistency $consistency,
    ) {}
}
```

---

# 156. PaginationContext

Podrá contener:

```text
DatabaseContext
TransactionContext
ReadIntent
TenantContext
ShardRoutingContext
CachePolicy
ResourceBudget
TelemetryContext
CancellationToken
```

---

# 157. No estado global

Nunca:

```php
Paginator::$currentPage
```

como dependencia interna global de Database.

---

# 158. Laravel-style convenience

Una capa superior podrá proporcionar ergonomía como:

```php
User::paginate(20);
```

pero internamente deberá resolver explícitamente `PaginationRequest`.

---

# 159. Current page resolution

Resolver automáticamente `page` desde HTTP pertenece al framework HTTP integration, no al core Database.

---

# 160. CLI

En CLI:

```php
$query->paginate(
    perPage: 100,
    page: 5,
);
```

debe funcionar sin HTTP.

---

# 161. Queue workers

Igualmente debe funcionar dentro de Jobs.

---

# 162. Persistent runtime

En FrankenPHP:

```text
Request A:
page = 3

Request B:
page = 1
```

B nunca deberá heredar page 3.

---

# 163. RoadRunner/OpenSwoole

Misma regla:

```text
worker reuse
≠
pagination request state reuse
```

---

# 164. Coroutine isolation

Cada coroutine/fiber tendrá su propio PaginationContext cuando exista mutable execution state.

---

# 165. Pagination Planner

Responsabilidades:

```text
validate request
resolve ordering
determine count strategy
derive data query
derive count query
evaluate distribution
evaluate consistency
apply resource policy
produce PaginationPlan
```

---

# 166. Planner no ejecuta

Siempre:

```text
Planner
≠
Executor
```

---

# 167. PaginationPlan

Propuesta:

```php
final readonly class PaginationPlan
{
    public function __construct(
        public QueryModel $dataQuery,
        public ?QueryModel $countQuery,
        public PaginationWindow $window,
        public PaginationOrderingPlan $ordering,
        public PaginationCountPlan $count,
        public PaginationConsistencyPlan $consistency,
    ) {}
}
```

---

# 168. Data Query

Podrá utilizar:

```text
offset
limit
lookahead
```

según plan.

---

# 169. Count Query

Puede ser:

```text
none
exact
estimated
deferred
```

---

# 170. Query optimizer independence

Pagination Planner podrá preparar la intención.

Query Optimizer sigue teniendo autoridad sobre optimizaciones seguras.

---

# 171. Pagination assembler

Responsabilidades:

```text
accept hydrated items
trim lookahead
combine count metadata
compute navigation metadata
build PageResult
```

---

# 172. Assembler no consulta DB

No deberá ejecutar count ocultamente salvo que explícitamente forme parte de un deferred resolver diseñado para ello.

---

# 173. Count overflow

`COUNT(*)` puede exceder `PHP_INT_MAX` en teoría.

El Type System deberá definir representación adecuada o producir error controlado.

---

# 174. Large page numbers

Igualmente deben validarse antes de convertir a offsets incompatibles con la plataforma.

---

# 175. Pagination limits capabilities

Platform puede imponer límites sobre:

```text
LIMIT
OFFSET
parameterization
integer width
```

El plan deberá validarse contra capabilities.

---

# 176. Cancellation

Si se cancela:

```text
count query
```

después de obtener data query, la policy decide:

```text
return page without total
```

o:

```text
cancel entire pagination
```

---

# 177. Retry

Una pagination operation puede involucrar dos queries.

Retry deberá respetar:

```text
86_DATABASE_EXECUTION_RETRY_SYSTEM
```

y consistencia.

---

# 178. No replay arbitrario

Si la operación está dentro de transaction:

```text
statement retry
```

no deberá violar transaction semantics.

---

# 179. Error hierarchy

Propuesta:

```text
DatabaseException
└── PaginationException
    ├── InvalidPaginationRequestException
    │   ├── InvalidPageException
    │   └── InvalidPageSizeException
    ├── PaginationLimitExceededException
    ├── PaginationOffsetOverflowException
    ├── PaginationOrderingException
    │   ├── MissingPaginationOrderException
    │   └── UnstablePaginationOrderException
    ├── PaginationCountException
    ├── PaginationPlanningException
    ├── PaginationDistributionException
    ├── PaginationConsistencyException
    ├── PaginationSecurityException
    └── PaginationResourceException
```

---

# 180. Directory structure

Propuesta:

```text
src/Quantum/Database/Pagination/
│
├── PaginationRequest.php
├── PaginationContext.php
├── PaginationStrategy.php
├── PaginationPolicy.php
├── PaginationConsistency.php
│
├── Page/
│   ├── PageResult.php
│   ├── PageNumber.php
│   ├── PageSize.php
│   ├── PageMetadata.php
│   └── PageOverflowPolicy.php
│
├── Window/
│   ├── PaginationWindow.php
│   ├── PaginationLimit.php
│   ├── PaginationOffset.php
│   └── PaginationLookahead.php
│
├── Ordering/
│   ├── PaginationOrderingPolicy.php
│   ├── PaginationOrderingResolver.php
│   ├── PaginationOrderingPlan.php
│   └── PaginationOrderingMetadata.php
│
├── Count/
│   ├── PaginationCountStrategy.php
│   ├── PaginationCountPlanner.php
│   ├── PaginationCountPlan.php
│   ├── PaginationCountMetadata.php
│   └── CountFailurePolicy.php
│
├── Planning/
│   ├── PaginationPlanner.php
│   └── PaginationPlan.php
│
├── Assembly/
│   └── PaginationAssembler.php
│
├── Distribution/
│   ├── DistributedPaginationPlanner.php
│   ├── DistributedPaginationPlan.php
│   └── PaginationMergeCoordinator.php
│
├── Resource/
│   ├── PaginationResourcePolicy.php
│   └── OffsetCostPolicy.php
│
├── Diagnostics/
│   ├── PaginationInspector.php
│   └── PaginationExplainer.php
│
├── Telemetry/
│   └── PaginationTelemetry.php
│
└── Exception/
    └── ...
```

---

# 181. Relación con Query AST

Los nodos estructurales de query deberán permanecer en el módulo Query correspondiente.

Por ejemplo:

```text
Query/AST/PaginationNode
```

mientras:

```text
Pagination/
```

contendrá políticas, planificación y resultado de alto nivel.

---

# 182. No duplicar Query Model

Pagination no tendrá un segundo lenguaje de consultas.

---

# 183. Telemetry

Eventos conceptuales:

```text
PaginationPlanningStarted
PaginationPlanCreated
PaginationExecutionStarted
PaginationDataQueryCompleted
PaginationCountQueryCompleted
PaginationCountFailed
HighOffsetDetected
PaginationCompleted
PaginationFailed
```

---

# 184. Métricas

```text
db.pagination.requests
db.pagination.duration
db.pagination.page_size
db.pagination.count.duration
db.pagination.count.failures
db.pagination.high_offset
db.pagination.lookahead
db.pagination.distributed
```

---

# 185. Cardinalidad

No utilizar:

```text
page number
raw query
user ID
tenant ID
cursor
```

como metric labels de alta cardinalidad.

---

# 186. Diagnostics

API conceptual:

```php
DB::pagination()->explain(
    User::query()
        ->where('active', true)
        ->orderBy('created_at'),
    page: 5000,
    perPage: 100,
);
```

Resultado:

```text
PAGINATION PLAN

Strategy:
    OFFSET

Page:
    5000

Per Page:
    100

Offset:
    499900

Ordering:
    created_at DESC
    id ASC [inferred tie-breaker]

Count:
    EXACT

Consistency:
    BEST_EFFORT

Lookahead:
    disabled

Distribution:
    single shard

Cost Warning:
    HIGH OFFSET

Suggestion:
    Cursor pagination may be more efficient.
```

---

# 187. Explain Count

Podrá mostrar:

```text
COUNT PLAN

Logical Result:
    User

Original Query:
    User + Orders eager relationship

Count Strategy:
    ROOT_ENTITY_COUNT

Removed:
    ORDER BY

Join Handling:
    non-cardinality-changing join removed

Expected:
    exact
```

---

# 188. Security diagnostics

No deberá imprimir valores sensibles de query parameters por defecto.

---

# 189. Testing matrix

| Área | Caso |
|---|---|
| Basic | page 1 |
| Basic | middle page |
| Basic | last page |
| Empty | empty dataset |
| Overflow | page beyond last |
| Validation | page 0 |
| Validation | negative page |
| Validation | perPage 0 |
| Resource | excessive perPage |
| Offset | arithmetic overflow |
| Ordering | explicit stable |
| Ordering | duplicate sort values |
| Ordering | inferred tie-breaker |
| Count | exact |
| Count | none |
| Count | estimated |
| Count | failure |
| Count | distinct |
| Count | group by |
| Count | joins |
| Count | union |
| ORM | entities |
| ORM | eager relations |
| Relationships | partial collection |
| Cache | page key |
| Cache | invalidation |
| Replica | lag |
| RYW | sticky writer |
| Shard | single |
| Shard | distributed |
| Security | sort whitelist |
| Runtime | request isolation |
| Cancellation | count cancelled |

---

# 190. Architectural invariants

## DB-PAGE-001
Pagination será una operación semántica.

## DB-PAGE-002
Pagination no será sinónimo de LIMIT/OFFSET.

## DB-PAGE-003
Pagination será distinta de Cursor Pagination.

## DB-PAGE-004
Pagination será distinta de Chunk Processing.

## DB-PAGE-005
Pagination será distinta de Streaming Result.

## DB-PAGE-006
Pagination será distinta de Lazy Collection.

## DB-PAGE-007
Pagination será distinta de Bulk Processing.

## DB-PAGE-008
Pagination no generará SQL directamente.

## DB-PAGE-009
Query Builder no generará SQL de paginación.

## DB-PAGE-010
Compiler será responsable de SQL específico de plataforma.

## DB-PAGE-011
Pagination podrá representarse en Query Model/AST.

## DB-PAGE-012
Page numbering será explícito.

## DB-PAGE-013
Default page numbering comenzará en 1.

## DB-PAGE-014
Page 0 será inválida bajo page-number semantics.

## DB-PAGE-015
Negative pages serán inválidas.

## DB-PAGE-016
Page size deberá ser positiva.

## DB-PAGE-017
Maximum page size será gobernable.

## DB-PAGE-018
Offset arithmetic deberá detectar overflow.

## DB-PAGE-019
Huge offsets podrán activar policy.

## DB-PAGE-020
Huge offset no será convertido automáticamente a cursor.

## DB-PAGE-021
PageResult será un resultado tipado.

## DB-PAGE-022
Unknown total no será cero.

## DB-PAGE-023
Total podrá ser nullable/unknown.

## DB-PAGE-024
LastPage podrá ser unknown.

## DB-PAGE-025
Exact count será distinto de estimated count.

## DB-PAGE-026
No-count será una estrategia válida.

## DB-PAGE-027
Deferred count será explícito.

## DB-PAGE-028
Deferred I/O no deberá ocultarse.

## DB-PAGE-029
Count Query será distinta de Data Query.

## DB-PAGE-030
Count Planner preservará cardinalidad lógica.

## DB-PAGE-031
JOIN row count no será asumido igual a root entity count.

## DB-PAGE-032
DISTINCT será preservado en count semantics.

## DB-PAGE-033
GROUP BY será preservado en count semantics.

## DB-PAGE-034
UNION será preservado en count semantics.

## DB-PAGE-035
CTE será analizada semánticamente.

## DB-PAGE-036
ORDER BY podrá eliminarse del count solo cuando sea seguro.

## DB-PAGE-037
Exact Count no implicará snapshot consistency.

## DB-PAGE-038
Count y data pueden observar estados diferentes.

## DB-PAGE-039
Snapshot consistency será una policy explícita.

## DB-PAGE-040
Pagination no abrirá transaction automáticamente por defecto.

## DB-PAGE-041
Pagination deberá considerar ordering.

## DB-PAGE-042
Unordered pagination podrá ser rechazada por policy.

## DB-PAGE-043
Stable ordering no implicará unique ordering.

## DB-PAGE-044
Tie-breakers podrán inferirse solo con metadata suficiente.

## DB-PAGE-045
Inferred ordering deberá ser explainable.

## DB-PAGE-046
Stable ordering no implicará immutable dataset.

## DB-PAGE-047
Offset pagination podrá sufrir drift.

## DB-PAGE-048
Offset drift no será ocultado.

## DB-PAGE-049
Cursor Pagination será alternativa explícita.

## DB-PAGE-050
Lookahead podrá determinar hasNext sin total.

## DB-PAGE-051
Lookahead no determinará total.

## DB-PAGE-052
Pagination funcionará con entities.

## DB-PAGE-053
Pagination funcionará con scalars.

## DB-PAGE-054
Pagination funcionará con tuples.

## DB-PAGE-055
Pagination funcionará con projections.

## DB-PAGE-056
Pagination no será Hydrator.

## DB-PAGE-057
Model API y Repository usarán el mismo Pagination Engine.

## DB-PAGE-058
No existirá un segundo paginator que genere SQL para Active Record.

## DB-PAGE-059
Paginated relationship será PARTIAL.

## DB-PAGE-060
Partial relationship no será marcada COMPLETE.

## DB-PAGE-061
Eager loading no deberá cambiar root pagination cardinality.

## DB-PAGE-062
LIMIT sobre joined physical rows no deberá sustituir root entity pagination cuando cambie semántica.

## DB-PAGE-063
Optimizer podrá optimizar pagination solo preservando semántica.

## DB-PAGE-064
Platform-specific pagination pertenecerá al Compiler/Dialect.

## DB-PAGE-065
Pagination Engine no contendrá vendor conditionals como arquitectura principal.

## DB-PAGE-066
Version no será Capability.

## DB-PAGE-067
Page size será validada antes de ejecución.

## DB-PAGE-068
Query timeout será respetado.

## DB-PAGE-069
Count podrá tener budget separado.

## DB-PAGE-070
Count failure tendrá policy explícita.

## DB-PAGE-071
Items válidos con count fallido podrán representarse como PARTIAL_METADATA cuando policy lo permita.

## DB-PAGE-072
PARTIAL_METADATA no significará partial hydration.

## DB-PAGE-073
PageResult podrá integrarse con Result Cache.

## DB-PAGE-074
Cache identity incluirá pagination boundary.

## DB-PAGE-075
Page 1 y Page 2 tendrán identidades de cache distintas.

## DB-PAGE-076
Cache consistency será evaluada.

## DB-PAGE-077
Cached count no será considerado actual sin evidencia.

## DB-PAGE-078
Pagination invalidation usará Semantic Cache Invalidation.

## DB-PAGE-079
Pagination no dependerá de FLUSHALL.

## DB-PAGE-080
Pagination cache deberá ser bounded.

## DB-PAGE-081
Read routing respetará consistency requirements.

## DB-PAGE-082
Active transaction podrá forzar writer.

## DB-PAGE-083
Sticky state será respetado.

## DB-PAGE-084
Replica lag será considerado.

## DB-PAGE-085
Count exacto en Replica A no garantizará coherencia con Data en Replica B.

## DB-PAGE-086
READ_YOUR_WRITES deberá poder dominar replica routing.

## DB-PAGE-087
Single-shard pagination será soportada.

## DB-PAGE-088
Multi-shard pagination requerirá global merge semantics.

## DB-PAGE-089
Offset no se aplicará independientemente a cada shard como resultado global.

## DB-PAGE-090
Distributed pagination podrá requerir over-fetch.

## DB-PAGE-091
Distributed high offsets podrán ser costosos.

## DB-PAGE-092
Distributed count solo sumará local counts cuando sea semánticamente composable.

## DB-PAGE-093
UNKNOWN composability no significará SUM.

## DB-PAGE-094
Shard failure no producirá página completa falsa.

## DB-PAGE-095
Partial distributed page será explícita.

## DB-PAGE-096
Tenant context será preservado.

## DB-PAGE-097
Cross-tenant pagination será prohibida por defecto.

## DB-PAGE-098
Tenant no será Shard.

## DB-PAGE-099
External sort fields serán validados.

## DB-PAGE-100
Sort input no se concatenará como SQL raw.

## DB-PAGE-101
Sort whitelist será soportada.

## DB-PAGE-102
Authorization se aplicará antes de pagination boundary.

## DB-PAGE-103
Unauthorized rows no se filtrarán simplemente después de paginar.

## DB-PAGE-104
Totals podrán ser security-sensitive.

## DB-PAGE-105
Count authorization podrá diferir de row-read authorization.

## DB-PAGE-106
Database Pagination no dependerá de HTTP.

## DB-PAGE-107
Database Pagination no leerá globals HTTP.

## DB-PAGE-108
URL generation no será responsabilidad de Database.

## DB-PAGE-109
Serialization no será responsabilidad del core Pagination.

## DB-PAGE-110
SPA integration será una capa superior.

## DB-PAGE-111
PageResult será immutable como container.

## DB-PAGE-112
PageResult immutability no implicará immutable entities.

## DB-PAGE-113
PaginationRequest será scoped.

## DB-PAGE-114
PaginationContext será scoped.

## DB-PAGE-115
Current page no será estado global mutable.

## DB-PAGE-116
CLI podrá utilizar Pagination.

## DB-PAGE-117
Jobs podrán utilizar Pagination.

## DB-PAGE-118
FrankenPHP request reuse no reutilizará pagination state.

## DB-PAGE-119
RoadRunner worker reuse no reutilizará pagination state.

## DB-PAGE-120
OpenSwoole worker reuse no reutilizará pagination state.

## DB-PAGE-121
Coroutine contexts estarán aislados.

## DB-PAGE-122
Pagination Planner no ejecutará queries.

## DB-PAGE-123
PaginationPlan será immutable después de validación.

## DB-PAGE-124
PaginationAssembler no será Query Executor.

## DB-PAGE-125
Pagination no creará un segundo Query Model.

## DB-PAGE-126
Pagination no creará un segundo ORM.

## DB-PAGE-127
Pagination no creará un segundo Hydration System.

## DB-PAGE-128
Pagination no creará un segundo Cache System.

## DB-PAGE-129
Pagination no creará un segundo Distribution System.

## DB-PAGE-130
Pagination no creará un segundo Security System.

## DB-PAGE-131
Cancellation será soportable.

## DB-PAGE-132
Cancellation de count podrá degradar metadata solo bajo policy.

## DB-PAGE-133
Retry respetará transaction semantics.

## DB-PAGE-134
Statement retry no violará transaction outcome.

## DB-PAGE-135
Telemetry tendrá bounded cardinality.

## DB-PAGE-136
Telemetry no expondrá parámetros sensibles.

## DB-PAGE-137
High offsets serán observables.

## DB-PAGE-138
Pagination plans serán explainable.

## DB-PAGE-139
Count plans serán explainable.

## DB-PAGE-140
Ordering inference será explainable.

## DB-PAGE-141
Resource policies serán configurables.

## DB-PAGE-142
Per-page limits podrán variar por context/policy.

## DB-PAGE-143
Security policy podrá imponer un máximo más estricto.

## DB-PAGE-144
Count puede omitirse por seguridad.

## DB-PAGE-145
Count puede omitirse por rendimiento.

## DB-PAGE-146
No-count pagination seguirá siendo navegable mediante lookahead cuando sea posible.

## DB-PAGE-147
Page overflow behavior será explícito.

## DB-PAGE-148
RETURN_EMPTY será un comportamiento soportado.

## DB-PAGE-149
CLAMP no será default recomendado.

## DB-PAGE-150
Empty dataset semantics serán deterministas.

## DB-PAGE-151
Database capabilities limitarán physical pagination strategy.

## DB-PAGE-152
Logical pagination contract permanecerá platform-neutral.

## DB-PAGE-153
MySQL será soportado.

## DB-PAGE-154
MariaDB será soportado.

## DB-PAGE-155
PostgreSQL será soportado.

## DB-PAGE-156
SQLite será soportado.

## DB-PAGE-157
Pagination podrá utilizar prepared statements.

## DB-PAGE-158
Limit/offset binding dependerá de capabilities/compiler.

## DB-PAGE-159
Page metadata no afirmará conocimiento no observado.

## DB-PAGE-160
UNKNOWN permanecerá UNKNOWN.

---

# 191. Modelo formal

Sea:

```text
Q = Logical Query
O = Ordering
p = Page Number
s = Page Size
```

Entonces:

```text
offset = (p - 1) × s
```

y:

```text
Page(Q,p,s)
=
Window(
    Order(Q,O),
    offset,
    s
)
```

siempre que la estrategia utilizada sea Offset Pagination.

---

# 192. Condición de orden determinista

Sea:

```text
K = ordered key tuple
```

Un orden será determinista para filas distintas cuando:

```text
∀ a,b ∈ Result(Q):

a ≠ b
⇒
K(a) ≠ K(b)
```

o exista un tie-breaker adicional que produzca esa propiedad.

---

# 193. Metadata de página

Con total exacto:

```text
lastPage
=
max(
    1,
    ceil(total / perPage)
)
```

bajo la convención de páginas basadas en 1.

---

# 194. Sin total

Si:

```text
CountStrategy = NONE
```

entonces:

```text
total = UNKNOWN
lastPage = UNKNOWN
```

pero:

```text
hasNext
```

puede obtenerse mediante lookahead.

---

# 195. Correctitud lógica

La página deberá satisfacer:

```text
PageItems
⊆
LogicalResult(Q)
```

y respetar:

```text
Ordering(PageItems)
=
Ordering(Q)
```

---

# 196. Correctitud ORM

Para paginación de root entities:

```text
PaginationBoundary
```

deberá aplicarse sobre:

```text
LogicalRootEntities
```

no necesariamente sobre:

```text
PhysicalJoinedRows
```

---

# 197. Consistencia observacional

```text
PageDataObservation
```

y:

```text
CountObservation
```

son observaciones distintas salvo que una política garantice snapshot común.

Por tanto:

```text
Exact(Data)
∧
Exact(Count)
```

no implica:

```text
SameSnapshot(Data, Count)
```

---

# 198. Coste conceptual

Para Offset Pagination tradicional:

```text
Cost
≈
scan/skip(offset)
+
fetch(pageSize)
```

dependiendo del plan DB.

Por ello:

```text
offset ↑
⇒
potential cost ↑
```

aunque el comportamiento real dependa del motor, índices y query plan.

---

# 199. Arquitectura final

```text
                   Developer API
                        │
           ┌────────────┴─────────────┐
           ▼                          ▼
      Model API                  Repository
           │                          │
           └────────────┬─────────────┘
                        ▼
                 Entity / Query API
                        │
                        ▼
                PaginationRequest
                        │
                        ▼
                PaginationPlanner
             ┌──────────┼───────────┐
             │          │           │
             ▼          ▼           ▼
         Ordering      Count    Consistency
             │          │           │
             └──────────┼───────────┘
                        ▼
                 PaginationPlan
                 ┌──────┴───────┐
                 ▼              ▼
             Data Query      Count Query
                 │              │
                 └──────┬───────┘
                        ▼
                   Query Engine
                        │
                        ▼
                     Planner
                        │
                        ▼
                    Compiler
                        │
                        ▼
                    Executor
                        │
                        ▼
                    Hydration
                        │
                        ▼
               PaginationAssembler
                        │
                        ▼
                    PageResult
```

---

# 200. Regla maestra final

> **Pagination en VoltStack será una operación de navegación sobre el resultado lógico de una consulta, no una manipulación textual de SQL.**

La arquitectura deberá preservar:

```text
Pagination
    decides navigation semantics

Query Engine
    represents query semantics

Optimizer
    optimizes safely

Compiler
    generates platform SQL

Executor
    executes

Hydrator
    creates result shapes

PaginationAssembler
    creates navigation metadata
```

Y siempre:

```text
Page
≠
Chunk
≠
Cursor
≠
Stream
```

además de:

```text
Exact Count
≠
Same Snapshot
```

y:

```text
Stable Order
≠
Immutable Dataset
```

y:

```text
Unknown Total
≠
Zero
```

---

# 201. Resultado arquitectónico

Con `199_DATABASE_PAGINATION_SYSTEM.md`, VoltStack establece la capa general que permitirá implementar:

```php
User::query()
    ->where('active', true)
    ->orderBy('created_at', 'desc')
    ->paginate(25);
```

sin acoplar esta ergonomía a:

```text
Laravel-style Active Record
HTTP request globals
PDO
LIMIT/OFFSET SQL strings
a specific database vendor
```

La arquitectura queda preparada para:

```text
simple applications
SPA interfaces
REST APIs
admin panels
large datasets
replicas
shards
multitenancy
persistent runtimes
```

manteniendo una separación estricta entre:

```text
Navigation
Query Semantics
Execution
Hydration
Consistency
Distribution
```

---

# 202. Siguiente documento

```text
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

El siguiente documento especializará la arquitectura para **keyset/cursor pagination**, incluyendo:

```text
Cursor Pagination
├── cursor model
├── cursor token
├── keyset pagination
├── continuation boundaries
├── compound sort keys
├── ASC / DESC traversal
├── next cursor
├── previous cursor
├── deterministic ordering
├── unique tie-breakers
├── nullable sort values
├── cursor encoding
├── cursor signing
├── cursor versioning
├── query fingerprint binding
├── tenant/shard binding
├── stale cursor semantics
├── distributed cursors
├── replica consistency
├── cache interaction
└── security
```

y establecerá especialmente:

```text
Cursor
≠
Offset
≠
Database Cursor
≠
Raw Primary Key
```

con una regla central:

> **El cursor deberá representar una posición verificable dentro de un orden lógico estable, no simplemente un OFFSET codificado.**