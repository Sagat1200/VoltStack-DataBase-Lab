# 02_DATABASE_DESIGN_PRINCIPLES.md

# VoltStack Database — Design Principles

## 1. Propósito

Este documento define los principios de diseño obligatorios para:

```text
VoltStack/Quantum/Database
```

Su propósito es establecer los criterios que deberán utilizarse durante:

```text
diseño
implementación
revisión arquitectónica
refactorización
extensión
optimización
testing
mantenimiento
evolución
```

del sistema Database.

Los principios definidos aquí complementan:

```text
00_DATABASE_PROJECT_CONTEXT.md
01_DATABASE_ARCHITECTURE.md
```

y deberán considerarse restricciones arquitectónicas del sistema, no simples recomendaciones de estilo.

---

# 2. Objetivo

El sistema Database deberá permanecer:

```text
modular
predecible
extensible
observable
seguro
eficiente
portable
testeable
runtime-safe
```

incluso mientras aumente su complejidad.

La principal finalidad de estos principios será impedir que el crecimiento funcional produzca nuevamente:

```text
dependencias circulares
responsabilidades ambiguas
God Objects
estado compartido accidental
acoplamiento ORM/SQL
acoplamiento Query/Driver
duplicación de motores internos
abstracciones innecesarias
```

---

# 3. Principio rector

El principio rector será:

> La simplicidad de la API pública no deberá conseguirse sacrificando claridad arquitectónica interna.

VoltStack deberá poder proporcionar:

```php
$users = User::where('active', true)->get();
```

sin implementar internamente un flujo monolítico.

La operación podrá recorrer:

```text
Model API
   ↓
ORM
   ↓
Query Model
   ↓
AST
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
   ↓
Connection
   ↓
Driver
```

La complejidad interna deberá estar organizada, no eliminada mediante atajos.

---

# 4. Simplicidad exterior, rigor interior

La API pública deberá privilegiar:

```text
clarity
discoverability
reasonable defaults
minimal ceremony
strong typing
predictable behavior
```

Mientras que los internals deberán privilegiar:

```text
explicit contracts
clear boundaries
state ownership
dependency direction
immutability
testability
```

Por tanto:

```text
Simple API
≠
Simple architecture
```

Una API sencilla puede estar sostenida por una arquitectura rigurosa.

---

# 5. Separation of Concerns

Cada componente deberá tener una responsabilidad claramente identificable.

Ejemplos:

```text
Builder
→ construir una representación de consulta

Semantic Analyzer
→ resolver significado

Optimizer
→ transformar representación

Planner
→ decidir estrategia

Compiler
→ producir SQL

Executor
→ ejecutar

Connection
→ administrar interacción lógica con driver

Driver
→ comunicación técnica con database

Hydrator
→ materializar resultados

UnitOfWork
→ registrar estado de entidades
```

Ninguna de estas responsabilidades deberá mezclarse arbitrariamente.

---

# 6. Single Responsibility

Cada clase, servicio o subsistema deberá responder principalmente a una razón de cambio.

Ejemplo correcto:

```text
PostgreSqlCompiler
```

cambia cuando cambia:

```text
la estrategia de compilación PostgreSQL
```

Ejemplo problemático:

```text
PostgreSqlDatabaseManager
```

si controla simultáneamente:

```text
connections
SQL compilation
migrations
entities
transactions
telemetry
```

---

# 7. Cohesión

Los elementos dentro de un módulo deberán estar relacionados por una responsabilidad común.

Alta cohesión:

```text
Query/AST/
├── QueryNode
├── SelectNode
├── ExpressionNode
├── PredicateNode
└── JoinNode
```

Baja cohesión:

```text
Database/Utils/
├── QueryNode
├── ConnectionPool
├── EntityHydrator
├── MigrationRunner
└── RetryPolicy
```

Se evitarán módulos genéricos que acumulen componentes sin relación arquitectónica.

---

# 8. Bajo acoplamiento

Las dependencias entre dominios deberán reducirse al mínimo necesario.

Preferir:

```text
ORM
 ↓
Query Contracts
```

sobre:

```text
ORM
 ↓
ConcreteQueryCompiler
 ↓
ConcreteConnection
 ↓
PDO
```

Cada dependencia deberá justificar por qué el componente necesita conocerla.

---

# 9. Dependency Direction

La dirección principal será descendente:

```text
High-Level Policy
       ↓
Lower-Level Mechanism
```

Ejemplo:

```text
ORM
 ↓
Query
 ↓
Execution
 ↓
Connection
 ↓
Driver
```

Las capas inferiores no deberán conocer las superiores.

---

# 10. Dependency Inversion

Cuando un componente de alto nivel necesite funcionalidad de infraestructura deberá depender de una abstracción.

Preferir:

```php
final class QueryExecutor
{
    public function __construct(
        private ConnectionResolverInterface $connections,
    ) {
    }
}
```

sobre:

```php
final class QueryExecutor
{
    public function __construct(
        private PdoConnection $connection,
    ) {
    }
}
```

cuando la abstracción represente una frontera real.

---

# 11. Las interfaces deben representar fronteras

No toda clase necesita una interface.

Una interface deberá existir cuando represente:

```text
public contract
extension point
implementation boundary
platform abstraction
test seam
integration port
```

No deberá crearse únicamente para cumplir mecánicamente un patrón.

Evitar:

```text
FooInterface
    ↓
Foo
```

si no existe una necesidad arquitectónica real de sustitución.

---

# 12. Abstracción mínima suficiente

VoltStack deberá abstraer diferencias reales, no diferencias hipotéticas.

Una abstracción se justificará cuando:

```text
existan varias implementaciones
se espere una extensión pública
exista una frontera de dominio
sea necesario aislar infraestructura
sea necesaria para testing o runtime
```

La arquitectura evitará introducir capas sin responsabilidad propia.

---

# 13. No accidental complexity

Toda capa deberá justificar su existencia.

Ejemplo:

```text
AST
Semantic
Optimizer
Planner
Compiler
```

son capas válidas porque cada una transforma o interpreta información de forma distinta.

No deberán crearse capas como:

```text
QueryProcessingManager
QueryProcessingCoordinator
QueryProcessingHandler
QueryProcessingService
```

si únicamente delegan sin aportar semántica.

---

# 14. Explicit over implicit

El sistema privilegiará comportamiento explícito.

Ejemplos:

```text
explicit connection selection
explicit transaction ownership
explicit tenant context
explicit raw SQL
explicit state lifecycle
explicit capability checks
```

sobre inferencias difíciles de predecir.

---

# 15. Convención con límites

VoltStack podrá utilizar convenciones para mejorar DX.

Por ejemplo:

```text
User
→ users
```

pero las convenciones deberán:

```text
ser documentadas
tener overrides
ser deterministas
no esconder comportamiento crítico
```

---

# 16. Principle of Least Surprise

Las APIs deberán comportarse de forma coherente con las expectativas generadas por sus nombres.

Ejemplo:

```php
$query->first();
```

deberá obtener como máximo un resultado.

No deberá ejecutar accidentalmente:

```text
full collection hydration
unexpected writes
implicit flush
hidden transaction commit
```

---

# 17. No hidden persistence

Una operación aparentemente de lectura no deberá producir escrituras inesperadas.

Ejemplo:

```php
$userRepository->find(10);
```

no deberá provocar automáticamente:

```text
flush
insert
update
commit
```

salvo comportamiento explícitamente documentado.

---

# 18. No hidden flush

VoltStack no deberá ejecutar automáticamente:

```text
EntityManager::flush()
```

como efecto lateral de operaciones ordinarias de consulta.

El flush deberá ser explícito salvo APIs específicamente documentadas como operaciones atómicas.

---

# 19. Explicit transaction semantics

Los límites de transacción deberán estar claramente definidos.

Correcto:

```php
Database::transaction(function () {
    // ...
});
```

También:

```php
$transaction = $manager->begin();
```

No deberán existir commits invisibles dentro de componentes aparentemente independientes.

---

# 20. No duplicated persistence engines

VoltStack podrá ofrecer:

```text
Data Mapper
Active Record API
```

pero ambos deberán utilizar el mismo motor de persistencia.

Arquitectura:

```text
Active Record
      │
      ▼
EntityManager
      │
      ▼
UnitOfWork
      │
      ▼
Persistence Engine
```

No:

```text
Active Record Engine
+
Data Mapper Engine
```

---

# 21. One Query Engine

De forma equivalente, todos los sistemas estructurados de consultas deberán converger sobre el mismo Query Engine.

```text
Query Builder ──────┐
                    │
ORM Query ──────────┼──→ Query Model → AST
                    │
Persistence ────────┘
```

Esto evita inconsistencias en:

```text
bindings
dialects
platform support
telemetry
optimization
security
```

---

# 22. Structured representation before SQL

Cuando sea posible, una operación deberá representarse estructuralmente antes de producir SQL.

Preferir:

```text
Intent
 ↓
Model
 ↓
AST
 ↓
Plan
 ↓
SQL
```

sobre:

```text
Intent
 ↓
string concatenation
```

Esto será especialmente obligatorio para:

```text
Query Builder
ORM persistence
Schema
Migrations
```

---

# 23. SQL es una salida, no el modelo

Dentro del Query Engine:

```text
SQL
```

deberá considerarse una representación compilada.

La consulta lógica deberá existir antes del SQL.

Esto permitirá:

```text
validation
rewriting
optimization
portability
diagnostics
tooling
```

---

# 24. Query AST independence

El Query AST no deberá contener:

```text
PDO
Connection
Driver
EntityManager
Runtime state
```

Podrá contener conceptos semánticos de consulta, pero deberá permanecer desacoplado de ejecución.

---

# 25. Semantic correctness before optimization

El sistema deberá validar significado antes de optimizar.

Flujo:

```text
AST
 ↓
Semantic Validation
 ↓
Optimizer
```

No deberá optimizarse una consulta estructuralmente inválida esperando que una regla posterior la corrija.

---

# 26. Correctness before performance

Toda optimización deberá preservar:

```text
semantics
transaction guarantees
data integrity
security
```

Si existe conflicto:

```text
correctness
>
performance
```

---

# 27. Performance must be measurable

No se añadirá complejidad arquitectónica por supuestas mejoras de rendimiento sin medición.

Toda optimización importante deberá poder validarse mediante:

```text
benchmark
profiling
telemetry
memory analysis
query count
latency measurement
```

---

# 28. Optimization boundaries

VoltStack optimizará aquello que conoce.

Ejemplos apropiados:

```text
query normalization
query deduplication
batch relation loading
metadata compilation
hydration planning
prepared statement reuse
```

No intentará sustituir:

```text
PostgreSQL query optimizer
MySQL execution engine
database buffer manager
physical storage planner
```

---

# 29. Immutable representations

Se favorecerá inmutabilidad en objetos que representen intención o estructura.

Ejemplos:

```text
AST nodes
Query Plans
Compiled Query
Metadata
Configuration
Capabilities
Value Objects
```

Esto reduce:

```text
unexpected mutation
shared-state bugs
concurrency issues
cache invalidation complexity
```

---

# 30. Controlled mutability

La mutabilidad estará permitida donde representa estado real.

Ejemplos:

```text
Connection
TransactionContext
UnitOfWork
IdentityMap
Cursor
RuntimeScope
```

Toda mutabilidad deberá tener:

```text
owner
lifecycle
reset rule
concurrency semantics
```

---

# 31. State ownership

Ningún estado mutable deberá existir sin propietario claro.

Ejemplo:

```text
State                   Owner

Entity state map        UnitOfWork
Object identity         IdentityMap
Transaction nesting     TransactionContext
Physical connection     Connection
Cursor position         Cursor
Tenant selection        TenantContext
```

---

# 32. No global mutable state

No se permitirá almacenar globalmente:

```text
current EntityManager
current Connection
current Transaction
current Tenant
current UnitOfWork
active Query
```

Especialmente en runtimes persistentes.

---

# 33. Scope awareness

Todo servicio mutable deberá declarar su alcance conceptual.

Posibles scopes:

```text
process
worker
request
transaction
persistence context
query
cursor
```

No deberá deducirse implícitamente.

---

# 34. Process-safe design

Los servicios a nivel proceso deberán ser preferentemente:

```text
immutable
stateless
configuration-based
thread/concurrency safe when applicable
```

Ejemplos:

```text
Dialect
Platform
Compiled Metadata
Compiler Registry
Type Registry
```

---

# 35. Request isolation

Todo estado asociado a una petición deberá poder destruirse o reiniciarse.

Ejemplo:

```text
Request A
 └── DatabaseContext A

Request complete
       ↓
RESET

Request B
 └── DatabaseContext B
```

Nunca:

```text
DatabaseContext A
    ↓ accidental reuse
DatabaseContext B
```

---

# 36. Runtime neutrality

Aunque FrankenPHP sea inicialmente el runtime recomendado, los internals de Database no deberán acoplarse a él.

Preferir:

```text
RuntimeLifecycleInterface
       ↑
       ├── FrankenPHP Adapter
       ├── RoadRunner Adapter
       └── OpenSwoole Adapter
```

sobre:

```text
if FrankenPHP
if RoadRunner
if OpenSwoole
```

distribuidos por Database.

---

# 37. Portable core, specialized edges

El núcleo deberá ser portable.

Las optimizaciones específicas deberán ubicarse cerca de las fronteras.

Ejemplo:

```text
Query AST
→ portable

Planner Strategy
→ platform aware

Compiler
→ platform specific

Driver
→ implementation specific
```

---

# 38. Capability-driven design

VoltStack deberá preguntar:

```text
¿la plataforma soporta esta característica?
```

en lugar de:

```text
¿esta base de datos se llama PostgreSQL?
```

Preferir:

```php
if ($platform->capabilities()->supportsReturning()) {
    // ...
}
```

sobre:

```php
if ($driver === 'pgsql') {
    // ...
}
```

---

# 39. Capability locality

Las capacidades deberán resolverse centralmente.

No se permitirán cientos de comprobaciones específicas distribuidas por el código.

Correcto:

```text
Platform
 └── Capabilities
```

Incorrecto:

```text
ORM       → if pgsql
Migration → if pgsql
Schema    → if pgsql
Query     → if pgsql
```

---

# 40. Graceful degradation

Cuando una plataforma no soporte una característica, deberá declararse una estrategia explícita.

```text
Native
Emulated
Fallback
Unsupported
```

Nunca deberá producirse degradación silenciosa que altere semántica.

---

# 41. Portability without lowest-common-denominator design

Portabilidad no significa limitar todas las bases de datos a las capacidades de SQLite.

VoltStack deberá permitir características avanzadas específicas.

Ejemplo:

```php
$query->returning('id');
```

Si una plataforma lo soporta:

```text
native implementation
```

Si no:

```text
fallback or explicit unsupported exception
```

---

# 42. Escape hatches are first-class

El framework deberá reconocer que existen consultas que no deben abstraerse.

Por ello deberá existir:

```text
Raw SQL
Raw Expression
Native Platform Feature
Driver Access
```

como escape hatches explícitos.

No deberán convertirse en vías internas predeterminadas.

---

# 43. Safe by default

Las APIs comunes deberán utilizar automáticamente:

```text
prepared statements
bindings
safe identifier handling
transaction safety
credential protection
runtime state reset
```

El desarrollador deberá realizar una acción explícita para abandonar estos defaults.

---

# 44. Parameterization by default

Los valores del usuario deberán convertirse en parámetros.

Ejemplo:

```php
$query->where('email', $email);
```

deberá conceptualizarse como:

```text
email = Parameter(:p1)
```

no como:

```text
email = '$email'
```

---

# 45. Values and identifiers are different

El sistema deberá diferenciar estrictamente:

```text
value
identifier
expression
raw fragment
```

Un nombre de tabla no es un parámetro de valor.

Cada categoría tendrá reglas específicas.

---

# 46. Raw boundaries must be visible

Una API raw deberá nombrarse claramente como tal.

Ejemplos:

```php
DB::raw(...)
$query->rawExpression(...)
$connection->executeRaw(...)
```

No deberá esconderse SQL arbitrario dentro de APIs aparentemente seguras.

---

# 47. Fail closed for ambiguous security behavior

Cuando Database no pueda determinar con seguridad cómo tratar una operación sensible deberá preferir:

```text
explicit error
```

sobre:

```text
unsafe automatic fallback
```

---

# 48. Sensitive-data minimization

Database no deberá exponer datos sensibles innecesariamente en:

```text
logs
exceptions
telemetry
debug output
profiler
query dumps
```

Especial atención a:

```text
passwords
tokens
credentials
personal information
bindings
connection strings
```

---

# 49. Observability by design

Las operaciones importantes deberán poder observarse.

Ejemplos:

```text
connection
query
transaction
hydration
flush
migration
schema operation
```

Pero la observabilidad deberá añadirse mediante puntos bien definidos.

---

# 50. Telemetry must not own business semantics

Telemetry podrá observar:

```text
QueryCompleted
TransactionCommitted
```

pero no deberá decidir:

```text
qué query ejecutar
si una entidad debe persistirse
qué transaction debe abrirse
```

---

# 51. Instrumentation should be cheap when disabled

Cuando observabilidad avanzada esté desactivada, el overhead deberá ser mínimo.

Preferir:

```text
NullTelemetryPort
```

a construir estructuras complejas que luego no serán utilizadas.

---

# 52. Events are notifications, not hidden control flow

Los eventos podrán informar de hechos.

Ejemplo:

```text
QueryExecuted
EntityPersisted
TransactionCommitted
```

No deberán convertirse en una red invisible que controle el flujo central de Database.

Las operaciones críticas deberán seguir siendo comprensibles siguiendo llamadas directas.

---

# 53. Hooks must preserve invariants

Los hooks y extensiones nunca deberán poder saltarse invariantes críticos sin una API explícitamente unsafe.

Por ejemplo:

```text
beforeCompile
```

podrá modificar una representación autorizada.

No deberá recibir acceso arbitrario a:

```text
active PDO handle
UnitOfWork internals
worker-global state
```

---

# 54. Extension by contracts

Los puntos de extensión deberán estar definidos mediante contratos.

Ejemplos:

```text
DriverInterface
DatabaseTypeInterface
OptimizationRuleInterface
HydratorInterface
DialectExtensionInterface
QueryFunctionInterface
```

Las extensiones no deberán depender de reflexión sobre internals.

---

# 55. No universal plugin context

No se deberá proporcionar a extensiones un objeto con acceso a todo Database.

Evitar:

```text
DatabaseExtensionContext
 ├── ORM internals
 ├── Connections
 ├── Cache
 ├── Telemetry
 ├── Transactions
 ├── Runtime
 └── Everything
```

Preferir contextos específicos por capacidad.

---

# 56. Open for extension, guarded against mutation

VoltStack deberá ser extensible, pero las extensiones deberán operar dentro de fronteras.

Ejemplo:

```text
register query function
register type
register driver
register optimization rule
```

no:

```text
replace arbitrary private service at runtime
```

salvo mecanismo avanzado explícitamente diseñado.

---

# 57. Registries must be specialized

Preferir:

```text
DriverRegistry
TypeRegistry
CompilerRegistry
HydratorRegistry
```

sobre:

```text
DatabaseRegistry
```

que almacene objetos heterogéneos.

---

# 58. Registry freeze

Los registries podrán ser mutables durante bootstrap.

Después:

```text
bootstrap complete
       ↓
registry freeze
```

para evitar modificaciones impredecibles durante ejecución.

Especialmente bajo workers persistentes.

---

# 59. Bootstrap must not perform unnecessary I/O

Registrar Database no deberá abrir automáticamente conexiones.

Durante bootstrap:

```text
configuration
services
registries
metadata definitions
```

Después, bajo demanda:

```text
physical database connection
```

---

# 60. Lazy infrastructure

Los recursos costosos deberán adquirirse cuando sean necesarios.

Ejemplo:

```text
Connection proxy
      ↓ first use
Physical connection
```

El lazy loading de infraestructura deberá ser transparente pero predecible.

---

# 61. Lazy domain behavior must be explicit

La carga lazy de relaciones es distinta.

Dado que puede causar consultas inesperadas:

```text
Entity
 ↓ property access
Query
```

deberá ser configurable y observable.

---

# 62. N+1 prevention without magic

VoltStack podrá detectar N+1 y ofrecer:

```text
warnings
diagnostics
batch loaders
development exceptions
```

pero no deberá reescribir silenciosamente lógica de negocio de forma impredecible.

---

# 63. Batch where semantics permit

Operaciones múltiples podrán agruparse cuando la semántica sea equivalente.

Ejemplos:

```text
relation loading
bulk insert
prepared statements
persistence operations
```

El batching nunca deberá cambiar:

```text
ordering guarantees
transaction semantics
generated identifier expectations
event semantics
```

---

# 64. Streaming over materialization for large datasets

El sistema deberá permitir procesamiento incremental.

Preferir para grandes conjuntos:

```php
foreach ($query->stream() as $row) {
    // ...
}
```

sobre materializar siempre toda la colección.

---

# 65. Memory is a first-class resource

El rendimiento no será evaluado únicamente por tiempo.

También deberán considerarse:

```text
peak memory
object allocation
result buffering
IdentityMap growth
metadata duplication
query-plan retention
```

---

# 66. Bounded request state

Los subsistemas request-scoped deberán poder limitar crecimiento.

Ejemplos:

```text
IdentityMap
UnitOfWork
query diagnostics
hydration buffers
```

Para tareas de larga duración deberán existir APIs de:

```text
clear
detach
flush-and-clear
stream
chunk
```

---

# 67. Long-running operation awareness

Database deberá soportar procesos que no correspondan a un request HTTP.

Ejemplos:

```text
queue worker
CLI command
import
export
migration
background job
```

Estos contextos deberán disponer de lifecycle explícito equivalente.

---

# 68. Transactions are correctness boundaries

Las transacciones se diseñarán primero por consistencia, no por comodidad.

Una operación transaccional deberá tener claramente:

```text
connection
begin
scope
commit
rollback
```

---

# 69. Retry is not transparent by default

Reintentar una lectura puede ser seguro.

Reintentar:

```text
INSERT
payment operation
external side effect
```

puede no serlo.

Las políticas de retry deberán considerar:

```text
idempotency
transaction state
statement type
error category
```

---

# 70. Error classification

Los errores deberán clasificarse por semántica.

Ejemplos:

```text
connection failure
timeout
deadlock
constraint violation
syntax error
serialization failure
authentication failure
```

No deberán tratarse todos como:

```text
DatabaseException
```

sin información adicional.

---

# 71. Exceptions should preserve causality

Una excepción de alto nivel deberá poder preservar su causa.

Ejemplo:

```text
UniqueConstraintViolationException
        ↓ caused by
DriverException
        ↓ caused by
PDOException
```

Esto facilita diagnóstico sin filtrar internals directamente.

---

# 72. Error messages must be actionable

Los errores orientados a desarrollador deberán indicar cuando sea posible:

```text
qué falló
dónde falló
qué componente lo detectó
qué configuración está relacionada
qué operación estaba ocurriendo
```

sin revelar información sensible.

---

# 73. Domain errors and infrastructure errors are distinct

Ejemplo:

```text
EntityNotFound
```

no deberá confundirse con:

```text
ConnectionLost
```

Las capas superiores podrán reaccionar de forma diferente.

---

# 74. Schema and ORM are separate models

El esquema físico y el mapping ORM no deberán confundirse.

```text
Schema Metadata
≠
ORM Metadata
```

Aunque puedan relacionarse.

Ejemplo:

```text
database column
```

es un concepto diferente de:

```text
entity property mapping
```

---

# 75. Schema is not subordinate to ORM

Debe ser posible utilizar:

```text
Schema
Migration
Query Builder
```

sin instalar o activar ORM.

El Schema System no dependerá de entidades.

---

# 76. ORM consumes schema knowledge selectively

El ORM podrá usar Schema Metadata para:

```text
validation
type resolution
diagnostics
mapping verification
```

pero no deberá necesitar introspección en cada query.

---

# 77. Metadata should be compilable

La metadata basada en reflexión deberá poder transformarse en representación eficiente.

```text
Attributes
 ↓
Load
 ↓
Validate
 ↓
Compile
 ↓
Runtime Metadata
```

Esto será especialmente importante en producción.

---

# 78. Reflection is a bootstrap/build concern where possible

La reflexión deberá reducirse durante hot paths.

Preferir:

```text
build/bootstrap
→ reflect

runtime
→ consume compiled metadata
```

---

# 79. Identity consistency

Dentro de un mismo `PersistenceContext`, una identidad lógica deberá representar una instancia canónica cuando aplique.

Ejemplo:

```php
$a = $repository->find(1);
$b = $repository->find(1);

$a === $b;
```

Esto deberá permanecer como principio fundamental del ORM Data Mapper.

---

# 80. Persistence context boundaries

La identidad anterior no deberá extenderse globalmente.

```text
PersistenceContext A
User#1 → Object A

PersistenceContext B
User#1 → Object B
```

Esto es correcto.

---

# 81. UnitOfWork only tracks persistence state

El UnitOfWork no deberá convertirse en:

```text
query builder
event bus
transaction manager
SQL compiler
connection manager
```

Su responsabilidad será gestionar estado de persistencia.

---

# 82. Flush as a pipeline

`flush()` deberá conceptualizarse como pipeline.

```text
Detect Changes
      ↓
Validate State
      ↓
Build Persistence Plan
      ↓
Order Operations
      ↓
Execute
      ↓
Synchronize State
```

No como un método monolítico que realiza todas las tareas internamente.

---

# 83. Flush must be deterministic

Dado el mismo:

```text
UnitOfWork state
Metadata
Configuration
```

el Persistence Planner deberá generar una secuencia lógica reproducible.

Cuando existan decisiones variables deberán estar explícitamente modeladas.

---

# 84. Persistence ordering

La persistencia deberá respetar dependencias.

Ejemplo:

```text
Parent INSERT
   ↓
Child INSERT
```

o:

```text
Join table DELETE
   ↓
Entity DELETE
```

La planificación de estas dependencias pertenecerá al Persistence Planner.

---

# 85. Cascades must be explicit

Las operaciones cascade deberán declararse mediante metadata o configuración.

Ejemplos:

```text
cascade persist
cascade remove
cascade detach
```

No deberán inferirse de forma peligrosa.

---

# 86. Lazy loading is an ORM policy

Lazy loading no deberá implementarse en:

```text
Driver
Connection
Query Compiler
```

Es una política de ORM/Relationship.

---

# 87. Hydration is not persistence

El Hydrator construye representaciones desde resultados.

```text
Result
 ↓
Hydrator
 ↓
Entity
```

No deberá ejecutar:

```text
INSERT
UPDATE
DELETE
```

---

# 88. Hydration strategies are pluggable

El sistema podrá soportar:

```text
EntityHydrator
DtoHydrator
ArrayHydrator
ScalarHydrator
StreamHydrator
```

sin alterar Executor.

---

# 89. Result ownership

Un resultado deberá definir claramente quién controla:

```text
cursor
buffer
connection dependency
closure
resource release
```

Los cursores deberán cerrarse explícita o determinísticamente.

---

# 90. Resource cleanup is mandatory

Todos los recursos con lifecycle deberán disponer de cleanup.

Ejemplos:

```text
connections
statements
cursors
transactions
persistence contexts
temporary buffers
```

El reset del runtime será la última defensa, no la primera.

---

# 91. Cleanup must tolerate partial failure

Si una operación falla:

```text
query
flush
migration
request
```

el cleanup deberá intentar devolver el contexto a un estado seguro.

Ejemplo:

```text
Failure
 ↓
rollback
 ↓
close cursors
 ↓
release connection
 ↓
clear scoped state
```

---

# 92. Broken-state detection

Algunos fallos pueden dejar componentes inutilizables.

Ejemplo:

```text
lost connection
failed transaction
partial driver state
```

El sistema deberá poder marcar recursos como:

```text
broken
unusable
must reconnect
```

y no devolverlos al pool.

---

# 93. Connection reuse requires sanitization

Antes de reutilizar una conexión deberá verificarse o restaurarse estado relevante.

Ejemplos:

```text
open transaction
session variables
temporary settings
schema/search path
isolation state
driver error state
```

Especialmente con pooling y workers persistentes.

---

# 94. Pooling is an infrastructure concern

El ORM no deberá saber si la conexión provino de:

```text
new connection
persistent connection
connection pool
proxy
```

La abstracción Connection deberá ocultar esta decisión.

---

# 95. Read/write routing is intent-driven

El Query/Execution layer deberá comunicar intención:

```text
READ
WRITE
LOCKING_READ
SCHEMA
```

Infraestructura podrá resolver una conexión apropiada.

No se deberá determinar routing inspeccionando arbitrariamente SQL compilado cuando pueda conocerse antes.

---

# 96. Consistency over replica optimization

Después de una escritura puede ser necesario leer del primary.

Por tanto, funcionalidades como:

```text
sticky connection
```

deberán priorizar consistencia observable.

---

# 97. Distributed features must remain optional

Sharding, replicas, tenant routing y distribución no deberán infiltrarse en cada API básica.

Preferir:

```text
ConnectionResolver
```

como frontera.

La aplicación simple no deberá pagar toda la complejidad distribuida.

---

# 98. Multitenancy is integration, not core identity

Database podrá transportar contexto de tenant cuando un adaptador lo agregue.

Pero no deberá asumir que toda consulta posee tenant.

```text
Database Core
      ↑
Multitenancy Adapter
```

---

# 99. Integration independence

Si `Quantum/Telemetry` no está instalado:

```text
Database works
```

Si `Quantum/Cache` no está instalado:

```text
Database works
```

Si `Quantum/Multitenancy` no está instalado:

```text
Database works
```

Esto deberá cumplirse para toda integración declarada opcional.

---

# 100. Null implementations

Para puertos opcionales simples podrán utilizarse implementaciones Null.

Ejemplos:

```text
NullTelemetry
NullEventDispatcher
NullQueryCache
```

Siempre que esto simplifique el core y tenga coste mínimo.

---

# 101. Optional means truly optional

No deberá existir:

```text
optional package
```

que en realidad sea necesario para bootstrap.

Las dependencias opcionales deberán descubrirse y adaptarse sin romper Database.

---

# 102. Public API stability

Las APIs públicas deberán evolucionar cuidadosamente.

Clasificación:

```text
Public Stable
Public Experimental
Extension API
Internal
```

El estado deberá poder documentarse explícitamente.

---

# 103. Internal freedom

La arquitectura deberá preservar libertad para cambiar internals como:

```text
AST implementation
planner implementation
optimizer rule representation
hydration internals
```

sin romper aplicaciones.

Por ello no deberán exponerse accidentalmente.

---

# 104. Do not leak implementation types

Una API pública no deberá devolver un objeto interno si existe una abstracción apropiada.

Evitar:

```php
public function connection(): PdoConnection
```

si el contrato es:

```php
public function connection(): ConnectionInterface
```

---

# 105. Strong typing

Los contratos deberán usar tipos específicos.

Preferir:

```php
ConnectionName
IsolationLevel
QueryTimeout
DatabaseIdentifier
```

cuando aporten semántica real.

Evitar convertir todo en strings sin estructura.

---

# 106. Value Objects for semantic concepts

Podrán utilizarse Value Objects para:

```text
ConnectionName
DatabaseName
TableName
ColumnName
ParameterName
QueryTag
TransactionId
```

cuando reduzcan errores o ambigüedad.

---

# 107. Avoid primitive obsession selectively

No cada string requiere un Value Object.

Se utilizarán cuando exista:

```text
validation
identity
behavior
strong semantic distinction
security relevance
```

---

# 108. Enums for finite semantics

Conceptos finitos deberán considerar enums.

Ejemplos:

```php
enum QueryIntent
{
    case Read;
    case Write;
    case LockingRead;
}
```

```php
enum EntityState
{
    case New;
    case Managed;
    case Dirty;
    case Removed;
    case Detached;
}
```

---

# 109. Deterministic naming

Los nombres generados deberán seguir reglas deterministas.

Ejemplos:

```text
parameters
aliases
migration identifiers
generated constraints
compiled cache keys
```

Esto ayuda a:

```text
testing
debugging
cache reuse
reproducibility
```

---

# 110. Stable cache keys

Un cache key deberá derivarse de información estable.

Ejemplo para query compilation:

```text
Query Shape
Platform
Compiler Version
Relevant Capabilities
```

No deberá depender accidentalmente de:

```text
object hash
memory address
request-specific state
```

---

# 111. Cache correctness before hit rate

La invalidación incorrecta es peor que un cache miss.

Toda capa cacheada deberá definir:

```text
key
scope
lifetime
invalidation
versioning
```

antes de implementarse.

---

# 112. Cache is an optimization

El sistema deberá seguir funcionando sin caché siempre que la funcionalidad base lo permita.

La cache no deberá convertirse en fuente primaria de verdad para:

```text
transaction state
entity identity
connection state
```

---

# 113. Metadata cache and entity cache are different

No se mezclarán:

```text
Metadata Cache
Query Compilation Cache
Result Cache
Entity Cache
```

Cada uno tiene semánticas distintas de invalidez.

---

# 114. Schema changes invalidate relevant artifacts

Cuando exista un cambio estructural podrán requerirse invalidaciones de:

```text
schema metadata
compiled ORM metadata
query plans
generated migrations
```

Esta relación deberá modelarse explícitamente.

---

# 115. Migration safety over convenience

Las migraciones deberán priorizar integridad.

Operaciones potencialmente destructivas deberán:

```text
detectarse
clasificarse
advertirse
bloquearse según política
```

Ejemplos:

```text
DROP TABLE
DROP COLUMN
type narrowing
nullable → non-nullable
```

---

# 116. Migration execution is not arbitrary application execution

Una migración deberá utilizar APIs controladas.

Se deberá evitar que el sistema dependa para su operación fundamental de ejecutar lógica de aplicación arbitraria.

Las escape hatches seguirán existiendo donde sean necesarias.

---

# 117. Reproducible migrations

La misma migración ejecutada sobre un estado compatible deberá producir el mismo resultado lógico.

Evitar dependencias implícitas de:

```text
current application data
wall-clock behavior
external service
request context
```

salvo operación explícitamente diseñada.

---

# 118. Zero-downtime features are policies

VoltStack podrá proporcionar análisis para migraciones online.

Pero:

```text
zero downtime
```

no será una promesa automática universal, pues depende del motor, versión, volumen y operación.

El sistema deberá proporcionar estrategias y diagnósticos.

---

# 119. Testing is architectural, not an afterthought

Los componentes deberán diseñarse para probarse sin levantar toda la plataforma.

Ejemplos:

```text
Compiler
→ test with Plan + Platform

Optimizer
→ test with semantic query

UnitOfWork
→ test with entities + metadata

Hydrator
→ test with synthetic rows
```

---

# 120. Conformance testing

Drivers, dialectos y plataformas deberán disponer de suites de conformidad.

Una implementación que declare:

```text
supportsSavepoints = true
```

deberá demostrarlo mediante pruebas.

---

# 121. Architecture tests

Las reglas de dependencias deberán verificarse cuando sea posible mediante CI.

Ejemplos:

```text
Driver cannot import ORM
Compiler cannot import Connection implementation
Query AST cannot import Executor
Schema cannot import EntityManager
```

---

# 122. Property-based testing

Partes especialmente estructurales podrán beneficiarse de property-based testing.

Ejemplos:

```text
AST transformations preserve invariants
identifier quoting
type conversions
query normalization
schema diff
```

---

# 123. Differential testing

Cuando sea útil, compiladores podrán verificarse ejecutando operaciones equivalentes en motores distintos.

Ejemplo:

```text
Logical Query
  ├── PostgreSQL
  ├── MySQL
  └── SQLite

Expected logical result
```

---

# 124. Failure-path testing

Toda funcionalidad importante deberá probar:

```text
success path
failure path
cleanup path
retry path where applicable
```

Ejemplo:

```text
failed transaction
→ rollback
→ connection reusable?
→ UnitOfWork state?
```

---

# 125. Runtime leakage testing

La suite deberá incluir pruebas donde múltiples requests conceptuales usen el mismo worker.

Ejemplo:

```text
Request A
→ load User#1

RESET

Request B
→ must not contain User#1 in IdentityMap
```

---

# 126. Performance regression testing

Componentes críticos deberán tener benchmarks repetibles.

Ejemplos:

```text
query compilation
hydration
metadata loading
IdentityMap lookup
change tracking
bulk insert
connection resolution
```

---

# 127. Fast path and diagnostic path

Cuando sea necesario podrán existir dos niveles:

```text
Production Fast Path
Development Diagnostic Path
```

pero deberán preservar la misma semántica funcional.

---

# 128. Development diagnostics should teach

Los mensajes de desarrollo deberán explicar problemas arquitectónicos comunes.

Ejemplo:

```text
N+1 query detected:
User.posts was loaded 120 times.
Consider eager or batch loading.
```

Esto forma parte de Developer Experience.

---

# 129. Defaults for common applications

Una aplicación sencilla deberá requerir pocas decisiones.

Defaults iniciales podrán incluir:

```text
single default connection
safe prepared statements
lazy physical connection
standard transaction manager
standard query compiler
standard ORM metadata strategy
request-scoped persistence context
```

---

# 130. Advanced capabilities remain opt-in

Características como:

```text
sharding
distributed routing
second-level entity cache
custom optimizer policies
tenant connection resolver
```

no deberán activarse sin necesidad.

---

# 131. Progressive complexity

El programador podrá comenzar con:

```php
DB::table('users')->get();
```

y avanzar gradualmente hacia:

```text
Repository
EntityManager
Custom Types
Multiple Connections
Replicas
Sharding
Custom Dialects
```

sin cambiar de sistema de base de datos.

---

# 132. API consistency

Conceptos equivalentes deberán compartir convenciones.

Ejemplo:

```text
where()
orderBy()
limit()
```

no deberían usar nomenclaturas radicalmente distintas entre Query Builder y Entity Query sin razón.

---

# 133. Consistency does not mean identical APIs

ORM y Query Builder modelan abstracciones diferentes.

No deberán forzarse APIs idénticas si eso debilita sus modelos.

Por ejemplo:

```text
Entity relation
```

no es necesariamente equivalente a:

```text
SQL join
```

aunque puedan generar estructuras relacionadas.

---

# 134. Domain language over database language at ORM level

En ORM se favorecerán conceptos de dominio:

```text
entity
relationship
repository
identity
persistence
```

En Query:

```text
table
column
join
predicate
```

No deberán mezclarse innecesariamente.

---

# 135. Infrastructure language stays low-level

Conceptos como:

```text
socket
PDO
driver attribute
prepared handle
```

deberán permanecer en infraestructura.

No deberán aparecer en entidades.

---

# 136. No cross-layer convenience shortcuts

Una API de conveniencia deberá delegar correctamente.

Ejemplo:

```text
Model::save()
→ ActiveRecordAdapter
→ EntityManager
```

No:

```text
Model::save()
→ connection->execute()
```

aunque resulte más corto inicialmente.

---

# 137. Architectural debt must be explicit

Si por razones temporales se implementa una excepción arquitectónica deberá registrarse como deuda.

Debe documentar:

```text
qué regla viola
por qué
alcance
riesgo
plan de eliminación
```

No deberá normalizarse como patrón.

---

# 138. Experimental features need containment

Una característica experimental deberá ubicarse detrás de:

```text
experimental namespace
feature flag
extension package
unstable contract
```

cuando corresponda.

No deberá comprometer APIs estables prematuramente.

---

# 139. Backward compatibility is intentional

La compatibilidad no deberá impedir mejorar internals.

El sistema distinguirá:

```text
behavioral compatibility
API compatibility
extension compatibility
storage compatibility
migration compatibility
```

---

# 140. Deprecation before removal

Las APIs públicas estables deberán pasar por política de deprecación antes de eliminarse.

El sistema deberá poder proporcionar:

```text
deprecation message
replacement
migration guidance
target removal version
```

---

# 141. Database-specific behavior must be documented

Cuando una API tenga diferencias inevitables entre motores deberá documentarlas.

Ejemplo:

```text
ALTER TABLE behavior
locking behavior
JSON capabilities
RETURNING
transactional DDL
```

Portabilidad no deberá ocultar diferencias importantes.

---

# 142. Platform version matters

Las capacidades podrán variar por versión.

Conceptualmente:

```text
Platform
├── engine
├── version
└── capabilities
```

No deberá asumirse que:

```text
PostgreSQL
```

o:

```text
MySQL
```

representan una única plataforma eterna.

---

# 143. Feature detection over static assumptions

Cuando sea razonable, capacidades podrán derivarse de:

```text
driver
server version
configuration
runtime probe
```

y almacenarse en un descriptor estable.

---

# 144. Runtime probing must be bounded

No deberán ejecutarse consultas de detección repetidamente en cada query.

Preferir:

```text
connection initialization
      ↓
capability resolution
      ↓
cached platform descriptor
```

---

# 145. Deterministic compilation

Una misma entrada:

```text
ExecutionPlan
Platform
CompilationContext
```

deberá producir la misma salida compilada salvo información explícitamente variable.

Esto facilita:

```text
cache
testing
debugging
reproducibility
```

---

# 146. Compiler purity

El compilador deberá aproximarse a una función pura:

```text
Plan + Platform
      ↓
Compiled Query
```

No deberá:

```text
open connection
query schema at random
modify UnitOfWork
emit persistence events
```

---

# 147. Optimizer purity

Las reglas de optimización deberán evitar efectos secundarios.

Conceptualmente:

```text
Plan A
 ↓ rule
Plan B
```

Esto facilita composición y testing.

---

# 148. Planner owns execution strategy, not execution

El Planner decide:

```text
qué estrategia usar
```

El Executor realiza:

```text
la operación
```

Esta separación deberá mantenerse incluso cuando una estrategia requiera múltiples statements.

---

# 149. Multi-statement plans are explicit

Una operación lógica podrá producir:

```text
Statement 1
Statement 2
Statement 3
```

pero deberá representarse explícitamente mediante un Execution Plan.

No deberá esconderse dentro del compiler como side effect.

---

# 150. Atomicity requirements travel with plans

Si un plan multi-statement requiere atomicidad deberá declararlo.

Ejemplo conceptual:

```text
ExecutionPlan
├── statements
└── requiresTransaction: true
```

El Execution layer podrá entonces garantizar la política apropiada.

---

# 151. Query intent is metadata

Una consulta deberá poder declarar su intención.

Ejemplo:

```text
READ
WRITE
DDL
LOCKING_READ
MAINTENANCE
```

Esto podrá influir en:

```text
connection routing
retry policy
telemetry
security
```

sin analizar strings SQL arbitrariamente.

---

# 152. Contexts must be narrow

Cada contexto deberá transportar solo lo necesario.

Ejemplo:

```text
QueryContext
```

no deberá convertirse en un contenedor para:

```text
application
request
container
user
ORM
connection
everything
```

---

# 153. Context propagation is explicit

Cuando información de contexto deba viajar entre capas se hará mediante campos o contratos definidos.

Ejemplo:

```text
query tag
trace context
tenant routing hint
timeout
```

No mediante globals.

---

# 154. Cancellation and timeout semantics

Las operaciones de larga duración deberán considerar:

```text
timeout
cancellation
resource cleanup
transaction impact
```

Una cancelación nunca deberá dejar silenciosamente un contexto transaccional ambiguo.

---

# 155. Backpressure awareness

Streaming e importaciones deberán diseñarse evitando producir datos más rápido de lo que pueden procesarse.

Cuando el runtime lo permita, las APIs futuras podrán exponer mecanismos de backpressure sin alterar el modelo base.

---

# 156. Async readiness without fake async

La arquitectura podrá prepararse para drivers futuros con capacidades async.

Pero no deberá etiquetar como async una operación bloqueante envuelta superficialmente.

La abstracción se incorporará únicamente cuando existan semánticas reales.

---

# 157. No framework lock-in inside domain entities

Las entidades Data Mapper deberán poder permanecer mayormente independientes de VoltStack.

Preferir:

```php
final class User
{
}
```

sobre requerir obligatoriamente:

```php
class User extends VoltStackDatabaseEntity
{
}
```

Active Record podrá utilizar una clase base opcional.

---

# 158. Data Mapper is the architectural foundation

El modelo Data Mapper será el fundamento interno del ORM.

Esto proporciona:

```text
entity independence
UnitOfWork
IdentityMap
Repository
Persistence Planner
```

Active Record será una capa de conveniencia.

---

# 159. Active Record must remain optional

Una aplicación deberá poder utilizar exclusivamente:

```text
Entity
Repository
EntityManager
```

sin depender de:

```text
Model base class
static facade
ActiveRecord traits
```

---

# 160. Query Builder must remain usable without ORM

Del mismo modo:

```php
DB::table(...)
```

deberá funcionar sin inicializar:

```text
EntityManager
UnitOfWork
ORM Metadata
```

---

# 161. Schema must remain usable without Query Builder API

Schema podrá compartir infraestructura de compilación y ejecución, pero deberá mantener su propio modelo semántico.

```text
Schema AST
≠
Query AST
```

---

# 162. Shared infrastructure, separate domains

Los distintos dominios podrán compartir:

```text
Platform
Dialect primitives
Execution
Connection
Diagnostics
```

sin fusionar modelos.

---

# 163. Avoid circular knowledge through shared contracts

Cuando dos dominios necesiten un concepto compartido:

```text
A ↔ B
```

no deberán depender mutuamente.

Preferir:

```text
      Shared Contract
       ↑          ↑
       │          │
       A          B
```

---

# 164. Shared Kernel remains minimal

El Shared Kernel solo deberá contener conceptos verdaderamente transversales.

No deberá utilizarse para evitar decidir correctamente dónde pertenece una clase.

---

# 165. Naming communicates architecture

Los nombres deberán reflejar responsabilidades.

Ejemplos buenos:

```text
QueryCompiler
QueryExecutor
ConnectionResolver
PersistencePlanner
EntityHydrator
```

Nombres problemáticos:

```text
DatabaseHandler
DatabaseProcessor
DatabaseHelper
DatabaseService
```

cuando oculten responsabilidades múltiples.

---

# 166. Avoid "Manager" without coordination responsibility

`Manager` deberá utilizarse cuando realmente coordine múltiples componentes o recursos.

Ejemplos válidos:

```text
ConnectionManager
EntityManager
TransactionManager
```

No deberá ser el sufijo por defecto de servicios sin modelo propio.

---

# 167. Avoid generic Helpers

Las clases `Helper`, `Utils` o `Common` deberán evitarse.

Una función suficientemente importante deberá ubicarse en un componente semántico.

---

# 168. Facades are convenience APIs

Las Facades:

```text
DB
Schema
```

serán accesos de conveniencia al Container.

No deberán:

```text
mantener state
implementar business logic
administrar connection lifecycle
```

---

# 169. Static syntax does not imply static state

Una llamada como:

```php
DB::table('users')
```

podrá utilizar sintaxis estática de facade.

Internamente:

```text
Facade
 ↓
Scoped service resolution
```

No:

```text
static global Connection
```

---

# 170. Service Container is composition, not service locator everywhere

El Container deberá utilizarse principalmente para:

```text
bootstrap
composition
top-level resolution
factory construction
```

Los componentes internos deberán recibir dependencias explícitas.

Evitar:

```php
container()->get(...)
```

distribuido por Query Engine, ORM o Driver.

---

# 171. Constructor injection by default

Servicios con dependencias estables deberán utilizar preferentemente constructor injection.

Esto hace visibles:

```text
dependencies
test requirements
architecture
```

---

# 172. Runtime dependencies may use narrow factories/resolvers

Cuando una dependencia dependa del contexto:

```text
current connection
current transaction
current tenant
```

podrá utilizarse un resolver explícito.

Ejemplo:

```text
ConnectionResolverInterface
```

No un Container genérico.

---

# 173. Factories own construction complexity

Cuando crear un componente requiera múltiples decisiones se utilizarán factories.

Ejemplos:

```text
ConnectionFactory
EntityManagerFactory
HydratorFactory
PlatformFactory
```

El consumidor no deberá conocer todos los detalles.

---

# 174. Builders build; factories create; managers coordinate

Convención semántica:

```text
Builder
→ construye representación progresiva

Factory
→ crea objeto/configuración compleja

Manager
→ coordina recursos o servicios

Resolver
→ selecciona una implementación/valor

Registry
→ registra y localiza definiciones

Compiler
→ transforma representación a otra

Planner
→ define estrategia

Executor
→ ejecuta
```

Estos nombres deberán usarse consistentemente.

---

# 175. Validate early

Errores de configuración o metadata deberán detectarse tan pronto como sea razonable.

Ejemplo:

```text
Invalid mapping
```

preferiblemente durante:

```text
metadata compile
```

y no en la consulta número 10,000 en producción.

---

# 176. Execute late

Aunque se valide temprano, los efectos externos deberán realizarse lo más tarde posible.

Ejemplo:

```text
build query
validate
compile
```

no debe abrir conexión hasta ejecución cuando no sea necesaria antes.

---

# 177. Separate validation phases

Podrán existir diferentes validaciones:

```text
structural validation
semantic validation
platform validation
runtime validation
```

No deberán mezclarse indiscriminadamente.

---

# 178. Configuration is data

La configuración deberá representarse como objetos validados.

Preferir:

```text
DatabaseConfiguration
ConnectionConfiguration
PoolConfiguration
```

sobre acceder repetidamente a arrays arbitrarios en internals.

---

# 179. Configuration is immutable at runtime

Después del bootstrap, configuración utilizada por servicios persistentes deberá ser preferentemente inmutable.

Los cambios dinámicos deberán pasar por mecanismos explícitos de reconfiguración.

---

# 180. Secrets are not configuration diagnostics

Las credenciales podrán formar parte de configuración, pero deberán redactarse en:

```text
dump
debug
exception
telemetry
```

---

# 181. Versioned compiled artifacts

Los artefactos compilados deberán incluir información suficiente para detectar incompatibilidad.

Ejemplos:

```text
metadata version
compiler version
schema signature
platform signature
```

---

# 182. Rebuild rather than trust stale artifacts

Cuando un artefacto compilado sea incompatible deberá:

```text
recompilarse
```

o provocar error claro.

No deberá utilizarse silenciosamente.

---

# 183. Database state is external state

El sistema deberá asumir que una base de datos puede cambiar fuera del proceso VoltStack.

Por tanto, ciertas caches o metadata derivadas deberán disponer de políticas de refresco cuando corresponda.

---

# 184. Avoid unnecessary introspection in hot paths

La posibilidad anterior no significa introspeccionar schema en cada query.

Las políticas deberán balancear:

```text
correctness
freshness
performance
```

---

# 185. Explicit administrative operations

Operaciones como:

```text
schema inspection
health checks
maintenance
backup
diagnostics
```

deberán estar separadas de hot paths de consultas ordinarias.

---

# 186. Principle of bounded responsibility for Database

`Quantum/Database` no deberá asumir responsabilidades de:

```text
authentication
authorization policy
business validation
HTTP
UI
job scheduling
distributed tracing backend
tenant lifecycle management
```

Podrá integrarse con sus respectivos módulos.

---

# 187. Database does enforce its own invariants

La regla anterior no significa que Database ignore seguridad o consistencia propias.

Sí deberá controlar:

```text
bindings
transaction correctness
connection safety
mapping validity
resource lifecycle
state isolation
```

---

# 188. Framework integration through adapters

Integraciones deberán adoptar:

```text
Database Core
     ↑
Adapter
     ↑
External Quantum
```

cuando una dependencia inversa produciría acoplamiento.

---

# 189. No optional-package imports in core paths

Los namespaces centrales no deberán importar directamente clases de paquetes opcionales.

Por ejemplo:

```text
Database/Query
```

no deberá depender de:

```text
Quantum/Telemetry concrete implementation
```

---

# 190. Architecture should be enforceable

Una buena regla arquitectónica deberá poder convertirse, cuando sea posible, en:

```text
test
static analysis rule
namespace rule
code review checklist
```

No deberá depender únicamente de memoria del equipo.

---

# 191. Architectural invariants

Se consideran invariantes obligatorias:

```text
01. Query Builder nunca genera SQL directamente.
02. ORM nunca accede directamente al driver.
03. ORM nunca accede directamente a PDO.
04. Compiler nunca ejecuta consultas.
05. Executor nunca hidrata entidades.
06. Driver nunca conoce ORM.
07. Connection nunca posee estado de persistencia.
08. UnitOfWork nunca genera SQL.
09. Schema nunca depende del ORM.
10. Query Engine puede funcionar sin ORM.
11. ORM usa el Query Engine común.
12. Active Record usa el Persistence Engine común.
13. Estado request-scoped nunca sobrevive al scope.
14. Estado mutable siempre tiene propietario.
15. Diferencias de plataforma se resuelven por capabilities.
16. Integraciones opcionales no son dependencias obligatorias.
17. Raw SQL debe ser explícito.
18. Valores se parametrizan por defecto.
19. Compiladores deben ser deterministas.
20. Recursos deben limpiarse de forma segura.
```

---

# 192. Invariantes de runtime

Adicionalmente:

```text
21. IdentityMap no es global.
22. UnitOfWork no es global.
23. TransactionContext no es global.
24. TenantContext no es global.
25. Cursors no sobreviven accidentalmente al scope.
26. Transacciones huérfanas se revierten durante cleanup.
27. Connections rotas no regresan al pool.
28. Services persistentes no almacenan estado de request.
```

---

# 193. Invariantes de extensibilidad

```text
29. Extensions usan contratos públicos.
30. Registries son específicos por dominio.
31. Registries persistentes se congelan después de bootstrap.
32. Extensions no obtienen acceso universal a internals.
33. APIs internas no tienen garantía accidental de compatibilidad.
```

---

# 194. Invariantes de ORM

```text
34. Data Mapper es fundamento del ORM.
35. Active Record es opcional.
36. Entity no necesita conocer SQL.
37. EntityManager no compila SQL.
38. Hydrator no persiste.
39. Persistence Planner ordena operaciones.
40. Cascades son explícitos.
41. Flush posee pipeline determinista.
42. IdentityMap está limitada al PersistenceContext.
```

---

# 195. Invariantes de Query

```text
43. Query Model es independiente del driver.
44. AST es independiente de Connection.
45. Semantic Analysis precede optimización.
46. Optimizer preserva semántica.
47. Planner decide estrategia.
48. Compiler materializa SQL.
49. Executor realiza efectos externos.
50. Query intent se conoce estructuralmente cuando sea posible.
```

---

# 196. Invariantes de plataforma

```text
51. Dialect representa sintaxis.
52. Platform representa capacidades.
53. Driver representa transporte técnico.
54. Connection representa interacción lógica.
55. Ninguno sustituye semánticamente al otro.
```

---

# 197. Review checklist

Toda nueva característica deberá responder como mínimo:

```text
1. ¿Qué dominio la posee?
2. ¿Cuál es su responsabilidad?
3. ¿Qué entrada recibe?
4. ¿Qué salida genera?
5. ¿Mantiene estado?
6. ¿Quién posee ese estado?
7. ¿Cuál es su lifecycle?
8. ¿De qué componentes depende?
9. ¿Quién puede depender de ella?
10. ¿Es API pública, extensión o internal?
11. ¿Necesita una nueva abstracción?
12. ¿Puede crear dependencia circular?
13. ¿Funciona bajo runtime persistente?
14. ¿Cómo limpia recursos?
15. ¿Cómo falla?
16. ¿Cómo se prueba?
17. ¿Cómo se observa?
18. ¿Qué coste de rendimiento añade?
19. ¿Qué implicaciones de seguridad introduce?
20. ¿Puede implementarse como extensión en lugar de core?
```

---

# 198. Criterio para incorporar una abstracción

Antes de crear una abstracción deberá comprobarse:

```text
Existe una frontera real?
Existen múltiples implementaciones?
Existe una variación esperada?
Reduce acoplamiento significativo?
Hace explícita una responsabilidad?
Mejora testabilidad de forma real?
```

Si la respuesta general es no, probablemente la abstracción no sea necesaria.

---

# 199. Criterio para incorporar una dependencia

Toda nueva dependencia entre módulos deberá justificar:

```text
por qué el consumidor necesita conocer al proveedor
por qué un contrato más estrecho no es suficiente
por qué la dirección de dependencia es correcta
si puede crear un ciclo
si altera modularidad
```

---

# 200. Criterio para introducir estado mutable

Antes de añadir estado mutable deberá definirse:

```text
owner
scope
initialization
mutation rules
concurrency semantics
cleanup
failure behavior
```

Sin estos datos, el estado no deberá incorporarse.

---

# 201. Criterio para optimización

Toda optimización deberá responder:

```text
qué problema medido resuelve
qué hot path afecta
cuál es el coste adicional
qué complejidad agrega
cómo se invalida
cómo se desactiva
cómo se prueba
```

---

# 202. Criterio para feature core

Una funcionalidad pertenecerá al core si:

```text
es fundamental para la mayoría de usos
define una abstracción base
es necesaria para mantener invariantes
no puede implementarse limpiamente como integración
```

---

# 203. Criterio para feature opcional

Una funcionalidad deberá considerarse extensión o integración si:

```text
solo afecta casos avanzados
depende de otro Quantum
requiere infraestructura externa
añade coste significativo
introduce semántica específica de dominio
```

Ejemplos:

```text
Multitenancy integration
Telemetry exporters
Sharding strategies
Specialized cloud drivers
Second-level cache
```

---

# 204. Criterio para API pública

Una API deberá hacerse pública solo si:

```text
representa una necesidad estable del usuario
puede mantenerse en versiones futuras
tiene semántica clara
no expone internals innecesarios
```

---

# 205. Criterio para API interna

Una API deberá mantenerse interna cuando:

```text
exista para coordinar implementación
pueda cambiar por optimización
exponga detalles de pipeline
no sea necesaria para extensiones legítimas
```

---

# 206. Criterio para escape hatch

Un escape hatch será apropiado cuando:

```text
la abstracción no pueda expresar una operación válida
la plataforma tenga una característica específica
sea necesario acceder a SQL nativo
una optimización avanzada lo requiera
```

Deberá ser:

```text
explícito
documentado
observable
aislado
```

---

# 207. Anti-principio: abstraction for abstraction's sake

Se evitará crear sistemas genéricos anticipando problemas no demostrados.

Ejemplo:

```text
UniversalDatabaseExpressionTransformationCoordinator
```

no será preferible a servicios especializados únicamente por parecer más extensible.

---

# 208. Anti-principio: clever internals

Se evitará código arquitectónicamente inteligente pero difícil de seguir.

Preferir:

```text
explicit pipeline
```

sobre:

```text
runtime magic
implicit decorators
hidden event orchestration
unbounded reflection
```

cuando no aporten ventaja sustancial.

---

# 209. Anti-principio: framework magic as correctness mechanism

Las convenciones pueden mejorar DX.

No deberán ser el único mecanismo que garantiza:

```text
transaction safety
security
scope isolation
mapping correctness
```

Estos deberán basarse en invariantes explícitos.

---

# 210. Anti-principio: premature distribution

Una aplicación de una base de datos no deberá atravesar obligatoriamente:

```text
shard resolver
replica coordinator
distributed transaction layer
```

si no utiliza esas capacidades.

---

# 211. Anti-principio: premature caching

No todo debe cachearse.

Una cache se añadirá cuando:

```text
existe coste medible
la semántica de invalidez está definida
el beneficio justifica complejidad
```

---

# 212. Anti-principio: silent recovery

Database no deberá esconder errores de integridad mediante recuperación silenciosa.

Ejemplo:

```text
failed commit
```

no deberá convertirse en éxito porque un retry pareció funcionar sin poder garantizar semántica.

---

# 213. Anti-principio: mutable singletons

Los servicios process-wide no deberán convertirse en contenedores mutables de contexto.

Especialmente:

```text
DatabaseManager singleton
```

no significa:

```text
current connection state stored globally
```

---

# 214. Anti-principio: leaking PDO

PDO podrá utilizarse internamente por un driver.

No deberá convertirse en la abstracción de todo Database.

```text
PDO
```

es una implementación posible.

No es el dominio de VoltStack Database.

---

# 215. Anti-principio: driver conditionals everywhere

No:

```php
if ($driver === 'mysql') {
}

if ($driver === 'pgsql') {
}
```

repetido por todo el código.

Sí:

```text
Platform
Dialect
Capabilities
Strategy
```

---

# 216. Anti-principio: ORM owns everything

El ORM no deberá absorber:

```text
Schema
Migrations
Connection
Query Compiler
Telemetry
Cache
```

Podrá consumirlos o integrarse con ellos.

---

# 217. Anti-principio: Builder owns execution

Un Builder deberá representar intención.

Métodos terminales como:

```php
$query->get();
```

podrán delegar ejecución para DX.

Pero el objeto Builder no deberá implementar internamente toda la infraestructura de ejecución.

---

# 218. Anti-principio: migrations own schema

Migration depende de Schema.

No al contrario.

```text
Migration
 ↓
Schema
```

Esto permite utilizar Schema independientemente.

---

# 219. Anti-principio: telemetry as dependency backbone

Telemetry no deberá convertirse en el bus mediante el cual se coordinan componentes.

Su ausencia no debe cambiar semántica funcional.

---

# 220. Anti-principio: EventBus-driven database core

El flujo central no deberá ser:

```text
emit event
unknown listeners
event
unknown listeners
event
```

para cada transición interna.

Los eventos serán extensiones del flujo, no su estructura primaria.

---

# 221. Principle of readable architecture

Un desarrollador deberá poder seguir una operación leyendo aproximadamente:

```text
API
→ model
→ plan
→ compile
→ execute
```

sin depender de conocimiento oculto del Container o listeners.

---

# 222. Principle of local reasoning

Un componente deberá poder entenderse analizando principalmente:

```text
its contract
its dependencies
its local state
```

y no todo VoltStack.

Esto será un indicador de bajo acoplamiento.

---

# 223. Principle of composability

Los componentes deberán poder componerse.

Por ejemplo:

```text
QueryCompiler
+
Different Platform
```

o:

```text
QueryExecutor
+
Different ConnectionResolver
```

sin duplicar todo el sistema.

---

# 224. Principle of substitutable infrastructure

Un driver diferente no deberá alterar la semántica de alto nivel cuando ambas plataformas soporten la misma capacidad.

```text
PostgreSQL
MySQL
```

pueden producir SQL distinto, pero:

```text
Query Intent
```

deberá mantenerse.

---

# 225. Principle of semantic honesty

Cuando dos plataformas no puedan garantizar la misma semántica, VoltStack deberá reconocer la diferencia.

No deberá fingir portabilidad perfecta.

---

# 226. Principle of explicit unsupported behavior

Las capacidades no soportadas deberán producir errores específicos.

Ejemplo conceptual:

```text
UnsupportedDatabaseFeatureException
```

con información como:

```text
feature
platform
version
possible fallback
```

---

# 227. Principle of reproducibility

Las operaciones estructurales deberán ser reproducibles.

Especialmente:

```text
query compilation
metadata compilation
schema diff
migration planning
persistence planning
```

---

# 228. Principle of diagnosable pipelines

Cada pipeline deberá poder identificar la fase donde falló.

Ejemplo:

```text
Query Build
✓

Semantic Analysis
✓

Optimization
✓

Planning
✗ Unsupported locking strategy
```

Esto mejorará DX.

---

# 229. Principle of traceable transformation

En modo diagnóstico deberá ser posible entender transformaciones como:

```text
Builder
 ↓
AST
 ↓
Normalized AST
 ↓
Logical Plan
 ↓
Physical Plan
 ↓
SQL
```

sin necesariamente exponer todos estos detalles en producción.

---

# 230. Principle of bounded diagnostics

Los diagnósticos no deberán almacenar indefinidamente:

```text
queries
bindings
stack traces
entities
plans
```

especialmente en workers persistentes.

Deberán tener lifecycle y límites.

---

# 231. Principle of explicit cleanup

Todo scope deberá tener un cierre definido.

Ejemplo:

```text
PersistenceContext::close()
TransactionContext::close()
Cursor::close()
DatabaseContext::reset()
```

aunque determinadas APIs también proporcionen cleanup automático.

---

# 232. Principle of cleanup idempotency

Cuando sea razonable:

```text
close()
reset()
release()
```

deberán tolerar invocación repetida sin corromper estado.

Esto facilita manejo de fallos.

---

# 233. Principle of safe defaults under exceptions

Cuando una excepción interrumpe un flujo, el estado resultante deberá ser conservador.

Ejemplo:

```text
Unknown transaction result
→ mark connection unsafe
```

en lugar de asumir que puede reutilizarse.

---

# 234. Principle of transaction-local failure awareness

Si un statement invalida la transacción en una plataforma:

```text
TransactionContext
```

deberá reflejar dicho estado.

No deberá continuar como si pudiera ejecutarse normalmente.

---

# 235. Principle of resource governance

Database deberá poder imponer límites o políticas sobre:

```text
query timeout
pool size
max buffered rows
batch size
diagnostics
metadata cache
```

especialmente en producción.

---

# 236. Principle of application override

Los defaults deberán ser buenos, pero el desarrollador podrá modificar políticas cuando exista una necesidad válida.

La configuración avanzada deberá permanecer explícita.

---

# 237. Principle of secure introspection

Herramientas como:

```text
db:inspect
db:diagnose
debug toolbar
```

deberán aplicar redacción y permisos adecuados.

No deberán exponer credenciales o bindings sensibles por defecto.

---

# 238. Principle of minimum privilege

La documentación y tooling deberán favorecer conexiones con privilegios mínimos.

Podrán existir conexiones diferenciadas para:

```text
application runtime
migration administration
read-only analytics
```

sin exigirlo para aplicaciones simples.

---

# 239. Principle of migration privilege separation

Una aplicación en producción no necesariamente deberá utilizar las mismas credenciales para:

```text
SELECT/INSERT/UPDATE
```

y:

```text
ALTER/DROP/CREATE
```

La arquitectura deberá permitir esta separación.

---

# 240. Principle of ORM/domain separation

Las reglas de negocio deberán permanecer en dominio/aplicación.

Database podrá ayudar con:

```text
constraints
mapping
persistence
```

pero no sustituirá el modelo de negocio.

---

# 241. Principle of database constraints as integrity defense

El ORM no deberá asumir que validaciones de aplicación sustituyen restricciones de base de datos.

Se deberá permitir aprovechar:

```text
UNIQUE
FOREIGN KEY
CHECK
NOT NULL
```

cuando corresponda.

---

# 242. Principle of constraint error normalization

Los errores de constraints deberán poder convertirse en excepciones semánticas.

Ejemplos:

```text
UniqueConstraintViolationException
ForeignKeyConstraintViolationException
NotNullConstraintViolationException
CheckConstraintViolationException
```

cuando el driver pueda identificarlos.

---

# 243. Principle of accurate affected-row semantics

Las APIs deberán documentar diferencias en:

```text
matched rows
changed rows
affected rows
```

entre motores.

No deberán interpretar incorrectamente valores de driver.

---

# 244. Principle of identifier-generation abstraction

La generación de IDs deberá soportar estrategias como:

```text
application-generated UUID
application-generated ULID
auto increment
sequence
database generated identity
custom generator
```

sin acoplar el ORM a una sola plataforma.

---

# 245. Principle of application-generated identifiers where useful

VoltStack deberá soportar eficientemente IDs generados antes de persistir.

Esto facilita:

```text
distributed systems
batch persistence
domain identity
testing
```

sin imponerlos como único modelo.

---

# 246. Principle of precise type conversion

El sistema de tipos deberá distinguir entre:

```text
database type
PHP storage type
domain type
```

Ejemplo:

```text
VARCHAR
 ↓
string
 ↓
Email ValueObject
```

Cada conversión deberá ser explícita.

---

# 247. Principle of loss awareness

Conversiones potencialmente con pérdida deberán detectarse o documentarse.

Ejemplos:

```text
decimal → float
timezone conversion
bigint → int
JSON numeric conversion
```

---

# 248. Principle of decimal correctness

Valores monetarios o de precisión arbitraria no deberán convertirse automáticamente a `float` si eso pierde precisión.

El Type System deberá permitir representaciones seguras.

---

# 249. Principle of timezone explicitness

Los tipos temporales deberán definir:

```text
timezone handling
database timezone
application timezone
immutable/mutable representation
```

No deberán depender de defaults ambiguos.

---

# 250. Principle of database portability at type layer

Un tipo lógico podrá mapearse a tipos físicos distintos.

Ejemplo:

```text
Logical Boolean
├── BOOLEAN
├── TINYINT
└── INTEGER
```

según plataforma.

---

# 251. Principle of query-shape reuse

El motor deberá separar:

```text
query structure
parameter values
```

para permitir:

```text
compiled query reuse
prepared statement reuse
cache effectiveness
```

---

# 252. Principle of binding metadata

Cada binding podrá transportar:

```text
value
logical type
database type hint
sensitivity classification
```

cuando sea necesario.

Esto ayuda a:

```text
drivers
telemetry redaction
type conversion
```

---

# 253. Principle of no secret values in cache keys

Los valores sensibles no deberán formar parte directa de keys de cache o diagnostics persistentes.

---

# 254. Principle of query tagging

VoltStack podrá permitir tags estructurados para:

```text
diagnostics
telemetry
operation origin
performance analysis
```

Ejemplo:

```php
$query->tag('billing.invoice-list');
```

Los tags no deberán modificar semántica SQL salvo mecanismos expresamente definidos.

---

# 255. Principle of transparent but inspectable generated SQL

El usuario no necesita escribir SQL para usar Query Builder.

Sin embargo deberá poder inspeccionar:

```text
compiled SQL
bindings metadata
platform
execution plan summary
```

en tooling de desarrollo.

---

# 256. Principle of no misleading SQL preview

Una API `toSql()` deberá dejar claro si muestra:

```text
template SQL
compiled SQL
interpolated debug representation
```

Los bindings no deberán interpolarse de forma insegura y presentarse como SQL real ejecutado.

---

# 257. Principle of immutable compiled query

Una vez compilada:

```text
CompiledQuery
```

no deberá modificarse durante ejecución.

Los bindings concretos podrán formar parte de una instancia de ejecución claramente separada si el diseño lo requiere.

---

# 258. Principle of query execution isolation

Dos ejecuciones concurrentes del mismo query shape no deberán compartir estado mutable de ejecución.

---

# 259. Principle of deterministic alias generation

Los aliases generados automáticamente deberán ser deterministas dentro del contexto apropiado.

Esto ayuda a caching y diagnostics.

---

# 260. Principle of parser avoidance where structured data exists

Si VoltStack ya posee AST no deberá recompilar SQL y después volver a parsearlo para obtener información que ya existía.

Ejemplo:

```text
Query Intent
```

deberá conservarse como metadata.

---

# 261. Principle of SQL parsing only at raw boundaries

Si en el futuro se requiere analizar Raw SQL, deberá considerarse una capacidad independiente y limitada.

No deberá convertirse en dependencia del flujo estructurado.

---

# 262. Principle of migration plan inspection

Antes de ejecutar migraciones podrá ser posible inspeccionar:

```text
planned operations
generated SQL
risk classification
transaction strategy
```

Esto mejorará seguridad operacional.

---

# 263. Principle of dry-run capability

Operaciones administrativas adecuadas podrán proporcionar:

```text
dry run
```

sin realizar efectos externos.

Ejemplos:

```text
migration
schema diff
maintenance planning
```

---

# 264. Principle of plan/execute separation

Siempre que una operación compleja pueda beneficiarse:

```text
Plan
 ↓
Review
 ↓
Execute
```

deberá evitar combinar planificación y ejecución de forma irreversible.

---

# 265. Principle of operational observability

Operaciones administrativas deberán producir información suficiente para responder:

```text
what happened?
how long?
which database?
which migration?
which operation failed?
```

---

# 266. Principle of production predictability

La configuración productiva deberá evitar comportamientos dinámicos inesperados como:

```text
automatic schema mutation
runtime metadata discovery every request
automatic migration
silent connection switching
```

salvo configuración explícita.

---

# 267. Principle of no automatic migrations on request

Las migraciones no deberán ejecutarse automáticamente como parte del manejo normal de requests HTTP.

---

# 268. Principle of deterministic bootstrap

El mismo conjunto de configuración y paquetes deberá producir el mismo grafo de servicios Database.

No deberá depender del orden accidental de primera consulta.

---

# 269. Principle of compile-time validation where possible

Errores detectables durante:

```text
container compilation
metadata compilation
configuration validation
```

deberán detectarse allí y no aplazarse al primer tráfico.

---

# 270. Principle of runtime validation where necessary

Las condiciones externas dinámicas como:

```text
database unavailable
server version changed
permission denied
```

seguirán requiriendo validación runtime.

---

# 271. Principle of bounded fallback behavior

Los fallbacks deberán estar definidos y ser finitos.

Nunca deberá existir una cadena impredecible:

```text
strategy A fails
→ silently B
→ silently C
→ silently D
```

sin observabilidad.

---

# 272. Principle of stable semantics across modes

`development` y `production` podrán diferir en:

```text
diagnostics
cache
reflection
assertions
```

pero no en la semántica esencial de consultas y persistencia.

---

# 273. Principle of explicit unsafe APIs

Cuando una API permita romper garantías deberá identificarse.

Ejemplos conceptuales:

```text
unsafeRaw()
disableConstraintChecks()
withoutTransactionSafety()
```

Su uso deberá ser excepcional.

---

# 274. Principle of scoped unsafe operations

Una operación unsafe deberá limitarse al scope más pequeño posible.

Evitar configuraciones globales permanentes para desactivar protecciones.

---

# 275. Principle of restoration after temporary configuration

Si Database modifica temporalmente estado de conexión:

```text
constraint checks
isolation
session variables
search path
```

deberá restaurarlo antes de reutilizarla cuando corresponda.

---

# 276. Principle of explicit connection poisoning

Cuando no pueda garantizarse restauración segura:

```text
connection
→ poisoned
→ discard
```

No deberá regresar al pool.

---

# 277. Principle of no cross-request ORM proxies

Los proxies o lazy loaders ligados a un PersistenceContext no deberán utilizar contextos de requests anteriores.

Su comportamiento después de cerrar contexto deberá ser explícito.

---

# 278. Principle of detached entity clarity

Una entidad detached deberá tener semántica clara.

No deberá comenzar a persistirse nuevamente por accidente al entrar en otro contexto.

---

# 279. Principle of merge behavior caution

Si se implementa `merge`, su semántica deberá estar rigurosamente definida.

No se asumirá como operación básica debido a la complejidad de reconciliar grafos detached.

---

# 280. Principle of explicit aggregate loading

Para dominios complejos, Repository deberá poder controlar qué parte del agregado se carga.

No deberá depender exclusivamente de lazy loading accidental.

---

# 281. Principle of relations as metadata, not hidden queries

Una relación define estructura.

La decisión de cargarla pertenece a una estrategia específica.

```text
Relationship Metadata
      ↓
Load Strategy
      ↓
Query Plan
```

---

# 282. Principle of immutable collection semantics where appropriate

Las colecciones ORM deberán definir claramente:

```text
loaded state
dirty state
ownership
ordering
duplicate semantics
```

No deberán ser arrays mágicos sin modelo.

---

# 283. Principle of relation change tracking separation

Los cambios de relaciones deberán poder registrarse separadamente de cambios escalares.

Ejemplo:

```text
EntityChanges
RelationChanges
```

Esto simplifica Persistence Planning.

---

# 284. Principle of orphan removal explicitness

`orphanRemoval` deberá ser una política explícita y claramente distinta de:

```text
cascade remove
```

---

# 285. Principle of ordering semantics

Cuando una relación o query declare orden:

```text
ORDER BY
```

la semántica deberá conservarse durante batching, eager loading o hydration.

---

# 286. Principle of pagination determinism

La paginación deberá recomendar o exigir orden determinista cuando sea necesario.

Especialmente:

```text
cursor pagination
```

---

# 287. Principle of cursor integrity

Los cursores deberán construirse con campos suficientes para continuar de forma estable.

No deberán depender únicamente de offsets cuando se promete semántica cursor-based.

---

# 288. Principle of query snapshot semantics awareness

La paginación sobre datos concurrentemente modificados puede variar según aislamiento.

VoltStack deberá documentar esta realidad en lugar de prometer resultados imposibles.

---

# 289. Principle of explicit locking

Las operaciones de locking deberán ser visibles.

Ejemplo:

```php
$query->forUpdate();
```

No deberán activarse implícitamente por cargar cierta entidad.

---

# 290. Principle of lock capability validation

El Planner deberá validar:

```text
lock type
platform support
transaction requirements
```

antes de compilar cuando sea posible.

---

# 291. Principle of lock scope clarity

APIs de locking deberán definir si bloquean:

```text
rows
tables
advisory key
```

y qué plataformas soportan cada semántica.

---

# 292. Principle of optimistic locking at ORM layer

Version fields y optimistic locking pertenecen al ORM/Persistence layer.

El Query Engine proporcionará las expresiones necesarias, pero no conocerá entidades.

---

# 293. Principle of conflict visibility

Un conflicto optimistic lock deberá producir una excepción específica.

No deberá convertirse en:

```text
0 rows affected
```

sin semántica ORM.

---

# 294. Principle of DDL transaction awareness

Schema/Migrations deberán conocer si una plataforma soporta:

```text
transactional DDL
```

y planificar en consecuencia.

---

# 295. Principle of operation classification

Schema operations deberán clasificarse por propiedades como:

```text
transactional
destructive
blocking
reversible
online-capable
```

cuando sea posible.

---

# 296. Principle of reversibility honesty

Una migration `down()` no deberá considerarse necesariamente perfectamente reversible.

La documentación y tooling deberán distinguir:

```text
syntactically reversible
data-preserving reversible
irreversible
```

---

# 297. Principle of backup before destructive operation guidance

Database tooling podrá recomendar backup o snapshot para operaciones clasificadas como destructivas.

La realización automática dependerá de capacidades operativas externas.

---

# 298. Principle of administration separation

El sistema administrativo deberá ser una capa distinta del request-time Database API.

Esto evita cargar capacidades de:

```text
backup
restore
maintenance
diagnostics
```

en cada petición.

---

# 299. Principle of package modularity

Aunque inicialmente `voltstack/database` pueda distribuir varios subsistemas juntos, sus dependencias internas deberán permitir futura separación física si fuese necesaria.

Arquitectura lógica primero; empaquetado después.

---

# 300. Principle of no premature package fragmentation

La posibilidad anterior no significa dividir desde el inicio cada namespace en un paquete Composer.

La separación física deberá justificarse por:

```text
independent lifecycle
optional dependency
reuse
deployment benefit
maintenance boundary
```

---

# 301. Principle of architectural documentation as contract

Los documentos Database deberán describir decisiones suficientemente concretas para evaluar código.

No serán documentación decorativa.

Cada implementación deberá poder compararse contra:

```text
architecture
design principles
contracts
invariants
```

---

# 302. Principle of ADR for major deviations

Las decisiones que modifiquen una regla fundamental deberán registrar un ADR o documento equivalente.

Especialmente:

```text
new persistence model
new dependency direction
new shared state
new compilation boundary
new runtime model
```

---

# 303. Principle of compatibility tests for architectural migrations

Si en el futuro se sustituye una implementación importante:

```text
PDO Driver
Compiler
Metadata Engine
```

las suites de conformidad deberán demostrar que mantiene contratos.

---

# 304. Principle of evolutionary architecture

VoltStack Database deberá poder evolucionar mediante:

```text
replaceable implementations
stable contracts
capability negotiation
extension points
architecture tests
```

sin necesitar reescrituras completas para cada nueva característica.

---

# 305. Principle of avoiding speculative generality

No se diseñarán extensiones para motores, runtimes o paradigmas hipotéticos si no existe un requisito claro.

La arquitectura sí deberá evitar decisiones que impidan razonablemente futuras extensiones.

---

# 306. Principle of data integrity over framework convenience

Cuando exista conflicto entre una API más cómoda y garantías de integridad, deberá favorecerse la segunda.

La comodidad podrá recuperarse mediante una API segura adicional.

---

# 307. Principle of explicit consistency models

Capacidades distribuidas deberán documentar consistencia.

Ejemplos:

```text
primary read-after-write
replica eventual consistency
sticky read consistency
```

No deberán ocultarse detrás de una única palabra `read`.

---

# 308. Principle of no distributed transaction illusion

VoltStack no deberá presentar como ACID global operaciones que no lo sean.

Si en el futuro existen coordinadores distribuidos, sus garantías deberán especificarse.

---

# 309. Principle of transactional side-effect awareness

Database transactions no pueden revertir automáticamente:

```text
HTTP calls
emails
external queues
filesystem operations
```

La integración con patrones como outbox podrá existir, pero Database no deberá fingir atomicidad inexistente.

---

# 310. Principle of extensible event/outbox integration

El sistema podrá proporcionar puntos para integrar:

```text
transactional outbox
domain events after commit
```

sin incorporar lógica de mensajería dentro del UnitOfWork base.

---

# 311. Principle of after-commit correctness

Eventos que se declaren `after commit` no deberán emitirse antes de confirmar el commit.

Si el commit falla:

```text
after-commit event
→ must not be emitted as successful
```

---

# 312. Principle of lifecycle event ordering

Cuando existan eventos ORM deberán tener orden documentado.

Ejemplo conceptual:

```text
prePersist
preFlush
onFlush
SQL execution
postPersist
postFlush
afterCommit
```

La semántica concreta será definida posteriormente.

---

# 313. Principle of no arbitrary entity mutation by low-level events

Eventos de Connection o Query no deberán recibir entidades para modificarlas.

Cada evento deberá respetar su capa.

---

# 314. Principle of event payload minimization

Los eventos deberán transportar la información necesaria.

No todo el Container, DatabaseManager o RuntimeContext.

---

# 315. Principle of privacy-aware diagnostics

Bindings y resultados deberán poder clasificarse como sensibles.

Telemetry deberá poder:

```text
redact
hash
omit
```

según política.

---

# 316. Principle of observability correlation

Query, transaction y request podrán transportar IDs de correlación.

Esto deberá implementarse como metadata de contexto, no global.

---

# 317. Principle of diagnostics without semantics changes

Activar profiler no deberá modificar:

```text
query result
transaction boundaries
flush ordering
connection routing
```

salvo overhead observable.

---

# 318. Principle of benchmark realism

Los benchmarks deberán incluir escenarios:

```text
cold start
warm cache
single query
many queries
hydration
large result
transaction
worker reuse
```

No únicamente microbenchmarks favorables.

---

# 319. Principle of performance budgets

Componentes críticos podrán definir presupuestos o metas de overhead.

Ejemplos:

```text
telemetry disabled overhead
metadata lookup
IdentityMap lookup
query builder construction
```

Las cifras concretas serán definidas en documentación de performance.

---

# 320. Principle of profiling before architectural optimization

Antes de introducir mecanismos como:

```text
object pooling
custom memory arenas
complex plan caches
```

deberá existir evidencia de que resuelven un cuello de botella real.

---

# 321. Principle of PHP runtime awareness

La arquitectura deberá aprovechar características modernas de PHP sin depender de comportamientos frágiles.

Se favorecerán:

```text
readonly objects where useful
enums
attributes
typed properties
strict types
```

según compatibilidad definida por VoltStack.

---

# 322. Principle of no serialization assumptions

No deberá asumirse que todo objeto interno puede serializarse automáticamente.

Cuando un artefacto necesite persistirse deberá tener una representación explícitamente serializable.

---

# 323. Principle of cacheable representation

Los objetos destinados a cache persistente deberán evitar referencias a:

```text
closures
resources
connections
runtime objects
```

---

# 324. Principle of explicit compiled formats

Metadata o planes compilados podrán utilizar:

```text
PHP arrays
generated PHP
binary representation
other format
```

pero el formato deberá estar versionado y separado del modelo conceptual.

---

# 325. Principle of implementation replaceability

El formato de cache no deberá convertirse en la API pública del sistema.

Debe poder sustituirse sin modificar aplicación.

---

# 326. Principle of deterministic schema diff

Dado:

```text
Schema A
Schema B
Platform
```

el Schema Diff deberá generar un conjunto lógico determinista de cambios.

---

# 327. Principle of schema diff conservatism

Cuando no pueda determinarse con seguridad si:

```text
rename
```

o:

```text
drop + add
```

el sistema deberá evitar inferencias destructivas silenciosas.

Podrá requerir una pista explícita.

---

# 328. Principle of data migration separation

Los cambios de estructura y transformaciones complejas de datos deberán poder separarse conceptualmente.

Ejemplo:

```text
Schema Migration
Data Migration
```

aunque puedan coordinarse.

---

# 329. Principle of operational idempotency where possible

Comandos administrativos deberán ser idempotentes cuando su semántica lo permita.

Ejemplo:

```text
db:status
db:inspect
metadata:compile
```

---

# 330. Principle of migration idempotency caution

Las migraciones versionadas no necesitan ejecutarse repetidamente.

Su sistema deberá registrar claramente:

```text
pending
running
completed
failed
```

y evitar duplicación accidental.

---

# 331. Principle of migration concurrency control

Dos procesos no deberán poder ejecutar de forma insegura la misma migración simultáneamente.

El sistema deberá proporcionar coordinación o locking apropiado.

---

# 332. Principle of crash recovery awareness

Si un proceso muere durante:

```text
migration
flush
transaction
bulk operation
```

el sistema deberá poder diagnosticar el estado restante hasta donde las capacidades del motor lo permitan.

---

# 333. Principle of transaction state as source of truth

El Transaction Manager no deberá asumir commit/rollback únicamente porque se llamó a una API.

Deberá actualizar su estado según resultado real del driver.

---

# 334. Principle of unknown transaction outcome

Algunos fallos de red pueden producir resultado desconocido.

VoltStack deberá poder representar:

```text
transaction outcome unknown
```

en vez de asumir automáticamente rollback.

---

# 335. Principle of conservative recovery from unknown outcome

Cuando el resultado sea desconocido, las políticas automáticas deberán evitar duplicar efectos potencialmente ya comprometidos.

---

# 336. Principle of retry classification

Los errores podrán clasificarse como:

```text
retryable
non-retryable
unknown
```

pero la posibilidad de retry también dependerá de la operación.

---

# 337. Principle of database-independent exceptions where possible

El usuario deberá poder capturar:

```php
UniqueConstraintViolationException
```

sin conocer el código específico de PostgreSQL o MySQL.

Los detalles nativos permanecerán accesibles cuando sean necesarios.

---

# 338. Principle of native detail preservation

Normalizar excepciones no deberá eliminar información útil como:

```text
SQLSTATE
native error code
constraint name
server message
```

siempre que sea seguro exponerla al nivel apropiado.

---

# 339. Principle of redaction layers

La información de error podrá tener niveles:

```text
Internal diagnostic
Developer diagnostic
Production public message
```

Cada uno con diferente nivel de detalle.

---

# 340. Principle of clear boundary between user input and identifiers

Las APIs no deberán permitir convertir input arbitrario del usuario en:

```text
column
table
direction
function
```

sin validación explícita.

---

# 341. Principle of allowlists for dynamic structural input

Cuando una aplicación necesite orden dinámico:

```php
$orderBy = $request->input('sort');
```

la documentación deberá recomendar mappings/allowlists.

Database podrá ofrecer APIs que faciliten esta práctica.

---

# 342. Principle of safe default ordering direction

Las direcciones deberán modelarse mediante valores finitos:

```text
ASC
DESC
```

no concatenar strings libres.

---

# 343. Principle of query functions registry

Las funciones SQL soportadas por Query AST deberán registrarse semánticamente.

Esto permitirá:

```text
validation
platform compilation
return type inference
```

---

# 344. Principle of native functions escape hatch

Funciones específicas de plataforma podrán utilizarse mediante extensiones o expresiones nativas explícitas.

---

# 345. Principle of semantic function mapping

Una función lógica podrá compilarse diferente según plataforma.

Ejemplo conceptual:

```text
StringLength
├── LENGTH(...)
└── alternative dialect form
```

cuando la semántica sea equivalente.

---

# 346. Principle of no false function equivalence

Si dos funciones parecen equivalentes pero tienen semánticas distintas, no deberán mapearse automáticamente sin documentarlo.

---

# 347. Principle of type-aware expressions

El Semantic Analyzer deberá poder conocer tipos cuando sean relevantes para:

```text
operator validity
parameter conversion
result hydration
function resolution
```

---

# 348. Principle of predictable null semantics

Query APIs deberán reflejar correctamente semántica SQL de `NULL`.

Ejemplo:

```php
->whereNull('deleted_at')
```

en lugar de tratar siempre:

```php
->where('deleted_at', '=', null)
```

como igualdad ordinaria.

La API podrá normalizarlo, pero la semántica deberá ser correcta.

---

# 349. Principle of three-valued logic awareness

Optimizer y simplificador deberán respetar lógica SQL con:

```text
TRUE
FALSE
UNKNOWN
```

No aplicar simplificaciones booleanas incorrectas de lenguajes de dos valores.

---

# 350. Principle of collation awareness

Comparaciones de texto pueden depender de:

```text
collation
case sensitivity
locale
```

VoltStack no deberá prometer equivalencia universal entre motores.

---

# 351. Principle of transactional ORM synchronization

El estado del UnitOfWork deberá sincronizarse únicamente cuando exista suficiente certeza sobre la persistencia.

Ejemplo:

```text
SQL successful
but transaction later rolled back
```

no equivale a estado permanentemente persistido.

---

# 352. Principle of post-rollback reconciliation

Tras rollback, el PersistenceContext deberá tener reglas claras para:

```text
generated IDs
entity states
snapshots
collections
```

Estas reglas se definirán en la documentación de UnitOfWork/Transactions.

---

# 353. Principle of no irreversible in-memory assumption before commit

Cuando una operación dependa del commit, el ORM no deberá asumir permanentemente que terminó antes de confirmarlo.

---

# 354. Principle of explicit generated-value synchronization

Valores generados por database deberán recuperarse mediante una estrategia explícita.

Ejemplos:

```text
RETURNING
last insert id
follow-up SELECT
sequence
```

según plataforma.

---

# 355. Principle of persistence plan capability awareness

El Persistence Planner podrá pedir:

```text
generated value requirement
batch capability
returning capability
transaction requirement
```

sin depender de dialecto concreto.

---

# 356. Principle of statement ordering visibility

En modo diagnóstico deberá poder visualizarse el orden lógico de operaciones de `flush()`.

Esto facilitará entender:

```text
constraint failures
cycles
unexpected cascades
```

---

# 357. Principle of cycle detection in persistence graphs

El Persistence Planner deberá detectar dependencias circulares relevantes.

Podrá resolverlas mediante:

```text
nullable FK phases
deferred constraints
multiple operations
```

cuando sea válido.

De lo contrario deberá informar claramente.

---

# 358. Principle of no silent cascade explosion

Una operación sobre una entidad que termine afectando miles de entidades por cascades deberá ser observable.

Podrán existir límites o advertencias configurables.

---

# 359. Principle of bulk operations semantic distinction

Bulk operations podrán saltarse determinadas capacidades ORM.

Ejemplo:

```text
bulk UPDATE
```

puede no hidratar cada entidad ni ejecutar lifecycle por entidad.

La API deberá documentar claramente estas diferencias.

---

# 360. Principle of no fake per-entity lifecycle on bulk SQL

Si un UPDATE masivo se ejecuta en un statement:

```text
UPDATE users SET ...
```

VoltStack no deberá fingir que ejecutó callbacks individualmente salvo que realmente cargue/procese las entidades.

---

# 361. Principle of IdentityMap invalidation after bulk operations

Las operaciones bulk que afectan entidades existentes en el PersistenceContext deberán definir:

```text
invalidate
refresh
clear
reject operation
```

según política.

No deberán dejar estado silenciosamente inconsistente.

---

# 362. Principle of external mutation awareness

Si otras aplicaciones modifican la base de datos, el IdentityMap puede quedar stale.

La arquitectura deberá documentar que un PersistenceContext representa una vista local temporal, no un cache global coherente.

---

# 363. Principle of short-lived persistence contexts by default

En aplicaciones web, el PersistenceContext deberá normalmente vivir:

```text
one request
```

No durante toda la vida del worker.

---

# 364. Principle of explicit persistence context rotation

En workers o importaciones de larga duración deberá ser posible:

```text
flush
clear
rotate context
```

para controlar memoria y frescura.

---

# 365. Principle of lifecycle-independent domain entities

Cerrar un PersistenceContext no deberá destruir las entidades como objetos PHP.

Pero operaciones que requieran contexto, como lazy loading, deberán reaccionar explícitamente.

---

# 366. Principle of deterministic detached behavior

Una entidad detached deberá seguir siendo utilizable como objeto de dominio.

No deberá ejecutar automáticamente queries mediante referencias obsoletas.

---

# 367. Principle of configurable lazy loading strictness

Podrán existir modos:

```text
allow lazy loading
warn on lazy loading
forbid lazy loading
```

especialmente útiles para evitar N+1.

---

# 368. Principle of query count visibility

En desarrollo deberá ser posible conocer:

```text
queries per request
queries per component
queries per transaction
```

mediante integración con Telemetry.

---

# 369. Principle of no profiler memory leak

El Query Profiler deberá tener límites por scope.

No deberá retener indefinidamente objetos, resultados o stack traces.

---

# 370. Principle of development tooling isolation

Debug Toolbar y Profiler deberán consumir contratos de diagnóstico.

No deberán invadir internals del Query Engine.

---

# 371. Principle of clear performance trade-offs

APIs con alto coste potencial deberán documentarlo.

Ejemplos:

```text
fetch all
eager load huge graph
schema introspection
full IdentityMap
```

---

# 372. Principle of sane batch defaults

Cuando el framework seleccione tamaños de batch predeterminados deberán ser conservadores y configurables.

No deberá asumir que todas las bases soportan:

```text
65,000 parameters
```

o cantidades arbitrarias.

---

# 373. Principle of parameter-limit awareness

Platform deberá poder exponer límites como:

```text
max parameters
max identifier length
max batch size
```

cuando sean relevantes.

Planner podrá adaptar estrategias.

---

# 374. Principle of identifier-length handling

Los nombres generados de constraints e indexes deberán respetar límites de plataforma.

La truncación deberá ser:

```text
deterministic
collision-resistant
diagnosable
```

---

# 375. Principle of deterministic generated names

Ejemplo:

```text
users_email_unique
```

si debe truncarse, deberá conservar una firma estable.

---

# 376. Principle of no hidden schema naming collisions

Schema Builder deberá detectar cuando nombres generados colisionen.

No deberá depender únicamente del error del servidor si puede detectarlo antes.

---

# 377. Principle of schema normalization

Schema Metadata deberá normalizar diferencias de introspección sin borrar información específica importante.

Podrá mantener:

```text
logical representation
native metadata
```

separadamente.

---

# 378. Principle of native metadata preservation

Información específica como:

```text
collation
storage engine
partial index predicate
generated expression
```

podrá conservarse en extensiones de metadata.

---

# 379. Principle of portable diff with native extensions

Schema Diff deberá comparar:

```text
portable core properties
+
platform-native properties
```

cuando ambos esquemas sean de la misma plataforma.

---

# 380. Principle of migration target platform explicitness

Una migration podrá ser portable o específica de plataforma.

La distinción deberá ser visible.

---

# 381. Principle of migration planning against capabilities

El Migration Planner deberá decidir estrategias según:

```text
platform
version
capabilities
operation properties
```

no mediante condicionales dispersos.

---

# 382. Principle of minimal locking migrations

Cuando existan alternativas equivalentes, el Planner podrá favorecer estrategias con menor bloqueo.

Esto deberá ser política configurable y observable.

---

# 383. Principle of no unbounded automatic data rewrites

Una migration que requiera modificar millones de filas no deberá convertir automáticamente un simple `alter` en una operación gigantesca sin advertencia.

---

# 384. Principle of explicit online migration phases

Capacidades avanzadas podrán representar:

```text
prepare
backfill
switch
cleanup
```

como fases separadas.

---

# 385. Principle of operational checkpointing

Operaciones largas como:

```text
backfill
import
export
```

podrán soportar checkpoints cuando sea apropiado.

Esto pertenecerá a sistemas específicos, no al Query Builder básico.

---

# 386. Principle of bounded retries

Toda política de retry deberá tener límites.

```text
max attempts
time budget
backoff
```

No existirán retries infinitos.

---

# 387. Principle of jitter for distributed retries

Cuando corresponda, políticas de retry distribuidas podrán utilizar jitter para evitar thundering herd.

Esto será responsabilidad de Resilience, no Query.

---

# 388. Principle of circuit breaker placement

Circuit Breaker deberá envolver recursos externos en infraestructura.

No deberá integrarse dentro del AST u ORM.

---

# 389. Principle of failure isolation

Un fallo en una conexión secundaria o replica no deberá necesariamente inutilizar todas las conexiones si la arquitectura permite aislarlo.

---

# 390. Principle of primary correctness

Si no existe una replica válida para una lectura que puede usar primary, la política podrá fallback.

Pero si la consistencia requiere primary, no deberá elegirse replica por disponibilidad.

---

# 391. Principle of connection role awareness

Las conexiones podrán declarar roles:

```text
primary
replica
migration
analytics
tenant
```

sin convertir estos nombres en comportamiento hardcoded.

---

# 392. Principle of role-policy separation

El rol describe intención.

La política decide cómo resolverlo.

---

# 393. Principle of configuration validation for routing

Configuraciones inválidas como:

```text
write connection points to read-only replica
```

deberán detectarse cuando sea posible.

---

# 394. Principle of observable routing

En diagnostics deberá poder conocerse:

```text
requested connection
resolved connection
role
host identifier safe representation
routing reason
```

sin exponer secretos.

---

# 395. Principle of connection identity

Una conexión lógica y una conexión física son conceptos distintos.

```text
Connection
→ logical API

DriverConnection
→ physical/session resource
```

Esto permite pooling y reconnection.

---

# 396. Principle of reconnection transparency limits

La infraestructura podrá reconectar antes de una operación segura.

No deberá reconectar silenciosamente dentro de una transacción rota y fingir continuidad.

---

# 397. Principle of transaction-affine connection

Una transacción deberá permanecer asociada a la misma sesión física mientras esté activa.

No podrá migrarse arbitrariamente entre conexiones del pool.

---

# 398. Principle of cursor-affine connection

Un cursor abierto podrá requerir conservar la conexión física hasta su cierre.

El pool deberá respetarlo.

---

# 399. Principle of resource leases

Pools podrán conceptualizar recursos mediante leases:

```text
acquire
use
release
```

Esto ayuda a modelar correctamente propiedad temporal.

---

# 400. Principle of no connection sharing across concurrent scopes unless safe

Una conexión física no deberá compartirse simultáneamente entre operaciones concurrentes si el driver no garantiza seguridad.

---

# 401. Principle of runtime-specific concurrency adapters

Diferencias de concurrencia de:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

deberán resolverse en adapters/runtime infrastructure.

No modificando semántica de Query.

---

# 402. Principle of no assumptions about destructor timing

El sistema no deberá depender exclusivamente de destructores PHP para liberar recursos críticos.

Deberá haber lifecycle explícito.

---

# 403. Principle of deterministic shutdown hooks

Runtime integration deberá ejecutar cleanup en:

```text
request success
request exception
worker reset
worker shutdown
```

cuando aplique.

---

# 404. Principle of worker corruption containment

Si Database detecta que no puede restablecer estado seguro:

```text
worker
```

podrá señalar que debe reciclarse mediante integración con RuntimeManagerServer.

---

# 405. Principle of infrastructure health distinction

Se deberá distinguir:

```text
application query failed
connection unhealthy
database unavailable
configuration invalid
```

para permitir acciones correctas.

---

# 406. Principle of health checks not causing business writes

Los health checks deberán ser:

```text
read-only
low-cost
bounded
```

cuando sea posible.

---

# 407. Principle of liveness vs readiness distinction

El tooling futuro deberá distinguir:

```text
liveness
readiness
database health
migration readiness
```

sin mezclar estos conceptos.

---

# 408. Principle of explicit maintenance mode integration

Operaciones que hagan incompatible temporalmente el schema podrán integrarse con mantenimiento de aplicación.

Database no deberá activar mantenimiento global arbitrariamente.

---

# 409. Principle of schema compatibility awareness

Herramientas de deployment podrán comparar:

```text
application expected schema
actual schema
```

cuando exista metadata suficiente.

---

# 410. Principle of no automatic destructive synchronization

VoltStack no deberá sincronizar automáticamente schema de producción eliminando diferencias no declaradas.

---

# 411. Principle of generated schema as advisory unless explicit

La generación de schema desde ORM podrá ayudar a:

```text
development
diff
migration generation
```

pero no deberá sustituir automáticamente el sistema de migraciones en producción.

---

# 412. Principle of code-first/database-first coexistence

La arquitectura deberá permitir:

```text
ORM metadata → schema
```

y:

```text
existing schema → introspection
```

sin asumir que uno siempre es fuente absoluta de verdad.

---

# 413. Principle of source-of-truth declaration

Cada proyecto podrá declarar qué sistema gobierna cambios:

```text
migrations
external DBA
schema definitions
```

Tooling deberá adaptarse.

---

# 414. Principle of developer intent preservation

Las herramientas automáticas deberán intentar preservar decisiones explícitas del desarrollador.

No deberán regenerar mappings o migrations destruyendo personalizaciones sin advertencia.

---

# 415. Principle of code generation boundaries

Code generation podrá producir:

```text
entity skeleton
migration
repository
metadata cache
```

pero los archivos generados deberán indicar si son:

```text
safe to edit
regenerable
generated-only
```

---

# 416. Principle of deterministic code generation

Con la misma entrada y configuración, generación deberá producir salida estable para minimizar diffs innecesarios.

---

# 417. Principle of no runtime code generation dependency

Una aplicación productiva no deberá requerir generar clases dinámicamente en cada request para funcionar.

Podrán generarse durante build/cache warmup.

---

# 418. Principle of production precomputation

Se favorecerá precomputar:

```text
metadata
container wiring
platform mappings
query function registries
```

cuando produzca beneficios.

---

# 419. Principle of development flexibility

En desarrollo podrá existir fallback a reflexión o discovery dinámico para mejorar experiencia.

---

# 420. Principle of parity validation

Las rutas:

```text
dynamic development metadata
compiled production metadata
```

deberán producir la misma semántica.

---

# 421. Principle of extensibility without performance tax

Registrar puntos de extensión no deberá obligar a ejecutar complejos loops dinámicos cuando no existen extensiones.

Podrán compilarse pipelines.

---

# 422. Principle of compiled pipelines

Pipelines como:

```text
optimizer rules
metadata loaders
query middleware
```

podrán preconstruirse durante bootstrap.

---

# 423. Principle of middleware scarcity

No todo deberá convertirse en middleware.

Middleware será apropiado cuando existe:

```text
ordered cross-cutting behavior
```

y no cuando una llamada directa expresa mejor la dependencia.

---

# 424. Principle of no middleware around pure transformation without need

Transformaciones puras como AST → Plan no necesitan necesariamente middleware genérico.

Las extension rules especializadas suelen ser más claras.

---

# 425. Principle of ordered extension pipelines

Cuando existan múltiples reglas, el orden deberá ser:

```text
deterministic
documented
inspectable
```

No depender accidentalmente del orden de carga de Composer.

---

# 426. Principle of extension priorities with caution

Las priorities podrán existir, pero deberán evitar convertirse en números mágicos difíciles de mantener.

Preferir fases semánticas cuando sea posible.

---

# 427. Principle of phased pipelines

Ejemplo:

```text
Normalization
 ↓
Semantic Rewrite
 ↓
Performance Optimization
 ↓
Platform Adaptation
```

puede ser más estable que una lista global de reglas.

---

# 428. Principle of no optimizer semantic extensions

Una optimización no deberá introducir una característica semántica inexistente.

Las características pertenecen al Query Model/Planner.

---

# 429. Principle of planner strategy extensibility

Las estrategias de planificación podrán ser extensibles por capability o feature.

Pero deberán producir un Execution Plan válido y verificable.

---

# 430. Principle of plan validation

Antes de ejecutar, un Execution Plan podrá validarse para asegurar:

```text
required connection role exists
transaction requirement satisfiable
platform capabilities valid
parameter count valid
```

---

# 431. Principle of no unvalidated generated SQL execution

El compiler deberá producir una representación estructurada (`CompiledQuery`), no únicamente un string suelto.

Esto permitirá validar metadata de ejecución.

---

# 432. Principle of CompiledQuery integrity

`CompiledQuery` deberá mantener relación consistente entre:

```text
SQL placeholders
bindings
binding types
execution metadata
```

---

# 433. Principle of placeholder abstraction

Los placeholders deberán generarse según driver/dialect.

El Query Model no deberá estar acoplado a:

```text
?
:p1
$1
```

---

# 434. Principle of parameter ordering determinism

El orden de parámetros deberá ser estable para un query shape.

Esto facilita prepared statement reuse.

---

# 435. Principle of driver normalization

Driver deberá convertir diferencias de APIs nativas hacia contratos VoltStack.

Ejemplos:

```text
result status
affected rows
last insert identity
error representation
```

---

# 436. Principle of no ORM mapping in driver

Reiteración obligatoria:

```text
Driver
```

nunca conocerá:

```text
entity class
entity property
relationship
repository
```

---

# 437. Principle of connection minimalism

Connection deberá representar:

```text
session
statement
transaction primitives
metadata access
```

No convertirse en una fachada universal Database.

---

# 438. Principle of direct connection API as low-level feature

El desarrollador podrá acceder a una Connection.

Pero hacerlo será considerado una API más baja que Query Builder/ORM.

---

# 439. Principle of safe low-level use

Incluso la Connection API deberá facilitar:

```text
prepared execution
typed bindings
resource cleanup
```

---

# 440. Principle of explicit native handle access

Si se permite obtener el handle PDO/native:

```text
nativeHandle()
```

será una escape hatch claramente marcada.

Las garantías de VoltStack podrán reducirse en ese nivel.

---

# 441. Principle of native-state invalidation awareness

Si el usuario modifica estado del native handle directamente, VoltStack puede perder capacidad de rastrearlo.

La API deberá documentar esta frontera.

---

# 442. Principle of minimal exposed internals

El número de APIs que permiten cruzar fronteras deberá mantenerse pequeño.

Esto conserva capacidad futura de cambiar implementación.

---

# 443. Principle of migration path for users

Cuando una abstracción cambie, documentación deberá proporcionar:

```text
old API
new API
reason
migration steps
```

particularmente para APIs ampliamente usadas.

---

# 444. Principle of DX parity with architecture

Una arquitectura más rigurosa no deberá obligar al usuario común a escribir:

```text
17 objetos para SELECT simple
```

Las APIs de conveniencia son parte esencial del diseño.

---

# 445. Principle of explicit advanced mode

Las capacidades avanzadas deberán estar disponibles sin contaminar flujo común.

Ejemplo:

```php
DB::table('users')->get();
```

y:

```php
$queryEngine->plan(
    $query,
    optimization: OptimizationProfile::Aggressive,
);
```

pueden coexistir.

---

# 446. Principle of progressively inspectable abstractions

Idealmente un desarrollador podrá avanzar:

```text
Builder
→ inspect AST
→ inspect Plan
→ inspect SQL
```

cuando necesite diagnosticar.

Sin requerirlo para uso normal.

---

# 447. Principle of useful defaults, not hidden policy

Los defaults deberán estar documentados.

Ejemplo:

```text
default fetch mode
default transaction isolation behavior
default lazy loading policy
default connection
```

---

# 448. Principle of configuration locality

La configuración de un subsistema deberá agruparse coherentemente.

Evitar una opción de query optimizer escondida dentro de configuración ORM sin razón.

---

# 449. Principle of environment overrides

Configuraciones como:

```text
host
credentials
pool limits
debug
```

podrán variar por environment.

Pero las decisiones estructurales del dominio deberán evitar depender de environment innecesariamente.

---

# 450. Principle of no environment-specific application semantics

`production` no deberá ejecutar una query semánticamente diferente de `development` únicamente por ser producción, salvo política explícita.

---

# 451. Principle of strict production validation

Producción podrá activar verificaciones como:

```text
metadata compiled
debug disabled
unsafe defaults rejected
```

sin cambiar resultados de negocio.

---

# 452. Principle of documented failure modes

Cada subsistema importante deberá documentar:

```text
failure conditions
exception types
state after failure
retryability
cleanup behavior
```

---

# 453. Principle of recovery-aware design

Diseñar una operación incluye diseñar:

```text
how it fails
how it cleans up
whether it retries
whether state remains usable
```

No únicamente el happy path.

---

# 454. Principle of state-machine modeling where useful

Componentes complejos con lifecycle deberán considerar máquinas de estado.

Ejemplos:

```text
Connection
Transaction
Entity state
Migration execution
```

Esto reduce estados inválidos.

---

# 455. Principle of impossible states should be difficult to represent

Tipos y APIs deberán impedir combinaciones inválidas cuando sea práctico.

Ejemplo:

```text
CommittedTransaction
```

no debería permitir otro `commit()` significativo.

---

# 456. Principle of explicit terminal states

Recursos con lifecycle deberán distinguir estados terminales:

```text
closed
committed
rolled back
failed
disposed
```

---

# 457. Principle of idempotent terminal operations where safe

Operaciones como `close()` podrán no hacer nada si ya están cerradas.

Operaciones como segundo `commit()` pueden producir error, porque ocultarlo podría esconder bugs.

Cada caso deberá decidirse semánticamente.

---

# 458. Principle of no undefined behavior

Cuando una operación no sea válida para el estado actual deberá:

```text
fail explicitly
```

No quedar a comportamiento accidental del driver.

---

# 459. Principle of contracts before implementations

Cada subsistema nuevo deberá comenzar definiendo:

```text
responsibility
inputs
outputs
state
errors
contracts
```

antes de elegir implementación concreta.

---

# 460. Principle of reference implementation follows contracts

Las implementaciones oficiales deberán utilizar las mismas APIs públicas/extensibles que terceros cuando sea razonable.

Esto ayuda a validar que las extensiones realmente funcionan.

---

# 461. Principle of internal privileged APIs only when justified

Algunas implementaciones oficiales podrán usar internals por performance.

Pero esas excepciones deberán ser limitadas y no convertir la Extension API en una segunda categoría inutilizable.

---

# 462. Principle of conformance over inheritance

Drivers, platforms y extensiones deberán cumplir contratos.

No se requerirá necesariamente heredar grandes clases base.

Se favorecerá composición.

---

# 463. Principle of inheritance restraint

Las clases base podrán ofrecer comportamiento común cuando exista una jerarquía natural.

No se utilizará herencia para compartir utilidades arbitrarias.

---

# 464. Principle of composition over inheritance

Preferir:

```text
Compiler
 ├── IdentifierQuoter
 ├── ParameterCompiler
 └── ExpressionCompiler
```

a una jerarquía profunda con métodos protegidos difíciles de reemplazar.

---

# 465. Principle of final-by-default consideration

Clases internas podrán ser `final` por defecto cuando extensión por herencia no sea una API soportada.

Los puntos de extensión deberán ser explícitos mediante contratos.

---

# 466. Principle of protected API caution

Un método `protected` en una clase pública puede convertirse accidentalmente en API de extensión.

Se deberá usar conscientemente.

---

# 467. Principle of semantic versioning awareness

Cambios a:

```text
public methods
contracts
behavior guarantees
extension points
```

deberán evaluarse conforme a la política de versiones del framework.

---

# 468. Principle of capability versioning

Nuevas capacidades deberán añadirse de manera que drivers antiguos puedan declarar:

```text
unsupported
```

sin romper interfaces innecesariamente.

---

# 469. Principle of interfaces that can evolve

Interfaces públicas deberán mantenerse enfocadas.

Interfaces enormes son difíciles de extender compatiblemente.

Preferir contratos pequeños y semánticos.

---

# 470. Principle of no marker-interface proliferation

Interfaces sin comportamiento deberán utilizarse únicamente cuando aporten semántica real al Type System o resolución.

---

# 471. Principle of documentation examples use public APIs

La documentación para usuarios deberá evitar enseñar internals.

Los documentos de arquitectura podrán mostrar internals claramente marcados.

---

# 472. Principle of tests as executable examples

Parte de la suite pública podrá funcionar como referencia de comportamiento esperado.

Especialmente para:

```text
drivers
types
query functions
platform capabilities
```

---

# 473. Principle of clear support matrix

La documentación deberá mantener matrices para:

```text
database engines
versions
features
drivers
runtime compatibility
```

sin asumir soporte uniforme.

---

# 474. Principle of compatibility declarations are tested

Si VoltStack declara soporte para una versión de PostgreSQL/MySQL/etc., deberá existir cobertura de integración razonable para dicha combinación.

---

# 475. Principle of minimum supported database versions

Se establecerán versiones mínimas oficiales.

Esto evita cargar el core con workarounds indefinidos para motores obsoletos.

Las versiones exactas serán definidas en documentación de plataformas.

---

# 476. Principle of deprecating old platform behavior

Cuando una versión antigua deje de soportarse, los workarounds asociados deberán poder retirarse sin afectar las abstracciones superiores.

---

# 477. Principle of security patch agility

La separación de drivers y plataformas deberá permitir corregir vulnerabilidades específicas sin rediseñar ORM.

---

# 478. Principle of dependency minimization

`voltstack/database` deberá limitar dependencias externas obligatorias.

Cada librería adicional deberá evaluarse por:

```text
security
maintenance
size
runtime cost
abstraction overlap
```

---

# 479. Principle of no competing core abstractions

No se integrará una librería externa que imponga un segundo:

```text
ORM
Query AST
Connection Manager
Transaction Model
```

en el core salvo decisión arquitectónica explícita.

---

# 480. Principle of wrapping external primitives

Cuando se utilice una librería externa para infraestructura, VoltStack deberá mantener sus contratos propios en las fronteras necesarias para evitar lock-in accidental.

---

# 481. Principle of use native capabilities intelligently

No se deberá reimplementar en PHP aquello que el motor hace mejor cuando exista una forma segura de delegarlo.

Ejemplos:

```text
constraints
indexes
aggregations
sorting
filtering
transactions
```

---

# 482. Principle of no needless data transfer

El Query Engine deberá favorecer ejecutar filtrado/agregación en database cuando semánticamente apropiado.

No cargar miles de filas para filtrarlas en PHP sin necesidad.

---

# 483. Principle of pushdown awareness

Optimizaciones futuras podrán realizar:

```text
predicate pushdown
projection reduction
```

en la representación de consulta.

Siempre preservando semántica.

---

# 484. Principle of minimum selected data

ORM deberá poder seleccionar únicamente campos requeridos cuando la estrategia lo permita.

Pero las entidades parciales deberán tener semántica explícita para evitar inconsistencias.

---

# 485. Principle of partial entities caution

Una entidad parcialmente hidratada puede ser peligrosa.

VoltStack deberá distinguir claramente:

```text
full entity
partial projection
DTO
```

y preferir DTO/projection cuando sea más seguro.

---

# 486. Principle of DTO projection as first-class feature

Consultas de lectura podrán proyectar directamente hacia DTOs.

Esto evita usar entidades administradas para reportes.

---

# 487. Principle of read model separation support

La arquitectura deberá permitir patrones CQRS/read-model sin imponerlos.

Query Builder/DTO hydration podrán funcionar independientemente de UnitOfWork.

---

# 488. Principle of managed entities only when needed

No todo resultado ORM tiene que quedar administrado.

Podrán existir consultas:

```text
read-only
detached
DTO
scalar
```

para reducir memoria.

---

# 489. Principle of read-only optimization

Un PersistenceContext o query read-only podrá omitir determinado change tracking cuando sea seguro.

La semántica deberá ser explícita.

---

# 490. Principle of write intent clarity

Una entidad administrada con cambios pendientes deberá pertenecer a un contexto capaz de persistirlos.

Los read-only contexts deberán rechazar flush de cambios.

---

# 491. Principle of transaction-aware read-only mode

Las transacciones read-only específicas de plataforma podrán utilizarse cuando la capability exista.

No deberán ser requisito para soportar read-only ORM semantics.

---

# 492. Principle of application-level batching APIs

Database podrá proporcionar APIs para:

```text
chunk
batch
stream
bulk
```

con contratos claros sobre memoria y transacciones.

---

# 493. Principle of no hidden transaction per row

Una operación bulk no deberá accidentalmente abrir una transacción independiente por entidad salvo que esa sea la política solicitada.

---

# 494. Principle of transaction batch sizing

Para operaciones enormes podrá permitirse commit por batches.

Esto cambiará garantías atómicas y deberá ser explícito.

---

# 495. Principle of resumability semantics

Si una operación batch puede reanudarse, deberá definir una clave o checkpoint estable.

---

# 496. Principle of idempotent import support

Herramientas de importación podrán facilitar:

```text
upsert
deduplication keys
checkpointing
```

sin asumir una única estrategia.

---

# 497. Principle of export streaming

Exportaciones grandes deberán favorecer streaming para evitar materialización total.

---

# 498. Principle of no arbitrary serialization inside Database Core

Export formats:

```text
CSV
JSON
Parquet
```

pueden pertenecer a extensiones específicas.

Database Core deberá proporcionar rows/streams.

---

# 499. Principle of data lifecycle modularity

Capacidades como:

```text
retention
archival
temporal history
soft deletes
```

deberán integrarse mediante políticas/subsistemas definidos.

No deberán modificar silenciosamente todas las queries del core.

---

# 500. Principle of query scopes visibility

Si se implementan global scopes como:

```text
soft delete
tenant
```

deberá ser posible inspeccionarlos y desactivarlos explícitamente cuando la API lo permita.

No deberán ser condiciones invisibles imposibles de diagnosticar.

---

# 501. Principle of security scopes distinction

Un query scope de conveniencia no sustituye Authorization.

Database no deberá presentar filtros ORM como garantía completa de seguridad de acceso.

---

# 502. Principle of tenant isolation defense in depth

Cuando Multitenancy se integre, podrá aplicar:

```text
connection routing
schema routing
query scope
database constraints
```

según estrategia.

Pero Database Core no asumirá una única forma.

---

# 503. Principle of no tenant leakage through pools

La integración multitenant deberá restaurar cualquier estado de sesión específico del tenant antes de devolver conexiones al pool.

---

# 504. Principle of tenant context immutability per operation

Una query iniciada bajo Tenant A no deberá cambiar silenciosamente a Tenant B durante ejecución.

---

# 505. Principle of transaction tenant affinity

Una transacción tenant-aware deberá permanecer ligada al mismo contexto/connection target durante toda su vida.

---

# 506. Principle of clear package boundaries

Las integraciones opcionales deberán ubicarse en namespaces reconocibles.

Ejemplo:

```text
Database/Integration/Telemetry
Database/Integration/Cache
Database/Integration/Multitenancy
```

o paquetes dedicados si posteriormente se separan.

---

# 507. Principle of no reverse optional dependency

`Quantum/Multitenancy` puede depender de contratos Database.

`Database Core` no deberá depender de implementación Multitenancy.

---

# 508. Principle of package installation order independence

Cuando dos paquetes opcionales sean compatibles, su resultado no deberá depender accidentalmente del orden Composer/service-provider en el que se registraron.

Las prioridades o phases deberán ser deterministas.

---

# 509. Principle of integration conflict detection

Si dos extensiones intentan ocupar una capability exclusiva:

```text
default driver name
same query function
same database type
```

el sistema deberá detectar conflicto en lugar de sobrescribir silenciosamente.

---

# 510. Principle of explicit override policy

Cuando sobrescribir esté permitido deberá requerir:

```text
replace
priority
decorator
```

según contrato.

---

# 511. Principle of extension introspection

Tooling deberá poder mostrar:

```text
registered drivers
types
compilers
query functions
optimizer rules
integrations
```

para diagnóstico.

---

# 512. Principle of deterministic extension resolution

Dados los mismos paquetes y configuración, la resolución de extensiones deberá ser estable.

---

# 513. Principle of build-time conflict detection

Cuando sea posible, conflictos de extensión deberán detectarse durante:

```text
bootstrap
container compilation
cache warmup
```

no bajo tráfico.

---

# 514. Principle of no reflection-based magic registration by default

Auto-discovery mediante reflexión podrá existir, pero deberá compilarse o limitarse.

Los registros críticos deberán ser explícitos o cacheables.

---

# 515. Principle of optional convention discovery

Para DX se podrán descubrir:

```text
entities
migrations
factories
seeders
```

por convenciones.

El usuario deberá poder configurar rutas explícitas.

---

# 516. Principle of discovery scope

Discovery deberá limitarse a directorios/namespaces definidos.

No escanear todo el proyecto indiscriminadamente en cada request.

---

# 517. Principle of discovery caching

Resultados de discovery estables deberán poder cachearse en producción.

---

# 518. Principle of developer cache reset tools

Cuando caches compiladas puedan volverse stale, CLI deberá proporcionar comandos claros de:

```text
clear
rebuild
warm
inspect
```

---

# 519. Principle of actionable stale-cache errors

Cuando se detecte una cache incompatible deberá indicarse cómo regenerarla.

---

# 520. Principle of no silent cache corruption recovery in production

Si un artefacto crítico está corrupto y no puede regenerarse con seguridad, deberá fallar claramente.

---

# 521. Principle of startup validation options

Producción podrá ejecutar una fase opcional de validación para comprobar:

```text
configuration
driver availability
metadata cache
migration status
```

antes de recibir tráfico.

---

# 522. Principle of database availability decoupled from application bootstrap

Salvo que configuración indique lo contrario, no deberá ser obligatorio conectar a database para construir el Container.

Esto permite comandos o páginas que no requieren DB.

---

# 523. Principle of fail-fast on first required use

Una vez que una operación requiere database, errores de configuración/conexión deberán producirse inmediatamente y con diagnóstico claro.

---

# 524. Principle of configuration layering

La resolución podrá combinar:

```text
framework defaults
application config
environment
runtime overrides
```

siguiendo prioridad documentada.

---

# 525. Principle of no arbitrary runtime configuration mutation

Las modificaciones dinámicas a configuración persistente deberán usar APIs específicas.

No modificar arrays globales esperando que servicios existentes se actualicen mágicamente.

---

# 526. Principle of configuration snapshot per service lifecycle

Una Connection o PersistenceContext deberá conocer qué configuración aplica durante su lifecycle.

Los cambios dinámicos no deberán modificar semántica en mitad de una transacción.

---

# 527. Principle of explicit dynamic connection creation

Si una aplicación necesita conexiones calculadas dinámicamente podrá utilizar:

```text
ConnectionDefinition
ConnectionFactory
DynamicConnectionResolver
```

sin editar configuración global.

---

# 528. Principle of bounded dynamic connections

Los runtimes persistentes deberán evitar acumular indefinidamente conexiones dinámicas.

Connection Manager deberá soportar políticas de:

```text
eviction
TTL
max size
cleanup
```

si esa feature está habilitada.

---

# 529. Principle of tenant connection cardinality awareness

Multitenancy database-per-tenant puede generar miles de connection definitions.

La integración deberá diseñarse para no crear un pool permanente por cada tenant automáticamente.

---

# 530. Principle of cache cardinality awareness

Los caches cuyos keys dependan de:

```text
tenant
query
schema
connection
```

deberán considerar crecimiento de cardinalidad.

---

# 531. Principle of telemetry cardinality awareness

Labels de métricas no deberán incluir valores no acotados como:

```text
raw SQL
user id
tenant id
query parameter
```

por defecto.

---

# 532. Principle of tracing detail separated from metrics labels

Información de alta cardinalidad podrá ir en traces/logs controlados en vez de labels de métricas.

---

# 533. Principle of query fingerprinting

Telemetry podrá usar un fingerprint basado en query shape en lugar de valores concretos.

Esto mejora privacidad y agregación.

---

# 534. Principle of stable query fingerprints

La generación de fingerprint deberá ser determinista para el mismo query shape y compatible con cache semantics cuando corresponda.

---

# 535. Principle of query origin diagnostics

En desarrollo podrá capturarse origen:

```text
repository method
controller/action
stack frame
component
```

pero deberá ser opcional debido a coste.

---

# 536. Principle of no stack trace by default in hot path production

Capturar stack traces por cada query será opt-in o debug-only.

---

# 537. Principle of performance attribution

Cuando Telemetry esté activa, deberá poder separar tiempo aproximado de:

```text
planning
compilation
connection acquisition
database execution
hydration
```

cuando el nivel de instrumentación lo permita.

---

# 538. Principle of no double instrumentation

Las integraciones deberán coordinarse para evitar medir la misma query múltiples veces accidentalmente.

---

# 539. Principle of standardized telemetry events

Los eventos de telemetría deberán usar modelos estables para permitir exporters independientes.

---

# 540. Principle of diagnostics redaction by default

Una query de:

```text
email = ?
password = ?
```

podrá mostrar tipos o placeholders, no valores sensibles.

---

# 541. Principle of configurable binding visibility

En desarrollo el usuario podrá permitir bindings no sensibles.

La clasificación de sensibilidad deberá poder respetarse.

---

# 542. Principle of sensitive type metadata

Tipos como:

```text
PasswordHash
AccessToken
Secret
```

podrán marcar bindings/result fields como sensibles mediante metadata.

---

# 543. Principle of no credentials in exception strings

Las excepciones de connection no deberán serializar DSNs completos con passwords.

---

# 544. Principle of safe DSN representation

Podrá existir:

```text
ConnectionDescriptor
```

para diagnostics con:

```text
driver
host
port
database
role
```

pero credenciales redactadas.

---

# 545. Principle of secure configuration object stringification

`__toString()`, dump y debug info de configuraciones deberán evitar secretos.

---

# 546. Principle of secret provider extensibility

Credenciales podrán provenir de:

```text
environment
secret manager
runtime provider
```

mediante integración, sin que Driver necesite conocer proveedores externos.

---

# 547. Principle of credential resolution timing

Las credenciales podrán resolverse justo antes de conectar para permitir rotación.

Pero su lifecycle y cache deberán definirse.

---

# 548. Principle of credential rotation support

Connection infrastructure deberá poder descartar/recrear conexiones cuando credenciales roten.

---

# 549. Principle of no automatic retry on authentication failure

Errores de autenticación normalmente no deberán generar retries agresivos.

---

# 550. Principle of bounded connection attempts

Connection creation deberá respetar:

```text
connect timeout
retry policy
cancellation
```

---

# 551. Principle of connection establishment observability

Se podrá medir:

```text
connection attempts
latency
failures
reconnections
```

sin exponer secretos.

---

# 552. Principle of query timeout separation

Deberán distinguirse cuando la plataforma lo permita:

```text
connection timeout
statement timeout
lock timeout
transaction timeout
```

---

# 553. Principle of timeout restoration

Si un timeout se configura mediante session state temporal deberá restaurarse antes de reutilizar la conexión.

---

# 554. Principle of cancellation state handling

Después de cancelar una consulta, Driver/Connection deberán determinar si la sesión sigue siendo reutilizable.

---

# 555. Principle of resource validity checks

Antes de devolver una conexión al pool podrán ejecutarse verificaciones ligeras cuando exista riesgo de estado inválido.

No necesariamente un ping por cada query.

---

# 556. Principle of pool policy abstraction

El Connection Pool deberá separar:

```text
resource creation
health validation
acquisition policy
eviction policy
```

---

# 557. Principle of poolless operation

Database deberá poder funcionar sin pooling interno.

Especialmente en runtimes donde otra capa gestione pooling.

---

# 558. Principle of external pool compatibility

Drivers futuros podrán conectarse a proxies/pools externos como infraestructura transparente.

VoltStack no deberá asumir control exclusivo del pooling físico.

---

# 559. Principle of layered resilience

La resiliencia puede existir en:

```text
driver
connection manager
query execution
application
```

pero cada retry deberá tener ownership claro para evitar retries multiplicativos.

---

# 560. Principle of retry amplification prevention

No deberá ocurrir:

```text
driver retries 3x
connection retries 3x
executor retries 3x
```

produciendo 27 intentos inesperados.

La política deberá centralizarse o coordinarse.

---

# 561. Principle of retry observability

Cada retry deberá ser observable como intento relacionado con la operación original.

---

# 562. Principle of backoff budgets

Los retries deberán respetar un tiempo total máximo además del número de intentos cuando corresponda.

---

# 563. Principle of transaction retries at correct abstraction

Retries por:

```text
serialization failure
deadlock
```

pueden requerir repetir toda la transacción, no únicamente el último statement.

Por ello Transaction Manager deberá poseer esta estrategia.

---

# 564. Principle of transactional callback retry awareness

Una callback transaccional retryable puede ejecutarse varias veces.

La API deberá documentarlo claramente porque no debe contener side effects externos no idempotentes.

---

# 565. Principle of no retry magic for arbitrary callbacks

Los retries de transacción deberán ser opt-in o seguir políticas claras.

No asumir que toda callback puede repetirse.

---

# 566. Principle of connection error translation

Drivers deberán mapear errores nativos hacia categorías estándar cuando sea posible.

---

# 567. Principle of SQLSTATE preservation

Cuando exista SQLSTATE deberá conservarse como metadata diagnóstica.

---

# 568. Principle of error taxonomy extensibility

Nuevos drivers deberán poder añadir metadata sin obligar a modificar todas las capas superiores.

---

# 569. Principle of exception chain transparency for developers

En debug mode deberá ser posible inspeccionar toda la cadena de causas.

---

# 570. Principle of stable high-level exception contracts

Las excepciones que aplicaciones puedan capturar deberán tener semántica estable.

---

# 571. Principle of query-builder immutability decision consistency

El diseño final deberá decidir explícitamente si Query Builder es:

```text
mutable fluent builder
```

o:

```text
immutable fluent builder
```

y aplicar la decisión consistentemente.

El AST resultante deberá favorecer inmutabilidad independientemente de esta elección.

---

# 572. Principle of reusable query definitions

El sistema deberá permitir definir queries reutilizables sin riesgo de mutación accidental.

Si el Builder es mutable deberán existir:

```text
clone
new query
immutable specification
```

según corresponda.

---

# 573. Principle of no state carry-over between terminal operations

Ejecutar:

```php
$query->get();
$query->count();
```

no deberá modificar silenciosamente el Builder de forma que cambie consultas posteriores, salvo API documentada.

---

# 574. Principle of terminal operation clarity

Métodos como:

```text
get
first
exists
count
stream
update
delete
```

deberán tener semántica predecible.

---

# 575. Principle of aggregate query correctness

Transformar una query a `count()` deberá respetar:

```text
distinct
grouping
subqueries
unions
```

mediante Planner/Query transformations correctas, no reemplazo ingenuo de SELECT.

---

# 576. Principle of pagination count separation

Cuando paginación requiera total, la query de count deberá modelarse separadamente.

Esto permitirá optimizar o evitar count cuando no sea necesario.

---

# 577. Principle of simple pagination support

El sistema podrá ofrecer paginación sin total para reducir coste.

---

# 578. Principle of cursor pagination preferred for scalable navigation

Para datasets grandes, cursor pagination deberá ser first-class cuando exista orden estable.

---

# 579. Principle of cursor opacity

Los cursores expuestos a clientes deberán poder codificarse como tokens opacos.

No depender de revelar SQL o estructura interna.

---

# 580. Principle of cursor validation

Un cursor deberá validarse antes de incorporarse a Query AST.

---

# 581. Principle of no direct user-controlled raw cursor expressions

La información del cursor deberá convertirse en bindings/estructuras seguras.

---

# 582. Principle of eager loading through plans

Eager loading deberá producir planes de carga explícitos.

No una serie arbitraria de callbacks ocultos.

---

# 583. Principle of relationship batch planner

El sistema podrá contar con un Relation Load Planner que determine:

```text
single query join
secondary IN query
multiple batches
```

según relación y cardinalidad.

---

# 584. Principle of avoiding cartesian explosion

Eager loading mediante JOIN no deberá utilizarse siempre.

El planner deberá poder elegir secondary queries cuando múltiples colecciones producirían duplicación masiva.

---

# 585. Principle of hydration identity deduplication

Cuando JOIN produzca varias filas para una entidad, el Hydrator + IdentityMap deberán evitar crear instancias duplicadas administradas.

---

# 586. Principle of collection assembly determinism

Las colecciones creadas por hydration deberán preservar:

```text
ordering
uniqueness semantics
ownership
```

según metadata.

---

# 587. Principle of query planner not owning ORM relation semantics

El Query Planner general podrá conocer joins y SQL.

La decisión de qué relaciones cargar deberá originarse en ORM Relation Load Planner y expresarse como queries.

---

# 588. Principle of no ORM-specific nodes in generic Query AST unless generalized

El AST genérico no deberá contener:

```text
LoadUserPostsNode
```

sino conceptos generales:

```text
Select
Join
Predicate
```

La traducción ORM ocurrirá antes.

---

# 589. Principle of domain-specific plans above generic plans

ORM podrá construir:

```text
HydrationPlan
PersistencePlan
RelationLoadPlan
```

que luego produzcan Generic Query Plans.

Esto conserva fronteras.

---

# 590. Principle of one-way lowering

La arquitectura deberá favorecer transformación hacia niveles más concretos:

```text
Domain Intent
 ↓
Generic Query
 ↓
Execution Plan
 ↓
SQL
```

No requerir que niveles bajos reconstruyan intención ORM.

---

# 591. Principle of information preservation during lowering

La información necesaria para diagnostics o semantics deberá viajar como metadata estructurada.

No deberá ser reconstruida desde SQL.

---

# 592. Principle of clear pipeline handoff types

Cada fase deberá recibir y devolver tipos reconocibles.

Ejemplo:

```text
QueryDefinition
→ QueryAst
→ SemanticQuery
→ LogicalPlan
→ PhysicalPlan
→ CompiledQuery
```

Los nombres concretos se formalizarán en documentos posteriores.

---

# 593. Principle of no ambiguous "Query" object

Se evitará usar una única clase:

```text
Query
```

que cambia internamente de significado a través de todo el pipeline.

Cada etapa deberá ser distinguible.

---

# 594. Principle of pipeline phase ownership

Cada transformación deberá pertenecer a un servicio específico.

No un método gigante:

```php
processQuery()
```

que construye, optimiza, compila y ejecuta.

---

# 595. Principle of pipeline bypass visibility

Raw SQL podrá omitir fases.

El sistema deberá saber que se trata de:

```text
RawExecution
```

para diagnostics y garantías.

---

# 596. Principle of platform-native query extensions as typed nodes where valuable

Features frecuentes como:

```text
JSON operators
full text
RETURNING
```

podrán modelarse con nodes/capabilities en vez de raw SQL cuando exista suficiente valor.

---

# 597. Principle of core AST size restraint

No se deberá intentar modelar cada extensión SQL conocida dentro del core inicial.

El sistema debe ser extensible.

---

# 598. Principle of AST extension namespace isolation

Nodes específicos de plataforma deberán ubicarse claramente como:

```text
Platform Extension
```

para evitar contaminar el AST portable.

---

# 599. Principle of compiler responsibility for syntax

AST expresa semántica.

Compiler/Dialect expresan sintaxis.

---

# 600. Principle of semantic graph bounded scope

El Semantic Graph deberá contener únicamente información necesaria para resolver/validar/planner.

No convertirse en un segundo ORM metadata graph completo.

---

# 601. Principle of schema metadata reference, not duplication

Semantic Analysis podrá referenciar Schema Metadata.

No copiar toda la estructura de schema por cada query.

---

# 602. Principle of immutable metadata sharing

Metadata inmutable podrá compartirse entre scopes para mejorar rendimiento.

---

# 603. Principle of no entities in shared caches

Entidades administradas no deberán almacenarse en caches process-wide por defecto.

Un eventual second-level entity cache almacenará datos/representaciones desacopladas, no instancias ligadas a PersistenceContext.

---

# 604. Principle of second-level cache optionality

Second-level cache será una capacidad avanzada opcional.

ORM deberá funcionar correctamente sin ella.

---

# 605. Principle of transactional cache coordination

Si se implementa entity/result cache con writes, la invalidación deberá coordinarse con commit.

No invalidar/publicar permanentemente antes de saber que el transaction commit tuvo éxito.

---

# 606. Principle of cache stampede awareness

Caches de alto tráfico podrán soportar estrategias contra stampede mediante integración Cache.

No deberán incorporarse locks globales en Database Core.

---

# 607. Principle of no cache-induced correctness regression

Desactivar cache no debe corregir bugs funcionales.

Si ocurre, la cache está violando semántica.

---

# 608. Principle of test-with-cache-disabled

Las suites deberán probar funcionamiento base sin caches opcionales.

---

# 609. Principle of test-with-runtime-reuse

Las suites deberán probar servicios persistent-worker con múltiples ciclos.

---

# 610. Principle of test-with-failure-injection

Infraestructura deberá poder probar:

```text
disconnect
timeout
deadlock
commit failure
cursor failure
```

mediante doubles o entornos controlados.

---

# 611. Principle of deterministic fake behavior

Database fakes para unit testing deberán tener semántica documentada.

No deberán pretender reproducir todo SQL si no lo hacen.

---

# 612. Principle of integration tests for database semantics

Constraints, transactions, locking y SQL deberán probarse contra motores reales.

Mocks no son suficientes.

---

# 613. Principle of testing pyramid

La suite deberá combinar:

```text
unit
contract
integration
architecture
performance
```

Cada nivel cubre riesgos distintos.

---

# 614. Principle of no production-only architecture path

El código que únicamente se ejecuta en producción debe tener forma de probarse en CI.

---

# 615. Principle of feature flags with cleanup plan

Features experimentales detrás de flags deberán tener:

```text
owner
default
expiry/review
migration path
```

para evitar flags eternos.

---

# 616. Principle of stable defaults across minor versions

Los defaults que cambien semántica no deberán modificarse silenciosamente en releases menores.

---

# 617. Principle of performance defaults may evolve carefully

Defaults puramente de rendimiento podrán ajustarse si conservan semántica, pero deberán documentarse cuando tengan impacto observable.

---

# 618. Principle of no unpredictable adaptive magic by default

Un sistema que cambia estrategia automáticamente según runtime metrics deberá ser:

```text
bounded
observable
deterministic enough
configurable
```

No activarse sin madurez suficiente.

---

# 619. Principle of explicit optimization profiles

Si se requieren diferentes perfiles podrán modelarse:

```text
Default
LowMemory
Throughput
Latency
```

en lugar de heurísticas ocultas.

---

# 620. Principle of sensible defaults first

La mayoría de aplicaciones no debería necesitar elegir un optimization profile.

---

# 621. Principle of portability test corpus

VoltStack deberá mantener consultas comunes que se compilen/ejecuten contra todas las plataformas soportadas.

---

# 622. Principle of canonical logical semantics

El Query Model deberá documentar la semántica lógica esperada independientemente del SQL resultante.

---

# 623. Principle of dialect-specific tests

Además de tests lógicos, cada dialecto deberá probar:

```text
quoting
DDL syntax
limit
returning
upsert
locks
JSON
```

según capabilities.

---

# 624. Principle of database version test matrix

Cuando una capability dependa de versión, los tests deberán cubrir los límites importantes.

---

# 625. Principle of CI cost awareness

La matriz completa puede distribuirse entre:

```text
fast PR suite
nightly suite
release suite
```

sin sacrificar validación antes de releases.

---

# 626. Principle of no unsupported-version accidental behavior

Si una versión de DB está fuera del soporte oficial:

```text
warn
reject
best-effort explicitly unsupported
```

según política.

No fingir soporte certificado.

---

# 627. Principle of graceful platform discovery failure

Si no puede determinarse versión/capability, el sistema deberá utilizar una política conservadora o error explícito.

---

# 628. Principle of metadata provenance

Metadata deberá poder indicar de dónde provino:

```text
attributes
compiled cache
programmatic mapping
schema introspection
```

Esto facilita diagnóstico.

---

# 629. Principle of mapping conflict detection

Si dos fuentes asignan contradictoriamente la misma propiedad, deberá existir una política determinista y preferiblemente error.

---

# 630. Principle of mapping precedence explicitness

Si se permiten múltiples fuentes, la prioridad deberá documentarse.

---

# 631. Principle of no accidental attribute dependency

Las entidades Data Mapper podrán usar Attributes, pero el ORM no deberá exigir que toda metadata viva dentro de la clase si se soporta mapping programático.

---

# 632. Principle of metadata validation completeness

La compilación deberá detectar:

```text
duplicate columns
invalid identifiers
broken relations
unknown types
invalid cascade
invalid ID configuration
```

antes de runtime cuando sea posible.

---

# 633. Principle of metadata error locality

Los errores deberán identificar:

```text
entity
property
mapping source
rule
```

---

# 634. Principle of metadata compilation determinism

La metadata compilada deberá ser estable independientemente del orden de filesystem discovery.

---

# 635. Principle of no reflection order dependency

El comportamiento no deberá depender de orden no garantizado por Reflection API.

---

# 636. Principle of model conventions centralized

Convenciones como:

```text
class → table
property → column
pivot naming
foreign key naming
```

deberán implementarse mediante componentes especializados y configurables.

No repetirse por todo ORM.

---

# 637. Principle of naming strategy abstraction

Podrá existir:

```text
NamingStrategyInterface
```

si se justifica por soporte de diferentes convenciones.

---

# 638. Principle of explicit identifiers over inferred strings internally

Después de resolver naming, internals deberán trabajar con tipos/metadata resueltos.

No recalcular nombres repetidamente.

---

# 639. Principle of compile conventions early

Las convenciones deberán convertirse a metadata explícita durante compilación.

En runtime:

```text
metadata.table = users
```

no:

```text
calculate table name again
```

---

# 640. Principle of no excessive reflection in repositories

Repository deberá consumir EntityMetadata ya compilada.

---

# 641. Principle of repository statelessness where possible

Repositories deberán ser preferentemente stateless respecto a entidades.

El estado vive en PersistenceContext.

Esto permite reutilización segura si sus dependencias son scoped correctamente.

---

# 642. Principle of repository identity via entity metadata

Repository resolver deberá identificar qué entidad administra de forma explícita.

---

# 643. Principle of custom repository compatibility

Entidades podrán declarar custom repositories sin sustituir el EntityManager.

---

# 644. Principle of no business logic requirement in repositories

El framework permitirá repositories ricos, pero no exigirá colocar lógica de dominio allí.

---

# 645. Principle of repository query reuse

Custom repositories deberán utilizar APIs ORM/Query públicas, no SQL interno del UnitOfWork.

---

# 646. Principle of entity manager orchestration restraint

EntityManager coordina.

No deberá convertirse en implementación directa de:

```text
hydration
SQL generation
metadata parsing
connection retry
```

---

# 647. Principle of entity manager scope explicitness

Resolver un EntityManager deberá implicar un PersistenceContext claro.

Bajo HTTP normalmente:

```text
request scoped
```

---

# 648. Principle of multiple entity managers only when semantically needed

La arquitectura podrá soportar múltiples managers/connections, pero la aplicación simple tendrá uno por defecto.

---

# 649. Principle of cross-manager entity prohibition

Una entidad administrada por PersistenceContext A no deberá insertarse silenciosamente dentro de contexto B.

Deberá existir error o operación explícita.

---

# 650. Principle of cross-database relationship limitations

Relaciones ORM entre bases independientes pueden no ser representables mediante FK/join.

La arquitectura deberá tratarlas como capacidad avanzada, no prometer comportamiento transparente.

---

# 651. Principle of no distributed joins illusion

Si dos entidades viven en conexiones distintas, Query Engine SQL no deberá fingir que puede realizar un JOIN normal.

Podrán existir loaders de aplicación específicos.

---

# 652. Principle of transactional guarantees stay local by default

EntityManager con una conexión puede garantizar transacción local.

Múltiples conexiones requieren garantías adicionales explícitas.

---

# 653. Principle of transaction manager composition

En el futuro podrá existir un coordinador multi-connection separado.

No deberá complicar Transaction Manager local inicial.

---

# 654. Principle of same-connection flush grouping

Persistence Planner deberá agrupar operaciones por conexión/contexto cuando corresponda.

---

# 655. Principle of transaction-required flush configurable

El ORM podrá ejecutar `flush()` dentro de transacción cuando haya múltiples operaciones.

La política concreta deberá ser segura y documentada.

---

# 656. Principle of no nested transaction illusion

Llamar una API transaccional dentro de otra no significa necesariamente dos transacciones físicas.

El sistema deberá documentar savepoint/nesting semantics.

---

# 657. Principle of transaction propagation model

Podrán existir estrategias futuras como:

```text
required
requires new
supports
```

si se justifican.

No deberán añadirse al core antes de necesidad real.

---

# 658. Principle of transaction callbacks lifecycle

Callbacks como:

```text
afterCommit
afterRollback
```

deberán pertenecer al TransactionContext y limpiarse al terminar.

---

# 659. Principle of no callback leakage

Callbacks de Transaction A no deberán sobrevivir a Transaction B en workers persistentes.

---

# 660. Principle of query lifecycle boundaries

Una query estructurada deberá recorrer etapas con estado limitado a la ejecución.

```text
QueryContext
→ disposed after operation
```

No almacenarse globalmente.

---

# 661. Principle of execution result lifecycle

Los resultados buffered pueden vivir independientemente.

Los resultados streaming pueden depender de conexión.

Esta diferencia deberá estar explícitamente modelada.

---

# 662. Principle of no use-after-close cursor

Acceder a un Cursor cerrado deberá producir comportamiento definido, preferentemente excepción clara.

---

# 663. Principle of cursor automatic cleanup fallback

Además de `close()`, el scope cleanup deberá cerrar cursores abandonados cuando pueda rastrearlos.

---

# 664. Principle of connection pool exhaustion visibility

Cuando no pueda adquirirse conexión dentro del límite deberá producirse excepción específica con diagnostics.

No bloquear indefinidamente.

---

# 665. Principle of pool queue policy explicitness

Si existe espera por conexión, deberá haber:

```text
timeout
max waiters
cancellation
```

según implementación.

---

# 666. Principle of no unbounded connection creation

Pooling/dynamic connections deberán respetar límites configurados.

---

# 667. Principle of connection pool metrics

Podrán exponerse:

```text
active
idle
waiting
created
discarded
timeouts
```

mediante Telemetry.

---

# 668. Principle of database backpressure

La infraestructura deberá poder proteger la base de datos de crecimiento ilimitado de concurrencia desde un worker.

---

# 669. Principle of concurrency limits are policy

Los límites deberán configurarse en infraestructura/runtime, no hardcodearse en Query.

---

# 670. Principle of no asynchronous assumption for PDO

El driver PDO se tratará como bloqueante.

Las APIs no deberán prometer concurrencia no existente.

---

# 671. Principle of future async driver separation

Cuando exista soporte async real podrá introducirse una abstracción compatible sin modificar modelos AST/ORM.

---

# 672. Principle of synchronous API remains valid

Añadir async en el futuro no deberá obligar a eliminar la API síncrona.

---

# 673. Principle of observability ports

Core deberá emitir observabilidad mediante contratos estrechos.

Ejemplo:

```text
QueryTelemetryPort
ConnectionTelemetryPort
TransactionTelemetryPort
```

o una abstracción equivalente bien delimitada.

---

# 674. Principle of event/telemetry separation

Un evento de dominio Database y un span de telemetry son conceptos diferentes.

Podrán originarse del mismo hecho, pero no serán necesariamente el mismo objeto.

---

# 675. Principle of audit separation

Audit logging puede requerir garantías distintas de telemetry.

Database deberá proporcionar hooks adecuados sin asumir que métricas sustituyen auditoría.

---

# 676. Principle of audit durability explicitness

Si un audit log requiere persistencia garantizada deberá usar una arquitectura específica.

No depender únicamente de listeners best-effort.

---

# 677. Principle of logging not in hot loops by default

El core no deberá escribir logs verbosos por cada fila/hydration salvo debug explícito.

---

# 678. Principle of structured diagnostics

Los diagnostics deberán utilizar objetos estructurados y luego formatearse.

No construir strings irreversibles demasiado pronto.

---

# 679. Principle of localization outside core diagnostics model

El core podrá proporcionar códigos/mensajes técnicos.

La localización para usuario final deberá mantenerse fuera de internals críticos.

---

# 680. Principle of stable diagnostic codes

Errores importantes podrán tener códigos estables para tooling.

Ejemplo:

```text
DB-CONN-001
DB-ORM-014
```

La nomenclatura exacta se definirá posteriormente.

---

# 681. Principle of no exception-message parsing

La aplicación/tooling no deberá necesitar parsear texto de excepción para conocer categoría.

Usar tipos y metadata.

---

# 682. Principle of source-location diagnostics

Errores de mapping/query podrán incluir localización cuando se conozca:

```text
entity property
migration file
query builder origin
```

---

# 683. Principle of platform SQL diagnostics separation

En error de SQL se podrá mostrar:

```text
compiled SQL template
```

y bindings redactados por separado.

No interpolar valores de forma potencialmente engañosa.

---

# 684. Principle of safe query logging

Los query logs deberán poder guardar:

```text
fingerprint
SQL template
duration
row count
```

sin bindings por defecto en producción.

---

# 685. Principle of transaction correlation

Queries ejecutadas dentro de transacción deberán poder correlacionarse con un Transaction ID diagnóstico.

---

# 686. Principle of persistence correlation

Un `flush()` podrá tener ID diagnóstico para asociar:

```text
change detection
queries
events
commit
```

sin exponerlo al dominio necesariamente.

---

# 687. Principle of migration correlation

Cada ejecución de migration deberá poder correlacionarse por:

```text
migration id
batch id
execution id
```

---

# 688. Principle of no telemetry identifiers as business identifiers

Los IDs diagnósticos son diferentes de IDs de dominio.

No deberán persistirse como lógica de negocio salvo intención explícita.

---

# 689. Principle of operational command safety

CLI destructivo deberá proporcionar mecanismos como:

```text
confirmation
--force
environment guard
dry-run
```

según severidad.

---

# 690. Principle of automation-friendly CLI

A la vez, CI/deployments deberán poder usar comandos sin interacción cuando se especifique claramente.

---

# 691. Principle of exit-code semantics

CLI deberá usar códigos de salida coherentes para:

```text
success
validation failure
connection failure
migration failure
unsafe operation rejected
```

---

# 692. Principle of machine-readable output

Herramientas administrativas podrán soportar salida:

```text
JSON
```

además de formato humano.

---

# 693. Principle of no CLI-only core logic

Los comandos deberán delegar en servicios.

La lógica no deberá existir únicamente dentro de comandos.

---

# 694. Principle of same APIs for automation

Migrations y Schema deberán usar servicios reutilizables desde:

```text
CLI
tests
deployment tooling
application utilities
```

---

# 695. Principle of test transaction isolation

Testing podrá proporcionar transacciones por test.

Pero deberá respetar diferencias de:

```text
nested transactions
DDL
multiple connections
```

---

# 696. Principle of no false test isolation

Si una base o feature no puede revertirse mediante transacción, el framework deberá utilizar otra estrategia o informar la limitación.

---

# 697. Principle of deterministic database reset in tests

Herramientas de testing deberán proporcionar estrategias explícitas:

```text
transaction rollback
truncate
migrate fresh
snapshot restore
```

---

# 698. Principle of test strategy selection

No se impondrá una sola estrategia para todos los proyectos.

La estrategia deberá elegirse según necesidades y capabilities.

---

# 699. Principle of production code path testing

Siempre que sea posible, tests deberán utilizar los mismos compiladores/executors que producción.

---

# 700. Principle of fakes limited scope

Un Database Fake podrá ser útil para verificar:

```text
query was requested
transaction callback
```

pero no deberá sustituir integration tests para semántica SQL.

---

# 701. Principle of no fake SQL engine unless intentionally built

VoltStack no intentará recrear MySQL/PostgreSQL en memoria para testing.

SQLite podrá utilizarse cuando la aplicación realmente sea compatible, no como sustituto universal.

---

# 702. Principle of test database parity awareness

La documentación deberá advertir diferencias cuando tests usan SQLite y producción PostgreSQL/MySQL.

---

# 703. Principle of configurable strictness

Algunas verificaciones podrán tener modos:

```text
off
warn
strict
```

Ejemplos:

```text
N+1
lazy loading
unsafe migration
schema mismatch
```

---

# 704. Principle of strict mode does not alter valid semantics

Strict mode podrá rechazar prácticas peligrosas, pero no deberá hacer que una query válida devuelva datos diferentes.

---

# 705. Principle of development warnings are actionable

Toda warning deberá proporcionar:

```text
problem
location
impact
recommended action
```

cuando sea posible.

---

# 706. Principle of warning deduplication

En workers o loops, la misma warning no deberá inundar logs indefinidamente.

Se podrán aplicar límites por scope/fingerprint.

---

# 707. Principle of warnings do not become hidden failures

Si una condición requiere detener la operación por seguridad, deberá ser error, no simple warning ignorada.

---

# 708. Principle of configuration schema

La configuración Database deberá poseer un schema o contratos verificables.

No depender únicamente de claves libres.

---

# 709. Principle of unknown configuration detection

Opciones desconocidas deberán poder detectarse para evitar typos silenciosos.

---

# 710. Principle of secure defaults for TLS

Cuando un motor/driver soporte conexión segura, VoltStack deberá facilitar TLS y verificación correcta.

La política exacta dependerá del entorno y driver.

---

# 711. Principle of no disable-TLS silent fallback

Si TLS es obligatorio y falla, no deberá reconectar automáticamente sin TLS.

---

# 712. Principle of server identity verification

Configuraciones TLS deberán distinguir cifrado de verificación de identidad.

---

# 713. Principle of credential separation from repository config

La documentación deberá recomendar no almacenar secretos directamente en repositorios de código.

---

# 714. Principle of secrets are resolved not logged

Los providers de secretos deberán retornar valores únicamente a infraestructura que los necesita.

---

# 715. Principle of raw query trust boundary

Raw SQL suministrado por desarrollador se considera código confiable.

Input de usuario nunca deberá pasar directamente a Raw SQL.

---

# 716. Principle of no automatic raw SQL sanitization claims

VoltStack no deberá prometer que puede volver seguro cualquier string SQL arbitrario mediante "sanitización".

La seguridad provendrá de:

```text
bindings
structured API
trusted raw fragments
```

---

# 717. Principle of structured query API preferred

La documentación deberá recomendar Query Builder/ORM para consultas dinámicas.

---

# 718. Principle of raw SQL remains powerful

Raw SQL no será artificialmente limitado.

Debe permitir aprovechar capacidades completas del motor cuando el desarrollador asume responsabilidad explícita.

---

# 719. Principle of raw SQL observability metadata

Raw APIs podrán aceptar metadata como:

```text
intent
timeout
query tag
sensitive bindings
```

para integrarse con infraestructura.

---

# 720. Principle of intent required when raw SQL ambiguity matters

Cuando routing/retry requiera conocer intención y no pueda inferirse seguramente, la API podrá solicitarla explícitamente.

---

# 721. Principle of no SQL regex as source of truth

Regex sobre SQL no deberá utilizarse como base principal para determinar:

```text
transaction safety
read/write semantics
schema changes
```

cuando exista metadata estructurada.

---

# 722. Principle of schema operations use schema model

Las operaciones generadas por migrations deberán producir Schema Operations tipadas.

No strings SQL prematuros.

---

# 723. Principle of execution infrastructure reuse

Query y Schema podrán compartir:

```text
Executor
Connection
Transaction primitives
Telemetry
```

cuando sus contratos sean suficientemente generales.

---

# 724. Principle of execution semantics typed

Executor deberá poder distinguir tipos de operaciones mediante Execution Plan metadata, no inferencia de SQL.

---

# 725. Principle of no query result assumptions for DDL

DDL puede devolver semántica de resultado diferente.

Execution Result deberá modelar variantes adecuadamente.

---

# 726. Principle of result types explicitness

Podrán existir:

```text
RowSetResult
ScalarResult
AffectedRowsResult
GeneratedValuesResult
CommandResult
CursorResult
```

según necesidad.

---

# 727. Principle of no universal mixed result array

Evitar una estructura genérica con keys opcionales para todo tipo de ejecución.

Tipos claros mejoran predictibilidad.

---

# 728. Principle of result laziness where useful

CursorResult podrá ser lazy.

BufferedResult será explícitamente materializado.

---

# 729. Principle of result buffering policy

El usuario o Planner podrá elegir buffering cuando exista trade-off entre:

```text
memory
connection occupancy
random access
```

---

# 730. Principle of no hidden buffering of huge streams

Una API llamada `stream()` no deberá materializar todo internamente.

---

# 731. Principle of connection occupancy diagnostics

Long-lived cursors podrán ser visibles porque retienen conexiones del pool.

---

# 732. Principle of configurable cursor fetch size

Cuando driver lo soporte, streaming podrá utilizar fetch size configurable.

---

# 733. Principle of platform-specific cursor semantics

No todos los motores/drivers soportan server-side cursors de la misma manera.

La capability deberá declararlo.

---

# 734. Principle of fallback transparency for streaming

Si una plataforma únicamente puede simular streaming mediante buffering parcial/completo, deberá documentarse.

---

# 735. Principle of binary/LOB streaming support extensibility

Large Objects podrán requerir APIs específicas de streaming.

No deberán forzarse como strings en memoria.

---

# 736. Principle of LOB lifecycle clarity

Un LOB stream puede depender de la conexión/transaction.

Su lifetime deberá ser explícito.

---

# 737. Principle of no serialization of live resources

Connections, cursors y LOB streams no deberán serializarse.

---

# 738. Principle of ORM serialization caution

Entidades podrán serializarse si el dominio lo permite.

Pero proxies/lazy loaders no deberán depender de contexto serializado invisible.

---

# 739. Principle of no PersistenceContext serialization

PersistenceContext, UnitOfWork e IdentityMap serán runtime state no serializable por defecto.

---

# 740. Principle of job boundaries

Cuando una entidad deba enviarse a un Job será preferible enviar:

```text
entity identifier
DTO
```

y recargar en el nuevo contexto.

No transportar entidad managed con contexto.

---

# 741. Principle of stale entity awareness across jobs

Una entidad recargada posteriormente puede haber cambiado.

La aplicación deberá usar locking/versioning si requiere detectar conflictos.

---

# 742. Principle of optimistic locking version semantics

El campo versión deberá actualizarse atómicamente en la misma operación de persistencia.

---

# 743. Principle of generated timestamp caution

Usar timestamps como versión optimistic lock puede tener limitaciones de precisión.

VoltStack deberá soportar versiones numéricas explícitas.

---

# 744. Principle of database clock vs application clock explicitness

Timestamps generados por database y aplicación pueden diferir.

Metadata deberá definir quién genera el valor.

---

# 745. Principle of generated column read-only semantics

Columnas generadas/computed no deberán tratarse como propiedades escribibles ordinarias.

---

# 746. Principle of default-value distinction

Debe distinguirse:

```text
application default
database default
null
unset
```

especialmente durante INSERT.

---

# 747. Principle of insert omission semantics

Una propiedad `unset` puede significar:

```text
omit column and let DB default
```

mientras `null` puede significar:

```text
explicit NULL
```

El Persistence Engine deberá preservar esta distinción cuando aplique.

---

# 748. Principle of dirty tracking precision

ChangeTracker deberá distinguir cambios reales.

Ejemplo:

```text
"1" vs 1
```

podrá requerir comparación según tipo lógico, no solo `!==`.

---

# 749. Principle of custom type equality

Los tipos podrán definir semántica de comparación para change tracking.

---

# 750. Principle of mutable value object caution

Value Objects usados en entidades deberían preferirse inmutables.

Si son mutables, ChangeTracker deberá poder detectar cambios internos o exigir notificación explícita.

---

# 751. Principle of collection dirty tracking

Colecciones ORM deberán rastrear:

```text
added
removed
reordered where relevant
```

sin requerir comparar grafos completos cuando sea posible.

---

# 752. Principle of large collection strategies

Relaciones enormes podrán requerir estrategias como:

```text
extra-lazy
count without loading
contains query
slice query
```

como capacidades ORM avanzadas.

---

# 753. Principle of no accidental full relation load

Operaciones como:

```text
count($entity->largeRelation)
```

no deberán forzosamente cargar millones de elementos si existe una estrategia mejor y está habilitada.

---

# 754. Principle of collection query separation

Operaciones extra-lazy deberán generar queries mediante Query Engine, no SQL desde Collection.

---

# 755. Principle of collection ownership explicitness

Las colecciones deberán conocer su owner metadata sin adquirir responsabilidad de EntityManager completo.

---

# 756. Principle of no hidden context capture in persistent objects

Objetos que puedan sobrevivir request no deberán capturar referencias a:

```text
EntityManager
Connection
RequestContext
TenantContext
```

salvo lifecycle estrictamente controlado.

---

# 757. Principle of proxy context references weak/controlled

Si lazy proxies requieren acceso a loader/context, el lifecycle deberá impedir leaks.

La implementación concreta se definirá posteriormente.

---

# 758. Principle of closed-context proxy behavior explicitness

Acceder a una relación no cargada después de cerrar PersistenceContext deberá:

```text
throw
remain unavailable
use explicit detached strategy
```

según política documentada.

No abrir un contexto global mágico.

---

# 759. Principle of no service locator in entities

Las entidades/proxies no deberán consultar el Container global para cargar relaciones.

---

# 760. Principle of dependency-free domain methods

Métodos de dominio deberán poder ejecutarse sin Database activo cuando no requieran persistencia.

---

# 761. Principle of repository dependency at application boundary

Servicios de aplicación podrán depender de Repository interfaces.

Las entidades no deberán depender del repository para su comportamiento ordinario.

---

# 762. Principle of persistence ignorance as preferred mode

Data Mapper deberá permitir entidades con mínima o nula dependencia en atributos VoltStack si mapping externo está configurado.

El uso de Attributes será una conveniencia, no requisito conceptual.

---

# 763. Principle of attributes as declarative metadata

Cuando se utilicen, Attributes deberán declarar metadata.

No ejecutar lógica compleja en constructores de Attributes.

---

# 764. Principle of attribute compilation

Attributes deberán procesarse durante metadata loading/compilation y convertirse a estructuras internas.

---

# 765. Principle of no repeated Attribute object creation in hot path

Runtime deberá consumir metadata compilada.

---

# 766. Principle of explicit inheritance mapping

Si ORM soporta herencia:

```text
single table
joined
concrete
```

deberá ser una estrategia explícita.

No inferirse automáticamente de PHP inheritance.

---

# 767. Principle of ORM inheritance restraint

No todas las jerarquías PHP deberían persistirse como jerarquías ORM.

El framework deberá permitir ignorarlas.

---

# 768. Principle of discriminator safety

Discriminator values deberán validarse y mapearse explícitamente.

---

# 769. Principle of polymorphic relation distinction

Las relaciones polimórficas estilo Active Record y la herencia ORM son conceptos distintos.

No deberán compartir implementación accidentalmente.

---

# 770. Principle of polymorphic relation integrity awareness

Las relaciones polimórficas pueden carecer de FK convencional.

El ORM deberá documentar esta menor integridad estructural.

---

# 771. Principle of native FK preference where possible

Las relaciones ordinarias deberán poder aprovechar foreign keys reales.

---

# 772. Principle of ORM cascade does not replace FK behavior

`cascade remove` y `ON DELETE CASCADE` son mecanismos distintos.

Su interacción deberá ser explícita.

---

# 773. Principle of avoiding duplicate cascade execution

Si database ejecuta cascade nativo, ORM no deberá producir operaciones redundantes que causen inconsistencias.

---

# 774. Principle of database-generated side effects awareness

Triggers pueden modificar datos.

VoltStack deberá permitir refrescar generated fields cuando sea necesario.

No intentará modelar automáticamente cualquier trigger desconocido.

---

# 775. Principle of trigger transparency limits

ORM no puede conocer todos los efectos de triggers externos.

La documentación deberá reconocer esta frontera.

---

# 776. Principle of refresh APIs

EntityManager deberá permitir sincronizar una entidad desde database cuando sea necesario.

---

# 777. Principle of refresh overwrites local state explicitly

`refresh()` deberá documentar que puede reemplazar cambios locales.

No deberá ocurrir automáticamente salvo estrategia específica.

---

# 778. Principle of lock-and-refresh composition

Operaciones que requieren lectura consistente podrán combinar locking + hydration mediante APIs explícitas.

---

# 779. Principle of aggregate versioning support

Optimistic locking podrá aplicarse a aggregate roots sin imponerlo a todas las entidades.

---

# 780. Principle of value-generation strategies as services

Los generators de ID/value deberán ser servicios/strategies especializados.

No métodos hardcoded en EntityManager.

---

# 781. Principle of deterministic application generators

UUID/ULID generators deberán poder sustituirse para tests.

---

# 782. Principle of database generator connection awareness

Sequence/identity generators que requieran DB deberán usar infraestructura mediante contratos, no PDO directo.

---

# 783. Principle of generated values belong to persistence synchronization

Asignar valores generados a entidades deberá ocurrir en una fase claramente definida de persistence.

---

# 784. Principle of no half-managed entity state

Si una inserción falla después de asignar ID generado localmente, UnitOfWork deberá conocer el estado correcto y política de rollback.

---

# 785. Principle of flush exception state documentation

Después de un `flush()` fallido deberá quedar claro si:

```text
EntityManager remains usable
must clear
transaction rolled back
entities remain managed
```

La política concreta se diseñará en UnitOfWork.

---

# 786. Principle of conservative EntityManager failure policy

Cuando no pueda garantizarse consistencia del PersistenceContext tras fallo grave, será preferible cerrarlo/marcarlo inválido.

---

# 787. Principle of new context recovery

La aplicación deberá poder crear un nuevo PersistenceContext después de invalidar uno fallido.

---

# 788. Principle of no request-wide fatal dependency when recoverable

Un EntityManager inválido no significa necesariamente que el worker entero deba morir si puede resetearse de forma segura.

---

# 789. Principle of worker recycle only as final containment

RuntimeManagerServer podrá reciclar worker cuando:

```text
resource leak
unknown global state
unrecoverable connection/runtime corruption
```

No como sustituto del cleanup normal.

---

# 790. Principle of library responsibility boundaries

Database deberá utilizar mecanismos de RuntimeManagerServer mediante contrato.

No controlar directamente el proceso del servidor.

---

# 791. Principle of no sleep/backoff blocking policy leakage

Si runtimes futuros son async, la estrategia de backoff deberá poder adaptarse.

El Query Engine no deberá llamar directamente `sleep()` como política universal.

---

# 792. Principle of clock abstraction where behavior depends on time

Retries, TTL y diagnostics podrán depender de una abstracción de clock para testing cuando se justifique.

---

# 793. Principle of random/jitter abstraction

Si resiliencia utiliza jitter, deberá poder probarse determinísticamente mediante generador inyectable.

---

# 794. Principle of no hidden randomness in query plans

El Planner no deberá seleccionar estrategias aleatorias salvo algoritmo explícito y reproducible/configurable.

---

# 795. Principle of stable defaults for reproducibility

Dos instancias iguales deberían generar planes equivalentes bajo mismas capabilities/configuración.

---

# 796. Principle of diagnostics can explain planner choices

En modo avanzado, tooling deberá poder responder:

```text
why was this strategy selected?
```

Ejemplo:

```text
secondary relation query selected
because join would load two to-many collections
```

---

# 797. Principle of no unexplained optimizer rewrites

Las reglas de optimización deberán tener nombres/IDs que puedan aparecer en diagnostics.

---

# 798. Principle of optimizer rule metrics

Podrá medirse:

```text
rule applied count
optimization duration
```

en perfiles avanzados, con overhead controlado.

---

# 799. Principle of optimizer timeout/budget

Si el optimizer se vuelve complejo en el futuro deberá poder operar con presupuesto limitado.

No gastar más tiempo optimizando que ejecutando queries simples.

---

# 800. Principle of cheap-query fast path

Consultas sencillas deberán evitar overhead excesivo.

El pipeline podrá tener fast paths siempre que preserve los mismos contratos y semántica.

---

# 801. Principle of fast paths are observable and tested

Un fast path no deberá convertirse en un segundo motor.

Debe producir resultados equivalentes a la ruta general.

---

# 802. Principle of one semantic source of truth

Aunque existan fast paths, las reglas semánticas deberán definirse una sola vez o verificarse contra una única fuente.

---

# 803. Principle of complexity budget

Cada feature deberá justificar no solo performance sino complejidad añadida.

La arquitectura deberá favorecer soluciones suficientemente simples.

---

# 804. Principle of deletion friendliness

Un componente bien desacoplado debería poder eliminarse o reemplazarse sin reescribir subsistemas no relacionados.

Este será un indicador de modularidad.

---

# 805. Principle of feature ownership

Cada característica deberá tener un único dominio propietario principal.

Otros dominios podrán colaborar mediante contratos.

---

# 806. Principle of cross-domain changes deserve scrutiny

Si una feature requiere modificar:

```text
Driver
Query
ORM
Schema
Runtime
```

simultáneamente, deberá revisarse si la abstracción está ubicada correctamente.

---

# 807. Principle of dependency graph review

Cambios importantes deberán revisar el grafo de dependencias para detectar nuevos ciclos.

---

# 808. Principle of architecture metrics as supporting tools

Podrán utilizarse métricas como:

```text
afferent coupling
efferent coupling
cycles
namespace dependencies
```

como apoyo, no como objetivo aislado.

---

# 809. Principle of no cyclic namespaces

Los namespaces principales deberán formar un DAG siempre que sea posible.

```text
Directed Acyclic Graph
```

---

# 810. Principle of dependency cycles require explicit resolution

Si aparece:

```text
A → B → C → A
```

no deberá resolverse ocultándolo mediante Container/service locator.

Deberá rediseñarse la frontera.

---

# 811. Principle of abstractions belong near policy owner

Una interface no necesariamente pertenece al proveedor.

Cuando un consumidor define lo que necesita, el contrato puede pertenecer al dominio consumidor.

Esto seguirá Dependency Inversion.

---

# 812. Principle of shared contracts only when truly shared

No toda interface deberá ir a:

```text
Database/Contracts
```

Los contratos específicos podrán vivir junto al dominio propietario.

---

# 813. Principle of no contracts dumping ground

`Contracts/` deberá permanecer organizado y limitado.

---

# 814. Principle of exception ownership

Las excepciones deberán ubicarse cerca del dominio que define su significado.

Podrá existir una raíz común:

```text
DatabaseException
```

sin centralizar todos los detalles arbitrariamente.

---

# 815. Principle of exception hierarchy shallow enough

Se evitarán jerarquías excesivamente profundas.

La composición con metadata puede ser más útil que 12 niveles de herencia.

---

# 816. Principle of errors vs exceptions

Errores de configuración/programación pueden requerir excepciones diferentes de fallos operativos temporales.

Esto ayudará a resilience.

---

# 817. Principle of no retry on programmer errors

Casos como:

```text
invalid column
invalid mapping
unsupported feature
syntax generated by bug
```

no deberán reintentarse.

---

# 818. Principle of retryable errors are contextual

Incluso `deadlock` solo es reintentable si la unidad de trabajo completa puede repetirse correctamente.

---

# 819. Principle of validation doesn't query database unless defined

La validación estructural deberá ser pura cuando sea posible.

La validación que requiere introspección deberá identificarse como tal.

---

# 820. Principle of schema-aware semantic analysis configurable

Query Semantic Analyzer podrá usar schema metadata para validación avanzada.

Pero el sistema deberá poder operar cuando schema metadata completa no esté disponible, dependiendo del modo.

---

# 821. Principle of validation confidence levels

Podrán existir:

```text
syntactic validation
metadata-informed validation
database-confirmed validation
```

sin confundir sus garantías.

---

# 822. Principle of no false certainty in tooling

Si Database no puede confirmar algo deberá mostrar:

```text
unknown
```

en lugar de afirmar incorrectamente.

---

# 823. Principle of explainability

Herramientas avanzadas deberán ser capaces de explicar:

```text
connection resolution
capability resolution
query compilation
migration risk
ORM mapping
```

de forma comprensible.

---

# 824. Principle of explain data is optional

La información usada solo para explain/debug no deberá cargarse permanentemente en hot paths cuando esté desactivada.

---

# 825. Principle of public contracts documented before stable

Una API no deberá marcarse estable sin:

```text
behavior documentation
error semantics
lifecycle
compatibility expectations
```

---

# 826. Principle of defaults documented as contracts

Un default ampliamente utilizado forma parte de comportamiento observable.

Deberá tratarse con cuidado.

---

# 827. Principle of no magic environment detection for database semantics

VoltStack no deberá cambiar de:

```text
MySQL behavior
```

a:

```text
SQLite behavior
```

solo porque detectó tests, salvo que configuración lo indique.

---

# 828. Principle of explicit test connection

Testing deberá seleccionar una conexión mediante configuración clara.

---

# 829. Principle of migration environment guards

Comandos destructivos podrán tener guardias por environment, pero con override explícito para automatización autorizada.

---

# 830. Principle of no production hostname heuristics

No decidir seguridad basándose en si host contiene:

```text
prod
localhost
```

Las políticas se configuran explícitamente.

---

# 831. Principle of deployment tool integration through APIs

Herramientas de deployment podrán consultar:

```text
migration status
schema compatibility
health
```

mediante servicios/CLI estructurados.

---

# 832. Principle of no database availability assumption for build artifacts

Compilar metadata ORM no debería requerir database si mapping contiene información suficiente.

Schema validation contra server podrá ser un paso separado.

---

# 833. Principle of offline compilation

Query/metadata/schema tooling deberá favorecer modos offline cuando sea posible.

---

# 834. Principle of online validation as explicit phase

Verificaciones que requieran server deberán ejecutarse explícitamente.

Ejemplo:

```text
db:validate --online
```

como concepto.

---

# 835. Principle of build/deploy/runtime separation

El sistema distinguirá operaciones apropiadas para:

```text
build
deploy
runtime
```

Ejemplos:

```text
metadata compile → build
migrations → deploy
queries → runtime
```

---

# 836. Principle of runtime hot path minimalism

Requests ordinarios no deberán ejecutar tareas de build/deploy accidentalmente.

---

# 837. Principle of package boot phases

Database podrá definir:

```text
register
compile
boot
runtime scope
shutdown
```

con responsabilidades distintas.

---

# 838. Principle of no connection acquisition during register

Registrar servicios deberá ser side-effect-light.

---

# 839. Principle of boot validation configurable

Boot podrá validar configuración sin necesariamente conectarse.

---

# 840. Principle of warmup extensibility

Un comando de warmup podrá preparar:

```text
metadata
query caches
platform metadata
```

si existe información suficiente.

---

# 841. Principle of warmup failure clarity

Si warmup requiere server y este no está disponible deberá informar qué artefactos pudieron/no pudieron generarse.

---

# 842. Principle of no stale warmup silent use

Los artefactos de warmup deberán invalidarse mediante firmas/versiones.

---

# 843. Principle of production cache atomics

Cuando se regeneren caches compiladas, deberá evitarse dejar archivos parcialmente escritos.

Utilizar estrategias atómicas cuando sea posible.

---

# 844. Principle of multi-worker cache coordination

La regeneración concurrente de artefactos compartidos deberá considerar locking o build externo.

---

# 845. Principle of runtime read-only caches where possible

En producción será preferible que workers lean caches preconstruidas en vez de modificarlas constantemente.

---

# 846. Principle of package filesystem independence where possible

Database core no deberá asumir que siempre puede escribir al filesystem durante requests.

---

# 847. Principle of storage abstractions only when needed

No introducir dependencia en Quantum/FileSystem para cada cache interna si PHP files/cache contracts simples son suficientes.

La integración se decidirá por subsystem.

---

# 848. Principle of modular optional integrations

La arquitectura deberá elegir el contrato más pequeño posible para cada integración.

---

# 849. Principle of no circular Quantum dependencies

Debe evitarse:

```text
Database → Cache → Database
```

o:

```text
Database → Telemetry → Database
```

Las integraciones mediante contracts/adapters deberán romper estos ciclos.

---

# 850. Principle of bootstrap dependency graph acyclic

No solo namespaces: el grafo de servicios durante bootstrap deberá evitar ciclos difíciles de resolver.

---

# 851. Principle of lazy proxy not used to hide architecture cycles

No solucionar un ciclo:

```text
A needs B
B needs A
```

insertando lazy proxies automáticamente sin revisar diseño.

---

# 852. Principle of factories not used as service locator disguise

Una factory deberá crear una categoría concreta de objetos.

No exponer:

```php
$factory->make(string $anything)
```

como Container alternativo.

---

# 853. Principle of registries return definitions or capabilities appropriately

Un registry podrá almacenar factories/definitions cuando instancias deban ser scoped.

No necesariamente instancias globales.

---

# 854. Principle of scoped service creation

Los servicios que contienen:

```text
UnitOfWork
TransactionContext
TenantContext
```

deberán crearse dentro del scope correspondiente.

---

# 855. Principle of scoped dependencies remain scoped

Un servicio singleton/process-wide no deberá capturar accidentalmente un servicio request-scoped durante bootstrap.

---

# 856. Principle of scope validation

El Container/architecture tests podrán detectar dependencias:

```text
process service
→ request service
```

cuando sean inseguras.

---

# 857. Principle of contextual resolvers for scope bridging

Si un servicio process-wide necesita trabajar con el contexto actual, deberá recibir un resolver/operation parameter, no capturar la instancia.

---

# 858. Principle of explicit operation contexts passed downward

Ejemplo:

```php
$executor->execute(
    $compiledQuery,
    $executionContext,
);
```

en vez de obtener contextos actuales mediante globals.

---

# 859. Principle of contexts immutable where possible

Aunque referencien información contextual, los objetos de contexto deberían ser inmutables durante una operación cuando sea práctico.

---

# 860. Principle of context derivation

Podrán derivarse contextos:

```text
RequestContext
 ↓
QueryContext
 ↓
ExecutionContext
```

copiando únicamente la información relevante.

---

# 861. Principle of no context bag anti-pattern

Evitar:

```text
Context {
    array $data;
}
```

como bolsa arbitraria para cualquier extensión.

Las extensiones deberán tener mecanismos tipados.

---

# 862. Principle of extensible metadata namespaces

Si contextos necesitan metadata extensible podrán utilizar keys tipadas/namespaced con límites claros.

---

# 863. Principle of no sensitive data propagation without need

Cada capa deberá recibir únicamente los datos sensibles que necesita.

---

# 864. Principle of configuration secrets resolution close to connection boundary

Query Builder no necesita contraseña DB.

Driver/Connection Factory sí.

---

# 865. Principle of privilege separation inside code architecture

Los componentes capaces de ejecutar raw SQL o DDL deberán estar claramente separados de componentes que solo construyen consultas.

---

# 866. Principle of restricted administrative services

Aplicaciones podrán registrar servicios administrativos únicamente en CLI/deployment context si se desea reducir superficie.

---

# 867. Principle of no web exposure of administrative API by default

Database no deberá crear rutas HTTP para administración automáticamente.

---

# 868. Principle of operations require explicit application exposure

Si el usuario quiere una UI administrativa deberá construirla o instalar tooling autorizado.

---

# 869. Principle of database diagnostics privacy

Incluso `db:diagnose` deberá evitar mostrar secrets.

---

# 870. Principle of sanitizable exception serialization

Si excepciones se envían a observability externa deberán poder serializarse con redaction.

---

# 871. Principle of no entity dumps in production query errors

Errores ORM no deberán serializar grafos completos de entidades por defecto.

---

# 872. Principle of entity identity diagnostics over full state

Para diagnóstico podrá bastar:

```text
EntityClass
Identifier
State
```

en lugar de todos los campos.

---

# 873. Principle of no PII assumptions

Database no puede saber automáticamente qué campos son PII.

Metadata deberá permitir marcarlos cuando sea necesario.

---

# 874. Principle of sensitive field annotations

ORM metadata podrá permitir:

```text
sensitive
redact
audit classification
```

como información transversal opcional.

---

# 875. Principle of sensitive metadata not affecting persistence semantics

Marcar un campo sensible deberá controlar diagnostics, no cambiar arbitrariamente cómo se persiste salvo configuración específica.

---

# 876. Principle of encryption integration boundaries

Field-level encryption podrá implementarse mediante Type/Value Conversion/Encryption integration.

No dentro de Driver.

---

# 877. Principle of encryption does not replace database TLS

Son capas diferentes.

VoltStack deberá tratarlas separadamente.

---

# 878. Principle of deterministic encryption caution

Si se permiten columnas cifradas consultables mediante deterministic encryption, deberá documentarse el trade-off de seguridad.

No será default.

---

# 879. Principle of key management outside Database core

Database podrá consumir un crypto/key provider mediante integración.

No administrar HSM/KMS directamente en core.

---

# 880. Principle of rotation-aware encrypted mappings

Capacidades de cifrado avanzadas podrán soportar versionado de keys mediante metadata.

---

# 881. Principle of no security-through-obscurity query transformations

Ocultar nombres de tablas o SQL no constituye control de seguridad.

---

# 882. Principle of database authorization remains database-level concern

VoltStack Authorization controla acceso de aplicación.

Los privilegios del usuario DB deberán configurarse independientemente.

---

# 883. Principle of no automatic superuser credentials

Tooling no deberá recomendar root/superuser como credencial normal de aplicación.

---

# 884. Principle of migrations may require elevated credentials

La arquitectura deberá permitir una conexión administrativa distinta.

---

# 885. Principle of explicit connection purpose

Connection configuration podrá declarar propósito/rol para evitar usar accidentalmente credenciales de migration en runtime.

---

# 886. Principle of production policy checks

Un modo estricto podrá advertir si runtime usa credenciales con privilegios excesivos cuando pueda detectarse razonablemente, sin prometer auditoría completa.

---

# 887. Principle of backward compatibility for schema representations

Cambios de Metadata/Schema Model deberán contemplar caches/migrations generados por versiones anteriores.

---

# 888. Principle of compiled artifact invalidation on framework upgrade

Las caches compiladas deberán incluir versión VoltStack relevante.

---

# 889. Principle of no PHP object unserialize for untrusted cache

Artefactos de cache no confiables no deberán deserializarse mediante mecanismos inseguros.

---

# 890. Principle of generated PHP cache safety

Si se utiliza PHP generado para metadata, deberá producirse únicamente desde fuentes confiables y almacenarse en rutas controladas.

---

# 891. Principle of cache permissions

Tooling deberá respetar permisos de filesystem seguros.

---

# 892. Principle of database driver dependency isolation

Dependencias específicas como extensiones PHP deberán verificarse al resolver driver.

Ejemplo:

```text
pdo_pgsql missing
```

deberá producir error claro.

---

# 893. Principle of no optional extension check in unrelated flows

Una aplicación SQLite no necesita verificar extensiones PostgreSQL en cada bootstrap.

---

# 894. Principle of lazy driver validation

Driver específico podrá validarse cuando se registre/configure o use, según política de bootstrap.

---

# 895. Principle of driver version awareness

Algunas extensiones/drivers pueden cambiar comportamiento por versión.

Platform/Driver descriptor podrá capturarlo cuando sea relevante.

---

# 896. Principle of database server and client capabilities distinction

Deben distinguirse:

```text
server supports feature
driver supports feature
```

Una capability efectiva puede requerir ambos.

---

# 897. Principle of effective capabilities

Conceptualmente:

```text
Effective Capability
=
Server Capability
∩
Driver Capability
∩
Framework Support
```

---

# 898. Principle of capability diagnostics

Tooling deberá poder indicar:

```text
server supports X
driver does not support X
VoltStack implementation status
```

cuando sea relevante.

---

# 899. Principle of framework capability not overclaiming

VoltStack no declarará una feature soportada solo porque el servidor la soporta si compiler/driver aún no la implementa.

---

# 900. Principle of feature support states

Podrán existir estados:

```text
supported
partially supported
experimental
unsupported
```

con detalles.

---

# 901. Principle of documentation generated from capability data where possible

Parte de matrices de soporte podrán derivarse de descriptors/tests para reducir divergencia entre docs y código.

---

# 902. Principle of implementation completeness gates

Una plataforma no deberá marcarse oficialmente soportada hasta superar su suite de conformidad.

---

# 903. Principle of no hidden platform fallback

Si una feature específica compila diferente deberá pasar por estrategia explícita.

---

# 904. Principle of SQL standards where practical

El Query Model podrá basarse en conceptos SQL estándar como baseline.

Pero no deberá sacrificar capacidades útiles de motores modernos.

---

# 905. Principle of database-native escape extensions

Podrán existir paquetes/extensiones:

```text
PostgreSQL FullText
PostgreSQL JSONB
MySQL Spatial
```

sobre las fronteras públicas.

---

# 906. Principle of extension portability declaration

Cada extensión deberá declarar si es:

```text
portable
platform-specific
version-specific
```

---

# 907. Principle of graceful unsupported extension detection

Utilizar una extensión PostgreSQL sobre MySQL deberá fallar antes de ejecutar SQL cuando sea posible.

---

# 908. Principle of query compilation does not mutate schema state

Compiler no debe hacer DDL ni actualizar metadata.

---

# 909. Principle of schema compiler does not inspect ORM state

Schema Compiler recibe Schema Plan + Platform.

No EntityManager.

---

# 910. Principle of migration planner can use schema state

Migration Planner puede comparar:

```text
desired schema
actual schema
```

pero esta introspección deberá ser explícita.

---

# 911. Principle of no automatic schema introspection per migration operation

La introspección deberá recopilarse eficientemente y compartirse dentro del plan.

---

# 912. Principle of schema snapshot

Una ejecución de migration/schema diff podrá trabajar con un snapshot consistente de metadata en vez de reconsultar después de cada pequeño paso, salvo operaciones que requieran refresh.

---

# 913. Principle of explicit schema refresh

Si una operación modifica schema y pasos posteriores requieren nuevo estado, el plan deberá incluir refresh explícito.

---

# 914. Principle of database metadata cache scope

Schema Metadata podrá cachearse por:

```text
connection
database
schema
platform version
```

con invalidación apropiada.

---

# 915. Principle of no ORM metadata invalidation on every query

El ORM metadata depende de mapping/configuración, no de cada ejecución.

---

# 916. Principle of mapping/schema validation separated from query hot path

La validación completa deberá ejecutarse durante tooling/bootstrap, no por cada `find()`.

---

# 917. Principle of fallback validation for dynamic schemas

Si una aplicación cambia schema dinámicamente, podrá optar por modos específicos, asumiendo coste.

---

# 918. Principle of stable architecture despite dynamic data

La existencia de multitenancy/schema dinámico no deberá modificar las reglas base de capas.

---

# 919. Principle of tenant-specific metadata only where necessary

Si todos los tenants comparten schema lógico, metadata ORM podrá compartirse.

No duplicarla por tenant sin necesidad.

---

# 920. Principle of tenant-specific schema signatures

Cuando schema varíe por tenant, caches deberán incluir una firma adecuada.

---

# 921. Principle of no unbounded tenant metadata caches

Cualquier cache tenant-specific deberá aplicar límites/eviction.

---

# 922. Principle of distributed cache optional for tenant metadata

Escenarios grandes podrán integrar Quantum/Cache.

No será necesario para el core simple.

---

# 923. Principle of tenant routing before connection acquisition

El Tenant Resolver deberá seleccionar target lógico antes de adquirir conexión física.

---

# 924. Principle of query semantics independent of tenant when possible

La misma Query AST podrá ejecutarse en diferentes tenant databases si schemas son compatibles.

---

# 925. Principle of tenant scopes above generic query semantics

Filtros tenant shared-database deberán introducirse mediante integración/policy antes de compilation.

No dentro del Driver.

---

# 926. Principle of tenant predicate protection

Si un modo multitenant requiere predicate obligatorio, la integración deberá poder impedir su eliminación accidental por Query Builder/Optimizer.

---

# 927. Principle of optimizer security invariants

Optimizer jamás deberá eliminar filtros marcados como:

```text
security critical
tenant isolation
```

por considerarlos redundantes sin prueba semántica válida.

---

# 928. Principle of semantic annotations

AST/Semantic nodes podrán transportar annotations internas para:

```text
security invariant
origin
optimizer restrictions
```

cuando sea necesario.

---

# 929. Principle of annotations are not arbitrary bags

Las annotations deberán estar tipadas o namespaced y documentadas.

---

# 930. Principle of optimizer proof obligation

Una regla que elimine/reordene condiciones deberá demostrar semánticamente que la transformación es válida bajo SQL semantics.

---

# 931. Principle of safe optimizer default set

Solo reglas maduras y claramente correctas estarán activadas por defecto.

---

# 932. Principle of experimental optimizer rules isolation

Reglas experimentales deberán estar opt-in y cubiertas por differential tests.

---

# 933. Principle of optimization can be disabled

Para diagnostics deberá poder deshabilitarse parte del optimizer y comparar comportamiento.

---

# 934. Principle of no difference in logical result with optimizer disabled

Salvo orden no garantizado u otros aspectos explícitos, optimizar/no optimizar deberá producir resultado lógico equivalente.

---

# 935. Principle of query plan cache invalidation

Plans que dependan de:

```text
platform capabilities
schema shape
configuration
```

deberán invalidarse cuando cambien.

---

# 936. Principle of no stale plan after platform reconnect to different server version

Si una conexión lógica puede apuntar a servidores con capabilities distintas, el cache key deberá reflejar la capability signature efectiva.

---

# 937. Principle of homogeneous pool assumption explicitness

Un pool podrá asumir servers homogéneos solo si configuración lo garantiza.

De lo contrario capability resolution deberá considerar target.

---

# 938. Principle of replica compatibility

Replicas usadas para la misma conexión lógica deberán ser compatibles con queries enviadas.

El Connection Manager deberá evitar replicas con versiones/capabilities insuficientes.

---

# 939. Principle of failover capability compatibility

Un failover target deberá cumplir capacidades requeridas por la operación.

Disponibilidad no es suficiente.

---

# 940. Principle of topology is infrastructure metadata

Información sobre:

```text
primary
replicas
regions
shards
```

pertenece a Connection/Topology infrastructure, no Query AST.

---

# 941. Principle of query routing hints, not topology knowledge

Query podrá declarar:

```text
requiresPrimary
preferReplica
regionHint
```

sin conocer hosts concretos.

---

# 942. Principle of routing policy resolves hints

Connection Resolver decide target usando hints + policy.

---

# 943. Principle of no business semantics in routing hints

Hints deberán expresar requisitos técnicos.

No reglas como:

```text
premiumCustomerUseServerA
```

Eso pertenece a aplicación/integration policy.

---

# 944. Principle of observability of policy decisions

Routing y resilience policies deberán poder explicar decisiones en diagnostics.

---

# 945. Principle of policy objects

Decisiones complejas deberán modelarse como policies/strategies en vez de condicionales dispersos.

---

# 946. Principle of policies stateless when possible

Policies compartidas deberán ser inmutables/stateless.

El contexto específico se pasa como argumento.

---

# 947. Principle of policy composition restraint

No construir un meta-policy engine universal para todas las decisiones.

Cada dominio tendrá policies apropiadas.

---

# 948. Principle of Database subsystem autonomy

Cada gran subsistema deberá poder evolucionar con mínimo conocimiento de los demás.

Ejemplo:

```text
Schema
```

puede mejorar introspección sin modificar Hydration.

---

# 949. Principle of integration tests at subsystem boundaries

Además de unit tests internos, deberán existir tests para fronteras:

```text
ORM → Query
Query → Compiler
Compiler → Executor
Executor → Connection
Migration → Schema
```

---

# 950. Principle of contracts define data ownership

Un contrato deberá aclarar si:

```text
input may be mutated
output owns resource
caller must close
service retains reference
```

especialmente para recursos.

---

# 951. Principle of immutable inputs by default

Servicios de transformación deberían recibir objetos que no modifican in-place.

Preferir retornar nueva representación cuando sea razonable.

---

# 952. Principle of explicit mutation for state machines

Servicios con lifecycle pueden mutar su propio estado, pero no objetos ajenos arbitrariamente.

---

# 953. Principle of no hidden reference retention

Un servicio process-wide no deberá retener accidentalmente:

```text
QueryContext
Entity
Result
Request
```

después de operación.

---

# 954. Principle of memory leak reviews for persistent runtime

Features que agreguen callbacks, registries dinámicos o diagnostic buffers deberán revisarse específicamente por leaks.

---

# 955. Principle of weak references only with semantic justification

WeakReference podrá ayudar en casos específicos, pero no deberá usarse para ocultar ownership confuso.

---

# 956. Principle of explicit ownership beats garbage collection assumptions

El GC de PHP es una red de seguridad.

La arquitectura deberá seguir definiendo ownership.

---

# 957. Principle of object pooling restraint

No se reutilizarán objetos PHP pequeños únicamente por supuesta performance sin benchmark.

Connection pooling es diferente porque representa recursos externos costosos.

---

# 958. Principle of immutable object allocation measurement

Si AST inmutable produce costes relevantes, se optimizará basado en profiling, no abandonando invariantes prematuramente.

---

# 959. Principle of representation specialization where measured

Podrán existir representaciones compactas internas para hot paths mientras mantengan contratos externos.

---

# 960. Principle of no premature C-extension dependency

VoltStack Database deberá funcionar en PHP estándar soportado.

Optimizaciones nativas podrán ser opcionales posteriormente.

---

# 961. Principle of developer ergonomics considered architectural

Características como:

```text
clear method names
great errors
facades
autocomplete
attributes
CLI
```

son parte del diseño, no capa cosmética.

---

# 962. Principle of no DX through global magic

La comodidad deberá implementarse sobre Container/scopes correctamente.

---

# 963. Principle of familiar defaults with VoltStack semantics

VoltStack podrá inspirarse en Laravel/Doctrine en APIs.

Pero el comportamiento final deberá definirse explícitamente por VoltStack, no depender de que el usuario conozca otro framework.

---

# 964. Principle of migration assistance from external ORMs

En el futuro tooling podrá ayudar a migrar desde:

```text
Eloquent
Doctrine
```

sin convertir sus abstracciones en core.

---

# 965. Principle of compatibility adapters remain outside canonical model

Un adaptador Eloquent-like podrá traducir APIs.

El modelo canónico sigue siendo VoltStack.

---

# 966. Principle of no permanent legacy pollution

Compatibilidad temporal deberá poder retirarse.

No diseñar nuevas capas alrededor de quirks heredados.

---

# 967. Principle of metrics as feedback loop

Telemetry y benchmarks deberán informar decisiones futuras de performance.

No rediseñar basándose únicamente en intuición.

---

# 968. Principle of production feedback privacy

La recopilación de métricas deberá respetar políticas de privacidad y configuración.

---

# 969. Principle of no telemetry required for correctness

Reiteración:

```text
Telemetry off
→ Database still correct
```

---

# 970. Principle of graceful optional integration failure

Si un exporter de telemetry falla, Database podrá registrar/degradar la observabilidad.

No deberá fallar la query por defecto.

Excepciones: sistemas explícitos de audit durable tendrán otras garantías.

---

# 971. Principle of event listener failure policy explicitness

Los listeners de eventos Database deberán definir si son:

```text
critical
best effort
after commit
```

No asumir una sola política.

---

# 972. Principle of no best-effort listener before critical commit logic

Lógica necesaria para garantizar persistencia no deberá vivir en listener best-effort.

---

# 973. Principle of transaction-aware event queues

Eventos destinados a emitirse después del commit podrán acumularse dentro del TransactionContext y descartarse en rollback.

---

# 974. Principle of no request event leakage

Las colas de eventos transaccionales deberán limpiarse al terminar scope.

---

# 975. Principle of exact semantics documented close to subsystem

Este documento establece principios generales.

Detalles como:

```text
flush rollback behavior
lazy loading proxy implementation
nested transaction semantics
```

deberán definirse en documentos especializados.

---

# 976. Principle of specialized documents may refine, not contradict

Un documento posterior podrá concretar una regla.

No deberá contradecir estos principios sin registrar una decisión arquitectónica explícita.

---

# 977. Principle of documentation hierarchy

Orden de autoridad conceptual:

```text
Project Context
      ↓
Architecture
      ↓
Design Principles
      ↓
Subsystem Architecture
      ↓
Component Specification
      ↓
Implementation Notes
```

Ante contradicción deberá revisarse el documento inferior.

---

# 978. Principle of architecture evolves deliberately

Los documentos superiores pueden cambiar.

Pero esos cambios deberán ser intencionales y propagarse a especificaciones dependientes.

---

# 979. Principle of no implementation-first architectural reversal

No deberá modificarse documentación únicamente para justificar accidentalmente una implementación ya acoplada.

La implementación deberá compararse contra principios y corregirse cuando sea necesario.

---

# 980. Principle of proof through reference implementation

Los principios deberán demostrarse mediante la implementación oficial.

Si una regla resulta impracticable, deberá revisarse formalmente, no ignorarse silenciosamente.

---

# 981. Principle of architecture review gates

Antes de fusionar subsistemas importantes deberán revisarse al menos:

```text
dependency direction
state lifecycle
runtime safety
error behavior
security
testability
performance impact
extension impact
```

---

# 982. Principle of no feature without ownership

Si el equipo no puede responder:

```text
What component owns this?
```

la feature no está suficientemente diseñada.

---

# 983. Principle of no state without lifecycle

Si no se puede responder:

```text
When is this state created?
When is it destroyed?
```

no deberá añadirse.

---

# 984. Principle of no dependency without direction

Si dos componentes se requieren mutuamente:

```text
A needs B
B needs A
```

deberá resolverse la arquitectura antes de implementar.

---

# 985. Principle of no abstraction without semantics

Una nueva interface/capa deberá representar una idea clara.

No solo reducir longitud de un archivo.

---

# 986. Principle of no optimization without benchmark

Una optimización significativa deberá tener evidencia.

---

# 987. Principle of no retry without idempotency analysis

Una política de retry deberá conocer qué se está repitiendo.

---

# 988. Principle of no cache without invalidation model

Toda cache deberá definir cómo deja de ser válida.

---

# 989. Principle of no persistent service with request state

Especialmente importante para VoltStack:

```text
Persistent Service
+
Request Mutable State
=
Architectural violation
```

salvo almacenamiento scoped explícitamente diseñado.

---

# 990. Principle of no hidden external effect

Una función que parezca transformar datos no deberá realizar I/O inesperadamente.

Ejemplo:

```text
Compiler
```

no conecta.

```text
Metadata object getter
```

no debería ejecutar query salvo API claramente lazy.

---

# 991. Principle of naming methods by effects

Operaciones con side effects deberán utilizar nombres que los comuniquen.

Ejemplo:

```text
execute
persist
flush
commit
refresh
```

---

# 992. Principle of query execution terminality

Métodos terminales serán el punto donde una query normalmente pasa de representación a efecto externo.

---

# 993. Principle of inspection is side-effect free

APIs como:

```text
toAst
toPlan
toSql
explainCompilation
```

deberán evitar ejecutar la query salvo que su nombre indique `EXPLAIN` real contra server.

---

# 994. Principle of offline explain vs server explain distinction

Deberá distinguirse:

```text
VoltStack Plan Inspection
```

de:

```text
Database EXPLAIN
```

Son capas diferentes.

---

# 995. Principle of server EXPLAIN as explicit operation

Solicitar `EXPLAIN ANALYZE` puede ejecutar la query según motor.

La API deberá advertirlo y ser explícita.

---

# 996. Principle of no unsafe EXPLAIN ANALYZE automatically

Profiler no deberá ejecutar queries de escritura nuevamente solo para obtener plan.

---

# 997. Principle of performance diagnosis respects semantics

Herramientas no deberán alterar datos durante diagnóstico salvo operación explícitamente autorizada.

---

# 998. Principle of final design balance

La arquitectura deberá equilibrar:

```text
developer experience
correctness
performance
modularity
extensibility
security
observability
runtime safety
```

Ninguno deberá dominar absolutamente a los demás sin contexto.

---

# 999. Principio fundamental final

Toda decisión de diseño dentro de `VoltStack/Quantum/Database` deberá poder justificarse mediante tres preguntas:

```text
¿Mantiene clara la responsabilidad?

¿Mantiene explícito el estado y su lifecycle?

¿Preserva la dirección de dependencias?
```

Si cualquiera de estas respuestas es negativa, la decisión deberá revisarse antes de incorporarse al sistema.

---

# 1000. Regla de diseño resumida

La arquitectura puede resumirse como:

```text
Intent
  ↓
Structured Model
  ↓
Validated Semantics
  ↓
Planned Operation
  ↓
Compiled Representation
  ↓
Controlled External Effect
```

y para ORM:

```text
Domain State
    ↓
Tracked Changes
    ↓
Persistence Plan
    ↓
Query Engine
    ↓
Controlled External Effect
```

Todo ello dentro de:

```text
Explicit Scope
Explicit State Ownership
Explicit Dependencies
Explicit Lifecycle
```

---

# 1001. Estado del documento

Este documento establece los principios de diseño generales y las restricciones que gobernarán toda la arquitectura de VoltStack Database.

Los documentos posteriores deberán utilizar estos principios para definir contratos y componentes concretos.

El siguiente documento es:

```text
03_DATABASE_DOMAIN_MODEL_AND_TERMINOLOGY.md
```

En él se formalizará el vocabulario canónico de Database y las diferencias exactas entre conceptos como:

```text
Database
Platform
Dialect
Driver
Connection
Session
Statement
Query
Query Model
AST
Semantic Query
Plan
Compiled Query
Entity
Model
Repository
PersistenceContext
EntityManager
UnitOfWork
Transaction
Schema
Migration
```

con el objetivo de impedir ambigüedades terminológicas durante la implementación.