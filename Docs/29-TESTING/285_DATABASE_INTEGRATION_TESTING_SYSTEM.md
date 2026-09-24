# 285_DATABASE_INTEGRATION_TESTING_SYSTEM.md

# VoltStack Quantum Database
## Database Integration Testing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 285 — Database Integration Testing System  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `284_DATABASE_UNIT_TESTING_SYSTEM.md`  
**Siguiente documento:** `286_DATABASE_DATABASE_TEST_ENVIRONMENT_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database Integration Testing System** de VoltStack.

Su responsabilidad es establecer cómo verificar el comportamiento del subsistema Database cuando sus componentes dejan de ejecutarse únicamente contra doubles controlados y comienzan a interactuar con infraestructura real.

La regla central será:

> **Cuando una garantía de VoltStack dependa del comportamiento real de un driver, conexión, protocolo, DBMS, transacción, esquema, proceso o recurso externo, dicha garantía deberá validarse mediante integración real y no inferirse únicamente desde mocks, fakes o pruebas unitarias.**

Formalmente:

```text
Integration Evidence
=
VoltStack Component
+
Real Integration Boundary
+
Controlled Environment
+
Observable External Effect
+
Verified Result
```

No:

```text
Integration Evidence
=
MockConnection
+
Expected Calls
```

---

# 2. Objetivos

El sistema deberá permitir verificar:

- conexiones reales;
- ejecución real de consultas;
- bindings;
- comportamiento del driver;
- transacciones;
- savepoints;
- aislamiento;
- locking;
- schema operations;
- migrations;
- ORM persistence;
- hydration;
- relationships;
- type conversion;
- read/write routing;
- replicas;
- sharding;
- cache integrations;
- events;
- telemetry;
- backup/restore;
- mantenimiento;
- runtime persistente;
- recuperación ante fallos.

---

# 3. Integration Testing ≠ Unit Testing

Unit Testing responde:

```text
¿Nuestra lógica produce la decisión correcta
bajo dependencias controladas?
```

Integration Testing responde:

```text
¿La decisión y los componentes de VoltStack
funcionan realmente al interactuar con
la infraestructura correspondiente?
```

---

# 4. Integration Testing ≠ Conformance Testing

Una prueba de integración puede verificar:

```text
PostgreSQL + Driver X + VoltStack
```

para una operación concreta.

Conformance Testing verificará sistemáticamente:

```text
Implementation
      ↓
VoltStack Database Contract
```

a través de una matriz completa de implementaciones.

Será definido posteriormente en:

```text
292_DATABASE_DRIVER_CONFORMANCE_TESTING_SYSTEM.md
```

---

# 5. Integration Testing ≠ End-to-End Testing

Una prueba Database Integration no necesita levantar:

```text
HTTP
Router
Controller
Authentication
UI
SPA Runtime
```

salvo que la integración específica lo requiera.

Ejemplo:

```text
ORM
 ↓
Query Engine
 ↓
Compiler
 ↓
Executor
 ↓
Connection
 ↓
Driver
 ↓
PostgreSQL
```

es suficiente para probar integración Database.

---

# 6. Integration Testing ≠ Production

Un entorno de integración real sigue siendo controlado.

```text
Real DBMS
≠
Production Database
```

Nunca se utilizarán datos productivos por defecto.

---

# 7. Arquitectura general

```text
                   Integration Test
                          │
                          ▼
                   Test Scenario
                          │
                          ▼
                  VoltStack Database
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
     Driver            Process            Storage
       │                  │                  │
       ▼                  ▼                  ▼
   Real DBMS         Real Tool         Real Adapter
       │
       ▼
 Observable Database State
       │
       ▼
 Verification / Assertions
       │
       ▼
 Integration Evidence
```

---

# 8. Principio de infraestructura mínima real

La infraestructura deberá ser tan real como requiera la propiedad evaluada.

Ejemplo:

Para probar SQL PostgreSQL:

```text
Real PostgreSQL
```

Para probar planificación de SQL PostgreSQL:

```text
no DB required
```

Para probar backup mediante `pg_dump`:

```text
Real PostgreSQL
+
compatible real backup tool
```

---

# 9. Real boundary principle

Si la propiedad depende de:

```text
DBMS
Driver
Network
Filesystem
External Process
Object Storage
```

la frontera relevante deberá ser real.

---

# 10. No unnecessary infrastructure

Esto no significa que toda prueba de integración deba utilizar todos los servicios.

Incorrecto:

```text
Test Type Conversion
+
PostgreSQL
+
Redis
+
S3
+
FrankenPHP
```

si únicamente PostgreSQL es relevante.

---

# 11. Integration Scope

Cada test deberá declarar conceptualmente:

```text
IntegrationScope
├── Components
├── Platform
├── Driver
├── External Resources
├── Required Capabilities
├── Initial State
├── Isolation Strategy
└── Cleanup Strategy
```

---

# 12. IntegrationTestContext

Podrá existir:

```php
final readonly class DatabaseIntegrationTestContext
{
    public function __construct(
        public PlatformDescriptor $platform,
        public DriverDescriptor $driver,
        public ConnectionConfiguration $connection,
        public CapabilitySnapshot $capabilities,
        public TestDatabaseIdentity $database,
        public TestRunId $runId,
    ) {}
}
```

---

# 13. Context ≠ global state

Nunca deberá existir:

```php
IntegrationTestContext::current();
```

como singleton mutable global.

El contexto deberá ser inyectado o scoped.

---

# 14. Supported platform matrix

La suite deberá contemplar al menos:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

como plataformas independientes.

---

# 15. MySQL ≠ MariaDB

VoltStack no utilizará:

```text
MySQL test passed
therefore
MariaDB test passed
```

---

# 16. SQLite ≠ portable substitute

SQLite podrá utilizarse para integración SQLite.

No podrá utilizarse como sustituto universal de:

```text
MySQL
MariaDB
PostgreSQL
```

---

# 17. Platform Matrix

Conceptualmente:

| Platform | Unit | Integration | Conformance |
|---|---:|---:|---:|
| MySQL | Sí | Sí | Sí |
| MariaDB | Sí | Sí | Sí |
| PostgreSQL | Sí | Sí | Sí |
| SQLite | Sí | Sí | Sí |

---

# 18. Version Matrix

Las versiones soportadas deberán probarse de acuerdo con la política de compatibilidad de VoltStack.

Conceptualmente:

```text
Platform
   │
   ├── Minimum Supported
   ├── Representative Stable
   └── Latest Supported
```

cuando resulte práctico.

---

# 19. Version ≠ Capability

La versión del servidor será información de contexto.

Las pruebas deberán seguir verificando las capacidades efectivas cuando la arquitectura así lo requiera.

---

# 20. Driver Matrix

La integración deberá distinguir:

```text
Platform
```

de:

```text
Driver
```

Ejemplo conceptual:

```text
PostgreSQL
   +
PDO PgSQL
```

es una combinación concreta.

---

# 21. Test database provisioning

Cada suite deberá disponer de una base controlada.

Ejemplo:

```text
voltstack_test_<run>
```

o un aislamiento equivalente.

---

# 22. Production protection

Antes de ejecutar operaciones destructivas, el sistema deberá validar que el destino pertenece a un entorno de pruebas permitido.

---

# 23. Test Environment Marker

Podrá utilizarse un marcador explícito:

```text
VOLTSTACK_TEST_DATABASE=true
```

más validaciones adicionales.

---

# 24. Marker ≠ sufficient protection

Una variable de entorno aislada no será suficiente para operaciones extremadamente destructivas.

Podrán comprobarse además:

```text
database name
host allowlist
environment
credentials
test metadata
explicit destructive-test capability
```

---

# 25. Isolation strategies

Podrán existir:

```text
DATABASE_PER_SUITE
DATABASE_PER_TEST
SCHEMA_PER_SUITE
SCHEMA_PER_TEST
TRANSACTION_PER_TEST
TRUNCATE_RESET
SNAPSHOT_RESET
CUSTOM
```

---

# 26. Isolation strategy selection

La estrategia dependerá de la propiedad.

No se impondrá:

```text
one transaction per test
```

universalmente.

---

# 27. Transaction isolation caveat

Un test que verifica:

```text
commit
rollback
deadlock
multiple connections
isolation level
locking
```

no puede envolverse ingenuamente en una única transacción externa de test.

---

# 28. Transactional test isolation

Será tratado específicamente en:

```text
287_DATABASE_TRANSACTIONAL_TESTING_SYSTEM.md
```

---

# 29. Database lifecycle

Conceptualmente:

```text
Provision
   ↓
Bootstrap
   ↓
Migrate
   ↓
Seed/Fixture
   ↓
Execute Test
   ↓
Verify
   ↓
Cleanup
   ↓
Destroy/Reset
```

---

# 30. Schema bootstrap

La suite podrá construir el esquema mediante:

```text
Migration System
Schema Builder
Prebuilt Test Snapshot
Platform-specific bootstrap
```

según la propiedad probada.

---

# 31. Do not hide migration bugs

Si la prueba pretende verificar migrations:

```text
Schema Builder shortcut
```

no podrá sustituirlas.

---

# 32. Query Execution Testing

Pipeline real:

```text
Query Builder
     ↓
AST
     ↓
Semantic Analysis
     ↓
Planner
     ↓
Compiler
     ↓
Executor
     ↓
Connection
     ↓
Driver
     ↓
DBMS
```

---

# 33. Query execution assertions

Podrán comprobarse:

```text
returned rows
affected rows
generated identifiers
types
ordering
NULL semantics
errors
transaction effects
```

---

# 34. Generated SQL

Podrá registrarse para diagnóstico.

Pero el resultado real será la principal evidencia funcional.

---

# 35. SQL accepted ≠ semantics correct

Que el servidor acepte:

```sql
SELECT ...
```

no demuestra que VoltStack haya producido el resultado correcto.

La prueba deberá validar estado/resultado.

---

# 36. Parameter Binding Integration

Se deberán verificar bindings reales para:

```text
integer
float
decimal
boolean
string
binary
NULL
date/time
JSON
large values
```

según plataforma.

---

# 37. Injection safety integration

Además de unit tests, podrán existir pruebas reales con valores como:

```text
'
"
;
--
/*
```

para comprobar que permanecen datos y no estructura SQL.

---

# 38. Identifier security

Las pruebas deberán diferenciar:

```text
value binding
```

de:

```text
identifier handling
```

ya que los identificadores normalmente no pueden parametrizarse.

---

# 39. Result System Integration

Se verificará:

```text
row fetching
cursor behavior
streaming
scalar result
tuple result
empty result
NULL
large result
```

---

# 40. Result Cursor

Las pruebas deberán verificar:

```text
open
iterate
exhaust
close
early termination
```

contra un driver real.

---

# 41. Streaming Result

Se deberá comprobar que la API puede procesar conjuntos grandes sin materializarlos completamente cuando la plataforma/driver lo permita.

---

# 42. Streaming ≠ constant memory proof

La prueba deberá medir específicamente el componente relevante antes de afirmar propiedades de memoria.

---

# 43. Query timeout

Cuando el driver/plataforma lo permita, deberá probarse un timeout real.

---

# 44. Cancellation

La cancelación real deberá verificarse únicamente donde exista soporte efectivo.

---

# 45. Execution errors

La suite deberá provocar errores controlados como:

```text
syntax error
constraint violation
unknown table
unknown column
duplicate key
foreign key violation
timeout
connection failure
```

---

# 46. Error mapping

El objetivo será comprobar:

```text
native error
      ↓
Driver
      ↓
VoltStack exception taxonomy
```

---

# 47. Native error code

Podrá conservarse como metadata diagnóstica cuando sea seguro.

---

# 48. Error mapping ≠ string matching only

Se preferirán códigos/estados estructurados disponibles en el driver.

---

# 49. Type Integration

El Type System deberá probarse contra valores realmente almacenados.

---

# 50. Type round-trip

```text
PHP
 ↓
VoltStack conversion
 ↓
Driver
 ↓
DBMS
 ↓
Driver
 ↓
VoltStack conversion
 ↓
PHP'
```

---

# 51. Round-trip contract

Cuando sea lossless:

```text
PHP' = PHP
```

---

# 52. Platform normalization

Cuando el DBMS normalice:

```text
precision
timezone
case
collation
decimal scale
```

la prueba deberá comparar según la semántica del tipo, no representación accidental.

---

# 53. Boolean integration

Especial atención a diferencias entre plataformas.

---

# 54. Decimal integration

No deberá convertirse inadvertidamente a `float` cuando se requiera precisión decimal exacta.

---

# 55. Binary integration

Se deberán probar:

```text
zero byte
arbitrary bytes
large binary
empty binary
```

según capacidades.

---

# 56. JSON integration

Casos:

```text
object
array
scalar
JSON null
SQL NULL
nested path
Unicode
```

---

# 57. Temporal integration

Casos:

```text
date
time
instant
local datetime
timezone
precision
DST boundary
```

cuando sean aplicables.

---

# 58. Schema Integration

Pipeline:

```text
Schema Definition
      ↓
Schema Compiler
      ↓
Execution
      ↓
DBMS
      ↓
Schema Introspection
      ↓
Normalized Schema Model
```

---

# 59. Schema round-trip

Conceptualmente:

```text
Define S
 ↓
Apply S
 ↓
Introspect S'
```

Se verificará:

```text
Equivalent(S, S')
```

según capacidades de la plataforma.

---

# 60. Physical representation

No siempre será idéntica a la definición lógica.

Por eso:

```text
Logical Schema Equivalence
```

será más importante que igualdad textual del DDL.

---

# 61. Table integration

Se probarán:

```text
create
alter
rename where supported
drop
```

---

# 62. Column integration

Se cubrirán:

```text
types
nullable
default
generated
identity/autoincrement
collation
```

según plataforma.

---

# 63. Index integration

Incluyendo cuando corresponda:

```text
unique
composite
partial
expression
full-text
spatial
```

---

# 64. Foreign keys

Se deberá verificar tanto:

```text
metadata
```

como:

```text
actual enforcement
```

cuando el motor lo soporte.

---

# 65. Constraint integration

No bastará con que introspection reporte una constraint.

Podrá verificarse que el DBMS realmente la aplique.

---

# 66. Schema Introspection

Deberá compararse con el estado creado directamente en DBMS y por VoltStack.

---

# 67. External schema

Algunas pruebas deberán crear estructuras usando SQL/platform tooling fuera del Schema Builder y comprobar que VoltStack las introspecta correctamente.

Esto evita:

```text
compiler and introspector sharing same bug
```

---

# 68. Schema Diff Integration

Pipeline:

```text
Actual DB Schema A
      ↓ introspect
Model A
      +
Target Model B
      ↓
Diff
      ↓
Migration Operations
      ↓
Execute
      ↓
Introspect
      ↓
Model B'
```

Verificación:

```text
Equivalent(B, B')
```

---

# 69. Migration Integration

Se deberá verificar:

```text
discovery
ordering
execution
repository
batch
rollback
schema result
failure handling
```

---

# 70. Migration repository

Los registros reales deberán reflejar únicamente migraciones cuyo estado pueda afirmarse correctamente.

---

# 71. Failed migration

La suite deberá probar fallos intermedios.

---

# 72. DDL transaction differences

Las plataformas tienen diferentes semánticas respecto a DDL transaccional.

VoltStack no deberá asumir uniformidad.

---

# 73. Migration rollback

Deberá verificarse:

```text
migration up
 ↓
expected schema
 ↓
migration down
 ↓
expected previous schema
```

cuando exista reversibilidad.

---

# 74. Irreversible migration

Deberá ser explícita.

---

# 75. Zero-Downtime Migration

Las partes verificables en un entorno de integración podrán probar:

```text
expand
backfill
compatibility
switch
contract
```

pero pruebas completas podrán requerir system/runtime testing.

---

# 76. ORM Persistence Integration

Pipeline:

```text
Entity
 ↓
EntityManager
 ↓
UnitOfWork
 ↓
Persistence Planner
 ↓
Query Engine
 ↓
Executor
 ↓
DBMS
```

---

# 77. Insert persistence

Deberá verificar:

```text
generated ID
stored values
state transition
snapshot
IdentityMap
```

---

# 78. Update persistence

Verificará:

```text
dirty detection
generated UPDATE semantics
affected state
new snapshot
database value
```

---

# 79. Delete persistence

Verificará:

```text
scheduled remove
flush
database absence/state
IdentityMap reconciliation
```

---

# 80. flush() ≠ commit()

Las pruebas deberán mantener esta separación.

---

# 81. Transaction-owned flush

Ejemplo:

```text
BEGIN
persist
flush
ROLLBACK
```

deberá dejar:

```text
Database = unchanged
```

aunque el flush haya ejecutado SQL.

---

# 82. Object graph after rollback

La prueba deberá comprobar la política de VoltStack.

No deberá asumir:

```text
rollback
=
automatic PHP object rewind
```

---

# 83. IdentityMap integration

Ejemplo:

```text
find User 1
find User 1
```

dentro del mismo scope deberá devolver la misma instancia managed.

---

# 84. Scope reset

Después de:

```text
clear/reset/new operation
```

no deberá preservarse indebidamente la identidad managed anterior.

---

# 85. Hydration Integration

La suite deberá probar:

```text
raw result
 ↓
type conversion
 ↓
hydration
 ↓
IdentityMap
 ↓
managed entity
```

---

# 86. Dirty entity protection

Podrá probarse:

```text
load entity
modify without flush
execute another query returning same entity
```

y comprobar que hydration no destruye silenciosamente cambios locales según la política.

---

# 87. Partial hydration

Deberá comprobar:

```text
LoadedFieldMask
```

contra resultados reales.

---

# 88. Relationships Integration

Se probarán:

```text
one-to-one
one-to-many
many-to-one
many-to-many
polymorphic
```

---

# 89. Relationship persistence

La prueba deberá verificar owning side real.

---

# 90. Cascade behavior

Se deberán verificar:

```text
persist
remove
orphan handling
```

cuando estén configurados.

---

# 91. Lazy Loading

Deberá comprobarse con una conexión real que:

```text
access relationship
```

produce carga sólo cuando está permitido.

---

# 92. FORBID policy

No deberá ejecutar consulta.

---

# 93. Eager Loading

Se verificará que:

```text
availability requirement
```

se satisfaga sin alterar incorrectamente la cardinalidad del root result.

---

# 94. N+1 Integration

Se podrá ejecutar un escenario real y utilizar telemetría/query events para verificar la detección semántica.

---

# 95. N+1 detector ≠ query count alone

La suite no deberá reducirlo simplemente a:

```text
queries > N
```

---

# 96. Pagination Integration

Se probarán:

```text
offset pagination
cursor pagination
ordering
tie breakers
NULL ordering
forward
backward
```

---

# 97. Cursor stability

Deberán existir datasets con valores repetidos para verificar tie-breakers.

---

# 98. Mutation between pages

Podrá comprobarse explícitamente la semántica definida bajo mutaciones concurrentes.

---

# 99. Chunk Processing Integration

Se probarán:

```text
chunk boundaries
keyset progression
mutation
checkpoint
resume
```

---

# 100. Large Dataset

Las pruebas no deberán depender únicamente de datasets diminutos cuando se intenta validar comportamiento de chunking/streaming.

---

# 101. Bulk Operations Integration

Se probarán:

```text
bulk insert
bulk update
bulk delete
```

con tamaños que crucen límites de batch relevantes.

---

# 102. Parameter limits

El planner deberá respetar límites reales del DBMS/driver.

---

# 103. ORM coherence after bulk mutation

Ejemplo:

```text
managed User
 ↓
bulk update users
 ↓
managed User may be stale
```

La política configurada deberá verificarse.

---

# 104. Transaction Integration

Es una categoría crítica.

---

# 105. Basic transaction

```text
BEGIN
INSERT
COMMIT
```

deberá persistir.

---

# 106. Rollback

```text
BEGIN
INSERT
ROLLBACK
```

no deberá persistir.

---

# 107. Savepoints

Cuando sean soportados:

```text
BEGIN
INSERT A
SAVEPOINT S
INSERT B
ROLLBACK TO S
COMMIT
```

resultado esperado:

```text
A exists
B does not
```

---

# 108. Nested transaction policy

Se probarán estrategias:

```text
JOIN
SAVEPOINT
REJECT
REQUIRES_NEW
```

cuando correspondan.

---

# 109. Isolation Levels

Deberán utilizar múltiples conexiones reales.

---

# 110. Isolation test topology

```text
Connection A
    │
    ├── Transaction A
    │
DBMS
    │
    └── Transaction B
Connection B
```

---

# 111. Dirty read

Se podrá verificar si la plataforma/level correspondiente lo permite o evita.

---

# 112. Non-repeatable read

Deberá probarse cuando sea parte del contrato.

---

# 113. Phantom behavior

Igualmente cuando resulte relevante.

---

# 114. Effective isolation

La prueba deberá verificar:

```text
requested isolation
```

frente a:

```text
effective DBMS behavior/configuration
```

cuando sea posible.

---

# 115. Locking Integration

Se probarán:

```text
FOR UPDATE
shared locks
nowait
skip locked
```

sólo donde existan capacidades correspondientes.

---

# 116. Lock timeout

Deberá producirse mediante concurrencia real controlada.

---

# 117. Deadlock Integration

Podrá construirse:

```text
Tx A locks row 1
Tx B locks row 2
Tx A requests row 2
Tx B requests row 1
```

para verificar:

```text
DBMS error
 ↓
Driver mapping
 ↓
VoltStack deadlock classification
```

---

# 118. Deadlock determinism

Dado que los DBMS pueden elegir víctimas distintas, la prueba deberá afirmar la propiedad semántica y no una víctima arbitraria salvo garantía de plataforma.

---

# 119. Optimistic Locking

Escenario:

```text
A loads version 1
B loads version 1
A updates → version 2
B updates using version 1
```

Esperado:

```text
optimistic lock conflict
```

---

# 120. Pessimistic Locking

Deberá utilizar transacciones/conexiones reales.

---

# 121. Connection Lifecycle Integration

Se probará:

```text
connect
reuse
reset
release
disconnect
failure
```

---

# 122. Connection state reset

Después de devolver una conexión reutilizable deberán verificarse estados relevantes:

```text
transaction
session variables
temporary settings
read-only mode
isolation
```

según plataforma y contrato.

---

# 123. Poisoned connection

Una conexión en estado desconocido no deberá regresar al pool como saludable.

---

# 124. Connection Pooling

Cuando exista implementación real de pooling, deberá probarse:

```text
acquire
release
reuse
limit
timeout
reset
failure
```

---

# 125. Persistent Runtime Integration

Especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 126. Sequential operations

Escenario:

```text
Worker
 ├── Operation A / Tenant A
 └── Operation B / Tenant B
```

deberá demostrar ausencia de contaminación.

---

# 127. State leakage checks

Se verificará:

```text
IdentityMap
UnitOfWork
EntityManager
transaction
tenant
query context
telemetry
temporary connection state
```

---

# 128. FrankenPHP

Como runtime predeterminado, deberá existir una suite específica de integración.

---

# 129. FrankenPHP integration scenarios

```text
worker reuse
multiple requests
exception cleanup
transaction cleanup
connection reuse
tenant switching
memory behavior
```

---

# 130. RoadRunner

Su suite será equivalente en invariantes, adaptada al runtime.

---

# 131. OpenSwoole

Además deberá comprobar:

```text
coroutine isolation
concurrent contexts
connection ownership
```

---

# 132. Concurrency Integration

Las pruebas concurrentes deberán coordinarse mediante barreras explícitas.

No mediante:

```php
sleep(2);
```

como mecanismo principal.

---

# 133. Test Barrier

Conceptualmente:

```text
Tx A reaches point X
       ↓
Barrier
       ↓
Tx B performs Y
       ↓
Release A
```

---

# 134. Race tests

Podrán repetirse para aumentar probabilidad de descubrir defectos, pero deberán registrar seed/iteration/context.

---

# 135. Read/Write Integration

Requiere topología real o representativa.

```text
Writer
   ↓ replication
Replica
```

---

# 136. Read routing

Se comprobará que lecturas elegibles puedan ir a replica.

---

# 137. Write routing

Toda escritura deberá dirigirse al writer correspondiente.

---

# 138. Locking reads

Deberán seguir las reglas de writer affinity.

---

# 139. Sticky connection

Escenario:

```text
WRITE
 ↓
READ immediately
```

deberá cumplir la política de read-your-writes configurada.

---

# 140. Replica lag

Cuando pueda controlarse o simularse en una integración suficientemente real, se verificará que una replica retrasada deje de ser elegible según política.

---

# 141. Replica health ≠ eligibility

La suite deberá demostrar ambos conceptos separadamente.

---

# 142. Failover Integration

Cuando la infraestructura de CI lo permita:

```text
writer failure
 ↓
topology update
 ↓
new eligible writer
```

---

# 143. Active transaction

Nunca deberá migrarse transparentemente una transacción activa al nuevo writer fingiendo continuidad.

---

# 144. UNKNOWN outcome

Fallos durante:

```text
COMMIT
```

deberán conservar `UNKNOWN` cuando no pueda demostrarse el resultado.

---

# 145. Sharding Integration

Topología:

```text
Logical Database
├── Shard A
└── Shard B
```

---

# 146. Shard ownership

Los datos deberán aparecer únicamente en el shard resuelto.

---

# 147. Cross-shard query

Sólo deberá ejecutarse cuando el plan lo permita explícitamente.

---

# 148. Distributed ordering

Si se soporta fan-out:

```text
Shard A ─┐
         ├─ Merge → ordered result
Shard B ─┘
```

deberá verificarse el orden global.

---

# 149. No fake distributed transaction

La integración deberá confirmar que VoltStack no expone ACID global si no existe protocolo que realmente lo garantice.

---

# 150. Multitenancy Integration

Como paquete opcional:

```text
VoltStack Multitenancy
        ↓
Database Integration
```

---

# 151. Tenant database isolation

Escenario:

```text
Tenant A → DB A
Tenant B → DB B
```

y verificar ausencia de contaminación.

---

# 152. Schema isolation

Escenario:

```text
Tenant A → schema_a
Tenant B → schema_b
```

---

# 153. Shared schema isolation

Cuando corresponda:

```text
tenant_id
```

deberá aplicarse correctamente en el contexto definido.

---

# 154. Tenant switch persistent runtime

Particularmente importante:

```text
Operation 1 → Tenant A
Operation 2 → Tenant B
```

sobre el mismo worker.

---

# 155. Cache Integration

Podrán existir:

```text
real cache provider
```

tests cuando se pretenda validar el adapter.

---

# 156. Database semantics without cache provider

La lógica central de cache policy seguirá siendo unitaria.

---

# 157. Result Cache Integration

Se comprobará:

```text
miss
populate
hit
invalidate
```

contra un provider real cuando corresponda.

---

# 158. Commit-aware invalidation

```text
BEGIN
mutation
cache remains unpublished
COMMIT
invalidate
```

según estrategia.

---

# 159. Rollback

No deberá publicar un estado mutado que nunca se confirmó.

---

# 160. UNKNOWN commit

Deberá adoptar la estrategia conservadora definida.

---

# 161. Event Integration

Se comprobará la integración real con Event System sin convertir eventos en semántica de DB.

---

# 162. Transaction events

Orden conceptual:

```text
TransactionStarted
...
CommitAttempted
...
TransactionCommitted
```

sólo cuando el commit pueda afirmarse.

---

# 163. UNKNOWN

No deberá emitir:

```text
TransactionCommitted
```

como hecho cierto si el resultado es desconocido.

---

# 164. afterCommit

Un listener fallido después de commit:

```text
does not rollback committed DB transaction
```

---

# 165. Telemetry Integration

Se deberá comprobar:

```text
trace correlation
query spans
transaction spans
connection metrics
ORM metrics
```

con un backend/adaptador real cuando sea necesario.

---

# 166. Telemetry failure

No deberá alterar el resultado funcional de Database salvo una política explícita excepcional.

---

# 167. Sensitive telemetry

Los tests deberán verificar que parámetros clasificados como sensibles no aparezcan en exporters reales.

---

# 168. Backup Integration

Pipeline:

```text
Database
 ↓
Backup Provider
 ↓
Native Tool/API
 ↓
Backup Artifact
 ↓
Verification
```

---

# 169. Backup artifact

La prueba deberá verificar:

```text
exists
manifest valid
parts complete
checksums valid
status complete
```

---

# 170. Backup created ≠ restorable

La integración de backup deberá complementarse periódicamente con restore real.

---

# 171. Logical backup

Deberá probarse contra DB real y herramienta/provider real.

---

# 172. Physical backup

Sólo donde la plataforma/entorno soporte una estrategia segura.

---

# 173. Snapshot backup

Una copia arbitraria de archivos no contará como prueba de backup válido.

---

# 174. Restore Integration

Pipeline:

```text
Verified Backup
      ↓
Clean Target
      ↓
Restore
      ↓
Database
      ↓
Verification
```

---

# 175. Restore verification

Podrá comprobar:

```text
schema
row counts
checksums
known records
constraints
metadata
application probes
```

---

# 176. PITR

Cuando se soporte:

```text
Base Backup
+
Log Stream
+
Recovery Target
```

deberá probarse en entorno controlado.

---

# 177. Maintenance Integration

Operaciones como:

```text
analyze
vacuum
optimize
reindex
check
```

se probarán únicamente donde correspondan a capacidades reales.

---

# 178. Maintenance safety

Nunca deberán ejecutarse indiscriminadamente sobre servidores no marcados como test.

---

# 179. Health Check Integration

Se probará contra:

```text
healthy server
unreachable server
invalid credentials
read-only state
degraded replica
```

según capacidades.

---

# 180. Health ≠ business correctness

Que:

```text
SELECT 1
```

funcione no significa que toda la base esté correctamente configurada.

---

# 181. Diagnostics Integration

Podrá recibir evidencia real de:

```text
connections
capabilities
schema
server metadata
health
topology
```

y verificar diagnósticos.

---

# 182. Administration Integration

Operaciones administrativas reales deberán estar:

```text
isolated
authorized
explicit
audited
destructive-safe
```

---

# 183. Failure Injection Architecture

La integración deberá soportar fallos controlados.

```text
Integration Scenario
        ↓
Failure Injector
        ↓
Real Boundary
        ↓
Observed Recovery
```

---

# 184. Failure categories

```text
connection refused
connection dropped
server restart
timeout
deadlock
lock timeout
disk/storage error
process failure
partial backup
network interruption
invalid credentials
permission denied
```

---

# 185. Failure injection realism

Un proxy/fault injector real puede ser necesario para determinadas propiedades.

Un mock no demostrará recuperación de red real.

---

# 186. Destructive failure tests

Podrán estar separados de la suite estándar.

Ejemplo:

```text
integration-failure
```

---

# 187. Retry Integration

Debe verificarse:

```text
failure
 ↓
classification
 ↓
retry policy
 ↓
new attempt
 ↓
result
```

---

# 188. Retry observability

La prueba deberá poder afirmar:

```text
number of attempts
failure type
final outcome
```

sin depender de sleeps.

---

# 189. Unknown outcome tests

Son especialmente importantes y pueden requerir infraestructura especializada.

---

# 190. Cleanup Architecture

Cada test deberá terminar con:

```text
Known Clean State
```

o destruir completamente su entorno.

---

# 191. Cleanup after success

Obligatorio.

---

# 192. Cleanup after failure

También obligatorio.

---

# 193. Cleanup after exception

También.

---

# 194. Cleanup after timeout

El runner deberá intentar reconciliar/destruir recursos.

---

# 195. Cleanup failure

No deberá ocultarse.

---

# 196. Test status

Podrá existir:

```text
PASSED
FAILED
SKIPPED
INCONCLUSIVE
ENVIRONMENT_FAILURE
CLEANUP_FAILURE
```

---

# 197. Test failure ≠ environment failure

Ejemplo:

```text
expected 5 rows, got 4
```

es test failure.

Mientras:

```text
PostgreSQL container failed to start
```

es environment failure.

---

# 198. Unsupported capability

Si una prueba requiere una capability legítimamente no soportada:

```text
SKIPPED_NOT_SUPPORTED
```

podrá ser correcto.

---

# 199. UNKNOWN capability

No deberá transformarse automáticamente en:

```text
SKIPPED_NOT_SUPPORTED
```

---

# 200. Capability-gated integration tests

Ejemplo conceptual:

```php
$this->requiresCapability(
    'transaction.savepoint'
);
```

---

# 201. Capability discovery verification

Algunas pruebas deberán verificar el propio proceso de discovery.

No podrán utilizar el snapshot que pretenden comprobar como única fuente de verdad.

---

# 202. External ground truth

Podrán utilizarse:

```text
native server metadata
known server configuration
direct behavior probe
```

para validar discovery.

---

# 203. Integration test metadata

Cada prueba podrá declarar:

```text
platform
driver
required capabilities
resources
isolation
timeout
destructive
parallel-safe
```

---

# 204. Example

```php
#[DatabaseIntegrationTest]
#[RequiresPlatform('postgresql')]
#[RequiresCapability('transaction.savepoint')]
#[Isolation('database')]
final class PostgreSqlSavepointTest
{
}
```

La sintaxis concreta podrá evolucionar.

---

# 205. Test grouping

```text
integration-query
integration-schema
integration-orm
integration-transaction
integration-runtime
integration-distributed
integration-backup
integration-failure
```

---

# 206. Fast integration suite

Podrá existir un subconjunto para feedback rápido.

---

# 207. Full integration suite

Se ejecutará en CI más completa.

---

# 208. Nightly matrix

Pruebas costosas como:

```text
multiple versions
failover
replication
backup/restore
PITR
large dataset
```

podrán ejecutarse en pipelines periódicos.

---

# 209. Parallel execution

Será permitida cuando el aislamiento lo garantice.

---

# 210. Unique resources

Cada worker deberá utilizar:

```text
unique database/schema/resource namespace
```

cuando corresponda.

---

# 211. Port collision

Los entornos deberán evitar asumir un puerto global único cuando se paralelicen.

---

# 212. TestRunId

Podrá formar parte de nombres temporales:

```text
vs_test_<run>_<worker>_<case>
```

---

# 213. Reproducibility

Una falla deberá registrar suficiente contexto:

```text
platform
server version
driver version
capabilities
test seed
schema generation
test case
environment profile
```

---

# 214. Secrets

Nunca deberán aparecer:

```text
password
token
private key
full credential URI
```

---

# 215. Integration Evidence

Podrá modelarse:

```php
final readonly class IntegrationTestEvidence
{
    public function __construct(
        public TestId $test,
        public PlatformDescriptor $platform,
        public DriverDescriptor $driver,
        public CapabilityFingerprint $capabilities,
        public TestStatus $status,
        public EvidenceScope $scope,
        public ?FailureEvidence $failure,
    ) {}
}
```

---

# 216. Evidence scope

Ejemplo:

```text
Verified:

PostgreSQL 17.x
PDO PgSQL
savepoint rollback behavior
specific test environment
```

No:

```text
All PostgreSQL versions always work
```

---

# 217. Environment fingerprint

Podrá incluir:

```text
DBMS
version
driver
OS
architecture
runtime
relevant extensions
configuration fingerprint
```

---

# 218. Configuration relevance

No se almacenará toda la configuración si no es relevante.

Se registrarán únicamente elementos necesarios para reproducibilidad/diagnóstico.

---

# 219. Integration assertions

Podrán clasificarse:

```text
Result Assertion
State Assertion
Metadata Assertion
Side-Effect Assertion
Error Assertion
Resource Assertion
Isolation Assertion
```

---

# 220. Result assertion

Ejemplo:

```text
query returned expected values
```

---

# 221. State assertion

Ejemplo:

```text
row persisted after commit
```

---

# 222. Metadata assertion

Ejemplo:

```text
column introspected as nullable integer
```

---

# 223. Side-effect assertion

Ejemplo:

```text
cache invalidated after commit
```

---

# 224. Error assertion

Ejemplo:

```text
unique violation mapped correctly
```

---

# 225. Resource assertion

Ejemplo:

```text
cursor closed after early break
```

---

# 226. Isolation assertion

Ejemplo:

```text
Tenant B cannot observe Tenant A state
```

---

# 227. Eventual systems

Replicas y sistemas distribuidos pueden requerir eventual consistency.

---

# 228. No arbitrary sleep

Incorrecto:

```php
sleep(5);
$this->assertReplicaHasData();
```

---

# 229. Condition waiting

Preferible:

```text
wait until condition
with bounded deadline
```

---

# 230. Deadline

Toda espera deberá tener límite.

---

# 231. Timeout result

Si no se cumple:

```text
FAILED
```

o:

```text
ENVIRONMENT_FAILURE
```

según causa demostrable.

---

# 232. Polling diagnostics

Al fallar deberá conservar evidencia de los últimos estados observados.

---

# 233. Concurrency barriers

Las pruebas concurrentes deberán coordinar fases explícitas.

---

# 234. Integration testing of UNKNOWN

VoltStack deberá tener escenarios donde deliberadamente no pueda demostrar el resultado.

El sistema deberá preservar:

```text
UNKNOWN
```

en lugar de fabricar certeza.

---

# 235. Example unknown commit

Conceptualmente:

```text
Client
  │
COMMIT
  │
  ▼
DBMS commits
  │
network breaks before acknowledgement
  X
Client
```

VoltStack puede no saber si ocurrió el commit.

---

# 236. Required behavior

```text
UNKNOWN
```

No:

```text
ROLLED_BACK
```

No:

```text
COMMITTED
```

sin evidencia.

---

# 237. Recovery integration

El sistema deberá probar mecanismos de:

```text
reconciliation
connection discard
state taint
operator diagnostics
```

cuando corresponda.

---

# 238. Persistent state reset

Después de una falla:

```text
Operation A fails
     ↓
reset
     ↓
Operation B
```

B no deberá heredar estado inválido.

---

# 239. Memory leak integration

Suites prolongadas podrán ejecutar múltiples operaciones para detectar crecimiento anormal.

---

# 240. Memory test ≠ benchmark

Su objetivo puede ser detectar:

```text
unbounded retained state
```

sin medir performance absoluta.

---

# 241. Resource cleanup

Se deberá observar:

```text
open connections
open cursors
transactions
temporary files
child processes
```

según escenario.

---

# 242. Integration test architecture

Propuesta:

```text
tests/Quantum/Database/Integration/
├── Connection/
├── Query/
├── CompilerExecution/
├── Result/
├── Type/
├── Schema/
├── Migration/
├── ORM/
├── Hydration/
├── Relationship/
├── Transaction/
├── Locking/
├── Concurrency/
├── Cache/
├── Events/
├── Telemetry/
├── ReadWrite/
├── Replication/
├── Sharding/
├── Multitenancy/
├── Runtime/
│   ├── FrankenPHP/
│   ├── RoadRunner/
│   └── OpenSwoole/
├── Backup/
├── Restore/
├── Maintenance/
├── Health/
├── Diagnostics/
├── Administration/
├── Failure/
└── Security/
```

---

# 243. Testing support namespace

```text
src/Quantum/Database/Testing/Integration/
├── Contract/
│   ├── IntegrationEnvironment.php
│   ├── IntegrationResource.php
│   └── IntegrationScenario.php
│
├── Environment/
│   ├── EnvironmentManager.php
│   ├── EnvironmentProfile.php
│   └── EnvironmentFingerprint.php
│
├── Database/
│   ├── TestDatabase.php
│   ├── TestDatabaseProvisioner.php
│   ├── TestDatabaseResetter.php
│   └── TestDatabaseGuard.php
│
├── Platform/
│   ├── PlatformTestProfile.php
│   └── PlatformMatrix.php
│
├── Capability/
│   ├── CapabilityRequirement.php
│   └── CapabilityTestGate.php
│
├── Concurrency/
│   ├── TestBarrier.php
│   ├── ConcurrentScenario.php
│   └── ConcurrentWorker.php
│
├── Failure/
│   ├── FailureInjector.php
│   └── FailureScenario.php
│
├── Assertion/
│   ├── DatabaseAssertions.php
│   ├── SchemaAssertions.php
│   ├── TransactionAssertions.php
│   └── ResourceAssertions.php
│
└── Evidence/
    ├── IntegrationTestEvidence.php
    └── EvidenceScope.php
```

---

# 244. Environment adapters

El core Testing no deberá depender directamente de:

```text
Docker
Podman
Kubernetes
GitHub Actions
specific cloud
```

---

# 245. Environment provider

Podrá existir:

```php
interface DatabaseTestEnvironmentProvider
{
    public function provision(
        TestEnvironmentRequest $request
    ): TestEnvironment;

    public function destroy(
        TestEnvironment $environment
    ): void;
}
```

---

# 246. Local environment

Un desarrollador podrá utilizar una instalación local compatible.

---

# 247. Container environment

CI podrá utilizar contenedores.

---

# 248. Remote environment

También podrá existir un servidor de pruebas remoto explícitamente autorizado.

---

# 249. Environment semantics

La fuente del entorno no deberá alterar las assertions semánticas.

---

# 250. Container ≠ requirement

VoltStack Database Integration Testing no deberá exigir conceptualmente Docker.

---

# 251. Environment discovery

El runner podrá descubrir qué perfiles están disponibles.

Ejemplo:

```text
postgresql-17
mysql-8
mariadb-11
sqlite-local
```

---

# 252. Missing environment

Resultado:

```text
SKIPPED_ENVIRONMENT_UNAVAILABLE
```

cuando la política lo permita.

---

# 253. Required CI environment

En CI oficial, un entorno obligatorio ausente deberá considerarse:

```text
ENVIRONMENT_FAILURE
```

y no un skip silencioso.

---

# 254. Test Database Guard

Antes de operaciones destructivas:

```text
Request
   ↓
Environment Guard
   ↓
Identity Validation
   ↓
Safety Policy
   ↓
ALLOW / REJECT
```

---

# 255. Production hostname guard

Podrán configurarse patrones explícitamente prohibidos.

---

# 256. Database name guard

Ejemplo:

```text
must start with:
voltstack_test_
```

para determinados perfiles.

---

# 257. Credential privilege

Las credenciales de integración deberán tener sólo los privilegios necesarios para la suite correspondiente.

---

# 258. Administration tests

Podrán requerir un perfil separado con mayores privilegios.

---

# 259. Principle of least privilege

No se utilizará:

```text
superuser/root
```

para toda la suite simplemente por conveniencia.

---

# 260. Security boundaries

Se podrán crear perfiles:

```text
runtime-role
migration-role
backup-role
admin-role
read-only-role
```

para verificar el permission model.

---

# 261. Permission Integration

Ejemplo:

```text
runtime role
  ↓
SELECT allowed
DROP DATABASE rejected
```

---

# 262. Migration role

Podrá verificar privilegios adicionales necesarios para DDL.

---

# 263. Backup role

Deberá probar el mínimo requerido para el provider correspondiente.

---

# 264. Read-only role

Una escritura deberá fallar de manera correctamente clasificada.

---

# 265. Sensitive data isolation

Fixtures deberán utilizar datos sintéticos.

---

# 266. No production dump

Queda prohibido como default:

```text
download production backup
 ↓
run tests
```

---

# 267. Sanitized dataset

Si algún proyecto necesita datasets representativos, deberá utilizar datos sintéticos o sanitizados mediante proceso separado y explícito.

---

# 268. Integration fixtures

Podrán utilizar:

```text
Factories
Fixtures
Seeders
```

definidos en los documentos 193–198.

---

# 269. Fixture determinism

El mismo seed deberá producir el mismo escenario lógico.

---

# 270. Database-generated values

IDs/sequences pueden variar según estrategia de reset.

Las pruebas no deberán depender de valores accidentales salvo que sean la propiedad evaluada.

---

# 271. Sequence reset

Cuando la estrategia lo requiera, deberá ser explícita.

---

# 272. Collation integration

Deberán existir pruebas específicas cuando:

```text
case sensitivity
sorting
Unicode
```

sean relevantes.

---

# 273. Unicode

La suite deberá incluir datos como:

```text
áéíóú
ñ
中文
日本語
emoji
```

según capacidades del tipo/plataforma.

---

# 274. Empty vs NULL

Deberá verificarse:

```text
''
≠
NULL
```

cuando el DBMS lo preserve.

---

# 275. Large values

Deberán probarse límites representativos de:

```text
TEXT
BLOB
JSON
parameters
rows
```

sin convertir toda la suite estándar en benchmark.

---

# 276. Platform-specific integration

Se permitirán tests específicos:

```text
PostgreSQL/
MySQL/
MariaDB/
SQLite/
```

cuando prueben capacidades genuinamente particulares.

---

# 277. Shared contract tests

Cuando la semántica sea portable:

```text
SharedIntegrationContract
```

deberá ejecutarse contra todas las plataformas aplicables.

---

# 278. Platform exception

Una diferencia legítima deberá declararse como capability/semantic difference.

No mediante:

```php
if ($platform === 'mysql') {
    // mysterious exception
}
```

sin explicación arquitectónica.

---

# 279. Capability-driven test selection

Preferible:

```text
requires json.path.query
```

sobre:

```text
if PostgreSQL
```

cuando la propiedad sea una capacidad.

---

# 280. Platform identity still matters

No se eliminará la identidad de plataforma cuando la prueba evalúe específicamente:

```text
PostgreSQL compiler
MariaDB introspector
MySQL driver
```

---

# 281. Integration test diagnostics

Ante fallo deberá reportarse:

```text
test
platform
driver
version
capabilities
query fingerprint
schema generation
transaction state
relevant native error
```

---

# 282. SQL logging

SQL podrá incluirse según política.

Bindings sensibles deberán redactarse.

---

# 283. Environment diagnostics

Podrá incluir:

```text
container/service status
connection status
extension list
server settings relevant to failure
```

---

# 284. Failure artifact

CI podrá conservar:

```text
logs
test report
schema snapshot
capability snapshot
diagnostic report
```

sin secretos.

---

# 285. Test reproducibility command

El sistema podrá producir conceptualmente:

```text
volt database:test:integration \
    --profile=postgresql-17 \
    --test=<id>
```

---

# 286. Reproduction metadata

Un fallo podrá generar:

```text
EnvironmentProfile
TestSeed
TestId
CapabilityFingerprint
```

---

# 287. Integration suite CLI

Futuras herramientas podrán incluir:

```text
volt database:test:integration
volt database:test:integration --platform=postgresql
volt database:test:integration --group=transaction
volt database:test:integration --profile=full
```

---

# 288. Destructive confirmation

Tests destructivos sólo podrán ejecutarse contra perfiles previamente autorizados.

---

# 289. CI noninteractive mode

La autorización deberá provenir de configuración segura del environment profile, no de prompts interactivos.

---

# 290. Local interactive safeguards

Localmente podrán existir confirmaciones adicionales.

---

# 291. Integration Test State Machine

```text
DECLARED
   ↓
ENVIRONMENT_RESOLVED
   ↓
PROVISIONING
   ↓
READY
   ↓
PREPARING
   ↓
RUNNING
   ↓
VERIFYING
   ↓
CLEANING
   ↓
PASSED
```

Errores:

```text
PROVISION_FAILED
TEST_FAILED
INCONCLUSIVE
ENVIRONMENT_FAILED
CLEANUP_FAILED
CANCELLED
```

---

# 292. Cleanup status precedence

Un test funcionalmente exitoso pero con cleanup fallido no deberá reportarse simplemente como:

```text
PASSED
```

---

# 293. Cancellation

Una cancelación deberá intentar cleanup seguro.

---

# 294. Interrupted CI

El environment provider deberá facilitar garbage collection de recursos abandonados.

---

# 295. Resource tagging

Recursos externos podrán etiquetarse con:

```text
test run
created at
owner
expiry
```

cuando la infraestructura lo soporte.

---

# 296. TTL ≠ cleanup guarantee

La expiración automática será protección adicional.

No sustituirá cleanup explícito.

---

# 297. Test resource governance

Las suites deberán respetar límites de:

```text
CPU
RAM
disk
connections
processes
network
duration
parallel workers
```

---

# 298. Integration timeout

Cada test/grupo deberá disponer de deadline apropiado.

---

# 299. Timeout classification

Un timeout podrá representar:

```text
expected database timeout
test timeout
environment timeout
```

y deberán distinguirse.

---

# 300. Benchmark separation

Integration tests no serán benchmarks.

---

# 301. Performance assertions

Sólo deberán establecer límites amplios cuando sean parte de correctness operacional.

Los benchmarks formales pertenecen a:

```text
293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

---

# 302. CI Architecture

```text
                 CI Matrix
                    │
      ┌─────────────┼──────────────┐
      ▼             ▼              ▼
   MySQL         MariaDB       PostgreSQL
      │             │              │
      └─────────────┼──────────────┘
                    │
                  SQLite
                    │
                    ▼
          Shared Contract Suites
                    +
         Platform-Specific Suites
                    │
                    ▼
             Evidence Report
```

---

# 303. Layered CI

Propuesta:

```text
PR
├── Unit
└── Fast Integration

Main
├── Unit
├── Full Integration
└── Conformance Core

Nightly
├── Version Matrix
├── Replication
├── Failover
├── Failure Injection
├── Backup/Restore
├── Runtime Matrix
└── Performance
```

---

# 304. CI policy

La distribución exacta podrá evolucionar sin modificar los contratos arquitectónicos.

---

# 305. Integration invariants

## DB-INT-001

Integration Testing verificará fronteras reales.

## DB-INT-002

MockConnection no será evidencia de conexión real.

## DB-INT-003

Fake DB no será evidencia de DBMS real.

## DB-INT-004

SQLite no sustituirá universalmente otras plataformas.

## DB-INT-005

MySQL no será MariaDB.

## DB-INT-006

Platform no será Driver.

## DB-INT-007

Version no será Capability.

## DB-INT-008

Integration no será Conformance automáticamente.

## DB-INT-009

Integration no requerirá HTTP salvo necesidad específica.

## DB-INT-010

Test DB no será production DB.

## DB-INT-011

Production data no será fixture por defecto.

## DB-INT-012

Toda operación destructiva tendrá environment guard.

## DB-INT-013

Environment marker aislado no será protección suficiente para operaciones críticas.

## DB-INT-014

Cada test tendrá aislamiento definido.

## DB-INT-015

Transaction-per-test no será universal.

## DB-INT-016

Tests de transaction semantics podrán controlar sus propias transacciones.

## DB-INT-017

Schema bootstrap no ocultará migration bugs.

## DB-INT-018

Query execution se verificará contra DB real.

## DB-INT-019

SQL accepted no significará semantic correctness.

## DB-INT-020

Bindings serán probados contra drivers reales.

## DB-INT-021

Identifier handling será distinto de value binding.

## DB-INT-022

Result cursors tendrán lifecycle tests.

## DB-INT-023

Streaming tendrá resource cleanup tests.

## DB-INT-024

Timeout real será distinto de fake timeout.

## DB-INT-025

Native errors se mapearán estructuradamente cuando sea posible.

## DB-INT-026

Type round-trip tendrá integración real.

## DB-INT-027

Decimal exacto no se degradará silenciosamente a float.

## DB-INT-028

SQL NULL será distinto de JSON null.

## DB-INT-029

Temporal precision differences serán explícitas.

## DB-INT-030

Schema creation podrá verificarse por introspection.

## DB-INT-031

Logical schema equivalence será preferida a DDL textual equality.

## DB-INT-032

Schema introspector tendrá tests contra schemas externos.

## DB-INT-033

Compiler e introspector no serán la única evidencia uno del otro.

## DB-INT-034

Schema diff podrá verificarse mediante apply + introspect.

## DB-INT-035

Migration repository reflejará estados demostrables.

## DB-INT-036

Failed migration tendrá integration tests.

## DB-INT-037

DDL transaction semantics serán platform-aware.

## DB-INT-038

Irreversible migration será explícita.

## DB-INT-039

ORM persistence atravesará el pipeline real.

## DB-INT-040

persist no será INSERT.

## DB-INT-041

flush no será commit.

## DB-INT-042

Rollback no será automatic object graph rewind.

## DB-INT-043

IdentityMap será verificada en integración.

## DB-INT-044

Hydration respetará managed identity.

## DB-INT-045

Dirty entity no será sobrescrita silenciosamente.

## DB-INT-046

Partial hydration conservará LoadedFieldMask.

## DB-INT-047

Relationship ownership tendrá verificación real.

## DB-INT-048

Cascade behavior será explícito.

## DB-INT-049

FORBID lazy loading no ejecutará query.

## DB-INT-050

Eager loading preservará root semantics.

## DB-INT-051

N+1 integration no será simple query counting.

## DB-INT-052

Cursor pagination tendrá tie-breaker tests.

## DB-INT-053

Chunk processing tendrá progression tests.

## DB-INT-054

Bulk operations respetarán límites reales.

## DB-INT-055

Bulk mutation no fingirá ORM coherence.

## DB-INT-056

Commit persistirá.

## DB-INT-057

Rollback no persistirá.

## DB-INT-058

Savepoint semantics serán verificadas cuando soportadas.

## DB-INT-059

Nested transaction policy será explícita.

## DB-INT-060

Isolation tests utilizarán múltiples conexiones.

## DB-INT-061

Effective isolation podrá diferir de requested isolation.

## DB-INT-062

Locking tests utilizarán concurrencia real.

## DB-INT-063

Deadlock synthetic unit test no sustituirá deadlock integration.

## DB-INT-064

Deadlock victim no se asumirá arbitrariamente.

## DB-INT-065

Optimistic locking tendrá concurrent update test.

## DB-INT-066

Pessimistic locking tendrá real transaction test.

## DB-INT-067

Connection reset será verificado.

## DB-INT-068

Unknown/poisoned connection no regresará al pool.

## DB-INT-069

Persistent worker no conservará request state.

## DB-INT-070

FrankenPHP tendrá suite dedicada.

## DB-INT-071

RoadRunner tendrá adapter suite.

## DB-INT-072

OpenSwoole tendrá coroutine isolation tests.

## DB-INT-073

Concurrency tests utilizarán barriers, no sleeps como sincronización principal.

## DB-INT-074

Writes se dirigirán al writer.

## DB-INT-075

Replica health no será replica eligibility.

## DB-INT-076

Sticky reads respetarán política configurada.

## DB-INT-077

Active transaction no migrará transparentemente durante failover.

## DB-INT-078

Unknown commit outcome permanecerá UNKNOWN.

## DB-INT-079

Shard ownership será verificable.

## DB-INT-080

Cross-shard execution será explícita.

## DB-INT-081

No se fingirá distributed ACID.

## DB-INT-082

Tenant isolation tendrá tests reales.

## DB-INT-083

Tenant context no se filtrará entre operaciones persistentes.

## DB-INT-084

Cache integration no alterará database truth.

## DB-INT-085

Uncommitted data no se publicará como committed cache state.

## DB-INT-086

Rollback no publicará invalidación/población incorrecta.

## DB-INT-087

UNKNOWN commit tendrá cache policy conservadora.

## DB-INT-088

Database events no redefinirán transaction outcome.

## DB-INT-089

afterCommit failure no deshará commit.

## DB-INT-090

Telemetry failure no cambiará semantics por defecto.

## DB-INT-091

Sensitive bindings no aparecerán en telemetry.

## DB-INT-092

Backup artifact creation no implicará restore success.

## DB-INT-093

Backup verification tendrá integración real.

## DB-INT-094

Restore utilizará target controlado.

## DB-INT-095

PITR se probará donde sea capability soportada.

## DB-INT-096

Maintenance operations estarán capability-gated.

## DB-INT-097

Health check simple no implicará business correctness.

## DB-INT-098

Diagnostics utilizará evidencia real sin inventar certeza.

## DB-INT-099

Administration tests estarán aislados y protegidos.

## DB-INT-100

Failure injection distinguirá mock failure de real boundary failure.

## DB-INT-101

Retry integration verificará número real de intentos.

## DB-INT-102

Unknown outcome no se reintentará ciegamente.

## DB-INT-103

Cleanup ocurrirá tras éxito.

## DB-INT-104

Cleanup ocurrirá tras fallo.

## DB-INT-105

Cleanup ocurrirá tras excepción.

## DB-INT-106

Cleanup failure será visible.

## DB-INT-107

Test failure será distinto de environment failure.

## DB-INT-108

Unsupported será distinto de UNKNOWN.

## DB-INT-109

Capability-gated tests utilizarán capability semantics.

## DB-INT-110

Capability discovery no se validará circularmente usando sólo su propio resultado.

## DB-INT-111

Required CI environment missing será failure, no silent skip.

## DB-INT-112

Integration evidence tendrá scope explícito.

## DB-INT-113

Evidence de una versión no demostrará todas las versiones.

## DB-INT-114

Environment fingerprint facilitará reproducción.

## DB-INT-115

Secrets no estarán en evidence reports.

## DB-INT-116

Eventual consistency waits tendrán deadlines.

## DB-INT-117

Arbitrary sleeps no serán synchronization strategy principal.

## DB-INT-118

Polling failure conservará evidencia diagnóstica.

## DB-INT-119

UNKNOWN será un resultado válido cuando corresponda.

## DB-INT-120

Recovery tests verificarán state taint/reset.

## DB-INT-121

Resource leaks serán detectables.

## DB-INT-122

Environment provider no estará acoplado obligatoriamente a Docker.

## DB-INT-123

Local, container y remote test environments podrán coexistir.

## DB-INT-124

Environment origin no alterará semantic contract.

## DB-INT-125

Test DB credentials seguirán least privilege.

## DB-INT-126

Admin privileges no serán usados por toda la suite.

## DB-INT-127

Runtime/migration/backup/admin roles podrán probarse separadamente.

## DB-INT-128

Synthetic data será default.

## DB-INT-129

Database-generated identifiers no se asumirán estables accidentalmente.

## DB-INT-130

Unicode tendrá integración representativa.

## DB-INT-131

Empty string será distinto de NULL cuando la plataforma lo preserve.

## DB-INT-132

Platform-specific tests serán permitidos.

## DB-INT-133

Portable semantics utilizarán shared contracts.

## DB-INT-134

Capability differences no se ocultarán como vendor hacks.

## DB-INT-135

Failure reports tendrán contexto reproducible.

## DB-INT-136

SQL diagnostics respetarán redaction.

## DB-INT-137

Test resources podrán tener TTL, pero TTL no sustituirá cleanup.

## DB-INT-138

Integration tests tendrán resource budgets.

## DB-INT-139

Database timeout será distinto de test-runner timeout.

## DB-INT-140

Integration tests no serán performance benchmarks.

## DB-INT-141

Fast y full integration suites podrán coexistir.

## DB-INT-142

Costly topology tests podrán ejecutarse periódicamente.

## DB-INT-143

Parallel tests utilizarán recursos aislados.

## DB-INT-144

TestRunId podrá formar parte de recursos temporales.

## DB-INT-145

Interrupted runs deberán poder reconciliar recursos abandonados.

## DB-INT-146

Environment state será conocido antes de iniciar assertions.

## DB-INT-147

Provision failure no será product failure.

## DB-INT-148

Cleanup failure podrá invalidar un resultado aparentemente exitoso.

## DB-INT-149

Cancellation intentará cleanup.

## DB-INT-150

Integration result será evidence, no garantía universal.

## DB-INT-151

Unit evidence y integration evidence permanecerán diferenciados.

## DB-INT-152

Integration evidence y conformance evidence permanecerán diferenciados.

## DB-INT-153

Platform + Driver + Configuration definirán parte del contexto probado.

## DB-INT-154

Capabilities efectivas formarán parte del contexto cuando sean relevantes.

## DB-INT-155

Los tests no fabricarán soporte para una capability ausente.

## DB-INT-156

Los tests no transformarán UNKNOWN en UNSUPPORTED sin evidencia.

## DB-INT-157

Una conexión exitosa no demostrará todas las operaciones.

## DB-INT-158

Una consulta exitosa no demostrará transaction correctness.

## DB-INT-159

Un backup exitoso no demostrará restore correctness.

## DB-INT-160

VoltStack deberá probar los límites donde realmente promete comportamiento.

---

# 306. Anti-patrones

## 306.1 SQLite como sustituto universal

```text
All tests pass on SQLite
→ therefore PostgreSQL/MySQL/MariaDB supported
```

Incorrecto.

---

## 306.2 Mock como integración

```text
MockConnection
    ↓
return expected result
```

No demuestra integración.

---

## 306.3 Test contra producción

Nunca como workflow normal.

---

## 306.4 Un único DB compartido

Genera:

```text
state contamination
parallel conflicts
order dependency
flakiness
```

---

## 306.5 `sleep()` como coordinación

Especialmente incorrecto para locking/concurrency/replication.

---

## 306.6 Ocultar failures como skips

Un entorno obligatorio roto no deberá aparecer como suite verde.

---

## 306.7 Root para todo

Oculta problemas reales de permisos.

---

## 306.8 Sólo comprobar SQL

```text
compiled SQL looks right
```

no es integración.

---

## 306.9 Sólo comprobar exit code

Especialmente peligroso para:

```text
backup
restore
maintenance
```

---

## 306.10 Ignorar cleanup failures

Puede contaminar suites posteriores.

---

## 306.11 Reintentar tests flaky hasta que pasen

Oculta defectos.

---

## 306.12 Version conditionals everywhere

```php
if ($version >= ...)
```

cuando la decisión real depende de capability.

---

## 306.13 Compartir EntityManager entre tests

Viola aislamiento.

---

## 306.14 Compartir tenant context

Especialmente peligroso en workers persistentes.

---

## 306.15 Suponer commit después de pérdida de conexión

Debe preservarse UNKNOWN.

---

# 307. Modelo formal

Sea:

```text
V = VoltStack implementation
E = Real external environment
I = Controlled initial state
A = Action
O = Observed external state
P = Expected property
```

Una integración ejecuta:

```text
O = Execute(V, E, I, A)
```

y comprueba:

```text
P(O, E) = true
```

---

# 308. Evidencia contextual

La evidencia deberá asociarse a:

```text
Evidence
=
Property
+
Platform
+
Driver
+
Relevant Configuration
+
Capabilities
+
Environment
+
Test Version
```

---

# 309. No universal inference

De:

```text
Pass(PostgreSQL 17, PDO_PgSQL, Test X)
```

no se deduce automáticamente:

```text
Pass(All PostgreSQL, All Drivers, All Configurations, X)
```

---

# 310. Relación con Unit Testing

```text
             Database Feature
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
      Unit Test          Integration Test
         │                     │
Pure/local logic        Real boundary behavior
         │                     │
         └──────────┬──────────┘
                    ▼
             Combined Evidence
```

---

# 311. Relación con Conformance

Posteriormente:

```text
Shared Database Contract
          │
          ▼
   Conformance Suite
          │
    ┌─────┼─────┬────────┐
    ▼     ▼     ▼        ▼
 MySQL MariaDB PostgreSQL SQLite
```

permitirá determinar sistemáticamente qué implementaciones satisfacen los contratos comunes.

---

# 312. Relación con Test Environment System

Este documento establece **qué significa y qué exige una integración real**.

El siguiente documento definirá cómo crear y administrar los entornos donde esas pruebas se ejecutan.

---

# 313. Arquitectura consolidada

```text
                DATABASE INTEGRATION TESTING
                           │
                           ▼
                    Test Definition
                           │
                           ▼
                 Environment Resolver
                           │
                           ▼
                 Safety / Test Guard
                           │
                           ▼
                    Provisioning
                           │
                           ▼
                 Real Infrastructure
                           │
       ┌───────────────────┼────────────────────┐
       ▼                   ▼                    ▼
     DBMS              External Tool         Storage
       │
       ▼
                 VoltStack Database
       │
       ├── Driver
       ├── Connection
       ├── Query Engine
       ├── Schema
       ├── ORM
       ├── Transaction
       ├── Cache Integration
       ├── Events
       ├── Telemetry
       └── Operations
                           │
                           ▼
                   Observable State
                           │
                           ▼
                      Assertions
                           │
                           ▼
                 Integration Evidence
                           │
                           ▼
                        Cleanup
```

---

# 314. Regla final

> **VoltStack Database no considerará demostrada una propiedad dependiente de infraestructura externa hasta haberla observado sobre una frontera real y controlada. Los mocks demostrarán la lógica de VoltStack frente a respuestas simuladas; las pruebas de integración demostrarán cómo VoltStack interactúa realmente con drivers, conexiones, motores, procesos y recursos externos. Ambas evidencias son necesarias, pero ninguna deberá hacerse pasar por la otra.**

En forma compacta:

```text
Unit Test
=
Does our logic make the right decision?
```

```text
Integration Test
=
Does that decision actually work
across the real boundary?
```

---

# 315. Siguiente documento

```text
286_DATABASE_DATABASE_TEST_ENVIRONMENT_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Database Test Environment System
│
├── Environment Model
├── Environment Profiles
├── Local Environments
├── Container Environments
├── Remote Test Environments
├── DBMS Provisioning
├── Version Matrix
├── Driver Matrix
├── Test Database Provisioning
├── Schema Provisioning
├── Test Credentials
├── Role Profiles
├── Isolation Strategies
├── Parallel Workers
├── Resource Namespacing
├── Environment Guards
├── Production Protection
├── Environment Fingerprints
├── Capability Discovery
├── Readiness Probes
├── Reset Strategies
├── Cleanup
├── Garbage Collection
├── Failure Recovery
├── Resource Budgets
├── CI Integration
├── Local Developer Integration
└── Persistent Runtime Test Environments
```

manteniendo como principio:

> **El entorno de pruebas deberá ser reproducible, aislado y suficientemente real para demostrar la propiedad evaluada, sin poner en riesgo infraestructura o datos ajenos a la ejecución de pruebas.**