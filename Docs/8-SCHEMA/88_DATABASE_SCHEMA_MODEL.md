# 88_DATABASE_SCHEMA_MODEL.md

# VoltStack Quantum Database
## Database Schema Model

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 88 — Database Schema Model  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Model` define la representación estructural canónica mediante la cual VoltStack describe el estado lógico conocido de una base de datos.

El modelo deberá representar:

```text
DatabaseSchema
├── Catalog
├── Namespace
├── Table
│   ├── Column
│   ├── PrimaryKey
│   ├── Index
│   ├── UniqueConstraint
│   ├── CheckConstraint
│   └── ForeignKey
├── Sequence
├── View
└── ExtensionObject
```

sin depender directamente de:

```text
SQL
PDO
Connection
Driver Handle
Migration
ORM Entity
Query Builder
Execution Engine
```

Principio fundamental:

```text
Schema Model
=
Immutable Structural Knowledge
```

y:

```text
Schema Model
≠
DDL
≠
Schema Operation
≠
Migration
≠
Execution
```

---

# 2. Objetivo arquitectónico

El modelo debe permitir representar de forma estable:

```text
what database structure exists
```

o:

```text
what database structure is desired
```

sin mezclar ambas cosas con la operación necesaria para llegar de un estado al otro.

Por ejemplo:

```text
CurrentSchema
```

y:

```text
TargetSchema
```

pueden ser dos instancias del mismo modelo.

Posteriormente:

```text
Diff(CurrentSchema, TargetSchema)
```

determinará las transformaciones.

---

# 3. Modelo de estado

Formalmente:

```text
SchemaModel = S
```

donde `S` representa un estado estructural.

Una operación:

```text
O
```

puede transformar:

```text
S₀
```

en:

```text
S₁
```

tal que:

```text
S₁ = Apply(O, S₀)
```

Pero:

```text
O ∉ S
```

La operación no forma parte del estado.

---

# 4. Schema Model vs Schema AST

Esta separación será obligatoria:

```text
Schema Model
=
structural state
```

```text
Schema AST
=
structural operation intent
```

Ejemplo de modelo:

```text
users
├── id BIGINT
├── name STRING
└── email STRING
```

Ejemplo de AST:

```text
CreateTable(users)
AddColumn(id)
AddColumn(name)
AddColumn(email)
```

---

# 5. Schema Model vs Migration

Una migration representa:

```text
managed evolution
```

El Schema Model representa:

```text
structural state
```

Por tanto:

```text
SchemaModel
≠
Migration
```

---

# 6. Schema Model vs metadata

También debemos distinguir:

```text
Schema Definition
```

de:

```text
Schema Metadata
```

y ambos del:

```text
Schema Model
```

Una definición puede describir la estructura deseada.

Metadata puede describir información observada.

El modelo canónico puede integrar ambas mediante objetos estructurales y metadata adjunta.

---

# 7. Raíz del modelo

El objeto raíz será conceptualmente:

```php
final readonly class DatabaseSchema
{
    public function __construct(
        public DatabaseSchemaId $id,
        public SchemaObjectCollection $objects,
        public SchemaMetadataSet $metadata,
        public SchemaModelVersion $version,
    ) {}
}
```

---

# 8. DatabaseSchemaId

Debe distinguirse:

```text
DatabaseSchemaId
≠
DatabaseName
≠
ConnectionName
≠
CatalogName
≠
SchemaFingerprint
```

`DatabaseSchemaId` es identidad interna del artefacto.

---

# 9. SchemaFingerprint

El contenido estructural podrá tener:

```text
SchemaFingerprint
```

Por tanto:

```text
Identity
≠
Content Fingerprint
```

Dos instancias diferentes:

```text
Schema A
Schema B
```

pueden tener IDs distintos pero el mismo fingerprint estructural.

---

# 10. Modelo jerárquico

Conceptualmente:

```text
DatabaseSchema
│
├── Catalog
│   │
│   └── Namespace
│       │
│       ├── Table
│       ├── Sequence
│       ├── View
│       └── ExtensionObject
│
└── Metadata
```

Sin embargo, no todas las plataformas implementan estos niveles de la misma manera.

---

# 11. Catalog ≠ Namespace ≠ Database target

Distinción:

```text
Catalog
≠
Schema Namespace
≠
Connection Database Target
```

Esto es especialmente importante para portabilidad entre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 12. PostgreSQL

Por ejemplo:

```text
database
└── schema
    └── table
```

puede corresponder a:

```text
Connection Target
└── Namespace
    └── Table
```

---

# 13. MySQL/MariaDB

En MySQL/MariaDB, el concepto comúnmente denominado:

```text
database
```

se comporta en varios contextos de forma similar a un schema namespace.

VoltStack no deberá concluir por ello:

```text
Catalog == Schema == Database
```

como verdad universal.

---

# 14. SQLite

SQLite tiene un modelo diferente:

```text
main
temp
attached databases
```

y deberá mapearse mediante su propio Platform Model.

---

# 15. QualifiedSchemaName

Los nombres cualificados deberán modelarse estructuralmente.

```php
final readonly class QualifiedSchemaObjectName
{
    public function __construct(
        public Identifier $localName,
        public ?SchemaIdentifier $schema = null,
        public ?CatalogIdentifier $catalog = null,
    ) {}
}
```

---

# 16. No raw qualified names

No deberá almacenarse internamente:

```php
"catalog.schema.users"
```

como identidad estructural principal.

Preferible:

```text
catalog = catalog
schema  = schema
name    = users
```

---

# 17. Identifier model

Todo identificador estructural deberá usar:

```text
Identifier
```

en lugar de strings SQL.

Ejemplo:

```php
final readonly class Identifier
{
    public function __construct(
        public string $value,
    ) {}
}
```

con validación apropiada.

---

# 18. Identifier ≠ SQL identifier text

El modelo almacena:

```text
users
```

No:

```text
"users"
```

ni:

```text
`users`
```

El quoting pertenece al compiler/dialect.

---

# 19. Identifier semantics

El sistema deberá poder conservar:

```text
original identifier
normalized identifier
comparison identity
```

cuando una plataforma lo requiera.

---

# 20. Case semantics

Las plataformas pueden aplicar:

```text
case folding
case sensitivity
quoted case preservation
filesystem-dependent behavior
```

por lo que:

```text
User
user
USER
```

no deberán compararse mediante un simple:

```php
strtolower()
```

universal.

---

# 21. IdentifierNormalizer

Se propone:

```php
interface IdentifierNormalizer
{
    public function normalize(
        Identifier $identifier,
        IdentifierNormalizationContext $context,
    ): NormalizedIdentifier;
}
```

---

# 22. Platform-neutral model

El modelo no deberá introducir:

```php
if ($database === 'postgresql')
```

para determinar identidad.

Recibirá políticas de comparación/resolución apropiadas.

---

# 23. SchemaObject

Todos los objetos estructurales principales podrán implementar:

```php
interface SchemaObject
{
    public function id(): SchemaObjectId;

    public function name(): QualifiedSchemaObjectName;

    public function kind(): SchemaObjectKind;

    public function metadata(): SchemaObjectMetadata;
}
```

---

# 24. SchemaObjectKind

Ejemplo:

```php
enum SchemaObjectKind: string
{
    case TABLE = 'table';
    case SEQUENCE = 'sequence';
    case VIEW = 'view';
    case MATERIALIZED_VIEW = 'materialized_view';
    case NATIVE_TYPE = 'native_type';
    case EXTENSION = 'extension';
}
```

---

# 25. Table children

Objetos como:

```text
Column
Index
Constraint
ForeignKey
```

pueden tener identidad propia sin necesariamente ser objetos raíz del namespace.

---

# 26. Table model

Conceptualmente:

```php
final readonly class Table
{
    public function __construct(
        public TableId $id,
        public TableIdentifier $name,
        public ColumnCollection $columns,
        public ?PrimaryKeyConstraint $primaryKey,
        public IndexCollection $indexes,
        public ConstraintCollection $constraints,
        public ForeignKeyCollection $foreignKeys,
        public TableOptionSet $options,
        public TableMetadata $metadata,
    ) {}
}
```

---

# 27. Table identity

Distinción:

```text
TableId
≠
TableIdentifier
```

`TableId` representa identidad interna.

`TableIdentifier` representa identidad estructural dentro del database namespace.

---

# 28. Internal IDs

IDs internos son especialmente importantes para:

```text
dependency graphs
schema diff
AST mapping
diagnostics
source maps
cross-object references
```

---

# 29. Internal ID stability

Un ID generado durante introspección no deberá asumirse estable entre:

```text
snapshot t₁
snapshot t₂
```

salvo que exista una política explícita que lo garantice.

---

# 30. Structural identity

Para comparar snapshots, la identidad principal será estructural:

```text
QualifiedObjectName
+
ObjectKind
+
Platform naming semantics
```

no el ID interno temporal.

---

# 31. Column model

```php
final readonly class Column
{
    public function __construct(
        public ColumnId $id,
        public ColumnIdentifier $name,
        public DatabaseType $type,
        public Nullability $nullability,
        public ?DefaultValue $default,
        public ValueGeneration $generation,
        public ColumnOptionSet $options,
        public ColumnMetadata $metadata,
    ) {}
}
```

---

# 32. Column order

El orden de columnas deberá preservarse cuando sea conocido.

```text
columns = ordered collection
```

No:

```text
unordered set
```

---

# 33. Column order semantics

Aunque muchas operaciones SQL no dependan del orden físico, éste puede ser relevante para:

```text
introspection
schema comparison
developer tooling
SELECT *
legacy applications
database dumps
generated SQL
```

---

# 34. ColumnId

Debe distinguirse:

```text
ColumnId
≠
ColumnIdentifier
≠
ResultColumnId
≠
ORMPropertyId
```

---

# 35. DatabaseType

El tipo de una columna deberá representarse mediante:

```text
DatabaseType
```

no mediante SQL textual.

Ejemplo:

```php
new StringType(length: 255);
```

en lugar de:

```php
'VARCHAR(255)'
```

---

# 36. Type layers

Distinción:

```text
Logical Database Type
≠
Physical SQL Type
≠
Driver Result Type
≠
PHP Runtime Type
```

---

# 37. Type metadata

Un tipo puede contener:

```text
length
precision
scale
unsigned semantics
timezone semantics
character semantics
platform extension metadata
```

cuando corresponda.

---

# 38. Nullability

No deberá modelarse únicamente como:

```php
bool $nullable;
```

si necesitamos distinguir metadata desconocida.

Puede utilizarse:

```php
enum Nullability
{
    case NULLABLE;
    case NOT_NULL;
    case UNKNOWN;
}
```

---

# 39. Unknown state

Principio:

```text
UNKNOWN
≠
false
```

Esto será recurrente en todo Schema Model.

---

# 40. Default value

El default deberá representarse estructuralmente.

```text
DefaultValue
├── LiteralDefault
├── NullDefault
├── ExpressionDefault
├── CurrentTimestampDefault
├── SequenceDefault
├── PlatformDefault
└── ExtensionDefault
```

---

# 41. No SQL default strings

Evitar:

```php
$default = 'CURRENT_TIMESTAMP';
```

como representación universal.

---

# 42. Literal default

Ejemplo:

```php
new LiteralDefault(
    new StringValue('active')
);
```

---

# 43. SQL NULL vs absence of default

Distinción esencial:

```text
No Default
≠
DEFAULT NULL
```

---

# 44. DefaultState

Podrá formalizarse:

```text
AbsentDefault
ExplicitNullDefault
ExplicitValueDefault
ExpressionDefault
```

---

# 45. Value generation

Generación de valores deberá modelarse separadamente.

```text
ValueGeneration
├── None
├── Identity
├── Sequence
├── GeneratedExpression
├── PlatformNative
└── Extension
```

---

# 46. Identity ≠ auto increment

La semántica general será:

```text
Generated Identity
```

No:

```text
AUTO_INCREMENT
```

como concepto universal.

---

# 47. Generated columns

Una columna generada deberá conservar:

```text
expression
storage mode
generation semantics
capability requirements
```

---

# 48. Column options

Opciones no fundamentales deberán ir en:

```text
ColumnOptionSet
```

pero mediante objetos tipados.

---

# 49. No arbitrary option arrays

Evitar:

```php
[
    'unsigned' => true,
    'whatever' => 'x',
]
```

como núcleo del modelo.

---

# 50. Primary key model

```php
final readonly class PrimaryKeyConstraint
{
    public function __construct(
        public ConstraintId $id,
        public ?ConstraintIdentifier $name,
        public ColumnReferenceList $columns,
        public ConstraintMetadata $metadata,
    ) {}
}
```

---

# 51. Composite primary keys

El modelo deberá soportar:

```text
PRIMARY KEY (tenant_id, id)
```

sin asumir una sola columna.

---

# 52. Primary key column order

En:

```text
PRIMARY KEY (tenant_id, id)
```

el orden:

```text
tenant_id
→ id
```

debe preservarse.

---

# 53. PrimaryKey ≠ Index

Aunque el servidor pueda crear un índice físico:

```text
PrimaryKeyConstraint
≠
Index
```

---

# 54. Constraint model

Base conceptual:

```php
interface Constraint
{
    public function id(): ConstraintId;

    public function name(): ?ConstraintIdentifier;

    public function kind(): ConstraintKind;
}
```

---

# 55. Constraint kinds

```php
enum ConstraintKind: string
{
    case PRIMARY_KEY = 'primary_key';
    case UNIQUE = 'unique';
    case CHECK = 'check';
    case FOREIGN_KEY = 'foreign_key';
    case EXTENSION = 'extension';
}
```

---

# 56. Unique constraint

```php
final readonly class UniqueConstraint implements Constraint
{
    public function __construct(
        public ConstraintId $id,
        public ?ConstraintIdentifier $name,
        public ColumnReferenceList $columns,
        public ConstraintMetadata $metadata,
    ) {}
}
```

---

# 57. UniqueConstraint ≠ UniqueIndex

Esto es obligatorio:

```text
UniqueConstraint
≠
UniqueIndex
```

aunque una plataforma pueda implementar la constraint mediante un índice.

---

# 58. Check constraint

```php
final readonly class CheckConstraint implements Constraint
{
    public function __construct(
        public ConstraintId $id,
        public ?ConstraintIdentifier $name,
        public SchemaExpression $expression,
        public ConstraintMetadata $metadata,
    ) {}
}
```

---

# 59. Check expression

Ejemplo conceptual:

```text
price >= 0
```

deberá representarse mediante:

```text
SchemaExpression
```

no necesariamente SQL textual.

---

# 60. Foreign key model

```php
final readonly class ForeignKeyConstraint implements Constraint
{
    public function __construct(
        public ConstraintId $id,
        public ?ConstraintIdentifier $name,
        public ColumnReferenceList $localColumns,
        public TableReference $referencedTable,
        public ColumnReferenceList $referencedColumns,
        public ReferentialAction $onUpdate,
        public ReferentialAction $onDelete,
        public ForeignKeyMatchType $match,
        public Deferrability $deferrability,
        public ConstraintMetadata $metadata,
    ) {}
}
```

---

# 61. FK arity

Debe cumplirse:

```text
count(localColumns)
=
count(referencedColumns)
```

---

# 62. Composite FK

Ejemplo:

```text
FOREIGN KEY (tenant_id, user_id)
REFERENCES users (tenant_id, id)
```

debe conservar pares ordenados.

---

# 63. ReferentialAction

```php
enum ReferentialAction
{
    case NO_ACTION;
    case RESTRICT;
    case CASCADE;
    case SET_NULL;
    case SET_DEFAULT;
}
```

---

# 64. NO ACTION ≠ RESTRICT

No deberán colapsarse automáticamente.

Algunas plataformas distinguen sus tiempos de evaluación.

---

# 65. Deferrability

Puede modelarse:

```php
enum Deferrability
{
    case NOT_DEFERRABLE;
    case INITIALLY_IMMEDIATE;
    case INITIALLY_DEFERRED;
    case UNKNOWN;
}
```

---

# 66. Foreign key resolution

Una FK puede estar:

```text
RESOLVED
UNRESOLVED
EXTERNAL
UNKNOWN
```

dependiendo del scope de introspección.

---

# 67. Scoped introspection

Si sólo se introspecciona:

```text
posts
```

y `posts` referencia:

```text
users
```

el modelo no deberá inventar el objeto `users`.

Puede conservar:

```text
ExternalTableReference(users)
```

---

# 68. Reference model

Debe distinguirse:

```text
ResolvedTableReference
ExternalTableReference
UnresolvedTableReference
```

---

# 69. Index model

```php
final readonly class Index
{
    public function __construct(
        public IndexId $id,
        public IndexIdentifier $name,
        public IndexKeyCollection $keys,
        public IndexUniqueness $uniqueness,
        public ?SchemaPredicate $predicate,
        public IndexIncludedColumnCollection $includedColumns,
        public IndexMethod $method,
        public IndexOptionSet $options,
        public IndexMetadata $metadata,
    ) {}
}
```

---

# 70. Index keys

Un índice puede contener:

```text
column key
expression key
```

por tanto:

```text
IndexKey
├── ColumnIndexKey
└── ExpressionIndexKey
```

---

# 71. Index ordering

Cada key podrá conservar:

```text
ASC
DESC
UNSPECIFIED
```

y, si procede:

```text
NULLS FIRST
NULLS LAST
PLATFORM_DEFAULT
```

---

# 72. Partial indexes

Un índice parcial podrá tener:

```text
predicate
```

Ejemplo:

```text
WHERE deleted_at IS NULL
```

---

# 73. Included columns

PostgreSQL y otras plataformas pueden soportar:

```text
INCLUDE (...)
```

Debe representarse como concepto distinto de las keys del índice.

---

# 74. Index method

Ejemplos:

```text
BTREE
HASH
GIN
GIST
BRIN
PLATFORM_DEFAULT
EXTENSION
```

pero no todos serán portables.

---

# 75. Index uniqueness

Puede representarse:

```php
enum IndexUniqueness
{
    case UNIQUE;
    case NON_UNIQUE;
    case UNKNOWN;
}
```

---

# 76. Index visibility

Algunas plataformas poseen metadata como:

```text
visible
invisible
```

Esto deberá ser metadata/opción capability-aware, no propiedad universal obligatoria.

---

# 77. Constraint collection

Una tabla deberá poder indexar constraints por:

```text
ConstraintId
ConstraintIdentifier
ConstraintKind
```

sin perder el orden canónico.

---

# 78. Immutable collections

Las colecciones del Schema Model serán inmutables.

Ejemplo:

```php
final readonly class ColumnCollection implements IteratorAggregate, Countable
{
}
```

---

# 79. No mutable arrays exposed

No deberá permitirse:

```php
$table->columns[] = $column;
```

sobre un `Table` publicado.

---

# 80. Structural replacement

Para modificar un modelo:

```text
Table₀
+
Column
↓
Table₁
```

se produce una nueva instancia.

---

# 81. Persistent data structures

En el futuro podrían utilizarse estructuras persistentes para reducir copias.

Pero la semántica pública seguirá siendo:

```text
immutable
```

---

# 82. Collection indexing

Para eficiencia:

```text
ColumnCollection
├── ordered vector
└── name index
```

puede mantener ambos internamente.

---

# 83. Deterministic iteration

La iteración deberá ser determinista.

---

# 84. Canonical ordering

Cada tipo deberá definir qué orden es:

```text
semantic
```

y cuál puede canonicalizarse.

---

# 85. Column order is semantic metadata

Columnas:

```text
preserve declared/observed order
```

---

# 86. Constraint order

Generalmente puede canonicalizarse cuando no sea semánticamente relevante.

---

# 87. Index order

El orden entre índices independientes puede canonicalizarse.

Pero:

```text
order of keys inside one index
```

debe preservarse.

---

# 88. Foreign key column order

Debe preservarse.

---

# 89. SchemaObjectCollection

El modelo raíz podrá mantener:

```php
final readonly class SchemaObjectCollection
{
    // ordered/canonical objects + indexed lookup
}
```

---

# 90. SchemaObjectRegistry

Puede existir una vista indexada:

```text
SchemaObjectRegistry
```

para resolver:

```text
qualified name → object
object id → object
kind → objects
```

---

# 91. Registry immutability

El registry asociado a un `DatabaseSchema` será inmutable.

---

# 92. Registry ≠ global service registry

No deberá confundirse con:

```text
Service Container
Extension Registry
```

Es simplemente un índice estructural del modelo.

---

# 93. Duplicate object names

Dos objetos incompatibles con la misma identidad estructural deberán provocar:

```text
DuplicateSchemaObjectException
```

---

# 94. Namespace-aware uniqueness

La unicidad deberá considerar:

```text
catalog
schema
object kind
identifier semantics
```

según la plataforma.

---

# 95. Tables

Dos tablas:

```text
public.users
tenant.users
```

son objetos distintos.

---

# 96. Same name different object kinds

Una plataforma podría permitir ciertos nombres coincidentes entre distintos tipos de objeto.

La política de identidad deberá ser platform-aware.

---

# 97. Schema namespace model

```php
final readonly class SchemaNamespace
{
    public function __construct(
        public SchemaNamespaceId $id,
        public SchemaIdentifier $name,
        public SchemaObjectCollection $objects,
        public SchemaNamespaceMetadata $metadata,
    ) {}
}
```

---

# 98. Catalog model

```php
final readonly class Catalog
{
    public function __construct(
        public CatalogId $id,
        public CatalogIdentifier $name,
        public SchemaNamespaceCollection $namespaces,
        public CatalogMetadata $metadata,
    ) {}
}
```

---

# 99. Flattened views

Aunque el modelo pueda ser jerárquico, podrá proporcionar:

```php
$schema->tables();
$schema->sequences();
$schema->views();
```

como vistas derivadas.

---

# 100. Derived view ≠ duplicate storage

No deberá mantenerse:

```text
tables in namespace
+
another independent mutable global table list
```

que pueda desincronizarse.

---

# 101. Sequence model

```php
final readonly class Sequence
{
    public function __construct(
        public SequenceId $id,
        public SequenceIdentifier $name,
        public IntegerValue $start,
        public IntegerValue $increment,
        public ?IntegerValue $minimum,
        public ?IntegerValue $maximum,
        public SequenceCycleMode $cycle,
        public SequenceMetadata $metadata,
    ) {}
}
```

---

# 102. Sequence portability

No todas las plataformas tienen secuencias explícitas.

Por tanto:

```text
Sequence
```

es un SchemaObject capability-aware.

---

# 103. Sequence ≠ identity

Una identity column puede utilizar internamente una sequence en una plataforma.

Pero:

```text
Identity
≠
Sequence
```

---

# 104. View model

La arquitectura deberá dejar espacio para:

```php
final readonly class View implements SchemaObject
{
}
```

aunque la especificación detallada pueda llegar después.

---

# 105. View definition

Una view puede necesitar:

```text
query definition
columns
security options
check options
materialization
dependencies
```

---

# 106. Query dependency problem

Una view puede depender de Query AST.

Debe evitarse un ciclo arquitectónico rígido.

Posibles estrategias:

```text
Shared Expression/Query Contract
```

o:

```text
OpaqueNormalizedViewDefinition
```

dependiendo de la fase futura.

---

# 107. V1 policy for views

Para V1:

```text
View support
=
capability-aware advanced schema object
```

sin obligar al Schema Model base a interpretar toda query de una view.

---

# 108. ExtensionObject

Para características no cubiertas:

```php
interface ExtensionSchemaObject extends SchemaObject
{
    public function extensionId(): SchemaExtensionId;

    public function payload(): ExtensionSchemaPayload;
}
```

---

# 109. Extension payload

No deberá ser simplemente:

```php
mixed
```

Preferible:

```text
typed
versioned
validated
serializable
```

---

# 110. Platform-specific metadata

Un objeto portable puede conservar metadata específica.

Ejemplo:

```text
Table
├── portable structure
└── metadata
    └── MySQL engine = InnoDB
```

---

# 111. Metadata separation

El contenido deberá distinguir:

```text
structural semantics
```

de:

```text
observational/platform metadata
```

---

# 112. Structural metadata

Algunas metadata sí afectan semántica.

Ejemplo:

```text
collation
```

puede cambiar comparación/orden.

Por tanto, no toda metadata puede ignorarse en semantic comparison.

---

# 113. Metadata classification

Se propone:

```php
enum SchemaMetadataImpact
{
    case INFORMATIONAL;
    case STRUCTURAL;
    case SEMANTIC;
    case PHYSICAL;
    case UNKNOWN;
}
```

---

# 114. Physical metadata

Ejemplos:

```text
storage size
page count
internal object ID
statistics
```

no deberán producir migration diff por defecto.

---

# 115. Semantic metadata

Ejemplos potenciales:

```text
collation
generated expression
constraint behavior
```

sí pueden afectar equivalencia.

---

# 116. SchemaObjectMetadata

```php
final readonly class SchemaObjectMetadata
{
    public function __construct(
        public SchemaMetadataEntryCollection $entries,
    ) {}
}
```

---

# 117. Metadata entry

Cada entrada deberá incluir al menos:

```text
key
value
source
certainty
impact
extension/platform ownership
```

---

# 118. Metadata source

```php
enum SchemaMetadataSource
{
    case DECLARED;
    case INTROSPECTED;
    case DRIVER_REPORTED;
    case PLATFORM_DERIVED;
    case INFERRED;
    case EXTENSION;
}
```

---

# 119. Metadata certainty

```php
enum SchemaMetadataCertainty
{
    case EXACT;
    case DECLARED;
    case DRIVER_REPORTED;
    case INFERRED;
    case APPROXIMATE;
    case UNKNOWN;
}
```

---

# 120. Provenance

El modelo deberá poder conservar:

```text
where did this information come from?
```

sin convertir provenance en dependencia del runtime.

---

# 121. Source provenance

Ejemplo:

```text
Introspected from pg_catalog
```

puede conservarse como descriptor estructurado.

No debe retener:

```text
live connection
PDO
request
```

---

# 122. Schema completeness

El modelo puede representar un schema:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 123. Completeness scope

Debe conocerse:

```text
complete relative to what?
```

Ejemplo:

```text
complete database
complete namespace
complete table
selected tables only
```

---

# 124. SchemaCoverage

Se propone:

```php
final readonly class SchemaCoverage
{
    public function __construct(
        public SchemaCoverageKind $kind,
        public SchemaObjectScope $scope,
    ) {}
}
```

---

# 125. Coverage kinds

```php
enum SchemaCoverageKind
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 126. Partial schema semantics

Un `DatabaseSchema` parcial sigue siendo válido.

Pero no puede afirmar:

```text
object does not exist
```

fuera de su scope.

---

# 127. Closed-world vs open-world

Schema completo:

```text
closed-world within scope
```

Schema parcial:

```text
open-world outside scope
```

---

# 128. Important formula

Para un snapshot parcial:

```text
Object ∉ KnownObjects
```

no implica:

```text
Object does not exist
```

---

# 129. Dependency model

Los objetos pueden depender unos de otros.

Ejemplos:

```text
ForeignKey → Table
ForeignKey → Column
Index → Column
GeneratedColumn → Column
View → Table
Sequence → Column
```

---

# 130. SchemaDependency

```php
final readonly class SchemaDependency
{
    public function __construct(
        public SchemaObjectReference $source,
        public SchemaObjectReference $target,
        public SchemaDependencyKind $kind,
    ) {}
}
```

---

# 131. Dependency kinds

```php
enum SchemaDependencyKind
{
    case CONTAINS;
    case REFERENCES;
    case USES_TYPE;
    case USES_SEQUENCE;
    case EXPRESSION_REFERENCE;
    case VIEW_REFERENCE;
    case GENERATION_DEPENDENCY;
    case EXTENSION;
}
```

---

# 132. Dependency graph

```text
Schema Model
    ↓
SchemaDependencyGraph
```

puede derivarse del modelo.

---

# 133. Derived graph

Preferiblemente:

```text
DependencyGraph
=
derived immutable view
```

en lugar de fuente independiente de verdad.

---

# 134. Single source of truth

Principio:

```text
Schema objects
=
source of structural truth
```

Los índices/grafos derivados deben poder reconstruirse.

---

# 135. Object references

Una referencia deberá distinguir:

```text
InternalResolvedReference
ExternalQualifiedReference
UnresolvedReference
```

---

# 136. Internal reference

Cuando ambos objetos están dentro del modelo:

```text
SchemaObjectId
```

puede utilizarse como resolución rápida.

Pero deberá conservarse identidad estructural cuando sea necesario para serialización/diff.

---

# 137. External reference

Ejemplo:

```text
orders.customer_id
→ crm.customers.id
```

cuando `crm.customers` no forma parte del snapshot.

---

# 138. Unresolved reference

Una referencia malformada o no resuelta deberá ser distinguible de una referencia externa intencional.

---

# 139. Referential integrity of model

Para un schema completo, referencias internas esperadas deberán resolverse.

---

# 140. Structural validation

El constructor/factory deberá impedir estados imposibles.

Ejemplos:

```text
duplicate column identity
duplicate object identity
foreign key with mismatched arity
index with zero keys
primary key with zero columns
```

---

# 141. Deep validation

Sin embargo, validaciones costosas podrán pertenecer a:

```text
SchemaValidationSystem
```

y no necesariamente al constructor.

---

# 142. Constructor invariants

Constructores deben garantizar invariantes locales.

---

# 143. Validation system invariants

El validator garantiza:

```text
cross-object
cross-namespace
capability
semantic
platform
```

invariants.

---

# 144. Valid-but-unresolved model

Puede ser útil representar temporalmente:

```text
structurally valid
but externally unresolved
```

durante introspection parcial.

---

# 145. Resolution state

Por ello una referencia puede incluir:

```php
enum SchemaReferenceResolution
{
    case RESOLVED;
    case EXTERNAL;
    case UNRESOLVED;
    case UNKNOWN;
}
```

---

# 146. Model construction pipeline

```text
Raw Definition / Introspection
            ↓
Identifier Normalization
            ↓
Object Construction
            ↓
Local Validation
            ↓
Reference Resolution
            ↓
Dependency Derivation
            ↓
Canonicalization
            ↓
Fingerprinting
            ↓
Immutable DatabaseSchema
```

---

# 147. Definition-origin model

Cuando proviene de Builder:

```text
Builder
↓
Definitions
↓
Schema Model Factory
↓
DatabaseSchema
```

---

# 148. Introspection-origin model

Cuando proviene del servidor:

```text
Native Metadata
↓
Normalization
↓
Schema Metadata
↓
Schema Model Factory
↓
DatabaseSchema
```

---

# 149. Same canonical model

Ambos caminos deben converger:

```text
Desired Definition ─────┐
                        ├→ DatabaseSchema
Introspected Metadata ──┘
```

Esto permite posteriormente:

```text
Diff(Current, Target)
```

---

# 150. Origin metadata

El modelo puede conservar:

```text
DECLARED
INTROSPECTED
GENERATED
MERGED
```

como provenance.

Pero el origen no deberá cambiar la semántica estructural.

---

# 151. SchemaOrigin

```php
enum SchemaOrigin
{
    case DECLARED;
    case INTROSPECTED;
    case GENERATED;
    case MERGED;
}
```

---

# 152. Structural equality

Dos schemas son estructuralmente iguales cuando sus objetos estructurales relevantes coinciden.

```text
StructuralEqual(A, B)
```

---

# 153. Exact equality

Puede exigir:

```text
same structure
same metadata
same ordering
same provenance
same model version
```

según definición.

---

# 154. Semantic equality

Dos representaciones pueden ser distintas estructuralmente pero equivalentes semánticamente para una plataforma.

---

# 155. Example

Una plataforma puede representar:

```text
BOOLEAN
```

físicamente mediante:

```text
TINYINT
```

pero esto no autoriza al Schema Model a considerarlos universalmente equivalentes.

---

# 156. Platform equivalence

Debe existir:

```text
PlatformSchemaComparator
```

para equivalencias dependientes de plataforma.

---

# 157. Comparison modes

```php
enum SchemaComparisonMode
{
    case EXACT;
    case STRUCTURAL;
    case SEMANTIC;
    case PLATFORM_NORMALIZED;
}
```

---

# 158. Equality ≠ identity

Siempre:

```text
Equal(A, B)
```

no implica:

```text
Identity(A) = Identity(B)
```

---

# 159. Canonicalization

El modelo deberá tener una forma canónica para:

```text
comparison
fingerprinting
cache
serialization
testing
diff
```

---

# 160. Canonicalization rules

Podrán incluir:

```text
normalize identifiers
normalize unordered option sets
canonical constraint ordering
canonical index ordering
normalize metadata keys
normalize type parameters
```

sin destruir semántica.

---

# 161. Canonicalization is not mutation

```text
Canonicalize(S)
→ S'
```

No:

```text
mutate S
```

---

# 162. Idempotence

Debe cumplirse:

```text
Canonicalize(
    Canonicalize(S)
)
=
Canonicalize(S)
```

---

# 163. Semantic preservation

También:

```text
Semantics(Canonicalize(S))
=
Semantics(S)
```

---

# 164. Fingerprinting

Después de canonicalizar:

```text
CanonicalSchema
↓
SchemaFingerprint
```

---

# 165. Fingerprint requirements

Debe ser:

```text
deterministic
versioned
content-derived
collision-resistant
platform-context-aware when necessary
```

---

# 166. Fingerprint version

```text
SchemaFingerprintV1
```

deberá poder coexistir en el futuro con:

```text
SchemaFingerprintV2
```

---

# 167. Fingerprint input

Conceptualmente:

```text
Fingerprint(
    CanonicalStructure,
    RelevantSemanticMetadata,
    SchemaModelVersion,
    CanonicalizationVersion
)
```

---

# 168. Fingerprint exclusions

No deberá incluir por defecto:

```text
connection handle
request ID
capture timestamp
tenant runtime object
PDO object
memory address
random internal IDs
```

---

# 169. Internal IDs and fingerprint

IDs efímeros no deberán alterar el fingerprint estructural.

---

# 170. Metadata fingerprint policy

Sólo metadata clasificada como relevante deberá participar según el modo.

Ejemplo:

```text
INFORMATIONAL
```

puede excluirse.

```text
SEMANTIC
```

normalmente deberá incluirse.

---

# 171. Multiple fingerprints

Podría ser útil mantener:

```text
StructuralFingerprint
SemanticFingerprint
ExactFingerprint
```

en lugar de un único hash ambiguo.

---

# 172. SchemaFingerprintSet

```php
final readonly class SchemaFingerprintSet
{
    public function __construct(
        public StructuralSchemaFingerprint $structural,
        public SemanticSchemaFingerprint $semantic,
        public ExactSchemaFingerprint $exact,
    ) {}
}
```

---

# 173. Diff fingerprint

El Diff System podrá utilizar:

```text
StructuralFingerprint
```

para evitar comparaciones profundas cuando sea seguro.

---

# 174. Fingerprint collision

Un fingerprint:

```text
match
```

podrá usarse como fast-path según política.

Para operaciones críticas, podrá confirmarse mediante comparación estructural.

---

# 175. Serialization

El Schema Model deberá ser serializable.

Casos:

```text
schema snapshots
migration planning
CI artifacts
cache
developer tooling
diagnostics
```

---

# 176. Serialization format

Debe ser:

```text
versioned
deterministic
explicitly typed
forward-aware
```

---

# 177. Serialization ≠ PHP serialize()

La arquitectura no deberá depender de:

```php
serialize($schema);
```

como formato persistente oficial.

---

# 178. Canonical representation

Podrá definirse posteriormente un formato como:

```text
Schema IR
```

para serialización estable.

---

# 179. Deserialization validation

Todo artefacto deserializado deberá:

```text
validate version
validate structure
validate extension compatibility
validate fingerprints where applicable
```

---

# 180. Untrusted serialized schemas

Un Schema Model cargado desde archivo externo deberá tratarse como:

```text
untrusted structured input
```

hasta ser validado.

---

# 181. Comments

Los comentarios de tabla/columna podrán formar parte del modelo.

Pero deberán clasificarse adecuadamente:

```text
informational metadata
```

salvo que una extensión les otorgue otra semántica.

---

# 182. Collation

Collation puede afectar:

```text
comparison
ordering
uniqueness
index behavior
```

por lo que debe tratarse como metadata semántica/estructural.

---

# 183. Charset

Charset puede afectar representación y compatibilidad.

Debe ser modelado explícitamente cuando corresponda.

---

# 184. Table options

Ejemplo conceptual:

```text
TableOptionSet
├── PortableTableOptions
└── PlatformTableOptions
```

---

# 185. MySQL metadata

Podrá conservar:

```text
engine
charset
collation
row format
```

cuando proceda.

---

# 186. PostgreSQL metadata

Podrá conservar:

```text
tablespace
persistence
access method
storage parameters
```

cuando proceda.

---

# 187. SQLite metadata

Podrá conservar:

```text
STRICT
WITHOUT ROWID
ROWID semantics
```

cuando proceda.

---

# 188. MariaDB metadata

Podrá conservar características propias de MariaDB independientemente de MySQL.

---

# 189. No platform contamination

El `Table` base no deberá terminar con:

```php
$table->mysqlEngine;
$table->postgresTablespace;
$table->sqliteWithoutRowId;
```

como campos obligatorios universales.

---

# 190. Typed platform metadata

Preferible:

```text
TableMetadata
└── PlatformMetadataSet
    ├── MySqlTableMetadata
    ├── MariaDbTableMetadata
    ├── PostgreSqlTableMetadata
    └── SqliteTableMetadata
```

---

# 191. Extension ownership

Toda metadata específica deberá declarar:

```text
owner
version
schema
```

para evitar colisiones.

---

# 192. No arbitrary metadata collision

No:

```text
"engine" => ...
```

globalmente.

Preferible identidad tipada:

```text
mysql.table.engine
```

conceptualmente.

---

# 193. Model factory

La construcción compleja deberá centralizarse en:

```php
interface DatabaseSchemaFactory
{
    public function create(
        SchemaModelInput $input,
        SchemaModelContext $context,
    ): DatabaseSchema;
}
```

---

# 194. SchemaModelContext

Puede contener:

```text
identifier policy
type registry snapshot
extension registry snapshot
canonicalization policy
validation profile
```

pero no:

```text
live connection
HTTP request
ORM EntityManager
service container
```

---

# 195. Model context immutability

El contexto deberá ser:

```text
immutable
operation-scoped or reusable if stateless
```

---

# 196. Schema model registry

Los tipos estructurales extensibles podrán resolverse mediante:

```text
SchemaModelRegistry
```

frozen después de bootstrap.

---

# 197. Registry content

Podrá registrar:

```text
SchemaObjectKind
SchemaTypeId
SchemaMetadataTypeId
SchemaExpressionKind
ExtensionSchemaObjectKind
```

---

# 198. No dynamic runtime mutation

No:

```php
$registry->register(...)
```

durante una request después del bootstrap.

---

# 199. Persistent runtime safety

Bajo FrankenPHP:

```text
Worker
├── Request A
└── Request B
```

los modelos publicados pueden compartirse sólo si son realmente:

```text
immutable
tenant-neutral where required
context-independent
```

---

# 200. Tenant-specific models

Un Schema Model tenant-specific puede existir.

Pero:

```text
tenant identity
```

no deberá quedar en un singleton global.

---

# 201. Tenant metadata

Cuando Multitenancy esté instalado podrá adjuntar metadata externa.

Database core no dependerá de:

```text
VoltStack/Quantum/Multitenancy
```

---

# 202. Schema model and tenant isolation

Debe evitarse:

```text
Tenant A Schema
→ global cache
→ Tenant B receives same tenant-specific model
```

sin una clave de aislamiento válida.

---

# 203. Cache identity

Si el modelo se cachea, la clave deberá considerar:

```text
structural fingerprint
platform context
scope
extension set
tenant isolation when semantically relevant
```

---

# 204. Schema model does not cache itself

El modelo no deberá conocer:

```text
Redis
filesystem
cache repository
```

---

# 205. Query integration

El Query Semantic Engine podrá consultar:

```text
Table
Column
Type
Nullability
Constraints
```

para:

```text
symbol resolution
type inference
join validation
constraint reasoning
```

---

# 206. Query cannot mutate schema

La integración será:

```text
Query Semantic Engine
→ read Schema Model
```

Nunca:

```text
Query Semantic Engine
→ mutate Schema Model
```

---

# 207. Optimizer integration

El optimizer podrá consumir hechos estructurales como:

```text
unique keys
nullability
foreign keys
```

si se presentan como hechos confiables.

---

# 208. Schema fact certainty

Los hechos derivados de metadata parcial deberán conservar su certeza.

---

# 209. Exact facts vs estimates

Schema Model contiene principalmente:

```text
structural facts
```

No estadísticas de cardinalidad estimadas.

Éstas pertenecen a otros subsistemas.

---

# 210. ORM integration

Posteriormente ORM podrá usar:

```text
Schema Model
```

para:

```text
mapping validation
schema generation
development diagnostics
```

---

# 211. ORM mapping ≠ Schema Model

Nunca:

```text
EntityProperty
=
Column
```

Son conceptos distintos relacionados mediante mapping.

---

# 212. Migration integration

Migration System podrá tener:

```text
CurrentSchema
TargetSchema
SchemaDiff
SchemaChangePlan
```

pero el Schema Model no conocerá:

```text
migration batch
migration history
rollback
migration class
```

---

# 213. Execution integration

Schema Model nunca llegará directamente al driver.

Flujo:

```text
Schema Model
↓
Schema Operation / Diff
↓
Schema Plan
↓
Schema Compiler
↓
Compiled Command
↓
Execution
```

---

# 214. Model transformations

Operaciones puras podrán existir:

```php
$schema2 = $schema1->withTable($table);
```

pero deberán devolver:

```text
new immutable model
```

---

# 215. with* API

Si se proporciona:

```php
withTable()
withoutTable()
replaceTable()
```

debe entenderse como:

```text
model transformation
```

no DDL execution.

---

# 216. Naming warning

Evitar APIs ambiguas:

```php
$schema->dropTable(...)
```

sobre el modelo.

Preferible:

```php
$schema->withoutTable(...)
```

para transformación in-memory.

`dropTable()` deberá reservarse para operación estructural.

---

# 217. Model editor

Puede existir un:

```text
SchemaModelEditor
```

mutable y operation-scoped para transformaciones masivas.

---

# 218. Freeze

```text
Immutable Schema
↓
ModelEditor
↓
mutations in local workspace
↓
freeze()
↓
New Immutable Schema
```

---

# 219. ModelEditor ≠ SchemaBuilder

Diferencia:

```text
SchemaBuilder
=
developer-facing definition DSL
```

```text
SchemaModelEditor
=
internal transformation utility
```

---

# 220. ModelEditor ≠ Migration

Modificar un modelo en memoria no cambia la base de datos.

---

# 221. Memory model

Para schemas grandes, copiar todo el grafo en cada transformación sería costoso.

La implementación podrá utilizar:

```text
structural sharing
copy-on-write
persistent maps
interned immutable values
```

sin cambiar la API.

---

# 222. Complexity target

Lookup de tabla:

```text
O(1)
```

promedio mediante índices hash adecuados.

Lookup de columna:

```text
O(1)
```

promedio.

Traversal:

```text
O(n)
```

---

# 223. Dependency derivation

Construcción del grafo:

```text
O(V + E)
```

aproximadamente.

---

# 224. Fingerprinting

Debe evitar recomputar recursivamente todo el schema innecesariamente.

Podrá utilizarse:

```text
hierarchical fingerprints
```

---

# 225. Hierarchical fingerprinting

Ejemplo:

```text
ColumnFingerprint
      ↓
TableFingerprint
      ↓
NamespaceFingerprint
      ↓
DatabaseSchemaFingerprint
```

---

# 226. Merkle-like structure

Conceptualmente:

```text
SchemaFingerprint
=
H(
    NamespaceFingerprint₁,
    NamespaceFingerprint₂,
    ...
)
```

Esto puede acelerar diff.

---

# 227. Semantic ordering caution

El hashing jerárquico deberá respetar qué colecciones son:

```text
ordered
```

y cuáles:

```text
canonical sets
```

---

# 228. Column hash

Debe incluir posición si el comparison mode considera el orden relevante.

---

# 229. Metadata hash

Debe seguir:

```text
SchemaMetadataImpact
```

y comparison mode.

---

# 230. Source location

Definitions creadas desde código pueden conservar:

```text
SourceLocation
```

para diagnósticos.

Ejemplo:

```text
database/migrations/...
line 42
```

---

# 231. SourceLocation ≠ structural semantics

No deberá cambiar:

```text
StructuralFingerprint
```

---

# 232. Introspection source

Metadata introspectada puede conservar:

```text
catalog source
```

sin cambiar structural equality.

---

# 233. Diagnostics

El modelo deberá permitir errores como:

```text
Foreign key "posts_user_id_fk"
references column "users.id",
but "users.id" does not exist in the current complete schema.
```

---

# 234. Diagnostic paths

Cada objeto podrá producir un path:

```text
catalog.app.public.posts.foreign_keys.posts_user_id_fk
```

para diagnóstico.

---

# 235. SchemaObjectPath

```php
final readonly class SchemaObjectPath
{
}
```

deberá ser estructurado, no un string arbitrario internamente.

---

# 236. Security

El Schema Model puede contener:

```text
table names
column names
comments
default expressions
platform metadata
```

que podrían ser sensibles.

---

# 237. Safe debug representation

`__toString()` no deberá imprimir automáticamente todo el schema.

---

# 238. Redaction

Debe existir una política:

```text
SchemaDiagnosticRedactionPolicy
```

para herramientas de debugging/telemetry.

---

# 239. Secrets

Credenciales de conexión:

```text
MUST NOT
```

formar parte del Schema Model.

---

# 240. Data values

Datos de filas:

```text
MUST NOT
```

formar parte del Schema Model.

---

# 241. Default values

Defaults estructurales sí pueden formar parte del modelo.

Pero defaults potencialmente sensibles deberán respetar sensitivity metadata.

---

# 242. Security classification

Opcionalmente:

```php
enum SchemaInformationSensitivity
{
    case PUBLIC;
    case INTERNAL;
    case SENSITIVE;
    case SECRET;
}
```

---

# 243. Telemetry

No deberá emitirse:

```text
full schema model
```

como atributo normal de tracing.

---

# 244. Metrics

Metrics deberán limitarse a:

```text
number of tables
number of columns
number of indexes
model construction duration
fingerprint duration
validation failures
```

sin identifiers de alta cardinalidad por defecto.

---

# 245. Extension model

Las extensiones deberán preservar:

```text
immutability
determinism
serialization
fingerprinting
validation
ownership
versioning
```

---

# 246. Extension equality

Una extensión deberá declarar cómo participa en:

```text
exact equality
structural equality
semantic equality
fingerprinting
serialization
```

---

# 247. Unknown extension

Si se carga un Schema Model que contiene una extensión no disponible:

```text
UnknownExtensionSchemaObject
```

deberá manejarse según política.

---

# 248. No silent deletion

Nunca:

```text
unknown extension
→ discard object
```

silenciosamente.

---

# 249. Unknown extension policy

```php
enum UnknownSchemaExtensionPolicy
{
    case FAIL;
    case PRESERVE_OPAQUE;
    case IGNORE_WITH_DIAGNOSTIC;
}
```

---

# 250. Error hierarchy

```text
DatabaseSchemaModelException
├── InvalidSchemaModelException
├── InvalidSchemaObjectException
├── DuplicateSchemaObjectException
├── DuplicateTableException
├── DuplicateColumnException
├── DuplicateConstraintException
├── DuplicateIndexException
├── InvalidSchemaReferenceException
├── UnresolvedSchemaReferenceException
├── SchemaIdentityException
├── SchemaCanonicalizationException
├── SchemaFingerprintException
├── SchemaSerializationException
├── SchemaExtensionModelException
├── SchemaCoverageException
└── SchemaModelInvariantException
```

---

# 251. Invalid model

Ejemplos:

```text
table without identifier
index without keys
foreign key arity mismatch
duplicate column identity
```

---

# 252. Unresolved reference

En schema completo:

```text
FK → missing table
```

puede ser error.

En schema parcial:

```text
FK → external table
```

puede ser válido.

El contexto importa.

---

# 253. Model validation profile

Puede existir:

```php
enum SchemaModelValidationProfile
{
    case STRICT;
    case INTROSPECTION;
    case PARTIAL;
    case EXTENSION_AWARE;
}
```

---

# 254. STRICT

Requiere máxima consistencia interna.

---

# 255. INTROSPECTION

Tolera metadata:

```text
UNKNOWN
```

cuando el servidor no la proporciona.

---

# 256. PARTIAL

Permite referencias externas fuera del scope conocido.

---

# 257. EXTENSION_AWARE

Permite objetos extension-owned registrados.

---

# 258. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Model
```

---

# 259. Proposed directory structure

```text
Schema/
└── Model/
    ├── DatabaseSchema.php
    ├── DatabaseSchemaId.php
    ├── SchemaModelVersion.php
    ├── SchemaOrigin.php
    │
    ├── Object/
    │   ├── SchemaObject.php
    │   ├── SchemaObjectId.php
    │   ├── SchemaObjectKind.php
    │   ├── SchemaObjectCollection.php
    │   ├── SchemaObjectRegistry.php
    │   ├── SchemaObjectPath.php
    │   └── SchemaObjectReference.php
    │
    ├── Identifier/
    │   ├── Identifier.php
    │   ├── NormalizedIdentifier.php
    │   ├── CatalogIdentifier.php
    │   ├── SchemaIdentifier.php
    │   ├── TableIdentifier.php
    │   ├── ColumnIdentifier.php
    │   ├── IndexIdentifier.php
    │   ├── ConstraintIdentifier.php
    │   ├── SequenceIdentifier.php
    │   └── QualifiedSchemaObjectName.php
    │
    ├── Catalog/
    │   ├── Catalog.php
    │   ├── CatalogId.php
    │   └── CatalogCollection.php
    │
    ├── Namespace/
    │   ├── SchemaNamespace.php
    │   ├── SchemaNamespaceId.php
    │   └── SchemaNamespaceCollection.php
    │
    ├── Table/
    │   ├── Table.php
    │   ├── TableId.php
    │   ├── TableCollection.php
    │   ├── TableOptionSet.php
    │   └── TableMetadata.php
    │
    ├── Column/
    │   ├── Column.php
    │   ├── ColumnId.php
    │   ├── ColumnCollection.php
    │   ├── Nullability.php
    │   ├── ColumnOptionSet.php
    │   ├── DefaultValue.php
    │   └── ValueGeneration.php
    │
    ├── Index/
    │   ├── Index.php
    │   ├── IndexId.php
    │   ├── IndexCollection.php
    │   ├── IndexKey.php
    │   ├── ColumnIndexKey.php
    │   ├── ExpressionIndexKey.php
    │   ├── IndexMethod.php
    │   └── IndexOptionSet.php
    │
    ├── Constraint/
    │   ├── Constraint.php
    │   ├── ConstraintId.php
    │   ├── ConstraintKind.php
    │   ├── PrimaryKeyConstraint.php
    │   ├── UniqueConstraint.php
    │   ├── CheckConstraint.php
    │   └── ConstraintCollection.php
    │
    ├── ForeignKey/
    │   ├── ForeignKeyConstraint.php
    │   ├── ReferentialAction.php
    │   ├── Deferrability.php
    │   ├── ForeignKeyMatchType.php
    │   └── SchemaReferenceResolution.php
    │
    ├── Sequence/
    │   ├── Sequence.php
    │   ├── SequenceId.php
    │   └── SequenceCollection.php
    │
    ├── View/
    │   ├── View.php
    │   └── MaterializedView.php
    │
    ├── Expression/
    │   ├── SchemaExpression.php
    │   └── SchemaPredicate.php
    │
    ├── Metadata/
    │   ├── SchemaObjectMetadata.php
    │   ├── SchemaMetadataEntry.php
    │   ├── SchemaMetadataSource.php
    │   ├── SchemaMetadataCertainty.php
    │   ├── SchemaMetadataImpact.php
    │   └── PlatformMetadataSet.php
    │
    ├── Coverage/
    │   ├── SchemaCoverage.php
    │   ├── SchemaCoverageKind.php
    │   └── SchemaObjectScope.php
    │
    ├── Dependency/
    │   ├── SchemaDependency.php
    │   ├── SchemaDependencyKind.php
    │   └── SchemaDependencyGraph.php
    │
    ├── Comparison/
    │   ├── SchemaComparisonMode.php
    │   ├── SchemaComparator.php
    │   └── PlatformSchemaComparator.php
    │
    ├── Canonicalization/
    │   ├── SchemaCanonicalizer.php
    │   └── SchemaCanonicalizationVersion.php
    │
    ├── Fingerprint/
    │   ├── SchemaFingerprintSet.php
    │   ├── StructuralSchemaFingerprint.php
    │   ├── SemanticSchemaFingerprint.php
    │   └── ExactSchemaFingerprint.php
    │
    ├── Factory/
    │   ├── DatabaseSchemaFactory.php
    │   └── SchemaModelContext.php
    │
    ├── Extension/
    │   ├── ExtensionSchemaObject.php
    │   ├── ExtensionSchemaPayload.php
    │   └── UnknownSchemaExtensionPolicy.php
    │
    └── Exception/
        ├── DatabaseSchemaModelException.php
        ├── InvalidSchemaModelException.php
        ├── InvalidSchemaObjectException.php
        ├── DuplicateSchemaObjectException.php
        ├── InvalidSchemaReferenceException.php
        ├── UnresolvedSchemaReferenceException.php
        ├── SchemaIdentityException.php
        ├── SchemaCanonicalizationException.php
        ├── SchemaFingerprintException.php
        ├── SchemaSerializationException.php
        ├── SchemaExtensionModelException.php
        └── SchemaModelInvariantException.php
```

---

# 260. Dependency rules

Permitido:

```text
Schema Model
→ shared immutable database types

Schema Model
→ shared identifier contracts

Schema Model
→ immutable extension contracts
```

No permitido:

```text
Schema Model
→ Connection
Schema Model
→ Driver
Schema Model
→ PDO
Schema Model
→ Execution Engine
Schema Model
→ Migration Runtime
Schema Model
→ ORM
Schema Model
→ HTTP
```

---

# 261. Architectural invariants

## DB-SCHEMA-MODEL-001

`DatabaseSchema` será un artefacto estructural inmutable.

## DB-SCHEMA-MODEL-002

Schema Model no ejecutará operaciones de base de datos.

## DB-SCHEMA-MODEL-003

Schema Model no generará SQL.

## DB-SCHEMA-MODEL-004

Schema Model no abrirá conexiones.

## DB-SCHEMA-MODEL-005

Schema Model será distinto de Schema AST.

## DB-SCHEMA-MODEL-006

Schema Model será distinto de Migration.

## DB-SCHEMA-MODEL-007

Schema Model será distinto de SchemaDiff.

## DB-SCHEMA-MODEL-008

Schema Model será distinto de SchemaChangePlan.

## DB-SCHEMA-MODEL-009

Schema Model será distinto de ORM Metadata.

## DB-SCHEMA-MODEL-010

Schema Model será distinto de Query AST.

## DB-SCHEMA-MODEL-011

`SchemaObjectId` será distinto de `SchemaObjectIdentifier`.

## DB-SCHEMA-MODEL-012

Internal object IDs no definirán structural equality.

## DB-SCHEMA-MODEL-013

IDs efímeros no participarán en structural fingerprint.

## DB-SCHEMA-MODEL-014

Qualified names serán estructurados.

## DB-SCHEMA-MODEL-015

Qualified names no se almacenarán principalmente como SQL strings.

## DB-SCHEMA-MODEL-016

Identifier quoting no pertenecerá al modelo.

## DB-SCHEMA-MODEL-017

Identifier normalization será platform-aware.

## DB-SCHEMA-MODEL-018

No se aplicará `strtolower()` como regla universal de identidad.

## DB-SCHEMA-MODEL-019

Catalog será distinto de Schema Namespace.

## DB-SCHEMA-MODEL-020

Schema Namespace será distinto de Connection Database Target.

## DB-SCHEMA-MODEL-021

Table será un objeto estructural inmutable.

## DB-SCHEMA-MODEL-022

Column será un objeto estructural inmutable.

## DB-SCHEMA-MODEL-023

Index será un objeto estructural inmutable.

## DB-SCHEMA-MODEL-024

Constraint será un objeto estructural inmutable.

## DB-SCHEMA-MODEL-025

ForeignKey será un objeto estructural inmutable.

## DB-SCHEMA-MODEL-026

Column order será preservable.

## DB-SCHEMA-MODEL-027

Composite key order será preservado.

## DB-SCHEMA-MODEL-028

Composite FK mapping order será preservado.

## DB-SCHEMA-MODEL-029

Index key order será preservado.

## DB-SCHEMA-MODEL-030

PrimaryKey será distinto de Index.

## DB-SCHEMA-MODEL-031

UniqueConstraint será distinta de UniqueIndex.

## DB-SCHEMA-MODEL-032

Logical Database Type será distinto de SQL physical type.

## DB-SCHEMA-MODEL-033

Logical Database Type será distinto de PHP runtime type.

## DB-SCHEMA-MODEL-034

No Default será distinto de DEFAULT NULL.

## DB-SCHEMA-MODEL-035

Identity será distinta de AUTO_INCREMENT.

## DB-SCHEMA-MODEL-036

Identity será distinta de Sequence.

## DB-SCHEMA-MODEL-037

Generated column conservará su expresión estructural.

## DB-SCHEMA-MODEL-038

Schema expressions no serán raw SQL por defecto.

## DB-SCHEMA-MODEL-039

NO ACTION será distinto de RESTRICT.

## DB-SCHEMA-MODEL-040

Foreign-key arity deberá ser válida.

## DB-SCHEMA-MODEL-041

External references serán representables.

## DB-SCHEMA-MODEL-042

Unresolved references serán distintas de external references.

## DB-SCHEMA-MODEL-043

Schema parcial podrá contener referencias externas válidas.

## DB-SCHEMA-MODEL-044

Absence from partial schema no implicará inexistencia.

## DB-SCHEMA-MODEL-045

Schema coverage será explícita.

## DB-SCHEMA-MODEL-046

UNKNOWN no se convertirá implícitamente en false.

## DB-SCHEMA-MODEL-047

Metadata tendrá provenance.

## DB-SCHEMA-MODEL-048

Metadata tendrá certainty cuando sea necesario.

## DB-SCHEMA-MODEL-049

Metadata podrá declarar semantic impact.

## DB-SCHEMA-MODEL-050

Physical metadata no generará structural diff por defecto.

## DB-SCHEMA-MODEL-051

Semantic metadata podrá participar en semantic comparison.

## DB-SCHEMA-MODEL-052

Platform metadata estará tipada.

## DB-SCHEMA-MODEL-053

MySQL metadata no contaminará Table base.

## DB-SCHEMA-MODEL-054

MariaDB metadata no será tratada automáticamente como MySQL metadata.

## DB-SCHEMA-MODEL-055

PostgreSQL metadata estará aislada.

## DB-SCHEMA-MODEL-056

SQLite metadata estará aislada.

## DB-SCHEMA-MODEL-057

Schema collections serán inmutables.

## DB-SCHEMA-MODEL-058

Schema collections tendrán deterministic iteration.

## DB-SCHEMA-MODEL-059

No se expondrán arrays mutables internos.

## DB-SCHEMA-MODEL-060

Model transformations producirán nuevos artefactos.

## DB-SCHEMA-MODEL-061

Derived indexes no serán una segunda mutable source of truth.

## DB-SCHEMA-MODEL-062

Dependency graph será derivable del modelo.

## DB-SCHEMA-MODEL-063

Dependencies tendrán tipos explícitos.

## DB-SCHEMA-MODEL-064

Schema references tendrán resolution state.

## DB-SCHEMA-MODEL-065

Complete schemas deberán resolver referencias internas requeridas.

## DB-SCHEMA-MODEL-066

Partial schemas podrán conservar external references.

## DB-SCHEMA-MODEL-067

Canonicalization no mutará el modelo.

## DB-SCHEMA-MODEL-068

Canonicalization será idempotente.

## DB-SCHEMA-MODEL-069

Canonicalization preservará semántica.

## DB-SCHEMA-MODEL-070

Structural equality será distinta de exact equality.

## DB-SCHEMA-MODEL-071

Semantic equality será distinta de structural equality.

## DB-SCHEMA-MODEL-072

Platform equality será platform-aware.

## DB-SCHEMA-MODEL-073

Equality será distinta de identity.

## DB-SCHEMA-MODEL-074

Schema fingerprint será determinista.

## DB-SCHEMA-MODEL-075

Schema fingerprint será versionado.

## DB-SCHEMA-MODEL-076

Schema fingerprint no será sólo hash de SQL.

## DB-SCHEMA-MODEL-077

Runtime IDs no participarán en structural fingerprint.

## DB-SCHEMA-MODEL-078

Connection handles no participarán en fingerprint.

## DB-SCHEMA-MODEL-079

Request IDs no participarán en fingerprint.

## DB-SCHEMA-MODEL-080

Tenant runtime objects no participarán en tenant-neutral fingerprints.

## DB-SCHEMA-MODEL-081

Multiple fingerprint semantics serán explícitas.

## DB-SCHEMA-MODEL-082

Serialization será versionada.

## DB-SCHEMA-MODEL-083

Serialization oficial no dependerá de PHP `serialize()`.

## DB-SCHEMA-MODEL-084

Deserialization deberá validar estructura.

## DB-SCHEMA-MODEL-085

Unknown extensions no serán descartadas silenciosamente.

## DB-SCHEMA-MODEL-086

Extensions tendrán typed ownership.

## DB-SCHEMA-MODEL-087

Extensions declararán serialization semantics.

## DB-SCHEMA-MODEL-088

Extensions declararán equality semantics.

## DB-SCHEMA-MODEL-089

Extensions declararán fingerprint semantics.

## DB-SCHEMA-MODEL-090

Schema Model no contendrá credenciales.

## DB-SCHEMA-MODEL-091

Schema Model no contendrá row data.

## DB-SCHEMA-MODEL-092

Schema Model no contendrá live driver handles.

## DB-SCHEMA-MODEL-093

Schema Model no contendrá HTTP request state.

## DB-SCHEMA-MODEL-094

Schema Model no contendrá EntityManager.

## DB-SCHEMA-MODEL-095

Schema Model no será service locator.

## DB-SCHEMA-MODEL-096

Schema Model será persistent-runtime safe.

## DB-SCHEMA-MODEL-097

Mutable model construction state será operation-scoped.

## DB-SCHEMA-MODEL-098

Concurrent model construction estará aislada.

## DB-SCHEMA-MODEL-099

FrankenPHP worker reuse no filtrará model state.

## DB-SCHEMA-MODEL-100

RoadRunner worker reuse futuro conservará aislamiento.

## DB-SCHEMA-MODEL-101

OpenSwoole coroutine reuse futuro conservará aislamiento.

## DB-SCHEMA-MODEL-102

Database core no dependerá de Multitenancy.

## DB-SCHEMA-MODEL-103

Tenant-specific schemas deberán aislarse explícitamente.

## DB-SCHEMA-MODEL-104

Global caches no mezclarán tenant-specific models.

## DB-SCHEMA-MODEL-105

Query Engine podrá leer Schema Model.

## DB-SCHEMA-MODEL-106

Query Engine no podrá mutar Schema Model.

## DB-SCHEMA-MODEL-107

Optimizer podrá consumir sólo hechos estructurales confiables.

## DB-SCHEMA-MODEL-108

ORM podrá consumir Schema Model sin invertir dependencia.

## DB-SCHEMA-MODEL-109

ORM Property será distinta de Column.

## DB-SCHEMA-MODEL-110

Migration podrá consumir Schema Model.

## DB-SCHEMA-MODEL-111

Schema Model no conocerá migration batches.

## DB-SCHEMA-MODEL-112

Schema Model no conocerá rollback execution.

## DB-SCHEMA-MODEL-113

Schema Model no llegará directamente al Driver.

## DB-SCHEMA-MODEL-114

Schema Model no llegará directamente al Execution Engine.

## DB-SCHEMA-MODEL-115

ModelEditor será distinto de SchemaBuilder.

## DB-SCHEMA-MODEL-116

ModelEditor será distinto de Migration.

## DB-SCHEMA-MODEL-117

ModelEditor será operation-scoped.

## DB-SCHEMA-MODEL-118

Structural sharing podrá utilizarse sin romper immutability.

## DB-SCHEMA-MODEL-119

Lookup indexes serán derivados y consistentes.

## DB-SCHEMA-MODEL-120

Hierarchical fingerprints preservarán ordering semantics.

## DB-SCHEMA-MODEL-121

SourceLocation no alterará structural semantics.

## DB-SCHEMA-MODEL-122

SourceLocation no alterará structural fingerprint.

## DB-SCHEMA-MODEL-123

Diagnostic paths serán estructurados.

## DB-SCHEMA-MODEL-124

Full schema dumps no se emitirán a telemetry por defecto.

## DB-SCHEMA-MODEL-125

Sensitive metadata será redactable.

## DB-SCHEMA-MODEL-126

Model factory no realizará hidden DB I/O.

## DB-SCHEMA-MODEL-127

Model canonicalization no realizará DB I/O.

## DB-SCHEMA-MODEL-128

Model comparison no realizará DB I/O.

## DB-SCHEMA-MODEL-129

Model fingerprinting no realizará DB I/O.

## DB-SCHEMA-MODEL-130

Schema Model será independiente del runtime concreto.

## DB-SCHEMA-MODEL-131

Schema Model será portable-by-default.

## DB-SCHEMA-MODEL-132

Platform-specific features serán preservables.

## DB-SCHEMA-MODEL-133

Portability no implicará pérdida silenciosa de metadata nativa.

## DB-SCHEMA-MODEL-134

Unknown native features deberán conservarse o fallar según política explícita.

## DB-SCHEMA-MODEL-135

Schema Model tendrá una única fuente estructural de verdad.

## DB-SCHEMA-MODEL-136

Derived registries podrán reconstruirse desde los objetos.

## DB-SCHEMA-MODEL-137

Structural model construction deberá ser determinista.

## DB-SCHEMA-MODEL-138

Same canonical structure deberá producir same structural fingerprint.

## DB-SCHEMA-MODEL-139

Schema Model deberá soportar schemas grandes sin búsquedas lineales repetidas.

## DB-SCHEMA-MODEL-140

Schema Model será la representación estructural canónica compartida del subsistema Database.

---

# 262. Anti-patterns

## 262.1 SQL como modelo

Incorrecto:

```php
$table->type = 'VARCHAR(255)';
```

Correcto:

```php
$table->type = new StringType(length: 255);
```

---

## 262.2 Mutable table

Incorrecto:

```php
$schema->tables['users']->columns[] = $column;
```

Correcto:

```text
Schema₀
↓
transformation
↓
Schema₁
```

---

## 262.3 Platform fields everywhere

Incorrecto:

```php
class Table
{
    public $mysqlEngine;
    public $postgresTablespace;
    public $sqliteWithoutRowId;
}
```

Correcto:

```text
Table
└── PlatformMetadataSet
```

---

## 262.4 Conflating identity and name

Incorrecto:

```text
TableId = "users"
```

Correcto:

```text
TableId
≠
TableIdentifier
```

---

## 262.5 Associative-array schema

Incorrecto:

```php
[
    'users' => [
        'id' => 'BIGINT',
        'email' => 'VARCHAR(255)',
    ],
]
```

como representación arquitectónica principal.

---

## 262.6 Unknown means false

Incorrecto:

```php
$nullable = false;
```

cuando la introspección realmente no lo sabe.

Correcto:

```text
Nullability::UNKNOWN
```

---

## 262.7 Global current schema

Incorrecto:

```php
SchemaModel::$current = $schema;
```

---

## 262.8 Hidden database lookup

Incorrecto:

```php
$schema->table('users')
```

que silenciosamente consulta la DB si no encuentra la tabla.

Un modelo es conocimiento ya disponible, no un lazy database client.

---

## 262.9 ORM contamination

Incorrecto:

```php
$column->entityProperty = User::$email;
```

La relación pertenece al ORM mapping.

---

## 262.10 Migration contamination

Incorrecto:

```php
$table->rollbackSql = '...';
```

El rollback pertenece a Migration.

---

# 263. Modelo conceptual final

```text
DatabaseSchema
│
├── Identity
│   ├── DatabaseSchemaId
│   ├── SchemaModelVersion
│   └── FingerprintSet
│
├── Coverage
│   ├── COMPLETE
│   ├── PARTIAL
│   └── UNKNOWN
│
├── Catalogs
│   │
│   └── Catalog
│       │
│       └── Namespace
│           │
│           ├── Table
│           │   ├── Columns
│           │   ├── PrimaryKey
│           │   ├── Indexes
│           │   ├── Constraints
│           │   ├── ForeignKeys
│           │   ├── Options
│           │   └── Metadata
│           │
│           ├── Sequences
│           ├── Views
│           └── ExtensionObjects
│
├── Metadata
│
├── Object Registry
│
└── Derived Dependency Graph
```

---

# 264. Flujo de construcción

```text
Developer Definition
        │
        ├──────────────┐
        │              │
        ↓              │
Schema Builder         │
        ↓              │
Typed Definitions      │
        │              │
        └──────┐       │
               ↓       │
        Schema Model Factory
               ↑
               │
Native Metadata│
        ↑      │
Introspection ─┘
               │
               ↓
     Identifier Normalization
               ↓
       Object Construction
               ↓
        Local Validation
               ↓
      Reference Resolution
               ↓
      Dependency Derivation
               ↓
       Canonicalization
               ↓
        Fingerprinting
               ↓
       DatabaseSchema
```

---

# 265. Relación con Query Engine

```text
DatabaseSchema
      ↓
Schema Object Registry
      ↓
Query Semantic Engine
      ├── table resolution
      ├── column resolution
      ├── type inference
      ├── nullability facts
      ├── unique-key facts
      └── FK facts
```

sin:

```text
Query Engine
→ mutate schema
```

---

# 266. Relación con Diff

```text
Current DatabaseSchema
          +
Target DatabaseSchema
          ↓
      SchemaDiffer
          ↓
       SchemaDiff
```

---

# 267. Relación con Migrations

```text
Current Schema
      +
Target Schema
      ↓
SchemaDiff
      ↓
SchemaChangePlan
      ↓
Migration Safety
      ↓
Schema Compiler
      ↓
Execution Engine
```

---

# 268. Relación con ORM

```text
DatabaseSchema
      ↓
ORM Schema Mapping Validator
      ↑
Entity Metadata
```

La dirección de dependencia seguirá preservando:

```text
Schema Model
X
ORM dependency
```

---

# 269. Master formula

```text
Database Schema Model
=
Immutable Structural Objects
+
Structured Identifiers
+
Typed Database Types
+
Constraints
+
Indexes
+
Relationships
+
Platform-Neutral Semantics
+
Typed Platform Metadata
+
Coverage
+
Reference Resolution
+
Dependency Modeling
+
Canonicalization
+
Comparison Semantics
+
Versioned Fingerprinting
+
Extension Preservation
+
Persistent Runtime Isolation
```

---

# 270. Correctness formula

Un Schema Model correcto deberá satisfacer:

```text
CorrectSchemaModel
=
StructurallyValid
∧
IdentityConsistent
∧
ReferenceConsistent
∧
OrderingPreserved
∧
TypeSafe
∧
CoverageHonest
∧
MetadataHonest
∧
Immutable
∧
Deterministic
∧
Serializable
∧
FingerprintStable
∧
ExtensionSafe
∧
RuntimeIsolated
```

---

# 271. Structural equality formula

```text
StructuralEqual(A, B)
=
CanonicalStructure(A)
=
CanonicalStructure(B)
```

dentro del mismo modelo de comparación.

---

# 272. Semantic equality formula

```text
SemanticEqual(A, B, P)
=
Semantics(
    Normalize(A, P)
)
=
Semantics(
    Normalize(B, P)
)
```

donde:

```text
P = Platform Semantic Context
```

---

# 273. Coverage formula

Para un schema parcial:

```text
x ∉ KnownObjects(S)
```

no implica:

```text
¬Exists(x)
```

si:

```text
x ∉ Coverage(S)
```

---

# 274. Immutability formula

Para cualquier modelo publicado `S`:

```text
Mutation(S) = forbidden
```

Una transformación:

```text
T(S)
```

produce:

```text
S'
```

tal que:

```text
S' ≠ S
```

como identidad de artefacto.

---

# 275. Dependency formula

```text
SchemaDependencyGraph
=
Derive(DatabaseSchema)
```

y no:

```text
DatabaseSchema
=
Derive(SchemaDependencyGraph)
```

El modelo estructural sigue siendo la fuente primaria de verdad.

---

# 276. Fingerprint formula

Conceptualmente:

```text
StructuralFingerprint(S)
=
H(
    SchemaModelVersion,
    CanonicalizationVersion,
    CanonicalStructuralRepresentation(S)
)
```

sin:

```text
request ID
connection object
runtime tenant object
memory address
ephemeral object IDs
```

---

# 277. Principio maestro

La regla principal del sistema será:

> **El Schema Model representa conocimiento estructural inmutable; no representa instrucciones para modificar la base de datos.**

En forma compacta:

```text
Schema Model describes state.

Schema AST describes change intent.

Schema Diff discovers change.

Schema Planner orders change.

Schema Compiler represents change.

Migration governs change.

Execution Engine performs change.
```

---

# 278. Resultado arquitectónico

Con este documento VoltStack obtiene una representación canónica compartida:

```text
                   ┌─────────────────────┐
                   │   Schema Builder    │
                   └──────────┬──────────┘
                              │
                              ▼
                    Typed Definitions
                              │
                              ▼
┌─────────────────┐   ┌─────────────────────┐
│ Introspection   │──▶│    Schema Model     │
└─────────────────┘   │                     │
                      │   DatabaseSchema    │
                      └──────────┬──────────┘
                                 │
              ┌──────────────────┼───────────────────┐
              │                  │                   │
              ▼                  ▼                   ▼
      Query Semantic         Schema Diff       ORM Validation
          Engine                 │
                                 ▼
                          Schema Planning
                                 │
                                 ▼
                          Schema Compiler
                                 │
                                 ▼
                         Execution Engine
```

Esto permite que Query, Schema, Migration y ORM compartan conocimiento estructural sin compartir responsabilidades.

---

# 279. Decisión arquitectónica resultante

VoltStack adoptará:

```text
Canonical Immutable Schema Model
```

como representación estructural central del subsistema Database.

El modelo será:

```text
immutable
typed
platform-neutral at its core
capability-aware
extension-preserving
coverage-aware
deterministic
versioned
serializable
fingerprintable
persistent-runtime safe
```

y explícitamente no será:

```text
SQL representation
DDL builder
migration
live database proxy
ORM metadata model
driver abstraction
```

---

# 280. Siguiente documento

```text
89_DATABASE_SCHEMA_AST_SYSTEM.md
```

El siguiente documento deberá definir la representación operacional del Schema System:

```text
SchemaNode
├── CreateSchemaNode
├── DropSchemaNode
├── CreateTableNode
├── AlterTableNode
├── RenameTableNode
├── DropTableNode
├── AddColumnNode
├── AlterColumnNode
├── RenameColumnNode
├── DropColumnNode
├── CreateIndexNode
├── DropIndexNode
├── AddForeignKeyNode
├── DropForeignKeyNode
├── AddConstraintNode
├── DropConstraintNode
└── ExtensionSchemaNode
```

estableciendo la frontera:

```text
Schema Model
=
structural state

Schema AST
=
structural transformation intent
```

y preparando la cadena:

```text
Schema Builder
      ↓
Schema AST
      ↓
Schema Validation
      ↓
Schema Change Planning
      ↓
Schema Compiler
      ↓
Execution Engine
```