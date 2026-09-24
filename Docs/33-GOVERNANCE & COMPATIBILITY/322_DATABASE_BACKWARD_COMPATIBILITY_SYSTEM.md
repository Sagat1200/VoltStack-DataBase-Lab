# 322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM.md

# VoltStack Quantum Database
## Database Backward Compatibility System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 322 — Database Backward Compatibility System  
**Bloque:** 33 — Governance and Compatibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `321_DATABASE_HTTP_REQUEST_LIFECYCLE_INTEGRATION.md`  
**Siguiente documento:** `323_DATABASE_VERSIONING_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de compatibilidad hacia atrás de:

```text
VoltStack/Quantum/Database
```

Su objetivo es permitir que Database evolucione sin romper arbitrariamente:

- aplicaciones;
- paquetes Quantum;
- entidades;
- modelos;
- repositorios;
- queries;
- migraciones;
- drivers;
- dialectos;
- compiladores;
- plugins;
- extensiones ORM;
- tipos personalizados;
- herramientas CLI;
- metadata;
- caches persistentes;
- formatos serializados;
- runtimes;
- integraciones externas.

La regla fundamental será:

> **La compatibilidad hacia atrás es una propiedad observable de contratos soportados, no simplemente la conservación de nombres de clases y métodos.**

Por tanto:

```text
Same API Signature
≠
Same Semantics
```

y:

```text
Code Still Compiles
≠
Backward Compatible
```

---

# 2. Objetivo arquitectónico

VoltStack deberá permitir:

```text
Database v1
   ↓
Database v1.x
   ↓
Database v2
   ↓
Database vN
```

manteniendo una estrategia explícita para distinguir:

```text
Compatible Evolution
Intentional Deprecation
Breaking Change
Bug Fix
Security Fix
Behavior Correction
Experimental Change
Internal Refactoring
```

---

# 3. Problema fundamental

Un sistema Database posee contratos mucho más amplios que su API PHP.

Por ejemplo, cambiar:

```php
$user = User::find(10);
```

puede parecer compatible si la firma permanece igual.

Pero podría romper aplicaciones si cambia:

```text
null behavior
exception behavior
tenant resolution
read/write routing
IdentityMap semantics
transaction behavior
soft-delete behavior
hydration behavior
```

Por ello:

```text
Backward Compatibility
>
Method Signature Compatibility
```

---

# 4. Dimensiones de compatibilidad

VoltStack reconocerá al menos:

```text
Backward Compatibility
├── Source Compatibility
├── Binary/Runtime Compatibility
├── Public API Compatibility
├── Semantic Compatibility
├── Behavioral Compatibility
├── Data Compatibility
├── Schema Compatibility
├── Migration Compatibility
├── Metadata Compatibility
├── Query Compatibility
├── Driver Compatibility
├── Dialect Compatibility
├── Extension Compatibility
├── Configuration Compatibility
├── Cache Format Compatibility
├── CLI Compatibility
├── Runtime Compatibility
└── Operational Compatibility
```

---

# 5. Source compatibility

Existe cuando código válido para una versión anterior continúa siendo válido.

Ejemplo:

```php
Database::connection('main');
```

continúa siendo aceptado.

Pero:

```text
Source Compatible
≠
Semantically Compatible
```

---

# 6. Semantic compatibility

Existe cuando una operación conserva su significado documentado.

Ejemplo:

```php
User::find(10);
```

debe continuar significando conceptualmente:

```text
Find User
by canonical identifier
within effective DatabaseContext
```

---

# 7. Behavioral compatibility

Incluye comportamiento observable como:

```text
return types
exceptions
transaction boundaries
query execution timing
lazy/eager behavior
identity guarantees
ordering guarantees
null semantics
retry behavior
```

---

# 8. Data compatibility

Datos persistidos por una versión anterior deberán permanecer utilizables según el contrato declarado.

Esto puede incluir:

```text
database rows
migration repository
serialized metadata
cursor tokens
cache payloads
outbox records
fixture references
```

---

# 9. Schema compatibility

La evolución del framework no deberá asumir que los usuarios reconstruirán sus bases de datos desde cero.

Por tanto:

```text
Framework Upgrade
≠
Schema Reset
```

---

# 10. Migration compatibility

Migraciones históricas forman parte del historial de una aplicación.

VoltStack deberá evitar que una actualización convierta migraciones válidas históricas en código inutilizable sin estrategia de transición.

---

# 11. Metadata compatibility

Cambios en:

```text
Entity Metadata
Relationship Metadata
Type Metadata
Schema Metadata
Compiled Metadata
```

deberán poseer versionado cuando se persistan o cacheen.

---

# 12. Extension compatibility

Los contratos oficiales para:

```text
Custom Driver
Custom Dialect
Custom Type
Custom Compiler
Query Extension
ORM Extension
Plugin
```

formarán parte de la política BC cuando estén declarados estables.

---

# 13. Internal implementation

Una clase interna podrá cambiar sin BC guarantee si nunca fue declarada API pública.

Regla:

```text
Accessible
≠
Public API
```

---

# 14. API stability classification

Cada API relevante deberá clasificarse como:

```text
STABLE
DEPRECATED
EXPERIMENTAL
INTERNAL
```

Opcionalmente:

```text
PREVIEW
```

para features candidatas a estabilización.

---

# 15. STABLE

Una API `STABLE` está cubierta por la política de compatibilidad.

Ejemplo conceptual:

```php
#[ApiStatus(ApiStability::STABLE)]
interface ConnectionInterface
{
}
```

---

# 16. INTERNAL

Una API interna podrá cambiar entre releases compatibles.

Ejemplo:

```php
#[Internal]
final class AstNormalizationPass
{
}
```

---

# 17. EXPERIMENTAL

Una API experimental:

```text
may change
may disappear
may be redesigned
```

sin promesa completa de BC.

Sin embargo, los cambios deberán seguir documentándose.

---

# 18. PREVIEW

`PREVIEW` podrá representar:

```text
feature intended for stabilization
```

pero todavía no incluida en el contrato estable.

---

# 19. DEPRECATED

Una API deprecated:

```text
still supported
+
scheduled for future removal
```

Por tanto:

```text
Deprecated
≠
Removed
```

---

# 20. Public API registry

VoltStack deberá mantener un inventario explícito:

```text
DatabasePublicApiRegistry
```

de contratos oficialmente soportados.

---

# 21. Why registry

Sin clasificación explícita ocurre:

```text
everything public becomes accidentally stable
```

especialmente en PHP, donde muchas clases son técnicamente accesibles.

---

# 22. Public API categories

El registry podrá clasificar:

```text
Class
Interface
Trait
Enum
Attribute
Method
Function
Facade
Helper
CLI Command
Configuration Key
Extension Point
Serialized Format
Event
Exception
```

---

# 23. Compatibility contract

Se propone:

```php
interface CompatibilityContract
{
    public function identifier(): CompatibilityContractId;

    public function stability(): ApiStability;

    public function since(): Version;

    public function deprecatedSince(): ?Version;

    public function removalVersion(): ?Version;
}
```

---

# 24. Compatibility domains

Cada contrato deberá pertenecer a un dominio:

```text
PUBLIC_API
ORM
QUERY
SCHEMA
MIGRATION
DRIVER
DIALECT
EXTENSION
CONFIG
CLI
SERIALIZATION
CACHE
RUNTIME
```

---

# 25. BC baseline

Cada release estable deberá generar un:

```text
Compatibility Baseline
```

---

# 26. Baseline contents

Conceptualmente:

```text
CompatibilityBaseline
├── Version
├── PublicApiSnapshot
├── ExtensionContractSnapshot
├── ConfigurationSchema
├── EventSchema
├── ExceptionTaxonomy
├── CLIContract
├── SerializedFormatVersions
└── CapabilityContractSnapshot
```

---

# 27. API comparison

Entre:

```text
v1.4
```

y:

```text
v1.5
```

podrá calcularse:

```text
CompatibilityDiff
```

---

# 28. CompatibilityDiff

Ejemplo:

```text
Removed APIs: 0
Changed signatures: 0
Narrowed parameter types: 0
Changed return contracts: 0
Deprecated APIs: 3
New APIs: 12
Experimental changes: 5
```

---

# 29. Signature compatibility

El sistema analizará cambios como:

```text
class removal
method removal
visibility reduction
parameter addition
parameter removal
parameter type change
return type change
interface method addition
property visibility change
```

---

# 30. PHP variance

Las reglas deberán considerar correctamente:

```text
covariance
contravariance
inheritance
interfaces
```

y no limitarse a comparar strings.

---

# 31. Interface evolution

Agregar un método a una interfaz estable puede romper implementaciones externas.

Ejemplo:

```php
interface Driver
{
    public function connect(): Connection;
}
```

cambiar a:

```php
interface Driver
{
    public function connect(): Connection;

    public function capabilities(): Capabilities;
}
```

puede romper custom drivers.

---

# 32. Preferred evolution

En vez de modificar el contrato original:

```php
interface CapabilityAwareDriver extends Driver
{
    public function capabilities(): Capabilities;
}
```

o utilizar un contrato de capabilities independiente.

---

# 33. Capability-based evolution

La arquitectura de capabilities permite evolucionar sin preguntar:

```php
if ($driver instanceof NewDriverVersion) {
}
```

Preferir:

```php
if ($capabilities->supports($feature)) {
}
```

---

# 34. Capability compatibility

Agregar una capability:

```text
does not automatically mean
all drivers support it
```

Default:

```text
UNKNOWN
```

o:

```text
UNSUPPORTED
```

según semántica del contrato.

---

# 35. UNKNOWN preservation

Nunca convertir automáticamente:

```text
missing new capability
```

en:

```text
SUPPORTED
```

para mantener aparente compatibilidad.

---

# 36. Semantic contracts

Las APIs críticas deberán documentar:

```text
Preconditions
Postconditions
Side Effects
Exceptions
Ordering
Identity
Transaction Behavior
Scope
Failure Semantics
```

---

# 37. Example: EntityManager::persist()

Contrato estable:

```text
persist(entity)
```

significa:

```text
register entity for persistence
```

No:

```text
execute INSERT immediately
```

Cambiar esto sería un cambio semántico incompatible aunque la firma permaneciera igual.

---

# 38. Example: flush()

Contrato:

```text
flush()
```

sincroniza cambios ORM con Database.

No significa:

```text
commit transaction
```

Cambiarlo sería BC break.

---

# 39. Example: IdentityMap

Si VoltStack garantiza:

```text
same entity identity
→
same managed object
```

dentro del mismo scope, eliminar esta propiedad sería un cambio incompatible.

---

# 40. Example: Repository

Cambiar:

```php
$repository->find($id);
```

de:

```text
Entity|null
```

a:

```text
throws NotFound
```

sería un cambio observable.

---

# 41. Exception compatibility

Las excepciones públicas forman parte del contrato.

---

# 42. Exception hierarchy

Preferir jerarquías estables:

```text
DatabaseException
├── ConnectionException
├── QueryException
├── TransactionException
├── PersistenceException
├── ConcurrencyException
└── SchemaException
```

---

# 43. Exception specialization

Agregar una excepción más específica podrá ser compatible si sigue siendo capturable mediante la excepción padre documentada.

---

# 44. Exception removal

Eliminar una excepción estable o cambiarla por una no relacionada podrá ser breaking.

---

# 45. Error codes

Errores programáticamente consumibles deberán preferir:

```text
stable canonical codes
```

sobre mensajes de texto.

Ejemplo:

```text
DB-CONNECTION-TIMEOUT
DB-UNIQUE-CONSTRAINT
DB-OPTIMISTIC-CONFLICT
```

---

# 46. Error message compatibility

Los textos humanos:

```text
should not normally be machine contracts
```

a menos que se declaren explícitamente.

---

# 47. Configuration compatibility

Las claves públicas de configuración también constituyen API.

Ejemplo:

```php
'database.connections.main.driver'
```

---

# 48. Configuration removal

No eliminar una clave estable silenciosamente.

---

# 49. Configuration migration

Si:

```text
old.key
```

cambia a:

```text
new.key
```

se deberá proporcionar:

```text
alias
deprecation warning
migration tool
upgrade diagnostic
```

cuando sea razonable.

---

# 50. Configuration precedence

Cambiar precedencia entre:

```text
defaults
config files
environment
runtime override
```

puede ser un breaking behavioral change.

---

# 51. Default value changes

Cambiar un default puede ser incompatible incluso sin cambiar APIs.

Ejemplo:

```text
lazy_loading = true
```

a:

```text
lazy_loading = false
```

podría romper aplicaciones.

---

# 52. Default behavior governance

Todo cambio de default deberá clasificarse:

```text
SAFE
BEHAVIORAL
SECURITY_REQUIRED
BREAKING
```

---

# 53. Secure defaults

Un cambio de seguridad podrá justificar modificar comportamiento.

Pero deberá:

- documentarse;
- incluir guía de migración;
- explicar riesgo;
- proporcionar compatibilidad temporal cuando sea seguro.

---

# 54. Security > accidental compatibility

VoltStack no mantendrá comportamiento inseguro indefinidamente sólo por BC.

---

# 55. Bug compatibility

Regla:

> **Backward compatibility no significa preservar indefinidamente bugs.**

Por tanto:

```text
Backward Compatible
≠
Bug-for-Bug Compatible
```

---

# 56. Bug fix classification

Un bug fix deberá evaluarse según:

```text
documented behavior
actual widespread behavior
security impact
data integrity impact
application dependency risk
```

---

# 57. Documented contract wins

Cuando comportamiento real contradiga claramente el contrato documentado, corregirlo podrá considerarse bug fix.

Pero si existe amplio uso dependiente del bug, se deberá considerar transición.

---

# 58. Security fix

Una vulnerabilidad crítica podrá requerir un cambio inmediato.

El release deberá marcarlo explícitamente:

```text
SECURITY BEHAVIOR CHANGE
```

---

# 59. Data integrity fix

Un bug que pueda corromper datos tendrá prioridad similar.

---

# 60. Query Builder compatibility

El Query Builder posee múltiples niveles de contrato:

```text
API
AST semantics
binding semantics
ordering semantics
result semantics
```

---

# 61. SQL text ≠ primary contract

VoltStack no garantizará necesariamente que una misma query produzca byte-for-byte el mismo SQL.

Ejemplo:

```text
SELECT * FROM users WHERE id = ?
```

podría convertirse en una forma SQL equivalente.

---

# 62. Semantic SQL compatibility

Lo importante será conservar:

```text
meaning
parameter safety
result semantics
transaction semantics
platform compatibility
```

---

# 63. SQL snapshot tests

Los tests internos de SQL exacto no deberán convertir accidentalmente formatting en API pública.

---

# 64. Query AST compatibility

Si el AST es un extension point público, sus nodos estables deberán seguir política BC.

Si es interno:

```text
AST internals
```

podrán evolucionar.

---

# 65. AST extension API

Se recomienda separar:

```text
Internal AST
```

de:

```text
Public Query Extension Contract
```

---

# 66. Compiler compatibility

Custom compilers no deberán depender de estructuras internas no estables.

---

# 67. Compiler SPI

Se deberá proporcionar:

```text
Compiler Service Provider Interface
```

estable.

---

# 68. SPI vs internal compiler

```text
Public Compiler SPI
≠
Compiler Internal Pipeline
```

---

# 69. Driver compatibility

Los custom drivers deberán implementar un contrato versionado.

---

# 70. Driver contract version

Ejemplo:

```php
interface DriverContractVersion
{
    public function version(): int;
}
```

o mediante metadata:

```text
driver-contract: 1
```

---

# 71. Driver negotiation

Durante bootstrap:

```text
Framework
   ↓
Driver Contract
   ↓
Compatibility Check
```

---

# 72. Unsupported driver

Si un plugin implementa:

```text
Driver Contract v1
```

pero el framework requiere:

```text
v3
```

sin bridge compatible:

```text
fail early
```

---

# 73. No late surprise

No esperar hasta producción para descubrir:

```text
undefined method
```

durante una query.

---

# 74. Dialect compatibility

Custom dialects deberán declarar:

```text
dialect contract version
supported platform family
capabilities
```

---

# 75. Platform version ≠ framework contract version

No confundir:

```text
PostgreSQL 18
```

con:

```text
VoltStack Driver Contract v2
```

---

# 76. Type extension compatibility

Custom types podrán depender únicamente de contratos públicos.

---

# 77. Stable TypeId

Tipos persistidos deberán usar:

```text
stable TypeId
```

y no necesariamente FQCN.

---

# 78. FQCN persistence problem

Persistir:

```text
App\Domain\Money
```

como identidad estructural puede dificultar renames.

Preferir:

```text
money
```

como alias estable cuando el formato sea persistente.

---

# 79. Entity mapping compatibility

Renombrar una clase PHP no deberá necesariamente implicar renombrar una tabla.

---

# 80. Logical identity

Separar:

```text
PHP class identity
Database table identity
ORM entity type identity
```

---

# 81. EntityTypeId

VoltStack podrá utilizar:

```text
EntityTypeId
```

estable para metadata/cache/serialization cuando corresponda.

---

# 82. Relationship compatibility

Cambiar cardinalidad:

```text
OneToMany
→
OneToOne
```

es un cambio semántico importante y puede requerir migración de datos.

---

# 83. ORM mapping change

No todos los cambios ORM son framework BC issues.

Se distinguirá:

```text
Framework Compatibility
```

de:

```text
Application Schema Evolution
```

---

# 84. Schema Builder compatibility

Métodos estables como:

```php
$table->string('name');
```

seguirán política BC.

---

# 85. Schema semantics

Cambiar silenciosamente:

```text
string()
```

de una longitud/default físico a otro puede tener impacto.

Por ello las abstracciones deberán documentar semántica lógica, no sólo implementación actual.

---

# 86. Platform physical mapping

VoltStack podrá mejorar mapping físico siempre que preserve el contrato lógico y evalúe migración/compatibilidad.

---

# 87. Migration historical compatibility

Migraciones antiguas deberán poder ejecutarse en una instalación nueva compatible cuando su contrato siga soportado.

---

# 88. Migration frozen history

Una vez publicada una migración de aplicación, se recomienda no modificarla.

---

# 89. Framework migration API

VoltStack deberá conservar suficiente compatibilidad para ejecutar migraciones históricas dentro del rango soportado.

---

# 90. Migration API removal

Antes de remover una operación de migración:

```text
deprecate
support old migration interpretation
provide replacement
```

cuando sea viable.

---

# 91. Migration repository format

La tabla que almacena:

```text
migration id
batch
checksum
execution metadata
```

es un formato persistente.

---

# 92. Repository format version

Deberá existir:

```text
MigrationRepositoryFormatVersion
```

si el formato puede evolucionar.

---

# 93. Internal schema migration

VoltStack deberá poder migrar su propio repository metadata.

---

# 94. Cache compatibility

Caches son derivados y normalmente descartables.

Por tanto:

```text
Cache Compatibility
```

puede resolverse mediante invalidación.

---

# 95. Cache generation

Usar:

```text
CacheGeneration
```

o:

```text
FormatVersion
```

en keys.

---

# 96. Example

```text
voltstack:db:metadata:v3:...
```

---

# 97. Old cache entries

Si cambia el formato:

```text
old entries
→
cache miss
```

en lugar de intentar interpretarlas inseguramente.

---

# 98. Cache ≠ persistent business data

Nunca diseñar un upgrade que requiera que un cache antiguo sea la única copia de información.

---

# 99. Compiled metadata compatibility

Compiled metadata puede versionarse:

```text
MetadataCompilerVersion
```

---

# 100. Recompile strategy

Ante incompatibilidad:

```text
invalidate
→
recompile
```

---

# 101. Query compiled cache

Igualmente:

```text
CompilerGeneration
PlatformGeneration
CapabilityGeneration
```

deberán formar parte de identidad cuando sea necesario.

---

# 102. Serialized format compatibility

Algunos artefactos no son fácilmente descartables.

Ejemplos:

```text
cursor tokens
queue payload references
outbox records
migration metadata
backup manifests
```

---

# 103. Format envelope

Se recomienda:

```json
{
  "format": "voltstack.database.cursor",
  "version": 2,
  "payload": {}
}
```

---

# 104. Reader compatibility

Un reader deberá declarar:

```text
supported format versions
```

---

# 105. Writer version

El writer deberá generar normalmente:

```text
current format
```

pero podrá ofrecer compatibilidad controlada.

---

# 106. Unknown format

Nunca interpretar heurísticamente un formato desconocido como si fuera actual.

---

# 107. Cursor compatibility

Cursor pagination tokens pueden sobrevivir entre requests y despliegues.

Por tanto su formato requiere estrategia explícita.

---

# 108. Cursor versioning

Ejemplo conceptual:

```text
CursorEnvelope
├── FormatVersion
├── QueryFingerprint
├── Ordering
├── Boundary
├── ContextFingerprint
└── Signature
```

---

# 109. Deployment compatibility

Durante rolling deployments pueden coexistir:

```text
Application v1
Application v2
```

---

# 110. N/N-1 compatibility

VoltStack podrá definir ventanas de compatibilidad operacional, por ejemplo:

```text
current release
+
previous compatible release
```

para formatos compartidos.

---

# 111. Rolling upgrade requirement

Cambios en:

```text
cache
queue payloads
outbox
cursor tokens
metadata
```

deberán considerar coexistencia temporal.

---

# 112. Expand/contract

Para cambios persistentes se recomienda:

```text
EXPAND
  ↓
MIGRATE
  ↓
SWITCH
  ↓
CONTRACT
```

---

# 113. Example

En vez de:

```text
rename column immediately
```

usar:

```text
add new representation
support both
backfill
switch reads/writes
remove old representation later
```

cuando zero downtime sea requisito.

---

# 114. Database server compatibility

VoltStack deberá declarar una matriz:

```text
MySQL versions
MariaDB versions
PostgreSQL versions
SQLite versions
```

soportadas.

---

# 115. Server support lifecycle

Una versión podrá estar:

```text
SUPPORTED
DEPRECATED
END_OF_SUPPORT
UNTESTED
```

---

# 116. Untested ≠ unsupported

Pero tampoco:

```text
Untested
=
Supported
```

---

# 117. Capability discovery

Cuando sea posible, decisiones runtime dependerán de capabilities.

---

# 118. Minimum platform versions

Sin embargo, VoltStack podrá establecer versiones mínimas por:

- seguridad;
- driver support;
- testing burden;
- required semantics.

---

# 119. Dropping DBMS version

Eliminar soporte de una versión previamente soportada será un cambio de compatibilidad operacional y deberá anunciarse.

---

# 120. PHP compatibility

La versión mínima de PHP también forma parte de la matriz de compatibilidad.

---

# 121. Runtime compatibility

Deberán declararse runtimes soportados:

```text
FrankenPHP
PHP-FPM
CLI
RoadRunner
OpenSwoole
```

según release.

---

# 122. Runtime semantics

El mismo contrato Database deberá conservarse independientemente del runtime soportado.

---

# 123. Persistent runtime

Un cambio que introduzca estado global podría funcionar en PHP-FPM pero romper FrankenPHP.

Por tanto los tests BC deberán incluir persistent runtimes.

---

# 124. Extension compatibility levels

Se propone:

```text
OFFICIAL
STABLE_THIRD_PARTY
EXPERIMENTAL
INTERNAL
```

---

# 125. Official extension

Los paquetes oficiales VoltStack podrán tener una matriz de compatibilidad coordinada.

---

# 126. Third-party extension

Se deberá proporcionar información machine-readable sobre:

```text
database contract version
required capabilities
supported framework versions
```

---

# 127. Package manifest

Ejemplo conceptual:

```json
{
  "voltstack": {
    "database-extension": {
      "contract": "^2.0",
      "capabilities": [
        "query.compiler.extension"
      ]
    }
  }
}
```

---

# 128. Extension startup validation

Bootstrap deberá validar:

```text
extension requirements
```

antes de atender tráfico.

---

# 129. Extension incompatibility

Error recomendado:

```text
DatabaseExtensionCompatibilityException
```

con diagnóstico accionable.

---

# 130. Event compatibility

Eventos públicos también son contratos.

---

# 131. Event schema

Un evento estable deberá definir:

```text
name
payload
field semantics
ordering guarantees
delivery semantics
transaction phase
```

---

# 132. Adding event field

Generalmente puede ser compatible si consumidores toleran campos adicionales.

Pero deberá documentarse.

---

# 133. Removing event field

Es breaking si el campo era estable.

---

# 134. Event meaning change

Cambiar cuándo se emite:

```text
TransactionCommitted
```

es un cambio semántico aunque payload sea idéntico.

---

# 135. Telemetry compatibility

Telemetry interna podrá evolucionar con mayor libertad.

Pero nombres públicos de métricas exportadas podrán considerarse contratos operacionales.

---

# 136. Metric contract

Si una métrica oficial se declara estable:

```text
voltstack_database_query_duration_seconds
```

renombrarla afecta dashboards y alerts.

---

# 137. Metric deprecation

Usar transición:

```text
old metric
+
new metric
```

durante ventana de deprecación cuando sea razonable.

---

# 138. CLI compatibility

Comandos públicos como:

```text
voltstack database:migrate
```

forman parte de DX y automatización.

---

# 139. CLI dimensions

Incluyen:

```text
command name
arguments
options
exit codes
machine-readable output
```

---

# 140. Human output

Texto decorativo para humanos podrá cambiar más libremente.

---

# 141. Machine output

Si existe:

```text
--json
```

su schema deberá versionarse.

---

# 142. Exit codes

Los códigos usados en CI deberán ser estables.

---

# 143. Facade compatibility

Facades oficiales serán API pública.

---

# 144. Helper compatibility

Helpers oficiales también.

---

# 145. Convenience API ≠ weaker contract

Que una API sea “helper” no significa que pueda romperse arbitrariamente si está marcada `STABLE`.

---

# 146. Naming compatibility

Renombrar namespaces/clases públicas es breaking salvo bridge.

---

# 147. Class alias transition

Puede utilizarse temporalmente:

```php
class_alias(
    NewConnectionManager::class,
    OldConnectionManager::class,
);
```

cuando sea seguro.

---

# 148. Alias ≠ permanent architecture

Los aliases deberán tener fecha/versión de retiro.

---

# 149. Method rename strategy

Ejemplo:

```text
oldMethod()
```

podrá:

```text
emit deprecation
delegate to newMethod()
```

---

# 150. Deprecation system

VoltStack deberá poseer:

```text
DatabaseDeprecationRegistry
```

integrado con el sistema general de deprecaciones.

---

# 151. Deprecation metadata

Cada deprecación deberá registrar:

```text
identifier
deprecated since
replacement
removal target
reason
migration instructions
```

---

# 152. Deprecation identifier

Ejemplo:

```text
DB-DEPRECATION-0042
```

---

# 153. Deprecation warning

Ejemplo:

```text
Database::oldTransaction() is deprecated since 1.8.
Use Database::transaction() instead.
Planned removal: 2.0.
```

---

# 154. Deprecation noise

Warnings repetitivos deberán deduplicarse por request/process según configuración.

---

# 155. Production deprecations

No deberán inundar logs de producción.

---

# 156. Development deprecations

En desarrollo/testing podrán ser:

```text
logged
collected
converted to test failures
```

---

# 157. Deprecation testing

CI podrá configurar:

```text
fail_on_deprecation = true
```

para aplicaciones que preparan upgrade.

---

# 158. Removal rule

Una API estable no deberá eliminarse sin:

```text
deprecation
migration path
major compatibility boundary
```

salvo casos excepcionales de seguridad/integridad.

---

# 159. Compatibility windows

Se podrán definir:

```text
Minor Release Window
Major Release Window
LTS Window
```

---

# 160. Semantic versioning relationship

La política detallada se formalizará en:

```text
323_DATABASE_VERSIONING_SYSTEM.md
```

Este documento sólo establece que:

```text
version numbers
```

deben reflejar el impacto de compatibilidad.

---

# 161. Minor release

Normalmente podrá:

```text
add APIs
add optional capabilities
fix bugs
deprecate APIs
improve internals
```

sin breaking changes intencionales.

---

# 162. Patch release

Deberá enfocarse en:

```text
bug fixes
security fixes
performance fixes preserving semantics
```

---

# 163. Major release

Podrá retirar APIs previamente deprecated y realizar cambios incompatibles documentados.

---

# 164. Security exception

Un patch/minor podrá contener cambio incompatible si mantener comportamiento previo implica vulnerabilidad significativa.

Deberá marcarse claramente.

---

# 165. Compatibility severity

Se propone:

```text
NONE
LOW
MODERATE
HIGH
BREAKING
SECURITY_REQUIRED
```

---

# 166. Change classification

Cada cambio relevante deberá producir:

```text
CompatibilityAssessment
```

---

# 167. Assessment model

```text
CompatibilityAssessment
├── ChangeId
├── Domain
├── Stability
├── SourceImpact
├── SemanticImpact
├── DataImpact
├── OperationalImpact
├── ExtensionImpact
├── Severity
├── MigrationRequired
└── Notes
```

---

# 168. Release gate

Antes de publicar:

```text
Release Candidate
```

se ejecutará un:

```text
Compatibility Gate
```

---

# 169. Gate inputs

```text
API diff
contract tests
migration tests
driver conformance
extension tests
serialized format tests
config schema diff
CLI diff
behavior tests
```

---

# 170. Gate output

```text
PASS
PASS_WITH_APPROVED_BREAK
FAIL
```

---

# 171. Approved break

Un breaking change intencional deberá tener:

```text
Architecture Decision
Migration Guide
Release Note
Compatibility Assessment
```

---

# 172. No accidental break

Un cambio no podrá convertirse en breaking simplemente porque:

```text
"it is cleaner"
```

sin evaluar ecosistema.

---

# 173. Compatibility testing architecture

```text
Previous Stable
       │
       ▼
Compatibility Baseline
       │
       ▼
Current Candidate
       │
       ▼
Compatibility Analyzer
       │
       ├── API Diff
       ├── Semantic Contract Tests
       ├── Data Format Tests
       ├── Migration Tests
       ├── Driver Tests
       └── Extension Tests
               │
               ▼
        Compatibility Report
```

---

# 174. Golden contract tests

Para contratos críticos se mantendrán suites que prueben comportamiento histórico.

---

# 175. Example EntityManager contract

```php
public function testPersistDoesNotInsertImmediately(): void
{
    // stable semantic contract
}
```

---

# 176. Example flush contract

```php
public function testFlushDoesNotCommitOuterTransaction(): void
{
}
```

---

# 177. IdentityMap contract test

```php
public function testSameIdentityReturnsSameManagedInstance(): void
{
}
```

---

# 178. Query contract tests

Probar:

```text
null semantics
binding
ordering
pagination
locking
transaction interaction
```

---

# 179. Driver compatibility suite

Custom/official drivers deberán ejecutar:

```text
DriverConformanceSuite
```

---

# 180. Driver conformance version

La suite tendrá versión propia:

```text
DriverConformanceSuite v2
```

asociada al contrato.

---

# 181. Extension contract tests

VoltStack podrá publicar test kits:

```text
DatabaseExtensionTestKit
DriverTestKit
DialectTestKit
TypeTestKit
CompilerExtensionTestKit
```

---

# 182. Application compatibility suite

Antes de major upgrades podrá existir:

```text
voltstack database:compatibility
```

---

# 183. Compatibility command

Ejemplo:

```bash
php voltstack database:compatibility
```

---

# 184. Possible output

```text
Database Compatibility Report

Framework target: 2.0

Deprecated APIs ............ 7
Removed APIs ............... 2
Configuration migrations .. 1
Driver incompatibilities .. 0
Metadata rebuild required . yes
Migration issues ........... 0

Status: ACTION REQUIRED
```

---

# 185. Static analysis

La herramienta podrá analizar:

```text
PHP source
configuration
migration source
extension manifests
```

sin ejecutar aplicación cuando sea posible.

---

# 186. Runtime diagnostics

Algunos problemas sólo podrán detectarse durante bootstrap/runtime.

---

# 187. Upgrade mode

Podrá existir:

```text
Database Upgrade Diagnostic Mode
```

para detectar:

```text
deprecated API usage
old config
legacy metadata
old cache formats
old driver contracts
```

---

# 188. Dry-run upgrade

Idealmente:

```text
database:upgrade-check
```

no modificará datos.

---

# 189. Migration guide generation

La herramienta podrá producir acciones:

```text
Replace X with Y
Rename config A to B
Upgrade driver package
Clear metadata cache
Run migration Z
```

---

# 190. Dependency direction compatibility

Los cambios no deberán romper la arquitectura fundamental:

```text
ORM
  ↓
Query Engine
  ↓
Execution
  ↓
Connection
  ↓
Driver
```

---

# 191. Internal refactor

Se podrá reemplazar completamente la implementación interna mientras los contratos estables permanezcan.

---

# 192. Example

```text
Optimizer v1
→
Optimizer v2
```

puede ser compatible aunque el algoritmo cambie.

---

# 193. Optimizer requirement

Debe conservar:

```text
query semantics
ordering requirements
locking semantics
parameter semantics
security
```

---

# 194. Performance compatibility

VoltStack no prometerá exactamente la misma latencia entre versiones.

Pero regresiones severas deberán detectarse.

---

# 195. Performance ≠ semantic BC

Una query 5% más lenta no es necesariamente breaking API.

Pero:

```text
100x memory growth
```

puede constituir regresión operacional grave.

---

# 196. Operational compatibility

Se deberán evaluar:

```text
memory
connections
pool size
timeouts
worker lifetime
cache size
query count
```

en cambios importantes.

---

# 197. N+1 regression

Una modificación que convierta una operación estable:

```text
2 queries
```

en:

```text
10,001 queries
```

podrá clasificarse como regresión operacional.

---

# 198. Query count guarantee

No todos los query counts serán contratos públicos.

Sólo los explícitamente documentados.

---

# 199. Ordering compatibility

Si una API documenta orden:

```text
ordered by created_at DESC
```

cambiarlo es breaking.

Si no existe orden:

```text
database natural order
```

no deberá tratarse como garantía.

---

# 200. Undefined behavior

Dependencias sobre comportamiento explícitamente `UNDEFINED` no estarán protegidas por BC.

---

# 201. Unspecified behavior

Se distinguirá:

```text
GUARANTEED
UNSPECIFIED
UNDEFINED
```

---

# 202. GUARANTEED

Forma parte del contrato.

---

# 203. UNSPECIFIED

VoltStack no promete una opción concreta entre varias válidas.

Ejemplo:

```text
physical SQL alias names
```

---

# 204. UNDEFINED

El usuario viola precondiciones y el resultado no está garantizado.

---

# 205. Determinism

Cuando el contrato declare comportamiento determinista, deberá conservarse.

---

# 206. Result ordering

Nunca asumir:

```text
SELECT without ORDER BY
```

tiene orden estable.

VoltStack deberá evitar convertir accidentes del DBMS en garantías BC.

---

# 207. Raw SQL compatibility

Raw SQL depende también del DBMS.

VoltStack sólo garantiza:

```text
transport
binding
execution contract
```

no portabilidad del SQL arbitrario.

---

# 208. Raw expression compatibility

Las APIs de escape hatch podrán tener menor portabilidad explícita.

---

# 209. Database portability

Se desarrollará en detalle en:

```text
327_DATABASE_DATABASE_PORTABILITY_SYSTEM.md
```

---

# 210. Legacy integration

Compatibilidad con bases existentes se desarrollará en:

```text
326_DATABASE_LEGACY_DATABASE_INTEGRATION_SYSTEM.md
```

---

# 211. External ORM migration

La transición desde otros ORMs se definirá en:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
```

---

# 212. BC governance

La compatibilidad deberá tener ownership.

Se propone:

```text
DatabaseArchitectureCouncil
```

conceptualmente, aunque la gobernanza inicial pueda recaer en maintainers.

---

# 213. Breaking change approval

Todo breaking change deberá responder:

```text
Why?
What breaks?
Who is affected?
Is there an alternative?
Was it deprecated?
How is it migrated?
Can it be automated?
What data risk exists?
```

---

# 214. Compatibility Decision Record

Se podrá crear:

```text
DatabaseCompatibilityDecision
```

---

# 215. Decision example

```text
DBC-0021
Remove legacy implicit transaction API

Introduced: 1.0
Deprecated: 1.8
Removed: 2.0
Replacement: Database::transaction()
Migration: automated
Risk: low
```

---

# 216. BC budget

Cada major release podrá tener un:

```text
Breaking Change Budget
```

para evitar rediseños innecesarios.

---

# 217. Major ≠ permission to break everything

Una versión major no deberá utilizarse como excusa para romper APIs sin beneficio arquitectónico suficiente.

---

# 218. Compatibility-first design

Antes de estabilizar una API se deberá preguntar:

```text
Can this interface evolve?
Can capabilities be added?
Can implementations remain external?
Can defaults evolve?
Can data formats migrate?
```

---

# 219. Stable API minimalism

Cuanto menor sea la superficie estable:

```text
easier evolution
```

Por ello VoltStack deberá evitar marcar internals como públicos prematuramente.

---

# 220. Extension surface

En vez de exponer internals completos, proporcionar:

```text
narrow explicit extension points
```

---

# 221. Compatibility adapters

Se podrán utilizar:

```text
LegacyAdapter
CompatibilityBridge
DeprecatedFacade
FormatReader
ConfigurationAlias
```

durante transiciones.

---

# 222. Adapter lifecycle

Todo adapter temporal deberá tener:

```text
introduced version
removal target
usage telemetry optional
```

---

# 223. No permanent compatibility sludge

Los bridges no deberán acumularse indefinidamente.

---

# 224. Compatibility telemetry

Con consentimiento/configuración apropiada, desarrollo/testing podrá medir uso de APIs deprecated.

---

# 225. Production privacy

No se enviará código de aplicación ni datos sensibles para medir compatibilidad.

---

# 226. Package compatibility

Cada paquete Quantum podrá declarar:

```text
DatabaseContract
```

requerido.

---

# 227. Example dependency

```text
Quantum/SaaS
      ↓
Database Public Contract v2
```

no:

```text
Quantum/SaaS
      ↓
Database internal class XYZ
```

---

# 228. Multitenancy compatibility

El paquete Multitenancy deberá depender de bridges públicos.

---

# 229. Authentication compatibility

Authentication deberá utilizar contratos de integración definidos, no internals ORM.

---

# 230. Authorization compatibility

Lo mismo para Authorization.

---

# 231. Framework integration stability

Los contratos de los documentos:

```text
311–321
```

forman una frontera importante de integración.

---

# 232. Container compatibility

Cambiar service IDs públicos puede ser breaking.

---

# 233. Service aliases

Podrán mantenerse aliases deprecated temporalmente.

---

# 234. Config integration compatibility

El schema de configuración deberá ser versionable.

---

# 235. Event integration compatibility

Los eventos públicos deberán versionarse semánticamente.

---

# 236. Telemetry integration compatibility

Los contratos del bridge podrán mantenerse estables aunque cambie el backend OpenTelemetry/Prometheus.

---

# 237. Validation compatibility

Cambiar cuándo una regla DB-backed se considera:

```text
VALID
INVALID
UNKNOWN
```

puede ser cambio semántico.

---

# 238. Authentication integration compatibility

Cambiar el modo en que ActorContext participa en DatabaseContext puede afectar seguridad y deberá evaluarse.

---

# 239. Authorization integration compatibility

Cambiar scopes implícitos puede modificar qué filas son visibles.

Esto constituye cambio de seguridad/semántica.

---

# 240. Job integration compatibility

Jobs serializados requieren especial cuidado durante rolling deployments.

---

# 241. HTTP lifecycle compatibility

Cambiar:

```text
implicit flush
transaction middleware
connection reset
```

puede afectar comportamiento de aplicaciones.

---

# 242. Persistent runtime compatibility

Un release compatible deberá preservar aislamiento entre requests.

---

# 243. Compatibility failure modes

Principales riesgos:

```text
Accidental Public API Break
Semantic Drift
Changed Default
Data Format Incompatibility
Driver Contract Drift
Migration History Break
Extension Break
Configuration Drift
Runtime State Leak
Security Scope Drift
```

---

# 244. Failure: accidental API exposure

Problema:

```text
internal class
```

es usada ampliamente porque era pública técnicamente.

Mitigación:

```text
#[Internal]
documentation
static analysis
namespace conventions
```

---

# 245. Failure: silent semantic change

Firma idéntica:

```php
find($id)
```

pero nueva versión consulta réplica stale por default.

Esto puede ser breaking aunque API diff diga:

```text
0 changes
```

---

# 246. Failure: cache format drift

Nuevo código intenta deserializar cache antiguo incompatible.

Mitigación:

```text
format generation
```

---

# 247. Failure: old worker/new worker coexistence

Rolling deployment:

```text
Worker v1 writes format A
Worker v2 expects format B
```

Mitigación:

```text
dual-read/compatible-write window
```

---

# 248. Failure: driver contract mismatch

Framework carga custom driver antiguo.

Mitigación:

```text
bootstrap compatibility negotiation
```

---

# 249. Failure: migration API break

Una instalación nueva no puede ejecutar migraciones históricas.

Mitigación:

```text
historical migration compatibility tests
```

---

# 250. Failure: default transaction change

Cambiar de:

```text
explicit transactions
```

a:

```text
transaction-per-request
```

sería un cambio enorme aunque ninguna API desaparezca.

---

# 251. Failure: security behavior change

Un scope de tenant deja de aplicarse.

Esto será:

```text
critical regression
```

no simple BC issue.

---

# 252. Compatibility priorities

Orden conceptual:

```text
Security
Data Integrity
Correctness
Explicit Stable Contracts
Migration Safety
Ecosystem Compatibility
Developer Convenience
Internal Implementation Freedom
```

---

# 253. Security caveat

El orden anterior no significa ignorar BC.

Significa que una incompatibilidad necesaria por seguridad deberá gestionarse explícitamente, no ocultarse.

---

# 254. Proposed namespace

```text
VoltStack\Quantum\Database\Compatibility
```

---

# 255. Proposed directory structure

```text
src/Quantum/Database/Compatibility/
├── Contract/
│   ├── CompatibilityContract.php
│   ├── CompatibilityAnalyzer.php
│   ├── CompatibilityBaselineProvider.php
│   └── CompatibilityGate.php
│
├── Api/
│   ├── ApiStability.php
│   ├── PublicApiRegistry.php
│   ├── PublicApiDescriptor.php
│   ├── PublicApiSnapshot.php
│   └── PublicApiDiff.php
│
├── Assessment/
│   ├── CompatibilityAssessment.php
│   ├── CompatibilitySeverity.php
│   ├── CompatibilityDomain.php
│   └── CompatibilityReport.php
│
├── Baseline/
│   ├── CompatibilityBaseline.php
│   ├── CompatibilityBaselineBuilder.php
│   └── CompatibilityBaselineRepository.php
│
├── Deprecation/
│   ├── DatabaseDeprecationRegistry.php
│   ├── DatabaseDeprecation.php
│   ├── DeprecationEmitter.php
│   └── DeprecationPolicy.php
│
├── Driver/
│   ├── DriverContractVersion.php
│   ├── DriverCompatibilityChecker.php
│   └── DriverCompatibilityReport.php
│
├── Extension/
│   ├── ExtensionContractVersion.php
│   ├── ExtensionCompatibilityChecker.php
│   └── ExtensionCompatibilityManifest.php
│
├── Format/
│   ├── FormatVersion.php
│   ├── VersionedEnvelope.php
│   ├── FormatReader.php
│   └── FormatCompatibilityRegistry.php
│
├── Config/
│   ├── ConfigurationCompatibilityAnalyzer.php
│   ├── ConfigurationAlias.php
│   └── ConfigurationMigration.php
│
├── Migration/
│   ├── MigrationCompatibilityAnalyzer.php
│   └── HistoricalMigrationTestRunner.php
│
├── Testing/
│   ├── CompatibilityTestKit.php
│   ├── SemanticContractSuite.php
│   ├── DriverContractTestKit.php
│   └── ExtensionContractTestKit.php
│
├── Tooling/
│   ├── CompatibilityCommand.php
│   ├── UpgradeCheckCommand.php
│   └── CompatibilityReportRenderer.php
│
└── Exception/
    ├── CompatibilityException.php
    ├── IncompatibleDriverException.php
    ├── IncompatibleExtensionException.php
    └── UnsupportedFormatVersionException.php
```

---

# 256. Stable namespace policy

Se recomienda distinguir:

```text
VoltStack\Quantum\Database\Contracts
VoltStack\Quantum\Database\Extension
```

como superficies deliberadamente públicas.

Internals podrán utilizar:

```text
VoltStack\Quantum\Database\Internal
```

cuando ayude a evitar uso accidental.

---

# 257. Attributes

Se podrán proporcionar:

```php
#[Stable]
#[Experimental]
#[Internal]
#[Deprecated(
    since: '1.8',
    removal: '2.0',
    replacement: Database::class.'::transaction'
)]
```

---

# 258. Documentation markers

Toda documentación pública deberá indicar estabilidad cuando no sea `STABLE`.

Ejemplo:

```text
@experimental
```

---

# 259. IDE support

Los attributes/docblocks permitirán que IDEs y static analyzers adviertan sobre APIs no estables.

---

# 260. Composer constraints

Los paquetes oficiales deberán usar constraints compatibles con la política de versionado.

---

# 261. Lockfile ≠ compatibility guarantee

Que Composer pueda resolver paquetes no demuestra compatibilidad semántica.

---

# 262. Bootstrap validation

VoltStack deberá validar contratos críticos al iniciar.

---

# 263. Fast failure

Preferir:

```text
application bootstrap fails
```

con explicación clara sobre:

```text
random production query fails hours later
```

---

# 264. Compatibility manifest

Cada release podrá incluir:

```text
database-compatibility.json
```

conceptualmente.

---

# 265. Manifest example

```json
{
  "database": {
    "version": "2.0.0",
    "public_api_contract": 2,
    "driver_contract": 3,
    "extension_contract": 2,
    "metadata_format": 4,
    "migration_repository_format": 2
  }
}
```

---

# 266. Independent contract versions

No todos los formatos deberán cambiar con el número global del framework.

Ejemplo:

```text
VoltStack 4.2
Driver Contract 3
Metadata Format 7
Cursor Format 2
```

---

# 267. Why independent versions

Permite evolucionar componentes sin:

```text
version = framework version
```

artificialmente.

---

# 268. Compatibility matrix

Ejemplo:

| VoltStack Database | Driver Contract | Metadata Format | Migration Repo | Extension Contract |
|---|---:|---:|---:|---:|
| 1.x | 1 | 1–2 | 1 | 1 |
| 2.x | 1–2 | 2–3 | 1–2 | 1–2 |
| 3.x | 2–3 | 3–4 | 2 | 2–3 |

La tabla es conceptual; los valores reales se definirán durante releases.

---

# 269. Upgrade planner

Futuro:

```text
Current Installation
      │
      ▼
Compatibility Inspector
      │
      ▼
Upgrade Plan
```

---

# 270. Upgrade Plan

Podrá contener:

```text
Upgrade packages
Replace APIs
Migrate config
Clear caches
Upgrade driver
Recompile metadata
Run framework metadata migration
Run application schema migration
Restart workers
```

---

# 271. Worker upgrade

Persistent workers deberán reiniciarse cuando carguen código/metadata incompatible.

---

# 272. Hot code replacement

VoltStack Database no asumirá que puede reemplazar arbitrariamente clases en workers activos.

---

# 273. Deployment generation

Cada worker podrá asociarse a:

```text
DeploymentGeneration
```

---

# 274. Mixed generation

Durante rolling deployment:

```text
Generation A
Generation B
```

podrán coexistir sólo si los recursos compartidos son compatibles.

---

# 275. Shared resource analysis

Revisar:

```text
database schema
cache
queue
outbox
migration repository
locks
cursor tokens
```

---

# 276. Schema compatibility window

Para zero-downtime deployments:

```text
Schema
```

deberá ser compatible temporalmente con ambas generaciones.

---

# 277. Compatibility equation

Conceptualmente:

```text
SafeRollingDeployment
=
SchemaCompatible(A, B)
∧
SharedFormatsCompatible(A, B)
∧
QueuePayloadCompatible(A, B)
∧
CacheCompatibleOrVersioned(A, B)
```

---

# 278. BC formal model

Sea:

```text
C(v)
```

el conjunto de contratos estables de una versión `v`.

Una release compatible `v2` respecto a `v1` deberá conservar:

```text
C(v1) ⊆ SupportedContracts(v2)
```

salvo excepciones explícitamente permitidas por la política.

---

# 279. Semantic compatibility model

Para una operación estable `O`:

```text
Semantics_v1(O, X)
≈
Semantics_v2(O, X)
```

para inputs válidos `X`.

`≈` significa equivalencia conforme al contrato documentado, no implementación idéntica.

---

# 280. Internal freedom

No se exige:

```text
Implementation_v1(O)
=
Implementation_v2(O)
```

---

# 281. Observable behavior

Se evaluará:

```text
return value
exception category
side effects
persistence
transaction outcome
identity guarantees
security scope
documented ordering
```

---

# 282. Compatibility under uncertainty

Si no puede demostrarse que un cambio es compatible:

```text
UNKNOWN
```

no deberá clasificarse automáticamente como:

```text
SAFE
```

---

# 283. Compatibility evidence

La clasificación deberá basarse en:

```text
API diff
tests
documentation
benchmarks
migration simulation
extension matrix
driver conformance
production feedback
```

según caso.

---

# 284. BC test matrix

Mínimo:

```text
Previous Patch → Current Patch
Previous Minor → Current Minor
Previous Major Supported → Current Major
```

según política de soporte.

---

# 285. Database matrix

Además:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 286. Runtime matrix

Y:

```text
FrankenPHP
CLI
PHP-FPM
RoadRunner
OpenSwoole
```

según soporte oficial de cada release.

---

# 287. Extension matrix

Probar paquetes oficiales:

```text
Multitenancy
SaaS
Authentication
Authorization
Telemetry
Cache
Jobs
```

cuando integren Database.

---

# 288. BC CI pipeline

```text
Pull Request
   │
   ▼
Build
   │
   ├── Unit Tests
   ├── API Diff
   ├── Contract Tests
   ├── Driver Matrix
   ├── Migration Compatibility
   ├── Format Compatibility
   ├── Extension Matrix
   └── Runtime Matrix
          │
          ▼
Compatibility Gate
```

---

# 289. PR compatibility annotation

Cambios podrán etiquetarse:

```text
BC-NONE
BC-DEPRECATION
BC-BEHAVIORAL
BC-BREAKING
BC-SECURITY
```

---

# 290. Automated rejection

CI podrá rechazar:

```text
removed stable API
```

en una minor release sin waiver aprobado.

---

# 291. Waiver

Un:

```text
CompatibilityWaiver
```

deberá explicar la excepción.

---

# 292. Changelog

Los releases deberán separar:

```text
Added
Changed
Deprecated
Removed
Fixed
Security
Migration Required
```

---

# 293. Upgrade documentation

Cada major deberá proporcionar:

```text
UPGRADE_FROM_1_TO_2.md
```

o equivalente.

---

# 294. Automated codemods

Cuando sea posible, VoltStack podrá ofrecer transformaciones automáticas.

Ejemplo:

```text
old facade call
→
new API
```

---

# 295. Codemod safety

Un codemod sólo deberá modificar código cuando pueda demostrar una transformación segura.

---

# 296. Human review

Transformaciones ambiguas deberán marcarse:

```text
MANUAL REVIEW REQUIRED
```

---

# 297. Compatibility and generated code

Código generado deberá incluir:

```text
generator version
```

cuando sea relevante.

---

# 298. Old generated code

Una nueva versión deberá evitar asumir que todo código generado fue regenerado inmediatamente.

---

# 299. Generator upgrade

Podrá existir:

```text
database:codegen --upgrade
```

---

# 300. Reference implementation

El documento:

```text
328_DATABASE_REFERENCE_IMPLEMENTATION.md
```

deberá respetar las fronteras de estabilidad definidas aquí.

---

# 301. Dependency rules

El documento:

```text
332_DATABASE_DEPENDENCY_RULES.md
```

formalizará qué internals no deberán consumirse externamente.

---

# 302. Architectural invariants

El documento:

```text
334_DATABASE_ARCHITECTURAL_INVARIANTS.md
```

consolidará las invariantes que no pueden romperse incluso entre major releases sin rediseño explícito del sistema.

---

# 303. Compatibility invariants

## DB-BC-001

Public ≠ Stable automáticamente.

## DB-BC-002

Accessible ≠ Public API.

## DB-BC-003

Same Signature ≠ Same Semantics.

## DB-BC-004

Code Compiles ≠ Backward Compatible.

## DB-BC-005

Deprecated ≠ Removed.

## DB-BC-006

Experimental ≠ Stable.

## DB-BC-007

Internal ≠ Extension Point.

## DB-BC-008

Bug Compatibility ≠ Backward Compatibility.

## DB-BC-009

Security may require intentional behavior change.

## DB-BC-010

Breaking changes deberán ser explícitos.

---

# 304. API invariants

## DB-BC-011

Stable APIs estarán inventariadas.

## DB-BC-012

Stable interface evolution considerará implementadores externos.

## DB-BC-013

Removing stable method será breaking.

## DB-BC-014

Reducing visibility será breaking.

## DB-BC-015

Narrowing accepted input podrá ser breaking.

## DB-BC-016

Changing documented return semantics será breaking.

## DB-BC-017

Exception contracts forman parte de API.

## DB-BC-018

Stable error codes no dependerán de mensajes humanos.

## DB-BC-019

Facades estables estarán protegidas.

## DB-BC-020

Helpers estables estarán protegidos.

---

# 305. Semantic invariants

## DB-BC-021

`persist()` no cambiará silenciosamente a immediate INSERT.

## DB-BC-022

`flush()` no cambiará silenciosamente a commit.

## DB-BC-023

IdentityMap guarantees no cambiarán silenciosamente.

## DB-BC-024

Transaction semantics forman parte del contrato.

## DB-BC-025

Query binding semantics forman parte del contrato.

## DB-BC-026

Null semantics documentadas forman parte del contrato.

## DB-BC-027

Ordering sólo será BC cuando esté garantizado.

## DB-BC-028

Unspecified behavior no será accidentalmente estable.

## DB-BC-029

Undefined behavior no tendrá garantía BC.

## DB-BC-030

Security scope forma parte de semántica observable.

---

# 306. Driver invariants

## DB-BC-031

Driver contract será versionado.

## DB-BC-032

Driver compatibility se validará en bootstrap.

## DB-BC-033

New capability no implicará soporte automático.

## DB-BC-034

UNKNOWN ≠ SUPPORTED.

## DB-BC-035

Vendor Version ≠ Driver Contract Version.

## DB-BC-036

Driver extensions usarán contratos públicos.

## DB-BC-037

Conformance suite acompañará contratos.

## DB-BC-038

Contract mismatch fallará temprano.

## DB-BC-039

Custom driver no dependerá de internals documentados como tales.

## DB-BC-040

Official drivers participarán en BC matrix.

---

# 307. Extension invariants

## DB-BC-041

Extension contract será versionado.

## DB-BC-042

Plugins declararán requirements.

## DB-BC-043

Bootstrap validará requirements.

## DB-BC-044

Extension point ≠ arbitrary internal override.

## DB-BC-045

Custom types tendrán identidad estable cuando se persista.

## DB-BC-046

Custom compilers usarán SPI.

## DB-BC-047

Custom ORM extensions respetarán UoW/IdentityMap.

## DB-BC-048

Extensions no podrán saltarse security invariants.

## DB-BC-049

Extensions no podrán inventar transaction outcomes.

## DB-BC-050

Third-party compatibility tendrá diagnóstico explícito.

---

# 308. Data invariants

## DB-BC-051

Framework upgrade ≠ schema reset.

## DB-BC-052

Persistent formats tendrán versionado cuando sea necesario.

## DB-BC-053

Unknown format ≠ current format.

## DB-BC-054

Cache incompatible podrá invalidarse.

## DB-BC-055

Business data no dependerá exclusivamente de cache.

## DB-BC-056

Migration repository evolution tendrá estrategia.

## DB-BC-057

Historical migrations serán consideradas.

## DB-BC-058

Cursor tokens tendrán formato identificable.

## DB-BC-059

Rolling deployments considerarán mixed versions.

## DB-BC-060

Persistent identifiers evitarán FQCN cuando requieran estabilidad.

---

# 309. Configuration invariants

## DB-BC-061

Stable config keys son API.

## DB-BC-062

Config key removal requiere transición.

## DB-BC-063

Default changes serán evaluados como behavior changes.

## DB-BC-064

Configuration precedence forma parte del comportamiento.

## DB-BC-065

Secret handling no se degradará por compatibilidad.

## DB-BC-066

Legacy aliases serán temporales.

## DB-BC-067

Deprecated config emitirá diagnóstico.

## DB-BC-068

Config migration podrá automatizarse.

## DB-BC-069

Unknown config no será ignorada silenciosamente cuando afecte seguridad.

## DB-BC-070

Configuration schema podrá versionarse.

---

# 310. Runtime invariants

## DB-BC-071

Persistent runtime isolation deberá mantenerse.

## DB-BC-072

Worker-global mutable state será incompatible con arquitectura.

## DB-BC-073

Runtime adapters respetarán Database semantics.

## DB-BC-074

FrankenPHP será parte de la matriz principal.

## DB-BC-075

Runtime upgrade considerará worker restart.

## DB-BC-076

Mixed deployment generations deberán ser evaluadas.

## DB-BC-077

Shared resources deberán ser format-compatible.

## DB-BC-078

Connection reset semantics no se debilitarán.

## DB-BC-079

Request scope semantics no se debilitarán.

## DB-BC-080

Coroutine isolation deberá mantenerse donde aplique.

---

# 311. Governance invariants

## DB-BC-081

Todo breaking change tendrá assessment.

## DB-BC-082

Breaking change intencional tendrá migration path cuando sea posible.

## DB-BC-083

Major release ≠ permission to break arbitrarily.

## DB-BC-084

Deprecations tendrán identificador.

## DB-BC-085

Deprecations tendrán replacement cuando exista.

## DB-BC-086

Deprecations tendrán target de removal cuando sea razonable.

## DB-BC-087

Release gate verificará BC.

## DB-BC-088

API diff no será única evidencia.

## DB-BC-089

Semantic contract tests serán obligatorios para invariantes críticas.

## DB-BC-090

UNKNOWN compatibility ≠ SAFE.

---

# 312. Operational invariants

## DB-BC-091

Operational regressions severas serán evaluadas.

## DB-BC-092

Performance exacta no será normalmente API.

## DB-BC-093

Resource behavior crítico sí será monitoreado.

## DB-BC-094

N+1 regression podrá bloquear release.

## DB-BC-095

Connection explosion podrá bloquear release.

## DB-BC-096

Memory leak podrá bloquear release.

## DB-BC-097

Worker contamination bloqueará release.

## DB-BC-098

Telemetry schema estable se versionará cuando sea pública.

## DB-BC-099

CLI machine output tendrá schema estable.

## DB-BC-100

CLI human formatting podrá evolucionar más libremente.

---

# 313. Final invariants

## DB-BC-101

ORM internals podrán evolucionar detrás de contratos.

## DB-BC-102

Query optimizer podrá evolucionar preservando semántica.

## DB-BC-103

Compiler output textual exacto no será API salvo declaración.

## DB-BC-104

SQL formatting no será contrato.

## DB-BC-105

Security corrections podrán superar BC accidental.

## DB-BC-106

Data integrity corrections podrán superar BC accidental.

## DB-BC-107

Every stable contract shall have an owner.

## DB-BC-108

Every persistent format shall have an evolution strategy.

## DB-BC-109

Every extension point shall define its stability.

## DB-BC-110

Every supported platform shall have compatibility evidence.

---

# 314. Additional invariants

## DB-BC-111

Compatibility bridges serán temporales.

## DB-BC-112

Legacy support no contaminará permanentemente core semantics.

## DB-BC-113

Upgrade diagnostics no modificarán datos por defecto.

## DB-BC-114

Codemods no harán transformaciones ambiguas silenciosamente.

## DB-BC-115

Generated code tendrá estrategia de upgrade.

## DB-BC-116

Metadata cache podrá reconstruirse.

## DB-BC-117

Compiled query cache podrá reconstruirse.

## DB-BC-118

Migration history no será cache descartable.

## DB-BC-119

Backup manifests tendrán versionado cuando corresponda.

## DB-BC-120

Outbox formats deberán considerar rolling deployment.

---

# 315. Compatibility rules summary

La arquitectura puede resumirse:

```text
                   Compatibility
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
      API             Semantics           Data
       │                 │                 │
       ▼                 ▼                 ▼
 Signatures         Behavior         Persistent Formats
 Exceptions         Transactions     Schema
 Config             Identity         Migrations
 CLI                Security         Metadata
       │                 │                 │
       └─────────────────┼─────────────────┘
                         ▼
                 Compatibility Gate
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
             PASS              BREAKING CHANGE
                                      │
                                      ▼
                              Migration Strategy
```

---

# 316. Compatibility hierarchy

VoltStack deberá pensar en compatibilidad desde:

```text
Application
    │
    ▼
Public Database API
    │
    ▼
Stable Extension Contracts
    │
    ▼
Semantic Architecture
    │
    ▼
Internal Implementation
```

La libertad de cambio aumenta hacia abajo.

---

# 317. Public contract principle

> **Cuanto más externa sea una superficie, mayor deberá ser su estabilidad. Cuanto más interna sea una implementación, mayor podrá ser su libertad de evolución.**

---

# 318. Persistent contract principle

> **Cuanto más tiempo sobreviva un artefacto fuera del proceso que lo creó, mayor deberá ser la disciplina de versionado aplicada a su formato.**

Por ejemplo:

```text
local temporary AST
<
metadata cache
<
cursor token
<
queue payload
<
migration history
<
business data
```

en términos de necesidad potencial de compatibilidad persistente.

---

# 319. Semantic contract principle

> **VoltStack protegerá principalmente significados, garantías y fronteras arquitectónicas; no accidentes de implementación.**

Por ello:

```text
Query Semantics
```

puede ser estable mientras:

```text
Generated SQL Formatting
```

cambia.

---

# 320. Security principle

> **La compatibilidad nunca deberá convertirse en una justificación para preservar vulnerabilidades, pérdida de aislamiento, corrupción de datos o resultados transaccionales falsos.**

Cuando una corrección incompatible sea necesaria:

```text
identify
document
mitigate
migrate
test
release explicitly
```

---

# 321. Extension principle

> **Los desarrolladores externos deberán poder construir extensiones estables sin depender de los internals del motor Database.**

Esto requiere:

```text
Public SPI
+
Capability Model
+
Contract Version
+
Conformance Tests
```

---

# 322. Persistent runtime principle

> **Una release compatible de VoltStack Database deberá conservar las garantías de aislamiento aun cuando el proceso PHP sobreviva durante miles de requests.**

Es decir:

```text
Backward Compatibility
```

también incluye que:

```text
FrankenPHP Request N
```

no observe estado mutable perteneciente a:

```text
FrankenPHP Request N-1
```

---

# 323. Release decision model

Todo cambio significativo deberá pasar conceptualmente por:

```text
Change
  │
  ▼
Is affected surface public?
  │
  ├── No → internal evolution
  │
  └── Yes
       │
       ▼
Is behavior guaranteed?
       │
       ├── No → evaluate operational risk
       │
       └── Yes
            │
            ▼
Does change preserve contract?
            │
       ┌────┴────┐
       ▼         ▼
      Yes        No
       │          │
       ▼          ▼
 Compatible   Can deprecate?
                  │
             ┌────┴────┐
             ▼         ▼
            Yes        No
             │          │
             ▼          ▼
        Deprecation   Security/
        Lifecycle     Integrity/
                      Major Break
```

---

# 324. V1 requirements

La primera versión estable del sistema deberá incluir al menos:

```text
API stability markers
Public API registry
Compatibility domains
Deprecation metadata
Deprecation emitter
API baseline generation
API diff
Driver contract version
Extension contract version
Format version abstraction
Metadata cache generation
Configuration compatibility metadata
Compatibility test suite
Driver conformance integration
Historical migration tests
Upgrade diagnostic command
Release compatibility gate
```

---

# 325. V2

Podrá incorporar:

```text
semantic contract diffing
automated config migration
codemods
extension compatibility manifests
rolling deployment analyzer
serialized format negotiation
automatic migration compatibility scanning
API usage reports
advanced deprecation analytics
```

---

# 326. V3

Podrá incorporar:

```text
automated upgrade planner
application compatibility graph
package ecosystem compatibility analysis
cross-version integration sandbox
schema deployment compatibility simulation
AI-assisted migration recommendations
```

La IA podrá:

```text
analyze
recommend
explain
```

pero no deberá modificar automáticamente datos o schemas de producción sin autorización y controles de seguridad.

---

# 327. Architecture overview

```text
              VoltStack Database Release
                         │
                         ▼
               Compatibility Analyzer
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
   Public API         Semantics        Persistent Data
       │                 │                 │
       ▼                 ▼                 ▼
    API Diff        Contract Tests    Format Analysis
       │                 │                 │
       └─────────────────┼─────────────────┘
                         ▼
                 Compatibility Report
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
 Compatible          Deprecated         Breaking
       │                 │                 │
       ▼                 ▼                 ▼
   Release        Migration Path     Approval + Guide
                         │
                         ▼
                  Compatibility Gate
```

---

# 328. Regla definitiva

> **VoltStack Database considerará backward compatibility como la conservación deliberada de contratos públicos, significados, garantías de seguridad, formatos persistentes y puntos de extensión oficialmente soportados; no como la congelación de su implementación interna.**

Por tanto:

```text
Backward Compatibility
=
Stable Contract Preservation
+
Semantic Preservation
+
Data Evolution Strategy
+
Extension Stability
+
Operational Safety
```

y nunca simplemente:

```text
Backward Compatibility
=
"No classes were renamed"
```

---

# 329. Resultado arquitectónico

Esta estrategia permite que VoltStack Database pueda evolucionar:

```text
Query Engine v1 → v5
ORM v1 → v4
Compiler v1 → v7
Driver Architecture v1 → v3
```

sin exigir que cada mejora interna rompa aplicaciones.

Al mismo tiempo evita congelar decisiones internas prematuras.

La frontera será:

```text
Stable Outside
+
Evolvable Inside
```

---

# 330. Relación con los siguientes documentos

Este documento establece **qué debe protegerse**.

Los siguientes documentos definirán **cómo evoluciona y se gobierna**.

```text
322 Backward Compatibility
       │
       ▼
323 Versioning
       │
       ▼
324 Deprecation Policy
       │
       ├── 325 External ORM Migration
       ├── 326 Legacy Database Integration
       └── 327 Database Portability
```

---

# 331. Siguiente documento

```text
323_DATABASE_VERSIONING_SYSTEM.md
```

El siguiente documento deberá formalizar el sistema completo de versionado de Database, incluyendo:

```text
Framework Version
Database Component Version
Public API Contract Version
Driver Contract Version
Extension Contract Version
Metadata Format Version
Migration Repository Version
Cursor Format Version
Cache Generation
Capability Generation
```

y establecer reglas para:

```text
MAJOR
MINOR
PATCH
```

junto con:

- Semantic Versioning;
- independent contract versions;
- package compatibility;
- release channels;
- pre-releases;
- LTS;
- security releases;
- support windows;
- DBMS compatibility;
- PHP compatibility;
- runtime compatibility;
- driver compatibility;
- extension compatibility;
- schema compatibility;
- rolling upgrades;
- downgrade policy;
- version negotiation;
- release manifests;
- compatibility matrices;
- upgrade paths;
- release gates.