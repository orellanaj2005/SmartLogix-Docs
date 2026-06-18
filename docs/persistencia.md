# Persistencia de datos — SmartLogix

## 1. Resumen

La persistencia se implementa con **Spring Data JPA sobre Hibernate** (Jakarta Persistence), siguiendo el patrón **Repository**. La BD productiva es **Oracle Autonomous Database**; las pruebas usan **H2 en memoria**.

| Aspecto | Implementación |
|---|---|
| ORM | Hibernate (JPA) vía `spring-boot-starter-data-jpa` |
| Acceso a datos | Interfaces `JpaRepository<T, ID>` |
| BD productiva | Oracle Autonomous DB, conexión por **Wallet** (driver `ojdbc11`) |
| BD de pruebas | H2 en memoria, perfil `test` |
| Esquema | **Uno por servicio** (`SL_USUARIOS`, `SL_INVENTARIO`, `SL_PEDIDOS`, `SL_NOTIFICACIONES`) |
| Pool de conexiones | HikariCP |
| Gestión del esquema | `ddl-auto: validate` (User, Inventory) / `none` (Order, Notification) — el DDL lo crean scripts SQL externos |
| Consultas analíticas | JPQL con agregación (`COUNT`, `SUM`, `AVG`) — **sin** PL/SQL ni procedimientos |
| Caché | Redis (`@Cacheable` / `@CacheEvict`) en Inventory |
| Transacciones | `@Transactional` (declarativo) |

> **No se usan procedimientos almacenados, funciones de BD ni triggers.** Toda la lógica de consulta vive en JPQL, manteniendo portabilidad Oracle (prod) ↔ H2 (test).

## 2. Esquema por servicio

| Servicio | Esquema | `ddl-auto` | Entidades principales |
|---|---|:--:|---|
| User-Service | `SL_USUARIOS` | `validate` | `Usuario`, `Region`, `Comuna` |
| Inventory-Service | `SL_INVENTARIO` | `validate` | `Producto` |
| Order-Service | `SL_PEDIDOS` | `none` | `Pedido`, `Estado`, `EstadoPedidoActual` |
| Notification-Service | `SL_NOTIFICACIONES` | `none` | `Notificacion`, `Envio`, `Estado`, `EstadoEnvioActual` |

Cada servicio fija su esquema con `spring.jpa.properties.hibernate.default_schema`.

## 3. Conexión a Oracle (Wallet)

La conexión usa un **Oracle Wallet** (mTLS) en lugar de host/puerto:

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@yunjwge5ttypxb6i_medium?TNS_ADMIN=C:/Oracle/Wallet
    driver-class-name: oracle.jdbc.OracleDriver
```

- `TNS_ADMIN` apunta al directorio del wallet.
  - **Local (Windows):** `C:/Oracle/Wallet`
  - **Docker:** `/opt/oracle/wallet` (se monta desde `ORACLE_WALLET_PATH`).
- El alias `..._medium` corresponde a un servicio de la Autonomous DB.
- HikariCP usa `connection-test-query: SELECT 1 FROM DUAL`.

> Las **credenciales nunca están en el código**: usuario y contraseña se inyectan vía variables de entorno / `.env`. Ver [seguridad.md](seguridad.md).

## 4. Perfil de pruebas (H2)

El perfil `test` (`src/test/resources/application-test.yml`) sustituye Oracle por H2 en memoria:

```yaml
spring:
  datasource:
    url: "jdbc:h2:mem:...;INIT=CREATE SCHEMA IF NOT EXISTS SL_..."
    driver-class-name: org.h2.Driver
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect   # sobreescribe el OracleDialect base
    hibernate:
      ddl-auto: create-drop
```

- `DataSourceConfig` (y `KafkaConfig` en Inventory) están anotados `@Profile("!test")`, de modo que el perfil de test **no** intenta conectar a Oracle/Kafka.
- Es obligatorio forzar `H2Dialect` para que Hibernate no emita DDL de Oracle que H2 rechazaría.

## 5. Patrón Repository

```java
public interface ProductoRepository extends JpaRepository<Producto, Long> {

    @Query("SELECT new com.smartlogix.inventario.dto.StockProductoDTO(p.id, p.nombre, p.stock) " +
           "FROM Producto p")
    List<StockProductoDTO> findStockProductos();
}
```

- Los repositorios exponen consultas derivadas y **JPQL** con `@Query`.
- Las proyecciones devuelven directamente **DTOs** (`record`) mediante `SELECT new ...`, evitando transferir entidades completas. Ver [patrones-de-diseno.md](patrones-de-diseno.md).
