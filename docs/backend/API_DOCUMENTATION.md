# 📡 Documentación de API - RetroApp Backend

## Descripción General

API REST construida con Laravel 11 para la gestión de dinámicas de retroalimentación en equipos de desarrollo. Implementa autenticación por tokens stateless con Laravel Sanctum y despacho de eventos en tiempo real con Soketi (compatible con el protocolo de Pusher).

---

## Estándares y Convenciones

- **URL Base**: `http://localhost:8000/api`
- **Headers Obligatorios** (rutas protegidas):
  ```http
  Authorization: Bearer {token}
  Content-Type: application/json
  Accept: application/json
  ```
- **Rate Limiting**: 60 solicitudes/minuto por IP en endpoints de autenticación pública.
- **Formato Estándar de Errores**:
  ```json
  {
    "message": "Descripción del error.",
    "errors": {
      "campo": ["Detalle de validación."]
    }
  }
  ```

---

## 🔐 Autenticación

### POST `/api/auth/login-admin`
Autentica administradores del sistema.

**Body (JSON):**
```json
{
  "email": "admin@admin.com",
  "password": "admin"
}
```

**Respuesta Exitosa (`200 OK`):**
```json
{
  "message": "Login successful",
  "user": {
    "id": 1,
    "name": "Administrador",
    "email": "admin@admin.com",
    "role": "admin"
  },
  "token": "1|laravel_sanctum_token...",
  "role": "admin"
}
```

### POST `/api/auth/login-guest`
Permite el acceso a miembros o evaluadores anónimos.

**Body (JSON):**
```json
{
  "name": "Invitado",
  "hash": "unique_guest_hash"
}
```

**Respuesta Exitosa (`200 OK`):**
```json
{
  "user": {
    "id": 2,
    "name": "Invitado",
    "email": null,
    "role": "guest",
    "guest_hash": "generated_hash"
  },
  "token": "2|laravel_sanctum_token..."
}
```

### POST `/api/auth/logout`
Revoca el token activo del usuario autenticado.

**Respuesta Exitosa (`200 OK`):**
```json
{
  "message": "Sesión cerrada correctamente"
}
```

### GET `/api/user`
Devuelve el perfil del usuario autenticado.

**Respuesta Exitosa (`200 OK`):**
```json
{
  "id": 1,
  "name": "Administrador",
  "email": "admin@admin.com",
  "role": "admin"
}
```

---

## 👥 Miembros del Equipo de Desarrollo (`/api/team-members`)

| Método | Endpoint | Permiso Requerido | Descripción |
|---|---|---|---|
| `GET` | `/api/team-members` | `admin` | Lista el directorio completo de integrantes. |
| `GET` | `/api/team-members-with-feedbacks` | Autenticado | Devuelve los miembros junto a sus notas asociadas. |
| `POST` | `/api/team-members` | `admin` | Registra un nuevo miembro del equipo. |
| `PUT` | `/api/team-members/{id}` | `admin` | Actualiza datos de un miembro existente. |
| `DELETE` | `/api/team-members/{id}` | `admin` | Elimina un miembro del equipo. |

### Crear / Actualizar Miembro

**Payload (`POST` / `PUT`):**
```json
{
  "name": "Ana López",
  "role": "Desarrollador Backend"
}
```

**Respuesta Exitosa (`201 Created` / `200 OK`):**
```json
{
  "id": 2,
  "name": "Ana López",
  "role": "Desarrollador Backend",
  "created_at": "2026-11-08T14:00:00.000000Z",
  "updated_at": "2026-11-08T14:00:00.000000Z"
}
```

---

## 💬 Feedback y Retrospectivas (`/api/feedbacks`)

### GET `/api/feedbacks`
Lista todas las notas de feedback con la información del receptor (`target`) y el autor (`owner`).

### POST `/api/feedbacks`
Crea una nueva tarjeta de feedback y despacha el evento WebSocket correspondiente.

**Body (JSON):**
```json
{
  "target_id": 1,
  "category": "achievements",
  "title": "Excelente entrega de sprint",
  "text": "Lideró la refactorización del módulo de colas con alta calidad."
}
```

**Reglas de Validación:**
- `target_id`: Requerido. Debe existir en la tabla `team_members`.
- `category`: Requerido. Valores admitidos:
  - `achievements`: Logros e hitos alcanzados.
  - `qualities`: Cualidades personales y habilidades blandas.
  - `potential`: Áreas de crecimiento y oportunidad de mejora.
- `title`: Requerido. Texto plano, máximo 50 caracteres.
- `text`: Requerido. Texto descriptivo, máximo 300 caracteres.

### PUT `/api/feedbacks/{id}`
Actualiza el contenido de una nota.

> 🔒 **Permisos:** Únicamente el autor de la tarjeta (`owner`) o un usuario con rol `admin` pueden editarla. De lo contrario, la API devuelve `403 Forbidden`.

### DELETE `/api/feedbacks/{id}`
Elimina la nota del tablero y emite el evento de remoción a los clientes conectados. Aplica la misma restricción de permisos (`owner` o `admin`).

---

## ⚡ Eventos en Tiempo Real (Soketi)

La API despacha eventos en canales públicos autenticados mediante Laravel Echo:

| Canal | Evento | Disparador | Contenido (`data`) |
|---|---|---|---|
| `feedback-updates` | `feedback.created` | `POST /api/feedbacks` | Entidad `feedback` completa con relaciones. |
| `feedback-updates` | `feedback.updated` | `PUT /api/feedbacks/{id}` | Entidad `feedback` actualizada. |
| `feedback-updates` | `feedback.deleted` | `DELETE /api/feedbacks/{id}` | Objeto con el `id` del feedback removido. |
| `team-updates` | `team-member.created` | `POST /api/team-members` | Entidad `team_member` creada. |
| `team-updates` | `team-member.updated` | `PUT /api/team-members/{id}` | Entidad `team_member` modificada. |
| `team-updates` | `team-member.deleted` | `DELETE /api/team-members/{id}` | Objeto con el `id` del miembro eliminado. |

**Esquema de Payload emitido:**
```json
{
  "event": "feedback.created",
  "channel": "feedback-updates",
  "data": {
    "feedback": {
      "id": 15,
      "target_id": 1,
      "owner_id": 2,
      "category": "achievements",
      "title": "Excelente entrega de sprint",
      "text": "Lideró la refactorización del módulo de colas con alta calidad.",
      "created_at": "2026-11-08T14:00:00.000000Z"
    }
  }
}
```

---

## 🧪 Pruebas Rápidas con cURL

```bash
# 1. Inicio de sesión como administrador
curl -s -X POST http://localhost:8000/api/auth/login-admin \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@admin.com","password":"admin"}'

# 2. Creación de nota de feedback (sustituir YOUR_TOKEN)
curl -X POST http://localhost:8000/api/feedbacks \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "target_id": 1,
    "category": "achievements",
    "title": "Gran despliegue",
    "text": "Completó la migración sin tiempo de inactividad."
  }'

# 3. Consulta de tablero con feedbacks asociados
curl -X GET http://localhost:8000/api/team-members-with-feedbacks \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Accept: application/json"
```
