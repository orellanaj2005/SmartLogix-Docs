# Patrones de diseño — SmartLogix

Resumen de los patrones aplicados y dónde se usan.

## Saga (con compensación) — Order ↔ Inventory

Coordina una transacción distribuida sin bloqueo a través de servicios.

- `Order-Service` reserva stock en `Inventory-Service` al crear un pedido.
- Si algo falla o se cancela, ejecuta la **compensación**: libera el stock reservado.
- Implementado en `PedidoSaga` / `PedidoActualizadoEvent`.

```
reservar  → POST /inventario/productos/{id}/reservar?cantidad=N
compensar → POST /inventario/productos/{id}/liberar?cantidad=N
```

## Circuit Breaker — Order-Service (Resilience4j)

Evita encadenar fallos cuando una dependencia (Inventory) no responde.

- Instancia `pedidos`: ventana de 5, umbral de fallo 50%, 10 s en estado abierto, 3 llamadas en *half-open*.
- Un `NotFoundException` (404 de negocio) **no** cuenta como fallo ni abre el circuito.
- Con el circuito abierto, el servicio responde **503**.

## Facade — Order y Notification

Expone una interfaz simplificada sobre la lógica de dominio y oculta la coordinación interna.

- `PedidoFacade` — fachada de las operaciones de pedidos (crear, estado, eliminar…).
- `NotificacionFacade` — fachada de notificaciones.
- En el **frontend**, `apiFacade` centraliza las llamadas HTTP sobre `httpClient` (axios).

## Observer — Notification y Order

Notifica a múltiples interesados ante un cambio de estado.

- **Notification-Service:** `EmailObserver`, `SMSObserver`, `WhatsAppObserver` reaccionan a cambios de estado de envíos (`NotificacionObserver`).
- **Order-Service:** `OrdenObserver` reacciona a cambios en pedidos.

## Strategy — User-Service (cifrado de contraseñas)

Permite intercambiar el algoritmo de cifrado sin tocar el código cliente.

- `PasswordEncoderStrategy` con implementaciones `BcryptPasswordStrategy` y `AesPasswordStrategy`.
- `PasswordEncoderContext` selecciona la estrategia.

## Repository — todos los servicios con BD

Abstrae el acceso a datos mediante interfaces `JpaRepository<T, ID>` de Spring Data. Ver [persistencia.md](persistencia.md).

## DTO (Data Transfer Object) — todos los servicios

Objetos inmutables (`record` de Java 21) cuya única responsabilidad es transportar datos.

- **Desacoplan** la API de las entidades JPA.
- **Ocultan** campos sensibles (p. ej. contraseñas).
- **Aplanan** relaciones perezosas para serializar a JSON sin sorpresas.
- Exponen un método de fábrica `from(entidad)` para concentrar el mapeo.

Categorías: salida, entrada (request), métrica/KPI, proyección de dashboard e integración (eventos Kafka).

## Singleton (gestionado por Spring)

Los `@Service`, `@Component`, `@Configuration` y repositorios son *beans* singleton gestionados por el contenedor de Spring. `JwtUtil` además expone una instancia accesible estáticamente para los filtros.
