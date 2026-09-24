# 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md

# VoltStack Quantum Database
## Full-Text Search System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 272 — Full-Text Search System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `271_DATABASE_DATA_ARCHIVAL_SYSTEM.md`  
**Siguiente documento:** `273_DATABASE_JSON_QUERY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Full-Text Search System** de VoltStack Database.

El sistema proporcionará una abstracción semántica, portable y extensible para realizar búsquedas textuales avanzadas sobre los motores de base de datos soportados por VoltStack:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin exponer directamente sus dialectos específicos a:

```text
ORM
Repository
Model API
Query Builder
Application
```

La regla central será:

> **Full-Text Search será una extensión semántica del Query Engine; el desarrollador expresa intención de búsqueda y la plataforma determina cómo representarla utilizando las capacidades reales del motor de base de datos.**

Por tanto:

```text
Search Intent
      ↓
Search Model / AST
      ↓
Semantic Analysis
      ↓
Capability Resolution
      ↓
Search Planning
      ↓
Platform Compiler
      ↓
Database-native Search
```

y nunca:

```text
ORM
 ↓
vendor-specific SQL strings
```

---

# 2. Objetivos

El sistema deberá proporcionar:

- API portable de búsqueda textual;
- Search AST;
- Search Expressions;
- Search Predicates;
- Search Documents;
- Search Fields;
- búsqueda por términos;
- búsqueda por frases;
- búsqueda booleana;
- prefijos cuando la plataforma los soporte;
- ranking;
- relevance score;
- highlighting cuando sea soportado;
- normalización;
- tokenización;
- configuración lingüística;
- stemming;
- stop words;
- pesos por campo;
- búsqueda multi-columna;
- índices full-text;
- metadata;
- integración ORM;
- integración Query Builder;
- integración con Pagination;
- integración con Cursor Pagination;
- integración con Multitenancy;
- integración con Sharding;
- telemetría;
- seguridad;
- optimización;
- extensibilidad.

---

# 3. No objetivos

El sistema no será inicialmente:

```text
Elasticsearch
OpenSearch
Solr
vector database
semantic search engine
embedding engine
web search engine
```

Tampoco sustituirá:

```text
LIKE
exact comparisons
JSON queries
geographic queries
vector similarity search
```

cuando esas operaciones sean semánticamente más apropiadas.

---

# 4. Distinciones fundamentales

VoltStack distinguirá:

```text
Full-Text Search
≠ LIKE
≠ Regex
≠ Exact Match
≠ Prefix Filter
≠ Vector Search
≠ Semantic Search
≠ Search Engine
≠ Search Index
≠ Database Index
```

---

# 5. Full-Text Search ≠ LIKE

Una consulta:

```sql
WHERE title LIKE '%database%'
```

no representa necesariamente búsqueda full-text.

Full-text puede incluir:

```text
tokenization
stemming
ranking
language processing
stop words
phrase semantics
document frequency
```

---

# 6. Full-Text Search ≠ Semantic Search

Buscar:

```text
"automobile"
```

y encontrar:

```text
"car"
```

por embeddings o similitud vectorial pertenece conceptualmente a otro sistema.

Full-text puede aplicar:

```text
stemming
lexical normalization
dictionary processing
```

pero no implica embeddings.

---

# 7. Full-Text Search ≠ Search Engine externo

La arquitectura definida aquí utiliza principalmente capacidades de búsqueda proporcionadas por:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

Adapters externos podrán incorporarse posteriormente mediante extensiones.

---

# 8. Arquitectura general

```text
Application
    │
    ▼
Search API
    │
    ▼
Search Expression
    │
    ▼
Query AST
    │
    ▼
Semantic Analysis
    │
    ▼
Search Metadata
    │
    ▼
Search Planner
    │
    ▼
Capability Resolver
    │
    ▼
Logical Search Plan
    │
    ▼
Platform Compiler
    │
    ▼
Native Full-Text Search
```

---

# 9. Integración con Query Engine

Full-Text Search no tendrá un segundo Query Engine.

Utilizará:

```text
DATABASE_QUERY_ARCHITECTURE
DATABASE_QUERY_MODEL
DATABASE_QUERY_AST_SYSTEM
DATABASE_QUERY_EXPRESSION_SYSTEM
DATABASE_QUERY_PREDICATE_SYSTEM
DATABASE_QUERY_TYPE_SYSTEM
DATABASE_SEMANTIC_QUERY_ARCHITECTURE
DATABASE_QUERY_OPTIMIZER_ARCHITECTURE
DATABASE_QUERY_PLANNER_ARCHITECTURE
DATABASE_SQL_COMPILER_ARCHITECTURE
```

---

# 10. Principio de integración

```text
FullTextSearch
      ↓
Query AST extension
      ↓
normal Query Engine pipeline
```

No:

```text
FullTextSearch
      ↓
custom SQL executor
```

---

# 11. Search Document

La unidad lógica principal será:

```text
SearchDocument
```

Representa el conjunto de información textual sobre la cual se realiza una búsqueda.

---

# 12. SearchDocument ≠ ORM Entity

Un SearchDocument puede estar formado por:

```text
one column
multiple columns
computed expressions
joined data
derived search representation
```

---

# 13. Ejemplo

Una entidad:

```text
Article
```

puede definir un documento:

```text
title
summary
body
```

---

# 14. Search Document conceptual

```php
final readonly class SearchDocument
{
    public function __construct(
        public SearchDocumentId $id,
        public array $fields,
        public ?SearchLanguage $language = null,
    ) {}
}
```

---

# 15. Search Field

Cada campo será representado por:

```text
SearchField
```

---

# 16. SearchField

Podrá contener:

```text
field/expression
weight
language
normalization policy
index metadata
```

---

# 17. Ejemplo

```php
new SearchDocument(
    id: new SearchDocumentId('article_search'),
    fields: [
        new SearchField('title', weight: 4),
        new SearchField('summary', weight: 2),
        new SearchField('body', weight: 1),
    ],
);
```

---

# 18. Weight

Los pesos permiten expresar:

```text
title > summary > body
```

en términos de importancia relativa.

---

# 19. Weight ≠ relevance score

Weight es una entrada de configuración.

Relevance score es un resultado calculado.

---

# 20. Search Query

La intención del usuario será representada mediante:

```text
SearchQuery
```

---

# 21. Ejemplo

```php
$search = SearchQuery::terms('distributed database');
```

---

# 22. Search Query ≠ SQL query

Es una representación semántica de búsqueda.

---

# 23. Search Expression

El Query AST incorporará:

```text
SearchExpression
```

---

# 24. Jerarquía conceptual

```text
ExpressionNode
└── SearchExpression
    ├── SearchTermsExpression
    ├── SearchPhraseExpression
    ├── SearchBooleanExpression
    ├── SearchPrefixExpression
    └── SearchCustomExpression
```

---

# 25. Search Predicate

Para filtros:

```text
SearchPredicate
```

---

# 26. Ejemplo conceptual

```php
Article::query()
    ->whereFullText(
        ['title', 'body'],
        'distributed databases'
    );
```

---

# 27. Resultado

El Query Builder producirá:

```text
SearchPredicate AST
```

No:

```text
MATCH(...) AGAINST(...)
```

---

# 28. API Laravel-like

VoltStack podrá ofrecer:

```php
Article::query()
    ->whereFullText(
        ['title', 'summary', 'body'],
        'database architecture'
    )
    ->get();
```

---

# 29. Repository API

También:

```php
$articles = $repository
    ->query()
    ->whereFullText(
        fields: ['title', 'body'],
        query: 'database architecture',
    )
    ->get();
```

Ambas APIs convergen al mismo Query Engine.

---

# 30. Search Builder

Para casos avanzados podrá existir:

```php
$search = Search::query()
    ->terms('distributed database')
    ->language('english')
    ->mode(SearchMode::NATURAL)
    ->minimumScore(0.45);
```

---

# 31. SearchMode

Conceptualmente:

```php
enum SearchMode
{
    case NATURAL;
    case BOOLEAN;
    case PHRASE;
    case PREFIX;
    case CUSTOM;
}
```

---

# 32. NATURAL

Búsqueda natural de términos.

---

# 33. BOOLEAN

Permite expresar combinaciones lógicas.

Ejemplo conceptual:

```text
database AND distributed
```

---

# 34. PHRASE

Busca secuencias textuales.

Ejemplo:

```text
"distributed database"
```

---

# 35. PREFIX

Búsqueda por prefijo cuando sea compatible.

Ejemplo:

```text
distrib*
```

---

# 36. SearchMode ≠ vendor syntax

El usuario no deberá escribir directamente:

```text
+database -mysql
```

como requisito de la API portable.

Ese tipo de sintaxis puede existir mediante un escape hatch explícito.

---

# 37. Boolean AST

La búsqueda booleana deberá representarse estructuralmente.

```text
SearchBooleanExpression
├── AND
├── OR
└── NOT
```

---

# 38. Ejemplo

```text
        AND
       /   \
 database   OR
           /  \
 distributed replicated
```

---

# 39. Ventaja

El compiler podrá convertir la misma intención a:

```text
MySQL syntax
PostgreSQL tsquery
SQLite FTS syntax
```

---

# 40. Search Term

Unidad léxica:

```text
SearchTerm
```

---

# 41. Search Phrase

Secuencia:

```text
SearchPhrase
```

---

# 42. Search term ≠ raw SQL fragment

---

# 43. User Search Input

Texto externo deberá pasar por:

```text
SearchInputParser
```

cuando se utilice una sintaxis de usuario.

---

# 44. SearchInputParser ≠ SQL parser

Interpreta únicamente el lenguaje de búsqueda permitido.

---

# 45. Ejemplo

Entrada:

```text
database AND "query optimizer"
```

Resultado:

```text
SearchBooleanExpression
```

---

# 46. Parser security

Nunca:

```php
$sql .= $userSearch;
```

---

# 47. Search Input Policy

Deberá definir:

```text
max length
max terms
max nesting depth
allowed operators
wildcard policy
phrase limits
```

---

# 48. Resource protection

Un search expression excesivamente complejo deberá poder rechazarse antes de llegar a la base de datos.

---

# 49. Search Language

La configuración lingüística será explícita.

```php
final readonly class SearchLanguage
{
    public function __construct(
        public string $id,
    ) {}
}
```

Ejemplos:

```text
english
spanish
simple
```

---

# 50. Language ≠ locale

```text
SearchLanguage
≠
Locale
```

Aunque puedan estar relacionados.

---

# 51. Language resolution

Podrá provenir de:

```text
explicit query
search document metadata
entity metadata
application configuration
platform default
```

---

# 52. No hidden language assumptions

Si la semántica depende fuertemente del idioma, deberá ser observable.

---

# 53. Tokenization

Conceptualmente:

```text
Text
 ↓
Tokenizer
 ↓
Tokens
```

---

# 54. Example

```text
"Databases are distributed"
```

podría producir:

```text
database
distributed
```

dependiendo de la plataforma/configuración.

---

# 55. Tokenization is platform-dependent

VoltStack no intentará fingir que todos los motores tokenizan idénticamente.

---

# 56. Normalization

Puede incluir:

```text
case normalization
accent normalization
unicode normalization
punctuation handling
```

según capacidades.

---

# 57. Normalization ≠ application mutation

El texto almacenado no necesita modificarse.

---

# 58. Stemming

Ejemplo conceptual:

```text
running
runs
ran
```

pueden relacionarse con un lexema común dependiendo del motor y lenguaje.

---

# 59. Stemming ≠ guaranteed portability

VoltStack deberá exponer diferencias de capacidad.

---

# 60. Stop Words

Palabras como:

```text
the
a
of
```

pueden ser ignoradas por ciertos motores/configuraciones.

---

# 61. Stop Words affect semantics

Una búsqueda no debe considerarse portable únicamente porque compile en todos los motores.

---

# 62. Search Semantic Profile

Para describir expectativas:

```php
enum SearchSemanticProfile
{
    case PORTABLE;
    case PLATFORM_NATIVE;
    case STRICT;
    case CUSTOM;
}
```

---

# 63. PORTABLE

VoltStack utiliza el subconjunto semántico común disponible.

---

# 64. PLATFORM_NATIVE

Permite aprovechar capacidades avanzadas del motor.

---

# 65. STRICT

Si una capacidad requerida no puede preservarse:

```text
fail
```

en lugar de degradarla.

---

# 66. No silent semantic degradation

Regla fundamental:

> Si una operación full-text no puede conservar su significado en la plataforma activa, VoltStack no deberá convertirla silenciosamente en otra búsqueda diferente.

---

# 67. Capability model

Full-text se integrará con:

```text
DATABASE_PLATFORM_CAPABILITY_SYSTEM
```

---

# 68. Search capabilities

Ejemplos:

```text
supportsFullTextSearch()
supportsFullTextIndex()
supportsBooleanSearch()
supportsPhraseSearch()
supportsPrefixSearch()
supportsSearchRanking()
supportsSearchHighlighting()
supportsLanguageConfiguration()
supportsWeightedSearchFields()
supportsSearchQueryNormalization()
```

---

# 69. Capability ≠ version

```text
DatabaseVersion
≠
SearchCapability
```

---

# 70. Capability result

Podrá ser:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 71. UNKNOWN ≠ SUPPORTED

---

# 72. MySQL

MySQL podrá utilizar capacidades como:

```text
FULLTEXT indexes
MATCH(...)
AGAINST(...)
natural language mode
boolean mode
query expansion where explicitly requested
```

mediante su compiler.

---

# 73. MariaDB

MariaDB tendrá su propio:

```text
MariaDBSearchCompiler
```

aunque comparta parte de la implementación con MySQL.

---

# 74. MySQL ≠ MariaDB

VoltStack continuará respetando la decisión arquitectónica:

```text
MySQL Platform
≠
MariaDB Platform
```

---

# 75. PostgreSQL

PostgreSQL podrá utilizar:

```text
tsvector
tsquery
plainto_tsquery
phraseto_tsquery
websearch_to_tsquery
to_tsvector
ts_rank
ts_rank_cd
```

cuando las capabilities efectivas lo permitan.

---

# 76. SQLite

SQLite podrá utilizar:

```text
FTS5
```

cuando la instalación activa disponga de esa capability.

---

# 77. SQLite capability discovery

No deberá asumirse:

```text
SQLite → FTS5 always available
```

---

# 78. Search Compiler

La extensión del compiler será:

```text
SearchCompiler
```

---

# 79. Platform compilers

```text
SearchCompiler
├── MySQLSearchCompiler
├── MariaDBSearchCompiler
├── PostgreSQLSearchCompiler
└── SQLiteSearchCompiler
```

---

# 80. Search Compiler ≠ SQL Compiler

Conceptualmente será una extensión/componente del SQL Compiler, no un pipeline SQL independiente.

---

# 81. Compilation

```text
Search AST
   ↓
Platform Search Compiler
   ↓
SQL AST / SQL Fragment Representation
   ↓
SQL Compiler
```

---

# 82. Search parameters

Los términos deberán permanecer parametrizados siempre que la plataforma lo permita.

---

# 83. Search syntax values

Incluso cuando el motor requiera una mini-sintaxis textual, ésta deberá ser construida por un compiler seguro desde AST validado.

---

# 84. No user syntax passthrough by default

---

# 85. Search Index

VoltStack representará índices de búsqueda mediante:

```text
FullTextIndexDefinition
```

---

# 86. Schema Builder

Ejemplo:

```php
$table->fullText(
    ['title', 'summary', 'body'],
    name: 'articles_search'
);
```

---

# 87. Search Index ≠ ordinary index

Aunque ambos formen parte del Schema System.

---

# 88. Schema AST

La definición producirá:

```text
CreateFullTextIndexOperation
```

o equivalente.

---

# 89. Platform compilation

```text
Schema AST
    ↓
Platform Capability
    ↓
Schema Compiler
```

---

# 90. No vendor DDL in Model metadata

---

# 91. Search Index Metadata

Podrá contener:

```text
index name
fields
language/configuration
weights
platform options
capabilities
```

---

# 92. Search metadata ≠ physical index

---

# 93. ORM metadata

Una entidad podrá declarar:

```php
#[SearchDocument(
    name: 'article_search',
    fields: [
        'title',
        'summary',
        'body',
    ]
)]
final class Article
{
}
```

---

# 94. Attribute metadata

También podrán declararse pesos.

Ejemplo conceptual:

```php
#[SearchField(weight: 4)]
private string $title;

#[SearchField(weight: 1)]
private string $body;
```

---

# 95. Mapping ≠ schema creation

Declarar metadata ORM no significa crear automáticamente el índice durante runtime.

Schema/Migrations seguirán gobernando evolución física.

---

# 96. SearchDocument metadata

Podrá compilarse junto con ORM metadata.

---

# 97. Metadata cache

Podrá almacenar:

```text
compiled SearchDocument definitions
field mappings
weights
language mappings
```

---

# 98. Ranking

Una búsqueda podrá producir:

```text
SearchScore
```

---

# 99. SearchScore

```php
final readonly class SearchScore
{
    public function __construct(
        public float $value,
    ) {}
}
```

---

# 100. Score ≠ universal probability

Un score:

```text
0.82
```

no significa necesariamente:

```text
82% relevant
```

---

# 101. Scores are platform-dependent

No deberán compararse ingenuamente entre motores distintos.

---

# 102. Search score expression

Query AST podrá incluir:

```text
SearchScoreExpression
```

---

# 103. Example

```php
Article::query()
    ->withSearchScore(
        ['title', 'body'],
        'database architecture',
        alias: 'relevance'
    )
    ->orderByDesc('relevance')
    ->get();
```

---

# 104. Logical projection

El resultado podrá incluir score sin convertirlo en propiedad persistente de la entidad.

---

# 105. Entity ≠ score container

Podrán utilizarse:

```text
projection
tuple
DTO
result metadata
```

---

# 106. Search Result

Conceptualmente:

```php
final readonly class SearchResult
{
    public function __construct(
        public mixed $item,
        public ?SearchScore $score,
        public ?SearchHighlight $highlight,
    ) {}
}
```

---

# 107. SearchResult ≠ managed entity

Puede contener una entidad managed, pero el wrapper no lo es.

---

# 108. Ranking Strategy

```php
enum SearchRankingStrategy
{
    case DEFAULT;
    case FREQUENCY;
    case COVER_DENSITY;
    case WEIGHTED;
    case CUSTOM;
}
```

Las opciones efectivas dependerán de capabilities.

---

# 109. Platform ranking

MySQL y PostgreSQL pueden producir escalas diferentes.

VoltStack preservará intención, no fingirá equivalencia matemática exacta.

---

# 110. Weighted fields

Ejemplo:

```text
title = 4
summary = 2
body = 1
```

---

# 111. Weighted Search Plan

El planner podrá traducir esos pesos a capacidades nativas cuando sea posible.

---

# 112. Unsupported weighting

Si la plataforma no soporta la semántica solicitada:

```text
STRICT → fail

PORTABLE → use supported portable behavior if semantically valid

PLATFORM_NATIVE → platform rules
```

---

# 113. Highlighting

El sistema podrá representar:

```text
SearchHighlightExpression
```

---

# 114. Highlighting ≠ search matching

Es una presentación derivada del resultado.

---

# 115. Example

Resultado conceptual:

```text
VoltStack introduces a <mark>database query optimizer</mark>...
```

---

# 116. HTML safety

El Database layer no deberá asumir que el highlight será HTML seguro.

---

# 117. Highlight representation

Preferentemente podrá producir:

```text
fragments
offsets
matched ranges
```

antes que HTML arbitrario.

---

# 118. SearchHighlight

```php
final readonly class SearchHighlight
{
    public function __construct(
        public array $fragments,
    ) {}
}
```

---

# 119. Highlight rendering

Pertenece a capas superiores.

---

# 120. Query Builder integration

Ejemplo básico:

```php
Article::query()
    ->whereFullText(
        ['title', 'body'],
        'distributed systems'
    )
    ->get();
```

---

# 121. Advanced example

```php
Article::query()
    ->whereFullText(
        document: 'article_search',
        search: Search::query()
            ->phrase('distributed systems')
            ->or()
            ->term('database')
    )
    ->get();
```

---

# 122. Search document name

Permite evitar repetir campos:

```php
Article::query()
    ->search(
        document: 'article_search',
        query: 'distributed database'
    );
```

---

# 123. Repository example

```php
$repository->search(
    document: 'article_search',
    query: SearchQuery::terms('database engine'),
);
```

---

# 124. Unified engine

```text
Model API
Repository
Entity Query
Query Builder
      │
      ▼
Search AST
      │
      ▼
Same Search Engine
```

---

# 125. Search Planner

Responsable de:

```text
validate search document
resolve fields
resolve language
resolve capabilities
determine index strategy
determine ranking
determine physical expressions
validate ordering
validate pagination
```

---

# 126. Planner input

```text
Search AST
Query Context
Metadata
Platform Capabilities
Schema Metadata
Tenant Context
Distribution Context
Resource Policy
```

---

# 127. Planner output

```text
SearchExecutionPlan
```

---

# 128. Planner ≠ Executor

No ejecutará DB I/O.

---

# 129. Search semantic analysis

Semantic Analyzer validará:

```text
field exists
field is searchable
field type compatible
document exists
language valid
operator supported
ranking expression valid
```

---

# 130. String type ≠ automatically searchable

Un campo `string` no implica que exista índice full-text.

---

# 131. Searchable ≠ indexed

Puede ser semánticamente searchable pero operacionalmente costoso/no soportado sin índice.

---

# 132. Search Index Requirement

La policy podrá ser:

```php
enum SearchIndexRequirement
{
    case REQUIRED;
    case PREFERRED;
    case OPTIONAL;
}
```

---

# 133. REQUIRED

Si falta índice:

```text
planning error
```

---

# 134. PREFERRED

Podrá:

```text
warn
```

si la plataforma permite búsqueda sin índice.

---

# 135. OPTIONAL

Se permite estrategia válida disponible.

---

# 136. No LIKE fallback by default

Regla crítica:

> VoltStack no convertirá silenciosamente una búsqueda full-text no soportada en `LIKE '%query%'`.

---

# 137. Why

Porque:

```text
FullText("distributed database")
```

y:

```text
LIKE '%distributed database%'
```

no tienen la misma semántica.

---

# 138. Explicit fallback

Si una aplicación desea fallback podrá configurarlo explícitamente:

```text
SearchFallbackPolicy
```

---

# 139. Fallback policies

```php
enum SearchFallbackPolicy
{
    case FAIL;
    case APPLICATION_DEFINED;
    case CUSTOM;
}
```

No existirá `LIKE` implícito como garantía portable.

---

# 140. Query Optimizer

El optimizer podrá analizar:

```text
search predicate duplication
search expression reuse
ranking reuse
unnecessary score calculation
limit pushdown
search index availability
```

---

# 141. Duplicate search expressions

Ejemplo:

```text
WHERE search(Q)
ORDER BY score(Q)
SELECT score(Q)
```

podrá compartir representación lógica cuando sea seguro.

---

# 142. Compiler optimization

La plataforma podrá evitar recalcular expresiones cuando disponga de una estrategia equivalente.

---

# 143. Search predicate ordering

El optimizer podrá combinar:

```text
tenant predicate
status predicate
date predicate
full-text predicate
```

sin alterar semántica.

---

# 144. Search + regular predicates

Ejemplo:

```php
Article::query()
    ->where('published', true)
    ->whereFullText(
        ['title', 'body'],
        'database'
    )
    ->get();
```

---

# 145. Authorization first

El Data Access Security System deberá aplicar scope autorizado antes de exponer resultados.

---

# 146. Search cannot bypass authorization

```text
Search Index
≠
Authorization Boundary
```

---

# 147. Search information leakage

Ranking, count y snippets podrían revelar existencia de datos no autorizados.

---

# 148. Therefore

Authorization predicates deberán formar parte de la consulta efectiva antes de:

```text
ranking
count
pagination
highlight
```

cuando sea necesario.

---

# 149. Search + Pagination

Full-text deberá integrarse con:

```text
199_DATABASE_PAGINATION_SYSTEM.md
```

---

# 150. Example

```php
Article::query()
    ->search('database architecture')
    ->orderBySearchScore()
    ->paginate(20);
```

---

# 151. Count query

Count Planner deberá eliminar score/highlight innecesarios cuando sea semánticamente seguro.

---

# 152. Search count ≠ ranking

Para contar:

```text
matching documents
```

no es necesario calcular score si no afecta elegibilidad.

---

# 153. Pagination order

Orden típico:

```text
score DESC
```

puede requerir tie-breaker.

---

# 154. Deterministic search order

Ejemplo:

```text
score DESC
published_at DESC
id ASC
```

---

# 155. Score ties

Múltiples documentos pueden compartir score.

Por ello:

```text
ORDER BY score
```

no siempre es determinista.

---

# 156. Search + Cursor Pagination

Se integrará con:

```text
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

---

# 157. Cursor complexity

Un cursor podría necesitar:

```text
score
+
tie-breaker
```

---

# 158. Example boundary

```text
(score=0.8234, id=891)
```

---

# 159. Score stability problem

El score puede cambiar cuando:

```text
documents change
index changes
statistics change
search configuration changes
```

---

# 160. Cursor eligibility

Search Cursor Pagination deberá evaluar:

```text
score determinism
score stability
ordering completeness
platform semantics
```

---

# 161. Search cursor ≠ always safe

Puede ser:

```text
ELIGIBLE
ELIGIBLE_WITH_LIMITATIONS
UNSTABLE
UNSUPPORTED
UNKNOWN
```

---

# 162. Query fingerprint

Cursor deberá vincularse a:

```text
search query
search document
language
ranking configuration
tenant/security scope
```

---

# 163. Cursor tampering

Sigue aplicando el modelo firmado de Cursor Pagination.

---

# 164. Search + Chunk Processing

Podrá utilizarse para:

```text
process all documents matching search
```

pero deberán respetarse las advertencias de orden/mutación.

---

# 165. Search score ordering for chunks

No será la opción preferida si el score puede cambiar durante traversal.

---

# 166. Stable identifier ordering

Para procesamiento exhaustivo puede ser preferible:

```text
search predicate
+
ORDER BY stable identifier
```

---

# 167. Search + Lazy Collection

Podrá existir:

```php
Article::query()
    ->search('database')
    ->lazy();
```

La Lazy Collection conservará las reglas del documento 202.

---

# 168. Search + Result Cache

Podrá cachearse si:

```text
query fingerprint
search configuration
tenant
security scope
index generation
```

forman parte de la identidad apropiada.

---

# 169. Search index generation

Podrá formar parte de metadata/cache invalidation cuando sea necesario.

---

# 170. Search result cache ≠ search index

---

# 171. Index freshness

Para DB-native search normalmente las garantías dependerán del propio motor.

Adapters externos futuros podrían tener:

```text
index lag
```

---

# 172. Index freshness model

La arquitectura deberá permitir:

```text
CURRENT
STALE
UNKNOWN
```

aunque la primera implementación DB-native pueda simplificarlo.

---

# 173. UNKNOWN freshness ≠ CURRENT

---

# 174. Search + Transactions

Una búsqueda dentro de transaction seguirá las reglas normales de:

```text
Transaction Context
Read/Write Routing
Isolation
```

---

# 175. Full-text does not open transactions

---

# 176. Search + replicas

Una búsqueda read-only podrá utilizar replica sólo cuando:

```text
routing policy
consistency policy
replica eligibility
```

lo permitan.

---

# 177. Search freshness

Un índice en una replica puede reflejar únicamente los datos que esa replica ya recibió.

---

# 178. Read-your-writes

Después de modificar un documento y buscarlo inmediatamente:

```text
sticky writer
```

puede ser necesario.

---

# 179. Search + Multitenancy

El Tenant Query Context deberá aplicarse antes de ejecutar la búsqueda.

---

# 180. Example

```text
tenant_id = 42
AND
full_text_match(...)
```

conceptualmente.

---

# 181. Tenant isolation

Un search document no podrá omitir accidentalmente el tenant predicate.

---

# 182. Search index ≠ tenant isolation

Incluso si varios tenants comparten índice.

---

# 183. Tenant-specific configuration

Podrá permitirse:

```text
language
search profile
synonym dictionary
```

por tenant mediante integración explícita.

---

# 184. Persistent runtime safety

Tenant search configuration será scoped.

No:

```php
static $currentSearchLanguage;
```

---

# 185. Search + Sharding

Si el dataset está distribuido:

```text
Search Query
    ↓
Partition Routing
    ↓
Shard Search
```

---

# 186. Single-shard search

Si el shard key puede resolverse:

```text
one shard
```

---

# 187. Multi-shard search

Puede requerir:

```text
search each shard
+
merge results
```

---

# 188. Ranking across shards

Problema crítico:

> Scores calculados independientemente en shards diferentes pueden no ser directamente comparables.

---

# 189. Global ranking ≠ local ranking merge

No deberá asumirse:

```text
top 10 shard A
+
top 10 shard B
→
global top 10
```

sin demostrar equivalencia.

---

# 190. Distributed search policy

Podrá declarar:

```php
enum DistributedSearchRanking
{
    case LOCAL_ONLY;
    case APPROXIMATE_GLOBAL;
    case EXACT_GLOBAL;
    case UNSUPPORTED;
}
```

---

# 191. EXACT_GLOBAL

Podría requerir estadísticas globales o arquitectura especializada.

No será fingida si el motor no puede ofrecerla.

---

# 192. Partial shard failure

Si un shard falla:

```text
complete results
```

no podrán declararse.

---

# 193. Distributed Search Result

Podrá marcar:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 194. Search + Soft Delete

Los predicates de Soft Delete se aplicarán normalmente antes de devolver resultados.

---

# 195. Search + Temporal Data

Podrá combinarse:

```text
AS OF T
+
full-text predicate
```

sólo si la plataforma/arquitectura temporal puede conservar la semántica.

---

# 196. Search + History

Buscar versiones históricas será distinto de buscar entidades actuales.

---

# 197. Search + Archive

El Data Archival System podrá mantener índices separados sobre archives.

Pero:

```text
Operational Search Index
≠
Archive Search Index
```

---

# 198. Search archived data

Será una integración explícita.

No se asumirá que el DB full-text index operacional contiene datos purgados.

---

# 199. Search + Data Retention

Los resultados deberán respetar datos ya:

```text
purged
restricted
held
archived
```

según el source consultado.

---

# 200. Security architecture

El sistema deberá integrarse con:

```text
226_DATABASE_SECURITY_ARCHITECTURE.md
227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
233_DATABASE_QUERY_AUDIT_SYSTEM.md
234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 201. Search injection

Existe una segunda superficie además de SQL injection:

```text
search-language injection
```

---

# 202. Example

Un motor puede interpretar caracteres especiales como operadores.

Por ello:

```text
user string
```

no debe convertirse directamente en:

```text
vendor search syntax
```

---

# 203. Safe Search AST

La arquitectura será:

```text
User Input
    ↓
Search Parser
    ↓
Validated Search AST
    ↓
Platform Compiler
    ↓
Native Search Syntax
```

---

# 204. Raw Search

Podrá existir escape hatch:

```php
Search::raw(...)
```

pero será:

```text
explicit
platform-specific
unsafe-capability-aware
auditable
```

---

# 205. Raw Search ≠ Raw SQL

Aun así serán APIs diferentes.

---

# 206. Search limits

Resource Governance podrá imponer:

```text
max query length
max terms
max phrase length
max boolean depth
max OR branches
max prefix terms
max returned rows
max execution time
```

---

# 207. Wildcards

Los wildcards pueden generar búsquedas costosas.

Su uso estará controlado por policy.

---

# 208. Prefix expansion

No deberá permitirse expansión ilimitada.

---

# 209. Search denial-of-service

Inputs como:

```text
huge OR trees
extreme wildcard expansion
```

deberán limitarse.

---

# 210. Sensitive Data Search

No todo campo sensible deberá ser searchable.

---

# 211. Searchable classification

Metadata podrá indicar:

```text
SEARCHABLE
NOT_SEARCHABLE
RESTRICTED_SEARCH
```

---

# 212. Encrypted fields

Campos cifrados a nivel aplicación normalmente no podrán utilizar full-text DB-native sin una estrategia especializada.

---

# 213. No transparent decryption indexing

VoltStack no creará automáticamente índices plaintext de información cifrada.

---

# 214. Search authorization

Pueden existir permisos:

```text
database.search
database.search.sensitive
database.search.raw
database.search.explain
```

---

# 215. Query Audit

Podrá registrar:

```text
search document
search mode
term count
result count
duration
actor
tenant
```

---

# 216. Sensitive search terms

Los términos completos no deberán registrarse por default.

---

# 217. Audit representation

Podrá utilizar:

```text
term count
query fingerprint
classification
```

en lugar de contenido.

---

# 218. Telemetry

Métricas:

```text
database_search_queries_total
database_search_duration
database_search_results_total
database_search_failures_total
database_search_timeout_total
database_search_capability_fallback_total
```

---

# 219. Bounded dimensions

Permitidas:

```text
platform
search mode
result status
ranking mode
```

---

# 220. Forbidden dimensions

No:

```text
search query text
customer name
tenant ID
article ID
```

como labels por default.

---

# 221. Tracing

Span conceptual:

```text
database.search
```

con atributos bounded:

```text
search.mode
search.field_count
search.term_count
search.ranking
database.platform
```

---

# 222. Query Profiler

El Query Profiler podrá identificar:

```text
full-text predicate
search index
ranking cost
rows examined
execution duration
```

cuando la plataforma lo exponga.

---

# 223. Slow Query Detection

Las búsquedas full-text estarán sujetas al sistema normal de slow query detection.

---

# 224. Search diagnostics

Podrá mostrar:

```text
Search Document:
  article_search

Fields:
  title
  summary
  body

Language:
  spanish

Mode:
  NATURAL

Ranking:
  WEIGHTED

Platform:
  PostgreSQL

Native Capability:
  supported

Index:
  articles_search_idx

Pagination:
  cursor
```

---

# 225. Explain Search

```php
$query->explainSearch();
```

podrá devolver información estructurada.

---

# 226. Explain ≠ SQL dump only

Debe explicar intención semántica.

---

# 227. Example

```text
Full-Text Search Plan
────────────────────────────────

Document:
  article_search

Fields:
  title × 4
  summary × 2
  body × 1

Query:
  3 terms

Language:
  spanish

Search Mode:
  NATURAL

Platform Strategy:
  PostgreSQL tsvector

Index:
  article_search_idx

Ranking:
  weighted

Ordering:
  relevance DESC
  id ASC

Capabilities:
  phrase: supported
  prefix: supported
  highlighting: supported

Security Scope:
  applied

Tenant Scope:
  tenant_aware
```

---

# 228. Performance architecture

El principal mecanismo de rendimiento será:

```text
native full-text indexes
```

cuando estén disponibles.

---

# 229. No full table scan assumption

Una query full-text sin índice podrá ser:

```text
unsupported
rejected
warned
```

según plataforma/policy.

---

# 230. Search index design

Deberá considerar:

```text
field count
document size
language
update frequency
write amplification
index size
query patterns
```

---

# 231. Index ≠ free performance

Full-text indexes aumentan:

```text
storage
write cost
maintenance cost
```

---

# 232. Query performance budgets

Podrán configurarse:

```text
max search duration
max result window
max score computation
max highlighting size
```

---

# 233. Highlight cost

Highlighting podrá ser deshabilitado automáticamente sólo si el usuario no lo solicitó.

Nunca se eliminará una feature explícitamente solicitada sin informar.

---

# 234. Score calculation

Si el resultado no necesita ranking:

```text
score calculation
```

podrá omitirse.

---

# 235. Example

```php
Article::query()
    ->whereFullText(['body'], 'database')
    ->orderBy('id')
```

no requiere necesariamente proyectar score.

---

# 236. Search compilation cache

La estructura compilada podrá reutilizarse cuando:

```text
AST shape
platform
capabilities
metadata generation
search configuration
```

sean compatibles.

---

# 237. User terms ≠ cache key SQL string

Los parámetros continuarán separados del plan cuando sea posible.

---

# 238. Metadata compilation

SearchDocument definitions podrán precompilarse.

---

# 239. Persistent Runtime

Compatible con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 240. Shared immutable state

Podrá compartirse:

```text
compiled SearchDocument metadata
platform capability descriptors
immutable search compiler definitions
```

---

# 241. Scoped mutable state

Deberá ser scoped:

```text
SearchContext
tenant
authorization
query
parameters
search language override
resource budget
```

---

# 242. Forbidden static state

No:

```php
static string $currentLanguage;
static ?SearchQuery $currentSearch;
static ?TenantId $tenant;
```

---

# 243. Worker reset

Después de cada request/job deberán liberarse:

```text
SearchContext
temporary query state
profiling state
tenant bindings
authorization bindings
```

---

# 244. Error model

Jerarquía:

```text
DatabaseException
└── SearchException
    ├── SearchSyntaxException
    ├── SearchValidationException
    ├── SearchDocumentException
    ├── SearchFieldException
    ├── SearchCapabilityException
    ├── SearchCompilationException
    ├── SearchIndexException
    ├── SearchRankingException
    ├── SearchSecurityException
    ├── SearchResourceException
    └── DistributedSearchException
```

---

# 245. SearchSyntaxException

Input de lenguaje de búsqueda inválido.

---

# 246. SearchValidationException

AST válido sintácticamente pero semánticamente inválido.

---

# 247. SearchCapabilityException

La plataforma no puede satisfacer la intención requerida.

---

# 248. SearchIndexException

Falta o incompatibilidad del índice requerido.

---

# 249. SearchRankingException

No puede preservarse la estrategia de ranking.

---

# 250. SearchSecurityException

Violación de política de búsqueda.

---

# 251. DistributedSearchException

No puede completar búsqueda distribuida con las garantías solicitadas.

---

# 252. Testing architecture

El sistema requerirá:

```text
unit tests
semantic tests
compiler tests
platform conformance tests
integration tests
security tests
performance tests
```

---

# 253. Semantic tests

La misma Search AST deberá mantener la intención lógica entre plataformas soportadas dentro del perfil portable.

---

# 254. Platform tests

Cada plataforma tendrá corpus conocido.

Ejemplo:

```text
Document A:
  "distributed database systems"

Document B:
  "database architecture"

Document C:
  "network protocol"
```

---

# 255. Expected semantics

Buscar:

```text
database
```

deberá devolver A y B cuando la configuración lo indique.

---

# 256. Ranking tests

No deberán exigir el mismo score numérico entre MySQL y PostgreSQL.

Deberán validar garantías declaradas por cada strategy.

---

# 257. Phrase tests

```text
"distributed database"
```

deberá distinguirse de términos independientes cuando la capability lo soporte.

---

# 258. Language tests

Probar:

```text
English
Spanish
simple/no stemming
```

según plataformas disponibles.

---

# 259. Stop-word tests

Deberán documentar diferencias por plataforma/configuración.

---

# 260. Security tests

Probar inputs con:

```text
quotes
operators
wildcards
parentheses
SQL fragments
vendor syntax
Unicode edge cases
extreme nesting
```

---

# 261. SQL injection tests

Ejemplo:

```text
' OR 1=1 --
```

deberá permanecer dato de búsqueda, no SQL ejecutable.

---

# 262. Search-language injection tests

Un operador no permitido deberá ser:

```text
escaped
parsed
or rejected
```

según policy.

---

# 263. Resource tests

Probar:

```text
100k terms
deep boolean nesting
huge phrase
wildcard abuse
```

y comprobar límites.

---

# 264. Pagination tests

Probar:

```text
score ties
insertions
updates
deletes
stable tie-breakers
```

---

# 265. Cursor tests

Validar:

```text
search fingerprint
ranking configuration
language
tenant
security scope
```

---

# 266. Multitenancy tests

Tenant A nunca deberá recibir documentos de Tenant B.

---

# 267. Sharding tests

Probar:

```text
single shard
multiple shards
score incompatibility
partial shard failure
```

---

# 268. Persistent runtime tests

Request A:

```text
language = spanish
tenant = A
```

Request B:

```text
language = english
tenant = B
```

No deberá existir state leakage.

---

# 269. Directory proposal

```text
src/Quantum/Database/Search/
│
├── Contract/
│   ├── SearchPlanner.php
│   ├── SearchCompiler.php
│   ├── SearchInputParser.php
│   ├── SearchCapabilityResolver.php
│   └── SearchMetadataResolver.php
│
├── Model/
│   ├── SearchQuery.php
│   ├── SearchDocument.php
│   ├── SearchDocumentId.php
│   ├── SearchField.php
│   ├── SearchTerm.php
│   ├── SearchPhrase.php
│   ├── SearchLanguage.php
│   ├── SearchScore.php
│   └── SearchResult.php
│
├── AST/
│   ├── SearchExpression.php
│   ├── SearchTermsExpression.php
│   ├── SearchPhraseExpression.php
│   ├── SearchBooleanExpression.php
│   ├── SearchPrefixExpression.php
│   ├── SearchPredicate.php
│   ├── SearchScoreExpression.php
│   └── SearchHighlightExpression.php
│
├── Parser/
│   ├── SearchLexer.php
│   ├── SearchParser.php
│   ├── SearchToken.php
│   └── SearchInputPolicy.php
│
├── Semantic/
│   ├── SearchSemanticAnalyzer.php
│   ├── SearchSemanticProfile.php
│   ├── SearchValidationResult.php
│   └── SearchFieldResolver.php
│
├── Metadata/
│   ├── SearchDocumentMetadata.php
│   ├── SearchFieldMetadata.php
│   ├── SearchMetadataRegistry.php
│   └── SearchMetadataCompiler.php
│
├── Planning/
│   ├── SearchExecutionPlan.php
│   ├── SearchRankingPlan.php
│   ├── SearchIndexPlan.php
│   ├── DistributedSearchPlan.php
│   └── DefaultSearchPlanner.php
│
├── Capability/
│   ├── SearchCapability.php
│   ├── SearchCapabilities.php
│   ├── SearchCapabilityStatus.php
│   └── DefaultSearchCapabilityResolver.php
│
├── Ranking/
│   ├── SearchRankingStrategy.php
│   ├── SearchRankingPolicy.php
│   └── SearchWeight.php
│
├── Highlight/
│   ├── SearchHighlight.php
│   ├── SearchHighlightFragment.php
│   └── SearchHighlightPolicy.php
│
├── Index/
│   ├── FullTextIndexDefinition.php
│   ├── SearchIndexMetadata.php
│   ├── SearchIndexRequirement.php
│   └── SearchIndexInspector.php
│
├── Compiler/
│   ├── AbstractSearchCompiler.php
│   ├── MySQLSearchCompiler.php
│   ├── MariaDBSearchCompiler.php
│   ├── PostgreSQLSearchCompiler.php
│   └── SQLiteSearchCompiler.php
│
├── Distribution/
│   ├── DistributedSearchPlanner.php
│   ├── DistributedSearchRanking.php
│   ├── SearchShardResult.php
│   └── SearchMergeResult.php
│
├── Security/
│   ├── SearchSecurityPolicy.php
│   ├── SearchInputSanitizer.php
│   └── SearchPermission.php
│
├── Diagnostics/
│   ├── SearchDiagnostics.php
│   ├── SearchExplain.php
│   └── SearchPlanFormatter.php
│
├── Telemetry/
│   └── SearchTelemetry.php
│
└── Exception/
    ├── SearchException.php
    ├── SearchSyntaxException.php
    ├── SearchValidationException.php
    ├── SearchDocumentException.php
    ├── SearchFieldException.php
    ├── SearchCapabilityException.php
    ├── SearchCompilationException.php
    ├── SearchIndexException.php
    ├── SearchRankingException.php
    ├── SearchSecurityException.php
    ├── SearchResourceException.php
    └── DistributedSearchException.php
```

---

# 270. Architectural invariants

## DB-SEARCH-001

Full-Text Search será una extensión del Query Engine.

## DB-SEARCH-002

No existirá un segundo query executor para Full-Text Search.

## DB-SEARCH-003

ORM no generará sintaxis full-text específica del proveedor.

## DB-SEARCH-004

Query Builder producirá Search AST.

## DB-SEARCH-005

Search AST será distinto de SQL.

## DB-SEARCH-006

Full-Text Search será distinto de LIKE.

## DB-SEARCH-007

Full-Text Search será distinto de Regex.

## DB-SEARCH-008

Full-Text Search será distinto de Vector Search.

## DB-SEARCH-009

Full-Text Search será distinto de Semantic Search.

## DB-SEARCH-010

SearchDocument será distinto de ORM Entity.

## DB-SEARCH-011

SearchField será distinto de DB column.

## DB-SEARCH-012

SearchQuery será distinto de SQL query.

## DB-SEARCH-013

SearchTerm nunca será tratado como SQL fragment.

## DB-SEARCH-014

Boolean search tendrá representación AST.

## DB-SEARCH-015

Vendor boolean syntax no será API portable.

## DB-SEARCH-016

Search input externo será validado.

## DB-SEARCH-017

Search parser será distinto de SQL parser.

## DB-SEARCH-018

Search input tendrá límites de complejidad.

## DB-SEARCH-019

Language será explícito cuando afecte semántica.

## DB-SEARCH-020

SearchLanguage será distinto de Locale.

## DB-SEARCH-021

Tokenization podrá ser platform-dependent.

## DB-SEARCH-022

Stemming podrá ser platform-dependent.

## DB-SEARCH-023

Stop words podrán ser platform-dependent.

## DB-SEARCH-024

VoltStack no fingirá equivalencia lingüística inexistente.

## DB-SEARCH-025

No existirá degradación semántica silenciosa.

## DB-SEARCH-026

Capabilities gobernarán compilación.

## DB-SEARCH-027

UNKNOWN capability será distinta de SUPPORTED.

## DB-SEARCH-028

Version será distinta de capability.

## DB-SEARCH-029

MySQL será distinto de MariaDB.

## DB-SEARCH-030

SQLite FTS no se asumirá siempre disponible.

## DB-SEARCH-031

Search Compiler será platform-specific.

## DB-SEARCH-032

Search Compiler se integrará al SQL Compiler.

## DB-SEARCH-033

Search parameters serán parametrizados cuando sea posible.

## DB-SEARCH-034

Vendor search syntax será construida desde AST validado.

## DB-SEARCH-035

Raw vendor search no será default.

## DB-SEARCH-036

Search Index será distinto de ordinary index.

## DB-SEARCH-037

Search metadata será distinta del índice físico.

## DB-SEARCH-038

ORM metadata no ejecutará DDL.

## DB-SEARCH-039

Schema/Migrations gobernarán índices físicos.

## DB-SEARCH-040

SearchDocument metadata podrá precompilarse.

## DB-SEARCH-041

Weight será distinto de relevance score.

## DB-SEARCH-042

SearchScore no será tratado como probabilidad universal.

## DB-SEARCH-043

Scores no serán asumidos comparables entre plataformas.

## DB-SEARCH-044

Search score podrá formar parte de projection.

## DB-SEARCH-045

SearchResult será distinto de managed entity.

## DB-SEARCH-046

Ranking strategy será capability-aware.

## DB-SEARCH-047

Unsupported ranking no se degradará silenciosamente.

## DB-SEARCH-048

Highlight será distinto de matching.

## DB-SEARCH-049

Database layer no generará HTML confiable por default.

## DB-SEARCH-050

Highlight rendering pertenecerá a capas superiores.

## DB-SEARCH-051

Model API y Repository usarán el mismo Search Engine.

## DB-SEARCH-052

Search Planner no ejecutará consultas.

## DB-SEARCH-053

Semantic Analyzer validará Search AST.

## DB-SEARCH-054

String field será distinto de searchable field.

## DB-SEARCH-055

Searchable será distinto de indexed.

## DB-SEARCH-056

Search index requirement será explícito.

## DB-SEARCH-057

LIKE no será fallback implícito.

## DB-SEARCH-058

Fallback deberá ser explícito.

## DB-SEARCH-059

Optimizer podrá reutilizar expresiones equivalentes.

## DB-SEARCH-060

Optimizer no alterará search semantics.

## DB-SEARCH-061

Full-text predicates podrán coexistir con predicates normales.

## DB-SEARCH-062

Search nunca bypassará authorization.

## DB-SEARCH-063

Ranking podrá ser información sensible.

## DB-SEARCH-064

Result count podrá ser información sensible.

## DB-SEARCH-065

Highlight podrá ser información sensible.

## DB-SEARCH-066

Authorization se aplicará antes de exposición.

## DB-SEARCH-067

Search será compatible con Pagination.

## DB-SEARCH-068

Count Query podrá omitir ranking innecesario.

## DB-SEARCH-069

Score-only ordering podrá ser no determinista.

## DB-SEARCH-070

Tie-breakers podrán ser necesarios.

## DB-SEARCH-071

Search será compatible con Cursor Pagination sólo cuando sea eligible.

## DB-SEARCH-072

Search score podrá ser inestable.

## DB-SEARCH-073

Cursor eligibility deberá analizar score stability.

## DB-SEARCH-074

Cursor deberá vincularse al search fingerprint.

## DB-SEARCH-075

Search cursor no será automáticamente seguro.

## DB-SEARCH-076

Search podrá integrarse con Chunk Processing.

## DB-SEARCH-077

Mutable score ordering podrá ser inseguro para chunk traversal.

## DB-SEARCH-078

Search podrá integrarse con Lazy Collection.

## DB-SEARCH-079

Result Cache será distinto de Search Index.

## DB-SEARCH-080

Search cache identity incluirá search semantics.

## DB-SEARCH-081

Index freshness será explícita cuando sea relevante.

## DB-SEARCH-082

UNKNOWN freshness será distinta de CURRENT.

## DB-SEARCH-083

Full-text search no abrirá transactions automáticamente.

## DB-SEARCH-084

Read routing seguirá Transaction Context.

## DB-SEARCH-085

Replica search respetará consistency policy.

## DB-SEARCH-086

Read-your-writes podrá requerir writer/sticky routing.

## DB-SEARCH-087

Search index será distinto de tenant isolation.

## DB-SEARCH-088

Tenant predicate no podrá omitirse.

## DB-SEARCH-089

Tenant search configuration será scoped.

## DB-SEARCH-090

No habrá tenant search state global.

## DB-SEARCH-091

Sharding utilizará Partition Routing.

## DB-SEARCH-092

Local shard score no será asumido globalmente comparable.

## DB-SEARCH-093

Global ranking no será fingido.

## DB-SEARCH-094

Partial shard failure no producirá resultado COMPLETE.

## DB-SEARCH-095

Search respetará Soft Delete.

## DB-SEARCH-096

Temporal Search requerirá semántica demostrable.

## DB-SEARCH-097

Historical Search será distinto de current Search.

## DB-SEARCH-098

Operational Search Index será distinto de Archive Search Index.

## DB-SEARCH-099

Purged operational data no permanecerá implícitamente searchable.

## DB-SEARCH-100

Search input será tratado como untrusted input.

## DB-SEARCH-101

Search-language injection será tratada explícitamente.

## DB-SEARCH-102

User text no será concatenado a SQL.

## DB-SEARCH-103

User text no será passthrough a vendor syntax por default.

## DB-SEARCH-104

Raw Search será escape hatch explícito.

## DB-SEARCH-105

Raw Search será auditable.

## DB-SEARCH-106

Search tendrá resource limits.

## DB-SEARCH-107

Wildcard expansion podrá limitarse.

## DB-SEARCH-108

Boolean depth podrá limitarse.

## DB-SEARCH-109

Searchable sensitive fields requerirán policy.

## DB-SEARCH-110

Encrypted fields no serán plaintext-indexed automáticamente.

## DB-SEARCH-111

Search permissions serán distintas de data eligibility.

## DB-SEARCH-112

Sensitive search terms no aparecerán en telemetry por default.

## DB-SEARCH-113

Telemetry dimensions serán bounded.

## DB-SEARCH-114

Query Profiler reconocerá Full-Text Search.

## DB-SEARCH-115

Slow Query Detection aplicará a Full-Text Search.

## DB-SEARCH-116

Search Explain describirá semántica, no sólo SQL.

## DB-SEARCH-117

Full-text indexes no serán considerados gratuitos.

## DB-SEARCH-118

Index write amplification será parte del performance model.

## DB-SEARCH-119

Highlighting tendrá resource budget.

## DB-SEARCH-120

Score computation podrá omitirse si no es necesario.

## DB-SEARCH-121

Search compilation podrá cachearse.

## DB-SEARCH-122

Search metadata podrá cachearse.

## DB-SEARCH-123

Cache keys incluirán metadata/capability generations cuando corresponda.

## DB-SEARCH-124

Persistent runtime state será scoped.

## DB-SEARCH-125

Compiled immutable metadata podrá compartirse.

## DB-SEARCH-126

Current search query nunca será static mutable state.

## DB-SEARCH-127

Current search language override nunca será static mutable state.

## DB-SEARCH-128

Worker reset eliminará SearchContext.

## DB-SEARCH-129

Search errors tendrán jerarquía propia.

## DB-SEARCH-130

Capability errors serán distintos de syntax errors.

## DB-SEARCH-131

Index errors serán distintos de ranking errors.

## DB-SEARCH-132

Distributed search errors serán explícitos.

## DB-SEARCH-133

Platform conformance será testeada.

## DB-SEARCH-134

Portable semantics serán testeadas independientemente de SQL.

## DB-SEARCH-135

Ranking tests no exigirán scores idénticos entre plataformas.

## DB-SEARCH-136

Phrase semantics serán testeadas.

## DB-SEARCH-137

Language behavior será testeado.

## DB-SEARCH-138

Stop-word behavior será documentado/testeado.

## DB-SEARCH-139

SQL injection será testeada.

## DB-SEARCH-140

Search-language injection será testeada.

## DB-SEARCH-141

Resource exhaustion será testeada.

## DB-SEARCH-142

Pagination score ties serán testeados.

## DB-SEARCH-143

Cursor fingerprints serán testeados.

## DB-SEARCH-144

Tenant isolation será testeada.

## DB-SEARCH-145

Distributed ranking limitations serán testeadas.

## DB-SEARCH-146

Persistent worker isolation será testeada.

## DB-SEARCH-147

Search AST será immutable o tratado como immutable después de planificación.

## DB-SEARCH-148

Search planning será deterministic bajo mismo input/context.

## DB-SEARCH-149

Search compiler no accederá directamente al ORM.

## DB-SEARCH-150

Search compiler no ejecutará queries.

## DB-SEARCH-151

Driver no interpretará Search AST.

## DB-SEARCH-152

Connection no conocerá SearchDocument metadata.

## DB-SEARCH-153

ORM no conocerá vendor search syntax.

## DB-SEARCH-154

Search System no será un segundo Schema System.

## DB-SEARCH-155

Search System no será un segundo Security System.

## DB-SEARCH-156

Search System no será un segundo Telemetry System.

## DB-SEARCH-157

Search System no será un segundo Pagination System.

## DB-SEARCH-158

Search System reutilizará Query AST.

## DB-SEARCH-159

Search System reutilizará Query Planner.

## DB-SEARCH-160

Search System reutilizará SQL Compiler.

## DB-SEARCH-161

Search System reutilizará Execution Engine.

## DB-SEARCH-162

Search System reutilizará Result/Hydration.

## DB-SEARCH-163

Search System reutilizará Platform Capabilities.

## DB-SEARCH-164

Search System reutilizará Schema metadata.

## DB-SEARCH-165

Search System preservará typed parameters.

## DB-SEARCH-166

Search result shape será explícito.

## DB-SEARCH-167

Search score no será propiedad persistente implícita.

## DB-SEARCH-168

Search highlighting no mutará entities.

## DB-SEARCH-169

Search query parsing será independiente de HTTP.

## DB-SEARCH-170

Database Search System no conocerá URLs.

## DB-SEARCH-171

Database Search System no conocerá request query strings HTTP.

## DB-SEARCH-172

HTTP adapters podrán traducir parámetros a SearchQuery.

## DB-SEARCH-173

Invalid search input no llegará como SQL inválido al driver.

## DB-SEARCH-174

Unsupported capability fallará antes de ejecución cuando sea detectable.

## DB-SEARCH-175

Search semantics serán explainable.

## DB-SEARCH-176

Search plans serán inspectables.

## DB-SEARCH-177

Search execution será observable.

## DB-SEARCH-178

Search telemetry no expondrá datos sensibles por default.

## DB-SEARCH-179

Search audit preservará minimización de datos.

## DB-SEARCH-180

VoltStack expresará claramente las diferencias semánticas entre plataformas.

---

# 271. Modelo formal

Sea un documento:

```text
D = {f₁, f₂, ..., fₙ}
```

donde cada:

```text
fᵢ
```

es un campo searchable.

Sea:

```text
Q
```

una SearchQuery.

Sea:

```text
L
```

la configuración lingüística.

Sea:

```text
P
```

la plataforma activa.

La operación conceptual será:

```text
Match(D, Q, L, P)
```

---

# 272. Matching

Un documento pertenece al resultado cuando:

```text
Match(D, Q, L, P) = true
```

según las semánticas declaradas por:

```text
SearchSemanticProfile
```

y las capabilities efectivas de `P`.

---

# 273. Ranking

Sea:

```text
R(D,Q,P)
```

la función de ranking efectiva de la plataforma.

Entonces:

```text
score = R(D,Q,P)
```

Pero:

```text
R_mysql
≠
R_postgresql
≠
R_sqlite
```

necesariamente.

---

# 274. Weighted document

Para campos:

```text
f₁ ... fₙ
```

con pesos:

```text
w₁ ... wₙ
```

el Search Plan podrá representar conceptualmente:

```text
D_w = {
    (f₁,w₁),
    (f₂,w₂),
    ...
    (fₙ,wₙ)
}
```

La traducción física será responsabilidad del platform compiler.

---

# 275. Search result ordering

Un ordering estable puede expresarse:

```text
O(r) =
(
    score(r) DESC,
    stableTieBreaker(r) ASC
)
```

---

# 276. Cursor boundary

Para cursor pagination:

```text
C =
(
    score,
    tieBreaker
)
```

si y sólo si la estrategia es elegible.

---

# 277. Distributed ranking

Sean shards:

```text
S₁ ... Sₙ
```

Cada uno puede calcular:

```text
Rᵢ(D,Q)
```

No deberá suponerse:

```text
R₁ ≡ R₂ ≡ ... ≡ Rₙ
```

si las estadísticas locales afectan scoring.

---

# 278. Arquitectura conceptual final

```text
                         Application
                              │
                              ▼
                    Model / Repository API
                              │
                              ▼
                       SearchQuery
                              │
                              ▼
                        Search AST
                              │
                              ▼
                    Semantic Analyzer
                              │
               ┌──────────────┼──────────────┐
               │              │              │
               ▼              ▼              ▼
         ORM Metadata   Search Metadata   Schema Metadata
               │              │              │
               └──────────────┼──────────────┘
                              ▼
                     Capability Resolver
                              │
                              ▼
                       Search Planner
                              │
                              ▼
                     Query Logical Plan
                              │
                              ▼
                        Query Optimizer
                              │
                              ▼
                    Platform SQL Compiler
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
            MySQL          PostgreSQL        SQLite
           FULLTEXT       tsvector/tsquery      FTS
              │               │                │
              └───────────────┼────────────────┘
                              ▼
                       Execution Engine
                              │
                              ▼
                         Result System
                              │
                    ┌─────────┴──────────┐
                    │                    │
                    ▼                    ▼
                Hydration          Search Metadata
                                      │
                              ┌───────┴────────┐
                              ▼                ▼
                           Score           Highlight
```

---

# 279. Flujo de búsqueda

```text
User Search Input
       │
       ▼
Input Policy
       │
       ▼
Search Parser
       │
       ▼
Search AST
       │
       ▼
Semantic Validation
       │
       ▼
Security Scope
       │
       ▼
Tenant Scope
       │
       ▼
Capability Analysis
       │
       ▼
Search Planning
       │
       ▼
Query Optimization
       │
       ▼
Platform Compilation
       │
       ▼
Prepared Execution
       │
       ▼
Database Search Index
       │
       ▼
Result
       │
       ├── Entity / DTO / Projection
       ├── Score
       └── Highlight
```

---

# 280. Dependencias permitidas

```text
Search
 ├── Query
 ├── AST
 ├── Semantic
 ├── Optimizer
 ├── Planner
 ├── Compiler
 ├── Schema
 ├── ORM Metadata
 ├── Platform Capabilities
 ├── Security
 ├── Telemetry
 ├── Pagination
 ├── Multitenancy
 └── Distribution
```

---

# 281. Dependencias prohibidas

No:

```text
Driver
  ↓
Search AST
```

No:

```text
Connection
  ↓
Search Metadata
```

No:

```text
Entity
  ↓
PostgreSQL tsquery
```

No:

```text
Repository
  ↓
MATCH AGAINST
```

No:

```text
Model
  ↓
SQLite FTS syntax
```

---

# 282. Dirección correcta

```text
Model / Repository
        ↓
Query Builder
        ↓
Search AST
        ↓
Semantic Engine
        ↓
Planner
        ↓
Optimizer
        ↓
Compiler
        ↓
Execution
        ↓
Connection
        ↓
Driver
```

---

# 283. Ejemplo completo

Aplicación:

```php
$results = Article::query()
    ->search(
        document: 'article_search',
        query: Search::query()
            ->phrase('distributed database')
            ->or()
            ->term('replication')
            ->language('english')
    )
    ->withSearchScore('relevance')
    ->orderBySearchScore('relevance', descending: true)
    ->orderBy('id')
    ->paginate(20);
```

Internamente:

```text
Model API
   ↓
Entity Query
   ↓
Search AST
   ↓
SearchDocument Metadata
   ↓
Semantic Validation
   ↓
Authorization Scope
   ↓
Tenant Scope
   ↓
Search Planner
   ↓
Query Planner
   ↓
Optimizer
   ↓
Platform Search Compiler
   ↓
SQL Compiler
   ↓
Prepared Statement
   ↓
Executor
   ↓
Result
   ↓
Hydration
   ↓
PageResult<SearchResult<Article>>
```

---

# 284. MySQL conceptual compilation

El Search AST podría terminar representándose físicamente mediante:

```text
MATCH(title, summary, body)
AGAINST(...)
```

pero esa sintaxis sólo existirá dentro de:

```text
MySQLSearchCompiler
```

---

# 285. PostgreSQL conceptual compilation

El mismo Search AST podría convertirse a:

```text
tsvector
+
tsquery
+
ts_rank
```

dentro de:

```text
PostgreSQLSearchCompiler
```

---

# 286. SQLite conceptual compilation

Podría convertirse a una consulta:

```text
FTS5 MATCH
```

cuando:

```text
supportsFullTextSearch() = true
```

---

# 287. Application portability

La aplicación continuará expresando:

```php
->search('database architecture')
```

sin conocer:

```text
MATCH AGAINST
tsvector
tsquery
FTS5
```

---

# 288. Escape hatch

Cuando el desarrollador requiera una característica exclusiva de PostgreSQL, MySQL, MariaDB o SQLite podrá utilizar una extensión explícita.

Ejemplo conceptual:

```php
Search::platform('postgresql')
    ->extension(...);
```

---

# 289. Portable core + native extensions

La estrategia será:

```text
Portable Search Core
        │
        ├── MySQL Extensions
        ├── MariaDB Extensions
        ├── PostgreSQL Extensions
        └── SQLite Extensions
```

---

# 290. Native extension isolation

Una extensión native:

```text
PostgreSQLSearchExtension
```

no deberá contaminar:

```text
SearchQuery core
ORM core
Entity metadata core
```

---

# 291. Decisión arquitectónica final

VoltStack implementará Full-Text Search como una **capacidad semántica especializada del Query Engine**, no como una colección de helpers SQL específicos de cada motor.

La arquitectura será:

```text
Search Intent
      ↓
SearchQuery
      ↓
Search AST
      ↓
Semantic Validation
      ↓
Search Metadata
      ↓
Capability Resolution
      ↓
Search Planning
      ↓
Query Optimization
      ↓
Platform Search Compiler
      ↓
SQL Compiler
      ↓
Execution Engine
      ↓
Database-native Full-Text Engine
```

La API pública podrá mantener ergonomía similar a Laravel:

```php
Article::query()
    ->whereFullText(['title', 'body'], 'database architecture')
    ->get();
```

mientras internamente conserva una arquitectura:

```text
typed
portable
capability-driven
compiler-based
secure
observable
extensible
```

El principio rector será:

> **La aplicación expresa qué desea buscar; nunca necesita expresar cómo MySQL, MariaDB, PostgreSQL o SQLite implementan físicamente esa búsqueda.**

Complementariamente:

> **Portabilidad no significará fingir que todos los motores poseen exactamente las mismas capacidades. VoltStack preservará la intención común cuando sea posible y expondrá explícitamente cualquier limitación, diferencia o extensión específica de plataforma.**

Y finalmente:

> **VoltStack nunca sacrificará semántica ni seguridad para aparentar portabilidad: una búsqueda no soportada fallará de forma explícita antes que convertirse silenciosamente en una consulta diferente.**

---

# 292. Bloque 27 — progreso

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES

✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
✓ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
✓ 269_DATABASE_SOFT_DELETE_SYSTEM.md
✓ 270_DATABASE_DATA_RETENTION_SYSTEM.md
✓ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
✓ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
□ 273_DATABASE_JSON_QUERY_SYSTEM.md
□ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
□ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 293. Siguiente documento

```text
273_DATABASE_JSON_QUERY_SYSTEM.md
```

El siguiente documento definirá el sistema de consultas sobre estructuras JSON, incluyendo:

```text
JSON logical type
JSON documents
JSON paths
JSON path AST
JSON extraction
JSON predicates
JSON existence
JSON containment
JSON scalar comparison
JSON arrays
JSON objects
JSON null
SQL NULL
missing paths
JSON mutation
JSON set
JSON remove
JSON merge
JSON aggregation
JSON projection
JSON indexing
generated columns
platform capabilities
MySQL JSON
MariaDB JSON
PostgreSQL JSON / JSONB
SQLite JSON functions
portable JSON semantics
Query Builder integration
ORM integration
Type System integration
security
parameter binding
optimization
index awareness
pagination
multitenancy
sharding
telemetry
testing
```

bajo la distinción fundamental:

> **SQL NULL, JSON null y una ruta JSON inexistente serán tres estados semánticos diferentes y VoltStack no deberá colapsarlos accidentalmente.**