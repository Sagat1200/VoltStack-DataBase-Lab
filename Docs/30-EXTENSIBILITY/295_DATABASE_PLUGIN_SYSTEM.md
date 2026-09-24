# 295_DATABASE_PLUGIN_SYSTEM.md

# VoltStack Quantum Database
## Database Plugin System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 295 — Database Plugin System  
**Bloque:** 30 — Extensibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `294_DATABASE_EXTENSION_ARCHITECTURE.md`  
**Siguiente documento:** `296_DATABASE_CUSTOM_DRIVER_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial del **Database Plugin System** de:

```text
VoltStack/Quantum/Database
```

El sistema permitirá distribuir, descubrir, registrar, habilitar, deshabilitar, configurar, validar y diagnosticar paquetes que proporcionen una o varias extensiones para Database.

La relación fundamental será:

```text
Package
   ↓
Plugin
   ↓
Extensions
   ↓
Extension Points
   ↓
Database Runtime
```

La regla central será:

> **Un plugin de VoltStack Database será una unidad explícita de distribución y administración capaz de proporcionar una o más extensiones, pero su instalación no implicará autoridad ilimitada, su descubrimiento no implicará activación y su desactivación no podrá dejar al runtime en un estado estructural parcialmente mutado.**

Formalmente:

```text
Plugin
=
Identity
+
Manifest
+
Package Provenance
+
Compatibility
+
Dependencies
+
Configuration
+
Extension Contributions
+
Lifecycle
+
Governance
```

pero:

```text
Plugin
≠
Arbitrary Runtime Mutation
```

---

# 2. Objetivos

El sistema deberá permitir:

1. distribuir extensiones Database;
2. descubrir plugins instalados;
3. identificar cada plugin de forma estable;
4. declarar versiones;
5. declarar compatibilidad;
6. declarar dependencias;
7. proporcionar una o varias extensiones;
8. habilitar plugins;
9. deshabilitar plugins;
10. configurar plugins;
11. validar configuración;
12. resolver dependencias;
13. detectar conflictos;
14. ordenar bootstrap;
15. diagnosticar fallos;
16. conservar provenance;
17. gobernar privilegios;
18. soportar plugins oficiales;
19. soportar plugins Quantum;
20. soportar plugins de aplicación;
21. soportar plugins de terceros;
22. preservar seguridad;
23. preservar determinismo;
24. preservar scope isolation;
25. preservar persistent runtime safety.

---

# 3. Plugin ≠ Extension

El documento anterior estableció:

```text
Extension
≠
Plugin
```

Una extensión representa:

```text
Database capability contribution
```

Un plugin representa:

```text
Distribution
+
Administration
+
Lifecycle
+
Packaging
```

de una o varias extensiones.

Ejemplo:

```text
acme/database-spatial
        │
        ▼
SpatialDatabasePlugin
        │
        ├── SpatialTypeExtension
        ├── SpatialQueryExtension
        ├── SpatialCompilerExtension
        └── SpatialSchemaExtension
```

---

# 4. Plugin ≠ Package

Un package Composer podrá contener:

```text
0 plugins
1 plugin
N plugins
```

Por tanto:

```text
ComposerPackage
≠
DatabasePlugin
```

---

# 5. Plugin ≠ Service Provider

Un Service Provider puede ayudar a integrar el package con VoltStack.

Sin embargo:

```text
DatabasePlugin
≠
ServiceProvider
```

El plugin tendrá semántica específica de Database.

---

# 6. Plugin ≠ Runtime Module

Un plugin instalado no significa que exista una instancia mutable permanente del plugin controlando cada operación.

La arquitectura preferirá:

```text
Plugin
   ↓
Bootstrap
   ↓
Extension Contributions
   ↓
CompiledExtensionGraph
   ↓
Runtime
```

---

# 7. Plugin ≠ Hot Patch

No será un mecanismo oficial para:

```text
replace methods
rewrite classes
inject private properties
modify frozen registries
```

---

# 8. Arquitectura general

```text
Composer / Package Source
          │
          ▼
   Package Discovery
          │
          ▼
     Plugin Manifest
          │
          ▼
    Plugin Discovery
          │
          ▼
     Plugin Registry
          │
          ▼
 ┌────────┼─────────┐
 ▼        ▼         ▼
State   Dependencies Compatibility
 │        │         │
 └────────┼─────────┘
          ▼
    Plugin Resolver
          │
          ▼
  Configuration Validation
          │
          ▼
   Permission Validation
          │
          ▼
     Plugin Bootstrap
          │
          ▼
 Extension Contributions
          │
          ▼
 Database Extension System
          │
          ▼
 CompiledExtensionGraph
          │
          ▼
     Database Runtime
```

---

# 9. Plugin Lifecycle

El lifecycle estructural será:

```text
DISCOVERED
    ↓
REGISTERED
    ↓
RESOLVED
    ↓
VALIDATED
    ↓
ENABLED
    ↓
BOOTSTRAPPED
    ↓
ACTIVE
```

Otros estados:

```text
DISABLED
INCOMPATIBLE
BLOCKED
FAILED
REMOVED
```

---

# 10. Installed ≠ Enabled

Regla fundamental:

```text
INSTALLED
≠
ENABLED
```

Que un package exista en:

```text
vendor/
```

no significa necesariamente que sus plugins deban participar en Database.

---

# 11. Enabled ≠ Active

Un plugin puede estar configurado como:

```text
ENABLED
```

pero no llegar a:

```text
ACTIVE
```

por:

```text
dependency failure
compatibility failure
invalid configuration
permission rejection
bootstrap failure
```

---

# 12. Discovered ≠ Enabled

Igualmente:

```text
DISCOVERED
≠
ENABLED
```

---

# 13. Plugin Identity

Todo plugin tendrá un:

```text
PluginId
```

estable.

Ejemplos:

```text
voltstack.database.mysql
voltstack.database.postgresql
voltstack.database.vector
acme.database.spatial
acme.database.analytics
```

---

# 14. PluginId

Conceptualmente:

```php
final readonly class PluginId
{
    public function __construct(
        public string $vendor,
        public string $name,
    ) {}
}
```

Deberá ser:

```text
stable
unique
normalized
case-safe
```

---

# 15. PluginId ≠ Package Name

Ejemplo:

```text
Composer:
acme/voltstack-spatial

Plugin:
acme.database.spatial
```

Podrán coincidir conceptualmente, pero no serán la misma identidad.

---

# 16. Plugin Version

Cada plugin tendrá:

```text
PluginVersion
```

que podrá derivarse del package version.

Pero:

```text
PluginVersion
≠
PackageVersion
```

como concepto arquitectónico.

---

# 17. Plugin Manifest

Todo plugin deberá proporcionar metadata declarativa.

Ejemplo conceptual:

```php
return [
    'id' => 'acme.database.spatial',
    'version' => '2.1.0',

    'database' => '^1.0',

    'extensions' => [
        Acme\Spatial\SpatialTypeExtension::class,
        Acme\Spatial\SpatialQueryExtension::class,
        Acme\Spatial\SpatialCompilerExtension::class,
    ],

    'requires' => [
        'capabilities' => [
            'database.platform.spatial',
        ],
    ],
];
```

---

# 18. Manifest ≠ Plugin Instance

El manifest deberá poder inspeccionarse sin instanciar el plugin completo.

Esto permitirá:

```text
dependency resolution
compatibility analysis
security inspection
diagnostics
```

antes de ejecutar lógica del plugin.

---

# 19. Declarative Discovery

Se preferirá que discovery utilice:

```text
package metadata
static manifest
generated package index
```

en lugar de ejecutar código arbitrario.

---

# 20. Manifest Model

Conceptualmente:

```php
final readonly class DatabasePluginManifest
{
    public function __construct(
        public PluginId $id,
        public PluginVersion $version,
        public PluginCompatibility $compatibility,
        public array $dependencies,
        public array $extensions,
        public array $permissions,
        public PluginConfigurationSchema $configuration,
    ) {}
}
```

---

# 21. Manifest Immutability

Una vez descubierto:

```text
PluginManifest
```

será tratado como metadata inmutable.

---

# 22. Package Provenance

El sistema deberá conservar:

```text
package name
package version
package source
plugin id
plugin version
```

cuando sea posible.

---

# 23. Provenance

Permitirá responder:

```text
Where did this plugin come from?
```

Ejemplo:

```text
Plugin:
acme.database.spatial

Package:
acme/voltstack-spatial

Package Version:
2.1.4

Plugin Version:
2.1.0
```

---

# 24. Plugin Discovery Sources

Podrán incluir:

```text
Composer metadata
VoltStack package metadata
explicit application registration
local application plugins
official Quantum package registry
```

---

# 25. Composer Discovery

Composer podrá declarar metadata como:

```json
{
    "extra": {
        "voltstack": {
            "database-plugins": [
                "Acme\\Spatial\\DatabasePlugin"
            ]
        }
    }
}
```

La sintaxis final podrá evolucionar.

---

# 26. Discovery Cache

La lista de plugins podrá compilarse para evitar escanear packages en cada request.

```text
Composer Metadata
      ↓
Discovery Compiler
      ↓
Plugin Discovery Cache
```

---

# 27. Persistent Runtime

En FrankenPHP:

```text
Plugin discovery
```

no deberá repetirse innecesariamente en cada request.

---

# 28. Discovery Fingerprint

Podrá depender de:

```text
composer.lock
installed package set
application plugin configuration
plugin manifests
```

---

# 29. Plugin Registry

El registro mantendrá:

```text
PluginRegistry
├── manifests
├── installation state
├── enablement state
├── compatibility
├── dependencies
├── diagnostics
└── provenance
```

---

# 30. Registry Lifecycle

```text
BUILDING
   ↓
RESOLVING
   ↓
VALIDATING
   ↓
COMPILED
   ↓
FROZEN
```

---

# 31. Frozen Plugin Registry

Una vez iniciado el runtime:

```text
enable()
disable()
install()
uninstall()
```

no modificarán directamente el registry activo.

---

# 32. Structural Change Requires Rebuild

Cambiar plugins deberá producir:

```text
New Plugin Configuration
        ↓
Rebuild Plugin Graph
        ↓
Rebuild Extension Graph
        ↓
New Runtime Generation
```

---

# 33. No Partial Runtime Mutation

Nunca:

```text
Request A
  ↓
Plugin enabled midway
  ↓
Request A continues with different architecture
```

---

# 34. Plugin State Model

Estados conceptuales:

```text
DISCOVERED
DISABLED
ENABLED
BLOCKED
INCOMPATIBLE
BOOTSTRAPPING
ACTIVE
FAILED
REMOVED
```

---

# 35. State Transition

Ejemplo:

```text
DISCOVERED
   ↓ enable
ENABLED
   ↓ resolve
VALIDATED
   ↓ bootstrap
ACTIVE
```

---

# 36. Disabled Plugin

Un plugin deshabilitado:

```text
may remain installed
```

pero:

```text
must not contribute active extensions
```

---

# 37. Plugin Enablement Configuration

Ejemplo:

```php
'database.plugins' => [
    'acme.database.spatial' => [
        'enabled' => true,
    ],
];
```

---

# 38. Default Enablement

Podrán existir políticas:

```text
AUTO
EXPLICIT
CORE_REQUIRED
DEVELOPMENT_ONLY
```

---

# 39. AUTO

Plugins seguros/oficiales podrán habilitarse automáticamente si están instalados.

---

# 40. EXPLICIT

Plugins con impacto significativo podrán requerir activación explícita.

---

# 41. CORE_REQUIRED

Plugins que formen parte de una instalación concreta y sean obligatorios podrán marcarse como requeridos.

---

# 42. DEVELOPMENT_ONLY

Plugins de:

```text
debugging
profiling
testing
```

podrán bloquearse automáticamente en producción.

---

# 43. Environment-aware Activation

Ejemplo:

```text
development → enabled
testing     → enabled
production  → disabled
```

cuando la política lo permita.

---

# 44. Environment ≠ Security Proof

Que:

```text
APP_ENV=test
```

no será suficiente para conceder operaciones destructivas.

Se mantendrán las garantías del:

```text
286_DATABASE_DATABASE_TEST_ENVIRONMENT_SYSTEM.md
```

---

# 45. Plugin Dependencies

Un plugin podrá depender de:

```text
another plugin
extension
capability
Database API version
Extension API version
PHP extension
runtime
external library
```

---

# 46. Plugin Dependency

Ejemplo:

```text
acme.database.analytics
        ↓ requires
acme.database.columnar-driver
```

---

# 47. Extension Dependency

Un plugin podrá requerir:

```text
extension:
database.type.json
```

sin depender del plugin concreto que la proporciona.

---

# 48. Capability Dependency

Preferido cuando:

```text
capability
```

representa realmente el requisito.

Ejemplo:

```text
requires:
database.capability.vector
```

---

# 49. Plugin Dependency ≠ Package Dependency

Composer puede resolver dependencias de packages.

VoltStack resolverá además dependencias semánticas entre plugins/extensions/capabilities.

---

# 50. Hard Dependencies

Si falta una dependencia requerida:

```text
Plugin
→
BLOCKED
```

---

# 51. Optional Dependencies

Permitirán integración condicional.

Ejemplo:

```text
SpatialPlugin
  └── optionally integrates TelemetryPlugin
```

---

# 52. Dependency Graph

```text
Plugin A
   ↓
Plugin B
   ↓
Plugin C
```

se representará como grafo explícito.

---

# 53. Cyclic Plugin Dependency

```text
A → B
B → A
```

será inválido por defecto.

---

# 54. Topological Bootstrap

El orden deberá derivarse de:

```text
dependency graph
```

y no de:

```text
filesystem
Composer iteration order
registration accident
```

---

# 55. Deterministic Resolution

Mismos:

```text
plugins
versions
configuration
capabilities
```

deberán producir el mismo:

```text
ResolvedPluginGraph
```

---

# 56. Plugin Compatibility

Cada plugin podrá declarar compatibilidad con:

```text
VoltStack Database version
Database Extension API
PHP version
platform capabilities
runtime capabilities
other plugin versions
```

---

# 57. Compatibility Model

Conceptualmente:

```text
COMPATIBLE
INCOMPATIBLE
UNKNOWN
```

---

# 58. UNKNOWN ≠ COMPATIBLE

Si un requisito crítico no puede validarse:

```text
UNKNOWN
```

no será convertido silenciosamente en:

```text
COMPATIBLE
```

---

# 59. Version Constraint

Ejemplo:

```text
database-extension-api: ^1.0
```

---

# 60. Capability Compatibility

Ejemplo:

```text
requires:
database.capability.full_text_search
```

---

# 61. Platform-specific Plugin

Cuando sea realmente necesario podrá declarar:

```text
platform:
postgresql
```

Pero capability-based compatibility será preferida cuando sea suficiente.

---

# 62. Plugin Configuration

Cada plugin tendrá un namespace propio.

Ejemplo:

```php
'database.plugins.acme_spatial' => [
    'enabled' => true,
    'srid' => 4326,
];
```

---

# 63. Configuration Schema

El plugin deberá poder proporcionar:

```text
PluginConfigurationSchema
```

---

# 64. Typed Configuration

Ejemplo conceptual:

```php
final readonly class SpatialPluginConfiguration
{
    public function __construct(
        public bool $enabled,
        public int $defaultSrid,
    ) {}
}
```

---

# 65. Configuration Validation

Deberá ocurrir antes de activar el plugin cuando sea posible.

---

# 66. Invalid Configuration

Resultado:

```text
PluginConfigurationException
```

y el plugin no deberá quedar parcialmente activo.

---

# 67. Configuration Defaults

Podrán existir defaults seguros.

---

# 68. Plugin Configuration ≠ Runtime Context

No se almacenará:

```text
current tenant
current transaction
current request
```

en la configuración del plugin.

---

# 69. Structural Configuration

Ejemplos:

```text
driver implementation
type registrations
query extensions
compiler handlers
```

deberán congelarse después del bootstrap.

---

# 70. Scoped Runtime State

Ejemplos:

```text
current operation
tenant
transaction
connection
```

deberán obtenerse del scope apropiado.

---

# 71. Plugin Bootstrap

Contrato conceptual:

```php
interface DatabasePlugin
{
    public function manifest(): DatabasePluginManifest;

    public function register(
        DatabasePluginRegistrar $registrar
    ): void;
}
```

---

# 72. Plugin Registrar

No será:

```text
global service container
```

Será una API restringida.

---

# 73. Plugin Contribution

El plugin podrá registrar:

```text
DatabaseExtension
```

o descriptors/factories de extensiones.

---

# 74. Plugin → Extension Boundary

Preferido:

```text
Plugin
   ↓
register extensions
   ↓
Extension Architecture
   ↓
extension points
```

en lugar de:

```text
Plugin
   ↓
modify Database internals directly
```

---

# 75. Multiple Extensions

Ejemplo:

```php
final class SpatialDatabasePlugin implements DatabasePlugin
{
    public function register(DatabasePluginRegistrar $registrar): void
    {
        $registrar->extension(new SpatialTypeExtension());
        $registrar->extension(new SpatialQueryExtension());
        $registrar->extension(new SpatialCompilerExtension());
    }
}
```

---

# 76. Plugin Bootstrap Idempotence

El bootstrap estructural deberá diseñarse para evitar registros duplicados.

Idealmente:

```text
bootstrap generation
```

se ejecuta una vez por runtime generation.

---

# 77. Duplicate Contribution

Será error explícito si viola identidad/contratos.

---

# 78. Plugin Ordering

Los plugins no deberán depender de:

```text
priority=9999
```

para resolver dependencias reales.

---

# 79. Priority

Sólo se utilizará cuando el extension point permita orden relativo semánticamente válido.

---

# 80. Plugin Conflicts

El sistema deberá detectar:

```text
duplicate PluginId
incompatible dependencies
exclusive provider collision
duplicate extension identity
alias collision
capability contradiction
invalid replacement
```

---

# 81. Conflict ≠ Last One Wins

Regla:

```text
Conflict
→
Diagnostic/Error
```

no:

```text
Conflict
→
Silent Override
```

---

# 82. Plugin Permissions

Los plugins podrán declarar capacidades sensibles.

Ejemplo:

```text
database.credentials
database.network
database.raw_sql
database.admin
filesystem
process_execution
testing.destructive
```

---

# 83. Permission Descriptor

Conceptualmente:

```php
enum PluginPermission
{
    case DATABASE_NETWORK;
    case DATABASE_CREDENTIALS;
    case DATABASE_RAW_SQL;
    case DATABASE_ADMIN;
    case FILESYSTEM;
    case PROCESS_EXECUTION;
}
```

---

# 84. Permission Metadata ≠ Sandbox

PHP instalado dentro de la aplicación continúa siendo código con capacidad de ejecución.

El sistema de permisos servirá para:

```text
visibility
policy
diagnostics
audit
governance
```

No deberá venderse conceptualmente como aislamiento de seguridad absoluto.

---

# 85. Least Plugin Authority

VoltStack deberá proporcionar a un plugin sólo APIs necesarias para sus contribuciones.

---

# 86. Credential Access

Un plugin que no necesite credenciales no deberá recibir:

```text
CredentialProvider
```

---

# 87. Raw SQL Permission

Plugins que generen SQL raw deberán declararlo cuando corresponda.

---

# 88. Admin Permission

Operaciones administrativas deberán usar:

```text
Database Administration System
```

y sus controles.

---

# 89. Testing Destructive Permission

No bastará con que el plugin declare:

```text
testing.destructive
```

El entorno deberá demostrar que la operación es segura.

---

# 90. Plugin Trust Classification

Podrá existir:

```text
CORE
OFFICIAL
APPLICATION
THIRD_PARTY
UNVERIFIED
```

---

# 91. Trust ≠ Permission

Un plugin oficial no obtiene automáticamente todas las capacidades.

---

# 92. Trust ≠ Correctness

Igualmente:

```text
Official
≠
Bug Free
```

---

# 93. Plugin Isolation

Se distinguirán dos conceptos:

```text
structural isolation
runtime state isolation
```

---

# 94. Structural Isolation

Un plugin sólo podrá contribuir mediante puntos autorizados.

---

# 95. Runtime State Isolation

Un plugin no deberá almacenar state scoped de otra operación.

---

# 96. Static Mutable State

Patrón prohibido:

```php
final class TenantPlugin
{
    public static ?string $tenant = null;
}
```

---

# 97. FrankenPHP Safety

En FrankenPHP:

```text
Plugin definitions
```

podrán sobrevivir múltiples requests si son inmutables.

Pero:

```text
Plugin request state
```

no.

---

# 98. RoadRunner Safety

Misma regla.

---

# 99. OpenSwoole Safety

Además deberá considerarse:

```text
coroutine isolation
```

---

# 100. Worker Scope

Un plugin podrá tener estado worker-scoped únicamente si:

```text
explicit
safe
bounded
reset-aware
```

---

# 101. Plugin Runtime Context

Podrá recibir contextos específicos:

```text
DatabaseContext
TransactionContext
ConnectionContext
```

únicamente en operaciones que los necesiten.

---

# 102. Plugin Context ≠ Global Locator

No será una forma de obtener cualquier servicio arbitrario.

---

# 103. Plugin Activation Architecture

```text
Installed Packages
       ↓
Discovery
       ↓
Plugin Registry
       ↓
Enablement Policy
       ↓
Dependency Resolution
       ↓
Compatibility
       ↓
Permission Policy
       ↓
Configuration Validation
       ↓
Bootstrap
       ↓
Extensions
       ↓
Compiled Runtime
```

---

# 104. Atomic Structural Activation

La activación deberá ser conceptualmente atómica:

```text
Validate everything
      ↓
Build new graph
      ↓
Freeze
      ↓
Publish runtime generation
```

No:

```text
register half
fail
leave half registered
```

---

# 105. Bootstrap Transaction

No implica una transacción de DB.

Es un concepto estructural:

```text
all-or-nothing runtime construction
```

---

# 106. Failed Activation

Si falla:

```text
new runtime graph
```

no deberá publicarse.

El runtime anterior podrá continuar si la arquitectura de despliegue lo permite.

---

# 107. Plugin Disable

Deshabilitar un plugin significa:

```text
future runtime generation
```

sin sus contribuciones.

---

# 108. Disable ≠ Runtime Object Deletion

No se recorrerán requests activos eliminando handlers.

---

# 109. Safe Disable

Antes de deshabilitar deberá analizarse:

```text
dependent plugins
dependent configuration
serialized metadata
schema dependencies
runtime requirements
```

---

# 110. Dependency Block

Si:

```text
Plugin B
requires
Plugin A
```

entonces:

```text
disable A
```

deberá:

```text
reject
```

o deshabilitar explícitamente dependientes bajo una operación planificada.

Nunca dejarlos activos rotos.

---

# 111. Plugin Uninstallation

Desinstalar un package es una operación distinta de deshabilitar un plugin.

```text
Disable
≠
Uninstall
```

---

# 112. Uninstall Safety

Un plugin puede haber introducido:

```text
schema objects
custom types
metadata
migration history
configuration
```

que sobreviven a su código.

---

# 113. Code Removal ≠ Data Removal

Regla crítica:

```text
Remove Plugin Package
≠
Remove Plugin Data
```

---

# 114. Uninstall Plan

Podrá requerir:

```text
preflight
dependency analysis
schema analysis
data retention decision
migration/down migration
configuration cleanup
package removal
cache invalidation
runtime rebuild
```

---

# 115. Automatic Destructive Cleanup

No deberá ocurrir por defecto.

Ejemplo:

```text
composer remove spatial-plugin
```

no deberá significar automáticamente:

```text
DROP spatial_data
```

---

# 116. Plugin Update

Actualizar un plugin podrá cambiar:

```text
code
manifest
extensions
configuration schema
capabilities
metadata
database schema requirements
```

---

# 117. Update ≠ Simple File Replacement

Desde la perspectiva Database:

```text
Plugin Update
```

puede requerir transición estructural.

---

# 118. Update Plan

Conceptualmente:

```text
Current Plugin
      ↓
Target Plugin
      ↓
Compatibility Check
      ↓
Dependency Check
      ↓
Configuration Migration
      ↓
Schema/Data Migration
      ↓
Extension Graph Rebuild
      ↓
Validation
      ↓
Activation
```

---

# 119. Plugin Schema Migration

Un plugin podrá distribuir migrations propias.

---

# 120. Plugin Migration Namespace

Deberán poseer identidad propia.

Ejemplo:

```text
acme.database.spatial::20260921_001
```

---

# 121. Plugin Migrations ≠ Core Migrations

Deberán distinguirse en metadata/repository cuando sea necesario.

---

# 122. Migration Ownership

El sistema deberá conocer:

```text
which plugin owns migration X
```

---

# 123. Plugin Disabled with Existing Schema

Puede ser válido.

Ejemplo:

```text
plugin disabled
schema retained
```

si la aplicación ya no usa esa funcionalidad.

---

# 124. Unknown Custom Types

Si se deshabilita un plugin que define tipos necesarios para metadata activa:

```text
bootstrap
→
must fail or mark incompatible
```

No deberá degradarse silenciosamente.

---

# 125. Plugin Cache Interaction

Cambios en plugins podrán invalidar:

```text
metadata cache
compiled query cache
schema cache
hydration plan cache
extension graph cache
```

---

# 126. Plugin Generation

Se definirá:

```text
PluginGeneration
```

para representar la configuración estructural activa.

---

# 127. PluginGeneration ≠ PluginVersion

Ejemplo:

```text
same package versions
different enabled plugin set
```

produce distinta generación.

---

# 128. Plugin Fingerprint

Conceptualmente:

```text
PluginFingerprint
=
H(
  PluginIds
  + PluginVersions
  + Enablement
  + StructuralConfiguration
  + DependencyResolution
)
```

---

# 129. Runtime Fingerprint

Podrá incorporar:

```text
PluginFingerprint
+
ExtensionFingerprint
+
DatabaseConfigurationGeneration
```

---

# 130. Plugin Diagnostics

El sistema deberá responder:

```text
Which plugins were discovered?
Which are enabled?
Which are active?
Which are blocked?
Why?
Which package provided them?
What extensions do they provide?
What permissions do they request?
What dependencies do they have?
```

---

# 131. CLI List

Futuro:

```bash
php voltstack database:plugins
```

Salida conceptual:

```text
Plugin                       Version    State
------------------------------------------------
voltstack.database.mysql     1.0.0      ACTIVE
acme.database.spatial        2.1.0      ACTIVE
acme.database.analytics      1.3.0      DISABLED
legacy.database.foo          0.8.0      INCOMPATIBLE
```

---

# 132. CLI Inspect

```bash
php voltstack database:plugin acme.database.spatial
```

Podrá mostrar:

```text
package
version
state
compatibility
dependencies
extensions
permissions
configuration status
diagnostics
```

---

# 133. CLI Enable

Futuro:

```bash
php voltstack database:plugin:enable acme.database.spatial
```

No deberá mutar directamente un worker activo.

Deberá modificar configuración estructural y requerir/reconstruir runtime según entorno.

---

# 134. CLI Disable

```bash
php voltstack database:plugin:disable acme.database.spatial
```

deberá realizar dependency preflight.

---

# 135. CLI Doctor

Podrá existir:

```bash
php voltstack database:plugin:doctor
```

para analizar:

```text
missing dependencies
incompatible versions
conflicts
invalid manifests
configuration problems
permission concerns
```

---

# 136. Diagnostics ≠ Secrets

Nunca mostrar:

```text
password
token
private key
full DSN with secret
```

---

# 137. Plugin Events

Eventos administrativos posibles:

```text
PluginDiscovered
PluginRegistered
PluginEnabled
PluginDisabled
PluginValidated
PluginActivated
PluginBlocked
PluginFailed
```

---

# 138. Event Timing

Eventos de bootstrap no deberán convertirse en una forma de modificar arbitrariamente el graph.

---

# 139. Plugin Telemetry

Podrán registrarse:

```text
plugin bootstrap duration
plugin failure count
plugin activation state
```

con cardinalidad controlada.

---

# 140. No plugin ID explosion

Plugin IDs provienen de un conjunto estructural acotado y son apropiados para diagnostics.

Aun así deberá respetarse la política global de telemetría.

---

# 141. Plugin Audit

Operaciones administrativas como:

```text
enable
disable
update
uninstall
```

podrán auditarse.

---

# 142. Plugin Failure Architecture

Tipos principales:

```text
Discovery Failure
Manifest Failure
Dependency Failure
Compatibility Failure
Configuration Failure
Permission Failure
Bootstrap Failure
Runtime Contribution Failure
Update Failure
Uninstall Failure
```

---

# 143. Exception Taxonomy

```text
DatabasePluginException
├── PluginDiscoveryException
├── InvalidPluginManifestException
├── DuplicatePluginException
├── PluginDependencyException
├── PluginDependencyCycleException
├── PluginCompatibilityException
├── PluginConfigurationException
├── PluginPermissionException
├── PluginConflictException
├── PluginBootstrapException
├── PluginActivationException
├── PluginDisableException
├── PluginUpdateException
├── PluginUninstallException
└── FrozenPluginRegistryException
```

---

# 144. Required Plugin Failure

Si un plugin requerido falla:

```text
Database bootstrap
→
FAIL
```

---

# 145. Optional Plugin Failure

Puede resultar:

```text
BLOCKED
```

o:

```text
FAILED
```

con diagnóstico visible.

---

# 146. Optional Failure ≠ Silent Ignore

Regla:

```text
Optional
≠
Invisible
```

---

# 147. Runtime Contribution Failure

Una vez activo, el fallo de una extensión se rige por la semántica del subsistema.

Ejemplo:

```text
Telemetry exporter failure
```

puede ser degradable.

Pero:

```text
Custom driver commit failure
```

puede producir:

```text
UNKNOWN transaction outcome
```

---

# 148. Plugin Failure Must Not Rewrite Semantics

El Plugin System no atrapará un error crítico y devolverá éxito ficticio.

---

# 149. Persistent Runtime Architecture

```text
Application Bootstrap
        ↓
Discover Plugins
        ↓
Resolve/Validate
        ↓
Compile Plugin Graph
        ↓
Compile Extension Graph
        ↓
Freeze
        ↓
Start Worker
        ↓
┌──────────────────────────┐
│ Request 1                │
│ scoped state only        │
└──────────────────────────┘
        ↓ reset
┌──────────────────────────┐
│ Request 2                │
│ scoped state only        │
└──────────────────────────┘
```

---

# 150. Plugin Definitions

Podrán ser compartidas si:

```text
immutable
stateless
thread/coroutine safe where relevant
```

---

# 151. Plugin State

Estado mutable deberá vivir en:

```text
RequestScope
OperationScope
TransactionScope
ConnectionScope
WorkerScope
```

según contrato.

---

# 152. Worker Restart

Cambios estructurales de plugins podrán requerir:

```text
worker reload
```

dependiendo del runtime.

---

# 153. FrankenPHP

FrankenPHP será la integración de referencia para validar:

```text
plugin graph persistence
runtime generation
request isolation
reload behavior
```

---

# 154. RoadRunner

Deberá respetar el mismo modelo.

---

# 155. OpenSwoole

Además deberá garantizar:

```text
coroutine-safe scoped plugin state
```

---

# 156. Plugin Testing Architecture

Todo plugin oficial deberá incluir:

```text
manifest tests
discovery tests
dependency tests
compatibility tests
configuration tests
extension registration tests
scope isolation tests
failure tests
persistent runtime tests
```

---

# 157. Plugin Contract Test

VoltStack podrá proporcionar:

```php
abstract class DatabasePluginContractTest
{
    abstract protected function plugin(): DatabasePlugin;
}
```

con assertions compartidas.

---

# 158. Manifest Test

Validará:

```text
valid ID
valid version
known permissions
valid dependencies
valid extension declarations
```

---

# 159. Discovery Test

Demostrará que el plugin puede descubrirse mediante su mecanismo declarado.

---

# 160. Dependency Test

Validará:

```text
missing dependency
compatible dependency
incompatible dependency
dependency cycle
```

---

# 161. Configuration Test

Validará:

```text
defaults
valid config
invalid config
unknown config
secret redaction
```

---

# 162. Activation Test

Validará:

```text
Plugin
→
Extensions
→
CompiledExtensionGraph
```

---

# 163. Disable Test

Validará que sus contribuciones desaparezcan de la siguiente generación.

---

# 164. Persistent Runtime Test

Deberá demostrar que:

```text
Request A state
```

no aparece en:

```text
Request B
```

---

# 165. Uninstall Test

Plugins con schema/data ownership deberán probar:

```text
safe removal planning
```

cuando soporten uninstall asistido.

---

# 166. Plugin Performance

Se medirá:

```text
discovery cost
bootstrap cost
compiled dispatch cost
runtime contribution cost
```

cuando sea relevante.

---

# 167. No Per-query Plugin Scan

Nunca debería ser necesario:

```php
foreach ($allPlugins as $plugin) {
    if ($plugin->supports($query)) {
        // ...
    }
}
```

para cada query si puede compilarse un dispatch eficiente.

---

# 168. Compile-time Resolution

Preferido:

```text
Plugin Graph
   ↓
Extension Graph
   ↓
Dispatch Tables
```

---

# 169. Plugin Count Scalability

El costo runtime de plugins no relacionados con una operación deberá permanecer mínimo.

---

# 170. Plugin Security

El sistema deberá asumir que un plugin PHP instalado forma parte del código ejecutable de la aplicación.

Por ello la seguridad real incluye:

```text
package provenance
dependency review
version pinning
code review
permission visibility
runtime contracts
least authority APIs
```

---

# 171. Plugin Signature

Una futura distribución oficial podría añadir:

```text
package signing
manifest signing
checksum verification
```

pero no será requisito de la arquitectura V1.

---

# 172. Plugin Repository

Database no requerirá inicialmente un marketplace propio.

Composer podrá actuar como sistema principal de distribución.

---

# 173. Future Plugin Catalog

VoltStack podrá proporcionar un catálogo oficial para descubrir:

```text
verified plugins
official plugins
community plugins
```

sin cambiar el modelo arquitectónico.

---

# 174. Official ≠ Installed Automatically

Un plugin oficial puede continuar siendo opcional.

---

# 175. Quantum Plugins

Los módulos Quantum podrán proporcionar plugins Database.

Ejemplo:

```text
VoltStack/Quantum/Multitenancy
        ↓
DatabaseMultitenancyPlugin
        ↓
Tenant Query Extension
Tenant Connection Extension
Tenant Migration Extension
```

---

# 176. Multitenancy Remains Optional

`Quantum/Database` no dependerá de:

```text
Quantum/Multitenancy
```

---

# 177. SaaS Integration

Igualmente:

```text
VoltStack/Quantum/SaaS
```

podrá registrar plugins/extensions opcionales.

---

# 178. Driver Plugins

Un custom driver podrá distribuirse como plugin.

Ejemplo:

```text
acme/database-oracle
        │
        ▼
OracleDatabasePlugin
        │
        ├── OracleDriverExtension
        ├── OracleDialectExtension
        ├── OraclePlatformExtension
        └── OracleSchemaExtension
```

---

# 179. Plugin Composition

Un plugin podrá agrupar varias extensiones que en conjunto implementan una plataforma.

Esto evita obligar al usuario a registrar manualmente cada pieza.

---

# 180. Plugin Granularity

No toda clase deberá convertirse en plugin.

El plugin representa una unidad razonable de:

```text
installation
configuration
versioning
administration
```

---

# 181. Fine-grained Extensions

Dentro del plugin:

```text
extensions
```

pueden ser más pequeñas y especializadas.

---

# 182. Plugin Replacement

Un plugin no podrá declarar:

```text
replace all database
```

como mecanismo genérico.

---

# 183. Explicit Provider Replacement

Cuando un contrato admita providers exclusivos podrá existir:

```text
replaces:
    provider-id
```

con validación.

---

# 184. Core Replacement

Reemplazar un componente central sólo será permitido donde exista un extension point público específico.

---

# 185. Plugin Capability Contributions

Plugins podrán aportar:

```text
CapabilityDefinition
CapabilityEvidenceProvider
```

mediante sus extensiones.

---

# 186. Plugin ≠ Capability Authority

La existencia del plugin no prueba que una capability esté disponible en el servidor.

Ejemplo:

```text
SpatialPlugin installed
```

no implica:

```text
PostGIS extension installed
```

---

# 187. Capability Discovery

El sistema deberá resolver:

```text
Plugin Installed
+
Server Evidence
+
Platform Evidence
+
Configuration
→
Capability Decision
```

---

# 188. Plugin Configuration Secrets

Secrets no deberán almacenarse directamente en manifest.

---

# 189. Secret References

Configuración podrá utilizar:

```text
SecretReference
CredentialReference
```

---

# 190. Manifest Serialization

El manifest podrá cachearse.

Nunca deberá contener secretos runtime.

---

# 191. Plugin Metadata Cache

Podrá contener:

```text
IDs
versions
dependencies
extensions
permissions
compatibility constraints
```

---

# 192. Cache Invalidation

Cambios en:

```text
composer.lock
plugin manifests
plugin config
Database Extension API
```

podrán invalidar el cache.

---

# 193. Plugin Graph Compilation

Resultado conceptual:

```php
final readonly class CompiledPluginGraph
{
    public function __construct(
        public PluginGeneration $generation,
        public array $plugins,
        public array $activationOrder,
        public array $dependencies,
        public PluginFingerprint $fingerprint,
    ) {}
}
```

---

# 194. CompiledPlugin

Podrá contener:

```text
PluginId
Version
State
ResolvedDependencies
Configuration
ExtensionDescriptors
Permissions
Provenance
```

---

# 195. Runtime Graph Separation

```text
CompiledPluginGraph
```

describe plugins.

```text
CompiledExtensionGraph
```

describe contribuciones Database.

Son estructuras relacionadas, pero distintas.

---

# 196. Plugin Graph ≠ Extension Graph

```text
Plugin Graph
→ distribution/admin dependencies

Extension Graph
→ Database capability contributions
```

---

# 197. Plugin Dependencies vs Extension Dependencies

Ambos niveles deberán poder coexistir.

---

# 198. Example

```text
Plugin A
 ├── Extension X
 └── Extension Y

Plugin B
 └── Extension Z
       └── requires Extension X
```

El resolver deberá considerar ambas relaciones.

---

# 199. Resolution Pipeline

```text
Discover Packages
      ↓
Parse Plugin Manifests
      ↓
Resolve Enablement
      ↓
Resolve Plugin Dependencies
      ↓
Validate Compatibility
      ↓
Validate Configuration
      ↓
Validate Permissions
      ↓
Instantiate/Bootstrap Plugins
      ↓
Collect Extensions
      ↓
Resolve Extension Dependencies
      ↓
Compile Plugin Graph
      ↓
Compile Extension Graph
      ↓
Freeze
```

---

# 200. Error Before Runtime

Siempre que sea posible:

```text
structural error
```

deberá detectarse durante bootstrap.

---

# 201. Lazy Plugin Bootstrap

No será la estrategia por defecto para plugins estructurales.

La arquitectura priorizará:

```text
fail early
```

---

# 202. Lazy Runtime Components

Un plugin podrá registrar factories lazy para recursos costosos.

Pero sus:

```text
identity
compatibility
dependencies
configuration
```

ya deberán estar validadas.

---

# 203. Lazy ≠ Unvalidated

Regla:

```text
Lazy Construction
≠
Lazy Structural Validation
```

---

# 204. Plugin Update Compatibility

Podrán existir políticas:

```text
BACKWARD_COMPATIBLE
MIGRATION_REQUIRED
BREAKING
UNKNOWN
```

---

# 205. Update Preflight

Antes de actualización gestionada:

```text
current version
target version
dependency compatibility
config compatibility
schema migration requirements
```

deberán analizarse.

---

# 206. Rollback

Rollback de package code no implica automáticamente rollback de:

```text
schema
data
external resources
```

---

# 207. Plugin Rollback ≠ Database Rollback

Regla fundamental.

---

# 208. Failed Update

Deberá evitar dejar:

```text
new code
old schema
partial config
mixed runtime graph
```

sin diagnóstico.

---

# 209. Deployment Integration

Actualizaciones complejas deberán coordinarse con el sistema de deployment de la aplicación.

Database Plugin System no pretenderá resolver por sí solo todo el deployment distribuido.

---

# 210. Plugin Health

Podrá existir health metadata para plugins que dependan de recursos externos.

Pero:

```text
Plugin Health
≠
Database Health
```

---

# 211. Plugin Health ≠ Capability

Un plugin puede estar activo pero una capability externa temporalmente no disponible.

---

# 212. Diagnostics Integration

Podrá integrarse con:

```text
281_DATABASE_DIAGNOSTICS_SYSTEM.md
```

---

# 213. Administration Integration

Operaciones privilegiadas podrán integrarse con:

```text
282_DATABASE_ADMINISTRATION_SYSTEM.md
```

---

# 214. Telemetry Integration

Podrá integrarse con:

```text
216_DATABASE_TELEMETRY_ARCHITECTURE.md
```

sin convertir telemetry en requisito para plugins.

---

# 215. Testing Integration

Se integrará con:

```text
283_DATABASE_TESTING_ARCHITECTURE.md
```

y especialmente:

```text
292_DATABASE_DRIVER_CONFORMANCE_TESTING_SYSTEM.md
293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

para plugins que implementen drivers o componentes sensibles.

---

# 216. Arquitectura de directorios propuesta

```text
src/Quantum/Database/Extension/Plugin/
├── Contract/
│   ├── DatabasePlugin.php
│   ├── PluginManifestProvider.php
│   └── PluginRegistrar.php
│
├── Identity/
│   ├── PluginId.php
│   ├── PluginVersion.php
│   ├── PluginGeneration.php
│   └── PluginFingerprint.php
│
├── Manifest/
│   ├── DatabasePluginManifest.php
│   ├── PluginManifestLoader.php
│   ├── PluginManifestParser.php
│   └── PluginManifestValidator.php
│
├── Discovery/
│   ├── PluginDiscovery.php
│   ├── ComposerPluginDiscovery.php
│   ├── ApplicationPluginDiscovery.php
│   └── PluginDiscoveryCache.php
│
├── Registry/
│   ├── PluginRegistry.php
│   ├── MutablePluginRegistry.php
│   └── FrozenPluginRegistry.php
│
├── State/
│   ├── PluginState.php
│   ├── PluginEnablement.php
│   └── PluginStateResolver.php
│
├── Dependency/
│   ├── PluginDependency.php
│   ├── PluginDependencyGraph.php
│   ├── PluginDependencyResolver.php
│   └── PluginDependencyCycleDetector.php
│
├── Compatibility/
│   ├── PluginCompatibility.php
│   ├── PluginCompatibilityStatus.php
│   └── PluginCompatibilityValidator.php
│
├── Configuration/
│   ├── PluginConfiguration.php
│   ├── PluginConfigurationSchema.php
│   └── PluginConfigurationValidator.php
│
├── Permission/
│   ├── PluginPermission.php
│   ├── PluginPermissionSet.php
│   └── PluginPermissionValidator.php
│
├── Bootstrap/
│   ├── PluginBootstrapper.php
│   ├── PluginBootstrapContext.php
│   └── PluginContributionCollector.php
│
├── Compilation/
│   ├── PluginGraphCompiler.php
│   ├── CompiledPlugin.php
│   └── CompiledPluginGraph.php
│
├── Provenance/
│   ├── PluginProvenance.php
│   └── PackageProvenance.php
│
├── Diagnostics/
│   ├── PluginDiagnostic.php
│   ├── PluginInspector.php
│   └── PluginDoctor.php
│
├── Lifecycle/
│   ├── PluginLifecycle.php
│   ├── PluginActivationPlan.php
│   ├── PluginDisablePlan.php
│   ├── PluginUpdatePlan.php
│   └── PluginUninstallPlan.php
│
└── Exception/
    ├── DatabasePluginException.php
    ├── PluginDiscoveryException.php
    ├── InvalidPluginManifestException.php
    ├── PluginDependencyException.php
    ├── PluginCompatibilityException.php
    ├── PluginConfigurationException.php
    ├── PluginPermissionException.php
    ├── PluginBootstrapException.php
    └── FrozenPluginRegistryException.php
```

---

# 217. Separación conceptual de namespaces

Podrá evolucionar finalmente hacia:

```text
Extension/
├── Contract/
├── Registry/
├── Capability/
└── Plugin/
```

para mantener:

```text
Extension Architecture
```

separada de:

```text
Plugin Management
```

---

# 218. Architectural Invariants

## DB-PLUGIN-001

Plugin ≠ Extension.

## DB-PLUGIN-002

Plugin ≠ Package.

## DB-PLUGIN-003

Plugin ≠ Service Provider.

## DB-PLUGIN-004

Plugin ≠ Hot Patch.

## DB-PLUGIN-005

Installed ≠ Enabled.

## DB-PLUGIN-006

Enabled ≠ Active.

## DB-PLUGIN-007

Discovered ≠ Enabled.

## DB-PLUGIN-008

Manifest ≠ Plugin Instance.

## DB-PLUGIN-009

Todo plugin tendrá PluginId estable.

## DB-PLUGIN-010

PluginId ≠ Package Name.

## DB-PLUGIN-011

PluginVersion ≠ PackageVersion conceptualmente.

## DB-PLUGIN-012

Discovery será declarativo cuando sea posible.

## DB-PLUGIN-013

Discovery no ejecutará innecesariamente lógica operacional.

## DB-PLUGIN-014

Package provenance será preservada.

## DB-PLUGIN-015

Plugin Registry será congelable.

## DB-PLUGIN-016

Registry congelado no se modificará durante requests.

## DB-PLUGIN-017

Cambio estructural requerirá nueva generación.

## DB-PLUGIN-018

Activación no dejará estado parcialmente registrado.

## DB-PLUGIN-019

Plugin deshabilitado no contribuirá extensiones activas.

## DB-PLUGIN-020

APP_ENV=test ≠ prueba de seguridad destructiva.

## DB-PLUGIN-021

Plugin dependency ≠ Package dependency.

## DB-PLUGIN-022

Hard dependency faltante bloqueará activación.

## DB-PLUGIN-023

Optional dependency no bloqueará necesariamente activación.

## DB-PLUGIN-024

Capability dependency será preferida cuando represente el requisito real.

## DB-PLUGIN-025

Ciclos serán rechazados por defecto.

## DB-PLUGIN-026

Bootstrap order será determinista.

## DB-PLUGIN-027

Filesystem order no definirá bootstrap.

## DB-PLUGIN-028

UNKNOWN compatibility ≠ COMPATIBLE.

## DB-PLUGIN-029

Configuración será namespaced.

## DB-PLUGIN-030

Configuración será validada antes de activación.

## DB-PLUGIN-031

Invalid configuration no dejará plugin parcialmente activo.

## DB-PLUGIN-032

Plugin Configuration ≠ Runtime Context.

## DB-PLUGIN-033

Plugin Registrar ≠ Service Container.

## DB-PLUGIN-034

Plugins contribuirán Database functionality mediante Extension Architecture.

## DB-PLUGIN-035

Duplicate contributions no serán silenciosas.

## DB-PLUGIN-036

Priority no sustituirá dependencias.

## DB-PLUGIN-037

Conflict ≠ Last One Wins.

## DB-PLUGIN-038

Permission metadata ≠ Sandbox.

## DB-PLUGIN-039

Trust ≠ Permission.

## DB-PLUGIN-040

Trust ≠ Correctness.

## DB-PLUGIN-041

Static mutable request state estará prohibido.

## DB-PLUGIN-042

Static mutable tenant state estará prohibido.

## DB-PLUGIN-043

Static mutable transaction state estará prohibido.

## DB-PLUGIN-044

Persistent workers compartirán sólo estado seguro.

## DB-PLUGIN-045

OpenSwoole deberá preservar coroutine isolation.

## DB-PLUGIN-046

Activación estructural será atómica conceptualmente.

## DB-PLUGIN-047

Failed graph no será publicado.

## DB-PLUGIN-048

Disable afectará una nueva runtime generation.

## DB-PLUGIN-049

Disable ≠ Uninstall.

## DB-PLUGIN-050

Code Removal ≠ Data Removal.

## DB-PLUGIN-051

Uninstall destructivo no será automático.

## DB-PLUGIN-052

Plugin update podrá requerir schema/data migration.

## DB-PLUGIN-053

Plugin migrations tendrán provenance.

## DB-PLUGIN-054

Plugin migrations serán distinguibles de core migrations.

## DB-PLUGIN-055

Unknown required custom type impedirá bootstrap correcto.

## DB-PLUGIN-056

Cambios estructurales invalidarán caches afectados.

## DB-PLUGIN-057

PluginGeneration ≠ PluginVersion.

## DB-PLUGIN-058

Plugin diagnostics no expondrán secretos.

## DB-PLUGIN-059

Optional plugin failure ≠ Silent Ignore.

## DB-PLUGIN-060

Plugin System no convertirá fallos semánticos en éxito.

## DB-PLUGIN-061

Plugin definitions podrán sobrevivir requests sólo si son seguras.

## DB-PLUGIN-062

Scoped state no sobrevivirá fuera de su scope.

## DB-PLUGIN-063

Official plugins tendrán contract tests.

## DB-PLUGIN-064

External behavior requerirá integration evidence.

## DB-PLUGIN-065

Runtime no recorrerá todos los plugins por query si puede compilar dispatch.

## DB-PLUGIN-066

Plugin PHP instalado será tratado como código ejecutable confiado por la aplicación, no como sandbox.

## DB-PLUGIN-067

Official ≠ Automatically Installed.

## DB-PLUGIN-068

Multitenancy continuará siendo opcional.

## DB-PLUGIN-069

SaaS continuará siendo opcional.

## DB-PLUGIN-070

Plugin granularity será distinta de class granularity.

## DB-PLUGIN-071

Plugin ≠ Capability Authority.

## DB-PLUGIN-072

Plugin instalado no prueba capability externa.

## DB-PLUGIN-073

Secrets no vivirán en manifest.

## DB-PLUGIN-074

Plugin Graph ≠ Extension Graph.

## DB-PLUGIN-075

Structural errors deberán detectarse temprano.

## DB-PLUGIN-076

Lazy Construction ≠ Lazy Structural Validation.

## DB-PLUGIN-077

Plugin Rollback ≠ Database Rollback.

## DB-PLUGIN-078

Plugin Health ≠ Database Health.

## DB-PLUGIN-079

Plugin Health ≠ Capability.

## DB-PLUGIN-080

Core no dependerá de third-party plugin implementation.

## DB-PLUGIN-081

Plugins respetarán Extension API pública.

## DB-PLUGIN-082

Internals no serán modificados mediante reflection como contrato soportado.

## DB-PLUGIN-083

Plugin enablement será explícitamente resoluble.

## DB-PLUGIN-084

Plugin dependencies tendrán diagnostics.

## DB-PLUGIN-085

Plugin provenance será consultable.

## DB-PLUGIN-086

Permission requests serán consultables.

## DB-PLUGIN-087

Un plugin no recibirá credenciales si no las necesita.

## DB-PLUGIN-088

Destructive test permission no reemplazará environment safety.

## DB-PLUGIN-089

Plugin update no publicará mixed runtime generation.

## DB-PLUGIN-090

Uninstall deberá considerar dependientes.

## DB-PLUGIN-091

Plugin fingerprint será determinista para misma estructura.

## DB-PLUGIN-092

Runtime fingerprint podrá incorporar plugin generation.

## DB-PLUGIN-093

Cache entries sensibles a plugins incluirán generación/fingerprint cuando corresponda.

## DB-PLUGIN-094

Plugin discovery cache será invalidable.

## DB-PLUGIN-095

Composer continuará siendo package manager; Database no creará otro innecesariamente.

## DB-PLUGIN-096

Database podrá añadir catálogo futuro sin cambiar Plugin Architecture.

## DB-PLUGIN-097

Plugins privilegiados serán identificables.

## DB-PLUGIN-098

Plugin event listeners no tendrán autoridad implícita sobre core semantics.

## DB-PLUGIN-099

Plugin architecture preservará architectural invariants.

## DB-PLUGIN-100

Plugin architecture preservará seguridad, aislamiento y determinismo.

---

# 219. Anti-patrones

## 219.1 Package instalado = plugin activo

Incorrecto.

---

## 219.2 Composer package = PluginId

Incorrecto.

---

## 219.3 Plugin = Extension

Incorrecto.

---

## 219.4 Ejecutar todo el plugin para descubrir metadata

Debe evitarse.

---

## 219.5 Último plugin registrado gana

Incorrecto.

---

## 219.6 Modificar registry durante request

Incorrecto.

---

## 219.7 Plugin accediendo al container completo

Debe evitarse como API normal.

---

## 219.8 Guardar tenant actual en singleton/plugin global

Incorrecto.

---

## 219.9 Deshabilitar plugin borrando handlers de un worker vivo

Incorrecto.

---

## 219.10 `composer remove` eliminando tablas automáticamente

Incorrecto.

---

## 219.11 Actualizar código sin analizar schema compatibility

Incorrecto.

---

## 219.12 Plugin instalado = capability disponible

Incorrecto.

---

## 219.13 Plugin oficial = acceso total

Incorrecto.

---

## 219.14 Plugin permission metadata = sandbox

Incorrecto.

---

## 219.15 Usar `APP_ENV=test` como única protección destructiva

Incorrecto.

---

## 219.16 Atrapar todos los errores del plugin y continuar

Incorrecto.

---

## 219.17 Lazy plugin = no validar hasta primer query

Incorrecto para estructura.

---

## 219.18 Usar prioridad para ocultar dependencias

Incorrecto.

---

## 219.19 Permitir ciclos por orden accidental

Incorrecto.

---

## 219.20 Plugin rollback = DB rollback

Incorrecto.

---

# 220. Modelo formal

Sea:

```text
P
```

un plugin.

Su estado activo deberá cumplir:

```text
Active(P)
=
Installed(P)
∧
Discovered(P)
∧
Enabled(P)
∧
Compatible(P)
∧
DependenciesSatisfied(P)
∧
ConfigurationValid(P)
∧
PermissionsAccepted(P)
∧
BootstrapSucceeded(P)
```

---

# 221. Dependencias

Sea:

```text
D(P)
```

el conjunto de dependencias obligatorias de `P`.

Entonces:

```text
DependenciesSatisfied(P)
=
∀ d ∈ D(P), Satisfied(d)
```

---

# 222. Activation Safety

Una nueva generación podrá publicarse sólo si:

```text
Publish(G)
=
PluginGraphValid(G)
∧
ExtensionGraphValid(G)
∧
ConfigurationValid(G)
∧
NoCriticalConflict(G)
```

---

# 223. Disable Safety

Para deshabilitar `P`:

```text
CanDisable(P)
=
NoActiveRequiredDependents(P)
∨
DependentsIncludedInDisablePlan(P)
```

---

# 224. Uninstall Safety

Conceptualmente:

```text
CanUninstall(P)
=
DependenciesResolved
∧
DataPolicyResolved
∧
SchemaPolicyResolved
∧
RuntimeNotUsingTargetGeneration
∧
OperationAuthorized
```

---

# 225. Runtime Isolation

Para dos requests:

```text
R1
R2
```

deberá cumplirse:

```text
ScopedPluginState(R1)
∩
ScopedPluginState(R2)
=
∅
```

salvo estructuras explícitamente inmutables/compartibles.

---

# 226. Arquitectura consolidada

```text
                     Package Ecosystem
                           │
                           ▼
                  Composer / Local Packages
                           │
                           ▼
                    Package Discovery
                           │
                           ▼
                     Plugin Manifests
                           │
                           ▼
                     Plugin Registry
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
     Enablement       Dependencies     Compatibility
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                  Configuration Validation
                           │
                           ▼
                   Permission Validation
                           │
                           ▼
                     Plugin Resolver
                           │
                           ▼
                  CompiledPluginGraph
                           │
                           ▼
                    Plugin Bootstrap
                           │
                           ▼
                    Extension Set
                           │
                           ▼
                Database Extension System
                           │
                           ▼
                CompiledExtensionGraph
                           │
                           ▼
                         Freeze
                           │
                           ▼
                    Runtime Generation
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Request       Transaction    Connection
           Scope           Scope         Scope
```

---

# 227. Flujo de instalación

La instalación física del package será responsabilidad principal del package manager:

```text
composer require acme/voltstack-spatial
```

Después:

```text
Package Installed
       ↓
Plugin Discovered
       ↓
Manifest Parsed
       ↓
Plugin Registered
       ↓
Enablement Policy
       ↓
Compatibility/Dependencies
       ↓
Plugin Activated
       ↓
Extensions Registered
```

---

# 228. Flujo de desactivación

```text
Disable Request
      ↓
Dependency Preflight
      ↓
Configuration Update
      ↓
Build New Plugin Graph
      ↓
Build New Extension Graph
      ↓
Validate
      ↓
Publish New Runtime Generation
      ↓
Reload Worker if Required
```

---

# 229. Flujo de actualización

```text
Current Plugin
      ↓
Target Package
      ↓
Manifest Comparison
      ↓
Compatibility Preflight
      ↓
Dependency Preflight
      ↓
Config Migration
      ↓
Schema/Data Migration Plan
      ↓
Install Target
      ↓
Compile New Graph
      ↓
Validate
      ↓
Publish
```

---

# 230. Flujo de eliminación

```text
Uninstall Request
      ↓
Dependency Analysis
      ↓
Schema/Data Ownership Analysis
      ↓
Retention Decision
      ↓
Disable Plugin
      ↓
Migrate/Cleanup if Explicitly Requested
      ↓
Remove Package
      ↓
Invalidate Structural Caches
      ↓
Rebuild Runtime
```

---

# 231. Estrategia V1

Para V1, el Plugin System deberá priorizar:

```text
PluginId
Manifest
Composer discovery
Enable/Disable configuration
Dependency resolution
Compatibility validation
Configuration validation
Extension registration
CompiledPluginGraph
Frozen runtime
Diagnostics
Persistent runtime safety
```

No será necesario implementar inicialmente:

```text
online marketplace
automatic remote installation
live hot-swapping
automatic destructive uninstall
complex permission enforcement sandbox
```

---

# 232. Evolución V2+

Posteriormente podrán incorporarse:

```text
official plugin catalog
signed manifests
verified publishers
update compatibility tooling
configuration migration
plugin-specific schema migration orchestration
deployment integration
richer permission governance
plugin health
```

---

# 233. Regla final

> **El Database Plugin System será la capa de distribución y administración de la extensibilidad de VoltStack Database. Un plugin podrá agrupar y entregar extensiones, pero no podrá saltarse los contratos de extensión, modificar silenciosamente un runtime congelado ni utilizar su condición de paquete instalado como prueba de compatibilidad, capacidad o seguridad.**

Por tanto:

```text
Plugin
≠
Extension
```

```text
Plugin
≠
Package
```

```text
Plugin
≠
Service Provider
```

```text
Installed
≠
Enabled
```

```text
Enabled
≠
Active
```

```text
Discovered
≠
Enabled
```

```text
Manifest
≠
Plugin Instance
```

```text
Plugin Dependency
≠
Package Dependency
```

```text
Plugin Configuration
≠
Runtime Context
```

```text
Plugin Registrar
≠
Service Container
```

```text
Trust
≠
Permission
```

```text
Trust
≠
Correctness
```

```text
Permission Metadata
≠
Sandbox
```

```text
Disable
≠
Uninstall
```

```text
Code Removal
≠
Data Removal
```

```text
Plugin Update
≠
Simple File Replacement
```

```text
Plugin Rollback
≠
Database Rollback
```

```text
Plugin Installed
≠
Capability Available
```

```text
Plugin Graph
≠
Extension Graph
```

```text
Lazy Construction
≠
Lazy Structural Validation
```

y finalmente:

```text
Safe Database Plugin System
=
Explicit Identity
+
Declarative Manifest
+
Package Provenance
+
Deterministic Discovery
+
Explicit Enablement
+
Dependency Resolution
+
Compatibility Validation
+
Typed Configuration
+
Bounded Permissions
+
Extension Architecture
+
Atomic Structural Activation
+
Immutable Runtime Generation
+
Scoped State Isolation
+
Diagnostics
+
Safe Updates
+
Safe Uninstallation
+
Persistent Runtime Safety
```

---

# 234. Siguiente documento

```text
296_DATABASE_CUSTOM_DRIVER_SYSTEM.md
```

El siguiente documento deberá definir cómo desarrolladores y paquetes externos podrán implementar nuevos drivers para Database sin acoplar el Query Engine, ORM o Schema directamente al protocolo de un proveedor.

La arquitectura deberá cubrir:

```text
Custom Driver System
│
├── Driver Identity
├── Driver Contract
├── Driver Descriptor
├── Driver Factory
├── Driver Registration
├── Driver Capabilities
├── Driver Configuration
├── Connection Factory
├── Physical Connection
├── Statement Preparation
├── Parameter Binding
├── Statement Execution
├── Result/Cursor Integration
├── Transaction Operations
├── Error Classification
├── Cancellation
├── Connection Reset
├── Persistent Runtime Safety
├── Driver Conformance
├── Driver Diagnostics
└── Driver Plugin Integration
```

manteniendo como principio central:

> **Un custom driver de VoltStack Database implementará el transporte y protocolo necesarios para comunicarse con un sistema de base de datos mediante los contratos del Driver Layer; no deberá convertirse en Query Builder, Dialect, SQL Compiler, ORM, Schema Engine ni autoridad de capacidades semánticas que correspondan a otras capas.**