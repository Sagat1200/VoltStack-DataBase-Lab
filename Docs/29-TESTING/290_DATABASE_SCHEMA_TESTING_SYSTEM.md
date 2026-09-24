# 290_DATABASE_SCHEMA_TESTING_SYSTEM.md

# VoltStack Quantum Database
## Database Schema Testing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 290 — Database Schema Testing System  
**Bloque:** 29 — Testing  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `289_DATABASE_QUERY_ASSERTION_SYSTEM.md`  
**Siguiente documento:** `291_DATABASE_ORM_TESTING_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database Schema Testing System** de VoltStack.

Su responsabilidad es establecer la arquitectura oficial para probar:

- Schema Model;
- Schema AST;
- Schema Builder;
- Table Definitions;
- Column Definitions;
- Indexes;
- Foreign Keys;
- Constraints;
- Schema Introspection;
- Schema Metadata;
- Schema Diff;
- Schema Compiler;
- Schema Platform Compatibility;
- integración Schema ↔ Migration;
- capacidades de plataforma;
- seguridad de cambios estructurales;
- portabilidad entre DBMS.

El sistema deberá poder responder preguntas diferentes como:

```text
¿La definición construida es correcta?

¿El Schema Model esperado contiene la tabla?

¿El Compiler produjo la operación correcta?

¿El DBMS creó realmente la estructura?

¿La introspección observó correctamente esa estructura?

¿El Diff detecta el cambio?

¿La migración resultante es segura?

¿La estructura es portable?

¿La feature está realmente soportada?
```

Estas preguntas no son equivalentes.

Regla central:

> **Una prueba de Schema deberá distinguir entre definición deseada, representación estructural normalizada, transformación planificada y estructura realmente observada en el DBMS; que una definición o compilación sea correcta no demuestra por sí mismo que el esquema físico real tenga esa estructura.**

Formalmente:

```text
SchemaDefinition
≠
SchemaModel
≠
SchemaAST
≠
SchemaDiff
≠
MigrationPlan
≠
CompiledDDL
≠
ObservedDatabaseSchema
```

---

# 2. Modelo conceptual

La arquitectura de Schema ya definida por VoltStack puede representarse como:

```text
Developer Definition
        │
        ▼
   Schema Builder
        │
        ▼
 Schema Definition
        │
        ▼
   Schema Model
        │
        ├───────────────┐
        │               │
        ▼               ▼
    Schema Diff      Schema AST
        │               │
        ▼               ▼
 Migration Planner   Compiler
        │               │
        └───────┬───────┘
                ▼
          Execution Layer
                │
                ▼
              DBMS
                │
                ▼
        Schema Introspector
                │
                ▼
      Observed Schema Model
```

Testing deberá poder insertar assertions en cada frontera.

---

# 3. Schema Testing ≠ Migration Testing

Schema Testing comprueba principalmente:

```text
structure
representation
comparison
compilation
introspection
compatibility
```

Migration Testing comprueba:

```text
evolution
ordering
execution
rollback
history
batching
safety
```

Existe integración entre ambos, pero no son equivalentes.

---

# 4. Schema Testing ≠ Database State Testing genérico

Schema Testing se concentra en estructura.

No en datos de negocio.

Por ejemplo:

```text
users table exists
```

es Schema Testing.

Mientras:

```text
users contains id=42
```

es Data State Testing.

---

# 5. Schema Testing ≠ SQL String Testing

El SQL DDL es sólo una de las representaciones posibles.

Preferir:

```text
Schema Model
Schema AST
Schema Diff
```

cuando la propiedad no sea específicamente SQL.

---

# 6. Capas de prueba

VoltStack distinguirá:

```text
Schema Testing
├── Definition Tests
├── Model Tests
├── AST Tests
├── Builder Tests
├── Diff Tests
├── Compiler Tests
├── Introspection Tests
├── Execution Tests
├── Compatibility Tests
├── Capability Tests
├── Migration Integration Tests
├── Safety Tests
└── Conformance Tests
```

---

# 7. Pirámide de evidencia

```text
                     Real DBMS
                        ▲
                        │
                Conformance Tests
                        ▲
                        │
                Integration Tests
                        ▲
                        │
                 Compiler Tests
                        ▲
                        │
                 Schema Tests
                        ▲
                        │
                    Unit Tests
```

Las capas superiores proporcionan evidencia más cercana al comportamiento real del DBMS.

---

# 8. Schema Definition Tests

Estas pruebas comprueban que una definición declarativa produce la representación esperada.

Ejemplo:

```php
$schema->create('users', function (TableBlueprint $table) {
    $table->id();
    $table->string('email', 255);
    $table->unique('email');
});
```

podrá comprobarse contra un modelo estructural.

---

# 9. No SQL required

Una prueba del Schema Builder no necesita comprobar:

```sql
CREATE TABLE users ...
```

si la propiedad evaluada es la construcción del modelo.

---

# 10. Schema Model Assertions

API conceptual:

```php
$this->assertSchemaModel($schema)
    ->hasTable('users')
    ->table('users', function ($table) {
        $table
            ->hasColumn('id')
            ->hasColumn('email')
            ->hasUniqueConstraint(['email']);
    });
```

---

# 11. Schema Model ≠ DBMS Schema

Esta assertion demuestra:

```text
VoltStack model contains users
```

no:

```text
PostgreSQL physically contains users
```

---

# 12. Real Schema Assertions

Para comprobar el DBMS real deberá utilizarse:

```text
DBMS
 ↓
Schema Introspector
 ↓
Observed Schema Model
 ↓
Assertion
```

---

# 13. Schema Assertion Architecture

```text
Schema Source
     │
     ▼
Schema Observation
     │
     ▼
Normalized Schema Model
     │
     ▼
Schema Matcher
     │
     ▼
Schema Expectation
     │
     ▼
Assertion Result
```

---

# 14. Schema Sources

Una assertion podrá operar sobre:

```text
DEFINITION
MODEL
AST
INTROSPECTION
DIFF
COMPILED_DDL
REAL_DATABASE
```

---

# 15. Explicit source

La API avanzada deberá permitir declarar la fuente.

Ejemplo:

```php
SchemaAssertion::from(
    SchemaAssertionSource::INTROSPECTION,
    $observedSchema
);
```

---

# 16. SchemaExpectation

Objeto declarativo:

```php
final class SchemaExpectation
{
    public function hasTable(string $name): self;

    public function lacksTable(string $name): self;

    public function table(
        string $name,
        callable $expectation
    ): self;
}
```

---

# 17. Expectation ≠ Schema Builder

El `SchemaExpectation` no será una segunda API de definición de schemas.

Sólo describe propiedades esperadas.

---

# 18. Partial vs Exact Matching

Modos:

```text
PARTIAL
EXACT
SEMANTIC
```

---

# 19. PARTIAL

Comprueba sólo lo declarado.

---

# 20. EXACT

Comprueba que no existan elementos adicionales relevantes.

---

# 21. SEMANTIC

Permite diferencias físicas que representan la misma semántica normalizada.

---

# 22. Table Assertions

El sistema deberá soportar:

```php
$this->assertTableExists('users');

$this->assertTableDoesNotExist('legacy_users');
```

---

# 23. Table identity

La identidad deberá considerar cuando corresponda:

```text
catalog
database
schema
table
tenant
```

---

# 24. Qualified Table Name

Preferir:

```text
QualifiedTableName
```

sobre strings concatenados.

---

# 25. Case sensitivity

No deberá asumirse una regla universal.

Depende de:

```text
platform
identifier rules
quoting
filesystem behavior
```

según DBMS.

---

# 26. Table Type

Podrá distinguir:

```text
BASE_TABLE
VIEW
MATERIALIZED_VIEW
TEMPORARY
SYSTEM
```

cuando la plataforma lo soporte.

---

# 27. Column Assertions

Ejemplos:

```php
$this->assertColumnExists('users', 'email');

$this->assertColumnType(
    'users',
    'email',
    DatabaseType::STRING
);
```

---

# 28. Column properties

Podrán comprobarse:

```text
name
logical type
physical type
length
precision
scale
nullable
default
generated
identity
collation
charset
comment
```

según disponibilidad.

---

# 29. Logical Type vs Physical Type

Debe mantenerse:

```text
Logical Type
≠
Physical SQL Type
```

Ejemplo:

```text
STRING
```

puede compilar a representaciones diferentes.

---

# 30. Type Assertion Modes

Podrán existir:

```text
LOGICAL
PHYSICAL
SEMANTIC
```

---

# 31. Logical type test

Adecuado para portabilidad.

---

# 32. Physical type test

Adecuado para compiler/platform-specific tests.

---

# 33. Semantic type test

Puede aceptar varias representaciones físicas equivalentes.

---

# 34. Nullability Assertions

Ejemplo:

```php
$this->assertColumnNullable(
    'users',
    'deleted_at'
);
```

---

# 35. Default Value Assertions

Deberán distinguir:

```text
literal default
expression default
generated default
no default
unknown default
```

---

# 36. NULL default ambiguity

No asumir:

```text
DEFAULT NULL
=
NO DEFAULT
```

en todas las representaciones.

---

# 37. Generated Column Assertions

Podrán comprobar:

```text
generation expression
stored/virtual mode
capability
```

---

# 38. Identity / Auto Increment

El modelo deberá normalizar conceptos como:

```text
AUTO_INCREMENT
IDENTITY
SEQUENCE
ROWID
```

sin afirmar que sean idénticos internamente.

---

# 39. Index Assertions

API conceptual:

```php
$this->assertIndexExists(
    'users',
    ['email']
);
```

---

# 40. Index properties

Podrán incluir:

```text
columns
expressions
order
unique
type
predicate
included columns
method
visibility
```

según capabilities.

---

# 41. Index ≠ Constraint

Regla fundamental:

```text
UniqueIndex
≠
UniqueConstraint
```

aunque un DBMS pueda implementar uno mediante el otro.

---

# 42. Unique Constraint Assertion

Preferir:

```php
$this->assertUniqueConstraint(
    'users',
    ['email']
);
```

si la propiedad semántica es uniqueness.

---

# 43. Index implementation detail

Una assertion semántica de uniqueness no deberá exigir automáticamente un índice concreto.

---

# 44. Primary Key Assertions

Ejemplo:

```php
$this->assertPrimaryKey(
    'users',
    ['id']
);
```

---

# 45. Composite PK

Deberá preservar orden cuando éste sea semánticamente relevante.

---

# 46. Foreign Key Assertions

Ejemplo:

```php
$this->assertForeignKey(
    table: 'orders',
    columns: ['user_id'],
    referencedTable: 'users',
    referencedColumns: ['id'],
);
```

---

# 47. FK properties

Podrán comprobar:

```text
source columns
target table
target columns
on delete
on update
deferrability
initial mode
match type
```

según plataforma.

---

# 48. FK name

El nombre físico no siempre deberá formar parte de la assertion.

---

# 49. Generated names

Si VoltStack genera nombres deterministas, podrán probarse en Compiler Tests.

---

# 50. Constraint Assertions

Tipos:

```text
PRIMARY_KEY
UNIQUE
FOREIGN_KEY
CHECK
NOT_NULL
EXCLUSION
CUSTOM
```

según soporte.

---

# 51. CHECK Assertions

Podrán operar sobre:

```text
normalized expression
```

cuando sea posible.

---

# 52. Expression normalization caution

No deberá pretender resolver equivalencia lógica arbitraria entre expresiones SQL.

---

# 53. Schema AST Tests

El Schema AST representa transformación estructural.

Ejemplo:

```text
AddTable
AddColumn
DropColumn
RenameColumn
AlterColumn
AddIndex
DropConstraint
```

---

# 54. AST Assertion

Ejemplo:

```php
$this->assertSchemaAstContains(
    SchemaOperationExpectation::addColumn(
        table: 'users',
        column: 'timezone'
    )
);
```

---

# 55. Schema AST ≠ Migration

El AST expresa transformación estructural.

Migration agrega:

```text
ordering
execution
history
safety
operational policy
```

---

# 56. Schema Builder Tests

Deberán comprobar:

```text
DSL input
→ typed definition
→ normalized model/AST
```

---

# 57. Builder determinism

La misma definición deberá producir la misma representación normalizada bajo el mismo contexto.

---

# 58. Builder validation

Deberán probarse errores como:

```text
duplicate column
invalid identifier
invalid type options
invalid foreign key
conflicting constraints
```

---

# 59. Schema Diff Tests

El Diff es uno de los sistemas críticos.

Propiedad base:

```text
Diff(Current, Target)
```

---

# 60. Identity property

```text
Diff(A, A) = ∅
```

---

# 61. Directionality

Generalmente:

```text
Diff(A, B)
≠
Diff(B, A)
```

---

# 62. Diff Assertion

Ejemplo:

```php
$this->assertSchemaDiff($diff)
    ->addsColumn('users', 'timezone')
    ->dropsNoTables();
```

---

# 63. Difference ≠ Migration

La assertion de Diff no deberá comprobar automáticamente orden de ejecución.

---

# 64. Diff categories

```text
ADD
REMOVE
ALTER
RENAME
UNKNOWN
```

---

# 65. Rename detection

Un rename inferido deberá indicar:

```text
confidence
evidence
policy
```

---

# 66. Rename caution

No asumir:

```text
drop column A
+
add column B
=
rename
```

sin evidencia.

---

# 67. NotObserved ≠ Absent

Regla crítica:

```text
NotObserved
≠
Absent
```

Si introspection coverage es parcial, el Diff no deberá inventar una eliminación.

---

# 68. Schema Coverage

Valores:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 69. Coverage Assertions

Ejemplo:

```php
$this->assertSchemaCoverage(
    SchemaCoverage::COMPLETE
);
```

---

# 70. Coverage by feature

Podrá existir granularidad:

```text
tables = COMPLETE
columns = COMPLETE
indexes = PARTIAL
check_constraints = UNKNOWN
```

---

# 71. Schema Introspection Tests

Dos niveles:

```text
Unit
Integration
```

---

# 72. Unit Introspection Tests

Podrán usar:

```text
FakeDriverMetadata
SyntheticNativeMetadata
```

para comprobar normalización.

---

# 73. Integration Introspection Tests

Deberán utilizar DBMS real.

Flujo:

```text
Create Schema
     ↓
Real DBMS
     ↓
Introspect
     ↓
Normalize
     ↓
Assert
```

---

# 74. Round-trip Introspection Test

Patrón fundamental:

```text
Definition
   ↓
Compile
   ↓
Execute
   ↓
DBMS
   ↓
Introspect
   ↓
Normalized Model
   ↓
Compare
```

---

# 75. Round-trip semantic comparison

No necesariamente:

```text
OriginalModel == IntrospectedModel
```

byte por byte.

Preferir:

```text
SemanticNormalize(Original)
=
SemanticNormalize(Introspected)
```

dentro de capabilities observables.

---

# 76. Platform normalization

Ejemplo:

```text
VARCHAR(255)
```

puede observarse con metadata diferente según DBMS.

El normalizador deberá preservar semántica relevante.

---

# 77. Introspection limitations

Una plataforma puede no exponer determinada metadata.

Esto deberá producir:

```text
UNKNOWN
```

o:

```text
PARTIAL
```

no un valor inventado.

---

# 78. Schema Metadata Assertions

Podrán comprobar:

```text
catalog
schema
table metadata
column metadata
constraint metadata
index metadata
```

---

# 79. Metadata source

La assertion deberá poder identificar:

```text
STATIC
INTROSPECTED
CACHED
SYNTHETIC
```

cuando sea relevante.

---

# 80. Metadata Cache Tests

Podrán comprobar:

```text
cache hit
cache invalidation
generation
staleness policy
```

sin confundir cache con DB truth.

---

# 81. Cached metadata ≠ current database state

Regla:

```text
MetadataCache
≠
DatabaseTruth
```

---

# 82. Schema Compiler Tests

El Compiler transforma operaciones válidas en comandos específicos de plataforma.

```text
Schema AST
    ↓
Compiler
    ↓
Compiled Schema Command
```

---

# 83. Compiler Assertion

Ejemplo:

```php
$this->assertCompiledSchemaCommand(
    $command,
    expectedSql: 'ALTER TABLE ...'
);
```

---

# 84. Compiler unit tests

Deberán probar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

por separado.

---

# 85. MySQL ≠ MariaDB

No deberán compartir ciegamente todos los expected SQL snapshots.

---

# 86. Compiler success ≠ DB acceptance

Regla:

```text
CompiledSuccessfully
≠
AcceptedByDBMS
```

---

# 87. Real execution tests

Por ello deberá existir:

```text
Compiler Test
+
Integration Execution Test
```

para features relevantes.

---

# 88. DDL Execution Assertion

Podrá comprobarse mediante introspection posterior.

Preferir:

```text
execute DDL
→ introspect
→ assert schema
```

sobre confiar únicamente en:

```text
execute returned success
```

---

# 89. Statement success ≠ desired schema state

El DBMS puede:

```text
normalize
ignore
reinterpret
limit
```

ciertas opciones.

---

# 90. Platform Compatibility Tests

El sistema definido en:

```text
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md
```

deberá tener suites específicas.

---

# 91. Compatibility statuses

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

según el modelo de capacidades aplicable.

---

# 92. Compatibility Assertion

Ejemplo:

```php
$this->assertSchemaFeatureCompatible(
    feature: $feature,
    expected: CompatibilityStatus::SUPPORTED
);
```

---

# 93. Compatibility ≠ Compiler

Que el Compiler sepa generar una representación no implica que la semántica requerida sea compatible.

---

# 94. Capability-aware Schema Testing

Toda feature relevante deberá probarse contra:

```text
CapabilityRequirement
```

en vez de sólo:

```text
vendor name
```

---

# 95. Example

Para:

```text
generated columns
```

la prueba deberá preguntar por la capability correspondiente.

---

# 96. Capability Snapshot

Los Unit Tests podrán utilizar:

```text
SyntheticCapabilitySnapshot
```

---

# 97. Real Capability Discovery

Integration Tests deberán validar contra:

```text
actual DBMS
actual driver
actual configuration
```

---

# 98. Synthetic capability ≠ real support

Se conserva:

```text
Synthetic Capability
≠
Discovered Capability
```

---

# 99. Schema Feature Matrix

La suite podrá construir matrices como:

| Feature | MySQL | MariaDB | PostgreSQL | SQLite |
|---|---|---|---|---|
| Generated column | capability-driven | capability-driven | capability-driven | capability-driven |
| Partial index | capability-driven | capability-driven | capability-driven | capability-driven |
| Deferrable FK | capability-driven | capability-driven | capability-driven | capability-driven |

Los valores reales deberán provenir de capability evidence, no quedar codificados únicamente en esta tabla.

---

# 100. Schema Portability Tests

Una definición portable podrá ejecutarse contra múltiples plataformas.

```text
Portable Schema
      │
 ┌────┼────┬─────┐
 ▼    ▼    ▼     ▼
MySQL MariaDB PG SQLite
```

---

# 101. Portable subset

VoltStack podrá definir perfiles como:

```text
PORTABLE_CORE
MYSQL_FAMILY_EXTENDED
POSTGRESQL_EXTENDED
SQLITE_COMPATIBLE
CUSTOM
```

---

# 102. Portability ≠ lowest common denominator always

El sistema podrá permitir capabilities avanzadas con fallback o rechazo explícito.

---

# 103. Portability Assertion

Podrá comprobar:

```text
all required capabilities satisfied
```

para un profile.

---

# 104. Platform-specific Schema

Será válido cuando se declare explícitamente.

---

# 105. No silent degradation

Si una feature no puede representarse con semántica equivalente:

```text
reject
```

en vez de degradarla silenciosamente.

---

# 106. Migration Integration

Schema Testing deberá integrarse con Migration Testing.

Flujo:

```text
Current Schema
      ↓
Target Schema
      ↓
Schema Diff
      ↓
Migration Planner
      ↓
Migration Operations
      ↓
Execution
      ↓
Introspection
      ↓
Target Verification
```

---

# 107. Migration Result Assertion

La verificación fuerte será:

```text
ObservedSchemaAfterMigration
≈
ExpectedTargetSchema
```

---

# 108. Migration repository alone is insufficient

Que Migration Repository marque:

```text
APPLIED
```

no demuestra por sí mismo que la estructura final sea correcta.

---

# 109. Forward Migration Tests

Deberán probar:

```text
A → B
```

---

# 110. Rollback Tests

Cuando rollback sea soportado:

```text
A → B → A'
```

y deberá evaluarse:

```text
SemanticEquivalent(A, A')
```

---

# 111. Rollback ≠ exact physical restoration

Algunos DBMS pueden recrear objetos con:

```text
different internal ids
generated names
storage metadata
```

---

# 112. Irreversible migrations

Deberán identificarse explícitamente.

---

# 113. Destructive Change Assertions

Cambios destructivos:

```text
DROP TABLE
DROP COLUMN
type narrowing
NOT NULL introduction
constraint tightening
```

deberán poder detectarse.

---

# 114. Example

```php
$this->assertSchemaDiffContainsNoDestructiveChanges();
```

---

# 115. Destructive ≠ unsafe always

La clasificación destructiva describe impacto potencial.

La seguridad operacional depende de:

```text
data
traffic
locks
deployment state
policy
```

---

# 116. Schema Diff ≠ Migration Safety

Regla:

```text
SchemaDifference
≠
MigrationSafetyDecision
```

---

# 117. Migration Safety Assertions

Podrán consumir:

```text
MigrationSafetyResult
```

del sistema correspondiente.

---

# 118. Safety statuses

Podrán incluir:

```text
SAFE
CONDITIONALLY_SAFE
UNSAFE
UNKNOWN
```

según modelo definido.

---

# 119. UNKNOWN ≠ SAFE

Regla crítica.

---

# 120. Zero-Downtime Schema Testing

El sistema deberá permitir probar patrones como:

```text
expand
migrate data
dual compatibility
switch
contract
```

---

# 121. ZDT ≠ DDL-only

Una migración zero-downtime involucra:

```text
application versions
schema versions
data transition
traffic
deployment order
```

---

# 122. Compatibility Window

Podrá definirse:

```text
App V1 ↔ Schema S1
App V1 ↔ Schema S2
App V2 ↔ Schema S2
```

---

# 123. ZDT assertion

Podrá comprobar que durante una fase:

```text
old app contract
+
new app contract
```

sean compatibles con el schema intermedio.

---

# 124. Expand/Contract Testing

Ejemplo:

```text
S1
 ↓ expand
S2-compatible
 ↓ deploy
S2-transition
 ↓ contract
S3
```

---

# 125. Schema Snapshot Testing

VoltStack podrá serializar Schema Models normalizados.

---

# 126. Snapshot contents

Podrán incluir:

```text
tables
columns
logical types
constraints
indexes
relationships at schema level
```

---

# 127. Snapshot exclusions

Evitar por defecto:

```text
internal object ids
volatile statistics
physical page information
timestamps
server-generated ephemeral metadata
```

---

# 128. Snapshot version

Todo formato deberá tener:

```text
SchemaSnapshotVersion
```

---

# 129. Snapshot generation

Ejemplo:

```text
schema.snapshot.json
```

podrá utilizarse para regression testing.

---

# 130. Snapshot ≠ Migration

Un snapshot describe estado.

Una migration describe transición.

---

# 131. Snapshot ≠ Backup

Tampoco contiene necesariamente datos.

---

# 132. Schema Drift Testing

VoltStack podrá comparar:

```text
Expected Schema
vs
Observed Schema
```

para detectar drift.

---

# 133. Drift categories

```text
MISSING
EXTRA
ALTERED
UNKNOWN
```

---

# 134. Drift Assertion

Ejemplo:

```php
$this->assertNoSchemaDrift();
```

---

# 135. Drift and partial introspection

Si coverage no es completo:

```text
NoSchemaDrift
```

no deberá afirmarse con certeza absoluta para dimensiones no observadas.

---

# 136. Result

Podrá ser:

```text
NO_DRIFT_OBSERVED
DRIFT_DETECTED
INCONCLUSIVE
```

---

# 137. Schema Round-Trip Tests

Propiedad:

```text
Model
→ Compile
→ Execute
→ Introspect
→ Normalize
≈
Model
```

---

# 138. Formal round-trip

Sea:

```text
M = SchemaModel
C = Compile
E = Execute
I = Introspect
N = Normalize
```

entonces:

```text
N(I(E(C(M))))
≈
N(M)
```

dentro del conjunto de capabilities observables.

---

# 139. Why ≈ instead of =

Porque representación física y metadata pueden variar sin cambiar la semántica requerida.

---

# 140. Schema Idempotency

Cuando aplique:

```text
Apply(Target)
Apply(Target)
```

no deberá generar cambios inesperados.

---

# 141. Diff Idempotency

Después de aplicar target:

```text
Diff(
    Introspect(Database),
    Target
)
=
∅
```

si la introspección tiene coverage suficiente.

---

# 142. Repeated Migration Tests

Podrán verificar que una migration marcada como one-time no se ejecute dos veces.

---

# 143. Schema Reset

Los tests necesitan volver a un estado conocido.

Estrategias:

```text
DROP_DATABASE
DROP_SCHEMA
DROP_ALL_OBJECTS
MIGRATE_FRESH
RESTORE_SNAPSHOT
RECREATE_CONNECTION_TARGET
```

---

# 144. Reset strategy

No habrá una estrategia universal.

---

# 145. SQLite

Puede ser eficiente:

```text
new temporary database
```

---

# 146. PostgreSQL

Puede utilizar:

```text
isolated schema
```

o database dedicada.

---

# 147. MySQL/MariaDB

Puede utilizar:

```text
isolated database
```

según entorno.

---

# 148. Reset safety

Nunca deberá ejecutarse reset destructivo contra un destino no reconocido como test environment.

---

# 149. Test Environment Guard

Antes de:

```text
DROP DATABASE
DROP SCHEMA
DROP TABLE
```

el sistema deberá validar:

```text
environment identity
test marker
allowed host
database name policy
credentials
```

---

# 150. Production guard

Una conexión marcada como:

```text
PRODUCTION
```

deberá rechazar operaciones destructivas de testing.

---

# 151. Defense in depth

No depender únicamente de:

```text
APP_ENV=test
```

---

# 152. Database marker

Un test environment podrá tener metadata/marker específico.

---

# 153. Schema Isolation

Tests paralelos deberán utilizar:

```text
separate database
separate schema
unique namespace
```

según plataforma.

---

# 154. Shared schema caution

Compartir tablas entre tests paralelos aumenta:

```text
DDL race conditions
lock contention
cleanup conflicts
```

---

# 155. Parallel DDL

Deberá ser limitado o aislado.

---

# 156. Migration concurrency

No deberán correr múltiples migradores contra el mismo test schema accidentalmente.

---

# 157. Unique Test Schema

Ejemplo conceptual:

```text
vs_test_worker_03_case_0182
```

---

# 158. Identifier length

El generador deberá respetar límites de la plataforma.

---

# 159. Deterministic prefix

Podrá combinar:

```text
run id
worker id
test id
hash
```

---

# 160. Cleanup

Toda suite deberá intentar:

```text
finally → cleanup
```

---

# 161. Cleanup failure

No deberá ocultar el fallo original.

---

# 162. Cleanup diagnostics

Reportará:

```text
test failure
cleanup failure
remaining schema
environment id
```

---

# 163. Quarantine

Un entorno cuyo cleanup haya fallado podrá marcarse:

```text
QUARANTINED
```

para impedir reuse inseguro.

---

# 164. Transactional Schema Testing

No deberá asumirse que DDL puede revertirse transaccionalmente en todos los DBMS.

---

# 165. Platform semantics

La suite deberá utilizar capabilities para saber:

```text
transactional DDL
implicit commits
savepoint interaction
```

---

# 166. Transaction rollback ≠ universal schema reset

Regla crítica.

---

# 167. Schema Test Fixtures

Podrán existir:

```text
EmptySchema
BasicUserSchema
RelationshipSchema
ComplexConstraintSchema
LargeSchema
LegacySchema
```

---

# 168. Fixture immutability

Schema fixtures reutilizables deberán ser inmutables.

---

# 169. Fixture ≠ Migration

Una fixture describe un estado de prueba.

---

# 170. Schema Test Factory

Podrá ayudar a crear:

```text
TableDefinition
ColumnDefinition
IndexDefinition
ConstraintDefinition
SchemaModel
```

---

# 171. Synthetic Schema Models

Muy útiles para:

```text
Diff
Planner
Compiler
Compatibility
```

Unit Tests.

---

# 172. Invalid Schema Factory

También podrá construir casos deliberadamente inválidos para Validation Tests.

---

# 173. Property-Based Schema Testing

Puede generar combinaciones de:

```text
tables
columns
indexes
constraints
```

bajo límites controlados.

---

# 174. Useful properties

Ejemplos:

```text
Diff(A,A)=∅
Normalize(Normalize(A))=Normalize(A)
```

---

# 175. Diff inverse property

No siempre podrá afirmarse:

```text
Apply(Diff(A,B), A) = B
```

si existen:

```text
unsupported features
unknown observations
lossy transformations
```

---

# 176. Conditional property

Será válida sólo bajo precondiciones explícitas.

---

# 177. Fuzz Testing

Podrá utilizarse para:

```text
identifier edge cases
large schemas
deep constraint graphs
unusual defaults
```

---

# 178. Reproducibility

Todo fuzz failure deberá registrar:

```text
seed
generated schema
platform profile
capabilities
```

---

# 179. Identifier Testing

Casos:

```text
reserved words
unicode
mixed case
long identifiers
quoted identifiers
special valid characters
```

---

# 180. SQL injection through identifiers

Schema APIs deberán validar identifiers.

Tests deberán incluir entradas maliciosas.

---

# 181. Raw Schema Expressions

Escape hatches deberán tener suites específicas.

---

# 182. Raw expression ≠ trusted automatically

El caller deberá declarar confianza según modelo de seguridad.

---

# 183. Constraint Graph Testing

Foreign keys pueden formar:

```text
DAG
cycles
self references
```

---

# 184. Planner Tests

Deberán verificar ordering de operaciones.

Ejemplo:

```text
create parent
create child
add FK
```

cuando sea requerido.

---

# 185. Cyclic FK

El planner deberá utilizar estrategias capability-aware.

---

# 186. Drop ordering

Ejemplo:

```text
drop FK
drop child
drop parent
```

según dependencias.

---

# 187. Schema Dependency Graph

Testing deberá comprobar:

```text
nodes
edges
cycles
topological ordering
```

---

# 188. Rename Tests

Casos:

```text
table rename
column rename
index rename
constraint rename
```

---

# 189. Capability-aware rename

No todos los objetos/plataformas soportan rename directo.

---

# 190. Emulation

Si el sistema utiliza:

```text
create-copy-drop
```

deberá probarse que la emulación preserva las propiedades requeridas.

---

# 191. Emulation ≠ native support

Las assertions deberán poder distinguir:

```text
NATIVE
EMULATED
DEGRADED
```

---

# 192. Data-preserving schema tests

Para determinadas migraciones será necesario insertar datos antes del cambio.

---

# 193. Example

Para:

```text
ALTER COLUMN TYPE
```

la suite puede:

```text
create schema
insert boundary data
migrate
verify data
verify schema
```

---

# 194. Schema correctness ≠ data preservation

Ambas deben probarse cuando sean requeridas.

---

# 195. NOT NULL Migration

Caso:

```text
nullable
→
not null
```

deberá probar:

```text
existing null data
backfill
constraint application
```

según plan.

---

# 196. Unique Constraint Migration

Debe considerar duplicados existentes.

---

# 197. Foreign Key Migration

Debe considerar referencias inválidas existentes.

---

# 198. Type narrowing

Debe considerar:

```text
overflow
truncation
conversion failure
```

---

# 199. Migration Safety Data Probe

Si el sistema utiliza probes de seguridad, deberán ser:

```text
bounded
read-only
timeout-aware
```

---

# 200. Safety probe failure

No significa:

```text
safe
```

sino potencialmente:

```text
UNKNOWN
```

---

# 201. Schema Lock Testing

Algunas operaciones DDL pueden adquirir locks significativos.

Esto requiere suites de integración especializadas.

---

# 202. Lock behavior ≠ Compiler property

No deberá probarse con SQL snapshot.

---

# 203. Concurrent Schema Tests

Podrán verificar:

```text
migration vs application read
migration vs application write
two migration processes
```

cuando el DBMS lo permita.

---

# 204. Synchronization

Usar:

```text
barriers
latches
controlled connections
```

en lugar de sleeps arbitrarios.

---

# 205. Bounded timeout

Toda prueba concurrente deberá tener deadline.

---

# 206. Deadlock

Un deadlock causado por DDL deberá clasificarse correctamente.

---

# 207. Schema Performance

No pertenece principalmente a este documento, pero podrán existir límites de regresión para:

```text
large schema diff
metadata normalization
dependency graph
```

---

# 208. Real DDL performance

Pertenece principalmente a:

```text
293_DATABASE_PERFORMANCE_TESTING_SYSTEM.md
```

---

# 209. Large Schema Tests

Ejemplo:

```text
1000 tables
10000 columns
multiple indexes
FK graph
```

para comprobar complejidad algorítmica.

---

# 210. Memory Tests

Schema Model, Diff y Metadata no deberán crecer de forma patológica.

---

# 211. Schema Cache Integration Tests

Podrán comprobar:

```text
compile metadata
cache
schema changes
invalidate
recompile
```

---

# 212. Schema Generation

Un cambio estructural podrá incrementar:

```text
SchemaGeneration
```

para invalidar caches relacionados.

---

# 213. Compiled Query Cache

Un cambio de schema relevante puede invalidar:

```text
compiled query assumptions
```

según dependencias.

---

# 214. Schema Test Diagnostics

Un fallo deberá mostrar:

```text
expected schema
observed schema
source
coverage
platform
capabilities
differences
```

---

# 215. Example diagnostic

```text
Schema assertion failed.

Table:
users

Expected column:
email
type=STRING
length=255
nullable=false

Observed:
email
type=STRING
length=191
nullable=false

Platform:
mysql

Coverage:
columns=COMPLETE
```

---

# 216. Missing vs Unknown diagnostic

Debe distinguir:

```text
COLUMN MISSING
```

de:

```text
COLUMN METADATA UNKNOWN
```

---

# 217. Diff Diagnostics

Ejemplo:

```text
Expected:
ADD COLUMN timezone

Observed Diff:
ALTER COLUMN timezone
```

---

# 218. Compiler Diagnostics

Mostrar:

```text
Schema Operation
Capability Snapshot
Dialect
Compiled Command
```

sin secretos.

---

# 219. Integration Diagnostics

Podrán incluir:

```text
DBMS
server version
driver
schema name
capability fingerprint
test seed
```

---

# 220. Schema Test Evidence Record

Conceptualmente:

```php
final readonly class SchemaTestEvidence
{
    public function __construct(
        public SchemaTestId $testId,
        public SchemaAssertionSource $source,
        public ?DatabaseEnvironmentFingerprint $environment,
        public ?CapabilitySnapshot $capabilities,
        public SchemaCoverage $coverage,
        public SchemaTestResult $result,
    ) {}
}
```

---

# 221. Evidence provenance

La suite deberá saber si una afirmación proviene de:

```text
synthetic model
fake introspector
compiler
real introspection
```

---

# 222. Fake introspection evidence

Demuestra comportamiento de VoltStack ante metadata controlada.

No demuestra comportamiento del DBMS.

---

# 223. Real introspection evidence

Sí proporciona evidencia sobre la configuración concreta probada.

---

# 224. Environment-specific evidence

Un resultado en:

```text
PostgreSQL version X
driver Y
configuration Z
```

no deberá generalizarse automáticamente a toda versión/configuración.

---

# 225. Cross-Platform Conformance

El Schema subsystem deberá ejecutar suites compartidas contra cada plataforma soportada.

---

# 226. Shared conformance tests

Ejemplos:

```text
basic table
nullable column
unique constraint
foreign key
index
schema introspection round-trip
```

---

# 227. Capability-conditioned conformance

Ejemplo:

```text
IF capability partial_index = supported
THEN partial index conformance suite must pass
```

---

# 228. Unsupported capability test

También deberá comprobarse que VoltStack:

```text
rejects correctly
```

cuando una feature no está soportada.

---

# 229. UNKNOWN capability test

Deberá comprobarse que el sistema aplica la policy definida:

```text
REJECT
PROBE_IF_SAFE
REQUIRE_OVERRIDE
```

---

# 230. Platform matrix

La suite completa deberá contemplar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 231. Multiple versions

Podrán existir niveles CI:

```text
minimum supported
representative current
latest validated
```

según política de compatibilidad.

---

# 232. SQLite fast suite

SQLite puede utilizarse para feedback rápido.

Pero:

```text
SQLite Pass
≠
MySQL Pass
≠
PostgreSQL Pass
```

---

# 233. MySQL ≠ MariaDB

Ambos tendrán suites propias.

---

# 234. CI Strategy

Ejemplo:

```text
PR
├── Unit Schema Tests
├── Schema Diff Tests
├── Compiler Tests
└── Fast SQLite Integration

Main
├── MySQL Integration
├── MariaDB Integration
├── PostgreSQL Integration
└── SQLite Integration

Nightly
├── Version Matrix
├── Conformance
├── Large Schema
├── Migration Safety
└── ZDT scenarios
```

---

# 235. Persistent Runtime Testing

Schema operations normalmente son menos frecuentes por request, pero el runtime persistente sigue siendo relevante.

---

# 236. Metadata leakage

No deberá ocurrir:

```text
Request A schema context
→
Request B
```

---

# 237. Tenant schema leakage

Especialmente en:

```text
schema-per-tenant
```

---

# 238. FrankenPHP

Deberán probarse operaciones repetidas en el mismo worker.

---

# 239. RoadRunner

Igualmente.

---

# 240. OpenSwoole

Dos coroutines con distintos schema contexts deberán permanecer aisladas.

---

# 241. Schema Introspector cache

No deberá compartir accidentalmente metadata entre:

```text
different tenant
different database
different schema
different server
```

---

# 242. Cache key

Deberá contener la identidad estructural necesaria.

---

# 243. Schema Assertions API

VoltStack podrá ofrecer helpers:

```php
assertTableExists();
assertTableDoesNotExist();

assertColumnExists();
assertColumnDoesNotExist();

assertColumnType();
assertColumnNullable();

assertIndexExists();
assertUniqueConstraint();
assertPrimaryKey();
assertForeignKey();

assertSchemaMatches();
assertSchemaDiff();
assertNoSchemaDrift();
```

---

# 244. Real DB assertions

La API deberá dejar claro cuándo se consulta/introspecta el DB real.

Ejemplo conceptual:

```php
$this->assertDatabaseSchema()
    ->table('users')
    ->hasColumn('email');
```

---

# 245. In-memory Model assertions

Podrá existir:

```php
$this->assertSchemaModel($model)
    ->table('users')
    ->hasColumn('email');
```

La diferencia debe ser visible en la API.

---

# 246. No hidden introspection

Una assertion sobre un `SchemaModel` no deberá abrir una conexión inesperadamente.

---

# 247. No hidden mutation

Una assertion nunca deberá:

```text
create
alter
drop
```

objetos salvo que el helper sea explícitamente un scenario runner.

---

# 248. Scenario Runner

Para round-trip tests podrá existir:

```php
$scenario = SchemaTestScenario::create()
    ->apply($schema)
    ->introspect()
    ->assertEquivalent();
```

---

# 249. Scenario runner lifecycle

```text
PROVISION
PREPARE
EXECUTE
INTROSPECT
ASSERT
CLEANUP
```

---

# 250. Failure handling

Si falla:

```text
EXECUTE
```

aun deberá intentarse cleanup seguro.

---

# 251. UNKNOWN DDL Outcome

Una pérdida de conexión durante DDL puede producir estado incierto.

El sistema deberá preservar:

```text
UNKNOWN
```

---

# 252. UNKNOWN resolution

Podrá requerirse:

```text
new connection
+
introspection
```

para determinar estado observado.

---

# 253. UNKNOWN ≠ rollback assumption

Nunca:

```text
connection lost
→
assume schema unchanged
```

---

# 254. DDL recovery tests

Podrán simular o ejecutar escenarios:

```text
connection lost during ALTER
```

según capacidades del entorno.

---

# 255. Test Environment Requirements

Cada scenario podrá declarar:

```text
requires isolated database
requires admin DDL
requires multiple connections
requires transactional DDL
requires specific capability
```

---

# 256. Skip semantics

Si el entorno no satisface una capability opcional:

```text
SKIPPED_UNSUPPORTED
```

puede ser correcto.

---

# 257. UNKNOWN ≠ skip automatically

Una capability UNKNOWN requerida para conformance deberá generar:

```text
INCONCLUSIVE
ERROR
```

o probing según policy, no un skip silencioso.

---

# 258. Schema Test Status

Valores conceptuales:

```text
PASS
FAIL
SKIPPED_UNSUPPORTED
INCONCLUSIVE
INFRASTRUCTURE_ERROR
```

---

# 259. Infrastructure Error

Ejemplos:

```text
DB unavailable
cleanup failure
test environment corrupted
credentials invalid
```

---

# 260. Product Failure

Ejemplo:

```text
expected FK not created
```

deberá distinguirse de infrastructure error.

---

# 261. Proposed namespace

```text
src/Quantum/Database/Testing/Schema/
├── Contract/
│   ├── SchemaAssertionEngine.php
│   ├── SchemaTestEnvironment.php
│   └── SchemaScenarioRunner.php
│
├── Assertion/
│   ├── SchemaAssertion.php
│   ├── TableAssertion.php
│   ├── ColumnAssertion.php
│   ├── IndexAssertion.php
│   ├── ConstraintAssertion.php
│   ├── ForeignKeyAssertion.php
│   ├── SchemaDiffAssertion.php
│   ├── SchemaCoverageAssertion.php
│   ├── SchemaCompatibilityAssertion.php
│   └── SchemaDriftAssertion.php
│
├── Expectation/
│   ├── SchemaExpectation.php
│   ├── TableExpectation.php
│   ├── ColumnExpectation.php
│   ├── IndexExpectation.php
│   ├── ConstraintExpectation.php
│   └── SchemaOperationExpectation.php
│
├── Match/
│   ├── SchemaMatcher.php
│   ├── TableMatcher.php
│   ├── ColumnMatcher.php
│   ├── ConstraintMatcher.php
│   └── SchemaSemanticComparator.php
│
├── Scenario/
│   ├── SchemaRoundTripScenario.php
│   ├── SchemaMigrationScenario.php
│   ├── SchemaRollbackScenario.php
│   ├── SchemaDriftScenario.php
│   └── ZeroDowntimeSchemaScenario.php
│
├── Snapshot/
│   ├── SchemaSnapshot.php
│   ├── SchemaSnapshotSerializer.php
│   └── SchemaSnapshotVersion.php
│
├── Fixture/
│   ├── SchemaFixture.php
│   ├── SchemaFixtureRegistry.php
│   └── SchemaTestFactory.php
│
├── Isolation/
│   ├── SchemaIsolationStrategy.php
│   ├── TestSchemaNameGenerator.php
│   └── SchemaCleanupManager.php
│
├── Evidence/
│   ├── SchemaTestEvidence.php
│   └── SchemaEnvironmentFingerprint.php
│
├── Diagnostic/
│   ├── SchemaAssertionDiagnostic.php
│   ├── SchemaDifferenceFormatter.php
│   └── SchemaTestFailureReport.php
│
├── PHPUnit/
│   └── InteractsWithDatabaseSchema.php
│
└── Exception/
    └── ...
```

---

# 262. Excepciones

```text
DatabaseSchemaTestingException
├── SchemaAssertionFailedException
├── TableAssertionFailedException
├── ColumnAssertionFailedException
├── IndexAssertionFailedException
├── ConstraintAssertionFailedException
├── ForeignKeyAssertionFailedException
├── SchemaDiffAssertionFailedException
├── SchemaCoverageAssertionFailedException
├── SchemaCompatibilityAssertionFailedException
├── SchemaDriftDetectedException
├── SchemaTestEnvironmentException
├── UnsafeSchemaTestEnvironmentException
├── SchemaCleanupException
├── SchemaScenarioException
└── SchemaTestInconclusiveException
```

---

# 263. Invariantes

## DB-SCHEMA-TEST-001

Schema Definition no será Schema Model.

## DB-SCHEMA-TEST-002

Schema Model no será Schema AST.

## DB-SCHEMA-TEST-003

Schema AST no será Migration.

## DB-SCHEMA-TEST-004

Schema Diff no será Migration Plan.

## DB-SCHEMA-TEST-005

Migration Plan no será Compiled DDL.

## DB-SCHEMA-TEST-006

Compiled DDL no será Observed Schema.

## DB-SCHEMA-TEST-007

Compiler success no demostrará DBMS acceptance.

## DB-SCHEMA-TEST-008

Statement success no demostrará schema final correcto.

## DB-SCHEMA-TEST-009

Schema Model assertion no abrirá DB implícitamente.

## DB-SCHEMA-TEST-010

Real schema assertion utilizará introspection real.

## DB-SCHEMA-TEST-011

Schema testing será distinto de data-state testing.

## DB-SCHEMA-TEST-012

Schema testing será distinto de migration testing.

## DB-SCHEMA-TEST-013

SQL string testing no será el default.

## DB-SCHEMA-TEST-014

Schema expectations serán distintas de Schema Builder.

## DB-SCHEMA-TEST-015

Partial y exact matching serán distintos.

## DB-SCHEMA-TEST-016

Semantic comparison permitirá diferencias físicas equivalentes.

## DB-SCHEMA-TEST-017

Qualified table identity será explícita.

## DB-SCHEMA-TEST-018

Case sensitivity no será asumida universalmente.

## DB-SCHEMA-TEST-019

Logical Type será distinto de Physical Type.

## DB-SCHEMA-TEST-020

Column defaults preservarán literal/expression/absence distinction.

## DB-SCHEMA-TEST-021

DEFAULT NULL no se asumirá igual a NO DEFAULT.

## DB-SCHEMA-TEST-022

Identity semantics serán distintas de vendor-specific auto increment.

## DB-SCHEMA-TEST-023

Index será distinto de Constraint.

## DB-SCHEMA-TEST-024

UniqueIndex será distinto de UniqueConstraint.

## DB-SCHEMA-TEST-025

Primary Key será distinto de Index.

## DB-SCHEMA-TEST-026

Foreign Key assertions preservarán referential semantics.

## DB-SCHEMA-TEST-027

Generated physical names no serán obligatorios en assertions semánticas.

## DB-SCHEMA-TEST-028

CHECK expression equivalence no se inferirá arbitrariamente.

## DB-SCHEMA-TEST-029

Schema Builder será determinista bajo mismo contexto.

## DB-SCHEMA-TEST-030

Builder validation tendrá pruebas negativas.

## DB-SCHEMA-TEST-031

Diff(A,A)=∅.

## DB-SCHEMA-TEST-032

Diff será direccional.

## DB-SCHEMA-TEST-033

Difference será distinta de Migration.

## DB-SCHEMA-TEST-034

Rename inference requerirá evidencia.

## DB-SCHEMA-TEST-035

NotObserved será distinto de Absent.

## DB-SCHEMA-TEST-036

Schema Coverage será explícita.

## DB-SCHEMA-TEST-037

PARTIAL coverage no producirá certeza completa.

## DB-SCHEMA-TEST-038

UNKNOWN metadata no será inventada.

## DB-SCHEMA-TEST-039

Fake introspection no será real DB evidence.

## DB-SCHEMA-TEST-040

Real introspection requerirá DBMS real.

## DB-SCHEMA-TEST-041

Round-trip testing utilizará normalización semántica.

## DB-SCHEMA-TEST-042

Round-trip equality no requerirá igualdad física absoluta.

## DB-SCHEMA-TEST-043

Metadata cache no será DB truth.

## DB-SCHEMA-TEST-044

Schema Compiler no ejecutará DDL.

## DB-SCHEMA-TEST-045

Compiler tests existirán por plataforma.

## DB-SCHEMA-TEST-046

MySQL será distinto de MariaDB.

## DB-SCHEMA-TEST-047

Compatibility será distinta de compilabilidad.

## DB-SCHEMA-TEST-048

Capabilities determinarán features, no sólo vendor names.

## DB-SCHEMA-TEST-049

Synthetic capability no demostrará soporte real.

## DB-SCHEMA-TEST-050

Real capability discovery tendrá integración.

## DB-SCHEMA-TEST-051

No habrá silent feature degradation.

## DB-SCHEMA-TEST-052

Portability profile será explícito.

## DB-SCHEMA-TEST-053

Platform-specific schema será permitido explícitamente.

## DB-SCHEMA-TEST-054

Migration applied record no demostrará schema final correcto.

## DB-SCHEMA-TEST-055

Forward migration verificará estado final.

## DB-SCHEMA-TEST-056

Rollback verificará equivalencia semántica cuando aplique.

## DB-SCHEMA-TEST-057

Irreversible migrations serán identificadas.

## DB-SCHEMA-TEST-058

Destructive change será distinto de unsafe change.

## DB-SCHEMA-TEST-059

Schema Diff será distinto de Migration Safety.

## DB-SCHEMA-TEST-060

UNKNOWN safety no será SAFE.

## DB-SCHEMA-TEST-061

Zero-downtime no será propiedad exclusiva del DDL.

## DB-SCHEMA-TEST-062

ZDT testing incluirá application/schema compatibility.

## DB-SCHEMA-TEST-063

Schema snapshot será distinto de Migration.

## DB-SCHEMA-TEST-064

Schema snapshot será distinto de Backup.

## DB-SCHEMA-TEST-065

Schema snapshot tendrá versionado.

## DB-SCHEMA-TEST-066

Snapshots excluirán metadata volátil por defecto.

## DB-SCHEMA-TEST-067

Schema drift utilizará Expected vs Observed.

## DB-SCHEMA-TEST-068

No drift observado con coverage parcial no será certeza total.

## DB-SCHEMA-TEST-069

Round-trip se evaluará dentro de capabilities observables.

## DB-SCHEMA-TEST-070

Schema reset será strategy-based.

## DB-SCHEMA-TEST-071

Transaction rollback no será schema reset universal.

## DB-SCHEMA-TEST-072

Reset destructivo requerirá test-environment guard.

## DB-SCHEMA-TEST-073

APP_ENV=test no será defensa suficiente por sí sola.

## DB-SCHEMA-TEST-074

Production connections rechazarán destructive test reset.

## DB-SCHEMA-TEST-075

Parallel schema tests estarán aislados.

## DB-SCHEMA-TEST-076

DDL races serán evitadas mediante aislamiento/coordinación.

## DB-SCHEMA-TEST-077

Test schema names respetarán límites del DBMS.

## DB-SCHEMA-TEST-078

Cleanup se ejecutará en finally cuando sea seguro.

## DB-SCHEMA-TEST-079

Cleanup failure no ocultará test failure.

## DB-SCHEMA-TEST-080

Environment con cleanup fallido podrá ser quarantined.

## DB-SCHEMA-TEST-081

Transactional DDL será capability-driven.

## DB-SCHEMA-TEST-082

Implicit commit behavior será probado por plataforma.

## DB-SCHEMA-TEST-083

Schema fixtures serán inmutables cuando se compartan.

## DB-SCHEMA-TEST-084

Fixture será distinta de Migration.

## DB-SCHEMA-TEST-085

Synthetic schemas serán válidos para Unit Tests.

## DB-SCHEMA-TEST-086

Invalid schema generation será soportada para negative testing.

## DB-SCHEMA-TEST-087

Property-based failures serán reproducibles por seed.

## DB-SCHEMA-TEST-088

Identifier edge cases tendrán suites específicas.

## DB-SCHEMA-TEST-089

Identifier injection será probada.

## DB-SCHEMA-TEST-090

Raw schema expressions tendrán security tests.

## DB-SCHEMA-TEST-091

Schema dependency graph tendrá tests.

## DB-SCHEMA-TEST-092

Cycle detection tendrá tests.

## DB-SCHEMA-TEST-093

Operation ordering respetará dependencies.

## DB-SCHEMA-TEST-094

Rename support será capability-aware.

## DB-SCHEMA-TEST-095

Native rename será distinto de emulated rename.

## DB-SCHEMA-TEST-096

Emulation deberá preservar required semantics.

## DB-SCHEMA-TEST-097

Schema correctness será distinta de data preservation.

## DB-SCHEMA-TEST-098

Data-preserving migrations tendrán tests con datos cuando corresponda.

## DB-SCHEMA-TEST-099

NOT NULL migrations considerarán datos existentes.

## DB-SCHEMA-TEST-100

Unique constraints considerarán duplicados existentes.

## DB-SCHEMA-TEST-101

Foreign keys considerarán invalid references existentes.

## DB-SCHEMA-TEST-102

Type narrowing considerará pérdida/conversión.

## DB-SCHEMA-TEST-103

Safety probes serán bounded.

## DB-SCHEMA-TEST-104

Probe failure no significará safe.

## DB-SCHEMA-TEST-105

DDL lock behavior requerirá integración real.

## DB-SCHEMA-TEST-106

Concurrent schema tests utilizarán sincronización explícita.

## DB-SCHEMA-TEST-107

Concurrent tests tendrán deadlines.

## DB-SCHEMA-TEST-108

Large-schema algorithm tests serán distintos de DB performance tests.

## DB-SCHEMA-TEST-109

Schema metadata caches tendrán invalidation tests.

## DB-SCHEMA-TEST-110

Schema generation podrá participar en cache invalidation.

## DB-SCHEMA-TEST-111

Diagnostics distinguirán missing de unknown.

## DB-SCHEMA-TEST-112

Diagnostics incluirán source provenance.

## DB-SCHEMA-TEST-113

Diagnostics incluirán capability context cuando sea relevante.

## DB-SCHEMA-TEST-114

Diagnostics no expondrán credentials.

## DB-SCHEMA-TEST-115

Evidence distinguirá synthetic de real.

## DB-SCHEMA-TEST-116

Real evidence será environment-specific.

## DB-SCHEMA-TEST-117

Cross-platform conformance usará DBMS reales.

## DB-SCHEMA-TEST-118

Unsupported features podrán producir SKIPPED_UNSUPPORTED.

## DB-SCHEMA-TEST-119

UNKNOWN capability no producirá skip silencioso.

## DB-SCHEMA-TEST-120

SQLite pass no demostrará MySQL/PostgreSQL compatibility.

## DB-SCHEMA-TEST-121

Version matrix podrá probar múltiples versiones soportadas.

## DB-SCHEMA-TEST-122

Persistent runtime no filtrará schema context.

## DB-SCHEMA-TEST-123

Tenant schema metadata estará correctamente scoped.

## DB-SCHEMA-TEST-124

FrankenPHP worker reuse tendrá schema-state tests.

## DB-SCHEMA-TEST-125

RoadRunner worker reuse tendrá schema-state tests.

## DB-SCHEMA-TEST-126

OpenSwoole coroutine schema contexts estarán aislados.

## DB-SCHEMA-TEST-127

Introspection cache key distinguirá database/schema/tenant cuando corresponda.

## DB-SCHEMA-TEST-128

Assertions no ejecutarán introspection oculta sobre in-memory models.

## DB-SCHEMA-TEST-129

Assertions no mutarán schema.

## DB-SCHEMA-TEST-130

Scenario Runner hará mutaciones sólo explícitamente.

## DB-SCHEMA-TEST-131

Scenario lifecycle será controlado.

## DB-SCHEMA-TEST-132

DDL UNKNOWN outcome permanecerá UNKNOWN.

## DB-SCHEMA-TEST-133

Connection loss no implicará schema unchanged.

## DB-SCHEMA-TEST-134

UNKNOWN podrá reconciliarse mediante nueva introspection.

## DB-SCHEMA-TEST-135

Scenario requirements serán declarativas.

## DB-SCHEMA-TEST-136

Infrastructure error será distinto de product failure.

## DB-SCHEMA-TEST-137

Schema tests serán deterministas donde no dependan del DBMS.

## DB-SCHEMA-TEST-138

Real DB tests registrarán environment fingerprint.

## DB-SCHEMA-TEST-139

Capability fingerprint podrá formar parte de evidencia.

## DB-SCHEMA-TEST-140

Schema assertion engine no reimplementará Schema Engine.

## DB-SCHEMA-TEST-141

Schema matcher no generará SQL.

## DB-SCHEMA-TEST-142

Schema matcher no ejecutará migrations.

## DB-SCHEMA-TEST-143

Schema comparator no asumirá vendor semantics sin capability evidence.

## DB-SCHEMA-TEST-144

Physical implementation details sólo se comprobarán cuando sean parte del contrato.

## DB-SCHEMA-TEST-145

Semantic assertions serán preferidas para portabilidad.

## DB-SCHEMA-TEST-146

Physical assertions serán válidas para platform/compiler tests.

## DB-SCHEMA-TEST-147

Schema test state será bounded.

## DB-SCHEMA-TEST-148

Test artifacts podrán conservar snapshots y diagnostics reproducibles.

## DB-SCHEMA-TEST-149

Schema Testing complementará, no sustituirá, Driver Conformance Testing.

## DB-SCHEMA-TEST-150

La máxima evidencia de una feature dependiente del DBMS requerirá ejecutarla contra el DBMS correspondiente.

---

# 264. Anti-patrones

## 264.1 Probar Schema Builder comparando únicamente SQL

Acopla una capa declarativa al Compiler.

---

## 264.2 Considerar compilación como prueba del DBMS

```text
SQL generated
≠
SQL accepted
```

---

## 264.3 Considerar ejecución exitosa como prueba estructural

Debe verificarse mediante introspection cuando la estructura final sea importante.

---

## 264.4 Tratar `NotObserved` como inexistente

Puede producir diffs destructivos incorrectos.

---

## 264.5 Tratar Index y Constraint como equivalentes

Rompe el modelo semántico.

---

## 264.6 Usar sólo SQLite

No valida comportamiento de MySQL, MariaDB o PostgreSQL.

---

## 264.7 Tratar MySQL y MariaDB como una plataforma idéntica

Deben mantener suites separadas.

---

## 264.8 Reset mediante DROP sin guardrails

Riesgo crítico.

---

## 264.9 Confiar sólo en `APP_ENV=test`

No es suficiente para autorizar destrucción.

---

## 264.10 Rollback de transaction como reset universal

DDL no tiene semántica transaccional uniforme.

---

## 264.11 Comparar metadata física byte por byte

Produce falsos negativos entre plataformas.

---

## 264.12 Ignorar coverage de introspection

Produce falsa certeza.

---

## 264.13 Inferir rename agresivamente

Puede convertir drop/add real en rename incorrecto.

---

## 264.14 Considerar capability UNKNOWN como unsupported

Pierde información.

---

## 264.15 Considerar probe failure como feature unsupported

El probe puede haber fallado por permisos o infraestructura.

---

## 264.16 Ocultar cambios destructivos

Toda transformación destructiva debe permanecer visible.

---

## 264.17 Llamar zero-downtime a una sentencia DDL

ZDT es una propiedad del proceso completo.

---

## 264.18 Snapshot como sustituto de Migration

Uno representa estado; otro transición.

---

## 264.19 Compartir schema mutable entre tests paralelos

Genera races.

---

## 264.20 Sleeps para coordinar DDL concurrente

Preferir barriers/latches.

---

# 265. Modelo formal de evidencia

Sea:

```text
D = Desired Schema
P = Platform
C = Capabilities
```

El Compiler produce:

```text
SQL = Compile(D, P, C)
```

Esto demuestra únicamente:

```text
CompilerBehavior(D,P,C)
```

No demuestra:

```text
DatabaseState = D
```

---

# 266. Evidencia física

Para obtener evidencia física:

```text
SQL
 ↓
Execute(DBMS)
 ↓
Introspect(DBMS)
 ↓
ObservedSchema
```

Por tanto:

```text
PhysicalSchemaEvidence
=
Normalize(
    Introspect(
        Execute(
            Compile(D)
        )
    )
)
```

---

# 267. Equivalencia

La condición buscada será:

```text
SemanticEquivalent(
    DesiredSchema,
    ObservedSchema
)
```

bajo:

```text
CapabilityCoverage
```

suficiente.

---

# 268. Diff correctness

Sea:

```text
Δ = Diff(A,B)
```

Una propiedad importante será:

```text
Apply(A,Δ) ≈ B
```

cuando:

```text
Δ
```

sea completamente representable y ejecutable en la plataforma.

---

# 269. Drift

Definimos:

```text
Drift(E,O)
=
SemanticDiff(E,O)
```

donde:

```text
E = Expected Schema
O = Observed Schema
```

Si coverage es incompleto:

```text
DriftResult
```

podrá ser:

```text
INCONCLUSIVE
```

---

# 270. Seguridad

Sea:

```text
S = Schema Change
E = Environment Evidence
P = Safety Policy
```

Entonces:

```text
Safety(S,E,P)
```

no deberá devolver:

```text
SAFE
```

cuando la evidencia requerida sea:

```text
UNKNOWN
```

---

# 271. Estrategia recomendada de prueba por componente

| Componente | Unit | Integration DB | Conformance |
|---|---:|---:|---:|
| Schema Model | Sí | No requerida | No |
| Schema Builder | Sí | Opcional | No |
| Schema AST | Sí | No | No |
| Schema Diff | Sí | Recomendable round-trip | Sí |
| Schema Compiler | Sí | Sí | Sí |
| Schema Introspector | Parcial | Sí | Sí |
| Schema Compatibility | Sí | Sí | Sí |
| Migration Planner | Sí | Sí | Sí |
| Migration Executor | Parcial | Sí | Sí |
| ZDT Safety | Sí | Sí | Escenarios especializados |

---

# 272. Arquitectura consolidada

```text
                    SCHEMA TEST
                         │
                         ▼
                Schema Test Scenario
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
     Definition       Existing        Synthetic
                       DBMS            Model
          │              │               │
          ▼              │               │
    Schema Builder       │               │
          │              │               │
          ▼              │               │
     Schema Model        │               │
          │              │               │
          ├───────┐      │               │
          ▼       ▼      │               │
       Diff      AST     │               │
          │       │      │               │
          ▼       ▼      │               │
      Planner  Compiler  │               │
          │       │      │               │
          └───┬───┘      │               │
              ▼          │               │
           Executor      │               │
              │          │               │
              ▼          │               │
             DBMS ◄──────┘               │
              │                          │
              ▼                          │
         Introspector                    │
              │                          │
              ▼                          │
       Observed Schema                   │
              │                          │
              └────────────┬─────────────┘
                           ▼
                     Normalization
                           │
                           ▼
                    Schema Matcher
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Structural    Semantic       Physical
          Assertions    Assertions     Assertions
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                    Evidence Report
                           │
                           ▼
            PASS / FAIL / INCONCLUSIVE
                           │
                           ▼
                        Cleanup
```

---

# 273. Regla final

> **VoltStack deberá probar Schema en el nivel donde exista la propiedad que se desea demostrar: las definiciones se comprobarán como definiciones, los modelos como modelos, los diffs como transformaciones, los compiladores como compiladores y el comportamiento físico mediante DBMS e introspección reales.**

En forma compacta:

```text
Definition
≠
Physical Schema
```

```text
Schema Model
≠
Observed Schema
```

```text
Diff
≠
Migration
```

```text
Index
≠
Constraint
```

```text
Compiler Success
≠
DBMS Acceptance
```

```text
DDL Success
≠
Desired Schema Verified
```

```text
NotObserved
≠
Absent
```

```text
UNKNOWN
≠
Unsupported
```

```text
Destructive
≠
Automatically Unsafe
```

```text
Migration Applied
≠
Target Schema Verified
```

y la prueba estructural más fuerte seguirá el ciclo:

```text
Define
   ↓
Compile
   ↓
Execute
   ↓
Introspect
   ↓
Normalize
   ↓
Compare
```

de forma que:

```text
Reliable Schema Testing
=
Pure Structural Tests
+
Capability-aware Compiler Tests
+
Real DBMS Integration
+
Introspection Verification
+
Cross-platform Conformance
+
Migration Safety Scenarios
+
Strong Test Isolation
```

---

# 274. Siguiente documento

```text
291_DATABASE_ORM_TESTING_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura oficial para probar el ORM completo:

```text
Database ORM Testing System
│
├── ORM Testing Architecture
├── Entity Tests
├── Entity Metadata Tests
├── Mapping Tests
├── EntityManager Tests
├── Repository Tests
├── Model API Tests
├── IdentityMap Tests
├── UnitOfWork Tests
├── Entity State Tests
├── Change Tracking Tests
├── Snapshot Tests
├── Persistence Planning Tests
├── Insert Tests
├── Update Tests
├── Delete Tests
├── Flush Tests
├── Hydration Tests
├── Partial Entity Tests
├── Relationship Tests
├── Lazy Loading Tests
├── Eager Loading Tests
├── Batch Loading Tests
├── N+1 Tests
├── Optimistic Locking Tests
├── Transaction Interaction Tests
├── Cache Interaction Tests
├── Event Tests
├── Tenant/Shard Context Tests
├── Persistent Runtime Tests
├── Real DBMS ORM Integration Tests
└── ORM Conformance Matrix
```

manteniendo como principio:

> **Una prueba del ORM deberá demostrar las invariantes del modelo de objetos y de persistencia sin confundir el estado en memoria con el estado confirmado en la base de datos; `persist()`, `flush()`, ejecución de statements, commit e introspección posterior representan niveles diferentes de evidencia.**