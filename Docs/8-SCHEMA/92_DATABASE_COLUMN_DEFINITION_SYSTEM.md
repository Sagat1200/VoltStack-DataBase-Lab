# 92_DATABASE_COLUMN_DEFINITION_SYSTEM.md

# VoltStack Quantum Database
## Database Column Definition System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 92 — Database Column Definition System  
**Bloque:** 8 — Schema  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Column Definition System` define la representación estructural, tipada, inmutable y portable de una columna dentro del Schema System de VoltStack.

Su pregunta central es:

> **¿Cómo representa VoltStack una columna sin convertirla prematuramente en SQL, sin acoplarla a un motor concreto y sin mezclarla con propiedades del ORM?**

Conceptualmente:

```text
ColumnDefinition
├── ColumnIdentifier
├── DatabaseType
├── Nullability
├── DefaultDefinition
├── GenerationStrategy
├── GeneratedExpression
├── IdentityDefinition
├── CharacterSet
├── Collation
├── Comment
├── ColumnOptionSet
├── CapabilityRequirements
├── Metadata
└── ExtensionMetadata
```

La regla fundamental será:

```text
ColumnDefinition
≠
ColumnBlueprint
≠
ColumnAlteration
≠
ObservedColumn
≠
ORM Property
≠
SQL Column Fragment
```

---

# 2. Principio central

Una columna deberá modelarse por su significado estructural.

No por la sintaxis de un proveedor.

Incorrecto:

```php
$type = 'BIGINT UNSIGNED AUTO_INCREMENT';
```

Correcto:

```text
ColumnDefinition
├── type
│   └── BigIntegerType
├── unsigned
│   └── capability/option
├── nullability
│   └── NOT_NULL
└── generation
    └── IDENTITY
```

Posteriormente:

```text
ColumnDefinition
      ↓
Schema Compiler
      ↓
Platform SQL
```

Por ejemplo:

```text
Canonical BigInteger + Identity
          │
          ├── MySQL ──────► BIGINT AUTO_INCREMENT
          ├── MariaDB ────► BIGINT AUTO_INCREMENT
          ├── PostgreSQL ─► BIGINT GENERATED ... AS IDENTITY
          └── SQLite ─────► INTEGER PRIMARY KEY / strategy
```

cuando las reglas semánticas y capabilities lo permitan.

---

# 3. Posición arquitectónica

```text
Developer DSL
     │
     ▼
ColumnBlueprint
     │
     │ lowering
     ▼
ColumnDefinition
     │
     ▼
TableDefinition
     │
     ▼
Schema Model / Schema AST
     │
     ▼
Schema Validation
     │
     ▼
Schema Planner
     │
     ▼
Schema Compiler
     │
     ▼
Execution Engine
```

El camino inverso de introspection será:

```text
Database
   │
   ▼
Schema Introspector
   │
   ▼
ObservedColumn
   ├── ColumnDefinition
   └── ObservationMetadata
```

---

# 4. Objetivos

El sistema deberá proporcionar:

- identidad tipada de columna;
- tipos de datos estructurados;
- nullability explícita;
- defaults tipados;
- generated columns;
- identity/autoincrement semantics;
- charset y collation cuando sean relevantes;
- comments;
- opciones tipadas;
- capability requirements;
- metadata;
- provenance;
- extensibilidad;
- validación;
- comparación;
- fingerprinting;
- serialización determinista;
- integración con Schema Diff;
- independencia del SQL;
- independencia del driver;
- seguridad para persistent runtimes.

---

# 5. No objetivos

`ColumnDefinition` no deberá:

- generar SQL;
- ejecutar SQL;
- abrir conexiones;
- consultar metadata;
- modificar tablas;
- decidir migrations;
- resolver ALTER COLUMN;
- decidir estrategias de table rebuild;
- representar propiedades ORM;
- realizar casting de entidades;
- contener valores de filas;
- realizar parameter binding;
- depender de PDO;
- conocer el request actual;
- resolver automáticamente el tenant.

---

# 6. Modelo principal

Se propone:

```php
final readonly class ColumnDefinition
{
    public function __construct(
        public ColumnIdentifier $identifier,
        public DatabaseType $type,
        public Nullability $nullability,
        public DefaultDefinition $default,
        public GenerationDefinition $generation,
        public ?CharacterSetDefinition $characterSet,
        public ?CollationDefinition $collation,
        public ?ColumnComment $comment,
        public ColumnOptionSet $options,
        public SchemaCapabilityRequirementSet $capabilities,
        public ColumnMetadata $metadata,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

La API concreta puede evolucionar.

La separación conceptual deberá mantenerse.

---

# 7. Inmutabilidad

`ColumnDefinition` deberá ser inmutable.

Incorrecto:

```php
$column->nullable = true;
```

Incorrecto:

```php
$column->type = 'varchar';
```

Una transformación deberá producir:

```text
ColumnDefinition₀
      │
      ▼
ColumnDefinition₁
```

---

# 8. ColumnDefinition ≠ ColumnBlueprint

La DSL puede permitir:

```php
$table
    ->string('email', 320)
    ->nullable()
    ->unique();
```

El `ColumnBlueprint` es una construcción mutable.

Después del lowering:

```text
ColumnDefinition(email)
├── type
│   └── StringType(320)
└── nullability
    └── NULLABLE

UniqueConstraintDefinition
└── email
```

La propiedad `unique()` no necesita permanecer como un simple boolean dentro de la columna.

---

# 9. ColumnDefinition ≠ ColumnAlteration

Una definición representa:

```text
state
```

Una alteración representa:

```text
transition
```

Por ejemplo:

```text
ColumnDefinition(name VARCHAR(100))
```

y:

```text
ColumnDefinition(name VARCHAR(200))
```

no dicen por sí solos:

```text
ALTER COLUMN name TYPE VARCHAR(200)
```

El cambio pertenece al Schema AST / Schema Diff / Migration layers.

---

# 10. ColumnDefinition ≠ ObservedColumn

Una columna introspectada necesita información adicional:

```text
ObservedColumn
├── ColumnDefinition
├── MetadataCertainty
├── DefinitionCompleteness
├── PlatformReportedType
├── PlatformDefaultExpression?
└── ObservationMetadata
```

No debe contaminarse el value object estructural con estado operacional.

---

# 11. ColumnDefinition ≠ ORM Property

Esto:

```php
final class User
{
    public string $email;
}
```

representa una propiedad de entidad.

Esto:

```text
users.email VARCHAR(320)
```

representa una columna.

La relación:

```text
ORM Property Metadata
        │
        ▼
ColumnIdentifier
```

puede existir.

La dependencia inversa no.

---

# 12. ColumnDefinition ≠ SQL fragment

Nunca almacenar como representación primaria:

```php
'email VARCHAR(320) NOT NULL DEFAULT \'unknown\''
```

La representación deberá estar descompuesta semánticamente.

---

# 13. ColumnIdentifier

Toda columna tendrá:

```text
ColumnIdentifier
```

Ejemplo:

```php
ColumnIdentifier::from('email');
```

---

# 14. Identifier ≠ raw SQL identifier

`ColumnIdentifier` representa identidad.

El quoting pertenece al compiler.

Por tanto:

```text
email
```

no deberá convertirse prematuramente en:

```text
`email`
```

o:

```text
"email"
```

---

# 15. Structured identity

Se recomienda:

```php
final readonly class ColumnIdentifier
{
    public function __construct(
        public string $value,
    ) {}
}
```

con validación centralizada.

---

# 16. Qualified column identity

Cuando se necesite referenciar una columna externamente:

```text
QualifiedColumnIdentifier
├── TableIdentifier
└── ColumnIdentifier
```

pero dentro de `TableDefinition` normalmente basta:

```text
ColumnIdentifier
```

---

# 17. ColumnDefinitionId

Podrá existir:

```text
ColumnDefinitionId
```

para identidad interna del AST/modelo.

Debe cumplirse:

```text
ColumnDefinitionId
≠
ColumnIdentifier
```

---

# 18. DatabaseType

El tipo deberá representarse mediante:

```text
DatabaseType
```

tipado.

No mediante un string arbitrario.

---

# 19. Type examples

Conceptualmente:

```text
DatabaseType
├── BooleanType
├── SmallIntegerType
├── IntegerType
├── BigIntegerType
├── DecimalType
├── FloatType
├── StringType
├── TextType
├── BinaryType
├── DateType
├── TimeType
├── DateTimeType
├── TimestampType
├── JsonType
├── UuidType
├── EnumType
└── ExtensionDatabaseType
```

La arquitectura completa del type system se desarrollará posteriormente.

---

# 20. Semantic type ≠ physical SQL type

Ejemplo:

```text
BooleanType
```

no deberá ser equivalente internamente a:

```text
TINYINT(1)
```

ni a:

```text
BOOLEAN
```

ni a:

```text
INTEGER
```

Esas son representaciones físicas posibles.

---

# 21. Canonical type

La definición conserva:

```text
canonical database type
```

El compiler decide:

```text
physical platform type
```

---

# 22. Type parameters

Los parámetros deberán ser tipados.

Ejemplo:

```text
StringType
└── length = 320
```

```text
DecimalType
├── precision = 18
└── scale = 4
```

---

# 23. No generic type arguments

Evitar:

```php
new Type('decimal', [
    'foo' => 18,
    'bar' => 4,
]);
```

Preferir:

```php
new DecimalType(
    precision: 18,
    scale: 4,
);
```

---

# 24. Type validation

Ejemplo:

```text
precision > 0
scale >= 0
scale <= precision
```

deberá poder comprobarse sin conexión a DB.

---

# 25. Portable vs platform-specific types

VoltStack deberá distinguir:

```text
PortableDatabaseType
```

de:

```text
PlatformSpecificDatabaseType
```

y:

```text
ExtensionDatabaseType
```

---

# 26. Platform-specific type example

Una aplicación podrá solicitar deliberadamente una capacidad particular.

Ejemplo conceptual:

```text
PostgreSqlTsVectorType
```

No debe fingirse que es portable.

---

# 27. Type capability requirements

Un tipo puede producir:

```text
SchemaCapabilityRequirementSet
```

Ejemplo:

```text
JsonType
→ JSON_TYPE
```

o una variante más específica según las capabilities.

---

# 28. Nullability

La nulabilidad deberá modelarse explícitamente.

Se propone:

```php
enum Nullability
{
    case NULLABLE;
    case NOT_NULL;
}
```

---

# 29. Nullability ≠ default

Una columna:

```text
NULLABLE
```

no implica:

```text
DEFAULT NULL
```

Y:

```text
DEFAULT NULL
```

no debe confundirse con ausencia de default.

---

# 30. Four-way distinction

Es crítico distinguir:

```text
NOT NULL + NO DEFAULT
NULLABLE + NO DEFAULT
NULLABLE + DEFAULT NULL
NOT NULL + NON-NULL DEFAULT
```

---

# 31. DefaultDefinition

Se propone:

```text
DefaultDefinition
├── NoDefault
├── LiteralDefault
├── NullDefault
├── ExpressionDefault
├── GeneratedDefault
└── ExtensionDefault
```

---

# 32. NoDefault

Debe existir como estado explícito:

```text
NO_DEFAULT
```

No utilizar simplemente:

```php
null
```

porque `null` puede representar un default SQL real.

---

# 33. Literal default

Ejemplo:

```php
LiteralDefault::string('pending');
```

o:

```php
LiteralDefault::integer(0);
```

---

# 34. Null default

Debe existir:

```text
NullDefault
```

separado de:

```text
NoDefault
```

Formalmente:

```text
DEFAULT NULL
≠
DEFAULT ABSENT
```

---

# 35. Expression default

Ejemplo:

```text
CURRENT_TIMESTAMP
```

deberá representarse mediante:

```text
SchemaExpression
```

cuando exista representación estructurada.

---

# 36. Raw default expression

Un escape hatch podrá utilizar:

```text
RawSchemaExpression
```

pero deberá quedar marcado explícitamente.

---

# 37. Default literal ≠ default expression

Esto:

```text
"CURRENT_TIMESTAMP"
```

como string literal es diferente de:

```text
CURRENT_TIMESTAMP
```

como expresión.

---

# 38. Typed defaults

Debe comprobarse compatibilidad conceptual:

```text
DefaultType
compatible with
ColumnType
```

cuando pueda determinarse estáticamente.

---

# 39. Example invalid default

```text
IntegerType
+
LiteralDefault("hello")
```

deberá generar diagnóstico salvo extensión/conversión explícita.

---

# 40. Default validation levels

Puede dividirse:

```text
Local Type Compatibility
Platform Capability Compatibility
Runtime/DDL Compatibility
```

---

# 41. GenerationDefinition

La generación automática de valores deberá modelarse explícitamente.

Se propone:

```text
GenerationDefinition
├── NoGeneration
├── IdentityGeneration
├── GeneratedColumn
├── SequenceGeneration
└── ExtensionGeneration
```

---

# 42. Identity ≠ generated column

```text
IdentityGeneration
≠
GeneratedColumn
```

Identity produce valores para una columna.

Generated column calcula su valor a partir de una expresión.

---

# 43. IdentityDefinition

Conceptualmente:

```text
IdentityDefinition
├── strategy
├── start?
├── increment?
├── min?
├── max?
├── cycle?
└── cache?
```

No todas las plataformas soportarán todos los campos.

---

# 44. Identity strategy

Puede existir:

```text
BY_DEFAULT
ALWAYS
PLATFORM_DEFAULT
```

cuando corresponda.

---

# 45. AUTO_INCREMENT

`AUTO_INCREMENT` no deberá convertirse en concepto universal.

Es una representación/semántica de plataforma.

VoltStack modelará:

```text
IdentityGeneration
```

y el platform layer decidirá la representación.

---

# 46. SQLite special semantics

SQLite posee reglas particulares alrededor de:

```text
INTEGER PRIMARY KEY
```

y:

```text
AUTOINCREMENT
```

El modelo no deberá esconder esas diferencias.

---

# 47. PostgreSQL identity

PostgreSQL puede compilar identity mediante:

```text
GENERATED BY DEFAULT AS IDENTITY
```

o:

```text
GENERATED ALWAYS AS IDENTITY
```

según definición.

---

# 48. Sequence generation

Cuando la plataforma use secuencias:

```text
SequenceGeneration
└── SequenceIdentifier
```

podrá declarar una dependencia estructural.

---

# 49. Generated columns

Una columna generada tendrá:

```text
GeneratedColumnDefinition
├── expression
├── storage strategy
└── capability requirements
```

---

# 50. Generated expression

Ejemplo:

```text
full_name =
first_name || ' ' || last_name
```

deberá modelarse mediante `SchemaExpression` cuando sea posible.

---

# 51. Generated storage

Puede distinguirse:

```text
VIRTUAL
STORED
PLATFORM_DEFAULT
```

---

# 52. Generated column dependencies

La expresión podrá declarar:

```text
ColumnDependencySet
```

Ejemplo:

```text
total
├── price
└── quantity
```

---

# 53. Generated dependency validation

Toda dependencia local deberá existir dentro de la tabla.

La validación completa ocurre cuando `ColumnDefinition` se encuentra dentro de `TableDefinition`.

---

# 54. Generation and defaults

Debe establecerse una política clara.

Por defecto:

```text
GeneratedColumn
+
ExplicitDefault
```

deberá considerarse conflictivo salvo que una capability específica permita la combinación.

---

# 55. Identity and default

Igualmente:

```text
IdentityGeneration
+
ExplicitDefault
```

puede tener semántica dependiente de plataforma.

No deberá aceptarse silenciosamente.

---

# 56. CharacterSetDefinition

Para columnas textuales podrá existir:

```text
CharacterSetDefinition
```

---

# 57. Charset applicability

No toda columna admite charset.

Ejemplo:

```text
IntegerType
+
CharacterSet(utf8mb4)
```

deberá ser inválido semánticamente.

---

# 58. CollationDefinition

Puede existir:

```text
CollationDefinition
```

para tipos compatibles.

---

# 59. Collation inheritance

Debe distinguirse:

```text
EXPLICIT
INHERITED
PLATFORM_DEFAULT
UNSPECIFIED
```

cuando sea necesario.

---

# 60. Column vs table collation

Resolución conceptual:

```text
Column explicit collation
        >
Table explicit/default collation
        >
Schema/database default
        >
Platform default
```

---

# 61. Do not flatten inheritance too early

No copiar automáticamente la table collation dentro de todas las columnas.

Eso destruiría provenance.

---

# 62. Character set vs collation

Aunque relacionados:

```text
CharacterSet
≠
Collation
```

deberán permanecer conceptos separados.

---

# 63. Column comment

Puede existir:

```text
ColumnComment
```

como value object.

---

# 64. Comment ≠ runtime metadata

Un comentario persistido en la base de datos forma parte del schema.

No debe confundirse con:

```text
developer source comment
```

---

# 65. Comment support

Si una plataforma no soporta comentarios directamente:

```text
ColumnDefinition
```

puede seguir representándolos.

El compatibility/compiler layer decide qué hacer.

---

# 66. ColumnOptionSet

Características adicionales deberán utilizar:

```text
ColumnOptionSet
```

---

# 67. Typed options

Ejemplo:

```text
ColumnOption
├── PortableColumnOption
├── MySqlColumnOption
├── MariaDbColumnOption
├── PostgreSqlColumnOption
├── SqliteColumnOption
└── ExtensionColumnOption
```

---

# 68. Unsigned

`UNSIGNED` es un buen ejemplo de característica que no debe asumirse universal.

Puede modelarse como:

```text
UnsignedNumericOption
```

con capability requirement.

---

# 69. Zerofill

Una característica histórica/específica como `ZEROFILL` deberá ser:

```text
platform-specific
```

si se decide soportarla.

---

# 70. Storage/compression options

Opciones futuras pueden incluir:

```text
compression
storage
statistics
encoding
```

cuando un motor lo soporte.

No deben añadirse como strings mágicos.

---

# 71. ColumnOption identity

Toda opción deberá tener:

```text
ColumnOptionTypeId
```

estable.

---

# 72. Option conflicts

El sistema deberá detectar:

```text
mutually exclusive options
duplicate options
invalid type/option combinations
```

---

# 73. Example

```text
StringType
+
UnsignedNumericOption
```

deberá fallar.

---

# 74. Capability requirements

`ColumnDefinition` podrá exponer:

```text
SchemaCapabilityRequirementSet
```

derivado de:

- type;
- generation;
- options;
- charset;
- collation;
- expressions;
- extensions.

---

# 75. Example

```text
GeneratedColumn(STORED)
```

puede requerir:

```text
GENERATED_COLUMNS
STORED_GENERATED_COLUMNS
```

---

# 76. Requirements are declarative

La columna dice:

```text
I require capability X
```

No pregunta:

```text
Does current database support X?
```

---

# 77. Capability validation

Ocurre posteriormente:

```text
ColumnDefinition
        │
        ▼
Schema Platform Compatibility
        │
        ▼
Capability Snapshot
```

---

# 78. ColumnMetadata

Se propone:

```php
final readonly class ColumnMetadata
{
    public function __construct(
        public SchemaObjectOrigin $origin,
        public ?SchemaSourceLocation $source,
        public AnnotationSet $annotations,
        public SensitivityMetadata $sensitivity,
    ) {}
}
```

---

# 79. Structural vs diagnostic metadata

Debe distinguirse:

```text
column.comment
```

que puede formar parte del schema,

de:

```text
source line
```

que sólo ayuda a diagnostics.

---

# 80. Provenance

Ciertos atributos podrán necesitar:

```text
ValueProvenance
```

Ejemplo:

```text
EXPLICIT
INHERITED
GENERATED
PLATFORM_GENERATED
INTROSPECTED
INFERRED
UNKNOWN
```

---

# 81. Why provenance matters

Supongamos:

```text
Desired:
VARCHAR(255)
```

e introspection reporta:

```text
VARCHAR(255)
COLLATION utf8mb4_general_ci
```

Si esa collation fue heredada automáticamente, no debe necesariamente generar una migration.

---

# 82. Unknown ≠ absent

Principio:

```text
UNKNOWN
≠
ABSENT
```

Ejemplo:

```text
driver cannot introspect generated expression
```

no significa:

```text
column has no generated expression
```

---

# 83. ObservedColumn

Se recomienda:

```php
final readonly class ObservedColumn
{
    public function __construct(
        public ColumnDefinition $definition,
        public DefinitionCompleteness $completeness,
        public ColumnObservationMetadata $observation,
    ) {}
}
```

---

# 84. Platform reported type

Introspection puede conservar:

```text
PlatformReportedType
```

por ejemplo:

```text
int8
varchar(255)
tinyint(1)
```

sin convertirlo en el tipo canónico principal.

---

# 85. Canonicalization

El introspector podrá normalizar:

```text
platform type
      ↓
canonical DatabaseType
```

conservando el valor original como observation metadata.

---

# 86. Lossless introspection goal

Idealmente:

```text
Canonical Meaning
+
Platform-Specific Metadata
```

deberán conservar suficiente información para evitar diffs falsos.

---

# 87. Canonical column form

Distintas DSLs equivalentes deberán poder producir:

```text
one canonical ColumnDefinition
```

---

# 88. Example

Estas APIs:

```php
$table->string('name');
```

y:

```php
$table->varchar('name', 255);
```

podrían canonicalizarse a:

```text
StringType(length: 255)
```

si la API considera ambas equivalentes.

---

# 89. Canonicalization boundary

Canonicalization no deberá convertir:

```text
StringType(255)
```

en:

```text
VARCHAR(255)
```

porque eso pertenece al platform compiler.

---

# 90. Canonical default

Estas formas:

```text
default(true)
```

y una representación booleana equivalente podrán convertirse a:

```text
LiteralDefault(BooleanValue(true))
```

---

# 91. Default canonicalization safety

Nunca tratar:

```text
'false'
```

como:

```text
false
```

sin una regla de tipos explícita.

---

# 92. Column structural equality

VoltStack deberá distinguir:

```text
Object Identity
Structural Equality
Normalized Structural Equality
Platform Compatibility
Migration Equivalence
```

---

# 93. Structural equality

Conceptualmente:

```text
StructuralEqual(A, B)
```

considerará los atributos estructurales de la columna.

---

# 94. Column name equality

Dependiendo del perfil:

```text
name
```

puede formar parte o no de una comparación.

Para comparar columnas dentro de la misma tabla:

```text
identifier
```

normalmente sí forma parte.

---

# 95. Rename-aware comparison

Schema Diff podrá utilizar un perfil especial para determinar similitud:

```text
ColumnSimilarity(A, B)
```

pero:

```text
Similarity
≠
Equality
```

---

# 96. Rename inference safety

Nunca:

```text
similar columns
⇒ definitely renamed
```

Rename inference será heurística o necesitará intención explícita.

---

# 97. Structural fingerprint

Podrá existir:

```text
ColumnStructuralFingerprint
```

---

# 98. Fingerprint inputs

Puede considerar:

```text
identifier
canonical type
type parameters
nullability
default
generation
charset
collation
comment
structural options
structural extensions
normalization version
```

---

# 99. Fingerprint exclusions

No incluir por defecto:

```text
source line
request ID
connection ID
object ID
introspection timestamp
telemetry data
```

---

# 100. Fingerprint profiles

Se proponen:

```text
FULL
PORTABLE
MIGRATION
TYPE_ONLY
CACHE
```

---

# 101. TYPE_ONLY fingerprint

Puede considerar:

```text
type
type parameters
nullability
generation
```

según propósito.

---

# 102. Fingerprint determinism

Debe cumplirse:

```text
Same Canonical Column
+
Same Fingerprint Profile
+
Same Algorithm Version
=
Same Fingerprint
```

---

# 103. Fingerprint ≠ SQL hash

Nunca definir:

```text
ColumnFingerprint
=
hash("VARCHAR(255) NOT NULL")
```

como identidad principal.

---

# 104. Serialization

`ColumnDefinition` deberá tener representación serializable y versionada.

Conceptualmente:

```json
{
  "identifier": "email",
  "type": {
    "id": "string",
    "length": 320
  },
  "nullability": "NOT_NULL",
  "default": {
    "kind": "NONE"
  },
  "generation": {
    "kind": "NONE"
  }
}
```

---

# 105. Serialization round-trip

Debe cumplirse:

```text
StructuralEqual(
    Column,
    Deserialize(Serialize(Column))
)
```

para formatos compatibles.

---

# 106. Serialization of extensions

Toda extensión estructural deberá incluir:

```text
extension ID
extension version
payload format version
structural impact
```

---

# 107. Serialization safety

No serializar:

```text
closures
PDO objects
live connections
request contexts
service container
tenant object
```

---

# 108. Column validation

Se propone:

```php
interface ColumnDefinitionValidator
{
    public function validate(
        ColumnDefinition $column,
        ColumnValidationContext $context,
    ): ColumnValidationResult;
}
```

---

# 109. Local validation

Podrá comprobar:

- valid identifier;
- valid type parameters;
- valid nullability/default combination;
- default/type compatibility;
- generation/default conflicts;
- charset/type compatibility;
- collation/type compatibility;
- option/type compatibility;
- duplicate/conflicting options;
- extension validity.

---

# 110. Table-context validation

Algunas reglas necesitan `TableDefinition`.

Ejemplo:

```text
GeneratedExpression
references
price
quantity
```

La existencia de esas columnas necesita table context.

---

# 111. Schema-context validation

Otras reglas pueden necesitar:

```text
SchemaValidationContext
```

por ejemplo:

```text
SequenceGeneration
→ referenced sequence exists
```

---

# 112. Platform validation

Posteriormente:

```text
Schema Platform Compatibility
```

verifica:

```text
type support
identity support
generated column support
collation support
platform options
DDL restrictions
```

---

# 113. Validation hierarchy

```text
Column-local validation
        ↓
Table-context validation
        ↓
Schema-context validation
        ↓
Platform compatibility validation
        ↓
Migration safety validation
        ↓
Execution feasibility
```

---

# 114. ColumnValidationResult

```php
final readonly class ColumnValidationResult
{
    public function __construct(
        public bool $valid,
        public SchemaDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 115. Diagnostics

Ejemplos:

```text
Column "price": decimal scale cannot exceed precision.

Column "name": unsigned option cannot be applied to StringType.

Column "total": generated expression references unknown column "quantity".

Column "id": identity generation conflicts with explicit generation strategy.
```

---

# 116. Diagnostic paths

Los diagnostics deberán poder identificar:

```text
table.users.column.email.default
```

o una ruta estructurada equivalente.

---

# 117. Source mapping

Cuando la columna provenga de DSL:

```text
database/schema/users.php:17
```

podrá conservarse como metadata diagnóstica.

---

# 118. Error hierarchy

Se propone:

```text
DatabaseColumnDefinitionException
├── InvalidColumnDefinitionException
├── InvalidColumnIdentifierException
├── InvalidColumnTypeException
├── InvalidTypeParameterException
├── InvalidNullabilityException
├── InvalidColumnDefaultException
├── IncompatibleColumnDefaultException
├── InvalidGenerationDefinitionException
├── ConflictingGenerationDefinitionException
├── InvalidGeneratedExpressionException
├── InvalidCharacterSetException
├── InvalidCollationException
├── InvalidColumnOptionException
├── ConflictingColumnOptionException
├── ColumnCapabilityException
├── ColumnSerializationException
├── ColumnExtensionException
├── ColumnDefinitionBudgetExceededException
└── ColumnDefinitionInvariantException
```

---

# 119. Error ≠ platform incompatibility

Debe diferenciarse:

```text
InvalidColumnDefinition
```

de:

```text
Valid definition unsupported by current platform
```

Ejemplo:

```text
valid generated-column definition
+
platform without generated columns
```

deberá producir capability/compatibility error downstream.

---

# 120. ColumnDefinitionFactory

Se propone:

```php
interface ColumnDefinitionFactory
{
    public function fromBlueprint(
        ColumnBlueprint $blueprint,
        ColumnDefinitionContext $context,
    ): ColumnDefinition;
}
```

---

# 121. Factory responsibilities

La factory:

- consume blueprint;
- canonicaliza;
- valida;
- crea typed values;
- deriva capability requirements;
- preserva provenance;
- produce definición inmutable.

---

# 122. Factory does not compile

Nunca:

```php
return 'VARCHAR(255)';
```

---

# 123. ColumnDefinitionNormalizer

Puede existir:

```php
interface ColumnDefinitionNormalizer
{
    public function normalize(
        ColumnDefinition $column,
        ColumnNormalizationContext $context,
    ): ColumnDefinition;
}
```

---

# 124. Normalization properties

Debe ser:

```text
deterministic
idempotent
semantics-preserving
side-effect free
```

---

# 125. Normalization idempotence

Idealmente:

```text
Normalize(Normalize(C))
=
Normalize(C)
```

---

# 126. Normalization must not query DB

No deberá obtener:

```text
server version
current collation
current database
```

de manera oculta.

---

# 127. Explicit context

Cuando la normalización requiera información:

```text
ColumnNormalizationContext
```

deberá recibirla explícitamente.

---

# 128. Type inference

Schema Builder puede ofrecer convenience APIs.

Pero una `ColumnDefinition` final deberá tener:

```text
explicit DatabaseType
```

No:

```text
infer later somehow
```

---

# 129. Default type inference

Para:

```php
$table->column('active')->default(true);
```

si no existe tipo explícito, la DSL podrá inferirlo si su contrato lo permite.

Pero la definición final deberá contener:

```text
BooleanType
```

explícito.

---

# 130. No unresolved builder state

No deberán llegar a `TableDefinition` columnas como:

```text
type = pending
nullable = maybe
```

salvo un modelo explícito de partial/introspected definitions.

---

# 131. Desired definitions must be complete

Una columna declarada para crear schema deberá ser suficientemente completa para validar y compilar.

---

# 132. Observed definitions may be partial

La introspection sí puede producir:

```text
ObservedColumn
```

con información parcial.

---

# 133. Do not mix the two

No introducir:

```php
bool $isPartial;
```

en cada aspecto del `ColumnDefinition` si puede modelarse mejor mediante observation wrappers.

---

# 134. Column transformations

Puede existir:

```php
interface ColumnDefinitionTransformer
{
    public function transform(
        ColumnDefinition $column
    ): ColumnDefinition;
}
```

---

# 135. Example transformation

```text
StringType(100)
      │
      │ widen
      ▼
StringType(255)
```

produce una nueva definición.

---

# 136. Transformation ≠ ALTER COLUMN

La transformación describe un nuevo estado.

No ejecuta ni implica automáticamente:

```sql
ALTER TABLE ...
```

---

# 137. Schema Diff integration

El futuro Schema Diff podrá calcular:

```text
ColumnDiff
├── TypeChanged
├── NullabilityChanged
├── DefaultChanged
├── GenerationChanged
├── CollationChanged
├── CommentChanged
└── OptionsChanged
```

---

# 138. Diff must preserve uncertainty

Si introspection reporta:

```text
default = UNKNOWN
```

y desired tiene:

```text
default = NoDefault
```

no deberá concluir automáticamente:

```text
DefaultRemoved
```

---

# 139. Three-state comparison

En presencia de metadata incompleta puede requerirse:

```text
EQUAL
DIFFERENT
UNKNOWN
```

en lugar de simple boolean.

---

# 140. Comparison result

Se propone:

```text
ColumnComparisonResult
├── EQUIVALENT
├── DIFFERENT
├── INDETERMINATE
└── diagnostics
```

---

# 141. Platform compatibility

Dos tipos distintos canónicamente podrían mapear al mismo tipo físico.

Ejemplo conceptual:

```text
CanonicalTypeA
CanonicalTypeB
      │
      ▼
same platform representation
```

Eso no significa que sean estructuralmente iguales.

---

# 142. Preserve semantic distinctions

El compiler puede producir la misma representación física sin que el Schema Model colapse los conceptos.

---

# 143. Numeric types

El modelo deberá evitar suposiciones como:

```text
IntegerType
=
32-bit everywhere
```

si la abstracción no lo garantiza.

---

# 144. Precision semantics

Para tipos exactos:

```text
DecimalType(p, s)
```

deberá definirse formalmente qué significan:

```text
p = precision
s = scale
```

---

# 145. Floating-point semantics

No tratar:

```text
FloatType
```

como:

```text
DecimalType
```

---

# 146. Boolean semantics

`BooleanType` representa dominio lógico booleano.

Su representación física puede variar.

---

# 147. String semantics

`StringType` deberá diferenciar conceptos como:

```text
bounded string
unbounded text
fixed-length string
```

cuando sea relevante.

---

# 148. Binary semantics

Igualmente:

```text
BinaryType
LargeBinaryType
```

deberán permanecer distintos.

---

# 149. Temporal semantics

Deberán distinguirse conceptos como:

```text
Date
Time
DateTime
Timestamp
```

y eventualmente:

```text
timezone-aware
timezone-naive
```

si el type system de VoltStack lo soporta.

---

# 150. JSON semantics

`JsonType` no debe degradarse inmediatamente a:

```text
TEXT
```

aunque SQLite u otro motor requiera una representación diferente.

---

# 151. UUID semantics

`UuidType` podrá compilarse de distintas maneras:

```text
native UUID
binary
character
```

según plataforma/configuración.

La definición conserva la semántica UUID.

---

# 152. Enum semantics

Un enum puede ser:

```text
application enum
database enum
check-backed enum
native platform enum
```

Estos conceptos no deberán confundirse.

---

# 153. Schema enum

Si VoltStack modela enum como objeto de schema:

```text
EnumTypeDefinition
```

la columna podrá depender de dicho objeto.

---

# 154. Value objects

`ColumnDefinition` no debe conocer PHP value objects del ORM.

El mapping:

```text
PHP Value Object
      ↓
ORM Type Mapping
      ↓
DatabaseType
```

ocurre en otra capa.

---

# 155. Column sensitivity

Metadata podrá marcar una columna como:

```text
PII
credential
financial
internal
secret
```

pero esto será metadata de seguridad/tooling.

---

# 156. Sensitivity ≠ encryption

Una columna marcada sensible no implica automáticamente:

```text
encrypt column
```

---

# 157. Sensitivity ≠ authorization

Tampoco implementa permisos.

---

# 158. Sensitive defaults

Diagnostics/telemetry no deberán imprimir automáticamente defaults potencialmente sensibles.

---

# 159. Security of raw expressions

Raw defaults/generated expressions pueden contener:

```text
database functions
identifiers
literals
vendor syntax
```

y deberán tratarse como contenido privilegiado.

---

# 160. Identifier binding misconception

Los nombres de columnas no se protegen mediante parameter binding.

```text
SELECT ? FROM users
```

no sustituye de forma portable un identifier.

Por eso:

```text
identifier validation
+
compiler quoting
```

son obligatorios.

---

# 161. Schema injection protection

Un identifier no confiable nunca deberá convertirse directamente en SQL.

Flujo correcto:

```text
input
 ↓
ColumnIdentifier validation
 ↓
Schema AST
 ↓
Compiler
 ↓
Dialect quoting
```

---

# 162. Extension model

Una extensión podrá aportar:

```text
custom type
custom generation strategy
custom option
custom metadata
custom capability requirement
custom serialization
```

---

# 163. Extension IDs

Toda extensión deberá usar identidad estable:

```text
ExtensionId
```

---

# 164. No anonymous extension semantics

Evitar:

```php
$options['custom'] = $anything;
```

---

# 165. Frozen extension registry

La interpretación deberá depender de:

```text
FrozenColumnExtensionRegistry
```

---

# 166. Extension conflict

Dos extensiones que intenten controlar la misma semántica deberán producir conflicto explícito.

---

# 167. Unknown structural extension

No deberá ignorarse durante:

```text
comparison
fingerprinting
serialization
compilation
```

---

# 168. Column definition budgets

Se propone:

```php
final readonly class ColumnDefinitionBudget
{
    public function __construct(
        public int $maxTypeDepth,
        public int $maxDefaultExpressionDepth,
        public int $maxGeneratedExpressionDepth,
        public int $maxOptions,
        public int $maxAnnotations,
        public int $maxExtensionPayloadBytes,
    ) {}
}
```

---

# 169. Budget overflow

Debe producir:

```text
ColumnDefinitionBudgetExceededException
```

No truncar silenciosamente.

---

# 170. Persistent runtime safety

`ColumnDefinition` deberá ser:

```text
immutable
connection-free
request-free
tenant-state-free
transaction-free
driver-handle-free
```

---

# 171. Safe sharing

Podrá compartirse:

```text
Canonical ColumnDefinition
```

entre operaciones cuando el contexto estructural sea compatible.

---

# 172. Unsafe sharing

No deberá contener:

```text
current Connection
current Platform instance with mutable state
current TenantContext
current Request
current Migration execution
```

---

# 173. FrankenPHP

En workers persistentes:

```text
Request A
    ↓
ColumnDefinition
```

no deberá dejar información mutable que afecte:

```text
Request B
```

---

# 174. RoadRunner/OpenSwoole

La misma regla aplica.

La arquitectura deberá ser runtime-neutral.

---

# 175. Column definition caching

Una definición inmutable podrá almacenarse en metadata caches.

La key completa pertenece al cache/schema context, no a `ColumnDefinition` por sí sola.

---

# 176. No global current column registry

Evitar:

```php
ColumnRegistry::$current['email'];
```

con estado global mutable.

---

# 177. Testing strategy

Las pruebas principales deberán ejecutarse sin DB.

---

# 178. Identifier tests

Probar:

```text
valid names
invalid names
Unicode policy
reserved-looking names
case distinctions
length limits at logical layer
```

---

# 179. Type tests

Probar:

```text
valid type parameters
invalid precision
invalid scale
invalid lengths
custom types
extension types
```

---

# 180. Default tests

Probar:

```text
NoDefault
NullDefault
LiteralDefault
ExpressionDefault
raw default
type mismatch
```

---

# 181. Generation tests

Probar:

```text
identity
sequence
generated virtual
generated stored
generation/default conflicts
```

---

# 182. Charset/collation tests

Probar:

```text
valid textual application
invalid numeric application
inheritance provenance
explicit override
```

---

# 183. Option tests

Probar:

```text
duplicates
conflicts
invalid type/option combinations
platform-specific options
extensions
```

---

# 184. Serialization tests

```text
ColumnDefinition
      ↓ serialize
SerializedColumn
      ↓ deserialize
ColumnDefinition'
```

Debe cumplirse:

```text
StructuralEqual(ColumnDefinition, ColumnDefinition')
```

---

# 185. Fingerprint tests

Misma definición canónica:

```text
same fingerprint
```

Diferencia estructural relevante:

```text
different fingerprint
```

según profile.

---

# 186. Normalization tests

Debe comprobarse:

```text
Normalize(C)
=
Normalize(Normalize(C))
```

---

# 187. Mutation tests

Intentar modificar una definición publicada deberá ser imposible mediante API soportada.

---

# 188. Persistent worker tests

Simular:

```text
Request A
Request B
Request C
```

sobre el mismo worker y verificar ausencia de state leakage.

---

# 189. Cross-platform tests

Una misma definición:

```text
UuidType
```

podrá tener distintas representaciones físicas sin alterar el objeto canónico.

---

# 190. Introspection round-trip

Futuro test:

```text
ColumnDefinition
      ↓
Compile
      ↓
Database
      ↓
Introspect
      ↓
ObservedColumn
```

y verificar:

```text
PlatformCompatible(
    Desired,
    Observed
)
```

---

# 191. Proposed namespace

```text
VoltStack\Quantum\Database\Schema\Definition\Column
```

---

# 192. Proposed directory structure

```text
Schema/
└── Definition/
    └── Column/
        ├── ColumnDefinition.php
        ├── ColumnDefinitionId.php
        ├── ColumnDefinitionFactory.php
        ├── ColumnDefinitionNormalizer.php
        │
        ├── Identifier/
        │   ├── ColumnIdentifier.php
        │   └── QualifiedColumnIdentifier.php
        │
        ├── Type/
        │   ├── DatabaseType.php
        │   ├── PortableDatabaseType.php
        │   ├── PlatformSpecificDatabaseType.php
        │   └── ExtensionDatabaseType.php
        │
        ├── Nullability/
        │   └── Nullability.php
        │
        ├── Default/
        │   ├── DefaultDefinition.php
        │   ├── NoDefault.php
        │   ├── NullDefault.php
        │   ├── LiteralDefault.php
        │   ├── ExpressionDefault.php
        │   └── ExtensionDefault.php
        │
        ├── Generation/
        │   ├── GenerationDefinition.php
        │   ├── NoGeneration.php
        │   ├── IdentityGeneration.php
        │   ├── IdentityDefinition.php
        │   ├── SequenceGeneration.php
        │   ├── GeneratedColumnDefinition.php
        │   └── ExtensionGeneration.php
        │
        ├── Charset/
        │   └── CharacterSetDefinition.php
        │
        ├── Collation/
        │   └── CollationDefinition.php
        │
        ├── Comment/
        │   └── ColumnComment.php
        │
        ├── Option/
        │   ├── ColumnOption.php
        │   ├── ColumnOptionSet.php
        │   ├── PortableColumnOption.php
        │   ├── PlatformColumnOption.php
        │   └── ExtensionColumnOption.php
        │
        ├── Metadata/
        │   ├── ColumnMetadata.php
        │   ├── ValueProvenance.php
        │   ├── ColumnObservationMetadata.php
        │   └── ObservedColumn.php
        │
        ├── Capability/
        │   └── ColumnCapabilityRequirementResolver.php
        │
        ├── Comparison/
        │   ├── ColumnDefinitionComparator.php
        │   ├── ColumnComparisonResult.php
        │   └── ColumnComparisonProfile.php
        │
        ├── Fingerprint/
        │   ├── ColumnFingerprintGenerator.php
        │   ├── ColumnStructuralFingerprint.php
        │   └── ColumnFingerprintProfile.php
        │
        ├── Serialization/
        │   ├── ColumnDefinitionSerializer.php
        │   └── ColumnDefinitionDeserializer.php
        │
        ├── Validation/
        │   ├── ColumnDefinitionValidator.php
        │   ├── ColumnValidationContext.php
        │   └── ColumnValidationResult.php
        │
        ├── Transformation/
        │   └── ColumnDefinitionTransformer.php
        │
        ├── Extension/
        │   ├── ColumnDefinitionExtension.php
        │   └── FrozenColumnExtensionRegistry.php
        │
        ├── Budget/
        │   └── ColumnDefinitionBudget.php
        │
        └── Exception/
            ├── DatabaseColumnDefinitionException.php
            ├── InvalidColumnDefinitionException.php
            ├── InvalidColumnIdentifierException.php
            ├── InvalidColumnTypeException.php
            ├── InvalidTypeParameterException.php
            ├── InvalidColumnDefaultException.php
            ├── IncompatibleColumnDefaultException.php
            ├── InvalidGenerationDefinitionException.php
            ├── ConflictingGenerationDefinitionException.php
            ├── InvalidGeneratedExpressionException.php
            ├── InvalidCharacterSetException.php
            ├── InvalidCollationException.php
            ├── InvalidColumnOptionException.php
            ├── ConflictingColumnOptionException.php
            ├── ColumnCapabilityException.php
            ├── ColumnSerializationException.php
            ├── ColumnExtensionException.php
            ├── ColumnDefinitionBudgetExceededException.php
            └── ColumnDefinitionInvariantException.php
```

---

# 193. Dependency rules

Permitido:

```text
ColumnDefinition
    ↓
ColumnIdentifier
DatabaseType
SchemaExpression
SchemaCapabilityRequirement
immutable metadata
immutable extension descriptors
```

No permitido:

```text
ColumnDefinition
    ↓
PDO
Connection
DriverStatement
QueryExecutor
EntityManager
UnitOfWork
MigrationRunner
HTTP Request
mutable TenantContext
```

---

# 194. Architectural invariants

## DB-COLUMN-001

Toda columna tendrá `ColumnIdentifier`.

## DB-COLUMN-002

`ColumnDefinition` será inmutable.

## DB-COLUMN-003

`ColumnDefinition` será distinta de `ColumnBlueprint`.

## DB-COLUMN-004

`ColumnDefinition` será distinta de `ColumnAlteration`.

## DB-COLUMN-005

`ColumnDefinition` será distinta de `ObservedColumn`.

## DB-COLUMN-006

`ColumnDefinition` será distinta de una propiedad ORM.

## DB-COLUMN-007

`ColumnDefinition` será distinta de SQL.

## DB-COLUMN-008

La columna no generará SQL.

## DB-COLUMN-009

La columna no ejecutará SQL.

## DB-COLUMN-010

La columna no abrirá conexiones.

## DB-COLUMN-011

Identifiers serán tipados.

## DB-COLUMN-012

Identifiers no contendrán quoting prematuro.

## DB-COLUMN-013

`ColumnDefinitionId` será distinto de `ColumnIdentifier`.

## DB-COLUMN-014

Todo desired column tendrá tipo explícito.

## DB-COLUMN-015

DatabaseType será estructurado.

## DB-COLUMN-016

DatabaseType no será SQL type string.

## DB-COLUMN-017

Semantic type será distinto de physical type.

## DB-COLUMN-018

Type parameters serán tipados.

## DB-COLUMN-019

Type parameters serán localmente validables.

## DB-COLUMN-020

Portable type será distinto de platform-specific type.

## DB-COLUMN-021

Platform-specific types serán explícitos.

## DB-COLUMN-022

Nullability será explícita.

## DB-COLUMN-023

Nullability será distinta de default.

## DB-COLUMN-024

NoDefault será distinto de NullDefault.

## DB-COLUMN-025

LiteralDefault será distinto de ExpressionDefault.

## DB-COLUMN-026

Raw default será explícito.

## DB-COLUMN-027

Defaults preservarán tipos.

## DB-COLUMN-028

Default compatibility será validable cuando sea posible.

## DB-COLUMN-029

Generation será explícita.

## DB-COLUMN-030

IdentityGeneration será distinta de GeneratedColumn.

## DB-COLUMN-031

Identity será distinta de vendor AUTO_INCREMENT syntax.

## DB-COLUMN-032

Generated expression será estructurada cuando sea posible.

## DB-COLUMN-033

Raw generated expression será explícita.

## DB-COLUMN-034

Generated dependencies podrán derivarse.

## DB-COLUMN-035

Generation/default conflicts no serán ignorados.

## DB-COLUMN-036

Sequence generation podrá declarar schema dependency.

## DB-COLUMN-037

Charset será distinto de collation.

## DB-COLUMN-038

Charset sólo será válido para tipos compatibles.

## DB-COLUMN-039

Collation sólo será válida para tipos compatibles.

## DB-COLUMN-040

Inherited collation será distinguible de explicit collation.

## DB-COLUMN-041

Platform default será distinguible de explicit value cuando sea relevante.

## DB-COLUMN-042

Column comments estructurales serán distintos de source comments.

## DB-COLUMN-043

Column options serán tipadas.

## DB-COLUMN-044

No se usará mixed option bag como modelo principal.

## DB-COLUMN-045

Platform options serán explícitas.

## DB-COLUMN-046

Unknown options no se ignorarán.

## DB-COLUMN-047

Conflicting options fallarán.

## DB-COLUMN-048

Invalid type/option combinations fallarán.

## DB-COLUMN-049

Capability requirements serán declarativos.

## DB-COLUMN-050

Capability requirement será distinto de platform support.

## DB-COLUMN-051

ColumnDefinition no consultará platform capabilities.

## DB-COLUMN-052

Capability validation será downstream.

## DB-COLUMN-053

Metadata estructural será distinta de diagnostic metadata.

## DB-COLUMN-054

Origin será distinto de certainty.

## DB-COLUMN-055

UNKNOWN será distinto de ABSENT.

## DB-COLUMN-056

Observed columns no fingirán metadata ausente.

## DB-COLUMN-057

ObservedColumn podrá envolver ColumnDefinition.

## DB-COLUMN-058

Observation metadata no contaminará el structural core.

## DB-COLUMN-059

Platform reported type podrá conservarse separadamente.

## DB-COLUMN-060

Canonical type será distinto de platform reported type.

## DB-COLUMN-061

Canonicalization será determinista.

## DB-COLUMN-062

Canonicalization será semantics-preserving.

## DB-COLUMN-063

Canonicalization será idempotente.

## DB-COLUMN-064

Canonicalization no será SQL compilation.

## DB-COLUMN-065

Canonicalization no consultará DB ocultamente.

## DB-COLUMN-066

Equivalent DSL forms podrán producir la misma definición.

## DB-COLUMN-067

Structural equality será distinta de object identity.

## DB-COLUMN-068

Structural equality será distinta de platform compatibility.

## DB-COLUMN-069

Platform compatibility será distinta de migration equivalence.

## DB-COLUMN-070

Similarity será distinta de equality.

## DB-COLUMN-071

Similarity no probará rename.

## DB-COLUMN-072

Rename intent pertenecerá al operation model.

## DB-COLUMN-073

Fingerprint será estructural.

## DB-COLUMN-074

Fingerprint no será SQL hash.

## DB-COLUMN-075

Fingerprint será determinista.

## DB-COLUMN-076

Fingerprint tendrá algorithm version.

## DB-COLUMN-077

Fingerprint excluirá request IDs.

## DB-COLUMN-078

Fingerprint excluirá connection IDs.

## DB-COLUMN-079

Fingerprint excluirá runtime object IDs.

## DB-COLUMN-080

Fingerprint podrá tener profiles.

## DB-COLUMN-081

Serialization será determinista.

## DB-COLUMN-082

Serialization será versionada.

## DB-COLUMN-083

Serialization no dependerá de live resources.

## DB-COLUMN-084

Serialization preservará structural extensions.

## DB-COLUMN-085

Serialization round-trip preservará semántica.

## DB-COLUMN-086

Local validation no requerirá DB.

## DB-COLUMN-087

Table-context validation será explícita.

## DB-COLUMN-088

Schema-context validation será explícita.

## DB-COLUMN-089

Platform validation será downstream.

## DB-COLUMN-090

Migration safety será downstream.

## DB-COLUMN-091

Execution feasibility será downstream.

## DB-COLUMN-092

ColumnDefinition no decidirá ALTER strategy.

## DB-COLUMN-093

ColumnDefinition no decidirá table rebuild.

## DB-COLUMN-094

ColumnDefinition no ejecutará migration.

## DB-COLUMN-095

ColumnDefinition representará state.

## DB-COLUMN-096

ColumnAlteration representará transition.

## DB-COLUMN-097

Schema Diff podrá comparar columnas.

## DB-COLUMN-098

Schema Diff preservará uncertainty.

## DB-COLUMN-099

UNKNOWN comparison no se convertirá en DIFFERENT automáticamente.

## DB-COLUMN-100

ColumnDefinition será ORM-independent.

## DB-COLUMN-101

ColumnDefinition no contendrá entity class.

## DB-COLUMN-102

ColumnDefinition no contendrá repository class.

## DB-COLUMN-103

ColumnDefinition no contendrá UnitOfWork state.

## DB-COLUMN-104

ColumnDefinition no contendrá dirty tracking.

## DB-COLUMN-105

ColumnDefinition no contendrá hydration state.

## DB-COLUMN-106

ColumnDefinition será Query Engine-independent.

## DB-COLUMN-107

Query Semantic Engine podrá consumir Schema Model.

## DB-COLUMN-108

ColumnDefinition será Compiler-independent.

## DB-COLUMN-109

Compiler podrá consumir ColumnDefinition.

## DB-COLUMN-110

ColumnDefinition será Driver-independent.

## DB-COLUMN-111

ColumnDefinition no contendrá PDO.

## DB-COLUMN-112

ColumnDefinition no contendrá driver statement.

## DB-COLUMN-113

ColumnDefinition no contendrá live connection.

## DB-COLUMN-114

ColumnDefinition no contendrá transaction state.

## DB-COLUMN-115

ColumnDefinition no contendrá request state.

## DB-COLUMN-116

ColumnDefinition no contendrá mutable tenant state.

## DB-COLUMN-117

Immutable definitions serán persistent-runtime safe.

## DB-COLUMN-118

Mutable builders no se almacenarán dentro de definitions.

## DB-COLUMN-119

Extensions serán tipadas.

## DB-COLUMN-120

Extensions tendrán stable IDs.

## DB-COLUMN-121

Structural extension impact será explícito.

## DB-COLUMN-122

Unknown structural extensions no se ignorarán.

## DB-COLUMN-123

Extension registry será frozen.

## DB-COLUMN-124

No habrá last-wins extension behavior.

## DB-COLUMN-125

Extension conflicts serán explícitos.

## DB-COLUMN-126

Definition budgets serán explícitos.

## DB-COLUMN-127

Budget overflow no truncará definiciones.

## DB-COLUMN-128

Sensitive metadata no se registrará automáticamente.

## DB-COLUMN-129

Sensitivity metadata será distinta de authorization.

## DB-COLUMN-130

Sensitivity metadata será distinta de encryption.

## DB-COLUMN-131

Raw expression provenance será preservado.

## DB-COLUMN-132

Raw expression trust no aumentará automáticamente.

## DB-COLUMN-133

Identifiers no usarán value parameter binding.

## DB-COLUMN-134

Identifiers serán validados y quoted por compiler.

## DB-COLUMN-135

Definition transformations producirán nuevas instancias.

## DB-COLUMN-136

Transformers no mutarán inputs.

## DB-COLUMN-137

Transformers no ejecutarán SQL.

## DB-COLUMN-138

Transformers no equivaldrán a migrations.

## DB-COLUMN-139

Desired definitions serán completas para compilación.

## DB-COLUMN-140

Observed definitions podrán ser parciales mediante wrappers explícitos.

## DB-COLUMN-141

Partial introspection no se confundirá con ausencia.

## DB-COLUMN-142

Type semantics se preservarán cross-platform.

## DB-COLUMN-143

BooleanType no se degradará conceptualmente a integer.

## DB-COLUMN-144

JsonType no se degradará conceptualmente a text.

## DB-COLUMN-145

UuidType no se degradará conceptualmente a string.

## DB-COLUMN-146

DecimalType será distinto de floating-point type.

## DB-COLUMN-147

Temporal types conservarán sus diferencias semánticas.

## DB-COLUMN-148

Platform physical equivalence no implicará canonical equality.

## DB-COLUMN-149

ColumnDefinition será testeable sin database.

## DB-COLUMN-150

`ColumnDefinition` será la unidad estructural canónica de columna del Schema System de VoltStack.

---

# 195. Anti-patterns

## 195.1 SQL type string

Incorrecto:

```php
$column->type = 'BIGINT UNSIGNED AUTO_INCREMENT';
```

---

## 195.2 Null for every default state

Incorrecto:

```php
$column->default = null;
```

si no permite distinguir:

```text
NO DEFAULT
```

de:

```text
DEFAULT NULL
```

---

## 195.3 Boolean unique

Demasiado pobre:

```php
$column->unique = true;
```

si semánticamente debe existir una `UniqueConstraintDefinition`.

---

## 195.4 ORM property inside schema

Incorrecto:

```php
$column->property = User::class . '::$email';
```

---

## 195.5 Platform branching

Incorrecto:

```php
if ($database === 'mysql') {
    $column->type = 'tinyint(1)';
}
```

---

## 195.6 Generated SQL inside definition

Incorrecto:

```php
$column->sql = '"email" VARCHAR(320) NOT NULL';
```

---

## 195.7 Mutable introspection object as canonical definition

Incorrecto:

```text
DriverColumnMetadata
=
ColumnDefinition
```

---

## 195.8 Unknown = false

Incorrecto:

```text
generated expression unavailable
⇒ column is not generated
```

---

## 195.9 AUTO_INCREMENT as universal concept

Incorrecto:

```text
ColumnDefinition.autoincrement = true
```

como única semántica de generación.

Preferir:

```text
IdentityGeneration
```

---

## 195.10 VARCHAR as canonical semantic type

Incorrecto:

```text
canonical type = VARCHAR
```

cuando el diseño busca separar semántica de dialecto.

---

# 196. Ejemplo completo

```php
$column = new ColumnDefinition(
    identifier: ColumnIdentifier::from('email'),

    type: new StringType(
        length: 320,
    ),

    nullability: Nullability::NOT_NULL,

    default: NoDefault::instance(),

    generation: NoGeneration::instance(),

    characterSet: null,

    collation: null,

    comment: new ColumnComment(
        'Primary contact email address'
    ),

    options: ColumnOptionSet::empty(),

    capabilities: SchemaCapabilityRequirementSet::empty(),

    metadata: ColumnMetadata::declared(),

    extensions: ExtensionMetadataSet::empty(),
);
```

Representación:

```text
ColumnDefinition(email)
│
├── Type
│   └── StringType
│       └── length: 320
│
├── Nullability
│   └── NOT_NULL
│
├── Default
│   └── NO_DEFAULT
│
├── Generation
│   └── NONE
│
├── Charset
│   └── inherited/unspecified
│
├── Collation
│   └── inherited/unspecified
│
├── Comment
│   └── Primary contact email address
│
├── Options
│   └── ∅
│
└── Extensions
    └── ∅
```

---

# 197. Ejemplo Identity

```php
$id = new ColumnDefinition(
    identifier: ColumnIdentifier::from('id'),
    type: new BigIntegerType(),
    nullability: Nullability::NOT_NULL,
    default: NoDefault::instance(),
    generation: new IdentityGeneration(
        strategy: IdentityStrategy::BY_DEFAULT,
    ),
    characterSet: null,
    collation: null,
    comment: null,
    options: ColumnOptionSet::empty(),
    capabilities: SchemaCapabilityRequirementSet::of(
        SchemaCapability::IDENTITY_COLUMNS,
    ),
    metadata: ColumnMetadata::declared(),
    extensions: ExtensionMetadataSet::empty(),
);
```

No contiene:

```text
AUTO_INCREMENT
SERIAL
IDENTITY SQL
```

Sólo intención estructural.

---

# 198. Ejemplo generated column

```text
ColumnDefinition(total)
│
├── type
│   └── DecimalType(18, 2)
│
├── generation
│   └── GeneratedColumn
│       ├── expression
│       │   └── price * quantity
│       └── storage
│           └── STORED
│
└── dependencies
    ├── price
    └── quantity
```

El sistema puede derivar:

```text
total
 ├────► price
 └────► quantity
```

---

# 199. Ejemplo de default

Estas cuatro columnas son semánticamente diferentes:

```text
A
├── NULLABLE
└── NO_DEFAULT
```

```text
B
├── NULLABLE
└── DEFAULT NULL
```

```text
C
├── NOT_NULL
└── DEFAULT 0
```

```text
D
├── NOT_NULL
└── NO_DEFAULT
```

VoltStack no deberá colapsarlas.

---

# 200. Ejemplo de compilación cross-platform

Definición:

```text
ColumnDefinition(id)
├── BigIntegerType
├── NOT_NULL
└── IdentityGeneration
```

Posibles representaciones:

```text
              ColumnDefinition
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
      MySQL      PostgreSQL     SQLite
        │            │            │
        ▼            ▼            ▼
     BIGINT       BIGINT       INTEGER
 AUTO_INCREMENT   IDENTITY    special rules
```

La definición no cambia.

---

# 201. Fórmula del sistema

```text
Column Definition System
=
Column Identity
+
Canonical Database Type
+
Type Parameters
+
Nullability
+
Default Semantics
+
Generation Semantics
+
Character Set
+
Collation
+
Column Options
+
Structural Metadata
+
Capability Requirements
+
Extension Metadata
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
Persistent Runtime Safety
```

---

# 202. Correctness formula

```text
CorrectColumnDefinition
=
Immutable
∧ Typed
∧ Canonical
∧ LocallyValid
∧ Deterministic
∧ PlatformIndependent
∧ DriverIndependent
∧ IntentPreserving
∧ DiffFriendly
∧ Serializable
∧ ExtensionSafe
∧ RuntimeSafe
```

---

# 203. Local validity formula

Para una columna `C`:

```text
Valid(C)
=
ValidIdentifier(C)
∧ ValidType(C)
∧ ValidTypeParameters(C)
∧ ValidNullability(C)
∧ CompatibleDefault(C)
∧ ConsistentGeneration(C)
∧ CompatibleCharacterSet(C)
∧ CompatibleCollation(C)
∧ CompatibleOptions(C)
∧ ValidExtensions(C)
```

---

# 204. Semantic preservation

Para cualquier compilación válida:

```text
Meaning(
    Compile(ColumnDefinition, Platform)
)
=
Meaning(ColumnDefinition)
```

dentro de las capabilities declaradas.

El compiler puede cambiar representación.

No significado.

---

# 205. Definition-state rule

```text
ColumnDefinition
=
Column Structural State
```

mientras:

```text
AddColumnNode
DropColumnNode
RenameColumnNode
AlterColumnNode
=
Column Structural Operations
```

---

# 206. Integration architecture

```text
ColumnBlueprint
      │
      ▼
ColumnDefinition
      │
      ▼
TableDefinition
      │
      ├──────────────► Schema Model
      │
      └──────────────► Schema AST
                            │
                            ▼
                       Schema Diff
                            │
                            ▼
                     Schema Planning
                            │
                            ▼
                     Schema Compiler
                            │
                            ▼
                    Execution Engine
```

Introspection:

```text
Database
   │
   ▼
Schema Introspector
   │
   ▼
ObservedColumn
├── ColumnDefinition
└── Observation Metadata
```

---

# 207. Final architectural rule

> **Una columna en VoltStack es una definición estructural tipada, inmutable y portable; nunca una cadena SQL, una propiedad ORM o un objeto mutable del driver.**

La arquitectura deberá mantener permanentemente:

```text
ColumnBlueprint
      │
      │ constructs
      ▼
ColumnDefinition
      │
      │ belongs to
      ▼
TableDefinition
      │
      │ represented/changed through
      ▼
Schema AST
      │
      │ analyzed/planned
      ▼
Schema Planner
      │
      │ represented physically by
      ▼
Schema Compiler
      │
      │ executed by
      ▼
Execution Engine
```

---

# 208. Resultado arquitectónico

Con `ColumnDefinition System`, VoltStack obtiene una representación común capaz de servir simultáneamente a:

```text
Schema Builder
Schema Model
Schema Introspection
Schema Validation
Schema Diff
Schema Compiler
Migration System
Query Semantic Analysis
ORM Mapping Tooling
Testing
Diagnostics
Metadata Cache
```

sin permitir que esas capas se mezclen.

La separación clave queda:

```text
                 ┌─────────────────┐
                 │ ColumnBlueprint │
                 └────────┬────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │ ColumnDefinition │
                 └────────┬─────────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
       TableDefinition  Schema Diff  Schema Compiler
             │            │            │
             ▼            ▼            ▼
       Schema Model    Change Set      SQL/DDL
```

De esta forma, el modelo estructural permanece independiente tanto de la ergonomía de construcción como de la representación física.

---

# 209. Siguiente documento

```text
93_DATABASE_INDEX_SYSTEM.md
```

El siguiente documento formalizará:

```text
IndexDefinition
├── IndexIdentifier
├── IndexKind
├── IndexKeySet
│   ├── ColumnIndexKey
│   └── ExpressionIndexKey
├── IncludedColumns
├── Predicate
├── Uniqueness
├── Ordering
├── NullOrdering
├── IndexMethod
├── IndexOptions
├── CapabilityRequirements
├── Metadata
└── ExtensionMetadata
```

y deberá establecer claramente:

```text
Index
≠
Constraint
≠
Primary Key
≠
Unique Constraint
≠
Foreign Key
≠
Query Access Path
≠
Physical Query Plan IndexScan
```

Principio del siguiente documento:

> **Un índice describe una estructura persistente de acceso de datos; no una constraint lógica ni una decisión de ejecución de una consulta.**