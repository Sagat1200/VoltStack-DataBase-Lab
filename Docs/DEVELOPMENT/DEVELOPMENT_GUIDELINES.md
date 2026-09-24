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
2. helper limitado
3. comandos CLI basicos
4. telemetria basica

### Prioridad 6: ORM y capas avanzadas

1. metadata
2. entity manager
3. hydration
4. persistence planning
5. relationships
6. identity map y unit of work

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

- integracion sobre el mismo Query/Execution Engine,
- ausencia de SQL dentro del ORM,
- ausencia de estado compartido entre requests,
- y pruebas separadas por metadata, hydration y persistence.

### Security, Telemetry, CLI y Plugins

Todo trabajo sobre `216-340` debe respetar:

- capas finas sobre servicios existentes,
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
