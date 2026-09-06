# 47_DATABASE_DELETE_QUERY_BUILDER.md

# VoltStack Quantum Database
## Delete Query Builder System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 47 — Delete Query Builder  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / DELETE  
**Versión:** 1.0

---

# 1. Propósito

`DeleteQueryBuilder` será la API especializada de VoltStack para construir operaciones de eliminación de filas.

Ejemplo:

```php
$affected = DB::table('sessions')
    ->where('expires_at', '<', $now)
    ->delete();
```

La experiencia pública podrá ser Laravel-like, pero internamente:

```text
delete()
≠
generar DELETE SQL
```

El flujo será:

```text
Developer
   │
   ▼
DeleteQueryBuilder
   │
   ▼
DeleteQueryModel
   │
   ▼
DeleteQueryNode
   │
   ▼
Normalization
   │
   ▼
Validation
   │
   ▼
Semantic Analysis
   │
   ▼
Optimizer
   │
   ▼
Planner
   │
   ▼
SQL Compiler
   │
   ▼
Executor
```

---

# 2. Regla fundamental

```text
DeleteQueryBuilder
≠
DELETE SQL Generator
```

El Builder describe:

```text
target
predicates
source relations
joins
ordering intent
limit intent
returning intent
safety intent
metadata
parameters
```

pero no decide:

```text
DELETE FROM syntax
USING syntax
JOIN DELETE syntax
RETURNING syntax
placeholder syntax
identifier quoting
physical connection
execution strategy
```

---

# 3. Definición arquitectónica

```text
DELETE Query
=
Mutation Intent
```

No debe confundirse con:

```text
physical row removal
soft delete
entity removal
cascade execution
truncate
schema drop
data retention
archival
```

Cada concepto tendrá un owner explícito.

---

# 4. Objetivos

El sistema deberá proporcionar:

- API Laravel-like.
- Delete Builder explícito.
- Target estructurado.
- Predicate integration.
- Parameterización automática.
- Subqueries.
- CTEs.
- Source relations.
- Join-aware deletes.
- Semantic `USING`.
- `RETURNING`.
- Mutation `ORDER BY` cuando sea soportable.
- Mutation `LIMIT` cuando sea soportable.
- Unbounded-delete protection.
- Cardinality analysis.
- Dependency analysis.
- Mutation Set.
- Constraint awareness.
- Foreign-key awareness.
- Transaction metadata.
- Retry classification.
- Security metadata.
- Audit metadata.
- Capability-driven portability.
- Persistent-runtime safety.
- integración con ORM sin acoplar el Builder al ORM.

---

# 5. No responsabilidades

`DeleteQueryBuilder` no será responsable de:

```text
SQL generation
identifier quoting
placeholder generation
native parameter binding
connection acquisition
transaction creation
foreign-key cascade execution
ORM cascade orchestration
soft-delete transformation
entity lifecycle events
UnitOfWork
IdentityMap
data archival
retention policy execution
TRUNCATE
DROP TABLE
physical query optimization
```

---

# 6. Arquitectura general

```text
                      DeleteQueryBuilder
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
     TargetBuilder     PredicateBuilder   SourceBuilder
           │                 │                 │
           ├─────────────────┼─────────────────┤
           │                 │                 │
           ▼                 ▼                 ▼
      JoinBuilder      ReturningBuilder  MetadataBuilder
           │                 │                 │
           └─────────────────┼─────────────────┘
                             ▼
                    DeleteBuilderState
                             │
                             ▼
                   DeleteBuilderFinalizer
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
          DeleteQueryModel           BindingSet
                 │
                 ▼
           DeleteQueryNode
```

---

# 7. API pública básica

```php
$affected = DB::table('users')
    ->where('id', $userId)
    ->delete();
```

Conceptualmente:

```text
DeleteQueryModel
├── target
│   └── users
└── predicate
    └── id = P1
```

Bindings:

```text
P1 → $userId
```

---

# 8. Builder explícito

También podrá existir:

```php
$query = DB::delete()
    ->from('users')
    ->where('id', $userId);

$result = $query->execute();
```

Ambas formas deberán converger al mismo modelo canónico.

---

# 9. Modelo canónico

Conceptualmente:

```php
final readonly class DeleteQueryModel
{
    public function __construct(
        public DeleteTarget $target,
        public ?PredicateNode $predicate,
        public RelationSourceSet $sources,
        public JoinSet $joins,
        public ?OrderingSpecification $ordering,
        public ?LimitSpecification $limit,
        public ?ReturningSpecification $returning,
        public ParameterDefinitionSet $parameters,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 10. DeleteBuilderState

Durante construcción:

```text
DeleteBuilderState
├── target
├── predicate
├── sources
├── joins
├── ordering
├── limit
├── returning
├── parameterRegistry
├── bindingBuilder
├── metadataBuilder
└── sourceMap
```

Será:

```text
operation-scoped
mutable
temporary
non-cacheable
non-shared
```

---

# 11. Finalización

```text
DeleteBuilderState
       │
       ▼
DeleteBuilderFinalizer
       │
       ├───────────────┐
       ▼               ▼
DeleteQueryModel    BindingSet
       │
       ▼
DeleteQueryNode
```

El modelo final será immutable.

---

# 12. Target

```php
DB::table('users')->delete();
```

produce:

```text
DeleteTarget
└── RelationIdentifier(users)
```

El string se convierte en un identifier estructurado en el boundary.

---

# 13. Qualified target

```php
DB::table('public.users')->delete();
```

podrá representar:

```text
QualifiedRelationIdentifier
├── namespace: public
└── relation: users
```

sin decidir quoting.

---

# 14. Alias

```php
DB::table('users as u')
    ->where('u.id', $id)
    ->delete();
```

podrá producir:

```text
AliasedDeleteTarget
├── relation: users
└── alias: u
```

La validez concreta se resolverá posteriormente.

---

# 15. Target semantics

Debe distinguirse:

```text
relation referenced
≠
relation deleted
```

En un DELETE con múltiples relaciones, sólo las relaciones declaradas como mutation targets serán modificadas.

---

# 16. V1 target policy

Como default seguro, V1 deberá favorecer:

```text
one DELETE target
```

por query.

Los multi-target deletes nativos de ciertos motores no formarán parte de la semántica básica.

---

# 17. Multi-target DELETE

Si se soporta posteriormente:

```text
DELETE target A
DELETE target B
```

deberá ser una capability explícita y un modelo semántico específico.

Nunca se inferirá porque dos relaciones aparezcan en un JOIN.

---

# 18. Predicate System

DELETE reutilizará completamente:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

Ejemplo:

```php
DB::table('sessions')
    ->where('expires_at', '<', $now)
    ->where('revoked', true)
    ->delete();
```

---

# 19. Predicate representation

```text
AND
├── expires_at < P1
└── revoked = P2
```

Bindings:

```text
P1 → $now
P2 → true
```

---

# 20. No DELETE-specific predicate engine

```text
SELECT
UPDATE
DELETE
```

reutilizarán el mismo sistema de predicates.

---

# 21. NULL predicates

```php
->whereNull('deleted_at')
```

produce:

```text
NullPredicateNode
```

No:

```text
deleted_at = NULL
```

---

# 22. Collection predicates

```php
->whereIn('id', $ids)
```

podrá producir:

```text
CollectionParameter(P1)
```

sin expandir inmediatamente placeholders.

---

# 23. Empty IN

La semántica de:

```php
->whereIn('id', [])
```

será determinada antes de SQL.

Si el contrato define:

```text
x IN empty-set
→ FALSE
```

el DELETE tendrá un predicate semánticamente insatisfacible.

---

# 24. No premature optimization

Aunque Constraint Analysis determine:

```text
predicate = impossible
```

el Builder no eliminará ni reescribirá la query.

---

# 25. DELETE sin WHERE

La operación:

```php
DB::table('users')->delete();
```

es arquitectónicamente válida como intención, pero extremadamente sensible.

---

# 26. UnboundedDelete

Se definirá:

```text
UnboundedDelete
```

como una mutación cuyo conjunto objetivo no está restringido semánticamente.

---

# 27. Regla importante

```text
No WHERE
→ potentially unbounded

WHERE exists
≠
bounded
```

---

# 28. Ejemplo engañoso

```php
DB::table('users')
    ->whereRaw('1 = 1')
    ->delete();
```

no deberá clasificarse como seguro sólo porque existe un `WHERE`.

---

# 29. Safety Policy

Se definirá:

```text
DeleteSafetyPolicy
```

con políticas como:

```text
ALLOW
WARN
REQUIRE_EXPLICIT_INTENT
FORBID
```

---

# 30. Default recomendado

VoltStack deberá favorecer un default más estricto para DELETE que para SELECT.

Una configuración recomendada podrá requerir:

```php
->allowUnboundedDelete()
```

para una eliminación completa.

---

# 31. Explicit intent

Ejemplo:

```php
DB::table('temporary_records')
    ->allowUnboundedDelete()
    ->delete();
```

registrará:

```text
UnboundedMutationIntent::EXPLICIT
```

---

# 32. Explicit intent ≠ authorization

La declaración:

```text
allowUnboundedDelete
```

no significa:

```text
authorized
safe for production
retryable
transactionally safe
compliant
```

---

# 33. Boundedness analysis

Constraint Analysis podrá determinar:

```text
DELETE FROM users
WHERE id = P1
```

sobre PK como:

```text
ResultMutationCardinality
→ AT_MOST_ONE
```

---

# 34. Cardinality classification

Podrá derivarse:

```text
ZERO
AT_MOST_ONE
BOUNDED_SET
UNBOUNDED_SET
UNKNOWN
```

---

# 35. Unique constraints

```text
WHERE email = P1
```

podrá implicar `AT_MOST_ONE` sólo si:

```text
email
→ semantically unique
```

bajo las reglas reales de nullability/uniqueness.

---

# 36. Composite unique keys

Schema:

```text
UNIQUE(tenant_id, external_id)
```

Query:

```text
tenant_id = P1
AND external_id = P2
```

podrá derivar:

```text
AT_MOST_ONE
```

si las reglas de unicidad lo permiten.

---

# 37. Partial key

```text
tenant_id = P1
```

por sí solo no implica una fila.

---

# 38. Constraint Analysis integration

Se reutilizará:

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

para derivar:

```text
uniqueness
key coverage
nullability
equality classes
contradictions
cardinality bounds
relationship evidence
```

---

# 39. Contradictory predicate

```text
id = 1
AND
id = 2
```

puede producir:

```text
UnsatisfiableConstraintSet
```

---

# 40. Optimizer responsibility

Convertir una mutación imposible en:

```text
no-op physical operation
```

pertenece al Optimizer/Planner, no al Builder.

---

# 41. DELETE vs TRUNCATE

Regla:

```text
DELETE all rows
≠
TRUNCATE
```

---

# 42. No automatic TRUNCATE rewrite

VoltStack nunca transformará automáticamente:

```php
DB::table('logs')
    ->allowUnboundedDelete()
    ->delete();
```

en:

```text
TRUNCATE logs
```

---

# 43. Razones

DELETE y TRUNCATE pueden diferir en:

```text
transaction semantics
triggers
identity reset
foreign keys
locking
permissions
logging
replication
affected-row semantics
RETURNING
```

---

# 44. Truncate System

Si existe una API:

```php
DB::table('logs')->truncate();
```

será una operación semántica distinta.

---

# 45. DELETE vs soft delete

Regla crítica:

```text
DELETE
≠
SOFT DELETE
```

---

# 46. Soft delete

Un soft delete normalmente equivale a:

```text
UPDATE target
SET deleted_at = timestamp
```

No a un `DeleteQueryNode`.

---

# 47. Ownership

El sistema:

```text
269_DATABASE_SOFT_DELETE_SYSTEM.md
```

será responsable de definir la política completa de soft deletion.

---

# 48. Query Builder core

`DeleteQueryBuilder` no preguntará:

```php
if ($model->usesSoftDeletes()) {
    ...
}
```

---

# 49. ORM delete()

Una API ORM:

```php
$user->delete();
```

podrá resolver mediante metadata que la entidad usa soft delete.

Entonces:

```text
EntityManager
     │
     ▼
SoftDelete Policy
     │
     ▼
UpdateQueryModel
```

---

# 50. Hard delete ORM

Una API como:

```php
$user->forceDelete();
```

podrá producir:

```text
DeleteQueryModel
```

si la política ORM lo permite.

---

# 51. Builder delete() remains physical intent

Por tanto:

```php
DB::table('users')->delete();
```

representará una intención de eliminación física de filas.

No consultará metadata ORM.

---

# 52. Foreign keys

Schema-aware semantic analysis podrá conocer:

```text
orders.user_id
→ users.id
```

---

# 53. FK dependency

Eliminar:

```text
users.id = P1
```

podrá tener dependencias referenciales conocidas.

---

# 54. Referential action metadata

Podrá conocerse:

```text
ON DELETE CASCADE
ON DELETE SET NULL
ON DELETE RESTRICT
ON DELETE NO ACTION
```

sin que el Builder ejecute dichas acciones.

---

# 55. Database cascade

Si el schema define:

```text
ON DELETE CASCADE
```

el servidor de base de datos es responsable de la cascada.

---

# 56. ORM cascade

Debe distinguirse:

```text
Database Cascade
≠
ORM Cascade
```

---

# 57. ORM cascade

ORM podrá decidir:

```text
remove child entity
emit lifecycle events
maintain in-memory graph
execute child deletes
```

antes o después de la operación principal.

Eso pertenece al Persistence Engine.

---

# 58. No fake cascade events

Un direct DELETE con DB cascade no deberá generar automáticamente eventos ORM por cada fila hija eliminada.

---

# 59. Referential impact

Semantic Analysis podrá construir:

```text
ReferentialImpactDescriptor
```

con información conocida.

---

# 60. ReferentialImpactDescriptor

Conceptualmente:

```text
ReferentialImpact
├── target: users
├── incomingForeignKeys
│   ├── orders.user_id
│   └── comments.user_id
└── actions
    ├── CASCADE
    └── SET_NULL
```

---

# 61. Referential impact ≠ execution simulation

No se intentará calcular necesariamente cuántas filas cascada serán eliminadas.

---

# 62. Source relations

DELETE podrá depender de relaciones adicionales.

Ejemplo conceptual:

```text
Delete inactive users
whose accounts are suspended
```

---

# 63. Join-aware API

Podrá existir:

```php
DB::table('users as u')
    ->join('accounts as a', 'a.user_id', '=', 'u.id')
    ->where('a.suspended', true)
    ->delete();
```

---

# 64. Semántica

Esto representa:

```text
Mutation Target
└── users AS u

Source Relation
└── accounts AS a

Join Predicate
└── a.user_id = u.id

Filter
└── a.suspended = TRUE
```

---

# 65. No vendor syntax

El Builder no decidirá si debe compilarse como:

```text
DELETE target FROM target JOIN ...
```

o:

```text
DELETE FROM target USING ...
```

o mediante:

```text
EXISTS(...)
```

---

# 66. Semantic USING

Podrá existir:

```php
DB::delete()
    ->from('users as u')
    ->using('accounts as a')
    ->whereColumn('a.user_id', 'u.id')
    ->where('a.suspended', true);
```

---

# 67. `using()` meaning

`using()` significará:

```text
additional source relation participating
in delete qualification
```

No:

```text
emit USING keyword
```

---

# 68. SourceRelationSet

```text
DeleteQuery
├── MutationTarget
└── SourceRelationSet
    ├── accounts
    └── ...
```

---

# 69. Relation resolution

Se reutilizará:

```text
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
```

para resolver:

```text
relation identity
aliases
scope
columns
join semantics
correlations
dependencies
lineage
multiplicity
```

---

# 70. Multiple source matches

Para DELETE, múltiples source rows que califican la misma target row normalmente no significan múltiples eliminaciones físicas de esa misma fila.

Sin embargo, VoltStack no asumirá reglas vendor-specific.

---

# 71. Semantic target identity

La semántica deberá modelar:

```text
target row qualifies for deletion
```

no:

```text
delete once per joined tuple
```

---

# 72. Planner responsibility

El Planner deberá elegir una estrategia que preserve esa semántica.

---

# 73. Semi-join interpretation

Muchos join-based deletes pueden representarse lógicamente como:

```text
Delete target rows
where matching source row exists
```

Esto puede acercarse a:

```text
SEMI JOIN / EXISTS
```

pero dicha transformación pertenece al Optimizer/Planner.

---

# 74. No Builder rewrite

El Builder no transformará:

```text
JOIN + DELETE
```

en:

```text
EXISTS
```

por sí mismo.

---

# 75. Subquery predicates

Ejemplo:

```php
DB::table('users')
    ->whereExists(
        DB::table('bans')
            ->select('id')
            ->whereColumn('bans.user_id', 'users.id')
    )
    ->delete();
```

---

# 76. Representation

```text
DeleteQuery
├── target: users
└── predicate
    └── ExistsPredicate
        └── SelectQueryArtifact
```

---

# 77. Child builders

El subquery Builder será finalizado antes de incorporarse.

```text
SelectQueryBuilder
       │
       ▼
SelectQueryArtifact
       │
       ▼
ExistsPredicate
```

---

# 78. No mutable child builders

El DeleteQueryModel final no retendrá builders hijos mutables.

---

# 79. Correlation

Las referencias:

```text
bans.user_id = users.id
```

serán resueltas por Symbol/Relation Resolution.

---

# 80. CTEs

DELETE deberá poder participar en queries con CTEs.

Ejemplo conceptual:

```text
WITH expired_sessions AS (...)
DELETE ...
```

---

# 81. CTE representation

```text
Query
├── CteDefinitionSet
└── DeleteQueryModel
```

---

# 82. Recursive CTE

Será capability-driven.

El Delete Builder no tendrá lógica vendor-specific para CTEs recursivos.

---

# 83. RETURNING

VoltStack deberá poder representar:

```php
$deleted = DB::table('sessions')
    ->where('user_id', $userId)
    ->returning('id', 'token_hash')
    ->delete();
```

---

# 84. ReturningSpecification

```text
ReturningSpecification
├── id
└── token_hash
```

---

# 85. Semántica

```text
RETURNING
=
return data associated with rows affected
by the delete mutation
```

No:

```text
append "RETURNING ..."
```

---

# 86. Capability requirement

Podrá derivarse:

```text
DML.DELETE.RETURNING
```

---

# 87. Returning security

Un campo como:

```text
token_hash
secret
credential
```

podrá ser clasificado como sensible incluso cuando forme parte de RETURNING.

---

# 88. Result modes

Podrán existir:

```text
AFFECTED_ROWS
RETURNING_ROWS
RETURNING_SCALAR
NONE
```

---

# 89. Internal result

```text
DeleteExecutionResult
├── affectedRows
├── returnedRows?
├── warnings?
└── executionMetadata
```

---

# 90. Public result

El caso común podrá devolver:

```text
int affectedRows
```

manteniendo internamente el resultado tipado.

---

# 91. ORDER BY

Algunas plataformas pueden permitir ordenamiento en determinados DELETEs.

VoltStack lo representará semánticamente.

---

# 92. OrderingSpecification

```php
DB::table('logs')
    ->where('processed', true)
    ->orderBy('created_at')
    ->limit(1000)
    ->delete();
```

produce intención:

```text
Ordering
└── created_at ASC

Limit
└── 1000
```

---

# 93. No SQL assumption

El Builder no asumirá:

```text
DELETE ... ORDER BY ... LIMIT ...
```

como sintaxis universal.

---

# 94. LIMIT

```php
->limit(1000)
```

representará:

```text
MutationLimitSpecification(1000)
```

---

# 95. Capability requirements

Podrán derivarse:

```text
DML.DELETE.ORDER_BY
DML.DELETE.LIMIT
```

---

# 96. Safe emulation

Si una plataforma no soporta DELETE LIMIT directamente, el Planner podrá considerar:

```text
select target keys
+
delete by key set
```

sólo si puede preservar:

```text
atomicity
ordering semantics
locking
visibility
concurrency
cardinality
transaction semantics
```

---

# 97. No naive two-query rewrite

No deberá convertirse automáticamente:

```text
DELETE LIMIT 100
```

en:

```text
SELECT 100 IDs
DELETE IDs
```

si existe una ventana de carrera entre ambas operaciones.

---

# 98. Deterministic limited delete

Una policy podrá exigir orden estable cuando `LIMIT` determina cuáles filas desaparecen.

---

# 99. Tie-breaking

Ejemplo:

```text
ORDER BY created_at
LIMIT 100
```

puede seguir siendo no determinista si muchas filas comparten `created_at`.

---

# 100. Stable ordering analysis

Planner/Semantic Analysis podrá determinar si el orden incluye una clave suficiente para estabilidad.

Ejemplo:

```text
ORDER BY created_at, id
```

con `id` único.

---

# 101. DELETE LIMIT ≠ batch delete system

El límite de una sentencia individual no sustituye:

```text
205_DATABASE_BULK_DELETE_SYSTEM.md
```

---

# 102. Bulk deletion

Eliminar millones de filas podrá requerir:

```text
batching
transaction boundaries
lock management
replication awareness
vacuum/maintenance awareness
backpressure
resource governance
```

---

# 103. Builder role in bulk operations

El Builder podrá ser utilizado para construir cada mutación lógica, pero no será el Bulk Delete Coordinator.

---

# 104. Batch deletion example

Conceptualmente:

```text
BulkDeleteCoordinator
       │
       ├── batch 1
       │    └── DeleteQueryArtifact
       │
       ├── batch 2
       │    └── DeleteQueryArtifact
       │
       └── ...
```

---

# 105. Retry semantics

DELETE no será automáticamente retry-safe.

---

# 106. Simple delete

```text
DELETE user WHERE id = P1
```

puede parecer idempotente:

```text
first execution  → 1 row
second execution → 0 rows
```

pero los efectos observables pueden diferir.

---

# 107. Retry hazards

Deben considerarse:

```text
triggers
cascades
audit records
CDC
RETURNING
affected row expectations
transaction uncertainty
external side effects
```

---

# 108. Retry classification

Podrá ser:

```text
SAFE
CONDITIONALLY_SAFE
UNSAFE
UNKNOWN
```

---

# 109. Exactly-once assumptions

VoltStack no prometerá:

```text
exactly-once delete
```

sólo porque una query use PK.

---

# 110. Execution uncertainty

Si una conexión se pierde después de enviar DELETE:

```text
client does not know
whether server committed mutation
```

Esto debe tratarse en Retry/Transaction/Resilience layers.

---

# 111. Transaction requirements

DELETE podrá declarar:

```text
TransactionRequirement
```

pero el Builder no iniciará una transacción.

---

# 112. Transaction context

Una transacción activa pertenecerá a:

```text
TransactionContext
```

---

# 113. QueryIntent

DELETE derivará:

```text
QueryIntent::WRITE
```

---

# 114. ConnectionIntent

Normalmente:

```text
ConnectionIntent::WRITE
```

---

# 115. Physical connection

El Builder no decidirá:

```text
primary
replica
pool
host
socket
PDO
```

---

# 116. Mutation Set

Semantic Analysis deberá producir:

```text
QueryMutationSet
└── target relation
```

Para DELETE, la unidad principal de mutación será la fila/relation target.

---

# 117. Column-level implications

Aunque DELETE elimine filas completas, podrá registrar dependencia sobre:

```text
target key columns
predicate columns
returning columns
source columns
relationship columns
```

---

# 118. Dependency Set

Ejemplo:

```text
Dependencies
├── users
├── users.id
├── accounts
├── accounts.user_id
└── accounts.suspended

Mutations
└── users
```

---

# 119. Dependency ≠ Mutation

Regla:

```text
QueryDependencySet
≠
QueryMutationSet
```

---

# 120. Referential mutations

Una DB cascade puede provocar mutaciones indirectas en otras relaciones.

---

# 121. Direct vs indirect mutation

Se deberá distinguir:

```text
DirectMutationSet
```

de:

```text
PotentialReferentialMutationSet
```

---

# 122. Example

```text
Direct
└── users

Potential indirect
├── orders     [CASCADE]
└── profiles   [CASCADE]
```

---

# 123. Conservative representation

Si el schema es parcial:

```text
indirect mutation impact
→ UNKNOWN
```

No:

```text
none
```

---

# 124. Cache invalidation

Estos mutation sets podrán alimentar:

```text
Query Cache
Result Cache
Entity Cache
application cache integrations
```

---

# 125. Builder no invalida caches

El Builder sólo construirá la intención.

---

# 126. Events

Construir:

```php
$query = DB::table('users')->where('id', 1);
```

no disparará eventos de ejecución.

---

# 127. Execution events

Podrán existir:

```text
DeleteQueryExecuting
DeleteQueryExecuted
DeleteQueryFailed
```

en capas posteriores.

---

# 128. Entity events

Un direct delete no deberá fingir:

```text
UserDeleting
UserDeleted
```

para cada fila.

---

# 129. ORM entity deletion

En ORM:

```text
EntityManager
    │
    ▼
UnitOfWork
    │
    ▼
Persistence Planner
    │
    ▼
DeleteQueryModel
```

Los lifecycle events se procesarán en la capa ORM.

---

# 130. IdentityMap

Después de una eliminación ORM exitosa:

```text
IdentityMap
```

podrá necesitar actualización.

Eso no es responsabilidad del Delete Builder.

---

# 131. Direct query and IdentityMap

Un direct DELETE puede dejar entidades previamente cargadas en memoria desactualizadas.

---

# 132. ORM synchronization policy

El ORM deberá definir cómo tratar direct bulk mutations respecto a:

```text
IdentityMap
UnitOfWork
loaded collections
entity state
```

---

# 133. No hidden synchronization

DeleteQueryBuilder no recorrerá automáticamente IdentityMap.

---

# 134. Security architecture

DELETE será una operación de alto impacto.

La seguridad deberá poder aplicar:

```text
target restrictions
predicate injection
tenant restrictions
authorization requirements
audit requirements
sensitive RETURNING restrictions
unbounded mutation policy
```

---

# 135. No Authorization dependency

Database core no dependerá directamente de:

```text
Authorization
Authentication
Tenant
HTTP
```

---

# 136. Policy integration

Una capa superior podrá transformar:

```text
DELETE users
WHERE id = P1
```

en:

```text
DELETE users
WHERE id = P1
AND organization_id = P2
```

---

# 137. Transformation provenance

El predicate añadido deberá conservar:

```text
provenance
policy id
security classification
mandatory status
```

---

# 138. Mandatory predicates

Un security predicate podrá marcarse:

```text
MANDATORY
```

para impedir que optimizaciones/extensiones inseguras lo eliminen.

---

# 139. Multitenancy

Una integración podrá añadir:

```text
tenant_id = Ptenant
```

de manera explícita.

---

# 140. No global tenant lookup

Prohibido dentro del Builder core:

```php
Tenant::current();
```

---

# 141. Tenant semantic schema

Si diferentes tenants tienen diferentes schemas, el Semantic Fingerprint deberá incluir la identidad semántica del schema correspondiente.

---

# 142. Sensitive parameters

Predicates como:

```text
token = P1
```

podrán marcar `P1` como sensible.

---

# 143. Redaction

Diagnostics deberán mostrar:

```text
token = [REDACTED]
```

en lugar del valor.

---

# 144. RETURNING redaction

Valores devueltos sensibles también deberán respetar políticas de exposición.

---

# 145. Audit metadata

Podrá existir:

```text
AuditRequirement
├── mutationType: DELETE
├── target
├── classification
└── reason?
```

---

# 146. Business reason

Una API superior podría adjuntar:

```php
->metadata(
    QueryMetadata::auditReason('expired-session-cleanup')
)
```

sin convertir ese dato en SQL.

---

# 147. Raw predicates

Se permitirá escape hatch explícito:

```php
->whereRaw(...)
```

según las reglas del Predicate System.

---

# 148. Raw does not mean safe

```text
RAW
≠
TRUSTED
≠
PORTABLE
≠
SEMANTICALLY UNDERSTOOD
```

---

# 149. Raw barrier

Un RawPredicate podrá limitar:

```text
constraint inference
cardinality inference
portability
optimizer transformations
security inspection
```

---

# 150. Unbounded analysis with RAW

Ante:

```text
WHERE RawPredicate
```

la clasificación segura puede ser:

```text
UNKNOWN
```

en vez de asumir boundedness.

---

# 151. Identifier security

Esto:

```php
DB::table($userInput)->delete();
```

deberá pasar por Identifier Policy.

---

# 152. Identifier ≠ parameter

No podrá tratarse el nombre de tabla como:

```text
value parameter
```

---

# 153. Schema-aware resolution

Semantic Analysis resolverá:

```text
target relation
target alias
predicate columns
source relations
foreign keys
unique keys
constraints
returning references
relation capabilities
```

mediante `QuerySchemaView`.

---

# 154. No hidden schema I/O

Prohibido:

```text
DeleteQueryBuilder
      │
      ▼
information_schema
```

---

# 155. Offline analysis

Será posible:

```text
DeleteQueryArtifact
+
Schema Snapshot
+
Capability Snapshot
      │
      ▼
Semantic Analysis
```

sin conexión activa.

---

# 156. Type inference

Los parameters del predicate reutilizarán:

```text
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
```

---

# 157. Example

```text
users.id : UUID

Predicate:
users.id = P1
```

permite inferir:

```text
P1 : UUID-compatible
```

---

# 158. Domain types

Si:

```text
users.id
→ Domain<UserId, UUID>
```

el sistema conservará la identidad de dominio cuando sea relevante.

---

# 159. Semantic Query Graph

Un DELETE podrá representarse como:

```text
DeleteQuery
│
├── MutationTarget
│   └── users
│
├── Predicate
│   └── id = P1
│
├── Parameter
│   └── P1
│
├── Constraint
│   └── users.id UNIQUE
│
├── Mutation
│   └── users
│
└── Capability
    └── DML.DELETE
```

---

# 160. Join-aware Semantic Graph

```text
accounts.suspended
        │
        ▼
    Predicate
        │
        ▼
accounts ── JOIN ── users
                    │
                    ▼
               DeleteTarget
```

---

# 161. Referential graph

```text
orders.user_id
      │ FK
      ▼
users.id
   │
   ▼
DeleteTarget
```

con metadata:

```text
ON DELETE CASCADE
```

---

# 162. Constraint Graph

Constraint Analysis podrá aportar:

```text
PK
UNIQUE
FK
NOT NULL
equality
constant
cardinality
relationship
contradiction
```

---

# 163. Semantic artifact

Conceptualmente:

```text
SemanticDeleteArtifact
├── resolvedTarget
├── resolvedPredicate
├── resolvedSources
├── relationGraph
├── typeTable
├── constraintGraph
├── directMutationSet
├── referentialImpact
├── dependencySet
├── cardinalityClassification
├── capabilityRequirements
├── portabilityProfile
├── provenance
└── semanticFingerprint
```

---

# 164. Optimizer

El Optimizer consumirá significado ya resuelto.

No volverá a interpretar nombres desde strings.

---

# 165. Possible optimizations

Podrá considerar:

```text
predicate simplification
constant propagation
constraint-based simplification
EXISTS/semi-join transformations
subquery decorrelation
redundant join elimination
impossible-mutation detection
```

siempre preservando semántica.

---

# 166. DELETE optimization conservatism

Una transformación deberá considerar efectos observables.

---

# 167. Example

Un predicate semánticamente `FALSE` podría permitir no enviar una sentencia al servidor.

Pero deben considerarse:

```text
query events
audit policy
statement-level triggers
execution expectations
```

según el contrato de VoltStack.

---

# 168. Mutation no-op policy

El Planner podrá producir:

```text
NoOpMutationPlan
```

sólo cuando el contrato semántico garantice equivalencia observable suficiente.

---

# 169. Planner

El Planner será responsable de:

```text
delete strategy
join delete strategy
USING/from strategy
EXISTS strategy
key selection strategy
limited delete strategy
returning strategy
parameter layout
capability emulation
```

---

# 170. Logical mutation plan

Podrá existir:

```text
LogicalDeletePlan
├── target
├── qualification
├── sources
├── ordering
├── limit
└── output
```

---

# 171. Physical plan

Podrá convertirse en:

```text
NativeDeletePlan
JoinedDeletePlan
UsingDeletePlan
KeySetDeletePlan
CorrelatedDeletePlan
NoOpDeletePlan
ExtensionDeletePlan
```

---

# 172. Physical plan ≠ SQL

Incluso:

```text
NativeDeletePlan
```

seguirá siendo estructurado.

---

# 173. Compiler

El Compiler decidirá:

```text
DELETE syntax
FROM syntax
USING syntax
JOIN syntax
CTE placement
ORDER BY syntax
LIMIT syntax
RETURNING syntax
placeholder syntax
identifier quoting
```

---

# 174. Executor

El Executor manejará:

```text
connection acquisition
prepare
bind
execute
affected rows
returning rows
timeout
cancellation
errors
cleanup
```

---

# 175. Driver

El Driver será responsable de la adaptación al API nativo:

```text
PDO
native extension
future driver
```

---

# 176. Capability model

Ejemplos:

```text
DML.DELETE
DML.DELETE.RETURNING
DML.DELETE.USING
DML.DELETE.JOIN
DML.DELETE.ORDER_BY
DML.DELETE.LIMIT
DML.DELETE.CTE
DML.DELETE.MULTI_TARGET
```

---

# 177. Capability checks

Nunca:

```php
if ($driver === 'pgsql') {
}
```

en Delete Builder.

---

# 178. Capability snapshot

Semantic Analysis/Planner consumirán:

```text
CapabilitySnapshot
```

estable durante la operación.

---

# 179. Portability profile

Podrá ser:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
UNKNOWN
```

---

# 180. MySQL/MariaDB isolation

El Builder no conocerá sus formas específicas de:

```text
joined DELETE
multi-table DELETE
ORDER BY/LIMIT
```

---

# 181. PostgreSQL isolation

El Builder no codificará:

```text
USING
RETURNING
```

como strings especiales.

---

# 182. SQLite isolation

Las diferencias por versión/capability permanecerán fuera del Builder.

---

# 183. Validation pipeline

```text
Construction Validation
       │
       ▼
Structural Validation
       │
       ▼
Semantic Validation
       │
       ▼
Safety Validation
       │
       ▼
Capability Validation
       │
       ▼
Runtime Validation
```

---

# 184. Construction validation

Podrá detectar:

```text
missing target
invalid identifier
negative limit
invalid builder state
duplicate configuration
invalid ordering declaration
malformed returning declaration
```

---

# 185. Structural validation

Podrá detectar:

```text
malformed predicate
invalid join shape
invalid source structure
invalid CTE structure
invalid returning expression structure
```

---

# 186. Semantic validation

Podrá detectar:

```text
unknown target
ambiguous column
unknown predicate column
invalid source reference
invalid correlation
invalid returning reference
invalid target relation
```

---

# 187. Safety validation

Podrá detectar:

```text
unbounded delete forbidden
unknown boundedness requiring explicit intent
mandatory predicate missing
unsafe mutation policy violation
```

---

# 188. Capability validation

Podrá detectar:

```text
RETURNING unsupported
USING unsupported without safe plan
JOIN delete unsupported without safe plan
LIMIT unsupported without safe plan
ORDER BY unsupported
CTE mutation unsupported
```

---

# 189. Runtime validation

Podrá detectar:

```text
missing binding
binding conversion failure
transaction requirement failure
connection failure
deadlock
lock timeout
statement timeout
constraint violation
```

---

# 190. Error hierarchy

```text
DeleteQueryBuilderException
├── MissingDeleteTargetException
├── InvalidDeleteTargetException
├── InvalidDeletePredicateException
├── InvalidDeleteSourceException
├── InvalidDeleteJoinException
├── InvalidDeleteLimitException
├── InvalidDeleteOrderingException
├── InvalidDeleteReturningException
├── UnboundedDeleteRejectedException
├── DeleteBuilderAlreadyFinalizedException
└── DeleteBuilderBudgetExceededException
```

---

# 191. Semantic errors

```text
UnknownDeleteTargetException
AmbiguousDeleteReferenceException
InvalidDeleteCorrelationException
InvalidDeleteReturningReferenceException
DeleteSafetyViolationException
UnsupportedDeleteCapabilityException
InvalidDeleteSourceSemanticsException
```

---

# 192. Lifecycle

```text
CREATED
   │
   ▼
TARGET_DEFINED
   │
   ▼
QUALIFICATION_DEFINED
   │
   ▼
CONFIGURED
   │
   ▼
FINALIZED
```

o:

```text
FAILED
```

---

# 193. Finalized state

Después de `finalize()`:

```text
DeleteQueryModel
```

será immutable.

---

# 194. Builder reuse

Un Builder finalizado no deberá mutarse.

---

# 195. Copy

Podrá existir:

```php
$base = DB::table('sessions')
    ->where('expired', true);

$a = $base->copy()->where('tenant_id', 1);
$b = $base->copy()->where('tenant_id', 2);
```

---

# 196. State isolation

Cada copia tendrá:

```text
independent predicate state
independent parameter registry
independent bindings
independent metadata
independent sources
```

---

# 197. Persistent runtime

Compatible con:

```text
FrankenPHP
RoadRunner
OpenSwoole
queue workers
long-running commands
coroutines
```

---

# 198. Shared services

Podrán compartirse:

```text
DeleteQueryBuilderFactory
PredicateFactory
ExpressionFactory
IdentifierFactory
immutable descriptors
frozen extension registries
stateless finalizers
```

---

# 199. Local state

Siempre operation-scoped:

```text
DeleteBuilderState
ParameterRegistry
BindingSetBuilder
PredicateBuilderState
JoinBuilderState
SourceBuilderState
diagnostics
```

---

# 200. No global builder

Prohibido:

```php
DeleteQueryBuilder::current();
```

---

# 201. No static parameter counter

Prohibido compartir:

```text
P1/P2/P3 counter
```

entre queries concurrentes.

---

# 202. Coroutine safety

```text
Coroutine A
└── DeleteBuilderState A

Coroutine B
└── DeleteBuilderState B
```

---

# 203. Request reset

No deberá requerirse limpiar Builder state global al terminar un request porque no deberá existir dicho estado global.

---

# 204. Fingerprints

Se distinguirán:

```text
builder fingerprint
normalized fingerprint
semantic fingerprint
planning fingerprint
compiled fingerprint
binding-shape fingerprint
execution fingerprint
```

---

# 205. Structural fingerprint

Podrá incluir:

```text
target
predicate structure
source relation structure
join structure
ordering structure
limit structure
returning structure
metadata affecting semantics
```

---

# 206. Runtime values excluded

Valores concretos:

```text
user id
token
timestamp
tenant id
```

no formarán parte del fingerprint estructural normal.

---

# 207. Semantic fingerprint

Podrá incorporar:

```text
resolved target identity
schema semantic fingerprint
resolved symbols
types
constraints
relations
referential metadata
capability requirements
policy versions
extension versions
```

---

# 208. Tenant schema identity

Si cambia la semántica del schema por tenant, deberá cambiar el semantic fingerprint.

---

# 209. Resource governance

Podrán existir límites:

```text
max predicates
max joins
max source relations
max subqueries
max CTEs
max parameters
max expression depth
max returning items
max graph nodes
```

---

# 210. Builder budget

```text
DeleteBuilderBudget
```

deberá impedir estructuras patológicas.

---

# 211. Semantic budget

Semantic Analysis tendrá sus propios budgets independientes.

---

# 212. Extensions

Podrán añadirse:

```text
custom delete sources
custom mutation hints
custom qualification nodes
custom returning modes
custom safety policies
custom capability requirements
```

---

# 213. Extension registry

Será:

```text
frozen after bootstrap
```

---

# 214. Extension descriptor

Deberá declarar:

```text
extension id
version
supported nodes
validation rules
semantic rules
optimizer support
planner support
compiler support
capabilities
portability
security classification
fingerprint impact
```

---

# 215. Extension conflict

Dos extensiones no podrán redefinir silenciosamente el mismo core semantic identifier.

---

# 216. Extension completeness

Un extension node que requiera compilation no podrá alcanzar Compiler sin soporte declarado.

---

# 217. Namespace recomendado

```text
VoltStack\Quantum\Database\Query\Builder\Delete
```

---

# 218. Estructura propuesta

```text
Query/
└── Builder/
    └── Delete/
        ├── Contract/
        │   ├── DeleteQueryBuilderInterface.php
        │   └── DeleteBuilderFinalizerInterface.php
        │
        ├── Core/
        │   ├── DeleteQueryBuilder.php
        │   ├── DeleteBuilderState.php
        │   ├── DeleteBuilderFinalizer.php
        │   └── DeleteBuilderSnapshot.php
        │
        ├── Target/
        │   ├── DeleteTarget.php
        │   ├── DeleteTargetBuilder.php
        │   └── DeleteTargetFactory.php
        │
        ├── Source/
        │   ├── DeleteSource.php
        │   ├── DeleteSourceSet.php
        │   └── DeleteSourceBuilder.php
        │
        ├── Join/
        │   ├── DeleteJoinBuilder.php
        │   └── DeleteJoinSet.php
        │
        ├── Predicate/
        │   └── DeletePredicateBuilder.php
        │
        ├── Returning/
        │   ├── DeleteReturningBuilder.php
        │   ├── ReturningSpecification.php
        │   └── DeleteResultExpectation.php
        │
        ├── Ordering/
        │   └── DeleteOrderingSpecification.php
        │
        ├── Limit/
        │   └── DeleteLimitSpecification.php
        │
        ├── Safety/
        │   ├── DeleteSafetyPolicy.php
        │   ├── UnboundedDeleteIntent.php
        │   └── DeleteCardinalityClassification.php
        │
        ├── Impact/
        │   ├── ReferentialImpactDescriptor.php
        │   ├── DirectMutationSet.php
        │   └── PotentialReferentialMutationSet.php
        │
        ├── Metadata/
        │   └── DeleteMetadataBuilder.php
        │
        ├── Extension/
        │   ├── DeleteBuilderExtension.php
        │   ├── DeleteExtensionDescriptor.php
        │   └── DeleteBuilderExtensionRegistry.php
        │
        ├── Diagnostic/
        │   └── DeleteBuilderDiagnostic.php
        │
        └── Exception/
            ├── DeleteQueryBuilderException.php
            ├── MissingDeleteTargetException.php
            ├── InvalidDeleteTargetException.php
            ├── InvalidDeletePredicateException.php
            ├── UnboundedDeleteRejectedException.php
            └── DeleteBuilderAlreadyFinalizedException.php
```

---

# 219. Ownership Matrix

| Concepto | Owner |
|---|---|
| Fluent DELETE API | DeleteQueryBuilder |
| Temporary state | DeleteBuilderState |
| Target declaration | Delete Target System |
| Predicates | Predicate System |
| Parameters | Parameter System |
| Runtime values | BindingSet |
| Source relations | Relation System |
| Join semantics | Relation/Join Resolution |
| Schema identity | Schema-Aware Resolution |
| Predicate typing | Type Inference |
| Cardinality facts | Constraint Analysis |
| Referential metadata | Schema/Semantic Engine |
| Semantic graph | Semantic Query Graph |
| Unbounded delete policy | Delete Safety Policy |
| Soft delete | Soft Delete System |
| ORM cascade | ORM/Persistence |
| DB cascade | Database server/schema |
| Bulk deletion | Bulk Delete System |
| SQL syntax | Compiler/Dialect |
| Connection | Connection System |
| Transaction | Transaction System |
| Native binding | Driver |
| Entity lifecycle | ORM |
| Cache invalidation | Cache Integration |
| Audit | Security/Audit Integration |

---

# 220. Architectural Invariants

## DB-DEL-001

`DeleteQueryBuilder` nunca generará SQL.

## DB-DEL-002

Nunca concatenará `DELETE`.

## DB-DEL-003

Nunca concatenará `FROM`.

## DB-DEL-004

Nunca concatenará `USING`.

## DB-DEL-005

Nunca generará placeholders físicos.

## DB-DEL-006

Nunca realizará identifier quoting.

## DB-DEL-007

Nunca realizará native parameter binding.

## DB-DEL-008

Nunca dependerá de PDO.

## DB-DEL-009

Construir DELETE nunca realizará I/O.

## DB-DEL-010

El mutation target será explícito.

## DB-DEL-011

Una source relation no será automáticamente mutation target.

## DB-DEL-012

V1 favorecerá un único mutation target.

## DB-DEL-013

Multi-target delete requerirá semántica explícita.

## DB-DEL-014

Predicates reutilizarán Predicate System.

## DB-DEL-015

Runtime values permanecerán fuera del AST.

## DB-DEL-016

Collection parameters no serán expandidos prematuramente.

## DB-DEL-017

Subqueries serán artifacts finalizados.

## DB-DEL-018

No se retendrán child builders mutables.

## DB-DEL-019

DELETE sin WHERE será identificado como potencialmente unbounded.

## DB-DEL-020

La presencia de WHERE no implicará boundedness.

## DB-DEL-021

RawPredicate no implicará boundedness.

## DB-DEL-022

Constraint Analysis podrá refinar boundedness.

## DB-DEL-023

Unique key coverage podrá derivar `AT_MOST_ONE`.

## DB-DEL-024

Partial unique-key coverage no implicará single row.

## DB-DEL-025

Unbounded mutation policy será explícita.

## DB-DEL-026

Safety override no implicará autorización.

## DB-DEL-027

DELETE no se convertirá automáticamente en TRUNCATE.

## DB-DEL-028

DELETE será semánticamente distinto de TRUNCATE.

## DB-DEL-029

DELETE será distinto de soft delete.

## DB-DEL-030

Soft delete pertenecerá a su propio sistema.

## DB-DEL-031

Direct Query Builder DELETE representará physical deletion intent.

## DB-DEL-032

ORM podrá transformar entity deletion según metadata.

## DB-DEL-033

Builder no conocerá ORM soft-delete metadata.

## DB-DEL-034

Builder no ejecutará DB cascades.

## DB-DEL-035

Builder no ejecutará ORM cascades.

## DB-DEL-036

DB cascade y ORM cascade permanecerán separados.

## DB-DEL-037

Direct DELETE no emitirá fake child entity events.

## DB-DEL-038

Referential impact podrá analizarse semánticamente.

## DB-DEL-039

Partial schema no significará ausencia de referential impact.

## DB-DEL-040

Source relations serán explícitas.

## DB-DEL-041

`using()` será semántico, no sintáctico.

## DB-DEL-042

JOIN DELETE será capability-driven.

## DB-DEL-043

Builder no decidirá JOIN vs USING vs EXISTS.

## DB-DEL-044

Target-row identity será preservada.

## DB-DEL-045

Multiple source matches no implicarán múltiples target deletions.

## DB-DEL-046

Semi-join transformations pertenecerán al Optimizer.

## DB-DEL-047

Correlations serán resueltas semánticamente.

## DB-DEL-048

CTEs reutilizarán CTE System.

## DB-DEL-049

RETURNING será semántico.

## DB-DEL-050

Builder no generará `RETURNING`.

## DB-DEL-051

Sensitive RETURNING podrá ser restringido.

## DB-DEL-052

Mutation ORDER BY será capability-driven.

## DB-DEL-053

Mutation LIMIT será capability-driven.

## DB-DEL-054

LIMIT emulation deberá preservar atomicity.

## DB-DEL-055

LIMIT emulation deberá preservar concurrency semantics.

## DB-DEL-056

Limited DELETE no sustituirá Bulk Delete System.

## DB-DEL-057

Builder no será Bulk Delete Coordinator.

## DB-DEL-058

DELETE no será automáticamente retry-safe.

## DB-DEL-059

PK predicate no garantizará exactly-once semantics.

## DB-DEL-060

Retry classification considerará observable effects.

## DB-DEL-061

Builder no iniciará transacciones.

## DB-DEL-062

Builder no almacenará active Transaction.

## DB-DEL-063

Builder no seleccionará physical connection.

## DB-DEL-064

DELETE declarará WRITE intent.

## DB-DEL-065

Dependency Set será distinto de Mutation Set.

## DB-DEL-066

Direct Mutation Set será distinto de Referential Mutation Set.

## DB-DEL-067

Mutation metadata podrá alimentar cache invalidation.

## DB-DEL-068

Builder no invalidará caches.

## DB-DEL-069

Builder no emitirá execution events.

## DB-DEL-070

Direct DELETE no emitirá ORM lifecycle events.

## DB-DEL-071

ORM UnitOfWork será externo.

## DB-DEL-072

IdentityMap será externo.

## DB-DEL-073

Builder no sincronizará IdentityMap.

## DB-DEL-074

Authorization será integración opcional.

## DB-DEL-075

Multitenancy será integración opcional.

## DB-DEL-076

Security predicates serán explícitos.

## DB-DEL-077

Mandatory predicates conservarán provenance.

## DB-DEL-078

Builder no consultará global current tenant.

## DB-DEL-079

Sensitive parameters serán redactables.

## DB-DEL-080

Raw predicates serán explícitos.

## DB-DEL-081

Raw no significará trusted.

## DB-DEL-082

Dynamic identifiers pasarán por identifier policy.

## DB-DEL-083

Builder no resolverá schema mediante I/O.

## DB-DEL-084

Semantic Analysis podrá funcionar offline.

## DB-DEL-085

Type inference reutilizará Query Type System.

## DB-DEL-086

Domain type identity será preservada.

## DB-DEL-087

Semantic Graph será authoritative meaning.

## DB-DEL-088

Optimizer no reinterpretará strings desde cero.

## DB-DEL-089

No-op mutation planning requerirá equivalencia segura.

## DB-DEL-090

Planner elegirá physical delete strategy.

## DB-DEL-091

Physical Plan no será SQL.

## DB-DEL-092

Compiler será responsable de dialect syntax.

## DB-DEL-093

Executor será responsable de execution.

## DB-DEL-094

Driver será responsable de native binding.

## DB-DEL-095

Capabilities reemplazarán vendor conditionals.

## DB-DEL-096

Builder no contendrá `if mysql`.

## DB-DEL-097

Builder no contendrá `if pgsql`.

## DB-DEL-098

Builder no contendrá `if sqlite`.

## DB-DEL-099

Validation estará separada por fases.

## DB-DEL-100

Construction Validation no sustituirá Semantic Validation.

## DB-DEL-101

Safety Validation no sustituirá Authorization.

## DB-DEL-102

Semantic Validation no sustituirá server constraints.

## DB-DEL-103

Finalized DeleteQueryModel será immutable.

## DB-DEL-104

DeleteBuilderState será operation-scoped.

## DB-DEL-105

No existirá global current Delete Builder.

## DB-DEL-106

No existirán static parameter counters compartidos.

## DB-DEL-107

Builder copies tendrán state isolation.

## DB-DEL-108

Bindings no se compartirán entre copies.

## DB-DEL-109

Builder será persistent-runtime safe.

## DB-DEL-110

Builder será coroutine-safe.

## DB-DEL-111

No habrá request state leakage.

## DB-DEL-112

No habrá worker state leakage.

## DB-DEL-113

No habrá coroutine state leakage.

## DB-DEL-114

Structural fingerprint excluirá runtime values.

## DB-DEL-115

Semantic fingerprint incluirá schema identity relevante.

## DB-DEL-116

Extension registry será frozen.

## DB-DEL-117

Extensions no sobrescribirán core semantics silenciosamente.

## DB-DEL-118

Builder podrá finalizarse sin ejecución.

## DB-DEL-119

La API Laravel-like será sólo una capa ergonómica.

## DB-DEL-120

Las fronteras internas tendrán prioridad sobre shortcuts de implementación.

---

# 221. Anti-patterns

## 221.1 SQL directo

```php
$sql = "DELETE FROM {$table} WHERE {$condition}";
```

**Rechazado.**

---

## 221.2 PDO en Builder

```php
$this->pdo->prepare(...);
```

**Rechazado.**

---

## 221.3 Vendor detection

```php
if ($driver === 'mysql') {
    // DELETE JOIN
}
```

**Rechazado.**

---

## 221.4 Soft-delete magic

```php
if ($model->softDeletes()) {
    return $this->update(...);
}
```

dentro del Query Builder core.

**Rechazado.**

---

## 221.5 Automatic truncate

```text
DELETE without WHERE
→ TRUNCATE
```

**Rechazado.**

---

## 221.6 Fake cascade events

```text
DB cascade
→ emit ORM Deleted events
```

**Rechazado.**

---

## 221.7 WHERE means safe

```text
hasPredicate()
→ bounded
```

**Rechazado.**

---

## 221.8 Blind retry

```text
DELETE failed by network
→ execute again
```

**Rechazado.**

---

## 221.9 Hidden tenant

```php
$tenant = Tenant::current();
```

dentro del Builder.

**Rechazado.**

---

## 221.10 IdentityMap synchronization

```php
foreach ($identityMap as $entity) {
    ...
}
```

dentro del Delete Builder.

**Rechazado.**

---

# 222. Ejemplo completo — DELETE por PK

```php
$affected = DB::table('users')
    ->label('users.remove')
    ->where('id', $userId)
    ->delete();
```

Builder:

```text
DeleteBuilderState
├── target
│   └── users
├── predicate
│   └── id = P1
├── parameters
│   └── P1
└── metadata
    ├── QueryLabel: users.remove
    ├── QueryIntent: WRITE
    └── ConnectionIntent: WRITE
```

---

# 223. Schema resolution

```text
users.id
├── UUID
├── PRIMARY KEY
├── UNIQUE
└── NOT NULL
```

Semantic Analysis deriva:

```text
P1
→ UserId/UUID compatible

Predicate
→ unique-key constrained

Mutation Cardinality
→ AT_MOST_ONE
```

---

# 224. Semantic artifact

```text
SemanticDeleteArtifact
│
├── Target
│   └── users
│
├── Predicate
│   └── users.id = P1
│
├── Type
│   └── P1 : UserId
│
├── Constraint
│   └── users.id PRIMARY KEY
│
├── DirectMutationSet
│   └── users
│
├── DependencySet
│   ├── users
│   └── users.id
│
└── Cardinality
    └── AT_MOST_ONE
```

---

# 225. Ejemplo — DELETE con relación

```php
DB::delete()
    ->from('users as u')
    ->using('accounts as a')
    ->whereColumn('a.user_id', 'u.id')
    ->where('a.suspended', true)
    ->execute();
```

Representación:

```text
DeleteQuery
│
├── MutationTarget
│   └── users AS u
│
├── SourceRelation
│   └── accounts AS a
│
├── RelationPredicate
│   └── a.user_id = u.id
│
└── Filter
    └── a.suspended = TRUE
```

---

# 226. Semantic Graph

```text
accounts
   │
   ├── user_id ─────────────┐
   │                        │
   └── suspended            │
          │                 │
          ▼                 ▼
       Predicate        users.id
          │                 │
          └──────┬──────────┘
                 ▼
          Qualification
                 │
                 ▼
              users
                 │
                 ▼
           DeleteTarget
```

---

# 227. Physical planning

Podrá resultar en:

```text
SemanticDelete
      │
      ▼
Planner
      │
      ├── UsingDeletePlan
      ├── JoinedDeletePlan
      ├── ExistsDeletePlan
      └── Unsupported
```

---

# 228. Ejemplo — Referential impact

Schema:

```text
users
  ▲
  │ FK CASCADE
orders.user_id

users
  ▲
  │ FK SET NULL
audit_entries.actor_id
```

Delete:

```text
users.id = P1
```

Semantic impact:

```text
DirectMutation
└── users

PotentialReferentialMutations
├── orders
│   └── CASCADE
└── audit_entries
    └── SET_NULL
```

---

# 229. Important distinction

```text
PotentialReferentialMutation
≠
ORM cascade plan
≠
actual affected row count
```

---

# 230. Ejemplo — soft delete

Entidad:

```text
User
└── SoftDeletePolicy
    └── deleted_at
```

Llamada ORM:

```php
$user->delete();
```

flujo:

```text
Entity API
   │
   ▼
ORM Metadata
   │
   ▼
SoftDeletePolicy
   │
   ▼
UpdateQueryModel
├── SET deleted_at = CurrentTimestamp
└── WHERE id = P1
```

No:

```text
DeleteQueryModel
```

---

# 231. Force delete

```php
$user->forceDelete();
```

podrá seguir:

```text
Entity API
   │
   ▼
Persistence Engine
   │
   ▼
DeleteQueryModel
   │
   ▼
Query Engine
```

---

# 232. Ejemplo — limited delete

```php
DB::table('logs')
    ->where('archived', true)
    ->orderBy('created_at')
    ->orderBy('id')
    ->limit(1000)
    ->delete();
```

Semántica:

```text
Delete
├── target: logs
├── predicate
│   └── archived = TRUE
├── ordering
│   ├── created_at ASC
│   └── id ASC
└── limit
    └── 1000
```

---

# 233. Capability analysis

Podrá producir:

```text
Requirements
├── DELETE
├── DELETE_LIMIT
└── DELETE_ORDERING
```

---

# 234. Planner

```text
Target supports native?
        │
   ┌────┴────┐
   │         │
  yes        no
   │         │
   ▼         ▼
Native     Can safe key-set
Delete     strategy preserve
Plan       semantics?
              │
         ┌────┴────┐
         │         │
        yes        no
         │         │
         ▼         ▼
      KeySet     Unsupported
      Plan
```

---

# 235. Pipeline completo

```text
Developer
   │
   ▼
DeleteQueryBuilder
   │
   ├── TargetBuilder
   ├── PredicateBuilder
   ├── SourceBuilder
   ├── JoinBuilder
   ├── ReturningBuilder
   ├── OrderingBuilder
   ├── LimitBuilder
   ├── ParameterRegistry
   ├── SafetyMetadata
   └── QueryMetadata
          │
          ▼
DeleteBuilderFinalizer
          │
          ├───────────────┐
          ▼               ▼
 DeleteQueryModel      BindingSet
          │
          ▼
    DeleteQueryNode
          │
          ▼
     Normalization
          │
          ▼
       Validation
          │
          ▼
    Semantic Analysis
          │
          ├── target resolution
          ├── source resolution
          ├── symbol resolution
          ├── type inference
          ├── relation resolution
          ├── correlation analysis
          ├── constraint analysis
          ├── cardinality analysis
          ├── referential impact
          ├── mutation/dependency sets
          ├── safety classification
          ├── capability requirements
          └── semantic graph
          │
          ▼
 SemanticQueryArtifact
          │
          ▼
       Optimizer
          │
          ▼
        Planner
          │
          ├── native strategy
          ├── join/using strategy
          ├── exists strategy
          ├── limited strategy
          ├── returning strategy
          └── binding layout
          │
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
 DeleteExecutionResult
```

---

# 236. Fórmula maestra

```text
DeleteQueryBuilder
=
Mutation Target
+
Predicate
+
Source Relations
+
Join Semantics
+
Ordering Intent
+
Limit Intent
+
Returning Intent
+
Safety Intent
+
Parameter Definitions
+
Query Metadata
```

---

# 237. Fórmula del artifact

```text
DeleteQueryArtifact
=
Immutable DeleteQueryModel
+
ParameterDefinitionSet
+
QueryMetadata
```

---

# 238. Fórmula semántica

```text
Delete Meaning
=
Resolved Mutation Target
+
Resolved Qualification Predicate
+
Resolved Source Relations
+
Resolved Correlations
+
Type Information
+
Constraint Knowledge
+
Mutation Cardinality
+
Referential Impact
+
Direct Mutation Set
+
Potential Indirect Mutation Set
+
Dependency Set
+
Capability Requirements
+
Provenance
```

---

# 239. Fórmula de seguridad

```text
Safe Delete
=
Structured Target
+
Parameterized Predicates
+
Semantic Boundedness Analysis
+
Explicit Unbounded Intent
+
Authorization Integration
+
Tenant Isolation Integration
+
Sensitive Data Redaction
+
Referential Impact Awareness
+
Retry Awareness
+
Resource Governance
```

---

# 240. Fórmula de portabilidad

```text
Portable Delete
=
Semantic Delete Model
-
Vendor DELETE Syntax Assumptions
+
Capability Requirements
+
Logical Qualification Semantics
+
Planner Strategies
+
Dialect Compilation
```

---

# 241. Fórmula de persistent-runtime safety

```text
Persistent-Safe Delete Builder
=
Immutable Shared Services
+
Frozen Registries
+
Operation-Scoped Builder State
+
Operation-Scoped Parameter State
+
Operation-Scoped Binding State
+
No Global Current Builder
+
No Runtime Resource Retention
+
Deterministic Finalization
```

---

# 242. Diseño definitivo

La API pública podrá ser tan sencilla como:

```php
DB::table('users')
    ->where('id', $id)
    ->delete();
```

mientras internamente:

```text
DeleteQueryBuilder
       │
       ▼
DeleteQueryModel
       │
       ▼
DeleteQueryNode
       │
       ▼
Semantic Query Engine
       │
       ▼
SemanticDeleteArtifact
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
```

---

# 243. Principio final

La separación central será:

```text
Builder
   │
   ▼
"What rows does the developer intend to delete?"
   │
   ▼
Semantic Engine
   │
   ▼
"What target, predicates, relations,
constraints and effects does that mean?"
   │
   ▼
Safety / Constraint Analysis
   │
   ▼
"How dangerous or bounded is the mutation?"
   │
   ▼
Optimizer
   │
   ▼
"What can be transformed safely?"
   │
   ▼
Planner
   │
   ▼
"How can this deletion be performed
on the target platform?"
   │
   ▼
Compiler
   │
   ▼
"What SQL represents that plan?"
   │
   ▼
Executor
   │
   ▼
"Execute the mutation."
```

---

# 244. Conclusión

`DeleteQueryBuilder` será un constructor semántico de intención de eliminación, no un generador de SQL ni un sistema ORM de lifecycle.

Permitirá representar:

```text
simple DELETE
predicate-based DELETE
PK/unique constrained DELETE
join-aware DELETE
source-relation DELETE
subquery DELETE
CTE DELETE
RETURNING
ordered/limited DELETE
unbounded DELETE intent
referential impact
security metadata
audit metadata
transaction requirements
```

sin acoplarse a:

```text
PDO
SQL dialect syntax
physical connections
EntityManager
UnitOfWork
IdentityMap
SoftDelete implementation
Authorization
Authentication
Multitenancy
HTTP
```

La frontera más importante será:

```text
Physical DELETE
        │
        ├── DeleteQueryBuilder
        │
        ▼
  DeleteQueryModel

Soft DELETE
        │
        ├── SoftDelete System
        │
        ▼
  UpdateQueryModel

Entity DELETE
        │
        ├── ORM / UnitOfWork
        │
        ├── lifecycle policies
        │
        ├── cascade policies
        │
        ▼
  Persistence Plan
        │
        ├── UpdateQueryModel
        │      or
        └── DeleteQueryModel
```

Con ello VoltStack podrá combinar:

```text
Laravel-like developer experience
+
Doctrine-style persistence boundaries
+
semantic query analysis
+
mutation safety
+
constraint awareness
+
multi-platform portability
+
persistent-runtime safety
```

sin mezclar responsabilidades.

> **`DeleteQueryBuilder` declara qué filas deben dejar de existir físicamente. Semantic Analysis determina exactamente qué significa esa intención. Constraint Analysis determina sus propiedades y alcance. Planner decide cómo realizarla. Compiler genera la representación SQL y Executor efectúa la mutación.**

---

# 245. Estado del bloque Query Builder

Con este documento quedan definidos los cuatro builders fundamentales:

```text
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
        │
        ├── 44_DATABASE_SELECT_QUERY_BUILDER.md
        ├── 45_DATABASE_INSERT_QUERY_BUILDER.md
        ├── 46_DATABASE_UPDATE_QUERY_BUILDER.md
        └── 47_DATABASE_DELETE_QUERY_BUILDER.md
```

La arquitectura común queda:

```text
                    Query Builder Architecture
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
          SELECT            INSERT            UPDATE
            │                 │                 │
            └────────────┬────┴─────┬──────────┘
                         │          │
                         ▼          ▼
                       DELETE    Extensions
                         │
                         ▼
                    Query Model / AST
                         │
                         ▼
                  Semantic Query Engine
```

Los siguientes documentos amplían capacidades composables compartidas por estos builders.

---

# 246. Siguiente documento

```text
48_DATABASE_JOIN_QUERY_BUILDER.md
```

El siguiente documento deberá definir la capa de construcción de JOINs reutilizable por SELECT y, cuando la semántica/capabilities lo permitan, UPDATE y DELETE:

```text
JoinQueryBuilder
JoinClauseBuilder
JoinType
JoinTarget
JoinAlias
JoinPredicate
ON
USING
CROSS JOIN
INNER JOIN
LEFT JOIN
RIGHT JOIN
FULL JOIN
LATERAL
subquery joins
CTE joins
table-function joins
join scopes
join parameterization
join composition
nested joins
join groups
relationship evidence
join metadata
semantic handoff
capability requirements
extension model
persistent-runtime safety
```

manteniendo la frontera:

```text
Join Builder
      │
      ▼
Join Structure
      │
      ▼
Relation & Join Resolution
      │
      ▼
Semantic Join
      │
      ▼
Constraint Analysis
      │
      ▼
Semantic Query Graph
      │
      ▼
Optimizer
      │
      ▼
Join Optimization
      │
      ▼
Planner
      │
      ▼
Physical Join Strategy
      │
      ▼
Compiler
```