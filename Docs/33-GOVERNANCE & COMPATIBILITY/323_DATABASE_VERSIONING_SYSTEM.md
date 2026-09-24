# 323_DATABASE_VERSIONING_SYSTEM.md

# VoltStack Quantum Database
## Database Versioning System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 323 — Database Versioning System  
**Bloque:** 33 — Governance and Compatibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md`  
**Siguiente documento:** `324_DATABASE_DEPRECATION_POLICY.md`

---

# 1. Propósito

Este documento define el sistema oficial de versionado para:

```text
VoltStack/Quantum/Database
```

El objetivo no es únicamente asignar números como:

```text
1.0.0
1.1.0
2.0.0
```

sino establecer un modelo que permita determinar:

- qué significa cada versión;
- qué contratos son compatibles;
- cuándo puede introducirse un cambio;
- cuándo se requiere una versión major;
- cómo evolucionan drivers y extensiones;
- cómo se versionan formatos persistentes;
- cómo se coordinan paquetes oficiales;
- cómo funcionan pre-releases;
- cómo se gestionan versiones LTS;
- cómo se publican security releases;
- cómo se realizan rolling upgrades;
- cómo se gestionan downgrades;
- cómo se negocian versiones de contratos;
- cómo se detectan incompatibilidades durante bootstrap;
- cómo se representa la compatibilidad de forma machine-readable.

La regla central será:

> **La versión del paquete indica la evolución del producto; las versiones de contrato indican la evolución de sus fronteras técnicas. Ambas están relacionadas, pero no son la misma cosa.**

Por tanto:

```text
Framework Version
≠
Database Package Version
≠
Public API Contract Version
≠
Driver Contract Version
≠
Extension Contract Version
≠
Persistent Format Version
```

---

# 2. Relación con backward compatibility

El documento:

```text
322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md
```

estableció qué superficies requieren protección.

Este documento determina:

```text
cómo representar esa evolución mediante versiones.
```

Relación:

```text
Backward Compatibility
        │
        ▼
Versioning Rules
        │
        ▼
Release Classification
        │
        ▼
Compatibility Matrix
        │
        ▼
Upgrade / Downgrade Strategy
```

---

# 3. Problema fundamental

Un sistema Database posee múltiples superficies que evolucionan a ritmos diferentes.

Ejemplo:

```text
VoltStack Database 3.4.1
```

podría utilizar:

```text
Public API Contract ........ 3
Driver Contract ............ 2
Extension Contract ......... 4
Metadata Format ............ 7
Cursor Format .............. 2
Migration Repository ....... 3
Backup Manifest Format ..... 2
```

Intentar utilizar:

```text
3.4.1
```

como versión de todos estos contratos produciría acoplamiento artificial.

---

# 4. Principio de versiones independientes

VoltStack adoptará:

```text
Independent Contract Versioning
```

cuando un contrato tenga ciclo de evolución independiente.

Ejemplo:

```text
VoltStack Database 5.2
```

no implica necesariamente:

```text
Driver Contract 5
```

El Driver Contract podría continuar siendo:

```text
Driver Contract 2
```

durante varias versiones del framework.

---

# 5. Taxonomía de versiones

VoltStack Database distinguirá:

```text
Versioning System
├── Product Version
├── Package Version
├── Public API Contract Version
├── Driver Contract Version
├── Dialect Contract Version
├── Extension Contract Version
├── Query Extension Contract Version
├── ORM Extension Contract Version
├── Metadata Format Version
├── Cache Format Generation
├── Migration Repository Version
├── Cursor Format Version
├── Backup Manifest Version
├── CLI Machine Schema Version
├── Event Schema Version
├── Telemetry Schema Version
└── Capability Generation
```

---

# 6. Product version

Representa la versión general del framework o distribución.

Ejemplo:

```text
VoltStack 2.4.0
```

---

# 7. Database package version

Representa:

```text
VoltStack/Quantum/Database
```

Ejemplo:

```text
voltstack/database 2.4.0
```

La estrategia inicial recomendada será mantener sincronía razonable con VoltStack cuando Database forme parte de la distribución oficial.

---

# 8. Package synchronization

Podrán existir dos estrategias.

### Estrategia A

```text
VoltStack 2.4
Database 2.4
```

### Estrategia B

```text
VoltStack 2.4
Database 4.7
```

La arquitectura deberá soportar ambas.

---

# 9. Recomendación inicial

Para las primeras versiones oficiales:

```text
VoltStack Framework Version
≈
Core Quantum Package Version
```

simplifica:

- documentación;
- soporte;
- dependency resolution;
- releases;
- debugging.

Pero no deberá convertirse en restricción arquitectónica permanente.

---

# 10. Semantic Versioning

El paquete utilizará conceptualmente:

```text
MAJOR.MINOR.PATCH
```

Ejemplo:

```text
3.7.2
```

donde:

```text
3 = MAJOR
7 = MINOR
2 = PATCH
```

---

# 11. MAJOR

Una versión MAJOR permite cambios incompatibles intencionales en contratos estables.

Ejemplo:

```text
2.x
↓
3.0
```

---

# 12. MINOR

Una versión MINOR podrá introducir:

```text
new functionality
new optional APIs
new capabilities
new platform support
new extension points
deprecations
compatible optimizations
```

manteniendo los contratos estables existentes.

---

# 13. PATCH

Una versión PATCH deberá concentrarse en:

```text
bug fixes
security fixes
correctness fixes
compatible performance fixes
documentation corrections
```

---

# 14. SemVer is necessary but insufficient

Semantic Versioning por sí solo no resuelve:

```text
driver compatibility
metadata formats
rolling deployments
DBMS support
extension compatibility
migration repository formats
```

Por ello VoltStack agregará:

```text
Contract Versioning
+
Format Versioning
+
Compatibility Metadata
```

---

# 15. MAJOR ≠ rewrite

Una versión major no significa:

```text
rewrite everything
```

Significa que:

```text
one or more stable compatibility boundaries
```

pueden cambiar.

---

# 16. MAJOR discipline

Incluso durante una major:

```text
breaking changes
```

deberán ser:

- justificados;
- documentados;
- agrupados;
- migrables cuando sea posible;
- cubiertos por upgrade guides.

---

# 17. MINOR discipline

Una minor no podrá introducir intencionalmente cambios incompatibles a APIs `STABLE`.

---

# 18. PATCH discipline

Una patch deberá poseer la superficie de cambio más pequeña posible.

---

# 19. Security exception

Un security release podrá modificar comportamiento aunque implique incompatibilidad limitada.

Ejemplo:

```text
2.4.5
```

podría deshabilitar un comportamiento vulnerable.

Deberá clasificarse:

```text
SECURITY_REQUIRED
```

---

# 20. Data integrity exception

Una corrección necesaria para impedir corrupción de datos podrá seguir una política similar.

---

# 21. Release classification

Toda release deberá clasificarse como:

```text
STANDARD
SECURITY
HOTFIX
LTS
PRE_RELEASE
EXPERIMENTAL
```

---

# 22. Release channels

VoltStack podrá utilizar:

```text
stable
preview
beta
alpha
dev
```

---

# 23. Stable channel

Ejemplo:

```text
3.2.0
```

Debe cumplir todos los release gates.

---

# 24. Release candidate

Ejemplo:

```text
3.2.0-rc.1
```

Destinado a:

```text
final compatibility validation
integration testing
ecosystem testing
```

---

# 25. Beta

Ejemplo:

```text
3.2.0-beta.2
```

Las funcionalidades están razonablemente definidas, pero pueden existir ajustes.

---

# 26. Alpha

Ejemplo:

```text
3.2.0-alpha.1
```

Puede contener APIs todavía inestables.

---

# 27. Dev

Ejemplo:

```text
3.3.x-dev
```

No deberá considerarse release estable.

---

# 28. Pre-release compatibility

Una API introducida exclusivamente en:

```text
alpha
beta
RC
```

podrá evolucionar con mayor libertad antes de la versión estable.

---

# 29. RC discipline

Después de `RC1`, cambios de API deberán ser excepcionales.

Idealmente:

```text
RC1
↓
Bug Fixes
↓
RC2
↓
Stable
```

---

# 30. Version object

VoltStack deberá poseer una representación tipada:

```php
final readonly class Version
{
    public function __construct(
        public int $major,
        public int $minor,
        public int $patch,
        public ?PreRelease $preRelease = null,
        public ?string $buildMetadata = null,
    ) {}
}
```

---

# 31. Version comparison

Nunca comparar versiones como strings arbitrarios.

Evitar:

```php
if ($version > '2.10') {
}
```

Preferir:

```php
if ($version->isAtLeast(Version::parse('2.10.0'))) {
}
```

---

# 32. VersionRange

Se requiere:

```text
VersionRange
```

para expresar:

```text
>=2.0 <4.0
```

---

# 33. ContractVersion

Los contratos independientes utilizarán una abstracción distinta:

```php
final readonly class ContractVersion
{
    public function __construct(
        public int $major,
        public int $minor = 0,
    ) {}
}
```

---

# 34. Contract major

Cambio incompatible:

```text
Driver Contract 2
→
Driver Contract 3
```

---

# 35. Contract minor

Podrá representar una extensión compatible:

```text
2.0
→
2.1
```

cuando resulte útil.

---

# 36. Simplicidad inicial

Para V1 podrá utilizarse únicamente:

```text
integer contract version
```

si no existe necesidad real de minor versions.

Ejemplo:

```text
DriverContractVersion = 1
```

---

# 37. Public API Contract Version

Representa la generación de la API pública estable de Database.

Ejemplo:

```text
Database Public API Contract: 2
```

---

# 38. Public API contract ≠ package major

Es posible:

```text
Database 3.0
Public API Contract 2
```

si la major se debió a otra superficie.

---

# 39. Driver Contract Version

Representa el contrato requerido para drivers.

Ejemplo:

```text
Driver Contract v3
```

---

# 40. Driver declaration

Un driver podrá declarar:

```php
final class PostgreSqlDriver implements Driver
{
    public function contractVersion(): DriverContractVersion
    {
        return new DriverContractVersion(3);
    }
}
```

---

# 41. Driver supported range

Más útil aún:

```php
public function supportedContractRange(): ContractVersionRange
{
    return ContractVersionRange::between(2, 3);
}
```

---

# 42. Framework required driver contract

Database declarará:

```text
required driver contract
```

o rango aceptado.

---

# 43. Negotiation

Durante bootstrap:

```text
Database
   │
   ├── requires Driver Contract 3
   │
   ▼
Driver
   │
   └── supports 2–3
         │
         ▼
      Compatible
```

---

# 44. Incompatible negotiation

```text
Database requires 3
Driver supports 1–2

→ IncompatibleDriverContractException
```

---

# 45. Fail early

Esta validación deberá ocurrir:

```text
before application traffic
```

cuando sea posible.

---

# 46. Dialect Contract Version

Custom dialects podrán tener:

```text
Dialect Contract Version
```

separado del driver.

---

# 47. Why separate driver and dialect

Porque:

```text
Driver
```

gestiona protocolo/conectividad.

Mientras:

```text
Dialect
```

describe representación SQL/plataforma.

Por tanto:

```text
Driver Contract
≠
Dialect Contract
```

---

# 48. Compiler Contract Version

Si se permite reemplazar/extender compiladores:

```text
Compiler Extension Contract
```

deberá estar versionado.

---

# 49. Query Extension Contract

Los plugins que agreguen:

```text
AST nodes
operators
functions
query features
```

deberán declarar el contrato requerido.

---

# 50. ORM Extension Contract

Aplicará a:

```text
mapping extensions
custom persistence behaviors
metadata extensions
hydration extensions
relationship extensions
```

cuando sean superficies oficialmente soportadas.

---

# 51. General Extension Contract

Además podrá existir:

```text
Database Extension Contract
```

para plugins generales.

---

# 52. Extension manifest

Ejemplo conceptual:

```json
{
  "name": "vendor/database-extension",
  "voltstack": {
    "database": {
      "package": "^3.0",
      "extension_contract": "^2",
      "driver_contract": null
    }
  }
}
```

---

# 53. Package compatibility

Composer continuará gestionando:

```text
package dependency resolution
```

pero Database realizará validación semántica adicional cuando corresponda.

---

# 54. Composer compatibility ≠ runtime compatibility

```text
Composer Installed Successfully
≠
Database Contracts Compatible
```

---

# 55. Persistent format versions

Los formatos persistentes deberán utilizar:

```text
FormatVersion
```

independiente.

---

# 56. Metadata Format Version

Ejemplo:

```text
Metadata Format v5
```

---

# 57. Metadata format change

Si cambia:

```text
v5
→
v6
```

el sistema podrá:

```text
read v5
recompile to v6
```

o simplemente:

```text
invalidate v5
recompile
```

si el metadata es derivado.

---

# 58. Derived data principle

Si un formato es completamente derivable:

```text
invalidate + rebuild
```

es preferible a mantener readers históricos indefinidamente.

---

# 59. Cache generation

Los caches podrán usar:

```text
CacheGeneration
```

en lugar de migración.

Ejemplo:

```text
database:metadata:g7
```

---

# 60. Generation ≠ version

Una generación indica:

```text
compatibility namespace
```

no necesariamente una release.

---

# 61. Query compilation generation

Ejemplo:

```text
CompiledQueryGeneration = 4
```

---

# 62. Capability generation

Cambios en evidencia/cálculo de capabilities podrán requerir:

```text
CapabilityGeneration
```

para evitar reutilizar decisiones antiguas.

---

# 63. Migration Repository Version

El repository que almacena historial de migraciones es persistente.

Deberá tener:

```text
MigrationRepositoryVersion
```

---

# 64. Repository upgrade

Ejemplo:

```text
Repository v1
   ↓
Internal Upgrade
   ↓
Repository v2
```

---

# 65. Migration repository safety

Nunca actualizar automáticamente un formato crítico sin:

```text
preflight
backup/rollback strategy
transaction capability analysis
verification
```

cuando corresponda.

---

# 66. Cursor Format Version

Los cursores pueden sobrevivir entre requests y despliegues.

Por tanto:

```text
Cursor Format v1
Cursor Format v2
```

deberán ser distinguibles.

---

# 67. Cursor envelope

```json
{
  "format": "voltstack.database.cursor",
  "version": 2,
  "payload": "...",
  "signature": "..."
}
```

---

# 68. Unsupported cursor

Un cursor desconocido deberá producir:

```text
UnsupportedCursorVersionException
```

no interpretación heurística.

---

# 69. Backup Manifest Version

Backups administrados por VoltStack deberán registrar:

```text
BackupManifestVersion
```

---

# 70. Restore compatibility

El restore engine deberá declarar:

```text
supported manifest versions
```

---

# 71. Backup version ≠ DBMS version

Un manifest puede incluir:

```text
Manifest Version
DBMS Vendor
DBMS Version
Schema Version
Application Version
```

como dimensiones diferentes.

---

# 72. Event Schema Version

Eventos públicos que crucen procesos podrán requerir:

```text
EventSchemaVersion
```

---

# 73. Internal event

Eventos exclusivamente internos al proceso podrán no necesitar versionado persistente.

---

# 74. Outbox event

Eventos almacenados en outbox sí requieren atención especial porque pueden sobrevivir a deployments.

---

# 75. CLI Machine Schema Version

Si:

```bash
php voltstack database:status --json
```

produce datos consumidos por automatización, el formato deberá tener versión.

---

# 76. Example

```json
{
  "schema_version": 1,
  "database": {
    "status": "healthy"
  }
}
```

---

# 77. Telemetry schema

Métricas/log schemas públicos podrán poseer:

```text
TelemetrySchemaVersion
```

si se ofrecen garantías estables.

---

# 78. Database release manifest

Cada release deberá producir conceptualmente:

```text
DatabaseReleaseManifest
```

---

# 79. Manifest contents

```text
DatabaseReleaseManifest
├── PackageVersion
├── ReleaseChannel
├── PublicApiContract
├── DriverContractRange
├── DialectContractRange
├── ExtensionContractRange
├── MetadataFormat
├── MigrationRepositoryFormat
├── CursorFormat
├── BackupManifestFormat
├── SupportedPHP
├── SupportedDBMS
├── SupportedRuntimes
├── DeprecatedFeatures
├── RemovedFeatures
└── SecurityAdvisories
```

---

# 80. Example manifest

```json
{
  "package": "voltstack/database",
  "version": "3.4.1",
  "channel": "stable",

  "contracts": {
    "public_api": 3,
    "driver": {
      "min": 2,
      "max": 3
    },
    "dialect": 2,
    "extension": 4
  },

  "formats": {
    "metadata": 7,
    "migration_repository": 2,
    "cursor": 2,
    "backup_manifest": 1
  }
}
```

---

# 81. Platform compatibility manifest

Además:

```json
{
  "platforms": {
    "php": ">=8.x",
    "mysql": "...",
    "mariadb": "...",
    "postgresql": "...",
    "sqlite": "..."
  }
}
```

Los valores reales se determinarán en el release correspondiente.

---

# 82. No premature version commitments

Este documento no fija todavía:

```text
PHP minimum version
MySQL minimum version
PostgreSQL minimum version
```

porque deberán definirse conforme al release real.

---

# 83. Supported DBMS matrix

Cada release deberá publicar una matriz.

Ejemplo conceptual:

| Platform | Version | Status |
|---|---|---|
| MySQL | A | Supported |
| MariaDB | B | Supported |
| PostgreSQL | C | Supported |
| SQLite | D | Supported |

---

# 84. Platform status

Valores:

```text
SUPPORTED
DEPRECATED
END_OF_SUPPORT
EXPERIMENTAL
UNTESTED
UNSUPPORTED
```

---

# 85. UNTESTED

Significa:

```text
no official compatibility guarantee
```

aunque técnicamente pudiera funcionar.

---

# 86. DBMS support removal

Eliminar soporte para una versión previamente soportada deberá anunciarse antes cuando sea razonable.

---

# 87. DBMS deprecation

Ejemplo:

```text
Release 3.5
DBMS X version Y → DEPRECATED

Release 4.0
→ END_OF_SUPPORT
```

---

# 88. Capability model remains authoritative

Incluso dentro de una versión soportada:

```text
Version
≠
Capability
```

La detección de features seguirá usando:

```text
DATABASE_DATABASE_CAPABILITY_DISCOVERY_SYSTEM
```

---

# 89. Why version still matters

Version puede determinar:

```text
minimum support
known bugs
protocol behavior
driver compatibility
security status
```

---

# 90. PHP compatibility matrix

Cada release declarará:

```text
Supported PHP Range
```

---

# 91. PHP major/minor removal

Eliminar una versión de PHP previamente soportada será un cambio operacional significativo.

Normalmente deberá ocurrir:

```text
major release
```

o mediante política previamente anunciada.

---

# 92. Runtime compatibility matrix

VoltStack Database deberá publicar soporte para:

```text
FrankenPHP
PHP-FPM
CLI
RoadRunner
OpenSwoole
```

---

# 93. FrankenPHP

Será runtime prioritario de VoltStack.

Por tanto:

```text
FrankenPHP compatibility
```

formará parte de los principales release gates.

---

# 94. Runtime status

Podrá utilizar:

```text
PRIMARY
SUPPORTED
EXPERIMENTAL
DEPRECATED
UNSUPPORTED
```

---

# 95. Runtime adapter version

Adapters independientes podrán tener su propia versión de paquete.

Ejemplo conceptual:

```text
voltstack/database-roadrunner
voltstack/database-openswoole
```

si se separan físicamente.

---

# 96. Runtime Contract Version

Podrá existir:

```text
Database Runtime Integration Contract
```

si adapters externos lo requieren.

---

# 97. Version negotiation architecture

```text
             Database Package
                    │
                    ▼
            Release Manifest
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
     Driver      Extension     Runtime
       │            │            │
       ▼            ▼            ▼
  Contract      Contract      Contract
   Version       Version       Version
       │            │            │
       └────────────┼────────────┘
                    ▼
            Compatibility Check
                    │
             ┌──────┴──────┐
             ▼             ▼
         Compatible    Incompatible
                           │
                           ▼
                      Fail Bootstrap
```

---

# 98. Compatibility negotiation result

Se propone:

```php
enum CompatibilityStatus
{
    case COMPATIBLE;
    case COMPATIBLE_WITH_DEPRECATIONS;
    case COMPATIBLE_WITH_LIMITATIONS;
    case INCOMPATIBLE;
    case UNKNOWN;
}
```

---

# 99. UNKNOWN

Regla:

```text
UNKNOWN ≠ COMPATIBLE
```

Para contratos críticos:

```text
UNKNOWN
```

deberá bloquear bootstrap o requerir override explícito según política.

---

# 100. Compatibility report

```php
final readonly class VersionCompatibilityReport
{
    public function __construct(
        public CompatibilityStatus $status,
        public array $requirements,
        public array $violations,
        public array $warnings,
    ) {}
}
```

---

# 101. Version requirement

```php
final readonly class VersionRequirement
{
    public function __construct(
        public string $component,
        public VersionRange $required,
        public Version $actual,
    ) {}
}
```

---

# 102. Compatibility matrix

VoltStack deberá poder generar:

```text
CompatibilityMatrix
```

---

# 103. Example

```text
Database 3.4
├── PHP ................. supported range
├── MySQL ............... supported range
├── MariaDB ............. supported range
├── PostgreSQL .......... supported range
├── SQLite .............. supported range
├── FrankenPHP .......... supported range
├── Driver Contract ..... 2–3
├── Extension Contract .. 4
└── Metadata Format ..... 7
```

---

# 104. Machine-readable matrix

La matriz deberá estar disponible para tooling.

---

# 105. Human-readable matrix

También deberá publicarse en documentación.

---

# 106. LTS

VoltStack podrá definir releases:

```text
Long-Term Support
```

---

# 107. LTS goals

Una versión LTS priorizará:

```text
stability
security
critical fixes
longer maintenance
predictable compatibility
```

---

# 108. LTS ≠ frozen

Una LTS podrá recibir:

```text
security fixes
critical correctness fixes
selected compatible fixes
```

---

# 109. LTS feature policy

Nuevas features importantes normalmente deberán ir a releases normales, no backportarse a LTS salvo política explícita.

---

# 110. Support window

Cada release family podrá definir:

```text
ACTIVE_SUPPORT
SECURITY_SUPPORT
END_OF_LIFE
```

---

# 111. Example lifecycle

```text
Release
   │
   ▼
Active Support
   │
   ▼
Security Support
   │
   ▼
End of Life
```

---

# 112. End of Life

Después de EOL:

```text
no official fixes guaranteed
```

---

# 113. Security release

Podrá publicarse:

```text
3.4.3
```

como security patch.

---

# 114. Security advisory linkage

El release manifest podrá referenciar:

```text
security advisory IDs
affected versions
fixed versions
```

---

# 115. Hotfix release

Un hotfix deberá ser:

```text
small
targeted
high-confidence
```

---

# 116. Backporting

Un fix podrá ser aplicado a varias branches:

```text
main
3.x
2.x-LTS
```

si las familias siguen soportadas.

---

# 117. Backport semantic equivalence

El código del fix puede diferir.

Lo importante:

```text
security/correctness property
```

deberá ser equivalente.

---

# 118. Release branches

Ejemplo conceptual:

```text
main
├── 4.x development
│
├── 3.x stable
│
└── 2.x LTS
```

---

# 119. Release candidate gate

Antes de RC:

```text
API freeze
feature freeze
migration compatibility
driver matrix
platform matrix
runtime matrix
```

deberán estar suficientemente estables.

---

# 120. Stable release gate

Antes de stable:

```text
Compatibility Gate
Security Gate
Test Gate
Performance Gate
Migration Gate
Documentation Gate
```

---

# 121. Release gate result

```text
PASS
PASS_WITH_APPROVED_EXCEPTION
FAIL
```

---

# 122. Package version constraints

Paquetes oficiales deberán declarar ranges.

Ejemplo:

```text
voltstack/authentication
requires database ^3.2
```

cuando corresponda.

---

# 123. Avoid exact pinning

Evitar innecesariamente:

```text
database = 3.2.4
```

si:

```text
^3.2
```

es compatible.

---

# 124. Avoid overly broad constraints

También evitar:

```text
database >=1
```

sin límites razonables.

---

# 125. Official package matrix

VoltStack podrá publicar:

| VoltStack | Database | Auth | Authorization | Telemetry |
|---|---|---|---|---|
| 3.x | 3.x | compatible | compatible | compatible |

La tabla real será generada por releases.

---

# 126. Package contract vs package version

Una integración deberá preferir depender de:

```text
stable contract
```

aunque Composer dependa del package version.

---

# 127. Versioned service provider

Un plugin podrá declarar durante bootstrap:

```php
public function databaseExtensionMetadata(): DatabaseExtensionMetadata
{
    return new DatabaseExtensionMetadata(
        extensionContract: 2,
    );
}
```

---

# 128. Schema versioning

Debe distinguirse:

```text
Database Engine Version
```

de:

```text
Application Schema Version
```

---

# 129. Application Schema Version

El historial de migraciones determina:

```text
application database state
```

no la versión del framework.

---

# 130. Framework upgrade ≠ schema version upgrade

Actualizar:

```text
VoltStack Database 3 → 4
```

no significa automáticamente:

```text
Application Schema v3 → v4
```

---

# 131. Internal framework schema

Si VoltStack mantiene tablas internas, éstas deberán tener:

```text
InternalSchemaVersion
```

cuando sea necesario.

Ejemplos potenciales:

```text
migration repository
outbox
framework metadata
```

---

# 132. Internal schema migrations

Deberán distinguirse de:

```text
application migrations
```

---

# 133. Naming

Ejemplo:

```text
FrameworkDatabaseMigration
ApplicationDatabaseMigration
```

---

# 134. Automatic internal upgrade

Sólo podrá ocurrir cuando:

```text
safe
authorized
recoverable
```

---

# 135. Production policy

En producción podría requerirse:

```text
explicit upgrade command
```

en lugar de modificación automática durante request bootstrap.

---

# 136. Version drift

El sistema deberá detectar:

```text
Code Version
≠
Expected Internal Schema Version
```

---

# 137. Drift result

Podrá producir:

```text
UP_TO_DATE
UPGRADE_REQUIRED
DOWNGRADE_DETECTED
INCOMPATIBLE
UNKNOWN
```

---

# 138. Upgrade

Definición:

```text
older supported state
→
newer supported state
```

---

# 139. Downgrade

Definición:

```text
newer state
→
older software
```

---

# 140. Downgrade ≠ inverse upgrade

Una actualización puede realizar transformaciones irreversibles.

Por tanto:

```text
Upgrade(A → B)
```

no implica:

```text
Downgrade(B → A)
```

sea posible.

---

# 141. Downgrade policy

Cada release deberá declarar:

```text
SUPPORTED
LIMITED
UNSUPPORTED
```

para downgrade cuando sea relevante.

---

# 142. Default downgrade policy

VoltStack Database no deberá prometer downgrade automático general.

---

# 143. Safe downgrade conditions

Podrá permitirse si:

```text
no irreversible schema changes
shared formats still readable
no unsupported metadata
no data transformation loss
```

---

# 144. Downgrade preflight

Antes de downgrade:

```text
inspect
validate
report
```

---

# 145. No blind downgrade

Nunca:

```text
install old package
hope it works
```

como estrategia oficial.

---

# 146. Rolling upgrades

Persistent runtimes y despliegues distribuidos requieren coexistencia temporal.

---

# 147. Rolling deployment

Ejemplo:

```text
Node A → Database package 3.4
Node B → Database package 3.5
Node C → Database package 3.4
```

---

# 148. Rolling compatibility

Para que esto sea seguro:

```text
Shared Schema
Shared Cache
Shared Queue
Shared Outbox
Shared Metadata
Shared Formats
```

deben ser compatibles.

---

# 149. N/N-1 policy

VoltStack podrá definir para determinadas superficies:

```text
N
+
N-1
```

como ventana operacional.

---

# 150. N/N-1 ≠ universal guarantee

No deberá asumirse para todos los formatos.

Cada recurso compartido deberá declararlo.

---

# 151. Expand/contract migration

Modelo recomendado:

```text
Release N
   │
   ▼
EXPAND
   │
   ▼
N + N+1 coexist
   │
   ▼
MIGRATE
   │
   ▼
SWITCH
   │
   ▼
CONTRACT
```

---

# 152. Schema expand

Agregar estructuras compatibles.

---

# 153. Migrate

Backfill o transformar datos.

---

# 154. Switch

Nueva aplicación comienza a utilizar representación nueva.

---

# 155. Contract

Eliminar representación antigua cuando ninguna versión anterior dependa de ella.

---

# 156. Version-aware migration planner

El Migration Planner podrá conocer:

```text
DeploymentCompatibilityContext
```

---

# 157. DeploymentCompatibilityContext

Conceptualmente:

```php
final readonly class DeploymentCompatibilityContext
{
    public function __construct(
        public Version $currentVersion,
        public Version $targetVersion,
        public bool $rollingDeployment,
        public array $coexistingVersions,
    ) {}
}
```

---

# 158. Zero downtime

Una migration marcada zero-downtime deberá considerar:

```text
old code
new code
schema intermediate state
```

---

# 159. Queue compatibility

Jobs pueden haber sido serializados por una versión anterior.

---

# 160. Job payload

Por ello:

```text
Job Payload Version
```

puede ser necesario fuera del Database core.

Database deberá respetar ese contexto cuando participe en jobs.

---

# 161. Outbox compatibility

Outbox records pueden sobrevivir deployment.

El payload deberá tener versionado apropiado.

---

# 162. Cache compatibility during rolling deploy

Si formato no es compatible:

```text
Version N cache namespace
≠
Version N+1 cache namespace
```

---

# 163. Cache generation switch

Ejemplo:

```text
db:metadata:g5
db:metadata:g6
```

permite coexistencia.

---

# 164. Dual-read strategy

Para formatos persistentes importantes:

```text
read old + new
write new
```

puede utilizarse temporalmente.

---

# 165. Dual-write caution

```text
dual-write
```

es más peligroso y deberá usarse sólo con estrategia de consistencia explícita.

---

# 166. Version negotiation at bootstrap

Secuencia:

```text
Application Bootstrap
        │
        ▼
Load Database Manifest
        │
        ▼
Discover Components
        │
        ├── Drivers
        ├── Dialects
        ├── Extensions
        └── Runtime Adapter
        │
        ▼
Read Contract Versions
        │
        ▼
Compatibility Resolver
        │
        ▼
Compatibility Report
        │
    ┌───┴────┐
    ▼        ▼
 Compatible  Fail
```

---

# 167. Runtime probes

No usar runtime probes para descubrir algo que ya debería estar declarado en package metadata.

---

# 168. Declared vs observed

Separar:

```text
Declared Version
Observed Capability
```

---

# 169. Version ≠ capability

Reiteración crítica:

```text
PostgreSQL 18
```

no demuestra automáticamente:

```text
supports feature X
```

si configuración/extensión/runtime puede modificar disponibilidad.

---

# 170. Capability ≠ version

Igualmente, detectar una capability no sustituye:

```text
supported platform policy
```

---

# 171. Version resolver

Se propone:

```php
interface DatabaseVersionResolver
{
    public function current(): DatabaseVersionContext;
}
```

---

# 172. DatabaseVersionContext

```php
final readonly class DatabaseVersionContext
{
    public function __construct(
        public Version $packageVersion,
        public ContractVersion $publicApiContract,
        public ContractVersionRange $driverContracts,
        public ContractVersionRange $extensionContracts,
        public DatabaseFormatVersions $formats,
    ) {}
}
```

---

# 173. DatabaseFormatVersions

```php
final readonly class DatabaseFormatVersions
{
    public function __construct(
        public int $metadata,
        public int $migrationRepository,
        public int $cursor,
        public int $backupManifest,
    ) {}
}
```

---

# 174. Compatibility resolver

```php
interface DatabaseCompatibilityResolver
{
    public function resolve(
        DatabaseVersionContext $database,
        ComponentVersionMetadata $component,
    ): VersionCompatibilityReport;
}
```

---

# 175. Component metadata

```text
ComponentVersionMetadata
├── ComponentId
├── PackageVersion
├── ContractVersions
├── RequiredContracts
├── SupportedFormats
└── RequiredCapabilities
```

---

# 176. Required capabilities

Una extensión podrá requerir:

```text
Capability Requirement
```

además de versión.

---

# 177. Example

```text
Extension
├── Database Extension Contract >=2
├── Query Extension Contract >=3
└── Capability: JSON_QUERY
```

---

# 178. Version passes, capability fails

Puede ocurrir:

```text
Contract compatible
Platform capability unavailable
```

Resultado:

```text
COMPATIBLE_WITH_LIMITATIONS
```

o feature deshabilitada.

---

# 179. Compatibility is multidimensional

Formalmente:

```text
Compatible
=
VersionCompatible
∧
ContractCompatible
∧
PlatformCompatible
∧
CapabilityCompatible
∧
FormatCompatible
∧
RuntimeCompatible
```

---

# 180. UNKNOWN dimension

Si una dimensión crítica es:

```text
UNKNOWN
```

la compatibilidad global no deberá convertirse automáticamente en `true`.

---

# 181. Release metadata storage

El package deberá incluir metadata accesible sin conexión DB.

---

# 182. No DB requirement

Consultar:

```text
Database package version
Driver contract version
```

no deberá requerir abrir una conexión.

---

# 183. Version constants

Puede existir:

```php
final class DatabaseVersion
{
    public const VERSION = '1.0.0';

    public const PUBLIC_API_CONTRACT = 1;

    public const DRIVER_CONTRACT = 1;

    public const EXTENSION_CONTRACT = 1;
}
```

---

# 184. Generated constants

Idealmente se generan durante build para evitar divergencia manual.

---

# 185. Single source of truth

Debe existir un origen canónico:

```text
Release Manifest
```

del cual puedan derivarse constants/documentation.

---

# 186. No duplicated mutable truth

Evitar mantener manualmente:

```text
composer.json version
VERSION file
PHP constant
manifest
docs
```

con valores independientes.

---

# 187. Build metadata

SemVer permite:

```text
3.2.1+build.42
```

para metadata no utilizada en precedencia.

---

# 188. Commit metadata

Builds internos podrán registrar:

```text
git commit
build id
build timestamp
```

sin convertirlos en semántica de compatibilidad.

---

# 189. Reproducible builds

Idealmente el mismo source/tag deberá producir artefactos equivalentes.

---

# 190. Version provenance

Diagnósticos deberán poder mostrar:

```text
package version
build identifier
commit
runtime
PHP
driver
DBMS
```

sin exponer secretos.

---

# 191. Diagnostics

Ejemplo:

```text
VoltStack Database
Version: 3.4.1
Driver Contract: 3
Extension Contract: 2
Metadata Format: 7

PHP: ...
Runtime: FrankenPHP
Platform: PostgreSQL ...
```

---

# 192. Version CLI

Se propone:

```bash
php voltstack database:version
```

---

# 193. Human output

```text
VoltStack Database 3.4.1

Public API Contract: 3
Driver Contract: 3
Extension Contract: 2
Metadata Format: 7
```

---

# 194. JSON output

```bash
php voltstack database:version --json
```

podrá exponer metadata estructurada.

---

# 195. JSON schema version

Ese output deberá incluir:

```text
schema_version
```

---

# 196. Compatibility CLI

```bash
php voltstack database:compatibility
```

---

# 197. Target version

Podrá aceptar:

```bash
php voltstack database:compatibility --target=4.0
```

---

# 198. Upgrade check

```bash
php voltstack database:upgrade-check --target=4.0
```

---

# 199. Output categories

```text
PASS
WARNING
ACTION_REQUIRED
BLOCKED
UNKNOWN
```

---

# 200. Version lock

Aplicaciones críticas podrán declarar:

```text
DatabaseCompatibilityLock
```

para evitar upgrades no evaluados.

---

# 201. Compatibility lock ≠ package lock

Composer lock controla paquetes.

Compatibility lock podría registrar:

```text
validated contracts
validated DBMS versions
validated runtime
```

---

# 202. Optional feature

No será necesario para aplicaciones simples.

---

# 203. Support policy

Cada release deberá indicar:

```text
Release Date
Active Support Until
Security Support Until
EOL
```

cuando exista calendario formal.

---

# 204. No fabricated support dates

Si no existe calendario oficial:

```text
UNSPECIFIED
```

será preferible a inventar fechas.

---

# 205. Deprecation relationship

El siguiente documento:

```text
324_DATABASE_DEPRECATION_POLICY.md
```

formalizará el ciclo:

```text
STABLE
   ↓
DEPRECATED
   ↓
REMOVED
```

---

# 206. Deprecation target version

Una deprecación podrá declarar:

```text
deprecated_since
planned_removal
```

---

# 207. Planned ≠ guaranteed

La versión de eliminación podrá ser:

```text
planned
```

no una obligación de eliminarla.

---

# 208. Removal cannot be earlier

Salvo seguridad/integridad:

```text
actual removal
```

no deberá ocurrir antes del target anunciado.

---

# 209. Versioning of configuration

El schema de configuración podrá tener:

```text
ConfigurationSchemaVersion
```

si se requiere.

---

# 210. Config migration

Ejemplo:

```text
Config Schema 2
→
Config Schema 3
```

---

# 211. Configuration aliases

Durante transición:

```text
old key
→
new key
```

---

# 212. Configuration version ≠ app version

Una aplicación puede conservar config antigua compatible durante varias releases.

---

# 213. Versioning of generated code

Code generators deberán registrar:

```text
GeneratorVersion
```

cuando sea útil.

---

# 214. Generated artifact header

Ejemplo:

```php
/**
 * Generated by VoltStack Database Code Generator 2.1.
 */
```

---

# 215. Generated code independence

Código generado debería depender de APIs públicas, no internals del generator.

---

# 216. Versioning of compiled artifacts

Compiled metadata deberá incluir:

```text
compiler generation
format version
```

---

# 217. Example header

```text
CompiledMetadata
├── FormatVersion: 5
├── CompilerGeneration: 8
├── FrameworkVersion: 3.4.1
└── MetadataFingerprint: ...
```

---

# 218. Compatibility check

Si:

```text
FormatVersion
```

es incompatible:

```text
invalidate/recompile
```

---

# 219. Compiler generation mismatch

También podrá forzar recompile aunque el formato estructural no haya cambiado.

---

# 220. Query cache versioning

Query cache keys deberán considerar las generaciones necesarias.

---

# 221. Result cache versioning

Result cache payloads deberán poseer suficiente contexto para evitar interpretar formatos incompatibles.

---

# 222. Entity cache versioning

Entity cache requiere especial cuidado porque contiene estado canonicalizado.

---

# 223. Cache migration policy

Default:

```text
invalidate
```

sobre:

```text
migrate every cache entry
```

---

# 224. Versioning of locks

Distributed lock names no deberán incluir package version sin necesidad.

De lo contrario:

```text
v1 worker lock
≠
v2 worker lock
```

podría destruir exclusión mutua durante rolling deployment.

---

# 225. Shared coordination identifiers

Identificadores de:

```text
locks
leases
migration mutexes
leader election
```

deben evolucionar deliberadamente.

---

# 226. Critical insight

No todo debe incluir versión.

Versionar incorrectamente un identificador compartido también puede romper compatibilidad.

---

# 227. Versioning of audit formats

Audit records, si se almacenan estructuradamente, deberán preservar legibilidad histórica.

---

# 228. Audit ≠ cache

No deberán invalidarse simplemente por upgrade.

---

# 229. Versioning of diagnostics

Diagnostics internos pueden evolucionar.

Machine-consumed diagnostics requerirán schema version.

---

# 230. Versioning of exceptions

No se asignará una versión a cada exception.

Su estabilidad dependerá del:

```text
Public API Contract
```

---

# 231. Canonical error codes

Podrán evolucionar bajo:

```text
ErrorCodeContract
```

si el ecosistema los consume intensamente.

---

# 232. Avoid over-versioning

Regla:

> **Versionar únicamente fronteras cuya evolución independiente tenga valor real.**

---

# 233. Over-versioning problem

Crear:

```text
ConnectionContractVersion
QueryContractVersion
HydrationContractVersion
IdentityMapContractVersion
...
```

sin necesidad produciría complejidad inmanejable.

---

# 234. Recommended initial contracts

V1 deberá comenzar con:

```text
Package Version
Public API Contract
Driver Contract
Extension Contract
Metadata Format
Migration Repository Format
Cursor Format
Cache Generations
Backup Manifest Format
```

---

# 235. Add versions when justified

Nuevos contratos independientes podrán introducirse cuando aparezca una frontera real de ecosistema.

---

# 236. Versioning registry

Se propone:

```php
interface DatabaseVersionRegistry
{
    public function package(): Version;

    public function publicApi(): ContractVersion;

    public function driver(): ContractVersionRange;

    public function extension(): ContractVersionRange;

    public function formats(): DatabaseFormatVersions;
}
```

---

# 237. Immutable registry

El registry deberá ser:

```text
immutable after bootstrap
```

---

# 238. Runtime mutation forbidden

Una request no podrá cambiar:

```text
Driver Contract Version
```

del proceso.

---

# 239. Version context vs database context

No confundir:

```text
DatabaseVersionContext
```

con:

```text
DatabaseContext
```

El primero describe software/contratos.

El segundo describe:

```text
request
tenant
logical database
transaction
routing
```

---

# 240. Version context sharing

Al ser inmutable podrá compartirse entre requests.

---

# 241. Persistent runtimes

FrankenPHP podrá compartir:

```text
DatabaseVersionRegistry
ReleaseManifest
CompatibilityMatrix
```

entre requests.

---

# 242. Scoped state

No deberá almacenarse allí:

```text
current tenant
current transaction
current connection
current actor
```

---

# 243. OpenSwoole

Version metadata podrá ser global/inmutable.

Mutable request state seguirá coroutine-scoped.

---

# 244. RoadRunner

Mismo principio:

```text
immutable version state shared
mutable request state isolated
```

---

# 245. Release process

Proceso conceptual:

```text
Development
    │
    ▼
Feature Complete
    │
    ▼
Compatibility Analysis
    │
    ▼
Version Classification
    │
    ▼
Release Candidate
    │
    ▼
Release Gates
    │
    ▼
Stable Release
    │
    ▼
Support Lifecycle
```

---

# 246. Version classification engine

Tooling podrá sugerir:

```text
PATCH
MINOR
MAJOR
```

basándose en diffs.

---

# 247. Tooling recommendation ≠ authority

La decisión final deberá considerar semántica.

Un API diff no detecta:

```text
flush now commits transaction
```

si la firma no cambió.

---

# 248. Semantic change review

Todo cambio de comportamiento deberá declarar:

```text
BehaviorChange
```

en PR/release metadata.

---

# 249. Change metadata

Ejemplo:

```yaml
change:
  domain: transaction
  compatibility: behavioral
  required_release: major
```

conceptualmente.

---

# 250. Version release calculator

Futuro tooling:

```text
Changes
   │
   ▼
Compatibility Assessments
   │
   ▼
Required Version Bump
```

---

# 251. Maximum impact wins

Conceptualmente:

```text
PATCH + PATCH + MINOR + PATCH
→ MINOR
```

y:

```text
PATCH + MAJOR + MINOR
→ MAJOR
```

---

# 252. Security override

Security fixes podrán usar release excepcional sin esperar una major.

---

# 253. Security documentation

El manifest deberá indicar:

```text
security_required_behavior_change = true
```

cuando aplique.

---

# 254. Version conflict detection

Bootstrap deberá detectar:

```text
Extension A requires contract 2
Extension B requires contract 3
Database provides only 2
```

---

# 255. Conflict result

```text
DatabaseVersionConflictException
```

con todos los requirements.

---

# 256. No first-match resolution

No resolver:

```text
Extension A first
→ choose v2
→ silently break B
```

---

# 257. Constraint solving

Para ecosistemas complejos podrá existir:

```text
ContractConstraintResolver
```

---

# 258. Resolution model

Encontrar:

```text
intersection(required ranges)
```

---

# 259. Formal example

Si:

```text
A = [2,4]
B = [3,5]
Database = [1,3]
```

entonces:

```text
A ∩ B ∩ Database = {3}
```

Compatible mediante contrato 3.

---

# 260. Empty intersection

Si:

```text
A ∩ B ∩ Database = ∅
```

bootstrap deberá fallar.

---

# 261. No runtime contract switching

No seleccionar Driver Contract 2 para una query y 3 para otra salvo arquitectura explícitamente diseñada para ello.

---

# 262. Contract adapter

Una compatibilidad histórica podrá implementarse mediante:

```text
ContractAdapter
```

---

# 263. Example

```text
Driver Contract v1
      │
      ▼
V1ToV2Adapter
      │
      ▼
Database v2 Contract
```

---

# 264. Adapter conditions

Sólo si:

```text
semantics can be preserved
```

---

# 265. No fake adaptation

Si v2 requiere una garantía imposible en v1:

```text
adapter
```

no deberá afirmar compatibilidad.

---

# 266. Adapter deprecation

Adapters históricos deberán tener lifecycle.

---

# 267. Version migration graph

Para formatos persistentes:

```text
v1 → v2 → v3 → v4
```

podrá representarse como graph.

---

# 268. Direct migrations

Puede existir:

```text
v1 → v4
```

si se implementa explícitamente.

---

# 269. Migration path resolver

```text
FormatMigrationPathResolver
```

encontrará una ruta válida.

---

# 270. No path

Resultado:

```text
UnsupportedUpgradePathException
```

---

# 271. Skipping versions

Actualizar:

```text
1.x
→
4.x
```

podrá requerir:

```text
1 → 2 → 3 → 4
```

para ciertos formatos.

---

# 272. Package jump ≠ format jump

El package puede permitir salto directo mientras internamente migra formatos secuencialmente.

---

# 273. Backup before format upgrade

Para formatos persistentes críticos deberá considerarse:

```text
backup
```

antes de transformación irreversible.

---

# 274. Version checks and security

Nunca confiar en:

```text
user-provided version
```

para activar código privilegiado.

---

# 275. Manifest trust

Release manifests oficiales forman parte del artefacto instalado.

---

# 276. External extension metadata

Metadata de terceros deberá validarse estructuralmente.

---

# 277. Version parser security

El parser deberá:

```text
bound input size
reject malformed versions
avoid arbitrary code evaluation
```

---

# 278. Version strings

Nunca ejecutar:

```text
eval(versionExpression)
```

para ranges.

---

# 279. Version information disclosure

Mostrar package versions puede ayudar a atacantes en ciertos contextos.

Por ello:

```text
public HTTP exposure
```

deberá ser configurable.

---

# 280. CLI/admin diagnostics

Podrán mostrar versiones completas a operadores autorizados.

---

# 281. HTTP headers

No añadir automáticamente:

```text
X-VoltStack-Database-Version
```

a respuestas públicas.

---

# 282. Telemetry

Telemetry podrá registrar:

```text
framework generation
driver family
contract generation
```

cuando tenga baja cardinalidad y sea seguro.

---

# 283. Metrics cardinality

No usar:

```text
build commit
```

como label de alta cardinalidad sin política.

---

# 284. Version events

Eventos de lifecycle podrán incluir:

```text
deployment generation
```

si resulta útil.

---

# 285. Versioning tests

Se requieren:

```text
VersionParserTest
VersionComparisonTest
VersionRangeTest
ContractRangeTest
ManifestValidationTest
CompatibilityResolverTest
UpgradePathTest
DowngradePolicyTest
```

---

# 286. Driver tests

Probar:

```text
older compatible driver
current driver
future unsupported driver
missing version
malformed version
```

---

# 287. Extension tests

Igualmente:

```text
compatible extension
deprecated contract
incompatible extension
conflicting extensions
unknown contract
```

---

# 288. Format tests

Para cada formato persistente:

```text
current read
previous read
unsupported future read
migration
corruption
unknown version
```

---

# 289. Rolling deployment tests

Probar coexistencia:

```text
N writer
N+1 reader

N+1 writer
N reader
```

según política declarada.

---

# 290. Schema compatibility tests

Probar:

```text
old application + expanded schema
new application + expanded schema
```

durante zero-downtime migration.

---

# 291. Persistent runtime tests

FrankenPHP:

```text
Request A
→ version state

Request B
→ same immutable version state
→ no mutable request leakage
```

---

# 292. Release manifest test

CI deberá verificar que manifest y código coincidan.

---

# 293. Documentation test

Podrá comprobarse que:

```text
release manifest
compatibility docs
package metadata
```

no diverjan.

---

# 294. Platform matrix tests

Cada combinación oficialmente soportada deberá tener evidencia suficiente.

---

# 295. Testing reality

No será posible probar infinitas combinaciones.

Por ello:

```text
Supported
```

deberá significar una combinación cubierta por política razonable de testing.

---

# 296. Minimum supported versions

La CI deberá incluir especialmente:

```text
minimum supported version
```

de cada plataforma.

---

# 297. Latest supported versions

También:

```text
latest supported version
```

---

# 298. Intermediate versions

Podrán utilizar:

```text
representative matrix
```

según recursos.

---

# 299. Proposed namespace

```text
VoltStack\Quantum\Database\Versioning
```

---

# 300. Proposed directory structure

```text
src/Quantum/Database/Versioning/
├── Contract/
│   ├── DatabaseVersionRegistry.php
│   ├── DatabaseVersionResolver.php
│   ├── DatabaseCompatibilityResolver.php
│   └── ReleaseManifestProvider.php
│
├── Version/
│   ├── Version.php
│   ├── VersionRange.php
│   ├── PreRelease.php
│   └── VersionParser.php
│
├── ContractVersion/
│   ├── ContractVersion.php
│   ├── ContractVersionRange.php
│   ├── PublicApiContractVersion.php
│   ├── DriverContractVersion.php
│   ├── DialectContractVersion.php
│   └── ExtensionContractVersion.php
│
├── Format/
│   ├── FormatVersion.php
│   ├── DatabaseFormatVersions.php
│   ├── MetadataFormatVersion.php
│   ├── CursorFormatVersion.php
│   ├── MigrationRepositoryVersion.php
│   └── BackupManifestVersion.php
│
├── Generation/
│   ├── CacheGeneration.php
│   ├── CompilerGeneration.php
│   ├── CapabilityGeneration.php
│   └── DeploymentGeneration.php
│
├── Manifest/
│   ├── DatabaseReleaseManifest.php
│   ├── ReleaseManifestLoader.php
│   ├── ReleaseManifestValidator.php
│   └── ComponentVersionMetadata.php
│
├── Compatibility/
│   ├── CompatibilityStatus.php
│   ├── VersionRequirement.php
│   ├── VersionCompatibilityReport.php
│   ├── CompatibilityMatrix.php
│   └── ContractConstraintResolver.php
│
├── Upgrade/
│   ├── UpgradePath.php
│   ├── UpgradePathResolver.php
│   ├── UpgradePreflight.php
│   └── UpgradePlan.php
│
├── Downgrade/
│   ├── DowngradePolicy.php
│   ├── DowngradePreflight.php
│   └── DowngradeReport.php
│
├── Release/
│   ├── ReleaseChannel.php
│   ├── ReleaseClassification.php
│   ├── ReleaseLifecycle.php
│   ├── SupportStatus.php
│   └── ReleaseGate.php
│
├── Runtime/
│   ├── RuntimeCompatibility.php
│   └── DeploymentCompatibilityContext.php
│
├── CLI/
│   ├── DatabaseVersionCommand.php
│   ├── DatabaseCompatibilityCommand.php
│   └── DatabaseUpgradeCheckCommand.php
│
└── Exception/
    ├── VersionException.php
    ├── InvalidVersionException.php
    ├── IncompatibleVersionException.php
    ├── IncompatibleDriverContractException.php
    ├── IncompatibleExtensionContractException.php
    ├── UnsupportedFormatVersionException.php
    └── UnsupportedUpgradePathException.php
```

---

# 301. Dependency direction

```text
Versioning
   │
   ├── may inspect metadata
   ├── may validate components
   └── may report compatibility

Versioning
   X
   └── must not execute queries merely to know package version
```

---

# 302. Versioning ≠ capability discovery

Versioning podrá proporcionar evidencia.

Pero:

```text
Versioning
```

no sustituye:

```text
Capability Discovery
```

---

# 303. Versioning ≠ migration engine

Versioning determina:

```text
what versions exist
what paths are compatible
```

Migration ejecuta:

```text
schema/data transformations
```

---

# 304. Versioning ≠ package manager

Composer resuelve:

```text
package installation
```

VoltStack Versioning resuelve:

```text
Database semantic compatibility
```

---

# 305. Versioning ≠ deployment orchestrator

Puede informar si un rolling deployment es compatible.

No necesariamente despliega servidores.

---

# 306. Versioning invariants

## DB-VER-001

Package Version ≠ Contract Version.

## DB-VER-002

Framework Version ≠ Persistent Format Version.

## DB-VER-003

Version ≠ Capability.

## DB-VER-004

Capability ≠ Supported Platform Policy.

## DB-VER-005

Composer Compatibility ≠ Runtime Compatibility.

## DB-VER-006

Upgrade ≠ Downgrade.

## DB-VER-007

Downgrade ≠ inverse Upgrade.

## DB-VER-008

Generation ≠ Version.

## DB-VER-009

Release Channel ≠ Stability Contract automáticamente.

## DB-VER-010

UNKNOWN ≠ COMPATIBLE.

---

# 307. Semantic versioning invariants

## DB-VER-011

Stable breaking API changes normalmente requieren MAJOR.

## DB-VER-012

Compatible feature additions pueden usar MINOR.

## DB-VER-013

Compatible fixes pueden usar PATCH.

## DB-VER-014

Security fixes pueden requerir excepciones.

## DB-VER-015

Data integrity fixes pueden requerir excepciones.

## DB-VER-016

MAJOR ≠ permission to break everything.

## DB-VER-017

MINOR no deberá romper contratos estables intencionalmente.

## DB-VER-018

PATCH tendrá superficie mínima.

## DB-VER-019

Pre-release puede evolucionar más libremente.

## DB-VER-020

RC deberá aproximarse al contrato final.

---

# 308. Contract invariants

## DB-VER-021

Driver Contract será independiente cuando sea necesario.

## DB-VER-022

Extension Contract será independiente.

## DB-VER-023

Dialect Contract podrá ser independiente.

## DB-VER-024

Public API Contract podrá ser independiente.

## DB-VER-025

Contract ranges serán explícitos.

## DB-VER-026

Contract mismatch fallará temprano.

## DB-VER-027

Missing critical contract metadata no se asumirá compatible.

## DB-VER-028

External components declararán requirements.

## DB-VER-029

Conflicting requirements no se resolverán silenciosamente.

## DB-VER-030

Adapters no fingirán garantías imposibles.

---

# 309. Format invariants

## DB-VER-031

Persistent formats tendrán identidad/version cuando sea necesario.

## DB-VER-032

Unknown format no será interpretado como current.

## DB-VER-033

Derived caches podrán invalidarse.

## DB-VER-034

Migration repository no será tratado como cache.

## DB-VER-035

Backup manifest tendrá reader compatibility explícita.

## DB-VER-036

Cursor format será identificable.

## DB-VER-037

Outbox formats considerarán rolling deployments.

## DB-VER-038

Audit records preservarán legibilidad histórica.

## DB-VER-039

Compiled metadata podrá reconstruirse.

## DB-VER-040

Format migration paths serán explícitos.

---

# 310. Platform invariants

## DB-VER-041

Supported DBMS versions estarán documentadas.

## DB-VER-042

UNTESTED ≠ SUPPORTED.

## DB-VER-043

Deprecated platform ≠ unsupported todavía.

## DB-VER-044

End-of-support será explícito.

## DB-VER-045

PHP support será versionado por release.

## DB-VER-046

Runtime support será documentado.

## DB-VER-047

FrankenPHP será parte principal del release matrix.

## DB-VER-048

RoadRunner support dependerá del adapter oficial.

## DB-VER-049

OpenSwoole support deberá incluir coroutine isolation.

## DB-VER-050

DBMS Version ≠ DBMS Capability.

---

# 311. Release invariants

## DB-VER-051

Cada release estable tendrá manifest.

## DB-VER-052

Manifest será machine-readable.

## DB-VER-053

Manifest tendrá fuente canónica.

## DB-VER-054

Version constants no divergirán del manifest.

## DB-VER-055

Release gates deberán pasar antes de stable.

## DB-VER-056

Security releases identificarán su naturaleza.

## DB-VER-057

LTS tendrá política explícita.

## DB-VER-058

EOL será explícito cuando exista calendario.

## DB-VER-059

No se inventarán support windows.

## DB-VER-060

Backports preservarán la propiedad corregida.

---

# 312. Upgrade invariants

## DB-VER-061

Upgrade tendrá preflight cuando afecte formatos críticos.

## DB-VER-062

Upgrade path podrá ser multi-step.

## DB-VER-063

Skipping package versions no implica skipping format migrations.

## DB-VER-064

Irreversible upgrade será identificado.

## DB-VER-065

Downgrade support será explícito.

## DB-VER-066

Blind downgrade no será estrategia soportada.

## DB-VER-067

Schema compatibility será evaluada en rolling deployments.

## DB-VER-068

Shared format compatibility será evaluada.

## DB-VER-069

Cache namespace podrá cambiar por generation.

## DB-VER-070

Distributed locks no se versionarán arbitrariamente.

---

# 313. Runtime invariants

## DB-VER-071

Version metadata será immutable.

## DB-VER-072

Version metadata podrá compartirse entre requests.

## DB-VER-073

Current tenant no pertenecerá al VersionRegistry.

## DB-VER-074

Current transaction no pertenecerá al VersionRegistry.

## DB-VER-075

Current actor no pertenecerá al VersionRegistry.

## DB-VER-076

Persistent workers no mutarán contract versions por request.

## DB-VER-077

Deployment generation será distinta de request scope.

## DB-VER-078

Mixed generations sólo coexistirán si shared resources son compatibles.

## DB-VER-079

Worker restart podrá ser parte del upgrade plan.

## DB-VER-080

Hot replacement no será asumido seguro.

---

# 314. Tooling invariants

## DB-VER-081

Version parser será determinista.

## DB-VER-082

Version comparison no usará comparación lexical ingenua.

## DB-VER-083

Compatibility CLI no modificará datos por defecto.

## DB-VER-084

Upgrade check será seguro por defecto.

## DB-VER-085

Machine output tendrá schema version.

## DB-VER-086

Human formatting podrá cambiar más libremente.

## DB-VER-087

Compatibility resolver mostrará todas las incompatibilidades relevantes.

## DB-VER-088

No first-match conflict resolution.

## DB-VER-089

Version tooling podrá sugerir release bump.

## DB-VER-090

Tooling no sustituirá semantic review.

---

# 315. Security invariants

## DB-VER-091

Version parser no evaluará código.

## DB-VER-092

Untrusted version input será validado.

## DB-VER-093

Public HTTP version disclosure estará deshabilitado por defecto.

## DB-VER-094

Secrets nunca formarán parte de release manifests.

## DB-VER-095

Compatibility errors no expondrán credenciales.

## DB-VER-096

Security fixes podrán superar accidental BC.

## DB-VER-097

Unsupported insecure platform podrá ser bloqueada.

## DB-VER-098

Manifest integrity deberá ser verificable mediante packaging confiable.

## DB-VER-099

Third-party metadata no será trusted code.

## DB-VER-100

Version negotiation no otorgará privilegios.

---

# 316. Additional invariants

## DB-VER-101

No todo componente requiere versión independiente.

## DB-VER-102

Over-versioning será evitado.

## DB-VER-103

Independent version sólo se introducirá con frontera real.

## DB-VER-104

Stable identifiers podrán sobrevivir package versions.

## DB-VER-105

Package major no obligará a incrementar todos los format versions.

## DB-VER-106

Format version change no obligará siempre a package major.

## DB-VER-107

Capability generation podrá cambiar en minor/patch si preserva contratos.

## DB-VER-108

Schema version pertenece a la aplicación, no al package version.

## DB-VER-109

Internal framework schema será distinguido del application schema.

## DB-VER-110

Version drift será diagnosticable.

---

# 317. Final invariants

## DB-VER-111

A release version describes product evolution.

## DB-VER-112

A contract version describes boundary evolution.

## DB-VER-113

A format version describes representation evolution.

## DB-VER-114

A generation describes compatibility namespace evolution.

## DB-VER-115

A capability describes supported behavior.

## DB-VER-116

These concepts shall not be conflated.

## DB-VER-117

Every stable release shall be reproducibly identifiable.

## DB-VER-118

Every critical component shall be compatibility-checkable.

## DB-VER-119

Every irreversible transition shall be explicit.

## DB-VER-120

Every unsupported state shall fail predictably rather than silently.

---

# 318. Anti-patterns

## Anti-pattern 1

```php
if ($mysqlVersion >= '8.0') {
    $supportsFeature = true;
}
```

sin consultar capability real.

---

## Anti-pattern 2

```text
VoltStack 5
→ therefore Driver Contract 5
```

---

## Anti-pattern 3

```text
Metadata cache incompatible
→ attempt unsafe unserialize anyway
```

---

## Anti-pattern 4

```text
Composer installed plugin
→ plugin must be semantically compatible
```

---

## Anti-pattern 5

```text
Unknown contract version
→ assume latest
```

---

## Anti-pattern 6

```text
Unknown format
→ try current parser
```

---

## Anti-pattern 7

```text
Upgrade succeeded
→ downgrade must work
```

---

## Anti-pattern 8

```text
Major release
→ remove every deprecated API regardless of migration cost
```

---

## Anti-pattern 9

```text
Patch release
→ silently change transaction semantics
```

---

## Anti-pattern 10

```text
Every internal class gets its own version number
```

---

## Anti-pattern 11

```text
Lock name includes app version automatically
```

rompiendo coordinación entre rolling nodes.

---

## Anti-pattern 12

```text
Expose exact VoltStack version in every public HTTP response
```

---

## Anti-pattern 13

```text
Migration repository format
=
cache generation
```

---

## Anti-pattern 14

```text
Database package version
=
application schema version
```

---

## Anti-pattern 15

```text
version_compare()
```

disperso por todo el core como mecanismo arquitectónico.

Las decisiones deberán centralizarse en:

```text
Versioning + Capability System
```

---

# 319. V1

La primera versión deberá implementar:

```text
Semantic Package Version
Release Manifest
Version
VersionRange
ContractVersion
ContractVersionRange
Public API Contract Version
Driver Contract Version
Extension Contract Version
Metadata Format Version
Migration Repository Version
Cursor Format Version
Backup Manifest Version
Cache Generation
Compiler Generation
Capability Generation
Compatibility Resolver
Compatibility Report
DBMS Support Matrix
PHP Support Matrix
Runtime Support Matrix
Version CLI
Compatibility CLI
Upgrade Check
Release Gates
```

---

# 320. V2

Podrá incorporar:

```text
Contract constraint solver
Format migration graph
Rolling deployment analyzer
N/N-1 compatibility analysis
Automatic manifest generation
Automated release bump suggestions
Package ecosystem matrix
Compatibility locks
Internal schema upgrade planner
```

---

# 321. V3

Podrá incorporar:

```text
Cross-version simulation
Automated deployment compatibility analysis
Schema evolution simulation
Multi-package constraint graph
Historical application compatibility laboratory
Automated downgrade feasibility analysis
```

---

# 322. Future AI integration

IA podrá analizar:

```text
API changes
contract diffs
migration history
extension requirements
platform support
```

y recomendar:

```text
PATCH
MINOR
MAJOR
```

o detectar potenciales incompatibilidades.

Pero:

> **La IA no será autoridad final para declarar que un cambio semántico es compatible.**

La decisión deberá apoyarse en:

```text
contracts
tests
architecture
human review
```

---

# 323. Architecture overview

```text
                         VoltStack
                            │
                            ▼
                  Database Package Version
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
 Public Contracts     Component Contracts   Formats
        │                   │                   │
        ├── API             ├── Driver          ├── Metadata
        ├── CLI             ├── Dialect         ├── Migration Repo
        └── Config          ├── Extension       ├── Cursor
                            └── Runtime         └── Backup
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
                   Compatibility Resolver
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Package       Platform       Capability
          Matrix         Matrix          Matrix
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                    Compatibility Report
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
             Compatible             Incompatible
                │                       │
                ▼                       ▼
             Bootstrap              Fail Early
```

---

# 324. Version model summary

La arquitectura completa puede resumirse como:

```text
Product Version
      │
      ▼
Release Lifecycle
      │
      ├── Stable
      ├── LTS
      ├── Security
      └── Pre-release
      │
      ▼
Contract Versions
      │
      ├── Public API
      ├── Driver
      ├── Dialect
      └── Extension
      │
      ▼
Format Versions
      │
      ├── Metadata
      ├── Migration Repository
      ├── Cursor
      └── Backup
      │
      ▼
Generations
      │
      ├── Cache
      ├── Compiler
      └── Capability
      │
      ▼
Compatibility Matrix
      │
      ▼
Upgrade / Downgrade / Rolling Deployment
```

---

# 325. Regla definitiva

> **VoltStack Database utilizará Semantic Versioning para expresar la evolución del paquete, versiones de contrato para expresar la evolución de fronteras técnicas, versiones de formato para representar datos persistentes y generaciones para separar artefactos derivados incompatibles.**

Formalmente:

```text
DatabaseEvolution
=
PackageVersion
+
ContractVersions
+
FormatVersions
+
Generations
+
CapabilityEvidence
+
CompatibilityPolicy
```

No:

```text
DatabaseEvolution
=
One Global Version Number
```

---

# 326. Resultado arquitectónico

Con este modelo VoltStack podrá tener, por ejemplo:

```text
VoltStack Database 4.7.2
```

mientras utiliza:

```text
Public API Contract ......... 4
Driver Contract ............. 3
Dialect Contract ............ 2
Extension Contract .......... 4
Metadata Format ............. 8
Migration Repository ........ 2
Cursor Format ............... 3
Backup Manifest ............. 2
Cache Generation ............ 11
Compiler Generation ......... 9
Capability Generation ....... 6
```

sin confundir estas dimensiones.

Esto permitirá:

```text
predictable upgrades
stable extensions
safe rolling deployments
clear platform support
explicit compatibility
controlled deprecation
rebuildable derived state
versioned persistent state
```

---

# 327. Relación con Backward Compatibility

Los documentos `322` y `323` juntos establecen:

```text
322
What must remain compatible?

323
How is compatibility represented across versions?
```

El siguiente paso será determinar:

```text
How does a stable contract leave the ecosystem safely?
```

Ese problema corresponde a la política de deprecación.

---

# 328. Siguiente documento

```text
324_DATABASE_DEPRECATION_POLICY.md
```

El siguiente documento deberá formalizar el lifecycle completo:

```text
STABLE
   │
   ▼
DEPRECATED
   │
   ▼
REMOVAL_ELIGIBLE
   │
   ▼
REMOVED
```

incluyendo:

```text
deprecation identifiers
deprecated_since
replacement
planned removal
runtime warnings
development diagnostics
test enforcement
configuration deprecations
API deprecations
driver contract deprecations
extension deprecations
event deprecations
CLI deprecations
format deprecations
DBMS version deprecations
runtime deprecations
security exceptions
removal gates
migration guides
codemods
telemetry
LTS behavior
```

y la regla:

> **Una deprecación no es simplemente una advertencia: es un contrato temporal de migración entre una API soportada y su futura eliminación.**