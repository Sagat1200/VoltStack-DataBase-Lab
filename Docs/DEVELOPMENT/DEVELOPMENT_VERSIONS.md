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

- Fecha de actualizacion: `2026-09-27`
- Estado general: `Bootstrap, acceso, Execution, Query, Schema, Migrations, Transaction, surface publica minima, ORM minimo, types ORM base, relaciones ManyToOne/OneToMany bidireccionales V1, value objects embedded multi-columna V1, Factories + Seeders minimo V1 y Repository Factory con DI tipado + helpers ergonomicos V1 de Database implementados`
- Foco del corte: `cerrar DV-DB-013 con contract RepositoryFactoryInterface, atributo #[RepositoryFor], CustomRepositoryRegistry singleton-safe dual (#[RepositoryFor] discovery + register explicito), EntityRepositoryFactory scoped, EntityMetadataRegistry upgrade resolviendo repositoryClass dual-source (#[Entity(repository:X)] primero, CustomRegistry fallback), EntityRepository helpers save/delete/count/exists + accessors tipados, Select/EntityQuery::count aggregator reutilizando DatabaseResult Countable, DatabaseInterface/Database shortcut repositoryFactory(), bindings provider, y feature test 70 aserciones sobre SQLite`

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

### DV-DB-010

- Estado: `Implementado`
- Bloque documental: `115-122`, `116-08-116-88`
- Alcance objetivo:
  - abrir modelo minimo de relaciones bidireccionales ORM,
  - definir atributos `ManyToOne` (lado owning) y `OneToMany` (lado inverse),
  - extender metadata canonica con descriptores de asociacion y mapeos de FK,
  - introducir construccion de metadata en dos fases para evitar recursion en relaciones bidireccionales,
  - integrar helpers de carga (to-one, to-many) en `EntityManager` sin abrir un runtime paralelo,
  - y extender `EntityQuery` con traduccion automatica de filtros por asociacion owning hacia la columna FK correspondiente.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/{ManyToOne,OneToMany}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Metadata/EntityAssociationMetadata.php`
  - actualizacion de:
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Metadata/{EntityMetadata,EntityMetadataRegistry}.php` (lista de asociaciones, acceso `associations()`/`association()`/`hasAssociation()`, build shell→final en dos fases, validacion de target y resolucion de join column)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityManager.php` (helpers `loadToOne`, `loadToMany`, lectura de valor FK desde campo fuente, asignacion de valor en propiedad de asociacion; `metadata` ahora expuesto publicamente para helpers auxiliares)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php` (traduccion `where()` y `orderBy()` por nombre de asociacion owning a columna FK; normalizacion de valor entidad→identifier)
  - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` (nueva prueba `test_bidirectional_many_to_one_and_one_to_many_load_and_query_over_sqlite` con entidades `OrmBlogPost` ↔ `OrmBlogComment`; validacion de metadata, persistencia de FK, consultas raw y via ORM, `loadToOne`, `loadToMany`, IdentityMap, `EntityQuery` con objeto entidad y con ID)
  - regresion en verde de `DatabasePublicApiFacadeTest`, `DatabaseQueryBuilderExecutionTest`, `DatabaseTransactionManagerTest` y `DatabaseOrmFeatureTest` (9 tests, 122 assertions) sobre el suite framework completo.
- Resultado:
  - `Quantum/Database` ahora dispone de un primer modelo minimo de relaciones ORM sin duplicar el engine SQL existente: `ManyToOne` owning y `OneToMany` inverse, con atributos declarativos, metadata canonica, y construccion en dos fases segura ante referencias cruzadas bidireccionales,
  - los usuarios pueden cargar to-one y to-many bajo demanda mediante `EntityManager::loadToOne` y `EntityManager::loadToMany`, reutilizando `IdentityMap` y el `DatabaseQueryManager`/`SelectQueryBuilder` existente,
  - `EntityQuery` acepta de forma ergonomica `where('post', $postObject)` y `where('post', $postId)` traduciendo automaticamente a la columna FK,
  - la persistencia de la columna FK no requiere escritura ORM especial: el usuario sigue usando el campo escalar FK declarado con `#[Column]` y luego puede cargar la asociacion, manteniendo asi el runtime minimo sin un write path paralelo,
  - y el vertical de relaciones queda validado con prueba feature real bidireccional sobre SQLite, regresion verde del ORM y del conjunto Database fundacional.

### DV-DB-011

- Estado: `Implementado`
- Bloque documental: `115-120`, `116 seccion embedded/value objects`
- Alcance objetivo:
  - abrir value objects embedded multi-columna en el ORM sin engine paralelo,
  - definir atributo `#[Embedded(class, prefix?)]` declarativo sobre propiedad entidad,
  - extender metadata canonica con `EntityEmbeddedMetadata` y `EntityEmbeddedFieldMetadata`,
  - abstraer el pipeline de types para aceptar campos escalar entidad y campos de embedded a traves de una interfaz comun,
  - integrar embedded en el write path (`extractForWrite`), snapshot/dirty check (`extract()`) y read path (`hydrate()`),
  - extender `EntityQuery::where/orderBy` con traduccion automatica de nested paths tipo `price.amount`,
  - integrar embedded nullable: si todas las columnas son NULL hydrate devuelve null; si propiedad entidad es null, extractForWrite escribe NULLs multi-columna,
  - y corregir el dirty check de UoW para detectar keys que desaparecen de `$current` vs snapshot al nullificar un embedded.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/Embedded.php`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Metadata/{EntityEmbeddedMetadata,EntityEmbeddedFieldMetadata,EntityTypedFieldInterface}.php`
  - actualizacion de:
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Metadata/{EntityMetadata,EntityMetadataRegistry,EntityFieldMetadata}.php` (lista `embeddeds`, accessors, shell/final build con embeddeds, parsing de `#[Embedded]` + reflexion de inner `#[Column]` fields, implementacion EntityTypedFieldInterface)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Types/Contracts/TypeHandlerInterface.php` (cambio tipado `EntityFieldMetadata` → `EntityTypedFieldInterface`, manteniendo back-compat via implements en EntityFieldMetadata)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Types/{ScalarTypeHandler,BackedEnumTypeHandler,DateTimeImmutableTypeHandler,JsonTypeHandler}.php` (actualizacion signature + uso de `->enumClass()` / `->name()`)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityManager.php` (nueva logica dirty check `flushUpdate` mergeando `$current` y `$original` keys para detectar embedded→NULL y traduccion embedded path)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php` (traduccion nested paths via `resolveEmbeddedPath()` para where() y orderBy())
  - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` (nueva prueba `test_embedded_value_objects_multicolumn_round_trip_and_query_over_sqlite` con fixtures `OrmMoney`, `OrmDimensions`, `OrmProduct`; cobertura de metadata, persistencia multi-columna, lectura raw vs ORM, hydration nullable, querys anidadas `where('price.amount', …)` / `where('price.currency', 'EUR')`, `where('dimensions.depth', '50')`, update con dirty-check tras mutar inner field y nullificar embedded completo)
  - regresion en verde de `DatabasePublicApiFacadeTest`, `DatabaseQueryBuilderExecutionTest`, `DatabaseTransactionManagerTest` y `DatabaseOrmFeatureTest` (10 tests, 172 assertions) sobre el suite framework completo despues del cambio de TypeHandlerInterface.
- Resultado:
  - `Quantum/Database` ORM ya puede modelar value objects multi-columna via atributos declarativos: un objeto PHP (Money{amount,currency}) mapea a multiples columnas (price_amount, price_currency), con prefijo por default inferido `{propiedad}_` o sobreescrito via `#[Embedded(prefix: …)]`,
  - el pipeline de conversion de tipos queda ahora desacoplado: `TypeHandlerInterface` consume `EntityTypedFieldInterface`, permitiendo que los mismos handlers sirvan tanto para `EntityFieldMetadata` (campos entidad) como para `EntityEmbeddedFieldMetadata` (campos embedded inner),
  - write path, read path y dirty check soportan embedded nullable de forma consistente: si el VO es NULL todas las columnas se escriben como NULL; si todas las columnas en row son NULL el VO se hidrata como NULL; al mutar inner fields o nullificar todo el VO, `flushUpdate` detecta el cambio correctamente (incluyendo keys que desaparecen),
  - `EntityQuery` traduce ergonomicamente `where('price.amount', '<', 15000)` y `orderBy('price.amount')` a las columnas reales con conversion de tipos via type handler del campo inner,
  - y el vertical de value objects embedded queda validado con prueba feature real sobre SQLite, regresion verde completa de Database y ORM incluido el cambio estructural de la interfaz TypeHandler.

### DV-DB-012

- Estado: `Implementado`
- Bloque documental: `05 factories/seeders`, `18 Factories & Seeders Fixtures`, `29 Testing`, `31 DevX`, `32 Integrations`
- Alcance objetivo:
  - abrir sistema mínimo de Factories + Seeders para acelerar generación de datos de prueba y seeding determinista,
  - definir contratos `FactoryInterface` y `SeederInterface` públicos,
  - implementar `AbstractFactory` base con `times(int)` inmutable, `make(array overrides)` sin persistir y `create(array overrides)` delegando persist al `EntityManager`,
  - implementar `FactoryDiscovery` que acepta 3 formas de retorno (instancia, class-string, `callable(Application): FactoryInterface`) desde `database/factories/*.php`,
  - implementar `FactoryRegistry` singleton-safe indexado por `entityClass()` con registro explícito y guardia contra duplicados,
  - implementar `AbstractSeeder` base con helpers `call()` anidado, `factory(string $entity, ?int $times)` shortcut y `flush()` conveniencia,
  - implementar `SeederDiscovery` con mismas 3 formas de retorno desde `database/seeders/*.php`,
  - implementar `SeederRunner` (intencionalmente scoped) que resuelve seeder por prioridad `--class` → discovery `DatabaseSeeder` → fallback, ejecuta run() dentro de `TransactionManagerInterface::begin()/flush()/commit()` con rollback completo ante cualquier `Throwable`,
  - exponer comando CLI `database:seed` con opciones `--class` y `--path` ejecutándose dentro de su propio Scope Request idéntico al patrón de los otros 3 comandos DB,
  - registrar bindings en `DatabaseServiceProvider`: singletons `FactoryDiscovery`, `FactoryRegistry`, `SeederDiscovery`; scoped `SeederRunner`; comando `DatabaseSeedCommand` en `commands()`,
  - y corregir bug estructural en `DatabaseResult` que no exponía `Countable` ni `IteratorAggregate`, causando que `count($db->table(...)->get())` retornara 0 aunque rows[] estuviera lleno (PDO SQLite `rowCount()` retorna 0 para SELECTs).
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/FactoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/SeederInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Factories/{AbstractFactory,DiscoveredFactory,FactoryDiscovery,FactoryRegistry}.php`
  - `vendor/voltstack/framework/src/Quantum/Database/Seeders/{AbstractSeeder,DiscoveredSeeder,SeederDiscovery,SeederRunner}.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/DatabaseSeedCommand.php`
  - actualización de `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php` (singletons FactoryDiscovery/FactoryRegistry/SeederDiscovery, scoped SeederRunner, DatabaseSeedCommand en commands())
  - bugfix estructural en `vendor/voltstack/framework/src/Quantum/Database/Execution/DatabaseResult.php`: agrega `implements \Countable, \IteratorAggregate`; `count()` retorna `count($this->rows)` para resultType Rows; `getIterator()` retorna `ArrayIterator($this->rows)`
  - bugfix en `vendor/voltstack/framework/src/Quantum/Database/Seeders/AbstractSeeder.php`: `$application` pasa a `private` con setter `setApplication(Application)` público; `call()` usa setter en lugar de escribir propiedad protegida directamente (evita acceso ilegal desde SeederRunner cuando seeder es clase anónima).
  - `vendor/voltstack/framework/src/Quantum/Database/Seeders/SeederRunner.php` actualizado para usar `$seeder->setApplication($app)`
  - `vendor/voltstack/framework/tests/Feature/DatabaseFactoriesSeedersFeatureTest.php` (prueba end-to-end sobre SQLite temp-dir con Schema real, EntityManager, DbsArticle fixture, factory/seeder discovery via archivos PHP escritos on-the-fly en dirs temp; 32 aserciones que cubren metadata extractForWrite, registry lookup, make sin id, manual persist baseline, `times(5)->create()`, `SeederRunner` via discovery, nested `$this->factory(...)` dentro seeder, consultas EntityQuery con where/orderBy para published/unpublished y repository findAll total).
  - regresión en verde de `DatabasePublicApiFacadeTest`, `DatabaseQueryBuilderExecutionTest`, `DatabaseTransactionManagerTest`, `DatabaseOrmFeatureTest`, `DatabaseConsoleCommandsTest` y `DatabaseFactoriesSeedersFeatureTest` (12 tests, 210 aserciones) sobre el suite framework completo después del fix DatabaseResult.
- Resultado:
  - `Quantum/Database` ahora dispone de una superficie mínima de Factories + Seeders V1 para acelerar productividad del desarrollador sin abrir runtime paralelo: tanto `Factory::create()` como `SeederRunner` delegan todo el write path al `EntityManager` + `TransactionManagerInterface` + `DatabaseQueryManager` ya existentes,
  - `FactoryRegistry` y `SeederDiscovery` son singleton-safe (solo leen archivos + indexan metadata), mientras que `SeederRunner` permanece scoped para depender del `EntityManager` + `TransactionManager` actualmente scoped (seguridad para runtime persistente FrankenPHP/RoadRunner),
  - `database:seed` CLI ya está registrado en `DatabaseServiceProvider::commands()` y ejecuta dentro de su propio Scope recién abierto, idéntico al patrón de status/migrate/rollback,
  - el bugfix de `DatabaseResult` habilitando `Countable` + `IteratorAggregate` es un cierre estructural transversal: cualquier código que usa `count($result)` o `foreach ($result as $row)` sobre el resultado de `$db->table(...)->get()` ahora funciona sin depender de `PDOStatement::rowCount()` (que en SQLite retorna 0 para SELECTs),
  - y el vertical Factories+Seeders V1 queda validado con prueba feature real sobre SQLite temp, incluyendo integración real con el ScopeManager, regresión verde completa sobre todo el conjunto Database + ORM + Console existente.

### DV-DB-013

- Estado: `Implementado`
- Bloque documental: `119 Repositorios`, `302 Public API`, `31 DevX`, `32 Integrations`, `10 ORM`, `11 IdentityMap-UnitOfWork`
- Alcance objetivo:
  - entregar Repository Factory con inyección de dependencias tipada para que servicios y controladores consuman repositorios sin depender del EntityManager directamente,
  - habilitar dos convenciones de binding declarativo: la existente `#[Entity(repository: X)]` sobre entidad y la nueva `#[RepositoryFor(Entity)]` sobre clase repositorio custom,
  - construir un `CustomRepositoryRegistry` singleton-safe que soporte tanto discovery de atributos como registro explícito (para testing/manifiestos),
  - actualizar `EntityMetadataRegistry` con resolución dual-source de repositoryClass (Entity attr primero, CustomRegistry fallback) en ambas fases shell/final,
  - implementar `EntityRepositoryFactory` scoped que delegue 100% a `EntityManager::repository()` preservando la cache única por EM sin duplicación,
  - añadir helpers ergonomicos mínimos a `EntityRepository`: `save(object,flush)`, `delete(object,flush)` con guardia de clase vía RuntimeException, `count(criteria)`, `exists(criteria)` + accessors tipados `getEntityManager/getMetadata/getEntityClass`,
  - añadir aggregator `SelectQueryBuilder::count()` y `EntityQuery::count()` sin abrir engine paralelo (reutiliza `DatabaseResult::count()` Countable desde DV-DB-012),
  - exponer shortcut público `DatabaseInterface::repositoryFactory(): RepositoryFactoryInterface` sobre la fachada `Database`,
  - y registrar bindings lifetime-correctos en `DatabaseServiceProvider`: `CustomRepositoryRegistry` singleton, `EntityRepositoryFactory` scoped, `RepositoryFactoryInterface` bind → concrete, `EntityMetadataRegistry` wired con CustomRegistry, `Database` constructor 9º arg con RepositoryFactory.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Contracts/RepositoryFactoryInterface.php` — contract público: `repositoryFor(string $entityClass): EntityRepositoryInterface`
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/RepositoryFor.php` — atributo `#[Attribute(TARGET_CLASS)]` con `entityClass` FQCN
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/CustomRepositoryRegistry.php` — singleton-safe; dual convetion: auto-discovery `#[RepositoryFor]` + `register(repoClass,?entityClass)` explícito; duplicate guard RuntimeException; class-existence + interface checks
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityRepositoryFactory.php` — scoped factory; 100% delegation a `EntityManagerInterface::repository(entity)` → single cache source
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityRepository.php` (upgrade): accessors `getEntityManager()/getMetadata()/getEntityClass()`; `save(object,flush:bool)` + `delete(object,flush:bool)` con entity-class guard RuntimeException; `count(array $criteria=[]): int` (delegates a EntityQuery count); `exists(array $criteria): bool` via count>0
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php:162-165` — `count(?string $column=null): int` delega a `$this->query->count()`
  - `vendor/voltstack/framework/src/Quantum/Database/Query/Builder/SelectQueryBuilder.php:129-143` — `count(?column): int` mediante builder clonado + wherePredicates transferidos + `count($builder->get())` sobre DatabaseResult Countable (no parallel compiler path)
  - `vendor/voltstack/framework/src/Quantum/Database/ORM/Metadata/EntityMetadataRegistry.php` upgrade: constructor 2º param `?CustomRepositoryRegistry $customRepositories=null`; `build()` shell/final ambos leen `resolveRepositoryClass(entityClass, attrRepo)`; helper privado prefiere `#[Entity(repository: X)]` → fallback `CustomRepositoryRegistry::repositoryFor(entity)`
  - `vendor/voltstack/framework/src/Quantum/Database/Contracts/DatabaseInterface.php:9-37` — import `RepositoryFactoryInterface`; método `repositoryFactory(): RepositoryFactoryInterface`
  - `vendor/voltstack/framework/src/Quantum/Database/Database.php:16-109` — constructor 9º param `RepositoryFactoryInterface $repositoryFactory`; accessor devuelve instancia
  - `vendor/voltstack/framework/src/Quantum/Database/Integration/DatabaseServiceProvider.php:33-42 (imports), 107-112 (singletons), 202-217 (scoped bindings)` — singleton CustomRepositoryRegistry → wired a EntityMetadataRegistry; scoped EntityRepositoryFactory; bind RepositoryFactoryInterface → concrete; Database scoped recibe 9º arg
  - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_repository_factory_with_di_typed_and_ergonomic_helpers` con fixtures `OrmTag` (entity id/name/slug/visible) y `#[RepositoryFor(OrmProduct::class)] OrmProductRepository` custom; 70 aserciones que cubren: CustomRegistry explicit register + metadata assertion, DI RepositoryFactoryInterface resolution, getEntityClass/getEntityManager identity, Database::repositoryFactory() same-obj, default EntityRepository resolution para OrmTag, save() sin-flush + save(flush:true) triple insert, assertNotNull ids, raw `$db->table(...)->count()` = 3, `EntityRepository::count()` = 3, `exists()` true/false, visible criteria counts 2/1, `EntityQuery::where(...)->count()`, findOneBy slug, delete(flush:true), findBy ordered asc name, custom OrmProductRepository via registry → `label()` retorna `sku-based-lookup`
  - fix: `SelectQueryBuilder::count()` v1 usaba `select("COUNT(*) AS aggregate")` que pasaba por SqlCompiler `quoteIdentifierPath()` envolviéndolo en comillas como identificador → retorno 0 sobre filas reales; corregido a `count($builder->get())` sobre DatabaseResult Countable.
  - simplificación: `EntityRepository::count()` eliminado código muerto `is_countable($result)` / `count($query->get())` fallback porque EntityQuery::count() ya retorna int canónico.
  - regresión GREEN de `DatabasePublicApiFacadeTest`, `DatabaseQueryBuilderExecutionTest`, `DatabaseTransactionManagerTest`, `DatabaseOrmFeatureTest`, `DatabaseConsoleCommandsTest`, `DatabaseFactoriesSeedersFeatureTest` y el nuevo test (21 tests, 280 assertions) sobre SQLite.
- Resultado:
  - `Quantum/Database` ORM dispone de una superficie ergonomica de repositorios V1 consumible vía DI tipada: los consumidores declaran `RepositoryFactoryInterface` en constructor y obtienen repos canónicos sin depender directamente de `EntityManager` ni de `Application::make()`,
  - las dos convenciones de binding declarativo coexisten sin colisión: equipos que acoplan entidad↔repo en el mismo bounded context usan `#[Entity(repository:X)]`, equipos con repositorios en módulos separados (sin tocar código fuente de entidad) usan `#[RepositoryFor(Entity)]` o `CustomRepositoryRegistry::register()` explícito,
  - el `EntityRepositoryFactory` permanece estrictamente scoped: cada request/scope de FrankenPHP recibe factory cableada al EntityManager activo, nunca un singleton staled entre fronteras de request; `CustomRepositoryRegistry` (metadata/registros) permanece singleton-safe sin estado mutable de runtime,
  - helpers `save/delete/count/exists` delegan 100% al runtime existente: `persist/remove` en UnitOfWork, `flush` en EntityManager y `count` en SelectQueryBuilder sobre DatabaseResult Countable — NO se abre segundo path SQL ni engine paralelo,
  - `SelectQueryBuilder::count()` queda resuelto sobre el count() nativo de Countable DatabaseResult (fix estructural heredado de DV-DB-012), eliminando riesgos de quoting de expresiones aggregate en SqlCompiler y funciona cross-driver sin depender de dialect-specific SQL,
  - y el vertical Repository Factory + helpers ergonomicos V1 queda validado con prueba feature real sobre SQLite temp (70 aserciones cubriendo resolución, identidad, persistencia, agregación, borrado y ordenación), regresión verde completa sobre 21 tests / 280 assertions en todo el conjunto Database + ORM + Console + FactoriesSeeders.

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
12. Base de relaciones ORM V1 con:
    - atributos `#[ManyToOne]` (owning) y `#[OneToMany]` (inverse),
    - `EntityAssociationMetadata` como descriptor canonico de asociaciones (kind, target, join columns, mappedBy/inversedBy, owning/inverse helpers),
    - `EntityMetadataRegistry::build()` en dos fases (shell → final) para resolver referencias cruzadas bidireccionales sin recursion infinita,
    - helpers `EntityManager::loadToOne()` y `EntityManager::loadToMany()` reutilizando `IdentityMap` + `DatabaseQueryManager`,
    - traduccion automatica en `EntityQuery::where()` / `orderBy()` desde nombre de asociacion owning a columna FK y normalizacion de valor entidad → identifier.
13. Base de value objects embedded V1 con:
    - atributo `#[Embedded(class, prefix?)]` declarativo sobre propiedad de entidad,
    - `EntityEmbeddedMetadata` (VO descriptor con prefix, inner fields, reflection helpers) y `EntityEmbeddedFieldMetadata` (campo inner con conversiones tipadas),
    - `EntityTypedFieldInterface` unificando el contrato de campo tipado para handlers de types,
    - integración multi-columna en `EntityMetadata::extract()`, `extractForWrite()` y `hydrate()` con semantica nullable completa,
    - traduccion nested paths en `EntityQuery::where()` / `orderBy()` via `resolveEmbeddedPath()`,
    - actualizacion coherente de handlers `ScalarTypeHandler`, `BackedEnumTypeHandler`, `DateTimeImmutableTypeHandler`, `JsonTypeHandler`,
    - y dirty check `EntityManager::flushUpdate()` mergeando keys `$current` + `$original` para detectar correctamente embedded→NULL.
14. Base de Factories + Seeders V1 con:
    - contratos públicos `FactoryInterface` (definition, entityClass, times, make, create) y `SeederInterface` (run(Application)),
    - `AbstractFactory` base con `times(int)` inmutable por clone, `make(array)` por reflection sin constructor ni persist, `create(array)` delega `EntityManager::persist()` sin flush implícito,
    - `FactoryDiscovery` / `SeederDiscovery` con 3 return shapes aceptados (instancia, class-string, `callable(Application): X`) desde `database/factories` / `database/seeders`,
    - `FactoryRegistry` singleton-safe indexado por entityClass con `for()`, registro explícito y guardia contra duplicados,
    - `AbstractSeeder` base con helpers `call()` para seeding anidado, `factory($class,?int)` shortcut y `flush()` conveniencia,
    - `SeederRunner` scoped que ejecuta seeder dentro de `TransactionManagerInterface::begin()/flush()/commit()` con rollback completo ante cualquier Throwable,
    - comando CLI `database:seed --class= --path=` registrado en `DatabaseServiceProvider::commands()` ejecutándose dentro de Scope Request propio,
    - bindings en DatabaseServiceProvider: singletons `FactoryDiscovery`, `FactoryRegistry`, `SeederDiscovery`; scoped `SeederRunner`,
    - y cierre estructural bugfix `DatabaseResult` ahora implementa `Countable` + `IteratorAggregate` (count() retorna rows para Rows; getIterator() itera rows).
15. Base de Repository Factory + helpers ergonomicos V1 con:
    - contract público DI-tipado `RepositoryFactoryInterface::repositoryFor(entityClass): EntityRepositoryInterface` para consumidores sin depender de EntityManager directamente,
    - atributo declarativo `#[RepositoryFor(EntityClass::class)]` sobre custom repositorios (independiente del código fuente de la entidad),
    - `CustomRepositoryRegistry` singleton-safe dual-mode: `#[RepositoryFor]` attribute discovery + `register(repoClass,?entityClass)` explícito con duplicate guard y type checks,
    - `EntityMetadataRegistry` upgrade: resolución dual-source `repositoryClass` (prefiere `#[Entity(repository: X)]` existente → fallback `CustomRegistry`), aplicado en ambas fases shell/final,
    - `EntityRepositoryFactory` scoped implementa RepositoryFactoryInterface; 100% delega a `EntityManager::repository()` preservando la cache única interna del EntityManager (no doble cache),
    - helpers ergonomicos en EntityRepository: `save(object,flush)` / `delete(object,flush)` con RuntimeException guard si entidad no coincide con entityClass del repo; accessors `getEntityManager/getMetadata/getEntityClass`; `count(criteria=[])` / `exists(criteria)` via EntityQuery aggregator,
    - aggregators `SelectQueryBuilder::count(?column): int` (mediante builder clonado + wherePredicates transfer + `count(DatabaseResult)` sobre Countable) y `EntityQuery::count(?column): int` (forwarding al SelectQueryBuilder subyacente),
    - surface público Database: `DatabaseInterface::repositoryFactory(): RepositoryFactoryInterface` + `Database` con noveno constructor param y accessor,
    - bindings lifetime discipline en `DatabaseServiceProvider`: `CustomRepositoryRegistry` singleton, `EntityRepositoryFactory` scoped, `RepositoryFactoryInterface` bind → concrete, `EntityMetadataRegistry` wired con CustomRegistry, `Database` scoped constructor recibe RepositoryFactory.

### Parcial o indirectamente disponible

1. Runtime persistente general del framework ya conectado a Database en lifecycle HTTP.
2. Telemetria general del framework reusable y ya consumida por Database en la primera capa de instrumentacion.
3. CLI y bootstrap general del framework ya reutilizados por Database con 5 comandos operativos (`database:status`, `database:migrate`, `database:rollback`, `database:seed`, aunque aun falta ampliar la superficie: comandos `make:factory`, `make:seeder`, `authz:manifest:*` análogos DB, seeds avanzados con DAG dependencias, repositories codegen, etc.).
4. Base de relaciones ManyToOne/OneToMany ya operativa pero todavia sin proxies, lazy transparente, joins en SQL, cascadas, orphan removal ni relationships de tipo OneToOne/ManyToMany.
5. Base de value objects embedded ya operativa pero todavia sin nested embedded (embedded dentro de embedded), sin embedded en relationships, sin embedded collection/JSON, sin equals/hashCode por valor, y sin soporte en repository `findBy()` shortcuts para paths anidados (aunque `EntityQuery` si la soporta).
6. Repository DI tipado ya operativo pero todavía sin manifests extensibles para discovery en módulos separados, sin interface bindings por entidad (ej `bind(OrmProductRepositoryInterface::class → concrete)`) y sin helpers `findByXxx()` mágicos ni Criteria API rich.

### Aun no desarrollado con evidencia suficiente

1. Hydration planificada/compilada, caches de metadata y surface ORM ampliada.
2. Relationships ampliados (OneToOne, ManyToMany, join tables, proxies/lazy transparente, eager joins, cascadas).
3. Value objects avanzados: nested embedded, embedded collection via JSON, value identity/equality helpers.
4. Security, Resilience, Plugin y Legacy migration runtime.
5. Capabilities avanzadas, pagination, batch/streaming y distribucion.

## Siguiente bloque recomendado

### Opcion recomendada posterior

Profundizar el vertical ORM posterior a `DV-DB-013`, en el siguiente orden natural:

1. **Lifecycle callbacks/events + cascade persist/remove mínimo + orphan removal básico**: política de persistencia sobre el mismo `UnitOfWork::flush()`, alineado con el pipeline ya existente. (bloques 11/ORM, 12/Hydration, 20/Events)
2. **Relationships ampliados**: OneToOne bidireccional, ManyToMany con join-table, y opcionalmente estrategias EAGER JOIN declarativas sobre el mismo `SelectQueryBuilder`. (bloques 10/ORM, 13/Relationships, 04/Query Builder joins)
3. **Factories & Seeders ampliados**: comandos generators `make:factory`, `make:seeder`, factory states/sequences nativos, soporte para seeder dependencias ordenado, y SeederRunner con progress/logger. (Prioridad 7, bloque 18 Factories)
4. **Repositories avanzados**: manifests extensibles para discovery en módulos, interfaces por entidad + container bindings, Criteria API typed, y codegen helpers.

Motivo:

- la base ORM ya convergió sobre `DatabaseQueryManager` + `TransactionManagerInterface`, incluyendo Types V1, Relaciones V1, Embedded V1, Factories + Seeders V1 y ahora Repository Factory DI + helpers V1,
- la vida scoped (`EntityManager`, `IdentityMap`, `UnitOfWork`, `SeederRunner`, `EntityRepositoryFactory`) y seguridad para runtime persistente siguen intactas,
- y el siguiente gap estructural dominante ya no es abrir ergonomía diaria (hecho con save/delete/count/exists + DI tipado): **sino cerrar policy layer de persistencia** (Lifecycle callbacks + Cascade/OrphanRemoval mínimo) **antes** de saltar a relaciones más ricas o generators CLI.

## Regla de actualizacion de esta bitacora

Cada nuevo cierre de fase Database debe registrar:

1. un nuevo identificador `DV-DB-00X`,
2. documentos impactados,
3. alcance implementado,
4. evidencia principal,
5. resultado operativo,
6. siguiente gap natural.
