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
| Parcial          |       13 |
| Pendiente        |       17 |
| Total bloques    |       34 |

## Bloques 01-34

| Bloque | Area | Estado | Evidencia visible | Gap principal |
| -----: | ---- | ------ | ----------------- | ------------- |
| 01 | Arquitectura general | Parcial | `src/Quantum/Database/{Config,Runtime,Integration,Contracts,Connection,Driver,Platform,Dialect,Execution,Query,Schema,Migration,Transaction}`, `DatabaseServiceProvider`, `DatabaseCompositionRoot`, `ConnectionManager`, `PdoDriver`, `SqlitePlatform`, `SqliteDialect`, `QueryExecutor`, `StatementExecutor`, `DatabaseQueryManager`, `SchemaManager`, `MigrationRunner`, `TransactionManager`, `config/database.php`, pruebas unit y feature del scope/conexion/ejecucion/query/schema/migration/transaction | Falta endurecer CLI, telemetry y capas no-MVP para salir del corte fundacional |
| 02 | Query Model y AST | Parcial | `QueryType`, `QueryMetadata`, `Select/Insert/Update/DeleteQuery`, `TableReference`, `Predicate`, `Ordering`, AST `Select/Insert/Update/DeleteQueryNode`, `QueryAstFactory`, pruebas `DatabaseQueryCompilerTest` y `DatabaseQueryBuilderExecutionTest` | Falta normalizacion/validacion mas rica, expresiones complejas, joins, aggregates y AST canonica mas profunda |
| 03 | Semantic Query Engine | Pendiente | Sin evidencia suficiente | Falta resolucion semantica, graph y type inference |
| 04 | Query Builder | Parcial | `DatabaseQueryManager`, `SelectQueryBuilder`, API `table(...)->select()->where()->orderBy()->get()/first()/insert()/update()/delete()`, pruebas unit y feature de ejecucion real | Faltan joins, grouping, subqueries, raw expressions controladas, unions y API publica ampliada |
| 05 | Optimizer y Planner | Pendiente | Sin evidencia suficiente | Faltan optimizer, logical plan y execution plan |
| 06 | SQL Compiler | Parcial | `SqlCompiler`, `CompiledQuery`, `QueryCompilerInterface`, quoting via dialect, placeholders seguros, compilacion `SELECT/INSERT/UPDATE/DELETE` hacia `CompiledDatabaseCommand` | Faltan pipeline formal, lowering mas rico, capability validation, source maps y cache de compilacion |
| 07 | Execution Engine | Parcial | `CompiledDatabaseCommand`, `RuntimeBindingSet`, `ExecutionContext`, `ExecutionFailure`, `ExecutionException`, `StatementExecutor`, `QueryExecutor`, `DatabaseResult`, bindings scoped en provider, pruebas `DatabaseExecutionPrimitivesTest` y `DatabaseQueryExecutionTest` | Falta prepared statement lifecycle mas rico, cursores/streaming, timeout/cancellation y clasificacion de errores mas profunda |
| 08 | Schema | Parcial | `ColumnDefinition`, `CreateTableDefinition`, `DropTableDefinition`, `TableBlueprint`, `ColumnBlueprint`, `SchemaCompiler`, `SchemaManager`, pruebas `DatabaseSchemaCompilerTest` y `DatabaseMigrationRunnerTest` | Faltan introspection rica, alter table, indexes, foreign keys, diff y capability model mas profundo |
| 09 | Migrations | Parcial | `MigrationInterface`, `MigrationDiscovery`, `MigrationRepository`, `MigrationRunner`, historial persistente en `quantum_migrations`, apply/rollback basicos desde archivos de migracion | Faltan planner, batches avanzados, safety analysis, checksums, locking y soporte zero-downtime |
| 10 | ORM | Parcial | `src/Quantum/Database/ORM/{Attributes,Contracts,Metadata,Types,EntityKey,EntityState,EntityQuery,EntityRepository,EntityManager,Model}`, extensiones en `DatabaseInterface`, `Database` y `DatabaseServiceProvider`, `TypeRegistry` y handlers base, prueba `DatabaseOrmFeatureTest` validando metadata, `EntityManager`, repository, Model API y round-trip tipado sobre SQLite | Faltan relationships, repository factory con DI, hydration planificada y surface ORM mas rica |
| 11 | Identity Map, Unit of Work y Persistence | Parcial | `IdentityMap`, `UnitOfWork`, `EntityManager::persist/remove/flush/refresh`, snapshots y dirty-check minimo, flush transaccional sobre `TransactionManagerInterface`, prueba `DatabaseOrmFeatureTest` cubriendo identity reuse, persist/update/delete y cleanup por `clear()` | Faltan graph planning, merge/detach avanzados, namespace de identidad, lifecycle callbacks y change tracking mas profundo |
| 12 | Hydration | Parcial | `Metadata/EntityFieldMetadata`, `Metadata/EntityMetadata::hydrate`, `EntityManager::hydrateManaged`, `Types/{TypeRegistry,ScalarTypeHandler,DateTimeImmutableTypeHandler,JsonTypeHandler,BackedEnumTypeHandler}`, casting tipado para scalar/`DateTimeImmutable`/`BackedEnum`/JSON y reuso de instancias gestionadas validado por `DatabaseOrmFeatureTest` | Faltan hydrators compilados, partial loads, relaciones, value objects y caches de hidratacion |
| 13 | Relationships | Pendiente | Sin evidencia suficiente | Faltan mappings, eager/lazy loading y N+1 control |
| 14 | Types, Casting y Value Objects | Parcial | `Attributes/Column` extendido con `type` y `enumType`, `TypeRegistry`, handlers base para scalar/`datetime_immutable`/`json`/`enum`, conversion ida/vuelta integrada en metadata, query y persistencia, prueba `DatabaseOrmFeatureTest` cubriendo enums, fechas y payload JSON | Faltan custom types, value objects multi-columna, registry extensible por aplicacion y converters mas ricos |
| 15 | Transactions y Concurrency | Parcial | `TransactionId`, `TransactionState`, `TransactionContext`, `TransactionException`, `TransactionManager`, nested transactions via savepoints, rollback-only, cleanup automatico al cerrar scope, pruebas `DatabaseTransactionContextTest` y `DatabaseTransactionManagerTest` | Faltan isolation levels, retries, deadlock handling, locking optimista/pesimista, eventos y politicas de concurrencia avanzadas |
| 16 | Read/Write y Distribution | Pendiente | Sin evidencia suficiente | Faltan replicas, routing, sticky policies y failover |
| 17 | Cache | Pendiente | Solo existe `Quantum/Cache` a nivel framework | Falta cache especifico de Database y reglas de consistencia |
| 18 | Factories, Seeders y Fixtures | Pendiente | El skeleton tiene `database/factories` y `database/seeders`, pero no Database engine | Faltan contracts, runner y acople con el subsistema Database |
| 19 | Pagination, Batch y Large Data | Pendiente | Sin evidencia suficiente | Faltan paginators, chunking, cursors y bulk operations |
| 20 | Events | Pendiente | No existe Event System Database propio | Faltan eventos semanticos de query, transaction y persistence |
| 21 | Telemetry y Debugging | Operativo | `Quantum/Database/Telemetry/DatabaseTelemetryEmitter`, instrumentacion en `StatementExecutor`, `TransactionManager` y `MigrationRunner`, integracion con `Quantum/Telemetry`, prueba `DatabaseTelemetryFeatureTest` | Faltan sampling, profiler, diagnostics ampliados y politicas avanzadas de seguridad/cardinality |
| 22 | Security | Pendiente | Solo existen lineamientos documentales | Falta `DatabaseSecurityContext`, policy layer y proteccion de credenciales/queries |
| 23 | Resilience | Pendiente | Sin evidencia suficiente | Faltan retry policies, circuit integration y failure handling |
| 24 | Performance | Pendiente | Sin evidencia suficiente | Faltan budgets, benchmarks, compilation cache y memory controls |
| 25 | Persistent Runtime | Parcial | `DatabaseExecutionScope`, `DatabaseExecutionScopeFactory`, `DatabaseScopeLifecycleManager`, integracion con `onScopeStart/onScopeEnd`, rollback de transacciones abiertas y desconexion de conexiones al cerrar scope, ORM mutable (`IdentityMap`, `UnitOfWork`, `EntityManager`) registrado por scope, `DatabaseRuntimeScopeTest`, `DatabaseSqliteConnectionTest`, `DatabaseTransactionManagerTest`, `DatabaseOrmFeatureTest` | Falta extender ownership a cursores, retries, streaming y politicas avanzadas de limpieza/aislamiento ORM |
| 26 | Multitenancy Integration | Pendiente | Sin evidencia suficiente | Faltan tenant-aware connections, context y schema/database isolation |
| 27 | Advanced Capabilities | Pendiente | Sin evidencia suficiente | Faltan JSON, FTS, temporal, archival y capability system operativo |
| 28 | Backup y Operations | Pendiente | Sin evidencia suficiente | Faltan backup/restore abstractions, diagnostics y maintenance services |
| 29 | Testing | Operativo | `DatabaseConfigurationBindingTest`, `DatabaseConnectionManagerTest`, `DatabaseExecutionPrimitivesTest`, `DatabaseQueryCompilerTest`, `DatabaseSchemaCompilerTest`, `DatabaseTransactionContextTest`, `DatabaseRuntimeScopeTest`, `DatabaseSqliteConnectionTest`, `DatabaseQueryExecutionTest`, `DatabaseQueryBuilderExecutionTest`, `DatabaseMigrationRunnerTest`, `DatabaseTransactionManagerTest`, `DatabasePublicApiFacadeTest`, `DatabaseConsoleCommandsTest`, `DatabaseTelemetryFeatureTest`, `DatabaseOrmFeatureTest`, regresiones verdes del vertical Database incluyendo round-trip tipado ORM | Faltan matriz por drivers adicionales, runtime concurrente, relationships ORM y coverage mas profunda de hydration/value objects |
| 30 | Extensibility | Pendiente | Sin evidencia suficiente | Faltan extension points, manifests y registries de extensiones/plugins |
| 31 | Developer Experience y API Publica | Operativo | `Quantum/Database/Contracts/DatabaseInterface`, `Quantum/Database/Database`, `Quantum/Database/Support/DatabaseStatus`, facades `Quantum/Facades/{DB,Schema}`, entrypoints ORM `entityManager()` y `repository(...)`, `Quantum/Database/ORM/Model`, comandos `database:status`, `database:migrate`, `database:rollback`, pruebas `DatabasePublicApiFacadeTest`, `DatabaseConsoleCommandsTest` y `DatabaseOrmFeatureTest` | Faltan helper limitado explicito, diagnostics/codegen, facade ORM dedicada y ampliacion ergonomica de la API publica |
| 32 | VoltStack Integrations | Operativo | `DatabaseServiceProvider` registrado por defecto en `Application.php`, bindings publicos `DatabaseInterface` y `Database`, servicios ORM (`TypeRegistry`, `EntityMetadataRegistry` singleton; `IdentityMap`, `UnitOfWork`, `EntityManager` scoped), comandos expuestos por provider, integracion HTTP/CLI via `ScopeManager`, telemetria Database sobre `Quantum/Telemetry`, `config/database.php` en skeleton | Faltan integraciones ORM avanzadas para relationships/value objects, runtime no HTTP de larga duracion y operaciones avanzadas |
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
