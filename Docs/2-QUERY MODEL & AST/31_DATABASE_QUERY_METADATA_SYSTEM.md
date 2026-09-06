# 31_DATABASE_QUERY_METADATA_SYSTEM.md

# VoltStack Quantum Database
## Query Metadata System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 31 — Query Metadata System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Metadata Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del **Query Metadata System** de:

```text
VoltStack/Quantum/Database
```

El sistema permitirá asociar información contextual, declarativa y operativa a una consulta sin contaminar:

- Query Model;
- Query AST;
- Expressions;
- Predicates;
- Parameters;
- Semantic Graph;
- Execution state.

La regla fundamental será:

```text
Query Structure
≠
Query Metadata
≠
Semantic Metadata
≠
Execution Metadata
≠
Runtime State
```

---

# 2. Problema arquitectónico

Una consulta necesita más información que únicamente su estructura.

Por ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

estructuralmente representa:

```text
SELECT
FROM users
WHERE active = ?
```

pero durante su procesamiento VoltStack podría necesitar conocer:

```text
origin
connection intent
consistency requirement
transaction requirement
timeout
cache policy
security classification
telemetry options
query label
retry policy hints
tenant policy provenance
ORM provenance
diagnostic information
extension metadata
```

Introducir todo esto directamente dentro del AST produciría:

```text
AST
├── SQL semantics
├── connection state
├── cache configuration
├── telemetry configuration
├── current tenant
├── timeout
├── tracing state
├── transaction
└── runtime resources
```

lo cual destruiría la separación arquitectónica.

---

# 3. Regla maestra

> La metadata describe propiedades, requisitos y procedencia de una consulta; nunca deberá convertir la consulta en un contenedor de estado mutable del runtime.

---

# 4. Separación fundamental

VoltStack distinguirá:

```text
Query Model
    │
    ├── estructura declarativa
    │
    ▼
Query AST
    │
    ├── representación estructural canónica
    │
    ▼
Semantic Metadata
    │
    ├── información derivada por análisis
    │
    ▼
Planning Metadata
    │
    ├── requisitos y decisiones
    │
    ▼
Execution Metadata
    │
    ├── instrucciones declarativas
    │
    ▼
Execution Context
         runtime state
```

---

# 5. Metadata ≠ Context

Esta distinción será obligatoria.

```text
Metadata
=
descripción declarativa

Context
=
estado de una operación concreta
```

Ejemplo:

```text
QueryMetadata:
timeout = 5 seconds
```

es válido.

Pero:

```text
QueryMetadata:
timerHandle = object(...)
```

no lo es.

---

# 6. Metadata ≠ resource

La metadata nunca contendrá:

- PDO;
- NativeConnection;
- ConnectionLease;
- Transaction;
- EntityManager;
- UnitOfWork;
- IdentityMap;
- ResultCursor;
- Stream;
- Fiber;
- Coroutine;
- Request object;
- Service Container.

---

# 7. Arquitectura general

```text
                    Query
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
     Query Structure         Query Metadata
          │                       │
          ▼                       ▼
         AST              Metadata Resolution
          │                       │
          └───────────┬───────────┘
                      ▼
              Semantic Analysis
                      │
                      ▼
               Semantic Graph
                      │
                      ▼
                  Planner
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
       Execution Plan    Effective Metadata
             │                 │
             └────────┬────────┘
                      ▼
                   Compiler
                      │
                      ▼
                CompiledQuery
                      │
                      ▼
                  Executor
                      │
                      ▼
              ExecutionContext
```

---

# 8. Objetivos

El Query Metadata System deberá proporcionar:

1. metadata estructurada;
2. metadata tipada;
3. metadata inmutable;
4. procedencia de consultas;
5. query intent;
6. connection intent;
7. consistency requirements;
8. transaction requirements;
9. execution policies;
10. cache policies;
11. security metadata;
12. telemetry metadata;
13. diagnostic metadata;
14. extension metadata;
15. metadata transformations;
16. metadata inheritance;
17. metadata merging;
18. metadata validation;
19. metadata fingerprinting;
20. metadata redaction.

---

# 9. No objetivos

El sistema no deberá:

- ejecutar queries;
- abrir conexiones;
- seleccionar replicas directamente;
- iniciar transacciones;
- almacenar objetos de transacción;
- almacenar current tenant;
- guardar spans activos;
- almacenar resultados;
- generar SQL;
- resolver EntityManager;
- implementar retries;
- funcionar como Service Locator.

---

# 10. QueryMetadata

El aggregate conceptual será:

```text
QueryMetadata
├── identity
├── origin
├── intent
├── requirements
├── policies
├── security
├── observability
├── diagnostics
├── provenance
└── extensions
```

---

# 11. QueryMetadata interface

Conceptualmente:

```php
interface QueryMetadata
{
    public function origin(): QueryOrigin;

    public function intent(): QueryIntent;

    public function requirements(): QueryRequirements;

    public function policies(): QueryPolicies;
}
```

La API final deberá evitar un contrato excesivamente grande.

---

# 12. Preferencia por composición

En vez de:

```text
QueryMetadata
  80 properties
```

se preferirá:

```text
QueryMetadata
├── QueryIdentityMetadata
├── QueryOriginMetadata
├── QueryIntentMetadata
├── QueryRequirementMetadata
├── QueryPolicyMetadata
├── QuerySecurityMetadata
├── QueryObservabilityMetadata
├── QueryDiagnosticMetadata
├── QueryProvenanceMetadata
└── QueryExtensionMetadata
```

---

# 13. Metadata categories

Categorías oficiales iniciales:

```text
IDENTITY
ORIGIN
INTENT
REQUIREMENT
POLICY
SECURITY
OBSERVABILITY
DIAGNOSTIC
PROVENANCE
EXTENSION
```

---

# 14. Query identity metadata

Podrá contener:

```text
QueryInstanceId
QueryShapeFingerprint
QueryLabel
CorrelationKey
```

siempre que no incluya estado mutable.

---

# 15. QueryInstanceId

Representará una instancia lógica de consulta.

Ejemplo:

```text
qry_01J...
```

No será:

- SQL hash;
- ExecutionId;
- TraceId;
- connection ID.

---

# 16. QueryInstanceId ≠ ExecutionId

Una consulta puede ejecutarse varias veces:

```text
QueryInstance
      │
      ├── Execution A
      ├── Execution B
      └── Execution C
```

---

# 17. QueryShapeFingerprint

Identifica la forma estructural de una consulta.

Ejemplo:

```text
SELECT users WHERE id = :parameter
```

Valores runtime normalmente no participan.

---

# 18. QueryShapeFingerprint ≠ CompiledQueryFingerprint

Porque:

```text
Query Shape
```

puede compilarse de forma diferente para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 19. QueryLabel

Permitirá proporcionar una etiqueta humana:

```text
user.profile.lookup
orders.pending.list
billing.invoice.create
```

---

# 20. QueryLabel purpose

Puede utilizarse para:

- telemetry;
- profiler;
- diagnostics;
- performance reports;
- tracing;
- debugging.

---

# 21. QueryLabel security

No deberá contener:

```text
user email
tenant secret
access token
password
credit card
arbitrary user input
```

---

# 22. Low-cardinality labels

Las etiquetas de telemetry deberán ser preferentemente de baja cardinalidad.

Correcto:

```text
orders.pending.list
```

Incorrecto:

```text
orders.user.918293.pending.list
```

---

# 23. QueryOrigin

Describe quién originó conceptualmente la consulta.

Valores iniciales:

```text
APPLICATION
ORM
REPOSITORY
ACTIVE_RECORD
SCHEMA
MIGRATION
SEEDER
FACTORY
QUEUE
FRAMEWORK
INTERNAL
EXTENSION
ADMINISTRATION
TESTING
```

---

# 24. Origin ≠ caller stack

No será necesario capturar stack trace para determinar el origin.

---

# 25. QueryOriginMetadata

Conceptualmente:

```text
QueryOriginMetadata
├── origin
├── component?
├── operation?
└── extensionId?
```

---

# 26. Example

```text
origin:
ORM

component:
UserRepository

operation:
findActiveUsers
```

---

# 27. Component metadata security

Los nombres deberán normalizarse y no exponer datos sensibles.

---

# 28. QueryIntent

Describe la intención semántica general.

Ejemplos:

```text
READ
WRITE
DDL
LOCKING_READ
MAINTENANCE
ADMINISTRATION
INTROSPECTION
UNKNOWN
```

---

# 29. QueryIntent ≠ SQL keyword

Por ejemplo:

```text
SELECT ... FOR UPDATE
```

es sintácticamente SELECT pero semánticamente:

```text
LOCKING_READ
```

---

# 30. Query intent derivation

Puede derivarse inicialmente de Query Model y refinarse durante Semantic Analysis.

---

# 31. Explicit vs inferred intent

Se distinguirá:

```text
INFERRED
EXPLICIT
DERIVED
```

---

# 32. Explicit intent cannot lie

Una consulta estructuralmente WRITE no podrá marcarse:

```text
READ
```

para forzar replica routing.

---

# 33. Intent validation

El Semantic Engine deberá detectar contradicciones.

---

# 34. ConnectionIntent

Representará qué clase lógica de conexión necesita la operación.

Ejemplos:

```text
READ
WRITE
PRIMARY
REPLICA_ALLOWED
READ_ONLY
MIGRATION
ADMINISTRATION
```

---

# 35. QueryIntent ≠ ConnectionIntent

Ejemplo:

```text
QueryIntent:
READ

ConnectionIntent:
PRIMARY
```

puede ser válido después de una escritura sticky o dentro de una transacción.

---

# 36. ConnectionPreference

Podrá expresar preferencias declarativas:

```text
DEFAULT
PRIMARY
REPLICA_PREFERRED
PRIMARY_PREFERRED
NAMED_CONNECTION
```

sin almacenar una `Connection` real.

---

# 37. Named connection metadata

Puede contener:

```text
ConnectionName
```

pero nunca:

```text
Connection object
```

---

# 38. Connection selection remains external

El Query Metadata System declara requisitos.

El:

```text
ConnectionManager
TopologyResolver
TransactionAffinityResolver
```

decide el target real.

---

# 39. ConsistencyRequirement

Representará la consistencia requerida.

Ejemplos:

```text
EVENTUAL
SESSION
READ_YOUR_WRITES
STRONG
PRIMARY
TRANSACTIONAL
```

La lista exacta podrá evolucionar.

---

# 40. Consistency is semantic

No deberá expresarse simplemente como:

```text
useReplica = false
```

---

# 41. Example

```text
Query:
show profile immediately after update

Consistency:
READ_YOUR_WRITES
```

El topology layer podrá resolverlo a primary/sticky connection.

---

# 42. ConsistencyRequirement ≠ routing decision

Metadata:

```text
READ_YOUR_WRITES
```

Planner/Topology:

```text
PRIMARY
```

---

# 43. TransactionRequirement

Representará requisitos transaccionales.

Conceptualmente:

```text
NONE
OPTIONAL
REQUIRED
REQUIRES_EXISTING
FORBIDS_TRANSACTION
```

---

# 44. TransactionRequirement ≠ Transaction

Correcto:

```text
transactionRequirement = REQUIRED
```

Incorrecto:

```text
transaction = TransactionObject
```

---

# 45. Transaction affinity

Si existe una transacción activa, el runtime resolverá la afinidad mediante `TransactionContext`.

No mediante QueryMetadata mutable.

---

# 46. Isolation requirement

Una consulta especializada podría declarar:

```text
MinimumIsolationRequirement
```

si realmente lo necesita.

---

# 47. Avoid overuse

La mayoría de las queries no deberán especificar isolation level individual.

---

# 48. CapabilityRequirement

Metadata puede transportar requisitos como:

```text
query.returning
query.window
transaction.savepoint
query.json.contains
```

cuando sean explícitos o derivados.

---

# 49. Capability requirements are semantic

No deberán expresarse:

```text
requiresPostgreSQL = true
```

---

# 50. Effective capability requirements

Podrán acumularse desde:

```text
Query Model
Expressions
Predicates
Types
Functions
Operators
Locking
Returning
Extensions
```

---

# 51. QueryRequirements

Aggregate conceptual:

```text
QueryRequirements
├── connection
├── consistency
├── transaction
├── capabilities
├── result
├── execution
└── security
```

---

# 52. Requirement ≠ Policy

Distinción:

```text
Requirement
=
must be satisfied

Policy
=
how framework should prefer or control behavior
```

---

# 53. Example

```text
Requirement:
strong consistency

Policy:
prefer replica when safe
```

La requirement domina.

---

# 54. Conflict resolution

```text
Strong Consistency
+
Replica Preferred
```

deberá resultar en una estrategia que preserve consistencia, normalmente primary.

---

# 55. QueryPolicies

Podrá contener:

```text
ExecutionPolicy
CachePolicy
RetryPolicyHint
TimeoutPolicy
TelemetryPolicy
DiagnosticPolicy
SafetyPolicy
```

---

# 56. ExecutionPolicy

Describe preferencias/restricciones de ejecución.

Ejemplos:

```text
NORMAL
STREAMING
BUFFERED
SINGLE_ROW
BULK
```

---

# 57. ResultRequirement

Puede ser más apropiado que ExecutionPolicy para algunas propiedades.

Ejemplo:

```text
ResultMode:
BUFFERED
STREAMING
CURSOR
SCALAR
SINGLE_ROW
```

---

# 58. Query metadata should express intent

No deberá dictar detalles físicos innecesarios.

---

# 59. Timeout metadata

Se distinguirán:

```text
ConnectionTimeout
PoolAcquisitionTimeout
StatementTimeout
LockTimeout
TransactionTimeout
```

---

# 60. Query-level timeout

Normalmente corresponde a:

```text
StatementTimeoutRequirement
```

---

# 61. Timeout ≠ timer

Metadata:

```text
statementTimeout = 5s
```

ExecutionContext:

```text
deadline
timer
cancellation token
```

---

# 62. Deadline

Un deadline absoluto normalmente pertenece a ExecutionContext.

---

# 63. Cancellation

Metadata podrá declarar:

```text
cancellable = true
```

si fuera necesario.

Pero:

```text
CancellationToken
```

pertenece al ExecutionContext.

---

# 64. Retry metadata

Las queries podrán proporcionar información para clasificación.

Ejemplo:

```text
RetrySafety:
SAFE
CONDITIONAL
UNSAFE
UNKNOWN
```

---

# 65. Retry policy ownership

El Query Metadata System no implementará retries.

Eso pertenece a:

```text
Execution Resilience
```

---

# 66. Idempotency metadata

Podrá existir:

```text
IdempotencyClassification
```

con valores:

```text
IDEMPOTENT
NON_IDEMPOTENT
CONDITIONALLY_IDEMPOTENT
UNKNOWN
```

---

# 67. Idempotency ≠ idempotency key

Un idempotency key de aplicación pertenece normalmente a una capa superior.

---

# 68. Retry safety derivation

Puede depender de:

- query intent;
- transaction state;
- generated values;
- side effects;
- locking;
- functions;
- platform semantics.

Por ello la metadata inicial puede refinarse posteriormente.

---

# 69. CachePolicy

Podrá declarar:

```text
DEFAULT
BYPASS
ALLOW
PREFER
REQUIRE
```

para determinados mecanismos de cache.

---

# 70. Query cache ≠ result cache

Deberán diferenciarse:

```text
CompiledQueryCachePolicy
ResultCachePolicy
EntityCachePolicy
MetadataCachePolicy
```

---

# 71. Query metadata and cache

Este documento se centrará principalmente en metadata declarativa.

La arquitectura completa de cache se definirá en documentos posteriores.

---

# 72. Result cache metadata

Podrá incluir:

```text
enabled
ttl
tags
namespace
consistencyRequirement
```

pero deberá evitar guardar servicios de cache.

---

# 73. Cache key

No deberá aceptar arbitrariamente datos sensibles sin normalización.

---

# 74. Cache key generation

Será responsabilidad del Cache System.

Metadata proporciona inputs declarativos.

---

# 75. Cache tags

Podrán ser:

```text
users
orders
tenant:{semantic-key}
```

pero deberán pasar por políticas de seguridad/cardinalidad.

---

# 76. Cache metadata and writes

Write queries no deberán convertirse en cacheable result queries por metadata incorrecta.

---

# 77. Security metadata

La consulta podrá transportar metadata de seguridad declarativa.

Ejemplos:

```text
sensitive
containsSensitiveBindings
sensitiveResult
auditRequired
redactionPolicy
trustedRawSql
privilegedOperation
```

---

# 78. SecurityMetadata

Conceptualmente:

```text
QuerySecurityMetadata
├── sensitivity
├── bindingSensitivity
├── resultSensitivity
├── auditRequirement
├── redactionPolicy
├── rawSqlTrust
└── privilegeClassification
```

---

# 79. Sensitive data classification

Podrá utilizar categorías como:

```text
PUBLIC
INTERNAL
CONFIDENTIAL
SENSITIVE
SECRET
```

sin intentar reemplazar un sistema completo de Data Governance.

---

# 80. Binding sensitivity

Idealmente podrá identificarse por `ParameterId`.

Ejemplo:

```text
p1 → NORMAL
p2 → SECRET
p3 → PERSONAL_DATA
```

---

# 81. No sensitive value storage

Metadata almacena:

```text
p2 is SECRET
```

no:

```text
p2 = "my-password"
```

---

# 82. Result sensitivity

Puede ayudar a:

- telemetry redaction;
- debug toolbar;
- query logging;
- profiler;
- cache policy.

---

# 83. Security metadata cannot weaken defaults

Una query no deberá poder marcar:

```text
redaction = OFF
```

si una política superior exige redacción.

---

# 84. Policy layering

```text
Framework Security Policy
        │
        ▼
Application Policy
        │
        ▼
Query Security Metadata
        │
        ▼
Effective Security Policy
```

Las capas inferiores pueden restringir más, no debilitar garantías superiores.

---

# 85. Raw SQL trust metadata

Raw SQL podrá clasificarse:

```text
TRUSTED_STATIC
TRUSTED_EXTENSION
DYNAMIC_REVIEWED
UNTRUSTED
UNKNOWN
```

---

# 86. Raw SQL metadata does not sanitize SQL

Clasificar una query como trusted no sustituye validación/binding.

---

# 87. AuditRequirement

Valores conceptuales:

```text
DEFAULT
REQUIRED
ENHANCED
NONE_ALLOWED_BY_POLICY
```

---

# 88. Audit ≠ telemetry

Audit tiene requisitos distintos de observability.

---

# 89. Telemetry metadata

Podrá incluir:

```text
query label
operation category
telemetry enabled
sampling hint
slow query threshold override
trace annotation policy
metrics classification
```

---

# 90. TelemetryMetadata

Conceptualmente:

```text
QueryTelemetryMetadata
├── label
├── category
├── samplingHint
├── slowQueryPolicy
├── metricDimensions
└── tracingAttributes
```

---

# 91. No active Span

Incorrecto:

```text
QueryMetadata
└── OpenTelemetry Span
```

Correcto:

```text
QueryMetadata
└── tracing attributes

ExecutionContext
└── active span
```

---

# 92. Telemetry integration optional

Database deberá funcionar aunque:

```text
Quantum/Telemetry
```

no esté instalado.

---

# 93. Telemetry ports

La integración se realizará mediante:

```text
ports
adapters
null implementations
```

---

# 94. Query telemetry labels

No deberán utilizar raw SQL como metric label.

---

# 95. High cardinality protection

El sistema podrá rechazar o normalizar metadata peligrosa.

---

# 96. DiagnosticMetadata

Información destinada a:

- profiler;
- debug toolbar;
- explain;
- testing;
- development diagnostics.

---

# 97. Diagnostic metadata examples

```text
source file?
source line?
builder operation?
repository method?
query name?
developer note?
```

---

# 98. Source location

Puede representarse mediante:

```text
SourceLocation
├── file
├── line
└── column?
```

---

# 99. Source location optional

Capturar stack/file/line puede tener costo.

Deberá ser configurable y normalmente activarse en desarrollo.

---

# 100. Source location security

Los paths absolutos pueden revelar infraestructura.

En producción deberán:

- omitirse;
- relativizarse;
- redaccionarse.

---

# 101. Developer note

Podrá existir metadata como:

```text
"load dashboard totals"
```

pero no deberá utilizarse para lógica del framework.

---

# 102. Diagnostic metadata cannot affect semantics

Un:

```text
developerNote
```

no deberá cambiar:

- routing;
- transaction;
- compiler;
- optimizer.

---

# 103. Provenance

Query provenance describe cómo se originó o transformó una consulta.

---

# 104. QueryProvenance

Conceptualmente:

```text
QueryProvenance
├── origin
├── parent?
├── transformations
├── policyApplications
├── extensionApplications
└── generatedBy?
```

---

# 105. Example

```text
Application Query
      │
      ▼
ORM Scope
      │
      ▼
Tenant Policy
      │
      ▼
Soft Delete Policy
      │
      ▼
Normalized Query
```

---

# 106. Provenance vs AST history

No será necesario guardar todas las versiones completas del AST.

---

# 107. Transformation record

Podrá registrar:

```text
TransformationRecord
├── transformationId
├── category
├── source
└── sequence
```

sin copiar la consulta completa.

---

# 108. Provenance categories

```text
BUILDER
ORM
POLICY
SECURITY
TENANT
NORMALIZATION
EXTENSION
OPTIMIZATION
PLANNING
```

---

# 109. Optimizer provenance

Podría almacenarse separadamente como OptimizationTrace en debug mode.

No necesariamente dentro de QueryMetadata persistente.

---

# 110. Query metadata evolution

Se distinguirán:

```text
Declared Metadata
Derived Metadata
Effective Metadata
Runtime Context
```

---

# 111. DeclaredMetadata

Proviene de:

- application;
- Builder;
- ORM;
- Repository;
- extension.

---

# 112. DerivedMetadata

Se calcula desde la estructura.

Ejemplo:

```text
SELECT FOR UPDATE
→ locking read
→ primary required
→ transaction requirement
```

---

# 113. EffectiveMetadata

Resultado de combinar:

```text
Declared
+
Derived
+
Framework Policy
+
Security Policy
+
Capability Resolution
```

---

# 114. Runtime Context

Finalmente:

```text
EffectiveMetadata
+
Runtime Environment
→ ExecutionContext
```

---

# 115. Example

Declared:

```text
consistency = DEFAULT
```

Derived:

```text
locking query
```

Framework policy:

```text
locking queries require primary
```

Effective:

```text
connection intent = PRIMARY
transaction requirement = REQUIRED
```

Runtime:

```text
actual connection = primary-1
transaction = tx-42
```

Los dos últimos valores no pertenecen a QueryMetadata.

---

# 116. Metadata precedence

Se necesitará un modelo explícito.

Posible orden:

```text
Security Invariants
        │
        ▼
Framework Mandatory Policy
        │
        ▼
Semantic Requirements
        │
        ▼
Explicit Query Requirements
        │
        ▼
Application Defaults
        │
        ▼
Framework Defaults
```

No será simplemente "last write wins".

---

# 117. MetadataMergePolicy

Conceptualmente:

```text
MetadataMergePolicy
```

decidirá cómo combinar cada categoría.

---

# 118. Merge strategies

Podrán incluir:

```text
OVERRIDE
RESTRICT
UNION
INTERSECTION
MINIMUM
MAXIMUM
MOST_STRICT
FIRST_EXPLICIT
ERROR_ON_CONFLICT
CUSTOM
```

---

# 119. Example timeout merge

Framework maximum:

```text
30s
```

Query requests:

```text
120s
```

Effective:

```text
30s
```

si la política define máximo obligatorio.

---

# 120. Example security merge

Framework:

```text
redaction = REQUIRED
```

Query:

```text
redaction = NONE
```

Resultado:

```text
REQUIRED
```

---

# 121. Example capability requirements

Query feature A:

```text
requires X
```

Expression B:

```text
requires Y
```

Effective:

```text
requires X AND Y
```

---

# 122. Example transaction requirement conflict

Component A:

```text
REQUIRED
```

Component B:

```text
FORBIDS_TRANSACTION
```

Resultado:

```text
MetadataConflictException
```

---

# 123. Metadata source

Cada valor importante podrá registrar su fuente.

Ejemplo:

```text
connectionIntent:
PRIMARY

source:
LockingSemanticRule
```

---

# 124. MetadataValue

Conceptualmente:

```text
MetadataValue<T>
├── value
├── source
├── strength
└── certainty
```

No será obligatorio envolver absolutamente todos los valores si el costo/complexidad no lo justifica.

---

# 125. Metadata strength

Puede distinguir:

```text
DEFAULT
PREFERENCE
REQUIREMENT
MANDATORY
```

---

# 126. Preference vs requirement

Ejemplo:

```text
replica preferred
```

puede perder frente a:

```text
primary required
```

---

# 127. Metadata certainty

Para metadata derivada:

```text
EXPLICIT
DERIVED
INFERRED
UNKNOWN
```

---

# 128. Metadata validation

Se realizará en diferentes fases.

---

# 129. Builder validation

Detecta errores locales obvios.

---

# 130. Structural metadata validation

Ejemplo:

```text
negative timeout
invalid label
invalid cache TTL
```

---

# 131. Semantic metadata validation

Ejemplo:

```text
READ intent
+
UPDATE query
```

---

# 132. Policy validation

Ejemplo:

```text
query asks to disable required audit
```

---

# 133. Capability validation

Ejemplo:

```text
streaming required
+
driver does not support required mode
```

---

# 134. Runtime validation

Ejemplo:

```text
REQUIRES_EXISTING_TRANSACTION
+
no active transaction
```

Este error sólo puede comprobarse en ejecución.

---

# 135. Validation stages

```text
Metadata Construction
        │
        ▼
Structural Validation
        │
        ▼
Semantic Validation
        │
        ▼
Policy Resolution
        │
        ▼
Capability Validation
        │
        ▼
Runtime Validation
```

---

# 136. Metadata immutability

`QueryMetadata` deberá ser inmutable una vez asociado al Query Model/AST.

---

# 137. Transformations

Una transformación deberá producir:

```text
Metadata A
→
Metadata B
```

en vez de mutar A.

---

# 138. Example

```text
Query A
Metadata A

TenantPolicy
      │
      ▼

Query B
Metadata B
```

---

# 139. Metadata builders

Puede existir un builder mutable exclusivamente durante construcción:

```text
QueryMetadataBuilder
```

que finalmente produzca metadata inmutable.

---

# 140. Builder lifetime

Operation-local.

Nunca singleton mutable.

---

# 141. Metadata inheritance

Subqueries requieren reglas claras.

---

# 142. Parent query ≠ child query

No toda metadata deberá heredarse.

---

# 143. Metadata inheritance classes

Podrán clasificarse:

```text
INHERIT
DO_NOT_INHERIT
MERGE
DERIVE
```

---

# 144. Example — security

Sensitivity puede heredarse o hacerse más estricta.

---

# 145. Example — timeout

Una subquery SQL no tiene ejecución independiente normalmente.

Por ello no debe recibir un timer separado automáticamente.

---

# 146. Example — query label

Una subquery puede tener label propio para diagnostics, pero no necesariamente telemetry independiente.

---

# 147. Example — connection

Una subquery compilada dentro de una query no puede seleccionar otra conexión.

---

# 148. Critical invariant

Toda consulta SQL compilada como una única statement utiliza una única ejecución/connection context.

---

# 149. CTE metadata

Las CTEs podrán tener metadata estructural/diagnóstica local.

Pero requisitos efectivos deberán integrarse al statement principal.

---

# 150. Set operation metadata

```text
UNION
INTERSECT
EXCEPT
```

deberán combinar requirements de todas las branches.

---

# 151. Capability requirement merge

```text
Branch A requires X
Branch B requires Y

Set Query requires:
X AND Y
```

---

# 152. Security merge

El resultado deberá adoptar como mínimo la sensibilidad más estricta de las branches relevantes.

---

# 153. Query composition

Metadata deberá poder sobrevivir:

```text
subquery composition
CTE composition
set operations
ORM transformations
policy transformations
normalization
optimization
```

sin perder invariantes.

---

# 154. Query normalization

La normalización estructural no deberá destruir metadata significativa.

---

# 155. Metadata normalization

Podrá existir:

```text
QueryMetadataNormalizer
```

para canonicalizar:

- labels;
- tags;
- defaults;
- aliases;
- duplicate requirements.

---

# 156. Query optimization

El Optimizer no deberá modificar arbitrariamente metadata declarada.

---

# 157. Optimization metadata

Información específica del Optimizer deberá vivir preferentemente en:

```text
OptimizationContext
OptimizationTrace
OptimizedQueryMetadata
```

según su naturaleza.

---

# 158. Semantic metadata

No toda información derivada debe vivir dentro de `QueryMetadata`.

Ejemplos:

```text
resolved symbols
resolved types
resolved relationships
semantic graph edges
```

pertenecen al Semantic Analysis artifact.

---

# 159. Important separation

```text
QueryMetadata
≠
SemanticGraph
```

---

# 160. Semantic artifacts

Podrán contener:

```text
SymbolTable
QueryTypeTable
RelationTable
ConstraintTable
CapabilityRequirementSet
SemanticAnnotations
```

---

# 161. Query metadata should stay small

No duplicar todo el Semantic Graph dentro de metadata.

---

# 162. Planning metadata

El Planner producirá artifacts propios.

Ejemplos:

```text
LogicalPlanMetadata
PhysicalPlanMetadata
ExecutionPlanMetadata
```

---

# 163. QueryMetadata vs ExecutionPlanMetadata

Query metadata expresa requisitos.

Plan metadata expresa decisiones.

---

# 164. Example

Query metadata:

```text
resultMode = STREAMING
```

Plan metadata:

```text
use server-side cursor strategy X
hold lease until cursor close
```

---

# 165. Compiler metadata

Compiler podrá producir:

```text
CompiledQueryMetadata
```

---

# 166. CompiledQueryMetadata

Puede incluir:

```text
statement classification
placeholder count
binding layout
result column metadata
dialect fingerprint
capability fingerprint
compiler version
```

---

# 167. QueryMetadata ≠ CompiledQueryMetadata

No deberán mezclarse.

---

# 168. Execution metadata

`CompiledQuery` puede contener instrucciones declarativas necesarias por Executor.

Ejemplo:

```text
connection requirement
transaction requirement
result mode
statement timeout
retry classification
security/redaction classification
```

---

# 169. Runtime execution context

El Executor crea:

```text
ExecutionContext
```

con recursos reales.

---

# 170. ExecutionContext example

```text
ExecutionContext
├── ExecutionId
├── DatabaseContext
├── ConnectionResolution
├── ConnectionLease
├── TransactionContext
├── CancellationToken
├── Deadline
├── TelemetryExecution
└── CleanupTracker
```

---

# 171. None of these belong to QueryMetadata

Especialmente:

```text
ConnectionLease
TransactionContext
CancellationToken
active telemetry span
```

---

# 172. QueryProvenance and policies

Una transformación de seguridad podrá registrar:

```text
policy:
tenant.scope

effect:
predicate added
```

---

# 173. Provenance does not replace audit

Provenance ayuda a explicar la query.

Audit registra eventos operativos.

---

# 174. Query provenance example

```text
Original Query
    │
    ├── SoftDeletePolicy
    │       + deleted_at IS NULL
    │
    ├── TenantPolicy
    │       + tenant_id = :tenant
    │
    └── AuthorizationDataScope
            + organization_id IN (...)
```

---

# 175. Security benefit

Esto permite responder:

```text
Why does this predicate exist?
```

sin analizar SQL textual.

---

# 176. PolicyTransformationId

Cada transformación podrá tener ID estable:

```text
orm.soft_delete
multitenancy.tenant_scope
authorization.resource_scope
```

---

# 177. Optional integrations

Database core no deberá depender obligatoriamente de:

```text
Quantum/Multitenancy
Quantum/Authorization
Quantum/Telemetry
Quantum/Cache
```

---

# 178. Integration metadata

Estos paquetes podrán agregar metadata mediante contratos de extensión.

---

# 179. Extension metadata

Se proporcionará:

```text
QueryExtensionMetadata
```

---

# 180. Extension metadata namespace

Cada extensión deberá utilizar un namespace estable.

Ejemplo:

```text
voltstack.multitenancy.*
voltstack.authorization.*
vendor.package.*
```

---

# 181. No arbitrary array bag

Evitar:

```php
$metadata['whatever'] = $anything;
```

como API principal.

---

# 182. Typed extension metadata

Preferir:

```text
ExtensionMetadataKey<T>
```

o descriptors tipados.

---

# 183. ExtensionMetadataKey

Conceptualmente:

```text
ExtensionMetadataKey
├── namespace
├── name
├── valueType
├── mergePolicy
├── inheritancePolicy
├── sensitivity
└── fingerprintPolicy
```

---

# 184. Why metadata descriptors

Permiten saber:

- cómo validar;
- cómo mergear;
- si heredar;
- si redaccionar;
- si participa en fingerprint;
- si puede llegar al compiler;
- si puede llegar al executor.

---

# 185. Extension registration

Durante bootstrap:

```text
Extension
   │
   ▼
Metadata Descriptor Registry
   │
   ▼
Validation
   │
   ▼
Freeze
```

---

# 186. No runtime descriptor registration

Compatible con la política general de extensiones de Database.

---

# 187. QueryMetadataRegistry

Si existe, será especializado únicamente para:

```text
metadata descriptors
```

No almacenará metadata de queries concretas.

---

# 188. Query metadata values remain local

La metadata de una query pertenece al artifact correspondiente.

---

# 189. Fingerprint policy

No toda metadata deberá participar en `QueryShapeFingerprint`.

---

# 190. Structural fingerprint

Debe depender principalmente de estructura semántica.

---

# 191. Compilation-affecting metadata

Si metadata cambia SQL compilado, deberá participar en el fingerprint apropiado.

Ejemplos:

```text
locking strategy
platform-specific hint
compiler extension option
```

---

# 192. Execution-only metadata

No debería invalidar compiled SQL si no cambia compilación.

Ejemplos:

```text
telemetry label
statement timeout
diagnostic note
```

---

# 193. Result-affecting metadata

Puede requerir un fingerprint distinto.

---

# 194. MetadataFingerprintPolicy

Cada descriptor podrá declarar:

```text
NONE
QUERY_SHAPE
SEMANTIC
PLANNING
COMPILATION
EXECUTION
CACHE
```

---

# 195. Multiple fingerprints

VoltStack deberá evitar un único hash universal.

Podrán existir:

```text
QueryShapeFingerprint
SemanticFingerprint
PlanFingerprint
CompiledQueryFingerprint
ResultCacheFingerprint
```

---

# 196. Example

Dos queries:

```text
label = dashboard.users
```

y:

```text
label = admin.users
```

con estructura idéntica pueden compartir compiled query.

---

# 197. Telemetry metadata not compilation-affecting

Por defecto:

```text
QueryLabel
```

no deberá cambiar `CompiledQueryFingerprint`.

---

# 198. Security metadata and cache

Aunque no cambie SQL, puede afectar si un resultado puede cachearse.

Por ello podría participar en:

```text
ResultCacheFingerprint
```

o directamente deshabilitar cache.

---

# 199. Cache fingerprint ownership

Lo decidirá el Cache System utilizando metadata efectiva.

---

# 200. Metadata serialization

Si Query artifacts se almacenan en cache, metadata serializable deberá ser:

- immutable;
- versioned;
- scalar/value-object based;
- resource-free;
- closure-free;
- secret-free.

---

# 201. No arbitrary object serialization

Prohibido depender de:

```php
serialize($metadata);
```

como protocolo arquitectónico.

---

# 202. Metadata schema version

Podrá existir:

```text
QueryMetadataSchemaVersion
```

para caches persistentes.

---

# 203. Metadata compatibility

Cambios de metadata deberán seguir reglas de backward compatibility.

---

# 204. Debug representation

Podrá mostrarse:

```text
Query Metadata
--------------
Origin: ORM
Intent: READ
Connection: REPLICA_ALLOWED
Consistency: EVENTUAL
Transaction: OPTIONAL
Result: BUFFERED
Cache: DEFAULT
Sensitive: false
```

---

# 205. Redacted representation

Cuando existan datos sensibles:

```text
Bindings:
p1: [REDACTED]
```

aunque los valores reales ni siquiera deberían vivir dentro de QueryMetadata.

---

# 206. Metadata inspect API

Podría existir:

```php
$query->metadata();
```

para artifacts públicos donde sea apropiado.

---

# 207. Metadata mutation API

Evitar:

```php
$query->metadata()->set(...)
```

---

# 208. Functional modification

Preferir:

```php
$query = $query->withMetadata($newMetadata);
```

o factories/builders apropiados.

---

# 209. Public Builder DX

El Builder puede ofrecer métodos como:

```php
$query
    ->label('users.active')
    ->usePrimary()
    ->timeout(seconds: 5);
```

---

# 210. Internal representation

Estos métodos producirán metadata tipada.

No flags dispersos.

---

# 211. Example — label

```php
DB::table('users')
    ->label('users.active')
    ->where('active', true)
    ->get();
```

Metadata:

```text
QueryLabel:
users.active
```

---

# 212. Example — primary requirement

```php
DB::table('users')
    ->usePrimary()
    ->where('id', $id)
    ->first();
```

Metadata:

```text
ConnectionPreference:
PRIMARY
```

No:

```text
connection:
PDO(...)
```

---

# 213. Example — consistency

```php
DB::table('users')
    ->consistency(Consistency::READ_YOUR_WRITES)
    ->find($id);
```

---

# 214. Example — timeout

```php
DB::table('reports')
    ->timeout(seconds: 10)
    ->get();
```

Metadata:

```text
StatementTimeoutRequirement:
10 seconds
```

---

# 215. Example — streaming

```php
DB::table('events')
    ->stream();
```

Metadata/requirement:

```text
ResultMode:
STREAMING
```

Planner decides strategy.

---

# 216. Example — sensitive query

```php
DB::table('credentials')
    ->sensitive()
    ->where('user_id', $id)
    ->first();
```

Effective metadata:

```text
QuerySensitivity:
SENSITIVE

Telemetry:
bindings redacted

Debug:
result preview disabled/restricted
```

---

# 217. Example — ORM origin

```php
User::where('active', true)->get();
```

Could produce:

```text
Origin:
ACTIVE_RECORD

GeneratedBy:
User model

Semantic engine:
same Query Engine as repository API
```

---

# 218. Example — Repository

```php
$userRepository->findActive();
```

Metadata:

```text
Origin:
REPOSITORY

Component:
UserRepository

Operation:
findActive
```

---

# 219. Example — Migration

```text
Origin:
MIGRATION

Intent:
DDL

ConnectionIntent:
MIGRATION

TransactionRequirement:
platform/migration-plan dependent
```

---

# 220. Migration caution

DDL uses primarily Schema/Migration systems.

Query Metadata should not become the primary schema operation metadata system.

---

# 221. Raw SQL metadata

Raw SQL requires special metadata.

---

# 222. RawSqlMetadata

Conceptually:

```text
RawSqlMetadata
├── trustClassification
├── declaredIntent
├── connectionRequirement
├── resultExpectation
├── parameterMetadata
└── portability
```

---

# 223. Raw intent validation

VoltStack puede intentar clasificar SQL conservadoramente, pero no deberá fingir comprender SQL arbitrario completamente.

---

# 224. Unknown raw query

Si no puede clasificarse:

```text
Intent:
UNKNOWN
```

---

# 225. Conservative routing

`UNKNOWN` no deberá enviarse automáticamente a replica.

---

# 226. Raw SQL and cache

Result caching deberá ser conservador.

---

# 227. Raw SQL and retries

Retry classification:

```text
UNKNOWN
```

por defecto.

---

# 228. Raw SQL and security

Bindings siguen siendo obligatorios para valores dinámicos.

---

# 229. Query hints

Los hints merecen separación.

---

# 230. Semantic hint

Ejemplo:

```text
prefer low latency
```

puede ser metadata portable.

---

# 231. Vendor SQL hint

Ejemplo conceptual:

```text
MySQL optimizer hint
```

será metadata/extension explícitamente:

```text
PLATFORM_SPECIFIC
```

---

# 232. Hint ≠ requirement

Un hint puede ignorarse.

Un requirement no.

---

# 233. Hint behavior

Cada hint deberá declarar:

```text
OPTIONAL
PREFERRED
REQUIRED
```

si la arquitectura permite esos niveles.

---

# 234. No hidden compiler hints

El Compiler no deberá descubrir metadata mediante nombres mágicos.

---

# 235. Typed hint descriptors

Extensiones podrán registrar hints explícitamente.

---

# 236. Metadata and Query Policies

VoltStack podrá tener:

```text
QueryPolicy
```

que transforme:

```text
Query + Metadata
→ Query + Metadata
```

---

# 237. Policy examples

```text
SoftDeletePolicy
TenantScopePolicy
DataAccessScopePolicy
SecurityPolicy
ReadConsistencyPolicy
```

---

# 238. Policy ordering

Deberá ser:

- determinista;
- explícito;
- validado;
- libre de ciclos.

---

# 239. Query Policy ≠ Optimizer Rule

Policy puede cambiar el conjunto de datos autorizado.

Optimizer sólo puede realizar transformaciones semánticamente equivalentes.

---

# 240. Critical distinction

```text
Policy Transformation
may intentionally alter query semantics

Optimizer Transformation
must preserve query semantics
```

---

# 241. Provenance records policy changes

Esto es especialmente importante para:

- multitenancy;
- authorization;
- soft deletes;
- data isolation.

---

# 242. Query Policy security

Security policies no deberán poder ser eliminadas por metadata de menor prioridad.

---

# 243. Tenant metadata

El core puede conocer metadata neutral como:

```text
IsolationRequirement
ScopeRequirement
```

pero no deberá depender de un `Tenant` domain object.

---

# 244. Multitenancy adapter

El paquete opcional podrá:

```text
TenantContext
      │
      ▼
Query Policy Adapter
      │
      ▼
Structured Predicate / Connection Requirement
      │
      ▼
Query + Metadata
```

---

# 245. No Tenant object in QueryMetadata

Especialmente si contiene state mutable.

---

# 246. Tenant identity

Si se necesita para planificación, deberá representarse mediante un value object neutral/scoped y con política explícita.

Aun así, debe evitar participar en caches globales incorrectamente.

---

# 247. Authorization integration

Authorization puede generar:

```text
resource scope predicates
```

pero Database no depende de Authorization.

---

# 248. Authentication integration

Query metadata no deberá contener:

```text
AuthenticatedUser
Session
Token
```

---

# 249. Jobs integration

Un Job puede originar queries.

Origin:

```text
QUEUE
```

pero el Query no contiene el Job object.

---

# 250. HTTP integration

Una request HTTP puede originar queries.

QueryMetadata no contiene:

```text
Request
Response
Route object
Controller object
```

---

# 251. Request correlation

Si se necesita correlación:

```text
CorrelationId
```

puede llegar al ExecutionContext/TelemetryContext.

No necesariamente debe formar parte de QueryMetadata estructural.

---

# 252. Trace ID

Generalmente pertenece a TelemetryContext.

No al Query artifact.

---

# 253. Execution correlation

`ExecutionId` será generado al ejecutar.

---

# 254. Query instance correlation

`QueryInstanceId` puede existir antes.

---

# 255. Metadata lifetimes

| Metadata | Lifetime |
|---|---|
| Descriptor metadata | Application |
| Query Model metadata | Query artifact |
| AST metadata | AST artifact |
| Semantic metadata | Analysis operation |
| Plan metadata | Planning artifact |
| Compiled metadata | Compiled artifact/cache |
| Execution metadata | Execution artifact |
| Runtime context | Execution scope |

---

# 256. Persistent runtime safety

En FrankenPHP:

```text
Worker
│
├── Query A Metadata
│
├── Execute A
│
├── Dispose Execution A
│
├── Query B Metadata
│
└── Execute B
```

No deberá quedar:

```text
current query metadata = A
```

en un singleton mutable.

---

# 257. OpenSwoole

```text
Shared:
Frozen MetadataDescriptorRegistry

Coroutine A:
QueryMetadata A
ExecutionContext A

Coroutine B:
QueryMetadata B
ExecutionContext B
```

---

# 258. No static current metadata

Prohibido:

```php
QueryMetadataContext::$current = ...
```

---

# 259. Metadata descriptors may be singleton

Siempre que sean:

- immutable;
- stateless;
- reentrant.

---

# 260. Metadata resolvers

Podrán ser singleton si no retienen query state.

---

# 261. Query metadata objects

Serán value objects/artifacts.

---

# 262. Metadata resolution context

Podrá existir:

```text
QueryMetadataResolutionContext
```

pero deberá ser operation-scoped.

---

# 263. Resolution context contents

Puede incluir:

```text
framework policy snapshot
capability snapshot
query semantic summary
execution mode
extension metadata rules
```

---

# 264. No resources

No incluir:

```text
PDO
ConnectionLease
ResultCursor
```

---

# 265. MetadataResolver

Conceptualmente:

```text
Declared Metadata
+
Derived Metadata
+
Policy Snapshot
+
Semantic Requirements
→ Effective Query Metadata
```

---

# 266. QueryMetadataResolver ≠ Planner

Resolver metadata no decide plan físico completo.

---

# 267. QueryMetadataResolver ≠ ConnectionResolver

No selecciona servidor real.

---

# 268. QueryMetadataResolver ≠ Executor

No ejecuta.

---

# 269. QueryMetadataResolver ≠ Security Manager

Aplica reglas declarativas ya disponibles mediante contratos apropiados.

---

# 270. EffectiveQueryMetadata

Podrá ser artifact separado.

```text
EffectiveQueryMetadata
├── effectiveIntent
├── effectiveRequirements
├── effectivePolicies
├── effectiveSecurity
├── effectiveObservability
└── provenance
```

---

# 271. Declared vs effective metadata

Mantener ambos puede ser útil para explain/debug.

---

# 272. Example explain

```text
Declared:
Replica Preferred

Derived:
Locking Read

Framework Rule:
Locking Read Requires Primary

Effective:
Primary Required
```

---

# 273. Metadata resolution trace

En development:

```text
MetadataResolutionTrace
```

puede registrar decisiones.

---

# 274. Production overhead

Trace detallado deberá poder desactivarse.

---

# 275. Metadata diagnostics API

Ejemplo conceptual:

```text
database:query:explain
```

podría mostrar:

```text
Query
Semantic Types
Requirements
Metadata
Capabilities
Plan
Compilation Target
```

---

# 276. Metadata redaction renderer

Un renderer especializado deberá aplicar políticas de redacción.

---

# 277. `__toString()` caution

No utilizar `__toString()` para volcar metadata sensible completa.

---

# 278. Logging

Logging estructurado deberá utilizar una representación segura.

---

# 279. Metadata size limits

Extension metadata y diagnostic metadata deberán tener límites razonables.

---

# 280. Why size limits

Evita:

- memory amplification;
- huge traces;
- telemetry abuse;
- accidental payload storage;
- cache inflation.

---

# 281. Metadata should not store result data

Prohibido:

```text
metadata.resultPreview = 10000 rows
```

---

# 282. Metadata should not store parameter values

Los valores viven en:

```text
ParameterBindingSet
```

no en metadata.

---

# 283. Parameter metadata

Sí puede almacenar:

```text
ParameterId
Type
Sensitivity
Origin
```

sin valor.

---

# 284. Integration with document 29

`29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md` controla:

```text
Parameter Definition
Parameter Binding
Runtime Value
Binding Set
```

El Metadata System puede referenciar `ParameterId`.

---

# 285. Integration with document 30

`30_DATABASE_QUERY_TYPE_SYSTEM.md` controla:

```text
ResolvedQueryType
QueryTypeTable
Type Constraints
```

Metadata no deberá duplicar esa tabla.

---

# 286. Integration with AST

AST podrá tener:

```text
AstNodeId
```

y metadata externa podrá referenciarlo.

---

# 287. Node metadata

Algunas extensiones pueden necesitar metadata por nodo.

---

# 288. AstNodeMetadataTable

Si se necesita:

```text
AstNodeId
→ Metadata
```

deberá existir como artifact separado.

---

# 289. Do not bloat nodes

Evitar:

```text
AstNode
├── metadata bag
├── annotations bag
├── extensions bag
└── runtime bag
```

---

# 290. Core AST purity

Los nodes deben permanecer enfocados en estructura semántica.

---

# 291. Node annotations

Información derivada podrá almacenarse en:

```text
SemanticAnnotationTable
```

---

# 292. Source locations

Pueden estar en:

```text
AstSourceMap
```

en vez de cada node, especialmente si se busca reducir memoria.

---

# 293. Metadata storage strategy

VoltStack podrá elegir entre:

```text
embedded immutable metadata
```

para metadata esencial de Query Model y:

```text
side tables
```

para metadata pesada/derivada/diagnóstica.

---

# 294. Recommended rule

```text
Essential declarative metadata
→ Query artifact

Derived/heavy metadata
→ side artifact
```

---

# 295. Metadata tiering

Propuesta:

```text
Tier 1 — Core Metadata
Always available

Tier 2 — Semantic Metadata
Analysis generated

Tier 3 — Operational Metadata
Planning/execution requirements

Tier 4 — Diagnostic Metadata
Development/telemetry optional
```

---

# 296. Core metadata

Ejemplos:

```text
origin
label
explicit connection preference
explicit consistency
explicit timeout
```

---

# 297. Semantic metadata

Ejemplos:

```text
effective query intent
capability requirements
resolved read/write classification
```

---

# 298. Operational metadata

Ejemplos:

```text
result mode
retry safety
transaction requirement
```

---

# 299. Diagnostic metadata

Ejemplos:

```text
source location
transformation trace
resolution trace
developer note
```

---

# 300. Metadata budget

El Query Engine deberá mantener el camino común ligero.

Una query simple no deberá crear docenas de objetos metadata vacíos.

---

# 301. Empty metadata singleton/value

Podrá existir:

```text
QueryMetadata::empty()
```

inmutable y reutilizable.

---

# 302. Lazy metadata sections

Se podrán crear únicamente cuando sean necesarias.

---

# 303. Copy-on-write semantics

Para modificaciones funcionales, se podrá compartir internamente metadata inmutable.

---

# 304. Structural sharing

Puede reducir allocations en pipelines de transformación.

---

# 305. No premature optimization

La implementación deberá medirse antes de introducir estructuras excesivamente complejas.

---

# 306. Performance goals

Evitar en hot path:

- reflection;
- container lookups;
- stack traces;
- arbitrary arrays;
- repeated normalization;
- dynamic class discovery;
- serialization.

---

# 307. Metadata descriptors precompiled

En producción podrán precompilarse.

---

# 308. Metadata merge plans

Para extensiones complejas podrían precompilarse:

```text
MetadataMergePlan
```

durante bootstrap.

---

# 309. Determinism

Dadas las mismas:

```text
Query
Declared Metadata
Semantic Information
Policy Snapshot
Capability Snapshot
```

deberá obtenerse la misma:

```text
Effective Metadata
```

---

# 310. External mutable state

No deberá influir silenciosamente.

---

# 311. Current time

No utilizar current time para metadata resolution salvo que se suministre explícitamente mediante contexto apropiado.

---

# 312. Current user

No acceder globalmente.

---

# 313. Current tenant

No acceder globalmente.

---

# 314. Current request

No acceder globalmente.

---

# 315. Environment variables

No leerlas desde MetadataResolver.

---

# 316. Service Container

No consultarlo desde metadata internals.

---

# 317. Metadata API stability

Core metadata pública deberá mantenerse pequeña.

---

# 318. Extension metadata stability

Los contratos de extensión deberán versionarse.

---

# 319. Internal metadata

Detalles de optimizer/planner/compiler permanecerán Internal.

---

# 320. Public API

Podría exponer:

```text
QueryLabel
QueryOrigin
QueryIntent
ConnectionPreference
ConsistencyRequirement
StatementTimeout
ResultMode
```

---

# 321. Extension API

Podría exponer:

```text
MetadataDescriptor
MetadataKey
MetadataMergePolicy
MetadataInheritancePolicy
MetadataSensitivity
MetadataFingerprintPolicy
```

---

# 322. Internal API

Podría incluir:

```text
MetadataResolutionGraph
MetadataMergePlan
MetadataResolutionTrace
MetadataCanonicalizer
```

---

# 323. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Metadata\
```

---

# 324. Proposed structure

```text
Query/
└── Metadata/
    ├── Contract/
    │   ├── QueryMetadataInterface.php
    │   ├── MetadataResolverInterface.php
    │   ├── MetadataNormalizerInterface.php
    │   └── MetadataValidatorInterface.php
    │
    ├── Core/
    │   ├── QueryMetadata.php
    │   ├── EmptyQueryMetadata.php
    │   └── EffectiveQueryMetadata.php
    │
    ├── Identity/
    │   ├── QueryInstanceId.php
    │   ├── QueryLabel.php
    │   └── QueryShapeFingerprint.php
    │
    ├── Origin/
    │   ├── QueryOrigin.php
    │   └── QueryOriginMetadata.php
    │
    ├── Intent/
    │   ├── QueryIntent.php
    │   ├── ConnectionIntent.php
    │   └── ConnectionPreference.php
    │
    ├── Requirement/
    │   ├── QueryRequirements.php
    │   ├── ConsistencyRequirement.php
    │   ├── TransactionRequirement.php
    │   ├── CapabilityRequirementSet.php
    │   ├── ResultRequirement.php
    │   └── ExecutionRequirement.php
    │
    ├── Policy/
    │   ├── QueryPolicies.php
    │   ├── ExecutionPolicy.php
    │   ├── TimeoutPolicy.php
    │   ├── CachePolicy.php
    │   ├── RetrySafety.php
    │   └── IdempotencyClassification.php
    │
    ├── Security/
    │   ├── QuerySecurityMetadata.php
    │   ├── QuerySensitivity.php
    │   ├── ParameterSensitivity.php
    │   ├── RedactionPolicy.php
    │   ├── AuditRequirement.php
    │   └── RawSqlTrust.php
    │
    ├── Observability/
    │   ├── QueryTelemetryMetadata.php
    │   ├── QueryMetricClassification.php
    │   └── QuerySamplingHint.php
    │
    ├── Diagnostic/
    │   ├── QueryDiagnosticMetadata.php
    │   ├── SourceLocation.php
    │   └── DeveloperNote.php
    │
    ├── Provenance/
    │   ├── QueryProvenance.php
    │   ├── TransformationRecord.php
    │   └── TransformationId.php
    │
    ├── Extension/
    │   ├── ExtensionMetadataKey.php
    │   ├── ExtensionMetadataDescriptor.php
    │   ├── ExtensionMetadataRegistry.php
    │   └── QueryExtensionMetadata.php
    │
    ├── Resolution/
    │   ├── QueryMetadataResolver.php
    │   ├── QueryMetadataResolutionContext.php
    │   ├── MetadataValue.php
    │   ├── MetadataStrength.php
    │   └── MetadataCertainty.php
    │
    ├── Merge/
    │   ├── MetadataMergePolicy.php
    │   ├── MetadataMergeStrategy.php
    │   └── MetadataMergePlan.php
    │
    ├── Inheritance/
    │   ├── MetadataInheritancePolicy.php
    │   └── MetadataInheritanceResolver.php
    │
    ├── Fingerprint/
    │   ├── MetadataFingerprintPolicy.php
    │   └── MetadataFingerprintContributor.php
    │
    ├── Redaction/
    │   ├── MetadataRedactor.php
    │   └── SafeMetadataRenderer.php
    │
    ├── Diagnostics/
    │   ├── MetadataResolutionTrace.php
    │   └── MetadataDiagnosticRenderer.php
    │
    └── Exception/
```

---

# 325. Related side-artifact namespaces

Podrán existir:

```text
Query/Semantic/Metadata/
Query/Planning/Metadata/
Query/Compilation/Metadata/
Query/Execution/Metadata/
```

siempre que no se duplique el significado.

---

# 326. Metadata ownership matrix

| Información | Owner |
|---|---|
| Query label | Query Metadata |
| Query origin | Query Metadata |
| Explicit timeout | Query Metadata |
| Connection preference | Query Metadata |
| Parameter value | Binding Set |
| Parameter semantic type | Query Type System |
| Resolved column | Semantic Engine |
| Relation graph | Semantic Engine |
| Capability requirement | Semantic/Metadata |
| Selected capability strategy | Planner |
| Generated SQL | Compiler |
| Placeholder layout | CompiledQuery |
| Physical connection | ExecutionContext |
| Connection lease | ExecutionContext |
| Active transaction | TransactionContext |
| Result cursor | Result/Execution |
| Active telemetry span | Telemetry/Execution Context |
| Tenant runtime object | Multitenancy Context |
| Query provenance | Query Metadata / side artifact |

---

# 327. DB-QMETA-001

Query structure y Query metadata serán conceptos diferentes.

---

# 328. DB-QMETA-002

Query metadata y runtime state serán conceptos diferentes.

---

# 329. DB-QMETA-003

Query metadata no contendrá conexiones físicas.

---

# 330. DB-QMETA-004

Query metadata no contendrá ConnectionLease.

---

# 331. DB-QMETA-005

Query metadata no contendrá Transaction objects.

---

# 332. DB-QMETA-006

Query metadata no contendrá EntityManager.

---

# 333. DB-QMETA-007

Query metadata no contendrá UnitOfWork.

---

# 334. DB-QMETA-008

Query metadata no contendrá ResultCursor.

---

# 335. DB-QMETA-009

Query metadata no contendrá Request objects.

---

# 336. DB-QMETA-010

Query metadata no contendrá Service Container.

---

# 337. DB-QMETA-011

Query metadata será inmutable una vez publicada.

---

# 338. DB-QMETA-012

Las transformaciones producirán nuevos artifacts.

---

# 339. DB-QMETA-013

Metadata descriptors compartidos serán inmutables.

---

# 340. DB-QMETA-014

Los registries de metadata se congelarán después del bootstrap.

---

# 341. DB-QMETA-015

No existirá un arbitrary untyped metadata bag como API principal.

---

# 342. DB-QMETA-016

Extension metadata será namespaced.

---

# 343. DB-QMETA-017

Extension metadata tendrá descriptors explícitos.

---

# 344. DB-QMETA-018

Cada metadata extension deberá declarar merge policy.

---

# 345. DB-QMETA-019

Cada metadata extension deberá declarar inheritance policy cuando corresponda.

---

# 346. DB-QMETA-020

Cada metadata extension deberá declarar sensitivity cuando corresponda.

---

# 347. DB-QMETA-021

Cada metadata extension deberá declarar fingerprint behavior cuando corresponda.

---

# 348. DB-QMETA-022

QueryIntent no será equivalente a SQL keyword.

---

# 349. DB-QMETA-023

QueryIntent y ConnectionIntent permanecerán separados.

---

# 350. DB-QMETA-024

ConnectionIntent no seleccionará una conexión física.

---

# 351. DB-QMETA-025

ConsistencyRequirement no será equivalente a replica flag.

---

# 352. DB-QMETA-026

TransactionRequirement no contendrá una Transaction real.

---

# 353. DB-QMETA-027

Capability requirements utilizarán IDs semánticos, no nombres de vendor.

---

# 354. DB-QMETA-028

Requirements tendrán precedencia sobre preferences.

---

# 355. DB-QMETA-029

Security policy superior no podrá debilitarse mediante metadata inferior.

---

# 356. DB-QMETA-030

Metadata conflicts críticos producirán errores explícitos.

---

# 357. DB-QMETA-031

Metadata merge no utilizará last-write-wins universal.

---

# 358. DB-QMETA-032

Metadata precedence será determinista.

---

# 359. DB-QMETA-033

Query labels deberán ser seguras y preferentemente low-cardinality.

---

# 360. DB-QMETA-034

Raw SQL no se considerará seguro por simple metadata.

---

# 361. DB-QMETA-035

Raw SQL con intent desconocido se manejará conservadoramente.

---

# 362. DB-QMETA-036

Query metadata no almacenará parameter values.

---

# 363. DB-QMETA-037

Parameter sensitivity podrá referenciar ParameterId.

---

# 364. DB-QMETA-038

Semantic types no se duplicarán dentro de metadata.

---

# 365. DB-QMETA-039

Resolved symbols no se duplicarán dentro de metadata.

---

# 366. DB-QMETA-040

Semantic Graph permanecerá separado de QueryMetadata.

---

# 367. DB-QMETA-041

Planning decisions permanecerán separadas de declarative query requirements.

---

# 368. DB-QMETA-042

Compiled metadata permanecerá separada de source QueryMetadata.

---

# 369. DB-QMETA-043

ExecutionContext permanecerá separado de QueryMetadata.

---

# 370. DB-QMETA-044

Active telemetry resources no vivirán en QueryMetadata.

---

# 371. DB-QMETA-045

CancellationToken no vivirá en QueryMetadata.

---

# 372. DB-QMETA-046

Runtime deadline no será necesariamente QueryMetadata.

---

# 373. DB-QMETA-047

Statement timeout declarativo sí podrá ser QueryMetadata.

---

# 374. DB-QMETA-048

Retry policy será ejecutada por Resilience, no Metadata System.

---

# 375. DB-QMETA-049

Cache implementation será responsabilidad de Cache System.

---

# 376. DB-QMETA-050

Audit y Telemetry serán conceptos diferentes.

---

# 377. DB-QMETA-051

Diagnostic metadata no cambiará semántica de consulta.

---

# 378. DB-QMETA-052

Policy transformations y optimizer transformations serán diferentes.

---

# 379. DB-QMETA-053

Optimizer transformations deberán preservar semántica.

---

# 380. DB-QMETA-054

Policy transformations podrán cambiar semántica de manera explícita y registrada.

---

# 381. DB-QMETA-055

Query provenance podrá registrar transformaciones sin almacenar copias completas del AST.

---

# 382. DB-QMETA-056

Multitenancy será integración opcional.

---

# 383. DB-QMETA-057

Authorization será integración opcional.

---

# 384. DB-QMETA-058

Telemetry será integración opcional.

---

# 385. DB-QMETA-059

Cache será integración opcional.

---

# 386. DB-QMETA-060

QueryMetadata core no dependerá de HTTP.

---

# 387. DB-QMETA-061

QueryMetadata core no dependerá de Queue.

---

# 388. DB-QMETA-062

QueryMetadata core no dependerá de Authentication.

---

# 389. DB-QMETA-063

QueryMetadata core no dependerá de Multitenancy.

---

# 390. DB-QMETA-064

QueryMetadata core no dependerá de Telemetry.

---

# 391. DB-QMETA-065

QueryMetadata core no dependerá de Cache.

---

# 392. DB-QMETA-066

Query metadata deberá ser segura para runtimes persistentes.

---

# 393. DB-QMETA-067

No existirá current metadata global mutable.

---

# 394. DB-QMETA-068

Metadata resolution context será operation-scoped.

---

# 395. DB-QMETA-069

Metadata descriptors podrán compartirse entre coroutines si son inmutables.

---

# 396. DB-QMETA-070

La metadata de una coroutine/request no podrá filtrarse a otra.

---

# 397. DB-QMETA-071

Metadata fingerprinting será específico por propósito.

---

# 398. DB-QMETA-072

No existirá un fingerprint universal para toda metadata.

---

# 399. DB-QMETA-073

Telemetry-only metadata no deberá invalidar compiled SQL.

---

# 400. DB-QMETA-074

Compilation-affecting metadata sí deberá participar en compilation fingerprint.

---

# 401. DB-QMETA-075

Runtime values no participarán en metadata fingerprints salvo protocolo explícito que realmente lo requiera.

---

# 402. DB-QMETA-076

Sensitive values nunca participarán directamente en telemetry metadata.

---

# 403. DB-QMETA-077

Metadata serialization será versionada.

---

# 404. DB-QMETA-078

Metadata serializable no contendrá closures.

---

# 405. DB-QMETA-079

Metadata serializable no contendrá resources.

---

# 406. DB-QMETA-080

Metadata serializable no contendrá secrets.

---

# 407. DB-QMETA-081

Metadata validation será por fases.

---

# 408. DB-QMETA-082

Errores estructurales deberán detectarse antes de semantic analysis cuando sea posible.

---

# 409. DB-QMETA-083

Contradicciones semánticas deberán detectarse antes de planning.

---

# 410. DB-QMETA-084

Violaciones de policy deberán detectarse antes de execution cuando sea posible.

---

# 411. DB-QMETA-085

Runtime-only requirements se validarán en Execution.

---

# 412. DB-QMETA-086

Una subquery no podrá seleccionar una conexión física distinta dentro del mismo statement.

---

# 413. DB-QMETA-087

Requirements de subqueries se integrarán al statement principal cuando corresponda.

---

# 414. DB-QMETA-088

Set operations combinarán requirements de todas sus branches.

---

# 415. DB-QMETA-089

Security metadata resultante utilizará al menos el nivel más restrictivo requerido.

---

# 416. DB-QMETA-090

El camino común de metadata deberá mantenerse ligero.

---

# 417. DB-QMETA-091

No se crearán artifacts diagnósticos pesados cuando diagnostics estén desactivados.

---

# 418. DB-QMETA-092

No se capturarán stack traces por query en producción por defecto.

---

# 419. DB-QMETA-093

Source paths deberán poder redaccionarse.

---

# 420. DB-QMETA-094

Metadata renderers serán seguros por defecto.

---

# 421. DB-QMETA-095

`__toString()` no deberá utilizarse para exposición irrestricta de metadata.

---

# 422. DB-QMETA-096

MetadataResolver no realizará I/O de base de datos.

---

# 423. DB-QMETA-097

MetadataResolver no leerá environment variables directamente.

---

# 424. DB-QMETA-098

MetadataResolver no consultará Service Container global.

---

# 425. DB-QMETA-099

MetadataResolver no accederá globalmente al current user.

---

# 426. DB-QMETA-100

MetadataResolver no accederá globalmente al current tenant.

---

# 427. Anti-pattern — metadata bag

Incorrecto:

```php
$query->metadata['foo'] = $bar;
```

Correcto:

```text
Typed Metadata
+
Registered Descriptor
+
Validation
+
Merge Policy
```

---

# 428. Anti-pattern — connection in metadata

Incorrecto:

```text
QueryMetadata
└── PDOConnection
```

Correcto:

```text
QueryMetadata
└── ConnectionRequirement

ExecutionContext
└── ConnectionLease
```

---

# 429. Anti-pattern — transaction in metadata

Incorrecto:

```text
QueryMetadata
└── currentTransaction
```

Correcto:

```text
QueryMetadata
└── TransactionRequirement

TransactionContext
└── current transaction
```

---

# 430. Anti-pattern — tenant object in metadata

Incorrecto:

```text
QueryMetadata
└── TenantEntity
```

Correcto:

```text
Multitenancy Adapter
→ Query Policy / Connection Requirement
```

---

# 431. Anti-pattern — telemetry span

Incorrecto:

```text
QueryMetadata
└── activeSpan
```

Correcto:

```text
QueryMetadata
└── telemetry classification

ExecutionContext
└── activeSpan
```

---

# 432. Anti-pattern — SQL as metadata

Incorrecto:

```text
metadata.sql = "SELECT ..."
```

SQL pertenece al compiled artifact.

---

# 433. Anti-pattern — result in metadata

Incorrecto:

```text
metadata.result = [...]
```

---

# 434. Anti-pattern — optimizer decision in source metadata

Incorrecto:

```text
metadata.joinAlgorithm = HASH_JOIN
```

desde Builder portable.

Correcto:

```text
Planner
→ PhysicalPlan decision
```

---

# 435. Anti-pattern — vendor flag

Incorrecto:

```text
metadata.postgresql = true
```

Correcto:

```text
CapabilityRequirement
```

o una extensión platform-specific explícita.

---

# 436. Anti-pattern — disable security

Incorrecto:

```text
$query->metadata([
    'redact' => false,
]);
```

cuando framework policy exige redacción.

---

# 437. Anti-pattern — retry all

Incorrecto:

```text
retry = true
```

sin clasificación semántica.

Correcto:

```text
RetrySafety
+
Execution Failure Classification
+
Transaction State
+
Resilience Policy
```

---

# 438. Anti-pattern — cache all SELECT

Incorrecto:

```text
SELECT
→ automatically safe to cache
```

La seguridad depende de:

- consistency;
- sensitivity;
- transaction;
- query semantics;
- tenant/data scope;
- cache policy.

---

# 439. Anti-pattern — query origin controls security

Origin:

```text
INTERNAL
```

no significa:

```text
trusted
```

automáticamente.

---

# 440. Anti-pattern — metadata becomes context

Incorrecto:

```text
QueryMetadata
├── current user
├── current tenant
├── current transaction
├── current connection
└── current span
```

Esto sería un Service Context disfrazado.

---

# 441. Correct architecture

```text
Query Artifact
├── Query Structure
└── Declarative Metadata

Semantic Artifact
├── Resolved Types
├── Resolved Symbols
├── Requirements
└── Semantic Annotations

Plan Artifact
├── Strategy Decisions
└── Plan Metadata

Compiled Artifact
├── SQL
├── Binding Layout
└── Compiled Metadata

Execution Context
├── Connection Lease
├── Transaction
├── Cancellation
├── Telemetry
└── Cleanup
```

---

# 442. Example complete lifecycle

Developer:

```php
DB::table('orders')
    ->label('orders.pending')
    ->consistency(Consistency::READ_YOUR_WRITES)
    ->timeout(seconds: 5)
    ->where('status', OrderStatus::Pending)
    ->get();
```

---

# 443. Declared metadata

```text
Label:
orders.pending

Consistency:
READ_YOUR_WRITES

StatementTimeout:
5 seconds

Origin:
APPLICATION
```

---

# 444. Query Model

```text
SelectQuery
├── Source: orders
└── Predicate:
    status = Parameter(p1)
```

---

# 445. Type analysis

```text
orders.status
→ OrderStatusType

p1
→ OrderStatusType
```

---

# 446. Derived metadata

```text
QueryIntent:
READ

ConnectionIntent:
READ

TransactionRequirement:
OPTIONAL
```

---

# 447. Policy resolution

Because:

```text
Consistency:
READ_YOUR_WRITES
```

effective connection requirement could become:

```text
PRIMARY
```

depending on topology/runtime state.

---

# 448. Planner

Produces:

```text
Logical Plan
      │
      ▼
Physical Plan
```

without modifying source metadata.

---

# 449. Compiler

Produces:

```text
CompiledQuery
├── SQL
├── bindings layout
├── result metadata
└── execution requirements
```

---

# 450. Execution

Runtime creates:

```text
ExecutionContext
├── ExecutionId
├── primary connection lease
├── statement deadline
├── telemetry span
└── cleanup tracker
```

---

# 451. After execution

`ExecutionContext` is disposed.

The immutable query artifact may remain reusable.

---

# 452. Query reuse

A reusable prepared Query Model must not retain:

```text
previous connection
previous transaction
previous tenant
previous bindings
previous result
previous telemetry span
```

---

# 453. Parameter bindings remain separate

Same query shape:

```text
status = :p1
```

can execute with different binding sets.

---

# 454. Metadata reuse

Stable metadata such as:

```text
label
origin
declared consistency
```

may be reused.

---

# 455. Runtime-derived metadata caution

Anything dependent on execution-specific state should not be cached into the reusable source query.

---

# 456. Effective metadata scope

`EffectiveQueryMetadata` may be specific to:

```text
semantic analysis
planning target
execution target
```

and therefore should have explicit ownership.

---

# 457. Capability snapshot

If effective metadata depends on:

```text
CapabilitySnapshot
```

its cache/fingerprint must include the capability fingerprint.

---

# 458. Topology state

Dynamic replica health should generally not become part of reusable QueryMetadata.

---

# 459. Routing resolution

```text
Query Requirements
+
Transaction Affinity
+
Consistency
+
Topology State
+
Runtime Health
→ Connection Resolution
```

---

# 460. This belongs below metadata

Specifically:

```text
Connection Manager / Topology Resolver
```

---

# 461. Metadata and deterministic compilation

Only metadata affecting compilation shall influence compiled SQL cache.

---

# 462. Example compilation-neutral metadata

```text
QueryLabel
SourceLocation
TelemetrySamplingHint
DeveloperNote
```

---

# 463. Example compilation-affecting metadata

Potentially:

```text
LockingRequirement
PlatformSpecificCompilerHint
ResultShapeRequirement
```

depending on implementation.

---

# 464. Query shape cache

Should not fragment because of diagnostic labels.

---

# 465. Result cache

May require more contextual metadata than compiled query cache.

---

# 466. Security-sensitive cache separation

Tenant/data-scope/security metadata must never accidentally allow cross-scope result reuse.

---

# 467. Cache safety invariant

If a metadata dimension changes visibility of data, it must affect cache isolation or disable caching.

---

# 468. Metadata sensitivity propagation

```text
Sensitive Parameter
      │
      ▼
Sensitive Query Classification
      │
      ├── Logging Redaction
      ├── Profiler Redaction
      ├── Trace Redaction
      └── Cache Restrictions
```

---

# 469. Sensitive result propagation

Similarly:

```text
Sensitive Result
→ restricted diagnostics
```

---

# 470. Metadata and observability

Telemetry should consume metadata, not mutate the query.

---

# 471. Observer isolation

An observability adapter must not change:

- query intent;
- connection requirements;
- transaction requirements;
- SQL;
- bindings.

---

# 472. Audit exception

Audit policies can reject an operation if mandatory audit infrastructure is unavailable, but this belongs to security/policy integration rather than passive telemetry.

---

# 473. Metadata testing

Required tests:

```text
Construction Tests
Immutability Tests
Merge Tests
Inheritance Tests
Validation Tests
Fingerprint Tests
Redaction Tests
Extension Tests
Policy Tests
Persistent Runtime Tests
Concurrency Tests
Security Tests
```

---

# 474. Merge tests

Examples:

```text
PRIMARY required + REPLICA preferred
→ PRIMARY

audit REQUIRED + audit NONE
→ REQUIRED / policy violation

transaction REQUIRED + FORBIDS
→ conflict
```

---

# 475. Inheritance tests

Verify:

- subqueries;
- CTEs;
- unions;
- nested expressions;
- policy transformations.

---

# 476. Fingerprint tests

Verify that:

```text
different telemetry label
```

does not change compiled fingerprint when SQL is identical.

---

# 477. Security tests

Verify:

- no binding values in metadata;
- redacted rendering;
- extension metadata validation;
- high-cardinality protection;
- no policy weakening.

---

# 478. Persistent runtime tests

Sequence:

```text
Request A
QueryMetadata A
Execute
Dispose

Request B
QueryMetadata B
Execute
```

Assert:

```text
A metadata/context not visible in B
```

---

# 479. Concurrency tests

For OpenSwoole-style execution:

```text
Coroutine A
Coroutine B
```

sharing only immutable registries.

---

# 480. Architecture tests

Fail if Query Metadata imports:

```text
PDO
NativeConnection
ConnectionLease
EntityManager
UnitOfWork
HTTP Request
Telemetry Span implementation
Tenant entity
Service Container
```

---

# 481. V1 priorities

V1 should implement:

```text
QueryMetadata
QueryLabel
QueryOrigin
QueryIntent
ConnectionIntent
ConnectionPreference
ConsistencyRequirement
TransactionRequirement
CapabilityRequirementSet
StatementTimeout
ResultMode
QuerySecurityMetadata
QueryTelemetryMetadata
QueryProvenance
MetadataMergePolicy
MetadataInheritancePolicy
EffectiveQueryMetadata
```

---

# 482. V1 extension support

Provide:

```text
ExtensionMetadataKey
ExtensionMetadataDescriptor
ExtensionMetadataRegistry
```

with:

- validation;
- merge;
- inheritance;
- sensitivity;
- fingerprint policy.

---

# 483. V1 diagnostics

Provide safe:

```text
MetadataDiagnosticRenderer
MetadataResolutionTrace
```

in development.

---

# 484. V2

Possible additions:

```text
Advanced Query Policies
Compiled Metadata Merge Plans
Query Source Maps
Fine-grained Parameter Sensitivity
Advanced Cache Metadata
Optimizer Provenance
Policy Explain Graph
Advanced Audit Classification
```

---

# 485. V3

Possible additions:

```text
Distributed Query Provenance
Cross-service Query Correlation
Advanced Data Governance Metadata
Static Query Metadata Analysis
Policy Compilation
Metadata Cost Model
```

---

# 486. Core metadata formula

```text
QueryMetadata
=
Identity
+
Origin
+
Intent
+
Requirements
+
Policies
+
Security
+
Observability
+
Provenance
+
Extensions
```

---

# 487. Effective metadata formula

```text
EffectiveQueryMetadata
=
Declared Metadata
+
Derived Semantic Metadata
+
Framework Policy
+
Security Policy
+
Capability Constraints
+
Extension Rules
```

---

# 488. Runtime execution formula

```text
ExecutionContext
=
Effective Query Requirements
+
DatabaseContext
+
TransactionContext
+
Topology State
+
Connection Resolution
+
Runtime Deadline
+
Cancellation
+
Telemetry Context
```

---

# 489. Critical separation formula

```text
Declarative Metadata
≠
Derived Semantics
≠
Planning Decision
≠
Compiled Artifact
≠
Runtime Resource
```

---

# 490. Metadata safety formula

```text
Safe Query Metadata
=
Typed Values
+
Immutability
+
Explicit Ownership
+
Deterministic Merge
+
Controlled Inheritance
+
Redaction
+
Fingerprint Discipline
+
No Runtime Resources
```

---

# 491. Persistent runtime formula

```text
Persistent-Safe Metadata
=
Immutable Shared Descriptors
+
Query-Owned Metadata
+
Operation-Scoped Resolution
+
No Static Current Context
+
No Request Objects
+
No Connection Resources
+
Deterministic Cleanup
```

---

# 492. Query Engine relationship

Los documentos:

```text
23_DATABASE_QUERY_ARCHITECTURE.md
24_DATABASE_QUERY_MODEL.md
25_DATABASE_QUERY_AST_SYSTEM.md
26_DATABASE_QUERY_AST_NODE_MODEL.md
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
30_DATABASE_QUERY_TYPE_SYSTEM.md
31_DATABASE_QUERY_METADATA_SYSTEM.md
```

establecen hasta este punto:

```text
Query Model
     │
     ▼
Query AST
     │
     ├── Expressions
     ├── Predicates
     ├── Parameters
     ├── Types
     └── Metadata
     │
     ▼
Semantic Analysis
```

---

# 493. Important refinement

Aunque el diagrama anterior agrupa conceptos visualmente, deberá recordarse:

```text
AST
≠
Types Table
≠
Metadata
```

La relación real será:

```text
                  Query Artifact
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
           AST              Query Metadata
             │
             ▼
      Semantic Analysis
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
    Types Symbols Requirements
       │     │     │
       └─────┼─────┘
             ▼
       Semantic Graph
```

---

# 494. Architectural boundary

El Query Metadata System será la frontera entre:

```text
"What is this query?"
```

y:

```text
"What requirements and policies accompany this query?"
```

sin responder todavía:

```text
"How exactly will it execute?"
```

---

# 495. Master rule

> Query Metadata deberá describir la consulta y sus requisitos sin poseer los recursos que finalmente serán utilizados para ejecutarla.

---

# 496. Final rule

VoltStack preservará permanentemente:

```text
Query Structure
      ≠
Query Metadata
      ≠
Semantic Analysis State
      ≠
Planning State
      ≠
Compilation State
      ≠
Execution State
```

---

# 497. Resultado arquitectónico

Con este sistema, una consulta podrá viajar por el Query Engine como un artifact estructurado:

```text
Query
├── Semantic Structure
└── Declarative Metadata
```

mientras cada etapa añade sus propios artifacts:

```text
Query
 │
 ├── QueryMetadata
 │
 ▼
AST
 │
 ├── SemanticTypeTable
 ├── SymbolTable
 ├── SemanticAnnotations
 │
 ▼
SemanticGraph
 │
 ├── EffectiveRequirements
 │
 ▼
LogicalPlan
 │
 ▼
PhysicalPlan
 │
 ▼
ExecutionPlan
 │
 ▼
CompiledQuery
 │
 ▼
ExecutionContext
```

sin convertir ningún objeto en un contenedor universal de estado.

---

# 498. Próximo documento

El siguiente documento será:

```text
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
```

y deberá formalizar el estado temporal necesario durante el procesamiento de una consulta.

La separación central será:

```text
QueryMetadata
≠
QueryContext
≠
ExecutionContext
≠
DatabaseContext
≠
TransactionContext
```

`QueryContext` será operation-scoped y podrá coordinar información temporal necesaria durante:

```text
Normalization
Validation
Semantic Analysis
Optimization
Planning
Compilation
```

sin almacenar estado global ni sobrevivir accidentalmente entre requests, workers, fibers o coroutines.