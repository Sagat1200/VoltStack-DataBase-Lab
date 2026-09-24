# 288_DATABASE_DATABASE_FAKE_AND_MOCK_SYSTEM.md

**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**Estado:** Architectural Specification


## 01. Propósito

Definir la arquitectura oficial de dobles de prueba de `VoltStack/Quantum/Database` V1. El sistema permite aislar código de aplicación, verificar contratos de interacción, simular errores y ejecutar pruebas deterministas sin confundir simulación con fidelidad real de un motor SQL.


## 02. Principio central

`Fakes prove application logic; mocks prove interactions; real databases prove database semantics.` Ningún doble de prueba sustituye las pruebas de integración sobre el motor utilizado en producción.


## 03. Objetivos

Proveer fakes, stubs, spies y mocks; soportar queries, resultados, conexiones y transacciones; facilitar fault injection, assertions, fixtures, aislamiento por test y observabilidad; integrarse con PHPUnit/Pest y el lifecycle persistente.


## 04. No objetivos

No implementar un motor SQL universal in-memory, un optimizador real, todas las peculiaridades de MySQL/PostgreSQL/SQLite/SQL Server, ni certificar integridad referencial o locking mediante mocks.


## 05. Terminología

**Stub:** respuestas predefinidas. **Fake:** implementación simplificada funcional. **Spy:** registra llamadas. **Mock:** expectativas verificables. **Real test database:** motor real aislado para integración. **Contract test:** ejecuta una misma especificación contra implementaciones compatibles.


## 06. Posición arquitectónica

El paquete vive en `VoltStack/Quantum/Database/Testing` o módulo de testing asociado, sin obligar al runtime productivo a cargar dependencias de PHPUnit/Pest.


## 07. Diagrama general

```text
Application / Repository / Service
              │
       Database Contract
              │
   ┌──────────┼─────────────┐
   ▼          ▼             ▼
 Real DB    Fake DB      Mock/Spy
   │          │             │
Integration  Logic      Interaction
   └──────────┼─────────────┘
              ▼
       Contract Evidence
```


## 08. Reglas de selección

Usar stub para respuesta fija, fake para lógica con estado, spy para observación, mock para interacción y base real para SQL, schema, constraints, isolation, concurrency y performance. Evitar mocks profundos de detalles internos del ORM.


## 09. Componentes

`DatabaseFakeAndMockSystem`, `FakeConnection`, `FakeConnectionFactory`, `FakeQueryExecutor`, `FakeResultSet`, `FakeTransactionManager`, `FakeDatabaseStore`, `FakeSchemaCatalog`, `DatabaseSpy`, `DatabaseMock`, `DatabaseExpectation`, `DatabaseFixtureLoader`, `DatabaseFaultInjector`, `DatabaseTestClock`, `DatabaseTestContext`, `DatabaseContractTestSuite`, `DatabaseTestResetManager`.


## 10. Contratos estables

Los dobles implementarán interfaces públicas de conexión, ejecución, resultado y transacción cuando corresponda. No deberán depender de reflection sobre internals ni reemplazar métodos finales por hacks.


## 11. FakeConnection

Representa una conexión lógica con identity, role, tenant, transaction state y query history. No abrirá sockets ni asumirá capacidades no implementadas.


## 12. FakeQueryExecutor

Podrá registrar `QueryIntent`, parámetros, operación y resultado. La evaluación de AST será un subconjunto explícito y versionado; SQL arbitrario no se interpretará silenciosamente.


## 13. FakeDatabaseStore

Almacén in-memory por tablas lógicas, claves y filas. Soportará inserción, actualización, eliminación y selección dentro del subconjunto declarado, con snapshots y reset por test.


## 14. FakeSchemaCatalog

Catálogo declarativo de tablas, columnas, claves y relaciones usado para fixtures y validación superficial. No simula automáticamente constraints, collation, triggers ni DDL vendor-specific.


## 15. FakeResultSet

Soportará scalar, row, rowset, empty, affected rows, generated identifiers y streams simulados. Distinguirá `null`, campo ausente y colección vacía.


## 16. FakeTransactionManager

Permitirá begin/commit/rollback, snapshots y savepoints si se declara esa capacidad. No afirmará reproducir MVCC, deadlocks o isolation levels reales.


## 17. Mock expectations

Expectativas declarativas de método/operación, parámetros, frecuencia, orden y resultado. La verificación ocurrirá al terminar el test y mostrará diferencias legibles.


## 18. Spy

Capturará query identity, SQL normalizado o AST, bindings redactados, conexión, tenant, transaction scope, duración simulada y eventos, con capacidad de filtrar/assert.


## 19. Stub

Respuestas fijas por operación, fingerprint o callback controlado. Una respuesta no configurada deberá producir diagnóstico o política explícita, no éxito implícito.


## 20. Fault injection

Inyectará timeout, connection loss, deadlock, serialization failure, constraint violation, read-only violation, commit outcome unknown, stale schema y exception during hydration. Debe indicar fase exacta: before execute, after execute, before commit, after commit.


## 21. Determinismo

Controlar clock, UUID/ID generator, orden de fixtures, secuencia de fallos y semillas. Los tests no dependerán de estado global ni orden de ejecución.


## 22. Test isolation

Crear contextos independientes por test, restaurar snapshots, limpiar expectations, spies, transacciones, identity maps, pools falsos y tenant state. Parallel tests requerirán namespaces independientes.


## 23. Tenant isolation

Cada fake store tendrá tenant scope explícito cuando Multitenancy esté instalado. Un tenant ID suministrado por request no reemplaza resolución/autorización. Probar fugas entre tenants y evitar claves de cache compartidas.


## 24. Security

Los dobles no deben normalizar prácticas inseguras: distinguir SQL parametrizado, raw SQL y identifiers; redaccionar secretos y PII; permitir simular denegaciones de policy y read-only.


## 25. ORM testing

Probar repositories, mappers, hydration, Unit of Work, dirty tracking y lifecycle mediante contratos públicos. La equivalencia de relaciones, cascade, SQL generado y constraints requiere suites con DB real.


## 26. Transaction testing

Verificar límites de begin/commit/rollback, propagación de errores, retries e idempotencia. Los outcomes ambiguos deben conservar estado `UNKNOWN` y no disparar fallback automático de escritura.


## 27. Connection testing

Simular role, database identity, TLS capabilities, credential resolution failure, reconnect y pool exhaustion sin afirmar que TLS real fue verificado.


## 28. Query testing

Verificar filtros, bindings, paginación y orden lógico en AST. Para semántica SQL, NULL, collation, JSON, fechas y funciones del motor usar integración real.


## 29. Migration testing

Permitir dry-run, journaling, checkpoints y simulación de errores de migrations. Aplicación real de DDL y recuperación de datos requieren entorno de integración aislado.


## 30. Telemetry

Emitir eventos de prueba y métricas con namespace separado, query fingerprints y bindings redactados. Spy debe poder verificar spans/eventos sin enviar datos a servicios externos.


## 31. PHPUnit/Pest

Ofrecer traits y helpers agnósticos al runner; adaptadores para PHPUnit y Pest deberán ser opcionales. No imponer dependencias dev en producción.


## 32. Developer API

```php
$database = DatabaseFake::create()
    ->withTable('users', [
        ['id' => 1, 'name' => 'Ada'],
    ]);

$database->expectQuery('users.by_id')
    ->withBindings([1])
    ->andReturnRow(['id' => 1, 'name' => 'Ada']);

$service = new UserService($database->connection());
$result = $service->find(1);

$database->assertQueryExecuted('users.by_id', times: 1);
$database->verify();
```


## 33. Fault example

```php
$database->faults()->once(
    phase: 'before_commit',
    fault: DatabaseFault::deadlock()
);

$service->transfer($source, $target, $amount);
$database->assertRollbackCount(1);
```


## 34. Spy example

```php
$spy = DatabaseSpy::wrap($connection);
$repository->findByEmail('a@example.test');
$spy->assertExecuted('users.by_email');
$spy->assertNoUnsafeInterpolation();
```


## 35. Capability declaration

Cada fake declarará `SUPPORTED`, `PARTIAL`, `UNSUPPORTED` por capacidad. No deberá convertir una operación unsupported en un resultado exitoso por conveniencia.


## 36. Contract tests

Una suite compartida verificará interfaces y comportamientos mínimos de real adapter y fake, con exclusions explícitas para diferencias inevitables.


## 37. Fidelity matrix

| Capacidad | Fake | Real DB |
|---|---|---|
| Flujo de repositorio | Sí | Sí |
| Expectativas de llamadas | Sí | Spy opcional |
| SQL vendor-specific | No | Sí |
| Constraints/triggers reales | No | Sí |
| MVCC/locking | No | Sí |
| Tenant context lógico | Sí | Sí |
| Performance real | No | Sí |
| Fallos deterministas | Sí | Mediante inyección/control |


## 38. FrankenPHP

El test runner deberá simular requests secuenciales sobre el mismo proceso: Tenant A → Tenant B → sin tenant, verificando reset de fake store scoped, context, transaction, spy, expectations y conexiones. Los datos de fixture compartidos deberán ser explícitos.


## 39. RoadRunner/OpenSwoole

Perfiles opcionales deberán probar coroutine-local context y evitar compartir stores/expectations mutables entre ejecuciones concurrentes.


## 40. Concurrency

Los fakes pueden simular scheduling y conflictos, pero no certificar condiciones de carrera reales del DB. Pruebas de deadlock, unique races e isolation se ejecutarán también con motor real.


## 41. CI strategy

PR: unit/contract/fake tests rápidos. Merge/release: DB integration matrix según plataformas soportadas. Nightly: concurrency, migration, fault injection y persistent-worker suites.


## 42. Diagnostics

Códigos `VSDB-TEST-FAKE-*`, `VSDB-TEST-MOCK-*`, `VSDB-TEST-FAULT-*`. Mostrar expectativa, operación observada, bindings sanitizados, scope, diferencia y pista de corrección.


## 43. Failure model

Estados `PASS`, `FAIL`, `INCONCLUSIVE`, `UNSUPPORTED`, `NOT_RUN`. Una operación unsupported o evidencia insuficiente nunca equivaldrá a PASS.


## 44. Integration with other docs

Se integra con Query AST/Compiler, Connections, ORM, Unit of Work, Transactions, Schema/Migrations, Security (226), Telemetry, Validation (337), CLI/DX (338), Reporting (339) y Recovery (340).


## 45. V1 scope

Incluye fakes básicos de conexión/query/result/store/transaction, mocks/spies/stubs, fixtures, reset, fault injection, capability declarations, contract tests y adapters de test runner. Excluye motor SQL universal, simulador MVCC completo, benchmark fidedigno y plataforma de chaos distribuido.


## 46. Decisiones arquitectónicas

1. Dobles basados en contratos públicos. 2. Fidelidad explícita por capacidad. 3. DB real obligatoria para semántica DB. 4. Estado aislado por test. 5. Fallos deterministas por fase. 6. No auto-success en operación desconocida. 7. Redacción de datos. 8. Integración runner opcional. 9. Reset obligatorio en persistent workers. 10. Mocks verifican interacción, no SQL real. 11. Sin dependencias dev en runtime productivo. 12. Contratos versionados.


## 47. Criterios de aceptación

Debe permitir inyectar dobles por Container, preparar fixtures, configurar resultados y errores, inspeccionar operaciones, verificar expectations, limpiar estado, declarar limitaciones y ejecutar la misma suite de contratos donde sea aplicable. La suite de seguridad y de FrankenPHP deberá detectar fugas.


## 48. Conclusión

`DatabaseFakeAndMockSystem` proporciona velocidad y control para probar la lógica de persistencia sin presentar una simulación como verdad del motor. Regla final: `Fake for speed. Mock for interaction. Spy for evidence. Real database for truth.`


---

**Documento:** `288_DATABASE_DATABASE_FAKE_AND_MOCK_SYSTEM.md`  
**Alcance:** Database V1
