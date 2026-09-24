# 304_DATABASE_HELPER_SYSTEM.md

# VoltStack Quantum Database
## Database Helper System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 304 — Database Helper System  
**Bloque:** 31 — Developer Experience / Public API  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `303_DATABASE_FACADE_SYSTEM.md`  
**Siguiente documento:** `305_DATABASE_MODEL_DEVELOPER_EXPERIENCE.md`

---

# 1. Propósito

Este documento define la arquitectura oficial del **Database Helper System** de VoltStack.

Su objetivo es proporcionar pequeñas APIs de conveniencia para operaciones frecuentes relacionadas con Database sin introducir:

- estado global mutable;
- conexiones globales;
- transacciones implícitas;
- resolución opaca de tenants;
- SQL inseguro;
- duplicación de APIs;
- service location arbitrario;
- dependencias ocultas difíciles de probar;
- acoplamiento entre capas internas.

VoltStack podrá ofrecer helpers para mejorar la experiencia del desarrollador, pero dichos helpers deberán permanecer deliberadamente pequeños.

La regla central será:

> **Un helper de VoltStack podrá reducir ceremonia sintáctica, pero nunca será propietario de estado Database, nunca sustituirá contratos de servicio y nunca deberá ocultar una dependencia contextual cuya semántica sea relevante.**

Por tanto:

```text
Helper
≠
Service

Helper
≠
Facade

Helper
≠
Service Container

Helper
≠
Global Service Locator

Helper
≠
Database Runtime

Helper
≠
Connection Manager

Helper
≠
Transaction Manager
```

---

# 2. Contexto

El documento anterior definió:

```text
DB
Schema
```

como Facades oficiales de conveniencia.

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

Sin embargo, existen operaciones más pequeñas donde una clase Facade puede resultar innecesariamente verbosa.

Por ejemplo:

```php
db();
```

podría representar:

```text
Database service for current scope
```

mientras:

```php
db('analytics');
```

podría proporcionar acceso a una conexión lógica específica.

También pueden existir helpers puros como:

```php
identifier('users.email');
```

o factories de expresiones/value objects.

Pero introducir funciones globales indiscriminadamente puede degradar rápidamente la arquitectura.

---

# 3. Problema arquitectónico

Una API aparentemente inocente como:

```php
function db()
{
    static $database;

    return $database ??= new Database();
}
```

sería incompatible con la arquitectura de VoltStack.

Especialmente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
long-running CLI workers
queue workers
```

porque:

```text
Process Lifetime
>
Request Lifetime
```

y por tanto:

```text
static local variable
```

puede sobrevivir entre operaciones.

---

# 4. Regla fundamental

Los helpers deberán cumplir:

```text
Convenience
without
Ownership
```

Es decir:

```text
Helper
→ delegate / construct / transform
```

y nunca:

```text
Helper
→ own mutable Database state
```

---

# 5. Objetivos

El sistema deberá proporcionar:

1. ergonomía;
2. sintaxis compacta;
3. APIs predecibles;
4. tipado fuerte;
5. compatibilidad IDE;
6. seguridad;
7. scope awareness cuando corresponda;
8. testabilidad;
9. baja magia;
10. cero estado Database global mutable;
11. integración coherente con Facades;
12. integración con Container;
13. compatibilidad con persistent runtimes;
14. compatibilidad con Multitenancy;
15. compatibilidad con sharding;
16. compatibilidad con múltiples conexiones;
17. errores claros;
18. mínima superficie pública;
19. extensibilidad controlada;
20. estabilidad de API.

---

# 6. No objetivos

Database Helper System no deberá convertirse en:

```text
Laravel helper clone
```

ni en:

```text
hundreds of global functions
```

Tampoco pretende sustituir:

```text
Dependency Injection
Facades
Repositories
EntityManager
Query Builder
Schema Builder
Transaction Manager
Connection Manager
```

---

# 7. Taxonomía

VoltStack distinguirá al menos cinco conceptos:

```text
Helper
Facade
Service
Value Factory
Pure Utility
```

---

# 8. Helper

Un helper es una API pequeña de conveniencia.

Ejemplo conceptual:

```php
db();
```

Puede delegar hacia un servicio existente.

---

# 9. Facade

Una Facade es una superficie estática tipada que proyecta una API de servicio.

Ejemplo:

```php
DB::transaction(...);
```

---

# 10. Service

Un Service implementa comportamiento real.

Ejemplo:

```text
TransactionManager
ConnectionManager
QueryExecutor
```

---

# 11. Value Factory

Una Value Factory crea objetos de valor.

Ejemplo:

```php
db_identifier('users.email');
```

podría producir:

```text
QualifiedIdentifier
```

sin consultar el runtime.

---

# 12. Pure Utility

Una utilidad pura transforma valores sin depender del runtime.

Formalmente:

```text
f(x) → y
```

donde:

```text
same x
→ same y
```

sin:

```text
Database
Container
Connection
Tenant
Transaction
Clock
Network
```

---

# 13. Clasificación principal

Los helpers podrán clasificarse como:

```text
PURE
FACTORY
CONTEXTUAL
DELEGATING
```

---

# 14. PURE Helper

Ejemplo conceptual:

```php
normalize_database_name($name);
```

No necesita infraestructura.

---

# 15. FACTORY Helper

Construye un objeto.

Ejemplo:

```php
db_identifier('users.id');
```

produce:

```text
Identifier
```

---

# 16. CONTEXTUAL Helper

Necesita el ExecutionScope actual.

Ejemplo:

```php
db();
```

---

# 17. DELEGATING Helper

Resuelve un servicio y delega una operación.

Ejemplo hipotético:

```php
transaction(fn () => ...);
```

aunque este tipo deberá utilizarse con mucha mayor cautela.

---

# 18. Pure First

Siempre que una operación pueda ser:

```text
PURE
```

no deberá convertirse innecesariamente en:

```text
CONTEXTUAL
```

---

# 19. Architecture

```text
Application
    │
    ├── db()
    │
    ├── db_identifier()
    │
    └── database helper
            │
            ▼
       Helper Function
            │
      ┌─────┴─────┐
      ▼           ▼
Pure/Factory   Context Resolver
                  │
                  ▼
            ExecutionScope
                  │
                  ▼
               Container
                  │
                  ▼
            Database Service
```

---

# 20. Helper ≠ Facade

Aunque puedan producir resultados similares:

```php
db()->connection();
```

y:

```php
DB::connection();
```

no son arquitectónicamente idénticos.

---

# 21. Facade role

La Facade proporciona:

```text
static projected API
```

---

# 22. Helper role

El helper puede proporcionar:

```text
compact entry point
```

---

# 23. Shared Resolution

Ambos deberán converger en la misma infraestructura:

```text
ExecutionScope
→ Container
```

Nunca:

```text
DB Facade → Resolver A

db() helper → Resolver B
```

con contextos independientes.

---

# 24. Critical Invariant

```text
Facade Context
=
Helper Context
=
Model API Context
=
Current ExecutionScope
```

cuando todos participen en la misma operación.

---

# 25. Candidate V1 Helper

El principal candidato para V1 será:

```php
db();
```

---

# 26. `db()` sin argumentos

Podría retornar:

```text
Database
```

del scope actual.

Ejemplo:

```php
$database = db();
```

---

# 27. Contract

Conceptualmente:

```php
function db(): Database
{
    return database_context()
        ->container()
        ->get(Database::class);
}
```

La implementación concreta deberá reutilizar la infraestructura oficial de scope resolution.

---

# 28. No Static Cache

Prohibido:

```php
function db(): Database
{
    static $database;

    return $database ??= resolve(...);
}
```

---

# 29. Reason

En FrankenPHP:

```text
Request A
→ db()
→ Database A

Request ends

Request B
→ db()
```

deberá producir:

```text
Database B
```

cuando Database sea scoped.

Nunca:

```text
Database A
```

por un cache estático.

---

# 30. `db()` con conexión

Una posible API:

```php
db('analytics');
```

podría devolver:

```text
Connection
```

---

# 31. Ambigüedad

Sin embargo:

```php
db()
```

retornando `Database` y:

```php
db('analytics')
```

retornando `Connection`

introduce un tipo de retorno dependiente de argumentos.

Conceptualmente:

```text
db()            → Database
db('analytics') → Connection
```

---

# 32. Problema IDE

Esto puede reducir:

```text
static analysis
autocomplete
type predictability
```

---

# 33. Opción A

Mantener:

```php
db(): Database
```

únicamente.

Y usar:

```php
db()->connection('analytics');
```

---

# 34. Opción B

Permitir overload conceptual:

```php
db(): Database
db(string $connection): Connection
```

mediante PHPDoc/static analysis.

---

# 35. Recomendación V1

Para VoltStack V1 se recomienda:

```php
db(): Database
```

como contrato único.

Y:

```php
db()->connection('analytics');
```

para named connections.

Esto maximiza:

```text
predictability
type safety
IDE support
API simplicity
```

---

# 36. `db()` vs `DB`

Ambos podrán coexistir.

```php
db()->transaction(...);
```

y:

```php
DB::transaction(...);
```

podrán ser equivalentes.

---

# 37. Preferencia de estilo

VoltStack no deberá obligar al desarrollador a usar uno.

Podrá utilizar:

```text
Dependency Injection
```

para código explícito.

```text
Facade
```

para ergonomía estática.

```text
Helper
```

para acceso compacto.

---

# 38. Dependency Injection remains first-class

Ejemplo:

```php
final class CreateInvoice
{
    public function __construct(
        private Database $database,
    ) {}
}
```

será siempre una API válida.

---

# 39. No architecture penalty

Usar:

```php
db()
```

no deberá activar un motor diferente.

Todos convergen en:

```text
same Database architecture
```

---

# 40. One Engine Rule

```text
DI
Facade
Helper
Model API
Repository
```

son superficies diferentes.

Pero:

```text
Database Engine
```

es uno.

---

# 41. Transaction Helper

Podría considerarse:

```php
transaction(function () {
    // ...
});
```

---

# 42. Problema

Una función global:

```php
transaction(...)
```

es muy genérica.

Podría entrar en conflicto con:

```text
other packages
application functions
future framework modules
```

---

# 43. Recomendación

Para V1 se recomienda utilizar:

```php
DB::transaction(...)
```

o:

```php
db()->transaction(...)
```

en lugar de introducir inmediatamente:

```php
transaction(...)
```

---

# 44. Helper proliferation rule

No deberá añadirse un helper simplemente porque:

```text
it saves 5 characters
```

---

# 45. Helper admission criteria

Un nuevo helper público deberá demostrar:

1. alta frecuencia de uso;
2. semántica inequívoca;
3. nombre no conflictivo;
4. valor ergonómico real;
5. tipado claro;
6. estabilidad probable;
7. seguridad;
8. ausencia de duplicación dañina.

---

# 46. Helper Cost

Cada helper global añade:

```text
API surface
namespace pollution
compatibility obligation
documentation
testing
maintenance
static-analysis support
```

---

# 47. Global Function Namespace

PHP permite funciones namespaced.

VoltStack deberá preferir:

```php
namespace VoltStack\Helper;
```

o una convención framework-level equivalente.

---

# 48. Global convenience import

El framework podrá decidir cargar algunos helpers globalmente.

Pero el conjunto deberá ser pequeño.

---

# 49. Database-specific helpers

Podrían residir conceptualmente en:

```text
src/Helper/Database/
```

o:

```text
src/Quantum/Database/Helper/
```

dependiendo de si son framework-wide o module-specific.

---

# 50. Recommended structure

```text
src/
├── Helper/
│   └── database.php
│
└── Quantum/
    └── Database/
        └── Helper/
            ├── DatabaseHelper.php
            ├── DatabaseHelperResolver.php
            └── Value/
```

Sin embargo, una función simple no requiere obligatoriamente una clase wrapper.

---

# 51. Helper Loading

Los helpers deberán registrarse durante bootstrap.

Pero:

```text
loading helper definitions
```

no deberá resolver:

```text
Database
Connection
EntityManager
TransactionContext
```

---

# 52. Bootstrap Rule

```text
Load Function
≠
Resolve Service
```

---

# 53. Lazy Resolution

Un helper contextual resolverá servicios únicamente cuando sea invocado.

---

# 54. ExecutionScope

Los helpers contextuales deberán usar:

```text
CurrentScopeResolver
```

o el contrato equivalente establecido por Platform.

---

# 55. No independent helper context

Prohibido:

```text
DatabaseHelper::$currentScope
```

---

# 56. No Last Container

Prohibido:

```php
function db()
{
    return GlobalContainer::last();
}
```

---

# 57. No previous scope fallback

Debe cumplirse:

```text
No Active Scope
≠
Previous Scope
```

---

# 58. Outside Scope

Si `db()` es utilizado fuera de un scope válido:

```php
db();
```

deberá producir un error claro cuando Database requiera scope.

---

# 59. Exception

Podrá existir:

```text
NoActiveDatabaseScopeException
```

o reutilizar:

```text
NoActiveExecutionScopeException
```

de Platform.

---

# 60. No implicit scope creation

Por defecto:

```php
db();
```

no deberá crear silenciosamente un request/operation scope.

---

# 61. Why

Crear scopes implícitamente puede esconder:

```text
lifecycle
cleanup
transaction ownership
connection ownership
tenant context
```

---

# 62. Explicit operation scopes

Cuando código standalone necesite Database podrá utilizar:

```php
$app->run(function () {
    db()->...
});
```

o la API oficial equivalente del runtime.

---

# 63. Helper and DatabaseContext

`db()` deberá resolver dentro del:

```text
current DatabaseContext
```

cuando corresponda.

---

# 64. DatabaseContext ≠ Helper state

El helper no almacena:

```text
DatabaseContext
```

---

# 65. Tenant awareness

Si Multitenancy está instalado:

```text
db()
```

podrá terminar operando bajo un tenant.

Pero la resolución del tenant no pertenece al helper.

---

# 66. Flow

```text
db()
 ↓
Database
 ↓
DatabaseContext
 ↓
Tenant-aware Connection Resolution
```

---

# 67. Wrong flow

No:

```text
db()
 ↓
read global tenant variable
 ↓
select tenant DB
```

---

# 68. Sharding

Igualmente:

```text
db()
```

no calculará shard keys.

---

# 69. Read/write routing

El helper no decidirá:

```text
writer
replica
```

---

# 70. Transaction affinity

Dentro de:

```php
DB::transaction(function () {
    db()->table('users')->...
});
```

el helper deberá resolver el mismo contexto transaccional.

---

# 71. Shared context guarantee

Si:

```php
DB::transaction(function () {
    $database = db();
});
```

entonces:

```text
DB Facade Transaction Context
=
db() Database Context
```

---

# 72. Connection helper

Podría existir en el futuro:

```php
db_connection();
```

pero no se recomienda inicialmente.

---

# 73. Reason

```php
db()->connection();
```

ya es:

```text
short
explicit
typed
discoverable
```

---

# 74. Query helper

Podría imaginarse:

```php
query('users');
```

pero tampoco se recomienda inicialmente.

---

# 75. Preferred API

```php
DB::table('users');
```

o:

```php
db()->table('users');
```

---

# 76. Schema helper

No se recomienda:

```php
schema();
```

para V1 si:

```php
Schema::
```

y:

```php
db()->schema()
```

ya cubren el caso.

---

# 77. Minimal helper philosophy

V1 deberá comenzar con:

```text
very small helper set
```

---

# 78. Value helpers

Un área donde los helpers pueden ser especialmente seguros es la construcción de valores.

Ejemplo conceptual:

```php
db_identifier('users.id');
```

---

# 79. But naming matters

Antes de crear:

```php
db_identifier()
```

deberá evaluarse si:

```php
Identifier::from('users.id');
```

es más claro.

---

# 80. Static Factory vs Helper

Muchas veces:

```php
Identifier::from(...)
```

será preferible a:

```php
identifier(...)
```

porque:

```text
type is explicit
namespace pollution is lower
IDE discovery is better
```

---

# 81. Helper threshold

Un helper de Value Factory deberá existir sólo cuando tenga valor significativo.

---

# 82. Expression Helpers

Evitar una explosión como:

```php
eq()
neq()
gt()
gte()
lt()
lte()
and_()
or_()
column()
table()
raw()
sql()
json()
```

globalmente.

---

# 83. Preferred Query DSL

Estas operaciones deberán vivir normalmente en:

```text
Query Builder
Expression Builder
```

---

# 84. Example

Preferible:

```php
$query->where('age', '>', 18);
```

a:

```php
where(gt(column('age'), 18));
```

para el API general.

---

# 85. Advanced AST API

Los usuarios avanzados podrán utilizar:

```text
ExpressionFactory
AST factories
```

sin necesidad de funciones globales.

---

# 86. Helper Security

Un helper jamás deberá introducir una ruta insegura como:

```php
sql($input);
```

---

# 87. Raw helper

No se recomienda un helper global:

```php
raw($sql);
```

---

# 88. Reason

`raw()` carece de contexto semántico.

Puede confundirse entre:

```text
HTML raw
SQL raw
JSON raw
HTTP raw
template raw
```

---

# 89. Explicit type preferred

Preferible:

```php
RawExpression::trusted($sql);
```

o una API equivalente.

---

# 90. Trusted ≠ Safe Input

`trusted()` deberá indicar:

```text
developer asserts this expression is trusted
```

No:

```text
framework sanitizes arbitrary SQL
```

---

# 91. User Input

Nunca:

```php
RawExpression::trusted($request->input('sort'));
```

---

# 92. Identifier helper security

Si existe una factory de identifier:

```php
Identifier::from($value);
```

deberá validar el formato correspondiente.

---

# 93. Identifier ≠ Value

Debe mantenerse:

```text
SQL Identifier
≠
SQL Parameter Value
```

---

# 94. Parameter helpers

Generalmente no se necesitarán funciones globales para parameters.

El Query Builder deberá encargarse de binding.

---

# 95. No manual escaping helper

No deberá existir:

```php
db_escape($value);
```

como API recomendada.

---

# 96. Reason

La estrategia correcta será:

```text
parameter binding
```

no:

```text
manual escaping
```

---

# 97. Quoting identifiers

Tampoco deberá exponerse como helper general:

```php
quote_identifier();
```

si esto depende del dialecto/plataforma.

---

# 98. Dialect responsibility

Identifier quoting pertenece al:

```text
SQL Compiler / Dialect
```

---

# 99. Helper ≠ Dialect

Critical invariant:

```text
Helper
≠
SQL Dialect
```

---

# 100. Helper ≠ Compiler

Nunca:

```php
mysql_limit_sql(...);
```

como helper del core.

---

# 101. Helper ≠ Driver

Nunca:

```php
pdo_mysql();
```

como arquitectura oficial.

---

# 102. Helper ≠ Capability Discovery

Nunca:

```php
postgres_supports_returning();
```

---

# 103. Correct API

Utilizar:

```php
db()->capabilities();
```

o el servicio correspondiente.

---

# 104. Capability-aware helper

Si un helper necesita una capability:

```text
Helper
→ service
→ capability system
```

No:

```text
Helper
→ vendor conditional
```

---

# 105. Vendor checks

Prohibido:

```php
if (db_driver() === 'mysql') {
    ...
}
```

como patrón de arquitectura interna.

---

# 106. Version checks

Igualmente:

```text
Version
≠
Capability
```

---

# 107. ORM helpers

No se recomienda introducir:

```php
entity();
repo();
em();
uow();
```

como helpers globales V1.

---

# 108. Why

Estas APIs ocultan conceptos importantes.

Por ejemplo:

```php
em();
```

es menos explícito que:

```php
$entityManager
```

o:

```php
db()->entityManager();
```

---

# 109. Repository access

Preferible:

```php
$entityManager->repository(User::class);
```

o inyección:

```php
UserRepository $users
```

---

# 110. Model API

Para experiencia Laravel-like ya existe:

```php
User::query();
User::find();
$user->save();
```

No es necesario duplicarla con helpers.

---

# 111. Active Record helper

No deberá existir:

```php
model(User::class);
```

sin una necesidad concreta.

---

# 112. UnitOfWork

No se recomienda:

```php
uow();
```

como API cotidiana.

---

# 113. IdentityMap

Tampoco:

```php
identity_map();
```

---

# 114. Internal architecture visibility

No todo componente interno necesita un helper.

---

# 115. Helper Public API Rule

```text
Internal Component
≠
Public Helper Candidate
```

---

# 116. Testing helpers

Database Testing podrá tener helpers propios dentro de namespaces/testing packages.

Ejemplo conceptual:

```php
database_test_environment();
```

pero no necesariamente deberán cargarse en producción.

---

# 117. Test-only helpers

Deberán estar:

```text
dev dependency / test namespace
```

cuando corresponda.

---

# 118. Production API pollution

No deberán cargarse cientos de funciones de testing en producción.

---

# 119. Test override

El helper `db()` deberá respetar overrides del Container del test.

Ejemplo:

```php
$testScope->replace(Database::class, $fake);

db();
```

deberá retornar:

```text
$fake
```

---

# 120. No helper-specific fake registry

Prohibido:

```php
DatabaseHelper::$fake
```

---

# 121. Parallel testing

Dos tests concurrentes:

```text
Test A
→ FakeDatabase A

Test B
→ FakeDatabase B
```

deberán permanecer aislados.

---

# 122. Persistent Runtime Safety

El Helper System deberá diseñarse explícitamente para:

```text
FrankenPHP
```

desde V1.

---

# 123. FrankenPHP scenario

```text
Worker
│
├── Request A
│   ├── db()
│   ├── Tenant A
│   └── Transaction A
│
└── Request B
    ├── db()
    ├── Tenant B
    └── no Transaction A
```

---

# 124. Required property

```text
MutableDatabaseState(Request A)
∩
MutableDatabaseState(Request B)
=
∅
```

---

# 125. Helper static local variables

Por tanto, helpers contextuales no deberán utilizar:

```php
static $service;
```

---

# 126. Memoization

Helpers puros sí podrían memoizar ciertos resultados si:

```text
immutable
context-independent
bounded
safe for process lifetime
```

---

# 127. Example

Una tabla inmutable de parsing podría potencialmente compartirse.

Pero esto será una optimización interna.

---

# 128. Contextual memoization

Si se necesita cachear resolución, deberá hacerlo:

```text
ExecutionScope Container
```

no la función helper.

---

# 129. RoadRunner

La misma regla aplicará a:

```text
Request/Job 1
Request/Job 2
```

dentro del mismo worker.

---

# 130. OpenSwoole

Los helpers deberán ser coroutine-safe.

---

# 131. Wrong design

```php
function db()
{
    global $currentDatabase;
}
```

sería incompatible.

---

# 132. Correct conceptual design

```text
db()
 ↓
Runtime Context Adapter
 ↓
Current Coroutine/Request Scope
 ↓
Container
 ↓
Database
```

---

# 133. CLI

`db()` podrá utilizarse en CLI cuando exista un scope activo.

---

# 134. Long-running CLI

Especial atención:

```text
import workers
queue consumers
scheduled daemons
```

---

# 135. One operation = one scope

Cuando corresponda:

```text
Job A
→ Scope A
→ close/reset

Job B
→ Scope B
```

---

# 136. Helper object escape

El helper puede retornar un objeto scoped.

Ejemplo:

```php
$db = db();
```

El desarrollador podría conservarlo.

---

# 137. Scope-bound semantics

Si el objeto está vinculado al scope, utilizarlo después del cierre deberá fallar o ser rechazado.

---

# 138. Example

```php
$db = db();

$scope->close();

$db->connection();
```

no deberá acceder accidentalmente al siguiente request.

---

# 139. Scope generation

Objetos scoped podrán conservar:

```text
ScopeId
ScopeGeneration
```

para detectar uso inválido en debug/testing.

---

# 140. Helper result ≠ globally reusable

La documentación deberá aclarar:

```text
db()
returns current scoped service
```

no:

```text
permanent application singleton
```

---

# 141. Dependency Injection recommendation

Servicios de larga duración no deberán capturar:

```php
$this->database = db();
```

durante bootstrap si el Database es scoped.

---

# 142. Correct long-lived service

Deberá depender de:

```text
resolver/factory/context-safe abstraction
```

o recibir Database dentro de cada operación.

---

# 143. Captive dependency

El Container deberá detectar cuando sea posible:

```text
singleton
→ captures scoped Database
```

---

# 144. Helper can hide captive dependency

Por eso utilizar helpers dentro de constructores singleton deberá ser desaconsejado.

---

# 145. Constructor Rule

Evitar:

```php
final class GlobalService
{
    public function __construct()
    {
        $this->database = db();
    }
}
```

---

# 146. Better

```php
final class OperationService
{
    public function __construct(
        private Database $database,
    ) {}
}
```

cuando el propio servicio sea scoped.

---

# 147. Helper use locations

Helpers contextuales son apropiados principalmente en:

```text
application operation code
controllers
commands
jobs
small callbacks
scripts within managed scope
```

---

# 148. Less appropriate locations

Evitar en:

```text
core constructors
static initializers
global bootstrap
configuration files requiring pure evaluation
immutable metadata compilation
```

---

# 149. Configuration

Config files no deberán ejecutar:

```php
db();
```

durante carga.

---

# 150. Why

Configuration evaluation ocurre antes de que exista necesariamente:

```text
ExecutionScope
```

---

# 151. Metadata compilation

Igualmente, metadata compilation deberá ser independiente del request.

---

# 152. Migration definitions

Las definiciones de migrations tampoco deberían depender arbitrariamente de:

```php
db();
```

para construir su estructura.

---

# 153. Migration execution

El Migration Executor sí tendrá servicios explícitos dentro de su execution context.

---

# 154. Helper diagnostics

En desarrollo podría existir una herramienta que indique:

```text
helper: db
classification: CONTEXTUAL
service: Database
scope required: yes
persistent-runtime-safe: yes
```

---

# 155. Helper Registry

¿Debe existir un `HelperRegistry`?

Para simples funciones:

```text
no necesariamente
```

---

# 156. Avoid overengineering

El Helper System no deberá crear:

```text
HelperManager
HelperFactory
HelperDispatcher
HelperBus
```

sin necesidad.

---

# 157. Architecture proportionality

```text
small helper
→ small implementation
```

---

# 158. Metadata Registry

Sólo podría justificarse para:

```text
documentation
IDE generation
plugin collision detection
diagnostics
```

---

# 159. Helper Definition

Si se implementa:

```php
final readonly class HelperDefinition
{
    public function __construct(
        public string $name,
        public HelperKind $kind,
        public string $package,
        public bool $requiresScope,
    ) {}
}
```

---

# 160. HelperRegistry state

Deberá ser:

```text
immutable after bootstrap
```

---

# 161. Registry ≠ resolved services

Nunca almacenará:

```text
Database instance
Connection
TransactionContext
```

---

# 162. Extension helpers

Plugins podrían distribuir funciones propias.

Pero no deberán poder:

```text
replace core helper silently
```

---

# 163. Name collisions

Una colisión:

```text
db()
```

deberá detectarse durante bootstrap/package loading.

---

# 164. Core helper reservation

Nombres oficiales podrán reservarse.

---

# 165. Plugin namespace preference

Plugins deberán preferir:

```text
namespaced functions
```

para reducir colisiones.

---

# 166. Helper Versioning

Una función global pública es parte del:

```text
Public API Contract
```

---

# 167. Breaking changes

Serán breaking changes:

```text
rename helper
remove helper
change parameters
change return type
change contextual semantics
change exception semantics
change scope behavior
```

---

# 168. Semantic compatibility

Por ejemplo, cambiar:

```text
db()
→ scoped Database
```

a:

```text
db()
→ root singleton Database
```

sería un breaking change aunque la firma PHP sea idéntica.

---

# 169. Deprecation

Los helpers deberán seguir:

```text
DATABASE_DEPRECATION_POLICY
```

cuando aplique.

---

# 170. No silent alias accumulation

VoltStack no mantendrá indefinidamente:

```text
db()
database()
database_manager()
db_manager()
```

para la misma operación.

---

# 171. One canonical helper

Para V1:

```text
db()
```

será suficiente como candidato principal.

---

# 172. Naming Guidelines

Los helpers deberán ser:

```text
short
specific
predictable
unlikely to collide
```

---

# 173. Avoid abbreviations without value

`db()` es ampliamente reconocido.

Pero algo como:

```php
dbcx();
```

sería poco discoverable.

---

# 174. Return Type

La función deberá declarar:

```php
function db(): Database
```

---

# 175. No `mixed`

Evitar:

```php
function db(): mixed
```

---

# 176. IDE support

El archivo helper deberá contener firmas reales.

---

# 177. Static Analysis

PHPStan/Psalm u otras herramientas deberán poder inferir:

```text
Database
```

sin plugins especiales cuando sea posible.

---

# 178. Documentation

Cada helper deberá documentar:

```text
purpose
return type
scope requirements
exceptions
lifecycle
examples
```

---

# 179. Helper error semantics

El helper no deberá envolver todas las excepciones Database.

---

# 180. Example

Si:

```php
db()->connection();
```

produce:

```text
ConnectionFailureException
```

el helper no deberá transformarla en:

```text
HelperException
```

---

# 181. Resolution errors

Sólo los fallos propios de resolución contextual podrán producir una excepción específica.

---

# 182. Pure helper errors

Helpers puros podrán producir:

```text
InvalidArgumentException
InvalidIdentifierException
```

según su dominio.

---

# 183. Telemetry

No deberá emitirse un span por cada:

```php
db();
```

por defecto.

---

# 184. Reason

Esto produciría:

```text
noise
overhead
high event volume
```

sin valor suficiente.

---

# 185. Debug telemetry

En modo diagnóstico podría medirse:

```text
helper resolution failures
scope mismatch
deprecated helper use
```

---

# 186. Database telemetry

Las operaciones reales seguirán instrumentándose en:

```text
Query
Connection
Transaction
ORM
```

---

# 187. Helper telemetry ≠ Database telemetry

Debe mantenerse esa separación.

---

# 188. Security logging

No registrar:

```text
credentials
raw sensitive parameters
tenant secrets
connection passwords
```

desde helpers.

---

# 189. Helper Performance

El helper:

```php
db();
```

añade aproximadamente:

```text
function call
+
current scope resolution
+
container lookup
```

---

# 190. Performance expectation

Este overhead deberá permanecer pequeño.

---

# 191. No premature optimization

No se permitirá:

```text
global cached Database
```

sólo para ahorrar una resolución de Container.

---

# 192. Rule

```text
Scope Correctness
>
Helper Micro-optimization
```

---

# 193. Container optimization

El Container puede optimizar:

```text
scoped service lookup
```

de manera segura.

---

# 194. Compiled Container

VoltStack podrá compilar service resolution.

Entonces:

```php
db();
```

puede resultar muy económico sin romper aislamiento.

---

# 195. Benchmarking

El Performance Testing System deberá poder medir:

```text
direct DI access
Facade resolution
helper resolution
```

por separado.

---

# 196. Benchmark interpretation

Esto sólo mide:

```text
API dispatch overhead
```

no performance completa de Database.

---

# 197. Helper Test Architecture

Se deberán cubrir:

```text
unit tests
integration tests
persistent runtime tests
parallel tests
security tests
public API tests
```

---

# 198. Unit Tests

Para helpers puros:

```text
input
→ deterministic output
```

---

# 199. Contextual helper tests

Para `db()`:

```text
Scope A
→ Database A

Scope B
→ Database B
```

---

# 200. No Scope Test

```php
db();
```

sin scope deberá producir el error esperado.

---

# 201. Closed Scope Test

```text
Scope A closed
→ db()
```

no deberá recuperar `Database A`.

---

# 202. FrankenPHP Test

Simular/ejecutar:

```text
Request A
→ db()

reset

Request B
→ db()
```

y verificar aislamiento.

---

# 203. OpenSwoole Test

Futuro:

```text
Coroutine A → db() → A
Coroutine B → db() → B
```

sin contaminación.

---

# 204. Test Override

```text
Test Scope
→ replace Database
→ db()
→ replacement
```

---

# 205. Parallel Test

```text
Worker A → Fake A
Worker B → Fake B
```

---

# 206. Tenant Test

```text
Tenant A → db() → A context
Tenant B → db() → B context
```

---

# 207. Transaction Test

```php
DB::transaction(function () {
    db()->table(...);
});
```

deberá usar la transacción vigente.

---

# 208. Connection Routing Test

El helper no deberá alterar decisiones de:

```text
writer
replica
shard
```

---

# 209. Security Test

Input malicioso no deberá convertirse en SQL sólo por pasar por un helper.

---

# 210. API Test

Deberá verificarse la firma:

```php
db(): Database
```

como parte del compatibility suite.

---

# 211. Proposed source structure

```text
src/
├── Helper/
│   └── database.php
│
└── Quantum/
    └── Database/
        └── Support/
            └── DatabaseResolver.php
```

si la infraestructura de resolución es compartida.

---

# 212. Better framework-level resolution

Idealmente:

```text
db()
 ↓
Platform CurrentScopeResolver
 ↓
Container
 ↓
Database
```

No será necesario un resolver específico sólo para el helper.

---

# 213. Dependency Direction

```text
Helper
    ↓
Platform Scope Contract
    ↓
Container
    ↓
Database Public Contract
```

---

# 214. Wrong dependency direction

No:

```text
Database internals
→ helper function
```

---

# 215. Internal components

Nunca deberán depender de:

```php
db();
```

cuando puedan recibir sus dependencias explícitamente.

---

# 216. Core Rule

Dentro de `Quantum/Database`:

```text
Dependency Injection
```

deberá ser la estrategia principal.

---

# 217. Helper target

Los helpers están dirigidos principalmente a:

```text
application-facing developer experience
```

no a construir internamente el framework.

---

# 218. Example: bad internal code

```php
final class QueryExecutor
{
    public function execute(...)
    {
        $connection = db()->connection();
    }
}
```

---

# 219. Correct internal code

```php
final class QueryExecutor
{
    public function __construct(
        private ConnectionManager $connections,
    ) {}
}
```

---

# 220. Reason

Esto preserva:

```text
dependency visibility
testability
component boundaries
architecture
```

---

# 221. Helper Usage Policy

Se propone:

```text
APPLICATION CODE
    helpers allowed

FRAMEWORK PUBLIC ADAPTERS
    helpers discouraged

FRAMEWORK CORE
    helpers prohibited for dependency acquisition
```

---

# 222. Compile-time enforcement

Architecture tests podrán detectar uso de:

```text
db()
```

dentro de ciertos namespaces internos.

---

# 223. Example rule

```text
VoltStack\Quantum\Database\Internal\*
must not call global db()
```

---

# 224. Exceptions

Sólo adapters explícitamente diseñados como developer-facing podrán hacerlo.

---

# 225. Database Helper Manifest

Opcionalmente podrá generarse:

```json
{
  "name": "db",
  "kind": "contextual",
  "return": "VoltStack\\Quantum\\Database\\Database",
  "requires_scope": true,
  "stability": "stable"
}
```

---

# 226. Uses

Esto puede alimentar:

```text
documentation
IDE tooling
API compatibility checks
deprecation tooling
```

---

# 227. Helper Stability

Estados posibles:

```text
INTERNAL
EXPERIMENTAL
PUBLIC_PREVIEW
PUBLIC_STABLE
DEPRECATED
```

---

# 228. V1 helper

Cuando se estabilice:

```text
db()
→ PUBLIC_STABLE
```

---

# 229. Helper extension policy

Plugins no deberán registrar helpers globales mediante un runtime registry dinámico.

Las funciones PHP deben definirse de forma determinista durante bootstrap/autoload.

---

# 230. Runtime mutation

No:

```text
register_helper_at_runtime(...)
```

para modificar semántica de helpers existentes.

---

# 231. Package discovery

Composer/package discovery podrá cargar archivos helper de plugins cuando la política lo permita.

---

# 232. Collision detection

El sistema de paquetes deberá validar nombres reservados.

---

# 233. Database helper namespace

Los helpers no globales pueden ser namespaced:

```php
use function VoltStack\Database\db_identifier;
```

si alguna API especializada lo justifica.

---

# 234. Global vs namespaced policy

Se recomienda:

```text
high-frequency universal helper
→ potentially global

specialized helper
→ namespaced

internal helper
→ private function/class
```

---

# 235. V1 minimal public set

Propuesta:

```text
db()
```

y ningún otro helper global específico de Database inicialmente.

---

# 236. Why so small

Porque ya existen:

```text
DB Facade
Schema Facade
Model API
Query Builder
Repository
EntityManager
Dependency Injection
```

---

# 237. Ergonomic coverage

Con sólo:

```php
db();
```

puede accederse de forma discoverable a:

```php
db()->connection();
db()->table('users');
db()->query();
db()->transaction(...);
db()->schema();
db()->entityManager();
db()->capabilities();
```

sin crear siete helpers.

---

# 238. API coherence

Esto produce:

```text
one entry point
→ typed object
→ IDE discovery
```

---

# 239. Facade equivalent

En paralelo:

```php
DB::connection();
DB::table('users');
DB::transaction(...);
```

---

# 240. User choice

VoltStack permitirá elegir entre:

```text
DI
Facade
Helper
Model API
Repository
```

según el contexto.

---

# 241. No semantic difference

Para una operación equivalente deberá cumplirse:

```text
DI semantics
=
Facade semantics
=
Helper semantics
```

---

# 242. Surface ≠ Engine

Esta separación es fundamental.

---

# 243. Formal model

Sea:

```text
H
```

un helper contextual,

```text
C
```

el ExecutionScope,

y:

```text
S
```

un servicio scoped.

Entonces:

```text
H(C) → resolve(S, C)
```

---

# 244. No global ownership

Debe cumplirse:

```text
owns(H, S) = false
```

---

# 245. Scope identity

Para:

```text
C1 ≠ C2
```

si `S` es scoped:

```text
H(C1) ≠ H(C2)
```

en términos de instancia mutable.

---

# 246. Root immutable exception

Si un helper retorna un value/service inmutable root:

```text
R(C1) = R(C2)
```

puede ser válido.

---

# 247. Pure helper model

Para helper puro:

```text
H(x) = y
```

y:

```text
H(x) = H(x)
```

independientemente de:

```text
request
tenant
transaction
worker
```

---

# 248. Contextual helper model

Para contextual:

```text
H(C, x) = y
```

aunque `C` no aparezca explícitamente en la firma.

---

# 249. Hidden context warning

Precisamente por eso los helpers contextuales deberán ser pocos.

---

# 250. Contextual complexity rule

Cuanto más importante sea el contexto para comprender la operación:

```text
more explicit API preferred
```

---

# 251. Example

Preferible:

```php
DB::transaction(...)
```

a:

```php
tx(...)
```

porque la primera expresa claramente el dominio.

---

# 252. Another example

Preferible:

```php
db()->connection('analytics')
```

a:

```php
conn('analytics')
```

---

# 253. Helper convenience ceiling

Un helper no deberá comprimir tanto la sintaxis que destruya significado.

---

# 254. Database Helper Invariants

## DB-HELP-001

Helper ≠ Service.

## DB-HELP-002

Helper ≠ Facade.

## DB-HELP-003

Helper ≠ Container.

## DB-HELP-004

Helper ≠ Global Service Locator.

## DB-HELP-005

Helper ≠ Database Runtime.

## DB-HELP-006

Helper ≠ Connection Manager.

## DB-HELP-007

Helper ≠ Transaction Manager.

## DB-HELP-008

Helper ≠ ORM.

## DB-HELP-009

Helper ≠ Query Engine.

## DB-HELP-010

Helper ≠ SQL Compiler.

---

# 255. State Invariants

## DB-HELP-011

Contextual helper no poseerá Database scoped.

## DB-HELP-012

Contextual helper no poseerá Connection.

## DB-HELP-013

Contextual helper no poseerá EntityManager.

## DB-HELP-014

Contextual helper no poseerá UnitOfWork.

## DB-HELP-015

Contextual helper no poseerá IdentityMap.

## DB-HELP-016

Contextual helper no poseerá TransactionContext.

## DB-HELP-017

Contextual helper no poseerá TenantContext.

## DB-HELP-018

Contextual helper no poseerá ShardContext.

## DB-HELP-019

Contextual helper no poseerá QueryContext.

## DB-HELP-020

Contextual helper no utilizará static service cache.

---

# 256. Scope Invariants

## DB-HELP-021

Contextual helper resolverá Current ExecutionScope.

## DB-HELP-022

No Current Scope ≠ Previous Scope.

## DB-HELP-023

Closed Scope no será reutilizado.

## DB-HELP-024

Helper y Facade compartirán scope.

## DB-HELP-025

Helper y Model API compartirán scope.

## DB-HELP-026

Helper no creará scopes implícitos por defecto.

## DB-HELP-027

Scope lifecycle pertenecerá al runtime.

## DB-HELP-028

Helper result podrá ser scope-bound.

## DB-HELP-029

Scope-bound result no migrará silenciosamente entre scopes.

## DB-HELP-030

Process Lifetime ≠ Helper Service Lifetime.

---

# 257. API Invariants

## DB-HELP-031

V1 favorecerá un helper `db()` pequeño y tipado.

## DB-HELP-032

`db()` retornará `Database`.

## DB-HELP-033

`db()` no cambiará retorno según argumentos en V1.

## DB-HELP-034

Named connection se obtendrá mediante `db()->connection()`.

## DB-HELP-035

Helper APIs tendrán firmas explícitas.

## DB-HELP-036

Helper APIs evitarán `mixed`.

## DB-HELP-037

Helper names serán estables.

## DB-HELP-038

Helper removal será breaking change.

## DB-HELP-039

Helper semantic change podrá ser breaking change.

## DB-HELP-040

Helpers públicos seguirán deprecation policy.

---

# 258. Query/Security Invariants

## DB-HELP-041

Helper no realizará manual SQL escaping.

## DB-HELP-042

Helper no reemplazará parameter binding.

## DB-HELP-043

Helper no hará identifier quoting específico de vendor.

## DB-HELP-044

Helper no introducirá SQL raw ambiguo.

## DB-HELP-045

Raw Expression ≠ Parameter Value.

## DB-HELP-046

Identifier ≠ Parameter Value.

## DB-HELP-047

Helper no bypassará Query Validation.

## DB-HELP-048

Helper no bypassará Capability System.

## DB-HELP-049

Helper no decidirá por vendor name cuando exista capability.

## DB-HELP-050

Version ≠ Capability.

---

# 259. Runtime Invariants

## DB-HELP-051

`db()` será seguro bajo FrankenPHP.

## DB-HELP-052

`db()` será adaptable a RoadRunner.

## DB-HELP-053

`db()` será adaptable a OpenSwoole.

## DB-HELP-054

Coroutine contexts no compartirán mutable Database state.

## DB-HELP-055

Queue jobs no compartirán DatabaseContext.

## DB-HELP-056

Long-running CLI operations no compartirán state accidentalmente.

## DB-HELP-057

Helper local static variables no almacenarán servicios scoped.

## DB-HELP-058

Contextual cache pertenecerá al scope/container.

## DB-HELP-059

Runtime adapter resolverá contexto actual.

## DB-HELP-060

Helper no conocerá detalles específicos del runtime.

---

# 260. ORM Invariants

## DB-HELP-061

No habrá `uow()` global en V1.

## DB-HELP-062

No habrá `identity_map()` global en V1.

## DB-HELP-063

No habrá `em()` global en V1.

## DB-HELP-064

No habrá `repo()` global en V1.

## DB-HELP-065

Model API seguirá siendo la API ORM convenience principal.

## DB-HELP-066

Repository seguirá siendo primera clase.

## DB-HELP-067

Helper no creará un segundo ORM engine.

## DB-HELP-068

Helper no alterará EntityManager semantics.

## DB-HELP-069

Helper no alterará UoW semantics.

## DB-HELP-070

Helper no alterará IdentityMap semantics.

---

# 261. Transaction Invariants

## DB-HELP-071

Helper no será propietario de transaction.

## DB-HELP-072

Helper no hará commit implícito.

## DB-HELP-073

Helper no hará rollback implícito fuera del Transaction System.

## DB-HELP-074

Helper no redefinirá nested transaction policy.

## DB-HELP-075

Helper no redefinirá retry policy.

## DB-HELP-076

UNKNOWN transaction outcome será preservado.

## DB-HELP-077

flush() ≠ commit().

## DB-HELP-078

Savepoint ≠ Independent Transaction.

## DB-HELP-079

Transaction affinity será preservada.

## DB-HELP-080

Facade y Helper observarán la misma TransactionContext.

---

# 262. Testing Invariants

## DB-HELP-081

Test overrides serán scope-local.

## DB-HELP-082

Helper no mantendrá fake global.

## DB-HELP-083

Parallel tests estarán aislados.

## DB-HELP-084

Persistent runtime leakage será probado.

## DB-HELP-085

Tenant leakage será probado.

## DB-HELP-086

No-scope behavior será probado.

## DB-HELP-087

Closed-scope behavior será probado.

## DB-HELP-088

Helper signatures formarán parte del API compatibility test.

## DB-HELP-089

Pure helpers serán deterministic-testable.

## DB-HELP-090

Fake evidence no sustituirá DBMS integration evidence.

---

# 263. Developer Experience Invariants

## DB-HELP-091

Helper deberá reducir ceremonia real.

## DB-HELP-092

Helper no existirá únicamente para ahorrar pocos caracteres.

## DB-HELP-093

Helper deberá ser discoverable.

## DB-HELP-094

Helper deberá tener nombre inequívoco.

## DB-HELP-095

Helper deberá minimizar namespace pollution.

## DB-HELP-096

Specialized helpers preferirán namespace.

## DB-HELP-097

Internal helpers no serán API pública.

## DB-HELP-098

Helper no comprimirá semántica importante hasta hacerla opaca.

## DB-HELP-099

One canonical helper será preferible a múltiples aliases.

## DB-HELP-100

API simplicity será un objetivo explícito.

---

# 264. Framework Architecture Invariants

## DB-HELP-101

Database core no utilizará helpers para adquirir dependencias internas.

## DB-HELP-102

Database core utilizará Dependency Injection.

## DB-HELP-103

Architecture tests podrán prohibir `db()` en namespaces internos.

## DB-HELP-104

Helper loading no resolverá servicios scoped.

## DB-HELP-105

Helper bootstrap será deterministic.

## DB-HELP-106

Plugin helper collision será detectable.

## DB-HELP-107

Plugin no reemplazará helper core silenciosamente.

## DB-HELP-108

Helper Registry, si existe, almacenará metadata y no servicios.

## DB-HELP-109

Helper System permanecerá proporcionalmente simple.

## DB-HELP-110

Helper System no se convertirá en otro Container.

---

# 265. Additional Invariants

## DB-HELP-111

DI, Facade y Helper convergerán en el mismo Database engine.

## DB-HELP-112

Surface API ≠ Persistence Engine.

## DB-HELP-113

Helper no decidirá replica routing.

## DB-HELP-114

Helper no decidirá shard routing.

## DB-HELP-115

Helper no resolverá tenant directamente.

## DB-HELP-116

Helper no implementará failover.

## DB-HELP-117

Helper no implementará connection pooling.

## DB-HELP-118

Helper no implementará retries.

## DB-HELP-119

Helper no implementará SQL dialect behavior.

## DB-HELP-120

Helper convenience nunca reducirá garantías semánticas silenciosamente.

---

# 266. Anti-patrones

## 266.1 Static Database

```php
function db()
{
    static $database;

    return $database;
}
```

Prohibido.

---

## 266.2 Global Database

```php
global $database;
```

Prohibido.

---

## 266.3 Last Container

```php
return Container::$last;
```

Prohibido.

---

## 266.4 Previous Request Fallback

```text
no current scope
→ previous request
```

Prohibido.

---

## 266.5 Hidden Transaction

```php
save($entity);
```

que secretamente abre y confirma una transaction global.

Evitar.

---

## 266.6 Manual Escape Helper

```php
db_escape($input);
```

No será API recomendada.

---

## 266.7 Generic Raw Helper

```php
raw($input);
```

Evitar.

---

## 266.8 Vendor Helper

```php
is_mysql();
```

Evitar como arquitectura Database.

---

## 266.9 Version Capability Logic

```php
if (mysql_version() >= ...)
```

en helpers.

Evitar.

---

## 266.10 ORM Helper Explosion

```text
em()
uow()
repo()
entity()
identity_map()
```

Evitar.

---

## 266.11 Query DSL Global Explosion

```text
select()
from()
where()
join()
eq()
gt()
and_()
or_()
```

Evitar.

---

## 266.12 Helper as Service Locator

```php
service('database');
```

usado indiscriminadamente por Database internals.

Evitar.

---

## 266.13 Framework Core Calling `db()`

Evitar para dependency acquisition.

---

## 266.14 Capturing `db()` in singleton constructor

Prohibido cuando produzca captive scoped dependency.

---

## 266.15 Helper-specific Fake Registry

Prohibido.

---

## 266.16 Hidden Connection Acquisition

Evitar cuando la API aparentemente sólo construye una query.

---

## 266.17 Hidden Tenant Mutation

Prohibido.

---

## 266.18 Hidden Shard Mutation

Prohibido.

---

## 266.19 Hundreds of aliases

Evitar.

---

## 266.20 Helpers without return types

Evitar.

---

# 267. Modelo recomendado para V1

El Database Helper System de VoltStack V1 será deliberadamente pequeño.

Superficie propuesta:

```php
db(): Database
```

Uso:

```php
db()->table('users')
    ->where('active', true)
    ->get();
```

```php
db()->transaction(function (): void {
    // ...
});
```

```php
db()->connection('analytics');
```

```php
db()->schema();
```

```php
db()->entityManager();
```

---

# 268. Equivalencia con Facade

```php
DB::table('users');
```

equivale conceptualmente a:

```php
db()->table('users');
```

---

# 269. Equivalencia con DI

Y ambos deberán utilizar el mismo motor que:

```php
final class Service
{
    public function __construct(
        private Database $database,
    ) {}

    public function execute(): void
    {
        $this->database
            ->table('users')
            ->get();
    }
}
```

---

# 270. Architecture

```text
                  Application
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
       DI            DB::            db()
        │            Facade           Helper
        │              │              │
        └──────────────┼──────────────┘
                       ▼
               ExecutionScope
                       │
                       ▼
                    Container
                       │
                       ▼
                    Database
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
     Query Engine Transaction     ORM/Schema
                       │
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

---

# 271. Persistent Runtime Model

```text
FrankenPHP Worker
│
├── Request A
│   ├── Scope A
│   └── db() → Database A
│
├── reset
│
└── Request B
    ├── Scope B
    └── db() → Database B
```

La función:

```text
db()
```

permanece definida durante todo el proceso.

Pero:

```text
Database A
```

no permanece almacenada en ella.

---

# 272. Fundamental distinction

```text
Function Lifetime
=
Process Lifetime
```

puede ser válido.

Pero:

```text
Resolved Scoped Service Lifetime
=
Scope Lifetime
```

debe mantenerse.

---

# 273. Security model

```text
db()
```

no aumenta privilegios.

Por tanto:

```text
Permissions(db())
=
Permissions(Current Database Context)
```

---

# 274. Context model

Formalmente:

```text
db(C) = resolve(Database, C)
```

donde:

```text
C = Current ExecutionScope
```

---

# 275. Isolation

Para dos scopes diferentes:

```text
C1 ≠ C2
```

si Database es scoped:

```text
db(C1) ≠ db(C2)
```

como instancia mutable.

---

# 276. No ownership

Debe cumplirse:

```text
owns(db, Database) = false
```

---

# 277. Helper correctness

Un helper contextual será correcto si:

```text
CurrentScopeResolved
∧
ServiceResolved
∧
ScopeValid
∧
NoCrossScopeLeak
```

---

# 278. Helper admission formula

Un nuevo helper debería añadirse sólo cuando:

```text
ErgonomicValue
>
APIComplexity
+
CollisionRisk
+
MaintenanceCost
+
SemanticAmbiguity
```

---

# 279. Acceptance Criteria

El Database Helper System estará preparado para V1 cuando pueda demostrarse que:

1. existe un helper `db()` tipado;
2. `db()` resuelve Database desde el scope vigente;
3. `db()` no almacena Database estáticamente;
4. `db()` no almacena conexiones;
5. `db()` no almacena EntityManager;
6. `db()` no almacena TransactionContext;
7. no existe fallback al scope anterior;
8. un scope cerrado no puede reutilizarse;
9. Request A y Request B están aislados;
10. FrankenPHP tiene pruebas específicas;
11. tests pueden reemplazar Database dentro de su propio scope;
12. parallel tests permanecen aislados;
13. Facade y Helper comparten contexto;
14. Model API y Helper comparten contexto;
15. TransactionContext se preserva entre las distintas superficies;
16. Multitenancy no requiere lógica dentro del helper;
17. sharding no requiere lógica dentro del helper;
18. read/write routing no requiere lógica dentro del helper;
19. el core Database no depende de helpers para DI;
20. no existen helpers inseguros de SQL escaping;
21. no existe un `raw()` global ambiguo;
22. no existe proliferación de helpers ORM;
23. IDE puede inferir `Database`;
24. helper loading no resuelve servicios durante bootstrap;
25. el sistema puede adaptarse a RoadRunner/OpenSwoole sin cambiar `db()`.

---

# 280. Decisiones V1

## 280.1 Helper principal

Se adopta como candidato oficial:

```php
db(): Database
```

---

## 280.2 Named connections

Se utilizará:

```php
db()->connection('analytics');
```

y no:

```php
db('analytics');
```

para mantener un retorno único y fuertemente tipado.

---

## 280.3 Transactions

No se añadirá inicialmente:

```php
transaction();
```

Se utilizará:

```php
DB::transaction();
```

o:

```php
db()->transaction();
```

---

## 280.4 Schema

No se añadirá inicialmente:

```php
schema();
```

Se utilizará:

```php
Schema::
```

o:

```php
db()->schema();
```

---

## 280.5 ORM

No se añadirán inicialmente:

```text
em()
uow()
repo()
identity_map()
```

---

## 280.6 Query expressions

No se añadirá un conjunto global de helpers AST/SQL.

---

## 280.7 Raw SQL

No se añadirá un:

```php
raw();
```

global ambiguo.

---

## 280.8 Internal framework

El core de Database utilizará:

```text
Dependency Injection
```

y no `db()` para adquirir dependencias.

---

# 281. Principio final

El Helper System debe mejorar la experiencia del desarrollador sin convertir conveniencia en dependencia invisible.

La regla definitiva será:

> **Un helper de VoltStack será una abreviación de una arquitectura correcta, no una excepción a esa arquitectura.**

Por tanto:

```text
Helper
≠
Service
```

```text
Helper
≠
Facade
```

```text
Helper
≠
Container
```

```text
Helper
≠
Global Service Locator
```

```text
Helper
≠
Static Database
```

```text
Helper
≠
Connection
```

```text
Helper
≠
Transaction
```

```text
Helper
≠
Tenant Context
```

```text
Helper
≠
Shard Context
```

```text
Helper
≠
SQL Escaper
```

```text
Helper
≠
Dialect
```

```text
Helper
≠
Capability Resolver
```

y:

```text
Convenience
≠
Hidden State
```

```text
Short Syntax
≠
Reduced Safety
```

```text
Global Function
≠
Global Mutable Service
```

```text
Process Lifetime
≠
Database Scope Lifetime
```

Finalmente:

```text
VoltStack Database Helper System
=
Minimal API Surface
+
Strong Typing
+
Execution Scope Resolution
+
Shared Context Infrastructure
+
Zero Scoped Static State
+
Persistent Runtime Safety
+
IDE Discoverability
+
Explicit Semantics
+
Security Preservation
+
Testability
+
Developer Convenience
```

La arquitectura V1 queda deliberadamente centrada en:

```php
db(): Database
```

como un punto de acceso compacto y tipado que complementa:

```text
Dependency Injection
DB Facade
Schema Facade
Model API
Repositories
```

sin sustituir ninguno de ellos.

---

# 282. Siguiente documento

```text
305_DATABASE_MODEL_DEVELOPER_EXPERIENCE.md
```

El siguiente documento definirá la experiencia completa que VoltStack ofrecerá al desarrollador al trabajar con Models y entidades:

```text
Model API
Active Record convenience layer
Entity API
Repository API
EntityManager
attributes
casts
relationships
querying
persistence
mass assignment
serialization boundaries
IDE typing
code completion
errors
debugging
testing
```

manteniendo la regla arquitectónica:

> **La experiencia Laravel-like del Model será una capa ergonómica sobre el mismo ORM Engine de VoltStack; nunca constituirá un segundo ORM ni un segundo Persistence Engine.**

Por tanto, deberá preservarse:

```text
Model API
        ─┐
         │
Repository├──→ EntityManager
         │         ↓
Direct ORM┘     UnitOfWork
                   ↓
            Persistence Engine
                   ↓
              Query Engine
                   ↓
            Execution Engine
```

y no:

```text
Model API → Persistence Engine A

Repository → Persistence Engine B
```