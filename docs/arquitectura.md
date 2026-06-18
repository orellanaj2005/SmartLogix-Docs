# Arquitectura — SmartLogix

## 1. Visión general

SmartLogix sigue una arquitectura de **microservicios** con un **API Gateway** como único punto de entrada. Cada servicio:

- Tiene una **responsabilidad de negocio acotada** (un *bounded context*).
- Es dueño de su **propio esquema** en Oracle (*schema-per-service*); ningún servicio lee la BD de otro.
- Se construye, prueba y despliega de forma **independiente** (cada uno en su repositorio).
- Expone una **API REST** documentada con OpenAPI/Swagger.

## 2. Componentes

| Capa | Componente | Tecnología |
|---|---|---|
| Presentación | Frontend SPA | React 18 + Vite + TypeScript |
| Borde (edge) | API Gateway | Spring Cloud Gateway (WebFlux) |
| Aplicación | User, Inventory, Order, Notification | Spring Boot 3.4.x |
| Datos | Esquemas Oracle por servicio | Oracle Autonomous DB |
| Infraestructura | Redis, Kafka + Zookeeper | Redis 7, Confluent Platform 7.6 |

## 3. Flujo de una petición

```
Cliente (navegador)
   │  GET /api/inventario/productos  (Authorization: Bearer <JWT>)
   ▼
API Gateway :8080
   │  1. Aplica CORS y rate limiting (por IP, token-bucket en Redis)
   │  2. JwtAuthFilter valida la firma y expiración del JWT
   │  3. StripPrefix=1  →  /inventario/productos
   ▼
Inventory-Service :8081
   │  4. Procesa la petición (servicio + repositorio JPA)
   ▼
Oracle (SL_INVENTARIO)
```

- Las rutas **públicas** (`/api/auth/**`) no pasan por `JwtAuthFilter`.
- Las rutas **protegidas** requieren un JWT válido; el gateway lo valida **una sola vez** en el borde.
- El frontend **nunca** llama directamente a un microservicio: todo pasa por el gateway (origen único → CORS solo se configura allí).

## 4. Enrutamiento del gateway

| Prefijo | Destino | Puerto | JWT |
|---|---|:--:|:--:|
| `/api/auth/**` | User-Service | 8084 | No (público) |
| `/api/usuarios/**` | User-Service | 8084 | Sí |
| `/api/inventario/**` | Inventory-Service | 8081 | Sí |
| `/api/pedidos/**` | Order-Service | 8082 | Sí |
| `/api/envios/**` | (servicio de envíos) | 8083 | Sí · *planificado* |
| `/api/notificaciones/**` | Notification-Service | 8085 | Sí |
| `/v3/api-docs/*` | OpenAPI de cada servicio | — | No |

El gateway también expone un **Swagger UI agregado** en `/swagger-ui.html` con un selector por servicio.

## 5. Autenticación (JWT centralizado)

```
1. POST /api/auth/login  →  User-Service valida credenciales y emite un JWT (HS256)
2. El cliente guarda el token y lo envía en  Authorization: Bearer <token>
3. El gateway valida el token en cada petición protegida (JwtAuthFilter)
4. User-Service mantiene una blacklist en Redis para logout / refresh / cambio de contraseña
```

- El **secreto JWT es compartido** por el gateway y los servicios que validan token; se inyecta vía `.env` (`JWT_SECRET`).
- User-Service e Inventory-Service tienen además su propio `JwtFilter` (defensa en profundidad); Order y Notification confían en la validación del gateway.

## 6. Comunicación entre servicios — Saga de pedidos

El flujo de creación/cancelación de un pedido coordina Order ↔ Inventory mediante una **saga** con compensación:

```
Crear pedido:
   Order-Service  ──POST /inventario/productos/{id}/reservar?cantidad=N──►  Inventory-Service
                  ◄── 200 (stock reservado)  /  400 (stock insuficiente) ──

Cancelar / compensar:
   Order-Service  ──POST /inventario/productos/{id}/liberar?cantidad=N──►   Inventory-Service
```

- Order-Service protege estas llamadas con un **circuit breaker** (Resilience4j, instancia `pedidos`): si Inventory falla repetidamente, el circuito se abre y devuelve **503** en vez de encadenar fallos. Un **404** de negocio no abre el circuito.

## 7. Eventos (Kafka)

Inventory-Service publica eventos de dominio en Kafka, en particular **`ALERTA_STOCK_BAJO`** cuando un producto cae en o por debajo de su stock mínimo. El productor está configurado con *timeouts* cortos para no bloquear las operaciones si el broker no está disponible.

## 8. Infraestructura compartida

| Servicio | Uso de Redis | Uso de Kafka |
|---|---|---|
| Api-Gateway | Rate limiting (token-bucket por IP) | — |
| User-Service | Blacklist de tokens JWT | — |
| Inventory-Service | Caché de consultas (`@Cacheable`) | Eventos de stock |

En Docker Compose, Redis y Kafka+Zookeeper se levantan como contenedores con *healthchecks*; los microservicios esperan a que estén `healthy`.

## 9. Decisiones de diseño

- **Schema-per-service** para aislamiento de datos y despliegue independiente.
- **JWT validado en el borde** para no repetir la lógica de seguridad en cada servicio.
- **DTOs (`record` inmutables)** para desacoplar la API de las entidades JPA.
- **Sin procedimientos almacenados**: toda la lógica de consulta/agregación vive en JPQL (portabilidad Oracle ↔ H2).
- **Secretos fuera del código**: se inyectan vía `.env` / variables de entorno (ver [seguridad.md](seguridad.md)).
