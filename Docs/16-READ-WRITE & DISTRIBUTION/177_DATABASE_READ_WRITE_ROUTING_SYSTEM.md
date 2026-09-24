# 177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md

# VoltStack Quantum Database
## Database Read/Write Routing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 177 — Database Read/Write Routing System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md`  
**Siguiente documento:** `178_DATABASE_REPLICA_SYSTEM.md`

---

# 1. Propósito

`Database Read/Write Routing System` define la arquitectura responsable de decidir **qué conexión elegible debe utilizar una operación de base de datos** una vez que VoltStack conoce:

- el dominio lógico de ejecución;
- la intención de la operación;
- los roles requeridos;
- las restricciones transaccionales;
- los requisitos de consistencia;
- la afinidad existente;
- los endpoints candidatos;
- las políticas activas.

El documento anterior estableció:

```text
Operation
→ ConnectionIntent
→ Required Capability
→ ConnectionRole
→ Candidate Set
```

Este documento añade:

```text
Candidate Set
      │
      ▼
Routing Context
      │
      ▼
Routing Constraints
      │
      ▼
Routing Policies
      │
      ▼
Candidate Evaluation
      │
      ▼
Routing Decision
      │
      ▼
Selected Endpoint
```

La regla central será:

> **El Read/Write Routing System de VoltStack seleccionará conexiones a partir de intención semántica, dominio de ejecución, consistencia, afinidad, estado transaccional y capacidades verificables; nunca decidirá el destino únicamente inspeccionando SQL ni utilizará una replica por el simple hecho de que una operación sea de lectura.**

Formalmente:

```text
RoutingDecision
=
f(
    ExecutionDomain,
    ConnectionRequirement,
    TransactionContext,
    ConsistencyRequirement,
    Affinity,
    CandidateSet,
    RoutingPolicy,
    RuntimeEvidence
)
```

---

# 2. Regla arquitectónica principal

Debe mantenerse:

```text
Connection Discovery
≠
Role Resolution
≠
Routing
≠
Load Balancing
≠
Failover
≠
Replica Management
≠
Connection Acquisition
```

Responsabilidades:

```text
Read/Write Connection System
    → define conexiones y capacidades

Read/Write Routing System
    → decide qué candidato elegible debe utilizarse

Replica System
    → modela replicas

Load Balancing System
    → distribuye carga entre candidatos equivalentes

Failover System
    → responde a pérdida de candidatos

Connection Manager
    → adquiere/libera conexiones físicas
```

---

# 3. Objetivos

El sistema deberá proporcionar:

1. routing determinista y explicable;
2. routing por intención semántica;
3. routing consciente de transaction;
4. routing consciente de consistencia;
5. soporte para `READ`;
6. soporte para `WRITE`;
7. soporte para `AUTHORITATIVE_READ`;
8. soporte para locking reads;
9. soporte para schema/admin operations;
10. filtros de candidatos;
11. routing policies;
12. routing constraints;
13. affinity;
14. transaction pinning;
15. fallback controlado;
16. read-after-write integration;
17. soporte futuro para replica lag;
18. soporte futuro para failover;
19. soporte futuro para load balancing;
20. soporte futuro para sharding;
21. soporte futuro para partition routing;
22. routing diagnostics;
23. telemetry;
24. persistent-runtime safety;
25. testing;
26. extensibilidad.

---

# 4. No objetivos

Este documento no implementa en profundidad:

```text
Replica discovery
Replica lag measurement
Failover orchestration
Load balancing algorithms
Shard selection
Partition selection
Distributed transactions
```

Se desarrollarán respectivamente en:

```text
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
182_DATABASE_LOAD_BALANCING_SYSTEM.md
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

---

# 5. Problema fundamental

Una arquitectura ingenua puede hacer:

```php
if ($query->isSelect()) {
    return $readConnection;
}

return $writeConnection;
```

VoltStack no deberá utilizar este modelo.

Una consulta puede ser:

```sql
SELECT * FROM users WHERE id = ? FOR UPDATE
```

y requerir:

```text
WRITE-capable
+
transactional
+
authoritative
```

Otra consulta:

```sql
SELECT *
FROM users
WHERE id = ?
```

después de una escritura reciente puede necesitar:

```text
AUTHORITATIVE_READ
```

aunque sea un `SELECT`.

Por tanto:

```text
SQL Verb
≠
Connection Routing Intent
```

---

# 6. Pipeline general

```text
Query / Persistence Operation
            │
            ▼
   Semantic Classification
            │
            ▼
 ConnectionRequirement
            │
            ▼
     RoutingContext
            │
            ▼
 Transaction Constraint
            │
            ▼
   Domain Constraint
            │
            ▼
    Role Constraint
            │
            ▼
 Consistency Constraint
            │
            ▼
  Affinity Constraint
            │
            ▼
Candidate Eligibility Filter
            │
            ▼
    Routing Policies
            │
            ▼
Candidate Ranking/Selection
            │
            ▼
    RoutingDecision
            │
            ▼
    ConnectionManager
            │
            ▼
    ConnectionLease
```

---

# 7. RoutingContext

El contexto central será un objeto immutable:

```php
final readonly class RoutingContext
{
    public function __construct(
        public RoutingOperationId $operationId,
        public DatabaseExecutionDomain $domain,
        public ConnectionRequirement $requirement,
        public ?TransactionRoutingContext $transaction,
        public ReadConsistencyRequirement $consistency,
        public ConnectionAffinity $affinity,
        public RoutingPolicySet $policies,
        public RoutingEvidenceSnapshot $evidence,
    ) {}
}
```

---

# 8. RoutingContext no ejecuta

`RoutingContext` no deberá:

- abrir conexiones;
- ejecutar queries;
- consultar entities;
- hacer `flush()`;
- cambiar transaction state;
- medir replica lag directamente;
- hacer health probes;
- mutar pools.

Es únicamente:

```text
immutable routing input
```

---

# 9. RoutingOperationId

Cada decisión tendrá identidad:

```php
final readonly class RoutingOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Permitirá correlacionar:

```text
Query
→ Routing
→ Connection acquisition
→ Execution
→ Telemetry
```

---

# 10. RoutingDecision

Resultado:

```php
final readonly class RoutingDecision
{
    public function __construct(
        public RoutingOperationId $operationId,
        public DatabaseConnectionId $connection,
        public EndpointId $endpoint,
        public ConnectionRole $role,
        public RoutingDecisionReason $reason,
        public RoutingDecisionMetadata $metadata,
    ) {}
}
```

---

# 11. Routing decision ≠ connection

Regla:

```text
RoutingDecision
≠
ConnectionLease
```

El router selecciona.

El `ConnectionManager` adquiere.

---

# 12. Routing decision ≠ execution

También:

```text
RoutingDecision
≠
QueryExecution
```

---

# 13. RoutingEngine

Contrato principal:

```php
interface ReadWriteRoutingEngine
{
    public function route(
        RoutingContext $context,
        ConnectionCandidateSet $candidates,
    ): RoutingDecision;
}
```

---

# 14. Responsabilidad del router

Debe responder:

> ¿Cuál de los candidatos actualmente elegibles satisface mejor las restricciones y políticas de esta operación?

No:

> ¿Cómo ejecuto la query?

---

# 15. Restricciones duras y preferencias

VoltStack distinguirá:

```text
Hard Constraints
```

de:

```text
Soft Preferences
```

---

# 16. Hard constraint

Una hard constraint nunca podrá violarse para producir una decisión.

Ejemplos:

```text
correct database
correct tenant
correct shard
WRITE capability
transaction pinned endpoint
required consistency
```

---

# 17. Soft preference

Ejemplos:

```text
prefer reader
prefer same region
prefer lower load
prefer sticky endpoint
```

Una preferencia puede sacrificarse.

---

# 18. RoutingConstraint

Contrato:

```php
interface RoutingConstraint
{
    public function evaluate(
        ConnectionCandidate $candidate,
        RoutingContext $context,
    ): RoutingConstraintResult;
}
```

---

# 19. RoutingConstraintResult

```php
enum RoutingConstraintStatus
{
    case ACCEPT;
    case REJECT;
    case UNKNOWN;
}
```

---

# 20. UNKNOWN

Para restricciones críticas:

```text
UNKNOWN
```

deberá tratarse conservadoramente.

Especialmente para:

```text
WRITE authority
transaction affinity
execution domain
```

---

# 21. Candidate elimination

Flujo:

```text
Candidates
    │
    ▼
Domain filter
    │
    ▼
Role filter
    │
    ▼
Transaction filter
    │
    ▼
Consistency filter
    │
    ▼
Health/availability filter
    │
    ▼
Eligible Candidates
```

---

# 22. RoutingPolicy

Una policy podrá influir en ranking/selección:

```php
interface RoutingPolicy
{
    public function apply(
        RoutingCandidateSet $candidates,
        RoutingContext $context,
    ): RoutingCandidateSet;
}
```

---

# 23. Constraint ≠ policy

```text
Constraint
→ defines what is allowed

Policy
→ influences what is preferred
```

---

# 24. Example

```text
WRITE capability
```

es constraint.

```text
prefer local writer
```

puede ser policy.

---

# 25. Routing stages

Arquitectura propuesta:

```text
1. Context Resolution
2. Transaction Resolution
3. Domain Validation
4. Intent Resolution
5. Role Eligibility
6. Consistency Eligibility
7. Affinity Evaluation
8. Runtime Evidence Evaluation
9. Candidate Ranking
10. Selection
11. Decision Validation
12. Decision Publication
```

---

# 26. Stage 1 — Context Resolution

Se construye:

```text
RoutingContext
```

a partir de:

```text
QueryExecutionContext
TransactionContext
DatabaseExecutionDomain
ConnectionRequirement
scope state
```

---

# 27. Stage 2 — Transaction Resolution

Antes de cualquier routing normal se pregunta:

```text
Is there an active transaction?
```

---

# 28. Transaction precedence

Si existe:

```text
ACTIVE transaction
+
PinnedConnectionLease
```

el routing normal queda esencialmente suspendido.

```text
Operation
    │
    ▼
TransactionContext
    │
    ▼
Pinned Lease
```

---

# 29. Transaction routing

Formalmente:

```text
ActiveTransaction(T)
⇒
Route(O) = PinnedConnection(T)
```

si:

```text
Requirements(O)
⊆
Capabilities(PinnedConnection(T))
```

---

# 30. Transaction mismatch

Si no:

```text
TransactionConnectionRoleMismatchException
```

---

# 31. No replica routing in transaction

Una query `READ` dentro de una write transaction no deberá pasar por:

```text
Replica Selector
```

---

# 32. Read-only transaction

Si la transaction está explícitamente configurada como read-only:

```text
READ
```

puede permanecer sobre reader.

Una escritura será rechazada.

---

# 33. Savepoints

Savepoints:

```text
do not create new routing domain
```

---

# 34. Nested transaction JOIN

Una nested transaction que participa en la misma physical transaction utiliza:

```text
same pinned connection
```

---

# 35. REQUIRES_NEW

Si en el futuro una política nested crea una nueva physical transaction:

```text
outer context suspended
↓
new routing context
↓
new transaction connection
```

---

# 36. Stage 3 — Domain Validation

Todos los candidatos deben pertenecer al:

```text
DatabaseExecutionDomain
```

correcto.

---

# 37. Domain dimensions

Puede incluir:

```text
logical database
tenant
shard
partition
cluster
```

según extensiones instaladas.

---

# 38. No cross-domain routing

Nunca:

```text
tenant A query
→ tenant B replica
```

---

# 39. Domain equality

Deberá existir un mecanismo explícito:

```php
interface DatabaseExecutionDomainMatcher
{
    public function matches(
        DatabaseExecutionDomain $required,
        DatabaseExecutionDomain $candidate,
    ): bool;
}
```

---

# 40. Stage 4 — Intent Resolution

El sistema recibe:

```text
ConnectionRequirement
```

del documento 176.

Ejemplos:

```text
READ
WRITE
AUTHORITATIVE_READ
```

---

# 41. Query semantic source

Preferentemente:

```text
Query AST
→ Semantic Query Model
→ Execution Plan
→ ConnectionRequirement
```

---

# 42. Query router no analiza ORM

El router no deberá recibir:

```text
User entity
Repository
UnitOfWork
```

para decidir.

---

# 43. Persistence writes

Persistence Engine ya habrá producido:

```text
WRITE requirement
```

---

# 44. Unknown intent

Si llega:

```text
UNKNOWN
```

se aplica política segura:

```text
REQUIRE_WRITE
```

o:

```text
REJECT
```

---

# 45. Stage 5 — Role Eligibility

Se eliminan candidatos incapaces de satisfacer el role requerido.

---

# 46. READ

Orden preferido típico:

```text
dedicated READ
↓
READ_WRITE
↓
WRITE
```

si fallback está permitido.

---

# 47. WRITE

Solo:

```text
WRITE
READ_WRITE
```

con write capability efectiva.

---

# 48. AUTHORITATIVE_READ

No deberá equivaler automáticamente a:

```text
any reader
```

La política de autoridad decidirá.

---

# 49. Locking read

```text
READ_FOR_UPDATE
```

se normalizará hacia:

```text
WRITE-capable
+
transaction-capable
```

---

# 50. Role fallback

El fallback deberá ser explícito:

```text
READ
→ WRITE
```

puede permitirse.

```text
WRITE
→ READ
```

queda prohibido.

---

# 51. Stage 6 — Consistency Eligibility

Una lectura puede tener:

```text
EVENTUAL
SESSION
AUTHORITATIVE
```

---

# 52. EVENTUAL

Puede permitir:

```text
replica candidates
```

sin exigir conocimiento de la escritura más reciente.

---

# 53. SESSION

Debe preservar garantías de lectura posteriores a escrituras relevantes del mismo scope lógico.

---

# 54. AUTHORITATIVE

Exige una fuente reconocida por la topología/policy como autoritativa.

---

# 55. Consistency filter

Contrato:

```php
interface ReadConsistencyConstraint
{
    public function evaluate(
        ConnectionCandidate $candidate,
        ReadConsistencyRequirement $requirement,
        RoutingContext $context,
    ): RoutingConstraintResult;
}
```

---

# 56. Consistency evidence

No deberá inventarse.

Si una replica no puede demostrar que satisface cierta posición de consistencia:

```text
UNKNOWN
```

no será tratado como:

```text
SATISFIED
```

---

# 57. Replica lag integration

El documento 179 añadirá evidencia como:

```text
lag duration
replication position
freshness
measurement timestamp
confidence
```

---

# 58. Consistency ≠ freshness guess

No:

```text
replica responded quickly
→ must be current
```

---

# 59. Stage 7 — Affinity Evaluation

Affinity permite expresar:

```text
prefer/require previous routing target
```

---

# 60. Affinity modes

Base:

```text
NONE
ROLE
ENDPOINT
TRANSACTION
```

---

# 61. ROLE affinity

Ejemplo:

```text
stick to WRITE role
```

pero no necesariamente al mismo endpoint.

---

# 62. ENDPOINT affinity

Preferencia o requisito hacia:

```text
EndpointId
```

con strength explícito.

---

# 63. TRANSACTION affinity

Hard constraint:

```text
exact pinned connection
```

---

# 64. Affinity strength

Modelo:

```php
enum ConnectionAffinityStrength
{
    case PREFER;
    case REQUIRE;
}
```

---

# 65. Prefer affinity

Si el endpoint ya no es elegible:

```text
continue with other eligible candidate
```

---

# 66. Required affinity

Si no está disponible:

```text
routing fails
```

---

# 67. Sticky routing

Sticky read-after-write podrá generar:

```text
ROLE affinity = WRITE
```

durante un scope.

---

# 68. Sticky ≠ transaction

No confundir:

```text
sticky writer role
```

con:

```text
same physical transaction connection
```

---

# 69. Stage 8 — Runtime Evidence

El routing puede consumir snapshots de:

```text
health
replica state
lag
load
availability
topology
circuit state
```

---

# 70. Router should not probe

El router no deberá hacer:

```text
network health probe
```

en el hot path como efecto oculto.

---

# 71. Evidence snapshot

Preferir:

```php
final readonly class RoutingEvidenceSnapshot
{
    public function __construct(
        public TopologyGeneration $generation,
        public EndpointEvidenceMap $endpoints,
        public Instant $capturedAt,
    ) {}
}
```

---

# 72. Evidence freshness

Toda evidencia dinámica deberá poder declarar:

```text
capturedAt
validUntil
confidence
```

cuando aplique.

---

# 73. Stale evidence

Una policy podrá:

```text
reject
degrade
refresh elsewhere
```

pero el router no asumirá automáticamente que stale = current.

---

# 74. Health states

Modelo general:

```php
enum RoutingEndpointHealth
{
    case HEALTHY;
    case DEGRADED;
    case UNHEALTHY;
    case UNKNOWN;
}
```

---

# 75. Write health

Para escritura, `UNKNOWN` podrá ser tratado más estrictamente.

---

# 76. Read health

Una lectura eventual puede tolerar políticas distintas, pero nunca utilizar un endpoint explícitamente:

```text
UNHEALTHY
```

salvo mecanismos excepcionales controlados.

---

# 77. Circuit breaker integration

Posteriormente:

```text
OPEN circuit
→ endpoint ineligible
```

sin que el router implemente el circuit breaker.

---

# 78. Stage 9 — Candidate Ranking

Después de hard constraints:

```text
EligibleCandidateSet
```

puede contener varios endpoints.

---

# 79. Ranking

Se podrán considerar:

```text
preferred role
affinity
region
health
lag
load
weight
latency class
```

---

# 80. Ranking ≠ load balancing

El router puede preparar candidatos/ranking.

El algoritmo de distribución profunda se definirá en:

```text
182_DATABASE_LOAD_BALANCING_SYSTEM.md
```

---

# 81. Candidate score

Una implementación extensible podría utilizar:

```php
final readonly class RoutingCandidateScore
{
    public function __construct(
        public EndpointId $endpoint,
        public int $priority,
        public array $factors,
    ) {}
}
```

---

# 82. Score caution

No deberá crearse una fórmula universal que mezcle sin semántica:

```text
lag + latency + health + role
```

como números arbitrarios.

---

# 83. Lexicographic ranking

VoltStack deberá favorecer ranking jerárquico:

```text
1. correctness
2. hard consistency
3. role compatibility
4. affinity
5. health
6. topology preference
7. load distribution
```

---

# 84. Correctness before performance

Regla:

```text
Correctness
>
Availability preference
>
Performance optimization
```

cuando las garantías solicitadas lo requieran.

---

# 85. Stage 10 — Selection

El selector recibe únicamente candidatos elegibles.

```php
interface RoutingCandidateSelector
{
    public function select(
        RoutingCandidateSet $candidates,
        RoutingContext $context,
    ): ConnectionCandidate;
}
```

---

# 86. Empty set

Si:

```text
EligibleCandidateSet = ∅
```

el sistema deberá fallar.

No inventar fallback.

---

# 87. Selection reasons

El resultado deberá indicar por qué se eligió.

Ejemplos:

```text
TRANSACTION_PIN
AUTHORITATIVE_WRITE
DEDICATED_READER
READ_FALLBACK_TO_WRITER
STICKY_WRITE
AFFINITY_ENDPOINT
LOAD_BALANCED_READER
FAILOVER_TARGET
```

---

# 88. RoutingDecisionReason

```php
enum RoutingDecisionReason
{
    case TRANSACTION_PIN;
    case WRITE_REQUIRED;
    case AUTHORITATIVE_READ;
    case DEDICATED_READ;
    case READ_ON_WRITER_FALLBACK;
    case STICKY_WRITE;
    case ENDPOINT_AFFINITY;
    case LOAD_BALANCED;
    case FAILOVER;
}
```

---

# 89. Stage 11 — Decision Validation

Antes de publicar:

```text
Decision
→ invariant validation
```

---

# 90. Validation

Debe comprobar:

```text
endpoint belongs to candidate set
domain matches
role satisfies requirement
transaction affinity preserved
consistency requirement satisfied
endpoint not explicitly unavailable
```

---

# 91. No selector bypass

Un custom selector no podrá devolver:

```text
arbitrary EndpointId
```

fuera del eligible set.

---

# 92. Stage 12 — Decision Publication

Después de validar:

```text
RoutingDecision
```

podrá pasar a:

```text
ConnectionManager
```

---

# 93. Connection acquisition

```text
RoutingDecision
      │
      ▼
ConnectionManager
      │
      ▼
Endpoint Pool
      │
      ▼
ConnectionLease
```

---

# 94. Acquisition failure

El endpoint seleccionado puede fallar al adquirir conexión.

Eso no significa automáticamente:

```text
reroute forever
```

---

# 95. Routing retry

Debe existir una política separada que determine si:

```text
acquisition failure
→ refresh candidate evidence
→ reroute
```

---

# 96. Routing retry ≠ query retry

```text
Routing Retry
≠
Statement Retry
≠
Transaction Retry
```

---

# 97. Before execution reroute

Si no se ha ejecutado ningún statement y falla la adquisición:

```text
rerouting may be safe
```

según policy.

---

# 98. After execution reroute

Después de un outcome incierto:

```text
do not simply reroute and execute again
```

especialmente para writes.

---

# 99. Write uncertainty

```text
write dispatched
+
connection lost
```

puede producir:

```text
UNKNOWN
```

No:

```text
route to another writer and repeat
```

automáticamente.

---

# 100. Transaction failure

Dentro de transaction:

```text
no rerouting to new endpoint
```

para continuar la misma physical transaction.

---

# 101. RoutingPolicySet

```php
final readonly class RoutingPolicySet
{
    public function __construct(
        public ReadRoutingPolicy $read,
        public WriteRoutingPolicy $write,
        public ConsistencyRoutingPolicy $consistency,
        public AffinityRoutingPolicy $affinity,
        public FallbackRoutingPolicy $fallback,
    ) {}
}
```

---

# 102. ReadRoutingPolicy

Podrá definir:

```text
prefer dedicated readers
allow writer fallback
require healthy
regional preference
```

---

# 103. WriteRoutingPolicy

Podrá definir:

```text
require authoritative writer
require verified write capability
allow discovered failover target
```

---

# 104. ConsistencyRoutingPolicy

Define:

```text
EVENTUAL
SESSION
AUTHORITATIVE
```

y cómo deben satisfacerse.

---

# 105. FallbackRoutingPolicy

Fallbacks deberán estar enumerados.

No existir:

```text
catch anything → use default
```

---

# 106. FallbackReason

```php
enum RoutingFallbackReason
{
    case NO_READER_AVAILABLE;
    case READER_UNHEALTHY;
    case CONSISTENCY_UNSATISFIED;
    case AFFINITY_UNAVAILABLE;
}
```

---

# 107. Fallback chain

Ejemplo:

```text
READ
↓
dedicated healthy reader
↓ unavailable
READ_WRITE reader
↓ unavailable
writer fallback
↓ unavailable
FAIL
```

---

# 108. No recursive fallback

El sistema deberá evitar:

```text
policy A → B → A → B
```

---

# 109. Bounded routing

Toda decisión deberá tener:

```text
bounded number of stages
bounded candidate set
bounded policy evaluation
```

---

# 110. RoutingBudget

Modelo opcional:

```php
final readonly class RoutingBudget
{
    public function __construct(
        public int $maxCandidates,
        public int $maxPolicyPasses,
        public int $maxReroutes,
    ) {}
}
```

---

# 111. Routing loop protection

Si se excede:

```text
RoutingBudgetExceededException
```

---

# 112. Routing consistency within one operation

Una operación no deberá cambiar de endpoint durante ejecución normal.

```text
RoutingDecision
→ Lease
→ Execute
```

---

# 113. Multi-statement operation

Si una operación lógica genera varios statements que requieren coherencia:

```text
operation-level affinity
```

podrá mantener el endpoint.

---

# 114. Persistence plan

Un `flush()` puede producir:

```text
INSERT
INSERT
UPDATE
DELETE
```

Si se ejecuta dentro de transaction:

```text
same pinned writer
```

---

# 115. ORM-owned transaction

Si Persistence Engine crea una transaction interna:

```text
routing occurs before BEGIN
→ writer selected
→ transaction pinned
```

---

# 116. Non-transactional multiple writes

Si alguna operación avanzada permite múltiples writes sin transaction, cada statement podría técnicamente reroute.

Pero esto no deberá presentarse como atomicidad.

---

# 117. Schema routing

Schema operations:

```text
WRITE
+
authoritative
```

por default.

---

# 118. Migration routing

Migraciones deberán utilizar:

```text
authoritative writer
```

y evitar replicas.

---

# 119. Introspection routing

Schema introspection presenta una decisión especial.

Puede ser:

```text
READ
```

pero algunas tareas administrativas pueden requerir:

```text
AUTHORITATIVE_READ
```

---

# 120. Administrative queries

Deberán declarar intención explícita.

No depender del parser SQL.

---

# 121. Query cache interaction

Si Query Cache satisface una operación:

```text
no routing decision may be required
```

para DB.

---

# 122. Result cache

Igualmente:

```text
cache hit
→ no connection acquisition
```

---

# 123. Cache miss

Entonces:

```text
normal routing
```

---

# 124. Cache consistency

Cache no deberá hacer parecer que una replica satisface una garantía que realmente no satisface.

Son capas separadas.

---

# 125. IdentityMap interaction

```text
IdentityMap hit
→ no DB routing
```

---

# 126. Lazy loading

Lazy relationship load:

```text
Relationship Loader
→ Query Model
→ ConnectionRequirement
→ Router
```

---

# 127. N+1

El N+1 detector podrá observar routing, pero no deberá alterar decisiones automáticamente.

---

# 128. Eager loading

Queries eager adicionales podrán rutear como READ salvo restricciones de transaction/consistency.

---

# 129. Read/write split transparency

Código:

```php
$users = User::query()
    ->where('active', true)
    ->get();
```

no deberá conocer:

```text
reader-3
```

---

# 130. Explicit routing override

API avanzada:

```php
DB::connection('default')
    ->routing()
    ->requireWrite()
    ->table('users')
    ->find(10);
```

---

# 131. Override semantics

Esto crea:

```text
ConnectionRequirement
```

No manipula directamente el pool.

---

# 132. Explicit endpoint override

Puede existir para administración/testing:

```php
->routeToEndpoint('reader-2')
```

pero deberá ser:

- API avanzada;
- validada;
- restringida;
- no derivada de HTTP input;
- incompatible con transaction pinning si cambia endpoint.

---

# 133. Forced endpoint

Un endpoint forzado aún debe satisfacer hard constraints.

---

# 134. No unsafe override

No:

```php
->routeToEndpoint('read-only')
```

para una escritura.

---

# 135. Routing scopes

Puede existir:

```php
DB::withRouting(
    RoutingOverride::authoritativeReads(),
    fn () => ...
);
```

---

# 136. Scoped override

Debe utilizar:

```text
push
try
finally
pop
```

---

# 137. No static override

Prohibido:

```php
DB::$useWriter = true;
```

---

# 138. Nested routing scopes

Deben comportarse LIFO.

---

# 139. Routing scope corruption

Pop fuera de orden:

```text
RoutingScopeCorruptionException
```

---

# 140. Routing context storage

Mutable scoped routing state podrá vivir en:

```php
interface RoutingContextStorage
{
    public function current(): ?RoutingScopeState;
}
```

---

# 141. Storage implementation

Adaptable a:

```text
PHP request
FrankenPHP worker
RoadRunner worker
Fiber
OpenSwoole coroutine
CLI command
queue job
test
```

---

# 142. Persistent runtime safety

No deberá existir:

```php
static $lastWriteOccurred = true;
```

---

# 143. Request A vs Request B

```text
Request A:
WRITE
→ sticky writer

Request B:
READ
→ must not inherit A sticky state
```

---

# 144. Worker cleanup

Al terminar scope:

```text
routing overrides cleared
affinity cleared
sticky state cleared
temporary evidence references cleared
leases released by owner
```

---

# 145. Topology definitions

Compiled topology definitions podrán ser shared immutable state.

---

# 146. Dynamic topology

Runtime topology snapshot podrá cambiar entre operaciones.

---

# 147. Generation

Cada snapshot podrá tener:

```text
TopologyGeneration
```

---

# 148. Decision generation

`RoutingDecision` podrá registrar:

```text
topologyGeneration = 42
```

para diagnóstico.

---

# 149. Topology changed after decision

Si cambia después de adquirir una connection:

```text
do not migrate an active operation automatically
```

---

# 150. New operations

Nuevas operaciones utilizarán la generación actual.

---

# 151. Replica integration

El próximo sistema añadirá:

```text
ReplicaIdentity
ReplicaState
ReplicaCapability
ReplicationTopology
```

al candidate model.

---

# 152. Replica lag integration

Después:

```text
ReplicaLagEvidence
```

permitirá filtrar/rankear readers.

---

# 153. Sticky integration

Documento 180 podrá añadir:

```text
WriteObservation
→ StickyRoutingState
→ WRITE role affinity
```

---

# 154. Failover integration

Documento 181:

```text
selected writer unavailable
→ Failover Policy
→ topology transition
→ new eligible writer
```

---

# 155. Load balancing integration

Documento 182:

```text
eligible equivalent readers
→ LoadBalancer
→ selected reader
```

---

# 156. Distributed architecture integration

Documento 183 establecerá límites para:

```text
clusters
regions
distributed topology
```

---

# 157. Sharding integration

Documento 184:

```text
ShardKey
→ Shard
→ ReadWriteConnectionGroup
→ Routing
```

No:

```text
ReadWriteRouter
→ guess shard
```

---

# 158. Partition routing integration

Documento 185 añadirá selección de partición antes o coordinada con routing.

---

# 159. Router extensibility

El sistema deberá permitir extensiones controladas.

---

# 160. RoutingPolicyContributor

```php
interface RoutingPolicyContributor
{
    public function contribute(
        RoutingPolicyRegistry $registry,
    ): void;
}
```

---

# 161. Bootstrap only

Contribuciones estructurales:

```text
bootstrap
→ compile
→ validate
→ freeze
```

---

# 162. No runtime policy registration

No:

```php
$request->registerRoutingPolicy(...)
```

---

# 163. RoutingPolicyRegistry

Contendrá:

```text
named policy definitions
constraint factories
ranking policies
selectors
```

como immutable compiled registry.

---

# 164. Custom policies

Ejemplos:

```text
geo.prefer_local
compliance.eu_only
analytics.read_only
premium.authoritative
```

---

# 165. Custom policy limitations

Una custom policy no podrá:

- ejecutar SQL arbitrario;
- mutar TransactionContext;
- cambiar tenant;
- saltarse role constraints;
- leer credentials;
- devolver endpoints externos al candidate set.

---

# 166. Policy priority

Orden explícito:

```php
final readonly class RoutingPolicyPriority
{
    public function __construct(
        public int $value,
    ) {}
}
```

---

# 167. Determinism

Mismos:

```text
context
candidate snapshot
policy generation
runtime evidence
```

deberán producir un conjunto elegible equivalente.

---

# 168. Load balancing nondeterminism

La selección final puede ser intencionalmente variable por:

```text
round-robin
randomized weighted selection
```

pero esa variabilidad deberá pertenecer al selector/load balancer, no a constraints.

---

# 169. Routing fingerprint

Podrá construirse:

```text
RoutingFingerprint
=
hash(
    domain
    + requirement
    + consistency
    + affinity class
    + policy generation
    + topology generation
)
```

---

# 170. Routing cache

Se debe ser extremadamente cuidadoso con cachear decisiones.

---

# 171. Why

Porque:

```text
health
lag
load
topology
transaction
sticky state
```

pueden cambiar.

---

# 172. Safe cache target

Puede cachearse:

```text
compiled routing policy
constraint pipeline
static eligibility structure
```

---

# 173. Unsafe cache target

No globalmente:

```text
READ → reader-2
```

---

# 174. Decision cache

Si alguna implementación lo usa deberá estar:

```text
short-lived
generation-aware
scope-aware
evidence-aware
```

---

# 175. Security

Routing es una frontera sensible porque determina qué infraestructura recibe una operación.

---

# 176. Input isolation

Datos del usuario no deberán controlar directamente:

```text
EndpointId
ConnectionRole
Shard
Writer
Replica
```

sin una capa autorizada de resolución.

---

# 177. Credentials

Router nunca necesita:

```text
database password
```

---

# 178. Endpoint metadata

Diagnostics deberán sanitizar:

```text
internal hostnames
provider identifiers
```

según configuración.

---

# 179. Cross-tenant attack prevention

Toda candidate selection deberá validar:

```text
execution domain
```

antes de preferencias de performance.

---

# 180. Forced routing security

Endpoint overrides podrán requerir:

```text
trusted internal capability
```

en APIs administrativas.

---

# 181. Telemetry architecture

Eventos conceptuales:

```text
RoutingStarted
CandidateRejected
FallbackApplied
RoutingDecisionMade
RoutingFailed
RoutingRerouted
```

---

# 182. Events observational

Listeners no podrán cambiar el resultado en eventos observacionales.

---

# 183. Policy hooks vs events

Si se necesita modificar routing:

```text
RoutingPolicy
```

No:

```text
event listener mutates decision
```

---

# 184. Metrics

Ejemplos:

```text
db.routing.decisions
db.routing.failures
db.routing.fallbacks
db.routing.reader
db.routing.writer
db.routing.authoritative
db.routing.transaction_pinned
db.routing.candidates_rejected
db.routing.duration
```

---

# 185. Metric dimensions

Bounded:

```text
intent
role
reason
fallback
transactional
outcome
driver family
```

---

# 186. Avoid high cardinality

No usar por default:

```text
query text
tenant id
entity id
user id
full host
```

---

# 187. Trace

Ejemplo:

```text
db.routing.intent = read
db.routing.required_consistency = session
db.routing.result_role = write
db.routing.reason = sticky_write
db.routing.transaction_pinned = false
```

---

# 188. Routing diagnostics

API conceptual:

```php
DB::routing()->explain(
    connection: 'default',
    intent: ConnectionIntent::READ,
);
```

---

# 189. Explain result

```text
Connection:
    default

Intent:
    READ

Consistency:
    EVENTUAL

Transaction:
    none

Candidates:
    3

Rejected:
    writer-1
        reason: dedicated readers preferred

Eligible:
    reader-1
    reader-2

Selected:
    reader-2

Reason:
    LOAD_BALANCED

Fallback:
    no
```

---

# 190. Authoritative explain

```text
Intent:
    AUTHORITATIVE_READ

Readers:
    rejected
    reason: authority not established

Writer:
    accepted

Selected:
    writer-1

Reason:
    AUTHORITATIVE_READ
```

---

# 191. Transaction explain

```text
Intent:
    READ

Transaction:
    ACTIVE

Pinned:
    writer-1

Normal routing:
    bypassed

Selected:
    writer-1

Reason:
    TRANSACTION_PIN
```

---

# 192. Candidate rejection report

Cada rechazo podrá registrar:

```text
EndpointId
ConstraintId
ReasonCode
EvidenceStatus
```

---

# 193. Bounded diagnostics

En producción:

```text
max rejection details
```

deberá estar limitado.

---

# 194. Error hierarchy

Propuesta:

```text
DatabaseRoutingException
├── RoutingContextException
├── RoutingDomainMismatchException
├── RoutingIntentException
├── RoutingCandidateUnavailableException
├── RoutingNoEligibleCandidateException
├── RoutingConstraintViolationException
├── RoutingConsistencyUnsatisfiedException
├── RoutingAffinityUnsatisfiedException
├── RoutingTransactionPinViolationException
├── RoutingWriteAuthorityUnavailableException
├── RoutingFallbackRejectedException
├── RoutingSelectionException
├── RoutingDecisionInvalidException
├── RoutingBudgetExceededException
├── RoutingScopeCorruptionException
├── RoutingEvidenceUnavailableException
├── RoutingEvidenceStaleException
└── RoutingInvariantViolationException
```

---

# 195. No eligible candidate

Mensaje:

```text
Unable to route database operation.

Connection: default
Intent: WRITE
Required role: WRITE
Candidates inspected: 3
Eligible candidates: 0
Reason: no candidate has verified write capability.
```

---

# 196. Consistency error

```text
Requested read consistency SESSION cannot currently
be satisfied by any eligible reader.
Writer fallback is disabled.
```

---

# 197. Testing architecture

Suite:

```text
RoutingContextTests
ReadWriteRoutingEngineTests
RoutingConstraintTests
RoutingPolicyTests
RoutingIntentTests
RoutingDomainTests
RoutingTransactionTests
RoutingConsistencyTests
RoutingAffinityTests
RoutingFallbackTests
RoutingCandidateSelectionTests
RoutingEvidenceTests
RoutingExtensionTests
RoutingPersistentRuntimeTests
RoutingCoroutineTests
RoutingTelemetryTests
RoutingDiagnosticsTests
RoutingSecurityTests
```

---

# 198. READ test

```text
READ
+
2 healthy readers
+
1 writer
```

deberá producir:

```text
eligible readers
```

antes de writer fallback.

---

# 199. WRITE test

```text
WRITE
```

deberá eliminar todos los read-only candidates.

---

# 200. Unknown write authority

```text
WRITE
+
candidate role UNKNOWN
```

no deberá seleccionarse como writer.

---

# 201. Transaction test

```text
ACTIVE transaction
+
pinned writer
+
READ operation
```

deberá:

```text
select pinned writer
```

sin consultar reader selector.

---

# 202. Transaction mismatch test

```text
read-only pinned connection
+
WRITE
```

deberá fallar.

---

# 203. Domain isolation test

Candidatos de:

```text
tenant A
tenant B
```

Una query de A jamás seleccionará B.

---

# 204. Authoritative test

```text
AUTHORITATIVE_READ
```

no utilizará replica cuya autoridad/freshness no pueda establecerse.

---

# 205. Fallback test

```text
READ
+
no reader
+
writer available
+
fallback enabled
```

→ writer.

---

# 206. Fallback disabled test

Mismo escenario:

```text
fallback disabled
```

→ explicit failure.

---

# 207. Affinity prefer test

Endpoint preferido unavailable:

```text
PREFER
```

→ otro candidato elegible.

---

# 208. Affinity require test

Endpoint requerido unavailable:

```text
REQUIRE
```

→ failure.

---

# 209. Sticky scope test

Request A realiza write.

Posterior read A:

```text
writer affinity
```

si sticky policy está activa.

Request B:

```text
no inherited affinity
```

---

# 210. Evidence test

Health snapshot:

```text
reader-1 = UNHEALTHY
reader-2 = HEALTHY
```

reader-1 deberá eliminarse.

---

# 211. Stale evidence test

Si una policy exige evidencia reciente:

```text
stale snapshot
```

no podrá considerarse fresh.

---

# 212. Selector safety test

Custom selector intenta devolver:

```text
endpoint outside eligible set
```

→ `RoutingDecisionInvalidException`.

---

# 213. Persistent worker test

Miles de scopes secuenciales deberán demostrar:

```text
no sticky leakage
no affinity leakage
no transaction pin leakage
bounded memory
```

---

# 214. Coroutine test

Múltiples routing contexts simultáneos deberán permanecer aislados.

---

# 215. Proposed directory structure

```text
src/Quantum/Database/Routing/
│
├── ReadWriteRoutingEngine.php
├── DefaultReadWriteRoutingEngine.php
├── RoutingOperationId.php
├── RoutingContext.php
├── RoutingDecision.php
├── RoutingDecisionReason.php
├── RoutingDecisionMetadata.php
│
├── Context/
│   ├── RoutingContextFactory.php
│   ├── RoutingContextResolver.php
│   ├── RoutingContextStorage.php
│   ├── RoutingScope.php
│   ├── RoutingScopeState.php
│   └── RoutingScopeToken.php
│
├── Constraint/
│   ├── RoutingConstraint.php
│   ├── RoutingConstraintResult.php
│   ├── RoutingConstraintStatus.php
│   ├── ExecutionDomainConstraint.php
│   ├── ConnectionRoleConstraint.php
│   ├── TransactionConstraint.php
│   ├── ReadConsistencyConstraint.php
│   ├── ConnectionAffinityConstraint.php
│   └── EndpointHealthConstraint.php
│
├── Candidate/
│   ├── RoutingCandidate.php
│   ├── RoutingCandidateSet.php
│   ├── RoutingCandidateScore.php
│   ├── RoutingCandidateSelector.php
│   └── DefaultRoutingCandidateSelector.php
│
├── Policy/
│   ├── RoutingPolicy.php
│   ├── RoutingPolicySet.php
│   ├── RoutingPolicyRegistry.php
│   ├── RoutingPolicyContributor.php
│   ├── ReadRoutingPolicy.php
│   ├── WriteRoutingPolicy.php
│   ├── ConsistencyRoutingPolicy.php
│   ├── AffinityRoutingPolicy.php
│   ├── FallbackRoutingPolicy.php
│   └── RoutingFallbackReason.php
│
├── Affinity/
│   ├── ConnectionAffinity.php
│   ├── ConnectionAffinityMode.php
│   └── ConnectionAffinityStrength.php
│
├── Evidence/
│   ├── RoutingEvidenceSnapshot.php
│   ├── EndpointRoutingEvidence.php
│   ├── EndpointEvidenceMap.php
│   ├── RoutingEndpointHealth.php
│   └── TopologyGeneration.php
│
├── Validation/
│   ├── RoutingDecisionValidator.php
│   └── RoutingInvariantValidator.php
│
├── Diagnostics/
│   ├── RoutingInspector.php
│   ├── RoutingExplainer.php
│   ├── RoutingExplanation.php
│   └── CandidateRejectionReport.php
│
├── Telemetry/
│   ├── RoutingTelemetry.php
│   ├── RoutingStarted.php
│   ├── RoutingDecisionMade.php
│   ├── RoutingFailed.php
│   └── RoutingFallbackApplied.php
│
└── Exception/
    ├── DatabaseRoutingException.php
    ├── RoutingContextException.php
    ├── RoutingDomainMismatchException.php
    ├── RoutingNoEligibleCandidateException.php
    ├── RoutingConstraintViolationException.php
    ├── RoutingConsistencyUnsatisfiedException.php
    ├── RoutingAffinityUnsatisfiedException.php
    ├── RoutingTransactionPinViolationException.php
    ├── RoutingWriteAuthorityUnavailableException.php
    ├── RoutingFallbackRejectedException.php
    ├── RoutingSelectionException.php
    ├── RoutingDecisionInvalidException.php
    ├── RoutingBudgetExceededException.php
    ├── RoutingScopeCorruptionException.php
    └── RoutingInvariantViolationException.php
```

---

# 216. Architectural invariants

## DB-RWR-001
Routing será distinto de Connection Management.

## DB-RWR-002
Routing será distinto de endpoint discovery.

## DB-RWR-003
Routing será distinto de load balancing.

## DB-RWR-004
Routing será distinto de failover.

## DB-RWR-005
Routing será distinto de replica management.

## DB-RWR-006
RoutingDecision será distinto de ConnectionLease.

## DB-RWR-007
RoutingDecision será distinto de QueryExecution.

## DB-RWR-008
RoutingContext será immutable.

## DB-RWR-009
RoutingContext no ejecutará I/O.

## DB-RWR-010
RoutingContext no abrirá conexiones.

## DB-RWR-011
RoutingContext no mutará transactions.

## DB-RWR-012
RoutingContext no contendrá credentials.

## DB-RWR-013
Cada decisión tendrá RoutingOperationId.

## DB-RWR-014
ConnectionRequirement deberá existir antes del routing.

## DB-RWR-015
ExecutionDomain deberá resolverse antes del routing.

## DB-RWR-016
SQL verb no será autoridad de routing.

## DB-RWR-017
Semantic Query Model será fuente preferida de intent.

## DB-RWR-018
ORM no seleccionará endpoints.

## DB-RWR-019
SQL Compiler no seleccionará endpoints.

## DB-RWR-020
Driver no decidirá application routing policy.

## DB-RWR-021
Hard constraints nunca podrán sacrificarse por performance.

## DB-RWR-022
Soft preferences podrán sacrificarse.

## DB-RWR-023
Correct execution domain será hard constraint.

## DB-RWR-024
WRITE capability será hard constraint para writes.

## DB-RWR-025
Transaction pinning será hard constraint.

## DB-RWR-026
Required consistency será hard constraint cuando la policy así lo defina.

## DB-RWR-027
Active transaction tendrá precedencia sobre ordinary routing.

## DB-RWR-028
Active transaction utilizará pinned connection.

## DB-RWR-029
Reads dentro de write transaction no saltarán a replicas.

## DB-RWR-030
Write dentro de read-only transaction será rechazado.

## DB-RWR-031
Savepoint no creará nuevo routing domain.

## DB-RWR-032
Joined nested transaction utilizará misma pinned connection.

## DB-RWR-033
Nueva physical transaction podrá crear nuevo routing context.

## DB-RWR-034
Router no transferirá transaction state entre endpoints.

## DB-RWR-035
Domain mismatch eliminará candidato.

## DB-RWR-036
Cross-tenant routing estará prohibido.

## DB-RWR-037
Cross-shard routing implícito estará prohibido.

## DB-RWR-038
Cross-database fallback estará prohibido.

## DB-RWR-039
READ podrá preferir dedicated readers.

## DB-RWR-040
READ podrá caer al writer solo mediante policy.

## DB-RWR-041
WRITE nunca caerá a read-only candidate.

## DB-RWR-042
AUTHORITATIVE_READ no equivaldrá automáticamente a READ.

## DB-RWR-043
READ_FOR_UPDATE requerirá write-capable routing.

## DB-RWR-044
Schema mutations requerirán writer.

## DB-RWR-045
Migrations requerirán authoritative writer.

## DB-RWR-046
Unknown query intent no se asumirá READ.

## DB-RWR-047
Unknown intent será WRITE o REJECT.

## DB-RWR-048
Read consistency será distinta de transaction isolation.

## DB-RWR-049
EVENTUAL no garantizará read-your-writes.

## DB-RWR-050
SESSION podrá requerir writer/sticky routing.

## DB-RWR-051
AUTHORITATIVE requerirá evidencia suficiente.

## DB-RWR-052
UNKNOWN consistency evidence no será SATISFIED.

## DB-RWR-053
Replica freshness no se inferirá por latency.

## DB-RWR-054
Affinity será distinta de transaction.

## DB-RWR-055
ROLE affinity no requerirá misma physical connection.

## DB-RWR-056
ENDPOINT affinity podrá ser PREFER o REQUIRE.

## DB-RWR-057
TRANSACTION affinity requerirá exact pinned connection.

## DB-RWR-058
PREFER affinity podrá degradarse.

## DB-RWR-059
REQUIRE affinity no podrá degradarse silenciosamente.

## DB-RWR-060
Sticky state será scoped.

## DB-RWR-061
Sticky state no será global.

## DB-RWR-062
Router consumirá runtime evidence mediante snapshots.

## DB-RWR-063
Router no realizará health probes ocultos.

## DB-RWR-064
Evidence deberá poder declarar freshness.

## DB-RWR-065
Stale evidence no será current evidence.

## DB-RWR-066
UNHEALTHY candidate no será normalmente elegible.

## DB-RWR-067
UNKNOWN write authority fallará cerrado.

## DB-RWR-068
Candidate filtering precederá ranking.

## DB-RWR-069
Ranking nunca reintroducirá candidato rechazado.

## DB-RWR-070
Selector recibirá solo candidatos elegibles.

## DB-RWR-071
Selector no podrá devolver endpoint externo al candidate set.

## DB-RWR-072
RoutingDecision será validada antes de publicación.

## DB-RWR-073
Selected endpoint deberá satisfacer required role.

## DB-RWR-074
Selected endpoint deberá pertenecer al execution domain.

## DB-RWR-075
Selected endpoint deberá satisfacer transaction constraints.

## DB-RWR-076
Selected endpoint deberá satisfacer required consistency.

## DB-RWR-077
Empty eligible set producirá error explícito.

## DB-RWR-078
Router no inventará fallback.

## DB-RWR-079
Fallback chain será bounded.

## DB-RWR-080
Fallback loops serán detectados.

## DB-RWR-081
Read fallback será observable.

## DB-RWR-082
Routing reason será explícita.

## DB-RWR-083
Routing retry será distinto de query retry.

## DB-RWR-084
Routing retry será distinto de transaction retry.

## DB-RWR-085
Acquisition failure antes de execution podrá permitir reroute.

## DB-RWR-086
Unknown write outcome no permitirá automatic reroute+replay.

## DB-RWR-087
Active transaction connection failure no permitirá continuation en otro endpoint.

## DB-RWR-088
Correctness tendrá prioridad sobre load distribution.

## DB-RWR-089
Consistency tendrá prioridad sobre reader preference.

## DB-RWR-090
Role compatibility tendrá prioridad sobre latency.

## DB-RWR-091
Health eligibility tendrá prioridad sobre load balancing.

## DB-RWR-092
Load balancing operará sobre candidatos ya elegibles.

## DB-RWR-093
Topology generation será diagnosticable.

## DB-RWR-094
Topology change no migrará una operación activa.

## DB-RWR-095
Nuevas operaciones podrán usar nueva topology generation.

## DB-RWR-096
Compiled routing policies podrán compartirse.

## DB-RWR-097
Mutable routing scopes no podrán compartirse entre requests.

## DB-RWR-098
Mutable routing scopes no podrán compartirse entre jobs.

## DB-RWR-099
Mutable routing scopes no podrán compartirse entre coroutines.

## DB-RWR-100
Routing override será scoped.

## DB-RWR-101
Routing overrides utilizarán stack LIFO.

## DB-RWR-102
Routing scope corruption fallará explícitamente.

## DB-RWR-103
Forced endpoint deberá pasar hard constraints.

## DB-RWR-104
Forced endpoint no podrá convertir reader en writer.

## DB-RWR-105
HTTP input no controlará EndpointId directamente.

## DB-RWR-106
HTTP input no controlará ConnectionRole directamente.

## DB-RWR-107
Custom policies serán trusted extensions.

## DB-RWR-108
Custom policy no podrá cambiar tenant.

## DB-RWR-109
Custom policy no podrá cambiar database identity.

## DB-RWR-110
Custom policy no podrá saltarse transaction pinning.

## DB-RWR-111
Custom selector no podrá saltarse candidate eligibility.

## DB-RWR-112
Policy registration estructural ocurrirá durante bootstrap.

## DB-RWR-113
Runtime policy registry estará frozen.

## DB-RWR-114
Policy conflicts deberán ser deterministas.

## DB-RWR-115
No existirá last-write-wins silencioso en policy registration.

## DB-RWR-116
Constraint evaluation deberá ser side-effect free.

## DB-RWR-117
Ranking policies deberán ser bounded.

## DB-RWR-118
Routing hot path no deberá realizar reflection innecesaria.

## DB-RWR-119
Routing hot path no deberá hacer service discovery remoto oculto.

## DB-RWR-120
Routing candidate count deberá ser bounded.

## DB-RWR-121
Routing policy passes deberán ser bounded.

## DB-RWR-122
Rerouting attempts deberán ser bounded.

## DB-RWR-123
Cache podrá almacenar compiled routing structures.

## DB-RWR-124
Global endpoint decision cache será inseguro por default.

## DB-RWR-125
Decision cache, si existe, será generation-aware.

## DB-RWR-126
Decision cache, si existe, será evidence-aware.

## DB-RWR-127
Transaction-pinned decisions no se reutilizarán fuera de transaction.

## DB-RWR-128
Sticky decisions no se reutilizarán fuera de scope.

## DB-RWR-129
IdentityMap hit podrá evitar routing completamente.

## DB-RWR-130
Query Cache hit podrá evitar routing completamente.

## DB-RWR-131
Result Cache hit podrá evitar routing completamente.

## DB-RWR-132
Cache miss reanudará routing normal.

## DB-RWR-133
Cache consistency será distinta de DB routing consistency.

## DB-RWR-134
Lazy loading utilizará normal query routing.

## DB-RWR-135
Eager loading utilizará normal query routing.

## DB-RWR-136
N+1 detector será observador, no router.

## DB-RWR-137
Telemetry listeners no mutarán RoutingDecision.

## DB-RWR-138
RoutingPolicy será mecanismo de modificación de routing.

## DB-RWR-139
Telemetry no expondrá credentials.

## DB-RWR-140
Telemetry no expondrá query values por default.

## DB-RWR-141
Metrics evitarán labels de alta cardinalidad.

## DB-RWR-142
Routing diagnostics serán sanitizables.

## DB-RWR-143
Candidate rejection reasons serán explicables.

## DB-RWR-144
Diagnostic detail será bounded.

## DB-RWR-145
FrankenPHP worker reuse no filtrará routing state.

## DB-RWR-146
RoadRunner worker reuse no filtrará routing state.

## DB-RWR-147
OpenSwoole coroutine state estará aislado.

## DB-RWR-148
Static current reader estará prohibido.

## DB-RWR-149
Static current writer estará prohibido.

## DB-RWR-150
Static sticky flag estará prohibido.

## DB-RWR-151
RoutingContextStorage será runtime-adaptable.

## DB-RWR-152
Scope cleanup será obligatorio.

## DB-RWR-153
Scope cleanup no hará commit implícito.

## DB-RWR-154
Scope cleanup no ejecutará queries implícitas para reparar routing.

## DB-RWR-155
Replica System extenderá candidate evidence sin reemplazar router.

## DB-RWR-156
Lag Awareness extenderá consistency evaluation sin reemplazar router.

## DB-RWR-157
Sticky System extenderá affinity sin reemplazar router.

## DB-RWR-158
Failover System extenderá topology/evidence sin romper transaction affinity.

## DB-RWR-159
Load Balancer seleccionará entre candidatos equivalentes elegibles.

## DB-RWR-160
Sharding resolverá shard antes del read/write endpoint routing.

## DB-RWR-161
Partition routing no permitirá cross-partition leakage.

## DB-RWR-162
Single-node READ_WRITE será soportado sin routing artificial complejo.

## DB-RWR-163
Read/write splitting será transparente para APIs normales.

## DB-RWR-164
Application code no deberá conocer replicas para uso normal.

## DB-RWR-165
Application code no deberá conocer writer hostname para uso normal.

## DB-RWR-166
Read/write topology podrá evolucionar sin cambiar Query Builder API.

## DB-RWR-167
Read/write topology podrá evolucionar sin cambiar ORM API.

## DB-RWR-168
Routing no otorgará atomicidad.

## DB-RWR-169
Routing no otorgará transaction isolation.

## DB-RWR-170
Routing no otorgará durability.

## DB-RWR-171
Routing no resolverá distributed transaction semantics.

## DB-RWR-172
Routing no fingirá consistency que la infraestructura no puede demostrar.

## DB-RWR-173
Routing no convertirá una topología incorrecta en correcta.

## DB-RWR-174
RoutingDecision deberá reflejar la evidencia disponible en ese momento.

## DB-RWR-175
UNKNOWN deberá permanecer UNKNOWN cuando no pueda resolverse de forma segura.

---

# 217. Modelo matemático de elegibilidad

Sea:

```text
C = conjunto inicial de candidatos
```

Para una operación `O`:

```text
Eligible(O) =
{
    c ∈ C |
    Domain(c) = Domain(O)
    ∧ Role(c) satisfies RoleRequirement(O)
    ∧ TransactionConstraint(c, O)
    ∧ ConsistencyConstraint(c, O)
    ∧ HealthConstraint(c, O)
    ∧ AffinityRequirement(c, O)
}
```

Entonces:

```text
RoutingDecision(O)
=
Select(
    Rank(
        Eligible(O),
        Preferences(O)
    )
)
```

Si:

```text
Eligible(O) = ∅
```

entonces:

```text
RoutingDecision(O) = FAILURE
```

salvo que una política de fallback explícita produzca un **nuevo conjunto válido de restricciones**.

---

# 218. Precedencia de garantías

VoltStack utilizará conceptualmente:

```text
Execution Domain
        ↓
Transaction Affinity
        ↓
Write Authority
        ↓
Required Consistency
        ↓
Endpoint Eligibility
        ↓
Preferred Role
        ↓
Sticky/Affinity Preferences
        ↓
Health Preference
        ↓
Topology Preference
        ↓
Load Balancing
        ↓
Performance Preference
```

Una capa inferior nunca deberá invalidar una garantía superior.

---

# 219. Flujo READ normal

```text
SELECT
  │
  ▼
Semantic Query Classification
  │
  ▼
READ
  │
  ▼
No active transaction
  │
  ▼
EVENTUAL consistency
  │
  ▼
Readers available?
  │
  ├── YES
  │    │
  │    ▼
  │  Eligible readers
  │    │
  │    ▼
  │  Load balancing
  │    │
  │    ▼
  │  Reader selected
  │
  └── NO
       │
       ▼
 Writer fallback allowed?
       │
       ├── YES → Writer
       └── NO  → Failure
```

---

# 220. Flujo WRITE

```text
INSERT / UPDATE / DELETE
          │
          ▼
        WRITE
          │
          ▼
Active transaction?
    │             │
   YES            NO
    │             │
    ▼             ▼
Pinned writer   Writer candidates
    │             │
    ▼             ▼
Validate       Authority/health
    │             │
    ▼             ▼
Execute         Select writer
```

Nunca:

```text
WRITE
→ no writer
→ reader
```

---

# 221. Flujo read-after-write

```text
WRITE
  │
  ▼
Successful commit
  │
  ▼
Write observation
  │
  ▼
Sticky/session consistency state
  │
  ▼
READ
  │
  ▼
SESSION consistency
  │
  ▼
Can replica prove required freshness?
       │
       ├── YES → replica eligible
       │
       └── NO
            │
            ▼
          writer
```

Esto permite que VoltStack evolucione posteriormente desde sticky routing básico hacia replication-position-aware routing.

---

# 222. Flujo transaccional

```text
BEGIN
  │
  ▼
Writer selected
  │
  ▼
Lease acquired
  │
  ▼
TransactionContext
  │
  ▼
PinnedConnection
  │
  ├── READ
  ├── WRITE
  ├── READ
  └── WRITE
  │
  ▼
COMMIT
```

El router no reevalúa replicas para cada `READ` dentro de esa transaction.

---

# 223. Architectural outcome

Con los documentos 176 y 177 se establece la separación:

```text
176 — Read/Write Connection System
        │
        │  ¿Qué conexiones existen?
        │  ¿Qué roles/capacidades tienen?
        │
        ▼
Candidate Set

177 — Read/Write Routing System
        │
        │  ¿Cuál candidato puede utilizarse?
        │  ¿Cuál debe preferirse?
        │  ¿Qué restricciones aplican?
        │
        ▼
Routing Decision
```

A partir de aquí podrán añadirse:

```text
Replica Awareness
Lag Awareness
Sticky Connections
Failover
Load Balancing
Sharding
Partition Routing
```

sin introducir esas responsabilidades directamente en:

```text
Query Builder
ORM
Persistence Engine
Transaction Manager
SQL Compiler
Driver
```

---

# 224. Regla maestra final

> **VoltStack tratará el routing de base de datos como una decisión semántica y verificable de infraestructura: primero determinará dónde puede ejecutarse correctamente una operación y solo después optimizará dónde conviene ejecutarla.**

Por tanto:

```text
Correctness
    ↓
Eligibility
    ↓
Consistency
    ↓
Affinity
    ↓
Availability
    ↓
Optimization
    ↓
Selection
```

y nunca:

```text
SELECT
→ choose random replica
```

---

# 225. Siguiente documento

```text
178_DATABASE_REPLICA_SYSTEM.md
```

El siguiente documento añadirá el modelo formal de replicas:

```text
Replication Topology
├── Authoritative Node
├── Replica Nodes
├── Replica Identity
├── Replica Role
├── Replica State
├── Replica Capability
├── Replica Health
├── Replication Position
├── Replica Eligibility
└── Replica Lifecycle
```

y establecerá una distinción fundamental:

```text
Read Connection
≠
Replica
```

porque una conexión de lectura es una **capacidad de routing**, mientras que una replica representa una **relación topológica y de replicación respecto a una fuente autoritativa**.

Esto permitirá que el documento 179 incorpore posteriormente:

```text
Replica Lag Awareness
```

y que el router definido aquí pueda tomar decisiones basadas no solo en:

```text
READ vs WRITE
```

sino también en:

```text
replication state
freshness
consistency evidence
topology generation
```

sin acoplar el Query Engine a la infraestructura física.