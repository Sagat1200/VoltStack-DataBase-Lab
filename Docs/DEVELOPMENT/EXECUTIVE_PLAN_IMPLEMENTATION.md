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

## Roadmap resumido

1. Fase 1: Bootstrap, Config y Runtime Scope
2. Fase 2: Driver, Connection, Platform y Dialect
3. Fase 3: Execution Engine minimo
4. Fase 4: Query MVP
5. Fase 5: Schema y Migrations MVP
6. Fase 6: Transaction System minimo
7. Fase 7: API publica, CLI y Telemetria minima
8. Fase 8: ORM minimo posterior a V1

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
