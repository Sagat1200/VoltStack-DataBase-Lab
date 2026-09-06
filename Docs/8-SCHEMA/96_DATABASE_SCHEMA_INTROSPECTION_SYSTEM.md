# 96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Introspection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 96 — Database Schema Introspection System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Schema Introspection System` define la arquitectura mediante la cual VoltStack puede **observar una base de datos existente y convertir su estructura física en una representación estructural tipada, normalizada y explícita respecto de su nivel de certeza**.

El sistema deberá permitir inspeccionar:

```text
Database
Catalog
Namespace / Schema
Table
Column
Primary Key
Unique Constraint
Check Constraint
Foreign Key
Index
Sequence
View
Generated Column
Identity Strategy
Table Options
Platform Extensions
```

sin convertir metadata incompleta en conocimiento absoluto.

La regla fundamental será:

```text
Observed
≠
Declared

Unknown
≠
Absent

Partial
≠
Complete

Native Metadata
≠
Canonical Schema Model
```

---

# 2. Principio central

> **El Schema Introspection System observa y normaliza la estructura existente de una base de datos; nunca debe inventar certeza que el motor, driver o mecanismo de introspección no pudieron proporcionar.**

Formalmente:

```text
Physical Database
        │
        ▼
Native Metadata
        │
        ▼
Observation
        │
        ▼
Normalization
        │
        ▼
Observed Schema
```

No:

```text
Physical Database
        │
        ▼
Assumptions
        │
        ▼
"Perfect" Schema Model
```

---

# 3. Separación arquitectónica

Debe mantenerse:

```text
Schema Introspection
≠
Schema Builder
≠
Schema Model
≠
Schema Metadata
≠
Schema Diff
≠
Schema Migration
≠
Schema Compiler
≠
Query Execution
```

Cada componente tiene una responsabilidad diferente.

---

# 4. Responsabilidades

| Sistema | Responsabilidad |
|---|---|
| Schema Builder | construir estructura deseada |
| Schema Model | representar estructura |
| Schema Introspection | observar estructura existente |
| Schema Metadata | representar evidencia y contexto observado |
| Schema Diff | comparar estructuras |
| Migration | gobernar evolución |
| Schema Planner | ordenar cambios |
| Schema Compiler | generar representación DDL |
| Execution Engine | ejecutar |

---

# 5. Pipeline general

```text
Database Server
      │
      ▼
Connection
      │
      ▼
Platform Schema Introspector
      │
      ▼
Native Metadata Reader
      │
      ▼
Raw Observation
      │
      ▼
Metadata Decoder
      │
      ▼
Structural Normalizer
      │
      ▼
Reference Resolver
      │
      ▼
Capability Analyzer
      │
      ▼
Observed Schema
      │
      ▼
Schema Snapshot
```

---

# 6. Posición dentro de Quantum Database

```text
VoltStack/Quantum/Database
│
├── Driver
├── Connection
├── Platform
│
├── Query
│
├── Execution
│
└── Schema
    ├── Model
    ├── Ast
    ├── Builder
    ├── Definition
    ├── Index
    ├── ForeignKey
    ├── Constraint
    │
    └── Introspection
            │
            ├── Platform
            ├── Reader
            ├── Observation
            ├── Normalization
            ├── Resolution
            ├── Coverage
            ├── Diagnostic
            └── Snapshot
```

---

# 7. Objetivos

El sistema deberá proporcionar:

- introspección tipada;
- soporte MySQL;
- soporte MariaDB;
- soporte PostgreSQL;
- soporte SQLite;
- observación completa;
- observación parcial;
- observación scoped;
- metadata nativa;
- normalización;
- provenance;
- certainty;
- coverage;
- reference resolution;
- capability-aware introspection;
- preservation de información desconocida;
- snapshots inmutables;
- resultados deterministas;
- diagnósticos estructurados;
- extensibilidad;
- seguridad;
- límites de recursos;
- compatibilidad con persistent runtimes.

---

# 8. No objetivos

El sistema no deberá:

- modificar el schema;
- ejecutar migraciones;
- generar DDL;
- inferir cambios;
- autorizar operaciones;
- reparar automáticamente inconsistencias;
- asumir features no observadas;
- convertir `UNKNOWN` en `ABSENT`;
- administrar transacciones de negocio;
- mantener un schema global mutable;
- depender del ORM.

---

# 9. Modelo conceptual

Se propone:

```text
SchemaIntrospectionRequest
          │
          ▼
SchemaIntrospector
          │
          ▼
SchemaObservation
          │
          ├── DatabaseObservation
          ├── NamespaceObservation
          ├── TableObservation
          ├── ColumnObservation
          ├── IndexObservation
          ├── ConstraintObservation
          ├── ForeignKeyObservation
          ├── SequenceObservation
          └── ViewObservation
          │
          ▼
SchemaObservationNormalizer
          │
          ▼
ObservedSchema
          │
          ▼
SchemaSnapshot
```

---

# 10. SchemaIntrospector

Contrato conceptual:

```php
interface SchemaIntrospector
{
    public function introspect(
        SchemaIntrospectionRequest $request,
        SchemaIntrospectionContext $context,
    ): SchemaIntrospectionResult;
}
```

---

# 11. Introspector no es global

No deberá existir:

```php
Schema::currentIntrospector();
```

dependiente de estado global mutable.

El introspector deberá resolverse explícitamente mediante:

```text
Connection
+
Platform
+
CapabilitySnapshot
+
IntrospectionContext
```

---

# 12. PlatformSchemaIntrospector

Cada plataforma tendrá implementación propia.

```text
PlatformSchemaIntrospector
│
├── MySqlSchemaIntrospector
├── MariaDbSchemaIntrospector
├── PostgreSqlSchemaIntrospector
└── SqliteSchemaIntrospector
```

MariaDB no será alias automático de MySQL.

---

# 13. Platform selection

La resolución deberá ser capability/platform-driven:

```text
Connection
    │
    ▼
DatabasePlatform
    │
    ▼
SchemaIntrospectorResolver
    │
    ▼
PlatformSchemaIntrospector
```

---

# 14. SchemaIntrospectionRequest

Debe representar explícitamente qué se desea observar.

Ejemplo:

```php
final readonly class SchemaIntrospectionRequest
{
    public function __construct(
        public SchemaIntrospectionScope $scope,
        public SchemaObjectSelection $selection,
        public SchemaIntrospectionDepth $depth,
        public SchemaIntrospectionOptions $options,
    ) {}
}
```

---

# 15. Scope

Se propone:

```text
SchemaIntrospectionScope
├── DATABASE
├── CATALOG
├── NAMESPACE
├── TABLE
├── OBJECT_SET
└── CUSTOM
```

---

# 16. Full introspection

Ejemplo:

```text
DATABASE
```

puede solicitar:

```text
all visible namespaces
all visible tables
all columns
all indexes
all constraints
all foreign keys
all sequences
all views
```

según capabilities y permisos.

---

# 17. Scoped introspection

También debe poder solicitarse:

```text
only table users
```

o:

```text
tables:
users
roles
permissions
```

Esto es crítico para rendimiento.

---

# 18. Scope ≠ completeness

Una introspección scoped puede ser:

```text
COMPLETE
```

dentro de su scope.

Ejemplo:

```text
Scope:
table users

Coverage:
complete for table users
```

pero no para toda la base de datos.

---

# 19. Coverage model

Debe existir un modelo explícito:

```text
SchemaCoverage
├── COMPLETE
├── PARTIAL
└── UNKNOWN
```

pero preferiblemente asociado a un scope.

---

# 20. Scoped coverage

Se propone:

```php
final readonly class SchemaCoverageDescriptor
{
    public function __construct(
        public SchemaIntrospectionScope $scope,
        public SchemaCoverage $coverage,
        public SchemaObjectCoverageSet $objects,
    ) {}
}
```

---

# 21. Critical rule

Debe cumplirse:

```text
Object not present
+
COMPLETE coverage for relevant scope
=
Object absent
```

pero:

```text
Object not present
+
PARTIAL coverage
≠
Object absent
```

---

# 22. Unknown ≠ absent

Ejemplo:

```text
Foreign key not returned
```

puede significar:

```text
ABSENT
```

o:

```text
NOT VISIBLE
```

o:

```text
NOT INTROSPECTED
```

o:

```text
UNSUPPORTED
```

o:

```text
UNKNOWN
```

No deberán colapsarse.

---

# 23. Presence state

Se propone:

```text
SchemaObjectPresence
├── PRESENT
├── ABSENT
├── UNKNOWN
├── NOT_VISIBLE
├── NOT_REQUESTED
└── UNSUPPORTED
```

---

# 24. Observation

Una observación deberá contener:

```text
value
+
evidence
+
certainty
+
provenance
+
coverage
```

---

# 25. ObservedValue

Conceptualmente:

```php
final readonly class ObservedValue
{
    public function __construct(
        public mixed $value,
        public ObservationCertainty $certainty,
        public ObservationProvenance $provenance,
    ) {}
}
```

En implementación real deberá preferirse un tipo genérico/documentado.

---

# 26. Observation certainty

Se propone:

```text
ObservationCertainty
├── EXACT
├── NORMALIZED
├── INFERRED
├── APPROXIMATE
├── UNKNOWN
└── UNAVAILABLE
```

---

# 27. Exact

`EXACT` significa que la plataforma proporcionó evidencia suficientemente directa.

No necesariamente significa:

```text
portable semantic representation
```

---

# 28. Normalized

Ejemplo:

```text
Native type:
varchar(255)
```

normalizado a:

```text
StringType(length=255)
```

puede marcarse:

```text
NORMALIZED
```

---

# 29. Inferred

Ejemplo:

```text
platform metadata
+
known platform rule
        ↓
inferred generation strategy
```

deberá conservar:

```text
certainty = INFERRED
```

---

# 30. Observation provenance

Se propone:

```text
ObservationProvenance
├── INFORMATION_SCHEMA
├── SYSTEM_CATALOG
├── PRAGMA
├── DRIVER_METADATA
├── NATIVE_QUERY
├── PLATFORM_INFERENCE
├── EXTENSION
└── UNKNOWN
```

---

# 31. Native metadata

VoltStack deberá conservar metadata nativa cuando tenga valor.

Ejemplo:

```php
final readonly class NativeSchemaMetadata
{
    public function __construct(
        public DatabasePlatformId $platform,
        public array $attributes,
    ) {}
}
```

No deberá exponerse como array arbitrario sin control en la API principal.

---

# 32. Native metadata classification

Se deberá distinguir:

```text
STRUCTURAL
SEMANTIC
PHYSICAL
INFORMATIONAL
UNKNOWN
```

---

# 33. Canonical model

El resultado final deberá mapearse a los objetos canónicos definidos anteriormente:

```text
DatabaseSchema
Table
Column
IndexDefinition
ConstraintDefinition
ForeignKeyDefinition
Sequence
View
```

sin perder metadata observacional.

---

# 34. Observed Schema ≠ DatabaseSchema

Se recomienda distinguir:

```text
ObservedSchema
=
DatabaseSchema
+
ObservationMetadata
+
Coverage
+
Diagnostics
+
Provenance
```

---

# 35. SchemaSnapshot

Un snapshot será una captura inmutable del conocimiento observado en un punto lógico.

```php
final readonly class SchemaSnapshot
{
    public function __construct(
        public SchemaSnapshotId $id,
        public DatabaseSchema $schema,
        public SchemaCoverageDescriptor $coverage,
        public SchemaObservationMetadata $metadata,
        public SchemaDiagnosticSet $diagnostics,
    ) {}
}
```

---

# 36. Snapshot immutability

Una vez creado:

```text
SchemaSnapshot(t0)
```

no deberá transformarse silenciosamente en:

```text
SchemaSnapshot(t1)
```

---

# 37. Refresh

Para actualizar:

```text
Snapshot A
    │
    ▼
New Introspection
    │
    ▼
Snapshot B
```

No:

```text
mutate Snapshot A
```

---

# 38. Snapshot identity

Debe existir:

```text
SchemaSnapshotId
```

pero:

```text
SchemaSnapshotId
≠
SchemaFingerprint
```

---

# 39. Snapshot timestamp

Puede incluir:

```text
observedAt
```

pero el timestamp por sí mismo no prueba consistencia transaccional.

---

# 40. Snapshot consistency

Se propone:

```text
SchemaSnapshotConsistency
├── TRANSACTIONALLY_CONSISTENT
├── STATEMENT_CONSISTENT
├── BEST_EFFORT
├── POTENTIALLY_MIXED
└── UNKNOWN
```

---

# 41. Why consistency matters

La introspección puede requerir múltiples consultas:

```text
query tables
query columns
query indexes
query constraints
```

y el schema podría cambiar entre ellas.

Por tanto:

```text
snapshot
```

no siempre significa:

```text
single atomic database instant
```

---

# 42. Introspection session

Se propone:

```php
interface SchemaIntrospectionSession
{
    public function connection(): Connection;

    public function platform(): DatabasePlatform;

    public function capabilities(): PlatformCapabilitySnapshot;

    public function request(): SchemaIntrospectionRequest;
}
```

Será operation-scoped.

---

# 43. Connection ownership

El Introspection System podrá utilizar una conexión suministrada.

Pero deberá definirse claramente:

```text
Borrowed Connection
vs
Owned Connection
```

---

# 44. Preferred model

Preferiblemente:

```text
IntrospectionSession
borrows Connection
```

y no la cierra.

El propietario superior controla lifecycle.

---

# 45. No hidden connections

Nunca:

```text
SchemaIntrospector
    ↓
secretly creates connection
```

La dependencia debe ser explícita.

---

# 46. Introspection query execution

El sistema podrá utilizar consultas nativas especializadas.

No necesita forzar el Query Builder de alto nivel.

---

# 47. Avoid circular dependency

No deberá existir:

```text
Schema Introspection
    ↓
High-level Query Builder
    ↓
Schema-aware Semantic Analyzer
    ↓
Schema Introspection
```

---

# 48. Native metadata reader

Se propone:

```text
PlatformSchemaIntrospector
        │
        ▼
NativeSchemaMetadataReader
```

capaz de utilizar:

```text
information_schema
system catalogs
PRAGMA
driver metadata
platform-native commands
```

---

# 49. Native queries are allowed internally

El uso de SQL nativo dentro de un adapter de introspección es legítimo.

Pero:

```text
Native SQL
```

deberá estar encapsulado dentro del adapter de plataforma.

---

# 50. MySQL introspection

El adapter MySQL podrá observar mediante mecanismos apropiados:

```text
schemas
tables
columns
indexes
constraints
foreign keys
generated columns
table options
views
```

sin propagar directamente detalles nativos al modelo portable.

---

# 51. MariaDB introspection

MariaDB tendrá:

```text
MariaDbSchemaIntrospector
```

propio.

Podrá compartir componentes con MySQL donde las capacidades realmente coincidan.

No por herencia conceptual obligatoria.

---

# 52. PostgreSQL introspection

Podrá requerir combinación de:

```text
information_schema
+
pg_catalog
```

para obtener metadata completa.

La capa platform adapter decide la estrategia.

---

# 53. SQLite introspection

Podrá utilizar:

```text
PRAGMA
sqlite_schema
```

y otros mecanismos propios.

Debe conservarse cualquier limitación de observabilidad.

---

# 54. Table observation

Se propone:

```php
final readonly class TableObservation
{
    public function __construct(
        public QualifiedTableIdentifier $identifier,
        public TableDefinition $definition,
        public TableObservationMetadata $metadata,
        public ObservationCompleteness $completeness,
    ) {}
}
```

---

# 55. Table definition reuse

Tal como se definió en el documento 91:

```text
Table(Model)
=
TableDefinition
+
Structural Identity
+
Observed Metadata
+
Provenance/Coverage/Resolution
```

Introspection podrá reconstruir el componente estructural:

```text
TableDefinition
```

sin afirmar que fue originalmente declarado así por el desarrollador.

---

# 56. Column introspection

Para cada columna deberán observarse cuando sea posible:

```text
name
logical/native type
length
precision
scale
nullability
default
generation
identity
collation
charset
comment
ordinal
platform options
```

---

# 57. Native type ≠ logical type

Ejemplo:

```text
Native:
BIGINT UNSIGNED
```

puede normalizarse hacia:

```text
IntegerType(
    width = 64,
    unsigned = true
)
```

si el Type System soporta esa semántica.

Debe conservarse native metadata cuando sea relevante.

---

# 58. Unknown type

Un tipo desconocido no deberá convertirse arbitrariamente a:

```text
string
```

Se requiere:

```text
UnknownDatabaseType
```

o:

```text
OpaquePlatformType
```

según política.

---

# 59. Type normalization

Pipeline:

```text
Native Type Descriptor
        │
        ▼
Platform Type Decoder
        │
        ▼
DatabaseType
        │
        ├── portable semantics
        └── native metadata
```

---

# 60. Column ordinal

El orden de columnas deberá preservarse.

```text
ordinal = structural observation
```

No ordenar columnas alfabéticamente.

---

# 61. Default introspection

Debe distinguirse:

```text
No Default
Default NULL
Literal Default
Expression Default
Generated Default
Unknown Default
```

---

# 62. Critical default invariant

Nunca:

```text
metadata default = NULL
```

deberá interpretarse automáticamente como:

```text
DEFAULT NULL
```

si la API nativa utiliza NULL para representar ausencia de metadata.

---

# 63. Default parser

Se propone:

```text
PlatformDefaultExpressionDecoder
```

responsable de transformar metadata nativa en:

```text
ColumnDefaultDefinition
```

---

# 64. Raw default preservation

Cuando no pueda analizarse una expresión:

```text
RawPlatformDefaultExpression
```

podrá preservarse con:

```text
platform
raw representation
certainty
trust
portability
```

---

# 65. Generated columns

Debe distinguirse:

```text
GeneratedColumn
≠
DefaultValue
```

---

# 66. Generation introspection

Se podrá observar:

```text
identity
sequence
auto increment
generated expression
stored generated
virtual generated
platform-generated
unknown
```

---

# 67. Abstract generation

La normalización deberá producir conceptos abstractos cuando sea seguro.

No:

```text
AUTO_INCREMENT
```

como modelo universal.

---

# 68. Index introspection

Debe reconstruir el modelo del documento 93:

```text
IndexDefinition
├── keys
├── uniqueness
├── method
├── predicate
├── included columns
├── ordering
├── options
└── metadata
```

---

# 69. Index visibility

Algunas plataformas pueden exponer:

```text
visible
invisible
disabled
unknown
```

Esto deberá preservarse mediante metadata/capability apropiada.

---

# 70. Expression indexes

Si la plataforma soporta expression indexes:

```text
INDEX(lower(email))
```

la introspección deberá intentar reconstruir:

```text
IndexExpressionKey
```

---

# 71. Failed expression parsing

Si no puede analizarse:

```text
RawIndexExpression
```

con provenance y diagnostics.

Nunca perder silenciosamente el índice.

---

# 72. Partial indexes

Ejemplo:

```text
WHERE deleted_at IS NULL
```

deberá mapearse a:

```text
IndexPredicate
```

si la plataforma lo permite.

---

# 73. Constraint introspection

Debe reconstruir:

```text
PrimaryKeyConstraintDefinition
UniqueConstraintDefinition
CheckConstraintDefinition
ForeignKeyDefinition
```

según los documentos 94 y 95.

---

# 74. Constraint vs index introspection

No deberá inferirse universalmente:

```text
unique index
=
unique constraint
```

---

# 75. Platform relationship metadata

Si una plataforma informa que un constraint está respaldado por un índice:

```text
ConstraintPhysicalSupportMetadata
```

puede preservar esa relación.

---

# 76. Primary key introspection

Debe conservar:

```text
name
ordered columns
platform metadata
supporting index relationship
certainty
```

---

# 77. Check introspection

Debe intentar reconstruir:

```text
SchemaExpression
```

desde la expresión nativa.

---

# 78. Expression parser

Cada plataforma podrá proporcionar:

```text
PlatformSchemaExpressionParser
```

---

# 79. Parsing failure

Debe producir:

```text
RawSchemaExpression
+
diagnostic
```

si la política permite preservación.

No:

```text
drop check constraint from model
```

---

# 80. Foreign key introspection

Debe reconstruir:

```text
local table
local ordered columns
referenced table
referenced ordered columns
ON UPDATE
ON DELETE
match semantics
deferrability
validation
name
```

cuando esté disponible.

---

# 81. Reference resolution

Inicialmente:

```text
ForeignKeyReference
```

puede ser:

```text
UNRESOLVED
```

Posteriormente:

```text
SchemaReferenceResolver
```

intentará resolverla contra el snapshot.

---

# 82. Resolution states

Se propone:

```text
ReferenceResolutionState
├── RESOLVED
├── EXTERNAL
├── UNRESOLVED
├── AMBIGUOUS
├── NOT_VISIBLE
└── UNKNOWN
```

---

# 83. External reference

Una FK puede apuntar a un objeto fuera del scope introspectado.

Eso no significa:

```text
invalid foreign key
```

---

# 84. Partial snapshot references

Ejemplo:

```text
scope:
orders table only
```

y:

```text
orders.customer_id → customers.id
```

`customers` puede no estar en el snapshot.

La referencia será:

```text
EXTERNAL_TO_SNAPSHOT
```

o equivalente.

---

# 85. Sequence introspection

Cuando la plataforma soporte sequences:

```text
SequenceDefinition
```

podrá contener:

```text
name
start
increment
min
max
cycle
cache
ownership
metadata
```

según disponibilidad.

---

# 86. Sequence absence

En plataformas sin sequences:

```text
UNSUPPORTED
```

no:

```text
ABSENT
```

como semántica global.

---

# 87. View introspection

Debe distinguir:

```text
ViewDefinition
MaterializedViewDefinition
OpaqueViewDefinition
```

según capacidades futuras.

---

# 88. View SQL

El SQL de una view puede ser difícil de convertir al Query AST portable.

Por tanto podrá preservarse inicialmente como:

```text
NativeViewDefinition
```

con platform scope.

---

# 89. View definition certainty

Debe distinguirse:

```text
parsed
normalized
raw
unavailable
```

---

# 90. Table options

Pueden incluir:

```text
engine
charset
collation
tablespace
storage
row format
strictness
platform flags
```

pero deberán clasificarse.

---

# 91. Portable vs platform options

```text
TableOption
├── PortableTableOption
└── PlatformTableOption
```

---

# 92. No false portability

Ejemplo:

```text
ENGINE=InnoDB
```

no deberá transformarse en una opción genérica falsa.

Debe conservarse como:

```text
MySqlTableEngineOption
```

---

# 93. Identifier normalization

Debe respetarse:

```text
Declared Identifier
Observed Identifier
Normalized Identifier
Rendered Identifier
```

como conceptos distintos.

---

# 94. Case sensitivity

Nunca:

```php
strtolower($identifier);
```

como normalización universal.

---

# 95. Identifier semantics

La normalización dependerá de:

```text
platform
quoting rules
case folding
catalog semantics
```

---

# 96. Identifier canonicalizer

Se propone:

```text
PlatformIdentifierCanonicalizer
```

con comportamiento capability-driven.

---

# 97. Catalog and namespace

No todas las plataformas tienen exactamente:

```text
catalog.schema.table
```

VoltStack no deberá inventar niveles inexistentes.

---

# 98. Qualified object path

Utilizar:

```text
SchemaObjectPath
```

estructurado.

Ejemplo:

```text
Catalog?
Namespace?
Object
```

---

# 99. Object identity

Debe distinguirse:

```text
SchemaObjectId
SchemaObjectPath
SchemaObjectFingerprint
```

---

# 100. Introspected object ID

Un `SchemaObjectId` generado durante introspection:

```text
snapshot-A:table-42
```

no deberá asumirse estable entre snapshots.

---

# 101. Structural identity

Para comparar snapshots se utilizarán:

```text
paths
normalized identifiers
structural fingerprints
semantic comparison
```

según perfil.

---

# 102. Metadata normalization

Se propone pipeline:

```text
Native Metadata
      │
      ▼
Decode
      │
      ▼
Typed Native Metadata
      │
      ▼
Normalize
      │
      ▼
Canonical Structural Metadata
      │
      ▼
Attach Provenance
      │
      ▼
Observed Schema
```

---

# 103. Normalization invariants

Debe ser:

```text
deterministic
idempotent where applicable
side-effect free
capability-aware
provenance-preserving
```

---

# 104. No hidden introspection during normalization

Nunca:

```text
normalize(table)
    ↓
query database again
```

La fase de observación deberá reunir explícitamente la información requerida.

---

# 105. Observation vs normalization

```text
Observation
=
database interaction
```

```text
Normalization
=
pure structural transformation
```

Esta separación es crítica.

---

# 106. Introspection planning

Para evitar cientos de round trips podrá existir:

```text
SchemaIntrospectionPlanner
```

---

# 107. Introspection plan

Ejemplo:

```text
Request
   │
   ▼
Introspection Plan
├── Fetch namespaces
├── Fetch tables
├── Fetch columns
├── Fetch constraints
├── Fetch indexes
└── Resolve references
```

---

# 108. Planner goals

Podrá optimizar:

```text
round trips
batching
catalog queries
metadata reuse
scope
dependency requirements
```

sin alterar el significado del request.

---

# 109. IntrospectionPlan ≠ QueryPlan

Debe mantenerse:

```text
SchemaIntrospectionPlan
≠
Query Logical Plan
≠
Query Physical Plan
```

---

# 110. Capability-aware planning

Si una plataforma permite obtener:

```text
tables + columns + constraints
```

en una sola consulta eficiente, podrá hacerlo.

Otra plataforma podrá requerir varias.

---

# 111. Snapshot consistency planning

El planner también podrá decidir:

```text
transactional snapshot
best effort
serialized reads
single connection
multiple reads
```

según capabilities y policy.

---

# 112. Transaction usage

Introspection podrá ejecutarse dentro de una transacción cuando sea apropiado.

Pero:

```text
SchemaIntrospector
```

no deberá asumir que toda plataforma proporciona transactional DDL/catalog visibility equivalente.

---

# 113. Transaction ownership

Si una transacción ya existe:

```text
Borrowed Transaction Context
```

deberá respetarse.

No hacer:

```text
BEGIN
...
COMMIT
```

ocultamente sobre una transacción ajena.

---

# 114. Error model

Se propone:

```text
DatabaseSchemaIntrospectionException
├── SchemaIntrospectionConnectionException
├── SchemaIntrospectionPermissionException
├── SchemaIntrospectionQueryException
├── SchemaIntrospectionDecodeException
├── SchemaIntrospectionNormalizationException
├── SchemaIntrospectionReferenceException
├── SchemaIntrospectionCapabilityException
├── SchemaIntrospectionUnsupportedFeatureException
├── SchemaIntrospectionBudgetExceededException
├── SchemaIntrospectionCancelledException
├── SchemaIntrospectionTimeoutException
├── SchemaIntrospectionConsistencyException
├── SchemaIntrospectionExtensionException
└── SchemaIntrospectionInvariantException
```

---

# 115. Failure ≠ empty schema

Regla crítica:

```text
Introspection failed
≠
Database has no objects
```

---

# 116. Permission denied ≠ absent

Si el usuario no puede ver una tabla:

```text
NOT_VISIBLE
```

o diagnóstico correspondiente.

Nunca:

```text
ABSENT
```

sin evidencia.

---

# 117. Partial failure

Puede ocurrir:

```text
tables        OK
columns       OK
indexes       FAILED
constraints   OK
```

La política deberá determinar si:

```text
FAIL_ALL
RETURN_PARTIAL
RETURN_PARTIAL_WITH_DIAGNOSTICS
```

---

# 118. Default safety policy

Para operaciones que posteriormente puedan producir cambios destructivos:

```text
partial introspection
```

deberá ser tratada conservadoramente.

---

# 119. Destructive diff safety

Debe cumplirse:

```text
PartialSnapshot
+
MissingObject
≠
SafeDropCandidate
```

---

# 120. Cancellation

Introspection puede ser costosa.

Deberá soportar:

```text
CancellationToken
Deadline
TimeoutPolicy
```

alineado con el Execution Engine.

---

# 121. Timeout semantics

Debe distinguirse:

```text
timeout
cancellation
connection failure
permission failure
decode failure
```

---

# 122. Partial result after cancellation

No deberá presentarse como:

```text
complete snapshot
```

---

# 123. Resource budgets

Se propone:

```php
final readonly class SchemaIntrospectionBudget
{
    public function __construct(
        public int $maxObjects,
        public int $maxTables,
        public int $maxColumns,
        public int $maxIndexes,
        public int $maxConstraints,
        public int $maxForeignKeys,
        public int $maxExpressionNodes,
        public int $maxMetadataBytes,
        public int $maxNativeRows,
        public int $maxRoundTrips,
    ) {}
}
```

---

# 124. Budget exhaustion

Debe producir:

```text
SchemaIntrospectionBudgetExceededException
```

o partial result explícitamente marcado según policy.

Nunca truncamiento silencioso.

---

# 125. Streaming metadata

Para schemas grandes podrá existir:

```text
SchemaObservationStream
```

internamente.

---

# 126. Streaming ≠ final mutable schema

Las observaciones pueden procesarse incrementalmente.

Pero el snapshot publicado deberá ser:

```text
immutable
sealed
```

---

# 127. Incremental assembly

```text
Observation Stream
      │
      ▼
Mutable Introspection Assembly
      │
      ▼
Seal
      │
      ▼
Immutable SchemaSnapshot
```

---

# 128. Assembly lifecycle

```text
CREATED
  ↓
OBSERVING
  ↓
NORMALIZING
  ↓
RESOLVING
  ↓
VALIDATING
  ↓
SEALED
```

---

# 129. No mutation after seal

Una vez:

```text
SEALED
```

cualquier modificación deberá fallar.

---

# 130. Determinism

Para el mismo conjunto de metadata observada:

```text
Normalize(M)
=
same canonical structural result
```

independientemente del orden en que el driver entregue filas, salvo donde el orden sea semántico.

---

# 131. Stable ordering

Colecciones no ordenadas semánticamente podrán canonicalizarse.

Pero:

```text
column order
index key order
constraint column order
FK column pair order
```

deberán preservarse.

---

# 132. Fingerprinting

El snapshot podrá tener:

```text
SchemaFingerprint
```

derivado del modelo normalizado.

---

# 133. Fingerprint profiles

```text
EXACT
STRUCTURAL
SEMANTIC
PORTABLE
PLATFORM_NORMALIZED
```

---

# 134. Observation metadata and fingerprint

Metadata como:

```text
observedAt
connection ID
query duration
```

no deberá afectar el fingerprint estructural.

---

# 135. Native metadata fingerprint

Un perfil `EXACT` podrá incluir metadata nativa estructural relevante.

Pero deberá estar versionado.

---

# 136. Snapshot cache

La introspección podrá integrarse posteriormente con Cache.

Pero:

```text
Schema Introspection
```

no dependerá obligatoriamente del Cache System.

---

# 137. Cached snapshot

Debe conservar:

```text
source
createdAt
coverage
platform identity
capability fingerprint
normalization version
```

---

# 138. Cache key

Conceptualmente:

```text
SchemaSnapshotCacheKey
=
ConnectionIdentity
+
Scope
+
PlatformIdentity
+
CapabilityFingerprint
+
IntrospectionProfile
+
NormalizationVersion
```

sin incluir secretos.

---

# 139. Staleness

Debe distinguirse:

```text
ValidSnapshot
StaleSnapshot
ExpiredSnapshot
UnknownFreshness
```

cuando cache integration esté activa.

---

# 140. Cached ≠ current

Nunca:

```text
cached snapshot
=
current database state
```

como verdad universal.

---

# 141. Security

Schema metadata puede contener información sensible:

```text
table names
column names
internal schemas
comments
generated expressions
view definitions
```

---

# 142. Credential protection

Nunca incluir:

```text
password
DSN with password
access token
secret
```

en:

```text
SchemaSnapshot
diagnostics
telemetry
cache key
exception message
```

---

# 143. Permission boundaries

Introspection deberá respetar los permisos de la conexión.

No intentará elevar privilegios.

---

# 144. SQL injection protection

Los object filters deberán utilizar:

```text
typed identifiers
bound parameters where supported
platform-safe quoting
```

No concatenación arbitraria.

---

# 145. User-supplied object names

Un request:

```php
Schema::inspectTable($userInput);
```

deberá pasar por:

```text
Identifier validation
+
Platform-safe metadata query construction
```

---

# 146. Telemetry

Podrán registrarse:

```text
introspection duration
objects observed
round trips
normalization duration
resolution duration
diagnostic count
coverage
platform
cache hit/miss
```

---

# 147. Telemetry privacy

Por defecto no deberá registrar:

```text
full schema
column contents
raw credentials
full view definitions
raw check expressions
```

---

# 148. Observability boundaries

El Introspection System observa:

```text
schema metadata
```

No:

```text
user row data
```

salvo que una feature futura lo declare explícitamente fuera de este subsistema.

---

# 149. Extension model

Se propone:

```text
SchemaIntrospectionExtension
```

para:

- custom object types;
- vendor extensions;
- custom type decoders;
- metadata enrichers;
- normalization adapters.

---

# 150. Frozen extension registry

```text
FrozenSchemaIntrospectionExtensionRegistry
```

deberá cerrarse después de bootstrap.

---

# 151. Extension restrictions

Una extensión no podrá:

- cambiar coverage silenciosamente;
- convertir unknown en absent;
- ocultar diagnostics;
- acceder a otro tenant implícitamente;
- almacenar mutable request state global;
- introducir secretos en metadata.

---

# 152. Custom object observation

Una extensión podrá registrar:

```text
ExtensionSchemaObjectObservation
```

con:

```text
extension ID
object type ID
payload version
capabilities
provenance
certainty
```

---

# 153. Unknown extension object

La política será:

```text
FAIL
PRESERVE_OPAQUE
IGNORE_WITH_DIAGNOSTIC
```

---

# 154. ORM integration

ORM podrá consumir:

```text
SchemaSnapshot
```

para:

- diagnostics;
- mapping validation;
- development tooling;
- schema verification.

Pero ORM no controlará introspection.

---

# 155. Query Semantic Engine integration

Podrá consumir facts derivados del schema:

```text
column types
nullability
keys
trusted constraints
foreign keys
```

---

# 156. Snapshot requirement

El Semantic Engine deberá recibir explícitamente:

```text
SchemaSnapshot
```

o una proyección inmutable equivalente.

No deberá ejecutar introspection ocultamente durante query analysis.

---

# 157. Query optimizer rule

Nunca:

```text
Optimizer
    ↓
database introspection
```

durante una optimización normal.

Eso introduciría:

```text
hidden I/O
nondeterminism
latency
state coupling
```

---

# 158. Schema Diff integration

Uso principal:

```text
Current Schema Snapshot
          │
          ▼
       Schema Diff
          ▲
          │
Target Schema Definition
```

---

# 159. Diff safety requirement

Antes de considerar un objeto ausente:

```text
CoverageRelevantTo(object)
=
COMPLETE
```

deberá ser demostrable.

---

# 160. Example

Target:

```text
users
orders
```

Observed partial snapshot:

```text
users
```

No puede concluirse:

```text
orders must be created
```

si el scope no incluía `orders`.

---

# 161. Migration integration

Migration tooling podrá usar introspection para:

```text
preconditions
drift detection
schema diff
verification
post-migration validation
```

pero el Migration System permanece separado.

---

# 162. Drift detection

Conceptualmente:

```text
Expected Schema
       │
       ▼
     Compare
       ▲
       │
Observed Snapshot
       │
       ▼
Schema Drift Report
```

---

# 163. Drift ≠ migration

Detectar:

```text
unexpected index
missing constraint
changed column type
```

no significa automáticamente aplicar cambios.

---

# 164. Public API

Podrá existir una API cómoda:

```php
$snapshot = Schema::inspect();

$table = Schema::inspectTable('users');
```

pero internamente:

```text
Facade
  ↓
SchemaInspectionManager
  ↓
SchemaIntrospector
```

---

# 165. Builder separation

`Schema::create()` y `Schema::inspect()` pueden coexistir en una facade.

Pero:

```text
SchemaBuilder
≠
SchemaIntrospector
```

---

# 166. hasTable()

Una API:

```php
Schema::hasTable('users');
```

deberá delegar al sistema de introspección/metadata.

No al Builder.

---

# 167. hasColumn()

Igualmente:

```php
Schema::hasColumn('users', 'email');
```

deberá usar:

```text
Schema Inspection API
```

---

# 168. Existence result

Preferible internamente:

```text
SchemaObjectExistenceResult
```

con:

```text
PRESENT
ABSENT
UNKNOWN
NOT_VISIBLE
```

La facade booleana deberá tener política clara.

---

# 169. Boolean convenience danger

Una API booleana puede ocultar:

```text
UNKNOWN
```

Por ello se recomienda:

```php
Schema::inspectTablePresence('users');
```

para tooling avanzado.

---

# 170. Persistent runtime safety

Shared immutable:

```text
Platform metadata decoders
Normalization rules
Frozen registries
Capability definitions
```

Operation-scoped mutable:

```text
IntrospectionSession
ObservationBuffer
Assembly
ReferenceResolverSession
Diagnostics
```

---

# 171. FrankenPHP

Bajo FrankenPHP:

```text
Request A
    ↓
IntrospectionSession A
    ↓
disposed

Request B
    ↓
IntrospectionSession B
```

No compartir mutable state.

---

# 172. RoadRunner/OpenSwoole

La misma regla deberá aplicarse.

Especialmente:

```text
coroutine-local
request-local
operation-local
```

no debe confundirse con:

```text
process-global
```

---

# 173. Multitenancy

Multitenancy será integración opcional.

Puede proporcionar:

```text
TenantConnectionResolver
TenantSchemaScopeResolver
```

antes de introspection.

---

# 174. Tenant isolation

Debe cumplirse:

```text
Tenant A introspection
≠
Tenant B introspection
```

en:

```text
connection
scope
snapshot cache
diagnostics
telemetry context
```

---

# 175. No tenant in core global state

Nunca:

```php
SchemaIntrospector::$tenantId;
```

---

# 176. Testing architecture

Las pruebas se dividirán en:

```text
Unit
Normalization
Parser
Reference Resolution
Platform Adapter
Integration
Conformance
Failure
Performance
Persistent Runtime
Security
```

---

# 177. Metadata fixture tests

Cada plataforma deberá tener fixtures de metadata nativa.

Ejemplo:

```text
MySQL metadata fixture
    ↓
Normalizer
    ↓
Expected TableDefinition
```

sin necesitar DB real para toda prueba.

---

# 178. Integration tests

También deberán existir pruebas reales sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 179. Platform conformance suite

Se propone:

```text
SchemaIntrospectorConformanceSuite
```

con casos comunes.

---

# 180. Conformance cases

Como mínimo:

```text
simple table
nullable column
literal default
expression default
primary key
composite primary key
unique constraint
composite unique
check
foreign key
composite foreign key
ordinary index
unique index
generated column
view
platform extension
```

---

# 181. Round-trip architecture test

```text
TableDefinition
      │
      ▼
Schema AST
      │
      ▼
Compiler
      │
      ▼
Execute
      │
      ▼
Introspect
      │
      ▼
Observed TableDefinition
      │
      ▼
PlatformNormalizedCompare
```

---

# 182. Round-trip limitation

No exigir:

```text
ExactEqual
```

si la plataforma normaliza físicamente la estructura.

Usar:

```text
PlatformNormalizedEquality
```

cuando corresponda.

---

# 183. Failure tests

Probar:

```text
permission denied
connection lost
timeout
cancellation
unsupported metadata
malformed metadata
unknown type
unparseable expression
partial visibility
budget exceeded
```

---

# 184. Performance tests

Medir:

```text
round trips
rows processed
objects/sec
normalization cost
memory/object
reference resolution cost
snapshot sealing cost
```

---

# 185. Complexity target

Para `n` objetos observados:

```text
Normalization ≈ O(n)
```

y resolución deberá buscar:

```text
O(1)
```

promedio mediante índices donde sea posible.

---

# 186. Reference indexing

Durante assembly:

```text
SchemaObjectPath → Object
ObjectId → Object
NormalizedIdentifier → CandidateSet
```

---

# 187. Ambiguous identifiers

Nunca resolver:

```text
first match
```

ante ambigüedad.

Debe producir:

```text
AMBIGUOUS
```

---

# 188. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Introspection
```

---

# 189. Proposed directory structure

```text
Schema/
└── Introspection/
    ├── Contract/
    │   ├── SchemaIntrospector.php
    │   ├── PlatformSchemaIntrospector.php
    │   ├── NativeSchemaMetadataReader.php
    │   ├── SchemaObservationNormalizer.php
    │   └── SchemaReferenceResolver.php
    │
    ├── Request/
    │   ├── SchemaIntrospectionRequest.php
    │   ├── SchemaIntrospectionScope.php
    │   ├── SchemaObjectSelection.php
    │   ├── SchemaIntrospectionDepth.php
    │   └── SchemaIntrospectionOptions.php
    │
    ├── Context/
    │   ├── SchemaIntrospectionContext.php
    │   └── SchemaIntrospectionSession.php
    │
    ├── Planning/
    │   ├── SchemaIntrospectionPlanner.php
    │   ├── SchemaIntrospectionPlan.php
    │   └── SchemaIntrospectionPlanStep.php
    │
    ├── Observation/
    │   ├── SchemaObservation.php
    │   ├── DatabaseObservation.php
    │   ├── NamespaceObservation.php
    │   ├── TableObservation.php
    │   ├── ColumnObservation.php
    │   ├── IndexObservation.php
    │   ├── ConstraintObservation.php
    │   ├── ForeignKeyObservation.php
    │   ├── SequenceObservation.php
    │   ├── ViewObservation.php
    │   ├── ObservedValue.php
    │   ├── ObservationCertainty.php
    │   └── ObservationProvenance.php
    │
    ├── Coverage/
    │   ├── SchemaCoverage.php
    │   ├── SchemaCoverageDescriptor.php
    │   ├── SchemaObjectCoverage.php
    │   └── SchemaObjectPresence.php
    │
    ├── Metadata/
    │   ├── NativeSchemaMetadata.php
    │   ├── SchemaObservationMetadata.php
    │   ├── TableObservationMetadata.php
    │   └── ColumnObservationMetadata.php
    │
    ├── Normalization/
    │   ├── Default/
    │   │   └── PlatformDefaultExpressionDecoder.php
    │   ├── Type/
    │   │   └── PlatformTypeDecoder.php
    │   ├── Expression/
    │   │   └── PlatformSchemaExpressionParser.php
    │   └── Identifier/
    │       └── PlatformIdentifierCanonicalizer.php
    │
    ├── Resolution/
    │   ├── SchemaReferenceResolver.php
    │   ├── ReferenceResolutionState.php
    │   └── SchemaObjectReferenceIndex.php
    │
    ├── Assembly/
    │   ├── SchemaObservationAssembler.php
    │   ├── SchemaAssemblyState.php
    │   └── SchemaSnapshotSealer.php
    │
    ├── Snapshot/
    │   ├── SchemaSnapshot.php
    │   ├── SchemaSnapshotId.php
    │   ├── SchemaSnapshotConsistency.php
    │   └── SchemaFingerprint.php
    │
    ├── Platform/
    │   ├── MySQL/
    │   │   ├── MySqlSchemaIntrospector.php
    │   │   ├── MySqlMetadataReader.php
    │   │   ├── MySqlTypeDecoder.php
    │   │   └── MySqlExpressionParser.php
    │   ├── MariaDB/
    │   │   ├── MariaDbSchemaIntrospector.php
    │   │   ├── MariaDbMetadataReader.php
    │   │   ├── MariaDbTypeDecoder.php
    │   │   └── MariaDbExpressionParser.php
    │   ├── PostgreSQL/
    │   │   ├── PostgreSqlSchemaIntrospector.php
    │   │   ├── PostgreSqlMetadataReader.php
    │   │   ├── PostgreSqlTypeDecoder.php
    │   │   └── PostgreSqlExpressionParser.php
    │   └── SQLite/
    │       ├── SqliteSchemaIntrospector.php
    │       ├── SqliteMetadataReader.php
    │       ├── SqliteTypeDecoder.php
    │       └── SqliteExpressionParser.php
    │
    ├── Budget/
    │   └── SchemaIntrospectionBudget.php
    │
    ├── Diagnostic/
    │   ├── SchemaIntrospectionDiagnostic.php
    │   └── SchemaIntrospectionDiagnosticSet.php
    │
    ├── Result/
    │   └── SchemaIntrospectionResult.php
    │
    ├── Extension/
    │   ├── SchemaIntrospectionExtension.php
    │   ├── ExtensionSchemaObjectObservation.php
    │   └── FrozenSchemaIntrospectionExtensionRegistry.php
    │
    └── Exception/
        ├── DatabaseSchemaIntrospectionException.php
        ├── SchemaIntrospectionConnectionException.php
        ├── SchemaIntrospectionPermissionException.php
        ├── SchemaIntrospectionQueryException.php
        ├── SchemaIntrospectionDecodeException.php
        ├── SchemaIntrospectionNormalizationException.php
        ├── SchemaIntrospectionReferenceException.php
        ├── SchemaIntrospectionCapabilityException.php
        ├── SchemaIntrospectionUnsupportedFeatureException.php
        ├── SchemaIntrospectionBudgetExceededException.php
        ├── SchemaIntrospectionCancelledException.php
        ├── SchemaIntrospectionTimeoutException.php
        ├── SchemaIntrospectionConsistencyException.php
        ├── SchemaIntrospectionExtensionException.php
        └── SchemaIntrospectionInvariantException.php
```

---

# 190. Architectural invariants

## DB-SCHEMA-INTRO-001
Schema Introspection observará estructura existente.

## DB-SCHEMA-INTRO-002
Schema Introspection no definirá estructura deseada.

## DB-SCHEMA-INTRO-003
Schema Introspection será distinta de Schema Builder.

## DB-SCHEMA-INTRO-004
Schema Introspection será distinta de Schema Diff.

## DB-SCHEMA-INTRO-005
Schema Introspection será distinta de Migration.

## DB-SCHEMA-INTRO-006
Schema Introspection será distinta de Schema Compiler.

## DB-SCHEMA-INTRO-007
Observed será distinto de Declared.

## DB-SCHEMA-INTRO-008
UNKNOWN será distinto de ABSENT.

## DB-SCHEMA-INTRO-009
PARTIAL será distinto de COMPLETE.

## DB-SCHEMA-INTRO-010
Native Metadata será distinta de Canonical Schema Model.

## DB-SCHEMA-INTRO-011
Introspection request tendrá scope explícito.

## DB-SCHEMA-INTRO-012
Coverage tendrá scope explícito.

## DB-SCHEMA-INTRO-013
Missing object bajo partial coverage no probará ausencia.

## DB-SCHEMA-INTRO-014
Permission failure no significará ausencia.

## DB-SCHEMA-INTRO-015
Unsupported metadata no significará ausencia.

## DB-SCHEMA-INTRO-016
Not requested no significará ausencia.

## DB-SCHEMA-INTRO-017
Observation tendrá provenance cuando esté disponible.

## DB-SCHEMA-INTRO-018
Observation tendrá certainty cuando sea necesaria.

## DB-SCHEMA-INTRO-019
EXACT será distinto de INFERRED.

## DB-SCHEMA-INTRO-020
INFERRED será distinto de UNKNOWN.

## DB-SCHEMA-INTRO-021
UNKNOWN será distinto de PLATFORM_DEFAULT.

## DB-SCHEMA-INTRO-022
SchemaSnapshot será inmutable.

## DB-SCHEMA-INTRO-023
Refresh producirá nuevo snapshot.

## DB-SCHEMA-INTRO-024
SchemaSnapshotId será distinto de SchemaFingerprint.

## DB-SCHEMA-INTRO-025
Snapshot timestamp no probará atomicidad.

## DB-SCHEMA-INTRO-026
Snapshot consistency será explícita.

## DB-SCHEMA-INTRO-027
IntrospectionSession será operation-scoped.

## DB-SCHEMA-INTRO-028
No existirán hidden connections.

## DB-SCHEMA-INTRO-029
Connection ownership será explícito.

## DB-SCHEMA-INTRO-030
Borrowed connection no será cerrada por introspector.

## DB-SCHEMA-INTRO-031
Platform introspection estará encapsulada.

## DB-SCHEMA-INTRO-032
MySQL tendrá adapter explícito.

## DB-SCHEMA-INTRO-033
MariaDB tendrá adapter first-class.

## DB-SCHEMA-INTRO-034
PostgreSQL tendrá adapter explícito.

## DB-SCHEMA-INTRO-035
SQLite tendrá adapter explícito.

## DB-SCHEMA-INTRO-036
Introspection podrá utilizar native metadata queries.

## DB-SCHEMA-INTRO-037
Native queries no escaparán como arquitectura portable.

## DB-SCHEMA-INTRO-038
High-level Query Builder no será dependencia obligatoria.

## DB-SCHEMA-INTRO-039
No existirán circular dependencies con Query Semantic Engine.

## DB-SCHEMA-INTRO-040
Table observation conservará structural definition.

## DB-SCHEMA-INTRO-041
Column order será preservado.

## DB-SCHEMA-INTRO-042
Native type será distinto de logical type.

## DB-SCHEMA-INTRO-043
Unknown type no será convertido arbitrariamente a string.

## DB-SCHEMA-INTRO-044
No Default será distinto de DEFAULT NULL.

## DB-SCHEMA-INTRO-045
Generated Column será distinta de Default.

## DB-SCHEMA-INTRO-046
AUTO_INCREMENT no será generación universal.

## DB-SCHEMA-INTRO-047
Index introspection preservará key order.

## DB-SCHEMA-INTRO-048
Unique Index será distinto de Unique Constraint.

## DB-SCHEMA-INTRO-049
Primary Key será distinta de backing index.

## DB-SCHEMA-INTRO-050
Expression index intentará preservar expresión estructurada.

## DB-SCHEMA-INTRO-051
Unparseable expression no desaparecerá silenciosamente.

## DB-SCHEMA-INTRO-052
Partial index predicate será preservado cuando sea observable.

## DB-SCHEMA-INTRO-053
Constraint introspection reutilizará canonical constraint definitions.

## DB-SCHEMA-INTRO-054
ForeignKey introspection reutilizará ForeignKeyDefinition.

## DB-SCHEMA-INTRO-055
Foreign key column pair order será preservado.

## DB-SCHEMA-INTRO-056
Unresolved reference será explícita.

## DB-SCHEMA-INTRO-057
External-to-snapshot reference no será inválida automáticamente.

## DB-SCHEMA-INTRO-058
Ambiguous reference no usará first-match resolution.

## DB-SCHEMA-INTRO-059
Sequence unsupported será distinto de absent.

## DB-SCHEMA-INTRO-060
View raw definition podrá preservarse.

## DB-SCHEMA-INTRO-061
Unparseable view no desaparecerá silenciosamente.

## DB-SCHEMA-INTRO-062
Platform-specific table options permanecerán tipadas.

## DB-SCHEMA-INTRO-063
No se inventará false portability.

## DB-SCHEMA-INTRO-064
Identifiers serán estructurados.

## DB-SCHEMA-INTRO-065
Identifier normalization será platform-aware.

## DB-SCHEMA-INTRO-066
Blind lowercase normalization estará prohibida.

## DB-SCHEMA-INTRO-067
Catalog y namespace no serán universalmente equivalentes.

## DB-SCHEMA-INTRO-068
SchemaObjectId será distinto de SchemaObjectPath.

## DB-SCHEMA-INTRO-069
SchemaObjectId será distinto de fingerprint.

## DB-SCHEMA-INTRO-070
Snapshot-local IDs no serán asumidos estables entre snapshots.

## DB-SCHEMA-INTRO-071
Observation será distinta de normalization.

## DB-SCHEMA-INTRO-072
Observation podrá realizar I/O.

## DB-SCHEMA-INTRO-073
Normalization será pure transformation.

## DB-SCHEMA-INTRO-074
Normalization no realizará hidden DB I/O.

## DB-SCHEMA-INTRO-075
Normalization será determinista.

## DB-SCHEMA-INTRO-076
Normalization preservará provenance.

## DB-SCHEMA-INTRO-077
Introspection planner podrá optimizar round trips.

## DB-SCHEMA-INTRO-078
Introspection planner no cambiará request semantics.

## DB-SCHEMA-INTRO-079
SchemaIntrospectionPlan será distinto de QueryPlan.

## DB-SCHEMA-INTRO-080
Capability checks gobernarán estrategias de introspection.

## DB-SCHEMA-INTRO-081
Version checks no reemplazarán capability checks en core.

## DB-SCHEMA-INTRO-082
Transaction ownership será explícito.

## DB-SCHEMA-INTRO-083
Introspector no hará COMMIT de transacción ajena.

## DB-SCHEMA-INTRO-084
Introspection failure será distinta de empty schema.

## DB-SCHEMA-INTRO-085
Partial failure será explícita.

## DB-SCHEMA-INTRO-086
Partial result tendrá diagnostics.

## DB-SCHEMA-INTRO-087
Partial snapshot no será safe destructive diff source por defecto.

## DB-SCHEMA-INTRO-088
Cancellation será distinta de timeout.

## DB-SCHEMA-INTRO-089
Timeout será distinto de connection failure.

## DB-SCHEMA-INTRO-090
Cancelled partial result no será marcado complete.

## DB-SCHEMA-INTRO-091
Resource budgets serán explícitos.

## DB-SCHEMA-INTRO-092
Budget exhaustion no truncará silenciosamente.

## DB-SCHEMA-INTRO-093
Streaming observation podrá existir.

## DB-SCHEMA-INTRO-094
Published snapshot será sealed.

## DB-SCHEMA-INTRO-095
Sealed snapshot no será mutable.

## DB-SCHEMA-INTRO-096
Column order no será canonicalizado alfabéticamente.

## DB-SCHEMA-INTRO-097
Constraint column order será preservado.

## DB-SCHEMA-INTRO-098
Index key order será preservado.

## DB-SCHEMA-INTRO-099
FK pair order será preservado.

## DB-SCHEMA-INTRO-100
Structural fingerprint excluirá runtime observation metadata.

## DB-SCHEMA-INTRO-101
Fingerprint será versionado.

## DB-SCHEMA-INTRO-102
Cache será integración opcional.

## DB-SCHEMA-INTRO-103
Cached snapshot no será asumido current.

## DB-SCHEMA-INTRO-104
Cache key no contendrá secretos.

## DB-SCHEMA-INTRO-105
Capability fingerprint participará en cache identity cuando corresponda.

## DB-SCHEMA-INTRO-106
Credentials no serán parte del snapshot.

## DB-SCHEMA-INTRO-107
Diagnostics no expondrán passwords.

## DB-SCHEMA-INTRO-108
Telemetry no expondrá credentials.

## DB-SCHEMA-INTRO-109
Introspection respetará database permissions.

## DB-SCHEMA-INTRO-110
Introspection no elevará privilegios.

## DB-SCHEMA-INTRO-111
Object filters serán tratados como identifiers/parameters seguros.

## DB-SCHEMA-INTRO-112
Schema introspection no leerá row data por defecto.

## DB-SCHEMA-INTRO-113
Telemetry será metadata-safe por defecto.

## DB-SCHEMA-INTRO-114
Extension registry será frozen.

## DB-SCHEMA-INTRO-115
Extension conflicts serán explícitos.

## DB-SCHEMA-INTRO-116
Extensions no convertirán unknown en absent silenciosamente.

## DB-SCHEMA-INTRO-117
Extensions no ocultarán diagnostics.

## DB-SCHEMA-INTRO-118
Unknown extension objects seguirán policy explícita.

## DB-SCHEMA-INTRO-119
ORM será consumidor, no propietario de introspection.

## DB-SCHEMA-INTRO-120
Query Semantic Engine no ejecutará hidden introspection.

## DB-SCHEMA-INTRO-121
Optimizer no ejecutará hidden introspection.

## DB-SCHEMA-INTRO-122
SchemaSnapshot podrá proyectar semantic facts.

## DB-SCHEMA-INTRO-123
Semantic facts conservarán evidence/trust.

## DB-SCHEMA-INTRO-124
Schema Diff verificará coverage.

## DB-SCHEMA-INTRO-125
Missing object bajo insufficient coverage no será drop candidate.

## DB-SCHEMA-INTRO-126
Migration podrá consumir introspection pero será subsistema separado.

## DB-SCHEMA-INTRO-127
Drift detection será distinta de migration.

## DB-SCHEMA-INTRO-128
Drift detection no aplicará cambios automáticamente.

## DB-SCHEMA-INTRO-129
Facade API no fusionará Builder e Introspector internamente.

## DB-SCHEMA-INTRO-130
hasTable pertenecerá conceptualmente a inspection.

## DB-SCHEMA-INTRO-131
hasColumn pertenecerá conceptualmente a inspection.

## DB-SCHEMA-INTRO-132
Boolean convenience API tendrá policy definida para UNKNOWN.

## DB-SCHEMA-INTRO-133
Shared introspection configuration será immutable.

## DB-SCHEMA-INTRO-134
Mutable introspection state será operation-scoped.

## DB-SCHEMA-INTRO-135
No habrá current schema mutable global.

## DB-SCHEMA-INTRO-136
No habrá current connection mutable global.

## DB-SCHEMA-INTRO-137
No habrá current tenant mutable global.

## DB-SCHEMA-INTRO-138
FrankenPHP request state será aislado.

## DB-SCHEMA-INTRO-139
RoadRunner worker state será aislado.

## DB-SCHEMA-INTRO-140
OpenSwoole coroutine/request state será aislado.

## DB-SCHEMA-INTRO-141
Multitenancy será integración opcional.

## DB-SCHEMA-INTRO-142
Tenant snapshot cache será aislado.

## DB-SCHEMA-INTRO-143
Tenant diagnostics context será aislado.

## DB-SCHEMA-INTRO-144
Conformance tests existirán por plataforma.

## DB-SCHEMA-INTRO-145
Round-trip tests utilizarán platform-normalized comparison cuando corresponda.

## DB-SCHEMA-INTRO-146
Native metadata fixtures serán testeables sin DB real.

## DB-SCHEMA-INTRO-147
Ambiguous metadata producirá diagnostic o estado explícito.

## DB-SCHEMA-INTRO-148
Introspection nunca inventará certainty.

## DB-SCHEMA-INTRO-149
Introspection nunca convertirá falta de evidencia en evidencia de ausencia.

## DB-SCHEMA-INTRO-150
SchemaSnapshot representará conocimiento observado, no verdad metafísica absoluta sobre la base de datos.

---

# 191. Anti-patterns

## 191.1 Ausencia por falta de metadata

Incorrecto:

```php
if (!$row) {
    return SchemaObjectPresence::ABSENT;
}
```

sin conocer coverage.

---

## 191.2 Empty schema after failure

Incorrecto:

```php
try {
    return $introspector->inspect();
} catch (\Throwable) {
    return DatabaseSchema::empty();
}
```

Esto convierte:

```text
failure
```

en:

```text
absence
```

---

## 191.3 Generic platform introspector

Incorrecto:

```php
class SqlSchemaIntrospector
{
    if ($driver === 'mysql') { ... }
    if ($driver === 'pgsql') { ... }
    if ($driver === 'sqlite') { ... }
}
```

Preferido:

```text
Platform-specific adapters
+
shared contracts
```

---

## 191.4 MySQL = MariaDB

Incorrecto:

```php
class MariaDbSchemaIntrospector
    extends MySqlSchemaIntrospector
{
}
```

sin analizar diferencias reales de capabilities.

---

## 191.5 Query Builder circularity

Incorrecto:

```text
SchemaIntrospector
    ↓
Schema-aware QueryBuilder
    ↓
Semantic Engine
    ↓
SchemaIntrospector
```

---

## 191.6 Blind lowercase

Incorrecto:

```php
$name = strtolower($nativeName);
```

---

## 191.7 Unknown type → string

Incorrecto:

```php
$type = $known[$nativeType] ?? new StringType();
```

Preferido:

```text
UnknownDatabaseType
```

o:

```text
OpaquePlatformType
```

---

## 191.8 Unique index → unique constraint

Incorrecto:

```text
Every UNIQUE INDEX
        ↓
UniqueConstraint
```

---

## 191.9 Lost check constraint

Incorrecto:

```text
Cannot parse CHECK expression
        ↓
Ignore constraint
```

Preferido:

```text
RawSchemaExpression
+
Diagnostic
```

---

## 191.10 Mutable snapshot

Incorrecto:

```php
$snapshot->tables[] = $table;
```

después de publicarlo.

---

## 191.11 Hidden refresh

Incorrecto:

```php
$snapshot->table('users');
// secretly queries database again
```

---

## 191.12 Hidden introspection in optimizer

Incorrecto:

```php
$optimizer->optimize($query);
// silently introspects database
```

---

# 192. Ejemplo conceptual completo

Base de datos física:

```text
users
├── id BIGINT PRIMARY KEY
├── email VARCHAR(255) NOT NULL
├── age INT
├── created_at TIMESTAMP
│
├── UNIQUE(email)
└── CHECK(age >= 0)
```

Observación:

```text
Native Metadata
│
├── Table: users
├── Columns
│   ├── id
│   ├── email
│   ├── age
│   └── created_at
│
├── Primary Key
├── Unique Constraint
└── Check Constraint
```

Normalización:

```text
TableDefinition(users)
│
├── ColumnDefinition(id)
│   └── IntegerType(64)
│
├── ColumnDefinition(email)
│   └── StringType(255)
│
├── ColumnDefinition(age)
│   └── IntegerType(...)
│
├── ColumnDefinition(created_at)
│   └── DateTimeType(...)
│
├── PrimaryKeyConstraint(id)
├── UniqueConstraint(email)
└── CheckConstraint(
        age >= 0
    )
```

Snapshot:

```text
SchemaSnapshot
├── schema
│   └── users
├── coverage
│   └── COMPLETE(users)
├── consistency
│   └── BEST_EFFORT
├── provenance
│   └── SYSTEM_CATALOG
└── diagnostics
```

---

# 193. Ejemplo de snapshot parcial

Request:

```text
Inspect:
orders
```

Resultado:

```text
SchemaSnapshot
├── scope
│   └── TABLE(orders)
├── coverage
│   └── COMPLETE within scope
│
└── orders
    ├── id
    ├── customer_id
    └── FK → customers.id
```

`customers` no está en el snapshot.

Resultado correcto:

```text
customers
=
EXTERNAL_TO_SNAPSHOT
```

No:

```text
customers
=
ABSENT
```

---

# 194. Ejemplo de incertidumbre

Metadata nativa:

```text
CHECK constraint exists
```

pero la expresión no puede reconstruirse.

Representación:

```text
CheckConstraintDefinition
├── name = ck_orders_total
├── predicate
│   └── RawSchemaExpression(...)
│
└── observation
    ├── presence = PRESENT
    ├── expression certainty = UNKNOWN/PARTIAL
    └── diagnostic = EXPRESSION_NOT_STRUCTURALLY_PARSED
```

Esto conserva más verdad que eliminar la constraint.

---

# 195. Fórmula de observación

Sea:

```text
D = physical database
R = introspection request
P = platform capabilities
A = access permissions
```

entonces:

```text
Observation
=
Observe(D, R, P, A)
```

y no necesariamente:

```text
Observation
=
CompleteTruth(D)
```

---

# 196. Fórmula de snapshot

```text
SchemaSnapshot
=
CanonicalSchema
+
Coverage
+
ObservationMetadata
+
Provenance
+
Certainty
+
Consistency
+
Diagnostics
```

---

# 197. Fórmula de ausencia segura

Para objeto `O`:

```text
SafelyAbsent(O)
=
NotObserved(O)
∧
CoverageRelevantTo(O) = COMPLETE
∧
VisibilityRelevantTo(O) = SUFFICIENT
∧
ObservationSucceeded
```

Por tanto:

```text
NotObserved(O)
≠
SafelyAbsent(O)
```

---

# 198. Fórmula de normalización

```text
CanonicalObservation
=
Normalize(
    NativeMetadata,
    PlatformCapabilities,
    NormalizationProfile
)
```

con:

```text
Normalize(Normalize(x))
≈
Normalize(x)
```

para representaciones compatibles.

---

# 199. Fórmula de seguridad para Schema Diff

```text
SafeDestructiveDiff
=
CompleteRelevantCoverage
∧
SufficientVisibility
∧
SuccessfulObservation
∧
ResolvedCriticalReferences
∧
NoCriticalUnknownMetadata
```

Esto será fundamental para los documentos posteriores.

---

# 200. Arquitectura final

```text
                  Physical Database
                         │
                         ▼
                 Database Connection
                         │
                         ▼
                Database Platform
                         │
                         ▼
             Platform Schema Introspector
                         │
                         ▼
                Native Metadata Reader
                         │
                         ▼
                 Raw Observations
                         │
                         ▼
                 Metadata Decoders
                         │
                         ▼
              Structural Normalization
                         │
                         ▼
                Reference Resolution
                         │
                         ▼
                Coverage Analysis
                         │
                         ▼
                  Schema Assembly
                         │
                         ▼
                     Seal
                         │
                         ▼
                 SchemaSnapshot
                  /      |      \
                 /       |       \
                ▼        ▼        ▼
          Schema Diff   ORM   Semantic Engine
                │
                ▼
          Migration Planning
```

---

# 201. Regla arquitectónica final

> **VoltStack no debe interpretar la introspección como una lectura perfecta de la realidad, sino como una observación estructurada acompañada de evidencia, alcance, certeza y procedencia.**

La regla puede resumirse como:

```text
Observe what is known.
Preserve what is unknown.
Never confuse unknown with absent.
```

---

# 202. Resultado arquitectónico

Con este sistema VoltStack podrá realizar:

```text
Database
   ↓
Introspection
   ↓
Typed Observation
   ↓
Normalization
   ↓
Immutable SchemaSnapshot
```

sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

manteniendo suficiente precisión para alimentar posteriormente:

```text
Schema Metadata
Schema Diff
Migrations
ORM Validation
Query Semantic Analysis
Developer Tooling
Diagnostics
Drift Detection
```

sin introducir dependencias circulares ni inferencias destructivas inseguras.

---

# 203. Siguiente documento

```text
97_DATABASE_SCHEMA_METADATA_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack representa y gobierna la metadata asociada al Schema Model y a los snapshots observados:

```text
Schema Object
      │
      ├── Structural Metadata
      ├── Semantic Metadata
      ├── Physical Metadata
      ├── Informational Metadata
      ├── Observation Metadata
      ├── Provenance
      ├── Certainty
      ├── Coverage
      └── Platform Extensions
```

Deberá formalizar especialmente:

```text
Schema Structure
≠
Schema Metadata

Metadata
≠
Structure

Observed Metadata
≠
Declared Metadata

Unknown Metadata
≠
Default Metadata

Informational Metadata
≠
Semantic Metadata
```

y proporcionar la base necesaria para:

```text
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
```