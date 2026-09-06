# 95_DATABASE_CONSTRAINT_SYSTEM.md

# VoltStack Quantum Database
## Database Constraint System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 95 — Database Constraint System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Constraint System` define el modelo arquitectónico mediante el cual VoltStack representa las **reglas estructurales de integridad declaradas y, cuando corresponda, impuestas por la base de datos**.

El sistema deberá proporcionar una abstracción común para constraints como:

```text
PRIMARY KEY
UNIQUE
CHECK
FOREIGN KEY
```

sin colapsar sus diferencias semánticas.

La pregunta fundamental es:

> **¿Cómo representa VoltStack una regla estructural de integridad de datos de forma tipada, inmutable, portable, introspectable y extensible, sin confundirla con índices, validación de aplicación, políticas de autorización o reglas ORM?**

Modelo conceptual:

```text
ConstraintDefinition
├── ConstraintDefinitionId
├── ConstraintIdentifier
├── ConstraintKind
├── ConstraintScope
├── EnforcementState
├── ValidationState
├── TrustState
├── Deferrability
├── CapabilityRequirements
├── Metadata
└── Extensions
```

Especializaciones:

```text
ConstraintDefinition
│
├── PrimaryKeyConstraintDefinition
├── UniqueConstraintDefinition
├── CheckConstraintDefinition
├── ForeignKeyDefinition
└── ExtensionConstraintDefinition
```

Separación fundamental:

```text
Database Constraint
≠
Index
≠
Application Validation Rule
≠
ORM Validation
≠
Authorization Policy
≠
Business Rule
≠
Trigger
≠
SQL Fragment
```

---

# 2. Principio central

Una database constraint representa una regla declarativa sobre el estado permitido de la estructura o los datos de una base de datos.

Formalmente, si:

```text
S
```

representa un estado de datos y:

```text
C
```

una constraint, entonces:

```text
Valid(S, C)
```

indica que el estado satisface la constraint.

Conceptualmente:

```text
Constraint C
      │
      ▼
Defines allowed database states
      │
      ▼
Valid State / Invalid State
```

Pero la definición estructural no ejecuta por sí misma la comprobación.

---

# 3. Regla maestra

> **ConstraintDefinition describe una regla de integridad estructural; no implementa el mecanismo físico mediante el cual una plataforma la impone.**

Por tanto:

```text
ConstraintDefinition
        │
        ▼
Schema Planner
        │
        ▼
Schema Compiler
        │
        ▼
Platform Representation
```

y no:

```text
ConstraintDefinition
        │
        ▼
raw SQL
```

---

# 4. Posición arquitectónica

```text
Schema Builder
     │
     ▼
Constraint Blueprint
     │
     ▼
Constraint Definition
     │
     ▼
Table Definition
     │
     ▼
Schema Model
     │
     ├──────────────► Schema Introspection
     │
     ├──────────────► Schema Diff
     │
     └──────────────► Semantic Consumers
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
```

---

# 5. Objetivos

El sistema deberá soportar:

- constraints tipadas;
- identidad estructural;
- nombres explícitos;
- nombres generados;
- constraints anónimas cuando la plataforma lo permita;
- primary keys;
- unique constraints;
- check constraints;
- foreign keys;
- constraints simples y compuestas;
- expresiones estructuradas;
- enforcement state;
- validation state;
- trust state;
- deferrability;
- capability requirements;
- introspection;
- metadata;
- provenance;
- canonicalization;
- comparison;
- fingerprinting;
- serialization;
- Schema Diff;
- extensiones;
- persistent-runtime safety.

---

# 6. No objetivos

`Database Constraint System` no deberá:

- generar SQL;
- ejecutar DDL;
- abrir conexiones;
- administrar transacciones;
- validar formularios;
- implementar reglas de negocio;
- implementar autorización;
- implementar ORM lifecycle validation;
- ejecutar triggers;
- crear índices ocultamente;
- modificar entidades;
- realizar consultas;
- acceder al request actual;
- mantener mutable global state.

---

# 7. Taxonomía principal

VoltStack reconocerá inicialmente:

```text
ConstraintKind
├── PRIMARY_KEY
├── UNIQUE
├── CHECK
├── FOREIGN_KEY
└── EXTENSION
```

Posibles extensiones futuras:

```text
EXCLUSION
ASSERTION
PLATFORM_SPECIFIC
```

si existen capacidades suficientemente estructuradas para modelarlas.

---

# 8. ConstraintDefinition

Se propone un contrato conceptual:

```php
interface ConstraintDefinition
{
    public function id(): ConstraintDefinitionId;

    public function identifier(): ConstraintIdentifier;

    public function kind(): ConstraintKind;

    public function enforcement(): ConstraintEnforcementState;

    public function validationState(): ConstraintValidationState;

    public function trustState(): ConstraintTrustState;

    public function capabilities(): SchemaCapabilityRequirementSet;

    public function metadata(): ConstraintMetadata;

    public function extensions(): ExtensionMetadataSet;
}
```

No necesariamente todas las constraints tendrán exactamente las mismas propiedades.

---

# 9. Evitar una God Constraint

No deberá diseñarse:

```php
final class ConstraintDefinition
{
    public ?array $columns;
    public ?string $expression;
    public ?string $referencedTable;
    public ?array $referencedColumns;
    public ?string $onDelete;
    public ?string $onUpdate;
    public ?bool $primary;
    public ?bool $unique;
}
```

Esto genera estados imposibles.

Ejemplo:

```text
CHECK constraint
+
referencedTable
+
ON DELETE CASCADE
```

no tiene sentido.

---

# 10. Diseño por especialización

Preferido:

```text
ConstraintDefinition
       │
       ├── PrimaryKeyConstraintDefinition
       │
       ├── UniqueConstraintDefinition
       │
       ├── CheckConstraintDefinition
       │
       ├── ForeignKeyDefinition
       │
       └── ExtensionConstraintDefinition
```

Cada tipo conserva sus invariantes particulares.

---

# 11. ConstraintDefinitionId

Debe existir identidad interna:

```php
final readonly class ConstraintDefinitionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Esta identidad sirve para:

- grafos internos;
- diagnostics;
- diff;
- mapping;
- AST;
- debugging.

---

# 12. ConstraintDefinitionId ≠ database name

Debe cumplirse:

```text
ConstraintDefinitionId
≠
ConstraintIdentifier
```

Una constraint puede no tener nombre explícito en el modelo físico.

---

# 13. ConstraintIdentifier

Representará el nombre estructural visible para la base de datos.

Ejemplo:

```text
pk_users
uq_users_email
ck_orders_total
fk_orders_customer
```

No contendrá quoting SQL.

---

# 14. Identifier rendering

Internamente:

```text
uq_users_email
```

El compiler podrá renderizar:

```text
"uq_users_email"
```

o:

```text
`uq_users_email`
```

según dialecto.

Por tanto:

```text
DeclaredIdentifier
≠
RenderedIdentifier
```

---

# 15. Explicit vs generated name

Debe distinguirse:

```text
ConstraintNameOrigin
├── EXPLICIT
├── GENERATED
├── PLATFORM_GENERATED
├── INTROSPECTED
└── UNKNOWN
```

---

# 16. Naming strategy

Los nombres generados deberán provenir de:

```text
SchemaNamingStrategy
```

Ejemplo:

```text
users.email
        │
        ▼
uq_users_email
```

---

# 17. Naming determinista

Debe cumplirse:

```text
Same Constraint Structure
+
Same Naming Strategy
+
Same Naming Strategy Version
=
Same Generated Identifier
```

---

# 18. Identifier length

Algunas plataformas limitan el tamaño de identifiers.

El naming strategy podrá producir:

```text
HumanReadablePrefix
+
StableHashSuffix
```

Ejemplo conceptual:

```text
uq_customer_notification_preferences_8f41a2
```

---

# 19. Collision handling

Debe existir detección determinista de colisiones.

Nunca:

```text
first one wins
```

ni:

```text
last one wins
```

---

# 20. ConstraintScope

Una constraint puede tener scope:

```text
TABLE
COLUMN
SCHEMA
DATABASE
PLATFORM_SPECIFIC
```

Sin embargo, la representación canónica deberá reflejar el scope semántico real.

---

# 21. Column syntax ≠ column scope

Ejemplo:

```sql
email VARCHAR(255) UNIQUE
```

puede ser sintaxis inline.

Pero estructuralmente:

```text
UNIQUE(email)
```

puede modelarse como table-level unique constraint.

Por tanto:

```text
SQL placement
≠
Constraint semantic scope
```

---

# 22. Primary Key Constraint

Se propone:

```php
final readonly class PrimaryKeyConstraintDefinition
    implements ConstraintDefinition
{
    public function __construct(
        public ConstraintDefinitionId $id,
        public ConstraintIdentifier $identifier,
        public OrderedColumnReferenceSet $columns,
        public ConstraintMetadata $metadata,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

---

# 23. Primary key semantics

Una primary key representa una clave principal candidata a identificar filas.

Conceptualmente:

```text
PRIMARY KEY(id)
```

o:

```text
PRIMARY KEY(tenant_id, id)
```

---

# 24. Primary Key ≠ Index

Debe mantenerse:

```text
PrimaryKeyConstraintDefinition
≠
IndexDefinition
```

Aunque una plataforma implemente la PK mediante un índice.

---

# 25. Physical backing index

Si una plataforma crea:

```text
PRIMARY KEY
        ↓
UNIQUE INDEX
```

esa relación será metadata/planning/platform behavior.

No identidad conceptual.

---

# 26. Composite primary key

Debe soportarse:

```text
PRIMARY KEY(
    tenant_id,
    invoice_id
)
```

El orden deberá preservarse.

---

# 27. Primary key cardinality

Una tabla tendrá normalmente como máximo una primary key estructural.

Formalmente:

```text
Count(Table.PrimaryKeyConstraint) ≤ 1
```

salvo que una extensión de plataforma declare explícitamente otra semántica.

---

# 28. Empty primary key

Nunca será válida:

```text
PRIMARY KEY()
```

Debe cumplirse:

```text
|PK.columns| >= 1
```

---

# 29. Duplicate PK columns

Será inválido:

```text
PRIMARY KEY(id, id)
```

---

# 30. Primary key and nullability

El Schema Semantic Validator deberá considerar las reglas de nullability asociadas a una primary key.

Pero la normalización no deberá modificar silenciosamente:

```text
nullable = true
```

a:

```text
nullable = false
```

sin representar esa consecuencia explícitamente.

---

# 31. Unique Constraint

Se propone:

```php
final readonly class UniqueConstraintDefinition
    implements ConstraintDefinition
{
    public function __construct(
        public ConstraintDefinitionId $id,
        public ConstraintIdentifier $identifier,
        public OrderedColumnReferenceSet $columns,
        public ConstraintDeferrability $deferrability,
        public ConstraintMetadata $metadata,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

---

# 32. Unique Constraint semantics

Conceptualmente:

```text
UNIQUE(email)
```

o:

```text
UNIQUE(
    tenant_id,
    external_id
)
```

---

# 33. Unique Constraint ≠ Unique Index

Regla crítica:

```text
UniqueConstraintDefinition
≠
UniqueIndexDefinition
```

Ambos pueden producir efectos de unicidad similares.

Pero arquitectónicamente representan conceptos diferentes.

---

# 34. Unique constraint

Expresa:

```text
integrity rule
```

Unique index expresa:

```text
access structure
+
uniqueness property
```

---

# 35. Platform normalization

Una plataforma puede implementar:

```text
UNIQUE CONSTRAINT
```

mediante:

```text
UNIQUE INDEX
```

Pero:

```text
PhysicalImplementation
≠
LogicalDefinition
```

---

# 36. Composite unique constraint

Debe preservarse:

```text
UNIQUE(tenant_id, slug)
```

El orden no deberá alterarse arbitrariamente.

---

# 37. NULL uniqueness semantics

La semántica de NULL en unique constraints puede variar por plataforma y configuración.

Por tanto:

```text
UNIQUE(nullable_column)
```

no deberá generar inferencias universales incorrectas.

---

# 38. Null distinctness

El modelo podrá necesitar:

```text
UniqueNullSemantics
├── PLATFORM_DEFAULT
├── NULLS_DISTINCT
├── NULLS_NOT_DISTINCT
└── UNKNOWN
```

cuando la plataforma soporte una semántica declarable.

---

# 39. Unique capability requirements

Una definición:

```text
NULLS NOT DISTINCT
```

deberá generar capability requirement correspondiente.

Nunca degradarse silenciosamente a:

```text
UNIQUE
```

convencional.

---

# 40. Check Constraint

Se propone:

```php
final readonly class CheckConstraintDefinition
    implements ConstraintDefinition
{
    public function __construct(
        public ConstraintDefinitionId $id,
        public ConstraintIdentifier $identifier,
        public SchemaExpression $predicate,
        public ConstraintEnforcementState $enforcement,
        public ConstraintValidationState $validationState,
        public ConstraintMetadata $metadata,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

---

# 41. Check semantics

Ejemplo:

```text
CHECK(total >= 0)
```

expresa una condición que las filas deben satisfacer bajo la semántica correspondiente de la plataforma.

---

# 42. Check ≠ application validation

Debe mantenerse:

```text
CHECK(age >= 18)
≠
Validator::min(18)
```

La primera es integridad de base de datos.

La segunda es validación de aplicación.

---

# 43. Application validation may be stricter

Ejemplo:

```text
Database:
CHECK(amount >= 0)

Application:
amount > 0
```

puede ser perfectamente válido.

No deben sincronizarse automáticamente.

---

# 44. Check expression

La expresión deberá ser estructurada:

```text
SchemaExpression
```

No simplemente:

```php
"total >= 0"
```

---

# 45. Example expression AST

```text
BinaryExpression(GREATER_THAN_OR_EQUAL)
├── ColumnReference(total)
└── Literal(0)
```

---

# 46. SchemaExpression ≠ QueryExpression

Aunque puedan compartir primitivas:

```text
SchemaExpression
≠
QueryExpression
```

porque tienen contextos, capacidades y restricciones diferentes.

---

# 47. Raw check expression

Se podrá proporcionar escape hatch explícito:

```text
RawSchemaExpression
```

con:

```text
trust level
platform scope
capability requirements
portability warning
```

---

# 48. Raw expression must be explicit

Nunca convertir automáticamente:

```php
$table->check('price > 0');
```

en un raw trusted expression sin que la API haga explícito ese boundary.

Podría ofrecerse una API claramente denominada:

```php
$table->rawCheck(
    RawSchemaExpression::trusted('price > 0')
);
```

---

# 49. Check expression validation

Deberá validar:

- referenced columns;
- expression node kinds;
- function capabilities;
- determinism requirements si la plataforma los exige;
- subquery restrictions;
- volatile functions;
- raw boundaries;
- extension nodes.

---

# 50. Check result semantics

Una check expression no deberá asumirse equivalente a un booleano PHP.

Debe respetarse SQL logic.

En particular:

```text
TRUE
FALSE
UNKNOWN
```

pueden tener semántica relevante.

---

# 51. Foreign Key integration

`ForeignKeyDefinition` del documento 94 deberá integrarse en la taxonomía de constraints.

Conceptualmente:

```text
ForeignKeyDefinition
implements
ConstraintDefinition
```

o utilizar un contrato especializado equivalente.

---

# 52. No duplicar ForeignKeyDefinition

No crear:

```text
ForeignKeyDefinition
```

y:

```text
ForeignKeyConstraintDefinition
```

como dos fuentes de verdad equivalentes.

Debe existir una única representación canónica.

---

# 53. Foreign key specialization

La foreign key añade:

```text
local columns
referenced table
referenced columns
match semantics
on update
on delete
deferrability
validation state
```

sobre las propiedades comunes de constraint.

---

# 54. Constraint common contract

Las propiedades verdaderamente comunes podrán ser:

```text
identity
kind
name
metadata
capability requirements
extensions
```

No debe forzarse que toda constraint tenga:

```text
deferrability
validation state
enforcement state
```

si el concepto no aplica.

---

# 55. Applicability modeling

Preferido:

```text
PrimaryKeyConstraintDefinition
UniqueConstraintDefinition
CheckConstraintDefinition
ForeignKeyDefinition
```

cada una con sus propiedades válidas.

No:

```text
deferrability = null
```

en 20 tipos distintos.

---

# 56. Absent ≠ not applicable

Si una propiedad no aplica:

```text
NOT_APPLICABLE
```

es conceptualmente diferente de:

```text
UNKNOWN
```

y:

```text
PLATFORM_DEFAULT
```

Pero preferiblemente el tipo especializado ni siquiera expondrá la propiedad.

---

# 57. Constraint enforcement

Algunas plataformas permiten constraints:

```text
ENFORCED
NOT ENFORCED
```

o estados equivalentes.

VoltStack deberá modelar:

```text
ConstraintEnforcementState
├── ENFORCED
├── NOT_ENFORCED
├── PLATFORM_DEFAULT
└── UNKNOWN
```

cuando aplique.

---

# 58. Enforcement ≠ validation

Debe cumplirse:

```text
ConstraintEnforcementState
≠
ConstraintValidationState
```

---

# 59. Example

Una constraint podría estar:

```text
NOT VALIDATED
```

pero:

```text
ENFORCED
```

para nuevas modificaciones, dependiendo de plataforma.

Por tanto ambas dimensiones son independientes.

---

# 60. Constraint validation state

Se propone:

```text
ConstraintValidationState
├── VALIDATED
├── NOT_VALIDATED
├── PLATFORM_DEFAULT
└── UNKNOWN
```

---

# 61. Validation state meaning

`VALIDATED` podrá significar:

```text
existing relevant data has been validated
```

según la semántica concreta de plataforma.

No significa:

```text
VoltStack PHP object passed constructor validation
```

---

# 62. Structural validity ≠ data validation

Debe distinguirse:

```text
ValidConstraintDefinition
```

de:

```text
ConstraintValidatedAgainstExistingData
```

---

# 63. Trust state

Los consumidores analíticos necesitan saber si pueden usar una constraint como evidencia.

Se propone:

```text
ConstraintTrustState
├── TRUSTED
├── UNTRUSTED
├── PARTIALLY_TRUSTED
├── DISABLED
├── UNKNOWN
└── PLATFORM_SPECIFIC
```

---

# 64. Trust ≠ enforcement

Una constraint puede estar:

```text
ENFORCED
```

para nuevos datos pero no completamente validada para datos históricos.

Por tanto:

```text
Enforcement
≠
Validation
≠
Trust
```

---

# 65. Trust derivation

`ConstraintTrustState` podrá ser:

- observado;
- derivado;
- declarado;
- unknown.

Su provenance deberá conservarse.

---

# 66. Semantic consumer safety

Query Optimizer nunca deberá asumir:

```text
Constraint exists
⇒
Constraint is trusted
```

---

# 67. Deferrability

Cuando una constraint soporte deferrability:

```text
ConstraintDeferrability
├── NOT_DEFERRABLE
├── DEFERRABLE_INITIALLY_IMMEDIATE
├── DEFERRABLE_INITIALLY_DEFERRED
├── PLATFORM_DEFAULT
└── UNKNOWN
```

---

# 68. Deferrability ≠ current runtime mode

Debe mantenerse:

```text
ConstraintDefinition.deferrability
≠
Transaction.currentConstraintMode
```

---

# 69. Primary key deferrability

Si una plataforma permite deferrable primary/unique constraints, deberá expresarse mediante capabilities.

No asumir universalmente que:

```text
PRIMARY KEY = NOT DEFERRABLE
```

---

# 70. Capability-driven semantics

Ejemplo:

```text
UniqueConstraint
└── DEFERRABLE INITIALLY DEFERRED
```

produce:

```text
CapabilityRequirement:
DEFERRABLE_UNIQUE_CONSTRAINT
```

---

# 71. Constraint capability model

Se propone:

```text
ConstraintCapabilityRequirementSet
```

derivado de:

```text
ConstraintKind
+
ConstraintProperties
+
ExpressionFeatures
+
Extensions
```

---

# 72. Capability examples

```text
CHECK_CONSTRAINTS
FOREIGN_KEYS
DEFERRABLE_CONSTRAINTS
DEFERRABLE_UNIQUE_CONSTRAINTS
DEFERRABLE_FOREIGN_KEYS
NOT_VALID_CONSTRAINTS
NOT_ENFORCED_CONSTRAINTS
NULLS_NOT_DISTINCT_UNIQUE
NAMED_CONSTRAINTS
ALTER_CONSTRAINT
DROP_CONSTRAINT
```

---

# 73. Version ≠ capability

Nunca:

```php
if ($postgresVersion >= 15) {
    ...
}
```

en el core Constraint System.

Debe utilizarse:

```php
if ($capabilities->supports(
    SchemaCapability::NULLS_NOT_DISTINCT_UNIQUE
)) {
    ...
}
```

---

# 74. Platform-specific semantics

MySQL, MariaDB, PostgreSQL y SQLite podrán tener diferencias significativas.

El core deberá expresar:

```text
structural intent
```

y:

```text
capability requirements
```

sin vendor conditionals dispersos.

---

# 75. MySQL

Las diferencias históricas o actuales respecto a:

- CHECK;
- FK;
- constraint naming;
- enforcement;
- metadata;

deberán encapsularse en:

```text
MySqlSchemaPlatform
MySqlSchemaIntrospector
MySqlSchemaCompiler
```

---

# 76. MariaDB

MariaDB será plataforma first-class.

Nunca:

```text
MariaDB = MySQL
```

como regla arquitectónica.

---

# 77. PostgreSQL

Características avanzadas podrán proyectarse mediante capabilities:

```text
DEFERRABLE
INITIALLY DEFERRED
NOT VALID
NULLS NOT DISTINCT
```

cuando correspondan.

---

# 78. SQLite

Las limitaciones de alteración, constraint representation e introspection deberán ser explícitas.

Una operación compleja puede requerir table rebuild.

Pero:

```text
ConstraintDefinition
```

no decide reconstruir la tabla.

---

# 79. Constraint collection

`TableDefinition` podrá contener:

```text
ConstraintDefinitionSet
```

además de vistas especializadas.

Ejemplo:

```text
TableDefinition
├── Columns
├── PrimaryKey?
├── UniqueConstraints
├── CheckConstraints
├── ForeignKeys
└── Indexes
```

---

# 80. One source of truth

Debe evitarse almacenar simultáneamente:

```text
$primaryKey
$constraints['primary']
```

como estructuras independientes.

Una puede ser una vista/index derivado de la otra.

---

# 81. ConstraintDefinitionSet

Se propone una colección inmutable:

```php
final readonly class ConstraintDefinitionSet
{
    // deterministic immutable collection
}
```

---

# 82. Deterministic iteration

El orden de iteración deberá ser determinista.

Pero:

```text
collection iteration order
≠
SQL execution order
```

---

# 83. Duplicate detection

El set deberá detectar:

- duplicate internal IDs;
- duplicate explicit identifiers;
- multiple PKs;
- structural duplicates;
- incompatible constraints.

---

# 84. Structural duplicate

Ejemplo:

```text
UNIQUE(email)
UNIQUE(email)
```

podrá producir diagnóstico de duplicidad.

Pero no deberá eliminarse automáticamente sin considerar:

- nombres;
- metadata;
- deferrability;
- platform semantics;
- extensions.

---

# 85. Constraint normalization

Se propone:

```text
ConstraintDefinitionNormalizer
```

---

# 86. Normalization requirements

Debe ser:

```text
deterministic
idempotent
side-effect free
semantics-preserving
```

---

# 87. No DB I/O during normalization

Nunca:

```text
normalize()
    ↓
query information_schema
```

---

# 88. Explicit context

Si la normalización necesita información:

```text
ConstraintNormalizationContext
```

deberá suministrarla explícitamente.

---

# 89. Column order preservation

Para constraints compuestas:

```text
UNIQUE(a,b)
PRIMARY KEY(a,b)
FOREIGN KEY(a,b) REFERENCES x(c,d)
```

el orden deberá preservarse.

---

# 90. Constraint expression normalization

Para CHECK:

```text
CHECK(price >= 0)
```

la expresión podrá canonicalizarse estructuralmente.

Pero no realizar reescrituras agresivas que cambien SQL three-valued semantics.

---

# 91. Constraint validation pipeline

```text
Constraint Definition
        │
        ▼
Local Structural Validation
        │
        ▼
Table Context Validation
        │
        ▼
Schema Reference Validation
        │
        ▼
Semantic Validation
        │
        ▼
Capability Validation
        │
        ▼
Planning Validation
        │
        ▼
Migration Safety
```

---

# 92. Local validation

Puede verificar:

```text
identifier
kind
non-empty column sets
duplicate columns
expression structure
extension validity
property combinations
```

---

# 93. Table-context validation

Puede verificar:

```text
referenced local columns exist
only one primary key
constraint names do not collide
check expressions reference valid columns
```

---

# 94. Schema-context validation

Puede verificar:

```text
foreign key targets
cross-schema references
referenced key eligibility
schema-level extension constraints
```

---

# 95. Platform validation

Puede verificar:

```text
constraint kind supported
deferrability supported
validation state supported
enforcement state supported
expression features supported
alter strategy possible
```

---

# 96. Unsupported ≠ invalid

Debe distinguirse:

```text
ValidConstraintDefinition
+
UnsupportedOnPlatform
```

de:

```text
InvalidConstraintDefinition
```

---

# 97. Example

```text
UNIQUE(email)
DEFERRABLE INITIALLY DEFERRED
```

puede ser estructuralmente válido.

Si plataforma `P` no soporta esa semántica:

```text
ConstraintCapabilityException
```

---

# 98. Constraint diagnostics

Los diagnostics deberán ser estructurados.

Ejemplo:

```text
code:
DB-CONSTRAINT-UNSUPPORTED-DEFERRABILITY

constraint:
uq_users_email

requiredCapability:
DEFERRABLE_UNIQUE_CONSTRAINT

platform:
mysql
```

---

# 99. Introspection architecture

```text
Database Catalog
      │
      ▼
Platform Schema Introspector
      │
      ▼
Native Constraint Metadata
      │
      ▼
Constraint Metadata Normalizer
      │
      ▼
ObservedConstraint
```

---

# 100. ObservedConstraint

Se propone:

```php
final readonly class ObservedConstraint
{
    public function __construct(
        public ConstraintDefinition $definition,
        public DefinitionCompleteness $completeness,
        public ConstraintObservationMetadata $observation,
    ) {}
}
```

---

# 101. Definition vs observation

Debe mantenerse:

```text
ConstraintDefinition
=
structural representation
```

```text
ObservedConstraint
=
structural representation
+
observation evidence
```

---

# 102. Completeness

Se podrá expresar:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 103. Property-level certainty

Una constraint introspectada puede tener:

```text
kind             EXACT
columns          EXACT
name             EXACT
deferrability    UNKNOWN
validation       UNKNOWN
enforcement      INFERRED
```

No debe perderse esa incertidumbre.

---

# 104. UNKNOWN ≠ default

Regla crítica:

```text
UNKNOWN
≠
PLATFORM_DEFAULT
```

---

# 105. UNKNOWN ≠ false

Nunca:

```php
$enforced = $metadata['enforced'] ?? false;
```

si ausencia significa desconocido.

---

# 106. Provenance

Debe poder rastrearse:

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

# 107. Native metadata preservation

Si la plataforma expone propiedades que VoltStack todavía no comprende completamente:

```text
NativeConstraintMetadata
```

deberá poder preservarlas.

---

# 108. Unknown feature policy

La política podrá ser:

```text
FAIL
PRESERVE_OPAQUE
IGNORE_WITH_DIAGNOSTIC
```

Nunca silent loss.

---

# 109. Comparison modes

Se requieren diferentes conceptos:

```text
EXACT
STRUCTURAL
SEMANTIC
NAME_INDEPENDENT
PLATFORM_NORMALIZED
MIGRATION
```

---

# 110. Exact equality

Puede incluir:

- identifier;
- metadata estructural;
- properties;
- extensions;
- canonical representation.

---

# 111. Structural equality

Puede ignorar metadata no estructural como:

```text
source file
line number
debug annotations
```

---

# 112. Semantic equality

Podrá considerar equivalencias conocidas y demostrables.

No deberá inventarlas por heurística insegura.

---

# 113. Platform-normalized equality

Ejemplo conceptual:

```text
UniqueConstraint
```

y una representación introspectada equivalente de plataforma pueden considerarse comparables mediante un perfil específico.

Eso no significa que:

```text
UniqueConstraint
=
UniqueIndex
```

globalmente.

---

# 114. Name-independent equality

Útil para rename detection.

Pero:

```text
NameIndependentEquivalent(A,B)
```

no implica:

```text
A was renamed to B
```

---

# 115. Fingerprinting

Se propone:

```text
ConstraintFingerprint
```

con perfiles:

```text
FULL
STRUCTURAL
SEMANTIC
NAME_INDEPENDENT
PORTABLE
MIGRATION
CACHE
```

---

# 116. Fingerprint determinism

Debe cumplirse:

```text
Fingerprint(
    Normalize(C),
    Profile,
    Version
)
```

es determinista.

---

# 117. Fingerprint ≠ identity proof

Debe mantenerse:

```text
same hash
⇒ candidate equality
```

No:

```text
same hash
⇒ absolute equality
```

---

# 118. SQL excluded from identity

Nunca:

```text
ConstraintFingerprint
=
hash(compiled SQL)
```

como contrato canónico.

---

# 119. Serialization

Las constraints deberán serializarse mediante formato:

```text
typed
versioned
deterministic
portable where possible
extension-aware
```

---

# 120. Example unique serialization

```json
{
  "kind": "unique",
  "name": "uq_users_email",
  "columns": [
    "email"
  ],
  "deferrability": "platform_default",
  "extensions": []
}
```

---

# 121. Example check serialization

```json
{
  "kind": "check",
  "name": "ck_products_price",
  "predicate": {
    "type": "binary",
    "operator": ">=",
    "left": {
      "type": "column",
      "name": "price"
    },
    "right": {
      "type": "literal",
      "value": 0
    }
  }
}
```

---

# 122. Serialization invariant

Debe cumplirse:

```text
StructuralEqual(
    C,
    Deserialize(Serialize(C))
)
```

para versiones compatibles.

---

# 123. No PHP serialize contract

No utilizar:

```php
serialize($constraint);
```

como formato persistente oficial.

---

# 124. Schema Diff integration

El Constraint System deberá alimentar:

```text
ConstraintDiff
```

---

# 125. Diff categories

```text
ConstraintAdded
ConstraintRemoved
ConstraintRenamed
ConstraintDefinitionChanged
ConstraintValidationChanged
ConstraintEnforcementChanged
ConstraintDeferrabilityChanged
ConstraintExtensionChanged
ConstraintChangeIndeterminate
```

---

# 126. Constraint type changes

Un cambio:

```text
UNIQUE(a)
```

a:

```text
PRIMARY KEY(a)
```

no deberá representarse como una simple property mutation.

Conceptualmente puede ser:

```text
DropConstraint
+
AddConstraint
```

según planificación.

---

# 127. Constraint replacement

Se podrá representar:

```text
ReplaceConstraintIntent
```

si el Schema AST necesita preservar que ambos estados están relacionados.

El planner decidirá operaciones físicas.

---

# 128. Constraint rename

Debe preservarse como intent cuando se conozca:

```text
RenameConstraint
```

No degradarlo inmediatamente a:

```text
DROP + ADD
```

---

# 129. Platform rename limitations

Si una plataforma no soporta rename directo:

```text
RenameConstraint
        │
        ▼
Schema Planner
        │
        ▼
Drop + Recreate
```

si es seguro y permitido.

---

# 130. Planner responsibility

El Schema Planner decidirá:

- ordering;
- temporary drops;
- recreation;
- table rebuild;
- deferred validation;
- capability-based emulation;
- dependencies.

ConstraintDefinition no lo decide.

---

# 131. Constraint dependencies

Las constraints introducen dependencias.

Ejemplos:

```text
PrimaryKey
    ↓
Columns
```

```text
UniqueConstraint
    ↓
Columns
```

```text
CheckConstraint
    ↓
Expression referenced columns/functions
```

```text
ForeignKey
    ↓
Local columns
Referenced table
Referenced columns
Referenced candidate key
```

---

# 132. Dependency graph

Podrá derivarse:

```text
ConstraintDependencyGraph
```

del Schema Model.

No será una segunda fuente mutable de verdad.

---

# 133. Drop dependency

Ejemplo:

```text
Column email
   ▲
   │
UNIQUE(email)
```

Antes de eliminar `email`, puede ser necesario:

```text
DROP UNIQUE
DROP COLUMN
```

---

# 134. Foreign key dependency

Ejemplo:

```text
customers.id
      ▲
      │
orders.customer_id FK
```

el planner deberá considerar esa dependencia antes de eliminar la target key.

---

# 135. Constraint dependency ≠ execution order

El grafo expresa restricciones.

El planner genera orden.

```text
Dependency Graph
≠
Execution Plan
```

---

# 136. Index interaction

Debe mantenerse:

```text
Constraint
≠
Index
```

pero existir una relación explícita:

```text
Constraint
        │
        ▼
Physical Support Requirement
        │
        ▼
Index Planner
```

---

# 137. SupportingIndexRequirement

Puede existir:

```text
SupportingIndexRequirement
├── NONE
├── REQUIRED
├── PLATFORM_MANAGED
├── REUSABLE_EXISTING
└── UNKNOWN
```

como resultado de planificación.

No necesariamente como propiedad permanente de la constraint.

---

# 138. Primary backing index

Si introspection revela que:

```text
PK users(id)
```

está físicamente respaldada por:

```text
PRIMARY index
```

podrá preservarse:

```text
ConstraintPhysicalSupportMetadata
```

sin fusionar objetos.

---

# 139. Unique backing index

Igualmente:

```text
UniqueConstraint
        │
        └── backed by
             UniqueIndex
```

puede ser una relación observada.

---

# 140. Check constraint and optimizer

Una check constraint trusted:

```text
CHECK(price >= 0)
```

puede producir un semantic fact:

```text
price >= 0
```

bajo condiciones seguras.

---

# 141. Constraint fact projection

Se propone:

```text
ConstraintSemanticFactProjector
```

---

# 142. Semantic fact ≠ constraint object

El Query Optimizer deberá consumir:

```text
SemanticConstraintFact
```

no depender directamente de todos los detalles del Schema Constraint implementation.

---

# 143. Fact evidence

Todo fact deberá conservar:

```text
source constraint
trust state
validation state
enforcement state
schema snapshot
certainty
```

---

# 144. Example optimizer rule

Dado:

```text
CHECK(quantity >= 0)
```

y query:

```text
WHERE quantity < 0
```

podría inferirse imposibilidad.

Pero sólo si:

```text
constraint trusted
AND
relevant SQL semantics proven
AND
snapshot applicable
```

---

# 145. Constraint ≠ authorization

Nunca utilizar:

```text
CHECK(tenant_id = ...)
```

como sustituto automático de authorization policy.

---

# 146. Constraint ≠ tenant security

Una constraint puede ayudar a integridad.

No representa por sí misma:

```text
current tenant access boundary
```

---

# 147. Multitenancy integration

El core no dependerá de Multitenancy.

Un paquete opcional podrá proyectar constraints para:

- tenant keys;
- composite tenant uniqueness;
- tenant-aware FKs;
- schema-per-tenant models.

---

# 148. Tenant uniqueness example

```text
UNIQUE(
    tenant_id,
    email
)
```

representa integridad estructural.

No implica que el sistema de Authorization permita acceder a todos esos registros.

---

# 149. Security boundaries

Identifiers y expressions deberán ser tipados.

Nunca usar concatenación arbitraria.

---

# 150. Check expression security

Input externo no deberá convertirse directamente en:

```text
RawSchemaExpression
```

---

# 151. Trust model

Para raw/extension expressions:

```text
SchemaExpressionTrust
├── FRAMEWORK
├── APPLICATION
├── TRUSTED_EXTENSION
├── EXTERNAL_UNTRUSTED
└── UNKNOWN
```

---

# 152. Trust never bypasses validation

Incluso:

```text
FRAMEWORK
```

no deberá significar:

```text
skip structural validation
```

---

# 153. Extension constraints

Debe existir:

```text
ExtensionConstraintDefinition
```

para conceptos que no pertenezcan al portable core.

---

# 154. Extension identity

Toda extension constraint deberá incluir:

```text
ExtensionId
ExtensionVersion
ConstraintTypeId
PayloadVersion
CapabilityRequirements
```

---

# 155. Extension collision

Dos extensiones no podrán registrar:

```text
same ConstraintTypeId
```

silenciosamente.

---

# 156. Frozen registry

Debe utilizarse:

```text
FrozenConstraintExtensionRegistry
```

después de bootstrap.

---

# 157. Extension restrictions

Una extension constraint no deberá:

- abrir conexiones;
- ejecutar SQL;
- modificar global state;
- acceder al request;
- bypass validation;
- bypass migration safety.

---

# 158. Unknown extension policy

Debe ser:

```text
FAIL
PRESERVE_OPAQUE
IGNORE_WITH_DIAGNOSTIC
```

según contexto.

---

# 159. Persistent runtime model

Objetos estructurales:

```text
ConstraintDefinition
ConstraintIdentifier
ConstraintMetadata
ConstraintFingerprint
```

deberán ser inmutables.

---

# 160. Operation-scoped state

Objetos como:

```text
ConstraintBuildSession
ConstraintValidationSession
ConstraintComparisonSession
ConstraintIntrospectionSession
```

serán operation-scoped.

---

# 161. Forbidden mutable singleton state

Nunca:

```php
ConstraintManager::$currentSchema;
ConstraintManager::$currentTenant;
ConstraintManager::$currentConnection;
```

---

# 162. Runtime compatibility

La arquitectura deberá funcionar correctamente bajo:

```text
PHP-FPM
FrankenPHP
RoadRunner
OpenSwoole
CLI
Testing
```

---

# 163. Resource independence

ConstraintDefinition no contendrá:

```text
PDO
PDOStatement
Connection
Transaction
Request
Response
Session
EntityManager
UnitOfWork
```

---

# 164. Performance model

Para `n` constraints:

```text
Build = O(n)
```

idealmente, con índices auxiliares para nombres e IDs.

---

# 165. Duplicate detection

Con mapas:

```text
Identifier → Constraint
ID → Constraint
Kind → ConstraintCollection
```

puede obtenerse detección aproximadamente:

```text
O(1)
```

promedio por inserción.

---

# 166. Constraint budgets

Se propone:

```php
final readonly class ConstraintBudget
{
    public function __construct(
        public int $maxConstraintsPerTable,
        public int $maxColumnsPerConstraint,
        public int $maxExpressionDepth,
        public int $maxExpressionNodes,
        public int $maxExtensions,
        public int $maxMetadataBytes,
    ) {}
}
```

---

# 167. Budget exhaustion

Debe producir:

```text
ConstraintBudgetExceededException
```

No truncar estructuras.

---

# 168. Expression complexity

Particularmente CHECK deberá limitar:

```text
depth
node count
raw payload size
extension payload size
```

para proteger tooling, serialization y analysis.

---

# 169. Error hierarchy

Se propone:

```text
DatabaseConstraintException
├── InvalidConstraintDefinitionException
├── InvalidConstraintIdentifierException
├── DuplicateConstraintIdentifierException
├── DuplicateConstraintDefinitionException
├── InvalidConstraintKindException
├── InvalidConstraintColumnSetException
├── UnknownConstraintColumnException
├── MultiplePrimaryKeyException
├── InvalidPrimaryKeyConstraintException
├── InvalidUniqueConstraintException
├── InvalidCheckConstraintException
├── InvalidCheckExpressionException
├── InvalidConstraintDeferrabilityException
├── InvalidConstraintValidationStateException
├── InvalidConstraintEnforcementStateException
├── InvalidConstraintTrustStateException
├── ConstraintCapabilityException
├── ConstraintDependencyException
├── ConstraintNormalizationException
├── ConstraintComparisonException
├── ConstraintSerializationException
├── ConstraintExtensionException
├── ConstraintExtensionConflictException
├── ConstraintBudgetExceededException
└── ConstraintInvariantException
```

Las excepciones especializadas de Foreign Key permanecen bajo su subsistema.

---

# 170. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Constraint
```

---

# 171. Proposed directory structure

```text
Schema/
└── Constraint/
    ├── Contract/
    │   ├── ConstraintDefinition.php
    │   ├── ConstraintDefinitionFactory.php
    │   ├── ConstraintDefinitionValidator.php
    │   ├── ConstraintDefinitionNormalizer.php
    │   ├── ConstraintDefinitionComparator.php
    │   └── ConstraintSemanticFactProjector.php
    │
    ├── Definition/
    │   ├── ConstraintDefinitionId.php
    │   ├── ConstraintKind.php
    │   ├── PrimaryKeyConstraintDefinition.php
    │   ├── UniqueConstraintDefinition.php
    │   ├── CheckConstraintDefinition.php
    │   └── ExtensionConstraintDefinition.php
    │
    ├── Collection/
    │   ├── ConstraintDefinitionSet.php
    │   ├── PrimaryKeyConstraintView.php
    │   ├── UniqueConstraintView.php
    │   └── CheckConstraintView.php
    │
    ├── Identifier/
    │   ├── ConstraintIdentifier.php
    │   └── ConstraintNameOrigin.php
    │
    ├── Column/
    │   └── OrderedColumnReferenceSet.php
    │
    ├── Expression/
    │   ├── SchemaExpression.php
    │   ├── SchemaExpressionTrust.php
    │   └── RawSchemaExpression.php
    │
    ├── Deferrability/
    │   └── ConstraintDeferrability.php
    │
    ├── Enforcement/
    │   └── ConstraintEnforcementState.php
    │
    ├── ValidationState/
    │   └── ConstraintValidationState.php
    │
    ├── Trust/
    │   └── ConstraintTrustState.php
    │
    ├── Metadata/
    │   ├── ConstraintMetadata.php
    │   ├── ConstraintObservationMetadata.php
    │   ├── NativeConstraintMetadata.php
    │   └── ObservedConstraint.php
    │
    ├── Capability/
    │   ├── ConstraintCapabilityRequirementSet.php
    │   └── ConstraintCapabilityRequirementResolver.php
    │
    ├── Dependency/
    │   ├── ConstraintDependency.php
    │   └── ConstraintDependencyGraph.php
    │
    ├── PhysicalSupport/
    │   ├── SupportingIndexRequirement.php
    │   └── ConstraintPhysicalSupportMetadata.php
    │
    ├── Semantic/
    │   ├── SemanticConstraintFact.php
    │   └── ConstraintSemanticFactEvidence.php
    │
    ├── Validation/
    │   ├── ConstraintValidationContext.php
    │   ├── ConstraintValidationResult.php
    │   ├── PrimaryKeyConstraintValidator.php
    │   ├── UniqueConstraintValidator.php
    │   └── CheckConstraintValidator.php
    │
    ├── Normalization/
    │   └── ConstraintNormalizationContext.php
    │
    ├── Comparison/
    │   ├── ConstraintComparisonProfile.php
    │   └── ConstraintComparisonResult.php
    │
    ├── Fingerprint/
    │   ├── ConstraintFingerprint.php
    │   ├── ConstraintFingerprintProfile.php
    │   └── ConstraintFingerprintGenerator.php
    │
    ├── Serialization/
    │   ├── ConstraintSerializer.php
    │   └── ConstraintDeserializer.php
    │
    ├── Extension/
    │   ├── ConstraintExtension.php
    │   ├── ConstraintExtensionTypeId.php
    │   └── FrozenConstraintExtensionRegistry.php
    │
    ├── Budget/
    │   └── ConstraintBudget.php
    │
    └── Exception/
        ├── DatabaseConstraintException.php
        ├── InvalidConstraintDefinitionException.php
        ├── InvalidConstraintIdentifierException.php
        ├── DuplicateConstraintIdentifierException.php
        ├── DuplicateConstraintDefinitionException.php
        ├── InvalidConstraintColumnSetException.php
        ├── UnknownConstraintColumnException.php
        ├── MultiplePrimaryKeyException.php
        ├── InvalidPrimaryKeyConstraintException.php
        ├── InvalidUniqueConstraintException.php
        ├── InvalidCheckConstraintException.php
        ├── InvalidCheckExpressionException.php
        ├── ConstraintCapabilityException.php
        ├── ConstraintDependencyException.php
        ├── ConstraintSerializationException.php
        ├── ConstraintExtensionException.php
        ├── ConstraintExtensionConflictException.php
        ├── ConstraintBudgetExceededException.php
        └── ConstraintInvariantException.php
```

El Foreign Key System permanecerá en:

```text
Schema/ForeignKey/
```

pero implementará el contrato correspondiente del Constraint System.

---

# 172. Dependency rules

Permitido:

```text
Constraint
   ↓
Schema Identifiers
Schema Column References
Schema Expressions
Schema Capability Contracts
Immutable Metadata
Extension Contracts
```

No permitido:

```text
Constraint
   ↓
PDO
Connection
Driver
EntityManager
UnitOfWork
HTTP Request
Migration Runner
Query Executor
Mutable Tenant Context
```

---

# 173. Builder integration

El Schema Builder podrá ofrecer:

```php
$table->primary('id');

$table->unique('email');

$table->unique([
    'tenant_id',
    'email',
]);

$table->check(
    SchemaExpression::column('price')
        ->greaterThanOrEqual(0)
);
```

---

# 174. Builder sugar

También podrá ofrecer:

```php
$table->id();
```

que produzca:

```text
ColumnDefinition(id)
+
PrimaryKeyConstraintDefinition(id)
```

según la recipe configurada.

---

# 175. Sugar ≠ canonical semantics

`id()` será:

```text
Builder Recipe
```

No:

```text
ConstraintKind::ID
```

---

# 176. Column unique modifier

Una API:

```php
$table->string('email')->unique();
```

deberá producir conceptualmente:

```text
ColumnDefinition(email)
+
UniqueConstraintDefinition(email)
```

o un `UniqueIndexDefinition` sólo cuando la API lo solicite explícitamente.

No fusionar ambos.

---

# 177. Column primary modifier

Igualmente:

```php
$table->integer('id')->primary();
```

deberá terminar en:

```text
PrimaryKeyConstraintDefinition
```

y no en una propiedad arbitraria escondida dentro del tipo de columna.

---

# 178. Check builder

Preferido:

```php
$table->check(
    fn (SchemaExpressionBuilder $expr) =>
        $expr->column('price')->gte(0)
);
```

que termine en AST estructurado.

---

# 179. Raw check escape hatch

Explícito:

```php
$table->rawCheck(
    RawSchemaExpression::trusted(
        'price >= 0'
    )
);
```

Debe quedar marcado como:

```text
raw
trusted
potentially nonportable
```

---

# 180. Schema AST integration

Las operaciones serán:

```text
AddConstraintNode
DropConstraintNode
RenameConstraintNode
ValidateConstraintNode
```

y operaciones especializadas cuando sean necesarias.

---

# 181. AST definition reuse

`AddConstraintNode` podrá contener:

```text
ConstraintDefinition
```

inmutable.

---

# 182. Drop constraint

Deberá usar referencia estructurada:

```text
ConstraintReference
```

No SQL:

```text
"DROP CONSTRAINT ..."
```

---

# 183. Constraint reference

Puede ser:

```text
ConstraintReference
├── ByIdentifier
├── ByDefinitionId
├── ByStructuralSignature
└── ResolvedConstraintReference
```

según contexto.

---

# 184. Ambiguous drop

Una referencia estructural ambigua deberá fallar.

Nunca eliminar "la primera que coincida".

---

# 185. Schema compiler integration

```text
ConstraintDefinition
        │
        ▼
Schema Change Plan
        │
        ▼
Platform Schema Compiler
        │
        ▼
DDL representation
```

El compiler podrá decidir sintaxis.

No semántica.

---

# 186. Inline vs table-level compilation

Una plataforma podrá renderizar:

```text
column definition inline constraint
```

o:

```text
table-level constraint
```

si ambas son semánticamente equivalentes.

Esto pertenece al compiler.

---

# 187. Compiler cannot weaken constraints

Si una plataforma no soporta:

```text
DEFERRABLE
```

el compiler no deberá simplemente omitirlo.

Debe producirse:

```text
capability/planning failure
```

o estrategia explícita aprobada.

---

# 188. Constraint execution

Execution Engine recibe comandos compilados.

No necesita interpretar:

```text
UniqueConstraintDefinition
```

ni:

```text
CheckConstraintDefinition
```

---

# 189. Testing strategy

El sistema deberá probarse principalmente mediante pure tests.

Categorías:

- construction;
- invariants;
- normalization;
- comparison;
- fingerprint;
- serialization;
- expression validation;
- dependency extraction;
- capability validation;
- introspection;
- diff;
- extension conformance;
- persistent runtime isolation.

---

# 190. Primary key tests

Verificar:

```text
PRIMARY KEY(id)
PRIMARY KEY(tenant_id,id)
```

y errores:

```text
PRIMARY KEY()
PRIMARY KEY(id,id)
multiple PKs
unknown columns
```

---

# 191. Unique constraint tests

Verificar:

```text
UNIQUE(email)
UNIQUE(tenant_id,email)
```

incluyendo:

```text
NULL semantics
deferrability
generated names
```

---

# 192. Check constraint tests

Verificar:

```text
CHECK(price >= 0)
```

con:

- typed AST;
- referenced columns;
- unknown columns;
- raw expressions;
- unsupported functions;
- depth budgets.

---

# 193. Foreign key conformance

El contrato general deberá reutilizar las pruebas del documento 94 para:

```text
kind
identity
metadata
capabilities
serialization
comparison
```

sin duplicar sus pruebas especializadas.

---

# 194. Cross-platform tests

Ejecutar matrices sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

verificando capacidades reales del adapter correspondiente.

---

# 195. Round-trip tests

Idealmente:

```text
Definition
    ↓
Compile
    ↓
Execute
    ↓
Introspect
    ↓
ObservedDefinition
```

y comparar mediante:

```text
PlatformNormalizedComparison
```

---

# 196. Persistent runtime tests

Simular:

```text
Request A → Schema A
Request B → Schema B
Request C → Schema C
```

sobre el mismo worker.

No deberá existir leakage de:

```text
constraints
connection
tenant
schema
validation state
extension session
```

---

# 197. Architectural invariants

## DB-CONSTRAINT-001
Toda constraint tendrá identidad estructural interna.

## DB-CONSTRAINT-002
ConstraintDefinition será inmutable.

## DB-CONSTRAINT-003
ConstraintDefinition será distinta de ConstraintBlueprint.

## DB-CONSTRAINT-004
ConstraintDefinition será distinta de Schema AST operation.

## DB-CONSTRAINT-005
ConstraintDefinition será distinta de SQL.

## DB-CONSTRAINT-006
Constraint System no generará SQL.

## DB-CONSTRAINT-007
Constraint System no ejecutará DDL.

## DB-CONSTRAINT-008
Constraint System no abrirá conexiones.

## DB-CONSTRAINT-009
Constraint será distinta de Index.

## DB-CONSTRAINT-010
Constraint será distinta de application validation.

## DB-CONSTRAINT-011
Constraint será distinta de ORM validation.

## DB-CONSTRAINT-012
Constraint será distinta de authorization policy.

## DB-CONSTRAINT-013
Constraint será distinta de business rule.

## DB-CONSTRAINT-014
Constraint será distinta de trigger.

## DB-CONSTRAINT-015
PrimaryKeyConstraint será distinta de IndexDefinition.

## DB-CONSTRAINT-016
UniqueConstraint será distinta de UniqueIndex.

## DB-CONSTRAINT-017
CheckConstraint utilizará SchemaExpression.

## DB-CONSTRAINT-018
ForeignKeyDefinition será una única representación canónica.

## DB-CONSTRAINT-019
No se duplicará ForeignKeyDefinition como ForeignKeyConstraintDefinition equivalente.

## DB-CONSTRAINT-020
ConstraintDefinitionId será distinto de ConstraintIdentifier.

## DB-CONSTRAINT-021
Constraint identifiers serán tipados.

## DB-CONSTRAINT-022
Constraint identifiers no contendrán SQL quoting.

## DB-CONSTRAINT-023
Identifier rendering pertenecerá al compiler.

## DB-CONSTRAINT-024
Generated names serán deterministas.

## DB-CONSTRAINT-025
Naming strategy será versionable.

## DB-CONSTRAINT-026
Naming collisions serán detectadas.

## DB-CONSTRAINT-027
No habrá first-wins collision handling.

## DB-CONSTRAINT-028
No habrá last-wins collision handling.

## DB-CONSTRAINT-029
SQL placement no determinará semantic scope.

## DB-CONSTRAINT-030
Primary key tendrá al menos una columna.

## DB-CONSTRAINT-031
Primary key no contendrá duplicate columns.

## DB-CONSTRAINT-032
Table tendrá como máximo una PK portable.

## DB-CONSTRAINT-033
Composite PK preservará column order.

## DB-CONSTRAINT-034
Unique constraint tendrá al menos una columna.

## DB-CONSTRAINT-035
Composite unique preservará column order.

## DB-CONSTRAINT-036
Unique NULL semantics serán platform-aware.

## DB-CONSTRAINT-037
NULLS_DISTINCT será distinto de NULLS_NOT_DISTINCT.

## DB-CONSTRAINT-038
PLATFORM_DEFAULT será distinto de UNKNOWN.

## DB-CONSTRAINT-039
Check expression será estructurada.

## DB-CONSTRAINT-040
SchemaExpression será distinta de QueryExpression.

## DB-CONSTRAINT-041
Raw expression será escape hatch explícito.

## DB-CONSTRAINT-042
Raw expression conservará trust metadata.

## DB-CONSTRAINT-043
Raw expression conservará portability metadata.

## DB-CONSTRAINT-044
Check evaluation no asumirá PHP boolean semantics.

## DB-CONSTRAINT-045
SQL three-valued logic será preservada donde aplique.

## DB-CONSTRAINT-046
Enforcement será distinto de validation.

## DB-CONSTRAINT-047
Validation será distinta de trust.

## DB-CONSTRAINT-048
Enforcement será distinto de trust.

## DB-CONSTRAINT-049
Structural validity será distinta de data validation state.

## DB-CONSTRAINT-050
Constraint existence no implicará trusted constraint.

## DB-CONSTRAINT-051
Constraint existence no implicará enforced constraint.

## DB-CONSTRAINT-052
Constraint existence no implicará validated constraint.

## DB-CONSTRAINT-053
Deferrability será distinta de transaction runtime mode.

## DB-CONSTRAINT-054
Capability requirements serán declarativos.

## DB-CONSTRAINT-055
Version checks no reemplazarán capability checks.

## DB-CONSTRAINT-056
Unsupported constraint será distinta de invalid constraint.

## DB-CONSTRAINT-057
Unsupported features no se degradarán silenciosamente.

## DB-CONSTRAINT-058
MySQL semantics estarán encapsuladas.

## DB-CONSTRAINT-059
MariaDB será first-class platform.

## DB-CONSTRAINT-060
PostgreSQL semantics estarán capability-driven.

## DB-CONSTRAINT-061
SQLite limitations serán explícitas.

## DB-CONSTRAINT-062
ConstraintDefinitionSet será inmutable.

## DB-CONSTRAINT-063
Constraint iteration será determinista.

## DB-CONSTRAINT-064
Constraint iteration order no será execution order.

## DB-CONSTRAINT-065
Duplicate IDs serán inválidos.

## DB-CONSTRAINT-066
Duplicate explicit identifiers serán inválidos dentro del mismo namespace aplicable.

## DB-CONSTRAINT-067
Structural duplicate detection será explícita.

## DB-CONSTRAINT-068
Structural duplicates no se eliminarán silenciosamente.

## DB-CONSTRAINT-069
Normalization será determinista.

## DB-CONSTRAINT-070
Normalization será idempotente.

## DB-CONSTRAINT-071
Normalization será semantics-preserving.

## DB-CONSTRAINT-072
Normalization no realizará DB I/O oculto.

## DB-CONSTRAINT-073
Composite column order será preservado.

## DB-CONSTRAINT-074
Check normalization preservará 3VL semantics.

## DB-CONSTRAINT-075
Validation tendrá capas separadas.

## DB-CONSTRAINT-076
Local validation será posible sin DB.

## DB-CONSTRAINT-077
Platform validation utilizará capability snapshot.

## DB-CONSTRAINT-078
Validation diagnostics serán estructurados.

## DB-CONSTRAINT-079
ObservedConstraint será distinto de ConstraintDefinition.

## DB-CONSTRAINT-080
Introspection uncertainty será explícita.

## DB-CONSTRAINT-081
UNKNOWN no será convertido a false.

## DB-CONSTRAINT-082
UNKNOWN no será convertido a PLATFORM_DEFAULT.

## DB-CONSTRAINT-083
Observation provenance será preservado.

## DB-CONSTRAINT-084
Unknown native metadata no se perderá silenciosamente.

## DB-CONSTRAINT-085
Comparison tendrá perfiles explícitos.

## DB-CONSTRAINT-086
Exact equality será distinta de structural equality.

## DB-CONSTRAINT-087
Structural equality será distinta de semantic equality.

## DB-CONSTRAINT-088
Semantic equality será distinta de platform-normalized equality.

## DB-CONSTRAINT-089
Name-independent equality no probará rename.

## DB-CONSTRAINT-090
Fingerprint será determinista.

## DB-CONSTRAINT-091
Fingerprint será versionado.

## DB-CONSTRAINT-092
Fingerprint no será SQL hash.

## DB-CONSTRAINT-093
Fingerprint collision no probará igualdad absoluta.

## DB-CONSTRAINT-094
Serialization será versionada.

## DB-CONSTRAINT-095
Serialization será determinista.

## DB-CONSTRAINT-096
Serialization preservará structural extensions.

## DB-CONSTRAINT-097
Serialization no contendrá live resources.

## DB-CONSTRAINT-098
Schema Diff preservará uncertainty.

## DB-CONSTRAINT-099
Rename intent será distinto de drop/create.

## DB-CONSTRAINT-100
Compiler no ocultará semantic weakening.

## DB-CONSTRAINT-101
Planner decidirá physical alteration strategy.

## DB-CONSTRAINT-102
Planner decidirá table rebuild strategy.

## DB-CONSTRAINT-103
ConstraintDefinition no decidirá transaction boundaries.

## DB-CONSTRAINT-104
ConstraintDefinition no decidirá migration safety.

## DB-CONSTRAINT-105
ConstraintDefinition no decidirá zero-downtime strategy.

## DB-CONSTRAINT-106
Constraint dependencies serán derivables.

## DB-CONSTRAINT-107
Dependency graph no será segunda mutable source of truth.

## DB-CONSTRAINT-108
Constraint dependency será distinta de execution order.

## DB-CONSTRAINT-109
Supporting index requirement será distinto de ConstraintDefinition.

## DB-CONSTRAINT-110
Physical backing index no fusionará Constraint e Index.

## DB-CONSTRAINT-111
Semantic optimizer facts serán proyecciones.

## DB-CONSTRAINT-112
Optimizer no dependerá de untrusted constraints como proof.

## DB-CONSTRAINT-113
Optimizer considerará validation state.

## DB-CONSTRAINT-114
Optimizer considerará enforcement state.

## DB-CONSTRAINT-115
Optimizer considerará snapshot applicability.

## DB-CONSTRAINT-116
Constraint no representará authorization.

## DB-CONSTRAINT-117
Constraint no representará tenant access policy.

## DB-CONSTRAINT-118
Multitenancy será integración opcional.

## DB-CONSTRAINT-119
Tenant constraints no otorgarán acceso.

## DB-CONSTRAINT-120
Identifiers serán validados antes de compilation.

## DB-CONSTRAINT-121
Raw expressions tendrán trust boundary.

## DB-CONSTRAINT-122
Trust no deshabilitará structural validation.

## DB-CONSTRAINT-123
Extension constraints serán tipadas.

## DB-CONSTRAINT-124
Extension constraints tendrán stable IDs.

## DB-CONSTRAINT-125
Extension registry será frozen.

## DB-CONSTRAINT-126
Extension conflicts serán explícitos.

## DB-CONSTRAINT-127
Unknown structural extensions no serán ignoradas silenciosamente.

## DB-CONSTRAINT-128
Extensions no ejecutarán SQL.

## DB-CONSTRAINT-129
Extensions no abrirán conexiones.

## DB-CONSTRAINT-130
Extensions no accederán a mutable request state.

## DB-CONSTRAINT-131
Constraint structural objects serán persistent-runtime safe.

## DB-CONSTRAINT-132
Mutable sessions serán operation-scoped.

## DB-CONSTRAINT-133
No existirá mutable global current schema state.

## DB-CONSTRAINT-134
No existirá mutable global current tenant state.

## DB-CONSTRAINT-135
No existirá mutable global current connection state.

## DB-CONSTRAINT-136
ConstraintDefinition no contendrá PDO.

## DB-CONSTRAINT-137
ConstraintDefinition no contendrá Connection.

## DB-CONSTRAINT-138
ConstraintDefinition no contendrá EntityManager.

## DB-CONSTRAINT-139
ConstraintDefinition no contendrá UnitOfWork.

## DB-CONSTRAINT-140
ConstraintDefinition no contendrá Request.

## DB-CONSTRAINT-141
Constraint budgets serán explícitos.

## DB-CONSTRAINT-142
Budget overflow fallará explícitamente.

## DB-CONSTRAINT-143
Budget overflow no truncará constraints.

## DB-CONSTRAINT-144
Builder sugar producirá canonical definitions.

## DB-CONSTRAINT-145
Column `unique()` no fusionará automáticamente UniqueConstraint y UniqueIndex.

## DB-CONSTRAINT-146
Column `primary()` producirá PrimaryKeyConstraintDefinition.

## DB-CONSTRAINT-147
Schema AST reutilizará immutable definitions.

## DB-CONSTRAINT-148
Execution Engine no interpretará constraints de alto nivel.

## DB-CONSTRAINT-149
Constraint System será runtime-neutral.

## DB-CONSTRAINT-150
ConstraintDefinition será la representación canónica de una regla estructural de integridad de base de datos.

---

# 198. Anti-patterns

## 198.1 Constraint como string SQL

Incorrecto:

```php
$constraint = 'UNIQUE(email)';
```

Correcto:

```text
UniqueConstraintDefinition
└── ColumnReference(email)
```

---

## 198.2 Unique index como unique constraint

Incorrecto:

```text
UniqueConstraint
=
UniqueIndex
```

---

## 198.3 Constraint como validator PHP

Incorrecto:

```php
$constraint = new MinValidator(18);
```

---

## 198.4 Check como closure PHP

Incorrecto:

```php
$table->check(
    fn ($row) => $row['price'] >= 0
);
```

Una closure PHP no representa semántica ejecutable por la base de datos.

---

## 198.5 God Constraint

Incorrecto:

```php
new Constraint(
    primary: true,
    unique: true,
    foreign: true,
    check: '...',
    references: '...',
);
```

---

## 198.6 Nullable properties everywhere

Incorrecto:

```php
$constraint->onDelete = null;
$constraint->checkExpression = null;
$constraint->referencedTable = null;
```

para tipos donde esas propiedades ni siquiera aplican.

---

## 198.7 Hidden index creation

Incorrecto:

```php
$constraint->createSupportingIndex();
```

---

## 198.8 Hidden platform downgrade

Incorrecto:

```text
DEFERRABLE requested
+
platform unsupported
        ↓
compiler omits DEFERRABLE
```

---

## 198.9 UNKNOWN collapse

Incorrecto:

```text
UNKNOWN enforcement
→
NOT ENFORCED
```

---

## 198.10 Optimizer trusts everything

Incorrecto:

```text
Constraint exists
→
optimizer assumes fact true
```

---

# 199. Ejemplo integrado

Definición conceptual:

```php
$table = TableDefinition::create('orders')
    ->withColumn(
        ColumnDefinition::integer('id')
    )
    ->withColumn(
        ColumnDefinition::decimal('total', 18, 2)
    )
    ->withColumn(
        ColumnDefinition::integer('customer_id')
    )
    ->withConstraint(
        PrimaryKeyConstraintDefinition::columns('id')
    )
    ->withConstraint(
        CheckConstraintDefinition::expression(
            SchemaExpression::column('total')
                ->greaterThanOrEqual(0)
        )
    )
    ->withConstraint(
        ForeignKeyDefinition::references(
            local: ['customer_id'],
            table: 'customers',
            columns: ['id'],
        )
    );
```

Estructura:

```text
TableDefinition(orders)
│
├── Columns
│   ├── id
│   ├── total
│   └── customer_id
│
└── Constraints
    ├── PrimaryKey
    │   └── id
    │
    ├── Check
    │   └── total >= 0
    │
    └── ForeignKey
        └── customer_id → customers.id
```

---

# 200. Ejemplo de constraint graph

```text
Table: orders
│
├── Column(id)
│      ▲
│      │
│   PK(id)
│
├── Column(total)
│      ▲
│      │
│   CHECK(total >= 0)
│
└── Column(customer_id)
       ▲
       │
       └──── FK ───────────► customers.id
```

Este grafo podrá alimentar posteriormente:

```text
Schema Diff
Schema Planner
Migration Safety
Semantic Query Engine
ORM Metadata
Diagnostics
```

sin fusionar responsabilidades.

---

# 201. Fórmula maestra

```text
Database Constraint System
=
Typed Constraint Identity
+
Constraint Specialization
+
Primary Key Semantics
+
Unique Semantics
+
Check Expressions
+
Foreign Key Integration
+
Enforcement State
+
Validation State
+
Trust State
+
Deferrability
+
Capability Requirements
+
Dependency Modeling
+
Metadata
+
Provenance
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
Introspection
+
Extension Safety
+
Persistent Runtime Isolation
```

---

# 202. Fórmula de validez

Para constraint `C`:

```text
ValidConstraint(C)
=
ValidIdentity(C)
∧
ValidKind(C)
∧
ValidStructure(C)
∧
ValidReferences(C)
∧
ValidProperties(C)
∧
ValidExpressions(C)
∧
ValidExtensions(C)
```

Para plataforma `P`:

```text
Compilable(C,P)
=
ValidConstraint(C)
∧
Capabilities(P) ⊇ Requirements(C)
```

---

# 203. Fórmula de confianza semántica

Para utilizar una constraint como evidencia:

```text
UsableAsSemanticProof(C)
=
StructurallyValid(C)
∧
Trusted(C)
∧
ApplicableSnapshot(C)
∧
RequiredValidationSatisfied(C)
∧
RequiredEnforcementSatisfied(C)
∧
SemanticConditionsSatisfied(C)
```

No:

```text
Exists(C)
⇒
UsableAsSemanticProof(C)
```

---

# 204. Arquitectura final

```text
                      ConstraintDefinition
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
   Primary / Unique         Check             ForeignKey
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                              ▼
                       TableDefinition
                              │
                              ▼
                         Schema Model
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
       Schema Diff     Dependency Graph   Semantic Facts
             │                │                │
             └──────────┬─────┘                │
                        ▼                      ▼
                 Schema Planner          Query/ORM
                        │                 Consumers
                        ▼
                 Schema Compiler
                        │
                        ▼
                 Execution Engine
```

---

# 205. Regla arquitectónica final

> **Una database constraint en VoltStack representa una regla declarativa de integridad estructural. La constraint describe qué estados son válidos; el Schema Planner decide cómo introducir o modificar esa regla; el Schema Compiler decide cómo representarla para una plataforma; y el Execution Engine ejecuta los comandos resultantes.**

Por tanto:

```text
Constraint describes integrity.
Index describes access structure.
Validator describes application validation.
ORM describes object persistence.
Authorization describes access rights.
Planner describes structural execution strategy.
Compiler describes platform representation.
Executor performs execution.
```

---

# 206. Resultado arquitectónico

Con este sistema VoltStack podrá representar:

```text
PRIMARY KEY
UNIQUE
CHECK
FOREIGN KEY
Extension Constraints
```

bajo una arquitectura común sin perder las diferencias semánticas de cada tipo.

La relación final será:

```text
ConstraintDefinition
        │
        ├── identity
        ├── structural semantics
        ├── capabilities
        ├── metadata
        ├── trust
        └── extensions
                │
                ▼
          Schema Model
                │
                ▼
      Schema Evolution Pipeline
```

manteniendo el principio:

```text
Integrity Rule
≠
Physical Implementation
```

---

# 207. Siguiente documento

```text
96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack observa una base de datos existente y transforma metadata nativa de MySQL, MariaDB, PostgreSQL y SQLite en un modelo estructural normalizado:

```text
Database
    │
    ▼
Platform Catalog APIs
    │
    ▼
Schema Introspector
    │
    ▼
Native Metadata
    │
    ▼
Metadata Normalization
    │
    ▼
Observed Schema
    │
    ▼
DatabaseSchema / SchemaSnapshot
```

Deberá distinguir especialmente:

```text
Introspection
≠
Schema Definition
≠
Schema Builder
≠
Schema Diff
≠
Migration
```

y formalizar:

```text
Observed
≠
Declared

Unknown
≠
Absent

Partial Snapshot
≠
Complete Schema

Native Metadata
≠
Canonical Schema Model
```

Principio del siguiente documento:

> **El Schema Introspection System observa y normaliza la estructura existente de una base de datos; nunca debe inventar certeza que el motor o el driver no pudieron proporcionar.**