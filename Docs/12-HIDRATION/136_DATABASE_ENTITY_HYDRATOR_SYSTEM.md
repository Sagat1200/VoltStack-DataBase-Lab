# 136_DATABASE_ENTITY_HYDRATOR_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Hydrator System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 136 — Database Entity Hydrator System  
**Bloque:** 12 — Hydration  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Hydrator System` define el componente especializado responsable de transformar resultados relacionales en entidades ORM correctamente materializadas.

Su responsabilidad principal puede expresarse como:

```text
Result Row
    ↓
Entity Hydration Plan
    ↓
Identifier Resolution
    ↓
EntityKey
    ↓
IdentityMap
    ↓
Reuse / Construct
    ↓
Populate
    ↓
Relationship Assembly
    ↓
Snapshot Baseline
    ↓
ORM Registration
    ↓
Lifecycle
    ↓
Canonical Managed Entity
```

El sistema deberá garantizar que la materialización de entidades preserve simultáneamente:

- identidad lógica;
- identidad canónica del objeto;
- tipos PHP;
- mapping ORM;
- estado de carga;
- snapshots;
- UnitOfWork;
- relaciones;
- lifecycle;
- aislamiento del `PersistenceContext`.

---

# 2. Principio central

> **El `EntityHydrator` materializa o completa la representación canónica de una entidad a partir de un resultado estructurado; nunca crea silenciosamente una segunda instancia administrada cuando el `IdentityMap` ya representa esa identidad.**

Formalmente:

```text
EntityKey k
+
PersistenceContext p
```

debe cumplir:

```text
Managed(a, k, p)
∧
Managed(b, k, p)

⇒

a === b
```

---

# 3. Responsabilidad arquitectónica

El Entity Hydrator responde:

```text
¿Cómo convierto estos valores
en la representación ORM correcta
de esta entidad?
```

No responde:

```text
¿Qué consulta debo ejecutar?
```

ni:

```text
¿Qué SQL debo generar?
```

ni:

```text
¿Qué cambios debo persistir?
```

---

# 4. EntityHydrator ≠ EntityManager

El `EntityManager` coordina el contexto ORM.

El `EntityHydrator` realiza materialización.

```text
EntityManager
    │
    └── EntityLoader
            │
            ▼
       Entity Query
            │
            ▼
         Result
            │
            ▼
      EntityHydrator
```

El Hydrator no deberá convertirse en un segundo coordinador global.

---

# 5. EntityHydrator ≠ IdentityMap

El Hydrator utiliza:

```text
IdentityMap
```

pero no redefine sus invariantes.

```text
EntityHydrator
    ↓
EntityKey
    ↓
IdentityMap.lookup()
```

---

# 6. EntityHydrator ≠ UnitOfWork

El Hydrator puede registrar una entidad recién cargada en el `UnitOfWork`.

Pero:

```text
Hydration
≠
Change Tracking
≠
Persistence Scheduling
```

---

# 7. EntityHydrator ≠ Entity Factory

Una factory construye objetos.

El Hydrator además:

- resuelve identidad;
- reutiliza objetos;
- convierte tipos;
- escribe campos;
- establece relaciones;
- crea baseline;
- registra estado;
- coordina lifecycle.

Por tanto:

```text
EntityConstruction
⊂
EntityHydration
```

---

# 8. EntityHydrator ≠ Serializer

El Hydrator no es un deserializador genérico.

Su input no es:

```text
arbitrary JSON
```

sino:

```text
validated database result
+
compiled ORM hydration metadata
```

---

# 9. Posición dentro del read path

```text
Entity Query
    ↓
Query Engine
    ↓
Compiler
    ↓
Execution Engine
    ↓
Result
    ↓
Result Hydration
    ↓
┌──────────────────────────┐
│ Entity Hydrator          │
└────────────┬─────────────┘
             │
      ┌──────┼────────┐
      ▼      ▼        ▼
 Metadata Identity   Types
             │
             ▼
        IdentityMap
             │
             ▼
          Entity
             │
      ┌──────┼──────────┐
      ▼      ▼          ▼
   Snapshot State      UoW
             │
             ▼
         Lifecycle
```

---

# 10. Input principal

El Entity Hydrator no deberá recibir un array arbitrario sin contexto.

Utilizará un request estructurado.

```php
final readonly class EntityHydrationRequest
{
    public function __construct(
        public ResultRow $row,
        public CompiledEntityHydrationPlan $plan,
        public EntityHydrationContext $context,
    ) {}
}
```

---

# 11. EntityHydrationContext

Ejemplo conceptual:

```php
final readonly class EntityHydrationContext
{
    public function __construct(
        public PersistenceContext $persistenceContext,
        public HydrationScope $hydrationScope,
        public ExistingEntityHydrationPolicy $existingEntityPolicy,
        public EntityHydrationPolicy $policy,
    ) {}
}
```

No deberá incluir acceso indiscriminado al framework.

---

# 12. Output

El resultado interno puede ser más rico que simplemente:

```php
object
```

Por ejemplo:

```php
final readonly class EntityHydrationResult
{
    public function __construct(
        public object $entity,
        public EntityKey $key,
        public EntityHydrationOrigin $origin,
        public EntityHydrationCompleteness $completeness,
    ) {}
}
```

---

# 13. EntityHydrationOrigin

```php
enum EntityHydrationOrigin
{
    case CONSTRUCTED;
    case IDENTITY_MAP_REUSED;
    case RESERVED_REUSED;
    case PROXY_INITIALIZED;
    case REFRESHED;
}
```

Esto facilita:

- lifecycle;
- telemetry;
- debugging;
- relationship assembly.

---

# 14. EntityHydrationCompleteness

```php
enum EntityHydrationCompleteness
{
    case COMPLETE;
    case PARTIAL;
}
```

`UNKNOWN` no deberá usarse cuando el ORM necesita decidir si una entidad puede considerarse completamente cargada.

---

# 15. Pipeline principal

```text
EntityHydrationRequest
        │
        ▼
Validate Plan
        │
        ▼
Read Identifier
        │
        ▼
Convert Identifier
        │
        ▼
Canonicalize Identifier
        │
        ▼
Resolve Identity Domain
        │
        ▼
Resolve Identity Namespace
        │
        ▼
Build EntityKey
        │
        ▼
IdentityMap Lookup
        │
     ┌──┴──────────────┐
     │                 │
    HIT               MISS
     │                 │
     ▼                 ▼
Existing Policy    Reserve Key
     │                 │
     │                 ▼
     │             Construct
     │                 │
     │                 ▼
     │              Populate
     │                 │
     └────────┬────────┘
              ▼
     Assemble Relationships
              │
              ▼
     Validate Completeness
              │
              ▼
      Build Snapshot Baseline
              │
              ▼
       ORM Registration
              │
              ▼
          Activate Entry
              │
              ▼
           postLoad
              │
              ▼
      EntityHydrationResult
```

---

# 16. Identifier-first Hydration

La identidad deberá resolverse antes de construir la entidad siempre que el mapping lo permita.

```text
row
 ↓
identifier
 ↓
EntityKey
 ↓
IdentityMap
```

Esto evita construir un objeto para descubrir después que ya existía.

---

# 17. Identifier Reader

El plan compilado deberá contener un lector especializado.

```php
interface EntityIdentifierReader
{
    public function read(
        ResultRow $row,
        CompiledIdentifierHydrationPlan $plan
    ): EntityIdentifierReadResult;
}
```

---

# 18. Identifier conversion

Un identificador puede ser:

```text
database BIGINT
→ PHP int
```

o:

```text
database UUID
→ UserId
```

o compuesto:

```text
(country_code, number)
→ CustomerIdentifier
```

La conversión deberá ocurrir antes de formar `EntityKey`.

---

# 19. Canonical Identifier

Dos representaciones equivalentes deberán producir la misma identidad.

Por ejemplo, si el mapping lo permite:

```text
"42"
42
```

no deberán generar accidentalmente dos keys distintas.

---

# 20. EntityKey

La fórmula permanece:

```text
EntityKey
=
EntityIdentityNamespace
×
IdentityDomainType
×
CanonicalIdentifier
```

---

# 21. Namespace

El Hydrator obtiene el namespace del `PersistenceContext`.

No lo deduce de:

- DSN;
- objeto PDO;
- hostname;
- tabla;
- conexión física.

---

# 22. Identity domain

No siempre será simplemente:

```php
User::class
```

Inheritance puede establecer un dominio de identidad compartido.

---

# 23. IdentityMap Lookup

```php
$lookup = $identityMap->lookup($entityKey);
```

Resultado conceptual:

```text
MISS
ACTIVE
RESERVED
```

---

# 24. ACTIVE

Existe una representación canónica completamente registrada.

El Hydrator deberá reutilizarla.

---

# 25. RESERVED

Existe una representación actualmente en construcción dentro del mismo protocolo de hidratación.

Esto puede ocurrir con:

```text
A → B → A
```

---

# 26. MISS

No existe representación en el contexto actual.

Solo entonces podrá comenzar la construcción normal.

---

# 27. IdentityMap hit

Un hit no significa:

```text
entity is fresh
```

ni:

```text
database was queried now
```

ni:

```text
entity matches every value in current row
```

Solo significa:

> El PersistenceContext ya posee una representación canónica para esa identidad.

---

# 28. ExistingEntityHydrationPolicy

```php
enum ExistingEntityHydrationPolicy
{
    case REUSE_ONLY;
    case INITIALIZE_MISSING;
    case REFRESH;
    case REJECT_CONFLICT;
}
```

---

# 29. REUSE_ONLY

Comportamiento normal para una entidad ya managed:

```text
IdentityMap HIT
→ return canonical entity
```

sin sobrescribir campos persistentes arbitrariamente.

---

# 30. Protección de cambios locales

Supongamos:

```text
DB:
name = "Alice"
```

La aplicación modifica:

```php
$user->name = 'Alicia';
```

Después otra query retorna:

```text
name = "Alice"
```

VoltStack no deberá hacer:

```text
Alicia
→ Alice
```

silenciosamente.

---

# 31. Regla

```text
NormalHydration
+
ExistingManagedEntity
+
LocallyModifiedField

⇒

PreserveLocalState
```

---

# 32. INITIALIZE_MISSING

Puede utilizarse para completar:

- proxy;
- deferred fields;
- partial entity;
- lazy property.

Solo los campos conocidos como no cargados podrán inicializarse.

---

# 33. LoadedFieldSet

Ejemplo:

```text
User#42

loaded:
  id
  name

missing:
  email
  created_at
```

Una query posterior puede completar:

```text
email
created_at
```

sin sobrescribir `name`.

---

# 34. REFRESH

`REFRESH` representa una operación explícita.

```text
EntityManager::refresh($entity)
```

puede autorizar sustitución del baseline según policy.

---

# 35. Refresh no reemplaza objeto

```text
before refresh:
Object A = User#42

after refresh:
Object A = User#42
```

No:

```text
Object B
```

---

# 36. REJECT_CONFLICT

Útil para operaciones donde la presencia de una entidad previa constituye una situación inesperada.

---

# 37. Identity Reservation

Cuando existe MISS:

```text
EntityKey
 ↓
reserve
```

antes de construir el grafo completo.

---

# 38. Reservation purpose

Garantiza:

```text
at most one materialization attempt
```

para una identidad dentro del scope coordinado.

---

# 39. Reservation token

```php
final readonly class EntityHydrationReservation
{
    public function __construct(
        public EntityKey $key,
        public HydrationReservationId $id,
    ) {}
}
```

---

# 40. Reservation lifecycle

```text
CREATED
   ↓
OBJECT_BOUND
   ↓
HYDRATING
   ↓
PREPARED
   ↓
ACTIVE
```

En error:

```text
*
↓
ABORTED
```

---

# 41. Reservation state ≠ EntityState

Estos estados pertenecen exclusivamente al protocolo interno.

No deberán aparecer como:

```text
NEW
MANAGED
REMOVED
DETACHED
```

---

# 42. Construction after reservation

Secuencia:

```text
reserve key
↓
construct object
↓
bind object to reservation
```

Esto permite que nested hydration encuentre el mismo objeto.

---

# 43. Early binding

Debe realizarse únicamente después de que:

```text
object type
```

y:

```text
identity compatibility
```

hayan sido validados.

---

# 44. Partially hydrated object

El objeto ligado a una reservation puede estar incompleto.

Solo componentes internos podrán observarlo.

---

# 45. Public lookup

La API normal del IdentityMap deberá diferenciar:

```text
ACTIVE
```

de:

```text
RESERVED
```

para no devolver objetos incompletos al usuario.

---

# 46. Circular graph example

```text
User#1
  ↓
Profile#5
  ↓
User#1
```

Flujo:

```text
reserve User#1
construct User A
bind User A

hydrate Profile#5
reserve Profile#5
construct Profile P

resolve Profile.user = User#1
IdentityMap internal lookup
→ RESERVED User A

Profile.user = User A

finish Profile
activate Profile#5

finish User
activate User#1
```

---

# 47. Entity Construction Strategy

Contrato:

```php
interface EntityConstructionStrategy
{
    public function construct(
        CompiledEntityConstructionPlan $plan,
        EntityConstructionContext $context
    ): object;
}
```

---

# 48. Construction strategies

VoltStack podrá soportar:

```text
CONSTRUCTOR
BYPASS_CONSTRUCTOR
STATIC_FACTORY
CUSTOM
```

---

# 49. Constructor hydration

Ejemplo:

```php
final class User
{
    public function __construct(
        private UserId $id,
        private string $name,
    ) {}
}
```

Si mapping lo declara compatible:

```text
Result
→ constructor args
→ User
```

---

# 50. Constructor side effects

Un constructor de dominio podría:

- generar UUID;
- registrar domain event;
- establecer fecha actual;
- validar creación;
- llamar servicios.

Por ello no deberá asumirse automáticamente adecuado para reconstitution.

---

# 51. BYPASS_CONSTRUCTOR

Permite reconstrucción sin semántica de "crear una nueva entidad".

Pero deberá ser:

- explícito;
- soportado por PHP/runtime;
- validado;
- compilado;
- encapsulado.

---

# 52. No direct Reflection everywhere

No:

```php
$reflection = new ReflectionClass($class);
$reflection->newInstanceWithoutConstructor();
```

por cada row.

El mecanismo deberá precompilarse.

---

# 53. STATIC_FACTORY

Ejemplo:

```php
User::reconstitute(...)
```

solo si metadata declara esa factory.

---

# 54. CUSTOM

Extensiones especializadas podrán definir construction strategies.

No podrán romper las reglas de identidad.

---

# 55. EntityConstructionPlan

Podrá contener:

```text
target class
strategy
constructor/factory
argument bindings
required fields
field initialization policy
readonly constraints
```

---

# 56. Field hydration

Después de construir:

```text
Database Value
↓
Type Conversion
↓
PHP Value
↓
Compiled Field Writer
↓
Entity Property
```

---

# 57. Compiled Field Writer

```php
interface EntityFieldWriter
{
    public function write(
        object $entity,
        mixed $value
    ): void;
}
```

---

# 58. Writer generation

Un plan podrá precompilar writers por campo:

```text
User.id
→ writer_01

User.name
→ writer_02

User.email
→ writer_03
```

---

# 59. Writer strategies

```text
CONSTRUCTOR_ARGUMENT
PROPERTY
GENERATED_ACCESSOR
REFLECTION_ACCESSOR
SETTER
CUSTOM
```

---

# 60. Setter policy

Un setter de dominio no será utilizado automáticamente.

Mapping deberá declararlo.

---

# 61. Example problem

```php
public function setEmail(Email $email): void
{
    $this->email = $email;
    $this->domainEvents[] = new EmailChanged();
}
```

Hydration no debería emitir:

```text
EmailChanged
```

solo por reconstruir la entidad.

---

# 62. Reconstitution-safe accessors

VoltStack podrá distinguir:

```text
DomainWriteAccessor
```

de:

```text
HydrationWriteAccessor
```

---

# 63. Typed properties

Antes de escribir:

```text
converted value
```

debe ser compatible con el tipo PHP efectivo.

---

# 64. Nullability

```text
SQL NULL
```

solo podrá convertirse en `null` cuando mapping y tipo PHP sean compatibles.

---

# 65. Missing

```text
MISSING
```

no se escribirá como:

```php
null
```

---

# 66. Readonly properties

Las propiedades readonly requieren un plan que garantice una única inicialización válida.

---

# 67. Invalid readonly mapping

Debe detectarse durante:

```text
metadata compilation
```

o:

```text
hydration plan compilation
```

cuando sea posible.

---

# 68. Required fields

El plan conocerá:

```text
RequiredHydrationFields
```

para entidades completas.

---

# 69. Completeness check

Antes de activación:

```text
RequiredFields
⊆
LoadedFields
```

debe cumplirse para `COMPLETE`.

---

# 70. Partial entities

Si no:

```text
RequiredFields
⊆
LoadedFields
```

solo podrá continuar cuando el modo sea explícitamente:

```text
PARTIAL_ENTITY
```

---

# 71. Partial entity state

El ORM deberá mantener externamente:

```text
LoadedFieldSet
```

sin contaminar necesariamente la entidad.

---

# 72. Embedded Values

Ejemplo:

```php
final class User
{
    private Address $address;
}
```

Resultado:

```text
address_street
address_city
address_zip
```

puede reconstruir:

```text
Address
```

---

# 73. EmbeddedHydrationPlan

```text
Embedded metadata
+
column bindings
+
type converters
+
construction strategy
```

---

# 74. Embedded value ≠ Entity

Normalmente:

```text
Address
```

no entra en IdentityMap.

---

# 75. Nullable embedded value

Mapping deberá decidir si:

```text
street = NULL
city   = NULL
zip    = NULL
```

significa:

```text
address = null
```

o:

```text
Address(null, null, null)
```

si eso fuera válido.

---

# 76. Inheritance

El Entity Hydrator deberá soportar resolución polimórfica.

---

# 77. Discriminator flow

```text
Result Row
   ↓
Discriminator Reader
   ↓
Discriminator Map
   ↓
Concrete Entity Metadata
   ↓
Identity Domain
   ↓
EntityKey
```

---

# 78. Example

```text
type = "employee"
```

puede mapear:

```text
Person identity domain
→ Employee concrete entity
```

---

# 79. Identity collision prevention

Si primero se carga:

```text
Person#10
```

y después se determina que corresponde a:

```text
Employee#10
```

el ORM no deberá producir dos objetos.

---

# 80. Canonical identity domain

Inheritance metadata definirá la key apropiada.

---

# 81. Unknown discriminator

Debe generar:

```text
UnknownEntityDiscriminatorException
```

o equivalente.

---

# 82. No dynamic class names from DB

Prohibido:

```php
$class = $row['type'];

return new $class();
```

---

# 83. Proxy Integration

`getReference(User::class, 42)` puede haber registrado una representación proxy.

---

# 84. Later hydration

Cuando llegue la row real:

```text
IdentityMap
User#42
→ Proxy A
```

el Hydrator deberá:

```text
initialize Proxy A
```

si la estrategia proxy lo soporta.

No:

```text
create User B
```

---

# 85. Proxy identity

```text
Proxy<User>
```

y:

```text
User
```

comparten el mismo `EntityKey`.

---

# 86. Ghost objects

Si VoltStack utiliza lazy ghost objects, el ghost deberá poder convertirse en instancia inicializada sin cambiar la identidad del objeto.

---

# 87. EntityReference

Si `EntityReference` es un objeto separado y no una entidad/proxy:

```text
EntityReference
≠
IdentityMap Entity Entry
```

hasta resolución según contrato.

---

# 88. Relationship Hydration

EntityHydrator podrá coordinarse con:

```text
RelationshipAssembler
```

pero no deberá contener toda la lógica de collections dentro de una única clase.

---

# 89. To-one relationship

```text
User
→ Organization
```

puede resolverse mediante:

```text
related EntityKey
```

y reutilizar IdentityMap.

---

# 90. To-many relationship

JOIN:

```text
User#1 Order#10
User#1 Order#11
User#1 Order#12
```

debe producir:

```text
User#1
  Orders:
    #10
    #11
    #12
```

---

# 91. Root entity deduplication

```text
same root EntityKey
→ same root object
```

---

# 92. Child entity deduplication

```text
same child EntityKey
→ same child object
```

---

# 93. Collection edge deduplication

Además:

```text
same owner
+
same relationship
+
same child
```

no deberá producir un edge duplicado cuando la semántica de colección sea set-like.

---

# 94. RelationshipAssemblyKey

```text
OwnerEntityKey
×
RelationshipId
×
RelatedEntityKey
```

---

# 95. JOIN multiplication

Ejemplo:

```text
User
× Orders
× Roles
```

puede producir:

```text
Order#10 + Role#1
Order#10 + Role#2
```

`Order#10` no deberá agregarse dos veces.

---

# 96. LEFT JOIN

Si related identity está ausente:

```text
related entity = absent
```

No crear objeto vacío.

---

# 97. All-null detection

Debe basarse en:

```text
identifier mapping
```

no en todas las columnas arbitrariamente.

---

# 98. Collection completeness

RelationshipAssembler deberá registrar:

```text
COMPLETE
PARTIAL
UNKNOWN
```

según semántica del query.

---

# 99. Filtered eager loading

```text
User
JOIN orders
WHERE orders.status = 'OPEN'
```

no implica necesariamente que:

```text
User.orders
```

esté completamente inicializada.

---

# 100. Critical safety rule

```text
FilteredRelationResult
⇏
CompleteCollection
```

---

# 101. Relationship fix-up

Para relación bidireccional:

```text
User.orders contains Order
```

podría requerir:

```text
Order.user = User
```

---

# 102. Fix-up semantics

El fix-up inicial deberá ejecutarse en modo:

```text
HYDRATION_INITIALIZATION
```

y no:

```text
APPLICATION_MUTATION
```

---

# 103. Dirty tracking

Por tanto:

```text
HydrationRelationshipFixup
⇏
DirtyRelationship
```

---

# 104. Snapshot baseline

Una vez que la entidad contiene el estado persistente obtenido de DB:

```text
Snapshot
=
DatabaseHydratedPersistentState
```

---

# 105. Snapshot order

Secuencia recomendada:

```text
construct
↓
hydrate persistent fields
↓
assemble initial relationship state
↓
prepare snapshot
↓
register managed baseline
↓
postLoad
```

---

# 106. Why before postLoad

Supongamos:

```php
#[PostLoad]
public function normalize(): void
{
    $this->name = trim($this->name);
}
```

Si `name` es persistente, el cambio deberá poder detectarse.

---

# 107. Correct baseline

DB:

```text
" Alice "
```

Snapshot:

```text
" Alice "
```

postLoad:

```text
"Alice"
```

Change tracking:

```text
" Alice " → "Alice"
```

---

# 108. Incorrect baseline

No:

```text
postLoad
↓
snapshot "Alice"
```

porque ocultaría la mutación.

---

# 109. UoW Registration

Una entidad cargada deberá registrarse como existente, no como nueva.

Conceptualmente:

```text
registerManaged(
    entity,
    EntityKey,
    snapshot
)
```

---

# 110. Hydrated ≠ NEW

Una entidad materializada desde una row existente:

```text
MANAGED
```

no:

```text
NEW
```

---

# 111. UoW registration ≠ persist()

El Hydrator no deberá llamar conceptualmente:

```php
$entityManager->persist($entity);
```

porque `persist()` representa intención de persistencia de una entidad introducida por aplicación.

---

# 112. Dedicated registration

Debe existir una operación interna:

```text
registerHydratedEntity
```

o equivalente.

---

# 113. EntityState registration

Después de activación:

```text
EntityState = MANAGED
```

salvo modos explícitos:

```text
READ_ONLY
UNMANAGED
PARTIAL_RESTRICTED
```

según diseño.

---

# 114. Atomic ORM registration

Se deberán coordinar:

```text
IdentityMap
SnapshotRegistry
EntityStateRegistry
UnitOfWork
LoadedFieldRegistry
```

---

# 115. Invariant

Nunca deberá quedar estable:

```text
IdentityMap = registered
Snapshot = missing
State = UNTRACKED
UoW = missing
```

para una entidad que se anuncia como `MANAGED`.

---

# 116. Registration Coordinator

```php
interface HydratedEntityRegistrationCoordinator
{
    public function register(
        HydratedEntityRegistration $registration
    ): void;
}
```

---

# 117. HydratedEntityRegistration

Puede incluir:

```text
entity
EntityKey
snapshot
loaded fields
read-only flag
relationship baselines
identity reservation
```

---

# 118. Internal atomicity

El coordinator deberá:

```text
prepare
↓
validate
↓
apply
↓
activate
```

con rollback interno cuando una etapa falle.

---

# 119. Not a DB transaction

```text
HydratedEntityRegistrationCoordinator
≠
TransactionManager
```

---

# 120. Lifecycle Integration

Una entidad materializada por primera vez podrá disparar:

```text
POST_LOAD
```

una vez que esté correctamente registrada.

---

# 121. postLoad ordering

```text
Entity complete enough
↓
IdentityMap ACTIVE
↓
Snapshot baseline
↓
EntityState MANAGED
↓
UoW aware
↓
POST_LOAD
```

---

# 122. Duplicate JOIN rows

No deben producir:

```text
POST_LOAD
POST_LOAD
POST_LOAD
```

para la misma materialización.

---

# 123. Rule

```text
POST_LOAD
=
once per logical materialization into PersistenceContext
```

salvo lifecycle contract más específico.

---

# 124. IdentityMap reuse

Si una query posterior encuentra la entidad ya activa:

```text
normal REUSE_ONLY
```

no dispara nuevamente `POST_LOAD`.

---

# 125. Refresh

Refresh utiliza:

```text
PRE_REFRESH
POST_REFRESH
```

no un nuevo `POST_LOAD`.

---

# 126. Proxy initialization

Deberá definirse si inicializar un proxy nunca cargado representa la primera materialización y por tanto dispara `POST_LOAD`.

Recomendación:

```text
uninitialized canonical proxy
↓
first complete DB materialization
↓
POST_LOAD once
```

---

# 127. Failure before registration

Ejemplo:

```text
type conversion failure
```

antes de activación.

Debe:

```text
abort reservation
release temporary state
```

---

# 128. Failure during registration

Si falla SnapshotRegistry después de IdentityMap preparation:

```text
rollback internal registration
```

---

# 129. Failure in postLoad

Situación distinta.

La entidad ya fue materializada correctamente.

Por tanto:

```text
HydrationCoreOutcome = SUCCEEDED
LifecycleOutcome = FAILED
```

---

# 130. postLoad failure

No deberá fingirse que la row nunca fue leída.

El contexto deberá aplicar la política del Lifecycle System.

---

# 131. Possible outcome model

```php
enum EntityHydrationCoreOutcome
{
    case SUCCEEDED;
    case FAILED;
}
```

junto con:

```text
LifecycleOutcome
```

para preservar dimensiones.

---

# 132. Hydration failure ≠ Transaction rollback

El Hydrator no ejecuta:

```text
ROLLBACK
```

por su cuenta.

---

# 133. Hydration failure ≠ Connection close

Tampoco cierra arbitrariamente la conexión.

---

# 134. Refresh Integration

Refresh deberá operar sobre la misma instancia.

Pipeline:

```text
managed entity
↓
PRE_REFRESH
↓
execute refresh query
↓
REFRESH hydration policy
↓
replace allowed persistent state
↓
new snapshot baseline
↓
POST_REFRESH
```

---

# 135. Dirty refresh policy

Si la entidad tiene cambios pendientes:

```text
refresh()
```

deberá tener política explícita.

Ejemplos:

```text
REJECT_IF_DIRTY
DISCARD_LOCAL_CHANGES
MERGE_ALLOWED_FIELDS
```

---

# 136. Recommended default

```text
REJECT_IF_DIRTY
```

o requerir intención explícita de descartar cambios.

Esto evita pérdida silenciosa.

---

# 137. Query rehydration ≠ refresh

Una consulta normal que encuentra una entidad managed:

```text
REUSE_ONLY
```

No equivale a `refresh()`.

---

# 138. External database changes

Si otro proceso modifica DB:

```text
IdentityMap entity
```

puede quedar stale.

El Entity Hydrator no intenta detectar automáticamente esa situación.

---

# 139. Freshness

```text
ManagedCanonicality
≠
DatabaseFreshness
```

---

# 140. Bulk updates

Una operación bulk puede dejar entidades managed stale.

La solución pertenece a:

- bulk operation policy;
- EntityManager;
- invalidation;
- refresh;
- clear.

No al EntityHydrator aislado.

---

# 141. Read-only entity hydration

Una estrategia podrá materializar:

```text
READ_ONLY_MANAGED
```

---

# 142. Read-only managed

Puede entrar al IdentityMap para conservar canonicalidad, pero UoW no genera cambios automáticamente.

---

# 143. Unmanaged hydration

Otra estrategia:

```text
UNMANAGED_READ_ONLY
```

podrá evitar:

```text
IdentityMap
SnapshotRegistry
UoW
```

permanentes.

---

# 144. Important semantic difference

```text
Managed Entity Hydration
```

garantiza canonicalidad dentro del PersistenceContext.

```text
Unmanaged Entity Hydration
```

no necesariamente.

---

# 145. Mixing modes

Si existe:

```text
User#42 → managed Object A
```

y una query unmanaged encuentra `User#42`, la policy deberá definir si:

- reutiliza A;
- crea objeto independiente read-only;
- rechaza mezcla.

---

# 146. Recommended default

Si la operación está asociada al mismo EntityManager:

```text
prefer canonical managed instance
```

para evitar dos representaciones contradictorias.

Un modo verdaderamente detached deberá declararse explícitamente.

---

# 147. Streaming Entity Hydration

Para millones de entidades:

```text
ResultCursor
↓
EntityHydrator
↓
yield
```

debe gobernar memoria.

---

# 148. Managed streaming

```text
yield
+
keep entity managed
```

mantiene máxima semántica ORM, pero aumenta memoria.

---

# 149. Detach-after-yield

```text
hydrate
↓
yield
↓
advance iterator
↓
detach previous entity
```

permite procesamiento acotado.

---

# 150. Important warning

El consumidor no deberá asumir que una entidad ya yielded sigue managed después de avanzar el cursor.

---

# 151. Windowed hydration

Ejemplo:

```text
hydrate 500 entities
process
flush if explicitly requested
clear window
continue
```

Hydration nunca ejecutará `flush()` automáticamente.

---

# 152. Streaming with JOIN

Debe completar una entidad raíz antes de yield cuando sus relaciones eager abarcan varias rows.

---

# 153. Root grouping

```text
User#1 Order#10
User#1 Order#11
User#2 Order#20
```

permite:

```text
collect User#1
yield User#1

collect User#2
yield User#2
```

---

# 154. Unordered result

Si rows de `User#1` pueden reaparecer después de `User#2`, un streaming graph sin buffering completo puede ser inseguro.

---

# 155. Hydrator responsibility

Debe detectar que la estrategia/plan requiere una precondición.

No deberá alterar SQL por sí mismo.

---

# 156. Result Hydration Coordination

`EntityHydrator` normalmente trabaja bajo:

```text
ResultHydrator
```

que controla:

- iteración de rows;
- root deduplication;
- tuple assembly;
- streaming;
- result cardinality.

---

# 157. Single-row responsibility

El EntityHydrator no deberá asumir que una row equivale necesariamente a una entidad raíz completa.

---

# 158. Result cardinality

Ejemplo:

```text
5 rows
```

pueden representar:

```text
1 User
+
5 Orders
```

---

# 159. HydrationPlan specialization

`CompiledEntityHydrationPlan` podrá contener:

```text
EntityMetadataId
IdentityDomain
IdentifierReader
DiscriminatorReader
ConstructionPlan
FieldHydrationPlans
EmbeddedHydrationPlans
RelationshipHydrationPlans
SnapshotPlan
LoadedFieldPlan
LifecyclePlan
```

---

# 160. FieldHydrationPlan

```php
final readonly class CompiledFieldHydrationPlan
{
    public function __construct(
        public FieldId $field,
        public ResultSlot $slot,
        public ValueConverter $converter,
        public EntityFieldWriter $writer,
        public bool $required,
    ) {}
}
```

---

# 161. Identifier plan

Debe distinguir:

```text
identifier source
database type
PHP type
canonicalization
composite ordering
nullability
```

---

# 162. Relationship plan

No deberá guardar mutable collection state.

Solo instrucciones inmutables.

---

# 163. Runtime state

Mutable state pertenece a:

```text
HydrationScope
```

---

# 164. Compiled Plan sharing

Puede compartirse entre requests si:

```text
metadata generation
type generation
result shape
```

son compatibles.

---

# 165. Entity object never cached in plan

Prohibido:

```text
CompiledEntityHydrationPlan
→ User object
```

---

# 166. EntityManager never cached in plan

Prohibido:

```text
CompiledEntityHydrationPlan
→ EntityManager
```

---

# 167. Tenant never captured mutably

Un plan no deberá cerrarse sobre:

```text
current tenant object
```

---

# 168. Performance model

El hot path ideal:

```text
row[index]
↓
precompiled converter
↓
precompiled writer
```

sin búsquedas repetidas por nombre.

---

# 169. Identity lookup complexity

Promedio esperado:

```text
O(1)
```

---

# 170. Field hydration complexity

Para `f` campos:

```text
O(f)
```

---

# 171. Row complexity

Aproximadamente:

```text
O(
  identifier components
  +
  hydrated fields
  +
  relationship edges
)
```

---

# 172. No metadata scanning per row

Evitar:

```php
foreach ($entityMetadata->allFields() as ...)
```

si el plan ya conoce los campos requeridos.

---

# 173. Generated hydration code

Una futura optimización podrá compilar:

```php
$id = $row[0];
$name = $row[1];
$email = $row[2];
```

y accessors especializados.

---

# 174. JIT-friendly design

Contratos simples, immutable plans y llamadas predecibles pueden beneficiar:

- OPcache;
- PHP JIT;
- FrankenPHP persistent workers.

---

# 175. Memory governance

El EntityHydrator deberá minimizar objetos temporales por row.

---

# 176. Avoid row copies

No convertir innecesariamente:

```text
ResultRow
→ associative array
→ normalized array
→ mapped array
→ entity
```

si puede utilizarse acceso posicional compilado.

---

# 177. Large values

BLOB/CLOB/stream fields deberán integrarse con Result/Type System sin copiar contenido completo innecesariamente cuando exista soporte streaming.

---

# 178. Telemetry

Métricas propuestas:

```text
orm.entity_hydration.operations
orm.entity_hydration.rows
orm.entity_hydration.constructed
orm.entity_hydration.reused
orm.entity_hydration.proxy_initialized
orm.entity_hydration.refreshed

orm.entity_hydration.identity.hit
orm.entity_hydration.identity.miss
orm.entity_hydration.identity.reserved

orm.entity_hydration.fields
orm.entity_hydration.relationships
orm.entity_hydration.partial

orm.entity_hydration.failures
orm.entity_hydration.duration
```

---

# 179. Construction ratio

```text
ConstructionRatio
=
ConstructedEntities
/
ResolvedEntityOccurrences
```

---

# 180. Identity reuse ratio

```text
IdentityReuseRatio
=
IdentityMapHits
/
IdentityResolutions
```

---

# 181. JOIN duplication metric

```text
EntityOccurrenceAmplification
=
EntityOccurrencesInRows
/
UniqueEntityKeys
```

---

# 182. Debug diagnostics

El profiler podrá mostrar:

```text
Entity Hydration

Entity: User
Rows encountered:       1,250
Unique users:              100
Constructed:               100
Reused occurrences:      1,150

Identity hit ratio:       92%

Fields hydrated:          800
Relations assembled:    2,350

Plan cache:               HIT
Duration:               8.4 ms
```

---

# 183. Sensitive information

No registrar por defecto:

```text
EntityKey raw value
email
token
password
personal fields
```

---

# 184. Persistent Runtime

En FrankenPHP:

```text
Worker
├── Request A
│   ├── EntityManager A
│   ├── IdentityMap A
│   └── HydrationScope A
│
└── Request B
    ├── EntityManager B
    ├── IdentityMap B
    └── HydrationScope B
```

---

# 185. Shared allowed

```text
CompiledEntityHydrationPlan
CompiledFieldWriter
CompiledTypeConverterPlan
CompiledConstructionPlan
```

si son inmutables.

---

# 186. Shared forbidden

```text
entity instances
current entity
reservation
current row
IdentityMap
UnitOfWork
SnapshotRegistry
EntityManager
relationship assembly state
```

---

# 187. No static entity state

Prohibido:

```php
EntityHydrator::$currentEntity
```

---

# 188. Reset

Al terminar un HydrationScope:

```text
temporary reservations
temporary graph state
row counters
assembly buffers
```

deben liberarse.

---

# 189. Request reset

Al finalizar request:

```text
PersistenceContext
IdentityMap
UoW
Snapshots
```

seguirán las reglas generales de Runtime Database.

---

# 190. Concurrent access

V1 no asumirá que un mismo EntityHydrator/PersistenceContext mutable pueda utilizarse concurrentemente desde múltiples fibers/coroutines.

---

# 191. Race example

```text
Fiber A:
lookup User#42 → MISS

Fiber B:
lookup User#42 → MISS
```

podría terminar en dos construcciones.

---

# 192. V1 policy

La concurrencia mutable sobre el mismo PersistenceContext podrá rechazarse mediante:

```text
HydrationConcurrencyGuard
```

---

# 193. Future policy

Una versión posterior podría hacer reservations concurrency-aware.

No es requisito V1.

---

# 194. Security

El Entity Hydrator deberá asumir que datos de DB pueden estar corruptos o ser incompatibles.

---

# 195. Type corruption

Si DB contiene:

```text
age = "not-a-number"
```

y mapping exige:

```text
int
```

debe fallar.

No:

```text
(int) "not-a-number" = 0
```

silenciosamente.

---

# 196. Enum corruption

Si DB contiene:

```text
status = "INVALID"
```

para:

```php
enum Status: string
```

debe producir error de conversión explícito.

---

# 197. Identifier corruption

Un identificador inválido nunca deberá crear:

```text
fake EntityKey
```

---

# 198. Discriminator security

Solo clases precompiladas en metadata podrán seleccionarse.

---

# 199. Extension security

Custom hydrators deberán registrarse mediante:

```text
HydrationExtensionRegistry
```

y respetar contratos del sistema.

---

# 200. Exception hierarchy

```text
DatabaseEntityHydrationException
├── EntityHydrationPlanException
├── EntityHydrationIdentifierException
├── EntityHydrationIdentityException
├── EntityHydrationIdentityConflictException
├── EntityHydrationReservationException
├── EntityHydrationConstructionException
├── EntityHydrationConstructorException
├── EntityHydrationFactoryException
├── EntityHydrationFieldException
├── EntityHydrationFieldTypeException
├── EntityHydrationReadonlyException
├── EntityHydrationMissingFieldException
├── EntityHydrationIncompleteException
├── EntityHydrationEmbeddedException
├── EntityHydrationDiscriminatorException
├── UnknownEntityDiscriminatorException
├── EntityHydrationProxyException
├── EntityHydrationRelationshipException
├── EntityHydrationRegistrationException
├── EntityHydrationSnapshotException
├── EntityHydrationLifecycleException
├── EntityHydrationRefreshException
├── EntityHydrationStreamingException
├── EntityHydrationConcurrentAccessException
├── EntityHydrationRuntimeIsolationException
└── EntityHydrationInvariantException
```

---

# 201. Failure context

Una excepción puede transportar:

```text
HydrationPlanId
EntityType
FieldId
ResultSlot
RelationshipId
HydrationScopeId
RowNumber
```

---

# 202. Failure context ≠ raw data dump

No incluir automáticamente:

```text
full row
```

---

# 203. Directory Structure

```text
src/Quantum/Database/ORM/Hydration/Entity/
│
├── Contract/
│   ├── EntityHydrator.php
│   ├── EntityIdentifierReader.php
│   ├── EntityConstructionStrategy.php
│   ├── EntityFieldWriter.php
│   └── HydratedEntityRegistrationCoordinator.php
│
├── Hydrator/
│   └── DefaultEntityHydrator.php
│
├── Request/
│   ├── EntityHydrationRequest.php
│   └── EntityHydrationContext.php
│
├── Result/
│   ├── EntityHydrationResult.php
│   ├── EntityHydrationOrigin.php
│   └── EntityHydrationCompleteness.php
│
├── Plan/
│   ├── EntityHydrationPlan.php
│   ├── CompiledEntityHydrationPlan.php
│   ├── CompiledIdentifierHydrationPlan.php
│   ├── CompiledFieldHydrationPlan.php
│   ├── CompiledEntityConstructionPlan.php
│   └── EntityHydrationPlanValidator.php
│
├── Identity/
│   ├── HydrationIdentityResolver.php
│   ├── EntityHydrationReservation.php
│   └── HydrationReservationId.php
│
├── Construction/
│   ├── ConstructorEntityConstructionStrategy.php
│   ├── BypassConstructorConstructionStrategy.php
│   ├── FactoryEntityConstructionStrategy.php
│   └── EntityConstructionContext.php
│
├── Field/
│   ├── PropertyEntityFieldWriter.php
│   ├── GeneratedEntityFieldWriter.php
│   ├── ReflectionEntityFieldWriter.php
│   └── SetterEntityFieldWriter.php
│
├── Embedded/
│   ├── EmbeddedValueHydrator.php
│   └── CompiledEmbeddedHydrationPlan.php
│
├── Inheritance/
│   ├── EntityDiscriminatorResolver.php
│   └── CompiledDiscriminatorMap.php
│
├── Relationship/
│   ├── RelationshipAssembler.php
│   ├── RelationshipAssemblyKey.php
│   └── RelationshipHydrationState.php
│
├── Registration/
│   ├── DefaultHydratedEntityRegistrationCoordinator.php
│   └── HydratedEntityRegistration.php
│
├── Refresh/
│   ├── EntityRefreshHydrator.php
│   └── DirtyRefreshPolicy.php
│
├── Streaming/
│   └── StreamingEntityHydrator.php
│
├── Runtime/
│   └── HydrationConcurrencyGuard.php
│
├── Telemetry/
│   ├── EntityHydrationTelemetry.php
│   └── EntityHydrationStatistics.php
│
└── Exception/
    └── ...
```

---

# 204. Integration map

```text
EntityHydrator
│
├── EntityMetadata
├── TypeSystem
├── ResultSystem
├── IdentityMap
├── EntitySnapshotSystem
├── EntityStateSystem
├── UnitOfWork
├── EntityLifecycleSystem
├── RelationshipHydration
├── ProxySystem
└── Telemetry
```

---

# 205. Dependency boundaries

No dependencia directa de:

```text
Driver
PDO
SQL Compiler
Migration
Schema Builder
HTTP
Authorization
Multitenancy package
```

---

# 206. Testing Strategy

El sistema deberá tener suites específicas para:

- identity;
- construction;
- fields;
- relationships;
- snapshots;
- lifecycle;
- refresh;
- streaming;
- runtime;
- failure recovery.

---

# 207. Identity tests

```text
same key → same instance
different key → different instance
same id/different type → different identity
same type/id/different namespace → different identity
composite IDs canonicalize
strongly typed IDs canonicalize
```

---

# 208. Duplicate object conflict

Registrar:

```text
User#42 → Object A
```

y luego intentar activar:

```text
User#42 → Object B
```

debe fallar explícitamente.

---

# 209. Existing object test

Normal query no sobrescribe campos dirty.

---

# 210. Partial completion test

Solo campos MISSING son inicializados bajo `INITIALIZE_MISSING`.

---

# 211. Refresh test

Refresh conserva:

```text
object identity
```

y actualiza baseline correctamente.

---

# 212. Construction tests

Cubrir:

```text
constructor
constructor bypass
factory
custom strategy
```

---

# 213. Constructor side-effect test

Hydration strategy no debe invocar constructor de creación cuando mapping exige reconstitution bypass.

---

# 214. Readonly tests

Configuraciones válidas funcionan.

Configuraciones imposibles fallan durante plan compilation cuando sea posible.

---

# 215. Type tests

Cubrir:

```text
int
string
bool
float
decimal value object
UUID
enum
DateTimeImmutable
JSON
custom type
```

---

# 216. Missing vs null test

```text
MISSING
```

nunca se confunde con:

```text
NULL
```

---

# 217. Circular graph test

```text
A → B → A
```

produce:

```text
A1 → B1 → A1
```

No:

```text
A1 → B1 → A2
```

---

# 218. JOIN duplication test

Multiplicación de rows no duplica objetos ni edges.

---

# 219. LEFT JOIN test

No crea entidad related cuando identity está ausente.

---

# 220. Collection completeness test

JOIN filtrado no marca colección completa.

---

# 221. Snapshot test

Snapshot corresponde al estado DB previo a postLoad.

---

# 222. postLoad test

Se ejecuta una sola vez por materialización.

---

# 223. postLoad mutation test

Cambio persistente realizado por callback queda dirty.

---

# 224. Proxy test

Hydration inicializa proxy canónico sin crear segundo objeto.

---

# 225. Inheritance test

Base/subclass no generan dos objetos para misma identidad.

---

# 226. Unknown discriminator test

Falla explícitamente.

---

# 227. Reservation cleanup test

Fallo de conversión elimina reservation.

---

# 228. Registration atomicity test

Fallo interno no deja registries inconsistentes.

---

# 229. Streaming tests

Cubrir:

```text
MANAGED
DETACH_AFTER_YIELD
UNMANAGED_READ_ONLY
WINDOWED
```

---

# 230. Persistent worker test

```text
Request A User#42
```

no aparece en:

```text
Request B IdentityMap
```

---

# 231. Concurrency test

Dos hydrations mutables simultáneas en mismo contexto se rechazan bajo policy V1.

---

# 232. Performance tests

Benchmarks mínimos:

```text
simple entity
10 fields
30 fields
composite identifier
custom value types
1:N JOIN
multiple JOINs
100k streaming entities
IdentityMap hit-heavy workload
IdentityMap miss-heavy workload
```

---

# 233. Architectural Invariants

## DB-ORM-ENTITY-HYDRATOR-001

EntityHydrator no ejecutará queries.

## DB-ORM-ENTITY-HYDRATOR-002

EntityHydrator no generará SQL.

## DB-ORM-ENTITY-HYDRATOR-003

EntityHydrator no persistirá entidades.

## DB-ORM-ENTITY-HYDRATOR-004

EntityHydrator no será EntityManager.

## DB-ORM-ENTITY-HYDRATOR-005

EntityHydrator no será UnitOfWork.

## DB-ORM-ENTITY-HYDRATOR-006

EntityHydrator no será IdentityMap.

## DB-ORM-ENTITY-HYDRATOR-007

EntityHydrator utilizará un plan estructurado.

## DB-ORM-ENTITY-HYDRATOR-008

Identifier deberá resolverse antes de construcción cuando sea posible.

## DB-ORM-ENTITY-HYDRATOR-009

Identifier será convertido mediante Type System.

## DB-ORM-ENTITY-HYDRATOR-010

Identifier será canonicalizado antes de formar EntityKey.

## DB-ORM-ENTITY-HYDRATOR-011

EntityKey incluirá IdentityNamespace.

## DB-ORM-ENTITY-HYDRATOR-012

EntityKey utilizará IdentityDomainType.

## DB-ORM-ENTITY-HYDRATOR-013

IdentityMap será consultado antes de crear una entidad managed.

## DB-ORM-ENTITY-HYDRATOR-014

IdentityMap ACTIVE hit reutilizará la instancia.

## DB-ORM-ENTITY-HYDRATOR-015

IdentityMap hit no implicará freshness.

## DB-ORM-ENTITY-HYDRATOR-016

IdentityMap hit no sobrescribirá campos arbitrariamente.

## DB-ORM-ENTITY-HYDRATOR-017

Cambios locales se preservarán en hidratación normal.

## DB-ORM-ENTITY-HYDRATOR-018

Refresh será explícito.

## DB-ORM-ENTITY-HYDRATOR-019

Refresh conservará object identity.

## DB-ORM-ENTITY-HYDRATOR-020

INITIALIZE_MISSING solo inicializará campos permitidos.

## DB-ORM-ENTITY-HYDRATOR-021

MISS deberá establecer reservation cuando el protocolo lo requiera.

## DB-ORM-ENTITY-HYDRATOR-022

Reservation será distinta de EntityState.

## DB-ORM-ENTITY-HYDRATOR-023

Reserved entity no escapará normalmente a API pública.

## DB-ORM-ENTITY-HYDRATOR-024

Reservation permitirá cerrar grafos circulares.

## DB-ORM-ENTITY-HYDRATOR-025

Reservation failure limpiará estado temporal.

## DB-ORM-ENTITY-HYDRATOR-026

Misma EntityKey no podrá activarse con dos objetos.

## DB-ORM-ENTITY-HYDRATOR-027

Construction strategy será explícita.

## DB-ORM-ENTITY-HYDRATOR-028

No se requerirá constructor vacío universal.

## DB-ORM-ENTITY-HYDRATOR-029

Constructor bypass será controlado.

## DB-ORM-ENTITY-HYDRATOR-030

Constructor de creación no será invocado accidentalmente para reconstitution.

## DB-ORM-ENTITY-HYDRATOR-031

Factory hydration requerirá metadata explícita.

## DB-ORM-ENTITY-HYDRATOR-032

Field writers serán precompilables.

## DB-ORM-ENTITY-HYDRATOR-033

Setters no serán usados automáticamente.

## DB-ORM-ENTITY-HYDRATOR-034

Hydration field writes no deberán generar domain semantics accidentalmente.

## DB-ORM-ENTITY-HYDRATOR-035

Typed properties serán respetadas.

## DB-ORM-ENTITY-HYDRATOR-036

Readonly properties serán validadas.

## DB-ORM-ENTITY-HYDRATOR-037

SQL NULL respetará nullability.

## DB-ORM-ENTITY-HYDRATOR-038

MISSING será distinto de NULL.

## DB-ORM-ENTITY-HYDRATOR-039

MISSING no será escrito como null.

## DB-ORM-ENTITY-HYDRATOR-040

Complete entity deberá satisfacer required fields.

## DB-ORM-ENTITY-HYDRATOR-041

Incomplete entity requerirá partial semantics explícitas.

## DB-ORM-ENTITY-HYDRATOR-042

Partial entity mantendrá LoadedFieldSet.

## DB-ORM-ENTITY-HYDRATOR-043

Embedded values no serán entidades por defecto.

## DB-ORM-ENTITY-HYDRATOR-044

Nullable embedded semantics procederán del mapping.

## DB-ORM-ENTITY-HYDRATOR-045

Inheritance discriminator será validado.

## DB-ORM-ENTITY-HYDRATOR-046

Database discriminator no será nombre arbitrario de clase.

## DB-ORM-ENTITY-HYDRATOR-047

Inheritance respetará canonical identity domain.

## DB-ORM-ENTITY-HYDRATOR-048

Base/subclass no podrán producir duplicados de misma identidad.

## DB-ORM-ENTITY-HYDRATOR-049

Proxy y entidad real compartirán EntityKey.

## DB-ORM-ENTITY-HYDRATOR-050

Proxy inicializable será reutilizado.

## DB-ORM-ENTITY-HYDRATOR-051

Proxy hydration no creará segundo objeto.

## DB-ORM-ENTITY-HYDRATOR-052

Relationship assembly utilizará identidad.

## DB-ORM-ENTITY-HYDRATOR-053

Duplicate JOIN rows no crearán duplicate roots.

## DB-ORM-ENTITY-HYDRATOR-054

Duplicate child rows no crearán duplicate child entities.

## DB-ORM-ENTITY-HYDRATOR-055

Set-like collections no recibirán duplicate edges.

## DB-ORM-ENTITY-HYDRATOR-056

LEFT JOIN absent identity no creará related entity.

## DB-ORM-ENTITY-HYDRATOR-057

Collection completeness será explícita.

## DB-ORM-ENTITY-HYDRATOR-058

Filtered JOIN no implicará complete collection.

## DB-ORM-ENTITY-HYDRATOR-059

Relationship fix-up inicial no generará dirty state artificial.

## DB-ORM-ENTITY-HYDRATOR-060

Snapshot representará estado persistente hidratado.

## DB-ORM-ENTITY-HYDRATOR-061

Snapshot baseline precederá postLoad.

## DB-ORM-ENTITY-HYDRATOR-062

postLoad mutation persistente será detectable.

## DB-ORM-ENTITY-HYDRATOR-063

Hydrated DB entity no será registrada como NEW.

## DB-ORM-ENTITY-HYDRATOR-064

Hydration no utilizará persist() para registrar una entidad cargada.

## DB-ORM-ENTITY-HYDRATOR-065

Hydrated entity tendrá protocolo de registro dedicado.

## DB-ORM-ENTITY-HYDRATOR-066

IdentityMap, Snapshot, State y UoW deberán permanecer coherentes.

## DB-ORM-ENTITY-HYDRATOR-067

Entidad anunciada MANAGED deberá estar correctamente registrada.

## DB-ORM-ENTITY-HYDRATOR-068

Internal registration atomicity no será DB transaction.

## DB-ORM-ENTITY-HYDRATOR-069

postLoad se ejecutará después del registro coherente.

## DB-ORM-ENTITY-HYDRATOR-070

Duplicate JOIN row no repetirá postLoad.

## DB-ORM-ENTITY-HYDRATOR-071

Query posterior con REUSE_ONLY no repetirá postLoad.

## DB-ORM-ENTITY-HYDRATOR-072

Refresh utilizará refresh lifecycle.

## DB-ORM-ENTITY-HYDRATOR-073

Proxy first materialization disparará postLoad como máximo según lifecycle contract.

## DB-ORM-ENTITY-HYDRATOR-074

Failure antes de activación abortará reservation.

## DB-ORM-ENTITY-HYDRATOR-075

Failure durante registro no dejará registries parcialmente activos.

## DB-ORM-ENTITY-HYDRATOR-076

postLoad failure será distinto de core hydration failure.

## DB-ORM-ENTITY-HYDRATOR-077

Hydrator no ejecutará rollback automáticamente.

## DB-ORM-ENTITY-HYDRATOR-078

Hydrator no cerrará conexión automáticamente.

## DB-ORM-ENTITY-HYDRATOR-079

Refresh dirty policy será explícita.

## DB-ORM-ENTITY-HYDRATOR-080

Normal query rehydration será distinta de refresh.

## DB-ORM-ENTITY-HYDRATOR-081

Canonicality será distinta de freshness.

## DB-ORM-ENTITY-HYDRATOR-082

Hydrator no detectará automáticamente cambios externos de DB.

## DB-ORM-ENTITY-HYDRATOR-083

Bulk update stale state se resolverá fuera del EntityHydrator.

## DB-ORM-ENTITY-HYDRATOR-084

Read-only semantics serán explícitas.

## DB-ORM-ENTITY-HYDRATOR-085

Unmanaged hydration no prometerá managed canonicality.

## DB-ORM-ENTITY-HYDRATOR-086

Mixing managed/unmanaged tendrá policy explícita.

## DB-ORM-ENTITY-HYDRATOR-087

Streaming no ejecutará flush implícito.

## DB-ORM-ENTITY-HYDRATOR-088

Detach-after-yield tendrá lifetime documentado.

## DB-ORM-ENTITY-HYDRATOR-089

Streaming eager graph completará root antes de yield cuando sea requerido.

## DB-ORM-ENTITY-HYDRATOR-090

Hydrator no añadirá ORDER BY ocultamente.

## DB-ORM-ENTITY-HYDRATOR-091

ResultHydrator gobernará iteración global de rows.

## DB-ORM-ENTITY-HYDRATOR-092

Una row no se asumirá equivalente a una root entity.

## DB-ORM-ENTITY-HYDRATOR-093

Compiled plan será inmutable.

## DB-ORM-ENTITY-HYDRATOR-094

Compiled plan no almacenará entidades.

## DB-ORM-ENTITY-HYDRATOR-095

Compiled plan no almacenará EntityManager.

## DB-ORM-ENTITY-HYDRATOR-096

Compiled plan no capturará tenant mutable.

## DB-ORM-ENTITY-HYDRATOR-097

Field access podrá utilizar índices precompilados.

## DB-ORM-ENTITY-HYDRATOR-098

Metadata no se escaneará innecesariamente por row.

## DB-ORM-ENTITY-HYDRATOR-099

Generated hydration code derivará solo de metadata validada.

## DB-ORM-ENTITY-HYDRATOR-100

Hydration minimizará copias de rows.

## DB-ORM-ENTITY-HYDRATOR-101

Identity lookup buscará complejidad promedio O(1).

## DB-ORM-ENTITY-HYDRATOR-102

Relationship deduplication evitará O(n²) innecesario.

## DB-ORM-ENTITY-HYDRATOR-103

Telemetry distinguirá constructed de reused.

## DB-ORM-ENTITY-HYDRATOR-104

Telemetry podrá medir identity reuse.

## DB-ORM-ENTITY-HYDRATOR-105

Telemetry no expondrá valores sensibles.

## DB-ORM-ENTITY-HYDRATOR-106

Mutable hydration state será scoped.

## DB-ORM-ENTITY-HYDRATOR-107

Current entity nunca será process-global.

## DB-ORM-ENTITY-HYDRATOR-108

Current row nunca será process-global.

## DB-ORM-ENTITY-HYDRATOR-109

Reservations nunca serán process-global.

## DB-ORM-ENTITY-HYDRATOR-110

FrankenPHP requests tendrán aislamiento de hydration state.

## DB-ORM-ENTITY-HYDRATOR-111

RoadRunner jobs/requests tendrán aislamiento de hydration state.

## DB-ORM-ENTITY-HYDRATOR-112

OpenSwoole contexts tendrán aislamiento lógico.

## DB-ORM-ENTITY-HYDRATOR-113

Mismo PersistenceContext no se asumirá concurrent-safe.

## DB-ORM-ENTITY-HYDRATOR-114

Concurrent mutable hydration no soportada se rechazará.

## DB-ORM-ENTITY-HYDRATOR-115

Datos de DB no se asumirán type-safe.

## DB-ORM-ENTITY-HYDRATOR-116

Conversiones inválidas fallarán explícitamente.

## DB-ORM-ENTITY-HYDRATOR-117

Invalid enum value no tendrá fallback silencioso.

## DB-ORM-ENTITY-HYDRATOR-118

Invalid identifier no generará fake EntityKey.

## DB-ORM-ENTITY-HYDRATOR-119

Extensions respetarán canonicalidad.

## DB-ORM-ENTITY-HYDRATOR-120

Extensions no podrán registrar MANAGED fuera del protocolo.

## DB-ORM-ENTITY-HYDRATOR-121

Hydration exception no expondrá full row por defecto.

## DB-ORM-ENTITY-HYDRATOR-122

EntityHydrator no implicará authorization.

## DB-ORM-ENTITY-HYDRATOR-123

EntityHydrator no implicará domain validation success.

## DB-ORM-ENTITY-HYDRATOR-124

EntityHydrator no implicará transaction commit.

## DB-ORM-ENTITY-HYDRATOR-125

EntityHydrator no implicará persistence success.

## DB-ORM-ENTITY-HYDRATOR-126

EntityHydrator no ejecutará lifecycle de persistencia incorrecto.

## DB-ORM-ENTITY-HYDRATOR-127

First materialization y refresh tendrán lifecycle distinto.

## DB-ORM-ENTITY-HYDRATOR-128

Circular graph resolution no dependerá de recursion ilimitada.

## DB-ORM-ENTITY-HYDRATOR-129

Relationship initialization deberá distinguirse de application mutation.

## DB-ORM-ENTITY-HYDRATOR-130

Loaded field state permanecerá externo a la entidad cuando sea posible.

## DB-ORM-ENTITY-HYDRATOR-131

Partial entity no será considerada completa accidentalmente.

## DB-ORM-ENTITY-HYDRATOR-132

Collection partial state no será considerada complete accidentalmente.

## DB-ORM-ENTITY-HYDRATOR-133

Snapshot no convertirá MISSING en null.

## DB-ORM-ENTITY-HYDRATOR-134

IdentityMap ACTIVE será la fuente canónica de object identity.

## DB-ORM-ENTITY-HYDRATOR-135

Reserved identity será una fase interna, no una segunda instancia managed.

## DB-ORM-ENTITY-HYDRATOR-136

Reservation activation requerirá hidratación válida.

## DB-ORM-ENTITY-HYDRATOR-137

Hydration core outcome y lifecycle outcome podrán diferir.

## DB-ORM-ENTITY-HYDRATOR-138

PostLoad failure no se reinterpretará como query no ejecutada.

## DB-ORM-ENTITY-HYDRATOR-139

Refresh baseline será reconstruido bajo policy explícita.

## DB-ORM-ENTITY-HYDRATOR-140

Read-only query no degradará una entidad writable existente silenciosamente.

## DB-ORM-ENTITY-HYDRATOR-141

Managed entity no será reemplazada por detached representation automáticamente.

## DB-ORM-ENTITY-HYDRATOR-142

Streaming memory policy será explícita.

## DB-ORM-ENTITY-HYDRATOR-143

Large result processing podrá limitar managed entity retention.

## DB-ORM-ENTITY-HYDRATOR-144

EntityHydrator no controlará transaction boundaries.

## DB-ORM-ENTITY-HYDRATOR-145

EntityHydrator no controlará connection routing.

## DB-ORM-ENTITY-HYDRATOR-146

EntityHydrator no conocerá PDO.

## DB-ORM-ENTITY-HYDRATOR-147

EntityHydrator no conocerá SQL dialect.

## DB-ORM-ENTITY-HYDRATOR-148

EntityHydrator no conocerá migration system.

## DB-ORM-ENTITY-HYDRATOR-149

EntityHydrator no tendrá dependencia obligatoria de Multitenancy.

## DB-ORM-ENTITY-HYDRATOR-150

IdentityNamespace será suministrado por contexto.

## DB-ORM-ENTITY-HYDRATOR-151

Same type/id en diferentes namespaces no colisionará.

## DB-ORM-ENTITY-HYDRATOR-152

Same object no podrá activarse bajo keys incompatibles.

## DB-ORM-ENTITY-HYDRATOR-153

Cloned entity no será automáticamente considerada canonical instance.

## DB-ORM-ENTITY-HYDRATOR-154

Unserialized entity no recuperará automáticamente managed membership.

## DB-ORM-ENTITY-HYDRATOR-155

Hydration plan validation deberá detectar mappings imposibles temprano.

## DB-ORM-ENTITY-HYDRATOR-156

Entity construction failure no dejará identity reservation activa.

## DB-ORM-ENTITY-HYDRATOR-157

Field assignment failure no dejará entidad parcialmente activa.

## DB-ORM-ENTITY-HYDRATOR-158

Relationship assembly failure deberá preservar consistencia del PersistenceContext.

## DB-ORM-ENTITY-HYDRATOR-159

Toda entidad managed recién materializada tendrá identidad, baseline y registro ORM coherentes.

## DB-ORM-ENTITY-HYDRATOR-160

Para una identidad lógica determinada dentro de un PersistenceContext, EntityHydrator deberá reutilizar la representación canónica existente o fallar explícitamente; nunca podrá crear silenciosamente una segunda representación administrada.

---

# 234. Fórmulas fundamentales

## 234.1 Entity hydration

```text
EntityHydration
=
ResultRow
+
CompiledEntityHydrationPlan
+
PersistenceContext
→
CanonicalEntityRepresentation
```

---

# 235. Identity resolution

```text
EntityIdentityResolution
=
ReadIdentifier
→
Convert
→
Canonicalize
→
EntityKey
→
IdentityMapLookup
```

---

# 236. Canonicality

```text
∀ e1,e2:

SamePersistenceContext(e1,e2)
∧
SameEntityKey(e1,e2)

⇒

e1 === e2
```

---

# 237. Construction condition

```text
MayConstructEntity(k)
=
IdentityMap.lookup(k) = MISS
∧
ReservationAcquired(k)
```

---

# 238. Safe field hydration

```text
SafeFieldHydration
=
ResultSlotResolved
∧
PresenceKnown
∧
TypeConversionValid
∧
WriterCompatible
∧
HydrationPolicyAllowsWrite
```

---

# 239. Complete entity

```text
CompleteEntity
=
RequiredPersistentFields
⊆
LoadedFieldSet
```

---

# 240. Safe activation

```text
SafeActivation
=
IdentityUnique
∧
EntityConstructedOrCanonical
∧
HydrationCompleteEnough
∧
SnapshotPrepared
∧
StateRegistrationPrepared
∧
UoWRegistrationPrepared
∧
RelationshipBaselineValid
```

---

# 241. Snapshot baseline

```text
InitialEntitySnapshot
=
PersistentStateAfterDatabaseHydration
BeforePostLoadPersistentMutation
```

---

# 242. Existing entity protection

```text
NormalHydration
∧
ManagedEntityExists
∧
FieldLocallyModified

⇒

DoNotOverwriteField
```

---

# 243. Refresh

```text
Refresh
=
SameCanonicalObject
+
ExplicitReloadIntent
+
RefreshPolicy
+
NewDatabaseState
+
NewBaseline
```

---

# 244. Relationship canonicality

```text
SameRelatedEntityKey
+
SamePersistenceContext

⇒

SameRelatedObjectInstance
```

---

# 245. Circular graph

```text
SafeCircularHydration
=
IdentityReservation
+
EarlyInternalObjectBinding
+
CanonicalNestedResolution
+
BoundedHydrationScope
```

---

# 246. Managed hydration

```text
ManagedHydration
=
IdentityMapRegistration
+
EntityState(MANAGED)
+
SnapshotBaseline
+
UnitOfWorkAwareness
```

---

# 247. Streaming hydration

```text
SafeStreamingEntityHydration
=
Cursor
+
ExplicitIdentityRetentionPolicy
+
BoundedTemporaryState
+
DeterministicDetachOrClearPolicy
```

---

# 248. Runtime safety

```text
SafeEntityHydrationRuntime
=
ImmutableSharedPlans
∧
ScopedIdentityMap
∧
ScopedHydrationState
∧
ScopedReservations
∧
NoCrossRequestEntities
∧
DeterministicCleanup
```

---

# 249. Master Formula

```text
Database Entity Hydrator System
=
Entity Hydration Requests
+
Compiled Entity Hydration Plans
+
Identifier-First Resolution
+
Canonical Identifier Conversion
+
EntityKey Construction
+
Identity Namespace Resolution
+
Identity Domain Resolution
+
IdentityMap Lookup
+
Existing Entity Protection
+
Identity Reservations
+
Two-Phase Materialization
+
Entity Construction Strategies
+
Reconstitution Semantics
+
Compiled Field Writers
+
Type Conversion
+
Readonly Property Handling
+
Typed Property Enforcement
+
Loaded Field Tracking
+
Embedded Value Hydration
+
Inheritance Resolution
+
Discriminator Validation
+
Proxy Initialization
+
Relationship Assembly
+
Root Deduplication
+
Child Deduplication
+
Collection Completeness
+
Relationship Fix-Up
+
Snapshot Baseline
+
EntityState Registration
+
UnitOfWork Registration
+
Atomic ORM Registration
+
Lifecycle Integration
+
Refresh Semantics
+
Read-Only Hydration
+
Streaming Entity Hydration
+
Failure Recovery
+
Persistent Runtime Isolation
+
Concurrency Governance
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 250. Master Rule

> **En VoltStack, `EntityHydrator` no significa "crear un objeto desde una fila". Significa resolver la identidad lógica representada por un resultado y materializarla de forma coherente dentro del `PersistenceContext`. Si la identidad ya existe, deberá reutilizarse su instancia canónica; si no existe, deberá reservarse antes de construirla. Solo después de completar correctamente tipos, campos, relaciones, snapshot, estado e integración con `UnitOfWork` podrá considerarse activa como entidad administrada.**

---

# 251. Arquitectura resultante

```text
                         Result Row
                             │
                             ▼
                CompiledEntityHydrationPlan
                             │
                             ▼
                    Identifier Reader
                             │
                             ▼
                     Type Conversion
                             │
                             ▼
                 Canonical Identifier
                             │
                             ▼
                        EntityKey
                             │
                             ▼
                       IdentityMap
                      /           \
                   HIT             MISS
                    │                │
                    │            Reservation
                    │                │
                    │           Construction
                    │                │
                    └──────┬─────────┘
                           ▼
                     Field Hydration
                           │
                 ┌─────────┼──────────┐
                 ▼         ▼          ▼
             Embedded   Relations   Proxy
              Values    Assembly   Initialize
                 └─────────┼──────────┘
                           ▼
                    Completeness
                           │
                           ▼
                       Snapshot
                           │
                           ▼
               ORM Registration Coordinator
                           │
               ┌───────────┼────────────┐
               ▼           ▼            ▼
          IdentityMap   EntityState     UoW
               │           │            │
               └───────────┼────────────┘
                           ▼
                        ACTIVE
                           │
                           ▼
                        postLoad
                           │
                           ▼
                 Canonical Managed Entity
```

Esta arquitectura preserva la separación fundamental:

```text
Query Engine
    │
    ▼
Execution
    │
    ▼
Result
    │
    ▼
Hydration
    │
    ▼
Entity
```

mientras el camino de escritura permanece independiente:

```text
Entity
    │
    ▼
Change Tracking
    │
    ▼
UnitOfWork
    │
    ▼
Persistence Engine
    │
    ▼
Query Engine
    │
    ▼
Execution
```

El `EntityHydrator` es, por tanto, la frontera controlada entre:

```text
Relational Result
```

y:

```text
ORM Object World
```

sin permitir que esa frontera rompa identidad, estado o consistencia.

---

# 252. Siguiente documento

```text
137_DATABASE_RESULT_HYDRATION_SYSTEM.md
```

El siguiente documento deberá definir el nivel superior que consume `Result`, itera rows y coordina las diferentes estrategias de hidratación:

```text
ResultHydrator
ResultHydrationRequest
ResultHydrationPlan
ResultShape
Result cardinality
row iteration
root grouping
root deduplication
entity collections
scalar results
tuples
projections
DTO results
mixed results
JOIN row amplification
relationship graph assembly
buffered hydration
streaming hydration
single-result semantics
first/one/one-or-null semantics
duplicate result detection
cursor integration
resource release
hydration cancellation
partial failures
memory governance
telemetry
persistent runtime isolation
```

Su regla central será:

> **`ResultHydrator` controla cómo un conjunto de rows se convierte en un resultado lógico; una row física no equivale necesariamente a un resultado lógico y la cardinalidad del resultado deberá determinarse después de aplicar la semántica del `HydrationPlan`.**