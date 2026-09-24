# 319_DATABASE_AUTHORIZATION_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Authorization Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 319 — Database Authorization Integration System  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `318_DATABASE_AUTHENTICATION_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `320_DATABASE_JOB_AND_QUEUE_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de integración entre:

```text
VoltStack/Quantum/Database
```

y el subsistema:

```text
VoltStack Authorization
```

incluyendo:

- Policies;
- Gates;
- Voters;
- permission evaluators;
- ownership rules;
- resource authorization;
- query scopes;
- tenant-aware authorization;
- bulk authorization;
- autorización de operaciones ORM;
- autorización de consultas;
- autorización de mutaciones;
- propagación de contexto;
- consistencia;
- transacciones;
- auditoría;
- telemetry;
- persistent runtimes.

La regla fundamental será:

> **Authorization decide si un actor puede realizar una acción sobre un recurso dentro de un contexto; Database proporciona mecanismos para localizar, consultar, bloquear y modificar datos, pero no convierte la accesibilidad técnica de un registro en permiso de acceso.**

Por tanto:

```text
Database
≠
Authorization
```

y:

```text
Record Exists
≠
Access Granted
```

---

# 2. Principio arquitectónico central

Debe mantenerse permanentemente:

```text
Authentication
≠
Authorization
≠
Database
```

Authentication responde principalmente:

```text
Who is the actor?
```

Authorization responde:

```text
May this actor perform this action
on this resource
under this context?
```

Database responde:

```text
How is the required persistent state
located, represented, read and modified?
```

---

# 3. Modelo conceptual

Una decisión de autorización puede expresarse como:

```text
AuthorizationDecision =
f(
    Actor,
    Action,
    Resource,
    Context,
    Policy,
    RelevantState
)
```

Database puede proporcionar:

```text
RelevantState
```

pero no deberá decidir por sí mismo:

```text
ALLOW
DENY
```

salvo enforcement técnico explícitamente delegado por Authorization.

---

# 4. Distinciones fundamentales

VoltStack deberá distinguir:

```text
Authentication
≠
Authorization
```

```text
Authorization
≠
Validation
```

```text
Authorization
≠
Database Constraint
```

```text
Authorization
≠
Query Scope
```

```text
Authorization
≠
Tenant Isolation
```

```text
Authorization
≠
Ownership
```

```text
Authorization
≠
Row-Level Security
```

```text
Authorization
≠
Data Filtering
```

```text
Authorization Check
≠
Authorization Enforcement
```

---

# 5. Arquitectura general

```text
HTTP / CLI / Job / API
          │
          ▼
     Authentication
          │
          ▼
   Authenticated Actor
          │
          ▼
      Authorization
          │
    ┌─────┴──────────┐
    │                │
    ▼                ▼
Policy Engine   Authorization Query Policy
    │                │
    └──────┬─────────┘
           ▼
Database Authorization Bridge
           │
    ┌──────┴──────────────┐
    ▼                     ▼
Resource Resolver   Query Constraint Contributor
    │                     │
    └──────────┬──────────┘
               ▼
          Database API
               │
       ┌───────┴────────┐
       ▼                ▼
   Query Engine         ORM
       │                │
       └───────┬────────┘
               ▼
       Persistence Engine
               │
               ▼
       Execution Engine
               │
               ▼
          Connection
               │
               ▼
             Driver
```

---

# 6. Dependency direction

La integración deberá respetar:

```text
Authorization
      ↓
Authorization Contracts
      ↑
Database Authorization Integration
      ↓
Database
```

Database Core no deberá depender del núcleo de Authorization.

---

# 7. Database Core independence

No deberá existir:

```text
QueryBuilder
→ Gate
```

ni:

```text
Driver
→ Policy
```

ni:

```text
SQLCompiler
→ AuthorizationManager
```

ni:

```text
Connection
→ Voter
```

ni:

```text
Hydrator
→ PermissionChecker
```

---

# 8. Authorization integration layer

Se propone:

```text
VoltStack\Quantum\Database\Integration\Authorization
```

como frontera entre ambos sistemas.

---

# 9. Responsabilidades de Authorization

Authorization será responsable de:

- interpretar actor;
- interpretar acción;
- interpretar recurso;
- resolver policies;
- ejecutar voters;
- aplicar reglas;
- combinar decisiones;
- evaluar ownership semántico;
- evaluar roles/permissions cuando correspondan;
- determinar ALLOW/DENY;
- explicar decisiones;
- producir metadata de enforcement;
- emitir eventos de autorización.

---

# 10. Responsabilidades de Database

Database será responsable de:

- recuperar datos necesarios;
- ejecutar queries;
- aplicar predicates recibidos;
- administrar transacciones;
- controlar locking;
- manejar concurrencia;
- respetar tenant/shard;
- garantizar constraints;
- reportar errores;
- proporcionar evidencia de persistencia;
- preservar seguridad Database.

---

# 11. Responsabilidades de la integración

La integración deberá:

- proporcionar recursos a Authorization;
- proyectar datos relevantes;
- traducir constraints de autorización a Query Model cuando sea seguro;
- aplicar scopes estructurados;
- proteger mutaciones;
- coordinar authorization + transaction;
- manejar consistencia;
- evitar bypasses;
- detectar operaciones bulk;
- propagar contexto;
- producir telemetry;
- permitir auditoría.

---

# 12. Actor model

Authorization operará sobre:

```text
AuthorizationActor
```

que podrá representar:

```text
User
ServiceAccount
API Client
System Actor
Job Actor
Administrator
Anonymous Actor
```

---

# 13. Actor ≠ ORM Entity

Un actor puede tener una entidad ORM asociada.

Pero:

```text
AuthorizationActor
≠
User Entity
```

---

# 14. Resource model

Authorization podrá evaluar:

```text
Resource
```

como:

```text
Entity
ResourceReference
ResourceProjection
ResourceDescriptor
ResourceCollection
ResourceType
```

---

# 15. Resource ≠ Database Row

Un recurso de autorización puede representar:

```text
Project
Invoice
Document
Tenant
Account
Server
```

aunque internamente se persista mediante múltiples tablas.

---

# 16. Action model

Ejemplos:

```text
VIEW
CREATE
UPDATE
DELETE
EXPORT
APPROVE
PUBLISH
ARCHIVE
RESTORE
MANAGE
```

Las acciones pertenecen a Authorization, no a SQL.

---

# 17. SQL operation ≠ authorization action

Por ejemplo:

```text
UPDATE
```

SQL puede corresponder a:

```text
approve invoice
archive document
change owner
reset status
```

Por tanto:

```text
SQL UPDATE
≠
Authorization UPDATE
```

---

# 18. AuthorizationContext

Se propone:

```text
AuthorizationContext
├── Actor
├── Action
├── Resource
├── TenantContext?
├── ShardContext?
├── AuthenticationContext?
├── RequestContext?
├── OperationId
├── AuthorizationAttemptId
├── PolicyContext
└── SecurityContext
```

---

# 19. DatabaseAuthorizationContext

La integración podrá derivar:

```text
DatabaseAuthorizationContext
├── AuthorizationAttemptId
├── ActorReference
├── Action
├── ResourceType
├── TenantContext?
├── ShardContext?
├── ConsistencyRequirement
├── TransactionContext?
├── EnforcementMode
└── AuditContext
```

---

# 20. Scoped context

Este contexto deberá ser:

```text
operation-scoped
```

Nunca:

```text
global mutable state
```

---

# 21. Authorization modes

La integración podrá distinguir:

```text
CHECK
FILTER
ENFORCE
AUDIT
```

---

# 22. CHECK

Evalúa:

```text
Can actor perform action on resource?
```

---

# 23. FILTER

Restringe un conjunto:

```text
Which resources may actor access?
```

---

# 24. ENFORCE

Garantiza que una operación persistente sólo afecte recursos autorizados.

---

# 25. AUDIT

Registra evidencia relevante de una decisión u operación.

---

# 26. Check ≠ Filter

No deberá asumirse:

```text
canView(resource)
```

equivalente a:

```text
query only viewable resources
```

---

# 27. Filter ≠ Enforcement

Una query filtrada puede reducir resultados.

Pero una mutación crítica puede necesitar enforcement adicional.

---

# 28. Authorization result

Se propone:

```text
AuthorizationDecision
├── ALLOW
├── DENY
└── ABSTAIN
```

según arquitectura del sistema Authorization.

---

# 29. Unknown state

Cuando la evidencia requerida no pueda obtenerse:

```text
UNKNOWN
```

no deberá transformarse automáticamente en:

```text
ALLOW
```

---

# 30. Fail closed

Para operaciones sensibles:

```text
UnableToDetermineAuthorization
→
DENY / ERROR
```

según policy.

---

# 31. Resource resolution

Authorization puede requerir cargar un recurso antes de decidir.

Ejemplo:

```text
PATCH /projects/42
```

requiere:

```text
Project #42
```

---

# 32. Naive flow

```text
SELECT project
      ↓
authorize(project)
      ↓
UPDATE project
```

puede introducir:

```text
TOCTOU
```

---

# 33. TOCTOU

TOCTOU:

```text
Time Of Check
≠
Time Of Use
```

Ejemplo:

```text
T1:
project.owner_id = Alice

T2:
Alice authorized

T3:
owner changes to Bob

T4:
Alice updates project
```

La decisión de T2 puede ya no ser válida.

---

# 34. Central TOCTOU rule

> **Una autorización basada en estado mutable no deberá considerarse automáticamente válida para una mutación posterior si dicho estado puede cambiar entre la comprobación y el uso.**

---

# 35. Authorization consistency

Debe distinguirse:

```text
Policy Evaluation Consistency
```

de:

```text
Database Transaction Isolation
```

---

# 36. Strategies against TOCTOU

Podrán utilizarse:

```text
transactional authorization
conditional mutation
optimistic versioning
pessimistic locking
authorization predicate in mutation
immutable authorization facts
database constraints
```

según caso.

---

# 37. Transactional authorization

```text
BEGIN
  load resource
  authorize
  mutate
COMMIT
```

reduce ciertas races.

Pero no elimina automáticamente todas.

---

# 38. Isolation matters

Con:

```text
READ COMMITTED
```

una segunda lectura puede observar otro estado.

Con niveles superiores existen otras garantías/costos.

Authorization no deberá asumir aislamiento no demostrado.

---

# 39. Conditional mutation

Una estrategia fuerte puede ser:

```text
UPDATE projects
SET ...
WHERE id = ?
AND owner_id = ?
```

cuando:

```text
owner_id
```

forma parte directa del predicate autorizado.

---

# 40. Authorization-aware mutation

Conceptualmente:

```text
AuthorizedMutation
=
Mutation
+
AuthorizationConstraint
```

---

# 41. Authorization constraint

No será SQL.

Será una representación semántica.

Ejemplo:

```text
OwnerEquals(CurrentActorId)
```

que podrá convertirse a Query AST.

---

# 42. Query constraint contributor

Se propone:

```php
interface AuthorizationQueryConstraintContributor
{
    public function contribute(
        AuthorizationQueryRequest $request
    ): AuthorizationQueryConstraint;
}
```

---

# 43. Semantic constraint

Ejemplo:

```text
AuthorizationConstraint
└── AND
    ├── tenant_id = CurrentTenant
    └── owner_id = CurrentActor
```

---

# 44. Constraint ≠ policy

El constraint es:

```text
enforcement representation
```

No reemplaza:

```text
Policy
```

---

# 45. Policy may not be query-translatable

Una policy puede depender de:

- external service;
- current time;
- risk engine;
- complex graph;
- dynamic organization hierarchy;
- business computation.

No todo puede traducirse a SQL.

---

# 46. Query translatability

Se deberá modelar explícitamente:

```text
TRANSLATABLE
PARTIALLY_TRANSLATABLE
NON_TRANSLATABLE
UNKNOWN
```

---

# 47. UNKNOWN ≠ translatable

Nunca:

```text
UNKNOWN
→
generate approximate predicate
```

para decisiones de seguridad.

---

# 48. Partial translation

Puede utilizarse:

```text
coarse database filtering
+
fine-grained policy evaluation
```

---

# 49. Example

Database filtra:

```text
tenant_id = 10
```

y Authorization posteriormente evalúa:

```text
canView(document)
```

sobre cada candidato.

---

# 50. Over-fetch concern

Este patrón puede ser correcto pero costoso.

Deberá existir:

- pagination awareness;
- resource budgets;
- bounded candidates;
- telemetry.

---

# 51. Under-filtering

Nunca deberá provocar exposición de recursos no autorizados al cliente antes de evaluación final.

---

# 52. Query authorization pipeline

```text
Application Query
      ↓
Authorization Query Request
      ↓
Policy Resolution
      ↓
Constraint Contribution
      ↓
Query Model
      ↓
Authorization Constraint Merge
      ↓
Semantic Validation
      ↓
Optimizer
      ↓
Planner
      ↓
Compiler
      ↓
Executor
```

---

# 53. Query Builder integration

Podrá ofrecerse una API como:

```php
Project::query()
    ->authorizedFor($actor, 'view')
    ->get();
```

---

# 54. API convenience

También:

```php
$db->table('projects')
    ->authorized($authorizationContext)
    ->get();
```

pero internamente deberá converger al mismo sistema.

---

# 55. authorized() ≠ magic SQL

El método deberá producir:

```text
Authorization Query Intent
```

no concatenar:

```text
WHERE ...
```

manualmente.

---

# 56. Repository integration

Ejemplo:

```php
$projects = $repository->findAuthorizedFor(
    actor: $actor,
    action: ProjectAction::VIEW,
);
```

---

# 57. Repository ≠ Authorization Engine

El repository puede coordinar la integración.

No deberá implementar policies internamente.

---

# 58. Model API integration

Podrá existir:

```php
Project::authorizedFor($actor, 'view')->get();
```

como API de conveniencia.

---

# 59. Model static state prohibited

Nunca:

```php
Project::$currentActor = $actor;
```

---

# 60. Current actor resolution

Si se ofrece:

```php
Project::authorized()->get();
```

el actor deberá resolverse mediante un:

```text
scoped AuthorizationContextResolver
```

---

# 61. Explicit API preferred internally

Internamente será preferible:

```php
->authorizedFor($actor, $action)
```

para mantener claridad.

---

# 62. EntityManager integration

El EntityManager podrá cooperar con Authorization durante:

```text
persist
remove
flush
```

pero no deberá convertirse en Policy Engine.

---

# 63. Persist ≠ authorization

```php
$em->persist($entity);
```

sólo registra intención de persistencia.

No implica:

```text
actor authorized
```

---

# 64. Flush ≠ authorization

Igualmente:

```text
flush()
```

no deberá asumir que toda mutación está autorizada salvo que la aplicación haya configurado enforcement automático.

---

# 65. Authorization enforcement strategies

Se podrán soportar:

```text
EXPLICIT
INTERCEPTOR
PERSISTENCE_POLICY
QUERY_CONSTRAINT
DATABASE_NATIVE
COMPOSITE
```

---

# 66. EXPLICIT

La aplicación ejecuta:

```php
$authorization->require(
    $actor,
    'update',
    $project,
);

$project->rename(...);

$em->flush();
```

---

# 67. INTERCEPTOR

Controller/Action interceptor puede autorizar antes de ejecutar.

---

# 68. PERSISTENCE_POLICY

Persistence Engine puede requerir evidencia previa para ciertas operaciones.

---

# 69. QUERY_CONSTRAINT

La mutación incluye constraint de autorización.

---

# 70. DATABASE_NATIVE

Puede utilizar:

```text
Row-Level Security
database roles
views
stored policies
```

cuando la plataforma lo permita.

---

# 71. COMPOSITE

Combina varias capas.

Esta será normalmente la estrategia más robusta para sistemas sensibles.

---

# 72. Defense in depth

VoltStack deberá favorecer:

```text
Application Authorization
+
Query Enforcement
+
Tenant Isolation
+
Database Constraints
+
DB Least Privilege
```

cuando corresponda.

---

# 73. No duplicate semantic authority

Defense in depth no significa mantener reglas empresariales contradictorias en cinco lugares.

Debe existir una fuente semántica clara.

---

# 74. Ownership

Ownership es un patrón frecuente:

```text
resource.owner_id == actor.id
```

pero:

```text
Ownership
≠
Authorization
```

---

# 75. Example

Un owner puede:

```text
view
edit
```

pero quizá no:

```text
approve
delete
transfer
```

---

# 76. Ownership metadata

Podrá declararse:

```text
OwnershipDescriptor
├── ResourceType
├── OwnerReference
├── OwnershipKind
└── QueryMapping
```

---

# 77. Ownership query optimization

Una policy:

```text
owner can view
```

puede traducirse eficientemente a:

```text
owner_id = actor.id
```

si está declarada como query-translatable.

---

# 78. Ownership change

Cambiar:

```text
owner_id
```

es una operación especialmente sensible porque puede cambiar simultáneamente autorización futura.

---

# 79. Ownership transfer

Deberá poder exigir:

```text
authorization to transfer
+
valid target owner
+
transaction
+
audit
```

---

# 80. Tenant isolation vs authorization

Tenant isolation responde:

```text
Which tenant's data can this execution context address?
```

Authorization responde:

```text
Within that allowed tenant scope,
what may this actor do?
```

---

# 81. Tenant isolation first

Idealmente:

```text
Tenant Scope
      ↓
Authorization Scope
      ↓
Application Query
```

---

# 82. Tenant filter is not permission

```text
tenant_id = 10
```

sólo demuestra pertenencia al tenant scope.

No demuestra:

```text
actor may view every row in tenant 10
```

---

# 83. Tenant-aware authorization

El contexto deberá incluir:

```text
TenantId
```

cuando las policies dependan de él.

---

# 84. Cross-tenant authorization

Deberá ser:

```text
explicit
privileged
audited
```

---

# 85. Tenant scope cannot be removed accidentally

Un authorization scope no deberá eliminar:

```text
TenantConstraint
```

existente.

---

# 86. Constraint composition

Conceptualmente:

```text
FinalPredicate =
TenantConstraint
AND
AuthorizationConstraint
AND
ApplicationPredicate
```

---

# 87. Constraint precedence

No deberá existir una operación ordinaria que haga:

```text
AuthorizationConstraint
OR
TenantConstraint
```

y accidentalmente amplíe el acceso.

---

# 88. Trusted constraint composition

Los predicates de seguridad deberán marcarse como:

```text
MANDATORY
NON_REMOVABLE
```

dentro del Query Model cuando corresponda.

---

# 89. Security predicate

Se propone un concepto:

```text
SecurityConstraint
```

con metadata:

```text
source
purpose
mandatory
scope
policy id
tenant binding
actor binding
```

---

# 90. Optimizer restrictions

El Query Optimizer podrá simplificar predicates.

Pero nunca deberá eliminar una restricción de seguridad salvo equivalencia semántica demostrable.

---

# 91. Optimization correctness

```text
Optimized Query
```

deberá preservar:

```text
Authorization Semantics
```

exactamente.

---

# 92. Compiler independence

SQL Compiler recibe el Query AST final.

No ejecuta policies.

---

# 93. Driver independence

Driver nunca conoce:

```text
actor
role
permission
policy
```

salvo metadata técnica requerida por una integración nativa específica.

---

# 94. Row-Level Security

PostgreSQL y otras plataformas pueden proporcionar mecanismos nativos de seguridad a nivel de fila.

VoltStack podrá integrarlos opcionalmente.

---

# 95. RLS ≠ Authorization Engine

RLS puede ser una capa de enforcement.

No reemplaza automáticamente:

```text
VoltStack Authorization
```

---

# 96. Database capability

El uso de RLS deberá depender de:

```text
Capability Discovery
```

no:

```php
if ($platform === 'postgresql') {
}
```

como regla arquitectónica principal.

---

# 97. Native authorization capability

Podrá modelarse:

```text
ROW_LEVEL_SECURITY
SESSION_SECURITY_CONTEXT
DATABASE_ROLES
SECURITY_BARRIER_VIEWS
```

---

# 98. Capability UNKNOWN

Si:

```text
ROW_LEVEL_SECURITY = UNKNOWN
```

no deberá asumirse:

```text
SUPPORTED
```

---

# 99. Session variables

Algunas estrategias pueden establecer:

```text
current_actor_id
current_tenant_id
```

en la sesión Database.

---

# 100. Persistent connection danger

En FrankenPHP:

```text
Request A
actor = Alice

connection reused

Request B
actor = Bob
```

podría heredar accidentalmente:

```text
current_actor_id = Alice
```

si el reset falla.

---

# 101. Native context reset

Cualquier contexto Authorization establecido en una conexión deberá:

```text
set
verify
reset
verify
```

según política.

---

# 102. Reset failure

Si el estado no puede demostrarse limpio:

```text
Connection
→
DISCARD
```

---

# 103. Database role switching

Si se utiliza:

```text
SET ROLE
```

o equivalente:

```text
Role State
```

deberá formar parte del Connection Reset Contract.

---

# 104. Role ≠ app permission

Un database role puede ser mecanismo de defensa.

No deberá confundirse automáticamente con:

```text
Application Role
```

---

# 105. Authorization projections

Para evaluar una policy no siempre es necesario hidratar la entidad completa.

---

# 106. Example

Una policy puede requerir:

```text
project.id
project.owner_id
project.status
project.tenant_id
```

únicamente.

---

# 107. AuthorizationProjection

Se propone:

```text
AuthorizationProjection
├── ResourceId
├── ResourceType
├── OwnershipState
├── SecurityState
├── TenantReference
├── Version
└── PolicyAttributes
```

---

# 108. Projection ≠ entity

No deberá insertarse en IdentityMap como entidad completa.

---

# 109. Projection benefits

Reduce:

- I/O;
- hydration;
- memory;
- accidental data exposure;
- lazy loading;
- unnecessary PII retrieval.

---

# 110. Policy data minimization

Authorization sólo deberá solicitar:

```text
minimum data required
```

para tomar la decisión.

---

# 111. Authorization data provider

Se podrá definir:

```php
interface AuthorizationResourceDataProvider
{
    public function resolve(
        ResourceReference $resource,
        AuthorizationContext $context,
    ): AuthorizationProjection;
}
```

---

# 112. Resource reference

Ejemplo:

```text
ResourceReference
├── Type = Project
└── Id = 42
```

---

# 113. Reference ≠ authorization

Conocer:

```text
Project#42
```

no significa que el actor pueda accederlo.

---

# 114. Existence hiding

Algunas policies requieren que:

```text
unauthorized resource
```

se comporte externamente como:

```text
not found
```

---

# 115. Internal classification

Internamente deberá distinguirse:

```text
NOT_FOUND
```

de:

```text
FOUND_BUT_FORBIDDEN
```

cuando sea necesario para seguridad/auditoría.

---

# 116. External mapping

HTTP podrá convertir ambos en:

```text
404
```

según policy.

Database no decide esta presentación.

---

# 117. Enumeration protection

La integración deberá evitar que:

```text
timing
error differences
row counts
debug information
```

expongan existencia de recursos protegidos.

---

# 118. Query result count

Una consulta autorizada:

```text
count()
```

deberá contar sólo recursos dentro del scope permitido cuando ésa sea la semántica solicitada.

---

# 119. Aggregate authorization

Agregaciones son especialmente importantes.

Ejemplo:

```text
SUM(invoice.amount)
```

no deberá incluir filas que el actor no puede considerar.

---

# 120. Authorization before aggregation

Conceptualmente:

```text
AuthorizedRows
      ↓
Aggregation
```

no:

```text
AllRows
      ↓
Aggregate
      ↓
hide unauthorized rows
```

---

# 121. GROUP BY

Las restricciones Authorization deberán aplicarse antes de producir grupos cuando la semántica así lo requiera.

---

# 122. Window functions

Igualmente, las filas no autorizadas no deberán afectar:

```text
rank
running totals
window counts
```

cuando el resultado visible dependa de ellas.

---

# 123. Pagination

La autorización debe aplicarse antes de:

```text
LIMIT/OFFSET
```

o cursor pagination.

---

# 124. Incorrect pagination

Incorrecto:

```text
fetch 20 rows
authorize each
return 7
```

como implementación general.

Produce:

- páginas inconsistentes;
- información lateral;
- rendimiento deficiente.

---

# 125. Correct model

Preferiblemente:

```text
authorization filter
→
ordering
→
pagination
```

---

# 126. Partial translatability

Si la policy no es completamente traducible, la paginación requerirá una estrategia especializada.

---

# 127. Candidate pagination

Puede utilizarse:

```text
bounded candidate windows
+
policy evaluation
+
continuation
```

pero deberá ser explícito.

---

# 128. Cursor security

Un cursor no deberá permitir eliminar/bypassear constraints de autorización.

---

# 129. Cursor ≠ authorization token

Un cursor válido:

```text
does not grant access
```

---

# 130. Cursor context binding

Cuando sea necesario, el cursor podrá incluir/bindear:

```text
authorization scope fingerprint
tenant
ordering
query fingerprint
```

para evitar reutilización incompatible.

---

# 131. Authorization scope fingerprint

Se propone:

```text
AuthorizationScopeFingerprint
```

derivado de metadata no sensible relevante.

---

# 132. Cache

Authorization puede cachear:

```text
policy metadata
compiled policy
permission graph
authorization decision
query constraints
```

según política.

---

# 133. Decision cache danger

Una decisión:

```text
Alice may update Project 42
```

puede quedar obsoleta si cambia:

```text
owner
role
permission
status
tenant membership
```

---

# 134. Cache key

Deberá incluir suficientes dimensiones:

```text
actor
action
resource
policy generation
security version
tenant
relevant state version
```

cuando corresponda.

---

# 135. TTL ≠ correctness

Un TTL de 5 minutos no garantiza:

```text
authorization still valid
```

durante esos cinco minutos.

---

# 136. Versioned authorization state

Puede utilizarse:

```text
AuthorizationVersion
```

o versiones de:

```text
role membership
permission graph
resource state
```

---

# 137. Cache invalidation

Cambios como:

```text
RoleChanged
PermissionChanged
OwnershipChanged
MembershipRemoved
ResourceStateChanged
TenantAccessRevoked
```

podrán invalidar decisiones derivadas.

---

# 138. afterCommit

La invalidación deberá coordinarse con:

```text
transaction outcome
```

---

# 139. UNKNOWN commit

Si una operación que cambia permissions tiene resultado:

```text
UNKNOWN
```

la política deberá ser conservadora.

---

# 140. Bulk reads

Una consulta de miles de recursos no deberá ejecutar:

```text
policy(resource)
```

individualmente cuando exista una representación segura más eficiente.

---

# 141. Bulk authorization

Se propone:

```text
BulkAuthorizationPlan
```

---

# 142. Bulk authorization strategies

```text
QUERY_TRANSLATED
BATCH_RESOURCE_EVALUATION
HYBRID
DENY_UNSUPPORTED
```

---

# 143. Bulk mutation

Ejemplo:

```php
Project::query()
    ->where('status', 'draft')
    ->update(['archived' => true]);
```

puede afectar miles de recursos.

---

# 144. Bulk mutation danger

No existe una entidad individual sobre la que ejecutar automáticamente:

```php
$policy->update($actor, $project);
```

---

# 145. Bulk mutation authorization

Deberá requerir:

```text
bulk policy
```

o:

```text
translatable authorization constraint
```

o estrategia explícita.

---

# 146. No implicit per-row loading

No deberá convertirse silenciosamente un bulk update en:

```text
SELECT all
hydrate all
authorize all
UPDATE one by one
```

---

# 147. Bulk operation model

```text
BulkMutation
      +
BulkAuthorizationPolicy
      +
SecurityConstraint
      ↓
Authorized Mutation Plan
```

---

# 148. Affected rows

Después de una mutación protegida:

```text
affected rows
```

puede aportar evidencia.

---

# 149. Zero affected rows

Pero:

```text
0 rows
```

puede significar:

```text
not found
not authorized
state changed
optimistic conflict
predicate mismatch
```

No deberá inferirse una causa sin evidencia suficiente.

---

# 150. Conditional delete

Ejemplo:

```text
DELETE
WHERE id = resource
AND owner_id = actor
```

puede combinar localización + enforcement.

---

# 151. Delete event semantics

Si no se afecta fila:

```text
DeletionSucceeded
```

no deberá emitirse.

---

# 152. UnitOfWork integration

El UnitOfWork conoce:

```text
Entity ChangeSets
```

pero:

```text
ChangeSet
≠
Authorization Decision
```

---

# 153. Change-aware authorization

Authorization podrá evaluar:

```text
old state
new state
changed fields
```

para policies de mutación.

---

# 154. Example

Un usuario puede editar:

```text
title
description
```

pero no:

```text
owner_id
status
security_level
```

---

# 155. Field-level authorization

Se podrá modelar:

```text
FieldAuthorizationPolicy
```

---

# 156. Field-level policy ≠ validation

Validation pregunta:

```text
is this value valid?
```

Authorization pregunta:

```text
may this actor change this field?
```

---

# 157. ChangeSet policy

Conceptualmente:

```text
authorize(
    Actor,
    UPDATE,
    Resource,
    ChangeSet
)
```

---

# 158. Dirty state

La autorización podrá ejecutarse antes de flush utilizando:

```text
original snapshot
+
proposed ChangeSet
```

---

# 159. Stale snapshot concern

El snapshot ORM puede no reflejar el estado actual del DB.

Para reglas sensibles puede requerirse:

```text
fresh state
version check
lock
conditional update
```

---

# 160. Optimistic authorization

Puede utilizarse:

```text
resource_version
```

junto con autorización.

Ejemplo:

```text
WHERE
id = ?
AND owner_id = ?
AND version = ?
```

---

# 161. Optimistic conflict

Si:

```text
affected_rows = 0
```

el sistema deberá distinguir cuando sea posible:

```text
stale version
authorization changed
resource missing
```

sin exponer información sensible.

---

# 162. Pessimistic authorization

Para operaciones críticas:

```text
BEGIN
SELECT ... FOR UPDATE
authorize
mutate
COMMIT
```

puede ser apropiado.

---

# 163. Lock ≠ authorization

El lock evita ciertas races.

No concede permiso.

---

# 164. Transaction ownership

Authorization no deberá abrir transacciones arbitrariamente sin coordinar:

```text
TransactionManager
```

---

# 165. AuthorizationTransactionCoordinator

Se podrá introducir:

```text
AuthorizationTransactionCoordinator
```

para operaciones que necesiten:

```text
fresh read
+
authorization
+
mutation
```

en una misma frontera.

---

# 166. Transaction isolation

El coordinador deberá conocer:

```text
requested isolation
effective isolation
```

cuando sea relevante.

---

# 167. UNKNOWN isolation

No deberá asumir garantías no demostradas.

---

# 168. Authorization retry

Si una transacción se reintenta:

```text
authorization
```

deberá reevaluarse dentro del nuevo intento cuando dependa de estado mutable.

---

# 169. Critical retry rule

Nunca:

```text
Attempt 1:
ALLOW

deadlock

Attempt 2:
reuse ALLOW
```

si el estado pudo cambiar.

---

# 170. Correct retry

```text
Attempt 2
├── reload relevant state
├── reevaluate authorization
└── perform mutation
```

---

# 171. Retryability

La integración deberá considerar:

```text
authorization side effects
```

si existieran.

Idealmente policy evaluation será side-effect free.

---

# 172. Policy purity

Se recomienda:

> **La evaluación de una policy debería ser determinista respecto a sus entradas observables y no producir efectos persistentes como consecuencia implícita de decidir ALLOW o DENY.**

---

# 173. Authorization side effects

Auditoría/telemetry se manejarán separadamente.

---

# 174. Authorization and events

Authorization podrá emitir:

```text
AuthorizationEvaluated
AuthorizationGranted
AuthorizationDenied
```

---

# 175. Database events

Database seguirá emitiendo:

```text
QueryExecuted
TransactionCommitted
PersistenceCompleted
```

---

# 176. Event separation

```text
AuthorizationGranted
≠
MutationCommitted
```

---

# 177. Example

Puede ocurrir:

```text
AuthorizationGranted
      ↓
UPDATE attempted
      ↓
Deadlock
      ↓
Rollback
```

---

# 178. Audit semantics

El audit deberá distinguir:

```text
permission granted
operation attempted
operation persisted
operation failed
transaction committed
```

---

# 179. Authorization audit

Ejemplo:

```text
Actor Alice
Action DELETE
Resource Project#42
Decision ALLOW
```

---

# 180. Persistence audit

Posteriormente:

```text
Project#42 deletion committed
```

es otro hecho.

---

# 181. AuthorizationAttemptId

Se utilizará para correlacionar ambos.

---

# 182. Sensitive audit

No deberán incluirse:

- secrets;
- credentials;
- unnecessary PII;
- complete query parameters;
- private resource contents.

---

# 183. Telemetry

Se podrán generar spans:

```text
authorization.evaluate
authorization.resource.resolve
authorization.query.constrain
authorization.bulk.plan
authorization.persistence.enforce
```

---

# 184. Database telemetry correlation

Con:

```text
TraceId
OperationId
AuthorizationAttemptId
```

---

# 185. Cardinality

No deberán utilizarse como metric labels:

```text
user id
resource id
email
token
```

de alta cardinalidad.

---

# 186. Diagnostics

Un diagnóstico interno podrá explicar:

```text
Policy: ProjectPolicy
Action: update
Decision: deny
ReasonCode: OWNER_REQUIRED
Constraint: owner_id = actor
```

sin exponer datos sensibles.

---

# 187. Production diagnostics

La información detallada deberá restringirse.

---

# 188. Error taxonomy

Se propone:

```text
DatabaseAuthorizationIntegrationException
├── AuthorizationResourceResolutionException
├── AuthorizationConstraintException
├── AuthorizationConstraintNotTranslatableException
├── AuthorizationEnforcementException
├── AuthorizationConsistencyException
├── AuthorizationStateConflictException
├── AuthorizationContextException
├── AuthorizationBulkOperationException
└── AuthorizationNativeEnforcementException
```

---

# 189. AuthorizationDenied

Una decisión DENY pertenece al dominio Authorization.

No deberá representarse como:

```text
DatabaseException
```

---

# 190. ResourceNotFound

Puede provenir de Database lookup.

Pero la capa superior decidirá su presentación.

---

# 191. Database unavailable

No deberá convertirse internamente en:

```text
DENY
```

sin conservar la causa.

---

# 192. External fail closed

Una aplicación puede responder como acceso denegado ante una indisponibilidad.

Pero internamente:

```text
DATABASE_UNAVAILABLE
≠
POLICY_DENIED
```

---

# 193. Constraints vs authorization

Un constraint:

```text
FOREIGN KEY
UNIQUE
CHECK
NOT NULL
```

protege invariantes persistentes.

No responde:

```text
may Alice perform this operation?
```

---

# 194. Database constraints as defense

Algunas reglas invariantes pueden y deben reforzarse en Database.

Ejemplo:

```text
tenant_id NOT NULL
```

pero no reemplaza policies.

---

# 195. Authorization invariant example

```text
Only project owner may rename project
```

es policy.

---

# 196. Structural invariant example

```text
Every project must belong to a tenant
```

es candidato a:

```text
NOT NULL
+
FOREIGN KEY
```

---

# 197. Policy changes

Authorization rules pueden cambiar sin modificar schema.

Esto refuerza la separación.

---

# 198. Data Access Security integration

El sistema deberá cooperar con:

```text
DATABASE_DATA_ACCESS_SECURITY_SYSTEM
```

---

# 199. Data Access Security ≠ Authorization

Database Data Access Security define mecanismos generales.

Authorization proporciona decisiones de aplicación.

---

# 200. Least privilege

Las credenciales Database runtime deberán tener sólo los permisos necesarios.

---

# 201. DB superuser prohibited

La aplicación ordinaria no deberá utilizar:

```text
database superuser
```

para facilitar authorization.

---

# 202. Native roles

Si se usan roles nativos, deberán mapearse explícitamente.

---

# 203. Query raw expressions

Una raw expression no deberá permitir eliminar/bypassear:

```text
SecurityConstraint
```

---

# 204. Unsafe query escape hatch

Si existe:

```text
unsafeRaw()
```

deberá:

- ser explícito;
- estar restringido;
- ser auditable;
- no desactivar security constraints silenciosamente.

---

# 205. Authorization bypass capability

Podrá existir un mecanismo privilegiado:

```text
AuthorizationBypassToken
```

para operaciones internas excepcionales.

---

# 206. Bypass token requirements

Deberá ser:

```text
explicit
capability-based
scoped
audited
non-serializable
non-global
```

---

# 207. Never boolean bypass

Evitar:

```php
$query->withoutAuthorization(true);
```

en APIs públicas ordinarias.

---

# 208. System operations

Migraciones, mantenimiento o procesos administrativos pueden operar fuera de Authorization de aplicación.

Esto deberá estar claramente separado.

---

# 209. Background jobs

Un Job puede ejecutar una operación originalmente iniciada por un usuario.

Debe decidirse si utiliza:

```text
Original Actor
Service Actor
Delegated Actor
System Actor
```

---

# 210. Actor propagation

Nunca deberá serializarse un objeto Authentication/Authorization completo al Job.

---

# 211. Actor reference

Se podrá transportar:

```text
ActorReference
DelegationContext
AuthorizationIntent
```

firmado/protegido cuando corresponda.

---

# 212. Reauthorization

Para jobs diferidos, normalmente deberá reevaluarse autorización al ejecutar.

---

# 213. Authorization at enqueue ≠ authorization at execution

```text
Authorized at T1
≠
Authorized at T2
```

---

# 214. Delegation

Algunas operaciones podrán usar una autorización delegada con:

```text
scope
expiration
action
resource constraints
issuer
```

---

# 215. Delegation ≠ copied permissions

Debe ser un concepto explícito.

---

# 216. CLI

Comandos administrativos deberán operar con:

```text
SystemActor
AdministratorActor
```

según arquitectura.

---

# 217. CLI bypass

No deberá asumirse:

```text
CLI
=
authorized
```

---

# 218. API

Igualmente:

```text
internal API
≠
trusted automatically
```

---

# 219. GraphQL

Si se integra en el futuro, field resolvers no deberán provocar N+1 authorization checks.

---

# 220. Authorization batching

El sistema deberá poder resolver:

```text
authorization requirements
```

por lote cuando sea seguro.

---

# 221. N+1 authorization

Ejemplo problemático:

```text
100 projects
×
1 policy DB query each
=
101+ queries
```

---

# 222. Authorization data preload

Podrá utilizarse:

```text
AuthorizationProjectionBatchLoader
```

---

# 223. Preload ≠ permission

Cargar los datos necesarios no concede acceso.

---

# 224. Relationship authorization

Una entidad puede ser visible pero no todas sus relaciones.

---

# 225. Example

Actor puede:

```text
view Project
```

pero no:

```text
view Project.billingDetails
```

---

# 226. Relationship loading

Eager/lazy loading deberá respetar las reglas de exposición definidas por la aplicación.

---

# 227. ORM relation ≠ authorization boundary

Una relación ORM:

```text
Project → Members
```

no significa que todos los miembros sean visibles.

---

# 228. Serialization

Authorization deberá aplicarse antes de exponer datos.

Database no reemplaza:

```text
Serialization Policy
```

---

# 229. Lazy loading security

Una property serializer no deberá disparar lazy loading de información no autorizada.

---

# 230. Search

Full-text search deberá incorporar scope Authorization antes de exponer resultados.

---

# 231. Search ranking leakage

Incluso si se ocultan documentos no autorizados, éstos no deberían afectar ranking visible si eso revela información sensible.

---

# 232. JSON queries

Authorization constraints deberán poder coexistir con JSON predicates.

---

# 233. Geographic queries

Igualmente con:

```text
spatial predicates
```

---

# 234. CTE

Security constraints deberán propagarse correctamente a CTEs relevantes.

---

# 235. Subqueries

No deberá existir bypass mediante subquery.

---

# 236. UNION

Cada branch deberá respetar authorization scope apropiado.

---

# 237. Aggregates

Cada input relation deberá ser autorizada según semántica.

---

# 238. Raw query escape hatch

Una consulta raw no puede recibir automáticamente todas las garantías semánticas.

---

# 239. Raw query authorization requirement

Deberá requerir:

```text
explicit authorization context
```

o:

```text
explicit privileged bypass
```

cuando participe en rutas protegidas.

---

# 240. Stored procedures

Igualmente deberán documentar:

```text
authorization assumptions
```

---

# 241. Import

Importar datos puede crear/modificar miles de recursos.

Deberá tener:

```text
Import Authorization Policy
```

---

# 242. Export

Exportar es especialmente sensible.

Deberá aplicar authorization antes de producir datos.

---

# 243. Export authorization

```text
Export
≠
SELECT permission
```

necesariamente.

Puede ser una acción separada.

---

# 244. Archive/restore

También pueden tener policies propias:

```text
ARCHIVE
RESTORE
```

---

# 245. Soft delete

Un registro soft-deleted puede seguir teniendo reglas de autorización.

---

# 246. History/versioning

Acceder al historial puede requerir permiso distinto de acceder al estado actual.

---

# 247. Temporal data

Una policy actual no necesariamente determina acceso histórico.

La semántica deberá definirse explícitamente.

---

# 248. Backup

El acceso a backups no deberá inferirse de permissions sobre registros ordinarios.

---

# 249. Administration

`DATABASE_ADMINISTRATION_SYSTEM` tendrá su propio modelo privilegiado.

---

# 250. Testing architecture

La integración deberá cubrir:

```text
Unit
Integration
Security
Concurrency
Transaction
Tenant
Shard
Bulk
Persistent Runtime
Performance
Failure
```

---

# 251. Unit tests

Deberán cubrir:

- constraint composition;
- policy metadata;
- translatability;
- scope fingerprints;
- actor/resource mapping;
- field-level ChangeSets;
- bypass tokens;
- error mapping.

---

# 252. Query tests

Ejemplo:

```text
Application predicate:
status = ACTIVE

Tenant constraint:
tenant_id = 10

Authorization constraint:
owner_id = 50
```

resultado semántico:

```text
tenant_id = 10
AND owner_id = 50
AND status = ACTIVE
```

---

# 253. Query assertion

Se deberá utilizar:

```text
DATABASE_QUERY_ASSERTION_SYSTEM
```

para verificar AST/constraints.

No depender únicamente de comparar SQL strings.

---

# 254. Integration tests

Con:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

según capabilities.

---

# 255. Owner test

```text
Alice owns A
Bob owns B
```

Consulta autorizada de Alice deberá producir:

```text
A
```

y nunca:

```text
B
```

---

# 256. Pagination test

Con:

```text
100 resources
50 authorized
50 unauthorized
```

la paginación deberá operar sobre el conjunto autorizado según estrategia.

---

# 257. Aggregate test

```text
SUM
COUNT
AVG
```

deberán utilizar únicamente filas permitidas.

---

# 258. Bulk update test

Verificar que:

```text
bulk update
```

no modifique recursos fuera del scope.

---

# 259. Bulk delete test

Mismo principio.

---

# 260. TOCTOU test

Utilizar dos conexiones físicas:

```text
Connection A
authorize

Connection B
change ownership

Connection A
attempt mutation
```

y verificar la estrategia configurada.

---

# 261. No sleeps

Los tests de concurrencia deberán utilizar:

```text
barriers
latches
explicit synchronization
```

no:

```text
sleep()
```

como mecanismo principal.

---

# 262. Transaction retry test

Provocar deadlock y comprobar:

```text
policy reevaluated
```

en el nuevo intento.

---

# 263. Optimistic conflict test

Modificar versión entre check y update.

La operación deberá fallar de manera segura.

---

# 264. Tenant leak test

```text
Tenant A
Actor A

Tenant B
Resource B
```

Actor A nunca deberá recibir B.

---

# 265. Shard test

La autorización no deberá consultar shards no requeridos silenciosamente.

---

# 266. Native RLS tests

Cuando exista capability:

```text
ROW_LEVEL_SECURITY
```

probar:

- actor context;
- tenant context;
- connection reset;
- role reset;
- persistent connection reuse.

---

# 267. Persistent runtime test

```text
Request 1
Actor Alice
Tenant A

Request 2
Actor Bob
Tenant B
```

en el mismo worker.

No deberá sobrevivir:

```text
Alice
Tenant A
Policy result
DB role
session variable
authorization scope
```

---

# 268. OpenSwoole test

Dos coroutines concurrentes deberán mantener contextos completamente aislados.

---

# 269. Cache test

Cambiar ownership/permission y comprobar invalidación.

---

# 270. UNKNOWN commit test

Cambiar permissions, perder conexión durante commit y verificar comportamiento conservador.

---

# 271. Failure test

Database unavailable no deberá convertirse internamente en:

```text
PolicyDenied
```

---

# 272. Performance test

Medir:

```text
authorization projection
constraint contribution
constraint merge
policy evaluation
authorized query
bulk authorization
```

sin sacrificar seguridad.

---

# 273. Proposed namespace

```text
VoltStack\Quantum\Database\Integration\Authorization
```

---

# 274. Proposed directory structure

```text
src/Quantum/Database/Integration/Authorization/
├── Contract/
│   ├── AuthorizationResourceDataProvider.php
│   ├── AuthorizationQueryConstraintContributor.php
│   ├── AuthorizationPersistenceEnforcer.php
│   └── AuthorizationScopeProvider.php
│
├── Context/
│   ├── DatabaseAuthorizationContext.php
│   ├── DatabaseAuthorizationContextResolver.php
│   ├── AuthorizationAttemptId.php
│   └── AuthorizationEnforcementMode.php
│
├── Resource/
│   ├── AuthorizationResourceReference.php
│   ├── AuthorizationProjection.php
│   ├── AuthorizationProjectionLoader.php
│   ├── AuthorizationProjectionBatchLoader.php
│   └── AuthorizationResourceMapper.php
│
├── Query/
│   ├── AuthorizationQueryRequest.php
│   ├── AuthorizationQueryConstraint.php
│   ├── AuthorizationConstraintSet.php
│   ├── AuthorizationConstraintMerger.php
│   ├── AuthorizationQueryScope.php
│   ├── AuthorizationScopeFingerprint.php
│   └── AuthorizationQueryPlanner.php
│
├── Constraint/
│   ├── SecurityConstraint.php
│   ├── OwnershipConstraint.php
│   ├── TenantConstraintBridge.php
│   ├── ActorConstraint.php
│   └── ConstraintTranslatability.php
│
├── ORM/
│   ├── OrmAuthorizationBridge.php
│   ├── EntityAuthorizationProjection.php
│   ├── ChangeSetAuthorizationAdapter.php
│   └── PersistenceAuthorizationEnforcer.php
│
├── Model/
│   ├── ModelAuthorizationBridge.php
│   └── AuthorizedModelQuery.php
│
├── Repository/
│   └── RepositoryAuthorizationBridge.php
│
├── Bulk/
│   ├── BulkAuthorizationPlan.php
│   ├── BulkAuthorizationPlanner.php
│   ├── BulkMutationAuthorization.php
│   └── BulkAuthorizationStrategy.php
│
├── Transaction/
│   ├── AuthorizationTransactionCoordinator.php
│   ├── AuthorizationConsistencyPolicy.php
│   └── AuthorizationRetryPolicy.php
│
├── Ownership/
│   ├── OwnershipDescriptor.php
│   ├── OwnershipMetadata.php
│   └── OwnershipConstraintContributor.php
│
├── Native/
│   ├── NativeAuthorizationAdapter.php
│   ├── RowLevelSecurityAdapter.php
│   ├── DatabaseRoleAdapter.php
│   └── NativeAuthorizationContextManager.php
│
├── Cache/
│   ├── AuthorizationConstraintCache.php
│   └── AuthorizationCacheInvalidator.php
│
├── Security/
│   ├── AuthorizationBypassToken.php
│   ├── AuthorizationBypassPolicy.php
│   └── AuthorizationDataClassifier.php
│
├── Telemetry/
│   └── DatabaseAuthorizationTelemetry.php
│
├── Audit/
│   └── DatabaseAuthorizationAuditBridge.php
│
├── Extension/
│   ├── DatabaseAuthorizationExtension.php
│   └── DatabaseAuthorizationExtensionRegistry.php
│
└── Exception/
    ├── DatabaseAuthorizationIntegrationException.php
    ├── AuthorizationResourceResolutionException.php
    ├── AuthorizationConstraintException.php
    ├── AuthorizationConstraintNotTranslatableException.php
    ├── AuthorizationEnforcementException.php
    ├── AuthorizationConsistencyException.php
    ├── AuthorizationStateConflictException.php
    └── AuthorizationBulkOperationException.php
```

---

# 275. Public integration example

```php
$projects = Project::query()
    ->authorizedFor(
        actor: $actor,
        action: ProjectAction::VIEW,
    )
    ->where('status', ProjectStatus::ACTIVE)
    ->orderBy('created_at', 'desc')
    ->paginate();
```

Conceptualmente:

```text
Application Predicate
+
Tenant Constraint
+
Authorization Constraint
        ↓
Query Model
        ↓
Semantic Validation
        ↓
Optimizer
        ↓
Planner
        ↓
Compiler
        ↓
Execution
```

---

# 276. Mutation example

```php
$database->transaction(function () use (
    $authorization,
    $actor,
    $projectId,
    $input,
) {
    $project = $this->projects->findForUpdate($projectId);

    $authorization->require(
        $actor,
        ProjectAction::UPDATE,
        $project,
    );

    $project->rename($input->name);

    $this->entityManager->flush();
});
```

---

# 277. Strong conditional mutation example

Para policies traducibles:

```text
Actor
  ↓
Authorization Constraint
  ↓
Mutation Query
  ↓
WHERE
    project.id = ?
AND project.owner_id = ?
AND project.version = ?
```

---

# 278. Architectural invariant set

## DB-AUTHZ-INT-001

Database ≠ Authorization.

## DB-AUTHZ-INT-002

Authentication ≠ Authorization.

## DB-AUTHZ-INT-003

Record Exists ≠ Access Granted.

## DB-AUTHZ-INT-004

Resource Found ≠ Resource Authorized.

## DB-AUTHZ-INT-005

Ownership ≠ Authorization.

## DB-AUTHZ-INT-006

Tenant Isolation ≠ Authorization.

## DB-AUTHZ-INT-007

Query Scope ≠ Authorization Decision.

## DB-AUTHZ-INT-008

Database Constraint ≠ Authorization Policy.

## DB-AUTHZ-INT-009

DB Role ≠ Application Role.

## DB-AUTHZ-INT-010

RLS ≠ Authorization Engine.

---

# 279. Query invariants

## DB-AUTHZ-INT-011

Security constraints serán typed.

## DB-AUTHZ-INT-012

Security constraints no serán raw SQL.

## DB-AUTHZ-INT-013

Application predicates no eliminarán security constraints.

## DB-AUTHZ-INT-014

Optimizer preservará security semantics.

## DB-AUTHZ-INT-015

Compiler no evaluará policies.

## DB-AUTHZ-INT-016

Driver no evaluará policies.

## DB-AUTHZ-INT-017

UNKNOWN translatability ≠ TRANSLATABLE.

## DB-AUTHZ-INT-018

Partial translation requerirá evaluación adicional.

## DB-AUTHZ-INT-019

Unauthorized candidates no serán expuestos.

## DB-AUTHZ-INT-020

Raw queries requerirán política explícita.

---

# 280. ORM invariants

## DB-AUTHZ-INT-021

persist() ≠ authorization.

## DB-AUTHZ-INT-022

flush() ≠ authorization.

## DB-AUTHZ-INT-023

ChangeSet ≠ AuthorizationDecision.

## DB-AUTHZ-INT-024

Entity ≠ AuthorizationResource necesariamente.

## DB-AUTHZ-INT-025

IdentityMap ≠ fresh authorization state.

## DB-AUTHZ-INT-026

Lazy loading no deberá provocar exposición accidental.

## DB-AUTHZ-INT-027

Repository ≠ Policy Engine.

## DB-AUTHZ-INT-028

Model API ≠ Policy Engine.

## DB-AUTHZ-INT-029

EntityManager ≠ Policy Engine.

## DB-AUTHZ-INT-030

Authorization projection ≠ managed entity.

---

# 281. Concurrency invariants

## DB-AUTHZ-INT-031

Authorized at T1 ≠ automatically authorized at T2.

## DB-AUTHZ-INT-032

TOCTOU será tratado explícitamente.

## DB-AUTHZ-INT-033

Transaction ≠ automatic TOCTOU elimination.

## DB-AUTHZ-INT-034

Lock ≠ authorization.

## DB-AUTHZ-INT-035

Version check ≠ authorization.

## DB-AUTHZ-INT-036

Retry reevaluará mutable authorization state.

## DB-AUTHZ-INT-037

Deadlock retry no reutilizará ALLOW stale.

## DB-AUTHZ-INT-038

Unknown commit permanecerá UNKNOWN.

## DB-AUTHZ-INT-039

Rollback ≠ authorization-state rewind.

## DB-AUTHZ-INT-040

Conditional mutation deberá preservar policy semantics.

---

# 282. Bulk invariants

## DB-AUTHZ-INT-041

Bulk operation ≠ per-entity operation.

## DB-AUTHZ-INT-042

Bulk update requerirá bulk authorization.

## DB-AUTHZ-INT-043

Bulk delete requerirá bulk authorization.

## DB-AUTHZ-INT-044

Bulk authorization no hidratará todas las entidades implícitamente.

## DB-AUTHZ-INT-045

Affected rows ≠ proof of authorization cause.

## DB-AUTHZ-INT-046

Zero affected rows será semánticamente ambiguo hasta obtener evidencia.

## DB-AUTHZ-INT-047

Bulk constraints serán composables.

## DB-AUTHZ-INT-048

Bulk operations respetarán tenant scope.

## DB-AUTHZ-INT-049

Bulk operations respetarán shard scope.

## DB-AUTHZ-INT-050

Bulk operations serán auditables.

---

# 283. Pagination/aggregation invariants

## DB-AUTHZ-INT-051

Authorization deberá aplicarse antes de pagination cuando sea traducible.

## DB-AUTHZ-INT-052

Unauthorized rows no contarán en authorized count.

## DB-AUTHZ-INT-053

Unauthorized rows no afectarán authorized aggregate.

## DB-AUTHZ-INT-054

Unauthorized rows no afectarán window semantics visibles cuando corresponda.

## DB-AUTHZ-INT-055

Cursor ≠ authorization token.

## DB-AUTHZ-INT-056

Cursor no bypassará security scope.

## DB-AUTHZ-INT-057

Authorization scope podrá formar parte del cursor context.

## DB-AUTHZ-INT-058

Partial policy translation requerirá pagination strategy explícita.

## DB-AUTHZ-INT-059

Over-fetch será bounded.

## DB-AUTHZ-INT-060

Under-filtering no expondrá datos.

---

# 284. Tenant/shard invariants

## DB-AUTHZ-INT-061

Tenant isolation precederá authorization filtering cuando corresponda.

## DB-AUTHZ-INT-062

Tenant membership ≠ permission over all tenant resources.

## DB-AUTHZ-INT-063

Authorization scope no eliminará tenant constraint.

## DB-AUTHZ-INT-064

Cross-tenant operations serán explícitas.

## DB-AUTHZ-INT-065

Cross-tenant operations serán auditadas.

## DB-AUTHZ-INT-066

Shard routing ≠ authorization.

## DB-AUTHZ-INT-067

Unknown shard ≠ DENY.

## DB-AUTHZ-INT-068

No habrá scatter silencioso para resolver autorización.

## DB-AUTHZ-INT-069

Tenant context será scoped.

## DB-AUTHZ-INT-070

Shard context será scoped.

---

# 285. Cache invariants

## DB-AUTHZ-INT-071

Cache Hit ≠ Authorization Valid.

## DB-AUTHZ-INT-072

TTL ≠ authorization correctness.

## DB-AUTHZ-INT-073

Decision cache tendrá scope explícito.

## DB-AUTHZ-INT-074

Policy generation formará parte de invalidation cuando corresponda.

## DB-AUTHZ-INT-075

Permission changes invalidarán estado derivado.

## DB-AUTHZ-INT-076

Ownership changes invalidarán estado derivado.

## DB-AUTHZ-INT-077

Tenant membership changes invalidarán estado derivado.

## DB-AUTHZ-INT-078

afterCommit coordinará invalidation.

## DB-AUTHZ-INT-079

UNKNOWN commit tendrá tratamiento conservador.

## DB-AUTHZ-INT-080

Cache no será authorization authority.

---

# 286. Runtime invariants

## DB-AUTHZ-INT-081

Actor state no será global mutable.

## DB-AUTHZ-INT-082

Authorization context será operation-scoped.

## DB-AUTHZ-INT-083

Authorization constraint state será operation-scoped.

## DB-AUTHZ-INT-084

Native DB role state será reseteado.

## DB-AUTHZ-INT-085

Native DB session variables serán reseteadas.

## DB-AUTHZ-INT-086

Reset failure descartará connection cuando sea necesario.

## DB-AUTHZ-INT-087

FrankenPHP requests estarán aisladas.

## DB-AUTHZ-INT-088

RoadRunner requests estarán aisladas.

## DB-AUTHZ-INT-089

OpenSwoole coroutines estarán aisladas.

## DB-AUTHZ-INT-090

Bypass capabilities no sobrevivirán al scope.

---

# 287. Security invariants

## DB-AUTHZ-INT-091

Authorization bypass será explícito.

## DB-AUTHZ-INT-092

Bypass será scoped.

## DB-AUTHZ-INT-093

Bypass será auditable.

## DB-AUTHZ-INT-094

Boolean global bypass estará prohibido.

## DB-AUTHZ-INT-095

Raw SQL no eliminará security constraints implícitamente.

## DB-AUTHZ-INT-096

Database superuser no será estrategia de autorización.

## DB-AUTHZ-INT-097

Diagnostics aplicará redaction.

## DB-AUTHZ-INT-098

Audit aplicará minimización de datos.

## DB-AUTHZ-INT-099

Database unavailable ≠ policy denied.

## DB-AUTHZ-INT-100

Fail-closed externo preservará causa interna.

---

# 288. Job invariants

## DB-AUTHZ-INT-101

Authorized at enqueue ≠ authorized at execution.

## DB-AUTHZ-INT-102

Job actor será explícito.

## DB-AUTHZ-INT-103

Actor object completo no será serializado por defecto.

## DB-AUTHZ-INT-104

Delegation será explícita.

## DB-AUTHZ-INT-105

Delegation tendrá scope.

## DB-AUTHZ-INT-106

Delegation tendrá expiration cuando corresponda.

## DB-AUTHZ-INT-107

Jobs diferidos reevaluarán autorización cuando corresponda.

## DB-AUTHZ-INT-108

System actor ≠ unrestricted actor automáticamente.

## DB-AUTHZ-INT-109

CLI actor será explícito.

## DB-AUTHZ-INT-110

Internal API ≠ automatically trusted.

---

# 289. Observability invariants

## DB-AUTHZ-INT-111

Authorization Event ≠ Database Event.

## DB-AUTHZ-INT-112

AuthorizationGranted ≠ PersistenceCommitted.

## DB-AUTHZ-INT-113

AuthorizationDenied ≠ DatabaseFailure.

## DB-AUTHZ-INT-114

Authorization audit ≠ query audit.

## DB-AUTHZ-INT-115

Events podrán correlacionarse.

## DB-AUTHZ-INT-116

Telemetry no expondrá resource data innecesaria.

## DB-AUTHZ-INT-117

Metrics evitarán IDs de alta cardinalidad.

## DB-AUTHZ-INT-118

AuthorizationAttemptId no será credential.

## DB-AUTHZ-INT-119

Policy diagnostics estarán restringidos en producción.

## DB-AUTHZ-INT-120

Sensitive predicates serán redactados cuando corresponda.

---

# 290. Anti-pattern: find then authorize then mutate

Problemático:

```php
$project = Project::find($id);

$authorization->require($user, 'update', $project);

sleep(5);

$project->update($data);
```

El estado relevante puede cambiar.

---

# 291. Anti-pattern: authorization by route alone

Incorrecto:

```text
Route protected
→
all database records authorized
```

---

# 292. Anti-pattern: tenant equals permission

Incorrecto:

```text
same tenant
→
ALLOW everything
```

---

# 293. Anti-pattern: policy inside Query Builder

Incorrecto:

```php
if ($gate->allows(...)) {
    // inside core QueryBuilder
}
```

Query Builder Core no deberá conocer Gate.

---

# 294. Anti-pattern: SQL inside policy

Evitar:

```php
class ProjectPolicy
{
    public function update(...)
    {
        return DB::raw(...);
    }
}
```

La policy no deberá convertirse en SQL generator.

---

# 295. Anti-pattern: one policy query per row

```text
1000 rows
→
1000 authorization DB queries
```

deberá evitarse mediante proyecciones, batching o constraints.

---

# 296. Anti-pattern: authorize after pagination

```text
LIMIT 20
↓
authorize
↓
return 3
```

como implementación general.

---

# 297. Anti-pattern: authorize after aggregate

Nunca:

```text
SUM(all rows)
↓
remove unauthorized resources
```

---

# 298. Anti-pattern: cache ALLOW forever

```text
ALLOW
→
cache indefinitely
```

ignora cambios de:

- ownership;
- permissions;
- status;
- tenant membership.

---

# 299. Anti-pattern: retry stale authorization

Nunca reutilizar:

```text
ALLOW
```

de un intento transaccional fallido cuando la policy depende de estado mutable.

---

# 300. Anti-pattern: persistent DB role leak

```text
Request Alice
SET ROLE admin

Request Bob
same connection
role still admin
```

es una vulnerabilidad crítica.

---

# 301. Anti-pattern: global bypass

Nunca:

```php
Authorization::$disabled = true;
```

en persistent runtimes.

---

# 302. Anti-pattern: bulk bypass

No permitir que:

```text
bulk update
```

ignore policies simplemente porque no hidrata entidades.

---

# 303. Anti-pattern: ORM relationship equals access

```text
$user->projects
```

no demuestra que cada proyecto sea accesible bajo la acción solicitada.

---

# 304. Anti-pattern: DB exception as deny

No transformar:

```text
ConnectionException
```

en:

```text
AuthorizationDenied
```

internamente.

---

# 305. Formal authorization model

Sea:

```text
A = Actor
X = Action
R = Resource
C = Context
P = Policy
S = Relevant State
```

entonces:

```text
Authorize(A, X, R, C, P, S)
→
{ALLOW, DENY, ABSTAIN}
```

---

# 306. Query authorization model

Sea:

```text
Q = Application Query
T = Tenant Constraint
Z = Authorization Constraint
```

entonces:

```text
Qauthorized =
Q ∩ T ∩ Z
```

conceptualmente.

---

# 307. Mandatory constraints

Si:

```text
T
```

y:

```text
Z
```

son mandatory security constraints:

```text
Optimizer(Q ∩ T ∩ Z)
```

deberá ser semánticamente equivalente respecto a seguridad.

---

# 308. Authorization validity over time

Sea:

```text
D(t1) = ALLOW
```

basado en estado:

```text
S(t1)
```

No se sigue necesariamente que:

```text
D(t2) = ALLOW
```

si:

```text
S(t2) ≠ S(t1)
```

---

# 309. TOCTOU formula

```text
Check(t1)
+
Use(t2)
```

es seguro únicamente si existe evidencia suficiente de que las condiciones relevantes se preservaron entre:

```text
t1 → t2
```

o la operación vuelve a validarlas atómicamente.

---

# 310. Conditional mutation model

Una mutación segura puede representarse como:

```text
M =
Update(Resource)
WHERE
ResourceId = expected
AND
AuthorizationFacts = expected
AND
Version = expected
```

---

# 311. Authorization correctness

Conceptualmente:

```text
Authorization Persistence Correctness
=
Correct Actor
∧
Correct Resource
∧
Correct Action
∧
Correct Policy
∧
Required State Freshness
∧
Tenant Isolation
∧
Shard Correctness
∧
Safe Enforcement
∧
Transaction Correctness
```

---

# 312. Query correctness

```text
Authorized Query Correctness
=
Application Query Semantics
∧
Mandatory Security Constraints
∧
Tenant Constraints
∧
Authorization Constraints
∧
No Unauthorized Leakage
```

---

# 313. V1 scope

La primera versión deberá incluir como mínimo:

```text
DatabaseAuthorizationContext
AuthorizationResourceReference
AuthorizationProjection
AuthorizationResourceDataProvider
AuthorizationQueryConstraint
SecurityConstraint
AuthorizationQueryConstraintContributor
AuthorizationConstraintMerger
OwnershipConstraint
Tenant-aware constraint composition
Repository integration
Model API integration
ORM integration
Explicit authorization enforcement
Query-translatable policies
Partial translation detection
Bulk authorization contracts
Conditional mutation support
Transaction-aware authorization
TOCTOU protections
Authorization telemetry
Authorization audit correlation
Scoped bypass capability
FrankenPHP isolation
Unit tests
Integration tests
Concurrency tests
Tenant isolation tests
```

---

# 314. V2

Podrá incorporar:

```text
Row-Level Security adapters
Database role integration
AuthorizationProjectionBatchLoader
advanced bulk authorization planner
authorization-aware cursor binding
authorization state versioning
advanced policy query compilation
shard-aware authorization planning
native DB security context
authorization diagnostics
```

---

# 315. V3

Podrá incorporar:

```text
distributed authorization-state coordination
multi-region authorization consistency
advanced policy optimizer
cross-shard authorization planning
delegated authorization capabilities
authorization query cost planning
policy-aware distributed execution
```

manteniendo siempre la separación:

```text
Authorization
≠
Database
```

---

# 316. Arquitectura final

```text
                     Authentication
                           │
                           ▼
                    Authorization
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Policies       Voters       Permissions
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                Authorization Decision
                           │
               ┌───────────┴───────────┐
               ▼                       ▼
          Resource Check        Query Constraint
               │                       │
               └───────────┬───────────┘
                           ▼
              Database Authorization Bridge
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           Query API      ORM       Persistence
              │            │            │
              └────────────┼────────────┘
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

# 317. Regla definitiva de acceso

> **La capacidad técnica de Database para localizar, cargar, modificar o eliminar un registro nunca deberá interpretarse como autorización para que el actor actual realice esa operación.**

Por tanto:

```text
Can Query
≠
May Access
```

---

# 318. Regla definitiva de filtrado

> **Cuando Authorization pueda expresarse como una restricción de consulta, VoltStack deberá incorporarla al Query Model como una restricción de seguridad estructurada y no como SQL arbitrario, preservándola durante normalización, optimización, planificación, compilación y ejecución.**

---

# 319. Regla definitiva de concurrencia

> **Una decisión ALLOW basada en estado mutable sólo será válida mientras se mantengan las condiciones relevantes que justificaron esa decisión; las operaciones sensibles deberán volver a comprobarlas, bloquearlas o incorporarlas atómicamente a la mutación.**

Por tanto:

```text
Authorized At T1
≠
Automatically Authorized At T2
```

---

# 320. Regla definitiva de separación

VoltStack deberá mantener:

```text
Authentication
identifies the actor.

Authorization
decides what the actor may do.

Database
persists and retrieves state.

Query Engine
represents data operations.

Transaction System
protects atomic boundaries.

Constraints
protect persistent invariants.

Security
protects execution boundaries.

Telemetry
observes.

Audit
records evidence.
```

Ninguno deberá apropiarse silenciosamente de las responsabilidades de los demás.

---

# 321. Invariante final

La arquitectura completa deberá preservar:

```text
Authentication Success
≠
Authorization Granted
```

```text
Authorization Granted
≠
Operation Executed
```

```text
Operation Executed
≠
Transaction Committed
```

```text
Transaction Committed
≠
External Side Effect Completed
```

y:

```text
Record Exists
≠
Access Granted
```

---

# 322. Siguiente documento

```text
320_DATABASE_JOB_AND_QUEUE_INTEGRATION_SYSTEM.md
```

Este documento definirá la integración entre:

```text
VoltStack/Quantum/Database
        ↕
VoltStack Jobs / Queue
```

incluyendo:

```text
job persistence
queue persistence boundaries
transactional dispatch
afterCommit dispatch
outbox pattern
job payload database references
entity serialization rules
job actor/tenant propagation
DatabaseContext reconstruction
connection lifecycle
worker isolation
idempotency
deduplication
retries
deadlocks
UNKNOWN transaction outcomes
job leasing
visibility timeouts
distributed workers
job state transitions
failed jobs
batch jobs
scheduled jobs
database-backed queues
queue drivers
sharding
multitenancy
telemetry
audit
FrankenPHP / worker runtime interaction
RoadRunner
OpenSwoole
testing
```

manteniendo como reglas centrales:

```text
Database Transaction
≠
Job Transaction
```

```text
Entity
≠
Job Payload
```

```text
Job Dispatched
≠
Job Executed
```

```text
Job Executed
≠
Job Completed
```

```text
Database Commit
≠
External Queue Publish
```

y resolviendo explícitamente el problema:

```text
Database Commit
        +
Queue Publish
        ↓
Dual-Write Consistency
```

mediante estrategias como:

```text
afterCommit
transactional outbox
idempotency
reconciliation
```