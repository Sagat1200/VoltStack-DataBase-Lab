# 200_DATABASE_CURSOR_PAGINATION_SYSTEM.md

# VoltStack Quantum Database
## Database Cursor Pagination System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 200 — Database Cursor Pagination System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `199_DATABASE_PAGINATION_SYSTEM.md`  
**Siguiente documento:** `201_DATABASE_CHUNK_PROCESSING_SYSTEM.md`

---

# 1. Propósito

`Database Cursor Pagination System` define la arquitectura mediante la cual VoltStack implementará paginación basada en **posición lógica dentro de un orden estable**, evitando depender de offsets crecientes.

Ejemplo:

```php id="a72h2m"
$page = User::query()
    ->where('active', true)
    ->orderBy('created_at', 'desc')
    ->orderBy('id', 'desc')
    ->cursorPaginate(
        perPage: 25,
        after: $cursor,
    );
```

Conceptualmente:

```text id="c1jm26"
ORDER BY created_at DESC, id DESC

Cursor:
    created_at = 2026-09-01T10:30:00Z
    id = 7812

Next query:
    WHERE
        created_at < cursor.created_at
        OR (
            created_at = cursor.created_at
            AND id < cursor.id
        )
```

La regla central será:

> **Un cursor de VoltStack representa una frontera verificable dentro de un orden lógico estable y completamente definido; nunca será simplemente un OFFSET codificado ni un identificador arbitrario sin contexto.**

Formalmente:

```text id="uh0bpu"
CursorPagination
=
StableOrdering
+
ContinuationBoundary
+
CursorState
+
QueryBinding
+
DomainBinding
+
CursorIntegrity
```

No:

```text id="xib8so"
CursorPagination
=
Base64(OFFSET)
```

---

# 2. Relación con Pagination System

`199_DATABASE_PAGINATION_SYSTEM.md` define la arquitectura general:

```text id="iy2fzw"
Pagination
├── navigation
├── page size
├── ordering
├── metadata
├── consistency
└── resource governance
```

Este documento especializa:

```text id="2zh5ls"
Pagination
    ↓
Cursor Pagination
```

No crea un segundo Pagination Engine.

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text id="k89tr4"
Cursor Pagination
≠
Offset Pagination
≠
Database Cursor
≠
Result Cursor
≠
Streaming Result
≠
Chunk Processing
≠
Lazy Collection
≠
Primary Key Pagination
```

---

# 4. Cursor Pagination vs Offset Pagination

Offset:

```text id="wbgx23"
page = 5000
perPage = 20

OFFSET = 99,980
```

Cursor:

```text id="9s75fs"
after:
    created_at = T
    id = 9182
```

La segunda consulta continúa desde una frontera lógica.

---

# 5. Cursor ≠ Database Cursor

Un database cursor puede representar estado interno de ejecución de una conexión.

VoltStack Cursor Pagination utiliza un token lógico portable:

```text id="fsb06r"
PaginationCursor
```

No almacena:

```text id="6rr3f1"
PDOStatement
Connection
server-side cursor handle
ResultCursor
```

---

# 6. Cursor ≠ Primary Key

Un cursor puede incluir:

```text id="11vpyk"
created_at
id
```

o:

```text id="zw8q7g"
score
published_at
id
```

Por tanto:

```text id="1ldaw3"
Cursor
≠
ID
```

aunque una query ordenada únicamente por PK pueda usarlo como componente.

---

# 7. Cursor ≠ Encoded Offset

Esto estará prohibido como implementación de keyset pagination:

```text id="nswxc7"
cursor = base64("offset=500000")
```

porque conserva el mismo problema de OFFSET.

---

# 8. Objetivos

El sistema deberá soportar:

1. keyset pagination;
2. cursores forward;
3. cursores backward;
4. compound ordering;
5. ASC;
6. DESC;
7. mixed directions;
8. deterministic tie-breakers;
9. nullable ordering;
10. typed cursor values;
11. cursor encoding;
12. signing;
13. encryption opcional;
14. query binding;
15. tenant binding;
16. shard binding;
17. topology/version binding;
18. expiration opcional;
19. schema/metadata compatibility;
20. ORM;
21. Query Builder;
22. projections;
23. replicas;
24. sharding;
25. cache;
26. persistent runtimes;
27. diagnostics;
28. extensibilidad.

---

# 9. Arquitectura general

```text id="pn64ol"
Developer API
     │
     ▼
CursorPaginationRequest
     │
     ▼
Cursor Decoder
     │
     ▼
CursorEnvelope
     │
     ▼
Cursor Validator
     │
     ├── Integrity
     ├── Version
     ├── Query Binding
     ├── Ordering Binding
     ├── Domain Binding
     └── Expiration
     │
     ▼
Cursor Pagination Planner
     │
     ▼
Continuation Predicate
     │
     ▼
Query Model / AST
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
Hydration
     │
     ▼
Cursor Page Assembler
     │
     ▼
CursorPageResult
```

---

# 10. Cursor model

El cursor interno deberá ser estructurado.

```php id="p2hj6o"
final readonly class PaginationCursor
{
    public function __construct(
        public CursorVersion $version,
        public CursorDirection $direction,
        public CursorOrdering $ordering,
        public CursorBoundary $boundary,
        public CursorBinding $binding,
    ) {}
}
```

---

# 11. CursorBoundary

Representará valores del último/primer elemento relevante.

Ejemplo:

```text id="8l0e92"
Boundary

created_at:
    Instant(2026-09-01T10:30:00Z)

id:
    Int(7812)
```

---

# 12. Tipos

Los valores no deberán degradarse todos a strings.

El cursor conservará información suficiente para reconstruir:

```text id="6fdc3n"
integer
decimal
string
enum
UUID
ULID
date/time
boolean
nullable value
custom supported scalar
```

---

# 13. Integración Type System

Deberá apoyarse en:

```text id="01nqvz"
155_DATABASE_TYPE_SYSTEM
156_DATABASE_TYPE_REGISTRY_SYSTEM
157_DATABASE_VALUE_CONVERSION_SYSTEM
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM
```

---

# 14. Cursor value ≠ raw DB value

Un cursor deberá usar una representación canónica portable.

No deberá depender directamente de:

```text id="d1du3f"
PDO return formatting
vendor-specific date string
driver-specific boolean
```

---

# 15. Orden lógico

Sea:

```text id="7rqdt2"
ORDER BY
    created_at DESC,
    id DESC
```

El cursor deberá estar vinculado a exactamente ese orden.

---

# 16. Orden completo

Cursor Pagination requiere un orden capaz de determinar una continuación no ambigua.

Ejemplo insuficiente:

```text id="b2e9q6"
ORDER BY created_at
```

si muchos registros comparten el mismo timestamp.

---

# 17. Tie-breaker

VoltStack podrá convertir:

```text id="24zrv0"
ORDER BY created_at DESC
```

en:

```text id="dp9rg2"
ORDER BY created_at DESC, id DESC
```

si metadata prueba que `id` proporciona unicidad apropiada.

---

# 18. Inferencia

La inferencia deberá ser:

```text id="6svr3n"
deterministic
metadata-driven
explainable
```

Nunca una heurística silenciosa basada únicamente en nombre `"id"`.

---

# 19. Composite identifiers

Una entidad con:

```text id="8iy0uf"
tenant_id
invoice_number
```

puede requerir ambos componentes como tie-breaker.

---

# 20. Stable ordering condition

Sea:

```text id="mqyszt"
K(r)
=
(k1(r), k2(r), ..., kn(r))
```

Para navegación determinista se busca:

```text id="hn9pja"
r1 ≠ r2
⇒
K(r1) ≠ K(r2)
```

dentro del dominio lógico de la query.

---

# 21. Unique DB constraint

Una unique constraint puede probar unicidad si:

```text id="ivv81k"
query semantics
+
NULL semantics
+
domain scope
```

la hacen aplicable.

---

# 22. Unique ≠ globally unique

Ejemplo:

```text id="88cgzl"
UNIQUE(tenant_id, invoice_number)
```

`invoice_number` solo no es tie-breaker global.

---

# 23. Forward pagination

```php id="75hs61"
$query->cursorPaginate(
    perPage: 25,
    after: $cursor,
);
```

significa continuar después de la frontera.

---

# 24. Backward pagination

```php id="kt68m9"
$query->cursorPaginate(
    perPage: 25,
    before: $cursor,
);
```

permite navegar en dirección contraria.

---

# 25. after y before

No deberán aceptarse simultáneamente salvo una futura operación explícita de rango.

---

# 26. CursorDirection

```php id="g0f4as"
enum CursorDirection
{
    case FORWARD;
    case BACKWARD;
}
```

---

# 27. Query order vs traversal direction

Muy importante:

```text id="9gnl6i"
Logical Order
≠
Traversal Direction
```

La query puede ordenar DESC y navegar backward.

---

# 28. Ejemplo forward DESC

Orden:

```text id="41bt5g"
created_at DESC
id DESC
```

Boundary:

```text id="ymum8q"
created_at = T
id = 100
```

Continuación:

```text id="xtgcbm"
created_at < T
OR
(created_at = T AND id < 100)
```

---

# 29. Ejemplo forward ASC

Orden:

```text id="yrf9gu"
created_at ASC
id ASC
```

Continuación:

```text id="c17r6c"
created_at > T
OR
(created_at = T AND id > 100)
```

---

# 30. Orden mixto

Debe soportarse:

```text id="ugob7v"
ORDER BY
    priority DESC,
    created_at ASC,
    id ASC
```

La continuación ya no puede generarse con una única comparación trivial.

---

# 31. Lexicographic predicate

Para:

```text id="ksn9d0"
K = (k1, k2, k3)
```

la continuación forward es conceptualmente:

```text id="9mgbza"
k1 after v1
OR
(
    k1 = v1
    AND k2 after v2
)
OR
(
    k1 = v1
    AND k2 = v2
    AND k3 after v3
)
```

donde `after` depende de ASC/DESC.

---

# 32. ContinuationPredicateBuilder

Propuesta:

```text id="s6c06n"
CursorContinuationPredicateBuilder
```

Responsabilidades:

```text id="c02h6n"
ordering
+
direction
+
boundary
+
NULL semantics
+
platform capabilities
→
semantic predicate
```

---

# 33. No SQL

El builder anterior producirá:

```text id="3c91jg"
Query Predicate AST
```

no SQL.

---

# 34. Tuple comparison

Algunas plataformas permiten expresiones tipo:

```sql id="p9fy2q"
(a, b) > (?, ?)
```

pero VoltStack no dependerá de ellas como semántica universal.

---

# 35. Optimización de plataforma

El Compiler/Optimizer podrá usar tuple comparison cuando:

```text id="34udly"
PlatformCapabilities
+
ordering semantics
+
NULL semantics
```

demuestren equivalencia.

---

# 36. NULL

Los valores NULL requieren tratamiento explícito.

---

# 37. Problema

```text id="bxpaxw"
ORDER BY score ASC
```

puede colocar NULL diferente según motor/configuración.

---

# 38. Null ordering

VoltStack deberá modelar:

```php id="gnsshu"
enum NullPlacement
{
    case FIRST;
    case LAST;
    case PLATFORM_DEFAULT;
}
```

---

# 39. Recomendación

Cursor pagination debería resolver `PLATFORM_DEFAULT` a una semántica efectiva antes de emitir cursor portable.

---

# 40. Cursor portable

Un cursor generado bajo:

```text id="9u47lo"
NULLS FIRST
```

no debe reinterpretarse posteriormente como:

```text id="6nvr3j"
NULLS LAST
```

---

# 41. Null boundary

El cursor deberá poder representar:

```text id="b6n6yy"
value = NULL
```

sin confundirlo con:

```text id="k4lilv"
cursor field missing
```

---

# 42. NULL ≠ missing

Regla:

```text id="8d79lp"
CursorNull
≠
AbsentCursorComponent
```

---

# 43. Collation

Orden de strings depende de:

```text id="p89tnv"
collation
case sensitivity
locale rules
database configuration
```

---

# 44. Cursor string values

No basta con comparar valores en PHP.

La continuación debe preservar semántica de orden DB.

---

# 45. Collation binding

Cuando sea necesario, el ordering fingerprint deberá reflejar la collation efectiva.

---

# 46. Expression ordering

Ejemplo:

```text id="ryajbd"
ORDER BY LOWER(email)
```

puede soportarse solo si el sistema sabe:

```text id="zhf2d1"
expression identity
result type
determinism
continuation semantics
```

---

# 47. Non-deterministic ordering

Esto deberá rechazarse por defecto:

```sql id="7g6vxp"
ORDER BY RANDOM()
```

---

# 48. Razón

No existe una frontera reproducible estable.

---

# 49. Time-dependent ordering

Ejemplo:

```text id="a09r2d"
ORDER BY distance_from(now())
```

puede cambiar entre requests.

Debe considerarse no estable salvo estrategia específica.

---

# 50. CursorEligibilityAnalyzer

VoltStack deberá analizar si una query es apta.

Resultado conceptual:

```php id="fgb9nu"
enum CursorEligibility
{
    case ELIGIBLE;
    case ELIGIBLE_WITH_INFERRED_TIE_BREAKER;
    case REQUIRES_EXPLICIT_ORDER;
    case UNSTABLE_ORDER;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 51. UNKNOWN ≠ eligible

Una política estricta deberá rechazar:

```text id="u2ln5o"
UNKNOWN
```

---

# 52. Cursor envelope

El token externo no deberá ser simplemente el boundary.

Propuesta:

```text id="08d9qn"
CursorEnvelope
├── version
├── direction
├── ordering fingerprint
├── query fingerprint
├── boundary
├── persistence domain
├── tenant binding?
├── shard binding?
├── metadata generation?
├── topology generation?
├── issuedAt?
├── expiresAt?
└── integrity
```

---

# 53. Query fingerprint

Evita reutilizar un cursor de:

```text id="jzntzx"
status = active
```

en:

```text id="xx0lbk"
status = deleted
```

---

# 54. Query fingerprint ≠ SQL hash

No deberá depender necesariamente del SQL compilado.

Preferir:

```text id="6y79ej"
normalized semantic query identity
```

---

# 55. Razón

El mismo query lógico puede compilar distinto entre:

```text id="uhd7zm"
MySQL
PostgreSQL
compiler versions
```

sin cambiar el contrato del cursor.

---

# 56. CursorQueryBinding

Podrá incluir:

```text id="fx5n1q"
query semantic fingerprint
parameter fingerprint
result root
ordering fingerprint
security scope fingerprint
```

según policy.

---

# 57. Parámetros

Un cursor generado para:

```text id="1lrn48"
tenant=5
status=active
```

no debe reutilizarse en:

```text id="thdmqa"
tenant=5
status=blocked
```

---

# 58. Sensitive values

No deberán incluirse en claro innecesariamente.

Preferir fingerprints canónicos.

---

# 59. Cursor domain binding

El cursor deberá estar ligado al persistence domain relevante.

```text id="w5jz0s"
Database
Tenant
Shard
Partition/Topology Generation
```

cuando aplique.

---

# 60. Tenant binding

Un cursor del Tenant A nunca deberá navegar datos del Tenant B.

---

# 61. Shard binding

Para single-shard query:

```text id="7nzp2j"
cursor shard = A
```

no deberá utilizarse silenciosamente en shard B.

---

# 62. Distributed cursor

En multi-shard queries el cursor puede requerir estado más complejo.

Ejemplo:

```text id="b7weao"
GlobalCursor
├── global boundary
├── shard map generation
└── optional shard continuation state
```

---

# 63. Cursor size

No deberá crecer ilimitadamente con el número de shards.

---

# 64. Stateless vs stateful cursor

VoltStack podrá soportar:

```text id="um9xpr"
STATELESS
STATEFUL_REFERENCE
```

---

# 65. Stateless

Todo estado necesario está codificado/autenticado en el token.

Ventajas:

```text id="tpm41n"
no server session
easy horizontal scaling
```

---

# 66. Stateful reference

El token contiene un identificador hacia estado almacenado.

Útil si el estado distribuido es demasiado grande.

---

# 67. Stateful cursor tradeoff

Introduce:

```text id="jxetme"
storage
expiration
cleanup
availability
distributed state
```

por lo que no será default para casos simples.

---

# 68. Cursor codec

Contrato:

```php id="ymctmx"
interface CursorCodec
{
    public function encode(
        PaginationCursor $cursor
    ): string;

    public function decode(
        string $token
    ): PaginationCursor;
}
```

---

# 69. Encoding ≠ security

Base64:

```text id="vqp38a"
Encoding
≠
Integrity
≠
Confidentiality
```

---

# 70. Cursor signing

Tokens externos deberán poder autenticarse.

Ejemplo conceptual:

```text id="l1cw3f"
payload
+
HMAC signature
```

---

# 71. Tampering

Modificar:

```text id="l3v0nr"
id = 100
```

a:

```text id="whn7a2"
id = 1
```

debe detectarse cuando el cursor esté firmado.

---

# 72. Default recomendado

Cursores externos generados por framework:

```text id="gtxqsi"
SIGNED
```

por defecto.

---

# 73. Encryption

Puede añadirse si el cursor contiene información que no debe ser visible.

Pero:

```text id="ss0wo9"
Encryption
≠
Integrity
```

a menos que se use autenticación integrada apropiada.

---

# 74. Signed ≠ encrypted

Un cursor firmado puede seguir siendo legible.

---

# 75. Key rotation

El codec deberá poder manejar:

```text id="fyckdu"
current signing key
previous verification keys
key ID
```

---

# 76. Cursor secret

Nunca deberá reutilizarse indiscriminadamente como:

```text id="0ax6ij"
database password
application encryption key
JWT key
```

sin una política de key management explícita.

---

# 77. Cursor versioning

El token deberá contener versión.

```text id="jcx5sq"
v1
v2
...
```

---

# 78. Version ≠ metadata generation

Separar:

```text id="bj8p5a"
CursorFormatVersion
MetadataGeneration
SchemaGeneration
TopologyGeneration
```

---

# 79. Compatibility

Un decoder podrá soportar varias versiones durante una ventana de migración.

---

# 80. Unknown cursor version

Debe producir error tipado.

No interpretar best-effort.

---

# 81. Cursor expiration

Podrá configurarse:

```text id="mep2v8"
NONE
ABSOLUTE_TTL
CUSTOM
```

---

# 82. Expiration ≠ staleness

Un cursor puede no estar expirado y aun corresponder a un dataset modificado.

---

# 83. Stale cursor

Cursor pagination no implica snapshot persistente.

Si se insertan/eliminan/modifican filas, el cursor sigue siendo una frontera de orden, no una fotografía completa del dataset.

---

# 84. Mutaciones posteriores

Supongamos:

```text id="6amvbe"
Page 1:
100
90
80
```

cursor boundary:

```text id="cnyblc"
80
```

Luego se inserta:

```text id="xxvdxk"
95
```

La siguiente página desde `80` no mostrará 95.

Eso es coherente con navegación forward desde la frontera.

---

# 85. Cursor ≠ snapshot token

Nunca afirmar:

```text id="3s4rvh"
same cursor
=
same database snapshot
```

---

# 86. Snapshot cursor

Una futura policy puede ligar cursor a:

```text id="upcb1l"
temporal snapshot
transaction snapshot
versioned dataset
```

si la infraestructura lo soporta.

Pero será una capacidad adicional.

---

# 87. Mutable ordering columns

Problema:

```text id="b71b7a"
ORDER BY score DESC
```

si `score` cambia entre páginas.

Un registro puede:

```text id="5ebx9g"
move before boundary
move after boundary
```

---

# 88. Resultado

Cursor pagination reduce problemas de OFFSET, pero no elimina anomalías causadas por mutar las claves de orden.

---

# 89. OrderingMutabilityPolicy

Podrá existir:

```text id="ff1vkt"
ALLOW
WARN
REQUIRE_IMMUTABLE
CUSTOM
```

---

# 90. Metadata

ORM/schema metadata puede ayudar a identificar:

```text id="l6e44m"
primary key
generated ID
immutable field
version field
```

pero no siempre puede probar mutabilidad de negocio.

---

# 91. Page result

Propuesta:

```php id="0r3cvi"
/**
 * @template T
 */
final readonly class CursorPageResult
{
    public function __construct(
        public array $items,
        public int $perPage,
        public ?string $nextCursor,
        public ?string $previousCursor,
        public bool $hasNextPage,
        public bool $hasPreviousPage,
        public CursorPaginationMetadata $metadata,
    ) {}
}
```

---

# 92. Total count

Cursor pagination no requiere total.

Default recomendado:

```text id="2x8x9u"
CountStrategy = NONE
```

---

# 93. Cursor + total

Puede permitirse:

```php id="5nhc3m"
$query->cursorPaginate(
    perPage: 25,
    count: PaginationCountStrategy::EXACT,
);
```

pero el count será una operación separada.

---

# 94. Total no forma parte de cursor semantics

```text id="dx7w18"
CursorNavigation
≠
TotalCount
```

---

# 95. Lookahead

Igual que Pagination general:

```text id="9iqr0k"
fetch perPage + 1
```

permite detectar continuidad.

---

# 96. Cursor generation

Después de hidratar la página, se necesitan los ordering values de los boundary items.

---

# 97. Hidden ordering projection

Si el usuario proyecta:

```php id="ewkxgx"
$query
    ->select('name')
    ->orderBy('created_at')
    ->cursorPaginate();
```

el cursor aún necesita `created_at`.

---

# 98. Internal projection

El planner podrá añadir internamente ordering values a la physical projection.

---

# 99. Hidden values

Esos valores:

```text id="z5bntn"
created_at
id
```

no deberán aparecer accidentalmente en el result shape público.

---

# 100. Logical projection ≠ physical projection

Regla:

```text id="dz6qcw"
LogicalProjection
≠
PhysicalExecutionProjection
```

cuando el planner requiere columnas auxiliares.

---

# 101. Alias collisions

Las columnas auxiliares deberán utilizar aliases internos seguros y no colisionar con aliases del usuario.

---

# 102. Entity results

Para Entity Query, los ordering values pueden obtenerse de:

```text id="98vcc3"
hydrated entity state
execution row metadata
```

según plan.

---

# 103. Partial entity

No deberá cargarse una entidad parcial insegura solo para construir cursor.

Preferir metadata de ejecución auxiliar.

---

# 104. Cursor extraction

Propuesta:

```text id="o2jic1"
CursorBoundaryExtractor
```

que recibe:

```text id="3i8kcc"
logical item
+
execution pagination metadata
+
ordering plan
```

---

# 105. Previous cursor

Para navegación backward se requiere especial cuidado.

---

# 106. Reverse execution

Una estrategia común:

```text id="qbn7dz"
logical order:
ASC

before cursor:
execute DESC
limit N+1
then reverse result in memory
```

siempre que preserve semántica.

---

# 107. Reverse plan

Debe ser explícito:

```text id="uy2t5s"
CursorTraversalPlan
```

---

# 108. Result order

Aunque physical execution se invierta, el `CursorPageResult` deberá respetar el logical order solicitado.

---

# 109. Previous cursor availability

No siempre es necesario ejecutar una consulta adicional para saber que existe una página previa.

Puede inferirse mediante lookahead/reverse traversal según plan.

---

# 110. First page

Sin cursor:

```text id="s4dxev"
after = null
before = null
```

representa el inicio lógico en dirección default.

---

# 111. Empty page

Si no hay elementos:

```text id="gv2vzk"
items = []
```

no se inventará un cursor boundary.

---

# 112. Cursor replay

Reutilizar el mismo cursor y query bajo estado DB compatible debería producir la misma continuación lógica.

No necesariamente los mismos rows si el dataset cambió.

---

# 113. Cursor idempotence ≠ result immutability

```text id="twg85k"
Same Cursor
≠
Same Result Forever
```

---

# 114. Query binding validation

Antes de ejecutar:

```text id="7bfwbm"
cursor.queryFingerprint
```

debe compararse con el query actual.

---

# 115. Ordering validation

También:

```text id="r92nzk"
cursor.orderingFingerprint
```

debe coincidir.

---

# 116. Direction validation

Un cursor puede ser usable para forward/backward según cómo se diseñe el envelope.

La policy debe ser explícita.

---

# 117. Bidirectional cursor

VoltStack puede almacenar suficiente boundary state para reutilizar una frontera en ambas direcciones.

Pero el token debe indicar qué usos son válidos.

---

# 118. Cursor binding failure

Nunca:

```text id="4k4d45"
ignore mismatch
and continue
```

Default:

```text id="v2hkh8"
reject cursor
```

---

# 119. Metadata generation

Si cambia ORM mapping de un ordering field, el cursor puede volverse incompatible.

---

# 120. Schema generation

No todo cambio de schema invalida todos los cursores.

La invalidación deberá ser dependency-aware cuando sea viable.

---

# 121. Conservative policy

Si no puede determinarse compatibilidad:

```text id="uk8qyn"
UNKNOWN
```

puede rechazarse bajo STRICT.

---

# 122. Cache integration

Una cursor page puede almacenarse en Result Cache.

La identidad deberá incluir:

```text id="pc0c4g"
query fingerprint
cursor boundary/token identity
page size
direction
ordering
domain
tenant/shard
consistency profile
```

---

# 123. Raw signed token como cache key

No es ideal.

Preferir:

```text id="a31u5e"
canonical cursor fingerprint
```

para evitar incorporar firmas/secret material.

---

# 124. Cache invalidation

Se aplican:

```text id="4xfqjn"
191_DATABASE_CACHE_INVALIDATION_SYSTEM
192_DATABASE_CACHE_CONSISTENCY_SYSTEM
```

---

# 125. Cursor cache ≠ cursor state

Cachear la página no convierte el cursor en estado de sesión.

---

# 126. Replica integration

Cursor query puede dirigirse a replica si la consistency policy lo permite.

---

# 127. Replica drift

Page 1 en Replica A y Page 2 en Replica B pueden observar diferentes replication positions.

---

# 128. Cursor provenance

Opcionalmente el cursor podrá registrar:

```text id="y7ezpe"
minimum observed data version
replication position
authority epoch
```

cuando la consistency policy lo requiera.

---

# 129. READ_YOUR_WRITES

Si una operación previa estableció:

```text id="dxph0m"
MinimumRequiredVersion
```

la continuación no deberá usar una replica que no satisfaga esa versión.

---

# 130. Sticky connection

Se integra con:

```text id="i42wwc"
180_DATABASE_STICKY_CONNECTION_SYSTEM
```

---

# 131. Failover

Tras failover:

```text id="f2zkuf"
AuthorityEpoch
```

puede cambiar.

Cursores ligados a una autoridad incompatible podrán ser rechazados.

---

# 132. Cursor ≠ failover guarantee

Un cursor válido criptográficamente no prueba que la nueva autoridad tenga el mismo estado.

---

# 133. Sharding

Single shard:

```text id="vyvgkl"
Query
→ Shard A
→ Cursor Boundary
```

es relativamente directo.

---

# 134. Multi-shard

Para orden global:

```text id="t79hpr"
Shard A ─┐
Shard B ─┼─► Merge
Shard C ─┘
```

se requiere una frontera global.

---

# 135. Global boundary

Si todos los shards utilizan el mismo orden global:

```text id="t6ct6v"
(created_at, global_id)
```

puede utilizarse como continuation boundary en todos.

---

# 136. Global tie-breaker

El tie-breaker deberá ser globalmente único dentro del conjunto distribuido.

Un `id=10` local por shard no basta.

---

# 137. Composite distributed key

Podría utilizarse:

```text id="ik86je"
(created_at, shard_id, local_id)
```

si esa semántica es válida y estable.

---

# 138. Exposición de shard ID

El token puede contenerlo internamente sin exponerlo semánticamente a la aplicación.

Si el cursor no está cifrado, debe evaluarse si esa información es sensible.

---

# 139. Resharding

Si cambia:

```text id="08ay8g"
ShardMapGeneration
```

un cursor puede seguir siendo válido si su boundary es global e independiente del mapa.

O puede requerir invalidación.

---

# 140. No asumir

```text id="evxg8e"
Resharding
⇒
always invalidate
```

ni:

```text id="q9pd9m"
Resharding
⇒
always valid
```

La policy depende del cursor model.

---

# 141. Distributed cursor planner

Propuesta:

```text id="mkebmx"
DistributedCursorPaginationPlanner
```

analizará:

```text id="6jig4q"
global ordering
shard ownership
tie-breaker uniqueness
topology generation
merge strategy
continuation strategy
```

---

# 142. Merge

Cada shard podrá recibir:

```text id="u0rj1x"
WHERE key after global boundary
ORDER BY key
LIMIT N+1
```

y el coordinator hará k-way merge.

---

# 143. Over-fetch

No necesariamente todos los shards necesitan devolver `N+1` completos si el planner puede optimizar adaptativamente.

---

# 144. Shard failure

Si un shard requerido falla:

```text id="foc0vf"
GlobalPageCompleteness = UNKNOWN/PARTIAL
```

No deberá devolverse como completa sin policy explícita.

---

# 145. Security

Cursores externos son input no confiable.

---

# 146. Decoder hardening

Debe limitar:

```text id="0b80w3"
token length
payload size
number of components
nesting depth
string lengths
version range
```

antes de procesar profundamente.

---

# 147. Signature verification

Debe realizarse antes de confiar en:

```text id="5pj8ku"
query binding
tenant
boundary
types
```

---

# 148. Type injection

Un token no deberá poder solicitar:

```text id="ntmk3e"
instantiate arbitrary PHP class
```

---

# 149. Stable type IDs

Utilizar:

```text id="4yow1s"
TypeId
```

del Type Registry.

No FQCN arbitrario proveniente del token.

---

# 150. Object deserialization

Nunca usar `unserialize()` de payload externo para reconstruir objetos arbitrarios.

---

# 151. Safe codec

Preferir:

```text id="1q4n3u"
versioned typed scalar envelope
```

con validación estricta.

---

# 152. SQL injection

Cursor values entrarán al Query Model como parámetros tipados.

Nunca concatenación SQL.

---

# 153. Authorization

El cursor no concede acceso.

Siempre:

```text id="r8d8b6"
Authorization
+
Current Query Scope
```

se evalúan normalmente.

---

# 154. Signed cursor ≠ authorization token

Regla crítica:

```text id="q9z4dt"
ValidSignature
≠
AuthorizedAccess
```

---

# 155. Tenant security

Incluso con cursor firmado:

```text id="b7qxuw"
CurrentTenant
```

debe coincidir con el dominio permitido.

---

# 156. Cursor confidentiality

Si el boundary revela:

```text id="25e4fp"
internal IDs
timestamps
business sequence
```

la aplicación podrá habilitar encryption.

---

# 157. Denial of service

Cursores malformados no deberán causar:

```text id="5d90me"
unbounded allocations
huge ASTs
expensive type resolution
deep recursive decode
```

---

# 158. Resource governance

`CursorPaginationPolicy` podrá definir:

```php id="t2y8te"
final readonly class CursorPaginationPolicy
{
    public function __construct(
        public int $defaultPageSize,
        public int $maxPageSize,
        public int $maxCursorBytes,
        public bool $requireSignedCursor,
        public CursorExpirationPolicy $expiration,
        public CursorOrderingPolicy $ordering,
    ) {}
}
```

---

# 159. HTTP independence

Database no leerá:

```text id="i5j6z4"
?cursor=
```

directamente.

HTTP integration convertirá input en:

```text id="j8xovq"
CursorPaginationRequest
```

---

# 160. URL generation

`next URL` / `previous URL` pertenecen a HTTP/UI.

Database devuelve tokens.

---

# 161. SPA integration

El frontend podrá recibir:

```json id="nq03pv"
{
  "data": [],
  "meta": {
    "hasNext": true,
    "hasPrevious": true
  },
  "cursor": {
    "next": "...",
    "previous": "..."
  }
}
```

sin que Database conozca JSON/HTTP.

---

# 162. Persistent runtime

FrankenPHP:

```text id="7kvy30"
Request A
cursor A

Request B
cursor B
```

no compartirán mutable cursor state.

---

# 163. Stateless codec

Puede compartirse entre workers si:

```text id="4gbn2z"
immutable/stateless
```

---

# 164. Signing key provider

Puede ser infraestructura compartida segura.

Pero current cursor/request state siempre será scoped.

---

# 165. RoadRunner/OpenSwoole

Misma regla:

```text id="4hmd2c"
WorkerReuse
≠
CursorStateReuse
```

---

# 166. Coroutine safety

Decoder/Planner deberán ser stateless o context-driven.

---

# 167. Cursor planner

Responsabilidades:

```text id="4xqf5j"
validate ordering
resolve tie-breakers
validate cursor
bind query
derive continuation predicate
resolve traversal direction
resolve physical ordering
resolve hidden projections
resolve lookahead
produce plan
```

---

# 168. Planner no ejecuta

Siempre:

```text id="jx4s91"
CursorPaginationPlanner
≠
QueryExecutor
```

---

# 169. CursorPaginationPlan

Propuesta:

```php id="a7av4r"
final readonly class CursorPaginationPlan
{
    public function __construct(
        public QueryModel $query,
        public CursorOrderingPlan $ordering,
        public CursorTraversalPlan $traversal,
        public ?CursorBoundary $boundary,
        public PaginationLookahead $lookahead,
        public CursorProjectionPlan $projection,
    ) {}
}
```

---

# 170. Cursor assembler

Responsabilidades:

```text id="8w2n73"
trim lookahead
restore logical ordering
extract first boundary
extract last boundary
build previous cursor
build next cursor
assemble metadata
```

---

# 171. Assembler no genera SQL

Ni ejecuta query adicional por sí mismo.

---

# 172. Error hierarchy

```text id="3jgl9d"
PaginationException
└── CursorPaginationException
    ├── InvalidCursorException
    │   ├── MalformedCursorException
    │   ├── UnsupportedCursorVersionException
    │   ├── CursorIntegrityException
    │   ├── CursorExpiredException
    │   └── CursorTypeException
    ├── CursorBindingException
    │   ├── CursorQueryMismatchException
    │   ├── CursorOrderingMismatchException
    │   ├── CursorTenantMismatchException
    │   ├── CursorShardMismatchException
    │   └── CursorTopologyMismatchException
    ├── CursorOrderingException
    │   ├── MissingCursorOrderException
    │   ├── UnstableCursorOrderException
    │   └── UnsupportedCursorExpressionException
    ├── CursorPlanningException
    ├── CursorSecurityException
    ├── CursorDistributionException
    └── CursorResourceException
```

---

# 173. Directory structure

```text id="fr7x0c"
src/Quantum/Database/Pagination/Cursor/
│
├── PaginationCursor.php
├── CursorVersion.php
├── CursorDirection.php
├── CursorBoundary.php
├── CursorBinding.php
├── CursorEnvelope.php
│
├── Request/
│   └── CursorPaginationRequest.php
│
├── Result/
│   ├── CursorPageResult.php
│   └── CursorPaginationMetadata.php
│
├── Ordering/
│   ├── CursorOrdering.php
│   ├── CursorOrderingPolicy.php
│   ├── CursorOrderingResolver.php
│   ├── CursorOrderingPlan.php
│   ├── CursorEligibility.php
│   ├── CursorEligibilityAnalyzer.php
│   └── NullPlacement.php
│
├── Boundary/
│   ├── CursorBoundaryExtractor.php
│   ├── CursorContinuationPredicateBuilder.php
│   └── CursorBoundaryValidator.php
│
├── Codec/
│   ├── CursorCodec.php
│   ├── CursorEncoder.php
│   ├── CursorDecoder.php
│   ├── CursorSigner.php
│   ├── CursorCipher.php
│   └── CursorKeyProvider.php
│
├── Binding/
│   ├── CursorQueryBinding.php
│   ├── CursorDomainBinding.php
│   └── CursorBindingValidator.php
│
├── Planning/
│   ├── CursorPaginationPlanner.php
│   ├── CursorPaginationPlan.php
│   ├── CursorTraversalPlan.php
│   └── CursorProjectionPlan.php
│
├── Distribution/
│   ├── DistributedCursorPaginationPlanner.php
│   ├── DistributedCursorPaginationPlan.php
│   └── DistributedCursorMergeCoordinator.php
│
├── Assembly/
│   └── CursorPaginationAssembler.php
│
├── Security/
│   └── CursorSecurityPolicy.php
│
├── Diagnostics/
│   ├── CursorInspector.php
│   └── CursorExplainer.php
│
├── Telemetry/
│   └── CursorPaginationTelemetry.php
│
└── Exception/
    └── ...
```

---

# 174. Telemetry

Eventos conceptuales:

```text id="rv6d1p"
CursorDecodeStarted
CursorDecoded
CursorValidationFailed
CursorPaginationPlanCreated
CursorContinuationApplied
CursorPageExecuted
CursorGenerated
CursorPaginationCompleted
CursorPaginationFailed
```

---

# 175. Métricas

```text id="a3dug0"
db.pagination.cursor.requests
db.pagination.cursor.duration
db.pagination.cursor.invalid
db.pagination.cursor.expired
db.pagination.cursor.binding_mismatch
db.pagination.cursor.distributed
db.pagination.cursor.page_size
```

---

# 176. Cardinalidad

No utilizar como labels:

```text id="qgxk41"
raw cursor
boundary values
user ID
tenant ID
query fingerprint completo
```

---

# 177. Diagnostics

API conceptual:

```php id="d7jfs2"
DB::pagination()
    ->cursor()
    ->explain(
        query: User::query()
            ->where('active', true)
            ->orderBy('created_at', 'desc'),
        cursor: $cursor,
    );
```

---

# 178. Explain output

```text id="wq6q7s"
CURSOR PAGINATION PLAN

Strategy:
    KEYSET

Direction:
    FORWARD

Logical Ordering:
    created_at DESC
    id DESC [inferred tie-breaker]

Cursor:
    version: 1
    signature: VALID
    query binding: VALID
    domain binding: VALID

Boundary:
    created_at: 2026-09-01T10:30:00Z
    id: 7812

Continuation:
    created_at < :cursor_1
    OR (
        created_at = :cursor_1
        AND id < :cursor_2
    )

Page Size:
    25

Physical Fetch:
    26

Total Count:
    NOT REQUESTED

Distribution:
    SINGLE SHARD

Eligibility:
    ELIGIBLE_WITH_INFERRED_TIE_BREAKER
```

---

# 179. Testing matrix

| Área | Caso |
|---|---|
| Basic | first page |
| Basic | next page |
| Basic | previous page |
| Ordering | ASC |
| Ordering | DESC |
| Ordering | mixed |
| Ordering | inferred tie-breaker |
| Ordering | composite key |
| Ordering | unstable |
| NULL | NULLS FIRST |
| NULL | NULLS LAST |
| Types | integer |
| Types | UUID |
| Types | datetime |
| Types | custom type |
| Codec | encode/decode |
| Security | signed valid |
| Security | modified token |
| Security | wrong key |
| Version | old supported |
| Version | unknown |
| Binding | query mismatch |
| Binding | ordering mismatch |
| Binding | tenant mismatch |
| Binding | shard mismatch |
| Projection | hidden sort field |
| ORM | entity |
| ORM | projection |
| Cache | cursor page |
| Replica | lag |
| RYW | sticky |
| Shard | single |
| Shard | distributed |
| Runtime | worker isolation |
| Resource | oversized cursor |
| Resource | excessive page size |

---

# 180. Architectural invariants

## DB-CURSOR-001
Cursor Pagination será una especialización de Pagination.

## DB-CURSOR-002
Cursor Pagination será distinta de Offset Pagination.

## DB-CURSOR-003
Pagination Cursor será distinto de Database Cursor.

## DB-CURSOR-004
Pagination Cursor será distinto de Result Cursor.

## DB-CURSOR-005
Cursor no será necesariamente Primary Key.

## DB-CURSOR-006
Cursor no será OFFSET codificado.

## DB-CURSOR-007
Cursor representará una frontera lógica.

## DB-CURSOR-008
Cursor requerirá ordering definido.

## DB-CURSOR-009
Ordering deberá ser suficientemente determinista.

## DB-CURSOR-010
Tie-breakers serán explícitos o metadata-driven.

## DB-CURSOR-011
Tie-breakers inferidos serán explainable.

## DB-CURSOR-012
Nombre `id` no probará unicidad por sí solo.

## DB-CURSOR-013
Composite identifiers serán soportables.

## DB-CURSOR-014
Tenant-scoped uniqueness no será global uniqueness.

## DB-CURSOR-015
Forward pagination será soportada.

## DB-CURSOR-016
Backward pagination será soportada.

## DB-CURSOR-017
after y before no coexistirán por defecto.

## DB-CURSOR-018
Traversal direction será distinta de logical ordering.

## DB-CURSOR-019
ASC será soportado.

## DB-CURSOR-020
DESC será soportado.

## DB-CURSOR-021
Mixed ordering será soportable.

## DB-CURSOR-022
Continuation será lexicográfica.

## DB-CURSOR-023
Continuation builder producirá Predicate AST.

## DB-CURSOR-024
Continuation builder no generará SQL.

## DB-CURSOR-025
Tuple comparison será una optimización de plataforma.

## DB-CURSOR-026
Tuple comparison solo se usará si preserva semántica.

## DB-CURSOR-027
NULL será modelado explícitamente.

## DB-CURSOR-028
Cursor NULL será distinto de missing.

## DB-CURSOR-029
Null ordering será explícito.

## DB-CURSOR-030
Platform default null ordering deberá resolverse cuando la portabilidad lo requiera.

## DB-CURSOR-031
String ordering respetará collation DB.

## DB-CURSOR-032
PHP comparison no sustituirá DB collation.

## DB-CURSOR-033
Ordering expressions requerirán identidad semántica.

## DB-CURSOR-034
Non-deterministic ordering será rechazado por defecto.

## DB-CURSOR-035
Time-dependent ordering no se considerará estable automáticamente.

## DB-CURSOR-036
UNKNOWN eligibility no será ELIGIBLE.

## DB-CURSOR-037
Cursor values serán tipados.

## DB-CURSOR-038
Cursor values usarán representación canónica.

## DB-CURSOR-039
Cursor no dependerá de formatting PDO.

## DB-CURSOR-040
Cursor Envelope será versionado.

## DB-CURSOR-041
Cursor se vinculará al query cuando policy lo requiera.

## DB-CURSOR-042
Query fingerprint no será necesariamente SQL hash.

## DB-CURSOR-043
Query parameters relevantes participarán en binding.

## DB-CURSOR-044
Sensitive query values no se expondrán innecesariamente.

## DB-CURSOR-045
Cursor se vinculará al persistence domain.

## DB-CURSOR-046
Tenant binding será preservado.

## DB-CURSOR-047
Cursor Tenant A no será válido para Tenant B.

## DB-CURSOR-048
Shard binding será preservado cuando aplique.

## DB-CURSOR-049
Distributed cursors tendrán semántica explícita.

## DB-CURSOR-050
Distributed cursor state será bounded.

## DB-CURSOR-051
Stateless cursors serán soportados.

## DB-CURSOR-052
Stateful-reference cursors serán opcionales.

## DB-CURSOR-053
Encoding no será considerado seguridad.

## DB-CURSOR-054
Signing proporcionará integridad/autenticidad.

## DB-CURSOR-055
Signed no significará encrypted.

## DB-CURSOR-056
Encryption será opcional.

## DB-CURSOR-057
Cursor signing keys podrán rotarse.

## DB-CURSOR-058
Cursor format version será distinta de metadata generation.

## DB-CURSOR-059
Unknown cursor version será rechazada.

## DB-CURSOR-060
Cursor expiration será configurable.

## DB-CURSOR-061
Expiration será distinta de staleness.

## DB-CURSOR-062
Cursor no será snapshot token.

## DB-CURSOR-063
Same cursor no garantizará same result forever.

## DB-CURSOR-064
Dataset mutation podrá cambiar páginas futuras.

## DB-CURSOR-065
Mutable ordering keys podrán producir movement anomalies.

## DB-CURSOR-066
Ordering mutability policy será configurable.

## DB-CURSOR-067
CursorPageResult será tipado.

## DB-CURSOR-068
Cursor pagination no requerirá total.

## DB-CURSOR-069
Total count será opcional.

## DB-CURSOR-070
Lookahead podrá determinar hasNext.

## DB-CURSOR-071
Logical projection podrá diferir de physical projection.

## DB-CURSOR-072
Ordering fields auxiliares podrán añadirse internamente.

## DB-CURSOR-073
Hidden ordering fields no se expondrán al usuario.

## DB-CURSOR-074
Internal aliases no colisionarán con aliases de usuario.

## DB-CURSOR-075
Partial entities no se crearán solo para extraer cursor.

## DB-CURSOR-076
CursorBoundaryExtractor será independiente del SQL Compiler.

## DB-CURSOR-077
Backward physical execution podrá invertir ordering.

## DB-CURSOR-078
Result final preservará logical ordering.

## DB-CURSOR-079
Empty page no inventará boundary.

## DB-CURSOR-080
Cursor replay no implicará immutable DB.

## DB-CURSOR-081
Query binding se validará antes de ejecución.

## DB-CURSOR-082
Ordering binding se validará antes de ejecución.

## DB-CURSOR-083
Binding mismatch no será ignorado.

## DB-CURSOR-084
Metadata incompatibility podrá invalidar cursor.

## DB-CURSOR-085
Schema changes no invalidarán todos los cursores indiscriminadamente si existe evidencia precisa.

## DB-CURSOR-086
UNKNOWN compatibility podrá rechazarse bajo STRICT.

## DB-CURSOR-087
Cursor pages podrán integrarse con Result Cache.

## DB-CURSOR-088
Cache key usará canonical cursor identity.

## DB-CURSOR-089
Raw signed token no será obligatorio como cache key.

## DB-CURSOR-090
Cache no convertirá cursor en session state.

## DB-CURSOR-091
Replica routing respetará consistency policy.

## DB-CURSOR-092
Replica drift será representable.

## DB-CURSOR-093
READ_YOUR_WRITES será respetado.

## DB-CURSOR-094
Sticky routing será respetado.

## DB-CURSOR-095
Failover authority epoch podrá participar en validation.

## DB-CURSOR-096
Valid signature no probará database freshness.

## DB-CURSOR-097
Single-shard cursor pagination será soportada.

## DB-CURSOR-098
Multi-shard pagination requerirá global ordering.

## DB-CURSOR-099
Distributed tie-breaker deberá ser globalmente suficiente.

## DB-CURSOR-100
Local ID no será asumido globalmente único.

## DB-CURSOR-101
Shard ID podrá formar parte de composite boundary.

## DB-CURSOR-102
Resharding no implicará automáticamente cursor válido o inválido.

## DB-CURSOR-103
Shard map generation podrá participar en binding.

## DB-CURSOR-104
Distributed continuation podrá ejecutarse por shard.

## DB-CURSOR-105
Distributed merge preservará global ordering.

## DB-CURSOR-106
Shard failure no producirá página completa falsa.

## DB-CURSOR-107
External cursors serán input no confiable.

## DB-CURSOR-108
Token size será limitado.

## DB-CURSOR-109
Payload depth será limitado.

## DB-CURSOR-110
Signature será verificada antes de confiar en payload.

## DB-CURSOR-111
Cursor no podrá solicitar arbitrary PHP classes.

## DB-CURSOR-112
Type IDs serán estables.

## DB-CURSOR-113
No se usará unsafe unserialize para cursores externos.

## DB-CURSOR-114
Cursor values serán query parameters tipados.

## DB-CURSOR-115
Cursor values no se concatenarán a SQL.

## DB-CURSOR-116
Signed cursor no será authorization token.

## DB-CURSOR-117
Authorization se evaluará normalmente.

## DB-CURSOR-118
Tenant authorization no será sustituida por cursor binding.

## DB-CURSOR-119
Confidential cursor fields podrán cifrarse.

## DB-CURSOR-120
Malformed cursors no causarán allocations ilimitadas.

## DB-CURSOR-121
Page size tendrá límite.

## DB-CURSOR-122
Cursor byte size tendrá límite.

## DB-CURSOR-123
Database Cursor Pagination no dependerá de HTTP.

## DB-CURSOR-124
Database no leerá query string directamente.

## DB-CURSOR-125
URL generation pertenecerá a HTTP/UI.

## DB-CURSOR-126
SPA serialization pertenecerá a capa superior.

## DB-CURSOR-127
Mutable cursor state será request-scoped.

## DB-CURSOR-128
Stateless codec podrá compartirse si es immutable.

## DB-CURSOR-129
FrankenPHP no compartirá request cursor state.

## DB-CURSOR-130
RoadRunner no compartirá request cursor state.

## DB-CURSOR-131
OpenSwoole no compartirá request cursor state.

## DB-CURSOR-132
Coroutine state estará aislado.

## DB-CURSOR-133
Cursor Planner no ejecutará queries.

## DB-CURSOR-134
Cursor Plan será immutable después de validación.

## DB-CURSOR-135
Cursor Assembler no generará SQL.

## DB-CURSOR-136
Cursor Assembler no será Query Executor.

## DB-CURSOR-137
Cursor Pagination reutilizará Query Engine.

## DB-CURSOR-138
Cursor Pagination reutilizará ORM.

## DB-CURSOR-139
Cursor Pagination reutilizará Hydration.

## DB-CURSOR-140
Cursor Pagination reutilizará Cache architecture.

## DB-CURSOR-141
Cursor Pagination reutilizará Distribution architecture.

## DB-CURSOR-142
Cursor Pagination no creará un segundo Type System.

## DB-CURSOR-143
Cursor Pagination no creará un segundo Security System.

## DB-CURSOR-144
Platform-specific SQL pertenecerá al Compiler.

## DB-CURSOR-145
Version no será Capability.

## DB-CURSOR-146
MySQL será soportable.

## DB-CURSOR-147
MariaDB será soportable.

## DB-CURSOR-148
PostgreSQL será soportable.

## DB-CURSOR-149
SQLite será soportable.

## DB-CURSOR-150
Cursor semantics permanecerán platform-neutral.

## DB-CURSOR-151
Telemetry tendrá bounded cardinality.

## DB-CURSOR-152
Telemetry no registrará raw cursors por defecto.

## DB-CURSOR-153
Diagnostics podrán explicar continuation predicates.

## DB-CURSOR-154
Diagnostics podrán explicar tie-breakers.

## DB-CURSOR-155
Diagnostics podrán explicar binding failures.

## DB-CURSOR-156
Diagnostics no expondrán secrets.

## DB-CURSOR-157
Cursor integrity failure será observable.

## DB-CURSOR-158
Cursor expiration será observable.

## DB-CURSOR-159
Cursor query mismatch será observable.

## DB-CURSOR-160
Cursor ordering mismatch será observable.

## DB-CURSOR-161
Cache consistency seguirá dominando cached cursor pages.

## DB-CURSOR-162
Cursor existence no implicará cursor validity.

## DB-CURSOR-163
Decoded no implicará trusted.

## DB-CURSOR-164
Signed no implicará authorized.

## DB-CURSOR-165
Valid no implicará fresh.

## DB-CURSOR-166
Fresh no implicará same snapshot.

## DB-CURSOR-167
Stable ordering no implicará immutable ordering values.

## DB-CURSOR-168
Cursor pagination reducirá offset cost pero no prometerá coste O(1) universal.

## DB-CURSOR-169
Query plan e índices seguirán determinando performance física.

## DB-CURSOR-170
Correctness tendrá prioridad sobre optimizaciones de cursor.

---

# 181. Modelo formal

Sea un resultado lógico ordenado:

```text id="96cczv"
R = [r1, r2, ..., rn]
```

y una función de clave:

```text id="jqlodk"
K(r)
=
(k1(r), ..., km(r))
```

con orden total efectivo:

```text id="7ivz07"
≺O
```

El cursor de un elemento `rb` representa:

```text id="fr9o9r"
C = K(rb)
```

---

# 182. Forward continuation

La siguiente página será conceptualmente:

```text id="0xy5qn"
Next(C)
=
{
    r ∈ R
    |
    C ≺O K(r)
}
```

tomando los primeros `N` elementos según `O`.

---

# 183. Backward continuation

Conceptualmente:

```text id="of43gd"
Previous(C)
=
{
    r ∈ R
    |
    K(r) ≺O C
}
```

seleccionando los elementos inmediatamente anteriores a la frontera.

---

# 184. Validez del cursor

Formalmente:

```text id="nyoz88"
ValidCursor(C,Q,X)
=
Decodeable(C)
∧ VersionSupported(C)
∧ IntegrityValid(C)
∧ QueryBindingMatches(C,Q)
∧ OrderingBindingMatches(C,Q)
∧ DomainBindingMatches(C,X)
∧ TypeCompatible(C)
∧ NotExpired(C)
```

según las policies habilitadas.

---

# 185. Usabilidad

Pero:

```text id="wcvwe9"
ValidCursor
```

no necesariamente implica:

```text id="00ph1s"
UsableUnderCurrentConsistencyRequirement
```

Puede requerirse además:

```text id="al49na"
AuthorityCompatible
ReplicaFreshEnough
TopologyCompatible
MetadataCompatible
```

---

# 186. Propiedad fundamental

```text id="cok8qd"
CursorPosition
=
OrderingValues
+
OrderingSemantics
+
QueryContext
```

No simplemente:

```text id="07a9j3"
CursorPosition
=
ID
```

---

# 187. Coste

Con índices apropiados, keyset pagination puede permitir:

```text id="nl3vsq"
seek(boundary)
+
fetch(N)
```

en lugar de:

```text id="d1pm0f"
skip(offset)
+
fetch(N)
```

---

# 188. No prometer O(1)

El coste real depende de:

```text id="s2nl7c"
indexes
predicates
joins
ordering
cardinality
database optimizer
distribution
```

Por tanto:

```text id="u0kak3"
CursorPagination
≠
Guaranteed O(1)
```

---

# 189. Comparación arquitectónica

| Característica | Offset | Cursor |
|---|---|---|
| Page numbers | Sí | No naturalmente |
| Jump to page 500 | Sí | No naturalmente |
| High offset cost | Puede ser alto | Evitado normalmente |
| Dataset mutation drift | Alto | Reducido |
| Stable ordering needed | Recomendado | Obligatorio |
| Unique tie-breaker | Recomendado | Normalmente requerido |
| Total count | Opcional | Opcional |
| Infinite scroll | Posible | Excelente |
| Distributed scaling | Difícil | Generalmente mejor |
| Token security | No aplica normalmente | Necesaria para externos |
| Forward navigation | Sí | Sí |
| Backward navigation | Sí | Sí, con planificación |

---

# 190. Arquitectura final

```text id="5zy5qv"
                     Developer API
                          │
                          ▼
               CursorPaginationRequest
                          │
                          ▼
                    CursorCodec
                          │
                          ▼
                   CursorEnvelope
                          │
                          ▼
                  CursorValidator
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          Integrity     Binding      Version
             │            │            │
             └────────────┼────────────┘
                          ▼
              CursorEligibilityAnalyzer
                          │
                          ▼
              CursorPaginationPlanner
             ┌────────────┼────────────┐
             ▼            ▼            ▼
         Ordering     Boundary     Projection
             │            │            │
             └────────────┼────────────┘
                          ▼
               Continuation Predicate
                          │
                          ▼
                     Query Model
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
                      Hydration
                          │
                          ▼
              CursorPaginationAssembler
                          │
               ┌──────────┴──────────┐
               ▼                     ▼
          Next Cursor          Previous Cursor
               │                     │
               └──────────┬──────────┘
                          ▼
                  CursorPageResult
```

---

# 191. Regla maestra final

> **VoltStack Cursor Pagination navegará por fronteras semánticas de un orden lógico estable, no por posiciones físicas dentro de un resultado temporal.**

Siempre:

```text id="u07k5c"
Cursor
≠
Offset
```

```text id="cr90hv"
Cursor
≠
Database Cursor
```

```text id="yylu65"
Cursor
≠
Primary Key
```

```text id="s1jvy8"
Encoded
≠
Trusted
```

```text id="9nyxrv"
Signed
≠
Authorized
```

```text id="55afj0"
Valid Cursor
≠
Same Snapshot
```

y:

```text id="x0bs7v"
Stable Ordering
≠
Immutable Dataset
```

---

# 192. Resultado arquitectónico

VoltStack podrá proporcionar una API simple:

```php id="jvts79"
$page = Order::query()
    ->where('status', OrderStatus::PAID)
    ->orderBy('created_at', 'desc')
    ->cursorPaginate(50);
```

y posteriormente:

```php id="1o98n2"
$page = Order::query()
    ->where('status', OrderStatus::PAID)
    ->orderBy('created_at', 'desc')
    ->cursorPaginate(
        perPage: 50,
        after: $nextCursor,
    );
```

mientras internamente conserva:

```text id="7d0ey4"
typed boundaries
stable ordering
tie-breakers
query binding
tenant/shard isolation
cursor integrity
platform-independent semantics
replica consistency
resource governance
```

sin convertir la API pública en una abstracción dependiente de SQL.

---

# 193. Relación con el siguiente sistema

Hasta ahora:

```text id="0hqpx9"
199 Pagination
    ↓
navigation by pages

200 Cursor Pagination
    ↓
navigation by logical boundaries
```

El siguiente problema será diferente:

```text id="lj3xvu"
"Necesito procesar todos los registros,
pero no quiero cargarlos todos a memoria."
```

Eso pertenece a:

```text id="o6fuh4"
Chunk Processing
```

---

# 194. Siguiente documento

```text id="f6kr91"
201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
```

El siguiente documento definirá:

```text id="g9yvlg"
Chunk Processing
├── chunk model
├── chunk size
├── chunk traversal
├── offset chunking
├── keyset chunking
├── chunkById
├── ordered chunking
├── mutation-safe traversal
├── processing callbacks
├── cancellation
├── checkpoints
├── retries
├── transaction boundaries
├── error policies
├── resumability
├── parallel processing
├── tenant/shard isolation
├── persistent runtime safety
├── telemetry
└── resource governance
```

estableciendo especialmente:

```text id="9kjohh"
Chunk
≠
Page
≠
Transaction
≠
Batch Write
≠
Stream
```

y la regla central:

> **Un chunk será una unidad acotada de lectura y procesamiento, nunca una frontera transaccional implícita ni una página destinada necesariamente a navegación humana.**