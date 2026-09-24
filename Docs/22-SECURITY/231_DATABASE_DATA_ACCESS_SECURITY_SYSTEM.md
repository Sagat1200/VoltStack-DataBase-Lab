# 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md

# VoltStack Quantum Database
## Data Access Security System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 231 — Data Access Security System  
**Bloque:** 22 — Security  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `230_DATABASE_CONNECTION_SECURITY_SYSTEM.md`  
**Siguiente documento:** `232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack Database controla y hace cumplir **qué operaciones sobre datos están permitidas dentro de un contexto determinado**.

El sistema deberá impedir que las restricciones de acceso puedan omitirse accidentalmente al utilizar diferentes APIs:

```text
Model API
Repository
EntityManager
Query Builder
Relationships
Eager Loading
Pagination
Chunk Processing
Lazy Collections
Bulk Operations
Import
Export
Raw Queries
Distributed Queries
```

La regla central será:

> **Toda operación protegida sobre datos deberá atravesar una frontera de autorización verificable antes de llegar al Execution Engine. Ninguna API de alto nivel deberá convertirse en una ruta alternativa capaz de omitir las restricciones de acceso.**

---

# 2. Problema arquitectónico

Una aplicación puede tener correctamente implementada autorización HTTP:

```text
HTTP Request
    ↓
Controller
    ↓
Authorization
```

pero posteriormente ejecutar:

```php
User::query()->get();
```

sin restricciones.

También podría ocurrir:

```php
$repository->findAll();
```

o:

```php
DB::table('users')->get();
```

o:

```php
Order::query()->export();
```

Por tanto:

```text
Controller Authorization
≠
Database Data Access Enforcement
```

VoltStack necesita una frontera adicional.

---

# 3. Data Access Security ≠ Authorization System

El sistema de Database no deberá convertirse en el sistema general de autorización del framework.

La responsabilidad general seguirá perteneciendo a:

```text
VoltStack Authorization
```

Database consume decisiones y políticas relevantes para datos.

Arquitectura:

```text
Authentication
      ↓
Principal
      ↓
Authorization
      ↓
Data Access Policy
      ↓
Database Query Scope
      ↓
Query Engine
```

---

# 4. Data Access Security ≠ Database Permissions

También:

```text
Application Data Access Policy
≠
Native Database Permissions
```

Ejemplo:

El usuario DB:

```text
voltstack_app
```

puede tener:

```sql
SELECT ON orders
```

pero el usuario de aplicación:

```text
Customer #42
```

solo puede ver sus propios pedidos.

Por tanto:

```text
DB GRANT SELECT
```

no expresa necesariamente:

```text
WHERE customer_id = 42
```

---

# 5. Defense in depth

La arquitectura ideal será:

```text
Application Authorization
        ↓
VoltStack Data Access Security
        ↓
Query Security Constraints
        ↓
Native Database Permissions
        ↓
Optional Native RLS
```

Cada capa complementa a las demás.

---

# 6. Objetivos

El sistema deberá proporcionar:

1. autorización de operaciones de datos;
2. aislamiento por recurso;
3. aislamiento por tenant;
4. aislamiento por shard;
5. restricciones por fila;
6. restricciones por campo;
7. protección de relaciones;
8. protección de agregaciones;
9. protección de conteos;
10. protección de exportaciones;
11. protección de operaciones bulk;
12. protección de raw queries;
13. composición de políticas;
14. fail-closed;
15. evidencia de decisiones;
16. integración con Authorization;
17. integración con Query AST;
18. integración con ORM;
19. integración con Telemetry/Audit;
20. seguridad en runtimes persistentes.

---

# 7. Principio fundamental

VoltStack deberá diferenciar:

```text
CanExecuteOperation
```

de:

```text
WhichDataCanOperationObserve
```

Ejemplo:

```text
ALLOW SELECT Order
```

puede seguir requiriendo:

```text
WHERE organization_id = currentOrganization
```

---

# 8. Access Decision ≠ Query Scope

Una decisión:

```text
ALLOW
```

no necesariamente significa acceso ilimitado.

Podrá producir:

```text
ALLOW
+
mandatory constraints
```

---

# 9. Modelo conceptual

```text
Principal
   +
Security Context
   +
Operation
   +
Resource
   +
Query Intent
        ↓
Data Access Policy Engine
        ↓
DataAccessDecision
        ↓
Mandatory Security Constraints
        ↓
Query AST
        ↓
Semantic Analysis
        ↓
Security Validation
        ↓
Optimizer
        ↓
Compiler
        ↓
Executor
```

---

# 10. Security boundary

La frontera deberá encontrarse **antes de la ejecución**.

No será suficiente:

```text
execute query
↓
filter unauthorized rows in PHP
```

porque para entonces los datos ya fueron obtenidos.

---

# 11. Regla de enforcement

```text
Protected Query
→
Access Evaluation
→
Security Constraint Application
→
Security Validation
→
Execution
```

Nunca:

```text
Protected Query
→
Execution
→
Authorization
```

---

# 12. DataAccessRequest

Contrato central:

```php
final readonly class DataAccessRequest
{
    public function __construct(
        public DataPrincipal $principal,
        public DataOperation $operation,
        public DataResource $resource,
        public DataAccessContext $context,
        public QueryIntent $intent,
    ) {}
}
```

---

# 13. DataPrincipal

Representa la identidad de seguridad relevante para Database.

```php
interface DataPrincipal
{
    public function id(): PrincipalId;

    public function type(): PrincipalType;
}
```

---

# 14. Principal ≠ User Entity

No debe asumirse:

```text
Principal
=
User ORM Entity
```

Un principal podría ser:

```text
authenticated user
service account
system process
queue worker
scheduled task
API client
machine identity
administrator
anonymous context
```

---

# 15. PrincipalId

Debe ser un identificador de seguridad estable.

No una referencia mutable a una Entity.

---

# 16. PrincipalType

Ejemplos:

```text
USER
SERVICE
SYSTEM
ANONYMOUS
MACHINE
CUSTOM
```

---

# 17. DataOperation

VoltStack deberá modelar operaciones semánticamente.

```php
enum DataOperation
{
    case READ;
    case CREATE;
    case UPDATE;
    case DELETE;

    case COUNT;
    case AGGREGATE;

    case EXPORT;
    case IMPORT;

    case BULK_INSERT;
    case BULK_UPDATE;
    case BULK_DELETE;

    case LOCK;
    case RAW_QUERY;

    case ADMINISTRATE;
}
```

---

# 18. CRUD no es suficiente

Operaciones como:

```text
EXPORT
AGGREGATE
LOCK
RAW_QUERY
```

pueden tener riesgos distintos de un simple `READ`.

---

# 19. DataResource

Representará el objetivo lógico.

```php
interface DataResource
{
    public function resourceId(): DataResourceId;
}
```

---

# 20. Resource types

Podrán existir:

```text
EntityResource
TableResource
FieldResource
RelationshipResource
ProjectionResource
DatabaseResource
SchemaResource
CustomResource
```

---

# 21. Entity Resource ≠ Table Resource

Una Entity puede:

```text
map to one table
map to several tables
share table
use inheritance
```

Por tanto ambos conceptos permanecerán separados.

---

# 22. Logical resource first

Las políticas de aplicación deberán favorecer recursos lógicos:

```text
Order
Invoice
Customer
```

sobre nombres físicos:

```text
orders
invoices
customers
```

---

# 23. Physical resource security

Raw Query y operaciones administrativas pueden requerir recursos físicos.

---

# 24. DataAccessContext

```php
final readonly class DataAccessContext
{
    public function __construct(
        public ?TenantId $tenant,
        public ?ShardId $shard,
        public ?OrganizationId $organization,
        public SecurityPurpose $purpose,
        public SecurityPolicyGeneration $policyGeneration,
        public array $attributes = [],
    ) {}
}
```

---

# 25. Context immutability

El contexto efectivo deberá ser:

```text
immutable
scoped
explicitly propagated
```

---

# 26. No static current user

Prohibido:

```php
DataAccess::$currentUser
```

en runtimes persistentes.

---

# 27. DataAccessDecision

```php
final readonly class DataAccessDecision
{
    public function __construct(
        public AccessDecisionEffect $effect,
        public SecurityConstraintSet $constraints,
        public DecisionEvidence $evidence,
    ) {}
}
```

---

# 28. AccessDecisionEffect

```php
enum AccessDecisionEffect
{
    case ALLOW;
    case DENY;
    case UNKNOWN;
}
```

---

# 29. UNKNOWN

Para operaciones protegidas:

```text
UNKNOWN
≠
ALLOW
```

Default:

```text
UNKNOWN
→
DENY
```

---

# 30. ALLOW puede estar restringido

Ejemplo:

```text
ALLOW READ Order

constraints:
    tenant_id = 20
    organization_id = 50
    status != 'internal'
```

---

# 31. SecurityConstraintSet

```php
final readonly class SecurityConstraintSet
{
    public function __construct(
        public array $rowConstraints,
        public array $fieldConstraints,
        public array $relationshipConstraints,
        public array $operationConstraints,
    ) {}
}
```

---

# 32. Security constraints ≠ arbitrary SQL

Nunca deberán representarse principalmente como:

```php
"tenant_id = {$tenantId}"
```

---

# 33. Security Predicate AST

Las restricciones deberán expresarse mediante AST tipado.

Ejemplo conceptual:

```text
SecurityPredicate
├── Equality
│   ├── Field(tenant_id)
│   └── SecurityValue(CurrentTenant)
│
└── Equality
    ├── Field(organization_id)
    └── SecurityValue(CurrentOrganization)
```

---

# 34. SecurityValue

Valores sensibles deberán convertirse en parámetros.

Nunca concatenarse.

---

# 35. Security predicate integration

```text
Application Query Predicate
        +
Mandatory Security Predicate
        ↓
Query Predicate AST
```

---

# 36. AND semantics

Normalmente:

```text
EffectivePredicate
=
ApplicationPredicate
AND
SecurityPredicate
```

---

# 37. Application predicate cannot weaken security predicate

Ejemplo:

```text
Security:
tenant_id = 10

Application:
tenant_id = 20
```

Resultado:

```text
tenant_id = 10
AND tenant_id = 20
```

→ cero resultados.

No:

```text
tenant_id = 20
```

---

# 38. Mandatory predicate provenance

Cada predicate deberá poder indicar:

```text
APPLICATION
SECURITY_MANDATORY
FRAMEWORK_INTERNAL
TENANT_ISOLATION
NATIVE_POLICY
```

---

# 39. Optimizer safety

El Query Optimizer podrá simplificar predicates solo preservando su semántica.

---

# 40. Mandatory security predicate

No deberá eliminarse por:

```text
optimizer
extension
raw builder
relationship loader
pagination planner
```

---

# 41. Security taint/provenance

AST nodes de seguridad podrán contener metadata como:

```text
mandatory = true
origin = DATA_ACCESS_SECURITY
policyGeneration = ...
```

---

# 42. SecurityConstraintId

Cada restricción importante podrá tener ID estable.

Ejemplo:

```text
SEC_ORDER_TENANT_BOUNDARY
```

---

# 43. Row-level access

El sistema deberá soportar restricciones por fila.

Ejemplo:

```text
User can read Order
where
Order.customer_id = Principal.customer_id
```

---

# 44. Row policy

```php
interface RowAccessPolicy
{
    public function constraints(
        DataAccessRequest $request,
    ): RowConstraintResult;
}
```

---

# 45. RowConstraintResult

Puede producir:

```text
ALLOW_ALL
RESTRICT(predicate)
DENY_ALL
UNKNOWN
```

---

# 46. ALLOW_ALL

Debe ser explícito.

La ausencia de policy no deberá interpretarse automáticamente como `ALLOW_ALL` en recursos protegidos.

---

# 47. DENY_ALL

Puede convertirse conceptualmente en:

```text
FALSE
```

pero preferiblemente abortará antes de ejecutar.

---

# 48. Field-level access

VoltStack deberá poder controlar:

```text
SELECT field
UPDATE field
INSERT field
ORDER BY field
GROUP BY field
HAVING field
```

de forma independiente cuando sea necesario.

---

# 49. Field operation matrix

Ejemplo:

| Campo | Read | Create | Update | Sort | Aggregate |
|---|---:|---:|---:|---:|---:|
| `name` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `email` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `salary` | ✗ | ✗ | ✗ | ✗ | ✗ |
| `internal_score` | ✗ | ✗ | ✗ | ✗ | ✓* |

`*` sujeto a policy específica.

---

# 50. Hidden field ≠ inaccessible field

Un campo no incluido en serialization:

```text
hidden
```

no significa que esté protegido a nivel de Database.

---

# 51. ORM hidden attribute ≠ security boundary

Nunca utilizar:

```php
protected $hidden = ['salary'];
```

como sustituto de Data Access Security.

---

# 52. FieldAccessPolicy

```php
interface FieldAccessPolicy
{
    public function evaluate(
        FieldAccessRequest $request,
    ): FieldAccessDecision;
}
```

---

# 53. Projection validation

Antes de ejecutar:

```text
SELECT name, salary
```

el sistema deberá comprobar permisos sobre la proyección.

---

# 54. Security-required hidden fields

Puede ocurrir que una query necesite internamente:

```text
tenant_id
```

para verificar aislamiento, aunque el usuario no pueda proyectarlo.

Distinción:

```text
Physical Security Projection
≠
Logical User Projection
```

---

# 55. Hidden security projection

Puede añadirse internamente y eliminarse antes de exposición.

---

# 56. Querying inaccessible fields

No deberá permitirse filtrar arbitrariamente por un campo protegido solo porque no se proyecta.

Ejemplo:

```text
WHERE salary > 100000
```

también puede revelar información.

---

# 57. Information inference

La seguridad debe considerar:

```text
SELECT
WHERE
ORDER BY
GROUP BY
HAVING
JOIN
COUNT
EXISTS
```

como posibles canales de información.

---

# 58. Field read ≠ field predicate permission

Puede existir:

```text
canFilter
```

separado de:

```text
canRead
```

---

# 59. Field sort permission

Ordenar por información sensible puede revelar ranking.

---

# 60. Field aggregate permission

Agregaciones pueden revelar información sin retornar valores individuales.

---

# 61. COUNT security

Ejemplo:

```text
COUNT(users WHERE medical_condition = X)
```

puede filtrar información sensible.

Por tanto:

```text
COUNT
≠
harmless READ
```

---

# 62. COUNT policy

Podrá requerir:

```text
resource access
predicate-field access
aggregation permission
minimum group size
```

según integración superior.

---

# 63. Aggregate access

Funciones como:

```text
SUM
AVG
MIN
MAX
COUNT
```

deberán evaluarse bajo Data Access Security.

---

# 64. Aggregate output ≠ unrestricted

Una policy puede permitir:

```text
COUNT
```

pero negar:

```text
SUM(salary)
```

---

# 65. GROUP BY leakage

Agrupar por campos sensibles puede revelar distribución.

---

# 66. HAVING leakage

Misma consideración.

---

# 67. EXISTS leakage

`EXISTS` revela al menos un bit de información.

Debe respetar scopes.

---

# 68. Relationship access

Una relación ORM deberá tratarse como recurso.

Ejemplo:

```php
$order->customer
```

---

# 69. RelationshipAccessRequest

```php
final readonly class RelationshipAccessRequest
{
    public function __construct(
        public DataPrincipal $principal,
        public RelationshipId $relationship,
        public DataOperation $operation,
        public DataAccessContext $context,
    ) {}
}
```

---

# 70. Relationship allowed ≠ target unrestricted

Permitir:

```text
Order.customer
```

no elimina las políticas de `Customer`.

---

# 71. Relationship effective security

Conceptualmente:

```text
RelationshipConstraint
AND
TargetEntityConstraint
AND
TenantConstraint
```

---

# 72. Lazy loading

Lazy Loading deberá pasar por la misma frontera.

Nunca:

```text
root authorized
→ lazy relation bypasses authorization
```

---

# 73. Eager loading

Misma regla.

```php
Order::with('customer')->get();
```

deberá aplicar:

```text
Order policy
+
relationship policy
+
Customer policy
```

---

# 74. Batch relationship loading

También deberá conservar restricciones.

---

# 75. N+1 optimization

Optimizar:

```text
100 authorized relationship queries
→ 1 batch query
```

no deberá cambiar la seguridad.

---

# 76. Security-preserving batching

```text
SecuritySemantics(before)
=
SecuritySemantics(after)
```

---

# 77. Polymorphic relationships

Cada target type deberá evaluarse.

No asumir:

```text
access to polymorphic relation
=
access to every target type
```

---

# 78. Repository integration

```php
$orderRepository->findAll();
```

deberá converger al mismo Data Access Security Engine.

---

# 79. Model API integration

```php
Order::all();
```

también.

---

# 80. EntityManager integration

```php
$em->find(Order::class, $id);
```

también.

---

# 81. Query Builder integration

```php
DB::table('orders')
```

deberá usar políticas de recursos físicos o requerir explicit trusted context.

---

# 82. ORM ≠ security bypass

Ninguna de las dos APIs ORM:

```text
Active Record-like
Data Mapper-like
```

deberá ser privilegiada.

---

# 83. find(id)

Incluso:

```php
Order::find(10);
```

debe convertirse conceptualmente en:

```text
WHERE id = 10
AND <security predicates>
```

---

# 84. Unauthorized entity identity

Cuando `find(10)` encuentra físicamente el registro pero el principal no puede verlo, el API no deberá revelar necesariamente su existencia.

---

# 85. Not found vs forbidden

La política podrá controlar si se diferencia:

```text
NOT_FOUND
FORBIDDEN
```

para evitar enumeration.

---

# 86. Default public behavior

Para lookup de recursos privados podrá favorecerse:

```text
NOT_VISIBLE
```

sin revelar existencia.

---

# 87. IdentityMap security

Este punto es crítico.

Si una Entity ya está en IdentityMap:

```text
Order#10
```

no significa que cualquier contexto de autorización pueda recibirla.

---

# 88. IdentityMap ≠ Authorization Cache

La presencia en IdentityMap no implica acceso.

---

# 89. Principal context stability

Idealmente EntityManager deberá operar dentro de un security context estable.

---

# 90. Security context drift

Cambiar principal dentro del mismo EntityManager scoped puede ser peligroso.

Por default:

```text
principal/security context change
→ new ORM scope
```

---

# 91. Persistent workers

Nunca compartir IdentityMap entre requests/principals.

Ya es una invariancia general del ORM y aquí se convierte además en requisito de seguridad.

---

# 92. Entity cache

Second-level Entity Cache tampoco implica autorización.

---

# 93. Cached entity state

Debe pasar por la misma decisión antes de exponerse.

---

# 94. Result Cache

Un cache hit no podrá omitir Data Access Security.

---

# 95. Security-aware cache key

Cuando el resultado dependa de scope:

```text
tenant
organization
principal class
policy
security predicate
```

las dimensiones necesarias deberán participar en la identidad semántica del cache.

---

# 96. Principal ID in cache key

No siempre será necesario.

Si muchos usuarios comparten exactamente la misma policy/scope, podrá usarse un:

```text
SecurityScopeFingerprint
```

---

# 97. SecurityScopeFingerprint

```php
final readonly class SecurityScopeFingerprint
{
    public function __construct(
        public string $value,
        public SecurityPolicyGeneration $generation,
    ) {}
}
```

---

# 98. Fingerprint ≠ authorization token

No deberá otorgar acceso por sí mismo.

---

# 99. Policy generation

Cuando cambien políticas:

```text
generation N
→ generation N+1
```

los caches dependientes podrán quedar incompatibles.

---

# 100. Cache invalidation

Authorization policy change puede requerir:

```text
security cache invalidation
result cache invalidation
```

según diseño.

---

# 101. Query Cache

El Query Cache estructural puede reutilizar Query Models, pero security constraints deberán formar parte de la compilación/instanciación efectiva.

---

# 102. Compiled Query Cache

No deberá incrustar accidentalmente:

```text
Tenant A
Principal A
```

como valores permanentes.

---

# 103. Parameterized security constraints

Preferir:

```text
tenant_id = :security_tenant
```

sobre constantes incrustadas.

---

# 104. Security parameter namespace

Los parámetros internos deberán tener namespace separado.

Ejemplo conceptual:

```text
__vs_security_tenant
```

---

# 105. Application parameter collision

La aplicación no podrá sobrescribir parámetros internos de seguridad.

---

# 106. Parameter provenance

```text
APPLICATION
SECURITY
FRAMEWORK
```

deberá ser conocida.

---

# 107. Pagination

Data Access Security deberá aplicarse **antes** de calcular la ventana.

Correcto:

```text
Authorized Dataset
      ↓
Pagination
```

Incorrecto:

```text
Global Dataset
      ↓
Pagination
      ↓
remove unauthorized rows
```

---

# 108. Pagination count

El total deberá contar únicamente recursos autorizados.

---

# 109. Count leakage

Incluso ese total podrá estar restringido por policy.

---

# 110. Cursor pagination

El cursor deberá estar ligado al security scope relevante.

---

# 111. Cursor security binding

Un cursor creado bajo:

```text
Tenant A
Security Scope X
```

no deberá reutilizarse bajo:

```text
Tenant B
Security Scope Y
```

---

# 112. Cursor fingerprint

Podrá incluir:

```text
SecurityScopeFingerprint
PolicyGeneration
```

---

# 113. Chunk processing

Chunk deberá operar sobre:

```text
Authorized Query
```

desde el inicio.

---

# 114. Chunk checkpoint

Un checkpoint deberá bindear el security scope cuando corresponda.

---

# 115. Resuming under changed policy

Si:

```text
checkpoint policy generation = 20
current policy generation = 21
```

el resume deberá:

```text
reject
re-plan
or explicitly re-authorize
```

según policy.

Nunca continuar ciegamente.

---

# 116. Lazy Collection

La creación diferida introduce un problema:

```text
create under Principal A
iterate under Principal B
```

---

# 117. Lazy security context

La Lazy Collection deberá capturar/bindear una especificación segura del contexto o exigir compatibilidad al iniciar iteración.

---

# 118. Lazy collection ≠ bearer capability

No deberá poder transportarse a otro request y otorgar acceso automáticamente.

---

# 119. Bulk Update

Ejemplo:

```php
Order::query()
    ->where('status', 'pending')
    ->bulkUpdate(['status' => 'cancelled']);
```

La operación deberá aplicar:

```text
row access
+
update operation permission
+
field update permission
```

---

# 120. Bulk Delete

Misma regla:

```text
row access
+
delete permission
```

---

# 121. Bulk operation danger

No deberá ocurrir:

```text
authorization checked against one representative row
→ update all rows
```

---

# 122. Security predicate in bulk mutation

Debe formar parte del mutation predicate efectivo.

```text
EffectiveMutationPredicate
=
ApplicationPredicate
AND
SecurityPredicate
```

---

# 123. Bulk affected-row count

Puede revelar información.

La policy podrá decidir si se expone el número exacto.

---

# 124. Bulk Insert

Deberá validar:

```text
create permission
field permissions
tenant ownership
immutable ownership fields
```

---

# 125. Ownership fields

La aplicación no deberá poder sobrescribir:

```text
tenant_id
organization_id
created_by
```

si son security-controlled.

---

# 126. Security-controlled fields

```php
enum FieldAuthority
{
    case APPLICATION;
    case SECURITY_CONTEXT;
    case FRAMEWORK;
    case DATABASE;
}
```

---

# 127. SECURITY_CONTEXT field

Ejemplo:

```text
tenant_id
```

podrá derivarse del contexto y no del input.

---

# 128. Import

Import deberá aplicar seguridad por:

```text
target resource
operation
fields
tenant
batch
```

---

# 129. Import ≠ trusted bypass

Un CSV importado no se vuelve trusted por venir de un administrador HTTP.

---

# 130. Export

Export es especialmente sensible.

```text
READ permission
≠
EXPORT permission
```

---

# 131. Export policy

Puede controlar:

```text
resources
fields
row scopes
maximum rows
sensitive fields
format
destination
```

en colaboración con otros subsistemas.

---

# 132. Export must not bypass row policy

Nunca:

```text
UI shows scoped data
Export downloads all database rows
```

---

# 133. Large Dataset Processing

Los pipelines de grandes datasets deberán conservar security scope durante toda su ejecución.

---

# 134. Long-running policy changes

Una operación larga puede comenzar bajo policy generation:

```text
42
```

y terminar cuando existe:

```text
43
```

---

# 135. Policy consistency strategy

Podrá definirse:

```php
enum DataAccessPolicyConsistency
{
    case PIN_GENERATION;
    case REVALIDATE_PER_BATCH;
    case REVALIDATE_ON_CHANGE;
    case CUSTOM;
}
```

---

# 136. PIN_GENERATION

Mantiene las reglas compiladas de una generación durante la operación, solo cuando policy lo permita.

---

# 137. REVALIDATE_PER_BATCH

Más seguro para trabajos largos y revocaciones rápidas.

---

# 138. Revocation sensitivity

Operaciones de alto riesgo podrán requerir revalidación frecuente.

---

# 139. Transactions

La autorización no debe confundirse con transaction isolation.

---

# 140. Transaction ≠ security scope

Una transacción puede ser consistente y aun así ejecutar acceso no autorizado.

---

# 141. Security context in transaction

Por default, el principal/security scope deberá permanecer estable durante una transacción.

---

# 142. Principal change mid-transaction

Deberá rechazarse salvo mecanismo explícito.

---

# 143. Optimistic locking

No sustituye autorización.

---

# 144. Pessimistic locking

Requiere permiso adicional cuando corresponda.

---

# 145. LOCK operation

Podrá modelarse separadamente.

---

# 146. Raw Query

Raw Query representa una de las principales escape hatches.

---

# 147. Raw Query ≠ automatic bypass

VoltStack deberá distinguir al menos:

```text
SAFE_RAW_EXPRESSION
TRUSTED_RAW_QUERY
UNSCOPED_RAW_QUERY
```

---

# 148. Safe raw expression

Una expresión dentro de Query AST puede seguir sometida a seguridad estructural.

---

# 149. Trusted raw query

Requerirá capability/permission explícita.

---

# 150. Unscoped raw query

En contextos protegidos deberá:

```text
DENY
```

por default.

---

# 151. Raw query security metadata

Podrá exigir:

```text
declared resources
declared operation
tenant/shard scope
security purpose
```

---

# 152. Example

```php
$database->trustedRaw(
    sql: '...',
    access: new RawQueryAccessDeclaration(
        operation: DataOperation::READ,
        resources: [Order::class],
    ),
);
```

Aun así, `trusted` no significa automáticamente `authorized`.

---

# 153. Internal framework queries

Algunas operaciones internas necesitan acceso especial:

```text
migration repository
metadata introspection
health checks
schema management
```

---

# 154. Internal purpose

Deberán utilizar:

```text
explicit system purpose
```

y no simplemente desactivar seguridad global.

---

# 155. System bypass

Si existe:

```text
SYSTEM_BYPASS
```

deberá ser:

```text
explicit
rare
non-user-controlled
auditable
purpose-bound
```

---

# 156. No boolean bypass flag

Evitar:

```php
$query->withoutSecurity(true);
```

como API pública casual.

---

# 157. Privileged capability

Preferir un objeto/capability interno no fabricable desde input.

---

# 158. SecurityCapability

```php
interface SecurityCapability
{
    public function purpose(): SecurityPurpose;
}
```

---

# 159. Capability ≠ serialized token

No deberá poder recrearse simplemente enviando un string desde HTTP.

---

# 160. Tenant isolation

Tenant scope deberá ser una restricción de primer nivel.

---

# 161. Tenant context ≠ ordinary filter

Aunque técnicamente pueda convertirse en predicate:

```text
tenant_id = ?
```

semánticamente deberá conservar origen:

```text
TENANT_ISOLATION
```

---

# 162. User predicate cannot disable tenant isolation

Nunca:

```php
->withoutGlobalScope('tenant')
```

como bypass casual equivalente.

---

# 163. Tenant-owned resources

Metadata podrá declarar:

```php
#[TenantScoped('tenant_id')]
final class Order {}
```

o equivalente en mapping metadata.

---

# 164. Tenant identifier

Será resuelto desde contexto confiable.

No desde input arbitrario.

---

# 165. Tenant route + query scope

Deberán concordar:

```text
Connection/Shard Tenant Context
=
Query Tenant Constraint
```

cuando el modelo lo requiera.

---

# 166. Tenant mismatch

```text
connection tenant A
query security tenant B
```

→ error.

---

# 167. Database-per-tenant

Aunque cada tenant tenga DB separada, Data Access Security sigue siendo útil.

---

# 168. Schema-per-tenant

Misma regla.

---

# 169. Row-per-tenant

El predicate es especialmente crítico.

---

# 170. Shard isolation

Shard routing deberá ocurrir de manera compatible con security scope.

---

# 171. Security predicate ≠ shard routing

Son sistemas distintos.

Pero pueden compartir evidencia:

```text
tenant
organization
partition key
```

---

# 172. Under-routing

No deberá utilizarse para “ocultar” datos como mecanismo de autorización.

---

# 173. Over-routing

No deberá consultar shards no autorizados si el security scope permite evitarlo.

---

# 174. Distributed queries

Cada subquery deberá conservar el scope.

---

# 175. Distributed merge

No podrá reintroducir resultados fuera de scope.

---

# 176. Partial shard failure

No deberá confundirse con:

```text
authorization denied
```

---

# 177. Native Row Level Security

PostgreSQL y otras plataformas pueden ofrecer controles nativos.

VoltStack deberá poder integrarlos mediante capabilities.

---

# 178. Native RLS ≠ application policy replacement

Puede actuar como segunda frontera.

---

# 179. RLS context

Si se utiliza session state:

```text
SET LOCAL app.tenant_id = ...
```

deberá integrarse con:

```text
Connection Security
Transaction Context
Data Access Security
```

---

# 180. RLS session leakage

Queda prohibido por las invariantes del documento 230.

---

# 181. Application predicates + RLS

Ambos pueden coexistir:

```text
VoltStack Security Predicate
+
Native RLS
```

---

# 182. Duplicate enforcement

Puede parecer redundante, pero es defense in depth.

---

# 183. Native policy capability

```php
interface NativeDataAccessCapabilities
{
    public function supportsRowLevelSecurity(): bool;

    public function supportsColumnPrivileges(): bool;

    public function supportsSessionSecurityContext(): bool;
}
```

---

# 184. MySQL/MariaDB/PostgreSQL/SQLite

No deberán tratarse como si ofrecieran exactamente las mismas primitivas.

---

# 185. Capability-driven integration

Nunca:

```php
if ($database === 'postgres') {
    // security everywhere
}
```

en componentes genéricos.

---

# 186. Authorization bridge

Contrato:

```php
interface DataAuthorizationBridge
{
    public function evaluate(
        DataAccessRequest $request,
    ): DataAccessDecision;
}
```

---

# 187. Framework Authorization

El bridge podrá adaptar:

```text
VoltStack Authorization DecisionEngine
```

a Data Access Security.

---

# 188. Database independent authorization core

`Quantum/Database` no deberá depender obligatoriamente de un sistema de roles específico.

---

# 189. No hard dependency on RBAC

Database no deberá asumir:

```text
Role → Permission
```

como único modelo.

Podrán utilizarse:

```text
RBAC
ABAC
ReBAC
policy engine
custom authorization
```

---

# 190. Policy composition

Podrán coexistir:

```text
Global Policy
Tenant Policy
Resource Policy
Field Policy
Relationship Policy
Operation Policy
Application Authorization Decision
```

---

# 191. Composition rule

Para restricciones obligatorias:

```text
EffectiveAccess
=
intersection(all mandatory allowed scopes)
```

---

# 192. DENY precedence

Por default:

```text
DENY
>
ALLOW
```

cuando ambas decisiones aplican a la misma dimensión.

---

# 193. UNKNOWN precedence

Para recurso protegido:

```text
UNKNOWN
→ DENY
```

---

# 194. Explicit override

Solo podrá existir en políticas con semántica formal, nunca como efecto accidental del orden de listeners.

---

# 195. Security policies ≠ event listeners

Una decisión crítica no deberá depender de:

```text
listener execution order
```

del Event System general.

---

# 196. Deterministic policy evaluation

La misma entrada y generación de policy deberá producir la misma decisión salvo dependencias declaradas.

---

# 197. Policy compilation

Políticas estructurales podrán compilarse.

```text
Policy Definitions
      ↓
Policy Compiler
      ↓
CompiledDataAccessPolicy
```

---

# 198. Compiled policy

Será immutable.

---

# 199. Runtime values

Valores como:

```text
principal ID
tenant ID
organization ID
```

se bindearán en runtime.

---

# 200. Policy cache

Podrá cachearse:

```text
compiled policy structure
```

pero no decisiones incorrectamente globalizadas.

---

# 201. Decision cache

Opcional.

Su key deberá contener todas las dimensiones relevantes.

---

# 202. Authorization cache danger

Nunca asumir:

```text
User can read Order
```

significa:

```text
User can read every Order
```

---

# 203. Decision evidence

Toda decisión deberá poder producir evidencia diagnóstica segura.

---

# 204. DecisionEvidence

```php
final readonly class DecisionEvidence
{
    public function __construct(
        public DecisionId $decisionId,
        public array $policyIds,
        public SecurityPolicyGeneration $generation,
        public DecisionReasonCode $reason,
    ) {}
}
```

---

# 205. Evidence ≠ sensitive policy dump

No deberá revelar expresiones internas sensibles innecesariamente.

---

# 206. Explainability

Diagnostics podrán decir:

```text
Access restricted by:
- tenant boundary
- organization boundary
- field policy
```

sin revelar datos privados.

---

# 207. Query security plan

Antes del Query Planner físico podrá existir:

```text
DataAccessSecurityPlan
```

---

# 208. Security plan contents

```php
final readonly class DataAccessSecurityPlan
{
    public function __construct(
        public DataAccessDecision $decision,
        public SecurityConstraintSet $constraints,
        public SecurityScopeFingerprint $scope,
        public SecurityPolicyGeneration $generation,
    ) {}
}
```

---

# 209. Security plan ≠ Query Plan

El primero representa autorización/restricciones.

El segundo representa ejecución.

---

# 210. Security plan lifecycle

```text
Query Model
   +
Security Context
      ↓
Security Planning
      ↓
Secured Query Model
      ↓
Semantic Validation
      ↓
Query Optimization
      ↓
Execution Planning
```

---

# 211. Secured Query Model

Deberá ser distinguible del query original.

---

# 212. Security validation

Antes de ejecución deberá comprobarse:

```text
mandatory constraints still present
field access valid
resource access valid
security context valid
```

---

# 213. Post-optimizer validation

Para mayor defensa:

```text
security validation
→ optimizer
→ security invariant validation
```

podrá utilizarse.

---

# 214. Compiler

El SQL Compiler no deberá decidir autorización.

---

# 215. Compiler responsibility

Solo compila el Query Model ya asegurado.

---

# 216. Executor

El Executor tampoco deberá inventar policies.

---

# 217. Executor guard

Sí podrá exigir:

```text
SecurityValidatedExecutionPlan
```

para operaciones protegidas.

---

# 218. SecurityValidatedExecutionPlan

Actúa como prueba estructural de que la frontera fue atravesada.

---

# 219. Type-state pattern

Podría modelarse:

```text
QueryModel
→ SecuredQueryModel
→ PlannedQuery
→ CompiledQuery
```

para reducir bypass accidental.

---

# 220. Internal trusted query path

Queries realmente internas podrán utilizar un tipo diferente:

```text
SystemQueryModel
```

con capability explícita.

---

# 221. Query extensions

Una extensión no deberá poder remover mandatory constraints.

---

# 222. Optimizer extensions

Misma regla.

---

# 223. Compiler extensions

No deberán reinterpretar seguridad.

---

# 224. Driver extensions

No deberán recibir autorización como responsabilidad.

---

# 225. Query event listeners

No podrán cancelar security predicates mediante eventos.

---

# 226. Event System

Eventos de observabilidad deberán ocurrir después de establecer decisión cuando sea apropiado.

---

# 227. Event payload security

No exponer:

```text
principal secrets
full policy state
sensitive predicates
```

innecesariamente.

---

# 228. Telemetry events

Ejemplos:

```text
DataAccessEvaluationStarted
DataAccessAllowed
DataAccessDenied
DataAccessRestricted
SecurityConstraintApplied
SecurityValidationFailed
PrivilegedDataAccessUsed
```

---

# 229. Per-row telemetry

Prohibida por default.

Podría producir cardinalidad y overhead enormes.

---

# 230. Metrics

Ejemplos:

```text
db.security.access.allowed
db.security.access.denied
db.security.access.restricted
db.security.field.denied
db.security.raw_query.denied
db.security.privileged_access
```

---

# 231. Metric labels

Bounded:

```text
operation
resource_type
decision
reason_class
```

Evitar:

```text
principal ID
tenant ID
record ID
raw field values
```

---

# 232. Audit integration

Accesos sensibles podrán generar audit records.

---

# 233. Audit-worthy operations

Ejemplos:

```text
EXPORT
BULK_DELETE
RAW_QUERY
SYSTEM_BYPASS
ADMINISTRATE
sensitive-field access
```

---

# 234. Audit ≠ Telemetry

```text
Telemetry
=
operational observation

Audit
=
security/accountability record
```

---

# 235. Query Audit

Será desarrollado específicamente en:

```text
233_DATABASE_QUERY_AUDIT_SYSTEM.md
```

---

# 236. Sensitive data

La clasificación y protección de datos sensibles pertenece al siguiente documento:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
```

Data Access Security deberá poder consumir esa clasificación.

---

# 237. Sensitive classification effect

Ejemplo:

```text
Field classification = SECRET
```

puede provocar:

```text
additional permission
no export
no aggregate
mandatory audit
```

---

# 238. Data masking

No deberá confundirse con autorización.

```text
Masking
≠
Permission
```

---

# 239. Masking after unauthorized access

No es suficiente si la aplicación no debía acceder al dato original.

---

# 240. Prefer DB-side protection

Cuando sea posible:

```text
do not fetch unauthorized field
```

en vez de:

```text
fetch secret
→ remove later
```

---

# 241. Write authorization

Reads no son la única preocupación.

---

# 242. Create

Deberá validar:

```text
resource create permission
field create permissions
ownership
tenant context
relationship references
```

---

# 243. Update

Deberá validar:

```text
target row scope
field mutation permissions
ownership invariants
relationship changes
```

---

# 244. Delete

Deberá validar target scope antes de eliminación.

---

# 245. Soft Delete

Soft delete sigue siendo:

```text
DELETE semantic operation
```

desde perspectiva de autorización.

---

# 246. Restore

Podrá ser una operación separada:

```text
RESTORE
```

en una extensión futura de `DataOperation`.

---

# 247. Relationship mutation

Operaciones como:

```text
attach
detach
associate
dissociate
sync
```

requieren autorización.

---

# 248. Relationship persistence

No basta con autorizar ambos objetos individualmente.

También puede requerirse permiso para modificar la relación.

---

# 249. Cascade operations

Un delete autorizado sobre parent no implica automáticamente autorización para cada cascade semántico de aplicación.

---

# 250. Database-native cascade

Si la DB ejecuta cascade, la policy deberá considerar sus efectos antes de iniciar la operación.

---

# 251. Persistence Engine

Antes de generar Persistence Plan deberá existir seguridad suficiente sobre los ChangeSets.

---

# 252. UnitOfWork

UoW detecta cambios.

No decide si están autorizados.

---

# 253. Flush

`flush()` deberá validar operaciones pendientes protegidas.

---

# 254. Authorization timing

Puede evaluarse:

```text
when mutation is registered
+
before flush
```

según policy.

---

# 255. Final pre-flush validation

Será necesaria para evitar:

```text
authorized entity
→ unauthorized field modified later
→ flush
```

---

# 256. ChangeSet security

```text
ChangeSet
      ↓
Data Mutation Security Analysis
      ↓
Authorized ChangeSet
      ↓
Persistence Planner
```

---

# 257. Field mutation analysis

Ejemplo:

```text
email: old → new
salary: old → new
```

puede producir:

```text
email = ALLOW
salary = DENY
```

→ operación rechazada.

---

# 258. Partial update authorization

VoltStack no deberá aplicar silenciosamente solo los campos autorizados y descartar los demás por default.

Preferir:

```text
fail entire semantic mutation
```

salvo API explícita.

---

# 259. Atomic authorization semantics

La aplicación deberá saber que la mutación solicitada no fue completamente autorizada.

---

# 260. Entity lifecycle hooks

No deberán poder añadir cambios protegidos después de la última validación sin nueva validación.

---

# 261. Persistence pipeline ordering

Conceptualmente:

```text
Change Detection
      ↓
Lifecycle Preparation
      ↓
Final ChangeSet
      ↓
Data Access Validation
      ↓
Persistence Planning
      ↓
Query Model
      ↓
Execution
```

---

# 262. Lifecycle mutation after validation

Si existe, deberá invalidar/repetir validación.

---

# 263. Hydration

Hydration no deberá otorgar acceso.

---

# 264. Hydrator assumption

El Hydrator consume resultados que ya atravesaron Data Access Security.

---

# 265. Partial entities

No deberán utilizarse para esconder campos como mecanismo principal de seguridad.

---

# 266. DTO projections

Son útiles para limitar exposición, pero siguen necesitando field access validation.

---

# 267. Serialization

Está fuera de Database.

Data Access Security protege la adquisición/mutación de datos, no sustituye serialization policies.

---

# 268. Security and errors

Jerarquía propuesta:

```text
DataAccessSecurityException
├── DataAccessDeniedException
├── DataAccessUnknownException
├── ResourceAccessDeniedException
├── FieldAccessDeniedException
├── RelationshipAccessDeniedException
├── AggregateAccessDeniedException
├── ExportAccessDeniedException
├── BulkAccessDeniedException
├── RawQueryAccessDeniedException
├── SecurityContextMismatchException
├── SecurityPolicyGenerationException
├── SecurityConstraintViolationException
├── SecurityScopeViolationException
└── PrivilegedAccessRequiredException
```

---

# 269. Public errors

No deberán revelar:

```text
record existence
hidden fields
policy internals
other tenant identifiers
```

innecesariamente.

---

# 270. Internal diagnostics

Podrán contener:

```text
DecisionId
PolicyIds
ConstraintIds
Operation
ResourceId
```

con redaction.

---

# 271. Persistent runtime architecture

El sistema deberá ser seguro bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 272. Request scope

Cada request tendrá su:

```text
DataAccessContext
Principal
SecurityScopeFingerprint
PolicyGeneration
```

---

# 273. Worker-global state

Solo podrá ser global:

```text
immutable compiled policy metadata
immutable resource metadata
policy definitions
```

cuando sean seguros.

---

# 274. Forbidden worker-global state

No:

```text
current principal
current tenant
current organization
current authorization decision
current security predicate
```

---

# 275. OpenSwoole

Context deberá ser coroutine-local.

---

# 276. RoadRunner

Reset obligatorio entre jobs/requests.

---

# 277. FrankenPHP

Reset obligatorio entre requests.

---

# 278. Long-running jobs

Deberán establecer security context explícitamente.

---

# 279. Queue job ≠ automatically trusted

Un job deserializado no obtiene acceso total por ejecutarse en background.

---

# 280. System jobs

Deberán declarar:

```text
SecurityPurpose
```

y capability cuando necesiten privilegios.

---

# 281. Testing architecture

Deberán existir pruebas para:

```text
row scopes
field scopes
relationships
eager loading
lazy loading
pagination
counts
cursor pagination
chunking
lazy collections
bulk update
bulk delete
imports
exports
raw queries
IdentityMap
caches
tenant isolation
shards
persistent workers
transactions
flush
policy changes
```

---

# 282. Cross-API conformance

Una misma policy deberá producir resultados equivalentes en:

```text
Model API
Repository
EntityManager
Query Builder
```

cuando representen la misma operación.

---

# 283. Example conformance test

```text
Principal A
can see Orders {1, 2, 3}
```

Entonces:

```php
Order::all();
```

```php
$repository->findAll();
```

y el Query Builder equivalente deberán observar:

```text
{1, 2, 3}
```

no conjuntos diferentes por bypass.

---

# 284. Relationship test

```text
Order authorized
Customer unauthorized
```

Verificar:

```text
lazy load
eager load
batch load
```

sin fuga.

---

# 285. Count test

Si solo 3 de 100 registros son visibles:

```text
pagination total
```

no deberá retornar `100` salvo permiso/policy explícita.

---

# 286. Bulk mutation test

```text
100 physical rows
10 authorized rows
```

Bulk Update deberá afectar como máximo las 10 permitidas bajo la semántica correspondiente.

---

# 287. Tenant isolation test

Generar datos para:

```text
Tenant A
Tenant B
```

y probar todas las APIs.

---

# 288. Cache isolation test

Un resultado cacheado por A nunca deberá aparecer para B por key incompleta.

---

# 289. IdentityMap isolation test

Una Entity cargada bajo A no deberá saltarse autorización al cambiar context.

Idealmente el cambio de contexto será rechazado antes.

---

# 290. Cursor security test

Cursor generado para A:

```text
A cursor
→ B context
```

deberá rechazarse.

---

# 291. Chunk resume test

Checkpoint de A no deberá reanudarse bajo B.

---

# 292. Policy generation test

Cambiar policy y verificar invalidación/revalidación.

---

# 293. Raw query test

Verificar que raw SQL no sea ruta de bypass.

---

# 294. Persistence test

Modificar campo protegido mediante:

```text
Model API
EntityManager
relationship mutation
bulk operation
```

y verificar DENY consistente.

---

# 295. Fuzzing de scopes

Puede generar combinaciones de:

```text
tenant
organization
principal
resource
operation
field
```

para buscar inconsistencias.

---

# 296. Property-based invariant

Una propiedad útil:

```text
No API may return a resource outside EffectiveAuthorizedScope.
```

---

# 297. Directory structure

```text
src/Quantum/Database/Security/DataAccess/
│
├── Contract/
│   ├── DataAccessSecurityManager.php
│   ├── DataAuthorizationBridge.php
│   ├── RowAccessPolicy.php
│   ├── FieldAccessPolicy.php
│   ├── RelationshipAccessPolicy.php
│   ├── AggregateAccessPolicy.php
│   └── NativeDataAccessCapabilities.php
│
├── Context/
│   ├── DataAccessContext.php
│   ├── DataAccessRequest.php
│   ├── FieldAccessRequest.php
│   ├── RelationshipAccessRequest.php
│   └── DataPrincipal.php
│
├── Operation/
│   ├── DataOperation.php
│   ├── SecurityPurpose.php
│   └── FieldAuthority.php
│
├── Resource/
│   ├── DataResource.php
│   ├── DataResourceId.php
│   ├── EntityResource.php
│   ├── TableResource.php
│   ├── FieldResource.php
│   ├── RelationshipResource.php
│   └── DatabaseResource.php
│
├── Decision/
│   ├── DataAccessDecision.php
│   ├── AccessDecisionEffect.php
│   ├── DecisionEvidence.php
│   ├── DecisionId.php
│   └── DecisionReasonCode.php
│
├── Constraint/
│   ├── SecurityConstraintSet.php
│   ├── SecurityConstraintId.php
│   ├── RowSecurityConstraint.php
│   ├── FieldSecurityConstraint.php
│   ├── RelationshipSecurityConstraint.php
│   └── OperationSecurityConstraint.php
│
├── Predicate/
│   ├── SecurityPredicate.php
│   ├── SecurityPredicateFactory.php
│   ├── SecurityValue.php
│   ├── SecurityPredicateOrigin.php
│   └── SecurityParameter.php
│
├── Scope/
│   ├── SecurityScope.php
│   ├── SecurityScopeFingerprint.php
│   ├── TenantSecurityScope.php
│   ├── ShardSecurityScope.php
│   └── OrganizationSecurityScope.php
│
├── Policy/
│   ├── DataAccessPolicy.php
│   ├── CompiledDataAccessPolicy.php
│   ├── DataAccessPolicyCompiler.php
│   ├── DataAccessPolicyRegistry.php
│   ├── DataAccessPolicyConsistency.php
│   └── SecurityPolicyGeneration.php
│
├── Planner/
│   ├── DataAccessSecurityPlanner.php
│   ├── DataAccessSecurityPlan.php
│   └── SecuredQueryModel.php
│
├── Validation/
│   ├── DataAccessValidator.php
│   ├── SecurityConstraintValidator.php
│   ├── FieldAccessValidator.php
│   ├── MutationAccessValidator.php
│   └── SecurityContextValidator.php
│
├── Persistence/
│   ├── ChangeSetSecurityAnalyzer.php
│   ├── PersistenceAccessValidator.php
│   └── RelationshipMutationAccessValidator.php
│
├── Raw/
│   ├── RawQueryAccessDeclaration.php
│   ├── RawQuerySecurityPolicy.php
│   └── PrivilegedRawQueryCapability.php
│
├── Native/
│   ├── NativeDataAccessAdapter.php
│   ├── RowLevelSecurityAdapter.php
│   └── NativeSecurityContextBinder.php
│
├── Cache/
│   ├── DataAccessDecisionCache.php
│   ├── SecurityPolicyCache.php
│   └── SecurityScopeCacheKey.php
│
├── Telemetry/
│   ├── DataAccessSecurityTelemetry.php
│   └── DataAccessSecurityEvent.php
│
├── Testing/
│   ├── DataAccessSecurityAssertions.php
│   ├── FakeDataAuthorizationBridge.php
│   ├── DataAccessConformanceSuite.php
│   └── DataAccessSecurityTestContext.php
│
└── Exception/
    ├── DataAccessSecurityException.php
    ├── DataAccessDeniedException.php
    ├── DataAccessUnknownException.php
    ├── ResourceAccessDeniedException.php
    ├── FieldAccessDeniedException.php
    ├── RelationshipAccessDeniedException.php
    ├── AggregateAccessDeniedException.php
    ├── ExportAccessDeniedException.php
    ├── BulkAccessDeniedException.php
    ├── RawQueryAccessDeniedException.php
    ├── SecurityContextMismatchException.php
    ├── SecurityConstraintViolationException.php
    └── PrivilegedAccessRequiredException.php
```

---

# 298. Flujo de lectura completo

```text
Application
    ↓
Model / Repository / Query Builder
    ↓
Query Model
    ↓
DataAccessRequest
    ↓
Authorization Bridge
    ↓
DataAccessDecision
    ↓
SecurityConstraintSet
    ↓
Security Predicate Injection
    ↓
Field/Relationship Validation
    ↓
Secured Query Model
    ↓
Semantic Engine
    ↓
Optimizer
    ↓
Security Invariant Validation
    ↓
Planner
    ↓
Compiler
    ↓
Executor
    ↓
Authorized Result
```

---

# 299. Flujo de escritura ORM

```text
Entity Mutation
      ↓
UnitOfWork
      ↓
ChangeSet
      ↓
Lifecycle Preparation
      ↓
Final ChangeSet
      ↓
Data Access Mutation Validation
      ↓
Authorized ChangeSet
      ↓
Persistence Planner
      ↓
Query Model
      ↓
Secured Mutation Query
      ↓
Compiler
      ↓
Executor
```

---

# 300. Flujo bulk

```text
Bulk Update Request
       ↓
Operation Authorization
       ↓
Field Mutation Authorization
       ↓
Row Security Predicate
       ↓
Application Predicate
       ↓
AND
       ↓
Secured Bulk Query
       ↓
Execution
```

---

# 301. Flujo de relación

```text
Order.customer
      ↓
Relationship Access Policy
      ↓
Target Resource Policy
      ↓
Tenant/Context Policy
      ↓
Effective Relationship Constraint
      ↓
Relationship Loader
      ↓
Query Engine
```

---

# 302. Architectural invariants

## DB-DACCESS-001
Data Access Security será distinta de Authentication.

## DB-DACCESS-002
Data Access Security será distinta de general Authorization.

## DB-DACCESS-003
Data Access Security será distinta de native DB permissions.

## DB-DACCESS-004
Database no implementará obligatoriamente RBAC.

## DB-DACCESS-005
Authorization Bridge permitirá integrar modelos externos.

## DB-DACCESS-006
Toda operación protegida atravesará security evaluation.

## DB-DACCESS-007
UNKNOWN no equivaldrá a ALLOW.

## DB-DACCESS-008
Protected UNKNOWN será DENY por default.

## DB-DACCESS-009
ALLOW podrá contener restricciones obligatorias.

## DB-DACCESS-010
Access Decision será distinta de Query Scope.

## DB-DACCESS-011
Security constraints serán estructuradas.

## DB-DACCESS-012
Security constraints no serán raw SQL por default.

## DB-DACCESS-013
Security values serán parameterized.

## DB-DACCESS-014
Application parameters no sobrescribirán security parameters.

## DB-DACCESS-015
Security predicate provenance será preservable.

## DB-DACCESS-016
Application predicates no debilitarán mandatory predicates.

## DB-DACCESS-017
Optimizer preservará mandatory security semantics.

## DB-DACCESS-018
Extensions no removerán mandatory constraints.

## DB-DACCESS-019
Row access será first-class.

## DB-DACCESS-020
Field access será first-class.

## DB-DACCESS-021
Relationship access será first-class.

## DB-DACCESS-022
Aggregate access será first-class.

## DB-DACCESS-023
COUNT será security-relevant.

## DB-DACCESS-024
EXISTS será security-relevant.

## DB-DACCESS-025
ORDER BY podrá ser security-relevant.

## DB-DACCESS-026
GROUP BY podrá ser security-relevant.

## DB-DACCESS-027
HAVING podrá ser security-relevant.

## DB-DACCESS-028
Filtering por campo protegido podrá requerir permiso.

## DB-DACCESS-029
Sorting por campo protegido podrá requerir permiso.

## DB-DACCESS-030
Hidden ORM field no será security boundary.

## DB-DACCESS-031
Serialization hidden field no será security boundary.

## DB-DACCESS-032
Physical security projection será distinta de logical projection.

## DB-DACCESS-033
Security-only projected fields no se expondrán accidentalmente.

## DB-DACCESS-034
Relationship authorization no implicará target unrestricted access.

## DB-DACCESS-035
Lazy Loading aplicará Data Access Security.

## DB-DACCESS-036
Eager Loading aplicará Data Access Security.

## DB-DACCESS-037
Batch Relation Loading aplicará Data Access Security.

## DB-DACCESS-038
N+1 optimization preservará security semantics.

## DB-DACCESS-039
Polymorphic targets se autorizarán por tipo.

## DB-DACCESS-040
Model API no será bypass.

## DB-DACCESS-041
Repository API no será bypass.

## DB-DACCESS-042
EntityManager no será bypass.

## DB-DACCESS-043
Query Builder no será bypass.

## DB-DACCESS-044
Active Record API y Data Mapper API convergerán al mismo security engine.

## DB-DACCESS-045
find(id) aplicará row security.

## DB-DACCESS-046
Entity existence podrá ocultarse según policy.

## DB-DACCESS-047
IdentityMap no será authorization cache.

## DB-DACCESS-048
IdentityMap hit no implicará access allowed.

## DB-DACCESS-049
Security context drift dentro del ORM scope será restringido.

## DB-DACCESS-050
Entity Cache no será authorization cache.

## DB-DACCESS-051
Result Cache no omitirá security evaluation.

## DB-DACCESS-052
Security-dependent cache keys incluirán scope suficiente.

## DB-DACCESS-053
SecurityScopeFingerprint no será bearer token.

## DB-DACCESS-054
Policy generation podrá invalidar caches.

## DB-DACCESS-055
Compiled Query Cache no fijará request-specific security values globalmente.

## DB-DACCESS-056
Pagination ocurrirá sobre authorized dataset.

## DB-DACCESS-057
Pagination total respetará authorized scope.

## DB-DACCESS-058
Pagination total podrá ser security-sensitive.

## DB-DACCESS-059
Cursor será security-scope bound cuando corresponda.

## DB-DACCESS-060
Cursor de Tenant A no será válido para Tenant B.

## DB-DACCESS-061
Chunk Processing operará sobre secured query.

## DB-DACCESS-062
Chunk checkpoint podrá bindear policy/security scope.

## DB-DACCESS-063
Resume no ignorará incompatible policy generation.

## DB-DACCESS-064
Lazy Collection no será bearer capability.

## DB-DACCESS-065
Lazy iteration validará security context compatibility.

## DB-DACCESS-066
Bulk Update aplicará row security.

## DB-DACCESS-067
Bulk Update aplicará field security.

## DB-DACCESS-068
Bulk Delete aplicará row security.

## DB-DACCESS-069
Bulk Insert aplicará create/field security.

## DB-DACCESS-070
Bulk authorization no se inferirá de una sola representative row.

## DB-DACCESS-071
Security predicate formará parte del bulk mutation predicate.

## DB-DACCESS-072
Affected-row count podrá ser security-sensitive.

## DB-DACCESS-073
Security-controlled fields no confiarán en client input.

## DB-DACCESS-074
Import no será security bypass.

## DB-DACCESS-075
Export requerirá policy explícita cuando corresponda.

## DB-DACCESS-076
READ no implicará EXPORT.

## DB-DACCESS-077
Export respetará row security.

## DB-DACCESS-078
Export respetará field security.

## DB-DACCESS-079
Large dataset processing preservará security scope.

## DB-DACCESS-080
Long-running operations tendrán policy consistency explícita.

## DB-DACCESS-081
Transaction no será security scope.

## DB-DACCESS-082
Principal cambiará mid-transaction solo explícitamente.

## DB-DACCESS-083
Optimistic locking no reemplazará authorization.

## DB-DACCESS-084
Pessimistic locking no reemplazará authorization.

## DB-DACCESS-085
LOCK podrá requerir permiso separado.

## DB-DACCESS-086
Raw Query no será automatic bypass.

## DB-DACCESS-087
Unscoped raw query será denegado en protected context por default.

## DB-DACCESS-088
Trusted raw query seguirá necesitando authorization.

## DB-DACCESS-089
Raw query podrá declarar resources/operation.

## DB-DACCESS-090
Internal queries usarán explicit security purpose.

## DB-DACCESS-091
System bypass será explicit capability.

## DB-DACCESS-092
System bypass no será boolean público casual.

## DB-DACCESS-093
Privileged capability no será derivable directamente de input.

## DB-DACCESS-094
Tenant isolation será mandatory security scope.

## DB-DACCESS-095
Tenant filter no será removable casual global scope.

## DB-DACCESS-096
Tenant ID vendrá de trusted context.

## DB-DACCESS-097
Tenant connection context y query context deberán concordar.

## DB-DACCESS-098
Database-per-tenant no eliminará necesidad de Data Access Security.

## DB-DACCESS-099
Schema-per-tenant no eliminará necesidad de Data Access Security.

## DB-DACCESS-100
Row-per-tenant tendrá mandatory tenant predicates.

## DB-DACCESS-101
Shard routing no sustituirá authorization.

## DB-DACCESS-102
Authorization no sustituirá shard routing.

## DB-DACCESS-103
Distributed subqueries conservarán security scope.

## DB-DACCESS-104
Distributed merge no reintroducirá unauthorized data.

## DB-DACCESS-105
Native RLS será optional defense in depth.

## DB-DACCESS-106
Native RLS no sustituirá necesariamente application policy.

## DB-DACCESS-107
RLS session state integrará Connection Security.

## DB-DACCESS-108
RLS context leakage estará prohibido.

## DB-DACCESS-109
Native security features serán capability-driven.

## DB-DACCESS-110
MySQL/MariaDB/PostgreSQL/SQLite no se asumirán equivalentes.

## DB-DACCESS-111
Policies podrán componerse.

## DB-DACCESS-112
Mandatory scopes se combinarán restrictivamente.

## DB-DACCESS-113
DENY tendrá precedencia por default.

## DB-DACCESS-114
Security policies no dependerán de arbitrary event listener order.

## DB-DACCESS-115
Policy evaluation será determinista bajo misma entrada/generación.

## DB-DACCESS-116
Compiled policy será immutable.

## DB-DACCESS-117
Runtime principal values no contaminarán shared compiled policies.

## DB-DACCESS-118
Decision caches tendrán keys completas.

## DB-DACCESS-119
DecisionEvidence será safe/redacted.

## DB-DACCESS-120
Security plan será distinto de Query Plan.

## DB-DACCESS-121
Secured Query Model será distinguible del original.

## DB-DACCESS-122
Security validation ocurrirá antes de execution.

## DB-DACCESS-123
Post-optimization security invariant validation será soportable.

## DB-DACCESS-124
Compiler no decidirá authorization.

## DB-DACCESS-125
Executor no inventará authorization policy.

## DB-DACCESS-126
Executor podrá exigir security-validated plan.

## DB-DACCESS-127
Type-state podrá reducir bypass accidental.

## DB-DACCESS-128
Query event listeners no removerán security constraints.

## DB-DACCESS-129
Telemetry no contendrá per-row principal-sensitive cardinality por default.

## DB-DACCESS-130
Audit y Telemetry permanecerán distintos.

## DB-DACCESS-131
Sensitive data classification podrá elevar access requirements.

## DB-DACCESS-132
Masking no sustituirá authorization.

## DB-DACCESS-133
Unauthorized sensitive fields no se obtendrán para ocultarlos después cuando pueda evitarse.

## DB-DACCESS-134
Create validará ownership/context fields.

## DB-DACCESS-135
Update validará final ChangeSet.

## DB-DACCESS-136
Delete validará target scope.

## DB-DACCESS-137
Soft Delete será semantic DELETE para authorization.

## DB-DACCESS-138
Relationship mutation requerirá autorización.

## DB-DACCESS-139
Cascade effects serán considerados.

## DB-DACCESS-140
UnitOfWork no decidirá authorization.

## DB-DACCESS-141
Flush validará pending protected mutations.

## DB-DACCESS-142
Final ChangeSet será validado antes de persistence planning.

## DB-DACCESS-143
Unauthorized fields no se descartarán silenciosamente por default.

## DB-DACCESS-144
Lifecycle changes posteriores invalidarán security validation previa.

## DB-DACCESS-145
Hydration no otorgará authorization.

## DB-DACCESS-146
DTO projection no sustituirá field security.

## DB-DACCESS-147
Serialization security será una capa diferente.

## DB-DACCESS-148
Public errors no revelarán hidden record existence innecesariamente.

## DB-DACCESS-149
Internal diagnostics usarán stable decision/policy IDs.

## DB-DACCESS-150
Current principal nunca será worker-global mutable state.

## DB-DACCESS-151
Current tenant nunca será worker-global mutable state.

## DB-DACCESS-152
Current security predicate nunca será worker-global mutable state.

## DB-DACCESS-153
FrankenPHP reseteará Data Access Context entre requests.

## DB-DACCESS-154
RoadRunner reseteará Data Access Context entre jobs/requests.

## DB-DACCESS-155
OpenSwoole usará coroutine-local Data Access Context.

## DB-DACCESS-156
Queue jobs establecerán security purpose/context explícitamente.

## DB-DACCESS-157
Background execution no implicará trusted execution.

## DB-DACCESS-158
Cross-API conformance será probado.

## DB-DACCESS-159
Tenant isolation será probado en todas las APIs principales.

## DB-DACCESS-160
Cache isolation será probado.

## DB-DACCESS-161
Cursor isolation será probado.

## DB-DACCESS-162
Checkpoint isolation será probado.

## DB-DACCESS-163
Raw Query bypass resistance será probado.

## DB-DACCESS-164
Persistence field security será probado.

## DB-DACCESS-165
Bulk security será probado.

## DB-DACCESS-166
Relationship loading security será probado.

## DB-DACCESS-167
Security decisions serán explainable.

## DB-DACCESS-168
Security explanations no revelarán protected data.

## DB-DACCESS-169
Security metadata será bounded.

## DB-DACCESS-170
Data Access Security será una frontera obligatoria, no una convención de aplicación.

## DB-DACCESS-171
Authorization checks no ocurrirán únicamente después de obtener datos.

## DB-DACCESS-172
Database security constraints se aplicarán antes de pagination/chunking cuando afecten cardinalidad.

## DB-DACCESS-173
Count Query y Data Query usarán security scope compatible.

## DB-DACCESS-174
Replica routing no cambiará authorization semantics.

## DB-DACCESS-175
Failover no cambiará authorization semantics.

## DB-DACCESS-176
Shard topology changes no eliminarán mandatory security scopes.

## DB-DACCESS-177
Security policy change será observable mediante generation.

## DB-DACCESS-178
UNKNOWN security context no se convertirá en anonymous allow implícito.

## DB-DACCESS-179
Anonymous access será una policy explícita.

## DB-DACCESS-180
Data access será fail-closed en recursos protegidos.

---

# 303. Modelo formal

Sea:

```text
P = principal
O = operation
R = resource
C = security context
Q = requested query
```

La evaluación produce:

```text
Evaluate(P,O,R,C)
=
(D,S)
```

donde:

```text
D = ALLOW | DENY | UNKNOWN
S = mandatory security constraint set
```

La ejecución solo será válida si:

```text
D = ALLOW
```

y:

```text
Qsecured
=
Q ∩ S
```

---

# 304. Scope efectivo

Sean:

```text
ST = tenant scope
SO = organization scope
SR = resource scope
SF = field scope
SP = principal policy scope
```

Entonces:

```text
EffectiveScope
=
ST
∩
SO
∩
SR
∩
SF
∩
SP
```

No:

```text
ST ∪ SO ∪ SR ...
```

para restricciones obligatorias.

---

# 305. Mutación efectiva

Para un Bulk Update:

```text
M(Q, Δ)
```

donde:

```text
Q = requested row predicate
Δ = requested field changes
```

la operación autorizada será:

```text
M(
    Q ∧ SecurityRowPredicate,
    Authorized(Δ)
)
```

pero si `Δ` contiene una mutación obligatoriamente denegada, el default será:

```text
DENY whole semantic mutation
```

no modificación parcial silenciosa.

---

# 306. Seguridad de lectura

Formalmente:

```text
VisibleRows
=
QueryRows
∩
AuthorizedRows
```

y:

```text
VisibleFields
⊆
AuthorizedFields
```

---

# 307. Seguridad de relaciones

Para relación:

```text
A → B
```

el target observable será:

```text
Visible(B)
=
RelationshipReachable(A,B)
∩
AuthorizedRelationship(A→B)
∩
AuthorizedRows(B)
∩
TenantScope(B)
```

---

# 308. Cache correctness

Un cache hit será usable solo si:

```text
CachedSecurityScope
compatible-with
CurrentSecurityScope
```

y:

```text
CachedPolicyGeneration
compatible-with
CurrentPolicyGeneration
```

Por tanto:

```text
CacheHit
≠
AuthorizedResult
```

---

# 309. Anti-patterns

## 309.1 Global ORM scopes as sole security system

Insuficiente si pueden removerse fácilmente.

---

## 309.2 Authorization only in controllers

Insuficiente.

---

## 309.3 Filtering unauthorized rows after query

Incorrecto.

---

## 309.4 Using `$hidden` as field security

Incorrecto.

---

## 309.5 Treating cache hit as authorized

Incorrecto.

---

## 309.6 Treating IdentityMap hit as authorized

Incorrecto.

---

## 309.7 Allowing raw SQL to bypass scopes

Incorrecto.

---

## 309.8 Applying security after pagination

Incorrecto.

---

## 309.9 Counting unauthorized rows

Puede provocar leakage.

---

## 309.10 Reusing cursors across security contexts

Incorrecto.

---

## 309.11 Treating tenant filter as optional query scope

Incorrecto cuando representa aislamiento de seguridad.

---

## 309.12 Using application input as tenant authority

Incorrecto.

---

## 309.13 Using admin DB credentials to solve authorization

Incorrecto.

---

## 309.14 Boolean `withoutSecurity()`

Demasiado peligroso como API casual.

---

## 309.15 Silently dropping unauthorized update fields

Oculta errores de seguridad y semántica.

---

# 310. Integración global de seguridad

Con este documento, el pipeline del Bloque 22 comienza a quedar:

```text
Untrusted Query/Input
        │
        ▼
Query Input Security
        │
        ▼
SQL Injection Prevention
        │
        ▼
Credential Security
        │
        ▼
Connection Security
        │
        ▼
Principal + Data Context
        │
        ▼
Data Access Security
        │
        ├── Resource Access
        ├── Row Access
        ├── Field Access
        ├── Relationship Access
        ├── Aggregate Access
        ├── Mutation Access
        └── Privileged Access
        │
        ▼
Secured Query Model
        │
        ▼
Query Engine
        │
        ▼
Compiler
        │
        ▼
Executor
        │
        ▼
Database
```

---

# 311. Relación con Authorization

Arquitectura recomendada:

```text
VoltStack Authentication
        ↓
Principal
        ↓
VoltStack Authorization
        ↓
Decision Engine
        ↓
Database Authorization Bridge
        ↓
Data Access Security
        ↓
Security Constraints
```

Database no deberá conocer:

```text
UI permissions
routes
menus
HTTP middleware
```

Solo las decisiones relevantes para acceso a datos.

---

# 312. Relación con Connection Security

```text
Connection Security
=
Is this database channel/session trustworthy?

Data Access Security
=
What may this context do through that channel?
```

Ambas deben cumplirse:

```text
SecureConnection
AND
AuthorizedOperation
```

---

# 313. Relación con Sensitive Data Protection

El siguiente documento añadirá:

```text
What kind of data is this?
How sensitive is it?
How should it be protected?
```

De forma que:

```text
Data Classification
      ↓
Sensitive Data Policy
      ↓
Data Access Security Requirements
      ↓
Query/Result Protection
```

---

# 314. Regla maestra final

> **En VoltStack, ninguna API que pueda leer, inferir, modificar, eliminar, relacionar, agregar, exportar o procesar datos deberá convertirse en una ruta alternativa para omitir la autorización. Las restricciones de seguridad deberán convertirse en parte estructural del Query/Persistence Model antes de la ejecución y conservarse durante optimización, paginación, relaciones, caching, distribución y procesamiento masivo.**

En forma compacta:

```text
Authenticate the principal.
Authorize the operation.
Constrain the resource.
Constrain the rows.
Constrain the fields.
Constrain the relationships.
Preserve the constraints.
Validate before execution.
Audit privileged access.
Fail closed.
```

La propiedad fundamental será:

```text
ExecutedDataOperation
⊆
AuthorizedDataOperation
```

y nunca:

```text
ExecutedDataOperation
⊃
AuthorizedDataOperation
```

---

# 315. Estado del Bloque 22

```text
BLOCK 22 — SECURITY

✓ 226_DATABASE_SECURITY_ARCHITECTURE.md
✓ 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
✓ 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
✓ 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
✓ 230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
✓ 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
○ 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
○ 233_DATABASE_QUERY_AUDIT_SYSTEM.md
○ 234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 316. Siguiente documento

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura de clasificación y protección de información sensible dentro de VoltStack Database:

```text
Data Classification
Public
Internal
Confidential
Restricted
Secret

Sensitive Field Metadata
PII classification
financial data
credentials/secrets
security tokens
regulated data

data-at-rest protection
data-in-transit requirements
field-level encryption
application-level encryption
database-native encryption
encryption context
key references
key rotation
ciphertext envelopes
deterministic vs randomized encryption
searchable encrypted fields
hashing/tokenization
masking
redaction
debug protection
telemetry protection
cache protection
export restrictions
backup protection
replica protection
temporary data
query parameter redaction
result redaction
ORM integration
hydration/dehydration
bulk operations
import/export
tenant isolation
key separation
persistent runtime safety
audit integration
```

bajo una separación fundamental:

```text
Authorization
≠
Encryption
≠
Masking
≠
Redaction
≠
Classification
```

y la regla:

> **VoltStack deberá conocer la sensibilidad semántica de los datos suficientemente pronto para impedir que información protegida termine accidentalmente en queries observables, logs, telemetry, caches, exports, exceptions, debug tools, backups o contextos que no satisfagan su política de protección.**