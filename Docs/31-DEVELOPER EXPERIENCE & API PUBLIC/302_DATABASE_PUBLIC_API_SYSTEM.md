# 302_DATABASE_PUBLIC_API_SYSTEM.md

# VoltStack Quantum Database
## Database Public API System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 302 — Database Public API System  
**Bloque:** 31 — Developer Experience / Public API  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `301_DATABASE_DATABASE_CAPABILITY_DISCOVERY_SYSTEM.md`  
**Siguiente documento:** `303_DATABASE_FACADE_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial del **Database Public API System** de:

```text
VoltStack/Quantum/Database
```

responsable de establecer la superficie pública mediante la cual aplicaciones, paquetes, módulos y desarrolladores interactúan con VoltStack Database.

Hasta este punto, Database ha sido diseñado como un sistema internamente sofisticado:

```text
Driver
Connection
Platform
Dialect
Query AST
Semantic Analysis
Optimizer
Planner
Compiler
Executor
Schema
Migration
ORM
UnitOfWork
IdentityMap
Hydration
Relationships
Transactions
Cache
Telemetry
Security
Resilience
Runtime
Extensions
Capabilities
```

Sin embargo, esa complejidad interna no deberá trasladarse directamente al código de aplicación.

La regla central será:

> **La API pública de VoltStack Database deberá ofrecer una interfaz simple, coherente, fuertemente tipada y predecible sobre una arquitectura interna rigurosamente separada, sin permitir que la comodidad pública destruya las fronteras del sistema.**

Formalmente:

```text
Public API
=
Stable Contracts
+
Developer Entry Points
+
Typed Fluent APIs
+
Scoped Context Resolution
+
Safe Defaults
+
Explicit Advanced APIs
+
Consistent Errors
+
IDE Discoverability
+
Backward Compatibility
```

y:

```text
Internal Complexity
≠
Public API Complexity
```

pero también:

```text
Simple API
≠
Hidden Semantic Ambiguity
```

---

# 2. Objetivo estratégico

VoltStack busca combinar principalmente dos propiedades:

```text
Laravel-like ergonomics
+
Doctrine-like architectural rigor
```

sin copiar internamente ninguno de los dos modelos.

La experiencia deseada será:

```php
$user = User::find(42);
```

o:

```php
$user = $users->find(42);
```

mientras ambas APIs convergen en:

```text
Model API / Repository
        ↓
EntityManager
        ↓
UnitOfWork
        ↓
Persistence Engine
        ↓
Query Engine
        ↓
Execution Engine
        ↓
Connection
        ↓
Driver
```

No existirán dos motores ORM independientes.

---

# 3. Public API ≠ Internal Architecture

La API pública representa una frontera.

```text
Application
    ↓
Public Database API
    ↓
Internal Database Architecture
```

El desarrollador no deberá necesitar conocer:

```text
PhysicalQueryPlan
CompilerPipeline
DriverConnectionState
HydrationAssemblyRegistry
IdentityReservation
CapabilityEvidenceSnapshot
```

para realizar operaciones comunes.

---

# 4. Public API ≠ Thin Wrapper Around Internals

Tampoco se expondrá indiscriminadamente cada clase interna.

Incorrecto:

```text
public API
=
every public PHP class
```

Correcto:

```text
public API
=
explicitly supported developer-facing contracts
```

---

# 5. API Stability Boundary

Una clase PHP puede ser técnicamente accesible y aun así pertenecer a:

```text
@internal
```

La compatibilidad de VoltStack se aplicará principalmente a la superficie declarada como:

```text
Public API
```

---

# 6. Objetivos

El sistema deberá proporcionar:

1. entry points claros;
2. API fuertemente tipada;
3. autocompletado IDE;
4. nombres consistentes;
5. comportamiento predecible;
6. defaults seguros;
7. acceso simple para operaciones comunes;
8. APIs avanzadas explícitas;
9. compatibilidad con dependency injection;
10. Facades opcionales;
11. Helpers limitados;
12. Model API;
13. Repository API;
14. EntityManager API;
15. Query API;
16. Schema API;
17. Migration API;
18. Transaction API;
19. Connection API;
20. diagnostics API;
21. streaming API;
22. bulk API;
23. extensibilidad controlada;
24. scoped state;
25. persistent-runtime safety;
26. errores consistentes;
27. backward compatibility;
28. documentación descubrible;
29. mínima dependencia de strings mágicos;
30. cero dependencia obligatoria de Facades.

---

# 7. Filosofía de diseño

La API seguirá:

```text
Simple things
should be simple.

Advanced things
should be possible.

Dangerous things
should be explicit.
```

---

# 8. Tres niveles de API

VoltStack Database podrá presentar tres niveles conceptuales:

```text
Level 1
Convenience API

Level 2
Structured API

Level 3
Advanced API
```

---

# 9. Level 1 — Convenience API

Diseñada para la mayoría del código de aplicación.

Ejemplos:

```php
$user = User::find(42);

$users = User::query()
    ->where('active', true)
    ->get();
```

o:

```php
Database::transaction(function () {
    // ...
});
```

---

# 10. Level 2 — Structured API

Diseñada para servicios y código de dominio con dependency injection.

Ejemplo:

```php
final class UserService
{
    public function __construct(
        private UserRepository $users,
        private TransactionManager $transactions,
    ) {}
}
```

---

# 11. Level 3 — Advanced API

Para infraestructura, paquetes y casos especiales.

Ejemplos:

```text
Query AST
Query Planner
Capability Requirements
Schema Model
Execution Options
Streaming
Custom Types
Custom Query Extensions
```

---

# 12. Convenience ≠ Reduced Correctness

Level 1 no podrá saltarse:

```text
UnitOfWork
IdentityMap
Transaction Rules
Security
Tenant Context
Query Validation
Capability Checks
```

---

# 13. Structured API ≠ More Correct API

Todos los niveles deberán converger en los mismos motores internos.

---

# 14. Public API Layers

```text
Application
│
├── Model API
├── Repository API
├── EntityManager API
├── Query API
├── Transaction API
├── Schema API
├── Connection API
├── Bulk API
├── Streaming API
├── Diagnostics API
│
├── Facades
└── Helpers
        │
        ▼
Public Contracts
        │
        ▼
Internal Database Services
```

---

# 15. Primary API Domains

La superficie pública se dividirá en dominios.

```text
Database
├── ORM
├── Query
├── Transaction
├── Connection
├── Schema
├── Migration
├── Bulk
├── Streaming
├── Pagination
├── Diagnostics
└── Extension
```

---

# 16. Root Database API

Podrá existir un punto de entrada:

```php
Database
```

como API coordinadora de alto nivel.

Ejemplo conceptual:

```php
$db = $container->get(Database::class);
```

---

# 17. Database ≠ God Object

`Database` no implementará internamente todos los subsistemas.

Será una fachada estructurada sobre servicios especializados.

---

# 18. Root Contract

Conceptualmente:

```php
interface Database
{
    public function connection(?string $name = null): Connection;

    public function query(): QueryFactory;

    public function transactions(): TransactionManager;

    public function schema(?string $connection = null): SchemaManager;

    public function entityManager(): EntityManager;

    public function capabilities(): CapabilityView;
}
```

La API exacta podrá evolucionar, pero la separación deberá mantenerse.

---

# 19. Dependency Injection como API primaria

La forma arquitectónicamente preferida será:

```php
public function __construct(
    Database $database,
) {}
```

o dependencias más específicas:

```php
public function __construct(
    UserRepository $users,
    TransactionManager $transactions,
) {}
```

---

# 20. Depend on Narrow Contracts

Preferentemente:

```php
public function __construct(
    TransactionManager $transactions,
)
```

en lugar de:

```php
public function __construct(
    Database $database,
)
```

si sólo se necesitan transacciones.

---

# 21. Facades

VoltStack podrá ofrecer Facades para ergonomía:

```php
DB::transaction(...);
```

pero:

```text
Facade
≠
Core Database Architecture
```

Se especificará en:

```text
303_DATABASE_FACADE_SYSTEM.md
```

---

# 22. Helpers

Podrán existir helpers cuidadosamente seleccionados.

Pero:

```text
Helper
≠
Hidden Global Mutable State
```

Se especificará en:

```text
304_DATABASE_HELPER_SYSTEM.md
```

---

# 23. Model API

La API orientada a modelos podrá permitir:

```php
$user = User::find(1);
```

```php
$users = User::query()
    ->where('status', UserStatus::Active)
    ->orderBy('created_at', 'desc')
    ->get();
```

```php
$user->save();
```

```php
$user->delete();
```

---

# 24. Model API ≠ Active Record Persistence Engine

La llamada:

```php
$user->save();
```

deberá terminar conceptualmente en:

```text
Model API
   ↓
ModelContextResolver
   ↓
EntityManager
   ↓
UnitOfWork
   ↓
Persistence Engine
```

No:

```text
Model
   ↓
construct SQL
   ↓
PDO
```

---

# 25. No Static ORM State

Aunque se permita:

```php
User::query()
```

no deberá existir:

```php
User::$entityManager
```

como estado mutable global.

---

# 26. ModelContextResolver

La sintaxis estática deberá resolver:

```text
current scoped database context
```

mediante:

```text
ModelContextResolver
```

---

# 27. Persistent Runtime Safety

En FrankenPHP:

```text
Request A
User::query()
```

no podrá reutilizar accidentalmente:

```text
EntityManager
TenantContext
TransactionContext
IdentityMap
```

del Request B.

---

# 28. Model Developer Experience

Será desarrollada específicamente en:

```text
305_DATABASE_MODEL_DEVELOPER_EXPERIENCE.md
```

---

# 29. Repository API

VoltStack soportará repositorios explícitos.

Ejemplo:

```php
final class UserRepository extends Repository
{
    public function active(): array
    {
        return $this->query()
            ->where('active', true)
            ->get();
    }
}
```

---

# 30. Repository ≠ Query Builder

Repository conoce:

```text
Entity semantics
```

Query Builder conoce:

```text
Query semantics
```

---

# 31. Repository ≠ EntityManager

El Repository utilizará EntityManager.

No lo sustituirá.

---

# 32. Repository Interface

Conceptualmente:

```php
interface Repository
{
    public function find(mixed $id): ?object;

    public function findOrFail(mixed $id): object;

    public function query(): EntityQuery;

    public function persist(object $entity): void;

    public function remove(object $entity): void;
}
```

---

# 33. persist() Semantics

```php
$repository->persist($user);
```

significa:

```text
register for persistence
```

No:

```text
execute INSERT immediately
```

---

# 34. remove() Semantics

Igualmente:

```php
$repository->remove($user);
```

no implica necesariamente:

```text
DELETE executed now
```

---

# 35. EntityManager API

La API podrá incluir:

```php
$em->persist($entity);
$em->remove($entity);
$em->flush();
$em->clear();
```

---

# 36. flush() ≠ commit()

Regla pública explícita:

```text
flush()
≠
commit()
```

---

# 37. flush()

Significa:

> sincronizar cambios pendientes del UnitOfWork con la base de datos dentro del contexto transaccional existente.

---

# 38. commit()

Pertenece al sistema transaccional.

---

# 39. Explicit Transaction API

La API recomendada será:

```php
$transactions->run(function () use ($em) {
    // ...

    $em->flush();
});
```

---

# 40. Convenience Transaction API

También podrá existir:

```php
Database::transaction(function () {
    // ...
});
```

---

# 41. Transaction Callback Return

Ejemplo:

```php
$user = $transactions->run(function () use ($users) {
    return $users->create(...);
});
```

---

# 42. Transaction API ≠ Automatic Retry Everywhere

Retries deberán depender de políticas explícitas.

---

# 43. Retry API

Podrá existir:

```php
$transactions->run(
    callback: $operation,
    retry: RetryPolicy::deadlocks(maxAttempts: 3),
);
```

---

# 44. UNKNOWN Commit Outcome

La API deberá representar explícitamente resultados transaccionales desconocidos.

Nunca:

```text
commit exception
→ automatically retry callback
```

cuando el commit pudo haber sido aplicado.

---

# 45. Query API

VoltStack tendrá Query Builder fluido.

Ejemplo:

```php
$users = $db->query()
    ->from('users')
    ->select('id', 'name')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

---

# 46. Query Builder ≠ SQL Builder

La API construirá:

```text
Query Model / AST
```

No strings SQL concatenados.

---

# 47. Public Query API

Conceptualmente:

```text
QueryFactory
   ↓
SelectQuery
InsertQuery
UpdateQuery
DeleteQuery
```

---

# 48. Query Immutability

Se favorecerá un modelo:

```php
$query = $query->where(...);
```

con estructuras internas inmutables cuando resulte práctico.

---

# 49. Fluent API

La ergonomía podrá permitir:

```php
$query
    ->where(...)
    ->where(...)
    ->orderBy(...)
    ->limit(...);
```

sin sacrificar AST tipado.

---

# 50. Strings vs Typed References

La API simple podrá aceptar:

```php
->where('email', $email)
```

pero internamente deberá normalizar a:

```text
ColumnReference
ParameterExpression
Predicate
```

---

# 51. Advanced Typed Query API

También podrá existir:

```php
$query->where(
    Column::of('users.email')->eq(
        Parameter::value($email)
    )
);
```

---

# 52. Simple API ≠ String Concatenation

```php
where('email', $email)
```

no deberá producir:

```php
"email = '$email'"
```

---

# 53. Parameterization by Default

Valores deberán convertirse en parámetros.

---

# 54. Identifier Safety

Los identifiers deberán validarse o representarse mediante objetos apropiados.

---

# 55. Raw SQL Escape Hatch

Podrá existir una API explícita:

```php
RawExpression::trusted(...)
```

o equivalente.

---

# 56. Raw ≠ Default

Nunca será la ruta normal.

---

# 57. Unsafe API Naming

Las APIs que reducen garantías deberán ser visiblemente explícitas.

Ejemplo conceptual:

```php
unsafeRaw(...)
```

en vez de:

```php
raw(...)
```

cuando corresponda.

---

# 58. Query Execution API

Podrá distinguir:

```php
$query->get();
$query->first();
$query->firstOrFail();
$query->exists();
$query->count();
$query->cursor();
$query->stream();
```

---

# 59. get() Semantics

`get()` normalmente materializará resultados.

---

# 60. stream() Semantics

`stream()` no deberá ocultar:

```text
open cursor
connection ownership
resource lifecycle
```

---

# 61. Streaming Resource API

Preferentemente soportará patrones seguros:

```php
$stream = $query->stream();

try {
    foreach ($stream as $row) {
        // ...
    }
} finally {
    $stream->close();
}
```

y cierre automático cuando sea posible.

---

# 62. Cursor ≠ Collection

La API no deberá hacerlos intercambiables semánticamente.

---

# 63. Result Shapes

La API podrá solicitar:

```text
ENTITY
ENTITY_COLLECTION
SCALAR
SCALAR_LIST
ARRAY
TUPLE
DTO
PROJECTION
```

---

# 64. Typed Result APIs

Ejemplo conceptual:

```php
$query->as(UserSummary::class)->get();
```

cuando exista mapping/projection válido.

---

# 65. ORM Query API

Podrá permitir:

```php
$users->query()
    ->where('status', UserStatus::Active)
    ->with('profile')
    ->paginate();
```

---

# 66. Entity Query ≠ Table Query

La API deberá diferenciar conceptualmente:

```text
EntityQuery
```

de:

```text
DatabaseQuery
```

aunque compartan infraestructura.

---

# 67. Entity Field ≠ Database Column

El ORM resolverá mapping.

---

# 68. Relationship API

Podrá ofrecer:

```php
$user->posts;
```

si lazy loading está permitido.

O:

```php
User::query()->with('posts')->get();
```

---

# 69. Lazy Loading Policy

No deberá ser invisible arquitectónicamente.

Podrá configurarse:

```text
ALLOW
WARN
FORBID
```

---

# 70. Serialization

La serialización no deberá disparar lazy queries por defecto.

---

# 71. N+1 Developer Experience

En desarrollo/testing podrá advertirse:

```text
Potential N+1 detected
```

sin convertir automáticamente la query.

---

# 72. Pagination API

Ejemplo:

```php
$page = User::query()
    ->orderBy('id')
    ->paginate(50);
```

---

# 73. Cursor Pagination

Ejemplo:

```php
$page = User::query()
    ->orderBy('created_at')
    ->orderBy('id')
    ->cursorPaginate(50);
```

---

# 74. Cursor ≠ Encoded Offset

La API no deberá prometer esa equivalencia.

---

# 75. Pagination Total

Podrá configurarse:

```text
EXACT
ESTIMATED
NONE
DEFERRED
```

---

# 76. Chunk API

Ejemplo:

```php
User::query()
    ->orderBy('id')
    ->chunk(1000, function ($users) {
        // ...
    });
```

---

# 77. Chunk ≠ Transaction

Cada chunk no será automáticamente una transacción salvo configuración explícita.

---

# 78. Lazy Collection API

Podrá existir:

```php
User::query()->lazy();
```

con lifecycle explícito.

---

# 79. Bulk API

Operaciones masivas deberán ser explícitas.

Ejemplo:

```php
$database->bulk()
    ->insert(User::class, $records);
```

---

# 80. Bulk ≠ Loop save()

```text
Bulk Insert
≠
foreach -> save()
```

---

# 81. Bulk ORM Coherence

Si una operación bulk modifica directamente datos que ya están en IdentityMap, la API deberá documentar y controlar la coherencia.

---

# 82. Bulk Safety

Podrá requerirse:

```php
->where(...)
```

antes de:

```php
->delete()
```

según políticas.

---

# 83. Dangerous Mutation Guard

Ejemplo:

```php
User::query()->delete();
```

sin predicado podrá:

```text
reject
```

por defecto.

---

# 84. Explicit Full-table Mutation

Podría requerir:

```php
->allowAllRows()
```

o equivalente.

---

# 85. Schema API

La API podrá ofrecer:

```php
Schema::create('users', function (Table $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
});
```

---

# 86. Schema Builder ≠ SQL Builder

Produce:

```text
Schema Definition / Schema AST
```

No SQL directamente.

---

# 87. Schema API Typed Definitions

Los shortcuts:

```php
$table->string(...)
```

deberán normalizarse a definiciones canónicas.

---

# 88. Schema Platform Awareness

El developer podrá expresar intención portable.

Ejemplo:

```php
$table->json('settings');
```

y Platform Capability System determinará representabilidad.

---

# 89. Unsupported Schema Feature

La API deberá fallar de manera explicable.

No degradar silenciosamente.

---

# 90. Migration API

La experiencia común podrá ser:

```php
return new class extends Migration
{
    public function up(): void
    {
        Schema::create(...);
    }

    public function down(): void
    {
        Schema::drop(...);
    }
};
```

---

# 91. Migration ≠ Schema Builder

Migration gobierna evolución.

Schema Builder describe transformaciones.

---

# 92. Migration Safety API

Operaciones peligrosas podrán requerir:

```text
explicit acknowledgment
```

o políticas específicas.

---

# 93. Connection API

El desarrollador podrá resolver:

```php
$connection = $database->connection();
```

o:

```php
$connection = $database->connection('analytics');
```

---

# 94. Connection ≠ Driver Handle

La API pública no deberá exponer PDO como abstracción principal.

---

# 95. Native Handle Escape Hatch

Podrá existir para integración avanzada:

```php
$connection->nativeHandle();
```

pero deberá considerarse API avanzada.

---

# 96. Native Handle Risks

Al utilizarlo, el desarrollador puede modificar:

```text
session state
transaction state
driver options
```

por fuera del conocimiento normal de VoltStack.

---

# 97. Native Handle Guard

La API podrá requerir:

```text
explicit unsafe/native access
```

y marcar la conexión como potencialmente dirty cuando sea necesario.

---

# 98. Connection Selection

La API podrá permitir:

```php
$database->connection('reporting');
```

pero routing read/write, tenant y shard deberán permanecer en servicios especializados.

---

# 99. Read/Write API

Podrán existir hints semánticos:

```php
$query->readIntent(ReadIntent::Consistent);
```

en vez de:

```php
$query->useReplica('replica-2');
```

para la mayoría de aplicaciones.

---

# 100. Physical Routing ≠ Application Concern

El desarrollador normalmente expresa intención.

Database decide endpoint.

---

# 101. Writer Pinning

Cuando sea necesario podrá existir una API explícita:

```php
$query->useWriter();
```

pero no deberá ser necesaria para operaciones normales correctamente clasificadas.

---

# 102. Transaction Pinning

Dentro de una transacción:

```text
connection/routing affinity
```

será manejada por Transaction System.

---

# 103. Capability API

Podrá consultarse:

```php
if ($database->capabilities()->supports(
    Capability::QUERY_RETURNING
)) {
    // ...
}
```

---

# 104. Capability API ≠ Vendor Detection

Evitar:

```php
if ($database->platform()->name() === 'postgresql') {
```

cuando la intención sea preguntar por una capacidad.

---

# 105. Capability Explanation

Para diagnostics:

```php
$database
    ->capabilities()
    ->explain(Capability::QUERY_RETURNING);
```

---

# 106. Public Capability Status

Podrá representar:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EXTENSION
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 107. UNKNOWN Visible

La API no convertirá:

```text
UNKNOWN
```

automáticamente en:

```php
false
```

si el caller requiere semántica completa.

---

# 108. supports() Convenience

Una convenience API booleana podrá existir sólo con política claramente definida.

Por ejemplo:

```php
supports()
```

puede significar:

```text
usable under current default policy
```

mientras:

```php
status()
```

expone el estado completo.

---

# 109. Error API

Database deberá utilizar una jerarquía consistente de excepciones.

Ejemplo conceptual:

```text
DatabaseException
├── ConnectionException
├── QueryException
├── TransactionException
├── SchemaException
├── MigrationException
├── OrmException
├── HydrationException
├── CapabilityException
├── SecurityException
└── ResourceException
```

---

# 110. Exception ≠ Raw Driver Exception

Los errores del Driver deberán normalizarse.

---

# 111. Preserve Original Cause

La excepción original podrá mantenerse como:

```text
previous/cause
```

cuando sea seguro.

---

# 112. Safe Error Context

Una excepción podrá exponer:

```text
query fingerprint
connection name
platform
error category
SQLSTATE
operation id
```

sin revelar datos sensibles.

---

# 113. Error Messages

Se desarrollará específicamente en:

```text
308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md
```

---

# 114. SQL in Exceptions

Por defecto deberá evitarse mostrar:

```text
raw sensitive parameters
```

---

# 115. Query Diagnostics

La API podrá ofrecer:

```php
$query->explain();
```

cuando la plataforma lo permita.

---

# 116. explain() ≠ Execute Application Query

Deberá utilizar un flujo controlado.

---

# 117. Debug API

Podrá existir:

```php
$query->debug();
```

que produzca información estructurada.

---

# 118. Debug ≠ Raw Secret Dump

Nunca.

---

# 119. Query Fingerprint

Será preferible mostrar:

```text
fingerprint
normalized query structure
parameter types
```

en lugar de valores sensibles.

---

# 120. API Return Types

VoltStack evitará retornos ambiguos.

Incorrecto:

```php
mixed
```

cuando exista un tipo razonable.

---

# 121. Typed Collections

Podrán existir:

```php
EntityCollection<User>
```

a nivel de análisis estático/documentación.

---

# 122. PHP Generics

Mientras PHP no disponga de generics nativos completos, podrán utilizarse anotaciones compatibles con analizadores estáticos.

Ejemplo:

```php
/**
 * @extends Repository<User>
 */
final class UserRepository extends Repository
{
}
```

---

# 123. IDE Discoverability

La API deberá optimizarse para:

```text
PHPStorm
VS Code
static analyzers
language servers
```

---

# 124. Minimal Magic

VoltStack podrá utilizar cierta magia ergonómica.

Pero:

> toda magia deberá tener una semántica determinista, documentable y analizable.

---

# 125. Magic Methods

Deberán limitarse.

Evitar:

```php
__call()
```

como mecanismo universal de API.

---

# 126. Generated Metadata

Cuando mejore IDE support podrán generarse:

```text
repository metadata
model metadata
relation metadata
query helpers
IDE stubs
```

---

# 127. Code Generation

Se especificará en:

```text
310_DATABASE_CODE_GENERATION_SYSTEM.md
```

---

# 128. Naming Conventions

La API deberá mantener verbos consistentes.

Ejemplos:

```text
find
findOrFail
get
first
firstOrFail
exists
count
persist
remove
flush
clear
transaction
stream
paginate
cursorPaginate
chunk
```

---

# 129. Same Verb, Same Meaning

`find()` no deberá significar una cosa radicalmente distinta entre Repository y Model API.

---

# 130. Method Semantic Stability

Cambiar la semántica de un método será considerado un cambio de compatibilidad aunque su firma PHP permanezca igual.

---

# 131. Boolean Naming

Se favorecerán:

```text
is...
has...
supports...
can...
```

cuando sean realmente booleanos.

---

# 132. Explicit Options Objects

Cuando una operación tenga demasiadas opciones se preferirá:

```php
$query->execute(
    new QueryExecutionOptions(...)
);
```

en vez de:

```php
execute(true, false, null, 3, 1000, true);
```

---

# 133. Named Arguments

La API deberá ser razonablemente segura para named arguments cuando el contrato lo permita.

---

# 134. Named Argument Compatibility

Los nombres de parámetros públicos podrán considerarse parte de la compatibilidad.

---

# 135. Value Objects

Se preferirán para conceptos con semántica propia:

```text
ConnectionName
TransactionIsolation
QueryTimeout
PageSize
CursorToken
CapabilityId
TenantId
ShardId
```

cuando aporten seguridad real.

---

# 136. Value Objects ≠ Ceremony Everywhere

No será necesario envolver cada integer/string.

---

# 137. Defaults

Los defaults deberán ser:

```text
safe
portable
predictable
observable
```

---

# 138. Safe Defaults

Ejemplos:

```text
parameterized queries
no destructive schema mutation without checks
no global mutable EntityManager
no automatic cross-tenant access
no hidden retry after unknown commit
no unbounded streaming buffers
no unsafe raw SQL
```

---

# 139. Zero Configuration

VoltStack buscará:

```text
low configuration
```

no:

```text
absence of explicit semantics
```

---

# 140. Convention over Configuration

Las convenciones podrán resolver:

```text
default connection
repository
table naming
identifier naming
```

pero siempre deberán poder inspeccionarse.

---

# 141. Convention ≠ Hidden Irreversible Decision

La configuración explícita deberá poder sobrescribir convenciones dentro de límites válidos.

---

# 142. Default Connection

Podrá configurarse:

```php
'database.default' => 'main'
```

---

# 143. Missing Default Connection

Deberá producir error temprano y explicable.

---

# 144. Multiple Connections

La API deberá soportarlas sin introducir global state.

---

# 145. Connection Context

Podrá existir:

```php
$database->using('analytics', function (Database $db) {
    // ...
});
```

si se considera útil.

---

# 146. using() ≠ Global Mutation

No deberá modificar:

```text
global default connection
```

para todo el worker.

---

# 147. Scoped Override

Cualquier override contextual deberá ser:

```text
scope-bound
```

---

# 148. Tenant Context

La API base no deberá exigir Multitenancy.

Cuando el paquete esté instalado:

```text
Tenant Context
```

podrá integrarse en resolution.

---

# 149. Multitenancy Optionality

```text
Database Core
```

no dependerá de:

```text
Quantum/Multitenancy
```

---

# 150. SaaS Optionality

Igualmente:

```text
Quantum/SaaS
```

no será dependencia de Database.

---

# 151. Query API and Tenant Scopes

Cuando Multitenancy esté instalado, podrá añadir:

```text
tenant predicates
tenant routing
tenant connection resolution
```

mediante extension points oficiales.

---

# 152. No Hidden Tenant Bypass

Una API como:

```php
withoutTenantScope()
```

deberá ser explícita, autorizada y auditable.

---

# 153. Security-sensitive APIs

Se podrán clasificar:

```text
SAFE
ADVANCED
PRIVILEGED
UNSAFE
```

---

# 154. Privileged API

Ejemplos:

```text
cross-tenant query
administrative connection
destructive migration
native driver handle
unsafe raw SQL
```

---

# 155. Unsafe ≠ Impossible

VoltStack deberá permitir casos avanzados.

Pero hará visible la reducción de garantías.

---

# 156. API Capability Negotiation

Si una operación no puede representarse:

```php
$query->returning('id');
```

en la plataforma actual, la API deberá producir una respuesta explícita.

---

# 157. No Silent Semantic Degradation

Nunca:

```text
RETURNING unsupported
→ silently ignore returning()
```

---

# 158. Portable API

La API deberá expresar semántica portable siempre que sea posible.

---

# 159. Platform-specific API

También podrán existir extensiones específicas:

```php
$query->platform(PostgreSql::class)
```

o namespaces especializados.

Pero deberán ser claramente no portables.

---

# 160. Portable ≠ Lowest Common Denominator

VoltStack no limitará toda su API a las capacidades mínimas de SQLite.

---

# 161. Capability-aware Advanced Features

Las features avanzadas podrán existir mediante:

```text
semantic API
+
capability requirements
+
platform compiler
```

---

# 162. Query Example

```php
$users = User::query()
    ->where('active', true)
    ->with('profile')
    ->orderBy('created_at', 'desc')
    ->limit(100)
    ->get();
```

Conceptualmente:

```text
Model API
   ↓
Entity Query
   ↓
Query AST
   ↓
Semantic Analysis
   ↓
Optimization
   ↓
Planning
   ↓
Capability Validation
   ↓
Compilation
   ↓
Execution
   ↓
Hydration
   ↓
IdentityMap
   ↓
EntityCollection<User>
```

---

# 163. Public Simplicity

El developer ve:

```php
->get();
```

No necesita invocar manualmente:

```text
SemanticAnalyzer
Optimizer
Planner
Compiler
Executor
Hydrator
```

---

# 164. Hidden Pipeline ≠ Hidden Semantics

Aunque el pipeline sea automático, su comportamiento deberá ser:

```text
inspectable
debuggable
explainable
```

---

# 165. Query Inspection

Podrá ofrecerse:

```php
$query->inspect();
```

para obtener representación segura del pipeline.

---

# 166. Query Compilation Inspection

En desarrollo:

```php
$query->compile();
```

podría devolver una representación compilada sin ejecutar.

---

# 167. compile() ≠ execute()

Deberá mantenerse estrictamente.

---

# 168. Execution API

Para Query Objects avanzados:

```php
$result = $database
    ->executor()
    ->execute($query);
```

podrá existir como API especializada.

---

# 169. Executor Exposure

No deberá convertirse en la ruta recomendada para código de aplicación normal.

---

# 170. Raw Query API

VoltStack podrá soportar:

```php
$database->query()->raw(...)
```

bajo políticas estrictas.

---

# 171. Raw Query Classification

Deberá distinguir:

```text
trusted SQL
bound parameters
unsafe identifiers
```

---

# 172. Parameter Binding

Ejemplo:

```php
$db->execute(
    'SELECT * FROM users WHERE email = :email',
    ['email' => $email],
);
```

podrá ser aceptable como escape hatch.

---

# 173. Raw SQL ≠ String Interpolation

Nunca promover:

```php
$db->execute(
    "SELECT * FROM users WHERE email = '$email'"
);
```

---

# 174. Native Result

Las raw queries podrán devolver:

```text
Result
```

no directamente un driver-specific result handle.

---

# 175. Public Result Contract

Conceptualmente:

```php
interface Result
{
    public function rows(): iterable;

    public function first(): mixed;

    public function affectedRows(): int;

    public function close(): void;
}
```

según tipo de operación.

---

# 176. Result ≠ Array

Aunque pueda convertirse a array.

---

# 177. Result Lifecycle

Deberá ser explícito cuando posea recursos.

---

# 178. Import API

Podrá existir:

```php
$database->import(...)
```

o servicio dedicado:

```php
Importer
```

---

# 179. Export API

Igualmente:

```php
Exporter
```

---

# 180. Large Dataset APIs

No deberán materializar todo en memoria por defecto.

---

# 181. Memory-aware API

Ejemplos:

```text
stream
cursor
chunk
lazy
bulk
```

deberán ser first-class.

---

# 182. Cancellation

Cuando esté soportado:

```php
$query->withCancellation($token);
```

o mediante execution options.

---

# 183. CancellationToken

Podrá ser compartido con otras capas VoltStack.

---

# 184. Timeout API

Ejemplo:

```php
$query->timeout(Duration::seconds(5));
```

---

# 185. Timeout Types

La API podrá distinguir:

```text
acquisition timeout
statement timeout
transaction timeout
overall operation deadline
```

---

# 186. Generic timeout() Semantics

Si existe:

```php
timeout()
```

deberá documentar exactamente cuál timeout controla.

---

# 187. Retry API

Retry no será un simple:

```php
->retry(3)
```

para cualquier query.

---

# 188. Retry Safety

La API deberá considerar:

```text
operation replayability
transaction boundary
idempotency
failure classification
outcome certainty
```

---

# 189. Read Retry

Puede ser más permisivo.

---

# 190. Write Retry

Requerirá reglas más estrictas.

---

# 191. UNKNOWN Outcome

Nunca deberá esconderse.

---

# 192. Cache API

La API pública podrá permitir:

```php
$query->cacheFor(Duration::minutes(5));
```

cuando corresponda.

---

# 193. Cache ≠ Correctness

La API deberá distinguir:

```text
cache policy
```

de:

```text
transactional consistency
```

---

# 194. Query Cache vs Result Cache

El developer no deberá necesitar conocer todas las capas internas para casos simples.

Pero APIs avanzadas podrán distinguirlas.

---

# 195. Cache Tags/Dependencies

Cuando existan deberán representar dependencias semánticas.

---

# 196. ORM Entity Cache

No devolverá objetos managed compartidos entre requests.

---

# 197. IdentityMap

No será expuesta como cache pública.

---

# 198. Event API

Database podrá emitir eventos.

Pero la API pública deberá distinguir:

```text
database lifecycle events
```

de:

```text
domain events
```

---

# 199. Event Listener ≠ Query Middleware Universal

No deberá permitirse que cualquier listener reescriba arbitrariamente semantics internas.

---

# 200. Extension API

Los paquetes podrán extender Database mediante:

```text
Extension System
Plugin System
Custom Driver
Custom Dialect
Custom Compiler
Query Extension
ORM Extension
Capability Provider
```

definidos en documentos 294–301.

---

# 201. Public Extension Contracts

Los extension points oficiales sí serán parte de Public API.

---

# 202. Internal Extension Hooks

Hooks experimentales podrán permanecer:

```text
@internal
```

---

# 203. Extension Stability Levels

Podrán clasificarse:

```text
STABLE
SUPPORTED
EXPERIMENTAL
INTERNAL
```

---

# 204. Experimental API

Deberá estar claramente marcada.

---

# 205. Internal API

No tendrá garantía normal de backward compatibility.

---

# 206. Public API Metadata

VoltStack podrá marcar mediante atributos/anotaciones:

```php
#[PublicApi]
```

```php
#[Experimental]
```

```php
#[Internal]
```

---

# 207. PHP Visibility ≠ API Stability

Una clase:

```php
public final class ...
```

puede seguir siendo:

```text
@internal
```

porque `public` en PHP significa accesibilidad, no promesa de compatibilidad.

---

# 208. Namespace Policy

Namespaces públicos deberán ser relativamente estables.

Ejemplo:

```text
VoltStack\Quantum\Database
VoltStack\Quantum\Database\ORM
VoltStack\Quantum\Database\Query
VoltStack\Quantum\Database\Schema
VoltStack\Quantum\Database\Transaction
```

---

# 209. Internal Namespace

Podrá utilizarse:

```text
VoltStack\Quantum\Database\Internal
```

para componentes explícitamente no públicos cuando sea útil.

---

# 210. Contract Namespace

Interfaces públicas importantes podrán vivir en:

```text
Contract/
```

---

# 211. Contract ≠ Automatically Public

La clasificación seguirá siendo explícita.

---

# 212. API Dependency Direction

Aplicación:

```text
Application
    ↓
Public Contracts
    ↓
Database Implementation
```

No:

```text
Application
    ↓
Internal Compiler Nodes
```

para operaciones normales.

---

# 213. Framework Integration

VoltStack podrá registrar automáticamente:

```text
Database
ConnectionManager
TransactionManager
EntityManager
RepositoryResolver
SchemaManager
```

en Container.

---

# 214. Container Scope

Servicios con mutable request state deberán tener scope correcto.

---

# 215. Singleton Candidates

Podrán ser singleton:

```text
immutable registries
metadata definitions
compiler definitions
platform definitions
type registry after freeze
capability definitions
```

---

# 216. Non-singleton Mutable State

No deberán compartirse globalmente:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
DatabaseContext
TenantContext
Query execution state
```

---

# 217. Public API and FrankenPHP

La API deberá funcionar igual desde la perspectiva del desarrollador:

```text
PHP-FPM
FrankenPHP
RoadRunner
OpenSwoole
```

dentro de las capabilities soportadas.

---

# 218. Runtime ≠ API Rewrite

El código:

```php
User::find(1);
```

no deberá cambiar simplemente porque la aplicación corre en FrankenPHP.

---

# 219. Runtime-specific APIs

Sólo existirán para casos realmente runtime-specific.

---

# 220. Coroutine Safety

En OpenSwoole futuro:

```text
static convenience syntax
```

deberá resolver contexto de coroutine/operation apropiado.

---

# 221. Async Future

La API deberá evitar decisiones que hagan imposible añadir en el futuro:

```text
async query execution
async streaming
concurrent queries
```

---

# 222. Sync API First

La V1 podrá ser primordialmente síncrona.

---

# 223. Future Async API

No deberá introducirse simplemente cambiando todos los retornos actuales a Promises.

Deberá diseñarse como capability separada.

---

# 224. Public API Compatibility

La estabilidad será gobernada posteriormente por:

```text
322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md
323_DATABASE_VERSIONING_SYSTEM.md
324_DATABASE_DEPRECATION_POLICY.md
```

---

# 225. Breaking Changes

Podrán incluir:

```text
removing public class
renaming public method
changing parameter type
changing return type
changing exception contract
changing method semantics
changing default behavior materially
changing named parameter
changing lifecycle behavior
```

---

# 226. Behavioral Compatibility

Importante:

```text
signature compatibility
≠
behavioral compatibility
```

---

# 227. Deprecation

Las APIs no deberán desaparecer inmediatamente.

---

# 228. Deprecation Metadata

Podrá utilizarse:

```php
#[Deprecated(
    since: '2.0',
    replacement: '...'
)]
```

cuando exista soporte adecuado.

---

# 229. Deprecation Diagnostics

En desarrollo podrán emitirse advertencias controladas.

---

# 230. No Production Spam

Las deprecations no deberán inundar logs de producción por defecto.

---

# 231. Public API Documentation

Cada API estable deberá documentar:

```text
purpose
parameters
return
exceptions
side effects
transaction semantics
resource ownership
runtime scope
portability
capability requirements
```

cuando aplique.

---

# 232. Side-effect Documentation

Especialmente para:

```text
save
delete
flush
commit
bulk
schema
migration
stream
native handle
```

---

# 233. Public API Error Contract

Una API deberá documentar categorías de error esperables.

No necesariamente cada clase interna.

---

# 234. Error Taxonomy

El caller deberá poder distinguir programáticamente:

```text
connection failure
constraint violation
deadlock
serialization conflict
timeout
cancellation
unknown transaction outcome
unsupported capability
```

---

# 235. Error Code

Podrá existir:

```text
DatabaseErrorCode
```

estable.

---

# 236. Vendor Error Code

Podrá conservarse como metadata secundaria.

---

# 237. Vendor Code ≠ Public Control Flow

El código de aplicación no debería necesitar:

```php
if ($e->vendorCode() === 1213)
```

para detectar deadlock.

---

# 238. Canonical Error Classification

Preferible:

```php
if ($e instanceof DeadlockException) {
```

o:

```php
if ($e->classification()->isDeadlock()) {
```

---

# 239. Nullability

La API deberá distinguir:

```php
find(): ?Entity
```

de:

```php
findOrFail(): Entity
```

---

# 240. Empty Result

No deberá confundirse con error.

---

# 241. Optional Relationships

Podrán retornar:

```text
null
```

cuando semánticamente corresponde.

---

# 242. Partial Entity Safety

La API pública deberá favorecer:

```text
DTO
Projection
Tuple
Scalar
```

sobre partial entities inseguros.

---

# 243. Explicit Partial Entity

Si se permite deberá ser explícito.

---

# 244. Loaded Fields

No deberá asumirse que:

```text
missing field
=
NULL
```

---

# 245. Database NULL ≠ Field Not Loaded

Regla pública importante.

---

# 246. Collections

Una relación collection podrá tener estados:

```text
UNLOADED
PARTIAL
LOADED
```

conceptualmente.

---

# 247. Empty ≠ Unloaded

No deberán confundirse.

---

# 248. Relation API

La API deberá evitar que:

```php
count($entity->posts)
```

cambie silenciosamente de significado entre:

```text
partial
```

y:

```text
fully loaded
```

sin mecanismos apropiados.

---

# 249. Serialization API

La serialización deberá respetar loaded state.

---

# 250. API Security

Las APIs públicas deberán ser seguras por defecto frente a:

```text
SQL injection
identifier injection
cross-tenant leakage
secret leakage
unsafe destructive operations
unbounded resource use
```

---

# 251. Parameterization Invariant

Valores externos nunca deberán requerir escape manual para queries normales.

---

# 252. No `escape()` Workflow

La API no promoverá:

```php
$value = $db->escape($input);
```

como mecanismo normal de seguridad.

---

# 253. Identifier API

Identifiers dinámicos deberán validarse contra:

```text
allowlist
schema metadata
typed identifier
```

según caso.

---

# 254. Order By Input

Ejemplo inseguro:

```php
->orderBy($_GET['sort']);
```

La API/documentación deberá promover:

```php
$sort = match ($request->sort) {
    'name' => UserSort::Name,
    'created' => UserSort::CreatedAt,
    default => UserSort::Name,
};
```

---

# 255. Public API Resource Governance

Operaciones potencialmente grandes podrán aceptar:

```text
limits
timeouts
budgets
batch sizes
```

---

# 256. Unbounded get()

La API podrá permitirlo, pero tooling/diagnostics podrán advertir cuando el resultado esperado sea masivo.

---

# 257. Safe Alternatives

Se promoverán:

```text
paginate()
cursorPaginate()
stream()
chunk()
lazy()
```

---

# 258. API Observability

Las operaciones públicas deberán poder correlacionarse con:

```text
query telemetry
transaction telemetry
ORM telemetry
request telemetry
trace context
```

sin requerir instrumentación manual.

---

# 259. Telemetry ≠ API Behavior

La instrumentación no deberá cambiar resultados semánticos.

---

# 260. Public API Performance

La ergonomía no deberá introducir capas de reflexión innecesarias en hot paths.

---

# 261. Convenience Overhead

Facades/Model static API podrán tener overhead pequeño de context resolution.

Pero deberán converger rápidamente al mismo engine.

---

# 262. Repository vs Model API

No se prometerá que una sea inherentemente más rápida.

Ambas utilizan la misma infraestructura.

---

# 263. Metadata Compilation

Permitirá que APIs ergonómicas sigan siendo eficientes.

---

# 264. Public API Testing

La API deberá tener una suite dedicada.

---

# 265. API Contract Tests

Cubrirán:

```text
method semantics
return types
exceptions
defaults
scoping
resource ownership
transaction behavior
capability behavior
security behavior
```

---

# 266. API Parity Tests

Model API y Repository API deberán producir comportamiento ORM coherente.

---

# 267. Example

```php
$a = User::find(10);

$b = $userRepository->find(10);
```

dentro del mismo EntityManager scope deberán respetar:

```php
$a === $b
```

cuando ambos representan la misma identidad administrada.

---

# 268. IdentityMap Across APIs

Esta propiedad es crítica.

```text
Dual Public API
≠
Dual IdentityMap
```

---

# 269. Transaction API Tests

Deberán demostrar:

```text
flush ≠ commit
rollback ≠ object graph rewind
unknown commit outcome preserved
nested policy respected
```

---

# 270. Query API Tests

Deberán demostrar:

```text
parameterization
identifier safety
AST generation
semantic equivalence
result shape
capability validation
```

---

# 271. Runtime Tests

Especialmente:

```text
FrankenPHP repeated requests
```

para comprobar ausencia de state leakage.

---

# 272. Static Analysis Tests

La API podrá verificarse mediante:

```text
PHPStan
Psalm
IDE stubs
reflection-based signature tests
```

según herramientas adoptadas.

---

# 273. API Snapshot Tests

Podrá generarse una representación machine-readable de la Public API.

Ejemplo:

```text
public-api.json
```

---

# 274. API Diff

Entre releases:

```text
Version N API Snapshot
          ↓
        Diff
          ↓
Version N+1 API Snapshot
```

---

# 275. API Diff Detection

Podrá detectar:

```text
removed classes
removed methods
changed parameters
changed return types
changed inheritance
changed interfaces
```

---

# 276. Behavioral Changes

No todas podrán detectarse automáticamente.

Requerirán:

```text
contract tests
review
release notes
```

---

# 277. Public API Manifest

Conceptualmente:

```json
{
  "version": 1,
  "namespaces": [
    "VoltStack\\Quantum\\Database"
  ],
  "stability": "stable"
}
```

La forma final podrá evolucionar.

---

# 278. API Stability Attributes

Ejemplo conceptual:

```php
#[PublicApi]
#[Stable]
interface TransactionManager
{
}
```

---

# 279. Public API Linter

Futuro tooling podrá detectar:

```text
Public API referencing Internal type
```

---

# 280. Critical Rule

Una firma pública no deberá exponer accidentalmente:

```text
Internal\CompilerNode
```

porque eso convierte el tipo interno en dependencia pública.

---

# 281. Public Type Closure

Formalmente, si:

```text
P
```

es una API pública, sus tipos visibles:

```text
parameters
returns
properties
parent types
generic bounds
exceptions contract
```

deberán pertenecer a la superficie pública o a tipos PHP compatibles.

---

# 282. Internal Leakage

Incorrecto:

```php
interface Query
{
    public function internalPlan(): InternalPhysicalPlan;
}
```

---

# 283. Diagnostic Representation

Preferible:

```php
interface Query
{
    public function inspect(): QueryInspection;
}
```

donde:

```text
QueryInspection
```

sea un DTO público estable.

---

# 284. API DTOs

DTOs públicos deberán diseñarse para estabilidad.

---

# 285. Public DTO ≠ Internal State Object

No deberá exponerse directamente mutable internal state.

---

# 286. Immutability

Se favorecerán:

```php
readonly
```

value objects y diagnostic DTOs.

---

# 287. Public Collections

No deberán exponer referencias mutables a registries internos.

---

# 288. Configuration API

La configuración podrá ser declarativa:

```php
'database' => [
    'default' => 'main',

    'connections' => [
        'main' => [
            'driver' => 'pgsql',
            // ...
        ],
    ],
],
```

---

# 289. Configuration ≠ Runtime API

La configuración define bootstrap.

No sustituye APIs runtime.

---

# 290. Typed Configuration

Internamente deberá convertirse a:

```text
DatabaseConfiguration
ConnectionConfiguration
PoolConfiguration
```

tipados.

---

# 291. Public Config Validation

Errores deberán detectarse temprano.

---

# 292. Secret Configuration

Passwords/tokens deberán soportar secret providers.

---

# 293. Environment Variables

Podrán alimentar config.

No deberán convertirse directamente en estado runtime sin validación.

---

# 294. CLI as Public Surface

Comandos CLI también son parte de developer experience.

Se especificarán en:

```text
309_DATABASE_CLI_SYSTEM.md
```

---

# 295. CLI Compatibility

Nombres/options de comandos estables podrán considerarse parte de Public API.

---

# 296. Code Generation as Public Surface

Generators también deberán seguir contracts estables.

---

# 297. Database Public API Architecture

La arquitectura final será:

```text
                       Application
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
     Model API        Repository API        DB API
        │                   │                   │
        └──────────────┬────┴──────────────┬────┘
                       │                   │
                       ▼                   ▼
                 Public Contracts     Facades/Helpers
                       │                   │
                       └─────────┬─────────┘
                                 ▼
                         Scoped Resolution
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
         EntityManager      Query Engine      Transactions
              │                  │                  │
              └──────────────────┼──────────────────┘
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

Paralelamente:

```text
Schema API
    ↓
Schema Model / AST
    ↓
Schema Planner
    ↓
Compiler
```

y:

```text
Migration API
    ↓
Migration Engine
    ↓
Safety
    ↓
Execution
```

---

# 298. API Classification Model

Toda superficie podrá clasificarse:

```text
PUBLIC_STABLE
PUBLIC_SUPPORTED
PUBLIC_EXPERIMENTAL
INTERNAL
```

---

# 299. PUBLIC_STABLE

Máxima expectativa de compatibilidad.

---

# 300. PUBLIC_SUPPORTED

Soportada, pero puede evolucionar bajo política de versionado.

---

# 301. PUBLIC_EXPERIMENTAL

Disponible para evaluación.

Puede cambiar con menor garantía.

---

# 302. INTERNAL

No forma parte del contrato público.

---

# 303. Public API Registry

VoltStack podrá mantener metadata de estas clasificaciones.

---

# 304. Reflection-based Verification

Durante CI podrá comprobarse:

```text
Public API
    ↓
No forbidden internal dependencies
```

---

# 305. Developer Experience Principle

La API deberá optimizar la operación frecuente:

```php
$user = User::find($id);
```

sin sacrificar la capacidad avanzada:

```php
$query = $database
    ->query()
    ->select(...)
    ->where(...)
    ->withExecutionOptions(...);
```

---

# 306. Progressive Disclosure

La complejidad deberá revelarse progresivamente:

```text
Basic
  ↓
Intermediate
  ↓
Advanced
  ↓
Infrastructure
```

---

# 307. Example — Basic

```php
$user = User::findOrFail($id);

$user->name = 'Alice';

$user->save();
```

---

# 308. Example — Service-oriented

```php
final class RenameUser
{
    public function __construct(
        private UserRepository $users,
        private TransactionManager $transactions,
    ) {}

    public function execute(int $id, string $name): void
    {
        $this->transactions->run(function () use ($id, $name) {
            $user = $this->users->findOrFail($id);

            $user->rename($name);

            $this->users->flush();
        });
    }
}
```

La forma exacta de `flush()` podrá permanecer en EntityManager si se decide evitarlo en Repository.

Lo importante es que:

```text
Repository API
```

no cree un persistence engine alternativo.

---

# 309. Example — Query

```php
$users = $database
    ->query()
    ->from('users')
    ->select('id', 'name')
    ->where('active', true)
    ->orderBy('created_at', 'desc')
    ->limit(100)
    ->get();
```

---

# 310. Example — Transaction

```php
$result = $transactions->run(function () use ($orders, $payments) {
    $order = $orders->create(...);

    $payments->charge(...);

    return $order;
});
```

Los side effects externos requerirán mecanismos como:

```text
Outbox
```

cuando se necesite atomicidad lógica.

---

# 311. Example — Streaming

```php
$stream = User::query()
    ->orderBy('id')
    ->stream();

try {
    foreach ($stream as $user) {
        // process
    }
} finally {
    $stream->close();
}
```

---

# 312. Example — Capability

```php
$status = $database
    ->capabilities()
    ->status(Capability::QUERY_RETURNING);

if ($status->isUsable()) {
    // ...
}
```

---

# 313. Example — Schema

```php
Schema::create('users', function (Table $table): void {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamps();
});
```

---

# 314. Example — Explicit unsafe operation

```php
$database
    ->unsafe()
    ->nativeConnection();
```

Una API similar podrá utilizarse para agrupar operaciones que reducen garantías.

---

# 315. Unsafe Namespace/API

Una posible arquitectura:

```text
Database
├── normal API
└── unsafe()
      ├── raw()
      ├── nativeHandle()
      └── privileged operations
```

Esto evita mezclar operaciones peligrosas con métodos cotidianos.

---

# 316. Unsafe ≠ Unprotected

Incluso APIs unsafe deberán conservar:

```text
authorization
audit
scope isolation
resource cleanup
```

cuando aplique.

---

# 317. Public API Invariants

## DB-PUBLIC-001

Internal Complexity ≠ Public API Complexity.

## DB-PUBLIC-002

Simple API ≠ Hidden Semantic Ambiguity.

## DB-PUBLIC-003

Public API ≠ Every Public PHP Class.

## DB-PUBLIC-004

PHP visibility ≠ API stability.

## DB-PUBLIC-005

Convenience API ≠ Alternative Database Engine.

## DB-PUBLIC-006

Structured API ≠ Different Semantics.

## DB-PUBLIC-007

Advanced API ≠ Permission to Break Core Invariants.

## DB-PUBLIC-008

Database root object ≠ God Object.

## DB-PUBLIC-009

Dependency Injection será la integración arquitectónica primaria.

## DB-PUBLIC-010

Facades serán opcionales.

## DB-PUBLIC-011

Helpers no introducirán global mutable state.

## DB-PUBLIC-012

Model API ≠ Persistence Engine.

## DB-PUBLIC-013

Static Model API ≠ Static ORM State.

## DB-PUBLIC-014

Repository ≠ Query Builder.

## DB-PUBLIC-015

Repository ≠ EntityManager.

## DB-PUBLIC-016

persist() ≠ INSERT.

## DB-PUBLIC-017

remove() ≠ DELETE inmediato.

## DB-PUBLIC-018

flush() ≠ commit().

## DB-PUBLIC-019

Rollback ≠ Object Graph Rewind.

## DB-PUBLIC-020

Query Builder ≠ SQL String Builder.

## DB-PUBLIC-021

Simple query syntax seguirá produciendo AST tipado.

## DB-PUBLIC-022

Query values serán parameterized por defecto.

## DB-PUBLIC-023

Raw SQL será escape hatch explícito.

## DB-PUBLIC-024

Cursor ≠ Collection.

## DB-PUBLIC-025

Streaming tendrá resource lifecycle explícito.

## DB-PUBLIC-026

Entity Query ≠ Table Query.

## DB-PUBLIC-027

Entity Field ≠ Database Column.

## DB-PUBLIC-028

Lazy Loading será policy-controlled.

## DB-PUBLIC-029

Serialization no disparará lazy loading por defecto.

## DB-PUBLIC-030

Cursor Pagination ≠ Encoded Offset.

## DB-PUBLIC-031

Chunk ≠ Transaction.

## DB-PUBLIC-032

Bulk Operation ≠ Loop of save().

## DB-PUBLIC-033

Schema Builder ≠ SQL Builder.

## DB-PUBLIC-034

Migration ≠ Schema Builder.

## DB-PUBLIC-035

Connection ≠ Driver Handle.

## DB-PUBLIC-036

Physical Routing ≠ Normal Application Concern.

## DB-PUBLIC-037

Capability API ≠ Vendor Detection.

## DB-PUBLIC-038

UNKNOWN no será silenciosamente false en APIs semánticas completas.

## DB-PUBLIC-039

Exception ≠ Raw Driver Exception.

## DB-PUBLIC-040

Sensitive parameters no aparecerán en errores por defecto.

## DB-PUBLIC-041

Result ≠ Array.

## DB-PUBLIC-042

Public result resource ownership será explícito.

## DB-PUBLIC-043

Unsafe APIs serán explícitamente identificables.

## DB-PUBLIC-044

Portable API ≠ Lowest Common Denominator.

## DB-PUBLIC-045

Unsupported feature no será silenciosamente ignorada.

## DB-PUBLIC-046

Same verb deberá preservar same semantic intent.

## DB-PUBLIC-047

Method semantics forman parte del compatibility contract.

## DB-PUBLIC-048

Value objects se utilizarán cuando aporten semántica real.

## DB-PUBLIC-049

Defaults serán seguros.

## DB-PUBLIC-050

Zero Configuration ≠ Zero Explicit Semantics.

---

# 318. Runtime Invariants

## DB-PUBLIC-051

No habrá global mutable EntityManager.

## DB-PUBLIC-052

No habrá global mutable UnitOfWork.

## DB-PUBLIC-053

No habrá global mutable IdentityMap.

## DB-PUBLIC-054

No habrá global mutable TransactionContext.

## DB-PUBLIC-055

No habrá global mutable TenantContext.

## DB-PUBLIC-056

Static convenience syntax resolverá state scoped.

## DB-PUBLIC-057

FrankenPHP request state será aislado.

## DB-PUBLIC-058

RoadRunner worker state será aislado.

## DB-PUBLIC-059

OpenSwoole coroutine state será aislado.

## DB-PUBLIC-060

Runtime adapter ≠ Public API rewrite.

---

# 319. ORM Invariants

## DB-PUBLIC-061

Model API y Repository API compartirán EntityManager.

## DB-PUBLIC-062

Model API y Repository API compartirán UnitOfWork.

## DB-PUBLIC-063

Model API y Repository API compartirán IdentityMap.

## DB-PUBLIC-064

Same identity + same context → same managed object.

## DB-PUBLIC-065

Partial entity ≠ Complete entity.

## DB-PUBLIC-066

Database NULL ≠ Field Not Loaded.

## DB-PUBLIC-067

Empty Relationship ≠ Unloaded Relationship.

## DB-PUBLIC-068

IdentityMap ≠ Public Cache.

## DB-PUBLIC-069

Entity Cache ≠ IdentityMap.

## DB-PUBLIC-070

Hydration ≠ Persistence.

---

# 320. Transaction Invariants

## DB-PUBLIC-071

Statement Success ≠ Transaction Commit.

## DB-PUBLIC-072

Commit Exception ≠ Confirmed Rollback.

## DB-PUBLIC-073

UNKNOWN transaction outcome será preservado.

## DB-PUBLIC-074

Retry no ocurrirá sobre UNKNOWN commit automáticamente.

## DB-PUBLIC-075

Nested transaction API respetará política configurada.

## DB-PUBLIC-076

Savepoint ≠ Independent Transaction.

## DB-PUBLIC-077

Requested Isolation ≠ Effective Isolation.

## DB-PUBLIC-078

External side effect ≠ Database transaction.

## DB-PUBLIC-079

afterCommit failure no revertirá commit.

## DB-PUBLIC-080

Transaction context será scoped.

---

# 321. Query Invariants

## DB-PUBLIC-081

Query API generará Query Model/AST.

## DB-PUBLIC-082

Compiler será el único responsable normal de SQL generation.

## DB-PUBLIC-083

Compiler ≠ Executor.

## DB-PUBLIC-084

Executor ≠ ORM.

## DB-PUBLIC-085

Query Builder no conocerá PDO.

## DB-PUBLIC-086

Compiler no ejecutará queries.

## DB-PUBLIC-087

Values ≠ Identifiers.

## DB-PUBLIC-088

Identifiers dinámicos serán validados.

## DB-PUBLIC-089

Raw Expression será explícita.

## DB-PUBLIC-090

Query inspection ≠ Query execution.

---

# 322. Security Invariants

## DB-PUBLIC-091

Parameterization será default.

## DB-PUBLIC-092

No se requerirá manual escaping de values.

## DB-PUBLIC-093

Cross-tenant bypass será explícito y autorizado.

## DB-PUBLIC-094

Native handle access será avanzado/unsafe.

## DB-PUBLIC-095

Secrets serán redacted.

## DB-PUBLIC-096

Destructive operations tendrán safety policy.

## DB-PUBLIC-097

Full-table destructive mutations podrán requerir confirmación semántica explícita.

## DB-PUBLIC-098

Security no será desactivada silenciosamente por convenience APIs.

## DB-PUBLIC-099

Public diagnostics no expondrán credentials.

## DB-PUBLIC-100

Unsafe ≠ Unaudited.

---

# 323. API Stability Invariants

## DB-PUBLIC-101

Public API tendrá clasificación explícita.

## DB-PUBLIC-102

Internal API no tendrá garantía normal de compatibilidad.

## DB-PUBLIC-103

Experimental API será identificable.

## DB-PUBLIC-104

Public API no expondrá accidentalmente tipos internos.

## DB-PUBLIC-105

Public DTO ≠ Internal Mutable State.

## DB-PUBLIC-106

Parameter names públicos podrán formar parte del compatibility contract.

## DB-PUBLIC-107

Return type changes podrán ser breaking changes.

## DB-PUBLIC-108

Exception semantic changes podrán ser breaking changes.

## DB-PUBLIC-109

Default semantic changes podrán ser breaking changes.

## DB-PUBLIC-110

Behavioral Compatibility ≠ Signature Compatibility.

---

# 324. Developer Experience Invariants

## DB-PUBLIC-111

Common operations tendrán rutas cortas.

## DB-PUBLIC-112

Advanced operations seguirán siendo posibles.

## DB-PUBLIC-113

Dangerous operations serán explícitas.

## DB-PUBLIC-114

IDE discoverability será objetivo de arquitectura.

## DB-PUBLIC-115

Magic será limitada y determinista.

## DB-PUBLIC-116

APIs evitarán `mixed` cuando exista tipo útil.

## DB-PUBLIC-117

APIs con múltiples opciones preferirán Options Objects.

## DB-PUBLIC-118

Error messages serán accionables.

## DB-PUBLIC-119

Developer-facing semantics serán consistentes entre APIs.

## DB-PUBLIC-120

Progressive disclosure será principio de diseño.

---

# 325. Resource Invariants

## DB-PUBLIC-121

Streaming resource ownership será conocido.

## DB-PUBLIC-122

Cursor resource ownership será conocido.

## DB-PUBLIC-123

Connection resources tendrán lifecycle explícito.

## DB-PUBLIC-124

Timeout semantics serán explícitas.

## DB-PUBLIC-125

Cancellation semantics serán explícitas.

## DB-PUBLIC-126

Unbounded operations tendrán alternativas bounded.

## DB-PUBLIC-127

Bulk operations tendrán resource governance.

## DB-PUBLIC-128

Persistent workers no retendrán resultados accidentalmente.

## DB-PUBLIC-129

Failed cleanup será observable.

## DB-PUBLIC-130

Resource exhaustion no será tratado como resultado vacío.

---

# 326. Anti-patrones

## 326.1 Exponer todos los internals

```text
Everything public
```

no es una Public API.

---

## 326.2 God Database Object

```php
$db->everything();
```

con cientos de responsabilidades.

---

## 326.3 Static global EntityManager

Prohibido.

---

## 326.4 Model construyendo SQL

Incorrecto.

---

## 326.5 Repository construyendo PDO statements

Incorrecto.

---

## 326.6 Query Builder concatenando valores

Incorrecto.

---

## 326.7 `flush()` haciendo commit implícito

Incorrecto.

---

## 326.8 `save()` implementando un segundo persistence engine

Incorrecto.

---

## 326.9 `find()` devolviendo distintas instancias para misma identidad en mismo scope

Incorrecto.

---

## 326.10 `supports()` implementado con vendor switch disperso

Incorrecto.

---

## 326.11 `if postgres`

cuando la intención es preguntar por una capability.

---

## 326.12 Silent fallback

```text
unsupported feature
→ ignore option
```

Incorrecto.

---

## 326.13 Raw SQL como API principal

Incorrecto.

---

## 326.14 Escaping manual como defensa primaria

Incorrecto.

---

## 326.15 Native PDO como Connection API

Incorrecto.

---

## 326.16 Facade obligatoria

Incorrecto.

---

## 326.17 Helpers con global mutable state

Incorrecto.

---

## 326.18 `mixed` everywhere

Incorrecto.

---

## 326.19 Boolean parameters múltiples

```php
run(true, false, true, null, false);
```

Incorrecto.

---

## 326.20 API mágica no analizable

Incorrecto.

---

## 326.21 Lazy loading durante serialization

Incorrecto por defecto.

---

## 326.22 `get()` para millones de registros como única opción

Incorrecto.

---

## 326.23 Cursor tratado como array

Incorrecto.

---

## 326.24 Bulk implementado con foreach/save

Incorrecto.

---

## 326.25 Exponer InternalPhysicalPlan en una interface pública

Incorrecto.

---

## 326.26 Cambiar semántica sin cambiar firma y llamarlo compatible

Incorrecto.

---

## 326.27 Exponer secrets en QueryException

Prohibido.

---

## 326.28 Retry automático después de commit ambiguo

Prohibido.

---

## 326.29 Full-table delete accidental

Deberá estar protegido.

---

## 326.30 Cross-tenant query accidental

Deberá estar protegido.

---

# 327. Modelo formal de la Public API

Sea:

```text
I
```

la arquitectura interna y:

```text
P
```

la API pública.

La relación deseada será:

```text
P = StableProjection(I)
```

No:

```text
P = I
```

La API pública es una proyección estable y controlada de capacidades internas.

---

# 328. Correctness Preservation

Para toda operación pública:

```text
op ∈ P
```

deberá existir una transformación:

```text
Normalize(op)
→ InternalOperation
```

tal que:

```text
Semantics(op)
=
Semantics(InternalOperation)
```

---

# 329. Convenience Equivalence

Si dos APIs públicas representan la misma operación:

```text
ModelAPI(op)
```

y:

```text
RepositoryAPI(op)
```

deberán converger conceptualmente:

```text
Normalize(ModelAPI(op))
≈
Normalize(RepositoryAPI(op))
```

bajo el mismo contexto.

---

# 330. Public Type Closure

Sea:

```text
Types(P)
```

el conjunto de tipos visibles en firmas públicas.

Debe cumplirse:

```text
Types(P)
∩
ForbiddenInternalTypes
=
∅
```

---

# 331. Scope Safety

Para estado mutable `S`:

```text
S(request A)
∩
S(request B)
=
∅
```

salvo objetos explícitamente inmutables/shareable.

---

# 332. Identity Invariant

Para:

```text
EntityType E
Identifier I
DatabaseContext C
```

dentro del mismo EntityManager scope:

```text
resolve(E, I, C)
=
same object identity
```

independientemente de si se accede mediante:

```text
Model API
Repository API
EntityManager API
```

---

# 333. Transaction Semantic Model

```text
persist(entity)
→ UnitOfWork registration
```

```text
flush()
→ synchronization attempt
```

```text
commit()
→ transaction outcome attempt
```

Por tanto:

```text
persist
≠
flush
≠
commit
```

---

# 334. Query Semantic Model

```text
Developer Query
      ↓
Normalization
      ↓
Query Model / AST
      ↓
Semantic Validation
      ↓
Optimization
      ↓
Planning
      ↓
Compilation
      ↓
Execution
```

Por tanto:

```text
Developer Query
≠
SQL String
```

---

# 335. API Safety Model

Una operación podrá clasificarse:

```text
Risk(op)
∈
{
    SAFE,
    ADVANCED,
    PRIVILEGED,
    UNSAFE
}
```

A mayor riesgo:

```text
higher explicitness
+
higher validation
+
higher observability
```

deberán ser requeridas.

---

# 336. Progressive Disclosure Model

```text
Developer Need
      ↓
Basic API
      ↓
Structured API
      ↓
Advanced API
      ↓
Infrastructure API
```

No será necesario comprender todo el sistema para ejecutar operaciones comunes.

---

# 337. API Design Decision Matrix

| Necesidad | API preferida |
|---|---|
| CRUD simple | Model API |
| Dominio/servicios | Repository |
| Coordinación ORM | EntityManager |
| Query relacional | Query API |
| Atomicidad | Transaction API |
| Schema | Schema API |
| Evolución | Migration API |
| Grandes volúmenes | Bulk/Streaming |
| Infraestructura | Advanced API |
| Diagnóstico | Inspection API |
| Extensión | Extension Contracts |
| DBMS específico | Platform Extension |
| Acceso nativo | Unsafe/Native API |

---

# 338. V1 Public API

La V1 deberá priorizar una superficie pequeña y coherente.

```text
Database
Connection
ConnectionManager

Query
SelectQuery
InsertQuery
UpdateQuery
DeleteQuery
Result
Cursor
Stream

EntityManager
Repository
Model
EntityQuery

TransactionManager

Schema
SchemaManager
Migration

Paginator
CursorPaginator

CapabilityView

DatabaseException hierarchy
```

---

# 339. V1 Convenience API

Podrá incluir:

```php
DB::connection();
DB::query();
DB::transaction();

User::query();
User::find();
User::findOrFail();

Schema::create();
```

siempre como adapters sobre los servicios reales.

---

# 340. V1 Advanced API

No será necesario declarar como estable toda la infraestructura interna desde V1.

Especialmente:

```text
optimizer internals
physical planner internals
compiler pipeline internals
hydration internals
persistence planner internals
```

podrán permanecer `INTERNAL` inicialmente.

---

# 341. Minimize Early Compatibility Debt

La regla será:

> **No convertir una clase interna en API pública estable sólo porque actualmente resulte útil para implementar otra feature.**

---

# 342. Public Extension Points First

Cuando exista necesidad real de extensibilidad se deberá diseñar:

```text
small stable contract
```

en vez de exponer:

```text
large internal subsystem
```

---

# 343. Arquitectura de namespaces propuesta

```text
src/Quantum/Database/
├── Contract/
│
├── Database.php
│
├── Connection/
│   ├── Contract/
│   └── ...
│
├── Query/
│   ├── Contract/
│   ├── Builder/
│   ├── Result/
│   └── ...
│
├── ORM/
│   ├── Contract/
│   ├── Model/
│   ├── Repository/
│   ├── EntityManager/
│   └── ...
│
├── Transaction/
│   ├── Contract/
│   └── ...
│
├── Schema/
│   ├── Contract/
│   └── ...
│
├── Migration/
│   ├── Contract/
│   └── ...
│
├── Pagination/
│
├── Bulk/
│
├── Capability/
│
├── Diagnostics/
│
├── Extension/
│
├── Exception/
│
└── Internal/
```

---

# 344. Public API Metadata Structure

Podrá existir:

```text
src/Quantum/Database/Api/
├── Attribute/
│   ├── PublicApi.php
│   ├── Experimental.php
│   ├── Internal.php
│   └── Deprecated.php
│
├── Stability/
│   └── ApiStability.php
│
├── Manifest/
│   ├── PublicApiManifest.php
│   └── PublicApiManifestGenerator.php
│
└── Validation/
    ├── PublicApiValidator.php
    └── InternalLeakDetector.php
```

---

# 345. Testing Structure

```text
tests/Quantum/Database/PublicApi/
├── Contract/
├── Model/
├── Repository/
├── Query/
├── Transaction/
├── Schema/
├── Connection/
├── Capability/
├── Security/
├── Runtime/
├── StaticAnalysis/
├── Compatibility/
└── DeveloperExperience/
```

---

# 346. Acceptance Criteria

La Public API podrá considerarse preparada para V1 cuando:

1. operaciones comunes requieran poca ceremonia;
2. DI funcione sin Facades;
3. Facades sean adapters opcionales;
4. Model API no mantenga static mutable state;
5. Model y Repository compartan ORM engine;
6. Query API produzca AST;
7. valores sean parameterized;
8. raw SQL sea explícito;
9. Transaction API preserve UNKNOWN outcomes;
10. `flush()` no implique commit;
11. streaming tenga lifecycle definido;
12. APIs bulk sean explícitas;
13. Schema API sea platform-aware;
14. capabilities sustituyan vendor checks;
15. errores sean normalizados;
16. secrets sean redacted;
17. runtime state sea request/operation scoped;
18. Public/Internal API estén clasificadas;
19. CI detecte leakage de tipos internos;
20. exista estrategia de compatibility testing.

---

# 347. Arquitectura consolidada

```text
                    VOLTSTACK APPLICATION
                            │
                            ▼
                   DATABASE PUBLIC API
                            │
       ┌────────────────────┼─────────────────────┐
       │                    │                     │
       ▼                    ▼                     ▼
   Model API          Repository API          Database API
       │                    │                     │
       └──────────────┬─────┴─────────────┬──────┘
                      ▼                   ▼
               Scoped Resolution      Query API
                      │                   │
                      ▼                   ▼
                 EntityManager       Query Model
                      │                   │
                      ▼                   ▼
                  UnitOfWork         Semantic Engine
                      │                   │
                      ▼                   ▼
             Persistence Engine        Planner
                      │                   │
                      └──────────┬────────┘
                                 ▼
                              Compiler
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

Sistemas paralelos:

```text
Public API
├── Transaction API
├── Schema API
├── Migration API
├── Pagination API
├── Streaming API
├── Bulk API
├── Capability API
├── Diagnostics API
└── Extension API
```

Adaptadores de conveniencia:

```text
Facades
Helpers
Static Model API
```

Todos deberán converger en:

```text
same scoped Database architecture
```

---

# 348. Principio final

La Public API deberá proteger simultáneamente dos objetivos aparentemente opuestos:

```text
Developer Simplicity
```

y:

```text
Architectural Rigor
```

VoltStack no deberá elegir uno sacrificando completamente el otro.

La regla final será:

> **La API pública de VoltStack Database deberá ocultar complejidad accidental, no complejidad semántica. El desarrollador podrá realizar operaciones comunes con una API breve y expresiva, mientras las invariantes de identidad, transacción, seguridad, capacidades, aislamiento y persistencia continúan siendo aplicadas por la arquitectura interna.**

Por tanto:

```text
Simple
≠
Simplistic
```

```text
Convenient
≠
Unsafe
```

```text
Fluent
≠
String-based
```

```text
Static Syntax
≠
Static State
```

```text
Model API
≠
Second ORM
```

```text
Repository API
≠
Second ORM
```

```text
Facade
≠
Core
```

```text
Helper
≠
Global State
```

```text
Query Builder
≠
SQL Builder
```

```text
persist()
≠
INSERT
```

```text
remove()
≠
DELETE
```

```text
flush()
≠
commit()
```

```text
Rollback
≠
Object Graph Rewind
```

```text
Result
≠
Array
```

```text
Cursor
≠
Collection
```

```text
Bulk
≠
foreach(save())
```

```text
Capability
≠
Vendor Check
```

```text
UNKNOWN
≠
UNSUPPORTED
```

```text
PHP public
≠
VoltStack Public API
```

y finalmente:

```text
VoltStack Database Public API
=
Developer Ergonomics
+
Stable Contracts
+
Strong Typing
+
Progressive Disclosure
+
Safe Defaults
+
Explicit Advanced Operations
+
Scoped Runtime State
+
Shared ORM Engine
+
Semantic Query API
+
Transaction Correctness
+
Capability Awareness
+
Security
+
Resource Safety
+
Diagnostics
+
IDE Discoverability
+
Backward Compatibility
```

---

# 349. Siguiente documento

```text
303_DATABASE_FACADE_SYSTEM.md
```

El siguiente documento definirá la capa de **Facades** de VoltStack Database y, especialmente, cómo proporcionar una experiencia similar a:

```php
DB::transaction(...);

DB::table('users')->get();

Schema::create(...);
```

sin introducir:

```text
global mutable database state
static EntityManager
static Connection
static TransactionContext
static TenantContext
```

La regla principal será:

> **Una Facade de VoltStack será una superficie estática de conveniencia sobre servicios resueltos desde el contexto actual; nunca será el propietario estático del servicio ni de su estado mutable.**

Es decir:

```text
Static Syntax
≠
Static State
```