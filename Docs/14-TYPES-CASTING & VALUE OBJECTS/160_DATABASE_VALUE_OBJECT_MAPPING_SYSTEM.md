# 160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md

# VoltStack Quantum Database
## Database Value Object Mapping System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 160 — Database Value Object Mapping System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `159_DATABASE_ENUM_MAPPING_SYSTEM.md`  
**Siguiente documento:** `161_DATABASE_JSON_TYPE_SYSTEM.md`

---

# 1. Propósito

`Database Value Object Mapping System` define la arquitectura mediante la cual VoltStack podrá persistir objetos de valor del dominio sin convertirlos en entidades ORM.

Ejemplos:

```php
final readonly class EmailAddress
{
    public function __construct(
        public string $value,
    ) {}
}
```

```php
final readonly class Money
{
    public function __construct(
        public string $amount,
        public Currency $currency,
    ) {}
}
```

```php
final readonly class Address
{
    public function __construct(
        public string $street,
        public string $city,
        public string $postalCode,
        public CountryCode $country,
    ) {}
}
```

Una entidad podrá contener:

```php
final class Customer
{
    private EmailAddress $email;
    private Address $billingAddress;
}
```

sin que `EmailAddress` o `Address` adquieran:

- EntityManager lifecycle;
- EntityState;
- IdentityMap identity;
- UnitOfWork identity;
- repository;
- relationships propias;
- primary key;
- persistence lifecycle independiente.

La regla central será:

> **Un Value Object persistente en VoltStack será tratado como un valor de dominio sin identidad ORM independiente: podrá ocupar una o varias representaciones persistentes, pero nunca será promovido implícitamente a Entity, IdentityMap entry o Relationship únicamente por estar compuesto por múltiples campos.**

---

# 2. Problema

Los tipos escalares son insuficientes para representar muchos conceptos del dominio.

Esto:

```php
private string $email;
```

permite demasiados estados.

Es preferible:

```php
private EmailAddress $email;
```

Igualmente:

```php
private string $amount;
private string $currency;
```

puede convertirse en:

```php
private Money $price;
```

El ORM debe comprender cómo transformar:

```text
Domain Value Object
        ↕
Persistent Field Set
```

sin romper las fronteras arquitectónicas del Database System.

---

# 3. Modelo fundamental

```text
Entity
  │
  ├── scalar property
  ├── enum property
  │
  └── Value Object property
          │
          ▼
   ValueObjectMapping
          │
          ▼
   Persistent Components
      ├── column A
      ├── column B
      └── column C
```

Ejemplo:

```text
Customer.address
        │
        ▼
Address
├── street
├── city
├── postalCode
└── country
        │
        ▼
customers
├── address_street
├── address_city
├── address_postal_code
└── address_country
```

---

# 4. Value Object ≠ Entity

Esta será la separación principal.

```text
Value Object
≠
Entity
```

Una Entity posee identidad persistente:

```text
User#42
```

Un Value Object posee valor estructural:

```text
Money("100.00", "MXN")
```

---

# 5. Entity identity

Dos referencias a:

```text
User#42
```

representan la misma identidad ORM.

---

# 6. Value equality

Dos objetos:

```php
new Money('100.00', Currency::MXN)
```

y:

```php
new Money('100.00', Currency::MXN)
```

pueden ser objetos PHP diferentes y representar el mismo valor.

Formalmente:

```text
ObjectIdentity(A) ≠ ObjectIdentity(B)
```

pero:

```text
Value(A) = Value(B)
```

---

# 7. Value Object ≠ Relationship

Esto:

```php
private Address $address;
```

no implica:

```text
ManyToOne(Address)
```

---

# 8. Value Object ≠ Table

Un Value Object compuesto no requiere automáticamente tabla propia.

Puede ser:

```text
flattened columns
```

en la tabla propietaria.

---

# 9. Value Object ≠ DTO

Un DTO transporta datos.

Un Value Object expresa semántica del dominio y normalmente invariantes.

```text
Value Object
≠
DTO
```

---

# 10. Value Object ≠ Cast

Un cast normalmente transforma una representación.

El Value Object Mapping puede abarcar:

```text
1 property
→
N persistent fields
```

Por eso:

```text
ValueObjectMapping
≠
Scalar Cast
```

---

# 11. Value Object ≠ Serialization

Persistir:

```text
Address
```

como columnas o JSON no implica cómo se presenta en una API.

---

# 12. Value Object ≠ JSON

Un Value Object puede persistirse como:

```text
columns
JSON
single scalar
custom composite representation
```

JSON será únicamente una estrategia posible.

---

# 13. Ownership

Todo Value Object persistido estará contenido dentro de una raíz persistente.

Normalmente:

```text
Entity
→ owns Value Object
```

---

# 14. No independent lifecycle

Un Value Object no tendrá:

```text
persist($address)
remove($address)
repository(Address::class)
find(Address::class, ...)
```

por defecto.

---

# 15. Persistence lifecycle

Su lifecycle será parte del propietario:

```text
Customer
  ↓
Address
```

Si cambia `Address`, cambia el estado persistente de `Customer`.

---

# 16. ValueObjectTypeId

Cada mapping podrá utilizar una identidad lógica estable:

```php
final readonly class ValueObjectTypeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
common.email_address
commerce.money
common.address
common.date_range
geo.point
```

---

# 17. TypeId ≠ PHP class

```text
common.address
≠
App\Domain\Address
```

El FQCN podrá cambiar sin alterar necesariamente el mapping lógico.

---

# 18. ValueObjectMetadata

```php
final readonly class ValueObjectMetadata
{
    /**
     * @param list<ValueObjectComponentMetadata> $components
     */
    public function __construct(
        public ValueObjectTypeId $type,
        public PHPValueObjectReference $phpType,
        public ValueObjectMappingStrategy $strategy,
        public ValueObjectConstructionMetadata $construction,
        public array $components,
        public ValueObjectEqualityPolicy $equality,
        public ValueObjectMutability $mutability,
    ) {}
}
```

---

# 19. Field mapping

La entidad requiere además:

```php
final readonly class EmbeddedValueObjectMetadata
{
    public function __construct(
        public PropertyPath $property,
        public ValueObjectTypeId $type,
        public ValueObjectNullability $nullability,
        public ColumnPrefix $prefix,
        public ValueObjectOverrides $overrides,
    ) {}
}
```

---

# 20. Mapping strategies

```php
enum ValueObjectMappingStrategy
{
    case SINGLE_COLUMN;
    case FLATTENED_COLUMNS;
    case JSON;
    case NATIVE_COMPOSITE;
    case CUSTOM;
}
```

---

# 21. Single-column Value Object

Ejemplo:

```text
EmailAddress
    ↓
string
```

Entidad:

```php
private EmailAddress $email;
```

DB:

```text
email VARCHAR(...)
```

---

# 22. Single-column pipeline

```text
EmailAddress
      ↓
Value Object Mapper
      ↓
canonical string
      ↓
Type System
      ↓
Value Conversion
      ↓
Database
```

---

# 23. Single-column vs Cast

Un Value Object de un solo valor puede implementarse eficientemente sobre Casting.

Por tanto:

```text
SingleColumnValueObjectMapping
→ may compile to Cast Pipeline
```

pero la metadata sigue declarando semántica de Value Object.

---

# 24. Flattened multi-column mapping

Ejemplo:

```php
final readonly class Money
{
    public function __construct(
        public string $amount,
        public Currency $currency,
    ) {}
}
```

DB:

```text
price_amount
price_currency
```

---

# 25. Mapping

```text
Money.amount
   → price_amount

Money.currency
   → price_currency
```

---

# 26. Address example

```text
Address.street
       ↓
billing_street

Address.city
       ↓
billing_city

Address.postalCode
       ↓
billing_postal_code

Address.country
       ↓
billing_country
```

---

# 27. ColumnPrefix

VoltStack podrá generar nombres mediante:

```text
property name
+
prefix strategy
+
component column
```

---

# 28. Default prefix

Ejemplo:

```php
private Address $billingAddress;
```

podría generar:

```text
billing_address_street
billing_address_city
billing_address_postal_code
billing_address_country
```

---

# 29. Explicit prefix

```php
#[Embedded(prefix: 'billing_')]
private Address $billingAddress;
```

produce:

```text
billing_street
billing_city
...
```

---

# 30. Prefix disabled

Podrá permitirse:

```php
#[Embedded(prefix: false)]
```

si no produce colisiones.

---

# 31. Collision detection

Esto deberá fallar en metadata compilation:

```text
Address.street → street
Contact.street → street
```

si ambas propiedades producen la misma columna.

---

# 32. Overrides

Un componente podrá sobrescribir su binding.

Ejemplo:

```text
Address.postalCode
→ zip_code
```

---

# 33. ValueObjectOverrides

```php
final readonly class ValueObjectOverrides
{
    /**
     * @param array<string, ValueObjectComponentOverride> $components
     */
    public function __construct(
        public array $components,
    ) {}
}
```

---

# 34. Nested Value Objects

Debe soportarse:

```text
Address
├── StreetAddress
│   ├── line1
│   └── line2
├── City
└── PostalCode
```

---

# 35. Nested path

La metadata utilizará rutas:

```text
address.street.line1
address.street.line2
address.city
address.postalCode
```

---

# 36. Flattening

Estas rutas podrán convertirse en:

```text
address_street_line1
address_street_line2
address_city
address_postal_code
```

---

# 37. Depth

La profundidad deberá estar limitada por policy.

---

# 38. Cycle detection

Esto será inválido:

```text
A contains B
B contains A
```

para mapping embebido por valor.

---

# 39. Why

Un Value Object graph persistente deberá ser:

```text
finite
+
acyclic
```

en su representación embebida.

---

# 40. Component metadata

```php
final readonly class ValueObjectComponentMetadata
{
    public function __construct(
        public ValueObjectComponentPath $path,
        public TypeReference|ValueObjectTypeId $type,
        public bool $nullable,
        public ValueObjectComponentBinding $binding,
    ) {}
}
```

---

# 41. Scalar component

```text
Money.amount
→ decimal
```

---

# 42. Enum component

```text
Money.currency
→ Currency enum
```

---

# 43. Nested Value Object component

```text
Address.postalCode
→ PostalCode Value Object
```

---

# 44. Relationship component

Por defecto:

```text
Value Object
→ ORM Relationship
```

no deberá permitirse.

---

# 45. Reason

Permitir relaciones dentro de Value Objects introduce:

- ownership ambiguity;
- cascade ambiguity;
- lazy loading dentro de values;
- EntityManager coupling;
- equality ambiguity.

V1 deberá evitarlo.

---

# 46. Entity reference inside Value Object

Si el dominio necesita algo equivalente, favorecer:

```text
EntityId Value Object
```

o rediseñar el aggregate mapping explícitamente.

---

# 47. Construction

Hydration debe poder reconstruir el Value Object.

---

# 48. Constructor strategy

Ejemplo:

```php
new Money(
    amount: $amount,
    currency: $currency,
);
```

---

# 49. Named constructor

También:

```php
Money::of(
    $amount,
    $currency,
);
```

---

# 50. Factory

Podrá existir:

```text
ValueObjectFactory
```

registrada explícitamente.

---

# 51. No unsafe hydration

VoltStack no deberá depender obligatoriamente de:

```text
ReflectionClass::newInstanceWithoutConstructor()
```

para Value Objects.

---

# 52. Constructor-first

Default recomendado:

> **Los Value Objects deberán construirse mediante sus invariantes públicas siempre que sea posible.**

---

# 53. Construction metadata

```php
final readonly class ValueObjectConstructionMetadata
{
    public function __construct(
        public ValueObjectConstructionStrategy $strategy,
        public ?string $factoryMethod,
        public array $argumentBindings,
    ) {}
}
```

---

# 54. Construction strategies

```php
enum ValueObjectConstructionStrategy
{
    case CONSTRUCTOR;
    case STATIC_FACTORY;
    case REGISTERED_FACTORY;
}
```

---

# 55. Constructor arguments

El mapping debe conocer:

```text
component path
→ constructor parameter
```

---

# 56. Constructor parameter order

No deberá depender de un orden implícito de columnas.

---

# 57. Named binding

Preferir:

```text
amount → $amount
currency → $currency
```

---

# 58. Hydration architecture

```text
Result Row
   ↓
Value Conversion
   ↓
Component Canonical Values
   ↓
Component Casting / Enum Mapping
   ↓
ValueObjectHydrationPlan
   ↓
ValueObjectAssembler
   ↓
Value Object
   ↓
Entity Hydrator
```

---

# 59. Hydration Plan integration

`DATABASE_HYDRATION_PLAN_SYSTEM` deberá poder contener:

```text
ValueObjectHydrationNode
```

---

# 60. ValueObjectHydrationNode

Conceptualmente:

```php
final readonly class ValueObjectHydrationNode
{
    public function __construct(
        public ValueObjectTypeId $type,
        public array $componentBindings,
        public ValueObjectConstructorPlan $constructor,
        public ValueObjectNullabilityPlan $nullability,
    ) {}
}
```

---

# 61. No IdentityMap

El assembler no consultará IdentityMap para deduplicar Value Objects.

---

# 62. Why

Dos entidades pueden contener:

```text
Money(100, MXN)
```

y seguir teniendo dos objetos PHP equivalentes sin violar ninguna identidad ORM.

---

# 63. Optional interning

VoltStack V1 no deberá internar Value Objects globalmente.

---

# 64. Reason

Interning introduce:

- memory retention;
- cross-request state;
- unexpected reference identity;
- complexity;
- limited benefit.

---

# 65. Nullability

Composite Value Objects requieren semántica explícita.

Ejemplo:

```text
address_street = NULL
address_city   = NULL
address_zip    = NULL
```

¿significa:

```text
address = null
```

o:

```text
Address(null, null, null)
```

?

---

# 66. ValueObjectNullability

```php
enum ValueObjectNullability
{
    case REQUIRED;
    case NULLABLE_AS_WHOLE;
    case COMPONENT_NULLABILITY;
}
```

---

# 67. Nullable as whole

Para:

```php
private ?Address $address;
```

todos los componentes NULL pueden representar:

```text
address = null
```

---

# 68. Partial NULL composite

Ejemplo:

```text
street = NULL
city   = "Mexico City"
zip    = NULL
```

no debe interpretarse automáticamente como `null`.

---

# 69. Composite null state

Debe distinguir:

```text
ALL_NULL
ALL_PRESENT
PARTIALLY_NULL
```

---

# 70. Partial null policy

Si los componentes requeridos están parcialmente NULL:

```text
→ inconsistent persistent state
```

salvo que la metadata permita esos componentes nullable.

---

# 71. ValueObjectNullState

```php
enum ValueObjectNullState
{
    case ABSENT;
    case PRESENT;
    case PARTIAL;
}
```

---

# 72. Required Value Object

Si es REQUIRED y todas las columnas son NULL:

```text
→ ValueObjectHydrationException
```

---

# 73. Missing columns

Nuevamente:

```text
MISSING
≠
NULL
```

---

# 74. Partial hydration

Una query puede seleccionar:

```text
address_city
```

pero no:

```text
address_street
address_zip
```

---

# 75. No fabricated Value Object

VoltStack no deberá construir:

```php
new Address(
    street: null,
    city: '...',
    postalCode: null,
);
```

si los otros campos simplemente no fueron seleccionados.

---

# 76. Partial Value Object

V1 deberá favorecer:

```text
projection
```

en vez de Value Objects parcialmente hidratados.

---

# 77. Strict default

Para una propiedad Value Object normal:

> **Todos los componentes requeridos por su Hydration Plan deberán estar disponibles antes de construirla.**

---

# 78. Projection exception

Una projection puede seleccionar:

```text
address.city
```

sin construir el `Address` completo.

---

# 79. Query Engine

Debe permitir:

```php
Customer::query()
    ->where('billingAddress.city', 'Monterrey');
```

---

# 80. Semantic resolution

```text
billingAddress.city
       ↓
Entity Metadata
       ↓
Embedded Value Object Metadata
       ↓
Component Binding
       ↓
billing_city
```

---

# 81. Query Builder ≠ flattening engine

El Query Builder conserva el property path.

La resolución ocurre en:

```text
Semantic Query System
```

---

# 82. Query AST

Puede representar:

```text
PropertyPathExpression(
    Customer.billingAddress.city
)
```

---

# 83. SQL Compiler

No necesita conocer `Address`.

Recibirá el binding físico ya resuelto.

---

# 84. Value Object equality query

Puede soportarse:

```php
->where('price', Money::of('100.00', 'MXN'))
```

si el mapping sabe expandirlo.

---

# 85. Composite predicate

Conceptualmente:

```text
price = Money(100, MXN)
```

se convierte en:

```text
price_amount = 100
AND
price_currency = MXN
```

---

# 86. Composite equality semantics

No debe ser hardcoded.

El mapping define qué componentes participan.

---

# 87. Query expansion

```text
ValueObjectPredicate
        ↓
Semantic Expansion
        ↓
Component Predicates
```

---

# 88. Query expansion ≠ SQL

El resultado seguirá siendo Query AST/semantic predicates.

---

# 89. Inequality

```text
price != Money(...)
```

es más complejo que simplemente:

```text
amount != X AND currency != Y
```

La negación correcta es:

```text
NOT (
    amount = X
    AND currency = Y
)
```

---

# 90. Semantic correctness

El Query Engine deberá expandir operadores compuestos respetando lógica booleana.

---

# 91. Ordering

Ordenar por:

```text
Money
```

no tiene semántica universal.

---

# 92. Component ordering

Sí puede ser válido:

```php
->orderBy('price.amount')
```

---

# 93. Value Object ordering

Solo podrá existir si metadata declara una:

```text
ValueObjectOrderingPolicy
```

---

# 94. Range objects

Ejemplo:

```php
DateRange(
    start,
    end
)
```

puede soportar operaciones semánticas:

```text
contains
overlaps
before
after
```

---

# 95. Operator extensions

Estas operaciones deberán integrarse mediante:

```text
Query Extension System
```

no mediante SQL embebido en el Value Object.

---

# 96. Persistence architecture

```text
Entity
   ↓
UnitOfWork
   ↓
Change Tracking
   ↓
Value Object Decomposition
   ↓
Component ChangeSet
   ↓
Persistence Planner
   ↓
Query Model
```

---

# 97. Decomposition

```php
interface ValueObjectDecomposer
{
    public function decompose(
        ValueObjectMetadata $metadata,
        object $value,
    ): ValueObjectComponentSet;
}
```

---

# 98. ComponentSet

```php
final readonly class ValueObjectComponentSet
{
    public function __construct(
        public array $values,
    ) {}
}
```

---

# 99. No SQL

`ValueObjectDecomposer` produce valores canónicos/semánticos.

No:

```text
column SQL fragments
```

---

# 100. Dirty tracking

Este sistema debe evitar:

```php
$oldMoney !== $newMoney
```

como única regla.

---

# 101. Structural equality

Para Value Objects:

```text
Dirty
=
!ValueEquivalent(
    OldValue,
    NewValue
)
```

---

# 102. Default equality

Puede basarse en:

```text
canonical persistent component set
```

---

# 103. Example

```php
new Money('100.0', MXN)
```

y:

```php
new Money('100.00', MXN)
```

pueden ser equivalentes si el decimal type canonicaliza ambos a:

```text
100.00
```

---

# 104. Canonical equality

Formalmente:

```text
VO(A) ≡ VO(B)

iff

CanonicalComponents(A)
=
CanonicalComponents(B)
```

para la policy default.

---

# 105. Custom equality

El dominio puede necesitar otra semántica.

Deberá declararse mediante:

```text
ValueObjectComparator
```

---

# 106. Comparator contract

```php
interface ValueObjectComparator
{
    public function equivalent(
        object $left,
        object $right,
        ValueObjectComparisonContext $context,
    ): bool;
}
```

---

# 107. Immutable Value Objects

Serán el modelo recomendado.

```php
final readonly class Money
```

---

# 108. Benefits

- snapshots simples;
- thread/coroutine safety;
- predictable dirty tracking;
- no hidden mutation;
- safer cache behavior.

---

# 109. Mutable Value Objects

Podrán soportarse únicamente con estrategia explícita.

---

# 110. Mutation example

```php
$customer->address->setCity('Puebla');
```

No existe property replacement.

---

# 111. Mutable snapshot

Change Tracking deberá conservar un snapshot independiente.

---

# 112. ValueObjectMutability

```php
enum ValueObjectMutability
{
    case IMMUTABLE;
    case MUTABLE;
}
```

---

# 113. Snapshot

```php
interface ValueObjectSnapshotStrategy
{
    public function snapshot(
        object $value,
        ValueObjectMappingContext $context,
    ): mixed;
}
```

---

# 114. Recommended snapshot

Guardar:

```text
Canonical Component Set
```

no el mismo objeto mutable.

---

# 115. Snapshot ≠ cloning blindly

`clone` puede no ser suficiente si el Value Object contiene estructuras mutables anidadas.

---

# 116. UnitOfWork snapshot

Ejemplo:

```text
Customer#42

price snapshot:
{
    amount: "100.00",
    currency: "MXN"
}
```

---

# 117. ChangeSet

Si cambia:

```text
Money(100, MXN)
→
Money(150, MXN)
```

se puede producir:

```text
price.amount:
    100 → 150
```

---

# 118. Component-level updates

Persistence Planner podrá actualizar únicamente componentes modificados cuando sea seguro.

---

# 119. Atomic Value Object semantics

Sin embargo, conceptualmente el cambio pertenece al Value Object completo.

---

# 120. Component update ≠ independent domain mutation

Aunque SQL actualice una sola columna:

```text
UPDATE price_amount
```

el ORM deberá conservar que:

```text
Customer.price
```

es una unidad de valor.

---

# 121. Atomic persistence policy

Podrá existir:

```php
enum ValueObjectUpdatePolicy
{
    case CHANGED_COMPONENTS;
    case ALL_COMPONENTS;
}
```

---

# 122. CHANGED_COMPONENTS

Optimiza escrituras.

---

# 123. ALL_COMPONENTS

Puede ser útil cuando:

- DB triggers;
- encryption;
- composite integrity;
- generated hashes;

requieren representación completa.

---

# 124. Generated values

Un Value Object podrá contener componentes DB-generated únicamente si la estrategia de hydration/reconciliation está definida explícitamente.

---

# 125. Recommended V1

Evitar Value Objects parcialmente DB-generated salvo necesidad clara.

---

# 126. Insert

En INSERT:

```text
Value Object
→ decompose
→ component values
→ Query Model
```

---

# 127. Update

En UPDATE:

```text
Current Value Object
+
Snapshot
→ compare/decompose
→ component ChangeSet
```

---

# 128. Delete

Eliminar la Entity elimina sus columnas embebidas como parte de la fila.

No existe:

```text
DELETE ValueObject
```

independiente.

---

# 129. Flush

`flush()` no persiste Value Objects independientemente.

Los cambios viajan mediante su owner.

---

# 130. Cascades

No existen cascades ORM tradicionales entre Entity y embedded Value Object.

---

# 131. Orphan removal

No aplica.

Reemplazar:

```text
Address A
→ Address B
```

no elimina una entidad `Address`.

---

# 132. Shared Value Objects

Dos entidades pueden utilizar el mismo objeto PHP immutable:

```php
$money = new Money('100.00', MXN);

$a->price = $money;
$b->price = $money;
```

sin crear ownership persistente compartido.

---

# 133. Persistence independence

Cada owner persiste su propia representación.

---

# 134. Value Object Registry

VoltStack utilizará:

```text
ValueObjectMappingRegistry
```

---

# 135. Contract

```php
interface ValueObjectMappingRegistry
{
    public function get(
        ValueObjectTypeId $type
    ): ValueObjectMetadata;

    public function forClass(
        string $class
    ): ValueObjectMetadata;
}
```

---

# 136. Registry lifecycle

```text
REGISTER
   ↓
NORMALIZE
   ↓
VALIDATE
   ↓
COMPILE
   ↓
FREEZE
```

---

# 137. No runtime discovery

No:

```text
property access
→ reflection
→ discover Value Object mapping
```

---

# 138. Compiled mapping

```php
final readonly class CompiledValueObjectMapping
{
    public function __construct(
        public ValueObjectMetadata $metadata,
        public ValueObjectHydrationPlan $hydration,
        public ValueObjectDecompositionPlan $decomposition,
        public ValueObjectMappingFingerprint $fingerprint,
    ) {}
}
```

---

# 139. Compilation

Debe resolver previamente:

- component paths;
- column bindings;
- types;
- casts;
- enums;
- nested Value Objects;
- constructor bindings;
- nullability;
- equality;
- snapshot strategy.

---

# 140. Mapping graph

```text
Address
├── street:string
├── city:string
├── postalCode:PostalCode
│   └── value:string
└── country:CountryCode(enum)
```

se compila en un graph finito.

---

# 141. Graph validation

Debe detectar:

- cycles;
- missing mappings;
- duplicate columns;
- incompatible types;
- invalid constructors;
- ambiguous nullability;
- invalid overrides.

---

# 142. JSON strategy

Un Value Object podrá persistirse en una sola columna JSON.

Ejemplo:

```text
Address
   ↓
structured canonical representation
   ↓
JSON Type System
   ↓
JSON column
```

---

# 143. JSON mapping ≠ serialization shortcut

No hacer simplemente:

```php
json_encode($object);
```

---

# 144. Why

Eso podría persistir:

- private implementation details;
- unstable property names;
- unwanted fields;
- arbitrary nested objects.

---

# 145. Structured JSON mapping

Debe existir un schema/mapping explícito:

```text
Address.street
→ $.street

Address.city
→ $.city
```

---

# 146. JSON versioning

Cambiar la estructura JSON puede requerir data migration.

---

# 147. JSON query

Consultas:

```php
->where('address.city', 'Puebla')
```

sobre JSON podrán delegarse al:

```text
273_DATABASE_JSON_QUERY_SYSTEM
```

según mapping/capabilities.

---

# 148. Flattened vs JSON

## Flattened columns

Ventajas:

- indexes convencionales;
- constraints;
- simple querying;
- portable.

## JSON

Ventajas:

- schema flexibility;
- fewer physical columns;
- nested structures.

---

# 149. Selection should be explicit

VoltStack no decidirá automáticamente que un Value Object complejo debe convertirse a JSON.

---

# 150. Native composite types

PostgreSQL y otras plataformas pueden tener capacidades adicionales.

Podrán soportarse mediante:

```text
NATIVE_COMPOSITE
```

en extensiones futuras.

---

# 151. Portability

La semántica lógica deberá permanecer estable aunque cambie:

```text
FLATTENED_COLUMNS
→
JSON
```

mediante migración explícita.

---

# 152. Mapping migration

Cambiar la estrategia de almacenamiento es un cambio de schema + data.

---

# 153. Schema projection

Para flattened mapping:

```text
ValueObjectMetadata
       ↓
Schema Projection
       ↓
Column recommendations
```

---

# 154. Mapping ≠ Schema truth

Que metadata espere:

```text
address_city VARCHAR
```

no demuestra que la DB tenga esa columna.

---

# 155. Schema validation

Podrá comparar:

```text
expected persistent representation
vs
observed schema
```

---

# 156. Indexes

Un component podrá ser indexable:

```text
address.postalCode
```

---

# 157. Index metadata

El Value Object Mapping no crea automáticamente todos los índices.

Puede proyectar:

```text
indexable component binding
```

hacia Schema Builder.

---

# 158. Composite index

Ejemplo:

```text
(country, postal_code)
```

puede corresponder a componentes de un Address.

---

# 159. Unique constraint

Ejemplo:

```text
EmailAddress
```

puede participar en:

```text
UNIQUE(email)
```

pero uniqueness pertenece al Schema/Domain constraint correspondiente.

---

# 160. Money

Mapping recomendado:

```text
Money
├── amount → DECIMAL
└── currency → Currency enum/string
```

Nunca:

```text
amount → FLOAT
```

por default.

---

# 161. Money precision

La precisión pertenece al decimal Type Mapping.

---

# 162. Currency

Podrá ser:

```text
Currency enum
```

o:

```text
CurrencyCode Value Object
```

según dominio.

---

# 163. EmailAddress

Puede ser single-column Value Object:

```text
EmailAddress
→ string
```

---

# 164. Email normalization

Debe decidirse en el dominio/mapping explícito.

No asumir que todos los emails pueden convertirse arbitrariamente a lowercase bajo cualquier semántica.

---

# 165. PostalCode

Puede ser Value Object aunque la DB utilice VARCHAR.

Esto evita:

```text
integer postal code
```

y pérdida de ceros iniciales.

---

# 166. DateRange

```text
DateRange
├── start
└── end
```

---

# 167. DateRange invariant

El constructor puede imponer:

```text
start <= end
```

Por eso constructor-first hydration es valioso.

---

# 168. Invalid persisted DateRange

Si DB contiene:

```text
start > end
```

el constructor puede fallar.

VoltStack deberá reportar:

```text
ValueObjectInvariantViolationException
```

en vez de crear un objeto inválido silenciosamente.

---

# 169. GeoPoint

```text
GeoPoint
├── latitude
└── longitude
```

puede mapearse a:

```text
two columns
```

o una extensión geográfica futura.

---

# 170. Native geographic mapping

Será responsabilidad de:

```text
274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM
```

no del core Value Object mapper.

---

# 171. Encryption

Un Value Object puede tener componentes sensibles.

Ejemplo:

```text
PersonalIdentifier
```

---

# 172. Component encryption

El pipeline podrá ser:

```text
Value Object
→ decompose
→ component
→ cast
→ encryption integration
→ type conversion
```

---

# 173. Whole-object encryption

También puede existir:

```text
Value Object
→ canonical structured representation
→ encryption
→ single persistent value
```

pero será una estrategia distinta.

---

# 174. Queryability tradeoff

Whole-object encryption puede impedir:

```text
where('valueObject.component', ...)
```

VoltStack deberá exponer esa capability limitation.

---

# 175. ValueObjectCapability

Ejemplo:

```php
final readonly class ValueObjectCapabilities
{
    public function __construct(
        public bool $componentQueryable,
        public bool $componentIndexable,
        public bool $partialUpdate,
        public bool $portable,
    ) {}
}
```

---

# 176. Collections of Value Objects

Ejemplo:

```text
List<Address>
```

no debe confundirse con:

```text
OneToMany<AddressEntity>
```

---

# 177. Storage strategies

Una colección de Value Objects podría usar:

```text
JSON
ARRAY
custom type
```

pero no tabla relacional de entidades automáticamente.

---

# 178. Collection complexity

V1 deberá priorizar:

```text
JSON-backed immutable value collections
```

cuando sean necesarias.

---

# 179. Relational collection

Si cada elemento necesita:

- query independiente;
- ID;
- lifecycle;
- relationship;
- update individual;

probablemente debe modelarse como Entity.

---

# 180. Value Object decision rule

Preguntar:

```text
¿Tiene identidad independiente?
```

Si sí:

```text
Entity candidate
```

Si no:

```text
Value Object candidate
```

---

# 181. Repository

No:

```php
$repository = $em->repository(Address::class);
```

para embedded Value Objects.

---

# 182. EntityManager

No:

```php
$em->persist($money);
```

---

# 183. IdentityMap

No:

```text
IdentityMap[Money]
```

---

# 184. UnitOfWork

Sí puede conocer componentes de Value Objects como parte del estado de su Entity owner.

---

# 185. Event lifecycle

No emitir automáticamente:

```text
prePersist(ValueObject)
postPersist(ValueObject)
```

como si fuera Entity.

---

# 186. Owner lifecycle events

Los eventos del owner pueden inspeccionar sus Value Objects.

---

# 187. Specialized value events

Si alguna extensión los necesita, deberán ser eventos de value mapping explícitos, no entity lifecycle events.

---

# 188. Validation integration

El Validation System puede validar Value Objects antes de llegar a persistence.

Pero:

```text
Value Object constructor invariants
+
Validation rules
```

son capas complementarias.

---

# 189. Security

El sistema deberá evitar:

- arbitrary class instantiation;
- unsafe unserialization;
- constructor bypass indiscriminado;
- hidden I/O;
- relationship loading desde constructor;
- unbounded nested structures;
- leaking sensitive component values.

---

# 190. Constructor side effects

Los Value Objects persistibles deberían tener constructores:

```text
pure
+
deterministic
+
side-effect free
```

---

# 191. Constructor query anti-pattern

```php
public function __construct(string $country)
{
    $this->country = Country::find($country);
}
```

deberá considerarse incompatible con hydration segura.

---

# 192. Determinism

Para los mismos componentes canónicos:

```text
construct(components)
```

deberá producir un valor semánticamente equivalente.

---

# 193. Resource governance

Nested Value Objects requieren límites:

```php
final readonly class ValueObjectResourcePolicy
{
    public function __construct(
        public int $maxDepth,
        public int $maxComponents,
        public int $maxCollectionItems,
        public int $maxSerializedBytes,
    ) {}
}
```

---

# 194. Persistent runtime

Compartible:

```text
ValueObjectMappingRegistry
CompiledValueObjectMapping
Hydration Plans
Decomposition Plans
Stateless factories
```

---

# 195. Scoped

```text
temporary assembly state
recursion guards
resource counters
diagnostics
```

---

# 196. FrankenPHP

```text
Worker
├── Frozen VO Registry
├── Compiled VO Plans
│
├── Request A scoped state
└── Request B scoped state
```

---

# 197. RoadRunner

Mismo modelo.

---

# 198. OpenSwoole

Mutable assembly state deberá ser coroutine-safe/scoped.

---

# 199. No static object state

Nunca:

```php
static array $hydratedValueObjects;
```

para compartir objetos hidratados entre requests.

---

# 200. Mapping generation

Cambios en:

- components;
- PHP class;
- constructor;
- column bindings;
- types;
- nested mappings;
- nullability;
- strategy;
- equality;

deberán cambiar:

```text
ValueObjectMappingGeneration
```

---

# 201. Fingerprint

```text
ValueObjectTypeId
+
Mapping Strategy
+
Component Graph
+
Type Registry Generation
+
Cast Registry Generation
+
Enum Registry Generation
+
Construction Metadata
+
Nullability
+
Equality Policy
```

---

# 202. Diagnostics

API conceptual:

```php
Database::types()
    ->valueObjects()
    ->explain(Address::class);
```

---

# 203. Diagnostic example

```text
VALUE OBJECT MAPPING

Type:
    common.address

PHP Class:
    App\Domain\Address

Strategy:
    FLATTENED_COLUMNS

Mutability:
    IMMUTABLE

Construction:
    CONSTRUCTOR

Components:
    street
        string
        → billing_street

    city
        string
        → billing_city

    postalCode
        common.postal_code
        → billing_postal_code

    country
        common.country_code
        → billing_country

Nullability:
    NULLABLE_AS_WHOLE

Equality:
    CANONICAL_COMPONENTS

Queryable:
    yes

Partial Update:
    yes
```

---

# 204. Telemetry

Posibles métricas:

```text
database.value_object.hydration
database.value_object.decomposition
database.value_object.mapping_failure
database.value_object.constructor_failure
database.value_object.partial_state
database.value_object.snapshot
database.value_object.dirty
database.value_object.plan_cache_hit
database.value_object.plan_cache_miss
```

---

# 205. Telemetry cardinality

Dimensions:

```text
value_object_type
strategy
outcome
```

solo cuando estén controladas.

---

# 206. Sensitive values

Nunca incluir componentes raw en métricas por defecto.

---

# 207. Tracing

No crear span por cada Value Object pequeño.

Acumular costo en hydration/persistence spans.

---

# 208. Error hierarchy

```text
DatabaseValueObjectMappingException
├── ValueObjectMappingNotFoundException
├── ValueObjectRegistrationException
├── InvalidValueObjectMappingException
├── ValueObjectConstructionException
├── ValueObjectInvariantViolationException
├── ValueObjectComponentException
├── ValueObjectTypeMismatchException
├── ValueObjectColumnCollisionException
├── ValueObjectNullabilityException
├── ValueObjectPartialHydrationException
├── ValueObjectCycleException
├── ValueObjectDepthLimitException
├── ValueObjectEqualityException
├── ValueObjectSnapshotException
├── ValueObjectQueryException
├── ValueObjectPersistenceException
├── ValueObjectSchemaCompatibilityException
├── ValueObjectResourceLimitException
├── ValueObjectRuntimeStateException
└── ValueObjectMappingInvariantViolationException
```

---

# 209. Testing architecture

Debe existir:

```text
ValueObjectMappingConformanceSuite
```

---

# 210. Single-column tests

Probar:

```text
EmailAddress
→ string
→ EmailAddress
```

---

# 211. Multi-column tests

Probar:

```text
Money
→ amount + currency
→ Money
```

---

# 212. Nested tests

```text
Address
→ PostalCode
→ scalar
```

---

# 213. Constructor invariant test

DB inválida:

```text
DateRange.start > DateRange.end
```

debe fallar.

---

# 214. Nullability tests

Probar:

```text
all null
all present
partially null
missing
```

separadamente.

---

# 215. Partial hydration tests

No deberá construir un Value Object completo a partir de campos faltantes.

---

# 216. Equality tests

```text
Money("100.0", MXN)
Money("100.00", MXN)
```

deberán seguir exactamente la configured canonicalization.

---

# 217. Dirty tracking test

```text
hydrate
→ no modification
→ flush
```

debe producir:

```text
NO UPDATE
```

---

# 218. Replacement test

```text
old Address
→ equivalent new Address
```

no deberá producir false dirty.

---

# 219. Mutation test

Para mutable Value Object:

```text
internal mutation
```

deberá detectarse mediante snapshot strategy.

---

# 220. Query tests

```php
where('billingAddress.city', 'Puebla')
```

deberá resolver el binding correcto.

---

# 221. Composite equality tests

```php
where('price', $money)
```

deberá expandirse semánticamente de forma correcta.

---

# 222. NOT equality tests

Deberán verificar De Morgan/lógica booleana correcta.

---

# 223. Schema tests

Verificar:

- prefix;
- overrides;
- collisions;
- component types;
- nullable representation;
- indexes;
- schema diff.

---

# 224. JSON strategy tests

Verificar:

- deterministic structure;
- nested mapping;
- version changes;
- query capability;
- invalid JSON state.

---

# 225. Persistent worker tests

Metadata podrá compartirse.

Hydration/assembly state no.

---

# 226. Performance benchmarks

Medir:

```text
single-column VO hydration/sec
multi-column VO hydration/sec
nested VO hydration/sec
decomposition/sec
snapshot comparison/sec
constructor overhead
compiled plan cache hit rate
```

---

# 227. Performance principle

El hot path no deberá hacer:

- reflection repetida;
- metadata discovery;
- string path parsing;
- container lookup por componente;
- schema introspection.

---

# 228. Directory structure

```text
src/Quantum/Database/Type/ValueObject/
│
├── Contract/
│   ├── ValueObjectMappingRegistry.php
│   ├── ValueObjectMappingCompiler.php
│   ├── ValueObjectAssembler.php
│   ├── ValueObjectDecomposer.php
│   ├── ValueObjectComparator.php
│   └── ValueObjectSnapshotStrategy.php
│
├── Metadata/
│   ├── ValueObjectMetadata.php
│   ├── EmbeddedValueObjectMetadata.php
│   ├── ValueObjectComponentMetadata.php
│   ├── ValueObjectTypeId.php
│   ├── ValueObjectComponentPath.php
│   ├── ValueObjectComponentBinding.php
│   ├── ValueObjectOverrides.php
│   └── PHPValueObjectReference.php
│
├── Strategy/
│   ├── ValueObjectMappingStrategy.php
│   ├── ValueObjectConstructionStrategy.php
│   ├── ValueObjectUpdatePolicy.php
│   └── ValueObjectMutability.php
│
├── Nullability/
│   ├── ValueObjectNullability.php
│   ├── ValueObjectNullState.php
│   └── ValueObjectNullabilityResolver.php
│
├── Construction/
│   ├── ValueObjectConstructionMetadata.php
│   ├── ValueObjectConstructorPlan.php
│   ├── ConstructorValueObjectAssembler.php
│   ├── StaticFactoryValueObjectAssembler.php
│   └── RegisteredFactoryValueObjectAssembler.php
│
├── Registry/
│   ├── DefaultValueObjectMappingRegistry.php
│   ├── ValueObjectMappingRegistryBuilder.php
│   └── ValueObjectMappingGeneration.php
│
├── Compiler/
│   ├── DefaultValueObjectMappingCompiler.php
│   ├── CompiledValueObjectMapping.php
│   ├── ValueObjectMappingFingerprint.php
│   ├── ValueObjectHydrationPlan.php
│   └── ValueObjectDecompositionPlan.php
│
├── Hydration/
│   ├── ValueObjectHydrationNode.php
│   └── DefaultValueObjectAssembler.php
│
├── Persistence/
│   ├── DefaultValueObjectDecomposer.php
│   ├── ValueObjectComponentSet.php
│   └── ValueObjectComponentChangeSet.php
│
├── Equality/
│   ├── ValueObjectEqualityPolicy.php
│   └── CanonicalComponentComparator.php
│
├── Snapshot/
│   ├── ImmutableValueObjectSnapshotStrategy.php
│   └── MutableValueObjectSnapshotStrategy.php
│
├── Query/
│   ├── ValueObjectPropertyResolver.php
│   ├── ValueObjectPredicateExpander.php
│   └── ValueObjectOrderingPolicy.php
│
├── Schema/
│   ├── ValueObjectSchemaProjector.php
│   └── ValueObjectSchemaCompatibilityAnalyzer.php
│
├── Policy/
│   └── ValueObjectResourcePolicy.php
│
├── Diagnostics/
│   ├── ValueObjectMappingExplainer.php
│   └── ValueObjectMappingDiagnosticReport.php
│
└── Exception/
    ├── DatabaseValueObjectMappingException.php
    ├── ValueObjectMappingNotFoundException.php
    ├── InvalidValueObjectMappingException.php
    ├── ValueObjectConstructionException.php
    ├── ValueObjectInvariantViolationException.php
    ├── ValueObjectComponentException.php
    ├── ValueObjectTypeMismatchException.php
    ├── ValueObjectColumnCollisionException.php
    ├── ValueObjectNullabilityException.php
    ├── ValueObjectPartialHydrationException.php
    ├── ValueObjectCycleException.php
    ├── ValueObjectEqualityException.php
    ├── ValueObjectSnapshotException.php
    ├── ValueObjectQueryException.php
    ├── ValueObjectPersistenceException.php
    ├── ValueObjectResourceLimitException.php
    └── ValueObjectMappingInvariantViolationException.php
```

---

# 229. Dependency model

Permitido:

```text
Value Object Mapping
        ↓
Type System

Value Object Mapping
        ↓
Casting

Value Object Mapping
        ↓
Enum Mapping

Value Object Mapping
        ↓
Value Conversion contracts
```

---

# 230. ORM integration

```text
ORM
 ↓
Value Object Mapping
```

pero:

```text
Value Object Mapping
↛ EntityManager mutable state
```

---

# 231. Query integration

```text
Semantic Query Engine
        ↓
Value Object Mapping Metadata
        ↓
Physical Component Bindings
```

---

# 232. Schema integration

```text
Value Object Mapping
        ↓
Schema Projection
```

sin invertir la dependencia.

---

# 233. Architectural invariants

## DB-VO-001
Un Value Object no tendrá identidad ORM independiente.

## DB-VO-002
Un Value Object no será Entity.

## DB-VO-003
Un Value Object no entrará al IdentityMap.

## DB-VO-004
Un Value Object no tendrá Repository por defecto.

## DB-VO-005
Un Value Object no será persistido independientemente.

## DB-VO-006
Un Value Object pertenecerá al estado persistente de su owner.

## DB-VO-007
Value equality será distinta de PHP object identity.

## DB-VO-008
Value Object Mapping será distinto de Casting.

## DB-VO-009
Value Object Mapping será distinto de Serialization.

## DB-VO-010
Value Object Mapping será distinto de Relationship Mapping.

## DB-VO-011
Un Value Object podrá ocupar una o varias persistent fields.

## DB-VO-012
Single-column mappings podrán reutilizar Casting.

## DB-VO-013
Multi-column mappings no se implementarán como scalar casts improvisados.

## DB-VO-014
Mapping strategy será explícita.

## DB-VO-015
Flattened column bindings serán deterministas.

## DB-VO-016
Column collisions fallarán durante compilation.

## DB-VO-017
Overrides serán explícitos.

## DB-VO-018
Nested mappings serán finitos.

## DB-VO-019
Nested mapping cycles serán rechazados.

## DB-VO-020
Mapping depth tendrá resource limits.

## DB-VO-021
Relationships dentro de Value Objects no serán parte del core V1.

## DB-VO-022
Hydration preferirá constructors/factories válidos.

## DB-VO-023
Constructor bypass no será default.

## DB-VO-024
Constructor argument binding será explícito/compilado.

## DB-VO-025
Column order no definirá constructor semantics.

## DB-VO-026
Value Object Hydration no consultará IdentityMap.

## DB-VO-027
Global Value Object interning no será default.

## DB-VO-028
Composite nullability será explícita.

## DB-VO-029
ALL_NULL será distinto de PARTIAL_NULL.

## DB-VO-030
MISSING será distinto de NULL.

## DB-VO-031
Partial hydration no fabricará Value Objects completos.

## DB-VO-032
Projections serán preferidas para component subsets.

## DB-VO-033
Query property paths podrán atravesar Value Objects.

## DB-VO-034
Query Builder no resolverá physical columns por sí mismo.

## DB-VO-035
SQL Compiler no necesitará conocer Value Object classes.

## DB-VO-036
Composite predicates se expandirán semánticamente antes del SQL compiler.

## DB-VO-037
Composite inequality preservará lógica booleana correcta.

## DB-VO-038
Whole-Value ordering no será asumido automáticamente.

## DB-VO-039
Domain-specific operators requerirán query extensions.

## DB-VO-040
Persistence decomposition no producirá SQL.

## DB-VO-041
Dirty tracking no dependerá de object identity.

## DB-VO-042
Canonical component equality será el default recomendado.

## DB-VO-043
Immutable Value Objects serán preferidos.

## DB-VO-044
Mutable Value Objects requerirán snapshot strategy.

## DB-VO-045
Mutable snapshot no compartirá referencias mutables con current state.

## DB-VO-046
Component-level UPDATE no romperá semántica del Value Object.

## DB-VO-047
Value Object update policy será explícita.

## DB-VO-048
Delete del owner no generará independent Value Object delete.

## DB-VO-049
Orphan removal no aplicará a embedded Value Objects.

## DB-VO-050
Cascade persist no aplicará como relación ORM.

## DB-VO-051
Shared PHP immutable object no implicará shared persistence identity.

## DB-VO-052
ValueObjectMappingRegistry será immutable en runtime productivo.

## DB-VO-053
Hot path no descubrirá mappings mediante reflection.

## DB-VO-054
Compiled mappings serán immutable.

## DB-VO-055
Component graph será validado en bootstrap.

## DB-VO-056
JSON strategy utilizará mapping estructurado explícito.

## DB-VO-057
JSON strategy no persistirá arbitrary object internals.

## DB-VO-058
JSON schema evolution podrá requerir migration.

## DB-VO-059
Flattened y JSON serán estrategias diferentes.

## DB-VO-060
Native composite será capability opcional.

## DB-VO-061
Storage strategy changes serán schema+data migrations.

## DB-VO-062
Mapping metadata no afirmará physical schema truth.

## DB-VO-063
Indexes pertenecerán al Schema System.

## DB-VO-064
Money no utilizará float por default.

## DB-VO-065
Decimal precision pertenecerá al Type System.

## DB-VO-066
Constructor invariants serán respetados durante hydration.

## DB-VO-067
Invalid persisted domain values no se normalizarán silenciosamente.

## DB-VO-068
Geographic native behavior será extensión separada.

## DB-VO-069
Encryption será integración separada.

## DB-VO-070
Whole-object encryption declarará pérdida de queryability.

## DB-VO-071
Collections of Value Objects no serán OneToMany implícitamente.

## DB-VO-072
Value collection storage será explícito.

## DB-VO-073
Objects con identidad independiente deberán modelarse como Entities.

## DB-VO-074
EntityManager no persistirá Value Objects directamente.

## DB-VO-075
Entity lifecycle events no se duplicarán para Value Objects.

## DB-VO-076
Constructors persistibles deberán evitar hidden I/O.

## DB-VO-077
Value Object construction deberá ser deterministic.

## DB-VO-078
Resource governance aplicará a nested mappings.

## DB-VO-079
Compiled metadata podrá compartirse en persistent runtimes.

## DB-VO-080
Assembly mutable state será scoped.

## DB-VO-081
No habrá static hydrated Value Object caches entre requests.

## DB-VO-082
Mapping changes producirán nueva generation.

## DB-VO-083
Fingerprint incluirá dependent registry generations.

## DB-VO-084
Diagnostics no revelarán sensitive components por default.

## DB-VO-085
Telemetry tendrá cardinalidad controlada.

## DB-VO-086
No habrá tracing span por cada pequeño Value Object.

## DB-VO-087
Round-trip tests serán obligatorios.

## DB-VO-088
Nullability tests cubrirán ALL_NULL/PARTIAL/PRESENT/MISSING.

## DB-VO-089
Dirty tracking tendrá equivalence tests.

## DB-VO-090
Nested mappings tendrán conformance tests.

## DB-VO-091
Persistent runtime isolation será probado.

## DB-VO-092
No habrá schema introspection en hydration hot path.

## DB-VO-093
No habrá container lookup por component en hot path.

## DB-VO-094
No habrá SQL dentro de Value Object assemblers.

## DB-VO-095
No habrá transaction control dentro de Value Object mapping.

## DB-VO-096
No habrá connection routing dentro de Value Object mapping.

## DB-VO-097
No habrá lazy relationship loading dentro de Value Object mapping.

## DB-VO-098
No habrá authorization decisions dentro del mapper.

## DB-VO-099
No habrá API serialization decisions dentro del mapper.

## DB-VO-100
VoltStack preservará separación entre domain value y physical representation.

## DB-VO-101
ValueObjectTypeId será estable e independiente del FQCN cuando se configure.

## DB-VO-102
Component TypeReference será validado antes del runtime normal.

## DB-VO-103
Nested enum components reutilizarán Enum Mapping.

## DB-VO-104
Nested Value Objects reutilizarán el mismo mapping engine.

## DB-VO-105
Acyclic mapping graph será obligatorio.

## DB-VO-106
Canonical decomposition deberá ser deterministic.

## DB-VO-107
Hydration y decomposition deberán ser semánticamente inversas cuando el mapping sea reversible.

## DB-VO-108
Equivalent values no producirán false dirty.

## DB-VO-109
Physical partial updates no alterarán la igualdad conceptual.

## DB-VO-110
Schema projection será advisory hasta ser validada contra Schema Model.

## DB-VO-111
Component queryability dependerá de storage strategy/capabilities.

## DB-VO-112
Encrypted opaque Value Objects no fingirán ser component-queryable.

## DB-VO-113
Invalid constructor output será tratado como mapping/runtime failure.

## DB-VO-114
Unknown component type será error de metadata.

## DB-VO-115
Duplicate component path será error.

## DB-VO-116
Duplicate physical binding será error salvo estrategia que lo permita explícitamente.

## DB-VO-117
Owner Entity seguirá siendo la unidad de persistencia ORM.

## DB-VO-118
Value Objects no se convertirán en entities por optimización interna.

## DB-VO-119
Value Object cache no reemplazará IdentityMap.

## DB-VO-120
Domain invariants tendrán prioridad sobre hydration convenience.

---

# 234. Anti-patterns

## 234.1 Convertir Money en Entity

```text
money_id
→ money table
```

sin que `Money` tenga identidad real.

**Rechazado como default.**

---

## 234.2 Multi-column cast improvisado

```php
MoneyCast::set($money)
{
    // writes amount and currency columns directly
}
```

**Rechazado.**

El Cast no escribe columnas.

---

## 234.3 Constructor bypass por default

```text
allocate object
→ write private fields
→ ignore invariants
```

**Rechazado.**

---

## 234.4 Partial object fabrication

```text
city selected
street missing
zip missing

→ Address(null, city, null)
```

**Rechazado.**

---

## 234.5 Relationship hidden in Value Object

```php
final class Address
{
    public Country $country;
}
```

donde `Country` es una Entity lazy-loaded.

**Fuera del core V1.**

---

## 234.6 JSON serialize arbitrary object

```php
json_encode($address);
```

como mapping persistente implícito.

**Rechazado.**

---

## 234.7 Object identity dirty tracking

```php
$old !== $new
```

como única comparación.

**Rechazado.**

---

## 234.8 Mutable snapshot by reference

```php
$snapshot = $entity->address;
```

para un Address mutable.

**Rechazado.**

---

# 235. Ejemplo completo — Money

```php
final readonly class Money
{
    public function __construct(
        public string $amount,
        public Currency $currency,
    ) {
        if (bccomp($amount, '0', 2) < 0) {
            throw new InvalidArgumentException();
        }
    }
}
```

Entidad:

```php
final class Product
{
    #[Embedded(prefix: 'price_')]
    private Money $price;
}
```

Representación:

```text
Product.price.amount
    → price_amount DECIMAL(...)

Product.price.currency
    → price_currency VARCHAR(...)
```

---

# 236. Money hydration

```text
price_amount = "1250.00"
price_currency = "MXN"
        │
        ▼
Type Conversion
        │
        ▼
"1250.00" + Currency::MXN
        │
        ▼
Money constructor
        │
        ▼
Money("1250.00", MXN)
```

---

# 237. Money persistence

```text
Money("1500.00", MXN)
        │
        ▼
Decomposition
        │
        ├── amount → "1500.00"
        └── currency → Currency::MXN
                         │
                         ▼
                    Enum Mapping
                         │
                         ▼
                       "MXN"
```

---

# 238. Example — Address

```php
final readonly class Address
{
    public function __construct(
        public string $street,
        public string $city,
        public PostalCode $postalCode,
        public CountryCode $country,
    ) {}
}
```

```text
billing_street
billing_city
billing_postal_code
billing_country
        │
        ▼
ValueObjectHydrationPlan
        │
        ▼
Address(...)
```

---

# 239. Example — Query

```php
Customer::query()
    ->where('billingAddress.country', CountryCode::MX)
    ->where('billingAddress.city', 'Ciudad de México')
    ->get();
```

Semantic resolution:

```text
billingAddress.country
    → billing_country

billingAddress.city
    → billing_city
```

Después:

```text
Query Model
→ Optimizer
→ Planner
→ Compiler
```

---

# 240. Example — Composite equality

```php
Product::query()
    ->where('price', new Money('100.00', Currency::MXN))
    ->get();
```

Semantic AST:

```text
AND
├── price.amount = Decimal("100.00")
└── price.currency = Currency::MXN
```

No SQL todavía.

---

# 241. Example — Dirty tracking

Hydrated:

```text
price =
Money("100.00", MXN)

snapshot =
{
    amount: "100.00",
    currency: "MXN"
}
```

Application:

```php
$product->price = new Money('100.0', Currency::MXN);
```

Canonical decomposition:

```text
{
    amount: "100.00",
    currency: "MXN"
}
```

Resultado:

```text
NOT DIRTY
```

---

# 242. Example — Actual change

```php
$product->price = new Money('125.00', Currency::MXN);
```

Canonical:

```text
old:
{
    amount: "100.00",
    currency: "MXN"
}

new:
{
    amount: "125.00",
    currency: "MXN"
}
```

ChangeSet:

```text
price.amount:
    "100.00"
    →
    "125.00"
```

---

# 243. Master hydration formula

Para Value Object `V` compuesto por `n` componentes:

```text
V
=
Construct(
    Convert(C₁),
    Convert(C₂),
    ...
    Convert(Cₙ)
)
```

si y solo si:

```text
RequiredComponentsAvailable
∧
NullabilityValid
∧
ConstructionInvariantsSatisfied
```

---

# 244. Master persistence formula

```text
PersistentComponents(V)
=
Canonicalize(
    Decompose(V)
)
```

---

# 245. Equality formula

Default:

```text
Equivalent(A, B)
iff
CanonicalComponents(A)
=
CanonicalComponents(B)
```

---

# 246. Dirty formula

```text
Dirty(V)
=
CanonicalComponents(CurrentV)
≠
SnapshotComponents(V)
```

---

# 247. Null formula

Para `NULLABLE_AS_WHOLE`:

```text
AllComponentsNULL
→
ValueObject = null
```

pero:

```text
SomeComponentsNULL
∧
SomeComponentsPresent
→
validate component nullability
```

Nunca:

```text
Partial
→
null automatically
```

---

# 248. Architectural master model

```text
                       ENTITY
                         │
                         ▼
                 VALUE OBJECT PROPERTY
                         │
                         ▼
              VALUE OBJECT MAPPING
         ┌───────────────┼────────────────┐
         │               │                │
     Metadata        Construction      Equality
         │               │                │
         └───────────────┼────────────────┘
                         ▼
                  COMPONENT GRAPH
         ┌───────────────┼────────────────┐
         │               │                │
      Scalar           Enum          Nested VO
         │               │                │
         └───────────────┼────────────────┘
                         ▼
                 TYPE / CAST SYSTEM
                         │
                         ▼
               PERSISTENT COMPONENTS
                         │
                ┌────────┴────────┐
                │                 │
         Flattened Columns       JSON
                │                 │
                └────────┬────────┘
                         ▼
                    QUERY ENGINE
                         │
                         ▼
                       DATABASE
```

---

# 249. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Value Semantics
```

en lugar de Entity semantics para Value Objects.

Adoptará:

```text
Single-column
+
Multi-column
+
Nested
+
JSON
```

como modelos de mapping.

Adoptará:

```text
Constructor-first Hydration
```

para preservar invariantes.

Adoptará:

```text
Canonical Component Equality
```

como estrategia de comparación por defecto.

Adoptará:

```text
Immutable Value Objects
```

como recomendación principal.

Adoptará:

```text
Compiled Mapping Graphs
```

para evitar reflection y discovery en hot paths.

Adoptará:

```text
Property-path Query Resolution
```

para consultar componentes embebidos.

Y mantendrá:

```text
Value Object
≠
Entity
≠
Relationship
≠
DTO
≠
Scalar Cast
```

---

# 250. Regla maestra final

> **VoltStack deberá permitir que el dominio utilice objetos ricos sin obligarlo a pensar en columnas, pero el ORM deberá conservar una representación persistente explícita, determinista, tipada y consultable de esos valores.**

La transformación fundamental será:

```text
Domain Value
      ↕
Value Object Mapping
      ↕
Canonical Component Graph
      ↕
Type / Cast / Enum Systems
      ↕
Persistent Representation
```

sin introducir:

```text
fake entity identity
```

ni:

```text
hidden persistence lifecycle
```

para valores que conceptualmente no poseen identidad.

De esta manera una entidad podrá expresar:

```php
$order->total = new Money('1250.00', Currency::MXN);
$customer->email = new EmailAddress('user@example.com');
$customer->address = new Address(...);
```

mientras el Database System conserva:

- tipado;
- queryability;
- dirty tracking;
- schema awareness;
- portability;
- persistent-runtime safety;
- performance;
- separación de responsabilidades.

---

# 251. Siguiente documento

```text
161_DATABASE_JSON_TYPE_SYSTEM.md
```

El siguiente documento deberá especializar el Type System para datos JSON y definir, entre otros:

- logical JSON type;
- JSON scalar/document distinction;
- object vs array semantics;
- canonical JSON representation;
- JSON null vs SQL NULL;
- missing key vs JSON null;
- encoding/decoding;
- numeric precision;
- Unicode;
- deterministic canonicalization;
- JSON objects;
- arrays;
- nested values;
- JSON Value Objects;
- JSON enums;
- JSON casting;
- query parameter binding;
- MySQL JSON;
- MariaDB JSON;
- PostgreSQL JSON/JSONB;
- SQLite JSON capabilities;
- platform capability detection;
- JSON schema integration;
- JSON path representation;
- mutation;
- dirty tracking;
- partial updates;
- patch operations;
- containment;
- extraction;
- indexing;
- generated/indexed paths;
- query integration;
- portability;
- validation boundaries;
- resource limits;
- security;
- telemetry;
- diagnostics;
- persistent runtime;
- testing;
- performance;
- extension points;
- architectural invariants.

Regla central propuesta:

> **JSON en VoltStack será un tipo lógico estructurado y no un string decorado: el framework deberá preservar la diferencia entre SQL NULL, JSON null, clave ausente, objeto, array y scalar, mientras delega a cada plataforma únicamente la representación física y las capacidades de consulta disponibles.**