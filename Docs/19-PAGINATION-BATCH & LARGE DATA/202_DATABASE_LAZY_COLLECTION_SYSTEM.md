# 202_DATABASE_LAZY_COLLECTION_SYSTEM.md

# VoltStack Quantum Database
## Database Lazy Collection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 202 — Database Lazy Collection System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `201_DATABASE_CHUNK_PROCESSING_SYSTEM.md`  
**Siguiente documento:** `203_DATABASE_BULK_INSERT_SYSTEM.md`

---

# 1. Propósito

`Database Lazy Collection System` define la arquitectura mediante la cual VoltStack podrá representar y consumir datasets potencialmente grandes a través de una interfaz iterable y componible, retrasando ejecución, lectura y materialización hasta que los elementos sean realmente solicitados.

Ejemplo:

```php id="8yjkx2"
$users = User::query()
    ->where('active', true)
    ->orderBy('id')
    ->lazy();
```

Consumo:

```php id="7hn3ka"
foreach ($users as $user) {
    // Procesamiento incremental
}
```

Pipeline:

```php id="w3p0ji"
$emails = User::query()
    ->where('active', true)
    ->lazy()
    ->filter(
        fn (User $user) => $user->canReceiveEmail()
    )
    ->map(
        fn (User $user) => $user->email()
    )
    ->take(1000);
```

La regla central será:

> **Una Lazy Collection en VoltStack representa un pipeline de consumo diferido sobre una fuente de datos potencialmente grande; difiere ejecución y materialización, pero nunca oculta que el consumo puede implicar I/O, recursos, hidratación, transacciones, consistencia y ciclos de vida de conexión.**

Formalmente:

```text id="4gkie0"
LazyCollection
=
DeferredSource
+
LazyPipeline
+
IterationStrategy
+
ResourceLifecycle
+
ConsumptionContract
```

Nunca:

```text id="cfq7i9"
LazyCollection
=
LoadedCollection
```

ni:

```text id="pmmk4e"
LazyCollection
=
QueryBuilder
```

---

# 2. Posición dentro del Bloque 19

```text id="9s01pm"
199 Pagination
      ↓
finite navigable windows

200 Cursor Pagination
      ↓
logical navigation boundaries

201 Chunk Processing
      ↓
bounded processing traversal

202 Lazy Collection
      ↓
deferred incremental consumption

203 Bulk Insert
204 Bulk Update
205 Bulk Delete
206 Import
207 Export
208 Large Dataset Processing
```

Lazy Collection será una capa de Developer Experience y composición construida sobre infraestructura ya definida.

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text id="cmpl7i"
Lazy Collection
≠
Loaded Collection
≠
Query Builder
≠
Chunk
≠
Cursor Pagination
≠
Result Cursor
≠
Streaming Result
≠
Generator PHP
≠
ORM Relationship Collection
```

Aunque pueda implementar internamente algunas operaciones mediante estas infraestructuras.

---

# 4. Lazy Collection vs Loaded Collection

Una colección normal:

```php id="9tk7up"
$users = User::query()->get();
```

normalmente implica:

```text id="jt445b"
execute query
↓
fetch all
↓
hydrate all
↓
materialize collection
```

Una Lazy Collection:

```php id="9wmhdg"
$users = User::query()->lazy();
```

implica:

```text id="ne0uhg"
define lazy source
↓
no data necessarily fetched yet
↓
iteration begins
↓
fetch bounded data
↓
yield
```

---

# 5. Lazy ≠ zero execution

`lazy()` podrá construir la colección sin ejecutar inmediatamente.

Pero al consumir:

```php id="3s6hc7"
foreach ($users as $user)
```

sí puede producir I/O.

---

# 6. Lazy Collection vs Query Builder

Query Builder representa:

```text id="815xas"
query intent
```

Lazy Collection representa:

```text id="n5s5j0"
consumption pipeline
```

Por tanto:

```text id="5cezof"
QueryBuilder
→
LazyCollection
```

pero no son el mismo objeto conceptual.

---

# 7. Query mutation after lazy()

Caso:

```php id="vqj2q3"
$query = User::query();

$lazy = $query->lazy();

$query->where('admin', true);
```

La semántica deberá ser clara.

---

# 8. Recomendación

`lazy()` deberá capturar un:

```text id="6qd7dl"
immutable query snapshot/model
```

para evitar que mutaciones posteriores externas cambien silenciosamente la fuente.

---

# 9. Lazy Collection vs Chunk

Chunk:

```text id="r6k4na"
bounded group
```

Lazy Collection:

```text id="wqa1jx"
element-oriented consumption abstraction
```

Puede implementarse mediante:

```text id="ay1atd"
LazyCollection
    ↓
ChunkTraversal
    ↓
yield items one-by-one
```

---

# 10. Lazy Collection vs Streaming Result

Streaming Result normalmente conserva:

```text id="jgpw2l"
statement/result cursor
+
connection
```

durante el consumo.

Lazy Collection puede utilizar:

```text id="zbop4z"
CHUNK_BACKED
CURSOR_BACKED
STREAM_BACKED
CUSTOM
```

---

# 11. Lazy Collection vs PHP Generator

PHP Generator puede ser un mecanismo de implementación:

```php id="j90we5"
yield $item;
```

pero:

```text id="jz2dqq"
LazyCollection
≠
Generator object
```

porque VoltStack deberá añadir contratos sobre:

```text id="p7r3wt"
replayability
resource lifecycle
query semantics
transformations
diagnostics
cancellation
```

---

# 12. Objetivos

El sistema deberá soportar:

1. deferred execution;
2. bounded memory;
3. iterable API;
4. generic typing;
5. map;
6. filter;
7. flatMap;
8. each;
9. reduce;
10. take;
11. skip;
12. takeWhile;
13. dropWhile;
14. tap;
15. batch;
16. materialization explícita;
17. chunk-backed iteration;
18. keyset-backed iteration;
19. streaming-backed iteration;
20. single-pass semantics;
21. replayability explícita;
22. cancellation;
23. resource cleanup;
24. ORM;
25. projections;
26. transaction awareness;
27. tenant/shard context;
28. persistent-runtime safety;
29. telemetry;
30. extensibilidad.

---

# 13. Arquitectura general

```text id="i5tkpg"
Query / Source
      │
      ▼
LazySourceDefinition
      │
      ▼
LazyCollection<T>
      │
      ├── map()
      ├── filter()
      ├── take()
      ├── skip()
      ├── flatMap()
      └── tap()
      │
      ▼
LazyPipeline
      │
      ▼
Iteration Planner
      │
      ├── Chunk-backed
      ├── Cursor-backed
      ├── Stream-backed
      └── Custom
      │
      ▼
LazyIterator
      │
      ▼
Consumer
```

---

# 14. API base

```php id="60w1yr"
/**
 * @template T
 */
interface LazyCollection extends IteratorAggregate
{
    public function map(callable $callback): LazyCollection;

    public function filter(callable $callback): LazyCollection;

    public function take(int $count): LazyCollection;

    public function each(callable $callback): void;

    public function collect(): Collection;
}
```

La API final podrá enriquecerse manteniendo semánticas claras.

---

# 15. Generic typing

Idealmente:

```text id="q8slmd"
LazyCollection<User>
```

y después:

```php id="lneoym"
$emails = $users->map(
    fn (User $user): EmailAddress => $user->email()
);
```

resultado:

```text id="bc5oku"
LazyCollection<EmailAddress>
```

para analizadores estáticos/IDE.

---

# 16. LazySource

La colección deberá separar:

```text id="2m39yq"
source definition
```

de:

```text id="ye895m"
active iterator
```

---

# 17. LazySourceDefinition

Conceptualmente:

```php id="btgndo"
final readonly class DatabaseLazySource
{
    public function __construct(
        public QueryModel $query,
        public LazyIterationPolicy $policy,
        public DatabaseContextDescriptor $context,
    ) {}
}
```

---

# 18. Source immutable

La definición deberá ser immutable y potencialmente reutilizable cuando su contract permita replay.

---

# 19. Active iteration state

En cambio:

```text id="y6ypwf"
current chunk
current row
continuation
result cursor
connection reference
cancellation state
```

será mutable y scoped a una iteración concreta.

---

# 20. Regla de estado

```text id="utv2si"
LazyCollection Definition
≠
LazyIteration State
```

---

# 21. Deferred execution

Ejemplo:

```php id="vov6hw"
$lazy = User::query()->lazy();
```

idealmente:

```text id="74sz49"
queries executed = 0
```

hasta consumir.

---

# 22. Terminal operations

Operaciones terminales activarán consumo.

Ejemplos:

```php id="frpymn"
$lazy->each(...);

$lazy->collect();

$lazy->count();

$lazy->reduce(...);

$lazy->first();
```

---

# 23. Intermediate operations

Operaciones como:

```text id="8u1pjv"
map
filter
take
tap
```

deberán ser lazy siempre que sea semánticamente posible.

---

# 24. Materialization boundaries

Operaciones como:

```text id="f1l4k6"
sort()
groupBy()
reverse()
shuffle()
```

pueden requerir materialización completa si no pueden transformarse a Database Query o a un algoritmo bounded.

---

# 25. No falsa pereza

Nunca llamar "lazy" a una operación que:

```text id="3dhzj2"
loads complete source into memory
```

sin hacer explícita esa frontera.

---

# 26. Operation classification

Podrá existir:

```php id="d2k0be"
enum LazyOperationKind
{
    case STREAMABLE;
    case QUERY_PUSHABLE;
    case BOUNDED_STATE;
    case MATERIALIZING;
    case TERMINAL;
}
```

---

# 27. STREAMABLE

Ejemplos:

```text id="o1psm1"
map
filter
tap
take
takeWhile
```

---

# 28. QUERY_PUSHABLE

Algunas operaciones podrán bajar al Query Engine.

Ejemplo:

```php id="m8z5f1"
User::query()
    ->lazy()
    ->take(10);
```

podría convertirse en:

```text id="49m3wp"
Query LIMIT 10
```

si preserva semántica.

---

# 29. Filter pushdown

Un callback arbitrario:

```php id="8uw0gb"
->filter(fn (User $u) => ...)
```

no puede convertirse automáticamente en SQL.

---

# 30. Typed query predicates

Si se utiliza una API expresable como Query Predicate:

```text id="ndjzo9"
filter(QueryPredicate)
```

sí podría pushdown.

---

# 31. Callback ≠ Query Predicate

Siempre:

```text id="3a4dht"
PHP Callback
≠
Database Predicate
```

---

# 32. Materialization transparency

Una operación materializante deberá:

```text id="c4b7vb"
be explicit
or
emit diagnostics/warning according to policy
```

---

# 33. collect()

```php id="g8ndh7"
$collection = $lazy->collect();
```

es una frontera explícita:

```text id="l86s4o"
Lazy
→
Fully Materialized
```

---

# 34. `all()`

Si se proporciona:

```php id="kkxshz"
$lazy->all();
```

deberá ser equivalente semánticamente a materializar todo.

---

# 35. Memory risk

Para millones de registros:

```text id="mu7qz4"
collect()
```

puede exceder memoria.

---

# 36. MaterializationPolicy

VoltStack podrá configurar:

```text id="b7llrj"
ALLOW
WARN_ABOVE_ESTIMATE
REQUIRE_LIMIT
REJECT_UNBOUNDED
CUSTOM
```

---

# 37. Resource governance

No ejecutar una materialización potencialmente gigantesca sin controles cuando la policy lo prohíba.

---

# 38. Iteration strategy

Propuesta:

```php id="dywksi"
enum LazyIterationStrategy
{
    case CHUNKED_KEYSET;
    case CHUNKED_OFFSET;
    case STREAMING;
    case AUTO;
    case CUSTOM;
}
```

---

# 39. AUTO

El planner podrá elegir según:

```text id="etgzsm"
query shape
ordering
platform capabilities
transaction context
memory budget
connection budget
dataset estimates
```

---

# 40. Default recomendado

Para ORM entities y workloads generales:

```text id="yp0cwu"
CHUNKED_KEYSET
```

cuando exista un ordering válido.

---

# 41. CHUNKED_KEYSET

Reutiliza:

```text id="d18tkh"
200 Cursor Pagination
201 Chunk Processing
```

---

# 42. CHUNKED_OFFSET

Fallback posible, pero conserva riesgos de:

```text id="ozoaor"
offset cost
mutation drift
```

---

# 43. STREAMING

Puede ser eficiente cuando:

```text id="pftq7i"
read-only
single-pass
long-lived connection acceptable
```

---

# 44. Stream lifetime

Un streaming iterator puede mantener:

```text id="pqpnlf"
Connection lease
Statement
Result cursor
```

hasta terminar o cerrarse.

---

# 45. Resource ownership

Lazy Collection deberá saber quién posee:

```text id="2s2moq"
iterator resources
```

y liberarlos determinísticamente.

---

# 46. Iterator lifecycle

Estados conceptuales:

```text id="2ja5zr"
CREATED
OPEN
ITERATING
EXHAUSTED
CLOSED
FAILED
CANCELLED
```

---

# 47. Start

La consulta podrá comenzar:

```text id="zyre3v"
getIterator()
```

o al primer:

```text id="p99byw"
move/next
```

según implementación.

El contrato deberá ser estable.

---

# 48. Recommended execution point

Preferiblemente:

```text id="dzjo18"
first actual iteration
```

para mantener deferred execution auténtica.

---

# 49. Early break

Caso:

```php id="cma593"
foreach ($lazy as $user) {
    if ($user->id() === $target) {
        break;
    }
}
```

Los recursos deberán liberarse.

---

# 50. Destruction no es suficiente

No depender exclusivamente de:

```text id="0f1ofz"
PHP object destructor at some later time
```

para devolver una conexión crítica.

---

# 51. Iterator close

Deberá existir un mecanismo determinista:

```text id="ll21e2"
close()
```

interno/público según API.

---

# 52. Finally

La implementación basada en Generators podrá utilizar:

```php id="vk33m3"
try {
    // yield
} finally {
    // release resources
}
```

---

# 53. Consumer abandonment

Si el consumer abandona la iteración, el iterator deberá cerrar:

```text id="f73sga"
ResultCursor
Statement
Connection lease where owned
temporary runtime state
```

---

# 54. Connection ownership

Distinguir:

```text id="a8xm6l"
OWNED
BORROWED
TRANSACTION_PINNED
```

---

# 55. Owned connection

El iterator puede devolverla al pool al cerrar.

---

# 56. Borrowed connection

No deberá cerrarla físicamente si pertenece a un contexto superior.

---

# 57. Transaction-pinned

Debe respetar la transacción activa.

---

# 58. Lazy iteration dentro de transaction

Ejemplo:

```php id="1gz6ps"
DB::transaction(function () {
    foreach (
        User::query()->lazy() as $user
    ) {
        // ...
    }
});
```

Toda lectura podrá quedar ligada al TransactionContext.

---

# 59. Iteration after transaction end

Caso peligroso:

```php id="4swyke"
$lazy = DB::transaction(
    fn () => User::query()->lazy()
);

// transaction already ended

foreach ($lazy as $user) {
}
```

---

# 60. Context capture

La Lazy Collection no deberá conservar una referencia inválida y fingir que la transacción sigue activa.

---

# 61. Policy

Puede:

```text id="qyg7o1"
capture only portable query context
```

y ejecutar posteriormente fuera de la transacción,

o marcar:

```text id="dhm06k"
REQUIRES_LIVE_CONTEXT
```

---

# 62. LazyContextRequirement

Propuesta:

```php id="umb42h"
enum LazyContextRequirement
{
    case PORTABLE;
    case SAME_TRANSACTION;
    case SAME_ENTITY_MANAGER;
    case SAME_DATABASE_CONTEXT;
}
```

---

# 63. Invalid context

Si una colección requiere transacción activa y ésta terminó:

```text id="apsotj"
LazyContextExpiredException
```

---

# 64. Deferred consistency

El momento de creación:

```text id="9i30uo"
T0
```

no necesariamente será el momento de observación:

```text id="y894ra"
T1
```

---

# 65. Important rule

```text id="kvotnq"
LazyCollectionCreatedAt(T0)
```

no implica:

```text id="7jmoiz"
DataSnapshotAt(T0)
```

---

# 66. Snapshot semantics

Si se requiere snapshot, deberá proveerse mediante:

```text id="xw6ylc"
transaction
temporal query
versioned dataset
snapshot-capable source
```

No por el hecho de ser lazy.

---

# 67. Lazy collection ≠ snapshot

Siempre:

```text id="2u7y0x"
Deferred
≠
Frozen
```

---

# 68. Iteration consistency

Una iteración chunk-backed puede observar DB en múltiples instantes:

```text id="0fzs27"
Chunk 1 at T1
Chunk 2 at T2
Chunk 3 at T3
```

---

# 69. Mutation drift

Aplican las reglas de `201_DATABASE_CHUNK_PROCESSING_SYSTEM.md`.

---

# 70. Read-only default recommendation

Las Lazy Collections DB deberían favorecer uso de lectura.

Mutaciones sobre los objetos consumidos serán permitidas, pero deberán respetar los peligros de traversal/order/IdentityMap.

---

# 71. Reiteración

Una pregunta crucial será:

```text id="dsxw0c"
¿Se puede iterar una LazyCollection dos veces?
```

No hay respuesta universal.

---

# 72. Replayability

Propuesta:

```php id="j6shji"
enum LazyReplayability
{
    case REPLAYABLE;
    case SINGLE_PASS;
    case CONTEXT_BOUND;
    case UNKNOWN;
}
```

---

# 73. REPLAYABLE

Cada nueva iteración crea una ejecución nueva desde la misma definición.

Ejemplo:

```php id="jwpjrq"
$lazy = User::query()->lazy();

foreach ($lazy as $user) {
}

foreach ($lazy as $user) {
}
```

ejecuta nuevamente la consulta.

---

# 74. Important

La segunda iteración puede devolver resultados diferentes si la DB cambió.

---

# 75. REPLAYABLE ≠ memoized

```text id="jv90r0"
Replayable
≠
Cached Results
```

---

# 76. SINGLE_PASS

Streaming sources o recursos específicos pueden permitir una sola iteración.

---

# 77. Second iteration

Debe producir:

```text id="gr4efk"
LazyCollectionAlreadyConsumedException
```

o contrato equivalente.

Nunca devolver vacío silenciosamente si eso oculta el error.

---

# 78. Memoization

Podrá existir una operación explícita:

```php id="pv3pcw"
$lazy->remember();
```

pero convertiría gradualmente elementos consumidos en memoria/cache.

---

# 79. remember() tradeoff

Puede romper bounded-memory si se recorre todo.

Por tanto:

```text id="ax60ko"
remember()
≠
free replayability
```

---

# 80. Transformations

El pipeline podrá representar nodos como:

```text id="5n52i9"
MapOperation
FilterOperation
TakeOperation
SkipOperation
TapOperation
FlatMapOperation
TakeWhileOperation
DropWhileOperation
```

---

# 81. Immutable pipeline

Cada operación preferiblemente devolverá una nueva Lazy Collection.

```php id="qzkry5"
$a = $lazy->filter($aPredicate);

$b = $lazy->filter($bPredicate);
```

sin mutar `$lazy`.

---

# 82. map()

```php id="47h812"
$names = $users->map(
    fn (User $user) => $user->name()
);
```

streamable en memoria elemento a elemento.

---

# 83. filter()

```php id="ox5ru4"
$premium = $users->filter(
    fn (User $user) => $user->isPremium()
);
```

streamable, pero filtra después de DB si es callback PHP.

---

# 84. Filter and chunk size

Si underlying chunks son de 500 pero solo 1% pasa el filtro:

```text id="qj1hmp"
take(100)
```

puede requerir leer muchos chunks.

---

# 85. Logical take

```text id="apft41"
filter(...)->take(100)
```

significa tomar:

```text id="fr6w3y"
100 items after filter
```

no `100 rows from DB before filter`.

---

# 86. Pushdown caution

No empujar `LIMIT 100` antes de un filtro PHP porque cambiaría semántica.

---

# 87. Operation algebra

El optimizer de Lazy Pipeline podrá reorganizar operaciones solo si son semánticamente equivalentes.

---

# 88. Example unsafe rewrite

Original:

```text id="jn80f0"
filter(P)
→
take(10)
```

No siempre equivale a:

```text id="ws036s"
take(10)
→
filter(P)
```

---

# 89. Query pushdown optimizer

Podrá existir:

```text id="7trd7x"
LazyPipelineOptimizer
```

independiente del Query Optimizer.

---

# 90. No arbitrary callback introspection

VoltStack no deberá intentar transformar closures PHP arbitrarias a SQL mediante magia.

---

# 91. flatMap()

Puede expandir elementos:

```text id="kdt241"
1 input
→
N outputs
```

y seguirá siendo streamable si el callback devuelve iterables bounded/lazy.

---

# 92. FlatMap explosion

Resource Governance podrá limitar:

```text id="0omwur"
maximum expansion
```

si el pipeline se utiliza en contexts controlados.

---

# 93. skip()

En una Lazy Collection ya convertida a pipeline PHP:

```text id="frynte"
skip(1_000_000)
```

puede requerir consumir un millón de elementos.

---

# 94. Pushdown

Si `skip()` está directamente sobre un query source y puede expresarse como OFFSET, el planner podría bajarlo al query.

Pero deberá considerar el coste.

---

# 95. take()

`take(N)` deberá detener la fuente en cuanto haya producido N elementos útiles.

---

# 96. Early source cancellation

Para chunk-backed source:

```text id="7cdq40"
no next chunk
```

después de alcanzar N.

Para streaming:

```text id="eww2fd"
close cursor
```

inmediatamente.

---

# 97. first()

```php id="48dm9u"
$user = $lazy->first();
```

deberá consumir únicamente lo necesario.

---

# 98. contains()

Puede short-circuit.

---

# 99. any()

Puede short-circuit.

---

# 100. all-match

`every()` debe parar al primer false.

---

# 101. count()

Para pipeline arbitrario, puede requerir consumir todo.

---

# 102. Count optimization

Si no existen transforms que cambien cardinalidad y source es DB Query, podría utilizar:

```text id="3cjjud"
Pagination/Query Count Planner
```

o Query Engine count.

---

# 103. No incorrect optimization

Después de:

```text id="vmafxp"
filter(PHP callback)
```

no puede sustituirse por DB `COUNT(*)`.

---

# 104. reduce()

Es terminal y normalmente consume todo, aunque memoria adicional puede permanecer O(1) según accumulator.

---

# 105. groupBy()

Normalmente requiere mantener grupos y puede crecer hasta O(N).

---

# 106. groupByLazy()

Podría ser bounded solo si input está ordenado por grouping key.

Una futura specialized operation podría aprovecharlo.

---

# 107. sort()

Sorting arbitrario requiere:

```text id="prebhi"
materialization
```

o external sort infrastructure.

---

# 108. Prefer query ordering

Para DB Lazy Collections:

```php id="rf36jt"
User::query()
    ->orderBy('name')
    ->lazy();
```

es preferible a:

```php id="p0ri4z"
User::query()
    ->lazy()
    ->sortBy('name');
```

para datasets grandes.

---

# 109. Query semantics preservation

Lazy operations no deberán modificar accidentalmente:

```text id="4yt0mh"
authorization scopes
tenant predicates
ordering
result shape
bindings
```

del query base.

---

# 110. ORM integration

Una Lazy Collection de entities:

```text id="9qbhpe"
LazyCollection<User>
```

deberá hidratar mediante el sistema 135–141.

---

# 111. IdentityMap pressure

Cada entidad administrada puede quedar retenida.

```text id="xk30li"
iteration 1..1,000,000
```

podría llenar IdentityMap aunque el iterator solo conserve un elemento.

---

# 112. Critical distinction

```text id="eajooq"
IteratorMemoryBounded
≠
ORMManagedMemoryBounded
```

---

# 113. ORM memory policies

Podrá integrarse con:

```php id="gyf063"
enum LazyOrmMemoryPolicy
{
    case CALLER_MANAGED;
    case KEEP_MANAGED;
    case DETACH_AFTER_YIELD;
    case CLEAR_ENTITY_TYPE_PERIODICALLY;
    case CUSTOM;
}
```

---

# 114. Default seguro

`CALLER_MANAGED` evita que la Lazy Collection altere silenciosamente el EntityManager.

---

# 115. DETACH_AFTER_YIELD problem

No se puede detach inmediatamente antes de que el consumidor termine con la entidad.

---

# 116. After-advance policy

Podría despegar la entidad anterior al avanzar al siguiente elemento, pero esto produce semántica sorprendente.

Por eso no debería ser default.

---

# 117. Explicit detached lazy mode

Una API futura podría ofrecer:

```php id="vrdjhe"
$query->lazyDetached();
```

con contrato explícito.

---

# 118. Projection preferred

Para grandes procesos:

```php id="2xr0hc"
$query
    ->select('id', 'email')
    ->lazy();
```

puede evitar IdentityMap por completo.

---

# 119. Scalar lazy collection

```text id="vjz5u4"
LazyCollection<string>
```

puede ser mucho más ligera.

---

# 120. Tuple/DTO

También:

```text id="qdueeq"
LazyCollection<Tuple>
LazyCollection<DTO>
LazyCollection<Projection>
```

---

# 121. Relationship lazy collection

No deberá confundirse:

```text id="f6otvz"
User.orders lazy relationship
```

con:

```text id="cjdk9h"
Database LazyCollection
```

---

# 122. Possible integration

Una relación to-many podría proporcionar una Lazy Collection explícita:

```php id="hgahvn"
$user->orders()->lazy();
```

pero eso sigue usando Relationship Loading metadata.

---

# 123. Partial relationship coverage

Consumir algunos elementos:

```php id="dwf8ng"
take(10)
```

no marca relación completa.

---

# 124. Full exhaustion

Incluso al agotar una Lazy Relationship Collection, la relación solo puede marcarse COMPLETE si la loading strategy tiene suficiente evidencia.

---

# 125. Serialization

Serializar una Lazy Collection no deberá consumirla automáticamente.

---

# 126. JSON

Nunca:

```text id="iey8fr"
json_encode($lazy)
→ automatically query millions of rows
```

por default.

---

# 127. Serialization policy

Puede:

```text id="cael9y"
REJECT
REQUIRE_MATERIALIZATION
LIMITED_PREVIEW
CUSTOM
```

---

# 128. Debugger

Inspeccionar una Lazy Collection no debería ejecutar toda la query.

---

# 129. Debug preview

Developer tools podrán mostrar:

```text id="jtvzfq"
source query
pipeline
strategy
state
```

sin consumo.

---

# 130. Optional preview

Si el usuario solicita preview, podrá ejecutar:

```text id="82xy9e"
take(5)
```

en un scope separado claramente indicado.

---

# 131. `__toString()` prohibition

Nunca utilizar `__toString()` para materializar datos.

---

# 132. Cache interaction

Lazy Collection definition no es Result Cache.

---

# 133. Result Cache

Cada chunk/fetch podría técnicamente utilizar Result Cache, pero para traversal largo el default debería ser conservador.

---

# 134. Recommended

```text id="aifxwf"
ResultCache = BYPASS
```

para large lazy iteration salvo policy explícita.

---

# 135. Why

Reduce riesgo de:

```text id="gy1c6q"
mixed cache generations
cache pollution
huge page-key explosion
```

---

# 136. Metadata cache

Puede utilizarse normalmente.

---

# 137. Entity cache

Depende de consistency policy y ORM mode.

No deberá asumirse automáticamente útil para un scan masivo.

---

# 138. Read/write routing

Lazy iteration puede durar mucho.

Routing debe permanecer compatible con consistency policy.

---

# 139. Endpoint pinning

Streaming requiere normalmente una única conexión/endpoint.

Chunk-backed puede cambiar endpoints si la policy lo permite.

---

# 140. Endpoint switching

Puede producir:

```text id="41lp8f"
non-monotonic observations
```

si replicas tienen distintos lags.

---

# 141. LazyConsistencyPolicy

Propuesta:

```php id="gfq8b4"
enum LazyConsistencyPolicy
{
    case BEST_EFFORT;
    case MONOTONIC;
    case READ_YOUR_WRITES;
    case PIN_ENDPOINT;
    case SNAPSHOT;
    case CUSTOM;
}
```

---

# 142. PIN_ENDPOINT ≠ snapshot

Siempre:

```text id="kx0g4d"
Same Endpoint
≠
Same Snapshot
```

---

# 143. Snapshot

Para una Lazy Collection que dure mucho, snapshot puede tener altos costos.

Debe ser explicit opt-in.

---

# 144. Writes during iteration

Si el consumidor modifica las mismas entidades/rows, aplican los riesgos de Chunk Processing.

---

# 145. Read-your-writes

Si el traversal depende de cambios realizados en chunks anteriores, el routing debe respetarlos.

---

# 146. Sharding

Lazy Collection podrá representar:

```text id="v9iktr"
single-shard source
multi-shard globally ordered source
per-shard source
```

---

# 147. Single shard

Directo:

```text id="13xucl"
LazySource
→ Shard A
```

---

# 148. Multi-shard global ordering

Puede reutilizar:

```text id="aafxb1"
Distributed Cursor Pagination
+
Chunk Merge
```

---

# 149. Per-shard lazy iteration

Para processing:

```text id="ld39up"
LazyCollection<ShardItem>
```

podrá procesar shards independientes cuando el ordering global no sea necesario.

---

# 150. Lazy merge

Un k-way merge puede mantener:

```text id="y4p8un"
O(number_of_shards)
```

estado de frontiers en vez de materializar todo.

---

# 151. Distributed failure

Si un shard requerido falla a mitad de iteración:

```text id="wclvxh"
iterator
→ FAILED / PARTIAL / UNKNOWN
```

según contrato.

---

# 152. Silent shard omission

Nunca:

```text id="uhw881"
ignore failed shard
and finish as success
```

sin policy explícita.

---

# 153. Tenant isolation

`TenantContext` deberá quedar ligado a la Lazy Source/active iteration cuando sea relevante.

---

# 154. No tenant switch mid-iteration

Una iteración normal no deberá cambiar:

```text id="yaux85"
Tenant A
→ Tenant B
```

por mutation de contexto global.

---

# 155. Cross-tenant source

Solo mediante API administrativa explícita y autorizada.

---

# 156. Security

El query base ya deberá incluir scopes/autorización necesarios.

---

# 157. Lazy pipeline filter ≠ security filter

Nunca utilizar:

```php id="y5mv20"
->lazy()
->filter(fn ($row) => $authorized($row))
```

como sustituto normal de filtrado DB de seguridad.

---

# 158. Razón

Podría:

```text id="1u4oi8"
load unauthorized rows
leak timing
leak counts
waste resources
```

---

# 159. Security before source

Preferir:

```text id="c2t6eh"
Authorization Scope
→ Query
→ Lazy Source
```

---

# 160. Cancellation

Active iterator deberá aceptar `CancellationToken`.

---

# 161. Check points

Puede comprobar cancellation:

```text id="47zryk"
before opening source
before fetch chunk
between chunks
between rows
before expensive transform
```

---

# 162. Cancellation and resources

Al cancelar:

```text id="zoo4an"
close active resources
mark iterator CANCELLED
```

---

# 163. Cancellation ≠ rollback

Si el consumidor realizó writes previamente:

```text id="q2k6hm"
cancel
≠
undo previous work
```

---

# 164. Exceptions inside pipeline

Ejemplo:

```php id="ledmkj"
$lazy->map(function ($item) {
    throw new DomainException();
});
```

Debe:

```text id="7yg9mr"
close resources
preserve cause
mark iteration failed
rethrow/wrap according to API
```

---

# 165. Error provenance

Debe distinguir:

```text id="5xpooe"
SOURCE_FAILURE
FETCH_FAILURE
HYDRATION_FAILURE
PIPELINE_OPERATION_FAILURE
CONSUMER_FAILURE
RESOURCE_CLEANUP_FAILURE
CANCELLATION
```

---

# 166. Consumer exception

Si el error ocurre en:

```php id="1hz085"
foreach (...) {
    throw ...
}
```

la Lazy Collection debe cerrar recursos durante unwind, pero no debería apropiarse semánticamente del error del consumidor.

---

# 167. Resource cleanup failure

Si además del error original falla cleanup, el error original debe conservarse como causa primaria, con cleanup failure adjunta/diagnosticada.

---

# 168. Retry

Retry de fetch puede seguir `86_DATABASE_EXECUTION_RETRY_SYSTEM`.

Pero reintentar un pipeline arbitrario ya parcialmente consumido es más complejo.

---

# 169. Iterator replay

No deberá reejecutarse automáticamente desde el inicio después de haber entregado elementos al consumidor.

Eso podría duplicar procesamiento.

---

# 170. Fetch retry safe boundary

Chunk-backed iteration puede reintentar un fetch antes de entregar sus resultados si la semántica lo permite.

---

# 171. After yield

Una vez que un elemento fue entregado:

```text id="pcck8z"
replay
```

puede ser observable.

---

# 172. Checkpoint integration

Lazy Collection en sí no será un recovery workflow.

Para procesamiento reanudable deberá usarse:

```text id="lq9us7"
Chunk Processing + Checkpoints
```

o `208 Large Dataset Processing`.

---

# 173. Lazy Collection ≠ resumable job

Regla:

```text id="1edtyu"
Lazy
≠
Durably Resumable
```

---

# 174. Parallel iteration

La misma replayable LazyCollection podrá crear dos iterators:

```php id="ic1gwe"
$a = $lazy->getIterator();
$b = $lazy->getIterator();
```

si policy/context lo permiten.

---

# 175. Separate state

Cada iterator deberá tener:

```text id="2vp4om"
own continuation
own source resource
own chunk buffer
own cancellation state
own telemetry span
```

---

# 176. Shared transaction

Si ambos usan la misma transaction/context, puede haber restricciones de driver sobre concurrent statements.

Platform capabilities deberán gobernarlo.

---

# 177. Concurrent streaming

Muchos drivers/connections no permiten múltiples active streaming results sobre la misma conexión.

No deberá fingirse soporte.

---

# 178. LazyConcurrencyPolicy

Podrá definir:

```text id="hym8xm"
ALLOW_INDEPENDENT
REJECT_SHARED_CONTEXT
SERIALIZE
CUSTOM
```

---

# 179. Persistent runtimes

En FrankenPHP, una LazyCollection activa no deberá sobrevivir accidentalmente al request boundary salvo que exista un owner explícito de background operation.

---

# 180. Request end

El lifecycle reset deberá cerrar:

```text id="flsb9o"
open iterators
result cursors
borrowed resource leases owned by request
```

según ownership.

---

# 181. Escaped Lazy Collection

Caso:

```text id="ztdbmj"
Request A creates LazyCollection
stores it globally
Request B consumes it
```

debe considerarse inválido si transporta context request-scoped.

---

# 182. Portable definition

Solo una definición explícitamente:

```text id="ja9e03"
PORTABLE
```

puede trasladarse entre scopes, resolviendo un nuevo runtime context al iterar.

---

# 183. No mutable static state

Nunca:

```php id="nud6ay"
LazyCollection::$currentIterator
```

---

# 184. RoadRunner / OpenSwoole

Misma regla:

```text id="oy1iph"
WorkerReuse
≠
IteratorReuse
```

---

# 185. Fibers/coroutines

Cada active iterator será aislado.

---

# 186. Backpressure

Lazy consumption provee una forma natural de backpressure:

```text id="h753ol"
Consumer asks next
→ Source produces next
```

---

# 187. Chunk prefetch

Una optimización podrá prefetch el siguiente chunk.

---

# 188. Prefetch tradeoff

Puede mejorar throughput, pero aumenta:

```text id="v8aswn"
memory
connection use
work performed ahead of demand
cancellation waste
```

---

# 189. PrefetchPolicy

```php id="93jcpm"
enum LazyPrefetchPolicy
{
    case NONE;
    case NEXT_CHUNK;
    case ADAPTIVE;
    case CUSTOM;
}
```

---

# 190. Default

```text id="1fvpk3"
NONE
```

o un prefetch conservador según runtime.

---

# 191. Prefetch ≠ eager materialization

Prefetch sigue siendo bounded, pero deja de ser estrictamente demand-one-chunk-at-a-time.

---

# 192. Resource budget

Propuesta:

```php id="rznbbf"
final readonly class LazyResourceBudget
{
    public function __construct(
        public ?int $maxItems,
        public ?int $maxDurationMs,
        public ?int $maxMemoryBytes,
        public ?int $maxOpenResources,
        public ?int $maxChunkSize,
        public ?int $maxPipelineDepth,
    ) {}
}
```

---

# 193. Pipeline depth

Un pipeline creado dinámicamente podría contener miles de operations.

El sistema podrá limitarlo para evitar abuso.

---

# 194. Max items

Puede actuar como hard safety limit independiente de `take()`.

---

# 195. Max duration

Una iteración larga podrá ser cancelada según policy.

---

# 196. Resource estimation

No siempre será posible conocer memoria exacta.

Deberán distinguirse:

```text id="44zpex"
KNOWN
ESTIMATED
UNKNOWN
```

---

# 197. Diagnostics

API conceptual:

```php id="42v2mq"
DB::lazy()->explain(
    User::query()
        ->where('active', true)
        ->orderBy('id')
        ->lazy()
        ->filter(fn (User $user) => $user->isPremium())
        ->take(1000),
);
```

---

# 198. Explain output

```text id="3jak41"
LAZY COLLECTION PLAN

Source:
    EntityQuery<User>

Execution:
    DEFERRED

Iteration Strategy:
    CHUNKED_KEYSET

Chunk Size:
    500

Ordering:
    id ASC

Replayability:
    REPLAYABLE

Pipeline:
    1. PHP FILTER
    2. TAKE 1000

Pushdown:
    Query predicate: already pushed
    FILTER: not pushable
    TAKE: cannot push before PHP filter

ORM Mode:
    MANAGED ENTITIES

ORM Memory Policy:
    CALLER_MANAGED

Result Cache:
    BYPASS

Consistency:
    BEST_EFFORT

Materialization:
    none

Warnings:
    IdentityMap may grow during long iteration.
```

---

# 199. Explain materialization

Ejemplo:

```text id="gfl1xy"
Pipeline:
    map()
    groupBy()

Materialization Boundary:
    groupBy()

Estimated Rows:
    3,500,000

Policy:
    WARN

Recommendation:
    Perform grouping in the Database Query when possible.
```

---

# 200. Telemetry

Eventos conceptuales:

```text id="7w7qom"
LazyCollectionCreated
LazyIterationStarted
LazySourceOpened
LazyChunkFetched
LazyPipelineOperationFailed
LazyIterationShortCircuited
LazyIterationMaterialized
LazyIterationCompleted
LazyIterationCancelled
LazyIterationFailed
LazySourceClosed
```

---

# 201. Métricas

```text id="2lg739"
db.lazy.iterations
db.lazy.duration
db.lazy.items_yielded
db.lazy.chunks_fetched
db.lazy.materializations
db.lazy.short_circuit
db.lazy.failures
db.lazy.cancelled
db.lazy.open_resources
```

---

# 202. Bounded cardinality

No utilizar como metric labels:

```text id="ij8yd0"
entity IDs
query parameters
raw SQL
tenant IDs
cursor boundaries
callback names with arbitrary values
```

---

# 203. Error hierarchy

```text id="js3e2s"
DatabaseException
└── LazyCollectionException
    ├── LazySourceException
    ├── LazyIterationException
    ├── LazyCollectionAlreadyConsumedException
    ├── LazyContextExpiredException
    ├── LazyReplayabilityException
    ├── LazyMaterializationException
    ├── LazyResourceLimitException
    ├── LazyPipelineException
    │   └── LazyPipelineOperationException
    ├── LazyConcurrencyException
    ├── LazyResourceCleanupException
    └── LazyConsistencyException
```

---

# 204. Directory structure

```text id="l56y4k"
src/Quantum/Database/Lazy/
│
├── LazyCollection.php
├── DatabaseLazyCollection.php
├── LazyCollectionFactory.php
├── LazyReplayability.php
├── LazyContextRequirement.php
│
├── Source/
│   ├── LazySource.php
│   ├── LazySourceDefinition.php
│   ├── DatabaseLazySource.php
│   ├── ChunkedLazySource.php
│   ├── StreamingLazySource.php
│   └── LazySourceState.php
│
├── Pipeline/
│   ├── LazyPipeline.php
│   ├── LazyPipelineOperation.php
│   ├── LazyOperationKind.php
│   ├── LazyPipelineOptimizer.php
│   └── Operation/
│       ├── MapOperation.php
│       ├── FilterOperation.php
│       ├── FlatMapOperation.php
│       ├── TakeOperation.php
│       ├── SkipOperation.php
│       ├── TapOperation.php
│       ├── TakeWhileOperation.php
│       └── DropWhileOperation.php
│
├── Iteration/
│   ├── LazyIterator.php
│   ├── LazyIterationContext.php
│   ├── LazyIterationState.php
│   ├── LazyIterationStrategy.php
│   ├── LazyIterationPlanner.php
│   └── LazyIterationPlan.php
│
├── Materialization/
│   ├── LazyMaterializer.php
│   ├── MaterializationPolicy.php
│   └── MaterializationBoundary.php
│
├── ORM/
│   ├── LazyOrmMemoryPolicy.php
│   └── LazyEntityIterationSupport.php
│
├── Consistency/
│   └── LazyConsistencyPolicy.php
│
├── Concurrency/
│   └── LazyConcurrencyPolicy.php
│
├── Prefetch/
│   ├── LazyPrefetchPolicy.php
│   └── LazyPrefetcher.php
│
├── Resource/
│   ├── LazyResourceBudget.php
│   ├── LazyResourceOwner.php
│   └── LazyResourceTracker.php
│
├── Diagnostics/
│   ├── LazyInspector.php
│   └── LazyExplainer.php
│
├── Telemetry/
│   └── LazyTelemetry.php
│
└── Exception/
    └── ...
```

---

# 205. Testing matrix

| Área | Caso |
|---|---|
| Deferred | no query before iteration |
| Deferred | query on first iteration |
| Replay | replayable source |
| Replay | single-pass rejection |
| Pipeline | map |
| Pipeline | filter |
| Pipeline | flatMap |
| Pipeline | take |
| Pipeline | skip |
| Pipeline | takeWhile |
| Short circuit | first |
| Short circuit | contains |
| Materialization | collect |
| Materialization | groupBy |
| Optimization | take pushdown |
| Optimization | unsafe filter/take reorder |
| Chunk source | keyset |
| Chunk source | offset |
| Stream source | early break cleanup |
| Resource | cursor closed |
| Resource | exception cleanup |
| ORM | IdentityMap growth |
| ORM | projection mode |
| Transaction | active |
| Transaction | expired context |
| Cache | bypass |
| Replica | pin endpoint |
| Shard | single |
| Shard | distributed merge |
| Tenant | no leakage |
| Concurrency | two iterators |
| Runtime | worker reset |
| Cancellation | before fetch |
| Cancellation | mid-stream |
| Serialization | no accidental consume |

---

# 206. Architectural invariants

## DB-LAZY-001
Lazy Collection será distinta de Loaded Collection.

## DB-LAZY-002
Lazy Collection será distinta de Query Builder.

## DB-LAZY-003
Lazy Collection será distinta de Chunk.

## DB-LAZY-004
Lazy Collection será distinta de Result Cursor.

## DB-LAZY-005
Lazy Collection será distinta de Streaming Result.

## DB-LAZY-006
Lazy Collection será distinta de PHP Generator.

## DB-LAZY-007
Lazy Collection podrá utilizar PHP Generator internamente.

## DB-LAZY-008
Lazy Collection representará deferred consumption.

## DB-LAZY-009
Crear una Lazy Collection no ejecutará query por default.

## DB-LAZY-010
Consumir una Lazy Collection podrá producir I/O.

## DB-LAZY-011
I/O diferido no deberá ocultarse conceptualmente.

## DB-LAZY-012
Source Definition será distinta de active iteration state.

## DB-LAZY-013
Source Definition será preferiblemente immutable.

## DB-LAZY-014
Active iterator state será scoped.

## DB-LAZY-015
lazy() capturará query semantics estables.

## DB-LAZY-016
Mutaciones posteriores del Query Builder no cambiarán silenciosamente la Lazy Source capturada.

## DB-LAZY-017
Intermediate operations permanecerán lazy cuando sea posible.

## DB-LAZY-018
Terminal operations podrán activar consumo.

## DB-LAZY-019
Materializing operations serán explícitas/diagnosticables.

## DB-LAZY-020
collect() será una frontera de materialización.

## DB-LAZY-021
all() no será memory-safe por definición.

## DB-LAZY-022
Materialization podrá estar gobernada por policy.

## DB-LAZY-023
Lazy operations tendrán clasificación semántica.

## DB-LAZY-024
PHP callbacks no se convertirán mágicamente a SQL.

## DB-LAZY-025
Callback será distinto de Query Predicate.

## DB-LAZY-026
Query pushdown solo ocurrirá cuando preserve semántica.

## DB-LAZY-027
filter→take no será reordenado incorrectamente.

## DB-LAZY-028
take podrá short-circuit source.

## DB-LAZY-029
first podrá short-circuit source.

## DB-LAZY-030
contains podrá short-circuit source.

## DB-LAZY-031
reduce podrá consumir todo sin materializar necesariamente todos los items.

## DB-LAZY-032
groupBy podrá requerir memoria O(N).

## DB-LAZY-033
sort podrá requerir materialización.

## DB-LAZY-034
Database ordering será preferible para grandes datasets.

## DB-LAZY-035
Iteration strategy será explícita o planificada.

## DB-LAZY-036
CHUNKED_KEYSET será soportado.

## DB-LAZY-037
CHUNKED_OFFSET será soportado.

## DB-LAZY-038
STREAMING será soportado.

## DB-LAZY-039
AUTO podrá elegir estrategia mediante capabilities/policy.

## DB-LAZY-040
Chunk semantics reutilizarán Chunk Processing.

## DB-LAZY-041
Keyset semantics reutilizarán Cursor Pagination.

## DB-LAZY-042
Streaming mantendrá resource ownership explícito.

## DB-LAZY-043
Iterator tendrá lifecycle explícito.

## DB-LAZY-044
Early break liberará recursos.

## DB-LAZY-045
Iterator failure liberará recursos cuando sea posible.

## DB-LAZY-046
Resource cleanup no dependerá exclusivamente de GC/destructor.

## DB-LAZY-047
Connection ownership será explícito.

## DB-LAZY-048
Borrowed connection no será cerrada arbitrariamente.

## DB-LAZY-049
Transaction-pinned connection será respetada.

## DB-LAZY-050
Lazy Collection no extenderá vida de transaction expirada silenciosamente.

## DB-LAZY-051
Context requirements serán representables.

## DB-LAZY-052
Expired required context producirá error.

## DB-LAZY-053
Deferred creation no significará snapshot creation.

## DB-LAZY-054
Lazy Collection será distinta de snapshot.

## DB-LAZY-055
Chunk-backed iteration podrá observar múltiples instantes DB.

## DB-LAZY-056
Snapshot será opt-in.

## DB-LAZY-057
Replayability será explícita.

## DB-LAZY-058
Replayable no significará memoized.

## DB-LAZY-059
Replayable second iteration podrá ejecutar nueva query.

## DB-LAZY-060
Second iteration podrá producir resultados distintos si DB cambió.

## DB-LAZY-061
Single-pass source no devolverá vacío silenciosamente al reusar.

## DB-LAZY-062
Memoization será explícita.

## DB-LAZY-063
Memoization podrá romper bounded-memory.

## DB-LAZY-064
Pipeline será preferiblemente immutable.

## DB-LAZY-065
map preservará lazy evaluation.

## DB-LAZY-066
filter preservará lazy evaluation.

## DB-LAZY-067
flatMap podrá expandir cardinalidad.

## DB-LAZY-068
flatMap expansion podrá estar gobernada.

## DB-LAZY-069
skip grande podrá ser costoso.

## DB-LAZY-070
skip pushdown deberá respetar semántica/coste.

## DB-LAZY-071
LogicalProjection podrá mantenerse separada de execution internals.

## DB-LAZY-072
Lazy Collection no será Hydrator.

## DB-LAZY-073
ORM entities serán hidratadas por Hydration System.

## DB-LAZY-074
Iterator bounded-memory no garantizará ORM bounded-memory.

## DB-LAZY-075
IdentityMap growth será explícitamente considerado.

## DB-LAZY-076
Lazy Collection no hará EntityManager::clear() silenciosamente.

## DB-LAZY-077
ORM memory management será policy-driven.

## DB-LAZY-078
Projection lazy iteration será soportada.

## DB-LAZY-079
Scalar lazy iteration será soportada.

## DB-LAZY-080
Tuple lazy iteration será soportada.

## DB-LAZY-081
DTO lazy iteration será soportada.

## DB-LAZY-082
Lazy Relationship Collection será distinta del core Lazy Collection, aunque pueda integrarse.

## DB-LAZY-083
Partial relationship consumption no significará COMPLETE.

## DB-LAZY-084
Serialization no consumirá automáticamente la colección.

## DB-LAZY-085
Debug inspection no consumirá toda la colección.

## DB-LAZY-086
`__toString()` no materializará la fuente.

## DB-LAZY-087
Result Cache será bypass recomendado para scans largos.

## DB-LAZY-088
Metadata Cache podrá utilizarse normalmente.

## DB-LAZY-089
Entity Cache usage será policy-driven.

## DB-LAZY-090
Cache no será fuente de Lazy state.

## DB-LAZY-091
Read routing respetará consistency policy.

## DB-LAZY-092
Streaming normalmente mantendrá endpoint pinning.

## DB-LAZY-093
Endpoint pinning no significará snapshot.

## DB-LAZY-094
Chunk-backed iteration podrá cambiar endpoint solo si policy lo permite.

## DB-LAZY-095
MONOTONIC consistency será soportable.

## DB-LAZY-096
READ_YOUR_WRITES será soportable.

## DB-LAZY-097
SNAPSHOT será explícito.

## DB-LAZY-098
Writes durante iteración estarán sujetos a Chunk mutation semantics.

## DB-LAZY-099
Single-shard sources serán soportadas.

## DB-LAZY-100
Distributed ordered source requerirá merge semantics.

## DB-LAZY-101
Per-shard lazy iteration será soportable.

## DB-LAZY-102
Distributed failure no será ocultado.

## DB-LAZY-103
TenantContext será preservado.

## DB-LAZY-104
Tenant no cambiará silenciosamente mid-iteration.

## DB-LAZY-105
Cross-tenant lazy source será explícita y autorizada.

## DB-LAZY-106
Lazy PHP filter no sustituirá authorization scope.

## DB-LAZY-107
Security filtering ocurrirá antes del source cuando sea posible.

## DB-LAZY-108
Cancellation será soportada.

## DB-LAZY-109
Cancellation liberará recursos.

## DB-LAZY-110
Cancellation no revertirá writes previos automáticamente.

## DB-LAZY-111
Pipeline failures conservarán original cause.

## DB-LAZY-112
Consumer failures no serán confundidos con source failures.

## DB-LAZY-113
Cleanup failure no ocultará causa primaria.

## DB-LAZY-114
Fetch retries respetarán Execution Retry System.

## DB-LAZY-115
La iteración no se reiniciará automáticamente después de entregar elementos.

## DB-LAZY-116
Lazy Collection no será resumable job.

## DB-LAZY-117
Durable resume pertenecerá a Chunk/Large Dataset Processing.

## DB-LAZY-118
Múltiples iterators tendrán estados separados.

## DB-LAZY-119
Concurrent iteration dependerá de capabilities/context.

## DB-LAZY-120
Concurrent streaming sobre la misma connection no será asumido.

## DB-LAZY-121
Persistent runtime state será scoped.

## DB-LAZY-122
Open iterators no sobrevivirán request boundary sin ownership explícito.

## DB-LAZY-123
FrankenPHP worker reuse no reutilizará iterator state.

## DB-LAZY-124
RoadRunner worker reuse no reutilizará iterator state.

## DB-LAZY-125
OpenSwoole worker reuse no reutilizará iterator state.

## DB-LAZY-126
Coroutine iterators estarán aislados.

## DB-LAZY-127
Immutable source definitions podrán compartirse si son context-independent.

## DB-LAZY-128
Mutable continuation no será global.

## DB-LAZY-129
Chunk buffers no serán globales.

## DB-LAZY-130
Cancellation state no será global.

## DB-LAZY-131
Telemetry span state no será global.

## DB-LAZY-132
Lazy consumption proveerá backpressure natural.

## DB-LAZY-133
Prefetch será bounded.

## DB-LAZY-134
Prefetch será policy-driven.

## DB-LAZY-135
Prefetch no será eager full materialization.

## DB-LAZY-136
ResourceBudget será soportado.

## DB-LAZY-137
Pipeline depth podrá limitarse.

## DB-LAZY-138
Max items podrá imponerse.

## DB-LAZY-139
Max duration podrá imponerse.

## DB-LAZY-140
Resource estimates conservarán UNKNOWN cuando corresponda.

## DB-LAZY-141
Lazy plans serán explainable.

## DB-LAZY-142
Pushdown decisions serán explainable.

## DB-LAZY-143
Materialization boundaries serán explainable.

## DB-LAZY-144
ORM memory risks serán diagnosticables.

## DB-LAZY-145
Consistency strategy será explainable.

## DB-LAZY-146
Iteration strategy será explainable.

## DB-LAZY-147
Telemetry tendrá bounded cardinality.

## DB-LAZY-148
Raw SQL no será metric label.

## DB-LAZY-149
Entity IDs no serán metric labels.

## DB-LAZY-150
Sensitive values no serán expuestos por diagnostics.

## DB-LAZY-151
LazyCollection no creará un segundo Query Engine.

## DB-LAZY-152
LazyCollection no creará un segundo ORM.

## DB-LAZY-153
LazyCollection no creará un segundo Transaction Manager.

## DB-LAZY-154
LazyCollection no creará un segundo Cursor system.

## DB-LAZY-155
LazyCollection no creará un segundo Chunk system.

## DB-LAZY-156
LazyCollection no creará un segundo Cache system.

## DB-LAZY-157
LazyCollection no creará un segundo Distribution system.

## DB-LAZY-158
Compiler seguirá siendo autoridad SQL.

## DB-LAZY-159
Executor seguirá siendo autoridad de ejecución.

## DB-LAZY-160
Hydrator seguirá siendo autoridad de hydration.

## DB-LAZY-161
ConnectionManager seguirá siendo autoridad de connections.

## DB-LAZY-162
Version no será Capability.

## DB-LAZY-163
MySQL será soportado.

## DB-LAZY-164
MariaDB será soportado independientemente.

## DB-LAZY-165
PostgreSQL será soportado.

## DB-LAZY-166
SQLite será soportado según capabilities.

## DB-LAZY-167
Correctness tendrá prioridad sobre pushdown agresivo.

## DB-LAZY-168
Laziness tendrá prioridad sobre conveniencias materializantes implícitas.

## DB-LAZY-169
Bounded-memory será una propiedad del pipeline/source, no una promesa sobre código del consumidor.

## DB-LAZY-170
Lazy Collection será una abstracción de consumo, no database truth.

---

# 207. Modelo formal

Sea:

```text id="ajjjqv"
S
=
data source
```

y una secuencia de operaciones:

```text id="42plbf"
P
=
[o1, o2, ..., on]
```

Entonces:

```text id="sl44qh"
LazyCollection
=
(S, P)
```

sin requerir ejecución inmediata.

---

# 208. Evaluación

Para un consumidor que solicita elementos:

```text id="fdqlh9"
Evaluate(S, P)
```

produce una secuencia incremental:

```text id="lzoxgb"
x1, x2, ..., xn
```

según demanda.

---

# 209. Pipeline composition

Si:

```text id="7m5az6"
L = Lazy(S, P)
```

y se aplica operación `o`:

```text id="joo4lx"
L'
=
Lazy(S, P ⧺ [o])
```

preferiblemente sin mutar `L`.

---

# 210. Laziness property

Para una operación intermedia streamable:

```text id="fhrvg6"
Create(L')
```

no deberá implicar:

```text id="27j6xb"
Consume(S)
```

---

# 211. Bounded memory

Para chunk-backed iteration con chunk size `C`:

```text id="ac3sh6"
M_source
≈
O(C)
```

siempre que:

```text id="ywfawh"
pipeline operations bounded
consumer does not retain items
ORM does not retain managed objects indefinitely
```

---

# 212. Critical qualification

Por tanto:

```text id="8d7f71"
ChunkBackedLazy
```

no implica automáticamente:

```text id="9m7b4g"
TotalProcessMemory = O(C)
```

porque pueden existir:

```text id="s1eh3g"
IdentityMap
callback retention
memoization
grouping
user caches
```

---

# 213. Replay model

Para una source replayable:

```text id="0bdnpm"
Iterate(L, T1)
```

y:

```text id="s7m6q7"
Iterate(L, T2)
```

son dos ejecuciones independientes.

Por tanto:

```text id="gccu7c"
Result(T1)
```

puede diferir de:

```text id="dupy24"
Result(T2)
```

si la DB cambia.

---

# 214. Short-circuit property

Para:

```text id="s4sgzv"
take(N)
```

el source deberá detener consumo una vez producidos `N` resultados posteriores a las operaciones previas.

---

# 215. Materialization model

```text id="lg4vqu"
collect(L)
=
Materialize(Evaluate(L))
```

y el coste de memoria puede aproximarse a:

```text id="velxjy"
O(number of produced items)
```

---

# 216. Arquitectura final

```text id="6zfbn5"
                       Query API
                          │
                          ▼
                  LazySourceDefinition
                          │
                          ▼
                    LazyCollection
                          │
                ┌─────────┴──────────┐
                ▼                    ▼
          Lazy Pipeline         Source Policy
                │                    │
         map/filter/take        strategy/context
                │                    │
                └──────────┬─────────┘
                           ▼
                LazyIterationPlanner
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   Chunked Keyset    Chunked Offset      Streaming
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                     LazyIterator
                           │
                           ▼
              Query/Execution/Hydration
                           │
                           ▼
                     Pipeline Eval
                           │
                           ▼
                        Consumer
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
            Continue           Short Circuit
                  │                 │
                  ▼                 ▼
             Next Item        Close Resources
```

---

# 217. Regla maestra final

> **La Lazy Collection de VoltStack será una abstracción de consumo diferido y incremental, no una colección precargada disfrazada ni un recurso de base de datos sin ciclo de vida explícito.**

Siempre:

```text id="48h9hr"
Lazy
≠
Loaded
```

```text id="5z67fi"
Lazy
≠
Snapshot
```

```text id="3fnkqo"
Lazy
≠
Streaming
```

```text id="fvxa4k"
Lazy
≠
Chunk
```

```text id="yz9ol2"
Replayable
≠
Memoized
```

```text id="f52glq"
Iterator Bounded Memory
≠
ORM Bounded Memory
```

```text id="qxp27n"
Deferred
≠
No I/O
```

y:

```text id="2il2z5"
Lazy Collection
≠
Durably Resumable Process
```

---

# 218. Resultado arquitectónico

VoltStack podrá ofrecer:

```php id="97qrya"
User::query()
    ->where('active', true)
    ->orderBy('id')
    ->lazy()
    ->filter(
        fn (User $user) => $user->isEligible()
    )
    ->map(
        fn (User $user) => $user->email()
    )
    ->take(1000)
    ->each(
        fn (EmailAddress $email) => $service->process($email)
    );
```

sin materializar necesariamente toda la tabla y manteniendo separadas las responsabilidades:

```text id="u2rvwc"
Query Engine
    defines source semantics

Chunk/Cursor/Streaming
    define physical traversal

Hydration
    creates result values/entities

Lazy Pipeline
    composes deferred operations

Resource Manager
    governs iterator lifecycle

Consumer
    performs application work
```

Además queda preparado para:

```text id="ee757o"
CLI jobs
background workers
ETL
large exports
maintenance tools
data migrations
analytics processing
administrative operations
```

sin convertir Lazy Collection en un sistema de procesamiento durable.

---

# 219. Bloque 19 hasta ahora

```text id="lfut1k"
✓ 199_DATABASE_PAGINATION_SYSTEM.md
✓ 200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
✓ 201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
✓ 202_DATABASE_LAZY_COLLECTION_SYSTEM.md

→ 203_DATABASE_BULK_INSERT_SYSTEM.md
→ 204_DATABASE_BULK_UPDATE_SYSTEM.md
→ 205_DATABASE_BULK_DELETE_SYSTEM.md
→ 206_DATABASE_IMPORT_SYSTEM.md
→ 207_DATABASE_EXPORT_SYSTEM.md
→ 208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md
```

La transición conceptual ahora será:

```text id="pjhemt"
Read Large Data
├── Pagination
├── Cursor
├── Chunk
└── Lazy

        ↓

Mutate Large Data
├── Bulk Insert
├── Bulk Update
└── Bulk Delete
```

---

# 220. Siguiente documento

```text id="6vkyo7"
203_DATABASE_BULK_INSERT_SYSTEM.md
```

El siguiente documento definirá la arquitectura de inserciones masivas:

```text id="frak62"
Bulk Insert
├── bulk insert model
├── row batches
├── typed values
├── multi-row inserts
├── prepared batch execution
├── generated IDs
├── returning support
├── conflict behavior
├── duplicate handling
├── upsert boundary
├── transaction policies
├── batch sizing
├── packet/parameter limits
├── adaptive batching
├── streaming input
├── ORM bypass semantics
├── UnitOfWork interaction
├── cache invalidation
├── events
├── sharding
├── tenant routing
├── retries
├── partial failure
├── telemetry
└── resource governance
```

estableciendo especialmente:

```text id="b6g1cd"
Bulk Insert
≠
EntityManager::persist() repeated N times
≠
Seeder
≠
Factory
≠
Import
```

y la regla central:

> **Bulk Insert será una operación explícitamente optimizada para insertar conjuntos grandes de filas mediante planes acotados y tipados, conservando las garantías de tipos, transacciones, routing, seguridad y outcome sin fingir semántica ORM completa cuando ésta haya sido deliberadamente omitida.**