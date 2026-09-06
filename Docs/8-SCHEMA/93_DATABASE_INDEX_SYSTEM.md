# 93_DATABASE_INDEX_SYSTEM.md

# VoltStack Quantum Database
## Database Index System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 93 — Database Index System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Index System` define la representación estructural, tipada, inmutable, portable y extensible de los índices persistentes de base de datos dentro del Schema System de VoltStack.

Su responsabilidad fundamental es responder:

> **¿Cómo representa VoltStack un índice de base de datos sin confundirlo con una constraint, una clave lógica, una decisión del Query Planner o SQL específico de un proveedor?**

Modelo conceptual:

```text
IndexDefinition
├── IndexIdentifier
├── IndexKind
├── IndexKeySet
│   ├── ColumnIndexKey
│   └── ExpressionIndexKey
├── IncludedColumnSet
├── Predicate
├── Uniqueness
├── IndexMethod
├── Ordering
├── NullOrdering
├── Collation
├── IndexOptionSet
├── CapabilityRequirements
├── Metadata
└── ExtensionMetadata
```

La separación fundamental será:

```text
IndexDefinition
≠
UniqueConstraintDefinition
≠
PrimaryKeyDefinition
≠
ForeignKeyDefinition
≠
Query AccessPath
≠
PhysicalIndexScan
≠
SQL CREATE INDEX
```

---

# 2. Principio central

Un índice representa una estructura persistente que puede facilitar el acceso a los datos.

No representa por sí mismo una regla lógica de integridad.

Formalmente:

```text
Index
=
Persistent Access Structure
```

mientras:

```text
Constraint
=
Data Integrity Rule
```

y:

```text
AccessPath
=
Query Planning Alternative
```

Por tanto:

```text
Index ≠ Constraint ≠ AccessPath
```

aunque una plataforma pueda utilizar internamente índices para implementar determinadas constraints.

---

# 3. Posición arquitectónica

```text
Schema Builder
      │
      ▼
IndexBlueprint
      │
      │ lowering
      ▼
IndexDefinition
      │
      ▼
TableDefinition
      │
      ▼
Schema Model
      │
      ├──────────────► Schema Diff
      │
      ▼
Schema AST
      │
      ▼
Schema Planner
      │
      ▼
Schema Compiler
      │
      ▼
Compiled DDL Commands
      │
      ▼
Execution Engine
      │
      ▼
Database
```

El camino inverso será:

```text
Database
   │
   ▼
Schema Introspection
   │
   ▼
ObservedIndex
├── IndexDefinition
└── ObservationMetadata
```

---

# 4. Responsabilidades

El sistema deberá proporcionar:

- identidad tipada de índices;
- representación de claves indexadas;
- índices simples;
- índices compuestos;
- índices únicos;
- índices parciales;
- índices basados en expresiones;
- included/covering columns;
- orden por clave;
- null ordering;
- collations cuando apliquen;
- index methods;
- opciones específicas de plataforma;
- capability requirements;
- metadata estructural;
- introspection compatibility;
- canonicalization;
- validation;
- comparison;
- fingerprinting;
- serialization;
- extensibilidad;
- seguridad;
- persistent-runtime safety.

---

# 5. No responsabilidades

`IndexDefinition` no deberá:

- generar SQL;
- ejecutar `CREATE INDEX`;
- ejecutar `DROP INDEX`;
- consultar el catálogo de la DB;
- seleccionar índices para queries;
- estimar costos;
- decidir `IndexScan`;
- decidir `IndexOnlyScan`;
- reemplazar constraints;
- modificar automáticamente columnas;
- realizar migrations;
- decidir online/concurrent DDL;
- abrir conexiones;
- conocer PDO;
- conocer ORM entities;
- almacenar estadísticas runtime.

---

# 6. Modelo principal

Se propone conceptualmente:

```php
final readonly class IndexDefinition
{
    public function __construct(
        public IndexIdentifier $identifier,
        public IndexKind $kind,
        public IndexKeySet $keys,
        public IncludedColumnSet $includedColumns,
        public ?SchemaPredicate $predicate,
        public IndexOptionSet $options,
        public SchemaCapabilityRequirementSet $capabilities,
        public IndexMetadata $metadata,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

Las clases concretas pueden evolucionar.

La separación conceptual deberá permanecer.

---

# 7. IndexDefinition ≠ IndexBlueprint

La API de construcción podrá ser ergonómica:

```php
$table->index('email');
```

o:

```php
$table->index(['tenant_id', 'email']);
```

o:

```php
$table
    ->index(['created_at'])
    ->name('idx_users_created_at');
```

Esto construye inicialmente:

```text
IndexBlueprint
```

Posteriormente:

```text
IndexBlueprint
      │
      │ normalize/lower
      ▼
IndexDefinition
```

El blueprint puede ser mutable.

La definición publicada no.

---

# 8. Inmutabilidad

`IndexDefinition` será inmutable.

Incorrecto:

```php
$index->unique = true;
$index->columns[] = 'email';
```

Correcto:

```text
IndexDefinition₀
       │
       │ transform
       ▼
IndexDefinition₁
```

---

# 9. IndexDefinition ≠ IndexOperation

Una definición representa:

```text
index state
```

Una operación representa:

```text
index transition
```

Por ejemplo:

```text
CreateIndex
DropIndex
RenameIndex
RebuildIndex
```

pertenecen al Schema AST / migration planning.

No a `IndexDefinition`.

---

# 10. IndexDefinition ≠ Query AccessPath

Esta distinción es crítica.

Schema:

```text
IndexDefinition
```

dice:

```text
An index named idx_users_email exists
over users.email.
```

Planner:

```text
IndexAccessPath
```

dice:

```text
This query could potentially use
idx_users_email.
```

Physical Plan:

```text
PhysicalIndexScan
```

dice:

```text
This physical candidate uses
a particular index access strategy.
```

Por tanto:

```text
IndexDefinition
      │
      ▼
Physical Catalog
      │
      ▼
AccessPath Candidates
      │
      ▼
PhysicalIndexScan
```

pero:

```text
IndexDefinition ≠ PhysicalIndexScan
```

---

# 11. IndexDefinition ≠ Constraint

Una constraint establece una regla de integridad.

Ejemplo:

```text
UNIQUE(email)
```

puede ser:

```text
UniqueConstraintDefinition
```

Un índice único puede representarse como:

```text
UniqueIndexDefinition
```

Aunque ambos puedan tener efectos similares en determinadas plataformas:

```text
UniqueConstraint
≠
UniqueIndex
```

---

# 12. ¿Por qué separar unique index y unique constraint?

Porque pueden diferir en:

- intención;
- metadata;
- introspection;
- DDL;
- nombres;
- foreign-key referencing rules;
- partial predicates;
- expression support;
- deferrability;
- portability;
- platform semantics;
- migration behavior.

---

# 13. Regla de intención

Si el desarrollador quiere expresar:

> Estos valores deben ser únicos como regla de integridad.

deberá preferirse:

```text
UniqueConstraintDefinition
```

Si quiere expresar:

> Quiero esta estructura de índice y además es unique.

podrá utilizar:

```text
UniqueIndexDefinition
```

---

# 14. Primary Key ≠ Index

Una primary key será modelada como constraint.

```text
PrimaryKeyDefinition
```

No como:

```text
IndexDefinition(kind: PRIMARY)
```

aunque el motor implemente internamente la PK mediante un índice.

---

# 15. Foreign Key ≠ Index

Igualmente:

```text
ForeignKeyDefinition
≠
IndexDefinition
```

Una plataforma puede exigir o crear índices auxiliares.

Ese comportamiento pertenece a:

```text
Platform Capability
+
Schema Planning
+
Schema Compilation
```

---

# 16. IndexIdentifier

Se propone:

```php
final readonly class IndexIdentifier
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 17. Identifier ≠ SQL identifier

Internamente:

```text
idx_users_email
```

No:

```text
`idx_users_email`
```

ni:

```text
"idx_users_email"
```

El quoting pertenece al Schema Compiler.

---

# 18. QualifiedIndexIdentifier

Cuando sea necesario:

```text
QualifiedIndexIdentifier
├── TableIdentifier
└── IndexIdentifier
```

o, cuando la plataforma tenga index namespaces diferentes:

```text
QualifiedIndexIdentifier
├── Catalog?
├── Namespace?
├── Table?
└── IndexIdentifier
```

La resolución concreta dependerá del modelo de identifiers definido por Schema Architecture.

---

# 19. IndexDefinitionId

Para identidad interna podrá existir:

```text
IndexDefinitionId
```

Debe cumplirse:

```text
IndexDefinitionId
≠
IndexIdentifier
```

---

# 20. IndexKind

Se propone:

```text
IndexKind
├── STANDARD
├── UNIQUE
└── EXTENSION
```

No conviene convertir cada detalle físico en `IndexKind`.

Por ejemplo:

```text
BTREE
HASH
GIN
GiST
BRIN
```

son mejor tratados como:

```text
IndexMethod
```

---

# 21. Index uniqueness

También puede modelarse separadamente:

```php
enum IndexUniqueness
{
    case NON_UNIQUE;
    case UNIQUE;
}
```

Esto puede ser preferible a sobrecargar `IndexKind`.

---

# 22. Recommended model

Conceptualmente:

```text
IndexDefinition
├── uniqueness
├── method
├── keys
├── includes
├── predicate
└── options
```

Esta separación permite combinar conceptos de forma más limpia.

---

# 23. IndexKey

El núcleo de un índice será:

```text
IndexKey
```

---

# 24. IndexKey types

Se propone:

```text
IndexKey
├── ColumnIndexKey
├── ExpressionIndexKey
└── ExtensionIndexKey
```

---

# 25. ColumnIndexKey

Ejemplo:

```text
INDEX(email)
```

se representa como:

```text
ColumnIndexKey
└── ColumnIdentifier(email)
```

---

# 26. ExpressionIndexKey

Ejemplo conceptual:

```sql
CREATE INDEX ...
ON users (LOWER(email))
```

se representa como:

```text
ExpressionIndexKey
└── SchemaExpression
    └── lower(email)
```

No como SQL string primario.

---

# 27. Expression index capability

Una `ExpressionIndexKey` podrá requerir:

```text
EXPRESSION_INDEX
```

dentro del capability set.

---

# 28. Raw expression index

Podrá existir escape hatch:

```text
RawSchemaExpression
```

pero deberá estar marcado como:

```text
explicit
trusted
platform-sensitive
```

---

# 29. IndexKeySet

Se propone:

```php
final readonly class IndexKeySet
{
    /** @var non-empty-list<IndexKey> */
    private array $keys;
}
```

Un índice deberá tener al menos una key salvo una extensión explícita.

---

# 30. Ordered key semantics

El orden importa:

```text
INDEX(a, b)
≠
INDEX(b, a)
```

Por tanto:

```text
IndexKeySet
```

es una secuencia ordenada.

No un set matemático no ordenado.

---

# 31. Composite indexes

Ejemplo:

```text
idx_orders_tenant_created
├── tenant_id
└── created_at
```

representa:

```text
IndexKeySet[
    ColumnIndexKey(tenant_id),
    ColumnIndexKey(created_at)
]
```

---

# 32. Duplicate keys

Por defecto:

```text
INDEX(a, a)
```

deberá ser inválido o requerir justificación explícita de una extensión.

---

# 33. Column existence

Toda `ColumnIndexKey` deberá referenciar una columna existente dentro de la tabla.

Esto requiere:

```text
TableValidationContext
```

---

# 34. Expression dependencies

Una `ExpressionIndexKey` deberá poder declarar:

```text
ColumnDependencySet
```

Ejemplo:

```text
LOWER(email)
      │
      ▼
email
```

---

# 35. Expression volatility

No toda expresión es válida para indexación.

El sistema deberá poder expresar requisitos como:

```text
IMMUTABLE
DETERMINISTIC
PLATFORM_ALLOWED
```

sin que `IndexDefinition` consulte directamente la base de datos.

---

# 36. IndexKey ordering

Cada key podrá tener:

```text
IndexSortDirection
├── ASC
├── DESC
└── UNSPECIFIED
```

---

# 37. Unspecified ≠ ASC

Internamente:

```text
UNSPECIFIED
```

no deberá convertirse automáticamente en:

```text
ASC
```

hasta que las reglas de canonicalization/platform semantics lo determinen.

---

# 38. Null ordering

Cuando aplique:

```text
IndexNullOrdering
├── FIRST
├── LAST
└── UNSPECIFIED
```

---

# 39. Null ordering capability

Una plataforma puede:

- soportarlo explícitamente;
- tener comportamiento fijo;
- ignorarlo;
- no permitir especificarlo.

Eso debe modelarse mediante capabilities.

---

# 40. Per-key collation

Una key textual podrá incluir:

```text
IndexKeyCollation
```

cuando la plataforma lo permita.

---

# 41. Index key definition

Conceptualmente:

```php
final readonly class ColumnIndexKey implements IndexKey
{
    public function __construct(
        public ColumnIdentifier $column,
        public IndexSortDirection $direction,
        public IndexNullOrdering $nullOrdering,
        public ?CollationDefinition $collation,
        public IndexKeyOptionSet $options,
    ) {}
}
```

---

# 42. Included columns

Algunos motores soportan:

```text
INCLUDE(...)
```

para covering indexes.

VoltStack deberá representarlo como:

```text
IncludedColumnSet
```

---

# 43. IncludedColumn ≠ IndexKey

Esta separación es crítica:

```text
Index Key
=
participates in index search/order semantics
```

mientras:

```text
Included Column
=
stored for coverage
but not necessarily part of key semantics
```

Por tanto:

```text
IncludedColumn
≠
IndexKey
```

---

# 44. Example

```text
INDEX
├── keys
│   └── email
└── include
    ├── id
    └── name
```

No debe convertirse a:

```text
INDEX(email, id, name)
```

porque cambia la semántica física.

---

# 45. IncludedColumnSet

Se propone:

```text
IncludedColumnSet
├── id
├── name
└── created_at
```

con identidad de columnas tipada.

---

# 46. Duplicate include

No deberá permitirse:

```text
include(id, id)
```

---

# 47. Key + include duplication

Por defecto:

```text
key(email)
include(email)
```

deberá diagnosticarse.

---

# 48. Covering index ≠ planner index-only scan

Que un índice tenga included columns no significa:

```text
Planner must use IndexOnlyScan
```

Sólo significa que la estructura persistente posee determinada cobertura potencial.

---

# 49. Partial indexes

Un índice parcial deberá soportar:

```text
IndexPredicate
```

Ejemplo:

```text
WHERE deleted_at IS NULL
```

---

# 50. Partial index model

```text
IndexDefinition
├── keys
│   └── email
└── predicate
    └── deleted_at IS NULL
```

---

# 51. Predicate representation

Preferir:

```text
SchemaPredicate
```

estructurado.

No:

```php
$where = 'deleted_at IS NULL';
```

---

# 52. Raw predicate escape hatch

Podrá existir:

```text
RawSchemaPredicate
```

con las mismas reglas de seguridad de otros raw schema expressions.

---

# 53. Partial index capability

La presencia de predicate puede derivar:

```text
PARTIAL_INDEX
```

como capability requirement.

---

# 54. Predicate dependencies

El predicate deberá declarar:

```text
ColumnDependencySet
```

para validación.

---

# 55. Predicate semantics

No deberá permitirse asumir que cualquier query predicate puede usarse como partial-index predicate.

Existen restricciones por plataforma.

---

# 56. IndexMethod

Se propone:

```text
IndexMethod
├── DEFAULT
├── BTREE
├── HASH
├── GIN
├── GIST
├── BRIN
├── FULLTEXT
├── SPATIAL
└── EXTENSION
```

La taxonomía final podrá evolucionar.

---

# 57. DEFAULT method

`DEFAULT` significa:

```text
Use platform's default index method
```

No significa necesariamente:

```text
BTREE
```

---

# 58. Method capability

Cada method podrá requerir:

```text
INDEX_METHOD_BTREE
INDEX_METHOD_HASH
INDEX_METHOD_GIN
INDEX_METHOD_GIST
INDEX_METHOD_BRIN
...
```

---

# 59. Method/type compatibility

Ejemplo conceptual:

```text
GIN
+
arbitrary scalar type
```

puede necesitar operator class u otra metadata.

La validación deberá ser capability/type aware.

---

# 60. PostgreSQL operator classes

Para plataformas que lo soporten, una key podrá declarar:

```text
IndexOperatorClass
```

como metadata estructural específica de plataforma.

---

# 61. OperatorClass ≠ generic string option

No usar:

```php
$options['opclass'] = 'gin_trgm_ops';
```

como única abstracción.

Preferir:

```text
PostgreSqlOperatorClass
```

tipada.

---

# 62. PostgreSQL index methods

La arquitectura deberá poder representar sin hacks:

```text
BTREE
HASH
GIN
GiST
SP-GiST
BRIN
```

mediante extensiones/capabilities.

---

# 63. MySQL/MariaDB index methods

El compiler/capability layer podrá manejar diferencias alrededor de:

```text
BTREE
HASH
FULLTEXT
SPATIAL
```

sin contaminar el core con vendor conditionals.

---

# 64. SQLite indexes

SQLite deberá participar mediante el mismo modelo canónico.

Las capabilities determinarán qué features puede expresar físicamente.

---

# 65. IndexOptionSet

Características adicionales deberán representarse mediante:

```text
IndexOptionSet
```

---

# 66. Typed options

Conceptualmente:

```text
IndexOption
├── PortableIndexOption
├── MySqlIndexOption
├── MariaDbIndexOption
├── PostgreSqlIndexOption
├── SqliteIndexOption
└── ExtensionIndexOption
```

---

# 67. Potential options

Podrán incluir:

```text
fill factor
storage parameters
tablespace
parser
algorithm preference
visibility
compression
concurrency mode
locking hints
platform extensions
```

si forman parte realmente de la definición persistente.

---

# 68. Online/concurrent creation

Debe distinguirse:

```text
IndexDefinition
```

de:

```text
IndexCreationStrategy
```

Por ejemplo:

```text
CREATE INDEX CONCURRENTLY
```

puede ser una estrategia de operación/migration.

No necesariamente una propiedad permanente del índice.

---

# 69. Persistent property vs operation option

Regla:

```text
If it describes what the index IS
→ IndexDefinition

If it describes how the index is CREATED
→ Schema Operation / Migration Plan
```

---

# 70. Example

Esto:

```text
index method = BTREE
```

pertenece a `IndexDefinition`.

Esto:

```text
create concurrently
```

pertenece normalmente a `CreateIndexOperation`.

---

# 71. Index visibility

Si una plataforma soporta índices visibles/invisibles y dicha propiedad persiste en el schema:

```text
IndexVisibility
```

puede formar parte de la definición.

---

# 72. Visibility ≠ planner guarantee

Un índice visible:

```text
does not guarantee use
```

y uno invisible:

```text
does not necessarily mean physically absent
```

---

# 73. Index metadata

Se propone:

```php
final readonly class IndexMetadata
{
    public function __construct(
        public SchemaObjectOrigin $origin,
        public ?SchemaSourceLocation $source,
        public AnnotationSet $annotations,
        public ValueProvenanceSet $provenance,
    ) {}
}
```

---

# 74. Index comment

Si una plataforma soporta comentarios estructurales de índices:

```text
IndexComment
```

podrá modelarse explícitamente.

No deberá confundirse con source documentation.

---

# 75. Capability requirements

La definición podrá derivar requisitos de:

```text
method
expression keys
partial predicate
included columns
sort direction
null ordering
collation
operator classes
options
extensions
```

---

# 76. Example capability set

```text
IndexDefinition
├── ExpressionIndexKey
├── Predicate
└── IncludedColumns
```

podría producir:

```text
SchemaCapabilityRequirementSet
├── EXPRESSION_INDEX
├── PARTIAL_INDEX
└── INCLUDED_COLUMNS
```

---

# 77. Capability requirements ≠ capability checks

La definición declara:

```text
requires X
```

El compatibility layer decide:

```text
platform supports X?
```

---

# 78. No vendor conditionals

Incorrecto:

```php
if ($driver === 'pgsql') {
    $index->include(...);
}
```

dentro del core.

Correcto:

```text
IndexDefinition
      ↓
Capability Requirements
      ↓
Platform Compatibility
```

---

# 79. Index naming

VoltStack deberá soportar:

```text
ExplicitIndexName
GeneratedIndexName
IntrospectedIndexName
```

como provenance, aunque todos terminen en `IndexIdentifier`.

---

# 80. Generated index names

El Schema Builder podrá generar nombres deterministas.

Ejemplo conceptual:

```text
users(email)
      ↓
idx_users_email
```

---

# 81. Naming strategy

La generación debe estar delegada a:

```text
SchemaNamingStrategy
```

No codificada dentro de `IndexDefinition`.

---

# 82. Deterministic names

Debe cumplirse:

```text
Same Naming Context
+
Same Structural Input
+
Same Naming Strategy Version
=
Same Generated Name
```

---

# 83. Identifier length limits

Si una plataforma limita nombres:

```text
GeneratedIndexName
```

podrá pasar por:

```text
PlatformIdentifierPolicy
```

antes de compilación.

---

# 84. Stable truncation

Cuando sea necesario truncar:

```text
prefix + deterministic hash suffix
```

es preferible a truncación que produzca colisiones silenciosas.

---

# 85. Explicit names should not be silently renamed

Si el usuario declara un nombre incompatible con la plataforma:

```text
compatibility diagnostic
```

es preferible a modificarlo silenciosamente.

---

# 86. Canonicalization

Debe existir:

```text
IndexDefinitionNormalizer
```

---

# 87. Normalization properties

Será:

```text
deterministic
idempotent
side-effect free
semantics-preserving
```

---

# 88. Normalization example

Una DSL:

```php
$table->index('email');
```

y:

```php
$table->index(['email']);
```

podrán producir el mismo:

```text
IndexDefinition
```

---

# 89. Normalization does not select method

Si method es:

```text
DEFAULT
```

la normalización no deberá convertirlo automáticamente a `BTREE` consultando la DB.

---

# 90. Explicit capability snapshot

Si alguna canonicalization depende legítimamente de capabilities:

```text
IndexNormalizationContext
```

deberá recibirlas explícitamente.

---

# 91. Structural equality

VoltStack deberá distinguir:

```text
Object Identity
IndexIdentifier Equality
Structural Equality
Normalized Structural Equality
Platform Equivalence
Migration Equivalence
Planner Equivalence
```

---

# 92. Structural equality example

Estos índices:

```text
INDEX(a, b)
```

y:

```text
INDEX(b, a)
```

no son estructuralmente iguales.

---

# 93. Name-independent equality

Schema Diff podrá necesitar un profile que compare estructura ignorando el nombre.

Ejemplo:

```text
idx_old(a,b)
idx_new(a,b)
```

para detectar posible rename.

Pero:

```text
NameIndependentEquivalent
≠
Same Index
```

---

# 94. Rename inference

Nunca asumir:

```text
same keys
+
different name
⇒ rename
```

Podría ser:

```text
drop old
+
create new
```

o coexistencia deliberada.

---

# 95. Index fingerprint

Se propone:

```text
IndexStructuralFingerprint
```

---

# 96. Fingerprint inputs

Puede incluir:

```text
identifier
uniqueness
method
ordered keys
key expressions
key directions
null ordering
collations
included columns
predicate
persistent options
structural extensions
normalization version
```

---

# 97. Fingerprint profiles

```text
FULL
NAME_INDEPENDENT
PORTABLE
MIGRATION
CACHE
PLANNER_CATALOG
```

---

# 98. Fingerprint exclusions

No incluir:

```text
request ID
connection ID
query execution stats
current index size
current selectivity
runtime cost
last-used timestamp
telemetry span
```

---

# 99. Schema metadata ≠ runtime statistics

`IndexDefinition` describe estructura.

Esto:

```text
index size
cardinality estimate
selectivity
page count
usage count
cache hit ratio
```

pertenece a:

```text
Physical Catalog / Statistics / Telemetry
```

---

# 100. Planner catalog projection

Podrá existir:

```text
IndexDefinition
      │
      ▼
PhysicalCatalogIndexDescriptor
      │
      ├── structural capabilities
      ├── statistics
      └── planner properties
```

El planner consume la proyección.

No el schema object como objeto runtime mutable.

---

# 101. Serialization

La representación deberá ser versionada.

Ejemplo:

```json
{
  "name": "idx_users_email",
  "unique": false,
  "method": "DEFAULT",
  "keys": [
    {
      "kind": "column",
      "column": "email",
      "direction": "UNSPECIFIED",
      "nullOrdering": "UNSPECIFIED"
    }
  ],
  "includedColumns": [],
  "predicate": null
}
```

---

# 102. Serialization round-trip

Debe cumplirse:

```text
StructuralEqual(
    Index,
    Deserialize(Serialize(Index))
)
```

para formatos compatibles.

---

# 103. Extensions in serialization

Toda extensión estructural deberá conservar:

```text
extension ID
extension version
payload version
structural impact
```

---

# 104. Validation architecture

Se propone:

```php
interface IndexDefinitionValidator
{
    public function validate(
        IndexDefinition $index,
        IndexValidationContext $context,
    ): IndexValidationResult;
}
```

---

# 105. Local validation

Puede comprobar:

- índice tiene keys;
- no hay keys inválidas;
- no hay duplicate keys;
- included columns no se duplican;
- options no se contradicen;
- expression AST es válido;
- predicate AST es válido;
- method descriptor es válido;
- extensions son válidas.

---

# 106. Table-context validation

Puede comprobar:

```text
referenced columns exist
expression dependencies exist
included columns exist
index name uniqueness
column compatibility
```

---

# 107. Schema-context validation

Puede comprobar:

```text
namespace conflicts
extension dependencies
collation references
platform schema object dependencies
```

---

# 108. Platform validation

Comprueba:

```text
method support
partial index support
expression index support
included columns support
sort direction support
null ordering support
operator class support
persistent option support
```

---

# 109. Validation hierarchy

```text
Index Local Validation
        ↓
Table Context Validation
        ↓
Schema Context Validation
        ↓
Platform Compatibility
        ↓
Schema Operation Validation
        ↓
Migration Safety
        ↓
Execution Feasibility
```

---

# 110. IndexValidationResult

Se propone:

```php
final readonly class IndexValidationResult
{
    public function __construct(
        public bool $valid,
        public SchemaDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 111. Diagnostic examples

```text
Index "idx_users_email" references unknown column "email_address".

Index "idx_orders_created" contains duplicate key "created_at".

Index "idx_users_search" requires expression-index support.

Index "idx_active_users" uses a predicate unsupported by the selected platform.

Index "idx_users_covering" contains "email" as both key and included column.
```

---

# 112. ObservedIndex

Schema Introspection podrá producir:

```php
final readonly class ObservedIndex
{
    public function __construct(
        public IndexDefinition $definition,
        public DefinitionCompleteness $completeness,
        public IndexObservationMetadata $observation,
    ) {}
}
```

---

# 113. Introspection uncertainty

Un driver podría detectar:

```text
index exists
keys known
method unknown
predicate unavailable
options partially known
```

Eso no significa:

```text
method = DEFAULT
predicate = NONE
options = EMPTY
```

Debe conservarse incertidumbre.

---

# 114. UNKNOWN ≠ ABSENT

Regla:

```text
UNKNOWN
≠
ABSENT
≠
DEFAULT
```

---

# 115. Introspected platform names

Una plataforma puede devolver nombres generados automáticamente.

Debe conservarse provenance:

```text
PLATFORM_GENERATED
```

---

# 116. Index Diff integration

Schema Diff podrá producir:

```text
IndexDiff
├── IndexAdded
├── IndexRemoved
├── IndexRenamed
├── IndexKeysChanged
├── IndexUniquenessChanged
├── IndexMethodChanged
├── IndexPredicateChanged
├── IncludedColumnsChanged
├── IndexOptionsChanged
└── IndeterminateIndexChange
```

---

# 117. Index alteration portability

Muchas plataformas no soportan modificar un índice arbitrariamente.

Un cambio conceptual:

```text
IndexDefinition A
      ↓
IndexDefinition B
```

puede convertirse en:

```text
DROP INDEX
+
CREATE INDEX
```

pero esa decisión pertenece al Schema Planner/Migration layer.

---

# 118. Definition must not decide rebuild

Incorrecto:

```php
$index->requiresDropAndRecreate = true;
```

como propiedad universal.

Correcto:

```text
Schema Diff
      ↓
Change
      ↓
Platform Planner
      ↓
Drop/Create strategy
```

---

# 119. Safety classification

Los cambios de índice podrán clasificarse posteriormente como:

```text
ONLINE_CAPABLE
LOCK_RISK
LONG_RUNNING
RESOURCE_INTENSIVE
NON_TRANSACTIONAL
PLATFORM_DEPENDENT
```

Pero eso pertenece principalmente al change/migration planning.

---

# 120. Index creation cost ≠ definition

La definición no deberá contener:

```text
estimated creation seconds
estimated memory
estimated lock duration
```

Es información operacional.

---

# 121. Index expressions and Query AST

Conviene evitar reutilización accidental del Query AST completo.

Puede existir un:

```text
SchemaExpression
```

compartido en una capa de expresiones común cuidadosamente diseñada.

Pero:

```text
IndexExpression
≠
Arbitrary Query
```

---

# 122. Expression safety

Las expresiones de índices deberán limitarse al subconjunto permitido por schema semantics.

---

# 123. No subqueries by default

Por defecto:

```text
IndexExpression
```

no deberá aceptar subqueries salvo capability/extension explícita.

---

# 124. No runtime parameters

Un índice persistente no deberá contener:

```text
ParameterId
ExecutionParameterSlot
RuntimeBinding
```

---

# 125. Deterministic expressions

Cuando la plataforma exija expresiones deterministas:

```text
ExpressionDeterminismRequirement
```

deberá declararse y validarse.

---

# 126. Security

Los nombres de índices, columnas, operator classes y otros identifiers deberán pasar por:

```text
typed identifier
+
validation
+
compiler quoting
```

---

# 127. Raw SQL protection

Nunca:

```php
$table->index($_GET['index_sql']);
```

como SQL raw implícito.

---

# 128. Raw extension boundary

Una API raw deberá ser explícita:

```php
RawIndexExpression::trusted(...)
```

o equivalente.

---

# 129. Raw does not mean safe

Debe mantenerse:

```text
RAW
≠
SAFE
≠
PORTABLE
```

---

# 130. Extension architecture

Se propone:

```text
IndexExtension
├── CustomIndexMethod
├── CustomIndexKey
├── CustomIndexOption
├── CustomOperatorClass
├── CustomMetadata
└── CustomCompilerContribution
```

---

# 131. Frozen extension registry

La interpretación de extensiones dependerá de:

```text
FrozenIndexExtensionRegistry
```

---

# 132. No last-wins

Si dos extensiones reclaman la misma semántica:

```text
ExtensionConflict
```

No:

```text
last registration wins
```

---

# 133. Unknown structural extension

No podrá ignorarse durante:

```text
comparison
fingerprinting
serialization
diff
compilation
```

---

# 134. Persistent runtime safety

`IndexDefinition` deberá ser:

```text
immutable
connection-free
statement-free
request-free
transaction-free
tenant-state-free
```

---

# 135. Safe cacheability

Una definición canónica podrá cachearse.

No deberá cachearse dentro de ella:

```text
current statistics
current connection
current tenant
current planner cost
```

---

# 136. Tenant context

Un índice puede formar parte del schema de un tenant.

Pero:

```text
IndexDefinition
```

no deberá contener un mutable:

```text
TenantContext
```

La pertenencia se determina externamente mediante:

```text
SchemaIdentity
SchemaSnapshot
TenantSchemaContext
```

---

# 137. Runtime neutrality

El sistema deberá funcionar igual conceptualmente bajo:

```text
PHP-FPM
FrankenPHP
RoadRunner
OpenSwoole
CLI
Testing Runtime
```

---

# 138. Index definition budget

Se propone:

```php
final readonly class IndexDefinitionBudget
{
    public function __construct(
        public int $maxKeys,
        public int $maxIncludedColumns,
        public int $maxExpressionDepth,
        public int $maxPredicateDepth,
        public int $maxOptions,
        public int $maxExtensions,
    ) {}
}
```

---

# 139. Budget overflow

Debe producir error explícito.

Nunca truncar:

```text
INDEX(a,b,c,d)
```

a:

```text
INDEX(a,b,c)
```

---

# 140. Index system services

Servicios principales:

```text
IndexDefinitionFactory
IndexDefinitionNormalizer
IndexDefinitionValidator
IndexDefinitionComparator
IndexFingerprintGenerator
IndexNamingStrategy
IndexCapabilityRequirementResolver
IndexDefinitionSerializer
IndexDefinitionDeserializer
IndexExtensionRegistry
```

---

# 141. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Index
```

---

# 142. Proposed directory structure

```text
Schema/
└── Index/
    ├── Contract/
    │   ├── IndexDefinitionFactory.php
    │   ├── IndexDefinitionNormalizer.php
    │   ├── IndexDefinitionValidator.php
    │   ├── IndexDefinitionComparator.php
    │   └── IndexNamingStrategy.php
    │
    ├── Definition/
    │   ├── IndexDefinition.php
    │   ├── IndexDefinitionId.php
    │   └── IndexUniqueness.php
    │
    ├── Identifier/
    │   ├── IndexIdentifier.php
    │   └── QualifiedIndexIdentifier.php
    │
    ├── Key/
    │   ├── IndexKey.php
    │   ├── IndexKeySet.php
    │   ├── ColumnIndexKey.php
    │   ├── ExpressionIndexKey.php
    │   ├── ExtensionIndexKey.php
    │   ├── IndexSortDirection.php
    │   ├── IndexNullOrdering.php
    │   └── IndexKeyOptionSet.php
    │
    ├── Include/
    │   └── IncludedColumnSet.php
    │
    ├── Predicate/
    │   └── IndexPredicate.php
    │
    ├── Method/
    │   ├── IndexMethod.php
    │   ├── DefaultIndexMethod.php
    │   ├── BTreeIndexMethod.php
    │   ├── HashIndexMethod.php
    │   └── ExtensionIndexMethod.php
    │
    ├── Option/
    │   ├── IndexOption.php
    │   ├── IndexOptionSet.php
    │   ├── PortableIndexOption.php
    │   ├── PlatformIndexOption.php
    │   └── ExtensionIndexOption.php
    │
    ├── OperatorClass/
    │   └── IndexOperatorClass.php
    │
    ├── Metadata/
    │   ├── IndexMetadata.php
    │   ├── ObservedIndex.php
    │   └── IndexObservationMetadata.php
    │
    ├── Capability/
    │   └── IndexCapabilityRequirementResolver.php
    │
    ├── Naming/
    │   ├── DefaultIndexNamingStrategy.php
    │   └── GeneratedIndexName.php
    │
    ├── Validation/
    │   ├── IndexValidationContext.php
    │   └── IndexValidationResult.php
    │
    ├── Comparison/
    │   ├── IndexComparisonResult.php
    │   └── IndexComparisonProfile.php
    │
    ├── Fingerprint/
    │   ├── IndexFingerprintGenerator.php
    │   ├── IndexStructuralFingerprint.php
    │   └── IndexFingerprintProfile.php
    │
    ├── Serialization/
    │   ├── IndexDefinitionSerializer.php
    │   └── IndexDefinitionDeserializer.php
    │
    ├── Extension/
    │   ├── IndexExtension.php
    │   └── FrozenIndexExtensionRegistry.php
    │
    ├── Budget/
    │   └── IndexDefinitionBudget.php
    │
    └── Exception/
        ├── DatabaseIndexException.php
        ├── InvalidIndexDefinitionException.php
        ├── InvalidIndexIdentifierException.php
        ├── EmptyIndexKeySetException.php
        ├── DuplicateIndexKeyException.php
        ├── UnknownIndexColumnException.php
        ├── InvalidIndexExpressionException.php
        ├── InvalidIndexPredicateException.php
        ├── InvalidIndexMethodException.php
        ├── UnsupportedIndexCapabilityException.php
        ├── ConflictingIndexOptionException.php
        ├── IndexSerializationException.php
        ├── IndexExtensionException.php
        ├── IndexDefinitionBudgetExceededException.php
        └── IndexInvariantException.php
```

---

# 143. Error hierarchy

Arquitectónicamente:

```text
DatabaseIndexException
├── InvalidIndexDefinitionException
├── InvalidIndexIdentifierException
├── EmptyIndexKeySetException
├── DuplicateIndexKeyException
├── UnknownIndexColumnException
├── DuplicateIncludedColumnException
├── IndexKeyIncludeConflictException
├── InvalidIndexExpressionException
├── InvalidIndexPredicateException
├── InvalidIndexMethodException
├── InvalidIndexOperatorClassException
├── InvalidIndexOptionException
├── ConflictingIndexOptionException
├── UnsupportedIndexCapabilityException
├── IndexSerializationException
├── IndexExtensionException
├── IndexDefinitionBudgetExceededException
└── IndexInvariantException
```

---

# 144. Testing strategy

La mayoría del Index System deberá probarse sin una DB real.

---

# 145. Definition tests

Probar:

```text
single-column index
composite index
unique index
expression index
partial index
included columns
ordered keys
custom methods
extensions
```

---

# 146. Ordering tests

Verificar:

```text
INDEX(a,b) ≠ INDEX(b,a)
```

y:

```text
INDEX(a ASC) ≠ INDEX(a DESC)
```

cuando la dirección sea estructuralmente significativa.

---

# 147. Include tests

Verificar:

```text
key(a) + include(b)
```

no sea igual a:

```text
key(a,b)
```

---

# 148. Predicate tests

Verificar:

- predicate normalization;
- dependency extraction;
- unsupported constructs;
- raw predicate handling;
- capability requirements.

---

# 149. Expression tests

Verificar:

- structured expressions;
- dependencies;
- deterministic restrictions;
- extension expressions;
- raw expressions.

---

# 150. Fingerprint tests

Debe cumplirse:

```text
Same Canonical Index
+
Same Fingerprint Profile
=
Same Fingerprint
```

---

# 151. Serialization tests

```text
Index
  ↓ serialize
Payload
  ↓ deserialize
Index'
```

y:

```text
StructuralEqual(Index, Index')
```

---

# 152. Normalization tests

Debe cumplirse:

```text
Normalize(Normalize(I))
=
Normalize(I)
```

---

# 153. Introspection tests

Para cada plataforma:

```text
Create
  ↓
Introspect
  ↓
ObservedIndex
```

y comparar contra la definición esperada bajo un profile de platform equivalence.

---

# 154. Cross-platform conformance

La suite deberá cubrir al menos:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 155. Planner isolation tests

Verificar que:

```text
IndexDefinition
```

no contenga:

```text
query cost
selectivity
chosen scan
execution statistics
```

---

# 156. Persistent runtime tests

Ejecutar múltiples schema operations sobre el mismo worker y verificar ausencia de state leakage.

---

# 157. Architectural invariants

## DB-INDEX-001

Todo índice tendrá identidad estructurada.

## DB-INDEX-002

`IndexDefinition` será inmutable.

## DB-INDEX-003

`IndexDefinition` será distinto de `IndexBlueprint`.

## DB-INDEX-004

`IndexDefinition` será distinto de `IndexOperation`.

## DB-INDEX-005

`IndexDefinition` será distinto de SQL.

## DB-INDEX-006

`IndexDefinition` será distinto de `UniqueConstraintDefinition`.

## DB-INDEX-007

`IndexDefinition` será distinto de `PrimaryKeyDefinition`.

## DB-INDEX-008

`IndexDefinition` será distinto de `ForeignKeyDefinition`.

## DB-INDEX-009

`IndexDefinition` será distinto de Query `AccessPath`.

## DB-INDEX-010

`IndexDefinition` será distinto de `PhysicalIndexScan`.

## DB-INDEX-011

Index System no generará SQL directamente.

## DB-INDEX-012

Index System no ejecutará DDL.

## DB-INDEX-013

Index System no abrirá conexiones.

## DB-INDEX-014

Index System no decidirá qué índice usa una query.

## DB-INDEX-015

Index System no estimará costos de query.

## DB-INDEX-016

Index System no almacenará runtime statistics.

## DB-INDEX-017

`IndexIdentifier` será tipado.

## DB-INDEX-018

Identifier quoting pertenecerá al compiler.

## DB-INDEX-019

`IndexDefinitionId` será distinto de `IndexIdentifier`.

## DB-INDEX-020

Todo índice tendrá al menos una key salvo extensión explícita.

## DB-INDEX-021

Index keys serán ordenadas.

## DB-INDEX-022

`INDEX(a,b)` será distinto de `INDEX(b,a)`.

## DB-INDEX-023

`IndexKey` será tipado.

## DB-INDEX-024

Column keys serán distintas de expression keys.

## DB-INDEX-025

Raw expression keys serán explícitas.

## DB-INDEX-026

Expression dependencies serán representables.

## DB-INDEX-027

Referenced columns deberán existir.

## DB-INDEX-028

Duplicate keys no serán aceptadas silenciosamente.

## DB-INDEX-029

Sort direction será estructurada.

## DB-INDEX-030

Null ordering será estructurado.

## DB-INDEX-031

UNSPECIFIED no será colapsado prematuramente.

## DB-INDEX-032

Per-key collation será distinta de table collation.

## DB-INDEX-033

Included columns serán distintas de index keys.

## DB-INDEX-034

Included columns preservarán su orden/canonical semantics según contrato.

## DB-INDEX-035

Duplicate included columns serán inválidas.

## DB-INDEX-036

Key/include conflicts serán diagnosticados.

## DB-INDEX-037

Included columns no implicarán IndexOnlyScan.

## DB-INDEX-038

Partial-index predicate será estructurado.

## DB-INDEX-039

Raw partial predicate será explícito.

## DB-INDEX-040

Predicate dependencies serán representables.

## DB-INDEX-041

Predicate support será capability-driven.

## DB-INDEX-042

Expression-index support será capability-driven.

## DB-INDEX-043

Included-column support será capability-driven.

## DB-INDEX-044

Index method será explícito cuando no sea default.

## DB-INDEX-045

DEFAULT method no significará universalmente BTREE.

## DB-INDEX-046

Index methods serán capability-driven.

## DB-INDEX-047

Platform-specific methods serán explícitos.

## DB-INDEX-048

Operator classes serán tipadas cuando se soporten.

## DB-INDEX-049

Operator classes no serán generic option strings.

## DB-INDEX-050

Persistent index properties serán distintas de creation strategies.

## DB-INDEX-051

Concurrent creation no será propiedad estructural por defecto.

## DB-INDEX-052

Online creation no será propiedad estructural por defecto.

## DB-INDEX-053

Index visibility podrá ser estructural cuando persista.

## DB-INDEX-054

Index visibility no garantizará planner behavior.

## DB-INDEX-055

Index options serán tipadas.

## DB-INDEX-056

Unknown structural options no serán ignoradas.

## DB-INDEX-057

Conflicting options fallarán.

## DB-INDEX-058

Platform options serán explícitas.

## DB-INDEX-059

Capability requirements serán declarativos.

## DB-INDEX-060

Capability requirements serán distintos de support checks.

## DB-INDEX-061

IndexDefinition no consultará capabilities ocultamente.

## DB-INDEX-062

Platform compatibility será downstream.

## DB-INDEX-063

Index naming será determinista.

## DB-INDEX-064

Generated naming pertenecerá a naming strategy.

## DB-INDEX-065

Explicit names no serán silenciosamente renombrados.

## DB-INDEX-066

Name truncation deberá ser collision-aware.

## DB-INDEX-067

Canonicalization será determinista.

## DB-INDEX-068

Canonicalization será idempotente.

## DB-INDEX-069

Canonicalization será semantics-preserving.

## DB-INDEX-070

Canonicalization no será SQL compilation.

## DB-INDEX-071

Canonicalization no consultará DB ocultamente.

## DB-INDEX-072

Structural equality será distinta de identifier equality.

## DB-INDEX-073

Structural equality será distinta de platform equivalence.

## DB-INDEX-074

Structural equality será distinta de migration equivalence.

## DB-INDEX-075

Structural equality será distinta de planner equivalence.

## DB-INDEX-076

Similarity no probará rename.

## DB-INDEX-077

Rename intent pertenecerá a Schema Operations.

## DB-INDEX-078

Fingerprint será estructural.

## DB-INDEX-079

Fingerprint no será SQL hash.

## DB-INDEX-080

Fingerprint será determinista.

## DB-INDEX-081

Fingerprint tendrá algorithm version.

## DB-INDEX-082

Fingerprint excluirá runtime statistics.

## DB-INDEX-083

Fingerprint excluirá connection identity.

## DB-INDEX-084

Fingerprint excluirá request identity.

## DB-INDEX-085

Serialization será versionada.

## DB-INDEX-086

Serialization será determinista.

## DB-INDEX-087

Serialization preservará structural extensions.

## DB-INDEX-088

Serialization no incluirá live resources.

## DB-INDEX-089

ObservedIndex será distinto de IndexDefinition.

## DB-INDEX-090

Observation uncertainty será explícita.

## DB-INDEX-091

UNKNOWN será distinto de ABSENT.

## DB-INDEX-092

UNKNOWN será distinto de DEFAULT.

## DB-INDEX-093

Introspection no inventará metadata faltante.

## DB-INDEX-094

Platform-generated names preservarán provenance.

## DB-INDEX-095

Schema Diff podrá comparar índices estructuralmente.

## DB-INDEX-096

Schema Diff preservará uncertainty.

## DB-INDEX-097

IndexDefinition no decidirá drop/recreate strategy.

## DB-INDEX-098

IndexDefinition no decidirá rebuild strategy.

## DB-INDEX-099

IndexDefinition no decidirá migration safety.

## DB-INDEX-100

IndexDefinition no decidirá lock strategy.

## DB-INDEX-101

IndexDefinition no decidirá transaction strategy.

## DB-INDEX-102

IndexDefinition no decidirá concurrent DDL.

## DB-INDEX-103

IndexDefinition no contendrá ORM entity metadata.

## DB-INDEX-104

IndexDefinition no contendrá UnitOfWork state.

## DB-INDEX-105

IndexDefinition no contendrá QueryExecution state.

## DB-INDEX-106

IndexDefinition no contendrá planner cost.

## DB-INDEX-107

IndexDefinition no contendrá query selectivity.

## DB-INDEX-108

IndexDefinition no contendrá usage telemetry.

## DB-INDEX-109

IndexDefinition no contendrá PDO.

## DB-INDEX-110

IndexDefinition no contendrá Connection.

## DB-INDEX-111

IndexDefinition no contendrá DriverStatement.

## DB-INDEX-112

IndexDefinition no contendrá Transaction.

## DB-INDEX-113

IndexDefinition no contendrá Request.

## DB-INDEX-114

IndexDefinition no contendrá mutable TenantContext.

## DB-INDEX-115

IndexDefinition será persistent-runtime safe.

## DB-INDEX-116

Extensions serán tipadas.

## DB-INDEX-117

Extensions tendrán stable IDs.

## DB-INDEX-118

Extension registry será frozen.

## DB-INDEX-119

No existirá last-wins extension behavior.

## DB-INDEX-120

Extension conflicts serán explícitos.

## DB-INDEX-121

Unknown structural extensions no se ignorarán.

## DB-INDEX-122

Raw expressions serán explícitas.

## DB-INDEX-123

RAW no implicará SAFE.

## DB-INDEX-124

RAW no implicará PORTABLE.

## DB-INDEX-125

Identifiers serán validados.

## DB-INDEX-126

Identifiers serán quoted por compiler.

## DB-INDEX-127

Index expressions no contendrán runtime parameters.

## DB-INDEX-128

Index predicates no contendrán runtime bindings.

## DB-INDEX-129

Expression restrictions serán explícitas.

## DB-INDEX-130

IndexDefinition será testeable sin database.

## DB-INDEX-131

Cross-platform compiler behavior será testeable por conformance suites.

## DB-INDEX-132

Index introspection será platform-aware pero normalizada.

## DB-INDEX-133

Definition budgets serán explícitos.

## DB-INDEX-134

Budget overflow no truncará definiciones.

## DB-INDEX-135

UniqueIndex será distinto de UniqueConstraint.

## DB-INDEX-136

PrimaryKey nunca será reducido conceptualmente a index.

## DB-INDEX-137

ForeignKey nunca será reducido conceptualmente a index.

## DB-INDEX-138

Un índice implícito creado por plataforma preservará provenance.

## DB-INDEX-139

Platform physical implementation no redefinirá logical schema intent.

## DB-INDEX-140

IndexDefinition podrá proyectarse hacia Physical Catalog.

## DB-INDEX-141

Physical Catalog podrá añadir statistics sin mutar IndexDefinition.

## DB-INDEX-142

Planner podrá consumir index descriptors sin mutar Schema Model.

## DB-INDEX-143

Chosen access path no se escribirá en IndexDefinition.

## DB-INDEX-144

Execution feedback no se escribirá en IndexDefinition.

## DB-INDEX-145

IndexDefinition describirá persistent structure.

## DB-INDEX-146

Schema operation describirá structural transition.

## DB-INDEX-147

Physical plan describirá execution strategy.

## DB-INDEX-148

Estas tres representaciones permanecerán separadas.

## DB-INDEX-149

Index System será runtime-neutral.

## DB-INDEX-150

`IndexDefinition` será la representación canónica de índices persistentes dentro del Schema System de VoltStack.

---

# 158. Anti-patterns

## 158.1 Index as string

Incorrecto:

```php
$index = 'INDEX idx_email (email)';
```

---

## 158.2 Index as SQL fragment

Incorrecto:

```php
$index->sql = 'CREATE INDEX idx_email ON users(email)';
```

---

## 158.3 Constraint/index collapse

Incorrecto:

```php
$table->primaryIndex('id');
```

si internamente convierte la primary key únicamente en un índice.

---

## 158.4 Planner state inside schema

Incorrecto:

```php
$index->cost = 2.4;
$index->selectivity = 0.01;
$index->chosen = true;
```

---

## 158.5 Vendor conditionals

Incorrecto:

```php
if ($driver === 'pgsql') {
    $index->method = 'gin';
}
```

en el modelo central.

---

## 158.6 Generic option bags

Incorrecto:

```php
$index->options = [
    'method' => 'gin',
    'where' => 'active = true',
    'include' => ['name'],
];
```

como representación canónica.

---

## 158.7 Included columns as keys

Incorrecto:

```text
keys(email, id, name)
```

si `id` y `name` eran solamente covering columns.

---

## 158.8 Unique index as universal unique constraint

Incorrecto:

```text
unique index
=
unique constraint
```

---

## 158.9 Introspection uncertainty collapse

Incorrecto:

```text
predicate unavailable
⇒ no predicate
```

---

## 158.10 Hidden DB lookup

Incorrecto:

```php
$index->normalize();
```

si internamente consulta el servidor para descubrir el método default.

---

# 159. Ejemplo: índice simple

```php
$index = new IndexDefinition(
    identifier: IndexIdentifier::from('idx_users_email'),

    kind: IndexKind::STANDARD,

    keys: IndexKeySet::of(
        new ColumnIndexKey(
            column: ColumnIdentifier::from('email'),
            direction: IndexSortDirection::UNSPECIFIED,
            nullOrdering: IndexNullOrdering::UNSPECIFIED,
            collation: null,
            options: IndexKeyOptionSet::empty(),
        ),
    ),

    includedColumns: IncludedColumnSet::empty(),

    predicate: null,

    options: IndexOptionSet::empty(),

    capabilities: SchemaCapabilityRequirementSet::empty(),

    metadata: IndexMetadata::declared(),

    extensions: ExtensionMetadataSet::empty(),
);
```

Representación:

```text
IndexDefinition(idx_users_email)
│
├── uniqueness
│   └── NON_UNIQUE
│
├── method
│   └── DEFAULT
│
├── keys
│   └── email
│
├── included
│   └── ∅
│
├── predicate
│   └── ∅
│
└── options
    └── ∅
```

---

# 160. Ejemplo: índice compuesto

```text
idx_orders_tenant_created
│
├── tenant_id ASC
└── created_at DESC
```

Formalmente:

```text
IndexKeySet
├── ColumnIndexKey
│   ├── tenant_id
│   └── ASC
└── ColumnIndexKey
    ├── created_at
    └── DESC
```

---

# 161. Ejemplo: índice de expresión

```text
idx_users_lower_email
│
└── ExpressionIndexKey
    │
    └── LOWER(email)
```

Dependencies:

```text
LOWER(email)
     │
     ▼
   email
```

Capability:

```text
EXPRESSION_INDEX
```

---

# 162. Ejemplo: índice parcial

```text
idx_users_active_email
│
├── key
│   └── email
│
└── predicate
    └── deleted_at IS NULL
```

Capability:

```text
PARTIAL_INDEX
```

---

# 163. Ejemplo: covering index

```text
idx_orders_customer
│
├── keys
│   └── customer_id
│
└── include
    ├── total
    ├── status
    └── created_at
```

El Query Planner podrá posteriormente observar que determinadas queries pueden satisfacerse con mayor cobertura.

Pero el Schema System no toma esa decisión.

---

# 164. Ejemplo combinado

```text
IndexDefinition(idx_orders_active_customer)
│
├── uniqueness
│   └── NON_UNIQUE
│
├── method
│   └── DEFAULT
│
├── keys
│   ├── customer_id ASC
│   └── created_at DESC
│
├── include
│   ├── total
│   └── status
│
├── predicate
│   └── deleted_at IS NULL
│
└── capabilities
    ├── INCLUDED_COLUMNS
    ├── PARTIAL_INDEX
    └── DESCENDING_INDEX_KEY
```

---

# 165. Query Planner integration

El flujo deberá ser:

```text
Schema Model
    │
    ▼
IndexDefinition
    │
    ▼
Physical Catalog Projection
    │
    ├── Index Structure
    ├── Platform Capabilities
    └── Statistics Snapshot
             │
             ▼
       AccessPath Discovery
             │
             ▼
       Physical Planner
             │
             ▼
       PhysicalIndexScan?
```

Nunca:

```text
IndexDefinition
      ↓
execute this index
```

---

# 166. Formula principal

```text
Database Index System
=
Persistent Index Identity
+
Ordered Index Keys
+
Expression Keys
+
Included Columns
+
Uniqueness Semantics
+
Index Method
+
Partial Predicate
+
Ordering Semantics
+
Platform Options
+
Capability Requirements
+
Metadata
+
Canonicalization
+
Validation
+
Comparison
+
Fingerprinting
+
Serialization
+
Extension Safety
+
Persistent Runtime Safety
```

---

# 167. Correctness formula

Para un índice `I`:

```text
ValidIndex(I)
=
ValidIdentifier(I)
∧ NonEmptyKeys(I)
∧ ValidKeyReferences(I)
∧ ValidExpressions(I)
∧ ValidPredicate(I)
∧ ValidIncludedColumns(I)
∧ CompatibleMethod(I)
∧ CompatibleOptions(I)
∧ ValidExtensions(I)
```

La compatibilidad de plataforma será una condición adicional:

```text
Compilable(I, P)
=
ValidIndex(I)
∧ Capabilities(P) ⊇ Requirements(I)
```

---

# 168. Separation formula

```text
IndexDefinition
=
Persistent Structural Intent
```

```text
IndexOperation
=
Structural Transition
```

```text
IndexAccessPath
=
Planning Alternative
```

```text
PhysicalIndexScan
=
Chosen Physical Execution Strategy
```

Por tanto:

```text
IndexDefinition
≠
IndexOperation
≠
IndexAccessPath
≠
PhysicalIndexScan
```

---

# 169. Integración con los documentos anteriores

La relación dentro del Block 8 queda:

```text
87 Schema Architecture
        │
        ▼
88 Schema Model
        │
        ▼
89 Schema AST
        │
        ▼
90 Schema Builder
        │
        ▼
91 Table Definition
        │
        ├───────────────┐
        ▼               ▼
92 Column System    93 Index System
```

`TableDefinition` actuará como aggregate estructural:

```text
TableDefinition
├── ColumnDefinitionSet
├── IndexDefinitionSet
├── ForeignKeyDefinitionSet
└── ConstraintDefinitionSet
```

sin colapsar esas categorías.

---

# 170. Decisión arquitectónica final

VoltStack adoptará la siguiente regla:

> **Un índice es una estructura persistente de acceso declarada por el Schema System. No es una constraint lógica, no es SQL y no es una decisión del Query Planner.**

La arquitectura completa será:

```text
Developer DSL
     │
     ▼
IndexBlueprint
     │
     ▼
IndexDefinition
     │
     ├──────────────► Schema Model
     │
     ├──────────────► Schema Diff
     │
     └──────────────► Physical Catalog Projection
     │                        │
     ▼                        ▼
Schema AST              Query Planner
     │                        │
     ▼                        ▼
Schema Planner          AccessPath
     │                        │
     ▼                        ▼
Schema Compiler        PhysicalIndexScan
     │
     ▼
Compiled DDL
     │
     ▼
Execution Engine
```

Con ello VoltStack mantiene separados los cuatro dominios fundamentales:

```text
Schema Structure
Schema Change
Query Planning
Query Execution
```

y evita uno de los errores arquitectónicos más frecuentes en sistemas de base de datos:

```text
Database has an index
        ≠
Query can use that index
        ≠
Planner should use that index
        ≠
Database will actually use that index
```

---

# 171. Resultado arquitectónico

`Database Index System` proporciona a VoltStack una base capaz de representar desde índices portables sencillos:

```text
INDEX(email)
```

hasta estructuras avanzadas:

```text
INDEX
├── expression keys
├── composite keys
├── ordering
├── partial predicate
├── included columns
├── custom method
├── operator classes
├── platform options
└── extensions
```

sin sacrificar:

```text
Portability
Determinism
Semantic clarity
Platform specialization
Extensibility
Schema Diff compatibility
Planner separation
Persistent-runtime safety
```

La separación definitiva queda:

```text
                ┌───────────────────┐
                │  IndexDefinition  │
                └─────────┬─────────┘
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
    Schema Compiler   Schema Diff    Physical Catalog
          │               │                │
          ▼               ▼                ▼
         DDL          Change Plan      Query Planner
                                            │
                                            ▼
                                      AccessPath
                                            │
                                            ▼
                                    PhysicalIndexScan
```

---

# 172. Siguiente documento

```text
94_DATABASE_FOREIGN_KEY_SYSTEM.md
```

El siguiente documento formalizará:

```text
ForeignKeyDefinition
├── ForeignKeyIdentifier
├── LocalColumnSet
├── ReferencedTableIdentifier
├── ReferencedColumnSet
├── MatchSemantics
├── OnUpdateAction
├── OnDeleteAction
├── Deferrability
├── ValidationState
├── CapabilityRequirements
├── Metadata
└── ExtensionMetadata
```

y establecerá especialmente:

```text
ForeignKey
≠
Index
≠
Relationship
≠
ORM Association
≠
Join
≠
Authorization Boundary
```

Principio del siguiente documento:

> **Una foreign key representa una regla estructural de integridad referencial entre datos persistentes; no una relación ORM ni una estrategia de JOIN.**