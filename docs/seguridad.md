# Seguridad — SmartLogix

## 1. Autenticación JWT

- **Emisión:** `User-Service` genera un **JWT firmado con HS256** en `POST /auth/login`.
- **Validación centralizada:** el **API Gateway** valida firma y expiración en cada petición protegida (`JwtAuthFilter`), antes de reenviar al microservicio.
- **Defensa en profundidad:** User-Service e Inventory-Service tienen además su propio `JwtFilter`.
- **Confianza en el borde:** Order-Service y Notification-Service confían en la validación del gateway (no revalidan).

El **secreto JWT (`JWT_SECRET`) debe ser el mismo** en todos los componentes que firman/validan, por lo que se inyecta de forma centralizada vía `.env`.

### Ciclo de sesión y blacklist

`User-Service` mantiene una **blacklist de tokens en Redis** para revocación:

| Acción | Efecto |
|---|---|
| `POST /auth/logout` | Añade el token a la blacklist hasta su expiración |
| `POST /auth/refresh` | Revoca el token anterior y emite uno nuevo |
| `POST /auth/change-password` | Revoca el token actual (obliga a re-login) |

## 2. Cifrado de contraseñas (Strategy)

User-Service cifra contraseñas mediante el patrón **Strategy** (`PasswordEncoderContext`):

- `BcryptPasswordStrategy` — hashing con BCrypt.
- `AesPasswordStrategy` / `AesUtil` — cifrado AES/CBC reversible.
  - **`AES_SECRET`**: 32 bytes (AES-256). **`AES_IV`**: 16 bytes. Se usan como bytes UTF-8.

## 3. Gestión de secretos (sin filtraciones)

> Principio: **ningún secreto real está escrito en el código versionado**. Todos se inyectan por entorno.

### Variables sensibles

| Variable | Uso |
|---|---|
| `*_DATASOURCE_PASSWORD` | Contraseñas de los esquemas Oracle (uno por servicio) |
| `JWT_SECRET` | Firma/validación de JWT (compartido) |
| `AES_SECRET` / `AES_IV` | Cifrado AES en User-Service |
| `REDIS_PASSWORD` | Acceso a Redis |

### Cómo se manejan

- En `application.yml`, las contraseñas de BD **no tienen valor por defecto** (`${SPRING_DATASOURCE_PASSWORD:}`); `JWT_SECRET`/`AES_*` tienen **placeholders dev claramente falsos** solo para desarrollo/pruebas locales.
- En `Docker-compose.yml`, los secretos son **obligatorios desde `.env`** (`${VAR:?Define VAR en .env}`): el arranque aborta con un mensaje claro si falta alguno.
- Los valores reales viven **solo en el `.env` raíz**, que está en `.gitignore` y **no se versiona** en ningún repositorio.
- Los **tests** usan valores ficticios dedicados (no los de producción), seguros de publicar.

### Al publicar repositorios

1. **Rotar** todos los secretos expuestos previamente (contraseñas de BD/Redis, `JWT_SECRET`, `AES_SECRET`/`AES_IV`).
2. Confirmar que el `.env` no esté trackeado y que el historial esté limpio (se usó BFG Repo-Cleaner para purgar secretos del historial).
3. Mantener los valores nuevos únicamente en el `.env` local / en los *secrets* del entorno de despliegue.

## 4. Rate limiting (API Gateway)

El gateway aplica un **`RequestRateLimiter`** global (token-bucket por IP en Redis):

| Propiedad | Por defecto | Significado |
|---|:--:|---|
| `rate-limiter.replenish-rate` | 10 | Peticiones/segundo sostenidas por IP |
| `rate-limiter.burst-capacity` | 20 | Ráfaga máxima puntual por IP |

Al superar el límite, el gateway responde **429 Too Many Requests**.

## 5. CORS

CORS se configura **únicamente en el gateway** (origen único de todo el tráfico del frontend). Los microservicios **no** configuran CORS. Orígenes permitidos por defecto: `http://localhost`, `:3000`, `:5173`, `:8080`.

## 6. Transporte y red

- En Docker Compose los servicios viven en la red interna `smartlogix-net`; solo se exponen al host los puertos necesarios.
- La conexión a Oracle Autonomous DB usa **wallet (mTLS)**.
- En producción, servir el gateway y el frontend tras **HTTPS** (terminación TLS en un proxy/inbound).
