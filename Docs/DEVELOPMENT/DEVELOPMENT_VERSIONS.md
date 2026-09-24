# DEVELOPMENT_VERSIONS

## Proposito

Esta bitacora registra el avance real del desarrollo del subsistema `Quantum/Database` contra la documentacion oficial ubicada en `vendor/voltstack/database-lab/Docs`.

Sirve como control operativo de:

- la linea base real del subsistema,
- lo ya implementado,
- lo que permanece parcial,
- lo que todavia falta por construir,
- y el siguiente bloque recomendado de ejecucion.

## Corte actual

- Fecha de actualizacion: `2026-09-24`
- Estado general: `Bootstrap, acceso, Execution, Query, Schema, Migrations, Transaction, surface publica minima, ORM minimo y types ORM base de Database implementados`
- Foco del corte: `cerrar DV-DB-009 con type registry ORM, casting extensible minimo y pruebas feature tipadas`

## Versionado de desarrollo

### DV-DB-000

- Estado: `Registrado`
- Bloque documental: `01`, `06`, `07`, `10`, `11`, `12`, `43`, `76`, `87`, `112`, `164`, `216`, `226`, `251`, `252`, `302`, `303`, `309`, `311`, `312`, `313`, `321`, `322`
- Alcance objetivo:
  - fijar una linea base honesta del estado real del subsistema,
  - dejar trazabilidad del hecho de que `Quantum/Database` aun no existe como implementacion operativa,
  - y ordenar el arranque del desarrollo para que el trabajo futuro no empiece por ORM o facades prematuramente.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database` sin implementacion operativa,
  - `vendor/voltstack/database-lab/Docs` con arquitectura extensa y consistente,
  - `Platform/Application.php`, `Quantum/Config`, `Quantum/Container`, `Runtime/Context`, `Quantum/Telemetry` y `Quantum/Console` como habilitadores reutilizables del framework,
  - construccion inicial de:
    - `Docs/DEVELOPMENT/DEVELOPMENT_GUIDELINES.md`
    - `Docs/DEVELOPMENT/DEVELOPMENT_MATRIX.md`
    - `Docs/DEVELOPMENT/DEVELOPMENT_VERSIONS.md`
    - `Docs/DEVELOPMENT/EXECUTIVE_PLAN_IMPLEMENTATION.md`
- Resultado:
  - queda formalizada la diferencia entre arquitectura aspiracional y evidencia real,
  - el subsistema pasa a tener control documental de desarrollo,
  - y se fija una secuencia recomendada para construir `Database V1` desde el nucleo.

## Secuencia de versiones recomendada

Las siguientes entradas representan el orden sugerido de ejecucion. No deben marcarse como implementadas hasta que exista evidencia en codigo y pruebas.

### DV-DB-001

- Estado: `Implementado`
- Bloque documental: `06`, `07`, `251`, `252`, `311`, `312`, `313`, `321`
- Alcance objetivo:
  - introducir `DatabaseServiceProvider`,
  - definir `DatabaseCompositionRoot`,
  - compilar configuracion tipada,
  - crear `DatabaseExecutionScope`,
  - y enlazar el lifecycle con el runtime del framework.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/DatabaseConfigurationProviderInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Config/DatabaseConfiguration.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Config/FrameworkDatabaseConfigurationProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Runtime/DatabaseExecutionScope.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Runtime/DatabaseExecutionScopeFactory.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Runtime/DatabaseContext.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Runtime/DatabaseScopeLifecycleManager.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseCompositionRoot.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
  - `vendor/voltstack/framework/src/Platform/Application.php`
  - `config/database.php`
  - `vendor/voltstack/framework/tests/Unit/DatabaseConfigurationBindingTest.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseRuntimeScopeTest.php`
- Resultado:
  - `Quantum/Database` deja de ser un namespace vacio y pasa a tener base de configuracion, runtime e integracion con el framework,
  - `DatabaseServiceProvider` queda registrado por defecto en `Application`,
  - `DatabaseExecutionScope` se crea por request y se finaliza correctamente al cerrar el scope,
  - y el skeleton ya expone un `config/database.php` minimo para consumir el subsistema.

### DV-DB-002

- Estado: `Implementado`
- Bloque documental: `10`, `11`, `12`, `13-22`
- Alcance objetivo:
  - construir `Driver`, `Connection`, `ConnectionManager`, `Platform` y `Dialect`,
  - soportar al menos una ruta minima operativa de conexion,
  - y dejar el subsistema listo para la frontera de ejecucion.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/{DriverInterface,NativeConnectionInterface,ConnectionInterface,ConnectionManagerInterface,DialectInterface,PlatformInterface}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Connection/{ConnectionDefinition,ConnectionDefinitionRegistry,Connection,ConnectionFactory,ConnectionManager}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Driver/{PdoDriver,PdoNativeConnection,DriverRegistry}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Platform/{PlatformCapabilities,GenericPlatform,SqlitePlatform,PlatformResolver}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Dialect/{GenericDialect,SqliteDialect,DialectResolver}.php`
  - actualizacion de `DatabaseServiceProvider` y `DatabaseScopeLifecycleManager`
  - `vendor/voltstack/framework/tests/Unit/DatabaseConnectionManagerTest.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseSqliteConnectionTest.php`
- Resultado:
  - `Quantum/Database` ya resuelve conexiones logicas compiladas desde configuracion tipada,
  - existe una ruta operativa real sobre `PDO + SQLite`,
  - `ConnectionManager` mantiene la misma instancia dentro del scope,
  - y las conexiones quedan desconectadas al cerrar la request.

### DV-DB-003

- Estado: `Implementado`
- Bloque documental: `76-86`
- Alcance objetivo:
  - abrir el `Execution Engine`,
  - introducir contextos de ejecucion, statement/result y error model,
  - y asegurar cleanup determinista en runtime persistente.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/{QueryExecutorInterface,StatementExecutorInterface}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Execution/{DatabaseResultType,CompiledDatabaseCommand,RuntimeBindingSet,ExecutionContext,ExecutionFailure,ExecutionException,DatabaseResult,StatementExecutor,QueryExecutor}.php`
  - actualizacion de `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
  - `vendor/voltstack/framework/tests/Unit/DatabaseExecutionPrimitivesTest.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseQueryExecutionTest.php`
- Resultado:
  - `Quantum/Database` ya puede ejecutar SQL compilado sobre la capa de conexion existente,
  - `QueryExecutor` y `StatementExecutor` quedan registrados por scope,
  - los bindings runtime se normalizan sin interpolacion SQL,
  - el resultado runtime queda desacoplado del `PDOStatement`,
  - y los errores de ejecucion se propagan mediante un `ExecutionException` con `ExecutionFailure` tipado.

### DV-DB-004

- Estado: `Implementado`
- Bloque documental: `23-75`
- Alcance objetivo:
  - construir Query Model / AST minimo,
  - exponer Query Builder para `select/insert/update/delete` basicos,
  - y conectar builder con compiler y execution.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Query/{QueryType,QueryMetadata,QueryInterface,DatabaseQueryRunner}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Query/Model/{TableReference,Predicate,Ordering,SelectQuery,InsertQuery,UpdateQuery,DeleteQuery}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Query/Ast/{TableNode,PredicateNode,OrderingNode,SelectQueryNode,InsertQueryNode,UpdateQueryNode,DeleteQueryNode,QueryAstFactory}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Query/Compiler/{CompiledQuery,QueryCompilerInterface,SqlCompiler}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Query/Builder/{DatabaseQueryManager,SelectQueryBuilder}.php`
  - actualizacion de `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
  - `vendor/voltstack/framework/tests/Unit/DatabaseQueryCompilerTest.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseQueryBuilderExecutionTest.php`
- Resultado:
  - `Quantum/Database` ya acepta consultas estructuradas sin depender de SQL manual en la capa de entrada,
  - existe un vertical minimo `Query Model -> AST -> Compiler -> Execution`,
  - el builder expone una API inicial tipo `table(...)->where(...)->get()/insert()/update()/delete()`,
  - y la compilacion respeta placeholders separados y quoting por dialecto.

### DV-DB-005

- Estado: `Implementado`
- Bloque documental: `87-111`
- Alcance objetivo:
  - construir Schema Model / Builder minimo,
  - implementar migration repository, discovery y execution,
  - y habilitar CLI inicial de schema/migration.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Schema/Model/{ColumnDefinition,CreateTableDefinition,DropTableDefinition}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Schema/Builder/{ColumnBlueprint,TableBlueprint}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Schema/Compiler/{CompiledSchemaOperation,SchemaCompiler}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Schema/SchemaManager.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Migration/{MigrationInterface,DiscoveredMigration,MigrationDiscovery,MigrationRepository,MigrationRunner}.php`
  - actualizacion de `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
  - `vendor/voltstack/framework/tests/Unit/DatabaseSchemaCompilerTest.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseMigrationRunnerTest.php`
- Resultado:
  - `Quantum/Database` ya puede describir y ejecutar creacion/eliminacion de tablas sobre el mismo runtime Database,
  - existe un repositorio persistente de migraciones en `quantum_migrations`,
  - las migraciones pueden descubrirse desde `database/migrations`, aplicarse y revertirse por batch,
  - y el subsistema deja de depender de SQL manual disperso para el primer flujo de evolucion estructural.

### DV-DB-006

- Estado: `Implementado`
- Bloque documental: `164-175`
- Alcance objetivo:
  - introducir `TransactionManager`,
  - modelar estados transaccionales,
  - y cerrar la integracion con execution, schema y lifecycle.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/TransactionManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Transaction/{TransactionState,TransactionId,TransactionException,TransactionContext,TransactionManager}.php`
  - actualizacion de `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
  - actualizacion de `vendor/voltstack/framework/src/Quantum/Database/Runtime/DatabaseScopeLifecycleManager.php`
  - `vendor/voltstack/framework/tests/Unit/DatabaseTransactionContextTest.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseTransactionManagerTest.php`
- Resultado:
  - `Quantum/Database` ya modela atomicidad de forma explicita mediante `TransactionManager`,
  - existen commit, rollback, rollback-only y nested transactions basadas en savepoints,
  - el cleanup del scope revierte automaticamente transacciones abiertas antes de desconectar conexiones,
  - y Query/Schema/Migrations ya pueden ejecutarse dentro de una frontera transaccional real.

### DV-DB-007

- Estado: `Implementado`
- Bloque documental: `216-226`, `302-309`
- Alcance objetivo:
  - exponer API publica minima,
  - `DB` facade contextual,
  - comandos CLI base,
  - y telemetria inicial del subsistema.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/DatabaseInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Database.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Support/DatabaseStatus.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Telemetry/DatabaseTelemetryEmitter.php`
  - actualizacion de:
    - `vendor/voltstack/framework/src/Quantum/Database/Execution/StatementExecutor.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Transaction/TransactionManager.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Migration/{MigrationRepository,MigrationRunner}.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
    - `vendor/voltstack/framework/src/Quantum/Console/ConsoleApplication.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/{DatabaseStatusCommand,DatabaseMigrateCommand,DatabaseRollbackCommand}.php`
  - `vendor/voltstack/framework/src/Quantum/Facades/{DB,Schema}.php`
  - `vendor/voltstack/framework/tests/Feature/{DatabasePublicApiFacadeTest,DatabaseConsoleCommandsTest,DatabaseTelemetryFeatureTest}.php`
- Resultado:
  - `Quantum/Database` ya expone un servicio publico tipado para `connection/query/table/schema/transaction/migrate/rollback/status`,
  - las facades `DB` y `Schema` resuelven servicios scoped sin introducir estado mutable estatico,
  - el subsistema ya es operable por CLI mediante `database:status`, `database:migrate` y `database:rollback`,
  - la telemetria minima del subsistema ya emite senales de query, transaction y migration usando `Quantum/Telemetry`,
  - y la integracion queda validada con pruebas feature dedicadas y regresiones verdes del vertical Database existente.

### DV-DB-008

- Estado: `Implementado`
- Bloque documental: `112-163`
- Alcance objetivo:
  - abrir ORM minimo,
  - metadata,
  - entity manager,
  - hydration,
  - persistence planning,
  - y base de relationships.
- Evidencia principal:
  - `src/Quantum/Database/ORM`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/{Entity,Table,Id,Column}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Metadata/{EntityFieldMetadata,EntityMetadata,EntityMetadataRegistry}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/{EntityKey,EntityState,IdentityMap,UnitOfWork,EntityQuery,EntityRepository,EntityManager,Model}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Contracts/{EntityManagerInterface,EntityRepositoryInterface}.php`
  - actualizacion de:
    - `vendor/voltstack/framework/src/Quantum/Database/Contracts/DatabaseInterface.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Database.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php`
- Resultado:
  - `Quantum/Database` ya expone un ORM minimo apoyado en el mismo engine de Query, Schema y Transaction existente, sin abrir un runtime paralelo,
  - existe una superficie inicial con atributos `#[Entity]`, `#[Table]`, `#[Id]`, `#[Column]`, metadata registry, `EntityManager`, repository por entidad y `Model` API minima,
  - `IdentityMap` y `UnitOfWork` quedan scoped para mantener seguridad en runtime persistente y aislamiento por request/scope,
  - `flush()` reutiliza `TransactionManagerInterface` y las lecturas/escrituras delegan al `DatabaseQueryManager` ya operativo,
  - y el vertical ORM queda validado con pruebas feature reales sobre SQLite para metadata, persistencia, repository, `Model` e identity reuse.

### DV-DB-009

- Estado: `Implementado`
- Bloque documental: `115-120`
- Alcance objetivo:
  - abrir type registry ORM minimo,
  - extender `#[Column]` con metadata de tipo declarativa,
  - mejorar hydration/persistencia para conversiones tipadas,
  - y validar round-trip tipado real sobre SQLite.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Types/{TypeRegistry,ScalarTypeHandler,DateTimeImmutableTypeHandler,JsonTypeHandler,BackedEnumTypeHandler}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Types/Contracts/TypeHandlerInterface.php`
  - actualizacion de:
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/Column.php`
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Metadata/{EntityFieldMetadata,EntityMetadata,EntityMetadataRegistry}.php`
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/{EntityQuery,EntityManager}.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php`
  - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php`
- Resultado:
  - `Quantum/Database` ya dispone de un `TypeRegistry` ORM minimo para centralizar conversiones en vez de depender de casts ad hoc dispersos,
  - `#[Column]` ahora puede declarar `type` y `enumType`, permitiendo metadata ORM mas expresiva sin abrir todavia el sistema completo de custom types,
  - hydration, criteria de `EntityQuery` y escrituras de persistencia convergen en la misma conversion tipada para scalar, `DateTimeImmutable`, `BackedEnum` y JSON,
  - y el vertical ORM queda validado con pruebas feature reales sobre SQLite para round-trip tipado y regresion del vertical Database existente.

## Estado consolidado del sistema Database

### Ya disponible hoy

1. Documentacion arquitectonica extensa del subsistema.
2. Habilitadores generales del framework:
   - container,
   - config,
   - runtime scope,
   - telemetry,
   - console,
   - service provider base.
3. `Quantum/Database` con base inicial de:
   - configuracion tipada,
   - composition root,
   - execution scope,
   - context,
   - lifecycle manager,
   - service provider.
4. Sistema de desarrollo documental para controlar la construccion futura.
5. Capa inicial de acceso con:
   - `ConnectionDefinition`,
   - `ConnectionManager`,
   - `PdoDriver`,
   - `SqlitePlatform`,
   - `SqliteDialect`,
   - pruebas con SQLite real.
6. Execution Engine minimo con:
   - `CompiledDatabaseCommand`,
   - `RuntimeBindingSet`,
   - `StatementExecutor`,
   - `QueryExecutor`,
   - `DatabaseResult`,
   - `ExecutionException`.
7. Query MVP con:
   - `Select/Insert/Update/DeleteQuery`,
   - AST minimo,
   - `SqlCompiler`,
   - `DatabaseQueryManager`,
   - `SelectQueryBuilder`,
   - ejecucion real por builder sobre SQLite.
8. Schema y Migrations MVP con:
   - `SchemaManager`,
   - `SchemaCompiler`,
   - `TableBlueprint`,
   - `MigrationDiscovery`,
   - `MigrationRepository`,
   - `MigrationRunner`,
   - apply/rollback real sobre SQLite.
9. Transaction MVP con:
   - `TransactionManager`,
   - `TransactionContext`,
   - `TransactionState`,
   - rollback-only,
   - nested savepoints,
   - rollback automatico al cerrar scope.
10. Surface publica minima con:
   - `DatabaseInterface` y `Database`,
   - `DatabaseStatus`,
   - facades `DB` y `Schema`,
   - comandos CLI `database:status`, `database:migrate`, `database:rollback`,
   - telemetria minima de query, transaction y migration.
11. Base de types ORM con:
    - `TypeRegistry`,
    - handlers para scalar, `DateTimeImmutable`, `BackedEnum` y JSON,
    - `#[Column(type: ..., enumType: ...)]`,
    - conversion consistente en hydration, query criteria y writes ORM.

### Parcial o indirectamente disponible

1. Runtime persistente general del framework ya conectado a Database en lifecycle HTTP.
2. Telemetria general del framework reusable y ya consumida por Database en la primera capa de instrumentacion.
3. CLI y bootstrap general del framework ya reutilizados por Database, aunque aun falta ampliar la superficie operativa mas alla del set minimo.

### Aun no desarrollado con evidencia suficiente

1. Relationships, value objects avanzados, hydration planificada y surface ORM ampliada.
2. Security, Resilience, Plugin y Legacy migration runtime.
3. capabilities avanzadas, pagination, batch/streaming y distribucion.

## Siguiente bloque recomendado

### Opcion recomendada posterior

Profundizar el vertical ORM posterior a `DV-DB-009`:

- relationships y relationship loading,
- value objects y custom types mas ricos,
- hydration planificada y caches,
- repository factory con DI,
- y politicas de persistencia mas ricas sobre el mismo engine existente.

Motivo:

- el ORM minimo ya converge sobre `DatabaseQueryManager` y `TransactionManagerInterface`,
- la base scoped (`EntityManager`, `IdentityMap`, `UnitOfWork`) y el type layer minimo ya quedaron validados en runtime real,
- y el siguiente gap estructural dominante ya no es abrir ORM, sino ampliar esa base sin romper el aislamiento del runtime persistente.

## Regla de actualizacion de esta bitacora

Cada nuevo cierre de fase Database debe registrar:

1. un nuevo identificador `DV-DB-00X`,
2. documentos impactados,
3. alcance implementado,
4. evidencia principal,
5. resultado operativo,
6. siguiente gap natural.
