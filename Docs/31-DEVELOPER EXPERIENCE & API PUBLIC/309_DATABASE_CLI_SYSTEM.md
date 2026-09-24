# 309_DATABASE_CLI_SYSTEM.md

# VoltStack Quantum Database
## Database CLI System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 309 — Database CLI System  
**Bloque:** 31 — Developer Experience / Public API  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md`  
**Siguiente documento:** `310_DATABASE_CODE_GENERATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial de la interfaz de línea de comandos de `VoltStack/Quantum/Database`.

El Database CLI proporcionará una superficie operativa y de desarrollo para interactuar con:

- conexiones;
- consultas;
- esquema;
- migraciones;
- seeders;
- fixtures;
- capabilities;
- diagnóstico;
- health checks;
- mantenimiento;
- backups;
- restore;
- administración;
- benchmarking;
- generación de código;
- inspección del ORM.

La regla arquitectónica principal será:

> **La CLI de Database será una superficie de interacción sobre servicios existentes de `VoltStack/Quantum/Database`; nunca será una segunda implementación de Query, Schema, Migration, ORM, Backup, Diagnostics o Administration.**

Por tanto:

```text
CLI
≠
Database Engine
```

y:

```text
CLI Command
→ Application Service / Database Service
→ Database Architecture
```

nunca:

```text
CLI Command
→ SQL construido manualmente
→ PDO
```

---

# 2. Objetivos

Database CLI deberá ofrecer:

1. comandos coherentes;
2. UX consistente;
3. ejecución segura;
4. integración con configuración;
5. resolución automática de conexiones;
6. selección explícita de tenant/shard cuando corresponda;
7. dry-run;
8. explain;
9. confirmación de operaciones peligrosas;
10. salida humana;
11. salida machine-readable;
12. integración con Diagnostics;
13. integración con Telemetry;
14. códigos de salida estables;
15. ejecución interactiva y no interactiva;
16. scripting;
17. extensibilidad;
18. soporte para plugins;
19. testing;
20. seguridad operacional.

---

# 3. Principios

La CLI seguirá los siguientes principios:

```text
Thin Commands
Explicit Intent
Safe Defaults
Structured Output
Deterministic Behavior
Capability Awareness
Context Awareness
Automation Friendly
Security First
No Hidden Database Semantics
```

---

# 4. Arquitectura general

```text
Developer / Operator / CI
          │
          ▼
     VoltStack CLI
          │
          ▼
 Database CLI Router
          │
          ▼
   Database Commands
          │
          ▼
 Command Input Model
          │
          ▼
 Context Resolution
          │
          ├── Configuration
          ├── Connection
          ├── Environment
          ├── Tenant
          ├── Shard
          └── Runtime
          │
          ▼
 Authorization / Safety
          │
          ▼
 Database Application Services
          │
   ┌──────┼─────────┬─────────┐
   ▼      ▼         ▼         ▼
 Query  Schema   Migration    ORM
   │      │         │         │
   └──────┼─────────┼─────────┘
          │
          ├── Backup
          ├── Restore
          ├── Health
          ├── Diagnostics
          ├── Maintenance
          └── Administration
          │
          ▼
 Command Result
          │
          ▼
 Output Renderer
   ┌──────┼──────────┐
   ▼      ▼          ▼
Human   JSON       Quiet
```

---

# 5. CLI ≠ Engine

La separación será estricta.

Por ejemplo:

```text
database:migrate
```

no implementará el algoritmo de migraciones.

Utilizará:

```text
MigrationSystem
MigrationPlanner
MigrationSafetySystem
MigrationExecutor
MigrationRepository
```

De manera similar:

```text
database:backup
```

utilizará:

```text
BackupSystem
```

y no ejecutará directamente:

```text
pg_dump
mysqldump
```

desde el comando.

Las herramientas nativas podrán utilizarse internamente mediante providers/adapters definidos por Backup.

---

# 6. Command Architecture

Cada comando deberá dividirse conceptualmente en:

```text
Command Definition
       ↓
Input Parsing
       ↓
Input Validation
       ↓
Context Resolution
       ↓
Authorization
       ↓
Safety Evaluation
       ↓
Service Invocation
       ↓
Result
       ↓
Rendering
       ↓
Exit Code
```

---

# 7. Thin Command

Un comando deberá ser pequeño.

Ejemplo conceptual:

```php
final class DatabaseMigrateCommand
{
    public function __invoke(
        DatabaseMigrateInput $input,
        MigrationApplicationService $migrations,
    ): CommandResult {
        return $migrations->migrate(
            $input->toRequest(),
        );
    }
}
```

No:

```php
final class DatabaseMigrateCommand
{
    public function handle(): void
    {
        // discover files
        // parse migrations
        // inspect schema
        // calculate diff
        // generate SQL
        // open PDO
        // execute DDL
        // update migration table
        // telemetry
        // retry
        // ...
    }
}
```

---

# 8. Command Input Models

Los comandos complejos deberán transformar argumentos CLI en objetos tipados.

Ejemplo:

```php
final readonly class MigrationCommandRequest
{
    public function __construct(
        public DatabaseName $database,
        public EnvironmentName $environment,
        public MigrationDirection $direction,
        public bool $dryRun,
        public bool $interactive,
        public ?TenantId $tenant,
    ) {}
}
```

Esto evita propagar arrays arbitrarios.

---

# 9. Command Result

Los servicios deberán devolver resultados estructurados.

Ejemplo:

```php
final readonly class MigrationCommandResult
{
    public function __construct(
        public MigrationExecutionStatus $status,
        public array $executed,
        public array $skipped,
        public array $failed,
        public DiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 10. Result ≠ Console Output

Debe mantenerse:

```text
Application Result
≠
Console Rendering
```

Esto permitirá:

```text
same service
├── CLI
├── HTTP Admin UI
├── Tests
├── Automation
└── IDE integration
```

---

# 11. Namespace de comandos

La forma recomendada será:

```text
database:<operation>
```

Ejemplos:

```text
database:status
database:health
database:diagnose
database:query
database:connections
database:capabilities
database:inspect
```

Subsistemas podrán utilizar:

```text
database:schema:*
database:migrate:*
database:cache:*
database:backup:*
database:orm:*
```

---

# 12. Comandos fundamentales V1

Se propone:

```text
database:status
database:health
database:diagnose
database:connections
database:capabilities

database:query

database:schema
database:schema:diff
database:schema:inspect

database:migrate
database:migrate:status
database:migrate:rollback

database:seed

database:orm:metadata
database:orm:inspect

database:cache:clear

database:backup
database:backup:list
database:backup:verify
database:restore

database:maintenance

database:benchmark
```

---

# 13. Comandos posteriores

Podrán incorporarse:

```text
database:migrate:plan
database:migrate:validate
database:migrate:repair

database:schema:validate
database:schema:export

database:fixtures:load

database:replicas
database:shards
database:tenants

database:cache:inspect

database:transactions

database:profile

database:admin:*
```

---

# 14. `database:status`

Proporcionará una visión general del subsistema.

Ejemplo:

```text
$ php voltstack database:status

VoltStack Database

Default connection    primary
Platform              PostgreSQL
Server                 17.x
Driver                 pdo_pgsql
Status                 READY

Schema                  synchronized
Migrations              42 / 42
Connection              healthy
Capabilities            87 detected
```

---

# 15. Status ≠ Health

Debe mantenerse:

```text
Status
≠
Health
```

Status podrá combinar información descriptiva.

Health responderá a evidencia operacional específica.

---

# 16. `database:health`

Utilizará:

```text
DATABASE_HEALTH_CHECK_SYSTEM
```

No implementará health checks directamente.

Ejemplo:

```text
$ php voltstack database:health

Database Health

Connection        PASS
Authentication    PASS
Query             PASS
Transactions      PASS
Schema            PASS
Replica           WARN

Overall
  DEGRADED
```

---

# 17. Health ≠ Eligibility

La CLI deberá conservar la distinción:

```text
Healthy
≠
Eligible for write
```

Una replica puede estar:

```text
reachable
```

pero no:

```text
eligible for strongly consistent read
```

---

# 18. `database:diagnose`

Ejecutará el sistema definido por:

```text
281_DATABASE_DIAGNOSTICS_SYSTEM.md
```

y utilizará la experiencia definida en:

```text
308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md
```

---

# 19. Ejemplo diagnose

```text
$ php voltstack database:diagnose

Database Diagnostics

✓ Configuration
✓ Driver
✓ Connection
✓ Authentication
✓ Schema
✓ Transactions
! Replica lag
✓ Runtime state

1 warning detected.

DB-REPLICA-004
Replica lag exceeds configured policy.
```

---

# 20. Diagnostic verbosity

Podrá soportar:

```text
-v
-vv
-vvv
```

sin que:

```text
-vvv
```

implique exposición automática de secretos.

---

# 21. `database:connections`

Mostrará información segura de conexiones.

Ejemplo:

```text
Connection       Role       Platform      State
primary          writer     PostgreSQL    READY
replica-1        reader     PostgreSQL    READY
replica-2        reader     PostgreSQL    DEGRADED
```

---

# 22. Información sensible

No deberá mostrar:

```text
password
secret DSN
TLS private key
access token
```

---

# 23. `database:capabilities`

Consumirá:

```text
301_DATABASE_DATABASE_CAPABILITY_DISCOVERY_SYSTEM.md
```

Ejemplo:

```text
$ php voltstack database:capabilities

Capability                  Status
transactions                SUPPORTED
savepoints                  SUPPORTED
returning                   SUPPORTED
window-functions            SUPPORTED
json-query                  SUPPORTED
full-text-search            SUPPORTED_WITH_LIMITATIONS
spatial                     UNKNOWN
```

---

# 24. Capability explanation

Podrá utilizarse:

```text
--explain
```

Ejemplo:

```text
$ php voltstack database:capabilities returning --explain
```

Resultado:

```text
Capability
  returning

Status
  SUPPORTED

Mode
  NATIVE

Evidence
  Platform declaration
  Server version
  Runtime probe

Confidence
  HIGH
```

---

# 25. UNKNOWN ≠ UNSUPPORTED

La CLI deberá mostrar explícitamente:

```text
UNKNOWN
```

cuando corresponda.

Nunca convertirlo en:

```text
UNSUPPORTED
```

por conveniencia visual.

---

# 26. `database:query`

Permitirá ejecutar consultas controladas para desarrollo/administración.

Deberá considerarse una superficie sensible.

---

# 27. Query modes

Podrán existir:

```text
Query Builder expression
Named query
Raw SQL
Query file
```

según evolución.

Raw SQL deberá ser explícito.

---

# 28. Raw SQL escape hatch

Ejemplo:

```text
database:query --raw="SELECT ..."
```

deberá considerarse:

```text
Unsafe / privileged operation
```

dependiendo del entorno.

---

# 29. Production query protection

En producción podrán requerirse:

```text
authorization
explicit environment
confirmation
read-only policy
audit
```

---

# 30. Query read-only mode

Podrá existir:

```text
--read-only
```

pero la seguridad real no deberá depender únicamente de analizar el primer token SQL.

Preferiblemente deberá utilizarse:

```text
read-only database credentials
+
query semantic analysis where available
```

---

# 31. Query output

Ejemplo:

```text
$ php voltstack database:query "users active"

Rows: 3

+----+-------------------+
| id | email             |
+----+-------------------+
| 1  | [REDACTED]        |
| 5  | [REDACTED]        |
| 9  | [REDACTED]        |
+----+-------------------+
```

---

# 32. Large result protection

La CLI no deberá imprimir millones de registros accidentalmente.

Deberán existir:

```text
default limit
explicit --limit
streaming
output file
pagination
```

---

# 33. Query streaming

Para datasets grandes:

```text
database:query ... --stream
```

podrá utilizar:

```text
StreamingResultSystem
```

sin cargar todo en memoria.

---

# 34. Query output formats

Podrán soportarse:

```text
table
json
jsonl
csv
plain
```

---

# 35. Machine-readable output

La CLI deberá ofrecer una convención global:

```text
--format=json
```

o equivalente.

---

# 36. Structured output contract

Ejemplo:

```json
{
  "schema_version": 1,
  "command": "database:health",
  "status": "degraded",
  "diagnostics": []
}
```

---

# 37. Human output ≠ Machine output

La salida humana podrá evolucionar visualmente.

La salida machine-readable deberá poseer contrato versionado.

---

# 38. `database:schema`

Mostrará el schema normalizado conocido.

Ejemplo:

```text
$ php voltstack database:schema

Database: primary

Tables
  users
  orders
  order_items
  products
```

---

# 39. Schema detail

```text
database:schema users
```

podrá mostrar:

```text
Columns
Indexes
Constraints
Foreign Keys
Capabilities
```

---

# 40. `database:schema:inspect`

Consumirá el:

```text
SchemaIntrospectionSystem
```

---

# 41. Introspection ≠ schema model declaration

La CLI deberá distinguir:

```text
Declared schema
```

de:

```text
Observed database schema
```

---

# 42. Coverage

Si introspection es parcial:

```text
Coverage:
PARTIAL
```

deberá mostrarse.

---

# 43. Not observed ≠ absent

La CLI nunca deberá representar un elemento no observado como inexistente.

---

# 44. `database:schema:diff`

Comparará:

```text
Current Schema
→
Target Schema
```

mediante:

```text
SchemaDiffSystem
```

---

# 45. Ejemplo

```text
$ php voltstack database:schema:diff

Schema differences

+ users.phone
~ users.email varchar(120) → varchar(255)
- users.legacy_code

Safety

ADD phone             SAFE
ALTER email           REVIEW
DROP legacy_code      DESTRUCTIVE
```

---

# 46. Diff ≠ Migration

Debe mantenerse:

```text
Schema Difference
≠
Migration
```

---

# 47. `--dry-run`

Será una opción transversal importante.

Ejemplo:

```text
database:migrate --dry-run
```

---

# 48. Dry-run semantics

`--dry-run` deberá significar:

> construir, resolver, validar, planificar y mostrar la operación sin ejecutar sus efectos mutables finales.

No:

```text
skip random lines of code
```

---

# 49. Dry-run ≠ guarantee

Una operación que pasa dry-run puede fallar posteriormente debido a:

```text
concurrency
environment changes
permissions
resource availability
schema drift
```

---

# 50. `--explain`

Podrá utilizarse para mostrar:

```text
why this plan
why this route
why this capability decision
why this safety decision
```

---

# 51. Explain ≠ Debug dump

Debe producir información semántica estructurada.

---

# 52. `database:migrate`

Consumirá la arquitectura 101–111.

Flujo:

```text
CLI
 ↓
Migration Request
 ↓
Discovery
 ↓
Repository
 ↓
Planner
 ↓
Safety
 ↓
Execution
 ↓
Repository Update
 ↓
Result
 ↓
CLI Renderer
```

---

# 53. Ejemplo migrate

```text
$ php voltstack database:migrate

Migration Plan

202609210001_create_users
202609210002_create_orders
202609210003_add_order_status

3 migrations pending.

Apply migrations? [y/N]
```

---

# 54. Non-interactive mode

CI deberá poder utilizar:

```text
--no-interaction
```

---

# 55. No interaction ≠ bypass safety

Regla crítica:

```text
--no-interaction
≠
--ignore-safety
```

---

# 56. Production migrations

Podrán requerir:

```text
--environment=production
```

y políticas adicionales.

---

# 57. Environment inference

La CLI podrá detectar el environment actual.

Pero operaciones destructivas deberán favorecer intención explícita.

---

# 58. `--force`

Si existe:

```text
--force
```

no deberá significar:

```text
disable all safeguards
```

---

# 59. Non-bypassable safeguards

Algunas protecciones deberán ser imposibles de omitir mediante `--force`.

Por ejemplo:

```text
proven production safety violation
cross-tenant unauthorized operation
missing authorization
unknown destructive target
```

---

# 60. Migration safety

Ejemplo:

```text
$ php voltstack database:migrate

BLOCKED

Migration:
202609210010_drop_legacy_table

Reason:
Destructive operation detected.

Safety:
DESTRUCTIVE

Diagnostic:
DB-MIG-SAFETY-004
```

---

# 61. `database:migrate:status`

Ejemplo:

```text
Migration                         Status      Batch
create_users                      EXECUTED    1
create_orders                     EXECUTED    1
add_order_status                  PENDING     -
```

---

# 62. `database:migrate:rollback`

Rollback deberá utilizar:

```text
MigrationRollbackSystem
```

y no invertir SQL automáticamente.

---

# 63. Rollback ≠ inverse SQL

Regla:

```text
Rollback
≠
Automatically reverse every SQL command
```

---

# 64. Rollback safety

Rollback también deberá pasar por evaluación de seguridad.

---

# 65. Migration plan command

Se recomienda incorporar:

```text
database:migrate:plan
```

para inspeccionar:

```text
dependencies
operations
safety
capabilities
transaction boundaries
estimated impact
```

antes de ejecutar.

---

# 66. `database:seed`

Consumirá:

```text
SeederSystem
```

---

# 67. Seed environment protection

Seeders podrán clasificarse:

```text
development
testing
demo
production-safe
```

---

# 68. Production seeding

No deberá asumirse que todo seeder es seguro para producción.

---

# 69. Seed selection

Ejemplo:

```text
database:seed UserDemoSeeder
```

---

# 70. Seed dependencies

La CLI podrá mostrar el DAG resuelto.

```text
RolesSeeder
   ↓
UsersSeeder
   ↓
DemoOrdersSeeder
```

---

# 71. Deterministic seed

Podrá soportarse:

```text
--seed=12345
```

para Test Data Generator.

---

# 72. Seed value ≠ seeder

Debe evitarse confundir ambos conceptos.

---

# 73. Fixture CLI

Posteriormente:

```text
database:fixtures:load
```

podrá integrarse con FixtureSystem.

---

# 74. `database:orm:metadata`

Permitirá inspeccionar metadata ORM compilada.

Ejemplo:

```text
Entity
  App\Entity\User

Table
  users

Identifier
  id

Fields
  id
  email
  createdAt

Relationships
  orders -> OneToMany
```

---

# 75. Metadata inspection

No deberá cargar entidades ni ejecutar consultas innecesarias.

---

# 76. `database:orm:inspect`

Podrá validar:

```text
mapping
metadata
schema compatibility
relationship definitions
type mappings
```

---

# 77. ORM validation ≠ schema migration

Detectar una diferencia no deberá modificar automáticamente la base.

---

# 78. `database:cache:clear`

Deberá identificar qué cache se limpia.

Ejemplo:

```text
--type=metadata
--type=query
--type=result
--type=entity
--all
```

---

# 79. Cache distinctions

Debe conservarse:

```text
Query Cache
≠
Compiled Query Cache
≠
Result Cache
≠
Metadata Cache
≠
Entity Cache
≠
IdentityMap
```

---

# 80. IdentityMap

No deberá existir un comando global:

```text
database:cache:clear identity-map
```

como si IdentityMap fuese cache persistente.

IdentityMap pertenece al scope ORM.

---

# 81. `database:backup`

Consumirá:

```text
DATABASE_BACKUP_SYSTEM
```

---

# 82. Backup command

Ejemplo:

```text
$ php voltstack database:backup

Backup Plan

Database        primary
Strategy        native
Consistency     transactional
Destination     configured-backup-storage

Start backup? [y/N]
```

---

# 83. Backup result

Debe distinguir:

```text
CREATED
VERIFIED
RESTORABLE
```

---

# 84. Created ≠ verified

La CLI no deberá mostrar simplemente:

```text
Backup successful
```

si sólo se creó el artefacto.

---

# 85. Ejemplo

```text
Backup
  backup_20260921_080500

Creation
  SUCCESS

Verification
  SUCCESS

Restore verification
  NOT PERFORMED

Restorable
  UNKNOWN
```

---

# 86. `database:backup:verify`

Permitirá verificar artefactos existentes.

---

# 87. `database:backup:list`

Podrá consultar el Backup Catalog.

---

# 88. `database:restore`

Será uno de los comandos de mayor riesgo.

---

# 89. Restore architecture

```text
CLI
 ↓
Restore Request
 ↓
Preflight
 ↓
Artifact Verification
 ↓
Restore Planner
 ↓
Safety Evaluation
 ↓
Authorization
 ↓
Execution
 ↓
Validation
 ↓
Cutover
```

---

# 90. Restore confirmation

No bastará una confirmación genérica:

```text
Are you sure?
```

Podrá requerirse confirmación del target.

Ejemplo:

```text
Type database name "production-primary" to continue:
```

según política.

---

# 91. Automation restore

En automatización no interactiva deberán existir mecanismos explícitos de autorización y aprobación.

No se dependerá de prompts humanos.

---

# 92. `database:maintenance`

Podrá ejecutar tareas del:

```text
DATABASE_DATABASE_MAINTENANCE_SYSTEM
```

---

# 93. Maintenance examples

```text
analyze
vacuum
optimize
reindex
statistics refresh
```

dependiendo de capabilities.

---

# 94. Semantic maintenance

La CLI solicitará:

```text
database:maintenance --operation=statistics-refresh
```

El Platform decidirá la representación concreta.

---

# 95. No vendor command leakage

La API principal no debería requerir:

```text
database:maintenance --postgres-vacuum
```

salvo herramientas administrativas explícitamente platform-specific.

---

# 96. `database:benchmark`

Consumirá:

```text
293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

---

# 97. Ejemplo

```text
php voltstack database:benchmark \
    --suite=query \
    --platform=postgresql \
    --profile=nightly
```

---

# 98. Benchmark result

```text
Benchmark
  query.simple_lookup

Samples
  1000

p50
  1.42 ms

p95
  2.11 ms

p99
  3.08 ms

Baseline
  comparable

Change
  +2.1%

Assessment
  NO_SIGNIFICANT_CHANGE
```

---

# 99. Benchmark ≠ profiler

Debe mantenerse:

```text
Benchmark
≠
Profiler
```

---

# 100. Benchmark environment

La CLI deberá registrar:

```text
environment fingerprint
dataset fingerprint
schema generation
driver
DBMS
runtime
capabilities
```

---

# 101. Connection selection

Los comandos podrán aceptar:

```text
--connection=primary
```

---

# 102. Logical connection

La opción deberá referirse preferentemente a una conexión lógica configurada.

No directamente a un objeto PDO.

---

# 103. Database selection

Podrá existir:

```text
--database=analytics
```

cuando la arquitectura lo requiera.

---

# 104. Tenant selection

Para instalaciones con Multitenancy:

```text
--tenant=<id>
```

podrá resolver TenantContext.

---

# 105. Tenant option ≠ arbitrary string injection

El valor deberá pasar por:

```text
TenantResolver
Authorization
TenantContext
```

---

# 106. Cross-tenant operations

Deberán ser explícitas.

Ejemplo futuro:

```text
--all-tenants
```

deberá considerarse privilegiado.

---

# 107. No implicit all-tenants

Si falta tenant:

```text
tenant = UNKNOWN
```

no deberá convertirse en:

```text
all tenants
```

---

# 108. Shard selection

Podrá soportarse:

```text
--shard=shard-03
```

para herramientas administrativas autorizadas.

---

# 109. Shard bypass

La selección manual de shard deberá considerarse operación avanzada.

No deberá permitir violar invariantes de ownership silenciosamente.

---

# 110. Replica selection

Herramientas diagnósticas podrán permitir:

```text
--endpoint=replica-2
```

para inspección explícita.

---

# 111. Query routing

Operaciones normales deberán utilizar el Routing System.

La CLI no deberá elegir endpoints arbitrariamente salvo comando administrativo/diagnóstico explícito.

---

# 112. Global options

Propuesta:

```text
--connection
--database
--tenant
--shard
--environment
--format
--quiet
--no-interaction
--dry-run
--explain
-v / -vv / -vvv
```

No todos aplicarán a todos los comandos.

---

# 113. Global option validation

Una opción no aplicable deberá generar error.

No ignorarse silenciosamente.

---

# 114. Example

```text
database:health --dry-run
```

podría ser inválido si health no tiene efectos mutables.

La CLI deberá explicarlo.

---

# 115. Interactive Mode

La CLI podrá solicitar:

```text
confirmation
selection
missing non-sensitive values
```

---

# 116. Secret input

Si alguna operación requiere secret:

```text
hidden input
```

deberá utilizarse.

---

# 117. Command history

Secrets no deberán pasarse preferentemente mediante argumentos que terminen en shell history.

Evitar:

```text
--password=mysecret
```

---

# 118. Secret providers

Preferir:

```text
environment
secret provider
secure prompt
credential store
```

---

# 119. Non-interactive secrets

CI deberá utilizar secret providers.

---

# 120. Environment awareness

La CLI deberá conocer:

```text
development
testing
staging
production
```

o el modelo de environments configurado.

---

# 121. Environment ≠ safety proof

Regla:

```text
APP_ENV=testing
≠
Database proven safe for destructive testing
```

La arquitectura de Test Environment ya estableció este principio.

---

# 122. Production banner

Operaciones sensibles podrán mostrar:

```text
WARNING: PRODUCTION DATABASE
```

como ayuda UX.

Pero el banner no sustituye las protecciones reales.

---

# 123. Safety levels

Los comandos podrán declarar:

```text
READ_ONLY
MUTATING
SCHEMA_MUTATING
DATABASE_DESTRUCTIVE
SERVER_ADMINISTRATIVE
```

---

# 124. Safety metadata

Ejemplo:

```php
#[DatabaseCommandSafety(
    level: SafetyLevel::SCHEMA_MUTATING
)]
final class DatabaseMigrateCommand
{
}
```

---

# 125. Safety evaluation

```text
Command
 ↓
Operation Intent
 ↓
Safety Classification
 ↓
Environment Evidence
 ↓
Authorization
 ↓
Policy
 ↓
ALLOW / DENY / REQUIRE_CONFIRMATION
```

---

# 126. Safety ≠ confirmation

Una operación insegura no se vuelve segura simplemente porque el usuario escribió:

```text
yes
```

---

# 127. Authorization

Operaciones administrativas deberán pasar por:

```text
DATABASE_DATABASE_PERMISSION_MODEL
```

cuando corresponda.

---

# 128. Local developer environment

Podrá existir una policy simplificada.

Pero el diseño seguirá soportando autorización completa.

---

# 129. Audit

Operaciones privilegiadas deberán integrarse con:

```text
DATABASE_QUERY_AUDIT_SYSTEM
```

y sistemas administrativos correspondientes.

---

# 130. Audit metadata

Podrá registrar:

```text
command
actor
environment
target
operation
authorization decision
outcome
timestamp
correlation
```

sin secretos.

---

# 131. CLI exit codes

La CLI deberá utilizar códigos de salida consistentes.

Propuesta:

```text
0   SUCCESS
1   GENERAL_FAILURE
2   INVALID_INPUT
3   CONFIGURATION_ERROR
4   CONNECTION_ERROR
5   OPERATION_FAILED
6   SAFETY_BLOCKED
7   AUTHORIZATION_DENIED
8   INCONCLUSIVE
9   PARTIAL_SUCCESS
10  UNKNOWN_OUTCOME
```

---

# 132. Exit codes ≠ Diagnostic codes

Debe mantenerse:

```text
Process Exit Code
≠
Database Diagnostic Code
```

---

# 133. Unknown outcome

Un commit incierto deberá poder producir:

```text
exit code:
UNKNOWN_OUTCOME
```

y diagnóstico:

```text
DB-TX-017
```

---

# 134. Partial success

Bulk/import/migration multi-target operations podrán terminar:

```text
PARTIAL_SUCCESS
```

---

# 135. Exit code stability

Una vez publicados para scripting, deberán tratarse como API.

---

# 136. Shell scripting

Ejemplo:

```bash
php voltstack database:health --format=json
```

deberá permitir automatización confiable.

---

# 137. JSON and stderr

Deberá definirse claramente:

```text
stdout = requested machine output
stderr = diagnostics/errors
```

cuando sea compatible con el framework CLI.

---

# 138. Machine mode

En:

```text
--format=json
```

no deberán mezclarse:

```text
ANSI colors
progress bars
interactive prompts
decorative banners
```

con JSON.

---

# 139. Quiet mode

```text
--quiet
```

reducirá output.

Pero no deberá ocultar errores críticos necesarios para automatización.

---

# 140. Progress reporting

Operaciones largas podrán mostrar:

```text
progress
current phase
elapsed time
processed count
```

---

# 141. Progress ≠ outcome

```text
100% uploaded
```

no necesariamente significa:

```text
backup verified
```

---

# 142. Cancellation

Ctrl+C / SIGINT deberá integrarse con:

```text
Database Query Cancellation
Migration cancellation policy
Backup cancellation
Restore cancellation
```

según operación.

---

# 143. Signal handling

La CLI no deberá simplemente matar el proceso cuando exista una oportunidad segura de:

```text
cancel
rollback
cleanup
release resources
```

---

# 144. Cancellation uncertainty

Si el proceso termina en una fase incierta:

```text
Outcome:
UNKNOWN
```

deberá preservarse.

---

# 145. SIGTERM

En CI/contenedores también deberá manejarse cuando el runtime CLI lo permita.

---

# 146. Transaction ownership

El comando deberá saber quién controla la transacción.

Ejemplo:

```text
MigrationSystem
```

y no la CLI arbitrariamente.

---

# 147. CLI transaction wrapper

No deberá existir un wrapper genérico:

```php
DB::transaction(fn () => $command->run());
```

alrededor de todos los comandos.

---

# 148. Reason

Algunas operaciones:

```text
DDL
backup
restore
maintenance
multi-database
```

poseen semánticas de transacción diferentes.

---

# 149. Connection lifecycle

La CLI deberá utilizar:

```text
ConnectionManager
```

---

# 150. No direct driver construction

Evitar:

```php
new PDO(...)
```

dentro de comandos.

---

# 151. CLI process model

Normalmente:

```text
CLI invocation
=
single operation scope
```

---

# 152. Long-running CLI

Sin embargo podrán existir:

```text
import
export
benchmark
maintenance
migration across tenants
```

de larga duración.

---

# 153. Long-running scope management

Deberán evitar crecimiento ilimitado de:

```text
IdentityMap
UnitOfWork
DiagnosticCollector
Query telemetry
buffers
```

---

# 154. Chunk boundaries

Operaciones largas deberán utilizar:

```text
ChunkProcessingSystem
StreamingResultSystem
BatchPersistenceSystem
```

cuando corresponda.

---

# 155. Memory

La CLI deberá respetar:

```text
Database Memory Management
Resource Governance
```

---

# 156. Resource budgets

Comandos podrán aceptar políticas:

```text
--timeout
--memory-budget
--batch-size
```

cuando sean semánticamente válidas.

---

# 157. User value ≠ unlimited

Ejemplo:

```text
--batch-size=999999999
```

deberá validarse contra políticas.

---

# 158. Timeout

Debe distinguirse:

```text
CLI timeout
query timeout
lock timeout
connection timeout
operation timeout
```

---

# 159. CLI timeout ≠ query timeout

No deberán fusionarse.

---

# 160. Command discovery

Los comandos Database deberán registrarse mediante:

```text
DatabaseCommandRegistry
```

o integración equivalente con el CLI core.

---

# 161. Registry

Deberá ser:

```text
deterministic
validated
frozen after bootstrap
```

---

# 162. Duplicate commands

Dos comandos con el mismo nombre deberán producir error durante bootstrap.

---

# 163. Plugin commands

Plugins Database podrán registrar comandos adicionales.

Ejemplo:

```text
database:vendor-feature
```

---

# 164. Plugin namespace

Se recomienda evitar colisiones mediante namespaces claros.

---

# 165. Plugin command restrictions

Un plugin no deberá poder reemplazar silenciosamente:

```text
database:migrate
```

del core.

---

# 166. Command metadata

Conceptualmente:

```php
final readonly class DatabaseCommandDefinition
{
    public function __construct(
        public CommandName $name,
        public string $description,
        public SafetyLevel $safety,
        public bool $interactive,
        public OutputCapabilities $output,
        public CommandCapabilityRequirements $requirements,
    ) {}
}
```

---

# 167. Capability requirements

Un comando podrá declarar:

```text
requires:
BACKUP_NATIVE
```

---

# 168. Capability check

Si:

```text
status = UNSUPPORTED
```

deberá fallar de forma explicable.

Si:

```text
status = UNKNOWN
```

no deberá presentarse como unsupported.

---

# 169. Conditional commands

Algunos comandos podrán existir pero indicar:

```text
unavailable under current platform capabilities
```

---

# 170. Help system

```text
php voltstack help database:migrate
```

deberá mostrar:

```text
purpose
usage
arguments
options
safety level
examples
required capabilities
```

---

# 171. Safety documentation

Para comandos peligrosos deberá ser visible.

---

# 172. Command examples

Los ejemplos de help deberán favorecer prácticas seguras.

---

# 173. Aliases

Podrán existir aliases limitados.

Pero nombres canónicos deberán permanecer estables.

---

# 174. Deprecation

Comandos/options obsoletos deberán utilizar:

```text
Database Diagnostic Deprecation
```

---

# 175. Example

```text
Option --conn is deprecated.

Use:
--connection

Removal:
VoltStack 2.0
```

---

# 176. No silent alias removal

Cambios deberán seguir:

```text
DATABASE_DEPRECATION_POLICY
```

---

# 177. Output abstraction

Propuesta:

```text
CommandResult
    ↓
OutputFormatter
    ↓
OutputRenderer
```

---

# 178. Formatter vs Renderer

```text
Formatter
=
semantic representation
```

```text
Renderer
=
terminal/JSON/etc. presentation
```

---

# 179. Human table

El renderer podrá adaptar ancho de terminal.

---

# 180. Non-TTY

Cuando stdout no sea TTY:

```text
colors disabled
interactive behavior disabled or explicit
```

según policy.

---

# 181. CI detection

La CLI podrá detectar CI para UX.

Pero:

```text
CI detected
≠
authorization granted
```

---

# 182. Color

Opciones:

```text
--ansi
--no-ansi
```

podrán ser heredadas del VoltStack CLI general.

---

# 183. Diagnostic rendering

El comando no deberá formatear excepciones manualmente.

Usará:

```text
ConsoleDiagnosticRenderer
```

definido en documento 308.

---

# 184. Example

No:

```php
catch (Throwable $e) {
    $output->error($e->getMessage());
}
```

como arquitectura general.

Preferible:

```php
catch (DatabaseException $e) {
    return $diagnostics->render($e->diagnostic());
}
```

en la frontera adecuada.

---

# 185. Central exception boundary

VoltStack CLI podrá tener:

```text
CLI Exception Boundary
```

que conozca `DatabaseDiagnostic`.

---

# 186. Command-specific recovery

Un comando sólo deberá capturar errores cuando pueda:

```text
recover
enrich
aggregate
perform required cleanup
```

---

# 187. No catch-and-hide

Prohibido:

```php
try {
    ...
} catch (Throwable) {
    return 0;
}
```

---

# 188. Dry-run rendering

Ejemplo:

```text
$ php voltstack database:migrate --dry-run

DRY RUN

3 migrations would be executed.

1. create_users
2. create_orders
3. add_order_status

Database changes:
NONE
```

---

# 189. Dry-run marker

La salida deberá indicar claramente que no hubo ejecución.

---

# 190. Explain plan

Ejemplo:

```text
$ php voltstack database:migrate --dry-run --explain

Migration
  add_orders_index

Operation
  CREATE INDEX

Capability
  SUPPORTED

Execution strategy
  ONLINE

Reason
  Platform supports online index creation.

Safety
  REVIEW
```

---

# 191. `--yes`

Podrá existir para aceptar prompts ordinarios.

Pero:

```text
--yes
≠
override authorization
```

y:

```text
--yes
≠
override non-bypassable safety
```

---

# 192. Destructive confirmation tokens

Operaciones de alto riesgo podrán requerir tokens/approvals específicos en lugar de `--yes`.

---

# 193. Administration mode

Podrá existir una familia:

```text
database:admin:*
```

separada de operaciones de desarrollo ordinarias.

---

# 194. Admin commands

Ejemplos futuros:

```text
database:admin:create-database
database:admin:drop-database
database:admin:terminate-connection
database:admin:permissions
```

---

# 195. High privilege separation

Estos comandos deberán requerir credenciales/roles administrativos específicos.

---

# 196. Runtime credentials ≠ Admin credentials

Regla:

```text
Runtime Credential
≠
Migration Credential
≠
Backup Credential
≠
Administration Credential
```

---

# 197. Least privilege

La CLI resolverá el credential role adecuado según operación.

---

# 198. Credential escalation

No deberá utilizar automáticamente superuser porque un comando ordinario falla por permisos.

---

# 199. Connection testing

Podrá existir:

```text
database:connections:test
```

---

# 200. Test connection

Deberá comprobar:

```text
resolve
connect
authenticate
minimal query
optional TLS
optional capability probe
cleanup
```

---

# 201. Test connection ≠ health

Aunque relacionados, no son idénticos.

---

# 202. Query profiling

Futuro:

```text
database:profile
```

podrá utilizar Query Profiler.

---

# 203. Profiling production

Deberá ser controlado por políticas debido al overhead.

---

# 204. Query explain

Podrá existir:

```text
database:query:explain
```

o:

```text
database:query --explain
```

---

# 205. Database EXPLAIN

No deberá confundirse con:

```text
VoltStack semantic --explain
```

Se recomienda distinguir:

```text
--explain
```

para decisión VoltStack y:

```text
--execution-plan
```

para plan del DBMS, o nomenclatura equivalente.

---

# 206. DBMS plan

El plan nativo deberá tratarse como platform-specific evidence.

---

# 207. Unsafe EXPLAIN variants

Algunos DBMS tienen variantes que ejecutan realmente la consulta.

La CLI deberá conocer la diferencia.

---

# 208. `EXPLAIN ANALYZE`

No deberá ejecutarse accidentalmente sobre:

```text
UPDATE
DELETE
INSERT
```

sin intención explícita y políticas adecuadas.

---

# 209. Code generation relation

El siguiente documento:

```text
310_DATABASE_CODE_GENERATION_SYSTEM.md
```

definirá generadores.

La CLI únicamente expondrá comandos como:

```text
make:migration
make:entity
make:repository
```

o namespace definitivo.

---

# 210. CLI ≠ Code Generator

Debe mantenerse:

```text
CLI command
→ CodeGenerationSystem
```

---

# 211. Testing architecture

La CLI deberá poseer pruebas en varias capas.

```text
Unit
Integration
Command Contract
Security
Safety
Platform Integration
```

---

# 212. Unit tests

Probarán:

```text
input parsing
validation
format selection
exit-code mapping
command metadata
```

sin DB real cuando no sea necesaria.

---

# 213. Integration tests

Probarán:

```text
command
→ real service
→ real database
```

cuando la garantía dependa del DBMS.

---

# 214. CLI tests ≠ Database conformance

Que:

```text
database:migrate
```

funcione una vez no demuestra conformance completa del driver.

---

# 215. Safety tests

Deberán comprobar:

```text
production protection
destructive operation blocking
tenant boundaries
authorization
secret redaction
```

---

# 216. Machine output tests

El JSON deberá validarse contra su schema/version.

---

# 217. Exit code tests

Los exit codes serán parte del contrato.

---

# 218. Signal tests

Operaciones largas deberán probar:

```text
SIGINT
SIGTERM
cleanup
transaction outcome
```

cuando la plataforma de testing lo permita.

---

# 219. Persistent CLI process

Si VoltStack incorpora en el futuro:

```text
interactive database shell
daemonized CLI
remote console
```

no deberá reutilizar contextos mutables entre operaciones sin reset.

---

# 220. Remote CLI

Si en el futuro la CLI opera contra servicios remotos:

```text
CLI
→ Admin API
→ Database Service
```

deberán mantenerse los mismos:

```text
authorization
diagnostics
audit
safety
```

---

# 221. No SSH architecture dependency

La arquitectura no deberá depender de SSH como único mecanismo remoto.

---

# 222. Command context

Podrá existir:

```php
final readonly class DatabaseCliContext
{
    public function __construct(
        public EnvironmentName $environment,
        public DatabaseContext $database,
        public InteractionMode $interaction,
        public OutputFormat $format,
        public CliSecurityContext $security,
    ) {}
}
```

---

# 223. CLI Context ≠ DatabaseContext

`DatabaseCliContext` contiene concerns de presentación/ejecución CLI.

`DatabaseContext` contiene contexto semántico de Database.

---

# 224. No CLI pollution

Database Engine no deberá recibir:

```text
ANSI
terminal width
progress bar
interactive prompt
```

---

# 225. Command pipeline

Propuesta formal:

```text
Parse
  ↓
Normalize
  ↓
Validate
  ↓
Resolve Context
  ↓
Resolve Capabilities
  ↓
Authorize
  ↓
Evaluate Safety
  ↓
Confirm if required
  ↓
Execute
  ↓
Collect Diagnostics
  ↓
Render
  ↓
Exit
```

---

# 226. Pipeline extensions

Plugins podrán añadir pasos controlados en puntos explícitos.

---

# 227. No arbitrary middleware mutation

Un plugin no deberá poder alterar silenciosamente:

```text
target database
tenant
safety decision
authorization result
```

después de su validación.

---

# 228. Context freeze

Antes de ejecutar una operación sensible podrá congelarse:

```text
ResolvedCommandContext
```

---

# 229. TOCTOU

Para operaciones críticas deberá considerarse:

```text
Time Of Check
vs
Time Of Use
```

---

# 230. Revalidation

Antes de operaciones destructivas podrá revalidarse:

```text
database identity
environment identity
target
authorization
schema generation
```

---

# 231. Database identity

Una conexión resuelta no será suficiente.

Podrá verificarse que el servidor/base real corresponde al target esperado.

---

# 232. Destructive target identity

Especialmente para:

```text
restore
drop
reset
test environment cleanup
```

---

# 233. Plan fingerprint

Operaciones complejas podrán generar:

```text
PlanFingerprint
```

---

# 234. Confirmation binding

Una aprobación podrá quedar vinculada a:

```text
target
plan fingerprint
environment
actor
expiration
```

para evitar aprobar A y ejecutar B.

---

# 235. Plan changed after confirmation

Si el plan cambia:

```text
confirmation invalid
```

para operaciones que requieran este nivel de seguridad.

---

# 236. Multi-target operations

Ejemplo:

```text
migrate all tenants
```

deberá representar cada target explícitamente.

---

# 237. Multi-target result

```text
Total tenants     100
Succeeded          97
Failed              2
Unknown             1
```

---

# 238. UNKNOWN target

No deberá contarse como simplemente failed.

---

# 239. Continue-on-error

Podrá existir:

```text
--continue-on-error
```

sólo donde la operación permita targets independientes.

---

# 240. Atomic multi-target illusion

La CLI no deberá prometer:

```text
atomic migration across 100 independent databases
```

si no existe una transacción distribuida real.

---

# 241. Concurrency

Multi-target commands podrán soportar:

```text
--concurrency=N
```

---

# 242. Resource governance

La concurrencia deberá limitarse por:

```text
connection budgets
CPU
memory
DB server capacity
policy
```

---

# 243. Concurrency default

El default deberá ser conservador.

---

# 244. Progress with concurrency

El renderer deberá separar:

```text
scheduled
running
succeeded
failed
unknown
```

---

# 245. Deterministic reporting

Aunque la ejecución sea concurrente, el resumen podrá ordenarse determinísticamente.

---

# 246. Telemetry

Cada command execution podrá crear:

```text
CLI span
```

si Telemetry está disponible.

---

# 247. Attributes

Ejemplo:

```text
database.command
database.operation
database.platform
database.outcome
```

con cardinalidad controlada.

---

# 248. Command arguments telemetry

No deberán enviarse argumentos completos indiscriminadamente.

Podrían contener:

```text
SQL
PII
paths
tokens
tenant IDs
```

---

# 249. Events

Podrán existir eventos:

```text
DatabaseCliCommandStarting
DatabaseCliCommandCompleted
DatabaseCliCommandFailed
```

si aportan valor.

Pero:

```text
CLI Event
≠
Database Transaction Event
```

---

# 250. CLI events do not redefine outcome

Un listener que falle después de una migration completada no convierte la migration en rollback.

---

# 251. Error example: connection

```text
$ php voltstack database:health

Database Error [DB-CONN-004]

Unable to connect to database.

Connection
  primary

Platform
  PostgreSQL

Phase
  CONNECT

Outcome
  NOT_EXECUTED

Cause
  Connection refused

Suggestion
  Verify that the configured database endpoint is reachable.

Exit code
  4
```

---

# 252. Error example: unknown commit

```text
$ php voltstack database:migrate

Database Error [DB-TX-017]

Transaction commit outcome cannot be determined.

Migration
  202609210045_add_invoice_index

Phase
  COMMIT

Outcome
  UNKNOWN

Automatic retry
  FORBIDDEN

Exit code
  10
```

---

# 253. Error example: safety

```text
$ php voltstack database:restore backup_42

Operation blocked.

Diagnostic
  DB-RESTORE-SAFETY-002

Target
  production-primary

Reason
  Restore would replace an active production database.

Safety
  DATABASE_DESTRUCTIVE

Exit code
  6
```

---

# 254. Error example: capability

```text
$ php voltstack database:maintenance --operation=online-reindex

Unable to plan operation.

Capability
  ONLINE_REINDEX

Status
  UNSUPPORTED

Platform
  SQLite

Outcome
  NOT_EXECUTED
```

---

# 255. UNKNOWN capability example

```text
Capability
  ONLINE_REINDEX

Status
  UNKNOWN

Reason
  Capability discovery could not complete.

Outcome
  NOT_EXECUTED
```

---

# 256. Suggested namespace

```text
src/Quantum/Database/
└── Cli/
    ├── Contract/
    │   ├── DatabaseCommandInterface.php
    │   ├── DatabaseCommandRegistryInterface.php
    │   ├── CommandResultInterface.php
    │   └── CommandOutputRendererInterface.php
    │
    ├── Command/
    │   ├── Status/
    │   ├── Health/
    │   ├── Diagnostics/
    │   ├── Connection/
    │   ├── Capability/
    │   ├── Query/
    │   ├── Schema/
    │   ├── Migration/
    │   ├── Seeder/
    │   ├── Orm/
    │   ├── Cache/
    │   ├── Backup/
    │   ├── Restore/
    │   ├── Maintenance/
    │   └── Benchmark/
    │
    ├── Input/
    ├── Result/
    ├── Context/
    ├── Registry/
    ├── Pipeline/
    ├── Safety/
    ├── Authorization/
    ├── Output/
    │   ├── Human/
    │   ├── Json/
    │   ├── JsonLines/
    │   └── Plain/
    │
    ├── Interaction/
    ├── Signal/
    ├── Telemetry/
    ├── Extension/
    └── Testing/
```

---

# 257. Application services

Cuando una operación necesite orquestar varios componentes, se recomienda:

```text
Database Application Service
```

en lugar de convertir el comando en orquestador complejo.

Ejemplo:

```text
Cli Command
    ↓
MigrationApplicationService
    ↓
Migration Components
```

---

# 258. Application Service ≠ Domain Core

El Application Service coordina.

No redefine:

```text
migration semantics
query semantics
transaction semantics
```

---

# 259. Dependency rules

Permitido:

```text
CLI
↓
Public/Application Database Services
↓
Database Components
```

Prohibido:

```text
Driver
↓
CLI
```

Prohibido:

```text
Query Compiler
↓
CLI
```

Prohibido:

```text
ORM
↓
Console Renderer
```

---

# 260. Invariantes generales

## DB-CLI-001

CLI ≠ Database Engine.

## DB-CLI-002

Command ≠ Application Service.

## DB-CLI-003

Command Result ≠ Console Output.

## DB-CLI-004

CLI Context ≠ DatabaseContext.

## DB-CLI-005

Exit Code ≠ Diagnostic Code.

## DB-CLI-006

Dry Run ≠ Execution.

## DB-CLI-007

Explain ≠ Debug Dump.

## DB-CLI-008

Confirmation ≠ Authorization.

## DB-CLI-009

Confirmation ≠ Safety.

## DB-CLI-010

`--force` ≠ Disable Safety.

---

# 261. Invariantes de arquitectura

## DB-CLI-011

Commands serán thin.

## DB-CLI-012

Commands no construirán PDO directamente.

## DB-CLI-013

Commands no compilarán SQL directamente.

## DB-CLI-014

Commands no implementarán Migration Engine.

## DB-CLI-015

Commands no implementarán Backup Engine.

## DB-CLI-016

Commands no implementarán ORM.

## DB-CLI-017

Commands utilizarán servicios existentes.

## DB-CLI-018

Database Core no dependerá de CLI.

## DB-CLI-019

Renderers no definirán semántica.

## DB-CLI-020

Input parsing no modificará semántica de Database.

---

# 262. Invariantes de seguridad

## DB-CLI-021

Secrets no serán mostrados.

## DB-CLI-022

Passwords no deberán pasarse por argumentos ordinarios.

## DB-CLI-023

DSNs serán sanitizados.

## DB-CLI-024

Raw SQL será explícito.

## DB-CLI-025

Production operations tendrán políticas adicionales.

## DB-CLI-026

Tenant boundaries serán respetados.

## DB-CLI-027

Shard selection manual será privilegiada cuando corresponda.

## DB-CLI-028

Admin credentials ≠ runtime credentials.

## DB-CLI-029

CLI no elevará privilegios automáticamente.

## DB-CLI-030

Sensitive operations serán auditables.

---

# 263. Invariantes de safety

## DB-CLI-031

Safety level será declarable.

## DB-CLI-032

Destructive operations requerirán mayor evidencia.

## DB-CLI-033

`--yes` no omitirá authorization.

## DB-CLI-034

`--yes` no omitirá non-bypassable safeguards.

## DB-CLI-035

`--no-interaction` no omitirá safety.

## DB-CLI-036

Environment name no será suficiente prueba de seguridad.

## DB-CLI-037

Target identity podrá verificarse.

## DB-CLI-038

Plan approval podrá vincularse al fingerprint.

## DB-CLI-039

Plan changes podrán invalidar approvals.

## DB-CLI-040

Unknown target identity bloqueará operaciones destructivas por defecto.

---

# 264. Invariantes de output

## DB-CLI-041

Human output podrá evolucionar.

## DB-CLI-042

Machine output será versionado.

## DB-CLI-043

JSON no contendrá decoración ANSI.

## DB-CLI-044

Machine output no contendrá prompts.

## DB-CLI-045

Output grande será bounded o streaming.

## DB-CLI-046

Truncation será explícita.

## DB-CLI-047

UNKNOWN se mostrará como UNKNOWN.

## DB-CLI-048

PARTIAL_SUCCESS será representable.

## DB-CLI-049

UNKNOWN_OUTCOME será representable.

## DB-CLI-050

Exit codes serán estables una vez publicados.

---

# 265. Invariantes de migración/schema

## DB-CLI-051

Schema Diff ≠ Migration.

## DB-CLI-052

Observed Schema ≠ Declared Schema.

## DB-CLI-053

Not Observed ≠ Absent.

## DB-CLI-054

Rollback ≠ automatic inverse SQL.

## DB-CLI-055

Migration CLI usará Migration Planner.

## DB-CLI-056

Migration CLI usará Migration Safety.

## DB-CLI-057

Migration CLI no administrará manualmente repository tables.

## DB-CLI-058

Dry-run no mutará schema.

## DB-CLI-059

Migration outcome UNKNOWN será preservado.

## DB-CLI-060

Multi-database migration no fingirá atomicidad global.

---

# 266. Invariantes de backup/restore

## DB-CLI-061

Backup Created ≠ Verified.

## DB-CLI-062

Backup Verified ≠ Restorable.

## DB-CLI-063

Restore usará preflight.

## DB-CLI-064

Restore verificará target.

## DB-CLI-065

Restore será operación de alto riesgo.

## DB-CLI-066

Restore no utilizará runtime credentials por defecto.

## DB-CLI-067

Backup tools nativos estarán detrás de providers.

## DB-CLI-068

CLI no dependerá directamente de `pg_dump`.

## DB-CLI-069

CLI no dependerá directamente de `mysqldump`.

## DB-CLI-070

Restore UNKNOWN será representable.

---

# 267. Invariantes de capability

## DB-CLI-071

Version ≠ Capability.

## DB-CLI-072

UNKNOWN ≠ UNSUPPORTED.

## DB-CLI-073

Capability evidence podrá explicarse.

## DB-CLI-074

Capability conflicts serán visibles.

## DB-CLI-075

Command requirements podrán expresarse mediante capabilities.

## DB-CLI-076

Unsupported feature fallará antes de ejecución cuando sea demostrable.

## DB-CLI-077

Unknown capability no será presentada como unsupported.

## DB-CLI-078

Capability discovery no estará implementado en CLI.

## DB-CLI-079

CLI consumirá Capability System.

## DB-CLI-080

Platform-specific evidence podrá mostrarse sin convertirse en API universal.

---

# 268. Invariantes runtime/resource

## DB-CLI-081

Long-running commands tendrán resource governance.

## DB-CLI-082

Streaming evitará buffering innecesario.

## DB-CLI-083

IdentityMap no crecerá indefinidamente.

## DB-CLI-084

DiagnosticCollector será bounded.

## DB-CLI-085

Connection budgets serán respetados.

## DB-CLI-086

Concurrency será limitada.

## DB-CLI-087

SIGINT tendrá manejo explícito cuando sea posible.

## DB-CLI-088

SIGTERM tendrá manejo explícito cuando sea posible.

## DB-CLI-089

Cancellation no fabricará outcome.

## DB-CLI-090

Cleanup failure será visible.

---

# 269. Invariantes de extensibilidad

## DB-CLI-091

Plugins podrán registrar comandos.

## DB-CLI-092

Registry será frozen tras bootstrap.

## DB-CLI-093

Duplicate command names fallarán.

## DB-CLI-094

Plugins no reemplazarán silenciosamente comandos core.

## DB-CLI-095

Command metadata será validada.

## DB-CLI-096

Extension hooks serán deterministas.

## DB-CLI-097

Extensions respetarán safety.

## DB-CLI-098

Extensions respetarán authorization.

## DB-CLI-099

Extensions respetarán diagnostics.

## DB-CLI-100

Extension failure no redefinirá un outcome ya confirmado.

---

# 270. Invariantes de testing

## DB-CLI-101

Input parsing tendrá unit tests.

## DB-CLI-102

Exit codes tendrán contract tests.

## DB-CLI-103

JSON output tendrá schema tests.

## DB-CLI-104

Safety tendrá tests.

## DB-CLI-105

Authorization tendrá tests.

## DB-CLI-106

Secret redaction tendrá tests.

## DB-CLI-107

Production protection tendrá tests.

## DB-CLI-108

Integration tests usarán DB real cuando corresponda.

## DB-CLI-109

Fake DB no probará vendor behavior.

## DB-CLI-110

Signal/cancellation behavior tendrá tests donde sea posible.

---

# 271. Invariantes finales

## DB-CLI-111

CLI será automation-friendly.

## DB-CLI-112

CLI será human-friendly.

## DB-CLI-113

Ambos modos compartirán la misma semántica.

## DB-CLI-114

CLI nunca será fuente de verdad de Database.

## DB-CLI-115

Database services serán reutilizables sin CLI.

## DB-CLI-116

Diagnostics serán compartidos con otras superficies.

## DB-CLI-117

Audit será compartido con otras superficies.

## DB-CLI-118

Telemetry será opcional.

## DB-CLI-119

Correctness tendrá prioridad sobre conveniencia.

## DB-CLI-120

Safe defaults tendrán prioridad sobre shortcuts destructivos.

---

# 272. Anti-patterns

## 272.1 SQL directo dentro del comando

```php
$pdo = new PDO(...);
$pdo->exec('ALTER TABLE ...');
```

**Prohibido como arquitectura normal.**

---

## 272.2 Migraciones implementadas por CLI

```php
foreach ($files as $migration) {
    // manual migration engine
}
```

Debe utilizar MigrationSystem.

---

## 272.3 `--force` universal

```text
--force
→ disable every safety check
```

**Prohibido.**

---

## 272.4 Environment como única protección

```php
if (APP_ENV !== 'production') {
    dropDatabase();
}
```

Insuficiente.

---

## 272.5 Mostrar secrets

```text
Database password: secret123
```

Prohibido.

---

## 272.6 Raw SQL como API principal

```text
database:anything --sql="..."
```

no deberá sustituir APIs semánticas.

---

## 272.7 Ignorar UNKNOWN

```text
UNKNOWN
→ UNSUPPORTED
```

Prohibido.

---

## 272.8 Prometer atomicidad falsa

```text
Migrating 500 tenant databases atomically...
```

sin protocolo distribuido real.

---

## 272.9 Catch genérico

```php
catch (Throwable $e) {
    echo 'Error';
    return 1;
}
```

pierde diagnóstico estructurado.

---

## 272.10 Salida JSON contaminada

```text
Starting...
████████
{"status":"ok"}
```

incompatible con automatización.

---

## 272.11 Confirmación como seguridad

```text
Are you sure? yes
→ bypass everything
```

incorrecto.

---

## 272.12 Admin credentials para todo

Viola least privilege.

---

## 272.13 Queries ilimitadas

```text
SELECT * FROM huge_table
```

impresas directamente al terminal.

---

## 272.14 CLI-specific business logic

La lógica no deberá quedar inaccesible a HTTP, tests, Workflows o futuras interfaces.

---

# 273. Ejemplo completo — migración segura

Usuario:

```text
php voltstack database:migrate
```

Flujo:

```text
DatabaseMigrateCommand
        ↓
MigrationCommandRequest
        ↓
DatabaseContextResolver
        ↓
MigrationApplicationService
        ↓
MigrationDiscovery
        ↓
MigrationPlanner
        ↓
Capability System
        ↓
Migration Safety
        ↓
Authorization
        ↓
Confirmation
        ↓
Migration Executor
        ↓
Migration Repository
        ↓
Diagnostic Collection
        ↓
Command Result
        ↓
Console Renderer
```

La CLI únicamente coordina la interacción.

---

# 274. Ejemplo completo — producción

```text
Environment
  production

Database
  primary

Pending migrations
  2

Migration #1
  ADD COLUMN
  Safety: SAFE

Migration #2
  DROP COLUMN
  Safety: DESTRUCTIVE
```

Resultado:

```text
Execution blocked.

Migration:
202609210002_remove_legacy_code

Diagnostic:
DB-MIG-SAFETY-004

Reason:
The migration contains a destructive schema operation that is not authorized by the active production policy.

Outcome:
NOT_EXECUTED
```

No importa que se utilice:

```text
--yes
```

si la policy exige una autorización adicional.

---

# 275. Ejemplo completo — multi-tenant

```text
php voltstack database:migrate \
    --all-tenants \
    --concurrency=4
```

Flujo:

```text
Tenant Enumeration
       ↓
Authorization
       ↓
Target Resolution
       ↓
Resource Governance
       ↓
Migration per Tenant
       ↓
Bounded concurrency
       ↓
Aggregated Result
```

Salida:

```text
Tenant migration summary

Targets       100
Succeeded      97
Failed          2
Unknown         1

Outcome
  PARTIAL_SUCCESS
```

El tenant con resultado `UNKNOWN` no deberá mezclarse con los dos fallidos.

---

# 276. Ejemplo completo — backup

```text
php voltstack database:backup
```

Resultado:

```text
Backup completed.

Artifact
  backup_20260921_091500

Creation
  SUCCESS

Integrity verification
  SUCCESS

Restore test
  NOT_PERFORMED

Restorable
  UNKNOWN

Duration
  48.2s
```

Esto evita el mensaje engañoso:

```text
Backup is fully restorable.
```

sin evidencia.

---

# 277. Ejemplo completo — health JSON

```text
php voltstack database:health --format=json
```

Salida conceptual:

```json
{
  "schema_version": 1,
  "command": "database:health",
  "status": "degraded",
  "checks": [
    {
      "name": "connection",
      "status": "pass"
    },
    {
      "name": "transactions",
      "status": "pass"
    },
    {
      "name": "replica-lag",
      "status": "warning"
    }
  ],
  "diagnostics": [
    {
      "code": "DB-REPLICA-004",
      "severity": "warning"
    }
  ]
}
```

---

# 278. Experiencia objetivo

VoltStack deberá buscar una experiencia similar a:

```text
$ php voltstack database:migrate --dry-run

VoltStack Database
────────────────────────────────────────

Connection      primary
Platform        PostgreSQL
Environment     production

Migration Plan
────────────────────────────────────────

✓ 202609210001_add_invoice_status
! 202609210002_drop_legacy_reference

Operations      2
Safe            1
Review           0
Destructive      1

Execution
  DRY RUN — no database changes were made.

Diagnostics
  DB-MIG-SAFETY-004
  Destructive schema operation detected.
```

La información visual es responsabilidad del renderer.

La semántica proviene del Database Engine.

---

# 279. Filosofía final

La CLI deberá representar una interfaz humana y automatizable sobre la arquitectura de Database.

No deberá convertirse en:

```text
collection of scripts
```

sino en:

```text
Database Architecture
        ↓
Application Services
        ↓
CLI Commands
        ↓
Structured Results
        ↓
Human / Machine Presentation
```

---

# 280. Regla arquitectónica definitiva

> **Toda operación disponible mediante Database CLI deberá conservar exactamente las mismas invariantes, políticas, capacidades, seguridad, contexto y semántica que tendría al invocarse mediante las APIs internas de VoltStack. La existencia de una interfaz CLI nunca constituirá una vía alternativa para omitir las reglas del Database Engine.**

Formalmente:

```text
CLI Operation Semantics
=
Database Service Semantics
```

y:

```text
CLI Convenience
∩
Database Correctness
=
Database Correctness
```

Nunca:

```text
CLI Convenience
>
Safety
```

---

# 281. Arquitectura final resumida

```text
                    VoltStack CLI
                         │
                         ▼
               Database Command Registry
                         │
                         ▼
                 Database Command
                         │
                         ▼
                  Typed Input Model
                         │
                         ▼
               CLI Context Resolution
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       Environment    Database      Tenant/Shard
            │            │            │
            └────────────┼────────────┘
                         ▼
                   Authorization
                         │
                         ▼
                  Safety System
                         │
                         ▼
               Application Service
                         │
      ┌──────────────────┼────────────────────┐
      ▼                  ▼                    ▼
 Query Engine       Schema/Migration          ORM
      │                  │                    │
      ├──────────────────┼────────────────────┤
      ▼                  ▼                    ▼
 Backup/Restore       Health              Administration
      │                  │                    │
      └──────────────────┼────────────────────┘
                         ▼
                   Command Result
                         │
              ┌──────────┼───────────┐
              ▼          ▼           ▼
          Diagnostics  Telemetry    Audit
              │
              ▼
                  Output Formatter
                         │
              ┌──────────┼───────────┐
              ▼          ▼           ▼
            Human       JSON       JSONL
                         │
                         ▼
                     Exit Code
```

---

# 282. Resultado arquitectónico

Con este sistema, VoltStack Database obtiene una CLI que podrá utilizarse para:

```text
Development
Testing
CI/CD
Operations
Administration
Diagnostics
Automation
Disaster Recovery
Performance Engineering
```

sin crear un segundo Database Engine.

La CLI se convierte en:

> **una superficie segura, estructurada y extensible para operar las capacidades reales de `VoltStack/Quantum/Database`.**

---

# 283. Siguiente documento

```text
310_DATABASE_CODE_GENERATION_SYSTEM.md
```

El siguiente documento definirá la arquitectura de generación de código de Database para producir de forma segura y extensible elementos como:

```text
Entity
Model
Repository
Migration
Seeder
Factory
Fixture
Custom Type
Database Extension
Driver skeleton
Dialect skeleton
Compiler extension
```

manteniendo la regla:

```text
Code Generator
≠
Runtime Database Engine
```

y definiendo generación basada en templates, metadata, schema introspection, naming conventions, dry-run, conflict detection, extensiones y protección contra sobrescritura accidental.