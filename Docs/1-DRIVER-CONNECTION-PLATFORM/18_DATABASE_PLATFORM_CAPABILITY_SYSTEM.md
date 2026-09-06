# 18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md

# VoltStack Quantum Database
## Database Platform Capability System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 18 — Database Platform Capability System  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure / Platform / Capability Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del:

```text
VoltStack/Quantum/Database
Platform Capability System
```

El sistema será responsable de representar de forma:

- estructurada;
- tipada;
- versionada;
- contextual;
- extensible;
- consultable;
- explicable;

las capacidades reales de una plataforma de base de datos.

Su objetivo principal será eliminar lógica como:

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

de las capas superiores del framework.

En su lugar:

```php
if ($capabilities->supports(DatabaseCapability::Returning)) {
    // ...
}
```

o:

```php
$capabilities->require(
    DatabaseCapability::SkipLocked
);
```

---

# 2. Regla fundamental

VoltStack preguntará:

```text
"What can this database do?"
```

en lugar de:

```text
"What database is this?"
```

Por tanto:

```text
Capability-driven architecture
>
Vendor-driven architecture
```

---

# 3. Separación fundamental

Debe mantenerse:

```text
Driver
≠
Dialect
≠
Platform
≠
Capability
≠
Feature
≠
Policy
```

Cada concepto responde una pregunta distinta.

---

# 4. Driver

El Driver responde:

```text
¿Cómo se comunica VoltStack
con el cliente nativo?
```

Ejemplos:

```text
PDO MySQL
PDO PostgreSQL
PDO SQLite
```

---

# 5. Dialect

El Dialect responde:

```text
¿Cómo se expresa una operación
mediante SQL?
```

Ejemplo:

```text
identifier quoting
RETURNING syntax
UPSERT syntax
locking syntax
```

---

# 6. Platform

Platform responde:

```text
¿Qué motor de base de datos
está siendo utilizado y cuáles
son sus reglas semánticas?
```

Ejemplos:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 7. Capability

Capability responde:

```text
¿Qué puede hacer efectivamente
esta plataforma en este contexto?
```

Ejemplos:

```text
supportsReturning
supportsSavepoints
supportsRecursiveCTE
supportsSkipLocked
```

---

# 8. Feature

Una Feature representa una funcionalidad de VoltStack.

Ejemplo:

```text
Query Upsert
```

puede requerir varias capacidades.

Por tanto:

```text
Feature
≠
Capability
```

---

# 9. Policy

Policy responde:

```text
Aunque técnicamente pueda hacerse,
¿VoltStack permite utilizarlo
bajo esta configuración?
```

Por tanto:

```text
can
≠
may
```

---

# 10. Modelo conceptual

```text
Database Engine
      │
      ▼
   Platform
      │
      ├── Server Version
      ├── Configuration
      ├── Driver Capabilities
      ├── Dialect Capabilities
      ├── Runtime Capabilities
      └── Extension Capabilities
              │
              ▼
      Capability Resolution
              │
              ▼
     EffectiveCapabilitySet
              │
      ┌───────┼─────────┐
      ▼       ▼         ▼
   Planner Compiler   Schema
      │       │         │
      ├───────┼─────────┤
      ▼       ▼         ▼
     ORM   Migration Transaction
```

---

# 11. Objetivos

El sistema deberá permitir:

1. describir capacidades de una plataforma;
2. distinguir soporte nativo y emulado;
3. modelar soporte parcial;
4. considerar versiones;
5. considerar configuración;
6. considerar rol de conexión;
7. considerar runtime;
8. considerar extensiones;
9. calcular capacidades efectivas;
10. validar requisitos;
11. explicar por qué existe o no una capacidad;
12. seleccionar estrategias;
13. mantener portabilidad;
14. evitar vendor checks;
15. soportar plataformas futuras.

---

# 12. No objetivos

Capability System no deberá:

- abrir conexiones;
- ejecutar SQL;
- compilar queries completas;
- administrar pools;
- hidratar entidades;
- administrar UnitOfWork;
- administrar IdentityMap;
- realizar routing;
- manejar tenants;
- implementar features de aplicación.

---

# 13. Arquitectura general

```text
DriverCapabilityProvider
          │
PlatformCapabilityProvider
          │
DialectCapabilityProvider
          │
RuntimeCapabilityProvider
          │
ExtensionCapabilityProvider
          │
ConfigurationCapabilityProvider
          │
          ▼
 CapabilityResolver
          │
          ▼
 EffectiveCapabilitySet
          │
          ▼
 CapabilityRequirementEvaluator
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
 Native Emulated Unsupported
```

---

# 14. CapabilityId

Toda capacidad tendrá un identificador estable.

Conceptualmente:

```php
final readonly class CapabilityId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
query.returning
query.upsert
query.cte
query.recursive_cte
query.window_functions
query.skip_locked

transaction.savepoints
transaction.nested
transaction.isolation.serializable

schema.drop_column
schema.partial_index
schema.generated_column
```

---

# 15. IDs namespaced

Los IDs deberán ser namespaced.

Ejemplo:

```text
query.returning.rows
schema.index.partial
transaction.savepoint
json.path
runtime.concurrent_queries
```

Esto permite extensiones futuras sin colisiones.

---

# 16. CapabilityId no será vendor-specific

Evitar:

```text
mysql.returning
postgres.returning
```

para capacidades semánticas generales.

Preferir:

```text
query.returning
```

---

# 17. Vendor-specific capabilities

Sólo cuando una capacidad sea genuinamente exclusiva:

```text
postgresql.listen_notify
postgresql.advisory_lock
mysql.get_lock
```

podrán existir namespaces vendor-specific.

---

# 18. Capability status

Cada capacidad podrá tener un estado.

```text
NATIVE
EMULATED
PARTIAL
DISABLED
UNSUPPORTED
UNKNOWN
```

---

# 19. NATIVE

Significa:

```text
la plataforma proporciona directamente
la capacidad requerida
```

---

# 20. EMULATED

Significa:

```text
VoltStack puede reproducir la semántica
mediante otra estrategia
```

---

# 21. PARTIAL

Significa:

```text
la capacidad existe,
pero sólo bajo ciertas condiciones
```

---

# 22. DISABLED

Significa:

```text
la plataforma podría soportarla,
pero la configuración/policy actual
la deshabilita
```

---

# 23. UNSUPPORTED

Significa:

```text
la plataforma no proporciona
la capacidad y no existe
una emulación válida
```

---

# 24. UNKNOWN

Significa:

```text
todavía no existe información suficiente
para determinar el soporte
```

Esto será útil antes del handshake con el servidor.

---

# 25. CapabilitySupport

Modelo conceptual:

```php
final readonly class CapabilitySupport
{
    public function __construct(
        public CapabilityId $capability,
        public CapabilityStatus $status,
        public ?CapabilityValue $value = null,
        public ?CapabilityProvenance $provenance = null,
    ) {}
}
```

---

# 26. Capacidades no sólo booleanas

Una capacidad no siempre será:

```text
true / false
```

También podrá contener valores.

---

# 27. Boolean capabilities

Ejemplo:

```text
supportsSavepoints = true
```

---

# 28. Numeric capabilities

Ejemplo:

```text
maxIdentifierLength = 63
```

---

# 29. Enum capabilities

Ejemplo:

```text
returningSupport =
NONE
GENERATED_KEYS_ONLY
SINGLE_ROW
MULTI_ROW
FULL
```

---

# 30. Set capabilities

Ejemplo:

```text
supportedIsolationLevels = {
    READ_COMMITTED,
    REPEATABLE_READ,
    SERIALIZABLE
}
```

---

# 31. Structured capabilities

Ejemplo:

```text
JsonCapability
{
    nativeType: true
    pathQueries: true
    mutation: true
    indexing: true
}
```

---

# 32. CapabilityValue

Podrá existir una jerarquía:

```text
CapabilityValue
├── BooleanCapabilityValue
├── IntegerCapabilityValue
├── EnumCapabilityValue
├── SetCapabilityValue
└── StructuredCapabilityValue
```

---

# 33. Evitar stringly typed values

Evitar:

```php
$capabilities->get('returning') === 'full';
```

Preferir tipos específicos.

---

# 34. CapabilityDescriptor

Cada capability podrá tener metadata.

```php
final readonly class CapabilityDescriptor
{
    public function __construct(
        public CapabilityId $id,
        public string $description,
        public CapabilityValueType $valueType,
        public CapabilityStability $stability,
    ) {}
}
```

---

# 35. CapabilityRegistry

Existirá:

```text
CapabilityRegistry
```

responsable de conocer capacidades registradas.

---

# 36. CapabilityRegistry no contiene valores efectivos

Debe distinguirse:

```text
CapabilityRegistry
=
qué capabilities existen

EffectiveCapabilitySet
=
qué valores tienen para un target concreto
```

---

# 37. Registry lifecycle

```text
Bootstrap
   │
   ▼
Core capabilities
   │
   ▼
Official platform capabilities
   │
   ▼
Extension capabilities
   │
   ▼
Validate
   │
   ▼
Freeze
```

---

# 38. CapabilityProvider

Las fuentes de capacidades implementarán contratos especializados.

Ejemplo:

```php
interface CapabilityProviderInterface
{
    public function provide(
        CapabilityResolutionContext $context
    ): CapabilityContribution;
}
```

---

# 39. Capability sources

Fuentes principales:

```text
Driver
Platform
Dialect
Server Version
Server Configuration
Connection
Runtime
Extensions
VoltStack Configuration
```

---

# 40. Capability layering

Modelo:

```text
Base Platform Capabilities
          │
          ▼
Version Refinement
          │
          ▼
Driver Refinement
          │
          ▼
Runtime Refinement
          │
          ▼
Connection Refinement
          │
          ▼
Configuration Policy
          │
          ▼
Extension Contributions
          │
          ▼
Effective Capability Set
```

---

# 41. Capability narrowing

Una capa inferior/contextual podrá restringir una capacidad.

Ejemplo:

```text
Platform supports writes
+
Connection role = READ_ONLY
=
effective writes unsupported
```

---

# 42. Capability enhancement

Una extensión podrá proporcionar una capacidad adicional sólo cuando implemente realmente la funcionalidad necesaria.

---

# 43. No false capabilities

Una extensión no podrá simplemente declarar:

```text
supportsFeature = true
```

sin proporcionar implementación cuando ésta sea necesaria.

---

# 44. PlatformCapabilityProvider

Cada plataforma oficial tendrá un provider.

```text
MySqlPlatformCapabilityProvider
MariaDbPlatformCapabilityProvider
PostgreSqlPlatformCapabilityProvider
SQLitePlatformCapabilityProvider
```

---

# 45. Platform baseline

Ejemplo conceptual:

```text
PostgreSQL Platform
    │
    ├── supportsSavepoints
    ├── supportsReturning
    ├── supportsRecursiveCTE
    ├── supportsWindowFunctions
    └── ...
```

---

# 46. Version refinement

Las capacidades podrán depender de:

```text
ServerVersion
```

---

# 47. VersionRange

Ejemplo conceptual:

```php
CapabilityRule(
    capability: DatabaseCapability::SomeFeature,
    versions: VersionRange::atLeast('X.Y'),
    status: CapabilityStatus::NATIVE,
);
```

---

# 48. No scattered version checks

Evitar:

```php
if ($serverVersion >= 'x.y') {
}
```

distribuido por Compiler, ORM, Schema y Migration.

---

# 49. Version rules centralizadas

Las reglas deberán residir en:

```text
Platform Capability Definitions
```

o componentes especializados.

---

# 50. ServerVersion

Será un value object.

No usar comparación lexicográfica de strings.

---

# 51. ServerVariant

Puede ser necesario distinguir:

```text
vendor
distribution
compatibility mode
```

---

# 52. MySQL vs MariaDB

Aunque compartan transportes:

```text
MySqlPlatform
≠
MariaDbPlatform
```

y sus capability sets podrán divergir.

---

# 53. PostgreSQL-compatible databases

Un servidor que hable protocolo PostgreSQL no implica automáticamente:

```text
all PostgreSQL capabilities
```

---

# 54. Compatibility modes

Podrán existir:

```text
PlatformCompatibilityProfile
```

para motores compatibles parcialmente.

---

# 55. Driver capabilities

El Driver describe capacidades de transporte.

Ejemplos:

```text
driver.prepared_statements
driver.native_prepared_statements
driver.streaming_results
driver.query_cancellation
driver.multiple_result_sets
driver.async
driver.connection_reset
```

---

# 56. Driver capability vs Platform capability

Ejemplo:

```text
Platform:
supportsStreamingResultSemantics

Driver:
canExposeStreamingResultAPI
```

La capacidad efectiva requiere ambas.

---

# 57. Capability intersection

Ejemplo:

```text
Effective Streaming
=
Platform Streaming Support
∩
Driver Streaming Support
```

---

# 58. Dialect capabilities

El Dialect podrá describir disponibilidad de una representación sintáctica.

Ejemplo:

```text
dialect.returning_syntax
dialect.upsert_syntax
```

---

# 59. Dialect syntax does not guarantee feature

```text
syntax available
≠
semantic feature available
```

---

# 60. Runtime capabilities

Runtime podrá contribuir:

```text
runtime.concurrent_execution
runtime.safe_connection_reuse
runtime.request_scope
runtime.worker_persistence
```

---

# 61. Persistent runtime capability

Ejemplo:

```text
FrankenPHP
+
Connection reset implementation
=
safe worker connection reuse
```

---

# 62. Connection capabilities

Una Connection puede restringir:

```text
writes
transactions
DDL
locking
```

según:

```text
role
endpoint
policy
replica status
```

---

# 63. Connection role example

```text
PostgreSQL Platform:
WRITE = supported

Replica Connection:
WRITE = disabled
```

---

# 64. Configuration capabilities

La configuración podrá desactivar una feature.

Ejemplo:

```text
technical capability:
native prepared statements = supported

policy:
native prepares = disabled

effective:
DISABLED
```

---

# 65. Technical support vs policy

Se mantendrán ambas vistas.

```text
TechnicalCapabilitySet
PolicyCapabilitySet
EffectiveCapabilitySet
```

o una representación equivalente.

---

# 66. Can vs may

API conceptual:

```php
$capabilities->can(DatabaseCapability::Returning);

$capabilities->may(DatabaseCapability::Returning);
```

si se decide exponer ambas dimensiones.

---

# 67. EffectiveCapabilitySet

Será la representación principal consumida por hot paths.

---

# 68. Inmutabilidad

`EffectiveCapabilitySet` será:

```text
immutable
```

---

# 69. Thread/fiber safety

Podrá compartirse cuando su contexto sea compatible.

---

# 70. No mutable current capability

Prohibido:

```php
$platform->setCurrentCapabilities(...);
```

---

# 71. CapabilityResolutionContext

Podrá contener:

```php
final readonly class CapabilityResolutionContext
{
    public function __construct(
        public PlatformDescriptor $platform,
        public ?ServerVersion $serverVersion,
        public DriverDescriptor $driver,
        public ?ConnectionRole $connectionRole,
        public RuntimeDescriptor $runtime,
        public DatabasePolicy $policy,
    ) {}
}
```

---

# 72. Context extensibility

El contexto podrá enriquecerse sin convertirlo en un array arbitrario.

---

# 73. Preliminary capability set

Antes de conectar:

```text
Configured Platform
+
Target Version
+
Driver
+
Config
=
PreliminaryCapabilitySet
```

---

# 74. Runtime capability set

Después del handshake:

```text
PreliminaryCapabilitySet
+
Actual Server Version
+
Server Configuration
=
ResolvedCapabilitySet
```

---

# 75. Effective capability set

Finalmente:

```text
ResolvedCapabilitySet
+
Connection Context
+
Runtime Context
+
Policy
+
Extensions
=
EffectiveCapabilitySet
```

---

# 76. Capability discovery

No toda capability requerirá una query de descubrimiento.

---

# 77. Static capabilities

Ejemplo:

```text
identifier quote style
```

puede conocerse sin conexión.

---

# 78. Version capabilities

Algunas se conocen con:

```text
server version
```

---

# 79. Runtime-discovered capabilities

Otras pueden requerir:

```text
server variables
extensions installed
configuration
permissions
```

---

# 80. Lazy discovery

VoltStack no abrirá conexiones durante bootstrap sólo para descubrir capabilities.

---

# 81. No eager database dependency

Esto permitirá:

```text
config cache
container compilation
migration SQL generation
static analysis
testing
```

sin servidor disponible.

---

# 82. Capability certainty

Podrá modelarse:

```text
DECLARED
INFERRED
DISCOVERED
VERIFIED
```

---

# 83. CapabilityConfidence

Opcionalmente:

```text
CapabilityConfidence
```

podrá indicar el nivel de certeza.

---

# 84. Capability provenance

Cada capability deberá poder explicar de dónde provino.

---

# 85. CapabilityProvenance

Ejemplo:

```text
Capability: query.returning
Status: NATIVE

Source:
PostgreSqlPlatformCapabilityProvider

Reason:
supported by target platform/version
```

---

# 86. Multiple provenance sources

Una capacidad efectiva puede depender de varias fuentes.

Ejemplo:

```text
streaming result
├── Platform
├── Driver
└── Connection policy
```

---

# 87. CapabilityResolutionTrace

En modo diagnóstico podrá construirse:

```text
CapabilityResolutionTrace
```

---

# 88. Example trace

```text
query.returning

Platform PostgreSQL:
    NATIVE

Server version:
    compatible

Dialect:
    syntax available

Connection role:
    allowed

Policy:
    enabled

Effective:
    NATIVE
```

---

# 89. Negative trace

```text
transaction.write

Platform:
    supported

Connection:
    role = replica

Policy:
    read-only

Effective:
    DISABLED
```

---

# 90. Trace cost

Los traces completos no deberán construirse obligatoriamente en cada query.

---

# 91. Production optimization

Producción podrá usar:

```text
precomputed capability set
+
fingerprint
```

sin mantener explicación detallada.

---

# 92. Debug mode

Desarrollo podrá conservar provenance detallado.

---

# 93. CapabilityRequirement

Las features podrán declarar requisitos.

```php
final readonly class CapabilityRequirement
{
    public function __construct(
        public CapabilityId $capability,
        public CapabilityConstraint $constraint,
    ) {}
}
```

---

# 94. Boolean requirement

Ejemplo:

```text
requires query.returning
```

---

# 95. Value requirement

Ejemplo:

```text
maxIdentifierLength >= 64
```

---

# 96. Set requirement

Ejemplo:

```text
transaction isolation contains SERIALIZABLE
```

---

# 97. Composite requirements

Se soportarán:

```text
ALL_OF
ANY_OF
NOT
```

---

# 98. Example composite

```text
Feature:
Native Upsert

Requires:

ALL_OF(
    query.upsert,
    query.conflict_target
)
```

---

# 99. CapabilityRequirementSet

Una feature podrá declarar:

```text
CapabilityRequirementSet
```

---

# 100. Requirement evaluator

Componente:

```text
CapabilityRequirementEvaluator
```

resolverá:

```text
SATISFIED
SATISFIED_WITH_EMULATION
PARTIALLY_SATISFIED
UNSATISFIED
UNKNOWN
```

---

# 101. Capability requirement result

Podrá contener:

```text
status
missing capabilities
emulated capabilities
warnings
selected strategy hints
```

---

# 102. FeatureDescriptor

Las funcionalidades del framework podrán describirse:

```php
final readonly class FeatureDescriptor
{
    public function __construct(
        public FeatureId $id,
        public CapabilityRequirementSet $requirements,
    ) {}
}
```

---

# 103. Feature examples

```text
query.upsert
query.returning
schema.zero_downtime_add_column
transaction.nested
orm.batch_insert_returning
```

---

# 104. Feature resolution

```text
Feature
   │
   ▼
Capability Requirements
   │
   ▼
EffectiveCapabilitySet
   │
   ▼
FeatureStrategyResolver
   │
   ├── Native
   ├── Emulated
   └── Unsupported
```

---

# 105. Native strategy

Preferida cuando:

```text
semantics correct
+
capability native
+
policy permits
```

---

# 106. Emulation strategy

Se permitirá únicamente cuando preserve la semántica requerida.

---

# 107. Emulation is not fallback SQL guessing

No:

```text
try syntax A
catch
try syntax B
```

---

# 108. Emulation contract

Toda emulación deberá documentar:

```text
semantic equivalence
atomicity
transaction requirements
concurrency behavior
performance cost
failure modes
```

---

# 109. Unsafe emulation

Si no puede garantizarse semántica correcta:

```text
UNSUPPORTED
```

será preferible a una emulación incorrecta.

---

# 110. Example emulation

Supongamos:

```text
NULLS LAST
```

no disponible sintácticamente.

Planner podría transformar:

```text
ORDER BY expression NULLS LAST
```

a una estrategia equivalente cuando sea segura.

---

# 111. Emulation belongs to Planner

El Capability System informa:

```text
native / emulatable / unsupported
```

pero:

```text
Planner
```

selecciona la estrategia concreta.

---

# 112. Dialect expresses selected strategy

Después:

```text
Dialect
```

produce la sintaxis correspondiente.

---

# 113. Capability strategy model

Podrá existir:

```text
CapabilityStrategy
├── NativeCapabilityStrategy
├── EmulatedCapabilityStrategy
└── UnsupportedCapabilityStrategy
```

---

# 114. Query Engine integration

Pipeline:

```text
Query AST
   │
   ▼
Semantic Analysis
   │
   ▼
Capability Requirements
   │
   ▼
Optimizer
   │
   ▼
Planner
   │
   ├── EffectiveCapabilitySet
   ▼
Execution Plan
```

---

# 115. Query Builder independence

Query Builder no deberá preguntar:

```php
if ($connection->supportsReturning()) {
}
```

mientras construye el AST portable.

---

# 116. Late capability resolution

Preferencia:

```text
express intent first
resolve capability later
```

---

# 117. Why late resolution

Permite que una misma Query Model pueda dirigirse a diferentes plataformas.

---

# 118. Semantic validation

Algunas incompatibilidades pueden detectarse en:

```text
Semantic Analysis
```

cuando el target ya es conocido.

---

# 119. Planner validation

Las decisiones estratégicas pertenecen principalmente al Planner.

---

# 120. Compiler defensive validation

Compiler deberá verificar que la estrategia seleccionada sea compatible con:

```text
Dialect
+
EffectiveCapabilitySet
```

---

# 121. ORM integration

ORM utilizará capabilities para decisiones como:

```text
generated identifiers
batch insert
RETURNING
locking
savepoints
transaction behavior
```

---

# 122. ORM does not check vendor

Prohibido:

```php
if ($platform->name() === 'postgresql') {
    $unitOfWork->...
}
```

---

# 123. Persistence planner

Deberá preguntar:

```text
can generated values be returned?
can multiple rows return generated values?
can inserts be batched?
```

---

# 124. Identity generation

Ejemplo:

```text
Generated Identity
      │
      ▼
Persistence Planner
      │
      ├── RETURNING
      ├── Generated Key API
      ├── Sequence
      └── Other Strategy
```

según capabilities.

---

# 125. Relationship persistence

Las estrategias también podrán considerar:

```text
deferrable constraints
transaction support
batch operations
```

---

# 126. Schema integration

Schema Planner consumirá:

```text
schema capabilities
```

---

# 127. Schema capability examples

```text
schema.create_table
schema.alter_table
schema.rename_table
schema.rename_column
schema.drop_column
schema.generated_column
schema.check_constraint
schema.foreign_key
schema.deferrable_constraint
schema.partial_index
schema.expression_index
schema.concurrent_index
```

---

# 128. Schema model independence

`Schema Model` no deberá convertirse en un modelo MySQL/PostgreSQL.

---

# 129. Schema Planner

Decidirá:

```text
native operation
table rebuild
multi-step migration
unsupported operation
```

según capabilities.

---

# 130. SQLite example

Una operación no disponible directamente podría requerir:

```text
create temporary table
copy data
drop old table
rename replacement
```

si la estrategia es segura.

---

# 131. Capability does not execute migration

Capability System sólo informa.

Migration Planner diseña el plan.

---

# 132. Migration integration

Migration Safety System podrá utilizar capabilities como:

```text
transactional DDL
online index creation
concurrent index creation
lock behavior
```

---

# 133. Transaction capabilities

Ejemplos:

```text
transaction.supported
transaction.savepoint
transaction.isolation
transaction.read_only
transaction.deferrable
transaction.transactional_ddl
transaction.lock_timeout
```

---

# 134. Nested transactions

VoltStack podrá modelar:

```text
Nested Transaction Feature
```

que normalmente requiere:

```text
savepoint capability
```

---

# 135. No fake nested transactions

Si no existen savepoints, no deberá declararse nested transaction real salvo que exista una semántica alternativa explícita.

---

# 136. Lock capabilities

Ejemplos:

```text
lock.for_update
lock.for_share
lock.nowait
lock.skip_locked
lock.advisory
```

---

# 137. Lock semantics

La sintaxis pertenece al Dialect.

La disponibilidad/semántica pertenece a Platform/Capabilities.

---

# 138. JSON capabilities

Podrán modelarse granularmente.

```text
json.native_type
json.extract
json.path
json.contains
json.exists
json.mutate
json.index
json.aggregate
```

---

# 139. Avoid giant JSON flag

Evitar:

```text
supportsJson = true
```

como única información.

---

# 140. Window capabilities

Ejemplos:

```text
query.window
query.window.frame.rows
query.window.frame.range
query.window.frame.groups
```

---

# 141. CTE capabilities

```text
query.cte
query.cte.recursive
query.cte.materialized_hint
query.cte.data_modifying
```

---

# 142. Returning capabilities

Granularidad sugerida:

```text
query.returning.insert
query.returning.update
query.returning.delete
query.returning.multi_row
```

---

# 143. Upsert capabilities

```text
query.upsert
query.upsert.conflict_columns
query.upsert.constraint_target
query.upsert.do_nothing
query.upsert.update
query.upsert.returning
```

---

# 144. Pagination capabilities

```text
query.limit
query.offset
query.keyset_pagination
```

---

# 145. Identifier capabilities

Ejemplos:

```text
identifier.max_length
identifier.case_behavior
identifier.catalog_support
identifier.schema_support
```

---

# 146. Parameter capabilities

```text
parameter.max_count
parameter.named
parameter.positional
parameter.numbered
```

cuando sean relevantes al target efectivo.

---

# 147. Prepared statement capability

Deberá considerar:

```text
Platform
+
Driver
+
Configuration
```

---

# 148. Bulk capabilities

```text
bulk.insert
bulk.update
bulk.delete
bulk.returning
bulk.parameter_limit
```

---

# 149. Streaming capabilities

```text
result.streaming
result.server_cursor
result.unbuffered
result.multiple_result_sets
```

---

# 150. Cancellation capabilities

```text
query.cancel
query.timeout.server
query.timeout.client
```

---

# 151. Connection capabilities

```text
connection.ping
connection.reset
connection.session_reset
connection.reauthentication
connection.read_only
```

---

# 152. Pool compatibility capabilities

Pooling puede requerir:

```text
connection.reset
+
session state tracking
+
transaction cleanup
```

---

# 153. Pool safety is composite

No deberá existir:

```text
pooling = supported
```

sólo porque el Driver puede reutilizar una conexión.

---

# 154. Pool feature requirement

Ejemplo conceptual:

```text
SafeConnectionReuse =
ALL_OF(
    connection.reset,
    transaction.rollback,
    result.cleanup
)
```

más garantías del lifecycle.

---

# 155. Runtime capabilities

Para FrankenPHP:

```text
runtime.persistent_worker
runtime.execution_scope
runtime.cleanup_hook
runtime.concurrent_execution
```

podrán influir en estrategias.

---

# 156. FrankenPHP integration

El Capability System no dependerá directamente de FrankenPHP.

El runtime adapter proporcionará:

```text
RuntimeCapabilityContribution
```

---

# 157. RoadRunner

Mismo contrato.

---

# 158. OpenSwoole

Mismo contrato.

---

# 159. Runtime-neutral core

```text
Database Capability Core
       ▲
       │
Runtime Capability Port
       ▲
       │
FrankenPHP / RoadRunner / OpenSwoole adapters
```

---

# 160. Extension capabilities

Una extensión podrá:

```text
introduce capability
provide capability
refine capability
restrict capability
```

según contrato.

---

# 161. Extension capability namespace

Ejemplo:

```text
extension.vector_search
```

o:

```text
postgresql.pgvector.search
```

---

# 162. Extension requirements

Una extensión podrá declarar:

```text
requires:
    json.native_type
    schema.expression_index
```

---

# 163. Extension activation

```text
Extension
   │
   ▼
Capability requirements
   │
   ▼
Evaluator
   │
   ├── activate
   └── reject
```

---

# 164. Capability conflicts

Dos providers no deberán sobrescribir capacidades arbitrariamente.

---

# 165. Resolution rules

Las contribuciones tendrán:

```text
source
priority
authority
scope
```

---

# 166. Authority model

Ejemplo conceptual:

```text
Platform baseline
<
Version refinement
<
Runtime discovery
<
Connection restriction
<
Security/policy restriction
```

Una restricción contextual puede reducir soporte.

---

# 167. Restrictions dominate

Si:

```text
Platform = NATIVE
Connection Policy = DISABLED
```

resultado:

```text
DISABLED
```

---

# 168. Unsupported cannot become native magically

Una configuración no puede convertir:

```text
UNSUPPORTED
```

en:

```text
NATIVE
```

---

# 169. Extension implementation exception

Una extensión sí puede aportar una implementación real.

Ejemplo:

```text
Platform:
feature unsupported

Extension:
provides semantically valid emulation

Effective:
EMULATED
```

---

# 170. CapabilityContribution

Modelo conceptual:

```php
final readonly class CapabilityContribution
{
    public function __construct(
        public CapabilityId $capability,
        public CapabilityStatus $status,
        public CapabilitySource $source,
        public ?CapabilityValue $value = null,
    ) {}
}
```

---

# 171. CapabilityResolver

Será responsable de combinar contribuciones.

---

# 172. Resolver determinista

Mismos inputs:

```text
same capability set
```

---

# 173. No registration-order semantics

Resultado no deberá depender accidentalmente del orden de carga de Composer.

---

# 174. Capability rule engine

Podrá existir:

```text
CapabilityRuleEngine
```

para aplicar:

```text
version rules
configuration rules
dependency rules
restrictions
```

---

# 175. Capability dependencies

Una capability puede depender de otra.

Ejemplo:

```text
query.upsert.returning
requires
query.upsert
AND
query.returning.insert
```

---

# 176. Dependency graph

Las dependencias deberán formar un grafo validable.

---

# 177. Cycle detection

Ciclos como:

```text
A requires B
B requires A
```

deberán detectarse.

---

# 178. Derived capabilities

Podrán existir capabilities derivadas.

Ejemplo:

```text
orm.batch_generated_ids
=
bulk.insert
+
generated_values.returning
```

---

# 179. Derived capability evaluation

No deberán duplicarse manualmente en cada Platform.

---

# 180. Capability expressions

Podrá utilizarse un modelo:

```text
CapabilityExpression
├── CapabilityReference
├── AllOf
├── AnyOf
├── Not
├── ValueComparison
└── SetContains
```

---

# 181. Example

```text
ALL_OF(
    transaction.savepoint,
    transaction.supported
)
```

---

# 182. Capability profiles

Podrán existir perfiles de alto nivel:

```text
BasicRelationalProfile
TransactionalProfile
AdvancedQueryProfile
OrmPersistenceProfile
MigrationProfile
```

---

# 183. Profiles are requirements

Un perfil no reemplaza las capabilities individuales.

Es simplemente:

```text
named requirement set
```

---

# 184. ORM minimum profile

VoltStack ORM podría requerir un conjunto mínimo.

---

# 185. Query Builder minimum profile

El Query Builder básico podrá funcionar con un conjunto menor.

---

# 186. Raw SQL minimum profile

Raw SQL podría requerir sólo:

```text
connection
statement execution
parameter binding
```

---

# 187. Graceful degradation

La ausencia de una feature avanzada no deberá inutilizar todo Database subsystem.

---

# 188. Capability boundaries

Ejemplo:

```text
No window functions
```

no implica:

```text
Database unsupported
```

---

# 189. Minimum platform capabilities

Cada plataforma oficial deberá cumplir:

```text
DatabasePlatformMinimumProfile
```

---

# 190. Conformance levels

Podrán definirse:

```text
CORE
STANDARD
ADVANCED
FULL
```

para diagnóstico/documentación, no para ocultar capacidades individuales.

---

# 191. Platform descriptor

Cada Platform tendrá:

```php
final readonly class PlatformDescriptor
{
    public function __construct(
        public PlatformId $id,
        public string $name,
        public PlatformFamily $family,
    ) {}
}
```

---

# 192. Platform ID

Ejemplos:

```text
mysql
mariadb
postgresql
sqlite
```

---

# 193. Platform family

Puede servir para compartir componentes, pero no para reemplazar capabilities.

---

# 194. DatabasePlatformInterface

Contrato conceptual:

```php
interface DatabasePlatformInterface
{
    public function descriptor(): PlatformDescriptor;

    public function capabilities(
        PlatformCapabilityContext $context
    ): PlatformCapabilitySet;
}
```

---

# 195. Keep Platform small

Evitar:

```php
interface DatabasePlatformInterface
{
    public function supportsEverything(): bool;
    public function compileEverything(): string;
    public function executeEverything(): mixed;
}
```

---

# 196. Platform composition

Preferencia:

```text
Platform
├── Capability Provider
├── Type Mapping
├── Schema Semantics
├── Transaction Semantics
├── Identifier Semantics
└── Server Metadata Interpreter
```

---

# 197. Platform ≠ Compiler

Platform no compila queries completas.

---

# 198. Platform ≠ Driver

Platform no abre sockets ni PDO.

---

# 199. Platform ≠ Dialect

Platform no será el propietario general de la sintaxis SQL.

---

# 200. Platform Resolver

Existirá:

```text
PlatformResolver
```

---

# 201. Platform resolution

Inputs posibles:

```text
configured platform hint
driver
server metadata
server version
compatibility mode
```

---

# 202. Preliminary platform

Antes de conexión:

```text
configured platform
```

podrá generar:

```text
PreliminaryPlatform
```

---

# 203. Resolved platform

Después del handshake:

```text
ResolvedPlatform
```

---

# 204. Platform mismatch

Si configuración dice:

```text
mysql
```

pero servidor se identifica incompatiblemente:

```text
PlatformResolutionException
```

---

# 205. MariaDB detection

Si el transporte es MySQL-compatible pero el servidor es MariaDB:

```text
Driver:
pdo.mysql

Platform:
mariadb

Dialect:
mariadb
```

será una combinación válida.

---

# 206. PlatformVersion

Podrá ser distinta del client version.

---

# 207. Capability snapshot

Para una operación se podrá utilizar:

```text
CapabilitySnapshot
```

---

# 208. Snapshot purpose

Garantiza que durante una operación:

```text
capability decisions remain stable
```

---

# 209. Snapshot immutability

No deberá cambiar a mitad de compilación.

---

# 210. CapabilityFingerprint

Cada set efectivo tendrá:

```text
CapabilityFingerprint
```

---

# 211. Fingerprint inputs

Podrá incluir:

```text
platform ID
platform version
driver capability profile
runtime capability profile
relevant server configuration
extensions
policy
```

---

# 212. No credentials

Nunca:

```text
password
secret
token
private key
```

---

# 213. Fingerprint purpose

Será útil para:

```text
compiled query cache
schema plan cache
metadata specialization
diagnostics
```

---

# 214. Query cache key

Conceptualmente:

```text
QueryFingerprint
+
DialectFingerprint
+
CapabilityFingerprint
+
CompilerVersion
```

---

# 215. Cache invalidation

Si cambia una capability relevante:

```text
old compiled plan
```

no deberá reutilizarse incorrectamente.

---

# 216. Capability generation

Podrá existir:

```text
CapabilityGeneration
```

cuando haya:

```text
server configuration reload
extension activation
policy update
runtime generation change
```

---

# 217. Stable request snapshot

Una request conservará su snapshot aunque una nueva generación sea publicada para futuras requests.

---

# 218. Persistent runtime safety

Modelo:

```text
Worker
│
├── CapabilityRegistry          immutable
├── Platform definitions       immutable
├── Capability rules           immutable
│
├── Request A
│   └── CapabilitySnapshot A
│
└── Request B
    └── CapabilitySnapshot B
```

---

# 219. No global current platform

Prohibido:

```php
Platform::setCurrent(...)
```

---

# 220. No global current capabilities

Prohibido:

```php
Capabilities::current()
```

si depende de estado mutable process-global.

---

# 221. Context propagation

El snapshot se pasará mediante:

```text
ExecutionContext
PlanningContext
CompilationContext
SchemaContext
PersistenceContext
```

según corresponda.

---

# 222. Scope-aware access

Una facade pública puede resolverlo desde el scope actual, pero las capas internas recibirán dependencias explícitas.

---

# 223. Concurrency safety

Request A y Request B podrán trabajar simultáneamente con:

```text
different platform
different tenant
different server version
different capabilities
```

sin contaminación.

---

# 224. Multi-database application

Ejemplo:

```text
primary
→ PostgreSQL

analytics
→ MySQL

local-cache
→ SQLite
```

cada una tendrá su:

```text
ResolvedPlatform
ResolvedDialect
EffectiveCapabilitySet
```

---

# 225. Multitenancy

Un tenant puede usar:

```text
Tenant A → PostgreSQL
Tenant B → MySQL
Tenant C → MariaDB
```

sin cambiar estado global.

---

# 226. Multitenancy optionality

Database core no dependerá del paquete Multitenancy.

---

# 227. Tenant integration

El paquete opcional podrá proporcionar:

```text
ConnectionDefinition
```

y contexto de resolución.

Capability System funcionará normalmente después.

---

# 228. Security capabilities

Algunas capacidades podrán describir seguridad técnica.

Ejemplos:

```text
connection.tls
connection.tls.verify_peer
connection.tls.verify_hostname
credential.rotation
```

---

# 229. Security policy

Aunque una plataforma soporte TLS opcionalmente:

```text
production policy
```

podrá exigirlo.

---

# 230. Required capability

Configuración:

```text
require:
    connection.tls.verify_peer
```

podrá provocar fail-fast.

---

# 231. Capability requirements in configuration

VoltStack podrá permitir:

```yaml
database:
  requirements:
    - transaction.savepoint
    - query.cte
```

conceptualmente.

---

# 232. Boot-time requirements

Si pueden validarse offline:

```text
fail during bootstrap
```

---

# 233. Runtime requirements

Si requieren servidor:

```text
validate on first connection
```

o mediante health/readiness check.

---

# 234. Strict mode

Podrá existir:

```text
capability_validation: strict
```

---

# 235. Compatibility mode

También:

```text
capability_validation: compatible
```

siempre sin aceptar pérdida silenciosa de semántica.

---

# 236. Unknown handling

Una capability `UNKNOWN` no deberá tratarse automáticamente como `SUPPORTED`.

---

# 237. Conservative default

En operaciones críticas:

```text
UNKNOWN
→ do not assume support
```

---

# 238. CapabilityNotSupportedException

Jerarquía sugerida:

```text
CapabilityException
├── CapabilityNotFoundException
├── CapabilityNotSupportedException
├── CapabilityDisabledException
├── CapabilityRequirementException
├── CapabilityResolutionException
├── CapabilityConflictException
├── CapabilityDependencyException
└── CapabilityDiscoveryException
```

---

# 239. Error information

Una excepción deberá poder incluir:

```text
requested feature
required capability
platform
version
driver
effective status
reason
possible alternative
```

---

# 240. Example

```text
Feature not available:

Feature:
query.lock.skip_locked

Platform:
SQLite

Required capability:
lock.skip_locked

Effective status:
UNSUPPORTED
```

---

# 241. Avoid vendor SQL errors

Si VoltStack ya conoce la incompatibilidad, no deberá esperar a que el servidor devuelva:

```text
syntax error
```

---

# 242. Fail as early as practical

Orden preferido:

```text
configuration validation
        │
semantic validation
        │
planning
        │
compilation
        │
execution
```

---

# 243. Capability diagnostics

CLI futura:

```text
php volt database:capabilities
```

podrá mostrar capabilities.

---

# 244. Connection-specific diagnostics

Ejemplo:

```text
php volt database:capabilities primary
```

---

# 245. Explain capability

Ejemplo:

```text
php volt database:capability query.returning --connection=primary
```

---

# 246. Example output

```text
Capability: query.returning.insert

Connection: primary
Platform: PostgreSQL
Version: <resolved-version>
Driver: pdo.pgsql

Technical support: NATIVE
Policy: ENABLED
Effective support: NATIVE
```

---

# 247. Explain provenance

Podrá mostrar:

```text
Platform baseline
Version refinement
Driver compatibility
Connection restrictions
Policy
Extensions
```

---

# 248. Redaction

Diagnostics nunca mostrarán:

```text
passwords
tokens
secret DSNs
private keys
```

---

# 249. Debug toolbar integration

Telemetry/Debug Toolbar podrá mostrar:

```text
Platform
Dialect
Capability profile
Emulated features
```

---

# 250. Emulation warnings

En desarrollo podrá advertirse:

```text
Query uses emulated capability:
query.null_ordering
```

---

# 251. Performance warning

Algunas emulaciones podrán marcar:

```text
performance_cost = HIGH
```

---

# 252. Capability cost metadata

Opcionalmente:

```text
LOW
MEDIUM
HIGH
UNKNOWN
```

---

# 253. Semantic risk metadata

También:

```text
EXACT
CONTEXT_DEPENDENT
LOSSY
```

---

# 254. Lossy emulation

No deberá activarse silenciosamente.

---

# 255. Exactness rule

La API portable sólo deberá seleccionar automáticamente emulaciones:

```text
semantically safe
```

---

# 256. Explicit lossy mode

Si algún día se permite una estrategia no idéntica, deberá requerir opt-in explícito.

---

# 257. Capability documentation

Cada capability deberá documentar:

```text
ID
meaning
value type
scope
providers
consumers
native semantics
emulation rules
version considerations
security considerations
performance considerations
```

---

# 258. Capability catalog

Podrá generarse automáticamente desde descriptors.

---

# 259. Documentation generation

Esto permitirá producir:

```text
MySQL Capability Matrix
MariaDB Capability Matrix
PostgreSQL Capability Matrix
SQLite Capability Matrix
```

sin mantener tablas manuales inconsistentes.

---

# 260. Capability matrix

Conceptualmente:

| Capability | MySQL | MariaDB | PostgreSQL | SQLite |
|---|---|---|---|---|
| `transaction.savepoint` | resolved | resolved | resolved | resolved |
| `query.returning.insert` | resolved | resolved | resolved | resolved |
| `query.cte.recursive` | resolved | resolved | resolved | resolved |
| `lock.skip_locked` | resolved | resolved | resolved | resolved |

Los valores concretos dependerán de:

```text
platform version
driver
configuration
runtime
```

por lo que no deberán codificarse como una tabla universal fija.

---

# 261. Why no static universal matrix

Una tabla:

```text
PostgreSQL = supports X
```

puede ser insuficiente porque:

```text
version
extensions
permissions
configuration
driver
connection role
```

pueden modificar el resultado efectivo.

---

# 262. Capability scopes

Las capabilities podrán tener scope.

```text
PLATFORM
DRIVER
CONNECTION
TRANSACTION
QUERY
SCHEMA
RUNTIME
EXTENSION
```

---

# 263. Platform-scoped capability

Ejemplo:

```text
schema.partial_index
```

---

# 264. Connection-scoped capability

Ejemplo:

```text
connection.write
```

---

# 265. Transaction-scoped capability

Ejemplo:

```text
transaction.current_isolation
```

si se modela como capability contextual.

---

# 266. Operation-scoped capability

Sólo cuando sea realmente necesario.

No convertir Capability System en un contenedor de todo estado runtime.

---

# 267. Capability vs state

Muy importante:

```text
Capability
=
what can be done

State
=
what is happening now
```

---

# 268. Example

```text
supportsTransactions
=
capability

transactionActive
=
state
```

No mezclarlos.

---

# 269. Capability vs configuration

```text
Capability
=
effective technical possibility

Configuration
=
desired framework setup
```

---

# 270. Capability vs permission

```text
Database technical capability
≠
Application authorization permission
```

---

# 271. Database server privileges

Los privilegios del usuario del servidor pueden restringir capabilities operativas.

---

# 272. Permission-aware discovery

Ejemplo:

```text
Platform supports CREATE TABLE

Credential lacks CREATE privilege

Effective administrative capability:
restricted
```

---

# 273. Avoid authorization coupling

Esto no implica dependencia de:

```text
VoltStack/Quantum/Authorization
```

Son conceptos diferentes.

---

# 274. Capability discovery security

Las queries de discovery deberán:

```text
use minimal privileges
avoid exposing secrets
avoid destructive operations
```

---

# 275. Discovery cache

Resultados costosos podrán cachearse.

---

# 276. Discovery cache key

Podrá considerar:

```text
ConnectionDefinitionFingerprint
PlatformVersion
CredentialGeneration
ServerConfigurationGeneration
```

sin incluir secretos.

---

# 277. Discovery invalidation

Invalidar cuando cambien:

```text
server version
connection generation
relevant server configuration
extensions
credentials/privileges when relevant
```

---

# 278. Capability refresh

No deberá ocurrir por cada query.

---

# 279. Background refresh

Podrá existir en futuras versiones para deployments dinámicos, pero no será requisito del core V1.

---

# 280. Hot reload

Si se soporta:

```text
new capability generation
```

se publicará de forma atómica.

---

# 281. In-flight operation safety

Una query ya planificada continuará usando:

```text
CapabilitySnapshot N
```

aunque aparezca:

```text
CapabilitySnapshot N+1
```

---

# 282. Platform capability compilation

Reglas estáticas podrán precompilarse.

---

# 283. Compiled capability rules

Ejemplo:

```text
PlatformCapabilityDefinition
      │
      ▼
CapabilityRuleCompiler
      │
      ▼
CompiledCapabilityProfile
```

---

# 284. Production optimization

Evitar:

```text
hundreds of dynamic rule evaluations
per query
```

---

# 285. Capability resolution frequency

Preferencia:

```text
resolve once per compatible context
reuse snapshot many times
```

---

# 286. Query hot path

Ideal:

```text
CapabilitySnapshot
    -> O(1) lookup
```

---

# 287. Capability index

Internamente podrá utilizarse:

```text
integer IDs
bitsets
typed arrays
immutable maps
```

si benchmarking demuestra beneficio.

---

# 288. Public API independence

La optimización interna no deberá cambiar los IDs/contracts públicos.

---

# 289. Boolean capability optimization

Boolean capabilities frecuentes podrían compilarse a:

```text
bitset
```

---

# 290. Structured capabilities

Permanecerían en tablas tipadas separadas.

---

# 291. No premature optimization

V1 deberá priorizar:

```text
correctness
clarity
determinism
```

antes que micro-optimizaciones.

---

# 292. Capability groups

Organización sugerida:

```text
Connection
Transaction
Query
Expression
Result
Schema
Migration
Type
JSON
Locking
Bulk
Runtime
Security
Administration
```

---

# 293. Connection group

Ejemplos:

```text
connection.read
connection.write
connection.reset
connection.ping
connection.cancel
connection.tls
```

---

# 294. Transaction group

```text
transaction.supported
transaction.savepoint
transaction.read_only
transaction.isolation
transaction.transactional_ddl
```

---

# 295. Query group

```text
query.cte
query.recursive_cte
query.window
query.returning
query.upsert
query.union
query.intersect
query.except
```

---

# 296. Expression group

```text
expression.regex
expression.case_insensitive_like
expression.null_ordering
expression.generated
```

---

# 297. Result group

```text
result.streaming
result.cursor
result.multiple_sets
```

---

# 298. Schema group

```text
schema.alter_table
schema.rename_column
schema.drop_column
schema.generated_column
schema.partial_index
schema.expression_index
```

---

# 299. Migration group

```text
migration.transactional_ddl
migration.online_index
migration.concurrent_index
migration.safe_column_add
```

---

# 300. JSON group

```text
json.native_type
json.path
json.query
json.mutation
json.index
```

---

# 301. Bulk group

```text
bulk.insert
bulk.update
bulk.delete
bulk.returning
```

---

# 302. Runtime group

```text
runtime.persistent_worker
runtime.concurrent_execution
runtime.connection_reuse
```

---

# 303. Capability naming rule

Usar nombres:

```text
semantic
stable
implementation-independent
```

---

# 304. Avoid method explosion

No se recomienda que `DatabasePlatformInterface` tenga cientos de métodos:

```php
supportsReturning();
supportsJson();
supportsCTE();
supportsRecursiveCTE();
supportsWindowFunctions();
// ... 200 more
```

---

# 305. Typed catalog instead

Preferir:

```php
$capabilities->supports(
    DatabaseCapability::QueryReturning
);
```

---

# 306. Convenience methods

Podrán existir wrappers especializados:

```php
$queryCapabilities->supportsReturning();
```

sobre el mismo catálogo central.

---

# 307. Capability views

Ejemplo:

```text
EffectiveCapabilitySet
├── query()
├── schema()
├── transaction()
├── result()
└── connection()
```

---

# 308. Typed view example

```php
$capabilities
    ->transaction()
    ->supportsSavepoints();
```

---

# 309. One source of truth

Los typed views no deberán mantener copias divergentes.

---

# 310. Platform capability builder

Durante resolución podrá utilizarse:

```text
CapabilitySetBuilder
```

---

# 311. Builder is mutable only during construction

Después:

```text
freeze()
```

---

# 312. Runtime immutable set

```text
CapabilitySetBuilder
      │
      ▼
EffectiveCapabilitySet
```

---

# 313. No mutation after freeze

```php
$capabilities->enable(...);
```

durante ejecución estará prohibido.

---

# 314. Extension registration timing

Extensiones deberán registrar:

```text
capability descriptors
providers
requirements
```

antes de freeze.

---

# 315. Dynamic extensions

No serán soportadas por defecto dentro de una request.

---

# 316. Capability override

Una extensión oficial podrá refinar una capability sólo mediante reglas explícitas.

---

# 317. Override diagnostics

Todo override deberá poder explicarse.

---

# 318. Capability conflict example

```text
Provider A:
NATIVE

Provider B:
UNSUPPORTED
```

sin relación de autoridad definida:

```text
CapabilityConflictException
```

---

# 319. No silent conflict resolution

No:

```text
last provider wins
```

---

# 320. Testing architecture

Capability System deberá tener varias capas de pruebas.

---

# 321. Unit tests

Para:

```text
CapabilityId
CapabilityValue
CapabilitySet
RequirementEvaluator
RuleEngine
Fingerprint
```

---

# 322. Platform tests

Cada Platform deberá probar:

```text
version-specific rules
feature combinations
unsupported cases
```

---

# 323. Driver/platform combination tests

Ejemplo:

```text
Platform A
+
Driver B
=
expected effective capabilities
```

---

# 324. Policy tests

Verificar:

```text
native capability
+
disabled policy
=
DISABLED
```

---

# 325. Connection role tests

```text
write-capable platform
+
read-only connection
=
write disabled
```

---

# 326. Emulation tests

Cada emulación deberá demostrar:

```text
semantic equivalence
```

---

# 327. Persistent runtime tests

Request A y B con capability sets diferentes no deberán contaminarse.

---

# 328. Parallel tests

Especialmente para:

```text
OpenSwoole
fibers
concurrent workers
```

---

# 329. Fingerprint tests

Mismos inputs:

```text
same fingerprint
```

Inputs relevantes distintos:

```text
different fingerprint
```

---

# 330. Secret exclusion tests

Cambiar sólo el valor secreto no deberá introducir el secreto en:

```text
logs
fingerprints
diagnostics
```

---

# 331. Architecture tests

Prohibir vendor checks en:

```text
ORM
Query Builder
Semantic Engine
generic Planner rules
generic Schema Model
Repository
EntityManager
UnitOfWork
```

salvo componentes vendor-specific autorizados.

---

# 332. Allowed vendor-specific locations

Principalmente:

```text
Platform/<Vendor>
Dialect/<Vendor>
Driver/<Vendor>
Compiler/<Vendor>
Schema/<Vendor>
Capability/<Vendor>
```

cuando realmente corresponda.

---

# 333. Suggested namespace

```text
VoltStack\Quantum\Database\Platform
```

---

# 334. Suggested directory structure

```text
Platform/
├── Contract/
│   ├── DatabasePlatformInterface.php
│   ├── PlatformResolverInterface.php
│   └── PlatformCapabilityProviderInterface.php
│
├── Descriptor/
│   ├── PlatformId.php
│   ├── PlatformDescriptor.php
│   ├── PlatformFamily.php
│   ├── PlatformVersion.php
│   └── PlatformCompatibilityProfile.php
│
├── Resolution/
│   ├── PlatformResolver.php
│   ├── PlatformResolutionContext.php
│   ├── PreliminaryPlatform.php
│   └── ResolvedPlatform.php
│
├── Capability/
│   ├── CapabilityId.php
│   ├── CapabilityDescriptor.php
│   ├── CapabilityStatus.php
│   ├── CapabilityValue.php
│   ├── CapabilitySupport.php
│   ├── CapabilityRegistry.php
│   ├── CapabilitySet.php
│   ├── EffectiveCapabilitySet.php
│   ├── CapabilitySnapshot.php
│   ├── CapabilityFingerprint.php
│   ├── CapabilityGeneration.php
│   ├── CapabilityProvenance.php
│   ├── CapabilityResolutionTrace.php
│   └── CapabilityResolver.php
│
├── Capability/Requirement/
│   ├── CapabilityRequirement.php
│   ├── CapabilityRequirementSet.php
│   ├── CapabilityRequirementEvaluator.php
│   ├── CapabilityExpression.php
│   ├── AllOf.php
│   ├── AnyOf.php
│   ├── Not.php
│   └── ValueComparison.php
│
├── Capability/Provider/
│   ├── CapabilityProviderInterface.php
│   ├── PlatformCapabilityProvider.php
│   ├── DriverCapabilityProvider.php
│   ├── RuntimeCapabilityProvider.php
│   ├── ConfigurationCapabilityProvider.php
│   └── ExtensionCapabilityProvider.php
│
├── Capability/Rule/
│   ├── CapabilityRule.php
│   ├── CapabilityRuleSet.php
│   ├── CapabilityRuleEngine.php
│   ├── VersionCapabilityRule.php
│   ├── DependencyCapabilityRule.php
│   └── RestrictionCapabilityRule.php
│
├── Capability/View/
│   ├── QueryCapabilities.php
│   ├── SchemaCapabilities.php
│   ├── TransactionCapabilities.php
│   ├── ResultCapabilities.php
│   └── ConnectionCapabilities.php
│
├── Feature/
│   ├── FeatureId.php
│   ├── FeatureDescriptor.php
│   ├── FeatureStrategy.php
│   └── FeatureStrategyResolver.php
│
├── MySql/
├── MariaDb/
├── PostgreSql/
├── SQLite/
│
└── Exception/
    ├── PlatformException.php
    ├── PlatformResolutionException.php
    ├── CapabilityException.php
    ├── CapabilityNotFoundException.php
    ├── CapabilityNotSupportedException.php
    ├── CapabilityDisabledException.php
    ├── CapabilityRequirementException.php
    ├── CapabilityConflictException.php
    └── CapabilityDiscoveryException.php
```

---

# 335. Dependency direction

```text
Query / ORM / Schema / Migration
              │
              ▼
      Capability Contracts
              ▲
              │
      Platform Capability
              ▲
       ┌──────┼──────┐
       │      │      │
     Driver Runtime Extensions
```

---

# 336. Platform dependency rule

Platform podrá depender de:

```text
Platform contracts
Capability contracts
Type abstractions
neutral metadata
```

pero no de:

```text
ORM
EntityManager
Repository
UnitOfWork
HTTP
Controller
Application domain
```

---

# 337. Capability dependency rule

Capability Core deberá permanecer independiente de:

```text
specific vendors
specific runtimes
specific applications
```

---

# 338. Public API

La mayoría de desarrolladores no necesitarán consultar capabilities directamente.

La API deberá funcionar automáticamente.

---

# 339. Advanced API

Para paquetes/infraestructura:

```php
$capabilities = DB::connection('primary')
    ->capabilities();

if ($capabilities->supports(
    DatabaseCapability::QueryReturning
)) {
    // advanced behavior
}
```

---

# 340. Require API

Conceptualmente:

```php
$capabilities->require(
    DatabaseCapability::TransactionSavepoint
);
```

---

# 341. Feature API

También podría existir:

```php
$connection->features()
    ->require(DatabaseFeature::NestedTransactions);
```

que evalúe varias capabilities.

---

# 342. Capability API should not expose vendor assumptions

No:

```php
$capabilities->isPostgres();
```

como mecanismo normal para seleccionar comportamiento.

---

# 343. Platform identity remains available

Conocer el Platform ID seguirá siendo válido para:

```text
diagnostics
vendor-specific extensions
explicit platform-specific APIs
```

pero no para lógica portable ordinaria.

---

# 344. Anti-pattern — Vendor branching

```php
switch ($platform->id()) {
    case 'mysql':
    case 'pgsql':
    case 'sqlite':
}
```

en código genérico.

**Prohibido.**

---

# 345. Anti-pattern — Giant Platform object

```text
Platform
├── compile SQL
├── execute SQL
├── open PDO
├── hydrate entities
├── manage transactions
├── manage pool
└── manage capabilities
```

**Prohibido.**

---

# 346. Anti-pattern — Hundreds of supports methods

```php
$platform->supportsA();
$platform->supportsB();
$platform->supportsC();
```

hasta convertir Platform en una interfaz inmanejable.

---

# 347. Anti-pattern — Static capability table

```php
$capabilities['postgresql']['returning'] = true;
```

como única arquitectura.

No considera:

```text
version
driver
configuration
connection
runtime
extensions
```

---

# 348. Anti-pattern — Capability equals version check

```php
if ($version >= 10) {
}
```

disperso por todo el framework.

---

# 349. Anti-pattern — Assume capability

```php
try {
    executeVendorSyntax();
} catch (...) {
    fallback();
}
```

como mecanismo normal de detección.

---

# 350. Anti-pattern — Runtime mutation

```php
$capabilities->set(
    'query.returning',
    true
);
```

durante una request.

---

# 351. Anti-pattern — Cross-request capability state

```text
Tenant A resolves capability
→ global singleton mutated
→ Tenant B sees Tenant A capability
```

**Crítico y prohibido.**

---

# 352. Anti-pattern — Feature equals capability

```text
NestedTransaction = supportsSavepoints
```

sin modelar las demás condiciones necesarias.

---

# 353. Anti-pattern — Unsafe emulation

```text
native unsupported
→ do something approximately similar
→ report supported
```

**Prohibido.**

---

# 354. Anti-pattern — Capability bypass

Un extension package no deberá saltarse:

```text
Capability validation
```

y generar vendor SQL arbitrariamente desde una API que afirma ser portable.

---

# 355. Anti-pattern — Configuration as capability truth

Configurar:

```text
supportsReturning: true
```

no hace que el servidor lo soporte.

---

# 356. Configuration hints

La configuración puede proporcionar:

```text
target hints
restrictions
required features
```

pero no falsificar soporte técnico.

---

# 357. Offline target assumptions

En offline mode podrán existir:

```text
declared capabilities
```

derivadas de:

```text
platform + target version
```

pero deberán marcarse con provenance adecuada.

---

# 358. Verification

Deployment tooling podrá verificar:

```text
declared target capabilities
vs
actual server capabilities
```

---

# 359. Capability drift

Si difieren:

```text
CapabilityDriftDetected
```

podrá producir warning/error.

---

# 360. Production readiness

Un comando futuro:

```text
php volt database:verify
```

podrá comprobar:

```text
configured target
actual platform
actual version
required capabilities
migration capabilities
runtime safety
```

---

# 361. Capability telemetry

Métricas/eventos útiles:

```text
capability_resolution_count
capability_resolution_failure
capability_emulation_selected
capability_requirement_failure
capability_drift
```

---

# 362. Avoid cardinality explosion

No usar arbitrariamente:

```text
tenant ID
raw connection string
query SQL
```

como labels.

---

# 363. Security of diagnostics

Capability traces pueden revelar:

```text
server version
installed extensions
configuration
```

por lo que su exposición HTTP deberá estar protegida.

---

# 364. Developer environment

Información detallada puede estar disponible en:

```text
CLI
Profiler
Debug Toolbar
```

según políticas.

---

# 365. Production environment

Exponer sólo información necesaria y segura.

---

# 366. Capability lifecycle

```text
REGISTER
   │
   ▼
DEFINE
   │
   ▼
RESOLVE BASELINE
   │
   ▼
REFINE BY VERSION
   │
   ▼
REFINE BY DRIVER
   │
   ▼
DISCOVER RUNTIME
   │
   ▼
APPLY CONNECTION RESTRICTIONS
   │
   ▼
APPLY POLICY
   │
   ▼
APPLY VALID EXTENSIONS
   │
   ▼
VALIDATE DEPENDENCIES
   │
   ▼
FREEZE
   │
   ▼
CapabilitySnapshot
```

---

# 367. Platform lifecycle

```text
Configuration
      │
      ▼
Platform Hint
      │
      ▼
Preliminary Platform
      │
      ▼
Physical Connection
      │
      ▼
Server Metadata
      │
      ▼
Resolved Platform
      │
      ▼
Capability Resolution
```

---

# 368. Connection lifecycle integration

El Capability Snapshot deberá asociarse a:

```text
compatible connection generation
```

sin convertirse en estado mutable del Connection singleton.

---

# 369. Reset interaction

Resetear una conexión no necesariamente invalida capabilities estáticas.

---

# 370. Session-dependent capability

Si una capability depende de session configuration:

```text
reset
```

deberá restaurar la configuración esperada o invalidar el snapshot contextual.

---

# 371. Server failover

Si una conexión cambia a otro servidor:

```text
capability compatibility
```

deberá verificarse.

---

# 372. Replica topology

Primary y replica podrían tener:

```text
different versions
different extensions
different capabilities
```

---

# 373. Routing safety

Topology/ReadWrite Router no deberá asumir:

```text
all replicas equivalent
```

sin verificación.

---

# 374. Capability-compatible routing

Una operación podrá declarar:

```text
ConnectionRequirement
+
CapabilityRequirementSet
```

---

# 375. Example

```text
Query requires:
query.cte.recursive

Replica A:
supported

Replica B:
unsupported

Router:
must select compatible target
```

---

# 376. Capability-aware topology

Esto permitirá en el futuro:

```text
routing by capability
```

además de:

```text
role
health
load
consistency
```

---

# 377. Sharding

Cada shard podrá tener un capability fingerprint.

---

# 378. Shard compatibility policy

Un cluster homogéneo podrá exigir:

```text
same required capability profile
```

---

# 379. Heterogeneous clusters

Podrán existir sólo con routing explícitamente capability-aware.

---

# 380. Migration safety in distributed systems

Antes de ejecutar migrations sobre múltiples targets:

```text
validate capability profile
```

de todos ellos.

---

# 381. Minimum cluster capability

Podrá calcularse:

```text
intersection
```

de capabilities.

---

# 382. ClusterCapabilitySet

Futuro:

```text
ClusterCapabilitySet
```

podrá representar:

```text
intersection
union
per-node differences
```

---

# 383. No premature distributed complexity

No será requisito del V1 core, pero el diseño no deberá impedirlo.

---

# 384. Capability conformance suite

Cada Platform oficial deberá ejecutar:

```text
PlatformCapabilityConformanceSuite
```

---

# 385. Conformance objective

Verificar que lo declarado por VoltStack coincide con el comportamiento real del servidor.

---

# 386. Test matrix

Idealmente:

```text
Platform
×
Supported Server Versions
×
Official Driver
```

---

# 387. Capability probe tests

Algunas capabilities podrán verificarse mediante probes no destructivos.

---

# 388. Destructive probes

No deberán ejecutarse automáticamente en producción.

---

# 389. Test-only probes

Operaciones destructivas podrán existir sólo en:

```text
integration/conformance test environment
```

---

# 390. Capability declarations are contracts

Si VoltStack declara:

```text
NATIVE
```

debe existir una prueba que demuestre el comportamiento.

---

# 391. Emulation declarations are stronger contracts

Si declara:

```text
EMULATED
```

debe demostrar equivalencia semántica.

---

# 392. Documentation consistency

La matriz pública de soporte deberá generarse desde los mismos descriptors probados.

---

# 393. Architectural invariants

## DB-CAP-001

Las capas genéricas preguntarán por capabilities, no por vendor.

## DB-CAP-002

Driver, Dialect, Platform y Capability permanecerán separados.

## DB-CAP-003

Feature y Capability no serán sinónimos.

## DB-CAP-004

Technical capability y policy serán conceptos distintos.

## DB-CAP-005

Capabilities podrán ser booleanas, numéricas, enum, set o estructuradas.

## DB-CAP-006

Capability IDs serán estables y semánticos.

## DB-CAP-007

Capabilities generales no utilizarán vendor names innecesariamente.

## DB-CAP-008

CapabilityRegistry describirá capabilities; no almacenará estado contextual mutable.

## DB-CAP-009

EffectiveCapabilitySet será inmutable.

## DB-CAP-010

Capability resolution será determinista.

## DB-CAP-011

Registration order no decidirá conflictos silenciosamente.

## DB-CAP-012

Version checks estarán centralizados.

## DB-CAP-013

Platform version será distinta de driver/client version.

## DB-CAP-014

MySQL y MariaDB tendrán Platform capability definitions separadas.

## DB-CAP-015

Un protocolo compatible no implicará capabilities idénticas.

## DB-CAP-016

Driver capabilities describirán transporte, no semántica general del motor.

## DB-CAP-017

Dialect syntax no implicará automáticamente feature support.

## DB-CAP-018

Connection context podrá restringir capabilities.

## DB-CAP-019

Runtime podrá restringir capabilities de lifecycle/concurrency.

## DB-CAP-020

Configuration podrá restringir, pero no falsificar soporte técnico.

## DB-CAP-021

Extensions sólo proporcionarán capabilities cuando exista implementación real.

## DB-CAP-022

UNKNOWN no será tratado como SUPPORTED.

## DB-CAP-023

Emulation sólo se utilizará cuando preserve semántica.

## DB-CAP-024

Emulación lossy nunca será silenciosa.

## DB-CAP-025

Planner seleccionará estrategias; Capability System describirá posibilidades.

## DB-CAP-026

Dialect expresará la estrategia elegida.

## DB-CAP-027

Query Builder no generará vendor branches.

## DB-CAP-028

ORM no utilizará vendor branches para comportamiento portable.

## DB-CAP-029

Schema Model permanecerá vendor-neutral cuando sea posible.

## DB-CAP-030

Migration Planner utilizará capabilities para estrategias.

## DB-CAP-031

Capability snapshots serán estables durante una operación.

## DB-CAP-032

No existirá global mutable current capability set.

## DB-CAP-033

No existirá global mutable current platform.

## DB-CAP-034

Capability resolution será segura para persistent runtimes.

## DB-CAP-035

Capability resolution será concurrency-safe.

## DB-CAP-036

Tenant-specific capabilities no contaminarán otros tenants.

## DB-CAP-037

Multi-connection applications podrán tener capability sets distintos.

## DB-CAP-038

Fingerprints nunca contendrán secretos.

## DB-CAP-039

Compiled query caches considerarán capabilities relevantes.

## DB-CAP-040

Capability discovery será lazy cuando requiera servidor.

## DB-CAP-041

Bootstrap no abrirá conexiones sólo para descubrir capabilities.

## DB-CAP-042

Offline target compilation será posible cuando haya metadata suficiente.

## DB-CAP-043

Capability provenance estará disponible para diagnostics.

## DB-CAP-044

Diagnostics estarán redacted.

## DB-CAP-045

Capability conflicts serán explícitos.

## DB-CAP-046

Capability dependencies serán validables.

## DB-CAP-047

Dependency cycles serán rechazados.

## DB-CAP-048

Derived capabilities no se duplicarán innecesariamente por Platform.

## DB-CAP-049

Cada Platform oficial tendrá conformance tests.

## DB-CAP-050

Cada capability declarada como NATIVE deberá ser verificable.

## DB-CAP-051

Cada capability EMULATED deberá documentar equivalencia y límites.

## DB-CAP-052

Platform no será un God object.

## DB-CAP-053

Platform no ejecutará queries.

## DB-CAP-054

Platform no administrará ORM state.

## DB-CAP-055

Capability System no administrará connection pools.

## DB-CAP-056

Capability System no será un service locator.

## DB-CAP-057

Capability sets runtime estarán frozen.

## DB-CAP-058

Hot reload, si existe, utilizará generations/snapshots.

## DB-CAP-059

In-flight operations no cambiarán de capability snapshot.

## DB-CAP-060

Capability-aware routing será posible sin introducir vendor coupling.

---

# 394. Decisiones arquitectónicas principales

Se adoptan oficialmente las siguientes decisiones:

### 1. Capability-first

```text
supports(feature)
```

antes que:

```text
isVendor(...)
```

### 2. Semantic capability IDs

```text
query.returning
transaction.savepoint
schema.partial_index
```

### 3. Rich capability model

No limitar todo a booleanos.

### 4. Effective capabilities

El soporte efectivo depende de:

```text
Platform
+
Version
+
Driver
+
Runtime
+
Connection
+
Configuration
+
Extensions
```

### 5. Immutable snapshots

Cada operación trabajará con un snapshot estable.

### 6. Planner-driven strategies

Capability System informa.

Planner decide.

### 7. Dialect-driven syntax

Dialect expresa.

### 8. Platform-driven semantics

Platform define comportamiento del motor.

### 9. Lazy discovery

No abrir DB durante bootstrap innecesariamente.

### 10. Persistent-runtime safety

Nada de current platform/capability global mutable.

---

# 395. Ejemplo completo — RETURNING

La aplicación expresa:

```php
DB::table('users')
    ->insertReturning(
        ['name' => 'Ada'],
        ['id']
    );
```

Query Model:

```text
Insert
├── table: users
├── values
└── returning: [id]
```

Semantic Engine:

```text
requires semantic returning
```

Capability System:

```text
query.returning.insert
```

Planner:

```text
if NATIVE
    NativeReturningStrategy

else if safe emulation exists
    EmulatedGeneratedValueStrategy

else
    Unsupported
```

Compiler:

```text
NativeReturningStrategy
+
ResolvedDialect
→ SQL
```

Executor:

```text
CompiledQuery
→ Connection
→ Driver
```

Ninguna capa genérica necesitó:

```php
if ($driver === 'pgsql')
```

---

# 396. Ejemplo completo — Nested transaction

Aplicación:

```php
DB::transaction(function () {
    DB::transaction(function () {
        // ...
    });
});
```

Feature:

```text
transaction.nested
```

Requirements:

```text
transaction.supported
+
transaction.savepoint
```

Capability System:

```text
savepoint = NATIVE
```

Transaction Manager:

```text
outer transaction
    │
    └── inner transaction
          │
          ▼
       savepoint
```

Si no existe soporte:

```text
CapabilityNotSupportedException
```

o una estrategia explícitamente definida.

---

# 397. Ejemplo completo — SQLite migration

Schema intent:

```text
Drop Column
```

Capability Snapshot:

```text
schema.drop_column = <effective support>
```

Migration Planner:

```text
Native
    or
TableRebuildStrategy
    or
Unsupported
```

Schema Compiler:

```text
selected strategy
+
SQLiteDialect
→ SQL sequence
```

---

# 398. Ejemplo completo — read-only replica

Platform:

```text
PostgreSQL
write capability = NATIVE
```

Connection:

```text
role = REPLICA
```

Connection restriction:

```text
write = DISABLED
```

Effective Capability Set:

```text
connection.write = DISABLED
```

Una operación de escritura no deberá llegar al servidor por accidente.

---

# 399. Arquitectura final

```text
                     DATABASE TARGET
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
       Driver           Platform          Dialect
          │                │                │
          ▼                ▼                ▼
 Driver Capabilities  Platform Rules   Syntax Rules
          │                │                │
          └────────────┬───┴────────────────┘
                       ▼
                Server Version
                       │
                       ▼
               Server Discovery
                       │
                       ▼
                Runtime Context
                       │
                       ▼
               Connection Context
                       │
                       ▼
                 Configuration
                       │
                       ▼
                   Extensions
                       │
                       ▼
               CapabilityResolver
                       │
                       ▼
             EffectiveCapabilitySet
                       │
          ┌────────────┼───────────────┐
          ▼            ▼               ▼
       Semantic      Planner         Schema
          │            │               │
          ▼            ▼               ▼
      Optimizer     Compiler       Migration
                       │               │
                       ├───────────────┤
                       ▼               ▼
                      ORM         Transaction
```

---

# 400. Fórmula maestra

```text
Effective Capabilities
=
Platform Baseline
+
Version Refinement
+
Driver Constraints
+
Runtime Constraints
+
Connection Constraints
+
Server Discovery
+
Valid Extension Contributions
-
Policy Restrictions
```

---

# 401. Fórmula de feature

```text
Feature Availability
=
Capability Requirements
×
Effective Capability Set
×
Policy
```

---

# 402. Fórmula de ejecución

```text
Semantic Intent
        │
        ▼
Capability Requirements
        │
        ▼
Effective Capability Set
        │
        ▼
Planner Strategy
        │
        ▼
Dialect Syntax
        │
        ▼
CompiledQuery
```

---

# 403. Regla maestra

> **Las capas superiores de VoltStack no deberán decidir comportamiento preguntando qué motor de base de datos están utilizando; deberán declarar qué necesitan y consultar qué capacidades efectivas están disponibles.**

En forma resumida:

```text
Vendor identity
=
diagnostic information

Capability
=
architectural decision information
```

---

# 404. Resultado arquitectónico

Con este sistema, VoltStack podrá soportar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

y posteriormente otros motores sin convertir:

```text
Query Builder
ORM
Schema
Migration
Transaction
Persistence
```

en una colección de:

```php
switch ($database) {
}
```

La arquitectura será:

```text
Semantic Intent
      │
      ▼
Capability Requirement
      │
      ▼
Effective Capability Set
      │
      ▼
Strategy Selection
      │
      ▼
Dialect Compilation
```

Esto proporciona simultáneamente:

```text
portability
extensibility
correctness
version awareness
runtime safety
testability
diagnostics
performance
```

---

# 405. Relación con los documentos anteriores

```text
10_DATABASE_DRIVER_ARCHITECTURE
          │
11_DATABASE_CONNECTION_SYSTEM
          │
12_DATABASE_CONNECTION_MANAGER
          │
13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION
          │
14_DATABASE_CONNECTION_POOLING_SYSTEM
          │
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM
          │
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM
          │
17_DATABASE_DIALECT_SYSTEM
          │
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM
```

Con esto quedan formalmente separadas:

```text
native communication
logical connection
connection resolution
pooling
lifecycle
state/reset
SQL syntax
database semantics/capabilities
```

---

# 406. Relación Dialect vs Platform Capability

La separación definitiva será:

```text
                 Semantic Operation
                         │
                         ▼
                       Planner
                         │
            ┌────────────┴────────────┐
            ▼                         ▼
       Capabilities                 Platform
   "Can we do it?"             "What does it mean
            │                   on this engine?"
            └────────────┬────────────┘
                         ▼
                      Compiler
                         │
                         ▼
                       Dialect
                 "How is it written?"
                         │
                         ▼
                         SQL
```

---

# 407. Siguiente etapa

Con `18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md` queda establecida la infraestructura genérica.

Los siguientes documentos aplicarán esta arquitectura a plataformas concretas:

```text
19_DATABASE_MYSQL_AND_MARIADB_PLATFORM.md
20_DATABASE_POSTGRESQL_PLATFORM.md
21_DATABASE_SQLITE_PLATFORM.md
22_DATABASE_DRIVER_EXTENSION_SYSTEM.md
```

El documento `19_DATABASE_MYSQL_AND_MARIADB_PLATFORM.md` deberá definir especialmente:

```text
MySqlPlatform
MariaDbPlatform

shared MySQL-family infrastructure
MySQL/MariaDB divergence model
server identification
version detection
capability providers
type semantics
identifier semantics
transaction semantics
schema semantics
locking
generated values
JSON
UPSERT
RETURNING strategy
DDL behavior
server variables
session state
charset/collation
platform discovery
compatibility fingerprints
conformance testing
```

manteniendo la regla:

```text
pdo.mysql
        │
        ├────────► MySqlPlatform
        │
        └────────► MariaDbPlatform
```

porque compartir un transporte no significa compartir completamente una plataforma.

---

# 408. Conclusión

`Database Platform Capability System` se convierte en una de las piezas centrales de la nueva arquitectura de `VoltStack/Quantum/Database`.

Su función no es implementar SQL ni comunicarse con el servidor.

Su función es proporcionar una respuesta confiable, contextual y tipada a:

```text
¿Qué puede hacer realmente
este target de base de datos?
```

La separación completa queda:

```text
Driver
=
communication

Connection
=
managed logical access

Dialect
=
SQL syntax

Platform
=
database semantics

Capability
=
effective technical possibility

Policy
=
allowed behavior

Feature
=
framework functionality

Planner
=
strategy selection

Compiler
=
translation

Executor
=
execution
```

La arquitectura final seguirá la regla:

> **Intent first. Capabilities second. Strategy third. Syntax last.**