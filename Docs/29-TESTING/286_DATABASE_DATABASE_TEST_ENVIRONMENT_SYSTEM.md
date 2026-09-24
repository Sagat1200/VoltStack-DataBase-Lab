# 286_DATABASE_DATABASE_TEST_ENVIRONMENT_SYSTEM.md

# VoltStack Quantum Database
## Database Test Environment System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 286 — Database Test Environment System  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `285_DATABASE_INTEGRATION_TESTING_SYSTEM.md`  
**Siguiente documento:** `287_DATABASE_TRANSACTIONAL_TESTING_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database Test Environment System** de VoltStack.

Su responsabilidad será proporcionar entornos de base de datos:

- reproducibles;
- aislados;
- identificables;
- protegidos;
- provisionables;
- reseteables;
- observables;
- destruibles;
- compatibles con ejecución paralela;
- adecuados para Unit, Integration, Conformance, Failure y Performance Testing.

Regla central:

> **Un entorno de pruebas Database en VoltStack será un recurso explícitamente identificado, aislado y gobernado cuyo ciclo de vida deberá poder ser demostrado antes de permitir operaciones destructivas.**

Formalmente:

```text
TestEnvironment
=
EnvironmentIdentity
+
Infrastructure
+
DatabasePlatform
+
Driver
+
Configuration
+
Capabilities
+
Isolation
+
Credentials
+
Lifecycle
+
SafetyPolicy
```

No:

```text
TestEnvironment
=
"whatever database is configured"
```

---

# 2. Problema arquitectónico

Las pruebas de Database pueden modificar:

```text
schemas
tables
indexes
constraints
rows
transactions
sequences
users
permissions
databases
replication state
backup artifacts
temporary files
```

Por tanto, un error de configuración podría convertir:

```text
test execution
```

en:

```text
production data destruction
```

El Test Environment System deberá impedir que el testing dependa únicamente de convenciones informales.

---

# 3. Objetivos

El sistema deberá permitir:

1. definir perfiles reproducibles;
2. provisionar DBMS;
3. seleccionar versiones;
4. seleccionar drivers;
5. crear bases de pruebas;
6. configurar credenciales;
7. verificar seguridad;
8. comprobar readiness;
9. descubrir capabilities;
10. preparar schemas;
11. cargar fixtures;
12. aislar tests;
13. ejecutar tests en paralelo;
14. resetear estado;
15. limpiar recursos;
16. destruir entornos;
17. detectar recursos abandonados;
18. reproducir fallos;
19. integrar CI;
20. soportar desarrollo local.

---

# 4. Environment ≠ Database

Un entorno puede contener:

```text
Test Environment
│
├── DBMS
├── Database
├── Credentials
├── Driver
├── Network
├── Filesystem
├── Native Tools
├── Replica(s)
├── Cache Provider
└── Auxiliary Resources
```

Por tanto:

```text
Environment ≠ Database
```

---

# 5. Environment ≠ Configuration

La configuración describe cómo obtener o construir un entorno.

No es el entorno mismo.

```text
EnvironmentProfile
        ↓
Provision
        ↓
TestEnvironment
```

---

# 6. Environment ≠ Fixture

```text
Environment
=
infrastructure
```

```text
Fixture
=
known data scenario
```

Ambos deberán permanecer separados.

---

# 7. Environment ≠ Test Context

El Environment puede sobrevivir a múltiples tests.

El Test Context pertenece a una ejecución específica.

```text
Environment
   │
   ├── Test A Context
   ├── Test B Context
   └── Test C Context
```

---

# 8. Environment ≠ Test Database

Un DBMS podrá alojar:

```text
Database A
Database B
Database C
```

para distintos workers/tests.

---

# 9. Arquitectura general

```text
                  Test Runner
                       │
                       ▼
               Environment Request
                       │
                       ▼
               Environment Resolver
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
           Local    Container   Remote
             │         │         │
             └─────────┼─────────┘
                       ▼
                 Provisioner
                       │
                       ▼
                Safety Validator
                       │
                       ▼
                  Readiness
                       │
                       ▼
              Capability Discovery
                       │
                       ▼
                Test Environment
                       │
           ┌───────────┼───────────┐
           ▼           ▼           ▼
      Test Database   Schema     Credentials
           │
           ▼
                    Tests
                       │
                       ▼
                     Reset
                       │
                       ▼
                    Cleanup
                       │
                       ▼
                    Destroy
```

---

# 10. Environment Request

La solicitud deberá expresar intención.

```php
final readonly class TestEnvironmentRequest
{
    public function __construct(
        public PlatformRequirement $platform,
        public ?VersionRequirement $version,
        public ?DriverRequirement $driver,
        public CapabilityRequirements $capabilities,
        public IsolationRequirement $isolation,
        public ResourceRequirements $resources,
    ) {}
}
```

---

# 11. Environment Resolution

El resolver deberá encontrar un perfil capaz de satisfacer la solicitud.

```text
Request
  ↓
Available Profiles
  ↓
Compatibility Evaluation
  ↓
Selected Profile
```

---

# 12. Resolution ≠ Provisioning

Primero:

```text
Which environment?
```

Después:

```text
Create/connect it.
```

Esto permite:

- explainability;
- dry-run;
- diagnostics;
- CI planning.

---

# 13. Environment Profile

Un perfil podrá contener:

```text
EnvironmentProfile
├── id
├── provider
├── platform
├── server version
├── driver
├── topology
├── credentials profile
├── isolation policy
├── reset policy
├── resource budget
├── safety policy
├── native tools
└── lifecycle policy
```

---

# 14. Profile example

```yaml
id: postgresql-17-integration

provider: container

database:
  platform: postgresql
  version: "17"

driver:
  name: pdo_pgsql

isolation:
  strategy: database_per_worker

reset:
  strategy: drop_and_recreate

safety:
  destructive: true
  production_protection: strict
```

La sintaxis definitiva podrá evolucionar.

---

# 15. Profiles are declarative

Un profile no deberá ejecutar infraestructura durante su lectura.

```text
Parse Profile
≠
Provision Environment
```

---

# 16. Profile Registry

Podrá existir:

```php
interface TestEnvironmentProfileRegistry
{
    public function get(EnvironmentProfileId $id): TestEnvironmentProfile;

    public function all(): iterable;
}
```

---

# 17. Registry lifecycle

El registry podrá congelarse después del bootstrap:

```text
REGISTERING
    ↓
FROZEN
```

evitando mutaciones arbitrarias durante los tests.

---

# 18. Environment Providers

El sistema deberá abstraer la procedencia de la infraestructura.

```php
interface TestEnvironmentProvider
{
    public function supports(
        TestEnvironmentProfile $profile
    ): bool;

    public function provision(
        TestEnvironmentProfile $profile,
        TestRunContext $context
    ): TestEnvironment;

    public function destroy(
        TestEnvironment $environment
    ): void;
}
```

---

# 19. Provider types

Inicialmente podrán existir:

```text
LocalEnvironmentProvider
ContainerEnvironmentProvider
RemoteEnvironmentProvider
ExternalEnvironmentProvider
```

---

# 20. Docker no será arquitectura

El sistema podrá usar Docker.

Pero:

```text
Database Test Environment
≠
Docker
```

Docker será un adapter/provider.

---

# 21. Podman

Podrá existir otro adapter sin alterar los contratos centrales.

---

# 22. CI services

GitHub Actions, GitLab CI u otros sistemas podrán proporcionar servicios externos.

VoltStack deberá poder consumirlos como entornos externos.

---

# 23. Local Environment

Permite usar:

```text
localhost PostgreSQL
localhost MySQL
local SQLite
local MariaDB
```

si están explícitamente configurados como testing.

---

# 24. Local environment safety

El hecho de usar:

```text
localhost
```

no demuestra que la base sea desechable.

Por tanto:

```text
localhost ≠ safe
```

---

# 25. Remote Environment

Podrá utilizarse un servidor remoto dedicado a testing.

Deberá estar:

```text
explicitly configured
authorized
identified
guarded
```

---

# 26. Remote ≠ production

La configuración deberá demostrar que el destino pertenece al dominio de testing permitido.

---

# 27. Environment Identity

Cada entorno tendrá una identidad estable durante su ciclo de vida.

```php
final readonly class TestEnvironmentIdentity
{
    public function __construct(
        public TestEnvironmentId $id,
        public TestRunId $run,
        public EnvironmentProfileId $profile,
        public EnvironmentOwner $owner,
    ) {}
}
```

---

# 28. TestRunId

Cada ejecución global deberá tener:

```text
TestRunId
```

Ejemplo conceptual:

```text
run_20260919_7f8c...
```

---

# 29. WorkerId

Ejecuciones paralelas podrán utilizar:

```text
WorkerId
```

---

# 30. Resource Namespace

Los recursos deberán poder derivarse de:

```text
TestRunId
+
WorkerId
+
ResourceType
```

---

# 31. Example naming

```text
voltstack_test_<run>_<worker>
```

para una base temporal.

---

# 32. Naming ≠ only safety mechanism

El nombre será una señal.

No será la única defensa.

---

# 33. Safety Architecture

```text
Destructive Operation
        │
        ▼
Environment Identity
        │
        ▼
Safety Evidence
        │
        ▼
Environment Guard
        │
        ▼
ALLOW / REJECT
```

---

# 34. Environment Guard

Podrá existir:

```php
interface TestEnvironmentGuard
{
    public function authorize(
        TestEnvironment $environment,
        TestEnvironmentOperation $operation
    ): EnvironmentGuardDecision;
}
```

---

# 35. Guard inputs

Podrá evaluar:

```text
environment type
profile
host
port
database name
server identity
credentials
test marker
run identity
resource ownership
operation severity
```

---

# 36. Safety Levels

Podrán definirse:

```text
READ_ONLY
MUTATING
SCHEMA_MUTATING
DATABASE_DESTRUCTIVE
SERVER_ADMINISTRATIVE
```

---

# 37. Stronger operation → stronger evidence

Formalmente:

```text
RequiredSafetyEvidence
∝
OperationDestructiveness
```

---

# 38. Read-only probe

Puede necesitar pocas verificaciones.

---

# 39. DROP DATABASE

Deberá requerir evidencia fuerte.

---

# 40. Production Protection

El sistema deberá permitir listas de:

```text
forbidden hosts
forbidden database names
forbidden connection fingerprints
forbidden environments
```

---

# 41. Production environment marker

Si:

```text
APP_ENV=production
```

las operaciones destructivas de testing deberán rechazarse por defecto.

---

# 42. Environment variable spoofing

No se confiará exclusivamente en:

```text
APP_ENV=test
```

---

# 43. Database Identity Verification

Después de conectar, podrán verificarse propiedades del servidor.

```text
expected platform
expected database
expected server marker
expected role
```

---

# 44. Server marker

En entornos administrados por VoltStack podrá existir metadata específica de testing.

---

# 45. Marker absence

No siempre implicará entorno inválido.

Dependerá de la política del provider.

---

# 46. Explicit external environments

Un servidor externo preexistente podrá declararse seguro mediante configuración explícita y restricciones adicionales.

---

# 47. Provisioning Lifecycle

```text
DECLARED
   ↓
RESOLVING
   ↓
PROVISIONING
   ↓
STARTING
   ↓
READY_CHECK
   ↓
VALIDATING
   ↓
READY
```

Errores:

```text
RESOLUTION_FAILED
PROVISION_FAILED
START_FAILED
READINESS_FAILED
VALIDATION_FAILED
```

---

# 48. Provisioner

Podrá existir:

```php
interface TestEnvironmentProvisioner
{
    public function provision(
        ProvisioningPlan $plan
    ): ProvisioningResult;
}
```

---

# 49. Provisioning Plan

```text
ProvisioningPlan
├── environment profile
├── resource namespace
├── platform
├── version
├── topology
├── credentials
├── network
├── storage
├── native tools
└── resource limits
```

---

# 50. Provisioning ≠ readiness

Que un proceso haya iniciado no significa:

```text
database ready
```

---

# 51. Readiness System

Pipeline:

```text
Process Started
     ↓
Network Reachable
     ↓
Protocol Reachable
     ↓
Authentication
     ↓
Database Query
     ↓
Capability Prerequisites
     ↓
READY
```

---

# 52. Readiness Probe

Podrá existir:

```php
interface DatabaseTestReadinessProbe
{
    public function probe(
        TestEnvironment $environment
    ): ReadinessResult;
}
```

---

# 53. Readiness result

```text
READY
NOT_READY
DEGRADED
FAILED
UNKNOWN
```

---

# 54. UNKNOWN ≠ READY

Principio crítico:

```text
UNKNOWN
≠
READY
```

---

# 55. Bounded readiness

Todo readiness loop deberá tener:

```text
deadline
poll interval policy
cancellation
diagnostics
```

---

# 56. No infinite waits

Nunca:

```text
while (!ready()) {
}
```

sin límite.

---

# 57. Startup diagnostics

Si falla readiness deberá conservar:

```text
process status
last probe
connection error
server logs where safe
elapsed time
```

---

# 58. DBMS Platforms

La matriz base:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 59. Independent platform environments

Cada plataforma tendrá sus propios profiles.

---

# 60. Version Profiles

Ejemplo:

```text
postgresql-min-supported
postgresql-current
postgresql-latest-supported

mysql-min-supported
mysql-current

mariadb-min-supported
mariadb-current
```

---

# 61. Version aliases

Los aliases deberán resolverse a versiones concretas para reproducibilidad.

---

# 62. `latest` caveat

Un fallo bajo:

```text
latest
```

deberá registrar la versión real resuelta.

---

# 63. Immutable evidence

El reporte deberá guardar:

```text
resolved server version
```

no sólo:

```text
latest
```

---

# 64. Driver Matrix

Los profiles podrán seleccionar:

```text
PDO MySQL
PDO PgSQL
PDO SQLite
custom driver
```

según soporte.

---

# 65. Driver Version

Cuando sea detectable deberá formar parte del fingerprint.

---

# 66. Platform Version ≠ Driver Version

Siempre separados.

---

# 67. Extensions

El environment podrá requerir extensiones como:

```text
PostGIS
full-text plugins
custom DB extensions
```

---

# 68. Extension provisioning

Deberá ser explícito.

---

# 69. Extension presence

Será verificada mediante capabilities/evidence.

---

# 70. Capability Discovery

Después de readiness:

```text
Environment
    ↓
Capability Providers
    ↓
Evidence
    ↓
Capability Snapshot
```

---

# 71. Capability Snapshot

Deberá quedar asociado al entorno.

---

# 72. Snapshot scope

Podrá ser:

```text
server
database
connection
endpoint
```

según capability.

---

# 73. Capability cache

Podrá reutilizarse cuando sea seguro.

Pero:

```text
cached capability
≠
eternal truth
```

---

# 74. Test Capability Requirements

Un test podrá declarar:

```text
requires:
    transaction.savepoint
    query.returning.insert
```

---

# 75. Environment compatibility

Un profile será elegible sólo si puede satisfacer los requisitos.

---

# 76. UNKNOWN capability

La política podrá:

```text
probe
reject
require explicit override
```

según criticidad.

---

# 77. Database Provisioning

Una vez listo el servidor:

```text
Server
   ↓
Test Database Provisioner
   ↓
Test Database
```

---

# 78. TestDatabase

Podrá modelarse:

```php
final readonly class TestDatabase
{
    public function __construct(
        public TestDatabaseId $id,
        public LogicalDatabaseName $logicalName,
        public PhysicalDatabaseName $physicalName,
        public ConnectionConfiguration $connection,
        public TestResourceOwner $owner,
    ) {}
}
```

---

# 79. Database-per-suite

```text
Suite
 └── Database
      ├── Test A
      ├── Test B
      └── Test C
```

requiere reset entre tests.

---

# 80. Database-per-worker

```text
Worker A → Database A
Worker B → Database B
```

equilibra aislamiento y costo.

---

# 81. Database-per-test

Máximo aislamiento.

Mayor costo.

---

# 82. Schema-per-test

Adecuado en plataformas con schemas útiles para aislamiento.

---

# 83. SQLite file-per-test

Puede proporcionar aislamiento sencillo para SQLite.

---

# 84. SQLite in-memory

Puede utilizarse cuando la propiedad evaluada sea compatible.

---

# 85. In-memory caveat

Dependiendo del driver:

```text
connection A
```

y:

```text
connection B
```

pueden no observar la misma base en memoria.

Las pruebas concurrentes deberán considerar esto.

---

# 86. Schema Provisioning

Estrategias:

```text
MIGRATIONS
SCHEMA_BUILDER
SNAPSHOT
NATIVE_SCRIPT
EMPTY
CUSTOM
```

---

# 87. Migrations

Será default cuando la suite necesite representar el estado real de una aplicación.

---

# 88. Schema Builder

Podrá utilizarse para escenarios pequeños y específicos.

---

# 89. Snapshot

Puede acelerar suites grandes.

Pero deberá estar versionado y verificable.

---

# 90. Native Script

Permitido para probar introspection o características externas.

---

# 91. EMPTY

Necesario para tests de:

```text
schema creation
migration initialization
administration
```

---

# 92. Schema generation identity

El environment podrá registrar:

```text
SchemaGeneration
```

para caches, diagnostics y reproducibilidad.

---

# 93. Fixture Provisioning

Después del schema:

```text
Schema Ready
    ↓
Fixture Plan
    ↓
Factories / Fixtures / Seeders
    ↓
Known Dataset
```

---

# 94. Fixtures optional

No toda prueba requiere datos iniciales.

---

# 95. Fixture identity

Un scenario deberá poder identificar:

```text
fixture set
seed
version
```

---

# 96. Deterministic test data

El environment deberá propagar:

```text
TestSeed
```

cuando corresponda.

---

# 97. Test Clock

También podrá existir:

```text
TestClock
```

para datos temporales deterministas.

---

# 98. Credentials Architecture

```text
Environment
   │
   ├── Runtime Credential
   ├── Migration Credential
   ├── ReadOnly Credential
   ├── Backup Credential
   └── Admin Credential
```

---

# 99. Credential separation

No se deberá usar una cuenta superusuario para toda prueba.

---

# 100. Runtime role

Deberá representar los permisos de ejecución normal de una aplicación.

---

# 101. Migration role

Podrá disponer de DDL adicional.

---

# 102. Read-only role

Permitirá verificar restricciones reales.

---

# 103. Backup role

Permitirá probar mínimos privilegios del Backup System.

---

# 104. Admin role

Sólo para suites administrativas explícitas.

---

# 105. Credential storage

Las credenciales deberán obtenerse mediante:

```text
Secret Provider
Environment Secret
Ephemeral Provisioning
Secure CI Secret
```

---

# 106. Credentials in config files

No deberán almacenarse en repositorio.

---

# 107. Generated credentials

Los providers podrán generar credenciales efímeras.

---

# 108. Credential lifetime

Idealmente:

```text
Credential Lifetime
≤
Environment Lifetime
```

---

# 109. Redaction

Nunca deberán aparecer passwords en:

```text
logs
exceptions
test reports
telemetry
environment fingerprints
```

---

# 110. Isolation Architecture

```text
Shared Infrastructure
       ↓
Isolation Strategy
       ↓
Test Resource Scope
```

---

# 111. Isolation Levels

Podrán existir:

```text
SERVER
DATABASE
SCHEMA
TRANSACTION
TABLE_RESET
FIXTURE_RESET
FILE
CUSTOM
```

---

# 112. Isolation ≠ transaction only

Un transaction rollback no resetea necesariamente:

```text
sequences
DDL
session state
external side effects
connections
native tools
```

---

# 113. Reset Strategy

El environment deberá definir cómo regresar a estado conocido.

---

# 114. Reset strategies

```text
ROLLBACK
TRUNCATE
DROP_SCHEMA
DROP_DATABASE
RECREATE_DATABASE
RESTORE_SNAPSHOT
REAPPLY_MIGRATIONS
CUSTOM
```

---

# 115. Reset ≠ Cleanup

Reset:

```text
reuse environment
```

Cleanup:

```text
release test-owned resources
```

Destroy:

```text
remove environment infrastructure
```

---

# 116. Reset Planner

Podrá seleccionar estrategia según:

```text
test type
platform
isolation
resources
cost
capabilities
```

---

# 117. Fastest ≠ always correct

`TRUNCATE` puede ser rápido pero incorrecto si la prueba modificó:

```text
schema
roles
session settings
extensions
```

---

# 118. Dirty Environment

Si reset falla:

```text
Environment
→ DIRTY
```

---

# 119. Dirty environment reuse

Prohibido por defecto.

---

# 120. Environment quarantine

Un entorno cuyo estado ya no pueda demostrarse podrá pasar a:

```text
QUARANTINED
```

---

# 121. Environment Lifecycle

```text
DECLARED
   ↓
PROVISIONING
   ↓
STARTING
   ↓
VALIDATING
   ↓
READY
   ↓
IN_USE
   ↓
RESETTING
   ↓
READY
```

Finalización:

```text
READY/IN_USE
     ↓
DRAINING
     ↓
DESTROYING
     ↓
DESTROYED
```

Estados excepcionales:

```text
FAILED
DIRTY
QUARANTINED
UNKNOWN
```

---

# 122. UNKNOWN environment

Si VoltStack no puede determinar el estado:

```text
UNKNOWN
```

no podrá reutilizarse para pruebas destructivas.

---

# 123. Parallel Execution

```text
Test Run
├── Worker 1
├── Worker 2
├── Worker 3
└── Worker 4
```

---

# 124. Worker isolation

Cada worker deberá tener recursos que eviten colisiones.

---

# 125. Database-per-worker

Estrategia recomendada para muchas suites.

---

# 126. Schema-per-worker

Alternativa cuando sea apropiado.

---

# 127. Resource Namespace

Podrá producir:

```text
vs_<run>_w1
vs_<run>_w2
vs_<run>_w3
```

---

# 128. Global resources

Algunos recursos son server-wide:

```text
extensions
roles
server variables
replication configuration
```

y requieren coordinación especial.

---

# 129. Global lock

Podrá existir un mecanismo de exclusión para tests que modifiquen recursos globales.

---

# 130. Parallel-safe metadata

Un test podrá declarar:

```text
parallel_safe = true
```

o:

```text
exclusive_environment = true
```

---

# 131. Exclusive tests

Ejemplos:

```text
server restart
extension install
replication reconfiguration
global variable mutation
```

---

# 132. Port Management

Container providers deberán asignar puertos evitando colisiones.

---

# 133. Fixed port

No deberá asumirse salvo que el profile lo garantice.

---

# 134. Network Namespace

Cuando sea posible, los entornos podrán utilizar redes aisladas.

---

# 135. Hostname resolution

Los tests deberán recibir endpoints resueltos desde el Environment, no codificados.

---

# 136. Native Tools

Algunas pruebas requieren:

```text
pg_dump
pg_restore
pg_basebackup
mysql tools
sqlite utilities
```

---

# 137. Tool Requirement

El profile podrá declarar:

```text
NativeToolRequirement
```

---

# 138. Tool Discovery

El environment deberá comprobar:

```text
available
version
compatible
```

antes del test.

---

# 139. Tool Version ≠ Server Version

Ambas se registrarán separadamente.

---

# 140. Tool incompatibility

Deberá fallar durante environment validation, no a mitad de una prueba si puede detectarse previamente.

---

# 141. Filesystem Resources

Podrán existir directorios temporales:

```text
backup artifacts
restore files
SQLite databases
logs
exports
```

---

# 142. Temp directories

Deberán estar asociados a:

```text
TestRunId
```

---

# 143. Path safety

Nunca deberá ejecutarse cleanup recursivo sobre una ruta no demostrada como propiedad del test.

---

# 144. Storage resources

Object storage de testing podrá ser:

```text
local compatible service
remote dedicated bucket
ephemeral storage adapter
```

---

# 145. Bucket isolation

Podrá utilizar:

```text
bucket-per-run
prefix-per-run
```

según provider.

---

# 146. Storage cleanup

Deberá respetar ownership.

---

# 147. Backup environment

Los tests de backup podrán requerir:

```text
DBMS
Native Backup Tool
Artifact Storage
Backup Credential
```

---

# 148. Restore environment

Preferiblemente:

```text
Source DB
      ↓ backup
Artifact
      ↓ restore
Clean Target DB
```

---

# 149. Restore target isolation

Nunca deberá restaurarse sobre una base no autorizada.

---

# 150. Replication Environment

Topología:

```text
Writer
  │
  ├── Replica A
  └── Replica B
```

---

# 151. Replication readiness

No basta que todos los procesos estén vivos.

Deberá comprobarse:

```text
topology configured
replication active
positions observable
credentials valid
```

---

# 152. Replica lag test control

El environment podrá exponer mecanismos controlados para:

```text
pause replication
resume replication
observe lag
```

si la plataforma lo permite.

---

# 153. Sharding Environment

```text
Logical DB
├── Shard 1
└── Shard 2
```

cada shard podrá ser una instancia o database separada.

---

# 154. Topology descriptor

Podrá modelarse:

```php
final readonly class TestDatabaseTopology
{
    public function __construct(
        public TopologyType $type,
        public array $endpoints,
        public TopologyGeneration $generation,
    ) {}
}
```

---

# 155. Topology Generation

Cambios de failover o reconfiguración deberán producir nueva generación.

---

# 156. Failure Injection Environment

Algunas suites podrán requerir:

```text
network proxy
process controller
latency injector
connection killer
disk limiter
```

---

# 157. Failure injector optionality

No formará parte obligatoria del runtime de producción.

Será infraestructura de testing.

---

# 158. Network Failure

Podrán simularse:

```text
drop
reset
latency
timeout
partition
```

de forma controlada.

---

# 159. Process Failure

El environment podrá:

```text
stop DBMS
kill connection
restart server
```

cuando la suite esté autorizada.

---

# 160. Storage Failure

Podrán existir escenarios controlados de:

```text
quota exhausted
write denied
missing artifact
corruption
```

---

# 161. Failure isolation

Los fallos inyectados no deberán afectar otros test runs.

---

# 162. Environment Fingerprint

Cada entorno deberá poder producir un fingerprint.

```text
EnvironmentFingerprint
├── profile id
├── provider
├── platform
├── server version
├── driver
├── driver version
├── topology
├── relevant extensions
├── capability fingerprint
├── schema generation
├── relevant configuration
└── runtime information
```

---

# 163. Fingerprint ≠ secrets

No contendrá:

```text
passwords
tokens
private keys
full secret DSNs
```

---

# 164. Stable fingerprint

Deberá ser suficientemente determinista para comparar entornos equivalentes.

---

# 165. Relevant configuration

No se incluirá toda la configuración del servidor.

Sólo propiedades relevantes para el test.

---

# 166. Reproducibility Bundle

Ante fallo podrá generarse:

```text
TestFailureBundle
├── test id
├── test seed
├── environment fingerprint
├── capability snapshot
├── schema snapshot
├── safe logs
└── failure diagnostics
```

---

# 167. Bundle portability

Deberá ser posible utilizarlo para reconstruir un entorno equivalente cuando sea razonable.

---

# 168. Reproduction ≠ exact hardware clone

No toda propiedad requiere CPU/hardware idéntico.

Performance Testing podrá exigir mayor control.

---

# 169. Environment Diagnostics

El environment deberá exponer información estructurada:

```text
status
readiness
resource state
database state
topology
capabilities
tool availability
cleanup state
```

---

# 170. Diagnostic command

Conceptualmente:

```text
volt database:test:environment:diagnose
```

---

# 171. Environment Listing

```text
volt database:test:environment:list
```

podrá mostrar:

```text
profile
platform
version
provider
availability
status
```

---

# 172. Provision command

Conceptualmente:

```text
volt database:test:environment:up postgresql-17
```

---

# 173. Destroy command

```text
volt database:test:environment:down postgresql-17
```

con guards apropiados.

---

# 174. Reset command

```text
volt database:test:environment:reset postgresql-17
```

---

# 175. CLI ≠ core semantics

Los comandos serán interfaces sobre servicios internos.

---

# 176. Local Developer Workflow

```text
Developer
    ↓
Select Profile
    ↓
Environment Up
    ↓
Run Integration Tests
    ↓
Environment Reuse
    ↓
Reset
    ↓
Environment Down
```

---

# 177. Reusable local environments

Podrán mantenerse activos para reducir tiempo de desarrollo.

---

# 178. Reuse validation

Antes de reutilizar:

```text
identity
health
version
capabilities
cleanliness
```

deberán verificarse.

---

# 179. Stale local environment

Si no coincide con el profile:

```text
STALE
```

y deberá reconstruirse o reconciliarse.

---

# 180. CI Workflow

```text
CI Job
  ↓
Resolve Environment Profile
  ↓
Provision
  ↓
Readiness
  ↓
Validate
  ↓
Run Tests
  ↓
Collect Evidence
  ↓
Cleanup
  ↓
Destroy
```

---

# 181. CI environment ownership

Todo recurso creado deberá asociarse al job/run correspondiente.

---

# 182. CI interruption

Los providers deberán facilitar posterior garbage collection.

---

# 183. Garbage Collection

Responsabilidad:

```text
find orphaned test resources
       ↓
verify ownership
       ↓
verify expiry
       ↓
safe delete
```

---

# 184. GC ≠ blind prefix deletion

Nunca:

```text
delete everything matching "test*"
```

sin ownership evidence.

---

# 185. Resource Ownership Record

Podrá contener:

```text
resource id
test run id
created at
owner
expiry
provider
environment profile
```

---

# 186. Orphan

Un recurso será orphan cuando:

```text
owner run no longer exists
+
resource lifetime exceeded
+
ownership can be demonstrated
```

---

# 187. Unknown ownership

```text
UNKNOWN ownership
→ do not destruct automatically
```

---

# 188. TTL

Podrá ayudar a identificar recursos antiguos.

Pero:

```text
TTL ≠ permission to delete unknown resource
```

---

# 189. Cleanup Planner

Podrá ordenar recursos según dependencias.

Ejemplo:

```text
Test DB
   ↓
Credentials
   ↓
Storage
   ↓
Network
   ↓
DBMS
```

dependiendo del provider.

---

# 190. Cleanup idempotency

Las operaciones de cleanup deberán ser idempotentes cuando sea posible.

---

# 191. Already absent

Un recurso ya eliminado podrá considerarse:

```text
CLEAN
```

si su ausencia puede demostrarse.

---

# 192. Unknown deletion result

Si la eliminación se envió pero se pierde confirmación:

```text
UNKNOWN
```

y deberá reconciliarse.

---

# 193. Environment Recovery

Si un test deja un entorno inconsistente:

```text
Detect
  ↓
Quarantine
  ↓
Diagnose
  ↓
Reset or Destroy
  ↓
Revalidate
```

---

# 194. Reset success

Sólo volverá a:

```text
READY
```

después de validación.

---

# 195. Restart ≠ reset

Reiniciar DBMS no garantiza que:

```text
schema
data
roles
files
```

hayan regresado al estado inicial.

---

# 196. Snapshot Reset

Un provider podrá soportar snapshots de infraestructura.

---

# 197. Snapshot semantics

Deberá quedar claro qué incluye:

```text
database storage
configuration
credentials
filesystem
```

---

# 198. Snapshot compatibility

No deberá restaurarse un snapshot incompatible con:

```text
DBMS version
storage format
environment generation
```

---

# 199. Persistent Runtime Test Environment

El entorno deberá poder ejecutar múltiples operaciones sobre el mismo proceso de aplicación.

```text
Worker
 ├── Request A
 ├── Request B
 ├── Job C
 └── Request D
```

---

# 200. FrankenPHP Environment

Como runtime principal:

```text
FrankenPHP worker
+
real database
+
connection reuse
```

deberá formar parte de la estrategia de pruebas.

---

# 201. Runtime environment separation

El runtime de aplicación y DBMS podrán provisionarse por providers distintos.

---

# 202. RoadRunner Environment

Adapter opcional.

---

# 203. OpenSwoole Environment

Deberá permitir concurrencia/coroutines reales cuando la suite lo requiera.

---

# 204. Worker lifecycle evidence

Se deberán poder observar:

```text
worker start
operation start
operation end
state reset
worker shutdown
```

---

# 205. Resource Budgets

Cada environment profile podrá definir:

```text
CPU
memory
disk
network
connections
duration
workers
storage
```

---

# 206. Resource limits ≠ benchmark configuration

Los límites protegen CI y detectan uso anormal.

Los benchmarks tendrán configuración propia.

---

# 207. Connection Budget

Podrá impedir que tests defectuosos agoten el servidor.

---

# 208. Storage Budget

Importante para:

```text
backup
restore
large datasets
logs
```

---

# 209. Timeout Budget

Podrán existir:

```text
provision timeout
readiness timeout
test timeout
reset timeout
cleanup timeout
destroy timeout
```

---

# 210. Timeout classification

Cada timeout deberá identificar la fase.

---

# 211. Environment Events

Podrán emitirse eventos:

```text
EnvironmentResolving
EnvironmentProvisioning
EnvironmentReady
EnvironmentResetting
EnvironmentQuarantined
EnvironmentDestroying
EnvironmentDestroyed
```

---

# 212. Events ≠ lifecycle authority

El estado real seguirá perteneciendo al Environment Manager.

---

# 213. Telemetry

Métricas:

```text
provision duration
readiness duration
reset duration
cleanup duration
environment failures
resource usage
```

---

# 214. Bounded cardinality

No utilizar:

```text
TestRunId
```

como label de métrica global de alta cardinalidad.

---

# 215. Logs

Podrán incluir IDs para correlación.

---

# 216. Audit

Operaciones especialmente destructivas podrán registrarse:

```text
environment
operation
actor/process
time
decision
result
```

---

# 217. Environment Security

Principios:

```text
least privilege
network isolation
ephemeral credentials
secret redaction
resource ownership
destructive guards
```

---

# 218. External network exposure

Los DBMS de test no deberán exponerse públicamente salvo necesidad explícita.

---

# 219. Default binding

Preferir:

```text
local/private network
```

---

# 220. TLS

Los entornos que prueben connection security deberán poder habilitar:

```text
TLS
certificate verification
invalid certificate scenarios
```

---

# 221. Security test profiles

Podrán existir perfiles especiales:

```text
postgresql-tls-valid
postgresql-tls-invalid-cert
postgresql-readonly
```

---

# 222. Configuration Override

Tests podrán modificar configuración sólo mediante mecanismos explícitos.

---

# 223. Immutable profile

El profile base no deberá mutarse globalmente durante la suite.

---

# 224. Derived Profile

Preferible:

```text
Base Profile
    ↓
Derived Test Profile
```

---

# 225. Runtime overrides

Deberán quedar asociados al Test Context.

---

# 226. Environment Capability Contract

Un entorno deberá declarar qué puede proporcionar:

```text
single database
multiple databases
replication
sharding
native tools
network fault injection
server restart
filesystem access
TLS
extensions
```

---

# 227. Environment capability ≠ database capability

Ejemplo:

```text
PostgreSQL supports replication
```

pero:

```text
current test environment
```

puede no estar configurado con replicas.

---

# 228. Two capability layers

```text
Database Capability
+
Environment Capability
```

ambas pueden ser necesarias.

---

# 229. Example

Para probar failover:

```text
DB capability:
replication supported

Environment capability:
multi-node topology controllable
```

---

# 230. Environment Requirement

Podrá modelarse:

```php
final readonly class EnvironmentRequirement
{
    public function __construct(
        public EnvironmentFeature $feature,
        public RequirementLevel $level,
    ) {}
}
```

---

# 231. Requirement levels

```text
REQUIRED
PREFERRED
OPTIONAL
```

---

# 232. Missing required feature

La suite no deberá ejecutarse.

---

# 233. Missing preferred feature

Podrá elegir estrategia alternativa.

---

# 234. Environment Selection Algorithm

```text
Test Requirements
       ↓
Candidate Profiles
       ↓
Platform Match
       ↓
Driver Match
       ↓
DB Capability Match
       ↓
Environment Capability Match
       ↓
Safety Match
       ↓
Resource Match
       ↓
Selected Profile
```

---

# 235. Deterministic selection

Dados:

```text
same requirements
same available profiles
same policy
```

deberá elegirse el mismo profile.

---

# 236. Selection explainability

El runner deberá poder explicar:

```text
why profile selected
why others rejected
```

---

# 237. Test Environment Manifest

Podrá producirse:

```json
{
  "profile": "postgresql-17-integration",
  "platform": "postgresql",
  "version": "17.x",
  "driver": "pdo_pgsql",
  "isolation": "database_per_worker",
  "topology": "single",
  "status": "ready"
}
```

sin secretos.

---

# 238. Manifest use

Podrá adjuntarse a CI artifacts.

---

# 239. Manifest ≠ connection secret

No contendrá credenciales.

---

# 240. Test Environment Cache

Algunos recursos costosos podrán reutilizarse entre suites.

---

# 241. Cache eligibility

Sólo si:

```text
environment healthy
+
identity valid
+
profile compatible
+
clean state demonstrable
```

---

# 242. Environment reuse ≠ state reuse

La infraestructura puede reutilizarse.

El estado mutable del test deberá resetearse.

---

# 243. Server reuse

Permitido.

---

# 244. Database reuse

Permitido sólo con reset válido.

---

# 245. EntityManager reuse

No pertenece al Environment y no deberá sobrevivir scopes.

---

# 246. Connection reuse

Podrá ocurrir según Connection System, pero requerirá reset de sesión.

---

# 247. Environment state vs application state

```text
Environment State
≠
DatabaseContext
≠
EntityManager State
≠
TestContext
```

---

# 248. Test Environment Manager

Coordinador principal:

```php
interface DatabaseTestEnvironmentManager
{
    public function resolve(
        TestEnvironmentRequest $request
    ): ResolvedEnvironmentProfile;

    public function provision(
        ResolvedEnvironmentProfile $profile
    ): TestEnvironment;

    public function reset(
        TestEnvironment $environment
    ): ResetResult;

    public function destroy(
        TestEnvironment $environment
    ): DestructionResult;
}
```

---

# 249. Manager ≠ God Object

Deberá delegar:

```text
resolution
provisioning
safety
readiness
database creation
reset
cleanup
diagnostics
```

a servicios especializados.

---

# 250. Proposed namespace

```text
src/Quantum/Database/Testing/Environment/
├── Contract/
│   ├── TestEnvironmentProvider.php
│   ├── TestEnvironmentProvisioner.php
│   ├── TestEnvironmentGuard.php
│   ├── TestEnvironmentResetter.php
│   └── TestEnvironmentDestroyer.php
│
├── Model/
│   ├── TestEnvironment.php
│   ├── TestEnvironmentIdentity.php
│   ├── TestEnvironmentRequest.php
│   ├── TestEnvironmentProfile.php
│   ├── EnvironmentFingerprint.php
│   └── EnvironmentManifest.php
│
├── Manager/
│   └── DatabaseTestEnvironmentManager.php
│
├── Resolution/
│   ├── EnvironmentResolver.php
│   ├── EnvironmentRequirement.php
│   └── EnvironmentSelectionPolicy.php
│
├── Provider/
│   ├── Local/
│   ├── Container/
│   ├── Remote/
│   └── External/
│
├── Provisioning/
│   ├── ProvisioningPlan.php
│   └── ProvisioningPlanner.php
│
├── Safety/
│   ├── EnvironmentSafetyPolicy.php
│   ├── EnvironmentGuard.php
│   └── ResourceOwnershipVerifier.php
│
├── Readiness/
│   ├── ReadinessProbe.php
│   └── ReadinessCoordinator.php
│
├── Database/
│   ├── TestDatabase.php
│   ├── TestDatabaseProvisioner.php
│   └── TestDatabaseNameGenerator.php
│
├── Credential/
│   ├── TestCredentialProvider.php
│   └── TestRoleProfile.php
│
├── Isolation/
│   ├── IsolationStrategy.php
│   └── IsolationPlanner.php
│
├── Reset/
│   ├── ResetStrategy.php
│   └── ResetPlanner.php
│
├── Topology/
│   ├── TestDatabaseTopology.php
│   └── TopologyGeneration.php
│
├── Resource/
│   ├── TestResource.php
│   ├── TestResourceOwner.php
│   └── ResourceNamespace.php
│
├── Cleanup/
│   ├── CleanupPlanner.php
│   └── GarbageCollector.php
│
├── Diagnostics/
│   └── EnvironmentDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 251. Exception taxonomy

```text
DatabaseTestEnvironmentException
├── EnvironmentResolutionException
├── EnvironmentProvisioningException
├── EnvironmentStartupException
├── EnvironmentReadinessException
├── EnvironmentValidationException
├── UnsafeEnvironmentException
├── EnvironmentCapabilityException
├── DatabaseProvisioningException
├── CredentialProvisioningException
├── EnvironmentResetException
├── EnvironmentCleanupException
├── EnvironmentDestructionException
├── ResourceOwnershipException
├── EnvironmentTimeoutException
└── UnknownEnvironmentStateException
```

---

# 252. UnsafeEnvironmentException

Deberá ser especialmente difícil de ignorar.

---

# 253. Force option

Si existe:

```text
--force
```

no deberá desactivar indiscriminadamente protecciones contra producción.

---

# 254. Force semantics

Podrá omitir una confirmación interactiva.

No deberá convertir:

```text
proven production environment
```

en:

```text
safe test environment
```

---

# 255. Invariantes

## DB-ENV-001

Un Test Environment tendrá identidad explícita.

## DB-ENV-002

Environment no será Database.

## DB-ENV-003

Environment no será Fixture.

## DB-ENV-004

Environment no será TestContext.

## DB-ENV-005

Profile no será Environment.

## DB-ENV-006

Resolution no será Provisioning.

## DB-ENV-007

Provisioning no será Readiness.

## DB-ENV-008

Readiness no será Safety Validation.

## DB-ENV-009

Docker será adapter, no arquitectura.

## DB-ENV-010

Podman podrá coexistir con Docker.

## DB-ENV-011

Localhost no implicará seguridad.

## DB-ENV-012

Remote no implicará producción.

## DB-ENV-013

Cada Test Run tendrá identidad.

## DB-ENV-014

Workers paralelos tendrán identidad.

## DB-ENV-015

Recursos temporales tendrán ownership.

## DB-ENV-016

Naming convention no será única protección.

## DB-ENV-017

Operaciones destructivas pasarán por Environment Guard.

## DB-ENV-018

Mayor destructividad requerirá mayor evidencia.

## DB-ENV-019

`APP_ENV=test` no será prueba suficiente.

## DB-ENV-020

`APP_ENV=production` bloqueará testing destructivo por defecto.

## DB-ENV-021

DROP DATABASE tendrá guard fuerte.

## DB-ENV-022

UNKNOWN safety no será ALLOW.

## DB-ENV-023

Proceso iniciado no significará READY.

## DB-ENV-024

Readiness tendrá deadline.

## DB-ENV-025

UNKNOWN readiness no será READY.

## DB-ENV-026

Readiness failure tendrá diagnostics.

## DB-ENV-027

MySQL será distinto de MariaDB.

## DB-ENV-028

SQLite será plataforma propia.

## DB-ENV-029

Version aliases se resolverán a versiones concretas.

## DB-ENV-030

`latest` se registrará como versión resuelta.

## DB-ENV-031

Platform version será distinta de Driver version.

## DB-ENV-032

Extensions serán explícitas.

## DB-ENV-033

Capability discovery ocurrirá sobre environment validado.

## DB-ENV-034

Capability snapshot tendrá scope.

## DB-ENV-035

Cached capabilities no serán verdad eterna.

## DB-ENV-036

UNKNOWN capability no será SUPPORTED.

## DB-ENV-037

Test database tendrá ownership.

## DB-ENV-038

Database-per-suite requerirá reset.

## DB-ENV-039

Database-per-worker podrá soportar paralelismo.

## DB-ENV-040

Database-per-test priorizará aislamiento sobre costo.

## DB-ENV-041

SQLite in-memory tendrá semántica de conexión explícita.

## DB-ENV-042

Schema provisioning será estrategia explícita.

## DB-ENV-043

Migration tests no serán inicializados ocultando migrations.

## DB-ENV-044

Native schemas podrán probar introspection.

## DB-ENV-045

Fixture provisioning será posterior al schema.

## DB-ENV-046

Fixtures serán opcionales.

## DB-ENV-047

Test data podrá ser determinista.

## DB-ENV-048

TestClock podrá ser scoped.

## DB-ENV-049

Runtime credential será distinta de Admin credential.

## DB-ENV-050

Superuser no será credential universal.

## DB-ENV-051

Secrets no estarán en repositorio.

## DB-ENV-052

Ephemeral credentials serán preferidas donde sea viable.

## DB-ENV-053

Credentials no aparecerán en diagnostics.

## DB-ENV-054

Isolation no será sinónimo de transaction rollback.

## DB-ENV-055

Reset será distinto de Cleanup.

## DB-ENV-056

Cleanup será distinto de Destroy.

## DB-ENV-057

Reset strategy dependerá de la propiedad modificada.

## DB-ENV-058

TRUNCATE no será reset universal.

## DB-ENV-059

Reset failure marcará entorno DIRTY.

## DB-ENV-060

DIRTY no será reutilizable por defecto.

## DB-ENV-061

UNKNOWN environment no será reutilizable destructivamente.

## DB-ENV-062

Environment podrá ser QUARANTINED.

## DB-ENV-063

Parallel workers no compartirán recursos mutables accidentalmente.

## DB-ENV-064

Server-global tests podrán requerir exclusividad.

## DB-ENV-065

Puertos no se asumirán globalmente fijos.

## DB-ENV-066

Endpoints vendrán del Environment.

## DB-ENV-067

Native tools tendrán discovery.

## DB-ENV-068

Tool version será distinta de Server version.

## DB-ENV-069

Incompatibilidad detectable fallará antes del test.

## DB-ENV-070

Temporary paths tendrán ownership.

## DB-ENV-071

Recursive cleanup requerirá path ownership demostrado.

## DB-ENV-072

Object storage test resources tendrán namespace.

## DB-ENV-073

Restore targets estarán protegidos.

## DB-ENV-074

Replication readiness verificará más que process liveness.

## DB-ENV-075

Replica lag control será environment capability.

## DB-ENV-076

Sharding topology será explícita.

## DB-ENV-077

Topology changes tendrán generation.

## DB-ENV-078

Failure injection será infraestructura de testing.

## DB-ENV-079

Failure injection no contaminará otros runs.

## DB-ENV-080

Environment fingerprint no contendrá secrets.

## DB-ENV-081

Fingerprint registrará versión realmente resuelta.

## DB-ENV-082

Failure bundle facilitará reproducción.

## DB-ENV-083

Reproduction no requerirá hardware idéntico salvo propiedad relevante.

## DB-ENV-084

Environment diagnostics serán estructurados.

## DB-ENV-085

CLI será fachada sobre servicios.

## DB-ENV-086

Entornos locales podrán reutilizarse.

## DB-ENV-087

Reuse requerirá revalidation.

## DB-ENV-088

Stale environment no será tratado como compatible.

## DB-ENV-089

CI resources tendrán ownership.

## DB-ENV-090

Interrupted CI podrá dejar recursos reconciliables.

## DB-ENV-091

GC no eliminará recursos por prefijo solamente.

## DB-ENV-092

UNKNOWN ownership no autorizará destrucción.

## DB-ENV-093

TTL no será prueba de ownership.

## DB-ENV-094

Cleanup será idempotente cuando sea posible.

## DB-ENV-095

Unknown deletion outcome requerirá reconciliation.

## DB-ENV-096

Restart no será reset.

## DB-ENV-097

Snapshot semantics serán explícitas.

## DB-ENV-098

Snapshots incompatibles serán rechazados.

## DB-ENV-099

Persistent runtime environments soportarán múltiples operations.

## DB-ENV-100

FrankenPHP tendrá environment profile oficial.

## DB-ENV-101

RoadRunner podrá ser adapter opcional.

## DB-ENV-102

OpenSwoole podrá probar coroutine isolation.

## DB-ENV-103

Runtime y DBMS podrán tener providers distintos.

## DB-ENV-104

Resource budgets serán explícitos.

## DB-ENV-105

Provision timeout será distinto de Test timeout.

## DB-ENV-106

Cleanup timeout será distinto de Destroy timeout.

## DB-ENV-107

Environment events no serán lifecycle authority.

## DB-ENV-108

Telemetry tendrá bounded cardinality.

## DB-ENV-109

DBMS test no se expondrá públicamente por defecto.

## DB-ENV-110

TLS test profiles podrán existir.

## DB-ENV-111

Base profile no se mutará globalmente durante tests.

## DB-ENV-112

Runtime overrides serán scoped.

## DB-ENV-113

Database Capability será distinta de Environment Capability.

## DB-ENV-114

Failover testing requerirá DB capability y environment capability.

## DB-ENV-115

Environment requirements serán explícitos.

## DB-ENV-116

Selection será determinista.

## DB-ENV-117

Selection será explainable.

## DB-ENV-118

Environment manifest no contendrá secrets.

## DB-ENV-119

Environment infrastructure podrá reutilizarse.

## DB-ENV-120

Mutable test state deberá resetearse antes de reuse.

## DB-ENV-121

EntityManager no será environment state.

## DB-ENV-122

DatabaseContext no será environment state.

## DB-ENV-123

Connection reuse requerirá session reset.

## DB-ENV-124

Environment Manager no será God Object.

## DB-ENV-125

UnsafeEnvironmentException no deberá ignorarse silenciosamente.

## DB-ENV-126

`--force` no deshabilitará protección contra producción.

## DB-ENV-127

Production DB nunca será target implícito de testing.

## DB-ENV-128

Production backup nunca será fixture implícita.

## DB-ENV-129

Un entorno deberá poder demostrar quién posee sus recursos.

## DB-ENV-130

No se destruirá un recurso cuya propiedad sea desconocida.

## DB-ENV-131

Un environment READY deberá haber pasado validation.

## DB-ENV-132

Un environment reset deberá revalidarse antes de READY.

## DB-ENV-133

Environment failure será distinto de Test failure.

## DB-ENV-134

Provisioning failure será distinto de assertion failure.

## DB-ENV-135

Environment capability missing será distinto de DB capability missing.

## DB-ENV-136

Una capability soportada por el DBMS no implicará que la infraestructura de test pueda ejercerla.

## DB-ENV-137

Los perfiles serán declarativos.

## DB-ENV-138

La lectura de configuración no provisionará infraestructura.

## DB-ENV-139

Environment Registry podrá congelarse tras bootstrap.

## DB-ENV-140

Los providers no podrán alterar los contratos semánticos del test.

## DB-ENV-141

La misma suite podrá ejecutarse sobre providers diferentes.

## DB-ENV-142

El origen de la infraestructura no determinará correctness.

## DB-ENV-143

Toda espera de infraestructura tendrá deadline.

## DB-ENV-144

Toda operación destructiva será auditable cuando la política lo requiera.

## DB-ENV-145

Environment diagnostics preservará incertidumbre.

## DB-ENV-146

UNKNOWN no será convertido a READY por conveniencia.

## DB-ENV-147

UNKNOWN no será convertido a DESTROYED sin evidencia.

## DB-ENV-148

Los recursos externos podrán reconciliarse después de fallos.

## DB-ENV-149

TestRunId no deberá utilizarse como métrica global de alta cardinalidad.

## DB-ENV-150

VoltStack deberá favorecer entornos reproducibles sobre configuraciones implícitas.

---

# 256. Anti-patrones

## 256.1 Usar la conexión default de la aplicación

```text
Database::connection()
```

sin verificar que sea de testing.

---

## 256.2 Confiar sólo en `APP_ENV=test`

Insuficiente para proteger datos.

---

## 256.3 DROP por prefijo

```text
DROP all databases starting with test_
```

sin ownership.

---

## 256.4 Root everywhere

Oculta errores de permisos y aumenta riesgo.

---

## 256.5 Un único DB para CI paralelo

Produce contaminación y flakiness.

---

## 256.6 Docker hardcoded en Database Testing

Impediría otros providers.

---

## 256.7 `sleep(10)` para readiness

No proporciona evidencia estructurada.

---

## 256.8 Considerar process running como READY

Incorrecto.

---

## 256.9 Reutilizar entorno DIRTY

Puede producir falsos resultados.

---

## 256.10 Ignorar cleanup failure

Genera deuda de infraestructura.

---

## 256.11 Destruir recursos UNKNOWN

Riesgo crítico.

---

## 256.12 Usar backup productivo como fixture

Riesgo de privacidad y seguridad.

---

## 256.13 Mutar profile global

Produce tests dependientes del orden.

---

## 256.14 Confundir DB capability con environment capability

Que PostgreSQL soporte replication no significa que el test tenga un cluster.

---

## 256.15 `--force` como bypass universal

No permitido.

---

# 257. Modelo formal

Sea:

```text
P = Environment Profile
R = Test Requirements
I = Infrastructure Provider
S = Safety Policy
```

La resolución produce:

```text
E_candidate = Resolve(P, R, I)
```

La validación deberá satisfacer:

```text
Compatible(E_candidate, R)
∧
Safe(E_candidate, S)
∧
Ready(E_candidate)
```

Entonces:

```text
E = READY
```

---

# 258. Regla de readiness

```text
READY(E)
=
Reachable(E)
∧
Authenticated(E)
∧
DatabaseOperational(E)
∧
RequiredEnvironmentCapabilities(E)
∧
SafetyValidated(E)
```

según el profile correspondiente.

---

# 259. Regla de destrucción

Un recurso `r` podrá destruirse automáticamente sólo cuando:

```text
OwnedBy(r, TestRun)
∧
Authorized(Operation)
∧
SafeTarget(r)
```

puedan demostrarse.

Si:

```text
Ownership(r) = UNKNOWN
```

entonces:

```text
AutomaticDelete(r) = REJECT
```

---

# 260. Regla de reutilización

```text
Reusable(E)
=
Healthy(E)
∧
Compatible(E, Profile)
∧
IdentityValid(E)
∧
Clean(E)
∧
NotQuarantined(E)
```

---

# 261. Relación con Integration Testing

```text
Integration Test
      ↓
Environment Requirement
      ↓
Database Test Environment System
      ↓
Real Controlled Infrastructure
      ↓
Integration Evidence
```

---

# 262. Relación con Unit Testing

Unit Testing normalmente no necesitará Test Environment real.

Podrá utilizar:

```text
Fake Driver
Fake Connection
Fake Clock
Synthetic Capability Snapshot
```

manteniendo las pruebas rápidas y deterministas.

---

# 263. Relación con Transactional Testing

El siguiente documento utilizará este sistema para construir entornos donde puedan probarse correctamente:

```text
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
ISOLATION
LOCKING
DEADLOCK
RETRY
UNKNOWN OUTCOME
```

sin que el propio mecanismo de aislamiento del test invalide las propiedades evaluadas.

---

# 264. Arquitectura consolidada

```text
                     TEST DEFINITION
                           │
                           ▼
                 Environment Request
                           │
                           ▼
                  Environment Resolver
                           │
                           ▼
                    Profile Registry
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
        Local          Container          Remote
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                      Provisioner
                           │
                           ▼
                     Safety Guard
                           │
                           ▼
                      Readiness
                           │
                           ▼
                  Capability Discovery
                           │
                           ▼
                  Environment READY
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        Credentials     Database      Topology
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                   Schema Provisioning
                           │
                           ▼
                    Fixture Loading
                           │
                           ▼
                       TEST RUN
                           │
                           ▼
                         Reset
                           │
                   ┌───────┴───────┐
                   ▼               ▼
                 Reuse           Cleanup
                                   │
                                   ▼
                                 Destroy
                                   │
                                   ▼
                             Evidence / GC
```

---

# 265. Regla final

> **VoltStack nunca deberá ejecutar una prueba destructiva simplemente porque existe una conexión disponible. Antes de utilizar una infraestructura como entorno de testing deberá resolver su perfil, demostrar su identidad, validar su seguridad, comprobar su disponibilidad y establecer quién posee los recursos que serán modificados.**

En forma compacta:

```text
Connected
≠
Ready
```

```text
Ready
≠
Safe
```

```text
Named "test"
≠
Owned by test
```

```text
Old resource
≠
Safe to delete
```

y:

```text
Valid Test Environment
=
Known Identity
+
Known Ownership
+
Required Capabilities
+
Readiness
+
Isolation
+
Safety
+
Controlled Lifecycle
```

---

# 266. Siguiente documento

```text
287_DATABASE_TRANSACTIONAL_TESTING_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura especializada para probar:

```text
Transactional Testing System
│
├── Transaction Test Model
├── Transaction Ownership
├── Test Transaction Isolation
├── BEGIN / COMMIT / ROLLBACK
├── Savepoints
├── Nested Transactions
├── Isolation Levels
├── Multiple Connections
├── Visibility Rules
├── Locking
├── Lock Timeouts
├── Deadlocks
├── Optimistic Locking
├── Pessimistic Locking
├── Transaction Retry
├── Transaction Events
├── ORM + Transaction Interaction
├── Flush vs Commit
├── Rollback vs Object State
├── Unknown Commit Outcome
├── Failure Injection
├── Concurrent Transaction Scenarios
├── Transactional Fixture Strategies
├── Cleanup
├── Persistent Runtime Isolation
└── Cross-Platform Transaction Contracts
```

manteniendo como principio:

> **Una prueba transaccional deberá controlar explícitamente quién posee la transacción, qué conexiones participan y qué resultado puede demostrarse; el mecanismo utilizado para aislar el propio test nunca deberá ocultar o alterar la semántica transaccional que se intenta verificar.**