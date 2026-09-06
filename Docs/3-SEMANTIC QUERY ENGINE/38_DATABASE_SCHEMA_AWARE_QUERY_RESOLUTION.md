# 38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md

# VoltStack Quantum Database
## Schema-Aware Query Resolution System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 38 — Schema-Aware Query Resolution System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Layer  
**Versión:** 1.0

---

# 1. Propósito

El **Schema-Aware Query Resolution System** conecta las referencias semánticas de una consulta con una representación estructurada y read-only del esquema de base de datos.

Su responsabilidad principal es transformar información como:

```text
RelationReference("users")
```

en información semántica enriquecida:

```text
RelationReference("users")
        │
        ▼
RelationSymbol R1
        │
        ▼
SchemaRelationDescriptor
        │
        ├── id
        ├── email
        ├── status
        ├── created_at
        └── ...
```

permitiendo posteriormente resolver:

```text
u.email
```

como:

```text
ColumnSymbol C2
        │
        ▼
SchemaColumnDescriptor(users.email)
        │
        ├── type
        ├── nullability
        ├── constraints
        └── capabilities
```

---

# 2. Problema que resuelve

El Symbol Resolution System puede determinar que:

```text
u
```

representa una relación.

Pero todavía necesita conocer:

```text
¿Qué relación representa?

¿Qué columnas posee?

¿Qué tipo tiene cada columna?

¿Puede ser NULL?

¿Forma parte de una clave?

¿Qué constraints posee?

¿Qué relaciones estructurales existen?

¿Qué operaciones soporta?
```

Estas preguntas requieren información del schema.

---

# 3. Regla maestra

> El Query Engine deberá consumir una vista estructurada, immutable y versionable del schema; nunca deberá descubrir el schema abriendo conexiones durante el análisis semántico.

---

# 4. Separación fundamental

VoltStack mantendrá:

```text
Physical Database
≠
Schema Introspection
≠
Schema Model
≠
Schema View
≠
Schema Resolution
≠
Query Symbol
≠
ORM Metadata
```

---

# 5. Arquitectura general

```text
                    Physical Database
                           │
                           │ introspection
                           ▼
                  Schema Introspection
                           │
                           ▼
                     Schema Model
                           │
                           ▼
                      Schema View
                           │
                           ▼
Query AST ──► Symbol Resolution ──► Schema-Aware Resolution
                                      │
                                      ▼
                              Semantic Relation
                                      │
                                      ▼
                               Semantic Column
                                      │
                                      ▼
                              Semantic Analysis
```

El camino crítico del Query Engine comienza en:

```text
Schema View
```

no en:

```text
Physical Database
```

---

# 6. SchemaView

La abstracción principal será:

```text
SchemaView
```

Una vista read-only del schema relevante para una operación de análisis.

Conceptualmente:

```php
interface SchemaViewInterface
{
    public function version(): SchemaVersion;

    public function resolveRelation(
        SchemaRelationReference $reference,
    ): SchemaRelationResolutionResult;
}
```

---

# 7. SchemaView no es Schema Builder

`SchemaView` sólo consulta información.

No podrá:

```text
create table
alter column
drop index
run migration
execute DDL
```

---

# 8. SchemaView no es introspector

No tendrá métodos como:

```php
$pdo->query(...);
```

ni abrirá conexiones.

La introspección pertenece al futuro:

```text
96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md
```

---

# 9. SchemaView como snapshot

Idealmente:

```text
SchemaView
=
Immutable Schema Snapshot
+
Resolution Rules
+
Version Identity
```

Esto permite análisis determinista.

---

# 10. Offline semantic analysis

Gracias a esta separación podrá ejecutarse:

```text
Query
  │
  ▼
Semantic Analysis
```

sin base de datos activa.

Ejemplos:

```text
CI
static analysis
query compilation
IDE tooling
migration analysis
tests
build pipeline
```

---

# 11. SchemaCatalogView

Para sistemas con múltiples catalogs/databases podrá existir:

```text
SchemaCatalogView
```

Conceptualmente:

```text
Catalog
└── Schema
    └── Relation
        └── Column
```

---

# 12. Jerarquía lógica

VoltStack modelará conceptualmente:

```text
Catalog
   │
   ▼
Schema Namespace
   │
   ▼
Relation
   │
   ▼
Column
```

pero sin asumir que todos los motores implementan exactamente estos niveles.

---

# 13. Portabilidad

SQLite, MySQL/MariaDB y PostgreSQL poseen conceptos diferentes alrededor de:

```text
database
catalog
schema
namespace
```

Por ello VoltStack no impondrá una equivalencia física universal.

---

# 14. Logical schema namespace

El Query Engine utilizará una representación lógica:

```text
SchemaNamespace
```

que la Platform podrá mapear correctamente.

---

# 15. SchemaRelationDescriptor

Una relación estructural podrá representarse mediante:

```text
SchemaRelationDescriptor
```

---

# 16. Estructura conceptual

```text
SchemaRelationDescriptor
├── relationId
├── qualifiedName
├── relationKind
├── columns
├── keys
├── indexes
├── constraints
├── capabilities
├── metadata
└── schemaVersion
```

---

# 17. RelationId

Cada relación del snapshot tendrá:

```text
SchemaRelationId
```

Su identidad no deberá depender simplemente del nombre.

---

# 18. Ejemplo

```text
SchemaRelationId
└── public.users
```

podrá tener una identidad interna estable como:

```text
REL-42
```

dentro de una versión concreta del schema.

---

# 19. Relation kinds

Inicialmente:

```text
TABLE
VIEW
MATERIALIZED_VIEW
VIRTUAL_TABLE
SYSTEM_RELATION
TEMPORARY_RELATION
EXTENSION_RELATION
UNKNOWN
```

La disponibilidad dependerá de capacidades.

---

# 20. Query relation vs schema relation

Debe mantenerse:

```text
RelationSymbol
≠
SchemaRelationDescriptor
```

Ejemplo:

```sql
FROM users u
```

produce:

```text
RelationSymbol
├── name = u
└── schemaRelation = public.users
```

---

# 21. Múltiples symbols sobre misma relación

Una query puede contener:

```sql
FROM users author
JOIN users editor ON ...
```

Entonces:

```text
RelationSymbol R1 → users
RelationSymbol R2 → users
```

pero:

```text
R1 ≠ R2
```

aunque ambos apunten al mismo:

```text
SchemaRelationDescriptor(users)
```

---

# 22. Importancia

Esto permite self joins correctamente.

---

# 23. SchemaColumnDescriptor

Cada columna se representará mediante:

```text
SchemaColumnDescriptor
```

---

# 24. Estructura

```text
SchemaColumnDescriptor
├── columnId
├── relationId
├── name
├── ordinal
├── semanticType
├── physicalTypeMetadata
├── nullability
├── generated
├── defaultMetadata
├── keyMembership
├── capabilities
└── metadata
```

---

# 25. ColumnId

Cada columna tendrá:

```text
SchemaColumnId
```

independiente del `ColumnSymbolId`.

---

# 26. Separación

```text
SchemaColumnId
≠
ColumnSymbolId
```

---

# 27. Ejemplo

Para:

```sql
SELECT a.id, b.id
FROM users a
JOIN users b ON ...
```

puede existir:

```text
SchemaColumnId(users.id) = SC17
```

pero:

```text
ColumnSymbol(a.id) = C1
ColumnSymbol(b.id) = C9
```

---

# 28. Schema column → semantic column

La resolución genera:

```text
ResolvedSchemaColumn
├── columnSymbolId
├── relationSymbolId
├── schemaColumnId
└── descriptor
```

---

# 29. SchemaResolutionTable

Los resultados se almacenarán en:

```text
SchemaResolutionTable
```

---

# 30. Ejemplo

```text
RelationSymbol R1
→ SchemaRelationId SR10

ColumnSymbol C1
→ SchemaColumnId SC20

ColumnSymbol C2
→ SchemaColumnId SC21
```

---

# 31. No mutar symbols

No será necesario convertir:

```text
RelationSymbol
```

en un objeto mutable con:

```text
$symbol->schemaTable = ...
```

La asociación estará en una tabla semántica separada.

---

# 32. Regla

```text
Symbol Table
≠
Schema Resolution Table
```

---

# 33. Ventajas

Permite:

- símbolos immutable;
- análisis con diferentes schema snapshots;
- caching independiente;
- análisis concurrente;
- mejor invalidación;
- testing aislado.

---

# 34. Relation resolution

Una referencia:

```text
users
```

podrá pasar por:

```text
RelationReference
       │
       ▼
Query-local resolution
       │
       ├── CTE
       ├── derived relation
       └── local relation
       │
       ▼
Schema fallback when required
       │
       ▼
SchemaView
       │
       ▼
SchemaRelationDescriptor
```

---

# 35. Local relation precedence

Si existe:

```sql
WITH users AS (...)
SELECT * FROM users
```

`users` puede resolver al CTE antes que a la tabla física.

---

# 36. Schema lookup sólo cuando corresponde

No toda `RelationSymbol` necesita:

```text
SchemaRelationDescriptor
```

Por ejemplo:

```text
CTE
Derived Relation
VALUES Relation
Table Function
Set Operation
```

pueden producir su propio output semántico.

---

# 37. RelationSourceKind

VoltStack distinguirá:

```text
SCHEMA_RELATION
CTE
DERIVED_QUERY
VALUES
TABLE_FUNCTION
SET_OPERATION
EXTENSION
```

---

# 38. RelationOutput

Toda relación visible al Query Engine deberá finalmente producir:

```text
RelationOutput
```

---

# 39. RelationOutput structure

```text
RelationOutput
├── relationSymbolId
├── columns[]
├── uniquenessMetadata
├── orderingMetadata?
├── provenance
└── capabilities
```

---

# 40. Schema relation output

Para una tabla física:

```text
SchemaRelationDescriptor
        │
        ▼
RelationOutputFactory
        │
        ▼
RelationOutput
```

---

# 41. Derived relation output

Para:

```sql
SELECT id, email
FROM users
```

usado como subquery:

```text
Projection Analysis
        │
        ▼
Derived RelationOutput
```

No requiere una tabla física equivalente.

---

# 42. Unified abstraction

Esto permite:

```text
Physical Table ─────────┐
CTE ────────────────────┤
Derived Query ──────────┤
VALUES ─────────────────┤
Table Function ─────────┼──► RelationOutput
Set Operation ──────────┤
Extension Relation ─────┘
```

---

# 43. Column resolution

Una referencia:

```text
u.email
```

se resolverá:

```text
ColumnReference
      │
      ▼
RelationSymbol(u)
      │
      ▼
RelationOutput(u)
      │
      ▼
Column candidate "email"
      │
      ▼
ColumnSymbol
```

---

# 44. Schema-aware column

Cuando `u` proviene de tabla física:

```text
ColumnSymbol
      │
      ▼
SchemaColumnDescriptor
```

---

# 45. Derived column

Cuando `u` proviene de subquery:

```text
ColumnSymbol
      │
      ▼
DerivedOutputColumnDescriptor
```

No existirá necesariamente `SchemaColumnDescriptor`.

---

# 46. SemanticOutputColumn

Para unificar ambos mundos podrá existir:

```text
SemanticOutputColumn
```

---

# 47. Estructura

```text
SemanticOutputColumn
├── outputColumnId
├── name
├── semanticType
├── nullability
├── provenance
├── sourceSymbols
├── schemaOrigin?
└── capabilities
```

---

# 48. Source provenance

Ejemplo:

```sql
SELECT price * quantity AS total
```

produce:

```text
SemanticOutputColumn(total)
├── source = expression
└── sourceSymbols
    ├── price
    └── quantity
```

---

# 49. Direct schema provenance

```sql
SELECT email
FROM users
```

puede producir:

```text
SemanticOutputColumn(email)
└── schemaOrigin = users.email
```

---

# 50. Multi-source provenance

Ejemplo:

```sql
SELECT COALESCE(primary_email, backup_email) AS email
```

tendrá múltiples source symbols.

---

# 51. Type information

El schema podrá aportar:

```text
column type
```

como evidencia al:

```text
Query Type System
```

---

# 52. Type pipeline

```text
SchemaColumnDescriptor
        │
        ▼
Schema Type Metadata
        │
        ▼
Query Type Resolver
        │
        ▼
Resolved Query Type
```

---

# 53. Schema type no equivale a query type

Debe mantenerse:

```text
Physical Column Type
≠
Query Semantic Type
```

Ejemplo:

```text
BIGINT
```

puede mapear semánticamente a:

```text
core.integer
```

o incluso:

```text
app.user_id
```

cuando exista metadata adicional.

---

# 54. Mapping boundary

La conversión será responsabilidad del sistema de tipos y mappings.

No del Symbol Resolver.

---

# 55. Nullability

El schema puede aportar:

```text
NOT NULL
NULLABLE
UNKNOWN
```

---

# 56. Query nullability

Pero:

```text
Schema Nullability
≠
Expression Result Nullability
```

---

# 57. Ejemplo outer join

Si:

```text
orders.id
```

es `NOT NULL` en schema:

```sql
users
LEFT JOIN orders
```

el resultado:

```text
orders.id
```

puede ser nullable dentro de la query.

---

# 58. Regla crítica

> La nullability del schema es evidencia inicial; la nullability semántica final depende de la estructura de la consulta.

---

# 59. Nullability transformation

```text
Schema Nullability
        +
Join Semantics
        +
Expression Semantics
        +
Predicate Refinement
        │
        ▼
Query Nullability
```

---

# 60. Primary keys

El schema podrá exponer:

```text
PrimaryKeyDescriptor
```

---

# 61. Structure

```text
PrimaryKeyDescriptor
├── keyId
├── relationId
├── columns[]
└── metadata
```

---

# 62. Composite primary keys

No asumir:

```text
one PK = one column
```

Ejemplo:

```text
PRIMARY KEY (tenant_id, user_id)
```

---

# 63. Unique keys

Se modelarán:

```text
UniqueConstraintDescriptor
```

con múltiples columnas cuando corresponda.

---

# 64. Unique metadata usefulness

Puede utilizarse posteriormente para:

- cardinality reasoning;
- join reasoning;
- optimization;
- identity inference;
- ORM hydration;
- duplicate elimination analysis.

---

# 65. Foreign keys

El schema podrá exponer:

```text
ForeignKeyDescriptor
```

---

# 66. Structure

```text
ForeignKeyDescriptor
├── sourceRelation
├── sourceColumns
├── targetRelation
├── targetColumns
├── updateAction
├── deleteAction
└── metadata
```

---

# 67. Foreign key ≠ ORM relationship

Debe mantenerse:

```text
Foreign Key
≠
ORM Relationship
```

Una relación ORM puede existir sin FK física.

Una FK puede existir sin relationship ORM.

---

# 68. Query semantic relationship

El Semantic Engine podrá derivar información a partir de ambos sistemas, pero sin confundirlos.

---

# 69. Constraints

El schema podrá exponer:

```text
CheckConstraintDescriptor
UniqueConstraintDescriptor
PrimaryKeyDescriptor
ForeignKeyDescriptor
NotNullConstraint
```

---

# 70. Constraint knowledge

Esta información puede alimentar posteriormente:

```text
Constraint Analysis
```

en:

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

---

# 71. Conservative reasoning

VoltStack no deberá asumir más de lo garantizado por el schema snapshot.

---

# 72. Example

Si existe:

```text
CHECK (age >= 0)
```

el Semantic Engine podrá conocer esa constraint si está representada estructuralmente.

Pero no deberá intentar interpretar arbitrary vendor SQL si no existe parser compatible.

---

# 73. Constraint representation

Preferir:

```text
structured constraint model
```

sobre:

```text
raw SQL string
```

cuando sea posible.

---

# 74. Raw constraints

Si una constraint sólo está disponible como SQL nativo:

```text
OpaqueConstraintDescriptor
```

podrá preservarla sin fingir comprensión semántica.

---

# 75. Indexes

Los índices también podrán estar disponibles.

Pero:

```text
Index Metadata
```

no debe confundirse con:

```text
Semantic Constraint
```

---

# 76. Index usefulness

Posteriormente podrá ayudar al:

```text
Query Planner
```

pero no modifica por sí mismo el significado lógico de la consulta.

---

# 77. Unique index

Un índice unique puede aportar información de unicidad cuando la Platform confirme su semántica.

---

# 78. Partial indexes

Los partial indexes requieren conservar su predicate cuando sea estructuralmente conocido.

---

# 79. Expression indexes

De manera similar:

```text
expression index
```

no equivale a columna ordinaria.

---

# 80. Generated columns

`SchemaColumnDescriptor` podrá indicar:

```text
generated = true
```

y opcionalmente:

```text
generationExpression
```

cuando sea estructuralmente conocida.

---

# 81. Identity / auto increment

La generación automática deberá modelarse mediante capacidades/metadata.

No simplemente:

```text
autoIncrement = true
```

como concepto universal.

---

# 82. ColumnGenerationStrategy

Ejemplos:

```text
NONE
IDENTITY
SEQUENCE
AUTO_INCREMENT
GENERATED_EXPRESSION
DEFAULT_EXPRESSION
PLATFORM_SPECIFIC
```

---

# 83. Defaults

Un default podrá representarse como:

```text
DefaultValueDescriptor
```

---

# 84. Default kinds

```text
NONE
LITERAL
EXPRESSION
SEQUENCE
IDENTITY
PLATFORM_SPECIFIC
OPAQUE
```

---

# 85. Defaults no afectan SELECT directamente

Pero serán relevantes para:

```text
INSERT semantic validation
Persistence Engine
Schema tools
Migrations
```

---

# 86. Insert column resolution

Ejemplo:

```sql
INSERT INTO users (email, status)
VALUES (...)
```

requiere:

```text
users
→ SchemaRelationDescriptor

email
→ SchemaColumnDescriptor

status
→ SchemaColumnDescriptor
```

---

# 87. Insert validation metadata

El schema podrá aportar:

```text
required column
generated column
nullable column
defaulted column
writable column
```

---

# 88. Writable column

Debe existir un concepto:

```text
ColumnWriteCapability
```

---

# 89. Example

Una generated column puede ser:

```text
selectable = true
insertable = false
updatable = false
```

---

# 90. Update resolution

```sql
UPDATE users
SET email = :email
```

requiere resolver:

```text
target relation
target column
column write capability
```

---

# 91. Delete resolution

`DELETE` necesita resolver principalmente:

```text
target relation
target relation capabilities
predicates
returning outputs
```

cuando aplique.

---

# 92. DML target capabilities

Una relation podrá exponer:

```text
READ
INSERT
UPDATE
DELETE
RETURNING
LOCK
```

como capacidades semánticas/estructurales efectivas.

---

# 93. View writability

No asumir que:

```text
VIEW
```

es siempre read-only o writable.

Debe provenir de capabilities.

---

# 94. Temporary relations

Una temporary relation puede pertenecer a runtime state físico.

Esto requiere cuidado.

---

# 95. Core V1 policy

El Schema-Aware Semantic Analyzer deberá trabajar principalmente con:

```text
stable schema snapshots
```

Las tablas temporales runtime-specific sólo podrán analizarse cuando exista un descriptor explícito suministrado al contexto.

---

# 96. No connection introspection fallback

Nunca:

```text
schema missing
→ open connection
→ DESCRIBE table
```

desde Semantic Analysis.

---

# 97. Missing schema information

Debe producir un resultado explícito.

---

# 98. SchemaKnowledgeState

Ejemplos:

```text
KNOWN
PARTIAL
UNKNOWN
OPAQUE
```

---

# 99. Unknown schema relation

Si la política requiere schema conocido:

```text
UnknownSchemaRelationException
```

---

# 100. Dynamic schema mode

VoltStack podrá soportar un modo más permisivo para:

```text
raw SQL
dynamic external schemas
legacy systems
```

pero deberá ser explícito.

---

# 101. SchemaResolutionMode

```text
STRICT
PERMISSIVE
OPAQUE_ALLOWED
```

---

# 102. STRICT

En modo:

```text
STRICT
```

una relación desconocida produce error semántico.

---

# 103. PERMISSIVE

Puede producir:

```text
UnresolvedExternalRelationDescriptor
```

pero limita:

```text
type inference
column validation
optimization
compile-time diagnostics
```

---

# 104. OPAQUE_ALLOWED

Reservado para escape hatches/extensions donde VoltStack acepta no comprender completamente la estructura.

---

# 105. No fake schema

Nunca inventar:

```text
unknown column → StringType
```

como fallback silencioso.

---

# 106. Unknown type

Si la columna existe pero su tipo es desconocido:

```text
UnknownQueryType
```

es preferible a adivinar.

---

# 107. Schema reference

Una relación podrá utilizar:

```text
SchemaRelationReference
```

---

# 108. Structure

```text
SchemaRelationReference
├── catalog?
├── schema?
├── relationName
└── qualificationMode
```

---

# 109. Qualification modes

```text
UNQUALIFIED
SCHEMA_QUALIFIED
CATALOG_QUALIFIED
FULLY_QUALIFIED
PLATFORM_SPECIFIC
```

---

# 110. Search path

Una referencia no calificada:

```text
users
```

puede depender de:

```text
schema search path
```

---

# 111. SearchPathSnapshot

El Query Processing Context podrá proporcionar:

```text
SchemaSearchPathSnapshot
```

---

# 112. Example PostgreSQL

Conceptualmente:

```text
search_path:
1. app
2. public
```

Entonces:

```text
users
```

puede resolver:

```text
app.users
```

antes de:

```text
public.users
```

---

# 113. No live session lookup

No leer:

```text
SHOW search_path
```

durante semantic analysis.

El valor efectivo deberá estar ya presente en el snapshot/contexto.

---

# 114. MySQL/MariaDB logical database

La Platform podrá interpretar correctamente la qualification según sus propias reglas.

---

# 115. SQLite

SQLite puede utilizar namespaces como:

```text
main
temp
attached database
```

pero el Query Engine seguirá utilizando abstracciones estructuradas.

---

# 116. Qualification policy

La lógica vendor-specific estará encapsulada en:

```text
SchemaQualificationPolicy
```

o servicios Platform equivalentes.

---

# 117. No vendor branching

Nunca dispersar:

```php
if ($platform === 'pgsql') {
    ...
}
```

en resolvers semánticos.

---

# 118. Relation candidate set

La búsqueda podrá producir:

```text
SchemaRelationCandidateSet
```

---

# 119. Resolution states

```text
RESOLVED
NOT_FOUND
AMBIGUOUS
PARTIAL
OPAQUE
```

---

# 120. Ambiguous relation

Si la qualification/search policy produce múltiples candidatos igualmente válidos:

```text
AMBIGUOUS
```

---

# 121. No arbitrary candidate selection

Nunca:

```text
first matching table wins
```

salvo que el search path defina formalmente esa precedencia.

---

# 122. Search path precedence

Cuando exista precedencia explícita:

```text
first visible relation according to search path
```

sí puede ser resolución determinista.

---

# 123. Schema aliases

VoltStack no confundirá:

```text
query relation alias
```

con:

```text
schema namespace
```

---

# 124. Example

```text
app.users u
```

contiene:

```text
schema = app
relation = users
query alias = u
```

tres conceptos distintos.

---

# 125. Schema snapshot identity

Todo `SchemaView` deberá tener:

```text
SchemaSnapshotId
```

o identidad/version equivalente.

---

# 126. SchemaVersion

Conceptualmente:

```text
SchemaVersion
├── logicalVersion
├── fingerprint
└── provenance
```

---

# 127. Version source

Puede provenir de:

```text
migration state
generated schema metadata
introspection snapshot
build artifact
explicit application schema
test schema
```

---

# 128. Fingerprinting

El fingerprint deberá depender de información estructural relevante.

---

# 129. Schema fingerprint

Conceptualmente:

```text
SchemaSemanticFingerprint
=
Relations
+
Columns
+
Types
+
Nullability
+
Keys
+
Relevant Constraints
+
Capabilities
+
Qualification Semantics
```

---

# 130. Physical irrelevant metadata

No toda metadata física debe invalidar semantic artifacts.

Ejemplo:

```text
table comment changed
```

normalmente no debería invalidar una query compilada.

---

# 131. Fingerprint levels

Podrán existir:

```text
SchemaIdentityFingerprint
SchemaSemanticFingerprint
SchemaPlanningFingerprint
SchemaMigrationFingerprint
```

---

# 132. Semantic fingerprint

Incluye sólo información capaz de modificar:

```text
resolution
types
constraints
query semantics
```

---

# 133. Planning fingerprint

Puede añadir:

```text
indexes
statistics categories
partition metadata
```

cuando el framework planner los utilice.

---

# 134. No runtime statistics here

Las estadísticas dinámicas pertenecen a otro nivel.

No deben mezclarse con:

```text
SchemaSemanticFingerprint
```

---

# 135. Snapshot immutability

Un `SchemaView` publicado será immutable.

---

# 136. Schema change

Si ocurre migration:

```text
SchemaView V1
```

no se mutará a V2.

Se construirá:

```text
SchemaView V2
```

---

# 137. Benefits

Esto evita:

```text
Query A analyzing schema
while migration mutates same metadata object
```

---

# 138. Persistent runtime

En FrankenPHP:

```text
Worker
├── SchemaView V42
├── Request A
├── Request B
└── Request C
```

puede compartir V42 mientras sea immutable.

---

# 139. Schema refresh

Cuando cambie el schema:

```text
V42
   │
   ▼
build V43
   │
   ▼
atomic publication
```

---

# 140. No in-place mutation

Nunca:

```text
$schemaView->tables['users']->columns[] = ...
```

sobre un snapshot compartido.

---

# 141. Request consistency

Una operación de análisis deberá utilizar un único snapshot coherente.

---

# 142. Rule

```text
one semantic analysis
→ one SchemaSnapshot identity
```

salvo arquitectura explícita avanzada futura.

---

# 143. No mid-analysis refresh

Si aparece V43 mientras Query A usa V42:

```text
Query A continues with V42
```

---

# 144. New analysis

La siguiente operación podrá usar:

```text
V43
```

---

# 145. QueryProcessingContext integration

El contexto podrá contener:

```text
SchemaViewInterface
```

como referencia immutable/read-only.

---

# 146. Context composition

```text
QueryProcessingContext
├── TargetEnvironment
├── CapabilitySnapshot
├── PolicySnapshot
├── TypeRegistry
├── FunctionRegistry
├── SchemaView
└── ProcessingBudget
```

---

# 147. SchemaView optionality

No todas las consultas requerirán schema.

Ejemplo:

```sql
SELECT 1
```

puede analizarse sin `SchemaView`.

---

# 148. Schema requirement

El analyzer podrá declarar:

```text
SchemaKnowledgeRequirement
```

---

# 149. Levels

```text
NONE
OPTIONAL
REQUIRED
FULL
```

---

# 150. Required knowledge

Una query como:

```text
SELECT unknown_column FROM users
```

en modo strict necesita resolver schema.

---

# 151. Raw expression

Una `RawExpression` puede reducir el conocimiento requerido, pero también reduce garantías.

---

# 152. Schema resolution coordinator

El componente principal será:

```text
SchemaAwareQueryResolver
```

---

# 153. Responsibilities

Debe:

- resolver relaciones físicas;
- construir relation outputs físicos;
- resolver columnas físicas;
- asociar symbols con schema descriptors;
- exponer type/nullability/key/constraint evidence;
- registrar provenance;
- respetar qualification/search path;
- producir diagnostics.

---

# 154. Non-responsibilities

No debe:

- abrir conexiones;
- ejecutar SQL;
- introspectar DB;
- inferir tipos completos de expressions;
- optimizar queries;
- elegir índices;
- ejecutar migrations;
- hidratar entities.

---

# 155. Relation resolver

Podrá existir:

```text
SchemaRelationResolver
```

---

# 156. Column resolver

Y:

```text
SchemaColumnResolver
```

---

# 157. RelationOutputFactory

Para convertir schema relations en outputs semánticos:

```text
SchemaRelationOutputFactory
```

---

# 158. Constraint provider

Podrá exponer:

```text
SchemaConstraintView
```

a fases posteriores.

---

# 159. Key metadata provider

```text
SchemaKeyView
```

podrá proporcionar:

```text
primary keys
unique keys
foreign keys
```

sin exponer el modelo interno completo.

---

# 160. Principle of minimum knowledge

Las fases deberán depender del view mínimo necesario.

Ejemplo:

```text
Type Inference
```

no necesita todo el:

```text
SchemaModel
```

sino:

```text
ColumnTypeView
```

cuando sea práctico.

---

# 161. Avoid God SchemaView

No convertir:

```text
SchemaView
```

en un objeto gigantesco con cientos de métodos.

---

# 162. Specialized views

Podrán existir:

```text
RelationLookupView
ColumnLookupView
TypeMetadataView
ConstraintMetadataView
KeyMetadataView
CapabilityMetadataView
```

---

# 163. Composition

`SchemaView` puede ser una composición de esas interfaces.

---

# 164. Schema resolution artifact

El resultado podrá formar:

```text
SchemaAwareSemanticArtifact
```

o integrarse dentro de:

```text
SemanticQueryArtifact
```

---

# 165. Recommended structure

```text
SemanticQueryArtifact
├── ScopeGraph
├── SymbolTable
├── SymbolResolutionTable
├── SchemaResolutionTable
├── QueryTypeTable
├── ConstraintTable
├── CorrelationMetadata
└── SemanticDiagnostics
```

---

# 166. SchemaResolutionEntry

Conceptualmente:

```text
SchemaResolutionEntry
├── symbolId
├── schemaObjectId
├── objectKind
├── resolutionMode
├── provenance
├── schemaVersion
└── knowledgeState
```

---

# 167. Schema object kinds

```text
RELATION
COLUMN
PRIMARY_KEY
UNIQUE_KEY
FOREIGN_KEY
CONSTRAINT
SEQUENCE
TYPE
EXTENSION_OBJECT
```

---

# 168. Provenance

Debe conservar:

```text
query reference
→ symbol
→ schema object
```

---

# 169. Example

```text
AST N17: u.email
      │
      ▼
ColumnSymbol C4
      │
      ▼
SchemaColumn SC12
      │
      ▼
public.users.email
```

---

# 170. Provenance chain

Esto permite explicar:

```text
Why is this expression typed as EmailAddress?
```

Respuesta:

```text
Expression N17
→ ColumnSymbol C4
→ SchemaColumn users.email
→ mapping metadata
→ app.email_address
```

---

# 171. Schema + ORM metadata

Cuando ORM participa, puede existir:

```text
Schema Metadata
+
ORM Mapping Metadata
```

---

# 172. Separation

Nunca:

```text
ORM Metadata = Schema Metadata
```

---

# 173. Example

Schema:

```text
users.id → BIGINT
```

ORM:

```text
User::$id → UserId
```

Query semantic type:

```text
app.user_id
```

---

# 174. Evidence combination

```text
Schema Column
      +
ORM Mapping
      +
Query Context
      │
      ▼
Query Type Resolution
```

---

# 175. Conflict

Si ORM declara:

```text
UUID
```

pero schema declara:

```text
INTEGER
```

no deberá elegirse silenciosamente uno.

---

# 176. MetadataConflictDiagnostic

Debe producirse:

```text
SchemaOrmMetadataConflict
```

según policy.

---

# 177. Schema authority

Para estructura física:

```text
Schema Metadata
```

es autoridad.

Para domain semantics:

```text
ORM Mapping
```

puede aportar información adicional.

---

# 178. No ORM requirement

El Query Engine debe funcionar perfectamente sin ORM.

---

# 179. Security integration

El SchemaView no deberá contener:

```text
credentials
connection passwords
secret DSNs
```

---

# 180. Sensitive schema metadata

Algunas organizaciones consideran sensibles:

```text
table names
column names
internal schemas
```

Los diagnostics deberán respetar:

```text
SchemaDiagnosticRedactionPolicy
```

---

# 181. Access control

Schema visibility puede ser filtrada por:

```text
SchemaVisibilityView
```

si una integración lo requiere.

---

# 182. Important distinction

```text
schema object exists
```

no significa necesariamente:

```text
application is authorized to access it
```

---

# 183. Authorization boundary

El Query Engine core no implementará authorization de aplicación.

Una integración podrá proporcionar una vista ya filtrada o policies semánticas.

---

# 184. No current user

`SchemaAwareQueryResolver` no deberá consultar:

```text
Auth::user()
Session
Request
Tenant::current()
```

---

# 185. Tenant integration

Multitenancy podrá proporcionar:

```text
TenantResolvedSchemaView
```

o:

```text
SchemaNamespaceSnapshot
```

al QueryProcessingContext.

---

# 186. No tenant object

El resolver no necesita:

```text
TenantEntity
```

---

# 187. Tenant isolation

El contexto ya deberá contener el schema view correcto para la operación.

---

# 188. Example

```text
TenantContext
     │
     ▼
Integration Adapter
     │
     ▼
SchemaView tenant_42
     │
     ▼
QueryProcessingContext
```

---

# 189. No core Multitenancy dependency

`Quantum/Database` continuará funcionando sin el paquete Multitenancy.

---

# 190. Dynamic relation providers

Extensiones podrán aportar relaciones virtuales.

Ejemplos:

```text
foreign data source
search index relation
analytics relation
virtual table
```

---

# 191. Extension contract

Podrá existir:

```php
interface SchemaRelationProviderInterface
{
    public function resolve(
        SchemaRelationReference $reference,
        SchemaResolutionContext $context,
    ): SchemaRelationResolutionResult;
}
```

---

# 192. Provider registry

Los providers se registrarán durante bootstrap.

---

# 193. Frozen registry

Después:

```text
SchemaRelationProviderRegistry
```

será immutable.

---

# 194. Provider precedence

La precedencia será explícita.

No dependerá del orden accidental del container.

---

# 195. Extension collision

Si dos providers reclaman la misma relation:

```text
AMBIGUOUS
```

salvo policy explícita.

---

# 196. Native vs extension relations

El resolver podrá distinguir provenance:

```text
SCHEMA_SNAPSHOT
EXTENSION_PROVIDER
GENERATED
TEST
OPAQUE
```

---

# 197. Capability model

Cada schema object podrá exponer capabilities relevantes.

---

# 198. Relation capabilities

Ejemplos:

```text
selectable
insertable
updatable
deletable
lockable
returningSupported
```

---

# 199. Column capabilities

Ejemplos:

```text
readable
insertable
updatable
comparable
orderable
groupable
indexable
```

---

# 200. Type capabilities

No duplicar el Query Type System.

Schema sólo aporta evidencia.

---

# 201. Capability composition

```text
Schema Object Capability
+
Platform Capability
+
Query Operation
        │
        ▼
Effective Semantic Capability
```

---

# 202. Example RETURNING

Una relation puede ser writable, pero:

```text
RETURNING
```

depende también de:

```text
Platform Capability
```

---

# 203. No schema-only conclusion

Por ello:

```text
relation.returningSupported
```

deberá interpretarse como parte de una composición, no como verdad universal aislada.

---

# 204. Constraint-aware symbol information

Una columna podrá tener:

```text
SchemaColumnFacts
```

---

# 205. Facts

Ejemplo:

```text
NOT_NULL
PRIMARY_KEY_MEMBER
UNIQUE_MEMBER
FOREIGN_KEY_SOURCE
FOREIGN_KEY_TARGET
GENERATED
HAS_DEFAULT
```

---

# 206. Facts ≠ inferred query facts

Posteriormente el Query Constraint Analyzer podrá derivar:

```text
WHERE id = 10
→ id has a single known value in this branch
```

Eso no pertenece al schema.

---

# 207. Static vs derived facts

```text
Schema Facts
≠
Query-Derived Facts
```

---

# 208. Schema completeness

Cada snapshot podrá declarar:

```text
SchemaCompleteness
```

---

# 209. Levels

```text
FULL
RELATIONS_ONLY
COLUMNS
TYPES
CONSTRAINTS
PARTIAL
UNKNOWN
```

preferiblemente mediante capabilities/flags componibles y no necesariamente un único enum.

---

# 210. Why completeness matters

Un schema snapshot podría conocer:

```text
tables + columns + types
```

pero no:

```text
foreign keys
```

El analyzer no deberá interpretar:

```text
missing FK metadata
```

como:

```text
there are no foreign keys
```

---

# 211. Critical distinction

```text
known absent
≠
not known
```

---

# 212. Knowledge<T>

Podrá utilizarse un modelo:

```text
Knowledge<T>
```

con estados:

```text
KNOWN(value)
KNOWN_ABSENT
UNKNOWN
OPAQUE
```

---

# 213. Avoid null ambiguity

No utilizar:

```php
null
```

para significar simultáneamente:

```text
does not exist
unknown
not loaded
not applicable
```

---

# 214. Schema knowledge example

```text
foreignKeys(users)
→ UNKNOWN
```

es diferente de:

```text
foreignKeys(users)
→ KNOWN([])
```

---

# 215. Partial snapshots

Esto permite herramientas rápidas que carguen únicamente metadata necesaria.

---

# 216. Schema projection

Podrá construirse un:

```text
ProjectedSchemaView
```

con sólo objetos relevantes para determinada compilación.

---

# 217. Benefits

Reduce:

- memoria;
- serialization size;
- build artifacts;
- lookup cost.

---

# 218. Lazy schema loading

Debe tratarse con cuidado.

---

# 219. Safe lazy loading

Puede existir lazy materialization desde:

```text
immutable local metadata source
```

---

# 220. Forbidden lazy loading

No:

```text
lookup missing table
→ silently query production database
```

---

# 221. Deterministic source requirement

Toda lazy resolution debe utilizar una fuente declarada en el contexto y compatible con determinismo.

---

# 222. Schema artifact serialization

Los snapshots podrán serializarse para:

```text
production metadata cache
CI
deployment
IDE
static analysis
```

---

# 223. Serialization requirements

Debe ser:

```text
versioned
deterministic
resource-free
closure-free
credential-free
portable where possible
```

---

# 224. Compiled schema metadata

En producción podrá existir:

```text
CompiledSchemaMetadata
```

---

# 225. Boot loading

```text
CompiledSchemaMetadata
      │
      ▼
SchemaView
      │
      ▼
shared immutable worker state
```

---

# 226. No reflection dependency

El schema físico no deberá depender de reflection sobre entities.

---

# 227. ORM metadata compilation separate

Schema metadata y ORM metadata podrán compilarse por separado.

---

# 228. Schema invalidation

La invalidación podrá dispararse por:

```text
migration completed
schema snapshot regenerated
extension schema changed
configuration changed
target platform changed
```

---

# 229. Generation counter

Opcionalmente:

```text
SchemaGeneration
```

podrá utilizarse junto al fingerprint.

---

# 230. Prepared query implications

Una query preparada semánticamente contra:

```text
Schema V10
```

no debe reutilizarse ciegamente con:

```text
Schema V11
```

si cambió metadata relevante.

---

# 231. Cache key

Conceptualmente:

```text
SemanticArtifactCacheKey
=
QuerySemanticFingerprint
+
SchemaSemanticFingerprint
+
PolicyFingerprint
+
CapabilityFingerprint
```

---

# 232. Planning cache

Puede requerir además:

```text
SchemaPlanningFingerprint
```

---

# 233. Compiled query cache

Puede requerir:

```text
DialectFingerprint
+
CompilerFingerprint
+
BindingShapeFingerprint
```

además de información semántica.

---

# 234. Schema changes irrelevant to query

Idealmente, cambios no relacionados no deberían invalidar todo.

---

# 235. Dependency set

Cada query podrá producir:

```text
SchemaDependencySet
```

---

# 236. Example

```text
Query dependencies
├── users.id
├── users.email
└── users.status
```

---

# 237. Fine-grained invalidation

Si cambia:

```text
orders.notes
```

la query anterior podría continuar válida.

---

# 238. V1 strategy

V1 puede comenzar con:

```text
whole SchemaSemanticFingerprint
```

por simplicidad.

---

# 239. Future strategy

V2/V3 podrán utilizar:

```text
relation-level fingerprints
column-level fingerprints
dependency-aware invalidation
```

---

# 240. SchemaDependency

```text
SchemaDependency
├── objectId
├── dependencyKind
├── semanticFingerprint
└── requiredKnowledge
```

---

# 241. Dependency kinds

```text
EXISTENCE
TYPE
NULLABILITY
KEY
CONSTRAINT
CAPABILITY
OUTPUT_SHAPE
QUALIFICATION
```

---

# 242. Important optimization

Una query que sólo necesita:

```text
users.id exists
```

no debería depender necesariamente de todos los índices de `users`.

---

# 243. Dependency precision

Esto será importante para caches de larga duración en persistent runtimes.

---

# 244. Query normalization

Schema resolution ocurre después de la normalización estructural necesaria para obtener referencias canónicas.

---

# 245. But before full type inference

Debe ocurrir antes de que:

```text
Query Type Inference
```

necesite tipos de columnas.

---

# 246. Semantic pipeline

```text
Normalized Query AST
        │
        ▼
Scope Construction
        │
        ▼
Symbol Definition
        │
        ▼
Symbol Resolution
        │
        ▼
Schema-Aware Resolution
        │
        ▼
Relation Output Construction
        │
        ▼
Type Inference
        │
        ▼
Relation / Join Resolution
        │
        ▼
Constraint Analysis
        │
        ▼
Semantic Graph
```

---

# 247. Cyclic semantic dependencies

Algunas fases pueden requerir iteración.

Ejemplo:

```text
derived relation output
depends on
expression types
```

mientras:

```text
outer query column resolution
depends on
derived relation output
```

---

# 248. Dependency graph

VoltStack deberá modelar estas dependencias explícitamente.

---

# 249. Staged resolution

Ejemplo:

```text
Pass 1:
resolve physical relations

Pass 2:
resolve physical columns

Pass 3:
analyze inner query outputs

Pass 4:
publish derived relation outputs

Pass 5:
resolve dependent outer references
```

---

# 250. No uncontrolled recursion

La coordinación deberá utilizar:

```text
semantic dependency graph
```

o un mecanismo bounded equivalente.

---

# 251. Recursive CTE complication

Los recursive CTE requieren:

```text
provisional output shape
```

seguido de:

```text
recursive term validation
```

---

# 252. Provisional relation output

Podrá existir:

```text
ProvisionalRelationOutput
```

durante semantic analysis.

---

# 253. Finalization

Antes de publicar el Semantic Artifact:

```text
all required relation outputs
```

deberán estar finalizados.

---

# 254. No provisional artifact leakage

El Optimizer no deberá recibir:

```text
half-resolved relation output
```

---

# 255. Set operations

Para:

```sql
SELECT id FROM users
UNION
SELECT user_id FROM orders
```

el output relation se deriva de ambas ramas.

---

# 256. Schema contribution

Cada rama puede obtener sus columnas del schema, pero:

```text
UNION output
```

es un:

```text
Derived RelationOutput
```

---

# 257. Common type

El tipo final se resolverá posteriormente mediante:

```text
CommonTypeResolver
```

---

# 258. CTE output columns

Un CTE puede declarar nombres:

```sql
WITH data(id, name) AS (...)
```

Esos nombres forman el output visible, aunque las expressions internas tengan otros labels.

---

# 259. Schema independence

Por tanto:

```text
CTE output
```

no debe tratarse como schema physical relation.

---

# 260. Table functions

Una table function puede obtener output shape mediante:

```text
FunctionDescriptor
```

en lugar de SchemaView.

---

# 261. Unified relation output provider

Podrá existir:

```text
RelationOutputResolver
```

que delegue:

```text
SCHEMA_RELATION → SchemaRelationOutputResolver
CTE             → CteOutputResolver
DERIVED_QUERY   → DerivedQueryOutputResolver
VALUES          → ValuesOutputResolver
TABLE_FUNCTION  → TableFunctionOutputResolver
SET_OPERATION   → SetOperationOutputResolver
EXTENSION       → ExtensionOutputResolver
```

---

# 262. Important boundary

`SchemaAwareQueryResolver` sólo posee la rama:

```text
SCHEMA_RELATION
```

aunque colabora con el sistema unificado de outputs.

---

# 263. Architecture benefit

Evita convertir:

```text
Schema
```

en centro universal de todas las relaciones SQL.

---

# 264. Error taxonomy

Errores principales:

```text
SchemaResolutionException
UnknownSchemaRelationException
AmbiguousSchemaRelationException
UnknownSchemaColumnException
AmbiguousSchemaColumnException
SchemaKnowledgeInsufficientException
SchemaSnapshotMismatchException
SchemaMetadataConflictException
InvalidSchemaQualificationException
UnsupportedSchemaObjectException
SchemaResolutionBudgetExceededException
```

---

# 265. Diagnostic codes

Sugeridos:

```text
DB-SCHEMA-RES-001 UNKNOWN_RELATION
DB-SCHEMA-RES-002 AMBIGUOUS_RELATION
DB-SCHEMA-RES-003 UNKNOWN_COLUMN
DB-SCHEMA-RES-004 AMBIGUOUS_COLUMN
DB-SCHEMA-RES-005 INSUFFICIENT_SCHEMA_KNOWLEDGE
DB-SCHEMA-RES-006 INVALID_QUALIFICATION
DB-SCHEMA-RES-007 SNAPSHOT_MISMATCH
DB-SCHEMA-RES-008 METADATA_CONFLICT
DB-SCHEMA-RES-009 UNSUPPORTED_OBJECT
DB-SCHEMA-RES-010 WRITE_NOT_SUPPORTED
DB-SCHEMA-RES-011 BUDGET_EXCEEDED
DB-SCHEMA-RES-012 INVALID_SCHEMA_STATE
```

---

# 266. Unknown relation diagnostic

Ejemplo:

```text
DB-SCHEMA-RES-001

Relation "userss" was not found in the active schema view.

Visible candidates:
- users
- user_sessions
```

---

# 267. Unknown column diagnostic

```text
DB-SCHEMA-RES-003

Column "emial" was not found on relation "users".

Did you mean "email"?
```

---

# 268. Insufficient knowledge

```text
DB-SCHEMA-RES-005

The current schema snapshot does not contain constraint metadata
required by this analysis.
```

---

# 269. Diagnostics and partial knowledge

Debe distinguir:

```text
object does not exist
```

de:

```text
current snapshot cannot determine whether object exists
```

---

# 270. Performance

La resolución debe ser principalmente lookup sobre estructuras indexadas.

---

# 271. Relation index

```text
SchemaRelationIndex
```

podrá mapear:

```text
QualifiedRelationKey
→ SchemaRelationId
```

---

# 272. Column index

```text
SchemaColumnIndex
```

mapeará:

```text
SchemaRelationId
+
CanonicalColumnName
→ SchemaColumnId
```

---

# 273. Constraint indexes

Podrán existir índices por:

```text
relation
column
constraint kind
```

---

# 274. Expected complexity

Lookup normal:

```text
O(1)
```

aproximado.

Search-path resolution:

```text
O(search path depth)
```

---

# 275. Avoid scanning schema

No recorrer:

```text
every table
```

para resolver cada relation.

---

# 276. Diagnostic fuzzy lookup

Los scans para sugerencias deberán ser bounded y sólo ejecutarse tras error.

---

# 277. Processing budgets

El Query Processing Context podrá limitar:

```text
schema lookups
relation candidates
column candidates
wildcard expansion
dependency count
diagnostic suggestions
```

---

# 278. Wildcard expansion

```sql
SELECT *
FROM huge_table
```

puede generar miles de output columns.

---

# 279. Budget rule

La expansión deberá respetar:

```text
maxExpandedColumns
```

---

# 280. Resource exhaustion

No permitir que una query dinámica provoque:

```text
unbounded metadata expansion
```

---

# 281. Persistent runtime safety

`SchemaView` compartido:

```text
immutable
```

State de resolución:

```text
operation-scoped
```

---

# 282. No static current schema

Nunca:

```php
SchemaResolver::$currentSchema
```

---

# 283. No mutable singleton

Nunca:

```text
singleton SchemaResolver
+
mutable active tenant schema
```

---

# 284. Safe singleton

Un resolver stateless puede ser application-scoped si recibe:

```text
SchemaView
ResolutionContext
```

explícitamente.

---

# 285. Concurrent schemas

Debe soportarse:

```text
Fiber A → SchemaView tenant-A
Fiber B → SchemaView tenant-B
```

simultáneamente.

---

# 286. No leakage

El resolver de Fiber B jamás deberá consultar accidentalmente el snapshot de A.

---

# 287. FrankenPHP model

```text
Worker
├── Resolver services (stateless)
├── Immutable registries
│
├── Request A
│   └── QueryContext
│       └── SchemaView A
│
└── Request B
    └── QueryContext
        └── SchemaView B
```

---

# 288. OpenSwoole compatibility

El mismo principio permite concurrencia coroutine-safe.

---

# 289. RoadRunner compatibility

Los workers persistentes reutilizan servicios, no mutable schema operation state.

---

# 290. Testing strategy

El sistema deberá probarse sin DB real en la mayoría de casos.

---

# 291. In-memory SchemaView

Testing proporcionará:

```text
InMemorySchemaView
```

---

# 292. Example fixture

```text
users
├── id: UserId NOT NULL PK
├── email: EmailAddress NOT NULL UNIQUE
├── status: UserStatus NOT NULL
└── deleted_at: DateTime NULL
```

---

# 293. Relation tests

Probar:

```text
unqualified relation
qualified relation
unknown relation
ambiguous relation
search path
CTE shadowing
```

---

# 294. Column tests

```text
known column
unknown column
qualified column
ambiguous unqualified column
generated column
hidden column
```

---

# 295. Type tests

```text
integer
decimal
uuid
json
enum
domain type
unknown type
```

---

# 296. Nullability tests

```text
NOT NULL
NULLABLE
UNKNOWN
outer join transformation
```

---

# 297. Key tests

```text
single PK
composite PK
unique constraint
composite unique
foreign key
composite foreign key
```

---

# 298. Partial schema tests

Debe probarse:

```text
KNOWN_ABSENT
UNKNOWN
PARTIAL
OPAQUE
```

por separado.

---

# 299. Snapshot tests

```text
V1 analysis
V2 publication
V1 remains immutable
new analysis uses V2
```

---

# 300. Concurrent snapshot tests

```text
Query A + Schema V1
Query B + Schema V2
```

sin contaminación.

---

# 301. Cache tests

Verificar invalidación cuando cambia:

```text
column type
nullability
relation existence
constraint
capability
qualification rules
```

---

# 302. Non-invalidation tests

Verificar que metadata irrelevante no invalide semantic artifacts cuando se utilicen fingerprints granulares.

---

# 303. Architecture tests

Prohibir dependencias hacia:

```text
PDO
NativeConnection
ConnectionLease
ConnectionManager
QueryExecutor
EntityManager
UnitOfWork
IdentityMap
HTTP Request
Session
Service Container
```

---

# 304. Extension tests

Probar:

```text
custom relation provider
provider collision
provider precedence
custom virtual relation
partial metadata provider
```

---

# 305. Budget tests

Probar:

```text
too many wildcard columns
too many relation candidates
too many dependencies
```

---

# 306. Invariantes arquitectónicos

## DB-SCHEMA-RES-001

Schema-Aware Query Resolution nunca abrirá una conexión física.

## DB-SCHEMA-RES-002

Schema-Aware Query Resolution nunca ejecutará SQL.

## DB-SCHEMA-RES-003

Schema-Aware Query Resolution consumirá un SchemaView explícito.

## DB-SCHEMA-RES-004

SchemaView será read-only.

## DB-SCHEMA-RES-005

SchemaView publicado será immutable.

## DB-SCHEMA-RES-006

SchemaView tendrá identidad/version.

## DB-SCHEMA-RES-007

Una operación semántica utilizará un snapshot coherente.

## DB-SCHEMA-RES-008

El snapshot no cambiará a mitad del análisis.

## DB-SCHEMA-RES-009

Un schema nuevo producirá un snapshot nuevo.

## DB-SCHEMA-RES-010

RelationSymbol no será SchemaRelationDescriptor.

## DB-SCHEMA-RES-011

ColumnSymbol no será SchemaColumnDescriptor.

## DB-SCHEMA-RES-012

SchemaRelationId no será RelationSymbolId.

## DB-SCHEMA-RES-013

SchemaColumnId no será ColumnSymbolId.

## DB-SCHEMA-RES-014

Múltiples RelationSymbols podrán apuntar a la misma SchemaRelation.

## DB-SCHEMA-RES-015

Esto deberá soportar self joins.

## DB-SCHEMA-RES-016

Schema Resolution Table será separada de Symbol Table.

## DB-SCHEMA-RES-017

AST no almacenará mutable schema descriptors.

## DB-SCHEMA-RES-018

Symbols no almacenarán mutable schema resolution state.

## DB-SCHEMA-RES-019

Toda relación visible producirá eventualmente RelationOutput.

## DB-SCHEMA-RES-020

No toda RelationOutput procederá de SchemaView.

## DB-SCHEMA-RES-021

CTE no será tratado como tabla física.

## DB-SCHEMA-RES-022

Derived Query no será tratada como tabla física.

## DB-SCHEMA-RES-023

VALUES no será tratada como tabla física.

## DB-SCHEMA-RES-024

Table Function no será tratada automáticamente como tabla física.

## DB-SCHEMA-RES-025

Set Operation producirá output derivado.

## DB-SCHEMA-RES-026

Schema relation lookup respetará qualification semantics.

## DB-SCHEMA-RES-027

Schema relation lookup respetará search path snapshot.

## DB-SCHEMA-RES-028

Search path no será leído desde una conexión activa.

## DB-SCHEMA-RES-029

Vendor-specific qualification estará encapsulada.

## DB-SCHEMA-RES-030

No existirán vendor conditionals dispersos.

## DB-SCHEMA-RES-031

Ambiguous relation producirá error explícito.

## DB-SCHEMA-RES-032

No existirá arbitrary first-match resolution.

## DB-SCHEMA-RES-033

Precedencia de search path deberá ser explícita.

## DB-SCHEMA-RES-034

Unknown relation no será convertida silenciosamente en opaque relation.

## DB-SCHEMA-RES-035

Permissive/opaque behavior será explícito.

## DB-SCHEMA-RES-036

Unknown column no recibirá un tipo inventado.

## DB-SCHEMA-RES-037

Unknown type permanecerá Unknown cuando no exista evidencia.

## DB-SCHEMA-RES-038

Known absent será diferente de Unknown.

## DB-SCHEMA-RES-039

Partial schema knowledge será representable.

## DB-SCHEMA-RES-040

Missing metadata no significará automáticamente ausencia.

## DB-SCHEMA-RES-041

Schema type será evidencia para Query Type System.

## DB-SCHEMA-RES-042

Physical type no será igualado directamente con Query Semantic Type.

## DB-SCHEMA-RES-043

Schema nullability será evidencia inicial.

## DB-SCHEMA-RES-044

Schema nullability no será la nullability final de toda expression.

## DB-SCHEMA-RES-045

Outer joins podrán modificar query nullability.

## DB-SCHEMA-RES-046

Primary keys podrán ser compuestas.

## DB-SCHEMA-RES-047

Unique constraints podrán ser compuestas.

## DB-SCHEMA-RES-048

Foreign keys podrán ser compuestas.

## DB-SCHEMA-RES-049

Foreign Key no será igual a ORM Relationship.

## DB-SCHEMA-RES-050

Constraints opacas no serán fingidas como comprendidas.

## DB-SCHEMA-RES-051

Index metadata no será semantic constraint automáticamente.

## DB-SCHEMA-RES-052

Generated columns tendrán write capabilities explícitas.

## DB-SCHEMA-RES-053

Generation strategy no asumirá AUTO_INCREMENT universal.

## DB-SCHEMA-RES-054

Default expressions serán estructuradas cuando sea posible.

## DB-SCHEMA-RES-055

Opaque defaults permanecerán opacos.

## DB-SCHEMA-RES-056

DML target columns deberán resolverse contra relation output/schema.

## DB-SCHEMA-RES-057

Column writability será explícita.

## DB-SCHEMA-RES-058

View writability dependerá de capabilities.

## DB-SCHEMA-RES-059

Temporary runtime relations requerirán descriptors explícitos.

## DB-SCHEMA-RES-060

No habrá introspection fallback durante semantic analysis.

## DB-SCHEMA-RES-061

Schema Resolution podrá ejecutarse offline.

## DB-SCHEMA-RES-062

Schema Resolution no dependerá del Driver.

## DB-SCHEMA-RES-063

Schema Resolution no dependerá del Connection Manager.

## DB-SCHEMA-RES-064

Schema Resolution no dependerá del Executor.

## DB-SCHEMA-RES-065

Schema Resolution no dependerá del ORM runtime.

## DB-SCHEMA-RES-066

Schema Resolution no dependerá del EntityManager.

## DB-SCHEMA-RES-067

Schema Resolution no dependerá del UnitOfWork.

## DB-SCHEMA-RES-068

Schema Resolution no utilizará Service Container como locator.

## DB-SCHEMA-RES-069

Schema Resolution no consultará HTTP Request.

## DB-SCHEMA-RES-070

Schema Resolution no consultará Session.

## DB-SCHEMA-RES-071

Schema Resolution no consultará current Tenant global.

## DB-SCHEMA-RES-072

Multitenancy será una integración opcional.

## DB-SCHEMA-RES-073

SchemaView podrá ser tenant-specific sin que core conozca Tenant entities.

## DB-SCHEMA-RES-074

Extension providers serán explícitos.

## DB-SCHEMA-RES-075

Extension provider registry se congelará después de bootstrap.

## DB-SCHEMA-RES-076

Provider precedence será determinista.

## DB-SCHEMA-RES-077

Provider collision no se resolverá accidentalmente.

## DB-SCHEMA-RES-078

Schema object capabilities serán estructuradas.

## DB-SCHEMA-RES-079

Platform capabilities permanecerán separadas.

## DB-SCHEMA-RES-080

Effective capability podrá combinar schema + platform + operation.

## DB-SCHEMA-RES-081

Schema facts permanecerán separados de query-derived facts.

## DB-SCHEMA-RES-082

Schema completeness será explícita.

## DB-SCHEMA-RES-083

Schema snapshots podrán serializarse sin resources.

## DB-SCHEMA-RES-084

Schema snapshots no contendrán credentials.

## DB-SCHEMA-RES-085

Compiled schema metadata será versionada.

## DB-SCHEMA-RES-086

Schema semantic fingerprint excluirá metadata irrelevante.

## DB-SCHEMA-RES-087

Schema planning fingerprint podrá incluir información adicional.

## DB-SCHEMA-RES-088

Runtime statistics no formarán parte automáticamente del semantic fingerprint.

## DB-SCHEMA-RES-089

Schema dependencies serán registrables.

## DB-SCHEMA-RES-090

Fine-grained invalidation deberá ser posible arquitectónicamente.

## DB-SCHEMA-RES-091

Mutable resolution workspace será operation-scoped.

## DB-SCHEMA-RES-092

Shared SchemaViews serán immutable.

## DB-SCHEMA-RES-093

No existirá global current SchemaView.

## DB-SCHEMA-RES-094

Queries concurrentes podrán utilizar diferentes SchemaViews.

## DB-SCHEMA-RES-095

Query A no podrá contaminar Schema Resolution de Query B.

## DB-SCHEMA-RES-096

Wildcard expansion será bounded.

## DB-SCHEMA-RES-097

Schema lookup será indexado.

## DB-SCHEMA-RES-098

Diagnostics serán deterministas.

## DB-SCHEMA-RES-099

Required provisional outputs deberán finalizar antes del Semantic Artifact.

## DB-SCHEMA-RES-100

Las fases posteriores no repetirán schema name resolution ya finalizada.

---

# 307. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Semantic\Schema\
```

---

# 308. Estructura propuesta

```text
Query/
└── Semantic/
    └── Schema/
        ├── Contract/
        │   ├── SchemaViewInterface.php
        │   ├── RelationLookupViewInterface.php
        │   ├── ColumnLookupViewInterface.php
        │   ├── SchemaRelationProviderInterface.php
        │   └── RelationOutputResolverInterface.php
        │
        ├── Identity/
        │   ├── SchemaSnapshotId.php
        │   ├── SchemaRelationId.php
        │   ├── SchemaColumnId.php
        │   └── SchemaGeneration.php
        │
        ├── View/
        │   ├── SchemaView.php
        │   ├── ProjectedSchemaView.php
        │   ├── SchemaCatalogView.php
        │   ├── SchemaConstraintView.php
        │   ├── SchemaKeyView.php
        │   └── SchemaCapabilityView.php
        │
        ├── Descriptor/
        │   ├── SchemaRelationDescriptor.php
        │   ├── SchemaColumnDescriptor.php
        │   ├── PrimaryKeyDescriptor.php
        │   ├── UniqueConstraintDescriptor.php
        │   ├── ForeignKeyDescriptor.php
        │   ├── CheckConstraintDescriptor.php
        │   ├── DefaultValueDescriptor.php
        │   └── OpaqueConstraintDescriptor.php
        │
        ├── Reference/
        │   ├── SchemaRelationReference.php
        │   ├── SchemaColumnReference.php
        │   └── SchemaNamespace.php
        │
        ├── Output/
        │   ├── RelationOutput.php
        │   ├── SemanticOutputColumn.php
        │   ├── SchemaRelationOutputFactory.php
        │   ├── DerivedRelationOutput.php
        │   └── ProvisionalRelationOutput.php
        │
        ├── Resolution/
        │   ├── SchemaAwareQueryResolver.php
        │   ├── SchemaRelationResolver.php
        │   ├── SchemaColumnResolver.php
        │   ├── SchemaResolutionContext.php
        │   ├── SchemaResolutionTable.php
        │   ├── SchemaResolutionEntry.php
        │   └── SchemaRelationCandidateSet.php
        │
        ├── Qualification/
        │   ├── SchemaQualificationPolicy.php
        │   ├── SchemaSearchPathSnapshot.php
        │   └── QualifiedRelationKey.php
        │
        ├── Knowledge/
        │   ├── Knowledge.php
        │   ├── SchemaKnowledgeState.php
        │   ├── SchemaCompleteness.php
        │   └── SchemaKnowledgeRequirement.php
        │
        ├── Dependency/
        │   ├── SchemaDependency.php
        │   ├── SchemaDependencySet.php
        │   └── SchemaDependencyKind.php
        │
        ├── Fingerprint/
        │   ├── SchemaSemanticFingerprint.php
        │   ├── SchemaPlanningFingerprint.php
        │   └── SchemaFingerprintBuilder.php
        │
        ├── Capability/
        │   ├── RelationCapabilitySet.php
        │   ├── ColumnCapabilitySet.php
        │   └── ColumnWriteCapability.php
        │
        ├── Extension/
        │   ├── SchemaRelationProviderRegistry.php
        │   ├── SchemaRelationProviderDescriptor.php
        │   └── ExtensionSchemaRelation.php
        │
        ├── Diagnostic/
        │   ├── SchemaResolutionDiagnostic.php
        │   ├── SchemaDiagnosticCode.php
        │   └── SchemaDiagnosticRedactionPolicy.php
        │
        └── Exception/
            ├── SchemaResolutionException.php
            ├── UnknownSchemaRelationException.php
            ├── AmbiguousSchemaRelationException.php
            ├── UnknownSchemaColumnException.php
            ├── SchemaKnowledgeInsufficientException.php
            ├── SchemaSnapshotMismatchException.php
            └── SchemaResolutionBudgetExceededException.php
```

---

# 309. Relación con futuros documentos de Schema

Este documento no sustituye:

```text
87_DATABASE_SCHEMA_ARCHITECTURE.md
88_DATABASE_SCHEMA_MODEL.md
89_DATABASE_SCHEMA_AST_SYSTEM.md
90_DATABASE_SCHEMA_BUILDER_SYSTEM.md
...
96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md
97_DATABASE_SCHEMA_METADATA_SYSTEM.md
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
```

Aquellos documentos definirán cómo se modela, construye, introspecta y compara el schema.

Este documento define únicamente:

> cómo el Query Semantic Engine consume ese conocimiento.

---

# 310. Boundary entre Query y Schema

La dependencia deberá ser:

```text
Query Semantic Engine
        │
        ▼
Schema Read Contracts
        │
        ▼
Schema Model / Metadata
```

Nunca:

```text
Query Semantic Engine
        │
        ▼
Schema Builder
        │
        ▼
Connection
```

---

# 311. Arquitectura final

```text
                         Query AST
                            │
                            ▼
                     Scope Resolution
                            │
                            ▼
                     Symbol Resolution
                            │
                            ▼
                    RelationSymbol
                            │
             ┌──────────────┴───────────────┐
             │                              │
             ▼                              ▼
     Query-local Relation             Schema Relation
     CTE / Derived / VALUES                 │
             │                              ▼
             │                         SchemaView
             │                              │
             │                              ▼
             │                  SchemaRelationDescriptor
             │                              │
             │                              ▼
             │                     Schema Columns
             │                              │
             └──────────────┬───────────────┘
                            ▼
                      RelationOutput
                            │
                            ▼
                       ColumnSymbol
                            │
                            ▼
                 SchemaResolutionTable
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
            Types       Constraints      Keys
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                    Semantic Analysis
```

---

# 312. Flujo completo de ejemplo

Consulta:

```sql
SELECT u.email
FROM users u
WHERE u.id = :id
```

## Paso 1 — AST

```text
SelectQuery
├── Projection
│   └── ColumnReference(u.email)
├── From
│   └── RelationReference(users, alias=u)
└── Where
    └── Comparison
        ├── ColumnReference(u.id)
        └── Parameter(:id)
```

## Paso 2 — Symbol Resolution

```text
u
→ RelationSymbol R1
```

## Paso 3 — Schema Resolution

```text
R1
→ SchemaRelation public.users
```

## Paso 4 — Relation Output

```text
R1
├── C1 id
├── C2 email
├── C3 status
└── C4 created_at
```

## Paso 5 — Column Resolution

```text
u.email
→ ColumnSymbol C2
→ SchemaColumn users.email

u.id
→ ColumnSymbol C1
→ SchemaColumn users.id
```

## Paso 6 — Schema evidence

```text
users.id
├── type = app.user_id
├── nullability = NON_NULL
├── primaryKey = true
└── unique = true

users.email
├── type = app.email_address
├── nullability = NON_NULL
└── unique = true
```

## Paso 7 — Parameter inference

Predicate:

```text
users.id = :id
```

produce constraint:

```text
type(:id) compatible with app.user_id
```

## Paso 8 — Semantic result

```text
Projection:
email → app.email_address / NON_NULL

Predicate:
users.id = :id
→ valid comparable operands
```

Ninguna conexión física fue requerida.

---

# 313. Fórmulas maestras

## Schema relation resolution

```text
Resolved Schema Relation
=
Relation Reference
+
Query-local Symbol Resolution
+
Schema Qualification Policy
+
Search Path Snapshot
+
Schema View
+
Target Environment
```

## Schema column resolution

```text
Resolved Schema Column
=
Column Reference
+
Resolved RelationSymbol
+
RelationOutput
+
Schema Column Metadata
```

## Relation output

```text
RelationOutput
=
Relation Source
+
Output Columns
+
Semantic Types
+
Nullability
+
Provenance
+
Relevant Capabilities
```

## Semantic schema evidence

```text
Schema Semantic Evidence
=
Types
+
Nullability
+
Keys
+
Constraints
+
Capabilities
+
Object Identity
+
Knowledge State
```

## Safe schema-aware analysis

```text
Safe Schema-Aware Analysis
=
Immutable Schema Snapshot
+
Explicit Query Context
+
Structured Resolution
+
No Live Connection
+
No Runtime Introspection
+
Knowledge-State Awareness
+
Deterministic Fingerprints
+
Operation-Scoped Resolution State
```

---

# 314. Decisión arquitectónica final

VoltStack adoptará un modelo donde el Query Engine nunca dependa directamente de una base de datos física para comprender una consulta.

El boundary será:

```text
Database
    │
    ▼
Introspection
    │
    ▼
Schema Model
    │
    ▼
Immutable Schema View
    │
    ═══════════════════════════
      Query Engine Boundary
    ═══════════════════════════
    │
    ▼
Schema-Aware Resolution
    │
    ▼
Semantic Query Information
```

Esto permitirá que el mismo Query Engine funcione en:

```text
HTTP request
CLI
Queue worker
FrankenPHP
RoadRunner
OpenSwoole
CI
IDE
Static analysis
Testing
Offline compilation
Migration tooling
```

sin introducir conexiones ocultas ni estado global.

---

# 315. Invariante central

> El schema es una fuente de conocimiento semántico para una consulta, no un recurso físico que el Semantic Query Engine deba descubrir o administrar.

Y, de forma complementaria:

> Un nombre resuelto identifica un símbolo; un símbolo puede asociarse con un objeto de schema; y esa asociación debe permanecer separada, immutable, explícita y versionable.

---

# 316. Estado del Semantic Query Engine

Con los documentos actuales:

```text
23 Query Architecture
       │
24 Query Model
       │
25 Query AST
       │
26 AST Node Model
       │
├── 27 Expressions
├── 28 Predicates
├── 29 Parameters
├── 30 Query Types
├── 31 Query Metadata
└── 32 Query Context
       │
33 Normalization
       │
34 Validation
       │
35 Semantic Architecture
       │
36 Semantic Analysis
       │
37 Symbol Resolution
       │
38 Schema-Aware Resolution
       ▼
39 Type Inference
       ▼
40 Relation / Join Resolution
       ▼
41 Constraint Analysis
       ▼
42 Semantic Graph
```

La resolución estructural de identidades queda así preparada para alimentar el razonamiento semántico avanzado.

---

# 317. Próximo documento

```text
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack utiliza:

```text
AST expressions
+
Resolved Symbols
+
Schema Evidence
+
Parameter Constraints
+
Function Signatures
+
Operator Signatures
+
ORM Mapping Metadata
+
Domain Types
```

para inferir:

```text
Resolved Query Types
```

mediante un sistema de constraints y propagación bidireccional, evitando estrategias débiles como:

```text
first type wins
PHP runtime type = database type
unknown string = VARCHAR
cast everything until it works
```

El pipeline resultante será:

```text
Resolved Symbols
      │
      ▼
Schema Evidence
      │
      ▼
Type Constraints
      │
      ▼
Constraint Graph
      │
      ▼
Type Solver
      │
      ▼
Resolved Query Types
      │
      ▼
QueryTypeTable
```