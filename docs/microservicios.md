# Microservicios — SmartLogix

Detalle de cada componente: responsabilidad, puerto, esquema, dependencias, patrones y endpoints principales.

> Las rutas se muestran tal como las expone el servicio. A través del gateway llevan el prefijo `/api` (p. ej. `GET /api/inventario/productos`).

---

## Api-Gateway · `:8080`

**Responsabilidad:** punto de entrada único. Enrutamiento reactivo, validación JWT centralizada, CORS, rate limiting y Swagger UI agregado.

- **Stack:** Spring Cloud Gateway 2024.0.1 (WebFlux), JJWT, Redis reactivo, Actuator.
- **Infra:** Redis (rate limiter `RequestRateLimiter` + `RedisRateLimiter`, token-bucket por IP).
- **Sin base de datos.**
- **Config clave:** rutas, CORS global, `JwtAuthFilter`, límites `rate-limiter.replenish-rate`/`burst-capacity`.
- **Health:** `GET /actuator/health` · **Swagger:** `/swagger-ui.html`.

---

## User-Service · `:8084` · esquema `SL_USUARIOS`

**Responsabilidad:** autenticación (JWT) y gestión de usuarios.

- **Stack:** Spring Boot 3.4.5, Spring Security, JJWT, Spring Data JPA (Oracle), Redis.
- **Patrones:** **Strategy** (cifrado de contraseñas BCrypt / AES), blacklist de tokens en Redis, `GlobalExceptionHandler`.
- **Seguridad:** `JwtFilter` + `SecurityConfig` propios; cifrado AES (`AesUtil`).

**Endpoints — Autenticación (`/auth`, público salvo sesión):**

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/auth/login` | Login → `{ "token": "..." }` |
| POST | `/auth/logout` | Revoca el token actual |
| GET | `/auth/me` | Usuario autenticado |
| POST | `/auth/refresh` | Nuevo token, revoca el anterior |
| POST | `/auth/change-password` | Cambia contraseña y revoca token |

**Endpoints — Usuarios (`/usuarios`, requiere JWT):**

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/usuarios` · `/usuarios/dto` | Listar (entidad / DTO sin contraseña) |
| GET | `/usuarios/{id}` · `/usuarios/{id}/dto` | Obtener usuario |
| GET | `/usuarios/por-rol` | Métrica: usuarios por rol |
| POST | `/usuarios` | Crear |
| PUT | `/usuarios/{id}` | Actualizar |
| DELETE | `/usuarios/{id}` | Eliminar |

---

## Inventory-Service · `:8081` · esquema `SL_INVENTARIO`

**Responsabilidad:** productos, stock, métricas, alertas de stock bajo y participación en la saga de pedidos.

- **Stack:** Spring Boot 3.4.4, Spring Data JPA (Oracle), Redis (caché), Kafka (eventos), JJWT.
- **Patrones:** Repository, DTOs de proyección, `GlobalExceptionHandler`, evento Kafka `ALERTA_STOCK_BAJO`.
- **Config clave:** `inventario.stock.umbral-minimo` (por defecto 10).

**Endpoints principales (`/inventario`, requiere JWT):**

| Método | Ruta | Descripción |
|---|---|---|
| GET/POST | `/inventario/productos` | Listar / crear productos |
| GET/PUT/DELETE | `/inventario/productos/{id}` | Obtener / actualizar / eliminar |
| GET | `/inventario/productos/stock-bajo` | Productos con stock bajo |
| GET | `/inventario/dashboard/*` | Proyecciones para dashboard |
| GET | `/inventario/metricas/*` | Total, valor, stock promedio, etc. |
| GET | `/inventario/alertas/stock-bajo` | Alertas accionables |
| POST | `/inventario/productos/{id}/reservar?cantidad=N` | **Saga:** reservar stock |
| POST | `/inventario/productos/{id}/liberar?cantidad=N` | **Saga:** liberar stock (compensación) |

---

## Order-Service · `:8082` · esquema `SL_PEDIDOS`

**Responsabilidad:** registro de pedidos y gestión de su estado, coordinando stock con Inventory.

- **Stack:** Spring Boot 3.4.5, Spring Data JPA (Oracle), Resilience4j.
- **Patrones:** **Saga** (`PedidoSaga`), **Facade** (`PedidoFacade`), **Observer** (`OrdenObserver`), **Circuit Breaker** (instancia `pedidos`), `GlobalExceptionHandler` (404 / 503).
- **Estados:** 1 Pendiente · 2 Confirmado · 3 Procesando · 4 Completado · 5 Cancelado.
- **Seguridad:** confía en el JWT validado por el gateway.

**Endpoints (`/pedidos`):**

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/pedidos` | Listar |
| GET | `/pedidos/{id}` | Obtener (404 / 503) |
| GET | `/pedidos/{id}/estado` | Estado actual |
| POST | `/pedidos` | Crear (estado inicial Pendiente) |
| PATCH | `/pedidos/{id}/estado?estadoId=N` | Cambiar estado |
| DELETE | `/pedidos/{id}` | Eliminar |

---

## Notification-Service · `:8085` · esquema `SL_NOTIFICACIONES`

**Responsabilidad:** generar y consultar notificaciones a partir de cambios de estado de envíos.

- **Stack:** Spring Boot 3.4.5, Spring Data JPA (Oracle).
- **Patrones:** **Observer** (`EmailObserver`, `SMSObserver`, `WhatsAppObserver`), **Facade** (`NotificacionFacade`).
- **Seguridad:** confía en el JWT validado por el gateway.

**Endpoints (`/api/notificaciones`):**

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/notificaciones/envio/{envioId}/estado?estadoId=&descripcionEstado=` | Notificar cambio de estado |
| GET | `/api/notificaciones` | Listar todas |
| GET | `/api/notificaciones/envio/{envioId}` | Listar por envío |

> El repositorio remoto de este servicio se llama históricamente *Shipping-Service*.

---

## frontend_ms · `:80` (Docker) / `:5173` (Vite dev)

**Responsabilidad:** SPA que consume todo el backend a través del gateway.

- **Stack:** React 18, TypeScript, Vite, Redux Toolkit, axios, react-router-dom.
- **Build:** `npm run build` → servido por Nginx en producción.
- **Config:** usa una única URL de gateway (`VITE_API_GATEWAY`, por defecto `http://localhost:8080`).
- **Estructura:** páginas (auth, dashboard, usuarios, productos, inventario, pedidos), servicios con un *Facade* (`apiFacade`) sobre un `httpClient` axios, y estado en `store/slices`.
