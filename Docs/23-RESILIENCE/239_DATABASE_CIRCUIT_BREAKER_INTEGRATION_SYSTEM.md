# 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Circuit Breaker Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 239 — Database Circuit Breaker Integration System  
**Bloque:** 23 — Resilience  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `238_DATABASE_RETRY_POLICY_SYSTEM.md`  
**Siguiente documento:** `240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack Database integra el patrón **Circuit Breaker** con conexiones, pools, endpoints, réplicas, writers, shards, routing, retry, failover y telemetría.

La finalidad principal es evitar que una degradación persistente de infraestructura provoque una cascada de:

```text
timeouts
retries
reconnections
queue buildup
pool exhaustion
worker saturation
memory pressure
latency amplification
retry storms
```

La regla central será:

> **El Circuit Breaker no intenta reparar una operación fallida; protege al sistema evitando que nuevas operaciones continúen ejerciendo presión sobre un recurso que presenta evidencia suficiente de degradación.**

Por tanto:

```text
Circuit Breaker
≠
Retry

Circuit Breaker
≠
Failover

Circuit Breaker
≠
Health Check

Circuit Breaker
≠
Connection Pool

Circuit Breaker
≠
Load Balancer
```

---

# 2. Problema

Supongamos:

```text
Application
   │
   ├── Worker 1
   ├── Worker 2
   ├── Worker 3
   ├── ...
   └── Worker 500
          │
          ▼
     PostgreSQL Replica A
```

Replica A comienza a responder lentamente.

Sin protección:

```text
500 workers
   ↓
timeouts
   ↓
500 retries
   ↓
new connections
   ↓
more load
   ↓
more timeouts
   ↓
more retries
```

Se genera:

```text
Failure Amplification
```

El problema inicial:

```text
one degraded database endpoint
```

puede convertirse en:

```text
application-wide resource exhaustion
```

---

# 3. Objetivos

El sistema deberá determinar:

```text
¿qué recurso está fallando?

¿qué failures cuentan para el circuito?

¿cuántos failures son suficientes?

¿en qué ventana temporal?

¿qué porcentaje de operaciones falla?

¿qué latencia se considera degradada?

¿cuándo abrir el circuito?

¿durante cuánto tiempo?

¿quién puede probar recuperación?

¿cuántos probes se permiten?

¿cuándo cerrar nuevamente?

¿cuándo mantener OPEN?

¿el estado pertenece al endpoint?

¿al pool?

¿al shard?

¿al writer?

¿a una replica?

¿puede Retry utilizar ese recurso?

¿puede Load Balancer seleccionarlo?

¿debe Failover buscar otro target?

¿cómo evitar un thundering herd al recuperarse?
```

---

# 4. Principio arquitectónico

El Circuit Breaker será una capa de **admission control basada en evidencia de fallos recientes**.

```text
Database Operation
       │
       ▼
Routing
       │
       ▼
Target Candidate
       │
       ▼
Circuit Breaker
       │
 ┌─────┴─────┐
 │           │
ALLOW      REJECT
 │           │
 ▼           ▼
Execute   Alternate Target /
         Fail Fast / Failover
```

---

# 5. Circuit Breaker ≠ Retry

Retry responde:

```text
¿debo repetir esta operación?
```

Circuit Breaker responde:

```text
¿debo permitir que una operación alcance este recurso?
```

---

# 6. Circuit Breaker ≠ Failover

Failover decide:

```text
¿quién debe asumir el servicio?
```

Circuit Breaker aporta evidencia:

```text
este target no debería recibir tráfico actualmente
```

---

# 7. Circuit Breaker ≠ Health Check

Health Check observa activamente:

```text
¿el recurso responde?
```

Circuit Breaker observa principalmente tráfico real:

```text
¿las operaciones recientes muestran degradación?
```

Ambos sistemas podrán compartir señales, pero no serán equivalentes.

---

# 8. Circuit Breaker ≠ Load Balancer

Load Balancer selecciona entre:

```text
eligible endpoints
```

Circuit Breaker ayuda a determinar:

```text
endpoint eligibility
```

---

# 9. Arquitectura general

```text
                 ┌────────────────────┐
                 │ Database Operation │
                 └──────────┬─────────┘
                            │
                            ▼
                 ┌────────────────────┐
                 │ Routing / Topology │
                 └──────────┬─────────┘
                            │
                            ▼
                  Candidate Endpoint
                            │
                            ▼
                 ┌────────────────────┐
                 │ Circuit Registry   │
                 └──────────┬─────────┘
                            │
                            ▼
                 ┌────────────────────┐
                 │ Circuit Breaker    │
                 └──────────┬─────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
           CLOSED        HALF_OPEN       OPEN
              │             │             │
              ▼             ▼             ▼
            ALLOW        PROBE ONLY      REJECT
              │             │
              ▼             ▼
                    Database Endpoint
                            │
                            ▼
                    Operation Outcome
                            │
                            ▼
                 Failure Classification
                            │
                            ▼
                  Circuit Observation
                            │
                            ▼
                   State Evaluation
```

---

# 10. Estados fundamentales

VoltStack utilizará:

```php
enum CircuitState
{
    case CLOSED;
    case OPEN;
    case HALF_OPEN;
}
```

---

# 11. CLOSED

Estado normal.

```text
CLOSED
   ↓
operations allowed
```

Las operaciones relevantes producen observaciones.

---

# 12. OPEN

El recurso ha superado el umbral de degradación.

```text
OPEN
   ↓
normal operations rejected
```

El objetivo es:

```text
stop pressure
allow recovery
fail fast
```

---

# 13. HALF_OPEN

Después del período OPEN:

```text
OPEN
  ↓
cooldown elapsed
  ↓
HALF_OPEN
```

Solo se permiten operaciones controladas:

```text
probe attempts
```

---

# 14. State machine

```text
                    failure threshold
             ┌─────────────────────────────┐
             │                             ▼
        ┌──────────┐                  ┌──────────┐
        │  CLOSED  │                  │   OPEN   │
        └────┬─────┘                  └────┬─────┘
             ▲                             │
             │                             │ cooldown
             │                             ▼
             │                       ┌───────────┐
             │                       │ HALF_OPEN │
             │                       └─────┬─────┘
             │                             │
      recovery threshold          ┌────────┴────────┐
             │                    │                 │
             └──────────────── SUCCESS          FAILURE
                                                   │
                                                   ▼
                                                 OPEN
```

---

# 15. CircuitState ≠ ResourceHealth

Un endpoint puede estar:

```text
CircuitState = CLOSED
```

pero:

```text
Health = DEGRADED
```

porque todavía no alcanzó el threshold.

También puede estar:

```text
CircuitState = OPEN
```

aunque una health probe aislada tenga éxito.

---

# 16. Circuit identity

Cada circuito tendrá una identidad estable.

```php
final readonly class CircuitId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 17. Circuit scope

El circuito no será global por default.

```php
enum CircuitScope
{
    case ENDPOINT;
    case CONNECTION_POOL;
    case REPLICA;
    case WRITER;
    case SHARD_ENDPOINT;
    case SHARD;
    case LOGICAL_DATABASE;
    case CUSTOM;
}
```

---

# 18. Endpoint circuit

Será el scope principal.

Ejemplo:

```text
postgres-replica-01.internal:5432
```

tendrá su propio circuito.

---

# 19. Replica circuit

Conceptualmente puede existir:

```text
Replica A → CLOSED
Replica B → OPEN
Replica C → CLOSED
```

Load Balancer deberá considerar únicamente:

```text
A
C
```

si las demás políticas también las consideran elegibles.

---

# 20. Writer circuit

El writer requiere tratamiento especial.

Si:

```text
Writer = OPEN
```

no significa que VoltStack pueda enviar writes a una replica.

---

# 21. Shard endpoint circuit

Ejemplo:

```text
Shard 17
├── Writer → CLOSED
├── Replica A → OPEN
└── Replica B → CLOSED
```

---

# 22. Shard circuit

Un shard completo podría marcarse degradado cuando:

```text
all valid endpoints unavailable
```

pero este circuito agregado será distinto de los circuitos individuales.

---

# 23. Logical database circuit

Será opcional y conservador.

No deberá abrirse porque una sola replica haya fallado.

---

# 24. Circuit key

Conceptualmente:

```text
CircuitKey =
    Scope
  + LogicalDatabase
  + EndpointIdentity
  + Role
  + ShardId?
  + TopologyGeneration?
```

---

# 25. Endpoint identity

La identidad deberá ser estable y no depender únicamente de:

```text
current connection object
```

---

# 26. Connection ≠ Circuit

Una conexión individual puede romperse.

Eso no demuestra:

```text
endpoint failure
```

---

# 27. Pool ≠ Circuit

Un pool puede agotarse por presión local aunque la base de datos esté saludable.

---

# 28. Failure observation

Cada operación relevante podrá producir:

```php
final readonly class CircuitObservation
{
    public function __construct(
        public CircuitId $circuitId,
        public CircuitObservationType $type,
        public Duration $duration,
        public Instant $observedAt,
        public FailureCategory|null $failure,
        public Confidence $confidence,
    ) {}
}
```

---

# 29. Observation types

```php
enum CircuitObservationType
{
    case SUCCESS;
    case FAILURE;
    case SLOW_SUCCESS;
    case TIMEOUT;
    case CONNECTION_FAILURE;
    case REJECTED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 30. REJECTED observation

Una operación rechazada porque el circuito ya estaba OPEN no deberá contarse como nuevo database failure.

De lo contrario:

```text
OPEN
→ rejected request
→ counted as failure
→ OPEN forever
```

---

# 31. CANCELLED observation

Una cancelación iniciada por el usuario/request normalmente no contará como failure del endpoint.

---

# 32. UNKNOWN observation

`UNKNOWN` no será automáticamente:

```text
SUCCESS
```

ni:

```text
FAILURE
```

Su impacto dependerá de la causa y política.

---

# 33. Circuit-relevant failures

Ejemplos:

```text
connection refused
connection timeout
server unavailable
connection reset
network unreachable
server overload
database shutdown
repeated query timeout caused by endpoint
```

---

# 34. Circuit-irrelevant failures

Ejemplos:

```text
syntax error
unique constraint violation
foreign key violation
invalid parameter
authorization failure
application cancellation
unsupported SQL feature
```

Estos errores no demuestran que el endpoint esté degradado.

---

# 35. Deadlock

Un deadlock normalmente no deberá abrir un endpoint circuit por sí solo.

Es:

```text
concurrency conflict
```

no necesariamente:

```text
endpoint unavailable
```

---

# 36. Serialization failure

Igualmente deberá tratarse principalmente mediante:

```text
Transaction Retry
```

y no como señal directa de infraestructura rota.

---

# 37. Failure classifier integration

```text
DatabaseFailure
      ↓
FailureClassifier
      ↓
CircuitFailureClassifier
      ↓
CircuitImpact
```

---

# 38. CircuitImpact

```php
enum CircuitImpact
{
    case POSITIVE_FAILURE_SIGNAL;
    case POSITIVE_SLOW_SIGNAL;
    case SUCCESS_SIGNAL;
    case IGNORE;
    case UNKNOWN;
}
```

---

# 39. Failure window

VoltStack no utilizará únicamente:

```text
consecutive failures
```

Podrá analizar una ventana.

---

# 40. Sliding window

Dos modelos principales:

```text
COUNT_BASED
TIME_BASED
```

---

# 41. Count-based window

Ejemplo:

```text
last 100 calls
```

---

# 42. Time-based window

Ejemplo:

```text
last 30 seconds
```

---

# 43. WindowType

```php
enum CircuitWindowType
{
    case COUNT_BASED;
    case TIME_BASED;
}
```

---

# 44. Failure rate

Sea:

```text
F = relevant failures
N = relevant observations
```

Entonces:

```text
FailureRate = F / N
```

---

# 45. Minimum throughput

No será correcto abrir circuito por:

```text
1 failure / 1 request
```

si la política requiere evidencia mayor.

Se definirá:

```text
MinimumThroughput
```

---

# 46. Opening condition

Conceptualmente:

```text
OpenCircuit =
    RelevantCalls >= MinimumThroughput
    ∧
    FailureRate >= FailureThreshold
```

---

# 47. Ejemplo

Configuración:

```text
minimum throughput = 20
failure threshold = 50%
```

Observaciones:

```text
20 calls
12 failures
8 success
```

Entonces:

```text
failure rate = 60%
```

y el circuito podrá abrirse.

---

# 48. Slow-call detection

Un endpoint puede responder sin errors pero con latencia extrema.

```text
SUCCESS
≠
HEALTHY PERFORMANCE
```

---

# 49. SlowCallThreshold

Ejemplo:

```text
query duration >= 2 seconds
```

puede considerarse:

```text
SLOW_SUCCESS
```

---

# 50. Slow call rate

```text
SlowRate =
SlowCalls / RelevantCalls
```

---

# 51. Slow circuit opening

Una policy podrá abrir cuando:

```text
SlowRate >= SlowRateThreshold
```

y exista suficiente throughput.

---

# 52. Slow call semantics

No toda query lenta demuestra endpoint degradation.

Una consulta inherentemente costosa podría tardar:

```text
30 seconds
```

en una base saludable.

Por ello deberán considerarse:

```text
query class
expected timeout
operation budget
profiler evidence
```

---

# 53. Integration with Slow Query Detection

El documento:

```text
222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
```

podrá aportar señales.

Pero:

```text
Slow Query
≠
Circuit Failure
```

---

# 54. CircuitPolicy

```php
interface CircuitPolicy
{
    public function evaluate(
        CircuitSnapshot $snapshot
    ): CircuitDecision;
}
```

---

# 55. CircuitSnapshot

```php
final readonly class CircuitSnapshot
{
    public function __construct(
        public CircuitId $id,
        public CircuitState $state,
        public int $sampleSize,
        public float $failureRate,
        public float $slowRate,
        public Duration $openDuration,
        public Instant|null $openedAt,
        public int $halfOpenAttempts,
    ) {}
}
```

---

# 56. CircuitDecision

```php
enum CircuitDecision
{
    case KEEP_CLOSED;
    case OPEN;
    case KEEP_OPEN;
    case ENTER_HALF_OPEN;
    case CLOSE;
    case REOPEN;
}
```

---

# 57. Opening circuit

Transición:

```text
CLOSED
  ↓
OPEN
```

deberá ser atómica respecto al estado del circuito.

---

# 58. Open duration

```php
final readonly class OpenDuration
{
    public function __construct(
        public Duration $duration,
    ) {}
}
```

Ejemplo:

```text
30 seconds
```

---

# 59. Fixed cooldown

Modelo simple:

```text
OPEN
↓
wait 30s
↓
HALF_OPEN
```

---

# 60. Adaptive cooldown

VoltStack podrá soportar:

```text
repeated recovery failure
        ↓
increase open duration
```

Ejemplo:

```text
30s
60s
120s
240s
```

con máximo configurable.

---

# 61. Adaptive cooldown ≠ infinite quarantine

Siempre deberá existir política explícita.

---

# 62. Half-open

HALF_OPEN será un estado altamente controlado.

---

# 63. Probe

Una operación autorizada durante HALF_OPEN será:

```text
Circuit Probe
```

---

# 64. Probe budget

```php
final readonly class HalfOpenProbeBudget
{
    public function __construct(
        public int $maxConcurrent,
        public int $requiredSuccesses,
        public int $maxAttempts,
    ) {}
}
```

---

# 65. Probe concurrency

Si:

```text
1000 requests
```

llegan cuando termina cooldown, no deberán convertirse en:

```text
1000 probes
```

---

# 66. Probe admission

Ejemplo:

```text
maxConcurrent = 3
```

Entonces:

```text
3 probes allowed
997 fail fast / rerouted
```

---

# 67. Half-open success

No necesariamente un solo success cerrará circuito.

Podrá requerirse:

```text
N successful probes
```

---

# 68. Half-open failure

Una failure relevante podrá provocar:

```text
HALF_OPEN
    ↓
OPEN
```

inmediatamente.

---

# 69. Probe ≠ ordinary retry

Una operación de retry no obtiene automáticamente permiso para ser probe.

---

# 70. Probe ownership

El Circuit Breaker decidirá qué attempt recibe:

```text
ProbePermit
```

---

# 71. ProbePermit

```php
final readonly class ProbePermit
{
    public function __construct(
        public CircuitId $circuit,
        public string $token,
        public Instant $expiresAt,
    ) {}
}
```

---

# 72. Permit lifecycle

```text
REQUESTED
   ↓
GRANTED
   ↓
USED
   ↓
RELEASED
```

o:

```text
EXPIRED
```

---

# 73. Permit leakage

En persistent runtimes será crítico liberar permits aunque:

```text
exception
cancellation
timeout
early return
```

ocurra.

---

# 74. Circuit admission

Contrato:

```php
interface CircuitBreaker
{
    public function acquire(
        CircuitKey $key,
        CircuitRequest $request
    ): CircuitPermit;
}
```

---

# 75. CircuitPermit

```php
enum CircuitPermitType
{
    case NORMAL;
    case PROBE;
    case REJECTED;
}
```

---

# 76. Fast rejection

OPEN deberá permitir:

```text
fail fast
```

sin esperar al timeout del servidor.

---

# 77. Rejection exception

```php
final class CircuitOpenException extends DatabaseResilienceException
{
}
```

---

# 78. Rejection ≠ database failure

El error deberá conservar información:

```text
operation not sent
```

Por tanto:

```text
Outcome = NOT_EXECUTED
```

---

# 79. Retry integration

Flujo:

```text
Attempt fails
    ↓
RetryPolicy
    ↓
Retry requested
    ↓
TargetResolver
    ↓
Circuit check
    ↓
┌───────────────┐
│ CLOSED        │ → retry allowed
│ HALF_OPEN     │ → only with probe permit
│ OPEN          │ → target rejected
└───────────────┘
```

---

# 80. Retry target selection

Retry podrá solicitar:

```text
new connection / same endpoint
```

pero si el circuito está OPEN:

```text
same endpoint
```

deberá rechazarse.

---

# 81. Retry alternative target

Podrá evaluarse:

```text
alternate replica
```

si routing y consistencia lo permiten.

---

# 82. Retry budget interaction

Un attempt rechazado por OPEN deberá tener semántica explícita respecto al budget.

Recomendación:

```text
database execution attempt = not consumed
routing/retry decision budget = may be consumed
```

---

# 83. Circuit rejection loop

Debe evitarse:

```text
Retry
→ OPEN target
→ rejected
→ Retry
→ same OPEN target
→ rejected
→ ...
```

---

# 84. Target exclusion

RetryContext podrá mantener:

```text
temporarily excluded targets
```

para el attempt lógico actual.

---

# 85. Failover integration

Circuit Breaker podrá producir:

```text
FailoverCandidateSignal
```

pero no ejecutará failover directamente.

---

# 86. Writer OPEN

```text
Writer Circuit OPEN
```

podrá activar evaluación de:

```text
240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM
```

---

# 87. Replica OPEN

Normalmente bastará con:

```text
remove replica from eligible read set
```

sin promover otra replica.

---

# 88. Failover threshold

La política de failover puede exigir evidencia adicional:

```text
Circuit OPEN
+
Health failure
+
Topology evidence
```

antes de cambiar autoridad.

---

# 89. Circuit OPEN ≠ authority lost

Especialmente importante para writers.

---

# 90. Split-brain protection

VoltStack nunca interpretará:

```text
Writer A circuit OPEN
```

como permiso automático para escribir en:

```text
Writer B
```

sin confirmación de autoridad.

---

# 91. Load Balancer integration

```text
Topology
   ↓
Healthy candidates
   ↓
Circuit filtering
   ↓
Consistency filtering
   ↓
Load balancing
```

---

# 92. Correct ordering

Conceptualmente:

```text
Candidates
→ Role Eligibility
→ Health Eligibility
→ Circuit Eligibility
→ Consistency Eligibility
→ Load Balancer
```

La implementación exacta podrá combinar fases, pero las semánticas deberán preservarse.

---

# 93. OPEN endpoint

No participará en load balancing normal.

---

# 94. HALF_OPEN endpoint

No participará normalmente en balanceo.

Solo operaciones con:

```text
probe permit
```

podrán alcanzarlo.

---

# 95. Connection Pool integration

Antes de crear nuevas conexiones hacia un endpoint OPEN:

```text
Circuit Breaker
```

podrá rechazar adquisición.

---

# 96. Existing idle connections

Un circuito OPEN no implica necesariamente destruir inmediatamente todas las conexiones existentes.

---

# 97. Pool quarantine

Podrá marcarse:

```text
endpoint pool = QUARANTINED
```

para evitar nuevas leases.

---

# 98. Connection destruction

Dependerá de:

```text
failure type
connection health
pool policy
```

---

# 99. Circuit OPEN ≠ close all sockets

Cerrar cientos de conexiones simultáneamente puede causar otro pico al recuperarse.

---

# 100. Pool exhaustion

Si el pool está agotado por carga de aplicación:

```text
Pool Exhaustion
```

no deberá abrir automáticamente el endpoint circuit.

---

# 101. Endpoint overload

Si existe evidencia del servidor:

```text
too many connections
server overloaded
```

sí podrá aportar señal.

---

# 102. Health integration

El Circuit Breaker podrá consumir:

```text
HealthEvidence
```

como señal secundaria.

---

# 103. Passive vs active signals

```text
Passive:
real application traffic

Active:
health probes
```

---

# 104. Passive evidence priority

Las operaciones reales suelen aportar evidencia más representativa.

---

# 105. Health probe success

Un:

```text
SELECT 1
```

exitoso no demuestra que:

```text
production workload
```

haya recuperado capacidad.

---

# 106. Health probe failure

Puede aportar evidencia fuerte de indisponibilidad, dependiendo del probe.

---

# 107. Read/write distinction

Podrán existir circuitos separados cuando un endpoint pueda:

```text
accept reads
```

pero no:

```text
accept writes
```

---

# 108. Capability-specific circuits

Opcionalmente:

```text
Endpoint
├── CONNECTION circuit
├── READ circuit
└── WRITE circuit
```

Esto deberá utilizarse solo cuando exista valor real; demasiada granularidad aumenta cardinalidad y complejidad.

---

# 109. Transaction integration

Una transaction activa no consultará Circuit Breaker para migrar cada statement.

Ya está:

```text
connection pinned
```

---

# 110. Transaction start

Antes de adquirir el endpoint para iniciar transaction sí podrá consultarse.

---

# 111. Circuit opens mid-transaction

No deberá abortarse automáticamente una transaction saludable solo porque otros requests abrieron el circuito.

---

# 112. Actual transaction failure

Será procesado por:

```text
Transaction Failure
Retry Policy
Connection Failure Handling
```

---

# 113. Circuit state is advisory to existing transaction

Una transacción ya establecida conserva su propia evidencia de connection health.

---

# 114. Sharding integration

Cada shard podrá tener circuitos independientes.

```text
Shard A → healthy
Shard B → degraded
Shard C → healthy
```

---

# 115. Shard failure isolation

El fallo de Shard B no deberá abrir:

```text
global database circuit
```

por default.

---

# 116. Shard writer

```text
Shard 42 Writer OPEN
```

afecta writes hacia ese shard.

No necesariamente otros shards.

---

# 117. Distributed query

Una consulta distribuida deberá conocer:

```text
which shards were unavailable because circuit open
```

---

# 118. Partial distributed results

VoltStack no deberá presentar:

```text
partial shard response
```

como resultado completo.

---

# 119. Tenant isolation

Circuit state normalmente será infraestructura-scoped, no tenant-scoped.

---

# 120. Tenant ≠ Circuit

Si múltiples tenants utilizan el mismo endpoint:

```text
Tenant A
Tenant B
Tenant C
      ↓
same endpoint
```

comparten evidencia de infraestructura.

---

# 121. Tenant-specific application failure

Un query inválido del Tenant A no deberá afectar circuito compartido.

---

# 122. No tenant cardinality explosion

Evitar:

```text
one circuit per tenant
```

salvo arquitectura explícita de infraestructura dedicada.

---

# 123. Local circuit state

Cada worker podría mantener su propio estado.

Ventajas:

```text
fast
simple
no network dependency
```

Desventajas:

```text
workers disagree
duplicate probes
slower global reaction
```

---

# 124. Shared circuit state

Puede utilizar:

```text
shared coordination provider
```

Ventajas:

```text
coordinated view
global probe control
```

Desventajas:

```text
latency
complexity
new dependency
shared-state failure
```

---

# 125. Hybrid model

VoltStack deberá permitir:

```text
local fast path
+
optional shared coordination
```

---

# 126. Shared provider optional

El core no dependerá obligatoriamente de:

```text
Redis
Memcached
database table
external service
```

---

# 127. CircuitStateStore

```php
interface CircuitStateStore
{
    public function get(CircuitKey $key): CircuitSnapshot;

    public function transition(
        CircuitKey $key,
        CircuitTransition $transition
    ): CircuitSnapshot;
}
```

---

# 128. LocalCircuitStateStore

Default viable para instalaciones simples.

---

# 129. SharedCircuitStateStore

Será extensión/provider.

---

# 130. Atomic transitions

Para shared state:

```text
CLOSED → OPEN
OPEN → HALF_OPEN
HALF_OPEN → CLOSED
HALF_OPEN → OPEN
```

deberán coordinarse atómicamente.

---

# 131. Distributed probe coordination

Especialmente:

```text
HALF_OPEN
```

requiere impedir:

```text
100 workers × 3 probes
```

cuando el límite global era 3.

---

# 132. Coordination unavailable

Si falla el provider compartido, deberá existir política:

```text
FAIL_OPEN
FAIL_CLOSED
LOCAL_FALLBACK
```

---

# 133. Fail-open terminology

Aquí:

```text
FAIL_OPEN
```

significa:

```text
permit database traffic despite circuit coordination failure
```

No debe confundirse con:

```text
CircuitState::OPEN
```

---

# 134. Conservative default

La elección dependerá de criticidad.

No existirá un comportamiento universal correcto.

---

# 135. Clock

El circuito utilizará:

```php
interface Clock
{
    public function now(): Instant;
}
```

No deberá depender directamente de:

```php
time();
```

---

# 136. Monotonic time

Duraciones y ventanas deberán utilizar una fuente adecuada para elapsed time cuando sea posible.

---

# 137. Wall clock jumps

Cambios del reloj del sistema no deberán abrir/cerrar circuitos incorrectamente.

---

# 138. Observation window storage

La implementación deberá ser acotada.

No se almacenará:

```text
infinite operation history
```

---

# 139. Ring buffer

Para count windows podrá utilizarse conceptualmente:

```text
bounded ring buffer
```

---

# 140. Bucketed time window

Para time windows:

```text
bucket 1
bucket 2
...
bucket N
```

con agregados:

```text
success
failure
slow
duration
```

---

# 141. Memory complexity

Debe ser:

```text
O(number of circuits × bounded window state)
```

no:

```text
O(total database calls)
```

---

# 142. High cardinality protection

CircuitKey no incluirá:

```text
query SQL
user ID
request ID
trace ID
raw tenant ID
```

por default.

---

# 143. Topology generation

Un endpoint reemplazado podrá requerir nueva generación.

Ejemplo:

```text
db-node-1 old instance
db-node-1 new instance
```

No siempre deberá heredar el circuito OPEN del nodo anterior.

---

# 144. CircuitGeneration

```php
final readonly class CircuitGeneration
{
    public function __construct(
        public int $value,
    ) {}
}
```

---

# 145. Topology change

Circuit Registry deberá reconciliar:

```text
added endpoints
removed endpoints
replaced endpoints
role changes
```

---

# 146. Writer promotion

Si una replica se convierte en writer:

```text
old replica circuit semantics
```

no deberán trasladarse ciegamente al nuevo role.

---

# 147. Circuit reset

Podrá existir operación administrativa:

```text
reset circuit
```

pero deberá ser explícita y auditable.

---

# 148. Manual force-open

Operadores podrán marcar:

```text
FORCED_OPEN
```

como override administrativo.

---

# 149. Manual force-close

Podrá existir:

```text
FORCED_CLOSED
```

pero es peligroso.

---

# 150. Effective state

Se distinguirá:

```text
ObservedState
AdministrativeOverride
EffectiveState
```

---

# 151. Extended state model

Internamente podrá utilizarse:

```php
enum CircuitOperationalState
{
    case CLOSED;
    case OPEN;
    case HALF_OPEN;
    case FORCED_OPEN;
    case FORCED_CLOSED;
    case DISABLED;
}
```

---

# 152. DISABLED

Significa:

```text
circuit breaker not enforcing admission
```

No significa:

```text
endpoint healthy
```

---

# 153. Forced closed risk

El sistema deberá mostrar diagnostics claros:

```text
circuit protection overridden
```

---

# 154. Configuration

Ejemplo conceptual:

```php
'database' => [

    'resilience' => [

        'circuit_breaker' => [

            'enabled' => true,

            'window' => [
                'type' => 'count',
                'size' => 50,
            ],

            'minimum_throughput' => 20,

            'failure_rate_threshold' => 0.50,

            'slow_call' => [
                'enabled' => true,
                'threshold' => '2s',
                'rate_threshold' => 0.70,
            ],

            'open_duration' => '30s',

            'half_open' => [
                'max_concurrent_probes' => 3,
                'required_successes' => 3,
            ],

        ],

    ],

];
```

---

# 155. Configuration profiles

Podrán existir:

```text
CONSERVATIVE
STANDARD
LATENCY_SENSITIVE
HIGH_AVAILABILITY
CUSTOM
```

---

# 156. Safe default

El perfil default deberá evitar circuitos excesivamente sensibles.

---

# 157. Minimum throughput mandatory

Cuando se use failure-rate opening deberá existir un mínimo razonable.

---

# 158. Consecutive failure trigger

Podrá existir adicionalmente:

```text
N consecutive infrastructure failures
```

como trigger rápido.

---

# 159. Combined policy

Ejemplo:

```text
OPEN if:

failure rate >= 50%
after minimum 20 calls

OR

5 consecutive connection failures
```

---

# 160. Fast trip

Algunos errores de infraestructura podrán justificar apertura rápida.

Ejemplo:

```text
endpoint explicitly removed
database shutdown
```

pero esto deberá basarse en evidencia fuerte.

---

# 161. Security errors

Nunca deberán causar circuit opening.

Ejemplo:

```text
invalid credentials
```

requiere cuidado.

Si todas las conexiones reciben:

```text
authentication failed
```

puede ser un problema de configuración global, pero no necesariamente una falla recuperable del endpoint.

Deberá clasificarse separadamente.

---

# 162. Credential rotation

Durante rotación, auth failures podrían ser transitorios.

Aun así:

```text
Circuit Breaker
```

no deberá ocultar el error de configuración.

---

# 163. Resource exhaustion integration

El siguiente documento:

```text
241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

definirá protección adicional.

Circuit Breaker podrá integrarse con:

```text
connection limits
queue limits
memory budgets
concurrency limits
```

---

# 164. Circuit Breaker as resource protection

Su función indirecta será reducir:

```text
waiting operations
```

cuando un recurso remoto está degradado.

---

# 165. Backpressure

Circuit rejection podrá generar:

```text
backpressure
```

hacia capas superiores.

---

# 166. Queue integration

Un job podrá decidir:

```text
reschedule
fail
reroute
```

ante:

```text
CircuitOpenException
```

pero Database core no controlará la cola.

---

# 167. HTTP independence

Circuit Breaker no conocerá:

```text
HTTP 503
HTTP responses
controllers
```

La capa HTTP podrá mapear errores posteriormente.

---

# 168. Telemetry

Eventos conceptuales:

```text
DatabaseCircuitOpened
DatabaseCircuitClosed
DatabaseCircuitHalfOpened
DatabaseCircuitReopened
DatabaseCircuitProbeGranted
DatabaseCircuitProbeRejected
DatabaseCircuitRequestRejected
DatabaseCircuitThresholdApproaching
DatabaseCircuitOverrideChanged
```

---

# 169. Avoid event storms

No deberá emitirse un evento de alto costo por cada:

```text
CLOSED successful query
```

por default.

---

# 170. Metrics

Ejemplos:

```text
db.circuit.state
db.circuit.opens
db.circuit.closes
db.circuit.reopens
db.circuit.rejections
db.circuit.probes
db.circuit.probe_success
db.circuit.probe_failure
db.circuit.failure_rate
db.circuit.slow_rate
```

---

# 171. Metric labels

Permitidos conceptualmente:

```text
logical_database
endpoint_role
shard_bucket
state
failure_class
```

pero deberá controlarse cardinalidad.

---

# 172. Forbidden metric labels

Evitar:

```text
raw SQL
query parameters
user ID
request ID
trace ID
```

---

# 173. Debug information

Ejemplo:

```text
Circuit:
    db-main/replica-02

State:
    OPEN

Opened:
    2026-09-17T01:32:12

Reason:
    Failure rate threshold exceeded

Window:
    50 calls

Relevant calls:
    50

Failures:
    31

Failure rate:
    62%

Threshold:
    50%

Open duration:
    30s

Retry target eligibility:
    REJECTED

Next transition:
    HALF_OPEN candidate
```

---

# 174. Half-open diagnostics

```text
Circuit:
    shard-17/writer

State:
    HALF_OPEN

Probe limit:
    3

Active probes:
    2

Successful probes:
    1 / 3

Normal traffic:
    REJECTED
```

---

# 175. Explain API

Conceptualmente:

```php
$analysis = $db
    ->resilience()
    ->circuits()
    ->explain($endpoint);
```

---

# 176. Query profiler integration

Profiler podrá mostrar:

```text
query not executed
reason: circuit open
```

sin registrar una duración ficticia de database execution.

---

# 177. Audit integration

Administrative overrides podrán registrarse mediante:

```text
Database Audit System
```

---

# 178. Sensitive information

Diagnostics no expondrán:

```text
credentials
connection passwords
secret tokens
raw sensitive bindings
```

---

# 179. Persistent runtime

En:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

Circuit Breaker deberá distinguir entre:

```text
shared immutable configuration
long-lived circuit state
request-scoped permits
```

---

# 180. Circuit state may be worker-long-lived

A diferencia de:

```text
EntityManager
TransactionContext
RetryAttempt
```

el Circuit Breaker puede legítimamente mantener estado entre requests.

---

# 181. Important exception to request reset

```text
Circuit history
```

no deberá borrarse al finalizar cada request.

De hacerlo:

```text
circuit never learns cross-request failures
```

---

# 182. Request state

En cambio:

```text
ProbePermit
CircuitAdmission
OperationObservationContext
```

sí serán request/operation scoped.

---

# 183. No leaked permits

Un worker persistente no deberá conservar un probe permit después de terminar la operación.

---

# 184. Concurrency safety

Circuit state deberá soportar múltiples requests concurrentes.

Especialmente en:

```text
OpenSwoole
```

---

# 185. Atomic counters

Los counters compartidos deberán actualizarse de forma concurrency-safe.

---

# 186. Immutable snapshots

Lecturas de estado podrán exponerse mediante:

```text
immutable CircuitSnapshot
```

---

# 187. Circuit registry

```php
interface CircuitRegistry
{
    public function for(CircuitKey $key): CircuitBreaker;

    public function snapshot(CircuitKey $key): CircuitSnapshot;
}
```

---

# 188. Registry lifecycle

Podrá ser:

```text
worker-long-lived
```

si mantiene estado local.

---

# 189. Registry cleanup

Endpoints eliminados deberán eventualmente ser removidos para evitar:

```text
unbounded circuit registry growth
```

---

# 190. Circuit eviction

Podrá aplicarse:

```text
inactive circuit TTL
```

solo para housekeeping.

---

# 191. Eviction ≠ circuit recovery

Eliminar un circuito de memoria no deberá utilizarse como mecanismo semántico para declarar endpoint recuperado.

---

# 192. Testing architecture

El sistema deberá soportar:

```text
FakeClock
FakeCircuitStateStore
DeterministicWindow
FailureInjector
FakeProbeCoordinator
```

---

# 193. Closed test

```text
State = CLOSED
```

Esperado:

```text
normal request allowed
```

---

# 194. Threshold test

Con:

```text
minimum throughput = 10
threshold = 50%
```

y:

```text
6 failures / 10 calls
```

esperado:

```text
OPEN
```

---

# 195. Minimum throughput test

Con:

```text
2 failures / 2 calls
minimum throughput = 10
```

esperado:

```text
remain CLOSED
```

salvo fast-trip policy explícita.

---

# 196. Open rejection test

```text
State = OPEN
```

Esperado:

```text
request not sent
Outcome = NOT_EXECUTED
```

---

# 197. Cooldown test

Antes del cooldown:

```text
OPEN
```

Después:

```text
HALF_OPEN eligibility
```

---

# 198. Probe concurrency test

Con:

```text
max probes = 2
```

y 100 concurrent requests:

```text
at most 2 probe permits
```

---

# 199. Probe success test

Después de suficientes probes exitosos:

```text
HALF_OPEN
→
CLOSED
```

---

# 200. Probe failure test

```text
HALF_OPEN
→ relevant failure
→ OPEN
```

---

# 201. Retry integration test

Retry hacia OPEN endpoint:

```text
rejected before database send
```

---

# 202. Alternative replica test

```text
Replica A OPEN
Replica B CLOSED
```

B solo podrá seleccionarse si:

```text
health
freshness
consistency
routing
```

también lo permiten.

---

# 203. Writer safety test

Writer A OPEN.

Replica B saludable.

Esperado:

```text
no automatic write to B
```

---

# 204. Transaction test

Circuit opens mientras existe una transaction saludable ya pinned.

Esperado:

```text
transaction not automatically migrated
```

---

# 205. Shard isolation test

Shard A OPEN.

Shard B CLOSED.

Esperado:

```text
B remains available
```

---

# 206. Application error test

100 unique constraint violations.

Esperado:

```text
circuit remains unaffected
```

---

# 207. Cancellation test

100 user cancellations.

Esperado:

```text
no infrastructure circuit opening
```

---

# 208. Slow-call test

Queries lentas inherentes al workload no deberán abrir circuito sin clasificación apropiada.

---

# 209. Persistent worker test

Request A produce failures.

Request B llega posteriormente al mismo worker.

Esperado:

```text
circuit history preserved
```

---

# 210. Request isolation test

Probe permit de Request A no será accesible por Request B.

---

# 211. Shared coordination test

Múltiples workers deberán respetar:

```text
global probe budget
```

cuando shared coordination esté habilitada.

---

# 212. Shared provider failure test

Verificar política:

```text
FAIL_OPEN
FAIL_CLOSED
LOCAL_FALLBACK
```

---

# 213. Topology replacement test

Endpoint con nueva generation no heredará estado incompatible accidentalmente.

---

# 214. Manual override test

`FORCED_OPEN` deberá rechazar tráfico independientemente del failure rate.

---

# 215. Directory structure

```text
src/Quantum/Database/Resilience/CircuitBreaker/
│
├── Contract/
│   ├── CircuitBreaker.php
│   ├── CircuitPolicy.php
│   ├── CircuitRegistry.php
│   ├── CircuitStateStore.php
│   ├── CircuitFailureClassifier.php
│   └── ProbeCoordinator.php
│
├── Model/
│   ├── CircuitId.php
│   ├── CircuitKey.php
│   ├── CircuitState.php
│   ├── CircuitOperationalState.php
│   ├── CircuitScope.php
│   ├── CircuitGeneration.php
│   ├── CircuitSnapshot.php
│   ├── CircuitObservation.php
│   ├── CircuitObservationType.php
│   ├── CircuitImpact.php
│   ├── CircuitDecision.php
│   ├── CircuitRequest.php
│   ├── CircuitPermit.php
│   └── CircuitPermitType.php
│
├── Window/
│   ├── CircuitWindow.php
│   ├── CircuitWindowType.php
│   ├── CountBasedWindow.php
│   ├── TimeBasedWindow.php
│   ├── RingBufferWindow.php
│   └── BucketedTimeWindow.php
│
├── Policy/
│   ├── DefaultCircuitPolicy.php
│   ├── FailureRatePolicy.php
│   ├── SlowCallPolicy.php
│   ├── ConsecutiveFailurePolicy.php
│   ├── CompositeCircuitPolicy.php
│   └── AdaptiveCooldownPolicy.php
│
├── Probe/
│   ├── ProbePermit.php
│   ├── HalfOpenProbeBudget.php
│   ├── LocalProbeCoordinator.php
│   └── SharedProbeCoordinator.php
│
├── Store/
│   ├── LocalCircuitStateStore.php
│   └── SharedCircuitStateStore.php
│
├── Registry/
│   └── DefaultCircuitRegistry.php
│
├── Integration/
│   ├── RetryCircuitIntegration.php
│   ├── RoutingCircuitIntegration.php
│   ├── PoolCircuitIntegration.php
│   ├── ReplicaCircuitIntegration.php
│   ├── FailoverCircuitIntegration.php
│   └── HealthCircuitIntegration.php
│
├── Override/
│   ├── CircuitOverride.php
│   └── CircuitOverrideRegistry.php
│
├── Runtime/
│   ├── CircuitOperationContext.php
│   └── CircuitPermitResetter.php
│
├── Telemetry/
│   └── CircuitBreakerTelemetry.php
│
├── Diagnostics/
│   ├── CircuitDiagnostics.php
│   └── CircuitExplainResult.php
│
├── Testing/
│   ├── FakeCircuitBreaker.php
│   ├── FakeCircuitStateStore.php
│   ├── FakeProbeCoordinator.php
│   └── CircuitScenarioBuilder.php
│
└── Exception/
    ├── CircuitBreakerException.php
    ├── CircuitOpenException.php
    ├── CircuitProbeRejectedException.php
    ├── CircuitStateException.php
    ├── CircuitCoordinationException.php
    └── CircuitConfigurationException.php
```

---

# 216. Dependencias

```text
Failure Handling
       ↓
Circuit Failure Classification
       ↓
Circuit Breaker
       ↓
Target Eligibility
```

Integraciones laterales:

```text
Circuit Breaker
├── Connection Manager
├── Connection Pool
├── Read/Write Routing
├── Replica System
├── Retry Policy
├── Failover
├── Health
├── Telemetry
├── Runtime
└── Resource Governance
```

---

# 217. Dependencias prohibidas

Circuit Breaker core no dependerá directamente de:

```text
ORM entities
Model API
HTTP controllers
Livewire-like runtime
application business logic
specific cache provider
specific Redis client
specific queue provider
```

---

# 218. Invariantes arquitectónicas

## DB-CIRCUIT-001
Circuit Breaker no será Retry System.

## DB-CIRCUIT-002
Circuit Breaker no será Failover System.

## DB-CIRCUIT-003
Circuit Breaker no será Health Check System.

## DB-CIRCUIT-004
Circuit Breaker no será Load Balancer.

## DB-CIRCUIT-005
Circuit Breaker no será Connection Pool.

## DB-CIRCUIT-006
Circuit Breaker controlará admission hacia recursos degradados.

## DB-CIRCUIT-007
CLOSED permitirá tráfico normal.

## DB-CIRCUIT-008
OPEN rechazará tráfico normal.

## DB-CIRCUIT-009
HALF_OPEN permitirá únicamente tráfico controlado.

## DB-CIRCUIT-010
HALF_OPEN no será equivalente a healthy.

## DB-CIRCUIT-011
OPEN no será equivalente a permanently failed.

## DB-CIRCUIT-012
CircuitState no será ResourceHealth.

## DB-CIRCUIT-013
Circuit identity será estable.

## DB-CIRCUIT-014
Connection individual no será Circuit.

## DB-CIRCUIT-015
Pool no será Endpoint Circuit.

## DB-CIRCUIT-016
Circuit scope será explícito.

## DB-CIRCUIT-017
Endpoint será scope principal recomendado.

## DB-CIRCUIT-018
Replica podrá tener circuito independiente.

## DB-CIRCUIT-019
Writer podrá tener circuito independiente.

## DB-CIRCUIT-020
Shard endpoint podrá tener circuito independiente.

## DB-CIRCUIT-021
Failure de una replica no abrirá automáticamente circuito global.

## DB-CIRCUIT-022
Failure de un shard no abrirá automáticamente otros shards.

## DB-CIRCUIT-023
Query error no será infrastructure failure.

## DB-CIRCUIT-024
Syntax error no afectará circuito.

## DB-CIRCUIT-025
Constraint violation no afectará circuito.

## DB-CIRCUIT-026
Invalid parameter no afectará circuito.

## DB-CIRCUIT-027
User cancellation no afectará circuito por default.

## DB-CIRCUIT-028
Circuit rejection no se contará como database failure.

## DB-CIRCUIT-029
Deadlock no abrirá circuito por default.

## DB-CIRCUIT-030
Serialization conflict no abrirá circuito por default.

## DB-CIRCUIT-031
Connection refusal podrá ser señal positiva de failure.

## DB-CIRCUIT-032
Network timeout podrá ser señal positiva de failure.

## DB-CIRCUIT-033
Server unavailable podrá ser señal positiva de failure.

## DB-CIRCUIT-034
Failure classification será semántica.

## DB-CIRCUIT-035
UNKNOWN no será automáticamente failure.

## DB-CIRCUIT-036
UNKNOWN no será automáticamente success.

## DB-CIRCUIT-037
Circuit window será bounded.

## DB-CIRCUIT-038
Observation history no crecerá indefinidamente.

## DB-CIRCUIT-039
Count window tendrá tamaño acotado.

## DB-CIRCUIT-040
Time window tendrá retención acotada.

## DB-CIRCUIT-041
Failure rate utilizará observaciones relevantes.

## DB-CIRCUIT-042
Minimum throughput evitará apertura por muestras triviales.

## DB-CIRCUIT-043
Failure threshold será configurable.

## DB-CIRCUIT-044
Slow-call threshold será configurable.

## DB-CIRCUIT-045
Slow query no será automáticamente slow endpoint.

## DB-CIRCUIT-046
Query cost será considerada cuando corresponda.

## DB-CIRCUIT-047
Opening transition será concurrency-safe.

## DB-CIRCUIT-048
Closing transition será concurrency-safe.

## DB-CIRCUIT-049
Half-open transition será concurrency-safe.

## DB-CIRCUIT-050
OPEN tendrá cooldown explícito.

## DB-CIRCUIT-051
Cooldown podrá ser adaptativo.

## DB-CIRCUIT-052
Adaptive cooldown tendrá límites.

## DB-CIRCUIT-053
HALF_OPEN tendrá probe budget.

## DB-CIRCUIT-054
Probe concurrency será limitada.

## DB-CIRCUIT-055
Probe permit será explícito.

## DB-CIRCUIT-056
Normal retry no será automáticamente probe.

## DB-CIRCUIT-057
Probe permit será operation-scoped.

## DB-CIRCUIT-058
Probe permit deberá liberarse.

## DB-CIRCUIT-059
Probe permit no sobrevivirá request accidentalmente.

## DB-CIRCUIT-060
Sufficient probe success podrá cerrar circuito.

## DB-CIRCUIT-061
Relevant probe failure podrá reabrir circuito.

## DB-CIRCUIT-062
1000 requests no producirán 1000 probes si budget es menor.

## DB-CIRCUIT-063
OPEN permitirá fail-fast.

## DB-CIRCUIT-064
Circuit rejection implicará operation not sent.

## DB-CIRCUIT-065
Circuit rejection podrá producir Outcome NOT_EXECUTED.

## DB-CIRCUIT-066
Retry consultará circuit eligibility antes del attempt.

## DB-CIRCUIT-067
Retry no insistirá infinitamente sobre target OPEN.

## DB-CIRCUIT-068
Retry podrá excluir temporalmente targets rechazados.

## DB-CIRCUIT-069
Circuit Breaker no ejecutará failover.

## DB-CIRCUIT-070
Circuit Breaker podrá aportar señal a failover.

## DB-CIRCUIT-071
Writer OPEN no autorizará escritura en replica.

## DB-CIRCUIT-072
Writer OPEN no demostrará pérdida de autoridad.

## DB-CIRCUIT-073
Failover deberá proteger contra split brain.

## DB-CIRCUIT-074
Replica OPEN podrá excluirse del read balancing.

## DB-CIRCUIT-075
OPEN endpoint no será target normal de load balancing.

## DB-CIRCUIT-076
HALF_OPEN endpoint no será target normal de load balancing.

## DB-CIRCUIT-077
Load Balancer elegirá únicamente targets elegibles.

## DB-CIRCUIT-078
Circuit eligibility no reemplazará consistency eligibility.

## DB-CIRCUIT-079
Circuit eligibility no reemplazará freshness eligibility.

## DB-CIRCUIT-080
Circuit eligibility no reemplazará role eligibility.

## DB-CIRCUIT-081
Circuit eligibility no reemplazará shard ownership.

## DB-CIRCUIT-082
Connection Pool podrá consultar circuit antes de crear connection.

## DB-CIRCUIT-083
OPEN no implicará cerrar todos los sockets.

## DB-CIRCUIT-084
Connection health seguirá siendo independiente.

## DB-CIRCUIT-085
Pool exhaustion no abrirá endpoint circuit automáticamente.

## DB-CIRCUIT-086
Endpoint overload podrá ser circuit signal.

## DB-CIRCUIT-087
Health probes podrán aportar evidencia.

## DB-CIRCUIT-088
Health probe success no demostrará workload recovery.

## DB-CIRCUIT-089
Passive traffic y active probes serán distinguibles.

## DB-CIRCUIT-090
Transaction activa no migrará por circuit state.

## DB-CIRCUIT-091
Circuit opening no abortará automáticamente transaction saludable.

## DB-CIRCUIT-092
Transaction start podrá consultar circuit eligibility.

## DB-CIRCUIT-093
Shard circuit será aislado por shard.

## DB-CIRCUIT-094
Distributed execution conservará shard failure information.

## DB-CIRCUIT-095
Circuit-open shard no producirá fake complete result.

## DB-CIRCUIT-096
Circuit state normalmente será infrastructure-scoped.

## DB-CIRCUIT-097
Circuit state no será tenant-scoped por default.

## DB-CIRCUIT-098
Tenant application errors no contaminarán shared endpoint circuit.

## DB-CIRCUIT-099
Circuit keys evitarán high cardinality.

## DB-CIRCUIT-100
Raw SQL no formará parte de circuit key por default.

## DB-CIRCUIT-101
Request ID no formará parte de circuit key.

## DB-CIRCUIT-102
Trace ID no formará parte de circuit key.

## DB-CIRCUIT-103
User ID no formará parte de circuit key.

## DB-CIRCUIT-104
Local state será soportado.

## DB-CIRCUIT-105
Shared state será opcional.

## DB-CIRCUIT-106
Core no dependerá obligatoriamente de Redis.

## DB-CIRCUIT-107
Shared transitions serán atómicas.

## DB-CIRCUIT-108
Shared probe budget será coordinado.

## DB-CIRCUIT-109
Shared provider failure tendrá policy explícita.

## DB-CIRCUIT-110
Local fallback podrá ser soportado.

## DB-CIRCUIT-111
Circuit state podrá sobrevivir múltiples requests.

## DB-CIRCUIT-112
Circuit state no deberá resetearse por request.

## DB-CIRCUIT-113
Operation permit sí será scoped.

## DB-CIRCUIT-114
Worker persistence no filtrará permits.

## DB-CIRCUIT-115
Circuit registry será concurrency-safe.

## DB-CIRCUIT-116
Circuit snapshots serán inmutables.

## DB-CIRCUIT-117
Clock será abstraído.

## DB-CIRCUIT-118
Elapsed-time calculations evitarán wall-clock anomalies cuando sea posible.

## DB-CIRCUIT-119
Topology generation podrá participar en circuit identity.

## DB-CIRCUIT-120
Replaced endpoint no heredará estado incompatible.

## DB-CIRCUIT-121
Writer promotion reevaluará circuit semantics.

## DB-CIRCUIT-122
Removed endpoints podrán evacuarse del registry.

## DB-CIRCUIT-123
Circuit eviction será housekeeping, no recovery semantics.

## DB-CIRCUIT-124
Manual reset será explícito.

## DB-CIRCUIT-125
Manual reset será auditable.

## DB-CIRCUIT-126
FORCED_OPEN será distinguible de observed OPEN.

## DB-CIRCUIT-127
FORCED_CLOSED será distinguible de observed CLOSED.

## DB-CIRCUIT-128
DISABLED no significará healthy.

## DB-CIRCUIT-129
Administrative override será visible en diagnostics.

## DB-CIRCUIT-130
Circuit telemetry tendrá cardinalidad acotada.

## DB-CIRCUIT-131
Telemetry no expondrá credentials.

## DB-CIRCUIT-132
Telemetry no expondrá raw bindings sensibles.

## DB-CIRCUIT-133
Circuit opening emitirá señal observable.

## DB-CIRCUIT-134
Circuit closing emitirá señal observable.

## DB-CIRCUIT-135
Circuit rejection será observable.

## DB-CIRCUIT-136
Probe outcome será observable.

## DB-CIRCUIT-137
Successful query no requerirá evento costoso por default.

## DB-CIRCUIT-138
Profiler distinguirá not-executed de slow execution.

## DB-CIRCUIT-139
Retry Budget será independiente del Circuit Breaker.

## DB-CIRCUIT-140
Retry backoff será independiente del Circuit Breaker.

## DB-CIRCUIT-141
Circuit cooldown será independiente de Retry backoff.

## DB-CIRCUIT-142
Circuit OPEN no consumirá DB execution timeout.

## DB-CIRCUIT-143
Circuit rejection deberá ser rápida.

## DB-CIRCUIT-144
Circuit state deberá ser explicable.

## DB-CIRCUIT-145
Opening reason deberá ser estructurado.

## DB-CIRCUIT-146
Observed failure rate deberá ser diagnosticable.

## DB-CIRCUIT-147
Observed slow rate deberá ser diagnosticable.

## DB-CIRCUIT-148
Probe budget deberá ser diagnosticable.

## DB-CIRCUIT-149
Cooldown remaining deberá ser diagnosticable.

## DB-CIRCUIT-150
Failure evidence deberá conservar confidence cuando corresponda.

## DB-CIRCUIT-151
Application errors no degradarán infrastructure confidence.

## DB-CIRCUIT-152
Circuit Breaker deberá proteger recursos locales indirectamente.

## DB-CIRCUIT-153
Circuit Breaker no sustituirá Resource Governance.

## DB-CIRCUIT-154
Circuit Breaker no sustituirá Backpressure System.

## DB-CIRCUIT-155
Circuit Breaker no sustituirá Rate Limiting.

## DB-CIRCUIT-156
Circuit Breaker no sustituirá Timeout.

## DB-CIRCUIT-157
Circuit Breaker no sustituirá Cancellation.

## DB-CIRCUIT-158
Circuit Breaker no sustituirá Retry.

## DB-CIRCUIT-159
Circuit Breaker no sustituirá Failover.

## DB-CIRCUIT-160
Circuit Breaker no sustituirá Health Monitoring.

## DB-CIRCUIT-161
Circuit state deberá preservar logical database boundaries.

## DB-CIRCUIT-162
Circuit state deberá preservar endpoint roles.

## DB-CIRCUIT-163
Circuit state deberá preservar shard boundaries.

## DB-CIRCUIT-164
Circuit state no cambiará tenant context.

## DB-CIRCUIT-165
Circuit state no cambiará transaction context.

## DB-CIRCUIT-166
Circuit state no cambiará query semantics.

## DB-CIRCUIT-167
Circuit state no cambiará consistency requirements.

## DB-CIRCUIT-168
Circuit state no cambiará authorization requirements.

## DB-CIRCUIT-169
Circuit opening deberá priorizar containment sobre transparent retry.

## DB-CIRCUIT-170
Circuit closing deberá requerir evidencia de recuperación.

## DB-CIRCUIT-171
Un solo probe exitoso no deberá implicar recuperación salvo policy explícita.

## DB-CIRCUIT-172
Half-open deberá limitar recovery traffic.

## DB-CIRCUIT-173
Recovery no deberá producir thundering herd.

## DB-CIRCUIT-174
Circuit Breaker deberá operar antes de adquirir recursos costosos cuando sea posible.

## DB-CIRCUIT-175
Rejected operation no deberá crear prepared statement.

## DB-CIRCUIT-176
Rejected operation no deberá abrir conexión nueva innecesariamente.

## DB-CIRCUIT-177
Rejected operation no deberá ejecutarse parcialmente.

## DB-CIRCUIT-178
Circuit state provider failure no deberá quedar implícito.

## DB-CIRCUIT-179
Unknown circuit coordination state deberá ser explícito.

## DB-CIRCUIT-180
Correctness y containment tendrán prioridad sobre transparent availability.

---

# 219. Modelo formal

Sea un circuito:

```text
C
```

con estado:

```text
S(C) ∈ {CLOSED, OPEN, HALF_OPEN}
```

y una ventana:

```text
W(C)
```

de observaciones relevantes.

Definimos:

```text
N(C) = total relevant calls
F(C) = relevant failures
L(C) = relevant slow calls
```

---

# 220. Failure rate

```text
FailureRate(C) =
F(C) / N(C)
```

cuando:

```text
N(C) > 0
```

---

# 221. Slow rate

```text
SlowRate(C) =
L(C) / N(C)
```

---

# 222. Opening rule

Para:

```text
M = minimum throughput
Tf = failure threshold
Ts = slow threshold
```

puede definirse:

```text
Open(C) =
N(C) >= M
∧
(
    FailureRate(C) >= Tf
    ∨
    SlowRate(C) >= Ts
)
```

más cualquier:

```text
FastTripCondition
```

explícita.

---

# 223. Admission function

```text
Admit(C, operation)
=
true
```

si:

```text
S(C) = CLOSED
```

Para:

```text
S(C) = OPEN
```

por default:

```text
Admit(C, operation) = false
```

Para:

```text
S(C) = HALF_OPEN
```

solo:

```text
Admit(C, operation)
=
ProbePermitAvailable(C)
```

---

# 224. Half-open recovery

Sea:

```text
Ps = successful probes
Pr = required successful probes
```

Entonces:

```text
Ps >= Pr
```

podrá producir:

```text
HALF_OPEN → CLOSED
```

si no existen otras condiciones que lo impidan.

---

# 225. Half-open failure

Para una failure relevante:

```text
HALF_OPEN
+
RelevantFailure
→
OPEN
```

por default.

---

# 226. Circuit versus retry

Formalmente:

```text
RetryEligible(O)
```

no implica:

```text
TargetAdmissible(T)
```

La ejecución requiere ambas:

```text
ExecuteRetry(O,T)
=
RetryEligible(O)
∧
TargetAdmissible(T)
```

---

# 227. Circuit versus routing

```text
RouteEligible(T)
=
RoleEligible(T)
∧
HealthEligible(T)
∧
CircuitEligible(T)
∧
ConsistencyEligible(T)
∧
ShardEligible(T)
```

Solo entonces:

```text
LoadBalancer
```

podrá seleccionar `T`.

---

# 228. Flujo final

```text
Query / Transaction / Connection Request
                  │
                  ▼
           Routing Context
                  │
                  ▼
           Candidate Targets
                  │
                  ▼
          Role / Shard Filter
                  │
                  ▼
             Health Filter
                  │
                  ▼
            Circuit Filter
                  │
          ┌───────┴────────┐
          │                │
       Eligible        OPEN / blocked
          │                │
          ▼                ▼
   Consistency Filter   Fail Fast /
          │             Alternate Target /
          ▼             Failover Evaluation
    Load Balancer
          │
          ▼
   Circuit Admission
          │
          ▼
    Connection / Query
          │
          ▼
       Outcome
          │
          ▼
 Failure Classification
          │
          ▼
 Circuit Observation
          │
          ▼
 Window Aggregation
          │
          ▼
 Circuit Policy
          │
          ▼
 CLOSED / OPEN / HALF_OPEN
```

---

# 229. Regla maestra

> **VoltStack utilizará Circuit Breakers como mecanismo de contención de fallos, no como mecanismo de corrección de resultados. Un circuito podrá impedir que nuevas operaciones alcancen un recurso degradado, pero nunca deberá alterar silenciosamente la semántica de una consulta, cambiar de shard, degradar consistencia, promover un writer, reintentar una transacción o declarar recuperado un recurso sin que los subsistemas responsables y la evidencia disponible lo permitan.**

En forma compacta:

```text
Failure
≠
Circuit Open

Slow Query
≠
Slow Endpoint

Connection Failure
≠
Endpoint Failure

OPEN
≠
Permanent Failure

HALF_OPEN
≠
Healthy

Health Check
≠
Circuit Breaker

Retry
≠
Circuit Breaker

Failover
≠
Circuit Breaker

Load Balancing
≠
Circuit Breaker

Writer OPEN
≠
Replica May Write

Circuit Rejection
=
Operation Not Sent

Recovery
≠
One Successful Probe
```

Prioridad:

```text
Correctness
    >
Authority Safety
    >
Shard Integrity
    >
Consistency
    >
Failure Containment
    >
Resource Protection
    >
Availability
    >
Transparent Recovery
    >
Convenience
```

---

# 230. Estado del Bloque 23

```text
BLOCK 23 — RESILIENCE

✓ 235_DATABASE_RESILIENCE_ARCHITECTURE.md
✓ 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
✓ 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
✓ 238_DATABASE_RETRY_POLICY_SYSTEM.md
✓ 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
○ 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
○ 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 231. Siguiente documento

```text
240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
```

El siguiente documento definirá la arquitectura mediante la cual VoltStack responderá a la pérdida o degradación de endpoints de base de datos y podrá seleccionar recursos alternativos sin violar autoridad, consistencia, transaction pinning, shard ownership ni garantías de resultado.

Se desarrollarán:

```text
Failover Architecture

Failure Domain
Endpoint Failure
Replica Failure
Writer Failure
Shard Failure

Failover Candidate
Failover Eligibility
Failover Decision
Failover Coordinator

Authority Model
Writer Authority
Authority Epoch
Fencing Token
Split-Brain Prevention

Automatic Failover
Manual Failover
Planned Switchover
Emergency Failover

Replica Promotion
Writer Replacement
Topology Reconciliation

Read Failover
Write Failover
Connection Failover
Transaction Failure

Recovery State Machine
DEGRADED
FAILING_OVER
RECOVERING
HEALTHY
UNKNOWN

Failback
Rejoin
Catch-up
Replica Reintegration

Retry Integration
Circuit Breaker Integration
Routing Integration
Load Balancing Integration
Connection Pool Integration

Shard-Aware Failover
Distributed Recovery

Unknown Outcome Handling
Commit Ambiguity
Consistency Preservation

Persistent Runtime Safety
Telemetry
Diagnostics
Testing
```

La regla central será:

> **Failover en VoltStack no significa simplemente “usar otro servidor”; significa cambiar de recurso únicamente cuando la nueva ruta conserva o restablece una autoridad válida y cuando VoltStack puede hacerlo sin inventar el resultado de operaciones anteriores ni violar las garantías de consistencia, transacción o distribución.**