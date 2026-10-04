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

- Fecha de actualizacion: `2026-10-04`
- Estado general: `Bootstrap, acceso, Execution, Query, Schema, Migrations, Transaction, surface publica minima, Query Builder con joins declarativos V1, SQL Compiler con JOIN/alias support V1, ORM minimo, joins ORM guiados por metadata V1 para asociaciones to-one y to-many en querying/proyección, eager hydration joined de resultados de entidad para relaciones to-one y colecciones to-many via get(), types ORM base, relaciones ManyToOne/OneToMany bidireccionales V1, value objects embedded multi-columna V1, Factories + Seeders minimo V1, Repository Factory con DI tipado + helpers ergonomicos V1, ORM Lifecycle + Cascade + OrphanRemoval V1, Relationships Ampliados V1 (OneToOne bidireccional + ManyToMany con JoinTable declarativo), explicit batch preloading V1 via EntityQuery::with(...), projection/scalar hydration ORM V1, partial entity hydration explícita V1 (detached + refreshable), y partial entity hydration managed V1 implementados`
- Foco del corte: `cerrar DV-DB-023 con joins ORM to-many V1 en EntityQuery::join()/leftJoin(), soporte OneToMany y ManyToMany, deduplicación de roots en get(), materialización de colecciones sin duplicados, count() root-aware, resincronización de snapshots de colección y guardrail explícito para first() sobre joins to-many, nueva feature ORM 16 assertions y regresión vertical ORM green (34 tests/586 assertions)`

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

### DV-DB-014

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `11 IdentityMap-UnitOfWork / Persistence Policies`, `20 Events`
- Alcance objetivo:
  - cerrar la capa de `Persistence Policies` del ORM UnitOfWork V1 mediante 7 hooks lifecycle canónicos (PrePersist / PostPersist / PreUpdate / PostUpdate / PreRemove / PostRemove / PostLoad),
  - entregar `Cascade` mínimo para operaciones `PERSIST` y `REMOVE` sobre asociaciones ManyToOne y OneToMany (V1: implementación activa; MERGE/DETACH/REFRESH aceptados como constantes pero sin wireado en flush),
  - habilitar `orphanRemoval` básico sólo en asociaciones inversas `OneToMany` con restricción fuerte: opera ÚNICAMENTE sobre entidades `Managed` después de un flush (no afecta entidades `New`),
  - upgrade el pipeline `EntityManager::flush()` para: (a) aplicar cascades BFS anti-circular antes de insert/update/delete, (b) recolectar orphans de colecciones inversas via snapshot+diff, (c) insertar entidades `NEW` en orden topológico (parent antes que child) mediante stall guard sin Kahn paralelo, (d) auto-popular valores FK del lado `ManyToOne owning-side` directamente desde referencias PHP ya persistidas,
  - mantener 100% backward compatibility: entidades sin atributos nuevos se comportan idénticamente a pre-014, SIN ampliar constructor EntityManager (se mantienen 5 args originales), SIN agregar bindings nuevos en DatabaseServiceProvider, SIN estado global mutable sobre metadata singleton,
  - y validar el cierre con suite Unit especializada ≥ 12 tests + subtest Feature ≥ 60 assertions sobre SQLite temp, junto a regresión vertical Database ORM existente.
- Evidencia principal:
  - **Contracts y atributos lifecycle:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Contracts/Cascade.php` — constantes string `PERSIST/REMOVE/MERGE/DETACH/REFRESH/ALL` (V1: PERSIST y REMOVE activos en flush, MERGE/DETACH/REFRESH como no-ops)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Contracts/EntityLifecycleListenerInterface.php` — contract class-level con 7 métodos abstract (prePersist/postPersist/preUpdate/postUpdate/preRemove/postRemove/postLoad)
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Contracts/AbstractEntityLifecycleListener.php` — conveniencia: implementa la interface con 7 cuerpos vacíos por defecto, para que consumidores sólo sobreescriban los hooks que necesitan
    - 7 atributos method-level nuevos: `Attributes/PrePersist.php`, `PostPersist.php`, `PreUpdate.php`, `PostUpdate.php`, `PreRemove.php`, `PostRemove.php`, `PostLoad.php` — todos `#[Attribute(TARGET_METHOD)]`
  - **Atributos Entity/ManyToOne/OneToMany upgrade:**
    - `Attributes/Entity.php` — nuevo named arg `array $lifecycleListeners = []` (lista FQCN clases listener class-level)
    - `Attributes/ManyToOne.php` — nuevo named arg `array $cascade = []` (valores `Cascade::*`)
    - `Attributes/OneToMany.php` — nuevo named args `array $cascade = []` + `bool $orphanRemoval = false`
  - **Metadata shape y helpers:**
    - `Metadata/EntityAssociationMetadata.php` — nuevos fields `cascade` (array) + `orphanRemoval` (bool) con defaults seguros; helpers `cascadesPersist()/cascadesRemove()/isManyToOne()/isOneToMany()`
    - `Metadata/EntityMetadata.php` — shape estricto `lifecycleCallbacks: array<7 keys>` mapeado a `KNOWN_LIFECYCLE_EVENTS`; helpers `hasCallbacks(string $event): bool` + `callbacksFor(string $event): list<Closure>`; validation RuntimeException en eventos desconocidos
  - **Metadata Registry discovery y validation:**
    - `Metadata/EntityMetadataRegistry.php` — imports 7 attrs + listener interface; `lifecycleAttributeMap()` helper; `buildLifecycleCallbacks()` en 2 partes: (1) method-level via `ReflectionMethod::getAttributes()` + setAccessible(true) invocable sobre instancia actual mediante Closure wrapper, (2) class-level via `new $className()` SIN Container, con validación `class_exists` e `implements Interface`; `mapAssociation` ManyToOne/OneToMany pasa cascade/orphanRemoval desde los atributos instanciados; build() shell y final pasan lifecycleCallbacks named arg a EntityMetadata
  - **UnitOfWork snapshots para OrphanRemoval:**
    - `ORM/UnitOfWork.php` — import `Metadata\EntityAssociationMetadata` (fix TypeError old-NS); nueva property privada scoped `originalCollections: oid → assocName → list<oid>` (NO estado singleton global); `registerManaged/synchronize` llaman `snapshotOneToManyCollections()`; `clear/detach` limpian originalCollections; API pública `snapshotOneToManyCollections($entity, $metadata)` y `collectionDiff($entity, $assocName): array{removed:list<object>, added:list<object>}`; helpers privados `readOneToManyCollection()` y `collectObjectIdsFromCollection()`
  - **EntityManager flush pipeline upgrade:**
    - `ORM/EntityManager.php` — import `Contracts\Cascade`; nuevos métodos privados:
      - `dispatchLifecycle(string $event, object $entity, array $context = [])` — reflection sobre metadata callbacks
      - `applyCascadesBeforeFlush(string $operation)` — BFS con visited `spl_object_id` anti-circular para `PERSIST` y `REMOVE`
      - `collectOrphansForRemoval()` — colección diff sobre entidades `Managed` únicamente (hard constraint)
      - `flushNewEntitiesInDependencyOrder()` — topological repeat-pass stall-guard: si pass sin avance + quedan NEW → RuntimeException circular reference
      - `newEntityIsInsertable($entity)` — ManyToOne targets ya tienen identifier asignado
      - `populateManyToOneForeignKeys($entity, $metadata)` — auto-sync FK values del lado owning usando la misma convención naming que `associationSourceValue`
      - `readSingleAssociationTarget()` / `assignOwningSideForeignKey()` / `readAssociationTargets()`
    - ordenamiento flush: hydrateManaged → dispatchLifecycle(postLoad); flush() → Paso 0 applyCascadesBeforeFlush(PERSIST) + applyCascadesBeforeFlush(REMOVE) + collectOrphansForRemoval → luego flushNewEntitiesInDependencyOrder() reemplaza iteración newEntities() raw; flushInsert: PrePersist → populateManyToOneForeignKeys → INSERT → sincronize → PostPersist; flushUpdate: populateManyToOneForeignKeys → compute changes → PreUpdate(context=[changes,currentValues,originalSnapshot]) → UPDATE SQL → sincronize → PostUpdate; flushDelete: PreRemove → DELETE SQL → PostRemove → detach UoW
  - **Pruebas:**
    - `vendor/voltstack/framework/tests/Unit/DVDB014LifecycleCascadeOrphanTest.php` — 14 tests / 119 assertions GREEN: EntityMetadata defaults 7 shape, unknown-event validation, callbacksFor/hasCallbacks, EntityAssociation cascade/orphan defaults, Cascade::ALL flags, Registry 7 method-level attrs discovery, dispatch closures reflection, invalid listeners (no existe / no implements), class listener registra 7 events + context propagation, backward compat BareSampleEntity
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nuevo subtest 7 `test_lifecycle_callbacks_cascade_and_orphan_removal_over_sqlite` 93 assertions: metadata sanity (cascade/orphan/isManyToOne/callbacks count por event), cascade persist post+3 comments (solo persist padre), orphan removal unset 1 comment (quedan 2), lifecycle method-level + class listener timestamped post, PreUpdate changes/currentValues/originalSnapshot context array, cascade REMOVE post → 0 comments, postLoad find after clear() forced hydration, backward compat OrmTag/OrmUser entidades preexistentes sin attrs = 0 callbacks todos events + asociaciones default cascade/orphan seguros
    - regresión GREEN vertical Database ORM entera: `DatabaseOrmFeatureTest` 7 tests / 265 assertions + `DVDB014LifecycleCascadeOrphanTest` 14 tests / 119 assertions = 21 tests / 384 assertions exit_code 0
- Resultado:
  - `Quantum/Database` ORM dispone de una policy layer de persistencia V1 completa alineada con estándares: los equipos ya pueden declarar hooks de lifecycle (tanto method-level atributos como class listeners) para mantener timestamps, auditoría updatedBy, validaciones automáticas, disparar side-effects dentro del mismo flush transaccional, SIN abrir una segunda frontera transactional,
  - Cascade PERSIST/REMOVE resuelve el error estructural "user olvidó persistir child comments" y "parent eliminado, hijos colgados FK NOT NULL" manual — todo el write path converge nuevamente al TransactionManager único sin engines paralelos,
  - OrphanRemoval V1 (Managed-only constraint + colecciones inverse OneToMany) cubre el caso mayoritario "retiro un item de una colección agregada y el ORM se encarga del DELETE" sin romper NEW entities o colecciones partial no sincronizadas todavía con la DB; snapshots permanecen estrictamente en UnitOfWork scoped (no en metadata singleton), preservando seguridad runtime persistente FrankenPHP/RoadRunner,
  - NEW inserts dependency-order stall-guard + ManyToOne owning-side FK auto-sync cierran 2 bugs persistentes de implementaciones previas: (a) iteración newEntities aleatoria provocaba FK violation insertando child antes que parent, (b) usuario debía setear manualmente `$comment->postId = $post->id` además de asignar `$comment->post = $post` — ya no, ambos paths se mantienen consistentes automáticamente antes de INSERT/UPDATE,
  - backward compat 100% confirmada: entidades sin atributos nuevos (OrmTag, OrmUser, OrmPost, etc) y código legacy consumer sigue funcionando idénticamente sin hooks disparados, sin nuevas dependencias externas Composer (0 librerías añadidas), sin cambios en firma constructor EntityManager ni bindings DatabaseServiceProvider,
  - y el vertical Lifecycle + Cascade + OrphanRemoval V1 queda validado con Unit (14/14) + Feature (93 assertions) sobre SQLite temp, regresión verde sobre 21 tests / 384 assertions existentes del subsistema Database + ORM.

### DV-DB-015

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `11 IdentityMap-UnitOfWork / Persistence`, `12 Hydration`, `13 Relationships`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - abrir el corte minimo de `Relationships Ampliados V1` sin reescribir el runtime ORM existente,
  - habilitar `OneToOne` bidireccional con soporte owning/inverse y resolución segura de `mappedBy`/`inversedBy`,
  - habilitar `ManyToMany` con tabla intermedia declarativa `#[JoinTable(name, joinColumns, inverseJoinColumns)]`,
  - ampliar carga y persistencia ORM para to-one/to-many avanzados manteniendo `IdentityMap`, `UnitOfWork` y `TransactionManagerInterface` como únicas fronteras runtime,
  - y validar el corte con suite unit especializada, feature real sobre SQLite y regresión completa del vertical ORM.
- Evidencia principal:
  - **Nuevos atributos públicos de mapping:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/OneToOne.php`
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/ManyToMany.php`
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/JoinTable.php`
  - **Metadata de asociaciones ampliada:**
    - `Metadata/EntityAssociationMetadata.php` — kinds `one_to_one` y `many_to_many`, fields `joinTable`, `joinTableSourceColumn`, `joinTableTargetColumn`, helpers `isToOne()`, `isToMany()`, `isOwningSide()`, `isInverseSide()`, `usesJoinTable()`
    - `Metadata/EntityMetadataRegistry.php` — discovery de `#[OneToOne]`, `#[ManyToMany]` y `#[JoinTable]`, resolución owning/inverse en build dos fases shell→final, y fallback vía reflection cuando el `mappedBy` todavía no está disponible en metadata final durante el shell build
  - **Runtime ORM ampliado sin segundo engine:**
    - `ORM/UnitOfWork.php` — snapshots de colecciones generalizados a asociaciones `to-many` (ya no sólo OneToMany) y `collectionDiff()` validando semántica to-many
    - `ORM/EntityManager.php` — `loadToOne()` soporta owning to-one e inverse `OneToOne`; `loadToMany()` soporta `OneToMany` y `ManyToMany`; `flushManyToManyMembershipChanges()` reconcilia contra filas reales de la join table; delete cleanup de memberships al remover entidades
    - `ORM/EntityQuery.php` — consultas por asociación limitadas a relaciones owning `to-one`; asociaciones `to-many` pasan a fallar explícitamente para evitar semántica inválida
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Unit/DVDB015ExtendedRelationshipsTest.php` — 5 tests / 30 assertions GREEN sobre metadata, owning/inverse `OneToOne`, owning/inverse `ManyToMany` y fallback two-phase
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_one_to_one_and_many_to_many_relationships_over_sqlite` con 46 assertions sobre metadata, FK sync OneToOne, query por asociación owning, load inverse OneToOne, inserts/remove de join rows ManyToMany, cascade persist de target nuevo y cleanup por delete
    - regresión GREEN vertical ORM: 27 tests / 460 assertions exit_code 0
- Resultado:
  - `Quantum/Database` ya cubre el segundo bloque mayoritario de modelado relacional ORM: `OneToOne` bidireccional y `ManyToMany` con join-table declarativa, sin introducir proxies, sin abrir un engine SQL paralelo y sin romper el contrato scoped del runtime,
  - la metadata ORM permanece singleton-safe y stateless: la resolución compleja de `mappedBy` en builds cruzados se resuelve con fallback de reflection, mientras que el estado mutable de colecciones y memberships sigue quedando en `UnitOfWork` y `EntityManager`,
  - `ManyToMany` persiste memberships contra estado real de base de datos, no sólo contra snapshots en memoria, evitando perder inserts para entidades recién sincronizadas,
  - el write path mantiene compatibilidad hacia atrás: `ManyToOne` y `OneToMany` existentes continúan verdes, y las consultas inválidas por asociaciones `to-many` ahora fallan de forma explícita y explicable,
  - y el vertical Relationships Ampliados V1 queda validado con Unit + Feature + regresión completa ORM en verde sobre SQLite.

### DV-DB-016

- Estado: `Implementado`
- Bloque documental: `04 Query Builder`, `10 ORM`, `11 IdentityMap-UnitOfWork / Persistence`, `12 Hydration`, `13 Relationships`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - abrir un corte incremental de `fetch strategies` antes de joins declarativos completos,
  - introducir precarga explícita `EntityQuery::with(...)` para evitar N+1 sobre asociaciones soportadas sin tocar todavía el modelo SQL de joins,
  - añadir soporte mínimo de predicates `IN` en el query layer para poder batch-load de forma real,
  - mantener la separación `Query Execution != Hydration != IdentityMap`, reutilizando el runtime ORM scoped ya existente,
  - y validar que colecciones precargadas siguen siendo mutables/flushables, especialmente en `ManyToMany`.
- Evidencia principal:
  - **Query layer mínimo ampliado para batch loading:**
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Builder/SelectQueryBuilder.php` — nuevo helper `whereIn(string $column, array $values): self`
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Compiler/SqlCompiler.php` — compilación segura de predicates `IN` / `NOT IN`, incluyendo semántica segura para listas vacías
  - **API ORM pública de precarga explícita:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php` — nuevo método `with(string ...$associations): self`; `get()` y `first()` disparan precarga batch tras hidratar entidades root
  - **Batch preload runtime sobre EntityManager:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityManager.php` — nuevo `preloadAssociations()` y loaders batch especializados para owning to-one, inverse `OneToOne`, `OneToMany` y `ManyToMany`, con refresh de snapshots para colecciones preloaded que luego se mutan y flushean
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_entity_query_with_preloads_batch_loads_supported_associations` (24 assertions) cubriendo `with('comments')`, `with('post')`, `with('profile')`, `with('students')`, `with('courses')` y flush posterior de colección `ManyToMany` precargada
    - regresión GREEN vertical ORM: `DVDB014LifecycleCascadeOrphanTest` + `DVDB015ExtendedRelationshipsTest` + `DatabaseOrmFeatureTest` = `28 tests / 484 assertions`
    - regresión focalizada query layer GREEN: `DatabaseQueryCompilerTest` + `DatabaseQueryBuilderExecutionTest`
- Resultado:
  - `Quantum/Database` ya ofrece una primera estrategia explícita de fetch útil para el 80% de los casos inmediatos: `EntityQuery::with(...)` resuelve asociaciones por lotes tras la carga root, sin depender de joins SQL ni de proxies,
  - la hidratación sigue siendo deliberadamente separada del compilador SQL: el query layer sólo aporta el predicate `IN` necesario para batching, mientras que la asociación de objetos y el reuso de identidad continúan viviendo en `EntityManager` + `IdentityMap`,
  - las colecciones `to-many` precargadas quedan sincronizadas con snapshots scoped para que un `flush()` posterior siga detectando cambios reales, evitando una precarga bonita pero inútil,
  - el corte no pretende cerrar eager joins declarativos ni partial hydration; deja esa frontera claramente abierta para la siguiente fase,
  - y el vertical queda nuevamente verde con cobertura feature real sobre SQLite y regresión ORM completa.

### DV-DB-017

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `12 Hydration`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - abrir un corte explícito de `projection/scalar hydration` sobre el ORM sin fingir soporte de partial entities,
  - exponer una API pública utilizable para selecciones parciales desde `EntityQuery`,
  - resolver columnas proyectadas usando metadata ORM (`field`, embedded path y owning to-one association),
  - convertir resultados de vuelta a PHP typed values usando el pipeline ya existente,
  - y bloquear de forma explícita la hidratación parcial accidental de entidades vía `get()` / `first()`.
- Evidencia principal:
  - **API pública nueva en ORM Query surface:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php`
    - nuevos métodos `select(string ...$fields)`, `rows()`, `firstRow()`, `pluck(string $field)`, `value(string $field)`
  - **Resolución ORM-aware de proyecciones:**
    - campos simples `field`
    - embedded paths `embedded.inner`
    - asociaciones owning `to-one` proyectadas como identifier escalar del target
    - reconversión a tipos PHP mediante `EntityFieldMetadata::castValue()` / `EntityEmbeddedFieldMetadata::castValue()` y canonicalización de identifiers target
  - **Guardrails honestos contra partial entities:**
    - `EntityQuery::get()` y `first()` lanzan `RuntimeException` cuando la query está en modo projection, forzando el uso de `rows()/firstRow()/pluck()/value()`
    - `with(...)` y `select(...)` no se pueden combinar en la misma query para no mezclar entity hydration con scalar/projection hydration
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_entity_query_select_rows_firstrow_pluck_and_value_support_projection_mode` (18 assertions) validando proyecciones tipadas de scalar fields, enums, `DateTimeImmutable`, JSON, embedded paths, owning association identifiers y guardrail contra `get()` parcial
    - regresión GREEN vertical ORM: `29 tests / 502 assertions`
    - regresión focalizada query layer GREEN: `DatabaseQueryCompilerTest` + `DatabaseQueryBuilderExecutionTest` (con 2 deprecations heredadas de PHPUnit en ese subset)
- Resultado:
  - `Quantum/Database` ya ofrece un primer corte útil de scalar/projection hydration desde el ORM, sin obligar al consumer a bajar a la API raw del query builder,
  - la API mantiene una frontera honesta: selección parcial sí, partial entity hydration no,
  - los valores proyectados respetan el pipeline de tipos ya existente para enums, `DateTimeImmutable`, JSON y embedded-inner fields,
  - las asociaciones owning `to-one` pueden proyectarse como identifiers escalares sin introducir joins ni materialización de entidades target,
  - y el siguiente gap ya no es “cómo pedir una proyección”, sino cómo evolucionar eso hacia partial hydration real, joins declarativos y planes de hidratación más ricos.

### DV-DB-018

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `12 Hydration`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - abrir un primer corte de `partial entity hydration` real, pero explícito y seguro,
  - devolver entidades parciales desde el ORM sin registrarlas en `IdentityMap` ni `UnitOfWork`,
  - exigir `refresh()` para convertir una entidad parcial en entidad completa y managed,
  - mantener la frontera honesta: partial entities sí, persistencia/mutación directa no hasta hacer upgrade explícito,
  - y validar que embedded paths parciales también funcionan en este modo.
- Evidencia principal:
  - **API pública nueva en ORM Query surface:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php`
    - nuevos métodos `partial(string ...$fields): self`, `getPartial(): array`, `firstPartial(): ?object`
  - **Runtime ORM ampliado para parciales detached:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityManager.php`
    - nuevo `hydratePartial(EntityMetadata $metadata, array $row): object`
    - sidecar interno de seguimiento `partialEntities`
    - `persist()` y `remove()` rechazan explícitamente entidades parciales hasta que el consumer haga `refresh()`
    - `refresh()` promueve la entidad parcial a managed completa, la rehidrata desde DB y la saca del registro de parciales
  - **Semántica del corte:**
    - `partial(...)` auto-incluye la PK aunque el usuario no la pida explícitamente
    - soporta scalar fields y embedded paths
    - rechaza asociaciones en este V1 para no prometer materialización parcial de relaciones
    - `get()/first()` siguen prohibidos en partial mode; se obliga a `getPartial()/firstPartial()`
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_entity_query_partial_hydration_returns_detached_entities_until_refresh` (23 assertions) validando entidad parcial detached, properties no seleccionadas sin inicializar, embedded parcial, rechazo de `persist()` directo y upgrade vía `refresh()`
    - regresión GREEN vertical ORM: `30 tests / 525 assertions`
    - regresión focalizada query layer GREEN: `DatabaseQueryCompilerTest` + `DatabaseQueryBuilderExecutionTest` (con 2 deprecations heredadas de PHPUnit en ese subset)
- Resultado:
  - `Quantum/Database` ya soporta partial entity hydration explícita sin degradar su modelo de consistencia,
  - el corte evita el error clásico de ORMs “entidad medio cargada pero aparentemente managed”: aquí la entidad parcial nace detached y sólo entra al runtime de tracking mediante `refresh()`,
  - embedded value objects también pueden materializarse parcialmente cuando sólo algunas columnas fueron seleccionadas,
  - el sistema sigue sin vender joins declarativos, partial updates mágicos ni snapshots parciales managed,
  - y el siguiente gap real pasa a ser una fase V2: partial entities managed con hydration plans más ricos y/o joins declarativos.

### DV-DB-019

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `11 IdentityMap-UnitOfWork / Persistence`, `12 Hydration`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - abrir un corte explícito de `partial entity hydration managed` sin depender todavía de joins declarativos,
  - permitir que una query parcial deje la entidad registrada en `IdentityMap` y `UnitOfWork`,
  - limitar dirty-check y write path exclusivamente a los campos realmente cargados,
  - promover automáticamente la instancia parcial managed a entidad completa cuando se hace `find()` o `refresh()`,
  - y evitar operaciones peligrosas sobre asociaciones o remove cascades desde ese estado parcial.
- Evidencia principal:
  - **API pública nueva en ORM Query surface:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php`
    - nuevos métodos `partialManaged(string ...$fields): self`, `getPartialManaged(): array`, `firstPartialManaged(): ?object`
  - **Tracking scoped para partial managed:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/UnitOfWork.php`
    - nuevo registro `partialManagedFields`
    - nuevos métodos `registerManagedPartial()`, `synchronizePartial()`, `isPartialManaged()`, `partialManagedFields()`
  - **Runtime ORM ampliado sin corrupción de snapshots:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityManager.php`
    - nuevo `hydrateManagedPartial(...)`
    - `find()` detecta instancia partial-managed ya presente en identity map y la completa vía `refresh()`
    - `dirtyManagedEntities()` y `flushUpdate()` comparan/escriben sólo sobre el subconjunto cargado
    - `remove()` rechaza explícitamente entidades managed-partial hasta que se haga `refresh()`
    - loops de orphanRemoval y ManyToMany diff/flush ignoran managed partial entities para no inferir cambios sobre asociaciones no cargadas
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_entity_query_partial_managed_hydration_tracks_only_loaded_fields_and_upgrades_on_find` (27 assertions) validando estado Managed, update sólo sobre campos cargados, no-overwrite de columnas no cargadas, bloqueo de remove y auto-upgrade a entidad completa vía `find()`
    - regresión GREEN vertical ORM: `31 tests / 552 assertions`
    - regresión focalizada query layer GREEN: `DatabaseQueryCompilerTest` + `DatabaseQueryBuilderExecutionTest` (con 2 deprecations heredadas de PHPUnit en ese subset)
- Resultado:
  - `Quantum/Database` ya soporta partial entities managed con un modelo de seguridad razonable: tracked sí, pero sólo dentro del subconjunto de fields realmente cargados,
  - el dirty-check deja de comparar snapshots completos para este modo y se acota al set de fields loaded, evitando sobrescribir defaults o columnas no seleccionadas,
  - `find()` y `refresh()` ya pueden convertir sin fricción una instancia managed-partial en entidad completa reutilizando la misma referencia,
  - el sistema sigue sin prometer asociaciones parciales, joins declarativos ni hydration plans compilados,
  - y el siguiente gap real pasa a ser la fase donde ese modelo managed parcial pueda convivir con estrategias de carga más ricas a nivel SQL.

### DV-DB-020

- Estado: `Implementado`
- Bloque documental: `04 Query Builder`, `06 SQL Compiler`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - abrir el primer corte útil de `joins declarativos` en el query layer,
  - soportar `INNER JOIN` y `LEFT JOIN` con condiciones `column-to-column`,
  - introducir alias de tabla base y de tablas joined,
  - permitir columnas seleccionadas con `AS alias`,
  - y validar el comportamiento con compilación unitaria y ejecución real sobre SQLite.
- Evidencia principal:
  - **Query Model / AST extendidos:**
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Model/TableReference.php` ahora soporta alias
    - nuevo `vendor/voltstack/framework/src/Quantum/Database/Query/Model/Join.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Model/SelectQuery.php` añade `joins`
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Ast/TableNode.php` ahora soporta alias
    - nuevo `vendor/voltstack/framework/src/Quantum/Database/Query/Ast/JoinNode.php`
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Ast/SelectQueryNode.php` añade `joins`
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Ast/QueryAstFactory.php` mapea alias + joins del modelo al AST
  - **API pública nueva en Query Builder:**
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Builder/SelectQueryBuilder.php`
    - nuevos métodos `as(string $alias)`, `join(...)`, `leftJoin(...)`
    - `count()`, `first()` y `toSelectQuery()` preservan alias + joins del builder actual
  - **SQL Compiler ampliado:**
    - `vendor/voltstack/framework/src/Quantum/Database/Query/Compiler/SqlCompiler.php`
    - soporte para compilar `FROM ... AS ...`
    - soporte para `INNER JOIN` / `LEFT JOIN`
    - soporte para `SELECT table.column AS alias`
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Unit/DatabaseQueryCompilerTest.php` — nueva prueba `test_it_compiles_select_queries_with_inner_and_left_joins_and_aliases` (2 assertions)
    - `vendor/voltstack/framework/tests/Feature/DatabaseQueryBuilderExecutionTest.php` — nueva prueba `test_query_builder_executes_inner_and_left_joins_against_sqlite` (6 assertions)
    - regresión GREEN query layer: `5 tests / 25 assertions`
    - el subset sigue reportando `2 deprecations` heredadas de PHPUnit, no introducidas por este corte
- Resultado:
  - `Quantum/Database` ya tiene joins declarativos utilizables en el Query Builder sin bajar a SQL manual,
  - el compilador puede generar SQL con alias consistentes tanto en `FROM` como en `JOIN` y columnas seleccionadas,
  - este corte sigue deliberadamente acotado al query layer: no resuelve todavía joins guiados por metadata ORM ni hydration relacional automática,
  - la base ya está lista para que el siguiente bloque conecte esos joins con estrategias de carga más ricas del ORM.

### DV-DB-021

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `12 Hydration`, `13 Relationships`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - abrir el primer puente entre metadata ORM y joins del query layer,
  - soportar `join()` / `leftJoin()` sobre asociaciones root `to-one`,
  - permitir `where/orderBy/select/value/pluck` sobre fields del target joined usando paths `association.field`,
  - mantener la hidratación de entidades root segura en presencia de joins,
  - y dejar explícito que este corte todavía no materializa automáticamente la asociación joined.
- Evidencia principal:
  - **API pública nueva en ORM Query surface:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php`
    - nuevos métodos `join(string $association, ?string $alias = null): self` y `leftJoin(string $association, ?string $alias = null): self`
  - **Traducción metadata → SQL join:**
    - soporte para asociaciones `ManyToOne`, `OneToOne` owning y `OneToOne` inverse en el root query
    - resolución de columnas `ON` usando `EntityAssociationMetadata::{sourceColumn,targetColumn}` y metadata del root/target
    - alias root automático `t0` para evitar colisiones y preservar entity hydration con `t0.*`
  - **Capacidades nuevas del querying ORM:**
    - `where('post.title', ...)`
    - `orderBy('profile.bio')`
    - `select('body', 'post.title')->rows()`
    - `value('post.title')` / `pluck(...)` sobre fields joined
  - **Guardrails del corte:**
    - joins sólo para asociaciones root `to-one`
    - rechazo explícito de joins `to-many`
    - `partial()` / `partialManaged()` no se combinan con joined-association mode en este V1
    - la asociación joined no se hidrata automáticamente en la propiedad PHP; este corte es de querying/proyección, no de materialización relacional completa
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_entity_query_can_join_to_one_associations_via_metadata_for_filters_and_projections` (8 assertions) validando `ManyToOne` + `OneToOne` inverse, proyección joined, filtro por field joined, `value()` joined y rechazo de join `to-many`
    - regresión GREEN vertical ORM: `32 tests / 560 assertions`
    - regresión GREEN query layer: `5 tests / 25 assertions` (con `2 deprecations` heredadas de PHPUnit en ese subset)
- Resultado:
  - `Quantum/Database` ya conecta metadata ORM con joins declarativos del query layer para casos `to-one`,
  - el consumer puede filtrar, ordenar y proyectar por fields del target sin bajar a SQL ni mapear joins manualmente,
  - entity hydration root sigue siendo segura porque el query root se reduce a `t0.*` cuando hay joins y no estamos en projection mode,
  - el sistema sigue sin vender eager hydration automática del target joined ni joins ORM sobre colecciones,
  - y el siguiente gap real pasa a ser cómo usar esta base para hydration relacional más rica o fetch strategies declarativas.

### DV-DB-022

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `12 Hydration`, `13 Relationships`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - cerrar el siguiente paso después del joined querying `to-one`,
  - permitir que `get()` y `first()` materialicen automáticamente la asociación `to-one` ya joined,
  - reutilizar `IdentityMap` para el target joined,
  - preservar la hidratación segura de la entidad root,
  - y seguir dejando fuera `to-many`, grafos profundos y combinaciones con partial modes.
- Evidencia principal:
  - **Hydration joined en entity mode:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php`
    - `get()` y `first()` detectan joined-association mode y ejecutan una selección interna `t0.*` + columnas aliased del target
    - nueva hidratación desde una sola fila SQL para root + target joined `to-one`
  - **Consistencia local + runtime:**
    - la entidad target joined se hidrata vía `EntityManager::hydrateManaged(...)`, reusando `IdentityMap`
    - `leftJoin()` sin fila target asigna `null`
    - back-reference local se enlaza sólo cuando la metadata inversa también es `to-one`
    - asociaciones ya materializadas por join no vuelven a pasar por preload redundante en `with(...)`
  - **Scope deliberadamente acotado:**
    - sólo asociaciones root `to-one`
    - no joins `to-many`
    - no hydration transitiva de asociaciones del target
    - no mezcla con `partial()` / `partialManaged()`
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_entity_query_joined_to_one_associations_are_hydrated_on_entity_results` (11 assertions) validando `ManyToOne` joined, reuse de `IdentityMap`, `OneToOne` owning joined, back-reference local y `leftJoin()` inverse con target ausente
    - regresión GREEN vertical ORM: `33 tests / 571 assertions`
    - regresión GREEN query layer: `5 tests / 25 assertions` (con `2 deprecations` heredadas de PHPUnit en ese subset)
- Resultado:
  - `Quantum/Database` ya no se limita a querying/proyección sobre joins ORM `to-one`: ahora también puede materializar la asociación joined en resultados de entidad,
  - la entity hydration root sigue protegida por selección aislada `t0.*`,
  - el target joined reutiliza `IdentityMap` y puede mantener consistencia local con back-reference `to-one`,
  - el sistema todavía no vende joins ORM `to-many`, hydration relacional profunda ni fetch strategies declarativas completas,
  - y el siguiente gap real pasa a ser cómo extender esta base a relaciones `to-many`, asociaciones parciales y planificación de hidratación más rica.

### DV-DB-023

- Estado: `Implementado`
- Bloque documental: `10 ORM`, `12 Hydration`, `13 Relationships`, `29 Testing`, `31 Developer Experience`
- Alcance objetivo:
  - extender joins ORM metadata-guided al caso `to-many`,
  - soportar `OneToMany` y `ManyToMany` en querying/proyección,
  - permitir que `get()` deduzca roots repetidos y materialice colecciones,
  - mantener `IdentityMap` y snapshots de `UnitOfWork` coherentes,
  - y dejar fuera `first()`/paginación root-aware hasta tener una semántica segura para joins de colección.
- Evidencia principal:
  - **Joins ORM `to-many` ya operativas en EntityQuery:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityQuery.php`
    - `join(...)/leftJoin(...)` ahora soportan `OneToMany` y `ManyToMany`
    - `where/orderBy/select/rows()/pluck()/value()` pueden usar paths joined sobre colecciones
  - **Hydration entity-mode con deduplicación:**
    - `get()` agrupa por identifier del root y evita duplicados de entidad
    - las colecciones materializadas no repiten targets ya vistos
    - `leftJoin()` sin targets produce `[]`
    - `first()` pasa a rechazarse cuando hay joins `to-many`
    - `count()` cuenta roots en lugar de filas multiplicadas por el join
  - **Integración runtime limpia:**
    - `vendor/voltstack/framework/src/Quantum/Database/ORM/EntityManager.php` añade `snapshotCollections()` para resincronizar snapshots de `UnitOfWork` tras materializar colecciones joined
    - `IdentityMap` se reutiliza tanto para roots como para targets joined
    - en `OneToMany`, el back-reference `ManyToOne` del target puede quedar enlazado al root ya materializado
  - **Scope deliberadamente acotado:**
    - `to-many` joined sólo se materializa en `get()`
    - `first()` no intenta fingir una colección completa con `LIMIT 1`
    - no hay todavía paginación root-aware sobre joins `to-many`
    - no hay hydration transitiva profunda ni asociaciones parciales
  - **Pruebas GREEN:**
    - `vendor/voltstack/framework/tests/Feature/DatabaseOrmFeatureTest.php` — nueva prueba `test_entity_query_can_join_to_many_associations_with_root_deduplication_and_collection_hydration` (16 assertions) validando `OneToMany`, `ManyToMany`, deduplicación de roots, `leftJoin()` vacío, `count()` root-aware y rechazo de `first()`
    - regresión GREEN vertical ORM: `34 tests / 586 assertions`
    - regresión GREEN query layer: `5 tests / 25 assertions` (con `2 deprecations` heredadas de PHPUnit en ese subset)
- Resultado:
  - `Quantum/Database` ya cubre joins ORM `to-one` y `to-many` dentro de `EntityQuery`,
  - el consumer puede proyectar por fields joined de colecciones y también obtener entidades root con sus colecciones materializadas en `get()`,
  - roots y targets siguen reutilizando `IdentityMap`, y los snapshots de colección quedan coherentes para posteriores `flush()`,
  - el sistema sigue siendo explícito al bloquear `first()` para joins `to-many` y al no prometer paginación segura por root todavía,
  - y el siguiente gap real pasa a ser cómo construir fetch strategies más maduras: paginación root-aware, asociaciones parciales y planificación/hydration relacional más profunda.

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
12. Base de relaciones ORM V1 ampliada con:
    - atributos `#[ManyToOne]`, `#[OneToMany]`, `#[OneToOne]`, `#[ManyToMany]` y `#[JoinTable(...)]`,
    - `EntityAssociationMetadata` como descriptor canonico de asociaciones con kinds `many_to_one`, `one_to_many`, `one_to_one`, `many_to_many`, shape de join-table y helpers owning/inverse, to-one/to-many y usesJoinTable,
    - `EntityMetadataRegistry::build()` en dos fases (shell → final) para resolver referencias cruzadas bidireccionales sin recursion infinita, incluyendo fallback vía reflection para inverse `mappedBy` en `OneToOne` y `ManyToMany`,
    - helpers `EntityManager::loadToOne()` y `EntityManager::loadToMany()` reutilizando `IdentityMap` + `DatabaseQueryManager` para owning/inverse `OneToOne`, `OneToMany` y `ManyToMany`,
    - precarga explícita `EntityQuery::with(...associations)` por lotes sobre asociaciones soportadas, sin joins SQL declarativos todavía,
    - projection/scalar hydration explícita vía `EntityQuery::select(...)->rows()/firstRow()/pluck()/value()` sobre fields, embedded paths y owning to-one identifiers,
    - partial entity hydration explícita y detached vía `EntityQuery::partial(...)->getPartial()/firstPartial()` con upgrade posterior vía `refresh()`,
    - partial entity hydration managed vía `EntityQuery::partialManaged(...)->getPartialManaged()/firstPartialManaged()` con dirty-check limitado a loaded fields,
    - Query Builder con `as(...)`, `join(...)` y `leftJoin(...)` sobre SQL declarativo V1,
    - ORM Query surface con `join(...)` / `leftJoin(...)` guiados por metadata para querying/proyección sobre asociaciones root `to-one` y `to-many`,
    - persistencia de memberships `ManyToMany` mediante reconciliación contra filas reales de la join table,
    - y traduccion automatica en `EntityQuery::where()` / `orderBy()` limitada a asociaciones owning `to-one`, con rechazo explícito de asociaciones `to-many`.
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
4. Base de relaciones ORM ya operativa para ManyToOne, OneToMany, OneToOne y ManyToMany con join-table declarativa, pero todavia sin proxies, lazy transparente, eager joins/fetch strategies declarativas, control sistemico de N+1 ni hydration planificada/compilada.
5. Base de value objects embedded ya operativa pero todavia sin nested embedded (embedded dentro de embedded), sin embedded en relationships, sin embedded collection/JSON, sin equals/hashCode por valor, y sin soporte en repository `findBy()` shortcuts para paths anidados (aunque `EntityQuery` si la soporta).
6. Repository DI tipado ya operativo pero todavía sin manifests extensibles para discovery en módulos separados, sin interface bindings por entidad (ej `bind(OrmProductRepositoryInterface::class → concrete)`) y sin helpers `findByXxx()` mágicos ni Criteria API rich.

### Aun no desarrollado con evidencia suficiente

1. Hydration planificada/compilada, caches de metadata y surface ORM ampliada.
2. Relationships V2 e hidratacion avanzada (paginación/root-limiting segura sobre joins ORM `to-many`, asociaciones parciales/partial hydration relacional, proxies/lazy transparente, control sistémico más profundo de N+1, hydration planificada/compilada y politicas avanzadas de relaciones).
3. Value objects avanzados: nested embedded, embedded collection via JSON, value identity/equality helpers.
4. Security, Resilience, Plugin y Legacy migration runtime.
5. Capabilities avanzadas, pagination, batch/streaming y distribucion.

## Siguiente bloque recomendado

### Opcion recomendada posterior

Profundizar el vertical ORM posterior a `DV-DB-015`, en el siguiente orden natural:

1. **Hydration + Fetch Strategies V2**: extender la base actual de joins ORM `to-one` + `to-many` hacia paginación/root-limiting segura, asociaciones parciales/partial hydration relacional, estrategia LAZY/EAGER declarativa por asociación, control sistémico del N+1 más allá de `with(...)`, y base para hidratación planificada/compilada. (bloques 04 Query Builder, 12 Hydration, 13 Relationships)
2. **Factories & Seeders V2 ampliados**: comandos generators CLI `make:factory`, `make:seeder`, factory states/sequences nativos, soporte para seeder dependencias ordenado, y SeederRunner con barras de progreso/logger integrado. (Prioridad 7, bloque 18 Factories)
3. **Repositories avanzados V1**: manifests extensibles para discovery de repositorios en módulos separados, interfaces por entidad + container bindings automáticos, Criteria API typed independiente de SQL, y codegen helpers para clases repositorio. (bloque 119 Repositorios, 32 Integrations)
4. **Cache de consultas + IdentityMap advanced**: second-level cache opcional driver-agnostic para find + where frecuentes, refresh/merge/detach profundo, y Policy isolation de lifecycle para listeners vía Container resolución (V2 lifecycle listeners con dependency injection, sustituyendo `new $className()` actual).

Motivo:

- la base ORM ya convergió sobre `DatabaseQueryManager` + `TransactionManagerInterface`, incluyendo Types V1, Relaciones ManyToOne/OneToMany/OneToOne/ManyToMany V1, precarga explícita `with(...)`, projection/scalar hydration V1, partial entity hydration detached V1, partial entity hydration managed V1, Embedded V1, Factories + Seeders V1, Repository Factory DI + helpers ergonomicos V1, y ahora también querying + eager hydration de joins ORM `to-one` y `to-many` en `get()`,
- la vida scoped (`EntityManager`, `IdentityMap`, `UnitOfWork`, `SeederRunner`, `EntityRepositoryFactory`, `originalCollections` snapshots en UoW) y seguridad para runtime persistente siguen intactas,
- y el siguiente gap estructural dominante ya no es la ausencia de joins ORM `to-many`, sino **cómo volver esa capacidad verdaderamente madura**: paginación/root-limiting segura, asociaciones parciales, fetch strategies declarativas y control del N+1 más profundo antes de abrir capas de codegen o APIs de repositorio más sofisticadas.

## Regla de actualizacion de esta bitacora

Cada nuevo cierre de fase Database debe registrar:

1. un nuevo identificador `DV-DB-00X`,
2. documentos impactados,
3. alcance implementado,
4. evidencia principal,
5. resultado operativo,
6. siguiente gap natural.
