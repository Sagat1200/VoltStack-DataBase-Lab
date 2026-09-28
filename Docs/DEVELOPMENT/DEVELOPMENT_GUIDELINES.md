# DEVELOPMENT_GUIDELINES

## Proposito

Este documento define la guia operativa para desarrollar `Quantum/Database` usando como fuente principal la documentacion ubicada en `vendor/voltstack/database-lab/Docs`.

Su objetivo es mantener alineados:

- la arquitectura documental,
- la implementacion real en `vendor/voltstack/framework/src/Quantum/Database`,
- las pruebas,
- y la trazabilidad del avance en `Docs/DEVELOPMENT`.

## Target de implementacion

El codigo del subsistema Database se desarrollara en:

- `vendor/voltstack/framework/src/Quantum/Database`

La integracion con el framework se apoyara en componentes ya existentes como:

- `vendor/voltstack/framework/src/Platform/Application.php`
- `vendor/voltstack/framework/src/Quantum/Config`
- `vendor/voltstack/framework/src/Quantum/Container`
- `vendor/voltstack/framework/src/Runtime/Context`
- `vendor/voltstack/framework/src/Quantum/Telemetry`
- `vendor/voltstack/framework/src/Quantum/Console`

## Fuentes de verdad

El orden de autoridad para decidir que construir y como validarlo es:

1. `vendor/voltstack/database-lab/Docs`
2. `Docs/DEVELOPMENT/DEVELOPMENT_MATRIX.md`
3. `Docs/DEVELOPMENT/DEVELOPMENT_VERSIONS.md`
4. `Docs/DEVELOPMENT/EXECUTIVE_PLAN_IMPLEMENTATION.md`
5. evidencia real en:
   - `vendor/voltstack/framework/src/Quantum/Database`
   - `vendor/voltstack/framework/src/Platform`
   - `vendor/voltstack/framework/tests/Unit`
   - `vendor/voltstack/framework/tests/Feature`

## Principios de desarrollo

### 1. El nucleo va primero

No abrir ORM, facades, code generation o integraciones avanzadas antes de cerrar:

- configuracion tipada,
- bootstrap y composition root,
- lifetimes correctos,
- connection layer,
- execution boundary.

### 2. Configuracion tipada antes que acceso dinamico a config

`Quantum/Database` no debe leer directamente:

- `config(...)`
- `$_ENV`
- `getenv()`

Los componentes internos deben recibir configuracion tipada e inmutable.

### 3. Container no es Service Locator

El container construye el grafo de servicios.

`Quantum/Database` no debe resolver dependencias arbitrariamente durante la logica ordinaria del subsistema.

### 4. Lifetime del container = lifetime semantico

Nada mutable y operation-scoped puede terminar como singleton de proceso.

Esto aplica especialmente a:

- `DatabaseExecutionScope`
- `DatabaseContext`
- `TransactionContext`
- `ConnectionLease`
- `EntityManager`
- `UnitOfWork`
- `IdentityMap`

### 5. Runtime persistente primero

Toda fase debe ser segura para:

- FrankenPHP
- RoadRunner
- OpenSwoole
- workers CLI persistentes

La limpieza del scope es parte de la correctitud del sistema, no un detalle operativo secundario.

### 6. Separaciones fundamentales

Se deben preservar siempre estas fronteras:

- `Connection != Driver`
- `Driver != Dialect`
- `Dialect != Platform`
- `Query Builder != SQL Compiler`
- `Execution != Query Planning`
- `Schema != Migration`
- `Transaction != UnitOfWork`
- `ORM != Query Builder`
- `Facade != static mutable state`
- `CLI != engine paralelo`

### 7. Query Builder no genera SQL

El builder construye modelo, AST y metadata de consulta.

La generacion SQL solo pertenece a compiler/dialect.

### 8. Execution ejecuta decisiones ya tomadas

El `Execution Engine` no debe:

- reinterpretar queries,
- replanificar,
- regenerar semantica,
- ni absorber responsabilidades de `ConnectionManager` o `TransactionManager`.

### 9. Telemetry, Security y CLI consumen el core

Estas capas deben vivir sobre contratos y servicios Database existentes.

No deben crear implementaciones paralelas del engine.

### 10. ORM completo no es el punto de arranque

El ORM es una capa tardia del roadmap.

Un Database V1 sano puede existir antes de cerrar:

- Identity Map,
- Unit of Work,
- Relationship loading,
- Active Record,
- dual ORM migration runtime.

### 11. No marcar completado por presencia de codigo

Un bloque solo puede considerarse cerrado si cumple:

1. implementacion visible,
2. integracion real con el framework,
3. pruebas relevantes,
4. actualizacion de matriz, bitacora y plan.

## Regla de priorizacion

El orden recomendado para construir `Quantum/Database` es:

### Prioridad 0: bootstrap seguro del subsistema

1. configuracion tipada,
2. `DatabaseServiceProvider`,
3. `DatabaseCompositionRoot`,
4. `DatabaseExecutionScope`,
5. validacion de lifetimes.

### Prioridad 1: infraestructura de acceso

1. `Driver`
2. `Connection`
3. `ConnectionManager`
4. `Platform`
5. `Dialect`

### Prioridad 2: frontera de ejecucion

1. `ExecutionContext`
2. `RuntimeBindingSet`
3. `QueryExecutor`
4. `Statement` y `Result`
5. error handling y cleanup determinista

### Prioridad 3: Query y Schema MVP

1. Query Model / AST minimo
2. Query Builder para operaciones basicas
3. Schema Model / Builder minimo
4. Migration repository y migration runner

### Prioridad 4: transacciones

1. `TransactionManager`
2. `TransactionContext`
3. savepoints y rollback-only
4. integracion con execution y schema

### Prioridad 5: API publica minima

1. `DB` facade contextual
2. servicio publico `Database`
3. comandos CLI basicos abriendo/cerrando su propio `ExecutionScope`
4. telemetria basica en componentes que ya poseen el lifecycle real de query/transaction/migration

### Prioridad 6: ORM y capas avanzadas

1. metadata (con relaciones ManyToOne/OneToMany bidireccionales V1, TypeRegistry V1 y embedded multi-column V1 ya implementados via `#[Embedded]` + `EntityTypedFieldInterface`; upgrade post-DV-DB-013: EntityMetadataRegistry ahora soporta resolución dual-source repositoryClass via `#[Entity(repository:X)]` explicit y fallback `CustomRepositoryRegistry::repositoryFor(entity)`, aplicado en ambas fases shell/final)
2. entity manager (con loadToOne/loadToMany helpers ya implementados y flushUpdate merged-keys dirty-check ampliado para embedded → NULL; consumers ahora pueden usar `EntityRepository::save(object,flush)` / `delete(object,flush)` sobre repos tipados con guardia RuntimeException de clase)
3. hydration y persistence planning (con any-non-null strategy para embedded VOs + flatten extract para comparacion sucia ya implementados)
4. identity map y unit of work
5. factories y seeders minimo ya implementados: contracts `FactoryInterface`/`SeederInterface`, bases `AbstractFactory`/`AbstractSeeder`, `FactoryDiscovery` / `FactoryRegistry` singleton-safe, `SeederDiscovery` + scoped `SeederRunner` (transaction + flush + rollback on Throwable), CLI comando `database:seed --class --path` con scope propio, bugfix `DatabaseResult` implements `\Countable` + `\IteratorAggregate`. Evidencia: regresion 12 tests / 210 assertions green, `DatabaseFactoriesSeedersFeatureTest` 32 assertions, 4 comandos DB CLI registrados en provider.
6. repository DI tipado y helpers ergonomicos ya implementados: contract público `RepositoryFactoryInterface::repositoryFor(entityClass): EntityRepositoryInterface`, atributo declarativo `#[RepositoryFor(EntityClass::class)]` sobre custom repositorios (independiente de entidad fuente), `CustomRepositoryRegistry` singleton-safe dual-mode (`#[RepositoryFor]` discovery + `register(repoClass,?entityClass)` explícito duplicate-guarded), `EntityRepositoryFactory` scoped con delegación 100% a `EntityManager::repository()` (única cache canónica), EntityRepository upgrade con `save/delete` (RuntimeException guard) + `count/exists` + accessors `getEntityManager/getMetadata/getEntityClass`, aggregators `SelectQueryBuilder::count()` y `EntityQuery::count()` sobre DatabaseResult Countable (fix: v1 usaba "COUNT(*) AS aggregate" → SqlCompiler lo quotaba como identificador → 0 rows; corregido a count(get())), shortcut público `DatabaseInterface::repositoryFactory()` + `Database` constructor 9º param, bindings lifetime-correctos en DatabaseServiceProvider (CustomRegistry singleton, EntityRepositoryFactory scoped bind → interface). Evidencia: feature test 70 aserciones sobre SQLite (OrmTag + #[RepositoryFor] OrmProductRepository, resolución dual-source metadata, DI RepositoryFactoryInterface, save/flush triple, count=3 raw/repo/exists, visible criteria counts, EntityQuery count where, findOneBy, delete(flush:true), findBy ordered asc, custom repo label method). Regresión verde: 21 tests / 280 assertions post-013.
7. lifecycle callbacks / events / politicas de persistencia (cascade / orphan removal minimo) — **siguiente bloque recomendado #1**
8. relationships ampliados (OneToOne bidireccional, ManyToMany join-table, proxies lazy-transparente opcional, eager joins declarativos sobre SelectQueryBuilder) — **siguiente bloque recomendado #2**
9. repositories avanzados: manifests extensibles para discovery en módulos/bounded-contexts separados, interface bindings por entidad (ej `OrmProductRepositoryInterface → concrete`), Criteria API typed, codegen helpers `make:repository` (siguiente bloque recomendado #3)

### Prioridad 7: Ampliacion funcional y capas non-MVP

1. factories / seeders / fixtures completos sobre `EntityManager` y `Database` (si Prioridad 6 item 5 fuera el minimo viable, aqui se completa ergonomia, CLI commands ricos, definicion declarativa)
2. pagination, chunking y bulk operations
3. cache de compilacion / query cache
4. isolation levels / locking optimista-pesimista / retries
5. resiliencia y distribucion (read/write, failover)
6. embedded anidados (VO dentro de VO) y custom types user-land via manifests extensibles

## Flujo obligatorio para abrir una fase nueva

Cada fase nueva debe seguir este flujo:

1. identificar el documento fuente principal,
2. detectar dependencias documentales asociadas,
3. revisar evidencia ya existente en el framework,
4. aislar el bloque minimo operable,
5. implementar,
6. validar con pruebas del nivel correcto,
7. actualizar `DEVELOPMENT_MATRIX.md`,
8. registrar el corte en `DEVELOPMENT_VERSIONS.md`,
9. ajustar `EXECUTIVE_PLAN_IMPLEMENTATION.md` si cambia la prioridad real.

## Reglas por tipo de bloque

### Bootstrap, Config y Runtime

Todo trabajo sobre `06`, `07`, `251`, `252`, `311`, `312`, `313`, `321` debe respetar:

- tipado e inmutabilidad,
- lifetimes explicitos,
- inicializacion lazy,
- ausencia de estado global mutable,
- y limpieza determinista del scope.

### Driver, Connection y Execution

Todo trabajo sobre `10-12`, `76-86`, `164-175` debe respetar:

- ownership claro de recursos,
- aislamiento de connection state,
- errores explicitamente modelados,
- y pruebas con motor real cuando la garantia dependa del servidor DB.

### Query, Schema y Migration

Todo trabajo sobre `23-111` debe respetar:

- separacion AST/compiler/execution,
- capability awareness,
- portabilidad razonable,
- y no acoplar el modelo a SQL string concatenado.

### ORM y Hydration

Todo trabajo sobre `112-163` debe respetar:

- integracion sobre el mismo Query/Execution Engine, sin abrir un runtime/connection/SQL paralelo para ORM,
- metadata inmutable o singleton-safe y estado mutable ORM siempre scoped,
- ausencia de SQL dentro del ORM,
- ausencia de estado compartido entre requests,
- conversion de tipos centralizada en metadata/type handlers, no en casts ad hoc repartidos entre `EntityManager`, `Repository` o `Model`,
- `Model` y repositories resolviendo el `EntityManager` del scope activo, sin contexto mutable estatico,
- relaciones bidireccionales resueltas con build de metadata en dos fases (shell primero, asociaciones despues) para evitar recursion,
- helpers de carga (to-one, to-many) reutilizando `IdentityMap` y `DatabaseQueryManager`, sin duplicar estrategias de lectura,
- traduccion automatica de filtros por nombre de asociacion usando metadata canonica, no por introspection ad-hoc,
- value objects embedded multi-columna mapeados via atributo declarativo `#[Embedded]` y metadata canonica, jamas via SQL manual o hardcoding de columnas internas,
- pipeline de tipos unificado para campos escalares y embedded inner fields via contrato compartido (`EntityTypedFieldInterface`); no duplicar logica de casting ni handlers separados,
- hidratacion nullable de embedded usando estrategia consistente (any-non-null: si al menos una columna interna no es NULL, reconstruir el VO; si todas son NULL, asignar NULL a la propiedad PHP),
- extraccion para comparacion sucia (`extract`) usando claves aplanadas `embeddedName.innerName`; flushUpdate debe mergear keys de snapshot y extract actual para detectar cuando un embedded VO completo se vuelve NULL y escribir NULLs en todas sus columnas,
- traduccion de queries con paths embedded anidados (`price.amount`) via metadata canonica en el EntityQuery, no via parsing de strings ad-hoc fuera de la capa de metadata,
- metadata de embedded recolectada en las mismas dos fases shell→final del EntityMetadataRegistry junto a asociaciones y fields, garantizando que embedded se resuelva con seguridad incluso dentro de entidades que son target de asociaciones bidireccionales,
- factories y seeders construidos SOBRE el EntityManager scoped existente, sin abrir runtime SQL paralelo ni sistema de conexiones independiente; `Factory::create()` delega persist sin flush implicito, el runner se hace cargo del batched flush,
- factories/seeders discovery y registries son singleton-safe (inmutables tras boot); runner y seeder instancias se resuelven scoped por request/comando CLI,
- archivos de factory/seeder descubiertos via `require` con 3 formas de retorno aceptadas: objeto instanciado, class-string o `callable(Application): object` para habilitar inyeccion de app en clases anonimas de prueba,
- comando `database:seed` abre y cierra su propio scope igual que status/migrate/rollback, jamas reutiliza un scope de request externo,
- `DatabaseResult` debe implementar `\Countable` y `\IteratorAggregate` ya que PDO SQLite retorna rowCount() = 0 para SELECTs; count() despacha a `rows[]` cuando type=Rows y a affectedRows cuando type=Affected,
- Repository Factory con DI tipado: contract `RepositoryFactoryInterface::repositoryFor(entityClass)` debe ser inyectable por constructor en consumidores (no usar `Application::make()` externamente como service locator); EntityRepositoryFactory **debe ser scoped** (depende del EntityManager scoped actualmente activo en request/comando) — NUNCA un singleton de proceso para no servir un EntityManager stale entre fronteras de request,
- CustomRepositoryRegistry **debe ser singleton-safe** y contener solo metadata/registros de binding (Custom repos FQCNs), nunca estado mutable de runtime; soporta dual convention: atributo `#[RepositoryFor(Entity)]` auto-descubierto + `register(repo,?entityClass)` explícito para manifests/tests,
- EntityMetadataRegistry resuelve `repositoryClass` en orden de prioridad fijo: primero `#[Entity(repository: X)]` sobre entidad (binding cerrado, alto acoplamiento) → luego fallback `CustomRepositoryRegistry::repositoryFor(entity)` (bajo acoplamiento, repositorio en módulo separado); resolución debe aplicarse idénticamente tanto en fase shell como en fase final del build two-phase para mantener consistencia con asociaciones y embeddeds,
- helpers ergonomicos sobre EntityRepository (`save(object,flush)`, `delete(object,flush)`) **deben delegar 100% a EntityManager persist/remove + optional flush**, sin escribir SQL ni Query Builder directo; guardia RuntimeException si la clase de entidad no coincide con `$this->metadata->className` (evita contaminación accidental de repos tipados),
- agregadores `SelectQueryBuilder::count()` y `EntityQuery::count()` **NO deben inyectar expresiones SQL crudas en la lista columns** (SqlCompiler aplica `quoteIdentifierPath()` a cada columna y convierte `COUNT(*) AS aggregate` en un identificador quoted que retorna 0); alternativa cross-driver segura: `count($clonedBuilder->get())` sobre DatabaseResult Countable ya habilitado desde DV-DB-012,
- shortcuts públicos `Database::repositoryFactory()` / `DatabaseInterface::repositoryFactory()` deben retornar la instancia scoped actual del RepositoryFactoryInterface (no una nueva); provider bindings deben inyectar RepositoryFactoryInterface como 9º argumento del constructor de Database (scoped), manteniendo el patrón de graph wiring sin service locator,
- y pruebas separadas por metadata, hydration, persistence y relaciones (con bloque especifico de embedded: prefix default/explicito, nullable hydrate, nulificacion completa de VO, nested where/orderBy, updates tras clear/reload), mas un feature test end-to-end de factories + seeders cubriendo: discovery, make sin persist, times(n)+create, SeederRunner transaccional con rollback on Throwable, comando CLI `database:seed` y consulta post-seed via EM query + repository; mas un feature test end-to-end de Repository Factory + helpers ergonomicos cubriendo: dual source metadata resolution, DI typed resolution, save/flush triple insert, raw count vs repo count, exists/criteria, EntityQuery count, delete + orderBy find, custom repo via #[RepositoryFor] attribute.

### Security, Telemetry, CLI y Plugins

Todo trabajo sobre `216-340` debe respetar:

- capas finas sobre servicios existentes,
- facades y comandos resolviendo servicios scoped, nunca conservando estado mutable estatico,
- fail-soft cuando corresponda,
- diagnosticos explicables,
- y compatibilidad retroactiva observada por contrato.

## Definition of Done minima

Un bloque solo puede marcarse como `Operativo` si cumple todos estos puntos:

1. Existe implementacion identificable en `src/Quantum/Database`.
2. La integracion real esta enlazada al framework cuando aplica.
3. Hay pruebas unitarias, component o feature segun la garantia que se afirma.
4. El cambio respeta runtime persistente y scope isolation.
5. Se actualiza `DEVELOPMENT_MATRIX.md`.
6. Se agrega una entrada nueva en `DEVELOPMENT_VERSIONS.md`.
7. Se ajusta el plan ejecutivo si cambian dependencias o prioridad.

## Validacion minima obligatoria

Antes de cerrar una fase, validar como minimo lo que aplique:

- pruebas unitarias del componente tocado,
- pruebas feature para bootstrap, request scope o CLI,
- pruebas de integracion con motor real para transacciones, locking, schema o SQL dialect,
- pruebas de cleanup y lifecycle cuando se toque runtime persistente,
- y pruebas de regresion de configuracion/bindings cuando se toque bootstrap.

## Regla de evidencia

No se debe subir el estado de un bloque a `Operativo` si no existe al menos una de estas evidencias:

1. clase o modulo principal identificable,
2. integracion real en bootstrap o runtime,
3. prueba unitaria o feature especifica,
4. comportamiento observable verificable,
5. trazabilidad actualizada en `Docs/DEVELOPMENT`.

## Anti-patrones a evitar

No continuar el desarrollo con estos patrones:

1. comenzar por Active Record o facades antes de cerrar Connection + Execution,
2. resolver dependencias internas usando el container como service locator,
3. guardar contexto mutable en propiedades estaticas,
4. mezclar query builder con SQL generation,
5. mezclar schema y migration con ejecucion directa via `PDO`,
6. considerar mocks como evidencia suficiente de garantias del motor real,
7. abrir demasiadas plataformas o drivers antes de cerrar un vertical minimo utilizable,
8. marcar bloques como implementados sin actualizar la carpeta `DEVELOPMENT`.

## Regla de mantenimiento

Cada ciclo de desarrollo que cierre o cambie un bloque relevante de Database debe actualizar, en el mismo ciclo:

1. `DEVELOPMENT_MATRIX.md`
2. `DEVELOPMENT_VERSIONS.md`
3. `EXECUTIVE_PLAN_IMPLEMENTATION.md`

Si esos documentos no se actualizan, el subsistema pierde trazabilidad operativa.
