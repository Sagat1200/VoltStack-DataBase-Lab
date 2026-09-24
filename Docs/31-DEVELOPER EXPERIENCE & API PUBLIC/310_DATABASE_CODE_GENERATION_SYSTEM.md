# 310_DATABASE_CODE_GENERATION_SYSTEM.md

# VoltStack Quantum Database
## Database Code Generation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 310 — Database Code Generation System  
**Bloque:** 31 — Developer Experience / Public API  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `309_DATABASE_CLI_SYSTEM.md`  
**Siguiente documento:** `311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md`

---

# 1. Propósito

Este documento define la arquitectura oficial del sistema de generación de código de `VoltStack/Quantum/Database`.

El objetivo es proporcionar una infraestructura unificada para generar, inspeccionar, previsualizar y actualizar artefactos de desarrollo relacionados con Database, incluyendo:

- Entities;
- Models;
- Repositories;
- Migrations;
- Seeders;
- Factories;
- Fixtures;
- Value Objects;
- custom database types;
- mapping metadata;
- driver skeletons;
- dialect skeletons;
- compiler extensions;
- query extensions;
- ORM extensions;
- testing skeletons.

La regla central será:

> **Database Code Generation transforma una descripción estructurada de intención o metadata en artefactos de código fuente reproducibles; nunca sustituye al Database Runtime ni convierte código generado en fuente absoluta de verdad sobre la base de datos.**

Formalmente:

```text
Code Generation
=
Structured Input
+ Metadata
+ Generation Rules
+ Templates
→ Source Artifacts
```

pero:

```text
Code Generator
≠
Database Engine
```

y:

```text
Generated Code
≠
Database State
```

---

# 2. Motivación

VoltStack Database posee una arquitectura extensa:

```text
Driver
Connection
Query Engine
Schema
Migration
ORM
Persistence
Hydration
Relationships
Transactions
Cache
Events
Telemetry
Security
Testing
Extensions
```

Crear manualmente cada artefacto puede provocar:

- errores de namespace;
- convenciones inconsistentes;
- metadata incompleta;
- relaciones incorrectas;
- duplicación;
- boilerplate;
- imports incorrectos;
- configuraciones incompatibles;
- skeletons de extensiones incompletos.

Code Generation reducirá esa fricción sin esconder la arquitectura.

---

# 3. Principio de diseño

VoltStack deberá mantener:

```text
Developer Intent
      ↓
Generation Request
      ↓
Generation Model
      ↓
Metadata / Schema / Conventions
      ↓
Generation Planner
      ↓
Artifact Definitions
      ↓
Template / AST Generation
      ↓
Validation
      ↓
Preview / Diff
      ↓
Write
```

Nunca:

```text
CLI
→ concatenate strings
→ write random PHP file
```

---

# 4. Code Generation ≠ Runtime

El sistema generado podrá producir código utilizado posteriormente por Runtime.

Pero el generador no deberá participar normalmente en:

```text
HTTP request execution
query execution
entity hydration
transaction execution
ORM flush
SQL compilation
```

La relación será:

```text
Development Time
     │
     ▼
Code Generator
     │
     ▼
Source Code
     │
     ▼
Application Build / Runtime
```

---

# 5. Objetivos

El sistema deberá proporcionar:

1. generación reproducible;
2. convenciones centralizadas;
3. templates versionados;
4. soporte para dry-run;
5. preview;
6. diff;
7. detección de conflictos;
8. protección contra sobrescritura;
9. validación sintáctica;
10. validación arquitectónica;
11. generación desde schema;
12. generación desde metadata;
13. generación desde intención del desarrollador;
14. extensibilidad;
15. testing;
16. integración CLI;
17. integración IDE futura;
18. soporte machine-readable;
19. compatibilidad con plugins;
20. trazabilidad.

---

# 6. No objetivos

El sistema no deberá:

- ejecutar queries arbitrariamente;
- sustituir Schema Builder;
- sustituir Migration System;
- inferir semántica empresarial inexistente;
- modificar producción automáticamente;
- generar SQL como responsabilidad principal;
- ejecutar migraciones;
- decidir autorización de negocio;
- introducir lógica oculta en entidades;
- reescribir archivos arbitrariamente.

---

# 7. Arquitectura general

```text
Developer / CLI / IDE
          │
          ▼
 Generation Request
          │
          ▼
 Request Normalizer
          │
          ▼
 Generation Context
          │
          ├── Project Context
          ├── Database Metadata
          ├── Schema Metadata
          ├── ORM Metadata
          ├── Naming Conventions
          ├── Platform Capabilities
          └── Extension Registry
          │
          ▼
 Generation Planner
          │
          ▼
 Generation Plan
          │
    ┌─────┼───────────┐
    ▼     ▼           ▼
Artifacts Templates Dependencies
    │
    ▼
 Artifact Generator
    │
    ▼
 Generated Artifact
    │
    ▼
 Validators
    │
    ▼
 Conflict Detector
    │
    ▼
 Preview / Diff
    │
    ▼
 Write Policy
    │
    ▼
 Filesystem Writer
```

---

# 8. Arquitectura por capas

Se proponen cinco capas:

```text
1. Intent Layer
2. Planning Layer
3. Generation Layer
4. Validation Layer
5. Persistence Layer
```

---

# 9. Intent Layer

Representará qué desea generar el desarrollador.

Ejemplo:

```text
Generate Entity:
User
```

o:

```text
Generate Migration:
Add status to orders
```

No deberá contener detalles innecesarios de filesystem.

---

# 10. Generation Request

Contrato conceptual:

```php
interface GenerationRequest
{
    public function type(): GenerationType;

    public function target(): GenerationTarget;
}
```

---

# 11. Generation Type

Ejemplos:

```php
enum GenerationType: string
{
    case ENTITY = 'entity';
    case MODEL = 'model';
    case REPOSITORY = 'repository';
    case MIGRATION = 'migration';
    case SEEDER = 'seeder';
    case FACTORY = 'factory';
    case FIXTURE = 'fixture';
    case VALUE_OBJECT = 'value_object';
    case CUSTOM_TYPE = 'custom_type';
    case DRIVER = 'driver';
    case DIALECT = 'dialect';
    case COMPILER = 'compiler';
    case QUERY_EXTENSION = 'query_extension';
    case ORM_EXTENSION = 'orm_extension';
}
```

---

# 12. Generation Context

El request deberá resolverse dentro de un contexto.

```php
final readonly class GenerationContext
{
    public function __construct(
        public ProjectGenerationContext $project,
        public NamingConventionSet $naming,
        public GenerationPolicy $policy,
        public ?SchemaMetadata $schema,
        public ?OrmMetadata $orm,
        public ?DatabaseCapabilitySnapshot $capabilities,
    ) {}
}
```

---

# 13. Generation Context ≠ DatabaseContext

Debe mantenerse:

```text
GenerationContext
≠
DatabaseContext
```

`DatabaseContext` pertenece a ejecución Database.

`GenerationContext` pertenece a generación de artefactos.

---

# 14. Generation Planner

El planner transformará:

```text
GenerationRequest
+
GenerationContext
```

en:

```text
GenerationPlan
```

---

# 15. Generation Plan

Ejemplo conceptual:

```php
final readonly class GenerationPlan
{
    /**
     * @param list<ArtifactPlan> $artifacts
     */
    public function __construct(
        public GenerationPlanId $id,
        public array $artifacts,
        public GenerationDependencyGraph $dependencies,
        public GenerationDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 16. Plan ≠ Files

Un `GenerationPlan` describe lo que deberá generarse.

No representa todavía archivos escritos.

```text
Generation Plan
≠
Filesystem Mutation
```

---

# 17. Artifact Plan

Cada artefacto deberá especificar:

```text
type
logical name
namespace
target path
template/generator
dependencies
overwrite policy
validation rules
```

---

# 18. Ejemplo

Generar:

```text
User Entity
```

podría producir:

```text
GenerationPlan
│
├── App\Entity\User.php
├── App\Repository\UserRepository.php
└── Database\Factory\UserFactory.php
```

si la política solicita artefactos relacionados.

---

# 19. Generación atómica conceptual

Un request podrá producir varios archivos relacionados.

La operación deberá tratarlos como:

```text
Generation Unit
```

para evitar dejar un proyecto parcialmente generado sin reportarlo.

---

# 20. Atomicidad filesystem

No siempre será posible garantizar atomicidad física universal.

Por ello deberán existir resultados:

```text
SUCCESS
PARTIAL_SUCCESS
FAILED
UNKNOWN
```

---

# 21. Artifact Model

Antes de renderizar archivos se recomienda utilizar una representación intermedia.

```text
Generation Request
      ↓
Artifact Model
      ↓
Renderer
      ↓
Source Code
```

---

# 22. Artifact Model ≠ Source String

Evitar:

```php
$code = "<?php\nclass {$name}...";
```

como arquitectura principal.

Preferible:

```text
PHP Artifact Model
├── Namespace
├── Imports
├── Attributes
├── Class
├── Interfaces
├── Traits
├── Properties
├── Constructor
└── Methods
```

---

# 23. PHP Code Model

VoltStack podrá definir:

```text
PhpFileDefinition
NamespaceDefinition
ImportDefinition
ClassDefinition
AttributeDefinition
PropertyDefinition
MethodDefinition
ParameterDefinition
TypeDefinition
ExpressionDefinition
```

---

# 24. AST vs Template

No todos los artefactos requerirán el mismo mecanismo.

Podrán coexistir:

```text
Structured PHP AST/Model
Template
Specialized Generator
```

---

# 25. Regla de selección

Preferencia:

```text
Structured Model
>
Template for structure
>
Raw string concatenation
```

---

# 26. Template System

Templates deberán ser:

- versionados;
- identificables;
- deterministas;
- extensibles;
- testeables.

---

# 27. Template Identity

Ejemplo:

```text
database.entity.php
version: 1
```

---

# 28. Template Registry

```php
interface GenerationTemplateRegistry
{
    public function get(TemplateId $id): GenerationTemplate;

    public function has(TemplateId $id): bool;
}
```

---

# 29. Registry freeze

Después del bootstrap:

```text
TemplateRegistry
→ FROZEN
```

para evitar cambios inesperados durante una generación.

---

# 30. Template overrides

El proyecto podrá personalizar templates.

Pero la personalización deberá ser explícita.

---

# 31. Template precedence

Ejemplo:

```text
Project Template
      ↓
Plugin Template
      ↓
VoltStack Default Template
```

La precedencia deberá ser determinista.

---

# 32. Template override ≠ Core mutation

Personalizar una plantilla no deberá requerir modificar archivos dentro de:

```text
vendor/
```

---

# 33. Template compatibility

Cada template podrá declarar:

```text
generator version
artifact type
minimum VoltStack version
required features
```

---

# 34. Naming System

Code Generation deberá consumir las convenciones definidas por VoltStack.

Ejemplos:

```text
User
→ users

OrderItem
→ order_items

CreateUsersTable
→ migration naming
```

---

# 35. Naming ≠ Guessing

El generador no deberá asumir que toda pluralización o transformación es inequívoca.

El desarrollador podrá especificar:

```text
--table=people
```

cuando sea necesario.

---

# 36. Naming Convention Service

```php
interface DatabaseNamingConvention
{
    public function entityToTable(EntityName $name): TableName;

    public function tableToEntity(TableName $table): EntityName;
}
```

---

# 37. Naming provenance

Cuando una decisión sea inferida, podrá registrarse:

```text
name:
User

table:
users

source:
default naming convention
```

---

# 38. Entity Generation

El generador deberá poder producir Entities.

Ejemplo:

```text
php voltstack make:entity User
```

---

# 39. Entity output

Ejemplo conceptual:

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use VoltStack\Quantum\Database\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table('users')]
final class User
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    private int $id;

    #[ORM\Column(type: 'string', length: 255)]
    private string $email;
}
```

La sintaxis final dependerá del mapping system oficial.

---

# 40. Entity Generator ≠ ORM

El generator conoce:

```text
mapping syntax
conventions
metadata model
```

pero no:

```text
IdentityMap state
UnitOfWork state
active transaction
```

---

# 41. Entity generation modes

Podrán existir:

```text
empty
interactive
from-schema
from-metadata
```

---

# 42. Empty entity

```text
make:entity User
```

produce skeleton mínimo.

---

# 43. Interactive entity

Podrá preguntar:

```text
Field name:
Type:
Nullable:
Length:
```

y construir un GenerationRequest.

---

# 44. Interactive input ≠ Generator semantics

La CLI únicamente recopila información.

El Code Generation System produce el artefacto.

---

# 45. Entity from schema

Podrá existir:

```text
database:generate:entities --from-schema
```

o API equivalente.

Flujo:

```text
Database
   ↓
Schema Introspection
   ↓
Normalized Schema Model
   ↓
Schema → ORM Mapping Analysis
   ↓
Generation Plan
   ↓
Entity Artifacts
```

---

# 46. Schema ≠ Entity Model

Regla crítica:

> Una tabla relacional no contiene suficiente información para reconstruir inequívocamente un modelo de dominio.

Por tanto:

```text
Table
≠
Entity Semantics
```

---

# 47. Reverse engineering

La generación desde schema será:

```text
Reverse Engineering
```

y deberá tratar sus inferencias como inferencias.

---

# 48. Ambiguity

Ejemplo:

```text
users.status VARCHAR(20)
```

no permite saber automáticamente si:

```text
status = string
status = enum
status = value object
```

---

# 49. Inference confidence

Podrá utilizarse:

```text
EXACT
HIGH
MEDIUM
LOW
AMBIGUOUS
UNKNOWN
```

---

# 50. Ambiguous generation

Por defecto:

```text
AMBIGUOUS
→ explicit developer decision
```

en decisiones estructurales importantes.

---

# 51. Relationship inference

Foreign keys podrán sugerir relaciones.

Ejemplo:

```text
orders.user_id
→ probable ManyToOne Order → User
```

Pero:

```text
FK
≠
complete ORM relationship semantics
```

---

# 52. Many-to-many inference

Una tabla:

```text
user_roles
```

podría ser:

```text
join table
```

o:

```text
association entity
```

si contiene datos propios.

El generator no deberá destruir esta distinción.

---

# 53. Association entity detection

Si una tabla contiene:

```text
user_id
role_id
created_at
granted_by
```

podría sugerirse:

```text
RoleAssignment Entity
```

en lugar de ManyToMany puro.

---

# 54. Model API generation

Si VoltStack utiliza una API Laravel-like:

```php
final class User extends Model
{
}
```

el generador podrá producirla.

---

# 55. Model API ≠ Entity Engine

Debe conservarse:

```text
Model API
→ ORM
```

y no:

```text
Model API
→ independent persistence engine
```

---

# 56. Repository Generation

Ejemplo:

```text
php voltstack make:repository UserRepository --entity=User
```

---

# 57. Repository skeleton

```php
final class UserRepository extends EntityRepository
{
    public function entityClass(): string
    {
        return User::class;
    }
}
```

La API definitiva dependerá del Repository System.

---

# 58. Repository generator

No deberá generar automáticamente decenas de métodos:

```text
findByName()
findByEmail()
findByStatus()
```

sin intención explícita.

---

# 59. Migration Generation

Ejemplo:

```text
php voltstack make:migration CreateUsersTable
```

---

# 60. Migration output

El generador deberá utilizar las APIs del Migration/Schema System.

No SQL hard-coded como default.

Ejemplo conceptual:

```php
final class CreateUsersTable extends Migration
{
    public function up(Schema $schema): void
    {
        $schema->create('users', function (Table $table): void {
            $table->id();
            $table->string('email');
        });
    }
}
```

---

# 61. Migration generator ≠ Migration planner

Debe mantenerse:

```text
Migration Code Generator
≠
Migration Planner
```

---

# 62. Migration timestamp/id

La identidad de migraciones deberá provenir del Migration System.

No de concatenaciones ad hoc en CLI.

---

# 63. Collision prevention

Dos generaciones simultáneas deberán evitar producir:

```text
same migration identifier
```

cuando el sistema de identificación lo requiera.

---

# 64. Schema diff generation

Podrá existir:

```text
database:migration:generate
```

basado en:

```text
Current Schema
→ Target Schema
→ Schema Diff
→ Migration Planner
→ Migration Generation Model
→ Migration Source Code
```

---

# 65. Diff ≠ generated migration automatically safe

Aunque una migration sea generada desde Schema Diff:

```text
Generated
≠
Safe
```

---

# 66. Rename ambiguity

Ejemplo:

```text
remove surname
add last_name
```

podría significar rename.

Pero también dos operaciones independientes.

El generator no deberá asumir rename sin evidencia suficiente.

---

# 67. Destructive migration generation

Si genera:

```text
DROP COLUMN
```

deberá emitir diagnostics visibles.

---

# 68. Seeder Generation

Ejemplo:

```text
php voltstack make:seeder UserSeeder
```

---

# 69. Seeder skeleton

Deberá integrar:

```text
SeederSystem
FactorySystem
TestDataGenerator
```

según corresponda.

---

# 70. Factory Generation

Ejemplo:

```text
php voltstack make:factory UserFactory --entity=User
```

---

# 71. Factory metadata

Podrá analizar Entity Metadata para proponer:

```text
field generators
default values
relationships
```

---

# 72. Generated fake data ≠ production data

Nunca deberá copiar automáticamente valores reales desde producción para crear factories.

---

# 73. Fixture Generation

Podrá generar:

```text
Fixture skeleton
reference registry
dependencies
```

---

# 74. Value Object Generation

Ejemplo:

```text
php voltstack make:value-object Money
```

podrá producir un skeleton compatible con:

```text
DATABASE_VALUE_OBJECT_MAPPING_SYSTEM
```

---

# 75. Custom Type Generation

```text
php voltstack make:database-type MoneyType
```

podrá producir:

```text
TypeId
PHP conversion
database conversion
platform mapping hooks
tests
```

---

# 76. Custom Type skeleton ≠ registered type

Generar el archivo no significa que el tipo esté activo.

Podrá requerir registro/configuración.

---

# 77. Driver Generation

El sistema podrá generar skeletons para drivers externos.

```text
php voltstack make:database-driver AcmeDb
```

---

# 78. Driver skeleton

Podrá incluir:

```text
Driver
Connection adapter
Statement adapter
Result adapter
Error mapper
Capability provider
Service provider/plugin
Conformance test bootstrap
Manifest
```

---

# 79. Driver generator ≠ Driver implementation

Regla:

```text
Generated Driver Skeleton
≠
Conformant Driver
```

---

# 80. Conformance integration

El skeleton deberá incluir integración con:

```text
292_DATABASE_DRIVER_CONFORMANCE_TESTING_SYSTEM.md
```

---

# 81. Driver manifest

Podrá generarse:

```text
voltstack-driver.json
```

con información mínima.

---

# 82. Dialect Generation

```text
php voltstack make:database-dialect AcmeSql
```

podrá generar:

```text
Dialect
identifier rules
literal rules
feature declarations
tests
```

---

# 83. Dialect ≠ Driver

El generador deberá conservar:

```text
Driver
≠
Dialect
≠
Platform
≠
Compiler
```

---

# 84. Compiler Generation

```text
php voltstack make:database-compiler AcmeSql
```

podrá generar skeletons para:

```text
compiler
node visitors
feature compilers
tests
```

---

# 85. Compiler extension

Podrá existir un modo:

```text
make:database-compiler-extension
```

para no requerir un compiler completo.

---

# 86. Query Extension Generation

Podrá generar:

```text
AST Node
Builder extension
Semantic rule
Optimizer rule
Compiler integration
tests
```

dependiendo del tipo de extensión.

---

# 87. Query extension completeness

Si una nueva operación requiere:

```text
AST
Semantic Analysis
Compiler
Capability
```

el generador podrá advertir cuando falte alguna pieza.

---

# 88. ORM Extension Generation

Podrá generar skeletons para:

```text
mapping extension
lifecycle extension
metadata extension
hydration extension
repository extension
type extension
```

según los puntos oficiales de extensión.

---

# 89. Generator recipes

Para artefactos complejos se recomienda utilizar:

```text
Generation Recipe
```

---

# 90. Recipe

Una recipe describe un conjunto de artefactos.

Ejemplo:

```text
CustomDriverRecipe
│
├── Driver
├── Connection
├── Statement
├── Result
├── ErrorMapper
├── CapabilityProvider
├── Manifest
└── ConformanceTests
```

---

# 91. Recipe ≠ Template

```text
Recipe
=
multi-artifact generation orchestration
```

```text
Template
=
representation strategy for one artifact or fragment
```

---

# 92. Generation dependency graph

Los artefactos podrán poseer dependencias.

```text
Entity
  ↓
Repository

Entity
  ↓
Factory
```

---

# 93. Dependency ordering

El planner deberá producir un DAG cuando sea necesario.

---

# 94. Cycles

Dependencias cíclicas inválidas deberán producir diagnostics antes de escribir archivos.

---

# 95. Existing file detection

Antes de escribir:

```text
Target Path
→ exists?
```

---

# 96. Default overwrite policy

La política por defecto será:

```text
DO_NOT_OVERWRITE
```

---

# 97. Overwrite modes

Podrán existir:

```text
CREATE_ONLY
FAIL_IF_EXISTS
MERGE_SAFE
REPLACE_GENERATED_REGION
REPLACE_FILE
```

---

# 98. Replace file

`REPLACE_FILE` deberá requerir intención explícita.

---

# 99. `--force` y code generation

Incluso si CLI ofrece:

```text
--force
```

no deberá sobrescribir silenciosamente código del desarrollador sin política clara.

---

# 100. Generated ownership

El sistema deberá distinguir:

```text
generator-owned file
developer-owned file
mixed ownership file
```

---

# 101. Fully generated files

Un archivo totalmente generado podrá contener metadata:

```text
@Generated(...)
```

o mecanismo equivalente.

---

# 102. Generated markers

Podrá incluir:

```text
generated by VoltStack
generator version
template version
```

cuando sea apropiado.

---

# 103. Generated marker ≠ permission to overwrite forever

Si el usuario modifica manualmente el archivo, el sistema deberá poder detectar drift cuando sea posible.

---

# 104. Generated regions

Para mixed ownership:

```text
// <voltstack-generated:fields>

// </voltstack-generated:fields>
```

podrían utilizarse regiones controladas.

---

# 105. Generated regions policy

Su uso deberá ser limitado.

Demasiados markers producen código difícil de mantener.

---

# 106. Prefer regeneration-friendly architecture

Cuando sea posible:

```text
generated base artifact
+
developer extension artifact
```

puede ser preferible a modificar continuamente el mismo archivo.

---

# 107. Example

```text
GeneratedUserMetadata.php
        ↑
User.php
```

sólo cuando esta separación tenga sentido arquitectónico.

---

# 108. Source parsing

Para modificar archivos existentes de forma segura, no deberá utilizarse únicamente regex.

---

# 109. PHP parser

Cuando se necesite merge estructural:

```text
Existing PHP
→ Parser
→ PHP AST
→ Modification
→ Printer
```

---

# 110. Regex limitation

Evitar:

```php
str_replace('class User', ...)
```

para transformaciones complejas.

---

# 111. Conflict Detection System

Antes de escribir deberá analizarse:

```text
Path Conflict
Namespace Conflict
Class Conflict
Existing Symbol Conflict
Generated Region Conflict
Template Conflict
Dependency Conflict
```

---

# 112. Conflict result

```php
enum GenerationConflictType
{
    case FILE_EXISTS;
    case SYMBOL_EXISTS;
    case GENERATED_REGION_CHANGED;
    case NAMESPACE_MISMATCH;
    case TARGET_CHANGED;
}
```

---

# 113. Conflict ≠ Error siempre

Algunos conflictos podrán resolverse mediante:

```text
skip
rename
merge
explicit replace
```

---

# 114. Silent conflict resolution

No deberá existir para cambios destructivos.

---

# 115. Preview System

Toda generación deberá poder ejecutarse en modo:

```text
PREVIEW
```

---

# 116. Preview

Ejemplo:

```text
php voltstack make:entity User --dry-run
```

Resultado:

```text
Would create:

app/Entity/User.php
app/Repository/UserRepository.php

No files were written.
```

---

# 117. Preview ≠ write

La regla deberá ser estricta:

```text
Preview
→ zero target source mutations
```

---

# 118. Diff preview

Para actualizaciones:

```text
php voltstack database:generate:entities --from-schema --dry-run
```

podrá mostrar:

```diff
 final class User
 {
+    private ?string $phone = null;
 }
```

---

# 119. Diff ≠ actual filesystem state after execution

Entre preview y write el archivo puede cambiar.

---

# 120. Optimistic filesystem concurrency

El plan podrá almacenar:

```text
FileFingerprint
```

---

# 121. Write precondition

Antes de escribir:

```text
CurrentFingerprint
==
PlannedFingerprint
```

cuando se modifica un archivo existente.

---

# 122. Changed file

Si cambió:

```text
GENERATION_CONFLICT
```

en lugar de sobrescribir.

---

# 123. Fingerprint

Podrá calcularse mediante:

```text
content hash
+
path identity
```

según implementación.

---

# 124. Filesystem writer

El writer será una abstracción.

```php
interface GeneratedArtifactWriter
{
    public function write(
        GeneratedArtifact $artifact,
        WritePolicy $policy,
    ): ArtifactWriteResult;
}
```

---

# 125. Generator ≠ Filesystem Writer

Debe mantenerse:

```text
Generator
≠
Writer
```

Esto facilita testing.

---

# 126. Temporary writes

Para reemplazos seguros podrá utilizarse:

```text
write temp
→ validate
→ atomic rename where supported
```

---

# 127. Atomic rename

No deberá asumirse universalmente.

La capability del filesystem deberá considerarse.

---

# 128. Partial write recovery

Si una generación multiarchivo falla:

```text
created files
modified files
failed files
unknown files
```

deberán registrarse.

---

# 129. Rollback filesystem

Podrá intentarse cuando sea seguro.

Pero:

```text
Generation Rollback
≠
Guaranteed filesystem transaction
```

---

# 130. Backup before modification

Para modificaciones sensibles podrá existir:

```text
backup existing file
```

según policy.

---

# 131. Validation Pipeline

Antes de persistir:

```text
Generated Artifact
      ↓
Structural Validation
      ↓
Syntax Validation
      ↓
Architecture Validation
      ↓
Convention Validation
      ↓
Conflict Validation
      ↓
Write
```

---

# 132. PHP syntax validation

Los archivos PHP generados deberán ser sintácticamente válidos.

---

# 133. Architecture validation

Ejemplos:

```text
Entity namespace valid
Repository references existing entity
Custom type has TypeId
Driver skeleton has required contracts
```

---

# 134. Validation ≠ runtime conformance

Un driver generado puede ser sintácticamente válido pero no conforme.

---

# 135. Post-write validation

Opcionalmente:

```text
Write
→ re-read
→ validate fingerprint/content
```

para detectar problemas de filesystem.

---

# 136. Formatting

El generator podrá producir formato canónico.

Pero no deberá depender obligatoriamente de herramientas externas para funcionar.

---

# 137. External formatter

Podrá integrarse opcionalmente con:

```text
PHP-CS-Fixer
Pint-like tools
project formatter
```

mediante adapters.

---

# 138. Formatting failure

No deberá confundirse con fallo de generación semántica.

Resultado ejemplo:

```text
Generation: SUCCESS
Formatting: FAILED
```

o política equivalente.

---

# 139. Code style

VoltStack deberá definir una base:

```text
strict_types
namespace rules
imports
visibility
readonly where appropriate
final where appropriate
typing
```

---

# 140. Import Resolution

El sistema deberá resolver imports determinísticamente.

---

# 141. Import collisions

Ejemplo:

```text
App\Entity\User
Vendor\Package\User
```

deberá generar alias cuando sea necesario.

---

# 142. Fully qualified fallback

Cuando exista conflicto podrá utilizarse:

```php
\Vendor\Package\User
```

según printer policy.

---

# 143. Namespace resolution

El sistema deberá conocer PSR-4/autoload mappings del proyecto.

---

# 144. Composer integration

Podrá leer metadata del proyecto para resolver namespaces.

Pero:

```text
Code Generator
≠
Composer
```

---

# 145. Autoload update

Si una recipe crea un nuevo package/namespace, cualquier modificación de autoload deberá ser explícita.

---

# 146. Schema introspection input

Cuando se genere desde una base real:

```text
SchemaIntrospectionSystem
```

será la fuente.

---

# 147. Direct information_schema queries

El generator no deberá consultar manualmente:

```text
information_schema
pg_catalog
sqlite_master
```

si ya existe Schema Introspection.

---

# 148. Coverage awareness

Si introspection devuelve:

```text
PARTIAL
```

el generator deberá conocerlo.

---

# 149. Partial introspection

No deberá inferir:

```text
not observed
→ absent
```

---

# 150. Capability awareness

Al generar artefactos platform-sensitive deberá consultar:

```text
DatabaseCapabilitySystem
```

---

# 151. Example

Generar una migration con:

```text
generated column
```

requiere conocer si la plataforma soporta la característica.

---

# 152. Capability UNKNOWN

Si:

```text
GENERATED_COLUMNS = UNKNOWN
```

el generator no deberá asumir soporte.

---

# 153. Platform portability

Por defecto, los generadores deberán favorecer APIs portables.

---

# 154. Platform-specific generation

Deberá ser explícita.

Ejemplo:

```text
--platform=postgresql
```

o metadata equivalente.

---

# 155. Portable ≠ lowest common denominator siempre

El sistema podrá generar una feature avanzada cuando la aplicación declare requerirla.

Pero deberá hacerlo explícitamente.

---

# 156. Generated migration portability

Una migration puede declarar:

```text
required capabilities
```

si usa features específicas.

---

# 157. Security

El generator no deberá insertar secrets en código generado.

---

# 158. Credentials

Prohibido generar:

```php
$password = 'production-secret';
```

desde configuración real.

---

# 159. Schema comments

Información sensible encontrada en metadata/comments deberá tratarse cuidadosamente.

---

# 160. Production introspection

Generar desde producción deberá ser una operación read-only por defecto.

---

# 161. Introspection credentials

Preferir credenciales de solo lectura.

---

# 162. Data sampling

El generador no deberá leer filas reales para inferir tipos por defecto.

---

# 163. Schema inference ≠ data profiling

Son procesos diferentes.

---

# 164. Optional profiling

Si en el futuro existe:

```text
Data Profiling System
```

deberá ser explícito y sujeto a seguridad/privacidad.

---

# 165. Determinism

Con las mismas entradas:

```text
Request
Context
Template versions
Generator version
```

deberá producirse el mismo resultado lógico.

---

# 166. Formalmente

```text
G(R, C, T, V)
=
A
```

donde:

```text
R = Request
C = Context
T = Templates
V = Generator version
A = Artifacts
```

Si:

```text
R1 = R2
C1 = C2
T1 = T2
V1 = V2
```

entonces idealmente:

```text
A1 = A2
```

salvo campos explícitamente variables.

---

# 167. Timestamps

No deberán introducirse timestamps arbitrarios en todos los archivos generados.

Esto perjudica reproducibilidad.

---

# 168. Migration IDs

Son una excepción cuando la estrategia oficial los requiere.

---

# 169. Clock abstraction

La generación deberá utilizar:

```text
GenerationClock
```

cuando necesite tiempo.

---

# 170. Randomness

Si alguna recipe requiere aleatoriedad:

```text
GenerationRandomSource
```

deberá ser controlable.

---

# 171. Testing determinism

Tests podrán utilizar:

```text
FixedClock
DeterministicRandomSource
```

---

# 172. Generator Manifest

Una ejecución podrá producir internamente:

```text
GenerationManifest
```

---

# 173. Manifest content

Ejemplo:

```text
generation id
generator version
template versions
request fingerprint
artifact paths
artifact fingerprints
created
modified
skipped
conflicts
diagnostics
```

---

# 174. Manifest ≠ required project file

No necesariamente deberá guardarse dentro del repositorio.

Podrá ser resultado de ejecución.

---

# 175. Provenance

Los artefactos podrán conocer:

```text
generated from:
manual intent
schema
ORM metadata
plugin recipe
```

---

# 176. Generation Diagnostics

El sistema deberá integrarse con:

```text
308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md
```

---

# 177. Diagnostic examples

```text
DB-GEN-001
Target file already exists.

DB-GEN-002
Entity name is invalid.

DB-GEN-003
Schema inference is ambiguous.

DB-GEN-004
Required capability is unknown.

DB-GEN-005
Existing generated region was modified.

DB-GEN-006
Template is incompatible.

DB-GEN-007
Generation plan contains dependency cycle.
```

---

# 178. Diagnostics ≠ Exceptions necesariamente

Una generación puede contener:

```text
warnings
recommendations
ambiguities
```

sin abortar inmediatamente.

---

# 179. Severity

```text
INFO
WARNING
ERROR
FATAL
```

podrá utilizarse.

---

# 180. CLI integration

Documento 309 podrá exponer:

```text
make:entity
make:model
make:repository
make:migration
make:seeder
make:factory
make:fixture
make:value-object
make:database-type
make:database-driver
make:database-dialect
make:database-compiler
```

---

# 181. CLI architecture

```text
CLI
↓
GenerationRequest
↓
CodeGenerationSystem
↓
GenerationResult
↓
CLI Renderer
```

---

# 182. CLI ≠ Generator

El comando:

```text
make:entity
```

no deberá implementar generación.

---

# 183. Interactive mode

La CLI podrá construir el request mediante preguntas.

Ejemplo:

```text
Entity name:
User

Add field?:
yes

Field:
email

Type:
string
```

---

# 184. Non-interactive mode

Todo deberá poder expresarse mediante request programático para:

```text
CI
IDE
Work
plugins
tests
```

---

# 185. IDE integration

La arquitectura deberá permitir en el futuro:

```text
VS Code
TRAE
W4 Forgeon
other IDEs
```

invocar Code Generation sin simular terminal input.

---

# 186. IDE preview

Podrá mostrar:

```text
files to create
diffs
diagnostics
conflicts
```

antes de aplicar.

---

# 187. IDE ≠ privileged bypass

Las mismas policies deberán aplicarse.

---

# 188. Programmatic API

Ejemplo conceptual:

```php
$result = $generator->generate(
    new EntityGenerationRequest(
        name: new EntityName('User'),
    ),
);
```

---

# 189. Generate ≠ Write necesariamente

Se recomienda separar:

```php
$plan = $generator->plan($request);

$preview = $generator->render($plan);

$result = $writer->apply($preview);
```

o fachada equivalente.

---

# 190. Three-phase model

La arquitectura ideal será:

```text
PLAN
↓
RENDER
↓
APPLY
```

---

# 191. PLAN

No muta filesystem.

---

# 192. RENDER

Produce contenido/diffs.

No muta filesystem.

---

# 193. APPLY

Realiza las mutaciones autorizadas.

---

# 194. Benefits

Esto facilita:

```text
dry-run
IDE preview
testing
conflict detection
auditing
approval
```

---

# 195. GenerationResult

Ejemplo:

```php
final readonly class GenerationResult
{
    public function __construct(
        public GenerationStatus $status,
        public array $created,
        public array $modified,
        public array $skipped,
        public array $conflicted,
        public GenerationDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 196. Status

Propuesta:

```text
SUCCESS
PARTIAL_SUCCESS
FAILED
CONFLICT
INVALID
INCONCLUSIVE
UNKNOWN
```

---

# 197. INCONCLUSIVE

Ejemplo:

```text
schema introspection incomplete
+
required inference cannot be established
```

---

# 198. UNKNOWN

Reservado para situaciones donde no pueda determinarse el resultado de una mutación.

---

# 199. Extensions

Plugins podrán registrar:

```text
Generator
Template
Recipe
Artifact Type
Naming Rule
Validator
PostProcessor
```

---

# 200. Extension architecture

```text
Database Plugin
     ↓
CodeGenerationExtension
     ↓
Extension Registry
     ↓
Frozen Generation Environment
```

---

# 201. Extension priority

Deberá ser explícita y determinista.

---

# 202. Extension collision

Dos plugins que reclamen el mismo:

```text
GeneratorId
TemplateId
RecipeId
```

deberán producir conflicto.

---

# 203. No silent override

Regla:

```text
duplicate extension
→ bootstrap error
```

salvo mecanismo oficial de override.

---

# 204. Custom templates

Los proyectos podrán publicar templates propios.

---

# 205. Custom recipe example

Una organización podría definir:

```text
CompanyEntityRecipe
```

que genere:

```text
Entity
Repository
Factory
Test
Policy skeleton
```

si integra otros subsistemas.

---

# 206. Cross-subsystem generation

Database Code Generation podrá coordinarse con otros generators de VoltStack.

Pero no deberá absorber todos los generadores del framework.

---

# 207. Framework Code Generation

A largo plazo podría existir:

```text
VoltStack Code Generation Platform
├── Database
├── Controllers
├── Routing
├── Authentication
├── Authorization
└── UI
```

Database contribuiría sus recipes.

---

# 208. Shared infrastructure

Elementos como:

```text
PHP AST
Template Engine
Filesystem Writer
Diff Engine
Naming
```

podrían vivir en:

```text
VoltStack/Platform/CodeGeneration
```

si son reutilizables.

---

# 209. Database-specific layer

Entonces:

```text
VoltStack/Quantum/Database/CodeGeneration
```

contendría:

```text
Entity generators
Migration generators
Database extension generators
Database metadata adapters
```

---

# 210. Proposed architecture

```text
VoltStack/Platform/CodeGeneration
│
├── Artifact
├── Template
├── PHP
├── Writer
├── Diff
├── Conflict
└── Validation
        ▲
        │
VoltStack/Quantum/Database/CodeGeneration
│
├── Entity
├── Model
├── Repository
├── Migration
├── Seeder
├── Factory
├── Fixture
├── Type
├── Driver
├── Dialect
├── Compiler
├── QueryExtension
└── OrmExtension
```

---

# 211. Testing Architecture

Code Generation deberá tener:

```text
Unit Tests
Golden Tests
Snapshot Tests
Parser Tests
Integration Tests
Conflict Tests
Compatibility Tests
Extension Tests
```

---

# 212. Unit tests

Probarán:

```text
naming
planning
dependency graph
artifact models
conflict classification
```

---

# 213. Golden tests

Un input conocido deberá producir código esperado.

Ejemplo:

```text
EntityGenerationRequest(User)
```

contra:

```text
expected/User.php
```

---

# 214. Golden test caution

No deberán utilizarse snapshots gigantes sin intención.

Cambios deberán revisarse.

---

# 215. Syntax tests

Todo PHP generado deberá poder pasar:

```text
PHP parser
```

---

# 216. Runtime integration tests

Algunos artefactos generados deberán probarse realmente.

Ejemplo:

```text
generate entity
→ load class
→ compile ORM metadata
→ validate mapping
```

---

# 217. Migration generation integration

```text
generate migration
→ load migration
→ build migration plan
```

---

# 218. Driver skeleton tests

```text
generate driver skeleton
→ static validation
→ conformance suite bootstrap
```

No implica que el driver pase conformance sin implementación.

---

# 219. Cross-version testing

Templates deberán probarse contra versiones soportadas del generator/framework cuando aplique.

---

# 220. Generated code quality

El objetivo deberá ser:

> **El código generado debe parecer código que un desarrollador de VoltStack escribiría manualmente siguiendo las convenciones oficiales.**

No deberá producir boilerplate innecesario.

---

# 221. Minimal generation

Preferir:

```text
minimum correct artifact
```

sobre:

```text
every possible method pre-generated
```

---

# 222. Example entity minimalism

No generar automáticamente:

```text
getId()
setId()
getEmail()
setEmail()
toArray()
jsonSerialize()
save()
delete()
refresh()
```

si esas APIs no son necesarias.

---

# 223. Domain preservation

El generator deberá evitar convertir entidades en simples DTOs anémicos por obligación.

El desarrollador podrá añadir comportamiento de dominio.

---

# 224. Generated setters

No deberán asumirse universalmente.

---

# 225. Immutable entities/value objects

El generator deberá soportar estilos:

```text
mutable
immutable
constructor-based
```

cuando el ORM lo permita.

---

# 226. Constructor strategy

Debe alinearse con:

```text
Entity Hydration Architecture
```

para no generar constructores incompatibles con el Hydrator.

---

# 227. Identifier generation

Debe respetar:

```text
assigned
database generated
UUID
ULID
custom
```

según metadata/policy.

---

# 228. Generated ID ≠ always integer

Regla:

```text
Generated Identifier
≠
Auto Increment Integer
```

---

# 229. Type generation

Tipos deberán provenir del:

```text
Database Type System
```

No de mapas duplicados dentro del generator.

---

# 230. Example

No mantener:

```php
$generatorTypes = [
    'varchar' => 'string',
];
```

como segunda fuente independiente.

Preferir:

```text
TypeRegistry
+
Platform Mapping
```

---

# 231. Schema types

Debe mantenerse:

```text
Physical SQL Type
≠
Logical Database Type
≠
PHP Type
```

---

# 232. Temporal generation

El generator deberá distinguir:

```text
Instant
LocalDate
LocalTime
LocalDateTime
ZonedDateTime
```

cuando metadata lo permita.

---

# 233. JSON generation

Un JSON column no implica necesariamente:

```php
array
```

Puede mapearse a:

```text
array
DTO
Value Object
custom type
```

---

# 234. Enum generation

Podrá sugerir enums si metadata explícita lo indica.

No deberá inferir enum exclusivamente por pocos valores observados en datos.

---

# 235. Nullable

Debe mantenerse:

```text
DB NULL
≠
Field Not Loaded
```

aunque el generator produzca tipos nullable.

---

# 236. Partial entities

El generator no deberá generar APIs que confundan:

```text
nullable field
```

con:

```text
unloaded field
```

---

# 237. Relationship generation

Debe respetar:

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
Polymorphic
Association Entity
```

---

# 238. Owning side

Cuando pueda determinarse, deberá generarse correctamente.

---

# 239. Ambiguous ownership

Debe solicitar decisión.

---

# 240. Cascade

No deberá habilitar automáticamente:

```text
cascade remove
```

por conveniencia.

---

# 241. Orphan removal

También deberá requerir semántica explícita.

---

# 242. Lazy/eager defaults

Deberán provenir de las convenciones ORM oficiales.

No del generador aisladamente.

---

# 243. Index generation

Cuando genera migrations desde entity metadata:

```text
ORM metadata
→ schema intent
```

deberá pasar por Schema/Migration systems.

---

# 244. ORM mapping ≠ complete physical schema

No todas las características físicas deberán necesariamente derivarse del ORM.

Ejemplos:

```text
specialized indexes
partitioning
storage options
```

---

# 245. Round-trip limitation

No deberá prometerse:

```text
Database
→ Entity
→ Database
```

como transformación perfectamente reversible.

---

# 246. Information loss

Formalmente:

```text
Schema → ORM Model
```

puede perder información física.

Y:

```text
ORM Model → Schema
```

puede no expresar todas las decisiones físicas.

---

# 247. Reverse engineering report

Una generación desde schema deberá poder mostrar:

```text
exact mappings
inferred mappings
ambiguous mappings
unsupported mappings
ignored physical details
```

---

# 248. Example

```text
Table: orders

Exact
  id → Order::$id

Inferred
  user_id → ManyToOne(User)

Ambiguous
  status → string | enum

Not mapped
  PostgreSQL storage parameter fillfactor
```

---

# 249. Safe generation workflow

Recomendado:

```text
Inspect
↓
Plan
↓
Review
↓
Preview
↓
Validate
↓
Apply
↓
Test
```

---

# 250. Telemetry

Code generation podrá emitir telemetry opcional.

Ejemplo:

```text
generator type
artifact count
duration
result status
```

---

# 251. Privacy

No deberá enviar:

```text
generated source code
schema names
field names
business entities
```

a telemetry externa por defecto.

---

# 252. Audit

La generación local ordinaria no requiere necesariamente audit de seguridad.

Pero generación administrativa/remota podrá integrarlo.

---

# 253. Performance

El generator deberá ser suficientemente rápido para interacción IDE.

---

# 254. Metadata caching

Podrá reutilizar:

```text
compiled ORM metadata
schema metadata
template cache
```

si sus generations/fingerprints son válidos.

---

# 255. Cache ≠ truth

Un cache obsoleto no deberá generar código basado en metadata incorrecta sin validación de generación.

---

# 256. Incremental generation

Podrá soportarse:

```text
only affected artifacts
```

---

# 257. Incremental identity

Deberá utilizar fingerprints estructurales, no sólo timestamps de archivos.

---

# 258. Watch mode

Futuro:

```text
database:generate --watch
```

podría existir para determinados artefactos.

Pero deberá evitar loops:

```text
generated file
→ watcher
→ regeneration
→ watcher
→ ...
```

---

# 259. CI check mode

Se recomienda:

```text
database:generate --check
```

---

# 260. Check mode

No escribe.

Comprueba si:

```text
current generated artifacts
==
expected generated artifacts
```

---

# 261. CI use

Ejemplo:

```text
php voltstack database:generate --check
```

podrá fallar si los artefactos generados están desactualizados.

---

# 262. Check ≠ regenerate

Esto evita mutar el workspace en CI.

---

# 263. Generation lock

Generaciones concurrentes sobre los mismos targets podrán coordinarse mediante:

```text
GenerationLock
```

---

# 264. Lock scope

Podrá ser:

```text
project
artifact group
target path
```

---

# 265. Lock ≠ database lock

Debe mantenerse:

```text
GenerationLock
≠
Database Lock
```

---

# 266. Versioning

El sistema deberá versionar:

```text
Generator API
Template contracts
Artifact schemas
Recipe contracts
```

según sea necesario.

---

# 267. Backward compatibility

Cambios en templates no deberán romper automáticamente proyectos existentes.

---

# 268. Template migration

Podrá existir tooling futuro:

```text
database:generate:upgrade
```

para actualizar artefactos generator-owned.

---

# 269. Upgrade ≠ blind rewrite

Deberá utilizar:

```text
fingerprints
AST
diff
conflict detection
```

---

# 270. Deprecation

Generators obsoletos deberán seguir la política general de VoltStack.

---

# 271. Proposed namespace

```text
src/Quantum/Database/
└── CodeGeneration/
    ├── Contract/
    │   ├── GeneratorInterface.php
    │   ├── GenerationPlannerInterface.php
    │   ├── ArtifactGeneratorInterface.php
    │   ├── TemplateRegistryInterface.php
    │   ├── ArtifactWriterInterface.php
    │   └── GenerationValidatorInterface.php
    │
    ├── Request/
    ├── Context/
    ├── Plan/
    ├── Artifact/
    ├── Template/
    ├── Recipe/
    ├── Naming/
    ├── Validation/
    ├── Conflict/
    ├── Diff/
    ├── Writer/
    ├── Manifest/
    ├── Diagnostic/
    │
    ├── Entity/
    ├── Model/
    ├── Repository/
    ├── Migration/
    ├── Seeder/
    ├── Factory/
    ├── Fixture/
    ├── Type/
    ├── Driver/
    ├── Dialect/
    ├── Compiler/
    ├── QueryExtension/
    ├── OrmExtension/
    │
    ├── Extension/
    └── Testing/
```

---

# 272. Possible shared Platform namespace

Si se generaliza:

```text
src/Platform/
└── CodeGeneration/
    ├── Php/
    ├── Template/
    ├── Artifact/
    ├── Writer/
    ├── Diff/
    ├── Conflict/
    └── Validation/
```

Database deberá depender de esta infraestructura genérica.

Nunca al revés.

---

# 273. Dependency direction

Correcto:

```text
Quantum/Database/CodeGeneration
        ↓
Platform/CodeGeneration
```

Incorrecto:

```text
Platform/CodeGeneration
        ↓
Quantum/Database/ORM
```

---

# 274. Core invariants

## DB-CODEGEN-001

Code Generator ≠ Database Runtime.

## DB-CODEGEN-002

Generated Code ≠ Database State.

## DB-CODEGEN-003

Schema ≠ Entity Model.

## DB-CODEGEN-004

Migration Generator ≠ Migration Planner.

## DB-CODEGEN-005

Generator ≠ Filesystem Writer.

## DB-CODEGEN-006

Recipe ≠ Template.

## DB-CODEGEN-007

Preview ≠ Apply.

## DB-CODEGEN-008

Generated Driver ≠ Conformant Driver.

## DB-CODEGEN-009

Generated Migration ≠ Safe Migration.

## DB-CODEGEN-010

Generated Code ≠ Developer Intent unless explicitly confirmed/inferred.

---

# 275. Planning invariants

## DB-CODEGEN-011

Toda generación deberá poder representarse como plan.

## DB-CODEGEN-012

Planning no modificará source files.

## DB-CODEGEN-013

Rendering no modificará source files.

## DB-CODEGEN-014

Apply será la fase mutante.

## DB-CODEGEN-015

Plan podrá contener múltiples artifacts.

## DB-CODEGEN-016

Dependencies serán explícitas.

## DB-CODEGEN-017

Cycles inválidos fallarán antes de apply.

## DB-CODEGEN-018

Target paths serán conocidos antes de apply.

## DB-CODEGEN-019

Conflicts serán evaluados antes de mutar cuando sea posible.

## DB-CODEGEN-020

Generation context será explícito.

---

# 276. Filesystem invariants

## DB-CODEGEN-021

Default overwrite policy será conservadora.

## DB-CODEGEN-022

Existing developer code no será sobrescrito silenciosamente.

## DB-CODEGEN-023

File fingerprints podrán proteger modificaciones concurrentes.

## DB-CODEGEN-024

Changed target invalidará modificaciones planificadas cuando corresponda.

## DB-CODEGEN-025

Partial writes serán reportados.

## DB-CODEGEN-026

Filesystem rollback no se fingirá como transacción garantizada.

## DB-CODEGEN-027

Temporary files serán limpiados.

## DB-CODEGEN-028

Writer errors serán estructurados.

## DB-CODEGEN-029

Unknown write outcome será representable.

## DB-CODEGEN-030

Generator no escribirá fuera de targets autorizados.

---

# 277. Schema invariants

## DB-CODEGEN-031

Schema introspection utilizará Schema System.

## DB-CODEGEN-032

Not Observed ≠ Absent.

## DB-CODEGEN-033

Partial coverage será preservada.

## DB-CODEGEN-034

FK ≠ complete ORM relationship.

## DB-CODEGEN-035

Join table ≠ always ManyToMany.

## DB-CODEGEN-036

Physical Type ≠ Logical Type.

## DB-CODEGEN-037

Logical Type ≠ PHP Type.

## DB-CODEGEN-038

Schema reverse engineering podrá ser ambiguo.

## DB-CODEGEN-039

Ambiguity no será ocultada.

## DB-CODEGEN-040

Data rows no serán inspeccionadas implícitamente.

---

# 278. ORM invariants

## DB-CODEGEN-041

Entity Generator no implementará ORM.

## DB-CODEGEN-042

Model API generator utilizará el ORM oficial.

## DB-CODEGEN-043

Repository generator utilizará Repository System.

## DB-CODEGEN-044

Generated Model no creará segundo persistence engine.

## DB-CODEGEN-045

Cascade remove no se habilitará arbitrariamente.

## DB-CODEGEN-046

Orphan removal requerirá semántica explícita.

## DB-CODEGEN-047

Identifier generation será metadata-aware.

## DB-CODEGEN-048

Generated ID ≠ integer necesariamente.

## DB-CODEGEN-049

Nullable ≠ unloaded.

## DB-CODEGEN-050

Constructor strategy respetará Hydration System.

---

# 279. Extension invariants

## DB-CODEGEN-051

Extensions tendrán identidades estables.

## DB-CODEGEN-052

Registries serán frozen.

## DB-CODEGEN-053

Duplicate IDs no serán resueltos silenciosamente.

## DB-CODEGEN-054

Plugins podrán aportar recipes.

## DB-CODEGEN-055

Plugins podrán aportar templates.

## DB-CODEGEN-056

Plugins no podrán saltar conflict detection.

## DB-CODEGEN-057

Plugins no podrán saltar validation.

## DB-CODEGEN-058

Plugins no podrán escribir fuera de su plan autorizado.

## DB-CODEGEN-059

Extension order será determinista.

## DB-CODEGEN-060

Core generators no serán reemplazados silenciosamente.

---

# 280. Security invariants

## DB-CODEGEN-061

Secrets no serán generados dentro de source code.

## DB-CODEGEN-062

Production credentials no serán copiadas.

## DB-CODEGEN-063

Production introspection será read-only por defecto.

## DB-CODEGEN-064

Sensitive metadata será protegida.

## DB-CODEGEN-065

Generated diagnostics serán redacted.

## DB-CODEGEN-066

Paths serán validados.

## DB-CODEGEN-067

Path traversal será rechazado.

## DB-CODEGEN-068

Template execution no implicará arbitrary code execution.

## DB-CODEGEN-069

Untrusted templates deberán tratarse como código potencialmente peligroso.

## DB-CODEGEN-070

Remote templates no se descargarán/ejecutarán implícitamente.

---

# 281. Reproducibility invariants

## DB-CODEGEN-071

Generation inputs serán identificables.

## DB-CODEGEN-072

Generator version será identificable.

## DB-CODEGEN-073

Template version será identificable.

## DB-CODEGEN-074

Randomness será controlable.

## DB-CODEGEN-075

Clock será controlable.

## DB-CODEGEN-076

Same logical input deberá producir same logical output.

## DB-CODEGEN-077

Environment-specific differences serán explícitas.

## DB-CODEGEN-078

Platform-specific generation será explícita.

## DB-CODEGEN-079

Generation manifest podrá reproducir provenance.

## DB-CODEGEN-080

CI check no modificará files.

---

# 282. Testing invariants

## DB-CODEGEN-081

Generated PHP será parseable.

## DB-CODEGEN-082

Golden outputs serán testeables.

## DB-CODEGEN-083

Conflict behavior tendrá tests.

## DB-CODEGEN-084

Overwrite policies tendrán tests.

## DB-CODEGEN-085

Dry-run tendrá zero writes.

## DB-CODEGEN-086

Extension collisions tendrán tests.

## DB-CODEGEN-087

Schema ambiguity tendrá tests.

## DB-CODEGEN-088

Capability UNKNOWN tendrá tests.

## DB-CODEGEN-089

Generated ORM metadata tendrá integration tests.

## DB-CODEGEN-090

Driver skeleton no se considerará conformance evidence.

---

# 283. Developer Experience invariants

## DB-CODEGEN-091

Generación simple deberá requerir mínima configuración.

## DB-CODEGEN-092

Defaults deberán ser seguros.

## DB-CODEGEN-093

Ambiguities deberán explicarse.

## DB-CODEGEN-094

Preview deberá ser legible.

## DB-CODEGEN-095

Diff deberá ser legible.

## DB-CODEGEN-096

Errors deberán incluir acciones correctivas cuando sea posible.

## DB-CODEGEN-097

CLI e IDE compartirán Generation API.

## DB-CODEGEN-098

Generated code será IDE-friendly.

## DB-CODEGEN-099

Generated code será strongly typed.

## DB-CODEGEN-100

Boilerplate innecesario deberá minimizarse.

---

# 284. Advanced invariants

## DB-CODEGEN-101

Generated source ownership será explícito cuando importe.

## DB-CODEGEN-102

Mixed ownership requerirá estrategia segura.

## DB-CODEGEN-103

Regex no será mecanismo principal de refactoring estructural.

## DB-CODEGEN-104

Structured PHP modifications utilizarán parser/AST cuando corresponda.

## DB-CODEGEN-105

Formatting ≠ semantic generation.

## DB-CODEGEN-106

Formatter failure será distinguible.

## DB-CODEGEN-107

Code generation cache ≠ metadata truth.

## DB-CODEGEN-108

Incremental generation usará fingerprints confiables.

## DB-CODEGEN-109

Concurrent generation no corromperá targets silenciosamente.

## DB-CODEGEN-110

GenerationLock ≠ DatabaseLock.

---

# 285. Final invariants

## DB-CODEGEN-111

Correctness > convenience.

## DB-CODEGEN-112

Explicit intent > destructive inference.

## DB-CODEGEN-113

Structured models > arbitrary string concatenation.

## DB-CODEGEN-114

Portable API > vendor SQL por defecto.

## DB-CODEGEN-115

Existing architecture > duplicated generator logic.

## DB-CODEGEN-116

Generated artifacts deberán utilizar APIs públicas/estables.

## DB-CODEGEN-117

Generator no dependerá de mutable runtime ORM state.

## DB-CODEGEN-118

Generator no dependerá de active transaction state.

## DB-CODEGEN-119

Generator no mantendrá una segunda Type Registry.

## DB-CODEGEN-120

Generator no mantendrá una segunda Capability System.

---

# 286. Anti-patterns

## 286.1 String concatenation everywhere

```php
$code = "<?php class " . $name . " { ... }";
```

No como infraestructura principal.

---

## 286.2 Sobrescribir sin comprobar

```php
file_put_contents($path, $code);
```

sin:

```text
existence
fingerprint
policy
conflict detection
```

---

# 287. Inferir dominio desde tablas

```text
table
→ perfect entity model
```

Incorrecto.

---

# 288. Asumir FK = relationship completa

Una FK sólo aporta parte de la información.

---

# 289. Generar cascade delete por default

Potencialmente destructivo.

---

# 290. Consultar information_schema directamente

Duplicaría Schema Introspection.

---

# 291. Duplicar Type Registry

El generator no tendrá su propio sistema paralelo de tipos.

---

# 292. Duplicar Capability Registry

También prohibido.

---

# 293. `--force` como destrucción total

```text
--force
→ overwrite every source file
```

Anti-pattern.

---

# 294. Regex para modificar PHP complejo

Frágil ante:

```text
attributes
comments
formatting
traits
anonymous classes
multiple imports
```

---

# 295. Templates remotos implícitos

No descargar y ejecutar templates desconocidos automáticamente.

---

# 296. Generar secrets

Nunca.

---

# 297. Generar métodos innecesarios

Evitar clases con cientos de getters/setters sin intención.

---

# 298. Inferir enums desde producción

Observar:

```text
active
inactive
```

en filas no demuestra un enum completo.

---

# 299. Reverse engineering destructivo

Nunca modificar la DB porque se está generando código.

---

# 300. Generator conectado permanentemente a producción

No será requisito arquitectónico.

---

# 301. Arquitectura final resumida

```text
                     Developer
                         │
                         ▼
                CLI / IDE / API
                         │
                         ▼
                 Generation Request
                         │
                         ▼
                Request Normalizer
                         │
                         ▼
                Generation Context
                         │
         ┌───────────────┼────────────────┐
         ▼               ▼                ▼
   Schema Metadata   ORM Metadata     Conventions
         │               │                │
         └───────────────┼────────────────┘
                         ▼
                Capability Context
                         │
                         ▼
                Generation Planner
                         │
                         ▼
                  Generation Plan
                         │
             ┌───────────┼────────────┐
             ▼           ▼            ▼
          Entity      Migration    Extension
         Artifacts    Artifacts    Artifacts
             │           │            │
             └───────────┼────────────┘
                         ▼
                  Artifact Model
                         │
                         ▼
              Template / PHP Model
                         │
                         ▼
                   Source Renderer
                         │
                         ▼
                    Validation
                         │
                         ▼
                Conflict Detection
                         │
               ┌─────────┴─────────┐
               ▼                   ▼
            Preview              Apply
               │                   │
               ▼                   ▼
             Diff          Filesystem Writer
                                   │
                                   ▼
                          Generation Result
                                   │
                  ┌────────────────┼───────────────┐
                  ▼                ▼               ▼
              Created          Modified        Conflicts
                                   │
                                   ▼
                              Diagnostics
```

---

# 302. Flujo recomendado para generación desde schema

```text
Real Database
      │
      ▼
Schema Introspection
      │
      ▼
Normalized Schema Model
      │
      ▼
Coverage Analysis
      │
      ▼
ORM Mapping Inference
      │
      ├── Exact
      ├── Inferred
      ├── Ambiguous
      └── Unsupported
      │
      ▼
Developer Decisions
      │
      ▼
Generation Request
      │
      ▼
Generation Plan
      │
      ▼
Preview
      │
      ▼
Validation
      │
      ▼
Apply
```

---

# 303. Flujo recomendado para migration generation

```text
Current Schema
      │
      ▼
Schema Diff
      │
      ▼
Migration Planner
      │
      ▼
Safety Analysis
      │
      ▼
Migration Generation Model
      │
      ▼
Migration Generator
      │
      ▼
Generated Migration Source
      │
      ▼
Developer Review
```

No:

```text
Schema Diff
→ Execute Database
```

---

# 304. Flujo recomendado para driver skeleton

```text
Driver Generation Request
        │
        ▼
Custom Driver Recipe
        │
        ├── Driver
        ├── Connection
        ├── Statement
        ├── Result
        ├── Error Mapper
        ├── Capability Provider
        ├── Plugin Manifest
        └── Conformance Tests
        │
        ▼
Generation Plan
        │
        ▼
Preview
        │
        ▼
Apply
        │
        ▼
Developer Implementation
        │
        ▼
Driver Conformance Suite
```

---

# 305. Regla arquitectónica definitiva

> **VoltStack Database Code Generation deberá reducir boilerplate sin reducir explicitud arquitectónica. Todo código generado deberá ser producto de una intención estructurada, un contexto conocido, reglas versionadas y un plan inspeccionable; ninguna inferencia ambigua deberá convertirse silenciosamente en una decisión irreversible sobre el modelo de dominio, el esquema o la persistencia.**

Formalmente:

```text
Generation Validity
=
Valid Request
∧
Valid Context
∧
Valid Plan
∧
Valid Artifacts
∧
No Blocking Conflict
```

y:

```text
Safe Apply
=
Generation Validity
∧
Write Policy Allows
∧
Target Preconditions Hold
```

---

# 306. Resultado arquitectónico

Con este sistema, VoltStack podrá proporcionar una experiencia como:

```text
$ php voltstack make:entity User
```

para casos simples, mientras mantiene detrás:

```text
Typed Generation Request
        ↓
Generation Context
        ↓
Naming
        ↓
Metadata
        ↓
Planner
        ↓
Artifact Model
        ↓
Validation
        ↓
Conflict Detection
        ↓
Safe Writer
```

La simplicidad de la API pública no requerirá sacrificar la arquitectura interna.

Esto preserva uno de los objetivos centrales de VoltStack:

> **experiencia de desarrollo simple en la superficie, arquitectura rigurosa y explícita internamente.**

---

# 307. Cierre del Bloque 31

Con este documento se completa el bloque:

```text
Block 31 — Developer Experience / Public API

302_DATABASE_PUBLIC_API_SYSTEM.md
303_DATABASE_FACADE_SYSTEM.md
304_DATABASE_HELPER_SYSTEM.md
305_DATABASE_MODEL_DEVELOPER_EXPERIENCE.md
306_DATABASE_QUERY_DEVELOPER_EXPERIENCE.md
307_DATABASE_SCHEMA_DEVELOPER_EXPERIENCE.md
308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md
309_DATABASE_CLI_SYSTEM.md
310_DATABASE_CODE_GENERATION_SYSTEM.md
```

Este bloque define cómo el desarrollador interactuará con la enorme arquitectura interna de Database mediante una superficie:

```text
simple
typed
discoverable
IDE-friendly
safe
diagnosticable
automatable
extensible
```

sin exponer innecesariamente la complejidad interna.

---

# 308. Siguiente bloque

A partir del siguiente documento comienza:

```text
Block 32 — VoltStack Integration
```

El objetivo será conectar formalmente Database con el resto del framework sin introducir dependencias circulares ni acoplamientos implícitos.

---

# 309. Siguiente documento

```text
311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md
```

Definirá la arquitectura maestra de integración entre:

```text
VoltStack Framework
        │
        ├── Platform
        ├── Container
        ├── Config
        ├── Cache
        ├── Events
        ├── Telemetry
        ├── Validation
        ├── Authentication
        ├── Authorization
        ├── Jobs / Queues
        ├── HTTP Runtime
        └── Quantum/Database
```

estableciendo especialmente:

```text
Database Core
≠
Framework Glue
```

y definiendo qué dependencias pueden ser directas, cuáles deberán utilizar contratos/adapters y cuáles deberán permanecer completamente opcionales.