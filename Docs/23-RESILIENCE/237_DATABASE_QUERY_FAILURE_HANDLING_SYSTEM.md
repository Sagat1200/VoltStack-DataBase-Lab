# 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md

# VoltStack Quantum Database
## Database Query Failure Handling System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 237 — Database Query Failure Handling System  
**Bloque:** 23 — Resilience  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md`  
**Siguiente documento:** `238_DATABASE_RETRY_POLICY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura responsable de detectar, clasificar, normalizar, contextualizar y propagar los fallos asociados al ciclo completo de una operación de consulta en `VoltStack/Quantum/Database`.

El sistema cubrirá fallos producidos durante:

```text
Query construction
Query validation
Semantic analysis
Optimization
Planning
Compilation
Statement preparation
Parameter binding
Transmission
Database execution
Result reception
Result decoding
Hydration
Cancellation
Timeout
Transaction interaction
```

La regla fundamental será:

> **Un error observado durante una query no determina por sí mismo qué ocurrió en la base de datos, si existieron efectos persistentes, si la transacción continúa siendo válida ni si la operación puede repetirse.**

Formalmente:

```text
Observed Query Error
≠
Known Database Outcome
```

y:

```text
Query Failure
≠
Retry Permission
```

---

# 2. Objetivos

El sistema deberá permitir responder con precisión:

```text
¿Qué falló?

¿En qué fase falló?

¿La operación llegó a la base de datos?

¿La base comenzó a ejecutarla?

¿Produjo efectos?

¿Esos efectos fueron confirmados?

¿La transacción continúa siendo utilizable?

¿La conexión continúa siendo utilizable?

¿El error es transitorio?

¿La operación es replay-safe?

¿Puede reintentarse?

¿Debe escalarse?

¿El resultado es FAILURE o UNKNOWN?
```

---

# 3. Relación con otros subsistemas

Este documento especializa y coordina:

```text
23_DATABASE_QUERY_ARCHITECTURE.md
24_DATABASE_QUERY_MODEL.md
25_DATABASE_QUERY_AST_SYSTEM.md
34_DATABASE_QUERY_VALIDATION_SYSTEM.md

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md

66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
67_DATABASE_SQL_COMPILER_PIPELINE.md
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md

76_DATABASE_EXECUTION_ENGINE_ARCHITECTURE.md
77_DATABASE_QUERY_EXECUTOR_SYSTEM.md
78_DATABASE_STATEMENT_EXECUTION_SYSTEM.md
79_DATABASE_PREPARED_STATEMENT_SYSTEM.md
80_DATABASE_PARAMETER_BINDING_SYSTEM.md

81_DATABASE_RESULT_SYSTEM.md
82_DATABASE_RESULT_CURSOR_SYSTEM.md
83_DATABASE_STREAMING_RESULT_SYSTEM.md

84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
85_DATABASE_EXECUTION_ERROR_SYSTEM.md
86_DATABASE_EXECUTION_RETRY_SYSTEM.md

164_DATABASE_TRANSACTION_ARCHITECTURE.md
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md
171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md

217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
221_DATABASE_QUERY_PROFILER_SYSTEM.md
222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md

227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md

235_DATABASE_RESILIENCE_ARCHITECTURE.md
236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
```

---

# 4. Principio de separación

VoltStack distinguirá como mínimo:

```text
Query Definition Failure
Query Planning Failure
Query Compilation Failure
Statement Failure
Execution Failure
Database Semantic Failure
Result Failure
Hydration Failure
Connection Failure
Transaction Failure
Application Callback Failure
```

No deberán colapsarse automáticamente en:

```php
throw new QueryException(...);
```

sin conservar su información semántica.

---

# 5. Pipeline de consulta

El ciclo conceptual será:

```text
Developer API
    │
    ▼
Query Builder
    │
    ▼
Query Model / AST
    │
    ▼
Validation
    │
    ▼
Semantic Analysis
    │
    ▼
Optimization
    │
    ▼
Planning
    │
    ▼
Compilation
    │
    ▼
Prepared Statement
    │
    ▼
Binding
    │
    ▼
Execution
    │
    ▼
Database
    │
    ▼
Result
    │
    ▼
Hydration
    │
    ▼
Application
```

Cada frontera podrá producir errores distintos.

---

# 6. Failure boundary

Una propiedad fundamental será identificar:

```text
FailureBoundary
```

El boundary representa hasta qué punto existe evidencia de progreso de la operación.

---

# 7. QueryFailureStage

```php
enum QueryFailureStage
{
    case CONSTRUCTION;

    case VALIDATION;
    case SEMANTIC_ANALYSIS;

    case OPTIMIZATION;
    case PLANNING;

    case COMPILATION;

    case STATEMENT_PREPARATION;
    case PARAMETER_BINDING;

    case TRANSMISSION;
    case EXECUTION;

    case RESULT_RECEIVE;
    case RESULT_DECODE;

    case HYDRATION;

    case CANCELLATION;

    case UNKNOWN;
}
```

---

# 8. QueryFailureCategory

```php
enum QueryFailureCategory
{
    case INVALID_QUERY;
    case SEMANTIC_ERROR;

    case UNSUPPORTED_FEATURE;
    case COMPILATION_ERROR;

    case PARAMETER_ERROR;
    case TYPE_ERROR;

    case SYNTAX_ERROR;

    case CONSTRAINT_VIOLATION;

    case PERMISSION_DENIED;
    case AUTHENTICATION_ERROR;

    case OBJECT_NOT_FOUND;

    case DEADLOCK;
    case SERIALIZATION_FAILURE;
    case LOCK_TIMEOUT;

    case QUERY_TIMEOUT;
    case CANCELLED;

    case CONNECTION_FAILURE;

    case RESOURCE_EXHAUSTION;

    case DATA_ERROR;
    case RESULT_ERROR;
    case HYDRATION_ERROR;

    case PLATFORM_ERROR;

    case UNKNOWN;
}
```

---

# 9. QueryFailure

Objeto normalizado:

```php
final readonly class QueryFailure
{
    public function __construct(
        public QueryFailureId $id,
        public QueryFailureStage $stage,
        public QueryFailureCategory $category,
        public QueryFailureSeverity $severity,
        public QueryOutcome $outcome,
        public QueryExecutionEvidence $evidence,
        public QueryFailureScope $scope,
        public QueryRetryAssessment $retryAssessment,
        public ?Throwable $cause,
    ) {}
}
```

---

# 10. QueryOutcome

```php
enum QueryOutcome
{
    case NOT_EXECUTED;

    case SUCCEEDED;
    case FAILED;

    case PARTIALLY_OBSERVED;

    case UNKNOWN;
}
```

---

# 11. NOT_EXECUTED

Existe evidencia suficiente de que la operación no llegó a ejecutarse en la base.

Ejemplo:

```text
Invalid AST
    ↓
Validation failure
```

Resultado:

```text
QueryOutcome = NOT_EXECUTED
```

---

# 12. FAILED

Existe evidencia suficiente de que la base rechazó o abortó la operación.

Ejemplo:

```text
INSERT
   ↓
Unique constraint violation
```

La operación fue procesada pero no produjo el efecto solicitado.

---

# 13. SUCCEEDED

Solo deberá utilizarse cuando exista evidencia suficiente de éxito.

---

# 14. UNKNOWN

Caso crítico.

Ejemplo:

```text
UPDATE sent
    ↓
Database executes
    ↓
Connection lost before acknowledgement
```

VoltStack no sabe si:

```text
UPDATE happened
```

Por tanto:

```text
QueryOutcome = UNKNOWN
```

---

# 15. PARTIALLY_OBSERVED

Útil principalmente para resultados incrementales:

```text
SELECT
 ↓
500 rows received
 ↓
connection failure
```

El consumidor observó una parte del resultado.

Pero:

```text
Partial Observation
≠
Complete Query Result
```

---

# 16. Query failure ≠ query outcome

Puede existir:

```text
Application observed failure
```

mientras:

```text
Database operation succeeded
```

Ejemplo clásico:

```text
INSERT succeeds
ACK lost
```

---

# 17. Execution evidence

```php
final readonly class QueryExecutionEvidence
{
    public function __construct(
        public QuerySendState $sendState,
        public QueryExecutionState $executionState,
        public QueryResponseState $responseState,
        public TransactionEvidence $transaction,
        public ConnectionEvidence $connection,
        public Instant $observedAt,
    ) {}
}
```

---

# 18. QuerySendState

```php
enum QuerySendState
{
    case NOT_SENT;
    case POSSIBLY_SENT;
    case SENT;
    case UNKNOWN;
}
```

---

# 19. QueryExecutionState

```php
enum QueryExecutionState
{
    case NOT_STARTED;

    case POSSIBLY_STARTED;

    case STARTED;

    case COMPLETED_SUCCESSFULLY;

    case COMPLETED_WITH_ERROR;

    case UNKNOWN;
}
```

---

# 20. QueryResponseState

```php
enum QueryResponseState
{
    case NONE;

    case PARTIAL;

    case COMPLETE;

    case LOST;

    case UNKNOWN;
}
```

---

# 21. Evidence lattice

El sistema deberá evitar convertir evidencia débil en certeza fuerte.

Conceptualmente:

```text
UNKNOWN
   │
   ├── possibly sent
   │
   ├── sent
   │
   ├── execution started
   │
   └── complete response
```

Cada transición deberá basarse en evidencia real del driver/protocolo.

---

# 22. Query construction failure

Ejemplo:

```php
$query->where('', '=', 10);
```

Si viola contratos internos:

```text
stage = CONSTRUCTION
outcome = NOT_EXECUTED
```

No existe database side effect.

---

# 23. Validation failure

Ejemplo:

```text
UPDATE without required target
invalid aggregate semantics
invalid AST
invalid expression type
```

Debe detectarse antes de compilación cuando sea posible.

---

# 24. Semantic analysis failure

Ejemplos:

```text
ambiguous column
unknown relation alias
invalid grouping semantics
incompatible expression types
invalid relationship path
```

---

# 25. Semantic failure safety

Normalmente:

```text
NOT_EXECUTED
```

porque todavía no se llegó al driver.

---

# 26. Optimization failure

El optimizer no deberá producir una query inválida.

Si una optimization rule falla internamente:

```text
OptimizerFailure
```

deberá distinguirse de:

```text
InvalidUserQuery
```

---

# 27. Optimization fallback

Una optimización opcional podrá ser descartada si existe un fallback semánticamente equivalente.

Ejemplo:

```text
Optimization Rule
    ↓
Cannot prove rewrite safe
    ↓
Use original Query Model
```

Esto no será considerado query failure.

---

# 28. Unsafe optimizer fallback prohibited

Si el optimizer ya transformó el modelo y no puede demostrar equivalencia:

```text
do not execute uncertain transformed query
```

---

# 29. Planning failure

Puede ocurrir por:

```text
unsupported operation
missing capability
invalid distributed plan
unknown shard routing
resource policy rejection
```

---

# 30. Planning failure ≠ database failure

Normalmente:

```text
outcome = NOT_EXECUTED
```

---

# 31. Compilation failure

El SQL Compiler podrá fallar por:

```text
unsupported platform feature
invalid compilation state
missing dialect capability
invalid type representation
```

---

# 32. Compilation failure safety

La compilación ocurre antes del I/O.

Por tanto:

```text
outcome = NOT_EXECUTED
```

---

# 33. SQL syntax error

Si el compiler produce SQL inválido, deberá clasificarse como posible:

```text
FrameworkCompilationDefect
```

no automáticamente como error del desarrollador.

---

# 34. Raw SQL syntax error

Cuando el usuario utiliza escape hatch/raw SQL:

```text
syntax error
```

puede ser responsabilidad de la query suministrada.

---

# 35. Statement preparation failure

Puede ser:

```text
SQL syntax error
unsupported prepared statement
resource exhaustion
connection failure
server error
```

---

# 36. Prepare semantics

Debe distinguirse:

```text
client-side prepare
```

de:

```text
server-side prepare
```

---

# 37. Server-side prepare

Puede implicar I/O antes de la ejecución.

Pero normalmente no implica aún efectos DML.

---

# 38. Parameter binding failure

Ejemplos:

```text
missing parameter
duplicate parameter mismatch
unsupported PHP value
invalid database type
overflow
encoding failure
```

---

# 39. Binding failure

Cuando ocurre completamente antes de enviar:

```text
outcome = NOT_EXECUTED
```

---

# 40. Parameter security

Los valores completos de parámetros no deberán incorporarse automáticamente en:

```text
exceptions
logs
telemetry
debug toolbar
```

---

# 41. Execution failure

Una vez iniciada ejecución, el sistema deberá analizar:

```text
database response
SQLSTATE
vendor code
driver state
connection state
transaction state
operation type
```

---

# 42. SQLSTATE

SQLSTATE podrá utilizarse como fuente estructurada de clasificación.

Pero:

```text
SQLSTATE
≠
Complete Recovery Policy
```

---

# 43. Vendor codes

Los códigos específicos de:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

podrán enriquecer clasificación.

---

# 44. Platform classifier

```php
interface QueryFailureClassifier
{
    public function classify(
        QueryFailureObservation $observation
    ): QueryFailure;
}
```

---

# 45. Classifiers específicos

```text
MySqlQueryFailureClassifier
MariaDbQueryFailureClassifier
PostgreSqlQueryFailureClassifier
SqliteQueryFailureClassifier
```

---

# 46. MySQL ≠ MariaDB

Aunque compartan compatibilidad significativa:

```text
MySQL error semantics
≠
MariaDB error semantics in all cases
```

Por tanto, podrán tener classifiers separados.

---

# 47. Constraint violations

Deberán distinguirse:

```text
UNIQUE
PRIMARY KEY
FOREIGN KEY
NOT NULL
CHECK
EXCLUSION
OTHER
```

cuando la plataforma permita identificarlo.

---

# 48. ConstraintViolation

```php
final readonly class ConstraintViolation
{
    public function __construct(
        public ConstraintViolationType $type,
        public ?ConstraintName $constraint,
        public ?TableIdentity $table,
        public ?ColumnIdentity $column,
    ) {}
}
```

---

# 49. Constraint failure outcome

Normalmente:

```text
QueryOutcome = FAILED
```

no:

```text
UNKNOWN
```

si la base devolvió explícitamente la violación.

---

# 50. Transaction effect of constraint violation

No todas las plataformas manejan errores de statement igual dentro de una transaction.

Ejemplo conceptual:

```text
statement failed
```

puede dejar:

```text
transaction still usable
```

o:

```text
transaction aborted
```

según plataforma/error.

---

# 51. Transaction usability

Se modelará explícitamente:

```php
enum TransactionUsability
{
    case USABLE;
    case REQUIRES_ROLLBACK;
    case TAINTED;
    case CLOSED;
    case UNKNOWN;
}
```

---

# 52. Statement failure ≠ transaction failure

Regla:

```text
Statement Failed
≠
Transaction Automatically Dead
```

---

# 53. Transaction aborted state

Por ejemplo, determinadas plataformas pueden marcar la transaction como fallida hasta rollback.

VoltStack deberá conocerlo mediante:

```text
platform semantics
+
driver evidence
```

---

# 54. Unique constraint example

```text
INSERT user
    ↓
unique(email) violation
    ↓
statement rejected
```

Resultado:

```text
Query outcome = FAILED
Connection = HEALTHY
Transaction = platform-dependent
Retry = normally NO unless operation changes
```

---

# 55. Foreign key violation

Normalmente representa:

```text
data/domain ordering problem
```

no infraestructura transitoria.

---

# 56. NOT NULL violation

No deberá reintentarse automáticamente esperando que desaparezca.

---

# 57. CHECK violation

Normalmente:

```text
PERMANENT_FOR_SAME_INPUT
```

---

# 58. QueryRetryAssessment

```php
final readonly class QueryRetryAssessment
{
    public function __construct(
        public RetryDisposition $disposition,
        public RetryReason $reason,
        public ReplaySafety $replaySafety,
        public Confidence $confidence,
    ) {}
}
```

---

# 59. RetryDisposition

```php
enum RetryDisposition
{
    case NOT_APPLICABLE;
    case NEVER;
    case POSSIBLE;
    case RECOMMENDED_BY_POLICY;
    case REQUIRES_TRANSACTION_REPLAY;
    case REQUIRES_IDEMPOTENCY_PROOF;
    case UNKNOWN;
}
```

---

# 60. Classification ≠ retry

Ejemplo:

```text
DEADLOCK
```

puede ser transitorio.

Pero eso no significa:

```text
retry current statement blindly
```

---

# 61. Deadlock

Debe integrarse con:

```text
171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md
```

Regla:

> Un deadlock normalmente se recupera repitiendo una unidad transaccional segura, no repitiendo arbitrariamente el último statement.

---

# 62. Deadlock outcome

La base puede haber abortado:

```text
statement
transaction
```

según plataforma.

Debe determinarse.

---

# 63. Serialization failure

Ejemplo:

```text
SERIALIZABLE transaction conflict
```

Puede ser candidato a transaction replay.

---

# 64. Serialization retry

Será gobernado por:

```text
170_DATABASE_TRANSACTION_RETRY_SYSTEM
238_DATABASE_RETRY_POLICY_SYSTEM
```

---

# 65. Lock timeout

Debe distinguirse de deadlock.

```text
Deadlock
≠
Lock Timeout
```

---

# 66. Lock timeout semantics

Puede significar:

```text
statement failed
transaction usable
```

o una semántica más fuerte según plataforma/configuración.

---

# 67. Query timeout

Un query timeout puede originarse en:

```text
client
driver
server
proxy
network
resource governor
```

---

# 68. Timeout source

```php
enum QueryTimeoutSource
{
    case CLIENT;
    case DRIVER;
    case SERVER;
    case PROXY;
    case RESOURCE_POLICY;
    case UNKNOWN;
}
```

---

# 69. Timeout ≠ cancellation confirmation

Regla:

```text
Client stopped waiting
≠
Database stopped executing
```

---

# 70. Server-side timeout

Si la base confirma:

```text
statement cancelled
```

existe evidencia más fuerte.

---

# 71. Client-side timeout

Puede dejar:

```text
execution outcome = UNKNOWN
```

especialmente para writes.

---

# 72. Cancellation

Se distinguirá:

```text
Cancellation Requested
Cancellation Sent
Cancellation Confirmed
Cancellation Failed
Cancellation Unknown
```

---

# 73. QueryCancellationState

```php
enum QueryCancellationState
{
    case NOT_REQUESTED;
    case REQUESTED;
    case SENT;
    case CONFIRMED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 74. Cancellation requested ≠ cancelled

---

# 75. Cancellation confirmed

Aun así deberá analizarse si:

```text
partial effects
```

eran posibles para el tipo de statement/plataforma.

---

# 76. Read query failure

Para:

```text
SELECT
```

una repetición puede ser menos riesgosa respecto a side effects.

Pero:

```text
Retry Read
≠
Same Result
```

---

# 77. Mutable database state

Un segundo SELECT puede observar:

```text
new rows
deleted rows
updated rows
different replica state
```

---

# 78. Read consistency

Retry deberá respetar:

```text
transaction
snapshot
read-your-writes
sticky connection
replica freshness
shard
```

---

# 79. Volatile SELECT

No todo SELECT es semánticamente read-only/replay-equivalent.

Puede contener:

```text
volatile functions
locking reads
database functions with side effects
sequences
advisory locks
```

---

# 80. Read classification

La seguridad de replay deberá provenir del Query Model/metadata/capabilities, no de comprobar únicamente si SQL comienza con:

```text
SELECT
```

---

# 81. Write query failure

Para:

```text
INSERT
UPDATE
DELETE
```

la pregunta principal será:

```text
Did database apply the mutation?
```

---

# 82. Confirmed database rejection

Ejemplo:

```text
UPDATE rejected by permission
```

Outcome:

```text
FAILED
```

---

# 83. Lost acknowledgement

Ejemplo:

```text
UPDATE sent
UPDATE committed/autocommitted
response lost
```

Outcome:

```text
UNKNOWN
```

---

# 84. Unknown write

Regla central:

```text
UNKNOWN WRITE
→
NO BLIND RETRY
```

---

# 85. Idempotency

Una operación puede ser matemáticamente idempotente:

```text
SET status = 'active'
```

pero eso no significa automáticamente que:

```text
application operation
```

sea segura de repetir.

---

# 86. Idempotency levels

Debe distinguirse:

```text
Statement Idempotency
Application Operation Idempotency
Transaction Idempotency
External Side Effect Idempotency
```

---

# 87. Query layer limitations

Query Failure Handling no deberá inventar:

```text
business idempotency
```

---

# 88. INSERT retry danger

Ejemplo:

```text
INSERT order
response lost
retry INSERT
```

puede crear:

```text
duplicate order
```

---

# 89. Idempotency key integration

Capas superiores podrán usar:

```text
idempotency key
unique operation identifier
outbox
deduplication record
```

pero no será inferido automáticamente por Query Engine.

---

# 90. UPDATE retry danger

Incluso:

```sql
UPDATE counters
SET value = value + 1
```

es claramente no idempotente.

---

# 91. DELETE retry

Puede parecer idempotente, pero:

```text
triggers
audit rows
cascades
version checks
affected-row expectations
```

pueden alterar semántica.

---

# 92. DDL failure

DDL deberá tener tratamiento especial.

---

# 93. DDL transactional semantics

Varían por plataforma.

No deberá asumirse:

```text
DDL always transactional
```

ni:

```text
DDL always auto-commits
```

---

# 94. Migration layer

Las queries DDL ejecutadas dentro de migrations deberán reportar evidencia hacia:

```text
Migration Execution System
Migration Safety System
```

---

# 95. Unknown DDL outcome

Es particularmente importante porque schema real puede haber cambiado.

---

# 96. Schema invalidation

Cuando DDL se confirma:

```text
schema metadata generation
```

deberá invalidarse/actualizarse.

Cuando outcome es UNKNOWN:

```text
schema state may be uncertain
```

---

# 97. Object not found

Ejemplos:

```text
table does not exist
column does not exist
schema does not exist
```

Puede indicar:

```text
programming error
migration mismatch
wrong database
schema drift
deployment race
```

---

# 98. No automatic migration

Un query failure por tabla inexistente no deberá provocar:

```text
run migrations automatically
```

desde Query Engine.

---

# 99. Permission denied

Debe clasificarse separadamente.

---

# 100. Permission failure retry

Normalmente:

```text
same credentials
+
same authorization
+
same query
=
not transient
```

---

# 101. Credential rotation exception

Si el error proviene de credencial expirada y existe un mecanismo autorizado de refresh:

```text
Connection Security System
```

podrá intentar resolver la conexión.

Pero:

```text
Query Failure System
```

no deberá cambiar permisos.

---

# 102. Data errors

Ejemplos:

```text
numeric overflow
string too long
invalid encoding
invalid date
invalid JSON
division by zero
```

---

# 103. Data failure classification

Cuando la misma entrada producirá el mismo error:

```text
PERMANENT_FOR_SAME_INPUT
```

---

# 104. QueryFailurePersistence

```php
enum FailurePersistence
{
    case TRANSIENT;
    case PERMANENT_FOR_SAME_INPUT;
    case PERMANENT_FOR_CONFIGURATION;
    case ENVIRONMENT_DEPENDENT;
    case UNKNOWN;
}
```

---

# 105. Transient ≠ retryable

Regla:

```text
Transient Failure
≠
Safe To Retry
```

---

# 106. Permanent ≠ unrecoverable application

Un error permanente para la misma query puede resolverse modificando:

```text
input
query
configuration
schema
permissions
```

---

# 107. Resource exhaustion

Puede incluir:

```text
memory
temporary disk
connection slots
statement limit
lock table
database disk
sort memory
query quota
```

---

# 108. Resource exhaustion handling

Se especializará además en:

```text
241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 109. Retry storm protection

Si el servidor está sin recursos:

```text
immediate retry
```

puede empeorar la situación.

---

# 110. Resource failure classification

Podrá resultar:

```text
TRANSIENT
```

pero:

```text
retry = policy-controlled
```

con backoff/budget/circuit breaker.

---

# 111. Result receive failure

Puede ocurrir después de ejecución exitosa.

---

# 112. Result decode failure

Ejemplos:

```text
protocol corruption
invalid encoding
driver conversion failure
unexpected field representation
```

---

# 113. Result decode failure ≠ query execution failure

La base puede haber ejecutado correctamente.

---

# 114. Hydration failure

Ejemplo:

```text
query succeeds
rows returned
entity hydration fails
```

Entonces:

```text
Database Query = SUCCESS
Application Materialization = FAILURE
```

---

# 115. Hydration failure

No deberá marcar la query como database execution failure.

---

# 116. Hydration and transaction

Una hydration failure puede ocurrir dentro de una transaction aun cuando el statement haya sido exitoso.

El Transaction Manager deberá decidir qué hacer con la unidad de trabajo.

---

# 117. ORM state

Si hydration falla después de crear parcialmente objetos:

```text
EntityManager
IdentityMap
HydrationSession
```

deberán aplicar sus propias reglas de cleanup/taint.

---

# 118. ORM failure ≠ SQL failure

---

# 119. Persistence query failures

Para ORM:

```text
UnitOfWork
    ↓
Persistence Plan
    ↓
Query Models
    ↓
Execution
```

Un query failure deberá propagarse al Persistence Engine conservando:

```text
which operation failed
which operations succeeded
transaction state
outcome certainty
```

---

# 120. Batch persistence

En múltiples statements:

```text
Statement 1 SUCCESS
Statement 2 SUCCESS
Statement 3 FAILURE
```

no debe reducirse a:

```text
flush failed
```

sin preservar progreso.

---

# 121. Transactional batch

Si todo estaba en transaction y rollback se confirma:

```text
database persistence outcome
```

puede volver a un estado conocido de rollback.

Pero:

```text
object graph
```

no se rebobina automáticamente.

---

# 122. Non-transactional batch

Puede existir:

```text
partial database mutation
```

---

# 123. Bulk operations

`203–205` definieron:

```text
Bulk Insert
Bulk Update
Bulk Delete
```

Sus fallos deberán conservar:

```text
batch/chunk identity
confirmed progress
unknown progress
affected ranges
```

cuando sea posible.

---

# 124. Large dataset processing

`208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md` deberá recibir failures estructurados.

---

# 125. Partial progress

Debe distinguirse:

```text
Query partial result
```

de:

```text
Batch partial progress
```

---

# 126. QueryFailureScope

```php
enum QueryFailureScope
{
    case QUERY;
    case STATEMENT;
    case TRANSACTION;
    case CONNECTION;
    case ENDPOINT;
    case SHARD;
    case DATABASE;
    case PLATFORM;
    case UNKNOWN;
}
```

---

# 127. Scope escalation

Un error puede comenzar como:

```text
CONNECTION
```

y generar evidencia de:

```text
ENDPOINT
```

pero no deberá escalar sin fundamento.

---

# 128. Query error ≠ endpoint unhealthy

Ejemplo:

```text
syntax error
```

no deberá degradar endpoint health.

---

# 129. Constraint violation ≠ endpoint unhealthy

---

# 130. Permission denied ≠ endpoint unhealthy

---

# 131. Connection reset

Sí puede contribuir a health evaluation, delegando a:

```text
236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
```

---

# 132. Query failure and circuit breaker

Solo categorías apropiadas deberán alimentar:

```text
Circuit Breaker
```

---

# 133. Circuit-relevant failures

Ejemplos potenciales:

```text
connection unavailable
server overloaded
server shutdown
repeated network timeout
resource exhaustion
```

---

# 134. Circuit-irrelevant failures

Normalmente:

```text
unique violation
syntax error
bad input
permission denied
unknown column
```

---

# 135. Query recovery pipeline

```text
Query Failure Observation
          │
          ▼
Failure Normalizer
          │
          ▼
Platform Classifier
          │
          ▼
Execution Evidence Resolver
          │
          ▼
Outcome Resolver
          │
          ▼
Connection Impact
          │
          ▼
Transaction Impact
          │
          ▼
Retry Assessment
          │
          ▼
Recovery Policy
          │
   ┌──────┼────────┐
   │      │        │
 FAIL   RETRY   ESCALATE
```

---

# 136. Failure observation

```php
final readonly class QueryFailureObservation
{
    public function __construct(
        public QueryExecutionContext $context,
        public QueryFailureStage $stage,
        public ?DriverError $driverError,
        public ?Throwable $cause,
        public QueryExecutionEvidence $evidence,
    ) {}
}
```

---

# 137. Outcome resolver

```php
interface QueryOutcomeResolver
{
    public function resolve(
        QueryFailureObservation $failure
    ): QueryOutcome;
}
```

---

# 138. Conservative outcome

Si no puede demostrarse:

```text
SUCCESS
```

o:

```text
FAILURE
```

y side effects son posibles:

```text
UNKNOWN
```

---

# 139. UNKNOWN is first-class

`UNKNOWN` no será:

```text
null
```

ni:

```text
generic exception
```

Será un estado explícito.

---

# 140. Query recovery actions

```php
enum QueryRecoveryAction
{
    case PROPAGATE;
    case RETRY_STATEMENT;
    case RETRY_TRANSACTION;
    case REACQUIRE_CONNECTION;
    case REQUEST_FAILOVER;
    case ABORT_TRANSACTION;
    case CANCEL_OPERATION;
    case MARK_UNKNOWN;
}
```

---

# 141. RETRY_STATEMENT

Solo permitido cuando exista prueba suficiente de que:

```text
statement replay
```

preserva semántica.

---

# 142. RETRY_TRANSACTION

Delega al Transaction Retry System.

---

# 143. REACQUIRE_CONNECTION

No significa automáticamente:

```text
retry query
```

---

# 144. REQUEST_FAILOVER

Delega a:

```text
240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
```

---

# 145. MARK_UNKNOWN

Acción válida y necesaria.

---

# 146. Failure severity

```php
enum QueryFailureSeverity
{
    case INFO;
    case WARNING;
    case ERROR;
    case CRITICAL;
}
```

No deberá utilizarse como sustituto de category/outcome.

---

# 147. Developer-facing exception model

```text
DatabaseQueryException
│
├── QueryDefinitionException
├── QueryValidationException
├── QuerySemanticException
├── QueryPlanningException
├── QueryCompilationException
│
├── StatementPreparationException
├── ParameterBindingException
│
├── DatabaseExecutionException
│   ├── DatabaseSyntaxException
│   ├── ConstraintViolationException
│   │   ├── UniqueConstraintViolationException
│   │   ├── ForeignKeyConstraintViolationException
│   │   ├── NotNullConstraintViolationException
│   │   └── CheckConstraintViolationException
│   │
│   ├── DatabasePermissionException
│   ├── DatabaseObjectNotFoundException
│   ├── DatabaseDataException
│   ├── DeadlockException
│   ├── SerializationFailureException
│   ├── LockTimeoutException
│   ├── QueryTimeoutException
│   ├── QueryCancelledException
│   └── DatabaseResourceException
│
├── QueryResultException
├── QueryHydrationException
│
├── UnknownQueryOutcomeException
└── PlatformQueryException
```

---

# 148. Exception context

Una exception podrá exponer de forma segura:

```text
query fingerprint
query type
failure category
stage
SQLSTATE
vendor code
transaction state
connection state
outcome
retry assessment
```

---

# 149. Raw SQL exposure

Producción:

```text
raw SQL
```

no deberá exponerse indiscriminadamente.

---

# 150. Parameter exposure

Mucho menos:

```text
raw bindings
```

---

# 151. Debug environments

Podrán ofrecer información adicional bajo:

```text
SensitiveDataPolicy
DebugInformationPolicy
```

---

# 152. Query fingerprint

Se preferirá:

```text
semantic query fingerprint
```

para correlación.

---

# 153. Query ID

Cada ejecución podrá tener:

```text
QueryExecutionId
```

---

# 154. Attempt ID

Cada intento tendrá:

```text
QueryAttemptId
```

---

# 155. Logical operation ID

Múltiples attempts podrán compartir:

```text
QueryOperationId
```

---

# 156. Identidades

```text
Operation
 ├── Attempt 1
 ├── Attempt 2
 └── Attempt 3
```

Esto evita mezclar telemetry de retries.

---

# 157. Retry telemetry

Debe ser posible saber:

```text
logical operation count
attempt count
retry count
```

por separado.

---

# 158. Telemetry

Integración con:

```text
217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
```

---

# 159. Failure event

Conceptualmente:

```text
DatabaseQueryFailed
```

podrá contener:

```text
operation id
attempt id
query fingerprint
stage
category
outcome
SQLSTATE class
retry disposition
transaction usability
connection health
duration
```

---

# 160. No sensitive bindings

Por default:

```text
password
token
email
PII
business payload
```

no aparecerán en telemetry.

---

# 161. Metrics

Ejemplos:

```text
db.query.failures
db.query.unknown_outcomes
db.query.constraint_violations
db.query.deadlocks
db.query.serialization_failures
db.query.lock_timeouts
db.query.timeouts
db.query.cancellations
db.query.resource_failures
db.query.retries
```

---

# 162. Cardinality

No usar directamente como labels:

```text
raw SQL
table name from arbitrary user input
parameter value
exception message
```

---

# 163. Slow query vs failed query

```text
Slow Query
≠
Failed Query
```

Una query puede ser ambas.

---

# 164. Profiler integration

Query Profiler deberá poder mostrar:

```text
Duration
Outcome
Failure Stage
Failure Category
Attempts
```

---

# 165. N+1 integration

Una query individual dentro de un N+1 puede fallar.

Eso no convierte:

```text
N+1
```

en failure category.

---

# 166. Event system

Los query failure events serán observacionales.

Listeners no deberán cambiar el resultado de la query de forma arbitraria.

---

# 167. Failure listeners

No deberán poder convertir:

```text
FAILED
```

en:

```text
SUCCESS
```

sin una recovery pipeline formal.

---

# 168. Exception listeners

Observabilidad:

```text
YES
```

Mutación del outcome:

```text
NO
```

por default.

---

# 169. Retry ownership

Solo una capa deberá ser propietaria del retry.

Evitar:

```text
Driver retries
Executor retries
TransactionManager retries
Repository retries
HTTP middleware retries
```

todos simultáneamente.

---

# 170. Retry amplification

Ejemplo:

```text
Driver: 3 attempts
Executor: 3 attempts
Transaction: 3 attempts
Job: 3 attempts
```

podría producir:

```text
3 × 3 × 3 × 3 = 81 attempts
```

---

# 171. Retry coordinator

`238_DATABASE_RETRY_POLICY_SYSTEM.md` deberá coordinar budgets.

---

# 172. Hidden driver retries

Cuando un driver tenga retry interno, VoltStack deberá:

```text
disable
configure
or account for it
```

cuando sea posible.

---

# 173. Retry budget propagation

```text
Request Budget
    ↓
Transaction Budget
    ↓
Query Budget
    ↓
Connection Budget
```

No deberán ser budgets independientes ilimitados.

---

# 174. Deadline propagation

Cada attempt deberá respetar:

```text
remaining deadline
```

---

# 175. No retry after deadline

---

# 176. Backoff

No todo query retry requiere backoff.

Pero fallos de infraestructura normalmente sí deberán considerar:

```text
backoff
jitter
```

---

# 177. Deadlock backoff

Puede utilizar pequeño randomized backoff para reducir repetición inmediata del conflicto.

---

# 178. Retry and transaction scope

Si query está dentro de transaction:

```text
statement-level retry
```

puede ser inválido aunque fuera de transaction fuera aceptable.

---

# 179. Savepoints

En algunos escenarios controlados, un savepoint puede proporcionar un recovery boundary.

Pero:

```text
Savepoint
≠
Universal Query Retry Mechanism
```

---

# 180. Nested transactions

Las reglas de:

```text
168_DATABASE_NESTED_TRANSACTION_SYSTEM.md
```

deberán preservarse.

---

# 181. Optimistic locking

Un:

```text
affected rows = 0
```

bajo version check deberá convertirse en:

```text
OptimisticLockConflict
```

no en generic query failure.

---

# 182. Optimistic conflict ≠ infrastructure failure

---

# 183. Pessimistic locking

Errores de:

```text
NOWAIT
SKIP LOCKED
lock timeout
```

deberán mantener semántica propia.

---

# 184. Read replicas

Un query failure en replica puede permitir:

```text
another eligible replica
```

pero únicamente si la política de consistencia lo permite.

---

# 185. Replica failure ≠ writer fallback automatically

---

# 186. Read fallback to writer

Podrá existir como política explícita:

```text
Replica unavailable
    ↓
Writer fallback
```

si:

```text
routing
consistency
transaction
load policy
```

lo permiten.

---

# 187. Sharding

Una query ligada a:

```text
Shard A
```

no podrá recuperarse ejecutándola en:

```text
Shard B
```

---

# 188. Unknown shard routing

Debe fallar antes de ejecución cuando sea posible:

```text
Planning Failure
```

---

# 189. Distributed query

Puede existir:

```text
Shard A SUCCESS
Shard B FAILURE
Shard C SUCCESS
```

Resultado:

```text
Distributed Query Partial Failure
```

---

# 190. Partial distributed failure

No deberá reportarse como:

```text
complete success
```

---

# 191. Distributed write

Sin protocolo distribuido explícito:

```text
partial commit
```

es posible.

VoltStack no deberá fingir atomicidad.

---

# 192. Distributed outcome

Podrá modelarse:

```php
final readonly class DistributedQueryOutcome
{
    /** @var array<ShardId, QueryOutcome> */
    public array $shards;
}
```

---

# 193. Tenant isolation

Query failures no deberán cambiar:

```text
TenantContext
```

durante recovery.

---

# 194. Tenant fallback prohibited

```text
Tenant database unavailable
→ use another tenant database
```

será imposible.

---

# 195. Authorization

Una query rechazada por policy antes de ejecución será:

```text
NOT_EXECUTED
```

---

# 196. Database permission error

Es diferente de framework authorization denial.

---

# 197. Security event

Permission failures sospechosos podrán integrarse con audit/security telemetry.

---

# 198. SQL injection prevention

Un input rechazado antes de query generation:

```text
SecurityInputFailure
```

no deberá aparecer como database syntax error.

---

# 199. Query audit

El audit system podrá registrar:

```text
attempted operation
outcome
failure category
```

sin almacenar datos sensibles innecesarios.

---

# 200. Persistent runtime

En FrankenPHP/RoadRunner/OpenSwoole:

```text
QueryFailureContext
```

será scope-local.

---

# 201. No static current query

Prohibido:

```php
static $currentQuery;
static $lastFailure;
```

como mutable request state.

---

# 202. Cleanup

Después de failure:

```text
statement resources
result resources
connection lease
transaction state
hydration state
```

deberán resolverse según ownership.

---

# 203. Cleanup failure

Un error durante cleanup no deberá ocultar el error original.

---

# 204. Multiple failure model

Puede existir:

```text
Primary Failure:
    QueryTimeout

Cleanup Failure:
    ConnectionReset
```

Ambos deberán conservarse.

---

# 205. Failure chain

```php
final readonly class DatabaseFailureChain
{
    public function __construct(
        public DatabaseFailure $primary,
        public array $secondary,
    ) {}
}
```

---

# 206. Primary failure

Representa el fallo que causó la salida de la operación.

---

# 207. Secondary failures

Pueden incluir:

```text
rollback failure
statement close failure
connection reset failure
telemetry exporter failure
```

---

# 208. Telemetry failure

Nunca deberá reemplazar el query failure original.

---

# 209. Event listener failure

Tampoco deberá ocultar el failure original.

---

# 210. Debug toolbar failure

Jamás deberá alterar query outcome.

---

# 211. Failure normalization pipeline

```text
Throwable / Driver Error
        │
        ▼
RawFailureCapture
        │
        ▼
SensitiveDataRedaction
        │
        ▼
PlatformClassification
        │
        ▼
ExecutionEvidenceResolution
        │
        ▼
OutcomeResolution
        │
        ▼
ConnectionImpactAssessment
        │
        ▼
TransactionImpactAssessment
        │
        ▼
RetryAssessment
        │
        ▼
Normalized QueryFailure
```

---

# 212. Order matters

No deberá decidirse retry antes de resolver:

```text
outcome
connection health
transaction usability
```

---

# 213. QueryFailureHandler

```php
interface QueryFailureHandler
{
    public function handle(
        QueryFailureObservation $observation
    ): QueryFailureResolution;
}
```

---

# 214. QueryFailureResolution

```php
final readonly class QueryFailureResolution
{
    public function __construct(
        public QueryFailure $failure,
        public QueryRecoveryAction $action,
        public ConnectionImpact $connectionImpact,
        public TransactionImpact $transactionImpact,
    ) {}
}
```

---

# 215. ConnectionImpact

```php
enum ConnectionImpact
{
    case NONE;
    case VALIDATE;
    case SUSPECT;
    case DISCARD;
    case UNKNOWN;
}
```

---

# 216. TransactionImpact

```php
enum TransactionImpact
{
    case NONE;
    case USABLE;
    case REQUIRES_ROLLBACK;
    case ABORTED;
    case TAINTED;
    case UNKNOWN;
}
```

---

# 217. Example: validation error

```text
Query:
    invalid aggregate

Failure:
    SEMANTIC_ERROR

Stage:
    SEMANTIC_ANALYSIS

Outcome:
    NOT_EXECUTED

Connection impact:
    NONE

Transaction impact:
    NONE

Retry:
    NEVER for same query
```

---

# 218. Example: unique violation

```text
Stage:
    EXECUTION

Category:
    CONSTRAINT_VIOLATION

Type:
    UNIQUE

Outcome:
    FAILED

Connection:
    HEALTHY

Retry:
    NEVER for same input by default
```

---

# 219. Example: deadlock

```text
Stage:
    EXECUTION

Category:
    DEADLOCK

Outcome:
    FAILED

Transaction:
    ABORTED/REQUIRES_ROLLBACK

Retry:
    REQUIRES_TRANSACTION_REPLAY
```

---

# 220. Example: lock timeout

```text
Stage:
    EXECUTION

Category:
    LOCK_TIMEOUT

Outcome:
    FAILED

Transaction:
    platform-dependent

Retry:
    policy-controlled
```

---

# 221. Example: connection lost before send

```text
Stage:
    TRANSMISSION

Send:
    NOT_SENT

Outcome:
    NOT_EXECUTED

Connection:
    BROKEN

Retry:
    POSSIBLE
```

---

# 222. Example: connection lost after write

```text
Stage:
    RESULT_RECEIVE

Send:
    SENT

Execution:
    UNKNOWN

Response:
    LOST

Outcome:
    UNKNOWN

Connection:
    BROKEN

Retry:
    NEVER blindly
```

---

# 223. Example: hydration failure

```text
Database execution:
    SUCCEEDED

Result:
    COMPLETE

Hydration:
    FAILED

Query database outcome:
    SUCCEEDED

Application operation:
    FAILED
```

---

# 224. Example: timeout

```text
Client timeout
    ↓
Cancellation request sent
    ↓
No confirmation
```

Resultado:

```text
Query outcome:
    UNKNOWN

Connection:
    SUSPECT/UNKNOWN

Retry:
    requires analysis
```

---

# 225. Directory structure

```text
src/Quantum/Database/Resilience/Query/
│
├── Contract/
│   ├── QueryFailureClassifier.php
│   ├── QueryFailureHandler.php
│   ├── QueryOutcomeResolver.php
│   ├── QueryRetryAssessor.php
│   └── QueryFailurePolicy.php
│
├── Failure/
│   ├── QueryFailure.php
│   ├── QueryFailureId.php
│   ├── QueryFailureStage.php
│   ├── QueryFailureCategory.php
│   ├── QueryFailureSeverity.php
│   ├── QueryFailureScope.php
│   ├── QueryFailureObservation.php
│   └── QueryFailurePersistence.php
│
├── Evidence/
│   ├── QueryExecutionEvidence.php
│   ├── QuerySendState.php
│   ├── QueryExecutionState.php
│   ├── QueryResponseState.php
│   └── QueryCancellationState.php
│
├── Outcome/
│   ├── QueryOutcome.php
│   ├── QueryOutcomeResolver.php
│   ├── DistributedQueryOutcome.php
│   └── UnknownOutcomePolicy.php
│
├── Constraint/
│   ├── ConstraintViolation.php
│   ├── ConstraintViolationType.php
│   └── ConstraintViolationResolver.php
│
├── Timeout/
│   ├── QueryTimeoutSource.php
│   └── QueryTimeoutClassifier.php
│
├── Recovery/
│   ├── QueryRecoveryAction.php
│   ├── QueryFailureResolution.php
│   ├── QueryRecoveryCoordinator.php
│   ├── ConnectionImpact.php
│   └── TransactionImpact.php
│
├── Retry/
│   ├── QueryRetryAssessment.php
│   ├── RetryDisposition.php
│   └── ReplaySafety.php
│
├── Classifier/
│   ├── CompositeQueryFailureClassifier.php
│   ├── MySqlQueryFailureClassifier.php
│   ├── MariaDbQueryFailureClassifier.php
│   ├── PostgreSqlQueryFailureClassifier.php
│   └── SqliteQueryFailureClassifier.php
│
├── Telemetry/
│   └── QueryFailureTelemetry.php
│
├── Testing/
│   ├── QueryFailureInjector.php
│   ├── QueryFailureScenario.php
│   └── FakeQueryFailureClassifier.php
│
└── Exception/
    ├── DatabaseQueryException.php
    ├── QueryDefinitionException.php
    ├── QueryValidationException.php
    ├── QuerySemanticException.php
    ├── QueryPlanningException.php
    ├── QueryCompilationException.php
    ├── StatementPreparationException.php
    ├── ParameterBindingException.php
    ├── DatabaseExecutionException.php
    ├── ConstraintViolationException.php
    ├── UniqueConstraintViolationException.php
    ├── ForeignKeyConstraintViolationException.php
    ├── NotNullConstraintViolationException.php
    ├── CheckConstraintViolationException.php
    ├── DatabasePermissionException.php
    ├── DatabaseObjectNotFoundException.php
    ├── DatabaseDataException.php
    ├── DeadlockException.php
    ├── SerializationFailureException.php
    ├── LockTimeoutException.php
    ├── QueryTimeoutException.php
    ├── QueryCancelledException.php
    ├── DatabaseResourceException.php
    ├── QueryResultException.php
    ├── QueryHydrationException.php
    └── UnknownQueryOutcomeException.php
```

---

# 226. Testing strategy

El sistema deberá probar cada failure boundary independientemente.

---

# 227. Pre-execution tests

Casos:

```text
construction
validation
semantic analysis
planning
compilation
binding
```

Esperado:

```text
Outcome = NOT_EXECUTED
```

---

# 228. Constraint tests

Probar:

```text
unique
foreign key
not null
check
```

y verificar clasificación exacta.

---

# 229. Deadlock tests

Simular transacciones concurrentes y comprobar:

```text
deadlock classified
transaction impact known
statement not blindly retried
```

---

# 230. Serialization tests

Verificar:

```text
transaction replay delegated
```

---

# 231. Lock timeout tests

Comprobar diferencia con:

```text
deadlock
query timeout
```

---

# 232. Timeout tests

Probar:

```text
server timeout
client timeout
driver timeout
```

---

# 233. Cancellation tests

Probar:

```text
requested
confirmed
failed
unknown
```

---

# 234. Lost ACK test

Escenario:

```text
INSERT executed
response lost
```

Debe resultar:

```text
UNKNOWN
```

y no retry automático.

---

# 235. Read partial result test

```text
100 rows expected
50 received
connection lost
```

Nunca devolver:

```text
50 rows as complete result
```

---

# 236. Hydration test

```text
query success
hydration failure
```

Debe conservar:

```text
database outcome = SUCCESS
application materialization = FAILURE
```

---

# 237. Transaction usability tests

Cada plataforma deberá probar cómo distintos statement errors afectan:

```text
transaction usability
```

---

# 238. Pool health tests

Errores de query puramente lógicos no deberán:

```text
discard healthy connection
```

---

# 239. Circuit tests

Constraint violations no deberán abrir circuit breaker.

---

# 240. Tenant tests

Failure recovery nunca cambiará tenant context.

---

# 241. Shard tests

Failure recovery nunca cambiará shard ownership arbitrariamente.

---

# 242. Sensitive data tests

Verificar ausencia de:

```text
passwords
tokens
PII bindings
raw credentials
```

en exceptions/telemetry.

---

# 243. Persistent runtime tests

```text
Request A fails query
Request B starts
```

Request B no deberá heredar:

```text
last query
failure state
transaction taint
bindings
retry attempt state
```

---

# 244. Failure injection

Testing API conceptual:

```php
$failures->inject(
    stage: QueryFailureStage::RESULT_RECEIVE,
    category: QueryFailureCategory::CONNECTION_FAILURE,
    after: 1,
);
```

---

# 245. Deterministic retries

Retry testing deberá utilizar:

```text
Clock
RandomSource
FailureInjector
```

---

# 246. Arquitectural invariants

## DB-QUERY-FAIL-001
Observed Query Error será distinto de Known Database Outcome.

## DB-QUERY-FAIL-002
Query Failure será distinto de Retry Permission.

## DB-QUERY-FAIL-003
Query Failure será distinto de Connection Failure.

## DB-QUERY-FAIL-004
Query Failure será distinto de Transaction Failure.

## DB-QUERY-FAIL-005
Query Failure será distinto de Hydration Failure.

## DB-QUERY-FAIL-006
Failure stage será explícito.

## DB-QUERY-FAIL-007
Failure category será explícita.

## DB-QUERY-FAIL-008
Outcome será explícito.

## DB-QUERY-FAIL-009
UNKNOWN será first-class.

## DB-QUERY-FAIL-010
UNKNOWN no será convertido en FAILURE.

## DB-QUERY-FAIL-011
UNKNOWN no será convertido en SUCCESS.

## DB-QUERY-FAIL-012
Pre-execution validation failure será NOT_EXECUTED.

## DB-QUERY-FAIL-013
Semantic failure será NOT_EXECUTED cuando no hubo I/O.

## DB-QUERY-FAIL-014
Planning failure será NOT_EXECUTED cuando no hubo I/O.

## DB-QUERY-FAIL-015
Compilation failure será NOT_EXECUTED.

## DB-QUERY-FAIL-016
Binding failure previo al send será NOT_EXECUTED.

## DB-QUERY-FAIL-017
Send state será preservado cuando sea observable.

## DB-QUERY-FAIL-018
POSSIBLY_SENT no será equivalente a NOT_SENT.

## DB-QUERY-FAIL-019
SENT no será equivalente a SUCCESS.

## DB-QUERY-FAIL-020
Lost response no será equivalente a failed execution.

## DB-QUERY-FAIL-021
Partial response no será complete result.

## DB-QUERY-FAIL-022
Constraint violation será distinguible.

## DB-QUERY-FAIL-023
Unique violation será distinguible.

## DB-QUERY-FAIL-024
Foreign key violation será distinguible.

## DB-QUERY-FAIL-025
Not-null violation será distinguible.

## DB-QUERY-FAIL-026
Check violation será distinguible.

## DB-QUERY-FAIL-027
Constraint violation no degradará endpoint health por default.

## DB-QUERY-FAIL-028
Statement failure no implicará transaction failure universalmente.

## DB-QUERY-FAIL-029
Transaction usability será explícita.

## DB-QUERY-FAIL-030
Platform transaction error semantics serán respetadas.

## DB-QUERY-FAIL-031
Deadlock será distinto de lock timeout.

## DB-QUERY-FAIL-032
Deadlock será distinto de serialization failure.

## DB-QUERY-FAIL-033
Deadlock no tendrá blind statement retry.

## DB-QUERY-FAIL-034
Transaction replay será delegado.

## DB-QUERY-FAIL-035
Serialization failure podrá requerir transaction replay.

## DB-QUERY-FAIL-036
Query timeout source podrá distinguirse.

## DB-QUERY-FAIL-037
Client timeout no significará server cancellation.

## DB-QUERY-FAIL-038
Cancellation request no significará cancellation confirmation.

## DB-QUERY-FAIL-039
Unknown cancellation podrá producir unknown outcome.

## DB-QUERY-FAIL-040
Read retry no garantizará same result.

## DB-QUERY-FAIL-041
SELECT no se asumirá automáticamente replay-safe.

## DB-QUERY-FAIL-042
Volatile query semantics serán consideradas.

## DB-QUERY-FAIL-043
Unknown write no tendrá blind retry.

## DB-QUERY-FAIL-044
Statement idempotency será distinta de business idempotency.

## DB-QUERY-FAIL-045
Transaction idempotency será distinta de statement idempotency.

## DB-QUERY-FAIL-046
External side-effect idempotency será distinta de database idempotency.

## DB-QUERY-FAIL-047
Query layer no inventará business idempotency.

## DB-QUERY-FAIL-048
INSERT lost ACK será tratado conservadoramente.

## DB-QUERY-FAIL-049
UPDATE retry safety será analizada semánticamente.

## DB-QUERY-FAIL-050
DELETE no se asumirá universalmente idempotente.

## DB-QUERY-FAIL-051
DDL semantics serán platform-aware.

## DB-QUERY-FAIL-052
DDL no se asumirá universalmente transactional.

## DB-QUERY-FAIL-053
Unknown DDL outcome será preservado.

## DB-QUERY-FAIL-054
Query Engine no ejecutará migrations automáticamente ante missing table.

## DB-QUERY-FAIL-055
Permission error será distinto de authorization denial.

## DB-QUERY-FAIL-056
Permission error no será retryable por default.

## DB-QUERY-FAIL-057
Data error podrá ser permanent-for-same-input.

## DB-QUERY-FAIL-058
Transient será distinto de retryable.

## DB-QUERY-FAIL-059
Resource exhaustion no tendrá immediate retry ilimitado.

## DB-QUERY-FAIL-060
Resource retry tendrá budget.

## DB-QUERY-FAIL-061
Result decode failure será distinto de execution failure.

## DB-QUERY-FAIL-062
Hydration failure será distinto de database execution failure.

## DB-QUERY-FAIL-063
Successful query + failed hydration conservará query success.

## DB-QUERY-FAIL-064
ORM cleanup tendrá ownership propio.

## DB-QUERY-FAIL-065
Persistence failure preservará statement progress.

## DB-QUERY-FAIL-066
Batch partial progress será explícito.

## DB-QUERY-FAIL-067
Query partial result será distinto de batch partial progress.

## DB-QUERY-FAIL-068
Failure scope será explícito.

## DB-QUERY-FAIL-069
Query error no implicará endpoint unhealthy.

## DB-QUERY-FAIL-070
Syntax error no alimentará circuit breaker por default.

## DB-QUERY-FAIL-071
Constraint error no alimentará circuit breaker por default.

## DB-QUERY-FAIL-072
Infrastructure failures podrán alimentar circuit breaker.

## DB-QUERY-FAIL-073
Recovery pipeline resolverá evidence antes de retry.

## DB-QUERY-FAIL-074
Outcome resolver será separado de retry policy.

## DB-QUERY-FAIL-075
Connection impact será explícito.

## DB-QUERY-FAIL-076
Transaction impact será explícito.

## DB-QUERY-FAIL-077
Retry assessment será explícito.

## DB-QUERY-FAIL-078
Retry assessment no ejecutará retry.

## DB-QUERY-FAIL-079
Recovery action será explícita.

## DB-QUERY-FAIL-080
REACQUIRE_CONNECTION no significará RETRY_QUERY.

## DB-QUERY-FAIL-081
REQUEST_FAILOVER será delegado.

## DB-QUERY-FAIL-082
MARK_UNKNOWN será una acción válida.

## DB-QUERY-FAIL-083
Exception hierarchy conservará semántica.

## DB-QUERY-FAIL-084
Exception type no será recovery policy.

## DB-QUERY-FAIL-085
Raw bindings no se expondrán por default.

## DB-QUERY-FAIL-086
Sensitive SQL data será protegida.

## DB-QUERY-FAIL-087
Semantic query fingerprint será preferido para correlación.

## DB-QUERY-FAIL-088
Logical operation ID será distinto de attempt ID.

## DB-QUERY-FAIL-089
Retries no inflarán logical query count.

## DB-QUERY-FAIL-090
Retry telemetry distinguirá attempts.

## DB-QUERY-FAIL-091
Telemetry tendrá bounded cardinality.

## DB-QUERY-FAIL-092
Raw SQL no será metric label.

## DB-QUERY-FAIL-093
Parameter value no será metric label.

## DB-QUERY-FAIL-094
Slow query será distinta de failed query.

## DB-QUERY-FAIL-095
Profiler podrá observar failures.

## DB-QUERY-FAIL-096
Event listeners no redefinirán outcome arbitrariamente.

## DB-QUERY-FAIL-097
Telemetry failure no ocultará query failure.

## DB-QUERY-FAIL-098
Debug toolbar failure no ocultará query failure.

## DB-QUERY-FAIL-099
Solo una capa será owner efectivo del retry.

## DB-QUERY-FAIL-100
Retry amplification será evitada.

## DB-QUERY-FAIL-101
Hidden driver retries serán considerados.

## DB-QUERY-FAIL-102
Retry budgets serán propagables.

## DB-QUERY-FAIL-103
Deadline será compartido entre attempts.

## DB-QUERY-FAIL-104
No habrá retry después del deadline.

## DB-QUERY-FAIL-105
Backoff podrá ser requerido.

## DB-QUERY-FAIL-106
Jitter podrá ser requerido.

## DB-QUERY-FAIL-107
Statement retry dentro de transaction requerirá validación adicional.

## DB-QUERY-FAIL-108
Savepoint no será universal retry mechanism.

## DB-QUERY-FAIL-109
Nested transaction semantics serán preservadas.

## DB-QUERY-FAIL-110
Optimistic lock conflict tendrá error propio.

## DB-QUERY-FAIL-111
Optimistic conflict no será infrastructure failure.

## DB-QUERY-FAIL-112
Pessimistic lock errors conservarán semántica.

## DB-QUERY-FAIL-113
Replica retry respetará consistency.

## DB-QUERY-FAIL-114
Replica failure no implicará writer fallback.

## DB-QUERY-FAIL-115
Writer fallback requerirá policy explícita.

## DB-QUERY-FAIL-116
Shard affinity será preservada.

## DB-QUERY-FAIL-117
Query no migrará a otro shard para recuperarse.

## DB-QUERY-FAIL-118
Distributed partial failure será explícito.

## DB-QUERY-FAIL-119
Distributed partial success no será complete success.

## DB-QUERY-FAIL-120
Cross-shard atomicity no será inventada.

## DB-QUERY-FAIL-121
Tenant context será preservado.

## DB-QUERY-FAIL-122
Tenant fallback estará prohibido.

## DB-QUERY-FAIL-123
Framework authorization failure será distinto de DB permission failure.

## DB-QUERY-FAIL-124
Security input rejection será pre-execution.

## DB-QUERY-FAIL-125
Audit podrá observar failure sin exponer payload sensible.

## DB-QUERY-FAIL-126
Failure context será scope-local.

## DB-QUERY-FAIL-127
No existirá static mutable current query.

## DB-QUERY-FAIL-128
No existirá static mutable last failure.

## DB-QUERY-FAIL-129
Cleanup respetará ownership.

## DB-QUERY-FAIL-130
Cleanup failure no ocultará primary failure.

## DB-QUERY-FAIL-131
Failure chain preservará secondary failures.

## DB-QUERY-FAIL-132
Rollback failure podrá coexistir con query failure.

## DB-QUERY-FAIL-133
Connection reset failure podrá coexistir con query failure.

## DB-QUERY-FAIL-134
Failure normalization será determinista.

## DB-QUERY-FAIL-135
Structured driver codes tendrán preferencia sobre message matching.

## DB-QUERY-FAIL-136
Message matching será fallback encapsulado.

## DB-QUERY-FAIL-137
Unknown driver error permanecerá UNKNOWN cuando corresponda.

## DB-QUERY-FAIL-138
MySQL podrá tener classifier específico.

## DB-QUERY-FAIL-139
MariaDB podrá tener classifier específico.

## DB-QUERY-FAIL-140
PostgreSQL podrá tener classifier específico.

## DB-QUERY-FAIL-141
SQLite podrá tener classifier específico.

## DB-QUERY-FAIL-142
Platform capability model será respetado.

## DB-QUERY-FAIL-143
Compiler no decidirá retries.

## DB-QUERY-FAIL-144
Driver no decidirá business recovery.

## DB-QUERY-FAIL-145
Query Builder no manejará physical connection failures.

## DB-QUERY-FAIL-146
Executor capturará execution evidence.

## DB-QUERY-FAIL-147
Connection layer determinará connection health.

## DB-QUERY-FAIL-148
Transaction layer determinará transaction state.

## DB-QUERY-FAIL-149
Retry layer determinará retry scheduling.

## DB-QUERY-FAIL-150
Failover layer determinará endpoint replacement.

## DB-QUERY-FAIL-151
ORM interpretará persistence impact, no driver.

## DB-QUERY-FAIL-152
Unknown outcome no será resuelto mediante suposición.

## DB-QUERY-FAIL-153
A successful reconnect no resolverá previous query outcome.

## DB-QUERY-FAIL-154
A healthy connection no demostrará previous operation failure.

## DB-QUERY-FAIL-155
A new attempt no demostrará old attempt outcome.

## DB-QUERY-FAIL-156
A retry success no implica que previous attempt failed.

## DB-QUERY-FAIL-157
Duplicate success será considerado posible ante unknown writes.

## DB-QUERY-FAIL-158
Retry policy deberá considerar replay safety.

## DB-QUERY-FAIL-159
Replay safety deberá considerar operation semantics.

## DB-QUERY-FAIL-160
Replay safety deberá considerar transaction scope.

## DB-QUERY-FAIL-161
Replay safety deberá considerar external side effects.

## DB-QUERY-FAIL-162
Retry policy deberá considerar remaining budget.

## DB-QUERY-FAIL-163
Retry policy deberá considerar deadline.

## DB-QUERY-FAIL-164
Retry policy deberá considerar endpoint health.

## DB-QUERY-FAIL-165
Retry policy deberá considerar consistency requirements.

## DB-QUERY-FAIL-166
Retry policy deberá considerar shard/tenant affinity.

## DB-QUERY-FAIL-167
Retry policy deberá considerar transaction usability.

## DB-QUERY-FAIL-168
UNKNOWN será preferido a una falsa certeza.

## DB-QUERY-FAIL-169
Correctness tendrá prioridad sobre transparent retry.

## DB-QUERY-FAIL-170
Database reality nunca será inferida únicamente de una excepción cliente.

---

# 247. Modelo formal

Sea una operación:

```text
Q
```

y un intento:

```text
Aᵢ
```

Definimos:

```text
Outcome(Aᵢ)
∈
{
    NOT_EXECUTED,
    SUCCEEDED,
    FAILED,
    PARTIALLY_OBSERVED,
    UNKNOWN
}
```

---

# 248. Regla de pre-ejecución

Si existe evidencia:

```text
SendState(Aᵢ) = NOT_SENT
```

entonces:

```text
Outcome(Aᵢ) = NOT_EXECUTED
```

respecto a la base de datos.

---

# 249. Regla de respuesta completa

Si:

```text
ExecutionState(Aᵢ) = COMPLETED_SUCCESSFULLY
```

y:

```text
ResponseState(Aᵢ) = COMPLETE
```

entonces puede establecerse:

```text
Outcome(Aᵢ) = SUCCEEDED
```

salvo semánticas superiores adicionales.

---

# 250. Regla de rechazo explícito

Si:

```text
ExecutionState(Aᵢ) = COMPLETED_WITH_ERROR
```

y la base confirma rollback/rechazo del statement:

```text
Outcome(Aᵢ) = FAILED
```

---

# 251. Regla de incertidumbre

Si:

```text
SendState(Aᵢ) ∈ {POSSIBLY_SENT, SENT, UNKNOWN}
```

y:

```text
DatabaseEffect(Aᵢ)
```

no puede determinarse:

```text
Outcome(Aᵢ) = UNKNOWN
```

---

# 252. Regla de retry

Definamos:

```text
RetryAllowed(Aᵢ)
```

Entonces:

```text
RetryAllowed(Aᵢ)
=
Transient(Failure)
∧
ReplaySafe(Q)
∧
TransactionAllowsReplay
∧
ConsistencyAllowsReplay
∧
BudgetAvailable
∧
DeadlineAvailable
∧
RecoveryPolicyAllows
```

Por tanto:

```text
Transient(Failure)
```

por sí solo es insuficiente.

---

# 253. Regla de unknown write

Si:

```text
Mutation(Q) = true
```

y:

```text
Outcome(Aᵢ) = UNKNOWN
```

entonces por default:

```text
RetryAllowed(Aᵢ) = false
```

salvo una prueba superior explícita de idempotencia/recovery.

---

# 254. Regla transaccional

Si:

```text
TransactionImpact(Aᵢ)
=
REQUIRES_ROLLBACK
```

entonces:

```text
NextStatementAllowed = false
```

hasta resolver la transaction.

---

# 255. Regla de hidratación

Si:

```text
DatabaseOutcome(Q) = SUCCEEDED
```

y:

```text
HydrationOutcome(Q) = FAILED
```

entonces:

```text
DatabaseOutcome(Q)
```

no deberá sobrescribirse como `FAILED`.

---

# 256. Regla distribuida

Para shards:

```text
S = {s₁, s₂, ..., sₙ}
```

el resultado global deberá derivarse de:

```text
Outcome(Q, s₁)
Outcome(Q, s₂)
...
Outcome(Q, sₙ)
```

y no asumirse `SUCCESS` si alguno es:

```text
FAILED
UNKNOWN
PARTIAL
```

sin semántica explícita que permita ese resultado.

---

# 257. Modelo final

```text
                         QUERY
                           │
                           ▼
                     Query Model
                           │
                           ▼
                    Pre-I/O Pipeline
                           │
          ┌────────────────┼─────────────────┐
          │                │                 │
      Validation       Planning         Compilation
          │                │                 │
          └──── failure ───┴──── failure ───┘
                           │
                           ▼
                    NOT_EXECUTED
                           │
                           │
                otherwise ▼
                      Statement
                           │
                           ▼
                        Binding
                           │
                           ▼
                         Send
                           │
                           ▼
                       Database
                           │
                 ┌─────────┼─────────┐
                 │         │         │
              Success    Error    Uncertain
                 │         │         │
                 ▼         ▼         ▼
              Result    Classify   UNKNOWN
                 │         │
                 │         ▼
                 │    Failure Type
                 │         │
                 │         ▼
                 │   Transaction Impact
                 │         │
                 │         ▼
                 │   Connection Impact
                 │         │
                 └────┬────┘
                      ▼
                Retry Assessment
                      │
          ┌───────────┼───────────┐
          │           │           │
       PROPAGATE    RETRY      ESCALATE
          │           │           │
          │           ▼           │
          │      Retry Policy     │
          │           │           │
          └───────────┼───────────┘
                      ▼
                Final Resolution
```

---

# 258. Regla maestra final

> **VoltStack nunca inferirá el estado real de una operación únicamente a partir de la excepción observada por la aplicación. El sistema conservará evidencia sobre construcción, envío, ejecución, respuesta, conexión y transacción para determinar si una query no fue ejecutada, falló de forma confirmada, tuvo éxito o quedó en un estado desconocido.**

En forma compacta:

```text
Exception
≠
Database Outcome

Query Failure
≠
Connection Failure

Query Failure
≠
Transaction Failure

Query Failure
≠
Retry Permission

Transient
≠
Retryable

SELECT
≠
Automatically Replay-Safe

Idempotent Statement
≠
Idempotent Business Operation

Timeout
≠
Cancellation

Cancellation Requested
≠
Cancellation Confirmed

Lost ACK
≠
Failed Write

Reconnect
≠
Safe Retry

Statement Failure
≠
Transaction Failure

Hydration Failure
≠
Database Execution Failure

Partial Result
≠
Complete Result

UNKNOWN
≠
FAILURE

UNKNOWN
≠
SUCCESS

UNKNOWN WRITE
→
NO BLIND RETRY
```

La prioridad será:

```text
Correctness
    >
Outcome Certainty
    >
Transaction Integrity
    >
Consistency
    >
Automatic Recovery
    >
Convenience
```

---

# 259. Estado del Bloque 23

```text
BLOCK 23 — RESILIENCE

✓ 235_DATABASE_RESILIENCE_ARCHITECTURE.md
✓ 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
✓ 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
○ 238_DATABASE_RETRY_POLICY_SYSTEM.md
○ 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
○ 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
○ 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 260. Siguiente documento

```text
238_DATABASE_RETRY_POLICY_SYSTEM.md
```

El siguiente documento definirá la arquitectura central de reintentos de VoltStack Database:

```text
RetryPolicy
RetryContext
RetryDecision
RetryBudget
RetryScope
RetryOwnership
RetryClassification
ReplaySafety
Idempotency
Attempt tracking
Backoff
Exponential backoff
Jitter
Deadlines
Retry amplification prevention
Statement retry
Transaction retry
Connection retry
Failover-aware retry
Replica-aware retry
Shard-aware retry
Unknown outcome protection
Retry telemetry
Persistent-runtime isolation
```

y establecerá una regla crítica para todo VoltStack:

> **Un retry no será una reacción automática ante una excepción; será una nueva ejecución autorizada únicamente cuando el sistema pueda justificar que existe un nuevo intento útil, acotado y semánticamente seguro.**