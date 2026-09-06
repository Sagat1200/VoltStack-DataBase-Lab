# 80_DATABASE_PARAMETER_BINDING_SYSTEM.md

# VoltStack Quantum Database
## Parameter Binding System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 80 — Parameter Binding System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Parameter Binding System` define la infraestructura responsable de transformar los parámetros lógicos y valores runtime de una operación en bindings concretos compatibles con el prepared statement y el driver activo.

La frontera principal es:

```text
CompiledDatabaseCommand
        │
        ├── CompiledBindingLayout
        │
        └── Parameter Metadata
                 +
RuntimeBindingSet
                 +
PreparedStatementLease
                 ↓
        Parameter Binding System
                 ↓
          DriverBindingSet
                 ↓
         Prepared Statement
```

Formalmente:

```text
Bind(
    PreparedStatementLease,
    CompiledBindingLayout,
    RuntimeBindingSet,
    ParameterBindingContext
)
    →
BoundPreparedStatement
```

El sistema deberá preservar de forma estricta:

- identidad del parámetro;
- correspondencia con placeholders;
- orden de binding;
- tipos;
- `NULL`;
- sensibilidad de datos;
- ownership de recursos;
- semántica de valores;
- compatibilidad con el driver;
- lifecycle de streams/LOBs;
- aislamiento entre ejecuciones;
- seguridad bajo runtimes persistentes.

---

# 2. Principio fundamental

```text
Parameter Binding
≠
SQL String Interpolation
```

El sistema nunca deberá convertir:

```text
SQL + Runtime Values
```

en:

```text
SQL concatenado
```

La operación correcta es:

```text
SQL estructural preparado
+
bindings separados
```

---

# 3. Invariante maestro de identidad

VoltStack deberá mantener separados:

```text
ParameterId
≠
ExecutionParameterSlotId
≠
BindingSlotId
≠
SQLPlaceholder
≠
DriverBindingPosition
```

Cada identidad pertenece a una etapa distinta.

---

# 4. Pipeline completo

```text
Query Parameter
      ↓
ParameterId
      ↓
Execution Planning
      ↓
ExecutionParameterSlotId
      ↓
SQL Compilation
      ↓
CompiledBindingLayout
      ↓
SQLPlaceholder
      ↓
Runtime
      ↓
RuntimeBindingSet
      ↓
Parameter Binding System
      ↓
DriverBindingPosition
      ↓
DriverPreparedStatement
```

---

# 5. Separación de responsabilidades

```text
Query Parameter
=
semantic identity
```

```text
Execution Parameter Slot
=
runtime value requirement
```

```text
SQL Placeholder
=
SQL representation
```

```text
Driver Binding Position
=
driver-specific binding location
```

```text
Runtime Binding
=
actual execution value
```

---

# 6. Parameter Binding System no genera SQL

Nunca:

```php
$sql = str_replace(':id', $value, $sql);
```

Ni:

```php
$sql = "WHERE id = {$id}";
```

El SQL ya fue compilado.

---

# 7. Relación con Prepared Statement System

Documento 79 produce:

```text
PreparedStatementLease
```

Documento 80 recibe ese lease y realiza:

```text
PreparedStatementLease
       +
RuntimeBindingSet
       +
CompiledBindingLayout
       ↓
ParameterBinder
       ↓
BoundPreparedStatement
```

---

# 8. Relación con Statement Execution

```text
Prepared Statement System
        ↓
PreparedStatementLease
        ↓
Parameter Binding System
        ↓
BoundPreparedStatement
        ↓
Statement Execution System
```

---

# 9. Contrato principal

```php
namespace VoltStack\Quantum\Database\Execution\Binding\Contract;

interface ParameterBinder
{
    public function bind(
        ParameterBindingRequest $request,
    ): BoundPreparedStatement;
}
```

---

# 10. ParameterBindingRequest

```php
final readonly class ParameterBindingRequest
{
    public function __construct(
        public PreparedStatementLease $statement,
        public CompiledBindingLayout $layout,
        public RuntimeBindingSet $bindings,
        public ParameterBindingContext $context,
    ) {}
}
```

---

# 11. ParameterBindingContext

```php
final readonly class ParameterBindingContext
{
    public function __construct(
        public DriverIdentity $driver,
        public PlatformIdentity $platform,
        public DriverBindingContract $driverContract,
        public DriverCapabilitySnapshot $capabilities,
        public BindingPolicy $policy,
        public BindingBudget $budget,
    ) {}
}
```

---

# 12. Contexto explícito

No deberán obtenerse mediante estado global:

```text
current connection
current tenant
current locale
current timezone
current driver
current request
current user
```

---

# 13. RuntimeBindingSet

Representa los valores requeridos para una ejecución concreta.

```php
final readonly class RuntimeBindingSet
{
    /**
     * @param array<string, RuntimeBinding> $bindings
     */
    public function __construct(
        private array $bindings,
    ) {}
}
```

---

# 14. RuntimeBinding

```php
final readonly class RuntimeBinding
{
    public function __construct(
        public ExecutionParameterSlotId $slot,
        public mixed $value,
        public RuntimeBindingMetadata $metadata,
    ) {}
}
```

---

# 15. RuntimeBindingSet es operation-scoped

Nunca deberá sobrevivir accidentalmente entre requests.

```text
Request A bindings
≠
Request B bindings
```

---

# 16. CompiledBindingLayout

Fue producido durante SQL compilation.

Describe cómo los parámetros deben mapearse hacia el SQL compilado.

```php
final readonly class CompiledBindingLayout
{
    /**
     * @param list<CompiledBindingSlot> $slots
     */
    public function __construct(
        public array $slots,
        public PlaceholderStyle $placeholderStyle,
        public BindingLayoutFingerprint $fingerprint,
    ) {}
}
```

---

# 17. CompiledBindingSlot

```php
final readonly class CompiledBindingSlot
{
    public function __construct(
        public BindingSlotId $id,
        public ExecutionParameterSlotId $executionSlot,
        public SqlPlaceholder $placeholder,
        public DriverBindingPosition $driverPosition,
        public QueryType $expectedType,
        public BindingDirection $direction,
        public BindingFlags $flags,
    ) {}
}
```

---

# 18. Layout ≠ values

```text
CompiledBindingLayout
=
structure
```

```text
RuntimeBindingSet
=
values
```

Nunca mezclar ambos.

---

# 19. Binding layout reusable

Un:

```text
CompiledBindingLayout
```

puede reutilizarse entre ejecuciones.

Los:

```text
RuntimeBindingSet
```

no.

---

# 20. Binding identity model

El flujo de identidad será:

```text
ParameterId
     ↓
ExecutionParameterSlotId
     ↓
BindingSlotId
     ↓
SqlPlaceholder
     ↓
DriverBindingPosition
```

---

# 21. ParameterId

Representa identidad semántica.

Ejemplo conceptual:

```text
:user_id
```

No significa necesariamente que el SQL final utilice:

```text
:user_id
```

---

# 22. ExecutionParameterSlotId

Representa una necesidad de valor runtime.

Ejemplo:

```text
exec_param_17
```

---

# 23. BindingSlotId

Representa una ocurrencia compilada que requiere binding.

---

# 24. SQLPlaceholder

Puede ser:

```text
?
?1
$1
:name
@name
```

dependiendo del target/driver.

---

# 25. DriverBindingPosition

Puede ser:

```text
1
2
3
```

o una key nominal según el driver.

---

# 26. Parámetros repetidos

Considérese:

```sql
WHERE user_id = :id
   OR owner_id = :id
```

Semánticamente existe:

```text
ParameterId P1
```

pero podrían existir:

```text
BindingSlot B1
BindingSlot B2
```

---

# 27. Placeholder duplication

Un driver puede permitir:

```text
:id
:id
```

mientras otro compilation contract puede producir:

```text
?
?
```

El binding system deberá seguir el layout compilado.

---

# 28. No asumir one parameter = one placeholder

```text
ParameterId
1:N
BindingSlot
```

debe estar soportado.

---

# 29. Binding pipeline

Propuesta:

```text
P0 Request Acceptance
P1 Statement Compatibility Validation
P2 Layout Validation
P3 Runtime Binding Validation
P4 Slot Resolution
P5 Value Classification
P6 Semantic Type Resolution
P7 Driver Type Resolution
P8 Value Conversion
P9 Resource Preparation
P10 Driver Binding
P11 Post-Binding Validation
P12 Bound Statement Finalization
```

---

# 30. P0 — Request Acceptance

Validar:

```text
PreparedStatementLease
CompiledBindingLayout
RuntimeBindingSet
BindingContext
```

---

# 31. P1 — Statement Compatibility

Debe comprobarse que:

```text
statement.bindingLayoutFingerprint
=
request.layoutFingerprint
```

o que ambos sean estructuralmente compatibles.

---

# 32. No bind to incompatible statement

Un statement preparado para:

```text
$1, $2
```

no deberá recibir arbitrariamente layout:

```text
?, ?, ?
```

---

# 33. P2 — Layout Validation

Validar:

```text
slot uniqueness
placeholder validity
driver position validity
direction validity
expected type validity
driver compatibility
```

---

# 34. P3 — Runtime Binding Validation

Detectar:

```text
missing values
unexpected values
duplicate runtime slots
invalid resource values
invalid directions
```

---

# 35. Missing binding

Si el layout requiere:

```text
P1
P2
P3
```

pero runtime sólo contiene:

```text
P1
P2
```

deberá fallar antes de ejecución.

---

# 36. Extra binding

Por defecto:

```text
runtime value without required slot
→ error
```

Esto ayuda a detectar bugs.

---

# 37. Optional parameters

Si existen parámetros opcionales deberán estar declarados explícitamente.

No se inferirá opcionalidad por ausencia.

---

# 38. P4 — Slot Resolution

```text
CompiledBindingSlot
        ↓
ExecutionParameterSlotId
        ↓
RuntimeBindingSet
        ↓
RuntimeBinding
```

---

# 39. Resolution complexity

Idealmente:

```text
O(n)
```

sobre número de binding slots.

---

# 40. No name guessing

Nunca:

```text
strip ":" from placeholder
→ search runtime array
```

El mapping ya debe estar compilado.

---

# 41. P5 — Value Classification

Valores pueden clasificarse como:

```text
NULL
BOOLEAN
INTEGER
FLOAT
DECIMAL
STRING
BINARY
DATE
TIME
DATETIME
UUID
ENUM
JSON
ARRAY
LOB
STREAM
CUSTOM
```

---

# 42. Runtime PHP type ≠ Database semantic type

Ejemplo:

```php
"42"
```

puede representar:

```text
STRING
INTEGER
DECIMAL
UUID fragment
JSON scalar
```

según metadata.

---

# 43. No inferencia ciega

Evitar:

```php
if (is_numeric($value)) {
    bindAsInteger();
}
```

---

# 44. Semantic type first

El binding debe partir de:

```text
QueryType / DatabaseTypeDescriptor
```

cuando esté disponible.

---

# 45. P6 — Semantic Type Resolution

Formalmente:

```text
ResolveSemanticType(
    ExpectedQueryType,
    RuntimeValue,
    BindingMetadata
)
→
ResolvedBindingType
```

---

# 46. Expected type dominates

Si el layout declara:

```text
UUID
```

un string PHP:

```text
"550e8400-e29b-41d4-a716-446655440000"
```

deberá tratarse como UUID, no como string genérico.

---

# 47. Runtime override

Un override explícito puede permitirse:

```php
Binding::typed($value, DatabaseType::JSON)
```

pero deberá validarse contra el expected type.

---

# 48. Type mismatch

Ejemplo:

```text
expected = INTEGER
runtime explicit = JSON
```

deberá fallar salvo conversión registrada y segura.

---

# 49. P7 — Driver Type Resolution

El semantic type deberá traducirse al tipo esperado por el driver.

```text
QueryType
   ↓
DriverBindingTypeResolver
   ↓
DriverBindingType
```

---

# 50. DriverBindingType

Ejemplos conceptuales:

```text
NULL
BOOL
INT
FLOAT
STRING
BINARY
LOB
STREAM
DRIVER_NATIVE
```

---

# 51. Driver types are not semantic types

```text
QueryType::UUID
```

puede terminar como:

```text
DriverBindingType::STRING
```

sin perder que semánticamente era UUID.

---

# 52. Type pipeline

```text
Semantic Type
    ↓
Storage Representation
    ↓
Driver Binding Type
```

---

# 53. Example

```text
UUID
 ↓
canonical string
 ↓
driver STRING
```

Otro:

```text
JSON
 ↓
UTF-8 JSON document
 ↓
driver STRING
```

---

# 54. P8 — Value Conversion

```text
Runtime Value
      +
Resolved Semantic Type
      +
Platform Representation
      ↓
BindingValueConverter
      ↓
Driver-Compatible Value
```

---

# 55. Conversion ≠ arbitrary casting

No:

```php
(string) $value;
```

como estrategia universal.

---

# 56. Conversion must preserve semantics

```text
SemanticMeaning(input)
=
SemanticMeaning(converted binding)
```

---

# 57. NULL binding

`NULL` deberá conservarse como `NULL`.

Nunca:

```text
NULL
→
"NULL"
```

ni:

```text
NULL
→
0
```

---

# 58. Nullability validation

Si un binding slot está declarado:

```text
NOT_NULL
```

y recibe:

```text
NULL
```

deberá fallar antes del driver cuando sea verificable.

---

# 59. Typed NULL

En algunos DBMS:

```text
NULL
```

puede necesitar contexto de tipo.

Por ello:

```text
value = NULL
type = UUID
```

es distinto de:

```text
value = NULL
type = UNKNOWN
```

---

# 60. Boolean binding

Boolean semantic values:

```text
true
false
```

podrán representarse según plataforma/driver como:

```text
TRUE/FALSE native
1/0
driver boolean
```

sin alterar semántica.

---

# 61. No boolean SQL interpolation

Nunca:

```php
$sql .= $value ? 'TRUE' : 'FALSE';
```

desde binding.

---

# 62. Integer binding

Deberá considerar:

```text
signed range
unsigned requirements
PHP integer range
platform range
driver range
```

---

# 63. Overflow

Un integer fuera del rango soportado deberá:

```text
fail
```

o utilizar una representación explícita compatible.

Nunca truncarse silenciosamente.

---

# 64. Floating point

El sistema deberá distinguir cuando sea relevante:

```text
FLOAT
DOUBLE
DECIMAL
```

---

# 65. Decimal

Valores monetarios o de precisión arbitraria no deberán pasar automáticamente por:

```php
(float) $value
```

---

# 66. Decimal representation

Puede utilizarse:

```text
canonical decimal string
```

para preservar precisión.

---

# 67. String binding

Deberá considerar:

```text
encoding
normalization policy
length limits
driver constraints
```

---

# 68. Encoding

UTF-8 será preferible dentro del framework, pero la conversión final deberá respetar el connection/driver contract.

---

# 69. Silent encoding corruption prohibited

Nunca reemplazar bytes inválidos silenciosamente salvo política explícita.

---

# 70. Binary binding

Binary:

```text
≠
text string
```

aunque PHP utilice `string` para ambos.

---

# 71. Binary descriptor

```php
final readonly class BinaryValue
{
    public function __construct(
        public string $bytes,
    ) {}
}
```

puede evitar ambigüedad.

---

# 72. UUID binding

Debe existir representación canónica.

Ejemplos:

```text
text UUID
binary UUID
native UUID
```

según plataforma/type mapping.

---

# 73. UUID validation

Debe validarse antes del binding si el type contract lo exige.

---

# 74. Enum binding

Distinguir:

```text
PHP enum
database enum
string-backed enum
integer-backed enum
custom mapped enum
```

---

# 75. Enum conversion

```text
PHP Enum Case
      ↓
EnumMapping
      ↓
Database Representation
```

---

# 76. Unknown enum case

No deberá convertirse silenciosamente.

---

# 77. JSON binding

Pipeline:

```text
PHP value / JsonValue
       ↓
JSON Type Converter
       ↓
validated JSON representation
       ↓
driver binding
```

---

# 78. JSON null distinction

Debe distinguirse:

```text
SQL NULL
```

de:

```json
null
```

---

# 79. Example

```text
RuntimeBinding(SQL_NULL)
```

no equivale a:

```text
RuntimeBinding(JsonNull)
```

---

# 80. JSON encoding failures

Errores como:

```text
invalid UTF-8
unsupported value
recursion
depth exceeded
```

deberán producir error de binding.

---

# 81. Date/time binding

Debe distinguir:

```text
DATE
TIME
DATETIME
TIMESTAMP
TIMESTAMP_WITH_TIME_ZONE
INSTANT
```

cuando el type system lo soporte.

---

# 82. No hidden current timezone

Nunca:

```text
DateTime
→ convert using global PHP timezone
```

sin política explícita.

---

# 83. Timezone policy

Puede formar parte de:

```text
DateTimeBindingPolicy
```

---

# 84. Example

```text
Instant
 ↓
UTC normalization
 ↓
platform representation
```

---

# 85. SQLite date/time

Puede usar políticas explícitas como:

```text
ISO8601_TEXT
UNIX_SECONDS_INTEGER
UNIX_MILLISECONDS_INTEGER
JULIAN_DAY_REAL
```

según metadata previamente resuelta.

---

# 86. PostgreSQL date/time

Podrá utilizar representación compatible con tipos PostgreSQL sin introducir lógica PostgreSQL en el binder genérico.

---

# 87. Array binding

No todos los drivers/plataformas soportan arrays nativos.

---

# 88. Array strategies

Pueden existir:

```text
NATIVE_ARRAY
SERIALIZED_ARRAY
EXPANDED_PARAMETERS
JSON_ARRAY
UNSUPPORTED
```

pero la estrategia debe haber sido determinada antes cuando cambie estructura SQL.

---

# 89. Critical rule

Si convertir:

```text
IN (:ids)
```

en:

```text
IN (?, ?, ?)
```

cambia la estructura SQL, eso pertenece a compilation/planning.

No al runtime binder.

---

# 90. Binder cannot change placeholder count

```text
Binding System
```

debe respetar:

```text
CompiledBindingLayout
```

---

# 91. LOB binding

Large Objects requieren lifecycle especial.

Ejemplos:

```text
BLOB
CLOB
large binary stream
large text stream
```

---

# 92. LOB ≠ ordinary scalar

Puede requerir:

```text
resource ownership
streaming
temporary handles
driver APIs
cleanup
transaction affinity
```

---

# 93. LobBindingResource

```php
interface LobBindingResource
{
    public function open(): mixed;

    public function close(): void;
}
```

---

# 94. Stream binding

Streams deberán tener ownership explícito.

---

# 95. Stream ownership modes

```text
BORROWED
TRANSFERRED
FRAMEWORK_OWNED
DRIVER_OWNED
```

---

# 96. Borrowed stream

VoltStack no deberá cerrarlo si el caller conserva ownership.

---

# 97. Transferred stream

VoltStack deberá cerrarlo según lifecycle.

---

# 98. Driver-owned

Después del binding/execution el driver puede ser responsable.

Esto deberá estar definido por contract.

---

# 99. Stream position

Debe declararse si el driver espera:

```text
current position
rewind to start
known length
unknown length
```

---

# 100. No implicit rewind

No deberá hacerse:

```php
rewind($stream);
```

sin contract.

---

# 101. Stream replayability

Puede clasificarse:

```text
REPLAYABLE
ONE_SHOT
UNKNOWN
```

---

# 102. Retry implication

Un binding con:

```text
ONE_SHOT stream
```

puede volver una ejecución no retryable.

El Binding System deberá propagar esa metadata.

---

# 103. Binding resource profile

```php
final readonly class BindingResourceProfile
{
    public function __construct(
        public bool $containsStreams,
        public bool $containsLobs,
        public bool $replayable,
        public bool $requiresCleanup,
    ) {}
}
```

---

# 104. Input/output parameters

Algunos drivers/procedimientos pueden soportar:

```text
IN
OUT
INOUT
```

---

# 105. BindingDirection

```php
enum BindingDirection
{
    case INPUT;
    case OUTPUT;
    case INPUT_OUTPUT;
}
```

---

# 106. Default

Para queries normales:

```text
INPUT
```

---

# 107. Output parameters

No deberán simularse si el driver no los soporta.

---

# 108. Output binding lifecycle

```text
Bind output slot
    ↓
Execute
    ↓
Driver populates value
    ↓
OutputBindingResult
```

---

# 109. BoundPreparedStatement

Resultado del binding:

```php
final readonly class BoundPreparedStatement
{
    public function __construct(
        public PreparedStatementLease $statement,
        public BoundParameterSet $parameters,
        public BindingResourceProfile $resources,
        public BindingFingerprint $fingerprint,
    ) {}
}
```

---

# 110. BoundPreparedStatement is operation-scoped

No deberá almacenarse en prepared statement cache.

---

# 111. BoundParameterSet

Contiene metadata runtime de los bindings realizados.

No necesariamente duplica valores sensibles.

---

# 112. Sensitive value policy

Cada binding podrá clasificarse:

```text
PUBLIC
INTERNAL
SENSITIVE
SECRET
```

---

# 113. Examples

```text
pagination limit → INTERNAL
user email       → SENSITIVE
password         → SECRET
API token        → SECRET
```

---

# 114. Classification does not change SQL semantics

Sólo afecta:

```text
logging
telemetry
debugging
error reporting
memory handling
```

---

# 115. SensitiveBindingValue

Puede existir un wrapper:

```php
final readonly class SensitiveBindingValue
{
    public function __construct(
        public mixed $value,
        public SensitivityLevel $level,
    ) {}
}
```

---

# 116. Redaction

Diagnostics:

```text
password = [REDACTED]
```

No:

```text
password = hunter2
```

---

# 117. Fingerprints must exclude values

Nunca:

```text
BindingFingerprint
=
hash(runtime values)
```

si eso puede exponer secretos o crear cardinalidad innecesaria.

---

# 118. Binding fingerprint

Debe describir estructura:

```text
BindingFingerprint
=
LayoutFingerprint
+
ResolvedTypeLayout
+
DriverBindingContract
+
BindingPolicyVersion
```

No valores.

---

# 119. P9 — Resource Preparation

Antes del driver binding puede requerirse preparar:

```text
stream handles
LOB handles
temporary buffers
encoded payloads
```

---

# 120. Resource ownership registry

```text
BindingResourceRegistry
```

deberá registrar todo recurso temporal creado.

---

# 121. Atomic binding

Si falla el binding número 8 de 10:

```text
bindings 1..7
```

no deberán quedar como estado reusable accidental.

---

# 122. Binding transaction concept

No es una DB transaction.

Puede existir:

```text
BindingSession
```

operation-scoped para cleanup atómico.

---

# 123. BindingSession

```php
final class BindingSession
{
    // temporary buffers
    // resources
    // cleanup actions
    // diagnostics
    // budget accounting
}
```

---

# 124. Binding failure

Flujo:

```text
bind slot 1
bind slot 2
bind slot 3
FAIL
   ↓
cleanup resources
   ↓
statement reset/invalidate
   ↓
throw normalized exception
```

---

# 125. Statement impact

Un binding error puede ser:

```text
RECOVERABLE_AFTER_RESET
STATEMENT_INVALID
CONNECTION_INVALID
```

según driver.

---

# 126. P10 — Driver Binding

Contrato:

```php
interface DriverParameterBinder
{
    public function bind(
        DriverPreparedStatement $statement,
        DriverBinding $binding,
    ): void;
}
```

---

# 127. DriverBinding

```php
final readonly class DriverBinding
{
    public function __construct(
        public DriverBindingPosition $position,
        public mixed $value,
        public DriverBindingType $type,
        public BindingDirection $direction,
        public DriverBindingOptions $options,
    ) {}
}
```

---

# 128. DriverBindingPosition validation

Antes de binding:

```text
position exists
position is unique where required
position is compatible with placeholder style
```

---

# 129. Binding order

Debe ser determinista.

---

# 130. Canonical order

Preferiblemente:

```text
CompiledBindingLayout order
```

---

# 131. Driver-specific ordering

Si el driver requiere otro orden, debe declararlo mediante:

```text
DriverBindingContract
```

---

# 132. Bind-by-value vs bind-by-reference

Debe distinguirse.

---

# 133. Default recommendation

Preferir:

```text
bind-by-value
```

cuando sea posible.

---

# 134. Why

Bind-by-reference puede introducir:

```text
mutable external state
unexpected value changes
persistent references
lifecycle ambiguity
```

---

# 135. Bind-by-reference

Sólo deberá utilizarse cuando:

```text
driver requires it
output parameter requires it
explicit API requests it
```

---

# 136. Reference ownership

Deberá quedar explícito y limpiarse después.

---

# 137. P11 — Post-Binding Validation

Verificar:

```text
all required slots bound
no duplicate incompatible bindings
resources registered
driver accepted bindings
statement state valid
budget not exceeded
```

---

# 138. P12 — Finalization

```text
PreparedStatement
    state:
    BINDING
       ↓
    READY
```

---

# 139. Statement state transition

```text
LEASED
   ↓
BINDING
   ↓
READY
```

En fallo:

```text
BINDING
   ↓
RESETTING / INVALID
```

---

# 140. Bound state does not imply executed

```text
READY
≠
EXECUTING
```

---

# 141. Binding validation levels

Podrán existir:

```text
MINIMAL
STANDARD
STRICT
DEBUG
```

---

# 142. Default

```text
STANDARD
```

---

# 143. Strict mode

Puede validar adicionalmente:

```text
range
encoding
length
type compatibility
resource replayability
driver constraints
```

---

# 144. Production performance

Las validaciones estructurales críticas nunca deberán desactivarse.

Sólo diagnostics adicionales podrán variar.

---

# 145. Binding conversion registry

```text
BindingValueConverterRegistry
```

resuelve convertidores por:

```text
SemanticType
+
Platform
+
DriverContract
```

---

# 146. Registry frozen

Después de bootstrap:

```text
BindingValueConverterRegistry
=
immutable
```

---

# 147. No last-wins

Dos convertidores exclusivos para:

```text
UUID + PostgreSQL + Driver X
```

deberán producir conflicto.

---

# 148. Converter resolution

Preferir exact identity sobre:

```php
supports(mixed $value)
```

demasiado amplio.

---

# 149. Resolution formula

```text
Converter Resolution
=
Semantic Type Identity
+
Platform Scope
+
Driver Scope
+
Capability Satisfaction
+
Specificity
+
Conflict Freedom
```

---

# 150. Custom types

El sistema deberá soportar tipos definidos por extensiones.

Ejemplo:

```text
Money
GeoPoint
ULID
EncryptedString
```

---

# 151. Custom type contract

```php
interface BindingValueConverter
{
    public function convert(
        mixed $value,
        BindingConversionContext $context,
    ): ConvertedBindingValue;
}
```

---

# 152. ConvertedBindingValue

```php
final readonly class ConvertedBindingValue
{
    public function __construct(
        public mixed $value,
        public DriverBindingType $driverType,
        public BindingResourceProfile $resources,
    ) {}
}
```

---

# 153. Custom converter limitations

No podrá:

```text
change SQL
add placeholders
remove placeholders
execute queries
open unrelated connections
commit transactions
inspect ORM entities implicitly
```

---

# 154. Value object binding

Value objects deberán convertirse mediante mapping explícito.

No:

```php
(string) $object
```

por defecto.

---

# 155. No __toString fallback

Unknown object:

```text
→ BindingUnsupportedValueException
```

---

# 156. ORM entities

El binder no deberá aceptar automáticamente:

```php
$userEntity
```

y convertirlo a:

```text
$userEntity->id
```

---

# 157. Why

Eso introduciría dependencia:

```text
Execution → ORM
```

que viola arquitectura.

---

# 158. Entity ID extraction

Debe ocurrir antes del Binding System.

---

# 159. Collections

Misma regla.

El binder no expandirá collections en placeholders.

---

# 160. Parameter normalization

Puede existir:

```text
RuntimeBindingNormalizer
```

pero sólo para estructura runtime, no SQL.

---

# 161. Example

Permitido:

```text
DateTimeImmutable
→ canonical temporal representation
```

No permitido:

```text
[1,2,3]
→ ?,?,?
```

porque cambia SQL.

---

# 162. Binding budgets

`BindingBudget` deberá limitar:

```text
number of parameters
converted bytes
LOB resources
stream resources
JSON serialization size
conversion depth
temporary buffer memory
custom converter invocations
```

---

# 163. Parameter count

Debe respetar límites de:

```text
compiled command
driver
platform
```

---

# 164. Unknown limit

```text
unknown
≠
unlimited
```

---

# 165. Large bindings

Valores enormes pueden requerir:

```text
streaming
LOB strategy
resource rejection
```

en vez de buffering completo.

---

# 166. Memory budget

Nunca cargar automáticamente un archivo de varios GB a memoria para convertirlo a string.

---

# 167. JSON budget

JSON serialization deberá tener:

```text
depth limit
byte limit
time/work budget
```

---

# 168. Recursive structures

Deben fallar de forma controlada.

---

# 169. Persistent runtime

En FrankenPHP:

```text
Request A
  ↓
bind secret S1
  ↓
execute
  ↓
cleanup

Request B
  ↓
same PreparedStatement
```

No deberá existir referencia residual a `S1`.

---

# 170. Request cleanup

Después de ejecución:

```text
RuntimeBindingSet
BoundParameterSet
BindingSession
temporary buffers
sensitive references
stream ownership records
```

deberán liberarse.

---

# 171. Prepared statement cache boundary

El cache podrá conservar:

```text
statement structure
driver handle
parameter metadata
```

pero no:

```text
previous runtime values
```

---

# 172. Long-lived singleton rule

Servicios compartidos podrán contener:

```text
immutable registries
stateless converters
immutable policies
```

No:

```text
current bindings
current statement
current stream
current tenant
```

---

# 173. Concurrency

Cada ejecución tendrá su propio:

```text
BindingSession
```

---

# 174. No shared mutable binding state

Especialmente bajo:

```text
OpenSwoole
RoadRunner
FrankenPHP workers
```

---

# 175. Coroutine safety

```text
Coroutine A bindings
≠
Coroutine B bindings
```

aunque compartan:

```text
CompiledDatabaseCommand
```

---

# 176. Statement exclusivity

Si Prepared Statement System declara statement:

```text
EXCLUSIVE
```

sólo una BindingSession podrá mutarlo simultáneamente.

---

# 177. Binding immutability

`RuntimeBindingSet` preferiblemente será immutable.

---

# 178. Driver mutation

La única mutación necesaria deberá ocurrir en:

```text
DriverPreparedStatement
```

bajo lease exclusivo.

---

# 179. Security model

La seguridad depende de mantener:

```text
SQL structure
```

separada de:

```text
runtime values
```

---

# 180. SQL injection invariant

```text
Untrusted Runtime Value
→ Binding
```

Nunca:

```text
Untrusted Runtime Value
→ SQL syntax
```

---

# 181. Identifier parameters prohibited

No deberá permitirse:

```sql
SELECT * FROM ?
```

esperando bindear un table name.

---

# 182. Why

Placeholders representan valores, no gramática SQL.

---

# 183. ORDER BY direction

No:

```sql
ORDER BY created_at ?
```

con `"DESC"` como binding.

Debe compilarse estructuralmente.

---

# 184. Operator binding

No:

```sql
WHERE age ? 18
```

con `">"` como runtime binding.

---

# 185. Function binding

No:

```sql
SELECT ?(column)
```

---

# 186. Collation binding

No:

```sql
COLLATE ?
```

salvo capacidad específica real y estructuralmente modelada.

---

# 187. Sensitive value lifetime

Debe minimizarse:

```text
creation
↓
binding
↓
execution
↓
cleanup
```

---

# 188. Diagnostics security

Errores deberán mostrar:

```text
parameter slot
expected type
actual runtime type
```

pero no necesariamente el valor.

---

# 189. Example safe diagnostic

```text
Binding failed for slot exec_param_12:
expected UUID,
received string with invalid UUID representation.
```

---

# 190. Unsafe diagnostic

```text
Invalid UUID: SECRET-CUSTOMER-TOKEN-...
```

---

# 191. Telemetry

Eventos sugeridos:

```text
ParameterBindingStarted
ParameterBindingValidated
ParameterConversionStarted
ParameterConverted
DriverBindingStarted
ParameterBound
ParameterBindingCompleted
ParameterBindingFailed
BindingResourceCreated
BindingResourceReleased
```

---

# 192. Telemetry metadata

Permitido:

```text
parameter count
type distribution
binding duration
converted byte count
LOB count
stream count
driver binding strategy
```

---

# 193. Telemetry values

Por defecto:

```text
runtime values
=
not recorded
```

---

# 194. Cardinality explosion

Nunca crear métricas con labels:

```text
user_id
email
UUID value
token
query parameter value
```

---

# 195. Metrics

Ejemplos:

```text
db.binding.count
db.binding.duration
db.binding.failures
db.binding.bytes
db.binding.streams
db.binding.lobs
db.binding.conversion.failures
```

---

# 196. Debug mode

Podrá mostrar:

```text
slot
placeholder
semantic type
driver type
direction
sensitivity
resource class
```

---

# 197. Debug values

Incluso en debug deberán respetar política de redacción.

---

# 198. Driver capability examples

El contract puede declarar:

```text
supports_named_binding
supports_positional_binding
supports_output_parameters
supports_stream_binding
supports_lob_binding
supports_native_boolean
supports_bind_by_reference
supports_rebinding
```

---

# 199. Capability-driven behavior

No usar:

```php
if ($driver === 'pdo_pgsql') {
    ...
}
```

en el binder genérico.

---

# 200. DriverBindingContract

```php
interface DriverBindingContract
{
    public function placeholderStyle(): PlaceholderStyle;

    public function resolveType(
        ResolvedBindingType $type,
    ): DriverBindingType;

    public function capabilities(): DriverBindingCapabilities;
}
```

---

# 201. PDO adapter

Puede existir:

```text
PdoParameterBinder
```

que traduzca a:

```text
PDO::PARAM_NULL
PDO::PARAM_BOOL
PDO::PARAM_INT
PDO::PARAM_STR
PDO::PARAM_LOB
```

---

# 202. Core ≠ PDO

El core seguirá trabajando con:

```text
DriverBindingType
```

---

# 203. MySQL binding

Las particularidades deberán vivir en:

```text
MySqlDriverBindingContract
```

o adapter correspondiente.

---

# 204. MariaDB binding

MariaDB tendrá su propio capability profile cuando difiera.

---

# 205. PostgreSQL binding

Puede requerir consideración especial para:

```text
UUID
arrays
JSON/JSONB
numeric
binary
date/time
```

pero mediante tipos/capabilities.

---

# 206. SQLite binding

Debe considerar:

```text
NULL
INTEGER
REAL
TEXT
BLOB
```

como storage-level representations sin confundirlas con QueryType.

---

# 207. SQLite invariant

```text
SQLite Storage Class
≠
VoltStack Semantic Type
```

---

# 208. Binding retryability metadata

El resultado del binding deberá poder contribuir a:

```text
ExecutionRetryEligibility
```

---

# 209. Replayability

```text
All bindings replayable
```

es condición útil para ciertos retries.

---

# 210. Formula

```text
BindingsReplayable
=
∀ b ∈ Bindings:
    Replayable(b)
```

---

# 211. Scalar replayability

Normalmente:

```text
scalar immutable value
→ replayable
```

---

# 212. Stream replayability

Depende del stream.

---

# 213. One-shot generator

No deberá aceptarse como generic binding iterable si su consumo es irreversible.

---

# 214. No lazy arbitrary evaluation

Evitar bindings como:

```php
fn () => getCurrentValue()
```

evaluados varias veces sin semántica explícita.

---

# 215. Deferred bindings

Si se soportan deberán ser un tipo explícito:

```text
DeferredBinding
```

---

# 216. Deferred binding evaluation

Debe ocurrir exactamente en una fase definida.

---

# 217. Determinism

Una vez materializado:

```text
RuntimeBindingSet
```

la misma ejecución no deberá reevaluar valores arbitrariamente.

---

# 218. Current time

Valores como:

```text
now()
```

deben resolverse antes o mediante una runtime value source explícita.

No dentro del converter.

---

# 219. Random values

Misma regla.

---

# 220. Binding extensions

Tipos de extensión:

```text
Value Converter
Driver Type Resolver
LOB Adapter
Stream Adapter
Custom Type Binder
Binding Validator
Binding Diagnostic Enricher
```

---

# 221. Extension descriptor

```php
final readonly class ParameterBindingExtensionDescriptor
{
    public function __construct(
        public ExtensionId $id,
        public Version $version,
        public array $semanticTypes,
        public PlatformScope $platforms,
        public DriverScope $drivers,
        public CapabilityRequirements $requirements,
    ) {}
}
```

---

# 222. Extension lifecycle

```text
bootstrap
↓
descriptor validation
↓
dependency resolution
↓
conflict detection
↓
capability validation
↓
registry composition
↓
freeze
```

---

# 223. No runtime registration

No:

```text
Request A registers custom binder
Request B sees custom binder
```

---

# 224. Core override policy

Por defecto:

```text
DENY
```

---

# 225. Third-party extension

Deberá añadir:

```text
new semantic type
new driver adapter
new explicit conversion
```

en vez de reemplazar silenciosamente bindings core.

---

# 226. Binding error hierarchy

```text
ParameterBindingException
├── BindingValidationException
├── MissingBindingException
├── UnexpectedBindingException
├── DuplicateBindingException
├── BindingLayoutMismatchException
├── BindingTypeException
├── BindingTypeMismatchException
├── BindingConversionException
├── BindingOverflowException
├── BindingEncodingException
├── BindingJsonException
├── BindingDateTimeException
├── BindingEnumException
├── BindingUuidException
├── BindingResourceException
├── BindingStreamException
├── BindingLobException
├── DriverBindingException
├── BindingUnsupportedTypeException
├── BindingUnsupportedDirectionException
├── BindingBudgetException
├── BindingSecurityException
├── BindingExtensionException
└── BindingInvariantException
```

---

# 227. Error normalization

Debe conservar:

```text
normalized category
driver error
statement impact
connection impact
retry metadata
safe diagnostics
```

---

# 228. Driver exception preservation

La excepción original puede conservarse como:

```text
previous/cause
```

sin exponerla indiscriminadamente al usuario.

---

# 229. Cleanup on exception

Toda excepción deberá atravesar:

```text
BindingSession cleanup
```

antes de salir.

---

# 230. Cleanup failure

Si cleanup también falla:

```text
primary failure
+
cleanup failure
```

deberán conservarse sin ocultar el error original.

---

# 231. Statement invalidation on binding failure

No todo error requiere invalidar statement.

Ejemplo:

```text
invalid UUID supplied by application
```

puede ocurrir antes de tocar driver.

---

# 232. Failure stages

```text
PRE_DRIVER
DRIVER_PARTIAL_BIND
DRIVER_COMPLETE_BIND
RESOURCE_CLEANUP
```

---

# 233. Pre-driver failure

Normalmente statement puede seguir:

```text
LEASED
```

y ser reseteado/liberado.

---

# 234. Partial driver bind

Puede requerir:

```text
statement reset
```

antes del reuse.

---

# 235. Driver-specific impact

Será determinado por:

```text
DriverBindingFailureClassifier
```

---

# 236. Testing strategy

Debe incluir:

```text
identity mapping
missing parameters
extra parameters
duplicate placeholders
repeated semantic parameters
positional placeholders
named placeholders
numbered placeholders
NULL
typed NULL
booleans
integer boundaries
decimal precision
floating point
strings
UTF-8
binary data
UUID
enum
JSON
SQL NULL vs JSON null
date/time
timezone policies
arrays
LOBs
streams
stream ownership
output parameters
custom types
driver errors
cleanup failures
budget exhaustion
persistent workers
concurrent executions
security redaction
```

---

# 237. Property-based testing

Útil para verificar:

```text
ParameterId mapping
BindingSlot mapping
placeholder ordering
type conversion
integer boundaries
decimal round-trips
```

---

# 238. Cross-platform conformance

Suite mínima:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 239. Driver conformance

Cada driver deberá demostrar:

```text
placeholder contract
binding type behavior
NULL behavior
binary behavior
LOB behavior
stream behavior
reset behavior
```

---

# 240. Persistent runtime tests

Ejemplo:

```text
Request A
bind SECRET_A
execute
release

Request B
reuse statement
bind SECRET_B
execute
```

Verificar:

```text
SECRET_A not retained
```

---

# 241. Concurrency test

```text
Coroutine A
   ↓
BindingSession A

Coroutine B
   ↓
BindingSession B
```

No deberá existir contaminación cruzada.

---

# 242. Failure injection

Simular:

```text
failure after slot 1
failure after slot N
driver rejection
stream read failure
LOB allocation failure
cleanup failure
connection loss
```

---

# 243. Performance model

Para `n` parameters:

```text
BindingCost
≈
O(n)
+
ConversionCost
+
DriverBindingCost
```

---

# 244. Avoid quadratic mapping

No:

```text
for each placeholder:
    scan all runtime bindings
```

---

# 245. Indexed runtime bindings

Preferir lookup:

```text
ExecutionParameterSlotId
→ RuntimeBinding
```

---

# 246. Conversion caching

No cachear runtime converted values globalmente.

---

# 247. Immutable conversion metadata

Sí puede cachearse:

```text
converter resolution
driver type mapping
type descriptor
```

si es immutable y compatible.

---

# 248. Hot path

El hot path ideal:

```text
layout already compiled
↓
converter already resolved
↓
runtime value
↓
convert
↓
driver bind
```

---

# 249. Precomputed binding plan

Podrá existir:

```text
CompiledParameterBindingPlan
```

para reducir resolución runtime.

---

# 250. Binding plan

```php
final readonly class CompiledParameterBindingPlan
{
    /**
     * @param list<CompiledParameterBindingInstruction> $instructions
     */
    public function __construct(
        public array $instructions,
        public BindingPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 251. Binding instruction

Puede contener:

```text
ExecutionParameterSlotId
DriverBindingPosition
ConverterId
DriverBindingType
Direction
Flags
SensitivityPolicy
```

---

# 252. Binding plan ≠ runtime values

Sigue siendo reusable.

---

# 253. Compiler boundary

El `Prepared Statement Compilation System` del documento 74 puede producir parte de esta información.

El runtime binder sólo ejecuta el plan.

---

# 254. Architectural optimization

Idealmente:

```text
Compilation Time
────────────────
Parameter identities
placeholder layout
converter strategy where static
driver binding positions
expected types

Runtime
───────
resolve value
validate dynamic constraints
convert
bind
```

---

# 255. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Binding/
                ├── Contract/
                │   ├── ParameterBinder.php
                │   ├── DriverParameterBinder.php
                │   ├── DriverBindingContract.php
                │   ├── BindingValueConverter.php
                │   └── DriverBindingFailureClassifier.php
                │
                ├── Binder/
                │   └── DefaultParameterBinder.php
                │
                ├── Request/
                │   └── ParameterBindingRequest.php
                │
                ├── Context/
                │   ├── ParameterBindingContext.php
                │   ├── BindingConversionContext.php
                │   └── BindingPolicy.php
                │
                ├── Runtime/
                │   ├── RuntimeBindingSet.php
                │   ├── RuntimeBinding.php
                │   ├── RuntimeBindingMetadata.php
                │   └── DeferredBinding.php
                │
                ├── Layout/
                │   ├── CompiledBindingLayout.php
                │   ├── CompiledBindingSlot.php
                │   ├── BindingSlotId.php
                │   └── BindingLayoutFingerprint.php
                │
                ├── Plan/
                │   ├── CompiledParameterBindingPlan.php
                │   ├── CompiledParameterBindingInstruction.php
                │   └── BindingPlanFingerprint.php
                │
                ├── Type/
                │   ├── ResolvedBindingType.php
                │   ├── DriverBindingType.php
                │   ├── DriverBindingTypeResolver.php
                │   └── BindingDirection.php
                │
                ├── Conversion/
                │   ├── BindingValueConverterRegistry.php
                │   ├── ConvertedBindingValue.php
                │   ├── NullBindingConverter.php
                │   ├── BooleanBindingConverter.php
                │   ├── IntegerBindingConverter.php
                │   ├── DecimalBindingConverter.php
                │   ├── StringBindingConverter.php
                │   ├── BinaryBindingConverter.php
                │   ├── UuidBindingConverter.php
                │   ├── EnumBindingConverter.php
                │   ├── JsonBindingConverter.php
                │   └── DateTimeBindingConverter.php
                │
                ├── Driver/
                │   ├── DriverBinding.php
                │   ├── DriverBindingPosition.php
                │   ├── DriverBindingOptions.php
                │   └── DriverBindingCapabilities.php
                │
                ├── Resource/
                │   ├── BindingSession.php
                │   ├── BindingResourceRegistry.php
                │   ├── BindingResourceProfile.php
                │   ├── LobBindingResource.php
                │   ├── StreamBindingResource.php
                │   └── StreamOwnership.php
                │
                ├── Result/
                │   ├── BoundPreparedStatement.php
                │   ├── BoundParameterSet.php
                │   └── OutputBindingResult.php
                │
                ├── Security/
                │   ├── BindingSensitivity.php
                │   ├── SensitivityLevel.php
                │   ├── SensitiveBindingValue.php
                │   └── BindingRedactor.php
                │
                ├── Budget/
                │   └── BindingBudget.php
                │
                ├── Extension/
                │   ├── ParameterBindingExtension.php
                │   ├── ParameterBindingExtensionDescriptor.php
                │   └── ParameterBindingExtensionRegistry.php
                │
                ├── Telemetry/
                │   ├── ParameterBindingObserver.php
                │   ├── ParameterBindingTelemetry.php
                │   └── ParameterBindingMetrics.php
                │
                └── Exception/
                    ├── ParameterBindingException.php
                    ├── BindingValidationException.php
                    ├── MissingBindingException.php
                    ├── UnexpectedBindingException.php
                    ├── DuplicateBindingException.php
                    ├── BindingLayoutMismatchException.php
                    ├── BindingTypeException.php
                    ├── BindingTypeMismatchException.php
                    ├── BindingConversionException.php
                    ├── BindingOverflowException.php
                    ├── BindingEncodingException.php
                    ├── BindingJsonException.php
                    ├── BindingResourceException.php
                    ├── DriverBindingException.php
                    ├── BindingUnsupportedTypeException.php
                    ├── BindingBudgetException.php
                    ├── BindingSecurityException.php
                    └── BindingInvariantException.php
```

---

# 256. Dependency direction

```text
Statement Execution
       ↓
Parameter Binding
       ↓
Prepared Statement Handle
       ↓
Driver Binding Contract
       ↓
Driver
```

El Type System puede ser consumido como metadata:

```text
Database Type System
       ↓
Parameter Binding
```

Nunca al revés.

---

# 257. Forbidden dependencies

Parameter Binding System no dependerá de:

```text
ORM
EntityManager
Repository
UnitOfWork
IdentityMap
Hydrator
Query Optimizer
Query Planner
HTTP Request
Authentication
Authorization
```

---

# 258. Architectural invariants

## DB-BIND-001

Parameter Binding no interpolará valores dentro del SQL.

## DB-BIND-002

Runtime values permanecerán separados del SQL estructural.

## DB-BIND-003

ParameterId será distinto de ExecutionParameterSlotId.

## DB-BIND-004

ExecutionParameterSlotId será distinto de BindingSlotId.

## DB-BIND-005

BindingSlotId será distinto de SQLPlaceholder.

## DB-BIND-006

SQLPlaceholder será distinto de DriverBindingPosition.

## DB-BIND-007

CompiledBindingLayout no contendrá runtime values.

## DB-BIND-008

RuntimeBindingSet será operation-scoped.

## DB-BIND-009

Binding layout podrá reutilizarse entre ejecuciones compatibles.

## DB-BIND-010

Runtime bindings no se reutilizarán implícitamente.

## DB-BIND-011

Un ParameterId podrá producir múltiples binding slots.

## DB-BIND-012

El binder respetará exactamente el placeholder layout compilado.

## DB-BIND-013

El binder no cambiará el número de placeholders.

## DB-BIND-014

El binder no generará SQL.

## DB-BIND-015

El binder no reinterpretará Query AST.

## DB-BIND-016

El binder no optimizará queries.

## DB-BIND-017

El binder no planificará queries.

## DB-BIND-018

Statement compatibility será validada antes del binding.

## DB-BIND-019

Missing bindings fallarán antes de ejecución cuando sean detectables.

## DB-BIND-020

Unexpected bindings fallarán por defecto.

## DB-BIND-021

Optional bindings serán explícitos.

## DB-BIND-022

Slot resolution no dependerá de parsing del placeholder.

## DB-BIND-023

Runtime PHP type no definirá por sí solo semantic database type.

## DB-BIND-024

Expected semantic type tendrá prioridad sobre inferencia genérica.

## DB-BIND-025

Explicit type override será validado.

## DB-BIND-026

Semantic type será distinto de driver binding type.

## DB-BIND-027

Value conversion preservará significado.

## DB-BIND-028

NULL permanecerá NULL.

## DB-BIND-029

NULL no se convertirá a string `"NULL"`.

## DB-BIND-030

Typed NULL será soportable.

## DB-BIND-031

NOT_NULL binding rechazará NULL cuando sea verificable.

## DB-BIND-032

Boolean semantic value será independiente de su storage representation.

## DB-BIND-033

Integer overflow no será truncado silenciosamente.

## DB-BIND-034

Decimal no será convertido automáticamente a float.

## DB-BIND-035

String y binary serán tipos conceptualmente distintos.

## DB-BIND-036

Encoding corruption no será silenciosa.

## DB-BIND-037

UUID tendrá mapping explícito.

## DB-BIND-038

Enum tendrá mapping explícito.

## DB-BIND-039

SQL NULL será distinto de JSON null.

## DB-BIND-040

JSON encoding failure producirá error.

## DB-BIND-041

Date/time conversion no dependerá de timezone global oculto.

## DB-BIND-042

Timezone policy será explícita.

## DB-BIND-043

Array expansion que cambie SQL no ocurrirá en runtime binding.

## DB-BIND-044

LOB tendrá lifecycle explícito.

## DB-BIND-045

Stream tendrá ownership explícito.

## DB-BIND-046

Borrowed stream no será cerrado por VoltStack sin permiso.

## DB-BIND-047

Transferred stream tendrá cleanup definido.

## DB-BIND-048

No se hará rewind implícito sin contract.

## DB-BIND-049

Stream replayability será explícita.

## DB-BIND-050

One-shot binding afectará retry eligibility.

## DB-BIND-051

BindingDirection será explícita.

## DB-BIND-052

OUTPUT no será simulado sobre driver incompatible.

## DB-BIND-053

BoundPreparedStatement será operation-scoped.

## DB-BIND-054

BoundPreparedStatement no entrará al prepared statement cache.

## DB-BIND-055

Sensitivity classification no alterará query semantics.

## DB-BIND-056

Sensitive values serán redacted en diagnostics.

## DB-BIND-057

Runtime values no participarán en fingerprints estructurales.

## DB-BIND-058

Temporary binding resources tendrán ownership registrado.

## DB-BIND-059

Binding failure ejecutará cleanup.

## DB-BIND-060

Partial binding no dejará statement reusable sin reset.

## DB-BIND-061

Driver binding se realizará mediante adapter/contract.

## DB-BIND-062

Core no dependerá directamente de PDO parameter constants.

## DB-BIND-063

Binding order será determinista.

## DB-BIND-064

Bind-by-value será preferido cuando sea posible.

## DB-BIND-065

Bind-by-reference será explícito.

## DB-BIND-066

Reference bindings serán limpiados.

## DB-BIND-067

Post-binding validation verificará todos los required slots.

## DB-BIND-068

READY no significará EXECUTED.

## DB-BIND-069

Critical structural validation no podrá deshabilitarse.

## DB-BIND-070

Converter registry será frozen después de bootstrap.

## DB-BIND-071

Converter conflicts no usarán last-wins.

## DB-BIND-072

Unknown objects no usarán __toString fallback.

## DB-BIND-073

ORM entities no serán convertidas automáticamente a IDs.

## DB-BIND-074

Collections no expandirán placeholders en binding runtime.

## DB-BIND-075

Binding budget será bounded.

## DB-BIND-076

Unknown parameter limit no significará unlimited.

## DB-BIND-077

Large values no serán buffered ilimitadamente.

## DB-BIND-078

JSON serialization será bounded.

## DB-BIND-079

Persistent workers no conservarán previous runtime values.

## DB-BIND-080

Prepared statement cache no almacenará previous bindings.

## DB-BIND-081

Shared singleton services no almacenarán current bindings.

## DB-BIND-082

Cada concurrent execution tendrá BindingSession independiente.

## DB-BIND-083

Exclusive prepared statement tendrá una sola binding mutation concurrente.

## DB-BIND-084

RuntimeBindingSet será preferiblemente immutable.

## DB-BIND-085

Untrusted runtime values siempre entrarán como bindings.

## DB-BIND-086

Identifiers no serán runtime value bindings.

## DB-BIND-087

Operators no serán runtime value bindings.

## DB-BIND-088

SQL keywords no serán runtime value bindings.

## DB-BIND-089

Function names no serán runtime value bindings.

## DB-BIND-090

ORDER BY direction no será runtime value binding.

## DB-BIND-091

Sensitive value lifetime será minimizado.

## DB-BIND-092

Telemetry no registrará runtime values por defecto.

## DB-BIND-093

Metrics no usarán parameter values como labels.

## DB-BIND-094

Driver behavior será capability-driven.

## DB-BIND-095

Vendor conditionals no estarán dispersos en binder core.

## DB-BIND-096

Core será independiente de PDO.

## DB-BIND-097

SQLite storage class será distinto de VoltStack semantic type.

## DB-BIND-098

MariaDB podrá divergir de MySQL mediante contract propio.

## DB-BIND-099

Replayability metadata será propagable al Execution Engine.

## DB-BIND-100

Deferred bindings serán explícitos.

## DB-BIND-101

Deferred values se evaluarán en una fase definida.

## DB-BIND-102

Converters no usarán wall clock implícitamente.

## DB-BIND-103

Converters no usarán randomness implícito.

## DB-BIND-104

Extensions serán tipadas.

## DB-BIND-105

Extension registry será immutable tras bootstrap.

## DB-BIND-106

Runtime requests no registrarán global binders.

## DB-BIND-107

Core override estará denegado por defecto.

## DB-BIND-108

Binding exceptions conservarán safe diagnostics.

## DB-BIND-109

Driver causes podrán preservarse sin exponer secretos.

## DB-BIND-110

Cleanup failure no ocultará primary failure.

## DB-BIND-111

Pre-driver application validation error no invalidará statement innecesariamente.

## DB-BIND-112

Partial driver binding podrá requerir reset.

## DB-BIND-113

Failure impact será driver-aware.

## DB-BIND-114

Parameter lookup evitará complejidad cuadrática.

## DB-BIND-115

Converter resolution metadata podrá precompilarse.

## DB-BIND-116

Converted runtime values no se cachearán globalmente.

## DB-BIND-117

CompiledParameterBindingPlan no contendrá runtime values.

## DB-BIND-118

Binding plan será reusable sólo bajo compatible driver contract.

## DB-BIND-119

Parameter Binding System no ejecutará el statement.

## DB-BIND-120

Parameter Binding System no hará retry de queries.

## DB-BIND-121

Parameter Binding System no hará transaction management.

## DB-BIND-122

Parameter Binding System no hará ORM hydration.

## DB-BIND-123

Parameter Binding System no consultará live schema.

## DB-BIND-124

Parameter Binding System no abrirá conexiones.

## DB-BIND-125

Parameter Binding System no modificará tenant context.

## DB-BIND-126

Binding security no dependerá de escaping manual de values.

## DB-BIND-127

Raw SQL seguirá siendo responsabilidad de capas anteriores.

## DB-BIND-128

Binding conversion será deterministic para input/context equivalentes.

## DB-BIND-129

Binding cleanup será idempotente cuando sea posible.

## DB-BIND-130

Binding resources no sobrevivirán más allá de su ownership contract.

## DB-BIND-131

Statement invalidation durante binding impedirá ejecución.

## DB-BIND-132

Connection invalidation durante binding impedirá ejecución.

## DB-BIND-133

Driver binding positions serán validadas.

## DB-BIND-134

Placeholder style será compatible con el prepared statement.

## DB-BIND-135

Binding layout fingerprint deberá corresponder al command preparado.

## DB-BIND-136

Output parameters tendrán lifecycle explícito.

## DB-BIND-137

Driver-native custom types serán capability-driven.

## DB-BIND-138

Custom converters no cambiarán estructura SQL.

## DB-BIND-139

Custom converters no ejecutarán queries.

## DB-BIND-140

Custom converters no realizarán transaction control.

## DB-BIND-141

Custom converters no dependerán de ORM.

## DB-BIND-142

Binding diagnostics respetarán sensitivity metadata.

## DB-BIND-143

Binding telemetry será bounded.

## DB-BIND-144

Source value y converted value tendrán ownership definido cuando sean resources.

## DB-BIND-145

Resource cleanup ocurrirá también ante cancellation.

## DB-BIND-146

Cancellation durante conversion impedirá nuevos driver bindings.

## DB-BIND-147

Binding budget exhaustion será atómico respecto a ejecución.

## DB-BIND-148

Un statement parcialmente bound nunca será ejecutado por error.

## DB-BIND-149

BoundPreparedStatement representará binding completo y validado.

## DB-BIND-150

El binder será una frontera de representación de valores, no una segunda capa de compilación.

---

# 259. Invariante maestro de completitud

Sea:

```text
RequiredSlots(L)
```

el conjunto de slots requeridos por `CompiledBindingLayout L`.

Y:

```text
ResolvedSlots(B)
```

los slots resueltos desde `RuntimeBindingSet B`.

Antes de ejecutar deberá cumplirse:

```text
RequiredSlots(L)
⊆
ResolvedSlots(B)
```

Para modo estricto:

```text
RequiredSlots(L)
=
ResolvedSlots(B)
```

salvo slots explícitamente opcionales.

---

# 260. Invariante maestro de correspondencia

Para cada binding slot:

```text
∀ s ∈ CompiledBindingLayout:
    exactlyOneRuntimeResolution(s)
```

salvo ocurrencias repetidas que referencien intencionalmente el mismo `ExecutionParameterSlotId`.

---

# 261. Invariante maestro de tipo

```text
Bindable(v, T, D)
=
Compatible(v, T)
∧
ConvertibleWithoutSemanticLoss(v, T)
∧
DriverSupports(Representation(T), D)
```

---

# 262. Invariante maestro de seguridad

```text
RuntimeValue
→
DriverBinding
```

y nunca:

```text
RuntimeValue
→
SQLGrammar
```

---

# 263. Invariante maestro de atomicidad

```text
BindingFailure
⇒
¬Executable(Statement)
```

hasta completar:

```text
reset
or
invalidation
```

---

# 264. Invariante maestro de persistent runtime

Después de liberar la ejecución:

```text
RetainedRuntimeBindingReferences = ∅
```

salvo recursos explícitamente transferidos a un owner todavía activo.

---

# 265. Invariante maestro de replayability

```text
ReplayableBindingSet
=
∀ binding:
    binding.replayable = true
```

Esta información podrá utilizarse posteriormente por el Retry System.

---

# 266. Ejemplo completo

Consulta lógica:

```sql
SELECT id, name
FROM users
WHERE tenant_id = ?
  AND status = ?
  AND created_at >= ?
```

Layout:

```text
BindingSlot B1
├── ExecutionSlot: E7
├── Placeholder: ?
├── DriverPosition: 1
└── Type: UUID

BindingSlot B2
├── ExecutionSlot: E8
├── Placeholder: ?
├── DriverPosition: 2
└── Type: ENUM(UserStatus)

BindingSlot B3
├── ExecutionSlot: E9
├── Placeholder: ?
├── DriverPosition: 3
└── Type: DATETIME
```

Runtime:

```text
E7 → TenantId(...)
E8 → UserStatus::ACTIVE
E9 → DateTimeImmutable(...)
```

Binding:

```text
TenantId
→ UUID converter
→ canonical UUID
→ Driver STRING
→ position 1

UserStatus::ACTIVE
→ Enum mapper
→ "active"
→ Driver STRING
→ position 2

DateTimeImmutable
→ DateTime policy
→ canonical timestamp
→ Driver STRING/NATIVE
→ position 3
```

Resultado:

```text
BoundPreparedStatement
```

listo para ejecución.

---

# 267. Anti-patterns

## Anti-pattern 1 — SQL interpolation

```php
$sql = "WHERE id = {$id}";
```

---

## Anti-pattern 2 — type guessing

```php
$type = is_numeric($value)
    ? INTEGER
    : STRING;
```

---

## Anti-pattern 3 — entity magic

```php
if ($value instanceof Entity) {
    $value = $value->getId();
}
```

---

## Anti-pattern 4 — collection expansion

```php
foreach ($ids as $id) {
    $sql .= '?,';
}
```

dentro del binder.

---

## Anti-pattern 5 — global bindings

```php
static $currentBindings;
```

---

## Anti-pattern 6 — secret telemetry

```php
$logger->debug('binding', [
    'value' => $password,
]);
```

---

## Anti-pattern 7 — float money

```php
$value = (float) $money;
```

---

## Anti-pattern 8 — hidden timezone

```php
$date->setTimezone(
    new DateTimeZone(date_default_timezone_get())
);
```

---

## Anti-pattern 9 — unbounded LOB buffering

```php
$data = stream_get_contents($hugeStream);
```

sin budget.

---

## Anti-pattern 10 — object stringify fallback

```php
$value = (string) $object;
```

para tipos desconocidos.

---

# 268. Fórmula arquitectónica

```text
Parameter Binding System
=
Binding Layout Validation
+
Runtime Slot Resolution
+
Semantic Type Resolution
+
Platform Representation Resolution
+
Driver Type Resolution
+
Value Conversion
+
Resource Ownership
+
Driver Binding
+
Binding Validation
+
Security
+
Budget Enforcement
+
Cleanup
+
Replayability Metadata
+
Telemetry
```

---

# 269. Fórmula del hot path

```text
Runtime Binding
    ↓
Precompiled Binding Instruction
    ↓
Resolved Converter
    ↓
Driver-Compatible Value
    ↓
Driver Bind
```

Idealmente:

```text
O(n)
```

para `n` parámetros.

---

# 270. Principio final

> **Parameters are values, not SQL syntax.**

Por tanto:

```text
Binding
=
Value Representation
```

No:

```text
Binding
=
SQL Generation
```

Y:

```text
Safe Binding
=
Explicit Identity
+
Explicit Type
+
Compiled Placeholder Mapping
+
Semantic-Preserving Conversion
+
Driver Contract
+
Resource Ownership
+
Sensitive Data Protection
+
Bounded Runtime State
```

---

# 271. Relación con el siguiente documento

Una vez que existe:

```text
BoundPreparedStatement
```

el Statement Execution System puede ejecutar el command.

El siguiente problema es representar de manera uniforme lo que el DBMS devuelve:

```text
Driver Execution
      ↓
raw driver result
      ↓
Result System
      ↓
DatabaseResult
```

Esta capa deberá abstraer:

```text
rows
affected rows
scalar results
RETURNING rows
metadata
multiple result sets
cursor-backed results
empty results
```

sin introducir todavía ORM hydration.

---

# 272. Siguiente documento

```text
81_DATABASE_RESULT_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Result System
├── Result Architecture
├── Result Contract
├── Result Kinds
├── DatabaseResult
├── Row Result
├── Scalar Result
├── Affected Rows Result
├── Returning Result
├── Empty Result
├── Result Metadata
├── Column Metadata
├── Result Shape
├── Row Representation
├── Value Representation
├── Driver Result Adapter
├── Result Ownership
├── Result Lifecycle
├── Buffered Results
├── Cursor-backed Results
├── Multiple Result Sets
├── Result Consumption
├── Result State
├── Result Cleanup
├── Result Errors
├── Result Security
├── Result Telemetry
├── Extension System
├── Persistent Runtime Safety
└── Architectural Invariants
```

manteniendo la separación fundamental:

```text
Database Result
≠
Result Cursor
≠
Hydrated Entity
≠
ORM Collection
```