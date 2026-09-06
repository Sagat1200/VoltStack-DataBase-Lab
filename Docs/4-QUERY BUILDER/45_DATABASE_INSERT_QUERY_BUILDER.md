# 45_DATABASE_INSERT_QUERY_BUILDER.md

# VoltStack Quantum Database
## Insert Query Builder System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 45 — Insert Query Builder  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / INSERT  
**Versión:** 1.0

---

# 1. Propósito

`InsertQueryBuilder` será la API especializada de VoltStack para construir operaciones de inserción de datos.

Ejemplo:

```php
DB::table('users')->insert([
    'name' => 'Ana',
    'email' => 'ana@example.com',
    'active' => true,
]);
```

Esta API tendrá ergonomía Laravel-like, pero internamente no generará:

```sql
INSERT INTO ...
```

directamente.

El flujo será:

```text
Developer
   │
   ▼
InsertQueryBuilder
   │
   ▼
InsertQueryModel
   │
   ▼
InsertQueryNode
   │
   ▼
Normalization
   │
   ▼
Validation
   │
   ▼
Semantic Analysis
   │
   ▼
Optimizer
   │
   ▼
Planner
   │
   ▼
SQL Compiler
   │
   ▼
Executor
```

---

# 2. Regla fundamental

```text
InsertQueryBuilder
≠
INSERT SQL generator
```

Su responsabilidad será expresar:

```text
target relation
target columns
source values/query
conflict intent
returning intent
metadata
parameters
```

mediante estructuras semánticas.

Por tanto:

> `InsertQueryBuilder` describe qué datos deben insertarse y bajo qué semántica; nunca decide directamente cómo debe escribirse el SQL correspondiente.

---

# 3. Responsabilidades

`InsertQueryBuilder` será responsable de construir:

```text
INSERT target
target column list
single-row values
multi-row values
INSERT ... SELECT
DEFAULT VALUES intent
explicit DEFAULT expressions
parameter definitions
runtime bindings
conflict/upsert intent
RETURNING intent
result expectations
query metadata
source provenance
```

---

# 4. No responsabilidades

No será responsable de:

```text
SQL generation
identifier quoting
native placeholders
native parameter binding
schema introspection
column existence resolution
column type inference
generated-column discovery
identity retrieval strategy
sequence handling
constraint discovery
conflict-target resolution
upsert SQL syntax
RETURNING syntax
bulk strategy selection
packet-size calculation
connection acquisition
transaction creation
query execution
ORM entity persistence
UnitOfWork
IdentityMap
```

---

# 5. Arquitectura general

```text
                     InsertQueryBuilder
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
     TargetBuilder     InsertSource      ConflictBuilder
          │                 │                 │
          │          ┌──────┼──────┐          │
          │          ▼      ▼      ▼          │
          │       Values  Query  Default       │
          │                                   │
          └─────────────────┬─────────────────┘
                            │
                            ▼
                   InsertBuilderState
                            │
                            ▼
                   InsertBuilderFinalizer
                            │
                            ▼
                    InsertQueryModel
                            │
                            ▼
                     InsertQueryNode
```

---

# 6. API pública básica

El caso común:

```php
DB::table('users')->insert([
    'name' => 'Ana',
    'email' => 'ana@example.com',
]);
```

conceptualmente producirá:

```text
InsertQueryModel
├── target
│   └── users
├── columns
│   ├── name
│   └── email
└── source
    └── ValuesSource
        └── Row
            ├── Parameter(P1)
            └── Parameter(P2)
```

con:

```text
P1 → "Ana"
P2 → "ana@example.com"
```

---

# 7. Builder explícito

También podrá existir una API más explícita:

```php
DB::insert()
    ->into('users')
    ->values([
        'name' => 'Ana',
        'email' => 'ana@example.com',
    ])
    ->execute();
```

Ambas APIs deberán converger hacia el mismo modelo interno.

---

# 8. InsertBuilderState

Durante la construcción podrá existir:

```php
final class InsertBuilderState
{
    // mutable operation-local construction state
}
```

Conceptualmente:

```text
InsertBuilderState
├── target
├── targetColumns
├── source
├── conflictSpecification
├── returningSpecification
├── parameterRegistry
├── bindingBuilder
├── metadataBuilder
└── sourceMap
```

---

# 9. Lifetime

`InsertBuilderState` será:

```text
operation-scoped
mutable
temporary
non-shared
non-cacheable
```

Nunca:

```text
singleton
static
worker-global
request-global current insert
```

---

# 10. Finalización

```text
InsertBuilderState
       │
       ▼
InsertBuilderFinalizer
       │
       ├───────────────┐
       ▼               ▼
InsertQueryModel    BindingSet
       │
       ▼
InsertQueryNode
```

El modelo final será immutable.

---

# 11. Modelo conceptual

```php
final readonly class InsertQueryModel
{
    public function __construct(
        public RelationReference $target,
        public InsertColumnList $columns,
        public InsertSource $source,
        public ?ConflictSpecification $conflict,
        public ?ReturningSpecification $returning,
        public ParameterDefinitionSet $parameters,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 12. InsertSource

La fuente de datos será una abstracción explícita.

```text
InsertSource
├── ValuesInsertSource
├── QueryInsertSource
├── DefaultValuesInsertSource
└── ExtensionInsertSource
```

Esto evita convertir todas las variantes de `INSERT` en una colección de flags.

---

# 13. Target relation

```php
DB::table('users')->insert(...);
```

produce:

```text
InsertTarget
└── RelationIdentifier(users)
```

---

# 14. Target estructurado

El target nunca deberá mantenerse como:

```text
"users"
```

con significado SQL implícito durante todo el pipeline.

En el boundary podrá recibirse un string, pero se convertirá a:

```text
RelationIdentifier
```

---

# 15. Qualified target

Podrá soportarse:

```php
DB::table('public.users')->insert(...);
```

representándose estructuralmente como:

```text
QualifiedRelationIdentifier
├── namespace/schema: public
└── relation: users
```

La semántica concreta será resuelta posteriormente.

---

# 16. Alias del target

Si una operación INSERT soporta target alias en determinados contextos/plataformas, éste deberá representarse mediante:

```text
AliasedInsertTarget
```

y capability requirements.

No mediante concatenación textual.

---

# 17. Single-row insert

Ejemplo:

```php
DB::table('users')->insert([
    'name' => 'Ana',
    'email' => 'ana@example.com',
    'active' => true,
]);
```

produce:

```text
ValuesInsertSource
└── Row
    ├── P1
    ├── P2
    └── P3
```

---

# 18. Column mapping

La entrada:

```php
[
    'name' => 'Ana',
    'email' => 'ana@example.com',
    'active' => true,
]
```

se separará en:

```text
Columns
├── name
├── email
└── active

Values
├── P1
├── P2
└── P3
```

---

# 19. Valores no forman parte del AST

El AST contendrá:

```text
ParameterExpression(P1)
ParameterExpression(P2)
ParameterExpression(P3)
```

mientras el `BindingSet` contendrá:

```text
P1 → "Ana"
P2 → "ana@example.com"
P3 → true
```

---

# 20. Query shape

Dos inserts:

```php
DB::table('users')->insert([
    'name' => 'Ana',
    'email' => 'ana@example.com',
]);

DB::table('users')->insert([
    'name' => 'Carlos',
    'email' => 'carlos@example.com',
]);
```

deberán poder compartir la misma estructura:

```text
INSERT users
├── name  → P1
└── email → P2
```

con distintos `BindingSet`.

---

# 21. Multi-row insert

Ejemplo:

```php
DB::table('users')->insert([
    [
        'name' => 'Ana',
        'email' => 'ana@example.com',
    ],
    [
        'name' => 'Carlos',
        'email' => 'carlos@example.com',
    ],
]);
```

produce:

```text
ValuesInsertSource
├── Row 1
│   ├── P1
│   └── P2
└── Row 2
    ├── P3
    └── P4
```

---

# 22. Canonical column order

Los associative arrays PHP no deberán convertirse ingenuamente en filas independientes.

Ejemplo:

```php
[
    [
        'name' => 'Ana',
        'email' => 'ana@example.com',
    ],
    [
        'email' => 'carlos@example.com',
        'name' => 'Carlos',
    ],
]
```

deberá normalizarse conceptualmente a:

```text
Columns
├── name
└── email

Row 1
├── Ana
└── ana@example.com

Row 2
├── Carlos
└── carlos@example.com
```

---

# 23. Column-set validation

Si una fila contiene:

```text
name
email
```

y otra:

```text
name
email
active
```

el sistema no deberá depender accidentalmente del orden/shape de arrays PHP.

Deberá aplicar una política explícita.

---

# 24. Política estricta por defecto

V1 deberá preferir:

```text
all multi-row records
must expose compatible column sets
```

De lo contrario:

```text
InconsistentInsertRowShapeException
```

---

# 25. Missing value ≠ NULL

Una columna ausente:

```text
missing
```

no significa necesariamente:

```text
NULL
```

Podría significar:

```text
DEFAULT
```

o error.

La semántica deberá ser explícita.

---

# 26. Explicit NULL

```php
[
    'middle_name' => null,
]
```

significa:

```text
Parameter(P1)
└── runtime value: NULL
```

si la API usa parameterización normal.

---

# 27. Explicit DEFAULT

VoltStack deberá soportar un valor semántico:

```php
DB::default()
```

conceptualmente:

```text
DefaultValueExpression
```

---

# 28. DEFAULT ≠ NULL

Regla crítica:

```text
DEFAULT
≠
NULL
≠
missing value
```

---

# 29. Example

```php
DB::table('users')->insert([
    'name' => 'Ana',
    'status' => DB::default(),
]);
```

produce:

```text
Row
├── name
│   └── Parameter(P1)
└── status
    └── DefaultValueExpression
```

---

# 30. Default values insert

Deberá poder expresarse:

```php
DB::insert()
    ->into('settings')
    ->defaultValues()
    ->execute();
```

produciendo:

```text
DefaultValuesInsertSource
```

---

# 31. DEFAULT VALUES capability

El Builder declara:

```text
DefaultValuesInsertSource
```

El Compiler determina la sintaxis concreta o el Planner una estrategia compatible.

---

# 32. Expression values

Los valores insertados podrán ser expresiones:

```php
DB::table('events')->insert([
    'created_at' => DB::expr()->currentTimestamp(),
]);
```

produce:

```text
FunctionExpression
└── semantic function: CURRENT_TIMESTAMP
```

---

# 33. No vendor functions

El Builder no generará directamente:

```text
NOW()
CURRENT_TIMESTAMP()
datetime('now')
```

---

# 34. Column references as values

En operaciones avanzadas, ciertas expresiones podrán contener referencias permitidas por la semántica correspondiente.

La validación contextual pertenecerá a Validation/Semantic Analysis.

---

# 35. Raw values

Podrá existir:

```php
DB::raw(...)
```

como escape hatch.

El resultado será:

```text
RawExpression
```

con:

```text
trust metadata
portability classification
binding support
semantic opacity
```

---

# 36. Parameter definitions

Cada valor normal deberá registrarse mediante el Parameter System.

Ejemplo:

```text
ParameterDefinition
├── id: P1
├── shape: SCALAR
├── declaredType: ?
├── nullable: ?
└── sensitivity: DEFAULT
```

---

# 37. Type inference

El Builder no decidirá que:

```php
42
```

debe bindearse necesariamente como:

```text
INTEGER
```

La resolución podrá considerar:

```text
explicit type
schema target column
ORM metadata
semantic constraints
safe runtime inference
```

---

# 38. Schema-aware typing

Ejemplo:

```text
users.id
schema type = UUID
```

y:

```php
'id' => '9df...'
```

deberá poder inferirse como:

```text
Query Semantic Type = UUID
```

no simplemente:

```text
PHP string
```

---

# 39. Domain types

Si:

```text
users.id → DomainType<UserId, UUID>
```

el Insert Semantic Analysis podrá preservar dicha identidad.

---

# 40. Type mismatch

Ejemplo:

```text
target:
age → Integer

value:
JSON expression
```

deberá producir un error semántico si no existe coerción válida.

El Builder no resolverá ese error.

---

# 41. Nullability

El Builder no deberá decidir por sí mismo si:

```php
'name' => null
```

es válido.

Schema-aware Semantic Analysis podrá detectar:

```text
name
→ NOT NULL
```

y derivar un error.

---

# 42. Required columns

Semantic Analysis podrá determinar:

```text
column:
email

NOT NULL
no default
not generated
not identity
```

y detectar su ausencia.

---

# 43. Generated columns

El Builder no descubrirá:

```text
generated
computed
virtual
stored generated
identity
auto increment
sequence-backed
```

mediante I/O.

---

# 44. Generated-column validation

Schema-aware resolution podrá determinar que:

```text
generated column
```

no acepta valor explícito bajo la política/plataforma correspondiente.

---

# 45. Identity columns

Una columna identity podrá omitirse:

```php
DB::table('users')->insert([
    'name' => 'Ana',
]);
```

sin que el Builder necesite conocer previamente la estrategia física.

---

# 46. Explicit identity value

Si el desarrollador intenta:

```php
[
    'id' => 100,
]
```

la validez dependerá de:

```text
schema metadata
platform capabilities
identity policy
```

---

# 47. Sequence handling

El Builder no obtendrá valores de secuencia.

No deberá hacer:

```text
SELECT nextval(...)
```

durante construcción.

---

# 48. Insert expressions and sequences

Si una secuencia se expresa explícitamente, deberá hacerse mediante un nodo semántico/extension apropiado.

---

# 49. INSERT ... SELECT

VoltStack soportará:

```php
$source = DB::table('legacy_users')
    ->select('name', 'email')
    ->where('active', true);

DB::insert()
    ->into('users')
    ->columns('name', 'email')
    ->fromQuery($source)
    ->execute();
```

---

# 50. QueryInsertSource

Esto producirá:

```text
InsertQueryModel
├── target: users
├── columns
│   ├── name
│   └── email
└── source
    └── QueryInsertSource
        └── SelectQueryArtifact
```

---

# 51. Child query finalization

El `SelectQueryBuilder` hijo deberá finalizase antes de incorporarse.

```text
SelectQueryBuilder
       │
       ▼
freeze/finalize
       │
       ▼
SelectQueryArtifact
       │
       ▼
QueryInsertSource
```

---

# 52. No mutable child builder retention

El `InsertQueryBuilder` nunca deberá conservar:

```text
mutable SelectQueryBuilder
```

como parte de su estado final.

---

# 53. INSERT SELECT arity

Si:

```text
target columns = 3
source output columns = 2
```

el error pertenece al Semantic Analysis.

---

# 54. INSERT SELECT type compatibility

Semantic Analysis deberá comprobar:

```text
target column 1 ↔ source output 1
target column 2 ↔ source output 2
...
```

---

# 55. INSERT SELECT lineage

El Semantic Graph podrá registrar:

```text
legacy_users.name
      │
      ▼
users.name
```

como data lineage.

---

# 56. Insert from VALUES relation

La arquitectura deberá permitir reutilizar estructuras semánticas cuando sea apropiado:

```text
ValuesRelation
```

y:

```text
ValuesInsertSource
```

sin necesariamente tratarlas como el mismo objeto.

---

# 57. Source abstraction

Regla:

```text
INSERT source
≠
always PHP array
```

Podrá ser:

```text
values
query
default values
extension source
```

---

# 58. Conflict handling

VoltStack deberá representar semánticamente la intención de conflicto.

No deberá modelarla inicialmente como:

```text
ON DUPLICATE KEY UPDATE ...
```

o:

```text
ON CONFLICT ...
```

---

# 59. ConflictSpecification

Conceptualmente:

```php
final readonly class ConflictSpecification
{
    public function __construct(
        public ConflictAction $action,
        public ?ConflictTarget $target,
        public AssignmentSet $updates,
        public ?PredicateNode $condition,
    ) {}
}
```

---

# 60. Conflict actions

Podrán existir:

```text
ERROR
IGNORE
DO_NOTHING
UPDATE
PLATFORM_SPECIFIC
```

---

# 61. Default behavior

Sin especificación:

```text
ConflictAction::ERROR
```

conceptualmente significa:

```text
normal database constraint behavior
```

No que VoltStack capture necesariamente el error y lo transforme.

---

# 62. insertOrIgnore()

API Laravel-like:

```php
DB::table('users')->insertOrIgnore([
    'email' => 'ana@example.com',
]);
```

deberá convertirse en:

```text
ConflictSpecification
└── action: IGNORE / DO_NOTHING semantic intent
```

---

# 63. IGNORE semantics

Debe definirse con cuidado.

Diferentes motores pueden tener conceptos de “ignore” con semánticas distintas.

Por tanto:

```text
IGNORE
```

no deberá asumir que todas las formas vendor-specific son equivalentes.

---

# 64. Portable conflict intent

La arquitectura podrá distinguir:

```text
DO_NOTHING_ON_TARGET_CONFLICT
IGNORE_SUPPORTED_CONFLICT
PLATFORM_IGNORE
```

si es necesario para preservar semántica.

---

# 65. Upsert

API:

```php
DB::table('users')->upsert(
    values: [
        [
            'email' => 'ana@example.com',
            'name' => 'Ana',
        ],
    ],
    uniqueBy: ['email'],
    update: ['name'],
);
```

---

# 66. Upsert representation

Conceptualmente:

```text
InsertQueryModel
├── target: users
├── source: Values
└── conflict
    ├── target
    │   └── email
    └── action: UPDATE
        └── assignments
            └── name
```

---

# 67. Upsert ≠ separate persistence engine

`upsert()` deberá bajar al mismo Query Engine.

No existirá:

```text
special raw SQL upsert subsystem
```

---

# 68. ConflictTarget

Podrá representar:

```text
ColumnSetConflictTarget
ConstraintConflictTarget
IndexConflictTarget
ExpressionConflictTarget
PlatformSpecificConflictTarget
```

según capabilities.

---

# 69. Conflict target resolution

El Builder puede recibir:

```php
uniqueBy: ['email']
```

pero Semantic Analysis determinará:

```text
column symbols
constraint relationships
target validity
capability requirements
```

---

# 70. Constraint names

Si se permite:

```php
->onConflictConstraint('users_email_unique')
```

deberá convertirse a:

```text
ConstraintIdentifier
```

no raw SQL.

---

# 71. Upsert update assignments

Ejemplo:

```text
ON CONFLICT
UPDATE name = incoming.name
```

deberá modelarse mediante expresiones semánticas.

---

# 72. Incoming row reference

Se necesitará una abstracción semántica como:

```text
IncomingValueReference(name)
```

o equivalente.

No deberá codificarse en core como:

```text
EXCLUDED.name
VALUES(name)
```

---

# 73. Vendor independence

Así:

```text
IncomingValueReference(name)
```

podrá compilarse según plataforma sin contaminar AST/Builder.

---

# 74. Upsert assignments

Podrán representarse mediante:

```text
Assignment
├── targetColumn
└── valueExpression
```

---

# 75. Conditional conflict update

Cuando la plataforma lo soporte:

```text
ConflictSpecification
└── condition: PredicateNode
```

podrá expresar condiciones adicionales.

---

# 76. Conflict capability requirements

Semantic Analysis podrá derivar:

```text
DML.CONFLICT.DO_NOTHING
DML.CONFLICT.UPDATE
DML.CONFLICT.TARGET_COLUMNS
DML.CONFLICT.TARGET_CONSTRAINT
DML.CONFLICT.CONDITIONAL_UPDATE
```

---

# 77. Planner responsibility

Si una semántica puede emularse de forma segura, el Planner podrá elegir una estrategia.

---

# 78. No unsafe upsert emulation

VoltStack no deberá emular automáticamente un upsert mediante:

```text
SELECT
then
INSERT/UPDATE
```

si ello introduce race conditions y cambia semántica.

---

# 79. Atomicity requirement

La semántica de upsert deberá poder declarar:

```text
atomic conflict handling required
```

---

# 80. RETURNING

VoltStack deberá modelar el retorno de valores insertados.

Ejemplo:

```php
$result = DB::table('users')
    ->returning('id', 'created_at')
    ->insert([
        'name' => 'Ana',
    ]);
```

---

# 81. ReturningSpecification

Conceptualmente:

```text
ReturningSpecification
├── Projection(id)
└── Projection(created_at)
```

---

# 82. RETURNING ≠ SQL keyword

El AST representará:

```text
return inserted values
```

no necesariamente la palabra:

```text
RETURNING
```

---

# 83. Capability requirement

Semantic Analysis podrá derivar:

```text
DML.INSERT.RETURNING
```

---

# 84. Generated key retrieval

El caso común:

```php
$id = DB::table('users')->insertGetId([
    'name' => 'Ana',
]);
```

no deberá estar ligado a una única técnica física.

---

# 85. insertGetId() intent

Podrá expresarse como:

```text
InsertResultExpectation
└── GENERATED_ID
```

o como:

```text
ReturningSpecification
└── identity column
```

cuando la columna sea conocida.

---

# 86. Identity retrieval strategies

Planner/Executor podrán seleccionar, según capabilities:

```text
RETURNING-like result
driver generated-id API
sequence strategy
platform-specific result
other safe mechanism
```

---

# 87. Builder no llama lastInsertId()

Prohibido:

```php
$pdo->lastInsertId();
```

dentro de `InsertQueryBuilder`.

---

# 88. Generated ID semantics

La API deberá distinguir:

```text
generated scalar identifier
returned row
returned projection set
affected row count
success/failure
```

---

# 89. Insert result modes

Podrán existir:

```text
AFFECTED_ROWS
GENERATED_ID
RETURNING_ROWS
RETURNING_SCALAR
NONE
```

---

# 90. Result expectation ≠ runtime result

El Builder sólo declara:

```text
expectation
```

El Executor produce:

```text
actual result
```

---

# 91. insert()

La API Laravel-like podrá retornar:

```text
bool
affected rows
InsertResult
```

según la decisión final de DX.

Internamente deberá existir un resultado rico y bien definido.

---

# 92. Recomendación interna

Aunque la facade pública pueda simplificar, el Query Engine deberá trabajar con:

```text
InsertExecutionResult
```

conceptualmente:

```text
InsertExecutionResult
├── affectedRows
├── returnedRows?
├── generatedIdentifier?
├── warnings?
└── executionMetadata
```

---

# 93. Public convenience vs core result

```text
Public DX
    │
    ▼
bool / id / rows

Internal Core
    │
    ▼
typed InsertExecutionResult
```

---

# 94. Bulk insert

Un insert con miles o millones de registros no deberá obligatoriamente convertirse en una única sentencia.

---

# 95. Builder responsibility

El Builder puede representar:

```text
large ValuesInsertSource
```

pero no deberá decidir la estrategia física.

---

# 96. Bulk planner

La estrategia pertenecerá posteriormente a:

```text
203_DATABASE_BULK_INSERT_SYSTEM.md
```

y al Planner.

---

# 97. Possible strategies

```text
single multi-row statement
chunked multi-row statements
native bulk protocol
COPY-like mechanism
driver batch
temporary staging relation
platform-specific loader
```

---

# 98. Semantic equivalence

Una estrategia bulk sólo podrá utilizarse si preserva:

```text
atomicity requirements
ordering requirements
generated-value requirements
conflict semantics
returning semantics
transaction semantics
error semantics
```

---

# 99. Parameter limits

El Builder no codificará límites como:

```php
if (count($params) > 65535) { ... }
```

---

# 100. Resource limits

Se obtendrán mediante:

```text
Platform Capability
Driver Capability
Resource Governance
Planning Policy
```

---

# 101. Packet limits

Igualmente, el Builder no calculará límites vendor-specific de paquetes SQL.

---

# 102. Batch shape

Una gran colección podrá describirse mediante:

```text
InsertBatchShape
├── rowCount
├── columnCount
├── parameterCount
├── containsDefaults
├── containsExpressions
├── conflictMode
└── returningMode
```

sin convertir eso todavía en estrategia física.

---

# 103. Streaming bulk source

Futuras APIs podrán permitir:

```text
IterableInsertSource
```

para datasets grandes.

---

# 104. Replayability

Un iterable de una sola pasada deberá clasificarse:

```text
REPLAYABLE
NON_REPLAYABLE
UNKNOWN
```

para retry/bulk planning.

---

# 105. No implicit materialization

El Builder/normalizer no deberá materializar silenciosamente millones de filas sólo para contar parámetros.

---

# 106. Resource governance

Podrán existir límites:

```text
max builder rows
max materialized bytes
max parameters
max row width
max expression nodes
```

---

# 107. Insert source ownership

Si una fuente usa stream/generator:

```text
ownership
consumption
replayability
cleanup
```

deberán definirse explícitamente.

---

# 108. Retry semantics

Un INSERT no deberá marcarse automáticamente como retry-safe.

---

# 109. Retry classification

Metadata podrá expresar:

```text
RetrySafety::UNSAFE
RetrySafety::CONDITIONALLY_SAFE
RetrySafety::IDEMPOTENT
RetrySafety::UNKNOWN
```

---

# 110. Insert usually non-idempotent

Ejemplo:

```php
insert([
    'event' => 'payment_received'
]);
```

repetido puede crear duplicados.

Por tanto:

```text
INSERT
≠
automatically retryable
```

---

# 111. Upsert retry semantics

Incluso un upsert no será automáticamente retry-safe.

Dependerá de:

```text
assignments
generated values
volatile expressions
side effects
triggers
result expectations
transaction outcome certainty
```

---

# 112. Volatile expressions

Ejemplo:

```text
random()
current_timestamp
sequence next value
platform volatile function
```

puede afectar replayability.

---

# 113. Semantic volatility

La clasificación vendrá del Expression/Semantic Function System.

---

# 114. Transactions

`InsertQueryBuilder` podrá declarar:

```text
TransactionRequirement
```

pero nunca iniciar una transacción por sí mismo.

---

# 115. Transaction ownership

```text
TransactionManager
```

será propietario de la transacción activa.

---

# 116. Insert inside transaction

```php
DB::transaction(function () {
    DB::table('users')->insert(...);
});
```

el Builder no necesita conocer el objeto Transaction.

---

# 117. Query context

La query podrá portar:

```text
TransactionRequirement::OPTIONAL
```

o requirements más estrictos.

La transacción real vive en `ExecutionContext` / `TransactionContext`.

---

# 118. Connection routing

INSERT normalmente implica:

```text
QueryIntent::WRITE
```

y:

```text
ConnectionIntent::WRITE
```

---

# 119. WRITE does not mean acquire now

El Builder únicamente declara la intención.

---

# 120. Primary routing

La selección de primary ocurre posteriormente:

```text
Execution Requirements
        │
        ▼
Connection Resolution
        │
        ▼
Primary Endpoint
```

---

# 121. Metadata

Podrá utilizarse:

```php
DB::table('users')
    ->label('users.create')
    ->timeout(seconds: 3)
    ->insert([...]);
```

---

# 122. Metadata possible fields

```text
QueryLabel
QueryOrigin
QueryIntent
ConnectionIntent
TransactionRequirement
StatementTimeout
RetrySafety
SecurityMetadata
AuditRequirement
TelemetryPolicy
DiagnosticPolicy
```

---

# 123. Sensitive values

Ejemplo:

```php
DB::table('users')->insert([
    'password_hash' => $hash,
]);
```

deberá poder marcar:

```text
password_hash binding
→ SENSITIVE
```

mediante schema/ORM/security metadata o API explícita.

---

# 124. Redaction

Logs/diagnostics nunca deberán mostrar por defecto:

```text
password_hash = "$2y$..."
```

---

# 125. Insert telemetry

Podrá registrarse:

```text
query label
target logical relation
row count category
duration
affected rows
success/failure
```

sin registrar todos los valores.

---

# 126. High-cardinality control

Valores insertados nunca deberán convertirse automáticamente en:

```text
metric labels
span names
query labels
```

---

# 127. Audit

Audit podrá necesitar registrar:

```text
operation
target
actor reference
timestamp
policy
result
```

pero Query Builder no conocerá al usuario actual.

---

# 128. Audit context integration

La capa superior proporcionará información audit-safe mediante integración explícita.

---

# 129. ORM integration

ORM podrá utilizar el mismo `InsertQueryBuilder`.

Ejemplo:

```text
EntityManager
      │
      ▼
Persistence Engine
      │
      ▼
Insert Persistence Planner
      │
      ▼
InsertQueryBuilder / InsertQueryModel
```

---

# 130. ORM does not get special SQL engine

Active Record:

```php
$user->save();
```

y EntityManager:

```php
$em->persist($user);
$em->flush();
```

deberán terminar usando el mismo Query Engine.

---

# 131. Active Record path

```text
Model::create()
      │
      ▼
ORM
      │
      ▼
UnitOfWork
      │
      ▼
Persistence Engine
      │
      ▼
Insert Query Model
      │
      ▼
Query Engine
```

---

# 132. Direct query path

```text
DB::table()->insert()
      │
      ▼
Insert Query Model
      │
      ▼
Query Engine
```

---

# 133. Shared lower engine

```text
Active Record ──────┐
                    │
EntityManager ──────┼──► Persistence Engine
                    │          │
Repository ─────────┘          ▼
                        Insert Query Model

DB::table() ───────────────────┘
```

La ruta exacta podrá variar, pero nunca existirán dos compiladores INSERT independientes.

---

# 134. ORM metadata

ORM puede aportar:

```text
table mapping
column mapping
domain types
generated fields
identity strategy
sensitive fields
```

como metadata/evidence.

---

# 135. Query Engine independence

`InsertQueryBuilder` seguirá funcionando sin ORM:

```php
DB::table('users')->insert(...);
```

---

# 136. Schema integration

Schema-aware Semantic Analysis resolverá:

```text
target table
columns
types
nullability
defaults
generated columns
identity
constraints
```

mediante `SchemaView`.

---

# 137. No hidden schema I/O

El Builder nunca deberá ejecutar:

```text
DESCRIBE users
PRAGMA table_info
information_schema query
```

---

# 138. Offline insert compilation

Deberá ser posible:

```text
InsertQueryArtifact
+
OfflineTarget
+
CompiledSchemaMetadata
      │
      ▼
Semantic Analysis
      │
      ▼
Planning
      │
      ▼
Compilation
```

sin conexión física.

---

# 139. Constraint analysis

Insert Semantic Analysis podrá utilizar:

```text
primary keys
unique constraints
foreign keys
check constraints
not-null constraints
generated/default metadata
```

como evidencia.

---

# 140. Constraint analysis ≠ constraint execution

VoltStack puede detectar algunos errores anticipadamente.

La base de datos seguirá siendo autoridad final para constraints físicos.

---

# 141. Foreign keys

El Builder no verificará que:

```text
user_id = 42
```

exista realmente.

Eso requeriría datos runtime y no pertenece a Query Construction.

---

# 142. Check constraints

Schema metadata puede ayudar con análisis estático limitado, pero no deberá intentarse reemplazar al motor de base de datos.

---

# 143. Conflict constraints

Para upsert, Constraint Analysis podrá validar que un conflict target corresponda a una estructura compatible cuando la información esté disponible.

---

# 144. Semantic graph

Un INSERT producirá un grafo semántico.

Ejemplo:

```text
InsertQuery
│
├── TargetRelation(users)
│
├── TargetColumn(name)
│
├── TargetColumn(email)
│
├── Parameter(P1)
│   └── ASSIGNED_TO → name
│
├── Parameter(P2)
│   └── ASSIGNED_TO → email
│
└── Output
    └── affected rows
```

---

# 145. INSERT SELECT graph

```text
legacy_users.name
       │
       └── DERIVES_FROM
               │
               ▼
          users.name
```

---

# 146. Conflict graph

Podrá representar:

```text
ConflictTarget
      │
      ▼
UniqueConstraint
      │
      ▼
ConflictAction
      │
      ▼
Assignments
```

---

# 147. Dependency set

El Semantic Artifact podrá registrar dependencias como:

```text
target table
target columns
source tables
source columns
types
constraints
functions
sequences
capabilities
extensions
```

---

# 148. Capability requirements

Ejemplos:

```text
DML.INSERT.MULTI_ROW
DML.INSERT.DEFAULT_VALUES
DML.INSERT.SELECT
DML.INSERT.RETURNING
DML.CONFLICT.DO_NOTHING
DML.CONFLICT.UPDATE
DML.CONFLICT.CONSTRAINT_TARGET
DML.CONFLICT.CONDITIONAL_UPDATE
DML.INSERT.IDENTITY_OVERRIDE
```

---

# 149. Builder does not resolve capabilities

El Builder puede crear una característica válida estructuralmente aunque el target todavía sea desconocido.

---

# 150. Targeted compilation

Posteriormente:

```text
Semantic Requirements
        +
Target Capability Snapshot
        │
        ▼
Capability Resolution
```

determinará:

```text
native
emulated
unsupported
platform-specific
```

---

# 151. Portability profile

El INSERT podrá clasificarse:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
UNKNOWN
```

---

# 152. MySQL/MariaDB

El Builder no deberá conocer conceptos sintácticos como:

```text
INSERT IGNORE
ON DUPLICATE KEY UPDATE
LAST_INSERT_ID()
```

---

# 153. PostgreSQL

Tampoco deberá conocer directamente:

```text
ON CONFLICT
RETURNING
OVERRIDING SYSTEM VALUE
```

---

# 154. SQLite

Tampoco deberá acoplarse directamente a:

```text
INSERT OR IGNORE
ON CONFLICT
RETURNING
```

---

# 155. Dialect responsibility

Estas diferencias aparecen finalmente en:

```text
Compiler
+
Dialect
+
Platform Capabilities
```

---

# 156. Validation stages

El INSERT atravesará varias validaciones.

```text
Builder Construction Validation
        │
        ▼
Query Structural Validation
        │
        ▼
Semantic Validation
        │
        ▼
Capability Validation
        │
        ▼
Execution-time Validation
```

---

# 157. Construction validation

Puede detectar:

```text
empty target
invalid identifier syntax
malformed rows
inconsistent row shapes
duplicate target columns
invalid builder state
conflicting source definitions
```

---

# 158. Structural validation

Puede detectar:

```text
VALUES and SELECT source simultaneously
DEFAULT VALUES with explicit rows
RETURNING malformed structurally
conflict action without required structure
invalid assignment node shape
```

---

# 159. Semantic validation

Puede detectar:

```text
unknown target relation
unknown target column
generated column assignment
type mismatch
missing required column
invalid conflict target
source/target arity mismatch
source/target type mismatch
invalid incoming-value reference
```

---

# 160. Capability validation

Puede detectar:

```text
RETURNING unsupported
conflict action unsupported
conditional conflict update unsupported
identity override unsupported
required bulk strategy unavailable
```

---

# 161. Runtime validation

Puede detectar:

```text
missing binding
binding conversion failure
transaction requirement unavailable
connection unavailable
server-side constraint violation
deadlock
timeout
```

---

# 162. Error hierarchy

Podrá existir:

```text
InsertQueryBuilderException
├── MissingInsertTargetException
├── MissingInsertSourceException
├── ConflictingInsertSourceException
├── InvalidInsertColumnException
├── DuplicateInsertColumnException
├── InvalidInsertRowException
├── InconsistentInsertRowShapeException
├── InvalidDefaultValueUsageException
├── InvalidConflictSpecificationException
├── InvalidConflictTargetException
├── InvalidConflictAssignmentException
├── InvalidReturningSpecificationException
├── InsertBuilderAlreadyFinalizedException
└── InsertBuilderBudgetExceededException
```

---

# 163. Semantic exceptions

Separadamente:

```text
UnknownInsertTargetException
UnknownInsertColumnException
InsertTypeMismatchException
MissingRequiredInsertColumnException
GeneratedColumnAssignmentException
InsertSourceArityMismatchException
InsertSourceTypeMismatchException
UnresolvedConflictTargetException
UnsupportedInsertCapabilityException
```

---

# 164. Builder lifecycle

```text
CREATED
   │
   ▼
TARGET_DEFINED
   │
   ▼
SOURCE_DEFINED
   │
   ▼
CONFIGURED
   │
   ▼
FINALIZED
```

o:

```text
FAILED
```

---

# 165. Finalized builder

Después de finalización, V1 deberá impedir mutaciones accidentales.

---

# 166. Reuse

Para construir variaciones:

```php
$base = DB::insert()
    ->into('events')
    ->columns('type', 'payload');

$a = $base->copy()->values(...);
$b = $base->copy()->values(...);
```

si esta API se considera útil.

---

# 167. Copy isolation

Los copies no compartirán:

```text
BindingSetBuilder
ParameterRegistry
row collection
conflict builder
returning builder
metadata mutable state
```

---

# 168. Fluent convenience API

Podrán ofrecerse:

```text
insert()
insertOrIgnore()
insertGetId()
upsert()
insertUsing()
```

como convenience operations.

---

# 169. Canonical internal operations

Estas APIs deberán reducirse a un conjunto menor:

```text
Insert Target
+
Insert Source
+
Conflict Specification
+
Returning Specification
+
Result Expectation
```

---

# 170. API surface discipline

No deberá crearse un Query Engine distinto para cada método público.

```text
insert()
insertOrIgnore()
upsert()
insertGetId()
insertUsing()
       │
       ▼
Canonical InsertQueryModel
```

---

# 171. `insertUsing()`

Laravel-like API:

```php
DB::table('pruned_users')->insertUsing(
    ['id', 'name'],
    DB::table('users')
        ->select('id', 'name')
        ->where('active', false)
);
```

deberá convertirse a `QueryInsertSource`.

---

# 172. `insertOrIgnoreUsing()`

Si se ofrece, será:

```text
QueryInsertSource
+
ConflictSpecification
```

no una implementación separada.

---

# 173. `insertGetId()`

Será:

```text
ValuesInsertSource
+
GeneratedIdentifierExpectation
```

---

# 174. `upsert()`

Será:

```text
ValuesInsertSource
+
ConflictSpecification(UPDATE)
```

---

# 175. Terminal execution

El Builder podrá tener:

```php
$query->execute();
```

pero conceptualmente:

```text
execute()
   │
   ▼
finalize builder
   │
   ▼
create QueryExecutionRequest
   │
   ▼
submit to Query Engine
```

---

# 176. Builder does not contain Executor

Preferiblemente la ejecución se realizará mediante un gateway abstraído:

```text
QuerySubmissionGateway
```

o infraestructura superior.

---

# 177. Dependency inversion

```text
InsertQueryBuilder
       │
       ▼
Query Submission Contract

Query Engine
       │
       └── implements/consumes execution path
```

El Builder no deberá importar directamente implementaciones físicas del Executor.

---

# 178. Construction-only builder

También deberá ser posible:

```php
$artifact = DB::insert()
    ->into('users')
    ->values([...])
    ->toQueryArtifact();
```

sin ejecución.

---

# 179. Offline tooling

Esto permitirá:

```text
query inspection
testing
static analysis
query visualization
semantic diagnostics
plan preview
SQL preview
migration tooling
developer debug tooling
```

sin ejecutar la operación.

---

# 180. SQL preview

Si existe:

```php
$query->toSql();
```

deberá ser una convenience API externa que invoque el Compiler.

No significa que el Builder genere SQL.

---

# 181. Better conceptual API

Internamente:

```text
Builder
   │
   ▼
QueryArtifact
   │
   ▼
QueryCompilerService
   │
   ▼
CompiledQuery
```

---

# 182. Explain

`EXPLAIN` de INSERT, cuando exista y sea seguro, será una operación del Query Processing/Administration System.

No una responsabilidad de construcción.

---

# 183. Persistent runtime

La arquitectura deberá funcionar correctamente en:

```text
FrankenPHP
RoadRunner
OpenSwoole
long-running CLI workers
queue workers
coroutines
```

---

# 184. Shared safe objects

Podrán compartirse:

```text
InsertQueryBuilderFactory
IdentifierFactory
ExpressionFactory
frozen extension registry
immutable descriptors
stateless finalizer services
```

---

# 185. Operation-local objects

No se compartirán:

```text
InsertBuilderState
ParameterRegistry
BindingSetBuilder
row normalization state
conflict mutable builder
returning mutable builder
source map
temporary diagnostics
```

---

# 186. Request isolation

```text
Request A
   │
   ▼
InsertBuilder A
   │
   ▼
finalize
   │
   ▼
destroy temporary state

Request B
   │
   ▼
InsertBuilder B
```

---

# 187. Coroutine isolation

```text
Coroutine A
├── ParameterRegistry A
└── BindingSet A

Coroutine B
├── ParameterRegistry B
└── BindingSet B
```

---

# 188. No static counters

Parameter IDs no deberán depender de:

```php
static $parameterCounter++;
```

global al worker.

---

# 189. Deterministic local IDs

Cada query tendrá su propio espacio:

```text
P1
P2
P3
...
```

o IDs equivalentes deterministas.

---

# 190. Insert fingerprints

Deberán distinguirse:

```text
raw builder fingerprint
normalized query fingerprint
semantic fingerprint
planning fingerprint
compiled fingerprint
binding-shape fingerprint
execution fingerprint
```

---

# 191. Runtime values excluded

El structural/semantic fingerprint no incluirá:

```text
email actual
password hash
payload
customer id
```

---

# 192. Row count and fingerprints

La cardinalidad de un multi-row insert puede afectar la forma compilada.

Por tanto deberán distinguirse:

```text
semantic insert shape
```

de:

```text
physical binding/compiled shape
```

---

# 193. Example

Semánticamente:

```text
Insert users(name,email)
from tabular values
```

Puede ser la misma familia de query.

Físicamente:

```text
2 rows
```

y:

```text
500 rows
```

pueden necesitar distintos compiled/bulk plans.

---

# 194. Binding shape specialization

El Planner podrá especializar:

```text
row cardinality
parameter cardinality
default-value distribution
expression distribution
```

sin cambiar el significado semántico.

---

# 195. Large payloads

Valores LOB/binary no deberán copiarse innecesariamente durante construcción.

---

# 196. LOB semantics

Podrán utilizar:

```text
BinaryValue
LobValue
StreamBindingValue
```

según el Parameter/Type System.

---

# 197. Stream lifetime

El Builder no deberá asumir que un stream sigue válido indefinidamente.

El Binding System deberá modelar ownership/replayability.

---

# 198. Security

El sistema deberá proteger contra:

```text
SQL injection through values
identifier injection
raw expression abuse
secret leakage
parameter explosion
resource exhaustion
unsafe retries
unsafe upsert emulation
```

---

# 199. Value security

Valores normales:

```text
always parameterized by default
```

---

# 200. Identifier security

Target/column/constraint identifiers:

```text
structured
validated
never value-bound
```

---

# 201. Raw security

Raw APIs:

```text
explicit
policy-controlled
diagnostically visible
portability-aware
not trusted automatically
```

---

# 202. Parameter explosion protection

Un multi-row insert de:

```text
1,000,000 rows × 30 columns
```

no deberá crear ingenuamente 30 millones de placeholders en memoria.

---

# 203. Resource governance integration

El Builder/Planner podrá consultar límites declarativos mediante:

```text
QueryProcessingBudget
ResourceGovernancePolicy
PlatformCapabilities
DriverCapabilities
```

según la fase correspondiente.

---

# 204. Builder budget

Durante construcción puede existir un límite temprano:

```text
max materialized rows
max builder nodes
max columns
max local bindings
```

para evitar abuso antes del Planner.

---

# 205. Extensions

Una extensión podrá añadir:

```text
custom InsertSource
custom ConflictTarget
custom ConflictAction
custom ReturningMode
custom ValueExpression
custom InsertHint
```

---

# 206. Extension registry

Será:

```text
specialized
typed
frozen after bootstrap
deterministic
versioned
```

---

# 207. No arbitrary extension bag

Rechazado:

```php
$insert->extensions['whatever'] = $object;
```

---

# 208. Extension descriptor

Podrá declarar:

```text
extension id
version
node kinds
validation support
semantic support
capability requirements
planner support
compiler support
fingerprint impact
portability
diagnostic behavior
```

---

# 209. Extension completeness

Una extensión que añada un nuevo `InsertSource` deberá proporcionar las fases necesarias.

No podrá llegar al Compiler como nodo desconocido accidentalmente.

---

# 210. Testing strategy

La mayor parte del sistema se probará sin base de datos.

---

# 211. Unit tests

Cubrirán:

```text
target construction
single-row values
multi-row values
canonical column ordering
row-shape validation
NULL
DEFAULT
expressions
raw expressions
INSERT SELECT
conflict specification
upsert
returning
result expectations
metadata
parameter separation
copy isolation
finalization
budgets
extensions
```

---

# 212. Semantic integration tests

Cubrirán:

```text
unknown target
unknown column
target types
nullability
required columns
generated columns
identity columns
source arity
source types
conflict targets
constraint resolution
returning projections
capability requirements
```

---

# 213. Compiler tests

Separadamente comprobarán SQL para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 214. Driver conformance tests

Separadamente comprobarán:

```text
native binding
generated identifiers
affected rows
RETURNING result handling
error translation
LOB handling
```

---

# 215. Persistent runtime tests

Deberán ejecutar:

```text
Insert A
reset
Insert B
```

verificando ausencia de:

```text
parameter leakage
binding leakage
metadata leakage
target leakage
row leakage
conflict-state leakage
```

---

# 216. Concurrency tests

Dos inserts concurrentes deberán producir estados completamente independientes.

---

# 217. Property tests

Podrán verificar:

```text
deterministic canonical column ordering
row permutation normalization
binding/value separation
copy isolation
finalization immutability
stable parameter identities
fingerprint independence from runtime values
```

---

# 218. Architecture tests

`InsertQueryBuilder` no podrá importar:

```text
PDO
PDOStatement
NativeConnection
ConnectionLease
ConnectionPool
SQL Compiler implementation
Query Executor implementation
EntityManager
UnitOfWork
IdentityMap
Transaction implementation
HTTP Request
AuthenticatedUser
Tenant Entity
OpenTelemetry Span
```

---

# 219. Namespace recomendado

```text
VoltStack\Quantum\Database\Query\Builder\Insert
```

---

# 220. Estructura propuesta

```text
Query/
└── Builder/
    └── Insert/
        ├── Contract/
        │   ├── InsertQueryBuilderInterface.php
        │   └── InsertBuilderFinalizerInterface.php
        │
        ├── Core/
        │   ├── InsertQueryBuilder.php
        │   ├── InsertBuilderState.php
        │   ├── InsertBuilderFinalizer.php
        │   └── InsertBuilderSnapshot.php
        │
        ├── Target/
        │   ├── InsertTargetBuilder.php
        │   └── InsertTargetFactory.php
        │
        ├── Column/
        │   ├── InsertColumnList.php
        │   └── InsertColumnNormalizer.php
        │
        ├── Source/
        │   ├── InsertSource.php
        │   ├── ValuesInsertSource.php
        │   ├── QueryInsertSource.php
        │   ├── DefaultValuesInsertSource.php
        │   └── ExtensionInsertSource.php
        │
        ├── Values/
        │   ├── InsertRow.php
        │   ├── InsertRowSet.php
        │   ├── InsertRowBuilder.php
        │   ├── InsertRowNormalizer.php
        │   ├── InsertBatchShape.php
        │   └── DefaultValueExpression.php
        │
        ├── Conflict/
        │   ├── ConflictSpecification.php
        │   ├── ConflictAction.php
        │   ├── ConflictTarget.php
        │   ├── ColumnSetConflictTarget.php
        │   ├── ConstraintConflictTarget.php
        │   ├── ConflictAssignmentSet.php
        │   ├── ConflictBuilder.php
        │   └── IncomingValueReference.php
        │
        ├── Returning/
        │   ├── ReturningSpecification.php
        │   ├── ReturningBuilder.php
        │   └── InsertResultExpectation.php
        │
        ├── Parameter/
        │   └── InsertParameterCollector.php
        │
        ├── Metadata/
        │   └── InsertMetadataBuilder.php
        │
        ├── Bulk/
        │   ├── InsertBatchDescriptor.php
        │   └── InsertSourceReplayability.php
        │
        ├── Extension/
        │   ├── InsertBuilderExtension.php
        │   ├── InsertExtensionDescriptor.php
        │   └── InsertBuilderExtensionRegistry.php
        │
        ├── Diagnostic/
        │   └── InsertBuilderDiagnostic.php
        │
        └── Exception/
            ├── InsertQueryBuilderException.php
            ├── MissingInsertTargetException.php
            ├── MissingInsertSourceException.php
            ├── ConflictingInsertSourceException.php
            ├── InvalidInsertColumnException.php
            ├── DuplicateInsertColumnException.php
            ├── InvalidInsertRowException.php
            ├── InconsistentInsertRowShapeException.php
            ├── InvalidConflictSpecificationException.php
            ├── InvalidReturningSpecificationException.php
            └── InsertBuilderAlreadyFinalizedException.php
```

---

# 221. Ownership matrix

| Concepto | Owner |
|---|---|
| Fluent INSERT API | InsertQueryBuilder |
| Temporary construction state | InsertBuilderState |
| Target declaration | Insert Target Builder |
| Target column declaration | Insert Column System |
| Row construction | Insert Values System |
| Runtime values | BindingSet |
| Parameter identity | Parameter System |
| INSERT SELECT source | QueryInsertSource |
| Conflict intent | Conflict System |
| Returning intent | Returning System |
| Immutable query structure | InsertQueryModel / AST |
| Table resolution | Semantic Engine |
| Column resolution | Schema-Aware Resolution |
| Type inference | Query Type Inference |
| Required-column analysis | Semantic Engine |
| Constraint analysis | Query Constraint Analysis |
| Semantic lineage | Semantic Graph |
| Bulk strategy | Planner / Bulk Insert System |
| Conflict implementation strategy | Planner |
| SQL syntax | Compiler / Dialect |
| Native parameter binding | Driver |
| Generated-ID physical retrieval | Executor / Driver |
| Active transaction | TransactionManager |
| Physical connection | Connection System |
| ORM entity state | UnitOfWork / ORM |
| Telemetry execution state | Telemetry integration |

---

# 222. Architectural invariants

## DB-INS-001

`InsertQueryBuilder` nunca generará SQL.

## DB-INS-002

Nunca concatenará `INSERT INTO`.

## DB-INS-003

Nunca realizará identifier quoting.

## DB-INS-004

Nunca asignará native placeholders.

## DB-INS-005

Nunca realizará native binding.

## DB-INS-006

Nunca dependerá de PDO.

## DB-INS-007

Nunca adquirirá una conexión.

## DB-INS-008

Construir un INSERT no realizará I/O.

## DB-INS-009

El target será estructurado.

## DB-INS-010

Las columnas serán identifiers estructurados.

## DB-INS-011

Los valores normales serán parámetros.

## DB-INS-012

Los runtime values permanecerán fuera del AST.

## DB-INS-013

`NULL` será diferente de `DEFAULT`.

## DB-INS-014

`DEFAULT` será diferente de valor ausente.

## DB-INS-015

Multi-row insert tendrá shape explícito.

## DB-INS-016

El orden de associative arrays no definirá accidentalmente la semántica de columnas.

## DB-INS-017

Las filas multi-row deberán normalizarse a un column set canónico.

## DB-INS-018

Inconsistent row shapes no serán aceptados silenciosamente.

## DB-INS-019

Builder no resolverá tipos físicos.

## DB-INS-020

Builder no resolverá nullability de schema.

## DB-INS-021

Builder no descubrirá generated columns.

## DB-INS-022

Builder no descubrirá identity columns mediante I/O.

## DB-INS-023

Builder no consumirá sequences.

## DB-INS-024

INSERT SELECT utilizará un query artifact estructurado.

## DB-INS-025

Insert Builder no retendrá child Select Builders mutables.

## DB-INS-026

Source/target arity será resuelto semánticamente.

## DB-INS-027

Source/target type compatibility será resuelta semánticamente.

## DB-INS-028

Conflict handling será semántico.

## DB-INS-029

Conflict handling no contendrá vendor SQL en core.

## DB-INS-030

Upsert utilizará el mismo Insert Query Engine.

## DB-INS-031

Incoming values tendrán representación semántica.

## DB-INS-032

Incoming values no se codificarán como `EXCLUDED` o `VALUES()` en core.

## DB-INS-033

Conflict targets serán estructurados.

## DB-INS-034

Constraint identifiers serán estructurados.

## DB-INS-035

Builder no resolverá constraints físicos.

## DB-INS-036

Unsafe upsert emulation estará prohibida por defecto.

## DB-INS-037

Atomic conflict semantics deberán preservarse.

## DB-INS-038

RETURNING será una intención semántica.

## DB-INS-039

Builder no generará `RETURNING` syntax.

## DB-INS-040

`insertGetId()` no llamará native generated-id APIs.

## DB-INS-041

Generated-ID strategy pertenecerá a Planner/Executor/Driver.

## DB-INS-042

Result expectation será diferente de runtime result.

## DB-INS-043

Bulk strategy no pertenecerá al Builder.

## DB-INS-044

Builder no contendrá vendor parameter limits.

## DB-INS-045

Builder no contendrá vendor packet limits.

## DB-INS-046

Large inserts estarán sujetos a resource governance.

## DB-INS-047

Builder no materializará ilimitadamente streaming sources.

## DB-INS-048

Replayability será explícita cuando aplique.

## DB-INS-049

INSERT no será automáticamente retry-safe.

## DB-INS-050

Upsert no será automáticamente retry-safe.

## DB-INS-051

Volatile expressions deberán afectar retry/replayability analysis.

## DB-INS-052

Builder no iniciará transactions.

## DB-INS-053

Builder no almacenará active Transaction.

## DB-INS-054

Builder no seleccionará primary físicamente.

## DB-INS-055

WRITE intent será declarativo.

## DB-INS-056

Builder no almacenará ConnectionLease.

## DB-INS-057

Builder metadata no contendrá current User.

## DB-INS-058

Builder metadata no contendrá Tenant Entity.

## DB-INS-059

Builder metadata no contendrá active Span.

## DB-INS-060

Sensitive values serán redactados por defecto.

## DB-INS-061

Runtime values no serán telemetry dimensions automáticamente.

## DB-INS-062

ORM reutilizará el mismo Insert Query Engine.

## DB-INS-063

Active Record no tendrá un segundo INSERT compiler.

## DB-INS-064

EntityManager no tendrá un segundo INSERT compiler.

## DB-INS-065

Direct DB API funcionará sin ORM.

## DB-INS-066

Schema metadata será suministrada mediante SchemaView.

## DB-INS-067

Builder no realizará hidden schema introspection.

## DB-INS-068

Offline processing será soportado.

## DB-INS-069

Semantic Graph conservará data lineage.

## DB-INS-070

Capability requirements serán semánticos.

## DB-INS-071

No existirán vendor-name conditionals en Builder core.

## DB-INS-072

Dialect determinará sintaxis final.

## DB-INS-073

Platform determinará capabilities semánticas.

## DB-INS-074

Driver determinará native binding behavior.

## DB-INS-075

Validation estará dividida por fases.

## DB-INS-076

Builder validation no sustituirá Semantic Validation.

## DB-INS-077

Semantic Validation no sustituirá server constraints.

## DB-INS-078

Finalized InsertQueryModel será immutable.

## DB-INS-079

InsertBuilderState será operation-scoped.

## DB-INS-080

No existirá global current Insert Builder.

## DB-INS-081

Builder copies tendrán estado independiente.

## DB-INS-082

Parameter registries no serán compartidos entre builders.

## DB-INS-083

Bindings no serán compartidos accidentalmente entre queries.

## DB-INS-084

Query shape podrá ser independiente de runtime values.

## DB-INS-085

Structural fingerprints excluirán valores runtime.

## DB-INS-086

Binding shape podrá afectar physical compilation sin cambiar semántica.

## DB-INS-087

Raw SQL será explícito.

## DB-INS-088

Raw SQL será policy-controlled.

## DB-INS-089

Raw values podrán usar safe bindings.

## DB-INS-090

Extensions serán typed y registradas.

## DB-INS-091

Extension registries serán frozen después de bootstrap.

## DB-INS-092

Extensions deberán declarar impacto en fingerprints.

## DB-INS-093

Extensions no introducirán hidden I/O.

## DB-INS-094

Insert Builder será persistent-runtime safe.

## DB-INS-095

Insert Builder será coroutine-safe.

## DB-INS-096

No existirán static parameter counters compartidos.

## DB-INS-097

No habrá state leakage entre requests.

## DB-INS-098

No habrá state leakage entre queue jobs.

## DB-INS-099

No habrá state leakage entre coroutines.

## DB-INS-100

Builder no contendrá EntityManager.

## DB-INS-101

Builder no contendrá UnitOfWork.

## DB-INS-102

Builder no contendrá IdentityMap.

## DB-INS-103

Builder no contendrá native Statement.

## DB-INS-104

Builder no contendrá HTTP Request.

## DB-INS-105

Builder no contendrá Service Container como service locator.

## DB-INS-106

Builder no será responsable de Optimization.

## DB-INS-107

Builder no será responsable de Planning.

## DB-INS-108

Builder no será responsable de SQL Compilation.

## DB-INS-109

Builder no será responsable de Execution.

## DB-INS-110

Builder no será responsable de ORM Hydration.

## DB-INS-111

Builder no será responsable de Persistence UnitOfWork state.

## DB-INS-112

Convenience APIs convergerán al mismo canonical InsertQueryModel.

## DB-INS-113

`insert()` no constituirá un engine separado.

## DB-INS-114

`insertOrIgnore()` no constituirá un engine separado.

## DB-INS-115

`upsert()` no constituirá un engine separado.

## DB-INS-116

`insertGetId()` no constituirá un engine separado.

## DB-INS-117

`insertUsing()` no constituirá un engine separado.

## DB-INS-118

Builder podrá finalizarse sin ejecutar.

## DB-INS-119

Builder deberá poder probarse sin base de datos.

## DB-INS-120

La ergonomía Laravel-like no podrá romper las fronteras internas.

---

# 223. Anti-patterns

## 223.1 SQL concatenation

```php
$sql = 'INSERT INTO '.$table;
$sql .= ' (...) VALUES (...)';
```

**Rechazado.**

---

## 223.2 PDO inside Builder

```php
$this->pdo->prepare(...);
```

**Rechazado.**

---

## 223.3 Generated ID retrieval inside Builder

```php
return $pdo->lastInsertId();
```

**Rechazado.**

---

## 223.4 Vendor upsert branching

```php
if ($driver === 'pgsql') {
    // ON CONFLICT
} elseif ($driver === 'mysql') {
    // ON DUPLICATE KEY
}
```

**Rechazado.**

---

## 223.5 Missing equals NULL

```text
missing field
→ automatically insert NULL
```

**Rechazado.**

---

## 223.6 Huge placeholder expansion

```text
1,000,000 rows
→ build millions of placeholders immediately
```

**Rechazado.**

---

## 223.7 Hidden schema lookup

```php
$columns = $connection->describeTable($table);
```

durante `insert()`.

**Rechazado.**

---

## 223.8 Upsert through SELECT-before-write

```text
SELECT exists
   │
   ├── yes → UPDATE
   └── no  → INSERT
```

como emulación automática general.

**Rechazado** por race conditions.

---

## 223.9 Raw RETURNING

```php
$sql .= ' RETURNING id';
```

desde Builder.

**Rechazado.**

---

## 223.10 ORM-specific insert compiler

```text
ORM Insert Compiler
DB Builder Insert Compiler
```

como motores separados.

**Rechazado.**

---

# 224. Ejemplo completo — Insert simple

```php
$query = DB::insert()
    ->into('users')
    ->values([
        'name' => 'Ana',
        'email' => 'ana@example.com',
        'active' => true,
    ])
    ->returning('id', 'created_at');
```

Builder:

```text
InsertBuilderState
│
├── TARGET
│   └── users
│
├── COLUMNS
│   ├── name
│   ├── email
│   └── active
│
├── SOURCE
│   └── VALUES
│       └── Row
│           ├── P1
│           ├── P2
│           └── P3
│
└── RETURNING
    ├── id
    └── created_at
```

Bindings:

```text
P1 → "Ana"
P2 → "ana@example.com"
P3 → true
```

---

# 225. Semantic resolution del ejemplo

Schema:

```text
users
├── id
│   ├── UUID
│   ├── PRIMARY KEY
│   └── GENERATED
│
├── name
│   ├── String
│   └── NOT NULL
│
├── email
│   ├── EmailAddress
│   ├── NOT NULL
│   └── UNIQUE
│
├── active
│   ├── Boolean
│   └── NOT NULL
│
└── created_at
    ├── DateTime
    ├── NOT NULL
    └── DEFAULT
```

Semantic Analysis podrá resolver:

```text
P1
→ String

P2
→ DomainType<EmailAddress, String>

P3
→ Boolean

id
→ generated
→ valid RETURNING projection

created_at
→ default-generated value
→ valid RETURNING projection
```

---

# 226. Ejemplo completo — Upsert

```php
$query = DB::table('users')->upsert(
    values: [
        [
            'email' => 'ana@example.com',
            'name' => 'Ana',
        ],
        [
            'email' => 'carlos@example.com',
            'name' => 'Carlos',
        ],
    ],
    uniqueBy: ['email'],
    update: ['name'],
);
```

Representación:

```text
InsertQuery
│
├── TARGET
│   └── users
│
├── COLUMNS
│   ├── email
│   └── name
│
├── SOURCE
│   └── VALUES
│       ├── Row
│       │   ├── P1
│       │   └── P2
│       └── Row
│           ├── P3
│           └── P4
│
└── CONFLICT
    ├── TARGET
    │   └── email
    │
    └── ACTION
        └── UPDATE
            └── name
                └── IncomingValueReference(name)
```

Bindings:

```text
P1 → ana@example.com
P2 → Ana
P3 → carlos@example.com
P4 → Carlos
```

---

# 227. Semantic resolution del upsert

```text
email
   │
   ▼
users.email
   │
   ├── type: EmailAddress
   └── UNIQUE
           │
           ▼
     ConflictTarget
           │
           ▼
      valid semantic target
```

y:

```text
IncomingValueReference(name)
       │
       ▼
incoming users.name
       │
       ▼
String
       │
       ▼
assignment compatible with users.name
```

---

# 228. Ejemplo completo — INSERT SELECT

```php
$source = DB::table('legacy_users as l')
    ->select([
        'l.full_name',
        'l.email_address',
    ])
    ->where('l.migrated', false);

$query = DB::insert()
    ->into('users')
    ->columns('name', 'email')
    ->fromQuery($source)
    ->returning('id');
```

Representación:

```text
InsertQuery
│
├── Target
│   └── users
│
├── TargetColumns
│   ├── name
│   └── email
│
├── QueryInsertSource
│   └── SelectQuery
│       ├── FROM legacy_users AS l
│       ├── SELECT
│       │   ├── l.full_name
│       │   └── l.email_address
│       └── WHERE
│           └── l.migrated = P1
│
└── Returning
    └── id
```

Semantic lineage:

```text
legacy_users.full_name
          │
          ▼
      users.name

legacy_users.email_address
          │
          ▼
      users.email
```

---

# 229. Pipeline completo

```text
Developer
   │
   ▼
InsertQueryBuilder
   │
   ├── TargetBuilder
   ├── RowBuilder
   ├── ExpressionBuilder
   ├── ConflictBuilder
   ├── ReturningBuilder
   ├── ParameterRegistry
   └── MetadataBuilder
          │
          ▼
InsertBuilderFinalizer
          │
          ├────────────────┐
          ▼                ▼
 InsertQueryModel       BindingSet
          │
          ▼
   InsertQueryNode
          │
          ▼
    Normalization
          │
          ▼
      Validation
          │
          ▼
   Semantic Analysis
          │
          ├── target resolution
          ├── column resolution
          ├── type inference
          ├── nullability
          ├── generated/default analysis
          ├── source compatibility
          ├── conflict resolution
          ├── constraints
          ├── capabilities
          └── semantic graph
          │
          ▼
 SemanticQueryArtifact
          │
          ▼
      Optimizer
          │
          ▼
       Planner
          │
          ├── bulk strategy
          ├── conflict strategy
          ├── returning strategy
          ├── generated-id strategy
          └── binding shape
          │
          ▼
       Compiler
          │
          ▼
    CompiledQuery
          │
          ▼
       Executor
          │
          ▼
InsertExecutionResult
```

---

# 230. Fórmula maestra del Builder

```text
InsertQueryBuilder
=
Target
+
Columns
+
InsertSource
+
ConflictIntent
+
ReturningIntent
+
ResultExpectation
+
ParameterDefinitions
+
QueryMetadata
```

---

# 231. Fórmula del Insert Query Artifact

```text
InsertQueryArtifact
=
Immutable InsertQueryModel
+
ParameterDefinitionSet
+
QueryMetadata
```

---

# 232. Fórmula runtime

```text
InsertExecutionRequest
=
InsertQueryArtifact
+
BindingSet
+
Execution Requirements
```

---

# 233. Fórmula semántica

```text
Insert Meaning
=
Resolved Target
+
Resolved Target Columns
+
Resolved Source
+
Assignment Compatibility
+
Defaults
+
Generated-Value Rules
+
Constraint Semantics
+
Conflict Semantics
+
Returning Semantics
+
Capability Requirements
+
Data Lineage
```

---

# 234. Fórmula de bulk planning

```text
Bulk Insert Strategy
=
Semantic Insert
+
Row Shape
+
Binding Shape
+
Row Cardinality
+
Platform Capabilities
+
Driver Capabilities
+
Parameter Limits
+
Resource Policy
+
Atomicity Requirement
+
Conflict Semantics
+
Returning Requirements
```

---

# 235. Fórmula de seguridad

```text
Safe Insert
=
Structured Target
+
Structured Columns
+
Structured Expressions
+
Parameterized Runtime Values
+
Typed Bindings
+
Explicit Raw Escape Hatches
+
Sensitive-Value Redaction
+
Resource Governance
+
Explicit Retry Semantics
```

---

# 236. Fórmula de portabilidad

```text
Portable Insert
=
Semantic Insert Model
-
Vendor SQL Assumptions
+
Capability Requirements
+
Platform Type Mapping
+
Planner Strategies
+
Dialect Compilation
```

---

# 237. Diseño definitivo

VoltStack combinará:

```text
Laravel-like INSERT DX
           +
Structured Insert Model
           +
Query AST
           +
Schema-Aware Semantic Analysis
           +
Query Type System
           +
Constraint Analysis
           +
Capability Model
           +
Planner
           +
Dialect Compiler
```

permitiendo:

```php
DB::table('users')->insert([
    'name' => 'Ana',
    'email' => 'ana@example.com',
]);
```

sin convertir el Query Builder en un SQL builder.

Internamente:

```text
Application Data
      │
      ▼
InsertQueryBuilder
      │
      ├───────────────┐
      ▼               ▼
InsertQueryModel   BindingSet
      │
      ▼
Insert AST
      │
      ▼
Semantic Query Engine
      │
      ▼
Semantic Insert
      │
      ▼
Planner
      │
      ▼
Target Compiler
      │
      ▼
Compiled INSERT
      │
      ▼
Executor
```

---

# 238. Conclusión

`InsertQueryBuilder` será la frontera pública entre los datos que la aplicación desea insertar y el Query Engine interno de VoltStack.

El desarrollador podrá utilizar APIs familiares:

```php
DB::table('users')->insert(...);

DB::table('users')->insertOrIgnore(...);

DB::table('users')->insertGetId(...);

DB::table('users')->upsert(...);

DB::table('archive')->insertUsing(...);
```

pero todas convergerán hacia:

```text
Canonical InsertQueryModel
```

La arquitectura mantendrá separadas:

```text
construction
semantics
planning
compilation
binding
execution
persistence
```

La regla central será:

> **`InsertQueryBuilder` expresa qué datos deben insertarse; Semantic Analysis determina qué significan respecto al schema; Planner decide cómo realizar la operación; Compiler genera la representación SQL y Executor realiza el trabajo físico.**

Por tanto:

```text
InsertQueryBuilder
      │
      ▼
InsertQueryModel
      │
      ▼
InsertQueryNode
```

y nunca:

```text
InsertQueryBuilder
      │
      X
      ▼
SQL String
```

Esta separación permitirá que VoltStack soporte de manera coherente:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

además de futuras plataformas, bulk inserts, upserts, generated identifiers, `RETURNING`, ORM persistence, offline compilation y runtimes persistentes sin introducir acoplamiento entre el Builder y la infraestructura física.

---

# 239. Siguiente documento

```text
46_DATABASE_UPDATE_QUERY_BUILDER.md
```

El siguiente documento definirá:

```text
UPDATE target
assignment system
single/multiple assignments
expression assignments
NULL/DEFAULT assignments
WHERE integration
JOIN-aware updates
subquery assignments
UPDATE ... FROM semantics
RETURNING
limits/order portability
optimistic locking integration
parameterization
type compatibility
constraint implications
bulk updates
safety against unbounded updates
ORM/Persistence integration
persistent-runtime safety
```

manteniendo:

```text
UpdateQueryBuilder
      │
      ▼
UpdateQueryModel
      │
      ▼
UpdateQueryNode
      │
      ▼
Semantic Analysis
      │
      ▼
Optimizer
      │
      ▼
Planner
      │
      ▼
Compiler
      │
      ▼
Executor
```