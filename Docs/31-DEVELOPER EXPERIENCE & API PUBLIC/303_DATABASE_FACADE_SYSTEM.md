# 303_DATABASE_FACADE_SYSTEM.md

# VoltStack Quantum Database
## Database Facade System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 303 — Database Facade System  
**Bloque:** 31 — Developer Experience / Public API  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `302_DATABASE_PUBLIC_API_SYSTEM.md`  
**Siguiente documento:** `304_DATABASE_HELPER_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial del **Database Facade System** de:

```text
VoltStack/Quantum/Database
```

responsable de proporcionar APIs estáticas de conveniencia para acceder a servicios de Database sin convertir esos servicios ni su estado mutable en objetos estáticos globales.

VoltStack busca permitir una experiencia como:

```php
DB::transaction(function () {
    // ...
});
```

```php
$users = DB::table('users')
    ->where('active', true)
    ->get();
```

```php
DB::connection('analytics');
```

```php
Schema::create('users', function (Table $table) {
    // ...
});
```

manteniendo intactas las reglas arquitectónicas establecidas anteriormente:

```text
Facade
≠
Service

Facade
≠
Service Locator global mutable

Facade
≠
Static Database Connection

Facade
≠
Static EntityManager

Facade
≠
Static TransactionContext

Facade
≠
Static TenantContext
```

La regla central será:

> **Una Facade de VoltStack será una superficie estática de conveniencia que resolverá dinámicamente un servicio desde el contexto de ejecución actual; nunca será propietaria estática del servicio ni de su estado mutable.**

En forma resumida:

```text
Static Syntax
≠
Static State
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. sintaxis concisa;
2. ergonomía similar a Laravel cuando resulte conveniente;
3. resolución contextual;
4. compatibilidad con FrankenPHP;
5. compatibilidad futura con RoadRunner;
6. compatibilidad futura con OpenSwoole;
7. aislamiento entre requests;
8. aislamiento entre jobs;
9. aislamiento entre operaciones;
10. aislamiento entre coroutines cuando aplique;
11. integración con Container;
12. testabilidad;
13. reemplazo controlado de servicios;
14. tipado y autocompletado;
15. errores claros;
16. ausencia de estado ORM global;
17. ausencia de conexión global mutable;
18. ausencia de contexto transaccional global;
19. soporte para múltiples conexiones;
20. soporte para extensiones;
21. compatibilidad con Multitenancy opcional;
22. compatibilidad con el sistema de capabilities;
23. mínimo uso de magia;
24. performance razonable;
25. trazabilidad de resolución.

---

# 3. Filosofía

Las Facades deberán cumplir:

```text
Facade
=
Convenience
+
Context Resolution
+
Public API Projection
```

No:

```text
Facade
=
Static Singleton
```

---

# 4. Problema de las APIs estáticas

Una API como:

```php
DB::query();
```

parece implicar que:

```text
DB
```

posee globalmente una instancia de Database.

En un modelo PHP tradicional de proceso corto, ciertos errores de diseño pueden permanecer ocultos porque:

```text
Request
→ PHP process lifecycle
→ memory discarded
```

Sin embargo, VoltStack tiene como runtime principal:

```text
FrankenPHP
```

con workers persistentes.

Por tanto:

```text
Process Lifetime
>
Request Lifetime
```

Una propiedad estática mutable puede sobrevivir entre requests.

---

# 5. Riesgo

Arquitectura incorrecta:

```php
final class DB
{
    private static ?Database $database = null;
}
```

podría producir:

```text
Worker
│
├── Request A
│   ├── Tenant A
│   ├── Transaction A
│   └── Connection A
│
└── Request B
    └── accidentally sees A state
```

Esto es inaceptable.

---

# 6. Regla fundamental

Las Facades no almacenarán:

```text
Database
Connection
EntityManager
UnitOfWork
IdentityMap
TransactionContext
DatabaseContext
TenantContext
ShardContext
QueryContext
Result
Cursor
Stream
```

como estado estático mutable persistente.

---

# 7. Arquitectura general

```text
Application
    │
    ▼
DB::transaction(...)
    │
    ▼
Database Facade
    │
    ▼
Facade Resolver
    │
    ▼
Current Execution Scope
    │
    ▼
Service Container
    │
    ▼
TransactionManager
    │
    ▼
Database Architecture
```

---

# 8. Facade ≠ Implementation

La clase:

```php
DB
```

no implementará:

```text
Query Engine
Transaction Manager
Connection Manager
ORM
Schema
```

Sólo delegará.

---

# 9. Facade Contract

Conceptualmente:

```php
abstract class Facade
{
    abstract protected static function accessor(): string;
}
```

Por ejemplo:

```php
final class DB extends Facade
{
    protected static function accessor(): string
    {
        return Database::class;
    }
}
```

---

# 10. Facade Accessor

El accessor identifica:

```text
qué servicio resolver
```

No:

```text
qué objeto estático almacenar
```

---

# 11. Resolución

Una llamada:

```php
DB::connection();
```

deberá convertirse conceptualmente en:

```text
DB
↓
FacadeRuntime
↓
CurrentScopeResolver
↓
Container
↓
Database
↓
connection()
```

---

# 12. FacadeRuntime

Podrá existir un servicio central:

```text
FacadeRuntime
```

responsable de coordinar la resolución.

---

# 13. FacadeRuntime ≠ Service Container

No reemplazará al Container.

Será un bridge entre:

```text
static facade call
```

y:

```text
current scoped container/context
```

---

# 14. CurrentScopeResolver

Será responsable de determinar el scope vigente.

Conceptualmente:

```php
interface CurrentScopeResolver
{
    public function current(): ExecutionScope;
}
```

---

# 15. ExecutionScope

Podrá representar:

```text
HTTP request
CLI command
queue job
scheduled task
test
worker operation
coroutine
manual scope
```

---

# 16. Scope Identity

Cada scope deberá poseer una identidad inequívoca.

Ejemplo:

```text
ScopeId
```

---

# 17. Scope Lifecycle

```text
CREATED
   ↓
ACTIVE
   ↓
CLOSING
   ↓
CLOSED
```

Una Facade sólo deberá resolver servicios scoped desde un scope válido.

---

# 18. No Active Scope

Si se ejecuta:

```php
DB::transaction(...)
```

fuera de un contexto válido, VoltStack deberá:

1. resolver un root-safe service si el contrato lo permite; o
2. lanzar una excepción clara.

Nunca deberá reutilizar silenciosamente el último scope conocido.

---

# 19. Critical Invariant

```text
No Current Scope
≠
Use Previous Scope
```

---

# 20. Root Services

Algunos servicios pueden ser globalmente inmutables:

```text
TypeRegistry
DriverRegistry
PlatformRegistry
CompilerRegistry
CapabilityDefinitions
```

Estos pueden residir en el root container.

---

# 21. Scoped Services

Otros deberán resolverse desde el scope:

```text
DatabaseContext
EntityManager
UnitOfWork
IdentityMap
TransactionContext
TenantContext
ConnectionLease
```

---

# 22. Facade Resolution Categories

Podrán existir:

```text
ROOT
SCOPED
CONTEXTUAL
```

---

# 23. ROOT

Servicio seguro para compartir.

Ejemplo:

```text
immutable registry
```

---

# 24. SCOPED

Una instancia por scope.

Ejemplo:

```text
EntityManager
```

---

# 25. CONTEXTUAL

La resolución depende adicionalmente de:

```text
tenant
connection
shard
read/write intent
runtime
```

---

# 26. Database Facade

La principal Facade será:

```php
DB
```

---

# 27. DB Responsibilities

Podrá exponer operaciones como:

```php
DB::connection();

DB::query();

DB::table('users');

DB::transaction(...);

DB::schema();

DB::capabilities();
```

---

# 28. DB ≠ ORM Model Facade

Las operaciones específicas de entidades deberán permanecer principalmente en:

```text
Model API
Repository
EntityManager
```

---

# 29. DB::connection()

Ejemplo:

```php
$connection = DB::connection();
```

equivale conceptualmente a:

```php
$database = $resolver->resolve(Database::class);

return $database->connection();
```

---

# 30. Named Connection

```php
$analytics = DB::connection('analytics');
```

La Facade no almacenará:

```text
current connection = analytics
```

globalmente.

---

# 31. Anti-pattern

Prohibido:

```php
DB::useConnection('analytics');

User::find(1);
```

si `useConnection()` modifica estado estático global.

---

# 32. Scoped Connection Override

Si se requiere:

```php
DB::using('analytics', function () {
    // ...
});
```

deberá implementarse mediante un contexto scoped.

---

# 33. Context Stack

Conceptualmente:

```text
Request Scope
    │
    ├── default DB context
    │
    └── temporary override
            │
            ▼
       analytics
```

Al finalizar:

```text
override removed
```

incluso ante excepciones.

---

# 34. Exception-safe Override

Conceptualmente:

```php
DB::using('analytics', function () {
    // ...
});
```

deberá operar como:

```text
push context
try
    callback
finally
    pop context
```

---

# 35. No Context Leakage

Después:

```php
DB::using('analytics', ...);
```

la conexión default anterior deberá restaurarse.

---

# 36. Nested Overrides

Podrán soportarse:

```text
default
  ↓
analytics
  ↓
archive
  ↓
analytics
  ↓
default
```

mediante stack contextual.

---

# 37. DB::table()

Ejemplo:

```php
DB::table('users');
```

deberá producir:

```text
Query Builder
```

sobre el Query Engine.

---

# 38. table() ≠ SQL

```php
DB::table('users')
```

no genera inmediatamente:

```sql
SELECT * FROM users
```

---

# 39. Query Flow

```text
DB::table('users')
       ↓
Database service
       ↓
QueryFactory
       ↓
Query Builder
       ↓
Query Model / AST
```

---

# 40. Query Execution

Sólo operaciones terminales como:

```php
->get();
->first();
->exists();
->count();
->stream();
```

inician ejecución.

---

# 41. DB::query()

Podrá proporcionar acceso a:

```text
QueryFactory
```

o iniciar un Query Builder según la API final.

---

# 42. DB::select()

Una API convenience podría existir:

```php
DB::select(...);
```

pero deberá evitar crear múltiples formas ambiguas de realizar la misma operación.

---

# 43. Prefer Builder

Para queries estructuradas se favorecerá:

```php
DB::table('users')
```

sobre SQL manual.

---

# 44. Raw Queries

Podrá existir:

```php
DB::rawQuery(
    'SELECT ... WHERE id = :id',
    ['id' => $id],
);
```

o equivalente.

---

# 45. Raw API Classification

Deberá estar claramente identificada como:

```text
advanced/escape hatch
```

---

# 46. DB::raw()

Si existe un método:

```php
DB::raw(...)
```

no deberá aceptar ambiguamente input no confiable.

---

# 47. Trusted Raw Expression

Podría requerirse:

```php
DB::trustedRaw('COUNT(*)');
```

o:

```php
RawExpression::trusted('COUNT(*)');
```

---

# 48. Raw Expression ≠ Parameter Value

Nunca será necesario hacer:

```php
DB::raw($userInput)
```

para valores.

---

# 49. DB::transaction()

Una de las APIs principales será:

```php
$result = DB::transaction(function () {
    // ...
});
```

---

# 50. Transaction Flow

```text
DB::transaction()
      ↓
Facade Resolution
      ↓
TransactionManager
      ↓
Transaction Context
      ↓
Connection
      ↓
callback
      ↓
commit / rollback / unknown
```

---

# 51. Facade ≠ Transaction Owner

`DB` sólo delega.

El propietario semántico será:

```text
TransactionManager
```

---

# 52. Nested Transactions

```php
DB::transaction(function () {
    DB::transaction(function () {
        // ...
    });
});
```

deberá seguir la política definida por Transaction System:

```text
JOIN
SAVEPOINT
REJECT
REQUIRES_NEW
```

No será decidido por la Facade.

---

# 53. flush() ≠ commit()

Una transaction Facade tampoco deberá alterar esta regla.

---

# 54. Transaction Retry

Podrá soportarse:

```php
DB::transaction(
    callback: $operation,
    retry: RetryPolicy::deadlocks(3),
);
```

---

# 55. Facade Retry ≠ Generic Retry

La Facade sólo pasa la política.

La seguridad del retry corresponde a Transaction System.

---

# 56. UNKNOWN Commit

Si el Transaction Manager determina:

```text
UNKNOWN
```

la Facade deberá preservar ese resultado/error.

Nunca:

```text
UNKNOWN
→ retry callback
```

automáticamente.

---

# 57. DB::schema()

Podrá devolver:

```text
SchemaManager
```

para la conexión/contexto actual.

---

# 58. Schema Facade

Además podrá existir:

```php
Schema
```

como Facade especializada.

---

# 59. Ejemplo

```php
Schema::create('users', function (Table $table): void {
    $table->id();
    $table->string('name');
});
```

---

# 60. Schema Facade Resolution

```text
Schema::create()
       ↓
Schema Facade
       ↓
Current Database Context
       ↓
SchemaManager
       ↓
Schema Builder
       ↓
Schema AST
```

---

# 61. Schema Facade ≠ Schema Engine

No compila SQL directamente.

---

# 62. Named Schema Connection

Podrá existir:

```php
Schema::connection('analytics')
    ->table(...);
```

pero deberá producir un objeto/contexto derivado.

---

# 63. connection() on Facade ≠ Static Mutation

No deberá hacer:

```text
Schema::$connection = analytics
```

---

# 64. Derived API

Preferible:

```text
Schema::connection('analytics')
    ↓
ConnectionBoundSchema
```

---

# 65. Facade Families

VoltStack Database podría proporcionar:

```text
DB
Schema
```

como Facades principales.

Otras sólo deberán añadirse si reducen complejidad real.

---

# 66. Avoid Facade Explosion

No será deseable tener:

```text
DB
Query
ORM
Entity
Hydrator
UnitOfWork
IdentityMap
Compiler
Planner
Driver
Dialect
Platform
```

todos como Facades.

---

# 67. Internal Components

Componentes como:

```text
Optimizer
Planner
Compiler
Hydrator
UnitOfWork
IdentityMap
```

normalmente no necesitarán Facades.

---

# 68. Model Static API

Aunque:

```php
User::query();
```

se parezca a una Facade, conceptualmente será parte del:

```text
Model API System
```

---

# 69. Shared Resolution Infrastructure

Sin embargo, podrá reutilizar:

```text
CurrentScopeResolver
DatabaseContextResolver
```

---

# 70. No Duplicate Context Infrastructure

No deberán existir:

```text
FacadeScopeResolver
ModelScopeResolver
RepositoryScopeResolver
```

con stacks incompatibles.

---

# 71. Shared Execution Context

Todos deberán converger en:

```text
ExecutionScope
```

---

# 72. Architecture

```text
                    ExecutionScope
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
             DB        Schema      Model API
              │           │           │
              └───────────┼───────────┘
                          ▼
                  Context Resolution
                          │
                          ▼
                       Container
```

---

# 73. Facade Base Architecture

Una implementación conceptual:

```php
abstract class Facade
{
    protected static function service(): string
    {
        throw new LogicException();
    }

    protected static function resolve(): object
    {
        return FacadeRuntime::resolve(
            static::service()
        );
    }
}
```

---

# 74. Important Qualification

El ejemplo anterior es conceptual.

La implementación real deberá evitar que:

```text
FacadeRuntime
```

se convierta en un static mutable service locator.

---

# 75. Bootstrapping Problem

La sintaxis estática necesita alguna forma de alcanzar el runtime actual.

Esto deberá resolverse cuidadosamente.

---

# 76. Runtime Bridge

Podrá existir un bridge mínimo:

```text
RuntimeContextBridge
```

capaz de localizar:

```text
current execution scope
```

sin almacenar directamente servicios Database.

---

# 77. Allowed Static State

Si la implementación necesita estado estático mínimo, sólo deberá contener información:

```text
immutable
process-safe
scope-resolution infrastructure
```

no servicios scoped.

---

# 78. Prefer Runtime-native Context

Cuando el runtime proporcione almacenamiento contextual, deberá aprovecharse.

Conceptualmente:

```text
FrankenPHP request context
RoadRunner operation context
OpenSwoole coroutine context
```

---

# 79. Runtime Context Adapter

```php
interface RuntimeContextAdapter
{
    public function currentScope(): ?ExecutionScope;
}
```

---

# 80. Runtime Adapters

```text
RuntimeContextAdapter
├── FrankenPhpContextAdapter
├── RoadRunnerContextAdapter
├── OpenSwooleContextAdapter
├── CliContextAdapter
└── TestContextAdapter
```

---

# 81. Facade Runtime

```text
Facade
   ↓
FacadeRuntime
   ↓
RuntimeContextAdapter
   ↓
ExecutionScope
   ↓
Container Scope
```

---

# 82. FrankenPHP

Será el runtime de referencia inicial.

---

# 83. FrankenPHP Requirement

Dos requests consecutivos en el mismo worker:

```text
Request A
Request B
```

deberán resolver:

```text
different scoped mutable services
```

---

# 84. FrankenPHP Test

Ejemplo conceptual:

```text
Request A:
DB::transaction(...)
Tenant = A

Request ends

Request B:
DB::connection()
Tenant = B
```

Debe demostrarse:

```text
no transaction A
no connection lease A
no tenant A
no EntityManager A
```

---

# 85. RoadRunner

El mismo principio aplicará al lifecycle:

```text
worker
→ job/request
→ reset
→ next operation
```

---

# 86. OpenSwoole

Será aún más estricto debido a concurrencia potencial:

```text
Coroutine A
Coroutine B
```

en el mismo worker.

---

# 87. Coroutine Isolation

Una variable estática mutable es especialmente peligrosa:

```text
Coroutine A
   ↓
DB context A

Coroutine B
   ↓
DB context B
```

No podrá existir:

```text
DB::$currentContext
```

compartido.

---

# 88. Scope-local Context

La resolución deberá ser:

```text
coroutine/request/operation-local
```

según runtime.

---

# 89. CLI

Comandos CLI también tendrán ExecutionScope.

Ejemplo:

```text
CLI command
    ↓
scope created
    ↓
DB facade usable
    ↓
scope closed/reset
```

---

# 90. Long-running CLI

Importante para:

```text
queue workers
consumers
imports
migrations
daemons
```

No deberá asumirse que:

```text
CLI = short-lived
```

---

# 91. Queue Jobs

Cada job deberá obtener su propio:

```text
DatabaseContext
EntityManager
UnitOfWork
IdentityMap
TransactionContext
```

---

# 92. Tests

El testing runtime deberá crear scopes explícitos.

---

# 93. Facade Testability

La sintaxis:

```php
DB::transaction(...)
```

deberá poder probarse sin depender de un singleton real.

---

# 94. Test Service Override

Podrá permitirse:

```php
$testScope->container()->replace(
    Database::class,
    $fakeDatabase,
);
```

Entonces:

```php
DB::connection();
```

resolverá el fake desde ese scope.

---

# 95. DB::fake()?

VoltStack deberá ser cuidadoso con APIs tipo:

```php
DB::fake();
```

porque pueden modificar estado global.

---

# 96. Preferred Testing Override

Preferible:

```text
Test Container Override
```

sobre:

```text
Facade Static Fake
```

---

# 97. Facade Mocking

Si se proporciona una API convenience de mocking, deberá modificar exclusivamente el:

```text
current test scope
```

---

# 98. Example

Conceptualmente:

```php
DB::swapForTest($fake);
```

sólo sería aceptable si significa:

```text
current test scope container override
```

y no:

```text
DB::$instance = $fake
```

---

# 99. Test Isolation

Cada test deberá comenzar sin overrides del test anterior.

---

# 100. Parallel Testing

Los overrides deberán soportar:

```text
Test Worker A
Test Worker B
```

sin colisión.

---

# 101. Facade Method Dispatch

VoltStack podrá elegir entre:

```text
explicit facade methods
```

y:

```text
dynamic __callStatic
```

---

# 102. Explicit Methods Preferred

Para APIs principales se preferirá:

```php
final class DB
{
    public static function transaction(...): mixed
    {
        // delegate
    }
}
```

porque mejora:

```text
IDE support
static analysis
documentation
signature stability
discoverability
```

---

# 103. __callStatic

Podrá existir como mecanismo limitado.

Pero:

```text
__callStatic everywhere
```

no será la estrategia principal.

---

# 104. Facade ≠ Magic Proxy Everything

La API pública debe ser visible.

---

# 105. Generated Facade Methods

Si mantener wrappers manuales resulta costoso, VoltStack podrá generar:

```text
IDE stubs
proxy metadata
```

sin convertir la API en runtime magic opaca.

---

# 106. Return Types

Las Facades deberán preservar tipos.

Ejemplo:

```php
public static function connection(
    ?string $name = null
): Connection;
```

---

# 107. Generic Return Types

Cuando sea necesario podrán utilizarse anotaciones de análisis estático.

---

# 108. Exceptions

Las Facades no deberán envolver innecesariamente todas las excepciones.

---

# 109. FacadeResolutionException

Sí deberá existir para errores propios de resolución.

Ejemplo:

```text
FacadeResolutionException
```

---

# 110. Error Conditions

Podrá producirse cuando:

```text
no runtime context
no execution scope
scope closed
service unavailable
invalid facade accessor
circular facade resolution
```

---

# 111. Preserve Database Exceptions

Si:

```text
TransactionManager
```

lanza:

```text
DeadlockException
```

DB Facade deberá preservarla.

---

# 112. No Generic FacadeException

No convertir:

```text
DeadlockException
```

en:

```text
FacadeException
```

---

# 113. Error Context

Un error de resolución podrá incluir:

```text
facade
service id
runtime
scope id
scope state
```

sin datos sensibles.

---

# 114. Facade Access Logging

No deberá loguearse cada llamada por defecto.

Sería demasiado costoso.

---

# 115. Debug Resolution Tracing

En modo debug podrá habilitarse:

```text
Facade resolution trace
```

para diagnosticar scope problems.

---

# 116. Performance

La Facade añade una capa:

```text
static call
→ scope resolution
→ container lookup
→ service call
```

---

# 117. Performance Goal

El overhead deberá ser pequeño respecto al trabajo real de Database.

---

# 118. No Expensive Reflection Per Call

Evitar:

```text
reflection
attribute scanning
service graph compilation
```

en cada llamada.

---

# 119. Compiled Accessor Metadata

La relación:

```text
DB → Database::class
```

podrá precompilarse.

---

# 120. Scope Service Cache

El Container podrá cachear:

```text
scoped service resolution
```

dentro del scope.

---

# 121. Important Distinction

```text
scope-local service cache
```

es válido.

```text
Facade static service cache
```

no.

---

# 122. Example

Correcto:

```text
Request Scope
└── Database instance
```

Incorrecto:

```text
DB::$database
└── Database instance
```

---

# 123. Contextual Connection Resolution

`DB::connection()` deberá considerar:

```text
DatabaseContext
default connection
tenant
shard
transaction affinity
read/write intent
```

según operación.

---

# 124. Facade Does Not Decide Routing

No:

```php
if ($tenant) {
    ...
}
```

dentro de `DB`.

---

# 125. Delegation

```text
Facade
→ Database Service
→ Connection Manager
→ Routing
```

---

# 126. Multitenancy

Si `Quantum/Multitenancy` está instalado:

```text
Tenant Context
```

podrá influir en resolución.

---

# 127. Core Independence

DB Facade no dependerá directamente de:

```text
Quantum\Multitenancy
```

---

# 128. Tenant Integration

Será mediante:

```text
Database Context Extension
Connection Resolver Extension
Query Context Extension
```

---

# 129. Tenant Leakage

Especialmente con Facades deberá probarse:

```text
Request Tenant A
→ DB facade

Request Tenant B
→ same worker
→ DB facade
```

sin leakage.

---

# 130. Sharding

La Facade tampoco decidirá:

```text
Shard 1
Shard 2
```

---

# 131. Replica Routing

`DB::select()` no deberá elegir directamente una replica.

---

# 132. Routing Intent

La operación se clasifica y:

```text
ReadWriteRoutingSystem
```

selecciona endpoint.

---

# 133. Sticky Connection

La Facade no mantiene:

```text
DB::$sticky = true
```

---

# 134. Sticky Context

Será scoped:

```text
DatabaseContext
```

---

# 135. Transaction Affinity

La Facade deberá resolver la transacción vigente desde el scope.

---

# 136. No Hidden Cross-transaction Connection

Dentro de:

```php
DB::transaction(function () {
    DB::table(...)->get();
});
```

la query deberá participar en la transacción correcta cuando corresponda.

---

# 137. EntityManager Integration

Aunque DB no sea la principal API ORM, podrá existir:

```php
DB::entityManager();
```

como acceso avanzado.

---

# 138. EntityManager Scope

Siempre devolverá el EntityManager del scope vigente.

---

# 139. No EntityManager Static Cache

Prohibido:

```php
DB::$entityManager
```

---

# 140. IdentityMap Safety

Dos llamadas:

```php
DB::entityManager();
DB::entityManager();
```

en el mismo scope pueden devolver la misma instancia scoped.

Entre scopes deberán ser distintas.

---

# 141. Facade Instance Semantics

```text
same scope
→ same scoped service may be returned

different scope
→ different mutable service
```

---

# 142. Schema Context

Schema operations podrán usar una conexión administrativa/migration role distinta del runtime role.

---

# 143. Schema Facade Resolution

Por tanto:

```text
Schema
```

no deberá asumir que:

```text
runtime Connection == migration Connection
```

---

# 144. Migration Context

Durante migraciones:

```text
Migration Execution Scope
```

podrá proporcionar credenciales/capabilities distintas.

---

# 145. Administrative APIs

No deberán agregarse indiscriminadamente a `DB`.

---

# 146. Example

Evitar:

```php
DB::dropDatabase();
DB::killConnection();
DB::restoreBackup();
```

como convenience APIs cotidianas.

---

# 147. Administration Service

Operaciones privilegiadas deberán residir en:

```text
Database Administration System
```

---

# 148. Privileged Facade

Si en el futuro se crea, deberá ser explícita:

```text
DatabaseAdmin
```

y sujeta a autorización.

---

# 149. No Privilege Escalation

Una Facade nunca deberá otorgar privilegios que el servicio subyacente no posee.

---

# 150. Security

Facade resolution deberá respetar:

```text
scope
credentials
authorization
tenant isolation
connection policies
```

---

# 151. SQL Injection

La Facade no deberá introducir APIs que eviten el Query Engine sin señalización.

---

# 152. DB::table()

Seguirá utilizando:

```text
parameter binding
identifier validation
query validation
```

---

# 153. DB::raw()

Deberá estar claramente documentada como escape hatch.

---

# 154. Credentials

Nunca:

```php
DB::password();
```

como API pública ordinaria.

---

# 155. Debug Information

`DB::connection()` no deberá exponer passwords mediante `__toString()` o debug dump.

---

# 156. Facade Serialization

Las Facades no son objetos de dominio y no deberán serializar servicios.

---

# 157. Closures

Debe prestarse atención a callbacks que sobrevivan al scope.

Incorrecto:

```text
capture scoped service
queue closure
execute after scope closed
```

---

# 158. Scope Escape Detection

En debug/testing podrá detectarse uso de objetos scoped después de cierre.

---

# 159. Facade Reference Escape

No existe un objeto DB que escape.

Pero los objetos retornados sí pueden hacerlo.

Ejemplo:

```php
$connection = DB::connection();
```

---

# 160. Connection After Scope

Si se utiliza tras cerrar el scope deberá:

```text
fail
```

o estar explícitamente diseñado como detached-safe.

---

# 161. Scope-bound Objects

Podrán implementar metadata:

```text
ScopeBound
```

---

# 162. Scope Guard

Conceptualmente:

```php
interface ScopeBound
{
    public function scopeId(): ScopeId;
}
```

---

# 163. Debug Guard

Una operación podrá validar:

```text
object.scopeId == currentScope.id
```

cuando sea necesario.

---

# 164. No Cross-request Resource Reuse by Application

Aunque internamente exista connection pooling:

```text
Connection Lease
```

de un request no deberá ser retenido por aplicación para otro.

---

# 165. Pool ≠ Facade State

La reutilización física de conexiones corresponde al:

```text
Connection Pool
```

no a `DB`.

---

# 166. Facade Boot

Durante bootstrap:

```text
Facade definitions
```

podrán registrarse.

---

# 167. Bootstrap ≠ Resolve Scoped Services

El framework no deberá resolver:

```text
EntityManager
Connection
TransactionContext
```

simplemente al registrar la Facade.

---

# 168. Lazy Resolution

Los servicios se resolverán al utilizar la API.

---

# 169. Eager Validation

Sin embargo, metadata de Facades podrá validarse en bootstrap:

```text
accessor exists
service contract valid
no invalid dependencies
```

---

# 170. Facade Registry

Podrá existir:

```text
FacadeRegistry
```

inmutable después de bootstrap.

---

# 171. FacadeDefinition

Conceptualmente:

```php
final readonly class FacadeDefinition
{
    public function __construct(
        public string $facade,
        public string $service,
        public FacadeStability $stability,
    ) {}
}
```

---

# 172. Registry ≠ Runtime State

El registry almacena:

```text
definitions
```

no:

```text
resolved services
```

---

# 173. Third-party Facades

Plugins podrán registrar Facades propias si el Extension System lo permite.

---

# 174. Namespace Governance

Las Facades oficiales podrán vivir en:

```text
VoltStack\Facades
```

o:

```text
VoltStack\Quantum\Database\Facade
```

según la convención global del framework.

---

# 175. Recommended Public Namespace

Considerando que VoltStack ya contempla:

```text
src/Facades
```

podría utilizarse:

```php
VoltStack\Facades\DB
VoltStack\Facades\Schema
```

---

# 176. Internal Implementation

Mientras que infraestructura interna podría vivir en:

```text
VoltStack\Platform\Facade
```

o:

```text
VoltStack\Support\Facade
```

si es compartida por todo el framework.

---

# 177. Important Architectural Decision

El mecanismo base de Facades no debería pertenecer exclusivamente a Database.

Database sólo define:

```text
DB facade
Schema facade
database-specific facade adapters
```

---

# 178. Framework-level Facade Runtime

Arquitectura preferida:

```text
VoltStack Platform
└── Facade Runtime
       │
       ├── Cache Facade
       ├── Event Facade
       ├── DB Facade
       └── Schema Facade
```

---

# 179. Database Dependency Direction

```text
Quantum/Database
        ↓
Platform Facade Contracts
```

No:

```text
Platform
↓
Quantum/Database
```

---

# 180. Proposed Structure

```text
src/
├── Platform/
│   └── Facade/
│       ├── Facade.php
│       ├── FacadeRuntime.php
│       ├── FacadeResolver.php
│       ├── FacadeRegistry.php
│       ├── FacadeDefinition.php
│       ├── RuntimeContextAdapter.php
│       └── Exception/
│
├── Facades/
│   ├── DB.php
│   └── Schema.php
│
└── Quantum/
    └── Database/
        └── Facade/
            ├── DatabaseFacadeAdapter.php
            ├── SchemaFacadeAdapter.php
            └── Support/
```

---

# 181. Alternative

Si el framework decide mantener `Facade` como módulo Quantum:

```text
Quantum/Facade
```

Database deberá depender únicamente de sus contratos públicos.

---

# 182. DB Facade API V1

Una posible superficie inicial:

```php
DB::connection();
DB::connection('name');

DB::query();
DB::table('table');

DB::transaction($callback);

DB::entityManager();

DB::schema();

DB::capabilities();
```

---

# 183. Potential Convenience APIs

Podrán evaluarse:

```php
DB::select();
DB::insert();
DB::update();
DB::delete();
```

pero no son obligatorias para V1.

---

# 184. API Minimalism

Es preferible comenzar con:

```text
small coherent facade
```

y ampliar posteriormente.

---

# 185. Avoid Legacy Convenience Debt

Cada método estático añadido se convierte potencialmente en:

```text
public compatibility obligation
```

---

# 186. Schema Facade V1

Podría incluir:

```php
Schema::create();
Schema::table();
Schema::drop();
Schema::dropIfExists();
Schema::hasTable();
Schema::hasColumn();
Schema::connection();
```

---

# 187. Schema Introspection

Métodos:

```php
Schema::hasTable(...)
```

deberán usar:

```text
Schema Introspection System
```

no consultas manuales dispersas.

---

# 188. Capability Awareness

Una operación Schema deberá validar:

```text
Platform Capabilities
```

antes de compilar/ejecutar.

---

# 189. Query Facade Return

```php
DB::table('users')
```

deberá devolver un Query Builder normal.

Después de eso:

```text
Facade is no longer involved
```

salvo que el builder utilice servicios scoped explícitamente diseñados.

---

# 190. Builder Scope

Debe decidirse si Query Builder es:

```text
scope-bound
```

o:

```text
detached immutable definition
```

---

# 191. Preferred Separation

Idealmente:

```text
Query Definition
```

puede ser detached.

Pero:

```text
execution
```

requiere un contexto válido.

---

# 192. Example

Un Query Model puro puede sobrevivir:

```text
scope A
→ build query definition
```

pero no deberá retener:

```text
Connection Lease A
TransactionContext A
```

---

# 193. Facade-generated Builder

Deberá evitar capturar recursos físicos prematuramente.

---

# 194. Lazy Connection Acquisition

```php
DB::table('users')
```

no debería adquirir conexión inmediatamente.

---

# 195. Connection Acquisition

Deberá ocurrir cerca de:

```text
execution
```

---

# 196. Benefit

Esto mejora:

```text
resource usage
routing
transaction affinity
tenant resolution
failure handling
```

---

# 197. Context Capture

Sin embargo, si la query debe preservar:

```text
tenant
database
shard
```

deberá hacerlo mediante un contexto lógico seguro.

---

# 198. Context Snapshot vs Live Context

Esta decisión deberá ser explícita.

---

# 199. Recommended Rule

Una query creada dentro de un contexto sensible no deberá ejecutarse posteriormente bajo otro contexto ambiguamente.

---

# 200. Scope-bound Query Execution

Una opción segura:

```text
Query Builder
→ carries logical ContextIdentity
→ execution validates compatible current scope
```

---

# 201. Detached Query Definition

Otra opción avanzada:

```text
Detached Query Definition
```

sin tenant/transaction state, que debe bindearse explícitamente antes de ejecutar.

---

# 202. No Accidental Context Migration

Nunca:

```text
build under Tenant A
execute under Tenant B
```

sin error o rebind explícito.

---

# 203. Facade Cache

No deberá existir:

```php
DB::rememberResolvedInstance();
```

global.

---

# 204. Service Container Cache

El Container sí podrá almacenar:

```text
resolved scoped instance
```

durante el scope.

---

# 205. Reset

Al finalizar request:

```text
ExecutionScope
→ closing
→ scoped services reset
→ connections returned/discarded
→ scope closed
```

---

# 206. Facade Reset

Idealmente la Facade no necesita limpiar servicios porque nunca los poseyó.

---

# 207. Important Property

```text
Facade cleanup
≈
no-op
```

porque:

```text
Facade owns no mutable service state
```

---

# 208. Scope Cleanup

La limpieza corresponde a:

```text
ExecutionScope
Container Scope
Database Runtime Lifecycle
```

---

# 209. Failure During Cleanup

No deberá dejar un scope marcado como reutilizable.

---

# 210. Facade Call During Closing

Deberá rechazarse salvo APIs explícitamente permitidas.

---

# 211. Scope State Validation

```text
ACTIVE
→ resolve allowed

CLOSING
→ normally reject

CLOSED
→ reject
```

---

# 212. Recursion

Una Facade puede indirectamente causar otra llamada a Facade.

Esto es aceptable si no genera ciclos.

---

# 213. Circular Resolution

Ejemplo:

```text
DB
→ Database
→ service provider
→ DB
→ Database
```

deberá detectarse.

---

# 214. Resolution Stack

El Facade Resolver/Container podrá mantener:

```text
resolution stack
```

scoped al proceso de resolución.

---

# 215. CircularFacadeResolutionException

Podrá existir.

---

# 216. Static Analysis

VoltStack deberá proporcionar firmas reales o stubs para:

```text
DB
Schema
```

---

# 217. IDE

El developer deberá obtener autocomplete:

```php
DB::
```

sin depender de plugins específicos.

---

# 218. Documentation Generation

Las firmas explícitas podrán alimentar documentación automática.

---

# 219. Public API Classification

Las Facades oficiales V1 podrán clasificarse:

```text
PUBLIC_STABLE
```

una vez estabilizadas.

---

# 220. Facade Base Runtime

La infraestructura base podrá ser:

```text
PUBLIC_SUPPORTED
```

para extensiones.

---

# 221. Internal Resolver Details

Podrán permanecer:

```text
INTERNAL
```

---

# 222. Compatibility

Cambios breaking incluyen:

```text
remove facade
rename facade
remove method
change method semantics
change return type
change parameter names/types
change resolution scope semantics
```

---

# 223. Behavioral Compatibility

Cambiar:

```php
DB::connection()
```

de:

```text
current scoped connection
```

a:

```text
new physical connection every call
```

sería un breaking semantic change aunque la firma sea idéntica.

---

# 224. Deprecation

Métodos de Facade deberán seguir:

```text
Database Deprecation Policy
```

---

# 225. No Hidden Alias Forever

Aliases antiguos podrán deprecarse y retirarse conforme a versioning.

---

# 226. Facade Aliases

VoltStack podría permitir aliases:

```text
DB
Database
```

pero deberán evitarse aliases excesivos.

---

# 227. Preferred Name

```text
DB
```

es conciso y familiar.

---

# 228. Schema

```text
Schema
```

es igualmente claro.

---

# 229. ORM Facade

No se recomienda inicialmente una:

```text
ORM
```

Facade.

---

# 230. Transaction Facade

Tampoco es necesaria si:

```php
DB::transaction()
```

cubre el caso común.

---

# 231. Testing Architecture

La Facade tendrá pruebas en:

```text
tests/Platform/Facade/
tests/Quantum/Database/Facade/
```

---

# 232. Unit Tests

Validarán:

```text
accessor mapping
delegation
resolution errors
no service caching
scope validation
```

---

# 233. Integration Tests

Validarán:

```text
Facade
→ Container
→ real Database service
```

---

# 234. Persistent Runtime Tests

Críticos:

```text
same worker
multiple requests
different contexts
```

---

# 235. Concurrent Runtime Tests

Para OpenSwoole:

```text
multiple coroutines
same process
different scopes
```

---

# 236. Tenant Isolation Tests

Cuando Multitenancy esté instalado.

---

# 237. Transaction Tests

Deberán comprobar que:

```php
DB::transaction(...)
```

usa exactamente Transaction Manager semantics.

---

# 238. Query Tests

Deberán demostrar:

```text
DB::table()
→ Query AST
```

y no SQL directo.

---

# 239. Schema Tests

Deberán demostrar:

```text
Schema Facade
→ Schema System
```

---

# 240. Security Tests

Incluyen:

```text
raw expressions
secret redaction
cross-scope access
tenant leakage
unsafe access
```

---

# 241. Performance Tests

Compararán conceptualmente:

```text
direct DI call
vs
Facade call
```

para vigilar overhead.

---

# 242. Performance Goal

La Facade no deberá introducir overhead material frente a una operación Database real.

---

# 243. Microbenchmark Caveat

Un nanosegundo adicional en dispatch no deberá optimizarse sacrificando scope safety.

---

# 244. Correctness > Micro-optimization

Regla:

```text
Scope Safety
>
Facade dispatch microbenchmark
```

---

# 245. Facade Diagnostics

En desarrollo podrá inspeccionarse:

```php
FacadeInspector::inspect(DB::class);
```

produciendo:

```text
Facade: DB
Service: Database
Scope: request
Runtime: FrankenPHP
Resolution: scoped
State: ACTIVE
```

---

# 246. No Object Dump

No deberá imprimir internals sensibles.

---

# 247. CLI Diagnostics

Futuro:

```text
php voltstack database:facade DB
```

podría mostrar resolución.

---

# 248. Facade Manifest

Podrá generarse:

```json
{
  "facade": "VoltStack\\Facades\\DB",
  "service": "VoltStack\\Quantum\\Database\\Database",
  "scope": "contextual",
  "stability": "stable"
}
```

---

# 249. Facade Conformance

Facades de terceros podrán validarse.

---

# 250. Conformance Requirements

Una Facade Database-compatible deberá:

```text
declare accessor
not own scoped service
respect execution scope
preserve exception semantics
preserve return types
not bypass security
not bypass capability system
```

---

# 251. Facade Invariants

## DB-FACADE-001

Static Syntax ≠ Static State.

## DB-FACADE-002

Facade ≠ Service.

## DB-FACADE-003

Facade ≠ Service Container.

## DB-FACADE-004

Facade ≠ Global Service Locator.

## DB-FACADE-005

Facade ≠ Static Singleton.

## DB-FACADE-006

Facade será una convenience projection.

## DB-FACADE-007

Facades no poseerán servicios scoped.

## DB-FACADE-008

Facades no poseerán EntityManager global.

## DB-FACADE-009

Facades no poseerán UnitOfWork global.

## DB-FACADE-010

Facades no poseerán IdentityMap global.

---

# 252. Context Invariants

## DB-FACADE-011

Facades resolverán el ExecutionScope actual.

## DB-FACADE-012

No Current Scope ≠ Previous Scope.

## DB-FACADE-013

Closed Scope no podrá resolver servicios normales.

## DB-FACADE-014

Scoped services pertenecerán al scope vigente.

## DB-FACADE-015

Context override será exception-safe.

## DB-FACADE-016

Nested context overrides restaurarán correctamente el contexto anterior.

## DB-FACADE-017

Context override ≠ Static Mutation.

## DB-FACADE-018

Tenant Context no será global.

## DB-FACADE-019

Shard Context no será global.

## DB-FACADE-020

Sticky Context no será global.

---

# 253. Connection Invariants

## DB-FACADE-021

DB::connection() ≠ Static Connection.

## DB-FACADE-022

Connection selection será delegada.

## DB-FACADE-023

Facade no elegirá replica directamente.

## DB-FACADE-024

Facade no elegirá shard directamente.

## DB-FACADE-025

Facade no implementará failover.

## DB-FACADE-026

Facade no implementará pooling.

## DB-FACADE-027

Connection pool ≠ Facade cache.

## DB-FACADE-028

Physical connection reuse será responsabilidad de Connection System.

## DB-FACADE-029

Scope-bound connection no escapará silenciosamente entre requests.

## DB-FACADE-030

Native handle no será estado de Facade.

---

# 254. Query Invariants

## DB-FACADE-031

DB::table() producirá Query Builder/Query Model.

## DB-FACADE-032

DB::table() no generará SQL inmediatamente.

## DB-FACADE-033

Facade no será SQL compiler.

## DB-FACADE-034

Facade no será Query Executor.

## DB-FACADE-035

Values seguirán parameterized.

## DB-FACADE-036

Identifiers seguirán validados.

## DB-FACADE-037

Raw SQL seguirá siendo explícito.

## DB-FACADE-038

Raw Expression ≠ Parameter Value.

## DB-FACADE-039

Facade-generated builders no adquirirán conexiones prematuramente.

## DB-FACADE-040

Query context no migrará accidentalmente entre tenants/scopes.

---

# 255. Transaction Invariants

## DB-FACADE-041

DB::transaction() delegará en TransactionManager.

## DB-FACADE-042

Facade ≠ Transaction Manager.

## DB-FACADE-043

Facade no decidirá nested transaction semantics.

## DB-FACADE-044

Facade no decidirá retry safety.

## DB-FACADE-045

UNKNOWN commit outcome será preservado.

## DB-FACADE-046

UNKNOWN no activará retry automático.

## DB-FACADE-047

flush() ≠ commit() incluso mediante Facades.

## DB-FACADE-048

Savepoint ≠ Independent Transaction.

## DB-FACADE-049

Transaction Context será scoped.

## DB-FACADE-050

Transaction affinity será respetada.

---

# 256. Runtime Invariants

## DB-FACADE-051

FrankenPHP requests estarán aislados.

## DB-FACADE-052

RoadRunner operations estarán aisladas.

## DB-FACADE-053

OpenSwoole coroutines estarán aisladas.

## DB-FACADE-054

CLI long-running operations estarán aisladas.

## DB-FACADE-055

Queue jobs estarán aislados.

## DB-FACADE-056

Test scopes estarán aislados.

## DB-FACADE-057

Process Lifetime ≠ Scope Lifetime.

## DB-FACADE-058

Worker Lifetime ≠ EntityManager Lifetime.

## DB-FACADE-059

Worker Lifetime ≠ TransactionContext Lifetime.

## DB-FACADE-060

Runtime adapter ≠ Database semantics.

---

# 257. Testing Invariants

## DB-FACADE-061

Facade fake ≠ Global Static Fake.

## DB-FACADE-062

Test overrides serán scope-local.

## DB-FACADE-063

Parallel tests no compartirán overrides.

## DB-FACADE-064

Facade tests no sustituirán integration tests.

## DB-FACADE-065

Facade tests comprobarán delegation.

## DB-FACADE-066

Persistent runtime tests comprobarán leakage.

## DB-FACADE-067

Multitenancy tests comprobarán tenant leakage.

## DB-FACADE-068

Transaction tests comprobarán semántica real del TransactionManager.

## DB-FACADE-069

Facade mocking no redefinirá Database semantics.

## DB-FACADE-070

Test cleanup será determinista.

---

# 258. Public API Invariants

## DB-FACADE-071

Métodos principales tendrán firmas explícitas.

## DB-FACADE-072

`__callStatic` no será mecanismo universal.

## DB-FACADE-073

Return types serán preservados.

## DB-FACADE-074

Database exceptions serán preservadas.

## DB-FACADE-075

FacadeResolutionException se limitará a resolución.

## DB-FACADE-076

Facades oficiales serán clasificadas por estabilidad.

## DB-FACADE-077

Facade API formará parte del compatibility contract.

## DB-FACADE-078

Facade semantics formarán parte del compatibility contract.

## DB-FACADE-079

Facade deprecation seguirá política oficial.

## DB-FACADE-080

Facade ≠ Internal API exposure mechanism.

---

# 259. Security Invariants

## DB-FACADE-081

Facade no elevará privilegios.

## DB-FACADE-082

Facade no bypassará authorization.

## DB-FACADE-083

Facade no bypassará tenant isolation.

## DB-FACADE-084

Facade no bypassará query security.

## DB-FACADE-085

Facade no expondrá credentials.

## DB-FACADE-086

Facade diagnostics serán redacted.

## DB-FACADE-087

Raw operations serán explícitas.

## DB-FACADE-088

Administrative operations no se mezclarán indiscriminadamente con DB Facade.

## DB-FACADE-089

Cross-tenant operations serán explícitas.

## DB-FACADE-090

Unsafe ≠ Uncontrolled.

---

# 260. Performance Invariants

## DB-FACADE-091

Facade resolution evitará reflection costosa por llamada.

## DB-FACADE-092

Accessor metadata podrá precompilarse.

## DB-FACADE-093

Scope-local service cache será permitido.

## DB-FACADE-094

Static service cache estará prohibido para servicios mutable/scoped.

## DB-FACADE-095

Facade resolution no abrirá conexión innecesariamente.

## DB-FACADE-096

DB::table() no adquirirá conexión física por defecto.

## DB-FACADE-097

Correctness tendrá prioridad sobre micro-optimización.

## DB-FACADE-098

Facade telemetry no tendrá cardinalidad descontrolada.

## DB-FACADE-099

Debug tracing estará desactivado por defecto en producción.

## DB-FACADE-100

Facade overhead será medible.

---

# 261. Additional Architectural Invariants

## DB-FACADE-101

Facade Registry almacenará definiciones, no servicios.

## DB-FACADE-102

Facade bootstrap no resolverá servicios scoped.

## DB-FACADE-103

Facade cleanup no será responsable del Database lifecycle.

## DB-FACADE-104

ExecutionScope será responsable del scoped lifecycle.

## DB-FACADE-105

Facade no retendrá Result/Cursor/Stream.

## DB-FACADE-106

Facade no retendrá ConnectionLease.

## DB-FACADE-107

Facade no retendrá Query Builder.

## DB-FACADE-108

Facade no retendrá callbacks.

## DB-FACADE-109

Facade no retendrá TenantId mutable actual.

## DB-FACADE-110

Facade no retendrá current connection mutable.

## DB-FACADE-111

Schema Facade ≠ Schema Compiler.

## DB-FACADE-112

DB Facade ≠ ORM.

## DB-FACADE-113

DB Facade ≠ Driver.

## DB-FACADE-114

DB Facade ≠ Platform.

## DB-FACADE-115

DB Facade ≠ Dialect.

## DB-FACADE-116

Model static API y DB Facade compartirán execution context infrastructure.

## DB-FACADE-117

No existirán context stacks incompatibles por API.

## DB-FACADE-118

Context-sensitive query execution validará su contexto.

## DB-FACADE-119

Third-party Facades deberán respetar conformance contracts.

## DB-FACADE-120

Facade convenience nunca reducirá garantías silenciosamente.

---

# 262. Anti-patrones

## 262.1 Static Database Instance

```php
final class DB
{
    public static Database $instance;
}
```

Prohibido.

---

## 262.2 Static EntityManager

```php
DB::$entityManager
```

Prohibido.

---

## 262.3 Static Connection

```php
DB::$connection
```

Prohibido.

---

## 262.4 Static Transaction

```php
DB::$transaction
```

Prohibido.

---

## 262.5 Static Tenant

```php
DB::$tenant
```

Prohibido.

---

## 262.6 Static Sticky State

```php
DB::$sticky = true;
```

Prohibido.

---

## 262.7 Previous Scope Fallback

```text
No current request
→ use last request
```

Prohibido.

---

## 262.8 Facade implementing SQL

Incorrecto.

---

## 262.9 Facade implementing ORM persistence

Incorrecto.

---

## 262.10 Facade implementing retry policy

Incorrecto.

---

## 262.11 Facade implementing replica selection

Incorrecto.

---

## 262.12 Facade implementing tenant filtering

Incorrecto.

---

## 262.13 Facade-specific transaction semantics

Incorrecto.

---

## 262.14 Global Test Swap

```php
DB::$instance = $fake;
```

Prohibido.

---

## 262.15 Reflection on Every Call

Evitar.

---

## 262.16 Opening Connection on DB::table()

Evitar.

---

## 262.17 Raw SQL Convenience Everywhere

Evitar.

---

## 262.18 Administrative God Facade

Evitar.

---

## 262.19 Hundreds of Facades

Evitar.

---

## 262.20 Catch-all `__callStatic`

Evitar como estrategia principal.

---

# 263. Modelo formal

Sea:

```text
F
```

una Facade,

```text
S
```

un servicio,

y:

```text
C
```

el execution context.

Entonces:

```text
resolve(F, C) → S(C)
```

No:

```text
resolve(F) → static S
```

---

# 264. Scope Isolation

Para dos scopes:

```text
C1 ≠ C2
```

y un servicio mutable scoped:

```text
S(C1) ≠ S(C2)
```

---

# 265. Facade Identity

La Facade puede ser conceptualmente la misma:

```text
F(C1) = F(C2)
```

porque es sólo sintaxis.

Pero sus servicios resueltos deberán respetar:

```text
S(C1) ≠ S(C2)
```

cuando sean scoped.

---

# 266. Root Service Exception

Para un servicio inmutable root:

```text
R(C1) = R(C2)
```

puede ser válido.

---

# 267. Service Ownership

Debe cumplirse:

```text
owns(F, S) = false
```

La Facade nunca es propietaria del servicio.

---

# 268. Lifecycle Ownership

```text
lifecycleOwner(S)
=
ContainerScope / RuntimeScope
```

No:

```text
Facade
```

---

# 269. Resolution Validity

Una llamada de Facade es válida cuando:

```text
ScopeExists
∧
ScopeActive
∧
ServiceRegistered
∧
ResolutionAllowed
```

---

# 270. Failure

Si cualquiera es falso:

```text
FacadeResolutionFailure
```

No fallback silencioso.

---

# 271. Query Context Safety

Para una query `Q` creada en contexto `C1`:

```text
execute(Q, C2)
```

sólo será válido si:

```text
compatible(Q.context, C2)
```

o la query es explícitamente detached.

---

# 272. Runtime Safety Formula

```text
Static Syntax
+
Scoped Resolution
+
Scoped Services
+
Deterministic Cleanup
=
Persistent Runtime Safe Facade
```

---

# 273. Arquitectura consolidada

```text
                     APPLICATION
                         │
                         ▼
                    DB / Schema
                      Facades
                         │
                         ▼
                  Facade Runtime
                         │
                         ▼
                Runtime Context Adapter
                         │
                         ▼
                   Execution Scope
                         │
                         ▼
                   Scoped Container
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
       Database     SchemaManager   EntityManager
          │              │               │
          ▼              ▼               ▼
     Query Engine    Schema Engine     UnitOfWork
          │                              │
          └──────────────┬───────────────┘
                         ▼
                  Execution Engine
                         │
                         ▼
                Connection Manager
                         │
                         ▼
                       Driver
                         │
                         ▼
                        DBMS
```

El punto crítico es:

```text
DB / Schema
```

no poseen:

```text
Database
SchemaManager
EntityManager
Connection
Transaction
```

Sólo los resuelven.

---

# 274. Flujo FrankenPHP

```text
FrankenPHP Worker
│
├── Request A
│   │
│   ├── ExecutionScope A
│   │      │
│   │      └── DatabaseContext A
│   │
│   ├── DB::transaction()
│   │      ↓
│   │   resolve A
│   │
│   └── close/reset A
│
├── Worker remains alive
│
└── Request B
    │
    ├── ExecutionScope B
    │      │
    │      └── DatabaseContext B
    │
    ├── DB::query()
    │      ↓
    │   resolve B
    │
    └── close/reset B
```

Debe cumplirse:

```text
MutableState(A)
∩
MutableState(B)
=
∅
```

---

# 275. Flujo OpenSwoole futuro

```text
Worker
│
├── Coroutine A
│   └── Scope A
│       └── DB → Database A
│
└── Coroutine B
    └── Scope B
        └── DB → Database B
```

Aunque ambas ejecuten simultáneamente:

```php
DB::connection();
```

deberán resolver el contexto correcto.

---

# 276. V1 Decisions

Para V1 se recomienda:

### Facades oficiales

```text
DB
Schema
```

### Resolution

```text
ExecutionScope
→ Scoped Container
```

### API implementation

Preferencia:

```text
explicit static methods
```

sobre proxy mágico universal.

### State

```text
zero scoped mutable state inside Facade
```

### Runtime principal

```text
FrankenPHP
```

### Future runtimes

```text
RoadRunner
OpenSwoole
```

mediante RuntimeContextAdapter.

---

# 277. V1 Non-goals

No será necesario en V1:

```text
Facade for every Database component
global Facade mocking
arbitrary service locator access
dynamic runtime-generated Facades
administrative Facade
async Facade API
```

---

# 278. Acceptance Criteria

El sistema estará preparado para V1 cuando pueda demostrarse:

1. `DB::connection()` resuelve desde el scope actual;
2. `DB::table()` utiliza Query Engine;
3. `DB::transaction()` utiliza TransactionManager;
4. `Schema::create()` utiliza Schema System;
5. ninguna Facade almacena servicios scoped;
6. ninguna Facade almacena EntityManager;
7. ninguna Facade almacena TransactionContext;
8. ninguna Facade almacena TenantContext;
9. Request A no contamina Request B;
10. scopes cerrados no pueden reutilizarse;
11. tests pueden reemplazar servicios dentro de su scope;
12. parallel tests no comparten overrides;
13. named connections no modifican global state;
14. context overrides son exception-safe;
15. DB exceptions se preservan;
16. resolución tiene errores claros;
17. autocompletado funciona;
18. no se utiliza reflection costosa por llamada;
19. FrankenPHP tiene pruebas dedicadas;
20. la arquitectura admite RoadRunner/OpenSwoole sin rediseñar la API.

---

# 279. Principio final

Las Facades de VoltStack deberán proporcionar:

```text
Laravel-like convenience
```

sin importar:

```text
Laravel-like assumptions about short-lived process state
```

a una arquitectura basada en workers persistentes.

La regla final será:

> **Una Facade de VoltStack no será un objeto global disfrazado de clase estática. Será únicamente una puerta sintáctica hacia un servicio correctamente resuelto dentro del execution scope vigente.**

Por tanto:

```text
Static Syntax
≠
Static State
```

```text
Facade
≠
Singleton
```

```text
Facade
≠
Service
```

```text
Facade
≠
Service Container
```

```text
Facade
≠
Global Service Locator
```

```text
DB
≠
Connection
```

```text
DB
≠
TransactionManager
```

```text
DB
≠
Query Engine
```

```text
Schema
≠
Schema Engine
```

```text
DB::table()
≠
SQL generation
```

```text
DB::transaction()
≠
Facade-owned transaction
```

```text
Named Connection
≠
Global Connection Mutation
```

```text
Test Fake
≠
Global Static Swap
```

```text
Process Lifetime
≠
Request Lifetime
```

```text
Worker Lifetime
≠
Database Context Lifetime
```

y finalmente:

```text
VoltStack Database Facade System
=
Static Developer Ergonomics
+
Explicit Public API
+
Execution Scope Resolution
+
Container Delegation
+
Zero Scoped Static State
+
Persistent Runtime Isolation
+
Transaction Correctness
+
Query Safety
+
Tenant Isolation
+
Testability
+
IDE Discoverability
+
Low Dispatch Overhead
```

---

# 280. Siguiente documento

```text
304_DATABASE_HELPER_SYSTEM.md
```

El siguiente documento definirá los **helpers de Database** y las reglas para ofrecer funciones de conveniencia sin convertir el framework en una colección de funciones globales con dependencias ocultas.

La regla central será:

> **Un helper de VoltStack podrá reducir ceremonia sintáctica, pero nunca será propietario de estado Database, nunca sustituirá contratos de servicio y nunca deberá ocultar una dependencia contextual cuya semántica sea relevante.**

Especial atención deberá darse a la relación:

```text
Helper
vs
Facade
vs
Service
vs
Value Factory
vs
Pure Utility
```

y a evitar:

```text
global mutable state
hidden connection resolution
hidden transaction boundaries
hidden tenant context
untyped magic helpers
```