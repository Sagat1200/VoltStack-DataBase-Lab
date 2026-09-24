# 328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md

## 1. Propósito

Este documento define la arquitectura oficial de **VoltStack/Quantum/Database** para migrar aplicaciones y capas de persistencia **legacy** que no utilizan un ORM moderno estandarizado o cuya arquitectura de acceso a datos ha evolucionado durante años mediante código personalizado.

El componente se denomina conceptualmente:

```text
LegacyDatabaseMigrationSystem
```

y extiende la arquitectura general definida en:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
```

Mientras los documentos:

```text
326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md
327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md
```

trabajan con ecosistemas cuya semántica es relativamente conocida, este sistema aborda escenarios donde la fuente puede contener:

```text
PDO directo
mysqli
extensiones históricas de PHP
SQL embebido
helpers personalizados
Active Record propio
Data Mapper propio
Table Gateway
Stored Procedures
DAO
repositories legacy
database service locators
global connections
static database classes
mixed persistence architectures
```

El objetivo es permitir una migración incremental hacia VoltStack Database sin exigir una reescritura completa de la aplicación.

---

## 2. Problema

Una aplicación legacy rara vez posee una única capa de persistencia claramente delimitada.

Es común encontrar:

```text
Application
│
├── PDO
├── custom DB class
├── static SQL helpers
├── stored procedures
├── legacy repositories
├── raw SQL in controllers
├── queries in templates
├── cron scripts
└── direct database access from services
```

Por tanto:

```text
Legacy Migration
!=
ORM Adapter Migration
```

El primer problema es descubrir qué constituye realmente la capa de datos.

---

## 3. Objetivo principal

Transformar progresivamente:

```text
Legacy Application
      │
      ▼
Fragmented Persistence
      │
      ▼
Database
```

en:

```text
Application
      │
      ▼
Defined Persistence Boundaries
      │
      ▼
VoltStack Database
      │
      ▼
Database
```

preservando primero el comportamiento y mejorando la arquitectura después.

---

## 4. Principio fundamental

```text
Preserve behavior first.
Modernize architecture second.
Optimize third.
```

No se deberán realizar simultáneamente, salvo necesidad explícita:

```text
database migration
schema redesign
domain redesign
query optimization
framework migration
business logic rewrite
```

---

## 5. Alcance

El sistema deberá analizar:

```text
PDO
mysqli
legacy mysql APIs in historical code
custom connection wrappers
database singletons
service locators
global connections
DAO classes
Table Gateways
Active Record implementations
Data Mappers
Repositories
SQL strings
query builders
stored procedures
database functions
database triggers
transactions
manual locking
schema scripts
SQL migration files
seed scripts
batch scripts
cron jobs
CLI scripts
reporting queries
database views
temporary tables
dynamic SQL
```

---

## 6. No objetivo

El sistema no pretende convertir automáticamente cualquier código PHP arbitrario en una arquitectura moderna.

Cuando no pueda determinarse la intención:

```text
MANUAL REVIEW
```

será el resultado correcto.

---

## 7. Arquitectura general

```text
Legacy Application
       │
       ▼
Discovery Engine
       │
       ▼
Persistence Pattern Detector
       │
       ▼
SQL / API Analyzer
       │
       ▼
Legacy Persistence Model
       │
       ▼
Intermediate Database Model
       │
       ▼
Migration Rule Engine
       │
       ▼
VoltStack Database
       │
       ▼
Verification
```

---

## 8. Legacy Source Adapter

El sistema implementará conceptualmente:

```php
final class LegacyDatabaseMigrationAdapter
    implements DatabaseMigrationSourceAdapterInterface
{
    public function inspect(MigrationSource $source): SourceModel;

    public function normalize(
        SourceModel $source
    ): IntermediateDatabaseModel;
}
```

A diferencia de Eloquent/Doctrine, deberá soportar múltiples patrones simultáneamente.

---

## 9. Discovery Engine

El primer paso será buscar indicadores de acceso a datos.

Ejemplos:

```text
new PDO
PDO::
mysqli_
mysqli::
mysql_
prepare(
query(
execute(
SELECT
INSERT
UPDATE
DELETE
CALL
BEGIN
COMMIT
ROLLBACK
```

También deberá buscar wrappers y abstracciones indirectas.

---

## 10. Persistence Entry Points

El scanner construirá un inventario:

```text
Persistence Entry Points

PDO Connections               4
Custom DB Wrappers            3
Repositories                 18
DAO Classes                  27
Raw SQL Locations           164
Stored Procedures            22
Transaction Boundaries       31
Unknown DB Helpers            7
```

---

## 11. Pattern Detector

El sistema intentará clasificar cada componente:

```text
DIRECT_SQL
PDO_WRAPPER
TABLE_GATEWAY
DAO
REPOSITORY
ACTIVE_RECORD
DATA_MAPPER
QUERY_OBJECT
SERVICE_LOCATOR
UNKNOWN
```

---

## 12. Mixed Architecture

Una aplicación puede contener:

```text
Module A → PDO
Module B → DAO
Module C → Active Record
Module D → Stored Procedures
```

El sistema deberá aceptar esta realidad.

No se requerirá normalizar toda la aplicación antes de comenzar la migración.

---

## 13. Static Analysis

El análisis deberá utilizar AST PHP para identificar:

```text
object construction
method calls
static calls
SQL strings
variable flow
connection creation
transaction calls
database helper calls
```

---

## 14. SQL String Detection

Se deberán detectar:

```php
$sql = 'SELECT * FROM users WHERE id = ?';
```

y:

```php
$db->query("SELECT ...");
```

incluyendo strings multilínea.

---

## 15. Dynamic SQL

Ejemplo:

```php
$sql = 'SELECT * FROM ' . $table;
```

deberá clasificarse como:

```text
DYNAMIC_SQL
```

y analizar el origen de cada fragmento.

---

## 16. SQL Taint Analysis

Cuando sea posible, el sistema deberá identificar valores provenientes de:

```text
HTTP input
CLI input
external messages
files
user-controlled variables
```

que terminen concatenados en SQL.

Esto permitirá detectar riesgos durante la migración.

---

## 17. Seguridad

Una consulta legacy insegura no deberá convertirse literalmente.

Ejemplo:

```php
$sql = "SELECT * FROM users WHERE id = $id";
```

deberá producir:

```text
SECURITY_FINDING:
UNBOUND_SQL_PARAMETER
```

y recomendar parameter binding.

---

## 18. Prepared Statements

PDO:

```php
$stmt = $pdo->prepare(
    'SELECT * FROM users WHERE id = :id'
);

$stmt->execute([
    'id' => $id,
]);
```

puede mapearse de forma relativamente directa al Connection/Query layer de VoltStack.

---

## 19. Parameter Semantics

Se deberán preservar:

```text
named parameters
positional parameters
binding order
binding types
NULL semantics
LOB handling
```

---

## 20. PDO Attributes

Configuraciones como:

```text
ATTR_ERRMODE
ATTR_EMULATE_PREPARES
ATTR_DEFAULT_FETCH_MODE
persistent connections
```

deberán inventariarse cuando alteren comportamiento observable.

---

## 21. Fetch Modes

El adapter deberá detectar dependencias sobre:

```text
FETCH_ASSOC
FETCH_NUM
FETCH_BOTH
FETCH_OBJ
FETCH_CLASS
FETCH_COLUMN
```

La forma del resultado forma parte del contrato legacy.

---

## 22. Result Shape

La migración deberá preservar inicialmente:

```text
array
object
scalar
row
collection
generator
```

antes de introducir entidades o DTOs.

---

## 23. mysqli

Código basado en `mysqli` deberá analizar:

```text
connections
prepared statements
bindings
result sets
transactions
multi_query
```

y migrarse preferentemente hacia las abstracciones nativas de VoltStack.

---

## 24. APIs históricas

Si se encuentra código extremadamente antiguo basado en APIs removidas de PHP, el scanner deberá registrarlo como:

```text
OBSOLETE_DATABASE_API
```

y requerir una ruta de modernización explícita.

---

## 25. Custom Database Wrappers

Ejemplo:

```php
final class Database
{
    public static function query(string $sql): array
    {
        // ...
    }
}
```

deberá analizarse como posible gateway central.

---

## 26. Wrapper Advantage

Si una aplicación ya centraliza acceso:

```text
Application
    │
    ▼
Legacy DB Wrapper
    │
    ▼
PDO
```

la migración puede comenzar sustituyendo internamente el wrapper:

```text
Application
    │
    ▼
Legacy Compatibility Wrapper
    │
    ▼
VoltStack Database
```

reduciendo cambios iniciales.

---

## 27. Compatibility Wrapper

Ejemplo conceptual:

```php
final class LegacyDatabaseAdapter
{
    public function __construct(
        private ConnectionInterface $connection
    ) {
    }
}
```

Esta capa deberá marcarse como temporal.

---

## 28. DAO Pattern

Clases:

```text
UserDAO
OrderDAO
InvoiceDAO
```

se analizarán para identificar:

```text
CRUD
queries
transactions
mapping
business logic leakage
```

---

## 29. DAO Migration

Flujo:

```text
Legacy DAO
    │
    ▼
Method Inventory
    │
    ▼
Query Analysis
    │
    ▼
Persistence Contract
    │
    ▼
VoltStack Repository / Query Service
```

---

## 30. Table Gateway

Clases centradas en tablas pueden migrarse inicialmente hacia:

```text
VoltStack Query Builder
```

sin obligar a utilizar ORM.

---

## 31. Active Record Legacy

Implementaciones propias de Active Record deberán analizar:

```text
save
delete
find
dirty tracking
relationships
callbacks
table resolution
```

No deberán confundirse con Eloquent.

---

## 32. Data Mapper Legacy

Un Data Mapper propio puede tener conceptos cercanos a VoltStack:

```text
mapper
identity map
unit of work
repository
```

pero sus contratos deberán analizarse antes de reutilizarlos.

---

## 33. Repository Legacy

Los repositories existentes pueden convertirse en una frontera útil.

Si su API no depende de detalles del driver, podrán mantenerse mientras se sustituye su implementación.

---

## 34. Business Logic Leakage

El scanner deberá identificar métodos donde:

```text
SQL
+
business calculations
+
side effects
```

estén mezclados.

Estos casos deberán dividirse cuidadosamente, no mediante transformación automática agresiva.

---

## 35. SQL Parser

Cuando sea posible:

```text
Raw SQL
   │
   ▼
SQL Parser
   │
   ▼
Intermediate Query AST
```

permitirá comprender consultas sin ejecutarlas.

---

## 36. SQL Dialect

El parser deberá conocer la plataforma:

```text
MySQL
MariaDB
PostgreSQL
SQLite
SQL Server
```

porque una misma cadena puede contener sintaxis específica.

---

## 37. Vendor-Specific SQL

Ejemplos:

```text
ON DUPLICATE KEY UPDATE
RETURNING
TOP
LIMIT
JSON operators
window functions
vendor functions
```

deberán conservar su clasificación de plataforma.

---

## 38. No Forced Portability

Migrar a VoltStack no significa obligatoriamente convertir todo SQL a SQL portable.

```text
working platform-specific SQL
```

puede preservarse inicialmente.

---

## 39. SQL Classification

Cada consulta se clasificará:

```text
SELECT
INSERT
UPDATE
DELETE
UPSERT
DDL
TRANSACTION
PROCEDURE_CALL
FUNCTION_CALL
UNKNOWN
```

---

## 40. Query Fingerprint

El sistema podrá generar fingerprints ignorando valores:

```text
SELECT * FROM users WHERE id = ?
```

para agrupar consultas equivalentes.

---

## 41. Duplicate Query Detection

Esto permite descubrir:

```text
same logical query
implemented in 14 different locations
```

y crear un punto de migración común.

---

## 42. Runtime Query Capture

Opcionalmente podrá instrumentarse la aplicación para registrar:

```text
SQL fingerprint
source location
duration
row count
transaction context
connection
```

sin registrar valores sensibles.

---

## 43. Runtime Evidence

La evidencia runtime complementa, pero no reemplaza, el análisis estático.

Una query no observada durante pruebas puede seguir existiendo.

---

## 44. Connection Discovery

El sistema deberá identificar:

```text
DSN
driver
host source
port
database
charset
SSL
persistent mode
connection factory
```

sin exponer credenciales.

---

## 45. Global Connections

Patrones como:

```php
global $db;
```

o singletons deberán marcarse:

```text
GLOBAL_MUTABLE_DATABASE_STATE
```

---

## 46. Service Locator

Ejemplo:

```php
Database::connection();
```

deberá analizarse para conocer:

```text
lifetime
connection selection
transaction state
```

---

## 47. Connection Migration

Objetivo:

```text
manual/global connection
        │
        ▼
VoltStack ConnectionManager
```

mediante adaptación gradual.

---

## 48. Multiple Databases

El sistema deberá descubrir si la aplicación usa:

```text
main database
reporting database
audit database
legacy database
```

y preservar estas fronteras.

---

## 49. Dynamic Database Selection

Selección basada en:

```text
customer
tenant
region
request
```

deberá marcarse como dinámica y mapearse a un resolver explícito.

---

## 50. Transactions

El scanner buscará:

```text
BEGIN
START TRANSACTION
COMMIT
ROLLBACK
PDO transaction APIs
mysqli transaction APIs
custom wrappers
```

---

## 51. Transaction Boundary Graph

Podrá generarse:

```text
Service A
  │
  ├── begin
  │    ├── DAO X
  │    └── DAO Y
  └── commit
```

para comprender ownership transaccional.

---

## 52. Hidden Transactions

Stored procedures o triggers pueden iniciar o alterar comportamiento transaccional según plataforma.

Estos casos deberán documentarse.

---

## 53. Nested Transactions

Implementaciones legacy pueden simular nesting mediante:

```text
counter
savepoints
ignore nested begin
```

El migrador deberá descubrir su semántica real.

---

## 54. Transaction Safety

No se considerará equivalente una migración si cambia:

```text
atomicity
rollback scope
isolation
locking
retry behavior
```

---

## 55. Locking

Se detectará:

```text
SELECT ... FOR UPDATE
LOCK IN SHARE MODE
advisory locks
table locks
application locks
```

---

## 56. Manual Concurrency Control

Patrones como:

```text
SELECT version
UPDATE ... WHERE version = ?
```

pueden implementar optimistic locking manualmente.

El analyzer deberá reconocerlos cuando sea posible.

---

## 57. Stored Procedures

El sistema deberá inventariar llamadas:

```text
CALL procedure(...)
EXEC procedure
SELECT function(...)
```

---

## 58. Stored Procedure Strategy

Opciones:

```text
KEEP
WRAP
REIMPLEMENT
REMOVE
```

La estrategia por defecto será:

```text
KEEP
```

si la procedure funciona y no existe una razón arquitectónica para reemplazarla durante la migración.

---

## 59. Procedure Contracts

Se deberán documentar:

```text
parameters
output parameters
result sets
side effects
transaction behavior
required privileges
```

---

## 60. Database Functions

Funciones utilizadas desde SQL deberán preservarse como dependencias de plataforma.

---

## 61. Triggers

Los triggers son especialmente importantes porque introducen comportamiento fuera del código PHP.

Se deberá inventariar:

```text
BEFORE INSERT
AFTER INSERT
BEFORE UPDATE
AFTER UPDATE
BEFORE DELETE
AFTER DELETE
```

según plataforma.

---

## 62. Trigger Effects

Ejemplos:

```text
audit rows
timestamps
derived fields
sequence generation
denormalized totals
security checks
```

El migrador no deberá duplicar estos efectos accidentalmente en VoltStack.

---

## 63. Views

Se deberán descubrir:

```text
views
materialized views
```

cuando la aplicación las consulte como tablas.

---

## 64. View Mapping

Una view podrá permanecer como:

```text
read-only query source
```

sin convertirse necesariamente en entidad persistible.

---

## 65. Temporary Tables

El analyzer deberá detectar:

```text
CREATE TEMPORARY TABLE
temp tables
session-scoped tables
```

porque dependen del lifecycle de la conexión.

---

## 66. Persistent Runtime Risk

Con FrankenPHP, una conexión reutilizada puede cambiar las expectativas sobre recursos session-scoped.

La migración deberá verificar cleanup de:

```text
temporary tables
session variables
locks
transactions
```

---

## 67. Session Variables

SQL como:

```text
SET ...
```

deberá inventariarse.

El Connection Manager deberá saber qué estado necesita reset entre requests.

---

## 68. Schema Discovery

El sistema deberá inspeccionar:

```text
tables
columns
types
indexes
constraints
foreign keys
views
triggers
sequences
generated columns
```

---

## 69. Missing Constraints

Aplicaciones legacy pueden mantener integridad solo en código.

Ejemplo:

```text
user_id
```

sin foreign key física.

El migrador no deberá crear una FK automáticamente durante la sustitución de persistence layer.

---

## 70. Schema Redesign Separation

Regla:

```text
Legacy Persistence Migration
            ≠
Schema Modernization
```

Agregar constraints será una decisión posterior y explícita.

---

## 71. Schema Scripts

Se deberán localizar:

```text
.sql files
install scripts
upgrade scripts
manual DBA scripts
```

como parte del historial de schema.

---

## 72. Unknown Schema History

Si no existe historial confiable:

```text
Actual Database Schema
```

se convierte en la referencia operacional principal para generar un baseline.

---

## 73. Baseline Strategy

```text
Production-like Schema
        │
        ▼
Schema Introspection
        │
        ▼
VoltStack Baseline
        │
        ▼
Future VoltStack Migrations
```

---

## 74. Data Mapping

Código legacy puede mapear filas manualmente:

```php
return new User(
    $row['id'],
    $row['name']
);
```

El analyzer deberá identificar estos mappers.

---

## 75. Hydration Contract

Se deberá preservar:

```text
column aliases
constructor order
NULL handling
type conversion
date conversion
boolean conversion
```

---

## 76. Manual Type Conversion

Ejemplo:

```php
$active = (bool) $row['active'];
```

es evidencia de una conversión que deberá representarse en el modelo intermedio.

---

## 77. Date Handling

Código legacy puede usar:

```text
string dates
timestamps
DateTime
custom date classes
timezone conversion
```

El migrador deberá identificar la representación real.

---

## 78. Decimal Handling

Especial atención a:

```text
money
decimal
numeric
float
```

y código que convierte valores a `float`.

No deberá introducirse pérdida de precisión.

---

## 79. Boolean Handling

Valores como:

```text
0/1
Y/N
T/F
yes/no
```

pueden representar booleanos.

No deberán normalizarse sin evidencia.

---

## 80. NULL Semantics

Código legacy puede convertir:

```text
NULL → ''
NULL → 0
NULL → false
```

El comportamiento deberá detectarse antes de cambiarlo.

---

## 81. Serialization

Columnas pueden contener:

```text
PHP serialize()
JSON
CSV
custom delimiter formats
encrypted payloads
```

El migrador deberá preservar codecs inicialmente.

---

## 82. Encrypted Fields

Se deberá detectar, cuando sea posible:

```text
encrypt before insert
decrypt after select
```

La migración no deberá modificar el formato criptográfico como efecto colateral.

---

## 83. Binary Data

BLOBs y streams requieren preservar:

```text
streaming
memory behavior
binding types
binary encoding
```

---

## 84. Large Result Sets

Código legacy puede procesar millones de filas mediante:

```text
unbuffered queries
cursor loops
pagination
ID windows
```

La migración deberá mantener las características de memoria.

---

## 85. Batch Jobs

Scripts CLI y cron deberán formar parte del análisis.

Muchas aplicaciones legacy contienen su acceso más crítico a datos fuera del request HTTP.

---

## 86. Cron Discovery

Se deberán buscar:

```text
bin/
scripts/
cron/
commands/
jobs/
```

y configuraciones disponibles cuando formen parte del proyecto.

---

## 87. Reporting Queries

Consultas de reporting suelen ser:

```text
large
read-only
aggregate-heavy
vendor-specific
```

Pueden mantenerse como SQL nativo detrás de Query Services.

---

## 88. Query Service

Una ruta de modernización útil:

```text
Raw Reporting SQL
        │
        ▼
VoltStack Query Service
        │
        ▼
Connection
```

sin forzar ORM.

---

## 89. Command Query Separation

La migración podrá identificar oportunidades para separar:

```text
write persistence
read/reporting queries
```

pero esto será una recomendación, no una reescritura automática.

---

## 90. Database Error Handling

El scanner deberá descubrir patrones como:

```php
catch (PDOException $e)
```

o códigos SQL específicos.

---

## 91. Error Semantics

Los errores deberán mapearse por intención:

```text
unique violation
foreign key violation
deadlock
timeout
connection failure
syntax error
serialization failure
```

---

## 92. Error Code Dependency

Código que compara:

```text
SQLSTATE
vendor error code
message text
```

deberá marcarse como dependencia específica de plataforma.

---

## 93. Retry Logic

El analyzer deberá detectar retries manuales:

```text
deadlock retry
connection retry
transaction retry
```

para evitar duplicarlos o eliminarlos.

---

## 94. Logging

Código legacy puede registrar SQL completo con parámetros.

El migrador deberá señalar riesgos de:

```text
passwords
tokens
PII
financial data
```

y adoptar la política de redacción de VoltStack.

---

## 95. Auditing

Tablas o procedimientos de auditoría deberán identificarse para evitar que el nuevo runtime omita eventos o genere duplicados.

---

## 96. Security Model

La migración deberá mejorar fronteras de seguridad sin cambiar funcionalidad innecesariamente.

Prioridades:

```text
parameter binding
credential isolation
least privilege
safe logging
connection separation
```

---

## 97. Credentials

El scanner podrá identificar ubicaciones de configuración, pero nunca deberá copiar secretos a reportes.

---

## 98. Hard-coded Credentials

Si detecta credenciales embebidas:

```text
SECURITY_FINDING:
HARDCODED_DATABASE_CREDENTIAL
```

El valor deberá redactarse.

---

## 99. Dynamic Credentials

Sistemas que obtienen credenciales por:

```text
vault
environment
tenant resolver
external secret manager
```

deberán conservar ese mecanismo mediante un adapter/configuration provider.

---

## 100. Runtime Analysis

Cuando el código sea demasiado dinámico podrá habilitarse un observer.

```text
Legacy Runtime
      │
      ▼
Database Observer
      │
      ├── connection usage
      ├── SQL fingerprints
      ├── transactions
      ├── latency
      └── source locations
```

---

## 101. Runtime Safety

El observer deberá:

```text
avoid query mutation
avoid result mutation
sanitize values
limit memory
support sampling
```

---

## 102. Query Coverage

El reporte deberá distinguir:

```text
statically discovered
runtime observed
both
```

No deberá afirmar cobertura total solo por tráfico observado.

---

## 103. Legacy Persistence Model

La salida especializada podrá contener:

```text
LegacyConnectionDescriptor
LegacyQueryDescriptor
LegacyTransactionDescriptor
LegacyMapperDescriptor
LegacyRepositoryDescriptor
LegacyProcedureDescriptor
LegacyTriggerDescriptor
LegacyPersistenceEntryPoint
```

---

## 104. Normalización

Estos descriptors se convertirán hacia:

```text
Intermediate Connection
Intermediate Query
Intermediate Entity/Row Model
Intermediate Transaction
Intermediate Procedure Dependency
Intermediate Schema Dependency
```

---

## 105. No Entity Requirement

VoltStack Database no deberá exigir crear una entidad para cada query legacy.

Una migración válida puede ser:

```text
PDO SQL
   ↓
VoltStack Connection SQL
```

---

## 106. Progressive Abstraction

Posteriormente:

```text
VoltStack Connection SQL
        ↓
Query Builder
        ↓
Repository / Entity
```

si aporta valor.

---

## 107. Migration Levels

El sistema podrá describir niveles:

```text
L0 - Legacy untouched
L1 - Connection migrated
L2 - Transactions migrated
L3 - SQL centralized
L4 - Query Builder / Query Services
L5 - Repository boundaries
L6 - ORM/Entity model where appropriate
```

No todos los módulos necesitan llegar a L6.

---

## 108. SQL-first Architecture

VoltStack deberá reconocer que algunos sistemas funcionan mejor con:

```text
SQL-first
```

especialmente:

```text
analytics
reporting
ETL
bulk processing
complex vendor queries
```

La migración no deberá imponer ORM universal.

---

## 109. Compatibility Layer

Durante transición:

```text
Legacy API
    │
    ▼
Compatibility Adapter
    │
    ▼
VoltStack Database
```

permitirá reducir el blast radius.

---

## 110. Compatibility Layer Rules

Debe ser:

```text
temporary
observable
deprecated
testable
removable
```

y nunca convertirse accidentalmente en una segunda API permanente.

---

## 111. Deprecation Integration

Las APIs legacy encapsuladas podrán integrarse con:

```text
324_DATABASE_DEPRECATION_POLICY.md
```

para registrar uso restante.

---

## 112. Legacy Usage Counter

Cada llamada al compatibility adapter podrá incrementar métricas:

```text
database.legacy.calls
database.legacy.query_calls
database.legacy.transaction_calls
```

---

## 113. Migration Burn-down

Ejemplo:

```text
Week 1   42,000 legacy calls/day
Week 4   18,000
Week 8    3,200
Week 12       0
```

Esto permite medir progreso real.

---

## 114. Source Location Tracking

Cada uso legacy podrá asociarse con:

```text
file
line
class
method
module
```

sin capturar datos sensibles.

---

## 115. Dependency Graph

El sistema construirá:

```text
Module
  │
  ▼
Legacy Service
  │
  ▼
DAO
  │
  ▼
DB Wrapper
  │
  ▼
PDO
```

para encontrar puntos de corte óptimos.

---

## 116. Migration Seams

Un **migration seam** es un punto donde puede sustituirse infraestructura sin cambiar el contrato superior.

Ejemplo:

```text
Service
   │
   ▼
UserDAO   ← seam
   │
   ▼
PDO
```

---

## 117. Seam Detection

El analyzer podrá sugerir:

```text
wrapper seam
repository seam
DAO seam
connection seam
query-service seam
```

---

## 118. Migration Strategy Selection

Según el código:

```text
Central Wrapper
→ replace wrapper internals

DAO Layer
→ migrate DAO implementations

Raw SQL Everywhere
→ introduce query gateway

Custom Active Record
→ adapter + gradual model migration

Stored Procedure Heavy
→ preserve procedures + migrate callers
```

---

## 119. Module-by-Module Migration

El sistema deberá permitir:

```text
Users → VoltStack
Billing → Legacy
Reports → Legacy SQL
```

hasta completar el proceso.

---

## 120. Ownership

Cada tabla/agregado deberá tener ownership operacional claro durante Dual Runtime.

---

## 121. Shared Table Risk

Dos capas escribiendo la misma tabla pueden generar:

```text
different casting
different defaults
different lifecycle
different transactions
```

Por tanto deberán verificarse explícitamente.

---

## 122. Read-first Migration

Una estrategia posible:

```text
Legacy writes
Legacy reads
      ↓
VoltStack reads
      ↓
VoltStack writes
```

---

## 123. Shadow Reads

Podrá ejecutarse:

```text
Legacy Query ───► Result A
VoltStack Query ─► Result B
                      │
                      ▼
                  Comparator
```

para validar consultas críticas.

---

## 124. Shadow Writes

No se habilitarán por defecto en producción.

Deberán limitarse preferentemente a:

```text
test
staging
replicas
disposable databases
```

---

## 125. Result Comparator

Deberá comparar:

```text
row count
column names
values
types
NULLs
ordering
aggregates
```

con normalización controlada.

---

## 126. Golden Dataset

Para código difícil de probar se recomienda crear datasets deterministas que cubran:

```text
normal cases
NULLs
boundaries
unicode
large numbers
dates
decimals
duplicates
foreign keys
```

---

## 127. Characterization Tests

Antes de cambiar una función legacy:

```text
Current behavior
      │
      ▼
Characterization Test
      │
      ▼
Migration
      │
      ▼
Same Test
```

---

## 128. Unknown Behavior

Si no existe documentación, los tests de caracterización serán especialmente importantes.

---

## 129. Migration Rule IDs

Las reglas legacy usarán:

```text
VSDB-MIG-LEGACY-0001
VSDB-MIG-LEGACY-0002
...
```

---

## 130. Clasificación

Cada hallazgo:

```text
DIRECT
TRANSFORMABLE
ADAPTABLE
MANUAL
UNSUPPORTED
RISKY
```

---

## 131. Ejemplo PDO

Origen:

```php
$stmt = $pdo->prepare(
    'SELECT id, name FROM users WHERE id = :id'
);

$stmt->execute(['id' => $id]);

$user = $stmt->fetch(PDO::FETCH_ASSOC);
```

Representación:

```text
Query Type:
SELECT

Source:
users

Predicate:
id = :id

Result Shape:
ASSOCIATIVE_ROW
```

---

## 132. Migración inicial PDO

Una primera migración válida puede ser:

```text
PDO
 ↓
VoltStack Connection
```

manteniendo SQL y result shape.

---

## 133. Migración posterior

Opcionalmente:

```text
VoltStack Connection
       ↓
Query Builder
       ↓
Repository
```

---

## 134. Ejemplo DAO

Origen:

```php
final class UserDAO
{
    public function find(int $id): ?array
    {
        // raw SQL
    }
}
```

Ruta:

```text
keep UserDAO contract
        │
        ▼
replace implementation
        │
        ▼
VoltStack Connection/QueryBuilder
```

Después podrá decidirse si sustituir el DAO por un repository nativo.

---

## 135. Example Dynamic SQL Finding

```text
[VSDB-MIG-LEGACY-0041]

File:
src/Reports/ReportBuilder.php

Finding:
Dynamic table identifier.

Expression:
"SELECT * FROM " . $table

Risk:
Identifier source cannot be proven safe.

Action:
Introduce an allow-listed identifier resolver
before migration.
```

---

## 136. Example Transaction Finding

```text
[VSDB-MIG-LEGACY-0068]

Transaction begins:
OrderService.php:84

Commit:
OrderService.php:142

Calls:
OrderDAO
PaymentDAO
AuditDAO

Risk:
Three persistence components participate in
one transaction boundary.
```

---

## 137. CLI

Integración:

```bash
php volt database:migrate-orm \
    --from=legacy \
    --analyze
```

Aunque el comando general conserve el nombre `migrate-orm`, el Migration System deberá aceptar fuentes no ORM.

Una futura alias de DX podrá exponer:

```bash
php volt database:migrate-legacy
```

sin duplicar el motor.

---

## 138. Scope por directorio

```bash
php volt database:migrate-legacy \
    --path=src/Billing \
    --analyze
```

---

## 139. SQL Inventory

```bash
php volt database:migrate-legacy \
    --sql-inventory
```

podrá generar un inventario sin transformar código.

---

## 140. Connection Inventory

```bash
php volt database:migrate-legacy \
    --connections
```

podrá mostrar conexiones y puntos de creación sanitizados.

---

## 141. Transaction Inventory

```bash
php volt database:migrate-legacy \
    --transactions
```

podrá mostrar fronteras detectadas.

---

## 142. Dry Run

```bash
php volt database:migrate-legacy \
    --dry-run
```

no modificará source ni schema.

---

## 143. Configuration

Conceptualmente:

```php
return [
    'legacy' => [
        'paths' => [
            'src',
            'scripts',
        ],

        'runtime_analysis' => false,

        'sql_dialect' => 'auto',

        'detect_security_findings' => true,
    ],
];
```

---

## 144. Custom Wrapper Definitions

El usuario podrá indicar:

```php
'wrappers' => [
    Legacy\Database::class => [
        'query_method' => 'query',
        'transaction_method' => 'transaction',
    ],
];
```

para ayudar al analyzer.

---

## 145. Custom Pattern Plugins

Podrán implementarse:

```php
final class CompanyLegacyDbPattern
    implements LegacyPersistencePatternInterface
{
}
```

---

## 146. Framework-independent

El Legacy Migration System no dependerá de:

```text
Laravel
Symfony
Doctrine
```

aunque pueda coexistir con sus adapters.

---

## 147. Hybrid Source

Una aplicación puede tener:

```text
Doctrine
+
PDO legacy
+
stored procedures
```

El Migration Analysis Engine deberá combinar findings de múltiples adapters.

---

## 148. Source Precedence

Ningún adapter será considerado automáticamente fuente de verdad superior.

Los conflictos deberán resolverse explícitamente.

---

## 149. Migration Analysis Integration

El análisis global se especificará en:

```text
329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md
```

---

## 150. Intermediate Model Integration

La representación neutral se formalizará en:

```text
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
```

---

## 151. Rule Engine Integration

Las transformaciones se coordinarán mediante:

```text
331_DATABASE_MIGRATION_RULE_ENGINE.md
```

---

## 152. Code Transformer Integration

Las modificaciones automáticas de source pertenecerán a:

```text
332_DATABASE_MIGRATION_CODE_TRANSFORMER.md
```

---

## 153. Schema Compatibility

La validación de schema se desarrollará en:

```text
333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md
```

---

## 154. Behavior Verification

La equivalencia se desarrollará en:

```text
334_DATABASE_MIGRATION_BEHAVIOR_VERIFICATION_SYSTEM.md
```

---

## 155. Dual Runtime

La coexistencia se desarrollará en:

```text
335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md
```

El nombre histórico del documento cubre también capas legacy no ORM.

---

## 156. Shadow Comparison

La comparación formal se desarrollará en:

```text
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
```

---

## 157. Testing

Las pruebas completas se desarrollarán en:

```text
337_DATABASE_MIGRATION_TESTING_AND_VALIDATION_SYSTEM.md
```

---

## 158. CLI / DX

La experiencia de migración se desarrollará en:

```text
338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md
```

---

## 159. Reporting

Los reportes y diagnósticos se desarrollarán en:

```text
339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md
```

---

## 160. Rollback

La recuperación se desarrollará en:

```text
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 161. FrankenPHP

Como runtime predeterminado de VoltStack, FrankenPHP exige especial atención a código legacy diseñado para:

```text
one request
=
one PHP process lifecycle
```

---

## 162. Persistent Worker Audit

Se deberán detectar:

```text
global PDO
static connection
static transaction flags
static current tenant
session variables
temporary tables
advisory locks
unclosed cursors
```

---

## 163. Request Cleanup

VoltStack deberá garantizar:

```text
rollback unfinished transactions
release request-local resources
reset session state where required
clear tenant context
clear migration observer context
```

---

## 164. RoadRunner / OpenSwoole

Los mismos contratos de aislamiento aplicarán a sus paquetes oficiales.

---

## 165. Performance Baseline

Antes de migrar se recomienda capturar:

```text
queries/request
database latency
slow queries
memory
rows read
transaction duration
connection count
```

---

## 166. Performance Preservation

El primer objetivo no será optimizar cada query.

Será evitar regresiones graves mientras se sustituye infraestructura.

---

## 167. Optimization Separation

Después de verificar equivalencia:

```text
Migration Complete
      │
      ▼
Performance Optimization
```

podrá ejecutarse como fase independiente.

---

## 168. Observability

El sistema podrá emitir:

```text
database.legacy.calls
database.legacy.queries
database.legacy.raw_sql
database.legacy.dynamic_sql
database.legacy.security_findings
database.legacy.transaction_findings
database.legacy.migrated_calls
```

---

## 169. Telemetry Privacy

Nunca se deberán incluir automáticamente:

```text
SQL parameter values
credentials
personal data
tokens
```

en telemetry.

---

## 170. CI Integration

Durante la migración podrá existir una política:

```text
Current legacy usages <= baseline
```

Si aumentan:

```text
CI FAIL
```

---

## 171. Legacy Baseline

Ejemplo:

```text
legacy-database.baseline
```

registrará deuda existente sin permitir deuda nueva.

---

## 172. Architecture Tests

Podrán establecerse reglas:

```text
New code MUST NOT instantiate PDO directly.
New modules MUST NOT use global DB helpers.
New code MUST use VoltStack persistence contracts.
```

---

## 173. Migration Progress

Ejemplo:

```text
Legacy Persistence Migration

Entry points discovered       248
Migrated                      181
Compatibility adapter          34
Manual review                  21
Blocked                        12
```

---

## 174. Completion Criteria

La migración podrá considerarse completa cuando:

```text
0 uncontrolled direct connections
0 unsupported legacy DB APIs
0 unresolved critical transaction boundaries
0 unresolved critical SQL injection findings
0 required compatibility adapters
all active queries accounted for
schema verification passes
behavior verification passes
test suite passes
```

No significa que todo SQL deba haberse convertido a ORM.

---

## 175. Important Completion Principle

Una aplicación puede finalizar correctamente con:

```text
VoltStack ORM
+
VoltStack Query Builder
+
VoltStack Native SQL
+
Stored Procedures
```

si todas esas rutas están controladas por la infraestructura Database.

---

## 176. Arquitectura final

```text
┌──────────────────────────────────────────────────────┐
│                  Legacy Application                  │
├──────────────────────────────────────────────────────┤
│ PDO │ mysqli │ DAO │ SQL │ Wrappers │ Procedures    │
└──┬─────┬──────┬─────┬───────┬───────────┬──────────┘
   │     │      │     │       │           │
   └─────┴──────┴─────┴───────┴───────────┘
                       │
                       ▼
          LegacyDatabaseMigrationSystem
                       │
      ┌────────────────┼────────────────┐
      ▼                ▼                ▼
 Discovery       SQL Analyzer     Pattern Detector
      │                │                │
      ├──────────┬─────┴──────┬─────────┤
      ▼          ▼            ▼         ▼
 Connections Transactions   Mappers  Procedures
      │          │            │         │
      └──────────┴─────┬──────┴─────────┘
                       ▼
            Legacy Persistence Model
                       │
                       ▼
                    Normalizer
                       │
                       ▼
            Intermediate Database Model
                       │
                       ▼
              Migration Rule Engine
                       │
                       ▼
               VoltStack Database
```

---

## 177. Flujo recomendado

```text
1. Discover persistence entry points
          ↓
2. Classify legacy patterns
          ↓
3. Inventory connections
          ↓
4. Inventory SQL
          ↓
5. Inventory transactions
          ↓
6. Discover procedures/triggers/views
          ↓
7. Inspect schema
          ↓
8. Identify migration seams
          ↓
9. Create characterization tests
          ↓
10. Introduce compatibility boundary
          ↓
11. Migrate connection/transactions
          ↓
12. Migrate queries module by module
          ↓
13. Verify behavior
          ↓
14. Remove legacy infrastructure
```

---

## 178. Decisiones arquitectónicas

### Decisión 1

VoltStack Database V1 incluirá una estrategia formal para migrar código legacy, no solamente ORMs conocidos.

### Decisión 2

El sistema aceptará múltiples patrones de persistencia simultáneamente.

### Decisión 3

La migración será static-first y runtime-assisted.

### Decisión 4

SQL válido podrá conservarse inicialmente.

### Decisión 5

VoltStack no obligará a convertir todo SQL a ORM.

### Decisión 6

Los wrappers y DAOs existentes podrán utilizarse como migration seams.

### Decisión 7

Las capas de compatibilidad serán temporales, observables y removibles.

### Decisión 8

La migración de persistence se separará del rediseño de schema.

### Decisión 9

Stored procedures, views y triggers se preservarán por defecto cuando formen parte del comportamiento actual.

### Decisión 10

Las vulnerabilidades SQL detectadas no deberán reproducirse silenciosamente.

### Decisión 11

Las transacciones tendrán prioridad sobre transformaciones sintácticas.

### Decisión 12

El sistema permitirá migraciones módulo por módulo.

### Decisión 13

El código legacy podrá permanecer temporalmente detrás de adapters mientras disminuye su uso.

### Decisión 14

El runtime final deberá centralizar conexiones y lifecycle mediante VoltStack Database.

### Decisión 15

La finalización no requiere eliminar SQL nativo; requiere eliminar acceso no controlado a la base.

---

## 179. Resultado esperado

Antes:

```text
Application
│
├── PDO
├── global DB
├── custom wrappers
├── DAO
├── raw SQL
└── stored procedures
      │
      ▼
   Database
```

Durante:

```text
Application
│
├── Legacy Compatibility Layer
│       │
│       ▼
│   VoltStack Database
│
├── Migrated Repositories
│       │
│       ▼
│   VoltStack Database
│
└── Remaining Legacy
```

Después:

```text
Application
      │
      ▼
VoltStack Database
      │
      ├── ORM / Entities
      ├── Repositories
      ├── Query Builder
      ├── Native SQL
      ├── Transactions
      └── Procedure Calls
              │
              ▼
           Database
```

---

## 180. Principio final

```text
Discover
   ↓
Classify
   ↓
Stabilize
   ↓
Create Boundaries
   ↓
Migrate Infrastructure
   ↓
Migrate Queries
   ↓
Verify
   ↓
Remove Legacy Access
```

La migración legacy no debe medirse por cuánto código fue reescrito.

Debe medirse por cuánto acceso a datos quedó:

```text
controlled
observable
testable
secure
transactionally correct
compatible with persistent runtimes
```

---

## 181. Conclusión

`DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM` permite que VoltStack Database V1 pueda adoptarse no solo en proyectos modernos, sino también en aplicaciones con años de evolución, SQL directo y arquitecturas de persistencia heterogéneas.

Su diseño evita dos extremos:

```text
keep legacy forever
```

y:

```text
rewrite everything at once
```

La estrategia de VoltStack será:

```text
understand existing behavior
introduce controlled boundaries
migrate incrementally
verify continuously
remove legacy infrastructure safely
```

La regla arquitectónica final será:

```text
Legacy SQL may survive.

Legacy uncontrolled persistence must not.
```

---

**Documento:** `328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**Estado:** Architectural Specification
