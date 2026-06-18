# Configuración y despliegue — SmartLogix

## 1. Requisitos

| Para… | Necesitas |
|---|---|
| Docker Compose (todo) | Docker Desktop, Oracle Wallet en disco, archivo `.env` |
| Ejecución local backend | Java 21 (JDK), Oracle Wallet, variables de entorno |
| Frontend en desarrollo | Node.js 20+, npm |

No se requiere Maven instalado: cada servicio incluye el **Maven Wrapper** (`mvnw` / `mvnw.cmd`).

## 2. Variables de entorno (`.env` raíz)

El `Docker-compose.yml` lee un archivo **`.env`** ubicado en la raíz del monorepo (junto al compose). **No se versiona** (está en `.gitignore`).

> Plantilla — reemplaza los valores `***` por secretos reales. Los secretos son **obligatorios**: el compose aborta si faltan.

```env
# ── JWT (mismo secreto en todos los servicios) ──────────────
JWT_SECRET=***            # cadena larga y aleatoria (≥32 bytes)
JWT_EXPIRATION=86400000   # 24 h en ms

# ── AES (User-Service) ──────────────────────────────────────
AES_SECRET=***            # EXACTAMENTE 32 bytes (AES-256)
AES_IV=***                # EXACTAMENTE 16 bytes

# ── Redis ───────────────────────────────────────────────────
REDIS_PASSWORD=***

# ── Oracle Autonomous Database ──────────────────────────────
DATASOURCE_URL=jdbc:oracle:thin:@yunjwge5ttypxb6i_medium?TNS_ADMIN=/opt/oracle/wallet

US_DATASOURCE_USERNAME=sl_usuarios
US_DATASOURCE_PASSWORD=***

INV_DATASOURCE_USERNAME=sl_inventario
INV_DATASOURCE_PASSWORD=***

NOT_DATASOURCE_USERNAME=sl_notificaciones
NOT_DATASOURCE_PASSWORD=***

PED_DATASOURCE_USERNAME=sl_pedidos
PED_DATASOURCE_PASSWORD=***

# Ruta LOCAL al Oracle Wallet (se monta en /opt/oracle/wallet dentro del contenedor)
#   Windows:   C:/Oracle/Wallet
#   Mac/Linux: /home/tuusuario/oracle/wallet
ORACLE_WALLET_PATH=C:/Oracle/Wallet

# ── Frontend (build-time, todo va por el gateway) ───────────
VITE_API_GATEWAY=http://localhost:8080
```

## 3. Oracle Wallet

1. Descarga el wallet de la Autonomous DB desde Oracle Cloud.
2. Descomprímelo en la ruta indicada por `ORACLE_WALLET_PATH` (p. ej. `C:/Oracle/Wallet`).
3. En Docker, esa carpeta se monta como `/opt/oracle/wallet` (de solo lectura) y `TNS_ADMIN` apunta allí.

## 4. Despliegue con Docker Compose

| Servicio (compose) | Carpeta | Puerto |
|---|---|:--:|
| `api-gateway` | `./Api-Gateway` | 8080 |
| `ms-usuario` | `./User-Service` | 8084 |
| `ms-inventario` | `./Inventory-Service` | 8081 |
| `ms-pedidos` | `./Order-Service` | 8082 |
| `ms-notificaciones` | `./Notification-Service` | 8085 |
| `frontend_ms` | `./frontend_ms` | 80 |
| `redis` | infraestructura | 6379 |
| `kafka` / `zookeeper` | infraestructura | 9092 / 2181 |

```bash
# Levantar toda la plataforma (build + arranque)
docker compose up --build

# En segundo plano
docker compose up -d

# Un servicio concreto (arrastra sus dependencias)
docker compose up ms-inventario

# Ver estado y logs
docker compose ps
docker compose logs -f ms-pedidos

# Detener y eliminar
docker compose down
```

Redis y Kafka tienen *healthchecks*; los microservicios esperan a que estén `healthy` antes de arrancar.

## 5. Ejecución local (por servicio)

```bash
# Backend (dentro de cada carpeta de servicio)
./mvnw spring-boot:run        # Linux/Mac
.\mvnw.cmd spring-boot:run    # Windows

# Frontend
cd frontend_ms
npm install
npm run dev        # http://localhost:5173
```

En ejecución local, exporta las credenciales necesarias antes de arrancar, por ejemplo en PowerShell:

```powershell
$env:SPRING_DATASOURCE_PASSWORD = "***"
.\mvnw.cmd spring-boot:run
```

## 6. Perfiles de Spring

| Perfil | Uso |
|---|---|
| *(default)* | Oracle + Redis/Kafka (producción / local real) |
| `test` | H2 en memoria; desactiva Oracle y Kafka (`@Profile("!test")`) |

## 7. Health checks y observabilidad

- Cada servicio expone `GET /actuator/health`.
- El gateway expone además endpoints de Actuator y el Swagger UI agregado en `/swagger-ui.html`.
