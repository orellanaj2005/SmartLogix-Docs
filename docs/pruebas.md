# Pruebas — SmartLogix

## 1. Estrategia

Pruebas **unitarias** con aislamiento de dependencias, siguiendo el patrón **AAA (Arrange–Act–Assert)**.

| Herramienta | Uso |
|---|---|
| **JUnit 5 (Jupiter)** | Framework de ejecución (`@Test`, `@DisplayName`, `@BeforeEach`) |
| **Mockito** | Mocks de repositorios, `KafkaTemplate`, Redis (`@Mock`, `@InjectMocks`, `verify`) |
| **AssertJ** | Aserciones fluidas (`assertThat`, `assertThatThrownBy`) |
| **Spring MockMvc** | Controladores REST en modo *standalone* (`standaloneSetup(...).setControllerAdvice(...)`) |
| **H2** | BD en memoria para los *smoke tests* de contexto (perfil `test`) |
| **Maven Surefire** | Ejecución y reportes |

Las dependencias externas (Oracle, Kafka, Redis) se **simulan o se desactivan** (perfil `test`) para que las pruebas sean **deterministas y rápidas**.

## 2. Cómo ejecutar

No hay Maven global instalado: usa el **wrapper** de cada servicio. En Windows, `mvnw` necesita `JAVA_HOME`:

```powershell
# PowerShell (Windows): fijar JAVA_HOME desde el java del PATH
$env:JAVA_HOME = Split-Path -Parent (Split-Path -Parent (Get-Command java).Source)
.\mvnw.cmd test
```

```bash
# Linux/Mac (dentro de cada servicio)
./mvnw test
```

El perfil `test` se activa automáticamente en los *smoke tests* de contexto (`*ApplicationTests` con `@ActiveProfiles("test")`), que usan H2 y evitan conectar a Oracle/Kafka.

## 3. Cobertura por servicio

> Resumen orientativo (las suites crecen con el tiempo). Todas en verde.

| Servicio | Pruebas | Áreas cubiertas |
|---|:--:|---|
| **User-Service** | ~56 | Servicio, controladores, JWT, AES, Strategy, blacklist Redis, DTOs |
| **Inventory-Service** | ~48 | Servicio, controlador, mapeo de DTOs |
| **Order-Service** | 31 | Servicio, controlador, facade, saga, observer |
| **Notification-Service** | ~29 | Servicio, facade, observers, controlador, DTOs |
| **Api-Gateway** | 13 | `JwtUtil`, `JwtAuthFilter`, `RateLimiterConfig`, contexto |

## 4. Notas por servicio

- **Order-Service** y **User/Inventory** tienen `GlobalExceptionHandler`, por lo que sus pruebas de controlador cubren también los caminos de error (404 / 401 / 409 / 503).
- **Notification-Service** aún **no** tiene `GlobalExceptionHandler`: sus pruebas de controlador cubren principalmente los caminos de éxito.
- Para que H2 acepte el DDL, el `application-test.yml` fuerza `database-platform: org.hibernate.dialect.H2Dialect`, sobreescribiendo el `OracleDialect` de la configuración base.

## 5. Reporte de cobertura (opcional)

Si el servicio tiene configurado JaCoCo:

```bash
./mvnw clean verify        # genera target/site/jacoco/
```
