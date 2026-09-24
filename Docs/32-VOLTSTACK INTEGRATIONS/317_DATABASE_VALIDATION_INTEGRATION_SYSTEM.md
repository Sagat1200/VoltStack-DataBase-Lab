# 317_DATABASE_VALIDATION_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Validation Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 317 — Database Validation Integration System  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `318_DATABASE_AUTHENTICATION_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de integración entre:

```text
VoltStack/Quantum/Database
```

y:

```text
VoltStack Validation
```

El objetivo es permitir que las aplicaciones puedan validar datos relacionados con persistencia de manera:

- segura;
- predecible;
- tipada;
- eficiente;
- tenant-aware;
- transaction-aware;
- connection-aware;
- compatible con ORM;
- compatible con Query Engine;
- compatible con persistent runtimes;
- extensible;
- observable;
- testeable.

La regla fundamental será:

> **Validation determina si un dato satisface reglas de aplicación bajo la evidencia disponible; Database determina y hace cumplir las garantías de persistencia que corresponden al almacenamiento.**

Por tanto:

```text
Validation
≠
Database Constraint
```

```text
Validation Success
≠
Persistence Success
```

y especialmente:

```text
Application Uniqueness Check
≠
Atomic Uniqueness Guarantee
```

---

# 2. Problema arquitectónico

VoltStack necesitará reglas como:

```php
'email' => [
    'required',
    'email',
    Rule::unique('users', 'email'),
],
```

o:

```php
Rule::exists('roles', 'id');
```

Estas reglas necesitan consultar Database.

Sin una arquitectura explícita podría terminarse con:

```text
Validator
   ↓
raw SQL
   ↓
PDO
```

o incluso:

```text
Validator
   ↓
ORM internals
   ↓
Driver
```

Esto violaría las fronteras de VoltStack.

La integración correcta será:

```text
Validation
     ↓
Database Validation Bridge
     ↓
Query Engine
     ↓
Execution Engine
     ↓
Connection
     ↓
Driver
```

---

# 3. Regla arquitectónica principal

Validation nunca deberá saltarse Database.

Incorrecto:

```text
Validation
   ↓
PDO
```

Incorrecto:

```text
Validation
   ↓
Driver
```

Incorrecto:

```text
Validation
   ↓
SQL string
```

Correcto:

```text
Validation
   ↓
DatabaseValidationGateway
   ↓
Query Model / Query Builder
   ↓
Database Query Pipeline
```

---

# 4. Objetivos

La integración deberá proporcionar:

1. reglas `exists`;
2. reglas `unique`;
3. validación sobre modelos;
4. validación sobre entidades;
5. validación previa a persistencia;
6. validación contextual;
7. integración ORM;
8. integración Model API;
9. integración Repository;
10. tenant-aware validation;
11. shard-aware validation;
12. connection-aware validation;
13. transaction-aware validation;
14. soft-delete awareness;
15. scope-aware validation;
16. composite uniqueness;
17. batch validation;
18. reglas custom basadas en Database;
19. seguridad de identifiers;
20. parameter binding;
21. prevención de SQL injection;
22. race-condition awareness;
23. constraint error translation;
24. telemetry;
25. persistent runtime isolation;
26. testing.

---

# 5. No objetivos

Este sistema no pretende convertir Validation en:

```text
ORM
```

ni:

```text
Schema Engine
```

ni:

```text
Constraint Engine
```

ni:

```text
Authorization Engine
```

ni:

```text
Database Transaction Manager
```

---

# 6. Separaciones fundamentales

VoltStack deberá preservar:

```text
Validation
≠
Persistence
```

```text
Validation
≠
Schema Constraint
```

```text
Validation
≠
Authorization
```

```text
Validation
≠
Sanitization
```

```text
Validation
≠
Type Conversion
```

```text
Validation
≠
Database Query
```

```text
Validation Rule
≠
Database Constraint
```

```text
Unique Validation
≠
UNIQUE Constraint
```

```text
Exists Validation
≠
Foreign Key Constraint
```

---

# 7. Validation vs database constraints

Supongamos:

```text
users.email UNIQUE
```

Validation puede ejecutar:

```text
Does email X already exist?
```

y obtener:

```text
NO
```

Pero inmediatamente después otra transacción puede insertar:

```text
email = X
```

antes del INSERT original.

Por tanto:

```text
T1: validate unique(X) → available
T2: validate unique(X) → available
T1: INSERT X
T2: INSERT X
```

Sólo:

```text
UNIQUE INDEX / UNIQUE CONSTRAINT
```

puede garantizar atomicidad en el almacenamiento.

---

# 8. TOCTOU

Este problema corresponde a:

```text
Time Of Check
    ↓
Time Of Use
```

La validación ocurre en un instante diferente a la persistencia.

Formalmente:

```text
Validate(t1)
≠
Guarantee(t2)
```

cuando:

```text
t2 > t1
```

y existe concurrencia.

---

# 9. Regla de unicidad

VoltStack deberá establecer:

> **Toda regla de unicidad que represente una invariancia real del dominio persistente deberá estar respaldada por una restricción atómica en Database cuando la plataforma pueda expresarla.**

---

# 10. Arquitectura general

```text
Application
    │
    ▼
Validation Engine
    │
    ├── Pure Rules
    │
    └── Database Rules
             │
             ▼
 DatabaseValidationBridge
             │
             ▼
 DatabaseValidationGateway
             │
             ▼
        Query Engine
             │
             ▼
      Execution Engine
             │
             ▼
       Connection
             │
             ▼
          Driver
             │
             ▼
          DBMS
```

---

# 11. Validation Bridge

Se propone:

```php
interface DatabaseValidationGateway
{
    public function exists(
        DatabaseExistenceRequirement $requirement,
        DatabaseValidationContext $context,
    ): DatabaseExistenceResult;

    public function isUnique(
        DatabaseUniquenessRequirement $requirement,
        DatabaseValidationContext $context,
    ): DatabaseUniquenessResult;
}
```

---

# 12. Gateway responsibility

El Gateway será responsable de traducir:

```text
Validation Intent
```

a:

```text
Database Query Model
```

pero no de decidir:

```text
Validation Message
HTTP Response
Form State
```

---

# 13. Validation rule flow

```text
Rule
 ↓
Requirement
 ↓
DatabaseValidationGateway
 ↓
Query Model
 ↓
Execution
 ↓
DatabaseValidationResult
 ↓
Rule Result
 ↓
Validation Error?
```

---

# 14. Rule::exists()

API propuesta:

```php
Rule::exists('users', 'id');
```

Ejemplo:

```php
$validator->validate([
    'user_id' => 42,
], [
    'user_id' => [
        'required',
        Rule::exists('users', 'id'),
    ],
]);
```

---

# 15. Exists semantics

Conceptualmente:

```text
Exists(table, column, value)
```

deberá preguntar:

```text
∃ row ∈ table
where row[column] = value
```

bajo el contexto Database aplicable.

---

# 16. Exists query

El Query Engine podrá generar conceptualmente:

```sql
SELECT 1
FROM users
WHERE id = ?
LIMIT 1
```

pero Validation no construirá ese SQL directamente.

---

# 17. Rule::unique()

API:

```php
Rule::unique('users', 'email');
```

Ejemplo:

```php
'email' => [
    'required',
    'email',
    Rule::unique('users', 'email'),
]
```

---

# 18. Unique semantics

Conceptualmente:

```text
Unique(table, column, value)
```

significa:

```text
¬∃ row
where row[column] = value
```

bajo el scope configurado.

---

# 19. Update scenario

Al actualizar:

```text
User #42
```

el propio registro debe poder ignorarse.

API:

```php
Rule::unique('users', 'email')
    ->ignore($user->id);
```

---

# 20. Ignore semantics

Conceptualmente:

```text
WHERE email = ?
AND id <> ?
```

pero nuevamente:

```text
Rule
≠
SQL Generator
```

---

# 21. Ignore by entity

API conveniente:

```php
Rule::unique('users', 'email')
    ->ignoreEntity($user);
```

El sistema resolverá:

```text
Entity Metadata
↓
Identifier
```

sin pedir al desarrollador que repita información.

---

# 22. Security rule for ignore

Nunca aceptar automáticamente:

```php
->ignore($_POST['id']);
```

como identificador confiable de la entidad actual sin validación contextual.

---

# 23. Composite uniqueness

Ejemplo:

```text
tenant_id + email
```

debe ser único.

API conceptual:

```php
Rule::unique('users', 'email')
    ->where('tenant_id', $tenantId);
```

o preferiblemente mediante contexto tenant automático.

---

# 24. Composite requirement

Modelo:

```text
DatabaseUniquenessRequirement
├── Resource
├── ValueField
├── Value
├── ScopePredicates
├── IgnoreIdentity?
├── ConnectionIntent?
└── Metadata
```

---

# 25. Entity-aware API

Podrá existir:

```php
Rule::uniqueEntity(User::class, 'email');
```

en lugar de:

```php
Rule::unique('users', 'email');
```

---

# 26. Entity metadata resolution

```text
User::class
   ↓
EntityMetadata
   ↓
table = users
column = email
identifier = id
```

---

# 27. Table API vs entity API

Ambas podrán coexistir:

```text
Table-oriented Validation
Entity-oriented Validation
```

pero convergerán sobre:

```text
DatabaseValidationGateway
```

---

# 28. Model API integration

Ejemplo:

```php
User::validate([
    'email' => $email,
]);
```

podrá ser una conveniencia.

Pero:

```text
Model API
≠
Validation Engine
```

---

# 29. Entity validation

Podrá existir:

```php
$validator->validateEntity($user);
```

usando metadata o reglas declaradas.

---

# 30. Entity metadata ≠ validation metadata

No deberá asumirse:

```text
VARCHAR(255)
```

como única fuente de reglas de aplicación.

---

# 31. Schema-derived hints

Schema/ORM metadata podrá aportar hints como:

```text
nullable
max storage length
enum mapping
```

pero las reglas de dominio pertenecen a Validation/Application.

---

# 32. Example distinction

DB:

```text
name VARCHAR(255)
```

Dominio:

```text
name required
minimum 3
maximum 100
```

Por tanto:

```text
Storage Capacity
≠
Business Validation Rule
```

---

# 33. Automatic rule generation

VoltStack podrá opcionalmente derivar reglas básicas desde metadata.

Pero deberán considerarse:

```text
generated hints
```

no reemplazo del contrato de dominio.

---

# 34. Validation lifecycle

Flujo recomendado:

```text
Input
 ↓
Structural Validation
 ↓
Type/Format Validation
 ↓
Database-backed Validation
 ↓
Domain Validation
 ↓
Application Operation
 ↓
Persistence
 ↓
Database Constraints
```

---

# 35. Cheap rules first

No ejecutar:

```text
SELECT ...
```

si previamente:

```text
email format invalid
```

ya invalida el dato.

---

# 36. Rule scheduling

Validation podrá ordenar:

```text
PURE
CPU_LOCAL
DATABASE
EXTERNAL
```

para reducir I/O innecesario.

---

# 37. Database validation batching

Problema:

```text
100 items
×
Rule::exists()
=
100 queries
```

puede crear:

```text
N+1 validation
```

---

# 38. Batch validation

El sistema deberá poder transformar conceptualmente:

```text
exists(id=1)
exists(id=2)
exists(id=3)
...
```

en:

```text
WHERE id IN (...)
```

cuando las reglas sean compatibles.

---

# 39. Batch compatibility

Dos reglas podrán agruparse sólo si comparten:

```text
resource
column
connection context
tenant
shard
transaction
scope predicates
soft-delete policy
```

---

# 40. Batch result

```text
Requested:
[1, 2, 3, 4]

Found:
[1, 3, 4]

Missing:
[2]
```

---

# 41. Unique batching

También podrá evaluarse:

```text
emails = [
  a@example.com,
  b@example.com,
  c@example.com
]
```

mediante una consulta agrupada.

---

# 42. Input duplicates

Debe distinguirse:

```text
duplicate in submitted batch
```

de:

```text
duplicate already persisted
```

---

# 43. Example

Entrada:

```text
a@example.com
a@example.com
```

puede violar reglas aun si Database todavía no contiene ese email.

---

# 44. Validation context

Se propone:

```text
DatabaseValidationContext
├── OperationId
├── LogicalDatabase
├── ConnectionIntent
├── TransactionContext?
├── TenantContext?
├── ShardContext?
├── SoftDeletePolicy?
├── ConsistencyRequirement
├── MetadataGeneration
└── ValidationPolicy
```

---

# 45. Context scope

Será:

```text
request
job
command
operation
```

según runtime.

Nunca global mutable.

---

# 46. Connection selection

Una validación Database deberá especificar intención.

Ejemplo:

```text
CONSISTENCY_SENSITIVE_READ
```

para unicidad previa a escritura.

---

# 47. Replica danger

Supongamos:

```text
Writer:
email X exists

Replica:
email X not replicated yet
```

Una validación `unique` contra réplica puede devolver:

```text
available
```

incorrectamente respecto al writer.

---

# 48. Unique read policy

Por defecto, una validación de unicidad asociada a una escritura deberá preferir:

```text
writer
```

o una fuente que satisfaga la consistencia requerida.

---

# 49. Exists read policy

`exists` puede tener políticas diferentes según el uso.

Por ejemplo:

```text
informational existence check
```

podría tolerar replica.

Pero:

```text
foreign-reference validation before write
```

puede requerir writer/transaction context.

---

# 50. Validation consistency

Se propone:

```text
EVENTUAL
SESSION
WRITE_CONSISTENT
TRANSACTIONAL
```

como intenciones conceptuales.

La implementación deberá mapearlas a capacidades reales.

---

# 51. Requested consistency ≠ effective consistency

No deberá afirmarse:

```text
TRANSACTIONAL
```

si la infraestructura no puede proporcionarla.

---

# 52. Transaction-aware validation

Dentro de:

```php
DB::transaction(function () {
    // validation
    // persistence
});
```

las reglas Database deberán participar en el contexto apropiado cuando sea necesario.

---

# 53. Same transaction visibility

Esto permite que Validation observe:

```text
writes performed earlier
inside same transaction
```

cuando el DBMS y aislamiento lo permitan.

---

# 54. Transaction-aware ≠ race-free

Incluso dentro de una transacción:

```text
SELECT no row
```

no implica necesariamente que otra transacción no pueda insertar la misma clave.

---

# 55. Unique constraint remains mandatory

Por tanto:

```text
Transactional validation
+
UNIQUE constraint
```

son complementarios.

---

# 56. Validation and locks

No deberá adquirirse automáticamente:

```text
FOR UPDATE
```

para cada `unique()`.

Esto podría:

- reducir concurrencia;
- causar deadlocks;
- no resolver todos los casos;
- introducir semánticas inesperadas.

---

# 57. Explicit lock policy

Si una regla avanzada requiere lock, deberá declararlo mediante una política especializada.

---

# 58. Validation ≠ concurrency control

Regla:

```text
Validation
≠
Locking Strategy
```

---

# 59. Tenant-aware validation

Con Multitenancy instalado:

```text
TenantContext
    ↓
DatabaseValidationContext
```

deberá integrarse automáticamente.

---

# 60. Tenant uniqueness

Ejemplo:

```text
email unique per tenant
```

podrá traducirse conceptualmente a:

```text
tenant_id = current tenant
AND email = ?
```

en shared-table tenancy.

---

# 61. Database-per-tenant

En ese modelo:

```text
TenantContext
↓
TenantConnectionResolver
↓
Tenant Database
```

y la regla no necesita necesariamente predicado `tenant_id`.

---

# 62. Schema-per-tenant

Podrá requerir:

```text
TenantContext
↓
Schema Resolution
```

antes de query planning.

---

# 63. Tenant isolation rule

Nunca deberá permitirse que input no confiable cambie:

```text
validation tenant context
```

arbitrariamente.

---

# 64. Cross-tenant validation

Sólo mediante contexto administrativo explícito y autorizado.

---

# 65. Multitenancy optionality

Database Validation Core no dependerá de:

```text
VoltStack Multitenancy
```

Utilizará:

```text
optional context provider
```

---

# 66. Shard-aware validation

Si el recurso está shardeado:

```text
Validation Requirement
↓
Partition Routing
↓
Shard
↓
Query
```

---

# 67. Single-shard requirement

Cuando la regla pueda resolverse desde shard key:

```text
single shard
```

será preferido.

---

# 68. Global uniqueness in sharded systems

Una regla:

```text
email globally unique
```

en un sistema shardeado no puede asumirse localmente.

---

# 69. Global uniqueness strategies

Podrían requerir:

```text
central uniqueness registry
global index
coordinator
partitioning by unique key
application reservation system
```

según arquitectura.

---

# 70. Validation must not fake global guarantees

Si sólo se consulta un shard:

```text
unique on shard
≠
globally unique
```

---

# 71. Soft delete integration

Supongamos:

```text
users.deleted_at
```

La regla deberá declarar si los registros soft-deleted:

```text
count
```

o:

```text
do not count
```

para unicidad/existencia.

---

# 72. No implicit assumption

No deberá asumirse automáticamente:

```text
soft deleted = nonexistent
```

---

# 73. APIs

Ejemplo:

```php
Rule::uniqueEntity(User::class, 'email')
    ->withoutSoftDeleted();
```

o:

```php
->includingSoftDeleted();
```

---

# 74. Relationship validation

Podrá validarse:

```text
related entity exists
relationship allowed
relationship cardinality
```

pero deberá distinguirse:

```text
Existence
```

de:

```text
Authorization
```

---

# 75. Example

Que:

```text
Project #42 exists
```

no significa:

```text
Current user may assign Project #42
```

---

# 76. Validation + Authorization

Flujo posible:

```text
exists
↓
valid input

authorization
↓
allowed operation
```

pero son sistemas independientes.

---

# 77. Foreign key validation

Una regla `exists` mejora UX:

```text
"The selected role does not exist."
```

Pero el FK sigue siendo la garantía de integridad.

---

# 78. Exists ≠ Foreign Key

La validación puede quedar obsoleta entre check y write.

---

# 79. Constraint translation

Cuando Database rechace una operación por:

```text
UNIQUE violation
FOREIGN KEY violation
CHECK violation
NOT NULL violation
```

el sistema podrá traducir el error hacia una forma consumible por Validation/Application.

---

# 80. Constraint violation mapper

Se propone:

```php
interface ConstraintViolationMapper
{
    public function map(
        DatabaseConstraintViolation $violation,
        ValidationMappingContext $context,
    ): ?ValidationViolation;
}
```

---

# 81. Constraint name mapping

Idealmente utilizar:

```text
stable constraint metadata
```

para relacionar:

```text
users_email_unique
```

con:

```text
email
```

---

# 82. Vendor messages

No deberán parsearse como primera estrategia si existe metadata estructurada.

---

# 83. Constraint violation ≠ validation exception

Database deberá conservar su excepción canónica.

La capa de aplicación podrá transformarla.

---

# 84. Example flow

```text
Validation says email available
        ↓
Concurrent INSERT occurs
        ↓
Application INSERT
        ↓
UNIQUE constraint violation
        ↓
Database canonical exception
        ↓
ConstraintViolationMapper
        ↓
Validation violation: email already used
```

---

# 85. This closes the race at UX level

La garantía sigue siendo Database.

Validation sólo transforma el resultado para la aplicación.

---

# 86. Persistence validation hooks

Podrán existir hooks como:

```text
beforePersistValidation
beforeFlushValidation
```

pero deberán ser explícitos.

---

# 87. Automatic validation before flush

No deberá imponerse universalmente sin política porque:

```text
flush
```

puede incluir muchas entidades y contextos diferentes.

---

# 88. Validation policy

Ejemplo:

```text
NONE
EXPLICIT
ON_PERSIST
ON_FLUSH
DOMAIN_CONTROLLED
```

---

# 89. Recommended default

Para V1:

```text
EXPLICIT
```

como comportamiento base.

Model API podrá ofrecer ergonomía adicional.

---

# 90. Why explicit

Evita que:

```php
$entityManager->flush();
```

dispare inesperadamente cientos de consultas de validación.

---

# 91. Domain validation

Reglas como:

```text
creditLimit >= 0
endDate >= startDate
```

no necesitan Database necesariamente.

Deberán permanecer fuera del Database Validation Gateway.

---

# 92. Database-backed domain rules

Algunas sí pueden necesitar Database:

```text
customer cannot have > N active contracts
```

pero requieren modelado explícito.

---

# 93. Aggregate validation

Estas reglas deberán poder expresarse mediante:

```text
Query Builder
```

o servicios de dominio especializados.

---

# 94. Generic validator caution

No toda regla empresarial compleja deberá convertirse en:

```php
Rule::database(...)
```

---

# 95. Custom database rule

Contrato posible:

```php
interface DatabaseValidationRule
{
    public function requirement(
        mixed $value,
        ValidationContext $context,
    ): DatabaseValidationRequirement;

    public function evaluate(
        DatabaseValidationResult $result,
    ): ValidationResult;
}
```

---

# 96. Rule cannot execute arbitrary SQL

Una custom rule no recibirá:

```text
PDO
Driver
raw connection
```

por defecto.

---

# 97. Query extension path

Reglas avanzadas podrán utilizar APIs controladas de Query Engine.

---

# 98. Identifier security

APIs como:

```php
Rule::exists($table, $column)
```

deberán tratar:

```text
$table
$column
```

como identifiers, no values.

---

# 99. Identifiers cannot be parameter-bound

Por tanto deberán resolverse mediante:

```text
validated identifier
schema metadata
entity metadata
trusted configuration
```

---

# 100. User-controlled identifiers

Esto deberá rechazarse:

```php
Rule::exists(
    $_GET['table'],
    $_GET['column']
);
```

si no existe whitelist explícita.

---

# 101. Values

Los valores sí deberán utilizar:

```text
parameter binding
```

---

# 102. SQL injection protection

El Validation System heredará las garantías del Query Engine:

```text
Values
↓
Parameters
```

y:

```text
Identifiers
↓
Typed/Validated Identifier Model
```

---

# 103. Raw predicates

API como:

```php
->whereRaw(...)
```

no deberá ser parte de la superficie segura principal.

---

# 104. Escape hatch

Si existe deberá ser:

```text
explicit
unsafe/trusted
auditable
```

siguiendo `DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM`.

---

# 105. NULL semantics

Una regla Database deberá definir comportamiento para:

```text
NULL
```

explícitamente.

---

# 106. Example unique nullable

Algunos DBMS permiten múltiples NULL bajo ciertas restricciones de unicidad.

Validation no deberá inventar una semántica universal.

---

# 107. Platform-aware uniqueness

La regla podrá consultar:

```text
Database Capability System
```

cuando la semántica dependa de plataforma.

---

# 108. Case sensitivity

Ejemplo:

```text
User@example.com
user@example.com
```

pueden considerarse iguales o diferentes según:

- collation;
- type;
- index;
- expression;
- domain normalization.

---

# 109. Validation must align with constraint

Una regla de unicidad útil debe aproximarse a la semántica real del constraint.

---

# 110. Normalized domain values

Para email podría definirse previamente:

```text
Domain Normalization
↓
Validation
↓
Persistence
```

pero no será responsabilidad genérica de Database.

---

# 111. Collation awareness

No deberá compararse en PHP:

```php
strtolower($a) === strtolower($b)
```

y asumir que reproduce exactamente la collation del DBMS.

---

# 112. Query database for database semantics

Cuando la regla dependa de semántica de comparación del DBMS:

```text
DBMS
```

deberá ser la autoridad de comparación.

---

# 113. Composite constraints

Ejemplo:

```text
UNIQUE (
    tenant_id,
    external_id
)
```

Validation deberá poder representar ambos componentes.

---

# 114. Nulls in composite keys

Deberán respetarse las capacidades/semánticas reales de la plataforma.

---

# 115. Expression indexes

Un constraint podría ser:

```text
UNIQUE(lower(email))
```

La validación simple:

```text
email = ?
```

puede no reproducirlo.

---

# 116. Metadata integration

Cuando sea posible:

```text
Constraint Metadata
↓
Validation Mapping
```

podrá ayudar a mantener consistencia.

---

# 117. Constraint-aware rules

Futuro:

```php
Rule::constraint(User::class, 'users_email_unique');
```

podría derivar semántica desde metadata compilada.

---

# 118. Constraint-derived validation limitation

Incluso así:

```text
pre-check
≠
atomic guarantee
```

---

# 119. Validation result model

Se propone:

```text
DatabaseValidationResult
├── Status
├── Evidence
├── Source
├── Consistency
├── Scope
└── Diagnostics
```

---

# 120. Status

Posibles estados internos:

```text
SATISFIED
VIOLATED
UNKNOWN
ERROR
```

---

# 121. UNKNOWN

Deberá utilizarse cuando el sistema no pueda demostrar la regla.

Ejemplo:

```text
replica state unknown
shard routing unresolved
connection lost
```

---

# 122. UNKNOWN ≠ SATISFIED

Regla crítica:

```text
UNKNOWN
≠
VALID
```

---

# 123. Infrastructure failure

Si Database no puede ejecutar la comprobación:

```text
Connection failure
```

no deberá convertirse automáticamente en:

```text
"email already exists"
```

---

# 124. Validation infrastructure error

Debe poder distinguirse:

```text
User input invalid
```

de:

```text
Validation could not be completed
```

---

# 125. Failure policy

Podrán existir:

```text
FAIL_CLOSED
FAIL_OPEN
PROPAGATE
CUSTOM
```

pero reglas de integridad deberán favorecer seguridad.

---

# 126. Recommended database-rule default

Para reglas críticas:

```text
UNKNOWN
→
validation infrastructure failure
```

no:

```text
UNKNOWN
→
valid
```

---

# 127. Timeouts

Database-backed validation deberá tener:

```text
timeout budget
```

apropiado.

---

# 128. Timeout ≠ invalid input

Un timeout es fallo operacional.

---

# 129. Cancellation

Si el request se cancela:

```text
validation query
```

deberá poder cancelarse cuando Database soporte esa capacidad.

---

# 130. Validation query metadata

Las queries deberán identificarse como:

```text
purpose = VALIDATION
```

dentro de Query Context.

---

# 131. Why purpose metadata

Permite:

- telemetry;
- routing;
- profiling;
- diagnostics;
- slow-query analysis;
- resource governance.

---

# 132. Telemetry integration

Database Validation podrá emitir:

```text
validation.database.check
validation.database.duration
validation.database.batch_size
validation.database.failure
```

mediante la integración definida en el documento 316.

---

# 133. Sensitive values

Nunca deberán incluirse automáticamente en telemetry.

---

# 134. Query fingerprint

Sí podrá utilizarse para análisis.

---

# 135. Validation rule name

Dimensiones bounded como:

```text
exists
unique
```

son apropiadas.

---

# 136. Resource names

Table/entity names deberán someterse a política de cardinalidad antes de usarse como labels.

---

# 137. Cache question

¿Debe cachearse:

```text
exists()
```

o:

```text
unique()
```

?

Por defecto:

```text
NO
```

para reglas sensibles a escritura.

---

# 138. Why

Un cache hit puede estar obsoleto.

Especialmente peligroso:

```text
unique(email)
```

---

# 139. Cacheable validation

Reglas sobre datos:

```text
immutable
reference
versioned
```

podrían permitir cache bajo política explícita.

---

# 140. Cache ≠ Database truth

Incluso en Validation.

---

# 141. Request-local deduplication

Sí puede evitarse ejecutar exactamente la misma regla repetidamente dentro del mismo contexto cuando:

```text
inputs
context
transaction state
database generation
```

sean compatibles.

---

# 142. Transaction mutation invalidation

Si ocurre una escritura relevante dentro de la misma transacción, un resultado previo puede dejar de ser válido.

---

# 143. Therefore

La deduplicación deberá considerar:

```text
transaction mutation generation
```

o mecanismo equivalente.

---

# 144. Async validation

Validaciones Database críticas para persistencia no deberán delegarse a un proceso asíncrono que termine después de la escritura.

---

# 145. Async use cases

Sí puede utilizarse para:

```text
background diagnostics
non-blocking UX hints
preliminary availability checks
```

pero deberán etiquetarse como no autoritativas.

---

# 146. Client-side validation

Una comprobación AJAX:

```text
"email available"
```

es sólo una indicación UX.

---

# 147. Server-side revalidation

La operación de escritura deberá repetir las comprobaciones necesarias y depender finalmente del constraint.

---

# 148. Validation layering

```text
Client Validation
      ↓
Server Validation
      ↓
Database Constraint
```

No:

```text
Client Validation
=
Integrity Guarantee
```

---

# 149. Model API example

```php
$user = new User();

$user->email = $input['email'];

$errors = User::validator()->validate($user);

if ($errors->isEmpty()) {
    $user->save();
}
```

---

# 150. Race-safe application example

```php
try {
    $validator->validateOrFail($input);

    $user->save();
} catch (UniqueConstraintViolation $e) {
    throw $constraintMapper->toValidationException($e);
}
```

---

# 151. Repository example

```php
$validator->validate([
    'email' => $email,
], [
    'email' => [
        Rule::uniqueEntity(User::class, 'email'),
    ],
]);

$repository->add(
    new User($email)
);
```

---

# 152. Validation on flush

Si se habilita:

```text
ON_FLUSH
```

el ORM deberá recopilar entidades relevantes.

---

# 153. Flush validation planner

Conceptualmente:

```text
UnitOfWork
↓
Pending Changes
↓
Validation Requirements
↓
Batch Validation Planner
↓
Database Validation
↓
Continue / Reject Flush
```

---

# 154. Flush validation ordering

Debe ocurrir antes de emitir las operaciones que pretende validar.

---

# 155. But

No reemplaza:

```text
database constraints
```

durante ejecución real.

---

# 156. Entity state

Validation failure no deberá cambiar automáticamente:

```text
MANAGED
```

a:

```text
DETACHED
```

ni limpiar ChangeSets.

---

# 157. UoW after validation failure

La política deberá definir si el UoW sigue:

```text
dirty but usable
```

o si la operación de alto nivel aborta.

No deberá inventarse rollback de objetos.

---

# 158. Transaction rollback

Si validation failure causa rollback:

```text
DatabaseRollback
≠
ObjectGraphRewind
```

se mantiene.

---

# 159. Validation groups

Podrán existir grupos:

```text
CREATE
UPDATE
IMPORT
ADMIN
API
BACKGROUND
```

---

# 160. Database rules by group

Ejemplo:

```text
CREATE
→ unique email

UPDATE
→ unique email ignoring current entity
```

---

# 161. Validation context ≠ HTTP context

Validation deberá funcionar en:

```text
HTTP
CLI
Jobs
Tests
Migrations tooling where appropriate
```

---

# 162. Import validation

Para imports masivos:

```text
per-row query
```

sería costoso.

Debe utilizarse:

```text
batch validation
```

---

# 163. Import pipeline

```text
Read Chunk
↓
Pure Validation
↓
Collect Database Requirements
↓
Batch Database Validation
↓
Map Violations
↓
Bulk Persistence
```

---

# 164. Bulk persistence constraint failures

Aun después del batch validation, Database puede rechazar por concurrencia.

La aplicación deberá manejar el resultado.

---

# 165. Pagination not relevant to existence

No deberá utilizarse una colección completa para responder:

```text
exists?
```

---

# 166. Minimal query shape

La consulta deberá recuperar la mínima información necesaria.

Para `exists`:

```text
boolean/existence evidence
```

no:

```text
SELECT *
```

---

# 167. Unique count

Tampoco es necesario:

```text
COUNT(*)
```

si sólo importa:

```text
any matching row?
```

---

# 168. Query optimization

Preferir semántica de:

```text
EXISTS
```

o equivalente optimizado por plataforma/compiler.

---

# 169. Validation planner

Se propone:

```text
DatabaseValidationPlanner
```

responsable de:

- deduplicar;
- agrupar;
- elegir batch shape;
- preservar scopes;
- aplicar budgets.

---

# 170. Planner ≠ Query Planner

Es un planner de requirements de validación.

El Query Planner Database seguirá siendo responsable del plan de consulta.

---

# 171. Flow

```text
Validation Requirements
        ↓
DatabaseValidationPlanner
        ↓
Validation Query Models
        ↓
Database Query Planner
        ↓
Execution Plans
```

---

# 172. Resource governance

Una petición no deberá poder generar:

```text
100,000 validation queries
```

sin límites.

---

# 173. Budgets

Podrán existir:

```text
max database validation rules
max query count
max batch size
max duration
max rows examined hint
```

---

# 174. Budget exceeded

Debe producir un error explícito.

No:

```text
remaining rules considered valid
```

---

# 175. Backpressure

Imports/jobs deberán procesar validaciones en chunks bounded.

---

# 176. Connection pool

Validation no deberá mantener conexiones más tiempo del necesario.

---

# 177. Streaming

Batch validation de grandes datasets deberá evitar cargar estados ilimitados en memoria.

---

# 178. Persistent runtime

En FrankenPHP:

```text
Worker
├── Request A ValidationContext
├── Request B ValidationContext
└── Request C ValidationContext
```

cada uno deberá ser aislado.

---

# 179. No global mutable validation state

Prohibido:

```php
static $currentTenant;
static $ignoredId;
static $validationConnection;
```

---

# 180. Shared components

Podrán compartirse:

```text
Rule definitions
Validation metadata
Compiled rule metadata
DatabaseValidationGateway implementation
```

si son stateless/inmutables.

---

# 181. Scoped components

Deberán ser scoped:

```text
DatabaseValidationContext
Batch accumulator
Validation result cache
Transaction-aware state
```

---

# 182. FrankenPHP reset

Al finalizar request:

```text
validation context
batch state
temporary results
transaction references
tenant references
```

deberán liberarse.

---

# 183. RoadRunner

Aplicará las mismas reglas.

---

# 184. OpenSwoole

El contexto deberá ser coroutine-safe.

---

# 185. Concurrent validation

```text
Coroutine A
Tenant A
```

y:

```text
Coroutine B
Tenant B
```

nunca deberán compartir contexto.

---

# 186. Validation configuration

Ejemplo:

```php
'database' => [
    'validation' => [
        'enabled' => true,

        'default_consistency' => 'write_consistent',

        'unique' => [
            'prefer_writer' => true,
        ],

        'batching' => [
            'enabled' => true,
            'max_size' => 500,
        ],

        'constraint_mapping' => true,
    ],
],
```

---

# 187. Configuration safety

Config no deberá permitir silenciosamente:

```text
unique checks against stale replica
```

cuando la regla requiera write consistency.

---

# 188. Extension architecture

Plugins podrán registrar:

```text
custom database validation rules
constraint mappers
validation requirement compilers
context contributors
```

mediante registries controlados.

---

# 189. Registry freeze

Los registries deberán congelarse tras bootstrap.

---

# 190. Plugin restrictions

Un plugin de Validation no deberá:

- reemplazar Transaction semantics;
- acceder arbitrariamente a PDO;
- modificar tenant context;
- saltarse authorization;
- declarar UNKNOWN como VALID globalmente sin policy.

---

# 191. Proposed namespace

```text
VoltStack\Quantum\Database\Integration\Validation
```

---

# 192. Proposed directory structure

```text
src/Quantum/Database/Integration/Validation/
├── Contract/
│   ├── DatabaseValidationGateway.php
│   ├── DatabaseValidationRule.php
│   └── ConstraintViolationMapper.php
│
├── Context/
│   ├── DatabaseValidationContext.php
│   ├── DatabaseValidationContextResolver.php
│   ├── ValidationConsistency.php
│   └── ValidationConnectionIntent.php
│
├── Requirement/
│   ├── DatabaseValidationRequirement.php
│   ├── DatabaseExistenceRequirement.php
│   ├── DatabaseUniquenessRequirement.php
│   └── DatabaseCompositeRequirement.php
│
├── Result/
│   ├── DatabaseValidationResult.php
│   ├── DatabaseExistenceResult.php
│   ├── DatabaseUniquenessResult.php
│   └── DatabaseValidationStatus.php
│
├── Rule/
│   ├── ExistsRule.php
│   ├── UniqueRule.php
│   ├── EntityExistsRule.php
│   └── EntityUniqueRule.php
│
├── Planner/
│   ├── DatabaseValidationPlanner.php
│   ├── ValidationBatchPlanner.php
│   └── ValidationRequirementGroup.php
│
├── Query/
│   ├── ValidationQueryFactory.php
│   ├── ExistsQueryFactory.php
│   └── UniqueQueryFactory.php
│
├── ORM/
│   ├── EntityValidationBridge.php
│   ├── ModelValidationBridge.php
│   ├── FlushValidationCoordinator.php
│   └── EntityIdentityResolver.php
│
├── Constraint/
│   ├── DatabaseConstraintViolationMapper.php
│   ├── UniqueConstraintValidationMapper.php
│   └── ForeignKeyValidationMapper.php
│
├── Tenant/
│   └── TenantValidationContextContributor.php
│
├── Sharding/
│   └── ShardValidationContextContributor.php
│
├── Security/
│   ├── ValidationIdentifierPolicy.php
│   └── DatabaseValidationSecurityPolicy.php
│
├── Telemetry/
│   └── DatabaseValidationTelemetry.php
│
├── Extension/
│   ├── DatabaseValidationExtension.php
│   └── DatabaseValidationRuleRegistry.php
│
└── Exception/
    ├── DatabaseValidationException.php
    ├── DatabaseValidationInfrastructureException.php
    ├── DatabaseValidationUnknownException.php
    ├── UnsafeValidationIdentifierException.php
    └── ValidationBudgetExceededException.php
```

---

# 193. Public API

Ejemplo:

```php
use VoltStack\Validation\Rule;

$rules = [
    'email' => [
        'required',
        'email',
        Rule::unique('users', 'email'),
    ],

    'role_id' => [
        'required',
        Rule::exists('roles', 'id'),
    ],
];
```

---

# 194. Entity API

```php
$rules = [
    'email' => [
        Rule::uniqueEntity(User::class, 'email')
            ->ignoreEntity($user),
    ],
];
```

---

# 195. Scoped API

```php
Rule::uniqueEntity(User::class, 'slug')
    ->where('organization_id', $organizationId);
```

---

# 196. Soft-delete API

```php
Rule::uniqueEntity(User::class, 'email')
    ->withoutSoftDeleted();
```

---

# 197. Connection API

Sólo para casos explícitos:

```php
Rule::exists('external_records', 'id')
    ->connection('legacy');
```

---

# 198. Connection names

Deberán ser:

```text
trusted configuration identifiers
```

no valores arbitrarios del usuario.

---

# 199. Composite API

Podrá existir:

```php
Rule::uniqueComposite(User::class, [
    'tenant_id' => $tenantId,
    'email' => $email,
]);
```

---

# 200. Constraint API future

```php
Rule::databaseConstraint(
    User::class,
    'users_tenant_email_unique',
);
```

---

# 201. Testing architecture

Deberá dividirse en:

```text
Unit
Integration
Concurrency
Platform Conformance
Security
Persistent Runtime
Performance
```

---

# 202. Unit testing

Validará:

- requirement construction;
- batching;
- grouping;
- context resolution;
- identifier validation;
- constraint mapping;
- result mapping.

Sin DBMS cuando la propiedad sea pura.

---

# 203. Integration testing

Deberá utilizar DBMS reales para:

```text
exists
unique
collations
NULL behavior
constraints
transactions
visibility
```

---

# 204. Platform matrix

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

por separado.

---

# 205. SQLite warning

```text
SQLite
≠
universal validation substitute
```

especialmente para:

- collations;
- concurrent writes;
- isolation;
- unique NULL semantics;
- locking;
- platform-specific constraints.

---

# 206. Unique race test

Escenario:

```text
T1 validate X → unique
T2 validate X → unique
T1 INSERT X → success
T2 INSERT X → constraint violation
```

La prueba deberá demostrar que:

```text
Validation cannot guarantee uniqueness
```

y:

```text
Constraint does.
```

---

# 207. Replica lag test

Escenario:

```text
Writer contains X
Replica lacks X
```

Una regla `unique` write-consistent no deberá confiar en la réplica.

---

# 208. Tenant test

Tenant A:

```text
email@example.com
```

Tenant B:

```text
email@example.com
```

deberá producir el resultado correspondiente a la política de aislamiento definida.

---

# 209. Cross-tenant leak test

Una validación en Tenant A nunca deberá observar registros de Tenant B accidentalmente.

---

# 210. Shard test

Una regla con shard key deberá consultar únicamente el shard correcto cuando sea demostrable.

---

# 211. Unknown shard test

Si no puede determinarse el shard:

```text
UNKNOWN
```

no deberá tratarse como:

```text
no matching row
```

---

# 212. Soft delete test

Deberán probarse ambos modos:

```text
including deleted
excluding deleted
```

---

# 213. Batch test

Validar que:

```text
100 compatible exists rules
```

puedan reducirse a un número bounded de queries.

---

# 214. Batch semantic test

La optimización nunca deberá cambiar qué inputs resultan válidos.

---

# 215. Security test

Intentar identifiers como:

```text
users; DROP TABLE users
```

deberá fallar antes de llegar al compiler.

---

# 216. Parameter injection test

Valor:

```text
' OR 1=1 --
```

deberá permanecer:

```text
bound value
```

---

# 217. Constraint mapping test

Forzar:

```text
UNIQUE violation
```

y verificar mapping correcto al campo esperado.

---

# 218. Unknown constraint test

Una constraint no mapeada deberá conservar:

```text
DatabaseConstraintViolation
```

sin inventar un campo.

---

# 219. Persistent runtime test

Ejecutar:

```text
Request A tenant A
Request B tenant B
```

en el mismo worker y demostrar aislamiento.

---

# 220. Performance test

Comparar:

```text
N individual checks
```

contra:

```text
batched checks
```

para justificar batching.

---

# 221. Telemetry test

Verificar:

```text
rule type
duration
batch size
outcome
```

sin exponer valores sensibles.

---

# 222. Core invariants

## DB-VAL-INT-001

Validation ≠ Persistence.

## DB-VAL-INT-002

Validation ≠ Database Constraint.

## DB-VAL-INT-003

Validation Success ≠ Persistence Success.

## DB-VAL-INT-004

Unique Validation ≠ UNIQUE Constraint.

## DB-VAL-INT-005

Exists Validation ≠ Foreign Key.

## DB-VAL-INT-006

Validation ≠ Authorization.

## DB-VAL-INT-007

Validation ≠ Sanitization.

## DB-VAL-INT-008

Validation ≠ Concurrency Control.

## DB-VAL-INT-009

Validation no generará SQL directamente.

## DB-VAL-INT-010

Validation no accederá directamente al Driver.

---

# 223. Query invariants

## DB-VAL-INT-011

Database-backed rules utilizarán Query Engine.

## DB-VAL-INT-012

Values serán parameterized.

## DB-VAL-INT-013

Identifiers serán typed/trusted/validated.

## DB-VAL-INT-014

User input no elegirá identifiers arbitrarios.

## DB-VAL-INT-015

Existence queries recuperarán información mínima.

## DB-VAL-INT-016

Validation query tendrá purpose metadata.

## DB-VAL-INT-017

Validation query podrá tener timeout.

## DB-VAL-INT-018

Timeout ≠ invalid input.

## DB-VAL-INT-019

Cancellation ≠ invalid input.

## DB-VAL-INT-020

UNKNOWN ≠ VALID.

---

# 224. Uniqueness invariants

## DB-VAL-INT-021

Pre-check de unicidad no será considerado garantía atómica.

## DB-VAL-INT-022

Invariantes persistentes de unicidad deberán respaldarse con constraints cuando sea posible.

## DB-VAL-INT-023

Unique check para escritura preferirá consistencia con writer.

## DB-VAL-INT-024

Replica stale no deberá demostrar unicidad.

## DB-VAL-INT-025

Ignore current entity utilizará identidad validada.

## DB-VAL-INT-026

Composite uniqueness preservará todos sus componentes.

## DB-VAL-INT-027

NULL semantics serán platform-aware.

## DB-VAL-INT-028

Collation semantics no se inventarán en PHP.

## DB-VAL-INT-029

Expression uniqueness no se reducirá incorrectamente a simple equality.

## DB-VAL-INT-030

Constraint violation posterior podrá mapearse a Validation.

---

# 225. Transaction invariants

## DB-VAL-INT-031

Transaction-aware validation utilizará el contexto correcto.

## DB-VAL-INT-032

Validation dentro de transaction no elimina TOCTOU universalmente.

## DB-VAL-INT-033

Validation no adquirirá locks implícitos por defecto.

## DB-VAL-INT-034

Locking será política explícita.

## DB-VAL-INT-035

Rollback ≠ ObjectGraphRewind.

## DB-VAL-INT-036

Validation failure no cambiará EntityState arbitrariamente.

## DB-VAL-INT-037

Validation no hará commit.

## DB-VAL-INT-038

Validation no hará rollback salvo coordinación explícita de capa superior.

## DB-VAL-INT-039

Validation no será propietaria de TransactionManager.

## DB-VAL-INT-040

Unknown transaction state se preservará.

---

# 226. ORM invariants

## DB-VAL-INT-041

Entity metadata ≠ Validation metadata.

## DB-VAL-INT-042

Schema metadata ≠ Domain validation.

## DB-VAL-INT-043

Model API será facade de conveniencia.

## DB-VAL-INT-044

Repository no será reemplazado por Validation.

## DB-VAL-INT-045

EntityManager no será Validator.

## DB-VAL-INT-046

persist() no ejecutará validación Database implícita salvo policy.

## DB-VAL-INT-047

flush() no ejecutará validación implícita salvo policy.

## DB-VAL-INT-048

ON_FLUSH validation será explícitamente configurable.

## DB-VAL-INT-049

Validation no modificará ChangeSets silenciosamente.

## DB-VAL-INT-050

Validation no modificará IdentityMap.

---

# 227. Multitenancy invariants

## DB-VAL-INT-051

Tenant context será scoped.

## DB-VAL-INT-052

Tenant identity no será tomada de input no confiable.

## DB-VAL-INT-053

Validation no cruzará tenants accidentalmente.

## DB-VAL-INT-054

Database-per-tenant utilizará tenant connection resolution.

## DB-VAL-INT-055

Schema-per-tenant utilizará schema context.

## DB-VAL-INT-056

Shared-table tenancy aplicará scope apropiado.

## DB-VAL-INT-057

Cross-tenant validation será explícita.

## DB-VAL-INT-058

Cross-tenant validation requerirá autorización externa.

## DB-VAL-INT-059

Multitenancy seguirá siendo dependencia opcional.

## DB-VAL-INT-060

Tenant context no será global mutable.

---

# 228. Sharding invariants

## DB-VAL-INT-061

Shard routing precederá query execution.

## DB-VAL-INT-062

Unknown shard ≠ no match.

## DB-VAL-INT-063

Single-shard uniqueness ≠ global uniqueness.

## DB-VAL-INT-064

Global uniqueness requerirá arquitectura capaz de garantizarla.

## DB-VAL-INT-065

Validation no hará scatter global silencioso.

## DB-VAL-INT-066

Scatter validation requerirá policy explícita.

## DB-VAL-INT-067

Shard context será parte de compatibility para batching.

## DB-VAL-INT-068

Validation no inventará distributed transactions.

## DB-VAL-INT-069

Shard map generation podrá formar parte del contexto.

## DB-VAL-INT-070

Stale routing evidence no será tratado como certeza.

---

# 229. Batch invariants

## DB-VAL-INT-071

Compatible rules podrán agruparse.

## DB-VAL-INT-072

Incompatible scopes no se agruparán.

## DB-VAL-INT-073

Batching preservará resultados individuales.

## DB-VAL-INT-074

Batching no cambiará consistency requirements.

## DB-VAL-INT-075

Batching no cruzará tenants.

## DB-VAL-INT-076

Batching no cruzará shards incompatibles.

## DB-VAL-INT-077

Batching respetará transaction context.

## DB-VAL-INT-078

Batch size será bounded.

## DB-VAL-INT-079

Input duplicates se distinguirán de persisted duplicates.

## DB-VAL-INT-080

Batch optimization será semánticamente transparente.

---

# 230. Runtime invariants

## DB-VAL-INT-081

ValidationContext será request/operation scoped.

## DB-VAL-INT-082

No habrá current validation context global mutable.

## DB-VAL-INT-083

FrankenPHP tendrá isolation entre requests.

## DB-VAL-INT-084

RoadRunner seguirá el mismo modelo.

## DB-VAL-INT-085

OpenSwoole tendrá coroutine isolation.

## DB-VAL-INT-086

Batch state se reseteará.

## DB-VAL-INT-087

Temporary result cache se reseteará.

## DB-VAL-INT-088

Transaction references no sobrevivirán al scope.

## DB-VAL-INT-089

Tenant references no sobrevivirán al scope.

## DB-VAL-INT-090

Reset failure será visible.

---

# 231. Security invariants

## DB-VAL-INT-091

Validation nunca concatenará values en SQL.

## DB-VAL-INT-092

Raw SQL no será API principal.

## DB-VAL-INT-093

Unsafe escape hatches serán explícitos.

## DB-VAL-INT-094

Connection names serán trusted identifiers.

## DB-VAL-INT-095

Constraint names no se confiarán desde user input.

## DB-VAL-INT-096

Telemetry no expondrá validation values sensibles.

## DB-VAL-INT-097

Error messages del DBMS serán sanitizados.

## DB-VAL-INT-098

Validation no otorgará authorization.

## DB-VAL-INT-099

Validation no modificará security context.

## DB-VAL-INT-100

Unknown sensitivity se omitirá/redactará.

---

# 232. Operational invariants

## DB-VAL-INT-101

Database validation tendrá resource budgets.

## DB-VAL-INT-102

Budget exceeded ≠ valid.

## DB-VAL-INT-103

Infrastructure failure ≠ validation violation.

## DB-VAL-INT-104

Infrastructure failure ≠ valid.

## DB-VAL-INT-105

Cache stale no demostrará validation success.

## DB-VAL-INT-106

Critical uniqueness checks no usarán cache stale.

## DB-VAL-INT-107

Async hint ≠ authoritative validation.

## DB-VAL-INT-108

Client validation ≠ server validation.

## DB-VAL-INT-109

Server validation ≠ database guarantee.

## DB-VAL-INT-110

Database constraint seguirá siendo última barrera de integridad.

---

# 233. Anti-pattern: unique without constraint

Incorrecto:

```php
if (!User::where('email', $email)->exists()) {
    User::create(['email' => $email]);
}
```

si se considera suficiente para garantizar unicidad.

Correcto:

```text
UX validation
+
UNIQUE constraint
+
constraint violation handling
```

---

# 234. Anti-pattern: exists replaces foreign key

Incorrecto:

```text
Validation says role exists
→ therefore referential integrity guaranteed
```

Debe existir FK cuando el modelo lo requiera y la plataforma pueda expresarlo.

---

# 235. Anti-pattern: stale replica uniqueness

```text
Replica
↓
email not found
↓
"email available"
```

mientras el writer ya contiene el email.

---

# 236. Anti-pattern: SQL in validator

Incorrecto:

```php
$pdo->query(
    "SELECT * FROM users WHERE email = '$email'"
);
```

---

# 237. Anti-pattern: dynamic table from request

Incorrecto:

```php
Rule::exists(
    $request->table,
    $request->column
);
```

---

# 238. Anti-pattern: validation as authorization

Incorrecto:

```text
record exists
⇒
user may access record
```

---

# 239. Anti-pattern: validation as sanitization

Un valor válido no significa que haya sido:

```text
escaped
encoded
sanitized
```

para todos los contextos.

---

# 240. Anti-pattern: global validation connection

```php
static $connection;
```

es inseguro para persistent runtimes.

---

# 241. Anti-pattern: one query per imported row

Puede producir:

```text
Validation N+1
```

---

# 242. Anti-pattern: cache unique forever

Un resultado:

```text
email available
```

puede quedar obsoleto inmediatamente.

---

# 243. Anti-pattern: count all rows

Incorrecto para existencia:

```sql
SELECT COUNT(*)
```

cuando sólo se necesita demostrar:

```text
at least one
```

---

# 244. Anti-pattern: catch every DB exception as validation error

No toda excepción Database representa input inválido.

---

# 245. Anti-pattern: parse vendor error strings blindly

Debe preferirse clasificación estructurada.

---

# 246. Anti-pattern: global uniqueness on one shard

Nunca deberá declararse global una comprobación local.

---

# 247. Anti-pattern: automatic lock

No convertir cada `unique()` en:

```text
SELECT ... FOR UPDATE
```

---

# 248. Anti-pattern: schema metadata is domain model

El hecho de que una columna permita 255 caracteres no significa que el dominio deba permitirlos.

---

# 249. Formal model

Sea:

```text
V = Validation Rule
D(t) = Database state at time t
```

una validación Database produce:

```text
R = V(D(t1))
```

La persistencia ocurre en:

```text
t2 ≥ t1
```

Por tanto:

```text
R(t1)
```

no implica necesariamente:

```text
ConstraintSatisfied(t2)
```

si:

```text
D(t1) ≠ D(t2)
```

---

# 250. Uniqueness theorem

Para una comprobación:

```text
UniqueCheck(X, t1) = true
```

no puede inferirse:

```text
Insert(X, t2) will succeed
```

bajo concurrencia.

---

# 251. Constraint authority

En cambio, si existe:

```text
UNIQUE(X)
```

el DBMS serializa/aplica la invariancia conforme a sus semánticas.

Por tanto:

```text
Validation
→ UX / early feedback

Constraint
→ integrity guarantee
```

---

# 252. Existence theorem

Similarmente:

```text
Exists(FK, t1)
```

no garantiza que el registro siga existiendo en:

```text
t2
```

sin garantías adicionales.

---

# 253. Database-backed validation validity

Un resultado será confiable sólo respecto a:

```text
Database state observed
+
Consistency level
+
Transaction context
+
Tenant context
+
Shard context
+
Scope predicates
+
Time of observation
```

---

# 254. Context equation

Conceptualmente:

```text
ValidationEvidence =
f(
    Requirement,
    DatabaseState,
    Connection,
    Transaction,
    Tenant,
    Shard,
    Consistency,
    Policy
)
```

---

# 255. Batch compatibility equation

Dos requirements:

```text
A
B
```

podrán agruparse sólo si:

```text
Resource(A) = Resource(B)
∧
Context(A) ≈ Context(B)
∧
Consistency(A) = Consistency(B)
∧
Transaction(A) = Transaction(B)
∧
Tenant(A) = Tenant(B)
∧
Shard(A) = Shard(B)
```

más cualquier otra condición semántica necesaria.

---

# 256. Architecture result

La integración final será:

```text
                     VoltStack Validation
                              │
             ┌────────────────┴────────────────┐
             │                                 │
       Pure Validation                 Database Validation
             │                                 │
             │                                 ▼
             │                     DatabaseValidationBridge
             │                                 │
             │                                 ▼
             │                         Validation Planner
             │                                 │
             │                                 ▼
             │                           Query Engine
             │                                 │
             │                                 ▼
             │                        Execution Engine
             │                                 │
             │                                 ▼
             │                         Connection System
             │                                 │
             │                                 ▼
             │                              Driver
             │                                 │
             └─────────────────────────────────┤
                                               ▼
                                              DBMS
                                               │
                                      ┌────────┴─────────┐
                                      │                  │
                                Validation Query    Constraints
                                      │                  │
                                Early Feedback     Final Integrity
```

---

# 257. Integration with persistence

```text
Input
 ↓
Validation
 ↓
Application
 ↓
ORM / Query Engine
 ↓
Persistence
 ↓
Database Constraint
 ↓
Success
   or
Constraint Violation
 ↓
Constraint Mapper
 ↓
Application / Validation Error
```

Esto permite buena experiencia de desarrollador sin sacrificar integridad.

---

# 258. V1 scope

La primera versión deberá incluir:

```text
DatabaseValidationGateway
DatabaseValidationContext
Rule::exists()
Rule::unique()
ignore()
entity-aware rules
parameter-safe queries
identifier validation
writer-aware uniqueness checks
transaction context integration
tenant context extension point
soft-delete policies
batch exists
batch unique
constraint violation mapping
telemetry
resource budgets
FrankenPHP isolation
unit tests
real DB integration tests
```

---

# 259. V2

Agregar:

```text
constraint-derived validation metadata
advanced composite validation
automatic batch planning
shard-aware validation
global uniqueness strategies
advanced import validation
validation query deduplication
compiled validation plans
```

---

# 260. V3

Agregar:

```text
distributed validation coordination
validation performance optimizer
adaptive batching
schema-aware validation generation
advanced consistency policies
validation diagnostics tooling
```

sin convertir Validation en sistema de integridad distribuida ficticio.

---

# 261. Developer experience principle

La API deberá permitir:

```php
Rule::uniqueEntity(User::class, 'email')
```

sin obligar al desarrollador a comprender:

```text
AST
Compiler
Connection routing
Driver
```

para casos normales.

Pero la simplicidad de la API nunca deberá ocultar que:

```text
unique()
```

es un:

```text
pre-check
```

y no la garantía definitiva.

---

# 262. Principio definitivo

> **VoltStack Validation deberá detectar problemas lo antes posible y ofrecer errores útiles al desarrollador y al usuario; VoltStack Database deberá seguir siendo la autoridad final sobre la integridad persistente.**

En forma resumida:

```text
Validation checks.
Database constrains.
Application coordinates.
```

---

# 263. Regla de concurrencia definitiva

```text
Validation at t1
+
Concurrent system
≠
Guarantee at t2
```

Por tanto:

```text
Validation
+
Database Constraints
+
Constraint Error Mapping
```

forman el modelo correcto.

---

# 264. Regla de seguridad definitiva

```text
Database-backed Validation
```

deberá utilizar:

```text
Typed Identifiers
+
Bound Parameters
+
Scoped Context
+
Query Engine
```

y nunca:

```text
Raw User-Controlled SQL
```

---

# 265. Regla de distribución definitiva

```text
Local Evidence
≠
Global Guarantee
```

Esto será especialmente importante para:

```text
replicas
shards
multitenancy
distributed databases
```

---

# 266. Regla de runtime definitiva

```text
Shared Immutable Validation Infrastructure
+
Scoped Mutable Validation Context
=
Persistent Runtime Safety
```

---

# 267. Resultado final

Con esta arquitectura, VoltStack podrá ofrecer ergonomía semejante a:

```php
Rule::exists(...)
Rule::unique(...)
Rule::uniqueEntity(...)
```

manteniendo debajo una arquitectura coherente con:

```text
Query Engine
Connection Routing
Transactions
ORM
Multitenancy
Sharding
Security
Telemetry
Persistent Runtimes
```

y preservando las reglas esenciales:

```text
Validation
≠
Constraint
```

```text
Validation Success
≠
Persistence Success
```

```text
Unique Check
≠
Atomic Uniqueness
```

```text
Exists Check
≠
Referential Integrity
```

```text
UNKNOWN
≠
VALID
```

---

# 268. Siguiente documento

```text
318_DATABASE_AUTHENTICATION_INTEGRATION_SYSTEM.md
```

Definirá la integración entre:

```text
VoltStack/Quantum/Database
        ↕
VoltStack Authentication
```

incluyendo:

```text
identity persistence
user providers
credential persistence boundaries
session persistence
remember-token persistence
authentication queries
guard/firewall integration
password hash storage
MFA persistence
WebAuthn credential persistence
OAuth2/OIDC persistence
device trust
token persistence
authentication read/write consistency
authentication transactions
account state
identity lookup
tenant-aware authentication
security-sensitive routing
credential redaction
authentication audit integration
cache boundaries
replica safety
persistent runtime isolation
telemetry
testing
```

manteniendo como reglas centrales:

```text
Database
≠
Authentication
```

```text
User Record
≠
Authenticated Identity
```

```text
Credential Exists
≠
Credential Valid
```

y:

```text
Successful Database Lookup
≠
Successful Authentication
```