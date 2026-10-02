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
- Siguiente bloque recomendado (reordenado post-cierre DV-DB-013): 1) Lifecycle callbacks/events + policies persistencia (cascade persist/remove min + orphan removal básico) sobre UnitOfWork flush pipeline — **IMPLEMENTADO en Fase 13 / DV-DB-014 (justo después)**; 2) Relationships ampliados (OneToOne bidireccional, ManyToMany join-table, opcional eager joins declarativos sobre SelectQueryBuilder); 3) Factories & Seeders ampliados (comandos make:factory/make:seeder, factory states/sequences, seeder dependencies DAG ordering, SeederRunner progress/logger); 4) Repositories avanzados (manifests extensibles discovery, interface bindings por entidad container, Criteria API typed, codegen make:repository); 5) Cache de consultas / second-level cache opcional + V2 lifecycle listeners con DI Container reemplazando `new $className()` V1.

## Fase 13 - ORM: Lifecycle Callbacks + Cascade (PERSIST/REMOVE) + OrphanRemoval V1

### Documentos fuente principales

- Bloques 10 (ORM), 11 (IdentityMap / UnitOfWork / Persistence Policies), 20 (Events).
- DEVELOPMENT_MATRIX.md filas 10 / 11 / 20.
- DEVELOPMENT_GUIDELINES.md sección `#### Persistence Policies (Lifecycle / Cascade / OrphanRemoval) — Hard rules desde DV-DB-014`.
- Plan oficial aprobado por usuario: `.trae/documents/DV-DB-014_plan.md` (14/09/2026 actualización post 013).

### Objetivo

Cerrar la capa de `Persistence Policies` del ORM UnitOfWork V1 para cubrir los policy hooks estándar del ciclo de vida de entidades (lifecycle canónico 7 eventos method-level), operaciones de propagación `Cascade` mínimas (PERSIST + REMOVE) sobre relaciones ManyToOne/OneToMany, y `OrphanRemoval` básico sobre asociaciones inversas OneToMany; incluyendo upgrade del pipeline `EntityManager::flush()` (orden topológico inserts parent→child, FK auto-populate owning-side, BFS anti-circular para cascades, snapshots de colecciones para orphan detection) — **TODO 100% compatible con entidades legacy sin atributos nuevos, sin nuevas dependencias Composer, sin ampliar constructor EntityManager ni bindings en DatabaseServiceProvider**.

### Entregables mínimos

1. **7 atributos lifecycle method-level TARGET_METHOD**: `PrePersist`, `PostPersist`, `PreUpdate`, `PostUpdate`, `PreRemove`, `PostRemove`, `PostLoad` — 1 FQCN `vendor/voltstack/framework/src/Quantum/Database/ORM/Attributes/{event}.php` cada uno.
2. **Contracts listeners + Cascade**:
   - `Contracts\Cascade` con constantes string `PERSIST/REMOVE/MERGE/DETACH/REFRESH` + `ALL = [PERSIST,REMOVE,MERGE,DETACH,REFRESH]` (V1: PERSIST y REMOVE con wireado activo; MERGE/DETACH/REFRESH aceptados sin wireado = no-op safe).
   - `Contracts\EntityLifecycleListenerInterface`: métodos abstract `prePersist/postPersist/preUpdate/postUpdate/preRemove/postRemove/postLoad($entity, EntityManagerInterface $em, array $context = [])` sin default bodies (PHP interface no soporta).
   - `Contracts\AbstractEntityLifecycleListener`: conveniencia implementa la interface con 7 default bodies vacíos; consumer code extiende esta clase y sobreescribe sólo los hooks que necesita.
3. **Atributos Entity / ManyToOne / OneToMany upgrade (backward compat 100% defaults cero)**:
   - `#[Entity(...)]` → nuevo trailing named arg `array $lifecycleListeners = []` (lista FQCN class listeners).
   - `#[ManyToOne(...)]` → nuevo trailing named arg `array $cascade = []` (valores `Cascade::*`).
   - `#[OneToMany(...)]` → nuevos trailing named args: `array $cascade = []` + `bool $orphanRemoval = false`.
4. **Metadata shape upgrade**:
   - `EntityAssociationMetadata` → constructor trailing defaults: `array $cascade = []`, `bool $orphanRemoval = false`; helpers `cascadesPersist(): bool`, `cascadesRemove(): bool`, `isManyToOne(): bool`, `isOneToMany(): bool`.
   - `EntityMetadata` → shape estricto `lifecycleCallbacks: array<7 keys>` cada valor `list<Closure>`; constante pública `KNOWN_LIFECYCLE_EVENTS = ['prePersist','postPersist','preUpdate','postUpdate','preRemove','postRemove','postLoad']`; `DEFAULT_LIFECYCLE_CALLBACKS = array_fill_keys(KNOWN_LIFECYCLE_EVENTS, [])`; helpers `hasCallbacks(string $event): bool` y `callbacksFor(string $event): list<Closure>` (validation RuntimeException para eventos unknown); constructor accept named arg `lifecycleCallbacks = self::DEFAULT_LIFECYCLE_CALLBACKS` para que positional calls antiguas sigan funcionando.
5. **EntityMetadataRegistry upgrade build two-phase**:
   - imports 7 atributos lifecycle + EntityLifecycleListenerInterface + ReflectionMethod + Closure;
   - nuevo helper privado `lifecycleAttributeMap(): array<attr-fqcn, event>`;
   - nuevo helper privado `buildLifecycleCallbacks(ReflectionClass, ?array classListeners): array<7 events, list<Closure>>` dos fases: (a) method-level: iterar ReflectionMethod del entity → getAttributes(attrClass IS_INSTANCEOF) → setAccessible(true) → Closure wrapper `static fn($entity,$em,$ctx) => $methodRef->invoke($entity)`; (b) class-level listeners: por cada FQCN string → `class_exists()` RuntimeException → `is_subclass_of($class, EntityLifecycleListenerInterface::class)` RuntimeException → `new $class()` SIN constructor args SIN Container → registra 7 Closures por event que invocan `$instance->$event($entity, $em, $ctx)`;
   - `build()` shell y final ambos pasan `lifecycleCallbacks` named arg a `new EntityMetadata(...)`;
   - `mapAssociation()` para ManyToOne/OneToMany pasa `cascade` y `orphanRemoval` desde los Attribute instances instanciados.
6. **UnitOfWork upgrade snapshots inverse OneToMany para OrphanRemoval**:
   - import `Metadata\EntityAssociationMetadata` (fix TypeError old-NS vs Metadata\);
   - nueva property privado scoped `array $originalCollections = []` (key: spl_object_id, subkey: associationName, value: list<int> oid collection items);
   - `registerManaged($entity, $metadata, $key)` y `synchronize($entity, $metadata)` llaman `snapshotOneToManyCollections($entity, $metadata)`;
   - `clear()` limpia `originalCollections = []`; `detach($entity)` elimina `unset($this->originalCollections[spl_object_id($entity)])`;
   - API pública: `snapshotOneToManyCollections(object $entity, EntityMetadata $metadata): void`;
   - API pública: `collectionDiff(object $entity, string $associationName): array{removed: list<object>, added: list<object>}`;
   - helpers privados: `readOneToManyCollection(EntityAssociationMetadata, object): iterable<object>` y `collectObjectIdsFromCollection(iterable): list<int>`.
7. **EntityManager flush pipeline upgrade (Paso 0-4 reordenados) — BACKWARD COMPAT entidades sin callbacks no ejecutan nada**:
   - import `Contracts\Cascade`;
   - `hydrateManaged($key, $metadata, $row)` → después registerManaged llama `dispatchLifecycle('postLoad', $entity)` (SÓLO cuando materializó entidad nueva: si ya estaba en identityMap NO ejecuta postLoad = no cache hit repetido);
   - método `flush()` orden nuevo:
     1. `applyCascadesBeforeFlush(Cascade::PERSIST)` — BFS por todas entidades NEW/MANAGED → si asociación cascadesPersist → si target no persistido aún → persist(target); con visited array `spl_object_id` anti-circular infinite loop;
     2. `applyCascadesBeforeFlush(Cascade::REMOVE)` — mismo BFS para cascade REMOVE;
     3. `collectOrphansForRemoval()` — iterar Managed entities, sus associations OneToMany con orphanRemoval=true → collectionDiff → removed items que sean Managed (Managed-only hard constraint) → `$this->remove($orphan)`;
     4. `flushNewEntitiesInDependencyOrder()` — reemplaza iteración `newEntities()` raw; repeat-pass stall-guard orden topológico: iterar NEW → `newEntityIsInsertable()` = ManyToOne targets todos tienen id → `flushInsert($entity)`; si pass sin avance y quedan NEW entities → RuntimeException circular reference; luego flushManaged/flushRemoved como antes;
   - `flushInsert($entity, $metadata, $key)` orden nuevo: `dispatchLifecycle('prePersist', $entity)` → `populateManyToOneForeignKeys($entity, $metadata)` → INSERT SQL → `$this->unitOfWork->synchronize(...)` → `dispatchLifecycle('postPersist', $entity)`;
   - `flushUpdate($entity, $metadata, $key, $snapshot)` orden nuevo: `populateManyToOneForeignKeys($entity, $metadata)` → calcular `$currentValues = extract`, `$changes = array_diff_assoc($currentValues, $snapshot)` → `dispatchLifecycle('preUpdate', $entity, ['changes'=>$changes,'currentValues'=>$currentValues,'originalSnapshot'=>$snapshot])` → UPDATE SQL → sincronize → `dispatchLifecycle('postUpdate', $entity, ['changes'=>$changes,'currentValues'=>$currentValues])`;
   - `flushDelete($entity, $metadata, $key)` orden nuevo: `dispatchLifecycle('preRemove', $entity)` → DELETE SQL → `dispatchLifecycle('postRemove', $entity)` → `$this->unitOfWork->detach($entity)` (detach DESPUÉS de postRemove no antes);
   - **Nuevos métodos privados**: `dispatchLifecycle(string $event, object $entity, array $context = [])` (iterar `callbacksFor($event)` closures), `applyCascadesBeforeFlush(string $op)` BFS visited, `collectOrphansForRemoval()` Managed-only, `readAssociationTargets()`, `flushNewEntitiesInDependencyOrder()` stall-guard, `newEntityIsInsertable()`, `populateManyToOneForeignKeys($entity, $metadata)` (mismo naming convention que associationSourceValue: sourceFieldId → match sourceColumn mappedFields → sourceField directo), `readSingleAssociationTarget()` y `assignOwningSideForeignKey()`.
8. **Suite pruebas GREEN obligatorias**:
   - Unit `DVDB014LifecycleCascadeOrphanTest.php` ≥ 12 tests exit_code 0 (≥ 60 assertions): EntityMetadata 7 defaults shape, unknown-event validation, callbacksFor/hasCallbacks, EntityAssociation cascade/orphan defaults, Cascade::ALL flags, Registry 7 method-level discovery, dispatch closures reflection, invalid listeners (not exists / not implements), class listener 7 events + context propagation, backward compat BareSampleEntity 0 callbacks default assoc.
   - Feature `DatabaseOrmFeatureTest.php` nueva subprueba 7 `test_lifecycle_callbacks_cascade_and_orphan_removal_over_sqlite` ≥ 60 assertions sobre SQLite temp con fixtures: `OrmCascadePost #[Table('orm_cascade_posts')]` / `OrmCascadeComment #[Table('orm_cascade_comments')]` OneToMany bidireccional (cascade PERSIST+REMOVE + orphanRemoval=true) / `OrmLifecyclePost #[Table('orm_timestamped_posts')]` (method-level #[PrePersist] set createdAt, lifecycleListeners [OrmLifecyclePostListener extends Abstract set updatedAt/updatedBy]). Escenarios mínimos: metadata sanity cascade/orphan/callbacks count/hasCallbacks event, cascade persist padre + 3 comments via solo persist padre, orphan removal unset 1 comment = 2 remain DB, lifecycle method+class timestamp + updatedBy no nulos, PreUpdate context has changes.title/current/original, cascade REMOVE padre = 0 comments, postLoad find after em clear forced fresh hydration, backward compat OrmTag/OrmUser sin attrs = 0 callbacks 7 events + asociaciones default cascade/orphan false.
   - Regresión vertical Database ORM Feature existente (bidirectional, types round-trip, model API, entity_manager, embedded V1, repository factory V1) + Unit DV014 = ≥ 21 tests / 280 assertions GREEN sin regresiones.
9. **Lint PHP 25+ archivos tocados**: `php -l` todos los archivos nuevos + modificados sin errores sintácticos; `composer dump-autoload` regenera classmap para AbstractEntityLifecycleListener nuevo archivo.

### Pruebas mínimas

1. Unit `DVDB014LifecycleCascadeOrphanTest` 14 tests/119 assertions (≥12 objetivo min):
   - `Entity metadata defaults lifecycle callbacks to seven empty arrays` — sin pasar lifecycleCallbacks a metadata constructor, shape = 7 arrays vacíos (backward positional legacy).
   - `Entity metadata rejects unknown lifecycle event keys` — pasando key 'prePersistInvalid' → RuntimeException.
   - `Callbacks for rejects unknown event name` — callbacksFor('bogusEvent') → RuntimeException.
   - `Has callbacks and callbacks for when populated` — shape con 3 callbacks prePersist → has true; otro event false.
   - `Association metadata cascade and orphan defaults to safe values` — defaults cascade=[], orphan=false; helpers cascadesPersist/Remove = false.
   - `Association metadata cascade all flags both persist and remove` — cascade=Cascade::ALL → ambos true.
   - `Association metadata single cascade value matches only one operation` — cascade=[Cascade::PERSIST] → persist true / remove false.
   - `Registry discovers seven method level attributes into correct event buckets` — fixture Sample con 7 methods cada attr → cada event bucket = 1 closure.
   - `Registry dispatches method level attribute closure against entity instance` — invoke sobre Sample con propiedad → propiedad mutada.
   - `Registry rejects class listener declaration that does not exist` → RuntimeException class NoExist.
   - `Registry rejects class listener that does not implement interface` → RuntimeException class WrongInterface.
   - `Registry registers class listener for all seven events` → 7 events count=1.
   - `Registry dispatches class listener callbacks correctly` + context array propagado al listener.
   - `Plain entity without new attributes exposes zero callbacks and default association shape` — BareSampleEntity = 0 callbacks 7 events, asociaciones default cascade/orphan seguros.
2. Feature `test_lifecycle_callbacks_cascade_and_orphan_removal_over_sqlite` (93 assertions ≥60 objetivo min):
   - Fixtures #[Table] nombres coinciden con tablas creadas via schema builder (no mismatch "no such table").
   - Metadata sanity: comments assoc isOneToMany/isInverseSide/cascadesPersist/cascadesRemove/orphanRemoval all true; post assoc isManyToOne/isOwningSide/cascadesPersist true/remove false/orphan false.
   - Listener wiring: prePersist=2 closures (1 method+1 class); postPersist/preUpdate/postUpdate/preRemove/postRemove=1 closure (solo class); postLoad=1 closure; hasCallbacks/!hasCallbacks unknown event.
   - Cascade persist: persist(post) sin persist(comments) → 3 comments tienen id asignado; postId FK sync en cada comment = post->id; back-reference $comment->post === post; createdAt method-level en todos 4 objetos; raw count 1 post 3 comments; raw postRow title/published/created_at verificados.
   - Orphan removal: post.comments = [c1,c3] tras flush; raw comments count=2; comment#2 fila null via where(body='#2'); comment1/3 still in DB + FK aun apuntan al post.
   - Lifecycle timestamped: tPost id/createdAt/updatedAt/updatedBy no-nulos; class listener prePersist/postPersist calls[] tiene keys + entity identico + EM mismo objeto que usamos.
   - PreUpdate context: beforeUpdateAt antes del flush/flush actualiza title/updatedAt>=beforeUpdateAt/class listener preUpdate+postUpdate llamados/preUpdate context changes.title='Callbacks v2'/currentValues.title='Callbacks v2'/originalSnapshot.title='Callbacks!'.
   - Cascade REMOVE: remove(post) sin remove(comments) → 0 filas posts / 0 comments; em->contains(post/c1/c3) = false (detach post remove).
   - PostLoad: em.clear() → find(tPost id) != tPost PHP object (fresh hydration NOT identity map cache); mismo id/title/createdAt/updatedAt/updatedBy retain DB values; class listener postLoad key exists en static::$calls + postLoad entity === tPostReload object recién hidratado.
   - Backward compat: OrmTag 0 callbacks 7 events sum; OrmUser 0 callbacks; OrmUser asociaciones foreach cascadesPersist/Remove/orphanRemoval todos false por default sin attrs.
3. Regresión GREEN: `DatabaseOrmFeatureTest` entero 7 tests/265 assertions + DVDB014LifecycleCascadeOrphan 14 tests/119 assertions = 21 tests/384 assertions exit_code 0.

### Criterio de salida

Consumidores del ORM ya pueden declarar 7 hooks lifecycle (method-level) o class listeners sin Container (V1 new $class()), cascade PERSIST/REMOVE funciona bidireccionalmente, orphanRemoval inverse OneToMany retira items DB sin manual delete, NEW entities se insertan siempre en orden parent→child sin FK violation (sin orden manual), ManyToOne owning-side FK auto-populate sin setter manual, backward compat 100% entidades preexistentes sin attrs funcionan igual (0 hooks ejecutados), sin librerías externas Composer, sin ampliar constructor EntityManager, sin bindings nuevos en DatabaseServiceProvider, sin estado mutable singleton metadata (snapshots en UoW scoped); todo test suite unit+feature+regresión GREEN exit_code 0.

### Resultado del corte DV-DB-014

1. **Lifecycle hooks system V1 operativo 7 eventos canónicos**: PrePersist/PostPersist/PreUpdate/PostUpdate/PreRemove/PostRemove/PostLoad dispatch reflection via EntityMetadata.callbacksFor() tanto method-level (ReflectionMethod + setAccessible + Closure wrapper instancia actual) como class listener via AbstractEntityLifecycleListener conveniencia.
2. **Cascade PERSIST/REMOVE BFS anti-circular**: visited spl_object_id evita loops infinitos en asociaciones bidireccionales; MERGE/DETACH/REFRESH aceptados como no-op seguros sin romper código.
3. **OrphanRemoval inverse OneToMany Managed-only hard constraint**: snapshots collection `originalCollections` en UnitOfWork::registerManaged/synchronize → collectionDiff spl_object_ids → removed items Managed solamente pasan a remove(); NEW entities en colección no activan orphan en el mismo flush (esperan siguiente flush tras synchronize).
4. **NEW inserts topológico stall-guard**: elimina bug persistente "FK NOT NULL constraint failed insert child antes parent" sin Kahn algorithm ni dependencias; RuntimeException circular reference si hay ciclo.
5. **ManyToOne FK auto-populate sync automático entre PHP reference ↔ FK scalar**: antes insert/update populateManyToOneForeignKeys lee target PHP reference y copia identifier a sourceFieldId usando misma convención naming que associationSourceValue (sin segunda convención paralela).
6. **Context typed PreUpdate/PostUpdate**: listeners reciben exactamente changes/currentValues/originalSnapshot arrays sin leak entity global.
7. **Backward Compat 100% confirmada**: OrmTag/OrmUser/Otros fixtures sin atributos nuevos → 0 callbacks 7 eventos; asociaciones sin cascade → cascade [] / orphanRemoval false. Constructor EntityManager 5 args originales intacto; DatabaseServiceProvider bindings sin cambios; EntityMetadata positional call legacy funciona.
8. **Zero external dependencies**: 0 librerías Composer añadidas. Todo vanilla PHP 8.4 reflection + Closure + spl_object_id.
9. **Test coverage GREEN**: Unit 14/14 tests/119 assertions; Feature 93 assertions subprueba 7; Regresión 21 tests/384 assertions exit_code 0.
10. **Documentación DEVELOPMENT actualizada**: DEVELOPMENT_VERSIONS.md corte 2026-09-29 + entrada DV-DB-014 detallada + siguiente bloque Relationships V1; DEVELOPMENT_MATRIX.md filas 10 (ORM)/11 (UnitOfWork Persistence)/12 (Hydration)/13 (Relationships)/20 (Events) Pendiente → Parcial con evidencia actualizada; DEVELOPMENT_GUIDELINES.md nueva sección #### Persistence Policies 10 hard rules inquebrantables DV-DB-014+; EXECUTIVE_PLAN Roadmap resumido Fase 13 añadida.

### Avance por bloques tras la fase

- Bloque 10 (ORM): Parcial mantiene; evidencia visible ampliada drásticamente (7 lifecycle attrs, 3 contracts listeners/Cascade, Entity/ManyToOne/OneToMany upgrade cascade/orphan/listeners, EntityManager flush pipeline entero reordenado con dispatch/cascade/orphan/order/FK sync, UnitOfWork snapshots colecciones, 2 suites pruebas verde 384 assertions); gaps restantes: **OneToOne/ManyToMany join-tables, proxies lazy transparente, eager joins declarativos, embedded anidados, manifests repo discovery, Criteria API typed, listeners V2 con Container DI**.
- Bloque 11 (Identity Map / Unit of Work / Persistence Policies): Parcial mantiene pero ahora policy layer V1 COMPLETO (hooks lifecycle + cascades PERSIST/REMOVE + orphan removal + orden inserts topológico + FK auto-sync); gaps restantes: cascades MERGE/DETACH/REFRESH wireado, merge/detach deep, change-tracking NOTIFY (actualmente snapshot), publisher events global fuera entity listeners, optimistic lock via version field.
- Bloque 12 (Hydration): Parcial mantiene; añade PostLoad dispatch hook tras hydrateManaged en hidratación real (no identity cache hit); gaps: hydrators compilados, partial loads, eager joins SQL directo, embedded anidados, proxies lazy inverse, postLoad context partial-fields.
- Bloque 13 (Relationships): Parcial mantiene; ahora asociaciones ManyToOne/OneToMany tienen cascade/orphanRemoval attrs + wiring metadata + inserts parent antes child + FK sync automático; gaps: **OneToOne bidireccional, ManyToMany #[JoinTable], proxies lazy, EAGER/LAZY fetch strategies, eager joins declarativos en query builder, control sistemico N+1**.
- Bloque 20 (Events): Pendiente → **Parcial** (ahora existe event system V1 DENTRO del ORM UnitOfWork via 7 lifecycle hooks + listeners + context typed + Cascade events implícitos; faltan events framework-wide bus fuera ORM: query.executed, transaction.committed, migration.applied, publishers async, listeners V2 Container DI, event execution profiler timeline, failure isolation policy configurable).
- Bloque 29 (Testing): Operativo mantiene; evidencia actualizada a **21 tests / 384 assertions** post-DV-DB-014 (DatabaseOrmFeature 265 + DVDB014LifecycleCascadeOrphan 119). Nueva subprueba 93 assertions Persistence Policies documentada.
- Bloque 31 (Developer Experience y API Pública): Operativo mantiene; nuevas APIs públicas: 7 lifecycle method attributes, Entity(lifecycleListeners) declarativo, ManyToOne(cascade) / OneToMany(cascade + orphanRemoval) declarativos, 3 contracts (Cascade / Interface / Abstract Listener), EntityMetadata hasCallbacks/callbacksFor públicos, UnitOfWork snapshotOneToManyCollections/collectionDiff públicos accesibles desde repositorios avanzados o custom listeners post-flush, Cascade::ALL/PERSIST/REMOVE constantes públicas convenience.
- Bloque 32 (VoltStack Integrations): Operativo mantiene; hard constraints NO tocar = se cumplieron: constructor EntityManager 5 args intactos, DatabaseServiceProvider bindings sin cambios, no-ampliar wiring provider, metadata singleton NO estado mutable runtime (snapshots en UoW scoped), listeners NO Container V1 (new $class() sin args).
- **Siguiente bloque recomendado post-cierre Fase 13 / DV-DB-014**: 1) **Relationships Ampliados V1** (OneToOne bidireccional owning/inverse, ManyToMany con #[JoinTable] declarativo join-table, opcional EAGER JOIN declarativo sobre SelectQueryBuilder para resolver N+1 en ManyToOne/OneToOne); 2) **Factories & Seeders V2 Ampliados** (generators CLI make:factory, make:seeder, factory states/sequences nativos, seeder dependencies DAG ordenado, SeederRunner progress/logger); 3) **Repositories Avanzados V1** (manifests extensibles para discovery repositorios módulos separados, interface bindings por entidad + container automáticos, Criteria API typed independiente SQL, codegen make:repository); 4) **Cache Queries + Listeners V2 DI** (second-level cache driver-agnostic find/where frecuentes, refresh/merge/detach deep, lifecycle listeners V2 via Container reemplazando new $class() V1 para habilitar DI en listeners).

## Fase 14 - ORM: Relationships Ampliados V1 (OneToOne + ManyToMany + JoinTable)

### Documentos fuente principales

- Bloques 10 (ORM), 11 (IdentityMap / UnitOfWork / Persistence), 12 (Hydration), 13 (Relationships), 29 (Testing), 31 (Developer Experience).
- DEVELOPMENT_MATRIX.md filas 10 / 11 / 12 / 13 / 29 / 31.
- DEVELOPMENT_GUIDELINES.md sección `#### Relationships Ampliados (OneToOne / ManyToMany / JoinTable) — Hard rules desde DV-DB-015`.

### Objetivo

Cerrar el primer corte de `Relationships Ampliados V1` sobre el ORM existente, sin abrir un segundo runtime SQL ni introducir proxies/lazy transparente: soporte bidireccional `OneToOne`, soporte `ManyToMany` con `#[JoinTable]` declarativo, extensión de metadata two-phase con fallback seguro para `mappedBy`, y write/read paths suficientes en `EntityManager` para persistir, cargar y limpiar memberships sobre SQLite manteniendo backward compatibility total.

### Entregables mínimos

1. **Nuevos atributos públicos de mapping**:
   - `Attributes/OneToOne.php`
   - `Attributes/ManyToMany.php`
   - `Attributes/JoinTable.php`
2. **Metadata de asociaciones ampliada**:
   - `EntityAssociationMetadata` añade kinds `one_to_one` y `many_to_many`.
   - Nuevos fields `joinTable`, `joinTableSourceColumn`, `joinTableTargetColumn`.
   - Helpers `isOneToOne()`, `isManyToMany()`, `isToOne()`, `isToMany()`, `usesJoinTable()`, con ownership consistente para owning/inverse.
3. **Registry two-phase robusto**:
   - `EntityMetadataRegistry::build()` descubre `#[OneToOne]`, `#[ManyToMany]` y `#[JoinTable]`.
   - `mapAssociation()` soporta owning/inverse `OneToOne` y owning/inverse `ManyToMany`.
   - Si `mappedBy` inverse aún no existe en metadata final durante shell build, se permite fallback vía reflection sobre la propiedad target para reconstruir metadata mínima consistente.
4. **Runtime ORM ampliado sin romper contratos actuales**:
   - `UnitOfWork` generaliza snapshots a asociaciones `to-many` (no sólo OneToMany).
   - `EntityManager::loadToOne()` soporta owning to-one e inverse `OneToOne`.
   - `EntityManager::loadToMany()` soporta `OneToMany` y `ManyToMany`.
   - `flush()` detecta y persiste cambios de memberships `ManyToMany`.
   - delete path limpia filas de join-table relacionadas con entidades eliminadas.
5. **Semántica de consulta explícita**:
   - `EntityQuery::where()/orderBy()` sólo traducen asociaciones owning `to-one`.
   - asociaciones `to-many` o inverse-side deben fallar con RuntimeException claro en V1.
6. **Pruebas GREEN obligatorias**:
   - suite unit dedicada para metadata `OneToOne` / `ManyToMany`.
   - feature SQLite end-to-end validando metadata, FK sync, inverse load, join rows insert/remove, cascade persist de target nuevo y delete cleanup.
   - regresión completa del vertical ORM en verde.

### Pruebas mínimas

1. Unit `DVDB015ExtendedRelationshipsTest` 5 tests / 30 assertions:
   - metadata soporta kinds `one_to_one` y `many_to_many`,
   - registry mapea owning `OneToOne` con join-column y referenced-column,
   - registry mapea inverse `OneToOne` vía `mappedBy`,
   - registry mapea owning `ManyToMany` con join-table declarativa,
   - registry mapea inverse `ManyToMany` incluso durante build shell/final con reflection fallback.
2. Feature `DatabaseOrmFeatureTest::test_one_to_one_and_many_to_many_relationships_over_sqlite` 46 assertions:
   - metadata sanity OneToOne y ManyToMany,
   - FK auto-sync owning `OneToOne`,
   - query por asociación owning `to-one`,
   - carga inverse `OneToOne`,
   - insert inicial de join rows ManyToMany,
   - mutación remove/add memberships,
   - cascade persist de target nuevo dentro de colección owning,
   - cleanup de join rows al borrar entidad participante.
3. Regresión ORM completa GREEN: 27 tests / 460 assertions.

### Criterio de salida

El ORM ya debe permitir modelar `OneToOne` y `ManyToMany` de forma declarativa y usable sobre SQLite, conservando `IdentityMap`, `UnitOfWork`, `TransactionManagerInterface` y metadata singleton-safe como únicas piezas estructurales del runtime. No se introducen proxies, eager joins declarativos ni cambios incompatibles en providers o constructores públicos.

### Resultado del corte DV-DB-015

1. **Public mapping surface completada para el corte V1**: `#[OneToOne]`, `#[ManyToMany]` y `#[JoinTable]` quedan disponibles para entidades consumidoras.
2. **Metadata ORM enriquecida sin estado mutable**: `EntityAssociationMetadata` describe join-tables y kinds nuevos; el estado transitorio de relaciones sigue viviendo fuera de metadata.
3. **Build two-phase reforzado**: inverse `mappedBy` en `OneToOne` y `ManyToMany` queda cubierto incluso durante shell build gracias al fallback por reflection.
4. **Carga relacional ampliada**: `loadToOne()` cubre inverse `OneToOne`; `loadToMany()` cubre `ManyToMany` mediante join table.
5. **Persistencia ManyToMany usable en V1**: memberships se reconcilian contra filas reales de la base de datos y se limpian al borrar entidades.
6. **Semántica de consulta endurecida**: query/order por asociaciones queda limitada a owning `to-one`, evitando falsas promesas sobre relaciones `to-many`.
7. **Cobertura verde**: Unit 5/30 + Feature 46 assertions + regresión ORM 27 tests / 460 assertions.
8. **Documentación DEVELOPMENT actualizada**: DEVELOPMENT_VERSIONS registra DV-DB-015, DEVELOPMENT_MATRIX actualiza evidencia/gaps de filas 10/11/12/13/29/31, y DEVELOPMENT_GUIDELINES añade reglas duras para OneToOne/ManyToMany/JoinTable.

## Fase 15 - ORM: Explicit Batch Preloading V1 (`EntityQuery::with(...)`)

### Documentos fuente principales

- Bloques 04 (Query Builder), 10 (ORM), 11 (IdentityMap / UnitOfWork / Persistence), 12 (Hydration), 13 (Relationships), 29 (Testing), 31 (Developer Experience).
- DEVELOPMENT_MATRIX.md filas 04 / 10 / 11 / 12 / 13 / 29 / 31.
- DEVELOPMENT_GUIDELINES.md sección `#### Explicit Batch Preloading (EntityQuery::with) — Hard rules desde DV-DB-016`.

### Objetivo

Abrir un corte incremental de `fetch strategies` sin saltar todavía a joins declarativos: habilitar `EntityQuery::with(...)` para precargar asociaciones soportadas por lotes después de la query root, usando predicates `IN` mínimos en el query layer y preservando la separación entre ejecución SQL, hidratación, IdentityMap y ensamblaje de relaciones.

### Entregables mínimos

1. **Primitive de batching en Query Layer**:
   - `SelectQueryBuilder::whereIn(string $column, array $values): self`
   - `SqlCompiler` expande `IN` / `NOT IN` a placeholders seguros y maneja listas vacías sin SQL inválido.
2. **API pública ORM de precarga explícita**:
   - `EntityQuery::with(string ...$associations): self`
   - `get()` y `first()` deben disparar la resolución batch después de hidratar roots.
3. **Batch preload en runtime ORM**:
   - `EntityManager::preloadAssociations(array $entities, array $associationNames): void`
   - soporte mínimo para owning `to-one`, inverse `OneToOne`, `OneToMany` y `ManyToMany`
   - reuso obligatorio de `IdentityMap`
   - refresh de snapshots en colecciones `to-many` precargadas para que luego puedan mutarse y flushearse correctamente
4. **No alcance explícito de esta fase**:
   - no joins SQL declarativos
   - no proxies/lazy transparente
   - no partial hydration
   - no fetch plans compilados
5. **Pruebas GREEN obligatorias**:
   - feature SQLite cubriendo `with(...)` sobre to-one y to-many
   - caso donde una colección `ManyToMany` precargada se muta y `flush()` sincroniza la join table
   - regresión ORM completa en verde

### Pruebas mínimas

1. Feature `DatabaseOrmFeatureTest::test_entity_query_with_preloads_batch_loads_supported_associations` 24 assertions:
   - `with('comments')` sobre `OrmBlogPost`
   - `with('post')` sobre `OrmBlogComment`
   - `with('profile')` sobre `OrmAccountUser`
   - `with('students')` sobre `OrmCourse`
   - `with('courses')` sobre `OrmStudent`
   - mutación post-preload de colección `ManyToMany` + `flush()` actualizando join rows
2. Regresión ORM completa GREEN:
   - `DVDB014LifecycleCascadeOrphanTest`
   - `DVDB015ExtendedRelationshipsTest`
   - `DatabaseOrmFeatureTest`
   - total: 28 tests / 484 assertions

### Criterio de salida

El consumer del ORM ya debe poder declarar explícitamente qué asociaciones quiere precargar desde `EntityQuery`, obtener objetos ya ensamblados en memoria sin N+1 inmediato sobre esos casos, y seguir mutando/flushando colecciones precargadas sin inconsistencias. La solución sigue siendo scoped, explícita y honesta: no vende joins declarativos ni hydration plans que todavía no existen.

### Resultado del corte DV-DB-016

1. **`EntityQuery::with(...)` operativo** para asociaciones root soportadas.
2. **`IN` minimalista pero útil** en Query Builder / Compiler para batching seguro.
3. **Precarga batch sin romper IdentityMap**: targets compartidos se reutilizan como la misma instancia managed.
4. **Colecciones precargadas siguen siendo flusheables** gracias al refresh de snapshots scoped tras la precarga.
5. **Scope del corte claramente acotado**: todavía no hay joins declarativos, partial hydration ni proxies lazy.
6. **Cobertura verde**: nueva feature 24 assertions + regresión ORM 28 tests / 484 assertions.
7. **Documentación DEVELOPMENT actualizada**: DEVELOPMENT_VERSIONS registra DV-DB-016, DEVELOPMENT_MATRIX actualiza filas 04/10/11/12/13/29/31, y DEVELOPMENT_GUIDELINES añade reglas duras para explicit batch preloading.

## Fase 16 - ORM: Projection / Scalar Hydration V1 (`EntityQuery::select(...)`)

### Documentos fuente principales

- Bloques 10 (ORM), 12 (Hydration), 29 (Testing), 31 (Developer Experience).
- DEVELOPMENT_MATRIX.md filas 10 / 12 / 29 / 31.
- DEVELOPMENT_GUIDELINES.md sección `#### Projection / Scalar Hydration ORM (EntityQuery::select) — Hard rules desde DV-DB-017`.

### Objetivo

Abrir un corte explícito de projection/scalar hydration sobre el ORM, sin vender partial entity hydration todavía: permitir selecciones parciales tipadas desde `EntityQuery`, resolverlas usando metadata ORM y devolver arrays/escalares ergonómicos para lectura, manteniendo separados el modo de entity hydration y el modo de proyección.

### Entregables mínimos

1. **API pública de proyección en ORM**:
   - `EntityQuery::select(string ...$fields): self`
   - `EntityQuery::rows(): array`
   - `EntityQuery::firstRow(): ?array`
   - `EntityQuery::pluck(string $field): array`
   - `EntityQuery::value(string $field): mixed`
2. **Resolución de campos soportados**:
   - scalar fields de entidad
   - embedded paths (`embedded.inner`)
   - asociaciones owning `to-one` proyectadas como identifier escalar del target
3. **Conversión tipada obligatoria**:
   - fields/embedded values deben pasar por su pipeline `castValue()`
   - identifiers de asociaciones deben canonicalizarse contra la metadata target
4. **Guardrails explícitos**:
   - `get()` / `first()` deben fallar en projection mode
   - `with(...)` y `select(...)` no se combinan en V1
   - asociaciones inverse/to-many deben rechazarse con error claro en projection mode
5. **No alcance de esta fase**:
   - no partial entity hydration
   - no snapshots parciales
   - no joins declarativos
   - no materialización de targets desde projection mode
6. **Pruebas GREEN obligatorias**:
   - feature SQLite cubriendo scalar fields tipados, embedded paths, owning association identifiers, `pluck()`, `value()` y guardrail contra `get()` parcial
   - regresión ORM completa en verde

### Pruebas mínimas

1. Feature `DatabaseOrmFeatureTest::test_entity_query_select_rows_firstrow_pluck_and_value_support_projection_mode` 18 assertions:
   - proyección tipada de `enum`, `DateTimeImmutable`, `json`
   - proyección de embedded paths
   - proyección de owning association identifier
   - helpers `pluck()` y `value()`
   - guardrail contra `get()` en projection mode
2. Regresión ORM completa GREEN:
   - `DVDB014LifecycleCascadeOrphanTest`
   - `DVDB015ExtendedRelationshipsTest`
   - `DatabaseOrmFeatureTest`
   - total: 29 tests / 502 assertions

### Criterio de salida

El consumer del ORM ya debe poder pedir proyecciones parciales tipadas sin bajar al query builder raw, pero el sistema debe seguir siendo totalmente honesto: proyección sí, partial entity hydration no. Las fronteras de `EntityManager`, `IdentityMap` y metadata singleton-safe no se alteran por este corte.

### Resultado del corte DV-DB-017

1. **Projection mode explícito operativo** sobre `EntityQuery`.
2. **Scalar hydration ORM-aware** usando metadata y type handlers existentes.
3. **Embedded paths y owning to-one identifiers proyectables** sin joins ni entidades parciales.
4. **Guardrails claros** para impedir `get()/first()` y mezcla con `with(...)`.
5. **Cobertura verde**: nueva feature 18 assertions + regresión ORM 29 tests / 502 assertions.
6. **Documentación DEVELOPMENT actualizada**: DEVELOPMENT_VERSIONS registra DV-DB-017, DEVELOPMENT_MATRIX actualiza filas 10/12/29/31, y DEVELOPMENT_GUIDELINES añade reglas duras para projection/scalar hydration.

## Fase 17 - ORM: Partial Entity Hydration V1 (`EntityQuery::partial(...)`)

### Documentos fuente principales

- Bloques 10 (ORM), 12 (Hydration), 29 (Testing), 31 (Developer Experience).
- DEVELOPMENT_MATRIX.md filas 10 / 12 / 29 / 31.
- DEVELOPMENT_GUIDELINES.md sección `#### Partial Entity Hydration V1 (EntityQuery::partial) — Hard rules desde DV-DB-018`.

### Objetivo

Abrir el primer corte real de partial entity hydration sin romper el modelo de consistencia del ORM: permitir que `EntityQuery` devuelva entidades parcialmente hidratadas, pero naciendo explícitamente detached, con upgrade posterior vía `refresh()` y sin soportar todavía joins, asociaciones parciales ni dirty tracking de campos no cargados.

### Entregables mínimos

1. **API pública de partial mode en ORM**:
   - `EntityQuery::partial(string ...$fields): self`
   - `EntityQuery::getPartial(): array`
   - `EntityQuery::firstPartial(): ?object`
2. **Runtime seguro para partial entities**:
   - `EntityManager::hydratePartial(EntityMetadata $metadata, array $row): object`
   - registro interno para reconocer entidades parciales
   - `persist()` / `remove()` deben rechazar parciales
   - `refresh()` debe promocionar parcial → managed completo
3. **Reglas mínimas del corte**:
   - auto-incluir identifier
   - soportar scalar fields y embedded paths
   - no soportar asociaciones
   - no combinar con `with(...)` ni con `select(...)`
   - `get()` / `first()` fallan en partial mode
4. **No alcance de esta fase**:
   - no joins declarativos
   - no asociaciones parciales
   - no partial entities managed desde el primer instante
   - no dirty tracking de campos no cargados
5. **Pruebas GREEN obligatorias**:
   - feature SQLite cubriendo detached partial entity, embedded parcial, persist guard y upgrade vía `refresh()`
   - regresión ORM completa en verde

### Pruebas mínimas

1. Feature `DatabaseOrmFeatureTest::test_entity_query_partial_hydration_returns_detached_entities_until_refresh` 23 assertions:
   - partial entity detached
   - properties no seleccionadas sin inicializar cuando aplica
   - embedded parcial
   - rechazo de `persist()` directo
   - upgrade exitoso vía `refresh()`
2. Regresión ORM completa GREEN:
   - `DVDB014LifecycleCascadeOrphanTest`
   - `DVDB015ExtendedRelationshipsTest`
   - `DatabaseOrmFeatureTest`
   - total: 30 tests / 525 assertions

### Criterio de salida

El consumer del ORM ya debe poder pedir entidades parciales reales desde `EntityQuery`, pero el runtime debe seguir siendo honesto: esas entidades no son managed, no son persistibles directamente y sólo entran al ciclo completo del ORM mediante `refresh()`. La solución debe preservar `IdentityMap`, `UnitOfWork` y los snapshots actuales sin introducir estados ambiguos.

### Resultado del corte DV-DB-018

1. **Partial mode explícito operativo** sobre `EntityQuery`.
2. **Entidades parciales detached y refreshables** sin contaminar `IdentityMap`.
3. **Persist/remove guardados** para impedir mutaciones peligrosas sobre parciales.
4. **Embedded parciales soportados** en el mismo modelo.
5. **Cobertura verde**: nueva feature 23 assertions + regresión ORM 30 tests / 525 assertions.
6. **Documentación DEVELOPMENT actualizada**: DEVELOPMENT_VERSIONS registra DV-DB-018, DEVELOPMENT_MATRIX actualiza filas 10/12/29/31, y DEVELOPMENT_GUIDELINES añade reglas duras para partial entity hydration V1.

## Fase 18 - ORM: Managed Partial Entity Hydration V1 (`EntityQuery::partialManaged(...)`)

### Documentos fuente principales

- Bloques 10 (ORM), 11 (IdentityMap / UnitOfWork / Persistence), 12 (Hydration), 29 (Testing), 31 (Developer Experience).
- DEVELOPMENT_MATRIX.md filas 10 / 11 / 12 / 29 / 31.
- DEVELOPMENT_GUIDELINES.md sección `#### Managed Partial Entity Hydration V1 (EntityQuery::partialManaged) — Hard rules desde DV-DB-019`.

### Objetivo

Abrir el corte donde una partial entity ya puede nacer managed, pero sin perder seguridad: tracked en `IdentityMap` + `UnitOfWork`, snapshots limitados al subset loaded, update limitado a esos mismos fields y upgrade automático a entidad completa cuando el consumer hace `find()` o `refresh()`.

### Entregables mínimos

1. **API pública nueva en ORM**:
   - `EntityQuery::partialManaged(string ...$fields): self`
   - `EntityQuery::getPartialManaged(): array`
   - `EntityQuery::firstPartialManaged(): ?object`
2. **Tracking scoped en UoW**:
   - registrar fields cargados por entidad partial-managed
   - snapshot parcial por subset loaded
   - sincronización parcial post-update
3. **Write-path seguro**:
   - dirty-check sólo contra loaded fields
   - `flushUpdate()` sólo escribe columnas cargadas
   - defaults PHP no deben pisar columnas no seleccionadas
4. **Guardrails del corte**:
   - `remove()` bloqueado para managed-partial
   - asociaciones fuera de alcance
   - loops relacionales (orphanRemoval / ManyToMany diff) ignoran managed partials
5. **Upgrade a entidad completa**:
   - `find()` o `refresh()` deben poder completar la misma instancia managed-partial
6. **Pruebas GREEN obligatorias**:
   - feature SQLite cubriendo estado managed, update acotado al subset loaded, no-overwrite de columnas no cargadas, bloqueo de remove y upgrade vía `find()`
   - regresión ORM completa en verde

### Pruebas mínimas

1. Feature `DatabaseOrmFeatureTest::test_entity_query_partial_managed_hydration_tracks_only_loaded_fields_and_upgrades_on_find` 27 assertions:
   - entidad partial-managed en estado `Managed`
   - update sólo de fields cargados
   - columnas no cargadas permanecen intactas
   - bloqueo de `remove()`
   - upgrade a entidad completa vía `find()`
2. Regresión ORM completa GREEN:
   - `DVDB014LifecycleCascadeOrphanTest`
   - `DVDB015ExtendedRelationshipsTest`
   - `DatabaseOrmFeatureTest`
   - total: 31 tests / 552 assertions

### Criterio de salida

El consumer del ORM ya debe poder trabajar con entidades parciales managed para updates locales de campos conocidos, sin que el sistema infiera nada sobre relaciones o columnas no cargadas. La coherencia de `UnitOfWork`, snapshots y write path debe permanecer scoped y explícita.

### Resultado del corte DV-DB-019

1. **Managed partial mode explícito operativo** sobre `EntityQuery`.
2. **Snapshots y dirty-check parciales** almacenados en `UnitOfWork`.
3. **Updates seguros sólo sobre fields cargados**.
4. **Upgrade transparente a entidad completa** mediante `find()` / `refresh()` sobre la misma instancia.
5. **Cobertura verde**: nueva feature 27 assertions + regresión ORM 31 tests / 552 assertions.
6. **Documentación DEVELOPMENT actualizada**: DEVELOPMENT_VERSIONS registra DV-DB-019, DEVELOPMENT_MATRIX actualiza filas 10/11/12/29/31, y DEVELOPMENT_GUIDELINES añade reglas duras para managed partial hydration.

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
13. Fase 13: ORM Lifecycle Callbacks + Cascade PERSIST/REMOVE + OrphanRemoval V1 (DV-DB-014: 7 atributos lifecycle, contracts Cascade/ListenerInterface/AbstractListener, Entity(lifecycleListeners)/ManyToOne(cascade)/OneToMany(cascade+orphanRemoval), metadata shape lifecycleCallbacks 7 events + cascade/orphan helpers, Registry buildLifecycleCallbacks 2-fases, UoW snapshots inverse collections, EM flush reordenado dispatch/cascade BFS anti-circular/orphan removal Managed-only/NEW inserts stall guard order/FK auto-sync owning-side, Unit test 14/119 assertions + Feature test 93 assertions + regresión 384 assertions GREEN)
14. Fase 14: ORM Relationships Ampliados V1 (DV-DB-015: atributos OneToOne/ManyToMany/JoinTable, EntityAssociationMetadata con kinds one_to_one/many_to_many + join-table shape, Registry two-phase con reflection fallback inverse mappedBy, UoW snapshots to-many, EntityManager loadToOne/loadToMany ampliado + persistencia/cleanup ManyToMany, Unit 5/30 + Feature 46 assertions + regresión ORM 27/460 GREEN)
15. Fase 15: ORM Explicit Batch Preloading V1 (DV-DB-016: `SelectQueryBuilder::whereIn`, `SqlCompiler` soporte `IN`, `EntityQuery::with(...)`, `EntityManager::preloadAssociations()` batch para to-one/to-many, refresh snapshots colecciones precargadas, Feature 24 assertions + regresión ORM 28/484 GREEN)
16. Fase 16: ORM Projection / Scalar Hydration V1 (DV-DB-017: `EntityQuery::select(...)->rows()/firstRow()/pluck()/value()`, resolución ORM-aware de fields/embedded/owning to-one, conversion tipada de valores proyectados, guardrails contra partial entity hydration y mezcla con `with(...)`, Feature 18 assertions + regresión ORM 29/502 GREEN)
17. Fase 17: ORM Partial Entity Hydration V1 (DV-DB-018: `EntityQuery::partial(...)->getPartial()/firstPartial()`, entidades parciales detached, auto-inclusión de PK, embedded parcial, guardrails en persist/remove, upgrade vía `refresh()`, Feature 23 assertions + regresión ORM 30/525 GREEN)
18. Fase 18: ORM Managed Partial Entity Hydration V1 (DV-DB-019: `EntityQuery::partialManaged(...)->getPartialManaged()/firstPartialManaged()`, snapshots parciales en UnitOfWork, dirty-check/update limitados al subset loaded, bloqueo de remove, upgrade vía `find()`/`refresh()`, Feature 27 assertions + regresión ORM 31/552 GREEN)

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
