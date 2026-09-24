# 158_DATABASE_CASTING_SYSTEM.md

# VoltStack Quantum Database
## Database Casting System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 158 — Database Casting System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `157_DATABASE_VALUE_CONVERSION_SYSTEM.md`

---

# 1. Propósito

`Database Casting System` define la infraestructura responsable de transformar un valor persistente ya tipado hacia una representación conveniente para la aplicación, y de convertir esa representación nuevamente hacia el valor canónico esperado por el sistema ORM cuando el cast sea bidireccional.

Ejemplos:

```text
Database canonical value
        ↓
Casting
        ↓
Application representation
```

```text
"active"
    ↓
Enum Cast
    ↓
Status::ACTIVE
```

```text
"1250.5000"
    ↓
Money Cast
    ↓
Money(1250.5000, "MXN")
```

```text
JSON canonical value
    ↓
Collection Cast
    ↓
Collection
```

La regla central será:

> **Casting en VoltStack será una transformación declarativa de representación aplicada sobre valores cuyo tipo persistente ya es conocido; nunca sustituirá al Type System, Value Conversion, Hydration, Serialization ni decidirá por sí mismo cómo se almacena físicamente un valor en la base de datos.**

---

# 2. Posición arquitectónica

El flujo completo será:

```text
Database
   │
   ▼
Driver Raw Value
   │
   ▼
Value Conversion
   │
   ▼
Canonical Persistent Value
   │
   ▼
Casting
   │
   ▼
Application Value
   │
   ▼
Entity / Model
```

Para escritura:

```text
Entity / Model
   │
   ▼
Application Value
   │
   ▼
Inbound Casting
   │
   ▼
Canonical Persistent Value
   │
   ▼
Value Conversion
   │
   ▼
Parameter Binding
   │
   ▼
Database
```

Por tanto:

```text
Database
↕ Value Conversion
Canonical Persistent Value
↕ Casting
Application Value
```

---

# 3. Casting ≠ Value Conversion

Esta separación será fundamental.

`Value Conversion` responde:

> ¿Cómo se representa este tipo lógico entre PHP y la plataforma/driver?

`Casting` responde:

> ¿Qué representación desea utilizar la aplicación para trabajar con ese valor?

Ejemplo:

```text
DB DECIMAL
   ↓ Value Conversion
"1250.5000"
   ↓ Casting
Money
```

---

# 4. Casting ≠ Type System

El Type System define:

```text
TypeId
TypeReference
TypeDefinition
TypeBehavior
```

Casting consume esa información.

No la reemplaza.

---

# 5. Casting ≠ Hydration

Hydration coordina:

```text
row
→ entity
```

Casting transforma:

```text
canonical field value
→ application field value
```

El Hydrator puede invocar un `CompiledCastPipeline`, pero:

```text
Hydrator
≠
Caster
```

---

# 6. Casting ≠ Serialization

Serialization transforma objetos para:

- JSON;
- HTTP;
- API;
- queues;
- logs;
- external formats.

Casting transforma representación dentro del modelo de aplicación.

Por tanto:

```text
Casting
≠
Serialization
```

---

# 7. Casting ≠ Validation

Un cast puede rechazar una representación imposible.

Eso no significa que implemente reglas de negocio.

Ejemplo:

```text
Money("100.00")
```

puede ser técnicamente válido aunque una regla de negocio requiera:

```text
price >= 500
```

---

# 8. Casting ≠ Encryption

Un encrypted cast podrá integrarse con el sistema de Encryption, pero:

```text
Casting System
≠
Cryptographic Engine
```

---

# 9. Casting ≠ Accessors/Mutators

VoltStack deberá distinguir:

```text
Persistent Cast
```

de:

```text
Application Accessor
```

Un accessor puede calcular:

```php
public function fullName(): string
```

sin formar parte del mapping persistente.

Un cast está asociado a la representación persistente de uno o más campos.

---

# 10. Modelo conceptual

```text
Persistent Representation
        │
        ▼
     CastDefinition
        │
        ▼
  CompiledCastPipeline
        │
        ▼
     CastExecutor
        │
        ▼
Application Representation
```

---

# 11. CastDefinition

```php
final readonly class CastDefinition
{
    public function __construct(
        public CastId $id,
        public CastDirection $direction,
        public CastTypeReference $source,
        public CastTypeReference $target,
        public CastBehaviorReference $behavior,
        public CastArguments $arguments,
    ) {}
}
```

---

# 12. CastId

Cada cast tendrá identidad estable:

```php
final readonly class CastId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
bool
int
float
string
array
collection
datetime
enum
money
encrypted
custom:geo-point
```

---

# 13. CastDirection

```php
enum CastDirection
{
    case INBOUND;
    case OUTBOUND;
    case BIDIRECTIONAL;
}
```

---

# 14. Outbound cast

Transforma:

```text
Persistent Value
→
Application Value
```

Ejemplo:

```text
string
→
Status enum
```

---

# 15. Inbound cast

Transforma:

```text
Application Value
→
Persistent Value
```

Ejemplo:

```text
Money
→
decimal string
```

---

# 16. Bidirectional cast

Implementa ambas transformaciones.

Formalmente:

```text
decode : P → A
encode : A → P
```

donde:

```text
P = Persistent representation
A = Application representation
```

---

# 17. Round-trip

Para casts reversibles:

```text
decode(encode(x)) ≡ x
```

dentro de la equivalencia semántica declarada.

---

# 18. CastBehavior

```php
interface CastBehavior
{
    public function inbound(
        mixed $value,
        CastContext $context,
    ): mixed;

    public function outbound(
        mixed $value,
        CastContext $context,
    ): mixed;
}
```

---

# 19. Specialized contracts

Podrán existir:

```php
interface InboundCast
{
    public function inbound(
        mixed $value,
        CastContext $context,
    ): mixed;
}

interface OutboundCast
{
    public function outbound(
        mixed $value,
        CastContext $context,
    ): mixed;
}
```

---

# 20. CastContext

```php
final readonly class CastContext
{
    public function __construct(
        public CastDirection $direction,
        public TypeReference $persistentType,
        public CastDefinition $definition,
        public CastPolicy $policy,
    ) {}
}
```

---

# 21. Contexto mínimo

El cast core no deberá necesitar automáticamente:

- EntityManager;
- UnitOfWork;
- Repository;
- Connection;
- HTTP Request;
- authenticated user;
- current tenant global.

---

# 22. Field context

Cuando sea necesario para diagnostics:

```php
final readonly class CastFieldContext
{
    public function __construct(
        public EntityType $entity,
        public PropertyPath $property,
    ) {}
}
```

---

# 23. Entity instance

Por defecto, el cast no recibirá la entidad completa.

Esto evita:

```text
hidden coupling
+
recursive access
+
lazy loading
+
non-determinism
```

---

# 24. Entity-aware casts

Si en el futuro se permiten, deberán ser una extensión explícita y no el mecanismo base.

---

# 25. Cast registry

El sistema utilizará:

```text
CastRegistry
```

para resolver definiciones y behaviors.

---

# 26. CastRegistry contract

```php
interface CastRegistry
{
    public function definition(CastId $id): CastDefinition;

    public function behavior(
        CastBehaviorReference $reference,
    ): CastBehavior;

    public function has(CastId $id): bool;
}
```

---

# 27. Registry lifecycle

```text
BOOTSTRAP
   ↓
REGISTERING
   ↓
VALIDATING
   ↓
COMPILED
   ↓
FROZEN
```

---

# 28. Frozen registry

Después de bootstrap:

```text
CastRegistry
```

deberá ser immutable/frozen para producción.

---

# 29. No hot-path discovery

Nunca:

```text
property access
→ reflection
→ discover cast
→ instantiate caster
```

por cada lectura.

---

# 30. Metadata compilation

Los casts declarados en entidades/modelos se compilarán dentro de `EntityMetadata`.

Ejemplo:

```php
#[Cast('datetime')]
private DateTimeImmutable $createdAt;
```

se transforma durante bootstrap en metadata normalizada.

---

# 31. Model API

La API Laravel-like podrá permitir:

```php
protected function casts(): array
{
    return [
        'active' => 'bool',
        'settings' => 'array',
        'status' => Status::class,
        'created_at' => 'datetime',
    ];
}
```

---

# 32. Single canonical metadata model

Attributes:

```php
#[Cast('bool')]
```

y Model API:

```php
'active' => 'bool'
```

deberán converger en:

```text
CastDefinition
```

canónico.

---

# 33. No dual casting engines

VoltStack no tendrá:

```text
ModelCastEngine
```

y otro:

```text
EntityCastEngine
```

independientes.

Ambos usarán:

```text
Database Casting System
```

---

# 34. Cast arguments

Ejemplo:

```text
decimal:2
datetime:UTC
collection:UserPreference
```

deberán normalizarse.

---

# 35. Structured arguments

Internamente evitar:

```text
explode(':', $cast)
```

en hot path.

Usar:

```php
final readonly class CastArguments
{
    public function __construct(
        public array $values,
    ) {}
}
```

---

# 36. Cast syntax parser

La sintaxis compacta pertenece al developer-facing mapping parser.

Después:

```text
"decimal:2"
    ↓
CastDefinition
```

---

# 37. Cast pipeline

Un campo puede requerir varias transformaciones.

Ejemplo:

```text
Persistent JSON
    ↓
Array Cast
    ↓
Collection Cast
```

---

# 38. CompiledCastPipeline

```php
final readonly class CompiledCastPipeline
{
    /**
     * @param list<CompiledCastStep> $steps
     */
    public function __construct(
        public array $steps,
        public CastPipelineFingerprint $fingerprint,
    ) {}
}
```

---

# 39. Pipeline directions

Outbound:

```text
Persistent
→ Cast A
→ Cast B
→ Application
```

Inbound invierte únicamente si cada etapa define una inversa válida.

---

# 40. Composition validation

No todo pipeline outbound es automáticamente inbound.

Ejemplo:

```text
timestamp
→ formatted string
```

puede perder información.

---

# 41. Reversibility

Cada cast podrá declarar:

```php
enum CastReversibility
{
    case REVERSIBLE;
    case NORMALIZING;
    case LOSSY;
    case ONE_WAY;
}
```

---

# 42. Lossy cast

Un cast `LOSSY` no podrá utilizarse automáticamente para persistencia bidireccional.

---

# 43. Example lossy cast

```text
DateTime
→ "September 2026"
```

no contiene suficiente información para reconstruir el instante.

---

# 44. Pipeline compiler

```php
interface CastPipelineCompiler
{
    public function compile(
        FieldCastMetadata $metadata,
    ): CompiledCastPipeline;
}
```

---

# 45. Pipeline executor

```php
interface CastPipelineExecutor
{
    public function inbound(
        CompiledCastPipeline $pipeline,
        mixed $value,
        CastContext $context,
    ): mixed;

    public function outbound(
        CompiledCastPipeline $pipeline,
        mixed $value,
        CastContext $context,
    ): mixed;
}
```

---

# 46. Compiler ≠ Executor

Mantener:

```text
Cast Pipeline Compiler
≠
Cast Pipeline Executor
```

---

# 47. Scalar casts

VoltStack podrá incluir:

```text
bool
integer
float
string
```

como convenience casts.

---

# 48. Scalar cast warning

Estos casts no deberán duplicar las responsabilidades del Type System.

Ejemplo:

```text
DB logical integer
→ application string
```

sí es casting.

Pero:

```text
driver "123"
→ logical integer
```

es Value Conversion.

---

# 49. Boolean cast

Ejemplo:

```text
canonical 1/0
```

no debería llegar normalmente al casting layer si el persistent TypeReference ya es boolean.

El cast `bool` principalmente normaliza la representación application-facing cuando el persistent type sea compatible.

---

# 50. Integer cast

No debe ocultar overflow.

---

# 51. Float cast

No debe utilizarse automáticamente para valores `DECIMAL`.

---

# 52. String cast

No deberá invocar `__toString()` arbitrariamente sobre objetos desconocidos salvo contrato explícito.

---

# 53. Array cast

Un tipo JSON puede proyectarse a:

```text
array
```

mediante:

```text
JSON Value Conversion
→ canonical JSON PHP value
→ Array Cast
```

---

# 54. Collection cast

Podrá transformar:

```text
array
→ Collection
```

---

# 55. Collection cast ≠ ORM relationship collection

Muy importante:

```text
JSON Collection Cast
≠
PersistentCollection
```

Una colección JSON es un valor.

Una `PersistentCollection` representa una relación ORM.

---

# 56. No relationship loading

Acceder a un cast de colección no deberá ejecutar queries.

---

# 57. JSON object cast

Podrá proyectar a:

```text
stdClass
```

o a un DTO/Value Object explícito.

---

# 58. Enum casts

Integración completa con:

```text
159_DATABASE_ENUM_MAPPING_SYSTEM.md
```

---

# 59. Enum outbound

```text
canonical backing value
→ enum case
```

---

# 60. Enum inbound

```text
enum case
→ canonical backing value
```

---

# 61. Unknown enum

Nunca:

```text
unknown value
→ null
```

silenciosamente.

---

# 62. Date/time casts

Integración con:

```text
162_DATABASE_DATE_TIME_TYPE_SYSTEM.md
```

---

# 63. Temporal layering

```text
DB
 ↓
Temporal Value Conversion
 ↓
Canonical Temporal Value
 ↓
Temporal Cast
 ↓
Application Representation
```

---

# 64. DateTimeImmutable default

VoltStack deberá favorecer:

```php
DateTimeImmutable
```

sobre objetos temporales mutables.

---

# 65. Mutable DateTime

Podrá soportarse explícitamente, pero tiene implicaciones para dirty tracking.

---

# 66. Date formatting ≠ temporal cast persistence

Formatear:

```text
2026-09-07
→
07/09/2026
```

es generalmente presentación/serialization, no mapping persistente.

---

# 67. Value Object casts

Ejemplo:

```text
canonical string
→ EmailAddress
```

---

# 68. Single-value object

```php
final readonly class EmailAddress
{
    public function __construct(
        public string $value,
    ) {}
}
```

puede ser manejado por un bidirectional cast.

---

# 69. Money

Si `Money` depende de:

```text
amount
currency
```

entonces normalmente deberá usar:

```text
Value Object Mapping System
```

del documento 160, no un scalar cast artificial.

---

# 70. Multi-column casts

V1 no deberá permitir que un cast escalar escriba arbitrariamente múltiples columnas.

---

# 71. Why

Eso mezclaría:

```text
Casting
+
Mapping
+
Persistence Planning
```

---

# 72. Composite value objects

Se resolverán mediante metadata de Value Object Mapping.

---

# 73. Custom casts

Aplicaciones podrán registrar:

```php
final class EmailAddressCast implements CastBehavior
{
    public function inbound(
        mixed $value,
        CastContext $context,
    ): mixed {
        // ...
    }

    public function outbound(
        mixed $value,
        CastContext $context,
    ): mixed {
        // ...
    }
}
```

---

# 74. Registration

Ejemplo conceptual:

```php
$casts->register(
    id: 'email',
    behavior: EmailAddressCast::class,
);
```

durante bootstrap.

---

# 75. No arbitrary class from mapping input

Una API externa no podrá enviar:

```text
cast = App\Whatever\UserProvidedClass
```

y provocar instanciación automática.

---

# 76. Service references

Metadata compilada deberá utilizar:

```text
CastBehaviorReference
```

en lugar de objetos arbitrarios.

---

# 77. Container integration

Los custom casts podrán ser servicios del container.

Pero su resolución debe ocurrir de manera controlada.

---

# 78. Stateless preference

Los casts compartibles deberán ser stateless.

---

# 79. Stateful cast

Si un cast necesita estado mutable por operación, deberá declararlo y resolverse scoped.

---

# 80. Cast purity

Ideal:

```text
output = f(input, definition, explicit context)
```

---

# 81. No hidden I/O

Un cast no deberá:

- ejecutar queries;
- abrir archivos;
- hacer HTTP;
- iniciar lazy loading;
- consultar repositories.

por defecto.

---

# 82. I/O casts

Si alguna extensión requiere I/O, deberá pertenecer a otra abstracción explícita y no al casting hot path estándar.

---

# 83. Null handling

NULL deberá ser tratado explícitamente.

---

# 84. Nullable field

Normalmente:

```text
null
→ null
```

sin invocar el cast.

---

# 85. Null-aware cast

Algunos casts podrían necesitar distinguir null.

Esto deberá declararse:

```php
interface NullAwareCast
{
}
```

---

# 86. Default

Default recomendado:

```text
null bypasses cast
```

cuando el field es nullable.

---

# 87. Non-nullable field

`null` deberá fallar antes de producir un application value inválido.

---

# 88. Missing ≠ NULL

Durante hydration parcial:

```text
field not selected
```

no es:

```text
field selected with NULL
```

---

# 89. No cast for missing field

Un campo `MISSING` no deberá convertirse a:

```text
null
```

y luego castearse.

---

# 90. LoadedFieldMask

Casting respetará:

```text
LoadedFieldMask
```

establecido por Hydration System.

---

# 91. Partial entities

Acceder a un campo no cargado deberá seguir la política de partial entity.

Casting no deberá ocultar esa ausencia.

---

# 92. Dirty tracking

Este es uno de los puntos críticos.

Debe decidirse qué representación utiliza el UoW para comparar cambios.

---

# 93. Recommended model

Mantener:

```text
Application Value
```

en la entidad, pero producir un:

```text
Canonical Persistent Value
```

para snapshots/comparación cuando el cast pueda alterar representación.

---

# 94. Canonical comparison

Ejemplo:

```text
EmailAddress("USER@example.com")
```

si el cast normaliza:

```text
"user@example.com"
```

la comparación persistente deberá usar la representación canónica definida.

---

# 95. Dirty equality

No deberá depender simplemente de:

```php
$old !== $new
```

para objetos casteados.

---

# 96. CastComparator

```php
interface CastComparator
{
    public function equivalent(
        mixed $left,
        mixed $right,
        CastComparisonContext $context,
    ): bool;
}
```

---

# 97. Default comparison

Para casts reversibles:

```text
canonicalize(left)
==
canonicalize(right)
```

---

# 98. Value Object equality

Puede usar semántica declarada por el Value Object mapping.

---

# 99. Mutable cast values

Ejemplo:

```php
ArrayObject
```

puede modificarse internamente sin reemplazar la propiedad.

---

# 100. Mutation tracking problem

```php
$model->settings['theme'] = 'dark';
```

puede no activar un setter.

---

# 101. Snapshot strategy

Para valores mutables, Change Tracking deberá conservar un snapshot canónico independiente.

---

# 102. No shared mutable snapshot

Nunca guardar:

```text
same mutable object reference
```

como snapshot y current value.

---

# 103. Immutable preference

VoltStack favorecerá casts hacia:

```text
immutable objects
```

por rendimiento, seguridad y dirty tracking.

---

# 104. Cast mutability metadata

```php
enum CastMutability
{
    case IMMUTABLE;
    case MUTABLE;
}
```

---

# 105. Mutable cast policy

Los casts mutables deberán declarar cómo:

- snapshot;
- compare;
- clone;
- canonicalize.

---

# 106. CastSnapshotStrategy

```php
interface CastSnapshotStrategy
{
    public function snapshot(
        mixed $value,
        CastContext $context,
    ): mixed;
}
```

---

# 107. Snapshot ≠ serialization

No usar JSON serialize indiscriminadamente para detectar cambios.

---

# 108. Why

Podría:

- perder tipos;
- alterar ordering;
- ser costoso;
- ocultar precisión;
- cambiar semántica.

---

# 109. Inbound normalization

Antes de persistence:

```text
Application Value
    ↓
Inbound Cast
    ↓
Canonical Persistent Value
```

---

# 110. ChangeSet

El ChangeSet deberá contener semántica persistente.

Conceptualmente:

```text
FieldChange
├── old canonical persistent value
└── new canonical persistent value
```

---

# 111. No DB representation in ChangeSet

No almacenar:

```text
driver-ready value
```

en el ChangeSet.

Eso pertenece posteriormente a Value Conversion.

---

# 112. Flush pipeline

```text
Entity Application Value
        ↓
Cast inbound
        ↓
Canonical Persistent Value
        ↓
Change Tracking
        ↓
Persistence Planner
        ↓
Query Model
        ↓
Value Conversion
        ↓
Parameter Binding
        ↓
Database
```

---

# 113. Hydration pipeline

```text
Driver Raw
   ↓
Value Conversion
   ↓
Canonical Persistent Value
   ↓
Cast outbound
   ↓
Application Value
   ↓
Entity assignment
```

---

# 114. Baseline snapshot timing

La baseline persistente deberá establecerse a partir del valor canónico después de Value Conversion y antes de que mutaciones posteriores de aplicación alteren el estado.

---

# 115. PostLoad events

Regla de ORM existente:

```text
baseline established
→ postLoad
```

Si `postLoad` modifica un valor casteado, deberá poder aparecer como dirty.

---

# 116. Cast caching

Hay dos conceptos diferentes:

```text
Cast Pipeline Cache
```

y:

```text
Cast Result Cache
```

---

# 117. Pipeline cache

Recomendado.

Almacena:

```text
compiled behavior
```

---

# 118. Result cache

Mucho más delicado.

---

# 119. Immutable cast result caching

Para un mismo field de una entidad, puede ser útil conservar el objeto casteado mientras el valor persistente no cambie.

---

# 120. Example

Evitar:

```php
$model->address === $model->address
```

produciendo dos Value Objects diferentes en cada acceso cuando se espera estabilidad de propiedad.

---

# 121. Entity-local cast result cache

Si se utiliza, deberá ser:

```text
entity/scoped
```

no global.

---

# 122. Global result cache forbidden

Nunca:

```text
raw email string
→ global EmailAddress object
```

entre requests.

---

# 123. Identity semantics

Un Value Object casteado no pertenece al ORM IdentityMap.

---

# 124. Important distinction

```text
Entity Identity
≠
Cast Value Identity
```

---

# 125. IdentityMap

Nunca registrar:

```text
Money
EmailAddress
DateRange
```

como entidades solo porque fueron creados por un cast.

---

# 126. Cast result invalidation

Si cambia el canonical persistent value:

```text
cached application cast value
```

deberá invalidarse.

---

# 127. Setter path

```text
$model->status = Status::ACTIVE
```

puede conservar application value y producir inbound cast durante change tracking/flush.

---

# 128. Immediate inbound casting

Otra estrategia sería convertir al asignar.

VoltStack no deberá imponerla universalmente.

---

# 129. Recommended V1

Preferir:

```text
application representation stored in entity
+
canonicalization at ORM boundaries
```

para mantener entidades naturales.

---

# 130. Active Record API

Ejemplo:

```php
final class User extends Model
{
    protected function casts(): array
    {
        return [
            'active' => 'bool',
            'settings' => 'array',
            'status' => Status::class,
            'created_at' => 'datetime',
        ];
    }
}
```

---

# 131. Data Mapper API

```php
final class User
{
    #[Column(type: 'string')]
    #[Cast(StatusCast::class)]
    private Status $status;
}
```

Ambas APIs convergen en la misma metadata.

---

# 132. Native typed properties

VoltStack deberá validar compatibilidad entre:

```text
property PHP type
```

y:

```text
cast output type
```

durante metadata compilation.

---

# 133. Example mismatch

```php
#[Cast(StatusCast::class)]
private int $status;
```

si el cast produce:

```text
Status
```

deberá fallar en bootstrap.

---

# 134. Union types

Podrán soportarse si la compatibilidad es demostrable.

Ejemplo:

```php
private Status|null $status;
```

---

# 135. mixed

Permitido técnicamente, pero reduce validación estática.

Diagnostics puede advertirlo.

---

# 136. Cast compatibility analysis

```text
Persistent Type
      ↓
Cast Input
      ↓
Cast Output
      ↓
Property PHP Type
```

todo deberá ser compatible.

---

# 137. CastTypeReference

```php
final readonly class CastTypeReference
{
    public function __construct(
        public string $phpType,
        public bool $nullable,
    ) {}
}
```

Podrá evolucionar hacia un modelo más rico.

---

# 138. Generic collection casts

Ejemplo:

```text
collection<UserPreference>
```

requiere metadata explícita del elemento.

---

# 139. Nested casts

Puede existir:

```text
JSON
→ array
→ typed collection
→ Value Objects
```

---

# 140. Depth control

Pipelines anidados deberán tener límites de profundidad.

---

# 141. Cycles

Ejemplo inválido:

```text
Cast A → Cast B → Cast A
```

deberá detectarse al compilar.

---

# 142. CastPipelineFingerprint

Puede derivarse de:

```text
FieldMetadata
+
CastDefinitions
+
CastArguments
+
CastRegistryGeneration
+
TypeRegistryGeneration
+
PolicyGeneration
```

---

# 143. Cache invalidation

Cambiar un custom cast deberá invalidar el pipeline.

---

# 144. Registry generation

```php
final readonly class CastRegistryGeneration
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 145. Determinism

Misma:

```text
input
+
cast definition
+
arguments
+
explicit context
```

deberá producir el mismo resultado para casts declarados deterministas.

---

# 146. Non-deterministic casts

No deberán formar parte del mapping persistente normal.

Ejemplo:

```text
value → random UUID
```

no es un cast apropiado.

---

# 147. Current time cast

```text
value → now()
```

también es no determinista.

Pertenece a default generators/lifecycle logic.

---

# 148. Environment independence

Los casts no deberán depender silenciosamente de:

- locale;
- timezone global;
- current request language;
- current authenticated user.

---

# 149. Locale-specific cast

Si realmente se necesita:

```text
locale
```

deberá ser argumento explícito.

---

# 150. Timezone-specific cast

Igual:

```text
timezone
```

debe declararse.

---

# 151. Serialization interaction

El application value casteado puede luego serializarse.

Ejemplo:

```text
DB
↓
Value Conversion
↓
decimal string
↓
Money Cast
↓
Money object
↓
Serializer
↓
{"amount":"100.00","currency":"MXN"}
```

---

# 152. Serializer independence

El cast no deberá conocer JSON HTTP salvo cast específicamente diseñado como value representation.

---

# 153. API representation

Una API puede querer:

```text
Money → "$100.00 MXN"
```

Eso pertenece a serialization/presentation.

---

# 154. Hidden casts

Casting no controla si un campo aparece en JSON.

Eso pertenece a serialization/model visibility.

---

# 155. Encrypted casts

Integración conceptual:

```text
Application Value
    ↓
Cast
    ↓
Canonical Serializable Value
    ↓
Encryption Integration
    ↓
Encrypted Persistent Value
```

---

# 156. Encryption ordering

El orden deberá ser explícito.

Ejemplo:

```text
Money
→ canonical decimal
→ encrypt
→ binary/string
```

---

# 157. Encryption ≠ ordinary cast

VoltStack podrá exponer una sintaxis cómoda:

```text
encrypted:string
```

pero internamente deberá delegar al sistema criptográfico.

---

# 158. Hashing

Hashing es one-way.

Por tanto:

```text
hashed
```

no puede declararse como bidirectional cast normal.

---

# 159. Password hashing

Debe pertenecer al Hashing/Authentication boundary correspondiente.

No a un reversible DB cast.

---

# 160. Compression integration

Un cast/adapter podría componer:

```text
value
→ serialize
→ compress
```

pero compression debería ser una transformación explícita y diagnosticable.

---

# 161. Ordering

El orden de transformaciones importa:

```text
serialize → encrypt
```

no es igual a:

```text
encrypt → serialize
```

---

# 162. Pipeline ordering validation

La compilación deberá verificar compatibilidad entre output/input de cada paso.

---

# 163. Cast aliases

Podrán existir aliases:

```text
boolean → bool
integer → int
```

---

# 164. Canonical CastId

Aliases se normalizan durante bootstrap.

Hot path solo utiliza `CastId` canónico.

---

# 165. Deprecation

Aliases legacy podrán marcarse deprecated.

---

# 166. Dynamic user includes

Nunca permitir que parámetros HTTP seleccionen arbitrariamente una clase de cast.

---

# 167. Security

El Casting System deberá prevenir:

- arbitrary class instantiation;
- unsafe deserialization;
- hidden I/O;
- recursive casting bombs;
- unbounded JSON/object graphs;
- leaking sensitive values in diagnostics.

---

# 168. Cast resource policy

```php
final readonly class CastResourcePolicy
{
    public function __construct(
        public int $maxPipelineDepth,
        public int $maxCollectionItems,
        public int $maxNestedDepth,
        public int $maxInputBytes,
    ) {}
}
```

---

# 169. Large collections

Un JSON cast de millones de elementos puede ser técnicamente válido pero operacionalmente peligroso.

Debe existir resource governance.

---

# 170. Resource failure

```text
CastResourceLimitException
```

---

# 171. Error hierarchy

```text
DatabaseCastingException
├── CastNotFoundException
├── CastRegistrationException
├── CastMetadataException
├── InvalidCastDefinitionException
├── CastCompatibilityException
├── CastDirectionException
├── CastInputTypeException
├── CastOutputTypeException
├── CastArgumentException
├── CastNullabilityException
├── CastPipelineException
├── CastPipelineCycleException
├── CastReversibilityException
├── CastLossException
├── CastExecutionException
├── CastSnapshotException
├── CastComparisonException
├── CastResourceLimitException
├── CastSecurityException
├── CastRuntimeStateException
└── CastingInvariantViolationException
```

---

# 172. Error diagnostics

Un error deberá poder indicar:

```text
Entity:
    User

Property:
    status

Cast:
    enum:UserStatus

Direction:
    OUTBOUND

Persistent Type:
    string

Expected Output:
    UserStatus

Actual Value Type:
    string

Failure:
    Unknown enum backing value
```

sin revelar valores sensibles innecesariamente.

---

# 173. Sensitive cast values

Diagnostics deberán usar:

```text
type
length
cast id
property
```

antes que raw values.

---

# 174. Telemetry

Métricas potenciales:

```text
database.cast.total
database.cast.failure
database.cast.pipeline_cache_hit
database.cast.pipeline_cache_miss
database.cast.inbound
database.cast.outbound
database.cast.resource_rejected
database.cast.snapshot
database.cast.custom
```

---

# 175. Metric dimensions

Podrán incluir:

```text
CastId
direction
entity type
failure class
```

si cardinalidad está controlada.

---

# 176. No raw values in telemetry

Nunca:

```text
email=user@example.com
```

---

# 177. Tracing

No crear span por cada field cast.

---

# 178. Query profiler

Podrá acumular:

```text
cast count
cast time
custom cast time
```

por hydration/flush.

---

# 179. Slow cast detection

Útil para detectar custom casts costosos.

---

# 180. Persistent runtime

Compartible:

```text
CastRegistry
CompiledCastPipeline
CastDefinitions
Stateless CastBehavior
```

---

# 181. Scoped

```text
CastExecutionContext
temporary recursion stack
resource counters
entity-local cast result cache
```

---

# 182. FrankenPHP

```text
Worker
├── frozen CastRegistry
├── compiled pipelines
├── stateless cast services
│
├── Request A scoped cast state
└── Request B scoped cast state
```

---

# 183. RoadRunner

Misma separación.

---

# 184. OpenSwoole

Mutable context deberá ser coroutine-safe.

---

# 185. No static entity cache

Nunca:

```php
static $castedValues = [];
```

para almacenar valores de entidades entre requests.

---

# 186. Development reload

Cambio en:

```text
CastDefinition
Custom Cast
Cast Arguments
```

produce nueva generation/fingerprint.

---

# 187. Model metadata cache

Compiled EntityMetadata puede contener:

```text
FieldCastMetadata
+
CompiledCastPipelineReference
```

---

# 188. HydrationPlan cache

Puede incorporar referencias a pipelines ya compilados.

---

# 189. No duplicate compilation

Mismo metadata fingerprint debería reutilizar pipeline.

---

# 190. Testing architecture

Cada cast core deberá tener:

```text
unit tests
round-trip tests
null tests
type compatibility tests
boundary tests
dirty tracking tests
snapshot tests
persistent runtime tests
```

---

# 191. Cast conformance suite

```php
abstract class CastConformanceTestCase
{
    abstract protected function cast(): CastBehavior;

    abstract protected function definition(): CastDefinition;
}
```

---

# 192. Bidirectional tests

Para casts reversibles:

```text
outbound(inbound(x)) ≡ x
```

o la dirección semántica correspondiente.

---

# 193. Dirty tracking tests

Caso:

```text
hydrate
→ no modification
→ flush
```

debe producir:

```text
no UPDATE
```

---

# 194. Equivalent representation test

Ejemplo:

```text
EmailAddress("USER@example.com")
```

y:

```text
EmailAddress("user@example.com")
```

si el cast define equivalencia case-insensitive/canonicalizada:

```text
no false dirty
```

---

# 195. Real modification

```text
old canonical != new canonical
```

debe producir ChangeSet.

---

# 196. Mutable cast test

Modificar internamente un valor mutable deberá detectarse según snapshot strategy.

---

# 197. Partial hydration test

Campo no seleccionado:

```text
must remain MISSING
```

y no ser casteado.

---

# 198. NULL test

```text
DB NULL
→ canonical null
→ application null
```

para nullable mapping.

---

# 199. Invalid cast mapping

Debe fallar durante metadata compilation cuando sea detectable.

---

# 200. Property type mismatch

Debe fallar antes de runtime normal.

---

# 201. Pipeline cycle test

Debe detectarse en bootstrap.

---

# 202. Persistent worker test

Request A:

```text
status = ACTIVE
```

Request B:

```text
status = DISABLED
```

no deberán compartir cast result state.

---

# 203. Concurrency test

Stateless custom casts deberán soportar ejecuciones concurrentes.

---

# 204. Performance tests

Medir:

```text
casts/sec
pipeline resolution
pipeline cache hit rate
hydration cast overhead
flush canonicalization overhead
snapshot overhead
custom cast overhead
```

---

# 205. Performance objective

Los casts core simples deberían acercarse al costo de una llamada especializada pre-resuelta, evitando:

- reflection;
- container lookup por field;
- parsing de cast strings;
- repeated metadata resolution.

---

# 206. Directory structure

```text
src/Quantum/Database/Type/Casting/
│
├── Contract/
│   ├── CastBehavior.php
│   ├── InboundCast.php
│   ├── OutboundCast.php
│   ├── CastRegistry.php
│   ├── CastPipelineCompiler.php
│   ├── CastPipelineExecutor.php
│   ├── CastComparator.php
│   └── CastSnapshotStrategy.php
│
├── Definition/
│   ├── CastDefinition.php
│   ├── CastId.php
│   ├── CastBehaviorReference.php
│   ├── CastArguments.php
│   ├── CastDirection.php
│   ├── CastReversibility.php
│   └── CastMutability.php
│
├── Metadata/
│   ├── FieldCastMetadata.php
│   ├── CastMetadataCompiler.php
│   └── CastCompatibilityAnalyzer.php
│
├── Registry/
│   ├── DefaultCastRegistry.php
│   ├── CastRegistryBuilder.php
│   └── CastRegistryGeneration.php
│
├── Pipeline/
│   ├── CompiledCastPipeline.php
│   ├── CompiledCastStep.php
│   ├── CastPipelineFingerprint.php
│   ├── DefaultCastPipelineCompiler.php
│   └── DefaultCastPipelineExecutor.php
│
├── Context/
│   ├── CastContext.php
│   ├── CastFieldContext.php
│   ├── CastComparisonContext.php
│   └── CastExecutionContext.php
│
├── Builtin/
│   ├── BooleanCast.php
│   ├── IntegerCast.php
│   ├── FloatCast.php
│   ├── StringCast.php
│   ├── ArrayCast.php
│   ├── CollectionCast.php
│   ├── DateTimeCast.php
│   ├── EnumCast.php
│   └── ValueObjectCast.php
│
├── Snapshot/
│   ├── ImmutableCastSnapshotStrategy.php
│   ├── MutableCastSnapshotStrategy.php
│   └── CanonicalCastComparator.php
│
├── Policy/
│   ├── CastPolicy.php
│   └── CastResourcePolicy.php
│
├── Cache/
│   ├── CastPipelineCache.php
│   └── EntityCastResultCache.php
│
├── Diagnostics/
│   ├── CastExplainer.php
│   └── CastDiagnosticReport.php
│
└── Exception/
    ├── DatabaseCastingException.php
    ├── CastNotFoundException.php
    ├── CastRegistrationException.php
    ├── CastMetadataException.php
    ├── InvalidCastDefinitionException.php
    ├── CastCompatibilityException.php
    ├── CastDirectionException.php
    ├── CastInputTypeException.php
    ├── CastOutputTypeException.php
    ├── CastArgumentException.php
    ├── CastNullabilityException.php
    ├── CastPipelineException.php
    ├── CastPipelineCycleException.php
    ├── CastReversibilityException.php
    ├── CastLossException.php
    ├── CastExecutionException.php
    ├── CastSnapshotException.php
    ├── CastComparisonException.php
    ├── CastResourceLimitException.php
    ├── CastSecurityException.php
    └── CastingInvariantViolationException.php
```

---

# 207. Dependency model

Permitido:

```text
Casting
   ↓
Type System

Casting
   ↓
Type Registry

Casting
   ↓
Value Conversion contracts

Casting
   ↓
Entity Metadata contracts
```

---

# 208. Integration direction

ORM puede consumir Casting:

```text
ORM
 ↓
Casting
 ↓
Type System
```

Casting no debe consumir ORM lifecycle services.

---

# 209. Prohibido

```text
Casting
 ↓
EntityManager
```

```text
Casting
 ↓
UnitOfWork mutable state
```

```text
Casting
 ↓
Query Executor
```

```text
Casting
 ↓
Connection
```

```text
Casting
 ↓
Driver
```

---

# 210. Architectural invariants

## DB-CAST-001
Casting transforma representaciones; no define tipos persistentes.

## DB-CAST-002
Casting no sustituye al Type System.

## DB-CAST-003
Casting no sustituye a Value Conversion.

## DB-CAST-004
Casting no sustituye a Hydration.

## DB-CAST-005
Casting no sustituye a Serialization.

## DB-CAST-006
Casting no sustituye a Validation.

## DB-CAST-007
Casting no ejecutará SQL.

## DB-CAST-008
Casting no abrirá conexiones.

## DB-CAST-009
Casting no ejecutará queries.

## DB-CAST-010
Casting no hará flush.

## DB-CAST-011
Casting no hará commit.

## DB-CAST-012
Casting no registrará entidades.

## DB-CAST-013
Cast definitions serán declarativas.

## DB-CAST-014
Cast definitions serán normalizadas en bootstrap.

## DB-CAST-015
CastRegistry estará frozen durante runtime productivo.

## DB-CAST-016
Hot path no descubrirá casts mediante reflection.

## DB-CAST-017
Model API y Data Mapper compartirán el mismo engine.

## DB-CAST-018
No existirán dos motores de casting.

## DB-CAST-019
Cast arguments serán estructurados internamente.

## DB-CAST-020
Cast string parsing no ocurrirá por field access.

## DB-CAST-021
Inbound y outbound serán direcciones explícitas.

## DB-CAST-022
Un outbound cast no será asumido inbound.

## DB-CAST-023
Un inbound cast no será asumido outbound.

## DB-CAST-024
Reversibility será metadata explícita.

## DB-CAST-025
Lossy casts no serán bidireccionales implícitamente.

## DB-CAST-026
Cast pipelines serán validados.

## DB-CAST-027
Cast pipelines serán finitos.

## DB-CAST-028
Cast cycles serán rechazados.

## DB-CAST-029
CompiledCastPipeline será immutable.

## DB-CAST-030
Pipeline Compiler será distinto de Pipeline Executor.

## DB-CAST-031
Scalar casts no duplicarán driver conversion.

## DB-CAST-032
Decimal no será casteado automáticamente a float.

## DB-CAST-033
Collection cast no será PersistentCollection.

## DB-CAST-034
Collection cast no disparará relationship loading.

## DB-CAST-035
Enum unknown value no será null silenciosamente.

## DB-CAST-036
Temporal casts respetarán semántica del temporal type.

## DB-CAST-037
Formatting de fechas no se confundirá con persistencia temporal.

## DB-CAST-038
Value Object cast no convertirá automáticamente multi-column mappings.

## DB-CAST-039
Multi-column Value Objects pertenecerán al Value Object Mapping System.

## DB-CAST-040
Custom casts se registrarán explícitamente.

## DB-CAST-041
DB values no seleccionarán custom cast classes.

## DB-CAST-042
User input no seleccionará arbitrary cast classes.

## DB-CAST-043
Core casts serán stateless cuando sea posible.

## DB-CAST-044
Casts no harán hidden I/O.

## DB-CAST-045
Casts no harán lazy loading.

## DB-CAST-046
Casts no consultarán repositories.

## DB-CAST-047
NULL handling será explícito.

## DB-CAST-048
MISSING no será tratado como NULL.

## DB-CAST-049
LoadedFieldMask será respetado.

## DB-CAST-050
Partial hydration no fabricará valores casteados.

## DB-CAST-051
Dirty tracking no dependerá únicamente de object identity.

## DB-CAST-052
Dirty tracking podrá comparar canonical persistent values.

## DB-CAST-053
Mutable casts deberán declarar snapshot semantics.

## DB-CAST-054
Snapshot mutable no compartirá referencia con current value.

## DB-CAST-055
Immutable cast values serán preferidos.

## DB-CAST-056
ChangeSet no contendrá driver-ready values.

## DB-CAST-057
Inbound cast precederá a Value Conversion en escritura.

## DB-CAST-058
Value Conversion precederá a outbound cast en lectura.

## DB-CAST-059
PostLoad mutations podrán ser detectadas como dirty.

## DB-CAST-060
Pipeline cache almacenará comportamiento, no entity values.

## DB-CAST-061
Global cast result cache estará prohibido.

## DB-CAST-062
Entity-local cast cache será scoped.

## DB-CAST-063
Value Objects casteados no entrarán al IdentityMap.

## DB-CAST-064
Entity identity y cast value identity permanecerán separadas.

## DB-CAST-065
Cast result cache se invalidará al cambiar el valor subyacente.

## DB-CAST-066
V1 favorecerá application representation dentro de la entidad.

## DB-CAST-067
Canonicalization ocurrirá en boundaries ORM apropiados.

## DB-CAST-068
Property PHP type deberá ser compatible con cast output.

## DB-CAST-069
Mismatch detectable deberá fallar en bootstrap.

## DB-CAST-070
Union types requerirán compatibilidad demostrable.

## DB-CAST-071
Nested cast pipelines tendrán depth limits.

## DB-CAST-072
Pipeline fingerprints incluirán registry generation.

## DB-CAST-073
Cambiar un cast invalidará pipelines afectados.

## DB-CAST-074
Deterministic casts producirán resultados semánticamente deterministas.

## DB-CAST-075
Non-deterministic generators no serán casts persistentes normales.

## DB-CAST-076
Casts no dependerán de global locale.

## DB-CAST-077
Casts no dependerán de global timezone.

## DB-CAST-078
Environment-dependent parameters serán explícitos.

## DB-CAST-079
Serialization seguirá siendo una capa separada.

## DB-CAST-080
Field visibility no será responsabilidad del cast.

## DB-CAST-081
Encryption delegará a Encryption System.

## DB-CAST-082
Hashing one-way no será tratado como reversible cast.

## DB-CAST-083
Transform order será explícito.

## DB-CAST-084
Pipeline compiler validará compatibilidad entre steps.

## DB-CAST-085
Aliases se canonicalizarán antes del hot path.

## DB-CAST-086
Deprecated aliases podrán emitir diagnostics.

## DB-CAST-087
Casting tendrá resource governance.

## DB-CAST-088
Unbounded nested structures serán rechazables.

## DB-CAST-089
Casting no utilizará unsafe unserialize por defecto.

## DB-CAST-090
Exceptions no revelarán valores sensibles por defecto.

## DB-CAST-091
Telemetry no incluirá raw cast values.

## DB-CAST-092
No se creará un span por cada cast normal.

## DB-CAST-093
Compiled pipelines podrán compartirse en persistent runtimes.

## DB-CAST-094
Mutable cast execution state será scoped.

## DB-CAST-095
FrankenPHP requests no compartirán entity cast state.

## DB-CAST-096
RoadRunner requests no compartirán entity cast state.

## DB-CAST-097
OpenSwoole requerirá coroutine-safe mutable state.

## DB-CAST-098
Static caches no almacenarán entity values.

## DB-CAST-099
Development reload generará nuevas generations.

## DB-CAST-100
HydrationPlan podrá referenciar compiled cast pipelines.

## DB-CAST-101
Repeated field access no recompilará metadata.

## DB-CAST-102
Round-trip será probado para casts reversibles.

## DB-CAST-103
Dirty tracking tendrá tests específicos por cast mutable.

## DB-CAST-104
No-op hydration+flush no deberá generar falsos UPDATE.

## DB-CAST-105
Equivalent canonical values no deberán generar false dirty.

## DB-CAST-106
Real canonical changes deberán producir ChangeSet.

## DB-CAST-107
NULL y MISSING tendrán tests distintos.

## DB-CAST-108
Property type mismatches tendrán bootstrap tests.

## DB-CAST-109
Persistent worker isolation será parte del conformance suite.

## DB-CAST-110
Casting correctness tendrá prioridad sobre convenience.

## DB-CAST-111
Casting será explainable.

## DB-CAST-112
Cast explain no necesitará raw sensitive values.

## DB-CAST-113
Cast registry lookup no dependerá de database content.

## DB-CAST-114
Cast behavior no seleccionará DB platform arbitrariamente.

## DB-CAST-115
Platform representation seguirá siendo responsabilidad de Value Conversion.

## DB-CAST-116
Casting no producirá SQL literals.

## DB-CAST-117
Casting no decidirá parameter binding type.

## DB-CAST-118
Casting no gestionará transaction state.

## DB-CAST-119
Casting no será segundo persistence engine.

## DB-CAST-120
VoltStack mantendrá una única representación persistente canónica entre Casting y Value Conversion.

---

# 211. Anti-patterns

### Cast que ejecuta query

```php
public function outbound($id)
{
    return User::find($id);
}
```

**Rechazado.**

Eso es relationship/entity loading.

---

### Cast que usa PDO

```php
$this->pdo->query(...);
```

**Rechazado.**

---

### Cast que redondea silenciosamente

```text
100.129 → 100.13
```

sin policy.

**Rechazado.**

---

### Cast que depende del usuario autenticado

```text
currentUser.locale
```

**Rechazado como cast persistente base.**

---

### Cast de relación

```text
user_id → User entity
```

**Rechazado.**

Eso pertenece al Relationship Loading System.

---

### Cast multi-column improvisado

```text
amount + currency → Money
```

desde un cast escalar.

**Rechazado.**

Usar Value Object Mapping.

---

### Cast global mutable

```php
static $lastValue;
```

**Rechazado.**

---

### Cast que genera valores

```text
null → UUID::random()
```

**Rechazado como cast.**

Usar value/default generator.

---

# 212. Ejemplo completo — Enum

Entidad:

```php
final class Order
{
    #[Column(type: 'string')]
    #[Cast('enum', arguments: [OrderStatus::class])]
    private OrderStatus $status;
}
```

Lectura:

```text
DB
 │
 ▼
"paid"
 │
 ▼
Value Conversion
 │
 ▼
canonical string "paid"
 │
 ▼
Enum Cast
 │
 ▼
OrderStatus::PAID
 │
 ▼
Order::$status
```

Escritura:

```text
OrderStatus::CANCELLED
 │
 ▼
Enum Cast
 │
 ▼
"cancelled"
 │
 ▼
Value Conversion
 │
 ▼
Driver binding
 │
 ▼
DB
```

---

# 213. Ejemplo — JSON Collection

```php
#[Column(type: 'json')]
#[Cast('collection')]
private Collection $preferences;
```

Lectura:

```text
DB JSON
   ↓
JSON Type Conversion
   ↓
canonical array
   ↓
Collection Cast
   ↓
Collection
```

No existe:

```text
relationship loading
```

en ese pipeline.

---

# 214. Ejemplo — EmailAddress

```php
#[Column(type: 'string')]
#[Cast('email')]
private EmailAddress $email;
```

Pipeline outbound:

```text
canonical string
→ EmailAddress
```

Pipeline inbound:

```text
EmailAddress
→ canonical string
```

Dirty comparison:

```text
canonical(old)
vs
canonical(new)
```

---

# 215. Ejemplo — Dirty tracking

Estado hidratado:

```text
canonical:
"user@example.com"

application:
EmailAddress("user@example.com")
```

Usuario asigna:

```php
$user->email = new EmailAddress('USER@example.com');
```

Si la semántica del cast define normalización:

```text
canonical(new)
=
"user@example.com"
```

entonces:

```text
old canonical == new canonical
```

Resultado:

```text
NO UPDATE
```

---

# 216. Ejemplo — Mutable JSON collection

Hydration:

```text
DB JSON
→ array
→ MutableCollection
```

Snapshot:

```text
canonical independent snapshot
```

Aplicación:

```php
$user->settings->set('theme', 'dark');
```

Change Tracking:

```text
current canonical value
≠
snapshot canonical value
```

Resultado:

```text
DIRTY
```

---

# 217. Cast explain

API conceptual:

```php
Database::types()
    ->casts()
    ->explain(User::class, 'status');
```

Resultado:

```text
CAST PLAN

Entity:
    User

Property:
    status

Persistent Type:
    string

Cast:
    enum

Arguments:
    UserStatus

Direction:
    BIDIRECTIONAL

Outbound:
    string → UserStatus

Inbound:
    UserStatus → string

Mutability:
    IMMUTABLE

Reversibility:
    REVERSIBLE

Snapshot:
    canonical persistent value

Pipeline Cache:
    enabled
```

---

# 218. Master formula

Para lectura:

```text
ApplicationValue
=
OutboundCast(
    CanonicalPersistentValue,
    CastDefinition
)
```

Para escritura:

```text
CanonicalPersistentValue
=
InboundCast(
    ApplicationValue,
    CastDefinition
)
```

---

# 219. Persist formula

```text
DatabaseValue
=
ValueConversion(
    InboundCast(
        ApplicationValue
    )
)
```

---

# 220. Hydration formula

```text
ApplicationValue
=
OutboundCast(
    ValueConversion(
        DriverValue
    )
)
```

---

# 221. Dirty formula

```text
Dirty
=
Canonicalize(CurrentApplicationValue)
≠
PersistentSnapshot
```

y no simplemente:

```text
CurrentObject !== OriginalObject
```

---

# 222. Architectural master model

```text
                      DATABASE
                          │
                          ▼
                   Driver Raw Value
                          │
                          ▼
                 VALUE CONVERSION
                          │
                          ▼
              Canonical Persistent Value
                          │
                          ▼
                       CASTING
                          │
                          ▼
                  Application Value
                          │
                          ▼
                       ENTITY
                          │
                     modification
                          │
                          ▼
                  Application Value
                          │
                          ▼
                 INBOUND CASTING
                          │
                          ▼
              Canonical Persistent Value
                          │
                          ▼
                  CHANGE TRACKING
                          │
                          ▼
                PERSISTENCE ENGINE
                          │
                          ▼
                 VALUE CONVERSION
                          │
                          ▼
                    DATABASE
```

---

# 223. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Type-Aware Casting
```

sobre casts aislados basados únicamente en strings.

Adoptará:

```text
Compiled Cast Pipelines
```

para hot paths.

Adoptará:

```text
Canonical Persistent Values
```

como frontera entre Casting y Value Conversion.

Adoptará:

```text
Bidirectional Cast Contracts
```

solo cuando exista reversibilidad demostrable.

Adoptará:

```text
Immutable Cast Values
```

como recomendación principal.

Adoptará:

```text
Canonical Dirty Comparison
```

para evitar falsos cambios.

Adoptará:

```text
One Casting Engine
```

para Active Record y Data Mapper.

---

# 224. Regla maestra final

> **El Casting System de VoltStack existe para hacer que el modelo de aplicación pueda trabajar con representaciones expresivas sin contaminar el Type System, el Driver o el Persistence Engine con detalles de conveniencia de la aplicación.**

La frontera será:

```text
Database Representation
        ↕
Value Conversion
        ↕
Canonical Persistent Value
        ↕
Casting
        ↕
Application Representation
```

y nunca:

```text
Database
↔
arbitrary cast
↔
Entity
```

De esta manera, VoltStack podrá ofrecer ergonomía comparable a sistemas Active Record manteniendo las propiedades de consistencia, tipado, aislamiento y predictibilidad de una arquitectura Data Mapper.

---

# 225. Siguiente documento

```text
159_DATABASE_ENUM_MAPPING_SYSTEM.md
```

El siguiente documento deberá especializar el Type/Casting architecture para enums y definir:

- native PHP enums;
- backed enums;
- pure enums;
- enum mapping metadata;
- canonical enum identity;
- backing values;
- database representation;
- native DB enum vs portable representation;
- string/integer-backed enums;
- unknown backing values;
- legacy values;
- enum aliases;
- enum migrations;
- enum evolution;
- removed cases;
- renamed cases;
- compatibility windows;
- zero-downtime enum changes;
- schema integration;
- Type System integration;
- Value Conversion;
- Casting;
- hydration;
- dirty tracking;
- query parameters;
- enum predicates;
- arrays/sets of enums;
- defaults;
- validation boundaries;
- security;
- diagnostics;
- telemetry;
- persistent runtime;
- testing;
- cross-platform behavior;
- architectural invariants.

Regla central propuesta:

> **Un enum persistente en VoltStack será un dominio cerrado de valores lógicos cuya identidad pertenece al modelo de aplicación y cuya representación física en la base de datos será una decisión de mapping; el backing value persistido nunca deberá confundirse con la identidad conceptual del enum ni permitir que valores desconocidos sean aceptados silenciosamente.**