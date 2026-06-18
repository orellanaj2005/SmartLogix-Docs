# Referencia de la API REST — SmartLogix

Todas las APIs se consumen a través del **API Gateway** (`http://localhost:8080`) con el prefijo **`/api`**. El gateway elimina ese prefijo (`StripPrefix=1`) antes de reenviar al microservicio.

## Autenticación

Salvo los endpoints públicos de `/api/auth`, **todas** las peticiones requieren la cabecera:

```
Authorization: Bearer <JWT>
```

El token se obtiene con `POST /api/auth/login`.

### Ejemplo de login

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<password>"}'
# → { "token": "eyJhbGciOiJIUzI1NiI..." }
```

### Ejemplo de llamada autenticada

```bash
curl http://localhost:8080/api/inventario/productos \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiI..."
```

---

## Usuarios y autenticación — `/api/auth`, `/api/usuarios`

| Método | Endpoint | Auth | Descripción |
|---|---|:--:|---|
| POST | `/api/auth/login` | — | Iniciar sesión |
| POST | `/api/auth/logout` | ✓ | Cerrar sesión (revoca token) |
| GET | `/api/auth/me` | ✓ | Usuario autenticado |
| POST | `/api/auth/refresh` | ✓ | Refrescar token |
| POST | `/api/auth/change-password` | ✓ | Cambiar contraseña |
| GET | `/api/usuarios` | ✓ | Listar usuarios |
| GET | `/api/usuarios/dto` | ✓ | Listar (sin contraseña) |
| GET | `/api/usuarios/{id}` | ✓ | Obtener usuario |
| GET | `/api/usuarios/por-rol` | ✓ | Métrica por rol |
| POST | `/api/usuarios` | ✓ | Crear |
| PUT | `/api/usuarios/{id}` | ✓ | Actualizar |
| DELETE | `/api/usuarios/{id}` | ✓ | Eliminar |

## Inventario — `/api/inventario`

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/api/inventario/productos` | Listar productos |
| GET | `/api/inventario/productos/{id}` | Obtener producto |
| POST | `/api/inventario/productos` | Crear producto |
| PUT | `/api/inventario/productos/{id}` | Actualizar producto |
| DELETE | `/api/inventario/productos/{id}` | Eliminar producto |
| GET | `/api/inventario/productos/stock-bajo` | Productos con stock bajo |
| GET | `/api/inventario/dashboard/stock-productos` | Stock por producto |
| GET | `/api/inventario/dashboard/stock-critico` | Stock crítico (con déficit) |
| GET | `/api/inventario/dashboard/ultimos-registrados` | Últimos productos |
| GET | `/api/inventario/metricas/total-productos` | Total de productos |
| GET | `/api/inventario/metricas/stock-bajo` | Cantidad con stock bajo |
| GET | `/api/inventario/metricas/valor-inventario` | Valor monetario |
| GET | `/api/inventario/metricas/stock-promedio` | Stock promedio |
| GET | `/api/inventario/alertas/stock-bajo` | Alertas accionables |
| POST | `/api/inventario/productos/{id}/reservar?cantidad=N` | Reservar stock (saga) |
| POST | `/api/inventario/productos/{id}/liberar?cantidad=N` | Liberar stock (compensación) |

## Pedidos — `/api/pedidos`

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/api/pedidos` | Listar pedidos |
| GET | `/api/pedidos/{id}` | Obtener pedido |
| GET | `/api/pedidos/{id}/estado` | Estado actual |
| POST | `/api/pedidos` | Crear pedido |
| PATCH | `/api/pedidos/{id}/estado?estadoId=N` | Cambiar estado |
| DELETE | `/api/pedidos/{id}` | Eliminar pedido |

## Notificaciones — `/api/notificaciones`

| Método | Endpoint | Descripción |
|---|---|---|
| POST | `/api/notificaciones/envio/{envioId}/estado?estadoId=&descripcionEstado=` | Notificar cambio de estado |
| GET | `/api/notificaciones` | Listar todas |
| GET | `/api/notificaciones/envio/{envioId}` | Listar por envío |

---

## Códigos de estado comunes

| Código | Significado |
|---|---|
| `200 OK` | Operación correcta |
| `201 Created` | Recurso creado (p. ej. pedido) |
| `204 No Content` | Eliminación correcta |
| `400 Bad Request` | Datos inválidos / stock insuficiente |
| `401 Unauthorized` | Credenciales inválidas o token ausente/expirado |
| `404 Not Found` | Recurso inexistente |
| `409 Conflict` | Recurso duplicado (User-Service) |
| `503 Service Unavailable` | Circuito abierto (Order-Service) |
| `429 Too Many Requests` | Rate limit superado (gateway) |

## Documentación interactiva (Swagger)

- **Agregada (gateway):** http://localhost:8080/swagger-ui.html
- **Por servicio:** `http://localhost:<puerto>/swagger-ui.html`

> Para probar peticiones con *Try it out* conviene usar el Swagger propio de cada servicio: el agregado del gateway no añade el prefijo `/api` automáticamente.
