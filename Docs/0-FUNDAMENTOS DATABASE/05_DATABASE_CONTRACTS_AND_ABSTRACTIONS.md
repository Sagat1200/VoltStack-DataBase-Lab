# 05_DATABASE_CONTRACTS_AND_ABSTRACTIONS.md

# VoltStack Quantum Database
## Contracts and Abstractions

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 05 — Database Contracts and Abstractions  
**Estado:** Architecture Specification  
**Nivel:** Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el sistema oficial de contratos y abstracciones de:

```text
VoltStack/Quantum/Database
```

Establece:

- qué componentes requieren interfaces;
- qué contratos son públicos;
- qué contratos son internos;
- qué contratos son puntos de extensión;
- qué contratos pertenecen a `VoltStack/Platform`;
- qué abstracciones permanecen en `Quantum/Database`;
- cómo se implementará Dependency Inversion;
- cómo se conectarán módulos opcionales;
- cómo se evitará el Service Locator;
- cómo se diseñarán Ports y Adapters;
- cómo se versionarán los contratos;
- cómo se probará su conformidad.

El objetivo no es convertir cada clase en una interfaz.

El objetivo es colocar abstracciones exactamente donde exista una frontera arquitectónica real.

---

# 2. Principio fundamental

VoltStack adopta la siguiente regla:

> Una abstracción debe existir porque protege una frontera, no simplemente porque una implementación existe.

Por tanto:

```text
Interface != automáticamente necesaria
```

Ejemplo incorrecto:

```text
UserFactoryInterface
UserFactory
```

cuando:

- existe una sola implementación;
- no hay extensión;
- no hay sustitución;
- no existe boundary arquitectónico.

Ejemplo correcto:

```text
DriverInterface
 ├── MySQLDriver
 ├── PostgreSQLDriver
 ├── MariaDBDriver
 └── SQLiteDriver
```

Aquí existe una frontera real.

---

# 3. Objetivos del Contract System

El sistema de contratos deberá permitir:

```text
Dependency Inversion
Replaceable Infrastructure
Module Isolation
Testing Seams
Extension Points
Optional Integrations
Runtime Adaptation
Stable Public APIs
```

sin provocar:

```text
interface explosion
abstraction leakage
service locator architecture
excessive indirection
unnecessary factories
```

---

# 4. Taxonomía de contratos

VoltStack Database clasificará los contratos en cinco categorías:

```text
1. Platform Contracts
2. Public Contracts
3. Extension Contracts
4. Integration Contracts
5. Internal Contracts
```

Conceptualmente:

```text
Contracts
│
├── Platform
├── Public
├── Extension
├── Integration
└── Internal
```

---

# 5. Platform Contracts

Los `Platform Contracts` representan abstracciones mínimas que otros subsistemas fundamentales de VoltStack pueden necesitar sin depender de `Quantum/Database`.

Residirán en:

```php
VoltStack\Platform\Database
```

Su número deberá mantenerse deliberadamente pequeño.

---

# 6. Regla de Platform

`VoltStack/Platform` representa una capa inferior.

Por tanto:

```text
Platform
   ✗
   │
   ▼
Quantum/Database
```

está prohibido.

En cambio:

```text
Quantum/Database
       │
       ▼
Platform
       ▲
       │
Other Quantum Modules
```

es válido.

---

# 7. Candidatos iniciales para Platform

Los contratos candidatos son:

```text
DatabaseManagerInterface
ConnectionInterface
TransactionManagerInterface
TransactionInterface
DatabaseContextInterface
```

Sin embargo, únicamente deberán promoverse a `Platform` si otro subsistema realmente necesita depender de ellos.

---

# 8. Minimal Platform Surface

El objetivo será mantener una superficie como:

```text
VoltStack\Platform\Database
│
├── DatabaseManagerInterface
├── ConnectionInterface
├── TransactionInterface
└── Exception/
    └── DatabaseExceptionInterface
```

y evitar trasladar a Platform componentes como:

```text
Query AST
ORM
UnitOfWork
IdentityMap
Hydrator
Compiler
Schema Builder
Migration Planner
```

Estos pertenecen al paquete Database.

---

# 9. Public Contracts

Los Public Contracts forman parte de la API estable disponible para aplicaciones.

Ejemplos potenciales:

```text
ConnectionInterface
TransactionInterface
RepositoryInterface
EntityManagerInterface
ResultInterface
SchemaManagerInterface
```

El desarrollador podrá type-hintear estos contratos.

Ejemplo:

```php
final class UserService
{
    public function __construct(
        private ConnectionInterface $connection,
    ) {}
}
```

---

# 10. Extension Contracts

Los Extension Contracts permiten incorporar implementaciones personalizadas.

Ejemplos:

```text
DriverInterface
DialectInterface
DatabasePlatformInterface
DatabaseTypeInterface
OptimizationRuleInterface
QueryCompilerInterface
HydratorInterface
MetadataLoaderInterface
IdentifierGeneratorInterface
```

Estos contratos tendrán requisitos de compatibilidad más estrictos que APIs puramente internas.

---

# 11. Integration Contracts

Los Integration Contracts conectan Database con otros subsistemas.

Ejemplos:

```text
DatabaseCacheInterface
DatabaseTelemetryInterface
DatabaseEventDispatcherInterface
TenantDatabaseContextInterface
RuntimeLifecycleInterface
DatabaseClockInterface
```

Su propósito es evitar dependencias obligatorias.

Ejemplo:

```text
QueryExecutor
      │
      ▼
DatabaseTelemetryInterface
      ▲
      │
QuantumTelemetryAdapter
```

---

# 12. Internal Contracts

Los Internal Contracts separan componentes internos donde una frontera resulta útil pero no debe convertirse en API pública.

Ejemplos:

```text
SemanticAnalyzerInterface
QueryPlannerInterface
PersistencePlannerInterface
ConnectionResetterInterface
ChangeSetComputerInterface
RelationLoadPlannerInterface
```

Pueden evolucionar con mayor libertad.

---

# 13. Niveles de estabilidad

Cada contrato deberá declarar su nivel de estabilidad.

```text
STABLE
EXTENSION
INTERNAL
EXPERIMENTAL
```

## STABLE

Compatibilidad pública.

## EXTENSION

Diseñado para implementaciones externas.

## INTERNAL

Uso exclusivo del framework.

## EXPERIMENTAL

API todavía susceptible a cambios importantes.

---

# 14. Marcadores de API

VoltStack podrá utilizar Attributes o anotaciones documentales como:

```php
#[PublicApi]
interface ConnectionInterface
{
}
```

```php
#[ExtensionApi]
interface DriverInterface
{
}
```

```php
#[Internal]
interface QueryPlannerInterface
{
}
```

Esto permitirá herramientas automáticas de verificación.

---

# 15. Ubicación de contratos

Se evitará un único directorio:

```text
Database/Contracts/
```

con cientos de interfaces.

Se preferirá colocación por dominio.

Ejemplo:

```text
Connection/
├── Contract/
│   ├── ConnectionInterface.php
│   └── ConnectionFactoryInterface.php
└── ...

Driver/
├── Contract/
│   └── DriverInterface.php
└── ...
```

Esto mantiene cohesión.

---

# 16. Excepción para Platform Contracts

Los contratos verdaderamente transversales podrán vivir en:

```text
VoltStack/Platform/Database/
```

porque su propósito es precisamente desacoplar paquetes Quantum.

---

# 17. DatabaseManagerInterface

Responsabilidad:

> Resolver el acceso lógico a las conexiones Database disponibles.

Ejemplo conceptual:

```php
interface DatabaseManagerInterface
{
    public function connection(
        ?string $name = null
    ): ConnectionInterface;
}
```

No deberá exponer:

```text
PDO
UnitOfWork
EntityManager internals
Compiler internals
```

---

# 18. ConnectionInterface

Representará una conexión lógica.

Ejemplo conceptual:

```php
interface ConnectionInterface
{
    public function name(): string;

    public function transaction(
        callable $callback
    ): mixed;
}
```

La API concreta será definida en documentos posteriores.

No debe asumirse:

```text
ConnectionInterface === PDO
```

---

# 19. PhysicalConnection Contract

Si resulta necesaria una abstracción para conexiones físicas, será interna:

```php
PhysicalConnectionInterface
```

Ejemplo:

```text
Logical Connection
       │
       ▼
PhysicalConnectionInterface
       │
       ▼
PDO / Native Handle
```

No deberá formar parte normalmente de Public API.

---

# 20. ConnectionFactoryInterface

Boundary:

```php
interface ConnectionFactoryInterface
{
    public function create(
        ConnectionConfiguration $configuration
    ): ConnectionInterface;
}
```

Su función será construir conexiones.

No resolver nombres ni topologías.

Eso corresponde a otros componentes.

---

# 21. ConnectionResolverInterface

Responsabilidad:

```text
logical name/context
        │
        ▼
connection resolution
```

Ejemplo:

```php
interface ConnectionResolverInterface
{
    public function resolve(
        ConnectionResolutionContext $context
    ): ConnectionInterface;
}
```

---

# 22. DriverInterface

Representa las primitivas específicas del driver.

Conceptualmente:

```php
interface DriverInterface
{
    public function connect(
        ConnectionConfiguration $configuration
    ): PhysicalConnectionInterface;

    public function platform(): DatabasePlatformInterface;
}
```

No deberá incluir métodos ORM.

Prohibido:

```php
$driver->saveEntity($entity);
```

---

# 23. DialectInterface

Responsabilidad:

> Exponer reglas necesarias para representar operaciones según un dialecto.

Ejemplo conceptual:

```php
interface DialectInterface
{
    public function quoteIdentifier(string $identifier): string;
}
```

No deberá ejecutar consultas.

---

# 24. DatabasePlatformInterface

Representará capacidades de la plataforma.

Ejemplo:

```php
interface DatabasePlatformInterface
{
    public function supports(
        DatabaseCapability $capability
    ): bool;
}
```

Esto permite:

```php
if ($platform->supports(DatabaseCapability::RETURNING)) {
    // ...
}
```

en lugar de:

```php
if ($driver === 'pgsql') {
    // ...
}
```

---

# 25. CapabilityProviderInterface

Los componentes que expongan capabilities podrán implementar:

```php
interface CapabilityProviderInterface
{
    public function capabilities(): CapabilitySet;
}
```

Esto permitirá extensiones sin condicionales de proveedor dispersos.

---

# 26. DatabaseTypeInterface

Contrato conceptual:

```php
interface DatabaseTypeInterface
{
    public function name(): string;

    public function toDatabase(
        mixed $value,
        TypeConversionContext $context
    ): mixed;

    public function toPHP(
        mixed $value,
        TypeConversionContext $context
    ): mixed;
}
```

Debe mantenerse independiente del ORM cuando sea posible.

---

# 27. Query Contract Model

No toda Query necesita una interfaz polimórfica tradicional.

Se preferirán modelos inmutables.

Ejemplo:

```text
Query
├── SelectQuery
├── InsertQuery
├── UpdateQuery
└── DeleteQuery
```

podrán implementar:

```php
QueryInterface
```

únicamente si existe comportamiento compartido real.

---

# 28. QueryNodeInterface

Los nodos AST podrán compartir:

```php
interface QueryNodeInterface
{
}
```

como marker contract.

Pero deberán evitarse interfaces artificiales con métodos irrelevantes para todos los nodos.

---

# 29. ExpressionInterface

Boundary estructural:

```php
interface ExpressionInterface extends QueryNodeInterface
{
}
```

Implementaciones:

```text
ColumnExpression
LiteralExpression
ParameterExpression
FunctionExpression
BinaryExpression
SubqueryExpression
```

---

# 30. PredicateInterface

Podrá especializar:

```php
interface PredicateInterface extends ExpressionInterface
{
}
```

con implementaciones:

```text
ComparisonPredicate
AndPredicate
OrPredicate
NotPredicate
ExistsPredicate
NullPredicate
```

---

# 31. QueryBuilderInterface

El Query Builder público podrá disponer de contratos específicos.

Se evitará una mega-interface con:

```text
select
insert
update
delete
join
union
CTE
window
...
```

si ello produce métodos inválidos según el tipo de consulta.

Preferencia:

```text
SelectQueryBuilderInterface
InsertQueryBuilderInterface
UpdateQueryBuilderInterface
DeleteQueryBuilderInterface
```

con contratos comunes pequeños cuando sea útil.

---

# 32. QueryNormalizerInterface

Responsabilidad:

```text
Raw Query Model
      │
      ▼
Normalized Query
```

Contrato interno:

```php
interface QueryNormalizerInterface
{
    public function normalize(
        QueryInterface $query,
        QueryContext $context
    ): QueryInterface;
}
```

---

# 33. QueryValidatorInterface

Responsabilidad:

```text
Query
  │
  ▼
structural validation
```

No debe confundirse con:

```text
Application Validation
```

---

# 34. SemanticAnalyzerInterface

Boundary interno:

```php
interface SemanticAnalyzerInterface
{
    public function analyze(
        QueryInterface $query,
        SemanticContext $context
    ): SemanticQuery;
}
```

Su implementación podrá evolucionar sin convertirse en API pública.

---

# 35. SymbolResolverInterface

Permite resolver:

```text
tables
columns
aliases
relationships
types
```

Ejemplo:

```php
interface SymbolResolverInterface
{
    public function resolve(
        SymbolReference $reference,
        SemanticContext $context
    ): ResolvedSymbol;
}
```

---

# 36. OptimizationRuleInterface

Este será un importante Extension Contract.

```php
interface OptimizationRuleInterface
{
    public function supports(
        OptimizationContext $context
    ): bool;

    public function optimize(
        LogicalQueryPlan $plan,
        OptimizationContext $context
    ): LogicalQueryPlan;
}
```

Permite agregar optimizaciones sin modificar el core.

---

# 37. QueryOptimizerInterface

Contrato interno o extension-stable:

```php
interface QueryOptimizerInterface
{
    public function optimize(
        LogicalQueryPlan $plan,
        OptimizationContext $context
    ): LogicalQueryPlan;
}
```

El optimizer coordina reglas.

No deberá ejecutar consultas.

---

# 38. QueryPlannerInterface

Responsabilidad:

```text
Semantic Query
      │
      ▼
Execution Plan
```

Ejemplo conceptual:

```php
interface QueryPlannerInterface
{
    public function plan(
        SemanticQuery $query,
        PlanningContext $context
    ): ExecutionPlan;
}
```

---

# 39. PlanningStrategyInterface

Podrán existir estrategias intercambiables para decisiones específicas.

Ejemplo:

```text
ConnectionSelectionStrategy
BatchPlanningStrategy
RelationPlanningStrategy
```

No necesariamente todas compartirán una interfaz global.

---

# 40. QueryCompilerInterface

Extension Contract importante:

```php
interface QueryCompilerInterface
{
    public function compile(
        ExecutionPlan $plan,
        CompilationContext $context
    ): CompiledQuery;
}
```

El compilador:

```text
receives structured model
produces executable representation
```

No ejecuta.

---

# 41. CompiledQuery

`CompiledQuery` debería ser preferentemente un Value Object inmutable y no una interfaz.

Ejemplo conceptual:

```php
final readonly class CompiledQuery
{
    public function __construct(
        public string $statement,
        public ParameterSet $parameters,
        public ExecutionMetadata $metadata,
    ) {}
}
```

---

# 42. QueryExecutorInterface

Boundary central:

```php
interface QueryExecutorInterface
{
    public function execute(
        CompiledQuery $query,
        ExecutionContext $context
    ): ResultInterface;
}
```

No deberá conocer Entities.

---

# 43. StatementInterface

Si se requiere abstraer statements preparados:

```php
interface StatementInterface
{
    public function execute(
        ParameterSet $parameters
    ): ResultInterface;
}
```

Este contrato será de bajo nivel.

---

# 44. ResultInterface

Public/Infrastructure Contract:

```php
interface ResultInterface
{
    public function affectedRows(): int;
}
```

Las APIs de lectura podrán especializarse.

---

# 45. ResultSetInterface

Ejemplo:

```php
interface ResultSetInterface extends ResultInterface, \IteratorAggregate
{
}
```

No deberá exponer Entities.

---

# 46. CursorInterface

Contrato para iteración incremental:

```php
interface CursorInterface extends \Iterator
{
    public function close(): void;
}
```

Su lifecycle deberá ser explícito.

---

# 47. TransactionInterface

Representará una transacción activa.

Ejemplo conceptual:

```php
interface TransactionInterface
{
    public function commit(): void;

    public function rollback(): void;

    public function isActive(): bool;
}
```

---

# 48. TransactionManagerInterface

Coordinará transacciones.

```php
interface TransactionManagerInterface
{
    public function transactional(callable $callback): mixed;
}
```

El contrato real deberá considerar:

```text
isolation
retry
nested transactions
savepoints
```

en documentos posteriores.

---

# 49. Savepoint Contract

Savepoints probablemente serán Value Objects/comandos, no necesariamente interfaces.

Ejemplo:

```php
final readonly class Savepoint
{
    public function __construct(
        public string $name,
    ) {}
}
```

---

# 50. SchemaManagerInterface

Public Contract potencial:

```php
interface SchemaManagerInterface
{
    public function inspect(): SchemaMetadata;
}
```

Las operaciones builder podrán vivir en APIs especializadas.

---

# 51. SchemaIntrospectorInterface

Extension/Internal Contract:

```php
interface SchemaIntrospectorInterface
{
    public function introspect(
        ConnectionInterface $connection
    ): SchemaMetadata;
}
```

Implementaciones específicas podrán variar por plataforma.

---

# 52. SchemaCompilerInterface

```php
interface SchemaCompilerInterface
{
    public function compile(
        SchemaPlan $plan,
        SchemaCompilationContext $context
    ): CompiledSchemaOperationSet;
}
```

Debe permanecer separado del QueryCompiler cuando las responsabilidades sean diferentes.

---

# 53. SchemaDifferInterface

Responsabilidad:

```text
Current Schema
     +
Desired Schema
     │
     ▼
Schema Difference
```

Ejemplo:

```php
interface SchemaDifferInterface
{
    public function diff(
        SchemaMetadata $current,
        SchemaMetadata $desired
    ): SchemaDiff;
}
```

---

# 54. MigrationInterface

Las migrations de aplicación podrán implementar:

```php
interface MigrationInterface
{
    public function up(Schema $schema): void;

    public function down(Schema $schema): void;
}
```

La API final podrá usar clases base o contratos adicionales.

---

# 55. MigrationRepositoryInterface

Representa almacenamiento del estado de migrations.

```php
interface MigrationRepositoryInterface
{
    public function applied(): array;

    public function record(MigrationRecord $migration): void;
}
```

Esto permite cambiar la implementación de tracking.

---

# 56. MigrationPlannerInterface

```php
interface MigrationPlannerInterface
{
    public function plan(
        MigrationSet $migrations,
        MigrationContext $context
    ): MigrationPlan;
}
```

---

# 57. EntityManagerInterface

Uno de los principales contratos ORM.

Conceptualmente:

```php
interface EntityManagerInterface
{
    public function find(
        string $entity,
        mixed $id
    ): ?object;

    public function persist(object $entity): void;

    public function remove(object $entity): void;

    public function flush(): void;

    public function clear(): void;
}
```

La API definitiva será especificada posteriormente.

---

# 58. EntityManager no es DatabaseManager

Debe mantenerse:

```text
DatabaseManager
    → connections

EntityManager
    → entity persistence
```

Nunca fusionarlos.

---

# 59. RepositoryInterface

Contrato base:

```php
interface RepositoryInterface
{
    public function find(mixed $id): ?object;
}
```

Podrá utilizar generic annotations para IDEs:

```php
/**
 * @template TEntity of object
 */
interface RepositoryInterface
{
}
```

---

# 60. RepositoryFactoryInterface

Resolverá repositories:

```php
interface RepositoryFactoryInterface
{
    public function create(
        EntityMetadata $metadata,
        EntityManagerInterface $entityManager
    ): RepositoryInterface;
}
```

---

# 61. EntityMetadataProviderInterface

Boundary:

```php
interface EntityMetadataProviderInterface
{
    public function for(
        string $entityClass
    ): EntityMetadata;
}
```

El consumidor no necesita saber si la metadata proviene de:

```text
PHP Attributes
compiled files
cache
conventions
generated code
```

---

# 62. MetadataLoaderInterface

Extension Contract:

```php
interface MetadataLoaderInterface
{
    public function load(
        string $entityClass
    ): ?EntityMetadataDefinition;
}
```

Implementaciones futuras:

```text
AttributeMetadataLoader
CompiledMetadataLoader
ConfigurationMetadataLoader
ConventionMetadataLoader
```

---

# 63. MetadataCompilerInterface

```php
interface MetadataCompilerInterface
{
    public function compile(
        EntityMetadataDefinition $definition
    ): EntityMetadata;
}
```

---

# 64. MetadataCacheInterface

No deberá obligar a utilizar `Quantum/Cache`.

```php
interface MetadataCacheInterface
{
    public function get(string $key): ?EntityMetadata;

    public function put(
        string $key,
        EntityMetadata $metadata
    ): void;
}
```

---

# 65. IdentityMap Contract

`IdentityMap` puede ser una implementación concreta interna si no existe necesidad de sustitución.

Si se requiere boundary:

```php
interface IdentityMapInterface
{
    public function get(
        string $entity,
        mixed $id
    ): ?object;

    public function add(
        string $entity,
        mixed $id,
        object $instance
    ): void;

    public function clear(): void;
}
```

Será internal, no Public API.

---

# 66. UnitOfWork Contract

Igualmente:

```php
interface UnitOfWorkInterface
{
    public function persist(object $entity): void;

    public function remove(object $entity): void;

    public function commit(): void;

    public function clear(): void;
}
```

No deberá exponerse como API cotidiana.

---

# 67. ChangeSetComputerInterface

Boundary interno:

```php
interface ChangeSetComputerInterface
{
    public function compute(
        object $entity,
        EntityMetadata $metadata,
        EntitySnapshot $snapshot
    ): ChangeSet;
}
```

Permite aislar estrategias de change tracking.

---

# 68. PersistenceEngineInterface

```php
interface PersistenceEngineInterface
{
    public function execute(
        PersistencePlan $plan,
        PersistenceContext $context
    ): PersistenceResult;
}
```

No deberá recibir SQL construido por EntityManager.

---

# 69. PersistencePlannerInterface

```php
interface PersistencePlannerInterface
{
    public function plan(
        UnitOfWorkState $state,
        PersistenceContext $context
    ): PersistencePlan;
}
```

Será responsable de:

```text
dependency ordering
insert ordering
update ordering
delete ordering
relationship synchronization
```

---

# 70. HydratorInterface

Extension Contract:

```php
interface HydratorInterface
{
    public function hydrate(
        ResultInterface $result,
        HydrationPlan $plan,
        HydrationContext $context
    ): mixed;
}
```

---

# 71. HydrationStrategyInterface

Podrán existir estrategias:

```text
EntityHydrationStrategy
ArrayHydrationStrategy
ScalarHydrationStrategy
DtoHydrationStrategy
```

No deberán introducir dependencias desde Result hacia ORM.

---

# 72. RelationshipLoaderInterface

Boundary:

```php
interface RelationshipLoaderInterface
{
    public function load(
        RelationshipLoadRequest $request,
        RelationshipContext $context
    ): mixed;
}
```

---

# 73. RelationLoadPlannerInterface

```php
interface RelationLoadPlannerInterface
{
    public function plan(
        RelationshipLoadRequest $request,
        RelationshipContext $context
    ): RelationshipLoadPlan;
}
```

Puede seleccionar:

```text
JOIN
batch query
lazy query
subselect
```

---

# 74. IdentifierGeneratorInterface

Extension Contract:

```php
interface IdentifierGeneratorInterface
{
    public function generate(
        object $entity,
        EntityMetadata $metadata
    ): mixed;
}
```

Implementaciones:

```text
AutoIncrementGenerator
UuidGenerator
UlidGenerator
SequenceGenerator
CustomGenerator
```

---

# 75. DatabaseCacheInterface

Integration Port:

```php
interface DatabaseCacheInterface
{
    public function get(string $key): mixed;

    public function put(
        string $key,
        mixed $value,
        ?int $ttl = null
    ): void;

    public function forget(string $key): void;
}
```

Podrá adaptarse a:

```text
Quantum/Cache
PSR-compatible cache
internal cache
NullCache
```

---

# 76. Cache Contracts especializados

Se evitará utilizar un único cache indiscriminadamente.

Podrán existir:

```text
MetadataCacheInterface
CompiledQueryCacheInterface
QueryPlanCacheInterface
ResultCacheInterface
```

Esto permite semánticas diferentes.

---

# 77. DatabaseTelemetryInterface

Integration Port:

```php
interface DatabaseTelemetryInterface
{
    public function queryStarted(
        QueryTelemetryContext $context
    ): QueryTelemetrySpan;

    public function queryFinished(
        QueryTelemetrySpan $span,
        QueryTelemetryResult $result
    ): void;

    public function queryFailed(
        QueryTelemetrySpan $span,
        \Throwable $error
    ): void;
}
```

La API final podrá simplificarse.

---

# 78. Null Telemetry

Database deberá poder operar con:

```text
NullDatabaseTelemetry
```

cuando `Quantum/Telemetry` no esté instalado.

Esto elimina condicionales repetidos:

```php
if ($telemetry !== null) {
}
```

---

# 79. DatabaseEventDispatcherInterface

Integration Port:

```php
interface DatabaseEventDispatcherInterface
{
    public function dispatch(object $event): void;
}
```

Adapter oficial:

```text
QuantumEventDatabaseAdapter
```

Sin EventSystem:

```text
NullDatabaseEventDispatcher
```

---

# 80. TenantDatabaseContextInterface

Database podrá definir un port neutral:

```php
interface TenantDatabaseContextInterface
{
    public function tenantIdentifier(): mixed;
}
```

La implementación pertenece al paquete Multitenancy.

Database no deberá conocer modelos Tenant concretos.

---

# 81. DatabaseTopologyResolverInterface

Boundary:

```php
interface DatabaseTopologyResolverInterface
{
    public function resolve(
        RoutingContext $context
    ): DatabaseTarget;
}
```

Podrá considerar:

```text
read/write
tenant
shard
replica
consistency
```

---

# 82. ReplicaSelectorInterface

Extension Contract potencial:

```php
interface ReplicaSelectorInterface
{
    public function select(
        ReplicaSet $replicas,
        ReplicaSelectionContext $context
    ): DatabaseTarget;
}
```

Implementaciones:

```text
RoundRobin
Random
Weighted
LatencyAware
LagAware
```

---

# 83. ShardResolverInterface

```php
interface ShardResolverInterface
{
    public function resolve(
        ShardKey $key,
        ShardMap $map
    ): DatabaseTarget;
}
```

Este contrato pertenecerá a capacidades distribuidas avanzadas.

---

# 84. RetryPolicyInterface

Extension/Integration Contract:

```php
interface RetryPolicyInterface
{
    public function shouldRetry(
        DatabaseFailure $failure,
        int $attempt
    ): bool;

    public function delay(
        DatabaseFailure $failure,
        int $attempt
    ): \DateInterval;
}
```

---

# 85. FailureClassifierInterface

```php
interface FailureClassifierInterface
{
    public function classify(
        \Throwable $error
    ): DatabaseFailure;
}
```

Permite normalizar:

```text
deadlock
timeout
connection loss
constraint violation
authentication failure
resource exhaustion
```

---

# 86. ConnectionResetterInterface

Crítico para persistent runtimes.

```php
interface ConnectionResetterInterface
{
    public function reset(
        ConnectionInterface $connection
    ): void;
}
```

Debe poder garantizar:

```text
no active transaction
no stale session state
no leaked locks
no request-specific configuration
```

---

# 87. ResettableInterface

Los componentes stateful request-scoped podrán implementar un contrato interno común:

```php
interface ResettableInterface
{
    public function reset(): void;
}
```

Sin embargo, no debe utilizarse para ocultar lifecycles incorrectos.

Cuando sea posible:

```text
destroy object
```

es preferible a:

```text
manually reset dozens of fields
```

---

# 88. DatabaseContextInterface

Boundary para lifecycle:

```php
interface DatabaseContextInterface
{
    public function id(): DatabaseContextId;
}
```

El contrato no deberá convertirse en una bolsa arbitraria de dependencias.

---

# 89. ExecutionScopeInterface

Permitirá abstraer:

```text
HTTP Request
Queue Job
CLI Command
Scheduled Task
WebSocket Operation
```

Conceptualmente:

```php
interface ExecutionScopeInterface
{
    public function id(): ExecutionScopeId;
}
```

---

# 90. DatabaseLifecycleInterface

Integration Contract:

```php
interface DatabaseLifecycleInterface
{
    public function begin(
        ExecutionScopeInterface $scope
    ): DatabaseContextInterface;

    public function end(
        DatabaseContextInterface $context
    ): void;
}
```

---

# 91. Runtime Adapter Contract

Los adapters de:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

no deberán modificar Database Core.

Podrán utilizar:

```text
DatabaseLifecycleInterface
```

para conectar eventos runtime con lifecycle Database.

---

# 92. Security Integration Contracts

Podrán existir ports específicos:

```text
CredentialProviderInterface
SensitiveDataRedactorInterface
QueryAuditSinkInterface
```

Database no deberá depender de servicios globales de seguridad innecesariamente.

---

# 93. CredentialProviderInterface

Ejemplo:

```php
interface CredentialProviderInterface
{
    public function credentials(
        CredentialReference $reference
    ): DatabaseCredentials;
}
```

Esto permite soportar posteriormente:

```text
environment variables
secret managers
vaults
rotating credentials
cloud identity
```

sin colocar secretos directamente en configuraciones persistentes.

---

# 94. SensitiveDataRedactorInterface

```php
interface SensitiveDataRedactorInterface
{
    public function redact(
        mixed $value,
        RedactionContext $context
    ): mixed;
}
```

Será útil para:

```text
logs
telemetry
exceptions
debug toolbar
query profiler
```

---

# 95. QueryAuditSinkInterface

Podrá recibir eventos auditables de alto nivel.

No deberá confundirse con Telemetry.

```text
Telemetry
    → observability

Audit
    → accountability / security record
```

---

# 96. Clock Abstraction

No todos los componentes deberán llamar directamente:

```php
new DateTimeImmutable();
```

cuando el tiempo sea relevante para lógica comprobable.

Un contrato común de Platform podrá utilizarse si ya existe:

```text
ClockInterface
```

Database deberá reutilizarlo, no crear otro innecesariamente.

---

# 97. Randomness Abstraction

Igualmente, generación de jitter, IDs o selección aleatoria deberá utilizar abstracciones comunes cuando tengan relevancia para testing o seguridad.

No se duplicarán primitivas ya existentes en `Platform`.

---

# 98. Contracts vs Value Objects

No todo debe ser interface.

Se preferirá `final readonly class` para:

```text
ConnectionConfiguration
DatabaseCredentials
QueryParameter
CompiledQuery
QueryFingerprint
ExecutionPlan metadata
DatabaseTarget
ShardKey
DatabaseCapability
ChangeSet
EntityMetadata
```

cuando representan datos con invariantes.

---

# 99. Contracts vs Enums

Se utilizarán enums para conjuntos cerrados.

Ejemplos:

```php
enum TransactionIsolationLevel
{
    case READ_UNCOMMITTED;
    case READ_COMMITTED;
    case REPEATABLE_READ;
    case SERIALIZABLE;
}
```

Otros candidatos:

```text
QueryType
EntityState
DatabaseCapability
ConnectionRole
FailureCategory
```

cuando el conjunto pueda considerarse cerrado.

---

# 100. Contracts vs Abstract Classes

Las clases abstractas podrán utilizarse para compartir implementación cuando exista una jerarquía verdadera.

Ejemplo potencial:

```text
AbstractSqlDialect
 ├── MySQLDialect
 ├── PostgreSQLDialect
 └── SQLiteDialect
```

Pero el consumidor dependerá de:

```text
DialectInterface
```

y no de la clase abstracta.

---

# 101. Interfaces pequeñas

VoltStack preferirá interfaces cohesionadas.

Incorrecto:

```text
DatabaseInterface
 ├── query()
 ├── saveEntity()
 ├── migrate()
 ├── cache()
 ├── telemetry()
 ├── shard()
 ├── backup()
 └── restore()
```

Correcto:

```text
ConnectionInterface
QueryExecutorInterface
EntityManagerInterface
SchemaManagerInterface
TransactionManagerInterface
```

---

# 102. Interface Segregation

Un consumidor sólo deberá conocer los métodos necesarios.

Ejemplo:

Un QueryCompiler necesita:

```text
Dialect capabilities
```

No necesita recibir:

```text
DatabaseManagerInterface
```

si eso le da acceso a todo el sistema.

---

# 103. Dependency Injection

Los contratos deberán inyectarse explícitamente.

Correcto:

```php
final class QueryExecutor
{
    public function __construct(
        private ConnectionResolverInterface $connections,
        private DatabaseTelemetryInterface $telemetry,
    ) {}
}
```

Incorrecto:

```php
final class QueryExecutor
{
    public function __construct(
        private ContainerInterface $container,
    ) {}
}
```

---

# 104. Prohibición de Service Locator

Dentro de Database Core deberá considerarse una violación arquitectónica:

```php
$container->get(ConnectionInterface::class);
```

en clases de dominio o infraestructura ordinaria.

Las excepciones deberán limitarse a bootstrap/factory infrastructure expresamente diseñada.

---

# 105. Factories

Factories serán apropiadas cuando la creación implique:

```text
runtime configuration
implementation selection
lifecycle scope
resource acquisition
complex construction
```

Ejemplos:

```text
ConnectionFactory
EntityManagerFactory
RepositoryFactory
HydratorFactory
```

---

# 106. Resolver vs Factory

La diferencia oficial será:

```text
Factory
    → crea

Resolver
    → selecciona/resuelve

Registry
    → registra/almacena definiciones

Manager
    → coordina
```

Ejemplo:

```text
ConnectionResolver
      │
      ▼
ConnectionFactory
      │
      ▼
Connection
```

---

# 107. Provider

`Provider` deberá utilizarse para suministrar información o recursos bajo demanda.

Ejemplo:

```text
MetadataProvider
CredentialProvider
CapabilityProvider
```

No será sinónimo de Manager.

---

# 108. Port

Un `Port` representa una frontera que el Core necesita consumir sin conocer la implementación.

Ejemplo:

```text
Database Core
     │
     ▼
Telemetry Port
     ▲
     │
Telemetry Adapter
```

---

# 109. Adapter

Un Adapter implementa un Port utilizando otro subsistema o tecnología.

Ejemplos:

```text
QuantumCacheAdapter
QuantumTelemetryAdapter
QuantumEventAdapter
FrankenPHPLifecycleAdapter
```

---

# 110. Ports de salida

Principales outbound ports:

```text
Cache
Telemetry
Events
Credentials
Runtime Lifecycle
Audit
```

Estos permiten que Database invoque capacidades externas.

---

# 111. Ports de entrada

Los inbound ports representan formas estables de utilizar Database.

Ejemplos:

```text
DatabaseManagerInterface
ConnectionInterface
EntityManagerInterface
SchemaManagerInterface
TransactionManagerInterface
```

---

# 112. Arquitectura Ports and Adapters

```text
                       Application
                           │
                           ▼
                     Inbound Ports
                           │
                           ▼
                  ┌─────────────────┐
                  │ Database Core   │
                  └────────┬────────┘
                           │
                     Outbound Ports
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
        Cache          Telemetry          Events
          ▲                ▲                ▲
          │                │                │
       Adapter          Adapter          Adapter
```

---

# 113. Optional Dependency Pattern

Un módulo opcional no deberá provocar:

```php
if (class_exists(Telemetry::class)) {
}
```

por todo el sistema.

Preferencia:

```text
DatabaseTelemetryInterface
       │
       ├── QuantumTelemetryAdapter
       └── NullDatabaseTelemetry
```

La selección ocurre durante bootstrap.

---

# 114. Null Object Pattern

Será apropiado para integraciones sin efecto.

Ejemplos:

```text
NullDatabaseTelemetry
NullDatabaseEventDispatcher
NullResultCache
NullQueryAuditSink
```

Siempre que el Null Object no oculte errores de configuración importantes.

---

# 115. Decorator Pattern

Los contratos permitirán decorators para comportamiento transversal.

Ejemplo:

```text
QueryExecutor
    │
    ▼
TelemetryQueryExecutor
    │
    ▼
RetryingQueryExecutor
    │
    ▼
NativeQueryExecutor
```

Sin embargo, el orden deberá definirse explícitamente.

---

# 116. Pipeline Contracts

Para procesos con múltiples etapas podrán utilizarse pipelines.

Ejemplos:

```text
QueryNormalizationPipeline
OptimizationPipeline
MetadataCompilationPipeline
PersistencePipeline
```

Las stages deberán tener contratos pequeños.

---

# 117. Middleware

El término `Middleware` se reservará principalmente para pipelines donde exista semántica explícita de:

```text
before
next
after
```

No se denominará middleware a cualquier estrategia o regla.

---

# 118. Hook Contracts

Hooks podrán utilizarse en extension points controlados.

Ejemplo:

```text
beforeCompile
afterCompile
beforeFlush
afterFlush
```

Pero los eventos serán preferibles cuando no se requiera alterar el resultado de la operación.

---

# 119. Command Contracts

Operaciones complejas internas podrán representarse mediante comandos inmutables.

Ejemplos:

```text
ExecuteQuery
PersistEntity
RemoveEntity
CreateTable
RunMigration
```

No significa adoptar obligatoriamente CQRS en todo Database.

---

# 120. Query Contracts vs CQRS Query

El término `Query` dentro de Quantum Database significa principalmente:

> Operación lógica contra almacenamiento de datos.

No deberá confundirse automáticamente con el concepto CQRS:

```text
Command / Query Separation
```

---

# 121. Error Contracts

Las APIs públicas deberán arrojar excepciones VoltStack estables.

Ejemplo:

```text
ConnectionInterface
       │
       └── ConnectionException
```

y no filtrar indiscriminadamente:

```text
PDOException
vendor-specific exception
native driver error object
```

---

# 122. Exception Interfaces

Puede existir:

```php
interface DatabaseExceptionInterface extends \Throwable
{
}
```

como marker para capturar errores Database.

Ejemplo:

```php
try {
    // database operation
} catch (DatabaseExceptionInterface $e) {
}
```

---

# 123. Conformance Contracts

Drivers y extensiones oficiales deberán superar suites de conformidad.

Ejemplo:

```text
DriverConformanceSuite
PlatformConformanceSuite
DialectConformanceSuite
TypeConformanceSuite
CompilerConformanceSuite
```

Esto garantizará que implementaciones externas respeten semántica.

---

# 124. Driver Conformance

Todo Driver deberá demostrar:

```text
connect
disconnect
execute
parameter binding
transaction primitives
error normalization
resource cleanup
capability reporting
```

según sus capacidades.

---

# 125. Dialect Conformance

Todo Dialect deberá verificar:

```text
identifier quoting
parameter representation
limit/offset
DDL syntax
supported expressions
platform-specific syntax
```

---

# 126. Platform Conformance

Una Platform deberá reportar capabilities coherentemente.

Ejemplo:

```text
supports(RETURNING)
supports(SAVEPOINTS)
supports(WINDOW_FUNCTIONS)
```

El framework deberá probar que dichas declaraciones coincidan con comportamiento real.

---

# 127. Contract Testing

Un contrato podrá tener una suite abstracta reutilizable.

Conceptualmente:

```php
abstract class DriverContractTest
{
    abstract protected function driver(): DriverInterface;

    // common behavior tests
}
```

Cada driver ejecutará la misma suite.

---

# 128. Fake Implementations

Testing podrá proporcionar:

```text
FakeConnection
FakeDriver
FakeQueryExecutor
FakeTransactionManager
FakeTelemetry
```

cuando sean útiles.

Sin embargo, para Database se deberá favorecer integración real cuando el comportamiento SQL sea importante.

---

# 129. Mocks

Los contratos permitirán mocking, pero la arquitectura no se diseñará exclusivamente para mocks.

El objetivo principal será:

```text
real boundaries
```

no:

```text
everything mockable
```

---

# 130. Contract Versioning

Los contratos `STABLE` seguirán Semantic Versioning.

Cambios como:

```text
remove method
change parameter type
change return type incompatibly
alter required behavior
```

serán breaking changes.

---

# 131. Extension Contract Versioning

Los contratos diseñados para implementación externa requieren cuidado adicional.

Agregar un método abstracto a:

```php
DriverInterface
```

puede romper todos los drivers externos.

Por ello, capacidades nuevas deberán preferir:

```text
capability interfaces
optional subcontracts
new extension interfaces
default adapters
```

cuando sea apropiado.

---

# 132. Capability Interfaces

Para capacidades opcionales podrán existir contratos específicos.

Ejemplo:

```php
interface ReturningCapabilityInterface
{
    public function compileReturning(...): ...;
}
```

cuando sea mejor que ampliar una interfaz central.

Esto evita:

```text
fat interfaces
```

---

# 133. Marker Interfaces

Se utilizarán con moderación.

Ejemplos razonables:

```text
QueryNodeInterface
DatabaseExceptionInterface
ResettableInterface
```

No deberán sustituir un sistema de tipos bien diseñado.

---

# 134. Generic Contracts

PHPDoc generics podrán utilizarse para mejorar IDE y análisis estático.

Ejemplo:

```php
/**
 * @template TEntity of object
 */
interface RepositoryInterface
{
    /**
     * @return TEntity|null
     */
    public function find(mixed $id): ?object;
}
```

VoltStack deberá mantener compatibilidad con herramientas de análisis estático.

---

# 135. Strong Types

Los contratos deberán favorecer tipos de dominio frente a strings ambiguos.

Preferencia:

```php
connection(ConnectionName $name)
```

cuando el valor tenga invariantes significativas.

No necesariamente:

```php
connection(string $name)
```

en todas las capas internas.

La API pública podrá seguir siendo ergonómica.

---

# 136. Public Ergonomics vs Internal Types

La API pública podrá aceptar:

```php
DB::connection('analytics');
```

y normalizar internamente:

```text
"analytics"
     │
     ▼
ConnectionName
```

Esto permite combinar:

```text
Laravel-like DX
+
strong internal model
```

---

# 137. Contract Dependency Rules

Un contrato de bajo nivel no deberá importar tipos de alto nivel.

Incorrecto:

```text
DriverInterface
    │
    ▼
EntityMetadata
```

Correcto:

```text
DriverInterface
    │
    ▼
ConnectionConfiguration
```

---

# 138. Contract Purity

Los contratos fundamentales deberán evitar dependencias accidentales hacia:

```text
HTTP
CLI
Laravel
Symfony
FrankenPHP
RoadRunner
OpenSwoole
```

Las integraciones pertenecen a adapters.

---

# 139. Runtime-Neutral Contracts

Database lifecycle deberá modelarse de forma neutral.

Correcto:

```text
ExecutionScope
```

No:

```text
FrankenPHPRequest
```

dentro del Core.

---

# 140. Framework-Neutral Database Core

Aunque Database pertenece a VoltStack, su motor interno deberá minimizar conocimiento innecesario de:

```text
Routing
Controllers
Views
Authentication
Authorization
```

Esto aumenta testabilidad y reutilización interna.

---

# 141. Service Container Boundary

El Container resolverá implementaciones durante composición.

Ejemplo:

```text
Bootstrap
   │
   ▼
Container
   │
   ├── DriverInterface → Driver
   ├── TelemetryPort → Adapter
   └── CachePort → Adapter
```

Una vez construidos los objetos:

```text
explicit dependencies
```

deberán dominar.

---

# 142. Composition Root

El lugar donde se conectan contratos e implementaciones se denominará conceptualmente:

```text
Database Composition Root
```

Podrá residir en:

```text
DatabaseServiceProvider
DatabaseBootstrapper
DatabaseModule
```

según las convenciones finales de VoltStack.

---

# 143. Composition Root Responsibilities

Será responsable de:

```text
register contracts
select drivers
configure adapters
select null implementations
configure lifecycle scopes
build registries
register extension points
```

No deberá contener lógica de consulta o persistencia.

---

# 144. Contract Ownership

Cada contrato tendrá un módulo propietario.

Ejemplo:

| Contrato | Propietario |
|---|---|
| DriverInterface | Driver |
| ConnectionInterface | Connection |
| QueryCompilerInterface | Compilation |
| QueryExecutorInterface | Execution |
| EntityManagerInterface | ORM |
| MetadataProviderInterface | Metadata |
| HydratorInterface | Hydration |
| RetryPolicyInterface | Resilience |

Esto evita interfaces huérfanas.

---

# 145. Cross-Module Contract Placement

Cuando módulo A consume capacidad de B, normalmente el contrato pertenece al módulo que define la capacidad.

Ejemplo:

```text
Execution
    │
    ▼
ConnectionInterface
```

`ConnectionInterface` pertenece a Connection.

Para outbound integration ports específicos del consumidor podrá aplicarse inversión.

Ejemplo:

```text
DatabaseTelemetryInterface
```

puede pertenecer al boundary Database Telemetry, no a `Quantum/Telemetry`.

---

# 146. Abstraction Leakage

Los contratos públicos no deberán filtrar detalles internos.

Incorrecto:

```php
public function execute(): PDOStatement;
```

Correcto:

```php
public function execute(): ResultInterface;
```

Incorrecto:

```php
public function query(): InternalSelectNode;
```

si `InternalSelectNode` no forma parte de Public API.

---

# 147. Native Escape Hatch

Cuando sea necesario acceder a primitivas nativas, deberá hacerse explícitamente.

Ejemplo conceptual:

```php
$connection->native();
```

o:

```php
$connection->unwrap(PDO::class);
```

La API definitiva se especificará posteriormente.

Debe considerarse:

```text
escape hatch
```

y no base de la arquitectura.

---

# 148. Contract Compatibility

Una implementación deberá respetar:

```text
method signature
return semantics
exceptions
lifecycle
state rules
thread/worker safety requirements
capability declarations
```

Cumplir únicamente la firma PHP no implica conformidad completa.

---

# 149. Behavioral Contracts

Cada contrato importante deberá documentar comportamiento.

Ejemplo:

```text
ConnectionInterface::transaction()
```

deberá definir:

```text
commit behavior
rollback behavior
exception behavior
nested transaction behavior
retry behavior
```

No basta con declarar el método.

---

# 150. Lifecycle Contracts

Los contratos stateful deberán declarar:

```text
creation scope
ownership
cleanup responsibility
reuse policy
reset policy
```

Ejemplo:

```text
CursorInterface
scope: operation
owner: caller/executor
cleanup: explicit close + guaranteed finalization
reuse: forbidden
```

---

# 151. Ownership Contract

Todo recurso deberá tener un owner claro.

Especialmente:

```text
PhysicalConnection
Transaction
Cursor
Stream
Temporary Statement
EntityManager
DatabaseContext
```

Esto evita fugas de recursos.

---

# 152. Concurrency Contract

Los servicios compartidos deberán declarar:

```text
immutable
concurrency-safe
worker-safe
not concurrency-safe
```

Un singleton mutable sin declaración será considerado sospechoso.

---

# 153. Reset Contract

El reset no sustituye ownership correcto.

Prioridad:

```text
1. destroy scoped state
2. release resources
3. reset reusable infrastructure
```

No:

```text
keep everything forever and reset manually
```

---

# 154. Architecture Tests

La suite deberá validar reglas como:

```text
Driver contracts cannot reference ORM
Connection contracts cannot reference EntityManager
Platform contracts cannot reference Quantum
Internal contracts are not exposed publicly
Extension contracts remain dependency-safe
```

---

# 155. Static Analysis

Se recomienda que el proyecto pueda integrar:

```text
PHPStan
Psalm
custom VoltStack architecture rules
```

sin convertir ninguna herramienta externa en requisito conceptual del diseño.

---

# 156. API Documentation

Cada Public o Extension Contract deberá documentar:

```text
purpose
stability
inputs
outputs
exceptions
lifecycle
thread/worker safety
extension expectations
```

---

# 157. Contract Naming

Convención general:

```text
ConnectionInterface
DriverInterface
QueryCompilerInterface
```

para contratos PHP tradicionales.

Sin embargo, se evitarán nombres redundantes cuando el estándar global de VoltStack determine otra convención.

---

# 158. Interface Suffix

Para Database v1 se recomienda conservar:

```text
Interface
```

en contratos PHP explícitos porque:

- facilita distinguir implementación y contrato;
- mejora búsqueda;
- hace visibles los boundaries;
- reduce ambigüedad en un subsistema grande.

La convención global definitiva se documentará en:

```text
331_DATABASE_NAMING_CONVENTIONS.md
```

---

# 159. Contract Map

Mapa resumido:

```text
Application
     │
     ▼
Public Contracts
     │
     ▼
Database Core
     │
     ├──────── Extension Contracts ◄── Third-party extensions
     │
     ├──────── Internal Contracts
     │
     └──────── Integration Ports
                    ▲
                    │
                  Adapters
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     Cache       Telemetry     Events
```

---

# 160. Platform Contract Map

```text
                VoltStack Platform
                       │
                       │ contracts
                       ▼
                Database Contracts
                       ▲
          ┌────────────┼─────────────┐
          │            │             │
          │            │             │
      Database       Jobs         Other Core
      Quantum        Quantum       Systems
```

La implementación sigue perteneciendo a:

```text
Quantum/Database
```

---

# 161. Contratos inicialmente candidatos a Platform

Primera clasificación propuesta:

| Contrato | Ubicación |
|---|---|
| DatabaseManagerInterface | Platform candidate |
| ConnectionInterface | Platform candidate |
| TransactionInterface | Platform candidate |
| DatabaseExceptionInterface | Platform candidate |
| DriverInterface | Quantum/Database |
| DialectInterface | Quantum/Database |
| QueryCompilerInterface | Quantum/Database |
| QueryExecutorInterface | Quantum/Database |
| EntityManagerInterface | Quantum/Database |
| RepositoryInterface | Quantum/Database |
| HydratorInterface | Quantum/Database |
| UnitOfWorkInterface | Internal Database |
| IdentityMapInterface | Internal Database |

---

# 162. Regla de promoción a Platform

Un contrato sólo se moverá a `Platform` si cumple todos o casi todos estos criterios:

```text
multiple framework subsystems need it
implementation-independent
small surface
stable semantics
no Database internal leakage
no optional subsystem dependency
```

No se promoverán contratos por conveniencia.

---

# 163. Contratos que deben permanecer fuera de Platform

Definitivamente:

```text
QueryNodeInterface
ExpressionInterface
OptimizationRuleInterface
SemanticAnalyzerInterface
QueryPlannerInterface
QueryCompilerInterface
HydratorInterface
PersistencePlannerInterface
UnitOfWorkInterface
IdentityMapInterface
```

porque representan internals o extensiones específicas de Database.

---

# 164. Active Record Contracts

La API Active Record no requerirá interfaces para cada Model.

Los Models utilizarán:

```text
Model Base API
      │
      ▼
EntityManagerInterface
```

La extensibilidad deberá concentrarse en el motor ORM, no en contratos artificiales por entidad.

---

# 165. Repository Contracts de aplicación

La aplicación podrá crear:

```php
interface UserRepository
{
    public function findByEmail(string $email): ?User;
}
```

y adaptar una implementación VoltStack.

Esto es independiente de:

```text
VoltStack RepositoryInterface
```

y permite Domain-Driven Design.

---

# 166. Domain Repository Compatibility

Ejemplo:

```text
Application Domain
       │
       ▼
UserRepository
       ▲
       │
VoltStackUserRepository
       │
       ▼
EntityManager
```

VoltStack no obligará al dominio de la aplicación a depender del ORM.

---

# 167. DDD Compatibility

La arquitectura de contratos deberá permitir:

```text
Domain Model
    │
    ▼
Domain Repository Contract
    ▲
    │
Infrastructure Adapter
    │
    ▼
VoltStack Database
```

sin exigir DDD para aplicaciones simples.

---

# 168. Simple Application Compatibility

También deberá permitir:

```php
User::where('active', true)->get();
```

sin requerir:

```text
Domain Repository
Application Service
UnitOfWork manual
```

VoltStack mantendrá ambos niveles.

---

# 169. Abstraction Levels

El sistema podrá utilizarse en distintos niveles:

```text
Level 1
Raw / Native

Level 2
Connection API

Level 3
Query Builder

Level 4
Repository

Level 5
ORM / Active Record

Level 6
Domain Repository / DDD
```

Todos deberán converger hacia la misma infraestructura inferior.

---

# 170. No Parallel Engines

Esta es una regla crítica.

No deberá existir:

```text
QueryBuilder SQL Engine
+
ORM SQL Engine
+
Migration SQL Engine
```

como sistemas completamente separados.

En su lugar:

```text
QueryBuilder ───────┐
ORM ────────────────┼──► Structured Models
Schema/Migration ───┘
                          │
                          ▼
                    Compilation/Execution
```

con compiladores especializados donde corresponda.

---

# 171. Contract Graph

```text
DatabaseManagerInterface
          │
          ▼
ConnectionInterface
          │
          ├────────────► TransactionInterface
          │
          ▼
QueryExecutorInterface
          ▲
          │
QueryCompilerInterface
          ▲
          │
QueryPlannerInterface
          ▲
          │
SemanticAnalyzerInterface
          ▲
          │
Query Model
```

ORM:

```text
EntityManagerInterface
       │
       ├──► MetadataProviderInterface
       ├──► RepositoryFactoryInterface
       ├──► UnitOfWorkInterface
       └──► PersistenceEngineInterface
                     │
                     ▼
               Query Contracts
```

---

# 172. Integration Graph

```text
                     Database Core
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
   CacheInterface  TelemetryInterface EventInterface
          ▲               ▲                ▲
          │               │                │
   Quantum Cache    Quantum Telemetry  Quantum Events
      Adapter           Adapter           Adapter
```

Opcionales:

```text
Multitenancy
Runtime
Audit
Secrets
```

seguirán el mismo patrón.

---

# 173. Contratos y rendimiento

Las abstracciones no deberán introducir overhead significativo en hot paths.

Especialmente:

```text
row hydration
parameter binding
query compilation
result iteration
```

Se evitarán:

```text
unnecessary virtual dispatch
deep decorator chains
container lookups per row
reflection per operation
allocation-heavy context objects
```

Los boundaries arquitectónicos no justifican abstracciones costosas sin necesidad.

---

# 174. Compilación de dependencias

Cuando sea posible, decisiones estables podrán resolverse durante bootstrap.

Ejemplo:

```text
Configuration
      │
      ▼
Composition Root
      │
      ▼
Resolved Driver
Resolved Platform
Resolved Compiler
Resolved Adapters
```

No será necesario resolverlos repetidamente en cada query.

---

# 175. Contract Caching

No se cachean "interfaces".

Se podrán cachear:

```text
resolved implementation maps
compiled metadata
driver capability maps
compiler maps
```

siempre respetando lifecycle.

---

# 176. Security de contratos

Los contratos públicos deberán evitar APIs que faciliten accidentalmente:

```text
unsafe SQL concatenation
credential exposure
cross-tenant access
connection state leakage
```

Las operaciones peligrosas deberán ser explícitas.

---

# 177. Escape Hatch Contract

Una operación raw podrá exponerse mediante API claramente identificada.

Ejemplo:

```php
DB::raw(...)
```

o equivalente.

Su semántica de seguridad deberá documentarse explícitamente.

---

# 178. Tenant-Safe Contracts

Los contratos de conexión deberán permitir que la resolución contextual ocurra antes de obtener el recurso final.

Esto evita:

```text
global default connection
      │
      ▼
tenant query accidentally sent there
```

El Tenant Integration podrá intervenir mediante resolver/topology ports.

---

# 179. Observability-Safe Contracts

Los parámetros sensibles no deberán formar parte obligatoria de objetos de telemetría.

Ejemplo:

```text
QueryTelemetryContext
    ├── fingerprint
    ├── duration
    ├── connection
    └── sanitized metadata
```

Los valores reales sólo se expondrán bajo políticas explícitas de debugging.

---

# 180. Contract Failure Model

Los contratos deberán definir qué ocurre ante fallos.

Ejemplo:

```text
ConnectionFactoryInterface
```

puede producir:

```text
ConfigurationException
ConnectionException
AuthenticationException
```

según la taxonomía final.

---

# 181. Idempotency Contracts

Operaciones donde la idempotencia sea relevante deberán declararla.

Ejemplos:

```text
reset
close
rollback
release
```

Idealmente:

```php
$cursor->close();
$cursor->close();
```

no debería provocar corrupción de estado.

La semántica concreta será especificada por componente.

---

# 182. Cleanup Contracts

Los recursos deberán proporcionar mecanismos explícitos de cleanup cuando corresponda.

```text
CursorInterface::close()
Connection release
Transaction rollback
Context end
```

El framework deberá además proporcionar cleanup defensivo durante lifecycle termination.

---

# 183. Contract Documentation Template

Cada contrato importante deberá documentarse con:

```text
Name
Owner Module
API Classification
Stability
Responsibility
Inputs
Outputs
Exceptions
Lifecycle
Concurrency
Implementations
Extension Rules
Performance Notes
Security Notes
```

---

# 184. Ejemplo de ficha contractual

```text
Contract:
QueryExecutorInterface

Owner:
Execution

Classification:
Internal / Extension candidate

Responsibility:
Execute compiled database operations.

Input:
CompiledQuery
ExecutionContext

Output:
ResultInterface

Must Not Depend On:
ORM
EntityManager
Repository

Lifecycle:
Application-safe if stateless.

Concurrency:
Must not store operation state.

Failure:
ExecutionException hierarchy.
```

---

# 185. Architecture Enforcement

VoltStack deberá crear pruebas automáticas capaces de comprobar:

```text
Platform does not depend on Quantum
Driver does not depend on ORM
Query contracts do not depend on Connection implementations
Result does not depend on Entity
Internal APIs are not accidentally exported
```

---

# 186. Contract Registry

No deberá existir un `ContractRegistry` global salvo necesidad demostrada.

El Container ya representa el composition mechanism principal.

Registries específicos sí son válidos:

```text
DriverRegistry
TypeRegistry
CompilerRegistry
MetadataRegistry
OptimizationRuleRegistry
```

---

# 187. Extension Registry

Los extension points podrán utilizar registries especializados.

Ejemplo:

```text
OptimizationRuleRegistry
    │
    ├── Rule A
    ├── Rule B
    └── Package Rule C
```

La ordenación/prioridad deberá ser determinista.

---

# 188. Priority Contracts

Cuando un extension point admita múltiples implementaciones, podrá utilizar:

```text
priority
phase
capability
supports()
```

en lugar de depender del orden accidental de registro.

---

# 189. Decorator Ordering

Si existen:

```text
Telemetry
Retry
Audit
Timeout
```

alrededor de Execution, el orden será explícito.

Ejemplo conceptual:

```text
Audit
  │
  ▼
Telemetry
  │
  ▼
Retry
  │
  ▼
Timeout
  │
  ▼
Native Executor
```

La semántica exacta se documentará en los subsistemas correspondientes.

---

# 190. Extension Safety

Las extensiones externas no deberán poder modificar estado interno arbitrario.

Se preferirán:

```text
immutable inputs
explicit outputs
restricted contexts
capability-based APIs
```

frente a:

```text
public mutable internals
```

---

# 191. Context Contract Safety

Un `OptimizationContext` no deberá exponer:

```text
Container
EntityManager
Request object
```

si el optimizer no los necesita.

Sólo deberá contener datos relevantes para optimización.

---

# 192. Interface Evolution

Antes de modificar un contrato STABLE o EXTENSION deberá evaluarse:

```text
Can this be a new interface?
Can this be a capability?
Can this be a context field?
Can this use a default implementation?
Is the change actually necessary?
```

Esto reducirá breaking changes.

---

# 193. Deprecated Contracts

La deprecación seguirá una transición:

```text
Active
  │
  ▼
Deprecated
  │
  ▼
Compatibility Period
  │
  ▼
Removed in Major Version
```

La política completa se definirá en:

```text
324_DATABASE_DEPRECATION_POLICY.md
```

---

# 194. Architectural Invariants

### DB-CONTRACT-001

No se creará una interfaz únicamente porque exista una clase.

### DB-CONTRACT-002

Todo contrato tendrá un owner claro.

### DB-CONTRACT-003

Platform Contracts deberán mantenerse mínimos.

### DB-CONTRACT-004

Platform nunca dependerá de Quantum Database.

### DB-CONTRACT-005

Extension Contracts deberán tener semántica documentada y estable.

### DB-CONTRACT-006

Internal Contracts no deberán filtrarse accidentalmente a Public API.

### DB-CONTRACT-007

Outbound integrations deberán utilizar Ports.

### DB-CONTRACT-008

Paquetes opcionales deberán utilizar Adapters.

### DB-CONTRACT-009

El Container no será un Service Locator interno.

### DB-CONTRACT-010

Los contratos de bajo nivel no dependerán de modelos de alto nivel.

### DB-CONTRACT-011

Driver Contracts nunca conocerán Entities.

### DB-CONTRACT-012

Execution Contracts nunca requerirán ORM.

### DB-CONTRACT-013

Result Contracts nunca devolverán Entities directamente.

### DB-CONTRACT-014

Los objetos de datos inmutables deberán preferirse sobre interfaces innecesarias.

### DB-CONTRACT-015

Los recursos stateful deberán documentar lifecycle y ownership.

### DB-CONTRACT-016

Las integraciones opcionales deberán disponer de comportamiento neutral o Null Adapter cuando sea apropiado.

### DB-CONTRACT-017

Los contratos deberán ser runtime-neutral.

### DB-CONTRACT-018

Las abstracciones no deberán comprometer los hot paths de rendimiento.

---

# 195. Arquitectura final de contratos

```text
                         APPLICATION
                             │
                             ▼
                       PUBLIC PORTS
                             │
                             ▼
┌───────────────────────────────────────────────────────────┐
│                    DATABASE CORE                          │
│                                                           │
│ Public Contracts                                          │
│ Internal Contracts                                        │
│ Extension Contracts                                       │
│                                                           │
│ Query │ ORM │ Schema │ Transaction │ Execution            │
└───────────────────────┬───────────────────────────────────┘
                        │
                        ▼
                  OUTBOUND PORTS
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
      Cache          Telemetry          Events
        ▲               ▲                ▲
        │               │                │
     Adapter          Adapter          Adapter
        │               │                │
        ▼               ▼                ▼
 Quantum/Cache   Quantum/Telemetry Quantum/EventSystem
```

Y para dependencias fundamentales:

```text
Quantum/Database
       │
       ▼
Platform Contracts
```

nunca al contrario.

---

# 196. Resultado esperado

El sistema de contratos permitirá que VoltStack Database sea:

```text
modular
replaceable
testable
extensible
runtime-safe
framework-integrated
independently usable internally
```

sin convertirse en una arquitectura dominada por interfaces vacías.

La filosofía será:

```text
Concrete by default.
Abstract at boundaries.
Stable where public.
Flexible where extensible.
Internal where implementation-specific.
```

---

# 197. Regla maestra

La regla central de este documento queda establecida como:

> Depender de abstracciones en las fronteras; depender de modelos concretos e inmutables dentro de los dominios cuando la abstracción no aporte valor.

Esto permite evitar dos extremos:

```text
Monolithic Concrete Architecture
```

y:

```text
Interface-For-Everything Architecture
```

VoltStack se situará entre ambos mediante boundaries explícitos.

---

# 198. Conclusión

`VoltStack/Quantum/Database` utilizará contratos como herramientas arquitectónicas, no como ceremonia.

Las principales fronteras serán:

```text
Application
      │
      ▼
Public Contracts

Database Modules
      │
      ▼
Internal Contracts

Third-Party Extensions
      │
      ▼
Extension Contracts

Optional VoltStack Systems
      │
      ▼
Integration Ports

Database Implementations
      │
      ▼
Platform Contracts
```

La separación permitirá que el sistema evolucione sin recrear el problema que motivó el rediseño de Database: componentes internamente acoplados que dependen circularmente unos de otros.

La arquitectura deberá poder cambiar una implementación sin obligar a modificar las capas que únicamente conocen su contrato.

---

# 199. Siguiente documento

El siguiente documento de la secuencia es:

```text
06_DATABASE_CONFIGURATION_SYSTEM.md
```

Este documento deberá definir el sistema completo de configuración de `Quantum/Database`, incluyendo:

```text
DatabaseConfiguration
ConnectionConfiguration
Driver configuration
Platform configuration
Read/Write configuration
Pool configuration
Runtime configuration
ORM configuration
Migration configuration
Cache integration
Telemetry integration
Environment resolution
Secret references
Configuration normalization
Configuration validation
Compiled configuration
Production configuration cache
Zero-configuration defaults
```

manteniendo la regla:

```text
Raw Framework Configuration
          │
          ▼
Configuration Loader
          │
          ▼
Normalization
          │
          ▼
Validation
          │
          ▼
Typed DatabaseConfiguration
          │
          ▼
Compiled Runtime Configuration
```

de modo que el resto del Database System nunca tenga que depender directamente de arrays globales, variables de entorno o archivos de configuración.