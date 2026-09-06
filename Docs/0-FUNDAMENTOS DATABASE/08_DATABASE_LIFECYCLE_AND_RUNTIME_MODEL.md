# 08_DATABASE_LIFECYCLE_AND_RUNTIME_MODEL.md

# VoltStack Quantum Database
## Lifecycle and Runtime Model

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 08 — Database Lifecycle and Runtime Model  
**Estado:** Architecture Specification  
**Nivel:** Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el modelo oficial de lifecycle y runtime de:

```text
VoltStack/Quantum/Database
```

Su objetivo es establecer formalmente:

- cuánto vive cada componente;
- quién posee cada estado mutable;
- cuándo comienza y termina un contexto Database;
- cómo se adquieren y liberan conexiones;
- cómo viven `EntityManager`, `UnitOfWork` e `IdentityMap`;
- cómo se administran transacciones;
- cómo se limpian cursores y streams;
- cómo se recupera el sistema ante errores;
- cómo se evita fuga de estado entre ejecuciones;
- cómo funciona Database bajo workers persistentes;
- cómo se soportará concurrencia futura;
- cómo se integrará con FrankenPHP;
- cómo se adaptará posteriormente a RoadRunner y OpenSwoole.

La seguridad del lifecycle será una propiedad arquitectónica de primer nivel.

---

# 2. Problema fundamental

El modelo tradicional de muchas aplicaciones PHP asume:

```text
Request
   │
   ▼
PHP Process
   │
   ▼
Application
   │
   ▼
Database State
   │
   ▼
Process Ends
```

Cuando el proceso termina, PHP elimina implícitamente:

```text
objects
static state
connections
EntityManager
IdentityMap
UnitOfWork
transaction state
temporary context
```

Este supuesto deja de ser válido con servidores persistentes.

---

# 3. Modelo persistent worker

Con FrankenPHP:

```text
Worker Start
     │
     ▼
Application Boot
     │
     ▼
Request A
     │
     ▼
Request B
     │
     ▼
Request C
     │
     ▼
...
     │
     ▼
Request N
```

El proceso puede continuar vivo.

Por tanto:

> El final del request ya no implica el final del estado PHP.

---

# 4. Riesgo principal

Sin lifecycle explícito:

```text
Request A
   │
   ▼
EntityManager
   │
   ├── User #15
   ├── Tenant A
   └── Transaction
          │
          ▼
      Worker survives
          │
          ▼
Request B
```

Request B podría observar accidentalmente estado perteneciente a Request A.

Esto constituye una violación crítica de aislamiento.

---

# 5. Principio maestro

VoltStack Database aplicará:

> Todo estado mutable deberá tener un propietario y un lifetime explícitos.

Nunca deberá existir estado cuya duración sea:

```text
"hasta que PHP lo destruya"
```

como estrategia arquitectónica.

---

# 6. Segundo principio

El proceso y el request son conceptos diferentes.

```text
Process Lifetime
       │
       ├── Worker Lifetime
       │
       │      ├── Execution Scope A
       │      ├── Execution Scope B
       │      ├── Execution Scope C
       │      └── ...
       │
       └── Persistent Infrastructure
```

---

# 7. Runtime hierarchy

La jerarquía conceptual será:

```text
Process
   │
   ▼
Worker
   │
   ▼
Execution Scope
   │
   ▼
Database Context
   │
   ├── Persistence Context
   ├── Transaction Context
   ├── Connection Scope
   └── Operation Contexts
```

---

# 8. Niveles de lifetime

Database reconocerá conceptualmente:

```text
Process Lifetime
Worker Lifetime
Execution Scope Lifetime
Transaction Lifetime
Operation Lifetime
Resource Lifetime
```

Cada componente deberá pertenecer claramente a uno.

---

# 9. Process Lifetime

Puede abarcar toda la ejecución del proceso PHP.

Ejemplos adecuados:

```text
immutable configuration
compiled service graph
static descriptors
compiled metadata
```

No deberá contener estado de usuario/request.

---

# 10. Worker Lifetime

Un worker puede procesar múltiples ejecuciones.

Puede mantener:

```text
connection pool
compiled metadata
registries
dialects
platform descriptors
compiler infrastructure
optimization rules
```

si son seguros para reutilización.

---

# 11. Execution Scope Lifetime

Representa una unidad lógica de trabajo de aplicación.

Ejemplos:

```text
HTTP request
queue job
CLI operation
scheduled task
message consumption
RPC invocation
WebSocket message/operation
```

según el runtime.

---

# 12. ExecutionScope

Conceptualmente:

```php
interface ExecutionScopeInterface
{
    public function id(): ExecutionScopeId;

    public function isActive(): bool;
}
```

La implementación general puede pertenecer al Runtime/Platform de VoltStack.

Database consume el concepto.

---

# 13. DatabaseContext

Cada Execution Scope podrá poseer un:

```text
DatabaseContext
```

que representa el estado Database correspondiente a esa ejecución.

Conceptualmente:

```php
final class DatabaseContext
{
    public function __construct(
        private ConnectionScope $connections,
        private PersistenceContext $persistence,
        private TransactionContext $transactions,
    ) {}
}
```

La estructura definitiva podrá ser lazy.

---

# 14. Creación lazy del DatabaseContext

Un request que no utiliza Database no debería pagar el coste completo.

```text
Request
   │
   ├── no DB operation
   │       │
   │       └── no DatabaseContext required
   │
   └── first DB operation
           │
           ▼
      DatabaseContext created
```

---

# 15. Context identity

Cada DatabaseContext deberá estar asociado a un único Execution Scope.

```text
Scope A
   │
   └── DatabaseContext A

Scope B
   │
   └── DatabaseContext B
```

Nunca:

```text
Scope A ──┐
          ├── DatabaseContext X
Scope B ──┘
```

para estado mutable.

---

# 16. DatabaseContext state

Puede contener o coordinar:

```text
connection leases
current transaction hierarchy
PersistenceContext
EntityManager state
UnitOfWork
IdentityMap
hydration state
tenant DB selection
sticky read state
resource registry
cleanup state
```

No necesariamente todos serán creados inmediatamente.

---

# 17. Context lifecycle states

Un DatabaseContext deberá tener estados explícitos.

Ejemplo:

```text
Created
   │
   ▼
Active
   │
   ▼
Closing
   │
   ▼
Closed
```

En caso de fallo:

```text
Active
   │
   ▼
Failed
   │
   ▼
Closing
   │
   ▼
Closed
```

---

# 18. Estado Created

El contexto existe pero todavía puede no haber adquirido recursos.

```text
Created
├── no physical connection
├── no transaction
├── empty IdentityMap
└── empty UnitOfWork
```

---

# 19. Estado Active

Permite operaciones Database.

```text
Active
   │
   ├── query
   ├── transaction
   ├── persistence
   └── hydration
```

---

# 20. Estado Closing

No deberán aceptarse nuevas operaciones ordinarias.

El sistema está realizando:

```text
cursor cleanup
transaction cleanup
persistence cleanup
connection reset
resource release
```

---

# 21. Estado Closed

Cualquier intento de reutilizar el contexto deberá fallar.

Ejemplo:

```text
DatabaseContextClosedException
```

No deberá reactivarse.

---

# 22. Scope generation

Para ayudar a detectar referencias antiguas podrá existir:

```text
ExecutionScopeId
```

o:

```text
ContextGeneration
```

Ejemplo:

```text
worker-3 / scope-9812
```

No necesita exponerse al usuario.

---

# 23. Stale context detection

Si un componente intenta utilizar:

```text
DatabaseContext A
```

durante:

```text
Scope B
```

deberá ser posible detectarlo en strict/debug mode.

---

# 24. PersistenceContext

El estado ORM mutable estará encapsulado en:

```text
PersistenceContext
```

que podrá poseer:

```text
EntityManager
UnitOfWork
IdentityMap
ChangeTracker state
persistence operation state
```

---

# 25. PersistenceContext lifetime

Regla:

```text
PersistenceContext
→ Execution Scope
```

salvo scopes internos explícitos de procesamiento batch.

Nunca:

```text
PersistenceContext
→ Worker
```

---

# 26. EntityManager lifecycle

El `EntityManager` deberá pertenecer al PersistenceContext.

```text
Execution Scope
      │
      ▼
PersistenceContext
      │
      ▼
EntityManager
```

Su estado no sobrevivirá al scope.

---

# 27. EntityManager creation

Preferentemente lazy:

```text
Request
  │
  ▼
uses only Query Builder
  │
  └── EntityManager not created
```

Sólo cuando ORM sea utilizado:

```text
ORM operation
    │
    ▼
EntityManager created/resolved
```

---

# 28. EntityManager state

Puede coordinar:

```text
managed entities
repositories
UnitOfWork
IdentityMap
flush operations
entity lifecycle
```

No deberá almacenar conexiones físicas como propiedad permanente innecesaria.

---

# 29. EntityManager close

Al finalizar el PersistenceContext:

```text
EntityManager
     │
     ▼
closed
```

Después:

```php
$entityManager->persist($entity);
```

deberá fallar si la instancia pertenece a un scope terminado.

---

# 30. EntityManager clear

Debe distinguirse:

```text
clear()
```

de:

```text
close()
```

`clear()` elimina estado administrado pero mantiene el EntityManager utilizable.

`close()` termina su lifecycle.

---

# 31. UnitOfWork lifecycle

```text
UnitOfWork
→ PersistenceContext
```

Al inicio:

```text
new entities: 0
dirty entities: 0
removed entities: 0
relationship changes: 0
```

---

# 32. UnitOfWork state

Podrá registrar:

```text
new
managed
dirty
removed
relation changes
pending persistence operations
```

Este estado es estrictamente scoped.

---

# 33. Dirty UnitOfWork at scope end

Si existen cambios no flushed:

```text
Scope Ending
     │
     ▼
Dirty UnitOfWork
```

Database no hará:

```text
automatic flush
```

por defecto.

---

# 34. Dirty UnitOfWork policy

Development:

```text
diagnostic/warning
```

Production:

```text
discard state
```

con telemetría opcional.

Puede existir strict mode:

```text
DirtyUnitOfWorkAtScopeEnd
```

---

# 35. No implicit persistence

Regla:

> El cleanup elimina estado; no inventa intención de persistencia.

Por tanto:

```text
scope end != flush()
```

---

# 36. IdentityMap lifecycle

```text
IdentityMap
→ PersistenceContext
```

Ejemplo:

```text
Scope A:
User#10 → Object A

Scope B:
User#10 → Object B
```

No existe garantía de identidad de objeto entre scopes.

---

# 37. IdentityMap cleanup

Al finalizar:

```text
IdentityMap::clear()
```

deberá eliminar todas las referencias administradas.

Esto es importante para:

```text
correctness
memory
tenant isolation
```

---

# 38. IdentityMap and long-running processing

Procesos batch podrán limpiar periódicamente:

```text
flush
  │
  ▼
clear
  │
  ▼
continue
```

para limitar memoria.

---

# 39. ChangeTracker lifecycle

El estado temporal de snapshots/change sets deberá estar ligado al PersistenceContext.

No deberá sobrevivir independientemente del UnitOfWork.

---

# 40. HydrationContext

Una hidratación puede requerir estado temporal:

```text
row position
partial entity state
relationship assembly
deduplication
hydration plan
```

Ese estado deberá ser:

```text
operation-scoped
```

o limitado a una operación de streaming.

---

# 41. QueryContext

Cada consulta estructurada podrá generar:

```text
QueryContext
```

para:

```text
semantic resolution
optimization
planning
compilation metadata
query tags
```

No deberá almacenarse globalmente.

---

# 42. QueryContext lifetime

```text
Query operation
     │
     ▼
QueryContext
     │
     ▼
disposed
```

Una vez compilada/ejecutada la operación, el contexto temporal puede descartarse.

---

# 43. ExecutionContext

La ejecución de un statement podrá usar:

```text
ExecutionContext
```

con:

```text
timeout
cancellation
query intent
retry state
telemetry context
connection role
```

---

# 44. Context separation

Regla crítica:

```text
QueryContext
      !=
ExecutionContext
      !=
TransactionContext
      !=
PersistenceContext
      !=
DatabaseContext
```

Cada uno representa responsabilidades diferentes.

---

# 45. TransactionContext

El estado transaccional pertenece al Execution Scope.

Puede contener:

```text
transaction depth
active connection lease
savepoint stack
isolation level
retry attempt
rollback-only state
```

---

# 46. Transaction lifecycle

```text
None
 │
 ▼
Beginning
 │
 ▼
Active
 │
 ├── Commit ─────► Committed
 │
 └── Rollback ───► RolledBack
```

Fallos:

```text
Active
   │
   ▼
Failed
   │
   ▼
Rollback
```

---

# 47. Transaction ownership

Una transacción deberá tener un propietario lógico.

Ejemplo:

```text
TransactionManager
      │
      ▼
TransactionHandle
      │
      ▼
TransactionContext
```

El Connection no deberá convertirse en propietario del lifecycle de aplicación.

---

# 48. Transaction and connection

Una transacción estará asociada a una conexión física o lease apropiado.

```text
Transaction
     │
     ▼
Connection Lease
     │
     ▼
Physical Connection
```

Ese lease no podrá cambiar arbitrariamente durante la transacción.

---

# 49. Transaction connection pinning

Una vez iniciada:

```text
transaction
```

las operaciones compatibles deberán mantenerse en el mismo target/conexión.

Especialmente con:

```text
read/write splitting
replicas
sharding
```

---

# 50. Commit

Commit exitoso:

```text
Active Transaction
        │
        ▼
COMMIT
        │
        ▼
Transaction State Finalized
        │
        ▼
Lease eligible for reset/release
```

---

# 51. Rollback

Rollback:

```text
Active Transaction
        │
        ▼
ROLLBACK
        │
        ▼
Transaction State Finalized
```

Si rollback falla, la conexión deberá considerarse sospechosa o poisoned.

---

# 52. Scope end with active transaction

Regla absoluta:

```text
Scope End
   +
Active Transaction
        │
        ▼
ROLLBACK
```

Nunca:

```text
implicit commit
```

---

# 53. Rollback-only state

Una transacción podrá marcarse:

```text
rollback-only
```

cuando un error hace inseguro continuar.

Un intento de commit deberá fallar.

---

# 54. Nested transaction lifecycle

Podrá representarse:

```text
Transaction Level 1
     │
     ├── Savepoint Level 2
     │       │
     │       └── Savepoint Level 3
     │
     └── Commit/Rollback
```

según capabilities.

---

# 55. Savepoint ownership

Los savepoints pertenecen al TransactionContext.

No a un estado global del ConnectionManager.

---

# 56. Retry transaction lifecycle

Un retry completo deberá crear un nuevo intento lógico:

```text
Attempt 1
   │
   └── serialization failure
          │
          ▼
       rollback
          │
          ▼
Attempt 2
```

El estado del intento anterior deberá descartarse.

---

# 57. ORM state after transaction rollback

Un rollback DB puede invalidar supuestos del ORM.

Por ello:

```text
DB rollback
```

no significa necesariamente:

```text
UnitOfWork remains safely reusable
```

La política ORM deberá poder:

```text
clear
invalidate
refresh
```

según el tipo de operación.

---

# 58. Connection lifecycle

Una conexión física tendrá lifecycle independiente del request.

```text
Created
  │
  ▼
Connecting
  │
  ▼
Ready
  │
  ▼
Leased
  │
  ▼
Resetting
  │
  ├── Ready
  └── Discarded
```

---

# 59. Connection creation

Ocurrirá bajo demanda:

```text
ConnectionManager
       │
       ▼
ConnectionFactory
       │
       ▼
Driver
       │
       ▼
Physical Connection
```

---

# 60. Connection acquisition

El consumidor solicita una conexión lógica:

```text
connection("primary")
```

El sistema resuelve:

```text
logical connection
        │
        ▼
topology
        │
        ▼
target
        │
        ▼
pool/factory
        │
        ▼
lease
```

---

# 61. ConnectionScope

El Execution Scope deberá rastrear leases adquiridos.

```text
ConnectionScope
├── primary/write lease
├── primary/read lease
└── analytics lease
```

según necesidad.

---

# 62. Connection reuse inside scope

Cuando sea semánticamente seguro:

```text
Query A
   │
   ▼
Connection Lease X

Query B
   │
   ▼
Connection Lease X
```

puede reutilizarse.

No es obligatorio adquirir una conexión por query.

---

# 63. Cross-scope connection reuse

Una conexión física podrá reutilizarse mediante pool:

```text
Scope A
  │
  ▼
Lease X
  │
  ▼
RESET
  │
  ▼
Pool
  │
  ▼
Scope B
```

pero el lease y estado lógico anterior no sobreviven.

---

# 64. Connection reset

Antes de reutilizar una conexión deberá restaurarse el estado necesario.

Puede incluir:

```text
rollback active transaction
close cursors
clear session-local changes
restore session defaults
clear tenant/schema state
reset temporary settings
reset driver-specific state
```

---

# 65. Reset guarantee

Regla:

> Una conexión sólo podrá volver al pool cuando Database pueda garantizar que es segura para otra ejecución.

Si no:

```text
discard connection
```

---

# 66. Connection poisoning

Motivos:

```text
failed rollback
failed protocol synchronization
network interruption
unknown transaction state
failed reset
unexpected session mutation
driver-reported fatal error
```

Resultado:

```text
Connection
   │
   ▼
POISONED
   │
   ▼
discard
```

---

# 67. Connection state categories

Se deberá distinguir:

```text
portable framework-managed state
driver-native state
server session state
transaction state
tenant state
user-created raw state
```

Esto influirá en reset safety.

---

# 68. Raw SQL and lifecycle

Raw SQL puede modificar estado de sesión:

```sql
SET ...
```

Por tanto, escape hatches pueden afectar la capacidad de reutilizar conexiones.

VoltStack deberá documentar y, cuando sea posible, detectar estas situaciones.

---

# 69. Unsafe native handle usage

Si el usuario obtiene acceso al handle nativo:

```text
PDO/native driver connection
```

Database puede perder visibilidad sobre cambios de estado.

Podrá existir una API como:

```text
connection marked dirty/unknown
```

para forzar reset fuerte o descarte.

---

# 70. Connection lease ownership

Un lease deberá tener propietario explícito.

Ejemplos:

```text
Execution Scope
Transaction
Streaming Cursor
```

La transferencia de ownership deberá ser controlada.

---

# 71. Cursor lifecycle

Un cursor podrá mantener una conexión ocupada.

```text
Query
  │
  ▼
Cursor
  │
  ▼
Connection Lease
```

Mientras el cursor esté abierto, el lease puede no ser liberable.

---

# 72. Cursor states

```text
Created
  │
  ▼
Open
  │
  ├── Exhausted
  └── Closed
```

Un cursor agotado deberá finalizar correctamente sus recursos.

---

# 73. Cursor cleanup

Al terminar scope:

```text
Open Cursor
    │
    ▼
close()
    │
    ▼
release dependent resources
```

---

# 74. Cursor ownership registry

`DatabaseContext` o `ConnectionScope` podrá mantener un:

```text
ResourceRegistry
```

con referencias a recursos que requieren cleanup.

---

# 75. ResourceRegistry

Podrá rastrear:

```text
cursors
streams
statements
connection leases
temporary resources
```

No deberá convertirse en registry universal de servicios.

---

# 76. Resource registration

Cuando un recurso de larga vida se crea:

```text
Resource
   │
   ▼
register ownership
```

Cuando se cierra:

```text
Resource
   │
   ▼
unregister
```

---

# 77. Streaming lifecycle

Un resultado streaming extiende el lifetime de ejecución de recursos.

```text
stream()
  │
  ▼
Cursor
  │
  ▼
Lease retained
  │
  ▼
iteration complete
  │
  ▼
close/release
```

---

# 78. Early stream termination

Si el consumidor rompe la iteración:

```php
foreach ($rows as $row) {
    break;
}
```

el recurso deberá poder cerrarse explícita o automáticamente mediante lifecycle seguro.

---

# 79. Garbage collection is not lifecycle

VoltStack no deberá depender de:

```text
PHP GC
```

para liberar recursos críticos.

GC puede ser fallback, no contrato principal.

---

# 80. Destructors

`__destruct()` podrá actuar como última defensa.

Nunca será el mecanismo primario para:

```text
transaction rollback
pool release
cursor cleanup
scope reset
```

---

# 81. Statement lifecycle

Un prepared statement podrá ser:

```text
connection-bound
```

y por tanto su lifetime deberá respetar el Connection.

No deberá sobrevivir a una conexión descartada.

---

# 82. Prepared statement reuse

Si el driver permite reuse seguro:

```text
Physical Connection
     │
     └── Prepared Statement Cache
```

puede existir.

Este cache pertenece a la conexión/pool infrastructure, no al request.

---

# 83. Statement cache invalidation

Debe invalidarse cuando:

```text
connection discarded
schema state invalidates statement
driver reports invalidation
```

según capacidades.

---

# 84. Read/write lifecycle

Fuera de transacción:

```text
Read
  │
  ▼
Read Target

Write
  │
  ▼
Write Target
```

El scope puede mantener sticky state después de un write.

---

# 85. Sticky read state

```text
Write occurs
    │
    ▼
ConnectionScope
    │
    ▼
sticky-to-primary = true
```

Este estado es:

```text
Execution Scope
```

Nunca worker-global.

---

# 86. Sticky state cleanup

Al terminar scope:

```text
sticky state
→ destroyed
```

Request B no hereda que Request A realizó un write.

---

# 87. Replica lifecycle

Una replica puede ser marcada temporalmente unhealthy por infraestructura persistente.

Eso es distinto de request state.

Ejemplo:

```text
ReplicaHealthState
→ worker/distributed infrastructure
```

mientras:

```text
chosen replica for operation
→ operation state
```

---

# 88. Tenant database state

Si Multitenancy está instalado:

```text
TenantDatabaseContext
```

deberá pertenecer al Execution Scope.

Nunca worker-global.

---

# 89. Tenant selection

```text
Scope
  │
  ▼
Tenant Resolver
  │
  ▼
TenantDatabaseContext
  │
  ▼
Connection Resolution
```

---

# 90. Tenant state reset

Al finalizar:

```text
tenant ID
tenant database
tenant schema
tenant connection aliases
tenant-specific sticky state
```

deberán desaparecer.

---

# 91. Tenant leak severity

Una fuga de tenant state será considerada:

```text
critical security failure
```

porque puede causar acceso cross-tenant.

---

# 92. Context propagation

Database deberá recibir el contexto actual mediante mecanismos oficiales de VoltStack Runtime.

No mediante:

```text
global variables
unscoped static properties
process-wide mutable singleton
```

---

# 93. Context-local storage

Si Platform proporciona:

```text
ContextLocal<T>
```

Database podrá utilizarlo para bridges que necesiten resolver contexto actual.

Especialmente APIs estáticas de DX.

---

# 94. Static API bridge

Ejemplo:

```php
User::query();
```

podrá conceptualmente resolver:

```text
Model API
   │
   ▼
Current PersistenceContext
```

mediante un bridge context-aware.

Nunca:

```text
static EntityManager singleton
```

---

# 95. DB facade

Igualmente:

```php
DB::transaction(...)
```

deberá operar sobre el Execution Scope actual cuando exista estado scoped.

La facade no será propietaria del estado.

---

# 96. Context propagation across callbacks

Ejemplo:

```php
DB::transaction(function () {
    // same Execution Scope
    // transaction context active
});
```

El callback deberá observar el contexto transaccional correcto.

---

# 97. Async context propagation

En una futura API async:

```text
Execution Scope
      │
      ▼
Async Operation
```

el contexto deberá propagarse explícitamente según Runtime.

No deberá asumirse que una variable global identifica correctamente la operación.

---

# 98. Concurrency model

La arquitectura deberá ser segura para:

```text
sequential persistent requests
concurrent coroutines
future async operations
parallel tasks where supported
```

---

# 99. No process-global current context

Prohibido:

```php
DatabaseRuntime::$currentContext = $context;
```

si múltiples ejecuciones concurrentes pueden compartir proceso.

---

# 100. Context isolation

Debe cumplirse:

```text
Context A state ∩ Context B state = ∅
```

para estado mutable scoped.

Pueden compartir únicamente infraestructura segura e inmutable.

---

# 101. Shared immutable infrastructure

Puede compartirse:

```text
CompiledMetadata
Dialect
Platform Capability Definitions
Compiler Rules
Configuration
Type Definitions
```

---

# 102. Shared mutable infrastructure

Sólo deberá compartirse cuando tenga sincronización y semántica explícita.

Ejemplos:

```text
connection pool
health registry
metrics aggregator
```

No por conveniencia.

---

# 103. Concurrency-safe registries

Registries congelados son preferibles porque permiten:

```text
lock-free reads
deterministic behavior
safe sharing
```

durante runtime.

---

# 104. FrankenPHP runtime model

FrankenPHP será la implementación de referencia inicial.

```text
FrankenPHP Worker
       │
       ▼
VoltStack Application
       │
       ├── Persistent Database Infrastructure
       │
       ├── Request Scope A
       │
       ├── Request Scope B
       │
       └── Request Scope N
```

---

# 105. FrankenPHP worker start

Secuencia:

```text
Worker Start
    │
    ▼
Load Compiled Container
    │
    ▼
Load Database Configuration
    │
    ▼
Initialize Persistent Services
    │
    ▼
Worker Ready
```

No se requiere abrir Database inmediatamente.

---

# 106. FrankenPHP request start

```text
Request Accepted
      │
      ▼
VoltStack Execution Scope Begins
      │
      ▼
Database Runtime Adapter notified
      │
      ▼
DatabaseContext available lazily
```

---

# 107. FrankenPHP request execution

Durante aplicación:

```text
Controller
   │
   ▼
Service
   │
   ▼
Repository / DB API
   │
   ▼
DatabaseContext
   │
   ▼
Database infrastructure
```

---

# 108. FrankenPHP request termination

```text
Application completes
       │
       ▼
Database enters Closing
       │
       ▼
Resource cleanup
       │
       ▼
Transaction safety
       │
       ▼
Persistence cleanup
       │
       ▼
Connection reset/release
       │
       ▼
DatabaseContext Closed
       │
       ▼
Container Scope destroyed
```

---

# 109. Request exception lifecycle

Si aplicación lanza excepción:

```text
ApplicationException
       │
       ▼
Scope still enters cleanup
```

No deberá saltarse Database cleanup.

---

# 110. Cleanup under exception

```text
Exception
   │
   ▼
stop ordinary operations
   │
   ▼
close resources
   │
   ▼
rollback transaction
   │
   ▼
discard ORM state
   │
   ▼
reset/release connection
   │
   ▼
propagate original exception
```

---

# 111. Cleanup ordering

Orden base recomendado:

```text
1. Mark DatabaseContext Closing
2. Prevent new ordinary operations
3. Cancel/finish pending DB operations
4. Close streams
5. Close cursors
6. Finalize statements
7. Detect active transactions
8. Rollback orphan transactions
9. Inspect dirty UnitOfWork
10. Clear UnitOfWork
11. Clear IdentityMap
12. Clear hydration state
13. Clear operation contexts
14. Reset connection session state
15. Release/discard connection leases
16. Clear tenant DB state
17. Clear sticky routing state
18. Close EntityManager
19. Mark DatabaseContext Closed
```

El orden concreto podrá refinarse.

---

# 112. Cleanup idempotency

Idealmente:

```text
cleanup()
cleanup()
```

no deberá duplicar efectos peligrosos.

La segunda llamada podrá ser no-op controlado.

---

# 113. Cleanup state machine

```text
Active
  │
  ▼
Closing
  │
  ▼
Closed
```

Si otro código intenta cerrar durante `Closing`, deberá coordinarse y no ejecutar dos pipelines simultáneos.

---

# 114. Cleanup failure

Si una etapa falla:

```text
Rollback failed
```

el pipeline deberá continuar con acciones defensivas compatibles:

```text
mark connection poisoned
discard connection
clear ORM state
close context
```

---

# 115. CleanupReport

Podrá generarse:

```text
DatabaseCleanupReport
```

con:

```text
closed cursors
rolled back transactions
discarded connections
dirty UnitOfWork detected
cleanup errors
worker recycle recommendation
```

---

# 116. Cleanup errors

Podrán clasificarse:

```text
Recoverable
ConnectionDiscardRequired
ContextInvalid
WorkerRecycleRecommended
FatalRuntimeFailure
```

---

# 117. Error preservation

Si existen:

```text
Original Application Error
+
Cleanup Error
```

la primera no deberá perderse.

La segunda deberá adjuntarse como:

```text
suppressed/secondary failure
telemetry
cleanup report
```

según modelo de exceptions de VoltStack.

---

# 118. Worker recycle

Database podrá solicitar:

```text
WorkerRecycleRecommendation
```

cuando ya no confíe en el estado del worker.

Ejemplos:

```text
unrecoverable context corruption
pool corruption
repeated cleanup failures
memory leak threshold
runtime invariant violation
```

---

# 119. Database does not kill workers

Database no deberá ejecutar:

```text
exit()
kill process
```

para manejar esto.

Emitirá una señal/resultado hacia RuntimeManagerServer.

---

# 120. RuntimeManagerServer responsibility

```text
Database
    │
    ▼
WorkerRecycleRecommendation
    │
    ▼
RuntimeManagerServer
    │
    ▼
FrankenPHP adapter
```

El Runtime Manager decide cómo reciclar.

---

# 121. Leak detection

Development/diagnostic mode podrá comprobar al final del scope:

```text
open transactions?
open cursors?
active streams?
leased connections?
dirty UnitOfWork?
non-empty IdentityMap?
tenant state?
stale operation context?
```

---

# 122. Strict runtime mode

Podrá existir:

```text
runtime.strict_reset = true
```

que convierte ciertas fugas en errores fuertes o señales operativas.

---

# 123. Production leak policy

Production deberá priorizar:

```text
isolation
cleanup
resource safety
diagnostics
```

sin convertir automáticamente cada warning en una caída completa.

---

# 124. Memory leak detection

Database podrá observar tendencias como:

```text
IdentityMap growth
UnitOfWork growth
cursor count
connection lease count
metadata duplication
```

pero el análisis global de memoria pertenece a Telemetry/Runtime.

---

# 125. Long-running process model

Un CLI/queue worker puede procesar grandes cantidades.

Ejemplo:

```text
Process
  │
  ├── Batch 1
  ├── Batch 2
  ├── Batch 3
  └── ...
```

No deberá mantener un PersistenceContext infinito.

---

# 126. Batch boundary

Puede definirse:

```text
load batch
   │
   ▼
process
   │
   ▼
flush
   │
   ▼
clear
   │
   ▼
next batch
```

---

# 127. Full scope vs persistence clear

Debe distinguirse:

```text
PersistenceContext clear
```

de:

```text
Execution Scope end
```

Un proceso largo puede limpiar ORM sin cerrar todo DatabaseContext.

---

# 128. Queue worker lifecycle

```text
Queue Worker
   │
   ├── Job A
   │    ├── Scope A
   │    └── Cleanup A
   │
   ├── Job B
   │    ├── Scope B
   │    └── Cleanup B
   │
   └── Job C
        ├── Scope C
        └── Cleanup C
```

---

# 129. Failed queue job

Aunque el job falle:

```text
JobException
    │
    ▼
Database cleanup
    │
    ▼
Scope closed
    │
    ▼
queue failure handling
```

---

# 130. Scheduled task lifecycle

Cada ejecución de tarea deberá tener su propio scope aunque el scheduler viva persistentemente.

---

# 131. CLI command lifecycle

Para comando corto:

```text
CLI Start
   │
   ▼
Execution Scope
   │
   ▼
Command
   │
   ▼
Cleanup
   │
   ▼
CLI End
```

---

# 132. Interactive CLI lifecycle

Una consola interactiva podrá necesitar:

```text
session scope
```

y subscopes explícitos.

La semántica deberá ser definida por la integración CLI.

---

# 133. Migration lifecycle

Migrations deberán usar:

```text
Migration Execution Scope
```

con:

```text
migration connection
migration transaction
migration lock
schema operation resources
```

---

# 134. Migration cleanup

Ante fallo:

```text
rollback when supported
release migration lock
reset connection
close scope
```

La liberación del lock será crítica.

---

# 135. Seeder lifecycle

Seeders podrán usar ORM.

Por ello deberán respetar:

```text
EntityManager scope
UnitOfWork lifecycle
batch clear
transaction lifecycle
```

---

# 136. Testing lifecycle

Cada test podrá obtener un Database scope independiente.

```text
Test A
  │
  └── DatabaseContext A

Test B
  │
  └── DatabaseContext B
```

---

# 137. Transactional tests

Un test puede abrir una transacción externa de testing.

El runtime deberá distinguir:

```text
test transaction ownership
```

de transacciones iniciadas por código probado.

---

# 138. Test cleanup

Aun con failure/assertion:

```text
rollback
clear ORM
reset connection
close context
```

deberá ejecutarse.

---

# 139. RoadRunner future adapter

RoadRunner deberá mapear:

```text
RoadRunner Worker
      │
      ▼
VoltStack Execution Scope
      │
      ▼
DatabaseLifecycle
```

sin modificar el Core Database.

---

# 140. RoadRunner invariant

La suite deberá comprobar:

```text
Job/Request A state
≠
Job/Request B state
```

aunque compartan worker.

---

# 141. OpenSwoole future adapter

OpenSwoole añade un reto adicional:

```text
concurrent coroutines
```

por tanto:

```text
Coroutine A
→ DatabaseContext A

Coroutine B
→ DatabaseContext B
```

deberán coexistir correctamente.

---

# 142. Coroutine isolation

Nunca:

```text
static $currentEntityManager
```

porque dos coroutines podrían sobrescribirlo.

---

# 143. Connection pool concurrency

Un pool compartido deberá garantizar:

```text
one exclusive lease
```

cuando una conexión no soporte multiplexación.

Dos contexts no deberán utilizar simultáneamente la misma conexión física de forma insegura.

---

# 144. Connection affinity

Determinadas operaciones requieren afinidad:

```text
transaction
temporary table
session variable
cursor
advisory lock
```

El sistema deberá conservar la conexión apropiada mientras dure esa semántica.

---

# 145. Temporary tables

Si una operación crea temporary tables, éstas pueden sobrevivir en session.

Esto afecta pool reset.

El Platform/Connection Reset System deberá conocer o tratar esta situación conservadoramente.

---

# 146. Advisory locks

Locks asociados a sesión/conexión deberán:

```text
release explicitly
```

o provocar descarte si no puede garantizarse su liberación.

---

# 147. Session variables

Cambios como:

```text
SET search_path
SET timezone
SET role
```

deberán formar parte de la estrategia de reset cuando sean gestionados por VoltStack.

---

# 148. Tenant schema switching

En una estrategia:

```text
SET search_path = tenant_x
```

el reset deberá garantizar:

```text
search_path → safe default
```

antes de devolver la conexión al pool.

---

# 149. Session role switching

Si se usa:

```text
SET ROLE
```

deberá restaurarse antes del reuse.

Esto es un requisito de seguridad.

---

# 150. Transaction-local state

Algunos motores soportan:

```text
SET LOCAL
```

que desaparece al terminar transacción.

El Platform podrá preferir mecanismos naturalmente scoped cuando existan.

---

# 151. Resource ownership principle

Todo recurso deberá responder:

```text
Who owns me?
When am I released?
What happens if release fails?
Can I outlive the current scope?
```

Si no puede responderse, el diseño está incompleto.

---

# 152. Ownership transfer

Cuando un método retorna un cursor:

```text
Executor
   │
   ▼
Cursor
```

el ownership se transfiere al consumidor/contexto de recursos.

Debe estar documentado.

---

# 153. Borrowed resources

Algunos objetos podrán recibir una conexión prestada.

No deberán cerrarla si no son propietarios.

---

# 154. Owned vs borrowed

Se recomienda modelar conceptualmente:

```text
Owned Resource
Borrowed Resource
Leased Resource
```

para evitar cleanup duplicado.

---

# 155. Cancellation lifecycle

Una operación podrá ser cancelada por:

```text
request abort
timeout
explicit cancellation
worker shutdown
```

---

# 156. Query cancellation

Flujo:

```text
Cancellation requested
        │
        ▼
Executor
        │
        ▼
Driver cancellation if supported
        │
        ▼
statement cleanup
        │
        ▼
connection state verification
```

---

# 157. Cancellation and connection safety

Después de cancelar una query, no deberá asumirse automáticamente que la conexión está lista.

El Driver/Platform deberá determinar:

```text
safe
needs reset
discard
```

---

# 158. Timeout lifecycle

Timeout de query:

```text
ExecutionContext
     │
     ▼
timeout
     │
     ▼
cancel/fail
     │
     ▼
resource cleanup
```

No deberá dejar cursores/statements huérfanos.

---

# 159. Request abort

Si el cliente HTTP desconecta:

```text
Client Disconnect
      │
      ▼
Runtime Cancellation
      │
      ▼
Database operation cancellation
```

cuando la integración lo permita.

---

# 160. Worker shutdown

En shutdown controlado:

```text
stop accepting scopes
      │
      ▼
finish/cancel active work
      │
      ▼
close contexts
      │
      ▼
close pools
      │
      ▼
worker exit
```

---

# 161. Graceful shutdown

Database deberá cooperar con:

```text
RuntimeManagerServer
```

para liberar recursos.

---

# 162. Forced shutdown

En terminación forzada no siempre será posible cleanup completo.

La arquitectura deberá minimizar dependencia de shutdown cleanup para correctness persistente.

---

# 163. Pool shutdown

Al finalizar worker:

```text
Pool
  │
  ▼
reject new leases
  │
  ▼
close idle connections
  │
  ▼
wait/cancel active leases according to policy
  │
  ▼
closed
```

---

# 164. Lifecycle events

Podrán existir eventos:

```text
DatabaseScopeStarted
DatabaseScopeClosing
DatabaseScopeClosed

DatabaseContextCreated
DatabaseContextClosed

ConnectionAcquired
ConnectionReleased
ConnectionDiscarded

TransactionStarted
TransactionCommitted
TransactionRolledBack

DatabaseCleanupStarted
DatabaseCleanupCompleted
DatabaseCleanupFailed
```

---

# 165. Event semantics

Los eventos de lifecycle deberán ser observacionales cuando sea posible.

Un listener no debería poder impedir cleanup crítico.

---

# 166. Cleanup events

Especialmente:

```text
DatabaseScopeClosing
```

no deberá permitir que un listener evite rollback/reset.

---

# 167. Telemetry integration

Telemetry podrá medir:

```text
scope DB duration
connections acquired
connections discarded
open transaction cleanup
dirty UnitOfWork
cursor leaks
reset duration
cleanup failures
```

---

# 168. Telemetry failure isolation

Si Telemetry falla durante cleanup:

```text
Database cleanup
```

deberá continuar.

Observabilidad no puede comprometer aislamiento.

---

# 169. Lifecycle diagnostics

En development, VoltStack podrá mostrar:

```text
Database Scope #9821

Connections acquired: 2
Transactions started: 1
Transactions committed: 1
Open cursors at close: 0
IdentityMap entities: 14
Dirty UoW at close: no
Cleanup duration: 0.7ms
```

---

# 170. Leak diagnostic example

```text
DATABASE RUNTIME WARNING

Scope:
    request-9812

Detected:
    1 open cursor
    1 active transaction

Actions:
    cursor closed
    transaction rolled back
    connection discarded
```

---

# 171. Lifecycle exception hierarchy

Conceptualmente:

```text
DatabaseRuntimeException
├── DatabaseContextException
│   ├── ContextClosedException
│   ├── ContextMismatchException
│   └── ContextLeakException
│
├── DatabaseCleanupException
│   ├── ResourceCleanupException
│   └── ConnectionResetException
│
├── DatabaseLifecycleException
└── DatabaseRuntimeInvariantViolation
```

---

# 172. Runtime invariants

### DB-RUNTIME-001

Cada Execution Scope tendrá estado Database independiente.

### DB-RUNTIME-002

Ningún EntityManager mutable sobrevivirá al Execution Scope.

### DB-RUNTIME-003

Ningún UnitOfWork sobrevivirá al Execution Scope.

### DB-RUNTIME-004

Ningún IdentityMap sobrevivirá al Execution Scope.

### DB-RUNTIME-005

Una transacción activa nunca será committed implícitamente durante cleanup.

### DB-RUNTIME-006

Toda transacción huérfana deberá intentar rollback.

### DB-RUNTIME-007

Una conexión insegura nunca volverá al pool.

### DB-RUNTIME-008

Los cursores abiertos deberán cerrarse antes de liberar su conexión.

### DB-RUNTIME-009

El tenant state nunca sobrevivirá al Execution Scope.

### DB-RUNTIME-010

Sticky routing state nunca sobrevivirá al Execution Scope.

### DB-RUNTIME-011

El Container no será utilizado como almacén de contexto mutable.

### DB-RUNTIME-012

El contexto actual nunca dependerá de una variable estática global incompatible con concurrencia.

### DB-RUNTIME-013

GC no será el mecanismo principal de cleanup.

### DB-RUNTIME-014

`__destruct()` no será el mecanismo principal de lifecycle.

### DB-RUNTIME-015

El cleanup deberá ejecutarse incluso cuando la aplicación termine con excepción.

### DB-RUNTIME-016

El error original de aplicación no deberá perderse por un error secundario de cleanup.

### DB-RUNTIME-017

Un recurso deberá tener ownership explícito.

### DB-RUNTIME-018

El reuse de conexiones requiere reset exitoso o una garantía equivalente.

### DB-RUNTIME-019

El final del scope no implica `EntityManager::flush()`.

### DB-RUNTIME-020

El final del scope implica destrucción o invalidación de todo estado ORM scoped.

### DB-RUNTIME-021

Database Core será independiente del runtime concreto.

### DB-RUNTIME-022

FrankenPHP, RoadRunner y OpenSwoole utilizarán el mismo contrato lógico de lifecycle.

### DB-RUNTIME-023

Los registries persistentes deberán ser inmutables o concurrency-safe.

### DB-RUNTIME-024

El estado de una operación no deberá almacenarse en servicios persistentes.

### DB-RUNTIME-025

Una referencia a un contexto cerrado no podrá reutilizarse silenciosamente.

---

# 173. Prohibited pattern — global EntityManager

```php
final class EntityManager
{
    private static ?self $instance = null;
}
```

**Prohibido.**

Motivo:

```text
Request A
   │
   ▼
global EntityManager
   │
   ▼
Request B
```

---

# 174. Prohibited pattern — worker current tenant

```php
TenantRuntime::$tenant = $tenant;
```

**Prohibido** como estado global mutable.

---

# 175. Prohibited pattern — connection transaction flag

```php
ConnectionManager::$inTransaction = true;
```

**Prohibido.**

El estado pertenece a `TransactionContext`.

---

# 176. Prohibited pattern — implicit flush

```text
Request End
   │
   ▼
EntityManager::flush()
```

**Prohibido por defecto.**

---

# 177. Prohibited pattern — pool without reset

```text
Request A
   │
   ▼
Connection
   │
   ▼
Pool
   │
   ▼
Request B
```

sin:

```text
reset/validation
```

**Prohibido.**

---

# 178. Prohibited pattern — connection global singleton

Una conexión física no deberá registrarse simplemente como:

```text
PDO
→ singleton
```

en un worker persistente.

---

# 179. Prohibited pattern — current connection property

Un singleton:

```php
final class DatabaseManager
{
    private ?Connection $currentConnection;
}
```

es sospechoso si representa estado de ejecución.

Deberá utilizarse `ConnectionScope`.

---

# 180. Prohibited pattern — context reuse

```text
Scope A closes
     │
     ▼
DatabaseContext A retained
     │
     ▼
Scope B uses A
```

deberá ser detectado y rechazado.

---

# 181. Lifecycle state machine general

```text
                   ┌──────────┐
                   │ Created  │
                   └────┬─────┘
                        │
                        ▼
                   ┌──────────┐
                   │ Active   │
                   └────┬─────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
         ┌─────────┐         ┌─────────┐
         │ Closing │◄────────│ Failed  │
         └────┬────┘         └─────────┘
              │
              ▼
         ┌─────────┐
         │ Closed  │
         └─────────┘
```

---

# 182. Resource lifecycle model

```text
Acquire
   │
   ▼
Use
   │
   ▼
Finalize
   │
   ▼
Reset
   │
   ├── safe ─────► Release
   │
   └── unsafe ───► Discard
```

---

# 183. Request isolation model

```text
                    WORKER
                      │
      ┌───────────────┼───────────────┐
      ▼               ▼               ▼
  REQUEST A        REQUEST B       REQUEST C
      │               │               │
      ▼               ▼               ▼
 DB Context A      DB Context B     DB Context C
      │               │               │
      ▼               ▼               ▼
  Cleanup A        Cleanup B        Cleanup C
      │               │               │
      └───────┬───────┴───────┬───────┘
              ▼               ▼
      Shared Immutable    Safe Shared
       Infrastructure      Resources
```

---

# 184. Persistence lifecycle model

```text
Execution Scope
      │
      ▼
PersistenceContext
      │
      ├── EntityManager
      ├── IdentityMap
      ├── UnitOfWork
      └── ChangeTracker
             │
             ▼
           flush()
             │
             ▼
     Persistence Engine
             │
             ▼
        Query Engine
```

At scope end:

```text
PersistenceContext
      │
      ▼
inspect
      │
      ▼
clear
      │
      ▼
close
```

---

# 185. Transaction lifecycle model

```text
Execution Scope
      │
      ▼
TransactionManager
      │
      ▼
TransactionContext
      │
      ▼
Connection Lease
      │
      ▼
BEGIN
      │
      ▼
Application Work
      │
  ┌───┴────┐
  ▼        ▼
COMMIT   ROLLBACK
  │        │
  └───┬────┘
      ▼
Finalize
```

---

# 186. Failure lifecycle

```text
Application
    │
    ▼
Exception
    │
    ▼
Database Scope Closing
    │
    ├── close cursor
    ├── cancel operation
    ├── rollback transaction
    ├── clear UoW
    ├── clear IdentityMap
    ├── reset connection
    └── release/discard
            │
            ▼
      Scope Closed
            │
            ▼
    Exception Propagated
```

---

# 187. Persistent runtime safety model

```text
                    Worker
                      │
       Persistent Infrastructure
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
    Scope A         Scope B         Scope C
       │              │              │
       ▼              ▼              ▼
   Context A       Context B       Context C
       │              │              │
       ▼              ▼              ▼
   RESET A         RESET B         RESET C
```

Regla:

```text
A state ∉ B
B state ∉ C
A state ∉ C
```

---

# 188. Lifecycle ownership table

| Componente | Owner | Lifetime |
|---|---|---|
| Compiled configuration | Worker/Application | Persistent |
| DriverRegistry | Worker/Application | Persistent |
| DialectRegistry | Worker/Application | Persistent |
| PlatformRegistry | Worker/Application | Persistent |
| TypeRegistry | Worker/Application | Persistent |
| Compiled Metadata | Worker/Application | Persistent |
| Connection Pool | Worker/Runtime | Persistent |
| DatabaseContext | Execution Scope | Scoped |
| ConnectionScope | DatabaseContext | Scoped |
| PersistenceContext | DatabaseContext | Scoped |
| EntityManager | PersistenceContext | Scoped |
| UnitOfWork | PersistenceContext | Scoped |
| IdentityMap | PersistenceContext | Scoped |
| TransactionContext | DatabaseContext | Scoped |
| QueryContext | Query Operation | Operation |
| ExecutionContext | Execution Operation | Operation |
| HydrationContext | Hydration Operation | Operation |
| Connection Lease | Scope/Transaction/Resource | Resource |
| Physical Connection | Pool/Connection Manager | Resource |
| Statement | Connection/Operation | Resource |
| Cursor | Consumer/Resource Registry | Resource |
| Stream | Consumer/Resource Registry | Resource |

---

# 189. Lifecycle responsibility matrix

| Acción | Responsable principal |
|---|---|
| Crear scope | VoltStack Runtime |
| Crear DatabaseContext | Database Runtime Integration |
| Resolver connection | ConnectionManager |
| Adquirir lease | ConnectionScope/Manager |
| Iniciar transaction | TransactionManager |
| Registrar transaction state | TransactionContext |
| Administrar entities | EntityManager |
| Registrar cambios | UnitOfWork |
| Mantener identidad | IdentityMap |
| Cerrar cursor | Cursor/Resource cleanup |
| Rollback huérfano | Transaction cleanup |
| Reset connection | Connection Reset System |
| Liberar connection | Pool/Connection Manager |
| Cerrar PersistenceContext | Database cleanup |
| Destruir scoped state | Runtime/Container |
| Solicitar recycle | Database Runtime Adapter |
| Reciclar worker | RuntimeManagerServer |

---

# 190. Lifecycle architecture layers

```text
Runtime Layer
    │
    ▼
Execution Scope
    │
    ▼
Database Runtime Integration
    │
    ▼
DatabaseContext
    │
    ├── Persistence Lifecycle
    ├── Transaction Lifecycle
    ├── Connection Lifecycle
    └── Resource Lifecycle
            │
            ▼
       Driver Resources
```

---

# 191. Separation from Container

El Container administra:

```text
service lifetime
scope construction
scope destruction
```

Database administra:

```text
database semantic cleanup
transaction safety
connection reset
resource ownership
```

El Container no debe adivinar cómo hacer rollback de una transacción.

---

# 192. Separation from RuntimeManagerServer

RuntimeManagerServer administra:

```text
worker lifecycle
server integration
worker recycling
graceful shutdown
```

Database administra:

```text
DatabaseContext lifecycle
DB resource cleanup
connection state
ORM state
```

---

# 193. Separation from ORM

ORM administra:

```text
entity state
UnitOfWork
IdentityMap
persistence lifecycle
```

pero no es propietario del worker/request lifecycle general.

---

# 194. Separation from Connection Manager

Connection Manager administra:

```text
logical connections
targets
leases
pool interaction
```

pero no es propietario de:

```text
EntityManager
UnitOfWork
tenant lifecycle
request lifecycle
```

---

# 195. Lifecycle API conceptual

Podrá existir:

```php
interface DatabaseLifecycleInterface
{
    public function begin(
        ExecutionScopeInterface $scope
    ): DatabaseContextInterface;

    public function end(
        DatabaseContextInterface $context
    ): DatabaseCleanupReport;
}
```

La API definitiva podrá ser distinta.

---

# 196. DatabaseContext interface conceptual

```php
interface DatabaseContextInterface
{
    public function id(): DatabaseContextId;

    public function state(): DatabaseContextState;

    public function isActive(): bool;
}
```

Los subsistemas especializados accederán mediante contracts específicos.

---

# 197. Scope termination API

El Runtime deberá garantizar:

```text
try
    application()
finally
    databaseLifecycle.end()
```

conceptualmente.

Esto garantiza cleanup incluso ante excepciones.

---

# 198. Lifecycle nesting

Si existen scopes anidados legítimos:

```text
Outer Scope
    │
    └── Inner Scope
```

deberá estar definido si:

```text
share DatabaseContext
```

o:

```text
create isolated DatabaseContext
```

No deberá inferirse accidentalmente.

---

# 199. Default nesting policy

Para operaciones internas ordinarias:

```text
same Execution Scope
→ same DatabaseContext
```

No crear contextos nuevos por:

```text
service call
repository call
transaction callback
query
```

---

# 200. Isolated subcontext

Podrá existir una API avanzada para:

```text
isolated persistence work
```

pero deberá declarar claramente:

```text
connection sharing?
transaction sharing?
identity sharing?
```

---

# 201. No ambiguous nested state

Nunca deberá existir un subcontexto cuya relación con la transacción padre sea indefinida.

---

# 202. Transaction propagation model

En el futuro podrán definirse estrategias como:

```text
REQUIRED
REQUIRES_NEW
SUPPORTS
NOT_SUPPORTED
```

si VoltStack adopta este modelo.

No forma parte obligatoria de la primera implementación.

---

# 203. Lifecycle and retries

Los retries deberán respetar boundaries.

Ejemplo:

```text
Query retry
→ operation boundary

Transaction retry
→ transaction callback boundary

Connection retry
→ acquisition boundary
```

No mezclar niveles.

---

# 204. Retry cleanup

Antes de reintentar:

```text
failed attempt
    │
    ▼
cleanup attempt resources
    │
    ▼
verify connection state
    │
    ▼
retry
```

---

# 205. Retry and ORM

Un transaction retry con ORM deberá reconstruir o restaurar apropiadamente el estado de persistencia.

No repetir ciegamente `flush()` sobre un UnitOfWork parcialmente modificado.

---

# 206. Lifecycle and events

Eventos ORM pueden disparar nuevas queries.

Estas queries pertenecen al mismo Execution Scope salvo que se defina otra cosa.

---

# 207. Reentrancy

El sistema deberá contemplar:

```text
Query
  │
  ▼
Event
  │
  ▼
Query
```

sin corromper QueryContext/ExecutionContext.

Cada operación deberá tener su propio contexto.

---

# 208. Recursive flush

ORM deberá detectar escenarios peligrosos como:

```text
flush()
  │
  ▼
lifecycle event
  │
  ▼
flush()
```

según las reglas del futuro Persistence System.

---

# 209. Runtime shutdown state

Database Runtime podrá tener estados:

```text
Running
Draining
Stopping
Stopped
```

---

# 210. Draining

Durante:

```text
Draining
```

no deberán aceptarse nuevos scopes Database, pero los existentes podrán finalizar según política.

---

# 211. Stopping

Durante `Stopping`:

```text
cancel/finish operations
close contexts
close pool
flush telemetry if applicable
```

sin iniciar nuevo trabajo.

---

# 212. Worker pool resource ownership

Si el pool pertenece al worker:

```text
Worker shutdown
→ pool shutdown
```

Si pertenece a infraestructura externa:

```text
Worker shutdown
→ release adapter/resources
```

La ownership debe ser explícita.

---

# 213. Fork/process safety

Si en el futuro VoltStack utiliza modelos con process fork, conexiones creadas antes del fork no deberán asumirse reutilizables automáticamente.

El Runtime Adapter deberá definir la política.

---

# 214. Signal handling

Database no deberá registrar señales del sistema directamente salvo infraestructura Runtime especializada.

RuntimeManagerServer deberá traducirlas a lifecycle events.

---

# 215. Runtime portability

Database Core deberá comprender:

```text
ExecutionScope
DatabaseContext
Lifecycle
Cancellation
Resource ownership
```

No:

```text
FrankenPHP request object
RoadRunner worker object
OpenSwoole coroutine API
```

---

# 216. Runtime adapters

Arquitectura:

```text
                    Database Core
                         ▲
                         │
               Database Runtime Port
                         ▲
             ┌───────────┼───────────┐
             │           │           │
       FrankenPHP   RoadRunner   OpenSwoole
         Adapter      Adapter       Adapter
```

---

# 217. Adapter responsibility

Cada adapter traduce:

```text
runtime start
scope start
scope end
cancellation
shutdown
worker recycle
```

hacia contratos neutrales.

---

# 218. FrankenPHP reference conformance

La primera implementación deberá validar al menos:

```text
1000+ sequential requests same worker
```

en tests de estrés/arquitectura, verificando ausencia de:

```text
IdentityMap leakage
UnitOfWork leakage
transaction leakage
tenant leakage
connection session leakage
cursor leakage
```

El número definitivo pertenecerá al sistema de performance/testing.

---

# 219. Concurrent conformance

Cuando exista runtime concurrente deberá probarse:

```text
Scope A
Scope B
Scope C
```

intercalados dentro del mismo proceso.

---

# 220. Failure injection tests

Deberán simularse:

```text
query failure
connection loss
rollback failure
cursor close failure
reset failure
telemetry failure
event listener failure
application exception
cancellation
timeout
```

para validar cleanup.

---

# 221. Lifecycle conformance suite

Podrá existir:

```text
DatabaseRuntimeConformanceSuite
```

que todos los runtime adapters oficiales deberán superar.

---

# 222. Connection reset conformance

Cada driver deberá demostrar que:

```text
dirty session
    │
    ▼
reset
    │
    ▼
known clean session
```

o declarar que cierto estado obliga a descartar la conexión.

---

# 223. Resource leak test

Después de cada scope:

```text
open cursors = 0
open scoped statements = 0
orphan transactions = 0
active scoped leases = 0
IdentityMap entries = 0
UnitOfWork pending = 0
tenant state = none
```

salvo recursos persistentemente administrados explícitos.

---

# 224. Runtime performance principle

Lifecycle safety no deberá significar reconstruir toda Database en cada request.

Se reutilizará:

```text
immutable infrastructure
compiled metadata
registries
compilers
safe connection pools
```

mientras se recrea únicamente:

```text
mutable scoped state
```

---

# 225. Performance model

```text
Worker Boot Cost
    │
    ▼
amortized across requests

Request Cost
    │
    ├── create tiny scope
    ├── lazy DatabaseContext
    ├── lazy connection
    └── deterministic cleanup
```

---

# 226. Lazy scoped components

Incluso dentro de un DatabaseContext:

```text
PersistenceContext
TransactionContext
ConnectionScope
```

podrán inicializarse sólo cuando se necesiten.

---

# 227. Empty request optimization

Un request sin DB deberá aproximarse a:

```text
Database lifecycle overhead ≈ minimal
```

sin:

```text
connection
EntityManager
UnitOfWork
metadata scanning
```

---

# 228. Cleanup performance

Cleanup deberá ser proporcional a recursos realmente utilizados.

No recorrer grandes registries globales en cada request.

---

# 229. Resource tracking performance

`ResourceRegistry` deberá ser ligero.

Registrar únicamente recursos que realmente requieren cleanup explícito.

---

# 230. Correctness before micro-optimization

Si existe conflicto:

```text
slightly faster connection reuse
vs
uncertain session reset
```

VoltStack deberá elegir:

```text
discard connection
```

Correctness y aislamiento tienen prioridad.

---

# 231. Security priority

Especialmente para:

```text
tenant state
database role
schema
session authorization
credentials
```

no deberá existir reuse si el reset es incierto.

---

# 232. Runtime state categories

Se formalizan cuatro categorías:

```text
A. Immutable Persistent State
B. Safe Mutable Infrastructure
C. Scoped Mutable State
D. External Resources
```

---

# 233. Category A — Immutable Persistent State

Ejemplos:

```text
compiled metadata
configuration
dialects
capability definitions
compiler definitions
```

Puede compartirse libremente.

---

# 234. Category B — Safe Mutable Infrastructure

Ejemplos:

```text
connection pool
health state
metrics aggregation
```

Debe ser concurrency-safe y tener ownership claro.

---

# 235. Category C — Scoped Mutable State

Ejemplos:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
TenantDatabaseContext
sticky routing
```

Nunca cruza Execution Scope.

---

# 236. Category D — External Resources

Ejemplos:

```text
physical connections
statements
cursors
streams
```

Requieren acquire/release semantics.

---

# 237. State classification rule

Todo nuevo componente Database deberá clasificarse en A, B, C o D durante architectural review.

Si no puede clasificarse, deberá revisarse su diseño.

---

# 238. Lifecycle review checklist

Para cada componente:

```text
1. ¿Quién lo crea?
2. ¿Quién lo posee?
3. ¿Cuánto vive?
4. ¿Es mutable?
5. ¿Puede compartirse?
6. ¿Es concurrency-safe?
7. ¿Qué recursos posee?
8. ¿Cómo se cierra?
9. ¿Qué ocurre si cleanup falla?
10. ¿Puede sobrevivir al request?
11. ¿Puede contener tenant/user state?
12. ¿Puede capturar scoped dependencies?
13. ¿Cómo se prueba ausencia de leaks?
```

---

# 239. Runtime architectural checklist

Antes de aceptar una feature:

```text
Does it introduce mutable static state?
Does it survive a scope?
Does it hold a connection?
Does it change session state?
Does it affect transaction affinity?
Does it require cleanup?
Does it behave correctly under exceptions?
Does it behave correctly under cancellation?
Does it work under persistent workers?
Does it work under concurrent contexts?
```

---

# 240. Example — safe repository

```text
UserRepository
     │
     ▼
EntityManager
     │
     ▼
PersistenceContext
     │
     ▼
Execution Scope
```

Repository:

```text
scoped
```

Correcto.

---

# 241. Example — unsafe repository singleton

```text
Singleton UserRepository
       │
       ▼
EntityManager A
       │
       ▼
Request A ends
       │
       ▼
Request B
```

Incorrecto.

---

# 242. Example — safe compiler

```text
QueryCompiler
├── Dialect
├── Platform
└── CompilerRules
```

Todos inmutables/stateless.

Puede ser persistentemente compartido.

---

# 243. Example — safe pool

```text
Worker
  │
  ▼
ConnectionPool
  │
  ├── Connection A
  ├── Connection B
  └── Connection C
```

Cada request recibe leases exclusivos y las conexiones pasan reset antes de reutilizarse.

---

# 244. Example — unsafe pool

```text
Request A
   │
   ▼
SET ROLE tenant_a
   │
   ▼
Pool
   │
   ▼
Request B
```

sin reset.

Resultado:

```text
cross-request security violation
```

---

# 245. Example — safe transaction cleanup

```text
Request
  │
  ▼
BEGIN
  │
  ▼
Exception
  │
  ▼
finally
  │
  ▼
ROLLBACK
  │
  ▼
connection reset
  │
  ▼
release
```

---

# 246. Example — unsafe cleanup

```text
Request
  │
  ▼
BEGIN
  │
  ▼
Exception
  │
  ▼
connection returned to pool
```

sin rollback.

**Prohibido.**

---

# 247. Final runtime architecture

```text
┌───────────────────────────────────────────────────────────┐
│                    VOLTSTACK WORKER                       │
│                                                           │
│  Persistent / Shareable                                   │
│  ──────────────────────                                   │
│  Configuration                                            │
│  Metadata                                                 │
│  Driver Registry                                          │
│  Dialects                                                 │
│  Platforms                                                │
│  Type Registry                                            │
│  Compilers                                                │
│  Optimizer Rules                                          │
│  Connection Pool                                          │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ EXECUTION SCOPE A                                   │  │
│  │                                                     │  │
│  │ DatabaseContext                                     │  │
│  │ ├── ConnectionScope                                 │  │
│  │ ├── PersistenceContext                              │  │
│  │ │   ├── EntityManager                               │  │
│  │ │   ├── UnitOfWork                                  │  │
│  │ │   └── IdentityMap                                 │  │
│  │ ├── TransactionContext                              │  │
│  │ └── ResourceRegistry                                │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │                                 │
│                       CLEANUP                             │
│                         │                                 │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │ EXECUTION SCOPE B                                   │  │
│  │                                                     │  │
│  │ completely new scoped Database state                │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

# 248. Core lifecycle equation

La arquitectura puede resumirse como:

```text
Persistent Infrastructure
        +
Fresh Scoped State
        +
Explicit Resource Ownership
        +
Deterministic Cleanup
        =
Safe Persistent Runtime
```

---

# 249. Criterio de éxito

La implementación será considerada correcta cuando:

```text
Request N
```

pueda ejecutarse en el mismo worker que:

```text
Request 1
```

sin poder observar ningún estado mutable específico de ese primer request.

Formalmente:

```text
ScopedState(Request A)
∩
ScopedState(Request B)
=
∅
```

para cualquier:

```text
A != B
```

excepto infraestructura explícitamente compartida que no contenga estado específico de la ejecución.

---

# 250. Principio final

La regla maestra del Runtime Database será:

> Persistir infraestructura; aislar estado; poseer recursos; limpiar determinísticamente.

En forma compacta:

```text
SHARE
  │
  └── immutable/stateless infrastructure

ISOLATE
  │
  └── mutable execution state

OWN
  │
  └── external resources

RESET
  │
  └── reusable resources

DISCARD
  │
  └── anything whose safety cannot be proven
```

---

# 251. Conclusión

El Runtime Model de `VoltStack/Quantum/Database` no estará diseñado alrededor del supuesto tradicional:

```text
one request
=
one PHP process
```

La arquitectura partirá desde el principio del modelo:

```text
one worker
=
many isolated execution scopes
```

Esto permitirá que FrankenPHP sea un runtime de primera clase sin introducir posteriormente mecanismos correctivos para limpiar una arquitectura originalmente diseñada para procesos efímeros.

El mismo modelo permitirá integrar:

```text
FrankenPHP
RoadRunner
OpenSwoole
Queue Workers
CLI
Schedulers
Long-running Processes
```

mediante adapters, manteniendo una sola semántica interna.

La frontera esencial será:

```text
Worker
    │
    ├── Persistent Safe Infrastructure
    │
    ├── Scope A → Cleanup
    ├── Scope B → Cleanup
    ├── Scope C → Cleanup
    └── Scope N → Cleanup
```

y la regla de seguridad más importante será:

> Ningún estado mutable perteneciente a una ejecución podrá convertirse accidentalmente en estado del worker.

---

# 252. Siguiente documento

El siguiente documento es:

```text
09_DATABASE_EXTENSION_AND_CAPABILITY_MODEL.md
```

Este documento deberá cerrar el bloque fundamental `00–09` definiendo formalmente:

```text
Database extensions
extension contracts
extension descriptors
extension lifecycle
extension registration
extension discovery
extension priorities
extension dependency graph
extension compatibility
extension isolation

capability model
platform capabilities
driver capabilities
runtime capabilities
feature capabilities
capability discovery
capability negotiation
native support
emulated support
unsupported features
capability requirements
capability fallbacks

extension points
registries
decorators
custom drivers
custom dialects
custom types
custom query features
custom compilers
custom ORM features
custom metadata loaders
custom integrations
```

con una regla fundamental:

```text
Feature
   │
   ▼
Capability Requirement
   │
   ▼
Effective Platform/Driver Capabilities
   │
   ├── Native
   ├── Emulated
   └── Unsupported
```

evitando completamente patrones dispersos como:

```php
if ($driver === 'pgsql') {
    // ...
}
```

en los subsistemas superiores.