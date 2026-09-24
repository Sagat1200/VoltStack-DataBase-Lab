# 138_DATABASE_SCALAR_HYDRATION_SYSTEM

## Propósito

El **Scalar Hydration System** es el componente responsable de convertir valores primitivos provenientes del motor de base de datos en valores PHP tipados y semánticamente correctos. A diferencia del `ResultHydrator`, aquí no existen entidades ni relaciones: únicamente valores escalares con preservación absoluta de precisión, nulabilidad y tipo.

**Principio fundamental**

> Un valor escalar nunca debe perder información durante la hidratación. La comodidad del desarrollador jamás tendrá prioridad sobre la integridad del dato.

---

## Arquitectura

```text
Database Driver
      │
      ▼
Raw Database Value
      │
      ▼
CompiledScalarHydrationPlan
      │
      ▼
ScalarHydrator
      │
      ├── NULL semantics
      ├── Type conversion
      ├── Precision validation
      ├── Overflow protection
      ├── Custom converters
      │
      ▼
Typed PHP Value
```

---

## Responsabilidades

El `ScalarHydrator` deberá:

- Convertir tipos SQL → PHP.
- Preservar precisión numérica.
- Mantener la semántica de `NULL`.
- Detectar conversiones inválidas.
- Soportar tipos personalizados.
- Ser completamente determinista.

No deberá:

- Crear entidades.
- Acceder al `IdentityMap`.
- Ejecutar consultas adicionales.
- Inferir tipos dinámicamente sin un plan compilado.

---

## Tipos soportados

| SQL | PHP recomendado |
|------|-----------------|
| BOOLEAN | bool |
| SMALLINT | int |
| INTEGER | int |
| BIGINT | string / BigInteger |
| DECIMAL | Decimal |
| FLOAT | float |
| VARCHAR | string |
| TEXT | string |
| UUID | Uuid |
| JSON | array / object |
| DATE | LocalDate |
| DATETIME | DateTimeImmutable |

---

## Plan compilado

```php
final readonly class ScalarHydrationPlan
{
    public function __construct(
        public ScalarType $type,
        public bool $nullable,
        public ?ValueConverter $converter,
        public PrecisionPolicy $precision,
    ) {}
}
```

---

## Conversión tipada

```text
"42"        → int(42)
"true"      → bool(true)
"2026-09-06"→ LocalDate
"550.35"    → Decimal
```

Todas las conversiones deberán ser explícitas mediante el `TypeRegistry`.

---

## NULL vs Missing

| Estado | Significado |
|---------|-------------|
| NULL | El valor existe y es nulo |
| Missing | La columna no fue proyectada |

Esta diferencia es crítica para projections y DTOs.

---

## Precisión numérica

### BIGINT

Nunca deberá convertirse automáticamente a `float`.

```text
9223372036854775807
```

Deberá mantenerse como `string` o `BigInteger`.

### DECIMAL

Los valores monetarios deberán conservar precisión exacta.

```text
550.35
```

No deberá convertirse automáticamente a `float`.

---

## Boolean normalizado

Se aceptan múltiples representaciones del driver:

```text
1
0
"1"
"0"
true
false
```

Todas convergen en un único `bool`.

---

## Fechas

El plan deberá conocer:

- Zona horaria.
- Precisión.
- Mutabilidad.
- Calendario.

Ejemplo:

```text
2026-09-06 13:45:10.123456 UTC
```

→ `DateTimeImmutable`

---

## UUID

Los UUID deberán hidratarse como objetos de valor.

```text
550e8400-e29b-41d4-a716-446655440000
```

→ `Uuid`

---

## JSON

Dependiendo del plan:

```text
JSON
```

→

- array
- objeto tipado
- documento inmutable

---

## Enums

Los enums deberán validarse contra la definición registrada.

```text
ACTIVE
```

→ `UserStatus::ACTIVE`

Valores desconocidos producen excepción.

---

## Tipos personalizados

Los desarrolladores podrán registrar convertidores:

```php
MoneyConverter
ColorConverter
GeoPointConverter
```

Todos deberán implementarse mediante `ValueConverter`.

---

## Agregados

Los resultados de funciones SQL deberán respetar su tipo lógico.

| SQL | Tipo |
|------|------|
| COUNT | int |
| SUM(DECIMAL) | Decimal |
| AVG | Decimal |
| MAX(date) | LocalDate |

---

## Colecciones escalares

Ejemplo:

```php
User::query()
    ->select('email')
    ->scalars();
```

Resultado:

```php
[
    "a@volt.dev",
    "b@volt.dev"
]
```

---

## Tuplas escalares

```php
SELECT id, total
```

Resultado:

```php
[
    [1, Decimal("550.35")],
    [2, Decimal("1200.00")]
]
```

---

## Protección contra overflow

Si un valor excede el rango permitido por el tipo destino, la hidratación deberá fallar.

Nunca deberá truncarse silenciosamente.

---

## Errores

Jerarquía propuesta:

```text
ScalarHydrationException
├── ScalarNullViolationException
├── ScalarOverflowException
├── ScalarPrecisionException
├── ScalarConversionException
├── ScalarEnumException
├── ScalarJsonException
└── ScalarTypeMismatchException
```

---

## Telemetría

Métricas principales:

```text
db.scalar_hydration.operations
db.scalar_hydration.duration
db.scalar_hydration.failures
db.scalar_hydration.decimal_conversions
db.scalar_hydration.bigint_conversions
db.scalar_hydration.json_conversions
```

---

## Runtime persistente

En FrankenPHP/RoadRunner únicamente podrán compartirse:

- Convertidores compilados.
- Metadata de tipos.
- Planes inmutables.

Toda conversión deberá ejecutarse dentro del `HydrationScope` de la petición.

---

## Invariantes

1. Nunca perder precisión numérica.
2. `NULL` y `Missing` son estados distintos.
3. BIGINT jamás se convierte automáticamente a `float`.
4. DECIMAL conserva precisión exacta.
5. Todo valor pasa por un `ScalarHydrationPlan`.
6. Las conversiones inválidas generan excepciones tipadas.
7. El sistema es completamente determinista e independiente del driver.

---

## Fórmula maestra

```text
TypedScalar
=
Convert(
    RawDatabaseValue,
    ScalarHydrationPlan,
    TypeRegistry,
    PrecisionPolicy
)
```

---

## Siguiente documento

**139_DATABASE_TUPLE_HYDRATION_SYSTEM.md**

Este documento formalizará la hidratación de resultados compuestos (Entity + Scalar, Scalar + Scalar, DTO Tuples y ResultTuple).
