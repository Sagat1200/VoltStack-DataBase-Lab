# 81_DATABASE_RESULT_SYSTEM.md

# VoltStack Quantum Database
## Result System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 81 — Result System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Result System` define la representación común, tipada y desacoplada de los resultados producidos por la ejecución de operaciones de base de datos dentro de VoltStack.

Su responsabilidad comienza cuando el driver ha ejecutado un statement y devuelve un resultado nativo:

```text
DriverPreparedStatement
        ↓
      execute
        ↓
Native Driver Result
        ↓
Driver Result Adapter
        ↓
   Result System
        ↓
   DatabaseResult
```

El objetivo es impedir que las capas superiores dependan directamente de:

```text
PDOStatement
mysqli_result
PgSql\Result
SQLite3Result
driver-specific cursors
driver-specific metadata
```

La abstracción general será:

```text
NativeDriverResult
        ↓
DriverResultAdapter
        ↓
DatabaseResult
```

---

# 2. Principio fundamental

```text
Database Result
≠
Driver Result
≠
Result Cursor
≠
Hydrated Entity
≠
ORM Collection
```

Cada concepto pertenece a una responsabilidad diferente.

---

# 3. Frontera arquitectónica

El flujo completo del Execution Engine será:

```text
CompiledDatabaseCommand
        ↓
Prepared Statement System
        ↓
Parameter Binding System
        ↓
BoundPreparedStatement
        ↓
Statement Execution System
        ↓
Native Driver Result
        ↓
Result System
        ↓
DatabaseResult
        ↓
Result Cursor / Streaming
        ↓
Hydration System
        ↓
ORM / Application
```

La frontera importante es:

```text
Driver-specific representation
             ↓
       Result System
             ↓
VoltStack database representation
```

---

# 4. Responsabilidades

`Result System` será responsable de:

- clasificar resultados;
- normalizar resultados nativos;
- representar filas;
- representar valores;
- representar affected rows;
- representar scalars;
- representar `RETURNING`;
- representar resultados vacíos;
- exponer metadata;
- controlar lifecycle;
- controlar ownership;
- abstraer result sets;
- detectar consumo inválido;
- normalizar errores;
- proporcionar extensibilidad;
- integrarse con telemetry;
- preservar seguridad;
- funcionar correctamente bajo runtimes persistentes.

---

# 5. Responsabilidades excluidas

`Result System` no deberá:

```text
hydrate entities
manage IdentityMap
manage UnitOfWork
resolve ORM relationships
execute additional queries
generate SQL
bind parameters
prepare statements
commit transactions
perform retries
interpret application DTOs
```

---

# 6. Result Contract

Durante planificación/compilación existe:

```text
ResultContract
```

que describe lo que una operación debe producir.

Ejemplos:

```text
ROW_STREAM
BUFFERED_ROWS
SCALAR
AFFECTED_ROWS
RETURNING_ROWS
NO_RESULT
EXTENSION_RESULT
```

El resultado runtime deberá satisfacer ese contrato.

Formalmente:

```text
RuntimeResult
⊨
ResultContract
```

---

# 7. Result Contract ≠ Result

```text
ResultContract
=
expected result shape
```

mientras:

```text
DatabaseResult
=
actual runtime result
```

---

# 8. Validación contractual

Después de ejecutar:

```text
Validate(
    NativeDriverResult,
    ExpectedResultContract
)
```

deberá comprobar que el resultado pueda representarse correctamente.

---

# 9. Contrato principal

```php
namespace VoltStack\Quantum\Database\Execution\Result\Contract;

interface DatabaseResult
{
    public function kind(): ResultKind;

    public function metadata(): ResultMetadata;

    public function state(): ResultState;

    public function close(): void;
}
```

---

# 10. ResultKind

```php
enum ResultKind: string
{
    case ROWS = 'rows';

    case SCALAR = 'scalar';

    case AFFECTED_ROWS = 'affected_rows';

    case RETURNING_ROWS = 'returning_rows';

    case NO_RESULT = 'no_result';

    case MULTIPLE_RESULT_SETS = 'multiple_result_sets';

    case EXTENSION = 'extension';
}
```

---

# 11. Result hierarchy

Conceptualmente:

```text
DatabaseResult
├── RowResult
├── ScalarResult
├── AffectedRowsResult
├── ReturningResult
├── NoResult
├── MultipleResultSetResult
└── ExtensionResult
```

---

# 12. DatabaseResult no deberá ser un array universal

Evitar:

```php
$result = [
    'rows' => ...,
    'affected' => ...,
    'scalar' => ...,
];
```

La semántica debe estar representada mediante tipos.

---

# 13. RowResult

Representa un resultado compuesto por filas.

```php
interface RowResult extends DatabaseResult
{
    public function rows(): ResultRowSource;
}
```

---

# 14. RowResult no implica buffering

```text
RowResult
```

puede estar respaldado por:

```text
BufferedResult
```

o:

```text
CursorResult
```

---

# 15. ResultRowSource

```php
interface ResultRowSource
{
    public function mode(): ResultConsumptionMode;

    public function metadata(): ResultMetadata;
}
```

Las capacidades concretas se implementarán posteriormente mediante:

```text
BufferedRowSource
CursorRowSource
StreamingRowSource
```

---

# 16. Buffered rows

Representación conceptual:

```text
Native Driver Result
        ↓
read all rows
        ↓
ResultRow[]
        ↓
BufferedRowResult
```

---

# 17. Cursor-backed rows

```text
Native Driver Cursor
        ↓
ResultCursor
        ↓
ResultRow
        ↓
ResultRow
        ↓
...
```

El detalle del cursor se formalizará en:

```text
82_DATABASE_RESULT_CURSOR_SYSTEM.md
```

---

# 18. Streaming results

Resultados de datasets grandes podrán consumirse progresivamente:

```text
Database
   ↓
Driver
   ↓
Cursor
   ↓
ResultRow
   ↓
Consumer
```

sin materializar todo el dataset.

El modelo especializado se definirá en:

```text
83_DATABASE_STREAMING_RESULT_SYSTEM.md
```

---

# 19. ResultRow

Una fila de base de datos tendrá representación propia.

```php
final readonly class ResultRow
{
    /**
     * @param list<ResultValue> $values
     */
    public function __construct(
        private array $values,
        private ResultRowMetadata $metadata,
    ) {}
}
```

---

# 20. ResultRow ≠ associative array

Internamente no deberá depender exclusivamente de:

```php
[
    'id' => 1,
    'name' => 'Alice',
]
```

porque pueden existir:

- nombres duplicados;
- aliases;
- columnas sin nombre portable;
- metadata por posición;
- columnas calculadas;
- driver-specific naming;
- case folding.

---

# 21. Positional identity

Cada columna tendrá una posición estable:

```text
ColumnIndex
```

---

# 22. Column identity

Cuando exista identidad lógica:

```text
ResultColumnId
```

deberá distinguirse de:

```text
column name
alias
ordinal position
source column
```

---

# 23. ResultValue

```php
final readonly class ResultValue
{
    public function __construct(
        public mixed $value,
        public ResultValueMetadata $metadata,
    ) {}
}
```

---

# 24. Raw driver value ≠ normalized result value

Ejemplo:

```text
Driver:
"42"

Metadata:
INTEGER

VoltStack:
42
```

cuando exista conversión segura y contractual.

---

# 25. Result conversion

El pipeline podrá ser:

```text
Driver Value
    ↓
Driver Result Type
    ↓
Result Type Resolution
    ↓
Result Value Converter
    ↓
VoltStack ResultValue
```

---

# 26. Result conversion ≠ ORM hydration

Ejemplo:

```text
"2026-09-05 10:00:00"
        ↓
DateTime result conversion
        ↓
DateTimeImmutable
```

puede pertenecer al Database Type/Result layer.

Pero:

```text
ResultRow
    ↓
new User(...)
```

pertenece al Hydration/ORM layer.

---

# 27. Fundamental boundary

```text
Database type conversion
≠
Entity hydration
```

---

# 28. ScalarResult

Representa una operación cuyo contrato produce un único valor.

```php
final class ScalarResult implements DatabaseResult
{
    public function value(): ResultValue;
}
```

Ejemplo:

```sql
SELECT COUNT(*) FROM users
```

podría producir:

```text
ScalarResult(42)
```

cuando el `ResultContract` declare resultado escalar.

---

# 29. ScalarResult no se infiere arbitrariamente

Una consulta:

```sql
SELECT id FROM users
```

no se convierte automáticamente en scalar sólo porque actualmente produzca una fila.

El contrato fue determinado antes.

---

# 30. Scalar cardinality

Podrán existir políticas:

```text
EXACTLY_ONE
ZERO_OR_ONE
FIRST
```

pero deberán ser explícitas.

---

# 31. EXACTLY_ONE

Requiere:

```text
row count = 1
column count = 1
```

cuando el contrato lo exija.

---

# 32. Zero rows

Si el contrato es:

```text
EXACTLY_ONE
```

y existen cero filas:

```text
ScalarCardinalityException
```

---

# 33. Multiple rows

Igualmente:

```text
> 1 row
```

deberá fallar cuando contradiga el contrato.

---

# 34. AffectedRowsResult

Para:

```text
INSERT
UPDATE
DELETE
```

puede existir:

```php
final readonly class AffectedRowsResult implements DatabaseResult
{
    public function __construct(
        public AffectedRowCount $affectedRows,
        public ResultMetadata $metadata,
    ) {}
}
```

---

# 35. Affected rows semantics

`affectedRows` puede tener diferencias entre plataformas.

Por ello:

```text
AffectedRowCount
```

no deberá reinterpretarse sin considerar el driver/platform contract.

---

# 36. Matched ≠ changed

En algunos sistemas:

```text
rows matched
```

puede diferir de:

```text
rows actually changed
```

VoltStack deberá conservar la semántica declarada por el driver/platform.

---

# 37. AffectedRowCount

```php
final readonly class AffectedRowCount
{
    public function __construct(
        public int $value,
        public AffectedRowSemantics $semantics,
    ) {}
}
```

---

# 38. AffectedRowSemantics

Ejemplo:

```php
enum AffectedRowSemantics
{
    case CHANGED;
    case MATCHED;
    case DRIVER_DEFINED;
}
```

---

# 39. No universal assumptions

Nunca:

```text
affectedRows = rows logically matched
```

como regla universal.

---

# 40. ReturningResult

Operaciones DML pueden producir:

```sql
INSERT ... RETURNING ...
UPDATE ... RETURNING ...
DELETE ... RETURNING ...
```

El resultado será:

```text
ReturningResult
```

---

# 41. ReturningResult

```php
interface ReturningResult extends RowResult
{
    public function mutationMetadata(): MutationResultMetadata;
}
```

---

# 42. RETURNING ≠ SELECT

Ambos pueden producir filas, pero semánticamente son distintos.

```text
SELECT rows
≠
mutation RETURNING rows
```

---

# 43. Why preserve distinction

Puede ser relevante para:

```text
telemetry
persistence
generated values
ORM synchronization
auditing
result validation
```

---

# 44. Generated identifiers

`lastInsertId()` no deberá confundirse con `RETURNING`.

```text
GeneratedIdentifier
≠
ReturningResult
```

---

# 45. Generated value channel

Un driver puede producir metadata adicional:

```text
generated identifier
sequence value
server-generated values
```

Esta información podrá formar parte de:

```text
MutationResultMetadata
```

---

# 46. NoResult

Algunas operaciones no producen datos consumibles.

```php
final readonly class NoResult implements DatabaseResult
{
}
```

---

# 47. NoResult ≠ null

Preferir:

```text
NoResult
```

sobre:

```php
null
```

porque conserva semántica explícita.

---

# 48. Multiple result sets

Algunos drivers/procedimientos pueden producir:

```text
ResultSet 1
ResultSet 2
ResultSet 3
```

---

# 49. MultipleResultSetResult

```php
interface MultipleResultSetResult extends DatabaseResult
{
    public function resultSets(): ResultSetSequence;
}
```

---

# 50. Multiple result sets capability

Sólo deberá habilitarse si:

```text
DriverCapabilities
```

lo soportan.

---

# 51. One SQL statement ≠ necessarily one result set

Aunque VoltStack genere por defecto una sola sentencia, determinados procedimientos o mecanismos del DBMS pueden producir múltiples resultados.

---

# 52. ExtensionResult

Para funcionalidades explícitamente extendidas:

```php
interface ExtensionResult extends DatabaseResult
{
    public function extensionId(): ResultExtensionId;
}
```

---

# 53. Unknown result

Un resultado nativo desconocido no deberá convertirse mediante:

```text
mixed
```

silenciosamente.

Debe producir:

```text
UnsupportedDriverResultException
```

---

# 54. ResultMetadata

```php
final readonly class ResultMetadata
{
    /**
     * @param list<ResultColumnMetadata> $columns
     */
    public function __construct(
        public array $columns,
        public ResultKind $kind,
        public ResultShape $shape,
        public ResultCapabilitySet $capabilities,
    ) {}
}
```

---

# 55. ResultColumnMetadata

Puede contener:

```text
ordinal
result column id
label
source relation
source column
semantic type
driver type
nullable
length
precision
scale
collation
sensitivity
```

cuando esa información esté disponible y sea confiable.

---

# 56. Metadata certainty

No toda metadata proveniente del driver tiene igual autoridad.

Podrá utilizarse:

```text
ResultMetadataCertainty
```

---

# 57. Certainty levels

Ejemplo:

```text
EXACT
DECLARED
DRIVER_REPORTED
INFERRED
UNKNOWN
```

---

# 58. Unknown ≠ false

Si el driver no informa nullability:

```text
nullable = UNKNOWN
```

no:

```text
nullable = false
```

---

# 59. ResultShape

Describe estructuralmente el resultado.

```php
final readonly class ResultShape
{
    /**
     * @param list<ResultColumnShape> $columns
     */
    public function __construct(
        public array $columns,
    ) {}
}
```

---

# 60. Compiled expected shape

El SQL Compiler ya puede haber generado:

```text
CompiledResultContract
```

con un shape esperado.

---

# 61. Runtime shape validation

El Result System puede comparar:

```text
ExpectedResultShape
       vs
ActualResultMetadata
```

---

# 62. Shape validation levels

```text
NONE
MINIMAL
STANDARD
STRICT
DEBUG
```

---

# 63. Critical validation

Las incompatibilidades capaces de causar corrupción semántica deberán validarse incluso en producción.

---

# 64. Column count

Ejemplo:

```text
Expected columns = 4
Actual columns = 3
```

deberá fallar cuando el contrato requiera cuatro.

---

# 65. Column order

El orden puede ser semánticamente relevante.

```text
Column 0
Column 1
Column 2
```

deberá conservarse.

---

# 66. Duplicate labels

Consulta:

```sql
SELECT users.id, orders.id
```

puede producir:

```text
id
id
```

Por ello un associative array puro es insuficiente.

---

# 67. ResultRow access

La API puede soportar:

```php
$row->at(0);

$row->at(1);
```

y, cuando sea no ambiguo:

```php
$row->get('name');
```

---

# 68. Ambiguous label

Si existen dos columnas:

```text
id
id
```

entonces:

```php
$row->get('id');
```

deberá fallar con:

```text
AmbiguousResultColumnException
```

salvo política explícita.

---

# 69. No last-column-wins

Nunca:

```php
[
    'id' => $secondId,
]
```

perdiendo silenciosamente la primera columna.

---

# 70. ResultColumnIndex

Podrá existir:

```text
label
→
list<ColumnIndex>
```

para detectar ambigüedad.

---

# 71. ResultRow immutability

Una fila materializada será preferiblemente:

```text
immutable
```

---

# 72. Why

Evita que:

```text
consumer A
```

modifique valores observados posteriormente por:

```text
consumer B
```

---

# 73. Driver result adapter

Cada driver deberá proporcionar:

```php
interface DriverResultAdapter
{
    public function adapt(
        DriverExecutionResult $result,
        ResultAdaptationContext $context,
    ): DatabaseResult;
}
```

---

# 74. Core independence

El core no deberá contener:

```php
if ($result instanceof PDOStatement) {
    ...
}
```

---

# 75. PDO integration

Podrá existir:

```text
PdoDriverResultAdapter
```

en la integración correspondiente.

---

# 76. Platform ≠ Driver Result Adapter

```text
Platform
```

describe semántica/capacidades del DBMS.

```text
DriverResultAdapter
```

describe cómo obtener el resultado desde el driver concreto.

---

# 77. DriverResultAdapter ≠ Hydrator

El adapter produce:

```text
DatabaseResult
```

no:

```text
User[]
```

---

# 78. ResultAdaptationContext

```php
final readonly class ResultAdaptationContext
{
    public function __construct(
        public ResultContract $contract,
        public DriverIdentity $driver,
        public PlatformIdentity $platform,
        public DriverResultContract $driverContract,
        public ResultConversionRegistry $conversions,
        public ResultPolicy $policy,
        public ResultBudget $budget,
    ) {}
}
```

---

# 79. Context is explicit

No acceder a:

```text
current tenant
current EntityManager
current HTTP request
global timezone
global locale
```

ocultamente.

---

# 80. Result value type system

VoltStack deberá mantener:

```text
Driver Result Type
≠
Database Semantic Type
≠
PHP Runtime Type
```

---

# 81. Example

PostgreSQL puede devolver:

```text
Driver:
"123.45"
```

para un `NUMERIC`.

El resultado semántico podría convertirse a:

```text
Decimal
```

sin pasar por `float`.

---

# 82. Decimal invariant

```text
NUMERIC/DECIMAL
```

no deberá sufrir pérdida silenciosa de precisión.

---

# 83. Integer result

Debe validarse contra:

```text
PHP integer range
semantic integer range
platform representation
```

---

# 84. Big integers

Si un valor excede `PHP_INT_MAX`, podrá representarse como:

```text
BigInteger
canonical integer string
```

según el Type System.

Nunca deberá truncarse.

---

# 85. Boolean result

Ejemplos driver:

```text
1
0
"1"
"0"
true
false
"t"
"f"
```

La conversión dependerá del driver/platform type mapping.

---

# 86. No truthiness conversion

Nunca usar como regla universal:

```php
(bool) $driverValue;
```

---

# 87. NULL result

SQL `NULL` deberá convertirse en:

```php
null
```

o wrapper tipado equivalente.

---

# 88. SQL NULL ≠ string NULL

```text
NULL
≠
"NULL"
```

---

# 89. JSON result

Debe conservarse la diferencia:

```text
SQL NULL
```

vs:

```json
null
```

---

# 90. JSON conversion

Puede producir:

```text
JsonValue
array
object
scalar
```

según política del Type System.

---

# 91. Invalid JSON

Si una columna declarada como JSON contiene datos inválidos:

```text
ResultConversionException
```

salvo política explícita de raw access.

---

# 92. Binary result

Datos binarios no deberán pasar por:

```text
text encoding conversion
```

automática.

---

# 93. Date/time results

El Result System deberá distinguir cuando corresponda:

```text
DATE
TIME
DATETIME
TIMESTAMP
TIMESTAMP WITH TIME ZONE
INSTANT
```

---

# 94. No hidden timezone

La conversión temporal deberá utilizar política explícita.

---

# 95. SQLite result typing

SQLite requiere especial cuidado porque:

```text
Storage Class
≠
Declared Type
≠
Type Affinity
≠
VoltStack QueryType
```

---

# 96. SQLite result resolution

Podrá combinar:

```text
expected result type
+
schema metadata
+
driver-reported storage representation
```

sin depender de heurísticas inseguras.

---

# 97. MySQL/MariaDB result differences

Los adapters deberán respetar diferencias reales entre:

```text
MySQL
MariaDB
```

sin tratarlos como una plataforma indistinguible.

---

# 98. PostgreSQL result types

Podrán existir converters especializados para:

```text
UUID
JSON
JSONB
ARRAY
NUMERIC
DATE/TIME
BYTEA
ENUM
custom types
```

mediante registries tipados.

---

# 99. Result conversion registry

```text
ResultValueConverterRegistry
```

será responsable de resolver converters.

---

# 100. Converter contract

```php
interface ResultValueConverter
{
    public function convert(
        mixed $driverValue,
        ResultValueConversionContext $context,
    ): ResultValue;
}
```

---

# 101. Registry resolution

Idealmente:

```text
SemanticTypeId
+
PlatformScope
+
DriverScope
+
DriverResultType
+
Capabilities
```

---

# 102. No broad magic converter

Evitar:

```php
supports(mixed $value): bool
```

como mecanismo principal.

Preferir identidad tipada.

---

# 103. Unknown type

Si no existe converter y el raw value puede representarse de manera segura:

```text
RawResultValue
```

puede ser permitido bajo contrato explícito.

---

# 104. RawResultValue

No deberá convertirse silenciosamente en tipo semántico conocido.

---

# 105. Result lifecycle

Un resultado puede atravesar:

```text
CREATED
   ↓
OPEN
   ↓
CONSUMING
   ↓
EXHAUSTED
   ↓
CLOSED
```

---

# 106. Alternative failure state

```text
OPEN
 ↓
FAILED
 ↓
CLOSED
```

---

# 107. ResultState

```php
enum ResultState
{
    case CREATED;
    case OPEN;
    case CONSUMING;
    case EXHAUSTED;
    case CLOSED;
    case FAILED;
}
```

---

# 108. Buffered result lifecycle

Un resultado completamente materializado puede:

```text
OPEN
↓
EXHAUSTED
```

desde el punto de vista del driver mientras sigue siendo legible desde memoria.

Por ello deberán distinguirse:

```text
driver resource state
```

y:

```text
logical result accessibility
```

cuando sea necesario.

---

# 109. Result resource ownership

Todo resultado deberá conocer quién posee:

```text
driver cursor
driver result handle
temporary buffers
stream resources
```

---

# 110. ResultOwnership

```php
enum ResultOwnership
{
    case FRAMEWORK;
    case DRIVER;
    case BORROWED;
    case TRANSFERRED;
}
```

---

# 111. Default

Los handles obtenidos por Execution Engine normalmente serán propiedad de VoltStack hasta su cierre/liberación.

---

# 112. close()

`close()` deberá ser:

```text
safe
```

e idealmente:

```text
idempotent
```

---

# 113. Resource cleanup

```text
Result.close()
    ↓
close cursor
    ↓
release driver result
    ↓
release temporary resources
    ↓
notify statement lifecycle
```

---

# 114. Result ↔ PreparedStatement lifecycle

Un cursor abierto puede impedir reutilizar el statement.

Por ello:

```text
Result lifecycle
```

y:

```text
PreparedStatement lease lifecycle
```

deberán coordinarse.

---

# 115. Statement cannot be released too early

Incorrecto:

```text
execute
↓
return cursor
↓
release statement
↓
cursor still reading
```

si el driver requiere que el statement permanezca vivo.

---

# 116. Resource dependency graph

Conceptualmente:

```text
ConnectionLease
      ↓
PreparedStatementLease
      ↓
DriverResultHandle
      ↓
ResultCursor
```

La liberación deberá respetar dependencias.

---

# 117. Resource lifetime

```text
Lifetime(child)
⊆
Lifetime(parent)
```

cuando el child dependa del parent.

---

# 118. Buffered optimization

Si todas las filas se materializan:

```text
DriverResultHandle
```

puede cerrarse antes, permitiendo liberar statement/connection según política.

---

# 119. Cursor result

En cambio:

```text
cursor-backed result
```

puede retener recursos hasta:

```text
EXHAUSTED
```

o:

```text
CLOSED
```

---

# 120. Result consumption model

Podrán existir:

```text
SINGLE_PASS
REPLAYABLE
RANDOM_ACCESS
```

---

# 121. Cursor

Normalmente:

```text
SINGLE_PASS
```

---

# 122. Buffered result

Normalmente:

```text
REPLAYABLE
```

y potencialmente:

```text
RANDOM_ACCESS
```

---

# 123. No implicit replay

Un cursor agotado no deberá reiniciarse automáticamente ejecutando nuevamente la query.

---

# 124. Critical invariant

```text
Result consumption
```

nunca deberá provocar:

```text
implicit query re-execution
```

---

# 125. ResultCollection convenience API

Una capa de conveniencia puede ofrecer:

```php
$result->all();
```

pero deberá hacer explícito que puede materializar datos.

---

# 126. Memory safety

`all()` sobre un cursor enorme puede superar memoria.

Podrán existir:

```text
ResultMaterializationPolicy
ResultBudget
```

---

# 127. ResultBudget

Debe limitar cuando aplique:

```text
buffered rows
buffered bytes
column count
row width
conversion work
LOB bytes
metadata size
result sets
```

---

# 128. Unknown size

```text
unknown result size
≠
safe to buffer
```

---

# 129. Materialization budget

Antes o durante buffering:

```text
if budget exceeded
→ ResultBudgetException
```

sin consumir ilimitadamente.

---

# 130. Partial materialization

Por defecto un `BufferedResult` no deberá exponerse como completo si el buffering falló a mitad.

---

# 131. Atomic buffered result

```text
Buffering success
→ publish BufferedResult

Buffering failure
→ cleanup + exception
```

---

# 132. Streaming semantics

Para streaming:

```text
partial consumption
```

sí es parte normal del modelo.

---

# 133. Cancellation

Un resultado cursor/stream deberá responder a cancellation cuando sea soportado.

---

# 134. Cancellation flow

```text
Cancellation requested
        ↓
stop result consumption
        ↓
driver cancellation if supported
        ↓
close result resources
        ↓
statement/connection state classification
```

---

# 135. Cancellation ≠ ordinary exhaustion

Debe conservarse estado diferenciado en diagnostics/telemetry.

---

# 136. Result errors

Jerarquía propuesta:

```text
DatabaseResultException
├── ResultContractException
├── ResultKindMismatchException
├── ResultShapeException
├── ResultColumnException
├── AmbiguousResultColumnException
├── MissingResultColumnException
├── ResultCardinalityException
├── ScalarCardinalityException
├── ResultConversionException
├── ResultTypeException
├── ResultEncodingException
├── ResultJsonException
├── ResultDateTimeException
├── ResultOverflowException
├── ResultLifecycleException
├── ResultAlreadyClosedException
├── ResultConsumptionException
├── ResultResourceException
├── ResultBudgetException
├── ResultCancellationException
├── MultipleResultSetException
├── UnsupportedDriverResultException
├── DriverResultException
├── ResultExtensionException
├── ResultSecurityException
└── ResultInvariantException
```

---

# 137. Result error normalization

Los errores deberán conservar:

```text
result kind
result state
driver identity
platform identity
statement impact
connection impact
safe metadata
cause
```

---

# 138. Row values in errors

No deberán incluirse indiscriminadamente.

---

# 139. Sensitive result data

Columnas podrán estar clasificadas:

```text
PUBLIC
INTERNAL
SENSITIVE
SECRET
```

---

# 140. Sources of sensitivity

Puede provenir de:

```text
query metadata
schema metadata
security metadata
explicit result contract
```

---

# 141. Result sensitivity

Ejemplo:

```text
password_hash
access_token
refresh_token
private_key
```

deberá marcarse como:

```text
SECRET
```

cuando metadata confiable lo indique.

---

# 142. Redaction

Debug:

```text
access_token = [REDACTED]
```

---

# 143. No result-value telemetry by default

Telemetry deberá observar:

```text
row count
column count
bytes
duration
conversion failures
consumption mode
```

no contenido de filas.

---

# 144. Row count semantics

Para streaming:

```text
row count
```

puede ser desconocido hasta agotamiento.

---

# 145. ResultCount

Podrá representar:

```text
KNOWN
UNKNOWN
PARTIAL
```

---

# 146. Unknown ≠ zero

Nunca:

```text
unknown row count
→ 0
```

---

# 147. Telemetry events

Sugeridos:

```text
ResultAdaptationStarted
ResultAdapted
ResultOpened
ResultConsumptionStarted
ResultRowRead
ResultMaterializationStarted
ResultMaterialized
ResultExhausted
ResultClosed
ResultConversionFailed
ResultBudgetExceeded
ResultCancelled
ResultFailed
```

---

# 148. High-frequency events

`ResultRowRead` puede ser demasiado costoso.

Deberá poder:

```text
sample
aggregate
disable
```

---

# 149. Metrics

Ejemplos:

```text
db.result.count
db.result.rows
db.result.bytes
db.result.duration
db.result.conversion.duration
db.result.failures
db.result.buffered.rows
db.result.streaming.rows
db.result.open.duration
```

---

# 150. Metrics cardinality

No utilizar como labels:

```text
row value
customer id
email
token
arbitrary SQL
```

---

# 151. SQL relation

Result telemetry podrá correlacionarse mediante:

```text
CompiledCommandFingerprint
ExecutionId
StatementExecutionId
```

sin incluir runtime values.

---

# 152. Result identity

Podrá existir:

```text
ResultId
```

operation-scoped para diagnostics.

---

# 153. ResultId ≠ query fingerprint

```text
ResultId
=
runtime instance
```

```text
QueryFingerprint
=
structural identity
```

---

# 154. Persistent runtime safety

Después de:

```text
Result.close()
```

no deberán quedar en singletons:

```text
current rows
current cursor
current result handle
current connection
current statement
sensitive result values
```

---

# 155. FrankenPHP

El mismo worker puede procesar:

```text
Request A
↓
Result A
↓
close/reset

Request B
↓
Result B
```

`Result A` no deberá contaminar `Result B`.

---

# 156. RoadRunner/OpenSwoole

La misma regla aplica a workers persistentes y ejecución concurrente.

---

# 157. Coroutine isolation

```text
Coroutine A
→ ResultContext A

Coroutine B
→ ResultContext B
```

No deberá existir state leakage.

---

# 158. Shared immutable state

Sí puede compartirse:

```text
Result converter registries
type descriptors
immutable driver result contracts
result policies
```

---

# 159. Operation-local state

Debe ser local:

```text
cursor
result handle
row buffer
conversion session
resource registry
current row
result counters
```

---

# 160. ResultSession

Podrá existir:

```php
final class ResultSession
{
    // operation-scoped state
    // resources
    // budget counters
    // diagnostics
    // conversion context
}
```

---

# 161. ResultSession lifecycle

```text
create
↓
adapt
↓
consume
↓
exhaust / cancel / fail
↓
cleanup
↓
dispose
```

---

# 162. Extension model

El Result System deberá ser extensible sin permitir arbitrary mutation.

---

# 163. Extension categories

Podrán existir:

```text
ResultValueConverterExtension
ResultAdapterExtension
ResultTypeExtension
ResultMetadataExtension
ResultDiagnosticExtension
ExtensionResultProvider
```

---

# 164. Result extension principle

```text
Result Extensibility
≠
Arbitrary Driver Result Mutation
```

---

# 165. Extension contract

```php
interface ResultSystemExtension
{
    public function descriptor(): ResultSystemExtensionDescriptor;

    public function contributions(): ResultSystemContributions;
}
```

---

# 166. Frozen registry

Las extensiones se resolverán durante bootstrap:

```text
discover
↓
validate
↓
resolve dependencies
↓
detect conflicts
↓
compose registries
↓
freeze
```

---

# 167. No runtime registration

Una request no podrá cambiar globalmente cómo se convierten resultados para las siguientes.

---

# 168. No last-wins

Dos converters exclusivos para el mismo:

```text
SemanticType
+
Platform
+
Driver
```

deberán generar conflicto.

---

# 169. Extension security

Una extensión no deberá:

```text
execute hidden queries
open hidden connections
hydrate ORM entities
log sensitive values
bypass result budget
retain result handles globally
```

---

# 170. Raw result access

VoltStack puede ofrecer escape hatch:

```text
RawDriverResult
```

pero deberá ser explícito.

---

# 171. Raw result caveat

Usarlo implica salir parcialmente de las garantías de portabilidad del Result System.

---

# 172. Raw access ownership

Aun con raw access, ownership/lifecycle deberá permanecer definido.

---

# 173. Result API levels

Podrían existir tres niveles:

```text
Level 1
DatabaseResult

Level 2
ResultCursor / BufferedResult / StreamingResult

Level 3
Hydration / ORM
```

---

# 174. Public API example

```php
$result = $database->execute($query);

foreach ($result->rows() as $row) {
    echo $row->get('name');
}
```

La API pública final se definirá posteriormente, pero la arquitectura deberá permitir ergonomía sin romper boundaries.

---

# 175. Scalar convenience

Ejemplo futuro:

```php
$count = $database
    ->execute($query)
    ->scalar()
    ->int();
```

sin convertir `DatabaseResult` en una API mágica universal.

---

# 176. Affected rows convenience

```php
$affected = $result
    ->affectedRows()
    ->value();
```

---

# 177. Result access mismatch

Si se intenta:

```php
$result->affectedRows();
```

sobre un `RowResult`, deberá producir error tipado o no estar disponible mediante la interfaz concreta.

Preferible:

```text
type-safe interfaces
```

sobre runtime magic.

---

# 178. Result covariance

Interfaces especializadas permiten:

```text
DatabaseResult
    ↑
RowResult
    ↑
ReturningResult
```

sin agregar métodos irrelevantes al contrato base.

---

# 179. Result disposal

El framework deberá intentar cleanup automático al finalizar operation scope.

Pero:

```text
automatic cleanup
```

no elimina la necesidad de:

```text
explicit close()
```

para recursos de larga duración.

---

# 180. Destructor

No deberá dependerse exclusivamente de:

```php
__destruct()
```

para liberar recursos críticos.

---

# 181. Why

En runtimes persistentes:

```text
GC timing
```

no equivale a:

```text
request lifecycle
```

---

# 182. Request-scope cleanup registry

Los resultados abiertos podrán registrarse en:

```text
DatabaseResourceScope
```

para cierre garantizado al terminar la operación/request.

---

# 183. Leak detection

En modo debug:

```text
Request finished
+
Open Result Handles > 0
→ ResourceLeakDiagnostic
```

---

# 184. Production cleanup

En producción:

```text
close safely
+
telemetry
```

sin revelar datos.

---

# 185. Transaction relationship

Un cursor puede depender de una transacción activa.

---

# 186. Transaction-bound result

Metadata:

```text
requiresTransactionLifetime = true
```

podrá impedir commit/release prematuro.

---

# 187. Transaction system ownership

Result System no decide:

```text
commit
rollback
```

Sólo declara sus dependencias/lifecycle.

---

# 188. Connection relationship

Algunos resultados requieren mantener connection lease abierto.

---

# 189. Connection release invariant

```text
DependentResultOpen
⇒
ConnectionNotReusable
```

cuando el driver contract así lo requiera.

---

# 190. Detached buffered result

Después de materializar:

```text
DriverResult
↓
Buffer
↓
close driver handle
↓
release statement
↓
release connection
```

el resultado podrá quedar:

```text
DETACHED
```

de recursos DB.

---

# 191. ResultResourceMode

```php
enum ResultResourceMode
{
    case ATTACHED;
    case DETACHED;
}
```

---

# 192. ATTACHED

Depende de recursos del driver.

---

# 193. DETACHED

Toda información necesaria ya reside en estructuras VoltStack.

---

# 194. Result consumption guarantees

El contrato deberá poder indicar:

```text
single-pass
replayable
seekable
buffered
streaming
known-size
```

mediante capabilities.

---

# 195. ResultCapabilitySet

Ejemplo:

```text
REPLAYABLE
SEEKABLE
STREAMING
MULTIPLE_RESULT_SETS
COLUMN_METADATA
ROW_COUNT_KNOWN
DETACHABLE
CANCELLABLE
```

---

# 196. Capability checks

Preferir:

```php
$result->capabilities()->supports(
    ResultCapability::REPLAYABLE
);
```

sobre:

```php
if ($driver === 'mysql') {
}
```

---

# 197. Driver limitations

Si una capacidad no existe:

```text
fail explicitly
```

o usar una estrategia ya autorizada.

No simular comportamiento con queries ocultas.

---

# 198. Result ordering

El Result System preservará el orden producido por la ejecución.

---

# 199. No reordering

Nunca:

```text
sort rows automatically
```

por comodidad.

---

# 200. Duplicate rows

Deben preservarse.

```text
bag semantics
```

continúan aplicando.

---

# 201. No deduplication

Result System no hará:

```text
array_unique(rows)
```

---

# 202. NULL ordering

No es responsabilidad del Result System.

El orden ya fue determinado por la query/DBMS.

---

# 203. Column aliases

Se conservarán según el result contract y driver metadata.

---

# 204. Case folding

No deberá normalizarse indiscriminadamente:

```text
UserID
→ userid
```

porque puede destruir identidad.

---

# 205. Column lookup policy

Podrá definirse:

```text
EXACT
PLATFORM_NORMALIZED
CASE_INSENSITIVE
```

pero el default deberá evitar ambigüedad.

---

# 206. Recommended default

```text
EXACT
```

para identidad interna.

Las APIs ergonómicas podrán utilizar metadata adicional.

---

# 207. Column index access

Siempre deberá existir cuando haya row result:

```php
$row->at($index);
```

---

# 208. Column label access

Disponible cuando:

```text
label exists
∧
label is unambiguous
```

---

# 209. Result shape fingerprint

Puede existir:

```text
ResultShapeFingerprint
```

basado en:

```text
column count
column identities
expected types
result contract version
```

---

# 210. Runtime values excluded

Nunca incluir valores de filas en:

```text
ResultShapeFingerprint
```

---

# 211. Result contract fingerprint

Podrá utilizarse para verificar compatibilidad entre:

```text
CompiledDatabaseCommand
```

y:

```text
DatabaseResult
```

---

# 212. Cache boundary

El Result System no deberá asumir que:

```text
Result Cache
```

existe.

El cache se diseñará posteriormente en el bloque correspondiente.

---

# 213. Result ≠ Cache Entry

```text
Live DatabaseResult
```

puede contener recursos no serializables.

---

# 214. Cacheable representation

Si posteriormente se requiere caching:

```text
DatabaseResult
↓
Detached Serializable Result Representation
↓
Cache
```

deberá ser una transformación explícita.

---

# 215. Cursor cannot be cached directly

```text
Driver Cursor
```

nunca deberá almacenarse en cache.

---

# 216. Serialization

Resultados detached podrán eventualmente ser serializables si:

```text
types are serializable
resources = none
security policy allows it
```

---

# 217. Security before serialization

Datos sensibles deberán obedecer políticas del futuro Result Cache System.

---

# 218. Result materialization

Debe distinguirse:

```text
materialize rows
```

de:

```text
hydrate entities
```

---

# 219. Formula

```text
Materialization
=
Driver Rows
→
VoltStack ResultRows
```

---

# 220. Hydration

```text
Hydration
=
VoltStack ResultRows
→
Domain Objects / Entities
```

---

# 221. Architectural separation

```text
Execution
      ↓
Result
      ↓
Hydration
      ↓
ORM
```

Nunca:

```text
Execution
↓
ORM Entity
```

directamente.

---

# 222. Result metadata source hierarchy

Metadata puede provenir de:

```text
Compiled Result Contract
Driver Result Metadata
Platform Metadata
Schema Metadata Snapshot
Type Mapping
Extension Metadata
```

---

# 223. Authority

Cuando exista conflicto, deberá existir una política explícita.

No:

```text
last metadata wins
```

---

# 224. Recommended authority model

Para semántica esperada:

```text
Compiled Result Contract
>
validated semantic metadata
>
driver metadata
>
inference
```

sin ignorar incompatibilidades reales del driver.

---

# 225. Metadata conflict

Ejemplo:

```text
expected INTEGER
driver reports BLOB
```

no deberá resolverse silenciosamente.

---

# 226. Metadata conflict result

```text
ResultMetadataConflictException
```

o error de type/shape según categoría.

---

# 227. Unknown metadata

Cuando no sea necesaria para correctness:

```text
UNKNOWN
```

será válido.

---

# 228. Metadata laziness

Algunos drivers obtienen metadata de forma costosa.

Podrá existir:

```text
LazyResultMetadata
```

sólo si su evaluación:

- no ejecuta queries ocultas;
- no altera el cursor;
- no rompe determinismo;
- respeta lifecycle.

---

# 229. Preferred approach

Metadata necesaria para correctness deberá resolverse antes de exponer el resultado.

Metadata puramente diagnóstica podrá ser lazy.

---

# 230. Result value laziness

Evitar convertir todos los valores antes de que sean consumidos en streaming.

---

# 231. Row conversion

En cursor mode:

```text
fetch native row
↓
convert row
↓
return ResultRow
```

---

# 232. Buffered mode

```text
fetch all native rows
↓
convert
↓
store ResultRows
↓
release driver resources
```

---

# 233. Conversion failure mid-stream

Si falla fila 1,024:

```text
rows 1..1023
```

ya pudieron haber sido consumidas.

El error deberá reflejar:

```text
partial consumption
```

---

# 234. Streaming atomicity

No puede prometer atomicidad de consumo de todo el dataset.

---

# 235. Buffered atomicity

Sí puede ofrecer:

```text
all rows materialized successfully
or
no BufferedResult published
```

---

# 236. Result failure metadata

Podrá incluir:

```text
rowsConsumed
bytesConsumed
currentColumn
conversionStage
resourceImpact
```

sin valores sensibles.

---

# 237. Multiple result set lifecycle

Cada result set deberá consumirse/cerrarse según el driver contract antes de avanzar al siguiente.

---

# 238. Result set ordering

Debe conservarse exactamente.

---

# 239. Empty result set

```text
0 rows
```

sigue siendo un `RowResult`.

No:

```text
NoResult
```

---

# 240. Critical distinction

```text
RowResult(rows = 0)
≠
NoResult
```

---

# 241. Empty scalar

Igualmente:

```text
ScalarResult(NULL)
```

puede diferir de:

```text
no scalar row
```

---

# 242. SQL aggregate example

```sql
SELECT MAX(age) FROM users WHERE false
```

puede producir:

```text
one row
one column
NULL
```

No:

```text
zero rows
```

---

# 243. Result cardinality model

Puede representar:

```text
ZERO
ONE
MANY
UNKNOWN
```

según información disponible.

---

# 244. Cardinality is runtime fact

No deberá confundirse con:

```text
planner cardinality estimate
```

---

# 245. Estimated vs actual

```text
EstimatedCardinality
≠
ActualResultCardinality
```

---

# 246. Telemetry feedback

Posteriormente:

```text
ActualResultCardinality
↓
Telemetry
↓
Statistics Feedback
↓
future optimization
```

pero no directamente:

```text
current Result System
→ mutate current optimizer
```

---

# 247. No feedback loops inside execution

El Result System no modificará el plan actual basado en filas ya observadas.

---

# 248. Result security boundary

El Result System deberá asumir que los datos provenientes de la DB pueden contener:

```text
untrusted text
binary payloads
malformed JSON
unexpected encoding
oversized values
```

---

# 249. Database data is not automatically trusted

Especialmente para:

```text
legacy databases
external databases
user-generated data
```

---

# 250. Output escaping

El Result System no hará:

```text
HTML escaping
JavaScript escaping
URL escaping
```

Eso pertenece a capas de presentación.

---

# 251. Result security ≠ presentation escaping

```text
Database Result
→ raw semantic value
```

La vista decide su escaping contextual.

---

# 252. Encryption

Si existe un Database Encryption Type:

```text
encrypted DB representation
↓
type converter / security integration
↓
semantic result value
```

deberá ser explícito.

---

# 253. Encryption ≠ ORM hydration

Sigue siendo transformación de representación.

---

# 254. Result diagnostics

Un debug descriptor podría mostrar:

```text
Result ID
Kind
State
Consumption mode
Resource mode
Columns
Expected types
Driver types
Rows consumed
Bytes consumed
Open duration
```

---

# 255. Sensitive diagnostics

No mostrar:

```text
actual secret column values
```

por defecto.

---

# 256. Explainability

Debe ser posible responder:

```text
¿Por qué esta columna se convirtió a este tipo?
```

mediante:

```text
ResultConversionTrace
```

en debug.

---

# 257. ResultConversionTrace

Podrá incluir:

```text
source driver type
expected semantic type
converter selected
platform
driver contract
conversion policy
```

---

# 258. No secrets in trace

El valor concreto podrá omitirse/redactarse.

---

# 259. Determinism

Dado:

```text
same native value
same driver metadata
same expected type
same conversion registry
same policy
```

la conversión deberá producir:

```text
same semantic result
```

salvo tipos explícitamente no deterministas.

---

# 260. No wall clock

Converters no deberán depender de:

```text
current time
```

ocultamente.

---

# 261. No random

Converters no deberán usar randomness oculto.

---

# 262. No service locator

Result conversion no deberá resolver servicios arbitrariamente desde container global durante el hot path.

---

# 263. Pre-resolved conversion plan

Puede existir:

```text
CompiledResultConversionPlan
```

---

# 264. CompiledResultConversionPlan

```php
final readonly class CompiledResultConversionPlan
{
    /**
     * @param list<CompiledColumnConversion> $columns
     */
    public function __construct(
        public array $columns,
        public ResultConversionPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 265. Column conversion instruction

Puede contener:

```text
column ordinal
ResultColumnId
expected type
converter id
nullable
sensitivity
conversion flags
```

---

# 266. Benefit

Hot path:

```text
native row
↓
pre-resolved converter per column
↓
ResultRow
```

evitando resolución repetitiva.

---

# 267. Complexity

Para:

```text
r = rows
c = columns
```

la conversión normal deberá aproximarse a:

```text
O(r × c)
```

más el costo de conversiones específicas.

---

# 268. Avoid metadata lookup per cell

No realizar:

```text
registry scan
```

para cada valor.

Resolver por columna cuando sea posible.

---

# 269. Result row allocation

En streaming de alto rendimiento podrán explorarse representaciones optimizadas, pero nunca sacrificando:

```text
correctness
ownership
isolation
type safety
```

---

# 270. Reusable mutable row object

Evitar devolver el mismo objeto mutable en cada iteración:

```php
foreach ($cursor as $row) {
    $saved[] = $row;
}
```

si después todos apuntan a la última fila.

---

# 271. Stable row invariant

Cada `ResultRow` expuesto al consumidor deberá conservar sus valores durante su lifetime.

---

# 272. Low-allocation internal views

Si se implementan, deberán ser explícitas y no escapar de su scope.

---

# 273. Recommended public model

```text
immutable ResultRow
```

para la API pública.

---

# 274. Namespace propuesto

```text
VoltStack\Quantum\Database\Execution\Result
```

---

# 275. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Result/
                ├── Contract/
                │   ├── DatabaseResult.php
                │   ├── RowResult.php
                │   ├── ReturningResult.php
                │   ├── MultipleResultSetResult.php
                │   ├── ExtensionResult.php
                │   ├── DriverResultAdapter.php
                │   └── ResultValueConverter.php
                │
                ├── Result/
                │   ├── DefaultRowResult.php
                │   ├── ScalarResult.php
                │   ├── AffectedRowsResult.php
                │   ├── DefaultReturningResult.php
                │   ├── NoResult.php
                │   └── DefaultMultipleResultSetResult.php
                │
                ├── Row/
                │   ├── ResultRow.php
                │   ├── ResultRowMetadata.php
                │   ├── ResultColumnIndex.php
                │   └── ResultValue.php
                │
                ├── Metadata/
                │   ├── ResultMetadata.php
                │   ├── ResultColumnMetadata.php
                │   ├── ResultValueMetadata.php
                │   ├── ResultMetadataCertainty.php
                │   ├── MutationResultMetadata.php
                │   └── GeneratedValueMetadata.php
                │
                ├── Shape/
                │   ├── ResultShape.php
                │   ├── ResultColumnShape.php
                │   ├── ResultShapeFingerprint.php
                │   └── ResultShapeValidator.php
                │
                ├── Type/
                │   ├── ResultTypeResolver.php
                │   ├── DriverResultType.php
                │   ├── ResultSemanticType.php
                │   └── RawResultValue.php
                │
                ├── Conversion/
                │   ├── ResultValueConverterRegistry.php
                │   ├── ResultValueConversionContext.php
                │   ├── CompiledResultConversionPlan.php
                │   ├── CompiledColumnConversion.php
                │   ├── NullResultConverter.php
                │   ├── BooleanResultConverter.php
                │   ├── IntegerResultConverter.php
                │   ├── DecimalResultConverter.php
                │   ├── StringResultConverter.php
                │   ├── BinaryResultConverter.php
                │   ├── JsonResultConverter.php
                │   ├── UuidResultConverter.php
                │   ├── EnumResultConverter.php
                │   └── DateTimeResultConverter.php
                │
                ├── Adaptation/
                │   ├── ResultAdaptationContext.php
                │   ├── DefaultResultAdapter.php
                │   └── DriverResultContract.php
                │
                ├── Lifecycle/
                │   ├── ResultState.php
                │   ├── ResultLifecycle.php
                │   ├── ResultOwnership.php
                │   ├── ResultResourceMode.php
                │   └── ResultResourceRegistry.php
                │
                ├── Consumption/
                │   ├── ResultConsumptionMode.php
                │   ├── ResultCapability.php
                │   ├── ResultCapabilitySet.php
                │   ├── ResultCount.php
                │   └── ResultMaterializationPolicy.php
                │
                ├── Scalar/
                │   ├── ScalarCardinality.php
                │   └── ScalarResultPolicy.php
                │
                ├── Mutation/
                │   ├── AffectedRowCount.php
                │   ├── AffectedRowSemantics.php
                │   └── GeneratedIdentifier.php
                │
                ├── Multiple/
                │   └── ResultSetSequence.php
                │
                ├── Session/
                │   ├── ResultSession.php
                │   └── ResultId.php
                │
                ├── Budget/
                │   └── ResultBudget.php
                │
                ├── Security/
                │   ├── ResultSensitivity.php
                │   ├── ResultRedactor.php
                │   └── ResultSecurityPolicy.php
                │
                ├── Diagnostic/
                │   ├── ResultDiagnostic.php
                │   ├── ResultConversionTrace.php
                │   └── ResultLeakDiagnostic.php
                │
                ├── Telemetry/
                │   ├── ResultTelemetry.php
                │   ├── ResultObserver.php
                │   └── ResultMetrics.php
                │
                ├── Extension/
                │   ├── ResultSystemExtension.php
                │   ├── ResultSystemExtensionDescriptor.php
                │   ├── ResultSystemContributions.php
                │   └── ResultExtensionRegistry.php
                │
                └── Exception/
                    ├── DatabaseResultException.php
                    ├── ResultContractException.php
                    ├── ResultKindMismatchException.php
                    ├── ResultShapeException.php
                    ├── ResultColumnException.php
                    ├── AmbiguousResultColumnException.php
                    ├── MissingResultColumnException.php
                    ├── ResultCardinalityException.php
                    ├── ScalarCardinalityException.php
                    ├── ResultConversionException.php
                    ├── ResultTypeException.php
                    ├── ResultMetadataConflictException.php
                    ├── ResultLifecycleException.php
                    ├── ResultAlreadyClosedException.php
                    ├── ResultConsumptionException.php
                    ├── ResultResourceException.php
                    ├── ResultBudgetException.php
                    ├── ResultCancellationException.php
                    ├── UnsupportedDriverResultException.php
                    ├── DriverResultException.php
                    ├── ResultExtensionException.php
                    ├── ResultSecurityException.php
                    └── ResultInvariantException.php
```

---

# 276. Dependency direction

```text
Statement Execution
       ↓
Result System
       ↓
Driver Result Adapter
       ↓
Driver
```

Para tipos:

```text
Database Type System
       ↓
Result System
```

Posteriormente:

```text
Result System
       ↓
Hydration System
       ↓
ORM
```

---

# 277. Forbidden dependency direction

Nunca:

```text
Result System
       ↓
ORM EntityManager
```

ni:

```text
Driver
  ↓
ORM Hydrator
```

---

# 278. Architectural invariants

## DB-RESULT-001

`DatabaseResult` será distinto del resultado nativo del driver.

## DB-RESULT-002

`DatabaseResult` será distinto de `ResultCursor`.

## DB-RESULT-003

`DatabaseResult` será distinto de una entidad hidratada.

## DB-RESULT-004

`DatabaseResult` será distinto de una colección ORM.

## DB-RESULT-005

`ResultContract` describirá el resultado esperado, no el resultado runtime.

## DB-RESULT-006

El resultado runtime deberá satisfacer el `ResultContract`.

## DB-RESULT-007

Result System no generará SQL.

## DB-RESULT-008

Result System no preparará statements.

## DB-RESULT-009

Result System no realizará parameter binding.

## DB-RESULT-010

Result System no ejecutará queries adicionales ocultas.

## DB-RESULT-011

Result System no realizará ORM hydration.

## DB-RESULT-012

Result System no administrará IdentityMap.

## DB-RESULT-013

Result System no administrará UnitOfWork.

## DB-RESULT-014

Result kinds serán explícitos.

## DB-RESULT-015

No se utilizará un array universal para representar todos los result kinds.

## DB-RESULT-016

`RowResult` no implicará buffering.

## DB-RESULT-017

`RowResult` podrá respaldarse por cursor o buffer.

## DB-RESULT-018

`ResultRow` preservará el orden de columnas.

## DB-RESULT-019

Result rows no dependerán exclusivamente de associative arrays.

## DB-RESULT-020

Columnas duplicadas no se sobrescribirán.

## DB-RESULT-021

Column identity será distinta de column label.

## DB-RESULT-022

Column label será distinto de ordinal position.

## DB-RESULT-023

Acceso por posición será estable.

## DB-RESULT-024

Acceso por label ambiguo fallará explícitamente.

## DB-RESULT-025

No existirá last-column-wins.

## DB-RESULT-026

Driver value será distinto de normalized result value.

## DB-RESULT-027

Result conversion será distinta de entity hydration.

## DB-RESULT-028

Scalar result sólo se producirá cuando el contrato lo declare.

## DB-RESULT-029

Scalar cardinality será validable.

## DB-RESULT-030

Zero rows será distinto de scalar NULL.

## DB-RESULT-031

Affected rows conservará semántica del driver/platform.

## DB-RESULT-032

Matched rows será distinto de changed rows cuando el DBMS lo distinga.

## DB-RESULT-033

`ReturningResult` será semánticamente distinto de un SELECT ordinario.

## DB-RESULT-034

`RETURNING` será distinto de `lastInsertId`.

## DB-RESULT-035

`NoResult` será distinto de `null`.

## DB-RESULT-036

Un RowResult vacío será distinto de NoResult.

## DB-RESULT-037

Multiple result sets serán capability-driven.

## DB-RESULT-038

Unknown driver results fallarán explícitamente.

## DB-RESULT-039

Result metadata tendrá certainty explícita cuando sea necesario.

## DB-RESULT-040

Unknown metadata no se convertirá en false.

## DB-RESULT-041

Result shape podrá validarse contra el compiled result contract.

## DB-RESULT-042

Critical shape mismatches no serán ignorados.

## DB-RESULT-043

Column order será preservado.

## DB-RESULT-044

Result rows serán preferiblemente immutable.

## DB-RESULT-045

DriverResultAdapter será distinto de Platform.

## DB-RESULT-046

DriverResultAdapter será distinto de Hydrator.

## DB-RESULT-047

ResultAdaptationContext será explícito.

## DB-RESULT-048

Result conversion no dependerá de current HTTP request.

## DB-RESULT-049

Result conversion no dependerá de current EntityManager.

## DB-RESULT-050

Driver Result Type será distinto de Semantic Type.

## DB-RESULT-051

Semantic Type será distinto de PHP runtime type.

## DB-RESULT-052

Decimal conversion no perderá precisión silenciosamente.

## DB-RESULT-053

Big integers no se truncarán silenciosamente.

## DB-RESULT-054

Boolean conversion no usará PHP truthiness como regla universal.

## DB-RESULT-055

SQL NULL permanecerá distinguible de valores textuales.

## DB-RESULT-056

SQL NULL será distinto de JSON null.

## DB-RESULT-057

Binary data no será tratado automáticamente como text.

## DB-RESULT-058

Temporal conversion no dependerá de timezone global oculto.

## DB-RESULT-059

SQLite storage class será distinto de VoltStack semantic type.

## DB-RESULT-060

MySQL y MariaDB podrán tener adapters/mappings distintos.

## DB-RESULT-061

PostgreSQL custom types serán registry-driven.

## DB-RESULT-062

Result converter registry será tipado.

## DB-RESULT-063

Converter registry será frozen después de bootstrap.

## DB-RESULT-064

Converter conflicts no usarán last-wins.

## DB-RESULT-065

Raw result access será explícito.

## DB-RESULT-066

Result lifecycle será explícito.

## DB-RESULT-067

close() será seguro e idealmente idempotente.

## DB-RESULT-068

Driver resource state será distinguible de logical result accessibility.

## DB-RESULT-069

Result resource ownership será explícito.

## DB-RESULT-070

Result resources serán liberados.

## DB-RESULT-071

Statement no será liberado mientras un dependent result lo necesite.

## DB-RESULT-072

Connection no será reutilizada mientras un dependent result lo impida.

## DB-RESULT-073

Buffered result podrá detached de driver resources.

## DB-RESULT-074

Cursor result podrá mantener resources attached.

## DB-RESULT-075

Result consumption mode será explícito.

## DB-RESULT-076

Cursor result será normalmente single-pass.

## DB-RESULT-077

Buffered result podrá ser replayable.

## DB-RESULT-078

Result consumption no reejecutará queries implícitamente.

## DB-RESULT-079

Materialization será distinta de hydration.

## DB-RESULT-080

Result buffering estará sujeto a budget.

## DB-RESULT-081

Unknown result size no significará safe-to-buffer.

## DB-RESULT-082

Buffered result no se publicará parcialmente como completo.

## DB-RESULT-083

Streaming podrá tener partial consumption.

## DB-RESULT-084

Cancellation será distinta de exhaustion.

## DB-RESULT-085

Cancellation liberará resources.

## DB-RESULT-086

Result errors no expondrán valores sensibles por defecto.

## DB-RESULT-087

Sensitive result metadata será preservada.

## DB-RESULT-088

Telemetry no registrará row values por defecto.

## DB-RESULT-089

Metrics no usarán row values como labels.

## DB-RESULT-090

Unknown row count será distinto de zero.

## DB-RESULT-091

ResultId será distinto de query fingerprint.

## DB-RESULT-092

Persistent workers no conservarán current result state entre requests.

## DB-RESULT-093

Concurrent results tendrán sessions independientes.

## DB-RESULT-094

Shared registries deberán ser immutable.

## DB-RESULT-095

Current cursor nunca vivirá en shared singleton state.

## DB-RESULT-096

Result extensions serán tipadas.

## DB-RESULT-097

Result extension registry será frozen.

## DB-RESULT-098

Runtime requests no registrarán global result converters.

## DB-RESULT-099

Extensions no ejecutarán hidden queries.

## DB-RESULT-100

Extensions no hidratarán ORM entities.

## DB-RESULT-101

Extensions no podrán evadir result budgets.

## DB-RESULT-102

Raw driver result ownership permanecerá explícito.

## DB-RESULT-103

Result API favorecerá type-safe specialized interfaces.

## DB-RESULT-104

Cleanup automático no dependerá exclusivamente de destructors.

## DB-RESULT-105

Open result handles podrán registrarse en request resource scope.

## DB-RESULT-106

Debug podrá detectar leaked results.

## DB-RESULT-107

Result System no decidirá commit o rollback.

## DB-RESULT-108

Transaction-bound result declarará su dependencia.

## DB-RESULT-109

Detached result no retendrá driver resources.

## DB-RESULT-110

Result capabilities serán capability-driven.

## DB-RESULT-111

Result System no reordenará filas.

## DB-RESULT-112

Result System no eliminará filas duplicadas.

## DB-RESULT-113

Result System preservará bag semantics.

## DB-RESULT-114

Column labels no serán case-folded indiscriminadamente.

## DB-RESULT-115

Result shape fingerprints excluirán row values.

## DB-RESULT-116

Live cursor no será cacheable directamente.

## DB-RESULT-117

Cacheable result representation requerirá explicit detachment.

## DB-RESULT-118

Materialization producirá ResultRows, no entities.

## DB-RESULT-119

Metadata conflicts relevantes no serán ignorados.

## DB-RESULT-120

Unknown metadata permanecerá unknown cuando sea válido.

## DB-RESULT-121

Lazy metadata no ejecutará hidden queries.

## DB-RESULT-122

Lazy metadata no alterará cursor semantics.

## DB-RESULT-123

Streaming conversion ocurrirá row-by-row.

## DB-RESULT-124

Conversion failure mid-stream podrá representar partial consumption.

## DB-RESULT-125

Buffered conversion podrá ser atómica respecto a publicación.

## DB-RESULT-126

Multiple result set ordering será preservado.

## DB-RESULT-127

Actual result cardinality será distinta de planner estimate.

## DB-RESULT-128

Result telemetry podrá alimentar feedback futuro, no modificar el plan actual.

## DB-RESULT-129

Database data no será considerada automáticamente trusted.

## DB-RESULT-130

Result System no realizará HTML escaping.

## DB-RESULT-131

Result System no realizará JavaScript escaping.

## DB-RESULT-132

Encryption conversion será distinta de ORM hydration.

## DB-RESULT-133

Diagnostics podrán explicar converter selection.

## DB-RESULT-134

Conversion traces no expondrán secrets.

## DB-RESULT-135

Converters serán deterministas para input/context equivalentes.

## DB-RESULT-136

Converters no dependerán de wall clock oculto.

## DB-RESULT-137

Converters no dependerán de randomness oculto.

## DB-RESULT-138

Converters no utilizarán service locator arbitrario en hot path.

## DB-RESULT-139

Result conversion plans podrán pre-resolverse.

## DB-RESULT-140

Pre-resolved conversion plans no contendrán row values.

## DB-RESULT-141

Converter resolution deberá evitar scans por cada cell cuando sea posible.

## DB-RESULT-142

Public ResultRow conservará valores estables durante su lifetime.

## DB-RESULT-143

Reusable mutable internal row views no escaparán de su scope.

## DB-RESULT-144

Result failure conservará resource impact.

## DB-RESULT-145

Result cleanup ocurrirá ante failure.

## DB-RESULT-146

Result cleanup ocurrirá ante cancellation.

## DB-RESULT-147

Result cleanup ocurrirá ante request termination.

## DB-RESULT-148

Result System será independiente del ORM.

## DB-RESULT-149

Result System será independiente de una implementación concreta de driver.

## DB-RESULT-150

Result System será la frontera normalizada entre ejecución nativa y consumo de datos por capas superiores.

---

# 279. Invariante maestro de semántica

Para todo resultado válido:

```text
Semantics(DatabaseResult)
=
Semantics(NativeDriverResult)
```

dentro del `ResultContract` correspondiente.

---

# 280. Invariante maestro de shape

Si:

```text
ExpectedShape = E
ActualShape = A
```

entonces antes de entregar el resultado a una capa que dependa estrictamente de `E`:

```text
Compatible(A, E) = true
```

---

# 281. Invariante maestro de columnas

Para una fila con `n` columnas:

```text
∀ i ∈ [0, n):
    ResultRow.at(i)
```

deberá referenciar exactamente la columna `i` del resultado original.

---

# 282. Invariante maestro de duplicados

Si el DBMS devuelve:

```text
R = [r1, r2, r2, r3]
```

Result System deberá preservar:

```text
R' = [r1, r2, r2, r3]
```

No:

```text
R' = [r1, r2, r3]
```

---

# 283. Invariante maestro de orden

Si el DBMS entrega:

```text
r1 → r2 → r3
```

VoltStack deberá observar:

```text
r1 → r2 → r3
```

salvo una capa superior que solicite explícitamente otra transformación.

---

# 284. Invariante maestro de recursos

Para cada recurso `r`:

```text
Owned(r)
⇒
EventuallyReleased(r)
```

ante:

```text
success
failure
cancellation
request termination
```

---

# 285. Invariante maestro de dependencia

Si:

```text
Result R depends on Resource X
```

entonces:

```text
Lifetime(R)
⊆
Lifetime(X)
```

o `R` deberá detached antes de liberar `X`.

---

# 286. Invariante maestro de seguridad

```text
ResultData
```

podrá fluir hacia consumers autorizados, pero:

```text
SensitiveResultData
```

no deberá fluir automáticamente hacia:

```text
logs
metrics
diagnostics
exceptions
traces
```

---

# 287. Invariante maestro de persistent runtime

Al finalizar el operation scope:

```text
OpenOperationResultHandles = 0
```

salvo recursos transferidos explícitamente a otro scope válido.

---

# 288. Invariante maestro de conversión

```text
SemanticMeaning(
    ConvertedResultValue
)
=
SemanticMeaning(
    DriverValue,
    TypeMetadata
)
```

---

# 289. Invariante maestro de arquitectura

```text
Execution Engine
    ↓
Result System
    ↓
Hydration
    ↓
ORM
```

y nunca:

```text
Result System
    ↓
EntityManager internals
```

---

# 290. Ejemplo completo — SELECT

SQL ejecutado:

```sql
SELECT id, email, active
FROM users
WHERE tenant_id = ?
ORDER BY id
```

Driver:

```text
[
    ["42", "alice@example.com", "1"],
    ["43", "bob@example.com",   "0"]
]
```

Expected result contract:

```text
ROWS
├── id     : BIG_INTEGER
├── email  : STRING
└── active : BOOLEAN
```

Result conversion:

```text
"42"
 ↓
BigInteger/Integer converter
 ↓
42

"alice@example.com"
 ↓
String converter
 ↓
"alice@example.com"

"1"
 ↓
Boolean converter
 ↓
true
```

Resultado:

```text
RowResult
└── ResultRow
    ├── id     = 42
    ├── email  = "alice@example.com"
    └── active = true
```

Todavía no existe:

```text
User Entity
```

---

# 291. Ejemplo — COUNT

```sql
SELECT COUNT(*)
FROM users
```

Contract:

```text
SCALAR
EXACTLY_ONE
INTEGER
```

Driver:

```text
"125"
```

Resultado:

```text
ScalarResult(
    ResultValue(125)
)
```

---

# 292. Ejemplo — UPDATE

```sql
UPDATE users
SET active = ?
WHERE tenant_id = ?
```

Driver informa:

```text
affected = 12
```

Resultado:

```text
AffectedRowsResult
└── value = 12
```

---

# 293. Ejemplo — RETURNING

```sql
UPDATE users
SET active = FALSE
WHERE last_login < ?
RETURNING id, email
```

Resultado:

```text
ReturningResult
├── mutation metadata
└── rows
    ├── ResultRow(...)
    ├── ResultRow(...)
    └── ResultRow(...)
```

---

# 294. Ejemplo — empty SELECT

```sql
SELECT id
FROM users
WHERE id = -1
```

Resultado:

```text
RowResult
└── rows = []
```

No:

```text
NoResult
```

---

# 295. Anti-patterns

## Anti-pattern 1 — Return PDOStatement

```php
return $pdoStatement;
```

---

## Anti-pattern 2 — ORM hydration inside executor

```php
return array_map(
    fn ($row) => new User(...),
    $rows
);
```

---

## Anti-pattern 3 — Universal fetchAll

```php
$rows = $statement->fetchAll();
```

para toda consulta.

---

## Anti-pattern 4 — Associative-only rows

```php
return $statement->fetchAll(PDO::FETCH_ASSOC);
```

como representación interna universal.

---

## Anti-pattern 5 — Duplicate column overwrite

```text
id = first column
id = second column
→ first lost
```

---

## Anti-pattern 6 — Float decimal

```php
$value = (float) $numeric;
```

---

## Anti-pattern 7 — Hidden timezone

```php
new DateTime($value);
```

dependiendo de configuración global.

---

## Anti-pattern 8 — Cursor cache

```text
Cache::put(queryKey, $driverCursor)
```

---

## Anti-pattern 9 — Result value telemetry

```php
$metrics->label('email', $row['email']);
```

---

## Anti-pattern 10 — Implicit query replay

```text
cursor exhausted
↓
execute query again automatically
```

---

## Anti-pattern 11 — Result == entity

```text
DatabaseResult<User>
```

en el core de ejecución.

---

## Anti-pattern 12 — Destructor-only cleanup

```php
public function __destruct()
{
    $this->cursor->close();
}
```

como única garantía de liberación.

---

# 296. Modelo arquitectónico final

```text
                  EXECUTION ENGINE

BoundPreparedStatement
          │
          ▼
 Statement Execution
          │
          ▼
 Native Driver Result
          │
          ▼
┌──────────────────────────────┐
│     DriverResultAdapter      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        Result System         │
│                              │
│  Result Contract Validation  │
│  Result Classification       │
│  Result Shape Validation     │
│  Metadata Normalization      │
│  Type Conversion             │
│  Resource Ownership          │
│  Lifecycle                   │
│  Security                    │
│  Budget                      │
│  Telemetry                   │
└──────────────┬───────────────┘
               │
               ▼
        DatabaseResult
               │
      ┌────────┼─────────┐
      │        │         │
      ▼        ▼         ▼
    Rows     Scalar   Mutation
      │
      ▼
 ResultCursor
      │
      ▼
 Streaming Result
      │
      ▼
 Hydration System
      │
      ▼
      ORM
```

---

# 297. Fórmula arquitectónica

```text
Result System
=
Driver Result Adaptation
+
Result Contract Enforcement
+
Result Classification
+
Result Shape Preservation
+
Metadata Normalization
+
Database Type Conversion
+
Result Lifecycle
+
Resource Ownership
+
Consumption Semantics
+
Budget Enforcement
+
Security
+
Telemetry
+
Extension Control
+
Persistent Runtime Isolation
```

---

# 298. Fórmula de corrección

```text
CorrectResult
=
ContractCompatible
∧
ShapeCompatible
∧
TypeSafe
∧
OrderPreserved
∧
DuplicatesPreserved
∧
ResourceSafe
∧
SecurityPreserved
```

---

# 299. Principio final

> **A database result is a normalized representation of what the database produced; it is not yet an application object.**

Por tanto:

```text
Driver Result
→
DatabaseResult
→
Hydration
→
Entity
```

y no:

```text
Driver Result
→
Entity
```

La separación permite que el mismo Execution Engine sea utilizado por:

```text
Query Builder
ORM
Schema System
Migration System
CLI
Telemetry
administrative tooling
raw database API
```

sin introducir dependencias hacia el ORM.

---

# 300. Siguiente documento

```text
82_DATABASE_RESULT_CURSOR_SYSTEM.md
```

El siguiente documento deberá formalizar el mecanismo de consumo incremental:

```text
Result Cursor
├── Cursor Architecture
├── Cursor Contract
├── Cursor State Machine
├── Forward-Only Cursor
├── Scrollable Cursor
├── Cursor Capabilities
├── Row Fetching
├── Fetch Strategies
├── Cursor Position
├── Cursor Exhaustion
├── Cursor Ownership
├── Statement Dependency
├── Connection Dependency
├── Transaction Dependency
├── Cursor Resource Lifecycle
├── Cursor Closing
├── Cursor Cancellation
├── Cursor Failure
├── Cursor Budgets
├── Cursor Telemetry
├── Cursor Security
├── Persistent Runtime Isolation
└── Cursor Extensions
```

manteniendo el invariante:

```text
Result Cursor
≠
Result Set
≠
Buffered Result
≠
Iterator Convenience API
≠
ORM Collection
```