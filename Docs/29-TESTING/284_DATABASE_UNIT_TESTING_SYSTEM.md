# 284_DATABASE_UNIT_TESTING_SYSTEM.md

# VoltStack Quantum Database
## Database Unit Testing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 284 — Database Unit Testing System  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `283_DATABASE_TESTING_ARCHITECTURE.md`  
**Siguiente documento:** `285_DATABASE_INTEGRATION_TESTING_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database Unit Testing System** de VoltStack.

Su responsabilidad es establecer cómo probar de manera:

- rápida;
- aislada;
- determinista;
- reproducible;
- precisa;
- extensible;
- independiente de infraestructura;

los componentes del subsistema:

```text
VoltStack/Quantum/Database
```

cuya semántica puede demostrarse sin ejecutar una base de datos real.

La regla central será:

> **Una prueba unitaria de VoltStack Database deberá demostrar una propiedad local del componente bajo prueba mediante entradas y dependencias controladas, sin convertir una simulación de infraestructura externa en evidencia del comportamiento real de un DBMS.**

Formalmente:

```text
Unit Test
=
Subject
+
Controlled Inputs
+
Controlled Collaborators
+
Deterministic Assertions
```

No:

```text
Unit Test
=
Fake PostgreSQL
+
Assume PostgreSQL Works
```

---

# 2. Objetivos

El sistema deberá proporcionar una base común para probar:

```text
Query AST
Expressions
Predicates
Query normalization
Query validation
Semantic analysis
Query rewrites
Query optimization rules
Logical planning
Physical planning decisions
SQL compilation
Metadata
Type conversion
Casting
Schema models
Schema diff
Migration planning
ORM state transitions
Change tracking
Hydration planning
Relationship metadata
Cache keys
Capability evaluation
Routing decisions
Security policies
Retry classification
Resource policies
```

cuando dichas operaciones puedan evaluarse sin infraestructura externa.

---

# 3. Unit Testing ≠ Database Testing completo

El sistema de pruebas Database completo comprende:

```text
Unit
Component
Integration
Conformance
Failure
Security
Performance
Runtime
```

Este documento define únicamente:

```text
Unit
```

---

# 4. Unit Test ≠ Integration Test

Una prueba deja de ser puramente unitaria cuando la propiedad evaluada depende directamente de:

```text
DBMS real
network socket
PDO connection real
filesystem externo
process externo
database server
container
cloud service
```

---

# 5. Unit Test ≠ Fake Database Test

Una prueba que utiliza una implementación falsa de `Connection` puede seguir siendo unitaria.

Pero sólo demuestra:

```text
Behavior of subject
under fake collaborator behavior
```

No demuestra:

```text
Behavior of MySQL
Behavior of MariaDB
Behavior of PostgreSQL
Behavior of SQLite
```

---

# 6. Unit Test ≠ SQL Execution Test

Ejemplo:

```php
$sql = $compiler->compile($query);
```

puede probarse unitariamente.

En cambio:

```php
$connection->execute($sql);
```

contra PostgreSQL real pertenece a integración/conformance.

---

# 7. Unidad de prueba

Una unidad no deberá definirse necesariamente como:

```text
one class
```

Puede ser una unidad semántica pequeña.

Ejemplo:

```text
AST Node
+
Normalizer
```

puede constituir una unidad cuando ambos forman una transformación pura inseparable para la propiedad probada.

---

# 8. Unit Boundary

Cada prueba deberá identificar:

```text
Subject Under Test
        │
        ├── Inputs
        ├── Collaborators
        └── Observable Outputs
```

---

# 9. Subject Under Test

Abreviado:

```text
SUT
```

representará el componente cuya propiedad se intenta demostrar.

---

# 10. Principio de aislamiento

El aislamiento deberá reducir variables externas.

No significa necesariamente:

```text
mock everything
```

---

# 11. Preferencia por objetos reales

Cuando un colaborador sea:

```text
small
deterministic
pure
cheap
stable
```

se preferirá utilizar su implementación real.

---

# 12. Mock only where useful

No se creará un mock simplemente porque exista una interfaz.

Incorrecto:

```text
QueryAst
 ├── mock Expression
 ├── mock Identifier
 ├── mock Predicate
 └── mock Parameter
```

si todos esos objetos son value objects puros.

---

# 13. Isolation strategy

La estrategia preferida será:

```text
Real Value Objects
+
Real Pure Components
+
Test Doubles only at architectural boundaries
```

---

# 14. Fronteras apropiadas para doubles

Ejemplos:

```text
Connection
Driver
Clock
Random Source
Capability Provider
Secret Provider
Telemetry Sink
Event Dispatcher
Filesystem Adapter
External Process Runner
```

---

# 15. Unit Testing Architecture

```text
                     Unit Test
                         │
                         ▼
                 Subject Under Test
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
         Real Value    Fake/Stub     Spy
          Objects     Collaborator   Observer
             │           │           │
             └───────────┼───────────┘
                         ▼
                 Observable Result
                         │
                         ▼
                     Assertion
```

---

# 16. No infrastructure provisioning

Unit tests no deberán necesitar:

```text
Docker
database server
network
external daemon
```

---

# 17. Fast execution

El conjunto unitario deberá poder ejecutarse frecuentemente durante desarrollo.

Objetivo conceptual:

```text
edit
 ↓
unit tests
 ↓
feedback
```

---

# 18. Speed ≠ correctness compromise

La rapidez nunca justificará reemplazar una prueba de integración necesaria por una unitaria insuficiente.

---

# 19. Determinismo

Para entradas iguales:

```text
Test(Input, Context)
```

deberá producir el mismo resultado observable salvo que la propiedad bajo prueba sea explícitamente probabilística.

---

# 20. Controlled clock

Componentes que dependan del tiempo deberán recibir:

```text
Clock
```

cuando arquitectónicamente corresponda.

Ejemplo:

```php
$clock = new FrozenClock(
    Instant::parse('2030-01-01T12:00:00Z')
);
```

---

# 21. Controlled randomness

Componentes aleatorios deberán utilizar:

```text
SeededRandomSource
```

o equivalente.

---

# 22. No ambient global state

Las pruebas no deberán depender de:

```text
global timezone
global locale
static current tenant
static current connection
previous test
execution order
machine hostname
developer environment
```

salvo que dicha variable sea explícitamente el objeto de la prueba.

---

# 23. TestContext

Podrá existir:

```php
final readonly class DatabaseUnitTestContext
{
    public function __construct(
        public TestSeed $seed,
        public Clock $clock,
        public PlatformDescriptor $platform,
        public CapabilitySnapshot $capabilities,
    ) {}
}
```

---

# 24. Unit context ≠ runtime context

El contexto de test proporciona dependencias controladas.

No deberá convertirse en una copia mutable del `DatabaseContext` productivo.

---

# 25. Unit test categories

VoltStack podrá organizar pruebas unitarias en:

```text
Unit/
├── Query/
├── Compiler/
├── Schema/
├── Migration/
├── ORM/
├── Hydration/
├── Relationship/
├── Type/
├── Transaction/
├── Connection/
├── Cache/
├── Capability/
├── Security/
├── Resilience/
├── Distribution/
├── Backup/
├── Maintenance/
└── Administration/
```

---

# 26. Query AST Unit Testing

Los AST son candidatos prioritarios para unit testing.

Ejemplo:

```php
$query = SelectQuery::from('users')
    ->select('id', 'name')
    ->where(
        Eq::of(
            Column::of('active'),
            Parameter::named('active')
        )
    );
```

La prueba puede verificar su estructura semántica.

---

# 27. AST assertions

Preferible:

```php
$this->assertInstanceOf(SelectQuery::class, $query);
$this->assertCount(2, $query->projection());
$this->assertInstanceOf(Eq::class, $query->predicate());
```

sobre depender únicamente de un snapshot textual.

---

# 28. AST immutability

Deberá comprobarse cuando el contrato lo requiera:

```php
$query2 = $query1->where($predicate);

$this->assertNotSame($query1, $query2);
```

---

# 29. AST transformation tests

Patrón:

```text
Input AST
   ↓
Transformation
   ↓
Expected AST
```

---

# 30. Query Normalization

Propiedades importantes:

```text
deterministic
idempotent
semantics preserving
canonical where possible
```

---

# 31. Idempotencia

Si la normalización es idempotente:

```text
N(N(Q)) = N(Q)
```

deberá probarse.

---

# 32. Semantic equivalence

Una transformación no deberá alterar intencionalmente:

```text
meaning(Q)
```

salvo que el contrato indique una transformación semántica.

---

# 33. Query Validation

Deberán probarse:

```text
valid query
invalid query
missing required data
invalid operator
incompatible expression
invalid identifier
invalid type
```

---

# 34. Negative testing

Una parte importante del unit testing será comprobar lo que el sistema rechaza.

Ejemplo:

```php
$this->expectException(InvalidQueryException::class);
```

---

# 35. Error specificity

Cuando sea parte del contrato, deberá comprobarse:

```text
exception type
error code
semantic path
diagnostic metadata
```

---

# 36. Error message caution

No deberán crearse pruebas excesivamente frágiles contra textos completos de error salvo que sean API pública.

---

# 37. Semantic Analysis

Podrán probarse unitariamente:

```text
symbol resolution
type inference
relation resolution
constraint analysis
semantic graph generation
```

mediante metadata controlada.

---

# 38. Fake schema metadata

Para estas pruebas podrá utilizarse:

```php
$schema = TestSchema::define([
    'users' => [
        'id' => 'integer',
        'email' => 'string',
    ],
]);
```

---

# 39. Fake metadata ≠ real introspection

La prueba demuestra:

```text
semantic analyzer behavior
```

no:

```text
PostgreSQL introspection behavior
```

---

# 40. Symbol resolution tests

Casos:

```text
known column
unknown column
ambiguous column
qualified column
alias
subquery scope
CTE scope
```

---

# 41. Type inference tests

Ejemplo:

```text
integer + integer → integer
date comparison date → boolean
invalid JSON operation → error
```

según el Type System definido.

---

# 42. Query Optimizer Unit Testing

Cada regla deberá poder probarse aisladamente.

---

# 43. Optimization rule contract

Conceptualmente:

```php
interface QueryOptimizationRule
{
    public function matches(
        QueryNode $node,
        OptimizationContext $context
    ): bool;

    public function apply(
        QueryNode $node,
        OptimizationContext $context
    ): QueryNode;
}
```

---

# 44. Positive optimizer test

```text
Preconditions satisfied
        ↓
Rule fires
        ↓
Expected transformation
```

---

# 45. Negative optimizer test

```text
Preconditions not satisfied
        ↓
Rule does NOT fire
```

---

# 46. Optimization fixed point

Si el pipeline busca un punto fijo:

```text
Optimize(Optimize(Q))
=
Optimize(Q)
```

deberá comprobarse donde aplique.

---

# 47. Optimization loops

Las pruebas deberán detectar:

```text
Rule A
 Q1 → Q2

Rule B
 Q2 → Q1
```

cuando el sistema tenga protección contra ciclos.

---

# 48. Query Planner Unit Testing

El Planner podrá probarse mediante:

```text
Query Model
+
Capability Snapshot
+
Planning Policy
```

sin conexión real.

---

# 49. Capability-driven plan

Ejemplo:

```text
Requirement:
INSERT returning generated id

Capability:
returning.insert = SUPPORTED
```

Resultado esperado:

```text
NativeReturningPlan
```

---

# 50. Alternative plan

Si:

```text
returning.insert = UNSUPPORTED
```

podrá seleccionarse otra estrategia sólo cuando preserve la semántica requerida.

---

# 51. UNKNOWN capability

Debe probarse explícitamente.

```text
UNKNOWN
≠
SUPPORTED
```

---

# 52. Compiler Unit Testing

El compilador es uno de los principales candidatos.

Patrón:

```text
Validated Plan
      ↓
Compiler
      ↓
CompiledStatement
```

---

# 53. Compiler output

Podrá contener:

```text
SQL
bindings
binding types
metadata
capability dependencies
```

---

# 54. Compiler tests por plataforma

Podrán existir:

```text
MySQLCompilerTest
MariaDBCompilerTest
PostgreSQLCompilerTest
SQLiteCompilerTest
```

---

# 55. Compiler unit test limitation

Compilar:

```sql
SELECT ...
```

correctamente no demuestra que el DBMS real lo acepte.

---

# 56. SQL semantic assertions

Cuando sea posible se preferirá verificar:

```text
statement type
placeholder count
binding order
quoted identifiers
dialect feature
```

además del string SQL.

---

# 57. Golden SQL tests

Podrán utilizarse cuando sean útiles.

Ejemplo:

```php
$this->assertSame(
    'SELECT "id" FROM "users" WHERE "active" = $1',
    $compiled->sql()
);
```

---

# 58. Golden SQL fragility

Cambios equivalentes como:

```text
whitespace
formatting
parentheses redundantes
```

no deberán romper innecesariamente tests semánticos.

---

# 59. Parameter binding unit tests

Se probará la construcción de:

```text
ParameterBinding
```

sin necesitar ejecutar el statement.

---

# 60. Binding properties

```text
name/index
logical type
converted value
driver binding intent
nullable state
```

---

# 61. Type System Unit Testing

Deberá cubrir:

```text
TypeRegistry
conversion
casting
enum mapping
value objects
JSON
temporal values
custom types
```

---

# 62. Conversion round-trip

Cuando sea matemáticamente válido:

```text
PHP Value
   ↓ toDatabase
DB Canonical
   ↓ toPHP
PHP Value'
```

y:

```text
Value' ≈ Value
```

---

# 63. Lossy conversions

No deberán exigir igualdad perfecta cuando el tipo declara pérdida controlada.

Ejemplo:

```text
timestamp precision
decimal scale normalization
```

---

# 64. NULL tests

Siempre deberán existir casos para:

```text
NULL
non-NULL
invalid NULL
```

---

# 65. JSON semantics

Unit tests deberán distinguir:

```text
SQL NULL
JSON null
missing JSON path
```

cuando la abstracción lo modele.

---

# 66. Temporal types

Deberán probar:

```text
Instant
LocalDate
LocalTime
LocalDateTime
ZonedDateTime
```

sin intercambiarlos.

---

# 67. Timezone tests

Casos relevantes:

```text
UTC
positive offset
negative offset
DST boundary
```

cuando el tipo lo requiera.

---

# 68. Enum tests

Deberán comprobar:

```text
stable backing representation
unknown value
invalid mapping
nullable enum
```

---

# 69. Value Object Mapping

Se probará:

```text
construction
decomposition
conversion
nullability
invalid representation
```

---

# 70. TypeRegistry tests

Deberán verificar:

```text
registration
duplicate ID
lookup
freeze
unknown type
extension registration
```

---

# 71. Frozen registry

Después de freeze:

```php
$registry->freeze();
```

la mutación deberá rechazarse.

---

# 72. Metadata Unit Testing

Aplica a:

```text
EntityMetadata
RelationshipMetadata
SchemaMetadata
TypeMetadata
QueryMetadata
```

---

# 73. Metadata compilation

Entrada:

```text
attributes/configuration
```

Salida:

```text
immutable compiled metadata
```

---

# 74. Metadata determinism

Misma definición:

```text
D
```

deberá producir metadata semánticamente equivalente.

---

# 75. Metadata invalid definitions

Casos:

```text
duplicate field
missing identifier
invalid relationship
unknown type
invalid mapped column
contradictory options
```

---

# 76. ORM Unit Testing

No todo el ORM requiere base real.

---

# 77. EntityState tests

Las transiciones de estado podrán probarse de forma aislada.

```text
NEW
 ↓ persist
MANAGED
 ↓ change
DIRTY
 ↓ remove
REMOVED
```

---

# 78. State machine

Transiciones inválidas deberán rechazarse o producir el resultado explícitamente definido.

---

# 79. IdentityMap Unit Testing

Puede probarse sin DB real.

---

# 80. IdentityMap invariant

Dentro del mismo scope:

```text
(EntityType, Identifier, EffectiveContext)
```

deberá mapear a una única instancia managed.

---

# 81. Identity key

Deberá incluir contexto suficiente.

Por ejemplo:

```text
entity type
identifier
tenant/database identity
```

cuando aplique.

---

# 82. Cross-tenant identity

Nunca:

```text
Tenant A / User 1
=
Tenant B / User 1
```

---

# 83. IdentityMap lifecycle

Pruebas:

```text
register
lookup
duplicate
detach
clear
reset
```

---

# 84. UnitOfWork Unit Testing

Podrá utilizar:

```text
entities
metadata
identity map
change detector
fake persistence planner
```

---

# 85. ChangeSet tests

Ejemplo:

```text
Original:
name = "Alice"

Current:
name = "Bob"

Expected:
name: "Alice" → "Bob"
```

---

# 86. No-change test

```text
snapshot == current
```

deberá producir:

```text
empty changeset
```

---

# 87. ChangeSet ≠ SQL

Las pruebas del UoW no deberán exigir SQL directamente.

---

# 88. Persistence planning

El UoW produce información semántica.

Después:

```text
Persistence Planner
```

decide operaciones.

---

# 89. persist() unit test

Debe demostrar:

```text
entity registered/scheduled
```

No:

```text
INSERT happened
```

---

# 90. remove() unit test

Debe demostrar:

```text
removal scheduled
```

No:

```text
DELETE happened
```

---

# 91. flush orchestration

Puede probarse con spies/fakes para verificar:

```text
change collection
planning
execution request
state reconciliation
```

---

# 92. flush() ≠ commit()

La prueba nunca deberá esperar `commit()` automáticamente salvo una API explícita que sea propietaria de la transacción.

---

# 93. Hydration Unit Testing

La transformación:

```text
Result Row
   ↓
Hydration Plan
   ↓
Object/Scalar/Tuple
```

puede probarse sin DB.

---

# 94. Entity hydrator tests

Casos:

```text
new entity
existing IdentityMap entity
nullable field
partial field set
embedded/value object
enum
temporal
```

---

# 95. Existing managed entity

La hidratación deberá reutilizarla según las reglas del ORM.

---

# 96. Dirty managed entity

No deberá sobrescribirse silenciosamente.

---

# 97. LoadedFieldMask

Deberán probarse:

```text
selected field
unselected field
NULL selected field
```

---

# 98. NULL ≠ unloaded

Invariante crítico:

```text
NULL value
≠
field not selected
```

---

# 99. Tuple hydration

Ejemplo:

```text
[UserEntity, scalarCount]
```

deberá conservar shape y tipos.

---

# 100. Hydration plan cache

Unit tests deberán verificar que:

```text
plan cached
```

no significa:

```text
entity cached
```

---

# 101. Relationship Metadata Tests

Se probarán:

```text
one-to-one
one-to-many
many-to-one
many-to-many
polymorphic
```

---

# 102. Owning side

Las reglas para determinar:

```text
owning
inverse
```

deberán tener tests unitarios.

---

# 103. Cascade configuration

Casos inválidos deberán detectarse.

---

# 104. Polymorphic alias

Se deberá probar:

```text
stable alias → EntityType
EntityType → stable alias
```

---

# 105. FQCN persisted

Deberá rechazarse si la política central prohíbe usar FQCN como discriminator persistido.

---

# 106. Relationship Loading Planner

Puede probarse con:

```text
metadata
requested relationship
capabilities
load policy
```

---

# 107. Eager strategy selection

Ejemplo:

```text
JOIN
SELECT_IN
BATCH
```

según contexto.

---

# 108. Lazy policy

Casos:

```text
ALLOW
WARN
FORBID
```

---

# 109. Detached lazy load

Deberá probar que no aparece mágicamente un EntityManager global.

---

# 110. N+1 Detection Unit Tests

Se podrán alimentar secuencias sintéticas de operaciones.

Ejemplo:

```text
load Users
load Orders for User 1
load Orders for User 2
load Orders for User 3
```

Esperado:

```text
N+1 candidate
```

---

# 111. N+1 confidence

Deberán probarse distintos niveles de evidencia.

---

# 112. False positives

Casos similares pero semánticamente diferentes deberán formar parte de los tests.

---

# 113. Schema Model Unit Testing

Podrán probarse:

```text
TableDefinition
ColumnDefinition
IndexDefinition
ForeignKeyDefinition
ConstraintDefinition
```

sin DB real.

---

# 114. Schema AST

Operaciones:

```text
CreateTable
DropTable
AddColumn
AlterColumn
DropColumn
CreateIndex
```

son candidatos unitarios.

---

# 115. Schema Diff

Una de las áreas más importantes para property-based testing.

---

# 116. Diff identity

```text
Diff(A, A)
=
∅
```

---

# 117. Diff direction

```text
Diff(A, B)
≠
Diff(B, A)
```

en general.

---

# 118. Unknown metadata

Deberá probarse:

```text
not observed
≠
absent
```

---

# 119. Rename inference

Los casos ambiguos deberán permanecer ambiguos si la política así lo establece.

---

# 120. Schema Compiler

Puede probarse unitariamente de forma similar al Query Compiler.

---

# 121. Migration Unit Testing

Componentes apropiados:

```text
Migration Discovery
Migration Graph
Dependency Resolution
Migration Planner
Batch Planner
Safety Classifier
```

---

# 122. Migration ordering

Si:

```text
A depends B
B depends C
```

el orden deberá respetar:

```text
C → B → A
```

---

# 123. Migration cycle

```text
A → B
B → A
```

deberá detectarse.

---

# 124. Migration Safety

Entrada:

```text
Migration Plan
+
Schema Metadata
+
Policy
+
Evidence
```

Salida:

```text
Safety Decision
```

---

# 125. UNKNOWN safety

Debe permanecer:

```text
UNKNOWN
```

cuando la evidencia sea insuficiente.

---

# 126. Safety ≠ Capability

Tests deberán evitar mezclar:

```text
can execute
```

con:

```text
safe to execute now
```

---

# 127. Transaction Unit Testing

Las semánticas reales de transacción requieren integración.

Pero componentes puros sí pueden probarse.

---

# 128. Transaction state machine

Ejemplo:

```text
IDLE
 ↓ begin
ACTIVE
 ↓ commit
COMMITTED
```

o:

```text
ACTIVE
 ↓ rollback
ROLLED_BACK
```

---

# 129. Invalid transition

Ejemplo:

```text
COMMITTED
 ↓ commit again
```

deberá producir comportamiento definido.

---

# 130. Transaction retry policy

Puede probarse unitariamente.

Entrada:

```text
failure classification
attempt
idempotency
transaction outcome
```

Salida:

```text
RETRY
DO_NOT_RETRY
UNKNOWN_REQUIRES_RECONCILIATION
```

---

# 131. Unknown commit

Invariante:

```text
UNKNOWN commit
```

nunca deberá clasificarse automáticamente como:

```text
safe retry
```

---

# 132. Deadlock classification

El mapper de errores podrá probarse con errores sintéticos.

Pero el deadlock real requiere integración.

---

# 133. Connection Unit Testing

Connection Manager y resolución pueden probarse parcialmente con fakes.

---

# 134. Connection resolution

Ejemplo:

```text
read intent
+
tenant
+
shard
+
transaction
+
topology
```

produce:

```text
connection target
```

---

# 135. No network

Esta prueba no necesita abrir la conexión.

---

# 136. Connection state machine

Estados y reset planning podrán probarse unitariamente.

---

# 137. Connection protocol

El protocolo real pertenece a integration/conformance.

---

# 138. Read/Write Routing Unit Testing

Casos:

```text
read → replica candidate
write → writer
locking read → writer
active transaction → pinned endpoint
sticky read → writer
```

---

# 139. Replica eligibility

Podrán suministrarse replicas sintéticas:

```text
R1 healthy/fresh
R2 healthy/stale
R3 unhealthy
```

y verificar candidatos.

---

# 140. Health ≠ eligibility

La prueba deberá demostrarlo.

---

# 141. Load Balancing

Algoritmos deterministas podrán probarse con un random source controlado.

---

# 142. Failover planning

Puede probarse con topologías sintéticas.

---

# 143. Mid-transaction failover

El planner deberá rechazar estrategias que pretendan mover transparentemente una transacción activa cuando el contrato lo prohíba.

---

# 144. Sharding Unit Testing

El partition router puede probarse sin shards reales.

---

# 145. Shard key

Ejemplo:

```text
tenantId = 105
```

deberá resolver:

```text
Shard 3
```

según el mapa suministrado.

---

# 146. Missing shard key

Una escritura ordinaria que requiera shard único deberá rechazarse si no puede resolverse.

---

# 147. UNKNOWN ≠ global

Debe existir prueba explícita.

---

# 148. Topology generation

Un plan asociado a:

```text
generation 17
```

no deberá reutilizarse indebidamente bajo:

```text
generation 18
```

---

# 149. Cache Unit Testing

Podrán probarse:

```text
cache key
policy
consistency decision
invalidation planning
TTL calculations
generation matching
```

---

# 150. Cache key determinism

Misma semántica:

```text
same semantic inputs
```

deberá producir misma key.

---

# 151. Tenant cache isolation

Keys de tenants diferentes deberán diferir cuando la información sea tenant-scoped.

---

# 152. Cache consistency

Entrada:

```text
cached entry
+
requested consistency
+
context
```

produce:

```text
USABLE
STALE
REJECTED
UNKNOWN
```

---

# 153. Physical hit

Debe existir prueba:

```text
cache contains key
```

pero política responde:

```text
REJECTED
```

---

# 154. Invalidation planning

Commit, rollback y UNKNOWN deberán producir comportamientos diferenciados.

---

# 155. Capability System Unit Testing

Será una de las suites unitarias principales.

---

# 156. Capability Resolver

Entrada:

```text
CapabilityDescriptor
Evidence[]
Constraints
Dependencies
Policy
```

Salida:

```text
CapabilityResult
```

---

# 157. Evidence ordering

La precedencia deberá ser determinista.

---

# 158. Runtime evidence

Cuando la política lo establezca, evidencia directa podrá prevalecer sobre inferencia por versión.

---

# 159. Conflict

Ejemplo:

```text
ServerVersion evidence:
SUPPORTED

RuntimeProbe:
UNAVAILABLE
```

deberá producir el estado definido por el resolver.

---

# 160. Capability dependency graph

Deberá probar:

```text
dependencies
missing dependency
cycles
```

---

# 161. Capability requirement composition

Casos:

```text
ALL_OF
ANY_OF
NONE_OF
```

---

# 162. Capability criticality

Tests para:

```text
CORRECTNESS
SAFETY
SEMANTICS
OPTIMIZATION_ONLY
```

---

# 163. UNKNOWN policy

```text
REJECT
PROBE_IF_SAFE
REQUIRE_EXPLICIT_OVERRIDE
```

deberá probarse.

---

# 164. Probe execution

El probe real no pertenece necesariamente a esta suite.

La decisión sobre si se permite probar sí puede evaluarse unitariamente.

---

# 165. Version parsing

Debe tener cobertura extensa.

Ejemplos:

```text
8.0.40
8.4
15.8
16.3
17
```

---

# 166. Version comparison

Nunca:

```text
"10.0" < "9.0"
```

por comparación lexicográfica.

---

# 167. Security Unit Testing

Componentes puros de seguridad deberán probarse extensamente.

---

# 168. Identifier validation

Casos:

```text
users
user_name
schema.users
malicious identifier
empty identifier
reserved identifier
```

según contrato.

---

# 169. Parameterization

El compiler deberá demostrar que valores externos se convierten en bindings y no en concatenación SQL.

---

# 170. Raw expressions

Deberán existir tests que demuestren que:

```text
RawExpression
```

requiere el mecanismo explícito definido.

---

# 171. Sensitive data redaction

Entrada:

```text
password
token
connection string
binding classified sensitive
```

Salida de diagnóstico:

```text
[REDACTED]
```

o equivalente.

---

# 172. Credential tests

Podrán verificar:

```text
SecretReference
```

sin utilizar secretos reales.

---

# 173. Audit event unit tests

Deberán comprobar:

```text
semantic operation
actor reference
resource
outcome
redaction
```

sin requerir sink externo.

---

# 174. Resilience Unit Testing

Ideal para:

```text
retry policy
failure classification
circuit breaker state
resource policy
timeout policy
```

---

# 175. Retry tests

Tabla conceptual:

| Failure | Idempotent | Outcome | Expected |
|---|---:|---|---|
| connection refused before execution | yes | NOT_STARTED | RETRY |
| deadlock | yes | ROLLED_BACK | RETRY_TRANSACTION |
| timeout after unknown execution | no | UNKNOWN | DO_NOT_BLIND_RETRY |
| commit acknowledgement lost | any | UNKNOWN | RECONCILE |

---

# 176. Retry attempt limits

Deberán probarse:

```text
attempt 1
attempt N
max attempts
backoff
budget exhausted
```

---

# 177. Backoff

El tiempo deberá calcularse con fuentes controladas.

No deberá hacer dormir realmente a la suite unitaria.

---

# 178. Circuit Breaker Unit Testing

Estados:

```text
CLOSED
 ↓ failures
OPEN
 ↓ timeout
HALF_OPEN
 ↓ success
CLOSED
```

---

# 179. Circuit breaker clock

Utilizará un clock falso/controlado.

---

# 180. Resource Governance

Puede probar:

```text
budget accounting
admission decision
quota policy
memory estimates
concurrency slots
```

---

# 181. Resource exhaustion

No será necesario agotar RAM real para probar el policy engine.

---

# 182. Backup Unit Testing

La arquitectura de backup contiene múltiples componentes unit-testable:

```text
Backup Planner
Strategy Selector
Manifest Builder
Artifact Naming
Consistency Policy
Dependency Graph
Retention Planner
```

---

# 183. Backup manifest

Deberá probar:

```text
backup id
source
scope
strategy
consistency
checksum metadata
parent backup
platform
capability fingerprint
```

---

# 184. Incremental dependency graph

Ejemplo:

```text
Full A
 ├── Incremental B
 │    └── Incremental C
```

el planner no deberá permitir eliminar `A` dejando `B/C` inválidos sin estrategia explícita.

---

# 185. Backup Completed

El state machine deberá impedir:

```text
CAPTURING → COMPLETED
```

si el contrato exige finalización/verificación previa.

---

# 186. Restore Unit Testing

Podrán probarse:

```text
manifest validation
compatibility checks
restore planning
dependency chain
target validation
safety policy
```

---

# 187. Restore real

La restauración real será integración.

---

# 188. Maintenance Unit Testing

Podrán probarse:

```text
maintenance planner
operation classification
capability checks
resource policy
safety decisions
```

---

# 189. Health Check Unit Testing

Health aggregation puede probarse mediante probes falsos.

Ejemplo:

```text
connection = HEALTHY
writer = HEALTHY
replica = DEGRADED
storage = HEALTHY
```

produce el estado agregado definido.

---

# 190. Health fake probe

No demuestra que una conexión real esté saludable.

---

# 191. Diagnostics Unit Testing

Se probará:

```text
evidence aggregation
diagnostic rules
explanation generation
redaction
confidence
```

---

# 192. Diagnostics determinism

La misma evidencia deberá producir diagnóstico equivalente.

---

# 193. Administration Unit Testing

Podrán probarse:

```text
authorization requirement
operation planning
safety classification
approval requirements
capability validation
```

---

# 194. Admin execution

La ejecución destructiva real pertenecerá a integration/system.

---

# 195. Test Double Architecture

VoltStack utilizará categorías explícitas.

```text
TestDouble
├── Stub
├── Fake
├── Mock
├── Spy
└── FaultStub
```

---

# 196. Stub

Ejemplo:

```php
final class StubCapabilityProvider implements CapabilityProvider
{
    public function evidence(...): array
    {
        return $this->evidence;
    }
}
```

---

# 197. Fake

Ejemplo:

```text
InMemoryMetadataRepository
```

que implementa comportamiento funcional limitado.

---

# 198. Mock

Útil para comprobar interacciones contractuales específicas.

---

# 199. Spy

Ejemplo:

```text
SpyEventDispatcher
SpyTelemetrySink
SpyConnection
```

---

# 200. Fault Stub

Devuelve fallos controlados.

Ejemplo:

```text
throw ConnectionLostException
on third call
```

---

# 201. Double semantics

Cada double deberá declarar su alcance.

---

# 202. No FakePostgreSQL

Se evitará diseñar una clase general:

```text
FakePostgreSQL
```

que pretenda reproducir completamente el DBMS.

---

# 203. Narrow fake

Preferible:

```text
FakeConnection
FakeSchemaMetadataProvider
FakeCapabilityProvider
```

con contratos limitados.

---

# 204. Interaction Testing

No deberá abusarse de:

```text
called exactly once
```

cuando la cantidad de llamadas no sea parte del contrato.

---

# 205. Behavioral assertion

Preferible probar resultado observable.

---

# 206. Interaction assertion

Se utilizará cuando la interacción sea la semántica.

Ejemplo:

```text
sensitive telemetry must never receive raw password
```

---

# 207. Property-Based Unit Testing

VoltStack deberá facilitarlo especialmente en componentes matemáticos/estructurales.

---

# 208. Candidates

```text
AST transformations
normalization
schema diff
type conversions
identifier handling
cache keys
capability graphs
version comparison
```

---

# 209. Normalization property

```text
N(N(x)) = N(x)
```

---

# 210. Schema property

```text
Diff(S, S) = ∅
```

---

# 211. Ordering property

Para un comparador válido:

```text
sign(compare(a,b))
=
-sign(compare(b,a))
```

cuando aplique.

---

# 212. Cache key property

Si dos contextos son semánticamente distintos en un atributo relevante:

```text
Relevant(A) ≠ Relevant(B)
```

entonces idealmente:

```text
Key(A) ≠ Key(B)
```

sin convertir esto en una afirmación criptográfica absoluta.

---

# 213. Generated test cases

Deberán conservar seed.

---

# 214. Shrinking

Cuando la librería utilizada lo permita, deberá conservarse el input mínimo que reproduce el fallo.

---

# 215. Mutation Testing

Podrá utilizarse para evaluar calidad de tests unitarios.

---

# 216. Example mutation

Código:

```php
if ($attempt >= $maxAttempts)
```

Mutación:

```php
if ($attempt > $maxAttempts)
```

La suite debería detectar el cambio si el boundary está correctamente probado.

---

# 217. Mutation score

Será una señal auxiliar.

No será garantía absoluta de calidad.

---

# 218. Boundary Value Testing

Especialmente importante en:

```text
limits
timeouts
batch sizes
identifier lengths
parameter counts
versions
numeric types
pagination
```

---

# 219. Boundary pattern

Probar:

```text
min - 1
min
min + 1

max - 1
max
max + 1
```

cuando tenga sentido.

---

# 220. Equivalence partitions

No es necesario probar todos los valores si pertenecen a la misma clase semántica.

---

# 221. Unit Test Naming

Convención recomendada:

```text
<subject>_<condition>_<expected_behavior>
```

Ejemplo:

```text
capability_resolver_unknown_required_capability_is_rejected
```

---

# 222. Alternative descriptive style

También:

```php
public function testUnknownRequiredCapabilityIsRejected(): void
```

---

# 223. Test names as documentation

El nombre deberá expresar:

```text
condition
+
expected behavior
```

---

# 224. AAA pattern

Cuando sea útil:

```text
Arrange
Act
Assert
```

---

# 225. Given/When/Then

También válido:

```text
Given
When
Then
```

---

# 226. One assertion myth

VoltStack no impondrá:

```text
one PHPUnit assertion per test
```

La unidad lógica de prueba será una propiedad/comportamiento.

---

# 227. One behavior

Preferencia:

```text
one coherent behavior per test
```

aunque requiera varias assertions.

---

# 228. Test helper discipline

Helpers deberán reducir ruido, no esconder la propiedad probada.

---

# 229. Bad helper

```php
$this->makeEverythingWork();
```

---

# 230. Better helper

```php
$metadata = $this->userMetadata();
```

---

# 231. Fixture builders

Para objetos complejos podrán existir:

```text
QueryTestBuilder
SchemaTestBuilder
EntityMetadataTestBuilder
CapabilitySnapshotBuilder
TopologyTestBuilder
```

---

# 232. Safe defaults

Los builders de test tendrán defaults mínimos y explícitos.

---

# 233. Builder override

Ejemplo:

```php
$capabilities = CapabilitySnapshotBuilder::new()
    ->support('query.returning')
    ->unsupported('query.merge')
    ->build();
```

---

# 234. Test builders ≠ production builders

No deberán convertirse accidentalmente en API productiva.

---

# 235. Snapshot Testing

Podrá utilizarse selectivamente para:

```text
large AST
compiled metadata
diagnostic trees
plans
```

---

# 236. Snapshot canonicalization

Antes de snapshot deberán eliminarse elementos no deterministas:

```text
memory addresses
random IDs
timestamps
absolute paths
```

si no son parte del contrato.

---

# 237. Snapshot review

Un snapshot actualizado automáticamente sin revisión reduce su valor.

---

# 238. Structural assertions

Para invariantes críticos se preferirán assertions explícitas además del snapshot.

---

# 239. Architecture Unit Tests

VoltStack podrá inspeccionar dependencias entre namespaces.

---

# 240. Example invariant

```text
Quantum\Database\ORM
        ↓ allowed
Quantum\Database\Query
```

pero:

```text
Quantum\Database\Query
        ↓ forbidden
Quantum\Database\ORM
```

---

# 241. Compiler invariant

```text
Compiler
```

no deberá depender de:

```text
Executor
Connection
PDO
EntityManager
```

---

# 242. Query Builder invariant

No deberá conocer:

```text
PDO
Driver
```

---

# 243. Connection invariant

No deberá conocer:

```text
ORM
```

---

# 244. ORM SQL invariant

El ORM no deberá generar SQL directamente.

---

# 245. Static state invariant

Componentes operation-scoped no deberán usar static mutable state.

---

# 246. Dependency tests

Podrán analizar:

```text
namespace dependencies
constructor types
imports
reflection metadata
```

---

# 247. Architecture tests ≠ runtime tests

No sustituyen las pruebas funcionales.

---

# 248. Error Testing

Cada componente deberá probar:

```text
expected failure
invalid state
invalid input
unsupported operation
unknown evidence
```

---

# 249. Exception hierarchy

Se comprobará cuando forme parte del contrato.

---

# 250. Exception context

Podrá verificarse:

```text
operation
entity
query fingerprint
capability
platform
```

sin incluir datos sensibles.

---

# 251. Redaction

Errores de test deberán confirmar que:

```text
password
token
secret
sensitive parameter
```

no aparecen.

---

# 252. Unit Test Performance

Aunque no sean benchmarks, los unit tests deberán permanecer razonablemente rápidos.

---

# 253. No sleep

Queda prohibido usar:

```php
sleep();
usleep();
```

para esperar comportamiento interno cuando pueda utilizarse un clock controlado.

---

# 254. No network

Unit suite no deberá depender de Internet.

---

# 255. No real DB

La suite marcada `Unit` no deberá requerir un servidor DB.

---

# 256. SQLite caveat

Incluso SQLite real introduce comportamiento de DB.

Una prueba que ejecuta SQL contra SQLite será normalmente:

```text
integration test
```

aunque sea rápida.

---

# 257. In-memory ≠ unit automatically

Regla:

> **Que una dependencia se ejecute en memoria no convierte automáticamente una prueba en unitaria.**

---

# 258. Filesystem

Un filesystem virtual/fake puede usarse unitariamente.

Un filesystem real temporal puede ser component/integration según la propiedad.

---

# 259. Process runner

Se utilizará un fake para unit tests de backup/maintenance que normalmente invoquen herramientas externas.

---

# 260. Command construction tests

Puede probarse:

```text
BackupPlan
 ↓
Provider
 ↓
ProcessInvocation
```

sin ejecutar el proceso.

---

# 261. Secret command-line safety

Deberá comprobarse que secretos no se incorporen indebidamente a argumentos visibles cuando la estrategia los prohíba.

---

# 262. Event Testing

Un `SpyEventDispatcher` podrá capturar eventos.

---

# 263. Event assertions

```text
event emitted
event not emitted
event order
event payload
```

cuando sean parte del contrato.

---

# 264. Telemetry Unit Testing

Un:

```text
InMemoryTelemetrySink
```

podrá recibir spans/metrics.

---

# 265. Telemetry semantic independence

Ejecutar con:

```text
NullTelemetry
```

y:

```text
SpyTelemetry
```

deberá producir el mismo resultado funcional.

---

# 266. Logging

Las pruebas no deberán depender de logs para determinar correctness salvo que el log sea el producto bajo prueba.

---

# 267. Test-specific APIs

El código productivo podrá exponer contratos que mejoren testabilidad si también mejoran diseño.

Pero no deberá llenarse de:

```text
if testing
getInternalStateForTest()
```

---

# 268. Observable behavior

Preferencia:

```text
public contract
```

sobre inspección arbitraria de privados.

---

# 269. Internal component tests

Como VoltStack prueba su propio framework, componentes internos estables también podrán probarse directamente.

---

# 270. Reflection

Se evitará usar reflection únicamente para acceder a privados si la misma propiedad puede demostrarse mediante contrato observable.

---

# 271. Test coverage model

La suite unitaria medirá más que líneas.

---

# 272. Unit coverage dimensions

```text
behavior coverage
branch coverage
error coverage
boundary coverage
invariant coverage
type coverage
capability-state coverage
```

---

# 273. Capability-state coverage

Para decisiones críticas:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EXTENSION
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

deberán considerarse.

---

# 274. State-machine coverage

Para componentes con estados:

```text
states
transitions
invalid transitions
terminal states
```

---

# 275. Failure classification coverage

Cada categoría de fallo deberá tener casos representativos.

---

# 276. Test suite partitioning

La suite podrá dividirse:

```text
unit-fast
unit-full
unit-property
unit-mutation
```

sin alterar la semántica de clasificación.

---

# 277. CI

En cada commit se espera ejecutar como mínimo:

```text
Static Architecture Tests
        +
Unit Tests
```

antes de suites más costosas.

---

# 278. Parallelization

Unit tests deberán ser altamente paralelizables.

---

# 279. No shared mutable state

Esto será requisito para paralelización segura.

---

# 280. Global registries

Cuando existan registries productivos congelados:

```text
TypeRegistry
CapabilityRegistry
MetadataRegistry
```

cada test deberá construir su instancia o fixture inmutable controlada.

---

# 281. Registry reset anti-pattern

Evitar:

```php
GlobalRegistry::resetForTests();
```

si esto implica un singleton mutable productivo.

---

# 282. Prefer instance lifecycle

```text
$registry = new TypeRegistry();
```

por prueba/suite cuando sea apropiado.

---

# 283. Test order randomization

La suite podrá ejecutar tests en orden aleatorio para descubrir dependencias ocultas.

---

# 284. Random order seed

El orden deberá ser reproducible mediante seed.

---

# 285. Flaky unit tests

Un unit test puro y determinista que sea flaky representa una señal particularmente grave.

---

# 286. Flaky investigation

Causas típicas:

```text
global state
time
randomness
parallelism
unordered collection assumptions
filesystem
locale
timezone
```

---

# 287. No retry as solution

No se solucionará un unit test flaky añadiendo reintentos permanentes.

---

# 288. Platform descriptors

Los unit tests podrán simular plataformas mediante:

```text
PlatformDescriptor
CapabilitySnapshot
DialectDefinition
```

---

# 289. Simulated platform ≠ platform conformance

Debe quedar claro en nombres/reportes.

---

# 290. Example test

```php
public function testPlannerUsesNativeReturningWhenCapabilityExists(): void
{
    $capabilities = CapabilitySnapshotBuilder::new()
        ->support('query.returning.insert')
        ->build();

    $planner = $this->planner($capabilities);

    $plan = $planner->plan(
        InsertQueryBuilder::into('users')
            ->values(['name' => 'Alice'])
            ->returning('id')
            ->build()
    );

    $this->assertInstanceOf(
        NativeReturningInsertPlan::class,
        $plan
    );
}
```

Esto demuestra:

```text
Planner decision
```

No demuestra:

```text
PostgreSQL RETURNING actually works
```

---

# 291. Complementary integration test

Posteriormente:

```text
real PostgreSQL
     ↓
INSERT ... RETURNING
     ↓
actual execution
```

demostrará la segunda propiedad.

---

# 292. Test traceability

Cada invariante crítico podrá asociarse con pruebas.

Ejemplo:

```text
DB-TYPE-...
     ↓
TypeRegistryTest
```

---

# 293. Unit invariant catalog

Podrá mantenerse:

```text
Invariant
  ↓
Unit Test
  ↓
Integration Test if required
```

---

# 294. Unit-testable ≠ sufficiently tested

Una característica puede tener excelente cobertura unitaria y seguir necesitando:

```text
integration
conformance
failure
performance
```

---

# 295. Example: Transaction Retry

Unit:

```text
Does policy classify deadlock as retryable?
```

Integration:

```text
Does actual DB deadlock map correctly?
```

System:

```text
Does transaction replay preserve application semantics?
```

---

# 296. Example: Compiler

Unit:

```text
Does AST compile to expected dialect representation?
```

Integration:

```text
Does DB accept it?
```

Conformance:

```text
Does behavior satisfy VoltStack contract on every supported platform?
```

---

# 297. Example: Backup

Unit:

```text
Does planner choose a consistent strategy?
```

Integration:

```text
Can provider produce artifact?
```

Restore:

```text
Can artifact restore correctly?
```

---

# 298. Example: ORM

Unit:

```text
Does change tracking detect mutation?
```

Integration:

```text
Does flush persist correct database state?
```

---

# 299. Unit Testing Public Support API

VoltStack podrá ofrecer herramientas como:

```php
DatabaseTest::query();
DatabaseTest::schema();
DatabaseTest::metadata();
DatabaseTest::capabilities();
DatabaseTest::topology();
```

para pruebas internas y de extensiones.

---

# 300. Public vs internal testing API

Se distinguirá:

```text
Framework Internal Test API
```

de:

```text
Application Developer Test API
```

---

# 301. Semantic versioning

Sólo APIs de testing declaradas públicas estarán sujetas a compatibilidad pública.

---

# 302. Extension testing

Plugins podrán reutilizar:

```text
Test Builders
Contract Tests
Capability Fixtures
Compiler Assertions
```

---

# 303. Custom dialect

Un plugin de dialecto podrá ejecutar primero:

```text
Dialect Unit Contract
```

antes de conformance real.

---

# 304. Custom type

Podrá ejecutar:

```text
Type Contract Suite
```

con valores:

```text
valid
invalid
null
boundary
round-trip
```

---

# 305. Custom optimizer rule

Podrá reutilizar:

```text
OptimizationRuleContract
```

---

# 306. Proposed namespaces

```text
src/Quantum/Database/Testing/Unit/
├── Contract/
│   ├── UnitTestSubject.php
│   ├── UnitTestDouble.php
│   └── UnitContractSuite.php
│
├── Context/
│   └── DatabaseUnitTestContext.php
│
├── Builder/
│   ├── QueryTestBuilder.php
│   ├── SchemaTestBuilder.php
│   ├── MetadataTestBuilder.php
│   ├── CapabilitySnapshotBuilder.php
│   └── TopologyTestBuilder.php
│
├── Double/
│   ├── Stub/
│   ├── Fake/
│   ├── Mock/
│   ├── Spy/
│   └── Fault/
│
├── Assertion/
│   ├── AstAssertions.php
│   ├── MetadataAssertions.php
│   ├── PlanAssertions.php
│   ├── CapabilityAssertions.php
│   └── ArchitectureAssertions.php
│
├── Property/
│   ├── PropertyTestCase.php
│   ├── Generator.php
│   └── Shrinker.php
│
└── Support/
    ├── FrozenClock.php
    ├── SeededRandomSource.php
    └── CanonicalSnapshot.php
```

---

# 307. Test directory proposal

```text
tests/Quantum/Database/Unit/
├── Query/
│   ├── Ast/
│   ├── Normalization/
│   ├── Validation/
│   ├── Semantic/
│   ├── Optimization/
│   └── Planning/
│
├── Compiler/
├── Schema/
├── Migration/
├── ORM/
├── Hydration/
├── Relationship/
├── Type/
├── Transaction/
├── Connection/
├── Distribution/
├── Cache/
├── Capability/
├── Security/
├── Resilience/
├── Backup/
├── Restore/
├── Maintenance/
├── Health/
├── Diagnostics/
├── Administration/
└── Architecture/
```

---

# 308. Unit Test Result

Conceptualmente:

```php
final readonly class UnitTestEvidence
{
    public function __construct(
        public TestId $testId,
        public UnitTestProperty $property,
        public TestStatus $status,
        public ?TestSeed $seed,
        public ?FailureEvidence $failure,
    ) {}
}
```

---

# 309. No fake platform evidence

No deberá contener:

```text
PostgreSQL verified
```

si sólo se utilizó:

```text
PostgreSQLCapabilityFixture
```

---

# 310. Evidence labeling

Ejemplo correcto:

```text
PostgreSQL compiler logic:
PASS
using PostgreSQL capability fixture
```

No:

```text
PostgreSQL:
PASS
```

---

# 311. Core invariants

## DB-UNIT-001

Unit Testing no será Integration Testing.

## DB-UNIT-002

Unit Testing no será Conformance Testing.

## DB-UNIT-003

Fake Database no será Real Database.

## DB-UNIT-004

Simulated Platform no será Platform Evidence.

## DB-UNIT-005

Unit tests no requerirán DB server.

## DB-UNIT-006

Unit tests no requerirán network.

## DB-UNIT-007

Unit tests no requerirán Docker.

## DB-UNIT-008

SQLite real no será automáticamente unit testing.

## DB-UNIT-009

In-memory no implicará unit automáticamente.

## DB-UNIT-010

Unit tests deberán ser deterministas por defecto.

## DB-UNIT-011

Time será controlable cuando afecte la propiedad.

## DB-UNIT-012

Randomness será reproducible.

## DB-UNIT-013

Global mutable state no determinará resultados.

## DB-UNIT-014

Test order no será dependencia.

## DB-UNIT-015

Mocks no se utilizarán por defecto para value objects puros.

## DB-UNIT-016

Real pure components serán preferidos a mocks innecesarios.

## DB-UNIT-017

Test doubles se concentrarán en fronteras arquitectónicas.

## DB-UNIT-018

Double limitations serán explícitas.

## DB-UNIT-019

FakePostgreSQL no pretenderá reproducir PostgreSQL completo.

## DB-UNIT-020

AST tendrá pruebas estructurales.

## DB-UNIT-021

AST immutability será probada donde sea contrato.

## DB-UNIT-022

Normalization idempotence será probada donde corresponda.

## DB-UNIT-023

Validation tendrá positive y negative tests.

## DB-UNIT-024

Semantic resolution utilizará metadata controlada.

## DB-UNIT-025

Fake metadata no será introspection evidence.

## DB-UNIT-026

Optimizer rules tendrán positive tests.

## DB-UNIT-027

Optimizer rules tendrán negative tests.

## DB-UNIT-028

Optimizer cycles serán detectables.

## DB-UNIT-029

Planner tendrá capability-aware tests.

## DB-UNIT-030

UNKNOWN capability será probado explícitamente.

## DB-UNIT-031

Compiler tests no serán execution tests.

## DB-UNIT-032

Golden SQL no demostrará DB compatibility.

## DB-UNIT-033

Binding type será parte de los tests.

## DB-UNIT-034

Type conversion tendrá null tests.

## DB-UNIT-035

SQL NULL no será JSON null.

## DB-UNIT-036

JSON null no será missing path.

## DB-UNIT-037

Temporal types permanecerán diferenciados.

## DB-UNIT-038

Enum mapping no dependerá de ordinal implícito.

## DB-UNIT-039

TypeRegistry freeze será probado.

## DB-UNIT-040

Metadata compilation será determinista.

## DB-UNIT-041

Invalid metadata tendrá tests.

## DB-UNIT-042

EntityState transitions serán probadas.

## DB-UNIT-043

IdentityMap será scope-aware.

## DB-UNIT-044

Tenant identity formará parte de identity cuando corresponda.

## DB-UNIT-045

UnitOfWork ChangeSet no será SQL.

## DB-UNIT-046

persist no significará INSERT.

## DB-UNIT-047

remove no significará DELETE.

## DB-UNIT-048

flush no significará commit.

## DB-UNIT-049

Hydration podrá probarse sin DB.

## DB-UNIT-050

Hydration respetará IdentityMap.

## DB-UNIT-051

Dirty managed entity no será sobrescrita silenciosamente.

## DB-UNIT-052

NULL no será unloaded field.

## DB-UNIT-053

Hydration plan cache no será entity cache.

## DB-UNIT-054

Relationship ownership será probado.

## DB-UNIT-055

Polymorphic aliases serán estables.

## DB-UNIT-056

Lazy loading policy tendrá tests.

## DB-UNIT-057

Detached entity no resolverá manager global mágicamente.

## DB-UNIT-058

N+1 detector tendrá false-positive tests.

## DB-UNIT-059

Schema Diff(A,A) será vacío.

## DB-UNIT-060

Schema diff será direccional.

## DB-UNIT-061

NotObserved no será Absent.

## DB-UNIT-062

Ambiguous rename no será inventado.

## DB-UNIT-063

Migration dependencies tendrán ordering tests.

## DB-UNIT-064

Migration cycles serán detectados.

## DB-UNIT-065

Migration Safety no será Capability.

## DB-UNIT-066

UNKNOWN safety permanecerá UNKNOWN.

## DB-UNIT-067

Real transaction isolation no se afirmará desde unit tests.

## DB-UNIT-068

Transaction state machine sí podrá probarse unitariamente.

## DB-UNIT-069

Retry policy podrá probarse unitariamente.

## DB-UNIT-070

UNKNOWN commit no será safe retry automático.

## DB-UNIT-071

Synthetic deadlock error no será real deadlock evidence.

## DB-UNIT-072

Connection routing podrá probarse sin abrir conexión.

## DB-UNIT-073

Connection protocol real requerirá integración.

## DB-UNIT-074

Locking read deberá rutear al writer cuando la política lo requiera.

## DB-UNIT-075

Transaction affinity será probada.

## DB-UNIT-076

Replica health no será eligibility.

## DB-UNIT-077

Load balancing randomness será controlable.

## DB-UNIT-078

Mid-transaction transparent failover será rechazado.

## DB-UNIT-079

Shard routing será determinista bajo mapa estable.

## DB-UNIT-080

UNKNOWN shard no será GLOBAL.

## DB-UNIT-081

Single-shard write deberá resolver shard único.

## DB-UNIT-082

Topology generation afectará planes cuando corresponda.

## DB-UNIT-083

Cache keys serán deterministas.

## DB-UNIT-084

Tenant-scoped cache keys estarán aisladas.

## DB-UNIT-085

Physical cache hit no será usable hit.

## DB-UNIT-086

Cache consistency tendrá tests.

## DB-UNIT-087

Commit, rollback y UNKNOWN producirán invalidation decisions explícitas.

## DB-UNIT-088

Capability evidence será distinto de decision.

## DB-UNIT-089

Capability confidence será distinto de status.

## DB-UNIT-090

Capability dependency cycles serán detectados.

## DB-UNIT-091

Capability requirements compuestos serán probados.

## DB-UNIT-092

Version será evidence, no capability.

## DB-UNIT-093

Version comparison no será lexicográfica.

## DB-UNIT-094

Identifier validation tendrá malicious cases.

## DB-UNIT-095

Values externos deberán convertirse en bindings.

## DB-UNIT-096

Raw expressions serán escape hatch explícito.

## DB-UNIT-097

Sensitive data tendrá redaction tests.

## DB-UNIT-098

Retry limits tendrán boundary tests.

## DB-UNIT-099

Backoff unit tests no dormirán realmente.

## DB-UNIT-100

Circuit breaker utilizará clock controlable.

## DB-UNIT-101

Resource governance no necesitará agotar recursos reales.

## DB-UNIT-102

Backup planning podrá probarse unitariamente.

## DB-UNIT-103

Backup execution real no será unit test.

## DB-UNIT-104

Restore planning podrá probarse unitariamente.

## DB-UNIT-105

Restore real no será unit test.

## DB-UNIT-106

Health aggregation podrá utilizar fake probes.

## DB-UNIT-107

Fake health probe no será real health evidence.

## DB-UNIT-108

Diagnostics rules serán deterministas.

## DB-UNIT-109

Administration planning podrá probarse sin operación destructiva.

## DB-UNIT-110

Test doubles no definirán producción.

## DB-UNIT-111

Interaction assertions se utilizarán sólo cuando interacción sea contrato.

## DB-UNIT-112

Observable behavior será preferido a implementación interna.

## DB-UNIT-113

Property-based testing complementará example-based testing.

## DB-UNIT-114

Property test failures conservarán seed.

## DB-UNIT-115

Mutation testing será señal auxiliar.

## DB-UNIT-116

Boundary conditions tendrán pruebas explícitas.

## DB-UNIT-117

Test helpers no ocultarán la propiedad evaluada.

## DB-UNIT-118

Snapshots serán canonicalizados.

## DB-UNIT-119

Snapshots no sustituirán assertions críticas.

## DB-UNIT-120

Architecture dependencies tendrán tests.

## DB-UNIT-121

ORM no generará SQL directamente.

## DB-UNIT-122

Compiler no dependerá de Executor.

## DB-UNIT-123

Compiler no dependerá de Connection.

## DB-UNIT-124

Query Builder no dependerá de Driver.

## DB-UNIT-125

Connection no dependerá de ORM.

## DB-UNIT-126

Operation-scoped state no será static mutable.

## DB-UNIT-127

Exception testing distinguirá tipo y contexto.

## DB-UNIT-128

Error diagnostics no expondrán secretos.

## DB-UNIT-129

Unit suite no utilizará sleeps para clocks controlables.

## DB-UNIT-130

Unit suite no dependerá de Internet.

## DB-UNIT-131

External process execution será fakeada en unit tests.

## DB-UNIT-132

Command generation no implicará command execution.

## DB-UNIT-133

Event spies no sustituirán Event System integration tests.

## DB-UNIT-134

Telemetry no cambiará semantics.

## DB-UNIT-135

Test-only APIs no introducirán ramas productivas artificiales.

## DB-UNIT-136

Reflection privada se evitará cuando exista comportamiento observable.

## DB-UNIT-137

Line coverage no será behavior coverage.

## DB-UNIT-138

State-machine coverage incluirá invalid transitions.

## DB-UNIT-139

Capability-state coverage incluirá UNKNOWN.

## DB-UNIT-140

Unit tests serán paralelizables por defecto.

## DB-UNIT-141

Parallel unit tests no compartirán mutable global state.

## DB-UNIT-142

Registries de test serán instancias controladas.

## DB-UNIT-143

GlobalRegistry::resetForTests no será diseño preferido.

## DB-UNIT-144

Test order randomization será reproducible.

## DB-UNIT-145

Flaky deterministic unit test será considerado defecto.

## DB-UNIT-146

Retries no serán solución permanente para flaky unit tests.

## DB-UNIT-147

Platform fixtures serán etiquetadas como simulación.

## DB-UNIT-148

Simulated PostgreSQL capability no será PostgreSQL conformance.

## DB-UNIT-149

Unit evidence tendrá alcance explícito.

## DB-UNIT-150

Unit-testable no significará sufficiently tested.

## DB-UNIT-151

Una característica podrá requerir unit + integration + conformance.

## DB-UNIT-152

Public testing APIs estarán separadas de internal testing APIs.

## DB-UNIT-153

Extension authors podrán reutilizar contract suites.

## DB-UNIT-154

Custom types tendrán reusable unit contracts.

## DB-UNIT-155

Custom optimizer rules tendrán reusable unit contracts.

## DB-UNIT-156

Custom dialect unit tests no sustituirán conformance real.

## DB-UNIT-157

Unit Test Context no será runtime mutable global context.

## DB-UNIT-158

No se introducirán vendor conditionals cuando capability fixtures sean suficientes.

## DB-UNIT-159

Failure classifications serán exhaustivamente probadas.

## DB-UNIT-160

Los tests unitarios demostrarán únicamente las propiedades que su modelo puede observar.

---

# 312. Modelo formal

Sea:

```text
S = Subject Under Test
I = Controlled Input
C = Controlled Collaborators
O = Observed Output
P = Property
```

Una prueba unitaria evalúa:

```text
O = S(I, C)
```

y comprueba:

```text
P(O, I, C) = true
```

---

# 313. Alcance de evidencia

Si:

```text
C = FakeConnection
```

la evidencia obtenida es:

```text
Behavior(S | FakeConnection)
```

No:

```text
Behavior(S | PostgreSQL)
```

---

# 314. Complementariedad

La confianza de una funcionalidad podrá expresarse conceptualmente:

```text
Feature Confidence
=
Unit Evidence
+
Integration Evidence
+
Conformance Evidence
+
Failure Evidence
+
Performance Evidence
```

según las propiedades relevantes.

---

# 315. Ejemplo completo — Capability Planner

## Arrange

```php
$capabilities = CapabilitySnapshotBuilder::new()
    ->support('query.returning.insert')
    ->unsupported('query.merge')
    ->build();
```

## Act

```php
$plan = $planner->plan($insert);
```

## Assert

```php
$this->assertInstanceOf(
    NativeReturningInsertPlan::class,
    $plan
);
```

Demuestra:

```text
Planner selects native RETURNING
when capability says supported.
```

No demuestra:

```text
Actual server implements RETURNING correctly.
```

---

# 316. Ejemplo completo — IdentityMap

```php
$map = new IdentityMap();

$user = new User();

$map->register(
    EntityIdentity::of(
        User::class,
        10,
        DatabaseContextId::of('main')
    ),
    $user
);

$result = $map->find(
    EntityIdentity::of(
        User::class,
        10,
        DatabaseContextId::of('main')
    )
);

$this->assertSame($user, $result);
```

La prueba demuestra:

```text
canonical in-scope identity
```

sin necesitar base real.

---

# 317. Ejemplo completo — UNKNOWN transaction outcome

```php
$failure = TransactionFailure::unknownCommitOutcome();

$decision = $retryPolicy->decide(
    failure: $failure,
    attempt: 1,
    operation: OperationSemantics::nonIdempotent(),
);

$this->assertSame(
    RetryDecision::RECONCILIATION_REQUIRED,
    $decision
);
```

Invariante demostrado:

```text
UNKNOWN
≠
SAFE_TO_RETRY
```

---

# 318. Ejemplo completo — Schema Diff

```php
$schema = SchemaTestBuilder::new()
    ->table('users', function ($table) {
        $table->integer('id')->primary();
        $table->string('email');
    })
    ->build();

$diff = $differ->diff($schema, $schema);

$this->assertTrue($diff->isEmpty());
```

Demuestra:

```text
Diff(S, S) = ∅
```

---

# 319. Ejemplo completo — Sensitive Data

```php
$error = QueryFailure::from(
    query: $query,
    bindings: [
        Binding::sensitive('password', 'super-secret'),
    ],
);

$output = $diagnosticRenderer->render($error);

$this->assertStringNotContainsString(
    'super-secret',
    $output
);
```

---

# 320. Arquitectura consolidada

```text
                  DATABASE UNIT TESTING
                           │
                  Controlled Test Context
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
    Real Pure          Test Doubles       Test Data
    Components         at Boundaries      Builders
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                   Subject Under Test
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
       Result            Error            Events
          │                │                 │
          └────────────────┼─────────────────┘
                           ▼
                       Assertions
                           │
                           ▼
                     Unit Evidence
                           │
                           ▼
                 Explicit Evidence Scope
```

---

# 321. Regla final

> **El Unit Testing System de VoltStack Database deberá aislar y demostrar con alta precisión toda lógica que no necesite infraestructura externa, utilizando componentes reales siempre que sean deterministas y doubles sólo en fronteras justificadas. Una simulación podrá demostrar la reacción de VoltStack ante un comportamiento controlado, pero nunca será utilizada como prueba de que MySQL, MariaDB, PostgreSQL, SQLite, un driver, una red o un sistema externo se comportan realmente de esa manera.**

En forma compacta:

```text
Unit Test
=
Controlled Proof of Local Behavior
```

mientras:

```text
Real Database Behavior
=
Integration / Conformance Evidence
```

---

# 322. Relación con el siguiente documento

La arquitectura definida aquí cubre:

```text
logic without external database dependency
```

El siguiente nivel deberá responder:

```text
¿Qué ocurre cuando VoltStack interactúa realmente
con conexiones, drivers y motores de base de datos?
```

Esto será responsabilidad de:

```text
285_DATABASE_INTEGRATION_TESTING_SYSTEM.md
```

---

# 323. Siguiente documento

```text
285_DATABASE_INTEGRATION_TESTING_SYSTEM.md
```

El siguiente documento definirá:

```text
Database Integration Testing
│
├── Real Database Environments
├── Driver Integration
├── Connection Integration
├── Query Execution
├── Transaction Integration
├── ORM Persistence
├── Schema Integration
├── Migration Integration
├── Cache Integration
├── Event Integration
├── Telemetry Integration
├── Backup/Restore Integration
├── Multi-Connection Testing
├── Failure Injection
├── Runtime Integration
├── Environment Isolation
├── Cleanup
└── Integration Evidence
```

manteniendo como regla:

> **Cuando la garantía dependa del comportamiento de un DBMS, driver, conexión, protocolo o recurso externo real, VoltStack deberá obtener evidencia mediante integración real y no inferirla a partir de mocks, fakes o compilación aislada.**