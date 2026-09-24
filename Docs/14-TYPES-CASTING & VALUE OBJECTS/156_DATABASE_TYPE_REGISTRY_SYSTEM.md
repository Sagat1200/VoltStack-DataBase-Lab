# 156_DATABASE_TYPE_REGISTRY_SYSTEM.md

# VoltStack Quantum Database
## Database Type Registry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 156 — Database Type Registry System  
**Bloque:** 14 — Types, Casting & Value Objects  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `155_DATABASE_TYPE_SYSTEM.md`

---

# 1. Propósito

`Database Type Registry System` define la infraestructura responsable de registrar, normalizar, resolver, validar, congelar, versionar y exponer los tipos conocidos por VoltStack Database.

El sistema será el catálogo canónico mediante el cual componentes como:

- ORM;
- Schema;
- Query Engine;
- Hydration;
- Parameter Binding;
- SQL Compiler;
- Schema Introspection;
- Migration;
- Platform;
- custom extensions;

podrán transformar un identificador lógico:

```text
uuid
```

en una definición canónica:

```text
TypeId("uuid")
        ↓
TypeDefinition
        ↓
TypeBehavior
```

sin utilizar:

- reflection en hot path;
- búsqueda dinámica de clases;
- `switch` globales;
- estado mutable entre requests;
- nombres de clase como identidad del tipo.

La regla fundamental será:

> **`TypeRegistry` será el catálogo canónico, determinista e immutable después del bootstrap de todos los tipos conocidos por una generación de VoltStack Database; ningún componente del hot path descubrirá tipos mediante reflection, class scanning o nombres de clase provenientes de datos externos.**

---

# 2. Relación con el Type System

El documento anterior estableció:

```text
155_DATABASE_TYPE_SYSTEM
```

y definió:

```text
TypeId
TypeDefinition
TypeReference
TypeBehavior
TypeFamily
TypeArguments
PlatformTypeResolver
TypeValueConverter
```

El Registry System responde ahora:

> ¿Cómo descubre VoltStack qué tipos existen y qué implementación corresponde a cada `TypeId`?

---

# 3. Problema arquitectónico

Sin un registro central, cada subsistema podría terminar implementando algo como:

```php
switch ($type) {
    case 'int':
        // ...
        break;

    case 'uuid':
        // ...
        break;

    case 'json':
        // ...
        break;
}
```

Esto provocaría:

```text
duplicación
+
vendor conditionals
+
inconsistencias
+
reflection
+
custom types difíciles
+
plugins inseguros
+
hot path costoso
```

VoltStack deberá evitar este modelo.

---

# 4. Arquitectura general

```text
                   Bootstrap
                      │
                      ▼
             TypeRegistryBuilder
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   Core Types     Extensions     Plugins
        │             │             │
        └─────────────┼─────────────┘
                      ▼
               Registrations
                      │
                      ▼
               Normalization
                      │
                      ▼
                 Validation
                      │
                      ▼
              Conflict Analysis
                      │
                      ▼
                  Compile
                      │
                      ▼
                  Freeze
                      │
                      ▼
              ┌───────────────┐
              │ TypeRegistry  │
              └───────┬───────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        ORM         Schema      Query Engine
          │           │           │
          └───────────┼───────────┘
                      ▼
                Type Resolution
```

---

# 5. Responsabilidades

El Registry System deberá encargarse de:

1. registrar tipos core;
2. registrar tipos oficiales;
3. registrar custom types;
4. registrar tipos de plugins;
5. resolver `TypeId`;
6. resolver aliases;
7. canonicalizar identificadores;
8. detectar duplicados;
9. validar definiciones;
10. validar behaviors;
11. validar dependencias;
12. controlar overrides;
13. compilar el registry;
14. congelar el registry;
15. generar fingerprints;
16. administrar generaciones;
17. invalidar caches dependientes;
18. producir diagnósticos;
19. proporcionar introspección;
20. mantener resolución O(1) promedio.

---

# 6. No responsabilidades

El Registry no deberá:

```text
convertir valores
generar SQL
ejecutar queries
hidratar entidades
persistir entidades
abrir conexiones
administrar transacciones
hacer casting
inferir schemas completos
```

---

# 7. Registry ≠ Container

Una distinción fundamental:

```text
TypeRegistry
≠
Service Container
```

El Service Container administra servicios.

El TypeRegistry administra tipos conocidos por Database.

---

# 8. Registry ≠ Class Map

Tampoco será simplemente:

```php
[
    'int' => IntegerType::class,
    'uuid' => UuidType::class,
]
```

El Registry almacenará información semántica compilada.

---

# 9. Registry ≠ Factory

El Registry resuelve definiciones.

No deberá crear arbitrariamente nuevos objetos de dominio por lookup.

---

# 10. TypeRegistry

Contrato principal:

```php
interface TypeRegistry
{
    public function has(TypeId|string $type): bool;

    public function get(TypeId|string $type): TypeDefinition;

    public function resolve(TypeId|string $type): TypeResolution;

    public function canonicalize(string $type): TypeId;

    public function generation(): TypeRegistryGeneration;

    public function fingerprint(): TypeRegistryFingerprint;

    public function all(): iterable;
}
```

---

# 11. Read-only public contract

Después del bootstrap, el contrato utilizado por runtime será exclusivamente de lectura.

No deberá exponer:

```php
$registry->register(...);
$registry->remove(...);
$registry->override(...);
```

---

# 12. Mutation boundary

La mutación ocurrirá mediante:

```text
TypeRegistryBuilder
```

durante bootstrap.

---

# 13. Arquitectura Build → Freeze

```text
MUTABLE BUILD PHASE
        │
        ▼
TypeRegistryBuilder
        │
        ▼
Registrations
        │
        ▼
Validation
        │
        ▼
Compilation
        │
        ▼
freeze()
        │
        ▼
IMMUTABLE RUNTIME PHASE
        │
        ▼
CompiledTypeRegistry
```

---

# 14. TypeRegistryBuilder

Contrato conceptual:

```php
interface TypeRegistryBuilder
{
    public function register(
        TypeRegistration $registration,
    ): void;

    public function alias(
        TypeAliasRegistration $alias,
    ): void;

    public function compile(): TypeRegistry;
}
```

---

# 15. Builder lifetime

El builder existirá únicamente durante:

```text
framework bootstrap
container compilation
test environment setup
```

No durante cada request.

---

# 16. TypeRegistration

Una registración deberá ser explícita.

```php
final readonly class TypeRegistration
{
    public function __construct(
        public TypeDefinition $definition,
        public TypeBehaviorReference $behavior,
        public TypeRegistrationOrigin $origin,
        public TypeRegistrationPriority $priority,
    ) {}
}
```

---

# 17. Registration origin

VoltStack deberá conocer de dónde proviene un tipo.

```php
enum TypeRegistrationOriginKind
{
    case CORE;
    case OFFICIAL_EXTENSION;
    case APPLICATION;
    case PLUGIN;
    case TEST;
}
```

---

# 18. TypeRegistrationOrigin

Ejemplo:

```php
final readonly class TypeRegistrationOrigin
{
    public function __construct(
        public TypeRegistrationOriginKind $kind,
        public string $name,
        public ?string $package = null,
    ) {}
}
```

Esto permitirá mejores diagnósticos de conflictos.

---

# 19. Core types

El framework registrará inicialmente tipos como:

```text
bool

tinyint
smallint
int
bigint

decimal
float
double

string
text

binary
blob

uuid
ulid

json

date
time
datetime
datetime_immutable
timestamp

enum
value_object
```

La lista podrá evolucionar mediante versionado.

---

# 20. Canonical TypeId

Cada tipo tendrá exactamente un identificador canónico.

Ejemplo:

```text
int
```

---

# 21. Aliases

Podrán existir aliases:

```text
integer
→ int

boolean
→ bool

big_integer
→ bigint
```

pero el resultado interno siempre será el identificador canónico.

---

# 22. Alias invariant

```text
Alias
≠
TypeId
```

---

# 23. Canonicalization

Ejemplo:

```php
$registry->canonicalize('integer');
```

resultado:

```text
TypeId("int")
```

---

# 24. Runtime representation

Después de canonicalizar:

```text
"integer"
```

no deberá seguir propagándose por el runtime.

Debe convertirse a:

```text
TypeId("int")
```

---

# 25. Alias chains

No deberán mantenerse cadenas arbitrarias como:

```text
integer
→ number_integer
→ signed_integer
→ int
```

---

# 26. Alias compilation

Durante bootstrap:

```text
Alias Graph
    ↓
Validation
    ↓
Flatten
    ↓
Canonical Alias Map
```

Resultado:

```text
integer → int
number_integer → int
signed_integer → int
```

---

# 27. Alias cycle

Debe rechazarse:

```text
a → b
b → c
c → a
```

con:

```text
TypeAliasCycleException
```

---

# 28. Unknown alias target

También será error:

```text
foo → nonexistent_type
```

salvo que la compilación aún tenga registraciones pendientes y pueda resolverlo al finalizar.

---

# 29. Case normalization

VoltStack deberá definir una política canónica.

Recomendación:

```text
TypeId
=
lowercase ASCII identifier
```

Ejemplo:

```text
UUID
Uuid
uuid
```

se normalizan en APIs textuales a:

```text
uuid
```

---

# 30. Internal strictness

Una vez creado `TypeId`, deberá utilizar la representación canónica.

---

# 31. TypeId grammar

Recomendación:

```text
[a-z][a-z0-9._-]*
```

permitiendo IDs como:

```text
uuid
json
geo.point
vendor.vector
```

---

# 32. Namespacing

Custom types podrán utilizar namespaces lógicos.

Ejemplo:

```text
w4.money
geo.point
postgres.vector
```

---

# 33. Namespace ≠ PHP namespace

```text
geo.point
```

no implica:

```text
Geo\Point
```

---

# 34. Core namespace

Los tipos fundamentales podrán conservar nombres simples:

```text
int
string
uuid
json
```

para DX.

---

# 35. Plugin namespace

Plugins deberían utilizar IDs namespaced cuando exista riesgo de colisión.

---

# 36. TypeDefinition storage

El registry almacenará:

```text
TypeId
    ↓
TypeDefinition
```

mediante una estructura indexada.

Conceptualmente:

```php
array<string, TypeDefinition>
```

---

# 37. Lookup complexity

Objetivo:

```text
get(TypeId)
≈ O(1)
```

promedio.

---

# 38. No scanning

Resolver:

```text
uuid
```

no deberá recorrer todos los tipos.

---

# 39. TypeResolution

Cuando se necesite información de resolución:

```php
final readonly class TypeResolution
{
    public function __construct(
        public TypeId $requested,
        public TypeId $canonical,
        public TypeDefinition $definition,
        public TypeResolutionKind $kind,
    ) {}
}
```

---

# 40. TypeResolutionKind

```php
enum TypeResolutionKind
{
    case CANONICAL;
    case ALIAS;
}
```

---

# 41. Unknown type

Por defecto:

```php
$registry->get('does_not_exist');
```

deberá producir:

```text
UnknownTypeException
```

---

# 42. has()

En cambio:

```php
$registry->has('does_not_exist');
```

retorna:

```text
false
```

sin excepción.

---

# 43. Suggestions

Para DX, un error:

```text
Unknown database type "datatime".
```

puede sugerir:

```text
Did you mean "datetime"?
```

---

# 44. Suggestions fuera del hot path

La búsqueda aproximada no deberá ejecutarse en cada lookup exitoso.

Solo en errores/diagnostics.

---

# 45. Behavior registry

El documento 155 separó:

```text
TypeDefinition
```

de:

```text
TypeBehavior
```

Esto se mantiene.

---

# 46. Razón

`TypeDefinition` deberá ser:

```text
immutable
serializable
cacheable
inspectable
```

mientras que `TypeBehavior` puede contener estrategias ejecutables stateless.

---

# 47. TypeBehaviorRegistry

Contrato conceptual:

```php
interface TypeBehaviorRegistry
{
    public function get(TypeId $type): TypeBehavior;

    public function has(TypeId $type): bool;
}
```

---

# 48. Logical architecture

```text
TypeRegistry
│
├── TypeDefinition
│
└── BehaviorReference
        │
        ▼
TypeBehaviorRegistry
        │
        ▼
TypeBehavior
```

---

# 49. Behavior identity

El behavior no deberá ser identificado por una clase arbitraria proveniente del usuario en runtime.

---

# 50. Compiled behavior binding

Durante bootstrap:

```text
TypeId
+
Behavior Provider
↓
validated binding
```

---

# 51. Stateless behaviors

Siempre que sea posible:

```text
TypeBehavior
=
stateless singleton-safe service
```

---

# 52. Stateful behavior

Si alguna extensión necesita estado operacional:

```text
shared TypeBehavior
        ↓
explicit scoped context
```

en vez de guardar estado mutable dentro del behavior.

---

# 53. Converter registry

El Registry System podrá integrar referencias hacia converters.

Conceptualmente:

```text
uuid
├── definition
├── converter
├── binding strategy
└── compatibility strategy
```

---

# 54. Single source of truth

No deberán existir registries independientes contradictorios como:

```text
OrmTypeRegistry
SchemaTypeRegistry
QueryTypeRegistry
HydrationTypeRegistry
```

para los mismos tipos persistibles.

---

# 55. Specialized views

Sí podrán existir vistas especializadas:

```text
ORM
    ↓
TypeRegistryView

Schema
    ↓
TypeRegistryView
```

pero ambas apuntarán al mismo catálogo canónico.

---

# 56. Type registry composition

El registry final podrá componerse de:

```text
Core Registry
+
Official Extensions
+
Application Types
+
Plugin Types
```

antes de congelarse.

---

# 57. Deterministic composition

El resultado no deberá depender accidentalmente del orden en que PHP descubrió archivos.

---

# 58. Registration phases

VoltStack podrá definir:

```text
CORE
OFFICIAL
PLUGIN
APPLICATION
FINALIZATION
```

como fases de bootstrap.

---

# 59. Registration priority

La prioridad no deberá utilizarse para ocultar conflictos silenciosamente.

---

# 60. Conflict detection

Si dos componentes registran:

```text
TypeId("money")
```

con definiciones diferentes:

```text
CONFLICT
```

por defecto.

---

# 61. Duplicate identical registration

Incluso si son aparentemente idénticas, la política recomendada es detectar el duplicado.

Esto evita dependencias accidentales.

---

# 62. Override policy

Overrides deberán ser explícitos.

```php
enum TypeOverridePolicy
{
    case FORBID;
    case EXPLICIT_APPLICATION_OVERRIDE;
    case EXPLICIT_EXTENSION_OVERRIDE;
}
```

---

# 63. Default

```text
FORBID
```

---

# 64. Core type protection

Tipos core como:

```text
int
string
bool
decimal
```

no deberán poder cambiar silenciosamente de semántica.

---

# 65. Application override

Si VoltStack decide soportarlo, deberá requerir algo explícito como:

```php
DatabaseTypes::override(
    type: 'uuid',
    with: CustomUuidType::class,
);
```

y producir diagnóstico.

---

# 66. Override ≠ alias

Si se desea otro nombre:

```text
my_uuid → uuid
```

utilizar alias.

No override.

---

# 67. Override compatibility

Un override permitido deberá pasar análisis de compatibilidad.

---

# 68. Override safety

Deberán validarse al menos:

- familia lógica;
- argumentos soportados;
- converter;
- binding;
- platform mappings;
- compatibility semantics.

---

# 69. Breaking override

Un override incompatible deberá fallar o requerir una política explícitamente peligrosa fuera del modo estándar.

---

# 70. Registration validation pipeline

```text
TypeRegistration
      │
      ▼
Structural Validation
      │
      ▼
TypeId Validation
      │
      ▼
Definition Validation
      │
      ▼
Behavior Validation
      │
      ▼
Dependency Validation
      │
      ▼
Conflict Validation
      │
      ▼
Platform Declaration Validation
      │
      ▼
Accepted Registration
```

---

# 71. Structural validation

Verifica que existan:

```text
TypeId
TypeDefinition
Behavior binding
Origin
```

---

# 72. Definition validation

Comprueba coherencia entre:

```text
TypeId
TypeFamily
PHP descriptor
argument schema
characteristics
```

---

# 73. Behavior validation

Comprueba que las operaciones requeridas existan.

---

# 74. Dependency validation

Un tipo puede depender de otro tipo.

Ejemplo conceptual:

```text
enum
→ backing type
```

o:

```text
custom value object
→ decimal
```

---

# 75. Type dependency

```php
final readonly class TypeDependency
{
    public function __construct(
        public TypeId $type,
        public TypeDependencyKind $kind,
    ) {}
}
```

---

# 76. Dependency graph

Durante compilación:

```text
Types
 ↓
Dependency Graph
 ↓
Validation
```

---

# 77. Missing dependency

Debe fallar:

```text
money
→ decimal
```

si `decimal` no está registrado.

---

# 78. Circular dependency

No todo ciclo es necesariamente inválido conceptualmente, pero behaviors de conversión que requieran inicialización recursiva deberán detectarse.

---

# 79. Preferencia

Las definiciones de tipos deberían minimizar dependencias entre tipos.

---

# 80. Registry compiler

```php
interface TypeRegistryCompiler
{
    public function compile(
        TypeRegistryBuildState $state,
    ): CompiledTypeRegistry;
}
```

---

# 81. Compilation phases

```text
Collect
 ↓
Normalize
 ↓
Validate
 ↓
Resolve Aliases
 ↓
Resolve Dependencies
 ↓
Detect Conflicts
 ↓
Compile Definitions
 ↓
Compile Behaviors
 ↓
Fingerprint
 ↓
Freeze
```

---

# 82. CompiledTypeRegistry

Implementación runtime:

```php
final class CompiledTypeRegistry implements TypeRegistry
{
    // immutable indexed maps
}
```

---

# 83. Frozen state

Después de `compile()`:

```text
RegistryState::FROZEN
```

---

# 84. Frozen invariant

Intentar modificarlo deberá ser imposible por API normal.

---

# 85. Registry state

Durante construcción puede existir:

```php
enum TypeRegistryState
{
    case BUILDING;
    case VALIDATING;
    case COMPILING;
    case FROZEN;
    case FAILED;
}
```

---

# 86. FAILED

Si la compilación falla, el registry parcial no deberá exponerse al runtime.

---

# 87. Atomic publication

Conceptualmente:

```text
Build temporary registry
        ↓
Validate completely
        ↓
Compile completely
        ↓
Publish immutable registry
```

Nunca:

```text
publish partial
→ continue registering
```

---

# 88. Registry generation

Cada compilación lógica tendrá una generación.

```php
final readonly class TypeRegistryGeneration
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 89. Generation purpose

Permitirá detectar:

```text
metadata compiled with registry A
runtime using registry B
```

---

# 90. Generation ≠ application version

No son necesariamente iguales.

---

# 91. Generation derivation

Puede derivarse de:

```text
core type version
+
registrations
+
aliases
+
behavior bindings
+
relevant policies
```

---

# 92. Registry fingerprint

```php
final readonly class TypeRegistryFingerprint
{
    public function __construct(
        public string $hash,
    ) {}
}
```

---

# 93. Deterministic fingerprint

Debe cumplirse:

```text
same semantic registry
→ same fingerprint
```

independientemente del orden incidental de registración.

---

# 94. Fingerprint canonicalization

Antes de hash:

```text
sort TypeIds
sort aliases
canonicalize arguments
canonicalize dependencies
canonicalize policies
```

---

# 95. Fingerprint exclusions

No incluir:

```text
memory addresses
object IDs
timestamps
random IDs
request IDs
```

---

# 96. Generation and cache

Caches dependientes deberán incluir:

```text
TypeRegistryGeneration
```

o:

```text
TypeRegistryFingerprint
```

---

# 97. Metadata cache

ORM metadata compilada con:

```text
generation=A
```

no deberá reutilizarse ciegamente con:

```text
generation=B
```

---

# 98. Schema cache

Mismo principio.

---

# 99. Query plan cache

Si el plan depende de tipos:

```text
TypeRegistryGeneration
```

formará parte del fingerprint relevante.

---

# 100. Hydration plan cache

Igualmente.

---

# 101. Persistent runtime

En FrankenPHP:

```text
Worker
│
├── CompiledTypeRegistry   ← shared immutable
├── TypeBehaviorRegistry   ← stateless/shared
│
├── Request A
├── Request B
└── Request C
```

---

# 102. No per-request rebuild

El TypeRegistry no deberá reconstruirse en cada request.

---

# 103. No per-request mutation

Tampoco deberá modificarse por una petición.

---

# 104. RoadRunner

La misma arquitectura deberá funcionar sin cambios semánticos.

---

# 105. OpenSwoole

El registry immutable podrá compartirse entre ejecuciones concurrentes dentro de los límites del runtime.

---

# 106. Application dynamic type registration

Por defecto:

```text
runtime dynamic registration
=
FORBIDDEN
```

---

# 107. Razón

Permitir:

```php
Database::types()->register(...)
```

durante una request introduciría:

- race conditions;
- cache incoherence;
- metadata inconsistente;
- diferencias entre workers;
- comportamiento no determinista.

---

# 108. Development hot reload

Si un entorno de desarrollo necesita reload:

```text
build new generation
→ validate
→ atomically replace worker/container generation
```

No mutar la generación actual.

---

# 109. Immutable generation model

```text
Registry Generation A
        │
        └── immutable

new configuration
        │
        ▼
Registry Generation B
        │
        └── immutable
```

---

# 110. Worker consistency

Una operación Database deberá utilizar una sola generación durante toda su ejecución.

---

# 111. No mid-query registry switch

Nunca:

```text
Query starts with generation A
        ↓
registry reload
        ↓
hydration uses generation B
```

---

# 112. DatabaseContext binding

Cuando sea necesario, `DatabaseContext` podrá conservar:

```text
TypeRegistryGeneration
```

para verificar consistencia.

---

# 113. Plugin registration

Plugins podrán contribuir tipos durante bootstrap.

Ejemplo conceptual:

```php
final class VectorDatabaseExtension
{
    public function registerDatabaseTypes(
        TypeRegistryBuilder $types,
    ): void {
        $types->register(...);
    }
}
```

---

# 114. Plugin discovery

La detección del plugin pertenece al sistema de extensiones/packages.

No al TypeRegistry.

---

# 115. Registry receives contributions

```text
Plugin System
    ↓
TypeRegistrationContributor
    ↓
TypeRegistryBuilder
```

---

# 116. Contributor contract

```php
interface TypeRegistrationContributor
{
    public function contribute(
        TypeRegistryBuilder $registry,
    ): void;
}
```

---

# 117. Deterministic contributors

VoltStack deberá ordenar contributors mediante reglas deterministas antes de compilar.

---

# 118. Contributor identity

Cada contributor deberá tener identidad conocida para diagnostics.

---

# 119. Duplicate plugin type

Ejemplo:

```text
plugin A → vector
plugin B → vector
```

debe producir conflicto explícito.

---

# 120. Plugin isolation

Un plugin no deberá poder eliminar silenciosamente tipos de otro plugin.

---

# 121. Platform-specific types

VoltStack deberá soportar tipos que solo existen en ciertas plataformas.

Ejemplo conceptual:

```text
postgres.vector
```

---

# 122. Registration vs support

Que un tipo esté registrado:

```text
Registry.has(postgres.vector) = true
```

no significa:

```text
MySQL supports postgres.vector
```

---

# 123. Platform capability

La representabilidad seguirá siendo responsabilidad de:

```text
PlatformTypeResolver
+
PlatformCapabilities
```

---

# 124. Global registry

Por tanto, el registry puede conocer un tipo incluso si la conexión actual no puede usarlo.

---

# 125. Why

Esto permite:

- analizar schemas;
- generar diagnostics;
- migraciones cross-platform;
- detectar incompatibilidades antes de ejecutar.

---

# 126. Platform type extensions

Una extensión puede aportar:

```text
Logical Type
+
Platform Resolver
```

sin alterar el registry core.

---

# 127. TypeRegistry vs PlatformTypeRegistry

Evitar dos fuentes de verdad.

La arquitectura preferida será:

```text
TypeRegistry
    │
    └── knows logical type

Platform
    │
    └── knows physical representation
```

---

# 128. PHP type mapping

Puede existir una tabla auxiliar:

```text
PHP Type
→ candidate Database Type(s)
```

pero no será equivalente al TypeRegistry.

---

# 129. Ambiguous PHP mapping

Ejemplo:

```text
string
```

puede representar:

```text
string
text
decimal
uuid
ulid
enum backing value
json text
```

Por tanto:

```text
PHP Type Registry
→ candidates
```

no:

```text
PHP Type
→ guaranteed DB Type
```

---

# 130. Type inference integration

El Query Type Inference System podrá consultar:

```text
TypeRegistry
```

para validar candidatos.

---

# 131. Explicit mapping wins

Si ORM metadata declara:

```text
uuid
```

no deberá reemplazarse por inferencia basada en que PHP utiliza `string`.

---

# 132. TypeReference creation

El Registry podrá proporcionar una factory:

```php
interface TypeReferenceFactory
{
    public function create(
        TypeId|string $type,
        array $arguments = [],
        Nullability $nullability = Nullability::NOT_NULL,
    ): TypeReference;
}
```

---

# 133. Factory responsibility

La factory:

1. canonicaliza alias;
2. obtiene definición;
3. normaliza argumentos;
4. valida argumentos;
5. construye `TypeReference`.

---

# 134. TypeReferenceFactory ≠ Registry

Separarlos mantiene responsabilidades claras.

---

# 135. Example

```php
$types->references()->create(
    'decimal',
    [
        'precision' => 19,
        'scale' => 4,
    ],
);
```

resultado:

```text
TypeReference
├── TypeId: decimal
├── precision: 19
├── scale: 4
└── nullable: false
```

---

# 136. Alias example

```php
$factory->create('integer');
```

produce:

```text
TypeReference(TypeId("int"))
```

---

# 137. No alias persistence

Compiled metadata deberá almacenar:

```text
int
```

no:

```text
integer
```

---

# 138. Type definition lookup

```php
$definition = $registry->get(
    new TypeId('uuid')
);
```

---

# 139. Behavior lookup

```php
$behavior = $behaviors->get(
    new TypeId('uuid')
);
```

---

# 140. TypeRegistryFacade

La API pública podría ofrecer:

```php
Database::types();
```

pero deberá ser una facade sobre servicios reales.

---

# 141. Facade state

No deberá contener registry mutable estático.

---

# 142. DX API

Ejemplo:

```php
Database::types()->has('uuid');

Database::types()->get('uuid');

Database::types()->resolve('integer');

Database::types()->explain('json');
```

---

# 143. Explain

`explain()` pertenece preferentemente a Diagnostics, aunque la facade pueda delegar.

---

# 144. Registry introspection

Podrá listar:

```text
TypeId
Family
Origin
Aliases
PHP representation
Platform support summary
```

---

# 145. Sensitive information

No deberá mostrar:

- credentials;
- connection strings;
- actual user values.

---

# 146. CLI

Posibles comandos futuros:

```text
volt database:type:list
volt database:type:show uuid
volt database:type:aliases
volt database:type:validate
```

---

# 147. `type:list`

Ejemplo conceptual:

```text
TYPE       FAMILY        ORIGIN
---------------------------------------
bool       BOOLEAN       core
int        INTEGER       core
decimal    DECIMAL       core
string     STRING        core
uuid       IDENTIFIER    core
json       STRUCTURED    core
money      DOMAIN        application
```

---

# 148. `type:show`

```text
Type:
    uuid

Canonical ID:
    uuid

Family:
    IDENTIFIER

Origin:
    core

Aliases:
    none

PHP representations:
    string
    Uuid

Registry generation:
    dbtypes:8f24...
```

---

# 149. Diagnostics for conflict

Ejemplo:

```text
Database type registration conflict.

Type:
    money

Existing registration:
    package: w4/payments
    origin: plugin

Conflicting registration:
    application
    origin: application

Policy:
    FORBID

Resolution:
    Rename the application type or configure an explicit supported override.
```

---

# 150. Registry compilation errors

Todos los errores deberían acumularse cuando sea seguro.

En vez de:

```text
fix one
restart
find next
```

preferir:

```text
5 type registry errors detected
```

con reportes individuales.

---

# 151. Fail-fast runtime

Sin embargo, un registry inválido no deberá iniciar el Database System.

---

# 152. Bootstrap failure

Si los tipos son inconsistentes:

```text
Database bootstrap
→ FAIL
```

antes de servir requests.

---

# 153. Registry validation report

```php
final readonly class TypeRegistryValidationReport
{
    public function __construct(
        public array $errors,
        public array $warnings,
        public array $notices,
    ) {}
}
```

---

# 154. Severity

```php
enum TypeRegistryDiagnosticSeverity
{
    case ERROR;
    case WARNING;
    case NOTICE;
}
```

---

# 155. ERROR

Impide compilar.

---

# 156. WARNING

Permite compilar pero señala riesgo.

Ejemplo:

```text
type only supported on PostgreSQL
```

si la aplicación también declara MySQL como target.

---

# 157. NOTICE

Información útil de DX.

---

# 158. Registry dependency graph

Conceptualmente:

```text
decimal

money
 └── decimal

percentage
 └── decimal

enum
 └── string/int backing candidates
```

---

# 159. Graph use

Permite:

- validar dependencias;
- ordenar compilación;
- explicar custom types;
- invalidar caches transitivamente.

---

# 160. Dependency fingerprint

Si cambia `decimal` y `money` depende de él:

```text
money effective fingerprint
```

deberá reflejarlo cuando la semántica dependa de esa definición.

---

# 161. Effective type fingerprint

Conceptualmente:

```text
EffectiveTypeFingerprint(T)
=
Hash(
    Definition(T)
    +
    BehaviorBinding(T)
    +
    DependencyFingerprints(T)
)
```

---

# 162. Cycle-safe fingerprinting

Si se permiten ciertos ciclos declarativos, el algoritmo deberá manejar SCCs o rechazar ciclos no soportados.

V1 debería preferir:

```text
acyclic behavior dependencies
```

---

# 163. Registry snapshot

Puede existir:

```php
final readonly class TypeRegistrySnapshot
{
    public function __construct(
        public TypeRegistryGeneration $generation,
        public TypeRegistryFingerprint $fingerprint,
        public array $definitions,
        public array $aliases,
    ) {}
}
```

---

# 164. Snapshot purpose

Útil para:

- compiled container;
- diagnostics;
- testing;
- cache validation.

---

# 165. Snapshot ≠ mutable registry

Es representación immutable.

---

# 166. Serialization

Solo metadata segura deberá serializarse.

---

# 167. Behaviors

Servicios ejecutables deberán resolverse mediante bindings compilados, no serializar closures arbitrarias.

---

# 168. No closures in compiled registry

Evitar:

```php
'type' => fn ($value) => ...
```

dentro de metadata cache persistente.

---

# 169. Service references

Preferir identificadores compilados:

```text
BehaviorServiceId
ConverterServiceId
BindingStrategyId
```

---

# 170. Service Container integration

Bootstrap:

```text
Database Service Provider
        │
        ▼
Type Contributors
        │
        ▼
TypeRegistryBuilder
        │
        ▼
CompiledTypeRegistry
        │
        ├── register singleton immutable registry
        │
        └── register behavior bindings
```

---

# 171. Dependency direction

```text
Database Type consumers
        ↓
TypeRegistry contract
        ↑
CompiledTypeRegistry
```

---

# 172. Registry must not depend on ORM

Invariante:

```text
Type Registry
↛ ORM
```

---

# 173. Registry must not depend on Schema

Igualmente:

```text
Type Registry
↛ Schema
```

Schema consume Registry.

---

# 174. Registry must not depend on Query Engine

Query Engine consume Registry.

---

# 175. Registry must not depend on Hydration

Hydration consume Registry/Conversion.

---

# 176. Shared contracts

Si ciertos contratos son suficientemente fundamentales podrán vivir en:

```text
VoltStack/Platform
```

pero la implementación Database seguirá en:

```text
VoltStack/Quantum/Database
```

---

# 177. Registry cache

Lookup directo probablemente no necesita cache adicional:

```text
array lookup
```

ya es suficiente.

---

# 178. What should be cached

Sí pueden cachearse:

```text
alias canonicalization
TypeReference normalization
platform resolution
compatibility analysis
```

en sus respectivos sistemas.

---

# 179. Avoid cache layering

No crear:

```text
registry cache
inside registry cache
inside type lookup cache
```

sin evidencia de beneficio.

---

# 180. Interning

El registry puede internar:

```text
TypeId
```

core y referencias muy comunes.

---

# 181. TypeId interning

Debido a que el conjunto de TypeIds registrados es acotado:

```text
TypeId interning
```

es seguro.

---

# 182. TypeReference interning

Es distinto porque argumentos pueden ser dinámicos.

Debe permanecer bounded.

---

# 183. Registry size

Normalmente:

```text
dozens
or
hundreds
```

de tipos.

No millones.

---

# 184. Registration resource governance

VoltStack podrá imponer:

```text
max registered types
max aliases
max aliases per type
max dependency depth
```

especialmente en entornos plugin-heavy.

---

# 185. Why resource limits

Un plugin defectuoso no deberá poder registrar millones de aliases durante bootstrap.

---

# 186. Security boundary

Los TypeIds provenientes de:

- request parameters;
- API payloads;
- database values;

no deberán provocar carga dinámica de clases.

---

# 187. External TypeId resolution

Si una API permite tipos dinámicos:

```text
external string
    ↓
validation
    ↓
allowlist/policy
    ↓
TypeRegistry
```

---

# 188. Registry membership ≠ API permission

Que un tipo exista no significa que un usuario pueda solicitarlo.

---

# 189. Authorization

El Registry no realizará authorization.

---

# 190. Security principle

```text
Known Type
≠
Allowed External Type
```

---

# 191. Database metadata input

Introspection puede recibir:

```text
VARCHAR
JSONB
UUID
```

pero esos son nombres físicos.

No deberán enviarse directamente a:

```text
TypeRegistry::get()
```

---

# 192. Correct path

```text
Physical DB Type
    ↓
Platform Introspection Type Resolver
    ↓
Canonical TypeId
    ↓
TypeRegistry
```

---

# 193. Prevent namespace confusion

Esto evita confundir:

```text
PostgreSQL "integer"
```

con un alias público de VoltStack sin consultar contexto de plataforma.

---

# 194. Type registry versioning

El catálogo core evolucionará con VoltStack.

---

# 195. Type semantic versioning

Cambiar el comportamiento semántico de un TypeId existente puede ser breaking.

---

# 196. Example breaking change

Si:

```text
decimal
```

antes se exponía como `float` y después como `string`, eso afecta:

- hydration;
- entity property compatibility;
- serialization;
- query parameters.

Debe tratarse como cambio importante.

---

# 197. New type

Agregar:

```text
ulid
```

normalmente puede ser backward-compatible.

---

# 198. New alias

Agregar un alias puede ser compatible, excepto si colisiona con un custom TypeId existente.

---

# 199. Alias collision

Ejemplo:

Aplicación existente:

```text
integer64
```

como custom type.

Nueva versión core intenta agregar:

```text
integer64 → bigint
```

Esto deberá detectarse.

---

# 200. Reserved IDs

VoltStack podrá mantener un conjunto de nombres reservados.

---

# 201. Reserved namespace

Por ejemplo:

```text
voltstack.*
```

podría reservarse para tipos oficiales.

---

# 202. Deprecation

Un TypeId podrá marcarse deprecated.

```php
final readonly class TypeDeprecation
{
    public function __construct(
        public string $since,
        public ?TypeId $replacement,
        public string $message,
    ) {}
}
```

---

# 203. Deprecated type lookup

El runtime podrá seguir resolviéndolo durante una ventana de compatibilidad.

---

# 204. Deprecation telemetry

No emitir un warning por cada row/query.

---

# 205. Compile-time warning

Preferir:

```text
ORM metadata compilation
Schema compilation
configuration validation
```

para advertir una vez.

---

# 206. Deprecated alias

Ideal para migraciones de naming.

```text
old_uuid
→ uuid
```

---

# 207. Canonical writes

Compiled metadata nueva utilizará:

```text
uuid
```

aunque haya recibido `old_uuid`.

---

# 208. Compatibility migrations

Los cambios del Registry deberán coordinarse con:

```text
DATABASE_BACKWARD_COMPATIBILITY_SYSTEM
DATABASE_VERSIONING_SYSTEM
DATABASE_DEPRECATION_POLICY
```

posteriormente.

---

# 209. Registry telemetry

Métricas posibles:

```text
database.type_registry.types
database.type_registry.aliases
database.type_registry.compile.duration
database.type_registry.compile.failure
database.type_registry.lookup.failure
database.type_registry.alias_resolution
database.type_registry.conflict
```

---

# 210. Lookup telemetry

No instrumentar cada lookup exitoso con un span.

Sería demasiado costoso.

---

# 211. Aggregate metrics

Preferir contadores agregados o diagnósticos de bootstrap.

---

# 212. High cardinality

No utilizar nombres arbitrarios de paquetes como labels sin control.

---

# 213. Compilation trace

Bootstrap puede incluir un span:

```text
database.type_registry.compile
```

con:

```text
types.count
aliases.count
plugins.count
warnings.count
```

---

# 214. Diagnostics API

```php
interface TypeRegistryDiagnostics
{
    public function inspect(): TypeRegistryReport;

    public function explain(TypeId|string $type): TypeDiagnosticReport;

    public function validate(): TypeRegistryValidationReport;
}
```

---

# 215. Registry report

```text
Generation
Fingerprint
State
Type count
Alias count
Core count
Extension count
Application count
Plugin count
Deprecated count
Platform-specific count
```

---

# 216. Debug output

No deberá incluir object dumps gigantes.

---

# 217. Testing architecture

El Registry System requerirá:

```text
Unit Tests
Integration Tests
Extension Tests
Conflict Tests
Determinism Tests
Persistent Runtime Tests
Security Tests
Performance Tests
```

---

# 218. Core registration test

Verificar que todos los tipos core estén presentes.

---

# 219. Unique TypeId test

```text
∀ T1,T2:
T1.id = T2.id
⇒
T1 = T2
```

dentro del registry final.

---

# 220. Alias canonicalization test

```text
integer
→ int
```

---

# 221. Alias cycle test

```text
a → b
b → a
```

debe fallar.

---

# 222. Missing target test

```text
foo → missing
```

debe fallar.

---

# 223. Conflict test

Dos tipos diferentes con mismo `TypeId` deben fallar.

---

# 224. Core override test

Override implícito de `string` debe fallar.

---

# 225. Explicit override test

Si la política lo permite, deberá verificar compatibilidad.

---

# 226. Freeze test

Después de compilación:

```text
registry mutation
```

debe ser imposible.

---

# 227. Determinism test

Registrar contribuciones equivalentes en distinto orden incidental deberá producir:

```text
same semantic registry
same fingerprint
```

si las reglas de composición las consideran equivalentes.

---

# 228. Generation test

Cambio semántico deberá producir nueva generación/fingerprint.

---

# 229. Non-semantic noise test

Cambios como:

```text
object address
registration timestamp
```

no deberán cambiar fingerprint.

---

# 230. Persistent runtime test

Request A no deberá modificar resolución observada por Request B.

---

# 231. Concurrent runtime test

OpenSwoole-style concurrent access deberá ser seguro para lectura.

---

# 232. Plugin conflict test

Plugins con mismo TypeId deberán generar diagnóstico determinista.

---

# 233. Security test

Un TypeId externo como:

```text
App\DangerousClass
```

no deberá causar autoload/instanciación.

---

# 234. Fuzz test

Fuzzing útil para:

```text
TypeId parser
alias graph
dependency graph
canonicalization
```

---

# 235. Performance tests

Medir:

```text
lookup
alias resolution
registry compile
fingerprint generation
large registry bootstrap
```

---

# 236. Performance targets

Runtime lookup:

```text
O(1) average
```

Alias lookup:

```text
O(1) average
```

después de flattening.

---

# 237. No recursive alias runtime resolution

La recursión pertenece a bootstrap.

Runtime:

```text
alias
→ canonical TypeId
```

en un lookup.

---

# 238. Large registry benchmark

Probar escenarios:

```text
50 types
500 types
5,000 types
```

aunque el uso normal sea mucho menor.

---

# 239. Memory benchmark

Medir:

```text
bytes per TypeDefinition
bytes per alias
bytes per behavior binding
```

para workers persistentes.

---

# 240. Error taxonomy

Base:

```text
DatabaseTypeRegistryException
```

---

# 241. UnknownTypeException

Tipo no registrado.

---

# 242. DuplicateTypeRegistrationException

Mismo TypeId registrado más de una vez sin política válida.

---

# 243. TypeRegistrationConflictException

Definiciones incompatibles.

---

# 244. InvalidTypeIdException

Identificador inválido.

---

# 245. TypeAliasConflictException

Alias colisiona con otro alias o TypeId.

---

# 246. TypeAliasCycleException

Ciclo en aliases.

---

# 247. UnknownTypeAliasTargetException

Alias apunta a tipo inexistente.

---

# 248. MissingTypeDependencyException

Dependencia no registrada.

---

# 249. CircularTypeDependencyException

Ciclo no permitido.

---

# 250. InvalidTypeDefinitionException

Definición inconsistente.

---

# 251. InvalidTypeBehaviorException

Behavior incompatible.

---

# 252. ForbiddenTypeOverrideException

Override no permitido.

---

# 253. IncompatibleTypeOverrideException

Override permitido en principio, pero incompatible.

---

# 254. FrozenTypeRegistryException

Intento interno de mutar una generación congelada.

Idealmente la API pública hará imposible llegar a este caso.

---

# 255. TypeRegistryCompilationException

Fallo general de compilación.

---

# 256. TypeRegistryGenerationMismatchException

Un artefacto compilado pertenece a otra generación incompatible.

---

# 257. TypeRegistryResourceLimitException

Contributor excede límites.

---

# 258. TypeRegistryInvariantViolationException

Estado que indica bug interno.

---

# 259. Directory structure

```text
src/Quantum/Database/Type/
│
├── Registry/
│   │
│   ├── Contract/
│   │   ├── TypeRegistry.php
│   │   ├── TypeRegistryBuilder.php
│   │   ├── TypeRegistryCompiler.php
│   │   ├── TypeBehaviorRegistry.php
│   │   └── TypeRegistrationContributor.php
│   │
│   ├── Registration/
│   │   ├── TypeRegistration.php
│   │   ├── TypeRegistrationOrigin.php
│   │   ├── TypeRegistrationOriginKind.php
│   │   ├── TypeRegistrationPriority.php
│   │   ├── TypeAliasRegistration.php
│   │   └── TypeOverridePolicy.php
│   │
│   ├── Builder/
│   │   ├── DefaultTypeRegistryBuilder.php
│   │   ├── TypeRegistryBuildState.php
│   │   └── TypeRegistrationCollector.php
│   │
│   ├── Compiler/
│   │   ├── DefaultTypeRegistryCompiler.php
│   │   ├── TypeDefinitionCompiler.php
│   │   ├── TypeAliasCompiler.php
│   │   ├── TypeDependencyCompiler.php
│   │   └── TypeBehaviorBindingCompiler.php
│   │
│   ├── Runtime/
│   │   ├── CompiledTypeRegistry.php
│   │   ├── CompiledTypeBehaviorRegistry.php
│   │   ├── TypeResolution.php
│   │   └── TypeResolutionKind.php
│   │
│   ├── Alias/
│   │   ├── TypeAliasResolver.php
│   │   ├── TypeAliasGraph.php
│   │   └── CanonicalTypeAliasMap.php
│   │
│   ├── Dependency/
│   │   ├── TypeDependency.php
│   │   ├── TypeDependencyKind.php
│   │   ├── TypeDependencyGraph.php
│   │   └── TypeDependencyValidator.php
│   │
│   ├── Validation/
│   │   ├── TypeRegistryValidator.php
│   │   ├── TypeDefinitionValidator.php
│   │   ├── TypeBehaviorValidator.php
│   │   ├── TypeOverrideValidator.php
│   │   └── TypeRegistryValidationReport.php
│   │
│   ├── Generation/
│   │   ├── TypeRegistryGeneration.php
│   │   ├── TypeRegistryFingerprint.php
│   │   ├── TypeRegistryFingerprinter.php
│   │   └── TypeRegistrySnapshot.php
│   │
│   ├── Reference/
│   │   ├── TypeReferenceFactory.php
│   │   └── DefaultTypeReferenceFactory.php
│   │
│   ├── Diagnostics/
│   │   ├── TypeRegistryDiagnostics.php
│   │   ├── TypeRegistryReport.php
│   │   └── TypeDiagnosticReport.php
│   │
│   └── Exception/
│       ├── DatabaseTypeRegistryException.php
│       ├── UnknownTypeException.php
│       ├── DuplicateTypeRegistrationException.php
│       ├── TypeRegistrationConflictException.php
│       ├── InvalidTypeIdException.php
│       ├── TypeAliasConflictException.php
│       ├── TypeAliasCycleException.php
│       ├── UnknownTypeAliasTargetException.php
│       ├── MissingTypeDependencyException.php
│       ├── CircularTypeDependencyException.php
│       ├── InvalidTypeDefinitionException.php
│       ├── InvalidTypeBehaviorException.php
│       ├── ForbiddenTypeOverrideException.php
│       ├── IncompatibleTypeOverrideException.php
│       ├── FrozenTypeRegistryException.php
│       ├── TypeRegistryCompilationException.php
│       ├── TypeRegistryGenerationMismatchException.php
│       ├── TypeRegistryResourceLimitException.php
│       └── TypeRegistryInvariantViolationException.php
```

---

# 260. Service container proposal

Conceptualmente:

```text
DatabaseServiceProvider
│
├── TypeRegistryBuilder
├── TypeRegistryCompiler
├── TypeRegistrationContributors[]
│
├── compile
│
├── CompiledTypeRegistry
├── CompiledTypeBehaviorRegistry
└── TypeReferenceFactory
```

---

# 261. Bootstrap sequence

```text
VoltStack Bootstrap
        │
        ▼
DatabaseServiceProvider
        │
        ▼
Create TypeRegistryBuilder
        │
        ▼
Register Core Types
        │
        ▼
Register Official Extensions
        │
        ▼
Invoke Plugin Contributors
        │
        ▼
Invoke Application Contributors
        │
        ▼
Collect Aliases
        │
        ▼
Normalize
        │
        ▼
Validate
        │
        ▼
Resolve Dependencies
        │
        ▼
Compile Behaviors
        │
        ▼
Generate Fingerprint
        │
        ▼
Freeze Registry
        │
        ▼
Publish in Container
        │
        ▼
Database Ready
```

---

# 262. Runtime resolution sequence

```text
Input:
    "integer"

        │
        ▼

Normalize textual identifier

        │
        ▼

Alias Map Lookup

        │
        ▼

Canonical TypeId:
    "int"

        │
        ▼

Definition Map Lookup

        │
        ▼

TypeDefinition

        │
        ▼

Consumer
```

---

# 263. TypeReference sequence

```text
"decimal"
+
precision=19
+
scale=4
        │
        ▼
TypeReferenceFactory
        │
        ├── canonicalize TypeId
        ├── resolve definition
        ├── normalize arguments
        ├── validate arguments
        └── create immutable reference
        │
        ▼
decimal(19,4)
```

---

# 264. Registry integration matrix

| Consumer | Registry use |
|---|---|
| ORM Metadata | Resolve persistent property types |
| Query Engine | Resolve typed expressions/parameters |
| Schema | Build canonical column types |
| Schema Introspection | Validate resolved logical types |
| Schema Diff | Obtain type semantics |
| Migration | Analyze type transitions |
| Hydration | Resolve conversion behavior |
| Parameter Binding | Resolve binding behavior |
| Platform | Resolve physical representation |
| Diagnostics | Explain type configuration |
| Extensions | Contribute logical types |

---

# 265. Architectural invariants

## DB-TYPE-REG-001

`TypeRegistry` será la fuente canónica de tipos persistibles conocidos por una generación Database.

## DB-TYPE-REG-002

El registry runtime será immutable.

## DB-TYPE-REG-003

La mutación ocurrirá únicamente durante build/bootstrap.

## DB-TYPE-REG-004

`TypeRegistryBuilder` no será utilizado en hot path.

## DB-TYPE-REG-005

El registry se publicará solo después de validación completa.

## DB-TYPE-REG-006

Un registry parcialmente compilado nunca será visible al runtime.

## DB-TYPE-REG-007

Cada tipo tendrá exactamente un `TypeId` canónico.

## DB-TYPE-REG-008

`TypeId` no será FQCN.

## DB-TYPE-REG-009

Aliases no serán TypeIds canónicos.

## DB-TYPE-REG-010

Aliases serán flattened durante compilación.

## DB-TYPE-REG-011

Runtime alias lookup no recorrerá cadenas recursivas.

## DB-TYPE-REG-012

Alias cycles serán rechazados.

## DB-TYPE-REG-013

Aliases hacia tipos inexistentes serán rechazados.

## DB-TYPE-REG-014

Alias collision será explícito.

## DB-TYPE-REG-015

Un alias no podrá ocultar silenciosamente un TypeId.

## DB-TYPE-REG-016

Compiled metadata utilizará canonical TypeIds.

## DB-TYPE-REG-017

Lookups exitosos serán O(1) promedio.

## DB-TYPE-REG-018

Lookup no utilizará class scanning.

## DB-TYPE-REG-019

Lookup no utilizará reflection.

## DB-TYPE-REG-020

Lookup no hará filesystem discovery.

## DB-TYPE-REG-021

Lookup no hará network I/O.

## DB-TYPE-REG-022

Lookup no abrirá conexiones DB.

## DB-TYPE-REG-023

Unknown type fallará explícitamente cuando se use `get()`.

## DB-TYPE-REG-024

`has()` no lanzará por un tipo simplemente inexistente.

## DB-TYPE-REG-025

Suggestions se calcularán solo en diagnostics/error paths.

## DB-TYPE-REG-026

Core types se registrarán durante bootstrap.

## DB-TYPE-REG-027

Plugins contribuirán tipos mediante contratos explícitos.

## DB-TYPE-REG-028

Plugin discovery pertenecerá al Extension/Package System.

## DB-TYPE-REG-029

TypeRegistry no descubrirá plugins.

## DB-TYPE-REG-030

Contributor ordering será determinista.

## DB-TYPE-REG-031

File discovery order no determinará semántica.

## DB-TYPE-REG-032

Duplicate TypeIds serán detectados.

## DB-TYPE-REG-033

Conflicts no se resolverán mediante last-write-wins por defecto.

## DB-TYPE-REG-034

Override policy default será FORBID.

## DB-TYPE-REG-035

Core types no serán sobrescritos silenciosamente.

## DB-TYPE-REG-036

Overrides permitidos serán explícitos.

## DB-TYPE-REG-037

Overrides serán validados por compatibilidad.

## DB-TYPE-REG-038

Alias será preferido cuando solo se necesite naming alternativo.

## DB-TYPE-REG-039

TypeDefinition será immutable.

## DB-TYPE-REG-040

TypeDefinition no contendrá EntityManager.

## DB-TYPE-REG-041

TypeDefinition no contendrá Connection.

## DB-TYPE-REG-042

TypeDefinition no contendrá request state.

## DB-TYPE-REG-043

TypeDefinition no contendrá tenant mutable state.

## DB-TYPE-REG-044

TypeDefinition no contendrá credentials.

## DB-TYPE-REG-045

TypeBehavior podrá separarse de metadata.

## DB-TYPE-REG-046

TypeBehavior será stateless cuando sea posible.

## DB-TYPE-REG-047

Stateful operations recibirán contexto explícito.

## DB-TYPE-REG-048

Behavior binding será validado durante bootstrap.

## DB-TYPE-REG-049

Database values no seleccionarán behaviors arbitrarios.

## DB-TYPE-REG-050

External strings no provocarán arbitrary class loading.

## DB-TYPE-REG-051

Registry membership no implicará API permission.

## DB-TYPE-REG-052

Authorization permanecerá separada.

## DB-TYPE-REG-053

Type dependencies serán explícitas.

## DB-TYPE-REG-054

Missing dependencies fallarán en bootstrap.

## DB-TYPE-REG-055

Unsupported dependency cycles serán rechazados.

## DB-TYPE-REG-056

Dependency resolution no ocurrirá repetidamente en hot path.

## DB-TYPE-REG-057

El registry tendrá una generación identificable.

## DB-TYPE-REG-058

El registry tendrá fingerprint determinista.

## DB-TYPE-REG-059

Fingerprint no incluirá timestamps.

## DB-TYPE-REG-060

Fingerprint no incluirá object IDs.

## DB-TYPE-REG-061

Fingerprint no incluirá request IDs.

## DB-TYPE-REG-062

Equivalent semantic registry producirá fingerprint equivalente.

## DB-TYPE-REG-063

Semantic registry change invalidará el fingerprint.

## DB-TYPE-REG-064

Caches type-dependent incorporarán registry generation/fingerprint.

## DB-TYPE-REG-065

ORM compiled metadata podrá verificar registry generation.

## DB-TYPE-REG-066

Hydration plans podrán verificar registry generation.

## DB-TYPE-REG-067

Query plans podrán incorporar type generation cuando corresponda.

## DB-TYPE-REG-068

Schema artifacts podrán incorporar type generation.

## DB-TYPE-REG-069

Una operación utilizará una sola registry generation.

## DB-TYPE-REG-070

No habrá mid-operation registry mutation.

## DB-TYPE-REG-071

Development reload creará nueva generación.

## DB-TYPE-REG-072

Development reload no mutará la generación actual.

## DB-TYPE-REG-073

Registry podrá compartirse entre requests en FrankenPHP.

## DB-TYPE-REG-074

Registry podrá compartirse entre jobs en RoadRunner.

## DB-TYPE-REG-075

Registry será read-safe para concurrencia en OpenSwoole.

## DB-TYPE-REG-076

No habrá per-request registry rebuild.

## DB-TYPE-REG-077

No habrá per-request type registration por defecto.

## DB-TYPE-REG-078

Application dynamic runtime registration estará prohibido por defecto.

## DB-TYPE-REG-079

Custom types se registrarán durante bootstrap.

## DB-TYPE-REG-080

Platform-specific logical types podrán existir en el registry global.

## DB-TYPE-REG-081

Registry membership no implicará platform support.

## DB-TYPE-REG-082

Platform support será resuelto por Platform Type Resolver.

## DB-TYPE-REG-083

TypeRegistry no generará SQL.

## DB-TYPE-REG-084

TypeRegistry no ejecutará queries.

## DB-TYPE-REG-085

TypeRegistry no convertirá entidades.

## DB-TYPE-REG-086

TypeRegistry no administrará UoW.

## DB-TYPE-REG-087

TypeRegistry no administrará IdentityMap.

## DB-TYPE-REG-088

TypeRegistry no hará hydration.

## DB-TYPE-REG-089

TypeRegistry no administrará transactions.

## DB-TYPE-REG-090

TypeRegistry no será Service Container.

## DB-TYPE-REG-091

TypeRegistry no será ClassMap.

## DB-TYPE-REG-092

TypeRegistry no será ORM metadata registry.

## DB-TYPE-REG-093

No existirán fuentes contradictorias para tipos persistibles.

## DB-TYPE-REG-094

Specialized registry views deberán delegar al catálogo canónico.

## DB-TYPE-REG-095

Physical DB type names no serán resueltos directamente como logical TypeIds sin Platform resolution.

## DB-TYPE-REG-096

Schema introspection usará Platform resolver antes del TypeRegistry.

## DB-TYPE-REG-097

PHP type inference no reemplazará explicit DB mapping.

## DB-TYPE-REG-098

`string` PHP no implicará automáticamente `string` DB.

## DB-TYPE-REG-099

TypeReferenceFactory canonicalizará aliases.

## DB-TYPE-REG-100

TypeReferenceFactory validará argumentos.

## DB-TYPE-REG-101

Invalid TypeReference no será creado silenciosamente.

## DB-TYPE-REG-102

Canonical TypeReference no conservará alias como identidad.

## DB-TYPE-REG-103

TypeIds registrados formarán un conjunto acotado.

## DB-TYPE-REG-104

TypeId interning podrá ser global por generación.

## DB-TYPE-REG-105

TypeReference interning dinámico deberá ser bounded.

## DB-TYPE-REG-106

Registry deberá resistir contribuciones excesivas.

## DB-TYPE-REG-107

Resource limits podrán aplicarse durante bootstrap.

## DB-TYPE-REG-108

Resource limit failure será explícito.

## DB-TYPE-REG-109

Compiled registry metadata será serializable cuando corresponda.

## DB-TYPE-REG-110

Compiled metadata no almacenará arbitrary closures.

## DB-TYPE-REG-111

Executable behavior se resolverá mediante bindings seguros.

## DB-TYPE-REG-112

Deprecated types permanecerán identificables.

## DB-TYPE-REG-113

Deprecation no emitirá warnings por cada row.

## DB-TYPE-REG-114

Deprecation se reportará preferentemente durante compilation/bootstrap.

## DB-TYPE-REG-115

Deprecated aliases canonicalizarán al replacement cuando la política lo defina.

## DB-TYPE-REG-116

Semantic changes a core TypeIds se tratarán como compatibility-sensitive.

## DB-TYPE-REG-117

New aliases deberán comprobar colisiones con application types.

## DB-TYPE-REG-118

Reserved namespaces serán respetados.

## DB-TYPE-REG-119

Plugin namespaces no serán PHP namespaces implícitos.

## DB-TYPE-REG-120

Diagnostics identificarán registration origin.

## DB-TYPE-REG-121

Conflict diagnostics identificarán ambos contributors.

## DB-TYPE-REG-122

Registry validation intentará reportar múltiples errores cuando sea seguro.

## DB-TYPE-REG-123

Un registry con errores no se publicará.

## DB-TYPE-REG-124

Bootstrap fallará ante errores estructurales.

## DB-TYPE-REG-125

Warnings no se convertirán automáticamente en success guarantees.

## DB-TYPE-REG-126

UNKNOWN platform support seguirá siendo UNKNOWN.

## DB-TYPE-REG-127

Registry diagnostics no expondrán credentials.

## DB-TYPE-REG-128

Registry diagnostics no expondrán user values.

## DB-TYPE-REG-129

Successful lookup no creará spans por defecto.

## DB-TYPE-REG-130

Registry compilation sí podrá ser instrumentada.

## DB-TYPE-REG-131

Telemetry evitará labels de cardinalidad ilimitada.

## DB-TYPE-REG-132

Core registry tendrá conformance tests.

## DB-TYPE-REG-133

Alias graph tendrá cycle tests.

## DB-TYPE-REG-134

Dependency graph tendrá validation tests.

## DB-TYPE-REG-135

Registry fingerprint tendrá determinism tests.

## DB-TYPE-REG-136

Persistent runtime tendrá isolation tests.

## DB-TYPE-REG-137

Concurrent reads deberán ser seguras.

## DB-TYPE-REG-138

External malicious TypeIds no causarán autoload arbitrario.

## DB-TYPE-REG-139

Registry compilation será reproducible.

## DB-TYPE-REG-140

Correctness tendrá prioridad sobre convenient last-write-wins behavior.

## DB-TYPE-REG-141

Un tipo conocido seguirá teniendo una sola definición canónica efectiva por generación.

## DB-TYPE-REG-142

Un alias resolverá como máximo a un TypeId canónico.

## DB-TYPE-REG-143

Un TypeId canónico nunca resolverá ambiguamente.

## DB-TYPE-REG-144

Registry generation mismatch será detectable.

## DB-TYPE-REG-145

No se fabricará compatibilidad entre generaciones incompatibles.

## DB-TYPE-REG-146

TypeRegistry no dependerá de ORM.

## DB-TYPE-REG-147

TypeRegistry no dependerá de Schema.

## DB-TYPE-REG-148

TypeRegistry no dependerá de Query Engine.

## DB-TYPE-REG-149

Los consumidores dependerán de contratos del Registry, no de su implementación concreta.

## DB-TYPE-REG-150

El catálogo lógico permanecerá independiente de la representación física de cada DBMS.

---

# 266. Anti-patterns

## 266.1 Mutable global registry

```php
TypeRegistry::register(...)
```

desde cualquier request.

**Rechazado.**

---

## 266.2 Last registration wins

```php
$types[$id] = $newType;
```

sin conflict analysis.

**Rechazado.**

---

## 266.3 FQCN como TypeId

```text
App\Database\Types\MoneyType
```

como identidad canónica.

**Rechazado.**

---

## 266.4 Reflection durante lookup

**Rechazado.**

---

## 266.5 Filesystem scanning durante query execution

**Rechazado.**

---

## 266.6 Alias chains en runtime

**Rechazado.**

---

## 266.7 Registry diferente para ORM y Schema

**Rechazado.**

---

## 266.8 Physical SQL types dentro del logical registry

```text
VARCHAR(255)
```

como TypeId.

**Rechazado.**

---

## 266.9 Plugin que reemplaza core types silenciosamente

**Rechazado.**

---

## 266.10 Runtime registration en FrankenPHP

**Rechazado.**

---

## 266.11 User input usado para instanciar clase

```php
new ($request->type)();
```

**Rechazado.**

---

## 266.12 Database value usado como FQCN

**Rechazado.**

---

## 266.13 Registry fingerprint con timestamps

**Rechazado.**

---

## 266.14 Rebuild por request

**Rechazado.**

---

## 266.15 Registry como Service Container genérico

**Rechazado.**

---

# 267. Ejemplo de registro core

Conceptualmente:

```php
$registry->register(
    new TypeRegistration(
        definition: new TypeDefinition(
            id: new TypeId('uuid'),
            family: TypeFamily::IDENTIFIER,
            php: new PhpTypeDescriptor(
                accepted: [
                    'string',
                    Uuid::class,
                ],
            ),
            arguments: new UuidTypeArgumentSchema(),
            behavior: new TypeBehaviorReference('database.type.uuid'),
        ),
        behavior: new TypeBehaviorReference(
            'database.type.uuid',
        ),
        origin: TypeRegistrationOrigin::core(),
        priority: TypeRegistrationPriority::CORE,
    ),
);
```

---

# 268. Ejemplo custom type

Aplicación:

```php
final class ApplicationDatabaseTypes
    implements TypeRegistrationContributor
{
    public function contribute(
        TypeRegistryBuilder $registry,
    ): void {
        $registry->register(
            TypeRegistration::application(
                definition: MoneyTypeDefinition::create(),
                behavior: new TypeBehaviorReference(
                    'app.database.type.money',
                ),
            ),
        );
    }
}
```

---

# 269. Resultado compilado

```text
TypeRegistry
│
├── bool
├── int
├── bigint
├── decimal
├── string
├── text
├── uuid
├── ulid
├── json
├── datetime
└── app.money
```

---

# 270. Alias example

```php
$registry->alias(
    new TypeAliasRegistration(
        alias: 'integer',
        target: new TypeId('int'),
    ),
);
```

Runtime:

```text
integer
   │
   ▼
  int
   │
   ▼
Integer TypeDefinition
```

---

# 271. Plugin example

```text
Package:
    voltstack/vector

Contributes:
    vector

Platforms:
    PostgreSQL → native extension mapping
    MySQL      → unsupported
    MariaDB    → unsupported
    SQLite     → optional emulation
```

El registry sabe:

```text
vector exists
```

La Platform decide:

```text
can vector be represented here?
```

---

# 272. Registry generation example

Generation A:

```text
bool
int
decimal
string
uuid
json
```

Fingerprint:

```text
dbtypes:89a41...
```

Aplicación agrega:

```text
app.money
```

Generation B:

```text
bool
int
decimal
string
uuid
json
app.money
```

Fingerprint:

```text
dbtypes:c72ef...
```

Un HydrationPlan compilado contra A podrá ser invalidado si depende del Registry y se intenta utilizar bajo B.

---

# 273. Integration with ORM metadata

```text
Entity Attribute
    type="integer"
         │
         ▼
TypeReferenceFactory
         │
         ▼
Alias Resolver
         │
         ▼
TypeId("int")
         │
         ▼
TypeRegistry
         │
         ▼
Integer TypeDefinition
         │
         ▼
Validated TypeReference
         │
         ▼
Compiled Entity Metadata
```

---

# 274. Integration with Schema

```text
$table->decimal('price', 19, 4)
              │
              ▼
       TypeReferenceFactory
              │
              ▼
       TypeId("decimal")
              │
              ▼
        TypeRegistry
              │
              ▼
     Decimal Definition
              │
              ▼
   Validate precision/scale
              │
              ▼
       TypeReference
```

---

# 275. Integration with Query Engine

```text
Query Expression
      │
      ▼
Semantic Analysis
      │
      ▼
Expected TypeId
      │
      ▼
TypeRegistry
      │
      ▼
Type Semantics
      │
      ▼
Parameter Type
```

---

# 276. Integration with Conversion

```text
TypeReference
      │
      ▼
TypeRegistry
      │
      ▼
BehaviorReference
      │
      ▼
TypeBehaviorRegistry
      │
      ▼
Value Converter
```

El siguiente documento especializará esta frontera.

---

# 277. Master formula

El Registry System puede resumirse como:

```text
R =
Compile(
    Normalize(
        Validate(
            CoreTypes
            ∪ OfficialTypes
            ∪ PluginTypes
            ∪ ApplicationTypes
            ∪ Aliases
        )
    )
)
```

sujeto a:

```text
UniqueCanonicalTypeId
∧
DeterministicResolution
∧
ValidDependencies
∧
NoAliasCycles
∧
ExplicitConflictPolicy
∧
StableBehaviorBindings
```

produciendo:

```text
R = Immutable Type Registry Generation
```

---

# 278. Canonical resolution formula

Para una entrada textual `x`:

```text
normalize(x)
      ↓
aliasMap
      ↓
canonical TypeId T
      ↓
definitionMap[T]
      ↓
TypeDefinition(T)
```

Formalmente:

```text
Resolve(x) =
Definition(
    Canonicalize(
        Normalize(x)
    )
)
```

y debe cumplirse:

```text
∀x:
Resolve(x)
```

es:

```text
exactly one TypeDefinition
```

o:

```text
explicit failure
```

Nunca una resolución ambigua.

---

# 279. Registry consistency formula

Para una generación `G`:

```text
G =
(
    Definitions,
    Aliases,
    BehaviorBindings,
    Dependencies,
    Policies
)
```

deberá cumplirse:

```text
Immutable(G)
∧
Deterministic(G)
∧
Validated(G)
∧
SelfConsistent(G)
```

---

# 280. Regla de runtime

Una operación `O` deberá observar:

```text
RegistryGeneration(O.start)
=
RegistryGeneration(O.end)
```

en términos de la generación efectiva utilizada por dicha operación.

---

# 281. Decisiones arquitectónicas finales

VoltStack adoptará:

```text
Build → Validate → Compile → Freeze
```

en lugar de un registry mutable permanente.

Adoptará:

```text
Canonical TypeId
```

en lugar de FQCN como identidad.

Adoptará:

```text
Flattened Alias Map
```

en lugar de cadenas dinámicas.

Adoptará:

```text
Explicit Registration Contributors
```

en lugar de class scanning durante runtime.

Adoptará:

```text
Conflict Detection
```

en lugar de last-write-wins.

Adoptará:

```text
Registry Generation + Fingerprint
```

para proteger metadata y caches compilados.

Adoptará:

```text
Immutable Shared Registry
```

para FrankenPHP, RoadRunner y OpenSwoole.

Adoptará:

```text
TypeDefinition
+
TypeBehavior binding
```

para separar metadata de comportamiento ejecutable.

---

# 282. Regla maestra

> **VoltStack conocerá todos los tipos disponibles antes de entrar al hot path. El runtime resolverá identificadores canónicos mediante estructuras immutable e indexadas; nunca descubrirá clases, plugins o comportamientos de tipos mientras ejecuta una query, hidrata una entidad o procesa una petición.**

Esto produce:

```text
Bootstrap Complexity
        ↓
compiled once

Runtime Complexity
        ↓
simple deterministic lookup
```

y permite:

```text
Extensibility
+
Performance
+
Determinism
+
Persistent Runtime Safety
+
Cross-System Consistency
```

sin convertir el Type System en un registro global mutable.

---

# 283. Relación con el siguiente documento

Hasta este punto:

```text
155_DATABASE_TYPE_SYSTEM
        │
        │ defines
        ▼
Type Model
        │
        ▼
156_DATABASE_TYPE_REGISTRY_SYSTEM
        │
        │ resolves
        ▼
TypeDefinition + TypeBehavior
        │
        ▼
157_DATABASE_VALUE_CONVERSION_SYSTEM
```

El Registry responde:

> ¿Qué tipo es este y qué comportamiento tiene registrado?

El siguiente sistema responderá:

> ¿Cómo se transforma de manera segura un valor entre su representación PHP/domain y su representación database/driver?

---

# 284. Siguiente documento

```text
157_DATABASE_VALUE_CONVERSION_SYSTEM.md
```

Regla central propuesta:

> **Value Conversion será una frontera bidireccional, explícita y tipada entre representaciones PHP/domain y database/driver; toda conversión deberá conocer su `TypeReference`, dirección y contexto de plataforma, preservar semántica cuando se declare lossless y rechazar overflow, pérdida de precisión, ambigüedad o coerción insegura en lugar de modificar silenciosamente el valor.**

Deberá cubrir:

- `TypeValueConverter`;
- conversion pipeline;
- PHP → Database;
- Database → PHP;
- logical → physical representation;
- conversion context;
- conversion direction;
- null handling;
- scalar conversion;
- integer overflow;
- decimal precision;
- floating-point semantics;
- boolean normalization;
- string conversion;
- binary/LOB conversion;
- UUID/ULID conversion;
- temporal conversion;
- JSON conversion;
- enum integration;
- Value Object integration;
- custom converters;
- conversion composition;
- lossless/lossy classification;
- explicit coercion policies;
- round-trip guarantees;
- driver raw values;
- platform-specific representations;
- conversion cache boundaries;
- persistent runtime safety;
- security;
- diagnostics;
- telemetry;
- testing;
- error taxonomy;
- architectural invariants.