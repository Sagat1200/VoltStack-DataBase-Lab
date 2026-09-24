# 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md

# VoltStack Quantum Database
## Query Input Security System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 228 — Query Input Security System  
**Bloque:** 22 — Security  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md`  
**Siguiente documento:** `229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack Database controlará **qué capacidades de consulta puede solicitar un consumidor externo**.

La arquitectura del documento 227 impide que los valores externos se conviertan accidentalmente en SQL.

Sin embargo, una consulta puede estar perfectamente protegida contra SQL Injection y aun ser insegura.

Ejemplo:

```php id="u0ansx"
$query
    ->where('salary', '>', 0)
    ->orderBy('salary')
    ->limit(1_000_000);
```

Todo puede estar:

```text id="x8b54r"
correctamente parametrizado
```

y aun así permitir:

```text id="ccv3lf"
sensitive field access
excessive result size
resource exhaustion
unauthorized filtering
data inference
expensive sorting
```

Por tanto:

> **Query Input Security controla las capacidades semánticas de una consulta externa; SQL Injection Prevention controla cómo dichas capacidades se transforman de forma segura en SQL.**

---

# 2. Regla maestra

La regla central será:

> **Una entrada externa nunca deberá poseer acceso implícito a todas las capacidades del Query Builder. Cada campo, operador, relación, proyección, ordenamiento, agregación y límite deberá existir dentro de una política explícita.**

En forma resumida:

```text id="bw9m2f"
External Query Input
        ↓
Parse
        ↓
Normalize
        ↓
Resolve
        ↓
Authorize
        ↓
Constrain
        ↓
Budget
        ↓
Typed Query Request
        ↓
Query AST
```

---

# 3. Query Input Security ≠ SQL Injection Prevention

Distinción esencial:

```text id="wnib8a"
SQL Injection Prevention
=
Can this input alter SQL structure unexpectedly?

Query Input Security
=
Should this caller be allowed to request this query structure?
```

Ejemplo:

```text id="w03a3f"
sort=salary
```

puede resolverse de forma completamente segura a:

```text id="vx42x5"
Employee.salary
```

sin SQL Injection.

Pero el usuario podría no estar autorizado a ordenar por salario.

---

# 4. Query Input Security ≠ Authorization

También:

```text id="3v1y60"
Query Input Security
≠
Business Authorization
```

Query Input Security define capacidades disponibles.

Authorization decide si el principal concreto puede utilizarlas en el contexto actual.

Ambos pueden colaborar.

---

# 5. Query Input Security ≠ Validation

Validar que:

```text id="8th6vx"
limit = 100
```

sea un entero no responde si:

```text id="yfz7v9"
100
```

es un límite permitido para ese endpoint.

Por tanto:

```text id="faz8nk"
Validation
≠
Policy
```

---

# 6. Threat model

El sistema deberá considerar entradas externas capaces de solicitar:

```text id="i76a6g"
forbidden fields
sensitive fields
forbidden relationships
deep relationship traversal
expensive filters
huge IN lists
unbounded pagination
expensive ordering
large projections
expensive aggregations
grouping explosions
full-text abuse
JSON traversal abuse
high shard fan-out
cross-tenant scopes
raw query capabilities
```

---

# 7. Ataques sin SQL Injection

Ejemplos:

```text id="751j03"
limit=10000000
```

```text id="09iw4h"
include=orders.items.product.vendor.orders.items...
```

```text id="8i5oqc"
filter[password_hash][not_null]=true
```

```text id="7ep2dl"
group_by=user_id,email,device_id,ip_address,...
```

```text id="9yuc7o"
search=*
```

Todo puede ser SQL sintácticamente seguro.

Aun así puede ser operacionalmente o lógicamente inseguro.

---

# 8. Arquitectura general

```text id="81sy22"
External Query Input
        │
        ▼
QueryInputDecoder
        │
        ▼
QueryInputModel
        │
        ▼
Normalizer
        │
        ▼
Semantic Resolver
        │
        ├── Fields
        ├── Operators
        ├── Relationships
        ├── Sorts
        ├── Includes
        ├── Aggregations
        └── Projection
        │
        ▼
Authorization-Aware Policy
        │
        ▼
Complexity Analyzer
        │
        ▼
Resource Budget
        │
        ▼
ValidatedQueryRequest
        │
        ▼
Query Builder / Query AST
```

---

# 9. External Query Input

Podrá provenir de:

```text id="akggkm"
HTTP query parameters
JSON request body
Graph-like query request
CLI arguments
API filters
admin UI
search forms
public APIs
machine-to-machine APIs
```

---

# 10. External query syntax independence

El Security System no deberá depender de una única sintaxis externa.

Ejemplos posibles:

```text id="kaz81e"
?filter[name]=John
```

o:

```json id="qhsjdp"
{
  "filters": [
    {
      "field": "name",
      "operator": "equals",
      "value": "John"
    }
  ]
}
```

Ambos deberán poder producir el mismo:

```text id="f96dw4"
QueryInputModel
```

---

# 11. QueryInputModel

Modelo intermedio:

```php id="diemw0"
final readonly class QueryInputModel
{
    public function __construct(
        public FilterInputCollection $filters,
        public SortInputCollection $sorts,
        public IncludeInputCollection $includes,
        public ProjectionInput $projection,
        public AggregationInputCollection $aggregations,
        public PaginationInput $pagination,
    ) {}
}
```

---

# 12. Input Model ≠ Query AST

El modelo externo todavía no es:

```text id="9bxna9"
Query AST
```

Primero deberá pasar por resolución y política.

---

# 13. Processing pipeline

```text id="z7ftjk"
Decode
  ↓
Structural Validation
  ↓
Normalization
  ↓
Semantic Resolution
  ↓
Capability Validation
  ↓
Authorization
  ↓
Complexity Analysis
  ↓
Resource Budget
  ↓
Canonical Query Request
  ↓
Query AST
```

---

# 14. Decode stage

Convierte input externo a objetos.

No decide autorización.

---

# 15. Structural validation

Verifica:

```text id="hsanav"
expected shape
types
required keys
maximum string length
array shape
maximum nesting
duplicate keys
```

---

# 16. Normalization

Ejemplos:

```text id="hwnwyr"
DESC → desc
Created → created
null textual representation → canonical null
```

sin cambiar significado arbitrariamente.

---

# 17. Semantic resolution

Convierte:

```text id="fc4kff"
"created"
```

a:

```text id="toztao"
User.created_at
```

mediante metadata/policy.

---

# 18. Capability validation

Decide si ese campo puede usarse para:

```text id="6knakl"
FILTER
SORT
SELECT
GROUP
AGGREGATE
SEARCH
```

---

# 19. Capability-specific fields

Un campo puede ser:

```text id="sfjfup"
filterable
```

pero no:

```text id="k3n0zk"
sortable
```

---

# 20. Field capability model

```php id="dlpfhq"
enum QueryFieldCapability
{
    case FILTER;
    case SORT;
    case PROJECT;
    case GROUP;
    case AGGREGATE;
    case SEARCH;
}
```

---

# 21. Allowed field definition

```php id="mg5h6e"
final readonly class AllowedQueryField
{
    public function __construct(
        public ExternalFieldAlias $alias,
        public QueryFieldReference $field,
        public QueryFieldCapabilitySet $capabilities,
        public QueryFieldPolicy $policy,
    ) {}
}
```

---

# 22. External alias

Preferir:

```text id="qrcct7"
created
```

como API pública.

No exponer necesariamente:

```text id="bc5gzi"
created_at
```

---

# 23. Beneficio

Esto desacopla:

```text id="0ye1st"
Public API
```

de:

```text id="s66cs2"
physical DB schema
```

---

# 24. Field registry

```php id="1eeg46"
interface QueryFieldRegistry
{
    public function resolve(
        ExternalFieldAlias $alias,
        QueryInputContext $context,
    ): QueryFieldResolution;
}
```

---

# 25. Registry freeze

Deberá poder compilarse/freeze durante bootstrap.

---

# 26. No request registration

Input externo no podrá registrar fields.

---

# 27. Sensitive field

Ejemplo:

```text id="5mut5j"
password_hash
```

puede existir en ORM metadata pero no aparecer en Query Input Registry.

---

# 28. Metadata presence ≠ external queryability

Fundamental:

> **Que un campo exista en la entidad o base de datos no significa que deba ser consultable dinámicamente desde una API externa.**

---

# 29. Field policies

Un campo podrá ser:

```text id="nlz7e4"
INTERNAL_ONLY
PUBLIC_FILTERABLE
PUBLIC_SORTABLE
AUTHORIZED_ONLY
SENSITIVE
```

---

# 30. QueryFieldPolicy

```php id="ti5fzc"
final readonly class QueryFieldPolicy
{
    public function __construct(
        public bool $filterable,
        public bool $sortable,
        public bool $projectable,
        public bool $groupable,
        public bool $aggregatable,
        public bool $searchable,
        public ?QueryInputAuthorizationRequirement $authorization,
    ) {}
}
```

---

# 31. Operator control

Cada field podrá soportar operadores específicos.

Ejemplo:

```text id="lhss4h"
age:
  EQ
  GT
  GTE
  LT
  LTE
```

pero no necesariamente:

```text id="qe6xjv"
LIKE
REGEX
FULL_TEXT
```

---

# 32. Operator matrix

```text id="00kth8"
Field
×
Operator
=
Allowed / Denied
```

---

# 33. Type-aware operators

Ejemplo:

```text id="ewtdze"
Integer
→ EQ, GT, GTE, LT, LTE, IN

String
→ EQ, LIKE, PREFIX, CONTAINS

Boolean
→ EQ

Date
→ EQ, BEFORE, AFTER, BETWEEN
```

---

# 34. Operator capability registry

```php id="4nbom3"
interface QueryOperatorPolicy
{
    public function allows(
        QueryFieldReference $field,
        QueryInputOperator $operator,
        QueryInputContext $context,
    ): bool;
}
```

---

# 35. Expensive operators

Algunos operadores serán válidos pero costosos.

Ejemplos:

```text id="m0ho85"
CONTAINS
REGEX
FULL_TEXT
JSON_CONTAINS
```

Podrán requerir:

```text id="up6sjs"
higher privilege
lower limits
special budgets
```

---

# 36. FilterInput

```php id="cq6e0e"
final readonly class FilterInput
{
    public function __construct(
        public ExternalFieldAlias $field,
        public ExternalOperatorAlias $operator,
        public mixed $value,
    ) {}
}
```

---

# 37. Resolved filter

```php id="tovf5t"
final readonly class ResolvedFilter
{
    public function __construct(
        public QueryFieldReference $field,
        public QueryInputOperator $operator,
        public QueryParameterValue $value,
    ) {}
}
```

---

# 38. Filter count limits

Una request podrá tener:

```text id="cvm730"
maximum filters
```

Ejemplo conceptual:

```text id="4qf0rl"
max_filters = 20
```

---

# 39. Why

Evita:

```text id="nywnfw"
extremely large predicates
planner pressure
SQL size explosions
parameter explosions
```

---

# 40. Boolean expression depth

Inputs con:

```text id="tz1n9q"
AND
OR
NOT
```

pueden construir árboles complejos.

---

# 41. Boolean depth budget

Deberá limitarse:

```text id="lw5ca8"
max_boolean_depth
max_boolean_nodes
```

---

# 42. Example

```text id="187dn7"
(A OR B)
AND
(C OR D)
AND
...
```

podría crecer mucho.

---

# 43. Query bomb

Un consumidor podría generar estructuras exponenciales si el parser/normalizer expande expresiones.

VoltStack deberá evitar expansiones no acotadas.

---

# 44. Normalization budget

Incluso la normalización debe estar bounded.

---

# 45. IN list security

Una lista:

```text id="pves5w"
id IN (...)
```

puede ser sintácticamente segura pero contener:

```text id="kprys5"
100,000 values
```

---

# 46. IN list limit

```text id="00e8z1"
max_in_values
```

deberá ser configurable/policy-driven.

---

# 47. Platform limits

Además del budget de VoltStack pueden existir límites del DBMS.

---

# 48. Parameter count

El Query Planner deberá considerar:

```text id="h8um47"
maximum parameter count
```

cuando esté disponible mediante capabilities.

---

# 49. Large list alternatives

Para workloads internos podrían existir:

```text id="yfl4y3"
temporary tables
bulk staging
array parameters
JSON table
```

según plataforma.

Pero una API pública no deberá seleccionar estas estrategias arbitrariamente.

---

# 50. Sort security

Dynamic sort deberá usar fields permitidos.

---

# 51. Sort count

Puede limitarse:

```text id="3547xx"
max_sort_fields
```

---

# 52. Expensive sort

Ordenar por:

```text id="g5v4m4"
unindexed large text
computed expression
JSON expression
```

puede ser costoso.

---

# 53. Sort policy

Podrá indicar:

```text id="o8a1o2"
LOW_COST
NORMAL
EXPENSIVE
FORBIDDEN
```

---

# 54. Query cost hints

Estas clasificaciones son hints/policy inputs.

No sustituyen al Database Optimizer.

---

# 55. Stable ordering

Pagination/cursor pagination pueden exigir ordering estable.

Query Input Security podrá imponer:

```text id="61roek"
allowed sortable fields
```

y dejar tie-breakers al Pagination Planner.

---

# 56. Projection security

Un consumidor externo puede solicitar:

```text id="x336rh"
fields=id,name,email
```

---

# 57. Projection policy

No todos los fields projectables deberán estar permitidos.

---

# 58. Sensitive projection

Ejemplo:

```text id="1uyblj"
password_hash
api_secret
internal_notes
```

deberá ser rechazado aunque exista en metadata.

---

# 59. ProjectionInput

```php id="rqvope"
final readonly class ProjectionInput
{
    /** @var list<ExternalFieldAlias> */
    public array $fields;
}
```

---

# 60. Projection count

Puede existir:

```text id="uurbe0"
max_projection_fields
```

---

# 61. Projection "*" semantics

No permitir automáticamente:

```text id="om0ux4"
fields=*
```

como acceso a todas las columnas.

---

# 62. Public default projection

Preferir:

```text id="2yrrng"
defined default projection
```

---

# 63. Relationship inclusion

Input externo podría solicitar:

```text id="1qzq1w"
include=orders
```

---

# 64. Includes

Podrán representar:

```text id="2o76eg"
relationships to preload/expose
```

---

# 65. Include ≠ ORM relationship existence

Una relación ORM existente no es automáticamente includable externamente.

---

# 66. AllowedInclude

```php id="7n4wv8"
final readonly class AllowedInclude
{
    public function __construct(
        public ExternalRelationshipAlias $alias,
        public RelationshipId $relationship,
        public QueryIncludePolicy $policy,
    ) {}
}
```

---

# 67. Relationship depth

Ejemplo:

```text id="09i5yg"
users.orders.items.product.vendor.country...
```

debe tener:

```text id="xm6md6"
max_relationship_depth
```

---

# 68. Why

Protege contra:

```text id="qzw4ws"
query explosion
hydration explosion
memory exhaustion
N+1 amplification
huge response graphs
```

---

# 69. Include count

También:

```text id="ohnao5"
max_includes
```

---

# 70. Relationship branching

No basta profundidad.

Ejemplo:

```text id="m9g68a"
User
├── orders
├── roles
├── permissions
├── sessions
├── devices
└── notifications
```

puede producir gran branching.

---

# 71. Graph complexity

Podrá modelarse:

```text id="v3aeub"
RelationshipGraphCost
```

---

# 72. Include cost

Cada relación podrá tener peso.

Ejemplo:

```text id="m4h8n8"
profile = 1
orders = 3
orders.items = 5
```

---

# 73. Query complexity score

VoltStack podrá calcular un score determinista de complejidad.

---

# 74. Important limitation

`ComplexityScore` no será una estimación precisa del costo del DBMS.

Será:

```text id="339vvh"
resource policy heuristic
```

---

# 75. Complexity contributors

Podrán incluir:

```text id="ik2vro"
filter count
boolean depth
sort count
relationship depth
relationship branching
aggregation count
grouping count
IN size
projection size
search operators
JSON operations
shard fan-out potential
```

---

# 76. QueryComplexity

```php id="s0dyse"
final readonly class QueryComplexity
{
    public function __construct(
        public int $score,
        public QueryComplexityBreakdown $breakdown,
    ) {}
}
```

---

# 77. QueryComplexityPolicy

```php id="481r3j"
interface QueryComplexityPolicy
{
    public function evaluate(
        QueryComplexity $complexity,
        QueryInputContext $context,
    ): QueryComplexityDecision;
}
```

---

# 78. Decisions

```php id="uw0zwu"
enum QueryComplexityDecision
{
    case ALLOW;
    case ALLOW_WITH_LIMITS;
    case DENY;
}
```

---

# 79. Complexity limits by context

Ejemplo:

```text id="6vkw3d"
Public API
  max score = low

Internal admin
  max score = medium

Background export job
  max score = high
```

---

# 80. Complexity ≠ Authorization

Un admin puede estar autorizado pero aún sujeto a resource limits.

---

# 81. Query resource budget

Podrá existir:

```php id="y2vmt0"
final readonly class QueryInputResourceBudget
{
    public function __construct(
        public int $maxFilters,
        public int $maxBooleanDepth,
        public int $maxInValues,
        public int $maxSorts,
        public int $maxIncludes,
        public int $maxRelationshipDepth,
        public int $maxProjectionFields,
        public int $maxAggregations,
        public int $maxGroups,
        public int $maxPageSize,
        public int $maxComplexityScore,
    ) {}
}
```

---

# 82. Budget profiles

```text id="zhl3gf"
PUBLIC
AUTHENTICATED
INTERNAL
ADMIN
BACKGROUND
CUSTOM
```

---

# 83. QueryInputProfile

```php id="uormro"
enum QueryInputProfile
{
    case PUBLIC_API;
    case AUTHENTICATED_API;
    case INTERNAL_API;
    case ADMIN_API;
    case BACKGROUND;
    case CUSTOM;
}
```

---

# 84. Profile ≠ principal

El profile define capacidad general.

El principal sigue siendo una dimensión independiente.

---

# 85. Aggregation security

Input externo puede solicitar:

```text id="1ruz8s"
count
sum
avg
min
max
```

---

# 86. Aggregation sensitivity

Ejemplo:

```text id="vrypl2"
AVG(salary)
```

puede filtrar información incluso sin exponer salarios individuales.

---

# 87. Aggregate authorization

Un field puede ser:

```text id="kpceus"
not projectable
```

pero:

```text id="7sgm8i"
aggregatable
```

o al revés.

---

# 88. Aggregate capability

Debe definirse por field y función.

---

# 89. AggregationPolicy

```php id="odjbbh"
interface AggregationPolicy
{
    public function allows(
        QueryFieldReference $field,
        AggregateFunction $function,
        QueryInputContext $context,
    ): bool;
}
```

---

# 90. Aggregation count

Podrá limitarse:

```text id="7wwqfd"
max_aggregations
```

---

# 91. GROUP BY security

Grouping puede aumentar cardinalidad drásticamente.

---

# 92. Grouping sensitivity

Agrupar por:

```text id="cm0g8h"
user_id
IP
email
device fingerprint
```

puede facilitar reidentificación.

---

# 93. GroupingPolicy

Fields deberán ser explícitamente:

```text id="ebwnya"
groupable
```

---

# 94. Group cardinality budget

Donde sea posible, podrá existir una policy/hint de cardinalidad esperada.

---

# 95. HAVING

HAVING deberá limitarse a aggregations/fields autorizados.

---

# 96. Search security

Search features suelen ser costosas.

---

# 97. Search modes

Ejemplos:

```text id="2j8z87"
EXACT
PREFIX
CONTAINS
FULL_TEXT
```

---

# 98. CONTAINS

Puede provocar:

```text id="x8m1q7"
full table scans
```

si no hay soporte apropiado.

---

# 99. SearchPolicy

Podrá:

```text id="7w65g9"
allow
deny
require minimum length
require indexed field
require full-text capability
```

---

# 100. Minimum search length

Ejemplo:

```text id="5enj1z"
contains search minimum = 3 chars
```

como resource policy configurable.

---

# 101. Wildcard-only queries

Input como:

```text id="idgq2x"
%
```

podrá rechazarse según search policy aunque esté SQL-safe.

---

# 102. Full-text query input

Deberá parsearse y limitarse si se acepta sintaxis avanzada.

---

# 103. Search language

Idioma/configuración será semantic config, no raw query input.

---

# 104. JSON query security

JSON introduce varias capacidades avanzadas.

---

# 105. JSON operations

```text id="aj715y"
path equality
contains
has key
array contains
numeric comparison
```

deberán estar explícitamente permitidas.

---

# 106. JSON path depth

Bounded.

---

# 107. JSON wildcard

Podrá prohibirse en APIs externas por default.

---

# 108. JSON traversal

No permitir acceso libre a cualquier path de documentos sensibles.

---

# 109. AllowedJsonPath

```php id="lhod5v"
final readonly class AllowedJsonPath
{
    public function __construct(
        public ExternalJsonPathAlias $alias,
        public JsonPath $path,
        public QueryFieldCapabilitySet $capabilities,
    ) {}
}
```

---

# 110. Public alias

Ejemplo:

```text id="8k9tgc"
profile.city
```

podría mapear internamente a un path conocido.

---

# 111. Dynamic arbitrary path

Podrá deshabilitarse completamente en APIs públicas.

---

# 112. Date/time input

Debe utilizar parser/type system.

No strings ambiguos sin normalización.

---

# 113. Time zone

Si la API acepta zonas horarias, deberá utilizar identificadores validados.

---

# 114. Enum input

Deberá resolverse contra valores permitidos.

---

# 115. Value object input

Podrá utilizar codecs específicos.

---

# 116. Numeric range

Podrán definirse bounds:

```text id="i70ylk"
page size
amount
age
distance
coordinates
```

según field/policy.

---

# 117. String size

Valores extremadamente grandes podrán rechazarse antes de llegar al driver.

---

# 118. Max value bytes

Un resource budget podrá limitar:

```text id="vhhejt"
max_string_bytes
max_json_bytes
```

---

# 119. Filter null semantics

Input:

```text id="ljgdxz"
value=null
```

deberá resolverse semánticamente.

---

# 120. Null operators

Preferir:

```text id="v6ldja"
IS_NULL
IS_NOT_NULL
```

sobre sobrecargar arbitrariamente `=`.

---

# 121. Duplicate filters

Deberá existir política:

```text id="1jpp2k"
ALLOW
MERGE
REJECT
```

según parser/API.

---

# 122. Conflicting filters

Ejemplo:

```text id="zzan19"
status=active
status=inactive
```

puede ser válido bajo OR o inválido según sintaxis.

Debe determinarse por modelo explícito.

---

# 123. Ambiguous input

Nunca interpretar ambiguamente una forma externa en silencio.

---

# 124. Strict parsing

Preferir:

```text id="xvk9ak"
reject ambiguous query input
```

a heurísticas que cambien su significado.

---

# 125. Unknown field

Default:

```text id="tjzpcy"
REJECT
```

No ignorar silenciosamente.

---

# 126. Why reject

Ignorar puede:

```text id="hns4fz"
hide client bugs
weaken policy assumptions
produce unintended broad queries
```

---

# 127. Unknown operator

También:

```text id="a5nx2c"
REJECT
```

---

# 128. Unknown include

```text id="m5pafd"
REJECT
```

por default.

---

# 129. Unknown sort

```text id="lr1q7p"
REJECT
```

por default.

---

# 130. Compatibility mode

Podría existir una policy para APIs legacy:

```text id="u0gxjo"
IGNORE_UNKNOWN
```

pero no será default.

---

# 131. Query input normalization

Ejemplo:

```text id="wg8vae"
sort=-created
```

puede normalizarse a:

```text id="rkj7iw"
field=created
direction=DESC
```

si la sintaxis externa lo define.

---

# 132. Normalization must be deterministic

Mismo input + mismo schema/policy:

```text id="f2zt6j"
same canonical representation
```

---

# 133. Query Input Fingerprint

Podrá existir un fingerprint de la request canónica.

---

# 134. Use cases

```text id="3l26sv"
rate limiting
debugging
analytics
cache
security audit
```

---

# 135. Fingerprint privacy

No incluir valores sensibles completos.

---

# 136. Authorization-aware fields

Un field puede depender del principal.

Ejemplo:

```text id="wy3uv9"
salary
```

solo disponible para:

```text id="rv8i7d"
HR_MANAGER
```

---

# 137. Access workflow

```text id="iv2u70"
Field alias
↓
Registry
↓
Field definition
↓
Authorization requirement
↓
Authorization Engine
↓
ALLOW/DENY
```

---

# 138. Query Input Security does not define roles

Utiliza contratos de Authorization.

---

# 139. Field-level authorization

Puede aplicarse a:

```text id="50r95q"
filter
sort
project
aggregate
group
```

por separado.

---

# 140. Important inference leak

Aunque un principal no pueda ver:

```text id="becuct"
salary
```

permitir:

```text id="6iykhg"
filter salary > 100000
```

puede filtrar información.

---

# 141. Therefore

```text id="bijdof"
not projectable
```

no implica que sea seguro hacerlo:

```text id="h4ms36"
filterable
```

---

# 142. Sorting leak

Ordenar por campo sensible también puede filtrar información indirectamente.

---

# 143. Grouping leak

Misma consideración.

---

# 144. Aggregation leak

Misma consideración.

---

# 145. SensitiveFieldQueryPolicy

```php id="apxay6"
interface SensitiveFieldQueryPolicy
{
    public function allows(
        QueryFieldReference $field,
        QueryFieldCapability $capability,
        QueryInputContext $context,
    ): SecurityDecision;
}
```

---

# 146. Tenant filtering

Una API multitenant no deberá permitir:

```text id="mzd8xg"
filter tenant_id=other
```

para escapar del tenant context.

---

# 147. Tenant field

Puede estar completamente ausente del public Query Input Registry.

---

# 148. Server-enforced tenant predicate

Preferir:

```text id="ey2o2z"
TenantContext
↓
Server-generated predicate
```

no:

```text id="7ss81o"
client-supplied tenant filter
```

---

# 149. Authorization scope before user filters

Orden lógico:

```text id="fwfpoy"
Authorization/Tenant Scope
       ↓
User Query Filters
       ↓
Pagination
```

---

# 150. Security scope cannot be overridden

Un filter externo no deberá eliminar o sustituir predicates de seguridad.

---

# 151. Predicate provenance

Podrá distinguirse:

```text id="podk18"
SECURITY
SYSTEM
APPLICATION
USER_INPUT
```

---

# 152. Security predicates

No deberán ser removibles por optimizaciones/rewrite que cambien semántica.

---

# 153. Query optimizer

Debe preservar predicate provenance cuando sea necesario para auditoría/seguridad.

---

# 154. Query DSL security

Si VoltStack expone un lenguaje de filtros:

```text id="97gwrf"
status = "active" AND age > 18
```

deberá tener gramática limitada.

---

# 155. Query DSL ≠ SQL

Nunca aceptar SQL completo bajo el nombre de filter DSL.

---

# 156. DSL pipeline

```text id="8oh34h"
Input
↓
Lexer
↓
Parser
↓
Input AST
↓
Semantic Resolver
↓
Capability Policy
↓
Canonical Query Request
↓
Database Query AST
```

---

# 157. DSL grammar limits

Bounded:

```text id="qzf49b"
tokens
nesting
literal size
expression count
```

---

# 158. Parser resource safety

Evitar parsers vulnerables a:

```text id="a5cj86"
pathological recursion
backtracking explosion
huge token streams
```

---

# 159. Query input timeout

Procesamiento de input también podrá tener deadline/budget.

---

# 160. Regex

Si alguna feature admite regex:

```text id="xsckcf"
regex complexity
```

puede ser vector de DoS tanto en PHP como DB.

---

# 161. Default public policy

Regex deberá:

```text id="ii8hwq"
disabled
```

o altamente restringido por default.

---

# 162. Pagination security

Input:

```text id="f376h4"
page
per_page
```

deberá validarse y boundarse.

---

# 163. maxPageSize

Debe venir de policy.

---

# 164. Huge offsets

Pueden ser:

```text id="o5mdhg"
WARN
REJECT
SUGGEST_CURSOR
```

según el Pagination System.

---

# 165. Cursor input

Debe respetar el sistema 200:

```text id="0qsmx3"
signed
bound
typed
versioned
```

cuando sea externo.

---

# 166. Cursor ≠ authorization token

Query Input Security deberá verificar que la nueva request siga estando autorizada.

---

# 167. Count exposure

Solicitar:

```text id="nh37sx"
include_total=true
```

puede revelar cardinalidad sensible.

---

# 168. CountPolicy

Podrá decidir:

```text id="p5mr5t"
ALLOW_EXACT
ALLOW_ESTIMATED
ALLOW_NONE
DENY
```

---

# 169. Total ≠ harmless metadata

Conteos pueden filtrar existencia de registros.

---

# 170. Distinct counts

Particularmente sensibles en conjuntos pequeños.

---

# 171. Minimum aggregation group size

Una policy avanzada podría imponer:

```text id="gfhe7g"
minimum group size
```

para determinados analytics sensibles.

---

# 172. Privacy integration

No será responsabilidad central completa del Database Query Input System, pero deberá admitir integration hooks.

---

# 173. Query complexity analyzer

```php id="luj1zt"
interface QueryInputComplexityAnalyzer
{
    public function analyze(
        CanonicalQueryInput $input,
        QueryInputContext $context,
    ): QueryComplexity;
}
```

---

# 174. Complexity breakdown

```php id="4pvoyi"
final readonly class QueryComplexityBreakdown
{
    public function __construct(
        public int $filters,
        public int $booleanDepth,
        public int $sorts,
        public int $includes,
        public int $relationshipDepth,
        public int $aggregations,
        public int $groups,
        public int $inValues,
        public int $searchCost,
        public int $distributionCost,
    ) {}
}
```

---

# 175. Query cost category

```php id="eijx8e"
enum QueryInputCostClass
{
    case LOW;
    case MEDIUM;
    case HIGH;
    case EXTREME;
}
```

---

# 176. Cost class ≠ DB optimizer cost

No confundirlos.

---

# 177. Budget evaluation

```text id="rkiu7n"
Complexity
+
Policy
+
Principal
+
Workload
+
Endpoint
=
Decision
```

---

# 178. Decision

```php id="ujkewp"
enum QueryInputDecision
{
    case ALLOW;
    case ALLOW_WITH_ADJUSTMENTS;
    case DENY;
}
```

---

# 179. Adjustments

Ejemplos permitidos:

```text id="fb74bh"
cap page size
disable total count
remove optional expansion
```

solo si la API define claramente este comportamiento.

---

# 180. Silent semantic mutation

No deberá cambiarse silenciosamente:

```text id="5o41m9"
filter
sort
authorization scope
```

---

# 181. Better default

Para solicitudes inválidas:

```text id="zv3tnb"
reject explicitly
```

---

# 182. Query input policy

```php id="7gza23"
interface QueryInputSecurityPolicy
{
    public function evaluate(
        CanonicalQueryInput $input,
        QueryInputContext $context,
    ): QueryInputSecurityDecision;
}
```

---

# 183. QueryInputContext

```php id="1cpe7z"
final readonly class QueryInputContext
{
    public function __construct(
        public QueryInputProfile $profile,
        public SecurityPrincipalReference $principal,
        public ?TenantContextReference $tenant,
        public QueryWorkloadClass $workload,
        public QueryInputResourceBudget $budget,
        public SecurityPolicyGeneration $policyGeneration,
    ) {}
}
```

---

# 184. Context immutability

Deberá ser readonly.

---

# 185. Policy generation

Permite diagnóstico y caching seguro de decisiones.

---

# 186. QueryInputSecurityDecision

```php id="zwdkpj"
final readonly class QueryInputSecurityDecision
{
    public function __construct(
        public QueryInputDecision $decision,
        public QueryInputFindingCollection $findings,
        public QueryComplexity $complexity,
        public QueryInputResourceBudget $effectiveBudget,
    ) {}
}
```

---

# 187. Findings

Ejemplos:

```text id="uxy9fo"
FIELD_NOT_ALLOWED
OPERATOR_NOT_ALLOWED
SORT_NOT_ALLOWED
INCLUDE_NOT_ALLOWED
PROJECTION_NOT_ALLOWED
AGGREGATION_NOT_ALLOWED
RELATIONSHIP_DEPTH_EXCEEDED
FILTER_LIMIT_EXCEEDED
IN_LIST_LIMIT_EXCEEDED
PAGE_SIZE_EXCEEDED
QUERY_COMPLEXITY_EXCEEDED
SENSITIVE_FIELD_ACCESS
COUNT_NOT_ALLOWED
```

---

# 188. Finding severity

```php id="bb6xqe"
enum QueryInputFindingSeverity
{
    case NOTICE;
    case WARNING;
    case HIGH;
    case CRITICAL;
}
```

---

# 189. Finding ≠ malicious intent

Nunca asumir.

---

# 190. Security diagnostics

Podrán generar códigos:

```text id="czrpkp"
DB_QUERY_INPUT_FIELD_DENIED
DB_QUERY_INPUT_OPERATOR_DENIED
DB_QUERY_INPUT_SORT_DENIED
DB_QUERY_INPUT_INCLUDE_DENIED
DB_QUERY_INPUT_COMPLEXITY_LIMIT
DB_QUERY_INPUT_PAGE_LIMIT
DB_QUERY_INPUT_SENSITIVE_FIELD
```

---

# 191. Error response

Para APIs externas:

```text id="lra9qp"
Invalid query parameter.
```

o un error estructurado seguro.

---

# 192. Development detail

En desarrollo:

```text id="mo0jjz"
Field "salary" cannot be used for sorting in profile PUBLIC_API.
```

---

# 193. API error format

Puede exponer:

```json id="w3vsb2"
{
  "code": "QUERY_INPUT_SORT_DENIED",
  "field": "salary"
}
```

si el field alias no es sensible.

---

# 194. Internal schema names

No deberán filtrarse innecesariamente.

---

# 195. QueryInputException

Jerarquía:

```text id="fm8msk"
QueryInputSecurityException
├── QueryInputDecodeException
├── QueryInputValidationException
├── QueryFieldNotAllowedException
├── QueryOperatorNotAllowedException
├── QuerySortNotAllowedException
├── QueryIncludeNotAllowedException
├── QueryProjectionNotAllowedException
├── QueryAggregationNotAllowedException
├── QueryComplexityExceededException
├── QueryResourceLimitException
└── QuerySensitiveFieldAccessException
```

---

# 196. Query Input vs Query Builder API

Developer-authored internal queries podrán seguir usando Query Builder directamente.

Query Input Security será especialmente relevante cuando:

```text id="fvt7ve"
external input
→
dynamic query
```

---

# 197. Internal API bypass

No deberá existir un bypass accidental.

La aplicación deberá elegir explícitamente:

```text id="gz25gu"
trusted internal query construction
```

o:

```text id="pyombk"
external query input pipeline
```

---

# 198. Public endpoint helper

Podría existir:

```php id="fqw9a3"
$queryRequest = QueryInput::fromRequest($request)
    ->using(UserQueryPolicy::class)
    ->resolve();
```

---

# 199. Query policy definition

Ejemplo conceptual:

```php id="1r0j8h"
final class UserQueryPolicy extends QueryInputDefinition
{
    protected function fields(): array
    {
        return [
            Field::make('name')
                ->filterable()
                ->sortable(),

            Field::make('email')
                ->filterable(),

            Field::make('created')
                ->mapsTo('created_at')
                ->sortable(),
        ];
    }
}
```

---

# 200. API ergonomics

Deberá ser sencillo declarar:

```text id="ht51yx"
allowed fields
allowed operators
allowed includes
budgets
```

sin construir manualmente todo el security pipeline.

---

# 201. Compile definitions

Las definiciones deberán compilarse durante bootstrap cuando sea posible.

---

# 202. QueryInputDefinition

```php id="t82oc6"
abstract class QueryInputDefinition
{
    abstract public function configure(
        QueryInputDefinitionBuilder $builder
    ): void;
}
```

---

# 203. Compiled definition

```php id="4pt5ld"
final readonly class CompiledQueryInputDefinition
{
    // immutable field/operator/include/budget metadata
}
```

---

# 204. Persistent runtime safety

Compiled definitions podrán compartirse.

---

# 205. Request state

No compartir:

```text id="jjr48i"
principal
tenant
input
resolved filters
budget counters
authorization results
```

---

# 206. No static request input

Prohibido:

```php id="14fh8z"
static $currentFilters;
```

---

# 207. Coroutine isolation

OpenSwoole:

```text id="0vc27f"
Coroutine A → QueryInputContext A
Coroutine B → QueryInputContext B
```

---

# 208. FrankenPHP

Fresh context por request.

---

# 209. RoadRunner

Fresh context por job/request.

---

# 210. Cache of compiled policies

Seguro si key incluye:

```text id="4hdtiu"
definition generation
metadata generation
security policy generation
```

---

# 211. Decision caching

Más delicado.

Una decisión que depende de principal/tenant no deberá reutilizarse incorrectamente.

---

# 212. QueryInput Decision Cache

Si existe, deberá incluir:

```text id="b8wt4j"
principal scope
tenant scope
policy generation
input fingerprint
```

según corresponda.

---

# 213. No cross-user decision reuse

Regla obligatoria.

---

# 214. Rate limiting

Query Input Security podrá proporcionar fingerprints/cost class al Rate Limiting System.

---

# 215. Example

Requests repetitivas con:

```text id="r3svs0"
EXTREME query complexity
```

podrán ser tratadas de manera distinta por capas superiores.

---

# 216. Query Input Security ≠ Rate Limiter

Solo provee evidencia/cost classification.

---

# 217. Sharding

Inputs que afectan shard key pueden alterar fan-out.

---

# 218. Shard-aware input policy

Un filtro sobre shard key puede ser obligatorio en ciertos endpoints.

---

# 219. Example

```text id="p03u8z"
analytics endpoint
```

puede permitir global fan-out.

Pero:

```text id="aa2bah"
interactive tenant endpoint
```

puede exigir single-shard routing.

---

# 220. Distribution budget

Podrá existir:

```text id="gkpj4l"
max_shards
```

dentro del resource budget.

---

# 221. Unknown routing

Si el input produce:

```text id="or5c1y"
UNKNOWN routing
```

una policy puede rechazarlo en endpoints sensibles.

---

# 222. Tenant scope

Un endpoint tenant-bound debería tener:

```text id="za3ihw"
max_shards = tenant-owned shard count
```

normalmente uno según topology.

---

# 223. Read/write risk

Dynamic query input público debería ser read-only por default.

---

# 224. QueryInputOperation

```php id="ufco6x"
enum QueryInputOperation
{
    case READ;
    case AGGREGATE;
    case EXPORT;
}
```

---

# 225. Mutations

Dynamic external mutations deberán usar un sistema separado y más estricto.

No reutilizar ciegamente generic filter input para:

```text id="bm9wy6"
UPDATE
DELETE
```

---

# 226. Why

Una filter API pensada para búsqueda puede no tener safeguards suficientes para operaciones destructivas.

---

# 227. Bulk mutation input

Deberá pasar además por:

```text id="mv8nbk"
Data Access Security
Bulk mutation safeguards
Affected-row policies
Audit
```

---

# 228. Query input and cache

Canonical Query Input puede contribuir al cache fingerprint.

---

# 229. Security scopes in cache

Authorization/tenant context deberá seguir formando parte de la identidad cuando afecte resultado.

---

# 230. Query input fingerprint ≠ result cache key

Es solo un componente potencial.

---

# 231. Telemetry

Eventos útiles:

```text id="fxp47v"
query_input.received
query_input.resolved
query_input.rejected
query_input.complexity
```

---

# 232. Cardinality

No registrar input completo como label.

---

# 233. Safe telemetry fields

```text id="jcy9m9"
profile
filter_count
sort_count
include_count
relationship_depth
complexity_class
decision
finding_code
```

---

# 234. Sensitive field telemetry

Si un intento usa field sensible:

```text id="yvtkpd"
finding=SENSITIVE_FIELD_ACCESS
```

puede ser suficiente.

No siempre debe registrar el nombre real.

---

# 235. Audit

Dependiendo de policy, accesos rechazados a campos sensibles podrán auditarse.

---

# 236. Audit ≠ every invalid filter

Evitar ruido excesivo.

---

# 237. Security levels

Podrán existir:

```text id="z7249b"
NORMAL_INVALID_INPUT
SECURITY_RELEVANT
HIGH_RISK
```

---

# 238. QueryInputSecurityClassification

```php id="7lwc7k"
enum QueryInputSecurityClassification
{
    case NORMAL_INVALID_INPUT;
    case POLICY_VIOLATION;
    case SENSITIVE_ACCESS_ATTEMPT;
    case RESOURCE_ABUSE_CANDIDATE;
}
```

---

# 239. Candidate ≠ attack

Mantener terminología prudente.

---

# 240. Developer Debug Toolbar

Podrá mostrar:

```text id="b2vcdk"
Query Input

Filters requested: 8
Filters accepted: 7
Rejected: 1

Complexity:
MEDIUM

Budget:
42 / 100
```

solo para contexts autorizados.

---

# 241. Rejected query

Normalmente no habrá QueryExecuting.

Pero podrá existir un:

```text id="slbbq4"
QueryInputRejected
```

debug record.

---

# 242. Input visibility

Valores sensibles permanecerán redactados.

---

# 243. Testing strategy

Debe cubrir:

```text id="80kmb1"
unknown field
unknown operator
forbidden field
forbidden capability
sensitive field filtering
sensitive field sorting
projection restrictions
aggregation restrictions
include restrictions
relationship depth
filter count
IN limits
sort count
page size
count policy
JSON path limits
query DSL depth
complexity budgets
tenant isolation
authorization-aware fields
persistent runtime isolation
```

---

# 244. Example test

```php id="tiu1ak"
$input = QueryInput::fromArray([
    'sort' => ['salary'],
]);

$result = $resolver->resolve(
    $input,
    $publicContext,
);

expect($result->decision)
    ->toBe(QueryInputDecision::DENY);
```

---

# 245. Complexity test

```php id="j9f2pc"
$input = QueryInputFactory::make([
    'filters' => 100,
]);

expect(fn () => $resolver->resolve($input, $context))
    ->toThrow(QueryComplexityExceededException::class);
```

---

# 246. Relationship depth test

```text id="nnr6v3"
user.orders.items.product.vendor.country
```

con:

```text id="w3wtkx"
max depth = 3
```

deberá rechazarse.

---

# 247. Sensitive inference test

Si `salary` no es visible, probar también:

```text id="2uf1xz"
filter salary
sort salary
group salary
aggregate salary
```

no solo projection.

---

# 248. Tenant override test

Input:

```text id="ncv8vq"
tenant_id=other
```

no deberá sustituir el TenantContext server-side.

---

# 249. Fuzzing

Targets:

```text id="ug9tgb"
query input decoder
DSL parser
boolean expression parser
JSON path parser
sort parser
include parser
projection parser
```

---

# 250. Pathological nesting

Fuzz tests deberán intentar:

```text id="pswmyv"
thousands of nested parentheses
deep JSON
huge repeated arrays
duplicate keys
invalid unicode
oversized tokens
```

---

# 251. Parser safety

Input parser no deberá consumir memoria/CPU no acotada.

---

# 252. Query input size

La capa HTTP puede imponer un body limit general.

El Query Input System además tendrá sus propios límites estructurales.

---

# 253. QueryInputBudgetException

Debe indicar internamente qué límite se excedió.

Externamente puede devolver error genérico controlado.

---

# 254. Query input policy composition

Podrán combinarse:

```text id="mr4xxv"
BaseDefinition
+
EndpointPolicy
+
PrincipalPolicy
+
TenantPolicy
+
WorkloadPolicy
```

---

# 255. Composition semantics

Deben ser deterministas.

---

# 256. Most restrictive wins

Para capacidades obligatorias, preferir:

```text id="v9c9sm"
effective permission
=
intersection
```

---

# 257. Example

Si:

```text id="rt2a2g"
Base allows salary filter
Endpoint denies salary filter
```

resultado:

```text id="okdscd"
DENY
```

---

# 258. Budget composition

Podrá usar:

```text id="9jqazy"
minimum effective limit
```

entre policies aplicables.

---

# 259. Example

```text id="f5ekci"
Global max page = 500
Public endpoint = 100
Principal policy = 200
```

resultado:

```text id="36vv6c"
100
```

---

# 260. Policy override

Overrides que amplían permisos deberán ser explícitos y autorizados.

---

# 261. QueryInputDefinition inheritance

Evitar herencia compleja difícil de razonar.

Preferir composición declarativa.

---

# 262. Endpoint-specific definition

Ejemplo:

```text id="pdkrvj"
UserPublicListQuery
UserAdminListQuery
UserExportQuery
```

pueden ser definiciones distintas.

---

# 263. One universal query policy

No es recomendable para toda la aplicación.

---

# 264. Public API versioning

Cambios en allowed fields/operators pueden ser breaking API changes.

---

# 265. Query definition version

Podrá existir:

```text id="hmc4tg"
QueryInputDefinitionVersion
```

para APIs versionadas.

---

# 266. Unknown deprecated field

Podrá devolver un diagnostic específico si la API soporta deprecations.

---

# 267. Alias migration

Ejemplo:

```text id="8o14mi"
created_on
→ deprecated alias
→ created
```

podrá resolverse durante un periodo controlado.

---

# 268. Deprecated alias ≠ physical schema alias

Mantener independencia.

---

# 269. Performance model

La mayoría de checks deberían ocurrir sobre:

```text id="z6jw4d"
compiled immutable policy metadata
```

---

# 270. No reflection per input

Evitar reflexión sobre entidades en cada request.

---

# 271. Precomputed lookup

External aliases:

```text id="aplr9l"
hash-map lookup
```

idealmente.

---

# 272. Complexity analysis

Deberá ser aproximadamente:

```text id="s6uddp"
O(number of input nodes)
```

y bounded por input limits.

---

# 273. Security must precede expensive query planning

Rechazar inputs inválidos antes de:

```text id="xyxyiz"
connection acquisition
full query optimization
execution
```

cuando sea posible.

---

# 274. Proposed namespace

```text id="ruv69a"
VoltStack\Quantum\Database\Security\QueryInput
```

---

# 275. Directory structure

```text id="vufbrp"
src/Quantum/Database/Security/QueryInput/
│
├── Contract/
│   ├── QueryInputDecoder.php
│   ├── QueryInputSecurityPolicy.php
│   ├── QueryFieldRegistry.php
│   ├── QueryOperatorPolicy.php
│   ├── AggregationPolicy.php
│   ├── SensitiveFieldQueryPolicy.php
│   └── QueryInputComplexityAnalyzer.php
│
├── Model/
│   ├── QueryInputModel.php
│   ├── CanonicalQueryInput.php
│   ├── FilterInput.php
│   ├── SortInput.php
│   ├── IncludeInput.php
│   ├── ProjectionInput.php
│   ├── AggregationInput.php
│   └── PaginationInput.php
│
├── Field/
│   ├── AllowedQueryField.php
│   ├── ExternalFieldAlias.php
│   ├── QueryFieldReference.php
│   ├── QueryFieldCapability.php
│   ├── QueryFieldCapabilitySet.php
│   ├── QueryFieldPolicy.php
│   └── QueryFieldResolution.php
│
├── Operator/
│   ├── QueryInputOperator.php
│   ├── ExternalOperatorAlias.php
│   ├── OperatorRegistry.php
│   └── OperatorResolution.php
│
├── Include/
│   ├── AllowedInclude.php
│   ├── ExternalRelationshipAlias.php
│   └── QueryIncludePolicy.php
│
├── Json/
│   ├── AllowedJsonPath.php
│   ├── ExternalJsonPathAlias.php
│   └── QueryJsonPolicy.php
│
├── Complexity/
│   ├── QueryComplexity.php
│   ├── QueryComplexityBreakdown.php
│   ├── QueryInputCostClass.php
│   ├── QueryComplexityPolicy.php
│   └── DefaultQueryInputComplexityAnalyzer.php
│
├── Budget/
│   ├── QueryInputResourceBudget.php
│   ├── QueryInputBudgetResolver.php
│   └── QueryInputBudgetUsage.php
│
├── Context/
│   ├── QueryInputContext.php
│   ├── QueryInputProfile.php
│   └── QueryInputOperation.php
│
├── Policy/
│   ├── QueryInputSecurityDecision.php
│   ├── QueryInputDecision.php
│   ├── QueryInputFinding.php
│   ├── QueryInputFindingCollection.php
│   ├── QueryInputFindingSeverity.php
│   └── CompiledQueryInputPolicy.php
│
├── Definition/
│   ├── QueryInputDefinition.php
│   ├── QueryInputDefinitionBuilder.php
│   ├── CompiledQueryInputDefinition.php
│   └── QueryInputDefinitionRegistry.php
│
├── Resolution/
│   ├── QueryInputResolver.php
│   ├── QueryFieldResolver.php
│   ├── QueryOperatorResolver.php
│   ├── QueryIncludeResolver.php
│   └── QueryProjectionResolver.php
│
├── DSL/
│   ├── QueryDslLexer.php
│   ├── QueryDslParser.php
│   ├── QueryDslAst.php
│   └── QueryDslLimits.php
│
├── Diagnostic/
│   ├── QueryInputDiagnostic.php
│   └── QueryInputDiagnosticCode.php
│
├── Telemetry/
│   └── QueryInputSecurityTelemetry.php
│
├── Testing/
│   ├── QueryInputAssertions.php
│   ├── QueryInputFuzzCorpus.php
│   ├── FakeQueryInputPolicy.php
│   └── QueryInputDefinitionTestKit.php
│
└── Exception/
    ├── QueryInputSecurityException.php
    ├── QueryInputDecodeException.php
    ├── QueryInputValidationException.php
    ├── QueryFieldNotAllowedException.php
    ├── QueryOperatorNotAllowedException.php
    ├── QuerySortNotAllowedException.php
    ├── QueryIncludeNotAllowedException.php
    ├── QueryProjectionNotAllowedException.php
    ├── QueryAggregationNotAllowedException.php
    ├── QueryComplexityExceededException.php
    ├── QueryResourceLimitException.php
    └── QuerySensitiveFieldAccessException.php
```

---

# 276. Integration pipeline

```text id="wg9dbs"
HTTP/API Input
      ↓
QueryInputDecoder
      ↓
QueryInputModel
      ↓
Structural Validation
      ↓
Normalization
      ↓
Compiled Query Input Definition
      ↓
Field / Operator / Include Resolution
      ↓
Authorization-Aware Capability Checks
      ↓
Complexity Analysis
      ↓
Resource Budget
      ↓
QueryInputSecurityDecision
      ↓
CanonicalQueryInput
      ↓
Query Builder / Query AST
      ↓
SQL Injection Prevention
      ↓
Query Engine
```

---

# 277. Security ordering

El sistema deberá preservar:

```text id="a09p06"
External Input
↓
Query Input Security
↓
Typed Query AST
↓
SQL Injection Prevention
↓
Optimization
↓
Compilation
↓
Execution
```

---

# 278. Architectural invariants

## DB-QINPUT-001
External query input será untrusted por default.

## DB-QINPUT-002
Query Input Security será distinto de SQL Injection Prevention.

## DB-QINPUT-003
Query Input Security será distinto de Authorization.

## DB-QINPUT-004
Query Input Security será distinto de simple validation.

## DB-QINPUT-005
QueryInputModel será distinto de Query AST.

## DB-QINPUT-006
External fields requerirán registry/resolution.

## DB-QINPUT-007
ORM field existence no implicará external queryability.

## DB-QINPUT-008
Physical column existence no implicará external queryability.

## DB-QINPUT-009
Filter capability será separada de sort capability.

## DB-QINPUT-010
Projection capability será separada de filter capability.

## DB-QINPUT-011
Group capability será separada de aggregate capability.

## DB-QINPUT-012
Search capability será separada de general filter capability.

## DB-QINPUT-013
Fields podrán tener authorization requirements.

## DB-QINPUT-014
Unknown fields serán rechazados por default.

## DB-QINPUT-015
Unknown operators serán rechazados por default.

## DB-QINPUT-016
Unknown sorts serán rechazados por default.

## DB-QINPUT-017
Unknown includes serán rechazados por default.

## DB-QINPUT-018
External aliases estarán desacoplados de physical schema names.

## DB-QINPUT-019
Input no registrará nuevos fields.

## DB-QINPUT-020
Input no registrará nuevos operators.

## DB-QINPUT-021
Input no registrará nuevas relationships.

## DB-QINPUT-022
Operators estarán limitados por field/type.

## DB-QINPUT-023
Expensive operators podrán tener policy adicional.

## DB-QINPUT-024
Filter count será bounded.

## DB-QINPUT-025
Boolean expression depth será bounded.

## DB-QINPUT-026
Boolean node count será bounded.

## DB-QINPUT-027
Normalization será bounded.

## DB-QINPUT-028
IN list size será bounded.

## DB-QINPUT-029
Parameter count considerará platform capabilities.

## DB-QINPUT-030
Sort count será bounded.

## DB-QINPUT-031
Expensive sort podrá ser restringido.

## DB-QINPUT-032
Projection field count será bounded.

## DB-QINPUT-033
Wildcard projection no implicará all database columns.

## DB-QINPUT-034
Default projection será explícita.

## DB-QINPUT-035
ORM relationship existence no implicará includable externally.

## DB-QINPUT-036
Include count será bounded.

## DB-QINPUT-037
Relationship depth será bounded.

## DB-QINPUT-038
Relationship branching será considerado.

## DB-QINPUT-039
Relationship graph podrá tener cost weight.

## DB-QINPUT-040
Query complexity será distinta de DB optimizer cost.

## DB-QINPUT-041
Complexity score será policy heuristic.

## DB-QINPUT-042
Complexity tendrá breakdown explicable.

## DB-QINPUT-043
Complexity policy será context-aware.

## DB-QINPUT-044
Resource budgets serán explícitos.

## DB-QINPUT-045
Resource budgets podrán variar por profile.

## DB-QINPUT-046
Profile será distinto de principal.

## DB-QINPUT-047
Aggregation access será explícito.

## DB-QINPUT-048
Aggregate function permission podrá depender del field.

## DB-QINPUT-049
Sensitive aggregations podrán ser rechazadas.

## DB-QINPUT-050
Grouping access será explícito.

## DB-QINPUT-051
Grouping podrá considerarse sensitive inference channel.

## DB-QINPUT-052
HAVING respetará mismas capability rules.

## DB-QINPUT-053
Search modes serán explícitos.

## DB-QINPUT-054
Wildcard-only searches podrán ser rechazadas.

## DB-QINPUT-055
Minimum search length será configurable.

## DB-QINPUT-056
Regex estará deshabilitado/restringido por default en public input.

## DB-QINPUT-057
JSON query capabilities serán explícitas.

## DB-QINPUT-058
JSON path depth será bounded.

## DB-QINPUT-059
Arbitrary JSON traversal podrá deshabilitarse.

## DB-QINPUT-060
Allowed JSON paths podrán usar aliases.

## DB-QINPUT-061
Date/time input utilizará Type System.

## DB-QINPUT-062
Timezone identifiers serán validados.

## DB-QINPUT-063
Enum values serán resueltos explícitamente.

## DB-QINPUT-064
Large strings podrán ser rechazados antes de DB execution.

## DB-QINPUT-065
JSON value size podrá ser bounded.

## DB-QINPUT-066
NULL semantics serán explícitas.

## DB-QINPUT-067
Ambiguous input será rechazado por default.

## DB-QINPUT-068
Compatibility ignore-unknown no será default.

## DB-QINPUT-069
Normalization será deterministic.

## DB-QINPUT-070
Input fingerprints no contendrán sensitive values completos.

## DB-QINPUT-071
Field visibility será distinta de filterability.

## DB-QINPUT-072
Field visibility será distinta de sortability.

## DB-QINPUT-073
Field visibility será distinta de groupability.

## DB-QINPUT-074
Field visibility será distinta de aggregatability.

## DB-QINPUT-075
Sensitive fields podrán producir inference leaks aun sin projection.

## DB-QINPUT-076
Sensitive filter access requerirá policy.

## DB-QINPUT-077
Sensitive sort access requerirá policy.

## DB-QINPUT-078
Sensitive grouping access requerirá policy.

## DB-QINPUT-079
Sensitive aggregation access requerirá policy.

## DB-QINPUT-080
Tenant selector externo no sustituirá server TenantContext.

## DB-QINPUT-081
Tenant security predicate será server-generated.

## DB-QINPUT-082
User filters no podrán remover security predicates.

## DB-QINPUT-083
Predicate provenance podrá conservarse.

## DB-QINPUT-084
Security predicates deberán sobrevivir rewrites semánticamente equivalentes.

## DB-QINPUT-085
Query DSL será distinto de SQL.

## DB-QINPUT-086
Query DSL tendrá lexer/parser/AST.

## DB-QINPUT-087
Query DSL grammar será bounded.

## DB-QINPUT-088
Parser resource usage será bounded.

## DB-QINPUT-089
Pagination page size será bounded.

## DB-QINPUT-090
Huge offsets podrán restringirse.

## DB-QINPUT-091
External cursor seguirá requiriendo authorization/context validation.

## DB-QINPUT-092
Cursor nunca será authorization token.

## DB-QINPUT-093
Exact total counts podrán restringirse.

## DB-QINPUT-094
Total count será potencial information channel.

## DB-QINPUT-095
Distinct counts podrán requerir policy adicional.

## DB-QINPUT-096
Query complexity analyzer no ejecutará queries.

## DB-QINPUT-097
Complexity decision será distinta de Authorization decision.

## DB-QINPUT-098
ALLOW_WITH_ADJUSTMENTS no modificará silenciosamente semántica sensible.

## DB-QINPUT-099
Invalid security input será rechazado explícitamente por default.

## DB-QINPUT-100
Query input policies serán composables.

## DB-QINPUT-101
Policy composition será determinista.

## DB-QINPUT-102
Effective capability usará intersección cuando corresponda.

## DB-QINPUT-103
Effective numeric budgets tenderán al límite más restrictivo.

## DB-QINPUT-104
Permission-expanding overrides serán explícitos.

## DB-QINPUT-105
Endpoint-specific definitions serán soportadas.

## DB-QINPUT-106
No se requerirá una única universal query policy.

## DB-QINPUT-107
Public query definitions podrán versionarse.

## DB-QINPUT-108
Deprecated aliases podrán soportarse explícitamente.

## DB-QINPUT-109
Compiled definitions serán immutable.

## DB-QINPUT-110
Compiled definitions podrán compartirse entre requests.

## DB-QINPUT-111
Principal no estará en compiled definition.

## DB-QINPUT-112
Tenant no estará en compiled definition mutable.

## DB-QINPUT-113
Request filters no serán static state.

## DB-QINPUT-114
Persistent runtime input context será scoped.

## DB-QINPUT-115
OpenSwoole query input context será coroutine-safe.

## DB-QINPUT-116
FrankenPHP tendrá fresh query input context por request.

## DB-QINPUT-117
RoadRunner tendrá fresh context por request/job.

## DB-QINPUT-118
Decision caching no cruzará principals incorrectamente.

## DB-QINPUT-119
Decision caching no cruzará tenants incorrectamente.

## DB-QINPUT-120
Decision caching respetará policy generation.

## DB-QINPUT-121
Query input complexity podrá alimentar rate limiting.

## DB-QINPUT-122
Query Input Security no será Rate Limiter.

## DB-QINPUT-123
Shard fan-out será considerado en resource policy.

## DB-QINPUT-124
max_shards podrá ser policy.

## DB-QINPUT-125
Unknown shard routing podrá rechazarse.

## DB-QINPUT-126
Public dynamic query input será read-only por default.

## DB-QINPUT-127
Generic read filter APIs no se reutilizarán ciegamente para destructive mutations.

## DB-QINPUT-128
Bulk mutation requerirá security layers adicionales.

## DB-QINPUT-129
Query input fingerprint será distinto de result cache key.

## DB-QINPUT-130
Cache seguirá incorporando security context cuando corresponda.

## DB-QINPUT-131
Telemetry labels serán bounded.

## DB-QINPUT-132
Telemetry no registrará full query input por default.

## DB-QINPUT-133
Sensitive field attempts no requerirán revelar field physical name.

## DB-QINPUT-134
Audit será selective/policy-driven.

## DB-QINPUT-135
Invalid input no será automáticamente considerado malicious.

## DB-QINPUT-136
Developer diagnostics podrán ser más detallados que external errors.

## DB-QINPUT-137
External errors minimizarán internal schema details.

## DB-QINPUT-138
Query input rejection no emitirá QueryExecuting.

## DB-QINPUT-139
Query input rejection podrá generar telemetry/debug record.

## DB-QINPUT-140
Query Input Security tendrá fuzz tests.

## DB-QINPUT-141
DSL parser tendrá fuzz tests.

## DB-QINPUT-142
Boolean parser tendrá fuzz tests.

## DB-QINPUT-143
JSON input parser tendrá fuzz tests.

## DB-QINPUT-144
Pathological nested inputs serán testeados.

## DB-QINPUT-145
Parser OOM/CPU amplification será parte del threat model.

## DB-QINPUT-146
Security processing ocurrirá antes de DB resource acquisition cuando sea posible.

## DB-QINPUT-147
No se realizará reflection completa por request.

## DB-QINPUT-148
Alias lookup deberá poder precompilarse.

## DB-QINPUT-149
Complexity analysis será aproximadamente linear al input bounded.

## DB-QINPUT-150
Safe API será ergonómica.

## DB-QINPUT-151
Security definitions serán declarativas.

## DB-QINPUT-152
Query input security no dependerá de HTTP concrete classes.

## DB-QINPUT-153
Query input security podrá usarse desde CLI/API/jobs.

## DB-QINPUT-154
Query input security seguirá siendo provider-agnostic.

## DB-QINPUT-155
Query Input Security no generará SQL.

## DB-QINPUT-156
Query Input Security no ejecutará SQL.

## DB-QINPUT-157
Query Input Security producirá canonical semantic requests.

## DB-QINPUT-158
Canonical requests serán convertidas posteriormente a Query AST.

## DB-QINPUT-159
SQL Injection Prevention seguirá siendo una capa posterior.

## DB-QINPUT-160
Authorization seguirá siendo una capa colaboradora independiente.

## DB-QINPUT-161
Resource governance será parte esencial de query input security.

## DB-QINPUT-162
Safe-from-injection no implicará safe-to-execute.

## DB-QINPUT-163
Authenticated input no implicará unrestricted query capability.

## DB-QINPUT-164
Authorized user no implicará unlimited resource budget.

## DB-QINPUT-165
Internal admin no implicará bypass automático de all limits.

## DB-QINPUT-166
UNKNOWN capability no será ALLOW.

## DB-QINPUT-167
UNKNOWN field resolution no será sanitize-and-use.

## DB-QINPUT-168
Security scope será server-controlled.

## DB-QINPUT-169
External query input será constrained capability, no general Query Builder access.

## DB-QINPUT-170
Query Input Security será la frontera oficial entre external query intent y internal Query AST.

---

# 279. Anti-patterns

## 279.1 Exponer Query Builder directamente

Incorrecto:

```php id="lsu6as"
$query->where(
    $request->input('field'),
    $request->input('operator'),
    $request->input('value'),
);
```

Correcto:

```text id="4hdlhl"
request
↓
Query Input Policy
↓
resolved field/operator/value
↓
Query AST
```

---

## 279.2 Todos los fields filtrables

Incorrecto:

```text id="8lcp07"
ORM metadata fields
=
public filters
```

---

## 279.3 Projection wildcard

Incorrecto:

```text id="c0eokk"
fields=*
→ every column
```

---

## 279.4 Unlimited includes

Incorrecto:

```text id="l1blnt"
include=*.*.*.*
```

---

## 279.5 Unlimited IN

Incorrecto:

```text id="yzlrxb"
id IN 100000 values
```

sin budget.

---

## 279.6 Authorization after query

Incorrecto:

```text id="ggmk5i"
query
↓
paginate
↓
remove unauthorized rows
```

---

## 279.7 Sensitive field only hidden from projection

Incorrecto si todavía puede:

```text id="rx6uwf"
filter
sort
group
aggregate
```

por ese field.

---

## 279.8 Complexity only by number of filters

Insuficiente.

Debe considerar también:

```text id="vy8nnd"
depth
branching
relationships
aggregations
search
distribution
```

---

## 279.9 Silent truncation

Incorrecto:

```text id="p3wrcc"
requested limit=10000
silently use 100
```

si la API no define esa semántica explícitamente.

---

## 279.10 Unknown input ignored

Por default, incorrecto.

Puede ampliar accidentalmente el resultado.

---

# 280. Security model formal

Sea:

```text id="gzluql"
I = external query input
D = compiled query input definition
C = security context
B = resource budget
```

La resolución produce:

```text id="jwgps8"
Resolve(I, D, C, B)
→
Q
```

solo si todas las capacidades solicitadas están permitidas.

---

# 281. Capability condition

Para cada operación solicitada:

```text id="x0fi84"
(field, capability)
```

debe cumplirse:

```text id="i3gtc6"
Allowed(field, capability, C) = true
```

---

# 282. Budget condition

Debe cumplirse además:

```text id="nza51k"
Complexity(Q) ≤ EffectiveBudget(C,D)
```

---

# 283. Effective field capability

Podrá conceptualizarse como:

```text id="74dobq"
EffectiveCapability
=
DefinitionCapability
∩
EndpointPolicy
∩
AuthorizationPolicy
∩
SensitiveDataPolicy
```

---

# 284. Effective resource budget

```text id="y0mpx3"
EffectiveBudget
=
min(
  GlobalBudget,
  ProfileBudget,
  EndpointBudget,
  PrincipalBudget,
  WorkloadBudget
)
```

por dimensión donde ese modelo sea apropiado.

---

# 285. Final security condition

```text id="p2wegw"
ExecutableQueryInput(I)
⇔
StructureValid(I)
∧
FieldsResolved(I)
∧
OperatorsAllowed(I)
∧
RelationshipsAllowed(I)
∧
SensitiveAccessAllowed(I)
∧
AuthorizationAllows(I)
∧
ComplexityWithinBudget(I)
∧
ResourceLimitsSatisfied(I)
```

---

# 286. Resultado arquitectónico

La arquitectura completa queda:

```text id="sjnrrs"
External Request
      ↓
QueryInputDecoder
      ↓
QueryInputModel
      ↓
Validation
      ↓
Normalization
      ↓
CompiledQueryInputDefinition
      ↓
Field Resolution
      ↓
Operator Resolution
      ↓
Relationship Resolution
      ↓
Projection/Aggregation Resolution
      ↓
Authorization-Aware Capability Check
      ↓
Sensitive Field Policy
      ↓
Complexity Analyzer
      ↓
Resource Budget
      ↓
QueryInputSecurityDecision
      ↓
CanonicalQueryInput
      ↓
Query Builder
      ↓
Typed Query AST
      ↓
SQL Injection Prevention
      ↓
Semantic Query Engine
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

# 287. Regla maestra final

> **VoltStack tratará el Query Builder completo como una capacidad interna del framework, no como una API que pueda proyectarse directamente hacia consumidores externos. Los consumidores solo podrán solicitar un subconjunto explícitamente definido, autorizado y acotado de operaciones de consulta.**

Por tanto:

```text id="hyqdp5"
External Query Capability
<
Internal Query Capability
```

siendo la primera una vista controlada de la segunda.

---

# 288. Relación con los documentos 226 y 227

```text id="k3dzzu"
226 Security Architecture
        ↓
227 SQL Injection Prevention
        ↓
228 Query Input Security
```

La cadena queda:

```text id="kjvl44"
External Input
      ↓
What may it request?
      │
      └── Query Input Security
              ↓
How is it represented safely?
              │
              └── SQL Injection Prevention
                      ↓
How is it executed?
                      │
                      └── Query Engine / Compiler / Executor
```

---

# 289. Estado del Bloque 22

```text id="baqhrr"
BLOCK 22 — SECURITY

✓ 226_DATABASE_SECURITY_ARCHITECTURE.md
✓ 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
✓ 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
○ 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
○ 230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
○ 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
○ 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
○ 233_DATABASE_QUERY_AUDIT_SYSTEM.md
○ 234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 290. Siguiente documento

```text id="4sj160"
229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura completa para proteger las credenciales utilizadas por VoltStack Database, incluyendo:

```text id="13f8h2"
Credential Model
SecretReference
SecretValue
Secret Providers
Environment Secrets
Encrypted Configuration
Cloud Secret Managers
Vault Integrations
Credential Purpose
Runtime Credentials
Migration Credentials
Backup Credentials
Administrative Credentials
Credential Leasing
Lazy Secret Resolution
Secret Lifetime
In-Memory Protection
Credential Redaction
Exception Safety
Telemetry Safety
Debug Safety
Serialization Safety
Credential Rotation
Rotation Generations
Pool Drain
Expired Credentials
Dynamic Database Tokens
Credential Refresh
Failure Modes
Persistent Runtime Safety
Multitenancy
Per-Tenant Credentials
Audit
Testing
Secure Defaults
```

bajo una regla central:

> **Las credenciales de base de datos nunca deberán tratarse como configuración ordinaria: VoltStack deberá manejarlas como secretos de vida limitada, con propósito explícito, resolución controlada, redacción obligatoria y rotación compatible con conexiones persistentes.**