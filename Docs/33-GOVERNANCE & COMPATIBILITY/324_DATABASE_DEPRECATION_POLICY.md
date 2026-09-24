# 324_DATABASE_DEPRECATION_POLICY.md

## 1. Propósito

Este documento define la política oficial de **deprecación, transición, compatibilidad y eliminación controlada de APIs** del subsistema **Database de VoltStack**.

La política existe para permitir que Database evolucione de forma continua sin introducir rupturas innecesarias en aplicaciones, paquetes oficiales, extensiones de terceros, drivers, integraciones ORM y componentes internos del framework.

La deprecación en VoltStack no significa eliminación inmediata. Una API deprecada continúa funcionando durante un periodo definido mientras el ecosistema migra hacia su reemplazo.

---

## 2. Objetivos

La política persigue los siguientes objetivos:

- mantener estabilidad en aplicaciones existentes;
- permitir evolución arquitectónica;
- reducir cambios incompatibles inesperados;
- proporcionar rutas claras de migración;
- detectar uso de APIs obsoletas durante desarrollo;
- permitir migraciones progresivas;
- coordinar deprecaciones entre Database y otros módulos Quantum;
- evitar mantener indefinidamente APIs técnicamente deficientes;
- documentar explícitamente cada ciclo de deprecación;
- facilitar herramientas automáticas de actualización.

---

## 3. Principios de diseño

### 3.1 Compatibilidad primero

Siempre que sea técnicamente razonable, una nueva implementación debe coexistir temporalmente con la anterior.

```text
Old API
   │
   ├── remains operational
   │
   ├── emits deprecation metadata
   │
   ▼
Compatibility Layer
   │
   ▼
New API
```

### 3.2 Deprecation is not removal

Una API deprecada:

- sigue siendo soportada;
- puede recibir correcciones críticas;
- no debe recibir nuevas capacidades;
- debe disponer de una alternativa recomendada;
- debe indicar cuándo podría eliminarse.

### 3.3 Migración explícita

Toda deprecación pública debe incluir, cuando sea posible:

```text
Deprecated API
      │
      ▼
Replacement API
      │
      ▼
Migration Example
      │
      ▼
Removal Target
```

### 3.4 Eliminación predecible

Las APIs públicas no deben desaparecer arbitrariamente entre versiones compatibles.

---

## 4. Alcance

Esta política cubre las APIs públicas y contratos relevantes de:

```text
VoltStack Database
│
├── Connections
├── Drivers
├── Query Builder
├── Query AST
├── SQL Compiler
├── ORM
├── Entity Metadata
├── Repositories
├── Unit of Work
├── Identity Map
├── Hydration
├── Transactions
├── Schema
├── Migrations
├── Seeders
├── Pagination
├── Database Events
├── Database Cache
├── CLI
├── Code Generation
├── Telemetry
├── Framework Integrations
└── Extension APIs
```

También aplica a contratos públicos utilizados por paquetes oficiales de VoltStack.

---

## 5. Tipos de deprecación

VoltStack Database distingue varias categorías.

### 5.1 API Deprecation

Afecta métodos, clases, interfaces, traits, atributos o funciones públicas.

Ejemplo:

```php
$connection->executeQuery($sql);
```

puede evolucionar hacia:

```php
$connection->query($sql);
```

### 5.2 Configuration Deprecation

Afecta claves de configuración.

Ejemplo:

```php
'database.default_connection'
```

podría migrar hacia:

```php
'database.default'
```

### 5.3 Behavioral Deprecation

La API permanece, pero un comportamiento histórico será modificado.

Ejemplo:

```text
NULL comparison
implicit casting
transaction retry behavior
hydration strategy
```

Estas deprecaciones requieren especial atención porque pueden no producir errores de compilación o análisis estático.

### 5.4 Extension Point Deprecation

Afecta contratos utilizados por drivers o paquetes externos.

Ejemplo:

```php
interface DriverInterface
```

Si cambia el contrato, debe existir una estrategia especial de transición.

### 5.5 Database Feature Deprecation

Puede afectar funcionalidades completas.

```text
LegacyQueryBuilder
LegacyHydrator
LegacyMigrationFormat
LegacySchemaComparator
```

### 5.6 Platform Deprecation

Permite retirar soporte para:

- versiones antiguas de motores SQL;
- versiones antiguas de PHP;
- drivers obsoletos;
- capacidades específicas de plataforma.

---

## 6. Estados del ciclo de vida

Cada API puede encontrarse en uno de estos estados:

```text
Experimental
     │
     ▼
Stable
     │
     ▼
Deprecated
     │
     ▼
Removal Scheduled
     │
     ▼
Removed
```

### Experimental

No existe garantía completa de compatibilidad.

### Stable

Forma parte del contrato público soportado.

### Deprecated

Continúa disponible, pero no debe utilizarse para nuevo desarrollo.

### Removal Scheduled

Existe una versión objetivo definida para eliminación.

### Removed

La API deja de formar parte del sistema.

---

## 7. Ciclo estándar de deprecación

El proceso recomendado es:

```text
API v1
 │
 │ stable
 ▼
Deprecation Decision
 │
 ▼
Replacement Introduced
 │
 ▼
Deprecation Warning
 │
 ▼
Compatibility Period
 │
 ▼
Migration Tooling
 │
 ▼
Removal in Allowed Version
```

Ejemplo:

```text
VoltStack 2.4
    API estable

VoltStack 2.7
    API deprecada
    nueva API disponible

VoltStack 2.x
    ambas APIs funcionan

VoltStack 3.0
    API antigua puede eliminarse
```

---

## 8. Versionado y deprecación

Esta política complementa:

```text
323_DATABASE_VERSIONING_SYSTEM.md
```

Como regla general:

### Patch release

```text
x.y.Z
```

No debe eliminar APIs públicas.

Puede:

- corregir bugs;
- añadir diagnósticos;
- mejorar mensajes de deprecación.

### Minor release

```text
x.Y.z
```

Puede:

- introducir APIs nuevas;
- marcar APIs existentes como deprecadas;
- añadir adaptadores;
- proporcionar herramientas de migración.

No debe eliminar APIs públicas estables salvo circunstancias excepcionales de seguridad.

### Major release

```text
X.y.z
```

Puede eliminar APIs previamente deprecadas siguiendo esta política.

---

## 9. Periodo mínimo de deprecación

Como política general, una API pública estable debería permanecer deprecada durante al menos:

```text
1 major transition window
```

Ejemplo:

```text
2.5 deprecated
2.6 supported
2.7 supported
2.8 supported
3.0 removal allowed
```

Para contratos ampliamente utilizados puede mantenerse durante más tiempo.

---

## 10. Deprecation Metadata

VoltStack deberá permitir expresar deprecaciones mediante metadata estructurada.

Ejemplo conceptual:

```php
#[Deprecated(
    since: '2.7',
    replacement: 'Connection::query()',
    removal: '3.0'
)]
public function executeQuery(string $sql): Result
{
    // ...
}
```

La metadata permite que diferentes herramientas consuman la información.

---

## 11. Deprecated Attribute

Database podrá proporcionar un atributo estándar:

```php
use VoltStack\Quantum\Database\Support\Deprecated;

#[Deprecated(
    since: '2.7',
    replacement: 'query()',
    removal: '3.0'
)]
public function executeQuery(string $sql): Result
{
}
```

El atributo puede contener:

```text
since
replacement
removal
reason
documentation
severity
```

---

## 12. PHPDoc Deprecation

Para compatibilidad con herramientas PHP existentes también puede utilizarse:

```php
/**
 * @deprecated since 2.7, use query() instead.
 */
public function executeQuery(string $sql): Result
{
}
```

Cuando sea posible se recomienda mantener sincronizados:

```text
PHP Attribute
+
PHPDoc
```

---

## 13. Deprecation Registry

Database debe disponer de un registro central de deprecaciones.

Conceptualmente:

```text
DeprecationRegistry
│
├── API deprecations
├── Configuration deprecations
├── Driver deprecations
├── Behavior deprecations
├── Platform deprecations
└── Removal schedule
```

Ejemplo:

```php
$registry->register(
    id: 'database.connection.execute_query',
    since: '2.7',
    removal: '3.0',
    replacement: 'Connection::query'
);
```

---

## 14. Identificadores de deprecación

Cada deprecación relevante debería poseer un identificador estable.

Ejemplo:

```text
VSDB-DEPR-0001
VSDB-DEPR-0002
VSDB-DEPR-0003
```

Esto permite:

- buscar documentación;
- filtrar logs;
- configurar excepciones;
- generar reportes;
- automatizar migraciones.

---

## 15. Deprecation Manager

El componente central puede representarse como:

```text
Database
   │
   ▼
DeprecationManager
   │
   ├── Registry
   ├── Detector
   ├── Reporter
   ├── Logger
   ├── Telemetry
   └── MigrationHints
```

Interfaz conceptual:

```php
interface DeprecationManagerInterface
{
    public function trigger(
        string $id,
        array $context = []
    ): void;
}
```

---

## 16. Runtime Detection

Cuando se utiliza una API deprecada:

```php
public function executeQuery(string $sql): Result
{
    $this->deprecations->trigger(
        'VSDB-DEPR-0001'
    );

    return $this->query($sql);
}
```

La ejecución puede continuar normalmente.

---

## 17. Modos de reporte

El sistema puede admitir:

```text
silent
log
warning
exception
telemetry
collect
```

Configuración conceptual:

```php
'database.deprecations' => [
    'mode' => 'log',
];
```

---

## 18. Entorno de producción

En producción, por defecto, las deprecaciones no deberían interrumpir solicitudes.

Recomendación:

```text
Production
   │
   ├── collect metrics
   ├── optional logging
   └── no exception
```

---

## 19. Entorno de desarrollo

En desarrollo:

```text
Development
   │
   ├── console warning
   ├── debug toolbar
   ├── stack trace
   ├── replacement suggestion
   └── documentation link
```

Ejemplo:

```text
[VSDB-DEPR-0001]

Connection::executeQuery() is deprecated since VoltStack 2.7.

Use:

Connection::query()

Removal planned for VoltStack 3.0.
```

---

## 20. Entorno de pruebas

Durante testing puede habilitarse:

```text
deprecation → exception
```

Ejemplo:

```php
'database.deprecations.mode' => 'exception';
```

Esto permite impedir que código nuevo introduzca dependencias de APIs obsoletas.

---

## 21. Deprecation Budget

Los proyectos pueden definir un presupuesto temporal de deprecaciones.

Ejemplo:

```text
Allowed deprecations: 15
Detected: 18
Build: FAILED
```

Esto resulta especialmente útil en CI/CD.

---

## 22. Baseline de deprecaciones

Para proyectos existentes puede crearse un baseline.

```text
database-deprecations.baseline
```

Ejemplo:

```text
VSDB-DEPR-0001: 24
VSDB-DEPR-0017: 6
VSDB-DEPR-0021: 2
```

CI puede fallar solamente cuando aparecen nuevas deprecaciones.

---

## 23. Deprecation Collector

Durante ejecución:

```text
Application
     │
     ▼
Database API
     │
     ▼
Deprecation Trigger
     │
     ▼
Deprecation Collector
     │
     ├── count
     ├── source
     ├── stack
     ├── package
     └── replacement
```

---

## 24. Integración con Telemetry

Debe integrarse con:

```text
316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md
```

Métricas posibles:

```text
database.deprecation.count
database.deprecation.unique
database.deprecation.package
database.deprecation.api
```

Ejemplo:

```text
database.deprecation.count{
    id="VSDB-DEPR-0001"
} 145
```

---

## 25. Integración con Logging

Las deprecaciones pueden enviarse a un canal dedicado:

```text
database.deprecations
```

Ejemplo:

```text
[database.deprecations]

VSDB-DEPR-0001
executeQuery() deprecated
replacement=query()
```

Esto evita contaminar innecesariamente logs operacionales.

---

## 26. Integración con Debug Toolbar

La barra de depuración de VoltStack puede mostrar:

```text
Database
 ├── Queries: 27
 ├── Time: 18 ms
 ├── Connections: 2
 └── Deprecations: 3
```

Al abrir:

```text
VSDB-DEPR-0001
Connection::executeQuery()

Called from:
UserRepository.php:84

Replacement:
Connection::query()
```

---

## 27. Integración con Profiler

El profiler puede agregar deprecaciones por:

```text
request
route
controller
repository
package
connection
```

Esto ayuda a localizar deuda técnica.

---

## 28. Deprecación de Query Builder

Ejemplo:

```php
$query->whereRaw(...)
```

podría ser reemplazado por una API AST segura:

```php
$query->where(
    Expression::raw(...)
);
```

La API antigua podría delegar temporalmente:

```text
whereRaw()
    │
    ▼
compatibility adapter
    │
    ▼
AST Expression
```

---

## 29. Deprecación del ORM

Cambios importantes del ORM deben utilizar adaptadores.

Ejemplo:

```text
LegacyEntityManager
       │
       ▼
CompatibilityAdapter
       │
       ▼
EntityManager
```

Esto permite migraciones graduales.

---

## 30. Deprecación de metadata

Si cambia el modelo de metadata:

```text
Annotations
    ↓
Attributes
```

el sistema puede soportar temporalmente ambos lectores:

```text
MetadataResolver
│
├── AttributeReader
└── LegacyAnnotationReader
```

---

## 31. Deprecación de configuración

El sistema puede resolver automáticamente configuraciones antiguas.

Ejemplo:

```php
'database.default_connection'
```

se transforma internamente a:

```php
'database.default'
```

Pipeline:

```text
Config
  │
  ▼
DeprecatedConfigDetector
  │
  ▼
ConfigNormalizer
  │
  ▼
Current Configuration
```

---

## 32. Deprecación de drivers

Los drivers requieren un ciclo especialmente conservador.

Ejemplo:

```text
DriverInterface v1
      │
      ▼
DriverCompatibilityAdapter
      │
      ▼
DriverInterface v2
```

Los paquetes externos deben disponer de tiempo suficiente para actualizarse.

---

## 33. Deprecación de plataformas SQL

Cuando VoltStack deje de soportar una versión de base de datos:

```text
MySQL X
PostgreSQL X
SQLite X
SQL Server X
```

debe anunciarse antes de la eliminación cuando sea posible.

El diagnóstico puede mostrar:

```text
PostgreSQL 14 support is deprecated.

Minimum version in VoltStack Database 4.0:
PostgreSQL 15.
```

---

## 34. Deprecación por seguridad

Existe una excepción al ciclo normal.

Si una API introduce un riesgo grave:

```text
security vulnerability
data corruption
privilege escalation
unsafe SQL generation
```

VoltStack puede:

```text
deprecate immediately
disable behavior
or remove capability
```

incluso en una versión que normalmente conservaría compatibilidad.

Estas excepciones deben documentarse claramente.

---

## 35. Deprecación por corrupción de datos

Una funcionalidad que pueda causar corrupción silenciosa de datos no debe mantenerse únicamente por compatibilidad.

Prioridad:

```text
Data Integrity
     >
Backward Compatibility
```

El sistema puede sustituir el comportamiento por un error explícito.

---

## 36. Deprecation Severity

Se pueden definir niveles:

```text
INFO
NOTICE
WARNING
CRITICAL
```

Ejemplo:

```text
INFO
future optimization change

NOTICE
API replacement available

WARNING
removal scheduled

CRITICAL
unsafe deprecated behavior
```

---

## 37. Static Analysis

Las herramientas de análisis estático deben poder detectar APIs deprecadas.

Integraciones posibles:

```text
PHPStan
Psalm
IDE
VoltStack Analyzer
```

Ejemplo:

```text
Deprecated API detected:

Connection::executeQuery()

Use:
Connection::query()
```

---

## 38. IDE Integration

Los IDEs pueden consumir metadata para mostrar:

```text
strikethrough
replacement
since version
removal version
documentation
```

Esto mejora la experiencia del desarrollador sin necesidad de ejecutar la aplicación.

---

## 39. CLI de deprecaciones

El CLI de Database puede proporcionar:

```bash
php volt database:deprecations
```

Salida:

```text
Database Deprecations

ID                Calls    Replacement
VSDB-DEPR-0001    24       Connection::query()
VSDB-DEPR-0007    8        Schema::table()
VSDB-DEPR-0012    2        EntityManager::persist()
```

---

## 40. Escaneo del proyecto

Comando conceptual:

```bash
php volt database:deprecations:scan
```

Puede analizar:

```text
app/
src/
modules/
packages/
tests/
```

y producir un reporte de migración.

---

## 41. Herramientas automáticas de migración

Cuando sea seguro, VoltStack puede ofrecer:

```bash
php volt database:upgrade
```

Pipeline:

```text
Source Code
    │
    ▼
Deprecated API Scanner
    │
    ▼
Migration Rules
    │
    ▼
Code Transformer
    │
    ▼
Updated Code
```

Los cambios automáticos deben ser deterministas y revisables.

---

## 42. Dry Run

Toda migración automática debería admitir:

```bash
php volt database:upgrade --dry-run
```

Salida:

```text
12 files analyzed
7 deprecated APIs detected
5 automatic replacements available
2 require manual migration
```

---

## 43. Migration Rules

Las reglas pueden representarse como:

```php
MigrationRule::from(
    'Connection::executeQuery'
)->to(
    'Connection::query'
);
```

Estas reglas pueden ser reutilizadas por:

```text
CLI
IDE
CI
code fixer
upgrade assistant
```

---

## 44. Compatibilidad interna

Las APIs internas no tienen necesariamente las mismas garantías que las públicas.

Convención:

```text
@internal
```

Ejemplo:

```php
/**
 * @internal
 */
final class QueryPlannerState
{
}
```

Código externo no debería depender de estos contratos.

---

## 45. Public API Boundary

Database debe definir claramente qué constituye API pública.

```text
Database Public API
│
├── documented classes
├── documented interfaces
├── documented configuration
├── documented events
├── documented extension points
└── documented CLI contracts
```

Todo lo demás puede considerarse interno salvo indicación explícita.

---

## 46. APIs experimentales

Una API experimental puede evolucionar sin seguir el ciclo completo.

Ejemplo:

```php
#[Experimental]
final class AdaptiveQueryPlanner
{
}
```

Sin embargo, los cambios deben seguir documentándose.

---

## 47. Feature Flags

Algunas transiciones complejas pueden utilizar feature flags.

```php
'database.features.new_query_planner' => true
```

Esto permite:

```text
Old Engine
   │
   ├── compatibility
   │
Feature Flag
   │
   ▼
New Engine
```

Antes de convertir el nuevo comportamiento en predeterminado.

---

## 48. Dual Runtime

Para migraciones críticas puede existir temporalmente:

```text
Database Runtime
│
├── Legacy Runtime
└── Current Runtime
```

Esto debe considerarse excepcional debido al coste de mantenimiento.

---

## 49. Compatibilidad de paquetes

Los paquetes oficiales deben declarar:

```text
minimum database version
maximum tested version
deprecated API usage
```

Ejemplo conceptual:

```json
{
  "voltstack/database": "^3.0"
}
```

---

## 50. Ecosistema externo

Los autores de paquetes externos deben disponer de:

- changelog;
- upgrade guide;
- deprecation identifiers;
- replacement APIs;
- release candidates cuando corresponda;
- documentación de contratos modificados.

---

## 51. Deprecation Dashboard

VoltStack Telemetry puede proporcionar un panel:

```text
Database Deprecations
────────────────────────────

Total calls             2,481
Unique APIs                 14
Critical                     0
Scheduled for next major     5
```

Esto resulta especialmente útil en aplicaciones empresariales.

---

## 52. Changelog

Cada deprecación debe aparecer en el changelog.

Ejemplo:

```markdown
### Deprecated

- `Connection::executeQuery()` has been deprecated.
  Use `Connection::query()` instead.
  Scheduled for removal in 3.0.
```

---

## 53. Upgrade Guide

Cada major release debe incluir una guía de actualización.

Ejemplo:

```text
UPGRADE_3.0.md
```

Secciones:

```text
Removed APIs
Changed behaviors
Configuration migrations
Driver changes
ORM changes
Schema changes
Migration changes
```

---

## 54. Release Candidates

Antes de una eliminación importante se recomienda utilizar:

```text
alpha
beta
RC
```

para permitir que el ecosistema pruebe compatibilidad antes del release estable.

---

## 55. CI/CD

Pipeline recomendado:

```text
Application
   │
   ▼
Tests
   │
   ▼
Deprecation Collector
   │
   ▼
Baseline Comparison
   │
   ├── no new deprecations → PASS
   │
   └── new deprecations → FAIL/WARN
```

---

## 56. Política para código nuevo

Código nuevo dentro del propio VoltStack Database:

```text
MUST NOT
```

utilizar APIs ya deprecadas salvo dentro de capas explícitas de compatibilidad.

---

## 57. Política para tests

Los tests deben separar:

```text
Current API Tests
Legacy Compatibility Tests
Deprecation Tests
Removal Tests
```

Los tests de compatibilidad pueden eliminarse junto con la API obsoleta.

---

## 58. Pruebas de deprecación

Ejemplo conceptual:

```php
public function test_old_api_triggers_deprecation(): void
{
    $collector = new DeprecationCollector();

    $connection->executeQuery('SELECT 1');

    $this->assertTrue(
        $collector->contains('VSDB-DEPR-0001')
    );
}
```

---

## 59. Documentación histórica

Las deprecaciones eliminadas deben permanecer documentadas en archivos históricos.

```text
docs/
└── database/
    └── upgrades/
        ├── 2.x-to-3.0.md
        ├── 3.x-to-4.0.md
        └── ...
```

Esto permite migrar aplicaciones antiguas incluso años después.

---

## 60. Comparación conceptual con Laravel

Laravel tradicionalmente ofrece una experiencia de actualización pragmática mediante:

- versionado mayor;
- upgrade guides;
- documentación de cambios;
- periodos de soporte definidos;
- herramientas del ecosistema para automatizar actualizaciones.

VoltStack debe conservar esa simplicidad para el desarrollador, pero formalizar adicionalmente la deprecación como un subsistema observable.

```text
Laravel-inspired DX
        +
Structured Deprecation Metadata
        +
Telemetry
        +
Migration Tooling
        =
VoltStack Database
```

---

## 61. Comparación conceptual con Symfony y Doctrine

Symfony posee una cultura especialmente sólida alrededor de:

```text
deprecation notices
compatibility layers
major-version preparation
debug tooling
```

Doctrine también mantiene ciclos de compatibilidad y evolución contractual.

VoltStack Database adopta este enfoque disciplinado, integrándolo profundamente con:

```text
CLI
Telemetry
Profiler
Debug Toolbar
Static Analysis
Code Generation
Upgrade Automation
```

---

## 62. Diferenciación de VoltStack

La política no debe limitarse a emitir advertencias.

VoltStack busca convertir la deprecación en un proceso completo:

```text
Detect
  │
  ▼
Explain
  │
  ▼
Locate
  │
  ▼
Measure
  │
  ▼
Suggest
  │
  ▼
Automate
  │
  ▼
Verify
  │
  ▼
Remove
```

---

## 63. Arquitectura propuesta

```text
┌──────────────────────────────────────────────┐
│             VoltStack Database               │
├──────────────────────────────────────────────┤
│                                              │
│              Public Database API             │
│                       │                      │
│                       ▼                      │
│              Deprecation Detector            │
│                       │                      │
│                       ▼                      │
│              Deprecation Manager             │
│                       │                      │
│       ┌───────────────┼───────────────┐      │
│       ▼               ▼               ▼      │
│    Registry        Collector       Reporter  │
│       │               │               │      │
│       └───────────────┼───────────────┘      │
│                       ▼                      │
│                 Integrations                 │
│                       │                      │
│    ┌──────────┬───────┼───────┬─────────┐   │
│    ▼          ▼       ▼       ▼         ▼   │
│ Logging   Telemetry   CLI   Profiler    IDE │
│                                              │
├──────────────────────────────────────────────┤
│               Migration Layer                │
│                                              │
│ Scanner → Rules → Transformer → Validator    │
└──────────────────────────────────────────────┘
```

---

## 64. Flujo completo

```text
Developer uses deprecated API
            │
            ▼
Deprecation Detector
            │
            ▼
Deprecation Registry
            │
            ▼
Deprecation Manager
            │
     ┌──────┼────────┐
     ▼      ▼        ▼
   Log   Telemetry   Collector
                    │
                    ▼
               Developer Tools
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
       CLI         IDE       Profiler
        │
        ▼
Migration Recommendation
        │
        ▼
Automated Upgrade
        │
        ▼
Compatibility Verification
```

---

## 65. Contratos propuestos

```php
interface DeprecationRegistryInterface
{
    public function register(Deprecation $deprecation): void;

    public function get(string $id): ?Deprecation;
}
```

```php
interface DeprecationReporterInterface
{
    public function report(
        Deprecation $deprecation,
        DeprecationContext $context
    ): void;
}
```

```php
interface DeprecationCollectorInterface
{
    public function collect(
        DeprecationOccurrence $occurrence
    ): void;
}
```

---

## 66. Modelo de dominio

Entidades principales:

```text
Deprecation
DeprecationId
DeprecationContext
DeprecationOccurrence
DeprecationReplacement
DeprecationSeverity
RemovalVersion
MigrationRule
DeprecationBaseline
```

Estas estructuras deben permanecer independientes de transportes concretos de logging o telemetry.

---

## 67. Requisitos de implementación

La implementación deberá:

1. introducir overhead mínimo cuando el sistema esté desactivado;
2. evitar generar el mismo warning miles de veces por request;
3. permitir agregación;
4. conservar contexto suficiente para diagnóstico;
5. funcionar en procesos persistentes;
6. ser compatible con concurrencia;
7. evitar estado global mutable no controlado;
8. integrarse con testing;
9. admitir paquetes externos;
10. permitir eliminación limpia de capas legacy.

---

## 68. FrankenPHP y procesos persistentes

Debido a que VoltStack utiliza FrankenPHP como servidor predeterminado, el sistema de deprecaciones debe considerar workers persistentes.

No debe asumirse:

```text
1 request = 1 process
```

El collector deberá diferenciar correctamente:

```text
process lifetime
worker lifetime
request lifetime
```

y limpiar el contexto correspondiente al finalizar cada request.

---

## 69. Concurrencia

En servidores concurrentes:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

la información de deprecación específica de una solicitud no debe filtrarse hacia otra.

Debe utilizarse el sistema de contexto de ejecución definido por VoltStack.

---

## 70. Rendimiento

El camino rápido debe aproximarse a:

```text
if (!deprecations_enabled) {
    continue;
}
```

Las operaciones costosas, como generación de stack traces, deben activarse únicamente cuando sean necesarias.

---

## 71. Seguridad

Los mensajes de deprecación no deben exponer accidentalmente:

- contraseñas;
- DSN completos;
- tokens;
- parámetros sensibles;
- datos personales;
- SQL con secretos embebidos.

El contexto debe pasar por mecanismos de sanitización.

---

## 72. Compatibilidad con observabilidad

La política se integra directamente con:

```text
Logging
Metrics
Tracing
OpenTelemetry
Profiler
Debug Toolbar
```

pero el núcleo de Database no debe depender obligatoriamente de un proveedor específico.

---

## 73. Relación con otros documentos

Esta especificación se relaciona especialmente con:

```text
307_DATABASE_SCHEMA_DEVELOPER_EXPERIENCE.md
308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md
309_DATABASE_CLI_SYSTEM.md
310_DATABASE_CODE_GENERATION_SYSTEM.md
311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md
312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md
313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md
314_DATABASE_CACHE_INTEGRATION_SYSTEM.md
315_DATABASE_EVENT_SYSTEM_INTEGRATION.md
316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md
317_DATABASE_VALIDATION_INTEGRATION_SYSTEM.md
318_DATABASE_AUTHENTICATION_INTEGRATION_SYSTEM.md
319_DATABASE_AUTHORIZATION_INTEGRATION_SYSTEM.md
320_DATABASE_JOB_AND_QUEUE_INTEGRATION_SYSTEM.md
321_DATABASE_HTTP_REQUEST_LIFECYCLE_INTEGRATION.md
322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md
323_DATABASE_VERSIONING_SYSTEM.md
```

---

## 74. Decisiones arquitectónicas

### Decisión 1

Las APIs públicas no se eliminarán normalmente en versiones minor.

### Decisión 2

Toda deprecación pública debe indicar una alternativa cuando exista.

### Decisión 3

Las deprecaciones deberán ser identificables mediante IDs estables.

### Decisión 4

Database dispondrá de un sistema estructurado de recolección de deprecaciones.

### Decisión 5

Las deprecaciones serán observables mediante logging, telemetry y herramientas de desarrollo.

### Decisión 6

El modo de producción no deberá convertir deprecaciones normales en errores fatales.

### Decisión 7

Testing podrá convertir deprecaciones en excepciones.

### Decisión 8

VoltStack deberá facilitar herramientas automáticas de migración cuando la transformación sea segura.

### Decisión 9

La integridad y seguridad de datos tienen prioridad sobre compatibilidad hacia atrás.

### Decisión 10

Las capas legacy deberán diseñarse para poder eliminarse completamente al finalizar el ciclo de compatibilidad.

---

## 75. Resultado esperado

Con esta política, una aplicación puede evolucionar:

```text
VoltStack Database v2
        │
        ▼
Deprecation Detection
        │
        ▼
Migration Guidance
        │
        ▼
Automated Refactoring
        │
        ▼
Compatibility Verification
        │
        ▼
VoltStack Database v3
```

sin depender de migraciones abruptas o cambios silenciosos.

---

## 76. Conclusión

El sistema de deprecación de VoltStack Database debe tratar la compatibilidad como una responsabilidad arquitectónica y no únicamente como una convención documental.

La estrategia combina la facilidad de evolución esperada por desarrolladores de Laravel con la disciplina de deprecaciones y compatibilidad característica del ecosistema Symfony/Doctrine, ampliándola mediante observabilidad, análisis estático, CI/CD y automatización.

El principio fundamental queda definido como:

```text
Stable API
    ↓
Explicit Deprecation
    ↓
Compatible Transition
    ↓
Observable Usage
    ↓
Guided Migration
    ↓
Verified Upgrade
    ↓
Controlled Removal
```

Esto permitirá que **VoltStack Database evolucione agresivamente a nivel interno sin obligar a las aplicaciones a evolucionar de manera abrupta**.

---

**Documento:** `324_DATABASE_DEPRECATION_POLICY.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Estado:** Architectural Specification  
