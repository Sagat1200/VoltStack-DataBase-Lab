# 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## Geographic Data Extension System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 274 — Geographic Data Extension System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `273_DATABASE_JSON_QUERY_SYSTEM.md`  
**Siguiente documento:** `275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Geographic Data Extension System** de VoltStack Database.

El objetivo es proporcionar una abstracción geoespacial tipada, portable y extensible para trabajar con:

```text
Point
LineString
Polygon
MultiPoint
MultiLineString
MultiPolygon
GeometryCollection
BoundingBox
Coordinate
Spatial Reference Systems
```

sobre los motores soportados por VoltStack:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

y extensiones geoespaciales disponibles para ellos.

La regla fundamental será:

> **Una coordenada no posee significado geoespacial completo sin un sistema de referencia; VoltStack no deberá comparar, medir ni transformar geometrías ignorando silenciosamente su SRID o sistema de coordenadas.**

La aplicación expresará:

```text
geographic intent
```

y no:

```text
vendor-specific spatial SQL
```

Por tanto:

```text
Application
    ↓
Spatial Query API
    ↓
Spatial AST
    ↓
Spatial Semantic Analysis
    ↓
CRS/SRID Resolution
    ↓
Capability Resolution
    ↓
Query Planner
    ↓
Platform Spatial Compiler
    ↓
Database Spatial Engine
```

---

# 2. Objetivos

El sistema deberá soportar arquitectónicamente:

- tipos geométricos;
- tipos geográficos;
- coordenadas;
- SRID;
- sistemas de referencia;
- validación geométrica;
- spatial predicates;
- relaciones topológicas;
- distancias;
- proximidad;
- nearest-neighbor;
- bounding boxes;
- spatial indexes;
- construcción de geometrías;
- transformación de coordenadas;
- serialización WKT;
- serialización WKB;
- GeoJSON;
- ORM mapping;
- Query Builder;
- Query AST;
- Spatial AST;
- Schema System;
- Type System;
- Platform Capabilities;
- optimización;
- seguridad;
- telemetría;
- distribución;
- Multitenancy;
- extensiones geoespaciales externas.

---

# 3. Alcance

El sistema se divide conceptualmente en:

```text
Spatial Type System
Spatial Value Model
Spatial Query System
Spatial Predicate System
Spatial Function System
Spatial Reference System
Spatial Index System
Spatial Compiler System
Spatial Extension System
```

---

# 4. No es un GIS completo

VoltStack Database no intentará convertirse en:

```text
QGIS
ArcGIS
full GIS platform
map rendering engine
tile server
routing engine
geocoding service
```

Su responsabilidad será:

> Proporcionar representación, persistencia y consultas geoespaciales coherentes dentro de la arquitectura Database.

---

# 5. Geographic ≠ Geometry

VoltStack distinguirá conceptualmente:

```text
Geometry
```

de:

```text
Geography
```

cuando el motor subyacente lo soporte.

---

# 6. Geometry

Generalmente representa coordenadas sobre un espacio definido por un CRS.

Las operaciones pueden utilizar geometría:

```text
planar
projected
cartesian
```

dependiendo del CRS.

---

# 7. Geography

Representa datos sobre la superficie terrestre y puede requerir cálculos geodésicos.

Conceptualmente:

```text
Geometry
≠
Geography
```

---

# 8. Ejemplo crítico

Los puntos:

```text
(-100, 25)
(-99, 25)
```

por sí solos no indican si las unidades son:

```text
degrees
meters
feet
arbitrary coordinates
```

---

# 9. CRS

Se introduce:

```text
CoordinateReferenceSystem
```

abreviado:

```text
CRS
```

---

# 10. SRID

Un identificador:

```text
SRID
```

referencia un sistema espacial determinado.

Ejemplo conocido:

```text
EPSG:4326
```

---

# 11. Regla crítica

```text
Coordinates
+
CRS
=
Spatial Meaning
```

---

# 12. SRID ≠ Coordinate

El SRID no forma parte de cada dimensión numérica.

Es metadata semántica de la geometría.

---

# 13. SRID ≠ Unit

Un SRID permite determinar propiedades del CRS, entre ellas potencialmente unidades.

Pero:

```text
SRID
≠
unit
```

---

# 14. Coordinate Reference System model

```php
final readonly class CoordinateReferenceSystem
{
    public function __construct(
        public SpatialReferenceId $id,
        public CoordinateSystemType $type,
        public AxisOrder $axisOrder,
        public ?SpatialUnit $unit,
    ) {}
}
```

---

# 15. SpatialReferenceId

```php
final readonly class SpatialReferenceId
{
    public function __construct(
        public int $value
    ) {}
}
```

---

# 16. No magic integer everywhere

Evitar:

```php
4326
```

propagado por todo el framework.

Preferir:

```php
SpatialReferenceId::from(4326)
```

o:

```php
SpatialReferenceSystem::WGS84
```

cuando exista un alias seguro.

---

# 17. Axis order

Debe modelarse explícitamente.

---

# 18. Problema lat/lon

Es común representar:

```text
latitude, longitude
```

mientras ciertas representaciones espaciales utilizan:

```text
X = longitude
Y = latitude
```

---

# 19. Regla

VoltStack no deberá intercambiar silenciosamente:

```text
lat
lon
```

con:

```text
x
y
```

---

# 20. Coordinate

Modelo básico:

```php
final readonly class Coordinate
{
    public function __construct(
        public float $x,
        public float $y,
        public ?float $z = null,
        public ?float $m = null,
    ) {}
}
```

---

# 21. Coordinate ≠ Point

Una coordenada es un conjunto ordenado de ordinadas.

Un Point es una geometría.

```text
Coordinate
≠
Point
```

---

# 22. Spatial dimensions

VoltStack deberá poder representar:

```text
XY
XYZ
XYM
XYZM
```

cuando la plataforma/capability lo permita.

---

# 23. Z

Puede representar, según dominio:

```text
height
elevation
depth
other third dimension
```

VoltStack no deberá asumir automáticamente:

```text
Z = elevation above sea level
```

---

# 24. M

Puede representar una medida adicional.

```text
M
≠
time
```

por default.

---

# 25. Geometry hierarchy

```text
SpatialValue
├── Geometry
│   ├── Point
│   ├── LineString
│   ├── Polygon
│   ├── MultiPoint
│   ├── MultiLineString
│   ├── MultiPolygon
│   └── GeometryCollection
│
└── Geography
    └── platform/capability-aware variants
```

---

# 26. Geometry base contract

```php
interface Geometry
{
    public function srid(): SpatialReferenceId;

    public function dimension(): CoordinateDimension;

    public function isEmpty(): bool;

    public function geometryType(): GeometryType;
}
```

---

# 27. Point

Representa una única posición.

```php
$point = Point::xy(
    x: -100.316,
    y: 25.686,
    srid: 4326,
);
```

---

# 28. Latitude/Longitude API

Podrá existir una API explícita:

```php
$point = GeographicPoint::fromLongitudeLatitude(
    longitude: -100.316,
    latitude: 25.686,
);
```

para reducir ambigüedad.

---

# 29. No ambiguous constructor

Evitar como API principal:

```php
new Point(25.686, -100.316);
```

cuando no sea evidente:

```text
lat/lon?
x/y?
```

---

# 30. LineString

Representa una secuencia ordenada de puntos.

```text
P₁ → P₂ → P₃ → ... → Pₙ
```

---

# 31. Minimum cardinality

La validación deberá respetar las reglas geométricas aplicables.

Una LineString válida normalmente requiere suficientes puntos para formar una línea.

---

# 32. Polygon

Representa:

```text
exterior ring
+
zero or more interior rings
```

---

# 33. Polygon model

```text
Polygon
├── ExteriorRing
└── InteriorRings[]
```

---

# 34. Ring

Un ring deberá cumplir las invariantes geométricas configuradas.

Por ejemplo:

```text
closed
sufficient points
coordinate dimensional consistency
```

---

# 35. Holes

Los interior rings representan huecos.

```text
Polygon
 ├── shell
 ├── hole 1
 └── hole 2
```

---

# 36. MultiPoint

```text
MultiPoint
=
collection of Points
```

---

# 37. MultiLineString

```text
MultiLineString
=
collection of LineStrings
```

---

# 38. MultiPolygon

```text
MultiPolygon
=
collection of Polygons
```

---

# 39. GeometryCollection

Puede contener múltiples tipos de geometría.

```text
GeometryCollection
├── Point
├── LineString
└── Polygon
```

---

# 40. GeometryCollection ≠ MultiGeometry

Una MultiPolygon sólo contiene polygons.

GeometryCollection puede ser heterogénea.

---

# 41. Empty geometries

El modelo deberá poder distinguir:

```text
empty geometry
```

de:

```text
SQL NULL
```

---

# 42. Critical null rule

```text
SQL NULL
≠
EMPTY GEOMETRY
```

---

# 43. Geometry validity

Debe distinguirse:

```text
syntactically representable geometry
```

de:

```text
topologically valid geometry
```

---

# 44. Invalid geometry example

Un polygon auto-intersectado puede ser representable pero inválido según las reglas aplicables.

---

# 45. Geometry validation

Conceptualmente:

```text
GeometryValidator
```

---

# 46. Validation levels

```php
enum SpatialValidationLevel
{
    case STRUCTURAL;
    case TOPOLOGICAL;
    case STRICT;
    case PLATFORM;
}
```

---

# 47. Structural validation

Comprueba:

```text
dimensions
ring structure
coordinate count
SRID presence
compatible child types
```

---

# 48. Topological validation

Puede comprobar:

```text
self intersections
ring validity
polygon topology
```

cuando exista soporte.

---

# 49. Client validation ≠ database validation

VoltStack puede realizar validaciones tempranas.

Pero:

```text
FrameworkValidation
≠
DatabaseSpatialValidation
```

---

# 50. Spatial Type System

Se integrará con:

```text
155_DATABASE_TYPE_SYSTEM.md
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md
```

---

# 51. Logical types

Podrán existir:

```text
spatial.geometry
spatial.point
spatial.linestring
spatial.polygon
spatial.multipoint
spatial.multilinestring
spatial.multipolygon
spatial.geometry_collection
spatial.geography
```

---

# 52. Logical type ≠ physical DB type

Ejemplo:

```text
VoltStack Point
```

puede mapearse de forma diferente según:

```text
MySQL
MariaDB
PostgreSQL/PostGIS
SQLite extension
```

---

# 53. Type Registry

Los tipos espaciales deberán registrarse mediante:

```text
TypeRegistry
```

sin crear un Type System paralelo.

---

# 54. Value conversion

Responsabilidad:

```text
DB spatial representation
        ↕
VoltStack SpatialValue
```

---

# 55. Querying ≠ conversion

El Spatial Query System no deberá decodificar manualmente resultados binarios.

Eso corresponde a:

```text
Type Conversion
```

---

# 56. Spatial AST

Se introduce una familia:

```text
SpatialExpression
```

dentro del Query AST.

---

# 57. Hierarchy

```text
ExpressionNode
└── SpatialExpression
    ├── SpatialValueExpression
    ├── SpatialConstructorExpression
    ├── SpatialPredicateExpression
    ├── SpatialDistanceExpression
    ├── SpatialTransformExpression
    ├── SpatialEnvelopeExpression
    └── SpatialAggregateExpression
```

---

# 58. SpatialValueExpression

Representa:

```text
geometry column
geometry parameter
geometry expression
computed geometry
```

---

# 59. Spatial constructor

Ejemplo:

```php
Spatial::point(
    longitude: -100.316,
    latitude: 25.686,
    srid: 4326
);
```

---

# 60. Constructor AST

```text
SpatialPointExpression
├── X
├── Y
├── Z?
└── SRID
```

---

# 61. Parameters

Las coordenadas deberán utilizar binding cuando formen parte de una query.

No concatenación SQL.

---

# 62. Query API

Ejemplo:

```php
Store::query()
    ->whereSpatialWithin(
        'location',
        $area
    )
    ->get();
```

---

# 63. Generic expression API

También:

```php
$query->where(
    Spatial::within(
        Spatial::column('location'),
        Spatial::parameter($area)
    )
);
```

---

# 64. Spatial predicates

El sistema deberá modelar al menos:

```text
equals
disjoint
intersects
touches
crosses
within
contains
overlaps
```

según capabilities.

---

# 65. Topological predicates

Jerarquía:

```text
SpatialPredicate
├── SpatialEquals
├── SpatialDisjoint
├── SpatialIntersects
├── SpatialTouches
├── SpatialCrosses
├── SpatialWithin
├── SpatialContains
└── SpatialOverlaps
```

---

# 66. Spatial equals ≠ ordinary equality

No asumir:

```text
geometry_a = geometry_b
```

como equivalente universal a igualdad espacial.

---

# 67. Binary representation equality ≠ topological equality

Dos geometrías pueden tener:

```text
different serialization
```

pero representar una relación espacial equivalente bajo cierta semántica.

---

# 68. Contains

Conceptualmente:

```text
Contains(A, B)
```

pregunta si B se encuentra espacialmente contenido por A según la semántica definida.

---

# 69. Within

Conceptualmente:

```text
Within(A, B)
```

pregunta si A está dentro de B.

---

# 70. Contains/Within duality

Puede existir relación lógica entre ambas operaciones.

El Optimizer podrá reescribirlas únicamente si preserva exactamente la semántica de plataforma.

---

# 71. Intersects

```text
Intersects(A,B)
```

evalúa si las geometrías tienen una intersección espacial según el modelo aplicable.

---

# 72. Disjoint

```text
Disjoint(A,B)
```

indica ausencia de intersección.

---

# 73. Touches

Debe mantener semántica topológica.

No deberá implementarse como una simple comparación de bounding boxes.

---

# 74. Overlaps

Igualmente requiere semántica espacial real.

---

# 75. Bounding-box overlap ≠ geometric overlap

Regla:

> **Una coincidencia de bounding boxes puede ser un filtro preliminar de optimización, pero no reemplaza un predicate topológico cuando la semántica exige exactitud geométrica.**

---

# 76. BoundingBox

Modelo:

```php
final readonly class BoundingBox
{
    public function __construct(
        public float $minX,
        public float $minY,
        public float $maxX,
        public float $maxY,
        public SpatialReferenceId $srid,
    ) {}
}
```

---

# 77. BoundingBox invariants

```text
minX <= maxX
minY <= maxY
```

salvo modelos especializados donde el CRS requiera tratamiento adicional.

---

# 78. BoundingBox ≠ Polygon

Aunque pueda convertirse a polygon:

```text
BoundingBox
≠
Polygon
```

---

# 79. Envelope

Podrá existir:

```text
SpatialEnvelopeExpression
```

para obtener la envolvente espacial.

---

# 80. Bounding-box filtering

Un planner podrá utilizar:

```text
bounding box prefilter
        ↓
exact spatial predicate
```

cuando la plataforma permita hacerlo de manera segura.

---

# 81. Distance

Operación:

```text
SpatialDistanceExpression
```

---

# 82. Example

```php
$query->selectDistance(
    from: 'location',
    to: $target,
    alias: 'distance'
);
```

---

# 83. Distance requires semantics

La distancia necesita conocer:

```text
geometry/geography mode
CRS
units
distance model
```

---

# 84. Euclidean ≠ geodesic

```text
EuclideanDistance
≠
GeodesicDistance
```

---

# 85. Geographic distance

Para coordenadas terrestres:

```text
longitude/latitude
```

una distancia cartesiana simple en grados no equivale a distancia física.

---

# 86. Distance model

```php
enum SpatialDistanceModel
{
    case PLANAR;
    case GEODESIC;
    case PLATFORM_DEFAULT;
}
```

---

# 87. Explicit unit

La API podrá solicitar:

```php
Spatial::distance(
    from: $a,
    to: $b,
    unit: SpatialUnit::METER
);
```

---

# 88. Unit conversion

Sólo deberá realizarse cuando exista información suficiente.

---

# 89. Unknown unit

```text
UNKNOWN UNIT
≠
METERS
```

---

# 90. No magical meters

VoltStack no asumirá metros universalmente.

---

# 91. Distance predicates

Ejemplo:

```php
$query->whereSpatialDistance(
    column: 'location',
    from: $point,
    operator: '<=',
    distance: 5000,
    unit: SpatialUnit::METER
);
```

---

# 92. Near API

Convenience:

```php
Store::query()
    ->near(
        column: 'location',
        point: $point,
        radius: 5,
        unit: SpatialUnit::KILOMETER,
    )
    ->get();
```

---

# 93. Near ≠ generic magic

Internamente deberá producir:

```text
distance predicate
+
explicit spatial semantics
```

---

# 94. Nearest neighbor

El sistema deberá permitir:

```text
ORDER BY spatial distance
LIMIT N
```

de forma semántica.

---

# 95. API

```php
Store::query()
    ->nearestTo(
        column: 'location',
        point: $point,
        limit: 20,
    )
    ->get();
```

---

# 96. Nearest-neighbor strategy

El planner podrá utilizar:

```text
spatial index
KNN index
bounding box
distance sort
platform-native operator
```

según capabilities.

---

# 97. API intent ≠ physical algorithm

```text
nearestTo()
```

describe:

```text
intent
```

no:

```text
implementation
```

---

# 98. Nearest-neighbor exactness

El planner deberá conocer si una estrategia produce:

```text
EXACT
APPROXIMATE
CANDIDATE_ONLY
```

---

# 99. No silent approximation

Si se solicita exactitud:

```text
approximate result
```

no podrá devolverse silenciosamente.

---

# 100. Spatial ordering

Puede existir:

```text
ORDER BY distance(...)
```

---

# 101. Stable ordering

Si dos objetos tienen la misma distancia:

```text
tie-breaker
```

será necesario para pagination/cursor pagination determinista.

---

# 102. Example

```text
ORDER BY
    distance(location, target),
    id
```

---

# 103. Cursor Pagination

Compatible cuando:

```text
distance expression
```

sea:

```text
deterministic enough
stable under requested semantics
serializable
typed
```

---

# 104. Cursor boundary

Puede contener:

```text
(distance, entity_id)
```

---

# 105. Moving target

Si el target cambia:

```text
cursor becomes semantically incompatible
```

---

# 106. Cursor binding

El target deberá formar parte del:

```text
query/cursor fingerprint
```

sin exponer datos sensibles innecesariamente.

---

# 107. Mutable location

Si las entidades cambian de ubicación durante el recorrido:

```text
duplicate/skip behavior
```

puede aparecer.

Cursor pagination no elimina ese problema.

---

# 108. Coordinate transformation

Operación:

```text
SpatialTransformExpression
```

---

# 109. Example

```php
Spatial::transform(
    expression: Spatial::column('location'),
    to: SpatialReferenceId::from(3857)
);
```

---

# 110. Transform ≠ assign SRID

Regla crítica:

```text
TransformCRS
≠
SetSRID
```

---

# 111. Set SRID

Asignar SRID declara:

```text
these coordinates are expressed in CRS X
```

---

# 112. Transform

Transformar significa:

```text
convert coordinates from CRS A to CRS B
```

---

# 113. Example

Incorrecto:

```text
coordinates in EPSG:4326
↓
change metadata to EPSG:3857
```

Eso no transforma las coordenadas.

---

# 114. Semantic safety

VoltStack deberá diferenciar APIs:

```php
Spatial::assignSrid(...)
Spatial::transform(...)
```

---

# 115. Unknown source CRS

Transformar una geometría cuyo CRS es desconocido deberá:

```text
fail
```

por default.

---

# 116. CRS mismatch

Ejemplo:

```text
A → SRID 4326
B → SRID 3857
```

No deberán compararse silenciosamente.

---

# 117. Policies

```php
enum SpatialCrsMismatchPolicy
{
    case REJECT;
    case TRANSFORM_IF_SAFE;
    case REQUIRE_EXPLICIT_TRANSFORM;
}
```

---

# 118. Default

La opción más segura será conceptualmente:

```text
REQUIRE_EXPLICIT_TRANSFORM
```

o rechazo equivalente.

---

# 119. Automatic transform

Si se permite deberá:

- conocer ambos CRS;
- conocer capability;
- conocer target CRS;
- preservar semántica;
- ser visible en diagnostics.

---

# 120. SRID 0 / unknown

No deberá interpretarse automáticamente como:

```text
WGS84
```

---

# 121. WKT

VoltStack podrá soportar:

```text
Well-Known Text
```

como formato de interoperabilidad.

---

# 122. Example

```text
POINT(-100.316 25.686)
```

---

# 123. WKT ≠ Geometry

WKT es una representación.

```text
WKT
≠
Geometry object
```

---

# 124. WKB

También:

```text
Well-Known Binary
```

---

# 125. WKB ≠ internal canonical object

Es formato de serialización/intercambio.

---

# 126. EWKT/EWKB

Podrán ser extensiones capability-aware cuando sean necesarias.

---

# 127. GeoJSON

Podrá integrarse como formato de interoperabilidad.

---

# 128. GeoJSON ≠ Database JSON Type

Aunque GeoJSON utilice JSON:

```text
GeoJSON geometry
≠
generic JSON document
```

---

# 129. GeoJSON conversion

Conceptualmente:

```text
GeoJSON
    ↓
GeoJsonParser
    ↓
SpatialValue
```

---

# 130. JSON Query System separation

El documento:

```text
273_DATABASE_JSON_QUERY_SYSTEM.md
```

no deberá interpretar automáticamente cualquier JSON como geometría.

---

# 131. Explicit conversion

Debe existir intención:

```php
Spatial::fromGeoJson($json);
```

---

# 132. Spatial Schema System

Integración con:

```text
87_DATABASE_SCHEMA_ARCHITECTURE.md
```

---

# 133. Schema API

Ejemplo:

```php
$table->point(
    'location',
    srid: 4326
);
```

---

# 134. Polygon

```php
$table->polygon(
    'service_area',
    srid: 4326
);
```

---

# 135. Generic geometry

```php
$table->geometry(
    'shape',
    type: GeometryType::GEOMETRY,
    srid: 4326,
);
```

---

# 136. Geography

Cuando la plataforma lo permita:

```php
$table->geography(
    'location',
    type: GeometryType::POINT,
    srid: 4326,
);
```

---

# 137. Schema intent

El Schema Model deberá conservar:

```text
logical spatial type
geometry subtype
SRID
dimensions
nullability
index intent
```

---

# 138. Physical representation

El Schema Compiler decidirá la representación física.

---

# 139. Spatial index

API conceptual:

```php
$table->spatialIndex('location');
```

---

# 140. SpatialIndexDefinition

Debe representar:

```text
semantic index intent
```

no:

```text
vendor DDL
```

---

# 141. Index capabilities

Resolver:

```text
supportsSpatialIndex()
supportsSpatialIndexForType()
supportsSpatialIndexForSrid()
supportsSpatialKnn()
```

---

# 142. Index ≠ exact predicate

Un spatial index puede generar candidatos.

La query aún puede requerir:

```text
exact predicate evaluation
```

---

# 143. Two-stage evaluation

```text
Spatial Index
     ↓
Candidate Set
     ↓
Exact Spatial Predicate
```

---

# 144. Optimizer

Podrá producir este plan automáticamente.

---

# 145. Query Optimizer integration

Podrá analizar:

```text
spatial index availability
bounding-box prefilters
constant geometry parameters
distance constraints
nearest-neighbor patterns
redundant transformations
SRID compatibility
```

---

# 146. Redundant transform

Ejemplo:

```text
Transform(Transform(A, B), B)
```

puede simplificarse cuando sea semánticamente seguro.

---

# 147. Transform cancellation

No asumir:

```text
Transform(Transform(A,B),A) = A
```

bit-for-bit.

Las transformaciones pueden introducir pérdida numérica.

---

# 148. Numeric precision

Spatial operations deberán reconocer:

```text
floating-point precision
coordinate precision
database numeric behavior
```

---

# 149. Exact coordinate equality

Puede ser una operación distinta de:

```text
topological equality
```

---

# 150. Tolerance

VoltStack podrá soportar operaciones con tolerance cuando sean semánticamente definidas.

---

# 151. No universal epsilon

No habrá un:

```text
GLOBAL_SPATIAL_EPSILON
```

aplicado silenciosamente.

---

# 152. Buffer

Operación potencial:

```php
Spatial::buffer(
    geometry: $geometry,
    distance: 100,
    unit: SpatialUnit::METER,
);
```

---

# 153. Buffer requires semantics

Especialmente para geography:

```text
buffer
```

puede tener implicaciones geodésicas.

Capability obligatorio.

---

# 154. Centroid

```php
Spatial::centroid($polygon);
```

---

# 155. Area

```php
Spatial::area(
    $polygon,
    unit: SpatialAreaUnit::SQUARE_METER
);
```

---

# 156. Area semantics

```text
planar area
≠
geodesic area
```

---

# 157. Length

Para:

```text
LineString
```

puede existir:

```php
Spatial::length(...);
```

---

# 158. Perimeter

Puede modelarse separadamente cuando corresponda.

---

# 159. Spatial function registry

Funciones avanzadas podrán registrarse mediante:

```text
SpatialFunctionRegistry
```

---

# 160. Registry freeze

Deberá congelarse después del bootstrap.

---

# 161. No FQCN from DB

Ningún valor almacenado deberá poder provocar:

```text
arbitrary class instantiation
```

---

# 162. Platform capabilities

Integración profunda con:

```text
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md
275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 163. Spatial capabilities

Ejemplos:

```text
supportsGeometry()
supportsGeography()
supportsPoint()
supportsLineString()
supportsPolygon()
supportsMultiGeometry()
supportsGeometryCollection()

supportsSrid()
supportsCoordinateTransform()
supportsSpatialIndex()
supportsKnn()

supportsSpatialEquals()
supportsIntersects()
supportsContains()
supportsWithin()
supportsTouches()
supportsCrosses()
supportsOverlaps()

supportsPlanarDistance()
supportsGeodesicDistance()
supportsBuffer()
supportsArea()
supportsCentroid()

supportsWkt()
supportsWkb()
supportsGeoJson()
```

---

# 164. Capability status

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EXTENSION
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 165. REQUIRES_EXTENSION

Es especialmente relevante para spatial.

Ejemplo conceptual:

```text
PostgreSQL
+
PostGIS
```

---

# 166. Database engine ≠ installed spatial extension

VoltStack no asumirá:

```text
PostgreSQL
⇒
PostGIS available
```

---

# 167. SQLite

Igualmente:

```text
SQLite
⇒
all spatial functionality
```

es falso como supuesto arquitectónico.

---

# 168. Runtime capability discovery

El sistema podrá descubrir:

```text
installed extension
extension version
available functions
supported types
```

cuando sea seguro y apropiado.

---

# 169. Version ≠ capability

Regla ya establecida:

```text
Version
≠
Capability
```

---

# 170. MySQL

Tendrá:

```text
MySQLSpatialCapabilityProvider
MySQLSpatialCompiler
```

---

# 171. MariaDB

Tendrá implementación separada:

```text
MariaDBSpatialCapabilityProvider
MariaDBSpatialCompiler
```

---

# 172. No MySQL inheritance assumption

Podrá existir reutilización interna.

Pero:

```text
MariaDB spatial semantics
≠
automatically MySQL spatial semantics
```

---

# 173. PostgreSQL

La arquitectura deberá permitir:

```text
PostgreSQL core
+
PostGIS extension provider
```

---

# 174. PostGIS

Debe tratarse como:

```text
optional platform extension
```

no como dependencia obligatoria del core VoltStack.

---

# 175. SQLite

Podrá integrarse con proveedores espaciales opcionales.

---

# 176. Extension architecture

```text
SpatialExtension
├── PostgisExtension
├── SQLiteSpatialExtension
└── FutureExtension
```

---

# 177. Core independence

```text
VoltStack Database Core
```

no dependerá directamente de ninguna extensión externa concreta.

---

# 178. Platform Spatial Compiler

Jerarquía conceptual:

```text
SpatialCompiler
├── MySQLSpatialCompiler
├── MariaDBSpatialCompiler
├── PostgreSQLSpatialCompiler
└── SQLiteSpatialCompiler
```

---

# 179. Extension compiler

Puede decorarse/extenderse:

```text
PostgreSQLSpatialCompiler
        +
PostgisCompilerExtension
```

---

# 180. Compiler responsibility

Convierte:

```text
Spatial AST
```

a:

```text
platform SQL representation
```

---

# 181. Compiler does not

No deberá:

```text
execute queries
open connections
hydrate entities
perform authorization
select tenants
```

---

# 182. ORM mapping

Ejemplo:

```php
final class Store
{
    #[Column(type: 'spatial.point', srid: 4326)]
    private Point $location;
}
```

---

# 183. Metadata

Entity Metadata deberá conservar:

```text
logical spatial type
SRID
dimension
nullable
conversion strategy
```

---

# 184. Hydration

Flujo:

```text
DB Spatial Value
    ↓
Result System
    ↓
Spatial Type Converter
    ↓
Point / Polygon / ...
    ↓
Entity Hydrator
```

---

# 185. Hydrator does not parse vendor SQL

---

# 186. Dirty tracking

Spatial values deberán preferirse:

```text
immutable
```

para simplificar Change Tracking.

---

# 187. Immutable Point

Ejemplo:

```php
$newLocation = $store->location()
    ->withX(-100.20);
```

en lugar de mutar silenciosamente el objeto managed.

---

# 188. Equality for Change Tracking

No deberá confundirse:

```text
object identity
topological equality
persistence equality
```

---

# 189. Persistence equality

El Type System deberá definir la igualdad relevante para detectar cambios persistentes.

---

# 190. Spatial relationship ≠ ORM relationship

Muy importante:

```text
SpatialRelation
≠
EntityRelationship
```

`contains()` geográfico no es:

```text
hasMany()
```

---

# 191. Query Builder example

```php
$stores = Store::query()
    ->whereSpatialWithin(
        column: 'location',
        geometry: $serviceArea
    )
    ->orderBySpatialDistance(
        column: 'location',
        from: $userLocation
    )
    ->limit(20)
    ->get();
```

---

# 192. Internal AST

```text
SelectQuery
├── From(Store)
├── Predicate
│   └── SpatialWithin
│       ├── Column(location)
│       └── Parameter(serviceArea)
│
├── Order
│   └── SpatialDistance
│       ├── Column(location)
│       └── Parameter(userLocation)
│
└── Limit(20)
```

---

# 193. No spatial SQL in Query Builder

Query Builder genera AST.

No:

```text
ST_Within(...)
ST_Distance(...)
vendor operator
```

directamente.

---

# 194. Semantic Analyzer

Validará:

```text
operand types
geometry subtype
CRS
SRID
dimensions
operation compatibility
distance model
units
capabilities
```

---

# 195. Example invalid query

```text
Contains(Point, Polygon)
```

puede ser semánticamente distinto o imposible respecto a:

```text
Contains(Polygon, Point)
```

---

# 196. Argument order matters

El analyzer no podrá intercambiar argumentos arbitrariamente.

---

# 197. Geometry type inference

Si:

```text
location: Point<4326>
```

metadata permite inferir:

```text
SpatialType = POINT
SRID = 4326
```

---

# 198. Parameter validation

Si `$polygon` tiene:

```text
SRID 3857
```

y la columna:

```text
SRID 4326
```

el analyzer detectará mismatch antes de ejecutar cuando sea posible.

---

# 199. Spatial Query Context

Podrá contener:

```text
default CRS policy
distance model
unit policy
precision policy
extension requirements
resource budget
```

---

# 200. No global mutable spatial context

Especialmente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 201. Spatial context scope

Debe ser:

```text
query/request/operation scoped
```

---

# 202. Multitenancy

Spatial queries deberán conservar:

```text
Tenant Query Context
```

---

# 203. Example

```text
tenant predicate
AND
spatial predicate
```

---

# 204. Security rule

Nunca:

```text
spatial filtering
```

antes de aplicar tenant/security scope si ello pudiera filtrar información entre tenants.

---

# 205. Spatial data ≠ authorization

Estar físicamente dentro de una región no concede acceso automáticamente.

---

# 206. Authorization integration

Podrá existir una policy que utilice spatial predicates.

Pero:

```text
Spatial Engine
≠
Authorization Engine
```

---

# 207. Sharding

Spatial data puede utilizarse para routing.

---

# 208. Geographic sharding

Ejemplo conceptual:

```text
world
├── region A → shard 1
├── region B → shard 2
└── region C → shard 3
```

---

# 209. Spatial shard routing

Requiere una función determinista:

```text
Geometry
    ↓
ShardRoutingKey
```

---

# 210. Query overlaps multiple regions

Puede requerir:

```text
multi-shard query
```

---

# 211. No fake single shard

Si un polygon cruza varios shards:

```text
Shard A
+
Shard B
+
Shard C
```

el planner no deberá seleccionar arbitrariamente uno.

---

# 212. Distributed spatial query

Arquitectura:

```text
Spatial Query
    ↓
Partition Routing
    ↓
Candidate Shards
    ↓
Local Spatial Queries
    ↓
Merge
    ↓
Global Result
```

---

# 213. Global nearest-neighbor

Especialmente complejo.

Cada shard puede devolver:

```text
local nearest N
```

y el coordinator deberá realizar:

```text
global distance merge
```

---

# 214. Local N may be insufficient

El Distributed Planner deberá demostrar que el algoritmo produce el resultado global requerido.

---

# 215. No fake correctness

Si no puede demostrarlo:

```text
UNSUPPORTED
```

o plan especializado.

---

# 216. Pagination

Spatial predicates son compatibles con Pagination.

---

# 217. Offset pagination

No presenta una semántica espacial especial por sí misma.

---

# 218. Distance pagination

Si ordenamos por distancia:

```text
distance ASC
```

deberá agregarse tie-breaker.

---

# 219. Cursor Pagination

Ejemplo:

```text
(distance, id)
```

como boundary.

---

# 220. Distance precision

El cursor deberá utilizar la representación canónica del tipo producido por el Query Engine.

---

# 221. Do not recalculate in PHP

El boundary no deberá depender de recalcular una distancia con una fórmula PHP diferente de la utilizada por la base.

---

# 222. Query fingerprint

Debe incluir:

```text
target geometry identity/fingerprint
CRS
distance model
unit
```

cuando afecten el orden.

---

# 223. Chunk Processing

Spatial queries podrán procesarse mediante chunks.

---

# 224. Mutation caveat

Si el chunk ordering depende de ubicación y el procesamiento modifica esa ubicación:

```text
ordering instability
```

deberá declararse.

---

# 225. Lazy Collection

Puede consumir spatial query results normalmente.

No cambia resource semantics.

---

# 226. Bulk updates

Podrán modificar spatial columns.

---

# 227. Example

```php
Store::query()
    ->whereSpatialWithin('location', $area)
    ->bulkUpdate([
        'region_id' => 5,
    ]);
```

---

# 228. IdentityMap

Las reglas de Bulk Update permanecen.

No se sincronizarán entidades managed mágicamente.

---

# 229. Spatial transformation update

Ejemplo conceptual:

```text
location =
Transform(location, 3857)
```

debe ser una operación explícita y capability-aware.

---

# 230. Schema Migration

Cambiar:

```text
SRID
```

puede implicar mucho más que cambiar metadata.

---

# 231. SRID migration

Debe distinguir:

```text
change declared SRID
```

de:

```text
transform stored coordinates
```

---

# 232. Dangerous migration

Nunca convertir:

```text
SRID 4326 → 3857
```

cambiando sólo metadata si los valores necesitan transformación real.

---

# 233. Migration planner

Deberá clasificar:

```text
metadata-only
data-transforming
index-rebuilding
unsupported
```

---

# 234. Zero-downtime migration

Transformar millones de geometrías puede requerir:

```text
expand
backfill
dual compatibility
cutover
contract
```

---

# 235. Spatial index migration

Crear un spatial index grande puede tener impacto operacional.

Debe pasar por Migration Safety System.

---

# 236. Security

Los datos geográficos pueden ser sensibles.

---

# 237. Sensitive location data

El sistema deberá integrarse con:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
```

---

# 238. Debugging

No imprimir por default:

```text
exact user coordinates
```

cuando el campo esté clasificado como sensible.

---

# 239. Telemetry

No utilizar:

```text
latitude
longitude
WKT
GeoJSON
```

como labels.

---

# 240. Audit

Puede registrar:

```text
spatial operation
field
geometry type
SRID
operation class
```

sin registrar necesariamente las coordenadas.

---

# 241. Input validation

Coordenadas externas deberán validarse.

---

# 242. Geographic ranges

Para APIs explícitamente longitude/latitude bajo un CRS compatible podrán aplicarse reglas como:

```text
longitude range
latitude range
```

---

# 243. But

No aplicar rangos lat/lon a coordenadas proyectadas.

---

# 244. CRS-aware validation

```text
CoordinateValidation
=
f(CRS, coordinate)
```

---

# 245. Geometry complexity attacks

Un input puede contener:

```text
millions of points
extreme nesting
huge polygons
many rings
```

---

# 246. Resource limits

```text
max geometry bytes
max coordinates
max rings
max collection members
max nesting depth
max spatial operation cost
```

---

# 247. Spatial bombs

Geometrías muy complejas pueden provocar operaciones extremadamente costosas.

Resource Governance deberá poder rechazarlas.

---

# 248. User-defined WKT

Debe parsearse mediante parser seguro.

Nunca concatenarse directamente a SQL.

---

# 249. Parameter binding

Spatial values deberán utilizar:

```text
typed parameter binding
```

cuando la plataforma lo permita.

---

# 250. Raw spatial expressions

Podrá existir escape hatch:

```php
Spatial::raw(...)
```

pero será:

```text
explicit
restricted
auditable
platform-specific
```

---

# 251. SQL injection

Raw spatial SQL seguirá las mismas reglas del Raw Expression System.

---

# 252. Serialization safety

No utilizar:

```php
unserialize($databaseValue)
```

para construir geometrías.

---

# 253. GeoJSON safety

GeoJSON input deberá:

```text
parse
validate structure
validate type
validate coordinate count
validate depth
validate CRS policy
```

---

# 254. Performance architecture

Spatial queries pueden ser extremadamente costosas.

---

# 255. Cost factors

```text
geometry complexity
coordinate count
index availability
predicate type
distance model
CRS transformation
number of candidate rows
distribution
result cardinality
```

---

# 256. Query Cost Hints

Spatial Planner podrá aportar:

```text
LOW
MEDIUM
HIGH
VERY_HIGH
UNKNOWN
```

o un modelo de coste más estructurado.

---

# 257. UNKNOWN cost ≠ cheap

---

# 258. Resource Governance

Puede aplicar:

```text
ALLOW
WARN
THROTTLE
REJECT
```

según operación/contexto.

---

# 259. Full-table spatial distance

Ejemplo:

```text
calculate geodesic distance
for 50 million rows
without index/prefilter
```

puede ser bloqueado por policy.

---

# 260. Query timeout

Se integrará con:

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
```

---

# 261. Cancellation

Spatial operations deberán responder a las capacidades normales de cancelación del Execution Engine.

---

# 262. Circuit breaker

No será responsabilidad del Spatial System.

---

# 263. Failover

Spatial semantics deberán conservarse tras failover.

Un replica sin la extensión requerida:

```text
is not eligible
```

para esa query.

---

# 264. Replica capability

Elegibilidad:

```text
healthy
AND
fresh enough
AND
required spatial capabilities available
```

---

# 265. Load balancing

Sólo entre endpoints spatial-capable.

---

# 266. Capability-aware routing

Una query podrá requerir:

```text
PostGIS
```

y por tanto el router deberá evitar endpoints que no tengan esa capability.

---

# 267. Cache

Result Cache podrá almacenar resultados spatial cuando:

```text
serialization
identity
consistency
security
```

sean válidos.

---

# 268. Cache key

Deberá incluir semántica relevante:

```text
geometry fingerprint
SRID
distance model
unit
query semantics
platform/metadata generation
tenant/shard
```

---

# 269. Raw WKT as cache key

No será recomendable.

Preferir:

```text
canonical spatial fingerprint
```

---

# 270. Canonicalization

Debe realizarse con cuidado.

---

# 271. Equivalent geometries

```text
topologically equivalent
```

no necesariamente deben producir la misma cache key.

---

# 272. Cache identity

Puede utilizar:

```text
canonical persistence representation
```

sin intentar resolver equivalencia topológica costosa.

---

# 273. Metadata Cache

Puede almacenar:

```text
SRID metadata
type mappings
capability descriptors
spatial index metadata
```

---

# 274. Entity Cache

Puede almacenar SpatialValue serializable/canonical.

Nunca connection-bound objects.

---

# 275. Telemetry architecture

Eventos conceptuales:

```text
SpatialQueryPlanned
SpatialCapabilityResolved
SpatialIndexSelected
SpatialTransformPlanned
SpatialQueryExecuted
SpatialQueryFailed
SpatialResourceLimitTriggered
```

---

# 276. Avoid event explosion

No generar eventos por cada:

```text
coordinate
vertex
ring
```

---

# 277. Metrics

Ejemplos:

```text
database_spatial_queries_total
database_spatial_query_duration
database_spatial_failures_total
database_spatial_transform_total
database_spatial_index_usage_total
database_spatial_resource_rejections_total
```

---

# 278. Bounded labels

Permitidas:

```text
platform
operation
geometry_type
capability_status
result_status
```

---

# 279. High-cardinality forbidden

No:

```text
coordinates
WKT
GeoJSON
entity ID
tenant ID
user location
```

---

# 280. Query Profiler

Podrá mostrar:

```text
Spatial Predicate
Geometry Types
SRID
Transformations
Spatial Index
Distance Model
Estimated Cost
Rows Examined
Duration
```

---

# 281. Developer Debug Toolbar

Ejemplo:

```text
Spatial Query
────────────────────────────

Operation:
  WITHIN

Column:
  stores.location

Type:
  POINT

Column SRID:
  4326

Parameter Type:
  POLYGON

Parameter SRID:
  4326

Spatial Index:
  stores_location_spatial_idx

Prefilter:
  bounding-box

Exact Predicate:
  yes

Platform:
  PostgreSQL

Extension:
  PostGIS

Duration:
  8.4 ms
```

---

# 282. Redaction

Una sensitive geometry podrá aparecer:

```text
Parameter:
  [SPATIAL VALUE REDACTED]
```

---

# 283. Spatial Explain

API conceptual:

```php
$query->explainSpatial();
```

---

# 284. Explain output

```text
Spatial Plan
├── Operation: nearest-neighbor
├── Source: stores.location
├── Type: Point
├── SRID: 4326
├── Distance: geodesic
├── Target: parameter
├── Index: spatial
├── Candidate Strategy: indexed
├── Exact Evaluation: required
├── Order: distance ASC, id ASC
└── Capability: supported
```

---

# 285. Explain ≠ SQL dump

---

# 286. Error architecture

```text
DatabaseException
└── SpatialException
    ├── SpatialTypeException
    ├── SpatialCoordinateException
    ├── SpatialGeometryException
    ├── SpatialValidationException
    ├── SpatialReferenceException
    ├── SpatialSridMismatchException
    ├── SpatialTransformationException
    ├── SpatialPredicateException
    ├── SpatialDistanceException
    ├── SpatialCapabilityException
    ├── SpatialIndexException
    ├── SpatialCompilationException
    ├── SpatialResourceException
    ├── SpatialSecurityException
    └── SpatialDistributionException
```

---

# 287. SpatialTypeException

Tipos incompatibles.

---

# 288. SpatialCoordinateException

Coordenadas inválidas.

---

# 289. SpatialGeometryException

Estructura geométrica inválida.

---

# 290. SpatialReferenceException

CRS desconocido/inválido.

---

# 291. SpatialSridMismatchException

Operación entre SRIDs incompatibles.

---

# 292. SpatialTransformationException

Transformación no soportada/fallida.

---

# 293. SpatialPredicateException

Predicate no válido para los operandos.

---

# 294. SpatialDistanceException

Distancia no representable bajo la semántica solicitada.

---

# 295. SpatialCapabilityException

Feature requerida no disponible.

---

# 296. SpatialIndexException

Problema de índice spatial.

---

# 297. SpatialResourceException

Presupuesto de complejidad excedido.

---

# 298. SpatialSecurityException

Operación o dato bloqueado por security policy.

---

# 299. SpatialDistributionException

No puede producirse un plan distribuido correcto.

---

# 300. Testing architecture

El sistema requerirá:

```text
unit tests
value-object tests
type tests
AST tests
semantic tests
CRS tests
compiler tests
platform conformance tests
index tests
ORM tests
security tests
resource tests
distribution tests
performance tests
persistent runtime tests
```

---

# 301. Point tests

Probar:

```text
XY
XYZ
XYM
XYZM
empty point
invalid coordinates
different SRIDs
```

cuando sean soportados.

---

# 302. LineString tests

```text
minimum points
empty
invalid sequence
mixed dimensions
```

---

# 303. Polygon tests

```text
closed rings
holes
invalid rings
self-intersection
empty polygon
```

---

# 304. MultiGeometry tests

Probar:

```text
MultiPoint
MultiLineString
MultiPolygon
```

---

# 305. GeometryCollection tests

Especialmente:

```text
heterogeneous members
SRID consistency
dimension consistency
```

---

# 306. CRS tests

```text
same SRID
different SRID
unknown SRID
invalid SRID
explicit transformation
```

---

# 307. Critical transformation test

Verificar:

```text
assign SRID
≠
transform coordinates
```

---

# 308. Predicate conformance

Corpus equivalente para:

```text
equals
contains
within
intersects
touches
crosses
overlaps
disjoint
```

---

# 309. Distance tests

```text
planar
geodesic
units
same point
large distance
CRS mismatch
```

---

# 310. Nearest-neighbor tests

Comprobar:

```text
correct global ordering
ties
limit
index/no index
```

---

# 311. Pagination tests

```text
distance ordering
tie-breakers
mutable positions
```

---

# 312. Cursor tests

Validar:

```text
distance boundary
query fingerprint
target binding
SRID binding
```

---

# 313. Schema tests

```text
point column
polygon column
SRID
spatial index
platform compatibility
```

---

# 314. Migration tests

```text
add spatial column
change spatial type
change SRID
transform data
create spatial index
drop spatial index
```

---

# 315. Security tests

Inputs:

```text
huge WKT
malformed WKT
huge GeoJSON
deep GeometryCollection
millions of coordinates
raw SQL payload
invalid CRS
```

---

# 316. Resource tests

Validar límites:

```text
geometry bytes
vertices
rings
nesting
query cost
result size
```

---

# 317. Multitenancy tests

Spatial predicates nunca deberán eliminar tenant scoping.

---

# 318. Sharding tests

```text
single shard
multi-shard polygon
global nearest neighbor
shard capability mismatch
```

---

# 319. Replica tests

Una replica sin spatial capability no deberá recibir la query.

---

# 320. Persistent runtime tests

Request A:

```text
tenant A
SRID 4326
sensitive location
```

Request B:

```text
tenant B
different spatial query
```

No deberá existir:

```text
geometry leakage
CRS leakage
tenant leakage
parameter leakage
```

---

# 321. Performance benchmarks

Benchmark corpus:

```text
point lookup
within polygon
intersects polygon
distance search
nearest neighbor
large polygon
many small polygons
indexed/unindexed
coordinate transformation
distributed query
```

---

# 322. Proposed directory structure

```text
src/Quantum/Database/Spatial/
│
├── Contract/
│   ├── SpatialValue.php
│   ├── Geometry.php
│   ├── SpatialCompiler.php
│   ├── SpatialCapabilityProvider.php
│   ├── SpatialTransformer.php
│   └── GeometryValidator.php
│
├── Value/
│   ├── Coordinate.php
│   ├── Point.php
│   ├── LineString.php
│   ├── LinearRing.php
│   ├── Polygon.php
│   ├── MultiPoint.php
│   ├── MultiLineString.php
│   ├── MultiPolygon.php
│   ├── GeometryCollection.php
│   └── BoundingBox.php
│
├── Reference/
│   ├── SpatialReferenceId.php
│   ├── CoordinateReferenceSystem.php
│   ├── CoordinateSystemType.php
│   ├── CoordinateDimension.php
│   ├── AxisOrder.php
│   ├── SpatialUnit.php
│   ├── SpatialAreaUnit.php
│   └── SpatialReferenceRegistry.php
│
├── Type/
│   ├── GeometryType.php
│   ├── SpatialLogicalType.php
│   ├── GeometryTypeHandler.php
│   ├── PointTypeHandler.php
│   ├── PolygonTypeHandler.php
│   └── GeographyTypeHandler.php
│
├── AST/
│   ├── SpatialExpression.php
│   ├── SpatialValueExpression.php
│   ├── SpatialConstructorExpression.php
│   ├── SpatialPredicateExpression.php
│   ├── SpatialDistanceExpression.php
│   ├── SpatialTransformExpression.php
│   ├── SpatialEnvelopeExpression.php
│   └── SpatialAggregateExpression.php
│
├── Predicate/
│   ├── SpatialPredicate.php
│   ├── SpatialEquals.php
│   ├── SpatialDisjoint.php
│   ├── SpatialIntersects.php
│   ├── SpatialTouches.php
│   ├── SpatialCrosses.php
│   ├── SpatialWithin.php
│   ├── SpatialContains.php
│   └── SpatialOverlaps.php
│
├── Function/
│   ├── SpatialDistance.php
│   ├── SpatialBuffer.php
│   ├── SpatialArea.php
│   ├── SpatialLength.php
│   ├── SpatialCentroid.php
│   ├── SpatialEnvelope.php
│   └── SpatialFunctionRegistry.php
│
├── Distance/
│   ├── SpatialDistanceModel.php
│   ├── SpatialDistancePolicy.php
│   └── SpatialDistanceResult.php
│
├── Transformation/
│   ├── SpatialTransform.php
│   ├── SpatialTransformPlan.php
│   ├── SpatialCrsMismatchPolicy.php
│   └── SpatialTransformationPlanner.php
│
├── Serialization/
│   ├── WktParser.php
│   ├── WktWriter.php
│   ├── WkbReader.php
│   ├── WkbWriter.php
│   ├── GeoJsonParser.php
│   └── GeoJsonWriter.php
│
├── Index/
│   ├── SpatialIndexDefinition.php
│   ├── SpatialIndexMetadata.php
│   ├── SpatialIndexStrategy.php
│   └── SpatialIndexResolver.php
│
├── Capability/
│   ├── SpatialCapability.php
│   ├── SpatialCapabilities.php
│   ├── SpatialCapabilityStatus.php
│   └── SpatialCapabilityResolver.php
│
├── Semantic/
│   ├── SpatialSemanticAnalyzer.php
│   ├── SpatialValidationResult.php
│   └── SpatialSemanticProfile.php
│
├── Planning/
│   ├── SpatialQueryPlan.php
│   ├── SpatialPredicatePlan.php
│   ├── SpatialNearestNeighborPlan.php
│   └── SpatialDistributedPlan.php
│
├── Compiler/
│   ├── AbstractSpatialCompiler.php
│   ├── MySQLSpatialCompiler.php
│   ├── MariaDBSpatialCompiler.php
│   ├── PostgreSQLSpatialCompiler.php
│   └── SQLiteSpatialCompiler.php
│
├── Extension/
│   ├── SpatialExtension.php
│   ├── SpatialExtensionRegistry.php
│   ├── Postgis/
│   │   ├── PostgisExtension.php
│   │   ├── PostgisCapabilityProvider.php
│   │   └── PostgisCompilerExtension.php
│   └── SQLite/
│       └── SQLiteSpatialExtension.php
│
├── Security/
│   ├── SpatialSecurityPolicy.php
│   ├── SpatialInputLimits.php
│   └── SpatialRedactor.php
│
├── Diagnostics/
│   ├── SpatialDiagnostics.php
│   ├── SpatialExplain.php
│   └── SpatialPlanFormatter.php
│
├── Telemetry/
│   └── SpatialTelemetry.php
│
└── Exception/
    ├── SpatialException.php
    ├── SpatialTypeException.php
    ├── SpatialCoordinateException.php
    ├── SpatialGeometryException.php
    ├── SpatialValidationException.php
    ├── SpatialReferenceException.php
    ├── SpatialSridMismatchException.php
    ├── SpatialTransformationException.php
    ├── SpatialPredicateException.php
    ├── SpatialDistanceException.php
    ├── SpatialCapabilityException.php
    ├── SpatialIndexException.php
    ├── SpatialCompilationException.php
    ├── SpatialResourceException.php
    ├── SpatialSecurityException.php
    └── SpatialDistributionException.php
```

---

# 323. Architectural invariants

## DB-SPATIAL-001

Spatial será una extensión del Query Engine.

## DB-SPATIAL-002

Spatial no tendrá Execution Engine independiente.

## DB-SPATIAL-003

Spatial utilizará Query AST.

## DB-SPATIAL-004

Spatial añadirá Spatial AST tipado.

## DB-SPATIAL-005

Spatial Query Builder no generará SQL.

## DB-SPATIAL-006

Spatial Compiler no ejecutará queries.

## DB-SPATIAL-007

Driver no interpretará Spatial AST.

## DB-SPATIAL-008

Connection no conocerá geometrías de dominio.

## DB-SPATIAL-009

Geometry será distinta de Geography.

## DB-SPATIAL-010

Coordinate será distinta de Point.

## DB-SPATIAL-011

CRS será distinto de Coordinate.

## DB-SPATIAL-012

SRID será distinto de unit.

## DB-SPATIAL-013

Coordinates sin CRS no adquirirán semántica geográfica automáticamente.

## DB-SPATIAL-014

SRID desconocido no será tratado como 4326.

## DB-SPATIAL-015

Latitude/longitude no serán intercambiados silenciosamente.

## DB-SPATIAL-016

X/Y no serán interpretados universalmente como lat/lon.

## DB-SPATIAL-017

Z no será interpretado automáticamente como elevation.

## DB-SPATIAL-018

M no será interpretado automáticamente como time.

## DB-SPATIAL-019

SQL NULL será distinto de empty geometry.

## DB-SPATIAL-020

GeometryCollection será distinta de MultiGeometry.

## DB-SPATIAL-021

Structural validity será distinta de topological validity.

## DB-SPATIAL-022

Framework validation será distinta de DB validation.

## DB-SPATIAL-023

Spatial types reutilizarán Type Registry.

## DB-SPATIAL-024

Spatial no creará un Type System paralelo.

## DB-SPATIAL-025

Logical spatial type será distinto de physical DB type.

## DB-SPATIAL-026

Value Conversion manejará DB ↔ SpatialValue.

## DB-SPATIAL-027

Hydrator no parseará vendor spatial SQL.

## DB-SPATIAL-028

Spatial predicates serán AST.

## DB-SPATIAL-029

Spatial parameters serán typed/bound.

## DB-SPATIAL-030

Spatial equality será distinta de ordinary SQL equality.

## DB-SPATIAL-031

Binary equality será distinta de topological equality.

## DB-SPATIAL-032

Bounding-box overlap será distinto de exact geometric overlap.

## DB-SPATIAL-033

Bounding box podrá usarse como prefilter.

## DB-SPATIAL-034

Exact predicate deberá conservarse cuando sea requerido.

## DB-SPATIAL-035

Distance requerirá semántica explícita.

## DB-SPATIAL-036

Euclidean distance será distinta de geodesic distance.

## DB-SPATIAL-037

Degrees no serán tratados automáticamente como meters.

## DB-SPATIAL-038

Unknown unit será distinta de meters.

## DB-SPATIAL-039

Near será convenience API sobre spatial semantics.

## DB-SPATIAL-040

Nearest-neighbor API no definirá algoritmo físico.

## DB-SPATIAL-041

Approximate nearest-neighbor no será presentado como exacto.

## DB-SPATIAL-042

Distance ordering deberá admitir tie-breaker.

## DB-SPATIAL-043

Cursor distance boundary utilizará valores canónicos.

## DB-SPATIAL-044

Cursor no recalculará distancia mediante semántica distinta.

## DB-SPATIAL-045

Target geometry formará parte de cursor/query binding cuando corresponda.

## DB-SPATIAL-046

Mutable locations podrán invalidar traversal stability.

## DB-SPATIAL-047

Transform será distinto de Assign SRID.

## DB-SPATIAL-048

Assign SRID no modificará coordenadas.

## DB-SPATIAL-049

Transform deberá modificar coordenadas según CRS.

## DB-SPATIAL-050

Unknown source CRS no será transformado silenciosamente.

## DB-SPATIAL-051

SRID mismatch será detectado.

## DB-SPATIAL-052

Automatic CRS transformation requerirá policy explícita.

## DB-SPATIAL-053

WKT será distinto de Geometry.

## DB-SPATIAL-054

WKB será distinto de Geometry.

## DB-SPATIAL-055

GeoJSON será distinto de generic JSON.

## DB-SPATIAL-056

GeoJSON conversion será explícita.

## DB-SPATIAL-057

JSON Query System no interpretará JSON arbitrario como spatial.

## DB-SPATIAL-058

Schema Model conservará spatial semantics.

## DB-SPATIAL-059

Schema Compiler decidirá physical representation.

## DB-SPATIAL-060

Spatial index será semantic intent.

## DB-SPATIAL-061

Spatial index no garantizará exact predicate.

## DB-SPATIAL-062

Optimizer podrá producir candidate + exact evaluation.

## DB-SPATIAL-063

Optimizer preservará CRS semantics.

## DB-SPATIAL-064

Optimizer no eliminará transformaciones inseguramente.

## DB-SPATIAL-065

Coordinate transformations podrán introducir pérdida numérica.

## DB-SPATIAL-066

No existirá global implicit spatial epsilon.

## DB-SPATIAL-067

Buffer será capability-aware.

## DB-SPATIAL-068

Area tendrá planar/geodesic semantics.

## DB-SPATIAL-069

Length tendrá spatial semantics explícitas.

## DB-SPATIAL-070

Spatial Function Registry será congelable.

## DB-SPATIAL-071

DB values no instanciarán FQCN arbitrarias.

## DB-SPATIAL-072

Spatial capabilities gobernarán features.

## DB-SPATIAL-073

Version será distinta de capability.

## DB-SPATIAL-074

Database engine será distinto de installed spatial extension.

## DB-SPATIAL-075

PostgreSQL no implicará PostGIS.

## DB-SPATIAL-076

SQLite no implicará full spatial capability.

## DB-SPATIAL-077

REQUIRES_EXTENSION será estado representable.

## DB-SPATIAL-078

UNKNOWN capability será distinta de SUPPORTED.

## DB-SPATIAL-079

MySQL y MariaDB tendrán capability providers independientes.

## DB-SPATIAL-080

PostGIS será extensión opcional.

## DB-SPATIAL-081

VoltStack core no dependerá obligatoriamente de PostGIS.

## DB-SPATIAL-082

Spatial compilers serán platform-aware.

## DB-SPATIAL-083

Spatial compiler no hará authorization.

## DB-SPATIAL-084

Spatial compiler no hará tenant resolution.

## DB-SPATIAL-085

Spatial compiler no abrirá conexiones.

## DB-SPATIAL-086

ORM mapping conservará SRID.

## DB-SPATIAL-087

Spatial values preferirán inmutabilidad.

## DB-SPATIAL-088

Object identity será distinta de spatial equality.

## DB-SPATIAL-089

Spatial equality será distinta de persistence equality.

## DB-SPATIAL-090

Spatial relationship será distinta de ORM relationship.

## DB-SPATIAL-091

Query Builder producirá Spatial AST.

## DB-SPATIAL-092

Semantic Analyzer validará geometry types.

## DB-SPATIAL-093

Semantic Analyzer validará SRID.

## DB-SPATIAL-094

Semantic Analyzer validará dimensions.

## DB-SPATIAL-095

Semantic Analyzer validará operation compatibility.

## DB-SPATIAL-096

Spatial context será scoped.

## DB-SPATIAL-097

No existirá global mutable SpatialContext.

## DB-SPATIAL-098

Tenant scope será preservado.

## DB-SPATIAL-099

Spatial predicate no sustituirá tenant predicate.

## DB-SPATIAL-100

Spatial data no concederá authorization automáticamente.

## DB-SPATIAL-101

Spatial Engine será distinto de Authorization Engine.

## DB-SPATIAL-102

Geographic sharding será explícito.

## DB-SPATIAL-103

Polygon multi-region podrá requerir multi-shard.

## DB-SPATIAL-104

Multi-shard query no será reducida falsamente a un shard.

## DB-SPATIAL-105

Distributed spatial query requerirá plan explícito.

## DB-SPATIAL-106

Global nearest-neighbor requerirá merge correcto.

## DB-SPATIAL-107

Local nearest-neighbor no implicará global nearest-neighbor.

## DB-SPATIAL-108

Unsupported distributed semantics serán rechazadas.

## DB-SPATIAL-109

Spatial queries serán compatibles con Pagination.

## DB-SPATIAL-110

Distance pagination requerirá stable ordering.

## DB-SPATIAL-111

Spatial Cursor Pagination conservará target semantics.

## DB-SPATIAL-112

Chunking no eliminará spatial ordering mutation risks.

## DB-SPATIAL-113

Lazy Collection no cambiará spatial resource semantics.

## DB-SPATIAL-114

Bulk spatial updates seguirán Bulk Update rules.

## DB-SPATIAL-115

Bulk spatial update no sincronizará IdentityMap mágicamente.

## DB-SPATIAL-116

SRID migration será distinta de coordinate transformation.

## DB-SPATIAL-117

Migration no cambiará sólo metadata cuando se requiera transformación.

## DB-SPATIAL-118

Large spatial migrations pasarán Migration Safety.

## DB-SPATIAL-119

Spatial index creation estará sujeta a operational safety.

## DB-SPATIAL-120

Sensitive location data podrá clasificarse.

## DB-SPATIAL-121

Sensitive coordinates serán redactadas en debugging.

## DB-SPATIAL-122

Coordinates no serán telemetry labels.

## DB-SPATIAL-123

WKT no será telemetry label.

## DB-SPATIAL-124

GeoJSON no será telemetry label.

## DB-SPATIAL-125

Audit podrá omitir exact coordinates.

## DB-SPATIAL-126

Coordinate validation será CRS-aware.

## DB-SPATIAL-127

Lat/lon range validation no se aplicará universalmente.

## DB-SPATIAL-128

Geometry complexity tendrá resource limits.

## DB-SPATIAL-129

WKT input será parseado de forma segura.

## DB-SPATIAL-130

Spatial values utilizarán parameter binding.

## DB-SPATIAL-131

Raw spatial expressions serán escape hatch explícito.

## DB-SPATIAL-132

Raw spatial expressions estarán sujetos a SQL security.

## DB-SPATIAL-133

Unsafe unserialize no será utilizado.

## DB-SPATIAL-134

GeoJSON tendrá input limits.

## DB-SPATIAL-135

Spatial cost UNKNOWN no será considerado cheap.

## DB-SPATIAL-136

Resource Governance podrá bloquear queries espaciales costosas.

## DB-SPATIAL-137

Spatial queries utilizarán query timeout/cancellation.

## DB-SPATIAL-138

Circuit breaker seguirá siendo responsabilidad de resilience layer.

## DB-SPATIAL-139

Replica eligibility incluirá spatial capabilities.

## DB-SPATIAL-140

Load balancing sólo elegirá endpoints compatibles.

## DB-SPATIAL-141

Failover no degradará spatial semantics silenciosamente.

## DB-SPATIAL-142

Result Cache podrá almacenar spatial results bajo contrato.

## DB-SPATIAL-143

Spatial cache key incluirá semántica relevante.

## DB-SPATIAL-144

Topological equivalence no será usada automáticamente para cache identity.

## DB-SPATIAL-145

Metadata Cache podrá almacenar spatial metadata.

## DB-SPATIAL-146

Entity Cache no almacenará connection-bound spatial objects.

## DB-SPATIAL-147

Telemetry será bounded.

## DB-SPATIAL-148

Telemetry no generará evento por vertex.

## DB-SPATIAL-149

Profiler reconocerá spatial operations.

## DB-SPATIAL-150

Debug Toolbar redactará sensitive geometries.

## DB-SPATIAL-151

Spatial Explain describirá semántica.

## DB-SPATIAL-152

Spatial Explain será distinto de SQL dump.

## DB-SPATIAL-153

Spatial errors tendrán jerarquía específica.

## DB-SPATIAL-154

SRID mismatch será error distinguible.

## DB-SPATIAL-155

Capability failure será distinta de invalid geometry.

## DB-SPATIAL-156

Resource failure será distinta de security failure.

## DB-SPATIAL-157

Distribution failure será explícita.

## DB-SPATIAL-158

Platform conformance será testeada.

## DB-SPATIAL-159

Spatial predicates serán testeados semánticamente.

## DB-SPATIAL-160

Tests no compararán únicamente SQL generado.

## DB-SPATIAL-161

CRS transformations serán testeadas.

## DB-SPATIAL-162

Assign SRID vs Transform será test obligatorio.

## DB-SPATIAL-163

Distance units serán testeadas.

## DB-SPATIAL-164

Nearest-neighbor correctness será testeada.

## DB-SPATIAL-165

Spatial index behavior será testeado.

## DB-SPATIAL-166

Pagination por distancia será testeada.

## DB-SPATIAL-167

Cursor distance semantics serán testeadas.

## DB-SPATIAL-168

Spatial migrations serán testeadas.

## DB-SPATIAL-169

Geometry complexity attacks serán testeados.

## DB-SPATIAL-170

Tenant isolation será testeada.

## DB-SPATIAL-171

Distributed spatial routing será testeado.

## DB-SPATIAL-172

Replica capability routing será testeado.

## DB-SPATIAL-173

Persistent worker isolation será testeada.

## DB-SPATIAL-174

Spatial AST será immutable o tratado como immutable tras planificación.

## DB-SPATIAL-175

Spatial planning será deterministic bajo el mismo contexto.

## DB-SPATIAL-176

No se fabricará portability mediante aproximaciones ocultas.

## DB-SPATIAL-177

Platform extensions permanecerán aisladas.

## DB-SPATIAL-178

Spatial API será independiente de HTTP.

## DB-SPATIAL-179

Spatial API será independiente de map rendering.

## DB-SPATIAL-180

VoltStack preservará CRS, SRID, tipo y semántica espacial desde el modelo hasta la ejecución.

---

# 324. Modelo formal

Sea una geometría:

```text
G = (T, C, S, D)
```

donde:

```text
T = geometry type
C = coordinates
S = spatial reference
D = coordinate dimension
```

Dos geometrías:

```text
G₁
G₂
```

no deberán considerarse directamente comparables si:

```text
S₁ ≠ S₂
```

salvo que exista una transformación explícita:

```text
Transform(G₂, S₁)
```

---

# 325. Predicate model

Un predicate:

```text
P(G₁,G₂)
```

sólo podrá evaluarse cuando:

```text
Compatible(
    type(G₁),
    type(G₂),
    CRS(G₁),
    CRS(G₂),
    dimensions,
    operation
)
```

sea válido.

---

# 326. Distance model

```text
Distance(G₁,G₂,M,U)
```

donde:

```text
M = distance model
U = requested unit
```

El resultado sólo será válido cuando exista:

```text
CRS compatibility
+
distance capability
+
unit conversion capability
```

---

# 327. Spatial query planning

Formalmente:

```text
SpatialPlan =
f(
    QueryAST,
    SpatialAST,
    TypeMetadata,
    CRSMetadata,
    PlatformCapabilities,
    ExtensionCapabilities,
    SpatialIndexes,
    TenantContext,
    DistributionContext,
    ConsistencyPolicy,
    ResourcePolicy
)
```

---

# 328. Nearest-neighbor model

Sea:

```text
Q
```

el conjunto de candidatos y:

```text
T
```

el target.

La operación:

```text
Nearest(Q,T,N)
```

deberá producir los `N` elementos con menor distancia efectiva bajo:

```text
DistanceModel M
CRS S
Unit U
```

y un ordering total:

```text
(distance, tie-breaker)
```

---

# 329. Distributed nearest-neighbor

Para shards:

```text
S₁...Sₙ
```

no basta afirmar:

```text
NearestGlobal =
Nearest(Nearest(S₁,N) ∪ ... ∪ Nearest(Sₙ,N), N)
```

sin demostrar que el modelo local y global, los límites, la disponibilidad y el ordering hacen válida esa composición.

El Distributed Planner será responsable de esa demostración.

---

# 330. Final architecture

```text
                           Application
                                │
                                ▼
                     Model / Repository API
                                │
                                ▼
                          Query Builder
                                │
                                ▼
                           Spatial API
                                │
                                ▼
                           Spatial AST
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
             ▼                  ▼                  ▼
        Spatial Types       CRS / SRID        Spatial Values
             │                  │                  │
             └──────────────────┼──────────────────┘
                                ▼
                       Semantic Analyzer
                                │
                                ▼
                       Capability Resolver
                                │
                   ┌────────────┼─────────────┐
                   │            │             │
                   ▼            ▼             ▼
             DB Platform     Extension     Spatial Index
             Capabilities   Capabilities     Metadata
                   │            │             │
                   └────────────┼─────────────┘
                                ▼
                         Query Planner
                                │
                ┌───────────────┼────────────────┐
                │               │                │
                ▼               ▼                ▼
           Tenant Scope     Shard Routing   Resource Policy
                │               │                │
                └───────────────┼────────────────┘
                                ▼
                           Optimizer
                                │
                                ▼
                       Spatial Compiler
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
        MySQL                MariaDB              PostgreSQL
                                                     │
                                                     ▼
                                                   PostGIS
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
                                ▼
                              SQLite
                                │
                                ▼
                     Optional Spatial Extension
                                │
                                ▼
                        Execution Engine
                                │
                                ▼
                         Result System
                                │
                                ▼
                     Spatial Type Conversion
                                │
                                ▼
                       Hydration / Projection
```

---

# 331. Query lifecycle

```text
Spatial Query Intent
        │
        ▼
Query Builder
        │
        ▼
Spatial AST
        │
        ▼
Type Resolution
        │
        ▼
CRS/SRID Resolution
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
Shard Resolution
        │
        ▼
Capability Resolution
        │
        ▼
Spatial Index Analysis
        │
        ▼
Resource Analysis
        │
        ▼
Query Planning
        │
        ▼
Optimization
        │
        ▼
Platform Compilation
        │
        ▼
Prepared Execution
        │
        ▼
Spatial Value Conversion
        │
        ▼
Hydration / Projection
```

---

# 332. Example end-to-end

```php
$target = GeographicPoint::fromLongitudeLatitude(
    longitude: -100.316,
    latitude: 25.686,
);

$stores = Store::query()
    ->whereSpatialWithin(
        column: 'location',
        geometry: $serviceArea,
    )
    ->orderBySpatialDistance(
        column: 'location',
        from: $target,
        model: SpatialDistanceModel::GEODESIC,
    )
    ->orderBy('id')
    ->limit(20)
    ->get();
```

Internamente:

```text
SelectQuery
│
├── FROM Store
│
├── TenantPredicate(...)
│
├── SpatialWithin
│   ├── location
│   └── serviceArea
│
├── ORDER
│   ├── SpatialDistance
│   │   ├── location
│   │   ├── target
│   │   └── GEODESIC
│   │
│   └── id
│
└── LIMIT 20
```

El planner resuelve:

```text
location type      → Point
location SRID      → 4326
serviceArea type   → Polygon
serviceArea SRID   → 4326
target SRID        → 4326
distance model     → geodesic
spatial index      → available
platform           → resolved
extension          → resolved
tenant             → scoped
shard              → resolved
```

y únicamente después:

```text
Spatial Compiler
        ↓
platform-specific representation
```

---

# 333. Decisión arquitectónica

VoltStack implementará las capacidades geoespaciales como una **extensión tipada del Query Engine y Type System**, con una capa explícita para:

```text
Spatial Values
Spatial AST
CRS/SRID
Capabilities
Spatial Indexes
Platform Compilation
Optional Spatial Extensions
```

No se diseñará como:

```text
collection of ST_* helper methods
```

acoplados directamente al SQL de un proveedor.

La aplicación podrá utilizar una API sencilla:

```php
Store::query()
    ->near(
        column: 'location',
        point: $location,
        radius: 10,
        unit: SpatialUnit::KILOMETER,
    )
    ->get();
```

mientras internamente VoltStack preservará:

```text
Geometry Type
CRS
SRID
Units
Distance Model
Spatial Predicate
Spatial Index Strategy
Platform Capability
Extension Capability
Tenant Scope
Shard Scope
Security Policy
Resource Policy
```

---

# 334. Regla principal

> **Una geometría en VoltStack no será tratada como un conjunto arbitrario de números. Su tipo, dimensionalidad y sistema de referencia forman parte de su significado semántico y deberán preservarse desde la definición del esquema hasta la consulta, compilación, ejecución, hidratación, caché y telemetría.**

---

# 335. Regla de transformación

> **Asignar un SRID y transformar una geometría son operaciones distintas. VoltStack nunca deberá convertir una geometría entre sistemas de referencia limitándose a cambiar su identificador SRID.**

---

# 336. Regla de portabilidad

> **La portabilidad espacial no consistirá en ocultar diferencias entre MySQL, MariaDB, PostgreSQL/PostGIS y SQLite. Consistirá en expresar una intención espacial común, descubrir las capacidades efectivas de cada plataforma y rechazar explícitamente aquellas operaciones cuya semántica no pueda conservarse.**

---

# 337. Regla de optimización

> **Un bounding box, índice espacial o algoritmo aproximado podrá reducir el conjunto de candidatos, pero nunca sustituirá silenciosamente una operación espacial exacta cuando el contrato de la consulta exija exactitud.**

---

# 338. Bloque 27 — progreso

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES

✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
✓ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
✓ 269_DATABASE_SOFT_DELETE_SYSTEM.md
✓ 270_DATABASE_DATA_RETENTION_SYSTEM.md
✓ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
✓ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
✓ 273_DATABASE_JSON_QUERY_SYSTEM.md
✓ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
□ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 339. Siguiente documento

```text
275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

El siguiente documento cerrará el **Bloque 27 — Advanced Database Capabilities** definiendo el sistema unificado mediante el cual VoltStack podrá responder preguntas como:

```text
¿Esta conexión soporta RETURNING?
¿Soporta CTE recursivos?
¿Soporta window functions?
¿Soporta JSON?
¿Soporta una operación JSON concreta?
¿Soporta full-text?
¿Soporta geometry?
¿Tiene PostGIS?
¿Soporta spatial index?
¿Soporta SKIP LOCKED?
¿Soporta savepoints?
¿Soporta generated columns?
¿Soporta expression indexes?
¿Soporta transactional DDL?
¿Soporta native UPSERT?
```

sin recurrir a:

```php
if ($database === 'mysql') {
    // ...
}
```

o:

```php
if ($version >= ...) {
    // assume capability
}
```

La arquitectura objetivo será:

```text
Database Platform
       │
       ▼
Capability Discovery
       │
       ├── Static Platform Capabilities
       ├── Version-derived Evidence
       ├── Runtime-discovered Capabilities
       ├── Installed Extensions
       ├── Connection-specific Features
       └── Configuration Restrictions
       │
       ▼
Capability Registry
       │
       ▼
Capability Resolution
       │
       ▼
Capability Evidence
       │
       ▼
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EXTENSION
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

con una regla especialmente importante para toda la arquitectura Database:

> **`DatabaseVendor + Version` no constituye por sí solo una capability. VoltStack tomará decisiones de planificación utilizando capacidades explícitas respaldadas por evidencia, y `UNKNOWN` nunca será tratado silenciosamente como `SUPPORTED`.**