# 163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## Database Custom Type Extension System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 163 — Database Custom Type Extension System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `162_DATABASE_DATE_TIME_TYPE_SYSTEM.md`  
**Siguiente documento:** `164_DATABASE_TRANSACTION_ARCHITECTURE.md`

---

# 1. Propósito

`Database Custom Type Extension System` define la arquitectura mediante la cual aplicaciones, paquetes oficiales y extensiones de terceros podrán incorporar nuevos tipos lógicos al Database System de VoltStack sin modificar el núcleo.

El sistema deberá permitir extensiones como:

```text
money
geo.point
vector
inet
cidr
ltree
encrypted_string
domain.email
domain.currency
domain.phone_number
custom.identifier
```

manteniendo las mismas garantías arquitectónicas de los tipos core.

La regla central será:

> **Un Custom Type de VoltStack ampliará el modelo lógico mediante contratos registrados, compilados y verificables; nunca mediante excepciones dispersas, `if` por vendor, reflection en runtime, SQL incrustado en el tipo o acceso directo del Custom Type al Driver.**

Un Custom Type deberá integrarse con:

```text
Type System
Type Registry
Value Conversion
Casting
Schema
Query Engine
Platform
Hydration
Persistence
Parameter Binding
Diagnostics
Telemetry
Testing
```

mediante fronteras explícitas.

---

# 2. Objetivo arquitectónico

VoltStack deberá permitir:

```text
Core Database
     │
     ▼
Extension Contracts
     │
     ├── Official Types
     ├── Package Types
     └── Application Types
```

sin convertir el Type System en:

```text
giant switch statement
+
vendor-specific conditionals
+
runtime reflection
+
mutable global registry
```

---

# 3. Custom Type ≠ SQL type

Un Custom Type lógico:

```text
geo.point
```

no es automáticamente:

```text
POINT
```

de MySQL/PostgreSQL u otra plataforma.

Debe mantenerse:

```text
Logical Custom Type
≠
Physical Database Type
```

---

# 4. Custom Type ≠ PHP class

También:

```text
TypeId("domain.money")
≠
App\Domain\Money
```

El FQCN puede ser parte de una representación PHP, pero no será la identidad persistente del tipo.

---

# 5. Custom Type ≠ Cast

Un cast transforma representaciones.

Un Custom Type introduce semántica de tipo.

Ejemplo:

```text
MoneyCast
```

puede mapear una clase PHP a un tipo existente.

En cambio:

```text
domain.money
```

puede introducir:

- compatibilidad;
- conversión;
- schema;
- query semantics;
- platform mappings.

---

# 6. Custom Type ≠ Value Object Mapping

Un `Value Object` puede reutilizar tipos existentes.

Ejemplo:

```text
Money
├── decimal
└── currency
```

Eso no obliga a crear un Custom Type.

Crear:

```text
domain.money
```

solo será apropiado cuando exista semántica reusable que deba ser tratada como un tipo lógico propio.

---

# 7. Custom Type ≠ Driver

Un tipo no deberá conocer:

```text
PDO
mysqli
pgsql
DriverConnection
```

directamente.

---

# 8. Custom Type ≠ Platform

El tipo define semántica lógica.

Platform define representación física.

---

# 9. Arquitectura general

```text
                    Extension Package
                           │
                           ▼
                 CustomTypeContributor
                           │
                           ▼
                  TypeRegistryBuilder
                           │
                           ▼
                   Custom Type Model
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        Type Behavior   Conversion   Compatibility
              │            │            │
              └────────────┼────────────┘
                           ▼
                  Platform Bindings
              ┌────────────┼────────────┐
              ▼            ▼            ▼
            MySQL       PostgreSQL    SQLite
                           │
                           ▼
                   Compiled Registry
                           │
                           ▼
                        Runtime
```

---

# 10. Extension layers

Un Custom Type podrá extender distintas capas:

```text
Logical Type
PHP Representation
Value Conversion
Platform Physical Mapping
Schema Mapping
Query Type Semantics
Parameter Binding Hints
Hydration
Casting
Custom Operators
Diagnostics
```

No todos los tipos deberán implementar todas.

---

# 11. Minimal Custom Type

El mínimo requerirá:

```text
TypeId
TypeDefinition
PHP representation contract
Value conversion contract
at least one supported platform mapping
```

---

# 12. Full Custom Type

Un tipo avanzado podrá añadir:

```text
Schema projection
Query operators
custom casts
index capabilities
aggregation semantics
compatibility rules
introspection mapping
```

---

# 13. Stable TypeId

Todo Custom Type tendrá:

```php
final readonly class TypeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos recomendados:

```text
app.money
w4.currency
geo.point
search.vector
network.inet
```

---

# 14. Namespacing

Los packages deberán favorecer:

```text
vendor.type
```

o:

```text
package.type
```

para reducir colisiones.

---

# 15. Reserved namespaces

VoltStack podrá reservar:

```text
voltstack.*
core.*
database.*
```

para tipos oficiales internos.

---

# 16. Application namespace

La aplicación podrá usar:

```text
app.*
domain.*
```

si así lo decide.

---

# 17. Namespace ≠ PHP namespace

```text
domain.money
```

no implica:

```php
Domain\Money
```

---

# 18. CustomTypeDefinition

```php
final readonly class CustomTypeDefinition
{
    public function __construct(
        public TypeId $id,
        public TypeFamily $family,
        public PhpTypeDescriptor $php,
        public TypeArgumentSchema $arguments,
        public TypeBehaviorReference $behavior,
        public CustomTypeVersion $version,
    ) {}
}
```

---

# 19. Type family

Un tipo podrá pertenecer a familias como:

```text
STRING
INTEGER
DECIMAL
BINARY
TEMPORAL
IDENTIFIER
STRUCTURED
SPATIAL
DOMAIN
CUSTOM
```

---

# 20. Family ≠ inheritance

`TypeFamily` expresa clasificación semántica.

No significa que:

```text
domain.email extends string
```

como herencia PHP.

---

# 21. Base logical type

Un Custom Type podrá declarar un tipo base.

Ejemplo:

```text
domain.email
    ↓
base logical type
    ↓
string
```

---

# 22. CustomTypeBase

```php
final readonly class CustomTypeBase
{
    public function __construct(
        public TypeId $type,
    ) {}
}
```

---

# 23. Base type purpose

Permite reutilizar:

- binding category;
- some compatibility rules;
- schema defaults;
- generic conversion behavior.

---

# 24. Base type ≠ alias

Muy importante:

```text
domain.email
```

no es alias de:

```text
string
```

si conserva semántica propia.

---

# 25. Alias case

Si solo se quiere otro nombre:

```text
varchar_string → string
```

usar alias.

---

# 26. Domain type

Si:

```text
domain.email
```

tiene:

- valid representation;
- semantic identity;
- custom query support;
- stricter conversion;

entonces sí es Custom Type.

---

# 27. Type contributor

```php
interface CustomTypeContributor
{
    public function contribute(
        TypeExtensionRegistryBuilder $extensions,
    ): void;
}
```

---

# 28. Bootstrap only

Contributions ocurrirán durante:

```text
application bootstrap
container compilation
test bootstrap
```

No durante queries.

---

# 29. Example contributor

```php
final class MoneyDatabaseExtension
    implements CustomTypeContributor
{
    public function contribute(
        TypeExtensionRegistryBuilder $extensions,
    ): void {
        $extensions->register(
            MoneyTypeExtension::definition(),
        );
    }
}
```

---

# 30. No runtime registration

Por defecto estará prohibido:

```php
Database::types()->register(...)
```

durante una request.

---

# 31. Why

En runtimes persistentes produciría:

- worker divergence;
- metadata invalidation;
- race conditions;
- cache inconsistency.

---

# 32. TypeExtensionDefinition

```php
final readonly class TypeExtensionDefinition
{
    public function __construct(
        public CustomTypeDefinition $type,
        public TypeBehaviorReference $behavior,
        public array $platformMappings,
        public array $queryExtensions,
        public array $schemaExtensions,
    ) {}
}
```

---

# 33. Registration pipeline

```text
Contributor
   ↓
Collect
   ↓
Normalize
   ↓
Validate
   ↓
Resolve Dependencies
   ↓
Resolve Platform Bindings
   ↓
Compile
   ↓
Fingerprint
   ↓
Freeze
```

---

# 34. Extension registry

```php
interface TypeExtensionRegistry
{
    public function get(
        TypeId $type
    ): CompiledTypeExtension;

    public function has(
        TypeId $type
    ): bool;
}
```

---

# 35. Relationship with TypeRegistry

`TypeExtensionRegistry` no deberá convertirse en una segunda fuente de verdad.

El registry final deberá presentar:

```text
TypeRegistry
    ├── Core Types
    └── Custom Types
```

---

# 36. Specialized registry

`TypeExtensionRegistry` podrá existir como infraestructura de bootstrap/diagnostics.

Pero el runtime consumidor debe resolver tipos desde:

```text
canonical TypeRegistry
```

---

# 37. Type behavior

Cada tipo podrá registrar:

```text
TypeBehavior
```

que agregue estrategias como:

```text
conversion
compatibility
canonicalization
binding hints
```

---

# 38. Behavior statelessness

Los behaviors deberán ser:

```text
immutable/stateless
```

cuando sea posible.

---

# 39. Context-dependent behavior

Cuando sea necesario:

```text
behavior
+
explicit scoped context
```

en lugar de mutable state interno.

---

# 40. Value conversion

Un Custom Type deberá poder proveer:

```php
interface CustomTypeValueConverter
{
    public function toDatabase(
        mixed $value,
        TypeReference $type,
        ValueConversionContext $context,
    ): mixed;

    public function toPhp(
        mixed $value,
        TypeReference $type,
        ValueConversionContext $context,
    ): mixed;
}
```

---

# 41. Converter input contract

El tipo deberá declarar las representaciones PHP aceptadas.

Ejemplo:

```text
domain.email

accepted:
    EmailAddress
    canonical string
```

---

# 42. Converter output contract

También deberá declarar qué produce hacia la representación lógica/base.

Ejemplo:

```text
EmailAddress
→ canonical string
```

---

# 43. Example email type

```text
TypeId:
    domain.email

PHP:
    EmailAddress|string

Base Type:
    string

Canonical Persistent:
    string

Physical Default:
    VARCHAR(...)
```

---

# 44. Custom type conversion ≠ domain validation

Aunque `domain.email` pueda validar estructura mínima, reglas como:

```text
email must belong to company domain
```

pertenecen al dominio/Validation System.

---

# 45. Strict conversion

El converter no deberá aceptar arbitrary values mediante coerción.

Nunca:

```text
object → string via accidental __toString()
```

salvo contrato explícito.

---

# 46. Platform mapping

Un Custom Type podrá tener mappings:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

independientes.

---

# 47. CustomPlatformTypeMapping

```php
interface CustomPlatformTypeMapping
{
    public function supports(
        TypeReference $type,
        DatabasePlatform $platform,
    ): bool;

    public function resolve(
        TypeReference $type,
        DatabasePlatform $platform,
    ): PhysicalTypeDescriptor;
}
```

---

# 48. Logical type stays constant

Ejemplo:

```text
geo.point
```

puede mapearse a:

```text
PostgreSQL/PostGIS → GEOMETRY(Point,...)
MySQL              → POINT
SQLite              → BLOB/TEXT/custom extension
```

pero sigue siendo:

```text
TypeId("geo.point")
```

---

# 49. Platform support status

Cada mapping deberá reportar:

```text
NATIVE
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
EMULATED
UNSUPPORTED
UNKNOWN
```

---

# 50. Unsupported ≠ fallback silently

Si `geo.point` no tiene mapping seguro en SQLite:

```text
UNSUPPORTED
```

es preferible a convertirlo arbitrariamente en string.

---

# 51. Explicit fallback

Una extensión podrá declarar:

```text
geo.point
→ JSON
```

como fallback solo si:

- semántica está definida;
- conversión es conocida;
- query limitations son explícitas.

---

# 52. Fallback capability report

Debe indicar:

```text
Storage:
    supported

Spatial Query:
    unavailable

Spatial Index:
    unavailable
```

---

# 53. Schema integration

Un Custom Type podrá proyectar:

```text
logical type
→ schema descriptor
```

---

# 54. Schema projector

```php
interface CustomTypeSchemaProjector
{
    public function project(
        TypeReference $type,
        SchemaProjectionContext $context,
    ): TypeSchemaProjection;
}
```

---

# 55. Schema projection ≠ SQL

El projector no generará:

```sql
CREATE TYPE ...
```

directamente.

Producirá:

```text
Schema AST / semantic schema operations
```

---

# 56. Custom native types

Algunos tipos requieren objetos DB adicionales.

Ejemplo PostgreSQL:

```text
CREATE TYPE ...
```

o extensions:

```text
CREATE EXTENSION ...
```

---

# 57. Auxiliary schema objects

Una extensión podrá declarar:

```text
required extension
custom DB type
check constraint
generated column
index
```

como semantic schema requirements.

---

# 58. AuxiliarySchemaRequirement

```php
interface AuxiliarySchemaRequirement
{
    public function identifier(): string;

    public function dependencies(): array;
}
```

---

# 59. Ownership

Schema System decide:

- planning;
- ordering;
- migration;
- execution.

Custom Type solo declara requirements.

---

# 60. Introspection

Un Custom Type podrá registrar un physical type recognizer.

---

# 61. Physical recognition

Ejemplo:

```text
PostgreSQL:
    vector(1536)
```

puede mapearse a:

```text
search.vector(dimensions=1536)
```

---

# 62. CustomTypeIntrospector

```php
interface CustomTypeIntrospector
{
    public function inspect(
        PhysicalTypeMetadata $physical,
        DatabasePlatform $platform,
    ): CustomTypeIntrospectionResult;
}
```

---

# 63. Introspection confidence

Resultado deberá incluir:

```text
EXACT
HIGH
PARTIAL
AMBIGUOUS
UNKNOWN
```

o equivalente.

---

# 64. Ambiguous physical type

Ejemplo:

```text
VARCHAR(255)
```

no demuestra:

```text
domain.email
```

aunque `domain.email` se persista así.

---

# 65. Rule

> **Un physical representation compartido por múltiples logical types no será suficiente para reconstruir automáticamente la intención lógica.**

---

# 66. Schema diff

Custom Type podrá participar en:

```text
type comparison
parameter comparison
physical strategy comparison
```

---

# 67. Type migration

Ejemplo:

```text
search.vector(768)
→
search.vector(1536)
```

puede requerir:

- data migration;
- index rebuild;
- model regeneration.

---

# 68. Custom migration analyzer

```php
interface CustomTypeMigrationAnalyzer
{
    public function analyze(
        TypeReference $from,
        TypeReference $to,
        MigrationContext $context,
    ): TypeMigrationAssessment;
}
```

---

# 69. Migration analyzer ≠ executor

No ejecutará migration.

Solo clasificará:

```text
SAFE
REQUIRES_DATA_MIGRATION
REQUIRES_REBUILD
LOSSY
UNSUPPORTED
UNKNOWN
```

---

# 70. Query Type System integration

Custom Types deberán poder declarar:

```text
equality
ordering
comparison
arithmetic
containment
hashability
aggregation
```

según aplique.

---

# 71. CustomTypeCapabilities

```php
final readonly class CustomTypeCapabilities
{
    public function __construct(
        public bool $equalityComparable,
        public bool $orderable,
        public bool $hashable,
        public bool $groupable,
        public bool $indexable,
    ) {}
}
```

---

# 72. Example vector

Un vector puede ser:

```text
equality:
    maybe

orderable:
    no

distance operators:
    yes

indexable:
    platform-specific
```

---

# 73. Query operators

La extensión podrá registrar operadores semánticos.

Ejemplo:

```text
vector_distance(a, b)
```

---

# 74. Custom query operator

```php
interface CustomTypeQueryOperator
{
    public function id(): QueryOperatorId;

    public function signature(): QueryOperatorSignature;

    public function inferResultType(
        array $operandTypes,
    ): TypeReference;
}
```

---

# 75. Operator ≠ SQL function

El operador lógico:

```text
VECTOR_DISTANCE
```

puede compilarse diferente por plataforma.

---

# 76. Operator registration

```text
Custom Type
     ↓
Query Extension Registry
     ↓
Semantic Operator
     ↓
Platform Compiler Extension
```

---

# 77. No SQL in model API

No deberá requerirse:

```php
->whereRaw('vector_distance(...)')
```

como API principal.

---

# 78. Compiler extension

Una plataforma podrá registrar:

```php
interface CustomTypeSqlCompilerExtension
{
    public function supports(
        QueryNode $node,
        DatabasePlatform $platform,
    ): bool;

    public function compile(
        QueryNode $node,
        CompilerContext $context,
    ): SqlFragment;
}
```

---

# 79. Compiler extension scope

Solo debe conocer:

- semantic AST;
- Platform;
- compiler context.

No:

- EntityManager;
- HTTP request;
- current model instance.

---

# 80. Query planner integration

Algunas operaciones custom pueden tener strategies.

Ejemplo vector search:

```text
exact distance
approximate index search
```

---

# 81. Planner extension

Podrá existir:

```php
interface CustomTypePlannerExtension
{
    public function plan(
        QuerySemanticNode $node,
        PlannerContext $context,
    ): QueryPlanFragment;
}
```

---

# 82. Planner extension optional

Tipos simples como:

```text
domain.email
```

no necesitan extender Planner.

---

# 83. Query optimization

Un Custom Type podrá registrar reglas específicas solo cuando estén aisladas y verificables.

---

# 84. Optimizer safety

Nunca permitir que un plugin reescriba arbitrariamente todo Query AST sin boundary.

---

# 85. Scoped optimizer extension

Preferir:

```text
operator-specific optimizer rule
```

sobre:

```text
global optimizer hook
```

sin restricciones.

---

# 86. Parameter binding

Custom Type podrá proporcionar:

```text
binding category
```

o hints.

---

# 87. Binding hint

Ejemplos:

```text
STRING
INTEGER
BINARY
LOB
PLATFORM_NATIVE
```

---

# 88. Custom binding ≠ direct driver access

Un tipo no deberá ejecutar:

```php
$statement->bindValue(...)
```

directamente.

---

# 89. Correct boundary

```text
Custom Type
→ BindingDescriptor
→ Parameter Binding System
→ Driver
```

---

# 90. BindingDescriptor

```php
final readonly class TypeBindingDescriptor
{
    public function __construct(
        public BindingCategory $category,
        public array $hints = [],
    ) {}
}
```

---

# 91. Platform-native binding

Algunas extensiones podrán requerir type metadata adicional.

Eso deberá pasar como:

```text
safe structured hint
```

no raw SQL.

---

# 92. Hydration

Custom Types participarán mediante:

```text
TypeReference
→ Compiled Conversion Pipeline
→ HydrationPlan
```

---

# 93. No custom hydrator by default

Un tipo scalar/custom no necesitará un Hydrator propio.

Value Conversion será suficiente.

---

# 94. Custom hydration extension

Solo cuando el resultado requiera estructuras especiales podrá registrar:

```text
Hydration Adapter
```

limitado.

---

# 95. Hydration Adapter ≠ Entity Hydrator

No deberá crear entities ni manipular IdentityMap.

---

# 96. Casting

Un Custom Type podrá ofrecer convenience casts.

Ejemplo:

```text
domain.email
→ EmailAddress
```

---

# 97. Type behavior first

Si la representación PHP natural del tipo ya es `EmailAddress`, no deberá necesitar un cast adicional artificial.

---

# 98. Cast when useful

Puede ser útil cuando:

```text
Logical Type:
    json

Application representation:
    Preferences
```

sin crear un nuevo logical type.

---

# 99. Decision rule

Crear Custom Type cuando la semántica pertenece al Database Type System.

Crear Cast cuando solo cambia la representación application-facing.

---

# 100. Custom type dependencies

Un tipo podrá depender de:

```text
base type
other custom type
platform extension
query extension
```

---

# 101. Dependency descriptor

```php
final readonly class CustomTypeDependency
{
    public function __construct(
        public TypeId|string $dependency,
        public CustomTypeDependencyKind $kind,
    ) {}
}
```

---

# 102. Dependency kinds

```text
LOGICAL_TYPE
PLATFORM_CAPABILITY
DATABASE_EXTENSION
QUERY_EXTENSION
CAST_EXTENSION
```

---

# 103. Dependency validation

Ejemplo:

```text
search.vector
requires:
    PostgreSQL pgvector capability
```

si se utiliza native mapping.

---

# 104. Optional dependency

Podrá existir:

```text
native path:
    pgvector

fallback:
    JSON
```

si la extensión lo declara.

---

# 105. Missing dependency

No deberá descubrirse cuando llegue la primera query.

Debe detectarse durante:

```text
bootstrap
schema validation
deployment diagnostics
```

cuando sea posible.

---

# 106. Extension version

Cada Custom Type podrá declarar versión semántica propia.

```php
final readonly class CustomTypeVersion
{
    public function __construct(
        public string $version,
    ) {}
}
```

---

# 107. Type version ≠ package version

Un package podría cambiar otras cosas sin alterar la semántica del tipo.

---

# 108. Type compatibility versioning

Cambiar:

```text
domain.money v1
```

a:

```text
domain.money v2
```

puede alterar:

- conversion;
- equality;
- schema;
- parameter semantics.

Debe considerarse cuidadosamente.

---

# 109. Breaking semantic change

Ejemplo:

```text
money amount:
    decimal(19,4)
```

se cambia a:

```text
integer minor units
```

Eso no es un simple refactor interno.

---

# 110. Type evolution

Deberá integrarse con:

```text
Database Versioning
Migration
Backward Compatibility
```

posteriormente.

---

# 111. Registry generation

Cambios en Custom Types deberán afectar:

```text
TypeRegistryGeneration
```

---

# 112. Cache invalidation

Deben invalidarse artefactos dependientes:

```text
ORM metadata
Hydration plans
Query plans
Compiled casts
Schema metadata cache
Conversion pipelines
```

cuando sus fingerprints dependan del tipo modificado.

---

# 113. CustomTypeFingerprint

```text
Hash(
    TypeId
    + version
    + definition
    + arguments
    + behavior bindings
    + platform mappings
    + query semantics
    + dependencies
)
```

---

# 114. Deterministic fingerprint

No incluir:

```text
timestamps
object IDs
random UUIDs
memory addresses
```

---

# 115. Override policy

Extensiones no deberán sobrescribir tipos existentes silenciosamente.

---

# 116. Default override

```text
FORBID
```

---

# 117. Core override

Cambiar:

```text
decimal
```

desde un plugin estará prohibido por default.

---

# 118. Explicit extension override

Si se soporta, deberá:

- estar habilitado explícitamente;
- pasar compatibility analysis;
- producir diagnostics.

---

# 119. Better alternative

En la mayoría de casos, crear:

```text
app.decimal
```

o:

```text
domain.money
```

será preferible a alterar `decimal`.

---

# 120. Conflict resolution

Si dos packages registran:

```text
search.vector
```

el bootstrap deberá fallar.

Nunca:

```text
last registration wins
```

---

# 121. Origin metadata

Diagnostics deberá conservar:

```text
package
provider
extension name
version
source
```

---

# 122. Plugin isolation

Un package no podrá:

- remover otros tipos;
- alterar aliases ajenos;
- mutar definitions post-freeze.

---

# 123. Extension sandboxing conceptual

Aunque PHP comparta proceso, los contratos deberán reducir el alcance de una extensión a los hooks explícitos.

---

# 124. Security boundary

Custom Types son código ejecutable.

Por tanto, solo deberán registrarse desde:

```text
trusted application/package configuration
```

no desde datos externos.

---

# 125. Never from DB value

Nunca:

```text
DB value:
    "App\\Type\\Foo"

→ instantiate custom type
```

---

# 126. Never from HTTP input

Nunca:

```text
?database_type=Vendor\Type\Whatever
```

→ dynamic class loading.

---

# 127. TypeId input

Incluso si un usuario externo envía:

```text
domain.email
```

registry membership no significa autorización para utilizarlo.

---

# 128. Unsafe serialization

Custom Types no deberán usar:

```php
unserialize($dbValue)
```

por default.

---

# 129. Arbitrary code execution

Ningún converter podrá evaluar:

```text
eval
dynamic PHP
SQL from value
```

como comportamiento core permitido.

---

# 130. Resource governance

Custom converters deberán respetar límites del Value Conversion System.

---

# 131. Custom resource policy

Una extensión podrá declarar límites adicionales:

```text
max dimensions
max bytes
max nesting
max precision
max element count
```

---

# 132. Example vector

```text
vector(dimensions=1536)
```

deberá validar:

```text
dimension count
numeric values
finite numbers
memory budget
```

---

# 133. Resource declaration

```php
interface CustomTypeResourcePolicy
{
    public function validateDefinition(
        TypeReference $type,
    ): void;

    public function validateValue(
        mixed $value,
        TypeReference $type,
    ): void;
}
```

---

# 134. Persistent runtime

Compartible:

```text
Custom Type Definitions
Compiled Platform Mappings
Compiled Conversion Pipelines
Stateless Behaviors
Query Operator Definitions
```

---

# 135. Scoped

```text
conversion context
temporary buffers
resource counters
diagnostic state
```

---

# 136. FrankenPHP

```text
Worker
├── Frozen Type Registry
├── Frozen Extension Registry
├── Shared Stateless Behaviors
│
├── Request A scoped state
└── Request B scoped state
```

---

# 137. RoadRunner

Mismo patrón.

---

# 138. OpenSwoole

Los extension services compartidos deberán ser concurrency-safe o stateless.

---

# 139. Forbidden mutable shared state

Evitar:

```php
final class MyType
{
    public mixed $lastValue;
}
```

como singleton.

---

# 140. Development reload

Cambiar un Custom Type deberá producir:

```text
new registry generation
```

en lugar de modificar la generación existente.

---

# 141. Extension initialization

No deberá ejecutar trabajo pesado por cada request.

---

# 142. Expensive compilation

Operaciones como:

```text
parse schema
build query operator maps
compile conversion chains
```

deberán hacerse durante bootstrap.

---

# 143. Diagnostics API

Conceptualmente:

```php
Database::types()
    ->extensions()
    ->explain('search.vector');
```

---

# 144. Diagnostic output

```text
CUSTOM DATABASE TYPE

Type:
    search.vector

Origin:
    voltstack/vector

Version:
    1.2

Family:
    STRUCTURED

PHP Type:
    Vector

Parameters:
    dimensions: 1536

Base Type:
    none

Platforms:
    PostgreSQL
        NATIVE

    MySQL
        UNSUPPORTED

    MariaDB
        UNSUPPORTED

    SQLite
        EMULATED

Query Operators:
    VECTOR_DISTANCE
    VECTOR_COSINE_DISTANCE

Indexing:
    platform-specific

Binding:
    PLATFORM_NATIVE

Persistent Runtime:
    SAFE

Registry Generation:
    dbtypes:...
```

---

# 145. Explain dependencies

```text
DEPENDENCIES

search.vector
├── PostgreSQL capability: pgvector
├── query operator: VECTOR_DISTANCE
└── custom compiler: PostgreSQLVectorCompiler
```

---

# 146. Portability report

```text
TYPE PORTABILITY

Type:
    search.vector(1536)

Targets:
    PostgreSQL: NATIVE
    MySQL: UNSUPPORTED
    MariaDB: UNSUPPORTED
    SQLite: EMULATED

Semantic Portability:
    PARTIAL

Query Portability:
    LOW

Storage Portability:
    MEDIUM
```

---

# 147. Telemetry

Métricas posibles:

```text
database.custom_type.registered
database.custom_type.compile_failure
database.custom_type.conversion
database.custom_type.conversion_failure
database.custom_type.platform_fallback
database.custom_type.unsupported_platform
database.custom_type.query_operator
database.custom_type.resource_rejection
```

---

# 148. No per-value type labels uncontrolled

`TypeId` es bounded por registry y puede utilizarse con cuidado.

No incluir:

```text
raw value
dynamic argument
user content
```

en labels.

---

# 149. Tracing

No crear span por cada conversión custom.

Sí se podrán agregar atributos a spans de:

```text
query
hydration
schema compilation
```

cuando sea útil.

---

# 150. Slow custom converter detection

Profiler podrá detectar:

```text
custom type conversion taking unusually long
```

sin que Type System dependa del Profiler.

---

# 151. Error hierarchy

```text
DatabaseCustomTypeException
├── CustomTypeRegistrationException
├── DuplicateCustomTypeException
├── InvalidCustomTypeDefinitionException
├── CustomTypeDependencyException
├── MissingCustomTypeDependencyException
├── CustomTypeConflictException
├── ForbiddenCustomTypeOverrideException
├── CustomTypeConversionException
├── CustomTypePlatformMappingException
├── UnsupportedCustomTypePlatformException
├── CustomTypeSchemaException
├── CustomTypeIntrospectionException
├── CustomTypeQueryExtensionException
├── CustomTypeCompilerExtensionException
├── CustomTypeBindingException
├── CustomTypeMigrationException
├── CustomTypeResourceLimitException
├── CustomTypeSecurityException
├── CustomTypeRuntimeStateException
└── CustomTypeInvariantViolationException
```

---

# 152. Registration diagnostics

Ejemplo:

```text
CUSTOM TYPE REGISTRATION FAILURE

Type:
    domain.money

Origin:
    app

Problem:
    Missing dependency

Dependency:
    domain.currency

Resolution:
    Register domain.currency before final registry compilation.
```

---

# 153. Platform diagnostics

```text
CUSTOM TYPE PLATFORM FAILURE

Type:
    geo.point

Platform:
    SQLite

Requested Capability:
    spatial-index

Status:
    UNSUPPORTED

Fallback:
    none
```

---

# 154. Testing architecture

Todo Custom Type deberá poder ejecutar un:

```text
CustomTypeConformanceSuite
```

---

# 155. Minimal conformance

Debe validar:

```text
registration
TypeId uniqueness
value conversion
round-trip
nullability
platform mapping
binding
persistent runtime safety
```

---

# 156. Extended conformance

Cuando aplique:

```text
schema create
schema introspection
schema diff
query operators
query compilation
index support
migration behavior
```

---

# 157. CustomTypeConformanceContract

```php
interface CustomTypeConformanceContract
{
    public function logicalType(): TypeReference;

    public function validValues(): iterable;

    public function invalidValues(): iterable;

    public function supportedPlatforms(): iterable;
}
```

---

# 158. Round-trip test

Para mapping lossless:

```text
PHP Value
→ Database Representation
→ PHP Value
```

deberá conservar equivalencia semántica.

---

# 159. Platform round-trip

Debe probarse por cada plataforma declarada como compatible.

---

# 160. Unsupported-platform test

Un platform no soportado deberá producir:

```text
explicit unsupported result
```

no fallback accidental.

---

# 161. Binding test

Verificar:

```text
custom value
→ conversion
→ binding
→ execute
→ read
→ conversion
```

---

# 162. Query operator test

Para cada operador custom:

```text
type check
AST creation
planner compatibility
compiler output
result typing
```

---

# 163. Schema tests

Probar:

```text
create
introspect
diff
migration
rollback feasibility
```

cuando el tipo declare integración de schema.

---

# 164. Migration test

Cambios de argumentos deben evaluarse.

Ejemplo:

```text
vector(768)
→ vector(1536)
```

---

# 165. Persistent runtime test

Request A y B no compartirán estado mutable del converter.

---

# 166. Concurrency test

Custom behaviors shared deberán ser thread/coroutine safe dentro del modelo del runtime.

---

# 167. Security test

Inputs externos no deben provocar:

```text
class loading
converter loading
SQL injection
schema identifier injection
```

---

# 168. Fuzz tests

Muy recomendables para custom parsers.

Ejemplo:

```text
geo coordinates
vector payload
network addresses
domain identifiers
```

---

# 169. Performance tests

Medir:

```text
conversion/sec
binding overhead
hydration overhead
query operator compilation
schema compilation
```

---

# 170. Custom type certification

VoltStack podría ofrecer una futura herramienta:

```text
volt database:type:test vendor.type
```

---

# 171. CLI validation

Posibles comandos:

```text
volt database:type:list
volt database:type:show search.vector
volt database:type:validate
volt database:type:test search.vector
volt database:type:portability search.vector
```

---

# 172. Developer experience

API propuesta:

```php
DatabaseTypes::extend(
    new MoneyTypeExtension()
);
```

durante bootstrap/configuración.

---

# 173. Attribute mapping

```php
#[Column(type: 'domain.money')]
private Money $price;
```

---

# 174. Parameterized Custom Type

```php
#[Column(
    type: 'search.vector',
    arguments: [
        'dimensions' => 1536,
    ],
)]
private Vector $embedding;
```

---

# 175. Type argument schema

Todo argumento deberá definirse estructuralmente.

---

# 176. CustomTypeArgumentDefinition

```php
final readonly class CustomTypeArgumentDefinition
{
    public function __construct(
        public string $name,
        public TypeArgumentValueType $type,
        public bool $required,
        public mixed $default = null,
    ) {}
}
```

---

# 177. No arbitrary argument bags

Evitar:

```php
array<string, mixed>
```

sin validation en hot path.

---

# 178. Argument normalization

Ejemplo:

```text
dimensions="1536"
```

podrá canonicalizarse a integer durante metadata compilation si la policy lo permite.

---

# 179. Argument fingerprint

`TypeReferenceFingerprint` deberá incluir argumentos normalizados.

---

# 180. Example Money Type

Arquitectura:

```text
domain.money
│
├── PHP:
│   Money
│
├── Logical Components:
│   amount
│   currency
│
├── Storage:
│   custom strategy
│
└── Query:
    equality
```

Sin embargo, antes de crear un Custom Type compuesto deberá considerarse si:

```text
Value Object Mapping
```

es suficiente.

---

# 181. Money decision

Si `Money` necesita dos columnas:

```text
amount
currency
```

el mecanismo preferido será normalmente:

```text
Value Object Mapping
```

del documento 160.

---

# 182. Custom Money Type appropriate when

Podría ser apropiado si la plataforma dispone de:

```text
native money-like type
```

y se desea una abstracción lógica única con fallbacks cuidadosamente definidos.

---

# 183. Example Email Type

```php
final class EmailTypeExtension
{
    public static function definition(): TypeExtensionDefinition
    {
        // logical: domain.email
        // base: string
        // PHP: EmailAddress
    }
}
```

Storage:

```text
VARCHAR
```

Logical semantics:

```text
EmailAddress
```

---

# 184. Example UUID subtype

Una aplicación podría crear:

```text
domain.user_id
```

base:

```text
uuid
```

representación PHP:

```text
UserId
```

---

# 185. Domain identifier benefit

Esto permite que:

```text
UserId
```

y:

```text
OrderId
```

sean distintos aunque ambos utilicen UUID.

---

# 186. Query compatibility

Por default:

```text
domain.user_id
≠
domain.order_id
```

aunque su base sea UUID.

---

# 187. Explicit coercion

Si alguna operación requiere convertir entre ellos, deberá ser explícita.

---

# 188. Type safety

Esto puede prevenir:

```php
$orderRepository->find($userId);
```

a nivel metadata/query type semantics cuando se integre profundamente.

---

# 189. Example network type

```text
network.ip
```

podrá aceptar:

```text
IPv4
IPv6
```

y mapear:

```text
PostgreSQL → inet
Others     → canonical string/binary
```

---

# 190. Semantic equality

`IPv6` textual representations diferentes pueden representar la misma dirección.

El Custom Type deberá definir canonicalization.

---

# 191. Custom comparator

```php
interface CustomTypeComparator
{
    public function equivalent(
        mixed $left,
        mixed $right,
        TypeComparisonContext $context,
    ): bool;
}
```

---

# 192. Dirty tracking integration

UoW podrá utilizar:

```text
Custom Type canonical value
```

o comparator semántico.

---

# 193. No raw string dirty comparison

Ejemplo IP:

```text
2001:0db8::1
```

vs:

```text
2001:db8::1
```

pueden ser equivalentes.

---

# 194. Equality metadata

Custom Type deberá declarar si:

```text
canonical equality
```

es suficiente o requiere comparator.

---

# 195. Hashability

Si un tipo es usado en:

```text
DISTINCT
GROUP BY
hash-based cache identity
```

deberá declarar hash semantics compatibles.

---

# 196. Ordering

No todos los tipos son orderable.

---

# 197. Type compatibility

Custom Types deberán poder aportar reglas:

```php
interface CustomTypeCompatibilityRule
{
    public function compare(
        TypeReference $source,
        TypeReference $target,
    ): TypeCompatibilityResult;
}
```

---

# 198. Compatibility levels

```text
EXACT
ASSIGNABLE
COERCIBLE_EXPLICITLY
LOSSY
INCOMPATIBLE
UNKNOWN
```

---

# 199. No implicit base coercion

Si:

```text
domain.email
base = string
```

no significa automáticamente que cualquier string sea un email válido.

---

# 200. Assignment rule

Podrá permitir:

```text
domain.email → string
```

para lectura/serialization interna.

Pero:

```text
string → domain.email
```

deberá pasar validation/conversion.

---

# 201. Extension package structure

Propuesta:

```text
src/Quantum/Database/Type/Extension/
│
├── Contract/
│   ├── CustomTypeContributor.php
│   ├── CustomPlatformTypeMapping.php
│   ├── CustomTypeSchemaProjector.php
│   ├── CustomTypeIntrospector.php
│   ├── CustomTypeMigrationAnalyzer.php
│   ├── CustomTypeQueryOperator.php
│   ├── CustomTypePlannerExtension.php
│   ├── CustomTypeSqlCompilerExtension.php
│   ├── CustomTypeCompatibilityRule.php
│   ├── CustomTypeComparator.php
│   └── CustomTypeResourcePolicy.php
│
├── Definition/
│   ├── CustomTypeDefinition.php
│   ├── TypeExtensionDefinition.php
│   ├── CustomTypeVersion.php
│   ├── CustomTypeBase.php
│   ├── CustomTypeCapabilities.php
│   ├── CustomTypeArgumentDefinition.php
│   └── CustomTypeDependency.php
│
├── Registry/
│   ├── TypeExtensionRegistry.php
│   ├── TypeExtensionRegistryBuilder.php
│   ├── DefaultTypeExtensionRegistry.php
│   └── CustomTypeFingerprint.php
│
├── Compiler/
│   ├── TypeExtensionCompiler.php
│   ├── CompiledTypeExtension.php
│   ├── PlatformMappingCompiler.php
│   └── QueryExtensionCompiler.php
│
├── Conversion/
│   ├── CustomTypeValueConverter.php
│   └── CustomConversionBehaviorAdapter.php
│
├── Platform/
│   ├── CustomPhysicalTypeDescriptor.php
│   └── CustomPlatformMappingResolver.php
│
├── Query/
│   ├── CustomTypeQueryOperatorRegistry.php
│   ├── QueryOperatorSignature.php
│   └── QueryOperatorId.php
│
├── Schema/
│   ├── TypeSchemaProjection.php
│   └── AuxiliarySchemaRequirement.php
│
├── Binding/
│   ├── TypeBindingDescriptor.php
│   └── BindingCategory.php
│
├── Compatibility/
│   ├── TypeCompatibilityResult.php
│   └── CustomCompatibilityAnalyzer.php
│
├── Diagnostics/
│   ├── CustomTypeExplainer.php
│   ├── CustomTypePortabilityReport.php
│   └── CustomTypeDiagnosticReport.php
│
├── Testing/
│   ├── CustomTypeConformanceSuite.php
│   └── CustomTypeConformanceContract.php
│
└── Exception/
    ├── DatabaseCustomTypeException.php
    ├── CustomTypeRegistrationException.php
    ├── DuplicateCustomTypeException.php
    ├── InvalidCustomTypeDefinitionException.php
    ├── CustomTypeDependencyException.php
    ├── MissingCustomTypeDependencyException.php
    ├── CustomTypeConflictException.php
    ├── ForbiddenCustomTypeOverrideException.php
    ├── CustomTypeConversionException.php
    ├── CustomTypePlatformMappingException.php
    ├── UnsupportedCustomTypePlatformException.php
    ├── CustomTypeSchemaException.php
    ├── CustomTypeQueryExtensionException.php
    ├── CustomTypeBindingException.php
    ├── CustomTypeResourceLimitException.php
    ├── CustomTypeSecurityException.php
    └── CustomTypeInvariantViolationException.php
```

---

# 202. Package-level example

```text
voltstack/vector
│
├── VectorTypeExtension
├── VectorValueConverter
├── VectorTypeComparator
├── PostgreSQLVectorMapping
├── SQLiteVectorFallback
├── VectorDistanceOperator
├── PostgreSQLVectorCompiler
├── VectorSchemaProjector
└── VectorConformanceTests
```

---

# 203. Bootstrap example

```php
final class VectorDatabaseServiceProvider
{
    public function databaseTypes(
        TypeExtensionRegistryBuilder $types,
    ): void {
        $types->register(
            VectorTypeExtension::definition()
        );
    }
}
```

---

# 204. Application usage

```php
final class DocumentEmbedding
{
    #[Column(
        type: 'search.vector',
        arguments: [
            'dimensions' => 1536,
        ],
    )]
    private Vector $embedding;
}
```

---

# 205. Query example

Developer-facing:

```php
Document::query()
    ->orderByVectorDistance(
        'embedding',
        $queryVector,
    )
    ->limit(10)
    ->get();
```

Internamente:

```text
VectorDistanceExpression
        ↓
Query AST
        ↓
Semantic Type Check
        ↓
Planner
        ↓
Platform Compiler Extension
        ↓
SQL
```

---

# 206. No vendor leak

La application API no deberá necesitar:

```text
<-> operator
VECTOR_DISTANCE(...)
pgvector-specific SQL
```

salvo escape hatch explícito.

---

# 207. Degraded platform

En plataforma sin vector-native:

```text
Storage:
    possible

Nearest-neighbor query:
    unsupported
```

debe informarse.

---

# 208. Custom type capability matrix

| Capability | Ejemplo |
|---|---|
| Equality | `domain.email` |
| Ordering | `domain.money_amount` |
| Hashing | `network.ip` |
| Aggregation | custom numeric type |
| Indexing | `search.vector` |
| Native storage | PostgreSQL vector |
| Portable fallback | string/JSON/binary |
| Partial support | storage only |
| Query operators | vector distance |
| Schema objects | extensions/custom types |

---

# 209. Extension lifecycle

```text
DISCOVERED
    ↓
REGISTERED
    ↓
VALIDATED
    ↓
DEPENDENCIES_RESOLVED
    ↓
COMPILED
    ↓
FROZEN
```

Error:

```text
FAILED
```

No partial publication.

---

# 210. Registration order

Contributors deberán ordenarse determinísticamente.

---

# 211. Dependency order

Si:

```text
domain.user_id
→ uuid
```

UUID deberá existir antes de completar compilation.

No necesariamente antes de registrar el descriptor temporalmente.

---

# 212. Dependency graph

```text
Type Extensions
      ↓
Dependency Graph
      ↓
Cycle Detection
      ↓
Topological Compilation
```

cuando aplique.

---

# 213. Cycles

Ciclos de comportamiento no soportados serán rechazados.

---

# 214. No arbitrary recursive types

Tipos recursive estructurados requerirán un sistema específico y no se resolverán mediante Custom Type dependencies genéricas sin límites.

---

# 215. Extension constraints

Un Custom Type no deberá:

- ejecutar queries durante registration;
- depender de current tenant;
- depender de current request;
- modificar global DB config.

---

# 216. Bootstrap environmental checks

Sí podrá comprobar:

```text
required PHP extension
required package version
required database capability metadata
```

cuando estén disponibles sin I/O destructivo.

---

# 217. DB capability validation

Al conectar/deployar podrá verificarse:

```text
required native extension exists
```

mediante health/deployment tooling.

No deberá ocurrir por cada query.

---

# 218. Failure modes

Distinguir:

```text
TYPE_REGISTERED_BUT_PLATFORM_UNSUPPORTED
TYPE_PLATFORM_SUPPORTED_BUT_DB_EXTENSION_MISSING
TYPE_VALUE_INVALID
TYPE_QUERY_OPERATOR_UNSUPPORTED
TYPE_SCHEMA_MAPPING_UNKNOWN
```

---

# 219. Correctness principle

Un Custom Type conocido no implica que todas sus operaciones estén soportadas.

---

# 220. Operation-specific capabilities

Ejemplo:

```text
search.vector

storage:
    supported

equality:
    supported

distance:
    supported

ANN index:
    unsupported
```

---

# 221. Query fallback policy

No deberá emular operaciones caras automáticamente.

Ejemplo:

```text
nearest-neighbor
```

sobre miles de vectores en PHP después de cargar todos los rows sería una mala emulación implícita.

---

# 222. Explicit emulation

Si existe una emulación, deberá declarar:

```text
cost
limitations
resource policy
semantic equivalence
```

---

# 223. No hidden application-side query execution

Custom Type no deberá descargar todos los datos para simular una operación SQL sin autorización del Planner/Resource Governance.

---

# 224. Schema compatibility

La extensión deberá poder explicar:

```text
Can TypeReference X be represented on Platform P?
```

separadamente de:

```text
Does current schema already represent X?
```

---

# 225. Mapping compatibility

```text
logical representability
≠
observed schema compatibility
```

---

# 226. Deployment readiness

Diagnostics podrá verificar:

```text
type registered
dependencies present
platform supported
schema compatible
conversion tested
```

---

# 227. Extension health report

```text
CUSTOM TYPE HEALTH

Type:
    search.vector

Registration:
    OK

Conversion:
    OK

Platform:
    PostgreSQL

DB Extension:
    AVAILABLE

Schema:
    COMPATIBLE

Query Operators:
    4/4 supported

Indexes:
    supported

Status:
    READY
```

---

# 228. Architectural invariants

## DB-CUSTOM-TYPE-001
Custom Types ampliarán el Type System mediante contratos explícitos.

## DB-CUSTOM-TYPE-002
Custom TypeId será identidad lógica estable.

## DB-CUSTOM-TYPE-003
Custom TypeId no será FQCN.

## DB-CUSTOM-TYPE-004
Custom Type será distinto de SQL physical type.

## DB-CUSTOM-TYPE-005
Custom Type será distinto de Cast.

## DB-CUSTOM-TYPE-006
Custom Type será distinto de Value Object Mapping.

## DB-CUSTOM-TYPE-007
Custom Type será distinto de Driver.

## DB-CUSTOM-TYPE-008
Custom Type será distinto de Platform.

## DB-CUSTOM-TYPE-009
Custom Types se registrarán durante bootstrap.

## DB-CUSTOM-TYPE-010
Runtime dynamic type registration estará prohibido por default.

## DB-CUSTOM-TYPE-011
Custom Type registrations serán deterministas.

## DB-CUSTOM-TYPE-012
Conflicting TypeIds serán rechazados.

## DB-CUSTOM-TYPE-013
Last-write-wins estará prohibido.

## DB-CUSTOM-TYPE-014
Core type override estará prohibido por default.

## DB-CUSTOM-TYPE-015
Override explícito requerirá compatibility analysis.

## DB-CUSTOM-TYPE-016
Custom Type podrá tener base logical type.

## DB-CUSTOM-TYPE-017
Base type no convertirá custom type en alias.

## DB-CUSTOM-TYPE-018
Aliases se usarán solo para identidad equivalente.

## DB-CUSTOM-TYPE-019
Type arguments serán estructurados.

## DB-CUSTOM-TYPE-020
Type arguments serán validados antes del runtime normal.

## DB-CUSTOM-TYPE-021
Type argument normalization será determinista.

## DB-CUSTOM-TYPE-022
Value converters se registrarán explícitamente.

## DB-CUSTOM-TYPE-023
Value converters no harán arbitrary coercion.

## DB-CUSTOM-TYPE-024
Custom converters no accederán al Driver.

## DB-CUSTOM-TYPE-025
Custom converters no accederán a EntityManager.

## DB-CUSTOM-TYPE-026
Custom converters no ejecutarán queries.

## DB-CUSTOM-TYPE-027
Custom conversion state mutable será scoped.

## DB-CUSTOM-TYPE-028
Platform mappings serán independientes por plataforma.

## DB-CUSTOM-TYPE-029
MySQL y MariaDB no se tratarán automáticamente como el mismo mapping.

## DB-CUSTOM-TYPE-030
Unsupported platform será explícito.

## DB-CUSTOM-TYPE-031
Fallback no será seleccionado silenciosamente.

## DB-CUSTOM-TYPE-032
Fallback deberá declarar semantic limitations.

## DB-CUSTOM-TYPE-033
Schema projection no generará SQL directamente.

## DB-CUSTOM-TYPE-034
Schema System seguirá controlando migration planning.

## DB-CUSTOM-TYPE-035
Auxiliary schema objects serán semantic requirements.

## DB-CUSTOM-TYPE-036
Custom Type no ejecutará CREATE TYPE directamente.

## DB-CUSTOM-TYPE-037
Introspection podrá registrar custom recognizers.

## DB-CUSTOM-TYPE-038
Ambiguous physical type no implicará custom logical intent.

## DB-CUSTOM-TYPE-039
VARCHAR físico no implicará domain.email.

## DB-CUSTOM-TYPE-040
Introspection preservará uncertainty.

## DB-CUSTOM-TYPE-041
Custom migrations serán analizadas explícitamente.

## DB-CUSTOM-TYPE-042
Migration analyzer no ejecutará migrations.

## DB-CUSTOM-TYPE-043
Query capabilities serán declaradas.

## DB-CUSTOM-TYPE-044
Orderability no se asumirá.

## DB-CUSTOM-TYPE-045
Equality no se asumirá.

## DB-CUSTOM-TYPE-046
Hashability no se asumirá.

## DB-CUSTOM-TYPE-047
Custom query operators serán semantic operators.

## DB-CUSTOM-TYPE-048
Custom operators no serán SQL strings.

## DB-CUSTOM-TYPE-049
Query Builder no contendrá vendor SQL del Custom Type.

## DB-CUSTOM-TYPE-050
Compiler extensions serán platform-scoped.

## DB-CUSTOM-TYPE-051
Compiler extensions no dependerán de ORM state.

## DB-CUSTOM-TYPE-052
Planner extensions estarán limitadas al dominio del tipo.

## DB-CUSTOM-TYPE-053
Optimizer extensions no obtendrán acceso irrestricto por default.

## DB-CUSTOM-TYPE-054
Parameter Binding seguirá separado.

## DB-CUSTOM-TYPE-055
Custom Type producirá BindingDescriptor, no bindValue directo.

## DB-CUSTOM-TYPE-056
Driver no deberá conocer domain object types.

## DB-CUSTOM-TYPE-057
Hydration reutilizará conversion pipelines.

## DB-CUSTOM-TYPE-058
Custom Hydration Adapter no manipulará IdentityMap.

## DB-CUSTOM-TYPE-059
Custom Types no crearán entities.

## DB-CUSTOM-TYPE-060
Casting será opcional y separado.

## DB-CUSTOM-TYPE-061
Custom Type dependencies serán explícitas.

## DB-CUSTOM-TYPE-062
Missing dependencies se detectarán antes del hot path cuando sea posible.

## DB-CUSTOM-TYPE-063
Unsupported dependency cycles serán rechazados.

## DB-CUSTOM-TYPE-064
Type version será distinta de package version.

## DB-CUSTOM-TYPE-065
Semantic type changes podrán ser breaking.

## DB-CUSTOM-TYPE-066
Type changes afectarán registry generation.

## DB-CUSTOM-TYPE-067
Dependent caches deberán invalidarse mediante fingerprints/generation.

## DB-CUSTOM-TYPE-068
Fingerprints serán deterministas.

## DB-CUSTOM-TYPE-069
Fingerprints no incluirán runtime noise.

## DB-CUSTOM-TYPE-070
Plugin isolation impedirá mutaciones ajenas post-freeze.

## DB-CUSTOM-TYPE-071
DB values no seleccionarán Type implementations.

## DB-CUSTOM-TYPE-072
HTTP values no seleccionarán Type implementation classes.

## DB-CUSTOM-TYPE-073
Registry membership no implicará external permission.

## DB-CUSTOM-TYPE-074
unsafe unserialize no será permitido por default.

## DB-CUSTOM-TYPE-075
Custom Types no utilizarán eval.

## DB-CUSTOM-TYPE-076
Resource governance aplicará a custom values.

## DB-CUSTOM-TYPE-077
Resource requirements podrán especializarse por tipo.

## DB-CUSTOM-TYPE-078
Definitions podrán compartirse en persistent runtimes.

## DB-CUSTOM-TYPE-079
Shared behaviors deberán ser stateless/concurrency-safe.

## DB-CUSTOM-TYPE-080
No habrá mutable static lastValue state.

## DB-CUSTOM-TYPE-081
Development reload producirá nueva registry generation.

## DB-CUSTOM-TYPE-082
Expensive compilation ocurrirá durante bootstrap.

## DB-CUSTOM-TYPE-083
Diagnostics identificarán origin/package.

## DB-CUSTOM-TYPE-084
Diagnostics explicarán platform support.

## DB-CUSTOM-TYPE-085
Telemetry no incluirá raw custom values.

## DB-CUSTOM-TYPE-086
Slow custom conversion podrá ser perfilada.

## DB-CUSTOM-TYPE-087
Conformance tests serán obligatorios para extensiones oficiales.

## DB-CUSTOM-TYPE-088
Round-trip testing será obligatorio para mappings lossless.

## DB-CUSTOM-TYPE-089
Unsupported platforms tendrán tests.

## DB-CUSTOM-TYPE-090
Binding integration tendrá tests.

## DB-CUSTOM-TYPE-091
Query operators tendrán typing tests.

## DB-CUSTOM-TYPE-092
Schema extensions tendrán conformance tests.

## DB-CUSTOM-TYPE-093
Persistent runtime isolation tendrá tests.

## DB-CUSTOM-TYPE-094
Security boundaries tendrán tests.

## DB-CUSTOM-TYPE-095
Fuzzing será recomendado para custom parsers.

## DB-CUSTOM-TYPE-096
Custom Type comparator podrá definir semantic equality.

## DB-CUSTOM-TYPE-097
Dirty tracking utilizará canonical/comparator semantics.

## DB-CUSTOM-TYPE-098
Raw physical equality no será obligatoria.

## DB-CUSTOM-TYPE-099
Type compatibility podrá ser extendida mediante reglas registradas.

## DB-CUSTOM-TYPE-100
Base type compatibility no implicará reverse implicit coercion.

## DB-CUSTOM-TYPE-101
Domain identifiers podrán ser incompatibles aunque compartan UUID base.

## DB-CUSTOM-TYPE-102
Explicit coercion será preferida entre semantic domain types.

## DB-CUSTOM-TYPE-103
Custom Type capabilities podrán variar por operation y platform.

## DB-CUSTOM-TYPE-104
Storage support no implicará query support.

## DB-CUSTOM-TYPE-105
Query support no implicará indexing support.

## DB-CUSTOM-TYPE-106
Index capability será platform-specific cuando corresponda.

## DB-CUSTOM-TYPE-107
Emulation deberá declarar cost and semantics.

## DB-CUSTOM-TYPE-108
Custom Types no realizarán hidden full-table application-side emulation.

## DB-CUSTOM-TYPE-109
Deployment diagnostics podrán verificar dependencies.

## DB-CUSTOM-TYPE-110
Database extension availability será distinta de Type registration.

## DB-CUSTOM-TYPE-111
Platform representability será distinta de observed schema compatibility.

## DB-CUSTOM-TYPE-112
Compiled extension registry no será segunda canonical type registry.

## DB-CUSTOM-TYPE-113
Canonical TypeRegistry seguirá siendo source of truth.

## DB-CUSTOM-TYPE-114
Application API no deberá conocer vendor-specific physical operators por default.

## DB-CUSTOM-TYPE-115
Custom Type Extension System no será Plugin Manager.

## DB-CUSTOM-TYPE-116
Plugin discovery pertenecerá al package/extension system.

## DB-CUSTOM-TYPE-117
Custom Type Extension System no será Service Container.

## DB-CUSTOM-TYPE-118
Custom Type Extension System no será Migration Executor.

## DB-CUSTOM-TYPE-119
Custom Type Extension System no será Query Executor.

## DB-CUSTOM-TYPE-120
Custom Type Extension System no será Authorization System.

## DB-CUSTOM-TYPE-121
Custom Type Extension System no será Runtime global state store.

## DB-CUSTOM-TYPE-122
Invalid extension no será publicada parcialmente.

## DB-CUSTOM-TYPE-123
Bootstrap fail será preferido a runtime ambiguity.

## DB-CUSTOM-TYPE-124
Extension capabilities serán explainable.

## DB-CUSTOM-TYPE-125
Type behavior será reproducible con mismos inputs/context.

## DB-CUSTOM-TYPE-126
Platform fallback no modificará silenciosamente logical TypeId.

## DB-CUSTOM-TYPE-127
Custom schema physical representation no definirá logical identity.

## DB-CUSTOM-TYPE-128
Type migration uncertainty permanecerá UNKNOWN cuando falte evidencia.

## DB-CUSTOM-TYPE-129
Custom Type evolution será tratada como compatibility-sensitive.

## DB-CUSTOM-TYPE-130
Deprecated custom mappings podrán mantener compatibility windows explícitas.

## DB-CUSTOM-TYPE-131
Deprecated physical aliases no serán canonical writes por default.

## DB-CUSTOM-TYPE-132
Custom Type metadata será immutable.

## DB-CUSTOM-TYPE-133
Compiled custom plans serán cacheables.

## DB-CUSTOM-TYPE-134
Type extension registration order incidental no cambiará semántica.

## DB-CUSTOM-TYPE-135
Custom types tendrán bounded TypeId namespace.

## DB-CUSTOM-TYPE-136
Reserved namespaces serán protegidos.

## DB-CUSTOM-TYPE-137
Type arguments estarán bounded por resource policy cuando corresponda.

## DB-CUSTOM-TYPE-138
Custom query AST permanecerá independiente del SQL dialect.

## DB-CUSTOM-TYPE-139
Platform compiler extensions permanecerán independientes del application model.

## DB-CUSTOM-TYPE-140
VoltStack permitirá extensibilidad sin romper las fronteras arquitectónicas del Database System.

---

# 229. Anti-patterns

## 229.1 Custom Type con PDO

```php
final class MoneyType
{
    public function save($value): void
    {
        $this->pdo->prepare(...);
    }
}
```

**Rechazado.**

---

## 229.2 Vendor switch dentro del logical type

```php
if ($driver === 'pgsql') {
    // ...
}
```

**Rechazado en el behavior lógico general.**

Usar Platform Mapping.

---

## 229.3 SQL en converter

```php
return "CAST('$value' AS VECTOR)";
```

**Rechazado.**

---

## 229.4 Reflection por value

```php
$class = $row['type'];
return new $class($value);
```

**Rechazado.**

---

## 229.5 Override silencioso

```text
plugin registers "uuid"
→ replaces core uuid
```

**Rechazado.**

---

## 229.6 Fallback string universal

```text
unsupported custom type
→ stringify everything
```

**Rechazado.**

---

## 229.7 Custom Type para todo Value Object

Crear un logical type por cada pequeña clase de dominio sin necesidad de semántica DB reutilizable.

**No recomendado.**

---

## 229.8 Cast como Custom Type innecesario

Si solo se necesita:

```text
string → EmailAddress
```

y no hay semántica adicional en Query/Schema, un Cast/Value Object mapping puede ser suficiente.

---

## 229.9 Runtime extension mutation

```php
Database::types()
    ->add(...)
```

durante una petición.

**Rechazado por default.**

---

## 229.10 Hidden application-side emulation

Traer todos los rows a PHP para simular un operador custom sin planner/resource policy.

**Rechazado.**

---

# 230. Ejemplo — Domain UserId

```php
final readonly class UserId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Custom Type:

```text
domain.user_id
```

Base:

```text
uuid
```

Representación PHP:

```text
UserId
```

---

# 231. UserId semantics

```text
UserId
→ UUID canonical semantics
```

pero:

```text
domain.user_id
≠
domain.order_id
```

aunque compartan representación física.

---

# 232. Query safety example

Esto podría aceptarse:

```php
User::query()
    ->where('id', $userId);
```

Esto podría rechazarse:

```php
User::query()
    ->where('id', $orderId);
```

si el Query Type System conserva domain type identity.

---

# 233. Physical mapping

Ambos pueden terminar como:

```text
PostgreSQL UUID
MySQL BINARY(16)
SQLite TEXT
```

sin perder su identidad lógica en la aplicación.

---

# 234. Ejemplo — Network IP

Logical Type:

```text
network.ip
```

PHP:

```text
IpAddress
```

Canonical value:

```text
normalized IP representation
```

Mappings:

```text
PostgreSQL:
    inet

MySQL:
    binary/string strategy

SQLite:
    text/binary strategy
```

---

# 235. Query semantics

Puede declarar:

```text
equality
network containment
family
```

cuando Platform lo soporte.

---

# 236. Ejemplo — Vector

Logical:

```text
search.vector(dimensions=1536)
```

PHP:

```text
Vector
```

Query operators:

```text
distance
cosineDistance
innerProduct
```

Platform:

```text
PostgreSQL:
    native vector extension

SQLite:
    optional extension/fallback

MySQL/MariaDB:
    capability-dependent
```

---

# 237. Vector query flow

```text
Vector object
    ↓
Type Reference
    ↓
Semantic Vector Operator
    ↓
Query AST
    ↓
Planner
    ↓
Platform Capability
    ↓
Compiler Extension
    ↓
SQL
```

---

# 238. Ejemplo — Email

Logical:

```text
domain.email
```

Base:

```text
string
```

PHP:

```text
EmailAddress
```

Physical:

```text
VARCHAR
```

Query:

```text
equality
```

Canonicalization:

```text
explicit domain-defined normalization
```

No universal lowercase assumption unless declared.

---

# 239. Extension registration example

```php
final class AppDatabaseTypeContributor
    implements CustomTypeContributor
{
    public function contribute(
        TypeExtensionRegistryBuilder $types,
    ): void {
        $types->register(
            UserIdTypeExtension::definition()
        );

        $types->register(
            EmailTypeExtension::definition()
        );
    }
}
```

---

# 240. Compiled result

```text
TypeRegistry

core:
    bool
    int
    decimal
    string
    uuid
    json
    instant
    ...

custom:
    domain.user_id
    domain.email
```

---

# 241. Master registration formula

```text
CompiledCustomType
=
Compile(
    Validate(
        Normalize(
            TypeDefinition
            + Behaviors
            + PlatformMappings
            + Dependencies
            + QueryExtensions
        )
    )
)
```

---

# 242. Master representability formula

```text
Representable(T, P)
=
LogicalDefinitionValid(T)
∧
PlatformMappingExists(T, P)
∧
RequiredCapabilitiesAvailable(T, P)
```

---

# 243. Master query capability formula

Para operación `O`:

```text
Executable(T, O, P)
=
TypeSupports(T, O)
∧
PlatformSupports(P, O)
∧
CompilerSupports(T, O, P)
```

---

# 244. Master compatibility formula

```text
Compatible(A, B)
=
CoreCompatibility(A, B)
∨
RegisteredCustomCompatibility(A, B)
```

sujeto a que las reglas custom:

```text
do not violate core invariants
```

---

# 245. Master extension model

```text
                     CUSTOM TYPE
                         │
        ┌────────────────┼─────────────────┐
        │                │                 │
    Definition       Behavior         Dependencies
        │                │                 │
        └────────────────┼─────────────────┘
                         ▼
                    Type Registry
                         │
        ┌────────────────┼───────────────────┐
        │                │                   │
   Conversion        Platform             Query
        │                │                   │
        ▼                ▼                   ▼
   PHP Values      Physical Type      Semantic Operators
        │                │                   │
        └────────────────┼───────────────────┘
                         ▼
                  Database Pipeline
```

---

# 246. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Stable namespaced TypeIds
```

para extensiones.

Adoptará:

```text
Bootstrap Registration
→ Validation
→ Compilation
→ Freeze
```

en lugar de runtime mutation.

Adoptará:

```text
Logical Type
+
Platform Mapping
```

como capas separadas.

Adoptará:

```text
Semantic Query Extensions
```

en lugar de SQL strings en Custom Types.

Adoptará:

```text
Binding Descriptors
```

en lugar de acceso directo al Driver.

Adoptará:

```text
Conformance Suites
```

para tipos oficiales y extensiones de calidad.

Adoptará:

```text
Capabilities + Portability Reports
```

para evitar falsas garantías cross-platform.

Y mantendrá:

```text
Custom Type
≠
Driver
≠
SQL Type
≠
Cast
≠
Value Object
≠
Entity
```

---

# 247. Cierre del Bloque 14

Con este documento queda definido el bloque:

```text
155_DATABASE_TYPE_SYSTEM.md
156_DATABASE_TYPE_REGISTRY_SYSTEM.md
157_DATABASE_VALUE_CONVERSION_SYSTEM.md
158_DATABASE_CASTING_SYSTEM.md
159_DATABASE_ENUM_MAPPING_SYSTEM.md
160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md
161_DATABASE_JSON_TYPE_SYSTEM.md
162_DATABASE_DATE_TIME_TYPE_SYSTEM.md
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md
```

Arquitectura resultante:

```text
                         TYPE SYSTEM
                              │
                ┌─────────────┼─────────────┐
                │             │             │
          Type Registry   Conversion      Casting
                │             │             │
                ├─────────────┼─────────────┤
                │             │             │
              Enums       Value Objects    JSON
                │             │             │
                └─────────────┼─────────────┘
                              │
                         Date & Time
                              │
                              ▼
                       Custom Types
                              │
                              ▼
                    Platform Resolution
                              │
                              ▼
                       Database Engine
```

El bloque establece una separación estable entre:

```text
Logical Type
PHP Representation
Application Cast
Domain Value
Canonical Persistent Value
Platform Physical Type
Driver Representation
```

---

# 248. Regla maestra final

> **VoltStack podrá crecer desde un conjunto pequeño de tipos core hacia un ecosistema completo de tipos especializados sin introducir knowledge específico de extensiones dentro del núcleo del Database System.**

El núcleo conocerá:

```text
contracts
registries
capabilities
pipelines
```

mientras las extensiones aportarán:

```text
semantics
converters
platform mappings
query operators
schema requirements
```

Esto permitirá incorporar en el futuro tecnologías como:

```text
PostGIS
pgvector
network types
domain identifiers
financial types
scientific types
specialized binary formats
```

sin convertir:

```text
Query Builder
SQL Compiler
ORM
Driver
```

en sistemas acoplados a cada nueva extensión.

La regla será siempre:

```text
Extension adds capability
```

nunca:

```text
Extension breaks architecture
```

---

# 249. Siguiente bloque

A partir del próximo documento comienza:

```text
BLOCK 15 — TRANSACTIONS & CONCURRENCY
```

con:

```text
164_DATABASE_TRANSACTION_ARCHITECTURE.md
165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md
167_DATABASE_TRANSACTION_ISOLATION_SYSTEM.md
168_DATABASE_NESTED_TRANSACTION_SYSTEM.md
169_DATABASE_SAVEPOINT_SYSTEM.md
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md
171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md
175_DATABASE_TRANSACTION_EVENT_SYSTEM.md
```

---

# 250. Siguiente documento

```text
164_DATABASE_TRANSACTION_ARCHITECTURE.md
```

Regla central propuesta:

> **Una transacción en VoltStack será una unidad explícita de atomicidad y consistencia asociada a un contexto de conexión concreto; no será equivalente a `flush()`, UnitOfWork, EntityManager ni request HTTP, y ningún componente podrá asumir que una operación ORM está comprometida hasta que el Transaction Manager confirme inequívocamente el resultado del commit.**

El siguiente documento deberá cubrir:

- transaction architecture;
- transaction boundaries;
- transaction state machine;
- begin;
- commit;
- rollback;
- UNKNOWN outcomes;
- connection affinity;
- UnitOfWork integration;
- `flush()` vs `commit()`;
- EntityManager lifecycle;
- nested transaction semantics;
- savepoints;
- isolation;
- retries;
- deadlocks;
- connection failures;
- transaction tainting;
- rollback-only state;
- callbacks;
- transaction-scoped resources;
- read/write routing;
- distributed transaction boundaries;
- persistent runtime isolation;
- telemetry;
- diagnostics;
- testing;
- architectural invariants.