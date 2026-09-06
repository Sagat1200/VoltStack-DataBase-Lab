# 94_DATABASE_FOREIGN_KEY_SYSTEM.md

# VoltStack Quantum Database
## Database Foreign Key System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 94 — Database Foreign Key System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Foreign Key System` define la representación estructural, tipada, inmutable, portable y extensible de las claves foráneas dentro del Schema System de VoltStack.

Su pregunta fundamental es:

> **¿Cómo representa VoltStack una regla de integridad referencial entre tablas sin confundirla con una relación ORM, un JOIN, un índice o SQL específico de un motor?**

Modelo conceptual:

```text
ForeignKeyDefinition
├── ForeignKeyIdentifier
├── LocalTableReference
├── LocalColumnReferenceSet
├── ReferencedTableReference
├── ReferencedColumnReferenceSet
├── ColumnReferenceMapping
├── MatchSemantics
├── OnUpdateAction
├── OnDeleteAction
├── Deferrability
├── ValidationState
├── CapabilityRequirements
├── Metadata
└── ExtensionMetadata
```

Separación fundamental:

```text
ForeignKeyDefinition
≠
IndexDefinition
≠
RelationshipDefinition
≠
ORM Association
≠
Query Join
≠
Authorization Boundary
≠
SQL FOREIGN KEY fragment
```

---

# 2. Principio central

Una foreign key representa:

```text
Referential Integrity Constraint
```

entre un conjunto de columnas locales y un conjunto de columnas referenciadas.

Formalmente:

```text
FK(
    LocalTable(LocalColumns)
    →
    ReferencedTable(ReferencedColumns)
)
```

Ejemplo:

```text
orders.customer_id
        │
        ▼
customers.id
```

La foreign key expresa:

> Todo valor local no nulo sujeto a la semántica de la constraint debe referenciar una clave válida en la tabla objetivo.

No expresa:

```text
how ORM loads Customer
```

ni:

```text
how Query Planner joins orders and customers
```

---

# 3. Posición arquitectónica

```text
Developer DSL
     │
     ▼
ForeignKeyBlueprint
     │
     │ lowering
     ▼
ForeignKeyDefinition
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
Schema Change Planner
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

Introspection:

```text
Database Catalog
      │
      ▼
Schema Introspector
      │
      ▼
ObservedForeignKey
├── ForeignKeyDefinition
├── DefinitionCompleteness
└── ObservationMetadata
```

---

# 4. Objetivos

El sistema deberá proporcionar:

- identidad tipada de foreign keys;
- referencias estructuradas de tabla;
- referencias estructuradas de columnas;
- foreign keys simples;
- foreign keys compuestas;
- correspondencia ordenada local → referenced;
- acciones `ON DELETE`;
- acciones `ON UPDATE`;
- match semantics;
- deferrability;
- initially deferred/immediate cuando aplique;
- validation state;
- referencias internas y externas;
- capability requirements;
- metadata y provenance;
- introspection;
- canonicalization;
- comparison;
- fingerprinting;
- Schema Diff;
- serialización;
- extensibilidad;
- seguridad;
- persistent-runtime safety.

---

# 5. No objetivos

`ForeignKeyDefinition` no deberá:

- generar SQL;
- ejecutar DDL;
- abrir conexiones;
- consultar catálogos;
- realizar JOINs;
- definir ORM relationships;
- cargar entidades;
- implementar eager loading;
- implementar lazy loading;
- crear automáticamente índices sin planificación;
- decidir estrategias de migration;
- iniciar transacciones;
- resolver autorización;
- implementar cascades ORM;
- contener runtime query state.

---

# 6. Modelo principal

Se propone conceptualmente:

```php
final readonly class ForeignKeyDefinition
{
    public function __construct(
        public ForeignKeyIdentifier $identifier,
        public LocalColumnReferenceSet $localColumns,
        public ReferencedTableReference $referencedTable,
        public ReferencedColumnReferenceSet $referencedColumns,
        public ReferentialAction $onUpdate,
        public ReferentialAction $onDelete,
        public MatchSemantics $match,
        public DeferrabilityDefinition $deferrability,
        public ForeignKeyValidationState $validationState,
        public SchemaCapabilityRequirementSet $capabilities,
        public ForeignKeyMetadata $metadata,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

La API concreta podrá evolucionar.

La separación semántica deberá mantenerse.

---

# 7. ForeignKeyDefinition ≠ ForeignKeyBlueprint

La DSL puede ofrecer:

```php
$table
    ->foreign('customer_id')
    ->references('id')
    ->on('customers');
```

o:

```php
$table->foreignId('customer_id')
    ->constrained('customers');
```

El resultado inicial puede ser:

```text
ForeignKeyBlueprint
```

Posteriormente:

```text
ForeignKeyBlueprint
        │
        │ resolve / normalize / lower
        ▼
ForeignKeyDefinition
```

---

# 8. Builder convenience ≠ canonical model

Una API conveniente:

```php
$table->foreignId('customer_id')->constrained();
```

puede inferir:

```text
table = customers
column = id
```

según una naming convention.

Pero la definición canónica final deberá contener referencias explícitas:

```text
ReferencedTableReference(customers)
ReferencedColumnReference(id)
```

No deberá dejar inferencias pendientes.

---

# 9. Inmutabilidad

`ForeignKeyDefinition` será inmutable.

Incorrecto:

```php
$foreignKey->onDelete = 'cascade';
```

Correcto:

```text
ForeignKeyDefinition₀
        │
        │ transform
        ▼
ForeignKeyDefinition₁
```

---

# 10. ForeignKeyDefinition ≠ ForeignKeyOperation

Una definición representa:

```text
structural state
```

Una operación representa:

```text
structural transition
```

Por ejemplo:

```text
AddForeignKey
DropForeignKey
RenameForeignKey
ReplaceForeignKey
ValidateForeignKey
```

pertenecen al Schema AST / Schema Change Plan.

---

# 11. Foreign Key ≠ ORM Relationship

Esta separación es crítica.

Schema:

```text
ForeignKeyDefinition
```

expresa:

```text
orders.customer_id
        ↓
customers.id
```

ORM:

```text
Order::customer
```

puede expresar:

```text
ManyToOne
```

Pero:

```text
ForeignKeyDefinition
≠
ManyToOneAssociation
```

---

# 12. Razones de la separación ORM

Una relación ORM puede:

- existir sin foreign key física;
- utilizar varias columnas;
- usar condiciones adicionales;
- ser unidireccional;
- ser bidireccional;
- tener cascade persist;
- tener cascade remove;
- tener orphan removal;
- utilizar lazy loading;
- utilizar eager loading.

Nada de eso pertenece necesariamente a la foreign key.

---

# 13. Referential cascade ≠ ORM cascade

Debe distinguirse:

```text
ON DELETE CASCADE
```

de:

```text
ORM cascade remove
```

Formalmente:

```text
Database Referential Cascade
≠
ORM Persistence Cascade
```

---

# 14. Ejemplo

```text
ForeignKey:
orders.customer_id
    ON DELETE CASCADE
```

significa que la base de datos puede eliminar filas dependientes.

No significa:

```text
EntityManager automatically calls remove()
on every Order entity.
```

---

# 15. Foreign Key ≠ Query Join

Una FK puede informar al Semantic Query Engine o al Optimizer.

Pero:

```text
ForeignKeyDefinition
≠
JoinPredicate
```

Una query puede hacer:

```sql
JOIN customers
ON orders.customer_id = customers.id
```

sin foreign key.

Y una foreign key puede existir sin que una query haga JOIN.

---

# 16. Foreign Key ≠ Index

La FK expresa:

```text
referential integrity
```

El índice expresa:

```text
persistent access structure
```

Por tanto:

```text
ForeignKeyDefinition
≠
IndexDefinition
```

---

# 17. Implicit index behavior

Algunas plataformas pueden:

- requerir índice;
- crear índice automáticamente;
- reutilizar uno existente;
- aplicar reglas diferentes al lado local o referenciado.

VoltStack no deberá modelar eso como:

```text
ForeignKeyDefinition automatically owns IndexDefinition
```

---

# 18. Correct architecture

```text
ForeignKeyDefinition
        │
        ▼
Schema Validation / Planning
        │
        ├── Existing Index?
        │
        ├── Platform Requires Index?
        │
        └── Platform Creates Index Automatically?
        │
        ▼
Schema Change Plan
```

---

# 19. ForeignKeyIdentifier

Se propone:

```php
final readonly class ForeignKeyIdentifier
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 20. Identifier ≠ SQL

Internamente:

```text
fk_orders_customer
```

No:

```text
`fk_orders_customer`
```

ni:

```text
"fk_orders_customer"
```

El quoting pertenece al Schema Compiler.

---

# 21. Anonymous constraints

Algunas bases de datos permiten omitir un nombre explícito.

VoltStack deberá distinguir:

```text
ExplicitForeignKeyName
GeneratedForeignKeyName
PlatformGeneratedForeignKeyName
```

---

# 22. Internal identity

Incluso cuando el nombre DB no sea conocido:

```text
ForeignKeyDefinitionId
```

deberá existir para identidad interna.

Debe cumplirse:

```text
ForeignKeyDefinitionId
≠
ForeignKeyIdentifier
```

---

# 23. Naming strategy

Los nombres generados deberán provenir de:

```text
SchemaNamingStrategy
```

Ejemplo:

```text
orders(customer_id)
→
fk_orders_customer_id
```

---

# 24. Deterministic naming

Debe cumplirse:

```text
Same Structural Input
+
Same Naming Strategy
+
Same Naming Version
=
Same Generated Name
```

---

# 25. LocalColumnReferenceSet

El lado local será:

```text
LocalColumnReferenceSet
```

Ejemplo:

```text
orders
├── tenant_id
└── customer_id
```

---

# 26. ReferencedColumnReferenceSet

El lado remoto será:

```text
ReferencedColumnReferenceSet
```

Ejemplo:

```text
customers
├── tenant_id
└── id
```

---

# 27. Ordered mapping

El orden es semánticamente significativo.

```text
LOCAL                REFERENCED

tenant_id   ───────► tenant_id
customer_id ───────► id
```

Por tanto:

```text
(localA, localB)
→
(remoteA, remoteB)
```

no es equivalente a:

```text
(localA, localB)
→
(remoteB, remoteA)
```

---

# 28. Arity invariant

Toda foreign key deberá cumplir:

```text
|LocalColumns|
=
|ReferencedColumns|
```

---

# 29. Non-empty invariant

Debe cumplirse:

```text
|LocalColumns| >= 1
```

---

# 30. ColumnReferenceMapping

Puede ser útil materializar:

```text
ColumnReferenceMapping
```

como representación derivada:

```text
[
    local[0] → referenced[0],
    local[1] → referenced[1],
    ...
]
```

---

# 31. Mapping ≠ unordered set

Nunca representar una composite FK simplemente como:

```text
Set<ColumnIdentifier>
```

porque perdería correspondencia posicional.

---

# 32. Local references

Dentro de `TableDefinition`, las columnas locales pueden representarse mediante:

```text
LocalColumnReference
└── ColumnIdentifier
```

---

# 33. Referenced table

El objetivo deberá utilizar:

```text
ReferencedTableReference
```

No un string raw.

---

# 34. ReferencedTableReference

Conceptualmente:

```text
ReferencedTableReference
├── LocalSchemaTableReference
├── QualifiedTableReference
├── ExternalTableReference
└── UnresolvedTableReference
```

según contexto.

---

# 35. Qualified references

Una referencia podrá incluir:

```text
Catalog?
Namespace?
Table
```

sin fingir niveles que la plataforma no soporte.

---

# 36. External references

VoltStack deberá poder representar una referencia cuya tabla objetivo no se encuentre dentro del snapshot/modelo actual.

Ejemplo:

```text
orders.customer_id
       │
       ▼
external_catalog.crm.customers.id
```

---

# 37. External ≠ unresolved

Debe distinguirse:

```text
ExternalReference
```

de:

```text
UnresolvedReference
```

Una referencia externa puede ser perfectamente conocida.

Simplemente el objeto objetivo no pertenece al modelo local.

---

# 38. Partial schema snapshots

En introspection parcial:

```text
orders
```

puede conocerse mientras:

```text
customers
```

está fuera del scope introspectado.

La FK no deberá descartarse.

---

# 39. Reference resolution state

Se propone:

```text
SchemaReferenceState
├── RESOLVED
├── EXTERNAL
├── UNRESOLVED
└── UNKNOWN
```

---

# 40. UNKNOWN ≠ missing

Si introspection no pudo resolver una referencia:

```text
UNKNOWN
```

no significa:

```text
referenced table does not exist
```

---

# 41. Column type compatibility

Una FK deberá validar compatibilidad entre:

```text
LocalColumnType
```

y:

```text
ReferencedColumnType
```

---

# 42. Compatibility ≠ exact equality

No siempre será correcto exigir:

```text
LocalType === ReferencedType
```

La regla real será:

```text
ForeignKeyTypeCompatible(
    LocalType,
    ReferencedType,
    PlatformCapabilities
)
```

---

# 43. Compatibility examples

Pueden ser relevantes:

- integer width;
- signedness;
- charset;
- collation;
- string length;
- binary representation;
- UUID storage strategy;
- enum representation;
- platform-specific compatibility.

---

# 44. Local structural validation

Sin plataforma podrá verificarse:

```text
obvious incompatible categories
```

La compatibilidad final puede necesitar:

```text
SchemaCapabilitySnapshot
```

---

# 45. Referenced key eligibility

No cualquier columna remota necesariamente puede ser target válido.

Dependiendo de plataforma, puede requerirse:

```text
Primary Key
Unique Constraint
Unique Index
Candidate Key
Platform-specific eligible key
```

---

# 46. Candidate key semantics

Se recomienda representar:

```text
ReferencedKeyEligibility
```

como resultado de validación.

No introducir directamente:

```text
mustReferencePrimaryKey = true
```

como regla universal.

---

# 47. Match semantics

SQL define conceptos como:

```text
MATCH SIMPLE
MATCH FULL
MATCH PARTIAL
```

aunque el soporte real varía.

Se propone:

```php
enum MatchSemantics
{
    case PLATFORM_DEFAULT;
    case SIMPLE;
    case FULL;
    case PARTIAL;
}
```

---

# 48. PLATFORM_DEFAULT ≠ SIMPLE

No colapsar prematuramente:

```text
PLATFORM_DEFAULT
```

en:

```text
SIMPLE
```

aunque una plataforma concreta use esa semántica.

---

# 49. Composite null semantics

En foreign keys compuestas, la semántica de NULL puede depender de `MATCH`.

Ejemplo:

```text
(a, b)
```

con:

```text
a = NULL
b = 42
```

puede tratarse de forma diferente según match semantics.

---

# 50. ReferentialAction

Se propone:

```php
enum ReferentialAction
{
    case PLATFORM_DEFAULT;
    case NO_ACTION;
    case RESTRICT;
    case CASCADE;
    case SET_NULL;
    case SET_DEFAULT;
}
```

---

# 51. ON DELETE

La definición tendrá:

```text
onDelete
```

---

# 52. ON UPDATE

La definición tendrá:

```text
onUpdate
```

separadamente.

---

# 53. ON DELETE ≠ ON UPDATE

Ejemplo:

```text
ON DELETE CASCADE
ON UPDATE RESTRICT
```

es perfectamente válido conceptualmente.

---

# 54. PLATFORM_DEFAULT

Debe preservarse cuando el usuario no haya declarado acción.

No convertir prematuramente a:

```text
NO_ACTION
```

---

# 55. NO ACTION ≠ RESTRICT

Aunque algunas plataformas los traten de forma similar:

```text
NO_ACTION
≠
RESTRICT
```

en el modelo canónico.

---

# 56. CASCADE

`CASCADE` representa comportamiento referencial del motor.

No una operación ORM.

---

# 57. SET NULL

`SET_NULL` requiere que las columnas locales sean compatibles con NULL.

Debe cumplirse conceptualmente:

```text
onDelete = SET_NULL
        ↓
local columns nullable
```

cuando la plataforma requiera dicha condición.

---

# 58. SET DEFAULT

`SET_DEFAULT` puede requerir:

- defaults existentes;
- defaults compatibles;
- soporte de plataforma.

Debe generar capability requirements.

---

# 59. Action validation

Puede expresarse:

```text
ValidReferentialAction(FK)
=
ActionSupported
∧
LocalColumnStateCompatible
∧
PlatformRulesSatisfied
```

---

# 60. Deferrability

Debe modelarse explícitamente.

Se propone:

```text
DeferrabilityDefinition
├── NOT_DEFERRABLE
├── DEFERRABLE_INITIALLY_IMMEDIATE
├── DEFERRABLE_INITIALLY_DEFERRED
└── PLATFORM_DEFAULT
```

---

# 61. Deferrability ≠ transaction state

La definición dice:

```text
constraint may be deferred
```

No dice:

```text
constraint is currently deferred in this transaction
```

Eso sería runtime transaction state.

---

# 62. Runtime defer operation

Una futura operación:

```text
SET CONSTRAINTS ...
```

no pertenece a `ForeignKeyDefinition`.

---

# 63. Deferrability capability

Puede derivar:

```text
DEFERRABLE_FOREIGN_KEYS
INITIALLY_DEFERRED_FOREIGN_KEYS
```

---

# 64. Validation state

Algunas plataformas permiten crear constraints sin validar datos históricos inmediatamente.

VoltStack podrá modelar:

```text
ForeignKeyValidationState
├── VALIDATED
├── NOT_VALIDATED
├── PLATFORM_DEFAULT
└── UNKNOWN
```

cuando corresponda.

---

# 65. Validation state ≠ definition validity

Debe distinguirse:

```text
ForeignKeyDefinition is structurally valid
```

de:

```text
Database has validated all existing rows
```

---

# 66. NOT VALID example

Una migration zero-downtime podría conceptualmente:

```text
ADD FOREIGN KEY NOT VALID
        │
        ▼
validate later
```

si la plataforma lo soporta.

---

# 67. Structural property vs operation strategy

Hay que separar:

```text
Persistent Constraint State
```

de:

```text
Migration Validation Strategy
```

Por ejemplo:

```text
create unvalidated then validate later
```

puede pertenecer principalmente al Migration Planner.

---

# 68. Foreign key metadata

Se propone:

```php
final readonly class ForeignKeyMetadata
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

# 69. Provenance

Podrán distinguirse:

```text
DECLARED
GENERATED
INTROSPECTED
PLATFORM_GENERATED
INFERRED
EXTENSION
UNKNOWN
```

---

# 70. Metadata ≠ semantics

Source line:

```text
database/schema/orders.php:45
```

no forma parte de la semántica referencial.

---

# 71. Capability requirements

Podrán derivarse de:

```text
match semantics
on delete
on update
deferrability
validation state
cross-schema reference
cross-catalog reference
extension metadata
```

---

# 72. Example capability set

```text
ForeignKeyDefinition
├── MATCH FULL
├── ON DELETE SET DEFAULT
└── DEFERRABLE INITIALLY DEFERRED
```

puede requerir:

```text
FOREIGN_KEYS
FOREIGN_KEY_MATCH_FULL
FOREIGN_KEY_SET_DEFAULT
DEFERRABLE_FOREIGN_KEYS
INITIALLY_DEFERRED_FOREIGN_KEYS
```

---

# 73. Requirements ≠ checks

La definición declara requisitos.

No ejecuta:

```php
$connection->supportsDeferrableForeignKeys();
```

---

# 74. Capability validation

Flujo:

```text
ForeignKeyDefinition
        │
        ▼
ForeignKeyCapabilityRequirementSet
        │
        ▼
Schema Platform Compatibility
        │
        ▼
SchemaCapabilitySnapshot
```

---

# 75. MySQL

El sistema deberá representar correctamente capacidades y limitaciones de MySQL sin contaminar el core.

No:

```php
if ($driver === 'mysql') {
    ...
}
```

disperso por el Foreign Key System.

---

# 76. MariaDB

MariaDB deberá ser tratada como plataforma first-class.

No simplemente:

```text
MariaDB = MySQL alias
```

---

# 77. PostgreSQL

Características como:

```text
DEFERRABLE
INITIALLY DEFERRED
NOT VALID
```

deberán expresarse mediante capabilities/compilers correspondientes.

---

# 78. SQLite

Las capacidades y restricciones de SQLite alrededor de foreign keys y ALTER TABLE deberán permanecer explícitas.

El core no deberá fingir soporte universal.

---

# 79. Schema dependency graph

Toda FK introduce una dependencia estructural.

Ejemplo:

```text
orders
  │
  │ FK customer_id
  ▼
customers
```

Por tanto:

```text
orders ─────► customers
```

dentro del `SchemaDependencyGraph`.

---

# 80. Dependency semantics

Para crear:

```text
orders FK → customers
```

normalmente:

```text
customers
must exist before
FK creation
```

---

# 81. Table creation ordering

Ejemplo:

```text
customers
    │
    ▼
orders
```

puede permitir:

```text
CREATE customers
CREATE orders with FK
```

---

# 82. Cyclic foreign keys

Considere:

```text
A ───► B
▲      │
│      ▼
└──────C
```

o simplemente:

```text
A ───► B
B ───► A
```

VoltStack deberá detectar ciclos.

---

# 83. Cycles are not invalid by definition

Un ciclo referencial puede ser estructuralmente válido.

Lo que cambia es la estrategia de creación.

---

# 84. Cycle planning

El planner puede decidir:

```text
CREATE TABLE A without FK
CREATE TABLE B without FK

ADD FK A → B
ADD FK B → A
```

cuando la plataforma lo permita.

---

# 85. Compiler must not invent cycle strategy

La decisión de separar constraints deberá estar visible en:

```text
SchemaChangePlan
```

No ocultarse dentro del compiler.

---

# 86. Foreign key graph

Puede derivarse:

```text
ForeignKeyDependencyGraph
```

como vista del Schema Model.

No es necesario almacenar un grafo mutable duplicado dentro de cada tabla.

---

# 87. Self-referencing foreign keys

Debe soportarse:

```text
employees.manager_id
        │
        ▼
employees.id
```

---

# 88. Self-reference ≠ graph cycle failure

Una self-reference no deberá romper automáticamente el dependency planner.

Necesita semántica específica.

---

# 89. Composite foreign key

Ejemplo:

```text
invoice_items
├── tenant_id
└── invoice_id
       │
       ▼
invoices
├── tenant_id
└── id
```

Mapping:

```text
invoice_items.tenant_id  ──► invoices.tenant_id
invoice_items.invoice_id ──► invoices.id
```

---

# 90. Composite order invariant

Debe preservarse:

```text
(local tenant_id, invoice_id)
→
(remote tenant_id, id)
```

No ordenar alfabéticamente estas listas durante canonicalization.

---

# 91. Canonicalization

Debe existir:

```text
ForeignKeyDefinitionNormalizer
```

---

# 92. Normalization properties

Será:

```text
deterministic
idempotent
side-effect free
semantics-preserving
```

---

# 93. Normalization must preserve mapping order

Incorrecto:

```text
sort(localColumns)
sort(referencedColumns)
```

independientemente.

Eso puede cambiar la correspondencia.

---

# 94. Correct canonicalization

La unidad ordenada debe ser:

```text
ColumnReferencePair
```

cuando se canonicalice un mapping.

---

# 95. Normalization must not resolve DB state secretly

No deberá consultar:

- current database;
- current schema;
- current indexes;
- current constraints;
- server version.

---

# 96. Explicit normalization context

Si se necesita información externa:

```text
ForeignKeyNormalizationContext
```

deberá recibirla explícitamente.

---

# 97. Structural equality

Deben distinguirse:

```text
Object Identity
Identifier Equality
Structural Equality
Name-Independent Structural Equality
Platform Equivalence
Migration Equivalence
```

---

# 98. Structural comparison

Puede considerar:

```text
local column mapping
referenced table
referenced columns
match semantics
on update
on delete
deferrability
validation state
structural extensions
```

---

# 99. Name-independent comparison

Schema Diff podrá comparar:

```text
fk_old
```

y:

```text
fk_new
```

ignorando temporalmente el nombre para detectar posible rename.

Pero:

```text
NameIndependentEquivalent
≠
Same ForeignKey
```

---

# 100. Rename inference

No deberá asumirse:

```text
same mapping
+
different name
=
rename
```

Puede representar drop/create deliberado.

---

# 101. Foreign key fingerprint

Se propone:

```text
ForeignKeyStructuralFingerprint
```

---

# 102. Fingerprint inputs

Puede incluir:

```text
identifier
ordered local columns
referenced table path
ordered referenced columns
match
on update
on delete
deferrability
validation semantics
structural extensions
normalization version
```

---

# 103. Fingerprint profiles

Se proponen:

```text
FULL
NAME_INDEPENDENT
PORTABLE
MIGRATION
CACHE
DEPENDENCY
```

---

# 104. Fingerprint ≠ SQL hash

Nunca:

```text
hash(
    "FOREIGN KEY (...) REFERENCES ..."
)
```

como única identidad estructural.

---

# 105. Fingerprint collision rule

Debe cumplirse:

```text
Fingerprint(A) = Fingerprint(B)
```

es:

```text
candidate equality
```

No prueba matemática de igualdad.

---

# 106. Serialization

La definición deberá tener formato versionado y determinista.

Ejemplo:

```json
{
  "name": "fk_orders_customer",
  "localColumns": [
    "customer_id"
  ],
  "referencedTable": {
    "table": "customers"
  },
  "referencedColumns": [
    "id"
  ],
  "match": "PLATFORM_DEFAULT",
  "onUpdate": "PLATFORM_DEFAULT",
  "onDelete": "CASCADE",
  "deferrability": "PLATFORM_DEFAULT"
}
```

---

# 107. Serialization round-trip

Debe cumplirse:

```text
StructuralEqual(
    FK,
    Deserialize(Serialize(FK))
)
```

para formatos compatibles.

---

# 108. No live resources in serialization

Nunca serializar:

```text
Connection
PDO
Transaction
Request
TenantContext mutable
Driver metadata handle
```

---

# 109. Extension serialization

Toda extensión estructural deberá preservar:

```text
ExtensionId
ExtensionVersion
PayloadVersion
StructuralImpact
```

---

# 110. Validation architecture

Se propone:

```php
interface ForeignKeyDefinitionValidator
{
    public function validate(
        ForeignKeyDefinition $foreignKey,
        ForeignKeyValidationContext $context,
    ): ForeignKeyValidationResult;
}
```

---

# 111. Local validation

Podrá comprobar:

- identifier válido;
- local column set no vacío;
- referenced column set no vacío;
- arity compatible;
- duplicate local columns;
- duplicate referenced columns;
- valid actions;
- valid match semantics;
- valid deferrability representation;
- valid extensions.

---

# 112. Table-context validation

Podrá comprobar:

```text
local columns exist
local columns are structurally valid
local duplicate mappings
self-reference validity
```

---

# 113. Schema-context validation

Podrá comprobar:

```text
referenced table exists
referenced columns exist
referenced key eligibility
type compatibility
dependency graph consistency
cross-schema validity
```

---

# 114. Platform validation

Podrá comprobar:

```text
foreign key support
referential action support
match support
deferrability support
cross-schema support
cross-catalog support
validation-state support
type-specific platform restrictions
```

---

# 115. Validation hierarchy

```text
Local ForeignKey Validation
            ↓
Local Table Validation
            ↓
Schema Reference Resolution
            ↓
Referenced Key Validation
            ↓
Type Compatibility
            ↓
Platform Capability Validation
            ↓
Schema Change Planning
            ↓
Migration Safety
            ↓
Execution Feasibility
```

---

# 116. ForeignKeyValidationResult

```php
final readonly class ForeignKeyValidationResult
{
    public function __construct(
        public bool $valid,
        public SchemaDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 117. Diagnostic examples

```text
Foreign key "fk_orders_customer" references unknown local column "customer_id".

Foreign key "fk_invoice_item_invoice" contains 2 local columns but 1 referenced column.

Foreign key "fk_orders_customer" references a non-eligible target key.

Foreign key "fk_orders_customer" uses SET NULL but local column "customer_id" is NOT NULL.

Foreign key "fk_a_b" requires deferrable-constraint support unavailable on the selected platform.
```

---

# 118. Error hierarchy

Se propone:

```text
DatabaseForeignKeyException
├── InvalidForeignKeyDefinitionException
├── InvalidForeignKeyIdentifierException
├── EmptyForeignKeyColumnSetException
├── ForeignKeyArityMismatchException
├── DuplicateForeignKeyColumnException
├── UnknownLocalColumnException
├── UnknownReferencedTableException
├── UnknownReferencedColumnException
├── UnresolvedForeignKeyReferenceException
├── IncompatibleForeignKeyColumnTypeException
├── InvalidReferencedKeyException
├── InvalidReferentialActionException
├── InvalidForeignKeyMatchException
├── InvalidForeignKeyDeferrabilityException
├── ForeignKeyCapabilityException
├── ForeignKeyDependencyException
├── ForeignKeySerializationException
├── ForeignKeyExtensionException
├── ForeignKeyBudgetExceededException
└── ForeignKeyInvariantException
```

---

# 119. Invalid definition ≠ unsupported definition

Debe diferenciarse:

```text
Structurally Invalid ForeignKey
```

de:

```text
Valid ForeignKey
but unsupported by Platform
```

Ejemplo:

```text
DEFERRABLE INITIALLY DEFERRED
```

puede ser estructuralmente válido pero no soportado por una plataforma.

---

# 120. ObservedForeignKey

Introspection deberá utilizar un wrapper:

```php
final readonly class ObservedForeignKey
{
    public function __construct(
        public ForeignKeyDefinition $definition,
        public DefinitionCompleteness $completeness,
        public ForeignKeyObservationMetadata $observation,
    ) {}
}
```

---

# 121. Introspection uncertainty

Puede conocerse:

```text
local columns = exact
referenced table = exact
referenced columns = exact
onDelete = exact
onUpdate = exact
deferrability = unknown
match = driver-reported
```

La incertidumbre deberá conservarse.

---

# 122. UNKNOWN ≠ DEFAULT

Si el driver no reporta `MATCH`:

```text
UNKNOWN
```

no debe convertirse automáticamente en:

```text
PLATFORM_DEFAULT
```

---

# 123. UNKNOWN ≠ NOT DEFERRABLE

Igualmente:

```text
deferrability unavailable
```

no implica:

```text
NOT_DEFERRABLE
```

---

# 124. Native metadata

Puede conservarse:

```text
NativeForeignKeyMetadata
```

como observation metadata para evitar pérdida de información.

---

# 125. Opaque platform metadata

Características desconocidas deberán seguir la política general:

```text
FAIL
PRESERVE_OPAQUE
IGNORE_WITH_DIAGNOSTIC
```

No silent loss.

---

# 126. Schema Diff integration

Schema Diff podrá producir:

```text
ForeignKeyDiff
├── ForeignKeyAdded
├── ForeignKeyRemoved
├── ForeignKeyRenamed
├── LocalColumnsChanged
├── ReferencedTableChanged
├── ReferencedColumnsChanged
├── MatchChanged
├── OnDeleteChanged
├── OnUpdateChanged
├── DeferrabilityChanged
├── ValidationStateChanged
└── IndeterminateForeignKeyChange
```

---

# 127. FK alteration

Muchas plataformas no permiten modificar directamente una FK.

Por tanto:

```text
FK₀
 ↓
FK₁
```

puede requerir:

```text
DROP FK
ADD FK
```

Pero esta decisión pertenece al Schema Planner.

---

# 128. No hidden drop/recreate

El compiler no deberá recibir:

```text
AlterForeignKey
```

y silenciosamente ejecutar:

```text
DROP
+
CREATE
```

si esa transformación tiene implicaciones operacionales.

Debe estar visible en `SchemaChangePlan`.

---

# 129. Migration safety

Agregar una FK a una tabla existente puede:

- escanear datos;
- bloquear;
- fallar por datos inválidos;
- requerir índices;
- consumir recursos;
- afectar disponibilidad.

La definición no decide si el cambio es seguro.

---

# 130. Safety classification downstream

El Migration Safety System podrá clasificar:

```text
SAFE
CONDITIONALLY_SAFE
POTENTIALLY_BLOCKING
DATA_VALIDATION_REQUIRED
RESOURCE_INTENSIVE
DESTRUCTIVE
UNKNOWN
```

---

# 131. Zero-downtime integration

Una futura estrategia puede ser:

```text
Add FK without validation
        ↓
Validate existing data asynchronously/controlled
        ↓
Mark validated
```

si la plataforma lo soporta.

Esto pertenece a:

```text
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM
```

no al core `ForeignKeyDefinition`.

---

# 132. Referential actions and mutation planning

El Query/Persistence Engine puede beneficiarse de saber que existe:

```text
ON DELETE CASCADE
```

pero no deberá asumir automáticamente que el ORM puede omitir toda coordinación.

---

# 133. Database cascade knowledge

Podrá proyectarse:

```text
ForeignKeyDefinition
        ↓
Persistence Referential Metadata
```

para que el ORM/Persistence Engine conozca efectos potenciales.

---

# 134. But no circular dependency

Debe mantenerse:

```text
ORM
 ↓
Schema Metadata
```

y nunca:

```text
Schema
 ↓
ORM
```

---

# 135. Query optimizer integration

Las FKs pueden proporcionar facts como:

```text
referential dependency
potential join cardinality facts
existence guarantees under nullability assumptions
```

Pero sólo mediante una proyección semántica validada.

---

# 136. FK ≠ optimizer proof by itself

Una FK no siempre permite concluir:

```text
every local row has matching remote row
```

sin considerar:

- NULL semantics;
- validation state;
- disabled constraints;
- partial/unknown metadata;
- transaction semantics;
- platform behavior.

---

# 137. Constraint trust

Puede ser necesario un descriptor:

```text
ConstraintTrustState
├── TRUSTED
├── UNVALIDATED
├── DISABLED
├── UNKNOWN
└── PLATFORM_SPECIFIC
```

para consumidores analíticos.

---

# 138. Optimizer safety

Sólo constraints con evidencia suficiente deberán utilizarse para optimizaciones que dependan de integridad referencial.

---

# 139. Multitenancy

Core Database no dependerá de Multitenancy.

Una FK podrá ser:

```text
tenant-local
```

simplemente porque pertenece al schema de ese tenant.

---

# 140. Cross-tenant FK

VoltStack no deberá inventar cross-tenant references automáticamente.

Cualquier integración deberá pasar por el paquete Multitenancy y las capabilities del storage model.

---

# 141. Tenant isolation

Nunca deberá resolverse una referencia:

```text
tenant A orders
→
tenant B customers
```

por simple coincidencia de nombres.

---

# 142. Security

Los identifiers deberán ser:

```text
typed
validated
compiler-quoted
```

---

# 143. No identifier interpolation

Incorrecto:

```php
$sql = "FOREIGN KEY ($column) REFERENCES $table($target)";
```

con valores arbitrarios.

---

# 144. Correct path

```text
User/DSL Input
      ↓
Typed Identifier Validation
      ↓
ForeignKeyDefinition
      ↓
Schema AST
      ↓
Schema Compiler
      ↓
Dialect Identifier Rendering
```

---

# 145. Raw references

No deberá existir un modo implícito:

```php
referencesRaw($_GET['reference']);
```

sin trust boundary explícito.

---

# 146. Persistent runtime safety

`ForeignKeyDefinition` será:

```text
immutable
connection-free
request-free
transaction-free
statement-free
tenant-state-free
```

---

# 147. Safe sharing

Una definición inmutable puede compartirse entre requests cuando su scope estructural lo permita.

---

# 148. Unsafe global state

Nunca:

```php
ForeignKeyRegistry::$currentTenantForeignKeys
```

como mutable global state.

---

# 149. Runtime neutrality

La arquitectura será segura bajo:

```text
PHP-FPM
FrankenPHP
RoadRunner
OpenSwoole
CLI
Testing
```

---

# 150. Extension system

Una extensión podrá proporcionar:

```text
CustomReferentialAction
CustomMatchSemantics
CustomForeignKeyOption
CustomReferenceType
CustomMetadata
CustomCapabilityRequirement
CustomCompilerContribution
```

---

# 151. Typed extensions

Toda extensión tendrá:

```text
ExtensionId
ExtensionVersion
StructuralImpact
CapabilityRequirements
```

---

# 152. Frozen extension registry

La interpretación dependerá de:

```text
FrozenForeignKeyExtensionRegistry
```

---

# 153. No last-wins

Dos extensiones incompatibles deberán producir:

```text
ForeignKeyExtensionConflict
```

---

# 154. Unknown structural extensions

No podrán ignorarse durante:

```text
comparison
fingerprinting
serialization
diff
compilation
```

---

# 155. Foreign key budget

Se propone:

```php
final readonly class ForeignKeyDefinitionBudget
{
    public function __construct(
        public int $maxColumns,
        public int $maxReferenceDepth,
        public int $maxOptions,
        public int $maxExtensions,
        public int $maxMetadataBytes,
    ) {}
}
```

---

# 156. Budget overflow

Debe fallar explícitamente.

Nunca truncar una composite FK.

---

# 157. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\ForeignKey
```

---

# 158. Proposed directory structure

```text
Schema/
└── ForeignKey/
    ├── Contract/
    │   ├── ForeignKeyDefinitionFactory.php
    │   ├── ForeignKeyDefinitionNormalizer.php
    │   ├── ForeignKeyDefinitionValidator.php
    │   ├── ForeignKeyDefinitionComparator.php
    │   └── ForeignKeyNamingStrategy.php
    │
    ├── Definition/
    │   ├── ForeignKeyDefinition.php
    │   └── ForeignKeyDefinitionId.php
    │
    ├── Identifier/
    │   ├── ForeignKeyIdentifier.php
    │   └── QualifiedForeignKeyIdentifier.php
    │
    ├── Reference/
    │   ├── LocalColumnReference.php
    │   ├── LocalColumnReferenceSet.php
    │   ├── ReferencedTableReference.php
    │   ├── ReferencedColumnReference.php
    │   ├── ReferencedColumnReferenceSet.php
    │   ├── ColumnReferencePair.php
    │   ├── ColumnReferenceMapping.php
    │   ├── SchemaReferenceState.php
    │   ├── ExternalTableReference.php
    │   └── UnresolvedTableReference.php
    │
    ├── Action/
    │   └── ReferentialAction.php
    │
    ├── Match/
    │   └── MatchSemantics.php
    │
    ├── Deferrability/
    │   └── DeferrabilityDefinition.php
    │
    ├── ValidationState/
    │   ├── ForeignKeyValidationState.php
    │   └── ConstraintTrustState.php
    │
    ├── Metadata/
    │   ├── ForeignKeyMetadata.php
    │   ├── ObservedForeignKey.php
    │   ├── ForeignKeyObservationMetadata.php
    │   └── NativeForeignKeyMetadata.php
    │
    ├── Capability/
    │   └── ForeignKeyCapabilityRequirementResolver.php
    │
    ├── Dependency/
    │   ├── ForeignKeyDependency.php
    │   └── ForeignKeyDependencyGraphProjection.php
    │
    ├── Naming/
    │   └── DefaultForeignKeyNamingStrategy.php
    │
    ├── Validation/
    │   ├── ForeignKeyValidationContext.php
    │   ├── ForeignKeyValidationResult.php
    │   ├── ForeignKeyTypeCompatibilityValidator.php
    │   └── ReferencedKeyEligibilityValidator.php
    │
    ├── Comparison/
    │   ├── ForeignKeyComparisonProfile.php
    │   └── ForeignKeyComparisonResult.php
    │
    ├── Fingerprint/
    │   ├── ForeignKeyFingerprintGenerator.php
    │   ├── ForeignKeyStructuralFingerprint.php
    │   └── ForeignKeyFingerprintProfile.php
    │
    ├── Serialization/
    │   ├── ForeignKeyDefinitionSerializer.php
    │   └── ForeignKeyDefinitionDeserializer.php
    │
    ├── Extension/
    │   ├── ForeignKeyExtension.php
    │   └── FrozenForeignKeyExtensionRegistry.php
    │
    ├── Budget/
    │   └── ForeignKeyDefinitionBudget.php
    │
    └── Exception/
        ├── DatabaseForeignKeyException.php
        ├── InvalidForeignKeyDefinitionException.php
        ├── InvalidForeignKeyIdentifierException.php
        ├── EmptyForeignKeyColumnSetException.php
        ├── ForeignKeyArityMismatchException.php
        ├── DuplicateForeignKeyColumnException.php
        ├── UnknownLocalColumnException.php
        ├── UnknownReferencedTableException.php
        ├── UnknownReferencedColumnException.php
        ├── UnresolvedForeignKeyReferenceException.php
        ├── IncompatibleForeignKeyColumnTypeException.php
        ├── InvalidReferencedKeyException.php
        ├── InvalidReferentialActionException.php
        ├── InvalidForeignKeyMatchException.php
        ├── InvalidForeignKeyDeferrabilityException.php
        ├── ForeignKeyCapabilityException.php
        ├── ForeignKeyDependencyException.php
        ├── ForeignKeySerializationException.php
        ├── ForeignKeyExtensionException.php
        ├── ForeignKeyBudgetExceededException.php
        └── ForeignKeyInvariantException.php
```

---

# 159. Dependency rules

Permitido:

```text
ForeignKeyDefinition
        ↓
Typed Schema Identifiers
Column References
Schema Capability Requirements
Immutable Metadata
Schema Reference Contracts
Schema Expression/Option primitives
```

No permitido:

```text
ForeignKeyDefinition
        ↓
PDO
Connection
DriverStatement
EntityManager
ORM Relationship
UnitOfWork
Query Planner
Migration Runner
HTTP Request
Mutable TenantContext
```

---

# 160. Testing strategy

La mayor parte del sistema deberá probarse sin una DB real.

---

# 161. Simple FK tests

```text
orders.customer_id
→
customers.id
```

Verificar:

- identity;
- mapping;
- defaults;
- serialization;
- fingerprint.

---

# 162. Composite FK tests

```text
(a, b)
→
(x, y)
```

Verificar:

```text
a → x
b → y
```

---

# 163. Arity tests

Debe fallar:

```text
(a, b)
→
(x)
```

---

# 164. Ordering tests

Debe comprobarse que:

```text
(a,b) → (x,y)
```

no sea equivalente a:

```text
(a,b) → (y,x)
```

---

# 165. Action tests

Probar:

```text
NO_ACTION
RESTRICT
CASCADE
SET_NULL
SET_DEFAULT
PLATFORM_DEFAULT
```

---

# 166. SET NULL tests

Verificar incompatibilidad con local columns `NOT NULL` cuando corresponda.

---

# 167. Deferrability tests

Probar:

```text
NOT_DEFERRABLE
DEFERRABLE_INITIALLY_IMMEDIATE
DEFERRABLE_INITIALLY_DEFERRED
PLATFORM_DEFAULT
```

---

# 168. Reference tests

Probar:

```text
resolved reference
external reference
unresolved reference
partial snapshot
self reference
cross-schema reference
```

---

# 169. Dependency cycle tests

Probar:

```text
A → B
B → A
```

y:

```text
A → A
```

sin convertirlos automáticamente en schema inválido.

---

# 170. Introspection tests

Para cada plataforma:

```text
Create FK
   ↓
Introspect
   ↓
ObservedForeignKey
```

comparando metadata normalizada.

---

# 171. Cross-platform conformance

La suite deberá cubrir:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

con capability-specific expectations.

---

# 172. Fingerprint tests

Debe cumplirse:

```text
Same Canonical FK
+
Same Fingerprint Profile
=
Same Fingerprint
```

---

# 173. Serialization tests

```text
FK
 ↓ serialize
Payload
 ↓ deserialize
FK'
```

y:

```text
StructuralEqual(FK, FK')
```

---

# 174. Persistent worker tests

Simular múltiples requests en un mismo worker:

```text
Request A
Request B
Request C
```

sin leakage de:

```text
connection
tenant
transaction
validation state
runtime metadata
```

---

# 175. Architectural invariants

## DB-FK-001

Toda foreign key tendrá identidad estructural interna.

## DB-FK-002

`ForeignKeyDefinition` será inmutable.

## DB-FK-003

`ForeignKeyDefinition` será distinta de `ForeignKeyBlueprint`.

## DB-FK-004

`ForeignKeyDefinition` será distinta de `ForeignKeyOperation`.

## DB-FK-005

`ForeignKeyDefinition` será distinta de SQL.

## DB-FK-006

Foreign Key System no generará SQL directamente.

## DB-FK-007

Foreign Key System no ejecutará DDL.

## DB-FK-008

Foreign Key System no abrirá conexiones.

## DB-FK-009

Foreign Key será distinta de Index.

## DB-FK-010

Foreign Key será distinta de ORM Relationship.

## DB-FK-011

Foreign Key será distinta de Query Join.

## DB-FK-012

Foreign Key será distinta de Authorization Boundary.

## DB-FK-013

Database cascade será distinta de ORM cascade.

## DB-FK-014

ForeignKeyIdentifier será tipado.

## DB-FK-015

Identifier quoting pertenecerá al compiler.

## DB-FK-016

ForeignKeyDefinitionId será distinto de ForeignKeyIdentifier.

## DB-FK-017

Generated FK names serán deterministas.

## DB-FK-018

Generated naming pertenecerá a SchemaNamingStrategy.

## DB-FK-019

Explicit names no serán modificados silenciosamente.

## DB-FK-020

Toda FK tendrá al menos una columna local.

## DB-FK-021

Toda FK tendrá al menos una columna referenciada.

## DB-FK-022

Local y referenced arity deberán coincidir.

## DB-FK-023

Column mapping será ordenado.

## DB-FK-024

Composite FK mapping no será representado como unordered set.

## DB-FK-025

Duplicate local columns serán inválidas salvo extensión explícita.

## DB-FK-026

Duplicate referenced mappings serán validados explícitamente.

## DB-FK-027

Local columns deberán existir en table context.

## DB-FK-028

Referenced columns deberán resolverse o conservar estado unresolved/external.

## DB-FK-029

External reference será distinta de unresolved reference.

## DB-FK-030

UNKNOWN será distinto de missing.

## DB-FK-031

Partial snapshot no implicará referenced-object absence.

## DB-FK-032

Column compatibility será explícita.

## DB-FK-033

Column compatibility no requerirá exact type identity universalmente.

## DB-FK-034

Referenced key eligibility será validada.

## DB-FK-035

Referenced target no se asumirá siempre Primary Key.

## DB-FK-036

Match semantics será explícita.

## DB-FK-037

PLATFORM_DEFAULT será distinto de SIMPLE.

## DB-FK-038

Composite NULL semantics no se simplificarán incorrectamente.

## DB-FK-039

ON DELETE será distinto de ON UPDATE.

## DB-FK-040

PLATFORM_DEFAULT action será preservada.

## DB-FK-041

NO_ACTION será distinto de RESTRICT.

## DB-FK-042

CASCADE representará database referential behavior.

## DB-FK-043

CASCADE no representará ORM cascade.

## DB-FK-044

SET_NULL requerirá validación de nullability.

## DB-FK-045

SET_DEFAULT requerirá validación de defaults/capabilities.

## DB-FK-046

Deferrability será estructurada.

## DB-FK-047

Deferrability será distinta de runtime transaction state.

## DB-FK-048

Constraint validation state será distinta de definition validity.

## DB-FK-049

NOT_VALIDATED no significará invalid definition.

## DB-FK-050

Capability requirements serán declarativos.

## DB-FK-051

Capability requirements serán distintos de platform checks.

## DB-FK-052

ForeignKeyDefinition no consultará DB capabilities ocultamente.

## DB-FK-053

MySQL behavior estará encapsulado por platform/capability layers.

## DB-FK-054

MariaDB será plataforma first-class.

## DB-FK-055

PostgreSQL-specific semantics serán capability-driven.

## DB-FK-056

SQLite limitations serán explícitas.

## DB-FK-057

Toda FK producirá dependencias estructurales derivables.

## DB-FK-058

Dependency graph no será mutable state duplicado dentro de FK.

## DB-FK-059

Cyclic FKs no serán automáticamente inválidas.

## DB-FK-060

Cycle handling pertenecerá al Schema Planner.

## DB-FK-061

Compiler no inventará hidden cycle resolution.

## DB-FK-062

Self-referencing FK será soportable.

## DB-FK-063

Self-reference no será automáticamente dependency error.

## DB-FK-064

Canonicalization será determinista.

## DB-FK-065

Canonicalization será idempotente.

## DB-FK-066

Canonicalization preservará semántica.

## DB-FK-067

Canonicalization preservará mapping order.

## DB-FK-068

Canonicalization no consultará DB ocultamente.

## DB-FK-069

Structural equality será distinta de identity equality.

## DB-FK-070

Structural equality será distinta de platform equivalence.

## DB-FK-071

Structural equality será distinta de migration equivalence.

## DB-FK-072

Name-independent equality no probará rename.

## DB-FK-073

Rename intent pertenecerá al Schema Operation layer.

## DB-FK-074

Fingerprint será estructural.

## DB-FK-075

Fingerprint no será SQL hash.

## DB-FK-076

Fingerprint será determinista.

## DB-FK-077

Fingerprint será versionado.

## DB-FK-078

Fingerprint podrá ignorar name mediante profile explícito.

## DB-FK-079

Fingerprint excluirá request identity.

## DB-FK-080

Fingerprint excluirá connection identity.

## DB-FK-081

Fingerprint excluirá runtime transaction state.

## DB-FK-082

Serialization será determinista.

## DB-FK-083

Serialization será versionada.

## DB-FK-084

Serialization preservará ordered mapping.

## DB-FK-085

Serialization preservará structural extensions.

## DB-FK-086

Serialization no contendrá live resources.

## DB-FK-087

ObservedForeignKey será distinto de ForeignKeyDefinition.

## DB-FK-088

Introspection uncertainty será explícita.

## DB-FK-089

UNKNOWN será distinto de PLATFORM_DEFAULT.

## DB-FK-090

UNKNOWN deferrability no será convertida a NOT_DEFERRABLE.

## DB-FK-091

Opaque platform metadata no se perderá silenciosamente.

## DB-FK-092

Schema Diff preservará uncertainty.

## DB-FK-093

ForeignKeyDefinition no decidirá DROP/CREATE strategy.

## DB-FK-094

ForeignKeyDefinition no decidirá validation strategy.

## DB-FK-095

ForeignKeyDefinition no decidirá zero-downtime strategy.

## DB-FK-096

ForeignKeyDefinition no decidirá lock strategy.

## DB-FK-097

ForeignKeyDefinition no decidirá transaction boundaries.

## DB-FK-098

ForeignKeyDefinition no decidirá migration safety.

## DB-FK-099

ForeignKeyDefinition no contendrá ORM metadata.

## DB-FK-100

ForeignKeyDefinition no contendrá EntityManager.

## DB-FK-101

ForeignKeyDefinition no contendrá UnitOfWork.

## DB-FK-102

ForeignKeyDefinition no contendrá lazy-loading state.

## DB-FK-103

ForeignKeyDefinition no contendrá eager-loading state.

## DB-FK-104

ForeignKeyDefinition no contendrá query join nodes.

## DB-FK-105

ForeignKeyDefinition no contendrá planner cost.

## DB-FK-106

ForeignKeyDefinition no contendrá execution state.

## DB-FK-107

ForeignKeyDefinition no contendrá PDO.

## DB-FK-108

ForeignKeyDefinition no contendrá Connection.

## DB-FK-109

ForeignKeyDefinition no contendrá DriverStatement.

## DB-FK-110

ForeignKeyDefinition no contendrá Request.

## DB-FK-111

ForeignKeyDefinition no contendrá mutable TenantContext.

## DB-FK-112

ForeignKeyDefinition será persistent-runtime safe.

## DB-FK-113

Extensions serán tipadas.

## DB-FK-114

Extensions tendrán stable IDs.

## DB-FK-115

Extension registry será frozen.

## DB-FK-116

No habrá last-wins extension behavior.

## DB-FK-117

Extension conflicts serán explícitos.

## DB-FK-118

Unknown structural extensions no se ignorarán.

## DB-FK-119

Identifiers serán validados.

## DB-FK-120

Identifiers serán rendered por Dialect/Compiler.

## DB-FK-121

FK no creará IndexDefinition automáticamente dentro del modelo.

## DB-FK-122

Required index handling pertenecerá a platform planning.

## DB-FK-123

Implicit platform indexes preservarán provenance.

## DB-FK-124

Foreign key facts podrán proyectarse al Semantic Engine.

## DB-FK-125

Semantic consumers deberán considerar NULL semantics.

## DB-FK-126

Semantic consumers deberán considerar validation/trust state.

## DB-FK-127

Unvalidated FK no será usada como universal optimizer proof.

## DB-FK-128

Disabled/unknown FK no será usada como trusted proof.

## DB-FK-129

ORM podrá consumir FK metadata.

## DB-FK-130

Schema System no dependerá del ORM.

## DB-FK-131

Multitenancy será integración externa.

## DB-FK-132

Cross-tenant references no serán inferidas automáticamente.

## DB-FK-133

Tenant identity no se resolverá por simple table-name equality.

## DB-FK-134

Definition budgets serán explícitos.

## DB-FK-135

Budget overflow no truncará composite mappings.

## DB-FK-136

Structural validation será posible sin DB.

## DB-FK-137

Platform validation será capability-driven.

## DB-FK-138

Cross-platform conformance será testeada.

## DB-FK-139

Persistent worker isolation será testeada.

## DB-FK-140

ForeignKeyDefinition representará referential structural intent.

## DB-FK-141

ForeignKeyOperation representará structural transition.

## DB-FK-142

ORM Relationship representará object association.

## DB-FK-143

Query Join representará query relational operation.

## DB-FK-144

Estas representaciones permanecerán separadas.

## DB-FK-145

Foreign Key System será runtime-neutral.

## DB-FK-146

Foreign Key System será driver-independent.

## DB-FK-147

Foreign Key System será SQL-independent.

## DB-FK-148

Foreign Key System será ORM-independent.

## DB-FK-149

Foreign Key System será Query Planner-independent.

## DB-FK-150

`ForeignKeyDefinition` será la representación canónica de integridad referencial dentro del Schema System de VoltStack.

---

# 176. Anti-patterns

## 176.1 Foreign key como SQL

Incorrecto:

```php
$fk = 'FOREIGN KEY (customer_id) REFERENCES customers(id)';
```

---

## 176.2 ORM relation como FK

Incorrecto:

```php
$fk = new BelongsTo(
    Order::class,
    Customer::class
);
```

---

## 176.3 FK como JOIN

Incorrecto:

```php
$fk->joinCondition = 'orders.customer_id = customers.id';
```

---

## 176.4 FK como index

Incorrecto:

```php
$table->index('customer_id')->foreign();
```

si internamente colapsa ambos conceptos.

---

## 176.5 Composite mapping como sets

Incorrecto:

```php
$local = ['tenant_id', 'invoice_id'];
sort($local);

$remote = ['tenant_id', 'id'];
sort($remote);
```

Esto puede destruir la correspondencia.

---

## 176.6 Default action collapse

Incorrecto:

```php
$onDelete = $value ?? ReferentialAction::NO_ACTION;
```

si `null` significaba "no especificado / platform default".

---

## 176.7 NO ACTION = RESTRICT

Incorrecto:

```text
NO_ACTION
=
RESTRICT
```

como equivalencia universal.

---

## 176.8 Hidden index creation

Incorrecto:

```php
ForeignKeyDefinition::create()
{
    $this->createIndex();
}
```

---

## 176.9 Hidden DB lookup

Incorrecto:

```php
$fk->normalize();
```

si consulta la conexión actual para descubrir la referenced key.

---

## 176.10 Unknown = absent

Incorrecto:

```text
deferrability metadata unavailable
⇒ NOT DEFERRABLE
```

---

# 177. Ejemplo simple

```php
$foreignKey = new ForeignKeyDefinition(
    identifier: ForeignKeyIdentifier::from(
        'fk_orders_customer'
    ),

    localColumns: LocalColumnReferenceSet::of(
        ColumnIdentifier::from('customer_id'),
    ),

    referencedTable: ReferencedTableReference::local(
        TableIdentifier::from('customers'),
    ),

    referencedColumns: ReferencedColumnReferenceSet::of(
        ColumnIdentifier::from('id'),
    ),

    onUpdate: ReferentialAction::PLATFORM_DEFAULT,

    onDelete: ReferentialAction::RESTRICT,

    match: MatchSemantics::PLATFORM_DEFAULT,

    deferrability: DeferrabilityDefinition::platformDefault(),

    validationState: ForeignKeyValidationState::PLATFORM_DEFAULT,

    capabilities: SchemaCapabilityRequirementSet::of(
        SchemaCapability::FOREIGN_KEYS,
    ),

    metadata: ForeignKeyMetadata::declared(),

    extensions: ExtensionMetadataSet::empty(),
);
```

Representación:

```text
ForeignKeyDefinition(fk_orders_customer)
│
├── local
│   └── orders.customer_id
│
├── referenced
│   └── customers.id
│
├── match
│   └── PLATFORM_DEFAULT
│
├── onUpdate
│   └── PLATFORM_DEFAULT
│
├── onDelete
│   └── RESTRICT
│
└── deferrability
    └── PLATFORM_DEFAULT
```

---

# 178. Ejemplo composite

```text
ForeignKeyDefinition(fk_items_invoice)
│
├── local
│   ├── tenant_id
│   └── invoice_id
│
└── referenced
    └── invoices
        ├── tenant_id
        └── id
```

Mapping:

```text
tenant_id  ─────────► tenant_id
invoice_id ─────────► id
```

Formalmente:

```text
Mapping = [
    (tenant_id, tenant_id),
    (invoice_id, id)
]
```

---

# 179. Ejemplo self-reference

```text
employees
├── id
└── manager_id
       │
       └──────────────┐
                      ▼
                 employees.id
```

Representación:

```text
FK(
    employees.manager_id
    →
    employees.id
)
```

No requiere un modelo especial de ORM.

---

# 180. Ejemplo de ciclo

```text
users
 └── profile_id ─────────► profiles.id

profiles
 └── owner_user_id ──────► users.id
```

Dependency graph:

```text
users ─────► profiles
  ▲            │
  └────────────┘
```

Schema Planner podrá convertir:

```text
Create Users
Create Profiles
Add FK Users → Profiles
Add FK Profiles → Users
```

La estrategia es explícita.

---

# 181. Ejemplo de SET NULL

```text
posts.author_id
      │
      ▼
users.id

ON DELETE SET NULL
```

Requisito:

```text
posts.author_id
must permit NULL
```

si las reglas de la plataforma lo requieren.

---

# 182. Ejemplo de deferrable constraint

Definición:

```text
ForeignKeyDefinition
├── DEFERRABLE
└── INITIALLY_DEFERRED
```

Puede compilarse en PostgreSQL utilizando su representación correspondiente.

En una plataforma que no soporte esa capacidad:

```text
SchemaCapabilityException
```

No silent degradation.

---

# 183. Fórmula principal

```text
Database Foreign Key System
=
Typed Foreign Key Identity
+
Ordered Local Column References
+
Referenced Table Identity
+
Ordered Referenced Column References
+
Column Mapping
+
Referential Actions
+
Match Semantics
+
Deferrability
+
Constraint Validation State
+
Capability Requirements
+
Structural Dependencies
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

# 184. Correctness formula

Para una FK `F`:

```text
ValidForeignKey(F)
=
ValidIdentifier(F)
∧ NonEmptyLocalColumns(F)
∧ NonEmptyReferencedColumns(F)
∧ EqualArity(F)
∧ UniqueMapping(F)
∧ ValidLocalReferences(F)
∧ ValidReferencedReferences(F)
∧ CompatibleColumnTypes(F)
∧ EligibleReferencedKey(F)
∧ ValidMatchSemantics(F)
∧ ValidReferentialActions(F)
∧ ValidDeferrability(F)
∧ ValidExtensions(F)
```

---

# 185. Platform compilability

Para plataforma `P`:

```text
Compilable(F, P)
=
ValidForeignKey(F)
∧
Capabilities(P) ⊇ Requirements(F)
```

---

# 186. Referential mapping formula

Para:

```text
L = [l₁, l₂, ..., lₙ]
R = [r₁, r₂, ..., rₙ]
```

la FK define:

```text
M(F) = {
    l₁ → r₁,
    l₂ → r₂,
    ...
    lₙ → rₙ
}
```

con:

```text
|L| = |R| = n ≥ 1
```

---

# 187. Structural dependency formula

Si:

```text
FK(A → B)
```

entonces:

```text
Dependency(A, B)
```

deberá formar parte del dependency model, sujeto al tipo de operación.

Pero:

```text
Dependency(A,B)
```

no implica necesariamente:

```text
CREATE B
must always occur immediately before
CREATE A
```

porque el planner puede diferir la creación de la constraint.

---

# 188. Separation formula

```text
ForeignKeyDefinition
=
Referential Structural Constraint
```

```text
ORMRelationship
=
Object Association Mapping
```

```text
QueryJoin
=
Relational Query Operation
```

```text
IndexDefinition
=
Persistent Access Structure
```

Por tanto:

```text
ForeignKeyDefinition
≠
ORMRelationship
≠
QueryJoin
≠
IndexDefinition
```

---

# 189. Integración arquitectónica

```text
                    Schema Model
                         │
                         ▼
                ForeignKeyDefinition
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
 Schema Diff      Dependency Graph   Semantic Projection
       │                 │                  │
       ▼                 ▼                  ▼
 Change Set       Schema Planner      Query/ORM Consumers
       │                 │
       └────────┬────────┘
                ▼
        Schema Change Plan
                │
                ▼
         Schema Compiler
                │
                ▼
      CompiledDatabaseCommand
                │
                ▼
         Execution Engine
```

---

# 190. Integración con TableDefinition

La estructura queda:

```text
TableDefinition
├── ColumnDefinitionSet
├── PrimaryKeyDefinition?
├── UniqueConstraintSet
├── CheckConstraintSet
├── ForeignKeyDefinitionSet
├── IndexDefinitionSet
└── TableOptions
```

Una FK referencia columnas.

No las posee.

---

# 191. Integración con Index System

```text
ForeignKeyDefinition
        │
        │ may imply platform requirements
        ▼
Schema Planner
        │
        ├── suitable index exists
        │       ↓
        │     reuse
        │
        ├── platform auto-creates
        │       ↓
        │     record provenance
        │
        └── explicit index required
                ↓
          Index Change Operation
```

Esto evita:

```text
FK creation
=
hidden index mutation
```

---

# 192. Integración futura con ORM

Posteriormente:

```text
ForeignKeyDefinition
        │
        ▼
Entity Mapping Analyzer
        │
        ▼
Relationship Metadata
```

puede ayudar a inferir configuraciones.

Pero la arquitectura deberá mantener:

```text
Database Schema
     ↑
     │ consumes
ORM Mapping
```

sin invertir la dependencia.

---

# 193. Integración futura con Query Optimizer

Una FK validada puede proporcionar:

```text
ReferentialConstraintFact
```

al Semantic Query Engine.

Ejemplo:

```text
orders.customer_id NOT NULL
+
trusted FK orders.customer_id → customers.id
```

puede permitir derivar facts más fuertes que una FK nullable/unvalidated.

---

# 194. Evidence-based optimization

El optimizer deberá utilizar:

```text
ForeignKey Fact
+
Nullability Fact
+
Constraint Trust State
+
Query Context
```

No simplemente:

```text
foreign key exists
⇒ optimization legal
```

---

# 195. Final architectural rule

> **Una foreign key en VoltStack representa exclusivamente una regla estructural de integridad referencial. No representa relaciones de objetos, JOINs, índices ni comportamiento de persistencia ORM.**

La cadena conceptual será:

```text
ForeignKeyBlueprint
        │
        │ constructs
        ▼
ForeignKeyDefinition
        │
        │ belongs to
        ▼
TableDefinition
        │
        │ contributes to
        ▼
Schema Model
        │
        ├────────────► Dependency Graph
        │
        └────────────► Schema Diff
                         │
                         ▼
                  Schema Change Plan
                         │
                         ▼
                  Schema Compiler
                         │
                         ▼
                  Execution Engine
```

---

# 196. Resultado arquitectónico

Con `Database Foreign Key System`, VoltStack podrá representar desde:

```text
orders.customer_id
→
customers.id
```

hasta:

```text
Composite Foreign Key
├── multi-column mapping
├── cross-schema reference
├── MATCH semantics
├── ON UPDATE
├── ON DELETE
├── deferrability
├── validation/trust state
├── platform capabilities
└── extensions
```

manteniendo independientes:

```text
Referential Integrity
Indexing
ORM Relationships
Query Joins
Schema Evolution
Physical Execution
```

La separación final queda:

```text
                ┌────────────────────────┐
                │ ForeignKeyDefinition   │
                └───────────┬────────────┘
                            │
          ┌─────────────────┼───────────────────┐
          │                 │                   │
          ▼                 ▼                   ▼
   Schema Planner     Semantic Facts       ORM Metadata
          │                 │                   │
          ▼                 ▼                   ▼
   DDL Operations     Query Optimizer     Relationship System
```

Todos pueden consumir conocimiento de la foreign key.

Ninguno redefine lo que la foreign key es.

---

# 197. Siguiente documento

```text
95_DATABASE_CONSTRAINT_SYSTEM.md
```

El siguiente documento formalizará el modelo general de constraints:

```text
ConstraintDefinition
├── ConstraintIdentity
├── ConstraintKind
├── PrimaryKeyConstraint
├── UniqueConstraint
├── CheckConstraint
├── ForeignKeyConstraint integration
├── ConstraintExpression
├── Deferrability
├── ValidationState
├── EnforcementState
├── TrustState
├── CapabilityRequirements
├── Metadata
└── ExtensionConstraint
```

y establecerá especialmente:

```text
Constraint
≠
Index
≠
Validation Rule
≠
ORM Validation
≠
Authorization Policy
≠
Application Business Rule
```

Principio del siguiente documento:

> **Una database constraint es una regla declarativa de integridad estructural impuesta o representada por la base de datos; no una regla genérica de validación de la aplicación.**