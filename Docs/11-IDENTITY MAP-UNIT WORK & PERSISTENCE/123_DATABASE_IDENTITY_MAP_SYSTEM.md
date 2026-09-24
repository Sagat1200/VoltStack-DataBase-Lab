# 123_DATABASE_IDENTITY_MAP_SYSTEM.md

# VoltStack Quantum Database
## Database Identity Map System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 123 — Database Identity Map System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Identity Map System` define el mecanismo mediante el cual el ORM de VoltStack mantiene una única instancia canónica administrada para cada identidad lógica de entidad dentro de un `PersistenceContext`.

Su responsabilidad principal puede expresarse como:

```text
EntityKey
    ↓
IdentityMap
    ↓
Canonical Managed Entity Instance
```

La invariante central es:

> **Dentro de un mismo PersistenceContext, un EntityKey establecido puede corresponder como máximo a una única instancia canónica administrada.**

Esto permite que múltiples consultas que representen la misma entidad:

```php
$userA = $entityManager->find(User::class, 10);
$userB = $entityManager->find(User::class, 10);
```

produzcan:

```php
$userA === $userB;
```

mientras ambas operaciones pertenezcan al mismo contexto ORM.

---

# 2. Problema que resuelve

Sin Identity Map podrían existir simultáneamente:

```text
User#10 instance A
User#10 instance B
User#10 instance C
```

dentro de la misma operación.

Entonces:

```php
$userA->rename('Alice');
$userB->rename('Bob');
```

generaría ambigüedad:

```text
¿Cuál representa User#10?
¿Cuál debe persistirse?
¿Qué snapshot es correcto?
¿Qué relaciones apuntan a la entidad correcta?
¿Qué instancia debe devolver una consulta posterior?
```

Identity Map elimina esta ambigüedad.

---

# 3. Identity Map ≠ Cache

Regla fundamental:

```text
IdentityMap
≠
Second-Level Cache
≠
Result Cache
≠
Query Cache
≠
Application Cache
```

Identity Map mantiene:

```text
EntityKey → live managed object instance
```

mientras que un cache puede mantener:

```text
CacheKey → serialized/cached data
```

---

# 4. Identity Map es scope-local

Identity Map pertenece a:

```text
PersistenceContext
```

y no al proceso completo.

Por tanto:

```text
Request A IdentityMap
≠
Request B IdentityMap
```

incluso bajo FrankenPHP.

---

# 5. Identity Map ≠ UnitOfWork

Identity Map responde:

> ¿Qué instancia representa actualmente esta identidad?

UnitOfWork responde:

> ¿Qué cambios de persistencia deben sincronizarse?

Por tanto:

```text
IdentityMap
≠
UnitOfWork
```

---

# 6. Identity Map ≠ Entity State

Identity Map contiene asociaciones entre identidad e instancia.

Entity State determina si una entidad está:

```text
NEW
MANAGED
REMOVED
DETACHED
UNKNOWN
```

Una entidad `NEW` sin identidad persistente establecida puede existir en UnitOfWork sin estar todavía registrada bajo un `EntityKey` persistente definitivo.

---

# 7. Identity Map ≠ Database existence

Encontrar:

```text
User#10
```

en Identity Map significa:

```text
the PersistenceContext currently associates User#10
with this object instance
```

No significa necesariamente:

```text
the database currently contains row User#10
```

La fila pudo:

- ser eliminada externamente;
- cambiar en otra transacción;
- estar pendiente de eliminación;
- encontrarse bajo incertidumbre operacional.

---

# 8. Identity Map ≠ Global Object Registry

Nunca deberá implementarse:

```php
static array $entities = [];
```

como registro global del proceso.

Esto sería especialmente peligroso en:

```text
FrankenPHP
RoadRunner
OpenSwoole
queue workers
long-running commands
```

---

# 9. Relación con Entity Model

El documento `113_DATABASE_ENTITY_MODEL.md` estableció conceptualmente:

```text
EntityKey
=
IdentityNamespace
+
EntityType
+
CanonicalIdentifier
```

Identity Map consume esa identidad.

No redefine la identidad de entidad.

---

# 10. EntityKey

Modelo conceptual:

```php
final readonly class EntityKey
{
    public function __construct(
        public EntityIdentityNamespace $namespace,
        public EntityType $entityType,
        public EntityIdentifier $identifier,
    ) {}
}
```

---

# 11. Fórmula

```text
EntityKey
=
IdentityNamespace
×
EntityType
×
CanonicalIdentifier
```

---

# 12. Por qué EntityType forma parte de la key

Esto:

```text
User#10
```

no es lo mismo que:

```text
Invoice#10
```

aunque ambos tengan:

```text
identifier = 10
```

Por tanto:

```text
EntityKey(User, 10)
≠
EntityKey(Invoice, 10)
```

---

# 13. Identity Namespace

La misma entidad lógica puede existir en distintos contextos de datos.

Ejemplo:

```text
Tenant A → User#10
Tenant B → User#10
```

No son necesariamente la misma entidad.

Por ello:

```text
EntityKey
```

debe incorporar un namespace lógico.

---

# 14. Namespace ≠ Connection Object

Incorrecto:

```text
EntityKey = spl_object_id(PDO)
```

Correcto:

```text
EntityIdentityNamespace
```

debe representar identidad lógica estable del contexto.

---

# 15. Ejemplo

```text
namespace:
tenant:acme/database:primary

entity:
User

identifier:
10
```

produce conceptualmente:

```text
tenant:acme/database:primary | User | int:10
```

---

# 16. Tenant no estará hardcoded

El núcleo ORM no deberá asumir:

```text
tenant_id
```

como parte obligatoria.

El Multitenancy package podrá contribuir al:

```text
EntityIdentityNamespace
```

mediante integración explícita.

---

# 17. Sharding

La misma consideración aplica a shards:

```text
Shard A → User#10
Shard B → User#10
```

Si ambos espacios permiten identidades independientes:

```text
IdentityNamespace
```

deberá diferenciarlos.

---

# 18. Replica routing

Primary y replica no necesariamente deberán crear namespaces diferentes.

Si ambos representan el mismo espacio lógico de identidad:

```text
primary
replica
```

deben converger en la misma identidad ORM.

---

# 19. Regla

> El Identity Namespace representa el espacio lógico de identidad de datos, no necesariamente la conexión física utilizada para leerlos.

---

# 20. Canonical Identifier

Identity Map nunca deberá comparar IDs mediante coerción PHP accidental.

Incorrecto:

```php
"10" == 10;
```

como política universal.

---

# 21. Canonicalización

El identifier deberá transformarse mediante el sistema de tipos:

```text
External Identifier
        ↓
Identifier Type
        ↓
Canonicalization
        ↓
EntityIdentifier
```

---

# 22. Ejemplo

Si el mapping define:

```text
User.id : integer
```

entonces:

```text
10
"10"
```

podrían converger a:

```text
CanonicalIntegerIdentifier(10)
```

si el tipo explícitamente permite esa conversión.

---

# 23. No coerción implícita

El Identity Map no deberá decidir por sí mismo:

```text
"10" == 10
```

La semántica pertenece al Identifier/Type System.

---

# 24. Empty identifier

No deberá suponerse:

```text
0 = no ID
"" = no ID
```

Eso dependerá de:

```text
identifier type
+
identity strategy
```

---

# 25. Float identifiers

Los identificadores `float` deberán rechazarse como identidad persistente estándar debido a problemas de canonicalización y precisión.

---

# 26. Simple identifiers

Ejemplos:

```text
int:10
uuid:550e8400-e29b-41d4-a716-446655440000
ulid:01J...
string:customer-A31
```

---

# 27. Strongly typed IDs

VoltStack deberá soportar:

```php
final readonly class UserId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Entonces:

```php
new UserId('01J...')
```

puede formar parte del `EntityIdentifier`.

---

# 28. Composite identifiers

Ejemplo:

```text
OrderLine
    orderId = 100
    lineNumber = 4
```

No deberá convertirse ingenuamente en:

```text
"100:4"
```

---

# 29. CompositeEntityIdentifier

```php
final readonly class CompositeEntityIdentifier
{
    /**
     * @param list<EntityIdentifierPart> $parts
     */
    public function __construct(
        public array $parts,
    ) {}
}
```

---

# 30. Orden canónico

Los componentes deberán utilizar el orden definido por metadata:

```text
(orderId, lineNumber)
```

y no:

```text
arbitrary array order
```

---

# 31. Typed composite equality

```text
(orderId:int:10, locale:string:"en")
```

será distinto de:

```text
(orderId:string:"10", locale:string:"en")
```

salvo que los tipos correspondientes canonicalicen ambos al mismo valor.

---

# 32. EntityKey hashing

Para eficiencia, `EntityKey` podrá generar:

```text
EntityKeyHash
```

---

# 33. Hash ≠ identity

```text
EntityKeyHash
≠
EntityKey
```

El hash será una representación optimizada para lookup.

---

# 34. Collision handling

Aunque el hash sea robusto:

```text
hash collision
```

no deberá producir aliasing de entidades.

La implementación deberá preservar suficiente información para verificar igualdad real del `EntityKey`.

---

# 35. No concatenación ambigua

Incorrecto:

```php
$key = $type . ':' . implode(':', $ids);
```

porque:

```text
["a:b", "c"]
```

puede colisionar semánticamente con otras combinaciones.

---

# 36. Canonical key encoding

Si se utiliza representación textual interna deberá ser:

```text
typed
length-safe
versioned
deterministic
```

---

# 37. IdentityMap contract

Contrato conceptual:

```php
interface IdentityMap
{
    public function contains(EntityKey $key): bool;

    public function get(EntityKey $key): ?object;

    public function register(
        EntityKey $key,
        object $entity
    ): void;

    public function remove(EntityKey $key): void;

    public function keyOf(object $entity): ?EntityKey;

    public function clear(): void;

    public function count(): int;
}
```

---

# 38. Lookup

```text
EntityKey
   │
   ▼
IdentityMap
   │
   ├── HIT
   │    ↓
   │  canonical entity
   │
   └── MISS
        ↓
      loader
```

---

# 39. Lookup ≠ Database Query

`IdentityMap::get()` nunca deberá consultar la base de datos.

---

# 40. Identity Map es in-memory

Su operación fundamental deberá ser:

```text
memory lookup
```

sin I/O.

---

# 41. EntityLoader integration

Carga:

```text
EntityManager::find(User, 10)
        │
        ▼
Canonical EntityKey
        │
        ▼
IdentityMap
      /       \
    HIT       MISS
    │          │
    ▼          ▼
 entity      Entity Query
                │
                ▼
             Result
                │
                ▼
            Hydration
                │
                ▼
          IdentityMap register
```

---

# 42. IdentityMap-first lookup

Antes de ejecutar una consulta por identidad:

```text
IdentityMap
```

deberá consultarse cuando la semántica de la operación lo permita.

---

# 43. find()

Conceptualmente:

```php
$entityManager->find(User::class, 10);
```

deberá realizar:

```text
resolve EntityType
→ canonicalize identifier
→ build EntityKey
→ IdentityMap lookup
→ database load only on MISS
```

---

# 44. find() y stale state

Un HIT no implica que la entidad tenga el estado más reciente de la base.

Eso es deliberado.

---

# 45. Persistence Context consistency

Identity Map proporciona:

```text
repeatable object identity
```

no necesariamente:

```text
latest database state
```

---

# 46. Refresh

Para obtener nuevo estado:

```php
$entityManager->refresh($user);
```

deberá actualizar la misma instancia administrada cuando sea semánticamente válido.

---

# 47. refresh() no reemplaza instancia

Idealmente:

```text
User#10 instance A
    ↓ refresh
User#10 instance A
```

y no:

```text
instance A
→ instance B
```

---

# 48. Registration

Registrar:

```text
EntityKey(User,10)
→ object A
```

cuando ya existe:

```text
EntityKey(User,10)
→ object A
```

deberá ser idempotente.

---

# 49. Identity conflict

Registrar:

```text
EntityKey(User,10)
→ object B
```

cuando ya existe:

```text
EntityKey(User,10)
→ object A
```

y:

```php
$objectA !== $objectB
```

deberá producir conflicto.

---

# 50. No last-wins

Nunca:

```text
map[key] = newObject
```

silenciosamente.

---

# 51. EntityIdentityConflictException

El conflicto deberá incluir:

```text
EntityType
Identifier
IdentityNamespace
existing object identity
incoming object identity
PersistenceContext
```

sin exponer datos sensibles innecesarios.

---

# 52. Why conflict matters

Permitir reemplazo silencioso destruiría:

```text
UnitOfWork tracking
snapshots
relationships
change sets
object references
lifecycle assumptions
```

---

# 53. Reverse lookup

Además de:

```text
EntityKey → object
```

es útil:

```text
object → EntityKey
```

---

# 54. Reverse identity map

Conceptualmente:

```text
ObjectIdentity
    ↓
EntityKey
```

para:

```text
detach
state resolution
diagnostics
identifier re-keying
UnitOfWork integration
```

---

# 55. PHP object identity

PHP dispone de:

```php
spl_object_id($entity);
```

pero este valor:

```text
≠ EntityIdentifier
≠ EntityKey
```

Solo identifica la instancia PHP durante su vida.

---

# 56. Forward + reverse indexes

Implementación conceptual:

```text
byKey:
EntityKeyHash → Entry

byObject:
ObjectIdentity → EntityKey
```

---

# 57. IdentityMapEntry

```php
final class IdentityMapEntry
{
    public function __construct(
        public readonly EntityKey $key,
        public readonly object $entity,
    ) {}
}
```

La entrada no deberá convertirse en un segundo contenedor de Entity State.

---

# 58. Identity Map no almacena snapshots

Snapshots pertenecen a:

```text
Entity Snapshot System
```

---

# 59. Identity Map no almacena ChangeSets

ChangeSets pertenecen a:

```text
Change Tracking / UnitOfWork
```

---

# 60. Identity Map no decide dirty state

No deberá hacer:

```text
entity changed?
```

---

# 61. Identity Map no persiste

No deberá ejecutar:

```text
INSERT
UPDATE
DELETE
```

---

# 62. Registration precondition

Una entidad solo podrá registrarse mediante un `EntityKey` cuando exista un identifier canónico suficiente para formar su identidad.

---

# 63. NEW entity con ID asignado

Ejemplo:

```php
$user = new User(
    id: UserId::generate()
);
```

Puede tener:

```text
identifier available
```

antes de persistirse.

---

# 64. Identifier assigned ≠ persistent identity established

Aunque tenga ID:

```text
User#UUID
```

todavía puede no existir en DB.

Identity Map deberá distinguir:

```text
identity key available
```

de:

```text
database persistence established
```

---

# 65. Registration of NEW assigned-ID entities

VoltStack podrá registrar una entidad `NEW` con ID asignado para prevenir duplicados dentro del contexto.

Pero esto no cambiará automáticamente su Entity State a `MANAGED persisted`.

---

# 66. Identity Map membership ≠ persisted state

Regla:

```text
registered in IdentityMap
≠
database row exists
```

---

# 67. Database-generated identifier

Ejemplo:

```text
AUTO_INCREMENT
IDENTITY
SEQUENCE-dependent assignment
```

La entidad puede iniciar:

```text
identifier unavailable
```

---

# 68. Temporary UnitOfWork identity

Antes del INSERT, UnitOfWork podrá utilizar:

```text
TemporaryEntityToken
```

para construir el graph.

---

# 69. TemporaryEntityToken ≠ EntityKey

Nunca:

```text
temp:object:173
```

deberá exponerse como identidad persistente.

---

# 70. Generated ID flow

```text
NEW Entity
    │
    ├── no persistent identifier
    │
    ▼
Temporary UoW Token
    │
    ▼
INSERT
    │
    ▼
Execution Result
    │
    ▼
Generated Identifier
    │
    ▼
Canonicalize Identifier
    │
    ▼
Build EntityKey
    │
    ▼
Register IdentityMap
```

---

# 71. Generated identity certainty

El generated ID solo deberá registrarse como identidad estable cuando el Execution/Persistence Engine tenga certeza suficiente sobre el resultado.

---

# 72. UNKNOWN insert outcome

Caso:

```text
INSERT sent
connection lost
outcome UNKNOWN
```

VoltStack no deberá inventar:

```text
entity definitely persisted
```

---

# 73. Generated ID + UNKNOWN

Si el driver recibió un ID pero el outcome durable es incierto:

```text
identifier value may be known
persistence outcome may remain UNKNOWN
```

Estas dimensiones deberán conservarse separadas.

---

# 74. IdentityMap after UNKNOWN

No deberá promoverse silenciosamente una entidad a estado sincronizado únicamente porque exista un identifier.

La reconciliación pertenece a Persistence/Entity State.

---

# 75. Re-keying

Una entidad puede requerir:

```text
no key
→ generated EntityKey
```

después del INSERT.

Esto no es mutación arbitraria del ID.

Es:

```text
initial identity establishment
```

---

# 76. Re-keying contract

Se propone:

```php
interface MutableIdentityMap extends IdentityMap
{
    public function establish(
        object $entity,
        EntityKey $key
    ): void;
}
```

El nombre final puede variar, pero la semántica deberá diferenciar:

```text
initial establishment
```

de:

```text
identifier mutation
```

---

# 77. Identifier mutation

Una vez establecida:

```text
User#10
```

cambiar a:

```text
User#11
```

deberá rechazarse por defecto.

---

# 78. Razón

Cambiar identity rompe:

```text
IdentityMap
UnitOfWork
relationships
snapshots
references
foreign-key semantics
cache keys
```

---

# 79. EntityIdentifierMutationException

Debe producirse si una entidad managed intenta cambiar su identifier establecido sin una operación explícitamente soportada.

---

# 80. Primary key update

Aunque una DB soporte:

```sql
UPDATE users SET id = 11 WHERE id = 10;
```

eso no significa que el ORM deba soportarlo como mutación normal de identidad.

---

# 81. Política VoltStack

V1:

```text
managed entity identifier mutation
=
unsupported
```

---

# 82. Natural identifiers

Si se utiliza una clave natural:

```text
email
sku
countryCode + number
```

deberá considerarse inmutable si forma parte de la identidad ORM.

---

# 83. Mutable business key

Si el valor puede cambiar:

```text
email
username
slug
```

es preferible no utilizarlo como identidad primaria del Entity Model.

---

# 84. Hydration integration

Hydration deberá consultar Identity Map antes de construir una nueva entidad.

---

# 85. Hydration algorithm

```text
Row
 ↓
Extract Identifier
 ↓
Canonicalize
 ↓
Build EntityKey
 ↓
IdentityMap lookup
 ├── HIT
 │    ↓
 │  reuse
 │
 └── MISS
      ↓
   instantiate
      ↓
   hydrate
      ↓
   register
```

---

# 86. JOIN duplication

Consulta:

```text
User#10 + Role#1
User#10 + Role#2
User#10 + Role#3
```

deberá producir:

```text
one User instance
three Role associations
```

---

# 87. No duplicate entity from JOIN

La cantidad de rows físicas no determina la cantidad de instancias entity.

---

# 88. Registration timing during hydration

Existe un problema de ciclos:

```text
User
 ↔
Profile
```

Si se hidrata completamente antes de registrar, las relaciones pueden intentar crear duplicados.

---

# 89. Early registration

Puede ser necesario:

```text
instantiate
→ establish identity
→ register
→ hydrate remaining fields
```

---

# 90. Partially initialized entry

Esto introduce una fase interna:

```text
INITIALIZING
```

que no deberá confundirse con Entity State público.

---

# 91. Internal IdentityMapEntryState

```php
enum IdentityMapEntryState
{
    case INITIALIZING;
    case READY;
}
```

---

# 92. INITIALIZING entry

Existe únicamente para resolver:

```text
cyclic hydration
recursive graph materialization
```

---

# 93. No exposure of incomplete entities

Una entidad `INITIALIZING` no deberá escapar arbitrariamente hacia código de aplicación antes de completar sus invariantes de hydration.

---

# 94. Hydration failure

Si hydration falla después de early registration:

```text
IdentityMap
```

deberá retirar la entrada incompleta.

---

# 95. Atomic registration semantics

Conceptualmente:

```text
reserve key
→ instantiate/register initializing
→ hydrate
→ finalize ready
```

o:

```text
failure
→ remove reservation
```

---

# 96. Identity reservation

Puede modelarse mediante:

```text
IdentityReservation
```

para evitar duplicados durante materialización recursiva.

---

# 97. Reservation ≠ Entity State

Es infraestructura interna del Hydration/Identity Map.

---

# 98. IdentityMap get during initialization

La política deberá ser explícita.

Posibles consumidores internos podrán obtener la instancia para cerrar ciclos.

Consumidores externos no deberán observarla como entidad completamente lista.

---

# 99. Proxy integration

Un lazy-loading proxy:

```text
UserProxy#10
```

no deberá crear una identidad distinta de:

```text
User#10
```

---

# 100. Proxy class ≠ EntityType

Nunca:

```php
get_class($proxy)
```

deberá utilizarse directamente como identidad lógica.

---

# 101. Proxy type resolution

```text
Proxy Class
    ↓
EntityTypeResolver
    ↓
User EntityType
```

---

# 102. Proxy key

```text
EntityKey(User,10)
```

será el mismo independientemente de:

```text
User
UserProxy
LazyGhost<User>
```

---

# 103. Proxy registration

Si Identity Map contiene:

```text
UserProxy#10
```

y posteriormente una consulta carga `User#10`, no deberá crear otra instancia real separada.

---

# 104. Proxy initialization

La instancia proxy canónica deberá inicializarse o reconciliarse según la estrategia de proxy.

---

# 105. Proxy replacement

Reemplazar:

```text
proxy instance
→ real entity instance
```

es peligroso porque referencias existentes continuarían apuntando al proxy.

Por tanto, se prefiere:

```text
initialize same proxy instance
```

cuando la estrategia lo permita.

---

# 106. EntityReference integration

`EntityReference` representa:

```text
EntityType + Identifier + IdentityNamespace
```

sin implicar una entidad cargada.

---

# 107. Reference resolution

```text
EntityReference
    ↓
EntityKey
    ↓
IdentityMap
    ├── HIT → entity
    └── MISS → optional loader
```

---

# 108. IdentityMap no resuelve referencias por I/O

La decisión de cargar pertenece a:

```text
EntityReferenceResolver
EntityLoader
Lazy Loading System
```

---

# 109. Reference ≠ IdentityMap entry

Puede existir:

```text
EntityReference(User#10)
```

sin que:

```text
User#10
```

esté en Identity Map.

---

# 110. Missing entity

Si una referencia se resuelve contra DB y no existe:

```text
MISSING
```

no deberá confundirse con:

```text
IdentityMap MISS
```

---

# 111. MISS semantics

```text
IdentityMap MISS
```

solo significa:

> Esta identidad no tiene actualmente una instancia canónica registrada en este PersistenceContext.

---

# 112. Inheritance

Supongamos:

```text
Person
 ├── Employee
 └── Customer
```

La identidad dependerá del mapping de herencia.

---

# 113. PHP inheritance ≠ ORM identity hierarchy

El hecho de que:

```php
Employee extends Person
```

no decide automáticamente si:

```text
Person#10
Employee#10
```

son el mismo `EntityKey`.

---

# 114. Root identity type

Para estrategias donde toda la jerarquía comparte identidad:

```text
IdentityRootType = Person
```

podrá utilizarse para construir la key.

---

# 115. EffectiveEntityType

Se propone distinguir:

```text
DeclaredEntityType
ConcreteEntityType
IdentityRootType
```

---

# 116. EntityKey type resolution

La metadata deberá proporcionar:

```text
IdentityEntityType
```

apropiado para la estrategia.

Identity Map no inferirá herencia por sí mismo.

---

# 117. Polymorphic query

Una query:

```text
find(Person, 10)
```

que materialice:

```text
Employee#10
```

deberá registrar una identidad coherente con la estrategia de mapping.

---

# 118. Identity collision in hierarchy

No deberán coexistir:

```text
Person#10 instance A
Employee#10 instance B
```

si metadata define que ambos representan la misma identidad persistente.

---

# 119. IdentityMap and Repository

Repository deberá compartir el mismo Identity Map del EntityManager.

---

# 120. No repository-local map

Incorrecto:

```text
UserRepository → UserIdentityMap
OrderRepository → OrderIdentityMap
```

si cada repository puede crear instancias duplicadas.

---

# 121. PersistenceContext-wide identity

La identidad deberá coordinarse en:

```text
EntityManager / PersistenceContext
```

no por repository.

---

# 122. Multiple repositories

```php
$userA = $userRepository->find(10);
$userB = $adminRepository->findUser(10);
```

dentro del mismo EntityManager deberán converger en:

```php
$userA === $userB;
```

cuando ambas consultas representan la misma identidad.

---

# 123. Entity Query integration

Entity Query deberá hidratar resultados mediante el mismo Identity Map.

---

# 124. Query A / Query B

```text
Query A → User#10
Query B → User#10
```

deberán converger a la misma instancia managed.

---

# 125. Projection queries

Una proyección:

```text
SELECT user.id, user.name
→ UserSummary DTO
```

no deberá registrarse en Identity Map como `User`.

---

# 126. Scalar results

Tampoco:

```text
COUNT(*)
SUM(...)
scalar tuple
```

---

# 127. Partial entity

Si VoltStack permite partial entities, Identity Map deberá manejar el riesgo de:

```text
same EntityKey
+
different loaded field sets
```

---

# 128. Partial load conflict

Caso:

```text
Query A → User#10 {id,name}
Query B → User#10 {id,email}
```

No podrán producir dos instancias.

---

# 129. Loaded field state

La información sobre:

```text
which fields are loaded
```

no pertenece conceptualmente al `EntityKey`.

Podrá pertenecer al:

```text
Entity State / Hydration State
```

---

# 130. Merge partial hydration

La instancia canónica podrá enriquecerse:

```text
{id,name}
+
{id,email}
→
{id,name,email}
```

si la estrategia lo permite.

---

# 131. Dirty-field protection

Hydration adicional nunca deberá sobrescribir silenciosamente un campo local `DIRTY` con un valor DB sin política explícita.

---

# 132. Merge policy

Se requerirá coordinación con:

```text
Entity State
Change Tracking
Hydration System
```

---

# 133. UnitOfWork integration

Identity Map y UnitOfWork deberán compartir la misma noción de instancia canónica.

---

# 134. Managed registration

Conceptualmente:

```text
Hydrate Entity
   ↓
IdentityMap.register()
   ↓
UnitOfWork.registerManaged()
```

como operación coordinada.

---

# 135. Atomic ORM registration

No deberá quedar:

```text
IdentityMap says managed
UnitOfWork knows nothing
```

por fallas intermedias.

---

# 136. PersistenceContext coordinator

Puede existir:

```text
PersistenceContextRegistry
```

que coordine:

```text
IdentityMap
UnitOfWork
EntityStateRegistry
Snapshots
```

sin fusionarlos conceptualmente.

---

# 137. IdentityMap remains focused

Aunque exista coordinación:

```text
IdentityMap
```

seguirá siendo responsable únicamente de identidad canónica.

---

# 138. detach()

```php
$entityManager->detach($user);
```

deberá retirar su asociación del Identity Map.

---

# 139. Detach flow

```text
Entity
 ↓
resolve EntityKey
 ↓
IdentityMap.remove()
 ↓
UnitOfWork detach
 ↓
snapshot cleanup
 ↓
DETACHED
```

La coordinación exacta pertenece al EntityManager/PersistenceContext.

---

# 140. Detached instance

Después:

```text
User#10 object A → DETACHED
```

una nueva carga puede producir:

```text
User#10 object B → MANAGED
```

---

# 141. Detached A ≠ canonical instance

Entonces:

```text
object A
```

ya no pertenece al Identity Map.

---

# 142. Re-persist detached instance

Si se intenta volver a asociar `A` cuando `B` ya es canónica:

```text
identity conflict
```

deberá resolverse explícitamente.

---

# 143. No silent replacement

Nunca reemplazar `B` por `A`.

---

# 144. merge()

Si VoltStack llegara a soportar:

```text
merge(detachedEntity)
```

deberá copiar/reconciliar state hacia la instancia managed canónica.

No deberá sustituir la instancia.

---

# 145. Recomendación V1

Evitar una API `merge()` mágica.

Preferir:

```text
load managed entity
→ apply explicit changes
```

---

# 146. clear()

```php
$entityManager->clear();
```

deberá vaciar Identity Map del contexto correspondiente.

---

# 147. clear() semantics

```text
IdentityMap count → 0
```

pero no significa:

```text
close database connection
clear metadata cache
destroy global compiled mappings
```

---

# 148. clear(EntityType)

Podrá soportarse:

```php
$entityManager->clear(User::class);
```

si existe necesidad.

---

# 149. Type-specific clear

Debe respetar inheritance identity roots y relaciones con UnitOfWork.

No deberá implementarse como simple:

```text
unset all keys whose class string == User
```

---

# 150. Remove after DELETE

Después de un DELETE confirmado, la entidad deberá retirarse del Identity Map según la política de Entity State.

---

# 151. Timing

El momento exacto deberá coordinarse con:

```text
Persistence Engine
UnitOfWork
Transaction semantics
```

---

# 152. flush() ≠ clear()

Una entidad actualizada exitosamente permanece normalmente:

```text
MANAGED
+
IdentityMap registered
```

después de flush.

---

# 153. Transaction rollback

Supongamos:

```text
NEW User
→ INSERT
→ generated ID 42
→ transaction ROLLBACK
```

La reconciliación no puede reducirse a:

```text
remove key 42
```

---

# 154. Rollback identity problem

La entidad puede conservar:

```text
id = 42
```

aunque la fila ya no exista.

---

# 155. Identity Map responsibility boundary

Identity Map ejecutará las operaciones de registro/remoción solicitadas por el coordinador.

No decidirá por sí mismo:

```text
what rollback means for entity state
```

---

# 156. Transaction rollback ≠ identity rollback

Regla:

```text
Transaction Rollback
≠
automatic Entity Identifier rollback
```

---

# 157. Database-generated IDs after rollback

Algunos motores no reutilizan secuencias/auto-increment de forma trivial.

Por tanto:

```text
id 42 assigned
transaction rolled back
```

no implica que pueda volver a tratarse como:

```text
id = null
```

---

# 158. EntityManager taint

Ciertos escenarios de rollback/unknown outcome podrán requerir:

```text
EntityManager → TAINTED
```

y:

```text
clear()/close()/reconciliation
```

---

# 159. UNKNOWN transaction outcome

Identity Map no deberá afirmar sincronización cuando:

```text
commit outcome = UNKNOWN
```

---

# 160. Memory management

Identity Map mantiene referencias a objetos.

Por tanto puede aumentar memoria en:

```text
large imports
streaming queries
long-running jobs
```

---

# 161. Strong references

Implementación tradicional:

```text
EntityKey → strong object reference
```

garantiza identidad mientras la entidad permanezca managed.

---

# 162. WeakReference temptation

PHP `WeakReference` podría parecer útil para reducir memoria.

Pero puede romper:

```text
managed identity guarantees
UnitOfWork consistency
relationship references
snapshot tracking
```

---

# 163. Default policy

Identity Map deberá usar:

```text
strong references
```

para entidades managed.

---

# 164. Weak identity mode

No deberá introducirse en V1 como optimización general.

Si alguna estrategia futura lo usa:

```text
read-only
detached-after-yield
```

deberá ser una política distinta, no el Identity Map managed normal.

---

# 165. Streaming

El ORM definió posibles modos:

```text
TRACKED
DETACHED_AFTER_YIELD
READ_ONLY
SCALAR
PROJECTION
```

---

# 166. TRACKED streaming

Cada entidad cargada permanece en Identity Map.

Resultado:

```text
memory grows with processed entities
```

---

# 167. DETACHED_AFTER_YIELD

Flujo:

```text
hydrate
→ yield entity
→ consumer advances
→ detach previous entity
```

según política.

---

# 168. Identity guarantee scope

En `DETACHED_AFTER_YIELD`:

```text
same EntityKey later
```

podrá producir una nueva instancia porque la anterior ya no está managed.

Eso no viola Identity Map.

---

# 169. READ_ONLY mode

Puede seguir utilizando Identity Map dentro de un batch/segmento para deduplicar joins.

Pero podrá aplicar políticas de liberación más agresivas.

---

# 170. Batch processing

Recomendación:

```php
foreach ($repository->chunkById(500) as $batch) {
    process($batch);
    $entityManager->clear();
}
```

cuando el caso de uso lo permita.

---

# 171. Long-running jobs

Nunca mantener millones de entidades en un único Identity Map indefinidamente.

---

# 172. Resource governance

Podrán existir límites:

```text
maxManagedEntities
warningThreshold
hardLimit
```

---

# 173. Hard limit

Si Identity Map excede un límite de seguridad:

```text
ManagedEntityLimitExceededException
```

podrá evitar agotamiento de memoria.

---

# 174. Limit ≠ automatic clear

VoltStack no deberá hacer:

```text
IdentityMap full
→ silently clear()
```

porque rompería semántica.

---

# 175. Memory telemetry

Métricas:

```text
orm.identity_map.size
orm.identity_map.hits
orm.identity_map.misses
orm.identity_map.registrations
orm.identity_map.removals
orm.identity_map.conflicts
orm.identity_map.peak_size
```

---

# 176. Hit ratio

Puede calcularse:

```text
HitRatio
=
Hits / (Hits + Misses)
```

cuando el denominador sea mayor que cero.

---

# 177. Hit ratio interpretation

No será automáticamente una métrica de rendimiento buena/mala.

Una aplicación puede legítimamente cargar muchas entidades únicas.

---

# 178. Debugging

Debug Toolbar podrá mostrar:

```text
PersistenceContext
Managed entities: 1,248

User          350
Order         420
OrderItem     478

Identity hits:   3,420
Identity misses: 1,248
Conflicts:       0
```

---

# 179. Entity inspection

Para debugging podrá exponerse:

```text
EntityType
Canonical identifier
Entity State
Object ID
```

pero Entity State provendrá del sistema correspondiente.

---

# 180. Sensitive IDs

Identifiers sensibles podrán:

```text
redact
hash
truncate
```

según diagnostics policy.

---

# 181. Persistent runtime

FrankenPHP hace crítica la separación:

```text
Worker Lifetime
≠
PersistenceContext Lifetime
```

---

# 182. Arquitectura FrankenPHP

```text
FrankenPHP Worker
│
├── Compiled Entity Metadata
├── Identifier Type Definitions
├── EntityType Registry
│
├── Request A
│   ├── EntityManager A
│   ├── IdentityMap A
│   └── UnitOfWork A
│
├── RESET
│
└── Request B
    ├── EntityManager B
    ├── IdentityMap B
    └── UnitOfWork B
```

---

# 183. Nunca process-global

Esto será una violación crítica:

```text
Worker
└── Global IdentityMap
```

---

# 184. Request termination

Al finalizar la operación:

```text
IdentityMap
→ clear/dispose
```

aunque el EntityManager quede destinado a garbage collection.

---

# 185. Deterministic reset

VoltStack deberá poder verificar:

```text
IdentityMap.count() == 0
```

después del reset del PersistenceContext.

---

# 186. RoadRunner

La misma regla:

```text
worker reuse
≠
IdentityMap reuse
```

---

# 187. OpenSwoole

Más crítico todavía cuando existan coroutines:

```text
Coroutine A IdentityMap
≠
Coroutine B IdentityMap
```

---

# 188. EntityManager concurrency

Un EntityManager/IdentityMap mutable no deberá considerarse thread-safe/coroutine-safe para uso concurrente.

---

# 189. One logical owner

Cada Identity Map tendrá:

```text
one logical PersistenceContext owner
```

---

# 190. No concurrent mutation

No permitir:

```text
Coroutine A register(User#10)
Coroutine B register(User#10)
```

sobre el mismo mutable context sin un modelo explícito de serialización.

---

# 191. Async operations

Si una operación async necesita ORM:

```text
new logical ORM scope
```

será preferible a compartir EntityManager mutable.

---

# 192. Multitenancy isolation

Supongamos:

```text
Tenant A User#10
Tenant B User#10
```

Deben producir:

```text
EntityKey(namespace=A, User, 10)
EntityKey(namespace=B, User, 10)
```

---

# 193. Tenant switch

No deberá permitirse:

```text
same EntityManager
Tenant A → Tenant B
```

mientras Identity Map contiene entidades de A.

---

# 194. Tenant switch policy

Preferir:

```text
new DatabaseContext
→ new EntityManager
→ new IdentityMap
```

---

# 195. Cross-context entity

Pasar una entidad managed de Tenant A a PersistenceContext B deberá detectarse.

---

# 196. CrossContextEntityException

Debe evitar:

```text
entity from namespace A
registered in namespace B
```

---

# 197. Sharding isolation

Misma regla para:

```text
shards
regional databases
independent partitions
```

cuando representen espacios de identidad distintos.

---

# 198. Read/write routing

Identity Map deberá ocultar diferencias físicas de:

```text
primary
replica
```

si ambas conexiones representan el mismo espacio lógico.

---

# 199. Sticky reads

Después de write:

```text
IdentityMap
```

puede devolver directamente la instancia managed.

No necesita ir a replica.

---

# 200. Replica lag

Una query explícita que lea replica no deberá sobrescribir silenciosamente estado managed más nuevo con datos retrasados.

---

# 201. Reconciliation policy

Hydration contra una instancia ya managed deberá considerar:

```text
local dirty state
read source
transaction context
refresh intent
```

antes de sobrescribir valores.

Identity Map solo proporciona la instancia.

---

# 202. Cache integration

Second-level entity cache puede alimentar Hydration:

```text
Entity Cache
    ↓
Cached State
    ↓
Hydrator
    ↓
IdentityMap
```

---

# 203. Cache HIT ≠ IdentityMap HIT

Son niveles diferentes.

---

# 204. Cache invalidation

Eliminar una entrada del second-level cache no elimina automáticamente la entidad managed del Identity Map.

---

# 205. IdentityMap precedence

Dentro del mismo PersistenceContext, una entidad managed normalmente tendrá precedencia sobre una copia de second-level cache.

---

# 206. Serialization

Identity Map nunca deberá serializarse como parte de:

```text
session
queue payload
cache
worker state
```

---

# 207. Entity serialization

Serializar una entidad no serializará automáticamente su Identity Map/PersistenceContext.

---

# 208. Queue boundary

Una entidad no deberá enviarse como objeto managed con infraestructura ORM adjunta.

Preferir:

```text
EntityReference
```

o:

```text
EntityType + Identifier
```

---

# 209. Rehydration in job

```text
Job payload
→ EntityReference
→ new job ORM scope
→ IdentityMap lookup/load
```

---

# 210. Security

Identity Map no implementa:

```text
authorization
row-level access policy
tenant authorization
```

---

# 211. Identity HIT ≠ access granted

Encontrar:

```text
User#10
```

en Identity Map no autoriza al caller a leerlo.

---

# 212. Authorization integration

Authorization deberá ocurrir en las capas apropiadas.

No se omitirá solo porque no hubo DB query.

---

# 213. Critical security implication

Un sistema mal diseñado podría hacer:

```text
request 1 loads secret entity
global IdentityMap retains it
request 2 finds same ID
```

Esto sería una fuga crítica.

Por eso runtime isolation es una invariante de seguridad, no solo de memoria.

---

# 214. Testing strategy

Se requieren:

```text
Unit Tests
Hydration Integration Tests
UnitOfWork Integration Tests
Transaction Tests
Inheritance Tests
Proxy Tests
Multitenancy Tests
Runtime Isolation Tests
Streaming Tests
Memory Tests
Concurrency Misuse Tests
```

---

# 215. Test: canonical instance

```php
$a = $em->find(User::class, 10);
$b = $em->find(User::class, 10);

assert($a === $b);
```

---

# 216. Test: different types

```text
User#10
Invoice#10
```

deben ser distintas keys.

---

# 217. Test: different namespaces

```text
Tenant A User#10
Tenant B User#10
```

deben ser distintas keys.

---

# 218. Test: canonical identifier

Si metadata permite normalización:

```text
find(User, "10")
find(User, 10)
```

deberán converger al mismo EntityKey.

---

# 219. Test: invalid coercion

Un valor no canonicalizable deberá producir:

```text
InvalidEntityIdentifierException
```

y no IdentityMap MISS.

---

# 220. Test: zero ID

```text
0
```

no deberá tratarse automáticamente como ausencia de identidad.

---

# 221. Test: composite ID

```text
(order=10,line=2)
```

deberá ser estable independientemente de la estructura accidental del input una vez canonicalizado.

---

# 222. Test: conflict

```text
register(User#10, A)
register(User#10, B)
```

con:

```php
A !== B
```

deberá fallar.

---

# 223. Test: idempotent registration

```text
register(User#10, A)
register(User#10, A)
```

deberá ser seguro.

---

# 224. Test: generated ID

```text
NEW entity without ID
→ INSERT
→ generated ID
→ establish EntityKey
→ map contains entity
```

---

# 225. Test: identifier mutation

```text
managed User#10
→ ID becomes 11
```

deberá ser detectado y rechazado.

---

# 226. Test: JOIN deduplication

Tres rows de `User#10` deberán producir una sola instancia.

---

# 227. Test: cyclic hydration

```text
A → B → A
```

no deberá crear:

```text
A1
A2
```

---

# 228. Test: hydration failure

Una entrada `INITIALIZING` deberá limpiarse si hydration falla.

---

# 229. Test: proxy

```text
UserProxy#10
```

y:

```text
User#10
```

deberán compartir EntityKey.

---

# 230. Test: repository convergence

Dos repositories dentro del mismo EntityManager deberán devolver la misma instancia para la misma identidad.

---

# 231. Test: entity query convergence

Dos Entity Queries deberán reutilizar la misma instancia.

---

# 232. Test: projection

Un DTO con `id=10` no deberá registrarse como `User#10`.

---

# 233. Test: detach

```text
load User#10 → A
detach A
load User#10 → B
```

podrá resultar:

```php
A !== B;
```

---

# 234. Test: clear

Después de:

```php
$em->clear();
```

Identity Map deberá estar vacío.

---

# 235. Test: flush

Después de UPDATE + flush, la entidad deberá seguir siendo la instancia canónica.

---

# 236. Test: rollback

Rollback no deberá realizar suposiciones inválidas sobre generated IDs.

---

# 237. Test: UNKNOWN outcome

Identity Map no deberá ocultar uncertainty del Persistence System.

---

# 238. Test: streaming tracked

El tamaño deberá crecer de forma esperada.

---

# 239. Test: detached-after-yield

Las entidades deberán liberarse del managed context según política.

---

# 240. Test: resource limit

Superar el hard limit deberá producir error explícito, nunca silent clear.

---

# 241. Test: FrankenPHP isolation

```text
Request A:
load User#10 → A

reset

Request B:
load User#10 → B
```

deberá garantizar:

```php
A !== B;
```

---

# 242. Test: tenant isolation

Tenant B nunca deberá recibir una instancia registrada bajo Tenant A.

---

# 243. Test: concurrent misuse

Compartir un IdentityMap mutable entre dos scopes concurrentes deberá detectarse o quedar explícitamente fuera del contrato soportado.

---

# 244. Error hierarchy

```text
DatabaseOrmException
└── IdentityMapException
    ├── InvalidEntityKeyException
    ├── InvalidEntityIdentifierException
    ├── EntityIdentityConflictException
    ├── EntityIdentifierMutationException
    ├── EntityIdentityUnavailableException
    ├── EntityIdentityUnknownException
    ├── EntityIdentityNamespaceException
    ├── CrossContextEntityException
    ├── IdentityMapRegistrationException
    ├── IdentityMapRemovalException
    ├── IdentityReservationException
    ├── IdentityInitializationException
    ├── IdentityMapCorruptionException
    ├── ManagedEntityLimitExceededException
    ├── IdentityMapConcurrencyException
    ├── IdentityMapRuntimeIsolationException
    └── IdentityMapInvariantException
```

---

# 245. Conflict diagnostic

```text
DB-ORM-IDENTITY-MAP-CONFLICT-001

Entity Type:
App\Entity\User

Identifier:
int:10

Identity Namespace:
tenant:acme

Existing Object:
object#183

Incoming Object:
object#241

Persistence Context:
01J...

Reason:
Two different object instances attempted to represent the same
EntityKey inside one PersistenceContext.

Action:
Reuse the existing managed entity, detach it explicitly, or
reconcile detached state through a supported persistence workflow.
```

---

# 246. Cross-context diagnostic

```text
DB-ORM-IDENTITY-MAP-CONTEXT-002

Entity:
App\Entity\User

Identifier:
10

Entity Namespace:
tenant:acme

Target Persistence Context:
tenant:globex

Reason:
An entity associated with one logical identity namespace cannot be
registered inside another persistence context.

Action:
Load the entity independently in the target context or use an
explicit cross-context transfer mechanism.
```

---

# 247. Memory diagnostic

```text
DB-ORM-IDENTITY-MAP-MEMORY-003

Managed Entities:
100,001

Configured Hard Limit:
100,000

Persistence Context:
01J...

Action:
Process entities in chunks and clear/detach managed entities at
safe boundaries.
```

---

# 248. Arquitectura de directorios

```text
src/Quantum/Database/ORM/
│
├── Entity/
│   ├── EntityType.php
│   ├── EntityKey.php
│   ├── EntityIdentifier.php
│   └── Identity/
│       ├── SimpleEntityIdentifier.php
│       ├── CompositeEntityIdentifier.php
│       ├── EntityIdentifierPart.php
│       ├── EntityIdentityNamespace.php
│       ├── EntityKeyHash.php
│       └── EntityIdentityCertainty.php
│
├── Identity/
│   ├── Contract/
│   │   ├── IdentityMap.php
│   │   ├── IdentityMapFactory.php
│   │   └── EntityKeyFactory.php
│   │
│   ├── Map/
│   │   ├── DefaultIdentityMap.php
│   │   ├── IdentityMapEntry.php
│   │   ├── IdentityMapEntryState.php
│   │   └── IdentityReservation.php
│   │
│   ├── Key/
│   │   ├── DefaultEntityKeyFactory.php
│   │   ├── EntityIdentifierCanonicalizer.php
│   │   ├── EntityIdentityNamespaceResolver.php
│   │   └── EntityIdentityTypeResolver.php
│   │
│   ├── Registration/
│   │   ├── EntityIdentityRegistrar.php
│   │   ├── EntityIdentityEstablisher.php
│   │   └── EntityIdentityConflictDetector.php
│   │
│   ├── Runtime/
│   │   ├── IdentityMapScope.php
│   │   ├── IdentityMapRuntimeResetter.php
│   │   └── IdentityMapOwnershipGuard.php
│   │
│   ├── Governance/
│   │   ├── IdentityMapResourcePolicy.php
│   │   └── ManagedEntityLimit.php
│   │
│   ├── Telemetry/
│   │   ├── IdentityMapTelemetry.php
│   │   └── IdentityMapProfiler.php
│   │
│   └── Exception/
│       └── ...
```

---

# 249. Dependency rules

Permitido:

```text
Identity Map
    ↓
Entity Model
    ↓
Compiled Metadata / Identifier Type Information
```

Coordinación:

```text
EntityManager
    ├── IdentityMap
    ├── UnitOfWork
    ├── EntityState
    └── Hydration
```

---

# 250. Dependencias prohibidas

```text
IdentityMap → SQL Compiler
IdentityMap → Query Executor
IdentityMap → Driver
IdentityMap → Migration
IdentityMap → HTTP
IdentityMap → Global Tenant State
```

---

# 251. Identity Map no carga

Nunca:

```php
public function get(EntityKey $key): ?object
{
    if (!isset($this->entities[$key])) {
        return $this->database->query(...);
    }
}
```

---

# 252. Identity Map no refresca

Tampoco:

```text
get()
→ check DB for freshness
```

---

# 253. Identity Map no autoriza

Tampoco:

```text
contains()
→ authorization passed
```

---

# 254. Identity Map no detecta cambios

Tampoco:

```text
register()
→ take snapshot
→ calculate dirty fields
```

Estas responsabilidades pertenecen a sistemas posteriores.

---

# 255. Invariantes arquitectónicas

## DB-ORM-IDENTITY-MAP-001
Cada Identity Map pertenecerá a un PersistenceContext lógico.

## DB-ORM-IDENTITY-MAP-002
Un EntityKey tendrá como máximo una instancia canónica managed por PersistenceContext.

## DB-ORM-IDENTITY-MAP-003
IdentityMap será distinto de UnitOfWork.

## DB-ORM-IDENTITY-MAP-004
IdentityMap será distinto de Entity State.

## DB-ORM-IDENTITY-MAP-005
IdentityMap será distinto de Entity Cache.

## DB-ORM-IDENTITY-MAP-006
IdentityMap será distinto de Query Cache.

## DB-ORM-IDENTITY-MAP-007
IdentityMap será distinto de Result Cache.

## DB-ORM-IDENTITY-MAP-008
IdentityMap membership no probará existencia actual en DB.

## DB-ORM-IDENTITY-MAP-009
IdentityMap membership no probará estado sincronizado.

## DB-ORM-IDENTITY-MAP-010
IdentityMap no será process-global.

## DB-ORM-IDENTITY-MAP-011
EntityKey incluirá EntityType.

## DB-ORM-IDENTITY-MAP-012
EntityKey incluirá canonical identifier.

## DB-ORM-IDENTITY-MAP-013
EntityKey incluirá identity namespace cuando el espacio lógico lo requiera.

## DB-ORM-IDENTITY-MAP-014
Identity namespace no será una conexión física.

## DB-ORM-IDENTITY-MAP-015
Tenant identity no estará hardcoded en ORM core.

## DB-ORM-IDENTITY-MAP-016
Shard identity no estará hardcoded en ORM core.

## DB-ORM-IDENTITY-MAP-017
Primary y replica podrán compartir namespace cuando representen el mismo espacio lógico.

## DB-ORM-IDENTITY-MAP-018
Identifier equality será typed.

## DB-ORM-IDENTITY-MAP-019
Identifier canonicalization pertenecerá al sistema de identidad/tipos.

## DB-ORM-IDENTITY-MAP-020
IdentityMap no aplicará PHP loose equality.

## DB-ORM-IDENTITY-MAP-021
Zero no significará automáticamente identifier unavailable.

## DB-ORM-IDENTITY-MAP-022
Empty string no significará automáticamente identifier unavailable.

## DB-ORM-IDENTITY-MAP-023
Float identifiers no serán identificadores persistentes estándar.

## DB-ORM-IDENTITY-MAP-024
Composite identifiers tendrán representación estructurada.

## DB-ORM-IDENTITY-MAP-025
Composite identifiers no usarán concatenación ambigua.

## DB-ORM-IDENTITY-MAP-026
Composite identifier ordering será metadata-driven.

## DB-ORM-IDENTITY-MAP-027
EntityKeyHash será distinto de EntityKey.

## DB-ORM-IDENTITY-MAP-028
Hash collisions no producirán identity aliasing.

## DB-ORM-IDENTITY-MAP-029
IdentityMap lookup no realizará I/O.

## DB-ORM-IDENTITY-MAP-030
IdentityMap lookup no ejecutará queries.

## DB-ORM-IDENTITY-MAP-031
IdentityMap lookup será memory-only.

## DB-ORM-IDENTITY-MAP-032
find() podrá consultar IdentityMap antes de DB.

## DB-ORM-IDENTITY-MAP-033
IdentityMap HIT devolverá la instancia canónica.

## DB-ORM-IDENTITY-MAP-034
IdentityMap HIT no implicará latest database state.

## DB-ORM-IDENTITY-MAP-035
Refresh deberá preservar object identity cuando sea posible.

## DB-ORM-IDENTITY-MAP-036
Registrar la misma key y misma instancia será idempotente.

## DB-ORM-IDENTITY-MAP-037
Registrar la misma key con distinta instancia será conflicto.

## DB-ORM-IDENTITY-MAP-038
Identity conflicts no usarán last-wins.

## DB-ORM-IDENTITY-MAP-039
Identity conflict será error explícito.

## DB-ORM-IDENTITY-MAP-040
IdentityMap podrá mantener reverse lookup.

## DB-ORM-IDENTITY-MAP-041
Object identity será distinta de entity identity.

## DB-ORM-IDENTITY-MAP-042
spl_object_id no será persistent identifier.

## DB-ORM-IDENTITY-MAP-043
IdentityMap entries no contendrán ChangeSets.

## DB-ORM-IDENTITY-MAP-044
IdentityMap entries no contendrán snapshots como responsabilidad propia.

## DB-ORM-IDENTITY-MAP-045
IdentityMap no calculará dirty state.

## DB-ORM-IDENTITY-MAP-046
IdentityMap no ejecutará persistence operations.

## DB-ORM-IDENTITY-MAP-047
Una persistent EntityKey requerirá identifier suficiente.

## DB-ORM-IDENTITY-MAP-048
Assigned identifier no implicará persisted entity.

## DB-ORM-IDENTITY-MAP-049
IdentityMap membership no implicará row existence.

## DB-ORM-IDENTITY-MAP-050
NEW assigned-ID entity podrá ser registrada sin ser considerada persisted.

## DB-ORM-IDENTITY-MAP-051
Database-generated identity podrá no existir antes del INSERT.

## DB-ORM-IDENTITY-MAP-052
Temporary UoW identity será distinta de persistent identity.

## DB-ORM-IDENTITY-MAP-053
TemporaryEntityToken no será EntityKey persistente.

## DB-ORM-IDENTITY-MAP-054
Temporary identity no se expondrá como entity ID.

## DB-ORM-IDENTITY-MAP-055
Generated identifier será canonicalizado antes de registro.

## DB-ORM-IDENTITY-MAP-056
Generated identifier establishment requerirá suficiente execution certainty.

## DB-ORM-IDENTITY-MAP-057
UNKNOWN insert outcome no se convertirá en persisted certainty.

## DB-ORM-IDENTITY-MAP-058
Known identifier value y known persistence outcome serán dimensiones separadas.

## DB-ORM-IDENTITY-MAP-059
Initial identity establishment será distinto de identifier mutation.

## DB-ORM-IDENTITY-MAP-060
Managed established identifier será immutable por defecto.

## DB-ORM-IDENTITY-MAP-061
Identifier mutation será rechazada en V1.

## DB-ORM-IDENTITY-MAP-062
Database support for PK UPDATE no obligará al ORM a soportarlo.

## DB-ORM-IDENTITY-MAP-063
Natural identifier usado como identity deberá tratarse como immutable.

## DB-ORM-IDENTITY-MAP-064
Hydration consultará IdentityMap para deduplicación.

## DB-ORM-IDENTITY-MAP-065
JOIN duplicate rows no crearán duplicate entities.

## DB-ORM-IDENTITY-MAP-066
Cyclic hydration no creará duplicate canonical instances.

## DB-ORM-IDENTITY-MAP-067
Early hydration registration podrá utilizar estado interno INITIALIZING.

## DB-ORM-IDENTITY-MAP-068
INITIALIZING no será Entity State público.

## DB-ORM-IDENTITY-MAP-069
Incomplete entity no escapará arbitrariamente a application code.

## DB-ORM-IDENTITY-MAP-070
Hydration failure limpiará identity reservation incompleta.

## DB-ORM-IDENTITY-MAP-071
Identity reservation será distinta de managed state.

## DB-ORM-IDENTITY-MAP-072
Proxy class no será canonical EntityType.

## DB-ORM-IDENTITY-MAP-073
Proxy y entity target compartirán EntityKey.

## DB-ORM-IDENTITY-MAP-074
Proxy initialization no creará segunda instancia canónica.

## DB-ORM-IDENTITY-MAP-075
Proxy replacement será evitado cuando rompa referencias existentes.

## DB-ORM-IDENTITY-MAP-076
EntityReference será distinta de IdentityMap entry.

## DB-ORM-IDENTITY-MAP-077
EntityReference podrá existir sin entidad cargada.

## DB-ORM-IDENTITY-MAP-078
IdentityMap no hará I/O para resolver EntityReference.

## DB-ORM-IDENTITY-MAP-079
IdentityMap MISS será distinto de entity missing in DB.

## DB-ORM-IDENTITY-MAP-080
Inheritance identity será metadata-driven.

## DB-ORM-IDENTITY-MAP-081
PHP inheritance no decidirá EntityKey automáticamente.

## DB-ORM-IDENTITY-MAP-082
Proxy subclass no creará nueva entity identity.

## DB-ORM-IDENTITY-MAP-083
Identity root type podrá diferir de concrete type.

## DB-ORM-IDENTITY-MAP-084
IdentityMap no inferirá inheritance strategy.

## DB-ORM-IDENTITY-MAP-085
Repositories compartirán IdentityMap del PersistenceContext.

## DB-ORM-IDENTITY-MAP-086
No existirán repository-local identity maps independientes para la misma context identity.

## DB-ORM-IDENTITY-MAP-087
Entity Queries compartirán IdentityMap.

## DB-ORM-IDENTITY-MAP-088
Múltiples entity queries convergerán en la instancia canónica.

## DB-ORM-IDENTITY-MAP-089
DTO projections no entrarán al IdentityMap como entities.

## DB-ORM-IDENTITY-MAP-090
Scalar results no entrarán al IdentityMap.

## DB-ORM-IDENTITY-MAP-091
Partial entities no crearán instancias duplicadas.

## DB-ORM-IDENTITY-MAP-092
Loaded-field state no formará parte de EntityKey.

## DB-ORM-IDENTITY-MAP-093
Partial hydration merge no sobrescribirá dirty state silenciosamente.

## DB-ORM-IDENTITY-MAP-094
IdentityMap y UnitOfWork compartirán canonical instance semantics.

## DB-ORM-IDENTITY-MAP-095
ORM registration será coordinada para evitar estado inconsistente entre subsistemas.

## DB-ORM-IDENTITY-MAP-096
IdentityMap seguirá siendo conceptualmente independiente aunque exista PersistenceContext coordinator.

## DB-ORM-IDENTITY-MAP-097
detach removerá la instancia de IdentityMap.

## DB-ORM-IDENTITY-MAP-098
Detached entity dejará de ser canonical managed instance.

## DB-ORM-IDENTITY-MAP-099
Una carga posterior a detach podrá crear nueva instancia.

## DB-ORM-IDENTITY-MAP-100
Reattaching detached instance no reemplazará silently una managed instance existente.

## DB-ORM-IDENTITY-MAP-101
Magic merge no será requisito de V1.

## DB-ORM-IDENTITY-MAP-102
clear() vaciará managed identity state del contexto.

## DB-ORM-IDENTITY-MAP-103
clear() no borrará compiled metadata.

## DB-ORM-IDENTITY-MAP-104
clear() no cerrará conexiones por definición.

## DB-ORM-IDENTITY-MAP-105
Type-specific clear respetará inheritance semantics.

## DB-ORM-IDENTITY-MAP-106
Successful UPDATE flush normalmente conservará IdentityMap entry.

## DB-ORM-IDENTITY-MAP-107
flush() será distinto de clear().

## DB-ORM-IDENTITY-MAP-108
Transaction rollback será distinto de identity rollback.

## DB-ORM-IDENTITY-MAP-109
Generated identifier no será automáticamente nulificado por rollback.

## DB-ORM-IDENTITY-MAP-110
IdentityMap no decidirá Entity State rollback semantics.

## DB-ORM-IDENTITY-MAP-111
UNKNOWN transaction outcome preservará uncertainty.

## DB-ORM-IDENTITY-MAP-112
Certain rollback/unknown scenarios podrán taint EntityManager.

## DB-ORM-IDENTITY-MAP-113
Managed entities tendrán strong references por defecto.

## DB-ORM-IDENTITY-MAP-114
WeakReference no será default managed identity strategy.

## DB-ORM-IDENTITY-MAP-115
Weak references no podrán debilitar UnitOfWork invariants.

## DB-ORM-IDENTITY-MAP-116
TRACKED streaming podrá aumentar IdentityMap size.

## DB-ORM-IDENTITY-MAP-117
DETACHED_AFTER_YIELD podrá liberar canonical managed instances.

## DB-ORM-IDENTITY-MAP-118
Nueva instancia tras detach no violará IdentityMap invariants.

## DB-ORM-IDENTITY-MAP-119
Long-running jobs deberán poder liberar managed entities explícitamente.

## DB-ORM-IDENTITY-MAP-120
Resource limits nunca provocarán silent clear.

## DB-ORM-IDENTITY-MAP-121
Managed entity hard limit será error explícito.

## DB-ORM-IDENTITY-MAP-122
IdentityMap telemetry no modificará semantics.

## DB-ORM-IDENTITY-MAP-123
Hit ratio no se interpretará automáticamente como quality score.

## DB-ORM-IDENTITY-MAP-124
Sensitive identifiers podrán redactarse en diagnostics.

## DB-ORM-IDENTITY-MAP-125
Worker lifetime será distinto de PersistenceContext lifetime.

## DB-ORM-IDENTITY-MAP-126
FrankenPHP requests no compartirán IdentityMap mutable.

## DB-ORM-IDENTITY-MAP-127
RoadRunner requests no compartirán IdentityMap mutable.

## DB-ORM-IDENTITY-MAP-128
OpenSwoole coroutines no compartirán IdentityMap mutable accidentalmente.

## DB-ORM-IDENTITY-MAP-129
IdentityMap mutable no será considerado concurrency-safe.

## DB-ORM-IDENTITY-MAP-130
Cada IdentityMap tendrá un logical owner.

## DB-ORM-IDENTITY-MAP-131
Concurrent mutation del mismo map estará fuera del contrato salvo soporte explícito.

## DB-ORM-IDENTITY-MAP-132
Async ORM work preferirá scopes separados.

## DB-ORM-IDENTITY-MAP-133
Tenant A y Tenant B tendrán identity namespaces independientes cuando corresponda.

## DB-ORM-IDENTITY-MAP-134
Tenant switching con managed entities será rechazado.

## DB-ORM-IDENTITY-MAP-135
Tenant switching preferirá nuevo EntityManager.

## DB-ORM-IDENTITY-MAP-136
Cross-context entity registration será error.

## DB-ORM-IDENTITY-MAP-137
Shard identity isolation seguirá las mismas reglas.

## DB-ORM-IDENTITY-MAP-138
Read/write physical routing no deberá fragmentar identidad lógica.

## DB-ORM-IDENTITY-MAP-139
Replica lag no deberá sobrescribir managed dirty state silenciosamente.

## DB-ORM-IDENTITY-MAP-140
IdentityMap solo proporciona canonical instance; hydration decide state reconciliation.

## DB-ORM-IDENTITY-MAP-141
Second-level cache será independiente de IdentityMap.

## DB-ORM-IDENTITY-MAP-142
Entity Cache HIT será distinto de IdentityMap HIT.

## DB-ORM-IDENTITY-MAP-143
Cache invalidation no eliminará managed entity automáticamente.

## DB-ORM-IDENTITY-MAP-144
Managed instance tendrá precedencia contextual sobre stale cached copies.

## DB-ORM-IDENTITY-MAP-145
IdentityMap no será serializable como persistence contract.

## DB-ORM-IDENTITY-MAP-146
IdentityMap no se almacenará en session.

## DB-ORM-IDENTITY-MAP-147
IdentityMap no se almacenará en queue payload.

## DB-ORM-IDENTITY-MAP-148
IdentityMap no sobrevivirá como mutable worker-global state.

## DB-ORM-IDENTITY-MAP-149
Entity serialization no serializará PersistenceContext automáticamente.

## DB-ORM-IDENTITY-MAP-150
Queue boundaries preferirán EntityReference o typed identifier.

## DB-ORM-IDENTITY-MAP-151
IdentityMap HIT no implicará authorization.

## DB-ORM-IDENTITY-MAP-152
IdentityMap no reemplazará Authorization System.

## DB-ORM-IDENTITY-MAP-153
Runtime isolation será también una security invariant.

## DB-ORM-IDENTITY-MAP-154
Request A entities nunca serán visibles como managed entities de Request B.

## DB-ORM-IDENTITY-MAP-155
IdentityMap reset será determinista.

## DB-ORM-IDENTITY-MAP-156
Reset completo dejará count cero.

## DB-ORM-IDENTITY-MAP-157
IdentityMap no dependerá de SQL Compiler.

## DB-ORM-IDENTITY-MAP-158
IdentityMap no dependerá de Driver.

## DB-ORM-IDENTITY-MAP-159
IdentityMap no dependerá de Migration System.

## DB-ORM-IDENTITY-MAP-160
IdentityMap nunca convertirá una identidad lógica en una consulta por sí mismo.

---

# 256. Anti-pattern: static Identity Map

Incorrecto:

```php
final class IdentityMap
{
    private static array $entities = [];
}
```

En FrankenPHP esto puede convertirse en:

```text
Request A
→ stores User#10

Request B
→ receives User#10 from Request A
```

lo cual sería una violación crítica.

---

# 257. Anti-pattern: key por class name

Incorrecto:

```php
$key = get_class($entity) . ':' . $entity->id;
```

Problemas:

```text
proxy classes
inheritance
typed identifiers
tenant isolation
composite IDs
class refactoring
```

---

# 258. Anti-pattern: key por table

Incorrecto:

```text
users:10
```

porque:

```text
EntityType
≠
TableName
```

---

# 259. Anti-pattern: loose equality

Incorrecto:

```php
if ($storedId == $incomingId) {
    // same entity
}
```

---

# 260. Anti-pattern: silent replacement

Incorrecto:

```php
$this->map[$key] = $newEntity;
```

cuando existe otra instancia.

---

# 261. Anti-pattern: Identity Map como cache

Incorrecto:

```text
IdentityMap MISS
→ load Redis
→ load DB
```

dentro del propio IdentityMap.

Eso corresponde a un loader/cache coordinator externo.

---

# 262. Anti-pattern: automatic freshness

Incorrecto:

```text
IdentityMap HIT
→ query database to verify freshness
```

---

# 263. Anti-pattern: identifier mutation

Incorrecto:

```php
$user->setId(20);
```

sobre una entidad managed `User#10`.

---

# 264. Anti-pattern: clear automático

Incorrecto:

```text
memory threshold exceeded
→ silently clear IdentityMap
```

---

# 265. Anti-pattern: repository-specific map

Incorrecto:

```text
UserRepository IdentityMap A
AdminRepository IdentityMap B
```

dentro del mismo PersistenceContext.

---

# 266. Anti-pattern: proxy type identity

Incorrecto:

```text
App\Proxy\UserProxy#10
≠
App\Entity\User#10
```

si ambos representan la misma entidad.

---

# 267. Anti-pattern: connection object namespace

Incorrecto:

```text
EntityKey(PDO object ID, User, 10)
```

---

# 268. Anti-pattern: tenant mutable global

Incorrecto:

```php
EntityKey::from(
    GlobalTenant::$current,
    User::class,
    10
);
```

en persistent runtime.

---

# 269. Flujo de find()

```text
find(User, 10)
      │
      ▼
Resolve Entity Metadata
      │
      ▼
Resolve Identity Namespace
      │
      ▼
Canonicalize Identifier
      │
      ▼
Build EntityKey
      │
      ▼
IdentityMap
   ┌──┴──┐
   │     │
  HIT   MISS
   │     │
   ▼     ▼
return   Entity Query
same        │
object      ▼
          Result
            │
            ▼
         Hydrator
            │
            ▼
       IdentityMap
            │
            ▼
      canonical entity
```

---

# 270. Flujo de hydration con JOIN

```text
Database Rows

User#10 Role#1
User#10 Role#2
User#10 Role#3
      │
      ▼
Identifier Extraction
      │
      ▼
EntityKey(User,10)
      │
      ▼
IdentityMap
      │
      ├── first row → MISS → create User A
      │
      ├── second row → HIT → reuse User A
      │
      └── third row → HIT → reuse User A
```

Resultado:

```text
User A
├── Role#1
├── Role#2
└── Role#3
```

---

# 271. Flujo generated ID

```text
NEW User
(no DB-generated ID)
      │
      ▼
UnitOfWork Temporary Token
      │
      ▼
Persistence Planner
      │
      ▼
INSERT
      │
      ▼
Execution Result
      │
      ├── FAILURE
      │
      ├── UNKNOWN
      │
      └── SUCCESS
             │
             ▼
      Generated Identifier
             │
             ▼
        Canonicalization
             │
             ▼
          EntityKey
             │
             ▼
      Identity Establishment
             │
             ▼
         IdentityMap
```

---

# 272. Flujo de conflicto

```text
IdentityMap

User#10 → object A

Incoming:
User#10 → object B
            │
            ▼
Compare Object Identity
            │
      ┌─────┴─────┐
      │           │
    same       different
      │           │
      ▼           ▼
idempotent    IDENTITY
registration  CONFLICT
```

---

# 273. Flujo de detach

```text
User#10 → object A
        │
        ▼
EntityManager::detach(A)
        │
        ▼
PersistenceContext Coordinator
        │
        ├── IdentityMap.remove(User#10)
        ├── UnitOfWork.detach(A)
        ├── Snapshot cleanup
        └── Entity State → DETACHED
```

---

# 274. Flujo persistent runtime

```text
Worker
 │
 ├── Shared Immutable Metadata
 │
 ├── Request A
 │    └── PersistenceContext A
 │         ├── EntityManager A
 │         ├── IdentityMap A
 │         └── UnitOfWork A
 │
 ├── RESET
 │
 └── Request B
      └── PersistenceContext B
           ├── EntityManager B
           ├── IdentityMap B
           └── UnitOfWork B
```

Nunca:

```text
Worker
└── IdentityMap shared across requests
```

---

# 275. Fórmula de identidad canónica

```text
CanonicalEntityInstance(Context, Key)
=
at most one managed object instance
```

---

# 276. Fórmula de EntityKey

```text
EntityKey
=
IdentityNamespace
×
IdentityEntityType
×
CanonicalEntityIdentifier
```

---

# 277. Fórmula de igualdad de key

```text
SameEntityKey(A,B)
=
SameIdentityNamespace(A,B)
∧
SameIdentityEntityType(A,B)
∧
SameCanonicalIdentifier(A,B)
```

---

# 278. Fórmula de canonical instance

```text
IdentityMap[K] = E₁
∧
Register(K,E₂)
∧
E₁ !== E₂

⇒

EntityIdentityConflict
```

---

# 279. Fórmula de lookup

```text
IdentityLookup(K)
=
HIT(E)
∨
MISS
```

Nunca:

```text
HIT
∨
MISS
∨
DATABASE_QUERY
```

El query pertenece al loader.

---

# 280. Fórmula de generated identity

```text
GeneratedIdentityEstablishment
=
ExecutionOutcomeSufficientlyCertain
∧
GeneratedIdentifierAvailable
∧
IdentifierCanonicalizable
∧
NoIdentityConflict
```

---

# 281. Fórmula de hydration

```text
HydrateEntity(Row)
=
ExtractIdentifier
→ Canonicalize
→ BuildEntityKey
→ IdentityLookup
→ ReuseOrInstantiate
→ Hydrate
→ Register/Reconcile
```

---

# 282. Fórmula de safe runtime

```text
SafeIdentityMapRuntime
=
ScopedPersistenceContext
∧
ScopedIdentityMap
∧
ImmutableSharedMetadata
∧
DeterministicReset
∧
NoCrossRequestManagedEntities
∧
NoCrossTenantIdentityCollision
```

---

# 283. Fórmula de memoria

Para modo tracked:

```text
ManagedMemory
≈
Σ(
    EntityObject
    + IdentityMapEntry
    + EntityState
    + Snapshot
    + UnitOfWorkMetadata
    + LoadedRelations
)
```

Por ello:

```text
IdentityMap size
```

es solo una parte del costo total.

---

# 284. Fórmula de scope

```text
IdentityGuarantee
=
PersistenceContextLifetime
```

No:

```text
ProcessLifetime
```

---

# 285. Master Formula

```text
Database Identity Map System
=
Canonical Entity Keys
+
Typed Identifier Canonicalization
+
Identity Namespaces
+
Canonical Managed Instances
+
Forward Identity Lookup
+
Reverse Object Lookup
+
Conflict Detection
+
Generated Identity Establishment
+
Hydration Deduplication
+
Proxy Identity Resolution
+
Inheritance Identity Resolution
+
Entity Reference Integration
+
UnitOfWork Coordination
+
Detach/Clear Semantics
+
Transaction-Aware Reconciliation Boundaries
+
Memory Governance
+
Streaming Policies
+
Persistent Runtime Isolation
+
Multitenancy Isolation
+
Telemetry
+
Diagnostics
```

---

# 286. Master Rule

> **En VoltStack, Identity Map no representa una caché de filas ni una prueba de existencia en la base de datos: representa la autoridad local del PersistenceContext sobre qué objeto PHP es la instancia canónica de una identidad lógica determinada.**

---

# 287. Resultado arquitectónico

Con este sistema:

```text
EntityKey
    │
    ▼
IdentityMap
    │
    ├── canonical object identity
    ├── hydration deduplication
    ├── repository convergence
    ├── query convergence
    ├── proxy convergence
    ├── relationship consistency
    └── persistence-context identity
```

VoltStack obtiene la base necesaria para construir correctamente:

```text
UnitOfWork
Change Tracking
Entity Snapshots
Persistence Planning
Flush
```

---

# 288. Relación con el siguiente bloque

Hasta este punto sabemos:

```text
Entity State
+
Entity Lifecycle
+
Identity Map
```

Pero todavía falta responder:

> ¿Cómo sabe VoltStack qué entidades deben insertarse, cuáles cambiaron, cuáles deben actualizarse o eliminarse, y cómo coordina todo ese trabajo antes de ejecutar persistencia?

Esa responsabilidad pertenece al:

```text
Unit of Work
```

---

# 289. Siguiente documento

```text
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
```

Este documento deberá formalizar:

```text
UnitOfWork architecture
PersistenceContext integration
entity registration
NEW/MANAGED/REMOVED tracking
scheduled operations
dirty checking coordination
ChangeSet ownership
Entity Snapshot integration
IdentityMap integration
relationship change tracking
collection changes
cascade discovery
orphan removal
EntityChangeGraph
dependency discovery
flush preparation
lifecycle stabilization
generated identifiers
persistence ordering inputs
transaction boundaries
failure semantics
unknown outcomes
state reconciliation
rollback implications
partial execution
memory management
large UnitOfWork governance
clear/detach
persistent runtime isolation
telemetry
diagnostics
extension boundaries
```

con la regla central:

> **UnitOfWork describe y coordina el conjunto de cambios ORM pendientes dentro de un PersistenceContext; no ejecuta SQL directamente y no debe confundirse con una transacción de base de datos.**