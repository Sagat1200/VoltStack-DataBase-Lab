# 09_DATABASE_EXTENSION_AND_CAPABILITY_MODEL.md

# VoltStack Quantum Database
## Extension and Capability Model

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 09 — Database Extension and Capability Model  
**Estado:** Architecture Specification  
**Nivel:** Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el modelo oficial de:

```text
Extension
+
Capability
```

para:

```text
VoltStack/Quantum/Database
```

El sistema deberá permitir extender Database sin:

- modificar constantemente el núcleo;
- introducir dependencias circulares;
- exponer componentes internos;
- acoplar subsistemas superiores a drivers concretos;
- utilizar comprobaciones dispersas por nombre de motor;
- permitir que extensiones violen invariantes arquitectónicos;
- comprometer persistent runtimes;
- convertir Database en un sistema de plugins sin control.

La arquitectura perseguirá:

> Núcleo estable, capacidades explícitas y extensiones controladas.

---

# 2. Dos conceptos diferentes

VoltStack distinguirá estrictamente:

```text
Extension
```

de:

```text
Capability
```

Una extensión modifica o amplía comportamiento.

Una capability describe comportamiento disponible.

Ejemplo:

```text
PostgreSQL Driver Extension
        │
        └── provides
               │
               ▼
        RETURNING capability
```

Pero la capability no es la extensión.

---

# 3. Extension

Una `Extension` representa una ampliación reconocida de Database.

Ejemplos:

```text
custom driver
custom dialect
custom platform
custom type
query extension
compiler extension
ORM extension
metadata extension
runtime adapter
telemetry adapter
cache integration
```

---

# 4. Capability

Una `Capability` representa una característica que puede estar disponible dentro de un contexto determinado.

Ejemplos:

```text
RETURNING
UPSERT
SAVEPOINTS
WINDOW_FUNCTIONS
CTE
RECURSIVE_CTE
JSON
JSON_QUERY
FULL_TEXT_SEARCH
TRANSACTIONAL_DDL
ADVISORY_LOCKS
SKIP_LOCKED
```

---

# 5. Problema que resuelve Capability

Sin este sistema es común terminar con:

```php
if ($driver === 'pgsql') {
    // ...
}

if ($driver === 'mysql') {
    // ...
}

if ($driver === 'sqlite') {
    // ...
}
```

distribuido por todo el framework.

VoltStack prohíbe esta arquitectura.

---

# 6. Regla fundamental

Los componentes superiores preguntarán:

```text
¿Qué puede hacer esta plataforma?
```

y no:

```text
¿Qué base de datos es?
```

Por tanto:

```php
$capabilities->supports(DatabaseCapability::RETURNING);
```

en lugar de:

```php
$driver === 'pgsql'
```

---

# 7. Arquitectura general

```text
Extension System
      │
      ├── Extension Contracts
      ├── Extension Descriptors
      ├── Extension Registry
      ├── Extension Resolver
      ├── Extension Validator
      └── Extension Lifecycle
                │
                ▼
        Capability Providers
                │
                ▼
       Capability Resolution
                │
                ▼
      Effective Capability Set
                │
                ▼
       Database Subsystems
```

---

# 8. Principio de extensión controlada

VoltStack no permitirá:

```text
Extension
   │
   ▼
arbitrary access to internals
```

La extensión deberá utilizar:

```text
public contracts
extension contracts
documented extension points
```

---

# 9. Extension boundary

Arquitectura:

```text
Database Internals
      │
      │ protected boundary
      ▼
Extension Contracts
      ▲
      │
Database Extensions
```

Las extensiones no deberán depender directamente de:

```text
Internal\
Implementation\
Private runtime objects
```

---

# 10. Extension API

Database tendrá una API específica para extensiones.

Conceptualmente:

```text
VoltStack\Quantum\Database\Extension
```

No todo API público será automáticamente API de extensión.

---

# 11. Clasificación de APIs

Database distinguirá:

| Tipo | Estabilidad |
|---|---|
| Public API | Alta |
| Extension API | Alta |
| Internal API | Sin garantía externa |
| Implementation Detail | No utilizable externamente |

---

# 12. Extension contract

Conceptualmente:

```php
interface DatabaseExtensionInterface
{
    public function descriptor(): ExtensionDescriptor;

    public function register(
        DatabaseExtensionRegistry $registry
    ): void;
}
```

La API definitiva podrá diferir.

---

# 13. ExtensionDescriptor

Cada extensión deberá declarar metadatos estructurados.

Ejemplo conceptual:

```php
final readonly class ExtensionDescriptor
{
    public function __construct(
        public ExtensionId $id,
        public string $name,
        public Version $version,
        public ExtensionType $type,
        public array $requirements,
        public array $provides,
    ) {}
}
```

---

# 14. ExtensionId

Toda extensión deberá poseer un identificador estable.

Ejemplo:

```text
voltstack.database.postgresql
vendor.database.spatial
vendor.database.custom-type.money
```

No depender exclusivamente del nombre de una clase PHP.

---

# 15. ExtensionType

Tipos conceptuales:

```text
DRIVER
DIALECT
PLATFORM
TYPE
QUERY
SEMANTIC
OPTIMIZER
PLANNER
COMPILER
EXECUTION
SCHEMA
MIGRATION
ORM
METADATA
HYDRATION
PERSISTENCE
RUNTIME
INTEGRATION
TOOLING
```

---

# 16. ExtensionRegistry

El registry deberá contener las extensiones instaladas y validadas.

No deberá convertirse en:

```text
global service locator
```

Su función es registrar descriptors/extensions.

---

# 17. No universal registry

No existirá:

```text
DatabaseRegistry
```

conteniendo indiscriminadamente:

```text
drivers
types
entities
connections
compilers
extensions
events
metadata
queries
```

Cada dominio tendrá registries especializados.

---

# 18. Specialized registries

Ejemplos:

```text
DriverRegistry
DialectRegistry
PlatformRegistry
TypeRegistry
CompilerRegistry
OptimizerRuleRegistry
MetadataLoaderRegistry
HydratorRegistry
ExtensionRegistry
```

---

# 19. Registry ownership

Los registries estructurales deberán configurarse durante:

```text
application bootstrap
```

y preferentemente congelarse antes del procesamiento normal.

---

# 20. Mutable bootstrap, immutable runtime

Modelo:

```text
BOOT
 │
 ▼
Mutable Registry
 │
 ├── register
 ├── validate
 └── compile
       │
       ▼
     freeze()
       │
       ▼
Immutable Runtime Registry
```

---

# 21. Beneficios del freeze

Permite:

```text
determinism
concurrency safety
faster lookup
persistent worker safety
architecture validation
```

---

# 22. Dynamic runtime extensions

No serán el modelo predeterminado.

Modificar el grafo de Database mientras existen requests activos introduce problemas de:

```text
concurrency
cache invalidation
compiled metadata
query compilation
service graph consistency
```

---

# 23. Hot registration

Si en el futuro se soporta:

```text
hot extension loading
```

deberá ser una capability explícita del runtime, no un efecto colateral del registry.

---

# 24. Extension discovery

Podrán existir mecanismos como:

```text
Composer package metadata
VoltStack package manifest
service provider
explicit configuration
compiled extension manifest
```

---

# 25. Explicit registration

Siempre deberá existir una forma explícita:

```php
Database::extend(
    new PostgreSqlExtension()
);
```

o equivalente mediante configuración/container.

---

# 26. Package discovery

Un paquete Composer podrá declarar que contiene una Database extension.

Ejemplo conceptual:

```text
vendor/package
      │
      ▼
VoltStack package metadata
      │
      ▼
Database extension discovered
```

Discovery no implica necesariamente activación.

---

# 27. Discovery vs registration

Se distinguirá:

```text
Discovered
```

de:

```text
Registered
```

de:

```text
Enabled
```

de:

```text
Active
```

---

# 28. Extension lifecycle

Estados conceptuales:

```text
Discovered
    │
    ▼
Registered
    │
    ▼
Validated
    │
    ▼
Enabled
    │
    ▼
Active
```

Fallos:

```text
Registered
    │
    ▼
Invalid
```

---

# 29. Disabled extension

Una extensión instalada podrá permanecer:

```text
Disabled
```

sin participar en runtime.

---

# 30. Extension validation

Antes de activarla deberán comprobarse:

```text
contract compatibility
VoltStack version
Database subsystem version
dependencies
conflicts
required capabilities
provided capabilities
runtime requirements
configuration validity
```

---

# 31. Extension dependencies

Una extensión podrá declarar:

```text
requires extension X
```

pero estas dependencias deberán formar un DAG.

---

# 32. Extension dependency graph

```text
Extension A
    │
    ▼
Extension B
    │
    ▼
Extension C
```

permitido.

No:

```text
Extension A
    │
    ▼
Extension B
    │
    └────► Extension A
```

---

# 33. Circular extension dependency

Deberá producir un error de bootstrap:

```text
CircularDatabaseExtensionDependencyException
```

---

# 34. Optional dependencies

Podrán declararse integraciones opcionales:

```text
uses-if-present
```

Ejemplo:

```text
Database Telemetry Extension
        │
        └── integrates with Quantum/Telemetry if installed
```

---

# 35. Extension ordering

Algunos extension points podrán necesitar orden.

Se evitará depender únicamente de:

```text
integer priority
```

cuando existan relaciones semánticas.

---

# 36. Ordering constraints

Podrán declararse:

```text
before
after
requires
conflicts
```

Ejemplo:

```text
Rule B
after Rule A
```

---

# 37. Deterministic ordering

Con la misma configuración, el orden deberá ser determinista.

Nunca depender de:

```text
filesystem order
Composer iteration accident
hash map order
```

---

# 38. Extension conflicts

Dos extensiones podrán declarar:

```text
conflictsWith
```

Ejemplo:

```text
two default PostgreSQL platform implementations
```

El bootstrap deberá rechazar configuraciones ambiguas.

---

# 39. Override semantics

Sobrescribir un componente oficial deberá ser explícito.

No:

```text
last registered wins
```

silenciosamente.

---

# 40. Replacement

Podrá existir un concepto:

```text
replaces
```

Ejemplo:

```text
CustomPostgreSqlCompiler
replaces
DefaultPostgreSqlCompiler
```

sujeto a contracts compatibles.

---

# 41. Decoration

Cuando no se necesite replacement completo:

```text
Core Component
      │
      ▼
Decorator A
      │
      ▼
Decorator B
```

podrá utilizarse un modelo de decoración.

---

# 42. Decorator ordering

El orden deberá ser explícito y determinista.

---

# 43. Extension isolation

Una extensión no deberá poder:

```text
mutate unrelated registries
access arbitrary scoped state
replace services silently
change security policies implicitly
disable parameter binding
```

---

# 44. Extension permissions

En el futuro podrá existir un descriptor de permisos/capabilities de extensión.

Ejemplo:

```text
register-driver
register-type
register-compiler-rule
observe-query
```

Esto permitiría mayor auditabilidad.

---

# 45. Persistent runtime safety

Toda extensión deberá declarar o demostrar que su estado persistente es:

```text
immutable
stateless
or concurrency-safe
```

---

# 46. Scoped extension state

Si una extensión necesita estado por ejecución deberá utilizar:

```text
DatabaseContext
ExecutionScope
specific scoped contracts
```

Nunca propiedades persistentes del extension object.

---

# 47. Unsafe extension example

```php
final class TenantExtension
{
    private ?Tenant $currentTenant = null;
}
```

si la extensión es singleton/persistente.

**Prohibido.**

---

# 48. Safe extension model

```text
Persistent Tenant Extension
         │
         ▼
TenantContextResolver
         │
         ▼
Execution-scoped TenantDatabaseContext
```

---

# 49. Extension shutdown

Extensiones que posean infraestructura persistente podrán participar en:

```text
runtime shutdown
```

mediante contracts específicos.

No todas las extensiones necesitan lifecycle hooks.

---

# 50. Extension hooks

Hooks deberán ser específicos.

Preferir:

```text
DriverExtension
TypeExtension
CompilerExtension
RuntimeExtension
```

sobre:

```text
onAnything()
```

---

# 51. Capability architecture

El sistema de capabilities responderá:

> ¿Está disponible una determinada característica bajo la configuración y contexto efectivos?

---

# 52. Capability identifier

Conceptualmente:

```php
enum DatabaseCapability: string
{
    case RETURNING = 'query.returning';
    case UPSERT = 'query.upsert';
    case SAVEPOINTS = 'transaction.savepoints';
}
```

No necesariamente deberá implementarse exclusivamente mediante enum para permitir extensiones externas.

---

# 53. Extensible capability IDs

Se recomienda un value object:

```php
final readonly class CapabilityId
{
    public function __construct(
        public string $value
    ) {}
}
```

permitiendo:

```text
core.query.returning
core.transaction.savepoints
vendor.spatial.geometry
```

---

# 54. Capability namespaces

Convención:

```text
database.<domain>.<feature>
```

o una convención equivalente estable.

Ejemplos:

```text
database.query.returning
database.query.cte
database.query.recursive_cte
database.query.window_functions

database.transaction.savepoints
database.transaction.isolation.serializable

database.schema.transactional_ddl

database.type.json
database.type.uuid
```

---

# 55. Capability provider

Una capability deberá provenir de un provider conocido.

Ejemplos:

```text
Driver
Platform
Dialect
Runtime
Extension
Configuration
```

---

# 56. Capability sources

Arquitectura:

```text
Driver Capabilities
        │
Platform Capabilities
        │
Dialect Capabilities
        │
Runtime Capabilities
        │
Extension Capabilities
        │
Configuration Constraints
        │
        ▼
Capability Resolver
        │
        ▼
Effective Capability Set
```

---

# 57. Effective capabilities

Los subsistemas superiores deberán consultar:

```text
EffectiveCapabilitySet
```

en vez de inspeccionar todos los providers.

---

# 58. CapabilitySet

Conceptualmente:

```php
interface CapabilitySetInterface
{
    public function supports(
        CapabilityId $capability
    ): bool;

    public function status(
        CapabilityId $capability
    ): CapabilityStatus;
}
```

---

# 59. CapabilityStatus

No basta siempre con boolean.

Estados:

```text
NATIVE
EMULATED
PARTIAL
DISABLED
UNSUPPORTED
```

---

# 60. Native capability

```text
NATIVE
```

significa que la plataforma puede ejecutar directamente la característica.

Ejemplo conceptual:

```text
PostgreSQL
RETURNING
→ NATIVE
```

---

# 61. Emulated capability

```text
EMULATED
```

significa que VoltStack puede ofrecer la semántica mediante una estrategia alternativa.

Ejemplo conceptual:

```text
Feature X
    │
    ▼
multiple operations / fallback
```

si puede garantizarse semántica equivalente.

---

# 62. Partial capability

```text
PARTIAL
```

indica que sólo una parte de la feature está disponible.

Debe incluir restricciones estructuradas.

---

# 63. Disabled capability

Una capability soportada técnicamente puede estar:

```text
DISABLED
```

por configuración o política.

---

# 64. Unsupported capability

```text
UNSUPPORTED
```

significa que VoltStack no puede garantizar esa semántica.

La operación deberá rechazarse antes de generar SQL inválido cuando sea posible.

---

# 65. Capability support is contextual

No siempre depende únicamente del motor.

Puede depender de:

```text
database vendor
database version
driver version
server configuration
runtime
installed extensions
application configuration
connection role
```

---

# 66. Version-sensitive capabilities

Ejemplo:

```text
DatabasePlatform
     │
     ├── vendor
     ├── version
     └── capabilities
```

Dos versiones del mismo motor pueden tener capabilities distintas.

---

# 67. No vendor assumptions

Incorrecto:

```php
if ($platform->name() === 'mysql') {
    return true;
}
```

Correcto:

```php
$capabilities->require(
    Capability::JSON_TABLE
);
```

---

# 68. CapabilityDescriptor

Cada capability podrá tener descriptor.

```php
final readonly class CapabilityDescriptor
{
    public function __construct(
        public CapabilityId $id,
        public CapabilityStatus $status,
        public ?string $provider = null,
        public array $constraints = [],
    ) {}
}
```

---

# 69. Capability constraints

Ejemplos:

```text
maximum parameter count
maximum identifier length
supported isolation levels
maximum bind variables
RETURNING supported only for certain operations
DDL transactional limitations
```

---

# 70. Capability values

Algunas capacidades no son booleanas.

Ejemplo:

```text
database.identifier.max_length = 63
database.query.max_parameters = 65535
database.transaction.isolation_levels = [...]
```

---

# 71. Capability model types

Se distinguirán:

```text
BooleanCapability
EnumCapability
SetCapability
NumericCapability
StructuredCapability
```

conceptualmente.

---

# 72. Feature vs capability

`Feature` representa funcionalidad del framework.

`Capability` representa requisito técnico.

Ejemplo:

```text
Feature:
Zero-downtime migration

requires:

rename column support
transactional DDL
online index capability
lock behavior knowledge
```

---

# 73. Feature requirement

Arquitectura:

```text
Database Feature
      │
      ▼
Capability Requirements
      │
      ▼
Effective Capability Set
      │
      ├── satisfied
      └── unsatisfied
```

---

# 74. CapabilityRequirement

Conceptualmente:

```php
interface CapabilityRequirement
{
    public function evaluate(
        CapabilitySetInterface $capabilities
    ): CapabilityRequirementResult;
}
```

---

# 75. Composite requirements

Podrán existir:

```text
ALL_OF
ANY_OF
NONE_OF
AT_LEAST
```

Ejemplo:

```text
RETURNING
OR
LAST_INSERT_ID
```

según una feature.

---

# 76. Capability negotiation

Una feature podrá negociar entre varias estrategias.

```text
Feature
   │
   ▼
Strategy Resolver
   │
   ├── Native Strategy
   ├── Emulation Strategy
   └── Unsupported
```

---

# 77. Strategy selection

Ejemplo:

```text
Insert and return generated values
          │
          ▼
Capability Resolver
          │
          ├── RETURNING
          │       └── ReturningStrategy
          │
          ├── GENERATED_KEYS
          │       └── GeneratedKeysStrategy
          │
          └── unsupported
```

---

# 78. Capability fallback

Un fallback sólo deberá declararse si conserva la semántica necesaria.

No:

```text
"approximately works"
```

---

# 79. Semantic equivalence

Una emulación deberá documentar:

```text
atomicity
consistency
concurrency behavior
performance impact
transaction requirements
limitations
```

---

# 80. Unsafe fallback

Si una emulación introduce race condition:

```text
INSERT
   │
   ▼
SELECT MAX(id)
```

no podrá considerarse capability equivalente.

---

# 81. Capability failure

Si falta una capability requerida:

```text
CapabilityNotSupportedException
```

deberá ser preferible a generar SQL que el servidor rechazará.

---

# 82. Capability error information

El error deberá indicar:

```text
requested feature
required capability
effective platform
capability status
available alternatives
```

---

# 83. Developer diagnostic

Ejemplo:

```text
DATABASE CAPABILITY ERROR

Feature:
    SELECT ... FOR UPDATE SKIP LOCKED

Required capability:
    database.query.lock.skip_locked

Platform:
    SQLite 3.x

Status:
    UNSUPPORTED

Alternative:
    none
```

---

# 84. Capability discovery

Podrá ocurrir mediante:

```text
static platform knowledge
server version
driver metadata
connection handshake
feature probes
configuration
```

---

# 85. Static capabilities

Muchas capabilities pueden conocerse sin conexión.

Ejemplo:

```text
SQLite dialect identifier quoting
```

---

# 86. Runtime-discovered capabilities

Algunas requieren conexión:

```text
server version
enabled server extension
session configuration
cluster capability
```

---

# 87. Lazy capability discovery

No deberá abrirse una conexión sólo para construir el Container si la capability no se necesita.

---

# 88. Capability discovery cache

Resultados estables podrán cachearse.

Pero deberán asociarse a:

```text
connection configuration
server identity/version
platform
```

cuando sea necesario.

---

# 89. Capability invalidation

El cache deberá invalidarse cuando cambie:

```text
server version
configuration
driver
installed DB extension
platform descriptor
```

---

# 90. Platform capability provider

`DatabasePlatform` será uno de los providers principales.

Conceptualmente:

```php
interface DatabasePlatformInterface
{
    public function capabilities(): CapabilitySetInterface;
}
```

---

# 91. Driver capabilities

Driver podrá declarar capacidades de transporte/API.

Ejemplos:

```text
prepared statements
server-side cursors
query cancellation
async execution
multiple result sets
```

Estas no son necesariamente capabilities SQL.

---

# 92. Dialect capabilities

Dialect podrá aportar conocimiento sintáctico.

Ejemplos:

```text
RETURNING syntax
LIMIT/OFFSET syntax
identifier quoting
upsert grammar
```

---

# 93. Platform capabilities

Platform representa semántica del motor.

Ejemplos:

```text
transactional DDL
isolation levels
JSON semantics
locking behavior
schema behavior
```

---

# 94. Runtime capabilities

Runtime puede aportar:

```text
concurrency
cancellation
worker lifecycle
async operations
context-local storage
```

Estas capabilities pueden afectar Database aunque no pertenezcan al motor SQL.

---

# 95. Connection capabilities

Una conexión concreta podrá tener capabilities efectivas adicionales.

Ejemplo:

```text
read-only
replica
specific server version
enabled extension
```

---

# 96. Capability layering

```text
Global Database Capabilities
            │
            ▼
Configured Platform Capabilities
            │
            ▼
Connection Capabilities
            │
            ▼
Operation Context Capabilities
```

Cada nivel puede restringir, pero no inventar soporte inexistente.

---

# 97. Capability narrowing

Ejemplo:

```text
Platform supports writes
        │
        ▼
Replica connection
        │
        ▼
WRITE capability disabled
```

---

# 98. Capability monotonic safety

Los contextos inferiores pueden:

```text
restrict
specialize
```

capabilities.

No deberán elevar una capability sin provider que la garantice.

---

# 99. Configuration and capabilities

Configuración podrá desactivar features:

```yaml
database:
  features:
    query_cache: false
```

aunque técnicamente estén disponibles.

---

# 100. Policy capabilities

Se distinguirá:

```text
technical capability
```

de:

```text
enabled policy
```

para no confundir:

```text
can do
```

con:

```text
may do
```

---

# 101. Capability composition

El resolver combinará providers según reglas explícitas.

No:

```text
last provider wins
```

arbitrariamente.

---

# 102. Capability precedence

La precedencia conceptual puede ser:

```text
Base Platform
      │
      ▼
Driver Constraints
      │
      ▼
Server Discovery
      │
      ▼
Connection Constraints
      │
      ▼
Configuration Restrictions
      │
      ▼
Runtime Restrictions
      │
      ▼
Effective Capabilities
```

---

# 103. Contradictory providers

Si:

```text
Provider A → supported
Provider B → unsupported
```

el resolver deberá conocer la semántica de ambos.

No escoger silenciosamente.

---

# 104. Restriction wins

Como principio de seguridad:

> Una restricción contextual explícita tendrá precedencia sobre una capacidad general.

---

# 105. Capability provenance

El resultado debería poder explicar:

```text
why
```

una capability tiene cierto estado.

Ejemplo:

```text
query.returning
status: NATIVE
source: PostgreSQLPlatform
version: 17
```

---

# 106. Explain capability

En desarrollo podrá existir:

```php
DB::capabilities()->explain(
    'database.query.returning'
);
```

conceptualmente.

---

# 107. CLI capability inspection

Podrá existir:

```text
php volt database:capabilities
```

Ejemplo:

```text
Platform: PostgreSQL

RETURNING             native
UPSERT                native
SAVEPOINTS            native
TRANSACTIONAL_DDL     native
JSON                   native
QUERY_CANCELLATION    native
```

---

# 108. Capability matrix

La documentación podrá generar automáticamente matrices.

| Capability | MySQL | MariaDB | PostgreSQL | SQLite |
|---|---|---|---|---|
| CTE | resolved | resolved | resolved | resolved |
| RETURNING | resolved | resolved | resolved | resolved |
| Savepoints | resolved | resolved | resolved | resolved |
| JSON | resolved | resolved | resolved | resolved |

Los valores concretos serán responsabilidad de los documentos Platform.

---

# 109. No hardcoded matrix in upper layers

La tabla documental no deberá convertirse en lógica:

```php
switch ($databaseName) {}
```

dentro de Query/ORM.

---

# 110. Query Builder and capabilities

Query Builder deberá poder representar una intención aunque la validación de soporte ocurra posteriormente.

Ejemplo:

```php
$query->returning(['id']);
```

produce Query Model/AST.

Después:

```text
Semantic / Planning / Compilation
        │
        ▼
Capability Validation
```

---

# 111. Early vs late capability validation

Algunas capabilities pueden validarse temprano.

Otras sólo cuando se conoce:

```text
effective connection/platform
```

Por ello habrá varios puntos controlados de validación.

---

# 112. AST portability

El AST no deberá contener:

```text
if PostgreSQL
if MySQL
```

Deberá expresar intención semántica.

---

# 113. Compiler capability use

El Compiler puede recibir:

```text
Dialect
Platform
CapabilitySet
```

para elegir generación válida.

---

# 114. Optimizer capability use

El Optimizer podrá habilitar reglas sólo cuando las capabilities permitan preservar semántica.

---

# 115. Planner capability use

El Planner podrá escoger:

```text
Native Plan
Fallback Plan
Unsupported Plan
```

---

# 116. Schema capability use

Schema deberá consultar capabilities para:

```text
generated columns
partial indexes
expression indexes
deferrable constraints
transactional DDL
```

---

# 117. Migration capability use

Migration Planner podrá determinar:

```text
direct alteration
table rebuild
multi-step migration
unsupported operation
```

según capabilities.

---

# 118. ORM capability use

ORM podrá seleccionar estrategias para:

```text
generated IDs
batch inserts
locking
RETURNING
upsert
```

sin conocer vendor names.

---

# 119. Transaction capability use

Transaction Manager consultará:

```text
savepoints
isolation levels
read-only transactions
deferrable transactions
```

---

# 120. Hydration capability independence

Hydration normalmente no deberá depender de vendor.

Las diferencias de representación deberán resolverse antes mediante:

```text
Result
Type System
Value Conversion
```

---

# 121. Type capability extensions

Un paquete podrá registrar:

```text
MoneyType
GeometryType
VectorType
```

mediante Type Extension contracts.

---

# 122. Custom type registration

Conceptualmente:

```php
$types->register(
    MoneyType::class
);
```

pero la implementación deberá incluir descriptor y validación.

---

# 123. Type collision

Dos extensiones no podrán registrar silenciosamente el mismo type ID.

Deberá requerirse:

```text
explicit replacement
```

---

# 124. Driver extension

Un driver extension deberá poder proporcionar:

```text
DriverFactory
DriverDescriptor
DriverCapabilities
native error translation
connection primitives
```

No ORM.

---

# 125. Dialect extension

Podrá proporcionar:

```text
identifier quoting
placeholder syntax
SQL rendering primitives
dialect compiler helpers
```

No abrir conexiones.

---

# 126. Platform extension

Podrá proporcionar:

```text
platform semantics
schema behavior
capabilities
type mappings
locking semantics
```

No administrar EntityManager.

---

# 127. Query extension

Podrá añadir:

```text
AST node
expression
semantic rule
planner rule
compiler handler
```

si el extension point correspondiente lo permite.

---

# 128. Complete query extension

Una nueva construcción SQL puede necesitar:

```text
AST
  │
  ▼
Semantic
  │
  ▼
Planner
  │
  ▼
Compiler
```

La extensión deberá registrar todas las piezas requeridas.

---

# 129. Partial query extension validation

Si registra:

```text
CustomAstNode
```

pero no existe compiler/planner compatible:

```text
ExtensionValidationException
```

durante bootstrap cuando pueda detectarse.

---

# 130. Optimizer extension

Podrá registrar:

```text
OptimizationRule
```

con:

```text
phase
priority/order
requirements
capability requirements
```

---

# 131. Optimizer rule purity

Una regla no deberá:

```text
execute SQL
open connections
modify EntityManager
emit events with side effects
```

---

# 132. Compiler extension

Podrá registrar handlers para:

```text
AST/Plan node
+
Dialect/Platform
```

pero no tendrá acceso directo al ConnectionManager.

---

# 133. Schema extension

Puede introducir:

```text
custom schema object
platform schema feature
specialized index
```

si mantiene el Schema Model estructurado.

---

# 134. ORM extension

Podrá ampliar:

```text
metadata
mapping
entity lifecycle
repository behavior
persistence strategy
```

mediante contratos específicos.

No podrá saltarse UnitOfWork/Persistence Engine para generar SQL arbitrariamente como comportamiento normal.

---

# 135. Metadata extension

Podrá proporcionar:

```text
AttributeMetadataLoader
XMLMetadataLoader
CustomMetadataSource
```

que produzca el modelo canónico de metadata.

---

# 136. Metadata normalization

Todas las fuentes:

```text
Attributes
XML
PHP
Custom
```

deberán terminar en:

```text
Canonical Metadata Model
```

---

# 137. Runtime extension

Podrá adaptar:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

al lifecycle neutral definido en `08_DATABASE_LIFECYCLE_AND_RUNTIME_MODEL.md`.

---

# 138. Integration extension

Integraciones con:

```text
Cache
EventSystem
Telemetry
Multitenancy
Validation
Authentication
Authorization
Jobs
RuntimeManagerServer
```

deberán depender de ports/contracts.

---

# 139. Cache extension

Database Core podrá exponer:

```text
DatabaseCachePort
```

y un adapter:

```text
QuantumCacheDatabaseAdapter
```

podrá conectarlo con `Quantum/Cache`.

---

# 140. Telemetry extension

Arquitectura:

```text
Database
   │
   ▼
Telemetry Port
   ▲
   │
Quantum/Telemetry Adapter
```

No:

```text
Database Core
   │
   ▼
Quantum/Telemetry implementation
```

---

# 141. Event extension

Mismo principio:

```text
Database Event Port
       ▲
       │
Quantum/EventSystem Adapter
```

---

# 142. Multitenancy extension

Multitenancy podrá aportar:

```text
TenantConnectionResolver
TenantDatabaseContextFactory
TenantSchemaResolver
```

sin modificar Database Core.

---

# 143. Extension capability provider

Una extensión podrá declarar:

```text
provides capability
```

pero deberá demostrar cómo se implementa.

Registrar el descriptor no hace mágicamente disponible la feature.

---

# 144. Capability ownership

Cada capability deberá tener:

```text
provider
status
scope
constraints
provenance
```

---

# 145. Capability aliases

Se evitarán aliases múltiples para la misma semántica.

Debe existir un ID canónico.

---

# 146. Capability deprecation

Una capability podrá marcarse deprecated cuando evolucione el modelo.

Deberá existir una ruta de migración.

---

# 147. Extension versioning

Cada Extension API deberá seguir la política de versionado de VoltStack.

Cambiar un Extension Contract estable será considerado cambio relevante de compatibilidad.

---

# 148. Extension compatibility

Una extensión podrá declarar:

```text
database-api >= 1.0 < 2.0
```

conceptualmente.

---

# 149. Compatibility check

Bootstrap:

```text
Extension
    │
    ▼
Compatibility Resolver
    │
    ├── compatible
    └── incompatible → fail
```

---

# 150. Extension migration

Si una extensión necesita actualizar configuración o metadata, deberá proporcionar mecanismos explícitos.

No modificar silenciosamente estado persistente durante request.

---

# 151. Extension boot phases

Fases recomendadas:

```text
Discover
   │
   ▼
Register Descriptors
   │
   ▼
Resolve Dependencies
   │
   ▼
Validate Compatibility
   │
   ▼
Register Components
   │
   ▼
Resolve Capabilities
   │
   ▼
Compile Registries
   │
   ▼
Freeze
   │
   ▼
Runtime
```

---

# 152. No connection during basic registration

Registrar una extensión no deberá abrir una conexión por defecto.

Discovery dependiente del servidor deberá ser lazy.

---

# 153. Compile extension graph

En production podrá generarse:

```text
compiled database extension manifest
```

para evitar discovery repetitivo.

---

# 154. Compiled capability metadata

Capabilities estáticas podrán precompilarse.

Capabilities dependientes del servidor se resolverán posteriormente.

---

# 155. Development mode

Development podrá mantener más validaciones y diagnósticos:

```text
duplicate extension detection
boundary validation
capability provenance
extension graph inspection
```

---

# 156. Production mode

Production deberá favorecer:

```text
compiled registries
frozen extension graph
prevalidated descriptors
fast capability lookup
minimal reflection
```

---

# 157. Extension error hierarchy

Conceptualmente:

```text
DatabaseExtensionException
├── ExtensionDiscoveryException
├── ExtensionRegistrationException
├── ExtensionValidationException
├── ExtensionCompatibilityException
├── ExtensionConflictException
├── ExtensionDependencyException
├── CircularExtensionDependencyException
└── ExtensionRuntimeException
```

---

# 158. Capability error hierarchy

```text
DatabaseCapabilityException
├── CapabilityNotFoundException
├── CapabilityNotSupportedException
├── CapabilityConflictException
├── CapabilityResolutionException
├── CapabilityRequirementException
└── CapabilityDiscoveryException
```

---

# 159. Extension failure at bootstrap

Una extensión requerida que falla deberá detener el bootstrap.

No continuar con un sistema parcialmente registrado.

---

# 160. Optional extension failure

Una integración explícitamente opcional podrá:

```text
disable itself
+
report diagnostic
```

si la política lo permite.

---

# 161. Runtime extension failure

Si una extensión activa falla durante runtime, su manejo dependerá del extension point.

No deberá asumirse que todas las extensiones pueden ignorarse.

---

# 162. Observational extension failure

Ejemplo:

```text
Telemetry Adapter failure
```

normalmente no deberá hacer fallar la query.

---

# 163. Semantic extension failure

Ejemplo:

```text
Custom Encryption Type failure
```

sí puede hacer inválida la operación.

---

# 164. Criticality descriptor

Las extensiones podrán declarar conceptualmente:

```text
OBSERVATIONAL
OPTIONAL
FUNCTIONAL
CRITICAL
```

aunque la política definitiva deberá evitar que una extensión se autoasigne privilegios injustificados.

---

# 165. Security extensions

Una extensión de seguridad no podrá ser bypassed por fallback automático.

Ejemplo:

```text
encrypted database field mapping
```

si falla, no deberá almacenar plaintext silenciosamente.

---

# 166. Capability security

Una capability técnica no implica autorización.

Ejemplo:

```text
database supports DROP TABLE
```

no significa:

```text
application may DROP TABLE
```

Capability y authorization son dominios distintos.

---

# 167. Capability and validation

Igualmente:

```text
database supports VARCHAR(65535)
```

no sustituye validación de dominio/aplicación.

---

# 168. Capability and configuration

Configuración podrá exigir:

```text
minimum capabilities
```

Ejemplo:

```yaml
database:
  require:
    - database.transaction.savepoints
    - database.query.cte
```

---

# 169. Startup capability requirements

Si pueden determinarse durante bootstrap:

```text
missing required capability
→ fail fast
```

---

# 170. Deferred capability requirements

Si requieren conexión:

```text
bootstrap
   │
   ▼
requirement registered
   │
   ▼
first connection
   │
   ▼
capability discovery
   │
   ▼
validate
```

---

# 171. Environment capability profiles

Podrán existir perfiles:

```text
development
testing
production
```

pero no deberán ocultar diferencias críticas entre motores.

---

# 172. Testing fake capabilities

Testing podrá construir:

```text
FakeCapabilitySet
```

para probar:

```text
native path
fallback path
unsupported path
```

sin requerir todos los motores.

---

# 173. Driver conformance

Cada driver oficial deberá pasar una suite que valide las capabilities que declara.

No basta declarar:

```text
supportsSavepoints = true
```

---

# 174. Capability conformance

Regla:

> Una capability declarada constituye una promesa arquitectónica.

Si se declara:

```text
NATIVE
```

la implementación deberá satisfacer su contrato.

---

# 175. Emulation conformance

Una capability `EMULATED` deberá tener tests de equivalencia semántica.

---

# 176. Capability matrix testing

Las plataformas oficiales:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

deberán ejecutar suites específicas.

---

# 177. Extension conformance

Podrá existir:

```text
DatabaseExtensionConformanceSuite
```

para validar paquetes externos.

---

# 178. Extension boundary tests

Deberán comprobar:

```text
no Internal namespace imports
no PDO access from forbidden extension types
no global mutable scoped state
no circular dependencies
valid extension contracts
```

---

# 179. Architecture tests

CI deberá impedir dependencias como:

```text
Query\Extension
    │
    ▼
Connection\Internal
```

si no están permitidas.

---

# 180. Capability architecture tests

También deberán detectar:

```text
if ($driver === ...)
switch ($databaseVendor)
```

en capas donde está prohibido.

---

# 181. Allowed vendor-specific code

Código específico del vendor sí será válido dentro de:

```text
Driver
Dialect
Platform
vendor-specific Compiler
vendor-specific Schema adapter
```

---

# 182. Vendor specialization boundary

```text
Generic Database Core
          │
          ▼
Capability Contracts
          │
          ▼
Platform Specialization
          │
          ├── MySQL
          ├── MariaDB
          ├── PostgreSQL
          └── SQLite
```

---

# 183. Extension directories

Estructura conceptual:

```text
Database/
├── Extension/
│   ├── Contracts/
│   ├── Descriptor/
│   ├── Registry/
│   ├── Resolver/
│   ├── Validation/
│   └── Exception/
│
└── Capability/
    ├── Contracts/
    ├── Descriptor/
    ├── Provider/
    ├── Resolver/
    ├── Requirement/
    ├── Strategy/
    └── Exception/
```

La estructura definitiva será establecida en `330_DATABASE_DIRECTORY_STRUCTURE.md`.

---

# 184. Core extension points

Inicialmente podrán existir extension points para:

```text
Driver
Dialect
Platform
Type
Query AST
Expression
Semantic Rule
Optimizer Rule
Planner Strategy
Compiler Handler
Schema Feature
Metadata Loader
Hydrator
Persistence Strategy
Runtime Adapter
Integration Adapter
```

---

# 185. Extension point descriptor

Cada extension point deberá documentar:

```text
accepted contract
lifetime
allowed dependencies
ordering semantics
state policy
failure policy
stability
```

---

# 186. Not everything is extensible

VoltStack no deberá convertir cada clase en:

```text
interface
```

sólo por extensibilidad hipotética.

---

# 187. Intentional extension points

Un extension point existirá cuando haya:

```text
clear use case
stable semantic boundary
testable contract
safe lifecycle
```

---

# 188. Internal customization

Algunas personalizaciones deberán realizarse mediante:

```text
configuration
strategy
policy
```

en lugar de plugins.

---

# 189. Extension vs strategy

Ejemplo:

```text
Read replica selection algorithm
```

puede ser una Strategy dentro del sistema existente.

No necesariamente una Database Extension completa.

---

# 190. Extension vs adapter

Una integración externa normalmente será:

```text
Adapter
```

que puede ser registrada por una Extension.

Los términos no son equivalentes.

---

# 191. Extension vs provider

Un provider aporta información/servicio a un extension point.

Una extensión puede registrar múltiples providers.

---

# 192. Extension package

Ejemplo conceptual:

```text
voltstack/database-postgresql
```

podría contener:

```text
PostgreSqlDriver
PostgreSqlDialect
PostgreSqlPlatform
PostgreSqlCompiler
PostgreSqlSchemaCompiler
PostgreSqlCapabilityProvider
```

coordinados por:

```text
PostgreSqlDatabaseExtension
```

---

# 193. Official extensions

Las extensiones oficiales podrán vivir dentro del repositorio principal o paquetes oficiales según la estrategia final.

Arquitectónicamente deberán respetar los mismos contracts que extensiones externas cuando sea razonable.

---

# 194. Core privilege minimization

Una extensión oficial no deberá recibir acceso arbitrario a internals sólo por ser oficial.

Esto ayuda a probar que la arquitectura de extensiones es real.

---

# 195. Extension portability

Una Query Extension genérica podrá funcionar en varias plataformas si:

```text
required capabilities
```

están disponibles.

---

# 196. Capability-driven extension

Ejemplo:

```text
Vector Search Extension
       │
       ▼
requires vector capability
       │
       ├── PostgreSQL provider
       ├── MySQL provider
       └── custom provider
```

La extensión superior no necesita preguntar vendor.

---

# 197. Feature strategy registry

Podrá existir un registry especializado:

```text
FeatureStrategyRegistry
```

que relacione:

```text
feature
+
capability predicate
+
strategy
```

---

# 198. Strategy selection example

```text
UPSERT
  │
  ▼
Capability Resolution
  │
  ├── NativeOnConflictStrategy
  ├── NativeDuplicateKeyStrategy
  ├── SafeTransactionalFallback
  └── Unsupported
```

La selección puede estar en Planner/Compiler según responsabilidad.

---

# 199. Semantic intent first

El Query Model deberá expresar:

```text
UPSERT intent
```

no:

```text
ON CONFLICT
```

o:

```text
ON DUPLICATE KEY
```

cuando la intención sea portable.

---

# 200. Vendor escape hatch

Si el desarrollador desea una feature explícitamente vendor-specific, podrá usar un extension point/raw expression documentado.

La portabilidad deja de estar garantizada de forma explícita.

---

# 201. Capability snapshot

Durante una operación podrá utilizarse un:

```text
CapabilitySnapshot
```

inmutable.

Esto evita que las capabilities cambien a mitad de planificación/compilación.

---

# 202. Deterministic compilation

Misma combinación:

```text
Query Model
+
Capability Snapshot
+
Dialect
+
Compiler Version
```

deberá producir compilación determinista.

---

# 203. Capability fingerprint

Podrá calcularse:

```text
CapabilityFingerprint
```

para participar en caches de:

```text
compiled queries
plans
schema compilation
```

---

# 204. Cache correctness

Una compiled query generada bajo:

```text
CapabilitySet A
```

no deberá reutilizarse bajo un set incompatible:

```text
CapabilitySet B
```

---

# 205. Extension fingerprint

Igualmente podrá existir:

```text
ExtensionGraphFingerprint
```

para invalidar artefactos compilados cuando cambian extensiones.

---

# 206. Metadata cache interaction

Si una ORM extension cambia metadata:

```text
metadata cache
```

deberá invalidarse mediante version/fingerprint apropiado.

---

# 207. Query cache interaction

Si una extensión altera compilación:

```text
compiled query cache
```

deberá distinguir esa configuración.

---

# 208. Capability cache interaction

Capability cache no deberá confundirse con:

```text
query result cache
```

Es metadata técnica de plataforma.

---

# 209. Serialization

Descriptors/capability sets compilables deberían poder serializarse cuando sea seguro.

Esto facilita:

```text
production cache
preloading
worker startup
```

---

# 210. No serialization of scoped state

Nunca incluir:

```text
connection
transaction
tenant
EntityManager
UnitOfWork
```

en artefactos persistentes de extension/capability compilation.

---

# 211. Extension observability

Telemetry podrá registrar:

```text
loaded extensions
extension versions
capability resolution
fallback strategy selected
unsupported capability attempts
```

sin exponer secretos.

---

# 212. Capability telemetry

Ejemplo:

```text
database.capability.fallback
feature = query.upsert
strategy = transactional_fallback
```

---

# 213. Avoid high-cardinality telemetry

No deberán incluirse indiscriminadamente:

```text
tenant IDs
query values
credentials
dynamic user data
```

en capability telemetry.

---

# 214. Extension diagnostics command

Conceptualmente:

```text
php volt database:extensions
```

podría mostrar:

```text
ID                              STATUS      VERSION
voltstack.database.mysql        active      1.0
voltstack.database.telemetry    active      1.0
vendor.database.spatial         disabled    2.3
```

---

# 215. Extension graph command

```text
php volt database:extensions --graph
```

podrá mostrar:

```text
PostgreSQL
├── Driver
├── Dialect
├── Platform
├── Compiler
└── Schema
```

---

# 216. Extension explain

```text
php volt database:extension vendor.database.spatial
```

podrá mostrar:

```text
requires
provides
conflicts
extension points
capabilities
version compatibility
```

---

# 217. Capability explain command

```text
php volt database:capability database.query.returning
```

podrá mostrar provenance y restricciones.

---

# 218. Debug toolbar integration

Developer Debug Toolbar podrá mostrar:

```text
Database Platform
Effective Capabilities
Fallbacks Used
Extensions Active
```

sin introducir dependencia desde Core hacia Toolbar.

---

# 219. Extension security review

Extensiones que toquen:

```text
credentials
SQL generation
parameter binding
data encryption
connection routing
tenant isolation
```

deberán considerarse security-sensitive.

---

# 220. Compiler extension security

Un custom compiler deberá seguir:

```text
identifier escaping
parameter binding
raw expression boundaries
```

No podrá degradar silenciosamente prepared statements.

---

# 221. Driver extension security

Debe respetar:

```text
TLS configuration
credential handling
error masking
connection reset
```

---

# 222. Capability spoofing

Una extensión no deberá poder declarar:

```text
transaction.savepoints = native
```

sin registrar un provider válido dentro del extension point correspondiente.

---

# 223. Capability trust model

Providers oficiales y externos pueden tener distinto origen, pero el Core validará estructura y contracts.

La confianza semántica final se verificará mediante conformance testing.

---

# 224. Extension sandboxing

PHP no ofrece aislamiento de seguridad fuerte entre paquetes ejecutados en el mismo proceso.

Por tanto:

> Extension boundaries son fronteras arquitectónicas, no un sandbox de código hostil.

Sólo deben instalarse paquetes confiables.

---

# 225. Capability resolution flow

```text
Feature Requested
       │
       ▼
Collect Capability Requirements
       │
       ▼
Resolve Effective Context
       │
       ├── Driver
       ├── Dialect
       ├── Platform
       ├── Connection
       ├── Runtime
       └── Configuration
       │
       ▼
Build/Resolve Capability Snapshot
       │
       ▼
Evaluate Requirements
       │
       ├── Native
       ├── Emulated
       ├── Partial
       └── Unsupported
       │
       ▼
Select Strategy
       │
       ▼
Execute / Reject
```

---

# 226. Extension boot flow

```text
Composer / VoltStack Packages
           │
           ▼
       Discovery
           │
           ▼
      Descriptors
           │
           ▼
 Dependency Resolution
           │
           ▼
 Compatibility Validation
           │
           ▼
  Extension Registration
           │
           ▼
 Specialized Registries
           │
           ▼
 Capability Providers
           │
           ▼
 Registry Compilation
           │
           ▼
         Freeze
           │
           ▼
        Runtime
```

---

# 227. Full model

```text
                    APPLICATION BOOT
                           │
                           ▼
                    Extension System
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           Drivers      Dialects     Platforms
              │            │            │
              └────────────┼────────────┘
                           ▼
                  Capability Providers
                           │
                           ▼
                  Capability Resolver
                           │
                           ▼
                 Effective Capabilities
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
           Query          Schema         ORM
             │             │             │
             ▼             ▼             ▼
         Optimizer      Migration    Persistence
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                      Execution
```

---

# 228. Extension invariants

## DB-EXT-001

Toda extensión deberá utilizar extension points documentados.

## DB-EXT-002

Una extensión no deberá depender de APIs `Internal`.

## DB-EXT-003

Extension registration deberá ser determinista.

## DB-EXT-004

Los registries deberán congelarse antes del runtime normal cuando sea posible.

## DB-EXT-005

No existirá un registry universal.

## DB-EXT-006

Los conflictos deberán resolverse explícitamente.

## DB-EXT-007

`last registered wins` no será una política implícita.

## DB-EXT-008

Las dependencias circulares entre extensiones están prohibidas.

## DB-EXT-009

Una extensión persistente no almacenará estado mutable de request.

## DB-EXT-010

Las extensiones deberán respetar el lifecycle definido por Database.

## DB-EXT-011

Los extension points deberán ser intencionales.

## DB-EXT-012

No toda clase interna será extensible.

## DB-EXT-013

Una extensión oficial deberá respetar boundaries equivalentes a una externa cuando sea posible.

## DB-EXT-014

La modificación dinámica del grafo de extensiones no será soportada por defecto.

## DB-EXT-015

Una extensión no podrá desactivar garantías de seguridad silenciosamente.

---

# 229. Capability invariants

## DB-CAP-001

Los subsistemas superiores deberán depender de capabilities, no de nombres de vendor.

## DB-CAP-002

Una capability deberá tener un identificador estable.

## DB-CAP-003

Capability support podrá ser contextual.

## DB-CAP-004

`NATIVE`, `EMULATED`, `PARTIAL`, `DISABLED` y `UNSUPPORTED` son estados semánticamente distintos.

## DB-CAP-005

Una emulación deberá preservar la semántica declarada.

## DB-CAP-006

Una capability declarada es una promesa verificable.

## DB-CAP-007

Las restricciones contextuales podrán reducir capabilities.

## DB-CAP-008

Una capa inferior no podrá inventar soporte sin provider válido.

## DB-CAP-009

Los fallbacks deberán ser explícitos.

## DB-CAP-010

Una operación unsupported deberá fallar de manera controlada.

## DB-CAP-011

Capability resolution deberá ser determinista.

## DB-CAP-012

Capability provenance deberá poder diagnosticarse.

## DB-CAP-013

Capabilities técnicas no sustituyen autorización.

## DB-CAP-014

Capabilities técnicas no sustituyen validación.

## DB-CAP-015

La compilación cacheada deberá considerar capabilities relevantes.

## DB-CAP-016

El AST portable deberá representar intención, no vendor.

## DB-CAP-017

Las diferencias de vendor deberán concentrarse en las capas autorizadas.

## DB-CAP-018

Capability discovery no deberá abrir conexiones prematuramente sin necesidad.

## DB-CAP-019

Las capabilities dependientes del servidor deberán invalidarse correctamente cuando cambie el entorno.

## DB-CAP-020

El modelo deberá permitir capabilities introducidas por paquetes externos sin modificar el enum/núcleo.

---

# 230. Patrones prohibidos

### Vendor condition disperso

```php
if ($driver === 'pgsql') {}
```

en ORM/Query Builder.

**Prohibido.**

### Registry universal

```php
$databaseRegistry->getAnything();
```

**Prohibido.**

### Runtime plugin mutation

```php
$registry->register($extension);
```

desde un request ordinario.

**Prohibido por defecto.**

### Silent override

```text
Extension B replaces A because it loaded later.
```

**Prohibido.**

### Capability lie

```text
supports = true
```

sin implementación semánticamente válida.

**Prohibido.**

### Unsafe fallback

```text
unsupported feature
→ approximate behavior
```

sin consentimiento/contrato.

**Prohibido.**

---

# 231. Ejemplo completo — RETURNING

La aplicación solicita:

```php
DB::table('users')
    ->insert($data)
    ->returning(['id']);
```

Internamente:

```text
Query Builder
     │
     ▼
Insert Query Model
     │
     ▼
AST
     │
     ▼
Semantic Analysis
     │
     ▼
Planner
     │
     ▼
requires:
database.query.returning
     │
     ▼
Capability Resolver
     │
     ├── NATIVE
     │      │
     │      ▼
     │  native strategy
     │
     ├── safe alternative
     │      │
     │      ▼
     │  fallback strategy
     │
     └── UNSUPPORTED
            │
            ▼
CapabilityNotSupportedException
```

En ningún punto Query Builder necesita preguntar:

```text
PostgreSQL?
MySQL?
SQLite?
```

---

# 232. Ejemplo completo — Savepoints

```php
DB::transaction(function () {
    DB::transaction(function () {
        // nested work
    });
});
```

Transaction Manager consulta:

```text
database.transaction.savepoints
```

Resultado:

```text
NATIVE
   │
   ▼
SAVEPOINT

UNSUPPORTED
   │
   ▼
configured nested transaction policy
```

No:

```php
if ($connection->driverName() === 'mysql') {}
```

---

# 233. Ejemplo completo — custom vector extension

Un paquete:

```text
vendor/voltstack-vector-database
```

podría registrar:

```text
VectorType
VectorExpression
VectorDistanceAstNode
VectorSemanticRule
VectorCapabilityRequirement
VectorCompilerHandler
```

y providers específicos:

```text
PostgreSQL + pgvector
MySQL vector support
custom engine
```

La aplicación utilizaría una semántica común cuando sea posible.

---

# 234. Ejemplo completo — capability narrowing

Platform:

```text
WRITE = supported
```

Connection:

```text
role = replica
```

Resultado:

```text
Effective Capability Set

WRITE = DISABLED
reason = read-only replica connection
```

El Query Executor puede rechazar un write antes de enviarlo.

---

# 235. Ejemplo completo — persistent runtime

```text
Worker
  │
  ├── Frozen Extension Registry
  ├── Frozen Type Registry
  ├── Platform Definitions
  ├── Capability Providers
  │
  ├── Request A
  │      └── Capability Snapshot A
  │
  ├── Request B
  │      └── Capability Snapshot B
  │
  └── Request C
         └── Capability Snapshot C
```

Los registries pueden compartirse.

El contexto mutable no.

---

# 236. Architectural review checklist

Toda nueva extensión deberá responder:

```text
What extension point is being used?
Why is an extension necessary?
What contracts does it implement?
What dependencies does it require?
What capabilities does it provide?
What capabilities does it require?
What state does it own?
What is its lifetime?
Is it concurrency-safe?
Can it affect security?
Can it affect SQL generation?
Can it affect connection state?
How is it tested?
How is compatibility versioned?
How is failure handled?
```

---

# 237. Capability review checklist

Toda nueva capability deberá responder:

```text
What semantic feature does it represent?
Is it boolean or valued?
Who provides it?
Can it be emulated?
What does emulation guarantee?
Can support vary by version?
Can support vary by connection?
Can configuration disable it?
What are its constraints?
How is it tested?
How is it exposed diagnostically?
Does it affect cache fingerprints?
```

---

# 238. Resultado arquitectónico

El modelo completo permite evolucionar:

```text
VoltStack Database
       │
       ├── stable core
       │
       ├── official drivers
       │
       ├── external drivers
       │
       ├── custom dialects
       │
       ├── custom types
       │
       ├── query extensions
       │
       ├── ORM extensions
       │
       ├── runtime adapters
       │
       └── framework integrations
```

sin que el Core necesite conocer cada implementación.

---

# 239. Principio final de extensibilidad

La regla será:

> Extender mediante contratos; nunca mediante conocimiento accidental de internals.

Formalmente:

```text
Extension
    │
    ▼
Stable Extension Contract
    │
    ▼
Database Extension Point
```

Nunca:

```text
Extension
    │
    ▼
Internal Implementation
```

---

# 240. Principio final de capabilities

La regla será:

> Preguntar por capacidades, no por nombres.

Formalmente:

```text
Feature
   │
   ▼
Capability Requirement
   │
   ▼
Effective Capability Set
   │
   ├── Native
   ├── Emulated
   ├── Partial
   ├── Disabled
   └── Unsupported
```

Nunca:

```text
Feature
   │
   ▼
if PostgreSQL
else if MySQL
else if SQLite
```

---

# 241. Ecuación arquitectónica

```text
Stable Core
    +
Explicit Extension Points
    +
Frozen Registries
    +
Capability-driven Specialization
    +
Deterministic Resolution
    +
Conformance Testing
    =
Extensible Database Architecture
```

---

# 242. Conclusión

`VoltStack/Quantum/Database` utilizará dos mecanismos complementarios.

El primero:

```text
Extension System
```

permitirá incorporar nuevas implementaciones y comportamientos sin modificar constantemente el Core.

El segundo:

```text
Capability System
```

permitirá que Query, Schema, Migration, ORM, Transaction y otros subsistemas adapten su comportamiento a las posibilidades reales de la plataforma sin conocer directamente el motor utilizado.

La separación será:

```text
               Database Feature
                      │
                      ▼
             Capability Requirement
                      │
                      ▼
              Capability Resolver
                      │
                      ▼
             Effective Capability
                      │
          ┌───────────┼────────────┐
          ▼           ▼            ▼
       Native      Emulated    Unsupported
          │           │            │
          ▼           ▼            ▼
       Strategy    Strategy       Error
```

mientras que la extensibilidad seguirá:

```text
External Package
      │
      ▼
Database Extension
      │
      ▼
Extension Contract
      │
      ▼
Specific Extension Point
      │
      ▼
Specialized Registry
      │
      ▼
Compiled/Frozen Runtime
```

Con esto, Database podrá soportar nuevos:

```text
drivers
database engines
SQL dialects
types
query constructs
optimizations
ORM features
runtime adapters
framework integrations
```

sin degradar progresivamente la arquitectura interna.

---

# 243. Cierre del bloque fundamental

Con este documento queda completado el bloque:

```text
00_DATABASE_PROJECT_CONTEXT.md
01_DATABASE_ARCHITECTURE.md
02_DATABASE_DESIGN_PRINCIPLES.md
03_DATABASE_DOMAIN_MODEL_AND_TERMINOLOGY.md
04_DATABASE_COMPONENT_AND_MODULE_ARCHITECTURE.md
05_DATABASE_CONTRACTS_AND_ABSTRACTIONS.md
06_DATABASE_CONFIGURATION_SYSTEM.md
07_DATABASE_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION.md
08_DATABASE_LIFECYCLE_AND_RUNTIME_MODEL.md
09_DATABASE_EXTENSION_AND_CAPABILITY_MODEL.md
```

Estos documentos establecen las reglas que deberán respetar todos los subsistemas posteriores.

---

# 244. Siguiente bloque

A partir del siguiente documento comienza la arquitectura de infraestructura Database:

```text
Driver
Connection
Dialect
Platform
Capabilities
```

La separación fundamental será:

```text
Connection
    ≠
Driver
    ≠
Dialect
    ≠
Platform
```

---

# 245. Siguiente documento

```text
10_DATABASE_DRIVER_ARCHITECTURE.md
```

Este documento deberá definir formalmente:

```text
Driver
Driver Contract
Driver Descriptor
Driver Registry
Driver Factory
Driver Resolution
Driver Configuration
Driver Capabilities

native connection creation
native statement preparation
native execution primitives
native transaction primitives
native result access
native error translation

Driver vs Connection
Driver vs Dialect
Driver vs Platform

driver lifecycle
driver state
persistent worker safety
driver extensions
driver conformance testing
```

y deberá mantener una regla esencial:

```text
Driver
   │
   ▼
native database communication primitives
```

pero nunca:

```text
Driver
   │
   ├── Query Builder
   ├── ORM
   ├── EntityManager
   ├── UnitOfWork
   └── application semantics
```