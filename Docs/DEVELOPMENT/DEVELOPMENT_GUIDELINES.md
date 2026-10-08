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
- y pruebas separadas por metadata, hydration, persistence y relaciones (con bloque especifico de embedded: prefix default/explicito, nullable hydrate, nulificacion completa de VO, nested where/orderBy, updates tras clear/reload), mas un feature test end-to-end de factories + seeders cubriendo: discovery, make sin persist, times(n)+create, SeederRunner transaccional con rollback on Throwable, comando CLI `database:seed` y consulta post-seed via EM query + repository; mas un feature test end-to-end de Repository Factory + helpers ergonomicos cubriendo: dual source metadata resolution, DI typed resolution, save/flush triple insert, raw count vs repo count, exists/criteria, EntityQuery count, delete + orderBy find, custom repo via #[RepositoryFor] attribute; mas un feature test end-to-end de Persistence Policies (Lifecycle/Cascade/OrphanRemoval) cubriendo: metadata sanity cascade/orphan/callbacks count/event hasCallbacks, method-level 7 hooks reflection dispatch, class-level listeners implement Interface validation, cascade persist padre + N hijos via solo persist padre, orphan removal unset item inverse OneToMany (Managed-only hard constraint), NEW inserts orden topológico parent→child stall-guard, ManyToOne owning-side FK auto-sync desde PHP reference ($comment->post asignado → $comment->postId poblado sin setter manual), PreUpdate context typed (changes / currentValues / originalSnapshot), Cascade REMOVE padre → hijos eliminados, postLoad find tras clear() forced hydration, backward compat entidades sin attrs = 0 callbacks todos events + asociaciones default cascade/orphan seguros.

#### Persistence Policies (Lifecycle / Cascade / OrphanRemoval) — Hard rules desde DV-DB-014

Toda implementación o extensión sobre `Lifecycle Callbacks`, `Cascade Operations` y `OrphanRemoval` en el ORM UnitOfWork/EntityManager **DEBE** respetar:

- **Orden canónico invariable de lifecycle hooks durante flush**: `PrePersist` → (populate owning-side FK values + snapshot collections) → `INSERT/UPDATE` SQL → sincronize UoW snapshots y asignación de identifier → `PostPersist/PostUpdate` (todos DENTRO de la misma frontera `TransactionManager::transaction()`, antes de commit externo); delete path: `PreRemove` → `DELETE SQL` → `PostRemove` → `detach()` UoW (nunca antes PostRemove, para que listeners lean el entity todavía managed); hydrate path: `hydrateManaged()` (solo cuando la instancia NO estaba en identityMap cache) → `registerManaged` → inmediatamente `dispatchLifecycle('postLoad')`. Si una implementación cambia este orden → es breaking change documentado y requiere nueva versión DV-DB no un fix trivial.
- **Backward compatibility 100% NO negociable**: entidades y consumer code preexistentes (sin atributos nuevos `#[Entity(lifecycleListeners)]` / `#[ManyToOne(cascade)]` / `#[OneToMany(cascade,orphanRemoval)]` / method-level lifecycle hooks) **DEBEN** comportarse idénticamente a pre-014: sin hooks disparados, sin cascades implícitas (default `cascade = []`), sin orphan removal (default `orphanRemoval = false`). Esto se garantiza con named args PHP 8 trailing params con defaults cero / false y shape EntityMetadata `KNOWN_LIFECYCLE_EVENTS` inicializado a 7 arrays vacíos incluso cuando NO se pasa lifecycleCallbacks al constructor. Las signatures de constructores de EntityManager (5 args) y DatabaseServiceProvider bindings NO se amplían ni se cambian positional args para romper consumer code.
- **Estado mutable scoped, NO metadata singleton mutable**: los snapshots de colecciones OneToMany para OrphanRemoval (`UnitOfWork::$originalCollections`), visited arrays de cascade BFS, collectionDiff results, y callbacks dispatch closure temporales **DEBEN** vivir en UnitOfWork/EntityManager scoped; metadata singleton (EntityMetadata / EntityAssociationMetadata) sólo contiene shape `lifecycleCallbacks` con closures reflection sin estado por instancia. En runtime persistente FrankenPHP/RoadRunner dos requests concurrentes NO comparten snapshots ni visited arrays.
- **Cascade operations V1 sólo implementa activamente `Cascade::PERSIST` y `Cascade::REMOVE`**: constantes `MERGE/DETACH/REFRESH` se aceptan en el atributo `#[ManyToOne(cascade: [...])]` sin error parse validation, pero el pipeline flush NO ejecuta wireado para ellas; si un equipo quiere esas operaciones debe declararlas explícitamente y el código core las ignora de forma segura (no exception, solo no-op). Cascade BFS **DEBE** usar `visited[spl_object_id]` como guard anti-circular infinite loop; entidades ya visitadas saltan.
- **OrphanRemoval hard constraints V1** (no romper under any circumstances without DV approval): (1) orphanRemoval **SOLO** aplica a asociaciones inverse-side `OneToMany` (nunca a ManyToOne owning, nunca a OneToOne V1 futuro sin explicit approval); (2) `collectOrphansForRemoval()` **SOLO** opera sobre entidades que están state = Managed en el UnitOfWork. Entidades New que acaban de ser persistidas en el mismo flush NO se ven afectadas (para considerarlas Managed primero deben pasar por synchronize() tras insert, snapshot se actualizará en el próximo flush); (3) la detección de orphans se basa en `collectionDiff()` comparando snapshot de spl_object_id integers tomado en registerManaged/synchronize con la colección PHP actual, NUNCA basado en comparación de IDs de DB (evita falsos positivos si el entity es New sin id todavía).
- **NEW entities insert order topológico stall guard**: el orden de `flushNewEntities()` NO es el spl_object_id arbitrario actual (que provocaba FK NOT NULL violation insertando comentario antes que post). **DEBE** usarse repeat-pass stall-guard: (1) iterar NEW entities, (2) `newEntityIsInsertable()` = todos sus ManyToOne target objects ya tienen identifier asignado (ya fueron insertados en passes anteriores), (3) si un pass insertó ≥ 1 entity, repetir; si un pass NO avanzó y todavía quedan NEW → throw RuntimeException "Circular reference detected while ordering NEW entities for flush". No se usa Kahn topological sort para evitar complejidad adicional sin librerías externas (0 dependencias constraint).
- **ManyToOne owning-side FK auto-populate**: antes de INSERT y antes de UPDATE **DEBE** ejecutarse `populateManyToOneForeignKeys($entity, $metadata)` que lee la PHP reference del ManyToOne target (ej: `$comment->post`), si el target tiene identifier asignado, copia su id al campo fuente FK del entity (`$comment->postId`), siguiendo **EXACTAMENTE la misma convención naming que `associationSourceValue`**: (a) prioridad `sourceField . 'Id'` default naming, (b) fallback match de `mappedFields[]` contra `sourceColumn` declarado, (c) fallback directo al `sourceField` string declarado. NO hay segunda convención naming paralela.
- **Lifecycle listeners V1 se instancian via `new $className()` SIN Container dependency injection**: al validar un `#[Entity(lifecycleListeners: [MyListener::class])]`, el registry (1) comprueba `class_exists($class)` → RuntimeException si no; (2) `is_subclass_of($class, EntityLifecycleListenerInterface::class)` → RuntimeException si no implementa; (3) instancia `new $class()` sin constructor args; (4) registra 7 closures por event que llaman a `$instance->$event($entity, $em, $context)`. En una futura V2 se podrá habilitar Container DI resolución vía `Application::make($class)` o servicio nombrado; hasta entonces NO pasar constructor args (default conveniencia via AbstractEntityLifecycleListener).
- **PreUpdate context typed y no-mutation SQL**: el contexto pasado a `dispatchLifecycle('preUpdate', $entity, [...])` **DEBE** contener keys fijas `changes: array<string, mixed>` (campos que realmente cambiaron entre snapshot y actual), `currentValues: array<string, mixed>` (extractForWrite actual completo), `originalSnapshot: array<string, mixed>` (snapshot al registerManaged). Listeners pueden mutar la entity PHP durante PrePersist/PreUpdate (ej: setear updatedAt) y el cambio debe reflejarse en el SQL/values de INSERT/UPDATE (populateManyToOneForeignKeys y changeset/extract se calculan DESPUÉS de PrePersist pero ANTES de extractForWrite actual; en preUpdate el changeset ya fue calculado, mutaciones hechas por listener en PreUpdate NO entran al SQL update del mismo flush — la mutación se reflejará en el próximo flush como dirty-check). Esta restricción evita doble-cálculo costoso de changesets cuando un listener modifica 20 campos.
- **TODOS los lifecycle callbacks y listeners (method-level reflection + class instance closures) se invocan dentro del try block de `TransactionManager::transaction()`**: un throw Throwable dentro de un listener aborta el flush completo y produce rollback automático, sin inserts parciales persistidos. No hay policy de "skip listener failure" configurable en V1; ese feature requiere V2 Event Bus framework-wide.

#### Relationships Ampliados (OneToOne / ManyToMany / JoinTable) — Hard rules desde DV-DB-015

Toda implementación o extensión sobre `OneToOne`, `ManyToMany` y `JoinTable` en el ORM **DEBE** respetar:

- **Metadata shell→final obligatoria con fallback controlado**: `EntityMetadataRegistry::build()` debe seguir resolviendo asociaciones en dos fases. Si una asociación inverse `mappedBy` todavía no aparece en metadata final durante el shell build, la resolución puede caer a reflection directa del property target para leer `#[OneToOne]`, `#[ManyToMany]` y `#[JoinTable]`, pero ese fallback es estrictamente de bootstrap; la metadata final sigue siendo la fuente de verdad observable en runtime.
- **Ownership canónico sin ambigüedad**: en `OneToOne` y `ManyToMany` debe existir exactamente un lado owning y, si hay lado inverse, éste se expresa vía `mappedBy` apuntando a una asociación real del target. El lado inverse nunca define su propia estrategia física de persistencia; siempre hereda la del owning-side.
- **`#[JoinTable]` sólo vive en el owning-side `ManyToMany`**: el lado inverse no declara nombre de tabla ni columnas propias. `joinTable`, `joinTableSourceColumn` y `joinTableTargetColumn` se derivan desde la definición owning-side y deben quedar consistentes para ambos lados en `EntityAssociationMetadata`.
- **Las queries por nombre de asociación sólo son válidas para relaciones owning `to-one`**: `EntityQuery::where()` y `orderBy()` pueden traducir asociaciones `ManyToOne` / owning `OneToOne` a columna FK. Cualquier asociación `to-many` o inverse-side debe fallar explícitamente con error claro; no se permiten semánticas implícitas ni JOINs mágicos en V1.
- **El estado mutable de colecciones sigue siendo scoped**: snapshots/diffs de asociaciones `to-many`, memberships actuales y resultados intermedios de reconciliación de join-tables deben vivir en `UnitOfWork` o `EntityManager` scoped. `EntityAssociationMetadata` y `EntityMetadata` permanecen stateless/singleton-safe para runtimes persistentes como FrankenPHP o RoadRunner.
- **Persistencia `ManyToMany` reconciliada contra la DB real**: la escritura de memberships no puede depender únicamente de snapshots en memoria, porque entidades recién insertadas o sincronizadas pueden dejar los snapshots alineados con la colección actual. El source of truth para altas/bajas de join rows es la comparación entre colección owning actual e IDs existentes realmente en la join table.
- **Delete cleanup de join-tables es obligatorio**: al remover una entidad que participa en asociaciones `ManyToMany`, el runtime debe limpiar las filas de la tabla intermedia relacionadas con su identifier antes o durante el path de borrado, evitando memberships huérfanos.
- **Carga relacional explícita y acotada en V1**: `loadToOne()` soporta owning to-one e inverse `OneToOne` por reverse lookup; `loadToMany()` soporta `OneToMany` y `ManyToMany` via join table. No se introducen proxies, lazy-transparente ni eager joins declarativos dentro de DV-DB-015; cualquier salto a fetch strategies V2 requiere un nuevo corte DV-DB.
- **Backward compatibility estricta**: entidades que sólo usan `ManyToOne`/`OneToMany`, o que no declaran asociaciones nuevas, deben seguir funcionando sin cambios. No se amplía el constructor de `EntityManager`, no se alteran los bindings públicos del provider y no se rompe la carga previa de metadata.
- **Prueba mínima obligatoria**: todo cambio en este bloque debe traer al menos (1) unit tests de metadata para owning/inverse `OneToOne` y `ManyToMany`, incluyendo fallback two-phase, y (2) feature test SQLite validando query owning to-one, inverse `OneToOne`, insert/remove ManyToMany, cascade persist de target nuevo y cleanup por delete.

#### Explicit Batch Preloading (`EntityQuery::with`) — Hard rules desde DV-DB-016

Toda implementación o extensión sobre `EntityQuery::with(...)`, predicates `IN` y batch preload ORM **DEBE** respetar:

- **Precarga explícita, nunca implícita**: el ORM no debe disparar batch preload automáticamente por detectar acceso posterior a propiedades. La activación V1 ocurre sólo cuando el consumer llama `EntityQuery::with('assoc', ...)` antes de `get()` o `first()`.
- **Separación estricta de responsabilidades**: `SelectQueryBuilder` y `SqlCompiler` sólo aportan el primitive `IN` necesario para cargar por lotes; la resolución de identidad, ensamblaje de asociaciones y asignación a propiedades pertenece exclusivamente a `EntityManager`/`IdentityMap`, no al compilador SQL.
- **Sin joins declarativos encubiertos**: DV-DB-016 NO autoriza introducir `JOIN` en `SelectQueryBuilder` ni semánticas mágicas en `where()/orderBy()` para relaciones `to-many`. `with(...)` carga primero entidades root y luego resuelve asociaciones por consultas adicionales controladas.
- **`with(...)` sólo acepta asociaciones reales del root entity**: nombres vacíos se ignoran; asociaciones inexistentes deben lanzar `RuntimeException` clara. No se soportan paths anidados (`comments.author`) ni árboles profundos en V1 sin un nuevo corte DV-DB.
- **Reuse de identidad obligatorio**: toda entidad objetivo obtenida en un preload debe pasar por `hydrateManaged()`/`IdentityMap`; si varias roots apuntan al mismo target, deben reusar exactamente la misma instancia administrada.
- **Snapshots refreshed tras precarga `to-many`**: cuando `with(...)` llena una colección `OneToMany` o `ManyToMany` de una entidad managed, `UnitOfWork` debe refrescar sus snapshots scoped inmediatamente para que un `flush()` posterior detecte adds/removes reales sobre esa colección precargada.
- **Compatibilidad total con runtime persistente**: cualquier grouping temporal, mapas `sourceId -> targets`, listas de identifiers y resultados de batch preload viven sólo dentro del scope actual. Metadata singleton y providers públicos no almacenan estado mutable de esas precargas.
- **Semántica honesta de alcance**: V1 cubre owning `to-one`, inverse `OneToOne`, `OneToMany` y `ManyToMany` por lotes. No cubre partial hydration, proxies lazy, fetch plans compilados, joins SQL ni estrategias declarativas `EAGER/LAZY`; eso pertenece a la siguiente fase.
- **Predicates `IN` deben ser seguros**: el compilador debe expandir placeholders por valor, rechazar usos inválidos (`IN` con no-array) y manejar listas vacías sin generar SQL inválido.
- **Prueba mínima obligatoria**: cada ampliación de este bloque debe traer feature tests sobre SQLite cubriendo `with(...)` para asociaciones to-one y to-many, reuse de identidad donde corresponda, y al menos un caso donde una colección `ManyToMany` precargada se muta y luego `flush()` actualiza la join table correctamente.

#### Projection / Scalar Hydration ORM (`EntityQuery::select`) — Hard rules desde DV-DB-017

Toda implementación o extensión sobre `EntityQuery::select(...)`, `rows()`, `firstRow()`, `pluck()` y `value()` **DEBE** respetar:

- **Projection mode explícito, nunca parcial implícito**: cuando una query entra en modo `select(...)`, deja de ser una query de entity hydration. `get()` y `first()` deben fallar explícitamente; el consumer debe usar `rows()`, `firstRow()`, `pluck()` o `value()`.
- **No mezclar projection con preload entity-mode**: `with(...)` y `select(...)` no se combinan en la misma query V1. Una query o hidrata entidades/preloads, o devuelve resultados escalares/proyectados; no ambas cosas a la vez.
- **Resolución ORM-aware obligatoria**: `select(...)` debe aceptar al menos:
  - fields escalares de entidad,
  - embedded paths (`price.amount`),
  - y asociaciones owning `to-one` como identifier escalar del target.
  Debe rechazar asociaciones inverse-side y `to-many` con error claro.
- **Conversión tipada de vuelta a PHP**: valores proyectados deben pasar por el mismo pipeline de cast ya disponible para fields y embedded-inner fields (`enum`, `datetime_immutable`, `json`, etc.). Proyectar no significa degradar silenciosamente todo a strings crudos.
- **Sin prometer partial entities**: este corte no autoriza crear instancias entidad parcialmente hidratadas, ni snapshots parciales, ni refresh parciales. Eso pertenece a una fase posterior de partial hydration real.
- **Sin joins mágicos**: proyectar una asociación owning `to-one` devuelve el identifier FK ya disponible en la tabla root; no autoriza introducir joins SQL automáticos ni materializar entidades target.
- **Compatibilidad total con Query Builder actual**: `select(...)` ORM debe apoyarse sobre el `SelectQueryBuilder` existente sin romper `where()`, `orderBy()`, `limit()`, `offset()` ni `count()`.
- **Guardrails antes que magia**: si el usuario selecciona campos vacíos, asociaciones no soportadas o intenta hidratar entidades completas desde una selección parcial, la API debe fallar con mensajes directos y explicables.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir feature tests con SQLite para:
  - scalar fields tipados,
  - embedded paths,
  - owning association identifier projection,
  - helpers `pluck()` y `value()`,
  - y el guardrail que bloquea `get()/first()` en projection mode.

#### Partial Entity Hydration V1 (`EntityQuery::partial`) — Hard rules desde DV-DB-018

Toda implementación o extensión sobre `EntityQuery::partial(...)`, `getPartial()` y `firstPartial()` **DEBE** respetar:

- **Partial entities nacen detached**: una entidad parcialmente hidratada NO entra a `IdentityMap` ni a `UnitOfWork` en este corte. No debe aparentar estar managed cuando no lo está.
- **Upgrade explícito vía `refresh()`**: si el consumer quiere convertir una partial entity en entidad completa y tracked, debe llamar `EntityManager::refresh($entity)`. Ese paso debe rehidratar desde DB, registrar la entidad y retirar cualquier marca interna de partial state.
- **Sin persist/remove directo**: `persist()` y `remove()` deben rechazar partial entities con error claro. No se permiten inserts, updates ni deletes desde una entidad sólo parcialmente cargada.
- **Auto-incluir identifier siempre**: aunque el consumer no seleccione la PK, `partial(...)` debe incluirla internamente para que la entidad sea refreshable y tenga identidad consistente.
- **Alcance V1 deliberadamente acotado**: partial hydration soporta scalar fields y embedded paths. No soporta asociaciones, joins, preload combinado, snapshots parciales managed ni dirty tracking de campos no cargados.
- **Sin mezclar modos**: `partial(...)` no se combina con `with(...)` ni con `select(...)` en la misma query. Cada modo (`entity`, `projection`, `partial`) debe permanecer explícito y mutuamente excluyente.
- **Propiedades no cargadas pueden permanecer sin inicializar**: eso es correcto en V1. La implementación no debe forzar valores dummy para aparentar completitud, salvo defaults ya propios de la clase PHP.
- **Embedded parciales son válidos**: si sólo algunas columnas de un embeddable fueron seleccionadas, el value object puede materializarse parcialmente con sólo esos inner fields inicializados.
- **Guardrails antes que magia**: `get()` / `first()` deben fallar en partial mode; el consumer debe usar `getPartial()` / `firstPartial()`.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe traer feature tests sobre SQLite cubriendo:
  - entidad parcial detached,
  - properties no seleccionadas sin inicializar cuando aplique,
  - embedded parcial,
  - rechazo de `persist()` o `remove()` directo,
  - y upgrade exitoso a entidad managed completa vía `refresh()`.

#### Managed Partial Entity Hydration V1 (`EntityQuery::partialManaged`) — Hard rules desde DV-DB-019

Toda implementación o extensión sobre `EntityQuery::partialManaged(...)`, `getPartialManaged()` y `firstPartialManaged()` **DEBE** respetar:

- **Managed sí, pero acotado al subset loaded**: una entidad managed-partial entra a `IdentityMap` y `UnitOfWork`, pero su snapshot y dirty-check cubren únicamente los fields explícitamente cargados.
- **Auto-incluir identifier siempre**: la PK debe formar parte del subset tracked, aunque el consumer no la seleccione.
- **Write path limitado a loaded fields**: `dirtyManagedEntities()` y `flushUpdate()` no pueden inferir cambios ni escribir columnas que no fueron cargadas por la query partial-managed.
- **No degradar otras columnas por defaults PHP**: propiedades con default de clase (por ejemplo arrays vacíos) no deben generar updates sobre columnas no seleccionadas.
- **Upgrade automático permitido**: `find()` o `refresh()` pueden completar una managed-partial entity en la misma instancia PHP, retirándola del modo parcial y sincronizando snapshot completo.
- **Remove bloqueado hasta completar**: una entidad managed-partial no se elimina directamente; primero debe convertirse a entidad completa (`refresh()` o `find()` que la complete) para evitar cascadas o deletes basados en asociaciones no cargadas.
- **Asociaciones siguen fuera del alcance**: `partialManaged(...)` soporta scalar fields y embedded paths; no soporta asociaciones parciales, colecciones parciales, joins ni preload combinado.
- **Loops relacionales deben ignorar managed partials**: orphanRemoval, diff de ManyToMany, cascade graph traversal que dependa de asociaciones cargadas y lógica similar no deben asumir que una managed-partial expone un grafo relacional completo.
- **Sin mezclar modos**: `partialManaged(...)` no se combina con `with(...)`, `select(...)` ni `partial(...)` en la misma query.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir feature tests sobre SQLite para:
  - entidad partial-managed en estado `Managed`,
  - update sólo de fields cargados,
  - verificación de que columnas no cargadas permanecen intactas,
  - bloqueo de `remove()` sobre managed-partial,
  - y upgrade a entidad completa vía `find()` o `refresh()`.

#### Query Builder Joins V1 (`SelectQueryBuilder::as/join/leftJoin`) — Hard rules desde DV-DB-020

Toda implementación o extensión sobre joins declarativos del query layer **DEBE** respetar:

- **El Query Builder sigue modelando, no ejecutando semántica ORM**: `join(...)` y `leftJoin(...)` construyen SQL relacional; no resuelven metadata ORM, no hidratan entidades y no infieren asociaciones automáticamente en este corte.
- **Alcance V1 explícito**: se soportan `INNER JOIN` y `LEFT JOIN` con condición `column-to-column`. No se abre todavía `RIGHT JOIN`, `FULL JOIN`, joins por subquery, `USING(...)`, múltiples condiciones `ON`, ni `GROUP BY`.
- **Alias de tabla deben ser first-class**: tanto la tabla `FROM` como las tablas joined pueden declarar alias. El compilador debe producir SQL con quoting consistente para nombre y alias.
- **Column aliases también deben compilarse limpiamente**: expresiones tipo `table.column AS alias` deben salir correctamente quoted sin degradar el resto del pipeline.
- **Sin raw SQL encubierto**: este corte no debe introducir un canal ambiguo para meter fragmentos SQL arbitrarios en joins. El contrato sigue siendo declarativo y acotado.
- **Compatibilidad hacia atrás total**: `select()/where()/whereIn()/orderBy()/first()/count()` deben seguir funcionando para queries sin joins y preservar alias/joins cuando corresponda.
- **`count()` debe preservar el shape de la query**: si el builder actual tiene alias o joins, el path interno usado por `count()` no puede perderlos silenciosamente.
- **La capa compiler debe seguir siendo dialect-aware y segura**: el quoting de identifiers continúa centralizado en el compilador; joins no justifican bypass de placeholders ni concatenación insegura de values.
- **Este bloque prepara, no resuelve, el fetch relacional del ORM**: cualquier integración con `EntityQuery`, metadata de asociaciones o hydration relacional pertenece a un corte posterior.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir:
  - test unitario de SQL compilado con alias + `INNER JOIN` + `LEFT JOIN`,
  - feature SQLite que ejecute joins reales y valide resultados,
  - y verificación explícita del SQL compilado con bindings.

#### ORM Metadata-Guided To-One Joins V1 (`EntityQuery::join/leftJoin`) — Hard rules desde DV-DB-021

Toda implementación o extensión sobre joins ORM guiados por metadata **DEBE** respetar:

- **Alcance V1 explícito: root associations `to-one` solamente**: `EntityQuery::join()` y `leftJoin()` cubren `ManyToOne` y `OneToOne` root-level. No cubren `OneToMany`, `ManyToMany`, joins encadenados sobre asociaciones de asociaciones, ni colecciones parciales.
- **Metadata primero, SQL después**: la condición `ON` debe derivarse de `EntityAssociationMetadata` (`sourceColumn`, `targetColumn`, owning/inverse) y no de strings duplicados a mano en el call-site ORM.
- **Joined querying, no eager hydration automática**: este corte habilita `where/orderBy/select/value/pluck` sobre `association.field`, pero no obliga a materializar la entidad target joined en la propiedad PHP ni reemplaza `with(...)`.
- **Entity hydration root debe seguir siendo segura**: cuando una query ORM con joins termina en `get()/first()`, la selección root debe evitar colisiones de columnas (`t0.*` o equivalente). No se debe hidratar accidentalmente la entidad root con columnas del target joined.
- **Projection mode sí puede leer fields joined**: `select('post.title')`, `value('profile.bio')`, etc. deben soportarse sólo si la asociación fue joined explícitamente.
- **No mezclar joins ORM con partial modes en V1**: `partial()` y `partialManaged()` no se combinan con joined-association mode hasta que exista un hydration plan relacional claro para ese caso.
- **Errores directos antes que ambigüedad**: usar un path `association.field` sin haber hecho `join('association')` o `leftJoin('association')` debe fallar con un mensaje explícito.
- **Joins to-many deben rechazarse**: si la asociación es `to-many`, la API debe fallar claramente; este corte no autoriza duplicación de roots ni hydration/aggregation implícita de colecciones.
- **Compatibilidad con `with(...)`**: una query puede usar joins ORM para filtrar/proyectar y luego `with(...)` para precargar asociaciones, pero cada mecanismo conserva su responsabilidad: join para SQL, `with(...)` para post-root preload.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir feature tests sobre SQLite para:
  - `ManyToOne` joined por metadata,
  - `OneToOne` inverse joined por metadata,
  - proyección joined (`rows()/value()/pluck()`),
  - filtro root por field del target joined,
  - y rechazo explícito de joins `to-many`.

#### ORM Joined To-One Eager Hydration V1 (`EntityQuery::get/first` sobre joins ORM) — Hard rules desde DV-DB-022

Toda implementación o extensión sobre eager hydration de asociaciones `to-one` usando joins ORM **DEBE** respetar:

- **Sólo activa en entity mode**: esta hidratación automática aplica a `get()` / `first()` cuando la query ya declaró `join(...)` o `leftJoin(...)`. No cambia el contrato de `rows()/firstRow()/value()/pluck()`.
- **Root entity protegida siempre**: la selección SQL del root debe mantenerse aislada (`t0.*` o equivalente). Columnas del target joined no pueden contaminar la hidratación de la entidad root.
- **Target joined debe venir fully-hydratable dentro de su alcance V1**: si se decide materializar una asociación joined, deben seleccionarse columnas suficientes del target para construir una entidad coherente según su metadata actual (fields + embedded columns mapeadas del target).
- **Respeto por `IdentityMap`**: la entidad joined debe hidratarse/reusarse mediante el mismo camino managed estándar. No se deben crear duplicados fuera del `IdentityMap`.
- **Sin prometer grafos arbitrarios**: este corte hidrata la asociación joined explícita `to-one` del root. No hidrata transitivamente asociaciones del target ni abre joins encadenados profundos.
- **`leftJoin()` debe poder producir `null` limpio**: si no existe fila target, la propiedad de asociación debe quedar en `null` sin estados intermedios raros.
- **Back-reference sólo cuando sea `to-one` y obvia**: si la metadata inversa existe y también es `to-one`, puede enlazarse sobre la misma instancia para mantener consistencia local. No se deben fabricar colecciones inverse-side en este corte.
- **Compatibilidad con `with(...)`**: asociaciones ya materializadas por join no deben forzar una segunda carga redundante si pueden evitarse; `with(...)` sigue reservado para lo no cubierto por la join actual.
- **Sin mezcla con partial modes**: `partial()` y `partialManaged()` continúan fuera de alcance cuando la query ORM usa joined eager hydration.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir feature tests sobre SQLite para:
  - `ManyToOne` joined e hidratado en `get()/first()`,
  - `OneToOne` owning o inverse joined e hidratado,
  - `leftJoin()` con target ausente devolviendo `null`,
  - y verificación de que la instancia joined reutiliza `IdentityMap`.

#### ORM Joined To-Many Hydration V1 (`EntityQuery::get` sobre joins ORM) — Hard rules desde DV-DB-023

Toda implementación o extensión sobre joins ORM `to-many` **DEBE** respetar:

- **Alcance V1 explícito base**: las joins `to-many` soportan querying, proyección y materialización de colecciones. La forma inicial de este bloque se consolidó alrededor de `get()`; el windowing root-safe (`limit()/offset()/first()`) queda normado por el corte siguiente.
- **Deduplicación de roots obligatoria**: una query joined `to-many` no puede devolver la misma entidad root repetida por cada fila del target. La salida de `get()` debe colapsar filas repetidas al mismo root managed.
- **Colecciones sin duplicados**: si varias filas representan el mismo target para la misma colección, el array materializado no puede repetir la misma instancia.
- **`IdentityMap` manda también en `to-many`**: tanto roots como targets joined deben pasar por la hidratación managed estándar y reutilizar instancias ya gestionadas.
- **Inicialización coherente en joins externas**: `leftJoin()` sin filas target debe dejar la colección como `[]`, no `null`, ni propiedades sin inicializar.
- **Back-reference sólo cuando el reverso sea `to-one`**: para `OneToMany`, enlazar el `ManyToOne` del target al root es correcto; para `ManyToMany`, este corte no obliga a poblar automáticamente la colección inversa.
- **Snapshots de colección deben resincronizarse**: después de materializar una colección `to-many` vía join, el runtime debe actualizar el snapshot usado por `UnitOfWork` para evitar diffs fantasma en el próximo `flush()`.
- **`count()` debe contar roots, no filas**: si la query ORM incluye joins `to-many`, `count()` no puede reflejar multiplicación por cardinalidad del join.
- **Projection mode sigue siendo row-oriented**: `rows()/pluck()/value()` sobre joins `to-many` pueden devolver repetición de roots porque describen filas SQL, no entidades agrupadas.
- **Este corte por sí solo no cerraba windowing seguro**: la paginación/root-limiting sobre joins `to-many` requiere reglas adicionales para no truncar colecciones por fila SQL.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir feature tests sobre SQLite para:
  - `OneToMany` joined con deduplicación de roots,
  - `ManyToMany` joined con colección materializada,
  - `leftJoin()` sin targets devolviendo `[]`,
  - y `count()` contando roots.

#### ORM Joined To-Many Root-Safe Windowing V1 (`limit/offset/first` sobre joins ORM) — Hard rules desde DV-DB-024

Toda implementación o extensión sobre windowing root-safe en joins ORM `to-many` **DEBE** respetar:

- **La ventana se aplica por root, no por fila SQL**: `limit()`, `offset()` y `first()` deben operar sobre la secuencia de entidades root resultante, nunca sobre filas joined crudas.
- **Correctitud primero en V1**: si para preservar esa semántica hace falta desactivar temporalmente el `LIMIT/OFFSET` SQL row-based y reagrupar en memoria, se permite. Este corte prioriza correctitud observable antes que optimización.
- **`first()` equivale al primer root de la ventana**: cuando hay joins `to-many`, `first()` debe devolver la primera entidad root completa según orden + offset root-safe, no la primera fila del join.
- **`count()` sigue ignorando la ventana**: igual que el builder base, `count()` expresa el total de roots que cumplen el criterio, no el tamaño de la página actual.
- **Orden preservado**: el orden de roots después del agrupamiento debe derivarse del orden SQL original; el primer encuentro de cada root fija su posición relativa.
- **Colecciones completas para cada root visible**: si un root entra en la ventana, su colección joined debe quedar completa dentro del alcance del query, no parcial por efecto del slicing.
- **Sin prometer eficiencia SQL todavía**: este corte no equivale a una implementación por subquery/root-id windowing ni a paginación escalable. La documentación debe mantener explícito ese gap.
- **Projection mode sigue siendo row-oriented**: `rows()/firstRow()/pluck()/value()` no heredan esta semántica root-safe; ahí la ventana sigue describiendo filas SQL.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir feature tests sobre SQLite para:
  - `limit()` root-safe sobre `OneToMany`,
  - `offset()+first()` root-safe sobre `OneToMany` o `ManyToMany`,
  - preservación de colección completa para el root devuelto,
  - y `count()` total consistente aun cuando la query tenga ventana configurada.

#### ORM Joined To-Many Root-Id Windowing V1 (ventana SQL por identifiers antes de hidratar) — Hard rules desde DV-DB-025

Toda optimización posterior al corte `DV-DB-024` sobre windowing joined `to-many` **DEBE** respetar, además de las reglas base anteriores:

- **La ventana visible se resuelve primero como roots, no como filas**: cuando hay joins `to-many` y existe `limit()/offset()/first()`, la implementación puede ejecutar un primer paso SQL que resuelva el conjunto ordenado de identifiers root de la ventana antes de hidratar entidades.
- **Paso 1 sólo decide qué roots entran**: la query inicial de ventana debe seleccionar identifiers root y preservar el orden observable del query original; no debe fingir que ya resolvió colecciones completas.
- **Paso 2 hidrata sólo los roots elegidos**: una segunda query sin `LIMIT/OFFSET` row-based puede cargar las filas joined completas, pero debe restringirse al subconjunto de identifiers root resuelto en el paso 1.
- **Orden final gobernado por la ventana de identifiers**: aunque la segunda query use `WHERE IN (...)`, el orden final de entidades devueltas debe reconstruirse según la secuencia de identifiers seleccionada por la ventana root-safe, no según el orden incidental del `IN`.
- **Colecciones siguen completas**: optimizar la ventana no autoriza truncar asociaciones joined del root visible; si el root entra, su colección debe hidratarse completa dentro del alcance del query.
- **`count()` mantiene semántica de total roots**: esta optimización no cambia el significado de `count()`, que sigue representando el total de roots coincidentes y no el tamaño de la ventana.
- **Projection mode sigue fuera de alcance**: `rows()/firstRow()/pluck()/value()` continúan siendo row-oriented; esta estrategia root-id-driven aplica sólo a entity hydration joined `to-many`.
- **Prueba mínima obligatoria**: toda implementación de este corte debe cubrir al menos un caso donde la ventana se combine con `orderBy()` sobre un field joined `to-many`, verificando que:
  - el root seleccionado sea el correcto,
  - su colección quede completa,
  - y `offset()+first()` siga devolviendo el root esperado.

#### ORM Relational Partial Hydration To-One V1 (`partial/partialManaged` + `join/leftJoin`) — Hard rules desde DV-DB-026

Toda implementación o extensión de partial hydration relacional sobre joins ORM **DEBE** respetar:

- **Sólo asociaciones `to-one` en este corte**: `partial(...)` y `partialManaged(...)` pueden combinarse con `join()/leftJoin()` únicamente cuando todas las asociaciones joined sean `ManyToOne` o `OneToOne`. Cualquier join `to-many` debe rechazarse explícitamente.
- **Selección explícita, sin magia implícita de grafos**: el consumer debe pedir fields concretos del target joined (`post.title`, `profile.bio`, etc.). Este corte no autoriza hidratar asociaciones anidadas (`post.author.name`) ni relaciones profundas.
- **Identificador automático del target parcial**: si se selecciona al menos un field de una asociación joined `to-one`, la implementación debe incluir automáticamente el identifier del target para permitir detached partial coherente o managed partial reusando `IdentityMap`.
- **`leftJoin()` con target ausente devuelve `null`**: cuando la fila joined no existe, la propiedad de asociación debe quedar en `null`, no en una entidad parcial vacía.
- **Managed partial sigue siendo managed partial**: cuando el modo es `partialManaged(...)`, tanto el root como el target joined deben registrarse como managed-partial con su subconjunto real de fields loaded; `find()` / `refresh()` deben poder promover esas instancias a entidades completas.
- **Detached partial sigue fuera de persistencia directa**: cuando el modo es `partial(...)`, ni el root ni el target parcial detached deben venderse como entidades persistibles sin `refresh()`.
- **Se preserva el linking local `to-one` cuando aplica**: si la metadata inversa también es `to-one`, puede enlazarse la back-reference local entre root y target, pero sin inferir otras asociaciones no seleccionadas.
- **No se mezcla con `with(...)` ni con projection mode**: la hidratación relacional parcial sigue siendo un modo separado; no debe mezclarse con `with(...)`, `select(...)->rows()` o estrategias declarativas todavía no abiertas.
- **Prueba mínima obligatoria**: toda ampliación de este bloque debe cubrir feature tests sobre SQLite para:
  - detached partial joined `ManyToOne` o `OneToOne`,
  - `leftJoin()` `to-one` con target ausente devolviendo `null`,
  - managed partial joined `to-one` con flush de un field cargado del target,
  - y rechazo explícito de joins `to-many` en partial mode.

#### ORM Declarative Fetch Strategies V1 (`fetch: 'lazy'|'eager'`) — Hard rules desde DV-DB-027

Toda implementación o extensión de fetch strategies declarativas en el ORM **DEBE** respetar:

- **`lazy` sigue siendo el default honesto**: la ausencia de `fetch` en atributos de asociación conserva el comportamiento actual bajo demanda; no se introduce magia silenciosa para asociaciones no marcadas.
- **`eager` es metadata-driven y explícito**: sólo las asociaciones declaradas con `fetch: 'eager'` pueden autocargarse al hidratar roots.
- **Alcance V1 limitado a roots**: este corte aplica a entidades root resueltas por `find()`, `refresh()`, `get()` y `first()`. No abre proxies lazy transparentes, fetch graphs profundos ni recursión automática multi-hop.
- **Reutilizar infraestructura existente**: la implementación debe apoyarse en `preloadAssociations()` y en los mecanismos ya correctos de `IdentityMap`, `UnitOfWork`, joins hidratados y snapshots; no debe abrir un pipeline paralelo de hidratación eager.
- **No recargar asociaciones ya joined**: si una asociación `eager` ya fue resuelta por `join()/leftJoin()`, el post-load no debe volver a consultarla.
- **Semántica de `leftJoin()`/nulos intacta**: marcar `eager` no autoriza inventar targets; cuando la asociación no existe, debe quedar `null` o colección vacía según corresponda.
- **Sin prometer lazy transparente todavía**: aceptar `fetch: 'lazy'` en metadata no significa proxies o interceptores; en V1 sólo documenta y conserva el modo bajo demanda existente.
- **Prueba mínima obligatoria**: toda implementación de este bloque debe cubrir feature tests sobre SQLite para:
  - `find()` sobre root con asociación `ManyToOne` o `OneToOne` marcada `eager`,
  - `find()` o query root con colección `OneToMany` o `ManyToMany` marcada `eager`,
  - `get()/first()` root respetando asociaciones `eager`,
  - y unit test de metadata verificando `fetch` explícito y default `lazy`.

#### ORM Relational Partial Hydration To-Many V1 (`partial(...)->getPartial()/firstPartial()`) — Hard rules desde DV-DB-028

Toda implementación o extensión de partial hydration relacional sobre joins `to-many` **DEBE** respetar:

- **Detached-only en este corte**: la capacidad se abre únicamente para `partial(...)->getPartial()/firstPartial()`. `partialManaged(...)` sobre joins `to-many` sigue fuera de alcance y debe rechazarse explícitamente.
- **Semántica root-safe, no row-safe**: cuando existan joins `to-many`, el resultado parcial debe deduplicar roots y materializar colecciones parciales por root; `limit()/offset()/firstPartial()` deben aplicarse por root y no por fila SQL raw.
- **Colecciones parciales explícitas**: si el consumer pide `comments.body`, la colección resultante contiene targets parciales detached con sólo los fields seleccionados más el identifier automático del target.
- **`leftJoin()` vacío => `[]`**: una asociación `to-many` sin filas joined debe quedar como colección vacía, no `null`.
- **Sin magic upgrade a managed**: este corte no autoriza snapshots, dirty-check ni flush sobre colecciones parciales detached.
- **Compatibilidad con joins `to-one` mezclados**: si una query combina joins `to-one` y `to-many`, el resultado detached debe seguir enlazando cada asociación según su cardinalidad sin duplicar roots ni targets.
- **Reverse linking sólo donde ya era seguro**: si el target parcial tiene una asociación inversa `to-one`, puede enlazarse de vuelta al root; no se infieren backrefs `to-many`.
- **Prueba mínima obligatoria**: toda implementación de este bloque debe cubrir feature tests sobre SQLite para:
  - `OneToMany` parcial detached con deduplicación de roots,
  - `leftJoin()` `to-many` devolviendo colección vacía,
  - `limit()/offset()/firstPartial()` root-safe sobre join `to-many`,
  - `ManyToMany` parcial detached,
  - y rechazo explícito de `partialManaged()` sobre joins `to-many`.

#### ORM Eager Cascade Batch V1 (segundo salto `eager` sin planners profundos) — Hard rules desde DV-DB-029

Toda implementación o extensión del control sistémico de N+1 en eager preloading **DEBE** respetar:

- **Un salto adicional, no recursión profunda**: este corte sólo autoriza propagar la precarga batch a las asociaciones `eager` de los targets recién cargados. No abre fetch graphs arbitrarios ni planners de múltiples niveles.
- **Reutilizar el pipeline batch existente**: la segunda ola debe construirse sobre `preloadAssociations()` / helpers batch ya existentes, no mediante `find()` por cada target.
- **Excluir la back-reference inmediata**: al propagar el segundo salto `eager`, debe omitirse la asociación inversa directa (`mappedBy` / `inversedBy`) para evitar rebotes triviales como `author -> books -> author`.
- **Mantener semántica de identidad**: los targets de segundo salto deben seguir reusando `IdentityMap` y no duplicar instancias ya gestionadas.
- **No vender lazy/proxies**: este corte reduce N+1 de segundo salto en precarga eager, pero no introduce intercepción transparente de acceso a propiedad.
- **No romper `with(...)` ni fetch declarativo**: la mejora debe aplicar tanto a precargas explícitas como a asociaciones root marcadas `fetch: 'eager'`.
- **Prueba mínima obligatoria**: toda implementación de este bloque debe cubrir feature tests sobre SQLite para:
  - root con colección `eager`,
  - target de esa colección con una asociación `eager` adicional,
  - verificación de que el segundo salto ya quede materializado tras `find()/get()/first()`,
  - y conservación de identidad compartida cuando varios targets apuntan al mismo segundo target.

#### ORM Relational Partial Hydration To-Many Managed V2 (`partialManaged(...)->getPartialManaged()/firstPartialManaged()`) — Hard rules desde DV-DB-031

Toda implementación o extensión de partial hydration relacional managed sobre joins `to-many` **DEBE** respetar:

- **Managed para fields y membership sólo cuando el runtime ya sabe sincronizarlo**: este corte abre edición/flush de fields cargados en roots y targets `to-many`, y además habilita mutación estructural de membership sobre colecciones partial-managed únicamente en rutas ya reconciliables por el runtime actual.
- **`ManyToMany` partial-managed sólo desde owning-side**: las asociaciones `ManyToMany` owning-side cargadas vía `partialManaged(...)` deben participar del mismo reconcile real contra la join-table que ya existe para entidades managed completas. Una mutación sobre inverse-side debe fallar con error claro; no se permiten no-ops silenciosos.
- **`OneToMany` partial-managed exige owning-side consistente**: si se agregan targets a una colección partial-managed `OneToMany`, el owning-side (`mappedBy`) ya debe apuntar al root. Si se remueven targets sin `orphanRemoval`, el owning-side debe haberse limpiado o reasignado; de lo contrario `flush()` debe fallar con error claro.
- **`orphanRemoval` también aplica en partial-managed**: una colección `OneToMany` partial-managed cargada con `orphanRemoval=true` puede remover miembros estructuralmente y el runtime debe marcarlos para eliminación sin exigir refresh completo del root.
- **Targets `ManyToMany` deben tener identifier al sincronizar**: si al momento del reconcile un target de la colección owning-side sigue sin identifier persistente, `flush()` debe fallar con mensaje explícito.
- **Semántica root-safe**: `getPartialManaged()/firstPartialManaged()` sobre joins `to-many` deben deduplicar roots y aplicar `limit()/offset()/firstPartialManaged()` por root, no por fila SQL.
- **Targets también managed**: los items de la colección parcial deben quedar registrados en `IdentityMap` / `UnitOfWork` como managed partials para que sus fields explícitos puedan flushear.
- **`leftJoin()` vacío => `[]`**: una asociación `to-many` sin filas joined debe materializarse como colección vacía también en modo managed.
- **Seguir sin vender proxies**: este bloque no introduce lazy/proxies transparentes ni cambia el acceso a propiedades fuera de los fields explícitamente cargados.
- **Prueba mínima obligatoria**: toda implementación de este bloque debe cubrir feature tests sobre SQLite para:
  - `OneToMany` managed partial con edición de field en root y en target,
  - `ManyToMany` managed partial con edición de field en target y mutación estructural add/remove sobre owning-side,
  - `OneToMany` partial-managed con `orphanRemoval` y alta válida usando owning-side consistente,
  - `leftJoin()` `to-many` devolviendo `[]`,
  - `offset()/firstPartialManaged()` root-safe,
  - y rechazo explícito de mutaciones no reconciliables (inverse `ManyToMany` o `OneToMany` sin owning-side consistente).

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
