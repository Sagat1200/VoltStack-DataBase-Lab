# 85_DATABASE_EXECUTION_ERROR_SYSTEM.md

# VoltStack Quantum Database
## Execution Error System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 85 — Execution Error System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Execution Error System` define la arquitectura mediante la cual VoltStack representa, normaliza, clasifica, conserva y propaga los errores producidos durante la ejecución de operaciones de base de datos.

La frontera principal es:

```text
Driver / Runtime Failure
        ↓
Execution Error Adapter
        ↓
Normalized Execution Error
        ↓
Impact Classification
        ↓
Recovery / Retry Metadata
        ↓
Execution Engine
        ↓
Application / Telemetry / Diagnostics
```

El objetivo fundamental es evitar dos extremos igualmente problemáticos:

```text
raw driver exceptions everywhere
```

y:

```text
over-normalized generic exceptions
that destroy useful native information
```

VoltStack deberá conservar ambos niveles:

```text
Normalized Framework Meaning
+
Native Driver Evidence
```

Principio central:

```text
Normalize Error Semantics
≠
Destroy Native Error Information
```

---

# 2. Principio fundamental

Todo error de ejecución deberá poder responder, cuando la evidencia lo permita:

1. ¿Qué ocurrió?
2. ¿Dónde ocurrió?
3. ¿En qué fase ocurrió?
4. ¿La operación llegó a ser enviada al servidor?
5. ¿Hubo efectos parciales?
6. ¿La conexión sigue siendo reusable?
7. ¿La transacción sigue siendo usable?
8. ¿El statement sigue siendo reusable?
9. ¿El resultado está parcial o completamente consumido?
10. ¿La operación puede reintentarse?
11. ¿El outcome es conocido o incierto?
12. ¿Qué información puede mostrarse de forma segura?
13. ¿Cuál fue la causa nativa original?
14. ¿Qué código SQLSTATE/error code reportó el driver?
15. ¿Qué capa originó realmente el error?

---

# 3. Posición arquitectónica

```text
Execution Engine
├── QueryExecutor
├── StatementExecutor
├── Prepared Statement System
├── Parameter Binding System
├── Result System
├── Result Cursor
├── Streaming Result
├── Timeout / Cancellation
└── Execution Error System
```

El `Execution Error System` es transversal.

No pertenece exclusivamente al:

```text
Driver
```

ni al:

```text
StatementExecutor
```

ni al:

```text
QueryExecutor
```

---

# 4. Flujo general

```text
Native Driver Error
      ↓
NativeErrorCapture
      ↓
ExecutionErrorClassifier
      ↓
ExecutionErrorNormalizer
      ↓
ExecutionFailure
      ├── Category
      ├── ErrorCode
      ├── SQLSTATE
      ├── Phase
      ├── Cause
      ├── ConnectionImpact
      ├── TransactionImpact
      ├── StatementImpact
      ├── OutcomeCertainty
      ├── RetryClassification
      ├── SecurityMetadata
      └── Diagnostics
```

---

# 5. Objetivos

El sistema deberá proporcionar:

- taxonomía común de errores;
- adaptación de excepciones nativas;
- preservación de SQLSTATE;
- preservación de códigos nativos;
- preservación de causa original;
- clasificación por fase;
- clasificación de impacto;
- clasificación de recoverability;
- clasificación de retry;
- outcome certainty;
- partial effects;
- source mapping;
- safe diagnostics;
- sensitive-data redaction;
- primary/suppressed error model;
- telemetry estructurada;
- compatibilidad con drivers múltiples;
- soporte para MySQL, MariaDB, PostgreSQL y SQLite;
- extensibilidad tipada;
- seguridad para runtimes persistentes.

---

# 6. No objetivos

Este sistema no deberá:

```text
retry queries by itself
rollback transactions by itself
reconnect by itself
compile SQL
parse SQL text to guess semantics
hydrate entities
perform authorization
hide native errors completely
log sensitive values
change query semantics
```

---

# 7. Error ≠ Exception

Conviene distinguir:

```text
ExecutionFailure
=
structured failure model
```

de:

```text
ExecutionException
=
PHP throwable representation
```

---

# 8. Structured error first

VoltStack deberá favorecer un modelo estructurado:

```php
final readonly class ExecutionFailure
{
    public function __construct(
        public ExecutionFailureCategory $category,
        public ExecutionFailureCode $code,
        public ExecutionPhase $phase,
        public OutcomeCertainty $certainty,
        public ConnectionImpact $connectionImpact,
        public TransactionImpact $transactionImpact,
        public StatementImpact $statementImpact,
        public RetryClassification $retry,
        public FailureSeverity $severity,
        public ?SqlState $sqlState,
        public ?NativeDatabaseErrorCode $nativeCode,
        public Throwable $cause,
        public ExecutionFailureContext $context,
    ) {}
}
```

---

# 9. ExecutionFailureCategory

Propuesta:

```php
enum ExecutionFailureCategory: string
{
    case CONNECTION_ACQUISITION = 'connection_acquisition';
    case CONNECTION_LOST = 'connection_lost';
    case CONNECTION_PROTOCOL = 'connection_protocol';

    case PREPARATION = 'preparation';
    case BINDING = 'binding';
    case EXECUTION = 'execution';

    case CONSTRAINT = 'constraint';
    case UNIQUE_CONSTRAINT = 'unique_constraint';
    case FOREIGN_KEY_CONSTRAINT = 'foreign_key_constraint';
    case NOT_NULL_CONSTRAINT = 'not_null_constraint';
    case CHECK_CONSTRAINT = 'check_constraint';

    case DEADLOCK = 'deadlock';
    case SERIALIZATION = 'serialization';
    case LOCK_TIMEOUT = 'lock_timeout';

    case QUERY_TIMEOUT = 'query_timeout';
    case CANCELLATION = 'cancellation';

    case RESULT = 'result';
    case CURSOR = 'cursor';
    case STREAM = 'stream';

    case TRANSACTION = 'transaction';
    case TRANSACTION_ABORTED = 'transaction_aborted';
    case COMMIT_UNKNOWN = 'commit_unknown';

    case RESOURCE_EXHAUSTION = 'resource_exhaustion';
    case MEMORY_LIMIT = 'memory_limit';

    case DRIVER = 'driver';
    case PLATFORM = 'platform';

    case SECURITY = 'security';
    case EXTENSION = 'extension';
    case INVARIANT = 'invariant';

    case UNKNOWN = 'unknown';
}
```

---

# 10. Category hierarchy

En implementación podrá preferirse un modelo más rico que un enum plano.

Conceptualmente:

```text
Execution Failure
├── Connectivity
├── Preparation
├── Binding
├── Execution
├── Constraint
├── Concurrency
├── Timeout
├── Cancellation
├── Result
├── Transaction
├── Resource
├── Driver
├── Security
├── Extension
└── Invariant
```

---

# 11. Error code

VoltStack deberá disponer de códigos internos estables:

```php
final readonly class ExecutionFailureCode
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
DB_EXEC_CONNECTION_LOST
DB_EXEC_UNIQUE_VIOLATION
DB_EXEC_FOREIGN_KEY_VIOLATION
DB_EXEC_DEADLOCK
DB_EXEC_SERIALIZATION_FAILURE
DB_EXEC_QUERY_TIMEOUT
DB_EXEC_CANCELLED
DB_EXEC_RESULT_SHAPE_MISMATCH
DB_EXEC_COMMIT_OUTCOME_UNKNOWN
```

---

# 12. Framework code ≠ SQLSTATE

```text
VoltStack Failure Code
≠
SQLSTATE
≠
Vendor Native Code
```

Los tres pueden coexistir.

---

# 13. SQLSTATE

Debe conservarse cuando exista.

```php
final readonly class SqlState
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 14. SQLSTATE preservation

Ejemplo:

```text
SQLSTATE = 23505
```

puede contribuir a clasificar:

```text
UNIQUE_CONSTRAINT
```

sin perder:

```text
23505
```

---

# 15. Native error code

Puede existir:

```php
final readonly class NativeDatabaseErrorCode
{
    public function __construct(
        public string|int $value,
    ) {}
}
```

Ejemplo:

```text
MySQL error 1062
PostgreSQL SQLSTATE 23505
SQLite constraint extended code
```

---

# 16. Native code ≠ portable semantics

La API pública deberá operar principalmente sobre:

```text
normalized category
```

pero permitir acceder a:

```text
native details
```

cuando sea necesario.

---

# 17. Native cause

Toda excepción normalizada deberá preservar:

```text
previous Throwable
```

cuando exista.

---

# 18. No exception flattening

Incorrecto:

```php
throw new DatabaseException('Database error');
```

sin conservar:

```text
native error
SQLSTATE
vendor code
context
```

---

# 19. Execution phase

El error deberá conocer la fase en la que ocurrió.

```php
enum ExecutionPhase: string
{
    case PREFLIGHT = 'preflight';
    case CONNECTION_ACQUISITION = 'connection_acquisition';
    case PREPARATION = 'preparation';
    case BINDING = 'binding';
    case DISPATCH = 'dispatch';
    case EXECUTION = 'execution';
    case RESULT_ACQUISITION = 'result_acquisition';
    case RESULT_CONSUMPTION = 'result_consumption';
    case CURSOR_FETCH = 'cursor_fetch';
    case STREAM_CONSUMPTION = 'stream_consumption';
    case TRANSACTION_CONTROL = 'transaction_control';
    case CLEANUP = 'cleanup';
    case EXTENSION = 'extension';
}
```

---

# 20. Phase matters

Ejemplo:

```text
Connection lost before execute
```

es muy distinto de:

```text
Connection lost after COMMIT was sent
```

---

# 21. Dispatch boundary

Uno de los boundaries más importantes será:

```text
NOT_DISPATCHED
vs
DISPATCH_STARTED
```

---

# 22. Dispatch evidence

Se deberá conservar:

```php
enum DispatchEvidence
{
    case NOT_STARTED;
    case STARTED;
    case DRIVER_ACKNOWLEDGED;
    case SERVER_RESPONSE_RECEIVED;
    case UNKNOWN;
}
```

---

# 23. Outcome certainty

```php
enum OutcomeCertainty: string
{
    case NOT_EXECUTED = 'not_executed';
    case EXECUTED_SUCCESSFULLY = 'executed_successfully';
    case EXECUTED_FAILED = 'executed_failed';
    case PARTIAL_EFFECTS_POSSIBLE = 'partial_effects_possible';
    case UNKNOWN = 'unknown';
}
```

---

# 24. Conservative certainty

Cuando no exista evidencia suficiente:

```text
UNKNOWN
```

es correcto.

No debe inventarse precisión.

---

# 25. Example

```text
UPDATE dispatched
↓
connection disappears
↓
no response
```

Resultado:

```text
OutcomeCertainty = UNKNOWN
```

No:

```text
NOT_EXECUTED
```

---

# 26. Partial effects

Debe poder representarse:

```php
final readonly class PartialEffectMetadata
{
    public function __construct(
        public bool $possible,
        public bool $observed,
        public ?int $rowsDelivered,
        public ?int $operationsCompleted,
    ) {}
}
```

---

# 27. Partial effect examples

```text
stream delivered 100 rows
batch executed first 3 operations
driver acknowledged write but connection lost
consumer processed partial result
```

---

# 28. ConnectionImpact

Se reutiliza el concepto de documentos anteriores:

```php
enum ConnectionImpact: string
{
    case NONE = 'none';
    case HEALTH_CHECK_REQUIRED = 'health_check_required';
    case RESET_REQUIRED = 'reset_required';
    case INVALIDATE = 'invalidate';
    case LOST = 'lost';
    case UNKNOWN = 'unknown';
}
```

---

# 29. StatementImpact

```php
enum StatementImpact: string
{
    case NONE = 'none';
    case REUSABLE = 'reusable';
    case RESET_REQUIRED = 'reset_required';
    case INVALIDATE = 'invalidate';
    case UNKNOWN = 'unknown';
}
```

---

# 30. TransactionImpact

```php
enum TransactionImpact: string
{
    case NONE = 'none';
    case STATEMENT_FAILED_ONLY = 'statement_failed_only';
    case MARKED_FAILED = 'marked_failed';
    case ROLLBACK_REQUIRED = 'rollback_required';
    case ABORTED = 'aborted';
    case COMMIT_OUTCOME_UNKNOWN = 'commit_outcome_unknown';
    case UNKNOWN = 'unknown';
}
```

---

# 31. Impact separation

```text
Error Category
≠
Connection Impact
≠
Statement Impact
≠
Transaction Impact
```

Un mismo error puede afectar varias dimensiones.

---

# 32. Example

```text
serialization failure
```

podría producir:

```text
category = SERIALIZATION
connectionImpact = NONE
statementImpact = REUSABLE
transactionImpact = ROLLBACK_REQUIRED
retry = CONDITIONALLY_RETRYABLE
```

---

# 33. FailureSeverity

```php
enum FailureSeverity: string
{
    case NOTICE = 'notice';
    case RECOVERABLE = 'recoverable';
    case OPERATION_FATAL = 'operation_fatal';
    case TRANSACTION_FATAL = 'transaction_fatal';
    case CONNECTION_FATAL = 'connection_fatal';
    case PROCESS_FATAL = 'process_fatal';
}
```

---

# 34. Severity ≠ retryability

```text
RECOVERABLE
```

no significa necesariamente:

```text
RETRYABLE
```

---

# 35. RetryClassification

Aunque el documento 86 definirá la política completa, el error deberá aportar clasificación.

```php
enum RetryClassification: string
{
    case NEVER = 'never';
    case SAFE = 'safe';
    case CONDITIONALLY_SAFE = 'conditionally_safe';
    case RETRYABLE_IF_IDEMPOTENT = 'retryable_if_idempotent';
    case RETRYABLE_AFTER_TRANSACTION_RESTART = 'retryable_after_transaction_restart';
    case UNKNOWN = 'unknown';
}
```

---

# 36. Error classification ≠ retry decision

```text
Error System
→ classify
```

```text
Retry System
→ decide
```

---

# 37. Recoverability

Puede existir:

```php
enum Recoverability: string
{
    case NONE;
    case STATEMENT_LEVEL;
    case TRANSACTION_LEVEL;
    case CONNECTION_LEVEL;
    case EXECUTION_LEVEL;
    case APPLICATION_LEVEL;
    case UNKNOWN;
}
```

---

# 38. Constraint violations

Debe existir clasificación portable de restricciones.

---

# 39. Unique constraint

Ejemplo:

```text
UNIQUE_CONSTRAINT
```

deberá preservar cuando sea posible:

```text
constraint name
table
columns
SQLSTATE
vendor code
```

sin inventarlos si no están disponibles.

---

# 40. ConstraintViolationMetadata

```php
final readonly class ConstraintViolationMetadata
{
    public function __construct(
        public ConstraintViolationKind $kind,
        public ?string $constraintName,
        public ?string $table,
        public array $columns,
        public MetadataCertainty $certainty,
    ) {}
}
```

---

# 41. ConstraintViolationKind

```php
enum ConstraintViolationKind
{
    case UNIQUE;
    case FOREIGN_KEY;
    case NOT_NULL;
    case CHECK;
    case EXCLUSION;
    case PRIMARY_KEY;
    case UNKNOWN;
}
```

---

# 42. Constraint parsing caution

No deberá dependerse únicamente de:

```text
parsing human-readable driver message
```

cuando existan códigos estructurados.

---

# 43. Message parsing fallback

Podrá existir como:

```text
best-effort diagnostic enrichment
```

pero no deberá ser la fuente primaria de correctness.

---

# 44. Deadlock

`DEADLOCK` deberá diferenciarse de:

```text
LOCK_TIMEOUT
SERIALIZATION
```

---

# 45. Why

Aunque todos pertenezcan a concurrencia, tienen diferentes:

```text
retry semantics
transaction impact
telemetry
diagnostics
```

---

# 46. Serialization failure

Ejemplo conceptual:

```text
SERIALIZATION
```

puede requerir:

```text
retry entire transaction
```

no simplemente volver a ejecutar el statement.

---

# 47. Lock timeout

Puede indicar:

```text
statement aborted
```

o:

```text
transaction affected
```

según DBMS.

La clasificación deberá ser driver/platform-aware.

---

# 48. Timeout errors

Los errores producidos por el documento 84 deberán integrarse bajo:

```text
QUERY_TIMEOUT
```

o subtipos más específicos.

---

# 49. Cancellation errors

Cancellation no deberá colapsarse en:

```text
generic driver failure
```

si existe evidencia de que fue solicitada por VoltStack.

---

# 50. Cancellation causality

Si:

```text
CancellationSource requested cancel
```

y el driver devuelve:

```text
operation interrupted
```

el Error System deberá poder correlacionarlos.

---

# 51. But no false attribution

Si el driver devuelve “interrupted” sin cancellation solicitada:

```text
do not automatically claim user cancelled
```

---

# 52. Result errors

Incluirán:

```text
shape mismatch
type conversion failure
metadata mismatch
cursor fetch failure
stream failure
result resource failure
```

---

# 53. Result conversion failure

Ejemplo:

```text
expected UUID
driver returned malformed value
```

podrá clasificarse:

```text
RESULT
```

con subtype:

```text
RESULT_CONVERSION
```

---

# 54. Driver protocol errors

Errores de protocolo deben distinguirse de query errors.

Ejemplo:

```text
connection packet corruption
driver protocol desync
unexpected response
```

---

# 55. Resource exhaustion

Puede incluir:

```text
connection pool exhausted
prepared statement limit
memory budget exhausted
result budget exhausted
stream budget exhausted
temporary storage exhausted
```

---

# 56. Resource exhaustion ≠ DB constraint

No confundir:

```text
resource limit
```

con:

```text
schema constraint
```

---

# 57. Invariant errors

Errores internos como:

```text
invalid state transition
double resource ownership
missing execution output
illegal executor state
```

se clasifican como:

```text
INVARIANT
```

---

# 58. Invariant severity

Generalmente:

```text
OPERATION_FATAL
```

y en ciertos casos:

```text
PROCESS_FATAL
```

si implica corrupción global del runtime.

---

# 59. Extension errors

Una extensión podrá fallar.

Debe preservarse:

```text
extension id
extension version
extension point
phase
```

---

# 60. Extension error isolation

Una extensión fallida no deberá reportarse simplemente como:

```text
driver error
```

---

# 61. Error normalization pipeline

```text
N0 Native Error Capture
N1 Execution Context Capture
N2 Driver Error Extraction
N3 SQLSTATE Extraction
N4 Vendor Code Extraction
N5 Phase Classification
N6 Portable Category Classification
N7 Impact Classification
N8 Outcome Certainty Classification
N9 Retry Metadata Classification
N10 Security Redaction
N11 Source Mapping
N12 Framework Exception Construction
N13 Telemetry Dispatch
```

---

# 62. N0 — Native Error Capture

Capturar:

```text
Throwable
driver error object
native diagnostics
operation state
```

antes de cleanup destructivo si esa información podría perderse.

---

# 63. N1 — Execution Context

Debe capturarse contexto estructurado mínimo.

---

# 64. ExecutionFailureContext

```php
final readonly class ExecutionFailureContext
{
    public function __construct(
        public ExecutionId $executionId,
        public ?ExecutionUnitId $unitId,
        public ?StatementExecutionId $statementId,
        public ExecutionPhase $phase,
        public DispatchEvidence $dispatch,
        public ?CompiledCommandFingerprint $commandFingerprint,
        public ?ConnectionIdentity $connection,
        public ?TransactionId $transaction,
        public ?ResultId $result,
        public ?CursorId $cursor,
        public ?StreamId $stream,
    ) {}
}
```

---

# 65. Context ≠ global state

Nunca obtener después:

```text
current execution
current tenant
current connection
```

desde globals para reconstruir errores.

El contexto se pasa explícitamente.

---

# 66. N2 — Driver extraction

Cada driver implementará:

```php
interface DriverErrorExtractor
{
    public function extract(
        Throwable $error,
        DriverErrorContext $context
    ): DriverErrorDescriptor;
}
```

---

# 67. DriverErrorDescriptor

```php
final readonly class DriverErrorDescriptor
{
    public function __construct(
        public ?SqlState $sqlState,
        public ?NativeDatabaseErrorCode $nativeCode,
        public ?string $nativeMessage,
        public DriverErrorClass $driverClass,
        public array $diagnostics,
    ) {}
}
```

---

# 68. Native message

Puede conservarse internamente.

No necesariamente deberá mostrarse directamente al usuario.

---

# 69. N3/N4 structured evidence

Se debe priorizar:

```text
SQLSTATE
native structured codes
driver typed exceptions
```

sobre:

```text
regex over English text
```

---

# 70. Driver error registry

```text
Driver Error Code
      ↓
DriverErrorClassifier
      ↓
Normalized Category
```

---

# 71. Platform-specific mappings

Ejemplos conceptuales:

```text
PostgreSQL SQLSTATE mapping
MySQL error-code + SQLSTATE mapping
MariaDB error-code mapping
SQLite result/extended-code mapping
```

---

# 72. MariaDB first-class

No deberá asumirse que:

```text
MariaDB error mapping = MySQL error mapping
```

universalmente.

---

# 73. SQLite

Debe poder utilizar:

```text
primary result code
extended result code
```

cuando el driver lo exponga.

---

# 74. N5 phase classification

La fase viene principalmente del execution context.

No se adivina desde el mensaje de error.

---

# 75. N6 category classification

Se utilizará una cadena de classifiers explícitos.

---

# 76. Classifier contract

```php
interface ExecutionErrorClassifier
{
    public function classify(
        DriverErrorDescriptor $driver,
        ExecutionFailureContext $context
    ): ExecutionFailureClassification;
}
```

---

# 77. Classification result

```php
final readonly class ExecutionFailureClassification
{
    public function __construct(
        public ExecutionFailureCategory $category,
        public ExecutionFailureCode $code,
        public FailureSeverity $severity,
        public Recoverability $recoverability,
    ) {}
}
```

---

# 78. Classifier ordering

Deberá ser:

```text
deterministic
typed
conflict-free
```

---

# 79. No last-wins

Si dos classifiers reclaman exactamente el mismo error de manera incompatible:

```text
classification ambiguity
→ bootstrap/configuration failure
```

o error explícito de runtime si depende de datos runtime inevitables.

---

# 80. N7 impact classification

Podrá delegarse en:

```text
ConnectionErrorImpactClassifier
StatementErrorImpactClassifier
TransactionErrorImpactClassifier
```

---

# 81. Impact depends on phase

Ejemplo:

```text
connection lost during PREPARATION
```

vs:

```text
connection lost during COMMIT
```

tienen impactos diferentes.

---

# 82. Impact depends on driver

Un mismo SQLSTATE puede afectar distinto el transaction state según plataforma.

---

# 83. N8 outcome certainty

Se deberá utilizar evidencia:

```text
dispatch state
driver acknowledgement
transaction state
result delivery
network failure timing
```

---

# 84. No guess from exception class alone

No:

```php
if ($e instanceof ConnectionException) {
    $certainty = NOT_EXECUTED;
}
```

porque puede haberse perdido la conexión después del dispatch.

---

# 85. N9 retry metadata

Error System aportará hechos.

Ejemplo:

```text
category = DEADLOCK
retry classification = RETRYABLE_AFTER_TRANSACTION_RESTART
```

---

# 86. But

Retry System todavía deberá verificar:

```text
idempotency
deadline
partial effects
transaction ownership
retry budget
```

---

# 87. N10 redaction

Toda información deberá pasar por:

```text
ExecutionErrorRedactor
```

antes de ser emitida a:

```text
logs
telemetry
debug toolbar
user-facing exception
```

---

# 88. Sensitive sources

Pueden aparecer secretos en:

```text
SQL
parameters
driver messages
constraint values
connection DSN
credentials
result rows
stored procedure messages
```

---

# 89. SQL redaction

Idealmente usar:

```text
compiled SQL with placeholders
```

y no SQL interpolado.

---

# 90. Binding redaction

Por defecto:

```text
parameter values omitted
```

---

# 91. Driver messages

Un driver podría incluir:

```text
duplicate key value = 'secret@example.com'
```

en su mensaje.

Por ello el raw native message no deberá exponerse indiscriminadamente.

---

# 92. SafeNativeMessage

Puede existir una política para:

```text
sanitize
truncate
classify
```

mensajes nativos.

---

# 93. N11 Source Mapping

El documento 66–68 introdujo:

```text
SqlSourceMap
```

---

# 94. Error source mapping

Si el driver informa:

```text
SQL position
near token
column offset
```

VoltStack puede intentar mapearlo a:

```text
compiled SQL range
upstream expression/node
```

---

# 95. Source mapping honesty

No deberá inventar precisión.

---

# 96. SourceMapConfidence

```php
enum SourceMapConfidence
{
    case EXACT;
    case APPROXIMATE;
    case UNAVAILABLE;
}
```

---

# 97. Example

```text
Database error at SQL byte 87
↓
SqlSourceMap
↓
Query AST node Q17
```

si existe mapping válido.

---

# 98. No SQL reparsing

El Error System no deberá volver a parsear SQL completo para reconstruir el AST.

Debe utilizar metadata producida durante compilation.

---

# 99. N12 Framework exception

Podrá construirse:

```php
final class DatabaseExecutionException extends RuntimeException
{
    public function __construct(
        public readonly ExecutionFailure $failure,
        Throwable $previous,
    ) {
        parent::__construct(
            message: $failure->context->safeMessage(),
            previous: $previous,
        );
    }
}
```

---

# 100. Exception hierarchy

Propuesta:

```text
DatabaseExecutionException
├── ConnectionExecutionException
│   ├── ConnectionAcquisitionException
│   ├── ConnectionLostException
│   └── ConnectionProtocolException
│
├── StatementExecutionException
│   ├── StatementPreparationException
│   ├── StatementBindingException
│   └── StatementDispatchException
│
├── ConstraintViolationException
│   ├── UniqueConstraintViolationException
│   ├── ForeignKeyConstraintViolationException
│   ├── NotNullConstraintViolationException
│   └── CheckConstraintViolationException
│
├── ConcurrencyExecutionException
│   ├── DeadlockException
│   ├── SerializationFailureException
│   └── LockTimeoutException
│
├── QueryTerminationException
│   ├── QueryTimeoutException
│   └── QueryCancellationException
│
├── ResultExecutionException
│   ├── ResultShapeException
│   ├── ResultConversionException
│   ├── CursorExecutionException
│   └── StreamingExecutionException
│
├── TransactionExecutionException
│   ├── TransactionAbortedException
│   └── CommitOutcomeUnknownException
│
├── ResourceExecutionException
├── DriverExecutionException
├── SecurityExecutionException
├── ExtensionExecutionException
└── ExecutionInvariantException
```

---

# 101. Typed convenience without hierarchy explosion

La jerarquía deberá ser útil, no infinita.

Los detalles finos podrán vivir en:

```text
ExecutionFailure.category
ExecutionFailure.code
metadata
```

sin crear una clase PHP para cada código posible.

---

# 102. Primary vs suppressed errors

Debe existir soporte explícito.

---

# 103. Example

```text
Primary:
DeadlockException

Cleanup:
CursorCloseException
ConnectionResetException
```

---

# 104. SuppressedFailureSet

```php
final readonly class SuppressedFailureSet
{
    /**
     * @param list<ExecutionFailure> $failures
     */
    public function __construct(
        public array $failures,
    ) {}
}
```

---

# 105. Primary failure invariant

```text
cleanup failure
```

no sustituirá:

```text
primary execution failure
```

salvo que no exista primary failure.

---

# 106. Cleanup-only error

Si la ejecución fue exitosa pero cleanup falla:

```text
success of DB operation
+
resource cleanup failure
```

debe representarse de forma explícita.

---

# 107. Success + cleanup failure

No debería reducirse automáticamente a:

```text
operation never happened
```

Puede existir:

```text
ExecutionOutcomeStatus = FAILURE
OperationEffect = SUCCESSFUL
Cleanup = FAILED
```

o un modelo equivalente.

---

# 108. Error during streaming

Streaming añade una sutileza:

```text
rows already delivered
```

---

# 109. Streaming failure model

```php
final readonly class StreamingFailureMetadata
{
    public function __construct(
        public int $rowsDelivered,
        public int $bytesDelivered,
        public bool $fullyConsumed,
        public StreamTerminationReason $terminationReason,
    ) {}
}
```

---

# 110. Partial delivery must survive exception

Un error a mitad del stream deberá conservar:

```text
rowsDelivered > 0
```

---

# 111. Result cursor error

También deberá conservar:

```text
cursor position
rows consumed
result state
```

cuando sea seguro.

---

# 112. Transaction errors

Debe distinguirse:

```text
BEGIN failure
COMMIT failure
ROLLBACK failure
SAVEPOINT failure
statement failure inside transaction
```

---

# 113. Commit unknown

Caso especial:

```text
COMMIT sent
connection lost
```

deberá producir:

```text
CommitOutcomeUnknownException
```

o equivalente estructurado.

---

# 114. Commit unknown retry

Nunca deberá producir:

```text
blind transaction retry
```

automático.

---

# 115. Rollback failure

Puede indicar:

```text
transaction state unknown
connection unsafe
```

---

# 116. Connection lost

Debe distinguirse:

```text
before dispatch
after dispatch
during result consumption
during commit
during cleanup
```

---

# 117. Driver retryable flags

Si el driver ofrece una señal de retryability, ésta se conserva como evidencia.

Pero:

```text
driver says retryable
≠
framework safe-to-retry
```

---

# 118. Error provenance

Debe existir:

```php
enum FailureOrigin: string
{
    case FRAMEWORK = 'framework';
    case DRIVER = 'driver';
    case DATABASE_SERVER = 'database_server';
    case NETWORK = 'network';
    case APPLICATION = 'application';
    case EXTENSION = 'extension';
    case RUNTIME = 'runtime';
    case UNKNOWN = 'unknown';
}
```

---

# 119. Origin ≠ category

Ejemplo:

```text
category = QUERY_TIMEOUT
origin = DATABASE_SERVER
```

o:

```text
category = QUERY_TIMEOUT
origin = FRAMEWORK
```

---

# 120. Error context immutability

Una vez creado:

```text
ExecutionFailure
```

será immutable.

---

# 121. Mutable lifecycle state

El runtime puede continuar cambiando durante cleanup.

Por tanto, podrá existir:

```text
InitialFailure
+
FinalImpactReport
```

---

# 122. Final impact report

```php
final readonly class ExecutionFailureReport
{
    public function __construct(
        public ExecutionFailure $primary,
        public SuppressedFailureSet $suppressed,
        public FinalResourceImpact $resources,
        public FinalTransactionImpact $transaction,
    ) {}
}
```

---

# 123. Why

El error puede ocurrir primero y la clasificación final de recursos sólo conocerse después de cleanup/reset.

---

# 124. Exception message strategy

User-facing message:

```text
Unique constraint violation.
```

Debug message:

```text
Unique constraint violation on constraint users_email_unique.
```

Internal diagnostics:

```text
SQLSTATE, vendor code, native sanitized message, source map, phase...
```

---

# 125. Message layers

Se recomienda:

```text
PublicMessage
DeveloperMessage
InternalDiagnostic
NativeMessage
```

---

# 126. No native-message-as-public-default

Evitar:

```text
throw driver native message directly to application UI
```

---

# 127. Localization

Los códigos internos deberán ser independientes del idioma.

Los mensajes podrán localizarse posteriormente en DX/public API.

---

# 128. Error fingerprint

Puede existir:

```text
ExecutionFailureFingerprint
```

para agrupar errores de forma segura.

---

# 129. Fingerprint inputs

Podrá incluir:

```text
normalized category
failure code
phase
platform
driver
command fingerprint
constraint metadata category
```

---

# 130. Fingerprint excludes

No deberá incluir:

```text
parameter values
row values
credentials
random runtime IDs
full raw native message
```

---

# 131. Telemetry correlation

Podrá correlacionarse con:

```text
ExecutionId
StatementExecutionId
QueryFingerprint
CompiledCommandFingerprint
```

en traces/logs, no necesariamente como metric labels.

---

# 132. N13 Telemetry

Eventos sugeridos:

```text
ExecutionFailureCaptured
ExecutionFailureClassified
ConstraintViolationDetected
DeadlockDetected
SerializationFailureDetected
ConnectionLostDetected
CommitOutcomeUnknownDetected
ExecutionFailureRedacted
ExecutionFailureReported
CleanupFailureSuppressed
```

---

# 133. Metrics

Ejemplos:

```text
db.execution.errors
db.execution.connection_errors
db.execution.constraint_errors
db.execution.deadlocks
db.execution.serialization_failures
db.execution.timeouts
db.execution.cancellations
db.execution.result_errors
db.execution.unknown_outcomes
db.execution.cleanup_errors
```

---

# 134. Metric labels

Permitidos:

```text
category
phase
platform
driver
severity
retry_classification
```

con cardinalidad controlada.

---

# 135. Forbidden metric labels

No:

```text
SQL raw
constraint value
email
tenant ID
ExecutionId
connection DSN
native message
```

---

# 136. Telemetry does not mutate errors

Telemetry será:

```text
observer
```

no:

```text
classifier authority
```

---

# 137. Logging

Structured logging preferable:

```json
{
  "category": "deadlock",
  "phase": "execution",
  "sqlstate": "40001",
  "retry": "retryable_after_transaction_restart",
  "connection_impact": "none",
  "transaction_impact": "rollback_required"
}
```

sin valores sensibles.

---

# 138. Security model

El Error System debe asumir que cualquier fuente de error puede contener información sensible.

---

# 139. Sensitive sources

```text
SQL literals
bindings
driver messages
constraint values
DSNs
connection usernames
file paths
server hostnames
stored procedure output
database object names
tenant identifiers
```

---

# 140. Redaction policy

```php
interface ExecutionErrorRedactor
{
    public function redact(
        ExecutionFailureDiagnostic $diagnostic,
        ErrorDisclosurePolicy $policy
    ): RedactedExecutionFailureDiagnostic;
}
```

---

# 141. Disclosure policies

```text
PRODUCTION
DEVELOPMENT
TESTING
ADMIN_DIAGNOSTIC
```

---

# 142. Production

Debe minimizar:

```text
schema details
server details
raw messages
SQL fragments
```

---

# 143. Development

Puede incluir más contexto, manteniendo secretos redacted.

---

# 144. Testing

Puede habilitar deterministic detailed diagnostics.

---

# 145. Admin diagnostic

Puede incluir información ampliada bajo permiso explícito.

---

# 146. No secret mode

No existirá un modo donde:

```text
"debug=true"
→ log all passwords
```

---

# 147. Error extension system

Drivers y extensiones podrán aportar classifiers.

---

# 148. Extension contracts

```php
interface ExecutionErrorExtension
{
    public function descriptor(): ExecutionErrorExtensionDescriptor;

    public function classifiers(): iterable;

    public function enrichers(): iterable;
}
```

---

# 149. Classifier vs enricher

```text
Classifier
=
portable meaning
```

```text
Enricher
=
additional metadata
```

---

# 150. Enricher cannot change core classification arbitrarily

Core classification override deberá estar restringido.

---

# 151. Registry composition

```text
bootstrap
↓
discover
↓
validate
↓
dependency resolution
↓
conflict detection
↓
deterministic ordering
↓
freeze
```

---

# 152. No runtime mutable error map

No:

```php
$request->registerErrorCode(...)
```

globalmente.

---

# 153. Persistent runtime

Los siguientes pueden compartirse:

```text
immutable classification maps
frozen registries
stateless extractors
stateless redactors
```

---

# 154. Operation-local data

Debe ser local:

```text
ExecutionFailureContext
native Throwable
SQLSTATE
partial progress
connection impact
transaction impact
suppressed failures
```

---

# 155. No last error singleton

Nunca:

```php
DatabaseError::last();
```

como state global compartido.

---

# 156. FrankenPHP safety

Request A:

```text
failure context A
```

no deberá quedar visible para:

```text
Request B
```

---

# 157. RoadRunner/OpenSwoole safety

Misma regla para:

```text
workers
coroutines
parallel execution contexts
```

---

# 158. Concurrency

Dos errores concurrentes deberán mantener:

```text
separate failure contexts
```

---

# 159. Error ordering

En operaciones paralelas puede haber múltiples fallos.

Debe existir política para seleccionar:

```text
primary failure
```

---

# 160. Primary failure selection

Preferiblemente según:

```text
causal relevance
execution dependency
terminal transition winner
explicit priority
```

no simplemente:

```text
first thread to log
```

---

# 161. AggregateExecutionFailure

Puede existir:

```php
final readonly class AggregateExecutionFailure
{
    public function __construct(
        public ExecutionFailure $primary,
        public array $secondary,
    ) {}
}
```

---

# 162. Fan-out execution

Ejemplo:

```text
Unit A fails
Unit B independently fails
Unit C cleanup fails
```

debe poder representarse sin perder errores.

---

# 163. Dependency-caused failures

Si B no ejecutó porque A falló:

```text
B = BLOCKED
```

no necesariamente:

```text
B = independent execution failure
```

---

# 164. Error causality graph

Puede existir:

```text
Failure A
   ↓ causes
Blocked B
   ↓ triggers
Cleanup C
```

---

# 165. No error multiplication

No crear múltiples “errores” artificiales para cada nodo bloqueado.

---

# 166. Public API

La API puede permitir:

```php
try {
    $database->execute($query);
} catch (UniqueConstraintViolationException $e) {
    // ...
}
```

---

# 167. Portable catches

También:

```php
catch (ConstraintViolationException $e) {
}
```

---

# 168. Generic catch

```php
catch (DatabaseExecutionException $e) {
}
```

---

# 169. Native access

Expert users podrán inspeccionar:

```php
$e->failure->sqlState;
$e->failure->nativeCode;
$e->getPrevious();
```

sin hacer esos detalles obligatorios para código portable.

---

# 170. ORM boundary

ORM podrá traducir ciertos errores a excepciones de dominio/persistence más específicas.

Pero:

```text
Execution Error System
```

no deberá depender del ORM.

---

# 171. Example

```text
UniqueConstraintViolationException
```

puede posteriormente convertirse en:

```text
EntityPersistenceException
```

en una capa ORM.

---

# 172. Schema/Migration boundary

Migraciones podrán utilizar el mismo error model.

---

# 173. Raw database API boundary

La API low-level también utilizará la misma taxonomía.

---

# 174. Testing strategy

Debe cubrir:

```text
connection acquisition failure
prepare failure
binding failure
syntax/server error
unique constraint
foreign key violation
not-null violation
check violation
deadlock
serialization failure
lock timeout
query timeout
cancellation
connection loss pre-dispatch
connection loss post-dispatch
connection loss during fetch
stream failure after partial delivery
commit unknown
rollback failure
cleanup failure
multiple concurrent failures
redaction
source mapping
persistent worker isolation
```

---

# 175. Driver conformance

Cada driver oficial deberá demostrar:

```text
SQLSTATE extraction
native error-code extraction
constraint classification
deadlock classification
serialization classification
connection-failure classification
timeout classification
cancellation classification
```

cuando la plataforma lo soporte.

---

# 176. MySQL conformance

Deberá probar:

```text
SQLSTATE
numeric native codes
constraint errors
deadlocks
lock wait timeout
connection errors
```

sin hard-codear lógica fuera del adapter.

---

# 177. MariaDB conformance

Tendrá su propia suite/mapping cuando existan diferencias.

---

# 178. PostgreSQL conformance

Especial atención a:

```text
SQLSTATE classes
constraint names
serialization failures
transaction-aborted behavior
deadlocks
statement cancellation
```

---

# 179. SQLite conformance

Especial atención a:

```text
primary result codes
extended result codes
constraint subtypes
busy/locked conditions
schema errors
```

---

# 180. Failure injection

La suite deberá poder simular:

```text
network lost after dispatch
network lost before response
driver throws during cleanup
cursor fails at row N
commit acknowledgement lost
cancellation race
timeout race
```

---

# 181. Deterministic tests

Classification deberá ser determinista:

```text
same descriptor
+
same context
+
same classifier registry
=
same normalized failure
```

---

# 182. Error parser testing

Si existen message parsers de fallback:

```text
locale changes
message variations
version changes
```

deberán demostrar que nunca son usados como única fuente para invariantes críticas.

---

# 183. Performance

Error handling no es el hot path de éxito, pero deberá evitar trabajo excesivo.

---

# 184. Lazy diagnostics

Metadata costosa puramente diagnóstica podrá resolverse lazy si:

```text
safe
no hidden DB I/O
no state mutation
```

---

# 185. No hidden diagnostic query

Nunca:

```text
error occurred
↓
query database schema for diagnostics
```

implícitamente dentro del error normalizer.

---

# 186. Error budget

Puede existir:

```text
ExecutionDiagnosticBudget
```

para limitar:

```text
native message length
source map detail
stack metadata
suppressed failure count
diagnostic payload size
```

---

# 187. Budget must not affect correctness classification

Si se agota:

```text
diagnostic detail
```

puede reducirse.

Pero:

```text
category
impact
outcome certainty
```

no deberán omitirse por conveniencia.

---

# 188. Suggested namespace

```text
VoltStack\Quantum\Database\Execution\Error
```

---

# 189. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Error/
                ├── Contract/
                │   ├── ExecutionErrorClassifier.php
                │   ├── DriverErrorExtractor.php
                │   ├── ExecutionErrorRedactor.php
                │   ├── ErrorImpactClassifier.php
                │   └── ExecutionErrorExtension.php
                │
                ├── Failure/
                │   ├── ExecutionFailure.php
                │   ├── ExecutionFailureReport.php
                │   ├── ExecutionFailureCategory.php
                │   ├── ExecutionFailureCode.php
                │   ├── FailureSeverity.php
                │   ├── Recoverability.php
                │   ├── FailureOrigin.php
                │   └── ExecutionFailureFingerprint.php
                │
                ├── Context/
                │   ├── ExecutionFailureContext.php
                │   ├── ExecutionPhase.php
                │   └── DispatchEvidence.php
                │
                ├── Driver/
                │   ├── DriverErrorDescriptor.php
                │   ├── DriverErrorClass.php
                │   ├── DriverErrorContext.php
                │   ├── SqlState.php
                │   └── NativeDatabaseErrorCode.php
                │
                ├── Classification/
                │   ├── DefaultExecutionErrorClassifier.php
                │   ├── ExecutionFailureClassification.php
                │   ├── RetryClassification.php
                │   └── OutcomeCertainty.php
                │
                ├── Constraint/
                │   ├── ConstraintViolationMetadata.php
                │   ├── ConstraintViolationKind.php
                │   └── ConstraintErrorClassifier.php
                │
                ├── Impact/
                │   ├── ConnectionImpact.php
                │   ├── StatementImpact.php
                │   ├── TransactionImpact.php
                │   └── FinalResourceImpact.php
                │
                ├── Partial/
                │   ├── PartialEffectMetadata.php
                │   └── StreamingFailureMetadata.php
                │
                ├── SourceMap/
                │   ├── ErrorSourceMapper.php
                │   ├── ErrorSourceLocation.php
                │   └── SourceMapConfidence.php
                │
                ├── Redaction/
                │   ├── DefaultExecutionErrorRedactor.php
                │   ├── ErrorDisclosurePolicy.php
                │   └── RedactedExecutionFailureDiagnostic.php
                │
                ├── Diagnostic/
                │   ├── ExecutionFailureDiagnostic.php
                │   ├── PublicErrorMessage.php
                │   ├── DeveloperErrorMessage.php
                │   └── InternalErrorDiagnostic.php
                │
                ├── Aggregate/
                │   ├── SuppressedFailureSet.php
                │   └── AggregateExecutionFailure.php
                │
                ├── Extension/
                │   ├── ExecutionErrorExtensionDescriptor.php
                │   └── ExecutionErrorExtensionRegistry.php
                │
                ├── Telemetry/
                │   ├── ExecutionErrorTelemetry.php
                │   ├── ExecutionErrorMetrics.php
                │   └── ExecutionErrorObserver.php
                │
                └── Exception/
                    ├── DatabaseExecutionException.php
                    ├── ConnectionExecutionException.php
                    ├── ConnectionAcquisitionException.php
                    ├── ConnectionLostException.php
                    ├── StatementExecutionException.php
                    ├── StatementPreparationException.php
                    ├── StatementBindingException.php
                    ├── StatementDispatchException.php
                    ├── ConstraintViolationException.php
                    ├── UniqueConstraintViolationException.php
                    ├── ForeignKeyConstraintViolationException.php
                    ├── NotNullConstraintViolationException.php
                    ├── CheckConstraintViolationException.php
                    ├── ConcurrencyExecutionException.php
                    ├── DeadlockException.php
                    ├── SerializationFailureException.php
                    ├── LockTimeoutException.php
                    ├── QueryTimeoutException.php
                    ├── QueryCancellationException.php
                    ├── ResultExecutionException.php
                    ├── CursorExecutionException.php
                    ├── StreamingExecutionException.php
                    ├── TransactionExecutionException.php
                    ├── TransactionAbortedException.php
                    ├── CommitOutcomeUnknownException.php
                    ├── ResourceExecutionException.php
                    ├── DriverExecutionException.php
                    ├── SecurityExecutionException.php
                    ├── ExtensionExecutionException.php
                    └── ExecutionInvariantException.php
```

---

# 190. Dependency direction

```text
Driver
   ↓
Driver Error Extractor
   ↓
Execution Error System
   ↓
Execution Engine
```

También:

```text
SqlSourceMap
   ↓
Error Source Mapping
```

y:

```text
Execution Error System
   ↓
Telemetry
```

como observación.

---

# 191. Forbidden dependencies

No:

```text
Execution Error System
→ ORM EntityManager
→ UnitOfWork
→ Repository
→ Query Optimizer
→ Query Planner
→ HTTP Controller
```

---

# 192. Architectural invariants

## DB-ERROR-001

Execution error normalization conservará la causa nativa.

## DB-ERROR-002

VoltStack failure code será distinto de SQLSTATE.

## DB-ERROR-003

SQLSTATE será distinto de native vendor code.

## DB-ERROR-004

La categoría portable no eliminará los códigos nativos disponibles.

## DB-ERROR-005

ExecutionFailure será structured data.

## DB-ERROR-006

ExecutionException será una representación Throwable del failure.

## DB-ERROR-007

ExecutionFailure será immutable.

## DB-ERROR-008

Execution phase será explícita.

## DB-ERROR-009

Phase no se inferirá de mensajes humanos cuando exista execution context.

## DB-ERROR-010

Dispatch evidence será explícita.

## DB-ERROR-011

Outcome certainty será explícita.

## DB-ERROR-012

UNKNOWN será un outcome válido.

## DB-ERROR-013

Connection loss no implicará automáticamente NOT_EXECUTED.

## DB-ERROR-014

Partial effects serán representables.

## DB-ERROR-015

Rows already delivered serán representables.

## DB-ERROR-016

ConnectionImpact será distinto de category.

## DB-ERROR-017

StatementImpact será distinto de category.

## DB-ERROR-018

TransactionImpact será distinto de category.

## DB-ERROR-019

Severity será distinta de retryability.

## DB-ERROR-020

Recoverability será distinta de retryability.

## DB-ERROR-021

Error System clasificará retry metadata pero no ejecutará retries.

## DB-ERROR-022

Constraint violations serán portable-classified.

## DB-ERROR-023

Constraint names sólo se reportarán si existe evidencia.

## DB-ERROR-024

Constraint columns sólo se reportarán si existe evidencia.

## DB-ERROR-025

Human message parsing será fallback, no primary correctness source.

## DB-ERROR-026

Deadlock será distinto de serialization failure.

## DB-ERROR-027

Deadlock será distinto de lock timeout.

## DB-ERROR-028

Serialization failure podrá requerir transaction restart.

## DB-ERROR-029

Timeout será distinto de cancellation.

## DB-ERROR-030

Cancellation atribuida requerirá evidencia correlacionada.

## DB-ERROR-031

Result errors serán distintos de driver protocol errors.

## DB-ERROR-032

Resource exhaustion será distinto de constraint violation.

## DB-ERROR-033

Invariant errors serán first-class.

## DB-ERROR-034

Extension errors conservarán extension identity.

## DB-ERROR-035

Normalization pipeline será deterministic.

## DB-ERROR-036

Native error se capturará antes de perder información durante cleanup.

## DB-ERROR-037

Execution failure context será explícito.

## DB-ERROR-038

No se reconstruirá contexto desde globals.

## DB-ERROR-039

Driver error extraction será adapter-driven.

## DB-ERROR-040

SQLSTATE extraction será driver-aware.

## DB-ERROR-041

Vendor code extraction será driver-aware.

## DB-ERROR-042

MariaDB tendrá mappings propios cuando difieran de MySQL.

## DB-ERROR-043

SQLite extended codes serán soportables.

## DB-ERROR-044

Classifier registry será frozen tras bootstrap.

## DB-ERROR-045

Classifier conflicts no usarán last-wins.

## DB-ERROR-046

Impact classification considerará execution phase.

## DB-ERROR-047

Impact classification considerará driver/platform behavior.

## DB-ERROR-048

Outcome certainty no se inferirá sólo de exception class.

## DB-ERROR-049

Retry metadata será evidencia, no decisión.

## DB-ERROR-050

Redaction ocurrirá antes de telemetry/public disclosure.

## DB-ERROR-051

Bindings sensibles no aparecerán por defecto en error diagnostics.

## DB-ERROR-052

Raw driver messages no se expondrán indiscriminadamente.

## DB-ERROR-053

SQL interpolado no será necesario para diagnostics.

## DB-ERROR-054

SqlSourceMap será utilizado cuando exista.

## DB-ERROR-055

Error System no reparsará SQL para reconstruir AST.

## DB-ERROR-056

Source mapping no inventará precisión.

## DB-ERROR-057

Framework exception conservará previous Throwable.

## DB-ERROR-058

Exception hierarchy no sustituirá structured failure metadata.

## DB-ERROR-059

Cleanup errors podrán ser suppressed.

## DB-ERROR-060

Cleanup failure no reemplazará primary failure.

## DB-ERROR-061

Success plus cleanup failure será representable.

## DB-ERROR-062

Streaming failure conservará rowsDelivered.

## DB-ERROR-063

Partial streaming failure no fingirá atomicity.

## DB-ERROR-064

Transaction begin/commit/rollback failures serán distinguibles.

## DB-ERROR-065

Commit outcome unknown será first-class.

## DB-ERROR-066

Commit unknown no desencadenará blind retry.

## DB-ERROR-067

Rollback failure podrá afectar connection safety.

## DB-ERROR-068

Driver retryable flag no equivaldrá a safe framework retry.

## DB-ERROR-069

FailureOrigin será distinto de failure category.

## DB-ERROR-070

Final impact podrá conocerse después del primary failure.

## DB-ERROR-071

Public, developer e internal messages podrán diferir.

## DB-ERROR-072

Framework codes serán locale-independent.

## DB-ERROR-073

Failure fingerprints excluirán sensitive runtime values.

## DB-ERROR-074

Failure fingerprints excluirán random runtime IDs cuando no sean necesarios.

## DB-ERROR-075

Telemetry será observational.

## DB-ERROR-076

Metrics no utilizarán sensitive values como labels.

## DB-ERROR-077

Native messages no serán metric labels.

## DB-ERROR-078

Error redaction tendrá políticas explícitas.

## DB-ERROR-079

Debug mode no deshabilitará secret redaction.

## DB-ERROR-080

Administrative diagnostics requerirán explicit policy.

## DB-ERROR-081

Error extensions serán tipadas.

## DB-ERROR-082

Extension registry será frozen.

## DB-ERROR-083

Runtime requests no registrarán global classifiers.

## DB-ERROR-084

Core classifications no serán reemplazadas silenciosamente por extensions.

## DB-ERROR-085

Shared error services serán stateless/immutable.

## DB-ERROR-086

Operation failure context será operation-scoped.

## DB-ERROR-087

No existirá global last database error.

## DB-ERROR-088

Persistent workers no compartirán last failure state.

## DB-ERROR-089

Concurrent executions tendrán failure contexts separados.

## DB-ERROR-090

Multiple concurrent failures podrán agregarse.

## DB-ERROR-091

Primary failure selection será deterministic/causal.

## DB-ERROR-092

Blocked units no se contarán como independent execution failures automáticamente.

## DB-ERROR-093

Error causality podrá representarse.

## DB-ERROR-094

Public API podrá capturar portable exception classes.

## DB-ERROR-095

Advanced API podrá acceder a SQLSTATE/native codes.

## DB-ERROR-096

ORM podrá traducir errors sin que Execution Error System dependa del ORM.

## DB-ERROR-097

Schema/Migration podrán reutilizar la misma taxonomía.

## DB-ERROR-098

Raw Database API utilizará la misma taxonomía base.

## DB-ERROR-099

Driver conformance deberá probar error extraction.

## DB-ERROR-100

Failure injection deberá cubrir post-dispatch connection loss.

## DB-ERROR-101

Failure injection deberá cubrir commit acknowledgement loss.

## DB-ERROR-102

Classification será deterministic.

## DB-ERROR-103

Diagnostic enrichment no ejecutará hidden DB queries.

## DB-ERROR-104

Diagnostic budgets no afectarán correctness classification.

## DB-ERROR-105

Native cause será preservada incluso con redaction.

## DB-ERROR-106

Sensitive diagnostic fields podrán omitirse sin perder error category.

## DB-ERROR-107

Connection safety unknown será tratada conservadoramente.

## DB-ERROR-108

Transaction outcome unknown será tratada conservadoramente.

## DB-ERROR-109

Statement safety unknown será tratada conservadoramente.

## DB-ERROR-110

Unique violation no se inferirá sólo por texto humano si existe structured code.

## DB-ERROR-111

Foreign-key violation no se inferirá sólo por texto humano si existe structured code.

## DB-ERROR-112

Deadlock mapping será platform-aware.

## DB-ERROR-113

Serialization mapping será platform-aware.

## DB-ERROR-114

Timeout mapping será platform-aware.

## DB-ERROR-115

Cancellation mapping será platform-aware.

## DB-ERROR-116

Result-conversion error no será clasificado como DB server error si nació en framework.

## DB-ERROR-117

Application consumer failure en streaming será distinto de driver failure.

## DB-ERROR-118

Failure origin será preservado.

## DB-ERROR-119

Error System no realizará transaction rollback por sí mismo.

## DB-ERROR-120

Error System no realizará connection reconnect por sí mismo.

## DB-ERROR-121

Error System no realizará query retry por sí mismo.

## DB-ERROR-122

Error System no alterará ExecutionPlan.

## DB-ERROR-123

Error System no generará SQL.

## DB-ERROR-124

Error System no realizará ORM hydration.

## DB-ERROR-125

Error System no realizará authorization.

## DB-ERROR-126

Error System no ocultará ambiguous outcomes.

## DB-ERROR-127

Error System no afirmará effects que no pueda probar.

## DB-ERROR-128

Error System no afirmará constraint metadata que no pueda probar.

## DB-ERROR-129

Error System no afirmará server-side cancellation que no pueda probar.

## DB-ERROR-130

Execution Error System será la taxonomía común de fallos runtime de Database.

---

# 193. Invariante maestro de normalización

```text
NormalizedFailure
=
PortableMeaning
+
NativeEvidence
+
ExecutionContext
+
ImpactClassification
+
OutcomeCertainty
```

---

# 194. Invariante maestro de preservación

```text
Normalize(NativeError)
```

no deberá provocar:

```text
Loss(
    SQLSTATE
    ∨ NativeCode
    ∨ NativeCause
    ∨ RelevantContext
)
```

cuando esos datos existan y puedan conservarse de forma segura.

---

# 195. Invariante maestro de outcome

```text
InsufficientEvidence
⇒
OutcomeCertainty = UNKNOWN
```

---

# 196. Invariante maestro de impacto

```text
FailureCategory
```

no determina por sí sola:

```text
ConnectionImpact
TransactionImpact
StatementImpact
```

---

# 197. Invariante maestro de retry

```text
RetryClassification
≠
RetryDecision
```

---

# 198. Invariante maestro de seguridad

```text
PublicDiagnostic
∩
SecretData
=
∅
```

por defecto.

---

# 199. Invariante maestro de suppressed errors

```text
PrimaryFailure
```

deberá permanecer distinguible de:

```text
CleanupFailures
SecondaryFailures
```

---

# 200. Invariante maestro de streaming

```text
RowsDelivered > 0
∧
StreamFailure
⇒
PartialDelivery = true
```

---

# 201. Invariante maestro de commit ambiguity

```text
CommitSent
∧
AcknowledgementLost
⇒
CommitOutcome may be UNKNOWN
```

---

# 202. Ejemplo — Unique violation

Driver:

```text
SQLSTATE 23505
```

Context:

```text
phase = EXECUTION
statement dispatched = true
```

Normalization:

```text
category = UNIQUE_CONSTRAINT
code = DB_EXEC_UNIQUE_VIOLATION
severity = OPERATION_FATAL
connectionImpact = NONE
statementImpact = REUSABLE
transactionImpact = platform-dependent
retry = NEVER / application-level decision
```

Excepción:

```text
UniqueConstraintViolationException
```

---

# 203. Ejemplo — Deadlock

Driver:

```text
deadlock detected
```

Normalization:

```text
category = DEADLOCK
transactionImpact = ROLLBACK_REQUIRED
connectionImpact = NONE
retry = RETRYABLE_AFTER_TRANSACTION_RESTART
```

Pero el Error System no ejecuta el retry.

---

# 204. Ejemplo — Connection lost before dispatch

```text
connection acquire succeeds
prepare succeeds
network drops
before execute()
```

Normalization:

```text
category = CONNECTION_LOST
dispatch = NOT_STARTED
certainty = NOT_EXECUTED
connectionImpact = LOST
```

---

# 205. Ejemplo — Connection lost after dispatch

```text
UPDATE dispatched
network drops
no response
```

Normalization:

```text
category = CONNECTION_LOST
dispatch = STARTED
certainty = UNKNOWN
connectionImpact = LOST
retry = UNKNOWN / unsafe
```

---

# 206. Ejemplo — Streaming error

```text
rows delivered = 500
network failure during fetch
```

Normalization:

```text
category = CURSOR / CONNECTION_LOST
rowsDelivered = 500
partialEffects = true
fullyConsumed = false
connectionImpact = LOST
retry = UNSAFE_AFTER_DELIVERY
```

---

# 207. Ejemplo — Commit unknown

```text
COMMIT sent
socket closes
```

Normalization:

```text
category = COMMIT_UNKNOWN
certainty = UNKNOWN
transactionImpact = COMMIT_OUTCOME_UNKNOWN
connectionImpact = LOST
```

Nunca:

```text
assume rollback
```

---

# 208. Anti-patterns

## Anti-pattern 1 — Generic exception

```php
catch (\Throwable $e) {
    throw new DatabaseException('Query failed');
}
```

---

## Anti-pattern 2 — Throw raw driver exception everywhere

```php
throw $pdoException;
```

como API pública principal.

---

## Anti-pattern 3 — Parse messages only

```php
if (str_contains($e->getMessage(), 'duplicate')) {
    // unique violation
}
```

como mecanismo principal.

---

## Anti-pattern 4 — Assume connection lost means no execution

```text
ConnectionException
→ NOT_EXECUTED
```

---

## Anti-pattern 5 — Assume timeout means rollback

```text
TimeoutException
→ transaction rolled back
```

---

## Anti-pattern 6 — Hide SQLSTATE

```text
normalize
→ discard native codes
```

---

## Anti-pattern 7 — Log bindings in exception

```php
$logger->error($sql, $bindings);
```

sin redaction.

---

## Anti-pattern 8 — Let cleanup overwrite primary error

```text
query fails
cursor close fails
→ only cursor close reported
```

---

## Anti-pattern 9 — Retry inside exception handler

```php
catch (DeadlockException $e) {
    return $this->executeAgain();
}
```

dentro del Error System.

---

## Anti-pattern 10 — Global last error

```php
Database::$lastError = $e;
```

---

# 209. Arquitectura final

```text
                     EXECUTION ERROR SYSTEM

                         Throwable
                            │
                            ▼
                  DriverErrorExtractor
                            │
                            ▼
                  DriverErrorDescriptor
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          SQLSTATE     Native Code    Native Cause
              └─────────────┼─────────────┘
                            ▼
                 ExecutionErrorClassifier
                            │
                            ▼
                 Normalized Failure Meaning
                            │
       ┌────────────────────┼─────────────────────┐
       ▼                    ▼                     ▼
 Connection Impact   Transaction Impact    Statement Impact
       │                    │                     │
       └────────────────────┼─────────────────────┘
                            ▼
                   Outcome Certainty
                            │
                            ▼
                   Retry Classification
                            │
                            ▼
                    Error Redaction
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
         Exception      Telemetry      Diagnostics
```

---

# 210. Fórmula arquitectónica

```text
Execution Error System
=
Native Error Capture
+
Portable Error Classification
+
SQLSTATE Preservation
+
Native Code Preservation
+
Execution Phase
+
Outcome Certainty
+
Partial Effect Metadata
+
Connection Impact
+
Statement Impact
+
Transaction Impact
+
Retry Classification
+
Source Mapping
+
Sensitive Data Redaction
+
Primary/Suppressed Error Modeling
+
Telemetry
+
Persistent Runtime Isolation
```

---

# 211. Fórmula de corrección

```text
CorrectErrorHandling
=
MeaningPreserved
∧
NativeEvidencePreserved
∧
NoFalseCertainty
∧
NoSensitiveLeak
∧
CorrectImpactClassification
∧
PrimaryCausePreserved
∧
RetryBoundaryPreserved
```

---

# 212. Principio final

> **VoltStack should simplify how applications understand database failures without simplifying away the evidence needed to recover safely.**

En forma resumida:

```text
Execution Error
=
Portable Meaning
+
Native Evidence
+
Runtime Context
+
Impact
+
Recovery Metadata
```

y nunca:

```text
Execution Error
=
"Database error"
```

sin contexto ni evidencia.

---

# 213. Siguiente documento

```text
86_DATABASE_EXECUTION_RETRY_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Execution Retry System
├── Retry Architecture
├── Retry Eligibility
├── Retry Decision
├── Retry Policy
├── Retry Scope
├── Statement Retry
├── Transaction Retry
├── Execution Region Retry
├── Whole Plan Retry
├── Idempotency
├── Retry Safety
├── Partial Effects
├── Unknown Outcomes
├── Deadlock Retry
├── Serialization Retry
├── Connection Failure Retry
├── Timeout Retry
├── Cancellation Boundary
├── Stream Retry
├── Replayable Bindings
├── Retry Budget
├── Retry Count
├── Backoff
├── Jitter
├── Deadline Preservation
├── Resource Reset
├── State Reset
├── Transaction Restart
├── Telemetry
├── Security
├── Persistent Runtime Safety
└── Architectural Invariants
```

manteniendo el principio:

```text
Retryable Error
≠
Safe Retry
```

y la fórmula:

```text
Safe Retry
=
Retryable Failure
∧
Known Retry Scope
∧
Provable Idempotency Or Transaction Restart Safety
∧
No Forbidden Partial Effects
∧
Replayable Inputs
∧
Valid Resource State
∧
Deadline Remaining
∧
Retry Budget Available
```