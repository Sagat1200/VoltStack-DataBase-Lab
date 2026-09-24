# 234_DATABASE_DATABASE_PERMISSION_MODEL.md

# VoltStack Quantum Database
## Database Permission Model

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 234 — Database Permission Model  
**Bloque:** 22 — Security  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `233_DATABASE_QUERY_AUDIT_SYSTEM.md`  
**Siguiente documento:** `235_DATABASE_RESILIENCE_ARCHITECTURE.md`

---

# 1. Propósito

Este documento define el modelo de permisos semánticos del subsistema Database de VoltStack.

El sistema deberá determinar si un principal puede realizar una operación concreta sobre un recurso Database bajo un contexto determinado antes de alcanzar un punto irreversible de ejecución.

La regla central será:

> **Ninguna operación protegida de VoltStack Database obtiene autoridad simplemente porque el código pueda alcanzar una API capaz de ejecutarla; la autoridad deberá derivarse explícitamente de un principal, una acción, un recurso, un scope y un contexto de seguridad verificable.**

Formalmente:

```text
PermissionDecision
=
Evaluate(
    Principal,
    Action,
    Resource,
    Scope,
    Context
)
```

y nunca:

```text
CanCallMethod
=
Authorized
```

---

# 2. Problema arquitectónico

Una aplicación puede exponer:

```php
User::query()->delete();

DB::table('users')->update(...);

$entityManager->remove($entity);

$schema->dropTable('users');

$migrations->run();

$exporter->export(...);
```

La existencia de estas APIs no debe significar:

```text
caller has permission
```

Database necesita conocer cuándo una operación está:

```text
AUTHORIZED
DENIED
NOT_APPLICABLE
UNKNOWN
```

antes de ejecutarla.

---

# 3. Permission Model ≠ Authentication

Authentication responde:

```text
¿Quién es?
```

Permission Model responde:

```text
¿Qué puede hacer
sobre qué recurso
bajo qué contexto?
```

Por tanto:

```text
Authentication
≠
Permission Evaluation
```

---

# 4. Permission Model ≠ Application Authorization

VoltStack ya dispone de una arquitectura general de Authorization.

Database Permission Model no deberá convertirse en otro Authorization Framework.

La relación será:

```text
VoltStack Authorization
        │
        ▼
Database Authorization Bridge
        │
        ▼
Database Permission Model
        │
        ▼
Database Security Enforcement
```

El sistema Database define:

```text
acciones
recursos
scopes
contexto
restricciones
puntos de enforcement
```

mientras Authorization puede proporcionar:

```text
policies
roles
permissions
decisions
principals
delegation
```

---

# 5. Database Permission Model ≠ Database Engine GRANT

Los motores SQL disponen de mecanismos como:

```sql
GRANT SELECT
GRANT INSERT
GRANT UPDATE
GRANT DELETE
```

Estos permisos siguen siendo importantes.

Pero:

```text
VoltStack Database Permission
≠
Database Native Permission
```

---

# 6. Defensa en profundidad

VoltStack utilizará conceptualmente:

```text
Application Authorization
        +
Database Permission Model
        +
Database Native Credentials/Privileges
        =
Defense in Depth
```

---

# 7. Native database permissions

Las credenciales físicas deberán seguir el principio de:

```text
Least Privilege
```

Por ejemplo:

```text
Application Runtime Credential
    SELECT
    INSERT
    UPDATE
    DELETE

Migration Credential
    ALTER
    CREATE
    DROP

Backup Credential
    backup-specific privileges

Audit Writer Credential
    append-oriented audit privileges
```

---

# 8. No privilege substitution

El hecho de que PostgreSQL/MySQL permita una operación no significa que VoltStack deba autorizarla.

Y viceversa:

```text
VoltStack ALLOW
+
Database DENY
=
operation fails
```

VoltStack nunca deberá intentar evadir los permisos físicos del motor.

---

# 9. Objetivos

El modelo deberá soportar:

1. principals;
2. semantic actions;
3. semantic resources;
4. hierarchical scopes;
5. explicit allow;
6. explicit deny;
7. default deny;
8. field-level restrictions;
9. row/data scopes;
10. tenant scopes;
11. database scopes;
12. schema/table scopes;
13. entity scopes;
14. query scopes;
15. administrative permissions;
16. sensitive-data permissions;
17. raw SQL permissions;
18. migration permissions;
19. import/export permissions;
20. backup/restore permissions;
21. audit permissions;
22. delegation;
23. impersonation;
24. temporary authority;
25. cross-tenant authority;
26. shard-aware authority;
27. permission caching;
28. auditing;
29. telemetry;
30. persistent-runtime isolation.

---

# 10. Arquitectura general

```text
Execution Request
       │
       ▼
Database Security Boundary
       │
       ▼
Permission Request
       │
       ├── Principal
       ├── Action
       ├── Resource
       ├── Scope
       └── Context
       │
       ▼
Permission Resolver
       │
       ▼
Permission Evaluator
       │
       ▼
Permission Decision
       │
       ├── ALLOW
       ├── DENY
       └── UNKNOWN
       │
       ├──────── DENY ───────► Reject
       │
       └──────── ALLOW
                 │
                 ▼
          Security Constraints
                 │
                 ▼
             Query / ORM
                 │
                 ▼
          Execution Engine
```

---

# 11. Cinco dimensiones fundamentales

Toda decisión deberá poder reducirse conceptualmente a:

```text
WHO
WHAT
RESOURCE
SCOPE
CONTEXT
```

Es decir:

```text
Principal
Action
Resource
Scope
Context
```

---

# 12. DatabasePrincipal

```php
interface DatabasePrincipal
{
    public function id(): DatabasePrincipalId;

    public function type(): DatabasePrincipalType;
}
```

---

# 13. Principal types

```php
enum DatabasePrincipalType
{
    case USER;
    case SERVICE;
    case API_CLIENT;
    case JOB;
    case WORKER;
    case SYSTEM;
    case ADMINISTRATOR;
    case DELEGATED;
    case ANONYMOUS;
    case UNKNOWN;
}
```

---

# 14. Principal ≠ User

Un proceso background puede actuar como:

```text
JobPrincipal
```

Un servicio interno:

```text
ServicePrincipal
```

Una migración:

```text
MigrationPrincipal
```

---

# 15. UNKNOWN principal

`UNKNOWN` deberá representarse explícitamente.

Nunca:

```text
UNKNOWN
→ SYSTEM
```

automáticamente.

---

# 16. Anonymous principal

`ANONYMOUS` significa que se conoce que la operación pertenece a un contexto anónimo.

`UNKNOWN` significa que no existe evidencia suficiente sobre la identidad.

Por tanto:

```text
ANONYMOUS
≠
UNKNOWN
```

---

# 17. DatabaseAction

Una acción será una capacidad semántica.

```php
enum DatabaseAction
{
    case READ;
    case CREATE;
    case UPDATE;
    case DELETE;

    case BULK_INSERT;
    case BULK_UPDATE;
    case BULK_DELETE;

    case EXECUTE_RAW_QUERY;

    case IMPORT;
    case EXPORT;

    case CREATE_SCHEMA;
    case ALTER_SCHEMA;
    case DROP_SCHEMA;

    case RUN_MIGRATION;
    case ROLLBACK_MIGRATION;

    case BACKUP;
    case RESTORE;

    case READ_AUDIT;
    case EXPORT_AUDIT;
    case MANAGE_AUDIT;

    case READ_SENSITIVE;
    case WRITE_SENSITIVE;
    case REVEAL_SENSITIVE;

    case CROSS_TENANT_ACCESS;

    case ADMINISTER_DATABASE;
}
```

---

# 18. SQL verb ≠ DatabaseAction

```text
SELECT
```

puede representar:

```text
READ
READ_SENSITIVE
EXPORT
READ_AUDIT
```

Por tanto:

```text
SQL Verb
≠
Permission Action
```

---

# 19. Composite actions

Podrán definirse capacidades de mayor nivel:

```text
READ_WRITE
CRUD
DATA_ADMIN
SCHEMA_ADMIN
AUDIT_ADMIN
DATABASE_ADMIN
```

pero deberán expandirse a acciones concretas.

---

# 20. No magic administrator

Evitar:

```text
isAdmin()
→ allow everything
```

como mecanismo interno principal.

Incluso administradores deberán recibir authority explícita.

---

# 21. DatabaseResource

Representa el objetivo semántico de la operación.

```php
interface DatabaseResource
{
    public function type(): DatabaseResourceType;

    public function identity(): DatabaseResourceIdentity;
}
```

---

# 22. Resource types

```php
enum DatabaseResourceType
{
    case DATABASE;
    case SCHEMA;
    case TABLE;
    case COLUMN;
    case ENTITY;
    case ENTITY_FIELD;
    case ENTITY_INSTANCE;
    case RELATIONSHIP;
    case QUERY;
    case MIGRATION;
    case IMPORT;
    case EXPORT;
    case BACKUP;
    case AUDIT_LOG;
    case ADMIN_OPERATION;
    case CUSTOM;
}
```

---

# 23. Resource hierarchy

Ejemplo:

```text
Database
└── Schema
    └── Table
        └── Column
```

ORM:

```text
Persistence Domain
└── Entity Type
    └── Entity Instance
        └── Entity Field
```

---

# 24. Resource hierarchy ≠ automatic inheritance

Que exista:

```text
Database → Table → Column
```

no significa automáticamente:

```text
ALLOW Database
→ ALLOW every column
```

La inheritance deberá ser policy-driven.

---

# 25. Resource identity

Ejemplo:

```text
EntityResource(
    type = Customer
)
```

o:

```text
EntityInstanceResource(
    type = Customer,
    id = 42
)
```

---

# 26. Sensitive resource identities

Los identificadores utilizados durante evaluación no deberán filtrarse innecesariamente hacia:

```text
logs
telemetry
exceptions
debug output
```

---

# 27. Permission

Una permission será una regla declarativa.

Conceptualmente:

```text
Permission
=
Effect
+
Action
+
ResourceSelector
+
Scope
+
Conditions
```

---

# 28. PermissionEffect

```php
enum PermissionEffect
{
    case ALLOW;
    case DENY;
}
```

---

# 29. Explicit deny

VoltStack deberá soportar:

```text
DENY
```

explícito.

---

# 30. Default deny

Para operaciones protegidas:

```text
No matching authority
→ DENY
```

---

# 31. UNKNOWN ≠ ALLOW

Si el sistema no puede resolver una condición de seguridad:

```text
UNKNOWN
```

no deberá convertirse silenciosamente en:

```text
ALLOW
```

---

# 32. PermissionDecision

```php
enum PermissionDecisionStatus
{
    case ALLOW;
    case DENY;
    case UNKNOWN;
}
```

---

# 33. Decision object

```php
final readonly class DatabasePermissionDecision
{
    public function __construct(
        public PermissionDecisionStatus $status,
        public PermissionReasonCode $reason,
        public PermissionConstraints $constraints,
        public PermissionEvidence $evidence,
    ) {}
}
```

---

# 34. Decision ≠ boolean

Internamente no bastará:

```php
true
false
```

porque se necesita preservar:

```text
reason
constraints
source
policy generation
scope
evidence
unknown state
```

---

# 35. Public convenience API

Podrá ofrecer:

```php
$permissions->allows($request);
```

pero será una vista sobre el decision object completo.

---

# 36. PermissionRequest

```php
final readonly class DatabasePermissionRequest
{
    public function __construct(
        public DatabasePrincipal $principal,
        public DatabaseAction $action,
        public DatabaseResource $resource,
        public DatabaseScope $scope,
        public DatabasePermissionContext $context,
    ) {}
}
```

---

# 37. Scope

Scope limita dónde es válida una permission.

---

# 38. Scope hierarchy

```text
Global
├── Database
│   ├── Schema
│   │   ├── Table
│   │   │   └── Column
│   │   └── Entity
│   │       ├── Instance
│   │       └── Field
│   └── Shard
└── Tenant
```

Pero no todos estos scopes forman un único árbol simple.

---

# 39. Scope dimensions

Se modelarán como dimensiones independientes cuando sea necesario:

```text
TenantScope
DatabaseScope
ShardScope
SchemaScope
TableScope
EntityScope
FieldScope
RowScope
OperationScope
```

---

# 40. Why multidimensional?

Una permission puede significar:

```text
Tenant = ACME
Entity = Invoice
Action = READ
Fields = [id, total, status]
Rows = owner_id = current_user
```

No puede expresarse correctamente con un simple string:

```text
invoice.read
```

---

# 41. TenantScope

```php
final readonly class TenantScope
{
    public function __construct(
        public TenantScopeType $type,
        public ?TenantId $tenant,
    ) {}
}
```

---

# 42. Tenant scope types

```text
CURRENT
SPECIFIC
SET
CROSS_TENANT
NONE
```

---

# 43. CURRENT

Autoridad únicamente sobre el tenant activo.

---

# 44. CROSS_TENANT

Deberá ser una capacidad explícita.

Nunca inferida de:

```text
administrator
system
worker
```

por sí sola.

---

# 45. Cross-tenant access

Regla:

```text
Normal permission
+
different tenant
=
DENY
```

salvo autoridad cross-tenant explícita.

---

# 46. Tenant switching

Cambiar `TenantContext` no otorga permisos.

```text
Context Switch
≠
Authority Grant
```

---

# 47. DatabaseScope

Puede restringir a:

```text
logical database
connection domain
database cluster
```

---

# 48. Logical database ≠ physical connection

La permission deberá expresarse sobre identidad lógica estable.

---

# 49. ShardScope

Podrá restringir autoridad a:

```text
single shard
shard set
all shards within domain
```

---

# 50. Shard routing ≠ permission

Que Partition Router determine:

```text
Shard 7
```

no significa que el actor tenga acceso a Shard 7.

---

# 51. Permission evaluation after routing?

Dependerá de la información requerida.

Conceptualmente:

```text
Initial Permission Evaluation
        ↓
Semantic Query
        ↓
Partition Routing
        ↓
Shard-aware Constraint Validation
```

---

# 52. No execution-before-security

La resolución tardía de shard nunca deberá requerir ejecutar primero una operación protegida.

---

# 53. SchemaScope

Ejemplo:

```text
schema = accounting
```

---

# 54. TableScope

Ejemplo:

```text
table = invoices
```

---

# 55. EntityScope

Ejemplo:

```text
entity = Invoice
```

---

# 56. ORM resource ≠ table resource

Una Entity puede involucrar:

```text
multiple tables
embedded values
relationships
inheritance
```

Por tanto:

```text
EntityPermission
≠
TablePermission
```

---

# 57. FieldScope

Permite restringir atributos.

Ejemplo:

```text
Customer:
    READ:
        id
        name
        status

    DENY:
        ssn
        password_hash
```

---

# 58. ColumnScope ≠ FieldScope

Un campo ORM puede mapearse a:

```text
one column
multiple columns
derived representation
encrypted representation
```

---

# 59. Sensitive field permissions

Integración directa con:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
```

---

# 60. Sensitive actions

Se distinguirán:

```text
READ
READ_SENSITIVE
REVEAL_SENSITIVE
```

---

# 61. Why separate reveal?

Una aplicación puede estar autorizada para procesar un dato cifrado sin estar autorizada para revelar su plaintext al actor.

Por tanto:

```text
CanProcess
≠
CanReveal
```

---

# 62. Example

```text
Service:
    READ encrypted tax identifier
    use for matching
    DENY reveal plaintext
```

---

# 63. RowScope

Permite restringir subconjuntos de registros.

Ejemplo:

```text
Invoice.owner_id = CurrentUserId
```

---

# 64. RowScope ≠ post-filter

Nunca:

```text
SELECT all invoices
→ hydrate
→ remove unauthorized rows
```

como estrategia general.

---

# 65. Security predicate

RowScope deberá traducirse, cuando sea posible, a una restricción semántica del Query Model:

```text
OriginalQuery
+
SecurityPredicate
=
AuthorizedQuery
```

---

# 66. Security predicates are mandatory constraints

No deberán tratarse como filtros opcionales.

---

# 67. Query optimizer interaction

Optimizer podrá reorganizar una security predicate únicamente si preserva exactamente su semántica.

---

# 68. Optimizer cannot remove security constraint

Invariante crítica.

---

# 69. Security predicate tagging

AST deberá poder identificar nodos:

```text
SECURITY_CONSTRAINT
```

para impedir eliminaciones/rewrite inseguro.

---

# 70. User predicate ≠ security predicate

Aunque ambos terminen como:

```text
WHERE
```

su origen semántico será diferente.

---

# 71. QueryScope

Podrá restringir:

```text
allowed operations
allowed resources
allowed joins
allowed projections
maximum rows
aggregation
locking
raw expressions
subqueries
```

---

# 72. Permission constraints

Un ALLOW puede incluir límites.

Ejemplo:

```text
ALLOW READ Customer
BUT
maxRows = 1000
fields = [id, name]
tenant = current
rawExpressions = false
```

---

# 73. ALLOW ≠ unrestricted

Esta distinción será fundamental.

---

# 74. PermissionConstraints

```php
final readonly class PermissionConstraints
{
    public function __construct(
        public ?FieldAccessConstraint $fields,
        public ?RowAccessConstraint $rows,
        public ?TenantAccessConstraint $tenant,
        public ?QueryCapabilityConstraint $query,
        public ?ResourceLimitConstraint $resources,
    ) {}
}
```

---

# 75. Constraint composition

Si varias reglas aplican:

```text
EffectiveConstraint
=
Intersection(
    applicable constraints
)
```

para restricciones de acceso.

---

# 76. Permission composition

Conceptualmente:

```text
Grants
→ determine candidate authority

Denies
→ remove prohibited authority

Constraints
→ narrow remaining authority
```

---

# 77. Explicit deny precedence

Default recomendado:

```text
Explicit DENY
>
Explicit ALLOW
>
Default DENY
```

dentro del mismo dominio de policy.

---

# 78. Policy-specific composition

Sistemas externos podrán definir reglas más sofisticadas mediante Authorization integration.

Database deberá recibir un resultado canónico.

---

# 79. PermissionSet

```php
final readonly class DatabasePermissionSet
{
    /**
     * @param list<DatabasePermission> $permissions
     */
    public function __construct(
        public array $permissions,
        public PermissionSetGeneration $generation,
    ) {}
}
```

---

# 80. Immutable PermissionSet

Una vez compilado:

```text
PermissionSet
→ immutable
```

---

# 81. Permission generation

Toda compilación deberá tener:

```text
PermissionSetGeneration
```

para cache/invalidation/debugging/audit.

---

# 82. PermissionResolver

Obtiene authority aplicable.

```php
interface DatabasePermissionResolver
{
    public function resolve(
        DatabasePrincipal $principal,
        DatabasePermissionContext $context,
    ): DatabasePermissionSet;
}
```

---

# 83. Resolver ≠ Evaluator

```text
Resolver
→ obtiene permisos

Evaluator
→ decide una solicitud
```

---

# 84. DatabasePermissionEvaluator

```php
interface DatabasePermissionEvaluator
{
    public function evaluate(
        DatabasePermissionRequest $request,
        DatabasePermissionSet $permissions,
    ): DatabasePermissionDecision;
}
```

---

# 85. PermissionContext

```php
final readonly class DatabasePermissionContext
{
    public function __construct(
        public ExecutionContextId $execution,
        public ?TenantId $tenant,
        public DatabaseDomainId $database,
        public ?ShardId $shard,
        public ?TransactionId $transaction,
        public SecurityContext $security,
        public PermissionContextMetadata $metadata,
    ) {}
}
```

---

# 86. Context-sensitive permissions

Una permission podrá depender de:

```text
tenant
database
operation origin
authentication strength
delegation
request type
job identity
security posture
transaction state
```

---

# 87. Time-dependent permissions

Si se soportan:

```text
validFrom
validUntil
```

deberán utilizar Clock abstraction.

---

# 88. Temporary permissions

Ejemplo:

```text
support engineer
→ temporary customer data access
→ 30 minutes
```

---

# 89. Temporary authority

Debe contener:

```text
issuer
principal
scope
actions
issuedAt
expiresAt
reason/reference
```

---

# 90. Expiration

```text
Expired permission
=
DENY
```

---

# 91. Clock uncertainty

No inventar validez cuando no puede verificarse una restricción crítica.

---

# 92. Delegation

Principal A podrá delegar authority a B únicamente cuando A tenga capacidad de delegación apropiada.

---

# 93. Delegated authority

Nunca podrá exceder el authority delegable del principal origen.

Formalmente:

```text
Authority(B delegated by A)
⊆
DelegableAuthority(A)
```

---

# 94. Delegation chain

```text
Administrator
    ↓
Support Session
    ↓
Support Agent
    ↓
Tenant Resource
```

deberá preservarse.

---

# 95. Delegation ≠ impersonation

Delegation:

```text
B acts with authority delegated by A
```

Impersonation:

```text
A operates under an effective identity/context of B
```

Son conceptos distintos.

---

# 96. Impersonation

Deberá preservar:

```text
authenticated principal
effective principal
impersonator
tenant
reason
session ID
```

---

# 97. No identity erasure

Nunca:

```text
Admin impersonates User
→ principal = User only
```

La identidad administrativa original deberá conservarse.

---

# 98. Audit integration

Todo acceso privilegiado podrá enviar esta cadena al Query Audit System.

---

# 99. Administrative permissions

Las operaciones administrativas tendrán acciones separadas.

---

# 100. Database admin ≠ application CRUD

Ejemplo:

```text
ADMINISTER_DATABASE
```

no deberá otorgarse a usuarios ordinarios.

---

# 101. Schema permissions

Acciones:

```text
CREATE_SCHEMA
ALTER_SCHEMA
DROP_SCHEMA
CREATE_TABLE
ALTER_TABLE
DROP_TABLE
CREATE_INDEX
DROP_INDEX
```

podrán modelarse con mayor granularidad.

---

# 102. Schema Builder security

Antes de compilar/ejecutar una operación protegida:

```text
Schema AST
→ Permission Analysis
→ Safety Analysis
→ Compiler
→ Executor
```

---

# 103. Permission ≠ migration safety

Tener permiso para:

```text
DROP COLUMN
```

no significa que la operación sea segura operacionalmente.

Por tanto:

```text
Authorization
≠
Migration Safety
```

---

# 104. Required combination

Una migration destructiva puede necesitar:

```text
Permission = ALLOW
AND
SafetyDecision = ALLOW
```

---

# 105. Migration permissions

Separar:

```text
RUN_MIGRATION
ROLLBACK_MIGRATION
RUN_DESTRUCTIVE_MIGRATION
MANAGE_MIGRATION_HISTORY
```

---

# 106. Migration role

Las credenciales de migration podrán tener privilegios físicos superiores a runtime credentials.

---

# 107. Runtime cannot borrow migration authority

Nunca elevar automáticamente credenciales porque una query falló por permisos.

---

# 108. Raw SQL permissions

Raw SQL será una capability especial.

---

# 109. Raw SQL ≠ unrestricted database access

Podrán existir:

```text
EXECUTE_RAW_READ
EXECUTE_RAW_WRITE
EXECUTE_RAW_DDL
```

---

# 110. Raw SQL security

Aun con permission:

```text
SQL injection protections
binding requirements
resource limits
credential limits
audit
```

seguirán activos.

---

# 111. Raw SQL resource discovery

Cuando sea posible, se analizarán recursos afectados.

Si no puede determinarse:

```text
ResourceScope = UNKNOWN
```

---

# 112. UNKNOWN raw resource

No deberá asumirse seguro.

Una policy estricta podrá rechazarlo.

---

# 113. Import permissions

Separar:

```text
IMPORT_DATA
IMPORT_SENSITIVE_DATA
IMPORT_CROSS_TENANT_DATA
```

---

# 114. Export permissions

Separar:

```text
EXPORT_DATA
EXPORT_SENSITIVE_DATA
EXPORT_AUDIT_DATA
EXPORT_CROSS_TENANT_DATA
```

---

# 115. Export ≠ READ

Un actor puede tener permiso para visualizar datos pero no para extraerlos masivamente.

---

# 116. Critical invariant

```text
READ
≠
EXPORT
```

---

# 117. Bulk permissions

Asimismo:

```text
UPDATE
≠
BULK_UPDATE
```

y:

```text
DELETE
≠
BULK_DELETE
```

---

# 118. Why?

El impacto potencial cambia radicalmente.

---

# 119. Bulk limits

Un permission decision podrá establecer:

```text
maxAffectedRows
requireTransaction
requireAudit
requireExplicitPredicate
```

---

# 120. Full-table mutation

Una policy puede requerir capability especial:

```text
ALLOW_FULL_TABLE_MUTATION
```

---

# 121. Predicate presence ≠ safe scope

Tener un `WHERE` no garantiza que la mutación sea pequeña.

Por tanto deberán existir resource limits independientes.

---

# 122. Backup permissions

Separar:

```text
CREATE_BACKUP
READ_BACKUP
EXPORT_BACKUP
DELETE_BACKUP
```

---

# 123. Restore permissions

Restore deberá tener permiso explícito.

---

# 124. Restore ≠ Write

Un restore puede reemplazar enormes cantidades de estado.

---

# 125. Audit permissions

Separar:

```text
READ_AUDIT
SEARCH_AUDIT
EXPORT_AUDIT
MANAGE_AUDIT_RETENTION
VERIFY_AUDIT_INTEGRITY
```

---

# 126. Audit writer

No necesita necesariamente:

```text
READ_AUDIT
```

---

# 127. Audit reader

No necesita:

```text
WRITE_APPLICATION_DATA
```

---

# 128. Least privilege

Estas separaciones deberán reflejarse también en infraestructura cuando sea posible.

---

# 129. Sensitive data permissions

Clasificaciones podrán exigir capabilities.

Ejemplo:

```text
PUBLIC
→ READ

INTERNAL
→ READ_INTERNAL

CONFIDENTIAL
→ READ_CONFIDENTIAL

RESTRICTED
→ READ_RESTRICTED
```

---

# 130. Classification ≠ permission

Classification describe sensibilidad.

Permission concede autoridad.

---

# 131. Reveal permissions

Para secretos/campos cifrados:

```text
READ_ENCRYPTED
DECRYPT
REVEAL
```

podrán ser distintas.

---

# 132. Query projection enforcement

Supongamos:

```php
Customer::query()
    ->select(['id', 'name', 'ssn'])
    ->get();
```

Si:

```text
ssn → DENY
```

la operación no deberá devolver silenciosamente `ssn`.

---

# 133. Reject vs rewrite

Policy podrá decidir:

```text
REJECT_QUERY
```

o, en APIs explícitamente diseñadas para ello:

```text
REMOVE_FORBIDDEN_FIELDS
```

---

# 134. Default

Para queries explícitas:

```text
forbidden requested field
→ DENY
```

es el default más seguro.

---

# 135. Silent field removal danger

Puede producir:

```text
incorrect business decisions
partial entities
unexpected nulls
security ambiguity
```

---

# 136. Wildcard projections

```sql
SELECT *
```

requieren expansión semántica cuando field permissions aplican.

---

# 137. ORM entity hydration

Si una entidad contiene campos que el actor no puede leer, VoltStack deberá evitar fingir una entidad completamente cargada.

---

# 138. Strategies

Podrán utilizarse:

```text
safe projection
DTO
partial entity with LoadedFieldMask
restricted value placeholder
deny
```

según contrato.

---

# 139. Restricted ≠ NULL

Un campo prohibido no deberá convertirse automáticamente en:

```text
NULL
```

porque:

```text
Restricted
≠
Database NULL
```

---

# 140. Restricted value state

Podrá existir:

```text
UNAVAILABLE_BY_PERMISSION
```

en capas que soporten esa representación.

---

# 141. Relationship permissions

Cargar:

```text
Order → Customer
```

requiere autoridad sobre el target cuando la relación expone sus datos.

---

# 142. Relationship permission ≠ FK permission

---

# 143. Eager loading

```php
Order::with('customer')->get();
```

deberá considerar:

```text
Order permission
+
Customer permission
+
relationship scope
```

---

# 144. Lazy loading

No podrá convertirse en bypass.

```text
root authorized
→ lazy relation
→ unauthorized target
```

deberá ser rechazado.

---

# 145. Batch relationship loading

Igualmente preservará permission context.

---

# 146. N+1 optimization

Una optimización de loading nunca podrá ampliar authority.

---

# 147. Aggregations

Pregunta:

```text
Can COUNT restricted rows be allowed
without allowing row details?
```

Sí, potencialmente.

---

# 148. Aggregate permission

Podrá distinguirse:

```text
READ_DETAIL
READ_AGGREGATE
```

---

# 149. Aggregation leakage

Counts, min/max y estadísticas también pueden revelar información.

Por tanto serán policy-aware.

---

# 150. GROUP BY

Puede exponer categorías sensibles aunque no exponga filas.

---

# 151. Existence queries

```text
EXISTS
```

también revelan información.

---

# 152. Metadata access

Schema introspection puede revelar:

```text
table names
column names
structure
indexes
relationships
```

y deberá tener permisos administrativos apropiados cuando sea externally reachable.

---

# 153. Query metadata ≠ harmless

---

# 154. Explain plans

`EXPLAIN` puede revelar estructura interna.

Podrá requerir:

```text
QUERY_DIAGNOSTICS
```

---

# 155. Query profiler access

Podrá requerir:

```text
READ_QUERY_PROFILE
```

---

# 156. Debug toolbar

No deberá mostrar recursos para los que el viewer carece de autoridad apropiada.

---

# 157. Permission enforcement points

VoltStack deberá aplicar controles en varios límites.

```text
Public API
   ↓
Semantic Query
   ↓
ORM / Schema / Operation
   ↓
Planner
   ↓
Execution Boundary
```

---

# 158. Why multiple enforcement points?

Porque algunas restricciones solo pueden conocerse después de:

```text
metadata resolution
relationship expansion
projection expansion
routing
```

---

# 159. Defense against internal bypass

No deberá depender únicamente de Controllers o HTTP middleware.

---

# 160. Database security boundary

El subsistema deberá ser capaz de recibir un `DatabasePermissionContext` independientemente de HTTP.

Esto permite:

```text
CLI
Jobs
Workers
Tests
WebSockets
RPC
Scheduled Tasks
```

---

# 161. HTTP ≠ security context

---

# 162. Query Permission Analysis

Pipeline:

```text
Query Model
    ↓
Semantic Resolution
    ↓
Resource Discovery
    ↓
Permission Requirements
    ↓
Permission Evaluation
    ↓
Security Constraint Injection
    ↓
Validation
    ↓
Optimization
    ↓
Compilation
```

---

# 163. Security before compilation

El Compiler no decidirá permisos.

---

# 164. Compiler rule

```text
Compiler
→ compile authorized Query Model
```

No:

```text
Compiler
→ inspect current user
```

---

# 165. Executor rule

Executor podrá verificar que el plan contiene evidencia de security validation cuando el modo protegido lo requiera.

---

# 166. AuthorizedPlanToken

Conceptualmente puede existir:

```php
final readonly class DatabaseSecurityValidation
{
    public function __construct(
        public PermissionDecisionId $decision,
        public PermissionSetGeneration $generation,
        public SecurityConstraintFingerprint $constraints,
    ) {}
}
```

---

# 167. Security validation ≠ authorization token for users

Será metadata interna.

No deberá exponerse como bearer credential.

---

# 168. Query mutation after authorization

Si el Query Model cambia después de ser autorizado:

```text
previous authorization
→ invalid
```

---

# 169. Query fingerprint binding

Security decision podrá vincularse a:

```text
semantic query fingerprint
```

---

# 170. Plan mutation

Igualmente deberá invalidar security evidence si altera semántica protegida.

---

# 171. TOCTOU

El sistema deberá reducir:

```text
Time Of Check
vs
Time Of Use
```

en permission evaluation.

---

# 172. Permission generation binding

Si permissions cambian entre planning y execution, policy decidirá si se requiere reevaluación.

---

# 173. Short-lived operation

Normalmente:

```text
resolve
evaluate
execute
```

dentro del mismo scoped context.

---

# 174. Long-running processing

Chunk/Lazy/Large Dataset pueden durar mucho tiempo.

---

# 175. Long operation permission policy

Podrá ser:

```text
SNAPSHOT_AT_START
REVALIDATE_PER_CHUNK
REVALIDATE_PERIODICALLY
CUSTOM
```

---

# 176. Default for privileged long operations

Revalidación periódica/per chunk puede ser apropiada.

---

# 177. Revocation semantics

Debe ser explícito cuándo una revocación afecta operaciones ya iniciadas.

---

# 178. PermissionSnapshot

Si se usa snapshot:

```text
PermissionSetGeneration
```

deberá preservarse.

---

# 179. Permission cache

Puede existir para reducir costo de resolución.

---

# 180. Permission Cache ≠ authority source

El cache reutiliza authority previamente derivada.

No la inventa.

---

# 181. Permission cache key

Conceptualmente:

```text
PrincipalId
+
SecurityDomain
+
TenantContext
+
PermissionSetGeneration
+
AuthorizationGeneration
```

---

# 182. Cache key security

Nunca usar únicamente:

```text
UserId
```

si authority cambia por tenant/contexto.

---

# 183. Permission invalidation

Cambios en:

```text
role
permission
policy
tenant membership
delegation
security generation
```

deberán invalidar authority cache relevante.

---

# 184. TTL ≠ revocation correctness

Un TTL largo puede retrasar revocaciones.

Por tanto:

```text
TTL
≠
Permission Correctness
```

---

# 185. Strong revocation

Sistemas que lo requieran podrán utilizar:

```text
generation tokens
event invalidation
central permission version
short-lived authority snapshots
```

---

# 186. Cached DENY

También puede cachearse cuando policy lo permita.

---

# 187. Unknown decision

No deberá cachearse como ALLOW.

---

# 188. Permission cache poisoning

Los inputs de key deberán ser canónicos y no controlables arbitrariamente.

---

# 189. Transaction permissions

Abrir una transacción no otorga autoridad sobre todas las operaciones ejecutadas dentro.

---

# 190. Transaction ≠ permission scope expansion

---

# 191. Transaction consistency

Las permission constraints deberán permanecer coherentes durante la operación según policy.

---

# 192. Savepoints

No modifican authority.

---

# 193. Retry

Un retry deberá preservar:

```text
logical principal
permission context
tenant
delegation
```

---

# 194. Retry and revocation

Policy podrá requerir reevaluación antes del siguiente attempt.

---

# 195. Unknown transaction outcome

No deberá provocar un retry de operación privilegiada sin aplicar las reglas del Transaction Retry System.

---

# 196. Read/write routing

Permission se expresa sobre recurso lógico, no sobre replica concreta.

---

# 197. Endpoint permission

Operaciones administrativas sobre endpoints sí podrán requerir recursos físicos/topológicos específicos.

---

# 198. Replica read

No debe reducir seguridad.

---

# 199. Failover

Cambiar writer no cambia authority semántica.

---

# 200. Sharding

Una query cross-shard deberá satisfacer authority para su alcance completo.

---

# 201. Partial shard authority

No deberá ejecutar silenciosamente solo shards permitidos si el caller solicitó una operación global, salvo API explícita para resultados parciales.

---

# 202. Global operation

```text
Requested Scope = all shards
Authorized Scope = shard A
```

Default:

```text
DENY
```

---

# 203. Distributed permission evidence

Podrá representar:

```text
Shard A → ALLOW
Shard B → ALLOW
Shard C → DENY
```

pero la decisión global depende de la semántica solicitada.

---

# 204. Partial authorization ≠ global authorization

---

# 205. Resource governance

Permission y resource governance serán distintas.

```text
Authorized
≠
Unlimited
```

---

# 206. Example

Usuario puede:

```text
EXPORT Customer
```

pero con:

```text
maxRows = 10,000
```

---

# 207. Resource limit decision

Puede producir:

```text
DENY_RESOURCE_LIMIT
```

---

# 208. Rate limits

Podrán integrarse externamente.

No forman parte central del Permission Model.

---

# 209. Query timeout

No es permiso.

---

# 210. Permission condition language

VoltStack no deberá introducir inicialmente un lenguaje arbitrario inseguro de expresiones ejecutables.

---

# 211. Typed conditions

Preferir:

```text
TenantEqualsCurrent
FieldInSet
OwnerEqualsPrincipal
ClassificationAtMost
ResourceBelongsToScope
```

---

# 212. Arbitrary PHP closure permissions

Podrán existir en application Authorization layer, pero no deberán convertirse en metadata serializable/compilable del Database core.

---

# 213. PermissionCondition

```php
interface DatabasePermissionCondition
{
    public function evaluate(
        DatabasePermissionEvaluationContext $context
    ): ConditionResult;
}
```

---

# 214. ConditionResult

```text
MATCH
NO_MATCH
UNKNOWN
```

---

# 215. UNKNOWN condition

No deberá convertirse en MATCH.

---

# 216. Security context propagation

```text
Request / Job / CLI
        ↓
SecurityContext
        ↓
DatabaseContext
        ↓
Query / ORM / Schema
        ↓
Permission Evaluation
```

---

# 217. Context capture

Una operación async deberá capturar únicamente authority segura/reconstructible.

---

# 218. Do not serialize live permission engine

Jobs no deberán serializar:

```text
service container
live evaluator
DB connection
current request
```

---

# 219. Job authority

Preferir:

```text
PrincipalReference
TenantReference
RequiredCapability
DelegationReference
```

y resolver authority al ejecutar.

---

# 220. Job creation authority ≠ execution authority

Una permission válida al encolar un job puede haber sido revocada cuando se ejecute.

---

# 221. Revalidation

Default seguro para operaciones privilegiadas async:

```text
revalidate at execution
```

---

# 222. System jobs

No deberán recibir `DATABASE_ADMIN` automáticamente.

---

# 223. Service principals

Cada servicio deberá tener capacidades mínimas.

---

# 224. Background workers

Worker identity será distinta del principal lógico de cada job.

---

# 225. Infrastructure identity ≠ logical actor

Ejemplo:

```text
Worker Process = worker-12
Logical Principal = billing-service
Requested By = user-42
```

Las tres identidades pueden ser relevantes.

---

# 226. Audit integration

`233_DATABASE_QUERY_AUDIT_SYSTEM.md` recibirá:

```text
principal
effective principal
action
resource
scope
decision
constraints
permission generation
reason code
delegation
impersonation
```

---

# 227. Denied audit

Las denegaciones sensibles podrán generar:

```text
AuditOutcome::DENIED
```

---

# 228. PermissionReasonCode

Ejemplos:

```text
ALLOW_EXPLICIT
DENY_EXPLICIT
DENY_DEFAULT
DENY_FIELD
DENY_ROW_SCOPE
DENY_TENANT
DENY_CROSS_TENANT
DENY_SHARD
DENY_SENSITIVE
DENY_RAW_SQL
DENY_EXPORT
DENY_RESOURCE_LIMIT
DENY_EXPIRED
DENY_REVOKED
DENY_UNKNOWN_CONTEXT
```

---

# 229. Reason code ≠ sensitive explanation

No incluir información que revele:

```text
existence of hidden record
secret policy
credential state
sensitive tenant information
```

---

# 230. External error messages

Una decisión interna:

```text
DENY_ROW_SCOPE
```

puede convertirse externamente en:

```text
Resource not available
```

según security policy.

---

# 231. Permission exceptions

Jerarquía:

```text
DatabasePermissionException
├── DatabaseAccessDeniedException
├── DatabasePermissionResolutionException
├── DatabasePermissionContextException
├── DatabaseScopeViolationException
├── DatabaseFieldAccessDeniedException
├── DatabaseRowAccessDeniedException
├── DatabaseTenantAccessDeniedException
├── DatabaseCrossTenantAccessDeniedException
├── DatabaseSensitiveAccessDeniedException
├── DatabaseRawQueryDeniedException
├── DatabaseExportDeniedException
├── DatabaseAdministrativeOperationDeniedException
└── DatabasePermissionUnknownException
```

---

# 232. Exception safety

No deberán contener:

```text
passwords
tokens
sensitive values
raw credentials
unredacted SQL bindings
```

---

# 233. Telemetry

El sistema deberá producir telemetry bounded.

---

# 234. Metrics

Ejemplos:

```text
db.permission.evaluations
db.permission.allowed
db.permission.denied
db.permission.unknown
db.permission.cache.hit
db.permission.cache.miss
db.permission.constraint.applied
db.permission.cross_tenant.denied
```

---

# 235. Metric dimensions

Permitidas:

```text
action category
resource category
decision
reason category
```

---

# 236. Avoid cardinality explosion

No usar como metric labels:

```text
UserId
TenantId
EntityId
SQL
email
token
```

---

# 237. Tracing

Span metadata podrá indicar:

```text
db.permission.decision = allow
db.permission.action = read
db.permission.resource_type = entity
```

sin revelar identidad sensible.

---

# 238. Debugging

Debug Information podrá mostrar:

```text
Action: READ
Resource: Customer
Decision: ALLOW
Constraints:
    tenant=current
    fields=7/12
Policy generation: 192
```

solo en contextos autorizados.

---

# 239. Debug ≠ permission bypass

Activar debug nunca deberá reducir controles.

---

# 240. Production debug

No deberá exponer permission internals innecesariamente.

---

# 241. Persistent runtimes

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 242. Request-scoped state

Deberá ser scoped:

```text
CurrentPrincipal
EffectivePrincipal
PermissionContext
PermissionSet
DelegationContext
ImpersonationContext
TenantContext
TemporaryDecisionCache
```

---

# 243. No static principal

Prohibido:

```php
DatabasePermission::$currentUser
```

---

# 244. No static PermissionSet

Un PermissionSet mutable global podría filtrar authority entre requests.

---

# 245. Shared immutable components

Podrán ser worker-global:

```text
permission metadata
compiled rule definitions
resource descriptors
immutable evaluators
```

si no contienen request state.

---

# 246. Worker reset

Al finalizar una operación:

```text
clear principal
clear effective principal
clear permission set
clear temporary decision cache
clear delegation
clear impersonation
clear tenant binding
```

---

# 247. Coroutine isolation

OpenSwoole deberá mantener contextos independientes.

---

# 248. Context leak severity

Una filtración de PermissionContext entre requests será considerada:

```text
CRITICAL SECURITY FAILURE
```

---

# 249. Permission decision cache

Deberá estar scoped o correctamente keyed por todas las dimensiones de autoridad.

---

# 250. Never reuse incomplete key

Nunca:

```text
cache["READ:Customer"]
```

sin principal/scope/context/generation cuando éstos sean relevantes.

---

# 251. Testing architecture

Deberán probarse:

```text
default deny
explicit allow
explicit deny
deny precedence
tenant isolation
cross-tenant authority
field restrictions
row restrictions
sensitive data
raw SQL
bulk operations
exports
migrations
backup/restore
audit access
delegation
impersonation
temporary authority
revocation
permission cache
persistent runtime isolation
```

---

# 252. Permission assertions

Ejemplo:

```php
$this->assertDatabasePermissionAllowed(
    principal: $user,
    action: DatabaseAction::READ,
    resource: Customer::class,
);
```

---

# 253. Denial assertion

```php
$this->assertDatabasePermissionDenied(
    principal: $user,
    action: DatabaseAction::EXPORT,
    resource: Payroll::class,
);
```

---

# 254. Query scope test

```php
$result = Customer::query()
    ->forPrincipal($user)
    ->get();
```

deberá verificar que no aparecen filas fuera del RowScope.

---

# 255. Security property test

Para cualquier fila `r`:

```text
¬Authorized(P, READ, r)
⇒
r ∉ Result(P)
```

---

# 256. Field property test

Para cualquier campo `f`:

```text
¬Authorized(P, READ, f)
⇒
f ∉ ExposedProjection(P)
```

salvo representación explícita de restricted/unavailable.

---

# 257. Tenant property

Para principal sin cross-tenant authority:

```text
Tenant(resource) ≠ CurrentTenant(P)
⇒
DENY
```

---

# 258. Fuzz testing

El Permission Model deberá probar combinaciones de:

```text
actions
resources
scopes
tenant states
permission sets
denies
delegations
```

---

# 259. Cache conformance

Debe comprobarse:

```text
permission revoked
→ stale cached ALLOW cannot survive invalidation contract
```

---

# 260. Runtime isolation tests

En worker persistente:

```text
Request A → Admin
Request B → Guest
```

B jamás deberá heredar authority de A.

---

# 261. Directory structure

```text
src/Quantum/Database/Security/Permission/
│
├── Contract/
│   ├── DatabasePrincipal.php
│   ├── DatabasePermissionResolver.php
│   ├── DatabasePermissionEvaluator.php
│   ├── DatabasePermissionCondition.php
│   ├── DatabaseResource.php
│   └── DatabaseScope.php
│
├── Principal/
│   ├── DatabasePrincipalId.php
│   ├── DatabasePrincipalType.php
│   ├── UserDatabasePrincipal.php
│   ├── ServiceDatabasePrincipal.php
│   ├── JobDatabasePrincipal.php
│   ├── SystemDatabasePrincipal.php
│   ├── DelegatedDatabasePrincipal.php
│   └── UnknownDatabasePrincipal.php
│
├── Action/
│   ├── DatabaseAction.php
│   ├── DatabaseActionSet.php
│   ├── DatabaseActionRegistry.php
│   └── DatabaseActionDescriptor.php
│
├── Resource/
│   ├── DatabaseResourceType.php
│   ├── DatabaseResourceIdentity.php
│   ├── EntityResource.php
│   ├── EntityInstanceResource.php
│   ├── EntityFieldResource.php
│   ├── TableResource.php
│   ├── ColumnResource.php
│   ├── SchemaResource.php
│   ├── QueryResource.php
│   ├── MigrationResource.php
│   ├── AuditResource.php
│   └── AdministrativeResource.php
│
├── Scope/
│   ├── GlobalScope.php
│   ├── TenantScope.php
│   ├── DatabaseScope.php
│   ├── ShardScope.php
│   ├── SchemaScope.php
│   ├── TableScope.php
│   ├── EntityScope.php
│   ├── FieldScope.php
│   ├── RowScope.php
│   ├── QueryScope.php
│   └── OperationScope.php
│
├── Permission/
│   ├── DatabasePermission.php
│   ├── DatabasePermissionId.php
│   ├── DatabasePermissionSet.php
│   ├── PermissionSetGeneration.php
│   ├── PermissionEffect.php
│   └── PermissionSource.php
│
├── Request/
│   ├── DatabasePermissionRequest.php
│   ├── DatabasePermissionContext.php
│   └── DatabasePermissionEvaluationContext.php
│
├── Decision/
│   ├── DatabasePermissionDecision.php
│   ├── PermissionDecisionStatus.php
│   ├── PermissionDecisionId.php
│   ├── PermissionReasonCode.php
│   ├── PermissionEvidence.php
│   └── PermissionConstraints.php
│
├── Constraint/
│   ├── FieldAccessConstraint.php
│   ├── RowAccessConstraint.php
│   ├── TenantAccessConstraint.php
│   ├── QueryCapabilityConstraint.php
│   ├── ResourceLimitConstraint.php
│   ├── SecurityPredicate.php
│   └── SecurityConstraintFingerprint.php
│
├── Condition/
│   ├── ConditionResult.php
│   ├── TenantEqualsCurrent.php
│   ├── OwnerEqualsPrincipal.php
│   ├── FieldInSet.php
│   ├── ClassificationConstraint.php
│   └── ResourceBelongsToScope.php
│
├── Delegation/
│   ├── DatabaseDelegation.php
│   ├── DelegationChain.php
│   ├── DelegationAuthority.php
│   └── DelegationValidator.php
│
├── Impersonation/
│   ├── DatabaseImpersonationContext.php
│   ├── EffectivePrincipal.php
│   └── ImpersonationValidator.php
│
├── Temporary/
│   ├── TemporaryDatabasePermission.php
│   ├── TemporaryAuthority.php
│   └── PermissionExpirationEvaluator.php
│
├── Query/
│   ├── QueryPermissionAnalyzer.php
│   ├── QueryResourceResolver.php
│   ├── QuerySecurityConstraintInjector.php
│   ├── QuerySecurityValidator.php
│   └── DatabaseSecurityValidation.php
│
├── ORM/
│   ├── EntityPermissionResolver.php
│   ├── FieldPermissionResolver.php
│   ├── RelationshipPermissionResolver.php
│   └── OrmSecurityConstraintInjector.php
│
├── Schema/
│   ├── SchemaPermissionAnalyzer.php
│   ├── MigrationPermissionAnalyzer.php
│   └── AdministrativePermissionAnalyzer.php
│
├── Cache/
│   ├── DatabasePermissionCache.php
│   ├── PermissionCacheKey.php
│   ├── PermissionCacheEntry.php
│   └── PermissionCacheInvalidator.php
│
├── Integration/
│   ├── AuthorizationPermissionBridge.php
│   ├── AuditPermissionBridge.php
│   ├── SensitiveDataPermissionBridge.php
│   └── TenantPermissionBridge.php
│
├── Telemetry/
│   └── DatabasePermissionTelemetry.php
│
├── Testing/
│   ├── FakeDatabasePrincipal.php
│   ├── FakePermissionResolver.php
│   ├── DatabasePermissionAssertions.php
│   └── DatabasePermissionConformanceSuite.php
│
└── Exception/
    ├── DatabasePermissionException.php
    ├── DatabaseAccessDeniedException.php
    ├── DatabasePermissionResolutionException.php
    ├── DatabasePermissionContextException.php
    ├── DatabaseScopeViolationException.php
    ├── DatabaseFieldAccessDeniedException.php
    ├── DatabaseRowAccessDeniedException.php
    ├── DatabaseTenantAccessDeniedException.php
    ├── DatabaseCrossTenantAccessDeniedException.php
    ├── DatabaseSensitiveAccessDeniedException.php
    ├── DatabaseRawQueryDeniedException.php
    ├── DatabaseExportDeniedException.php
    └── DatabasePermissionUnknownException.php
```

---

# 262. Ejemplo conceptual

Supongamos:

```text
Principal:
    User #42

Tenant:
    ACME

Permission:
    READ Customer

Constraints:
    tenant = ACME
    rows = account_manager_id == 42
    fields =
        id
        name
        email
        status

Denied fields:
    password_hash
    tax_identifier
```

Query solicitada:

```php
Customer::query()
    ->where('status', 'active')
    ->get();
```

VoltStack construirá conceptualmente:

```text
User Predicate:
    status = active

Security Predicate:
    tenant_id = ACME
    AND
    account_manager_id = 42
```

Resultado lógico:

```text
WHERE
    status = 'active'
AND
    tenant_id = ?
AND
    account_manager_id = ?
```

Los valores siguen utilizando bindings normales.

---

# 263. Projection derivation

Si la query no define fields explícitos:

```text
Requested Projection
=
Customer default fields
```

Permission layer obtiene:

```text
Allowed Projection
=
{id, name, email, status}
```

y:

```text
Effective Projection
=
Requested
∩
Allowed
```

solo si la API permite implicit safe projection.

---

# 264. Explicit forbidden field

Si:

```php
Customer::query()
    ->select(['id', 'tax_identifier'])
    ->get();
```

y:

```text
tax_identifier = DENY
```

default:

```text
DatabaseFieldAccessDeniedException
```

---

# 265. Bulk example

Solicitud:

```php
Customer::query()
    ->where('inactive', true)
    ->delete();
```

Permission:

```text
DELETE Customer
```

pero no:

```text
BULK_DELETE Customer
```

Resultado:

```text
DENY
```

---

# 266. Cross-tenant example

Principal:

```text
Tenant = ACME
```

query intenta acceder:

```text
Tenant = OTHER
```

sin:

```text
CROSS_TENANT_ACCESS
```

Resultado:

```text
DENY_CROSS_TENANT
```

---

# 267. Export example

Usuario puede:

```text
READ Payroll
```

pero no:

```text
EXPORT Payroll
```

Entonces:

```php
$payrollExporter->export($query);
```

deberá ser rechazado aunque el mismo usuario pueda visualizar registros individuales.

---

# 268. Security pipeline completo

```text
Principal
    │
    ▼
Authentication Context
    │
    ▼
Authorization Bridge
    │
    ▼
Database Permission Set
    │
    ▼
Semantic Operation
    │
    ▼
Resource Resolution
    │
    ▼
Scope Resolution
    │
    ▼
Permission Evaluation
    │
    ├── DENY
    │     │
    │     ├──► Security Audit
    │     └──► Reject
    │
    └── ALLOW
          │
          ▼
Permission Constraints
          │
          ├── Field Constraints
          ├── Row Constraints
          ├── Tenant Constraints
          ├── Query Constraints
          └── Resource Limits
          │
          ▼
Security-Constrained Query Model
          │
          ▼
Semantic Validation
          │
          ▼
Optimizer
          │
          ▼
Planner
          │
          ▼
Compiler
          │
          ▼
Executor
          │
          ▼
Database Native Privileges
          │
          ▼
Database
```

---

# 269. Architectural invariants

## DB-PERM-001
API accessibility no implicará authority.

## DB-PERM-002
Authentication será distinta de permission evaluation.

## DB-PERM-003
Database Permission Model será distinto del framework Authorization System.

## DB-PERM-004
Database Permission será distinta de native DB GRANT.

## DB-PERM-005
Native database permissions seguirán aplicando.

## DB-PERM-006
VoltStack nunca evadirá native DB permissions.

## DB-PERM-007
Database credentials seguirán least privilege.

## DB-PERM-008
Migration credentials podrán separarse de runtime credentials.

## DB-PERM-009
Audit credentials podrán separarse.

## DB-PERM-010
Principal será distinto de User.

## DB-PERM-011
UNKNOWN principal será explícito.

## DB-PERM-012
UNKNOWN no se convertirá automáticamente en SYSTEM.

## DB-PERM-013
ANONYMOUS será distinto de UNKNOWN.

## DB-PERM-014
DatabaseAction será semántica.

## DB-PERM-015
SQL verb será distinto de DatabaseAction.

## DB-PERM-016
Administrador no implicará authority ilimitada automáticamente.

## DB-PERM-017
DatabaseResource será semántico.

## DB-PERM-018
Resource hierarchy no implicará automatic permission inheritance.

## DB-PERM-019
Resource identities sensibles serán protegidas.

## DB-PERM-020
Permissions soportarán ALLOW y DENY.

## DB-PERM-021
Operaciones protegidas usarán default deny.

## DB-PERM-022
UNKNOWN nunca será ALLOW.

## DB-PERM-023
PermissionDecision será más rico que boolean.

## DB-PERM-024
Permission request incluirá principal.

## DB-PERM-025
Permission request incluirá action.

## DB-PERM-026
Permission request incluirá resource.

## DB-PERM-027
Permission request incluirá scope.

## DB-PERM-028
Permission request incluirá context.

## DB-PERM-029
Scope podrá ser multidimensional.

## DB-PERM-030
Tenant scope será explícito.

## DB-PERM-031
Cross-tenant authority será explícita.

## DB-PERM-032
Cambiar TenantContext no otorgará authority.

## DB-PERM-033
Logical database será distinta de physical connection.

## DB-PERM-034
Shard routing será distinto de shard permission.

## DB-PERM-035
Schema scope será soportado.

## DB-PERM-036
Table scope será soportado.

## DB-PERM-037
Entity scope será soportado.

## DB-PERM-038
Entity permission será distinta de table permission.

## DB-PERM-039
Field scope será soportado.

## DB-PERM-040
Field scope será distinto de column scope.

## DB-PERM-041
Sensitive fields podrán exigir capabilities adicionales.

## DB-PERM-042
CanProcess será distinto de CanReveal.

## DB-PERM-043
Row scope será soportado.

## DB-PERM-044
Row security no será post-filter por default.

## DB-PERM-045
Security predicates se aplicarán antes de ejecución.

## DB-PERM-046
Optimizer no eliminará security predicates.

## DB-PERM-047
Security predicate será distinto de user predicate.

## DB-PERM-048
QueryScope será soportado.

## DB-PERM-049
ALLOW podrá contener constraints.

## DB-PERM-050
ALLOW no significará unrestricted.

## DB-PERM-051
Constraints se compondrán restrictivamente.

## DB-PERM-052
Explicit DENY tendrá precedencia según policy default.

## DB-PERM-053
PermissionSet será immutable.

## DB-PERM-054
PermissionSet tendrá generation.

## DB-PERM-055
Resolver será distinto de Evaluator.

## DB-PERM-056
PermissionContext será scoped.

## DB-PERM-057
Temporary permissions tendrán expiration.

## DB-PERM-058
Expired authority será DENY.

## DB-PERM-059
Delegated authority no excederá delegable authority.

## DB-PERM-060
Delegation chain será preservada.

## DB-PERM-061
Delegation será distinta de impersonation.

## DB-PERM-062
Impersonation preservará authenticated principal.

## DB-PERM-063
Impersonation preservará effective principal.

## DB-PERM-064
Original identity no será borrada.

## DB-PERM-065
Administrative authority será explícita.

## DB-PERM-066
Schema permissions serán separables.

## DB-PERM-067
Permission será distinta de migration safety.

## DB-PERM-068
Runtime no elevará automáticamente credenciales.

## DB-PERM-069
Raw SQL requerirá capability explícita cuando policy lo determine.

## DB-PERM-070
Raw SQL permission no significará unrestricted access.

## DB-PERM-071
Unknown raw SQL scope no será asumido seguro.

## DB-PERM-072
Import authority será distinta de normal CREATE.

## DB-PERM-073
Export authority será distinta de READ.

## DB-PERM-074
Sensitive export podrá requerir capability separada.

## DB-PERM-075
Bulk update será distinto de UPDATE.

## DB-PERM-076
Bulk delete será distinto de DELETE.

## DB-PERM-077
Full-table mutation podrá requerir capability adicional.

## DB-PERM-078
Predicate presence no garantizará safe scope.

## DB-PERM-079
Backup tendrá permissions propias.

## DB-PERM-080
Restore tendrá permissions propias.

## DB-PERM-081
Audit access tendrá permissions propias.

## DB-PERM-082
Audit writer no necesitará necesariamente audit read.

## DB-PERM-083
Classification será distinta de permission.

## DB-PERM-084
Decryption será distinta de reveal.

## DB-PERM-085
Forbidden explicit projection será rechazada por default.

## DB-PERM-086
Restricted field será distinto de NULL.

## DB-PERM-087
Relationship loading preservará permission context.

## DB-PERM-088
Lazy loading no será permission bypass.

## DB-PERM-089
Eager loading no será permission bypass.

## DB-PERM-090
Batch loading no será permission bypass.

## DB-PERM-091
N+1 optimization no ampliará authority.

## DB-PERM-092
Aggregate access podrá diferir de detail access.

## DB-PERM-093
Aggregations podrán revelar información.

## DB-PERM-094
EXISTS será considerado data access.

## DB-PERM-095
Schema metadata podrá requerir permission.

## DB-PERM-096
EXPLAIN podrá requerir diagnostic permission.

## DB-PERM-097
Profiler access podrá requerir permission.

## DB-PERM-098
Debug no será permission bypass.

## DB-PERM-099
Security no dependerá únicamente de HTTP middleware.

## DB-PERM-100
Database permission context funcionará en CLI/jobs/workers.

## DB-PERM-101
Compiler no resolverá current user.

## DB-PERM-102
Compiler recibirá query ya security-validada.

## DB-PERM-103
Executor podrá verificar security validation evidence.

## DB-PERM-104
Internal validation token no será bearer credential.

## DB-PERM-105
Query mutation invalidará previous security decision.

## DB-PERM-106
Security decision podrá ligarse a query fingerprint.

## DB-PERM-107
Plan semantic mutation invalidará security evidence.

## DB-PERM-108
TOCTOU será considerado.

## DB-PERM-109
Permission generation podrá ligarse a operation.

## DB-PERM-110
Long operations tendrán explicit permission revalidation policy.

## DB-PERM-111
Revocation semantics serán explícitas.

## DB-PERM-112
Permission cache será distinto de authority source.

## DB-PERM-113
Permission cache key incluirá context relevante.

## DB-PERM-114
UserId solo no será cache key suficiente en sistemas contextuales.

## DB-PERM-115
Permission invalidation será explícita.

## DB-PERM-116
TTL será distinto de revocation correctness.

## DB-PERM-117
UNKNOWN decision no se cacheará como ALLOW.

## DB-PERM-118
Permission cache deberá resistir key poisoning.

## DB-PERM-119
Transaction no ampliará authority.

## DB-PERM-120
Savepoint no ampliará authority.

## DB-PERM-121
Retry preservará principal lógico.

## DB-PERM-122
Retry preservará tenant context.

## DB-PERM-123
Retry podrá reevaluar authority.

## DB-PERM-124
Read/write routing no alterará semantic authority.

## DB-PERM-125
Failover no alterará semantic authority.

## DB-PERM-126
Cross-shard operation deberá autorizar alcance completo.

## DB-PERM-127
Partial shard authorization no será global authorization.

## DB-PERM-128
Authorized será distinto de unlimited.

## DB-PERM-129
Resource governance será distinto de permission.

## DB-PERM-130
Permission conditions serán typed cuando sea posible.

## DB-PERM-131
Arbitrary code no será permission metadata core.

## DB-PERM-132
UNKNOWN condition no será MATCH.

## DB-PERM-133
SecurityContext será propagado explícitamente.

## DB-PERM-134
Async jobs no serializarán live evaluator.

## DB-PERM-135
Job creation authority será distinta de execution authority.

## DB-PERM-136
Privileged async jobs revalidarán authority por default.

## DB-PERM-137
System jobs no recibirán DATABASE_ADMIN automáticamente.

## DB-PERM-138
Service principals seguirán least privilege.

## DB-PERM-139
Worker identity será distinta del logical actor.

## DB-PERM-140
Permission decisions serán auditables.

## DB-PERM-141
Denied operations serán auditables según policy.

## DB-PERM-142
Reason codes serán estables.

## DB-PERM-143
Reason codes no revelarán recursos ocultos.

## DB-PERM-144
External errors podrán ser menos específicos que internal reason.

## DB-PERM-145
Exceptions no expondrán secrets.

## DB-PERM-146
Telemetry será bounded.

## DB-PERM-147
Metric labels no incluirán high-cardinality identities.

## DB-PERM-148
Debug output respetará permission/security policy.

## DB-PERM-149
CurrentPrincipal será scoped.

## DB-PERM-150
PermissionSet mutable no será global.

## DB-PERM-151
DelegationContext será scoped.

## DB-PERM-152
ImpersonationContext será scoped.

## DB-PERM-153
TenantContext será scoped.

## DB-PERM-154
Decision cache temporal será scoped.

## DB-PERM-155
Worker reset limpiará permission state.

## DB-PERM-156
OpenSwoole mantendrá coroutine isolation.

## DB-PERM-157
RoadRunner mantendrá request isolation.

## DB-PERM-158
FrankenPHP mantendrá request isolation.

## DB-PERM-159
Context leak será critical security failure.

## DB-PERM-160
Permission cache nunca reutilizará incomplete context keys.

## DB-PERM-161
Testing verificará default deny.

## DB-PERM-162
Testing verificará deny precedence.

## DB-PERM-163
Testing verificará tenant isolation.

## DB-PERM-164
Testing verificará field restrictions.

## DB-PERM-165
Testing verificará row restrictions.

## DB-PERM-166
Testing verificará sensitive-data permissions.

## DB-PERM-167
Testing verificará raw SQL restrictions.

## DB-PERM-168
Testing verificará export restrictions.

## DB-PERM-169
Testing verificará delegation.

## DB-PERM-170
Testing verificará impersonation.

## DB-PERM-171
Testing verificará revocation.

## DB-PERM-172
Testing verificará persistent runtime isolation.

## DB-PERM-173
Unauthorized rows nunca aparecerán en results.

## DB-PERM-174
Unauthorized fields nunca se expondrán silenciosamente.

## DB-PERM-175
Security predicates utilizarán parameter binding normal.

## DB-PERM-176
Permission Model no generará SQL.

## DB-PERM-177
Permission Model no ejecutará queries de negocio.

## DB-PERM-178
Permission Model no sustituirá Authorization System.

## DB-PERM-179
Permission Model no sustituirá native DB security.

## DB-PERM-180
Database security utilizará defense in depth.

---

# 270. Modelo formal de autorización

Sea:

```text
P = Principal
A = Action
R = Resource
S = Scope
C = Context
```

Entonces:

```text
D = Evaluate(P, A, R, S, C)
```

donde:

```text
D ∈ {ALLOW, DENY, UNKNOWN}
```

Para ejecutar:

```text
Executable(D)
⇔
D = ALLOW
```

Por tanto:

```text
DENY
→ Reject

UNKNOWN
→ Reject
```

bajo fail-closed.

---

# 271. Modelo formal de constraints

Sea un conjunto de constraints:

```text
C₁, C₂, ..., Cₙ
```

La autoridad efectiva será:

```text
EffectiveAuthority
=
GrantedAuthority
∩ C₁
∩ C₂
...
∩ Cₙ
```

No:

```text
GrantedAuthority
∪ Constraints
```

---

# 272. Modelo de row security

Sea query original:

```text
Q
```

y security predicate:

```text
S(P)
```

Entonces:

```text
Qauthorized
=
Q ∧ S(P)
```

La condición fundamental será:

```text
Result(Qauthorized)
⊆
AuthorizedRows(P)
```

---

# 273. Modelo de field security

Sea:

```text
RequestedFields = Fq
AllowedFields = Fa
```

Entonces:

```text
EffectiveFields
=
Fq ∩ Fa
```

cuando la API permita narrowing.

Si el caller exige explícitamente:

```text
f ∈ Fq
AND
f ∉ Fa
```

default:

```text
DENY
```

---

# 274. Modelo tenant

Sea:

```text
Tresource
```

el tenant del recurso y:

```text
Tprincipal
```

el scope normal del principal.

Sin cross-tenant authority:

```text
Tresource ≠ Tprincipal
⇒
DENY
```

---

# 275. Modelo de delegación

Sea:

```text
Authority(P)
```

y:

```text
Delegable(P)
```

Si P delega a Q:

```text
AuthorityDelegated(Q)
⊆
Delegable(P)
⊆
Authority(P)
```

---

# 276. Modelo de bulk operation

Una permission normal:

```text
UPDATE R
```

no implica:

```text
BULK_UPDATE R
```

Formalmente:

```text
UPDATE ∉⇒ BULK_UPDATE
```

y:

```text
DELETE ∉⇒ BULK_DELETE
```

---

# 277. Modelo de exportación

```text
READ R
∉⇒
EXPORT R
```

porque el riesgo y volumen de exposición son distintos.

---

# 278. Modelo de sensitive data

```text
READ R
∉⇒
READ_SENSITIVE R
```

y:

```text
READ_SENSITIVE R
∉⇒
REVEAL_SENSITIVE R
```

---

# 279. Arquitectura final de seguridad Database

Con los documentos 226–234, el bloque queda conceptualmente:

```text
                 Database Security
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
 Input Security    Access Security   Infrastructure
        │               │                │
        ▼               ▼                ▼
SQL Injection     Permission Model   Credentials
Prevention              │            Connections
        │               ▼
        │          Data Access
        │               │
        │        ┌──────┴──────┐
        │        ▼             ▼
        │    Row/Field      Sensitive
        │    Security         Data
        │        │             │
        └────────┼─────────────┘
                 ▼
            Query / ORM
                 │
                 ▼
             Execution
                 │
                 ▼
              Audit
```

---

# 280. Security decision pipeline

```text
Untrusted Input
      │
      ▼
Input Validation
      │
      ▼
Query Construction
      │
      ▼
Semantic Analysis
      │
      ▼
Resource Resolution
      │
      ▼
Permission Evaluation
      │
      ▼
Data Access Constraints
      │
      ▼
Sensitive Data Rules
      │
      ▼
Query Validation
      │
      ▼
Optimizer
      │
      ▼
Compiler
      │
      ▼
Parameter Binding
      │
      ▼
Connection Security
      │
      ▼
Native DB Privileges
      │
      ▼
Database
      │
      ▼
Audit Evidence
```

---

# 281. Regla maestra final

La regla de seguridad del Database Permission Model será:

> **La autoridad de una operación Database deberá estar ligada al principal, acción, recurso, scope y contexto exactos de esa operación; cualquier ampliación de recurso, tenant, shard, campo, fila, sensibilidad o capacidad requerirá autoridad explícita adicional.**

Formalmente:

```text
Authority
=
f(
    Principal,
    Action,
    Resource,
    Scope,
    Context,
    Constraints
)
```

y nunca:

```text
Authority
=
Reachability(API)
```

Además:

```text
Authentication
≠
Authorization

Authorization
≠
Database Permission

Database Permission
≠
Native DB Privilege

Permission
≠
Unlimited Access

READ
≠
EXPORT

UPDATE
≠
BULK_UPDATE

DELETE
≠
BULK_DELETE

READ
≠
READ_SENSITIVE

READ_SENSITIVE
≠
REVEAL_SENSITIVE

TenantContext
≠
CrossTenantAuthority

ShardRouting
≠
ShardAuthority

Permission
≠
MigrationSafety

UNKNOWN
≠
ALLOW
```

La propiedad final buscada será:

```text
EveryProtectedOperation
→ ExplicitSecurityDecision
→ ConstrainedAuthority
→ SafeExecution
→ AuditableOutcome
```

---

# 282. Estado del Bloque 22

```text
BLOCK 22 — SECURITY

✓ 226_DATABASE_SECURITY_ARCHITECTURE.md
✓ 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
✓ 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
✓ 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
✓ 230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
✓ 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
✓ 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
✓ 233_DATABASE_QUERY_AUDIT_SYSTEM.md
✓ 234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

**Bloque 22 — Security: COMPLETO.**

---

# 283. Siguiente bloque

```text
BLOCK 23 — RESILIENCE
```

Documentos:

```text
235_DATABASE_RESILIENCE_ARCHITECTURE.md
236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
238_DATABASE_RETRY_POLICY_SYSTEM.md
239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 284. Siguiente documento

```text
235_DATABASE_RESILIENCE_ARCHITECTURE.md
```

El siguiente documento establecerá la arquitectura transversal de resiliencia de `VoltStack/Quantum/Database`.

Su objetivo será separar claramente:

```text
Failure Detection
Failure Classification
Failure Containment
Retry
Backoff
Circuit Breaking
Failover
Recovery
Cancellation
Timeout
Resource Protection
Transaction Uncertainty
Connection Recovery
Query Recovery
```

bajo reglas fundamentales como:

```text
Failure
≠
Retryable Failure

Connection Failure
≠
Query Failure

Statement Retry
≠
Transaction Retry

Retry
≠
Recovery

Failover
≠
Retry

Timeout
≠
Cancellation

Unknown Outcome
≠
Failure

Resource Exhaustion
≠
Ordinary Error
```

y especialmente:

> **VoltStack Database deberá recuperarse automáticamente únicamente cuando pueda demostrar que la recuperación preserva la semántica y la seguridad de la operación; ante resultados inciertos, nunca deberá fabricar éxito, fracaso, rollback o seguridad de replay que no pueda demostrar.**