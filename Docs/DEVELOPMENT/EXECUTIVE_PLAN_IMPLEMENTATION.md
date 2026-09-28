# EXECUTIVE_PLAN_IMPLEMENTATION

## Proposito

Este documento define el plan ejecutivo para construir `Quantum/Database` dentro de:

- `vendor/voltstack/framework/src/Quantum/Database`

El plan usa como fuente principal la arquitectura de `vendor/voltstack/database-lab/Docs` y la traduce a un roadmap ejecutable, incremental y verificable.

## Objetivo general

Construir un `Database V1` real, usable y seguro para runtime persistente, sin empezar por capas altas que dependan de un nucleo inexistente.

## Objetivo de Database V1

`Database V1` se considerara logrado cuando exista evidencia operativa de:

1. configuracion tipada y composition root,
2. `DatabaseExecutionScope` integrado al lifecycle del framework,
3. `Driver`, `Connection`, `ConnectionManager`, `Platform` y `Dialect`,
4. `Execution Engine` minimo,
5. Query Builder minimo conectado al compiler y execution,
6. Schema Builder y Migration System minimos,
7. `TransactionManager` minimo,
8. surface publica inicial:
   - facade contextual,
   - CLI basica,
   - telemetria minima.

## No objetivos de V1

Quedan fuera del primer cierre, salvo necesidad explicita:

- ORM completo,
- Identity Map completa,
- Unit of Work completa,
- lazy/eager relationship system completo,
- dual ORM runtime,
- legacy migration tooling,
- plugin system completo,
- arquitectura distribuida avanzada.

## Criterios ejecutivos

El plan debe preservar siempre:

- runtime persistente seguro,
- lifetimes correctos,
- ownership claro de recursos,
- integracion con framework sin acoplamiento impropio,
- trazabilidad documental en `Docs/DEVELOPMENT`,
- y pruebas al nivel correcto de garantia.

## Dependencias marco ya disponibles

El framework ya aporta habilitadores reales para arrancar:

1. `Quantum/Container`
2. `Quantum/Config`
3. `Runtime/Context`
4. `Quantum/Telemetry`
5. `Quantum/Console`
6. `Platform/Application.php`
7. `VoltStack\Framework\ServiceProvider`

## Fase 0 - Linea base y preparacion

### Objetivo

Formalizar el sistema de desarrollo y el alcance real del subsistema antes de abrir codigo productivo.

### Entregables

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

### Estado actual

- `Completado` en este corte documental.

## Fase 1 - Bootstrap, Config y Runtime Scope

### Documentos fuente principales

- `06_DATABASE_CONFIGURATION_SYSTEM.md`
- `07_DATABASE_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION.md`
- `251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md`
- `252_DATABASE_REQUEST_SCOPE_SYSTEM.md`
- `311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md`
- `312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md`
- `313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md`
- `321_DATABASE_HTTP_REQUEST_LIFECYCLE_INTEGRATION.md`

### Objetivo

Crear el nucleo de integracion que permita que Database exista como subsistema del framework sin fugas de estado.

### Codigo objetivo

- `src/Quantum/Database/Config`
- `src/Quantum/Database/Runtime`
- `src/Quantum/Database/Integration`
- `src/Quantum/Database/Contracts`

### Estado actual

- `Implementado en DV-DB-001`

### Entregables minimos

1. `DatabaseServiceProvider`
2. `DatabaseCompositionRoot`
3. `DatabaseConfiguration` tipada e inmutable
4. `DatabaseExecutionScope`
5. lifecycle hooks de inicio y fin de scope
6. validacion inicial de lifetimes

### Pruebas minimas

1. binding tests
2. scope lifecycle tests
3. config compilation tests
4. request cleanup tests

### Criterio de salida

Se puede abrir codigo de conexion y ejecucion sin depender de estado global ni de `config()` dinamico.

### Resultado del corte DV-DB-001

1. `DatabaseServiceProvider` registrado por defecto en `Application`.
2. `DatabaseCompositionRoot` disponible como singleton.
3. `DatabaseConfiguration` tipada e inmutable.
4. `DatabaseExecutionScope` creado y finalizado durante el lifecycle HTTP.
5. `DatabaseContext` y `DatabaseScopeLifecycleManager` como base de runtime.
6. `config/database.php` minimo en el skeleton.
7. pruebas unitarias y feature en verde para bindings y scope.

## Fase 2 - Driver, Connection, Platform y Dialect

### Estado actual

- `Implementado en DV-DB-002`

### Documentos fuente principales

- `10_DATABASE_DRIVER_ARCHITECTURE.md`
- `11_DATABASE_CONNECTION_SYSTEM.md`
- `12_DATABASE_CONNECTION_MANAGER.md`
- `13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION.md`
- `14_DATABASE_CONNECTION_POOLING_SYSTEM.md`
- `15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM.md`
- `16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md`
- `17_DATABASE_DIALECT_SYSTEM.md`
- `18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md`
- `19_DATABASE_MYSQL_AND_MARIADB_PLATFORM.md`
- `20_DATABASE_POSTGRESQL_PLATFORM.md`
- `21_DATABASE_SQLITE_PLATFORM.md`
- `22_DATABASE_DRIVER_EXTENSION_SYSTEM.md`

### Objetivo

Cerrar la capa de acceso logico y la separacion formal entre canal nativo, semantica de plataforma y sintaxis SQL.

### Codigo objetivo

- `src/Quantum/Database/Driver`
- `src/Quantum/Database/Connection`
- `src/Quantum/Database/Platform`
- `src/Quantum/Database/Dialect`

### Entregables minimos

1. contracts base de driver y native connection
2. `Connection`
3. `ConnectionManager`
4. `ConnectionDefinition` y aliases
5. `Platform` y `Dialect` iniciales
6. al menos una ruta operativa de conexion

### Pruebas minimas

1. unit tests de resolucion y lifetimes
2. integration tests de conexion real
3. state reset tests

### Criterio de salida

El subsistema puede resolver una conexion logica segura dentro del scope y liberarla correctamente.

### Resultado del corte DV-DB-002

1. contracts base de driver, native connection, connection, dialect y platform.
2. `ConnectionDefinition` y `ConnectionDefinitionRegistry`.
3. `DriverRegistry` con ruta inicial `PDO + SQLite`.
4. `ConnectionManager` scoped con cache por scope.
5. `SqlitePlatform` y `SqliteDialect` como primer camino operativo.
6. desconexion de conexiones al cerrar la request.
7. pruebas unitarias y feature con SQLite real en verde.

## Fase 3 - Execution Engine minimo

### Estado actual

- `Implementado en DV-DB-003`

### Documentos fuente principales

- `76_DATABASE_EXECUTION_ENGINE_ARCHITECTURE.md`
- `77_DATABASE_QUERY_EXECUTOR_SYSTEM.md`
- `78_DATABASE_STATEMENT_EXECUTION_SYSTEM.md`
- `79_DATABASE_PREPARED_STATEMENT_SYSTEM.md`
- `80_DATABASE_PARAMETER_BINDING_SYSTEM.md`
- `81_DATABASE_RESULT_SYSTEM.md`
- `82_DATABASE_RESULT_CURSOR_SYSTEM.md`
- `83_DATABASE_STREAMING_RESULT_SYSTEM.md`
- `84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md`
- `85_DATABASE_EXECUTION_ERROR_SYSTEM.md`
- `86_DATABASE_EXECUTION_RETRY_SYSTEM.md`

### Objetivo

Establecer la frontera entre conocimiento compilado y estado runtime mutable.

### Codigo objetivo

- `src/Quantum/Database/Execution`

### Entregables minimos

1. `ExecutionContext`
2. `RuntimeBindingSet`
3. `QueryExecutor`
4. `Statement` / `PreparedStatement`
5. `Result` / `Cursor`
6. error model y cleanup determinista

### Pruebas minimas

1. unit tests de bindings y result model
2. integration tests de ejecucion real
3. timeout/cancellation cleanup tests cuando aplique

### Criterio de salida

El subsistema puede ejecutar una orden compilada con ownership explicito de recursos.

### Resultado del corte DV-DB-003

1. `CompiledDatabaseCommand` como entrada runtime de ejecucion.
2. `RuntimeBindingSet` para bindings separados y normalizados.
3. `StatementExecutor` sobre la capa de conexion existente.
4. `QueryExecutor` como orquestador minimo sobre statement execution.
5. `DatabaseResult` desacoplado del resultado nativo de PDO.
6. `ExecutionFailure` y `ExecutionException` para error handling tipado inicial.
7. pruebas unitarias y feature con SQLite real en verde.

## Fase 4 - Query MVP

### Estado actual

- `Implementado en DV-DB-004`

### Documentos fuente principales

- `23_DATABASE_QUERY_ARCHITECTURE.md`
- `24_DATABASE_QUERY_MODEL.md`
- `25_DATABASE_QUERY_AST_SYSTEM.md`
- `26_DATABASE_QUERY_AST_NODE_MODEL.md`
- `27-34`
- `43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md`
- `44-54`
- `66_DATABASE_SQL_COMPILER_ARCHITECTURE.md`
- `67-75`

### Objetivo

Habilitar construccion y compilacion de queries basicas sin contaminar el builder con SQL directo.

### Codigo objetivo

- `src/Quantum/Database/Query`
- `src/Quantum/Database/Compiler`

### Entregables minimos

1. Query Model minimo
2. AST minimo
3. builders para `select`, `insert`, `update`, `delete`
4. compiler basico conectado a `Dialect`
5. paso de builder a execution

### Pruebas minimas

1. unit tests de AST y builder
2. unit tests de compiler
3. integration tests de consultas reales

### Criterio de salida

Existe un vertical completo `builder -> compiler -> execution` para operaciones basicas.

### Resultado del corte DV-DB-004

1. `QueryType`, `QueryMetadata` y contratos base de query.
2. Query Models minimos para `SELECT`, `INSERT`, `UPDATE` y `DELETE`.
3. AST minimo y `QueryAstFactory`.
4. `SqlCompiler` basico hacia `CompiledDatabaseCommand`.
5. `DatabaseQueryManager` y `SelectQueryBuilder` con API publica inicial.
6. ejecucion real del vertical completo contra SQLite.
7. pruebas unitarias y feature en verde para compiler y builder execution.

## Fase 5 - Schema y Migrations MVP

### Estado actual

- `Implementado en DV-DB-005`

### Documentos fuente principales

- `87_DATABASE_SCHEMA_ARCHITECTURE.md`
- `88-100`
- `101_DATABASE_MIGRATION_ARCHITECTURE.md`
- `102-111`

### Objetivo

Permitir evolucion de estructura con modelo, compiler y ejecucion coherentes con el resto del subsistema.

### Codigo objetivo

- `src/Quantum/Database/Schema`
- `src/Quantum/Database/Migration`

### Entregables minimos

1. schema model minimo
2. schema builder minimo
3. migration repository
4. migration discovery
5. migration runner
6. rollback simple

### Pruebas minimas

1. schema builder tests
2. migration repository tests
3. feature tests de migracion

### Criterio de salida

La base puede crearse y evolucionar mediante Database propio, no via SQL manual disperso.

### Resultado del corte DV-DB-005

1. `ColumnDefinition`, `CreateTableDefinition` y `DropTableDefinition`.
2. `TableBlueprint` y `ColumnBlueprint` como API minima de schema.
3. `SchemaCompiler` y `SchemaManager`.
4. `MigrationInterface`, `MigrationDiscovery`, `MigrationRepository` y `MigrationRunner`.
5. repositorio persistente de migraciones en `quantum_migrations`.
6. apply y rollback de migraciones desde `database/migrations`.
7. pruebas unitarias y feature en verde con SQLite real.

## Fase 6 - Transaction System minimo

### Estado actual

- `Implementado en DV-DB-006`

### Documentos fuente principales

- `164_DATABASE_TRANSACTION_ARCHITECTURE.md`
- `165-175`

### Objetivo

Separar claramente transaccion, request scope y execution.

### Codigo objetivo

- `src/Quantum/Database/Transaction`

### Entregables minimos

1. `TransactionManager`
2. `TransactionContext`
3. commit / rollback
4. nested transaction policy minima
5. rollback-only
6. integracion con execution y migration

### Pruebas minimas

1. unit tests de estados
2. integration tests de commit/rollback
3. tests de nested/savepoint cuando aplique

### Criterio de salida

La atomicidad deja de ser implicita y pasa a estar modelada por contratos del subsistema.

### Resultado del corte DV-DB-006

1. `TransactionManagerInterface`, `TransactionManager` y `TransactionContext`.
2. `TransactionId`, `TransactionState` y `TransactionException`.
3. commit, rollback y rollback-only.
4. nested transactions basadas en savepoints para SQLite.
5. rollback automatico de transacciones abiertas al cerrar el scope.
6. pruebas unitarias y feature en verde para contexto, commit/rollback y cleanup por scope.

## Fase 7 - API publica, CLI y Telemetria minima

### Estado actual

- `Implementado en DV-DB-007`

### Documentos fuente principales

- `216_DATABASE_TELEMETRY_ARCHITECTURE.md`
- `217-225`
- `302_DATABASE_PUBLIC_API_SYSTEM.md`
- `303_DATABASE_FACADE_SYSTEM.md`
- `304-310`

### Objetivo

Exponer una superficie usable sin romper las reglas del nucleo.

### Codigo objetivo

- `src/Quantum/Database/Support`
- `src/Quantum/Database/Integration`

### Entregables minimos

1. facade `DB` contextual
2. servicio publico `Database`
3. comandos CLI basicos:
   - status
   - migrate
   - rollback
4. senales minimas de telemetria

### Pruebas minimas

1. facade resolution tests
2. CLI feature tests
3. telemetry emission tests

### Criterio de salida

Database V1 es consumible por aplicacion, runtime y CLI con ergonomia minima viable.

### Resultado del corte DV-DB-007

1. `DatabaseInterface` y `Database` como surface publica tipada del subsistema.
2. `DatabaseStatus` como DTO publico minimo de diagnostico operacional.
3. facades `DB` y `Schema` resolviendo servicios scoped sin estado mutable estatico.
4. comandos `database:status`, `database:migrate` y `database:rollback`.
5. `DatabaseServiceProvider` exponiendo comandos y bindings publicos.
6. `ConsoleApplication` cargando comandos declarados por providers ya registrados.
7. `DatabaseTelemetryEmitter` e instrumentacion minima de query, transaction y migration sobre `Quantum/Telemetry`.
8. pruebas feature dedicadas para API publica/facades, CLI y telemetria.

## Fase 8 - ORM minimo posterior a V1

### Documentos fuente principales

- `112_DATABASE_ORM_ARCHITECTURE.md`
- `113-163`

### Objetivo

Abrir ORM solo despues de que el nucleo ya exista y sea utilizable.

### Codigo objetivo

- `src/Quantum/Database/ORM`
- `src/Quantum/Database/Hydration`

### Entregables minimos

1. metadata base
2. entity manager
3. repository base
4. hydration inicial
5. persistence planning inicial

### Estado

- `Implementado en DV-DB-008`

### Resultado del corte DV-DB-008

1. atributos ORM minimos `#[Entity]`, `#[Table]`, `#[Id]` y `#[Column]`.
2. metadata base con `EntityFieldMetadata`, `EntityMetadata` y `EntityMetadataRegistry`.
3. runtime ORM scoped con `IdentityMap`, `UnitOfWork` y `EntityManager`.
4. repository base (`EntityRepository`) y query layer (`EntityQuery`) reutilizando `DatabaseQueryManager`.
5. `Model` API minima resolviendo el `EntityManager` scoped desde `Application`.
6. extension de `DatabaseInterface` y `Database` con `entityManager()` y `repository(...)`.
7. integracion del vertical ORM en `DatabaseServiceProvider`.
8. prueba feature `DatabaseOrmFeatureTest` cubriendo metadata, persistencia, repository, model API e identity reuse sobre SQLite.

### Resultado del corte DV-DB-009

1. `TypeRegistry` ORM minimo y contrato `TypeHandlerInterface`.
2. handlers base para `int`, `float`, `bool`, `string`, `datetime_immutable`, `json` y `enum`.
3. `#[Column]` extendido con `type` y `enumType`.
4. metadata ORM resolviendo y preservando tipo de campo y enum class.
5. hydration y persistence usando conversion tipada consistente en lugar de casts ad hoc dispersos.
6. `EntityQuery::where(...)` convirtiendo criterios al valor de base de datos correspondiente.
7. `DatabaseServiceProvider` exponiendo `TypeRegistry` como singleton compartible.
8. `DatabaseOrmFeatureTest` ampliado con round-trip real de `DateTimeImmutable`, `BackedEnum` y JSON sobre SQLite.

## Fase 9 - ORM: Relaciones bidireccionales ManyToOne/OneToMany

### Documentos fuente principales

- `115_DATABASE_ENTITY_METADATA_SYSTEM.md`
- `116_DATABASE_ENTITY_MAPPING_SYSTEM.md`
- `118_DATABASE_ENTITY_MANAGER_SYSTEM.md`
- `120_DATABASE_ENTITY_QUERY_SYSTEM.md`

### Objetivo

Abrir el primer vertical minimo de relaciones ORM sin inventar un segundo runtime SQL: atributos declarativos, metadata canonica, construccion segura ante referencias cruzadas bidireccionales, helpers de carga to-one/to-many reutilizando `IdentityMap` y `DatabaseQueryManager`, y traduccion FK automatica en `EntityQuery`.

### Codigo objetivo

- `src/Quantum/Database/ORM/Attributes`
- `src/Quantum/Database/ORM/Metadata`
- `src/Quantum/Database/ORM` (EntityManager, EntityQuery)

### Estado actual

- `Implementado en DV-DB-010`

### Entregables minimos

1. atributos `ManyToOne` (owning: targetEntity, inversedBy, joinColumn, referencedColumn) y `OneToMany` (inverse: targetEntity, mappedBy).
2. VO `EntityAssociationMetadata` con kind constants, join mappings, helpers isOwningSide/isInverseSide.
3. lista de asociaciones y accessors `associations()`/`association()`/`hasAssociation()` en `EntityMetadata`.
4. `EntityMetadataRegistry::build()` en dos fases shell→final para evitar recursion infinita en relaciones bidireccionales.
5. helpers `EntityManager::loadToOne()` y `EntityManager::loadToMany()` reutilizando `IdentityMap` + `DatabaseQueryManager`.
6. traduccion automatica de asociaciones en `EntityQuery::where()` / `orderBy()` hacia columna FK y normalizacion entidad→identifier.
7. prueba feature bidireccional Post↔Comment validando metadata, FK, queries raw/ORM, loaders e IdentityMap.

### Pruebas minimas

1. metadata de asociaciones resuelta: kind, target, mappedBy/inversedBy, source/target column.
2. FK persistido correctamente via campo escalar `#[Column]` y visible en query raw.
3. `loadToOne` retorna misma instancia que IdentityMap.
4. `loadToMany` retorna lista correcta y asignable a propiedad inverse.
5. `EntityQuery` filtra por asociacion tanto con entidad managed como con ID.
6. regresion verde de los tests Database feature existentes.

### Criterio de salida

El ORM puede describir una relacion bidireccional ManyToOne/OneToMany con atributos declarativos, el usuario puede cargar ambos lados bajo demanda sin escribir SQL manual, y la base sigue usando el mismo `DatabaseQueryManager`, `TransactionManagerInterface` y scoped runtime sin segundo engine.

### Resultado del corte DV-DB-010

1. atributos `#[ManyToOne]` y `#[OneToMany]` como mapping declarativo de asociaciones.
2. `EntityAssociationMetadata` como descriptor VO canonico de asociacion (kind, target, join columns, mappedBy/inversedBy, owning/inverse helpers).
3. `EntityMetadata` extendido con lista de asociaciones y accessors `associations()`/`association()`/`hasAssociation()`/`hasField()`.
4. `EntityMetadataRegistry::build()` en dos fases: shell de fields/identifier primero (registrado en metadata), luego asociaciones ya con target metadata disponible, evitando recursion infinita bidireccional.
5. helpers `EntityManager::loadToOne()` y `EntityManager::loadToMany()` cargando asociaciones via find/query reutilizando `IdentityMap` y `DatabaseQueryManager`, asignando el resultado en la propiedad PHP via `ReflectionProperty`.
6. `EntityQuery::where()` y `orderBy()` traduciendo nombre de asociacion owning a columna FK y normalizando valor (object→identifier, scalar/string/null→si mismo).
7. prueba `test_bidirectional_many_to_one_and_one_to_many_load_and_query_over_sqlite` con fixtures `OrmBlogPost` ↔ `OrmBlogComment` cubriendo metadata, FK persist, raw vs ORM queries, loadToOne/loadToMany, IdentityMap y query por asociacion.
8. regresion en verde de `DatabasePublicApiFacadeTest`, `DatabaseQueryBuilderExecutionTest`, `DatabaseTransactionManagerTest` y `DatabaseOrmFeatureTest` (9 tests, 122 assertions) tras cerrar relaciones V1.

## Fase 10 - ORM: Value Objects Embedded Multi-columna V1

### Documentos fuente principales

- `115_DATABASE_ENTITY_METADATA_SYSTEM.md`
- `116_DATABASE_ENTITY_MAPPING_SYSTEM.md`
- `117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md`
- `118_DATABASE_ENTITY_MANAGER_SYSTEM.md`
- `120_DATABASE_ENTITY_QUERY_SYSTEM.md`

### Objetivo

Añadir soporte de primera clase para value objects incrustados multi-columna sobre el ORM scoped existente, sin abrir un segundo runtime SQL ni reescribir el pipeline de tipos: atributo declarativo `#[Embedded]`, metadata canonica de embedded y sus inner fields, integración en el build two-phase, hidratación nullable y extracción aplanada, traducción de paths anidados en EntityQuery, detección de suciedad merged-keys para nulificar VO completos, y pipeline de tipos unificado via `EntityTypedFieldInterface` para que handlers Scalar/Enum/DateTime/JSON funcionen igual sobre campos escalares que sobre inner fields de embedded.

### Codigo objetivo

- `src/Quantum/Database/ORM/Attributes/Embedded.php`
- `src/Quantum/Database/ORM/Metadata/EntityTypedFieldInterface.php`
- `src/Quantum/Database/ORM/Metadata/EntityEmbeddedMetadata.php`
- `src/Quantum/Database/ORM/Metadata/EntityEmbeddedFieldMetadata.php`
- `src/Quantum/Database/ORM/Metadata/EntityFieldMetadata.php` (refactor a interfaz)
- `src/Quantum/Database/ORM/Metadata/EntityMetadataRegistry.php` (embedded integration two-phase)
- `src/Quantum/Database/ORM/Metadata/EntityMetadata.php` (hydrate/extract/extractForWrite embedded-aware)
- `src/Quantum/Database/ORM/Types/Contracts/TypeHandlerInterface.php` (signature refactor)
- `src/Quantum/Database/ORM/Types/{Scalar,BackedEnum,DateTimeImmutable,Json}TypeHandler.php` (signature update)
- `src/Quantum/Database/ORM/EntityManager.php` (flushUpdate merged keys + embedded path resolution)
- `src/Quantum/Database/ORM/EntityQuery.php` (resolveEmbeddedPath where/orderBy)
- `tests/Feature/DatabaseOrmFeatureTest.php` (embedded VOs test)

### Estado actual

- `Implementado en DV-DB-011`

### Entregables minimos

1. atributo `#[Embedded(class: FQCN, prefix: ?string)]` sobre propiedades de entidad.
2. contrato compartido `EntityTypedFieldInterface` (`name/column/type/enumClass`) refactorizando TypeHandlerInterface + 4 built-in handlers.
3. VOs de metadata `EntityEmbeddedMetadata` (descriptor por embedded: class, prefix, reflection, innerFields map, nullable helpers) y `EntityEmbeddedFieldMetadata` (inner column con TypeHandler conversions).
4. `EntityMetadata` extendido con map `embeddeds` y accessors; `extract()` aplana inner fields como `embeddedName.innerName`; `extractForWrite()` escribe todas las inner columns (NULL si el VO entero es NULL); `hydrate()` reconstruye VO con estrategia any-non-null o asigna NULL.
5. `EntityMetadataRegistry::build()` two-phase extendido para recolectar `embeddedProperties` junto a `associationProperties`, parsear `#[Embedded]`, resolver inner `#[Column]` via TypeRegistry, aplicar prefix default `{prop}_` o explicito, y cablear todo en shell y final metadata.
6. `EntityManager::flushUpdate` con merged keys (`array_unique(array_merge(array_keys($current), array_keys($original)))`) y resolución scalar-or-embedded para detectar `embedded→NULL` y traducir paths anidados a columnas reales.
7. `EntityQuery::where()` / `orderBy()` con helper `resolveEmbeddedPath()` que traduce paths punteados (`price.amount`) a inner column y normaliza valor via `innerField->databaseValueFrom()`.

### Pruebas minimas

1. metadata assertions: prefix (default + explicito), columnas resueltas, listado `embeddeds()`.
2. INSERT y UPDATE multi-columna via EntityManager + flush.
3. inspección raw row confirmando escritura aplanada.
4. hidratación nullable: entidad sin columnas embedded hidrata NULL.
5. queries anidadas `where('price.currency', 'EUR')`, `where('price.amount', '<', 15000)`, `where('dimensions.depth', '50')` + nested `orderBy`.
6. update de inner fields tras `clear` + reload y nulificación completa del embedded VO (todas columnas NULL post-flush).
7. regresion verde PublicApiFacade + QueryBuilder + TransactionManager + ORM.

### Criterio de salida

El ORM puede mapear un value object multi-columna declarativamente via atributos, el usuario puede escribir queries con paths anidados como si fueran campos escalares (sin SQL manual), la nulificación de un VO completo produce NULLs en todas sus columnas, y el subsistema sigue reutilizando el mismo `DatabaseQueryManager`, `TransactionManagerInterface`, `TypeRegistry` y scoped runtime sin ningún engine paralelo ni estado mutable de proceso.

### Resultado del corte DV-DB-011

1. atributo `#[Embedded]` declarativo con class obligatorio y prefix opcional.
2. `EntityTypedFieldInterface` compartido; `EntityFieldMetadata` y `EntityEmbeddedFieldMetadata` ambos lo implementan; `TypeHandlerInterface` firma actualizada; 4 handlers built-in actualizados (Scalar/BackedEnum/DateTimeImmutable/Json) usando accessor methods.
3. `EntityEmbeddedMetadata` como descriptor immutable con `newEmbeddableInstance()` sin constructor, helpers de reflection property, map `innerFields<string, EntityEmbeddedFieldMetadata>` y estrategia de prefijo.
4. `EntityMetadata::extract()` con flattening `embeddedName.innerName` para dirty-compare; `extractForWrite()` escribe todas las inner columns (NULLs si VO es null); `hydrate()` usa any-non-null strategy: reconstruye VO si alguna inner column no es NULL, asigna PHP null en caso contrario.
5. `EntityMetadataRegistry::build()` two-phase recolecta embeddedProperties, parsea `#[Embedded]`, lee inner `#[Column]` fields via reflection con TypeRegistry, aplica prefix default `{prop}_` o explicito, inserta `EntityEmbeddedMetadata` en shell y final metadata.
6. `EntityManager::flushUpdate` reemplaza `foreach ($current as $field => $value)` por merged keys `array_unique(array_merge(array_keys($current), array_keys($original)))` con resolución condicional scalar vs embedded-dotted usando `hasField()`/`hasEmbedded()`; traduce cada campo a columna DB.
7. `EntityQuery::where()` y `orderBy()` detectan paths punteados; `resolveEmbeddedPath()` retorna tuple `[EntityEmbeddedFieldMetadata, column]`; aplica `databaseValueFrom()` si el valor.
8. feature test `test_embedded_value_objects_multicolumn_round_trip_and_query_over_sqlite` con fixtures `OrmMoney` (amount, currency) prefix default `price_`), `OrmDimensions` (width/height/depth) prefix explicito `dim_`), `OrmProduct`; cubriendo metadata, writes, raw-inspection, hydration NULL, nested where/orderBy, inner-field updates tras clear/reload, nulificacion complete de VO + IdentityMap consistency.
9. regresion en verde de `DatabasePublicApiFacadeTest`, `DatabaseQueryBuilderExecutionTest`, `DatabaseTransactionManagerTest` y `DatabaseOrmFeatureTest` (10 tests, 172 assertions) tras cerrar embedded V1.

## Fase 11 - ORM: Factories + Seeders minimo V1 + cierre DatabaseResult Countable/IteratorAggregate

### Documentos fuente principales

- `118_DATABASE_ENTITY_MANAGER_SYSTEM.md`
- `119_DATABASE_REPOSITORY_SYSTEM.md`
- `302_DATABASE_PUBLIC_API_SYSTEM.md`
- `304_DATABASE_CLI_COMMAND_INTEGRATION_SYSTEM.md`
- (bloque 18 Factories/Seeders Fixtures y bloque 31 DevX de DEVELOPMENT_MATRIX.md)

### Objetivo

Entregar la capa minima de generacion de datos (factories) y poblacion determinista (seeders) sobre el EntityManager scoped existente, SIN abrir runtime SQL paralelo, SIN engine de conexiones secundario, y respetando el lifetime discipline: discovery + registries = singleton-safe; runner + instancias = scoped. Como prerequisito habilitar el consumo ergonomico del result layer (fix estructural en DatabaseResult para Countable/IteratorAggregate) que era necesario para validar el round-trip via `count()` sobre result sets de SELECT en PDO SQLite (donde rowCount() = 0 es el comportamiento estandar).

### Codigo objetivo

- `src/Quantum/Database/Contracts/FactoryInterface.php`
- `src/Quantum/Database/Contracts/SeederInterface.php`
- `src/Quantum/Database/Factories/{AbstractFactory,DiscoveredFactory,FactoryDiscovery,FactoryRegistry}.php`
- `src/Quantum/Database/Seeders/{AbstractSeeder,DiscoveredSeeder,SeederDiscovery,SeederRunner}.php`
- `src/Quantum/Console/Commands/DatabaseSeedCommand.php`
- `src/Quantum/Database/Integration/DatabaseServiceProvider.php` (bindings singletons/scoped + comando registrado)
- `src/Quantum/Database/Execution/DatabaseResult.php` (implements Countable, IteratorAggregate)

### Estado actual

- `Implementado en DV-DB-012`

### Entregables minimos

1. Contracts: `FactoryInterface::definition/entityClass/times/make/create`, `SeederInterface::run(Application)`.
2. Abstract Factory base con `times(int)` fluent, `make()` sin persist, `create()` = persist via EM SIN flush implicito; instanciacion via `newInstanceWithoutConstructor` + `setValue` sobre `definition()` (valores de dominio, no raw DB).
3. Abstract Seeder base con `call(SeederInterface|class-string)` sub-seeder recursivo, `factory(Entity)` via FactoryRegistry, `flush()` al EntityManager scoped, `setApplication(Application)` publico.
4. Discovery classes con 3 return-shapes por archivo PHP: instancia objeto / class-string / `callable(Application): object` (habilita clases anonimas en tests que reciben app).
5. `FactoryRegistry` singleton-safe con eager discovery, duplicate guard, `get(EntityClass): FactoryInterface`.
6. `SeederRunner` scoped: `begin()` transaction via `TransactionManagerInterface`, `run(Seeder)` invoca `$seeder->run()`, unico `$em->flush()` final, `commit()` o `rollback()` on Throwable.
7. Comando CLI `database:seed --class=X --path=database/seeders` registrado en provider; abre/cierra su propio scope via `ScopeManager` idempotentemente al igual que status/migrate/rollback.
8. Binding graph en `DatabaseServiceProvider`: singletons `FactoryDiscovery`, `FactoryRegistry`, `SeederDiscovery`; scoped `SeederRunner`; comando agregado a `commands()`.
9. Bugfix estructural `DatabaseResult implements \Countable, \IteratorAggregate<int, array<string, mixed>>`: `count()` retorna `count(rows)` cuando type=Rows y `affectedRows` cuando type=Affected; `getIterator()` retorna `ArrayIterator(rows)`. Resuelve el caso PDO SQLite SELECT donde rowCount()=0 y los consumidores usaban `count($result)` = 0 aun cuando `$result->rows` tenia datos.

### Pruebas minimas

1. Feature test end-to-end `DatabaseFactoriesSeedersFeatureTest` sobre SQLite temp-dir con:
   - conexion path assertion,
   - `extractForWrite` 4-column check (Hyp A initial),
   - ArticleFactory + DatabaseSeeder escritos on-fly como callable(Application) en temp-dirs,
   - FactoryRegistry lookup por entity class,
   - `make()` sin id (no persist),
   - baseline manual persist 3 rows raw + read-back via `table()->get()` count,
   - `ArticleFactory::times(5)->create()` + flush → 8 total,
   - clear EM + SeederRunner via discovery → 16 total,
   - `where(published=false, views=0)` count=3,
   - published `where + orderBy` count=13,
   - repository `findAll()` count=16.
2. Lint PHP sintactico 14/14 archivos tocados OK.
3. Regresion subset verde: PublicApiFacade + QueryBuilder + TransactionManager + ORM + Console + FactoriesSeeders (12 tests / 210 assertions).

### Criterio de salida

El usuario puede declarar factories y seeders en `database/factories/` y `database/seeders/` (convención skeleton), descubrirlos automaticamente, generar entidades con atributos de dominio via `times(n)->create()`, correr seeders transaccionalmente via CLI `database:seed`, todo usando el MISMO `EntityManager`, `DatabaseQueryManager` y `TransactionManagerInterface` scoped ya existentes; NO se abre un segundo SQL engine ni un segundo sistema de conexiones. `DatabaseResult` responde correctamente a `count()` y `foreach` independientemente del valor que retorne `PDOStatement::rowCount()` en SELECTs SQLite.

### Resultado del corte DV-DB-012

1. Contracts publicos `FactoryInterface` y `SeederInterface` con surface minima estable.
2. `AbstractFactory` con fluent `times(n)`, `make()` sin persist, `create()` persist sin flush; reflection-based hidratacion sobre atributos de dominio (enums, embedded VOs, DateTimeImmutable — TypeRegistry se encarga en flush).
3. `AbstractSeeder` con `call()` recursivo, `factory(Entity)` via registry, `flush()` delegado al EM scoped; `setApplication(Application)` public para evitar protected-access error en clases anonimas.
4. `FactoryDiscovery` y `SeederDiscovery` con 3 return-shapes (instancia/class-string/callable(Application)).
5. `FactoryRegistry` singleton-safe con eager discovery + duplicate FQCN guard + lookup por entityClass.
6. `SeederRunner` scoped con transaction begin → run seeders → single batched flush → commit; rollback completo on Throwable.
7. `DatabaseSeedCommand` CLI con opciones `--class` y `--path`; scope propio via `beginScope/endScope`.
8. `DatabaseServiceProvider` actualizado: singletons FactoryDiscovery/FactoryRegistry/SeederDiscovery; scoped SeederRunner; comando agregado a `commands()` array (4to comando DB CLI).
9. Bugfix estructural `DatabaseResult` implementando Countable y IteratorAggregate, resolviendo el bug 0-rows en lecturas post-persist (PDO SQLite SELECT rowCount=0 vs rows[] poblado).
10. Suite pruebas: `DatabaseFactoriesSeedersFeatureTest` 1 test / 32 assertions GREEN; lint 14/14 archivos OK; regresion subset 12 tests / 210 assertions GREEN.

### Avance por bloques tras la fase

- Bloque 18 (Factories/Seeders/Fixtures): Pendiente → Parcial (contracts, runners, CLI min, discovery; quedan ergonomia, estados, CLI rico - Prioridad 7).
- Bloque 29 (Testing): DatabaseFactoriesSeedersFeatureTest agregada; regresion = 12 tests / 210 assertions (subset DB vertical).
- Bloque 31 (DevX): comando `database:seed` agregado; surface factories/seeders disponible para aplicación/CLI.
- Bloque 32 (Integrations): bindings provider singletons/scoped + bugfix DatabaseResult Countable/IteratorAggregate.
- Siguiente bloque recomendado (reordenado post-cierre): 1) Repository DI tipado y helpers ergonomicos, 2) Lifecycle callbacks + policies de persistencia (cascade/orphan-removal min), 3) Relationships ampliados (OneToOne, ManyToMany, proxies lazy/eager joins declarativos), 4) Factories/Seeders ampliados (Prioridad 7).

## Fase 12 - ORM: Repository Factory con DI tipado + helpers ergonomicos minimo V1

### Documentos fuente principales

- `118_DATABASE_ENTITY_MANAGER_SYSTEM.md`
- `119_DATABASE_REPOSITORY_SYSTEM.md`
- `302_DATABASE_PUBLIC_API_SYSTEM.md`
- `304_DATABASE_CLI_COMMAND_INTEGRATION_SYSTEM.md`
- (bloque 10 ORM, bloque 11 Repos, bloque 31 DevX y bloque 32 Integrations de DEVELOPMENT_MATRIX.md)

### Objetivo

Entregar surface consumible por DI tipado para repositorios ORM (inyectar `RepositoryFactoryInterface` por constructor en servicios/controladores sin depender de `EntityManager` ni de `Application::make()` externo). Habilitar dos convenciones de binding declarativo (complementarias, sin colisión): `#[Entity(repository: X)]` sobre entidad (ya existente) + `#[RepositoryFor(Entity)]` sobre custom repositorio (nuevo, útil para bounded contexts separados donde no se puede tocar el código fuente de la entidad). Añadir helpers ergonomicos mínimos a EntityRepository sin abrir runtime SQL paralelo: save/delete/count/exists + accessors tipados. Fundamentar los aggregators count() sobre la infraestructura Countable de DatabaseResult habilitada en DV-DB-012 (cross-driver safe, sin riesgo de quoting identificador erróneo en SqlCompiler).

### Codigo objetivo

- `src/Quantum/Database/ORM/Contracts/RepositoryFactoryInterface.php`
- `src/Quantum/Database/ORM/Attributes/RepositoryFor.php`
- `src/Quantum/Database/ORM/CustomRepositoryRegistry.php`
- `src/Quantum/Database/ORM/EntityRepositoryFactory.php`
- `src/Quantum/Database/ORM/EntityRepository.php` (upgrade: accessors + save/delete/count/exists)
- `src/Quantum/Database/ORM/EntityQuery.php` (count() forward)
- `src/Quantum/Database/Query/Builder/SelectQueryBuilder.php` (count() aggregator via DatabaseResult Countable)
- `src/Quantum/Database/ORM/Metadata/EntityMetadataRegistry.php` (upgrade: 2nd constructor param + resolveRepositoryClass helper dual-source en shell/final)
- `src/Quantum/Database/Contracts/DatabaseInterface.php` (shortcut repositoryFactory method)
- `src/Quantum/Database/Database.php` (9th constructor param + accessor)
- `src/Quantum/Database/Integration/DatabaseServiceProvider.php` (bindings lifetime-correctos + wiring)

### Estado actual

- `Implementado en DV-DB-013`

### Entregables minimos

1. Contract público DI tipado: `RepositoryFactoryInterface::repositoryFor(string $entityClass): EntityRepositoryInterface`.
2. Atributo declarativo `#[RepositoryFor(EntityClass::class)]` sobre clases repositorio custom; FQCN en readonly property.
3. `CustomRepositoryRegistry` singleton-safe: dual convention support (attribute discovery + explicit `register(repoClass,?entityClass)`); duplicate-entity guard RuntimeException; class existence + interface type-checks.
4. `EntityMetadataRegistry` upgrade: 2nd constructor param nullable `?CustomRepositoryRegistry`; `build()` en dos fases (shell + final) ambos invocan `resolveRepositoryClass(entityClass, attrRepo)`; helper privado prioridad fija: `#[Entity(repository: X)]` explicito primero → fallback `CustomRepositoryRegistry::repositoryFor(entity)` si no hay attr repo.
5. `EntityRepositoryFactory` SCOPED (NUNCA singleton): implementa RepositoryFactoryInterface, depende del EntityManagerInterface scoped actualmente activo; delega 100% a `EntityManager::repository(entityClass)` preservando la cache única interna del EntityManager (sin doble cache, sin stale instances).
6. EntityRepository upgrade ergonomics: `getEntityManager(): EntityManager`, `getMetadata(): EntityMetadata`, `getEntityClass(): string` accessors; `save(object $entity, bool $flush = false): void` con RuntimeException guard si `!$entity instanceof expectedClass` (persist via manager + optional flush); `delete(object $entity, bool $flush = false): void` mismo guard + remove + optional flush; `count(array $criteria = []): int` apply criteria via where luego delegates a EntityQuery count; `exists(array $criteria): bool` via count>0.
7. Aggregator `SelectQueryBuilder::count(?string $column = null): int`: builder clonado con wherePredicates transferidos; columna específica agrega `where column != NULL`; retorna `count($builder->get())` sobre DatabaseResult Countable; NO usa select(expr AS aggregate) que SqlCompiler envuelve en comillas como identificador (causaba 0 rows en v1).
8. Aggregator forward `EntityQuery::count(?string $column = null): int { return $this->query->count($column); }`.
9. Surface público: `DatabaseInterface::repositoryFactory(): RepositoryFactoryInterface` declarado; `Database` class noveno constructor param `RepositoryFactoryInterface $repositoryFactory`; accessor retorna la instancia (no crea nueva).
10. Bindings lifetime-correctos en `DatabaseServiceProvider`: imports `RepositoryFactoryInterface/CustomRepositoryRegistry/EntityRepositoryFactory`; singleton `CustomRepositoryRegistry::class`; EntityMetadataRegistry singleton ahora wired pasándole CustomRegistry como 2nd arg; scoped `EntityRepositoryFactory(EntityManagerInterface)`; bind `RepositoryFactoryInterface → EntityRepositoryFactory::class`; Database::class scoped constructor ahora recibe 9º arg `RepositoryFactoryInterface`.
11. Fix: `SelectQueryBuilder::count()` v1 usaba `select("COUNT(*) AS aggregate")` → SqlCompiler `quoteIdentifierPath()` lo envolvía en comillas como identificador SQL → retornaba 0 aunque rows estaban insertadas; corregido a `count($clonedBuilder->get())` sobre DatabaseResult Countable (cross-driver safe, reutiliza fix estructural DV-DB-012).
12. Simplificación: `EntityRepository::count()` v1 tenía código muerto `if (is_countable($result)) return count($result);` y `return count($query->get())` fallback; eliminados porque EntityQuery::count() ya retorna int canónico (forward a SelectQueryBuilder count que retorna int).

### Pruebas minimas

1. Feature test end-to-end `test_repository_factory_with_di_typed_and_ergonomic_helpers` dentro de DatabaseOrmFeatureTest sobre SQLite temp-dir con:
   - `CustomRepositoryRegistry::register(OrmProductRepository::class)` explícito + metadata assertion `repositoryClass === OrmProductRepository::class` (prueba resolución dual-source: entity no tenía `#[Entity(repository)]`, así que vino del registry),
   - DI via `$app->make(RepositoryFactoryInterface::class)` (no service locator manual — el binding del provider es el que hace la magia),
   - `OrmUser` resolución custom repo (tenía `#[Entity(repository: OrmUserRepository::class)]` desde DV-DB-008): `assertInstanceOf(OrmUserRepository)`, `getEntityClass() === OrmUser::class`, `getEntityManager() === $database->entityManager()`,
   - `$database->repositoryFactory()` shortcut: `assertSame($factory, $dbRepoFactory)` (misma instancia scoped, no nueva),
   - `OrmTag` repo default (sin attr ni registry entry): `assertInstanceOf(EntityRepository::class)` genérico,
   - save pattern: dos `save()` sin flush + tercero `save($hidden, flush: true)` → triple persist; assertNotNull de los tres ids (flush generó IDs autoincrement),
   - raw count: `$db->table('orm_tags')->count() === 3`,
   - repository `count() === 3`, `exists(['slug'=>'php']) === true`, `exists(['slug'=>'missing']) === false`,
   - criteria counts: `count(['visible'=>true]) === 2`, `count(['visible'=>false]) === 1`,
   - `EntityQuery::where('visible', true)->count() === 2`,
   - `findOneBy(['slug'=>'php'])` retorna OrmTag con name === 'PHP',
   - `delete($hidden, flush: true) + count() === 2 + exists('hidden-draft') === false`,
   - `findBy([], ['name' => 'asc'])` retorna Framework antes que PHP alfabéticamente,
   - CustomRegistry ya poblado → `$factory->repositoryFor(OrmProduct::class)` retorna `OrmProductRepository` instancia con método custom `label() === 'sku-based-lookup'`,
   - todo ejecutado dentro de `ScopeManager begin()/end()` idéntico al runtime HTTP real.
2. Lint PHP sintáctico 13/13 archivos tocados OK (contracts, registry, factory, upgrades EntityRepository/EntityQuery/SelectQueryBuilder/EntityMetadataRegistry/DatabaseInterface/Database/DatabaseServiceProvider) + DatabaseConsoleCommandsTest (todavía tocado en 012) + DatabaseOrmFeatureTest (fixtures OrmTag/OrmProductRepository añadidos al final del archivo).
3. Regresión subset GREEN Database vertical completo: DatabasePublicApiFacadeTest + DatabaseQueryBuilderExecutionTest + DatabaseTransactionManagerTest + DatabaseOrmFeatureTest + DatabaseConsoleCommandsTest + DatabaseFactoriesSeedersFeatureTest (21 tests, 280 assertions post-DV-DB-013 sobre SQLite).

### Criterio de salida

Los consumidores declaran `RepositoryFactoryInterface` como dependencia de constructor (DI tipada), obtienen repositorios canónicos sin depender de EntityManager ni de Application::make() externo; las dos convenciones de binding declarativo coexisten sin colisión; save/delete/count/exists delegan 100% al runtime existente sin SQL paralelo; EntityRepositoryFactory permanece scoped (nunca stale EntityManager); SelectQueryBuilder::count() funciona cross-driver sin riesgo de quoting por SqlCompiler; y toda la regresión Database vertical está GREEN sin regresiones.

### Resultado del corte DV-DB-013

1. Contract público `RepositoryFactoryInterface` con surface canónica `repositoryFor(entityClass)` consumible por DI tipado.
2. Atributo declarativo `#[RepositoryFor(EntityClass::class)]` TARGET_CLASS, FQCN readonly.
3. `CustomRepositoryRegistry` singleton-safe: dual mode attribute + explicit; duplicate guard + class/type checks.
4. `EntityMetadataRegistry` upgrade: 2nd nullable constructor param + resolveRepositoryClass en ambos shell/final → dual source priority Entity attr → CustomRegistry fallback.
5. `EntityRepositoryFactory` scoped: 100% delegation EM->repository preservando cache única sin duplicación.
6. EntityRepository: 3 accessors + save/delete con RuntimeException class-guard + count/exists sobre EntityQuery.
7. `SelectQueryBuilder::count()` fixed: cloned builder + wherePredicates + count(DatabaseResult get()) (v1 fix de 0 filas por quoting aggregate).
8. `EntityQuery::count()` forward a SelectQueryBuilder count().
9. Surface público: `DatabaseInterface` y `Database` ambos con shortcut `repositoryFactory()` / 9º constructor param scoped.
10. `DatabaseServiceProvider` bindings lifetime-correctos: singleton CustomRegistry, scoped EntityRepositoryFactory, interface bind, wiring EntityMetadataRegistry + Database constructor args.
11. Simplification `EntityRepository::count()` código muerto eliminado (EntityQuery count retorna int).
12. Suite pruebas: DatabaseOrmFeatureTest nueva subprueba 70 aserciones GREEN; lint 13+2 archivos OK; regresión subset vertical Database (21 tests / 280 assertions GREEN).

### Avance por bloques tras la fase

- Bloque 10 (ORM): Parcial se mantiene (faltan OneToOne/ManyToMany, proxies/lazy, hydration compilada, etc.); evidencia visible se amplía drásticamente con RepositoryFactory/#[RepositoryFor]/CustomRegistry/EntityRepositoryFactory/helpers save/delete/count/exists/aggregators count/wiring provider.
- Bloque 11 (Identity Map / Unit of Work / Persistence): Parcial mantiene; evidencia añadida de EntityRepository save/delete delegando persist/remove/flush sobre UoW existente con class-guard.
- Bloque 29 (Testing): evidencia actualizada a 21 tests / 280 assertions post-013; subprueba 70 aserciones Repository Factory DI ergonomics documentada.
- Bloque 31 (Developer Experience / API Pública): Operativa mantiene; agregados shortcuts Database::repositoryFactory(), atributo #[RepositoryFor], helpers EntityRepository save/delete/count/exists, aggregators EntityQuery/SelectQueryBuilder count, CustomRegistry registro explícito para testing/manifiestos.
- Bloque 32 (Integrations): Operativa mantiene; bindings lifetime-correctos documentados (CustomRegistry singleton, EntityRepositoryFactory scoped wired a EM, Database 9º constructor param); fix quoting count v1 documentado.
- Siguiente bloque recomendado (reordenado post-cierre DV-DB-013): 1) Lifecycle callbacks/events + policies persistencia (cascade persist/remove min + orphan removal básico) sobre UnitOfWork flush pipeline; 2) Relationships ampliados (OneToOne bidireccional, ManyToMany join-table, opcional eager joins declarativos sobre SelectQueryBuilder); 3) Factories & Seeders ampliados (comandos make:factory/make:seeder, factory states/sequences, seeder dependencies DAG ordering, SeederRunner progress/logger); 4) Repositories avanzados (manifests extensibles discovery, interface bindings por entidad container, Criteria API typed, codegen make:repository).

## Roadmap resumido

1. Fase 1: Bootstrap, Config y Runtime Scope
2. Fase 2: Driver, Connection, Platform y Dialect
3. Fase 3: Execution Engine minimo
4. Fase 4: Query MVP
5. Fase 5: Schema y Migrations MVP
6. Fase 6: Transaction System minimo
7. Fase 7: API publica, CLI y Telemetria minima
8. Fase 8: ORM minimo posterior a V1
9. Fase 9: ORM relaciones bidireccionales ManyToOne/OneToMany V1
10. Fase 10: ORM Value Objects Embedded Multi-columna V1
11. Fase 11: ORM Factories + Seeders minimo V1 + cierre DatabaseResult Countable/IteratorAggregate
12. Fase 12: ORM Repository Factory con DI tipado + helpers ergonomicos minimo V1 (RepositoryFactoryInterface/RepositoryFor/CustomRepositoryRegistry/EntityRepositoryFactory/EntityRepository save/delete/count/exists/aggregators count/wiring provider)

## Regla de control ejecutivo

Ninguna fase se considera cerrada si:

1. no existe evidencia en codigo,
2. no existe prueba relevante,
3. no se actualiza `DEVELOPMENT_MATRIX.md`,
4. no se registra el corte en `DEVELOPMENT_VERSIONS.md`.

## Regla de recalibracion

Si durante la implementacion se descubre que una fase estaba sobreestimada o subestimada, se debe:

1. ajustar este plan,
2. reflejar el cambio en la matriz,
3. registrar la recalibracion en la bitacora de versiones.
