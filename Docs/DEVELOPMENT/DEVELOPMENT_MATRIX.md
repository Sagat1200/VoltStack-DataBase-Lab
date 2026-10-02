# DEVELOPMENT_MATRIX

## Proposito

Esta matriz controla el estado real del subsistema `Quantum/Database` frente a la documentacion arquitectonica ubicada en `vendor/voltstack/database-lab/Docs`.

El criterio de estado en este corte es conservador y se basa en evidencia visible en:

- `vendor/voltstack/framework/src/Quantum/Database`
- `vendor/voltstack/framework/src/Platform/Application.php`
- `vendor/voltstack/framework/src/Quantum/Config`
- `vendor/voltstack/framework/src/Quantum/Container`
- `vendor/voltstack/framework/src/Runtime/Context`
- `vendor/voltstack/framework/src/Quantum/Telemetry`
- `vendor/voltstack/framework/src/Quantum/Console`
- `vendor/voltstack/framework/tests/Unit`
- `vendor/voltstack/framework/tests/Feature`

## Leyenda

- `Operativo`: existe implementacion usable y evidencia directa en tests o integracion real.
- `Parcial`: existe habilitador real del framework o base estructural reutilizable, pero no existe aun cierre funcional del bloque Database.
- `Pendiente`: no hay evidencia suficiente en `Quantum/Database` para considerarlo implementado.

## Nota de alcance

El documento `00_DATABASE_PROJECT_CONTEXT.md` se usa como contexto base y no se contabiliza como bloque de implementacion en esta matriz.

## Resumen del corte

| Estado           | Cantidad |
| ---------------- | -------: |
| Operativo        |        4 |
| Parcial          |       16 |
| Pendiente        |       14 |
| Total bloques    |       34 |

## Bloques 01-34

| Bloque | Area | Estado | Evidencia visible | Gap principal |
| -----: | ---- | ------ | ----------------- | ------------- |
| 01 | Arquitectura general | Parcial | `src/Quantum/Database/{Config,Runtime,Integration,Contracts,Connection,Driver,Platform,Dialect,Execution,Query,Schema,Migration,Transaction}`, `DatabaseServiceProvider`, `DatabaseCompositionRoot`, `ConnectionManager`, `PdoDriver`, `SqlitePlatform`, `SqliteDialect`, `QueryExecutor`, `StatementExecutor`, `DatabaseQueryManager`, `SchemaManager`, `MigrationRunner`, `TransactionManager`, `config/database.php`, pruebas unit y feature del scope/conexion/ejecucion/query/schema/migration/transaction | Falta endurecer CLI, telemetry y capas no-MVP para salir del corte fundacional |
| 02 | Query Model y AST | Parcial | `QueryType`, `QueryMetadata`, `Select/Insert/Update/DeleteQuery`, `TableReference`, `Predicate`, `Ordering`, AST `Select/Insert/Update/DeleteQueryNode`, `QueryAstFactory`, pruebas `DatabaseQueryCompilerTest` y `DatabaseQueryBuilderExecutionTest` | Falta normalizacion/validacion mas rica, expresiones complejas, joins, aggregates y AST canonica mas profunda |
| 03 | Semantic Query Engine | Pendiente | Sin evidencia suficiente | Falta resolucion semantica, graph y type inference |
| 04 | Query Builder | Parcial | `DatabaseQueryManager`, `SelectQueryBuilder`, API `table(...)->as()->select()->join()->leftJoin()->where()->whereIn()->orderBy()->get()/first()/insert()/update()/delete()`, soporte mínimo `IN` + `JOIN` en `SqlCompiler`, pruebas `DatabaseQueryCompilerTest` y `DatabaseQueryBuilderExecutionTest`, y consumo real desde batch preload ORM `EntityQuery::with(...)` | Faltan grouping, subqueries, raw expressions controladas, joins guiados por metadata ORM, unions y API publica ampliada |
| 05 | Optimizer y Planner | Pendiente | Sin evidencia suficiente | Faltan optimizer, logical plan y execution plan |
| 06 | SQL Compiler | Parcial | `SqlCompiler`, `CompiledQuery`, `QueryCompilerInterface`, quoting via dialect, placeholders seguros, compilacion `SELECT/INSERT/UPDATE/DELETE` hacia `CompiledDatabaseCommand`, soporte `FROM ... AS ...`, `INNER JOIN` / `LEFT JOIN` y columnas `table.column AS alias` | Faltan pipeline formal, lowering mas rico, capability validation, source maps, subqueries, joins más ricos y cache de compilacion |
| 07 | Execution Engine | Parcial | `CompiledDatabaseCommand`, `RuntimeBindingSet`, `ExecutionContext`, `ExecutionFailure`, `ExecutionException`, `StatementExecutor`, `QueryExecutor`, `DatabaseResult`, bindings scoped en provider, pruebas `DatabaseExecutionPrimitivesTest` y `DatabaseQueryExecutionTest` | Falta prepared statement lifecycle mas rico, cursores/streaming, timeout/cancellation y clasificacion de errores mas profunda |
| 08 | Schema | Parcial | `ColumnDefinition`, `CreateTableDefinition`, `DropTableDefinition`, `TableBlueprint`, `ColumnBlueprint`, `SchemaCompiler`, `SchemaManager`, pruebas `DatabaseSchemaCompilerTest` y `DatabaseMigrationRunnerTest` | Faltan introspection rica, alter table, indexes, foreign keys, diff y capability model mas profundo |
| 09 | Migrations | Parcial | `MigrationInterface`, `MigrationDiscovery`, `MigrationRepository`, `MigrationRunner`, historial persistente en `quantum_migrations`, apply/rollback basicos desde archivos de migracion | Faltan planner, batches avanzados, safety analysis, checksums, locking y soporte zero-downtime |
| 10 | ORM | Parcial | `src/Quantum/Database/ORM/{Attributes,Contracts,Metadata,Types,EntityKey,EntityState,EntityQuery,EntityRepository,EntityManager,Model,CustomRepositoryRegistry,EntityRepositoryFactory}`; atributos `#[ManyToOne]`, `#[OneToMany]`, `#[OneToOne]`, `#[ManyToMany]`, `#[JoinTable]`, `#[Embedded]`, `#[RepositoryFor]` y 7 lifecycle attrs; `EntityAssociationMetadata` con kinds `one_to_one` y `many_to_many`, shape de join-table y helpers `isToOne/isToMany/isOwningSide/usesJoinTable`; `EntityMetadataRegistry` two-phase con fallback reflection para inverse `mappedBy`; `EntityQuery` añade `with(...associations)`, `select(...)->rows()/firstRow()/pluck()/value()`, `partial(...)->getPartial()/firstPartial()` y `partialManaged(...)->getPartialManaged()/firstPartialManaged()`; `EntityManager` añade `preloadAssociations()` batch, `hydratePartial()`, `hydrateManagedPartial()` y write-path acotado para managed partial; pruebas `DatabaseOrmFeatureTest`, `DVDB014LifecycleCascadeOrphanTest` y `DVDB015ExtendedRelationshipsTest` | Faltan proxies/lazy transparente, joins/fetch strategies declarativas de nivel SQL, asociaciones parciales/partial hydration relacional, hydrators compilados, embedded anidados, manifests extensibles para repository discovery, Criteria API typed, interface bindings por entidad y lifecycle listeners V2 con DI |
| 11 | Identity Map, Unit of Work y Persistence | Parcial | `IdentityMap`, `UnitOfWork`, `EntityManager::persist/remove/flush/refresh`, dirty-check de embedded, flush transaccional sobre `TransactionManagerInterface`, `loadToOne/loadToMany` reutilizando IdentityMap; `UnitOfWork::$originalCollections` cubre asociaciones to-many y nuevo `partialManagedFields` guarda snapshots parciales de fields cargados; `EntityManager` añade batch preload scoped, reconciliación ManyToMany contra DB real, cleanup de join-table al remover entidades, y dirty-check/update limitados al subset loaded en managed partial mode | Faltan graph planning completo, cascadas MERGE/DETACH/REFRESH activas, merge/detach deep, namespace de identidad, change tracking NOTIFY, propagation más rica sobre inverse-side, optimistic lock y aislamiento avanzado de scopes ORM |
| 12 | Hydration | Parcial | `Metadata/EntityFieldMetadata` + `EntityEmbeddedFieldMetadata` implementando `EntityTypedFieldInterface`, `EntityMetadata::hydrate` con estrategia any-non-null para embedded, `EntityManager::hydrateManaged` con `postLoad` real, casting tipado scalar/DateTimeImmutable/BackedEnum/JSON/embedded inner fields, reuso de instancias gestionadas, precarga explícita post-root via `EntityQuery::with(...)`, projection/scalar hydration ORM V1, partial entity hydration detached V1, y partial entity hydration managed V1 con upgrade automático a entidad completa en `find()/refresh()` | Faltan hydrators compilados, joins/eager loading directo en SQL, asociaciones parciales/partial hydration relacional, embedded anidados, caches de hidratación, proxies lazy-transparente y contextos enriquecidos de postLoad para hydration parcial |
| 13 | Relationships | Parcial | atributos `#[ManyToOne]`, `#[OneToMany]`, `#[OneToOne]`, `#[ManyToMany]`, `#[JoinTable]`; `EntityAssociationMetadata` con cascade/orphanRemoval, join-table y helpers owning/inverse/to-one/to-many; `EntityMetadataRegistry::build()` en dos fases shell→final con validación de target y reflection fallback para inverse `mappedBy`; `EntityManager::loadToOne/loadToMany` soporta ManyToOne, OneToMany, OneToOne y ManyToMany; `EntityQuery::with(...)` resuelve esas asociaciones por lotes tras la query root; flush mantiene orden topológico parent→child, FK auto-populate owning-side y persistencia de join rows ManyToMany | Faltan proxies/lazy transparente sobre acceso a propiedad, joins/fetch strategies declarativas en query builder, control sistémico de N+1 más profundo, cascade MERGE/DETACH/REFRESH y orphanRemoval V2 fuera del alcance actual |
| 14 | Types, Casting y Value Objects | Parcial | `Attributes/Column` extendido con `type` y `enumType`, `Attributes/Embedded(class, prefix?)` para value objects multi-columna, `TypeRegistry`, contrato `EntityTypedFieldInterface`, `EntityEmbeddedMetadata` + `EntityEmbeddedFieldMetadata`, handlers base para scalar/`datetime_immutable`/`json`/`enum` aplicables tanto a campos escalares como a inner fields de embedded, prefix strategy default `{prop}_` o explicito, estrategia nullable any-non-null en hydrate, nulificacion completa (todas columnas internas NULL) en extractForWrite, dirty-check mergeado para detectar embedded→NULL, traduccion dotted-path en EntityQuery, prueba `DatabaseOrmFeatureTest` cubriendo enums, fechas, payload JSON y round-trip multi-columna Money/Dimensions + queries anidadas por `price.amount/currency` y `dimensions.width/height/depth` + nulificacion de VO | Faltan custom types user-land, embedded anidados (VO dentro de VO), registry extensible por aplicacion sin tocar providers core, converters mas ricos, y soporte de embedded dentro de OneToMany/ManyToMany target entities (actualmente si se soporta indirectamente al build en dos fases, pero no hay prueba feature que lo afirme) |
| 15 | Transactions y Concurrency | Parcial | `TransactionId`, `TransactionState`, `TransactionContext`, `TransactionException`, `TransactionManager`, nested transactions via savepoints, rollback-only, cleanup automatico al cerrar scope, pruebas `DatabaseTransactionContextTest` y `DatabaseTransactionManagerTest` | Faltan isolation levels, retries, deadlock handling, locking optimista/pesimista, eventos y politicas de concurrencia avanzadas |
| 16 | Read/Write y Distribution | Pendiente | Sin evidencia suficiente | Faltan replicas, routing, sticky policies y failover |
| 17 | Cache | Pendiente | Solo existe `Quantum/Cache` a nivel framework | Falta cache especifico de Database y reglas de consistencia |
| 18 | Factories, Seeders y Fixtures | Parcial | Contracts `FactoryInterface`, `SeederInterface`; `AbstractFactory` base (times/make/create por reflection sin constructor); `FactoryDiscovery` / `SeederDiscovery` con 3 return shapes (instancia, class-string, `callable(Application): X`); `FactoryRegistry` singleton-safe indexado por entityClass; `AbstractSeeder` base con helpers `call()` anidado, `factory($class,?int)` shortcut y `flush()` conveniencia; `SeederRunner` scoped ejecuta seeder dentro de `TransactionManagerInterface::begin()/flush()/commit()` con rollback completo ante Throwable; comando CLI `database:seed --class= --path=` registrado en `DatabaseServiceProvider::commands()`; bindings provider (`FactoryDiscovery`/`FactoryRegistry`/`SeederDiscovery` singletons; `SeederRunner` scoped); cierre estructural bugfix `DatabaseResult` ahora implementa `Countable` + `IteratorAggregate` (corrige count(SELECT) en SQLite PDO rowCount=0); prueba feature `DatabaseFactoriesSeedersFeatureTest` 32 aserciones sobre SQLite temp; regresión verde completa 12 tests / 210 aserciones | Faltan generators CLI `make:factory` / `make:seeder`, factory states/sequences nativos, seeder dependencies DAG ordenado, `SeederRunner` progress logger, fixtures avanzados con transactions y matriz cross-driver para factories/seeders |
| 19 | Pagination, Batch y Large Data | Pendiente | Sin evidencia suficiente | Faltan paginators, chunking, cursors y bulk operations |
| 20 | Events | Parcial | **Sistema de Events V1 dentro del ORM UnitOfWork/EntityManager flush pipeline (no es Event Bus framework-wide todavía): 7 hooks lifecycle canónicos method-level (PrePersist/PostPersist/PreUpdate/PostUpdate/PreRemove/PostRemove/PostLoad) + class-level listeners via EntityLifecycleListenerInterface/AbstractEntityLifecycleListener, dispatch reflection por EntityMetadata.callbacksFor(event) con Closures invocables sobre instancia actual, context typed para PreUpdate/PostUpdate (changes/currentValues/originalSnapshot), Cascade consts PERSIST/REMOVE/ALL, listeners instanciados via `new $className()` V1 (sin Container DI), evidencia en EntityManager::dispatchLifecycle + EntityMetadataRegistry::buildLifecycleCallbacks + UnitOfWork snapshots collectionDiff para OrphanRemoval events implícitos sobre colecciones inverse OneToMany, **Unit `DVDB014LifecycleCascadeOrphanTest` 14 tests validación callbacks y listeners (incluyendo invalid class/exists y context propagation)**, **Feature test 93 assertions sobre persist/hydrate/remove con class listener method static::$calls trackear invocación y context values** | Falta Event Bus global con dispatcher + subscribers fuera del ORM, events semanticos fuera de persistence (query.executed, transaction.committed, migration.applied, connection.opened), publishers async (Queue/Worker), listeners V2 con Container dependency injection reemplazando `new $className()` V1, profiler/debug de event execution timelines, y policies de isolation entre listeners del mismo flush (error en un listener = rollback transaction? policy configurable) |
| 21 | Telemetry y Debugging | Operativo | `Quantum/Database/Telemetry/DatabaseTelemetryEmitter`, instrumentacion en `StatementExecutor`, `TransactionManager` y `MigrationRunner`, integracion con `Quantum/Telemetry`, prueba `DatabaseTelemetryFeatureTest` | Faltan sampling, profiler, diagnostics ampliados y politicas avanzadas de seguridad/cardinality |
| 22 | Security | Pendiente | Solo existen lineamientos documentales | Falta `DatabaseSecurityContext`, policy layer y proteccion de credenciales/queries |
| 23 | Resilience | Pendiente | Sin evidencia suficiente | Faltan retry policies, circuit integration y failure handling |
| 24 | Performance | Pendiente | Sin evidencia suficiente | Faltan budgets, benchmarks, compilation cache y memory controls |
| 25 | Persistent Runtime | Parcial | `DatabaseExecutionScope`, `DatabaseExecutionScopeFactory`, `DatabaseScopeLifecycleManager`, integracion con `onScopeStart/onScopeEnd`, rollback de transacciones abiertas y desconexion de conexiones al cerrar scope, ORM mutable (`IdentityMap`, `UnitOfWork`, `EntityManager`) registrado por scope, `DatabaseRuntimeScopeTest`, `DatabaseSqliteConnectionTest`, `DatabaseTransactionManagerTest`, `DatabaseOrmFeatureTest` | Falta extender ownership a cursores, retries, streaming y politicas avanzadas de limpieza/aislamiento ORM |
| 26 | Multitenancy Integration | Pendiente | Sin evidencia suficiente | Faltan tenant-aware connections, context y schema/database isolation |
| 27 | Advanced Capabilities | Pendiente | Sin evidencia suficiente | Faltan JSON, FTS, temporal, archival y capability system operativo |
| 28 | Backup y Operations | Pendiente | Sin evidencia suficiente | Faltan backup/restore abstractions, diagnostics y maintenance services |
| 29 | Testing | Operativo | `DatabaseConfigurationBindingTest`, `DatabaseConnectionManagerTest`, `DatabaseExecutionPrimitivesTest`, `DatabaseQueryCompilerTest` (incluye joins + aliases), `DatabaseSchemaCompilerTest`, `DatabaseTransactionContextTest`, `DatabaseRuntimeScopeTest`, `DatabaseSqliteConnectionTest`, `DatabaseQueryExecutionTest`, `DatabaseQueryBuilderExecutionTest` (incluye ejecución real de `INNER JOIN` / `LEFT JOIN`), `DatabaseMigrationRunnerTest`, `DatabaseTransactionManagerTest`, `DatabasePublicApiFacadeTest`, `DatabaseConsoleCommandsTest`, `DatabaseTelemetryFeatureTest`, `DatabaseOrmFeatureTest` (entity_manager, model_api, types round-trip, relationships ManyToOne/OneToMany, embedded multi-columna, lifecycle/cascade/orphan, repository factory, OneToOne/ManyToMany, batch preload, projection/scalar hydration, partial detached y partial managed), `DVDB014LifecycleCascadeOrphanTest` (14 tests / 119 assertions) y `DVDB015ExtendedRelationshipsTest` (5 tests / 30 assertions); regresión verde query layer: 5 tests / 25 assertions, regresión verde ORM: 31 tests / 552 assertions | Faltan matriz por drivers adicionales, runtime concurrente, cobertura más profunda de joins ORM relacionales, asociaciones parciales/partial hydration relacional y matriz cross-driver para factories/seeders/generators |
| 30 | Extensibility | Pendiente | Sin evidencia suficiente | Faltan extension points, manifests y registries de extensiones/plugins |
| 31 | Developer Experience y API Publica | Operativo | `Quantum/Database/Contracts/DatabaseInterface`, `Quantum/Database/Database`, `Quantum/Database/Support/DatabaseStatus`, facades `Quantum/Facades/{DB,Schema}`, entrypoints ORM `entityManager()` y `repository(...)`, shortcut `Database::repositoryFactory()`, `Quantum/Database/ORM/Model`, comandos `database:status`, `database:migrate`, `database:rollback`, `database:seed`, atributos públicos `#[ManyToOne]`, `#[OneToMany]`, `#[OneToOne]`, `#[ManyToMany]`, `#[JoinTable]`, `#[Embedded]`, `#[RepositoryFor]`, helpers `EntityManager::loadToOne()` / `loadToMany()`, `EntityQuery::with(...associations)`, projection/scalar hydration `EntityQuery::select(...)->rows()/firstRow()/pluck()/value()`, partial entity hydration detached `EntityQuery::partial(...)->getPartial()/firstPartial()`, partial entity hydration managed `EntityQuery::partialManaged(...)->getPartialManaged()/firstPartialManaged()`, y Query Builder declarativo `table(...)->as()->join()->leftJoin()`, traducción de asociaciones owning to-one y paths embedded en `EntityQuery::where()/orderBy()`, repositorios con `save/delete/count/exists`, y surface Factories/Seeders V1 | Faltan helper limitado explícito, diagnostics/codegen, facade ORM dedicada, joins ORM guiados por metadata, asociaciones parciales/partial hydration relacional, generators CLI `make:factory`/`make:seeder`/`make:repository`, factory states avanzados, manifests de discovery extensibles, interface bindings por entidad y Criteria API typed |
| 32 | VoltStack Integrations | Operativo | `DatabaseServiceProvider` registrado por defecto en `Application.php`, bindings publicos `DatabaseInterface` y `Database`, servicios ORM (`TypeRegistry`, `EntityMetadataRegistry` singleton-safe con metadata de fields/associations/embedded build two-phase; `IdentityMap`, `UnitOfWork`, `EntityManager` scoped), **bindings Repository Factory V1 lifetime-correctos: `CustomRepositoryRegistry` singleton wired a EntityMetadataRegistry constructor 2º param; `EntityRepositoryFactory` scoped depende de EntityManagerInterface scoped actualmente activo; `RepositoryFactoryInterface` bind → EntityRepositoryFactory concrete; `Database` scoped constructor 9º arg recibe RepositoryFactoryInterface**, bindings Factories + Seeders V1 (`FactoryDiscovery`, `FactoryRegistry`, `SeederDiscovery` singletons; `SeederRunner` scoped), comandos expuestos por provider (`database:status`, `database:migrate`, `database:rollback`, `database:seed`), integracion HTTP/CLI via `ScopeManager`, telemetria Database sobre `Quantum/Telemetry`, `config/database.php` en skeleton, convenciones de carpetas `database/factories` y `database/seeders` con discovery auto, `EntityTypedFieldInterface` y `EntityEmbeddedMetadata` singleton-friendly sin estado mutable de proceso, cierre estructural bugfix `DatabaseResult` implementa `Countable` (count() = rows para resultType Rows) + `IteratorAggregate` (foreach sobre rows), **fix SelectQueryBuilder::count() v1 usaba "COUNT(*) AS aggregate" → SqlCompiler quoteIdentifierPath lo envolvía en comillas como identificador → retornaba 0 aun con filas; corregido a count($builder->get()) sobre DatabaseResult Countable cross-driver** | Faltan integraciones ORM avanzadas para proxies lazy-transparente, runtime no HTTP de larga duracion, operaciones avanzadas y extensibilidad de TypeRegistry via manifests/plugins, generators codegen via artisan/volt, manifests/plugins para factories/seeders y manifest discovery extensible para CustomRepositoryRegistry plugins/bounded-contexts separados |
| 33 | Governance y Compatibility | Pendiente | Sin evidencia suficiente | Faltan politicas de compatibilidad, versioning y adapters de migracion |
| 34 | Final Architecture y Migration from Legacy | Pendiente | Solo existe documentacion arquitectonica | Falta toolchain real de analisis, transformacion, shadowing y dual runtime |

## Habilitadores ya presentes en el framework

El framework ya aporta bases reutilizables y ahora cuenta ademas con una base inicial real de `Quantum/Database`:

### 1. Container y lifetimes

- `Quantum/Container/Container.php`
- soporte para `singleton`, `scoped` y `flushScope()`

### 2. Configuracion

- `Quantum/Config/ConfigRepository.php`
- bootstrap general en `Platform/Application.php`

### 3. Runtime persistente

- `Runtime/Context/RuntimeContext.php`
- `Runtime/Context/ScopeManager.php`
- `Runtime/Context/WorkerLifecycle.php`

### 4. Telemetria general y primera capa Database

- `Quantum/Telemetry`
- exporters `null`, `in_memory`, `jsonl`, `http`
- `Quantum/Database/Telemetry/DatabaseTelemetryEmitter`

### 5. Console y bootstrap del framework

- `Quantum/Console`
- `Platform/Application.php`
- patron de `ServiceProvider`
- comandos Database registrados por provider en `ConsoleApplication`

## Orden recomendado de cierre tecnico

1. `01 + 25 + 32`: composition root, scopes e integracion framework.
2. `10-14`: profundizar ORM, hydration, relationships y types partiendo de la base minima ya operativa.
3. `16-20 + 22-24 + 26-34`: distribucion, seguridad, resiliencia, plugins y migracion legacy.

## Regla de mantenimiento

Cada vez que se cierre un bloque relevante del subsistema Database, esta matriz debe actualizar:

1. el estado del bloque impactado,
2. la evidencia visible en codigo/tests,
3. el gap principal restante,
4. la prioridad de trabajo posterior.
