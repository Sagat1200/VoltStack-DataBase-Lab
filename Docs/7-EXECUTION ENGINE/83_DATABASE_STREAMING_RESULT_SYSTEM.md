# 83_DATABASE_STREAMING_RESULT_SYSTEM.md

# VoltStack Quantum Database
## Streaming Result System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 83 — Streaming Result System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Streaming Result System` define la arquitectura mediante la cual VoltStack puede entregar resultados de base de datos progresivamente al consumidor sin materializar el conjunto completo en memoria.

El sistema se construye sobre:

```text
DatabaseResult
    ↓
ResultCursor
    ↓
StreamingResult
    ↓
Consumer
```

Su responsabilidad fundamental es convertir un cursor incremental en un contrato de streaming controlado que incorpore:

- backpressure;
- ownership;
- lifecycle;
- cancellation;
- deadlines;
- límites de memoria;
- early termination;
- propagación de fallos;
- resource cleanup;
- integración con transactions;
- persistent-runtime safety;
- telemetry.

Principio central:

```text
StreamingResult
=
Progressive Result Delivery
+
Flow Control
+
Resource Lifecycle
```

---

# 2. Problema

Una consulta puede devolver:

```text
10 rows
10,000 rows
10,000,000 rows
```

Un sistema basado exclusivamente en:

```php
$rows = $statement->fetchAll();
```

produce una relación aproximada:

```text
Memory ≈ O(total result size)
```

Esto resulta inaceptable para:

- exportaciones;
- ETL;
- procesamiento batch;
- analytics;
- migraciones;
- sincronizaciones;
- reportes;
- grandes datasets;
- jobs de larga duración.

VoltStack necesita soportar:

```text
Database
   ↓
small bounded window
   ↓
application
```

en lugar de:

```text
Database
   ↓
entire result
   ↓
memory
   ↓
application
```

---

# 3. Principio fundamental

```text
StreamingResult
≠
ResultCursor
≠
BufferedResult
≠
Iterator
≠
Generator
≠
LazyCollection
≠
ORM Collection
≠
Pagination
```

Cada abstracción resuelve un problema diferente.

---

# 4. StreamingResult vs ResultCursor

`ResultCursor` representa:

```text
low-level incremental traversal
```

`StreamingResult` representa:

```text
managed progressive consumption
```

Por tanto:

```text
ResultCursor
=
mechanism
```

mientras:

```text
StreamingResult
=
consumption protocol
+
lifecycle
+
flow control
```

---

# 5. StreamingResult vs BufferedResult

```text
BufferedResult
```

materializa el conjunto de resultados.

```text
StreamingResult
```

mantiene una ventana acotada.

Idealmente:

```text
Memory(StreamingResult)
≈
O(window size)
```

y no:

```text
O(total rows)
```

---

# 6. StreamingResult vs Generator

PHP:

```php
function rows(): \Generator
{
    while ($row = fetch()) {
        yield $row;
    }
}
```

puede ofrecer lazy iteration, pero no modela suficientemente:

- ownership;
- connection pinning;
- transaction dependency;
- cancellation;
- timeout;
- failure after partial consumption;
- stream status;
- backpressure policy;
- resource budgets;
- close semantics;
- driver capabilities;
- telemetry.

Por ello:

```text
Generator
⊂
possible StreamingResult adapter
```

pero no constituye la arquitectura.

---

# 7. StreamingResult vs LazyCollection

Una `LazyCollection` es una abstracción de colecciones.

`StreamingResult` es una abstracción de recursos runtime.

Puede existir:

```text
StreamingResult
    ↓ adapter
LazyCollection
```

pero no:

```text
LazyCollection
=
database streaming lifecycle
```

---

# 8. StreamingResult vs Pagination

Streaming responde:

```text
¿Cómo consumir progresivamente este resultado ya ejecutado?
```

Pagination responde:

```text
¿Cómo dividir lógicamente una consulta en páginas?
```

Por tanto:

```text
Streaming
≠
Pagination
```

---

# 9. Posición arquitectónica

```text
Query Executor
      ↓
Statement Executor
      ↓
Database Result
      ↓
ResultCursor
      ↓
┌───────────────────────┐
│ StreamingResultSystem │
└───────────┬───────────┘
            ↓
     StreamingResult
            ↓
      Result Consumer
```

El ORM podrá consumir posteriormente esta capa:

```text
StreamingResult
      ↓
Hydration
      ↓
Entity Stream
```

pero:

```text
StreamingResult
```

no conoce entidades.

---

# 10. Objetivos

El sistema deberá soportar:

1. consumo progresivo;
2. memoria acotada;
3. pull-based streaming;
4. backpressure;
5. early termination;
6. cancellation;
7. deadlines;
8. cursor-backed streams;
9. explicit ownership;
10. connection pinning;
11. transaction affinity;
12. bounded buffering;
13. partial-consumption semantics;
14. streaming failures;
15. deterministic cleanup;
16. telemetry;
17. security;
18. persistent workers;
19. futuras capacidades async.

---

# 11. No objetivos

El Streaming Result System no deberá:

```text
generate SQL
change LIMIT/OFFSET
paginate queries
execute hidden queries
retry queries automatically
hydrate ORM entities
track UnitOfWork
perform authorization
sort result sets
deduplicate rows
cache complete results
```

---

# 12. Contrato principal

Se propone:

```php
namespace VoltStack\Quantum\Database\Execution\Result\Streaming\Contract;

interface StreamingResult
{
    public function id(): StreamId;

    public function state(): StreamState;

    public function metadata(): ResultMetadata;

    public function capabilities(): StreamCapabilitySet;

    public function next(): StreamReadResult;

    public function close(): void;
}
```

---

# 13. StreamReadResult

No se recomienda utilizar:

```php
public function next(): ?ResultRow;
```

como contrato interno principal.

Se propone:

```php
interface StreamReadResult
{
    public function status(): StreamReadStatus;
}
```

---

# 14. StreamReadStatus

```php
enum StreamReadStatus: string
{
    case ITEM = 'item';
    case END = 'end';
}
```

---

# 15. StreamItem

```php
final readonly class StreamItem implements StreamReadResult
{
    public function __construct(
        public ResultRow $row,
        public StreamSequence $sequence,
    ) {}

    public function status(): StreamReadStatus
    {
        return StreamReadStatus::ITEM;
    }
}
```

---

# 16. StreamEnd

```php
final readonly class StreamEnd implements StreamReadResult
{
    public function status(): StreamReadStatus
    {
        return StreamReadStatus::END;
    }
}
```

---

# 17. Stream lifecycle

Estados propuestos:

```php
enum StreamState: string
{
    case CREATED = 'created';
    case OPEN = 'open';
    case ACTIVE = 'active';
    case DRAINING = 'draining';
    case EXHAUSTED = 'exhausted';
    case CANCELLED = 'cancelled';
    case TIMED_OUT = 'timed_out';
    case FAILED = 'failed';
    case CLOSING = 'closing';
    case CLOSED = 'closed';
}
```

---

# 18. State machine

```text
CREATED
   │
   ▼
 OPEN
   │
   ▼
ACTIVE ───────────────────────────┐
   │                             │
   │                             ▼
   │                         CANCELLED
   │                             │
   ├──────────────► TIMED_OUT    │
   │                  │          │
   ├──────────────► FAILED       │
   │                  │          │
   ▼                  │          │
DRAINING              │          │
   │                  │          │
   ▼                  │          │
EXHAUSTED             │          │
   │                  │          │
   └──────────┬───────┴──────────┘
              ▼
           CLOSING
              │
              ▼
            CLOSED
```

---

# 19. CREATED

El stream ha sido construido pero todavía no ha iniciado consumo.

---

# 20. OPEN

Los recursos necesarios están disponibles y puede comenzar lectura.

---

# 21. ACTIVE

El consumidor está solicitando elementos.

---

# 22. DRAINING

El stream sigue ligado a recursos y está terminando su flujo.

Este estado es especialmente importante en la arquitectura de VoltStack.

---

# 23. EXHAUSTED

El resultado completo fue consumido correctamente.

---

# 24. CANCELLED

El consumidor o Execution Engine solicitó interrupción.

---

# 25. TIMED_OUT

El deadline fue excedido.

---

# 26. FAILED

El stream no puede continuar debido a un error.

---

# 27. CLOSING

Se está realizando cleanup.

---

# 28. CLOSED

No quedan recursos owned por el stream.

---

# 29. Streaming y ExecutionInstance

El documento 76 estableció:

```text
RUNNING
   ↓
DRAINING
   ↓
COMPLETED
   ↓
CLEANING
   ↓
CLOSED
```

para ejecución streaming.

Por tanto:

```text
QueryExecutor.execute()
```

puede retornar un stream antes de que la ejecución esté completamente terminada.

---

# 30. Punto crítico

Retornar:

```text
StreamingResult
```

no significa necesariamente:

```text
ExecutionInstance = COMPLETED
```

Puede significar:

```text
ExecutionInstance = DRAINING
```

---

# 31. Por qué

Porque siguen vivos:

```text
cursor
statement
connection
possibly transaction
```

mientras el consumidor continúa leyendo.

---

# 32. ExecutionResultHandle

La arquitectura deberá favorecer:

```php
interface ExecutionResultHandle
{
    public function result(): DatabaseResult;

    public function executionState(): ExecutionState;
}
```

con especialización:

```text
ImmediateExecutionResultHandle
StreamingExecutionResultHandle
```

---

# 33. StreamingExecutionResultHandle

Conceptualmente:

```text
StreamingExecutionResultHandle
├── StreamingResult
├── ExecutionInstance reference
├── Resource ownership
└── Finalization callback
```

---

# 34. Success semantics

Para resultados streaming deben distinguirse:

```text
STREAM_OPENED_SUCCESSFULLY
```

de:

```text
STREAM_FULLY_CONSUMED_SUCCESSFULLY
```

---

# 35. No false success

No declarar:

```text
execution completed successfully
```

cuando sólo se abrió el cursor.

---

# 36. Completion semantics

La ejecución completa ocurre cuando:

```text
stream exhausted
∨
stream explicitly terminated
∨
stream cancelled
∨
stream failed
```

y el lifecycle correspondiente ha finalizado.

---

# 37. Pull-based streaming

VoltStack V1 utilizará principalmente:

```text
PULL
```

El consumidor solicita:

```text
next
```

y el productor responde.

---

# 38. Pull model

```text
Consumer
   │
   │ next()
   ▼
StreamingResult
   │
   │ fetch()
   ▼
ResultCursor
   │
   ▼
Driver
```

---

# 39. Backpressure natural

En un modelo pull:

```text
Consumer demand
=
producer permission
```

El sistema no necesita producir filas más rápido que el consumidor salvo buffers explícitos.

---

# 40. Principio

```text
No Demand
⇒
No Unbounded Production
```

---

# 41. Backpressure

Backpressure representa el mecanismo mediante el cual el consumidor controla el ritmo de producción.

Formalmente:

```text
Produced(t)
≤
Demanded(t)
+
BufferCapacity
```

---

# 42. Buffer boundedness

Para buffer máximo `B`:

```text
BufferedItems ≤ B
```

en todo momento.

---

# 43. No unbounded queues

Nunca:

```text
Database
 ↓
Producer
 ↓
[unbounded queue]
 ↓
slow consumer
```

---

# 44. Default V1

El modelo más simple:

```text
BufferCapacity = 0 or 1
```

cuando el driver/cursor lo permita.

---

# 45. Driver buffering

Un driver puede realizar buffering interno.

VoltStack deberá distinguir:

```text
FrameworkBuffer
```

de:

```text
DriverBuffer
```

---

# 46. Unknown driver buffering

Si el driver no permite conocer/controlar exactamente el buffer:

```text
BufferKnowledge = PARTIAL
```

deberá ser representable.

---

# 47. Streaming guarantee levels

Se propone:

```php
enum StreamingGuarantee: string
{
    case TRUE_INCREMENTAL = 'true_incremental';
    case DRIVER_BUFFERED = 'driver_buffered';
    case PARTIALLY_BUFFERED = 'partially_buffered';
    case FULLY_BUFFERED_BY_DRIVER = 'fully_buffered_by_driver';
    case UNKNOWN = 'unknown';
}
```

---

# 48. Importancia

Una API no deberá afirmar:

```text
constant memory streaming
```

si el driver realmente materializa todo el resultado.

---

# 49. Capability honesty

VoltStack deberá distinguir:

```text
API streams incrementally
```

de:

```text
database driver physically streams incrementally
```

---

# 50. StreamCapability

```php
enum StreamCapability: string
{
    case PULL = 'pull';
    case PUSH = 'push';
    case ASYNC = 'async';
    case CANCELLABLE = 'cancellable';
    case BATCH_READ = 'batch_read';
    case FETCH_SIZE_HINT = 'fetch_size_hint';
    case DETACHABLE = 'detachable';
    case TRUE_INCREMENTAL = 'true_incremental';
    case BACKPRESSURE = 'backpressure';
    case REACTIVE_ADAPTER = 'reactive_adapter';
}
```

---

# 51. StreamCapabilitySet

```php
final readonly class StreamCapabilitySet
{
    /**
     * @param array<StreamCapability, StreamCapabilityDescriptor> $capabilities
     */
    public function __construct(
        private array $capabilities,
    ) {}
}
```

---

# 52. StreamingPolicy

```php
final readonly class StreamingPolicy
{
    public function __construct(
        public StreamingMode $mode,
        public int $bufferCapacity,
        public ?int $fetchSize,
        public StreamClosePolicy $closePolicy,
        public StreamBudgetPolicy $budget,
    ) {}
}
```

---

# 53. StreamingMode

```php
enum StreamingMode: string
{
    case PULL = 'pull';
    case BATCH_PULL = 'batch_pull';
    case ASYNC_PULL = 'async_pull';
}
```

V1:

```text
PULL
```

como baseline.

---

# 54. Batch streaming

Podrá existir:

```php
interface BatchStreamingResult extends StreamingResult
{
    public function nextBatch(
        int $maxItems
    ): StreamBatchReadResult;
}
```

---

# 55. Batch semantics

```text
nextBatch(100)
```

significa:

```text
deliver up to 100 available result rows
```

No:

```sql
LIMIT 100
```

---

# 56. Batch size ≠ query cardinality

Batch size afecta:

```text
consumption
memory
driver fetch strategy
```

no la semántica de la query.

---

# 57. Batch backpressure

Para demanda:

```text
Demand = 100
```

el stream puede producir:

```text
0 ≤ Produced ≤ 100
```

dependiendo de EOF/failure/cancellation.

---

# 58. Stream sequence

Cada elemento podrá tener:

```php
final readonly class StreamSequence
{
    public function __construct(
        public int $ordinal,
    ) {}
}
```

---

# 59. Sequence invariant

Para stream forward-only:

```text
sequence(row_i) = i
```

---

# 60. Sequence ≠ database identity

No confundir con:

```text
primary key
ROWID
CTID
cursor pagination key
```

---

# 61. Result ordering

Streaming debe preservar exactamente el orden entregado por `ResultCursor`.

---

# 62. No invented ordering

Si la query no tiene orden observable garantizado:

```text
StreamingResult
```

no inventará uno.

---

# 63. Duplicate preservation

Si el resultado contiene:

```text
A
A
B
A
```

el stream entregará:

```text
A
A
B
A
```

---

# 64. NULL preservation

SQL NULL permanecerá representado por el sistema de resultados.

---

# 65. Stream consumer

Contrato conceptual:

```php
interface StreamConsumer
{
    public function consume(
        ResultRow $row,
        StreamConsumerContext $context
    ): StreamConsumerDecision;
}
```

---

# 66. Consumer decision

```php
enum StreamConsumerDecision
{
    case CONTINUE;
    case STOP;
}
```

---

# 67. Managed consumption

```php
$stream->consume(
    function (ResultRow $row): StreamConsumerDecision {
        process($row);

        return StreamConsumerDecision::CONTINUE;
    }
);
```

---

# 68. Managed lifecycle

Internamente:

```text
try
    consume
finally
    close
```

---

# 69. Early termination

Si el consumer retorna:

```text
STOP
```

esto representa:

```text
CONSUMER_STOP
```

no:

```text
EXHAUSTED
```

---

# 70. StreamTerminationReason

```php
enum StreamTerminationReason: string
{
    case EXHAUSTED = 'exhausted';
    case CONSUMER_STOP = 'consumer_stop';
    case EXPLICIT_CLOSE = 'explicit_close';
    case CANCELLED = 'cancelled';
    case TIMED_OUT = 'timed_out';
    case BUDGET_EXCEEDED = 'budget_exceeded';
    case FAILURE = 'failure';
    case REQUEST_END = 'request_end';
    case WORKER_SHUTDOWN = 'worker_shutdown';
    case TRANSACTION_END = 'transaction_end';
    case CONNECTION_LOST = 'connection_lost';
}
```

---

# 71. Partial consumption

Sea:

```text
R = [r0, r1, ..., rn]
```

Si el consumidor termina después de `k` elementos:

```text
Delivered = prefix(R, k)
```

---

# 72. Partial ≠ complete

Un stream parcialmente consumido jamás deberá presentarse como:

```text
complete result
```

---

# 73. StreamCompletion

```php
final readonly class StreamCompletion
{
    public function __construct(
        public StreamTerminationReason $reason,
        public int $rowsDelivered,
        public bool $fullyConsumed,
        public StreamResourceImpact $resourceImpact,
    ) {}
}
```

---

# 74. fullyConsumed

Sólo:

```text
true
```

si:

```text
terminationReason = EXHAUSTED
```

y no ocurrió error/cancellation/timeout.

---

# 75. close()

```php
$stream->close();
```

antes de exhaustion significa:

```text
early termination
```

no completion total.

---

# 76. close() idempotence

Idealmente:

```text
close()
close()
close()
```

será seguro.

---

# 77. next() after close

Debe producir:

```text
StreamClosedException
```

---

# 78. next() after exhaustion

Podrá devolver:

```text
StreamEnd
```

repetidamente hasta `close()`.

---

# 79. Ownership

El stream puede poseer:

```text
ResultCursor
```

y transitivamente recursos asociados.

---

# 80. Ownership chain

Ejemplo:

```text
StreamingResult
     │ owns
     ▼
ResultCursor
     │ depends
     ▼
PreparedStatement
     │ depends
     ▼
ConnectionLease
```

---

# 81. Streaming ownership transfer

Cuando QueryExecutor retorna un streaming result:

```text
ExecutionInstance
      │
      │ transfer
      ▼
StreamingResult
```

puede transferirse ownership de recursos.

---

# 82. Transfer invariant

Después de transferir recurso `R`:

```text
Owner(R) = StreamingResult
```

y el QueryExecutor no deberá liberarlo prematuramente.

---

# 83. Conservation of resources

Manteniendo el invariante del documento 76:

```text
AcquiredResources
=
ReleasedResources
∪
TransferredResources
```

y:

```text
ReleasedResources
∩
TransferredResources
=
∅
```

---

# 84. Streaming finalization

Cuando el stream termina:

```text
StreamingResult
      ↓
close cursor
      ↓
close/release statement
      ↓
reset/release connection
      ↓
notify ExecutionInstance
      ↓
final cleanup
```

según ownership graph.

---

# 85. StreamingResult finalizer

Podrá existir:

```php
interface StreamingResultFinalizer
{
    public function finalize(
        StreamingSession $session,
        StreamTerminationReason $reason
    ): StreamCompletion;
}
```

---

# 86. Destructors

Un destructor puede actuar como:

```text
last-resort leak protection
```

pero nunca será el mecanismo principal de cleanup.

---

# 87. Leak detection

Si un stream framework-owned es destruido abierto:

```text
StreamLeakDetected
```

podrá registrarse.

---

# 88. Connection pinning

Un stream basado en cursor puede requerir:

```text
ConnectionLease = PINNED
```

durante todo su lifecycle.

---

# 89. Consecuencia

Un stream lento puede reducir capacidad del pool.

---

# 90. Resource governance

Por ello deberán existir límites como:

```text
max open streams
max stream lifetime
max pinned connections
max idle duration
max rows
max bytes
```

---

# 91. Pool starvation protection

El sistema deberá poder detectar:

```text
too many long-lived streams
```

antes de agotar el pool.

---

# 92. Stream resource reservation

Puede existir:

```php
interface StreamingResourceGovernor
{
    public function reserve(
        StreamingResourceRequest $request
    ): StreamingResourceReservation;
}
```

---

# 93. Reservation dimensions

```text
connection slot
memory
buffer capacity
stream slot
temporary storage
execution time
```

---

# 94. Transaction interaction

Algunos streams requerirán una transacción activa.

---

# 95. Transaction-bound stream

```text
Open(Stream)
⇒
Active(Transaction)
```

cuando el driver contract lo requiera.

---

# 96. Caller-owned transaction

Si la transaction pertenece al caller:

```text
StreamingResult
```

no deberá hacer:

```text
COMMIT
```

ni:

```text
ROLLBACK
```

automáticamente salvo protocolo explícito.

---

# 97. Engine-owned transaction

Si la transaction fue creada específicamente para el stream:

```text
Stream Finalization
```

puede coordinar su finalización mediante `TransactionExecutionCoordinator`.

---

# 98. Stream does not own transaction semantics

El stream no decide por sí mismo:

```text
commit
rollback
savepoint
```

---

# 99. Transaction affinity

Un stream transaction-bound deberá permanecer asociado al:

```text
same transaction
same connection
same session
```

según requirements.

---

# 100. Commit with open stream

Debe ser capability/policy-driven.

Posibles comportamientos:

```text
FORBIDDEN
AUTO_CLOSE_STREAM
STREAM_SURVIVES
DRIVER_DEFINED
```

---

# 101. Rollback with open stream

Puede invalidar inmediatamente:

```text
cursor
statement
stream
```

---

# 102. Stream invalidation

Contrato:

```php
interface InvalidatableStreamingResult
{
    public function invalidate(
        StreamInvalidationReason $reason
    ): void;
}
```

---

# 103. StreamInvalidationReason

```php
enum StreamInvalidationReason
{
    case TRANSACTION_COMMIT;
    case TRANSACTION_ROLLBACK;
    case CONNECTION_RESET;
    case CONNECTION_LOST;
    case STATEMENT_INVALIDATED;
    case EXECUTION_CANCELLED;
    case REQUEST_TERMINATED;
    case WORKER_SHUTDOWN;
}
```

---

# 104. Invalid stream

No continuará entregando filas.

---

# 105. Cancellation

Streaming cancellation es first-class.

```text
Consumer
   ↓
CancellationToken
   ↓
StreamingResult
   ↓
ResultCursor
   ↓
Driver
```

---

# 106. Cancellation contract

```php
interface StreamCancellationController
{
    public function cancel(
        StreamCancellationReason $reason
    ): StreamCancellationOutcome;
}
```

---

# 107. Cancellation reasons

```text
USER_REQUEST
EXECUTION_CANCELLED
REQUEST_DISCONNECTED
RESOURCE_PRESSURE
WORKER_SHUTDOWN
DEADLINE
APPLICATION_STOP
```

---

# 108. Cancellation ≠ failure

```text
CANCELLED
≠
FAILED
```

aunque el driver pueda producir una excepción técnica durante cancellation.

---

# 109. Cancellation ≠ timeout

```text
TIMED_OUT
```

representa expiration de deadline.

Puede utilizar internamente cancellation, pero la causa semántica sigue siendo timeout.

---

# 110. Cancellation unsupported

Si el driver no puede cancelar server-side:

VoltStack podrá:

1. dejar de entregar filas;
2. cerrar cursor;
3. cerrar statement;
4. invalidar connection si es necesario;
5. reportar capacidad real.

No deberá afirmar que el servidor canceló la query si no puede probarlo.

---

# 111. Deadlines

Un stream puede estar limitado por:

```text
Execution Deadline
Stream Lifetime Deadline
Fetch Deadline
Idle Deadline
```

---

# 112. Deadline inheritance

Por defecto:

```text
StreamDeadline
≤
ExecutionDeadline
```

si la ejecución original tiene deadline.

---

# 113. No deadline extension

El stream no deberá extender silenciosamente un deadline de ejecución.

---

# 114. Monotonic time

Duraciones deberán medirse con:

```text
MonotonicClock
```

cuando esté disponible.

---

# 115. Idle timeout

Especialmente importante:

```text
consumer receives row
↓
does nothing for 20 minutes
↓
connection remains pinned
```

---

# 116. Idle tracking

```text
IdleDuration
=
now
-
lastConsumerActivity
```

---

# 117. StreamBudget

```php
final class StreamBudget
{
    public function consumeRow(): void;

    public function consumeBytes(int $bytes): void;

    public function consumeBufferBytes(int $bytes): void;

    public function consumeFetch(): void;

    public function checkDeadline(): void;
}
```

---

# 118. Budget dimensions

```text
max rows delivered
max bytes delivered
max bytes buffered
max row size
max LOB size
max fetches
max duration
max idle duration
max conversion cost
max temporary storage
```

---

# 119. Budget ≠ SQL LIMIT

Si:

```text
maxRowsDelivered = 100000
```

eso es una política operacional.

No cambia el SQL a:

```sql
LIMIT 100000
```

---

# 120. Budget exhaustion

Si se alcanza:

```text
StreamBudgetExceeded
```

el stream termina explícitamente.

---

# 121. No silent truncation

Nunca:

```text
budget exceeded
↓
return StreamEnd
```

como si el resultado hubiera sido completamente consumido.

---

# 122. Correct behavior

```text
budget exceeded
↓
termination = BUDGET_EXCEEDED
↓
cleanup
↓
exception/outcome
```

---

# 123. Buffer manager

Puede existir:

```php
interface StreamBufferManager
{
    public function offer(ResultRow $row): void;

    public function poll(): ?ResultRow;

    public function size(): int;

    public function bytes(): int;
}
```

---

# 124. Bounded buffer

Siempre:

```text
BufferBytes
≤
ConfiguredMaximum
```

---

# 125. Buffer overflow

No deberá provocar crecimiento automático sin límite.

---

# 126. Buffer policy

```php
enum StreamBufferOverflowPolicy
{
    case BLOCK_PRODUCER;
    case STOP_PRODUCTION;
    case FAIL;
}
```

Para pull-based V1, normalmente:

```text
BLOCK_PRODUCER
```

equivale a no solicitar más datos.

---

# 127. No DROP policy for database rows

No se deberá usar:

```text
DROP_OLDEST
DROP_NEWEST
```

para filas de resultados normales.

Eso alteraría la semántica.

---

# 128. Memory governance

El sistema deberá conocer al menos la memoria que controla directamente.

---

# 129. Memory formula

Aproximadamente:

```text
M_stream
=
M_metadata
+
M_cursor
+
M_framework_buffer
+
M_current_rows
+
M_conversion_state
+
M_driver_unknown
```

---

# 130. Known vs unknown memory

Debe distinguirse:

```text
FrameworkControlledMemory
```

de:

```text
DriverControlledMemory
```

---

# 131. Streaming claim

VoltStack sólo deberá prometer memoria acotada sobre los componentes que realmente controla.

---

# 132. Streaming conversion

Cada fila puede atravesar:

```text
Native Row
    ↓
Result Conversion
    ↓
ResultRow
    ↓
StreamingResult
```

---

# 133. Conversion plan

Debe reutilizar:

```text
CompiledResultConversionPlan
```

del Result System.

---

# 134. No conversion re-planning

No:

```text
row 1 → discover conversion
row 2 → rediscover conversion
...
```

---

# 135. Conversion failure

Si la fila `k` no puede convertirse:

```text
rows 0 ... k-1
```

ya pudieron haberse entregado.

---

# 136. Streaming failure semantics

Por tanto:

```text
Streaming Failure
```

puede ocurrir después de efectos observables.

---

# 137. Important invariant

```text
Failure after partial delivery
≠
Atomic failure
```

---

# 138. No rollback of delivered rows

VoltStack no puede:

```text
un-deliver
```

filas que el consumidor ya procesó.

---

# 139. Application responsibility

El consumidor que necesite atomicidad lógica deberá diseñar su procesamiento en consecuencia.

---

# 140. Streaming and retry

Retry automático después de haber entregado filas es peligroso.

---

# 141. Example

Original:

```text
A B C D
```

Consumer recibió:

```text
A B
```

falla conexión.

Retry ingenuo:

```text
A B C D
```

produciría:

```text
A B A B C D
```

---

# 142. Default rule

```text
RowsDelivered > 0
⇒
AutomaticWholeStreamRetry = FORBIDDEN
```

salvo protocolo explícito de resumability/idempotency.

---

# 143. Retry belongs elsewhere

Las políticas completas se definirán en:

```text
86_DATABASE_EXECUTION_RETRY_SYSTEM.md
```

---

# 144. Streaming Result sólo reporta

Debe proporcionar información como:

```text
rowsDelivered
lastSequence
terminationReason
retrySafety
```

---

# 145. RetrySafety

```php
enum StreamRetrySafety
{
    case SAFE_BEFORE_FIRST_DELIVERY;
    case UNSAFE_AFTER_DELIVERY;
    case RESUMABLE_BY_PLAN;
    case UNKNOWN;
}
```

---

# 146. Resumable stream

Una futura capability puede permitir:

```text
resume token
```

pero no deberá confundirse con reejecución automática de SQL.

---

# 147. Resume requires plan support

```text
Resumability
```

debe estar diseñada por Query/Planner/Execution layers.

StreamingResult no inventará una clave de reanudación.

---

# 148. Backpressure across pipeline

Pipeline:

```text
Database
   ↓
Driver
   ↓
Cursor
   ↓
StreamingResult
   ↓
Hydrator
   ↓
Application
```

La demanda debe propagarse:

```text
Application Demand
        ↓
Hydration Demand
        ↓
Stream Demand
        ↓
Cursor Fetch
```

---

# 149. No eager hydration

Si se solicitan diez elementos:

```text
StreamingResult
```

no deberá hidratar 100,000 entidades anticipadamente.

---

# 150. ORM boundary

Una capa futura podrá crear:

```text
StreamingEntityResult<T>
```

sobre:

```text
StreamingResult<ResultRow>
```

---

# 151. But

El Database Streaming Result System seguirá siendo:

```text
ORM-independent
```

---

# 152. IdentityMap consideration

El ORM deberá decidir posteriormente si una entity stream:

```text
retains
detaches
evicts
```

entidades de IdentityMap.

Eso no pertenece a este documento.

---

# 153. UnitOfWork consideration

Igualmente, streaming de entidades modificables tiene implicaciones para UoW que serán resueltas en los documentos ORM/Persistence.

---

# 154. Stream transformations

Puede ser tentador ofrecer:

```php
$stream
    ->filter(...)
    ->map(...)
    ->take(...);
```

---

# 155. Architectural decision

El core de `StreamingResult` no deberá convertirse en una biblioteca general de streams funcionales.

---

# 156. Why

Eso mezclaría:

```text
database resource lifecycle
```

con:

```text
application data transformation
```

---

# 157. Adapters

Podrán existir adapters en:

```text
Support
Collections
Reactive
```

que proporcionen:

```text
map
filter
take
reduce
```

sin contaminar el núcleo.

---

# 158. take() danger

Una transformación:

```text
take(10)
```

debe cerrar correctamente el stream después del décimo elemento.

---

# 159. Application filtering

Un adapter puede filtrar filas para consumo de aplicación.

Pero deberá quedar claro:

```text
application stream transformation
≠
database result semantics
```

---

# 160. Security

Todos los elementos del stream se consideran potencialmente sensibles.

---

# 161. Telemetry rule

No registrar:

```text
row values
LOB content
credentials
tokens
PII
```

por defecto.

---

# 162. Streaming diagnostics

Puede incluir:

```text
StreamId
ExecutionId
StatementExecutionId
state
rows delivered
bytes delivered
buffer usage
duration
idle duration
termination reason
connection pinned
transaction bound
driver guarantee
streaming guarantee
```

---

# 163. StreamId

```php
final readonly class StreamId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 164. StreamId ≠ query identity

Es runtime identity.

---

# 165. Telemetry events

```text
StreamCreated
StreamOpened
StreamActivated
StreamItemDelivered
StreamBatchDelivered
StreamBackpressureApplied
StreamIdle
StreamCancelled
StreamTimedOut
StreamBudgetExceeded
StreamFailed
StreamExhausted
StreamClosing
StreamClosed
StreamLeakDetected
```

---

# 166. Per-row telemetry

Deberá estar desactivada o altamente optimizada por defecto.

---

# 167. Aggregated metrics

Preferir:

```text
rows delivered
bytes delivered
fetch count
batch count
duration
idle time
buffer high-water mark
```

---

# 168. Metrics

Ejemplos:

```text
db.stream.open
db.stream.active
db.stream.duration
db.stream.rows
db.stream.bytes
db.stream.buffer.bytes
db.stream.buffer.high_watermark
db.stream.fetches
db.stream.failures
db.stream.cancellations
db.stream.timeouts
db.stream.early_termination
db.stream.leaks
db.stream.pinned_connections
```

---

# 169. Cardinality

No usar:

```text
StreamId
customer ID
query raw text
tenant ID
```

como metric label de alta cardinalidad.

---

# 170. Tenant isolation

Tenant context ya debió afectar la query antes de ejecución.

El stream no decidirá:

```text
which tenant can see row
```

---

# 171. No global tenant

Nunca:

```php
TenantContext::current()
```

desde un stream global compartido.

---

# 172. Tenant metadata

Sólo metadata segura y operation-scoped podrá acompañar al stream cuando sea necesaria para diagnostics/security.

---

# 173. Persistent runtime

FrankenPHP puede reutilizar un worker:

```text
Worker
├── Request A
│   └── Stream A
└── Request B
    └── Stream B
```

---

# 174. Isolation invariant

```text
MutableState(Stream A)
∩
MutableState(Stream B)
=
∅
```

---

# 175. Request termination

Si Request A termina con Stream A abierto:

```text
Request Scope Cleanup
       ↓
Cancel/Close Stream A
       ↓
Close Cursor
       ↓
Release Statement
       ↓
Reset/Release Connection
```

---

# 176. Detached streams across requests

Por defecto:

```text
FORBIDDEN
```

---

# 177. Why

Un stream puede depender de:

```text
request cancellation
transaction
tenant routing
connection lease
execution context
resource budget
```

---

# 178. Background ownership transfer

Mover un stream a otro job/worker requeriría un protocolo explícito.

No se permitirá transferir un live DB cursor arbitrariamente.

---

# 179. FrankenPHP

La implementación deberá asumir que los servicios compartidos pueden sobrevivir múltiples requests.

Por tanto:

```text
StreamingResult service
```

puede ser singleton sólo si es:

```text
stateless
```

---

# 180. RoadRunner

Misma regla:

```text
worker reuse
⇒
operation-local streaming sessions
```

---

# 181. OpenSwoole

Con coroutines:

```text
Coroutine A
    ↓
StreamingSession A

Coroutine B
    ↓
StreamingSession B
```

---

# 182. No coroutine globals

No:

```php
static $currentStream;
```

---

# 183. Concurrent consumption

Por defecto:

```text
one stream
→
one active consumer
```

---

# 184. Concurrent next()

Dos llamadas simultáneas:

```text
next A
next B
```

pueden romper ordering.

---

# 185. V1 rule

Fail-fast con:

```text
ConcurrentStreamConsumptionException
```

---

# 186. Future async consumption

Un stream async podrá soportar:

```text
await next()
```

sin permitir necesariamente múltiples consumers.

---

# 187. Async ≠ parallel consumption

```text
ASYNC
≠
MULTI_CONSUMER
```

---

# 188. Reactive adapter

Arquitectura futura:

```text
StreamingResult
      ↓
ReactiveStreamAdapter
      ↓
Publisher
      ↓
Subscriber
```

---

# 189. Reactive Streams principle

El adapter deberá preservar:

```text
demand
cancellation
ordering
errors
completion
```

---

# 190. Reactive demand

Conceptualmente:

```text
subscriber.request(n)
```

podrá mapearse a:

```text
stream demand += n
```

---

# 191. No reactive dependency in core

El Database package no deberá depender obligatoriamente de una librería Reactive Streams.

---

# 192. Stream producer abstraction

Puede existir:

```php
interface StreamProducer
{
    public function produce(
        StreamDemand $demand,
        StreamingSession $session
    ): StreamProductionResult;
}
```

---

# 193. CursorStreamProducer

Implementación principal:

```text
CursorStreamProducer
    ↓
ResultCursor
```

---

# 194. StreamDemand

```php
final readonly class StreamDemand
{
    public function __construct(
        public int $items,
    ) {}
}
```

---

# 195. Demand invariant

```text
items > 0
```

---

# 196. Production invariant

```text
ProducedItems
≤
DemandedItems
```

excepto buffer preexistente ya autorizado.

---

# 197. Demand accounting

Podrá existir:

```text
Requested
Produced
Delivered
Buffered
```

como contadores distintos.

---

# 198. Important equation

```text
Produced
=
Delivered
+
Buffered
```

para items válidos no descartados.

---

# 199. No dropped rows

Para database streaming normal:

```text
Produced
-
Delivered
-
Buffered
=
0
```

---

# 200. Stream session

```php
final class StreamingSession
{
    private StreamState $state;

    private int $rowsDelivered = 0;

    private int $bytesDelivered = 0;

    private int $requested = 0;

    private int $produced = 0;

    // operation-local mutable state
}
```

---

# 201. Session contains

Puede contener:

```text
StreamId
state
cursor lease
resource ownership
buffer
budget
deadline
cancellation token
telemetry context
termination state
sequence
execution reference
```

---

# 202. Session does not contain

No deberá contener:

```text
service container
HTTP kernel
global tenant
global EntityManager
global transaction singleton
```

---

# 203. Stream context

```php
final readonly class StreamingContext
{
    public function __construct(
        public StreamingPolicy $policy,
        public StreamCapabilitySet $capabilities,
        public StreamDeadline $deadline,
        public StreamCancellationToken $cancellation,
        public StreamSecurityContext $security,
        public StreamInstrumentationContext $instrumentation,
    ) {}
}
```

---

# 204. Context immutability

`StreamingContext` será immutable.

---

# 205. Session mutability

`StreamingSession` será mutable pero estrictamente operation-scoped.

---

# 206. Resource ownership graph

Ejemplo:

```text
StreamingSession
       │
       ▼
StreamingResult
       │
       ▼
ResultCursor
       │
       ▼
DriverResult
       │
       ▼
PreparedStatement
       │
       ▼
ConnectionLease
```

---

# 207. Cleanup order

Normalmente:

```text
StreamingResult
↓
ResultCursor
↓
DriverResult
↓
PreparedStatement
↓
ConnectionLease
```

respetando contratos reales.

---

# 208. Cleanup coordinator

El stream no necesita implementar todos los releases directamente.

Puede delegar a:

```text
CleanupCoordinator
```

del Execution Engine.

---

# 209. Cleanup request

```php
$cleanupCoordinator->closeStreamingScope(
    $session->resourceScope(),
    $termination
);
```

---

# 210. Primary failure preservation

Si:

```text
fetch failed
+
cursor close failed
+
connection reset failed
```

el error primario seguirá siendo:

```text
fetch failed
```

con los demás errores adjuntos al cleanup report.

---

# 211. Failure taxonomy

```text
StreamingResultException
├── StreamStateException
├── StreamClosedException
├── StreamCapabilityException
├── StreamReadException
├── StreamCursorException
├── StreamConversionException
├── StreamBufferException
├── StreamBudgetException
├── StreamBudgetExceededException
├── StreamCancellationException
├── StreamTimeoutException
├── StreamResourceException
├── StreamOwnershipException
├── StreamTransactionException
├── StreamInvalidationException
├── StreamBackpressureException
├── StreamConsumerException
├── StreamDriverException
├── StreamSecurityException
├── StreamExtensionException
├── ConcurrentStreamConsumptionException
└── StreamInvariantException
```

---

# 212. Consumer exception

Si el callback del consumidor falla:

```text
ConsumerException
```

debe provocar:

```text
stop production
↓
cleanup
↓
preserve original consumer exception
```

---

# 213. Consumer failure ≠ database failure

Diagnostics deberá distinguir ambos.

---

# 214. Failure origin

```php
enum StreamFailureOrigin
{
    case DATABASE;
    case DRIVER;
    case CURSOR;
    case CONVERSION;
    case CONSUMER;
    case RESOURCE;
    case CANCELLATION;
    case TIMEOUT;
    case EXTENSION;
    case INVARIANT;
}
```

---

# 215. Failure after delivery

El error deberá poder informar:

```text
rowsDelivered = N
```

sin incluir valores de esas filas.

---

# 216. Execution outcome

Para streaming, el Execution Engine necesita resultado final diferido.

---

# 217. Completion callback

Puede existir:

```php
interface StreamCompletionListener
{
    public function completed(
        StreamCompletion $completion
    ): void;
}
```

---

# 218. QueryExecutor integration

Flujo:

```text
QueryExecutor
    ↓
StatementExecutor
    ↓
ResultCursor
    ↓
StreamingResultFactory
    ↓
transfer resource ownership
    ↓
StreamingResult returned
    ↓
ExecutionInstance = DRAINING
```

---

# 219. Consumer completes stream

```text
StreamingResult
    ↓
EXHAUSTED
    ↓
cleanup
    ↓
StreamCompletion
    ↓
ExecutionInstance
    ↓
COMPLETED
    ↓
CLEANING
    ↓
CLOSED
```

---

# 220. Consumer stops early

```text
StreamingResult
    ↓
CONSUMER_STOP
    ↓
cancel/close cursor
    ↓
cleanup
    ↓
ExecutionInstance finalization
```

---

# 221. Stream failure

```text
StreamingResult
    ↓
FAILED
    ↓
cleanup
    ↓
ExecutionInstance = FAILED
```

---

# 222. Stream timeout

```text
StreamingResult
    ↓
TIMED_OUT
    ↓
cancellation
    ↓
cleanup
    ↓
ExecutionInstance = TIMED_OUT
```

---

# 223. Execution result status

No deberá reducir todos estos casos a:

```text
boolean success
```

---

# 224. Result contract

El ExecutionPlan ya declaró:

```text
ROW_STREAM
```

por tanto QueryExecutor no decide dinámicamente convertir un resultado buffered en streaming.

---

# 225. Critical invariant

```text
StreamingResult
exists
⇐
Execution Result Contract = ROW_STREAM
```

o una adaptación explícitamente autorizada.

---

# 226. No hidden mode switching

No:

```text
result too big
↓
silently switch buffered → stream
```

ni:

```text
result small
↓
silently stream → buffer
```

si cambia el contrato observable.

---

# 227. Streaming factory

```php
interface StreamingResultFactory
{
    public function create(
        ResultCursor $cursor,
        StreamingResultDescriptor $descriptor,
        StreamingContext $context,
        StreamingResourceScope $resources
    ): StreamingResult;
}
```

---

# 228. StreamingResultDescriptor

Immutable:

```php
final readonly class StreamingResultDescriptor
{
    public function __construct(
        public ResultMetadata $metadata,
        public StreamingGuarantee $guarantee,
        public StreamCapabilitySet $capabilities,
        public StreamingPolicy $policy,
    ) {}
}
```

---

# 229. Factory does not execute query

Sólo construye el streaming lifecycle sobre un cursor/result ya ejecutado.

---

# 230. Extension model

Las extensiones podrán proporcionar:

```text
custom stream producer
custom async adapter
custom reactive adapter
specialized bounded buffer
driver-specific streaming capability
telemetry observer
diagnostic enricher
```

---

# 231. Extension restrictions

No podrán:

```text
drop rows
duplicate rows
reorder rows
execute hidden SQL
remove security metadata
ignore cancellation
ignore budgets
retain connections globally
change transaction ownership
```

---

# 232. Extension registry

```text
discover
↓
validate descriptors
↓
resolve dependencies
↓
detect conflicts
↓
deterministic ordering
↓
compose
↓
freeze
```

---

# 233. No last-wins

Dos extensiones reclamando el mismo exact extension point de forma incompatible deberán producir error.

---

# 234. Runtime extension state

Todo estado mutable:

```text
extension stream state
```

vivirá dentro del `StreamingSession`.

---

# 235. Suggested namespace

```text
VoltStack\Quantum\Database\Execution\Result\Streaming
```

---

# 236. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Result/
                └── Streaming/
                    ├── Contract/
                    │   ├── StreamingResult.php
                    │   ├── StreamProducer.php
                    │   ├── StreamConsumer.php
                    │   ├── BatchStreamingResult.php
                    │   ├── StreamCompletionListener.php
                    │   └── StreamingResultFactory.php
                    │
                    ├── Stream/
                    │   ├── DefaultStreamingResult.php
                    │   ├── CursorStreamingResult.php
                    │   └── StreamingResultDescriptor.php
                    │
                    ├── Read/
                    │   ├── StreamReadResult.php
                    │   ├── StreamReadStatus.php
                    │   ├── StreamItem.php
                    │   ├── StreamEnd.php
                    │   ├── StreamSequence.php
                    │   └── StreamBatchReadResult.php
                    │
                    ├── Producer/
                    │   ├── CursorStreamProducer.php
                    │   ├── StreamDemand.php
                    │   └── StreamProductionResult.php
                    │
                    ├── Consumer/
                    │   ├── ManagedStreamConsumer.php
                    │   ├── StreamConsumerContext.php
                    │   └── StreamConsumerDecision.php
                    │
                    ├── State/
                    │   ├── StreamState.php
                    │   ├── StreamId.php
                    │   ├── StreamingSession.php
                    │   ├── StreamingContext.php
                    │   └── StreamCompletion.php
                    │
                    ├── Lifecycle/
                    │   ├── StreamTerminationReason.php
                    │   ├── StreamInvalidationReason.php
                    │   ├── StreamRetrySafety.php
                    │   └── StreamingResultFinalizer.php
                    │
                    ├── Capability/
                    │   ├── StreamCapability.php
                    │   ├── StreamCapabilitySet.php
                    │   ├── StreamCapabilityDescriptor.php
                    │   └── StreamingGuarantee.php
                    │
                    ├── Policy/
                    │   ├── StreamingPolicy.php
                    │   ├── StreamingMode.php
                    │   └── StreamClosePolicy.php
                    │
                    ├── Buffer/
                    │   ├── StreamBufferManager.php
                    │   ├── BoundedStreamBuffer.php
                    │   ├── StreamBufferOverflowPolicy.php
                    │   └── StreamBufferMetrics.php
                    │
                    ├── Backpressure/
                    │   ├── BackpressureController.php
                    │   ├── DemandTracker.php
                    │   └── BackpressureState.php
                    │
                    ├── Resource/
                    │   ├── StreamingResourceScope.php
                    │   ├── StreamingResourceGovernor.php
                    │   ├── StreamingResourceRequest.php
                    │   ├── StreamingResourceReservation.php
                    │   └── StreamResourceImpact.php
                    │
                    ├── Cancellation/
                    │   ├── StreamCancellationToken.php
                    │   ├── StreamCancellationController.php
                    │   ├── StreamCancellationReason.php
                    │   └── StreamCancellationOutcome.php
                    │
                    ├── Timeout/
                    │   ├── StreamDeadline.php
                    │   ├── StreamTimeoutPolicy.php
                    │   └── StreamIdleDeadline.php
                    │
                    ├── Budget/
                    │   ├── StreamBudget.php
                    │   ├── StreamBudgetPolicy.php
                    │   └── StreamBudgetSnapshot.php
                    │
                    ├── Transaction/
                    │   ├── StreamTransactionRequirement.php
                    │   └── StreamTransactionAffinity.php
                    │
                    ├── Failure/
                    │   ├── StreamFailureOrigin.php
                    │   ├── StreamFailureContext.php
                    │   └── StreamFailureClassifier.php
                    │
                    ├── Telemetry/
                    │   ├── StreamTelemetry.php
                    │   ├── StreamMetrics.php
                    │   └── StreamObserver.php
                    │
                    ├── Diagnostic/
                    │   ├── StreamDiagnostic.php
                    │   ├── StreamLeakDiagnostic.php
                    │   └── StreamFailureDiagnostic.php
                    │
                    ├── Async/
                    │   ├── AsyncStreamingResult.php
                    │   └── AsyncStreamAdapter.php
                    │
                    ├── Reactive/
                    │   └── ReactiveStreamAdapter.php
                    │
                    ├── Extension/
                    │   ├── StreamingExtension.php
                    │   ├── StreamingExtensionDescriptor.php
                    │   └── StreamingExtensionRegistry.php
                    │
                    └── Exception/
                        ├── StreamingResultException.php
                        ├── StreamStateException.php
                        ├── StreamClosedException.php
                        ├── StreamCapabilityException.php
                        ├── StreamReadException.php
                        ├── StreamCursorException.php
                        ├── StreamConversionException.php
                        ├── StreamBufferException.php
                        ├── StreamBudgetException.php
                        ├── StreamBudgetExceededException.php
                        ├── StreamCancellationException.php
                        ├── StreamTimeoutException.php
                        ├── StreamResourceException.php
                        ├── StreamOwnershipException.php
                        ├── StreamTransactionException.php
                        ├── StreamInvalidationException.php
                        ├── StreamBackpressureException.php
                        ├── StreamConsumerException.php
                        ├── StreamDriverException.php
                        ├── StreamSecurityException.php
                        ├── StreamExtensionException.php
                        ├── ConcurrentStreamConsumptionException.php
                        └── StreamInvariantException.php
```

---

# 237. Testing strategy

El sistema deberá probar:

```text
empty stream
single row
millions of rows
NULL values
duplicate rows
duplicate column labels
early stop
explicit close
cursor exhaustion
cursor failure
conversion failure
consumer failure
connection loss
transaction rollback
cancellation
timeout
idle timeout
budget exhaustion
bounded memory
buffer limits
persistent workers
coroutines
resource leaks
```

---

# 238. Backpressure tests

Verificar:

```text
consumer requests 1
→ producer does not deliver 2
```

salvo buffering previamente autorizado.

---

# 239. Memory tests

Para `n` muy grande:

```text
Memory(n)
```

deberá permanecer aproximadamente estable respecto a `n`, dentro de las garantías reales del driver.

---

# 240. Slow consumer test

```text
fast database
+
slow consumer
```

no deberá provocar crecimiento ilimitado de memoria.

---

# 241. Early close test

Después de:

```text
10 rows consumed
↓
close()
```

verificar:

```text
cursor closed
statement released
connection released/reset
execution finalized
```

---

# 242. Partial failure test

```text
deliver 100 rows
↓
driver failure
```

debe producir:

```text
rowsDelivered = 100
fullyConsumed = false
state = FAILED
```

---

# 243. Persistent worker test

```text
Request A opens/closes stream
Request B opens stream
```

deberá demostrar:

```text
no mutable state leakage
```

---

# 244. Concurrency test

Dos consumers sobre el mismo stream deberán fallar o serializarse según contrato; V1 deberá fallar explícitamente.

---

# 245. Transaction affinity test

Verificar que un stream transaction-bound no cambie de connection.

---

# 246. Cancellation test

Verificar:

```text
cancel
↓
no new rows
↓
driver cancellation if supported
↓
cleanup
```

---

# 247. Capability conformance

Los drivers oficiales deberán declarar correctamente:

```text
TRUE_INCREMENTAL
DRIVER_BUFFERED
PARTIALLY_BUFFERED
FULLY_BUFFERED_BY_DRIVER
UNKNOWN
```

según comportamiento real.

---

# 248. Performance targets

El hot path deberá evitar:

```text
reflection
service container lookups
global context lookup
metadata reconstruction
unbounded allocation
per-row logger calls
```

---

# 249. Per-row complexity

Para una fila de `c` columnas:

```text
T_stream_row
≈
T_cursor_fetch
+
O(c)
+
T_conversion
+
O(1) stream accounting
```

---

# 250. Memory complexity

Para buffer `B`:

```text
Memory
≈
O(B × average row width)
+
O(metadata)
+
O(runtime state)
```

---

# 251. Ideal forward-only case

Con:

```text
B ≈ 1
```

la memoria controlada por VoltStack es aproximadamente independiente de la cardinalidad total.

---

# 252. Anti-pattern — fetchAll + yield

```php
$rows = $statement->fetchAll();

foreach ($rows as $row) {
    yield $row;
}
```

Esto es lazy delivery, no true streaming.

---

# 253. Anti-pattern — hidden query pagination

```text
stream next
↓
SELECT ... LIMIT 100 OFFSET ...
```

sin que el plan lo haya declarado.

---

# 254. Anti-pattern — unbounded producer

```text
while ($row = fetch()) {
    $queue[] = $row;
}
```

---

# 255. Anti-pattern — auto retry after delivery

```text
100 rows delivered
↓
network failure
↓
rerun query from beginning
```

---

# 256. Anti-pattern — implicit buffering

```php
public function rewind(): void
{
    $this->bufferAllRemainingRows();
}
```

---

# 257. Anti-pattern — transaction ownership confusion

```php
public function close(): void
{
    $this->connection->commit();
}
```

---

# 258. Anti-pattern — connection returned early

```text
stream open
+
connection already returned to pool
```

cuando cursor depende de ella.

---

# 259. Anti-pattern — global stream state

```php
static $currentStream;
```

---

# 260. Anti-pattern — treating cancellation as EOF

```text
cancelled
↓
StreamEnd
```

sin informar cancellation.

---

# 261. Anti-pattern — silent truncation

```text
memory limit reached
↓
pretend end-of-stream
```

---

# 262. Anti-pattern — per-row sensitive logging

```php
$logger->debug('stream row', $row->toArray());
```

---

# 263. Anti-pattern — hidden authorization filtering

```php
if (!$authorization->canRead($row)) {
    continue;
}
```

dentro del core streaming.

---

# 264. Anti-pattern — God StreamingResult

No concentrar en una sola clase:

```text
cursor management
buffers
transactions
telemetry
cancellation
resource pool
driver calls
security
extensions
```

---

# 265. Architectural invariants

## DB-STREAM-001

`StreamingResult` será distinto de `ResultCursor`.

## DB-STREAM-002

`StreamingResult` será distinto de `BufferedResult`.

## DB-STREAM-003

`StreamingResult` será distinto de `Generator`.

## DB-STREAM-004

`StreamingResult` será distinto de `LazyCollection`.

## DB-STREAM-005

`StreamingResult` será distinto de ORM Collection.

## DB-STREAM-006

`StreamingResult` será distinto de pagination.

## DB-STREAM-007

Streaming consumirá un resultado ya ejecutado.

## DB-STREAM-008

Streaming no generará SQL.

## DB-STREAM-009

Streaming no ejecutará hidden queries.

## DB-STREAM-010

Streaming no modificará query cardinality.

## DB-STREAM-011

Streaming no añadirá LIMIT/OFFSET.

## DB-STREAM-012

V1 utilizará pull-based streaming como baseline.

## DB-STREAM-013

Backpressure será first-class.

## DB-STREAM-014

Buffers controlados serán bounded.

## DB-STREAM-015

No existirán unbounded queues internas.

## DB-STREAM-016

Demand controlará producción en pull mode.

## DB-STREAM-017

Produced no excederá demanda más buffer autorizado.

## DB-STREAM-018

Database rows no serán dropped por buffer pressure.

## DB-STREAM-019

Database rows no serán duplicated por streaming.

## DB-STREAM-020

Database rows no serán reordered por streaming.

## DB-STREAM-021

SQL NULL será preservado.

## DB-STREAM-022

Duplicate rows serán preservadas.

## DB-STREAM-023

Stream sequence será distinta de database identity.

## DB-STREAM-024

Stream sequence será distinta de pagination cursor.

## DB-STREAM-025

Streaming guarantee será explícita.

## DB-STREAM-026

Driver-buffered streaming no se presentará como true incremental.

## DB-STREAM-027

Unknown driver buffering será representable.

## DB-STREAM-028

Batch size será distinto de SQL LIMIT.

## DB-STREAM-029

Batch size será distinto de pagination.

## DB-STREAM-030

Early termination será distinta de exhaustion.

## DB-STREAM-031

Cancellation será distinta de exhaustion.

## DB-STREAM-032

Timeout será distinto de cancellation.

## DB-STREAM-033

Failure será distinto de cancellation.

## DB-STREAM-034

Partial consumption será representable.

## DB-STREAM-035

Partial consumption no se presentará como complete.

## DB-STREAM-036

fullyConsumed sólo será true después de exhaustion válida.

## DB-STREAM-037

close antes de exhaustion será early termination.

## DB-STREAM-038

close será idealmente idempotente.

## DB-STREAM-039

next después de CLOSED fallará.

## DB-STREAM-040

Streaming resources tendrán ownership explícito.

## DB-STREAM-041

Resource transfer será explícito.

## DB-STREAM-042

Transferred resources no serán liberados por owner anterior.

## DB-STREAM-043

Streaming podrá mantener ExecutionInstance en DRAINING.

## DB-STREAM-044

Stream creation success será distinto de full execution success.

## DB-STREAM-045

Execution completion podrá ser diferida hasta stream finalization.

## DB-STREAM-046

Cursor lifetime será respetado.

## DB-STREAM-047

Statement lifetime será respetado.

## DB-STREAM-048

Connection lifetime será respetado.

## DB-STREAM-049

Transaction lifetime será respetado cuando aplique.

## DB-STREAM-050

Connection pinning será explícito.

## DB-STREAM-051

Long-lived streams estarán sujetos a resource governance.

## DB-STREAM-052

Pool starvation deberá poder limitarse.

## DB-STREAM-053

Streaming no hará commit de caller-owned transaction.

## DB-STREAM-054

Streaming no hará rollback de caller-owned transaction arbitrariamente.

## DB-STREAM-055

Transaction affinity será preservada.

## DB-STREAM-056

Invalidated streams no continuarán leyendo.

## DB-STREAM-057

Cancellation se propagará al cursor.

## DB-STREAM-058

Cancellation se propagará al driver cuando sea soportado.

## DB-STREAM-059

VoltStack no afirmará server-side cancellation sin garantía.

## DB-STREAM-060

Deadline del stream no extenderá execution deadline silenciosamente.

## DB-STREAM-061

Monotonic clock se utilizará para duraciones cuando sea posible.

## DB-STREAM-062

Idle timeout será distinto de execution timeout.

## DB-STREAM-063

Budgets serán explícitos.

## DB-STREAM-064

Budgets no cambiarán SQL semantics.

## DB-STREAM-065

Budget exhaustion no será tratado como EOF.

## DB-STREAM-066

No existirá silent truncation.

## DB-STREAM-067

Framework buffer memory será contabilizada.

## DB-STREAM-068

Driver memory será distinguida de framework memory.

## DB-STREAM-069

Streaming conversion reutilizará compiled conversion plans.

## DB-STREAM-070

Streaming no hará ORM hydration.

## DB-STREAM-071

Conversion failure después de delivery será partial failure.

## DB-STREAM-072

Delivered rows no pueden ser retroactivamente retiradas.

## DB-STREAM-073

Whole-stream retry después de delivery será unsafe por defecto.

## DB-STREAM-074

Streaming no inventará resume keys.

## DB-STREAM-075

Resumability requerirá soporte explícito del plan.

## DB-STREAM-076

Backpressure deberá propagarse a través de adapters.

## DB-STREAM-077

Hydration adapters no deberán eager-load todo el stream.

## DB-STREAM-078

Core StreamingResult no será biblioteca funcional general.

## DB-STREAM-079

Functional stream operations pertenecerán a adapters.

## DB-STREAM-080

Adapters deberán preservar cleanup.

## DB-STREAM-081

Rows serán consideradas potencialmente sensibles.

## DB-STREAM-082

Telemetry no registrará row values por defecto.

## DB-STREAM-083

Per-row telemetry no será obligatoria.

## DB-STREAM-084

Metrics tendrán cardinalidad controlada.

## DB-STREAM-085

StreamId será runtime identity.

## DB-STREAM-086

StreamId será distinto de query fingerprint.

## DB-STREAM-087

Tenant authorization no será realizada por streaming.

## DB-STREAM-088

No existirá global current tenant dentro del stream.

## DB-STREAM-089

Mutable stream state será operation-scoped.

## DB-STREAM-090

Persistent workers no compartirán mutable stream state.

## DB-STREAM-091

Coroutines no compartirán mutable stream state.

## DB-STREAM-092

Detached live streams entre requests estarán prohibidos por defecto.

## DB-STREAM-093

Shared StreamingResult services serán stateless/immutable.

## DB-STREAM-094

Un stream tendrá un active consumer por defecto.

## DB-STREAM-095

Concurrent consumption no será ambiguo.

## DB-STREAM-096

V1 fallará ante unsupported concurrent consumption.

## DB-STREAM-097

Async será distinto de multi-consumer.

## DB-STREAM-098

Reactive adapters serán opcionales.

## DB-STREAM-099

Core Database no dependerá obligatoriamente de reactive libraries.

## DB-STREAM-100

Reactive adapters preservarán demand.

## DB-STREAM-101

Reactive adapters preservarán cancellation.

## DB-STREAM-102

Reactive adapters preservarán ordering.

## DB-STREAM-103

Reactive adapters preservarán errors.

## DB-STREAM-104

Reactive adapters preservarán completion.

## DB-STREAM-105

StreamingSession será operation-scoped.

## DB-STREAM-106

StreamingContext será immutable.

## DB-STREAM-107

StreamingSession no será service locator.

## DB-STREAM-108

StreamingSession no contendrá global HTTP state.

## DB-STREAM-109

Cleanup seguirá resource dependency ordering.

## DB-STREAM-110

Cleanup failures no ocultarán primary failure.

## DB-STREAM-111

Consumer exceptions serán preservadas.

## DB-STREAM-112

Consumer failure será distinguible de database failure.

## DB-STREAM-113

ExecutionInstance será notificado de stream completion.

## DB-STREAM-114

ExecutionInstance será notificado de stream failure.

## DB-STREAM-115

ExecutionInstance será notificado de cancellation.

## DB-STREAM-116

ExecutionInstance será notificado de timeout.

## DB-STREAM-117

ROW_STREAM deberá provenir del Execution Result Contract.

## DB-STREAM-118

QueryExecutor no inventará streaming arbitrariamente.

## DB-STREAM-119

StreamingResultFactory no ejecutará queries.

## DB-STREAM-120

Streaming extensions serán tipadas.

## DB-STREAM-121

Extension registry será frozen.

## DB-STREAM-122

Extensions no ejecutarán hidden SQL.

## DB-STREAM-123

Extensions no reordenarán rows.

## DB-STREAM-124

Extensions no eliminarán rows.

## DB-STREAM-125

Extensions no duplicarán rows.

## DB-STREAM-126

Extensions no ignorarán budgets.

## DB-STREAM-127

Extensions no ignorarán cancellation.

## DB-STREAM-128

Extensions no alterarán transaction ownership.

## DB-STREAM-129

Extension mutable state será stream-scoped.

## DB-STREAM-130

Streaming memory deberá permanecer bounded respecto al result total dentro de las garantías controlables.

## DB-STREAM-131

Slow consumers no provocarán framework buffer growth ilimitado.

## DB-STREAM-132

Resource leaks deberán ser detectables.

## DB-STREAM-133

Destructors serán sólo safety net.

## DB-STREAM-134

Explicit close será el mecanismo normal de cleanup.

## DB-STREAM-135

Request termination cerrará streams owned por ese scope.

## DB-STREAM-136

Worker shutdown cerrará streams activos.

## DB-STREAM-137

Connection loss invalidará streams dependientes.

## DB-STREAM-138

Rollback invalidará streams dependientes cuando corresponda.

## DB-STREAM-139

Streaming no realizará semantic query reinterpretation.

## DB-STREAM-140

Streaming preservará exactamente las decisiones del ExecutionPlan.

---

# 266. Invariante maestro de semántica

Sea:

```text
R = ordered database result
```

y:

```text
S = stream output
```

si el stream termina por exhaustion exitosa:

```text
S = R
```

preservando:

```text
ordering
duplicates
NULL
column identity
typed values
```

---

# 267. Invariante de partial delivery

Si el consumidor termina después de `k` filas:

```text
S = prefix(R, k)
```

y:

```text
fullyConsumed = false
```

---

# 268. Invariante de backpressure

Para cualquier instante `t`:

```text
Produced(t)
≤
Demanded(t)
+
BufferCapacity
```

---

# 269. Invariante de buffer

```text
0
≤
BufferedItems
≤
ConfiguredBufferCapacity
```

---

# 270. Invariante de no pérdida

```text
Produced
=
Delivered
+
Buffered
```

mientras los items sigan vivos y no exista termination/failure explícita.

---

# 271. Invariante de memoria

Para buffer bounded `B`:

```text
FrameworkControlledMemory
=
O(B × AverageRowWidth)
+
O(Metadata)
+
O(StreamRuntimeState)
```

y no:

```text
O(TotalRows)
```

---

# 272. Invariante de ownership

Para todo recurso vivo `R`:

```text
∃! Owner(R)
```

es decir, existe exactamente un owner responsable o un contrato explícito de borrowed ownership.

---

# 273. Invariante de finalización

```text
Terminate(Stream)
⇒
EventuallyFinalizeResources(Stream)
```

---

# 274. Invariante de retry

```text
RowsDelivered > 0
∧
NoExplicitResumability
⇒
AutomaticWholeStreamRetry = false
```

---

# 275. Invariante de execution lifecycle

Mientras un streaming result posea recursos de ejecución:

```text
StreamResourcesAlive
⇒
ExecutionInstance ≠ CLOSED
```

---

# 276. Invariante de completion

```text
StreamFullyConsumed
∧
CleanupSuccessful
⇒
ExecutionMayBecomeCompleted
```

---

# 277. Invariante de seguridad

```text
StreamingTelemetry
∩
SensitiveRowValues
=
∅
```

por defecto.

---

# 278. Invariante de aislamiento

Para streams de operaciones diferentes:

```text
MutableStreamState(A)
∩
MutableStreamState(B)
=
∅
```

---

# 279. Invariante de no reejecución

```text
StreamingReadOperation
⇒
0 hidden query re-executions
```

---

# 280. Flujo completo

```text
ExecutionPlan
      ↓
QueryExecutor
      ↓
StatementExecutor
      ↓
Driver Execution
      ↓
DatabaseResult
      ↓
ResultCursor
      ↓
StreamingResultFactory
      ↓
StreamingSession
      ↓
StreamingResult
      ↓
Consumer Demand
      ↓
ResultCursor.fetch()
      ↓
ResultRow
      ↓
Consumer
      ↓
next demand
      ↓
...
      ↓
EXHAUSTED
      ↓
Stream Finalizer
      ↓
Cursor Cleanup
      ↓
Statement Cleanup
      ↓
Connection Release
      ↓
ExecutionInstance Completion
```

---

# 281. Early-stop flow

```text
Consumer
   ↓
STOP
   ↓
StreamingResult
   ↓
termination = CONSUMER_STOP
   ↓
stop producing
   ↓
cancel/close cursor
   ↓
cleanup resources
   ↓
notify ExecutionInstance
```

---

# 282. Failure flow

```text
Cursor fetch
   ↓
Driver failure
   ↓
Stream FAILED
   ↓
record partial progress
   ↓
stop production
   ↓
classify connection impact
   ↓
cleanup
   ↓
notify ExecutionInstance
   ↓
propagate normalized failure
```

---

# 283. Timeout flow

```text
Deadline exceeded
   ↓
TIMED_OUT
   ↓
cancel active cursor/driver
   ↓
stop delivery
   ↓
cleanup
   ↓
ExecutionInstance TIMED_OUT
```

---

# 284. Arquitectura final

```text
                     STREAMING RESULT SYSTEM

                          Consumer
                             │
                          Demand
                             │
                             ▼
                   ┌──────────────────┐
                   │ StreamingResult  │
                   └────────┬─────────┘
                            │
             ┌──────────────┼───────────────┐
             │              │               │
             ▼              ▼               ▼
       Backpressure      Lifecycle        Budget
             │              │               │
             └──────────────┼───────────────┘
                            │
                            ▼
                    StreamingSession
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
        ▼                   ▼                    ▼
 Cancellation         Resource Scope         Telemetry
        │                   │
        │                   ▼
        │              ResultCursor
        │                   │
        └───────────────────┤
                            ▼
                       Driver Result
                            │
                            ▼
                         Database
```

---

# 285. Fórmula arquitectónica

```text
Streaming Result System
=
Result Cursor
+
Demand
+
Backpressure
+
Bounded Buffering
+
Streaming Lifecycle
+
Resource Ownership
+
Connection Pinning
+
Transaction Affinity
+
Cancellation
+
Deadlines
+
Budgets
+
Partial Consumption Semantics
+
Failure Propagation
+
Deterministic Cleanup
+
Telemetry
+
Persistent Runtime Isolation
```

---

# 286. Fórmula de corrección

```text
CorrectStreaming
=
SemanticPreservation
∧
OrderedDelivery
∧
DuplicatePreservation
∧
BoundedFrameworkMemory
∧
Backpressure
∧
ExplicitOwnership
∧
ExplicitTermination
∧
CancellationSafety
∧
FailureTransparency
∧
PersistentRuntimeIsolation
```

---

# 287. Fórmula de backpressure

```text
StreamingBackpressure
=
ConsumerDemand
-
OutstandingProduction
+
BoundedBufferControl
```

sujeto a:

```text
OutstandingProduction
≤
Demand + BufferCapacity
```

---

# 288. Fórmula de resource safety

```text
StreamingResourceSafety
=
Ownership Transfer
+
Pinned Resource Tracking
+
Termination Detection
+
Ordered Cleanup
+
Connection State Classification
+
Execution Finalization
```

---

# 289. Principio final

> **Streaming is not `fetchAll()` delivered slowly. Streaming is a bounded, demand-driven lifecycle over live execution resources.**

Por tanto:

```text
True VoltStack Streaming
=
Progressive Consumption
+
Bounded Memory
+
Backpressure
+
Explicit Resource Lifetime
+
Failure-aware Partial Delivery
```

y nunca:

```text
fetchAll()
↓
yield rows one by one
```

como sustituto arquitectónico.

---

# 290. Siguiente documento

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
```

El siguiente documento deberá formalizar los mecanismos transversales de timeout y cancellation aplicables a:

```text
ExecutionInstance
Statement Execution
Prepared Statement
Result Cursor
Streaming Result
Transaction-bound operations
Connection acquisition
Driver operations
```

incluyendo:

```text
Cancellation Token Model
Cancellation Source
Cancellation Propagation
Cancellation Scope
Execution Deadlines
Query Deadlines
Statement Timeouts
Fetch Timeouts
Stream Deadlines
Idle Timeouts
Driver Cancellation Capabilities
Server-side Cancellation
Client-side Cancellation
Connection Impact
Transaction Impact
Partial Effects
Race Conditions
Cancellation vs Completion
Cancellation vs Failure
Cancellation vs Timeout
Deadline Propagation
Nested Deadline Composition
Monotonic Clocks
Cleanup
Telemetry
Security
Persistent Runtime Safety
```

manteniendo como principio:

```text
Cancellation
≠
Failure
≠
Timeout
≠
Successful Completion
```

y:

```text
Timeout
=
Deadline Expiration
```

mientras:

```text
Cancellation
=
Explicit Termination Request
```

aunque un timeout pueda utilizar internamente el mecanismo de cancellation para detener una operación activa.