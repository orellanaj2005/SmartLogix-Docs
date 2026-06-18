# SmartLogix

![Java](https://img.shields.io/badge/Java-21-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.x-brightgreen)
![Spring Cloud Gateway](https://img.shields.io/badge/Spring%20Cloud%20Gateway-2024.0.1-blue)
![React](https://img.shields.io/badge/React-18-61DAFB?logo=react)
![Oracle](https://img.shields.io/badge/Oracle-Autonomous%20DB-F80000?logo=oracle)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)

Repositorio de **documentación central** de la plataforma **SmartLogix**, un sistema de gestión logística construido con una arquitectura de **microservicios**.

---

## 📖 Descripción del proyecto

**SmartLogix** es una plataforma de logística y gestión de inventario diseñada como un conjunto de **microservicios independientes** que se comunican a través de un **API Gateway**. Cada microservicio es responsable de un dominio de negocio acotado (usuarios, inventario, pedidos, notificaciones), posee su **propio esquema de base de datos** (patrón *schema-per-service*) y se despliega de forma autónoma.

El sistema cubre el ciclo logístico básico: autenticación y gestión de usuarios, administración de productos y stock, registro y seguimiento de pedidos, y generación de notificaciones ante los cambios de estado de los envíos. Un **frontend SPA** en React consume todo el backend a través del gateway.

## 🎯 Objetivo

Proveer una plataforma logística **modular, escalable y desacoplada** que permita:

- **Independencia de despliegue y evolución**: cada servicio se construye, prueba y despliega por separado.
- **Aislamiento de datos**: un esquema de base de datos por servicio, sin acoplamiento entre dominios.
- **Punto de entrada único y seguro**: autenticación JWT centralizada, CORS y rate limiting en el gateway.
- **Resiliencia**: tolerancia a fallos mediante *circuit breaker* y operaciones de compensación (*saga*).
- **Demostrar buenas prácticas**: patrones de diseño (Saga, Facade, Observer, Strategy), pruebas unitarias y gestión de secretos fuera del código.

> Proyecto académico desarrollado para la asignatura **Desarrollo Fullstack III** (Duoc UC).

## 🛠️ Tecnologías usadas

### Backend
- **Java 21** · **Spring Boot 3.4.x**
- **Spring Cloud Gateway 2024.0.1** (WebFlux reactivo) — API Gateway
- **Spring Data JPA / Hibernate** — persistencia (patrón Repository)
- **Spring Security** + **JWT (JJWT 0.12)** — autenticación stateless
- **Resilience4j** — circuit breaker (Order-Service)
- **Apache Kafka** — eventos de dominio (alertas de stock)
- **OpenAPI / Swagger UI** (springdoc) — documentación de APIs

### Datos e infraestructura
- **Oracle Autonomous Database** (Oracle Cloud) — BD productiva, conexión por *Wallet*
- **H2** (en memoria) — BD para pruebas (perfil `test`)
- **Redis** — rate limiting (gateway), caché (inventario) y blacklist de tokens (usuarios)
- **Docker** + **Docker Compose** — orquestación local

### Frontend
- **React 18** · **TypeScript** · **Vite**
- **Redux Toolkit** — estado global · **axios** — HTTP · **react-router-dom** — enrutado
- **Nginx** — sirve el build en producción

### Calidad
- **JUnit 5** · **Mockito** · **AssertJ** · **Spring MockMvc** — pruebas unitarias

## 🏗️ Arquitectura general

```
                            ┌──────────────────────┐
                            │   Frontend (React)    │
                            │   Nginx :80 / Vite    │
                            └───────────┬───────────┘
                                        │  HTTP (todo el tráfico)
                                        ▼
                            ┌──────────────────────┐
                            │     API Gateway       │  :8080
                            │  Spring Cloud Gateway │
                            │  · Routing            │
                            │  · JWT centralizado   │
                            │  · CORS · Rate Limit   │
                            └───────────┬───────────┘
              ┌──────────────┬──────────┼───────────┬──────────────────┐
              ▼              ▼          ▼           ▼                  ▼
       ┌────────────┐ ┌────────────┐ ┌─────────┐ ┌──────────────┐  (planificado)
       │   User     │ │ Inventory  │ │  Order  │ │ Notification │   /api/envios
       │  :8084     │ │  :8081     │ │ :8082   │ │   :8085      │     :8083
       │SL_USUARIOS │ │SL_INVENTAR.│ │SL_PEDID.│ │SL_NOTIFICAC. │
       └─────┬──────┘ └──────┬─────┘ └────┬────┘ └──────┬───────┘
             │               │            │             │
             ▼               ▼            ▼             ▼
       ┌──────────────────────────────────────────────────────┐
       │     Oracle Autonomous Database (esquema por servicio) │
       └──────────────────────────────────────────────────────┘

   Infraestructura compartida:  Redis (rate-limit / caché / blacklist)
                                Kafka + Zookeeper (eventos de stock)
```

- **Todo el tráfico del cliente entra por el gateway**, que valida el JWT y enruta a cada microservicio (`StripPrefix=1`).
- **Schema-per-service**: cada servicio es dueño de su esquema Oracle y no accede a los de otros.
- La comunicación entre servicios para el flujo de pedidos usa una **saga** con compensación (reservar/liberar stock).

Detalle completo en **[docs/arquitectura.md](docs/arquitectura.md)**.

## 🧩 Lista de microservicios

| Servicio | Puerto | Esquema BD | Responsabilidad | Infra extra |
|---|:--:|---|---|---|
| **Api-Gateway** | 8080 | — | Enrutamiento, JWT, CORS, rate limiting, Swagger agregado | Redis |
| **User-Service** | 8084 | `SL_USUARIOS` | Autenticación (JWT), CRUD de usuarios, cifrado de contraseñas | Redis |
| **Inventory-Service** | 8081 | `SL_INVENTARIO` | Productos, stock, métricas, alertas, reserva/liberación (saga) | Redis, Kafka |
| **Order-Service** | 8082 | `SL_PEDIDOS` | Pedidos y su estado, saga, circuit breaker | — |
| **Notification-Service** | 8085 | `SL_NOTIFICACIONES` | Notificaciones por cambios de estado de envíos (Observer) | — |
| **frontend_ms** | 80 / 5173 | — | SPA React que consume el backend vía gateway | — |

> La ruta `/api/envios` (puerto 8083) está **reservada** en el gateway pero el servicio de envíos no está desplegado como módulo independiente en esta fase.

Detalle de endpoints y patrones por servicio en **[docs/microservicios.md](docs/microservicios.md)**.

## 🚀 Cómo levantar el proyecto

### Opción A — Docker Compose (recomendada)

Levanta toda la plataforma (gateway + microservicios + Redis + Kafka + frontend) con un solo comando.

**Requisitos:** Docker Desktop, el **Oracle Wallet** en disco y un archivo **`.env`** en la raíz.

```bash
# 1. Crear el .env en la raíz (junto a Docker-compose.yml) con las credenciales.
#    Ver docs/configuracion-y-despliegue.md para la plantilla completa.

# 2. Levantar todo
docker compose up --build

# Solo un servicio (con sus dependencias)
docker compose up ms-inventario
```

| Componente | URL |
|---|---|
| Frontend | http://localhost |
| API Gateway | http://localhost:8080 |
| Swagger UI agregado | http://localhost:8080/swagger-ui.html |

> El `Docker-compose.yml` **exige** que el `.env` defina los secretos (contraseñas de BD, `JWT_SECRET`, `AES_SECRET`, `AES_IV`, `REDIS_PASSWORD`): aborta con un mensaje claro si falta alguno.

### Opción B — Ejecución local por servicio

Cada microservicio se ejecuta con su Maven Wrapper. Requiere **Java 21** y las variables de entorno con credenciales.

```bash
# Dentro de la carpeta de cada servicio
./mvnw spring-boot:run        # Linux/Mac
.\mvnw.cmd spring-boot:run    # Windows
```

Guía detallada (variables, wallet, perfiles) en **[docs/configuracion-y-despliegue.md](docs/configuracion-y-despliegue.md)**.

## 🔗 Enlaces a cada repositorio

| Componente | Repositorio |
|---|---|
| Api-Gateway | https://github.com/orellanaj2005/Api-Gateway |
| User-Service | https://github.com/orellanaj2005/User-Service |
| Inventory-Service | https://github.com/orellanaj2005/Inventory-Service |
| Order-Service | https://github.com/orellanaj2005/Order-Service |
| Notification-Service | https://github.com/orellanaj2005/Inventory-Service
| frontend_ms | https://github.com/prograshaco/frontend_ms |
| SmartLogix-Docs (este repo) | https://github.com/orellanaj2005/SmartLogix-Docs |

## 📚 Documentación

| Documento | Contenido |
|---|---|
| [docs/arquitectura.md](docs/arquitectura.md) | Arquitectura, flujo de peticiones, JWT, saga, infraestructura |
| [docs/microservicios.md](docs/microservicios.md) | Detalle de cada servicio: endpoints, esquema, patrones |
| [docs/api-rest.md](docs/api-rest.md) | Referencia de la API REST (rutas vía gateway) |
| [docs/persistencia.md](docs/persistencia.md) | Modelo de datos, esquemas, Oracle/H2, patrón Repository |
| [docs/seguridad.md](docs/seguridad.md) | JWT, AES, gestión de secretos, rate limiting |
| [docs/patrones-de-diseno.md](docs/patrones-de-diseno.md) | Saga, Facade, Observer, Strategy, Circuit Breaker, DTO |
| [docs/configuracion-y-despliegue.md](docs/configuracion-y-despliegue.md) | Variables de entorno, wallet, Docker Compose, perfiles |
| [docs/pruebas.md](docs/pruebas.md) | Estrategia de pruebas, ejecución y cobertura |

---

_Última actualización: junio de 2026._
