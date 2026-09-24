# 176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md

# VoltStack Quantum Database
## Database Read/Write Connection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 176 — Database Read/Write Connection System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `175_DATABASE_TRANSACTION_EVENT_SYSTEM.md`  
**Siguiente documento:** `177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md`

---

# 1. Propósito

`Database Read/Write Connection System` define la arquitectura mediante la cual VoltStack representará, resolverá y administrará conexiones de base de datos con diferentes roles operativos de lectura y escritura.

El sistema permitirá que una aplicación configure una conexión lógica como:

```text
database.default
```

mientras internamente pueda disponer de:

```text
READ
WRITE
READ_WRITE
```

como roles diferenciados.

Ejemplo conceptual:

```text
Application
    │
    ▼
Logical Database Connection
    │
    ├── READ
    │    ├── node-r1
    │    └── node-r2
    │
    └── WRITE
         └── node-primary
```

Sin embargo, esta separación no deberá introducir en las capas superiores conocimiento sobre:

- hosts concretos;
- replicas;
- primary nodes;
- failover;
- load balancing;
- sharding;
- topología física.

La regla central será:

> **VoltStack distinguirá entre la identidad lógica de una base de datos, el rol requerido por una operación y la conexión física finalmente utilizada; `READ` y `WRITE` serán capacidades/intenciones lógicas de conexión, no sinónimos rígidos de `replica` y `primary`.**

Formalmente:

```text
DatabaseIdentity
≠
ConnectionRole
≠
ConnectionEndpoint
≠
PhysicalConnection
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. conexión lógica única;
2. roles de conexión;
3. `READ`;
4. `WRITE`;
5. `READ_WRITE`;
6. conjuntos de conexiones por rol;
7. configuración separada;
8. herencia de configuración;
9. resolución de role-capable connections;
10. connection leases;
11. conexión autoritativa;
12. transaction pinning;
13. soporte de read-after-write;
14. promotion de intención;
15. prevención de write sobre conexiones read-only;
16. integración con Connection Manager;
17. integración con Query Engine;
18. integración con ORM;
19. integración con Transaction System;
20. aislamiento de tenant/database/shard;
21. connection state/reset;
22. persistent-runtime safety;
23. extensibilidad;
24. telemetry;
25. diagnostics;
26. testing.

---

# 3. No objetivos

Este documento no define completamente:

- algoritmo de routing de queries;
- selección entre múltiples replicas;
- replica lag;
- sticky connections;
- failover;
- load balancing;
- distributed transactions;
- sharding;
- partition routing.

Estos temas se desarrollarán en:

```text
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
182_DATABASE_LOAD_BALANCING_SYSTEM.md
183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

---

# 4. Problema fundamental

Una aplicación simple puede utilizar:

```text
Application
    │
    ▼
Database
    │
    ▼
Single Server
```

Pero una infraestructura mayor puede utilizar:

```text
                     ┌── Replica A
Application ── DB ───┼── Replica B
                     ├── Replica C
                     └── Primary
```

Las capas superiores no deberían cambiar su API debido a esa topología.

Idealmente:

```php
$user = User::find(42);

$user->name = 'Alice';

$user->save();
```

sin especificar manualmente:

```php
useReplica();
usePrimary();
```

para cada operación normal.

---

# 5. Separación lógica

VoltStack deberá distinguir:

```text
Logical Database
        │
        ▼
Connection Role
        │
        ▼
Connection Candidate
        │
        ▼
Connection Lease
        │
        ▼
Physical Connection
```

Cada nivel tendrá responsabilidad diferente.

---

# 6. Logical Database Connection

Una aplicación podrá declarar:

```text
default
analytics
billing
legacy
```

como conexiones/bases lógicas.

Ejemplo:

```php
DB::connection('default');
```

Esto no deberá implicar un único socket físico.

---

# 7. Logical connection ≠ physical connection

Regla:

```text
LogicalConnection
≠
PhysicalConnection
```

Una conexión lógica podrá representar:

```text
1 writer
+
N readers
```

o incluso:

```text
N role-capable endpoints
```

---

# 8. DatabaseConnectionId

Identidad conceptual:

```php
final readonly class DatabaseConnectionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
default
```

---

# 9. ConnectionRole

Modelo:

```php
enum ConnectionRole
{
    case READ;
    case WRITE;
    case READ_WRITE;
}
```

---

# 10. READ

`READ` representa una conexión que puede satisfacer operaciones clasificadas como lectura.

No significa necesariamente:

```text
replica
```

---

# 11. WRITE

`WRITE` representa una conexión capaz de satisfacer operaciones que requieren escritura o autoridad de escritura.

No significa necesariamente:

```text
one specific primary server forever
```

---

# 12. READ_WRITE

Representa un endpoint/conexión capaz de satisfacer ambos roles.

Ejemplo típico:

```text
single-node deployment
```

---

# 13. Deployment simple

VoltStack deberá permitir:

```text
READ_WRITE
     │
     ▼
localhost:3306
```

sin obligar al usuario a configurar artificialmente:

```text
read:
write:
```

---

# 14. Deployment distribuido

Posteriormente:

```text
READ
├── db-r1
├── db-r2
└── db-r3

WRITE
└── db-primary
```

---

# 15. Role ≠ topology

Regla fundamental:

```text
READ ≠ REPLICA
WRITE ≠ PRIMARY
```

Un primary puede atender lecturas.

Una infraestructura gestionada puede ocultar completamente qué servidor es primary.

---

# 16. Endpoint

Modelo conceptual:

```php
final readonly class DatabaseEndpoint
{
    public function __construct(
        public EndpointId $id,
        public ConnectionRoleSet $roles,
        public DatabaseEndpointConfiguration $configuration,
    ) {}
}
```

---

# 17. EndpointId

La identidad de endpoint será distinta de:

```text
DatabaseConnectionId
```

Ejemplo:

```text
DatabaseConnectionId = default

EndpointIds:
    mysql-a
    mysql-b
    mysql-c
```

---

# 18. Role set

Un endpoint podrá declarar:

```text
{READ}
{WRITE}
{READ, WRITE}
```

internamente normalizable a capacidades.

---

# 19. ConnectionRoleSet

```php
final readonly class ConnectionRoleSet
{
    public function supports(ConnectionRole $role): bool;
}
```

---

# 20. Connection capability

La selección deberá basarse en:

```text
required role
⊆
endpoint capabilities
```

No en nombres de hosts.

---

# 21. ReadWriteConnectionDefinition

Modelo:

```php
final readonly class ReadWriteConnectionDefinition
{
    public function __construct(
        public DatabaseConnectionId $id,
        public ConnectionDefinition $base,
        public ConnectionEndpointSet $endpoints,
        public ReadWriteConnectionPolicy $policy,
    ) {}
}
```

---

# 22. Base configuration

Configuración común:

```text
driver
database
username
password reference
charset
timezone
options
```

podrá definirse una sola vez.

---

# 23. Endpoint overrides

Cada endpoint podrá sobrescribir únicamente valores necesarios:

```text
host
port
socket
TLS
role
weight
```

---

# 24. Configuration inheritance

Ejemplo:

```php
'database' => [
    'connections' => [
        'default' => [
            'driver' => 'mysql',
            'database' => 'app',

            'write' => [
                'host' => 'db-primary.internal',
            ],

            'read' => [
                ['host' => 'db-r1.internal'],
                ['host' => 'db-r2.internal'],
            ],
        ],
    ],
];
```

Conceptualmente:

```text
Base Configuration
      │
      ├── READ overrides
      └── WRITE overrides
```

---

# 25. Credentials

Los endpoints podrán:

```text
share credentials
```

o utilizar:

```text
role-specific credentials
```

---

# 26. Read-only credentials

Una configuración avanzada podrá utilizar:

```text
reader_user
```

con permisos DB read-only.

Esto proporciona defensa adicional.

---

# 27. Role declaration ≠ DB permission

Sin embargo:

```text
ConnectionRole::READ
```

es una semántica de VoltStack.

No demuestra por sí misma que el usuario DB carezca de permisos de escritura.

---

# 28. Defense in depth

Idealmente:

```text
VoltStack READ role
+
DB read-only credentials
```

cuando la infraestructura lo permita.

---

# 29. Connection group

Una conexión lógica podrá contener:

```php
final readonly class ReadWriteConnectionGroup
{
    public function __construct(
        public DatabaseConnectionId $database,
        public ReadConnectionSet $read,
        public WriteConnectionSet $write,
    ) {}
}
```

---

# 30. ReadConnectionSet

Representa candidatos capaces de atender lectura.

```php
interface ReadConnectionSet
{
    public function candidates(): iterable;
}
```

---

# 31. WriteConnectionSet

Representa candidatos capaces de atender escritura.

```php
interface WriteConnectionSet
{
    public function candidates(): iterable;
}
```

---

# 32. Candidate ≠ open connection

Un candidato no deberá implicar:

```text
open socket
```

Puede ser únicamente una definición.

---

# 33. Lazy physical connection

VoltStack deberá favorecer:

```text
resolve definition
→ select candidate
→ acquire lease
→ connect if necessary
```

---

# 34. Connection intent

`ConnectionRole` describe capacidades.

La operación deberá expresar una intención.

Modelo:

```php
enum ConnectionIntent
{
    case READ;
    case WRITE;
    case AUTHORITATIVE_READ;
}
```

---

# 35. Intent ≠ role

```text
ConnectionIntent
≠
ConnectionRole
```

Ejemplo:

```text
AUTHORITATIVE_READ
```

es una intención de consistencia.

Puede requerir un endpoint:

```text
WRITE
```

aunque la operación SQL sea `SELECT`.

---

# 36. Read intent

```text
READ
```

significa:

> La operación puede ejecutarse sobre una conexión elegible para lectura bajo la política de consistencia actual.

---

# 37. Write intent

```text
WRITE
```

significa:

> La operación requiere una conexión con autoridad/capacidad de escritura.

---

# 38. Authoritative read

```text
AUTHORITATIVE_READ
```

significa:

> La operación es una lectura, pero requiere una fuente considerada autoritativa por la política activa.

---

# 39. Example

Después de:

```php
$user->save();
```

una lectura inmediata:

```php
User::find($user->id);
```

podría necesitar:

```text
AUTHORITATIVE_READ
```

si las replicas pueden tener retraso.

---

# 40. Sticky routing

Cómo se determina automáticamente esa intención será responsabilidad principalmente de:

```text
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

---

# 41. ConnectionRequirement

Modelo más extensible:

```php
final readonly class ConnectionRequirement
{
    public function __construct(
        public ConnectionIntent $intent,
        public bool $transactionRequired,
        public bool $authoritative,
        public ?ConsistencyRequirement $consistency,
    ) {}
}
```

---

# 42. Connection requirement expansion

En el futuro podrá incluir:

```text
tenant
shard
region
read consistency
transaction affinity
capability requirements
```

sin contaminar Query Builder con infraestructura física.

---

# 43. ConnectionRoleResolver

Contrato conceptual:

```php
interface ConnectionRoleResolver
{
    public function resolve(
        ConnectionRequirement $requirement,
        ReadWriteConnectionGroup $group,
    ): ConnectionRoleResolution;
}
```

---

# 44. Role resolution

Ejemplos:

```text
READ
→ READ candidate preferred
```

```text
WRITE
→ WRITE-capable candidate
```

```text
AUTHORITATIVE_READ
→ authoritative role candidate
```

---

# 45. Role resolution ≠ endpoint selection

Muy importante:

```text
Role Resolution
≠
Endpoint Selection
```

El primero determina:

```text
qué clase de conexión necesitamos
```

El segundo:

```text
qué endpoint concreto utilizamos
```

---

# 46. Routing separation

Arquitectura:

```text
Query/Operation
      │
      ▼
Connection Intent
      │
      ▼
Role Resolution
      │
      ▼
Eligible Endpoint Set
      │
      ▼
Routing / Selection
      │
      ▼
Endpoint
      │
      ▼
Connection Lease
```

---

# 47. Document boundary

Este documento se concentra hasta:

```text
Eligible Endpoint Set
```

El documento 177 profundizará el routing.

---

# 48. Read fallback

Una política podrá permitir:

```text
READ intent
→ no READ endpoint
→ WRITE endpoint
```

porque un writer normalmente puede servir lecturas si su capability profile lo permite.

---

# 49. Write fallback

Nunca:

```text
WRITE intent
→ READ-only endpoint
```

---

# 50. Asymmetric compatibility

Formalmente:

```text
WRITE-capable → may satisfy READ
READ-only → cannot satisfy WRITE
```

siempre sujeto a capabilities/policy.

---

# 51. READ_WRITE compatibility

```text
READ_WRITE
```

puede satisfacer:

```text
READ
WRITE
```

---

# 52. ConnectionRoleCompatibility

```php
interface ConnectionRoleCompatibility
{
    public function canSatisfy(
        ConnectionRoleSet $capabilities,
        ConnectionRequirement $requirement,
    ): bool;
}
```

---

# 53. Explicit read connection

El usuario podrá pedir:

```php
DB::connection('default')
    ->read();
```

como override avanzado.

---

# 54. Explicit write connection

También:

```php
DB::connection('default')
    ->write();
```

---

# 55. Explicit API semantics

Estos métodos deberán expresar:

```text
connection intent
```

No devolver necesariamente un socket inmediatamente.

---

# 56. Prefer automatic intent

API normal:

```php
DB::table('users')->get();

DB::table('users')->insert([...]);
```

deberá poder determinar intención automáticamente.

---

# 57. Query classification

En general:

```text
SELECT
→ READ
```

y:

```text
INSERT
UPDATE
DELETE
→ WRITE
```

pero VoltStack no deberá basarse únicamente en el primer token SQL.

---

# 58. Why

VoltStack posee:

```text
Query AST
Semantic Query Model
Execution Plan
```

por lo que la intención puede determinarse semánticamente antes de compilar SQL.

---

# 59. QueryIntent

El Query Engine podrá producir:

```php
enum QueryExecutionIntent
{
    case READ;
    case WRITE;
    case READ_FOR_UPDATE;
    case SCHEMA;
    case ADMINISTRATIVE;
}
```

---

# 60. READ_FOR_UPDATE

Una consulta:

```sql
SELECT ... FOR UPDATE
```

es sintácticamente una lectura, pero operacionalmente requiere:

```text
WRITE / transactional authoritative connection
```

---

# 61. Pessimistic locking integration

Por tanto:

```text
Pessimistic Lock
→ WRITE-capable connection
```

aunque el statement principal sea SELECT.

---

# 62. Schema operations

Operaciones como:

```text
CREATE TABLE
ALTER TABLE
DROP INDEX
```

requieren write authority.

---

# 63. Raw SQL

Raw SQL presenta mayor dificultad.

El usuario podrá necesitar declarar intención:

```php
DB::rawStatement(
    $sql,
    intent: ConnectionIntent::WRITE,
);
```

---

# 64. Raw SQL classification

VoltStack podrá intentar clasificación conservadora, pero:

```text
uncertain raw SQL
```

nunca deberá enviarse silenciosamente a una conexión read-only si existe riesgo de escritura.

---

# 65. Safe default for unknown

Regla recomendada:

```text
UNKNOWN execution intent
→ WRITE
```

cuando no pueda demostrarse que es read-only.

---

# 66. Safety over optimization

Esto sacrifica potencialmente:

```text
read distribution
```

pero preserva:

```text
correctness
```

---

# 67. Connection lease

La capa de conexión ya establecida deberá continuar utilizando:

```text
ConnectionLease
```

como representación de una conexión física adquirida.

---

# 68. Role-aware lease

Una lease podrá exponer:

```php
interface RoleAwareConnectionLease extends ConnectionLease
{
    public function role(): ConnectionRoleSet;

    public function endpointId(): EndpointId;
}
```

---

# 69. Lease role immutable

Durante la vida de una lease:

```text
endpoint identity
role capabilities
```

no cambiarán mágicamente.

---

# 70. Connection promotion

Una operación puede comenzar sin lease y requerir:

```text
READ
```

para después necesitar:

```text
WRITE
```

---

# 71. Promotion before acquisition

Si todavía no existe lease:

```text
READ intent
→ WRITE intent
```

simplemente se resuelve nuevamente.

---

# 72. Promotion after acquisition

Si ya existe:

```text
READ-only lease
```

no deberá convertirse internamente en write-capable.

---

# 73. No physical role mutation

Prohibido:

```text
ReadLease.role = WRITE
```

---

# 74. Promotion means reacquisition

Fuera de transaction:

```text
READ lease
→ release
→ resolve WRITE
→ acquire WRITE lease
```

cuando sea seguro.

---

# 75. Transaction promotion

Dentro de una transacción el comportamiento es distinto.

---

# 76. Transaction connection affinity

Una transacción debe utilizar:

```text
one pinned physical connection
```

según los documentos 164–175.

---

# 77. Write transaction

Una transacción capaz de escribir deberá adquirir:

```text
WRITE-capable connection
```

desde el inicio.

---

# 78. Transaction default

La política por defecto de VoltStack deberá favorecer:

```text
DB::transaction(...)
→ WRITE-capable connection
```

aunque la primera query sea SELECT.

---

# 79. Why

Evita:

```text
BEGIN on replica
↓
later UPDATE
↓
cannot promote without changing physical transaction
```

---

# 80. Read-only transaction

Una transacción explícitamente:

```text
readOnly = true
```

podrá utilizar una conexión READ si:

```text
platform
+
topology
+
transaction policy
```

lo permiten.

---

# 81. Read-only transaction write

Si posteriormente intenta escritura:

```text
reject
```

No promover transparentemente a otra conexión.

---

# 82. Transaction cannot migrate

Regla:

```text
ActiveTransaction
→ pinned connection
→ no endpoint migration
```

---

# 83. Failover implication

Si la conexión muere dentro de una transacción:

```text
do not reconnect transaction to another node
```

y fingir continuidad.

---

# 84. Failover boundary

El documento 181 profundizará esto.

---

# 85. Transaction read

Dentro de una write transaction:

```text
SELECT
```

deberá utilizar la conexión pinned de la transaction.

---

# 86. No replica hopping

Prohibido por default:

```text
BEGIN writer
UPDATE
SELECT from replica
COMMIT writer
```

como comportamiento implícito.

---

# 87. Why

La replica puede no observar:

```text
uncommitted writes
```

ni ofrecer las mismas garantías de aislamiento.

---

# 88. Transaction context integration

`TransactionContext` almacena:

```text
PinnedConnectionLease
```

El Read/Write Connection System deberá respetarlo.

---

# 89. Resolution order

Conceptualmente:

```text
Is active transaction?
    │
    ├── YES → use pinned lease
    │
    └── NO  → resolve intent normally
```

---

# 90. Transaction requirement mismatch

Si una operación WRITE aparece dentro de una transaction pinned a READ:

```text
TransactionConnectionRoleMismatchException
```

---

# 91. No hidden recovery

VoltStack no deberá:

```text
open writer
copy transaction state
continue
```

porque no existe una forma general de transferir una transacción física.

---

# 92. ORM integration

ORM no seleccionará hosts.

Flujo:

```text
ORM
 ↓
Persistence Engine
 ↓
Query Model
 ↓
Execution Plan
 ↓
Connection Requirement
 ↓
Read/Write Connection System
```

---

# 93. ORM reads

Entity query:

```php
User::find(10);
```

normalmente:

```text
READ
```

fuera de transaction/sticky context.

---

# 94. ORM writes

```php
$user->save();
```

eventualmente produce:

```text
WRITE
```

durante persistence execution.

---

# 95. flush()

Un `flush()` que contiene cambios requiere:

```text
WRITE
```

---

# 96. flush inside transaction

Debe utilizar:

```text
transaction pinned write-capable lease
```

---

# 97. IdentityMap warning

Una consulta ORM puede no tocar DB si:

```text
IdentityMap
```

ya contiene la entidad.

Por tanto:

```text
ORM lookup
≠
necessarily database READ
```

---

# 98. Refresh

Un:

```php
$entityManager->refresh($user);
```

podrá solicitar:

```text
AUTHORITATIVE_READ
```

según policy.

---

# 99. Consistency requirement

Modelo:

```php
enum ReadConsistencyRequirement
{
    case EVENTUAL;
    case SESSION;
    case AUTHORITATIVE;
}
```

Este modelo podrá expandirse posteriormente.

---

# 100. Eventual

```text
EVENTUAL
```

permite usar una fuente de lectura elegible aunque pueda estar ligeramente retrasada.

---

# 101. Session

```text
SESSION
```

expresa que la aplicación requiere observar determinadas escrituras realizadas dentro del contexto lógico actual.

---

# 102. Authoritative

```text
AUTHORITATIVE
```

requiere una fuente considerada autoritativa bajo la topología activa.

---

# 103. Consistency ≠ isolation

Regla:

```text
ReadConsistencyRequirement
≠
TransactionIsolation
```

---

# 104. Example

Una lectura fuera de transacción puede requerir:

```text
AUTHORITATIVE
```

sin existir isolation level.

---

# 105. Replica consistency

El significado exacto de eventual/session consistency con replicas se desarrollará en 178–180.

---

# 106. Read-after-write

Problema:

```text
WRITE primary
     │
     ▼
COMMIT
     │
     ▼
READ replica
```

La replica podría aún no contener la escritura.

---

# 107. Connection system responsibility

Este documento define que la lectura puede expresar:

```text
authoritative/session requirement
```

pero no define todavía toda la sticky policy.

---

# 108. Write observation token

Arquitectura futura podrá producir:

```text
WriteObservationToken
```

después de una escritura confirmada.

---

# 109. Token purpose

Podría indicar:

```text
this execution scope has observed a successful write
```

sin contener necesariamente posición de replication log.

---

# 110. Sticky system

El documento 180 podrá utilizar esa información para decidir:

```text
subsequent reads → writer
```

durante un scope/periodo.

---

# 111. Connection configuration model

Propuesta:

```php
final readonly class DatabaseConnectionConfiguration
{
    public function __construct(
        public DatabaseConnectionId $id,
        public DriverId $driver,
        public DatabaseName $database,
        public ConnectionEndpointSet $endpoints,
        public ReadWriteConnectionPolicy $readWrite,
    ) {}
}
```

---

# 112. Endpoint configuration

```php
final readonly class DatabaseEndpointConfiguration
{
    public function __construct(
        public EndpointId $id,
        public ConnectionRoleSet $roles,
        public HostDefinition $host,
        public CredentialReference $credentials,
        public ConnectionOptions $options,
    ) {}
}
```

---

# 113. Credentials as references

Evitar:

```text
plain password copied into diagnostic objects
```

Preferir:

```text
CredentialReference
```

---

# 114. Endpoint tags

Opcionalmente:

```text
region
zone
cluster
provider
```

podrán existir como metadata.

---

# 115. Tags ≠ routing logic

Core no deberá codificar:

```text
if region == ...
```

en el Connection System.

Serán consumidos por políticas posteriores.

---

# 116. Endpoint health

Un endpoint podrá tener health state.

Pero:

```text
ConnectionDefinition
≠
EndpointHealth
```

---

# 117. Immutable vs mutable

Definición:

```text
immutable/shared
```

Health:

```text
runtime/dynamic
```

---

# 118. Health system

La integración detallada llegará con:

```text
failover
load balancing
health checks
```

---

# 119. Candidate model

```php
final readonly class ConnectionCandidate
{
    public function __construct(
        public EndpointId $endpoint,
        public ConnectionRoleSet $roles,
        public ConnectionCandidateMetadata $metadata,
    ) {}
}
```

---

# 120. Candidate metadata

Puede contener:

```text
region
zone
weight
health eligibility
pool identity
```

pero no debe incluir estado mutable global directamente.

---

# 121. Candidate set

```php
final readonly class ConnectionCandidateSet
{
    /** @var list<ConnectionCandidate> */
    public array $candidates;
}
```

---

# 122. Empty candidate set

Si no existe endpoint compatible:

```text
ConnectionRoleUnavailableException
```

---

# 123. No silent cross-database fallback

Si:

```text
database.default
```

no tiene writer:

VoltStack no podrá usar:

```text
database.other
```

automáticamente.

---

# 124. Database identity invariant

Routing nunca deberá alterar silenciosamente:

```text
DatabaseIdentity
```

---

# 125. Tenant integration

Cuando el paquete Multitenancy esté instalado:

```text
Tenant
→ DatabaseConnectionId
→ ReadWriteConnectionGroup
```

podrá variar.

---

# 126. Core independence

Sin embargo:

```text
ReadWriteConnectionSystem
```

no dependerá directamente del paquete Multitenancy.

---

# 127. Execution domain

Se reutilizará el concepto:

```text
DatabaseExecutionDomain
```

con potencial información de:

```text
logical database
tenant context
shard
partition
```

mediante contratos genéricos.

---

# 128. Domain first

Antes de seleccionar READ/WRITE deberá estar resuelto:

```text
qué base lógica
```

se está consultando.

---

# 129. Correct ordering

```text
Execution Domain
       ↓
Logical Database
       ↓
Connection Intent
       ↓
Role Resolution
       ↓
Endpoint Selection
```

---

# 130. Incorrect ordering

Nunca:

```text
pick replica
↓
guess tenant/database
```

---

# 131. Sharding integration

En el futuro:

```text
Shard Routing
→ shard-specific ReadWriteConnectionGroup
→ READ/WRITE selection
```

---

# 132. Partition routing

Igualmente:

```text
Partition
→ eligible connection group
→ role selection
```

---

# 133. Connection pools

Podrán existir pools separados:

```text
ReadPool
WritePool
```

---

# 134. Pool ≠ role

Un pool puede contener conexiones a endpoints con determinado role, pero:

```text
ConnectionPool
≠
ConnectionRole
```

---

# 135. Pool identity

Ejemplo:

```text
pool.default.read.r1
pool.default.read.r2
pool.default.write.primary
```

es detalle de implementación.

---

# 136. Pool acquisition

Flujo:

```text
Endpoint selected
      ↓
Pool resolved
      ↓
Lease acquired
```

---

# 137. Pool exhaustion

Si el pool READ está agotado, una policy podría:

```text
wait
fail
fallback to WRITE
```

para lecturas.

---

# 138. Write pool exhaustion

No podrá hacer fallback hacia:

```text
READ-only pool
```

---

# 139. Fallback policy

Modelo:

```php
final readonly class ReadWriteFallbackPolicy
{
    public function __construct(
        public bool $allowReadOnWrite,
        public bool $allowWriteOnRead = false,
    ) {}
}
```

---

# 140. allowWriteOnRead

En core deberá permanecer:

```text
false
```

y conceptualmente no debería ofrecerse como configuración normal.

---

# 141. Read fallback cost

Enviar reads al writer puede aumentar:

```text
writer load
```

por lo que el routing system deberá poder observarlo.

---

# 142. ReadWriteConnectionPolicy

```php
final readonly class ReadWriteConnectionPolicy
{
    public function __construct(
        public bool $preferDedicatedReaders,
        public bool $allowReadOnWriter,
        public UnknownIntentPolicy $unknownIntent,
        public TransactionConnectionPolicy $transactions,
    ) {}
}
```

---

# 143. UnknownIntentPolicy

```php
enum UnknownIntentPolicy
{
    case REQUIRE_WRITE;
    case REJECT;
}
```

---

# 144. No REQUIRE_READ default

No deberá existir como default seguro:

```text
unknown → read
```

---

# 145. Connection pinning

Además de transaction pinning, otros sistemas podrán solicitar:

```text
connection affinity
```

---

# 146. Affinity ≠ transaction

Ejemplo:

```text
sticky reads
```

pueden preferir el mismo role/endpoint sin existir una transaction.

---

# 147. ConnectionAffinity

Modelo futuro-compatible:

```php
final readonly class ConnectionAffinity
{
    public function __construct(
        public ConnectionAffinityMode $mode,
        public ?EndpointId $preferredEndpoint,
    ) {}
}
```

---

# 148. Affinity modes

Conceptualmente:

```text
NONE
ROLE
ENDPOINT
TRANSACTION
```

---

# 149. Transaction affinity strongest

```text
TRANSACTION
```

significa:

```text
exact pinned lease
```

No solo mismo role.

---

# 150. Sticky affinity weaker

Sticky routing puede significar:

```text
prefer WRITE role
```

sin exigir la misma physical connection.

---

# 151. Connection state

Las conexiones físicas pueden tener estado:

```text
session variables
timezone
isolation defaults
temporary tables
session locks
```

---

# 152. Role reuse safety

Una conexión devuelta a pool deberá pasar por:

```text
Connection State Reset System
```

definido anteriormente.

---

# 153. No state leakage

Especialmente:

```text
transaction isolation
read-only session state
session variables
```

no deberán filtrarse entre borrowers.

---

# 154. Read-only session flag

Si una plataforma utiliza un flag de sesión para reforzar read-only:

```text
set read only
```

deberá restaurarse/resetearse antes de reutilización incompatible.

---

# 155. Endpoint role is not session flag

```text
EndpointRole
≠
CurrentSessionReadOnlyState
```

---

# 156. Connection reset failure

Si el reset falla:

```text
connection quarantine/discard
```

No devolver al pool como healthy.

---

# 157. Role mismatch after reuse

Antes de entregar una lease deberá validarse que:

```text
physical connection
```

sigue siendo válida para el endpoint/pool esperado.

---

# 158. Persistent runtime

FrankenPHP, RoadRunner y OpenSwoole hacen especialmente importante distinguir:

```text
shared immutable topology
```

de:

```text
scoped mutable routing/lease state
```

---

# 159. Shared state

Puede compartirse:

```text
CompiledConnectionDefinitions
EndpointDefinitions
RoleCapabilities
ReadWritePolicies
```

---

# 160. Scoped state

Debe permanecer scoped:

```text
current lease
transaction pin
sticky write observation
connection affinity
routing decision
temporary fallback
```

---

# 161. No static current writer

Prohibido:

```php
static $currentWriter;
```

---

# 162. No static current reader

Igualmente:

```php
static $currentReader;
```

---

# 163. Request isolation

Formalmente:

```text
MutableConnectionState(Request A)
∩
MutableConnectionState(Request B)
=
∅
```

---

# 164. Coroutine isolation

En OpenSwoole:

```text
ConnectionLease(Coroutine A)
≠
ConnectionLease(Coroutine B)
```

salvo un mecanismo de multiplexing explícitamente soportado por driver/protocol.

---

# 165. Shared physical connection warning

No deberá asumirse que una única conexión física PDO puede utilizarse concurrentemente por múltiples coroutines.

---

# 166. Driver capabilities

Driver/connection capability model deberá declarar si soporta:

```text
concurrent use
multiplexing
read-only enforcement
transaction read-only
connection role validation
```

---

# 167. Platform capabilities

Platform podrá declarar:

```text
supportsReadOnlyTransaction()
supportsReadOnlySession()
supportsReplicaReadOnlyDetection()
```

si existe una semántica portable suficiente.

---

# 168. Capability ≠ vendor conditional

No:

```php
if ($driver === 'pgsql') { ... }
```

en capas superiores.

Preferir:

```php
$capabilities->supportsReadOnlyTransaction();
```

---

# 169. Read-only detection

Una extensión podrá verificar si un endpoint esperado como reader está realmente en modo read-only.

Pero esto será:

```text
health/capability evidence
```

no identidad absoluta.

---

# 170. Topology changes

En sistemas gestionados:

```text
node A primary
node B replica
```

puede convertirse en:

```text
node B primary
node A replica
```

---

# 171. Static labels limitation

Por eso:

```text
host name
```

no deberá ser la autoridad semántica del role.

---

# 172. Dynamic role evidence

Arquitectura futura podrá combinar:

```text
configured role
+
provider topology
+
health probe
+
runtime discovery
```

---

# 173. Effective role

Podrá distinguirse:

```text
DeclaredRole
EffectiveRole
ObservedRole
```

---

# 174. Role confidence

Para infra avanzada:

```php
enum ConnectionRoleConfidence
{
    case CONFIGURED;
    case DISCOVERED;
    case VERIFIED;
    case UNKNOWN;
}
```

---

# 175. Unknown role

Un endpoint con role desconocido no deberá recibir escritura automáticamente.

---

# 176. Fail closed

Para WRITE:

```text
UNKNOWN role/capability
→ reject
```

salvo que el endpoint sea la conexión autoritativa explícitamente configurada bajo otra evidencia segura.

---

# 177. Read uncertainty

Para READ puede existir mayor flexibilidad, pero siempre bajo policy.

---

# 178. Configuration validation

Durante boot deberán detectarse:

```text
no endpoints
invalid roles
duplicate EndpointId
write configuration without writer
incompatible driver
invalid inheritance
credential reference errors
contradictory policies
```

---

# 179. Writer requirement

Una aplicación declarada:

```text
readOnlyApplication = true
```

podría no necesitar writer.

---

# 180. Normal application

Por default, una conexión lógica de aplicación deberá tener:

```text
at least one WRITE-capable candidate
```

si se espera persistencia.

---

# 181. Multiple writers

El modelo no deberá asumir:

```text
exactly one writer
```

porque ciertas arquitecturas futuras pueden exponer múltiples write-capable endpoints.

---

# 182. Multiple writers warning

Que existan múltiples:

```text
WRITE
```

candidates no implica:

```text
multi-primary correctness
```

---

# 183. Selection delegated

La topología/selector determinará cuál puede usarse.

---

# 184. Database semantics remain external

VoltStack no podrá convertir una infraestructura insegura multi-writer en consistente simplemente etiquetándola:

```text
WRITE
```

---

# 185. Read-only application mode

Podrá existir:

```php
DB::connection('analytics')
    ->readOnly();
```

como política de logical connection.

---

# 186. Read-only logical connection

Esto significa:

```text
all WRITE requirements rejected
```

aunque un endpoint físicamente pudiera escribir.

---

# 187. Policy enforcement

```text
LogicalConnectionPolicy
```

puede ser más restrictiva que:

```text
EndpointCapability
```

---

# 188. Capability vs permission

Formalmente:

```text
EffectiveAllowedOperations
=
EndpointCapabilities
∩
ConnectionPolicy
∩
ExecutionContextPolicy
```

---

# 189. Security integration

Authorization de usuario no deberá confundirse con:

```text
database connection role
```

---

# 190. User authorization ≠ connection authorization

```text
User may edit invoice
```

es una decisión de Authorization System.

```text
Query requires WRITE connection
```

es infraestructura DB.

---

# 191. Query security

Un atacante no deberá poder enviar:

```text
role=WRITE
```

desde input HTTP y controlar routing directamente.

---

# 192. Intent source trust

`ConnectionIntent` deberá derivarse de:

```text
trusted query semantics
framework APIs
validated internal policy
```

No de input sin validar.

---

# 193. Raw APIs

APIs raw que permitan indicar intent serán consideradas:

```text
developer-controlled trusted surface
```

---

# 194. Telemetry

El sistema deberá emitir observabilidad sobre:

```text
logical connection
requested intent
resolved role
endpoint selection class
fallback usage
transaction pinning
```

sin exponer secretos.

---

# 195. Metrics

Ejemplos:

```text
db.connection.role.acquire
db.connection.role.fallback
db.connection.role.unavailable
db.connection.role.mismatch
db.connection.authoritative_read
```

---

# 196. Metric labels

Apropiados:

```text
driver
role
intent
fallback
outcome
```

---

# 197. High cardinality

Evitar:

```text
host
database name
tenant id
user id
query text
```

como labels por default.

---

# 198. Tracing

Span attributes podrán incluir bounded metadata:

```text
db.connection.intent = read
db.connection.role = read
db.connection.fallback = false
```

---

# 199. Transaction trace

Dentro de transaction:

```text
db.connection.pinned = true
```

---

# 200. Diagnostics

API conceptual:

```php
DB::connection('default')
    ->readWrite()
    ->inspect();
```

---

# 201. Diagnostic output

Ejemplo:

```text
DATABASE CONNECTION: default

Driver:
    mysql

Logical Mode:
    READ_WRITE

Readers:
    2

Writers:
    1

Read-on-write fallback:
    enabled

Unknown intent:
    require_write

Transaction default:
    WRITE

Compiled:
    yes

Topology generation:
    17
```

---

# 202. Explain API

```php
DB::connection('default')
    ->readWrite()
    ->explain(ConnectionIntent::READ);
```

podría producir:

```text
Requested intent:
    READ

Required capability:
    READ

Preferred role:
    READ

Fallback:
    WRITE allowed

Eligible endpoint count:
    2

Transaction:
    none
```

---

# 203. Transaction explain

```text
Requested intent:
    READ

Active transaction:
    yes

Pinned endpoint:
    writer-1

Resolution:
    PINNED_TRANSACTION_CONNECTION

No replica selection performed.
```

---

# 204. Error hierarchy

Propuesta:

```text
DatabaseReadWriteConnectionException
├── ConnectionRoleUnavailableException
├── ConnectionRoleMismatchException
├── ConnectionIntentUnknownException
├── ConnectionIntentViolationException
├── ReadOnlyConnectionWriteException
├── ReadOnlyTransactionWriteException
├── TransactionConnectionRoleMismatchException
├── ConnectionPromotionNotAllowedException
├── ConnectionEndpointUnavailableException
├── ConnectionEndpointRoleUnknownException
├── ConnectionConfigurationRoleException
├── ConnectionFallbackRejectedException
├── ConnectionAffinityViolationException
└── ReadWriteConnectionInvariantViolationException
```

---

# 205. ReadOnlyConnectionWriteException

Ejemplo:

```text
Logical connection "analytics" is read-only,
but operation requires WRITE.
```

---

# 206. Role mismatch diagnostic

Deberá indicar:

```text
requested intent
required role
actual role
transaction state
logical connection
```

sin credenciales.

---

# 207. Testing architecture

Suite propuesta:

```text
ReadWriteConnectionConfigurationTests
ConnectionRoleTests
ConnectionIntentTests
ConnectionRoleCompatibilityTests
ReadWriteConnectionResolutionTests
ReadFallbackTests
WriteSafetyTests
AuthoritativeReadTests
TransactionPinningTests
ReadOnlyTransactionTests
ConnectionPromotionTests
ConnectionPoolIntegrationTests
ConnectionResetTests
OrmReadWriteIntegrationTests
RawQueryIntentTests
PersistentRuntimeIsolationTests
CoroutineConnectionIsolationTests
ReadWriteConnectionTelemetryTests
ReadWriteConnectionDiagnosticsTests
```

---

# 208. Single connection test

Configurar únicamente:

```text
READ_WRITE
```

y comprobar:

```text
READ → same endpoint
WRITE → same endpoint
```

---

# 209. Split connection test

```text
reader
writer
```

Verificar:

```text
READ → reader
WRITE → writer
```

---

# 210. Read fallback test

Sin reader dedicado:

```text
READ → writer
```

si policy lo permite.

---

# 211. Write safety test

Con solo reader:

```text
WRITE
```

debe fallar.

---

# 212. Unknown intent test

```text
UNKNOWN
```

debe:

```text
WRITE
```

o:

```text
REJECT
```

según policy.

Nunca READ implícito.

---

# 213. Locking test

`SELECT FOR UPDATE` deberá requerir:

```text
WRITE
```

---

# 214. Transaction pinning test

```text
BEGIN
SELECT
UPDATE
SELECT
COMMIT
```

todas las operaciones deberán utilizar:

```text
same pinned lease
```

---

# 215. Read-only transaction test

```text
readOnly transaction
→ SELECT
```

permitido.

```text
readOnly transaction
→ UPDATE
```

rechazado.

---

# 216. Promotion test

Fuera de transaction:

```text
READ
→ release
→ WRITE
```

podrá resolverse.

---

# 217. Transaction promotion test

Dentro de read-only transaction:

```text
READ
→ WRITE
```

deberá rechazarse.

---

# 218. Pool state leakage test

Borrow:

```text
READ session
```

modify state.

Return.

Borrow next scope.

Verificar reset.

---

# 219. Worker isolation test

Request A:

```text
WRITE affinity
```

Request B:

```text
READ
```

B no deberá heredar la affinity de A.

---

# 220. Coroutine isolation test

Dos coroutines con diferentes intents no deberán compartir mutable routing state.

---

# 221. Proposed directory structure

```text
src/Quantum/Database/Connection/ReadWrite/
│
├── ConnectionRole.php
├── ConnectionRoleSet.php
├── ConnectionIntent.php
├── ConnectionRequirement.php
├── ReadConsistencyRequirement.php
│
├── Definition/
│   ├── ReadWriteConnectionDefinition.php
│   ├── DatabaseEndpoint.php
│   ├── EndpointId.php
│   ├── DatabaseEndpointConfiguration.php
│   ├── ConnectionEndpointSet.php
│   └── ReadWriteConnectionGroup.php
│
├── Set/
│   ├── ReadConnectionSet.php
│   ├── WriteConnectionSet.php
│   └── ConnectionCandidateSet.php
│
├── Candidate/
│   ├── ConnectionCandidate.php
│   └── ConnectionCandidateMetadata.php
│
├── Resolution/
│   ├── ConnectionRoleResolver.php
│   ├── DefaultConnectionRoleResolver.php
│   ├── ConnectionRoleResolution.php
│   └── ConnectionRoleCompatibility.php
│
├── Policy/
│   ├── ReadWriteConnectionPolicy.php
│   ├── ReadWriteFallbackPolicy.php
│   ├── UnknownIntentPolicy.php
│   └── TransactionConnectionPolicy.php
│
├── Affinity/
│   ├── ConnectionAffinity.php
│   └── ConnectionAffinityMode.php
│
├── Lease/
│   └── RoleAwareConnectionLease.php
│
├── Capability/
│   ├── ConnectionRoleCapability.php
│   └── ConnectionRoleConfidence.php
│
├── Diagnostics/
│   ├── ReadWriteConnectionInspector.php
│   ├── ConnectionRoleExplanation.php
│   └── ReadWriteConnectionDiagnosticReport.php
│
└── Exception/
    ├── DatabaseReadWriteConnectionException.php
    ├── ConnectionRoleUnavailableException.php
    ├── ConnectionRoleMismatchException.php
    ├── ConnectionIntentUnknownException.php
    ├── ConnectionIntentViolationException.php
    ├── ReadOnlyConnectionWriteException.php
    ├── ReadOnlyTransactionWriteException.php
    ├── TransactionConnectionRoleMismatchException.php
    ├── ConnectionPromotionNotAllowedException.php
    ├── ConnectionEndpointUnavailableException.php
    ├── ConnectionEndpointRoleUnknownException.php
    ├── ConnectionFallbackRejectedException.php
    └── ReadWriteConnectionInvariantViolationException.php
```

---

# 222. Architectural invariants

## DB-RWC-001
Logical Database Connection será distinta de Physical Connection.

## DB-RWC-002
ConnectionRole será distinto de ConnectionIntent.

## DB-RWC-003
READ no será sinónimo de replica.

## DB-RWC-004
WRITE no será sinónimo de primary.

## DB-RWC-005
READ_WRITE podrá satisfacer READ y WRITE.

## DB-RWC-006
READ-only no podrá satisfacer WRITE.

## DB-RWC-007
WRITE-capable podrá satisfacer READ solo cuando policy/capabilities lo permitan.

## DB-RWC-008
Database identity se resolverá antes del role.

## DB-RWC-009
Role resolution ocurrirá antes del endpoint selection.

## DB-RWC-010
Endpoint selection será distinto de role resolution.

## DB-RWC-011
Connection candidate no implicará socket abierto.

## DB-RWC-012
Physical connections podrán adquirirse lazy.

## DB-RWC-013
EndpointId será distinto de DatabaseConnectionId.

## DB-RWC-014
Connection configuration será immutable después de compilación.

## DB-RWC-015
Mutable health state no formará parte de immutable definition.

## DB-RWC-016
Credentials se representarán mediante referencias seguras cuando sea posible.

## DB-RWC-017
Diagnostics no expondrán credentials.

## DB-RWC-018
ConnectionRole no probará permisos reales del DB user.

## DB-RWC-019
Read-only DB credentials podrán utilizarse como defense in depth.

## DB-RWC-020
ConnectionIntent::READ representará una intención semántica.

## DB-RWC-021
ConnectionIntent::WRITE representará requerimiento de write authority.

## DB-RWC-022
Authoritative read será distinto de ordinary read.

## DB-RWC-023
Authoritative read podrá requerir writer.

## DB-RWC-024
Read consistency será distinta de transaction isolation.

## DB-RWC-025
Eventual read no significará incorrect read.

## DB-RWC-026
Session consistency podrá afectar role selection.

## DB-RWC-027
Read-after-write deberá ser representable explícitamente.

## DB-RWC-028
Sticky semantics no serán implementadas como global mutable flag.

## DB-RWC-029
Query intent deberá derivarse preferentemente del semantic Query Model.

## DB-RWC-030
Routing no dependerá únicamente de SQL string parsing.

## DB-RWC-031
SELECT FOR UPDATE requerirá write-capable connection.

## DB-RWC-032
Schema mutation requerirá WRITE.

## DB-RWC-033
Unknown raw query intent nunca se asumirá READ por default.

## DB-RWC-034
Unknown intent será WRITE o REJECT según policy.

## DB-RWC-035
Safety tendrá prioridad sobre read distribution.

## DB-RWC-036
Explicit read/write APIs expresarán intent.

## DB-RWC-037
Explicit intent no requerirá adquisición inmediata de socket.

## DB-RWC-038
Normal Query Builder deberá inferir intent automáticamente.

## DB-RWC-039
ORM no seleccionará hosts directamente.

## DB-RWC-040
Persistence Engine producirá write requirements.

## DB-RWC-041
Entity read normalmente producirá READ fuera de contexts especiales.

## DB-RWC-042
IdentityMap hit podrá evitar DB read.

## DB-RWC-043
ORM refresh podrá solicitar authoritative read.

## DB-RWC-044
flush con cambios requerirá WRITE.

## DB-RWC-045
flush no será commit.

## DB-RWC-046
Connection lease expondrá role capability sin permitir mutarla.

## DB-RWC-047
Read lease no se convertirá mágicamente en write lease.

## DB-RWC-048
Promotion fuera de transaction requerirá nueva resolución/adquisición.

## DB-RWC-049
Active transaction no migrará entre endpoints.

## DB-RWC-050
Transaction utilizará pinned physical lease.

## DB-RWC-051
Write-capable transaction adquirirá writer-capable lease desde el inicio.

## DB-RWC-052
DB::transaction utilizará WRITE-capable connection por default.

## DB-RWC-053
Read-only transaction podrá utilizar reader solo si capabilities/policy lo permiten.

## DB-RWC-054
Read-only transaction no podrá promoverse transparentemente a writer.

## DB-RWC-055
Write dentro de read-only transaction será rechazado.

## DB-RWC-056
Reads dentro de write transaction utilizarán pinned connection.

## DB-RWC-057
Transaction no saltará implícitamente a replica.

## DB-RWC-058
Connection failure no moverá una active transaction a otro endpoint.

## DB-RWC-059
Transaction context tendrá precedencia sobre ordinary routing.

## DB-RWC-060
Transaction role mismatch fallará explícitamente.

## DB-RWC-061
VoltStack no intentará copiar physical transaction state entre conexiones.

## DB-RWC-062
ReadConnectionSet podrá contener cero o más candidates.

## DB-RWC-063
WriteConnectionSet podrá contener cero o más candidates.

## DB-RWC-064
Una aplicación read-only podrá carecer de writer.

## DB-RWC-065
Una aplicación normal podrá validar presencia de writer durante bootstrap.

## DB-RWC-066
Multiple writers no implicarán multi-primary correctness.

## DB-RWC-067
Topology correctness no será inferida únicamente por role labels.

## DB-RWC-068
Configured role podrá diferenciarse de observed/effective role.

## DB-RWC-069
Unknown write capability fallará cerrado.

## DB-RWC-070
Topology changes no modificarán silenciosamente immutable definitions.

## DB-RWC-071
Runtime topology evidence será state separado.

## DB-RWC-072
Read fallback a writer será policy-driven.

## DB-RWC-073
Write fallback a reader estará prohibido.

## DB-RWC-074
Read fallback deberá ser observable.

## DB-RWC-075
Pool exhaustion no autorizará write-on-reader.

## DB-RWC-076
ConnectionPool será distinto de ConnectionRole.

## DB-RWC-077
Endpoint selection precederá pool acquisition.

## DB-RWC-078
Pool lease será liberada correctamente después de uso.

## DB-RWC-079
Connection state deberá resetearse antes de reuse.

## DB-RWC-080
Failed reset deberá descartar/quarantine connection.

## DB-RWC-081
Session isolation no deberá filtrarse entre borrowers.

## DB-RWC-082
Read-only session state no deberá filtrarse entre borrowers.

## DB-RWC-083
Endpoint role será distinto de session read-only state.

## DB-RWC-084
Persistent workers no almacenarán current reader global.

## DB-RWC-085
Persistent workers no almacenarán current writer global.

## DB-RWC-086
Request mutable connection state estará aislado.

## DB-RWC-087
Job mutable connection state estará aislado.

## DB-RWC-088
Coroutine mutable connection state estará aislado.

## DB-RWC-089
Fiber mutable connection state estará aislado.

## DB-RWC-090
Shared connection definitions podrán reutilizarse entre requests.

## DB-RWC-091
Scoped leases no se reutilizarán como current lease entre requests.

## DB-RWC-092
Sticky write observations serán scoped.

## DB-RWC-093
Connection affinity será scoped.

## DB-RWC-094
Transaction affinity será más fuerte que role affinity.

## DB-RWC-095
Sticky writer affinity no requerirá necesariamente misma physical connection.

## DB-RWC-096
Driver concurrency capabilities serán explícitas.

## DB-RWC-097
Una physical connection no se asumirá coroutine-safe.

## DB-RWC-098
Capability checks reemplazarán vendor conditionals en capas superiores.

## DB-RWC-099
Read-only transaction support será capability-driven.

## DB-RWC-100
Read-only session support será capability-driven.

## DB-RWC-101
Multitenancy será integración opcional.

## DB-RWC-102
Core Read/Write System no dependerá de Tenant classes.

## DB-RWC-103
Execution domain podrá transportar tenant/shard mediante contratos genéricos.

## DB-RWC-104
Routing nunca cambiará DatabaseIdentity silenciosamente.

## DB-RWC-105
No existirá cross-database fallback implícito.

## DB-RWC-106
Shard deberá resolverse antes de role selection cuando sharding aplique.

## DB-RWC-107
Partition deberá resolverse antes de role selection cuando aplique.

## DB-RWC-108
Connection policy podrá ser más restrictiva que endpoint capability.

## DB-RWC-109
Logical read-only mode rechazará WRITE aunque el endpoint pueda escribir.

## DB-RWC-110
User authorization será distinta de connection role authorization.

## DB-RWC-111
HTTP input no controlará ConnectionIntent directamente.

## DB-RWC-112
ConnectionIntent procederá de trusted internal semantics.

## DB-RWC-113
Raw developer APIs podrán declarar intent explícitamente.

## DB-RWC-114
Telemetry no expondrá credentials.

## DB-RWC-115
Telemetry no expondrá bound values por default.

## DB-RWC-116
Metrics evitarán high-cardinality endpoint identifiers por default.

## DB-RWC-117
Role resolution deberá ser explicable.

## DB-RWC-118
Diagnostics distinguirán requested intent y resolved role.

## DB-RWC-119
Diagnostics indicarán transaction pinning.

## DB-RWC-120
Diagnostics no serán fuente autoritativa de topology.

## DB-RWC-121
Configuration inheritance será determinista.

## DB-RWC-122
Endpoint overrides no mutarán base definition.

## DB-RWC-123
Duplicate EndpointId será inválido.

## DB-RWC-124
Contradictory role policy fallará durante bootstrap.

## DB-RWC-125
No eligible writer producirá error explícito.

## DB-RWC-126
No eligible reader podrá usar writer únicamente si fallback lo permite.

## DB-RWC-127
Connection candidates deberán pertenecer al logical database correcto.

## DB-RWC-128
Connection candidate selection no podrá cruzar execution domains.

## DB-RWC-129
Transaction pinned lease deberá pertenecer al mismo execution domain.

## DB-RWC-130
Role resolution no alterará tenant context.

## DB-RWC-131
Role resolution no alterará shard context.

## DB-RWC-132
Role resolution no alterará database name.

## DB-RWC-133
Read/write roles serán extensibles mediante capabilities, no vendor names.

## DB-RWC-134
MariaDB y MySQL podrán compartir conceptos sin asumir topología idéntica.

## DB-RWC-135
PostgreSQL topology será adaptada mediante platform/provider integration.

## DB-RWC-136
SQLite single-file deployments podrán representarse naturalmente como READ_WRITE.

## DB-RWC-137
Single-node deployments no requerirán configuración duplicada.

## DB-RWC-138
Distributed deployments no cambiarán la API normal del Query Builder.

## DB-RWC-139
Distributed deployments no cambiarán la API normal del ORM.

## DB-RWC-140
Application code no deberá conocer host names para operaciones normales.

## DB-RWC-141
Connection role será parte de execution infrastructure, no domain model.

## DB-RWC-142
Connection role no será persistido dentro de entities.

## DB-RWC-143
Connection selection no será responsabilidad del Hydrator.

## DB-RWC-144
Connection selection no será responsabilidad del SQL Compiler.

## DB-RWC-145
SQL Compiler no conocerá replicas.

## DB-RWC-146
Query Builder no conocerá physical endpoints.

## DB-RWC-147
ORM Repository no conocerá physical endpoints.

## DB-RWC-148
Driver no decidirá business-level routing policy.

## DB-RWC-149
Connection Manager seguirá siendo autoridad de adquisición/release.

## DB-RWC-150
Read/Write System producirá requisitos/candidatos, no reemplazará Connection Manager.

## DB-RWC-151
ConnectionRoleResolution será immutable.

## DB-RWC-152
Candidate sets serán bounded.

## DB-RWC-153
Runtime routing state tendrá lifecycle explícito.

## DB-RWC-154
Scope cleanup eliminará affinities temporales.

## DB-RWC-155
Scope cleanup eliminará write observations temporales.

## DB-RWC-156
Scope cleanup liberará leases no transferidas.

## DB-RWC-157
Worker reuse no cambiará semantics entre requests.

## DB-RWC-158
Unknown role nunca será tratado como WRITE por conveniencia.

## DB-RWC-159
Unknown query intent nunca será tratado como READ por conveniencia.

## DB-RWC-160
Correctness tendrá precedencia sobre distribución de carga.

---

# 223. Flujo general

```text
Application / ORM / Query Engine
              │
              ▼
       Query Semantics
              │
              ▼
     ConnectionRequirement
              │
              ▼
     Execution Domain
              │
              ▼
Logical Database Connection
              │
              ▼
   Active Transaction?
       │             │
      YES            NO
       │             │
       ▼             ▼
Pinned Lease    Role Resolver
                     │
                     ▼
              Eligible Roles
                     │
                     ▼
             Candidate Endpoints
                     │
                     ▼
           Routing / Selection
                     │
                     ▼
              Connection Pool
                     │
                     ▼
             Connection Lease
                     │
                     ▼
               Query Executor
```

---

# 224. Single-node architecture

```text
Application
    │
    ▼
ConnectionRequirement
    │
    ├── READ
    └── WRITE
         │
         ▼
   READ_WRITE Endpoint
         │
         ▼
      DB Server
```

VoltStack deberá conservar prácticamente el mismo coste conceptual que una conexión tradicional.

---

# 225. Primary/read topology

```text
                        ┌─────────────┐
                 ┌─────►│ Reader A    │
                 │      └─────────────┘
READ ────────────┤
                 │      ┌─────────────┐
                 └─────►│ Reader B    │
                        └─────────────┘

WRITE ─────────────────►┌─────────────┐
                        │ Writer      │
                        └─────────────┘
```

Sin que la aplicación tenga que conocer esos nodos.

---

# 226. Transaction architecture

```text
DB::transaction(...)
        │
        ▼
WRITE-capable requirement
        │
        ▼
Writer candidate
        │
        ▼
Connection Lease
        │
        ▼
BEGIN
        │
        ▼
TransactionContext
        │
        ▼
PinnedConnectionLease
        │
        ├── SELECT
        ├── UPDATE
        ├── INSERT
        └── SELECT
        │
        ▼
COMMIT / ROLLBACK
```

---

# 227. Read-after-write architecture

```text
WRITE
  │
  ▼
Writer
  │
  ▼
COMMIT
  │
  ▼
Write Observed
  │
  ▼
Subsequent READ
  │
  ▼
Consistency Policy
  │
  ├── eventual ──────► reader eligible
  │
  └── session/auth ──► writer eligible
```

La implementación completa se desarrollará posteriormente.

---

# 228. Dependency boundaries

```text
ORM
 │
 ▼
Query Engine
 │
 ▼
Execution Engine
 │
 ▼
Read/Write Connection System
 │
 ▼
Connection Manager
 │
 ▼
Connection Pool
 │
 ▼
Driver
```

Nunca:

```text
Driver
→ ORM
```

ni:

```text
SQL Compiler
→ Replica Selector
```

---

# 229. Fórmula arquitectónica

Podemos representar una resolución como:

```text
R =
ResolveConnection(
    D,
    I,
    T,
    C,
    P
)
```

donde:

```text
D = Execution Domain
I = Connection Intent
T = Transaction Context
C = Connection Capabilities
P = Read/Write Policy
```

Si existe transacción activa:

```text
T.active = true
⇒
R = T.pinnedConnection
```

siempre que:

```text
Capabilities(R)
⊇
Requirements(I)
```

---

# 230. Regla de compatibilidad

```text
Eligible(E, R)
=
DomainCompatible(E)
∧
RoleCompatible(E, R)
∧
PolicyAllows(E, R)
```

Posteriormente el Routing System añadirá:

```text
Health
Lag
Load
Affinity
Topology
```

---

# 231. Regla maestra

> **El Read/Write Connection System de VoltStack no decidirá “qué servidor ejecutar” observando simplemente si una consulta parece un `SELECT`; representará explícitamente el dominio de ejecución, la intención semántica, las capacidades de conexión y las restricciones transaccionales antes de producir el conjunto de conexiones elegibles.**

En forma compacta:

```text
Operation
→ Intent
→ Required Capability
→ Eligible Connection Role
→ Candidate Set
```

y no:

```text
SQL starts with SELECT
→ replica
```

---

# 232. Resultado arquitectónico

Con este sistema VoltStack podrá evolucionar progresivamente desde:

```text
Application
    │
    ▼
Single Database
```

hacia:

```text
Application
    │
    ▼
Logical Database
    │
    ▼
Read/Write Roles
    │
    ▼
Routing
    │
    ├── Primary/Writer
    ├── Replicas
    ├── Failover
    ├── Load Balancing
    ├── Shards
    └── Partitions
```

sin modificar la arquitectura fundamental del:

```text
Query Builder
ORM
Persistence Engine
SQL Compiler
```

---

# 233. Siguiente documento

```text
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
```

El siguiente documento utilizará los conceptos definidos aquí:

```text
ConnectionIntent
ConnectionRole
ConnectionRequirement
ReadWriteConnectionGroup
ConnectionCandidateSet
ConnectionAffinity
ReadConsistencyRequirement
```

para diseñar el motor que decidirá:

```text
Eligible Candidate Set
          │
          ▼
Read/Write Routing System
          │
          ├── policy
          ├── transaction affinity
          ├── consistency
          ├── health
          ├── fallback
          └── routing constraints
          │
          ▼
Selected Connection Endpoint
```

estableciendo una separación esencial:

```text
Connection System
=
what connections exist
and what they can do
```

mientras:

```text
Routing System
=
which eligible connection
should be used now
```