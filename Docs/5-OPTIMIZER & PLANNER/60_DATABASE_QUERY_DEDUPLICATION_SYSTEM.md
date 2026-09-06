# 60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md

# VoltStack Quantum Database
## Query Deduplication System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 60 — Query Deduplication System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Optimizer / Deduplication  
**Versión:** 1.0

---

# 1. Propósito

`Query Deduplication System` define la infraestructura responsable de detectar representaciones duplicadas, estructuras equivalentes y estados de optimización ya explorados dentro del Query Engine de VoltStack.

Su objetivo principal es impedir que:

```text
Normalization
Rewrite
Optimization
Join Enumeration
Subquery Transformation
Alternative Generation
```

produzcan cantidades innecesarias o exponenciales de representaciones equivalentes.

La deduplicación podrá aplicarse a:

```text
Expressions
Predicates
Relations
Join Alternatives
Subqueries
CTEs
Set Operations
Aggregate Expressions
Window Expressions
Logical Operators
Optimization States
Logical Plans
```

pero siempre respetando la semántica completa de la consulta.

---

# 2. Regla maestra

VoltStack nunca considerará dos estructuras intercambiables únicamente porque:

```text
se ven iguales
```

o:

```text
generarían SQL parecido
```

La deduplicación deberá establecer explícitamente qué clase de equivalencia está siendo utilizada.

```text
Identity
    ≠
Structural Equality
    ≠
Normalized Structural Equality
    ≠
Semantic Equivalence
    ≠
Logical Equivalence
    ≠
Runtime Equality
```

---

# 3. Problema fundamental

Considérese:

```text
a = 1 AND b = 2
```

y:

```text
b = 2 AND a = 1
```

Pueden ser lógicamente equivalentes bajo ciertas condiciones.

Sin embargo:

```text
volatile_function() = 1
AND
b = 2
```

no deberá reordenarse o deduplicarse ingenuamente si el modelo semántico permite efectos observables o volatilidad.

Por tanto:

```text
same-looking structure
≠
safe interchangeable structure
```

---

# 4. Posición arquitectónica

```text
Query Builder
      │
      ▼
Query Model / AST
      │
      ▼
Normalization
      │
      ▼
Semantic Analysis
      │
      ▼
SemanticQueryArtifact
      │
      ▼
Query Optimizer
      │
      ├── Rewrite System
      ├── Optimization Rules
      ├── Predicate Optimizer
      ├── Join Optimizer
      │
      └── Deduplication System
                 │
                 ▼
          Unique Alternatives
                 │
                 ▼
            Query Planner
```

---

# 5. Deduplication como servicio transversal

La deduplicación no será una única fase aislada.

Será utilizada en diferentes puntos:

```text
Normalization
Optimization
Alternative Generation
Memoization
Planning
Compilation Cache
```

pero cada consumidor deberá especificar:

```text
qué equivalencia necesita
```

---

# 6. Deduplication ≠ Normalization

Normalization transforma una estructura hacia una representación canónica cuando esa transformación está permitida.

Ejemplo:

```text
NOT (NOT x)
```

podría normalizarse a:

```text
x
```

si las reglas semánticas lo permiten.

Deduplication, en cambio, responde:

```text
¿ya tengo esta representación?
```

---

# 7. Deduplication ≠ Optimization

Deduplication no intenta encontrar una consulta más rápida.

Sólo evita conservar alternativas redundantes.

---

# 8. Deduplication ≠ Query Cache

Query Cache almacena resultados o artifacts reutilizables entre operaciones.

Deduplication trabaja principalmente dentro de:

```text
una operación de procesamiento
```

aunque sus fingerprints puedan reutilizarse posteriormente como claves de caches.

---

# 9. Deduplication ≠ Result Cache

Nunca deberá confundirse:

```text
same query semantics
```

con:

```text
same database result forever
```

El resultado depende del estado de datos.

---

# 10. Deduplication ≠ Common Subexpression Execution

Detectar que dos expresiones son equivalentes no implica automáticamente ejecutarlas una sola vez.

Ejemplo:

```text
random()
random()
```

aunque estructuralmente iguales, pueden requerir evaluaciones independientes.

---

# 11. Identity

Cada nodo puede poseer identidad propia:

```text
ExpressionId
PredicateId
RelationId
JoinId
SubqueryId
CteId
WindowId
```

Dos nodos diferentes pueden tener estructura idéntica.

```text
Identity(A) ≠ Identity(B)
```

no implica:

```text
Structure(A) ≠ Structure(B)
```

---

# 12. Structural Equality

Dos estructuras son estructuralmente iguales cuando sus componentes relevantes coinciden exactamente.

Ejemplo:

```text
Add(Column(a), Parameter(p1))
```

contra:

```text
Add(Column(a), Parameter(p1))
```

---

# 13. Structural equality sensible a identidad

Dependiendo del contexto, `ParameterId` puede formar parte de la comparación.

Por ejemplo:

```text
a = :p1
```

y:

```text
a = :p2
```

no son necesariamente el mismo artifact estructural.

---

# 14. Normalized Structural Equality

Después de normalización:

```text
A
```

y:

```text
B
```

pueden producir la misma forma canónica.

Entonces:

```text
NormalizedFingerprint(A)
=
NormalizedFingerprint(B)
```

---

# 15. Semantic Equivalence

Dos estructuras pueden ser semánticamente equivalentes aunque no tengan exactamente la misma forma.

Ejemplo potencial:

```text
a = 5
AND
a = b
```

puede implicar:

```text
b = 5
```

pero esa equivalencia depende del contexto semántico y SQL 3VL.

---

# 16. Logical Equivalence

Dentro del Optimizer, dos árboles distintos pueden representar la misma operación relacional.

Ejemplo:

```text
(A INNER JOIN B) INNER JOIN C
```

y:

```text
A INNER JOIN (B INNER JOIN C)
```

pueden pertenecer al mismo grupo lógico si se cumplen las precondiciones de asociatividad.

---

# 17. Runtime Equality

Dos ejecuciones pueden devolver los mismos valores en un instante concreto.

Eso no demuestra equivalencia semántica.

Ejemplo:

```text
WHERE a = 1
```

y:

```text
WHERE a = 2
```

pueden devolver cero filas ambas.

No son consultas equivalentes.

---

# 18. EquivalenceLevel

VoltStack definirá explícitamente:

```php
enum EquivalenceLevel
{
    case IDENTITY;
    case STRUCTURAL;
    case NORMALIZED_STRUCTURAL;
    case SEMANTIC;
    case LOGICAL;
}
```

---

# 19. No nivel implícito

Una API como:

```php
$deduplicator->same($a, $b);
```

será evitada.

Preferible:

```php
$deduplicator->equivalent(
    $a,
    $b,
    EquivalenceLevel::SEMANTIC
);
```

---

# 20. Fingerprint architecture

El sistema utilizará múltiples tipos de fingerprints.

```text
StructuralFingerprint
NormalizedFingerprint
SemanticFingerprint
LogicalFingerprint
```

No existirá un único:

```text
queryHash
```

para todos los propósitos.

---

# 21. StructuralFingerprint

Representa la estructura exacta relevante de un artifact.

Ejemplo conceptual:

```text
Predicate(
    "=",
    Column(users.id),
    Parameter(p1)
)
```

---

# 22. SemanticFingerprint

Incluye información resuelta como:

```text
symbols
types
domains
scopes
relations
functions
collations
nullability
capabilities
schema identity
semantic extension versions
```

---

# 23. LogicalFingerprint

Describe una operación lógica ya abstraída de detalles irrelevantes para una fase concreta.

Puede utilizarse para:

```text
join alternative deduplication
memo groups
logical plan equivalence
```

---

# 24. Fingerprint ≠ hash

Conceptualmente:

```text
Fingerprint
```

es una representación canónica de identidad/equivalencia.

Puede derivarse un:

```text
Hash(Fingerprint)
```

para indexación rápida.

Nunca se asumirá:

```text
same hash
→ definitely equivalent
```

sin política de colisiones.

---

# 25. Fingerprint model

```php
interface Fingerprint
{
    public function bytes(): string;

    public function version(): FingerprintVersion;
}
```

---

# 26. Typed fingerprints

Preferible:

```php
StructuralFingerprint
SemanticFingerprint
LogicalFingerprint
```

en lugar de:

```php
string $hash
```

---

# 27. Fingerprint versioning

Cada algoritmo tendrá:

```text
FingerprintVersion
```

para evitar reutilizar fingerprints producidos por reglas incompatibles.

---

# 28. Structural fingerprint inputs

Podrá incluir:

```text
node kind
operator
child order
identifiers
parameter identity policy
metadata
extension identity
explicit grouping
query modifiers
```

---

# 29. Semantic fingerprint inputs

Podrá incluir:

```text
resolved SymbolId
resolved RelationId
resolved function semantic ID
QueryType
domain identity
nullability semantics
collation
scope
correlation
schema fingerprint
capability requirements
security semantics
extension semantics
```

---

# 30. Runtime values

Los valores runtime no formarán parte normalmente de:

```text
SemanticFingerprint
```

Ejemplo:

```text
users.id = :p1
```

deberá poder compartir estructura compilada para:

```text
p1 = 10
```

y:

```text
p1 = 50
```

---

# 31. Exceptions

Algunas capacidades pueden requerir:

```text
compile-time constants
```

por ejemplo ciertos frame offsets o platform constructs.

En esos casos podrá existir:

```text
SpecializationFingerprint
```

separado.

---

# 32. Parameter identity

VoltStack distinguirá:

```text
ParameterId
ParameterPosition
Placeholder
RuntimeBinding
```

---

# 33. Placeholder exclusion

Esto:

```text
$1
?
:p1
```

pertenece al Compiler.

No deberá definir equivalencia semántica del Query Model.

---

# 34. Alpha-equivalent parameters

Dos consultas:

```text
a = Parameter(P17)
```

y:

```text
a = Parameter(P94)
```

pueden ser estructuralmente distintas bajo identidad estricta pero alpha-equivalentes bajo una política de renombrado canónico.

---

# 35. Parameter canonicalization

Podrá existir:

```text
CanonicalParameterIndex
```

asignado según recorrido determinista.

Ejemplo:

```text
P17 → CP0
P94 → CP1
```

---

# 36. Alpha-equivalence

Dos artifacts podrán considerarse equivalentes si sólo difieren en identidades locales renombrables.

Esto aplica potencialmente a:

```text
parameters
local relation aliases
local symbol IDs
CTE internal IDs
subquery-local IDs
```

pero no a identidades semánticas externas.

---

# 37. Alpha-equivalence ≠ string alias equality

Ejemplo:

```sql
FROM users u
```

y:

```sql
FROM users x
```

pueden representar la misma relación lógica cuando las referencias internas se remapean consistentemente.

---

# 38. Canonical renaming

El fingerprint podrá utilizar:

```text
R0
R1
R2

S0
S1

P0
P1
```

según orden estructural determinista.

---

# 39. Scope safety

La canonicalización nunca mezclará IDs de scopes distintos.

---

# 40. Correlation safety

Una referencia:

```text
OuterSymbol(R3)
```

no podrá renombrarse como si fuera un símbolo local.

---

# 41. ScopeFingerprint

Cada scope podrá producir:

```text
ScopeFingerprint
```

basado en su estructura semántica y dependencias externas.

---

# 42. Expression deduplication

El sistema podrá detectar:

```text
price * quantity
```

repetido varias veces.

---

# 43. Expression identity vs reuse

Detectar equivalencia no implica necesariamente reemplazar ambos nodos por la misma instancia.

Los AST/artifacts continuarán siendo inmutables.

---

# 44. Common expression candidate

Se podrá producir:

```php
final readonly class CommonExpressionCandidate
{
    public function __construct(
        public SemanticFingerprint $fingerprint,
        public array $occurrences,
        public ExpressionProperties $properties,
    ) {}
}
```

---

# 45. Pure expressions

Las expresiones:

```text
deterministic
side-effect free
context independent
```

son mejores candidatos para reutilización.

---

# 46. Volatile expressions

Ejemplos conceptuales:

```text
RANDOM()
CURRENT_SEQUENCE_VALUE()
extension_volatile()
```

no deberán deduplicarse como evaluaciones.

---

# 47. Stable vs volatile

El sistema podrá reconocer:

```php
enum EvaluationStability
{
    case IMMUTABLE;
    case STABLE;
    case VOLATILE;
    case UNKNOWN;
}
```

---

# 48. IMMUTABLE

Mismo input semántico produce mismo output.

---

# 49. STABLE

Puede ser estable dentro de un:

```text
statement
transaction
operation
```

según descriptor.

No equivale necesariamente a `IMMUTABLE`.

---

# 50. VOLATILE

Cada evaluación puede ser observable de forma independiente.

---

# 51. UNKNOWN

Se tratará conservadoramente.

---

# 52. Predicate deduplication

Ejemplo:

```text
a = 1
AND
a = 1
```

puede reducirse potencialmente a:

```text
a = 1
```

si el predicate es seguro para idempotencia.

---

# 53. SQL 3VL

Predicate deduplication deberá preservar:

```text
TRUE
FALSE
UNKNOWN
```

---

# 54. Idempotence

Para un predicate puro `P`:

```text
P AND P
≡
P
```

y:

```text
P OR P
≡
P
```

bajo SQL truth semantics apropiadas.

---

# 55. Volatile predicate

Si `P` contiene una función volatile:

```text
P AND P
```

no deberá reducirse automáticamente a una sola evaluación.

---

# 56. Predicate fingerprint

Incluirá:

```text
predicate kind
operator semantics
child expressions
type/domain
collation
null semantics
scope
volatility
extensions
```

---

# 57. Commutative predicate canonicalization

Para operadores demostrablemente conmutativos podrá utilizarse orden canónico.

Ejemplo:

```text
a = b
```

y:

```text
b = a
```

pueden obtener fingerprint equivalente cuando:

```text
comparison semantics
type coercion
collation
domain rules
```

lo permiten.

---

# 58. No universal operand sorting

No se ordenarán operandos arbitrariamente para:

```text
<
>
LIKE
IS DISTINCT FROM
extension operators
```

sin descriptor apropiado.

---

# 59. AND/OR canonicalization

Una conjunción pura puede representarse canónicamente como:

```text
AND(
    sorted(unique(children))
)
```

sólo cuando:

```text
reordering safe
deduplication safe
evaluation semantics preserved
```

---

# 60. Structural order preservation

Aunque un fingerprint lógico pueda ignorar cierto orden, el AST original seguirá preservándolo.

---

# 61. Query expression deduplication

El sistema podrá detectar duplicados en:

```text
SELECT expressions
GROUP BY expressions
ORDER BY expressions
window partition expressions
window ordering expressions
aggregate arguments
```

pero la acción permitida dependerá del contexto.

---

# 62. Duplicate projection

```sql
SELECT a, a
```

no deberá convertirse a:

```sql
SELECT a
```

porque cambia:

```text
output arity
output column identity
```

---

# 63. Detection ≠ removal

El sistema podrá reconocer:

```text
same expression
```

sin eliminarla.

---

# 64. Duplicate GROUP BY key

```text
GROUP BY a, a
```

podría simplificarse a:

```text
GROUP BY a
```

cuando el Aggregation Optimizer determine que es seguro.

Deduplication sólo proporciona equivalence facts.

---

# 65. Duplicate ORDER BY key

La eliminación requiere considerar:

```text
direction
null ordering
collation
stability
```

---

# 66. Subquery deduplication

Dos subqueries pueden ser estructural o semánticamente equivalentes.

Ejemplo:

```text
SELECT id FROM users WHERE active = TRUE
```

repetido en dos lugares.

---

# 67. Equivalent subquery ≠ shared execution

Aunque sean equivalentes:

```text
Subquery A
Subquery B
```

no significa automáticamente:

```text
execute once
```

---

# 68. Correlated subqueries

Dos subqueries textualmente iguales pueden pertenecer a diferentes outer scopes.

Por tanto pueden no ser semánticamente equivalentes.

---

# 69. Correlation fingerprint

Deberá incluir:

```text
outer scope relationship
outer symbols
correlation depth
correlation semantics
```

---

# 70. Subquery fingerprint

Incluirá:

```text
query structure
output relation
semantic types
correlations
dependencies
metadata
capabilities
security boundaries
schema fingerprint
```

---

# 71. CTE deduplication

Dos CTE definitions pueden tener bodies equivalentes.

Esto no significa que deban fusionarse automáticamente.

---

# 72. CTE name semantics

CTE names participan en:

```text
scope
symbol resolution
recursive references
external references
```

---

# 73. Recursive CTE

La deduplicación deberá preservar:

```text
recursive identity
anchor/member relationship
dependency graph
```

---

# 74. CTE reuse analysis

Podrá detectarse:

```text
equivalent CTE body
```

como oportunidad de optimización futura.

Pero fusionar CTEs será responsabilidad de una regla con proof.

---

# 75. Materialization intent

Dos CTEs con mismo body pero diferente:

```text
materialization intent
```

no serán necesariamente intercambiables.

---

# 76. Security metadata

Dos CTEs estructuralmente iguales con distinta security provenance no deberán fusionarse.

---

# 77. Join alternative deduplication

Documento 59 puede generar:

```text
Alternative A
Alternative B
Alternative C
...
```

Diferentes secuencias de rewrites pueden producir el mismo árbol lógico.

---

# 78. Ejemplo

Ruta 1:

```text
strengthen J1
→ reorder
→ eliminate J3
```

Ruta 2:

```text
eliminate J3
→ strengthen J1
→ reorder
```

pueden terminar en la misma estructura.

---

# 79. JoinAlternativeFingerprint

Será utilizado para detectar esa convergencia.

---

# 80. Join fingerprint inputs

Podrá incluir:

```text
logical join type
canonical relation groups
join predicates
dependency constraints
output relation semantics
null-extension
correlation
barriers
```

---

# 81. Commutative inner joins

Un fingerprint lógico puede canonicalizar:

```text
A INNER JOIN B
```

y:

```text
B INNER JOIN A
```

dentro de un equivalence group si todas las propiedades requeridas se cumplen.

---

# 82. Outer joins

No tendrán esa canonicalización por defecto.

---

# 83. SEMI JOIN

```text
A SEMI JOIN B
```

no equivale a:

```text
B SEMI JOIN A
```

---

# 84. ANTI JOIN

Tampoco es conmutativo.

---

# 85. Optimization state deduplication

El optimizador mantendrá un registro de:

```text
already explored states
```

---

# 86. OptimizationState

Modelo conceptual:

```php
final readonly class OptimizationState
{
    public function __construct(
        public LogicalArtifact $artifact,
        public OptimizationFactSet $facts,
        public OptimizationBarrierSet $barriers,
        public OptimizationStateFingerprint $fingerprint,
    ) {}
}
```

---

# 87. State fingerprint

No deberá considerar únicamente el árbol.

Dos estados con mismo árbol pero diferentes facts relevantes pueden permitir diferentes reglas futuras.

---

# 88. Example

```text
State A:
    Tree X
    Unique(R2.id) known

State B:
    Tree X
    uniqueness unknown
```

No deberán fusionarse si esa diferencia afecta reglas posteriores.

---

# 89. Fact-sensitive deduplication

Por tanto:

```text
OptimizationStateFingerprint
=
LogicalFingerprint
+
RelevantFactFingerprint
+
BarrierFingerprint
```

---

# 90. Relevant facts

No todos los diagnostics deberán formar parte del fingerprint.

Sólo facts que puedan modificar comportamiento futuro.

---

# 91. Diagnostic independence

Ejemplo:

```text
warning already emitted
```

no deberá crear un nuevo estado lógico.

---

# 92. Rule history independence

Dos estados equivalentes alcanzados mediante reglas diferentes pueden deduplicarse.

La historia se conserva en provenance, pero no necesita definir equivalencia.

---

# 93. Provenance merge

Cuando dos caminos convergen:

```text
State A
State B
   │
   ▼
Same State C
```

podrá conservarse:

```text
multiple provenance paths
```

sin duplicar el estado.

---

# 94. Provenance budget

No se almacenarán infinitos caminos alternativos de provenance.

Existirá:

```text
ProvenanceBudget
```

---

# 95. Memoization

El sistema podrá soportar:

```text
Memo
```

como estructura para registrar equivalencias y alternativas.

---

# 96. Memo ≠ generic cache

El `Optimizer Memo` representa:

```text
equivalence groups
logical expressions
alternative structures
```

dentro del proceso de optimización.

---

# 97. MemoGroup

```php
final class MemoGroup
{
    public function __construct(
        public MemoGroupId $id,
        public LogicalFingerprint $logicalFingerprint,
    ) {}
}
```

---

# 98. MemoExpression

```php
final readonly class MemoExpression
{
    public function __construct(
        public MemoExpressionId $id,
        public LogicalOperator $operator,
        public array $childGroups,
        public LogicalPropertySet $properties,
    ) {}
}
```

---

# 99. Memo group semantics

Un grupo representa:

```text
logically equivalent alternatives
```

no:

```text
identical object instances
```

---

# 100. Cascades-style compatibility

La arquitectura no obligará V1 a implementar un optimizer Cascades completo.

Pero el modelo deberá permitir en el futuro:

```text
Memo Groups
Transformation Rules
Implementation Rules
Physical Properties
Costing
```

sin rediseñar la identidad lógica.

---

# 101. V1 strategy

V1 podrá utilizar:

```text
FingerprintSet
+
AlternativeSet
+
OptimizationStateRegistry
```

como implementación más simple.

---

# 102. Future Memo

Posteriormente:

```text
FingerprintSet
→ Memo
```

sin cambiar contratos externos.

---

# 103. DeduplicationSet

```php
final class DeduplicationSet
{
    public function contains(Fingerprint $fingerprint): bool;

    public function add(
        Fingerprint $fingerprint,
        object $artifact,
    ): DeduplicationResult;
}
```

---

# 104. Collision safety

Nunca se almacenará solamente:

```text
64-bit hash
```

como prueba definitiva de equivalencia.

---

# 105. Two-stage comparison

Modelo:

```text
Fast Hash
   │
   ▼
Candidate Bucket
   │
   ▼
Canonical Fingerprint / Deep Equality
   │
   ▼
Equivalent?
```

---

# 106. Cryptographic hash

No se exige necesariamente un hash criptográfico para estructuras internas.

La prioridad es:

```text
determinism
collision handling
performance
versioning
```

---

# 107. Stable hashing

No utilizar:

```text
PHP object hash
memory address
process-random hash
```

como fingerprint persistente.

---

# 108. Cross-process stability

Si un fingerprint se utiliza en cache persistente deberá ser reproducible entre:

```text
requests
workers
processes
machines
```

cuando inputs/versiones sean iguales.

---

# 109. Canonical serialization

Los fingerprints podrán derivarse de una serialización canónica binaria/interna.

No de:

```text
serialize($object)
```

sin contrato estable.

---

# 110. Canonical writer

Propuesta:

```php
interface FingerprintWriter
{
    public function writeTag(string $tag): void;

    public function writeInt(int $value): void;

    public function writeString(string $value): void;

    public function writeFingerprint(Fingerprint $fingerprint): void;
}
```

---

# 111. Type tags

Cada elemento deberá incluir tags explícitos.

Ejemplo:

```text
COLUMN
SYMBOL:42
TYPE:UUID
```

para evitar ambigüedad de concatenación.

---

# 112. Length framing

La serialización deberá ser:

```text
length-delimited
```

o equivalente.

No:

```text
"a" + "bc"
```

vs:

```text
"ab" + "c"
```

sin separación segura.

---

# 113. Metadata

No toda metadata participa en todos los fingerprints.

---

# 114. Metadata classification

Podrá existir:

```php
enum FingerprintParticipation
{
    case NONE;
    case STRUCTURAL;
    case SEMANTIC;
    case LOGICAL;
    case COMPILATION;
    case EXECUTION;
}
```

---

# 115. Example metadata

```text
debug label
```

normalmente:

```text
NONE
```

---

# 116. Security metadata

Puede ser:

```text
SEMANTIC
LOGICAL
COMPILATION
```

dependiendo del tipo.

---

# 117. Query timeout

Normalmente no cambia la semántica lógica.

Podría participar en:

```text
ExecutionFingerprint
```

pero no en `LogicalFingerprint`.

---

# 118. Trace ID

Nunca deberá afectar query equivalence.

---

# 119. Schema fingerprint

La semántica de una query puede depender de:

```text
resolved column type
collation
constraints
function resolution
```

Por tanto algunos fingerprints incluirán:

```text
SchemaFingerprint
```

---

# 120. Schema version ≠ semantic schema fingerprint

Una versión global puede invalidar demasiado.

Preferible, cuando sea viable:

```text
dependency-aware schema fingerprint
```

---

# 121. Dependency fingerprint

Podrá calcularse sobre los objetos realmente utilizados:

```text
tables
columns
types
functions
collations
constraints
```

---

# 122. Schema-independent fingerprints

Structural fingerprints anteriores a semantic analysis no necesitan schema fingerprint.

---

# 123. Capability fingerprint

Algunas equivalencias dependen de capacidades disponibles.

Ejemplo:

```text
native semantic operator
exact emulation
unsupported extension
```

---

# 124. CapabilitySetFingerprint

Podrá incluirse en:

```text
optimized artifact
planner memo
compiled query cache
```

cuando sea relevante.

---

# 125. Platform identity

No se incluirá:

```text
mysql
postgres
```

directamente cuando basten capabilities.

---

# 126. Rule fingerprint

Optimized artifacts pueden depender de:

```text
enabled rules
rule versions
optimizer profile
```

---

# 127. OptimizerProfileFingerprint

```text
OptimizerProfileFingerprint
=
RuleSet
+
RuleVersions
+
SemanticPolicies
+
OptimizationBudgetsClass
```

según necesidad.

---

# 128. Budget values

Un budget distinto puede producir diferente conjunto de alternativas.

Por tanto puede ser relevante para fingerprints de:

```text
optimization result
```

pero no para:

```text
query semantics
```

---

# 129. Fingerprint domains

Se definirán dominios separados:

```text
QUERY_STRUCTURE
QUERY_SEMANTICS
OPTIMIZER_STATE
LOGICAL_ALTERNATIVE
PLANNER_STATE
COMPILATION
```

---

# 130. Domain separation

El mismo contenido en dos dominios distintos nunca deberá producirse como clave accidentalmente intercambiable.

Conceptualmente:

```text
HASH(
    "VOLTSTACK:QUERY:SEMANTIC",
    ...
)
```

vs:

```text
HASH(
    "VOLTSTACK:PLAN:LOGICAL",
    ...
)
```

---

# 131. Set operations

Deduplication deberá respetar:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

y sus multiplicity semantics.

---

# 132. UNION operand deduplication

```text
A UNION A
```

puede ser semánticamente equivalente a:

```text
A
```

para `UNION DISTINCT` bajo condiciones apropiadas.

Pero:

```text
A UNION ALL A
```

no lo es.

---

# 133. INTERSECT

La idempotencia dependerá de:

```text
DISTINCT
ALL
```

y multiset semantics.

---

# 134. EXCEPT

No deberá aplicarse una regla de duplicación basada únicamente en set algebra clásico ignorando SQL bag semantics.

---

# 135. Aggregates

Dos aggregate expressions:

```text
SUM(amount)
SUM(amount)
```

pueden ser equivalentes como expresiones.

---

# 136. Aggregate reuse

La deduplicación de cálculo deberá considerar:

```text
DISTINCT
FILTER
aggregate ordering
function descriptor
type
domain
collation
volatility
```

---

# 137. COUNT

```text
COUNT(*)
```

y:

```text
COUNT(column)
```

no son equivalentes.

---

# 138. Window functions

Dos window expressions requieren igualdad/equivalencia de:

```text
function
arguments
partition
window ordering
effective frame
null treatment
named-window resolution
```

---

# 139. Named windows

Dos nombres diferentes pueden resolver a la misma:

```text
EffectiveWindowSpecification
```

y entonces sus window expressions pueden ser semánticamente equivalentes.

---

# 140. Implicit frame

El fingerprint semántico deberá usar:

```text
EffectiveWindowFrame
```

no simplemente:

```text
frame omitted
```

cuando se evalúe equivalencia semántica.

---

# 141. ROW_NUMBER

Dos `ROW_NUMBER()` con mismo window spec pueden compartir equivalence facts.

Pero la estabilidad respecto a ties seguirá formando parte de semantic properties.

---

# 142. Raw expressions

`RawExpression` será una barrera de deduplicación conservadora.

---

# 143. Raw fingerprint

Podrá existir fingerprint estructural del raw payload para cache local.

Pero no se inferirá automáticamente equivalencia semántica.

---

# 144. Example

```text
RAW("a + b")
```

y:

```text
RAW("a+b")
```

no se canonicalizarán mediante parsing improvisado.

---

# 145. Semantic extensions

Una extensión podrá proporcionar:

```text
FingerprintContributor
```

---

# 146. FingerprintContributor

```php
interface FingerprintContributor
{
    public function contribute(
        FingerprintContext $context,
        FingerprintWriter $writer,
    ): void;
}
```

---

# 147. Extension version

La versión semántica de la extensión deberá participar cuando pueda cambiar significado.

---

# 148. Unknown extension

Si una extensión no proporciona información suficiente:

```text
deduplication barrier
```

---

# 149. Security barriers

Dos expresiones estructuralmente iguales con distinta:

```text
security provenance
security domain
policy identity
```

pueden no ser fusionables.

---

# 150. Security-aware fingerprint

El `SemanticFingerprint` podrá incluir:

```text
SecuritySemanticFingerprint
```

cuando sea relevante.

---

# 151. Policy-generated predicates

No deberán fusionarse con predicates de usuario si hacerlo elimina provenance necesaria para:

```text
audit
security verification
barrier enforcement
```

---

# 152. Logical equivalence vs provenance

Dos predicates pueden ser lógicamente equivalentes y aun así necesitar conservar dos provenance records.

Por tanto:

```text
deduplicate computation
≠
discard provenance
```

---

# 153. Provenance union

Cuando se fusionen equivalence entries:

```text
ProvenanceSet
=
Provenance(A)
∪
Provenance(B)
```

si la política lo permite.

---

# 154. Lineage

Lineage tampoco deberá perderse.

---

# 155. Equivalent expression lineage

Si dos expresiones equivalentes provienen de diferentes source columns, eso puede ser información relevante.

Ejemplo derivado por equality constraints.

Por tanto:

```text
semantic equivalence
≠
identical lineage
```

---

# 156. Equivalence class

El sistema podrá representar:

```php
final class EquivalenceClass
{
    public function __construct(
        public EquivalenceClassId $id,
        public EquivalenceLevel $level,
        public Fingerprint $fingerprint,
    ) {}
}
```

---

# 157. Members

Una clase puede contener múltiples:

```text
artifact identities
```

con:

```text
shared equivalence
different provenance
```

---

# 158. Representative

Podrá seleccionarse un:

```text
canonical representative
```

para procesamiento interno.

---

# 159. Representative selection

Será:

```text
deterministic
stable
policy-driven
```

---

# 160. Representative ≠ mutation

Los demás artifacts no serán mutados.

---

# 161. Equivalence proof

Para equivalencias no triviales podrá almacenarse:

```php
final readonly class EquivalenceProof
{
    public function __construct(
        public EquivalenceLevel $level,
        public array $supportingFacts,
        public array $rules,
        public ProofFingerprint $fingerprint,
    ) {}
}
```

---

# 162. Structural equality proof

Puede ser implícita mediante canonical fingerprint + deep comparison.

---

# 163. Semantic equivalence proof

Puede requerir:

```text
type facts
constraint facts
operator properties
scope facts
```

---

# 164. Logical equivalence proof

Puede requerir:

```text
associativity
commutativity
null-preservation
cardinality preservation
```

---

# 165. No proof fabrication

Si la equivalencia no puede demostrarse:

```text
keep both
```

---

# 166. Query-level deduplication

Dos queries completas podrán tener:

```text
same semantic fingerprint
```

aunque tengan IDs internos distintos.

---

# 167. QuerySemanticFingerprint

Modelo:

```text
QuerySemanticFingerprint
=
QueryForm
+
SemanticGraph
+
OutputRelation
+
Dependencies
+
Capabilities
+
SemanticMetadata
+
SchemaDependencies
```

---

# 168. Output contract

Dos queries no son intercambiables si producen diferente:

```text
column count
column order
column semantic types
column names when API-observable
domain identities
```

---

# 169. Output aliases

La participación de aliases dependerá del fingerprint.

Para ejecución interna quizá no importen.

Para API result mapping sí pueden importar.

---

# 170. ResultShapeFingerprint

Por ello existirá:

```text
ResultShapeFingerprint
```

separado.

---

# 171. Compilation fingerprint

Posteriormente podrá definirse:

```text
CompilationFingerprint
=
SemanticFingerprint
+
TargetCapabilities
+
CompilerVersion
+
RequiredSpecializations
```

---

# 172. Prepared statement cache

Esto permitirá compartir compiled queries sin mezclar runtime bindings.

---

# 173. Query cache key layering

Arquitectura futura:

```text
StructuralFingerprint
        │
        ▼
SemanticFingerprint
        │
        ▼
LogicalFingerprint
        │
        ▼
CompilationFingerprint
        │
        ▼
Execution/Result Cache Key
```

Cada capa añade contexto.

---

# 174. Deduplication workflow

```text
Artifact
   │
   ▼
Select Equivalence Domain
   │
   ▼
Canonicalize IDs
   │
   ▼
Build Typed Fingerprint
   │
   ▼
Fast Hash Lookup
   │
   ▼
Candidate Exists?
   │
   ├── no ──► register
   │
   └── yes
          │
          ▼
   Collision-safe comparison
          │
          ▼
   Equivalent?
      │       │
     no      yes
      │       │
      ▼       ▼
 register   merge/deduplicate
```

---

# 175. DeduplicationResult

```php
final readonly class DeduplicationResult
{
    public function __construct(
        public DeduplicationStatus $status,
        public object $representative,
        public ?EquivalenceProof $proof,
    ) {}
}
```

---

# 176. Status

```php
enum DeduplicationStatus
{
    case UNIQUE;
    case DUPLICATE;
    case EQUIVALENT;
    case COLLISION;
    case BARRIER;
}
```

---

# 177. Collision

Una hash collision no será tratada como query equivalence.

---

# 178. Collision diagnostics

Podrá registrarse internamente:

```text
FingerprintHashCollision
```

sin afectar correctness.

---

# 179. Deduplication budgets

La deduplicación también consumirá recursos.

---

# 180. DeduplicationBudget

```php
final readonly class DeduplicationBudget
{
    public function __construct(
        public int $maxEntries,
        public int $maxFingerprintDepth,
        public int $maxFingerprintBytes,
        public int $maxEquivalenceClasses,
        public int $maxMembersPerClass,
        public int $maxDeepComparisons,
        public int $maxMemoGroups,
        public int $maxMemoExpressions,
        public int $maxProvenancePaths,
    ) {}
}
```

---

# 181. Budget exhaustion

No deberá cambiar semántica.

Si se agota:

```text
stop deduplicating aggressively
```

y continuar con estructuras válidas dentro de los límites generales del Optimizer.

---

# 182. Memory governance

El sistema podrá descartar:

```text
nonessential provenance
low-priority fingerprints
completed search-state indexes
```

según políticas explícitas.

Nunca artifacts necesarios para correctness.

---

# 183. Persistent runtime safety

Los registros de deduplicación de una query serán:

```text
operation-scoped
```

---

# 184. Shared infrastructure

Podrán compartirse:

```text
fingerprint algorithms
immutable descriptor registries
operator property descriptors
extension fingerprint contributors
```

---

# 185. Prohibido

```php
static array $seenQueries = [];
```

como estado mutable global.

---

# 186. Request isolation

Dos requests en FrankenPHP no compartirán accidentalmente:

```text
OptimizationStateRegistry
Memo
DeduplicationSet
EquivalenceClasses
```

---

# 187. Cross-request cache

Si se desea reutilización cross-request, se hará mediante un subsistema de cache explícito y versionado.

No reutilizando el state interno del optimizer.

---

# 188. Thread/coroutine safety

El diseño será compatible con futuros runtimes concurrentes evitando:

```text
mutable singleton state
```

---

# 189. Determinism

Mismo artifact y mismo contexto deberán producir:

```text
same canonical representation
same fingerprint
same representative ordering
```

---

# 190. Locale independence

Los fingerprints no dependerán del locale PHP activo.

---

# 191. Floating point

Valores literales floating-point, cuando formen parte estructural especializada, deberán tener encoding canónico.

No:

```text
locale-formatted string
```

---

# 192. Unicode identifiers

La canonicalización de identifiers respetará:

```text
platform identifier semantics
quoting semantics
case folding rules
```

sin aplicar `strtolower()` globalmente.

---

# 193. Identifier semantics

```text
User
user
"User"
```

pueden tener significados distintos según el dialect/platform.

---

# 194. Semantic IDs preferred

Después de Symbol Resolution, fingerprints semánticos utilizarán:

```text
SymbolId
RelationId
SchemaObjectIdentity
```

en lugar de reanalizar strings.

---

# 195. Collation

Comparaciones y ordering pueden depender de:

```text
collation
```

Por tanto la collation forma parte de semantic equivalence cuando corresponda.

---

# 196. Time zone semantics

Funciones y tipos sensibles a timezone deberán incluir la política semántica relevante.

---

# 197. Domain types

Dos columnas físicamente `BIGINT` pero pertenecientes a:

```text
UserId
OrderId
```

no deberán considerarse semánticamente idénticas si VoltStack preserva domain identity.

---

# 198. Casts

Un cast explícito forma parte de la estructura.

No deberá eliminarse del fingerprint porque el storage type parezca igual.

---

# 199. Implicit coercions

El fingerprint semántico deberá incorporar coerciones resueltas cuando afecten meaning.

---

# 200. Function identity

No utilizar sólo:

```text
function name
```

Preferible:

```text
FunctionSemanticId
+
FunctionDescriptorVersion
```

---

# 201. Overloaded functions

Dos llamadas con mismo nombre pero overload distinto no son equivalentes.

---

# 202. Operator identity

Lo mismo aplica a operadores.

---

# 203. Extension operators

Deberán registrar identidad semántica estable.

---

# 204. Diagnostics

Propuesta:

```text
FingerprintBudgetExceeded
FingerprintDepthExceeded
FingerprintSizeExceeded
FingerprintCollisionDetected
UnknownFingerprintContributor
UnsafeDeduplicationBlocked
VolatileExpressionDeduplicationBlocked
SecuritySensitiveDeduplicationBlocked
CorrelationSensitiveDeduplicationBlocked
SchemaFingerprintUnavailable
CapabilityFingerprintUnavailable
MemoBudgetExceeded
EquivalenceClassBudgetExceeded
ProvenanceMergeBudgetExceeded
```

---

# 205. Explainability

El sistema podrá exponer:

```text
Duplicate alternative discarded
```

con información como:

```text
Alternative: J42
Equivalent to: J17
Equivalence: LOGICAL
Fingerprint: ...
Reason:
    same canonical join graph
    same predicates
    same dependencies
    same output semantics
```

---

# 206. Fingerprint secrecy

Fingerprints no deberán incorporar directamente secretos runtime.

---

# 207. Sensitive literals

Los valores runtime están fuera del fingerprint general.

Si existe una especialización que necesite un valor sensible:

```text
value
→ safe keyed digest / specialized opaque identity
```

según política.

No deberá exponerse en telemetry.

---

# 208. Telemetry

Métricas posibles:

```text
fingerprints generated
structural duplicates found
semantic duplicates found
logical duplicates found
optimization states skipped
join alternatives deduplicated
subquery duplicates detected
memo groups created
memo expressions created
hash collisions
deep comparisons
deduplication time
deduplication memory
budget exhaustion
```

---

# 209. Performance objective

La deduplicación debe costar menos que la explosión de alternativas que evita.

---

# 210. Fast path

Para artifacts pequeños:

```text
cached immutable fingerprint
```

podrá evitar recomputación dentro de una operación.

---

# 211. Artifact-local fingerprint cache

Un artifact inmutable puede mantener o asociarse a:

```text
FingerprintMemo
```

siempre que no introduzca mutable cross-request state inseguro.

---

# 212. Side-table approach

Preferencia:

```text
OperationFingerprintTable
ArtifactId → Fingerprint
```

---

# 213. Immutable artifact compatibility

El cálculo de fingerprint no mutará el AST.

---

# 214. Parallel fingerprinting

La arquitectura podrá permitir cálculo paralelo de fingerprints de subárboles independientes.

---

# 215. Merkle-style composition

Los fingerprints podrán componerse jerárquicamente:

```text
Fingerprint(Node)
=
H(
    NodeDescriptor,
    Fingerprint(Child1),
    Fingerprint(Child2),
    ...
)
```

---

# 216. Merkle advantage

Un subárbol inmutable ya fingerprinted puede reutilizar su fingerprint al calcular el padre.

---

# 217. Child order

Para operaciones order-sensitive:

```text
Child1, Child2
```

se preserva.

---

# 218. Commutative child order

Para operaciones demostrablemente conmutativas y canonicalizables:

```text
sort(
    ChildFingerprints
)
```

podrá utilizarse.

---

# 219. Associative flattening

Para operaciones demostrablemente asociativas:

```text
AND(AND(A,B),C)
```

puede fingerprintarse como:

```text
AND[A,B,C]
```

si el nivel de equivalencia elegido lo permite.

---

# 220. Structural fingerprint exception

El fingerprint `STRUCTURAL` estricto no hará ese flattening si cambia la estructura exacta.

---

# 221. Logical fingerprint

El fingerprint `LOGICAL` sí podrá hacerlo cuando exista proof de equivalencia.

---

# 222. FingerprintContext

```php
final readonly class FingerprintContext
{
    public function __construct(
        public FingerprintDomain $domain,
        public EquivalenceLevel $equivalence,
        public FingerprintVersion $version,
        public SemanticEnvironmentFingerprint $environment,
        public FingerprintPolicy $policy,
    ) {}
}
```

---

# 223. FingerprintPolicy

Podrá controlar:

```text
parameter alpha-renaming
alias alpha-renaming
commutative canonicalization
associative flattening
metadata participation
provenance participation
security participation
schema dependency strategy
```

---

# 224. Policy immutability

Las políticas serán:

```text
immutable
versioned
```

---

# 225. No ad-hoc flags

Evitar APIs como:

```php
fingerprint(
    ignoreAliases: true,
    sortChildren: true,
    ignoreMetadata: true,
    ...
);
```

dispersas por el código.

Se utilizarán perfiles tipados.

---

# 226. Profiles

Ejemplos:

```text
StrictStructuralFingerprintProfile
SemanticQueryFingerprintProfile
OptimizerLogicalFingerprintProfile
CompilationFingerprintProfile
```

---

# 227. Query deduplication service

```php
interface QueryDeduplicator
{
    public function register(
        DeduplicableArtifact $artifact,
        DeduplicationContext $context,
    ): DeduplicationResult;
}
```

---

# 228. Specialized deduplicators

Podrán existir:

```text
ExpressionDeduplicator
PredicateDeduplicator
SubqueryDeduplicator
JoinAlternativeDeduplicator
OptimizationStateDeduplicator
LogicalPlanDeduplicator
```

sobre infraestructura común.

---

# 229. No God Deduplicator

No crear:

```php
DatabaseDeduplicator
```

con cientos de `instanceof`.

---

# 230. Descriptor-driven dispatch

Los node descriptors podrán aportar su estrategia de fingerprint.

---

# 231. Visitor support

Para AST conocido podrá utilizarse:

```text
FingerprintVisitor
```

---

# 232. Extension dispatch

Extensiones utilizarán registry explícito.

---

# 233. Registry freeze

El registry de fingerprint contributors se congelará tras bootstrap.

---

# 234. Dynamic registration

No se permitirá cambiar las reglas de fingerprint a mitad de una operación.

---

# 235. Semantic environment fingerprint

Podrá incluir:

```text
QueryTypeSystemVersion
FunctionRegistryVersion
OperatorRegistryVersion
ExtensionRegistryVersion
SemanticPolicyVersion
```

---

# 236. Cache invalidation

Cambiar cualquiera de esos elementos puede invalidar fingerprints persistentes derivados.

---

# 237. Deduplication and planner

El Planner podrá reutilizar la infraestructura para detectar:

```text
equivalent logical plans
equivalent physical alternatives
```

pero utilizará dominios de fingerprint diferentes.

---

# 238. Physical plan fingerprint

No pertenece directamente a este documento, pero podrá incluir:

```text
physical operators
access paths
physical properties
distribution
ordering
parallelism
```

---

# 239. Logical ≠ physical fingerprint

Nunca compartir el mismo tipo de fingerprint.

---

# 240. Deduplication and compiler

El Compiler podrá usar:

```text
CompilationFingerprint
```

para cachear SQL compilado.

Pero el Query Deduplication System no genera SQL.

---

# 241. Deduplication and execution

El Executor no deberá volver a analizar semantic equivalence.

Recibirá artifacts ya procesados.

---

# 242. Deduplication and ORM

ORM podrá beneficiarse indirectamente de query deduplication.

Pero:

```text
Entity identity
```

pertenece al:

```text
Identity Map
```

y no a este sistema.

---

# 243. Query deduplication ≠ Identity Map

Muy importante:

```text
Query equivalence
≠
Entity identity
```

---

# 244. Testing strategy

Se requerirán:

```text
unit tests
fingerprint golden tests
property tests
alpha-equivalence tests
collision tests
semantic-equivalence tests
negative-equivalence tests
optimizer convergence tests
cross-process determinism tests
persistent-runtime isolation tests
```

---

# 245. Golden fingerprints

Podrán utilizarse para detectar cambios intencionales en versiones.

Pero no deberán impedir evolucionar el algoritmo.

Cambio intencional:

```text
increment FingerprintVersion
```

---

# 246. Property test — determinism

Para artifact `Q`:

```text
Fingerprint(Q)
=
Fingerprint(Q)
```

entre múltiples ejecuciones equivalentes.

---

# 247. Property test — alpha renaming

Si la política permite alpha-equivalence:

```text
renameLocalIds(Q)
```

deberá conservar:

```text
SemanticFingerprint
```

---

# 248. Property test — structural distinction

Para fingerprint estricto:

```text
A AND B
```

y:

```text
B AND A
```

podrán ser distintos si el perfil es estrictamente estructural.

---

# 249. Property test — logical commutativity

Bajo un perfil lógico y operadores puros conmutativos:

```text
A AND B
```

y:

```text
B AND A
```

deberán coincidir.

---

# 250. Negative test — volatile

```text
random() + random()
```

no deberá reducirse a una única evaluación.

---

# 251. Negative test — projection

```text
SELECT a, a
```

deberá conservar output arity 2.

---

# 252. Negative test — UNION ALL

```text
A UNION ALL A
```

no deberá deduplicarse a `A`.

---

# 253. Negative test — correlated subquery

Mismo body textual con distinto outer scope no deberá fusionarse.

---

# 254. Negative test — security provenance

Dos predicates equivalentes pero pertenecientes a security domains incompatibles no deberán fusionarse destructivamente.

---

# 255. Negative test — collation

```text
name COLLATE X
```

y:

```text
name COLLATE Y
```

no deberán tener mismo semantic fingerprint si la collation afecta meaning.

---

# 256. Collision testing

Se deberá poder inyectar un hash deliberadamente pequeño para verificar:

```text
collision bucket
+
deep comparison
```

sin afectar correctness.

---

# 257. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Query/
            └── Optimizer/
                └── Deduplication/
                    ├── Contract/
                    │   ├── QueryDeduplicator.php
                    │   ├── DeduplicableArtifact.php
                    │   ├── Fingerprint.php
                    │   ├── FingerprintFactory.php
                    │   ├── FingerprintWriter.php
                    │   └── FingerprintContributor.php
                    │
                    ├── Fingerprint/
                    │   ├── StructuralFingerprint.php
                    │   ├── NormalizedFingerprint.php
                    │   ├── SemanticFingerprint.php
                    │   ├── LogicalFingerprint.php
                    │   ├── ResultShapeFingerprint.php
                    │   ├── CompilationFingerprint.php
                    │   ├── OptimizationStateFingerprint.php
                    │   ├── FingerprintVersion.php
                    │   ├── FingerprintDomain.php
                    │   └── FingerprintHasher.php
                    │
                    ├── Canonical/
                    │   ├── Canonicalizer.php
                    │   ├── CanonicalIdMapper.php
                    │   ├── CanonicalParameterMapper.php
                    │   ├── CanonicalRelationMapper.php
                    │   ├── CanonicalSymbolMapper.php
                    │   └── CanonicalFingerprintWriter.php
                    │
                    ├── Context/
                    │   ├── FingerprintContext.php
                    │   ├── DeduplicationContext.php
                    │   └── SemanticEnvironmentFingerprint.php
                    │
                    ├── Policy/
                    │   ├── FingerprintPolicy.php
                    │   ├── StrictStructuralFingerprintProfile.php
                    │   ├── SemanticQueryFingerprintProfile.php
                    │   ├── OptimizerLogicalFingerprintProfile.php
                    │   └── CompilationFingerprintProfile.php
                    │
                    ├── Expression/
                    │   ├── ExpressionDeduplicator.php
                    │   └── CommonExpressionCandidate.php
                    │
                    ├── Predicate/
                    │   └── PredicateDeduplicator.php
                    │
                    ├── Subquery/
                    │   └── SubqueryDeduplicator.php
                    │
                    ├── Join/
                    │   └── JoinAlternativeDeduplicator.php
                    │
                    ├── State/
                    │   ├── OptimizationStateDeduplicator.php
                    │   ├── OptimizationStateRegistry.php
                    │   └── OptimizationState.php
                    │
                    ├── Equivalence/
                    │   ├── EquivalenceLevel.php
                    │   ├── EquivalenceClass.php
                    │   ├── EquivalenceClassId.php
                    │   ├── EquivalenceProof.php
                    │   └── EquivalenceRegistry.php
                    │
                    ├── Memo/
                    │   ├── OptimizerMemo.php
                    │   ├── MemoGroup.php
                    │   ├── MemoGroupId.php
                    │   ├── MemoExpression.php
                    │   └── MemoExpressionId.php
                    │
                    ├── Registry/
                    │   └── FingerprintContributorRegistry.php
                    │
                    ├── Budget/
                    │   ├── DeduplicationBudget.php
                    │   └── ProvenanceBudget.php
                    │
                    ├── Diagnostic/
                    │   └── DeduplicationDiagnostic.php
                    │
                    └── Exception/
                        └── DeduplicationException.php
```

---

# 258. Arquitectura conceptual

```text
                    Query Artifact
                          │
                          ▼
                 Equivalence Domain
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
      Structural       Semantic          Logical
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                  Canonicalization
                          │
                          ▼
                  Typed Fingerprint
                          │
                          ▼
                     Fast Hash
                          │
                          ▼
                  Candidate Bucket
                          │
                          ▼
               Collision-Safe Check
                          │
               ┌──────────┴──────────┐
               ▼                     ▼
            Unique               Equivalent
               │                     │
               ▼                     ▼
           Register            Equivalence Class
                                     │
                                     ▼
                               Representative
                                     │
                                     ▼
                              Optimizer / Planner
```

---

# 259. Memo architecture

```text
Transformation Rules
        │
        ▼
Logical Expression
        │
        ▼
Logical Fingerprint
        │
        ▼
┌───────────────────────┐
│   Optimizer Memo      │
├───────────────────────┤
│ Group G1              │
│ ├── Expression E1     │
│ ├── Expression E2     │
│ └── Expression E3     │
│                       │
│ Group G2              │
│ ├── Expression E4     │
│ └── Expression E5     │
└───────────────────────┘
        │
        ▼
Unique Logical Search Space
```

---

# 260. Invariantes arquitectónicos

## DB-DEDUP-001

Identity no será equivalente a structural equality.

## DB-DEDUP-002

Structural equality no será equivalente a semantic equivalence.

## DB-DEDUP-003

Semantic equivalence no será equivalente a runtime equality.

## DB-DEDUP-004

Logical equivalence será un concepto explícito.

## DB-DEDUP-005

Toda deduplicación declarará su nivel de equivalencia.

## DB-DEDUP-006

No existirá una función universal `same()` sin contexto.

## DB-DEDUP-007

Fingerprints serán tipados.

## DB-DEDUP-008

StructuralFingerprint será distinto de SemanticFingerprint.

## DB-DEDUP-009

SemanticFingerprint será distinto de LogicalFingerprint.

## DB-DEDUP-010

LogicalFingerprint será distinto de CompilationFingerprint.

## DB-DEDUP-011

Fingerprint no será sinónimo de hash.

## DB-DEDUP-012

Hash collision no implicará equivalencia.

## DB-DEDUP-013

Collision handling preservará correctness.

## DB-DEDUP-014

Fingerprint algorithms serán versionados.

## DB-DEDUP-015

Fingerprint domains estarán separados.

## DB-DEDUP-016

Structural fingerprint preservará estructura relevante.

## DB-DEDUP-017

Semantic fingerprint utilizará semantic identities resueltas.

## DB-DEDUP-018

Logical fingerprint podrá abstraer diferencias demostrablemente irrelevantes.

## DB-DEDUP-019

Runtime bindings quedarán fuera del semantic fingerprint general.

## DB-DEDUP-020

Compiler placeholders quedarán fuera del query semantic fingerprint.

## DB-DEDUP-021

Compile-time specialization tendrá fingerprint separado cuando sea necesario.

## DB-DEDUP-022

ParameterId será distinto de placeholder.

## DB-DEDUP-023

Alpha-equivalence será explícita.

## DB-DEDUP-024

Alpha-renaming sólo afectará identidades locales renombrables.

## DB-DEDUP-025

Outer symbols no se tratarán como local IDs.

## DB-DEDUP-026

Scope boundaries serán preservadas.

## DB-DEDUP-027

Correlation formará parte de semantic equivalence.

## DB-DEDUP-028

Expression deduplication no implicará shared evaluation.

## DB-DEDUP-029

Volatile expressions no se fusionarán ingenuamente.

## DB-DEDUP-030

Unknown volatility se tratará conservadoramente.

## DB-DEDUP-031

Predicate deduplication respetará SQL 3VL.

## DB-DEDUP-032

Predicate idempotence requerirá purity/evaluation safety.

## DB-DEDUP-033

Commutativity será descriptor-driven.

## DB-DEDUP-034

Associativity será descriptor-driven.

## DB-DEDUP-035

Operands no se ordenarán universalmente.

## DB-DEDUP-036

AST order no será destruido para obtener logical fingerprint.

## DB-DEDUP-037

Duplicate projection expressions no reducirán output arity.

## DB-DEDUP-038

Detection será distinta de removal.

## DB-DEDUP-039

Subquery equivalence incluirá correlation.

## DB-DEDUP-040

Equivalent subqueries no implicarán shared execution.

## DB-DEDUP-041

CTE body equivalence no implicará CTE merging.

## DB-DEDUP-042

Recursive CTE identity será preservada.

## DB-DEDUP-043

CTE materialization intent será considerado.

## DB-DEDUP-044

Security metadata podrá impedir CTE fusion.

## DB-DEDUP-045

Join alternatives utilizarán logical fingerprints.

## DB-DEDUP-046

Diferentes rewrite paths podrán converger en un único state.

## DB-DEDUP-047

Optimization state fingerprint incluirá facts relevantes.

## DB-DEDUP-048

Mismo tree con facts diferentes no será deduplicado si esos facts afectan futuras reglas.

## DB-DEDUP-049

Diagnostics irrelevantes no crearán nuevos logical states.

## DB-DEDUP-050

Rule history no definirá logical equivalence.

## DB-DEDUP-051

Provenance podrá fusionarse sin duplicar states.

## DB-DEDUP-052

Provenance estará bounded.

## DB-DEDUP-053

Optimizer Memo será distinto de generic cache.

## DB-DEDUP-054

MemoGroup representará logical equivalence.

## DB-DEDUP-055

V1 no dependerá de implementar Cascades completo.

## DB-DEDUP-056

La arquitectura permitirá adoptar Memo completo posteriormente.

## DB-DEDUP-057

Fingerprint lookup tendrá collision-safe comparison.

## DB-DEDUP-058

No se usarán memory addresses como stable fingerprints.

## DB-DEDUP-059

No se usarán object hashes como persistent fingerprints.

## DB-DEDUP-060

Persistent fingerprints serán cross-process deterministic.

## DB-DEDUP-061

Canonical serialization tendrá contrato estable.

## DB-DEDUP-062

Canonical serialization será type-tagged.

## DB-DEDUP-063

Canonical serialization evitará concatenation ambiguity.

## DB-DEDUP-064

Metadata participation será explícita.

## DB-DEDUP-065

Debug labels no alterarán semantic equivalence por defecto.

## DB-DEDUP-066

Trace IDs no alterarán query equivalence.

## DB-DEDUP-067

Security semantics participarán cuando sean relevantes.

## DB-DEDUP-068

Schema fingerprints sólo aparecerán en niveles que los necesiten.

## DB-DEDUP-069

Structural fingerprints pre-semantic no dependerán del schema.

## DB-DEDUP-070

Dependency-aware schema fingerprint será preferible a invalidación global cuando sea viable.

## DB-DEDUP-071

Capabilities podrán participar en optimized/compiled fingerprints.

## DB-DEDUP-072

Vendor names no sustituirán capability fingerprints.

## DB-DEDUP-073

Optimizer rules serán versionables.

## DB-DEDUP-074

Optimizer profile podrá formar parte del optimization-result fingerprint.

## DB-DEDUP-075

Budgets no definirán query semantics.

## DB-DEDUP-076

UNION DISTINCT y UNION ALL tendrán distinta deduplication semantics.

## DB-DEDUP-077

Bag semantics serán preservadas.

## DB-DEDUP-078

Aggregate fingerprints incluirán DISTINCT.

## DB-DEDUP-079

Aggregate fingerprints incluirán FILTER.

## DB-DEDUP-080

Aggregate fingerprints incluirán aggregate ordering.

## DB-DEDUP-081

COUNT(*) será distinto de COUNT(expression).

## DB-DEDUP-082

Window fingerprints utilizarán EffectiveWindowSpecification.

## DB-DEDUP-083

EffectiveWindowFrame participará en semantic equivalence.

## DB-DEDUP-084

Raw expressions serán barriers conservadoras.

## DB-DEDUP-085

Raw SQL no será reparsed informalmente para deduplicación.

## DB-DEDUP-086

Extensions deberán proporcionar fingerprint semantics.

## DB-DEDUP-087

Unknown extensions serán deduplication barriers.

## DB-DEDUP-088

Extension semantic versions serán consideradas.

## DB-DEDUP-089

Security provenance no se perderá.

## DB-DEDUP-090

Logical equivalence no implicará identical provenance.

## DB-DEDUP-091

Lineage no se perderá por deduplicación.

## DB-DEDUP-092

Semantic equivalence no implicará identical lineage.

## DB-DEDUP-093

Equivalence classes podrán contener múltiples artifact identities.

## DB-DEDUP-094

Canonical representative será determinista.

## DB-DEDUP-095

Representative selection no mutará artifacts.

## DB-DEDUP-096

Semantic equivalence podrá tener proof.

## DB-DEDUP-097

Logical equivalence podrá tener proof.

## DB-DEDUP-098

Si no puede demostrarse equivalencia se conservarán ambas estructuras.

## DB-DEDUP-099

Query semantic fingerprint incluirá output semantics.

## DB-DEDUP-100

Output arity será observable.

## DB-DEDUP-101

ResultShapeFingerprint será separado.

## DB-DEDUP-102

CompilationFingerprint será separado.

## DB-DEDUP-103

Prepared statement reuse no dependerá de runtime values generales.

## DB-DEDUP-104

Deduplication será bounded.

## DB-DEDUP-105

Budget exhaustion no cambiará query semantics.

## DB-DEDUP-106

Budget exhaustion no convertirá query válida en inválida por defecto.

## DB-DEDUP-107

Memory governance nunca descartará artifacts requeridos para correctness.

## DB-DEDUP-108

Deduplication registries serán operation-scoped.

## DB-DEDUP-109

No habrá global mutable seen-query registry.

## DB-DEDUP-110

Cross-request reuse requerirá cache explícito.

## DB-DEDUP-111

FrankenPHP workers no compartirán optimizer memo mutable.

## DB-DEDUP-112

RoadRunner podrá reutilizar la misma arquitectura.

## DB-DEDUP-113

OpenSwoole podrá reutilizar la misma arquitectura.

## DB-DEDUP-114

Fingerprint generation será determinista.

## DB-DEDUP-115

Fingerprint generation será locale-independent.

## DB-DEDUP-116

Identifier canonicalization respetará platform semantics.

## DB-DEDUP-117

No se aplicará lowercase universal a identifiers.

## DB-DEDUP-118

Semantic fingerprints preferirán resolved IDs.

## DB-DEDUP-119

Collation participará cuando afecte meaning.

## DB-DEDUP-120

Domain identity será preservada.

## DB-DEDUP-121

Explicit casts serán preservados.

## DB-DEDUP-122

Resolved coercions participarán cuando afecten semantics.

## DB-DEDUP-123

FunctionSemanticId será preferido al function name.

## DB-DEDUP-124

Function descriptor version podrá participar.

## DB-DEDUP-125

Operator semantic identity será explícita.

## DB-DEDUP-126

Deduplication traces serán explicables.

## DB-DEDUP-127

Sensitive runtime values no aparecerán en traces.

## DB-DEDUP-128

Telemetry no expondrá secretos mediante fingerprints.

## DB-DEDUP-129

Fingerprinting podrá utilizar composición tipo Merkle.

## DB-DEDUP-130

Order-sensitive children conservarán orden.

## DB-DEDUP-131

Commutative canonicalization requerirá proof/descriptor.

## DB-DEDUP-132

Associative flattening dependerá del equivalence level.

## DB-DEDUP-133

Strict structural fingerprints no realizarán logical rewrites.

## DB-DEDUP-134

Fingerprint policies serán tipadas.

## DB-DEDUP-135

Fingerprint policies serán inmutables.

## DB-DEDUP-136

No se utilizarán colecciones arbitrarias de flags dispersos.

## DB-DEDUP-137

No existirá God Deduplicator.

## DB-DEDUP-138

Specialized deduplicators compartirán infraestructura común.

## DB-DEDUP-139

Extension registries se congelarán después de bootstrap.

## DB-DEDUP-140

Fingerprint semantics no cambiarán durante una operación.

## DB-DEDUP-141

Planner podrá reutilizar infraestructura con otro fingerprint domain.

## DB-DEDUP-142

Logical y physical fingerprints permanecerán separados.

## DB-DEDUP-143

Compiler podrá consumir CompilationFingerprint.

## DB-DEDUP-144

Deduplication System nunca generará SQL.

## DB-DEDUP-145

Executor no reinterpretará semantic equivalence.

## DB-DEDUP-146

Query deduplication será distinta de ORM Identity Map.

## DB-DEDUP-147

Testing verificará alpha-equivalence.

## DB-DEDUP-148

Testing verificará collision safety.

## DB-DEDUP-149

Testing verificará cross-process determinism.

## DB-DEDUP-150

Correctness tendrá prioridad absoluta sobre deduplication ratio.

---

# 261. Anti-patterns

## 261.1 Un único query hash

```php
$query->hash();
```

usado para:

```text
optimizer
compiler
result cache
planner
```

**Rechazado.**

---

## 261.2 Hash = equivalencia

```text
if ($hashA === $hashB) {
    return true;
}
```

**Rechazado.**

---

## 261.3 Deduplicar por SQL string

**Rechazado.**

El SQL todavía puede no existir y además mezcla Compiler con Optimizer.

---

## 261.4 Deduplicar parámetros por placeholder

**Rechazado.**

---

## 261.5 Fusionar funciones volátiles

**Rechazado.**

---

## 261.6 Eliminar proyecciones repetidas

```sql
SELECT id, id
```

→

```sql
SELECT id
```

**Rechazado.**

---

## 261.7 Fusionar subqueries sólo porque su texto coincide

**Rechazado.**

---

## 261.8 Ignorar correlation scope

**Rechazado.**

---

## 261.9 Fusionar CTEs por body textual

**Rechazado.**

---

## 261.10 Deduplicar `UNION ALL`

como si fuera teoría clásica de conjuntos.

**Rechazado.**

---

## 261.11 Ignorar collation

**Rechazado.**

---

## 261.12 Ignorar domain identity

**Rechazado.**

---

## 261.13 Ignorar security provenance

**Rechazado.**

---

## 261.14 Utilizar `spl_object_hash()`

como fingerprint persistente.

**Rechazado.**

---

## 261.15 Utilizar `serialize()` como contrato canónico

**Rechazado.**

---

## 261.16 Estado global `seen`

**Rechazado.**

---

## 261.17 Memo compartido entre requests

sin cache/versionado explícito.

**Rechazado.**

---

# 262. Decisión arquitectónica final

VoltStack implementará `Query Deduplication System` como infraestructura:

```text
typed
multi-level
canonical
semantic-aware
scope-aware
correlation-aware
type-aware
domain-aware
collation-aware
volatility-aware
security-aware
lineage-preserving
collision-safe
versioned
bounded
deterministic
explainable
persistent-runtime safe
```

La regla central será:

> **VoltStack no preguntará simplemente si dos consultas “son iguales”. Siempre definirá primero qué clase de igualdad necesita demostrar y sólo entonces aplicará la canonicalización, fingerprint y prueba de equivalencia correspondiente.**

---

# 263. Resultado arquitectónico

```text
Query Structure
      │
      ├───────────────┐
      ▼               ▼
Structural        Semantic Resolution
Fingerprint            │
                       ▼
               Semantic Fingerprint
                       │
                       ▼
                 Query Optimizer
                       │
              ┌────────┴─────────┐
              ▼                  ▼
       Rewrite States       Join Alternatives
              │                  │
              └────────┬─────────┘
                       ▼
               Logical Fingerprint
                       │
                       ▼
               Deduplication/Memo
                       │
                       ▼
              Unique Search Space
                       │
                       ▼
                  Query Planner
```

---

# 264. Relación con documentos anteriores

```text
25 DATABASE_QUERY_AST_SYSTEM
        │
        ▼
33 DATABASE_QUERY_NORMALIZATION_SYSTEM
        │
        ▼
35-42 Semantic Query Engine
        │
        ▼
55 DATABASE_QUERY_OPTIMIZER_ARCHITECTURE
        │
        ├── 56 Query Rewrite
        ├── 57 Optimization Rules
        ├── 58 Predicate Optimization
        ├── 59 Join Optimization
        └── 60 Query Deduplication
```

La deduplicación se convierte así en una infraestructura fundamental para controlar el crecimiento del espacio de búsqueda del Optimizer.

---

# 265. Bloque actual

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md              ← actual
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 266. Siguiente documento

```text
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
```

El siguiente documento deberá definir la infraestructura mediante la cual VoltStack podrá proporcionar al Optimizer y posteriormente al Planner información orientativa sobre el costo relativo de diferentes estructuras sin mezclar todavía esa información con el modelo físico completo de ejecución.

Deberá distinguir explícitamente:

```text
Logical Fact
≠
Cardinality Estimate
≠
Selectivity Estimate
≠
Cost Hint
≠
Statistics
≠
Physical Cost
≠
Planner Cost
```

y cubrir conceptos como:

```text
QueryCostHint
RelationSizeHint
CardinalityHint
SelectivityHint
JoinPriorityHint
PredicateSelectivityHint
IndexAvailabilityHint
DataDistributionHint
OrderingAvailabilityHint
LocalityHint
NetworkCostHint
MemoryPressureHint

HintSource
HintConfidence
HintScope
HintStrength

PREFER
AVOID
REQUIRE
PROHIBIT

Static Hints
Schema-Derived Hints
Statistics-Derived Hints
Application Hints
Extension Hints

Hint Validation
Hint Conflicts
Hint Precedence
Hint Merging

Stale Hints
Statistics Versioning
Hint Fingerprints

Security Boundaries
Tenant Isolation
Persistent Runtime Safety

Optimizer Integration
Planner Integration
Explainability
Telemetry
```

El principio central deberá ser:

```text
Cost Hint
=
Evidence / Preference

not

Physical Plan Decision
```