# 283_DATABASE_TESTING_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Testing Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 283 — Database Testing Architecture  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `282_DATABASE_ADMINISTRATION_SYSTEM.md`  
**Siguiente documento:** `284_DATABASE_UNIT_TESTING_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura general del sistema de pruebas de:

```text
VoltStack/Quantum/Database
```

El objetivo es establecer cómo VoltStack demostrará de forma:

- reproducible;
- determinista;
- aislada;
- portable;
- observable;
- automatizable;
- eficiente;

que el subsistema Database conserva sus garantías en:

```text
Query Engine
ORM
Schema
Migrations
Transactions
Connections
Drivers
Hydration
Relationships
Caching
Concurrency
Distribution
Backup
Restore
Maintenance
Security
Resilience
Persistent Runtimes
```

La regla central será:

> **Una prueba deberá ejecutarse en el nivel más bajo capaz de demostrar la propiedad evaluada, pero ninguna abstracción simulada podrá utilizarse como evidencia de una garantía que dependa del comportamiento real del motor de base de datos.**

Por tanto:

```text
Fast isolated tests
        +
Real database integration tests
        +
Cross-platform conformance tests
        +
Failure tests
        +
Performance tests
        =
Database confidence
```

Nunca:

```text
Mocks pass
   ↓
Therefore PostgreSQL/MySQL/MariaDB/SQLite work
```

---

# 2. Problema arquitectónico

VoltStack Database contiene múltiples capas:

```text
Application API
      ↓
ORM
      ↓
Query Engine
      ↓
Compiler
      ↓
Execution Engine
      ↓
Connection
      ↓
Driver
      ↓
Database Server
```

Cada capa requiere tipos de pruebas diferentes.

Una prueba del Query AST no necesita necesariamente una base real.

Una prueba de:

```text
transaction isolation
deadlock detection
locking
savepoints
RETURNING
JSON semantics
full-text search
spatial queries
connection reset
```

sí puede depender del motor real.

Por ello:

```text
Testing Strategy
≠
Single Testing Technique
```

---

# 3. Objetivos

La arquitectura deberá permitir comprobar:

```text
Correctness
Isolation
Consistency
Portability
Compatibility
Resilience
Security
Performance
Resource Safety
Runtime Safety
Backward Compatibility
```

---

# 4. Testing ≠ Production Logic

El framework no deberá introducir comportamiento productivo diferente únicamente para hacer pasar pruebas.

Incorrecto:

```php
if ($environment === 'testing') {
    return true;
}
```

Correcto:

```text
Production Contract
      ↓
Test through contract
```

---

# 5. Testing ≠ Mocking

Regla:

```text
Testing
⊃
Mocking
```

Los mocks son sólo una herramienta.

---

# 6. Testing ≠ Integration Testing

La arquitectura contendrá varios niveles:

```text
Unit
Component
Integration
Conformance
System
Failure
Performance
```

---

# 7. Testing ≠ Benchmarking

Una prueba funcional responde:

```text
Is this correct?
```

Un benchmark responde:

```text
How does this behave under a defined workload?
```

---

# 8. Testing ≠ Diagnostics

Testing intenta demostrar propiedades bajo escenarios controlados.

Diagnostics investiga el estado de una ejecución/sistema.

---

# 9. Testing ≠ Health Check

Health determina si un sistema operativo parece saludable.

Testing verifica comportamiento esperado.

---

# 10. Principio de evidencia

Cada prueba producirá evidencia con un alcance determinado.

Ejemplo:

```text
Query compiler unit test
```

puede demostrar:

```text
AST X compiles into expected SQL for PostgreSQL dialect
```

pero no demuestra:

```text
PostgreSQL accepts and executes that SQL
```

---

# 11. Evidence Scope

Conceptualmente:

```php
enum TestEvidenceScope
{
    case UNIT;
    case COMPONENT;
    case INTEGRATION;
    case PLATFORM;
    case CONFORMANCE;
    case SYSTEM;
    case PERFORMANCE;
}
```

---

# 12. Evidence strength

La evidencia dependerá de la propiedad evaluada.

No existirá una jerarquía universal:

```text
integration > unit
```

Una prueba unitaria puede ser evidencia suficiente de una función pura.

---

# 13. Real-behavior rule

Cuando una propiedad dependa de:

```text
database protocol
transaction isolation
locking
SQL semantics
server configuration
driver behavior
platform capability
```

deberá existir una prueba contra infraestructura real compatible.

---

# 14. Plataformas objetivo

La matriz principal será:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 15. MySQL ≠ MariaDB

Se mantiene el invariante:

> **Una prueba exitosa en MySQL no constituye evidencia de conformidad de MariaDB y viceversa.**

---

# 16. SQLite ≠ servidor SQL

SQLite tendrá su propia matriz.

No se utilizará como sustituto universal de:

```text
MySQL
MariaDB
PostgreSQL
```

---

# 17. Error clásico

Queda explícitamente prohibida la inferencia:

```text
Tests pass on SQLite
        ↓
Application works on PostgreSQL
```

---

# 18. Arquitectura general

```text
                    Test Suite
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
    Unit Tests     Integration Tests   Conformance
       │                │                │
       ▼                ▼                ▼
 Pure Components   Real Database      Platform Matrix
       │                │                │
       └────────────────┼────────────────┘
                        ▼
                 Test Evidence
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      Correctness   Compatibility   Regression
```

---

# 19. Testing layers

Propuesta:

```text
Layer 0 — Static validation
Layer 1 — Unit
Layer 2 — Component
Layer 3 — Integration
Layer 4 — Platform Conformance
Layer 5 — System
Layer 6 — Failure/Chaos
Layer 7 — Performance
```

---

# 20. Layer 0 — Static validation

Podrá incluir:

```text
type analysis
architecture dependency rules
API compatibility
code style
static invariants
```

---

# 21. Static validation limitations

No demostrará comportamiento del DBMS.

---

# 22. Layer 1 — Unit

Evaluará componentes aislables:

```text
AST nodes
normalizers
validators
planners
metadata
type conversion
query rewrites
capability evaluation
```

---

# 23. Unit test principle

```text
No database unless database behavior is the subject.
```

---

# 24. Layer 2 — Component

Probará varias unidades colaborando.

Ejemplo:

```text
Query Builder
     ↓
AST
     ↓
Normalizer
     ↓
Compiler
```

sin necesariamente ejecutar SQL.

---

# 25. Layer 3 — Integration

Probará componentes contra:

```text
real driver
real connection
real database
```

cuando corresponda.

---

# 26. Layer 4 — Platform Conformance

Ejecutará contratos comunes sobre cada plataforma soportada.

```text
Database Contract Suite
        │
 ┌──────┼───────┬───────┐
 ▼      ▼       ▼       ▼
MySQL MariaDB PostgreSQL SQLite
```

---

# 27. Layer 5 — System

Probará flujos completos.

Ejemplo:

```text
Entity
 ↓
ORM
 ↓
UoW
 ↓
Query Engine
 ↓
Compiler
 ↓
Executor
 ↓
Database
 ↓
Hydration
 ↓
Entity
```

---

# 28. Layer 6 — Failure

Evaluará:

```text
connection loss
timeouts
deadlocks
lock contention
server restart
partial failure
unknown transaction outcome
resource exhaustion
```

---

# 29. Layer 7 — Performance

Evaluará:

```text
latency
throughput
memory
CPU
allocations
connection usage
query count
hydration cost
```

bajo workloads definidos.

---

# 30. Testing Pyramid

Conceptualmente:

```text
                  /\
                 /  \
                /System\
               /--------\
              /Integration\
             /-------------\
            /  Component    \
           /-----------------\
          /       Unit        \
         /_____________________\
```

Pero VoltStack no asumirá que cantidad implica importancia.

---

# 31. Testing Matrix

La arquitectura real será más parecida a una matriz:

```text
                     UNIT  INT  CONF  FAIL  PERF

Query AST              ✓
Compiler               ✓    ✓    ✓
Connection                  ✓    ✓     ✓
Transaction                 ✓    ✓     ✓
ORM                     ✓    ✓    ✓
Hydration               ✓    ✓    ✓
Schema                  ✓    ✓    ✓
Migration               ✓    ✓    ✓     ✓
Driver                       ✓    ✓     ✓
Backup                       ✓    ✓     ✓
Runtime                       ✓          ✓
```

---

# 32. TestCase abstraction

VoltStack podrá proporcionar:

```php
abstract class DatabaseTestCase
{
}
```

pero deberá mantenerse modular.

---

# 33. Specialized test cases

Podrán existir:

```text
DatabaseUnitTestCase
DatabaseIntegrationTestCase
DatabaseConformanceTestCase
DatabasePerformanceTestCase
```

---

# 34. Framework independence

El core de testing no deberá depender obligatoriamente de PHPUnit como concepto arquitectónico.

---

# 35. Testing adapter

Podrán existir adaptadores para:

```text
PHPUnit
Pest
otros runners
```

---

# 36. Assertion core

VoltStack podrá proporcionar assertions reutilizables independientemente del runner.

---

# 37. TestEnvironment

Objeto central:

```php
final readonly class DatabaseTestEnvironment
{
    public function __construct(
        public TestEnvironmentId $id,
        public DatabasePlatformId $platform,
        public DatabaseVersion $version,
        public TestDatabaseConfiguration $database,
        public TestIsolationPolicy $isolation,
        public TestSeed $seed,
    ) {}
}
```

---

# 38. Environment ≠ connection

Un entorno de pruebas puede contener múltiples conexiones.

---

# 39. Environment identity

Deberá identificar al menos:

```text
platform
version
driver
configuration profile
extensions
capabilities
```

---

# 40. Capability snapshot

Cada entorno real podrá producir:

```text
CapabilitySnapshot
```

---

# 41. Tests by capability

Las pruebas deberán poder declarar:

```php
#[RequiresCapability('query.returning')]
```

---

# 42. Capability-based testing

Preferible a:

```php
if ($db === 'postgres') {
}
```

---

# 43. Platform-specific tests

Serán válidos cuando la funcionalidad sea realmente específica.

Ejemplo:

```text
PostGIS
```

---

# 44. Skip semantics

Un test no deberá saltarse silenciosamente.

---

# 45. Skip reason

Debe registrar:

```text
unsupported capability
missing extension
environment unavailable
version limitation
explicit exclusion
```

---

# 46. Skip ≠ Pass

Regla crítica:

```text
SKIPPED
≠
PASSED
```

---

# 47. Unsupported ≠ Failed

Si una característica es oficialmente no soportada:

```text
UNSUPPORTED
```

no necesariamente constituye fallo de conformidad.

---

# 48. Required capability

Pero si el perfil de compatibilidad exige esa capacidad:

```text
missing capability
=
conformance failure
```

---

# 49. TestProfile

```php
enum DatabaseTestProfile
{
    case FAST;
    case STANDARD;
    case FULL;
    case CONFORMANCE;
    case FAILURE;
    case PERFORMANCE;
}
```

---

# 50. FAST

Podrá ejecutar:

```text
unit
component
selected SQLite/in-memory-safe tests
```

---

# 51. STANDARD

Podrá incluir una plataforma real configurada.

---

# 52. FULL

Podrá ejecutar:

```text
all supported platforms
multiple versions
integration
system
failure
```

---

# 53. CONFORMANCE

Ejecutará principalmente contratos de compatibilidad.

---

# 54. FAILURE

Activará escenarios destructivos/controlados.

---

# 55. PERFORMANCE

Ejecutará benchmarks.

---

# 56. Local developer workflow

Ejemplo:

```text
volt test database --profile=fast
```

---

# 57. Integration workflow

```text
volt test database --profile=standard
```

---

# 58. Full matrix

```text
volt test database --profile=full
```

---

# 59. Platform selection

```text
volt test database --platform=postgresql
```

---

# 60. Multiple platforms

```text
volt test database \
  --platform=mysql \
  --platform=mariadb \
  --platform=postgresql \
  --platform=sqlite
```

---

# 61. Test orchestration

```text
Test Runner
    ↓
Environment Resolver
    ↓
Environment Provisioner
    ↓
Capability Discovery
    ↓
Test Planner
    ↓
Test Execution
    ↓
Isolation Reset
    ↓
Evidence Collector
    ↓
Report
```

---

# 62. Test Planner

Podrá decidir qué tests aplican según:

```text
profile
platform
capabilities
version
extensions
runtime
available resources
```

---

# 63. Test discovery

Tests deberán descubrirse de forma determinista.

---

# 64. Test metadata

Ejemplo conceptual:

```php
#[DatabaseTest]
#[RequiresDatabase]
#[RequiresCapability('transaction.savepoint')]
#[Isolation(TestIsolation::TRANSACTION)]
final class SavepointTest
{
}
```

---

# 65. Test isolation

Será una preocupación de primer nivel.

---

# 66. Isolation strategies

```php
enum DatabaseTestIsolation
{
    case NONE;
    case TRANSACTION;
    case TRUNCATE;
    case RESET_SCHEMA;
    case DATABASE_PER_TEST;
    case SNAPSHOT_RESTORE;
    case CUSTOM;
}
```

---

# 67. Transaction isolation strategy

Patrón:

```text
BEGIN
 ↓
test
 ↓
ROLLBACK
```

---

# 68. Limitation

No sirve para todos los tests.

Ejemplo:

```text
transaction commit behavior
connection loss
DDL auto-commit behavior
multi-connection visibility
```

---

# 69. Transactional test ≠ transaction test

Distinción crítica:

```text
Transactional Test
=
test wrapped in transaction
```

mientras:

```text
Transaction Test
=
test whose subject is transaction semantics
```

---

# 70. Truncate isolation

Podrá limpiar tablas entre pruebas.

---

# 71. Foreign keys

La estrategia deberá respetar:

```text
FK dependencies
platform behavior
triggers
identity sequences
```

---

# 72. Reset schema

Podrá reconstruir:

```text
schema
indexes
constraints
sequences
```

---

# 73. Database-per-test

Mayor aislamiento, mayor costo.

---

# 74. Snapshot restore

Puede ser útil para entornos pesados.

---

# 75. Isolation selection

No habrá una estrategia universal.

---

# 76. Isolation guarantee

Cada estrategia declarará qué garantiza.

---

# 77. Test contamination

El framework deberá detectar cuando sea posible:

```text
unexpected tables
remaining rows
open transactions
open connections
modified session state
```

---

# 78. Cleanup

Cada prueba deberá terminar en estado controlado.

---

# 79. Cleanup failure

No deberá ocultarse detrás del resultado del test.

---

# 80. Test result + cleanup result

Conceptualmente:

```text
TestExecutionResult
+
TestCleanupResult
=
FinalTestResult
```

---

# 81. Cleanup failure severity

Un test funcionalmente exitoso con cleanup fallido podrá considerarse:

```text
FAILED
```

porque compromete la siguiente prueba.

---

# 82. Persistent runtime testing

VoltStack deberá probar específicamente:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 83. Worker reuse test

Patrón:

```text
Request A
   ↓
Database State A
   ↓
reset
   ↓
Request B
   ↓
assert no A state
```

---

# 84. State leakage

Deberá comprobar:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
TenantContext
ShardContext
QueryContext
Connection state
```

---

# 85. Coroutine tests

OpenSwoole requerirá escenarios concurrentes.

```text
Coroutine A → Tenant A
Coroutine B → Tenant B
```

Nunca:

```text
A observes B state
```

---

# 86. Connection reuse tests

Deberán verificar:

```text
session variables reset
transaction closed
temporary state reset
tenant state reset
isolation reset
```

---

# 87. Connection poisoning

Un connection outcome incierto deberá poder probar:

```text
connection discarded
```

---

# 88. Test fixtures

Integración con:

```text
197_DATABASE_FIXTURE_SYSTEM.md
```

---

# 89. Test factories

Integración con:

```text
193_DATABASE_FACTORY_SYSTEM.md
194_DATABASE_MODEL_FACTORY_SYSTEM.md
195_DATABASE_ENTITY_FACTORY_SYSTEM.md
198_DATABASE_TEST_DATA_GENERATION_SYSTEM.md
```

---

# 90. Fixture ≠ isolation

Una fixture define datos.

No garantiza limpieza.

---

# 91. Deterministic test data

Tests deberán poder recibir:

```text
seed
clock
locale
```

deterministas.

---

# 92. Randomized tests

Toda aleatoriedad deberá poder reproducirse.

---

# 93. Failure report

Deberá incluir:

```text
seed
```

cuando sea relevante.

---

# 94. Time

Los tests sensibles al tiempo deberán utilizar un clock controlable cuando la semántica lo permita.

---

# 95. Database server time

Cuando se pruebe:

```text
CURRENT_TIMESTAMP
database-generated timestamps
```

se estará evaluando tiempo del servidor real.

---

# 96. Application clock ≠ DB clock

No deberán confundirse.

---

# 97. ORM tests

Deberán cubrir:

```text
identity
state
change tracking
persistence
hydration
relationships
loading
locking
```

---

# 98. ORM identity invariant

Prueba esencial:

```php
$a = $repository->find(10);
$b = $repository->find(10);

assert($a === $b);
```

dentro del mismo IdentityMap scope.

---

# 99. Identity scope reset

Después de:

```text
clear/reset/new operation
```

no se exige misma instancia PHP.

---

# 100. UnitOfWork tests

Deberán verificar:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

y sus transiciones.

---

# 101. flush tests

Deberán demostrar:

```text
flush ≠ commit
```

---

# 102. persist tests

Deberán demostrar:

```text
persist ≠ immediate INSERT
```

---

# 103. remove tests

Deberán demostrar:

```text
remove ≠ immediate DELETE
```

---

# 104. Hydration tests

Cubrirán:

```text
entity
scalar
tuple
DTO
projection
partial data
joins
deduplication
null semantics
```

---

# 105. Dirty entity hydration

Una consulta no deberá sobrescribir silenciosamente cambios pendientes de una entidad managed.

---

# 106. Partial entity

Deberá comprobarse:

```text
partial entity
≠
complete entity
```

---

# 107. Relationship tests

Cubrirán:

```text
1:1
1:N
N:1
N:N
polymorphic
eager
lazy
batch
```

---

# 108. N+1 tests

Podrán utilizar:

```text
semantic query observation
```

en vez de depender sólo de strings SQL.

---

# 109. Query Engine tests

Podrán separarse:

```text
Builder
AST
Normalization
Validation
Semantic Analysis
Optimization
Planning
Compilation
Execution
```

---

# 110. Query AST tests

Deberán ser altamente unitarios.

---

# 111. AST snapshots

Podrán utilizarse representaciones canónicas.

---

# 112. Snapshot testing caution

Los snapshots no deberán sustituir assertions semánticas importantes.

---

# 113. Compiler tests

Patrón:

```text
AST
 ↓
Compiler
 ↓
SQL + bindings
```

---

# 114. Compiler golden tests

Podrán existir golden files para SQL esperado.

---

# 115. Golden test limitations

Un SQL esperado puede estar sintácticamente correcto pero no ser aceptado por el motor.

Por eso deberá complementarse con conformance cuando corresponda.

---

# 116. SQL formatting

Los tests no deberán romperse innecesariamente por whitespace si el formato no es parte del contrato.

---

# 117. Binding tests

Deberán comprobar:

```text
value
logical type
driver type
placeholder mapping
```

---

# 118. SQL injection tests

Casos maliciosos deberán formar parte del sistema.

---

# 119. Identifier injection

También deberán probarse:

```text
table
column
order by
JSON path
raw expression boundaries
```

---

# 120. Query optimizer tests

Cada rewrite deberá demostrar:

```text
semantic equivalence
```

bajo las condiciones declaradas.

---

# 121. Optimizer negative tests

También:

```text
rule must NOT fire
```

cuando las precondiciones no se cumplan.

---

# 122. Planner tests

Deberán verificar selección de:

```text
logical plans
physical plans
capability-dependent alternatives
```

---

# 123. Capability tests

El sistema de documento 275 requerirá pruebas de:

```text
evidence
constraints
dependencies
confidence
conflicts
UNKNOWN
emulation
```

---

# 124. Version tests

Deberán evitar comparaciones lexicográficas incorrectas.

---

# 125. Evidence conflict

Ejemplo:

```text
version inference = SUPPORTED
runtime probe = NOT AVAILABLE
```

deberá producir el resultado definido por Capability System.

---

# 126. Schema tests

Cubrirán:

```text
definitions
model
AST
introspection
diff
compiler
platform compatibility
```

---

# 127. Schema round-trip

Cuando sea viable:

```text
Schema Definition
      ↓
Compile/Create
      ↓
Database
      ↓
Introspection
      ↓
Normalized Schema Model
```

comparado contra expectativa.

---

# 128. Round-trip caveat

Algunos motores normalizan o pierden detalles.

La comparación deberá ser semántica.

---

# 129. Migration tests

Cubrirán:

```text
discovery
planning
execution
history
rollback
batch
safety
zero downtime
```

---

# 130. Migration round-trip

Cuando sea semánticamente válido:

```text
A
↓ migrate
B
↓ rollback
A'
```

verificar:

```text
SemanticallyEquivalent(A, A')
```

No necesariamente representación textual idéntica.

---

# 131. Destructive migrations

Deberán probar:

```text
safety detection
approval/policy
```

sin destruir recursos fuera del sandbox.

---

# 132. Transaction tests

Requerirán bases reales.

---

# 133. Isolation tests

Podrán necesitar:

```text
Connection A
Connection B
```

coordinadas.

---

# 134. Concurrency harness

Se diseñará un:

```text
DatabaseConcurrencyTestHarness
```

---

# 135. Harness

Permitirá secuencias como:

```text
A: BEGIN
A: UPDATE row
B: BEGIN
B: SELECT/UPDATE row
A: COMMIT
B: observe
```

---

# 136. Synchronization points

Deberán ser deterministas cuando sea posible.

---

# 137. Sleep-based tests

Se evitarán patrones como:

```php
sleep(2);
```

como sincronización primaria.

---

# 138. Barriers

Se preferirán:

```text
barriers
latches
signals
server-state observations
```

---

# 139. Deadlock tests

Podrán crear intencionalmente:

```text
TX-A locks X
TX-B locks Y
TX-A waits Y
TX-B waits X
```

en entorno controlado.

---

# 140. Deadlock assertion

No se asumirá que todos los motores eligen la misma víctima.

---

# 141. Semantic assertion

Se comprobará:

```text
deadlock detected
one transaction aborted
survivor semantics valid
retry classification correct
```

---

# 142. Locking tests

Cubrirán:

```text
optimistic
pessimistic
NOWAIT
SKIP LOCKED
```

sólo cuando las capacidades existan.

---

# 143. Unknown transaction outcome

Debe probarse.

Ejemplo:

```text
COMMIT sent
connection lost
acknowledgment unavailable
```

---

# 144. Fault injection

Será necesaria para reproducir escenarios difíciles.

---

# 145. FaultInjector

Contrato conceptual:

```php
interface DatabaseFaultInjector
{
    public function inject(
        FaultPoint $point,
        FaultBehavior $behavior
    ): void;
}
```

---

# 146. Fault points

Ejemplos:

```text
before_connect
after_connect
before_execute
after_server_execute
before_commit
after_commit_send
before_commit_ack
during_result_read
during_reset
```

---

# 147. Production separation

Fault injection deberá estar:

```text
test-only
explicit
unavailable by default in production
```

---

# 148. Driver conformance

Cada driver deberá cumplir un contrato común.

---

# 149. Driver suite

Conceptualmente:

```php
abstract class DriverConformanceSuite
{
    abstract protected function createDriver(): Driver;
}
```

---

# 150. Driver contract tests

Podrán cubrir:

```text
connect
disconnect
prepare
bind
execute
fetch
transaction
savepoint
error mapping
reset
cancellation
```

---

# 151. Platform conformance

Separado de Driver Conformance.

---

# 152. Driver ≠ Platform

Un driver puede conectarse a una plataforma.

La semántica de plataforma no pertenece enteramente al driver.

---

# 153. Dialect tests

Deberán comprobar:

```text
quoting
placeholders
DDL
DML
functions
returning
locking
JSON
full text
spatial
```

según capacidades.

---

# 154. Cross-version matrix

Idealmente:

```text
Platform × Version × Driver
```

---

# 155. Version support policy

No todas las versiones históricas deberán ejecutarse en cada commit.

---

# 156. Tiered matrix

Podrá existir:

```text
Tier 1 — every commit
Tier 2 — nightly
Tier 3 — release
```

---

# 157. Tier 1

Versiones principales prioritarias.

---

# 158. Tier 2

Matriz ampliada.

---

# 159. Tier 3

Matriz completa soportada antes de release.

---

# 160. Extension matrix

Características como:

```text
PostGIS
FTS extension
SQLite extensions
```

tendrán perfiles propios.

---

# 161. Backup tests

Cubrirán:

```text
plan
capture
manifest
checksum
verification
failure
cancellation
```

---

# 162. Restore tests

Deberán incluir restore real cuando sea viable.

---

# 163. Backup created ≠ restorable

Regla de testing:

> **Una prueba que sólo demuestra que el comando de backup terminó no demuestra que el artefacto puede restaurarse.**

---

# 164. Restore verification

La evidencia fuerte será:

```text
Backup
 ↓
Restore into isolated target
 ↓
Validate schema/data
```

---

# 165. Maintenance tests

Deberán verificar:

```text
capability
planning
execution
resource policy
result classification
```

---

# 166. Health tests

Podrán utilizar entornos:

```text
healthy
degraded
unavailable
misconfigured
```

---

# 167. Diagnostics tests

Deberán comprobar explicaciones con evidencia conocida.

---

# 168. Administration tests

Del documento 282:

```text
authorization
capability
planning
safety
approval
execution
verification
audit
UNKNOWN
```

---

# 169. Security testing

Será transversal.

---

# 170. Security suites

Cubrirán:

```text
SQL injection
identifier injection
credential leakage
cross-tenant leakage
authorization bypass
raw expression misuse
audit redaction
telemetry redaction
```

---

# 171. Secret scanning

Los resultados de tests podrán inspeccionarse para comprobar que secretos de prueba no aparecen en:

```text
logs
exceptions
telemetry
debug reports
```

---

# 172. Multitenancy tests

Cuando el paquete esté instalado:

```text
Tenant A
Tenant B
```

deberán permanecer aislados.

---

# 173. Cross-tenant invariant

```text
A cannot observe B
```

salvo operación explícita autorizada.

---

# 174. Tenant connection tests

Cubrirán:

```text
database-per-tenant
schema-per-tenant
shared schema
```

según estrategias soportadas.

---

# 175. Tenant migration tests

Podrán simular:

```text
100 tenants
some success
some fail
some unknown
```

---

# 176. Sharding tests

Deberán verificar:

```text
routing
single-shard writes
fan-out reads
merge
topology changes
partial failure
```

---

# 177. Under-routing

Será un error crítico a detectar.

---

# 178. Over-routing

También deberá medirse, aunque puede ser principalmente problema de performance.

---

# 179. Shard topology tests

Planes deberán invalidarse cuando:

```text
TopologyGeneration changes
```

si la semántica lo requiere.

---

# 180. Replica tests

Cubrirán:

```text
read routing
lag
sticky reads
health
eligibility
failover
```

---

# 181. Replica health ≠ eligibility

Los tests deberán preservar esta separación.

---

# 182. Sticky read tests

Después de escritura:

```text
write
 ↓
read
```

deberá respetar la política de consistencia configurada.

---

# 183. Cache tests

Deberán separar:

```text
Query Cache
Compiled Query Cache
Result Cache
Metadata Cache
Entity Cache
Hydration Cache
```

---

# 184. Cache hit ≠ usable hit

Se probará:

```text
physical hit
+
consistency policy
=
usable/rejected
```

---

# 185. Cache invalidation

Deberá probarse alrededor de:

```text
commit
rollback
unknown commit
```

---

# 186. Uncommitted data

Nunca deberá publicarse en cache compartido.

---

# 187. Event tests

Cubrirán:

```text
ordering
payload
failure policy
transaction phase
afterCommit semantics
```

---

# 188. Event listener failure

Un listener posterior al commit no podrá transformar mágicamente un commit exitoso en rollback.

---

# 189. Telemetry tests

Deberán verificar:

```text
span relationships
metrics
redaction
bounded cardinality
context reset
```

---

# 190. Telemetry ≠ semantics

Desactivar telemetry no deberá cambiar el resultado de la consulta.

---

# 191. Memory tests

Importantes para persistent runtimes.

---

# 192. Memory growth

Se podrán ejecutar:

```text
N operations
```

y observar:

```text
IdentityMap growth
metadata cache growth
query cache growth
hydration allocations
```

---

# 193. Memory leak ≠ cache

Un crecimiento esperado de cache deberá distinguirse de una fuga.

---

# 194. Bounded cache tests

Caches limitados deberán respetar sus budgets.

---

# 195. Resource governance tests

Podrán provocar:

```text
connection limit
memory budget
query timeout
result limit
concurrency limit
```

---

# 196. Admission control

Deberá comprobarse que la carga rechazada no comienza parcialmente una operación cuando no corresponde.

---

# 197. Timeout tests

No dependerán únicamente del tiempo de pared.

---

# 198. Time tolerance

Cuando exista temporización real deberá definirse tolerancia explícita.

---

# 199. Performance regression

Los benchmarks podrán mantener baselines.

---

# 200. Baseline ≠ absolute truth

El baseline dependerá de:

```text
hardware
OS
PHP version
database version
configuration
runtime
```

---

# 201. Benchmark metadata

Todo resultado deberá registrar ese contexto.

---

# 202. Query count assertions

VoltStack podrá soportar:

```php
$this->assertDatabaseQueryCount(2);
```

pero con cuidado.

---

# 203. Query count ≠ performance

Menos queries no significa necesariamente mayor rendimiento.

---

# 204. Semantic assertions

Se favorecerán assertions como:

```text
no N+1
uses bounded batches
does not load relation lazily
```

cuando sea posible.

---

# 205. Database assertions

Ejemplos:

```php
assertDatabaseHas(...)
assertDatabaseMissing(...)
assertDatabaseCount(...)
assertEntityExists(...)
assertSoftDeleted(...)
```

---

# 206. Assertion implementation

No deberá saltarse las abstracciones de seguridad de forma accidental.

---

# 207. Raw test inspection

Podrá existir para testing interno, pero será explícita.

---

# 208. Test transaction visibility

Assertions deberán usar conexión/contexto apropiados.

---

# 209. Common trap

Si el test tiene una transacción no confirmada:

```text
Connection A sees row
Connection B may not
```

según aislamiento.

---

# 210. Assertion context

Por ello deberá ser explícito.

---

# 211. Parallel test execution

VoltStack deberá soportarlo.

---

# 212. Parallel isolation

Opciones:

```text
database per worker
schema per worker
namespace/prefix per worker
isolated SQLite file
```

---

# 213. Shared database danger

Dos workers no deberán limpiar mutuamente sus datos.

---

# 214. Worker identity

Podrá formar parte del namespace de pruebas.

---

# 215. Parallel transactions

Deberán distinguirse de tests de concurrencia intencional.

---

# 216. Test resource naming

Se utilizarán nombres deterministas y collision-safe.

---

# 217. Cleanup after crash

El sistema deberá poder descubrir recursos huérfanos.

---

# 218. Resource ownership metadata

Podrá incluir:

```text
test run id
worker id
created at
resource type
```

---

# 219. Safe cleanup

Nunca deberá eliminar una base que no pueda demostrar que pertenece al entorno de testing.

---

# 220. Critical safety rule

> **El sistema de testing jamás deberá ejecutar operaciones destructivas sobre una base de datos que no haya sido identificada de forma verificable como recurso de pruebas.**

---

# 221. Production guard

Podrá requerirse:

```text
DatabaseTestEnvironmentMarker
```

---

# 222. Marker

Puede incluir:

```text
database naming convention
metadata table
generated test-run token
environment configuration
```

---

# 223. Multiple evidence

Para operaciones destructivas se favorecerá más de una señal.

---

# 224. Environment variable alone

No será evidencia suficiente en escenarios críticos.

---

# 225. Test database provisioning

Podrá realizarse mediante adapters.

---

# 226. Provisioning backends

Ejemplos conceptuales:

```text
existing local database
Docker/container
ephemeral process
CI service
remote isolated test server
```

---

# 227. Core independence

Database Testing Core no dependerá obligatoriamente de Docker.

---

# 228. Provisioner contract

```php
interface DatabaseTestEnvironmentProvisioner
{
    public function provision(
        TestEnvironmentDefinition $definition
    ): ProvisionedTestEnvironment;

    public function destroy(
        ProvisionedTestEnvironment $environment
    ): void;
}
```

---

# 229. Provision ≠ migrate

Crear servidor/base y crear schema de aplicación son pasos distintos.

---

# 230. Environment lifecycle

```text
DEFINED
   ↓
PROVISIONING
   ↓
READY
   ↓
PREPARING
   ↓
RUNNING
   ↓
RESETTING
   ↓
DESTROYING
   ↓
DESTROYED
```

Errores:

```text
FAILED
TAINTED
UNKNOWN
```

---

# 231. Tainted environment

Si el reset falla:

```text
TAINTED
```

y no deberá reutilizarse silenciosamente.

---

# 232. Unknown environment state

Si una operación destructiva pierde confirmación:

```text
UNKNOWN
```

requiere reconciliación.

---

# 233. Schema setup

Podrá realizarse mediante:

```text
migrations
schema compiler
snapshot
fixture
```

según perfil.

---

# 234. Test speed

El perfil rápido podrá utilizar snapshots o schema cache cuando preserven semántica.

---

# 235. Schema cache invalidation

Deberá depender de:

```text
migration generation
schema fingerprint
platform
capabilities
```

---

# 236. Test suite state

No se guardará en static mutable global state.

---

# 237. TestRunContext

```php
final readonly class DatabaseTestRunContext
{
    public function __construct(
        public TestRunId $runId,
        public DatabaseTestProfile $profile,
        public TestSeed $seed,
        public TestClock $clock,
        public array $environments,
    ) {}
}
```

---

# 238. TestCaseContext

Cada test tendrá contexto propio.

---

# 239. Test context leakage

Queda prohibido:

```php
DatabaseTest::$currentTenant;
DatabaseTest::$currentConnection;
DatabaseTest::$currentTransaction;
```

como estado global mutable.

---

# 240. Dependency injection

Los tests internos del framework deberán poder recibir componentes explícitamente.

---

# 241. Test doubles

Categorías:

```text
Stub
Fake
Mock
Spy
Fault Injector
Simulator
```

---

# 242. Stub

Devuelve respuestas predeterminadas.

---

# 243. Fake

Implementación funcional simplificada.

---

# 244. Mock

Verifica interacciones.

---

# 245. Spy

Registra interacciones.

---

# 246. Fault Injector

Introduce fallos en puntos controlados.

---

# 247. Simulator

Modela comportamiento externo.

---

# 248. Fake DB warning

Un Fake Database no deberá presentarse como emulador completo de PostgreSQL/MySQL/etc.

---

# 249. Test double contract

Todo double deberá documentar qué semántica NO reproduce.

---

# 250. Contract testing

Una interfaz podrá definir:

```text
Contract Test Suite
```

ejecutable sobre implementaciones reales y fakes.

---

# 251. Fake conformance

Un fake puede cumplir un subconjunto explícito del contrato.

---

# 252. Conformance level

Conceptualmente:

```php
enum TestDoubleConformance
{
    case STRUCTURAL;
    case BEHAVIORAL_SUBSET;
    case FULL_CONTRACT;
}
```

---

# 253. Full-contract caution

Para un DBMS complejo, declarar FULL_CONTRACT será excepcional.

---

# 254. Regression tests

Todo bug corregido debería poder convertirse en:

```text
reproducible failing test
        ↓
fix
        ↓
permanent regression test
```

---

# 255. Regression identity

Podrá relacionarse con:

```text
issue
bug id
release
platform
```

---

# 256. Regression portability

Un bug específico de MariaDB no deberá ocultarse detrás de una prueba sólo en MySQL.

---

# 257. Flaky tests

Deberán tratarse como defecto.

---

# 258. Retry flaky test

Reintentar automáticamente puede recopilar evidencia, pero no deberá convertir:

```text
fail → retry → pass
```

en un PASS completamente limpio sin reportar la inestabilidad.

---

# 259. Flakiness status

Podrá existir:

```text
FLAKY
```

en reportes avanzados.

---

# 260. Quarantine

Tests inestables podrán aislarse temporalmente.

Pero:

```text
quarantine
≠
delete evidence
```

---

# 261. TestResult

```php
enum DatabaseTestStatus
{
    case PASSED;
    case FAILED;
    case SKIPPED;
    case FLAKY;
    case ERROR;
    case CANCELLED;
}
```

---

# 262. Test evidence record

```php
final readonly class DatabaseTestEvidence
{
    public function __construct(
        public TestId $test,
        public DatabaseTestStatus $status,
        public TestEnvironmentFingerprint $environment,
        public array $capabilities,
        public Duration $duration,
        public ?TestFailure $failure,
    ) {}
}
```

---

# 263. Environment fingerprint

Podrá incluir:

```text
platform
version
driver
PHP version
VoltStack version
capability fingerprint
extensions
runtime
configuration profile
```

---

# 264. Reproducibility

Un fallo deberá proporcionar información suficiente para intentar reproducirlo.

---

# 265. Test report

Ejemplo:

```text
VoltStack Database Test Report

Profile: CONFORMANCE

PostgreSQL 18.x
  Passed: 1842
  Failed: 0
  Skipped: 12

MySQL 8.x
  Passed: 1781
  Failed: 0
  Skipped: 73

MariaDB
  Passed: ...
```

Los números son sólo ilustrativos.

---

# 266. Skip reporting

Deberá poder agrupar:

```text
unsupported capability
extension unavailable
version exclusion
environment unavailable
```

---

# 267. Conformance report

Podrá producir:

```text
Feature                PostgreSQL MySQL MariaDB SQLite

Transactions               PASS    PASS    PASS   PASS
Savepoints                  PASS    PASS    PASS   PASS
RETURNING                   PASS    ...     ...    ...
JSON Path                   PASS    ...     ...    ...
```

basado en pruebas reales.

---

# 268. Capability documentation

A futuro, resultados de conformance podrán ayudar a validar documentación de compatibilidad.

Pero:

```text
test result
≠
runtime capability discovery
```

---

# 269. CI architecture

```text
Commit
  ↓
Static
  ↓
Unit
  ↓
Component
  ↓
Tier-1 Integration
```

Posteriormente:

```text
Nightly
  ↓
Extended platform matrix
  ↓
Failure suites
  ↓
Performance samples
```

Y antes de release:

```text
Release Candidate
  ↓
Full Conformance
  ↓
Security
  ↓
Failure
  ↓
Performance
  ↓
Persistent Runtime Matrix
```

---

# 270. Fail fast

CI podrá detener ciertas fases tras fallos críticos.

---

# 271. Fail fast ≠ hide failures

En matrices paralelas deberán preservarse resultados ya obtenidos.

---

# 272. Test sharding

Suites grandes podrán dividirse entre workers.

---

# 273. Test order independence

Por defecto:

```text
Test A
```

no deberá requerir que:

```text
Test B
```

se haya ejecutado antes.

---

# 274. Ordered scenarios

Cuando exista una prueba de escenario multi-etapa, deberá encapsularse explícitamente como un escenario.

---

# 275. TestScenario

```php
interface DatabaseTestScenario
{
    public function stages(): iterable;
}
```

---

# 276. Scenario ≠ suite ordering

No se utilizará el orden global del runner para simular workflows.

---

# 277. Architecture tests

VoltStack deberá probar sus propias reglas de dependencia.

Ejemplo:

```text
ORM may depend on Query Engine
Query Engine must not depend on ORM
```

---

# 278. Forbidden dependency test

Podrán existir assertions sobre namespaces/imports.

---

# 279. Architecture invariant testing

Los invariantes documentados podrán convertirse progresivamente en reglas ejecutables.

---

# 280. Example

Invariante:

```text
Compiler never executes queries
```

podrá comprobarse mediante arquitectura/dependency tests.

---

# 281. Invariant catalog

A futuro podrá existir:

```text
DatabaseInvariantCatalog
```

relacionando:

```text
Invariant ID
    ↓
Tests proving it
```

---

# 282. Traceability

Ejemplo:

```text
DB-ADMIN-056
UNKNOWN outcome remains UNKNOWN
        ↓
AdministrationUnknownOutcomeTest
```

---

# 283. Specification coverage

No se medirá únicamente:

```text
line coverage
```

sino también:

```text
contract coverage
capability coverage
platform coverage
failure-mode coverage
invariant coverage
```

---

# 284. Code coverage

Será una señal auxiliar.

---

# 285. 100% line coverage

No demuestra:

```text
correct concurrency
correct transaction semantics
cross-platform portability
```

---

# 286. Mutation testing

Podrá incorporarse como herramienta opcional.

---

# 287. Property-based testing

También podrá integrarse.

Especialmente útil para:

```text
AST
normalization
type conversion
query generation
schema diff
```

---

# 288. Property example

```text
normalize(normalize(X))
=
normalize(X)
```

si la normalización debe ser idempotente.

---

# 289. Schema diff property

Ejemplo:

```text
diff(A, A)
=
empty
```

---

# 290. Query serialization property

Cuando exista representación canónica:

```text
decode(encode(AST))
≈
AST
```

---

# 291. Fuzz testing

Podrá utilizarse para:

```text
SQL parser boundaries
JSON paths
identifiers
AST combinations
type conversion
result decoding
```

---

# 292. Fuzz reproducibility

Todo fallo deberá conservar:

```text
seed
minimal failing input
```

cuando sea posible.

---

# 293. Security fuzzing

Especialmente importante en:

```text
raw expressions
identifiers
JSON paths
full-text syntax
import formats
```

---

# 294. Database version upgrades

La matriz podrá detectar cambios de comportamiento al introducir una nueva versión de DBMS.

---

# 295. New version process

Conceptualmente:

```text
Add version
   ↓
Discover capabilities
   ↓
Run conformance
   ↓
Compare
   ↓
Investigate differences
   ↓
Declare support
```

---

# 296. Version number ≠ support

Que una nueva versión inicie correctamente no significa que VoltStack la soporte oficialmente.

---

# 297. Support declaration

Requerirá evidencia definida por la política de release.

---

# 298. Backward compatibility tests

Integración futura con:

```text
322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md
```

---

# 299. Public API tests

Deberán verificar:

```text
signatures
contracts
behavior
deprecations
```

---

# 300. Serialization compatibility

Sólo para artefactos cuya persistencia/serialización sea contrato público.

---

# 301. Test architecture namespaces

Propuesta:

```text
src/Quantum/Database/Testing/
├── Contract/
│   ├── DatabaseTest.php
│   ├── TestEnvironmentProvisioner.php
│   ├── TestIsolationStrategy.php
│   ├── FaultInjector.php
│   └── ConformanceSuite.php
│
├── Context/
│   ├── DatabaseTestRunContext.php
│   ├── DatabaseTestCaseContext.php
│   └── TestEnvironmentContext.php
│
├── Environment/
│   ├── DatabaseTestEnvironment.php
│   ├── TestEnvironmentDefinition.php
│   ├── ProvisionedTestEnvironment.php
│   ├── TestEnvironmentFingerprint.php
│   └── TestEnvironmentMarker.php
│
├── Provisioning/
│   ├── TestEnvironmentProvisioner.php
│   ├── ExistingDatabaseProvisioner.php
│   └── EphemeralEnvironmentProvisioner.php
│
├── Isolation/
│   ├── DatabaseTestIsolation.php
│   ├── TransactionIsolationStrategy.php
│   ├── TruncateIsolationStrategy.php
│   ├── SchemaResetIsolationStrategy.php
│   ├── DatabasePerTestIsolationStrategy.php
│   └── SnapshotIsolationStrategy.php
│
├── Planning/
│   ├── DatabaseTestPlanner.php
│   ├── TestPlan.php
│   └── TestSelection.php
│
├── Execution/
│   ├── DatabaseTestExecutor.php
│   ├── TestExecutionResult.php
│   └── TestCleanupResult.php
│
├── Assertion/
│   ├── DatabaseAssertions.php
│   ├── QueryAssertions.php
│   ├── SchemaAssertions.php
│   └── ORMAssertions.php
│
├── Double/
│   ├── Fake/
│   ├── Mock/
│   ├── Stub/
│   └── Spy/
│
├── Fault/
│   ├── DatabaseFaultInjector.php
│   ├── FaultPoint.php
│   └── FaultBehavior.php
│
├── Concurrency/
│   ├── DatabaseConcurrencyTestHarness.php
│   ├── TestBarrier.php
│   └── TestSignal.php
│
├── Conformance/
│   ├── DriverConformanceSuite.php
│   ├── PlatformConformanceSuite.php
│   ├── CapabilityConformanceSuite.php
│   └── ConformanceResult.php
│
├── Evidence/
│   ├── DatabaseTestEvidence.php
│   ├── TestEvidenceScope.php
│   └── TestEvidenceCollector.php
│
├── Report/
│   ├── DatabaseTestReport.php
│   ├── ConformanceReport.php
│   └── TestReportRenderer.php
│
├── Runtime/
│   ├── FrankenPHP/
│   ├── RoadRunner/
│   └── OpenSwoole/
│
└── Exception/
    └── ...
```

---

# 302. Tests internos del componente

Los tests del propio framework podrán residir en:

```text
tests/Quantum/Database/
├── Unit/
├── Component/
├── Integration/
├── Conformance/
├── System/
├── Failure/
├── Security/
├── Performance/
└── Runtime/
```

---

# 303. Platform test directories

Dentro de Integration/Conformance:

```text
Platform/
├── MySQL/
├── MariaDB/
├── PostgreSQL/
└── SQLite/
```

---

# 304. Shared contract suites

Las pruebas comunes deberán reutilizar contratos en lugar de copiar test suites completas.

---

# 305. Example

```php
abstract class TransactionConformanceTest
{
    abstract protected function environment(): DatabaseTestEnvironment;

    public function testCommitPersistsChanges(): void
    {
        // shared contract
    }

    public function testRollbackDoesNotPersistChanges(): void
    {
        // shared contract
    }
}
```

---

# 306. Platform specialization

Una plataforma podrá agregar pruebas adicionales sin alterar el contrato común.

---

# 307. Error architecture

Excepciones de infraestructura de testing:

```text
DatabaseTestingException
├── TestEnvironmentException
├── TestProvisioningException
├── TestIsolationException
├── TestCleanupException
├── TestCapabilityException
├── TestConfigurationException
├── TestConformanceException
├── TestFaultInjectionException
├── TestTimeoutException
└── UnsafeTestEnvironmentException
```

---

# 308. Assertion failure ≠ infrastructure error

Deben distinguirse.

```text
Expected 5 rows, got 4
=
ASSERTION FAILURE
```

mientras:

```text
PostgreSQL test container could not start
=
INFRASTRUCTURE ERROR
```

---

# 309. Environment unavailable

No deberá convertirse automáticamente en:

```text
test passed
```

---

# 310. CI policy

Podrá decidir:

```text
required environment unavailable
=
pipeline failure
```

---

# 311. Developer local policy

Podrá permitir:

```text
optional environment unavailable
=
skip with explicit reason
```

---

# 312. Testing telemetry

El sistema de testing podrá emitir telemetry propia.

---

# 313. Metrics

Ejemplos:

```text
database_tests_total
database_tests_failed_total
database_tests_skipped_total
database_test_duration_seconds
database_test_environment_provision_seconds
database_test_cleanup_failures_total
```

---

# 314. Production telemetry separation

La telemetry de test no deberá mezclarse accidentalmente con producción.

---

# 315. Test database naming

Ejemplo conceptual:

```text
volt_test_<run>_<worker>
```

---

# 316. Naming alone not enough

El nombre será una señal, no la única protección.

---

# 317. Destructive operation guard

Antes de:

```text
DROP DATABASE
TRUNCATE
RESET SCHEMA
```

deberá verificarse la propiedad del recurso.

---

# 318. Remote testing

Una base remota podrá utilizarse sólo si está explícitamente designada para testing.

---

# 319. Production endpoint detection

Si un endpoint coincide con configuración productiva conocida:

```text
ABORT
```

por defecto.

---

# 320. Test credentials

Deberán tener privilegios limitados al entorno de pruebas.

---

# 321. Test database user

Idealmente no tendrá acceso a bases productivas.

---

# 322. Defense in depth

```text
Config validation
+
Resource marker
+
Naming
+
Credentials
+
Endpoint policy
+
Explicit test mode
```

---

# 323. Test runtime

El runner podrá utilizar:

```text
CLI process
worker
container
CI agent
```

---

# 324. No HTTP requirement

La arquitectura de testing de Database no dependerá del servidor HTTP.

---

# 325. Framework integration

VoltStack Testing podrá construir APIs superiores:

```php
class UserRepositoryTest extends DatabaseTestCase
{
    use RefreshDatabase;
}
```

sin que `RefreshDatabase` se convierta en el núcleo arquitectónico.

---

# 326. RefreshDatabase

Será una política de alto nivel construida sobre:

```text
Test Environment
+
Isolation Strategy
+
Schema Setup
+
Cleanup
```

---

# 327. Developer ergonomics

La experiencia deseada será simple:

```php
$user = User::factory()->create();

$this->assertDatabaseHas('users', [
    'id' => $user->id,
]);
```

mientras internamente se preservan:

```text
tenant
connection
transaction
platform
scope
```

---

# 328. Simplicidad externa

Regla:

> **La complejidad necesaria para garantizar portabilidad y aislamiento debe residir en la infraestructura de testing, no repetirse en cada prueba de aplicación.**

---

# 329. Invariantes arquitectónicos

## DB-TEST-001

Testing no será Production Logic.

## DB-TEST-002

Testing no será Mocking.

## DB-TEST-003

Mock success no demostrará DBMS compatibility.

## DB-TEST-004

Las garantías dependientes del motor tendrán pruebas reales.

## DB-TEST-005

MySQL no será MariaDB.

## DB-TEST-006

SQLite no será sustituto universal de otros DBMS.

## DB-TEST-007

PASS en SQLite no implicará PASS en PostgreSQL.

## DB-TEST-008

PASS en MySQL no implicará PASS en MariaDB.

## DB-TEST-009

Unit tests serán preferidos para lógica pura.

## DB-TEST-010

Integration tests usarán infraestructura real cuando la semántica lo requiera.

## DB-TEST-011

Conformance será distinto de Integration.

## DB-TEST-012

Performance testing será distinto de correctness testing.

## DB-TEST-013

Diagnostics no será Testing.

## DB-TEST-014

Health Check no será Testing.

## DB-TEST-015

Cada resultado tendrá un alcance de evidencia.

## DB-TEST-016

Evidence strength dependerá de la propiedad evaluada.

## DB-TEST-017

Platform behavior no se inferirá sólo de mocks.

## DB-TEST-018

Test environment identificará plataforma.

## DB-TEST-019

Test environment identificará versión.

## DB-TEST-020

Test environment identificará driver.

## DB-TEST-021

Test environment podrá registrar capabilities.

## DB-TEST-022

Tests podrán seleccionarse por capabilities.

## DB-TEST-023

Vendor conditionals no sustituirán capability requirements.

## DB-TEST-024

Platform-specific tests estarán permitidos cuando sean semánticamente específicos.

## DB-TEST-025

SKIPPED no será PASSED.

## DB-TEST-026

Skip tendrá razón explícita.

## DB-TEST-027

Missing required capability podrá fallar conformance.

## DB-TEST-028

Test profiles serán explícitos.

## DB-TEST-029

Fast profile no fingirá cobertura completa.

## DB-TEST-030

Full profile podrá ejecutar matriz multi-platform.

## DB-TEST-031

Test discovery será determinista.

## DB-TEST-032

Isolation será explícita.

## DB-TEST-033

No existirá una isolation strategy universal.

## DB-TEST-034

Transactional test no será Transaction test.

## DB-TEST-035

Transaction wrapper no se usará cuando invalide la propiedad probada.

## DB-TEST-036

Cleanup será parte del resultado operacional de una prueba.

## DB-TEST-037

Cleanup failure no será ignorado.

## DB-TEST-038

Tainted environment no será reutilizado silenciosamente.

## DB-TEST-039

UNKNOWN environment state permanecerá UNKNOWN.

## DB-TEST-040

Fixtures no serán isolation.

## DB-TEST-041

Factories no serán isolation.

## DB-TEST-042

Randomness será reproducible.

## DB-TEST-043

Random failure report incluirá seed cuando corresponda.

## DB-TEST-044

Application clock no será DB clock.

## DB-TEST-045

ORM IdentityMap tendrá pruebas de identidad.

## DB-TEST-046

persist no será probado como immediate INSERT.

## DB-TEST-047

remove no será probado como immediate DELETE.

## DB-TEST-048

flush no será probado como commit.

## DB-TEST-049

Partial entity no será tratado como complete entity.

## DB-TEST-050

Hydration no podrá crear segunda identidad managed.

## DB-TEST-051

Relationship tests cubrirán loading semantics.

## DB-TEST-052

N+1 testing podrá ser semántico.

## DB-TEST-053

Query Engine podrá probarse por etapas.

## DB-TEST-054

AST tests serán independientes del DBMS cuando sea posible.

## DB-TEST-055

Compiler tests no serán por sí solos execution tests.

## DB-TEST-056

Golden SQL no demostrará aceptación por el servidor.

## DB-TEST-057

SQL formatting irrelevante no romperá tests innecesariamente.

## DB-TEST-058

Binding tests incluirán tipos.

## DB-TEST-059

SQL injection tendrá tests negativos.

## DB-TEST-060

Identifier injection tendrá tests negativos.

## DB-TEST-061

Optimizer rewrite deberá preservar semántica.

## DB-TEST-062

Optimizer tendrá negative-rule tests.

## DB-TEST-063

Planner tendrá capability-dependent tests.

## DB-TEST-064

Capability UNKNOWN será probado.

## DB-TEST-065

Capability evidence conflicts serán probados.

## DB-TEST-066

Schema introspection tendrá pruebas reales.

## DB-TEST-067

Schema round-trip comparará semántica cuando corresponda.

## DB-TEST-068

Migration rollback no exigirá igualdad textual cuando la semántica sea equivalente.

## DB-TEST-069

Destructive migration tests permanecerán en sandbox.

## DB-TEST-070

Transaction semantics requerirán DB real.

## DB-TEST-071

Isolation semantics podrán requerir múltiples conexiones.

## DB-TEST-072

Concurrency harness evitará sleeps como mecanismo principal.

## DB-TEST-073

Deadlock victim no será asumida universalmente.

## DB-TEST-074

Deadlock semantics serán verificadas.

## DB-TEST-075

Locking tests dependerán de capabilities.

## DB-TEST-076

UNKNOWN transaction outcome tendrá pruebas.

## DB-TEST-077

Fault injection será test-only.

## DB-TEST-078

Fault injection no estará habilitada por defecto en producción.

## DB-TEST-079

Driver Conformance será distinto de Platform Conformance.

## DB-TEST-080

Driver no será Platform.

## DB-TEST-081

Dialect tendrá pruebas específicas.

## DB-TEST-082

Version matrix será explícita.

## DB-TEST-083

No todas las versiones deberán ejecutarse en cada commit.

## DB-TEST-084

Release conformance será más amplia que local fast tests.

## DB-TEST-085

Backup created no será backup restorable.

## DB-TEST-086

Restore tests reales proporcionarán evidencia superior de recuperabilidad.

## DB-TEST-087

Health scenarios incluirán degraded/unavailable.

## DB-TEST-088

Diagnostics tests utilizarán evidencia controlada.

## DB-TEST-089

Administration UNKNOWN outcomes serán probados.

## DB-TEST-090

Security testing será transversal.

## DB-TEST-091

Secret leakage tendrá pruebas.

## DB-TEST-092

Cross-tenant leakage tendrá pruebas.

## DB-TEST-093

Tenant A no observará Tenant B por defecto.

## DB-TEST-094

Sharding under-routing será error.

## DB-TEST-095

Topology generation será probada.

## DB-TEST-096

Replica health no será replica eligibility.

## DB-TEST-097

Sticky reads tendrán pruebas.

## DB-TEST-098

Cache categories no serán confundidas.

## DB-TEST-099

Physical cache hit no será usable hit.

## DB-TEST-100

Uncommitted data no llegará a cache compartido.

## DB-TEST-101

Event afterCommit failure no revertirá commit.

## DB-TEST-102

Telemetry disabled no cambiará semantics.

## DB-TEST-103

Persistent runtime state leakage tendrá pruebas.

## DB-TEST-104

FrankenPHP tendrá runtime tests.

## DB-TEST-105

RoadRunner tendrá runtime tests.

## DB-TEST-106

OpenSwoole tendrá coroutine isolation tests.

## DB-TEST-107

Connection reuse tendrá reset tests.

## DB-TEST-108

Poisoned connection deberá poder descartarse.

## DB-TEST-109

Memory growth será observable.

## DB-TEST-110

Expected cache growth no será automáticamente memory leak.

## DB-TEST-111

Resource budgets tendrán tests.

## DB-TEST-112

Timeouts tendrán tolerancias explícitas.

## DB-TEST-113

Performance baselines registrarán entorno.

## DB-TEST-114

Query count no será equivalente a performance.

## DB-TEST-115

Parallel tests estarán aislados.

## DB-TEST-116

Workers no limpiarán recursos ajenos.

## DB-TEST-117

Test resource ownership será verificable.

## DB-TEST-118

Cleanup nunca destruirá recursos no identificados como testing.

## DB-TEST-119

Destructive testing tendrá production guards.

## DB-TEST-120

Environment variable sola no será suficiente para operaciones destructivas críticas.

## DB-TEST-121

Test credentials seguirán least privilege.

## DB-TEST-122

Provisioning no dependerá obligatoriamente de Docker.

## DB-TEST-123

Provisioning será distinto de schema migration.

## DB-TEST-124

Mutable TestRun state no será static global.

## DB-TEST-125

TestCase state será scoped.

## DB-TEST-126

Test doubles declararán sus limitaciones.

## DB-TEST-127

Fake DB no fingirá ser un DBMS real completo.

## DB-TEST-128

Contract tests podrán ejecutarse sobre implementaciones reales.

## DB-TEST-129

Regression bug deberá tender a producir regression test.

## DB-TEST-130

Flaky test será tratado como defecto.

## DB-TEST-131

Retry no ocultará flakiness.

## DB-TEST-132

Quarantine no eliminará evidencia.

## DB-TEST-133

Infrastructure error no será assertion failure.

## DB-TEST-134

Required environment unavailable podrá fallar CI.

## DB-TEST-135

Optional local environment unavailable podrá skippear explícitamente.

## DB-TEST-136

Test reports incluirán environment fingerprint.

## DB-TEST-137

Capability conformance será reportable.

## DB-TEST-138

CI tendrá niveles de matriz.

## DB-TEST-139

Test order será independiente por defecto.

## DB-TEST-140

Scenario ordering será local al scenario.

## DB-TEST-141

Architecture dependencies tendrán tests.

## DB-TEST-142

Documented invariants podrán mapearse a tests.

## DB-TEST-143

Line coverage no será specification coverage.

## DB-TEST-144

100% line coverage no demostrará correctness total.

## DB-TEST-145

Property-based testing podrá complementar tests tradicionales.

## DB-TEST-146

Fuzz failures serán reproducibles cuando sea posible.

## DB-TEST-147

New DB version no será automáticamente supported.

## DB-TEST-148

Support declaration requerirá evidencia.

## DB-TEST-149

Testing Core no dependerá obligatoriamente de PHPUnit.

## DB-TEST-150

Testing Core no dependerá obligatoriamente de Pest.

## DB-TEST-151

Runner integrations serán adapters.

## DB-TEST-152

Database assertions podrán reutilizarse entre runners.

## DB-TEST-153

RefreshDatabase será política, no arquitectura completa.

## DB-TEST-154

La complejidad de aislamiento residirá en infraestructura.

## DB-TEST-155

Developer tests deberán mantener API simple.

## DB-TEST-156

Test context respetará tenant/shard/connection.

## DB-TEST-157

Assertions respetarán transaction visibility.

## DB-TEST-158

Una assertion desde otra conexión no asumirá visibilidad de cambios no confirmados.

## DB-TEST-159

Testing telemetry estará separada de production telemetry.

## DB-TEST-160

Test DB naming será defensa adicional, no única defensa.

## DB-TEST-161

Remote test databases requerirán designación explícita.

## DB-TEST-162

Production endpoint detection abortará operaciones destructivas por defecto.

## DB-TEST-163

Database testing no dependerá de HTTP.

## DB-TEST-164

Long-running test infrastructure podrá ejecutarse en workers.

## DB-TEST-165

Cross-platform semantics tendrán assertions semánticas, no sólo SQL textual.

## DB-TEST-166

Capability discovery y conformance testing serán conceptos distintos.

## DB-TEST-167

Una capability declarada podrá validarse mediante conformance tests.

## DB-TEST-168

Conformance failure podrá revelar capability metadata incorrecta.

## DB-TEST-169

Test evidence será atribuible a un entorno concreto.

## DB-TEST-170

VoltStack no declarará portabilidad basándose únicamente en mocks.

## DB-TEST-171

VoltStack no declarará portabilidad basándose únicamente en SQLite.

## DB-TEST-172

VoltStack no declarará compatibilidad basándose únicamente en SQL compilado.

## DB-TEST-173

Las propiedades distribuidas tendrán pruebas distribuidas cuando sea necesario.

## DB-TEST-174

Las propiedades concurrentes tendrán pruebas concurrentes cuando sea necesario.

## DB-TEST-175

Las propiedades de recuperación tendrán pruebas de recuperación cuando sea necesario.

## DB-TEST-176

Las propiedades de seguridad tendrán pruebas negativas.

## DB-TEST-177

Las propiedades de aislamiento tendrán pruebas de contaminación.

## DB-TEST-178

Las propiedades de cleanup tendrán pruebas de failure.

## DB-TEST-179

La ausencia de tests fallidos no equivaldrá automáticamente a evidencia suficiente de soporte.

## DB-TEST-180

La confianza de VoltStack Database será construida mediante evidencia complementaria de múltiples niveles.

---

# 330. Modelo formal

Sea:

```text
P = property under test
T = test
E = environment
```

La evidencia obtenida puede expresarse conceptualmente:

```text
Evidence(T, P, E)
```

Una prueba sólo demuestra una propiedad dentro del alcance de su entorno y modelo.

Por tanto:

```text
Evidence(UnitTest, TransactionIsolation, Mock)
```

no implica:

```text
Evidence(PostgreSQL, TransactionIsolation)
```

---

# 331. Portabilidad

Sea:

```text
Platforms = {MySQL, MariaDB, PostgreSQL, SQLite}
```

Una propiedad portable `P` deberá satisfacer los contratos requeridos para cada plataforma oficialmente incluida:

```text
∀ p ∈ SupportedPlatforms(P):
    Conformance(P, p) = PASS
```

salvo diferencias explícitamente documentadas por capabilities.

---

# 332. Capability-aware conformance

Más precisamente:

```text
Required(P, PlatformCapabilities)
        ↓
ExpectedBehavior
        ↓
ObservedBehavior
        ↓
ConformanceResult
```

---

# 333. Testing y Capability System

El flujo será bidireccional en términos de validación:

```text
Capability Definition
        ↓
Conformance Expectation
        ↓
Real Database Test
        ↓
Observed Evidence
```

Pero runtime capability resolution no dependerá de que CI haya ejecutado previamente una prueba.

---

# 334. Arquitectura consolidada

```text
                     VOLTSTACK DATABASE TESTING
                                │
                  ┌─────────────┴─────────────┐
                  │                           │
             Test Planning              Environments
                  │                           │
          ┌───────┼────────┐          ┌──────┼───────┐
          ▼       ▼        ▼          ▼      ▼       ▼
        Unit  Integration Conformance MySQL MariaDB PostgreSQL
          │       │        │                         │
          │       │        └───────────────┐         │
          │       │                        │         │
          │       └───────────────┐        │         │
          └───────────────┐       │        │         │
                          ▼       ▼        ▼         ▼
                        Test Execution & Evidence
                                  │
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
     Correctness              Failure Modes          Performance
          │                       │                       │
          └───────────────────────┼───────────────────────┘
                                  ▼
                             Test Report
                                  │
                  ┌───────────────┼───────────────┐
                  ▼               ▼               ▼
              Developer           CI            Release
```

---

# 335. Flujo de una prueba real

Ejemplo:

```text
TransactionSavepointConformanceTest
        ↓
Requires:
 transaction.savepoint
        ↓
PostgreSQL environment selected
        ↓
Provision
        ↓
Discover capabilities
        ↓
Prepare schema
        ↓
Connection A
        ↓
BEGIN
        ↓
SAVEPOINT
        ↓
Mutation
        ↓
ROLLBACK TO SAVEPOINT
        ↓
Assertion
        ↓
ROLLBACK
        ↓
Connection reset
        ↓
Environment cleanup
        ↓
Evidence
```

---

# 336. Flujo de prueba portable

```text
Shared Contract
      │
      ├── MySQL
      │      ↓
      │    Result
      │
      ├── MariaDB
      │      ↓
      │    Result
      │
      ├── PostgreSQL
      │      ↓
      │    Result
      │
      └── SQLite
             ↓
           Result
```

Luego:

```text
Conformance Aggregator
        ↓
Capability-aware Report
```

---

# 337. Filosofía de testing de VoltStack

VoltStack buscará evitar dos extremos.

El primero:

```text
everything mocked
```

produce pruebas rápidas pero puede ocultar incompatibilidades reales.

El segundo:

```text
everything requires a real DB server
```

produce suites lentas y hace difícil aislar errores.

La estrategia será:

```text
Pure logic
   ↓
Unit

Component collaboration
   ↓
Component tests

Database semantics
   ↓
Real integration

Portability
   ↓
Conformance matrix

Failure behavior
   ↓
Fault testing

Performance
   ↓
Benchmarks
```

---

# 338. Regla final

> **VoltStack Database deberá probar cada garantía en el nivel que realmente pueda demostrarla. Las abstracciones, mocks, fakes y SQLite podrán acelerar el desarrollo, pero nunca sustituirán la evidencia de un motor real cuando la garantía dependa de SQL, protocolo, concurrencia, transacciones, locking, capacidades o comportamiento específico de plataforma.**

Por tanto:

```text
Mock Evidence
     ≠
Database Evidence
```

y:

```text
One Platform Passing
     ≠
Portable
```

La confianza se construirá mediante:

```text
Isolation
+
Determinism
+
Real Integration
+
Cross-Platform Conformance
+
Failure Injection
+
Security Testing
+
Runtime Testing
+
Performance Testing
+
Reproducibility
```

---

# 339. Relación con los siguientes documentos

Esta arquitectura será especializada mediante:

```text
283_DATABASE_TESTING_ARCHITECTURE.md
│
├── 284_DATABASE_UNIT_TESTING_SYSTEM.md
│
├── 285_DATABASE_INTEGRATION_TESTING_SYSTEM.md
│
├── 286_DATABASE_DATABASE_TEST_ENVIRONMENT_SYSTEM.md
│
├── 287_DATABASE_TRANSACTIONAL_TESTING_SYSTEM.md
│
├── 288_DATABASE_DATABASE_FAKE_AND_MOCK_SYSTEM.md
│
├── 289_DATABASE_QUERY_ASSERTION_SYSTEM.md
│
├── 290_DATABASE_SCHEMA_TESTING_SYSTEM.md
│
├── 291_DATABASE_ORM_TESTING_SYSTEM.md
│
├── 292_DATABASE_DRIVER_CONFORMANCE_TESTING_SYSTEM.md
│
└── 293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

Cada documento especializará una responsabilidad, pero todos utilizarán el mismo modelo de:

```text
Test Context
      ↓
Environment
      ↓
Capabilities
      ↓
Isolation
      ↓
Execution
      ↓
Assertions
      ↓
Cleanup
      ↓
Evidence
```

---

# 340. Siguiente documento

```text
284_DATABASE_UNIT_TESTING_SYSTEM.md
```

El siguiente documento definirá específicamente la capa de **Unit Testing** de VoltStack Database:

```text
Database Unit Testing
│
├── Unit Test Boundaries
├── Pure Components
├── Test Subjects
├── Test Doubles
├── Deterministic Context
├── AST Testing
├── Metadata Testing
├── Type Testing
├── Planner Testing
├── Compiler Unit Testing
├── Capability Evaluation Testing
├── Error Testing
├── Property Testing
└── Architecture Invariants
```

manteniendo como regla:

> **Una prueba unitaria deberá aislar la lógica que realmente puede evaluarse sin infraestructura externa; simular una base de datos para mantener una prueba “unitaria” nunca deberá utilizarse para afirmar que se ha comprobado el comportamiento real de esa base de datos.**