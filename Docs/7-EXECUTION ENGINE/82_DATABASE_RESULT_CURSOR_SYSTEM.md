# 82_DATABASE_RESULT_CURSOR_SYSTEM.md

# VoltStack Quantum Database
## Result Cursor System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 82 — Result Cursor System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Result Cursor System` define la abstracción de VoltStack para consumir progresivamente las filas producidas por una operación de base de datos sin exigir que el resultado completo sea cargado en memoria.

Su función principal es transformar el mecanismo de lectura incremental proporcionado por un driver en una abstracción:

- tipada;
- portable;
- controlada;
- observable;
- cancelable cuando sea posible;
- segura respecto a recursos;
- compatible con runtimes persistentes;
- independiente del ORM.

Flujo conceptual:

```text
Database
   ↓
Driver Result Handle
   ↓
Driver Cursor
   ↓
Result Cursor System
   ↓
ResultCursor
   ↓
ResultRow
   ↓
Consumer
```

El cursor constituye una frontera runtime entre:

```text
driver-specific row fetching
```

y:

```text
VoltStack ResultRow consumption
```

---

# 2. Principio fundamental

```text
ResultCursor
≠
DatabaseResult
≠
DriverCursor
≠
BufferedResult
≠
StreamingResult
≠
Iterator
≠
ORM Collection
```

Aunque algunos de estos conceptos puedan colaborar, representan responsabilidades distintas.

---

# 3. Posición arquitectónica

```text
CompiledDatabaseCommand
        ↓
Prepared Statement
        ↓
Parameter Binding
        ↓
Statement Execution
        ↓
Native Driver Result
        ↓
Result System
        ↓
RowResult
        ↓
ResultCursor
        ↓
ResultRow
        ↓
Hydration / Consumer
```

`ResultCursor` no ejecuta nuevamente la consulta.

Consume un resultado ya creado por el Execution Engine.

---

# 4. Objetivos

El sistema deberá proporcionar:

1. lectura incremental;
2. cursor forward-only como baseline;
3. capabilities explícitas;
4. lifecycle formal;
5. ownership de recursos;
6. dependencia con statement y connection;
7. integración con transactions;
8. conversión tipada de filas;
9. detección de exhaustion;
10. cierre explícito;
11. cleanup automático de respaldo;
12. cancellation;
13. límites de consumo;
14. telemetry;
15. diagnostics;
16. aislamiento entre requests/workers;
17. extensibilidad controlada.

---

# 5. No objetivos

El Cursor System no deberá:

```text
generate SQL
prepare statements
bind parameters
execute new queries
retry queries
hydrate entities
manage UnitOfWork
manage IdentityMap
load ORM relationships
paginate automatically
cache query results
sort rows
deduplicate rows
```

---

# 6. Cursor ≠ Query Execution

Un cursor existe después de la ejecución:

```text
execute()
   ↓
Driver Result
   ↓
ResultCursor
```

No:

```text
ResultCursor
   ↓
execute query
```

---

# 7. Cursor ≠ Streaming Result

Aunque están relacionados:

```text
ResultCursor
=
low-level incremental row traversal abstraction
```

mientras:

```text
StreamingResult
=
higher-level controlled streaming consumption model
```

El documento `83_DATABASE_STREAMING_RESULT_SYSTEM.md` construirá esa capa.

---

# 8. Cursor ≠ Iterator

PHP `Iterator` puede ser una API de conveniencia.

Pero:

```text
Iterator
```

no expresa adecuadamente por sí mismo:

- ownership;
- cancellation;
- driver resources;
- statement dependency;
- transaction dependency;
- failure state;
- cursor capabilities;
- telemetry;
- budgets.

Por tanto:

```text
Iterator API
⊂
ResultCursor API
```

conceptualmente.

---

# 9. Contrato principal

```php
namespace VoltStack\Quantum\Database\Execution\Result\Cursor\Contract;

interface ResultCursor
{
    public function id(): CursorId;

    public function state(): CursorState;

    public function capabilities(): CursorCapabilitySet;

    public function metadata(): ResultMetadata;

    public function fetch(): CursorFetchResult;

    public function close(): void;
}
```

---

# 10. fetch()

La operación fundamental será:

```text
fetch()
```

y no:

```text
next(): ?ResultRow
```

como único contrato interno.

La razón es distinguir explícitamente:

```text
ROW
EXHAUSTED
```

sin utilizar `null` ambiguamente.

---

# 11. CursorFetchResult

```php
interface CursorFetchResult
{
    public function status(): CursorFetchStatus;
}
```

Implementaciones:

```text
CursorRowFetched
CursorExhausted
```

---

# 12. CursorRowFetched

```php
final readonly class CursorRowFetched implements CursorFetchResult
{
    public function __construct(
        public ResultRow $row,
        public CursorPosition $position,
    ) {}

    public function status(): CursorFetchStatus
    {
        return CursorFetchStatus::ROW;
    }
}
```

---

# 13. CursorExhausted

```php
final readonly class CursorExhausted implements CursorFetchResult
{
    public function status(): CursorFetchStatus
    {
        return CursorFetchStatus::EXHAUSTED;
    }
}
```

---

# 14. Por qué no `null`

Una fila puede contener:

```text
NULL
```

pero eso no significa que el cursor esté agotado.

Aunque `ResultRow` nunca sea `null`, un resultado tipado hace explícito el protocolo.

---

# 15. CursorFetchStatus

```php
enum CursorFetchStatus: string
{
    case ROW = 'row';
    case EXHAUSTED = 'exhausted';
}
```

Los errores se expresarán mediante excepciones/resultados de error de la infraestructura, no como una tercera pseudo-fila.

---

# 16. Estado del cursor

```php
enum CursorState: string
{
    case CREATED = 'created';
    case OPEN = 'open';
    case READING = 'reading';
    case EXHAUSTED = 'exhausted';
    case CANCELLED = 'cancelled';
    case FAILED = 'failed';
    case CLOSED = 'closed';
}
```

---

# 17. State machine

```text
CREATED
   │
   ▼
 OPEN
   │
   ├───────────────┐
   │               │
   ▼               ▼
READING          CLOSED
   │
   ├───────────────┐
   │               │
   ▼               ▼
READING         EXHAUSTED
   │               │
   │               ▼
   │             CLOSED
   │
   ├───────────────► CANCELLED
   │                    │
   │                    ▼
   │                  CLOSED
   │
   └───────────────► FAILED
                        │
                        ▼
                      CLOSED
```

---

# 18. CREATED

El wrapper existe, pero su lectura todavía no ha comenzado.

Dependiendo del driver, el recurso nativo puede estar ya abierto.

---

# 19. OPEN

El cursor está listo para consumir filas.

---

# 20. READING

Al menos una operación de lectura está en progreso o ya ocurrió y quedan potencialmente más filas.

---

# 21. EXHAUSTED

El driver ha indicado inequívocamente:

```text
no more rows
```

---

# 22. CANCELLED

La lectura fue detenida explícitamente.

---

# 23. FAILED

Ocurrió un fallo que impide continuar consumiendo el cursor.

---

# 24. CLOSED

Los recursos controlados por el cursor han sido liberados o desvinculados.

---

# 25. Estado terminal lógico vs cleanup

Estados como:

```text
EXHAUSTED
CANCELLED
FAILED
```

pueden requerir una transición adicional:

```text
→ CLOSED
```

para completar cleanup.

---

# 26. Cursor lifecycle invariant

Todo cursor creado deberá eventualmente alcanzar:

```text
CLOSED
```

salvo transferencia explícita de ownership a otro scope válido.

---

# 27. Forward-only cursor

El baseline portable de VoltStack será:

```text
FORWARD_ONLY
```

Flujo:

```text
row 0
 ↓
row 1
 ↓
row 2
 ↓
row 3
 ↓
EXHAUSTED
```

No existe implícitamente:

```text
↑ previous
```

---

# 28. Razón

Forward-only puede implementarse eficientemente sobre prácticamente cualquier driver sin fingir capacidades inexistentes.

---

# 29. Scrollable cursor

Algunos drivers pueden soportar:

```text
FIRST
LAST
NEXT
PREVIOUS
ABSOLUTE
RELATIVE
```

Esto será capability-driven.

---

# 30. No emulación oculta

Si el driver sólo ofrece forward-only:

```text
previous()
```

no deberá provocar:

```text
buffer all rows
```

silenciosamente.

---

# 31. CursorCapability

```php
enum CursorCapability: string
{
    case FORWARD_ONLY = 'forward_only';

    case SCROLLABLE = 'scrollable';

    case SEEKABLE = 'seekable';

    case REWINDABLE = 'rewindable';

    case POSITION_KNOWN = 'position_known';

    case ROW_COUNT_KNOWN = 'row_count_known';

    case CANCELLABLE = 'cancellable';

    case DETACHABLE = 'detachable';

    case ASYNC_FETCH = 'async_fetch';

    case MULTIPLE_RESULT_SETS = 'multiple_result_sets';

    case COLUMN_METADATA = 'column_metadata';

    case FETCH_SIZE_HINT = 'fetch_size_hint';
}
```

---

# 32. Capabilities ≠ vendor checks

Preferir:

```php
$cursor->capabilities()->supports(
    CursorCapability::SCROLLABLE
);
```

No:

```php
if ($driver === 'pgsql') {
}
```

---

# 33. CursorCapabilitySet

```php
final readonly class CursorCapabilitySet
{
    /**
     * @param array<CursorCapability, CursorCapabilityDescriptor> $capabilities
     */
    public function __construct(
        private array $capabilities,
    ) {}
}
```

---

# 34. Capability descriptor

Algunas capacidades pueden requerir detalles:

```text
supported
mode
limitations
driver guarantee
platform guarantee
```

---

# 35. Cursor mode

```php
enum CursorMode: string
{
    case FORWARD_ONLY = 'forward_only';
    case SCROLLABLE = 'scrollable';
}
```

---

# 36. Default

```text
CursorMode::FORWARD_ONLY
```

deberá ser el modo portable predeterminado.

---

# 37. Cursor position

Podrá existir:

```php
final readonly class CursorPosition
{
    public function __construct(
        public int $ordinal,
    ) {}
}
```

con:

```text
ordinal >= 0
```

para filas consumidas.

---

# 38. Cursor position ≠ primary key

```text
CursorPosition
```

representa posición dentro del result set.

No:

```text
database row identity
```

---

# 39. Cursor position ≠ SQL OFFSET

Aunque puedan coincidir numéricamente:

```text
cursor position
≠
query OFFSET
```

---

# 40. Position states

Internamente pueden existir:

```text
BEFORE_FIRST
AT_ROW
AFTER_LAST
UNKNOWN
```

---

# 41. CursorPositionState

```php
enum CursorPositionState
{
    case BEFORE_FIRST;
    case AT_ROW;
    case AFTER_LAST;
    case UNKNOWN;
}
```

---

# 42. Unknown position

Algunos drivers pueden no proporcionar una posición confiable.

VoltStack no deberá inventarla si el comportamiento real no permite derivarla.

---

# 43. Derived position

En forward-only, VoltStack puede mantener un contador local:

```text
rows successfully delivered
```

sin consultar al driver.

---

# 44. Position semantics

Si el contador comienza antes de la primera fila:

```text
deliveredRows = 0
```

la primera fila puede tener:

```text
ordinal = 0
```

---

# 45. Fetch pipeline

```text
ResultCursor.fetch()
        ↓
Validate Cursor State
        ↓
Check Cancellation
        ↓
Check Budget
        ↓
Driver Cursor Fetch
        ↓
Native Row
        ↓
Result Conversion Plan
        ↓
ResultRow
        ↓
Update Position
        ↓
Telemetry
        ↓
CursorRowFetched
```

Si no existen más filas:

```text
Driver EOF
   ↓
EXHAUSTED
   ↓
cleanup according to policy
```

---

# 46. Driver cursor abstraction

El core deberá depender de un contrato como:

```php
interface DriverResultCursor
{
    public function fetchNext(): DriverCursorFetchResult;

    public function close(): void;
}
```

---

# 47. DriverResultCursor ≠ ResultCursor

```text
DriverResultCursor
=
native-driver abstraction
```

```text
ResultCursor
=
VoltStack normalized cursor
```

---

# 48. Driver cursor adapter

Puede existir:

```text
PDOResultCursor
PgSqlResultCursor
SQLiteResultCursor
MySqlResultCursor
```

dependiendo del driver implementado.

---

# 49. No PDO leakage

Las capas superiores nunca deberán requerir:

```text
PDO::FETCH_ASSOC
PDO::FETCH_NUM
PDO::FETCH_CLASS
```

para consumir resultados VoltStack.

---

# 50. Fetch mode

VoltStack deberá controlar su propia representación de fila.

Preferiblemente el driver entregará una forma suficientemente cruda para construir:

```text
ResultRow
```

sin perder columnas duplicadas.

---

# 51. Associative fetch danger

Esto puede ser incorrecto internamente:

```php
$statement->fetch(PDO::FETCH_ASSOC);
```

cuando dos columnas tienen el mismo label.

---

# 52. Preferred internal representation

Cuando sea posible:

```text
positional native row
+
column metadata
```

---

# 53. Example

SQL:

```sql
SELECT users.id, orders.id
FROM users
JOIN orders ...
```

Native positional row:

```text
[
    42,
    983
]
```

Metadata:

```text
0 → users.id
1 → orders.id
```

Resultado:

```text
ResultRow
├── [0] = 42
└── [1] = 983
```

sin pérdida.

---

# 54. Row conversion plan

Cada cursor podrá recibir:

```text
CompiledResultConversionPlan
```

resuelto antes del hot loop.

---

# 55. Hot path

Ideal:

```text
fetch native row
      ↓
converter[0]
converter[1]
converter[2]
...
      ↓
ResultRow
```

No:

```text
for every cell:
    search registry
    inspect services
    infer everything again
```

---

# 56. CursorExecutionContext

```php
final readonly class CursorExecutionContext
{
    public function __construct(
        public CursorId $cursorId,
        public ResultMetadata $metadata,
        public CompiledResultConversionPlan $conversionPlan,
        public CursorPolicy $policy,
        public CursorBudget $budget,
        public CursorCancellationToken $cancellation,
        public CursorTelemetry $telemetry,
    ) {}
}
```

---

# 57. Context isolation

No deberá contener referencias ocultas a:

```text
current HTTP request
global tenant
global EntityManager
global transaction
service locator
```

---

# 58. Resource model

Un cursor puede depender de:

```text
ConnectionLease
PreparedStatementLease
DriverResultHandle
DriverCursorHandle
TransactionScope
```

---

# 59. Dependency graph

```text
Transaction Scope
       │
       ▼
Connection Lease
       │
       ▼
Prepared Statement Lease
       │
       ▼
Driver Result Handle
       │
       ▼
Driver Cursor
       │
       ▼
ResultCursor
```

No todos los drivers requerirán todas las dependencias, pero el modelo deberá poder representarlas.

---

# 60. ResourceDependencySet

```php
final readonly class CursorResourceDependencySet
{
    /**
     * @param list<CursorResourceDependency> $dependencies
     */
    public function __construct(
        public array $dependencies,
    ) {}
}
```

---

# 61. Dependency kind

```php
enum CursorResourceDependencyKind
{
    case CONNECTION;
    case PREPARED_STATEMENT;
    case DRIVER_RESULT;
    case TRANSACTION;
    case TEMPORARY_BUFFER;
    case EXTENSION_RESOURCE;
}
```

---

# 62. Dependency requirement

Cada dependencia puede indicar:

```text
requiredUntil:
    EXHAUSTED
    CLOSED
    DETACHED
```

---

# 63. Lifetime invariant

Si cursor `C` depende del recurso `R`:

```text
Open(C)
⇒
Available(R)
```

hasta el punto definido por el driver contract.

---

# 64. Connection pinning

Algunos cursors obligarán a mantener una conexión:

```text
PINNED
```

mientras están abiertos.

---

# 65. Cursor pinning ≠ transaction

Una connection puede estar pinned por un cursor aunque no exista una transacción explícita.

---

# 66. Prepared statement pinning

Algunos drivers requieren que el statement permanezca válido durante fetch.

Entonces:

```text
CursorOpen
⇒
PreparedStatementLeaseActive
```

---

# 67. Driver-specific dependency

Estas reglas deberán provenir de:

```text
DriverCursorContract
```

no de condicionales dispersos.

---

# 68. DriverCursorContract

```php
final readonly class DriverCursorContract
{
    public function __construct(
        public bool $requiresOpenStatement,
        public bool $requiresPinnedConnection,
        public bool $requiresActiveTransaction,
        public bool $supportsCancellation,
        public bool $supportsScrollable,
        public bool $supportsFetchSizeHint,
    ) {}
}
```

La implementación real probablemente utilizará value objects/capability descriptors más ricos.

---

# 69. Cursor ownership

```php
enum CursorOwnership
{
    case FRAMEWORK;
    case BORROWED;
    case TRANSFERRED;
}
```

---

# 70. FRAMEWORK

VoltStack debe cerrar el recurso.

---

# 71. BORROWED

VoltStack puede usarlo pero no posee completamente su lifetime.

Debe existir un contrato explícito.

---

# 72. TRANSFERRED

Ownership fue transferido a otro scope controlado.

---

# 73. No ambiguous ownership

Nunca:

```text
"someone will probably close it"
```

---

# 74. Cursor close

```php
$cursor->close();
```

deberá:

1. detener futuras lecturas;
2. cerrar driver cursor/result cuando corresponda;
3. liberar dependencias liberables;
4. actualizar estado;
5. emitir telemetry;
6. eliminarse del resource scope.

---

# 75. close() idempotence

Idealmente:

```text
close();
close();
close();
```

deberá ser seguro.

---

# 76. Reading after close

```php
$cursor->close();
$cursor->fetch();
```

deberá producir:

```text
CursorClosedException
```

---

# 77. Reading after exhaustion

Podrán existir dos políticas:

```text
return CursorExhausted repeatedly
```

o:

```text
throw CursorExhaustedException
```

Para ergonomía y determinismo, se recomienda:

```text
fetch after EXHAUSTED
→ CursorExhausted
```

mientras el cursor no haya sido cerrado.

---

# 78. After CLOSED

Siempre error de lifecycle.

---

# 79. Exhaustion detection

Sólo deberá marcarse `EXHAUSTED` cuando el driver indique que no quedan filas.

---

# 80. Row count ≠ exhaustion

No asumir:

```text
known row count reached
```

como única fuente de verdad si el driver contract requiere otra señal.

---

# 81. Unknown row count

Es perfectamente válido:

```text
rowCount = UNKNOWN
```

durante toda la lectura.

---

# 82. CursorRowCount

```php
final readonly class CursorRowCount
{
    public function __construct(
        public CursorRowCountState $state,
        public ?int $value,
    ) {}
}
```

---

# 83. Row count states

```php
enum CursorRowCountState
{
    case KNOWN;
    case UNKNOWN;
    case PARTIAL;
}
```

---

# 84. PARTIAL

Puede significar:

```text
rows delivered so far = N
```

no:

```text
total rows = N
```

---

# 85. Critical distinction

```text
RowsConsumed
≠
TotalRows
```

hasta que el cursor esté agotado o el driver proporcione total confiable.

---

# 86. Fetch strategies

El sistema podrá modelar:

```text
ROW_BY_ROW
BATCHED_DRIVER_FETCH
BUFFERED_WINDOW
DRIVER_DEFAULT
```

---

# 87. Logical API vs physical fetch

Aunque la API entregue:

```text
one ResultRow
```

el driver podría obtener internamente:

```text
100 rows
```

por round trip.

---

# 88. FetchSize

Podrá existir:

```php
final readonly class CursorFetchSize
{
    public function __construct(
        public int $rows,
    ) {}
}
```

---

# 89. Fetch size is a hint

En general:

```text
FetchSize
=
performance/resource hint
```

no garantía semántica.

---

# 90. Fetch size ≠ LIMIT

```text
fetch size 100
```

no significa:

```sql
LIMIT 100
```

---

# 91. Fetch size ≠ pagination

No altera el conjunto lógico de resultados.

---

# 92. Driver capability

Si el driver ignora fetch size:

```text
capability descriptor
```

deberá reflejarlo cuando sea conocido.

---

# 93. Batched cursor buffer

Puede existir un pequeño buffer interno:

```text
Driver
 ↓
[100 native rows]
 ↓
Cursor internal batch
 ↓
ResultRow one by one
```

---

# 94. Internal buffer ≠ BufferedResult

```text
Cursor internal fetch buffer
```

es una optimización operacional limitada.

```text
BufferedResult
```

materializa el result set como representación durable/replayable.

---

# 95. Buffer budget

Buffers internos también deberán contabilizarse.

---

# 96. CursorBudget

```php
final class CursorBudget
{
    public function consumeRow(): void;

    public function consumeBytes(int $bytes): void;

    public function consumeConversionUnits(int $units): void;

    public function consumeDriverFetch(): void;
}
```

---

# 97. Budget dimensions

Podrán incluir:

```text
max rows
max bytes
max row bytes
max LOB bytes
max fetch operations
max conversion units
max duration
max internal buffer bytes
max metadata bytes
```

---

# 98. Row budget

Puede utilizarse para protección operacional.

No deberá reinterpretarse como SQL `LIMIT`.

---

# 99. Budget exceeded

```text
CursorBudgetExceededException
```

seguido de cleanup/cancellation apropiado.

---

# 100. Partial consumption on budget failure

Si ya se entregaron 1,000 filas:

```text
rowsDelivered = 1000
```

deberá conservarse en diagnostics.

---

# 101. No hidden continuation

Después de budget exhaustion:

```text
cursor
→ FAILED/CANCELLED
→ CLOSED
```

según política.

No deberá continuar silenciosamente.

---

# 102. Large objects

LOBs pueden requerir tratamiento especial:

```text
BLOB
CLOB
BYTEA
large TEXT
```

---

# 103. LOB value ≠ cursor

Un stream de LOB puede tener su propio lifecycle.

El cursor deberá declarar dependencias si un `ResultValue` expone un stream ligado al cursor/connection.

---

# 104. Recommended default

Para evitar lifetimes complejos, la representación estándar deberá favorecer valores detached cuando el tamaño y budget lo permitan.

Streaming LOB será una capability explícita.

---

# 105. Nested resource lifetime

Si:

```text
ResultRow
└── LobStream
```

depende del cursor:

```text
Lifetime(LobStream)
⊆
Lifetime(Cursor)
```

salvo detachment explícito.

---

# 106. Cursor cancellation

Contrato conceptual:

```php
interface CancellableResultCursor extends ResultCursor
{
    public function cancel(): CursorCancellationResult;
}
```

---

# 107. Capability-based API

Alternativamente, el contrato principal puede exponer:

```php
public function cancel(): void;
```

y fallar con:

```text
CursorCancellationUnsupportedException
```

si no está soportado.

Arquitectónicamente se prefiere evitar que capabilities inexistentes parezcan universales.

---

# 108. Cancellation token

El cursor podrá recibir:

```text
CursorCancellationToken
```

proveniente del Execution Engine.

---

# 109. Cancellation sources

Ejemplos:

```text
HTTP request cancelled
application cancellation
query timeout
worker shutdown
execution plan cancellation
resource governance
```

---

# 110. Cancellation propagation

```text
ExecutionCancellation
        ↓
CursorCancellationToken
        ↓
ResultCursor
        ↓
Driver cancellation
        ↓
Resource cleanup
```

cuando el driver lo soporte.

---

# 111. Unsupported driver cancellation

Si no puede cancelar físicamente:

```text
stop consumer delivery
+
close resources as safely as possible
+
classify connection state
```

---

# 112. Cancellation may poison connection

Algunos drivers/protocolos pueden dejar la conexión:

```text
UNSAFE_FOR_REUSE
```

tras cancellation.

---

# 113. Connection impact

El cursor deberá comunicar:

```text
CursorTerminationImpact
```

al Connection Manager.

---

# 114. Termination impact

```php
enum CursorTerminationImpact
{
    case NONE;
    case STATEMENT_INVALID;
    case CONNECTION_REQUIRES_RESET;
    case CONNECTION_NOT_REUSABLE;
    case TRANSACTION_ABORTED;
    case UNKNOWN;
}
```

---

# 115. Driver determines impact

No deberá inferirse únicamente a partir del tipo de excepción PHP.

---

# 116. Cursor failure

Puede ocurrir durante:

```text
driver fetch
network read
result conversion
LOB read
metadata access
cancellation
resource cleanup
```

---

# 117. Failure pipeline

```text
Failure
   ↓
classify
   ↓
mark FAILED
   ↓
record partial progress
   ↓
determine resource impact
   ↓
cleanup
   ↓
propagate normalized exception
```

---

# 118. Error hierarchy

```text
DatabaseCursorException
├── CursorStateException
├── CursorClosedException
├── CursorFailedException
├── CursorExhaustionException
├── CursorCapabilityException
├── CursorUnsupportedOperationException
├── CursorPositionException
├── CursorSeekException
├── CursorFetchException
├── CursorConversionException
├── CursorDriverException
├── CursorResourceException
├── CursorOwnershipException
├── CursorDependencyException
├── CursorCancellationException
├── CursorCancellationUnsupportedException
├── CursorBudgetException
├── CursorBudgetExceededException
├── CursorTimeoutException
├── CursorMetadataException
├── CursorSecurityException
├── CursorExtensionException
└── CursorInvariantException
```

---

# 119. Error context

Una excepción normalizada podrá incluir:

```text
CursorId
CursorState
rows delivered
bytes consumed
position
driver
platform
execution id
statement execution id
resource impact
safe cause classification
```

---

# 120. No row values in exceptions

Por defecto no incluir:

```text
email
password hash
token
private content
binary payload
```

---

# 121. Cursor telemetry

Eventos sugeridos:

```text
CursorCreated
CursorOpened
CursorFetchStarted
CursorRowFetched
CursorBatchFetched
CursorExhausted
CursorCancelled
CursorFailed
CursorClosed
CursorBudgetExceeded
CursorLeakDetected
```

---

# 122. Hot-path telemetry

No deberá emitirse obligatoriamente un evento pesado por cada fila.

Podrán utilizarse:

```text
sampling
aggregation
counters
periodic reporting
```

---

# 123. Cursor metrics

Ejemplos:

```text
db.cursor.open
db.cursor.duration
db.cursor.rows
db.cursor.bytes
db.cursor.fetch.count
db.cursor.fetch.duration
db.cursor.conversion.duration
db.cursor.failures
db.cursor.cancellations
db.cursor.budget_exceeded
db.cursor.leaks
```

---

# 124. Metric labels

Permitidos:

```text
driver
platform
cursor mode
result kind
termination reason
```

con cardinalidad controlada.

---

# 125. Forbidden metric labels

No:

```text
customer_id
email
token
raw SQL
row value
random CursorId
```

---

# 126. Cursor diagnostics

En debug:

```text
Cursor
├── ID
├── state
├── mode
├── capabilities
├── rows delivered
├── bytes consumed
├── current position
├── open duration
├── statement dependency
├── connection dependency
├── transaction dependency
├── fetch size
├── budget
└── termination impact
```

---

# 127. SQL correlation

Podrá correlacionarse con:

```text
ExecutionId
ExecutionUnitId
StatementExecutionId
CompiledCommandFingerprint
```

---

# 128. CursorId ≠ command fingerprint

```text
CursorId
=
runtime resource identity
```

```text
CompiledCommandFingerprint
=
structural compiled command identity
```

---

# 129. Security

Cursor System deberá tratar todos los datos leídos como potencialmente sensibles.

---

# 130. Cursor itself carries no authorization policy

La autorización de ejecutar la query debió resolverse antes.

Pero el cursor debe preservar metadata de sensibilidad necesaria para:

```text
redaction
telemetry
diagnostics
result handling
```

---

# 131. Security predicate preservation

Cursor System jamás podrá:

```text
drop tenant filters
drop authorization predicates
```

porque no modifica la query.

---

# 132. Result mutation forbidden

No deberá transformar:

```text
row A
```

en:

```text
row B
```

excepto conversiones de representación autorizadas por el Result Type System.

---

# 133. Conversion semantics

```text
"42" → 42
```

puede ser válido.

Pero:

```text
salary = 1000 → salary = 0
```

no es una conversión de representación genérica.

---

# 134. Row filtering forbidden

Cursor System no deberá ocultar filas basándose en lógica de aplicación.

---

# 135. Why

Eso rompería:

```text
cardinality
pagination
aggregation semantics
telemetry
consistency
```

La seguridad row-level debe formar parte de la query/plan.

---

# 136. Cursor decorators

Podrán existir decoradores controlados para:

```text
telemetry
budget
diagnostics
conversion
cancellation
```

pero no para cambiar arbitrariamente la semántica del result set.

---

# 137. Recommended composition

```text
DriverResultCursor
       ↓
ConvertingCursor
       ↓
BudgetedCursor
       ↓
TelemetryCursor
       ↓
ResultCursor
```

o una implementación integrada equivalente.

---

# 138. Avoid decorator explosion

La arquitectura no obliga a crear un objeto por preocupación.

El modelo conceptual puede implementarse eficientemente mediante una sesión coordinada.

---

# 139. CursorSession

```php
final class CursorSession
{
    private CursorState $state;

    private int $rowsDelivered = 0;

    private int $bytesConsumed = 0;

    // operation-local lifecycle/resources
}
```

---

# 140. CursorSession scope

```text
one ResultCursor
=
one CursorSession
```

salvo implementación explícitamente compartida y segura.

---

# 141. No singleton cursor session

Nunca:

```php
CursorSession::current()
```

global.

---

# 142. Persistent runtime

En FrankenPHP:

```text
Worker
├── Request A
│   └── Cursor A
│
└── Request B
    └── Cursor B
```

Al terminar A:

```text
Cursor A = CLOSED
references A = released
```

---

# 143. Worker reuse invariant

```text
State(Cursor A)
∩
State(Cursor B)
=
∅
```

para estado mutable operation-specific.

---

# 144. RoadRunner

La misma regla aplica a workers persistentes.

---

# 145. OpenSwoole

Con coroutines:

```text
Coroutine A → CursorSession A
Coroutine B → CursorSession B
```

sin compartir:

```text
current position
current row
current native cursor
budgets
cancellation state
```

---

# 146. Concurrency

Un cursor individual no deberá asumirse thread/coroutine-safe para lecturas concurrentes.

---

# 147. Default concurrency rule

```text
one cursor
→ one active consumer
```

---

# 148. Concurrent fetch

Dos operaciones simultáneas:

```text
fetch A
fetch B
```

sobre el mismo cursor deberán:

```text
serialize
```

o:

```text
fail
```

según implementation contract.

---

# 149. Recommended V1

Fail-fast:

```text
ConcurrentCursorConsumptionException
```

---

# 150. Why

Evita resultados no deterministas:

```text
Consumer A gets row?
Consumer B gets row?
```

---

# 151. Cursor lease

Podrá existir:

```text
CursorConsumptionLease
```

para garantizar un único consumidor.

---

# 152. Cursor iterator adapter

API de conveniencia:

```php
foreach ($cursor->iterate() as $row) {
    // ...
}
```

---

# 153. Iterator adapter behavior

Debe:

```text
fetch
yield
fetch
yield
...
```

hasta exhaustion.

---

# 154. Iterator break

Si el consumidor hace:

```php
foreach ($cursor->iterate() as $row) {
    break;
}
```

debe quedar clara la política de lifecycle.

---

# 155. Recommended iterator ownership

Una API:

```text
cursor->iterate()
```

no debería cerrar automáticamente un cursor externally-owned simplemente porque el loop termina temprano.

---

# 156. Managed iteration

Puede existir una API separada:

```php
$cursor->consume(function (ResultRow $row) {
    // ...
});
```

con ownership explícito:

```text
open
consume
finally close
```

---

# 157. Convenience APIs must declare ownership

No ocultar si una operación:

```text
closes cursor
keeps cursor open
buffers remaining rows
```

---

# 158. Rewind behavior

PHP Iterator normalmente permite:

```text
rewind()
```

pero un cursor forward-only no.

---

# 159. No fake rewind

El adapter deberá impedir la ilusión de rewind.

Puede usar:

```text
IteratorAggregate
```

con semántica single-pass controlada, en lugar de implementar una interfaz incompatible con las garantías reales.

---

# 160. Scrollable cursor API

Contrato opcional:

```php
interface ScrollableResultCursor extends ResultCursor
{
    public function first(): CursorFetchResult;

    public function last(): CursorFetchResult;

    public function previous(): CursorFetchResult;

    public function absolute(int $position): CursorFetchResult;

    public function relative(int $offset): CursorFetchResult;
}
```

---

# 161. Scrollable cursor semantics

Estas operaciones sólo existen si el driver/cursor contract garantiza comportamiento definido.

---

# 162. Seekable ≠ rewindable

Un cursor podría soportar:

```text
absolute forward seek
```

pero no rewind completo.

Capabilities deberán ser precisas.

---

# 163. Scrollable cursor ≠ buffered result

Aunque ambos permitan revisitar filas:

```text
Scrollable Cursor
```

puede seguir dependiendo del DB/driver.

```text
Buffered Result
```

es una representación materializada.

---

# 164. Detachment

Un cursor puede potencialmente convertirse en un resultado detached:

```text
Cursor
 ↓
consume remaining rows
 ↓
BufferedResult
 ↓
close cursor
```

---

# 165. Detachment is explicit

Nunca automáticamente para implementar:

```text
rewind
count
previous
```

---

# 166. detach()

Podrá existir en una capa superior:

```php
$buffered = $cursor->materialize($policy);
```

sujeto a budget.

---

# 167. Materialization failure

Si se supera memoria/budget:

```text
ResultMaterializationException
```

y el cursor puede quedar parcialmente consumido.

La API deberá declarar esta consecuencia.

---

# 168. Cursor counting

No deberá implementarse:

```php
$count = count($cursor);
```

consumiendo implícitamente todas las filas.

---

# 169. Count semantics

Podrán existir:

```text
knownTotal()
rowsConsumed()
```

como conceptos distintos.

---

# 170. `rowCount()` danger

APIs de drivers como:

```text
PDOStatement::rowCount()
```

no tienen semántica portable para SELECT.

VoltStack no deberá exponerlas como verdad universal.

---

# 171. Cursor count invariant

```text
UNKNOWN
```

es preferible a un número incorrecto.

---

# 172. Transaction dependency

Algunos DBMS/cursors pueden requerir una transacción activa para streaming/server-side cursor.

---

# 173. Transaction-bound cursor

```text
CursorOpen
⇒
TransactionActive
```

si `DriverCursorContract` lo exige.

---

# 174. Commit with open cursor

La política deberá ser explícita.

Opciones posibles según driver:

```text
commit forbidden
cursor auto-closed
cursor survives commit
driver-defined
```

---

# 175. No universal assumption

Transaction Manager deberá consultar resource dependencies/capabilities.

---

# 176. Rollback impact

Rollback puede invalidar:

```text
open cursors
statements
result handles
```

El Cursor System deberá recibir la invalidación correspondiente.

---

# 177. Resource invalidation

Podrá existir:

```php
interface InvalidatableCursor
{
    public function invalidate(
        CursorInvalidationReason $reason
    ): void;
}
```

---

# 178. Invalidation reasons

```text
TRANSACTION_COMMIT
TRANSACTION_ROLLBACK
CONNECTION_RESET
CONNECTION_LOST
STATEMENT_CLOSED
WORKER_SHUTDOWN
REQUEST_TERMINATED
```

---

# 179. Invalidated cursor

No deberá seguir intentando fetch.

---

# 180. Connection reset

Antes de devolver una connection al pool:

```text
open dependent cursors = 0
```

o deberán ser cerrados/invalidados según policy.

---

# 181. Pool integration

```text
ResultCursor
     ↓
Resource Registry
     ↓
Connection Lease
     ↓
Connection Pool
```

El pool no deberá reutilizar conexiones con recursos incompatibles abiertos.

---

# 182. Cursor registry

Cada connection lease podrá mantener conocimiento de:

```text
dependent resources
```

sin que el cursor dependa directamente del pool concreto.

---

# 183. Request resource scope

También podrá existir:

```text
DatabaseResourceScope
```

que registre:

```text
open connections
open statements
open results
open cursors
```

---

# 184. Scope termination

```text
request ends
   ↓
cancel/close cursors
   ↓
close results
   ↓
release statements
   ↓
reset connections
   ↓
return safe connections
```

---

# 185. Cleanup ordering

Debe respetar dependencias.

No:

```text
release connection
↓
close cursor
```

si el cursor necesita esa conexión para cerrarse correctamente.

---

# 186. Topological cleanup

Conceptualmente:

```text
Cursor
↓
Driver Result
↓
Prepared Statement
↓
Connection
```

---

# 187. Cleanup errors

Un error cerrando cursor no deberá impedir intentar cleanup de recursos restantes.

---

# 188. Composite cleanup failure

Podrá utilizarse:

```text
ResourceCleanupReport
```

para registrar múltiples fallos sin perder el error original.

---

# 189. Primary exception preservation

Si:

```text
fetch fails
+
close fails
```

la excepción principal deberá seguir siendo:

```text
fetch failure
```

con cleanup failure adjunto/suprimido.

---

# 190. Cursor timeout

Cursor consumption puede estar sujeto a:

```text
query timeout
fetch timeout
total consumption deadline
idle cursor timeout
```

---

# 191. Timeout types

No deben confundirse.

```text
QueryExecutionTimeout
≠
CursorFetchTimeout
≠
CursorIdleTimeout
≠
CursorLifetimeTimeout
```

---

# 192. Cursor idle timeout

Especialmente útil para evitar:

```text
consumer forgets cursor
→ connection remains pinned indefinitely
```

---

# 193. Runtime governance

Un Resource Governor podrá detectar cursors abiertos demasiado tiempo.

---

# 194. Timeout policy

El Cursor System aplicará políticas declaradas, no valores mágicos internos.

---

# 195. Deadline

Preferible modelar:

```text
absolute monotonic deadline
```

internamente cuando sea posible.

---

# 196. Wall clock

No utilizar wall clock para medir duración si existe monotonic clock.

---

# 197. Clock dependency

Puede inyectarse:

```text
MonotonicClock
```

como infraestructura explícita.

---

# 198. Cursor extension system

Extensiones permitidas podrán añadir:

```text
driver cursor adapters
cursor capability descriptors
diagnostic enrichers
telemetry observers
specialized result conversion support
extension resource dependencies
```

---

# 199. Extension prohibition

Una extensión no podrá:

```text
silently execute another query
silently buffer entire cursor
reorder rows
remove rows
duplicate rows
alter authorization semantics
retain cursor globally
bypass cancellation
bypass budget
```

---

# 200. Extension descriptor

```php
interface CursorExtension
{
    public function descriptor(): CursorExtensionDescriptor;

    public function contributions(): CursorExtensionContributions;
}
```

---

# 201. Extension registry

Se construirá:

```text
bootstrap
 ↓
discover
 ↓
validate
 ↓
resolve dependencies
 ↓
detect conflicts
 ↓
compose
 ↓
freeze
```

---

# 202. Persistent-safe extension state

Shared extension objects deberán ser:

```text
immutable
```

o:

```text
stateless
```

El estado mutable deberá vivir en:

```text
CursorSession
```

---

# 203. Testing architecture

El Cursor System requerirá suites de:

```text
unit tests
driver conformance tests
integration tests
resource lifecycle tests
failure injection tests
cancellation tests
persistent worker tests
concurrency tests
large dataset tests
security tests
performance tests
```

---

# 204. Core test cases

Cada driver deberá demostrar al menos:

```text
empty result
one row
many rows
NULL values
duplicate labels
large values
early close
full exhaustion
fetch failure
conversion failure
connection loss
statement invalidation
cancellation
budget exhaustion
request cleanup
```

---

# 205. Forward-only conformance

Todos los drivers oficiales deberán soportar correctamente el baseline:

```text
FORWARD_ONLY
```

para row results que permitan cursor.

---

# 206. Optional capabilities tests

Sólo se ejecutarán cuando:

```text
capability = supported
```

---

# 207. Capability truthfulness

Si un driver declara:

```text
SCROLLABLE = true
```

la conformance suite deberá demostrarlo.

---

# 208. No optimistic capability declaration

Una capacidad desconocida deberá ser:

```text
UNKNOWN / UNSUPPORTED
```

según modelo, no asumida como disponible.

---

# 209. Performance objectives

El hot path deberá minimizar:

```text
allocations
registry lookups
reflection
container resolution
metadata reconstruction
telemetry overhead
```

---

# 210. Precomputation

Antes del primer fetch podrán resolverse:

```text
column metadata
conversion plan
capabilities
resource dependencies
telemetry context
budget policy
```

---

# 211. Fetch complexity

Para una fila de `c` columnas:

```text
T(fetch row)
≈
T(driver fetch)
+
O(c)
+
T(value conversions)
```

---

# 212. Cursor memory complexity

Para forward-only sin LOB/batch significativo:

```text
Memory
≈
O(row width)
```

más metadata y buffers controlados.

---

# 213. Batched fetch memory

Con batch de `b` filas:

```text
Memory
≈
O(b × average row width)
```

---

# 214. Buffered result comparison

Para `n` filas:

```text
BufferedResult Memory
≈
O(n × average row width)
```

Esta diferencia justifica el Cursor System.

---

# 215. Backpressure foundation

El cursor naturalmente permite:

```text
consumer asks for row
↓
cursor fetches/provides row
```

lo que servirá de base para el Streaming Result System.

---

# 216. Cursor pull model

V1 deberá favorecer:

```text
PULL
```

sobre un modelo push indiscriminado.

---

# 217. Why pull

Permite que el consumidor controle:

```text
pace
memory
cancellation
processing
```

---

# 218. Async future

Un futuro runtime puede soportar:

```text
fetchAsync()
```

pero deberá ser capability-driven.

---

# 219. Async cursor ≠ concurrent cursor

```text
asynchronous fetch
```

no implica:

```text
multiple simultaneous consumers
```

---

# 220. Reactive integration

El futuro sistema reactivo de VoltStack podrá construir:

```text
ResultCursor
↓
StreamingResult
↓
Async/Reactive Adapter
```

sin cambiar el contrato semántico del cursor.

---

# 221. Public API example

```php
$cursor = $result->cursor();

try {
    while (true) {
        $fetch = $cursor->fetch();

        if ($fetch->status() === CursorFetchStatus::EXHAUSTED) {
            break;
        }

        $row = $fetch->row;

        // consume row
    }
} finally {
    $cursor->close();
}
```

---

# 222. Convenience iteration

```php
foreach ($cursor->iterate() as $row) {
    // consume
}
```

será una capa ergonómica sobre el mismo protocolo.

---

# 223. Managed consumption

Una API futura puede ofrecer:

```php
$cursor->consume(
    function (ResultRow $row): void {
        // ...
    }
);
```

con cleanup garantizado mediante `finally`.

---

# 224. Early termination

```php
$cursor->consume(
    function (ResultRow $row): CursorDecision {
        if ($row->get('id') === 100) {
            return CursorDecision::STOP;
        }

        return CursorDecision::CONTINUE;
    }
);
```

---

# 225. STOP semantics

`STOP` no significa:

```text
EXHAUSTED
```

Significa:

```text
consumer terminated early
```

El cursor deberá cerrarse/cancelarse según política.

---

# 226. TerminationReason

```php
enum CursorTerminationReason
{
    case EXHAUSTED;
    case CONSUMER_STOP;
    case CANCELLED;
    case BUDGET_EXCEEDED;
    case TIMEOUT;
    case FAILURE;
    case REQUEST_END;
    case WORKER_SHUTDOWN;
    case EXPLICIT_CLOSE;
}
```

---

# 227. Telemetry uses termination reason

Esto permite distinguir:

```text
query fully consumed
```

de:

```text
consumer stopped after first row
```

---

# 228. First-row convenience

Una API como:

```php
$result->first();
```

puede consumir una fila y cerrar el cursor explícitamente.

---

# 229. first() ≠ LIMIT 1

Si la query original devuelve 1 millón de filas:

```text
first()
```

sobre el resultado no cambia la query ejecutada.

---

# 230. Performance recommendation

Si sólo se necesita una fila:

```text
Query Builder
→ LIMIT 1
```

es preferible a ejecutar un result set grande y consumir sólo la primera.

---

# 231. Cursor has no query rewrite authority

El cursor no añadirá:

```sql
LIMIT 1
```

retroactivamente.

---

# 232. Cursor and N+1

Cursor System no detecta por sí mismo N+1 ORM.

Eso pertenece a:

```text
ORM telemetry
N+1 detection system
```

---

# 233. Cursor and pagination

No deberá generar:

```text
next page
previous page
cursor token
```

Ese sistema pertenece a:

```text
199_DATABASE_PAGINATION_SYSTEM.md
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

---

# 234. Important terminology

`ResultCursor` de este documento:

```text
runtime database result cursor
```

mientras `Cursor Pagination` significa:

```text
pagination strategy based on stable ordering/key boundaries
```

Son conceptos distintos.

---

# 235. Cursor pagination ≠ Result cursor

```text
Database Result Cursor
≠
Pagination Cursor Token
```

---

# 236. Namespace propuesto

```text
VoltStack\Quantum\Database\Execution\Result\Cursor
```

---

# 237. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Result/
                └── Cursor/
                    ├── Contract/
                    │   ├── ResultCursor.php
                    │   ├── DriverResultCursor.php
                    │   ├── ScrollableResultCursor.php
                    │   └── CursorExtension.php
                    │
                    ├── Cursor/
                    │   ├── DefaultResultCursor.php
                    │   ├── ForwardOnlyResultCursor.php
                    │   └── DefaultScrollableResultCursor.php
                    │
                    ├── Fetch/
                    │   ├── CursorFetchResult.php
                    │   ├── CursorRowFetched.php
                    │   ├── CursorExhausted.php
                    │   ├── CursorFetchStatus.php
                    │   ├── CursorFetchStrategy.php
                    │   └── CursorFetchSize.php
                    │
                    ├── Position/
                    │   ├── CursorPosition.php
                    │   ├── CursorPositionState.php
                    │   └── CursorPositionTracker.php
                    │
                    ├── Capability/
                    │   ├── CursorCapability.php
                    │   ├── CursorCapabilitySet.php
                    │   └── CursorCapabilityDescriptor.php
                    │
                    ├── Lifecycle/
                    │   ├── CursorState.php
                    │   ├── CursorLifecycle.php
                    │   ├── CursorTerminationReason.php
                    │   ├── CursorTerminationImpact.php
                    │   └── CursorInvalidationReason.php
                    │
                    ├── Resource/
                    │   ├── CursorOwnership.php
                    │   ├── CursorResourceDependency.php
                    │   ├── CursorResourceDependencyKind.php
                    │   ├── CursorResourceDependencySet.php
                    │   └── CursorResourceRegistry.php
                    │
                    ├── Driver/
                    │   ├── DriverCursorContract.php
                    │   ├── DriverCursorFetchResult.php
                    │   └── DriverCursorAdapter.php
                    │
                    ├── Conversion/
                    │   ├── CursorRowConverter.php
                    │   └── CursorConversionContext.php
                    │
                    ├── Session/
                    │   ├── CursorId.php
                    │   ├── CursorSession.php
                    │   └── CursorConsumptionLease.php
                    │
                    ├── Budget/
                    │   ├── CursorBudget.php
                    │   ├── CursorBudgetPolicy.php
                    │   └── CursorBudgetSnapshot.php
                    │
                    ├── Cancellation/
                    │   ├── CursorCancellationToken.php
                    │   ├── CursorCancellationSource.php
                    │   └── CursorCancellationResult.php
                    │
                    ├── Timeout/
                    │   ├── CursorDeadline.php
                    │   ├── CursorTimeoutPolicy.php
                    │   └── CursorIdleTimeoutPolicy.php
                    │
                    ├── Iteration/
                    │   ├── CursorIterator.php
                    │   ├── CursorDecision.php
                    │   └── ManagedCursorConsumer.php
                    │
                    ├── Diagnostic/
                    │   ├── CursorDiagnostic.php
                    │   ├── CursorLeakDiagnostic.php
                    │   └── CursorFailureDiagnostic.php
                    │
                    ├── Telemetry/
                    │   ├── CursorTelemetry.php
                    │   ├── CursorMetrics.php
                    │   └── CursorObserver.php
                    │
                    ├── Extension/
                    │   ├── CursorExtensionDescriptor.php
                    │   ├── CursorExtensionContributions.php
                    │   └── CursorExtensionRegistry.php
                    │
                    └── Exception/
                        ├── DatabaseCursorException.php
                        ├── CursorStateException.php
                        ├── CursorClosedException.php
                        ├── CursorFailedException.php
                        ├── CursorCapabilityException.php
                        ├── CursorUnsupportedOperationException.php
                        ├── CursorPositionException.php
                        ├── CursorSeekException.php
                        ├── CursorFetchException.php
                        ├── CursorConversionException.php
                        ├── CursorDriverException.php
                        ├── CursorResourceException.php
                        ├── CursorOwnershipException.php
                        ├── CursorDependencyException.php
                        ├── CursorCancellationException.php
                        ├── CursorCancellationUnsupportedException.php
                        ├── CursorBudgetException.php
                        ├── CursorBudgetExceededException.php
                        ├── CursorTimeoutException.php
                        ├── CursorMetadataException.php
                        ├── ConcurrentCursorConsumptionException.php
                        ├── CursorSecurityException.php
                        ├── CursorExtensionException.php
                        └── CursorInvariantException.php
```

---

# 238. Dependency rules

Permitido:

```text
ResultCursor
    ↓
ResultRow
    ↓
Result Type Conversion
```

Permitido:

```text
ResultCursor
    ↓
DriverResultCursor contract
```

Permitido:

```text
ResultCursor
    ↓
Resource Lifecycle abstractions
```

No permitido:

```text
ResultCursor
    ↓
ORM EntityManager
```

No permitido:

```text
ResultCursor
    ↓
Query Optimizer
```

No permitido:

```text
ResultCursor
    ↓
SQL Compiler
```

---

# 239. Architectural invariants

## DB-CURSOR-001

`ResultCursor` será distinto de `DatabaseResult`.

## DB-CURSOR-002

`ResultCursor` será distinto de un cursor nativo del driver.

## DB-CURSOR-003

`ResultCursor` será distinto de `BufferedResult`.

## DB-CURSOR-004

`ResultCursor` será distinto de `StreamingResult`.

## DB-CURSOR-005

`ResultCursor` será distinto de una colección ORM.

## DB-CURSOR-006

Un cursor consumirá un resultado ya ejecutado.

## DB-CURSOR-007

Un cursor no ejecutará nuevamente la query.

## DB-CURSOR-008

Un cursor no generará SQL.

## DB-CURSOR-009

Un cursor no preparará statements.

## DB-CURSOR-010

Un cursor no realizará parameter binding.

## DB-CURSOR-011

Un cursor no hidratará entidades.

## DB-CURSOR-012

El baseline portable será forward-only.

## DB-CURSOR-013

Scrollable behavior será capability-driven.

## DB-CURSOR-014

Un cursor forward-only no simulará rewind mediante buffering oculto.

## DB-CURSOR-015

`fetch()` distinguirá fila de exhaustion.

## DB-CURSOR-016

SQL NULL será distinto de cursor exhaustion.

## DB-CURSOR-017

Cursor lifecycle será explícito.

## DB-CURSOR-018

Todo cursor framework-owned deberá eventualmente cerrarse.

## DB-CURSOR-019

Lectura después de CLOSED fallará.

## DB-CURSOR-020

EXHAUSTED será distinto de CLOSED.

## DB-CURSOR-021

CANCELLED será distinto de EXHAUSTED.

## DB-CURSOR-022

FAILED será distinto de CANCELLED.

## DB-CURSOR-023

Capabilities serán explícitas.

## DB-CURSOR-024

Capabilities no dependerán de vendor conditionals en consumers.

## DB-CURSOR-025

Cursor position será distinta de database row identity.

## DB-CURSOR-026

Cursor position será distinta de SQL OFFSET.

## DB-CURSOR-027

Unknown position no será inventada.

## DB-CURSOR-028

El orden de filas será preservado.

## DB-CURSOR-029

Las filas duplicadas serán preservadas.

## DB-CURSOR-030

El cursor no filtrará filas.

## DB-CURSOR-031

El cursor no ordenará filas.

## DB-CURSOR-032

El cursor no deduplicará filas.

## DB-CURSOR-033

Driver cursor será encapsulado.

## DB-CURSOR-034

PDO-specific fetch modes no escaparán a capas superiores.

## DB-CURSOR-035

Duplicate column labels no deberán perder información.

## DB-CURSOR-036

La representación interna favorecerá posiciones + metadata.

## DB-CURSOR-037

Row conversion será distinta de ORM hydration.

## DB-CURSOR-038

Conversion plans podrán pre-resolverse.

## DB-CURSOR-039

No se realizará registry scan por cada cell cuando pueda evitarse.

## DB-CURSOR-040

Cursor resources tendrán ownership explícito.

## DB-CURSOR-041

Cursor dependencies serán explícitas.

## DB-CURSOR-042

Statement no será liberado prematuramente.

## DB-CURSOR-043

Connection no será liberada prematuramente.

## DB-CURSOR-044

Transaction dependency será explícita.

## DB-CURSOR-045

Connection pinning será distinto de transaction ownership.

## DB-CURSOR-046

Cursor close será seguro.

## DB-CURSOR-047

Cursor close será idealmente idempotente.

## DB-CURSOR-048

Exhaustion sólo será declarada cuando esté confirmada.

## DB-CURSOR-049

Unknown row count será distinto de zero.

## DB-CURSOR-050

RowsConsumed será distinto de TotalRows.

## DB-CURSOR-051

Fetch size será distinto de SQL LIMIT.

## DB-CURSOR-052

Fetch size será distinto de pagination.

## DB-CURSOR-053

Fetch size será un hint operacional.

## DB-CURSOR-054

Internal batch buffer será distinto de BufferedResult.

## DB-CURSOR-055

Internal buffers estarán sujetos a budgets.

## DB-CURSOR-056

Cursor budgets no cambiarán la semántica de la query.

## DB-CURSOR-057

Budget exhaustion será explícita.

## DB-CURSOR-058

Budget exhaustion provocará termination/cleanup apropiado.

## DB-CURSOR-059

LOB resource lifetimes serán explícitos.

## DB-CURSOR-060

Streaming LOB será capability-driven.

## DB-CURSOR-061

Cancellation será capability-aware.

## DB-CURSOR-062

Cancellation se propagará al driver cuando sea posible.

## DB-CURSOR-063

Cancellation no será tratada como ordinary exhaustion.

## DB-CURSOR-064

Cancellation podrá afectar la reutilización de connection.

## DB-CURSOR-065

Termination impact será comunicado.

## DB-CURSOR-066

Fetch failures serán normalizados.

## DB-CURSOR-067

Partial progress será preservado en diagnostics.

## DB-CURSOR-068

Cleanup ocurrirá después de fetch failure.

## DB-CURSOR-069

Cleanup failures no ocultarán el error principal.

## DB-CURSOR-070

Cursor telemetry no registrará row values por defecto.

## DB-CURSOR-071

Hot-path telemetry podrá agregarse o samplearse.

## DB-CURSOR-072

Metric labels tendrán cardinalidad controlada.

## DB-CURSOR-073

CursorId será distinto de command fingerprint.

## DB-CURSOR-074

Cursor System tratará datos de DB como potencialmente sensibles.

## DB-CURSOR-075

Cursor System no realizará authorization decisions tardías.

## DB-CURSOR-076

Cursor System preservará sensitivity metadata.

## DB-CURSOR-077

Cursor System no cambiará semantic row values salvo conversiones autorizadas.

## DB-CURSOR-078

Cursor decorators no podrán cambiar result semantics.

## DB-CURSOR-079

Cursor mutable state será operation-local.

## DB-CURSOR-080

CursorSession no será singleton global.

## DB-CURSOR-081

Persistent workers no compartirán cursor state entre requests.

## DB-CURSOR-082

Coroutines no compartirán current cursor state.

## DB-CURSOR-083

Un cursor tendrá por defecto un único consumidor activo.

## DB-CURSOR-084

Concurrent fetch no será ambiguo.

## DB-CURSOR-085

V1 deberá fallar ante concurrent cursor consumption no soportado.

## DB-CURSOR-086

Iterator será sólo una API de conveniencia.

## DB-CURSOR-087

Iterator no deberá fingir rewind capability.

## DB-CURSOR-088

Convenience APIs declararán ownership/cleanup semantics.

## DB-CURSOR-089

Scrollable cursor será distinto de BufferedResult.

## DB-CURSOR-090

Detachment será explícito.

## DB-CURSOR-091

Detachment estará sujeto a budget.

## DB-CURSOR-092

Counting no consumirá implícitamente un cursor salvo API explícita.

## DB-CURSOR-093

Driver rowCount no será tratado como portable SELECT count.

## DB-CURSOR-094

UNKNOWN será preferible a un count incorrecto.

## DB-CURSOR-095

Transaction Manager deberá respetar cursor dependencies.

## DB-CURSOR-096

Rollback podrá invalidar cursors.

## DB-CURSOR-097

Connection reset podrá invalidar cursors.

## DB-CURSOR-098

Invalidated cursor no continuará fetching.

## DB-CURSOR-099

Connection pool no reutilizará connection con cursors incompatibles abiertos.

## DB-CURSOR-100

Request resource scope podrá registrar cursors abiertos.

## DB-CURSOR-101

Request termination cerrará cursors framework-owned.

## DB-CURSOR-102

Cleanup respetará dependency ordering.

## DB-CURSOR-103

Cursor timeout será distinto de query execution timeout.

## DB-CURSOR-104

Idle timeout será distinto de total cursor lifetime timeout.

## DB-CURSOR-105

Duraciones deberán usar monotonic time cuando sea posible.

## DB-CURSOR-106

Cursor extensions serán tipadas.

## DB-CURSOR-107

Extension registries serán frozen.

## DB-CURSOR-108

Extensions no ejecutarán hidden queries.

## DB-CURSOR-109

Extensions no bufferizarán todo el cursor silenciosamente.

## DB-CURSOR-110

Extensions no reordenarán filas.

## DB-CURSOR-111

Extensions no eliminarán filas.

## DB-CURSOR-112

Extensions no duplicarán filas.

## DB-CURSOR-113

Extensions no evadirán budgets.

## DB-CURSOR-114

Extensions no evadirán cancellation.

## DB-CURSOR-115

Extensions no retendrán cursor resources globalmente.

## DB-CURSOR-116

Official drivers deberán superar forward-only conformance.

## DB-CURSOR-117

Optional capability claims deberán probarse.

## DB-CURSOR-118

Hot path minimizará reflection.

## DB-CURSOR-119

Hot path minimizará container lookups.

## DB-CURSOR-120

Hot path minimizará repeated metadata resolution.

## DB-CURSOR-121

Forward-only memory deberá mantenerse acotada por fila/buffer.

## DB-CURSOR-122

Cursor pull model será el baseline de V1.

## DB-CURSOR-123

Async fetch será capability-driven.

## DB-CURSOR-124

Async fetch será distinto de concurrent consumption.

## DB-CURSOR-125

Result cursor será distinto de pagination cursor.

## DB-CURSOR-126

Cursor System será independiente del ORM.

## DB-CURSOR-127

Cursor System será independiente de Query Optimizer.

## DB-CURSOR-128

Cursor System será independiente de SQL Compiler.

## DB-CURSOR-129

Cursor System será driver-agnostic mediante contratos.

## DB-CURSOR-130

ResultCursor será la abstracción estándar de consumo incremental de filas dentro del Execution Engine.

---

# 240. Invariante maestro de semántica

Sea:

```text
R = ordered result produced by database
```

y:

```text
C = rows delivered by ResultCursor
```

si el cursor alcanza `EXHAUSTED` sin error:

```text
C = R
```

preservando:

```text
order
duplicates
NULLs
column structure
semantic values
```

---

# 241. Invariante de consumo parcial

Si el consumidor detiene el cursor después de `k` filas:

```text
C = prefix(R, k)
```

No deberá haber:

```text
reordering
skipping
duplication
```

causados por el Cursor System.

---

# 242. Invariante de posición

Para un cursor forward-only exitoso:

```text
fetch_i
→
row_i
```

y:

```text
position(row_i) = i
```

para:

```text
i = 0 ... n-1
```

cuando VoltStack mantenga posición local.

---

# 243. Invariante de lifetime

Si:

```text
Cursor C
```

depende de:

```text
Statement S
Connection K
```

entonces mientras dichas dependencias sean requeridas:

```text
Open(C)
⇒
Alive(S)
∧
Alive(K)
```

---

# 244. Invariante de cleanup

Para un cursor framework-owned:

```text
Terminate(C)
⇒
EventuallyClosed(C)
```

donde:

```text
Terminate
=
Exhaustion
∨ ExplicitClose
∨ Cancellation
∨ Failure
∨ ScopeTermination
```

---

# 245. Invariante de memoria

Para forward-only con batch máximo `b`:

```text
CursorMemory
≤
Metadata
+
CurrentRow
+
InternalBatch(b)
+
BoundedOperationalState
```

No deberá crecer linealmente con todo el result set salvo materialización explícita.

---

# 246. Invariante de no reejecución

```text
CursorOperation
⇒
0 hidden query re-executions
```

---

# 247. Invariante de seguridad

```text
CursorTelemetry
∩
SensitiveRowValues
=
∅
```

por defecto.

---

# 248. Invariante de aislamiento

Para dos operation scopes diferentes:

```text
MutableCursorState(A)
∩
MutableCursorState(B)
=
∅
```

---

# 249. Invariante de capabilities

```text
RequestedCursorOperation
⇒
DeclaredCapability
```

o la operación deberá fallar explícitamente.

---

# 250. Invariante de driver portability

```text
Application Cursor Semantics
```

no deberá depender de APIs específicas como:

```text
PDOStatement
mysqli_result
PgSql\Result
SQLite3Result
```

---

# 251. Ejemplo completo

Query:

```sql
SELECT id, name, balance
FROM accounts
ORDER BY id
```

Driver produce progresivamente:

```text
["1", "Alice", "1250.50"]
["2", "Bob",   "820.25"]
["3", "Carol", "100.00"]
EOF
```

Conversion plan:

```text
column 0 → INTEGER
column 1 → STRING
column 2 → DECIMAL
```

Cursor:

```text
OPEN
 ↓
fetch
 ↓
ResultRow(1, "Alice", Decimal("1250.50"))
 ↓
READING
 ↓
fetch
 ↓
ResultRow(2, "Bob", Decimal("820.25"))
 ↓
fetch
 ↓
ResultRow(3, "Carol", Decimal("100.00"))
 ↓
fetch
 ↓
EXHAUSTED
 ↓
close
 ↓
CLOSED
```

---

# 252. Ejemplo de early termination

```text
Database result:
A B C D E F
```

Consumer:

```text
fetch → A
fetch → B
fetch → C
STOP
```

Resultado observado:

```text
A B C
```

Termination:

```text
CONSUMER_STOP
↓
close/cancel
↓
release resources
```

Nunca:

```text
STOP
↓
pretend EXHAUSTED
```

---

# 253. Ejemplo de failure

```text
row 1
row 2
row 3
network failure
```

Estado:

```text
rowsDelivered = 3
state = FAILED
connectionImpact = CONNECTION_NOT_REUSABLE
```

Luego:

```text
cleanup
↓
CLOSED
```

La aplicación recibe una excepción normalizada.

---

# 254. Ejemplo de cursor + pool

```text
ConnectionPool
     ↓ lease
Connection #17
     ↓
Statement
     ↓
Cursor
```

Mientras cursor esté abierto:

```text
Connection #17
=
PINNED
```

Después:

```text
cursor close
↓
statement release
↓
connection reset if required
↓
Connection #17
→ pool
```

---

# 255. Anti-patterns

## Anti-pattern 1 — fetchAll como cursor

```php
final class Cursor
{
    public function __construct(PDOStatement $statement)
    {
        $this->rows = $statement->fetchAll();
    }
}
```

Esto no es un cursor real.

---

## Anti-pattern 2 — reexecute on rewind

```php
public function rewind(): void
{
    $this->query->executeAgain();
}
```

---

## Anti-pattern 3 — hidden buffering

```php
public function previous(): ResultRow
{
    $this->loadEverythingIntoMemory();
}
```

---

## Anti-pattern 4 — raw PDO exposure

```php
public function statement(): PDOStatement;
```

como API portable.

---

## Anti-pattern 5 — associative-only rows

```php
$row = $statement->fetch(PDO::FETCH_ASSOC);
```

como representación universal.

---

## Anti-pattern 6 — global cursor

```php
CursorManager::$current = $cursor;
```

---

## Anti-pattern 7 — release connection early

```text
execute
↓
create cursor
↓
return connection to pool
↓
continue fetching
```

---

## Anti-pattern 8 — rowCount for SELECT

```php
$total = $statement->rowCount();
```

como garantía portable.

---

## Anti-pattern 9 — implicit count

```php
count($cursor);
```

consumiendo todo el result set sin indicarlo.

---

## Anti-pattern 10 — cursor as ORM collection

```php
$cursor->nextUser();
```

---

## Anti-pattern 11 — hidden authorization

```php
if (!$auth->canSee($row)) {
    continue;
}
```

dentro del cursor.

---

## Anti-pattern 12 — log every row

```php
$logger->debug('row', $row->toArray());
```

---

## Anti-pattern 13 — concurrent consumption

```text
Consumer A ─┐
            ├─ same cursor
Consumer B ─┘
```

sin coordinación explícita.

---

## Anti-pattern 14 — GC-only cleanup

```text
"PHP will eventually destroy it"
```

---

## Anti-pattern 15 — vendor branching everywhere

```php
if ($driver === 'mysql') { ... }
elseif ($driver === 'pgsql') { ... }
elseif ($driver === 'sqlite') { ... }
```

---

# 256. Modelo arquitectónico final

```text
                RESULT SYSTEM

                 RowResult
                    │
                    ▼
           ┌───────────────────┐
           │   ResultCursor    │
           └─────────┬─────────┘
                     │
          ┌──────────┼───────────┐
          │          │           │
          ▼          ▼           ▼
      Lifecycle   Capabilities  Budget
          │          │           │
          └──────────┼───────────┘
                     │
                     ▼
              CursorSession
                     │
         ┌───────────┼────────────┐
         │           │            │
         ▼           ▼            ▼
   Cancellation   Telemetry   Resource Model
         │                         │
         └───────────┬─────────────┘
                     │
                     ▼
             DriverResultCursor
                     │
                     ▼
             Native Driver Result
                     │
                     ▼
                  Database
```

---

# 257. Relación con Resource Lifecycle

```text
ResultCursor
    │
    ├── owns/borrows → DriverResultHandle
    │
    ├── depends on  → PreparedStatementLease
    │
    ├── depends on  → ConnectionLease
    │
    └── optionally  → TransactionScope
```

---

# 258. Relación con Streaming

El próximo nivel será:

```text
ResultCursor
    ↓
StreamingResult
    ↓
Controlled Consumption
    ↓
Backpressure
    ↓
Application / Hydration Pipeline
```

---

# 259. Fórmula arquitectónica

```text
Result Cursor System
=
Incremental Row Fetching
+
Cursor State Machine
+
Capability Modeling
+
Position Tracking
+
Typed Row Conversion
+
Resource Ownership
+
Dependency Lifetimes
+
Fetch Strategy
+
Budget Enforcement
+
Cancellation
+
Failure Classification
+
Telemetry
+
Security
+
Persistent Runtime Isolation
```

---

# 260. Fórmula de corrección

```text
CorrectCursor
=
Ordered
∧
DuplicatePreserving
∧
SinglePassCorrect
∧
TypeSafe
∧
ResourceSafe
∧
CapabilityHonest
∧
CancellationAware
∧
BudgetBounded
∧
SecurityPreserving
```

---

# 261. Fórmula de resource safety

```text
CursorResourceSafety
=
ExplicitOwnership
+
ExplicitDependencies
+
OrderedCleanup
+
ConnectionImpactClassification
+
ScopeTerminationCleanup
```

---

# 262. Fórmula de portabilidad

```text
PortableCursor
=
Common Forward-Only Semantics
+
Driver Cursor Contract
+
Capability Discovery
+
No Hidden Emulation
```

---

# 263. Principio final

> **A cursor is a controlled view over an already-executed result, not another query execution mechanism.**

Por ello:

```text
Execute once
    ↓
Result
    ↓
Cursor
    ↓
Consume progressively
```

y nunca:

```text
Cursor operation
    ↓
silently execute query again
```

El diseño de VoltStack deberá favorecer:

```text
Forward-only by default
Capability-driven enhancements
Explicit resource ownership
Bounded memory
Deterministic consumption
```

manteniendo completamente separadas las responsabilidades de:

```text
Query Execution
Result Cursor
Streaming
Hydration
ORM
Pagination
```

---

# 264. Siguiente documento

```text
83_DATABASE_STREAMING_RESULT_SYSTEM.md
```

El siguiente documento deberá construir sobre `ResultCursor` el sistema formal de streaming de grandes resultados:

```text
Streaming Result
├── Streaming Architecture
├── Stream Contract
├── Pull-based Streaming
├── Backpressure
├── Streaming State Machine
├── Stream Consumer
├── Stream Producer
├── Cursor-backed Streams
├── Batch Streaming
├── Async Streaming
├── Flow Control
├── Buffer Management
├── Memory Bounds
├── Cancellation
├── Deadlines
├── Early Termination
├── Resource Ownership
├── Connection Pinning
├── Transaction Interaction
├── Streaming Conversion
├── Streaming Hydration Boundary
├── Failure Semantics
├── Partial Consumption
├── Telemetry
├── Security
├── Persistent Runtime Isolation
├── FrankenPHP
├── RoadRunner
├── OpenSwoole
└── Extension Model
```

manteniendo el invariante:

```text
StreamingResult
≠
ResultCursor
≠
BufferedResult
≠
LazyCollection
≠
ORM Collection
```

y estableciendo la base que posteriormente permitirá procesar datasets masivos sin que el consumo de memoria crezca proporcionalmente al número total de filas.