# 291_DATABASE_ORM_TESTING_SYSTEM.md

# VoltStack Quantum Database
## Database ORM Testing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 291 — Database ORM Testing System  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `290_DATABASE_SCHEMA_TESTING_SYSTEM.md`  
**Siguiente documento:** `292_DATABASE_DRIVER_CONFORMANCE_TESTING_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial del **Database ORM Testing System** de VoltStack.

Su objetivo es establecer cómo deberán probarse, de forma aislada e integrada, los componentes que forman el ORM:

```text
Entity
Metadata
Mapping
Model API
EntityManager
Repository
Entity Query
Entity State
IdentityMap
UnitOfWork
Change Tracking
Snapshots
Persistence Engine
Persistence Planner
Insert / Update / Delete
Flush
Hydration
Relationships
Lazy / Eager / Batch Loading
N+1 Detection
Type Conversion
Transactions
Locking
Cache
Events
Multitenancy
Sharding
Persistent Runtimes
```

El sistema deberá demostrar tanto las invariantes internas del ORM como su interacción real con los DBMS soportados.

La regla central será:

> **Una prueba del ORM deberá distinguir el estado del objeto PHP, el estado conocido por el EntityManager/UnitOfWork, las operaciones de persistencia planificadas, los statements ejecutados, el resultado transaccional y el estado realmente observable en la base de datos; ninguno de estos niveles será tratado automáticamente como equivalente a los demás.**

Formalmente:

```text
ObjectState
≠
EntityState
≠
UnitOfWorkState
≠
ChangeSet
≠
PersistencePlan
≠
ExecutedStatement
≠
TransactionOutcome
≠
DatabaseReality
```

---

# 2. Alcance

El ORM Testing System cubrirá:

```text
ORM Testing
├── Entity Testing
├── Metadata Testing
├── Mapping Testing
├── Model API Testing
├── EntityManager Testing
├── Repository Testing
├── Entity Query Testing
├── Entity State Testing
├── IdentityMap Testing
├── UnitOfWork Testing
├── Change Tracking Testing
├── Snapshot Testing
├── Persistence Testing
├── Flush Testing
├── Hydration Testing
├── Relationship Testing
├── Loading Strategy Testing
├── N+1 Testing
├── Type Integration Testing
├── Transaction Integration Testing
├── Locking Testing
├── Cache Integration Testing
├── Event Testing
├── Tenant Testing
├── Shard Testing
├── Runtime Isolation Testing
└── Real DBMS ORM Testing
```

---

# 3. ORM Testing ≠ Database Testing genérico

Una prueba ORM responde preguntas como:

```text
¿La entidad queda MANAGED?

¿IdentityMap reutiliza la instancia?

¿El ChangeSet es correcto?

¿flush() genera la operación semántica correcta?

¿Hydrator preserva identidad?

¿La relación se carga correctamente?

¿Optimistic Locking detecta conflicto?
```

Mientras una prueba genérica de Database puede comprobar:

```text
¿la conexión funciona?

¿el statement fue ejecutado?

¿el DBMS acepta RETURNING?

¿la transacción fue confirmada?
```

Ambos niveles deberán integrarse sin confundirse.

---

# 4. ORM ≠ Query Engine

Regla arquitectónica:

```text
ORM
 ↓
Query Engine
 ↓
Execution Engine
 ↓
Connection
 ↓
Driver
 ↓
DBMS
```

Nunca:

```text
Driver
 ↓
ORM
```

Las pruebas deberán reforzar esta dirección.

---

# 5. Dual API, Single Engine

VoltStack podrá ofrecer:

```text
Model API
```

y:

```text
EntityManager / Repository API
```

pero ambas deberán utilizar:

```text
same Metadata
same IdentityMap
same UnitOfWork
same Persistence Engine
same Query Engine
```

---

# 6. Active Record ≠ segundo ORM

Ejemplo:

```php
$user = User::find(10);
$user->name = 'Ana';
$user->save();
```

deberá converger conceptualmente en:

```text
Model API
   ↓
Scoped ORM Context
   ↓
EntityManager
   ↓
UnitOfWork
   ↓
Persistence Engine
```

---

# 7. Data Mapper API

Ejemplo:

```php
$user = $entityManager->find(User::class, 10);

$user->rename('Ana');

$entityManager->flush();
```

utilizará el mismo motor.

---

# 8. Test Matrix

VoltStack dividirá ORM Testing en:

| Nivel | Objetivo |
|---|---|
| Unit | invariantes locales |
| Component | colaboración ORM interna |
| Integration | ORM + Query/Execution |
| DB Integration | ORM + DBMS real |
| Conformance | comportamiento entre plataformas |
| Runtime | aislamiento/reuse |
| Failure | errores/incertidumbre |
| Performance | comportamiento bajo carga |

---

# 9. Pirámide de evidencia

```text
                  Real DBMS ORM Tests
                         ▲
                         │
                 Conformance Tests
                         ▲
                         │
                 Integration Tests
                         ▲
                         │
                 Component Tests
                         ▲
                         │
                    Unit Tests
```

---

# 10. Entity Testing

Las entidades deberán poder probarse como objetos de dominio sin base de datos cuando su comportamiento no dependa de persistencia.

Ejemplo:

```php
$user = new User('Ana');

$user->rename('Laura');

$this->assertSame('Laura', $user->name());
```

No deberá necesitar:

```text
EntityManager
Connection
Driver
DBMS
```

---

# 11. Entity ≠ Persistence Test

Si una entidad contiene reglas de dominio:

```text
activate()
deactivate()
rename()
changeEmail()
```

éstas deberán poder probarse independientemente del ORM.

---

# 12. Persistence ignorance

Una entidad Data Mapper pura no deberá necesitar conocer:

```text
SQL
Connection
EntityManager
UnitOfWork
```

para ejecutar comportamiento de dominio.

---

# 13. Model API Testing

Los modelos estilo Laravel podrán tener APIs como:

```php
User::query();
User::find(10);

$user->save();
$user->delete();
```

pero los tests deberán verificar que dichas APIs delegan al ORM canónico.

---

# 14. No static ORM state

Debe comprobarse:

```text
static convenience API
≠
static mutable EntityManager
```

---

# 15. ModelContextResolver

Los tests deberán demostrar:

```text
User::find()
   ↓
ModelContextResolver
   ↓
Current Operation Scope
   ↓
EntityManager
```

---

# 16. Context isolation

Dos scopes diferentes:

```text
Scope A
Scope B
```

no deberán compartir accidentalmente:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
```

---

# 17. Metadata Testing

Entity Metadata deberá probarse principalmente mediante Unit Tests.

Ejemplo:

```php
$metadata = $metadataFactory->for(User::class);

$this->assertSame('users', $metadata->table()->name());
```

---

# 18. Metadata properties

Deberán probarse:

```text
EntityType
Table Mapping
Identifier
Fields
Types
Relationships
Inheritance
Lifecycle Hooks
Version Field
Soft Delete
Tenant Metadata
```

cuando correspondan.

---

# 19. Metadata determinism

La misma definición deberá producir:

```text
same normalized metadata
```

bajo la misma configuración.

---

# 20. Metadata immutability

Metadata compilada deberá ser:

```text
immutable
```

y segura para compartir cuando su diseño así lo permita.

---

# 21. Metadata cache

Tests deberán separar:

```text
Metadata Compilation
```

de:

```text
Metadata Cache
```

---

# 22. Mapping Testing

Deberá comprobarse:

```text
PHP property
↔
ORM field
↔
logical database field
```

---

# 23. Mapping ≠ Schema

Una entidad mapeada a:

```text
users.email
```

no demuestra que la columna exista físicamente.

---

# 24. Mapping Validation

Casos inválidos:

```text
missing identifier
duplicate field
duplicate column mapping
invalid relationship
unknown type
invalid cascade
invalid version field
conflicting mapping
```

deberán generar errores deterministas.

---

# 25. Attribute Mapping

Ejemplo:

```php
#[Entity]
#[Table('users')]
final class User
{
    #[Id]
    #[Column(type: 'uuid')]
    private UserId $id;
}
```

deberá poder convertirse en metadata normalizada.

---

# 26. Attribute syntax ≠ Metadata semantics

Los tests deberán preferir metadata normalizada sobre reflection raw cuando se pruebe el significado ORM.

---

# 27. Entity State Testing

Estados conceptuales:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

---

# 28. State transitions

Ejemplo:

```text
NEW
 ↓ persist()
MANAGED
 ↓ mutation
DIRTY
 ↓ flush()
MANAGED
 ↓ remove()
REMOVED
 ↓ flush()
DETACHED
```

Los detalles finales dependerán del lifecycle definido, pero las transiciones deberán ser explícitas.

---

# 29. persist() ≠ INSERT

Regla crítica:

```text
persist(entity)
```

deberá probarse como:

```text
register/manage entity
```

no como SQL inmediato.

---

# 30. remove() ≠ DELETE inmediato

Igualmente:

```text
remove(entity)
```

programa intención de eliminación.

---

# 31. flush() ≠ commit()

Debe existir una suite específica que garantice:

```text
flush()
≠
transaction commit
```

---

# 32. EntityManager Testing

EntityManager es coordinador.

No deberá convertirse en God Object.

Los tests comprobarán sus responsabilidades de coordinación.

---

# 33. EntityManager lifecycle

Estados:

```text
OPEN
TAINTED
CLOSED
```

---

# 34. OPEN

Permite operaciones normales.

---

# 35. TAINTED

Se utilizará cuando VoltStack ya no pueda afirmar con seguridad que el estado ORM es coherente.

---

# 36. CLOSED

Rechazará nuevas operaciones que requieran manager activo.

---

# 37. Tainted EntityManager Tests

Casos:

```text
unknown transaction outcome
failed flush with uncertain DB state
connection failure during critical persistence
```

deberán probar transición a `TAINTED` cuando corresponda.

---

# 38. clear()

Debe probarse:

```text
clear()
 ↓
IdentityMap cleared
UnitOfWork cleared
managed state detached/reset
```

según contrato.

---

# 39. clear() ≠ database rollback

No deberá modificar por sí mismo la base de datos.

---

# 40. Repository Testing

Repository deberá probar:

```text
find
findOneBy
findBy
query construction
domain-specific queries
```

sin convertirse en SQL generator.

---

# 41. Repository ≠ Query Builder

Repository utiliza el Query Engine.

No lo reemplaza.

---

# 42. Repository Unit Tests

Podrán usar:

```text
FakeEntityQueryEngine
SyntheticResult
```

para comprobar coordinación.

---

# 43. Repository Integration Tests

Deberán utilizar DBMS real cuando se quiera demostrar:

```text
actual query execution
actual mapping
actual hydration
```

---

# 44. Entity Query Testing

Flujo:

```text
Entity Query
    ↓
Metadata Resolution
    ↓
Query Model
    ↓
Query Engine
    ↓
Execution
    ↓
Hydration
```

---

# 45. Entity Query ≠ SQL

Assertions deberán preferir:

```text
semantic query model
```

antes que SQL cuando la propiedad sea ORM.

---

# 46. SQL Compiler Tests

La representación SQL pertenece a suites del Query/Compiler subsystem.

---

# 47. IdentityMap Testing

Invariante principal:

```text
same EntityType
+
same Identifier
+
same effective database context
=
same managed object instance
```

dentro del mismo scope.

---

# 48. IdentityMap test

```php
$a = $em->find(User::class, 10);
$b = $em->find(User::class, 10);

$this->assertSame($a, $b);
```

---

# 49. IdentityMap ≠ Entity Cache

Dos EntityManagers independientes no tienen por qué devolver la misma instancia PHP.

---

# 50. Context identity

La identidad podrá incluir:

```text
EntityType
Identifier
Logical Database
Tenant
Shard
```

cuando corresponda.

---

# 51. Cross-tenant identity

Nunca deberá ocurrir:

```text
Tenant A User#10
===
Tenant B User#10
```

en IdentityMap.

---

# 52. Composite IDs

Deberán normalizarse determinísticamente.

---

# 53. Identity reservation

Hydration con ciclos podrá utilizar:

```text
RESERVED
→
ACTIVE
```

Los tests deberán cubrirlo.

---

# 54. Duplicate row hydration

JOINs pueden producir varias filas para la misma entidad.

La IdentityMap deberá deduplicar identidad.

---

# 55. UnitOfWork Testing

UnitOfWork será uno de los componentes con mayor cobertura unitaria.

---

# 56. UoW ≠ Transaction

Regla:

```text
UnitOfWork
≠
Database Transaction
```

---

# 57. UoW responsibilities

Tests deberán cubrir:

```text
entity registration
state tracking
change detection
scheduled insert
scheduled update
scheduled delete
relationship changes
ordering dependencies
flush coordination
```

---

# 58. ChangeSet Testing

Ejemplo:

```text
Before:
name = Ana

After:
name = Laura

ChangeSet:
name: Ana → Laura
```

---

# 59. ChangeSet ≠ SQL

El ChangeSet es semántico.

---

# 60. No-op ChangeSet

Si:

```text
old == new
```

no deberá generarse UPDATE innecesario cuando la estrategia pueda detectarlo.

---

# 61. Snapshot Testing

Entity Snapshot representa baseline conocido.

---

# 62. Snapshot ≠ Entity clone

No deberá asumirse que snapshot es una copia completa del objeto.

---

# 63. Partial entity snapshots

Deberán respetar:

```text
LoadedFieldMask
```

---

# 64. Unloaded field

No deberá interpretarse como:

```text
null
```

---

# 65. DB NULL ≠ Unloaded

Regla crítica:

```text
DB NULL
≠
FIELD NOT LOADED
```

---

# 66. Change Tracking Strategies

Podrán probarse estrategias como:

```text
SNAPSHOT
EXPLICIT
NOTIFY
CUSTOM
```

si VoltStack las soporta.

---

# 67. Dirty detection

Deberá evitar falsos positivos causados por:

```text
type normalization
value object reconstruction
date representation
```

---

# 68. Persistence Engine Testing

Flujo:

```text
ChangeSet
   ↓
Persistence Planner
   ↓
Persistence Operations
   ↓
Query Models
   ↓
Query Engine
```

---

# 69. Persistence Engine ≠ SQL Compiler

No generará SQL directamente.

---

# 70. Insert Persistence Tests

Casos:

```text
new entity
generated identifier
assigned identifier
default columns
nullable fields
relationships
version field
```

---

# 71. Generated identifier

Deberá probarse:

```text
INSERT
 ↓
DB-generated identity
 ↓
identifier extraction
 ↓
entity identity update
 ↓
IdentityMap registration
```

---

# 72. Generated ID barrier

Si otra entidad depende del ID generado:

```text
Parent INSERT
   ↓
Generated ID
   ↓
Child INSERT
```

el planner deberá respetar la dependencia.

---

# 73. Update Persistence Tests

Deberán probar:

```text
dirty fields only
version predicates
type conversion
nullable transitions
relationship ownership
```

---

# 74. Delete Persistence Tests

Deberán cubrir:

```text
physical delete
soft delete integration
relationship dependencies
cascade policy
optimistic version
```

---

# 75. Soft Delete ≠ Physical Delete

Las suites deberán distinguir ambos comportamientos.

---

# 76. Flush Testing

`flush()` es una frontera crítica.

Flujo:

```text
EntityManager
    ↓
UnitOfWork
    ↓
ChangeSets
    ↓
Persistence Planner
    ↓
Query Models
    ↓
Execution
    ↓
ORM State Reconciliation
```

---

# 77. Unit Flush Tests

Podrán utilizar:

```text
FakeExecutor
```

para comprobar planificación y reconciliación.

---

# 78. Integration Flush Tests

Deberán utilizar:

```text
real Query Engine
real Connection
real Driver
real DBMS
```

para garantías físicas.

---

# 79. Strong persistence test

Patrón recomendado:

```text
create entity
 ↓
persist()
 ↓
flush()
 ↓
clear()
 ↓
reload()
 ↓
assert
```

---

# 80. Why clear()

Sin:

```text
clear()
```

`find()` podría devolver la misma instancia desde IdentityMap.

---

# 81. Weak persistence test

Este patrón:

```php
$em->persist($user);
$em->flush();

$this->assertSame(
    $user,
    $em->find(User::class, $user->id())
);
```

no demuestra por sí mismo que el estado fue recuperado del DBMS.

---

# 82. Stronger test

```php
$em->persist($user);
$em->flush();

$id = $user->id();

$em->clear();

$reloaded = $em->find(User::class, $id);

$this->assertSame('Ana', $reloaded->name());
```

---

# 83. Strongest commit verification

Cuando se necesite probar commit:

```text
Connection A
 ↓
persist / flush / commit
 ↓
Connection B
 ↓
read
```

---

# 84. flush success ≠ commit success

Especialmente con transacción externa.

---

# 85. Statement success ≠ transaction success

Debe mantenerse esta separación.

---

# 86. Transaction success ≠ ORM knowledge automatically

Después de determinados errores, ORM state puede necesitar reconciliación.

---

# 87. Hydration Testing

Hydration deberá tener suites especializadas.

---

# 88. Hydration ≠ Execution

Tests unitarios pueden proporcionar rows sintéticas.

---

# 89. Entity Hydration

Deberá comprobar:

```text
row
→
type conversion
→
entity construction
→
field assignment
→
IdentityMap
→
snapshot
→
EntityState
```

---

# 90. Existing managed entity

Si una entidad ya está activa en IdentityMap:

```text
hydration
```

deberá reutilizarla según política.

---

# 91. Dirty managed entity

Hydration normal no deberá sobrescribir silenciosamente cambios locales dirty.

---

# 92. Hydration assignment ≠ user mutation

Asignar datos durante hidratación no deberá marcar automáticamente la entidad como dirty.

---

# 93. postLoad

El baseline deberá establecerse en el punto definido por lifecycle.

Si `postLoad` muta una propiedad posteriormente, dicha mutación deberá poder detectarse.

---

# 94. Partial Entity Testing

Si VoltStack permite partial entities:

```text
LoadedFieldMask
```

será obligatorio.

---

# 95. Partial entity ≠ complete entity

No deberá tratarse como completa.

---

# 96. Missing identifier

Una entidad managed no deberá hidratarse sin identidad suficiente.

---

# 97. Projection preferred

Cuando sólo se requiera:

```text
id
name
```

una Projection/DTO podrá ser preferible a partial entity.

---

# 98. Scalar Hydration

Deberá probar:

```text
single scalar
scalar list
NULL
typed scalar
```

---

# 99. Tuple Hydration

Deberá probar:

```text
ordered values
aliases
entity + scalar
multiple entities
```

según contrato.

---

# 100. Relationship Testing

Tipos:

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
Polymorphic
```

---

# 101. Owning side

Las pruebas deberán demostrar qué lado tiene autoridad de persistencia.

---

# 102. Inverse side mutation

Modificar únicamente el inverse side no deberá generar persistencia no definida.

---

# 103. Relationship metadata

Deberá probar:

```text
ownership
join columns
target type
cascade
loading strategy
orphan policy
```

---

# 104. One-to-One

Casos:

```text
optional
required
owning
inverse
replacement
removal
```

---

# 105. One-to-Many

Deberá probar:

```text
collection initialization
add/remove
orphan semantics
batch loading
```

---

# 106. Many-to-One

Deberá probar:

```text
FK ownership
target identity
nullable relation
```

---

# 107. Many-to-Many

La prueba deberá tratar la relación como:

```text
membership relation
```

no simplemente como join table.

---

# 108. Membership persistence

Casos:

```text
add member
remove member
duplicate membership
clear collection
```

---

# 109. Rich association

Si la relación tiene atributos propios:

```text
membership_since
role
position
```

deberá modelarse y probarse como association entity.

---

# 110. Polymorphic Relationship Testing

Deberá probar:

```text
stable alias
identifier
registry resolution
unknown discriminator
```

---

# 111. No persisted FQCN

Por defecto, los tests deberán garantizar que no se dependa de FQCN persistido.

---

# 112. Unknown polymorphic alias

Deberá producir error controlado.

---

# 113. Relationship Loading Testing

Arquitectura:

```text
Relationship Metadata
       ↓
Load Request
       ↓
Loading Strategy
       ↓
Query Engine
       ↓
Execution
       ↓
Hydration
       ↓
IdentityMap
       ↓
Relationship Assembly
```

---

# 114. Lazy Loading Testing

Deberán existir modos:

```text
ALLOW
WARN
FORBID
```

cuando estén configurados.

---

# 115. Lazy load allowed

El acceso a relación no cargada podrá ejecutar query sólo cuando el contexto lo permita.

---

# 116. Detached lazy load

Una entidad detached no deberá recuperar mágicamente un EntityManager global.

---

# 117. Serialization

Serializar una entidad no deberá disparar lazy loading por defecto.

---

# 118. Lazy loading after scope end

Deberá fallar de forma explícita.

---

# 119. Eager Loading Testing

Eager loading expresa:

```text
availability requirement
```

no:

```text
must use JOIN
```

---

# 120. Strategy selection

El planner podrá seleccionar:

```text
JOIN
SELECT_IN
BATCH
```

---

# 121. Root semantics

Eager loading no deberá alterar incorrectamente:

```text
root cardinality
ordering
pagination
```

---

# 122. Constrained eager loading

Una colección cargada parcialmente deberá quedar marcada como:

```text
PARTIAL
```

cuando corresponda.

---

# 123. PARTIAL ≠ EMPTY

Regla:

```text
No rows observed in constrained subset
≠
relationship globally empty
```

---

# 124. Batch Loading Testing

Deberá probar agrupación sólo de cargas compatibles.

---

# 125. Compatibility dimensions

```text
relationship
database
tenant
shard
transaction
read intent
shape
metadata generation
polymorphic type
```

---

# 126. Cross-tenant batching

Deberá rechazarse por defecto.

---

# 127. N+1 Testing

VoltStack deberá probar su detector semántico.

---

# 128. N+1 ≠ repeated SQL strings

La detección utilizará correlación de:

```text
root operation
relationship
owner entities
query pattern
```

---

# 129. N+1 assertion

Podrá existir:

```php
$this->assertNoNPlusOne(function () {
    // ORM scenario
});
```

---

# 130. N+1 confidence

El resultado podrá incluir:

```text
CERTAIN
HIGH
MEDIUM
LOW
```

según evidencia.

---

# 131. Detection ≠ automatic optimization

El detector no deberá reescribir silenciosamente la consulta.

---

# 132. Type Integration Testing

ORM deberá probar integración con:

```text
Type Registry
Value Conversion
Casting
Enum
Value Object
JSON
Date/Time
Custom Types
```

---

# 133. Type round trip

Patrón:

```text
PHP value
 ↓
persist
 ↓
DB value
 ↓
reload
 ↓
PHP value'
```

y comprobar:

```text
SemanticEquivalent(value, value')
```

---

# 134. Exact equality

No siempre será apropiada.

Ejemplo:

```text
timezone normalization
decimal representation
```

---

# 135. JSON Testing

Distinguir:

```text
SQL NULL
JSON null
missing property
```

---

# 136. Temporal Testing

Distinguir:

```text
Instant
LocalDateTime
ZonedDateTime
LocalDate
LocalTime
```

---

# 137. Enum Testing

No depender de ordinal implícito.

---

# 138. Value Object Testing

Deberá probar:

```text
construction
conversion
change tracking
equality semantics
```

---

# 139. Custom Type Testing

Una custom type no deberá ejecutar SQL directamente.

---

# 140. Transaction Integration Testing

ORM deberá probarse dentro y fuera de transacciones.

---

# 141. EntityManager ≠ TransactionManager

El EntityManager podrá participar en una transacción sin poseerla.

---

# 142. Flush inside transaction

```text
BEGIN
 ↓
persist
 ↓
flush
 ↓
ROLLBACK
```

deberá demostrar:

```text
database rollback
```

pero no asumir:

```text
automatic object graph rewind
```

---

# 143. Rollback ≠ Object Graph Rewind

Después de rollback:

```text
entity PHP state
```

puede seguir conteniendo cambios.

---

# 144. ORM rollback policy

Los tests deberán comprobar la política elegida:

```text
clear
detach
taint
refresh
explicit reconciliation
```

según escenario.

---

# 145. Commit Testing

```text
flush()
 ↓
commit()
```

deberá probarse separadamente.

---

# 146. Unknown Commit

Escenario:

```text
COMMIT sent
 ↓
connection lost
```

resultado:

```text
UNKNOWN
```

---

# 147. UNKNOWN ≠ rollback

El ORM no deberá asumir que la transacción falló.

---

# 148. UNKNOWN ORM state

EntityManager podrá quedar:

```text
TAINTED
```

---

# 149. Retry Testing

ORM no deberá repetir arbitrariamente:

```text
INSERT
UPDATE
DELETE
```

después de resultado incierto.

---

# 150. Whole transaction retry

Cuando sea seguro:

```text
recreate transaction boundary
+
replay safe operation
```

---

# 151. Deadlock Testing

Real deadlock behavior requerirá:

```text
multiple real connections
real DBMS
synchronization barriers
```

---

# 152. Optimistic Locking Testing

Ejemplo:

```text
Connection A loads version 4
Connection B loads version 4

B updates → version 5
A updates WHERE version=4

affected rows = 0
```

Resultado:

```text
OptimisticLockConflict
```

---

# 153. Conflict ≠ generic not found

Deberá clasificarse específicamente.

---

# 154. Version update

Después de persistencia exitosa:

```text
entity version
snapshot version
IdentityMap state
```

deberán reconciliarse.

---

# 155. Pessimistic Locking Testing

Requiere:

```text
real transaction
real DBMS
multiple connections
```

para evidencia fuerte.

---

# 156. Lock request ≠ lock acquired

El test deberá distinguir ambos estados.

---

# 157. Lock timeout

Deberá clasificarse correctamente.

---

# 158. Cache Integration Testing

ORM puede interactuar con:

```text
Entity Cache
Metadata Cache
Query Cache
Result Cache
```

pero cada uno conserva semántica distinta.

---

# 159. IdentityMap ≠ Entity Cache

Invariante obligatoria.

---

# 160. Cached entity state

Nunca deberá materializarse como una instancia managed compartida entre scopes.

---

# 161. Cache hit hydration

Un cache hit deberá pasar por mecanismos que preserven:

```text
IdentityMap
types
snapshot
entity state
```

según diseño.

---

# 162. Cache invalidation

Después de commit:

```text
relevant entity cache
```

deberá invalidarse/actualizarse según policy.

---

# 163. No uncommitted publication

Nunca:

```text
flush
 ↓
publish cache
 ↓
rollback
```

dejando estado no confirmado visible.

---

# 164. UNKNOWN transaction

Deberá utilizar invalidación conservadora cuando corresponda.

---

# 165. Event Testing

ORM events:

```text
prePersist
postPersist
preUpdate
postUpdate
preRemove
postRemove
postLoad
flush events
```

deberán probarse.

---

# 166. Event ordering

El orden deberá ser determinista cuando forme parte del contrato.

---

# 167. Event ≠ Transaction outcome

`postUpdate` interno no implica necesariamente:

```text
transaction committed
```

---

# 168. afterCommit

Deberá ser distinto de post-persistence events.

---

# 169. Listener failure

Tests deberán comprobar policy.

---

# 170. afterCommit failure

No puede revertir un commit ya confirmado.

---

# 171. ORM Security Testing

Deberán probarse:

```text
mass assignment policy
field exposure
query input
tenant boundaries
sensitive diagnostics
raw query escape hatches
```

cuando apliquen.

---

# 172. ORM authorization

ORM no será el sistema de autorización completo.

Pero las integraciones deberán comprobar que scopes/autorización se aplican en la frontera definida.

---

# 173. Authorization before pagination

Una query no deberá paginar un conjunto más amplio y filtrar autorización después si ello cambia semántica.

---

# 174. Sensitive fields

Diagnostics no deberán exponer:

```text
password hash
token
secret
credentials
```

---

# 175. Multitenancy ORM Testing

Multitenancy seguirá siendo integración opcional.

---

# 176. Tenant identity

Deberá formar parte del contexto ORM cuando sea relevante.

---

# 177. Tenant EntityManager

Un EntityManager asociado a:

```text
Tenant A
```

no deberá cambiar silenciosamente a:

```text
Tenant B
```

durante su scope.

---

# 178. Tenant IdentityMap

Deberá impedir colisiones entre IDs iguales de tenants diferentes.

---

# 179. Tenant queries

Toda Entity Query tenant-scoped deberá conservar el contexto hasta execution.

---

# 180. Tenant relationship

Cross-tenant relationships deberán rechazarse por defecto salvo feature explícita.

---

# 181. Tenant lazy loading

La carga diferida deberá conservar el tenant original.

---

# 182. Tenant cache

Cache keys deberán incluir tenant cuando la semántica lo requiera.

---

# 183. Tenant test isolation

Los tests deberán utilizar al menos:

```text
Tenant A
Tenant B
```

para detectar leakage.

---

# 184. Sharding ORM Testing

La identidad efectiva podrá incluir:

```text
ShardId
```

---

# 185. Entity routing

Persistencia deberá resolver shard antes de ejecución.

---

# 186. Cross-shard relationship

No deberá asumir transacción ACID global.

---

# 187. Cross-shard flush

Si una UnitOfWork contiene entidades de múltiples shards:

```text
reject
or
explicit distributed policy
```

según configuración.

Nunca deberá fingirse atomicidad.

---

# 188. Shard migration

Mover una entidad entre shards no será tratado como UPDATE normal salvo que exista un sistema explícito.

---

# 189. Read/Write Routing Tests

ORM reads podrán utilizar replicas cuando sean elegibles.

---

# 190. Transaction affinity

Dentro de una transacción:

```text
read/write
```

deberán permanecer en el writer/connection apropiado según política.

---

# 191. Sticky reads

Después de write/commit, una lectura ORM podrá requerir writer temporalmente.

---

# 192. Replica lag

No deberá producir falsa read-your-write consistency.

---

# 193. ORM Failure Testing

Escenarios:

```text
connection failure
statement failure
constraint violation
deadlock
lock timeout
serialization failure
unknown commit
hydration failure
type conversion failure
metadata failure
```

---

# 194. Error classification

ORM deberá preservar la causa estructurada.

---

# 195. Constraint Violation

Podrá mapearse a excepciones semánticas:

```text
UniqueConstraintViolation
ForeignKeyConstraintViolation
NotNullConstraintViolation
CheckConstraintViolation
```

cuando exista evidencia suficiente.

---

# 196. Vendor error ≠ ORM exception leak

El ORM podrá conservar vendor details internamente sin obligar al dominio a depender de ellos.

---

# 197. Persistence failure reconciliation

Después de fallo, los tests deberán comprobar:

```text
UoW state
EntityState
IdentityMap
Transaction state
EntityManager state
```

---

# 198. Failure before execution

Puede ser seguro conservar el manager.

---

# 199. Failure with known rollback

Podrá aplicarse una política definida.

---

# 200. Failure with unknown outcome

Deberá preservar incertidumbre.

---

# 201. No fabricated clean state

Nunca:

```text
UNKNOWN
→
mark all entities clean
```

sin evidencia.

---

# 202. Persistent Runtime Testing

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 203. Request Scope

Cada operación deberá recibir:

```text
new EntityManager
new UnitOfWork
new IdentityMap
new TransactionContext
```

o equivalentes correctamente reseteados según arquitectura.

---

# 204. Worker reuse test

Patrón:

```text
Request A
 ↓
load User#10
 ↓
end scope

Request B
 ↓
load User#10
```

La instancia de A no deberá filtrarse a B.

---

# 205. Dirty entity leak

Una entidad dirty de Request A no deberá aparecer en B.

---

# 206. Transaction leak

Una transacción no terminada deberá:

```text
rollback/reset/discard connection
```

según policy antes de reuse.

---

# 207. Tenant leak

```text
Request A → Tenant A
Request B → Tenant B
```

no deberá conservar Tenant A.

---

# 208. FrankenPHP Testing

FrankenPHP será el runtime persistente principal de referencia.

Deberán ejecutarse secuencias repetidas sobre workers reutilizados.

---

# 209. RoadRunner Testing

La misma suite de lifecycle deberá poder ejecutarse mediante adapter.

---

# 210. OpenSwoole Testing

Deberá incluir concurrencia de coroutines.

---

# 211. Coroutine isolation

```text
Coroutine A
EntityManager A

Coroutine B
EntityManager B
```

no compartirán mutable ORM state.

---

# 212. Serialization Testing

Entidades no deberán serializar accidentalmente:

```text
EntityManager
Connection
Driver
LazyLoader runtime context
```

---

# 213. Queue Integration

Si una entidad se envía a job:

preferir:

```text
EntityIdentifier
```

sobre serializar graph managed completo.

---

# 214. Rehydration in job

El job resolverá una nueva entidad dentro de su propio ORM scope.

---

# 215. Detached Entity Testing

Una entidad detached deberá:

```text
retain domain state
```

pero no actuar como managed automáticamente.

---

# 216. Reattachment

Si VoltStack soporta merge/reattach, deberá tener semántica explícita y suites separadas.

---

# 217. Detached ≠ NEW

No deberán confundirse.

---

# 218. Detached ≠ MANAGED

Igualmente.

---

# 219. ORM Test Data

Podrá utilizar:

```text
EntityFactory
ModelFactory
Fixture
TestDataGenerator
```

---

# 220. Factory ≠ Persistence evidence

`factory()->make()` no demuestra persistencia.

---

# 221. create()

Si:

```php
User::factory()->create();
```

internamente persiste, la prueba deberá distinguir:

```text
construction
persistence
flush
transaction
```

---

# 222. Deterministic test data

Factories deberán aceptar:

```text
seed
clock
locale
```

cuando corresponda.

---

# 223. Production PII

No deberá utilizarse como fixture por defecto.

---

# 224. ORM Fixtures

Podrán representar graphs como:

```text
User
├── Profile
├── Orders
│   └── Items
└── Roles
```

---

# 225. Fixture references

Deberán utilizar aliases estables.

---

# 226. Test Isolation

Opciones:

```text
transaction rollback
database reset
schema reset
fixture cleanup
```

---

# 227. Transaction rollback limitation

No sirve para pruebas que necesitan demostrar:

```text
commit visibility
afterCommit
multiple connection visibility
connection loss after commit
```

---

# 228. Integration DB reset

Para esos escenarios podrá utilizarse:

```text
isolated database/schema
```

---

# 229. Parallel ORM Tests

No deberán compartir:

```text
mutable fixtures
same primary keys without isolation
same EntityManager
same transaction
```

---

# 230. Unique scenario namespace

Cada worker podrá recibir:

```text
TestRunId
WorkerId
ScenarioId
```

---

# 231. ORM Assertions

VoltStack podrá ofrecer assertions como:

```php
$this->assertEntityManaged($user);
$this->assertEntityDirty($user);
$this->assertEntityRemoved($user);
$this->assertEntityDetached($user);

$this->assertScheduledForInsert($user);
$this->assertScheduledForUpdate($user);
$this->assertScheduledForDelete($user);

$this->assertIdentityMapContains($user);
$this->assertSameManagedEntity($a, $b);

$this->assertChangeSetContains(
    $user,
    'name',
    'Ana',
    'Laura'
);
```

---

# 232. Database assertions

También:

```php
$this->assertEntityPersisted(User::class, $id);
$this->assertEntityNotPersisted(User::class, $id);
```

pero deberán especificar claramente si usan DBMS real.

---

# 233. ORM State Assertion ≠ DB Assertion

Ejemplo:

```php
assertEntityManaged($user)
```

no implica:

```php
assertEntityPersisted(...)
```

---

# 234. Query Count Assertions

Podrán existir:

```php
$this->assertQueryCount(2, function () {
    // scenario
});
```

principalmente para diagnostics/performance/N+1.

---

# 235. Query count brittleness

No deberá usarse como sustituto universal de assertions semánticas.

---

# 236. No Query Assertion

Útil para comprobar IdentityMap:

```php
$this->assertNoQueries(function () use ($em) {
    $em->find(User::class, 10);
});
```

si la entidad ya está managed.

---

# 237. Relationship Assertions

Podrán existir:

```php
$this->assertRelationshipLoaded($user, 'roles');
$this->assertRelationshipNotLoaded($user, 'orders');
$this->assertRelationshipPartiallyLoaded($user, 'comments');
```

---

# 238. Loaded ≠ Non-empty

Una relación puede estar:

```text
LOADED + EMPTY
```

---

# 239. Not loaded ≠ empty

Regla obligatoria.

---

# 240. ORM Test Diagnostics

Un fallo deberá poder mostrar:

```text
Entity Type
Entity ID
Entity State
Loaded Fields
ChangeSet
UoW Schedule
Transaction State
Database Context
Tenant
Shard
```

según disponibilidad.

---

# 241. Diagnostic redaction

Nunca deberá mostrar secretos automáticamente.

---

# 242. Failure example

```text
ORM assertion failed.

Entity:
App\Domain\User

Identifier:
42

Expected state:
MANAGED

Observed state:
DETACHED

EntityManager:
OPEN

IdentityMap:
entity not registered

Tenant:
tenant_17
```

---

# 243. Persistence failure diagnostic

Podrá incluir:

```text
Persistence Operation
Query Fingerprint
Execution Phase
Transaction Outcome
EntityManager State
```

---

# 244. Raw SQL

Sólo deberá incluirse si:

```text
debug policy
+
redaction policy
```

lo permiten.

---

# 245. Bindings

Datos sensibles deberán redactarse.

---

# 246. ORM Test Evidence

Objeto conceptual:

```php
final readonly class OrmTestEvidence
{
    public function __construct(
        public OrmScenarioId $scenarioId,
        public OrmTestLayer $layer,
        public OrmEnvironmentFingerprint $environment,
        public ?CapabilitySnapshot $capabilities,
        public OrmTestResult $result,
    ) {}
}
```

---

# 247. Test layers

```text
UNIT
COMPONENT
INTEGRATION
REAL_DATABASE
CONFORMANCE
RUNTIME
FAILURE
PERFORMANCE
```

---

# 248. Evidence scope

Una prueba con:

```text
FakeExecutor
```

demuestra:

```text
ORM behavior with FakeExecutor
```

no comportamiento real de PostgreSQL.

---

# 249. Real DB evidence

Deberá registrar:

```text
DBMS
version
driver
PHP version
runtime
capabilities
schema generation
```

---

# 250. DBMS Matrix

El ORM deberá probarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 251. SQLite ≠ universal ORM backend proof

Una suite verde en SQLite no demuestra compatibilidad total con otros DBMS.

---

# 252. MySQL ≠ MariaDB

Ambos deberán tener evidencia separada.

---

# 253. Platform differences

Tests deberán contemplar diferencias en:

```text
generated IDs
RETURNING
locking
transaction isolation
JSON
datetime
DDL
affected rows
```

---

# 254. Capability-conditioned ORM Tests

Ejemplo:

```text
IF supportsReturning(INSERT)
THEN run RETURNING ORM suite
ELSE run fallback ORM suite
```

---

# 255. Unsupported capability

No deberá convertirse automáticamente en test failure si la feature es opcional.

---

# 256. Required capability

Si VoltStack declara una capability como obligatoria para una plataforma soportada y ésta falta:

```text
CONFORMANCE FAILURE
```

---

# 257. UNKNOWN capability

No deberá interpretarse silenciosamente como supported.

---

# 258. ORM Conformance Suite

Podrá definir contratos como:

```text
EntityPersistenceContract
IdentityMapContract
ChangeTrackingContract
RelationshipContract
TransactionContract
HydrationContract
TypeRoundTripContract
LockingContract
```

---

# 259. Driver conformance boundary

La suite ORM no sustituirá:

```text
292_DATABASE_DRIVER_CONFORMANCE_TESTING_SYSTEM.md
```

---

# 260. ORM Performance Testing

Deberá existir integración con:

```text
293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

---

# 261. Performance concerns

Ejemplos:

```text
large IdentityMap
large UnitOfWork
mass hydration
relationship graph hydration
batch persistence
N+1
metadata startup
```

---

# 262. Performance ≠ correctness

Una optimización no podrá romper:

```text
identity
change tracking
transaction semantics
tenant isolation
```

---

# 263. Architecture Tests

Deberán impedir dependencias prohibidas.

---

# 264. Forbidden dependency

```text
ORM
→ Driver implementation directly
```

---

# 265. Forbidden dependency

```text
Entity
→ SQL Compiler
```

---

# 266. Forbidden dependency

```text
UnitOfWork
→ PDO
```

---

# 267. Forbidden dependency

```text
Hydrator
→ Query Executor
```

si el contrato de Hydrator sólo consume resultados.

---

# 268. Forbidden dependency

```text
Repository
→ vendor-specific SQL
```

por defecto.

---

# 269. Static mutable state detection

Architecture tests deberán detectar:

```text
static EntityManager
static IdentityMap
static UnitOfWork
static TransactionContext
static TenantContext
```

---

# 270. Proposed namespace

```text
src/Quantum/Database/Testing/Orm/
├── Contract/
│   ├── OrmTestCase.php
│   ├── OrmScenario.php
│   ├── OrmAssertion.php
│   └── OrmConformanceContract.php
│
├── Assertion/
│   ├── EntityStateAssertion.php
│   ├── IdentityMapAssertion.php
│   ├── UnitOfWorkAssertion.php
│   ├── ChangeSetAssertion.php
│   ├── PersistenceAssertion.php
│   ├── HydrationAssertion.php
│   ├── RelationshipAssertion.php
│   ├── LoadingAssertion.php
│   └── LockingAssertion.php
│
├── Scenario/
│   ├── EntityPersistenceScenario.php
│   ├── EntityUpdateScenario.php
│   ├── EntityRemovalScenario.php
│   ├── EntityReloadScenario.php
│   ├── RelationshipScenario.php
│   ├── OptimisticLockScenario.php
│   ├── PessimisticLockScenario.php
│   ├── TransactionScenario.php
│   ├── TenantScenario.php
│   └── RuntimeIsolationScenario.php
│
├── Fixture/
│   ├── OrmFixture.php
│   ├── EntityGraphFixture.php
│   └── OrmFixtureRegistry.php
│
├── Evidence/
│   ├── OrmTestEvidence.php
│   ├── OrmEnvironmentFingerprint.php
│   └── OrmTestResult.php
│
├── Diagnostic/
│   ├── OrmFailureDiagnostic.php
│   ├── EntityStateFormatter.php
│   ├── UnitOfWorkFormatter.php
│   └── ChangeSetFormatter.php
│
├── Conformance/
│   ├── EntityPersistenceContract.php
│   ├── IdentityMapContract.php
│   ├── ChangeTrackingContract.php
│   ├── HydrationContract.php
│   ├── RelationshipContract.php
│   ├── TransactionContract.php
│   └── TypeRoundTripContract.php
│
└── PHPUnit/
    └── InteractsWithOrm.php
```

Tests:

```text
tests/Quantum/Database/
├── Unit/Orm/
├── Integration/Orm/
├── Conformance/Orm/
├── Runtime/Orm/
├── Failure/Orm/
└── Performance/Orm/
```

---

# 271. Exception hierarchy

```text
DatabaseOrmTestingException
├── EntityAssertionFailedException
├── EntityStateAssertionFailedException
├── IdentityMapAssertionFailedException
├── UnitOfWorkAssertionFailedException
├── ChangeSetAssertionFailedException
├── PersistenceAssertionFailedException
├── HydrationAssertionFailedException
├── RelationshipAssertionFailedException
├── LoadingAssertionFailedException
├── LockingAssertionFailedException
├── OrmScenarioFailedException
├── OrmConformanceException
└── OrmTestInconclusiveException
```

---

# 272. Invariantes

## DB-ORM-TEST-001

Entity será distinta de Database Row.

## DB-ORM-TEST-002

Entity State será distinto de Object State.

## DB-ORM-TEST-003

Entity State será distinto de Database State.

## DB-ORM-TEST-004

UnitOfWork será distinto de Transaction.

## DB-ORM-TEST-005

ChangeSet será distinto de SQL.

## DB-ORM-TEST-006

Persistence Plan será distinto de SQL.

## DB-ORM-TEST-007

Executed Statement será distinto de Transaction Outcome.

## DB-ORM-TEST-008

Transaction Outcome será distinto de ORM Knowledge.

## DB-ORM-TEST-009

Database Reality será distinta de ORM Knowledge.

## DB-ORM-TEST-010

ORM no generará SQL directamente.

## DB-ORM-TEST-011

ORM no dependerá directamente de Driver implementation.

## DB-ORM-TEST-012

Active Record no será un segundo ORM.

## DB-ORM-TEST-013

Model API y EntityManager compartirán motor ORM.

## DB-ORM-TEST-014

Static Model API no implicará static mutable ORM state.

## DB-ORM-TEST-015

Entity domain behavior podrá probarse sin DB.

## DB-ORM-TEST-016

Mapping será distinto de Schema.

## DB-ORM-TEST-017

Metadata será determinista.

## DB-ORM-TEST-018

Compiled Metadata será inmutable.

## DB-ORM-TEST-019

Metadata Cache será distinto de Metadata.

## DB-ORM-TEST-020

persist() no significará INSERT inmediato.

## DB-ORM-TEST-021

remove() no significará DELETE inmediato.

## DB-ORM-TEST-022

flush() no significará commit().

## DB-ORM-TEST-023

clear() no significará rollback.

## DB-ORM-TEST-024

Repository será distinto de Query Builder.

## DB-ORM-TEST-025

Entity Query será distinta de SQL.

## DB-ORM-TEST-026

IdentityMap será scope-local.

## DB-ORM-TEST-027

IdentityMap será distinta de Entity Cache.

## DB-ORM-TEST-028

Misma identidad efectiva devolverá misma instancia managed dentro del scope.

## DB-ORM-TEST-029

Identidad efectiva incluirá tenant/shard cuando corresponda.

## DB-ORM-TEST-030

Tenant A Entity#X será distinto de Tenant B Entity#X.

## DB-ORM-TEST-031

Composite identifiers se normalizarán determinísticamente.

## DB-ORM-TEST-032

Hydration con JOIN no creará identidades duplicadas.

## DB-ORM-TEST-033

UnitOfWork tendrá pruebas de scheduling.

## DB-ORM-TEST-034

No-op ChangeSet no producirá update innecesario cuando pueda detectarse.

## DB-ORM-TEST-035

Entity Snapshot será distinto de Entity clone.

## DB-ORM-TEST-036

LoadedFieldMask será preservado.

## DB-ORM-TEST-037

DB NULL será distinto de unloaded field.

## DB-ORM-TEST-038

Dirty detection respetará semantic equality.

## DB-ORM-TEST-039

Persistence Engine no será SQL Compiler.

## DB-ORM-TEST-040

Generated identifiers serán reconciliados con IdentityMap.

## DB-ORM-TEST-041

Generated ID dependencies serán respetadas.

## DB-ORM-TEST-042

Flush unit test será distinto de real persistence test.

## DB-ORM-TEST-043

Persistence fuerte requerirá clear/reload o nueva conexión según propiedad.

## DB-ORM-TEST-044

IdentityMap hit no demostrará DB persistence.

## DB-ORM-TEST-045

Flush success no demostrará commit success.

## DB-ORM-TEST-046

Statement success no demostrará transaction success.

## DB-ORM-TEST-047

Hydration será distinta de Query Execution.

## DB-ORM-TEST-048

Hydration reutilizará managed identity.

## DB-ORM-TEST-049

Hydration no sobrescribirá dirty state silenciosamente.

## DB-ORM-TEST-050

Hydration assignment no será user mutation.

## DB-ORM-TEST-051

Partial entity será distinta de complete entity.

## DB-ORM-TEST-052

Managed entity requerirá identidad suficiente.

## DB-ORM-TEST-053

Projection será preferible a partial entity cuando corresponda.

## DB-ORM-TEST-054

Relationship persistence tendrá owning side explícito.

## DB-ORM-TEST-055

Inverse side no tendrá autoridad implícita.

## DB-ORM-TEST-056

Many-to-Many será membership relation.

## DB-ORM-TEST-057

Rich association será association entity.

## DB-ORM-TEST-058

Polymorphic mapping usará aliases estables.

## DB-ORM-TEST-059

FQCN no se persistirá por defecto como discriminator.

## DB-ORM-TEST-060

Unknown polymorphic alias será rechazado.

## DB-ORM-TEST-061

Lazy Loading requerirá contexto válido.

## DB-ORM-TEST-062

Detached entity no resolverá EntityManager global.

## DB-ORM-TEST-063

Serialization no disparará lazy loading por defecto.

## DB-ORM-TEST-064

Eager Loading no implicará JOIN.

## DB-ORM-TEST-065

Eager Loading preservará root semantics.

## DB-ORM-TEST-066

PARTIAL relationship será distinta de EMPTY relationship.

## DB-ORM-TEST-067

Not-loaded relationship será distinta de empty relationship.

## DB-ORM-TEST-068

Batch loading agrupará sólo requests compatibles.

## DB-ORM-TEST-069

Cross-tenant batching será rechazado por defecto.

## DB-ORM-TEST-070

N+1 detection será semántica.

## DB-ORM-TEST-071

N+1 detection será distinta de automatic optimization.

## DB-ORM-TEST-072

Type round-trip tendrá integration tests.

## DB-ORM-TEST-073

SQL NULL será distinto de JSON null.

## DB-ORM-TEST-074

JSON null será distinto de missing JSON path.

## DB-ORM-TEST-075

Temporal types mantendrán semántica diferenciada.

## DB-ORM-TEST-076

Enum mapping no dependerá de ordinal implícito.

## DB-ORM-TEST-077

Custom Type no ejecutará SQL directamente.

## DB-ORM-TEST-078

EntityManager será distinto de TransactionManager.

## DB-ORM-TEST-079

Rollback será distinto de object graph rewind.

## DB-ORM-TEST-080

Commit UNKNOWN permanecerá UNKNOWN.

## DB-ORM-TEST-081

UNKNOWN no será convertido en rollback.

## DB-ORM-TEST-082

UNKNOWN podrá taint EntityManager.

## DB-ORM-TEST-083

Retry no repetirá statements arbitrariamente.

## DB-ORM-TEST-084

Deadlock real requerirá integración concurrente.

## DB-ORM-TEST-085

Optimistic conflict será distinto de generic not found.

## DB-ORM-TEST-086

Optimistic version será reconciliada tras éxito.

## DB-ORM-TEST-087

Pessimistic lock request será distinto de lock acquired.

## DB-ORM-TEST-088

IdentityMap será distinta de second-level cache.

## DB-ORM-TEST-089

Cache no almacenará managed objects cross-scope.

## DB-ORM-TEST-090

Uncommitted state no se publicará a cache compartida.

## DB-ORM-TEST-091

Rollback cancelará publicación/invalidation correspondiente según policy.

## DB-ORM-TEST-092

UNKNOWN outcome provocará tratamiento conservador de cache.

## DB-ORM-TEST-093

ORM event será distinto de transaction outcome.

## DB-ORM-TEST-094

postPersist no significará committed.

## DB-ORM-TEST-095

afterCommit failure no deshará commit.

## DB-ORM-TEST-096

Diagnostics ORM redactarán secretos.

## DB-ORM-TEST-097

Multitenancy será integración opcional.

## DB-ORM-TEST-098

Tenant context será estable durante scope ORM.

## DB-ORM-TEST-099

Tenant lazy loading conservará tenant original.

## DB-ORM-TEST-100

Tenant cache keys serán aisladas cuando corresponda.

## DB-ORM-TEST-101

Shard identity será preservada cuando sea relevante.

## DB-ORM-TEST-102

Cross-shard flush no fingirá ACID global.

## DB-ORM-TEST-103

Entity routing ocurrirá antes de ejecución.

## DB-ORM-TEST-104

Transaction affinity será respetada.

## DB-ORM-TEST-105

Replica read no implicará read-your-write automáticamente.

## DB-ORM-TEST-106

Persistence errors conservarán clasificación estructurada.

## DB-ORM-TEST-107

Vendor errors no definirán la API ORM pública.

## DB-ORM-TEST-108

Failure reconciliation evaluará UoW, IdentityMap y EntityManager.

## DB-ORM-TEST-109

UNKNOWN failure no fabricará clean state.

## DB-ORM-TEST-110

Persistent runtime no compartirá EntityManager entre requests.

## DB-ORM-TEST-111

Persistent runtime no compartirá UnitOfWork entre requests.

## DB-ORM-TEST-112

Persistent runtime no compartirá IdentityMap entre requests.

## DB-ORM-TEST-113

Persistent runtime no filtrará transaction context.

## DB-ORM-TEST-114

Persistent runtime no filtrará tenant context.

## DB-ORM-TEST-115

FrankenPHP tendrá worker-reuse ORM tests.

## DB-ORM-TEST-116

RoadRunner tendrá worker-reuse ORM tests.

## DB-ORM-TEST-117

OpenSwoole tendrá coroutine isolation ORM tests.

## DB-ORM-TEST-118

Entity serialization no incluirá EntityManager.

## DB-ORM-TEST-119

Entity serialization no incluirá Connection.

## DB-ORM-TEST-120

Jobs deberán preferir identifiers a managed graph serialization.

## DB-ORM-TEST-121

Detached será distinto de NEW.

## DB-ORM-TEST-122

Detached será distinto de MANAGED.

## DB-ORM-TEST-123

Factory make() no demostrará persistence.

## DB-ORM-TEST-124

Test data será determinista cuando corresponda.

## DB-ORM-TEST-125

Production PII no será test fixture por defecto.

## DB-ORM-TEST-126

Transactional test isolation no demostrará commit visibility.

## DB-ORM-TEST-127

Parallel ORM tests estarán aislados.

## DB-ORM-TEST-128

ORM state assertions serán distintas de DB assertions.

## DB-ORM-TEST-129

Query count no sustituirá assertions semánticas.

## DB-ORM-TEST-130

Relationship loaded será distinto de relationship non-empty.

## DB-ORM-TEST-131

Diagnostics incluirán entity state cuando sea seguro.

## DB-ORM-TEST-132

Diagnostics incluirán UoW state cuando sea seguro.

## DB-ORM-TEST-133

Test evidence registrará su layer.

## DB-ORM-TEST-134

FakeExecutor evidence no será DBMS evidence.

## DB-ORM-TEST-135

Real DB evidence registrará environment fingerprint.

## DB-ORM-TEST-136

MySQL y MariaDB tendrán evidencia separada.

## DB-ORM-TEST-137

SQLite no sustituirá otros DBMS.

## DB-ORM-TEST-138

Capability-conditioned tests usarán Capability System.

## DB-ORM-TEST-139

UNKNOWN capability no será supported.

## DB-ORM-TEST-140

Conformance tests serán reutilizables entre drivers/plataformas.

## DB-ORM-TEST-141

ORM conformance no sustituirá Driver conformance.

## DB-ORM-TEST-142

Performance optimization no romperá correctness.

## DB-ORM-TEST-143

Architecture tests impedirán ORM → Driver implementation.

## DB-ORM-TEST-144

Architecture tests impedirán Entity → SQL Compiler.

## DB-ORM-TEST-145

Architecture tests impedirán UnitOfWork → PDO.

## DB-ORM-TEST-146

Repository no contendrá vendor SQL por defecto.

## DB-ORM-TEST-147

Static mutable ORM state estará prohibido.

## DB-ORM-TEST-148

EntityManager TAINTED no será reutilizado como OPEN.

## DB-ORM-TEST-149

ORM Test System no creará un segundo ORM.

## DB-ORM-TEST-150

La evidencia más fuerte de persistencia requerirá observación independiente del estado managed original.

---

# 273. Anti-patrones

## 273.1 Probar persistencia sólo con IdentityMap

```text
persist
flush
find same ID
```

puede devolver la misma instancia.

No demuestra reload real.

---

## 273.2 Considerar `persist()` como INSERT

Rompe UnitOfWork.

---

## 273.3 Considerar `flush()` como COMMIT

Rompe separación ORM/transacción.

---

## 273.4 Considerar rollback como rewind de objetos

La base de datos y el object graph son estados distintos.

---

## 273.5 Crear un Fake ORM completo

Un fake que intenta imitar todo el ORM puede ocultar las invariantes que deberían probarse realmente.

---

## 273.6 Usar SQLite como fake universal

SQLite es un DBMS real, pero no sustituye MySQL, MariaDB o PostgreSQL.

---

## 273.7 Mockear value objects puros

Preferir objetos reales.

---

## 273.8 Mockear UnitOfWork internamente en todos los tests

Reduce el valor de probar colaboración real entre componentes ORM.

---

## 273.9 SQL assertions para toda prueba ORM

Acopla ORM al Compiler.

---

## 273.10 Compartir EntityManager entre tests

Produce leakage.

---

## 273.11 Compartir IdentityMap entre requests

Incompatible con runtime persistente seguro.

---

## 273.12 Permitir lazy loading después del scope

Oculta errores de lifecycle.

---

## 273.13 Tratar partial collection como completa

Puede producir pérdida de relaciones.

---

## 273.14 Tratar unloaded field como NULL

Puede producir UPDATE destructivo.

---

## 273.15 Sobrescribir dirty entity durante hydration

Puede perder cambios de aplicación.

---

## 273.16 Reintentar INSERT después de commit incierto

Puede duplicar datos.

---

## 273.17 Publicar cache después de flush antes de commit

Puede exponer estado no confirmado.

---

## 273.18 Compartir managed entities mediante cache

Rompe IdentityMap/scope.

---

## 273.19 FQCN como polymorphic discriminator por defecto

Acopla almacenamiento a implementación PHP.

---

## 273.20 Ocultar diferencias entre plataformas

ORM portability no significa DBMS equivalence.

---

# 274. Modelo formal de estado

Sea una entidad:

```text
e
```

Su estado PHP será:

```text
O(e)
```

El estado conocido por UoW:

```text
U(e)
```

Su snapshot:

```text
S(e)
```

y el estado real de la base:

```text
D(e)
```

No deberá asumirse:

```text
O(e) = U(e) = S(e) = D(e)
```

en todo momento.

---

# 275. Dirty detection

Conceptualmente:

```text
Dirty(e)
=
Difference(
    Normalize(O(e)),
    S(e)
)
```

restringido a:

```text
LoadedPersistentFields(e)
```

y a la estrategia de tracking configurada.

---

# 276. Persistence planning

Sea:

```text
Δe = ChangeSet(e)
```

entonces:

```text
PersistencePlan
=
Plan(
    Δe,
    Metadata,
    Relationships,
    Capabilities,
    DatabaseContext
)
```

---

# 277. Flush

Conceptualmente:

```text
Flush(UoW)
=
DetectChanges
→
BuildPersistenceGraph
→
Plan
→
Execute
→
ReconcileKnownOutcome
```

No:

```text
Flush = Commit
```

---

# 278. Strong persistence evidence

Sea:

```text
e1 = original managed entity
```

Después de persistencia:

```text
flush
clear
reload
```

obtenemos:

```text
e2
```

La prueba fuerte compara:

```text
PersistentSemantics(e1)
≈
PersistentSemantics(e2)
```

sin depender de identidad PHP:

```text
e1 === e2
```

---

# 279. IdentityMap property

Dentro de un scope `s`:

```text
Resolve(s, EntityType, ID, Context)
```

deberá satisfacer:

```text
Resolve(...) === Resolve(...)
```

mientras la entidad permanezca managed.

---

# 280. Cross-scope property

Para scopes diferentes:

```text
s1 ≠ s2
```

no deberá requerirse:

```text
Resolve(s1, E, ID) === Resolve(s2, E, ID)
```

y normalmente deberá ser falso.

---

# 281. Transaction evidence

Definimos:

```text
F = flush succeeded
C = commit outcome
```

Entonces:

```text
F = success
```

no implica:

```text
C = committed
```

---

# 282. Unknown commit

Si:

```text
C = UNKNOWN
```

entonces:

```text
DatabaseReality = UNKNOWN
```

hasta obtener evidencia adicional.

El ORM no deberá inferir:

```text
COMMITTED
```

ni:

```text
ROLLED_BACK
```

sin evidencia.

---

# 283. ORM consistency states

Podrán reutilizarse estados como:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

según el sistema de consistencia definido.

---

# 284. CI Strategy

```text
Pull Request
├── Entity Unit Tests
├── Metadata Tests
├── Mapping Tests
├── IdentityMap Tests
├── UnitOfWork Tests
├── Change Tracking Tests
├── Hydration Tests
├── Relationship Tests
├── ORM Architecture Tests
└── Fast SQLite Integration

Main
├── MySQL ORM Integration
├── MariaDB ORM Integration
├── PostgreSQL ORM Integration
├── SQLite ORM Integration
├── Transaction Tests
├── Optimistic Locking
├── Cache Integration
└── Runtime Reset Tests

Nightly
├── DBMS Version Matrix
├── Full ORM Conformance
├── Deadlock Scenarios
├── Pessimistic Locking
├── Failure Injection
├── FrankenPHP Worker Reuse
├── RoadRunner Adapter
├── OpenSwoole Concurrency
├── Multitenancy Isolation
├── Sharding Scenarios
└── Large ORM Workloads
```

---

# 285. Arquitectura consolidada

```text
                       ORM TEST SCENARIO
                              │
                              ▼
                         Entity / Model
                              │
                              ▼
                         EntityManager
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         IdentityMap      UnitOfWork       Repository
              │               │               │
              │               ▼               │
              │          ChangeSets            │
              │               │               │
              │               ▼               │
              │       Persistence Planner      │
              │               │               │
              └───────────────┼───────────────┘
                              ▼
                         Query Models
                              │
                              ▼
                         Query Engine
                              │
                              ▼
                       Execution Engine
                              │
                              ▼
                          Connection
                              │
                              ▼
                            Driver
                              │
                              ▼
                             DBMS
                              │
                              ▼
                         Result System
                              │
                              ▼
                           Hydrator
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                Identity   Snapshot   EntityState
                   Map                   │
                    └─────────┬──────────┘
                              ▼
                       Managed Entity
                              │
                              ▼
                       ORM Assertions
                              │
             ┌────────────────┼─────────────────┐
             ▼                ▼                 ▼
          Memory          ORM State         Database
         Assertions       Assertions        Assertions
             │                │                 │
             └────────────────┼─────────────────┘
                              ▼
                        Evidence Record
                              │
                              ▼
              PASS / FAIL / INCONCLUSIVE
```

---

# 286. Regla final

> **VoltStack deberá probar el ORM como un sistema de identidad, estado, cambio, persistencia e hidratación coordinado sobre el Query Engine, no como una abstracción que oculta las diferencias entre memoria y base de datos.**

Por ello:

```text
Entity
≠
Row
```

```text
Object State
≠
Database State
```

```text
IdentityMap
≠
Entity Cache
```

```text
UnitOfWork
≠
Transaction
```

```text
ChangeSet
≠
SQL
```

```text
persist()
≠
INSERT
```

```text
remove()
≠
DELETE inmediato
```

```text
flush()
≠
commit()
```

```text
Rollback
≠
Object Graph Rewind
```

```text
Hydration
≠
Query Execution
```

```text
Partial
≠
Complete
```

```text
Unloaded
≠
NULL
```

```text
Relationship Not Loaded
≠
Relationship Empty
```

```text
Eager Loading
≠
JOIN
```

```text
N+1 Detection
≠
Automatic Optimization
```

```text
Statement Success
≠
Transaction Success
```

```text
Transaction Success
≠
ORM Knowledge Automatically
```

```text
UNKNOWN
≠
ROLLBACK
```

y una prueba fuerte de persistencia deberá aproximarse a:

```text
Construct
   ↓
Persist
   ↓
Flush
   ↓
Commit if required
   ↓
Clear original ORM state
   ↓
Reload independently
   ↓
Hydrate
   ↓
Compare semantics
```

Por tanto:

```text
Reliable ORM Testing
=
Pure Entity Tests
+
Metadata & Mapping Tests
+
IdentityMap Invariants
+
UnitOfWork Invariants
+
Persistence Planning Tests
+
Hydration Tests
+
Relationship Tests
+
Transaction Tests
+
Real DBMS Round Trips
+
Cross-platform Conformance
+
Failure Injection
+
Persistent Runtime Isolation
```

sin crear un ORM paralelo exclusivamente para testing y sin convertir mocks, IdentityMap o SQLite en evidencia de comportamientos que sólo un DBMS concreto puede demostrar.

---

# 287. Siguiente documento

```text
292_DATABASE_DRIVER_CONFORMANCE_TESTING_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura oficial de **Driver Conformance Testing**, incluyendo:

```text
Driver Conformance Testing
│
├── Driver Contract Suite
├── Connection Contract
├── Connection Lifecycle
├── Prepared Statements
├── Parameter Binding
├── Result Sets
├── Cursor Semantics
├── Streaming
├── Transactions
├── Savepoints
├── Isolation Levels
├── Error Classification
├── Timeout / Cancellation
├── Retry-relevant Evidence
├── Connection Reset
├── Persistent Runtime Reuse
├── Data Type Round Trips
├── Binary Data
├── Unicode
├── Decimal Precision
├── Date / Time
├── JSON
├── Generated Identifiers
├── Affected Rows
├── Server Metadata
├── Capability Discovery
├── Security Properties
├── Failure Injection
├── MySQL Conformance
├── MariaDB Conformance
├── PostgreSQL Conformance
└── SQLite Conformance
```

manteniendo como principio central:

> **Un driver de VoltStack sólo podrá declararse conforme cuando demuestre, mediante una suite contractual reutilizable y evidencia obtenida contra infraestructura real cuando el contrato dependa del DBMS, que preserva las semánticas exigidas por la capa Database; implementar las interfaces PHP correctas no será suficiente para demostrar conformidad.**