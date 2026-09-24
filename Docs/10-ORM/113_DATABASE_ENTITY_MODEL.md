# 113_DATABASE_ENTITY_MODEL.md

# VoltStack Quantum Database
## Database Entity Model

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 113 — Database Entity Model  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Model` define el modelo conceptual, estructural y semántico de una **Entity** dentro del ORM de VoltStack.

Este documento establece:

- qué es una entidad;
- qué no es una entidad;
- cómo se representa su identidad;
- cómo se distingue identidad de igualdad;
- cómo se representan identificadores simples y compuestos;
- cómo se manejan identificadores generados;
- cómo se representan entidades nuevas, persistentes y detached;
- cómo se modelan referencias a entidades;
- cómo se manejan proxies;
- cómo se preserva la identidad dentro de un ORM scope;
- cómo interactúan las entidades con Metadata, Identity Map y Unit of Work;
- qué información pertenece a la entidad y qué información pertenece al ORM.

El principio fundamental será:

> **Una entidad es un objeto de dominio cuya continuidad está determinada por identidad, no por igualdad de todos sus valores ni por su representación física en una tabla.**

---

# 2. Posición arquitectónica

```text
Application / Domain
        │
        ▼
      Entity
        │
        ▼
 Entity Metadata
        │
        ├───────────────┐
        ▼               ▼
   Identity Map      Unit of Work
        │               │
        └───────┬───────┘
                ▼
         Entity Manager
                │
                ▼
       Persistence Engine
                │
                ▼
          Query Engine
```

La entidad se encuentra en la frontera superior del ORM.

No deberá conocer las capas inferiores.

---

# 3. Separaciones fundamentales

VoltStack preservará:

```text
Entity
≠
Database Row

Entity
≠
Table

Entity
≠
Record

Entity
≠
DTO

Entity
≠
Value Object

Entity
≠
Model Metadata

Entity
≠
Entity Metadata

Entity
≠
Entity State

Entity
≠
UnitOfWork Entry

Entity
≠
IdentityMap Entry

Entity
≠
Query Result Row

Entity
≠
SQL

Entity Identity
≠
PHP Object Identity

Entity Identity
≠
Database Identifier Representation

Entity Identifier
≠
Primary Key Definition

Entity Identifier
≠
EntityKey

Entity Equality
≠
Value Equality

Persistent Identity
≠
Object Memory Address

Entity Reference
≠
Loaded Entity

Proxy
≠
Entity Metadata

Detached Entity
≠
Deleted Entity

New Entity
≠
Managed Entity

Null Identifier
≠
Universal Definition Of New Entity
```

---

# 4. Definición de Entity

Una entidad puede modelarse como:

```text
Entity
=
Domain Object
+
Entity Type
+
Identity Semantics
+
Domain State
+
Behavior
```

No necesariamente:

```text
Entity
=
Database Row
```

---

# 5. Ejemplo básico

```php
#[Entity]
final class User
{
    public function __construct(
        private UserId $id,
        private Email $email,
        private string $name,
    ) {}

    public function rename(string $name): void
    {
        $this->name = $name;
    }
}
```

La entidad contiene:

```text
identity
domain state
domain behavior
```

No necesita contener:

```text
PDO
Connection
SQL
QueryBuilder
UnitOfWork
IdentityMap
EntityManager
```

---

# 6. Entity ≠ Row

Una fila puede ser:

```text
users
────────────────────────────────────────
id | name | email | active | created_at
```

mientras la entidad puede ser:

```text
User
├── UserId
├── Name
├── Email
├── Status
└── behavior
```

Incluso pueden existir diferencias de representación.

Por ejemplo:

```text
DB
first_name
last_name
```

podría convertirse en:

```text
Entity
Name
```

---

# 7. Entity ≠ Active Record

La entidad conceptual no deberá requerir métodos como:

```php
save();
delete();
refresh();
```

VoltStack podrá ofrecerlos mediante su `Model API`, pero serán una capa de Developer Experience.

Por tanto:

```text
Domain Entity
        │
        ├── may be plain PHP object
        │
        └── may extend VoltStack Model
```

Ambos deberán utilizar el mismo ORM Core.

---

# 8. Entity Type

Toda entidad gestionada por el ORM tendrá un:

```text
EntityType
```

conceptual.

Ejemplo:

```php
final readonly class EntityType
{
    public function __construct(
        public string $className,
    ) {}
}
```

---

# 9. EntityType ≠ class string

Aunque inicialmente pueda derivarse de:

```php
User::class
```

el concepto interno será más fuerte.

Permitirá representar:

```text
canonical entity type
inheritance mapping
proxy normalization
aliases
compiled metadata identity
```

---

# 10. Canonical Entity Type

Un proxy:

```text
UserProxy
```

no deberá crear un EntityType diferente de:

```text
User
```

si representa la misma entidad ORM.

Por tanto:

```text
EntityType(UserProxy)
=
EntityType(User)
```

cuando el proxy representa `User`.

---

# 11. Entity Descriptor

Se propone un descriptor ligero:

```php
final readonly class EntityDescriptor
{
    public function __construct(
        public EntityType $type,
        public EntityIdentityDescriptor $identity,
    ) {}
}
```

Este descriptor describe el tipo de entidad y su modelo de identidad.

No contiene:

```text
live entity instance
UnitOfWork state
connection
query
transaction
```

---

# 12. Identidad de entidad

La identidad responde:

> ¿Qué hace que este objeto siga representando la misma entidad aunque cambien sus atributos?

Ejemplo:

```text
User#42
```

puede cambiar:

```text
name
email
status
address
```

y continuar siendo:

```text
User#42
```

---

# 13. Identidad ≠ estado

Formalmente:

```text
Identity(E_t0)
=
Identity(E_t1)
```

aunque:

```text
State(E_t0)
≠
State(E_t1)
```

---

# 14. Entity Identifier

El valor utilizado para representar la identidad persistente será:

```text
EntityIdentifier
```

---

# 15. Modelo conceptual

```text
EntityIdentifier
├── SimpleEntityIdentifier
├── CompositeEntityIdentifier
└── GeneratedEntityIdentifier
```

`GeneratedEntityIdentifier` describe principalmente estrategia/lifecycle; después de materializarse deberá poder convertirse en una identidad concreta.

---

# 16. Simple identifier

Ejemplos:

```text
integer
UUID
ULID
string
binary UUID
domain ID value object
```

Ejemplo:

```php
final readonly class UserId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 17. Entity Identifier ≠ DB representation

Un:

```text
UserId
```

podría persistirse como:

```text
PostgreSQL UUID
MySQL BINARY(16)
MariaDB BINARY(16)
SQLite TEXT
```

Por tanto:

```text
Domain Identifier
≠
Physical Database Identifier
```

---

# 18. Identifier Mapping

La traducción pertenece a:

```text
Entity Metadata
+
Type System
+
Mapping System
```

No al objeto `EntityIdentifier` por sí solo.

---

# 19. Entity Key

El ORM necesita una clave canónica para Identity Map.

Se propone:

```php
final readonly class EntityKey
{
    public function __construct(
        public EntityType $type,
        public NormalizedEntityIdentifier $identifier,
        public EntityIdentityNamespace $namespace,
    ) {}
}
```

---

# 20. Identity namespace

`EntityIdentityNamespace` permite incorporar el contexto necesario para evitar colisiones.

Por ejemplo:

```text
Database
Tenant
Shard
Entity Type
Identifier
```

cuando corresponda.

---

# 21. Fórmula de EntityKey

Conceptualmente:

```text
EntityKey
=
IdentityNamespace
+
EntityType
+
NormalizedIdentifier
```

---

# 22. Tenant example

```text
Tenant A / User / 42
```

no es necesariamente la misma identidad ORM que:

```text
Tenant B / User / 42
```

Por tanto:

```text
EntityKey(A, User, 42)
≠
EntityKey(B, User, 42)
```

---

# 23. Sharding

Lo mismo puede aplicar para:

```text
Shard A / Customer / 100
Shard B / Customer / 100
```

si los espacios de identidad no son globales.

---

# 24. Identity Namespace ≠ Connection

No se almacenará un objeto:

```text
Connection
```

dentro de `EntityKey`.

Se almacenará identidad lógica normalizada.

---

# 25. Normalized Entity Identifier

El Identity Map no deberá comparar identificadores de forma ingenua.

Ejemplo:

```text
42
"42"
```

no deberán asumirse iguales sin conocer el mapping.

---

# 26. Normalización

Pipeline:

```text
Domain Identifier
       ↓
Identifier Mapping
       ↓
Identifier Normalizer
       ↓
NormalizedEntityIdentifier
       ↓
EntityKey
```

---

# 27. Canonical representation

Ejemplo UUID:

```text
550E8400-E29B-41D4-A716-446655440000
```

y:

```text
550e8400-e29b-41d4-a716-446655440000
```

pueden normalizarse a la misma identidad cuando el tipo así lo establece.

---

# 28. Normalización ≠ coerción arbitraria

Nunca:

```text
anything vaguely similar
→ same identifier
```

La normalización será definida por el tipo/mapping.

---

# 29. Composite identifiers

VoltStack deberá soportar identificadores compuestos cuando el modelo existente lo requiera.

Ejemplo:

```text
OrderLine
├── order_id
└── line_number
```

---

# 30. CompositeEntityIdentifier

Conceptualmente:

```php
final readonly class CompositeEntityIdentifier
{
    public function __construct(
        public array $components,
    ) {}
}
```

pero internamente deberá usar componentes tipados, no un array ambiguo.

---

# 31. Modelo recomendado

```php
final readonly class EntityIdentifierComponent
{
    public function __construct(
        public string $name,
        public mixed $value,
        public DatabaseType $type,
    ) {}
}
```

y:

```php
final readonly class CompositeEntityIdentifier
{
    /**
     * @param list<EntityIdentifierComponent> $components
     */
    public function __construct(
        public array $components,
    ) {}
}
```

---

# 32. Orden compuesto

El orden será canónico.

```text
(order_id, line_number)
```

no deberá producir una clave distinta por recibir:

```text
line_number
order_id
```

en otro orden externo.

Metadata definirá el orden.

---

# 33. Composite identity equality

Formalmente:

```text
CompositeId(A) = CompositeId(B)
```

si:

```text
∀ component_i:
Normalize(A_i) = Normalize(B_i)
```

según la definición canónica de identidad.

---

# 34. Identificadores mutables

Default:

> **La identidad persistente de una entidad gestionada será inmutable.**

Cambiar:

```text
User#42
```

a:

```text
User#84
```

no será una actualización ordinaria.

---

# 35. Primary key mutation

La mutación de identificador será:

```text
forbidden by default
```

o requerirá una operación explícita especializada.

---

# 36. Motivo

Identity Map depende de:

```text
EntityKey → Object
```

Si el ID cambia arbitrariamente:

```text
User#42
→
User#84
```

pueden romperse:

```text
IdentityMap
relationships
foreign references
snapshots
UnitOfWork
cache
```

---

# 37. Generated identifiers

Una entidad nueva puede requerir que la base genere su identificador.

Ejemplo:

```text
AUTO_INCREMENT
IDENTITY
SEQUENCE
database-generated UUID
```

---

# 38. Identidad pre-persistencia

Esto crea una distinción importante:

```text
Persistent Identity
```

vs:

```text
Temporary Runtime Identity
```

---

# 39. Temporary Entity Identity

Antes de obtener ID persistente, el ORM podrá asignar:

```text
TemporaryEntityKey
```

interno.

Ejemplo conceptual:

```text
NEW:User@runtime:7f93...
```

---

# 40. Temporary identity ≠ persistent identity

Nunca:

```text
temporary runtime key
=
database identifier
```

---

# 41. Generated ID transition

```text
NEW Entity
     │
     ▼
TemporaryEntityKey
     │
     ▼
INSERT
     │
     ▼
Generated ID = 42
     │
     ▼
Persistent EntityKey(User, 42)
```

---

# 42. Atomic re-keying

La transición:

```text
TemporaryEntityKey
→
PersistentEntityKey
```

deberá actualizar de forma consistente:

```text
IdentityMap
UnitOfWork
pending relationships
snapshots
persistence graph
```

---

# 43. Client-generated IDs

VoltStack deberá favorecer cuando sea apropiado:

```text
UUID
ULID
domain-generated identifiers
```

porque permiten identidad estable antes de persistir.

Ejemplo:

```php
$user = new User(
    UserId::new(),
    $email,
);
```

---

# 44. No universal null-ID rule

Incorrecto:

```text
id === null
⇒
entity is always new
```

El estado ORM no deberá inferirse universalmente solo desde nullabilidad del ID.

---

# 45. Motivo

Pueden existir:

```text
assigned identifiers
natural keys
detached objects
legacy schemas
composite identifiers
generated identifiers
```

Por tanto:

```text
Entity State
```

pertenece al UnitOfWork/Entity State System.

---

# 46. Natural identifiers

Podrán existir IDs como:

```text
country_code
tax_identifier
email
external_reference
```

pero el ORM deberá distinguir:

```text
Persistent Identifier
```

de:

```text
Unique Business Attribute
```

---

# 47. Unique ≠ identity

Un campo `email` puede ser UNIQUE sin ser el identificador ORM.

---

# 48. Surrogate IDs

Se recomienda para nuevas aplicaciones:

```text
stable surrogate ID
```

cuando facilite:

```text
relationships
migration
sharding
domain evolution
```

sin imponerlo como requisito universal.

---

# 49. Entity equality

Existen al menos tres conceptos:

```text
Object Identity
Entity Identity Equality
Value/State Equality
```

---

# 50. PHP object identity

```php
$a === $b;
```

representa identidad de instancia PHP.

---

# 51. Entity identity equality

Conceptualmente:

```text
same EntityType
+
same EntityIdentifier
+
same IdentityNamespace
```

---

# 52. State equality

Dos entidades podrían tener:

```text
same field values
```

pero diferente identidad.

Ejemplo:

```text
User#10
name = Ana

User#20
name = Ana
```

No son la misma entidad.

---

# 53. Entity equality formula

Para entidades persistentes:

```text
SameEntity(A, B)
=
SameIdentityNamespace(A, B)
∧
SameCanonicalEntityType(A, B)
∧
SameNormalizedIdentifier(A, B)
```

---

# 54. New entity equality

Dos entidades nuevas sin persistent ID no deberán considerarse iguales solo porque sus valores coinciden.

---

# 55. Temporary object identity

Mientras no exista identidad persistente:

```text
RuntimeIdentity
```

puede utilizarse internamente para seguimiento.

---

# 56. PHP equals methods

VoltStack no impondrá que toda entidad implemente:

```php
equals()
```

porque la semántica de igualdad pertenece al dominio.

El ORM utilizará su propio `EntityKey`.

---

# 57. hashCode-style semantics

No se dependerá de un hash mutable construido con todos los campos de la entidad.

---

# 58. Entity instance

Se distinguirá:

```text
EntityType
```

de:

```text
EntityInstance
```

Una instancia es un objeto concreto asociado opcionalmente a un contexto ORM.

---

# 59. Managed instance

Una entidad managed cumple:

```text
EntityInstance
+
EntityKey
+
IdentityMap registration
+
UnitOfWork registration
```

---

# 60. Entity state lifecycle

Aunque el documento 121 definirá formalmente el state machine, el modelo base reconoce:

```text
NEW
MANAGED
REMOVED
DETACHED
```

y estados internos derivados.

---

# 61. NEW

Una entidad `NEW`:

- existe en memoria;
- todavía no está sincronizada como entidad persistente gestionada;
- puede tener o no ID definitivo;
- puede ser registrada mediante `persist()`.

---

# 62. MANAGED

Una entidad `MANAGED`:

- pertenece al EntityManager actual;
- está registrada en UnitOfWork;
- cuando posee identidad persistente está registrada en IdentityMap;
- puede participar en dirty checking.

---

# 63. REMOVED

`REMOVED` significa:

```text
scheduled for removal
```

No necesariamente:

```text
already deleted from database
```

---

# 64. DETACHED

Una entidad `DETACHED`:

- continúa existiendo como objeto PHP;
- puede conservar persistent identity;
- ya no pertenece al ORM scope actual.

---

# 65. Detached ≠ deleted

Ejemplo:

```php
$user = $entityManager->find(User::class, 42);

$entityManager->clear();
```

`$user` sigue existiendo.

Su row puede seguir existiendo.

Pero:

```text
$user = DETACHED
```

---

# 66. Detached behavior

Una entidad detached no deberá:

- dirty-check automáticamente;
- persistirse automáticamente;
- lazy-load mágicamente;
- reutilizar recursos de un EntityManager cerrado.

---

# 67. Entity scope

La pertenencia a un ORM context será externa.

No se recomienda:

```php
private EntityManager $manager;
```

dentro de toda entidad.

---

# 68. EntityContextHandle

Si alguna funcionalidad avanzada requiere contexto, deberá utilizar mecanismos indirectos/controlados.

Nunca una dependencia permanente hacia Connection/EntityManager dentro del dominio.

---

# 69. Entity references

El ORM deberá poder representar una referencia a una entidad sin cargarla completamente.

Ejemplo:

```text
Order.customer
→
Customer#42
```

---

# 70. EntityReference

Se propone:

```php
final readonly class EntityReference
{
    public function __construct(
        public EntityType $type,
        public EntityIdentifier $identifier,
    ) {}
}
```

con namespace contextual resuelto de forma segura.

---

# 71. EntityReference ≠ Entity

Una referencia contiene:

```text
identity
```

pero no necesariamente:

```text
full state
```

---

# 72. Uso

Permite expresar:

```text
Order
  customer_id = 42
```

sin requerir siempre:

```text
SELECT * FROM customers WHERE id = 42
```

---

# 73. Reference resolution

```text
EntityReference
       ↓
EntityManager
       ↓
IdentityMap
       ├── HIT → entity
       └── MISS
             ↓
         load strategy
```

---

# 74. References and foreign keys

`EntityReference` pertenece al ORM.

`ForeignKey` pertenece al Schema/Database integrity model.

No son equivalentes.

---

# 75. getReference()

Podrá existir:

```php
$customer = $entityManager->getReference(
    Customer::class,
    42,
);
```

sin ejecutar inmediatamente una query.

---

# 76. Reference strategies

La implementación podrá utilizar:

```text
Identity Reference
Proxy
Lazy Ghost
Explicit EntityReference
```

según el tipo de API.

---

# 77. Proxy

Un proxy ORM es una representación runtime capaz de posponer la carga de ciertos datos.

---

# 78. Proxy ≠ new entity type

Si:

```text
CustomerProxy
```

representa `Customer#42`, su canonical type continúa siendo:

```text
Customer
```

---

# 79. Proxy identity

```text
EntityKey(CustomerProxy#42)
=
EntityKey(Customer#42)
```

cuando ambos representan la misma entidad y namespace.

---

# 80. Proxy constraints

Un proxy deberá preservar:

- entity identity;
- declared type compatibility;
- lifecycle rules;
- tenant/database namespace;
- detached behavior;
- no hidden cross-request context.

---

# 81. Proxy initialization

Conceptualmente:

```text
UNINITIALIZED
      ↓
initialize
      ↓
INITIALIZING
      ↓
INITIALIZED
```

con failure state explícito si la carga falla.

---

# 82. Proxy initialization ≠ entity state

`UNINITIALIZED` describe estado del proxy.

No sustituye:

```text
NEW
MANAGED
DETACHED
```

del ORM.

---

# 83. Proxy failure

Si la entidad referenciada no existe:

```text
EntityReference#42
        ↓
initialize
        ↓
NOT FOUND
```

deberá producir un resultado/error explícito según contrato.

---

# 84. Lazy ghost

Para PHP moderno, VoltStack podrá explorar estrategias de lazy objects/ghosts compatibles con el runtime soportado.

La arquitectura no deberá depender de una única técnica de proxy.

---

# 85. Proxy-free mode

El ORM deberá poder funcionar sin proxies cuando el desarrollador prefiera:

```text
explicit loading
```

---

# 86. Domain purity

Una entidad de dominio idealmente podrá escribirse como:

```php
final class Order
{
    public function __construct(
        private OrderId $id,
        private Customer $customer,
    ) {}

    public function addItem(Product $product, int $quantity): void
    {
        // domain behavior
    }
}
```

sin importar:

```text
QueryBuilder
SQL Compiler
Connection
Driver
```

---

# 87. Persistence ignorance

VoltStack deberá soportar un grado alto de:

```text
Persistence Ignorance
```

aunque el uso de PHP Attributes introduzca metadata declarativa opcional.

---

# 88. Attributes and domain purity

Un attribute:

```php
#[Entity]
```

es metadata declarativa.

No convierte automáticamente a la entidad en dependiente del motor de ejecución.

---

# 89. Attribute-free mapping

Para dominios que requieran máxima independencia deberá existir mapping externo/programático.

Ejemplo conceptual:

```php
$mapping->entity(User::class)
    ->table('users')
    ->id('id', type: UserIdType::class)
    ->field('email', 'email');
```

---

# 90. Constructor semantics

El ORM deberá distinguir:

```text
Domain Construction
```

de:

```text
Persistence Reconstitution
```

---

# 91. New domain object

```php
$user = User::register(
    UserId::new(),
    $email,
);
```

puede ejecutar invariantes de creación.

---

# 92. Reconstitution

Al cargar una entidad histórica desde DB, no siempre será correcto ejecutar:

```text
new registration workflow
```

de nuevo.

---

# 93. Entity Instantiator

Por tanto, Hydration utilizará:

```text
EntityInstantiator
```

con estrategia declarada por metadata.

---

# 94. Instantiation strategies

Posibles:

```text
CONSTRUCTOR
FACTORY
REFLECTION_RECONSTITUTION
COMPILED_RECONSTITUTOR
CUSTOM
```

---

# 95. No constructor bypass by accident

Si se omite el constructor, será por una estrategia explícita de reconstitución.

No por accidente de implementación.

---

# 96. Domain invariants

VoltStack distinguirá:

```text
Creation Invariants
```

de:

```text
Persistent State Invariants
```

El mapping deberá ser capaz de validar que la reconstitución produce un objeto estructuralmente válido.

---

# 97. Immutable entities

El Entity Model deberá permitir entidades inmutables cuando el mapping y UnitOfWork lo soporten.

Ejemplo:

```php
final readonly class Account
{
    public function __construct(
        public AccountId $id,
        public AccountStatus $status,
    ) {}
}
```

---

# 98. Immutable update semantics

Una operación:

```php
$account = $account->suspend();
```

puede producir otra instancia.

Esto crea un problema de identidad:

```text
old object
new object
same persistent entity identity
```

---

# 99. Identity Map replacement

Si se soporta este modelo, deberá existir una política explícita de:

```text
ManagedInstanceReplacement
```

Nunca dos instancias managed simultáneas para el mismo `EntityKey`.

---

# 100. Default mutable model

Para la primera implementación se recomienda:

```text
mutable managed entities
```

como modelo principal.

Immutable managed entities podrán ser una capacidad avanzada.

---

# 101. Identity collision

Si el ORM intenta registrar:

```text
Object A → User#42
Object B → User#42
```

en el mismo scope:

```text
A !== B
```

se produce:

```text
IdentityCollision
```

---

# 102. Identity collision rule

Nunca deberá ocurrir silenciosamente:

```text
last registered wins
```

---

# 103. Identity collision resolution

Posibles políticas explícitas:

```text
REJECT
MERGE_EXPLICITLY
REUSE_MANAGED_INSTANCE
```

pero `REJECT` será la base más segura para registro directo.

---

# 104. Identity Map lookup

Formalmente:

```text
IdentityMap[EntityKey] → EntityInstance
```

con cardinalidad:

```text
0..1
```

por key dentro del scope.

---

# 105. Entity registration

Registrar una entidad requiere:

```text
canonical type
normalized identity
identity namespace
entity instance
ORM scope
```

---

# 106. Entity identity completeness

Un identificador compuesto parcialmente conocido no será una persistent identity completa.

Ejemplo:

```text
(order_id = 42, line = UNKNOWN)
```

no puede registrarse como `OrderLine` persistente definitivo.

---

# 107. Identifier completeness

Se propone:

```text
COMPLETE
PARTIAL
UNASSIGNED
UNKNOWN
```

para describir conocimiento de identificador cuando sea necesario.

---

# 108. UNKNOWN ≠ UNASSIGNED

`UNASSIGNED`:

```text
se sabe que todavía no existe ID
```

`UNKNOWN`:

```text
no se conoce si existe o cuál es
```

Son diferentes.

---

# 109. Null identifier component

En composite IDs:

```text
NULL
```

puede ser un valor permitido solo si el mapping de identidad lo permite explícitamente.

No será equivalente a:

```text
UNASSIGNED
```

---

# 110. Entity snapshot

Una snapshot pertenece al UnitOfWork.

No a la Entity.

```text
Entity
      │
      ▼
UnitOfWork Entry
      ├── EntityState
      └── EntitySnapshot
```

---

# 111. Entity runtime metadata

Tampoco se recomienda introducir:

```php
private bool $__dirty;
private bool $__managed;
private array $__original;
```

en entidades de dominio.

---

# 112. Model API exception

`VoltStack\Model` podrá utilizar infraestructura interna para ergonomía.

Aun así, la fuente autoritativa del estado de persistencia continuará en UnitOfWork.

---

# 113. Domain state

El estado de dominio puede contener:

```text
fields
value objects
collections
relationships
domain flags
```

---

# 114. Persistence state

El estado ORM puede contener:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

No deben mezclarse conceptualmente.

---

# 115. Dirty ≠ domain status

Una entidad puede tener:

```text
status = ACTIVE
```

y al mismo tiempo estar:

```text
DIRTY
```

Son dimensiones distintas.

---

# 116. Entity collections

Una colección de relaciones:

```php
private Collection $orders;
```

puede ser:

```text
initialized
partially initialized
uninitialized
```

sin cambiar la identidad de la entidad.

---

# 117. Collection state ≠ Entity state

```text
LazyCollection::UNINITIALIZED
```

no significa:

```text
Entity::DETACHED
```

---

# 118. Entity references in collections

Una colección podrá contener referencias/identidades antes de materializar entidades completas, según estrategia de loading.

---

# 119. Entity graph

Un grafo ORM puede representarse como:

```text
Entity Nodes
+
Relationship Edges
```

---

# 120. Entity graph ≠ persistence graph

El Persistence Planner puede derivar:

```text
Persistence Dependency Graph
```

pero no son lo mismo.

---

# 121. Entity graph cycles

El dominio puede contener:

```text
User → Team → Owner(User)
```

sin que ello implique recursión infinita de hydration.

---

# 122. Graph traversal

Hydration, serialization y persistence deberán utilizar:

```text
visited identity sets
```

o mecanismos equivalentes.

---

# 123. Entity serialization

Serializar una entidad para HTTP/JSON no será responsabilidad fundamental del Entity Model.

---

# 124. Serialization ≠ persistence mapping

```text
API JSON representation
≠
Database mapping
```

---

# 125. Sensitive fields

El hecho de que un campo esté mapeado:

```text
password_hash
```

no significa que deba aparecer en serialization.

---

# 126. Entity clone

Clonar una entidad persistente presenta ambigüedad.

```php
$copy = clone $user;
```

Si conserva ID:

```text
User#42
```

entonces dos objetos representan la misma identidad.

---

# 127. Clone policy

El ORM no deberá registrar automáticamente clones con persistent identity duplicada.

---

# 128. Domain duplication

Para duplicar una entidad deberá preferirse una operación de dominio explícita:

```php
$copy = $user->duplicate(
    UserId::new(),
);
```

---

# 129. Entity merging

`merge()` estilo ORM tradicional puede introducir semánticas complejas.

No será requisito de la primera versión.

Preferencia:

```text
load managed entity
+
apply explicit state changes
```

---

# 130. Detached merge

Si se implementa posteriormente deberá ser una operación explícita, no un comportamiento automático de `persist()`.

---

# 131. persist(detached)

`persist()` sobre una entidad detached con persistent ID no deberá asumirse universalmente como UPDATE.

Deberá seguir una política definida.

---

# 132. Persist ≠ upsert

Regla:

```text
persist()
≠
INSERT OR UPDATE based on guess
```

---

# 133. Upsert

UPSERT será una operación Query/Persistence explícita.

No una inferencia basada únicamente en el ID.

---

# 134. Entity existence

Tener:

```text
UserId(42)
```

no demuestra que exista una fila correspondiente.

---

# 135. Reference existence

Igualmente:

```text
EntityReference(User, 42)
```

no afirma que la entidad exista.

Solo afirma la identidad referenciada.

---

# 136. Existence state

Cuando sea necesario:

```text
KNOWN_EXISTS
KNOWN_ABSENT
UNKNOWN
```

deberá derivarse de evidencia explícita.

---

# 137. Entity version

Una entidad podrá tener un campo de concurrency version:

```php
#[Version]
private int $version;
```

---

# 138. Version ≠ identity

```text
User#42 version 7
```

y:

```text
User#42 version 8
```

continúan representando la misma identidad.

---

# 139. Entity temporal versioning

Si en el futuro se implementan temporal entities/history:

```text
Entity Identity
```

y:

```text
Historical Version Identity
```

deberán modelarse por separado.

---

# 140. Soft delete

Una entidad soft-deleted puede conservar:

```text
persistent identity
```

aunque su política de query la oculte.

---

# 141. Soft deleted ≠ absent

Importante:

```text
Filtered From Default Query
≠
Does Not Exist
```

---

# 142. Tenant identity

Si Multitenancy utiliza DB por tenant:

```text
DatabaseContext
```

puede formar parte implícita del namespace de identidad.

Si utiliza shared table:

```text
tenant identity
```

puede requerirse explícitamente.

---

# 143. Tenant ID ≠ Entity ID

Aunque exista:

```text
tenant_id
```

no deberá fusionarse conceptualmente con `EntityIdentifier` salvo que realmente forme parte del primary/domain identity.

---

# 144. EntityKey scope

La construcción del `EntityKey` deberá considerar el namespace efectivo sin contaminar la entidad con infraestructura multitenant.

---

# 145. Cross-tenant references

Por default deberán rechazarse salvo que exista una relación explícitamente diseñada para ello.

---

# 146. Cross-database references

Una `EntityReference` a otra base deberá ser explícita y capability/policy-aware.

No deberá fingirse una FK que la base no puede garantizar.

---

# 147. Entity identity security

Los identificadores ORM no equivalen a autorización.

Ejemplo:

```text
User can reference Invoice#42
```

no significa:

```text
User is authorized to read Invoice#42
```

---

# 148. ID ≠ permission

Regla crítica:

```text
EntityIdentifier
≠
AuthorizationGrant
```

---

# 149. Entity loading security

Authorization deberá integrarse en capas apropiadas.

El Entity Model no incorporará políticas de autorización como identidad.

---

# 150. Mass-assignment security

Igualmente:

```text
Entity Fields
≠
Mass Assignable Fields
```

---

# 151. Sensitive identity

Algunos identificadores pueden ser sensibles.

Telemetry deberá permitir redacción/hash cuando sea necesario.

---

# 152. EntityKey logging

No se asumirá que todo `EntityKey` puede escribirse completo en logs.

---

# 153. Stable identity

Un identificador persistente debería idealmente ser:

```text
stable
unique within identity namespace
canonicalizable
serializable
comparable
```

---

# 154. Deterministic normalization

La función:

```text
NormalizeIdentifier(x)
```

deberá ser determinista.

Formalmente:

```text
Normalize(x)
=
Normalize(x)
```

para el mismo mapping/contexto.

---

# 155. Idempotent normalization

Idealmente:

```text
Normalize(Normalize(x))
=
Normalize(x)
```

---

# 156. Equality preservation

Si:

```text
IdentifierEquivalent(a, b)
```

entonces:

```text
Normalize(a)
=
Normalize(b)
```

según las reglas del tipo.

---

# 157. No lossy identity normalization

Una normalización que pueda colisionar IDs distintos será inválida.

---

# 158. Fingerprint de EntityType

Metadata compilada podrá utilizar:

```text
EntityTypeFingerprint
```

para caches internos.

Pero:

```text
EntityTypeFingerprint
≠
EntityIdentifier
```

---

# 159. Runtime object IDs

Funciones como:

```php
spl_object_id($entity)
```

pueden utilizarse internamente para tracking de instancia.

Pero:

```text
spl_object_id
≠
persistent identity
```

---

# 160. UnitOfWork indexing

El UnitOfWork podrá mantener dos índices:

```text
ObjectIdentityIndex
EntityIdentityIndex
```

conceptualmente:

```text
spl_object_id → UnitOfWorkEntry
EntityKey     → ManagedEntity
```

---

# 161. Razón

Esto permite responder dos preguntas distintas:

```text
What ORM state belongs to this PHP object?
```

y:

```text
Which PHP object represents User#42?
```

---

# 162. Entity metadata lookup

Metadata deberá resolverse por:

```text
CanonicalEntityType
```

no por nombre de proxy arbitrario.

---

# 163. Inheritance identity

Con herencia ORM:

```text
Person#42
Employee#42
```

la política de identidad deberá evitar objetos duplicados para la misma row/entidad cuando formen parte de una misma jerarquía persistente.

---

# 164. Root entity type

Podrá existir:

```text
IdentityRootType
```

para jerarquías de herencia.

---

# 165. Ejemplo

```text
Person
 ├── Employee
 └── Customer
```

Si utilizan una única identidad compartida:

```text
IdentityRoot = Person
```

---

# 166. EntityKey con herencia

Dependiendo de mapping:

```text
EntityKey
=
IdentityRootType
+
Identifier
+
Namespace
```

mientras el runtime type puede ser `Employee`.

---

# 167. Type mismatch

Si IdentityMap contiene:

```text
Person#42 → Employee
```

y una consulta intenta hidratar:

```text
Customer#42
```

deberá producirse inconsistencia de discriminador/mapping, no una segunda entidad.

---

# 168. Entity discriminators

Discriminator metadata pertenece al Mapping System.

No al EntityIdentifier.

---

# 169. Entity reference type safety

Una referencia:

```text
EntityReference<Customer>
```

no deberá resolverse silenciosamente como `Product`.

---

# 170. Typed references

Idealmente la API podrá ofrecer:

```php
/** @var EntityReference<Customer> */
$customer;
```

para análisis estático.

---

# 171. Generic metadata contracts

PHPDoc/generics de herramientas estáticas podrán utilizarse:

```php
/**
 * @template TEntity of object
 */
interface EntityRepository
{
    /** @return TEntity|null */
    public function find(mixed $id): ?object;
}
```

---

# 172. IDE friendliness

El Entity Model deberá favorecer:

```text
PHPStan
Psalm
IDE inference
reflection tooling
code generation
```

sin hacer depender el runtime de una herramienta concreta.

---

# 173. Entity requirements

Una clase será una entidad válida si:

```text
MappedAsEntity
∧
ValidIdentityMapping
∧
ValidFieldMapping
∧
ConstructableOrReconstitutable
∧
MetadataValid
```

---

# 174. No base class requirement

El ORM Core no exigirá:

```php
extends Model
```

para reconocer una entidad.

---

# 175. POPO support

Esto será válido:

```php
#[Entity]
final class Customer
{
}
```

sin heredar infraestructura.

---

# 176. Model support

También:

```php
final class Customer extends Model
{
}
```

podrá ser válido.

Ambos convergen al mismo:

```text
EntityMetadata
EntityManager
UnitOfWork
PersistenceEngine
```

---

# 177. Entity marker

El mapping podrá declarar entidad mediante:

```text
#[Entity]
programmatic metadata
compiled mapping
```

No se inferirá solo porque una clase tenga propiedad `$id`.

---

# 178. No magic entity detection

Incorrecto:

```text
class has id property
→ automatically ORM entity
```

---

# 179. Entity ID access

El ORM deberá soportar identificadores:

```text
private
protected
readonly
value-object based
```

mediante metadata/access strategy.

---

# 180. No public-ID requirement

No se exigirá:

```php
public $id;
```

---

# 181. Property access strategy

Metadata podrá compilar:

```text
IdentifierReader
IdentifierWriter
FieldReader
FieldWriter
```

---

# 182. Generated identifier writer

Si DB genera ID, la entidad deberá declarar una estrategia válida para asignarlo.

---

# 183. Readonly generated ID problem

Una propiedad PHP `readonly` no inicializada puede requerir estrategia específica.

El mapping deberá validarla antes de runtime.

---

# 184. Invalid generated identity

Si una entidad declara:

```text
DB_GENERATED ID
```

pero el ORM no puede asignarlo después del INSERT:

```text
mapping invalid
```

deberá fallar durante metadata validation cuando sea detectable.

---

# 185. Assigned identity

Con:

```text
ASSIGNED
```

el identificador debe existir antes de persistence planning.

---

# 186. Generated identity strategy

Se distinguirá:

```text
ASSIGNED
CLIENT_GENERATED
DATABASE_GENERATED
SEQUENCE
IDENTITY
CUSTOM
```

a nivel lógico.

---

# 187. Logical strategy ≠ physical mechanism

```text
DATABASE_GENERATED
```

no equivale universalmente a:

```text
AUTO_INCREMENT
```

---

# 188. Entity identity lifecycle

```text
┌─────────────────────┐
│ Entity Construction │
└──────────┬──────────┘
           │
           ▼
   ┌───────────────┐
   │ ID available? │
   └──────┬────────┘
      yes │       no
          │        │
          ▼        ▼
 Persistent/   Temporary
 Assigned ID   Runtime Identity
          │        │
          │        ▼
          │    Persistence
          │        │
          │        ▼
          │   Generated ID
          │        │
          └────┬───┘
               ▼
         Canonical EntityKey
               │
               ▼
           IdentityMap
```

---

# 189. Entity model serialization

Los value objects internos:

```text
EntityType
EntityIdentifier
EntityKey
EntityReference
```

deberán tener representaciones deterministas cuando se serialicen para:

```text
cache keys
telemetry
debugging
queue references
```

---

# 190. Entity reference serialization

Para queues se deberá preferir:

```text
EntityReference
```

a serializar una managed entity completa.

---

# 191. Queue boundary

Incorrecto conceptualmente:

```text
serialize entire managed User object
+
UnitOfWork internals
```

Correcto:

```text
User reference
+
required immutable payload
```

---

# 192. Entity across requests

Una entidad no deberá conservar su estado `MANAGED` al cruzar:

```text
HTTP request
job boundary
worker boundary
process boundary
```

---

# 193. Persistent identity survives; management does not

```text
Persistent Entity Identity
```

puede sobrevivir al boundary.

```text
Managed State
```

no.

---

# 194. Cross-request formula

```text
EntityKey
→ serializable/reconstructable

EntityManager membership
→ scope-local only
```

---

# 195. FrankenPHP safety

En FrankenPHP:

```text
Worker
│
├── Request A
│   ├── EntityManager A
│   ├── IdentityMap A
│   └── User#42 → Object A
│
├── RESET
│
└── Request B
    ├── EntityManager B
    ├── IdentityMap B
    └── User#42 → Object B
```

`Object A` y `Object B` pueden representar la misma persistent identity en momentos distintos, pero nunca serán la misma managed instance compartida entre requests.

---

# 196. RoadRunner/OpenSwoole

La misma regla será obligatoria para futuros runtimes persistentes.

---

# 197. Entity memory retention

El ORM deberá evitar que caches compartidos mantengan referencias fuertes a entidades managed después del scope.

---

# 198. Shared cache contents

Un cache process-shared podrá almacenar:

```text
EntityMetadata
HydrationPlan
PropertyAccessor
MappingDescriptor
```

pero no:

```text
live User object
```

---

# 199. Weak references

Weak references podrían utilizarse para tooling interno, pero no reemplazarán el lifecycle explícito del IdentityMap.

---

# 200. Entity model and telemetry

Telemetry podrá observar:

```text
entity type
entity state transition
identity-map hit
hydration
detach
identity collision
```

con políticas de redacción.

---

# 201. Entity debugging

Developer tooling podrá mostrar:

```text
Entity: User
ID: 42
State: MANAGED
Loaded fields: ...
Dirty fields: ...
EntityManager scope: ...
```

sin introducir esta metadata dentro de la entidad.

---

# 202. Entity introspection API

Podrá existir:

```php
$orm->inspect($user);
```

que devuelva:

```text
EntityInspection
```

read-only.

---

# 203. EntityInspection

Conceptualmente:

```text
EntityInspection
├── EntityType
├── EntityKey
├── PersistenceState
├── LoadedState
├── ChangeState
└── ScopeInformation
```

---

# 204. Inspection ≠ mutation

Developer tooling no deberá alterar UnitOfWork accidentalmente al inspeccionar.

---

# 205. Entity model security boundary

El Entity Model no deberá:

- abrir conexiones;
- ejecutar queries;
- resolver permisos;
- alterar schema;
- iniciar transactions;
- descubrir tenants;
- leer configuración global mutable.

---

# 206. Raw database identity

El ORM no deberá exponer obligatoriamente el valor físico de DB como identidad de dominio.

Ejemplo:

```text
Domain:
UserId(UUID)

DB:
BINARY(16)
```

---

# 207. Identifier conversion

```text
UserId
    ↓
IdentifierMapping
    ↓
Database Type
    ↓
Bound Parameter
```

y en sentido inverso:

```text
DB Result
    ↓
Type Conversion
    ↓
UserId
```

---

# 208. Identity round trip

Idealmente:

```text
Decode(Encode(EntityIdentifier))
=
EntityIdentifier
```

según igualdad semántica.

---

# 209. Identity round-trip failure

Una representación física incapaz de preservar la identidad requerida será incompatible.

---

# 210. Database capability interaction

El Entity Model declara la semántica requerida.

Platform/Type/Compatibility systems determinan si puede representarse.

---

# 211. No vendor identity in domain

Evitar:

```php
PostgresUuidEntityId
```

salvo que el dominio realmente dependa de esa semántica.

Preferir:

```php
UserId
```

---

# 212. Legacy database support

Para esquemas legacy deberán soportarse:

```text
composite keys
natural keys
nonstandard column names
assigned IDs
legacy sequences
```

sin degradar el modelo general.

---

# 213. Legacy identity adapter

Mapping podrá adaptar:

```text
legacy DB identity
```

a:

```text
canonical EntityIdentifier
```

---

# 214. Entity aliases

Renombrar una clase PHP no debería necesariamente cambiar la identidad conceptual del tipo dentro de metadata compilada/migrations/cache.

Podrá existir:

```text
EntityTypeName
```

estable separado del FQCN.

---

# 215. Stable entity type identifier

Se propone opcionalmente:

```php
#[Entity(name: 'user')]
final class User
{
}
```

donde:

```text
user
```

es un identificador lógico estable.

---

# 216. Entity type name collisions

Dos entidades no podrán registrar el mismo logical entity type dentro del mismo ORM registry.

---

# 217. FQCN mapping

El registry podrá mantener:

```text
logical type → class
class → logical type
```

---

# 218. Refactoring resilience

Esto permite que:

```text
App\Models\User
```

pase a:

```text
Domain\Identity\User
```

sin necesariamente invalidar identificadores lógicos persistidos en tooling/cache.

---

# 219. Entity type versioning

Metadata podrá incluir versión de mapping.

Pero:

```text
Mapping Version
≠
Entity Identity
```

---

# 220. Entity schema evolution

Una entidad puede cambiar mapping a lo largo del tiempo sin cambiar su identidad conceptual.

---

# 221. Entity rename

Renombrar la tabla:

```text
users
→
accounts
```

no implica:

```text
User identity changed
```

---

# 222. Table splitting

Una entidad podría mapearse en futuras capacidades a varias estructuras relacionales.

Por tanto:

```text
Entity
≠
Table
```

debe permanecer como invariante fuerte.

---

# 223. Multiple entities per table

Inheritance u otros mappings pueden permitir varias entidades sobre una tabla.

Otra razón para no igualar ambos conceptos.

---

# 224. Entity aggregate

DDD Aggregate será concepto de dominio.

No deberá asumirse:

```text
Entity
=
Aggregate Root
```

---

# 225. Aggregate root ≠ Entity

Un aggregate puede contener varias entidades.

---

# 226. Persistence cascade ≠ aggregate semantics

El ORM no deberá inferir automáticamente límites de aggregate a partir de cascades.

---

# 227. Entity ownership

Relationship ownership ORM tampoco equivale a ownership de dominio.

---

# 228. Entity reference integrity

Un `EntityReference` deberá contener suficiente información para reconstruir su canonical key bajo un contexto compatible.

---

# 229. Invalid context resolution

Si una referencia creada bajo:

```text
Tenant A
```

intenta resolverse bajo:

```text
Tenant B
```

deberá rechazarse o requerir traducción explícita según política.

---

# 230. Identity namespace propagation

El namespace deberá propagarse explícitamente por:

```text
EntityManager
EntityReference
IdentityMap
PersistenceContext
```

sin depender de global mutable tenant state.

---

# 231. Entity model API

Se proponen contratos:

```php
interface EntityIdentityNormalizer
{
    public function normalize(
        EntityType $type,
        mixed $identifier,
        EntityIdentityContext $context,
    ): NormalizedEntityIdentifier;
}
```

---

# 232. EntityKeyFactory

```php
interface EntityKeyFactory
{
    public function create(
        EntityType $type,
        mixed $identifier,
        EntityIdentityContext $context,
    ): EntityKey;
}
```

---

# 233. EntityTypeResolver

```php
interface EntityTypeResolver
{
    public function resolve(object|string $entity): EntityType;
}
```

---

# 234. EntityIdentityAccessor

```php
interface EntityIdentityAccessor
{
    public function read(object $entity): EntityIdentifierState;

    public function assign(
        object $entity,
        EntityIdentifier $identifier,
    ): void;
}
```

La asignación solo estará disponible para mappings que lo permitan.

---

# 235. EntityReferenceFactory

```php
interface EntityReferenceFactory
{
    public function create(
        EntityType $type,
        EntityIdentifier $identifier,
        EntityIdentityContext $context,
    ): EntityReference;
}
```

---

# 236. Entity model value objects

Namespace sugerido:

```text
VoltStack\Quantum\Database\ORM\Entity
├── EntityType
├── EntityTypeName
├── EntityDescriptor
├── EntityIdentifier
├── SimpleEntityIdentifier
├── CompositeEntityIdentifier
├── EntityIdentifierComponent
├── NormalizedEntityIdentifier
├── EntityIdentifierState
├── EntityKey
├── TemporaryEntityKey
├── EntityIdentityNamespace
├── EntityReference
└── EntityInspection
```

---

# 237. Identity subsystem

```text
VoltStack\Quantum\Database\ORM\Identity
├── EntityTypeResolver
├── EntityIdentityAccessor
├── EntityIdentityNormalizer
├── EntityKeyFactory
├── EntityReferenceFactory
├── IdentityCollisionPolicy
└── IdentifierGenerationStrategy
```

---

# 238. Proposed directory structure

```text
src/
└── Quantum/
    └── Database/
        └── ORM/
            ├── Entity/
            │   ├── EntityType.php
            │   ├── EntityTypeName.php
            │   ├── EntityDescriptor.php
            │   ├── EntityIdentifier.php
            │   ├── SimpleEntityIdentifier.php
            │   ├── CompositeEntityIdentifier.php
            │   ├── EntityIdentifierComponent.php
            │   ├── NormalizedEntityIdentifier.php
            │   ├── EntityIdentifierState.php
            │   ├── EntityKey.php
            │   ├── TemporaryEntityKey.php
            │   ├── EntityIdentityNamespace.php
            │   ├── EntityReference.php
            │   └── EntityInspection.php
            │
            ├── Identity/
            │   ├── EntityTypeResolver.php
            │   ├── EntityIdentityAccessor.php
            │   ├── EntityIdentityNormalizer.php
            │   ├── EntityKeyFactory.php
            │   ├── EntityReferenceFactory.php
            │   ├── IdentifierGenerationStrategy.php
            │   └── IdentityCollisionPolicy.php
            │
            └── Exception/
                ├── InvalidEntityException.php
                ├── InvalidEntityTypeException.php
                ├── InvalidEntityIdentifierException.php
                ├── IncompleteEntityIdentifierException.php
                ├── EntityIdentityMutationException.php
                ├── EntityIdentityCollisionException.php
                ├── EntityIdentityNormalizationException.php
                ├── EntityReferenceException.php
                ├── EntityReferenceContextException.php
                ├── EntityReconstitutionException.php
                └── EntityModelInvariantException.php
```

---

# 239. Exception hierarchy

```text
DatabaseOrmException
└── EntityModelException
    ├── InvalidEntityException
    ├── InvalidEntityTypeException
    ├── InvalidEntityIdentifierException
    ├── IncompleteEntityIdentifierException
    ├── EntityIdentityMutationException
    ├── EntityIdentityCollisionException
    ├── EntityIdentityNormalizationException
    ├── EntityReferenceException
    │   └── EntityReferenceContextException
    ├── EntityReconstitutionException
    └── EntityModelInvariantException
```

---

# 240. Testing strategy

El Entity Model deberá probarse principalmente sin base de datos.

---

# 241. Entity type tests

Verificar:

```text
canonical class resolution
proxy normalization
logical type names
duplicate logical type rejection
inheritance root identity
```

---

# 242. Identifier tests

Verificar:

```text
simple IDs
UUID
ULID
value object IDs
composite IDs
assigned IDs
generated IDs
incomplete IDs
```

---

# 243. Normalization tests

Propiedades:

```text
deterministic
idempotent
collision-free for valid domain
type-aware
context-aware
```

---

# 244. EntityKey tests

Comprobar:

```text
same type + same ID + same namespace
→ same key

same type + same ID + different tenant
→ different key

different type + same scalar ID
→ different key
```

---

# 245. Generated identity tests

```text
temporary identity
→ generated ID
→ atomic re-key
```

sin dejar entradas duplicadas.

---

# 246. Identity collision tests

Registrar dos objetos diferentes para la misma persistent identity deberá fallar.

---

# 247. Detached tests

Comprobar:

```text
clear()
→ object survives
→ ORM membership removed
→ persistent identity retained
```

---

# 248. Reference tests

Comprobar:

```text
reference creation
reference serialization
reference resolution
tenant mismatch
missing entity
proxy normalization
```

---

# 249. Runtime isolation tests

```text
Request A
→ EntityKey(User, 42)
→ Object A

reset

Request B
→ EntityKey(User, 42)
→ Object B

assert Object A !== Object B
```

---

# 250. Property-based tests

Para identificadores:

```text
Normalize(x) == Normalize(x)

Normalize(Normalize(x)) == Normalize(x)
```

cuando el dominio de tipos lo permita.

---

# 251. Round-trip tests

```text
Identifier
→ database representation
→ Identifier
```

deberá preservar igualdad semántica.

---

# 252. Invariantes

## DB-ENTITY-MODEL-001
Una Entity será distinta de una database row.

## DB-ENTITY-MODEL-002
Una Entity será distinta de una table.

## DB-ENTITY-MODEL-003
Una Entity será distinta de un DTO.

## DB-ENTITY-MODEL-004
Una Entity será distinta de un Value Object.

## DB-ENTITY-MODEL-005
Una Entity nunca generará SQL.

## DB-ENTITY-MODEL-006
Una Entity no dependerá de Connection.

## DB-ENTITY-MODEL-007
Una Entity no dependerá de Driver.

## DB-ENTITY-MODEL-008
Una Entity no dependerá obligatoriamente de EntityManager.

## DB-ENTITY-MODEL-009
EntityType será distinto de EntityInstance.

## DB-ENTITY-MODEL-010
CanonicalEntityType será estable frente a proxies equivalentes.

## DB-ENTITY-MODEL-011
Entity identity será distinta de PHP object identity.

## DB-ENTITY-MODEL-012
Entity identity será distinta de physical DB representation.

## DB-ENTITY-MODEL-013
EntityIdentifier será distinto de Schema PrimaryKeyDefinition.

## DB-ENTITY-MODEL-014
EntityIdentifier será distinto de EntityKey.

## DB-ENTITY-MODEL-015
EntityKey incluirá el namespace de identidad requerido.

## DB-ENTITY-MODEL-016
Tenant context podrá formar parte del identity namespace.

## DB-ENTITY-MODEL-017
Connection objects nunca formarán parte directa del EntityKey.

## DB-ENTITY-MODEL-018
Identifier normalization será type-aware.

## DB-ENTITY-MODEL-019
Identifier normalization será determinista.

## DB-ENTITY-MODEL-020
Identifier normalization no será coerción arbitraria.

## DB-ENTITY-MODEL-021
Composite identifiers tendrán orden canónico.

## DB-ENTITY-MODEL-022
Composite identifier components serán tipados.

## DB-ENTITY-MODEL-023
Persistent entity identity será inmutable por default.

## DB-ENTITY-MODEL-024
Primary key mutation no será una actualización ordinaria.

## DB-ENTITY-MODEL-025
Generated identifiers tendrán lifecycle explícito.

## DB-ENTITY-MODEL-026
Temporary runtime identity será distinta de persistent identity.

## DB-ENTITY-MODEL-027
Temporary-to-persistent re-keying será consistente.

## DB-ENTITY-MODEL-028
Null ID no definirá universalmente que una entidad es NEW.

## DB-ENTITY-MODEL-029
Unique field será distinto de Entity Identifier.

## DB-ENTITY-MODEL-030
State equality será distinta de entity identity equality.

## DB-ENTITY-MODEL-031
Dos NEW entities no serán iguales por tener los mismos valores.

## DB-ENTITY-MODEL-032
El ORM no dependerá de equals() del dominio para IdentityMap.

## DB-ENTITY-MODEL-033
Hashes de estado mutable no definirán persistent identity.

## DB-ENTITY-MODEL-034
Managed entity pertenecerá a un ORM scope.

## DB-ENTITY-MODEL-035
Detached será distinto de deleted.

## DB-ENTITY-MODEL-036
Detached entities no harán lazy load mágico.

## DB-ENTITY-MODEL-037
EntityReference será distinta de loaded Entity.

## DB-ENTITY-MODEL-038
EntityReference no probará existencia.

## DB-ENTITY-MODEL-039
EntityReference preservará type safety.

## DB-ENTITY-MODEL-040
Proxy será distinto de canonical entity type.

## DB-ENTITY-MODEL-041
Proxy y entity real compartirán EntityKey cuando representen la misma identidad.

## DB-ENTITY-MODEL-042
Proxy initialization state será distinto de ORM persistence state.

## DB-ENTITY-MODEL-043
Proxy-free operation será posible.

## DB-ENTITY-MODEL-044
ORM Core no exigirá base Model.

## DB-ENTITY-MODEL-045
POPO entities serán soportadas.

## DB-ENTITY-MODEL-046
Model entities y POPO entities compartirán ORM Core.

## DB-ENTITY-MODEL-047
La presencia de propiedad id no convertirá automáticamente una clase en Entity.

## DB-ENTITY-MODEL-048
Private identifiers serán soportables.

## DB-ENTITY-MODEL-049
Generated identifier mapping deberá declarar estrategia de escritura válida.

## DB-ENTITY-MODEL-050
ASSIGNED identity deberá estar disponible antes de persistencia.

## DB-ENTITY-MODEL-051
Logical generation strategy será distinta de physical DB mechanism.

## DB-ENTITY-MODEL-052
Entity Snapshot no pertenecerá a la Entity.

## DB-ENTITY-MODEL-053
Dirty state no necesitará almacenarse en la Entity.

## DB-ENTITY-MODEL-054
Domain state será distinto de ORM state.

## DB-ENTITY-MODEL-055
Collection loading state será distinto de Entity persistence state.

## DB-ENTITY-MODEL-056
Entity graph será distinto de persistence dependency graph.

## DB-ENTITY-MODEL-057
Graph cycles no causarán hydration recursiva infinita.

## DB-ENTITY-MODEL-058
API serialization será distinta de persistence mapping.

## DB-ENTITY-MODEL-059
Mapped field no implicará serialized field.

## DB-ENTITY-MODEL-060
Clonar una entity persistente no registrará automáticamente una segunda instancia con mismo key.

## DB-ENTITY-MODEL-061
Identity collisions nunca usarán last-wins silencioso.

## DB-ENTITY-MODEL-062
IdentityMap tendrá máximo una managed instance por EntityKey.

## DB-ENTITY-MODEL-063
Partial composite identifier no será persistent identity completa.

## DB-ENTITY-MODEL-064
UNKNOWN identifier state será distinto de UNASSIGNED.

## DB-ENTITY-MODEL-065
NULL identifier value será distinto de UNASSIGNED.

## DB-ENTITY-MODEL-066
persist() será distinto de upsert.

## DB-ENTITY-MODEL-067
Persistent ID no probará existencia de row.

## DB-ENTITY-MODEL-068
EntityReference no probará existencia de row.

## DB-ENTITY-MODEL-069
Concurrency version será distinta de Entity identity.

## DB-ENTITY-MODEL-070
Soft deleted será distinto de absent.

## DB-ENTITY-MODEL-071
Tenant ID será distinto de Entity ID salvo mapping explícito.

## DB-ENTITY-MODEL-072
Cross-tenant references serán explícitas.

## DB-ENTITY-MODEL-073
Entity ID será distinto de authorization permission.

## DB-ENTITY-MODEL-074
Sensitive identifiers podrán redactarse en telemetry.

## DB-ENTITY-MODEL-075
Persistent identifiers serán canonicalizables.

## DB-ENTITY-MODEL-076
Normalization será idempotente cuando el tipo lo permita.

## DB-ENTITY-MODEL-077
Normalization no introducirá colisiones semánticas.

## DB-ENTITY-MODEL-078
spl_object_id será distinto de persistent identity.

## DB-ENTITY-MODEL-079
UnitOfWork podrá indexar object identity y entity identity por separado.

## DB-ENTITY-MODEL-080
Metadata lookup utilizará canonical entity type.

## DB-ENTITY-MODEL-081
Inheritance mapping deberá preservar una identidad coherente.

## DB-ENTITY-MODEL-082
Identity root podrá ser distinto de runtime concrete type.

## DB-ENTITY-MODEL-083
Discriminator inconsistency no creará segunda managed entity.

## DB-ENTITY-MODEL-084
Discriminator metadata será distinta de EntityIdentifier.

## DB-ENTITY-MODEL-085
Entity logical type podrá ser distinto del FQCN.

## DB-ENTITY-MODEL-086
Logical entity type IDs deberán ser únicos dentro del registry.

## DB-ENTITY-MODEL-087
Renombrar FQCN no deberá obligatoriamente cambiar logical entity type.

## DB-ENTITY-MODEL-088
Mapping version será distinta de Entity identity.

## DB-ENTITY-MODEL-089
Table rename no cambiará automáticamente Entity identity.

## DB-ENTITY-MODEL-090
Entity será distinta de Aggregate Root.

## DB-ENTITY-MODEL-091
ORM relationship ownership será distinta de domain ownership.

## DB-ENTITY-MODEL-092
Identity namespace se propagará sin global mutable state.

## DB-ENTITY-MODEL-093
EntityManager scope no será serializable como parte de Entity identity.

## DB-ENTITY-MODEL-094
Managed state no cruzará request boundaries.

## DB-ENTITY-MODEL-095
Persistent identity sí podrá reconstruirse entre scopes.

## DB-ENTITY-MODEL-096
Live managed entities no se almacenarán en process-shared metadata caches.

## DB-ENTITY-MODEL-097
FrankenPHP requests no compartirán managed entity instances.

## DB-ENTITY-MODEL-098
RoadRunner workers no compartirán managed entity instances entre requests.

## DB-ENTITY-MODEL-099
OpenSwoole workers no compartirán managed entity instances entre scopes.

## DB-ENTITY-MODEL-100
Entity debugging no modificará estado ORM.

## DB-ENTITY-MODEL-101
EntityInspection será read-only.

## DB-ENTITY-MODEL-102
Entity Model no abrirá conexiones.

## DB-ENTITY-MODEL-103
Entity Model no ejecutará queries.

## DB-ENTITY-MODEL-104
Entity Model no modificará schema.

## DB-ENTITY-MODEL-105
Entity Model no iniciará transactions.

## DB-ENTITY-MODEL-106
Entity Model no resolverá authorization.

## DB-ENTITY-MODEL-107
Domain identifier podrá diferir del DB physical identifier.

## DB-ENTITY-MODEL-108
Identifier conversion utilizará Mapping + Type System.

## DB-ENTITY-MODEL-109
Identifier round-trip deberá preservar igualdad semántica.

## DB-ENTITY-MODEL-110
Lossy identity representation será incompatible.

## DB-ENTITY-MODEL-111
Vendor-specific physical identity no contaminará el dominio por default.

## DB-ENTITY-MODEL-112
Legacy composite identities serán soportables.

## DB-ENTITY-MODEL-113
Legacy mapping no cambiará los invariantes centrales.

## DB-ENTITY-MODEL-114
Entity references para queues serán preferibles a managed entity serialization.

## DB-ENTITY-MODEL-115
Managed Entity no serializará UnitOfWork internals.

## DB-ENTITY-MODEL-116
Entity Type y Entity Identifier tendrán representación diagnóstica determinista.

## DB-ENTITY-MODEL-117
Entity identity mutation será detectada.

## DB-ENTITY-MODEL-118
Identity collision será detectada.

## DB-ENTITY-MODEL-119
Invalid generated-ID assignment será detectada.

## DB-ENTITY-MODEL-120
Entity reconstitution strategy será explícita.

## DB-ENTITY-MODEL-121
Domain construction será distinta de persistence reconstitution.

## DB-ENTITY-MODEL-122
Constructor bypass no será accidental.

## DB-ENTITY-MODEL-123
Immutable managed entity support requerirá replacement policy explícita.

## DB-ENTITY-MODEL-124
Dos managed instances para el mismo key no coexistirán silenciosamente.

## DB-ENTITY-MODEL-125
Mutable managed entities serán el modelo inicial recomendado.

## DB-ENTITY-MODEL-126
EntityReference resolverá mediante contexto compatible.

## DB-ENTITY-MODEL-127
Tenant mismatch durante reference resolution no será ignorado.

## DB-ENTITY-MODEL-128
Cross-database identity será explícita.

## DB-ENTITY-MODEL-129
Entity existence será distinta de identity knowledge.

## DB-ENTITY-MODEL-130
KNOWN_ABSENT será distinto de UNKNOWN.

## DB-ENTITY-MODEL-131
Identity normalization utilizará metadata congelada.

## DB-ENTITY-MODEL-132
Identity normalization no realizará DB I/O.

## DB-ENTITY-MODEL-133
EntityKeyFactory no realizará DB I/O.

## DB-ENTITY-MODEL-134
EntityTypeResolver no ejecutará queries.

## DB-ENTITY-MODEL-135
Entity identity accessors podrán compilarse/cachearse.

## DB-ENTITY-MODEL-136
Shared identity metadata será immutable.

## DB-ENTITY-MODEL-137
Runtime temporary keys serán scope-local.

## DB-ENTITY-MODEL-138
Temporary keys no se persistirán como DB identifiers.

## DB-ENTITY-MODEL-139
Temporary key replacement será atómico respecto al ORM scope.

## DB-ENTITY-MODEL-140
Generated identity failure no dejará IdentityMap corrupto.

## DB-ENTITY-MODEL-141
Entity Model será compatible con static analysis.

## DB-ENTITY-MODEL-142
Entity Model no dependerá de PHPStan/Psalm en runtime.

## DB-ENTITY-MODEL-143
Entity marker será explícito.

## DB-ENTITY-MODEL-144
Entity type registry será determinista.

## DB-ENTITY-MODEL-145
Entity type registry podrá congelarse.

## DB-ENTITY-MODEL-146
Duplicate logical entity type será error.

## DB-ENTITY-MODEL-147
Identity semantics no dependerán del orden de descubrimiento de clases.

## DB-ENTITY-MODEL-148
EntityKey serialization será versionada si se usa fuera del proceso.

## DB-ENTITY-MODEL-149
EntityReference serialization no incluirá live ORM resources.

## DB-ENTITY-MODEL-150
La identidad será la base de continuidad de una Entity dentro del ORM.

---

# 253. Anti-patterns

## 253.1 Entity como row genérica

```php
class User
{
    public array $attributes;
}
```

puede existir como DX, pero no deberá definir la arquitectura completa del Entity Model.

---

## 253.2 ID basado en `spl_object_id`

Incorrecto:

```php
$id = spl_object_id($user);
```

como persistent identity.

---

## 253.3 Null ID = NEW universal

Incorrecto:

```php
if ($entity->id === null) {
    INSERT;
} else {
    UPDATE;
}
```

---

## 253.4 Persist = upsert

Incorrecto:

```text
has ID?
├── yes → UPDATE
└── no  → INSERT
```

como regla universal.

---

## 253.5 Entity = table

Incorrecto:

```text
1 Entity
=
exactly 1 table forever
```

---

## 253.6 Global identity namespace

Incorrecto en multitenancy:

```text
User#42
```

sin considerar el contexto requerido.

---

## 253.7 Mutable primary identity

Incorrecto:

```php
$user->id = 200;
```

mientras está managed.

---

## 253.8 Duplicate managed entities

Incorrecto:

```text
User#42 → Object A
User#42 → Object B
```

en el mismo IdentityMap.

---

## 253.9 Serialize EntityManager with Entity

Incorrecto:

```text
User
├── data
└── EntityManager
```

cruzando queues.

---

## 253.10 Detached lazy loading

Incorrecto:

```text
detached entity
→ property access
→ magically find global EntityManager
→ query
```

---

## 253.11 Authorization through obscurity of ID

Incorrecto:

```text
UUID is hard to guess
⇒ authorization unnecessary
```

---

## 253.12 Equality by all fields

Incorrecto para identidad:

```text
User#10(name=Ana)
=
User#20(name=Ana)
```

---

## 253.13 Entity state inside domain flags

Evitar:

```php
private bool $__ormManaged;
private bool $__ormDirty;
```

como requisito universal.

---

## 253.14 Physical DB type leaking into domain

Evitar:

```php
private PostgresUuid $id;
```

si el dominio solo necesita `UserId`.

---

# 254. Ejemplo completo

Entidad:

```php
#[Entity(name: 'user')]
#[Table('users')]
final class User
{
    public function __construct(
        #[Id]
        #[Column(type: 'uuid')]
        private UserId $id,

        #[Column(length: 150)]
        private string $name,

        #[Column]
        private Email $email,
    ) {}

    public function id(): UserId
    {
        return $this->id;
    }

    public function rename(string $name): void
    {
        $this->name = $name;
    }
}
```

Metadata:

```text
EntityType
    name = user
    class = App\Domain\User

IdentifierMapping
    field = id
    domainType = UserId
    databaseType = UUID
    strategy = CLIENT_GENERATED
```

---

# 255. Entity creation

```php
$user = new User(
    UserId::new(),
    'Ana',
    Email::fromString('ana@example.test'),
);
```

Estado:

```text
Object Identity
    ↓
PHP object

Persistent Identity
    ↓
UserId(...)

ORM State
    ↓
NEW / unmanaged
```

---

# 256. Persist

```php
$entityManager->persist($user);
```

produce:

```text
EntityTypeResolver
      ↓
EntityMetadata
      ↓
IdentifierAccessor
      ↓
UserId
      ↓
IdentifierNormalizer
      ↓
Normalized ID
      ↓
EntityKey
      ↓
IdentityMap
      ↓
UnitOfWork
```

---

# 257. Identity Map

Después:

```text
IdentityMap
└── [Tenant/DB Context, user, UUID]
      ↓
    $user
```

---

# 258. Re-query

```php
$again = $entityManager->find(
    User::class,
    $user->id(),
);
```

deberá producir:

```php
assert($again === $user);
```

dentro del mismo scope.

---

# 259. Clear

```php
$entityManager->clear();
```

después:

```text
$user
    ↓
still PHP object
    ↓
persistent identity retained
    ↓
DETACHED
```

---

# 260. New lookup

```php
$loaded = $entityManager->find(
    User::class,
    $user->id(),
);
```

puede producir:

```text
$loaded !== $user
```

porque `$user` estaba detached y el IdentityMap fue limpiado.

Sin embargo:

```text
SameEntityIdentity($loaded, $user)
=
true
```

---

# 261. Fórmula maestra

```text
Entity Model
=
Canonical Entity Type
+
Persistent Identity Semantics
+
Typed Identifier
+
Identity Namespace
+
Canonical Entity Key
+
Domain State
+
Domain Behavior
+
Explicit Construction/Reconstitution
+
Reference Semantics
+
Runtime Scope Isolation
```

---

# 262. Fórmula de identidad

```text
EntityIdentity(E)
=
IdentityNamespace(E)
+
CanonicalEntityType(E)
+
NormalizedIdentifier(E)
```

---

# 263. Fórmula de igualdad ORM

```text
SamePersistentEntity(A, B)
=
Namespace(A) = Namespace(B)
∧
CanonicalType(A) = CanonicalType(B)
∧
NormalizedIdentifier(A) = NormalizedIdentifier(B)
```

---

# 264. Fórmula de managed identity

Dentro de un ORM scope:

```text
∀ key:

|ManagedInstances(key)| ≤ 1
```

---

# 265. Fórmula de seguridad de scope

```text
SafeManagedEntity
=
ValidEntityMetadata
∧
ValidIdentity
∧
CanonicalEntityKey
∧
SingleManagedInstancePerKey
∧
ScopedManagementState
∧
NoCrossRequestReferenceLeak
```

---

# 266. Fórmula de identificador válido

```text
ValidEntityIdentifier
=
TypeValid
∧
CompleteWhenRequired
∧
Canonicalizable
∧
Stable
∧
NonLossyNormalization
∧
MappingCompatible
```

---

# 267. Regla maestra

> **Una Entity en VoltStack es un objeto de dominio identificado mediante una identidad persistente tipada y contextualizada; el ORM administra externamente su estado de persistencia sin convertir a la entidad en una fila, una conexión, una consulta o un contenedor de infraestructura.**

La separación final será:

```text
Entity
    ↓
"What domain object is this?"

EntityType
    ↓
"What kind of persistent object is this?"

EntityIdentifier
    ↓
"What persistent identity does it declare?"

EntityKey
    ↓
"What canonical identity does the ORM use?"

IdentityMap
    ↓
"Which PHP object currently represents that identity?"

UnitOfWork
    ↓
"What persistence state does that object currently have?"

EntityMetadata
    ↓
"How is that object mapped?"

Persistence Engine
    ↓
"How are its changes persisted?"
```

---

# 268. Resultado arquitectónico

El Entity Model permite que VoltStack soporte simultáneamente:

```text
Simple application models
        +
Rich domain entities
        +
DDD-style entities
        +
Value-object identifiers
        +
Legacy database identities
        +
Composite identifiers
        +
Generated identifiers
        +
Lazy references
        +
Persistent runtimes
        +
Multitenant identity isolation
```

sin hacer que las entidades conozcan SQL o infraestructura de base de datos.

La arquitectura queda:

```text
Domain-friendly
+
ORM-aware externally
+
Persistence-ignorant internally
+
Identity-safe
+
Runtime-safe
```

---

# 269. Relación con documentos siguientes

Este documento define:

```text
Entity semantics
+
identity semantics
+
runtime entity boundaries
```

Los siguientes documentos profundizarán:

```text
114_DATABASE_MODEL_API_SYSTEM.md
    ↓
Developer-facing Active Record-like API

115_DATABASE_ENTITY_METADATA_SYSTEM.md
    ↓
Compiled description of entities

116_DATABASE_ENTITY_MAPPING_SYSTEM.md
    ↓
Entity ↔ relational mapping

117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
    ↓
PHP Attribute mapping source

118_DATABASE_ENTITY_MANAGER_SYSTEM.md
    ↓
ORM context coordination

119_DATABASE_REPOSITORY_SYSTEM.md
    ↓
Entity collection access

120_DATABASE_ENTITY_QUERY_SYSTEM.md
    ↓
Entity-aware querying

121_DATABASE_ENTITY_STATE_SYSTEM.md
    ↓
Formal persistence state machine

122_DATABASE_ENTITY_LIFECYCLE_SYSTEM.md
    ↓
Entity lifecycle behavior
```

Posteriormente:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
```

tomará:

```text
EntityKey
+
Single Managed Instance Invariant
+
Identity Namespace
```

y los convertirá en el sistema runtime completo de identidad.

---

# 270. Siguiente documento

```text
114_DATABASE_MODEL_API_SYSTEM.md
```

Este documento definirá la capa de Developer Experience estilo Laravel de VoltStack:

```text
Model
Model::query()
Model::find()
Model::create()
save()
delete()
refresh()
fresh()
replicate()
attributes
mass assignment
casts
serialization boundaries
relationship access
Model Context Resolver
Model Persistence Bridge
Model Query Bridge
scoped static API
```

manteniendo el principio:

> **`Model` será una API ergonómica sobre el ORM Core; nunca un segundo ORM, un segundo UnitOfWork ni un Persistence Engine alternativo.**