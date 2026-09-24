# 326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md

## 1. Propósito

Este documento define la arquitectura del adaptador oficial de migración desde **Laravel Eloquent** hacia **VoltStack/Quantum/Database**.

El componente se denomina conceptualmente `EloquentMigrationAdapter` y forma parte de la infraestructura definida en `325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md`.

Su responsabilidad es comprender aplicaciones basadas en Eloquent, extraer su modelo de persistencia, normalizarlo hacia el modelo intermedio de VoltStack y proporcionar información para analizar, planificar, transformar, verificar y reportar la migración.

El adaptador no convierte `Quantum/Database` en una implementación compatible con Laravel. Eloquent es una fuente de migración; VoltStack Database es el destino.

## 2. Objetivos

El adaptador deberá:

- detectar automáticamente Eloquent y su versión;
- descubrir modelos y jerarquías de modelos;
- resolver tablas, conexiones e identificadores;
- analizar casts, timestamps y Soft Deletes;
- descubrir relaciones y sus claves físicas;
- analizar scopes locales y globales;
- descubrir accessors, mutators, observers y eventos;
- analizar consultas Eloquent y uso de `DB`;
- detectar SQL directo y extensiones personalizadas;
- congelar convenciones implícitas como metadata explícita;
- detectar acoplamiento con Eloquent fuera de los modelos;
- producir un modelo neutral consumible por el Migration Core;
- identificar transformaciones automáticas, riesgos y tareas manuales.

## 3. Principio arquitectónico

```text
Laravel / Eloquent
       │
       ▼
Discovery
       │
       ▼
Static Analysis
       │
       ├── optional Runtime Analysis
       ▼
Eloquent Semantic Model
       │
       ▼
Normalization
       │
       ▼
Intermediate Database Model
       │
       ▼
VoltStack Migration Pipeline
```

La dependencia siempre apunta desde el adapter hacia VoltStack:

```text
Eloquent Adapter → Migration Core → Quantum/Database
```

Nunca:

```text
Quantum/Database → Illuminate\Database
```

## 4. Distribución

Se recomienda distribuir el adapter como paquete separado:

```text
VoltStack/Database-Migration-Eloquent
```

Preferentemente será una dependencia de desarrollo removible al finalizar la migración.

## 5. Alcance

El adapter deberá comprender:

```text
Models
Tables
Connections
Primary Keys
Key Types
Incrementing
Fillable / Guarded
Casts
Timestamps
SoftDeletes
Relationships
Pivot Models
Morph Maps
Scopes
Global Scopes
Accessors / Mutators
Attribute Objects
Observers
Model Events
Factories / Seeders
Custom Builders
Custom Collections
DB Facade
Query Builder
Raw SQL
Transactions
Locking
Pagination
Chunking / Cursor processing
```

No deberá migrar componentes Laravel no relacionados con persistencia.

## 6. Compatibilidad por versión

Cada versión del adapter deberá declarar las versiones de Eloquent verificadas. Un `EloquentVersionProfile` describirá capacidades, diferencias y reglas aplicables.

```text
EloquentVersionDetector
        │
        ▼
Version Profile
        │
        ├── supported capabilities
        ├── migration rules
        └── known incompatibilities
```

No se asumirá equivalencia semántica entre versiones mayores.

## 7. Discovery Engine

La detección utilizará Composer, namespaces, herencia, traits y AST PHP.

```text
composer.json
     ↓
Dependency Detector
     ↓
PHP Source Scanner
     ↓
Model Detector
     ↓
Behavior / Query Detectors
     ↓
Eloquent Source Model
```

El análisis será **static-first**. No deberá ser necesario arrancar Laravel, conectar a producción ni ejecutar código arbitrario para realizar el inventario inicial.

## 8. Descubrimiento de modelos

El scanner resolverá modelos directos e indirectos:

```php
abstract class BaseModel extends Model {}
final class User extends BaseModel {}
```

Se resolverá la jerarquía completa hasta `Illuminate\Database\Eloquent\Model`.

Cada modelo producirá un `EloquentModelDescriptor` con:

```text
class
parent
traits
table
connection
primary key
key type
incrementing
timestamps
fillable
guarded
casts
relationships
scopes
events
observers
custom behavior
```

## 9. Resolución de tablas

El adapter deberá distinguir tablas explícitas e inferidas.

```php
protected $table = 'app_users';
```

se conserva directamente.

Si Eloquent infiere `UserProfile → user_profiles`, el resultado físico deberá congelarse explícitamente en la metadata migrada.

### Naming Freeze

Durante la migración:

```text
Implicit Eloquent convention
          ↓
Explicit physical mapping
```

Esto evita que diferencias de convención en VoltStack alteren el esquema utilizado.

## 10. Identificadores

Se detectarán:

```text
$primaryKey
$keyType
$incrementing
UUID
ULID
assigned identifiers
custom generators
```

La estrategia de generación se conservará independientemente del tipo de columna.

Claves compuestas implementadas mediante paquetes, traits o código personalizado se clasificarán para revisión salvo que exista una regla certificada.

## 11. Conexiones

El adapter analizará:

```php
protected $connection = 'mysql';
```

y resoluciones dinámicas como `setConnection()` o `getConnectionName()`.

Clasificación:

```text
STATIC
DYNAMIC
TENANT_AWARE
UNKNOWN
```

Una conexión dependiente del tenant o de estado runtime no podrá transformarse como una simple constante.

## 12. Fillable y Guarded

`$fillable` y `$guarded` se preservarán como información de asignación masiva, pero no se confundirán con:

```text
schema
validation
authorization
domain invariants
```

Asimismo, `$hidden` y `$visible` se reconocerán como concerns principalmente de serialización.

## 13. Casts

El adapter extraerá casts estándar:

```text
integer
float / double / real
decimal
string
boolean
array / object / json
date
datetime
immutable date/datetime
enum
```

y los convertirá a tipos semánticos intermedios antes de seleccionar el tipo VoltStack/plataforma.

### Custom Casts

Los casts personalizados deberán analizar:

```text
PHP representation
storage representation
serialization behavior
bidirectional conversion
```

Si no puede demostrarse equivalencia, se marcarán como `ADAPTABLE` o `MANUAL`.

## 14. Timestamps

Se detectará:

```php
public $timestamps = false;
```

así como nombres personalizados de `created_at` y `updated_at`.

No se asumirá que todos los modelos usan timestamps.

## 15. Soft Deletes

La presencia de `SoftDeletes` producirá metadata de comportamiento, no solamente una columna.

Se deberá preservar:

```text
deleted_at
default exclusion
with trashed
only trashed
restore
force delete
relationship interaction
```

## 16. Traits

Los traits se clasificarán como:

```text
KNOWN
ANALYZABLE
UNKNOWN
RISKY
```

incluyendo `SoftDeletes`, `HasFactory`, UUID/ULID y traits propios o de paquetes externos.

## 17. Relaciones

El analyzer deberá reconocer:

```text
hasOne
hasMany
belongsTo
belongsToMany
hasOneThrough
hasManyThrough
morphTo
morphOne
morphMany
morphToMany
morphedByMany
```

Para cada relación se resolverán, cuando sea posible:

```text
source
target
cardinality
foreign key
local/owner key
pivot table
pivot keys
constraints
fetch behavior
```

## 18. belongsTo / hasMany / hasOne

Ejemplo:

```php
return $this->belongsTo(User::class);
```

podrá normalizarse como:

```text
MANY_TO_ONE
source: Post
target: User
foreign_key: user_id
owner_key: id
```

Las claves inferidas por Eloquent deberán congelarse explícitamente.

## 19. Many-to-Many y Pivot

`belongsToMany()` deberá capturar:

```text
pivot table
foreign pivot key
related pivot key
parent/related keys
pivot columns
pivot timestamps
custom pivot model
```

Un custom Pivot puede representar una entidad asociativa y no deberá reducirse automáticamente a una tabla de unión trivial.

## 20. Relaciones polimórficas

Se conservarán:

```text
morph name
type column
id column
morph aliases
target types
```

Los `morphMap` son críticos: cambiar aliases puede invalidar datos históricos.

Si la base almacena FQCNs, el migrador deberá advertir sobre renombrados simultáneos de clases.

## 21. Constraints en relaciones

Una relación como:

```php
return $this->hasMany(Post::class)
    ->where('published', true)
    ->orderBy('created_at');
```

deberá separarse en:

```text
Structural Relationship
+
Query Constraints
```

para preservar su comportamiento.

## 22. Scopes locales

Los scopes simples se traducirán a predicados/query specifications cuando sea seguro.

Los scopes que dependan de usuario actual, tenant, servicios, reloj, APIs externas o PHP arbitrario se clasificarán para migración manual o adaptación.

## 23. Global Scopes

Los global scopes serán considerados comportamiento crítico.

Se detectarán registros mediante:

```text
addGlobalScope
boot / booted
traits
attributes compatibles
```

Ejemplos críticos:

```text
tenant filtering
visibility
active-record filtering
Soft Deletes
security filtering
```

Una entidad no se considerará verificada mientras sus global scopes relevantes permanezcan sin resolver.

## 24. Query Builder y API estática

El analyzer reconocerá patrones como:

```php
User::query();
User::where(...);
User::find(...);
User::first();
User::with(...);
```

Las APIs mágicas se normalizarán por semántica.

```text
User::find($id)
→ FIND_ENTITY_BY_IDENTIFIER
```

No se realizará una mera sustitución de nombres de métodos.

## 25. Dynamic Where

Expresiones como:

```php
User::whereEmail($email);
```

deberán convertirse preferentemente a predicados explícitos para reducir dependencia de magia dinámica.

## 26. Query Chains

Una cadena:

```php
User::query()
    ->where('active', true)
    ->whereNotNull('email')
    ->orderBy('name')
    ->limit(100)
    ->get();
```

podrá normalizarse al Intermediate Query Model y posteriormente al Query AST nativo de VoltStack.

## 27. Raw SQL

Se detectarán:

```text
DB::raw
selectRaw
whereRaw
orderByRaw
havingRaw
DB::select
DB::statement
```

Cuando una transformación segura no sea posible, preservar SQL parametrizado será preferible a producir una abstracción incorrecta.

El adapter deberá detectar concatenaciones potencialmente inseguras y reportarlas.

## 28. DB Facade

`DB::table()` y demás APIs sin modelos también forman parte del alcance.

Podrán migrarse directamente al Query Builder/Connection layer de VoltStack sin crear entidades artificiales.

## 29. Transacciones

Se analizarán:

```text
DB::transaction
beginTransaction
commit
rollBack
nested transactions
savepoints
deadlock retry behavior
```

Los límites transaccionales deberán preservarse semánticamente.

## 30. Locking

Se reconocerán:

```text
lockForUpdate
sharedLock
lock
```

y se preservará la intención de concurrencia y bloqueo.

## 31. Chunking y streaming

Se analizarán:

```text
chunk
chunkById
lazy
lazyById
cursor
```

La migración deberá conservar características relevantes de memoria, orden, progresión por ID y streaming.

`cursor()` no deberá convertirse accidentalmente en una carga completa en memoria.

## 32. Paginación

Se diferenciarán:

```text
paginate
simplePaginate
cursorPaginate
```

para mapear correctamente offset/simple/cursor-keyset pagination.

## 33. Eager y Lazy Loading

Se analizarán:

```text
with
load
loadMissing
withCount
withExists
withAggregate
```

y accesos potencialmente lazy como:

```php
$user->posts;
```

El análisis runtime opcional podrá detectar N+1 y diferencias de fetch behavior.

## 34. Accessors, Mutators y Attribute

Se reconocerán APIs legacy y modernas.

Cada transformación se clasificará como:

```text
domain
cast
serialization
presentation
derived property
```

Los accessors/mutators con side effects, queries o llamadas a servicios serán marcados como riesgos y no se transformarán automáticamente.

## 35. Eventos y Observers

Se inventariarán eventos de modelo y observers, incluyendo callbacks registrados mediante `boot()`/`booted()`.

Cada callback se clasificará como:

```text
ORM lifecycle
domain event
audit
integration
validation
side effect
```

El orden de ejecución deberá verificarse en la migración de comportamiento.

## 36. Touching

`$touches` deberá detectarse porque puede producir escrituras implícitas sobre entidades relacionadas.

Esta semántica no deberá desaparecer silenciosamente.

## 37. Serialización

Se inventariarán:

```text
hidden
visible
appends
toArray overrides
JSON serialization
date serialization
```

aunque el destino final de parte de esta lógica pueda encontrarse fuera de Database.

## 38. Builders, Collections y Macros personalizados

El adapter deberá descubrir:

```text
custom Eloquent Builder
custom Query Builder
custom Collections
Builder macros
Relation macros
Collection macros
mixins
```

Estos puntos pueden modificar profundamente la API observable de persistencia.

## 39. Factories y Seeders

Se analizarán como componentes migrables independientes.

Su conversión no deberá bloquear el cutover del runtime si pueden mantenerse temporalmente de forma segura.

## 40. Laravel Migrations

Las migraciones existentes podrán seguir estrategias:

```text
KEEP_HISTORY
BASELINE_CURRENT_SCHEMA
CONVERT_SELECTED
FULL_CONVERSION
```

Para aplicaciones maduras, la estrategia preferida normalmente será conservar el historial y establecer un baseline del esquema actual.

## 41. Triple fuente de verdad

El adapter comparará cuando sea posible:

```text
Eloquent Model
      +
Laravel Migration History
      +
Actual Database Schema
```

Si existen contradicciones, generará un `SCHEMA_CONFLICT` y evitará transformaciones que dependan de información ambigua.

## 42. Tablas dinámicas

`setTable()` o `getTable()` dinámicos deberán clasificarse como `DYNAMIC_TABLE`.

No deberán transformarse como mappings estáticos sin una estrategia explícita.

## 43. Multi-Tenancy

El adapter podrá detectar indicios de tenancy en:

```text
connections
tables
global scopes
boot callbacks
schema/database selection
```

La integración final pertenecerá al paquete oficial opcional de Multitenancy cuando esté instalado.

## 44. Semántica CRUD especial

El analyzer deberá comprender diferencias entre:

```text
find / findOrFail
first / firstOrFail / sole
create / forceCreate
firstOrCreate / firstOrNew
updateOrCreate
upsert
bulk update
bulk delete
truncate
replicate
```

Estas operaciones no deberán reducirse a CRUD genérico cuando sus garantías difieran.

## 45. Bulk operations

Las actualizaciones y eliminaciones masivas pueden omitir lifecycle hooks que sí se ejecutan al persistir entidades individualmente.

El Migration System deberá preservar esta diferencia.

## 46. Aggregates y subqueries

Se analizarán:

```text
count
sum
avg
min
max
exists
withCount
withSum
withAvg
selectSub
fromSub
joinSub
whereExists
```

y se normalizarán al Query AST cuando sea seguro.

## 47. Closures en consultas

Closures puramente estructurales podrán convertirse a grupos lógicos del AST.

Closures con filesystem, HTTP, servicios, reflection u otras operaciones arbitrarias deberán requerir revisión.

## 48. Introducción de Repository Boundary

Para aplicaciones fuertemente acopladas a Active Record, el migrador podrá facilitar una transición:

```text
Service
  ↓
Eloquent Model
```

hacia:

```text
Service
  ↓
Persistence Contract / Repository
  ↓
VoltStack Database
```

sin obligar a reestructurar toda la aplicación en una sola fase.

## 49. Modelo intermedio

La conversión conceptual será:

```text
Eloquent Model
      ↓
EloquentModelDescriptor
      ↓
IntermediateEntity
      ↓
VoltStack Entity Metadata
```

El conocimiento específico de Eloquent termina antes del core nativo de Database.

## 50. Componentes internos

```text
EloquentMigrationAdapter
│
├── EloquentVersionDetector
├── EloquentModelScanner
├── EloquentModelAnalyzer
├── EloquentRelationshipAnalyzer
├── EloquentCastAnalyzer
├── EloquentScopeAnalyzer
├── EloquentEventAnalyzer
├── EloquentQueryAnalyzer
├── EloquentSchemaAnalyzer
├── EloquentRuntimeObserver
└── EloquentNormalizer
```

## 51. Contrato conceptual

```php
final class EloquentMigrationAdapter
    implements OrmMigrationSourceAdapterInterface
{
    public function supports(MigrationSource $source): bool;

    public function inspect(MigrationSource $source): SourceModel;

    public function normalize(
        SourceModel $source
    ): IntermediateDatabaseModel;
}
```

## 52. Runtime Observer

El runtime observer será opcional y podrá observar:

```text
actual SQL
relationship loading
query counts
connection selection
transaction boundaries
```

No deberá alterar resultados de negocio y estará desactivado por defecto en producción.

## 53. Reglas de migración

Las reglas Eloquent usarán IDs estables:

```text
VSDB-MIG-ELOQ-0001
VSDB-MIG-ELOQ-0002
...
```

Cada hallazgo será clasificado como:

```text
DIRECT
TRANSFORMABLE
ADAPTABLE
MANUAL
UNSUPPORTED
RISKY
```

## 54. Ejemplo de análisis

Origen:

```php
final class Post extends Model
{
    use SoftDeletes;

    protected $fillable = ['title', 'body', 'user_id'];

    protected $casts = [
        'published' => 'boolean',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

Resultado conceptual:

```text
Entity: Post
Table: posts

Behaviors:
SOFT_DELETE
TIMESTAMPS

Known attributes:
title
body
user_id
published:boolean

Relationship:
Post MANY_TO_ONE User

Mass Assignment:
title
body
user_id
```

`$fillable` por sí solo no se considerará prueba suficiente de que un atributo es una columna física.

## 55. CLI

El adapter se integrará con el CLI general:

```bash
php volt database:migrate-orm --from=eloquent --analyze
```

y permitirá scopes como:

```text
entity
module
directory
package
```

Además soportará planificación y dry-run mediante el sistema general de migración.

## 56. Overrides explícitos

Cuando un valor no pueda resolverse estáticamente, el proyecto podrá proporcionar mappings explícitos de migración para tablas, conexiones, morph maps u otros elementos.

Los overrides no modificarán el código fuente original durante la fase de análisis.

## 57. Extensiones y paquetes

Paquetes Composer que extiendan Eloquent deberán clasificarse como:

```text
KNOWN_EXTENSION
UNKNOWN_EXTENSION
```

El adapter podrá tener un Extension Registry para plugins de migración específicos de paquetes o del propio proyecto.

Una extensión desconocida que modifique persistencia deberá producir una advertencia prioritaria.

## 58. Dual ORM Mode

Durante la transición podrá existir:

```text
Module A → Eloquent
Module B → VoltStack
```

pero ambos runtimes no deberán gestionar libremente el mismo agregado dentro de una unidad lógica de trabajo.

La coexistencia formal se especificará en `335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md`.

## 59. Shadow Reads

Consultas críticas podrán ejecutarse mediante:

```text
Eloquent Query ───► Result A
                       │
VoltStack Query ──► Result B
                       │
                       ▼
                  Comparator
```

Durante validación, solo el resultado primario se devolverá a la aplicación.

La comparación formal se define en `336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md`.

## 60. Characterization Tests

Antes de migrar modelos complejos podrán generarse pruebas que capturen comportamiento actual de:

```text
queries
relationships
casts
events
transactions
```

La misma expectativa deberá poder ejecutarse contra el sistema migrado.

## 61. Rendimiento

Se compararán:

```text
query count
database latency
hydration time
memory
lazy loads
```

Una migración funcionalmente correcta pero con una regresión grave de N+1 o memoria no se considerará completamente validada.

## 62. Seguridad

El adapter no deberá:

```text
exponer credenciales
persistir secretos
volcar datos sensibles
interpolar parámetros en SQL
incluir PII innecesaria en reportes
```

Los snapshots y diagnósticos deberán sanitizarse.

## 63. Runtimes persistentes

La migración hacia VoltStack/FrankenPHP deberá auditar:

```text
static model state
persistent observers
global mutable scopes
connection state
unclosed transactions
tenant leakage
```

El contexto de persistencia deberá poder reiniciarse correctamente entre requests.

## 64. Escaneo de aplicación

El análisis no se limitará a `app/Models`.

También buscará dependencias Eloquent en:

```text
Controllers
Services
Actions
Jobs
Commands
Listeners
Policies
Tests
Modules
Packages
```

Esto permite medir el verdadero acoplamiento de la aplicación.

## 65. Dependency Leakage

El reporte podrá cuantificar referencias de Eloquent fuera de la capa de persistencia.

El objetivo durante la migración será que las nuevas dependencias del ORM legacy no aumenten y finalmente lleguen a cero en runtime.

## 66. Dependency Graph

El adapter podrá construir grafos de modelos y relaciones para sugerir unidades coherentes de migración.

Relaciones circulares o componentes fuertemente conectados podrán requerir migración conjunta.

## 67. Separación de responsabilidades

El adapter se concentra en **comprender Eloquent y normalizarlo**.

Las siguientes responsabilidades pertenecen a documentos posteriores:

```text
329 → Migration Analysis Engine
330 → Intermediate Model
331 → Migration Rule Engine
332 → Code Transformer
333 → Schema Compatibility
334 → Behavior Verification
335 → Dual ORM Runtime
336 → Shadow Query Comparison
337 → Testing and Validation
338 → CLI and Developer Experience
339 → Reporting and Diagnostics
340 → Rollback and Recovery
```

## 68. Arquitectura

```text
┌─────────────────────────────────────────────────────┐
│                 Laravel Application                 │
├─────────────────────────────────────────────────────┤
│ Models │ Builders │ DB Facade │ Migrations │ Events │
└────┬────────┬──────────┬────────────┬──────────┬─────┘
     │        │          │            │          │
     └────────┴──────────┴────────────┴──────────┘
                         │
                         ▼
             EloquentMigrationAdapter
                         │
      ┌──────────────────┼───────────────────┐
      ▼                  ▼                   ▼
 Model Scanner     Query Analyzer      Behavior Analyzer
      │                  │                   │
      ├────────────┬─────┴───────┬───────────┤
      ▼            ▼             ▼           ▼
 Relations       Casts         Scopes       Events
      │            │             │           │
      └────────────┴──────┬──────┴───────────┘
                          ▼
               Eloquent Source Model
                          │
                          ▼
                     Normalizer
                          │
                          ▼
              Intermediate Database Model
                          │
                          ▼
                 Migration Rule Engine
                          │
                          ▼
                   VoltStack Model
```

## 69. Flujo recomendado

```text
1. Detect Eloquent version
          ↓
2. Discover model hierarchy
          ↓
3. Resolve physical tables/connections
          ↓
4. Extract identifiers
          ↓
5. Extract casts and behaviors
          ↓
6. Analyze relationships
          ↓
7. Analyze scopes
          ↓
8. Analyze events/observers
          ↓
9. Analyze query usage
          ↓
10. Inspect schema
          ↓
11. Detect conflicts
          ↓
12. Normalize
          ↓
13. Generate migration findings
```

## 70. Decisiones arquitectónicas

1. Eloquent tendrá un adapter oficial independiente del core.
2. Será preferentemente una dependencia de desarrollo removible.
3. El análisis será estático por defecto.
4. Las convenciones inferidas por Eloquent se congelarán como metadata explícita.
5. Los nombres físicos del esquema se preservarán por defecto.
6. Casts, scopes y extensiones ambiguas no se automatizarán sin una regla segura.
7. Los morph aliases y su representación persistida deberán preservarse.
8. Los global scopes se tratarán como comportamiento crítico.
9. Las APIs mágicas se convertirán por semántica, no por sustitución textual.
10. SQL nativo podrá conservarse cuando sea la opción más segura.
11. Migrar Eloquent no implicará rediseñar simultáneamente schema o dominio.
12. Se analizará el acoplamiento con Eloquent en toda la aplicación.
13. Dual ORM será temporal y controlado.
14. La migración deberá verificarse por schema, comportamiento, tests y rendimiento.
15. Finalizada la migración, Eloquent no será una dependencia runtime de VoltStack Database.

## 71. Criterios de finalización

Una migración Eloquent podrá declararse completada cuando:

```text
0 Eloquent models required at runtime
0 Eloquent builders required at runtime
0 DB facade persistence dependencies
0 unresolved critical global scopes
0 unresolved critical model events
0 unresolved critical custom casts
0 compatibility adapters required
schema verification passes
behavior verification passes
test suite passes
```

## 72. Resultado esperado

La transición final será:

```text
Laravel Application
       │
       ▼
Eloquent
       │
       ▼
Database
```

hacia:

```text
Application
       │
       ▼
VoltStack Persistence Contracts
       │
       ▼
Quantum/Database
       │
       ├── Entity Metadata
       ├── Repository
       ├── Unit of Work
       ├── Query AST
       ├── Transactions
       └── Drivers
               │
               ▼
            Database
```

sin requerir una reescritura monolítica.

## 73. Principio final

```text
Understand Eloquent
        ↓
Extract Semantics
        ↓
Freeze Implicit Conventions
        ↓
Normalize
        ↓
Transform Safely
        ↓
Verify Behavior
        ↓
Remove Eloquent
```

El objetivo no es que VoltStack se comporte como Laravel indefinidamente. El objetivo es permitir que una aplicación basada en Eloquent alcance de forma controlada una arquitectura nativa de VoltStack.

## 74. Conclusión

`DATABASE_ELOQUENT_MIGRATION_ADAPTER` constituye la capa especializada mediante la cual el Migration System comprende Eloquent sin contaminar la arquitectura de `Quantum/Database`.

La regla final es:

```text
Eloquent is a migration source.

VoltStack Database is the destination.

The migration layer disappears when its job is complete.
```

---

**Documento:** `326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**Estado:** Architectural Specification
