# 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md

# VoltStack Quantum Database
## Developer Debug Toolbar Integration

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 225 — Developer Debug Toolbar Integration  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `224_DATABASE_DEBUG_INFORMATION_SYSTEM.md`  
**Siguiente documento:** `226_DATABASE_SECURITY_ARCHITECTURE.md`

---

# 1. Propósito

`Developer Debug Toolbar Integration` define la arquitectura mediante la cual VoltStack Database expondrá su información diagnóstica a herramientas interactivas de desarrollo sin permitir que dichas herramientas accedan directamente al estado vivo del motor de base de datos.

La regla central será:

> **La Developer Debug Toolbar es una capa de presentación y exploración sobre `DatabaseDebugInformation`; nunca será una puerta de acceso al Database Runtime.**

La arquitectura deberá mantener:

```text
Database Runtime
      ↓
Telemetry
      ↓
Debug Information
      ↓
Debug Toolbar Integration
      ↓
Developer UI
```

y nunca:

```text
Developer UI
      ↓
EntityManager
Connection
UnitOfWork
Driver
PDO
TransactionContext
ResultCursor
```

---

# 2. Objetivos

La integración deberá permitir inspeccionar de manera rápida:

```text
queries
duraciones
conexiones
transacciones
ORM
IdentityMap
UnitOfWork
cache
slow queries
N+1
replicas
shards
diagnósticos
errores
runtime
```

sin comprometer:

```text
seguridad
aislamiento
rendimiento
compatibilidad
arquitectura
persistent runtimes
```

---

# 3. Principio de separación

Deben mantenerse claramente separados:

```text
Database Debug Information
≠
Debug Toolbar Integration
≠
Debug Toolbar UI
```

La primera contiene información diagnóstica.

La segunda adapta dicha información para herramientas de desarrollo.

La tercera presenta visualmente los datos.

---

# 4. Posición arquitectónica

```text
VoltStack Database
│
├── Telemetry
│   ├── Query
│   ├── Connection
│   ├── Transaction
│   └── ORM
│
├── Profiler
├── Slow Query Detection
├── N+1 Detection
│
├── Debug Information System
│
└── Debug Toolbar Integration
        │
        ├── Database Panel
        ├── Query Explorer
        ├── Timeline
        ├── Diagnostics
        └── Toolbar API
```

---

# 5. Dependencia correcta

```text
Toolbar
    ↓
Toolbar Adapter
    ↓
DatabaseDebugInformation
```

No:

```text
Toolbar
    ↓
Database Services
```

---

# 6. Beneficio

Esta frontera permite sustituir la interfaz sin modificar Database:

```text
VoltStack Toolbar
Clockwork adapter
IDE extension
CLI inspector
Web profiler
Custom enterprise UI
```

todos consumiendo el mismo modelo diagnóstico.

---

# 7. Arquitectura general

```text
Database Runtime
      │
      ▼
Database Telemetry
      │
      ▼
DebugInformationCollector
      │
      ▼
DatabaseDebugInformation
      │
      ▼
DatabaseDebugToolbarAdapter
      │
      ▼
DatabaseDebugToolbarPayload
      │
      ├── Summary
      ├── Queries
      ├── Timeline
      ├── Connections
      ├── Transactions
      ├── ORM
      ├── Cache
      ├── Slow Queries
      ├── N+1
      ├── Distribution
      └── Diagnostics
```

---

# 8. Toolbar Integration Contract

Contrato principal:

```php
interface DatabaseDebugToolbarAdapter
{
    public function build(
        DatabaseDebugInformation $information,
        DebugToolbarContext $context,
    ): DatabaseDebugToolbarPayload;
}
```

El adapter:

```text
reads
transforms
formats
groups
sorts
filters
```

pero no ejecuta operaciones de base de datos.

---

# 9. DebugToolbarContext

```php
final readonly class DebugToolbarContext
{
    public function __construct(
        public DebugToolbarMode $mode,
        public DebugToolbarAccess $access,
        public DebugToolbarPreferences $preferences,
    ) {}
}
```

---

# 10. Toolbar modes

```php
enum DebugToolbarMode
{
    case DEVELOPMENT;
    case TESTING;
    case STAGING;
    case PRODUCTION_RESTRICTED;
}
```

---

# 11. Mode ≠ authorization

`DEVELOPMENT` no significa:

```text
show everything
```

La autorización sigue siendo independiente.

---

# 12. Database toolbar panel

La toolbar general de VoltStack podrá contener:

```text
Request
Routing
Views
Components
Events
Cache
Jobs
Database
Telemetry
Security
```

Database será un panel especializado.

---

# 13. Database panel badge

La vista compacta podría mostrar:

```text
DB
18 queries
42 ms
2 slow
1 N+1
```

sin abrir el panel completo.

---

# 14. Severity indicator

El badge podrá reflejar:

```text
NORMAL
NOTICE
WARNING
ERROR
CRITICAL
```

derivado de diagnostics.

---

# 15. No opaque score

Evitar:

```text
DB Health: 87/100
```

como mecanismo principal.

Preferir evidencia concreta.

---

# 16. DatabaseDebugToolbarPayload

```php
final readonly class DatabaseDebugToolbarPayload
{
    public function __construct(
        public ToolbarDatabaseSummary $summary,
        public ToolbarQueryCollection $queries,
        public ToolbarTimeline $timeline,
        public ToolbarConnectionCollection $connections,
        public ToolbarTransactionCollection $transactions,
        public ToolbarORMView $orm,
        public ToolbarCacheView $cache,
        public ToolbarDiagnosticCollection $diagnostics,
        public ToolbarDistributionView $distribution,
        public ToolbarMetadata $metadata,
    ) {}
}
```

---

# 17. Payload ≠ DatabaseDebugInformation

El payload de toolbar será una proyección UI-friendly.

```text
DatabaseDebugInformation
       ↓
Toolbar Adapter
       ↓
DatabaseDebugToolbarPayload
```

---

# 18. Razón

El modelo diagnóstico puede contener información útil para:

```text
CLI
tests
logs
IDE
support tools
```

que no necesita la toolbar.

---

# 19. Toolbar summary

Ejemplo:

```text
DATABASE

Queries              18
Database time        42.7 ms
Connection wait       3.4 ms
Hydration             9.1 ms
Transactions            1
Slow queries            2
N+1 patterns            1
Cache hits              7
Warnings                3
```

---

# 20. Time accounting

La toolbar deberá distinguir:

```text
Database Execution Time
Connection Acquisition Time
Hydration Time
ORM Processing Time
```

cuando la información esté disponible.

---

# 21. Aggregate time caveat

Con concurrencia:

```text
sum(query durations)
```

puede ser mayor que:

```text
request wall time
```

La UI deberá evitar presentarlos como equivalentes.

---

# 22. Query explorer

El panel principal deberá incluir un Query Explorer.

Columnas sugeridas:

```text
#
Type
Query
Duration
Rows
Connection
Transaction
Cache
Origin
Status
```

---

# 23. Query record

```php
final readonly class ToolbarQueryRecord
{
    public function __construct(
        public DebugRecordId $id,
        public QueryOperation $operation,
        public string $displayQuery,
        public Duration $duration,
        public ?int $rows,
        public ToolbarQueryStatus $status,
        public ToolbarQueryFlags $flags,
    ) {}
}
```

---

# 24. Query flags

```text
SLOW
N+1
CACHED
RETRIED
FAILED
CANCELLED
TRANSACTIONAL
REPLICA
SHARDED
```

---

# 25. Query detail

Seleccionar una query podrá abrir:

```text
Query
Parameters
Timing
Execution
Connection
Transaction
ORM
Cache
Source
Diagnostics
```

---

# 26. Query detail source

Todo deberá derivarse de:

```text
DatabaseDebugInformation
```

o de una operación diagnóstica explícita autorizada.

---

# 27. SQL display

La UI podrá mostrar:

```sql
SELECT *
FROM users
WHERE email = ?
  AND active = ?
```

con parámetros:

```text
email   string   [REDACTED]
active  bool     true
```

según policy.

---

# 28. Copy SQL

La toolbar podrá ofrecer:

```text
Copy normalized SQL
```

y, cuando esté permitido:

```text
Copy executable-style SQL
```

---

# 29. Copy-safe rule

Nunca copiar automáticamente secretos dentro del SQL.

---

# 30. Safe copy modes

```php
enum ToolbarSqlCopyMode
{
    case NORMALIZED;
    case PLACEHOLDERS;
    case REDACTED;
}
```

---

# 31. Parameter interpolation

No deberá hacerse mediante concatenación ingenua.

---

# 32. Query formatting

La presentación SQL será responsabilidad de un formatter:

```php
interface DebugSqlFormatter
{
    public function format(
        DebugQueryText $query
    ): FormattedDebugSql;
}
```

---

# 33. Formatter ≠ compiler

El formatter nunca generará SQL para ejecución.

---

# 34. Syntax highlighting

Será una preocupación puramente visual.

---

# 35. Query filtering

La UI podrá filtrar por:

```text
SELECT
INSERT
UPDATE
DELETE
DDL
slow
N+1
failed
cached
transaction
writer
replica
shard
origin
```

---

# 36. Query search

Podrá buscar sobre:

```text
normalized SQL
fingerprint
entity type
repository
source
diagnostic code
```

solo en información ya autorizada.

---

# 37. Filtering ≠ new database query

Todos los filtros serán sobre el debug payload.

---

# 38. Sorting

Permitido:

```text
duration
sequence
rows
operation
status
```

---

# 39. Default order

Orden cronológico de ejecución.

---

# 40. Slowest view

También podrá existir:

```text
Top Slowest Queries
```

---

# 41. Query grouping

La toolbar podrá agrupar por:

```text
fingerprint
source
entity
connection
transaction
relationship
```

---

# 42. Fingerprint grouping

Ejemplo:

```text
Fingerprint q_42

Executions: 125
Total:      310 ms
Average:    2.48 ms
Maximum:    6.10 ms
```

---

# 43. Grouping and N+1

Un fingerprint repetido no deberá etiquetarse automáticamente como N+1.

Solo mostrar:

```text
N+1
```

cuando el detector canónico lo determine.

---

# 44. Timeline

La toolbar podrá visualizar:

```text
Request
│
├── Connection Acquire
├── Query #1
├── Query #2
├── Transaction
│   ├── Query #3
│   ├── ORM Flush
│   └── Commit
└── Query #4
```

---

# 45. Timeline representation

Conceptualmente:

```text
0ms       20ms       40ms       60ms

Request      ━━━━━━━━━━━━━━━━━━━━━━━━

Connection   ━━

Query #1       ━━━

Query #2           ━━━━━━━

Hydration               ━━━

Tx                          ━━━━━━━
```

---

# 46. Concurrent operations

La timeline deberá representar overlaps.

No deberá forzar una secuencia falsa.

---

# 47. Timeline layers

Podrán existir lanes:

```text
Queries
Connections
Transactions
ORM
Hydration
Cache
Distribution
```

---

# 48. Zoom

La UI podrá permitir zoom temporal sin modificar los datos originales.

---

# 49. Query timeline selection

Seleccionar una query en timeline deberá abrir el mismo record del Query Explorer.

Esto se logra mediante:

```text
DebugRecordId
```

---

# 50. Connection panel

Podrá mostrar:

```text
Logical Connection
Role
Endpoint Alias
Pool
Acquisition Time
Reuse
Queries
Errors
```

---

# 51. Example

```text
writer-primary

Role:          WRITE
Pool:          default
Acquired:      2.1 ms
Reused:        yes
Queries:       4
Transactions:  1
```

---

# 52. Connection secret protection

Nunca mostrar:

```text
password
full DSN
private key
token
```

---

# 53. Host information

La exposición del host dependerá de policy.

---

# 54. Pool panel

Cuando pooling esté activo:

```text
Pool
├── Active
├── Idle
├── Waiting
├── Capacity
├── Acquisition P95
└── Saturation warnings
```

si esas métricas están disponibles.

---

# 55. Snapshot limitation

Estos valores representan el scope observado.

No necesariamente el estado global actual del pool.

---

# 56. Transaction panel

Deberá mostrar:

```text
Transaction ID
Outcome
Duration
Isolation
Attempts
Savepoints
Queries
Retries
Deadlocks
```

---

# 57. UNKNOWN outcome

Visualmente deberá destacarse.

Ejemplo:

```text
Outcome: UNKNOWN
```

No:

```text
Failed
```

---

# 58. Transaction timeline

Podrá visualizar:

```text
BEGIN
  Query
  Query
  SAVEPOINT
  Query
  RELEASE
COMMIT
```

---

# 59. Nested transaction display

Debe distinguir:

```text
logical nested scope
```

de:

```text
physical transaction
```

---

# 60. Retry display

Ejemplo:

```text
Transaction
├── Attempt 1
│   └── DEADLOCK
└── Attempt 2
    └── COMMITTED
```

---

# 61. ORM panel

La toolbar deberá poder mostrar:

```text
EntityManager
IdentityMap
UnitOfWork
Hydration
Flush
Relationships
Lazy Loads
Eager Loads
```

---

# 62. ORM summary

Ejemplo:

```text
EntityManager

State:              OPEN
IdentityMap:         42
Peak IdentityMap:    67

UnitOfWork
  NEW:                2
  DIRTY:              4
  REMOVED:            0

Flushes:              1
Hydrated entities:   52
Lazy loads:           8
```

---

# 63. No live entity browser

La V1 no deberá permitir:

```text
browse every live entity object
```

desde la toolbar.

---

# 64. Entity summary

Puede mostrar:

```text
App\Domain\User
managed: 30
dirty: 2
```

sin serializar las entidades.

---

# 65. IdentityMap warning

Si crece significativamente:

```text
IdentityMap peak: 150,000
```

podrá aparecer:

```text
DB_ORM_IDENTITY_MAP_GROWTH
```

---

# 66. UnitOfWork detail

Puede mostrar:

```text
NEW
MANAGED
DIRTY
REMOVED
```

por tipo de entidad.

---

# 67. Dirty field values

No deberán mostrarse por default.

---

# 68. Hydration panel

Podrá mostrar:

```text
Hydration operations
Rows processed
Entities created
Entities reused
DTOs
Scalars
Hydration duration
```

---

# 69. Identity reuse

Una métrica útil:

```text
Entities reused from IdentityMap
```

sin exponer objetos.

---

# 70. Relationship panel

Podrá mostrar:

```text
eager loads
lazy loads
batch loads
relationship queries
partial relationship loads
```

---

# 71. N+1 integration

Las relaciones detectadas por N+1 deberán enlazarse desde:

```text
ORM
Queries
Diagnostics
```

al mismo detection record.

---

# 72. N+1 panel

Ejemplo:

```text
N+1 DETECTED

Relationship:
User.posts

Parent records:
100

Dependent queries:
100

Total DB time:
73 ms

Confidence:
HIGH
```

---

# 73. Recommendation

Podrá mostrar:

```text
Consider eager or batch loading.
```

sin modificar el código automáticamente.

---

# 74. Source location

Cuando esté disponible:

```text
app/Service/UserService.php:84
```

---

# 75. IDE integration

La toolbar podrá generar un:

```text
SourceLink
```

para abrir el archivo en un IDE.

---

# 76. SourceLink

```php
final readonly class ToolbarSourceLink
{
    public function __construct(
        public string $relativePath,
        public int $line,
        public ?string $ideUri,
    ) {}
}
```

---

# 77. IDE URI security

Debe ser generada desde rutas autorizadas.

Nunca desde user input arbitrario.

---

# 78. Supported IDE adapters

La arquitectura podrá soportar adapters para:

```text
VS Code
PhpStorm
Trae
custom IDE
```

sin que Database conozca ninguno de ellos.

---

# 79. IDELinkGenerator

```php
interface IDELinkGenerator
{
    public function generate(
        DebugSourceLocation $location
    ): ?ToolbarSourceLink;
}
```

---

# 80. Absolute paths

La UI no deberá necesitar conocer rutas absolutas del servidor.

---

# 81. Slow query panel

Podrá listar:

```text
Query
Duration
Threshold
Classification
Dominant Phase
Severity
```

---

# 82. Example

```text
SELECT users...

Total:      821 ms
DB:          31 ms
Hydration:  770 ms

Classification:
HYDRATION_DOMINATED
```

---

# 83. Importance

Esto evita asumir:

```text
slow query
=
slow database
```

---

# 84. Query profiler integration

El panel podrá mostrar:

```text
compile
connection
execution
fetch
hydration
ORM
```

si el Query Profiler produjo dichos datos.

---

# 85. Profile waterfall

```text
Compilation       1.1 ms
Connection        2.4 ms
DB Execution     20.2 ms
Fetch             4.8 ms
Hydration        42.7 ms
ORM               3.2 ms
```

---

# 86. Cache panel

Debe diferenciar:

```text
Query Cache
Result Cache
Metadata Cache
Entity Cache
Hydration Cache
Compiled Query Cache
```

---

# 87. Cache semantics

No reducir todo a:

```text
hit/miss
```

si el cache subsystem distingue:

```text
PHYSICAL_HIT
USABLE_HIT
STALE_REJECTED
MISS
```

---

# 88. Cache summary

Ejemplo:

```text
Result Cache

Usable hits:       12
Misses:             4
Stale rejected:     1
```

---

# 89. Cache key display

Mostrar:

```text
fingerprint
```

en lugar de la key completa cuando sea sensible.

---

# 90. Distribution panel

Cuando aplique:

```text
Read/Write Routing
Replicas
Failover
Sharding
Partition Routing
```

---

# 91. Read/write routing

Una query podrá mostrar:

```text
Intent: READ
Selected role: WRITER
Reason: ACTIVE_TRANSACTION
```

---

# 92. Routing reason

Debe provenir del sistema de routing.

La toolbar no inferirá la razón.

---

# 93. Replica view

Ejemplo:

```text
replica-02

Eligible: yes
Lag evidence: 14 ms
Confidence: HIGH
Queries: 8
```

---

# 94. Unknown lag

Mostrar:

```text
Lag: UNKNOWN
```

no:

```text
0 ms
```

---

# 95. Sticky routing

Puede visualizar:

```text
Writer selected because:
READ_YOUR_WRITES_STICKY_WINDOW
```

---

# 96. Sharding panel

Podrá mostrar:

```text
Query #31
Shards: 4
Strategy: GLOBAL_ORDERED
Merge: 6.4 ms
```

---

# 97. Fan-out warning

Una consulta distribuida amplia podrá producir:

```text
DB_DISTRIBUTION_HIGH_FANOUT
```

si existe tal diagnostic.

---

# 98. Partial shard failure

Debe mostrar explícitamente:

```text
PARTIAL
```

o:

```text
UNKNOWN
```

según outcome.

---

# 99. Diagnostics panel

Será una vista unificada de:

```text
performance
correctness
resource
security
ORM
distribution
runtime
```

---

# 100. Diagnostic card

Ejemplo:

```text
WARNING

N+1 relationship loading

Relationship:
User.posts

Queries:
101

Confidence:
HIGH

Recommendation:
Use eager/batch loading.
```

---

# 101. Diagnostic filters

```text
severity
category
code
confidence
```

---

# 102. Diagnostic ordering

Default:

```text
CRITICAL
ERROR
WARNING
NOTICE
INFO
```

con prioridad secundaria por impacto.

---

# 103. Documentation links

Cada diagnostic podrá enlazar a documentación oficial mediante un identificador seguro.

---

# 104. No dynamic external URL requirement

La UI deberá poder resolver documentación local/offline.

---

# 105. Debug toolbar API

La integración podrá exponer un endpoint interno conceptual:

```text
/_voltstack/debug/database/{session}
```

pero la ruta concreta pertenecerá a la integración HTTP, no al Database core.

---

# 106. Database API contract

El core puede ofrecer:

```php
interface DatabaseDebugToolbarProvider
{
    public function payload(
        DebugSessionId $sessionId,
        DebugToolbarContext $context,
    ): DatabaseDebugToolbarPayload;
}
```

---

# 107. Endpoint authorization

El endpoint deberá verificar:

```text
environment
session
authorization
expiration
access policy
```

---

# 108. DebugSessionId ≠ access token

Nunca confiar únicamente en conocer el ID.

---

# 109. Session retrieval

Flujo:

```text
Browser Request
     ↓
Database operations
     ↓
Debug snapshot finalized
     ↓
Debug store
     ↓
Response contains opaque debug reference
     ↓
Toolbar requests debug payload
     ↓
Authorization
     ↓
Payload
```

---

# 110. Inline payload alternative

Para pequeños payloads podría incluirse información básica directamente en la respuesta de desarrollo.

---

# 111. Preferred architecture

Para payloads grandes:

```text
reference
+
lazy panel retrieval
```

es preferible.

---

# 112. Lazy toolbar retrieval

`Lazy` aquí significa carga de UI.

No:

```text
DATABASE_LAZY_COLLECTION_SYSTEM
```

---

# 113. Panel-specific retrieval

La toolbar podría solicitar:

```text
summary
queries
transactions
ORM
diagnostics
```

por separado.

---

# 114. Snapshot consistency

Todos los paneles de una sesión deberán derivarse del mismo debug snapshot o de una versión identificable.

---

# 115. Payload version

Cada respuesta incluirá:

```text
debug schema version
toolbar payload version
```

---

# 116. Toolbar payload version

Es distinta de:

```text
DatabaseDebugInformationVersion
```

---

# 117. Why separate versions?

Porque:

```text
diagnostic schema
```

puede evolucionar independientemente de:

```text
UI transport schema
```

---

# 118. ToolbarMetadata

```php
final readonly class ToolbarMetadata
{
    public function __construct(
        public ToolbarPayloadVersion $version,
        public DebugSessionId $sessionId,
        public DebugCoverage $coverage,
        public bool $truncated,
        public int $droppedRecords,
    ) {}
}
```

---

# 119. Partial data indicator

La UI deberá indicar:

```text
Partial telemetry data
```

cuando coverage no sea COMPLETE.

---

# 120. Truncated indicator

Ejemplo:

```text
Showing 500 of 24,818 queries.
```

---

# 121. Never hide truncation

No presentar una lista truncada como si fuera completa.

---

# 122. Query pagination inside toolbar

La toolbar podrá paginar localmente el debug dataset.

Esto es distinto de:

```text
DATABASE_PAGINATION_SYSTEM
```

porque no pagina la consulta de negocio.

---

# 123. Large debug snapshots

Podrán utilizar:

```text
server-side debug payload pagination
```

sobre el store de diagnostics.

---

# 124. Debug pagination ≠ application database pagination

No compartir semántica innecesariamente.

---

# 125. Search indexes

Un debug store avanzado podrá indexar:

```text
DebugRecordId
fingerprint
diagnostic code
```

pero no es requisito del core V1.

---

# 126. On-demand EXPLAIN

La toolbar podrá ofrecer:

```text
Explain Query
```

como acción explícita.

---

# 127. Critical distinction

```text
Viewing Query
≠
Running EXPLAIN
```

---

# 128. EXPLAIN architecture

```text
Toolbar
   ↓
Explicit Diagnostic Action
   ↓
Authorization
   ↓
Safety Validation
   ↓
Query Diagnostic Service
   ↓
Query Engine / Connection
   ↓
Platform EXPLAIN capability
   ↓
Safe Explain Snapshot
   ↓
Toolbar
```

---

# 129. EXPLAIN not automatic

Nunca ejecutar automáticamente al abrir el panel.

---

# 130. EXPLAIN authorization

Debe ser explícita.

---

# 131. EXPLAIN capability

Usará:

```text
Platform Capability System
```

No:

```php
if ($database === 'mysql')
```

---

# 132. EXPLAIN mode

Distinguir:

```text
PLAN_ONLY
ANALYZE
```

cuando la plataforma lo permita.

---

# 133. ANALYZE danger

Algunas variantes pueden ejecutar realmente la query.

Por tanto:

> **EXPLAIN ANALYZE no deberá ejecutarse automáticamente ni tratarse como una operación puramente descriptiva.**

---

# 134. Mutation queries

Acciones de explain sobre:

```text
INSERT
UPDATE
DELETE
DDL
```

deberán tener políticas más restrictivas.

---

# 135. Safe default

V1:

```text
read-only SELECT plan inspection
```

cuando pueda demostrarse segura.

---

# 136. Stored query problem

El Debug Information no deberá conservar objetos ejecutables.

Por tanto el sistema necesitará una:

```text
DiagnosticQueryReference
```

segura si se pretende hacer EXPLAIN posterior.

---

# 137. DiagnosticQueryReference

No deberá ser SQL arbitrario enviado por navegador.

---

# 138. Server-side resolution

La toolbar enviará:

```text
DebugRecordId
```

y el backend resolverá una referencia diagnóstica autorizada.

---

# 139. No client SQL execution

Prohibido:

```text
POST /debug/explain
sql="..."
```

como diseño general.

---

# 140. Explain plan model

```php
final readonly class ToolbarExplainPlan
{
    public function __construct(
        public DebugRecordId $query,
        public ExplainPlanMode $mode,
        public array $nodes,
        public DebugCoverage $coverage,
    ) {}
}
```

---

# 141. Explain plan ≠ raw vendor output

Debe normalizarse cuando sea posible.

---

# 142. Raw vendor details

Podrán existir en una sección avanzada bajo policy.

---

# 143. Explain plan security

Debe pasar por redaction.

---

# 144. On-demand diagnostic actions

Además de EXPLAIN, podrían existir:

```text
Copy Query
Open Source
View Fingerprint Group
View N+1 Group
Export Safe Debug Report
```

---

# 145. Read-only default

La toolbar Database será read-only por default.

---

# 146. Forbidden default actions

No:

```text
Run SQL
Edit Entity
Delete Row
Commit Transaction
Rollback Transaction
Clear Cache
Kill Connection
Run Migration
```

---

# 147. Future admin tools

Si se desarrollan, deberán ser:

```text
separate subsystem
separate permissions
separate security model
```

---

# 148. Debug Toolbar ≠ Database Admin Console

Regla fundamental.

---

# 149. Security architecture

La toolbar deberá aplicar múltiples capas:

```text
Environment Policy
       ↓
Authentication
       ↓
Authorization
       ↓
Session Validation
       ↓
Debug Access Policy
       ↓
Redaction
       ↓
Payload Minimization
       ↓
UI
```

---

# 150. Development environment

Puede habilitarse por default en:

```text
local development
```

pero deberá existir una forma clara de deshabilitarla.

---

# 151. Production

Por default:

```text
disabled
```

para acceso interactivo.

---

# 152. Production diagnostics

Telemetry puede seguir activa.

Esto no implica que la toolbar deba estar disponible.

---

# 153. Production override

Si una organización decide habilitarla:

```text
explicit enable
+
strong authentication
+
authorization
+
network policy
+
redaction
+
short TTL
```

---

# 154. Never environment-only security

No confiar únicamente en:

```php
APP_ENV === 'local'
```

como control de acceso.

---

# 155. Toolbar access policy

```php
interface DebugToolbarAccessPolicy
{
    public function authorize(
        DebugToolbarAccessRequest $request
    ): DebugToolbarAccessDecision;
}
```

---

# 156. Access decision

```php
enum DebugToolbarAccessDecision
{
    case ALLOW;
    case DENY;
    case ALLOW_REDACTED;
}
```

---

# 157. Parameter security

La toolbar no deberá recibir valores que el Debug Information System ya haya clasificado como secretos.

---

# 158. Double redaction

La integración podrá aplicar redaction adicional.

---

# 159. Client-side redaction insufficient

Nunca:

```text
send secret
↓
hide with CSS
```

---

# 160. Server-side minimization

El secreto no deberá llegar al navegador.

---

# 161. HTML escaping

Todo texto dinámico deberá escaparse correctamente.

Especialmente:

```text
SQL
exception messages
entity names
table names
source metadata
```

---

# 162. XSS

SQL puede contener literales controlados por usuario.

Por tanto deberá tratarse como:

```text
untrusted display data
```

---

# 163. CSP integration

La toolbar oficial deberá poder operar bajo una política CSP estricta.

---

# 164. No inline script requirement

Preferir assets empaquetados y nonces/hashes cuando corresponda.

---

# 165. CSRF

Acciones on-demand que realizan requests deberán respetar el modelo CSRF del framework.

---

# 166. Read actions and CSRF

Aunque sean diagnósticas, no se asumirá automáticamente que carecen de impacto.

Ejemplo:

```text
EXPLAIN ANALYZE
```

puede ejecutar trabajo costoso.

---

# 167. Rate limiting

Endpoints de debug podrán tener límites.

---

# 168. Explain rate limiting

Especialmente:

```text
EXPLAIN
profile retrieval
large export
```

---

# 169. Resource governance

La toolbar no deberá convertir una request lenta en una request mucho más costosa.

---

# 170. ToolbarBudget

```php
final readonly class DebugToolbarBudget
{
    public function __construct(
        public int $maxPayloadBytes,
        public int $maxQueryRecords,
        public int $maxTimelineEntries,
        public int $maxDiagnostics,
        public Duration $maxBuildDuration,
    ) {}
}
```

---

# 171. Debug Information budget vs Toolbar budget

Son distintos.

```text
Telemetry Budget
      ↓
Debug Information Budget
      ↓
Toolbar Payload Budget
```

---

# 172. Toolbar can reduce

Puede presentar menos información que el snapshot.

---

# 173. Toolbar cannot recover

No puede inventar información descartada anteriormente.

---

# 174. Query list virtualization

El frontend podrá usar:

```text
virtual scrolling
```

para miles de registros.

---

# 175. Virtualization ≠ backend semantics

Es optimización de UI.

---

# 176. JSON payload

Para sesiones grandes:

```text
JSON
```

deberá evitar duplicar strings repetidas innecesariamente si el diseño de transporte lo permite.

---

# 177. Fingerprint dictionary

Una optimización futura podría representar fingerprints repetidos mediante referencias.

---

# 178. Compression

HTTP compression puede utilizarse en la capa web.

No pertenece al Database core.

---

# 179. Debug payload cache

Un snapshot finalizado podrá ser cacheado temporalmente para múltiples requests del toolbar.

---

# 180. Debug payload cache ≠ Database Result Cache

No mezclar ambos subsistemas.

---

# 181. Debug session TTL

Ejemplo conceptual:

```text
5–30 minutes development
```

configurable.

No deberá asumirse como una constante arquitectónica.

---

# 182. Session expiration

La UI deberá manejar:

```text
DEBUG_SESSION_EXPIRED
```

de forma explícita.

---

# 183. Debug store unavailable

Debe degradar limpiamente:

```text
Debug information unavailable
```

sin romper la aplicación observada.

---

# 184. Toolbar failure isolation

> **Una falla en la Developer Debug Toolbar no deberá convertirse en una falla de la aplicación observada.**

---

# 185. Exception boundary

Errores del adapter/UI deberán quedar aislados del Database execution path.

---

# 186. Toolbar self-observation

Si la toolbar consulta un debug store usando Database:

```text
Toolbar Request
↓
Database query
↓
Telemetry
↓
Toolbar debug
```

puede producir ruido/recursión.

---

# 187. Internal request marker

Las requests de toolbar podrán marcarse como:

```text
DebugInternalRequest
```

para aplicar una policy especializada.

---

# 188. Do not globally disable telemetry

Solo excluir el ruido estrictamente necesario.

---

# 189. Debug toolbar own database queries

Si ocurren, deberán poder:

```text
exclude
tag
separate
```

de las queries de la request observada.

---

# 190. Persistent runtimes

Especial atención a:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 191. Request isolation

Cada request tendrá:

```text
DebugSessionId
ToolbarContext
AccessContext
```

independientes.

---

# 192. No static session

Prohibido:

```php
static $currentDebugSession;
```

---

# 193. FrankenPHP lifecycle

```text
Worker
│
├── Request A
│   ├── DatabaseDebugInformation A
│   └── Toolbar Reference A
│
├── reset
│
└── Request B
    ├── DatabaseDebugInformation B
    └── Toolbar Reference B
```

---

# 194. RoadRunner lifecycle

Misma separación por job/request.

---

# 195. OpenSwoole lifecycle

La integración deberá ser:

```text
coroutine-safe
```

---

# 196. Concurrent requests

Nunca mezclar:

```text
Query A1
Query B1
Query A2
```

dentro de una misma sesión por error de estado global.

---

# 197. Toolbar assets

La UI oficial pertenecerá al Developer Tools subsystem.

Database solo proporcionará:

```text
data
contracts
adapters
```

---

# 198. Database package independence

Será posible instalar:

```text
VoltStack Database
```

sin instalar la toolbar visual completa.

---

# 199. Optional integration

La toolbar será una integración opcional.

---

# 200. Suggested package boundary

Conceptualmente:

```text
voltstack/database
voltstack/developer-tools
```

o módulos equivalentes internos.

---

# 201. Database side

Proporciona:

```text
DatabaseDebugInformation
DatabaseDebugToolbarAdapter
Toolbar payload contracts
```

---

# 202. Developer Tools side

Proporciona:

```text
HTML
CSS
JavaScript
HTTP routes
toolbar shell
IDE integrations
UI rendering
```

---

# 203. No frontend dependency in Database core

`VoltStack/Quantum/Database` no deberá depender de:

```text
React
Vue
Livewire
VoltStack Component Runtime
browser APIs
```

---

# 204. Official VoltStack reactive UI

La toolbar oficial sí podrá utilizar el runtime reactivo propio de VoltStack desde el paquete Developer Tools.

---

# 205. Progressive rendering

Paneles grandes podrán cargarse al abrirse.

---

# 206. Initial toolbar payload

Idealmente pequeño:

```text
session
query count
DB duration
warnings
slow count
N+1 count
```

---

# 207. Detailed payload

Solo cuando el desarrollador abre Database.

---

# 208. Query detail

Solo cuando abre una query.

---

# 209. Benefit

Reduce:

```text
response size
serialization cost
browser memory
```

---

# 210. Source map

La toolbar podrá mapear:

```text
QueryOrigin
```

a:

```text
Controller
Repository
Model
Relationship
Migration
Seeder
CLI
Job
```

si la telemetry dispone de esa evidencia.

---

# 211. No guessed origin

UNKNOWN deberá permanecer UNKNOWN.

---

# 212. Query origin badges

Ejemplos:

```text
ORM
Repository
Relationship
Raw
Migration
```

---

# 213. Raw query indicator

Una query originada por escape hatch podrá mostrarse como:

```text
RAW
```

---

# 214. Security warning

`RAW` no implica automáticamente vulnerabilidad.

---

# 215. Parameter binding indicator

Podrá mostrar:

```text
Prepared: yes
Bindings: 3
```

---

# 216. SQL injection UI

No deberá afirmar:

```text
SQL injection safe
```

basándose únicamente en que existen placeholders.

Ese análisis pertenece al Security System.

---

# 217. Query compilation panel

Modo avanzado podría mostrar:

```text
Query Model
AST
Logical Plan
Physical Plan
Compiled SQL
```

si esas representaciones fueron capturadas explícitamente.

---

# 218. Default

No capturar AST completo de todas las queries por default.

---

# 219. AST size

Puede ser considerable.

---

# 220. AST visualization

Deberá consumir una representación debug-safe.

Nunca el AST mutable original.

---

# 221. Query pipeline view

Ejemplo:

```text
Query Builder
     ↓
Query Model
     ↓
Semantic Analysis
     ↓
Optimizer
     ↓
Planner
     ↓
Compiler
     ↓
Executor
```

con tiempos cuando estén disponibles.

---

# 222. Optimization diagnostics

Puede mostrar:

```text
rules applied
rules skipped
```

si Query Profiler lo captura.

---

# 223. Compiler details

Podrá mostrar:

```text
Dialect: PostgreSQL
Compiler: PostgreSQLSqlCompiler
```

en development autorizado.

---

# 224. Capability information

Podrá mostrar:

```text
RETURNING: supported
Window functions: supported
```

cuando sea relevante al diagnostic.

---

# 225. Version ≠ capability

La UI no deberá inferir capacidades solo desde versión del servidor.

---

# 226. Platform panel

Opcionalmente:

```text
Logical platform
Dialect
Capabilities generation
```

---

# 227. Schema/migration panel

La Database Toolbar puede integrar resumen:

```text
Schema generation
Pending migration diagnostics
Migration state
```

pero no ejecutar migrations.

---

# 228. Run migration button

No pertenece a este panel por default.

---

# 229. CLI link

Podría ofrecer:

```text
Suggested command:
php voltstack migrate:status
```

como texto/copy action.

---

# 230. Toolbar action model

Acciones deberán clasificarse:

```php
enum DebugToolbarActionRisk
{
    case LOCAL_ONLY;
    case READ_ONLY_REMOTE;
    case DATABASE_WORK;
    case MUTATING;
}
```

---

# 231. LOCAL_ONLY

Ejemplos:

```text
filter
sort
copy safe SQL
expand record
```

---

# 232. READ_ONLY_REMOTE

Ejemplos:

```text
fetch panel payload
fetch source metadata
```

---

# 233. DATABASE_WORK

Ejemplo:

```text
EXPLAIN
```

aunque no sea mutating.

---

# 234. MUTATING

No deberá formar parte del Database Debug Toolbar V1.

---

# 235. Explicit confirmation

Acciones `DATABASE_WORK` podrán requerir confirmación según policy.

---

# 236. Explain budget

Podrá limitar:

```text
max plan time
max concurrent explains
allowed query types
```

---

# 237. Toolbar diagnostics export

La UI podrá permitir:

```text
Export Debug Report
```

---

# 238. Export modes

```text
SHARE_SAFE
DEVELOPER
SUPPORT
```

según el documento 224.

---

# 239. Default export

```text
SHARE_SAFE
```

---

# 240. Export preview

La UI debería mostrar qué categorías serán eliminadas.

---

# 241. Export format

Inicialmente:

```text
JSON
```

y posteriormente:

```text
HTML report
```

podría agregarse.

---

# 242. No raw runtime serialization

La exportación siempre utiliza el snapshot.

---

# 243. Toolbar extension model

Otros módulos podrán añadir paneles.

---

# 244. DatabaseToolbarExtension

```php
interface DatabaseToolbarExtension
{
    public function id(): ToolbarExtensionId;

    public function build(
        DatabaseDebugInformation $information,
        DebugToolbarContext $context,
    ): ?ToolbarExtensionPayload;
}
```

---

# 245. Extension examples

```text
PostGIS
Custom Database Driver
Tenant Database
Search Engine Integration
Custom Query Extension
```

---

# 246. Extension isolation

Una extensión no deberá recibir automáticamente acceso a Connection.

---

# 247. Extension security

Debe respetar:

```text
access policy
redaction
budget
payload schema
```

---

# 248. Extension failure

No deberá romper el panel Database completo.

---

# 249. Extension timeout

Debe ser bounded.

---

# 250. Toolbar registry

```php
interface DatabaseToolbarExtensionRegistry
{
    public function register(
        DatabaseToolbarExtension $extension
    ): void;
}
```

---

# 251. Registry freeze

En persistent runtimes:

```text
register during bootstrap
freeze before request processing
```

---

# 252. Dynamic per-request mutation

No permitida por default.

---

# 253. Configuration

Ejemplo conceptual:

```php
'database' => [
    'debug_toolbar' => [
        'enabled' => env('APP_DEBUG', false),

        'queries' => true,
        'timeline' => true,
        'connections' => true,
        'transactions' => true,
        'orm' => true,
        'cache' => true,
        'diagnostics' => true,

        'sql' => 'normalized_redacted',
        'parameters' => 'types',

        'explain' => [
            'enabled' => true,
            'analyze' => false,
        ],

        'limits' => [
            'queries' => 500,
            'timeline_entries' => 1000,
        ],
    ],
];
```

---

# 254. Configuration validation

Configuraciones inseguras deberán producir warnings o errores según entorno.

---

# 255. Example

```text
production
+
full parameter values
+
public toolbar
```

deberá considerarse una configuración crítica/insegura.

---

# 256. Secure defaults

V1:

```text
production toolbar: disabled
parameters: types only
SQL: normalized/redacted
EXPLAIN ANALYZE: disabled
mutating actions: unavailable
session TTL: bounded
payload: bounded
```

---

# 257. Toolbar service provider

Conceptualmente:

```php
final class DatabaseDebugToolbarServiceProvider
{
    public function register(): void
    {
        // contracts/adapters
    }

    public function boot(): void
    {
        // freeze integration configuration
    }
}
```

---

# 258. Boot ≠ request state

El provider no almacenará la sesión actual.

---

# 259. Request lifecycle integration

```text
Request begins
↓
Debug session created
↓
Database operations
↓
Debug information finalized
↓
Snapshot stored
↓
Debug reference attached to response
↓
Request state reset
```

---

# 260. Response integration

La referencia podrá añadirse mediante el framework HTTP.

Database no deberá modificar directamente HTTP responses.

---

# 261. Integration event

Podría emitirse:

```text
DatabaseDebugInformationFinalized
```

para que Developer Tools recoja el snapshot.

---

# 262. Event payload

Debe contener snapshot/reference segura.

No runtime objects.

---

# 263. Event failure

Toolbar listener failure no debe invalidar la operación DB ya completada.

---

# 264. Streaming HTTP responses

El snapshot puede finalizar al final real del request scope.

No asumir que:

```text
response headers sent
=
database work finished
```

en todos los modelos de runtime.

---

# 265. Long-lived requests

WebSocket/SSE requerirán policies específicas.

---

# 266. Debug session segmentation

Podría segmentarse por:

```text
operation
message
frame
```

en runtimes long-lived.

---

# 267. Not V1 default

La V1 puede limitar la toolbar a request/job scopes finitos.

---

# 268. CLI integration

Aunque el documento trate Toolbar, el adapter deberá conservar compatibilidad con:

```text
CLI debug viewer
```

mediante `DatabaseDebugInformation`.

---

# 269. Job integration

Un queue job podrá producir un debug snapshot sin browser toolbar.

---

# 270. Toolbar replay

Una sesión almacenada podrá abrirse posteriormente mientras no expire.

---

# 271. Replay ≠ runtime replay

Solo se reproduce la información diagnóstica.

No se reejecuta la request.

---

# 272. Offline debug report

Una exportación share-safe podría abrirse sin conexión a Database.

---

# 273. Important property

Esto demuestra que:

```text
Debug Toolbar
```

no necesita live Database access para inspección normal.

---

# 274. Testing

Deberán existir pruebas para:

```text
payload mapping
redaction
access control
truncation
session isolation
persistent runtimes
query grouping
timeline
diagnostic links
extension failures
EXPLAIN authorization
```

---

# 275. Adapter unit test

```php
$payload = $adapter->build(
    $debugInformation,
    $context,
);

expect($payload->summary->queryCount)
    ->toBe(5);
```

---

# 276. Redaction test

```php
expect($payload->toArray())
    ->not->toContain('secret-password');
```

---

# 277. Cross-request isolation test

```text
Request A query
```

nunca aparecerá en:

```text
Request B toolbar payload
```

---

# 278. Cross-tenant isolation test

Mismo principio.

---

# 279. Truncation test

Con:

```text
10,000 queries
```

y budget:

```text
500
```

deberá indicar:

```text
total: 10,000
shown: 500
dropped: 9,500
```

---

# 280. UNKNOWN test

Transaction UNKNOWN deberá renderizarse literalmente como:

```text
UNKNOWN
```

---

# 281. Query count concurrency test

Aggregate DB duration no deberá presentarse como request duration cuando existe overlap.

---

# 282. Explain test

Verificar:

```text
no automatic EXPLAIN
```

al abrir panel.

---

# 283. Explain authorization test

Usuario sin permiso:

```text
DENIED
```

---

# 284. Explain mutation test

Mutating query deberá rechazarse según default policy.

---

# 285. Persistent runtime test

Ejecutar múltiples scopes secuenciales en el mismo worker y verificar ausencia de leakage.

---

# 286. OpenSwoole concurrency test

Dos coroutines deberán mantener snapshots independientes.

---

# 287. Performance tests

Medir:

```text
toolbar disabled overhead
snapshot adaptation time
serialization time
payload size
browser-oriented record count
```

---

# 288. Performance target philosophy

No fijar prematuramente números universales.

Definir budgets configurables y benchmarks reproducibles.

---

# 289. Debug toolbar disabled path

Debe aproximarse a:

```text
no-op integration
```

---

# 290. Toolbar enabled path

El costo deberá ser:

```text
observable
bounded
configurable
```

---

# 291. Diagnostics

La propia integración podrá generar diagnostics como:

```text
DB_TOOLBAR_PAYLOAD_TRUNCATED
DB_TOOLBAR_STORE_UNAVAILABLE
DB_TOOLBAR_EXTENSION_FAILED
DB_TOOLBAR_EXPLAIN_DENIED
```

---

# 292. Toolbar diagnostics separation

Estos diagnostics describen la herramienta.

No necesariamente la operación Database observada.

---

# 293. Namespace separation

Podrán usar:

```text
DB_TOOLBAR_*
```

para evitar confusión.

---

# 294. Logging

Fallos internos importantes podrán enviarse al Logging System.

---

# 295. Avoid recursion

El logging de un error de toolbar no deberá iniciar un nuevo ciclo de toolbar/database telemetry.

---

# 296. Accessibility

La toolbar oficial deberá considerar:

```text
keyboard navigation
semantic HTML
screen reader labels
non-color-only severity
contrast
```

aunque la implementación visual viva fuera de Database.

---

# 297. Mobile

No será prioridad primaria, pero el payload no deberá asumir desktop.

---

# 298. Developer UX

La UI deberá optimizar el flujo:

```text
problem
↓
evidence
↓
source
↓
recommendation
```

---

# 299. Example N+1 flow

```text
DB badge shows warning
↓
open Database
↓
N+1 panel
↓
User.posts
↓
100 dependent queries
↓
open source
↓
recommend batch/eager loading
```

---

# 300. Example slow query flow

```text
Slow Query
↓
open profile
↓
execution 30 ms
hydration 600 ms
↓
problem is hydration
↓
inspect projection
```

---

# 301. Example connection flow

```text
Request slow
↓
Database panel
↓
DB execution normal
Connection wait high
↓
Pool panel
↓
saturation diagnostic
```

---

# 302. Example transaction flow

```text
Transaction warning
↓
Attempt 1 deadlock
↓
Retry
↓
Attempt 2 committed
```

---

# 303. Example UNKNOWN flow

```text
Commit sent
↓
Connection lost
↓
Outcome UNKNOWN
↓
Toolbar displays UNKNOWN
↓
No retry recommendation unless evidence permits
```

---

# 304. UI must preserve semantics

Nunca simplificar:

```text
UNKNOWN → FAILED
PARTIAL → SUCCESS
STALE_REJECTED → MISS
logical nested tx → physical tx
replica lag unknown → zero
```

---

# 305. Database panel navigation

Estructura sugerida:

```text
Database
│
├── Overview
├── Queries
├── Timeline
├── Connections
├── Transactions
├── ORM
├── Cache
├── Slow Queries
├── N+1
├── Distribution
└── Diagnostics
```

---

# 306. Overview

Debe responder rápidamente:

```text
¿Qué pasó?
¿Dónde se gastó el tiempo?
¿Hay errores?
¿Hay N+1?
¿Hay slow queries?
¿Hay problemas de conexión?
```

---

# 307. Queries

Debe responder:

```text
¿Qué queries se ejecutaron?
¿Cuánto tardaron?
¿Por qué se ejecutaron?
¿Desde dónde?
```

---

# 308. Timeline

Debe responder:

```text
¿Cuándo ocurrió cada operación?
¿Qué se solapó?
```

---

# 309. Connections

Debe responder:

```text
¿Dónde se ejecutó?
¿Cuánto tardó adquirir conexión?
```

---

# 310. Transactions

Debe responder:

```text
¿Qué transacciones ocurrieron?
¿Cuál fue su outcome?
```

---

# 311. ORM

Debe responder:

```text
¿Qué trabajo realizó el ORM?
```

---

# 312. Cache

Debe responder:

```text
¿Qué fue reutilizado?
¿Qué fue rechazado?
```

---

# 313. Diagnostics

Debe responder:

```text
¿Qué debería investigar?
```

---

# 314. Toolbar API stability

Los contracts PHP internos podrán evolucionar bajo las políticas generales de versionado de VoltStack.

El payload público deberá tener versión explícita.

---

# 315. Internal DTOs

No deberán convertirse accidentalmente en API pública estable.

---

# 316. Public transport DTOs

Deberán documentarse y versionarse.

---

# 317. JSON response example

```json
{
  "version": "1.0",
  "session": "dbg_xxx",
  "coverage": "COMPLETE",
  "summary": {
    "queries": 18,
    "database_time_ns": 42700000,
    "slow_queries": 2,
    "n_plus_one": 1,
    "warnings": 3
  },
  "links": {
    "queries": true,
    "timeline": true,
    "orm": true
  }
}
```

---

# 318. No infrastructure leakage

No incluir automáticamente:

```text
absolute server paths
private IPs
credentials
environment secrets
cloud metadata
```

---

# 319. Toolbar source links

Deben utilizar rutas relativas y mapping local.

---

# 320. Remote development

Un IDE adapter podrá mapear:

```text
server project root
→
local project root
```

sin modificar DatabaseDebugInformation.

---

# 321. Container development

Mismo principio para:

```text
Docker
Dev Containers
remote servers
```

---

# 322. SourcePathMapper

```php
interface SourcePathMapper
{
    public function map(
        DebugSourceLocation $source
    ): ?MappedSourceLocation;
}
```

---

# 323. Mapper configuration

Pertenece a Developer Tools.

---

# 324. Query fingerprint links

Todos los records con el mismo fingerprint podrán navegar entre sí.

---

# 325. Transaction links

Query → Transaction.

---

# 326. Connection links

Query → Connection.

---

# 327. ORM links

Query → ORM operation.

---

# 328. N+1 links

Query → N+1 detection.

---

# 329. Slow links

Query → Slow Query profile.

---

# 330. Correlation graph

La toolbar deberá explotar:

```text
DebugCorrelationRegistry
```

del documento 224.

---

# 331. No UI-created semantic correlation

La UI no deberá decidir que dos operaciones están relacionadas solo por parecido textual.

---

# 332. Correlation visualization

Modo avanzado:

```text
HTTP Request
     │
     ▼
Repository::findUsers
     │
     ▼
Query #12
     │
     ├── Connection replica-1
     │
     ├── Slow Query Diagnostic
     │
     └── Hydration #4
```

---

# 333. Export correlation graph

Puede incluirse en reportes de debug.

---

# 334. Developer toolbar and telemetry sampling

Si telemetry fue sampled:

```text
coverage=PARTIAL
```

deberá mostrarse.

---

# 335. No false totals

Una muestra no deberá mostrarse como total exacto.

---

# 336. Toolbar and disabled subsystems

Si ORM Telemetry está deshabilitado:

```text
ORM telemetry disabled
```

no:

```text
ORM operations: 0
```

---

# 337. Feature availability model

```php
enum ToolbarFeatureAvailability
{
    case AVAILABLE;
    case DISABLED;
    case UNSUPPORTED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 338. Feature capability

Cada panel podrá exponer availability.

---

# 339. Unsupported ≠ disabled

Ejemplo:

```text
EXPLAIN unsupported
```

es distinto de:

```text
EXPLAIN disabled by policy
```

---

# 340. Toolbar initialization

El frontend recibirá un pequeño manifest:

```text
session
available panels
severity
summary
capabilities
```

---

# 341. Toolbar manifest

```php
final readonly class DatabaseToolbarManifest
{
    public function __construct(
        public DebugSessionId $sessionId,
        public array $panels,
        public ToolbarDatabaseSummary $summary,
        public ToolbarCapabilitySet $capabilities,
    ) {}
}
```

---

# 342. ToolbarCapabilitySet

Ejemplo:

```text
query_detail
timeline
source_links
explain
safe_export
```

---

# 343. Capability ≠ permission

Puede existir:

```text
EXPLAIN capability supported
```

pero:

```text
user not authorized
```

---

# 344. Effective toolbar capability

Formalmente:

```text
EffectiveCapability
=
SystemCapability
∩
Configuration
∩
Policy
∩
Authorization
∩
SessionAvailability
```

---

# 345. Developer toolbar integration lifecycle

```text
BOOTSTRAP
   ↓
REGISTER ADAPTERS
   ↓
FREEZE REGISTRIES
   ↓
REQUEST START
   ↓
DEBUG SESSION
   ↓
DATABASE OPERATIONS
   ↓
DEBUG FINALIZATION
   ↓
TOOLBAR PROJECTION
   ↓
STORE / MANIFEST
   ↓
REQUEST RESET
```

---

# 346. State ownership

```text
Immutable adapters        → shared
Compiled policies         → shared
Extension registry        → shared/frozen

Debug session             → scoped
Toolbar context           → scoped
Payload builder state     → scoped
Session access            → scoped
```

---

# 347. Recommended directory structure

```text
src/Quantum/Database/Telemetry/Debug/Toolbar/
│
├── Contract/
│   ├── DatabaseDebugToolbarAdapter.php
│   ├── DatabaseDebugToolbarProvider.php
│   ├── DebugToolbarAccessPolicy.php
│   ├── DebugSqlFormatter.php
│   ├── IDELinkGenerator.php
│   ├── SourcePathMapper.php
│   └── DatabaseToolbarExtensionRegistry.php
│
├── Context/
│   ├── DebugToolbarContext.php
│   ├── DebugToolbarAccess.php
│   ├── DebugToolbarAccessRequest.php
│   └── DebugToolbarPreferences.php
│
├── Model/
│   ├── DatabaseDebugToolbarPayload.php
│   ├── DatabaseToolbarManifest.php
│   ├── ToolbarMetadata.php
│   ├── ToolbarPayloadVersion.php
│   ├── ToolbarFeatureAvailability.php
│   └── ToolbarCapabilitySet.php
│
├── Summary/
│   ├── ToolbarDatabaseSummary.php
│   └── ToolbarSummaryBuilder.php
│
├── Query/
│   ├── ToolbarQueryRecord.php
│   ├── ToolbarQueryCollection.php
│   ├── ToolbarQueryStatus.php
│   ├── ToolbarQueryFlags.php
│   ├── ToolbarQueryDetail.php
│   ├── ToolbarSqlCopyMode.php
│   ├── FormattedDebugSql.php
│   └── DefaultDebugSqlFormatter.php
│
├── Timeline/
│   ├── ToolbarTimeline.php
│   ├── ToolbarTimelineLane.php
│   └── ToolbarTimelineEntry.php
│
├── Connection/
│   ├── ToolbarConnectionRecord.php
│   └── ToolbarConnectionCollection.php
│
├── Transaction/
│   ├── ToolbarTransactionRecord.php
│   └── ToolbarTransactionCollection.php
│
├── ORM/
│   ├── ToolbarORMView.php
│   ├── ToolbarUnitOfWorkView.php
│   ├── ToolbarIdentityMapView.php
│   ├── ToolbarHydrationView.php
│   └── ToolbarRelationshipView.php
│
├── Cache/
│   └── ToolbarCacheView.php
│
├── Diagnostic/
│   ├── ToolbarDiagnostic.php
│   └── ToolbarDiagnosticCollection.php
│
├── Distribution/
│   ├── ToolbarDistributionView.php
│   ├── ToolbarReplicaView.php
│   └── ToolbarShardView.php
│
├── Source/
│   ├── ToolbarSourceLink.php
│   ├── MappedSourceLocation.php
│   └── DefaultSourcePathMapper.php
│
├── Explain/
│   ├── DiagnosticQueryReference.php
│   ├── ToolbarExplainPlan.php
│   ├── ExplainPlanMode.php
│   ├── DebugExplainPolicy.php
│   └── DebugExplainService.php
│
├── Action/
│   ├── DebugToolbarActionRisk.php
│   ├── DebugToolbarAction.php
│   └── DebugToolbarActionPolicy.php
│
├── Security/
│   ├── DefaultDebugToolbarAccessPolicy.php
│   └── ToolbarPayloadRedactor.php
│
├── Policy/
│   ├── DebugToolbarMode.php
│   ├── DebugToolbarBudget.php
│   └── CompiledDebugToolbarPolicy.php
│
├── Adapter/
│   └── DefaultDatabaseDebugToolbarAdapter.php
│
├── Provider/
│   └── DefaultDatabaseDebugToolbarProvider.php
│
├── Extension/
│   ├── DatabaseToolbarExtension.php
│   ├── ToolbarExtensionId.php
│   ├── ToolbarExtensionPayload.php
│   └── DefaultDatabaseToolbarExtensionRegistry.php
│
├── Testing/
│   ├── FakeDatabaseDebugToolbarAdapter.php
│   ├── RecordingToolbarProvider.php
│   └── ToolbarAssertions.php
│
└── Exception/
    ├── DebugToolbarException.php
    ├── DebugToolbarAccessException.php
    ├── DebugToolbarPayloadException.php
    ├── DebugToolbarSessionExpiredException.php
    ├── DebugToolbarExplainException.php
    └── DebugToolbarExtensionException.php
```

---

# 348. Developer Tools package

La implementación visual podría organizarse separadamente:

```text
src/Platform/DeveloperTools/
└── Toolbar/
    ├── Database/
    │   ├── Panel
    │   ├── Components
    │   ├── Routes
    │   ├── Controllers
    │   └── Assets
    └── ...
```

La ubicación final dependerá de la arquitectura general de Developer Tools.

---

# 349. Invariantes arquitectónicas

## DB-TOOLBAR-001
La toolbar será una vista sobre `DatabaseDebugInformation`.

## DB-TOOLBAR-002
La toolbar no accederá directamente a Connection.

## DB-TOOLBAR-003
La toolbar no accederá directamente a EntityManager.

## DB-TOOLBAR-004
La toolbar no accederá directamente a UnitOfWork.

## DB-TOOLBAR-005
La toolbar no accederá directamente a IdentityMap.

## DB-TOOLBAR-006
La toolbar no accederá directamente a ResultCursor.

## DB-TOOLBAR-007
La toolbar no accederá directamente a PDO.

## DB-TOOLBAR-008
La toolbar no accederá directamente a TransactionContext.

## DB-TOOLBAR-009
La toolbar será distinta del Debug Information System.

## DB-TOOLBAR-010
La toolbar será distinta del Database Admin Console.

## DB-TOOLBAR-011
La toolbar será read-only por default.

## DB-TOOLBAR-012
La toolbar no permitirá SQL arbitrario por default.

## DB-TOOLBAR-013
La toolbar no permitirá mutaciones DB en V1.

## DB-TOOLBAR-014
La toolbar no ejecutará migrations.

## DB-TOOLBAR-015
La toolbar no hará rollback de transacciones.

## DB-TOOLBAR-016
La toolbar no modificará entidades.

## DB-TOOLBAR-017
La toolbar no limpiará cache.

## DB-TOOLBAR-018
La toolbar no cerrará conexiones.

## DB-TOOLBAR-019
La toolbar no matará queries por default.

## DB-TOOLBAR-020
Query Explorer operará sobre snapshots.

## DB-TOOLBAR-021
Filtering no ejecutará nuevas queries de negocio.

## DB-TOOLBAR-022
Sorting no ejecutará nuevas queries de negocio.

## DB-TOOLBAR-023
Fingerprint repetition no implicará N+1.

## DB-TOOLBAR-024
N+1 será mostrado solo desde detección canónica.

## DB-TOOLBAR-025
Slow Query classification será consumida del profiler/detector.

## DB-TOOLBAR-026
UNKNOWN permanecerá UNKNOWN.

## DB-TOOLBAR-027
PARTIAL permanecerá PARTIAL.

## DB-TOOLBAR-028
Replica lag UNKNOWN no será cero.

## DB-TOOLBAR-029
Aggregate query time no será presentado necesariamente como wall time.

## DB-TOOLBAR-030
Timeline soportará operaciones concurrentes.

## DB-TOOLBAR-031
Timeline utilizará IDs para correlación.

## DB-TOOLBAR-032
SQL será mostrado según policy.

## DB-TOOLBAR-033
Parameters serán mostrados según policy.

## DB-TOOLBAR-034
Secret values nunca llegarán al browser.

## DB-TOOLBAR-035
Client-side hiding no contará como redaction.

## DB-TOOLBAR-036
DSN completo nunca será mostrado.

## DB-TOOLBAR-037
Credentials nunca serán mostradas.

## DB-TOOLBAR-038
Source paths serán redactables.

## DB-TOOLBAR-039
SQL será tratado como untrusted display data.

## DB-TOOLBAR-040
Dynamic output será escapado.

## DB-TOOLBAR-041
Toolbar access no dependerá solo del environment.

## DB-TOOLBAR-042
DebugSessionId no será authorization token.

## DB-TOOLBAR-043
Producción tendrá toolbar interactiva deshabilitada por default.

## DB-TOOLBAR-044
Telemetry podrá permanecer activa con toolbar deshabilitada.

## DB-TOOLBAR-045
Production override requerirá configuración explícita.

## DB-TOOLBAR-046
Toolbar payload será bounded.

## DB-TOOLBAR-047
Query list será bounded.

## DB-TOOLBAR-048
Timeline será bounded.

## DB-TOOLBAR-049
Diagnostics serán bounded.

## DB-TOOLBAR-050
Truncation será visible.

## DB-TOOLBAR-051
Partial telemetry será visible.

## DB-TOOLBAR-052
Disabled telemetry no será representada como cero.

## DB-TOOLBAR-053
Toolbar payload tendrá versión.

## DB-TOOLBAR-054
Toolbar payload version será distinta de Debug Information version.

## DB-TOOLBAR-055
Initial manifest será pequeño.

## DB-TOOLBAR-056
Detailed panels podrán cargarse on-demand.

## DB-TOOLBAR-057
On-demand UI loading no será Database Lazy Collection.

## DB-TOOLBAR-058
EXPLAIN nunca se ejecutará automáticamente.

## DB-TOOLBAR-059
EXPLAIN requerirá autorización.

## DB-TOOLBAR-060
EXPLAIN utilizará Platform Capabilities.

## DB-TOOLBAR-061
EXPLAIN ANALYZE será considerado potencialmente ejecutable/costoso.

## DB-TOOLBAR-062
EXPLAIN ANALYZE estará deshabilitado por default.

## DB-TOOLBAR-063
Mutating explain operations serán restringidas.

## DB-TOOLBAR-064
El browser no enviará SQL arbitrario para ejecución diagnóstica.

## DB-TOOLBAR-065
Diagnostic query references serán server-resolved.

## DB-TOOLBAR-066
Explain plans serán snapshots seguros.

## DB-TOOLBAR-067
Raw vendor plans estarán sujetos a redaction.

## DB-TOOLBAR-068
Debug actions tendrán clasificación de riesgo.

## DB-TOOLBAR-069
Mutating actions no estarán disponibles en V1.

## DB-TOOLBAR-070
Database-work actions estarán sujetas a resource governance.

## DB-TOOLBAR-071
Toolbar failure no romperá la aplicación observada.

## DB-TOOLBAR-072
Debug store failure no romperá la aplicación observada.

## DB-TOOLBAR-073
Extension failure no romperá el panel completo.

## DB-TOOLBAR-074
Extensions no recibirán Connection automáticamente.

## DB-TOOLBAR-075
Extensions respetarán security policy.

## DB-TOOLBAR-076
Extensions respetarán budget.

## DB-TOOLBAR-077
Extension registry será frozen antes del request loop.

## DB-TOOLBAR-078
No habrá mutable extension registry per request.

## DB-TOOLBAR-079
Persistent runtime sessions serán aisladas.

## DB-TOOLBAR-080
No habrá static current session.

## DB-TOOLBAR-081
FrankenPHP tendrá sesión por request.

## DB-TOOLBAR-082
RoadRunner tendrá sesión por job/request.

## DB-TOOLBAR-083
OpenSwoole tendrá contexto coroutine-safe.

## DB-TOOLBAR-084
Concurrent requests no compartirán records.

## DB-TOOLBAR-085
Concurrent tenants no compartirán records.

## DB-TOOLBAR-086
Request reset eliminará state temporal.

## DB-TOOLBAR-087
Shared adapters deberán ser stateless.

## DB-TOOLBAR-088
Compiled policies podrán ser shared.

## DB-TOOLBAR-089
Database core no dependerá de React.

## DB-TOOLBAR-090
Database core no dependerá de Vue.

## DB-TOOLBAR-091
Database core no dependerá de browser APIs.

## DB-TOOLBAR-092
Visual UI podrá vivir en Developer Tools.

## DB-TOOLBAR-093
Database será utilizable sin toolbar visual.

## DB-TOOLBAR-094
Toolbar será integración opcional.

## DB-TOOLBAR-095
Source links serán generados mediante adapters.

## DB-TOOLBAR-096
IDE links no serán construidos desde input arbitrario.

## DB-TOOLBAR-097
Absolute paths no serán necesarios en browser.

## DB-TOOLBAR-098
Remote path mapping pertenecerá a Developer Tools.

## DB-TOOLBAR-099
Query origins no serán inferidos sin evidencia.

## DB-TOOLBAR-100
RAW no significará automáticamente unsafe.

## DB-TOOLBAR-101
Prepared statements no significarán automáticamente security completa.

## DB-TOOLBAR-102
AST completo no será capturado por default.

## DB-TOOLBAR-103
AST visualization utilizará snapshot seguro.

## DB-TOOLBAR-104
Compiler details serán development diagnostics.

## DB-TOOLBAR-105
Version no será tratada como capability.

## DB-TOOLBAR-106
Schema panel no ejecutará introspection automáticamente.

## DB-TOOLBAR-107
Migration panel no ejecutará migrations.

## DB-TOOLBAR-108
Cache panel preservará physical-hit vs usable-hit semantics.

## DB-TOOLBAR-109
Connection panel preservará role/routing semantics.

## DB-TOOLBAR-110
Transaction panel distinguirá logical y physical nesting.

## DB-TOOLBAR-111
ORM panel no serializará entidades completas.

## DB-TOOLBAR-112
IdentityMap panel mostrará estadísticas, no object references.

## DB-TOOLBAR-113
UnitOfWork panel no expondrá dirty values por default.

## DB-TOOLBAR-114
Relationship panel preservará partial coverage semantics.

## DB-TOOLBAR-115
Distribution panel no fingirá complete success ante shard failure.

## DB-TOOLBAR-116
Routing reason vendrá del Routing System.

## DB-TOOLBAR-117
Toolbar no inferirá replica freshness.

## DB-TOOLBAR-118
Toolbar no inferirá failover cause sin evidencia.

## DB-TOOLBAR-119
Toolbar no recalculará diagnostics semánticos.

## DB-TOOLBAR-120
Diagnostic intelligence permanecerá backend-side.

## DB-TOOLBAR-121
Diagnostic codes permanecerán estables.

## DB-TOOLBAR-122
Recommendations no serán auto-fixes.

## DB-TOOLBAR-123
Export utilizará snapshot seguro.

## DB-TOOLBAR-124
Share-safe será export mode por default.

## DB-TOOLBAR-125
Exports nunca contendrán credentials.

## DB-TOOLBAR-126
Export mode será distinto de authorization.

## DB-TOOLBAR-127
Debug store tendrá TTL bounded.

## DB-TOOLBAR-128
Expired sessions serán explícitas.

## DB-TOOLBAR-129
Stored debug data será distinta del Result Cache.

## DB-TOOLBAR-130
Toolbar self-observation será controlada.

## DB-TOOLBAR-131
Telemetry suppression nunca será worker-global.

## DB-TOOLBAR-132
Internal toolbar queries podrán ser tagged/excluded.

## DB-TOOLBAR-133
Toolbar requests estarán aisladas de la request observada.

## DB-TOOLBAR-134
CSP-compatible design será soportado.

## DB-TOOLBAR-135
Database-work debug actions podrán ser rate-limited.

## DB-TOOLBAR-136
Resource-intensive actions tendrán budgets.

## DB-TOOLBAR-137
UI virtualization no cambiará backend semantics.

## DB-TOOLBAR-138
Compression será responsabilidad de HTTP layer.

## DB-TOOLBAR-139
Debug payload cache será distinto de Database Cache.

## DB-TOOLBAR-140
Correlation graph utilizará canonical correlations.

## DB-TOOLBAR-141
UI no inventará correlations.

## DB-TOOLBAR-142
Feature availability será explícita.

## DB-TOOLBAR-143
UNSUPPORTED será distinto de DISABLED.

## DB-TOOLBAR-144
Capability será distinta de permission.

## DB-TOOLBAR-145
Effective capability combinará system/config/policy/auth/session.

## DB-TOOLBAR-146
Request lifecycle finalizará debug antes de reset.

## DB-TOOLBAR-147
Toolbar listener failure no alterará DB outcome.

## DB-TOOLBAR-148
Long-lived runtime support requerirá scopes explícitos.

## DB-TOOLBAR-149
Replay de debug no reejecutará la request.

## DB-TOOLBAR-150
Offline reports no requerirán live DB access.

## DB-TOOLBAR-151
Cross-request leakage tendrá pruebas específicas.

## DB-TOOLBAR-152
Cross-tenant leakage tendrá pruebas específicas.

## DB-TOOLBAR-153
UNKNOWN rendering tendrá pruebas específicas.

## DB-TOOLBAR-154
EXPLAIN authorization tendrá pruebas específicas.

## DB-TOOLBAR-155
Disabled toolbar overhead será benchmarkeado.

## DB-TOOLBAR-156
Enabled toolbar cost será observable.

## DB-TOOLBAR-157
Toolbar internal diagnostics usarán namespace propio.

## DB-TOOLBAR-158
Toolbar logging evitará recursion.

## DB-TOOLBAR-159
Toolbar será reemplazable.

## DB-TOOLBAR-160
Toolbar transport será versionado.

## DB-TOOLBAR-161
Toolbar deberá poder coexistir con CLI/IDE consumers.

## DB-TOOLBAR-162
Debug snapshot será la fuente diagnóstica canónica para la toolbar.

## DB-TOOLBAR-163
UI nunca será fuente de Database Truth.

## DB-TOOLBAR-164
UI nunca cambiará Database Truth por default.

## DB-TOOLBAR-165
Debug information deberá seguir siendo útil sin UI.

## DB-TOOLBAR-166
Toolbar deberá preservar todos los estados de incertidumbre relevantes.

## DB-TOOLBAR-167
Toolbar deberá priorizar evidencia sobre inferencias visuales.

## DB-TOOLBAR-168
Toolbar deberá priorizar navegación problema→evidencia→origen→recomendación.

## DB-TOOLBAR-169
Database internals permanecerán encapsulados.

## DB-TOOLBAR-170
La integración oficial no convertirá debugging en un canal privilegiado de administración.

---

# 350. Modelo formal

Sea:

```text
D
```

un `DatabaseDebugInformation`.

Sea:

```text
C
```

el contexto de toolbar.

Sea:

```text
P
```

la policy.

Sea:

```text
A
```

la autorización.

El payload efectivo será:

```text
ToolbarPayload
=
Project(
    Redact(
        Authorize(
            Bound(D, P),
            A
        )
    ),
    C
)
```

---

# 351. Effective capability

Para una acción `x`:

```text
CanUse(x)
=
SystemSupports(x)
∧ ConfigAllows(x)
∧ PolicyAllows(x)
∧ UserAuthorized(x)
∧ SessionSupports(x)
```

---

# 352. Toolbar read path

```text
DatabaseDebugInformation
        ↓
Authorization
        ↓
Toolbar Projection
        ↓
Additional Redaction
        ↓
Payload Budget
        ↓
Serialization
        ↓
Browser
```

---

# 353. Diagnostic action path

Para una acción que requiere trabajo adicional:

```text
Browser
   ↓
DebugRecordId
   ↓
Authentication
   ↓
Authorization
   ↓
Action Policy
   ↓
Resource Budget
   ↓
Server-side Reference Resolution
   ↓
Diagnostic Operation
   ↓
Safe Snapshot
   ↓
Redaction
   ↓
Browser
```

---

# 354. Nunca

```text
Browser
↓
Raw SQL
↓
Database execute
```

como arquitectura del Debug Toolbar.

---

# 355. Arquitectura final

```text
                       VoltStack Application
                                │
                                ▼
                       Database Operations
                                │
                                ▼
                      Database Telemetry
                                │
                                ▼
                  DatabaseDebugInformation
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
               Debug Store           Debug Manifest
                    │                       │
                    └───────────┬───────────┘
                                ▼
                   Debug Toolbar Provider
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
                 Access      Policy      Budget
                    │           │           │
                    └───────────┼───────────┘
                                ▼
                   Debug Toolbar Adapter
                                │
                                ▼
                DatabaseDebugToolbarPayload
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
       Summary                Queries             Timeline
          │                     │                     │
          ├──────────────┬───────┴───────────┬─────────┤
          ▼              ▼                   ▼         ▼
    Connections     Transactions            ORM      Cache
          │              │                   │         │
          └──────────────┼───────────────┬───┴─────────┘
                         ▼               ▼
                    Distribution    Diagnostics
                         │               │
                         └───────┬───────┘
                                 ▼
                      Developer Toolbar UI
                                 │
                  ┌──────────────┼──────────────┐
                  ▼              ▼              ▼
              Source Link    Safe Export    Explicit
                                             EXPLAIN
```

---

# 356. Filosofía de diseño

VoltStack deberá preferir:

```text
Debug snapshots
over
live runtime access

Read-only inspection
over
administrative mutation

Server-side redaction
over
client-side hiding

Structured diagnostics
over
raw dumps

Explicit EXPLAIN
over
automatic extra queries

Stable correlations
over
UI inference

Evidence
over
guesswork

Bounded payloads
over
unlimited history

Lazy UI retrieval
over
huge response payloads

Runtime isolation
over
worker-global state

Secure defaults
over
maximum debug exposure
```

---

# 357. Regla maestra

> **La Developer Debug Toolbar de VoltStack Database deberá ser una herramienta de observación, navegación y diagnóstico construida exclusivamente sobre snapshots seguros y contratos explícitos; no deberá convertirse en un canal alternativo para acceder, modificar o controlar el estado vivo del motor de base de datos.**

En forma compacta:

```text
Observe
↓
Diagnose
↓
Explain
↓
Navigate
```

pero no:

```text
Observe
↓
Mutate Runtime
```

---

# 358. Resultado del Bloque 21

Con este documento queda completado el bloque:

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
✓ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
✓ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
✓ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
✓ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
✓ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
✓ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
✓ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

La cadena completa queda:

```text
Database Operation
      ↓
Telemetry
      ↓
Profiling
      ↓
Detection
      ↓
Debug Information
      ↓
Developer Debug Toolbar
```

sin romper las fronteras arquitectónicas del Database Engine.

---

# 359. Siguiente bloque

A partir del siguiente documento comienza:

```text
BLOCK 22 — SECURITY
```

con:

```text
226_DATABASE_SECURITY_ARCHITECTURE.md
227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
233_DATABASE_QUERY_AUDIT_SYSTEM.md
234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 360. Siguiente documento

```text
226_DATABASE_SECURITY_ARCHITECTURE.md
```

Este documento deberá establecer la arquitectura maestra de seguridad de todo `VoltStack/Quantum/Database`, incluyendo:

```text
Security Boundaries
Trust Model
Threat Model
SQL Injection Defense
Query Input Security
Identifier Security
Parameter Binding
Raw SQL Security
Credential Protection
Connection Security
TLS
Certificate Validation
Data Access
Authorization Integration
Tenant Isolation
Shard Isolation
Sensitive Data Classification
Encryption
Logging Redaction
Telemetry Security
Debug Security
Audit
Database Permissions
Least Privilege
Secret Rotation
Runtime Isolation
Extension Security
Driver Security
Migration Security
Backup Security
Failure Security
Security Diagnostics
Secure Defaults
```

bajo una regla central:

> **La seguridad de VoltStack Database deberá ser transversal y basada en múltiples fronteras de confianza; ninguna capa individual —ORM, Query Builder, parameter binding, driver, autorización o configuración— deberá considerarse suficiente por sí sola para proteger el sistema completo.**