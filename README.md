# wl_server

**Integración Google Calendar + Recordatorios (CRUD)**

Esta guía explica cómo autenticar Google Calendar desde el frontend y cómo usar las rutas de Recordatorios. Usa la palabra RUTA_BASE en lugar de localhost o dominios reales.

**Google Calendar: Autenticación**

**Actividades (API)**

Base URL: https://wlserver-production.up.railway.app/

Ruta base: `/api/actividades`

- Formato general de respuestas:
	- Éxito (200/201): `{ "success": true, "data": ... }` o `{ "success": true, "items": [...] }`
	- Error servidor (500): `{ "success": false, "error": "<mensaje>" }`
	- No encontrado (404): `{ "success": false, "error": "<mensaje>" }` (cuando aplica)

- Headers comunes:
	- `Content-Type`: `application/json` para JSON; `multipart/form-data` para uploads.
	- `x-device-id`: RECOMENDADO / requerido en endpoints de creación/edición/eliminación. Se usa para asociar la acción a un usuario.
	- Rate-limit: rutas de edición usan un limitador (25 requests/minuto por `x-device-id`).

Endpoints principales (resumen y ejemplos curl)

1) Listar todas las actividades
- Método: GET
- Path: `/api/actividades`
- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/actividades"
```
- Respuesta ejemplo:
```json
{ "success": true, "data": [ /* actividades */ ] }
```

2) Buscar actividades por texto
- Método: GET
- Path: `/api/actividades/buscar`
- Query: `q` (texto de búsqueda)
- Curl:
```bash
curl -G "https://wlserver-production.up.railway.app/api/actividades/buscar" --data-urlencode "q=texto a buscar"
```

3) Obtener actividades por rango de fechas
- Método: GET
- Path: `/api/actividades/fechas`
- Query: `fechaInicio` (ISO), `fechaFin` (opcional)
- Curl:
```bash
curl -G "https://wlserver-production.up.railway.app/api/actividades/fechas" --data-urlencode "fechaInicio=2026-01-01" --data-urlencode "fechaFin=2026-01-07"
```

4) Obtener actividad por ID
- Método: GET
- Path: `/api/actividades/:id` (notionId con o sin guiones)
- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/actividades/1234abcd-..."
```
- Respuesta éxito:
```json
{ "success": true, "data": { "id":"...","titulo":"...","status":"...","pendientes":[...] } }
```

5) Crear actividad (con archivos y pendientes)
- Método: POST
- Path: `/api/actividades`
- Headers: `x-device-id: TU_DEVICE_ID`; Content-Type: `multipart/form-data`
- Form fields:
	- `titulo`, `status`, `prioridad`, `tipo`, `proyectoId`, `anotaciones`, `pasosYLinks`, `dueStart`, `dueEnd`, etc.
	- `pendientes`: puede ser un array o un string JSON. Ej: `'[{"text":"Subtarea 1"}]'`
	- Archivos: `archivos` (multiple), `pendienteImages` (imagenes para pendientes)
- Curl ejemplo:
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/actividades" \
	-H "x-device-id: TU_DEVICE_ID" \
	-F "titulo=Mi tarea nueva" \
	-F "pendientes=[{\"text\":\"Subtarea 1\"}]" \
	-F "archivos=@/ruta/archivo.pdf" \
	-F "pendienteImages=@/ruta/imagen.jpg"
```
- Respuesta éxito (201):
```json
{ "success": true, "data": { /* actividad creada (formateada) */ } }
```

6) Crear con `tarjet`
- POST `/api/actividades/crear-con-tarjet` (igual a crear, acepta `tarjet` y `webname` en el body)

7) Crear actividad vacía
- POST `/api/actividades/crear-vacia`
- Body JSON: `{ "titulo":"...", "copyFromId":"opcional" }`

8) Actualizar actividad
- PUT `/api/actividades/:id`
- Headers: `x-device-id` requerido para ediciones; Content-Type: `application/json` o `multipart/form-data` si hay archivos
- Curl JSON:
```bash
curl -X PUT "https://wlserver-production.up.railway.app/api/actividades/1234..." \
	-H "Content-Type: application/json" -H "x-device-id: TU_DEVICE_ID" \
	-d '{"titulo":"Titulo actualizado","status":"En progreso"}'
```

9) Eliminar actividad
- DELETE `/api/actividades/:id` (Header `x-device-id` requerido)
- Curl:
```bash
curl -X DELETE "https://wlserver-production.up.railway.app/api/actividades/1234..." -H "x-device-id: TU_DEVICE_ID"
```

10) Operaciones batch y utilidades (resumen)
- Reprogramar atrasados: POST `/api/actividades/reprogramar-atrasados` (rate-limited)
- Mover FTF a mañana: POST `/api/actividades/mover-ftf-manana`
- Asignar horarios largas hoy: POST `/api/actividades/asignar-horarios-largos-hoy`
- Mover fechas varias: POST `/api/actividades/mover-fechas` (body: `{ ids: [...], horas: N }`)
- Actualizar propiedades varias: POST `/api/actividades/actualizar-propiedades` (body: `{ ids: [...], propiedades: {...} }`)

11) Pendientes (sub-recursos)
- Crear pendiente: POST `/api/actividades/:notionId/pendientes` (form-data o JSON; `images` como archivos)
- Reordenar: PATCH `/api/actividades/:notionId/pendientes/reordenar` (body: `{ order: [...] }`)
- Actualizar pendiente: PATCH `/api/actividades/:notionId/pendientes/:blockId` (puede incluir `images`)
- Actualizar bloque: PATCH `/api/actividades/:notionId/pendientes/:blockId/bloque` (body: `{ bloque: N }`)
- Eliminar pendiente: DELETE `/api/actividades/:notionId/pendientes/:blockId`

12) Consultas por assignee / proyecto / ftf
- GET `/api/actividades/assignee/:assignee/del-dia`
- GET `/api/actividades/assignee/:assignee`
- GET `/api/actividades/proyecto/:projectId`
- GET `/api/actividades/ftf/:assignee/status`

13) Destacado
- PUT `/api/actividades/:id/destacado` (body ejemplo: `{ "destacado": true, "destacadoColor": "#ff0" }`)
- DELETE `/api/actividades/:id/destacado`

14) Previews Notion
- GET `/api/actividades/:id/preview`
- POST `/api/actividades/:id/preview` (forzar refresh, rate-limited)

Errores y notas importantes:
- Límite de archivos: 10MB por archivo (multer configurado)
- Protección contra duplicados: crea evita duplicados por título en últimos 5 minutos
- Concurrencia: actualizaciones a una misma actividad pasan por una cola interna para evitar race conditions

Si quieres, puedo convertir esta sección en una página separada `docs/actividades.md` o generar OpenAPI.

**Auth (API)**

Ruta base: `/api/auth`

- Formato de respuesta (usado por los controladores de auth):
	- Éxito: `{ "ok": true, ... }` (puede incluir `token` y `user` según endpoint)
	- Error: `{ "error": "<mensaje>" }`
	- Caso: dispositivo no registrado: respuesta 404 `{ "ok": false, "needsRegister": true }`

- Notes:
	- Muchos endpoints aceptan `x-device-id` en headers o `deviceId` en body/query.
	- Para endpoints que requieren autenticación, usar header `Authorization: Bearer <token>`.

Endpoints de `auth`:

1) Registrar / crear cuenta
- Método: POST
- Path: `/api/auth/register`
- Content-Type: `multipart/form-data` (opcional `avatar` file)
- Headers: opcional `x-device-id` (si no, puede venir en body como `deviceId`)
- Campos (form-data): campos de perfil (`firstName`, `lastName`, `email`, `phone`, `password`, `deviceId`... según cliente)
- Curl ejemplo:
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/auth/register" \
	-F "firstName=Juan" -F "email=juan@example.com" -F "password=secret" \
	-F "deviceId=MI_DEVICE_ID" -F "avatar=@/ruta/avatar.jpg"
```
- Respuesta éxito (200):
```json
{ "ok": true, "token": "<jwt>", "user": { /* objeto usuario */ } }
```
- Error (400): `{ "error": "<mensaje>" }`

2) Obtener perfil (me)
- Método: GET
- Path: `/api/auth/me`
- Headers: `Authorization: Bearer <token>` (requerido)
- Curl:
```bash
curl -H "Authorization: Bearer <token>" "https://wlserver-production.up.railway.app/api/auth/me"
```
- Respuesta éxito (200):
```json
{ "ok": true, "user": { /* user */, "token": "..."? }, ... }
```
- Errores:
	- 401 si falta/invalid token: `{ "error": "token requerido" }` o `{ "error": "token inválido" }`

3) Check device (login by device) - POST
- Método: POST
- Path: `/api/auth/check-device`
- Body JSON: `{ "deviceId": "...", "email": "opcional" }` (o enviar `x-device-id` header)
- Curl ejemplo:
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/auth/check-device" \
	-H "Content-Type: application/json" \
	-d '{"deviceId":"MI_DEVICE_ID"}'
```
- Respuestas:
	- Si existe sesión: `{ "ok": true, "token": "<jwt>", "user": { ... } }`
	- Si NO existe (necesita registro): HTTP 404 `{ "ok": false, "needsRegister": true }`
	- Error 400: `{ "error": "<mensaje>" }`

4) Check device (login by device) - GET
- Método: GET
- Path: `/api/auth/check-device?deviceId=...&email=...`
- Header alterno: `x-device-id`
- Uso: equivalente a POST pero via query
- Curl ejemplo:
```bash
curl "https://wlserver-production.up.railway.app/api/auth/check-device?deviceId=MI_DEVICE_ID"
```

5) Actualizar perfil
- Método: PATCH
- Path: `/api/auth/me`
- Headers: `Authorization: Bearer <token>` (requerido)
- Content-Type: `multipart/form-data` si se envía `avatar` (campo `avatar`), o `application/json` para solo campos
- Body ejemplo JSON: `{ "firstName":"Nuevo", "phone":"+521..." }`
- Curl multipart ejemplo (avatar):
```bash
curl -X PATCH "https://wlserver-production.up.railway.app/api/auth/me" \
	-H "Authorization: Bearer <token>" \
	-F "firstName=Nuevo" -F "avatar=@/ruta/avatar.jpg"
```
- Respuesta éxito:
```json
{ "ok": true, "user": { /* usuario actualizado */ } }
```
- Errores:
	- 401 si token faltante/inválido: `{ "error": "token requerido" }` o `{ "error": "token inválido" }`
	- 400 en caso de validación: `{ "error": "mensaje" }`

6) Refresh token
- Método: POST
- Path: `/api/auth/refresh`
- Headers: `Authorization: Bearer <token>` (token de refresh/antiguo)
- Curl:
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/auth/refresh" -H "Authorization: Bearer <token>"
```
- Respuesta éxito:
```json
{ "ok": true, "token": "<nuevo_token>" }
```
- Errores: 401/400 con `{ "error": "mensaje" }`

Resumen de códigos y formatos relevantes para `auth`:
- 200: éxito general (`{ ok: true, ... }`).
- 400: error de request/validación (`{ error: "..." }`).
- 401: auth requerida o inválida (`{ error: "token requerido" }` / `{ error: "token inválido" }`).
- 404: dispositivo no registrado (`{ ok: false, needsRegister: true }`).

Archivos relevantes:
- Rutas: `src/routes/authRoutes.js`
- Controlador: `src/controllers/authController.js`
- Servicio/logic: `src/services/authService.js`

**Shared Auth (Login Compartido)**

Esta API proporciona un login compartido para permitir que el frontend inicie una sesión global (un único usuario compartido) sin interferir con el login por `deviceId`. Se monta en la ruta base `/api/shared-auth`.

- Nota de seguridad: las credenciales se configuran en el archivo `.env` con `SHARED_USER` y `SHARED_PASS`. Para mayor seguridad, define `JWT_SHARED_SECRET` en `.env`.

Endpoints y formato JSON de respuesta

1) Login compartido
- Método: `POST`
- Path: `/api/shared-auth/login`
- Body JSON de ejemplo:
```json
{ "user": "wlteam@anfeta.com", "pass": "Anfeta2026" }
```
- Respuesta éxito (200) — JSON:
```json
{
	"token": "<jwt_access_token>",
	"refreshToken": "<jwt_refresh_token>"
}
```
- Errores (ejemplos):
```json
{ "message": "Missing credentials" }            // 400
{ "message": "Invalid credentials" }            // 401
```

2) Acceder a rutas protegidas (ejemplo)
- Requisito: enviar el header `x-shared-token` con el `token` devuelto por `/login`.
- Ejemplo de header:
```
x-shared-token: <jwt_access_token>
```
- Respuesta éxito típica (depende de la ruta):
```json
{
	"success": true,
	"data": { /* contenido según endpoint */ },
	"shared": { "shared": true, "iat": 1670000000, "exp": 1670003600 }
}
```
- Errores:
```json
{ "message": "Missing shared token" }            // 401
{ "message": "Invalid or expired shared token" } // 401
```

3) Refresh token
- Método: `POST`
- Path: `/api/shared-auth/refresh`
- Enviar el refresh token en el header `x-shared-refresh`:
```
x-shared-refresh: <jwt_refresh_token>
```
- Respuesta éxito (200) — JSON:
```json
{
	"token": "<new_jwt_access_token>",
	"refreshToken": "<new_jwt_refresh_token>"
}
```
- Errores (ejemplos):
```json
{ "message": "Missing refresh token header" }   // 400
{ "message": "Invalid or expired refresh token" } // 401
```

Uso en rutas existentes
- El middleware `validateSharedToken` valida el header `x-shared-token` y añade `req.shared` cuando es válido. Ejemplo de uso en rutas: `router.post("/", validateSharedToken, handler)`.

Pruebas rápidas (curl)
```bash
# Login
curl -X POST http://localhost:4000/api/shared-auth/login \
	-H "Content-Type: application/json" \
	-d '{"user":"wlteam@anfeta.com","pass":"Anfeta2026"}'

# Usar token en ruta protegida
curl http://localhost:4000/api/actividades \
	-H "x-shared-token: <token_aqui>"

# Refresh
curl -X POST http://localhost:4000/api/shared-auth/refresh \
	-H "x-shared-refresh: <refresh_token_aqui>"
```

**Presence (online users)**

Ruta base: `/api/presence`

- Endpoint REST principal:
	- Método: GET
	- Path: `/api/presence/online`
	- Qué envías: ninguno (opcional `x-device-id` no requerido)
	- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/presence/online"
```
	- Respuesta (JSON):
```json
{ "items": [ { "name": "Juan Perez", "email": "juan@example.com", "phone": "+521...", "avatar": "https://.../avatar.jpg" }, ... ] }
```
	- Nota: el servicio elimina entradas con `lastPingAt` mayor a 10 minutos, por lo que "online" significa actividad en los últimos 10 minutos.

- Eventos en tiempo real (Socket.IO):
	- Conexión: cliente debe conectar con `auth` en el handshake: `{ token, deviceId, meta? }`.
	- Evento emitido por el servidor: `presence:online` con payload `{ online: [ { name, email, phone, avatar }, ... ] }` cada vez que cambia la lista.
	- Ejemplo de conexión usando `socket.io-client`:
```javascript
import { io } from "socket.io-client";
const socket = io("https://wlserver-production.up.railway.app", {
	auth: { token: "<JWT>", deviceId: "MI_DEVICE_ID", meta: { platform: "web" } }
});
socket.on("presence:online", (data) => console.log("Online:", data.online));
```

- Fuente de datos y formato:
	- `src/services/presenceService.js` devuelve una lista simplificada con campos `name`, `email`, `phone`, `avatar`.
	- El servidor mantiene colecciones `Presence` en Mongo y actualiza en `connection` / `disconnect` y `presence:ping`.


Frontend: flujo recomendado
- Paso 1: Solicita la URL de auth al backend con el teléfono del usuario (10 dígitos preferentemente).
	- GET RUTA_BASE/google/auth?userId=+5217712045261
- Paso 2: Redirige al usuario a `authUrl` (popup o nueva pestaña).
- Paso 3: Tras otorgar permisos, el callback muestra confirmación y se guardan tokens.
- Paso 4: Reintenta la acción que requería Calendar (p. ej., crear recordatorio) en caso de que previamente recibieras `authRequired=true`.

Cerrar sesión (revocar tokens)
- Endpoint para cerrar sesión y eliminar tokens del usuario (útil para pruebas con otra cuenta):
	- Método: DELETE
	- Ruta: RUTA_BASE/google/logout?userId=<telefono>
	- Respuesta (200):
		{
			"success": true,
			"message": "Tokens revocados y eliminados"
		}
- Ejemplo curl:
```bash
curl -X DELETE "RUTA_BASE/google/logout?userId=+5217712045261"
```

Manejo de authRequired en Recordatorios
- Cuando intentes crear/actualizar un recordatorio, si no hay tokens de Google para ese `userId`, la respuesta incluirá:
	{
		"recordatorio": { ... },
		"google": {
			"authRequired": true,
			"authUrl": "..."
		}
	}
- En el frontend, redirige a `authUrl`. Luego vuelve a intentar la operación.

**Recordatorios: Modelo (resumen)**
- userId: string (requerido)
- mensaje: string (requerido) – título del evento en Calendar
- fechaHora: Date (requerido) – ISO 8601 recomendado
- tipo: "unica_vez" | "diaria" | "semanal" | "mensual" (por ahora informativo)
- activo: boolean (default: true)
- enviado: boolean (default: false)
- revisionId: string (notionId de la Revisión) recomendado
- googleEventId: string (si se sincronizó con Calendar)
- googleHtmlLink: string (link al evento en Calendar)
- duracionMinutos: number (default: 30)
- timezone: string (default: America/Mexico_City)

Notas de formato de fechas
- Usa ISO 8601 con zona horaria cuando sea posible (ej: "2025-12-26T10:30:00-06:00").
- Si envías sin offset, el backend convertirá a ISO según tu fecha local del servidor.

**Endpoints Recordatorios (centrados en Revisión)**

Crear recordatorio para una Revisión (recomendado)
- POST RUTA_BASE/api/recordatorios/revision/:revisionId
- Body mínimo: userId, mensaje, fechaHora
- Body opcional: duracionMinutos, timezone, tipo
- Ejemplo curl:
```bash
curl -X POST RUTA_BASE/api/recordatorios/revision/rv123 \
	-H "Content-Type: application/json" \
	-d '{
		"userId": "+5217712045261",
		"mensaje": "Revisar entregable",
		"fechaHora": "2025-12-26T15:00:00-06:00",
		"duracionMinutos": 30,
		"timezone": "America/Mexico_City"
	}'
```
- Respuesta (200):
	- Con tokens válidos:
		{
			"recordatorio": {
				"_id": "677000000000000000000001",
				"userId": "7712045261",
				"mensaje": "Revisar entregable",
				"fechaHora": "2025-12-26T21:00:00.000Z",
				"tipo": "unica_vez",
				"activo": true,
				"enviado": false,
				"revisionId": "rv123",
				"googleEventId": "abcd1234eventid",
				"googleHtmlLink": "https://www.google.com/calendar/event?eid=...",
				"duracionMinutos": 30,
				"timezone": "America/Mexico_City",
				"createdAt": "2025-12-24T06:00:00.000Z",
				"updatedAt": "2025-12-24T06:00:00.000Z"
			},
			"google": {
				"authRequired": false,
				"eventId": "abcd1234eventid",
				"link": "https://www.google.com/calendar/event?eid=..."
			}
		}
	- Sin tokens (requiere autorización):
		{
			"recordatorio": { ...sin googleEventId... },
			"google": {
				"authRequired": true,
				"authUrl": "https://accounts.google.com/o/oauth2/v2/auth?..."
			}
		}

Nota: existe un endpoint por actividad solo para compatibilidad antigua; se recomienda usar siempre el de revisión.

Listar todos
- GET RUTA_BASE/api/recordatorios
```bash
curl RUTA_BASE/api/recordatorios
```

Listar por id
- GET RUTA_BASE/api/recordatorios/:id
```bash
curl RUTA_BASE/api/recordatorios/677000000000000000000001
```

Listar por usuario
- GET RUTA_BASE/api/recordatorios/usuario/:userId
```bash
curl RUTA_BASE/api/recordatorios/usuario/7712045261
```

Listar por revisión
- GET RUTA_BASE/api/recordatorios/revision/:revisionId
```bash
curl RUTA_BASE/api/recordatorios/revision/rv123
```

Recordatorios por Revisión (recomendado)
- Crear para una revisión:
	- POST RUTA_BASE/api/recordatorios/revision/:revisionId
	- Body igual al de creación general (sin revisionId, va en la URL)
```bash
curl -X POST RUTA_BASE/api/recordatorios/revision/rv123 \
	-H "Content-Type: application/json" \
	-d '{
		"userId": "+5217712045261",
		"mensaje": "Revisar entregable",
		"fechaHora": "2025-12-26T15:00:00-06:00",
		"duracionMinutos": 30
	}'
```
- Listar por revisión:
	- GET RUTA_BASE/api/recordatorios/revision/:revisionId
```bash
curl RUTA_BASE/api/recordatorios/revision/rv123
```

Listar pendientes (no enviados y vencidos hasta ahora)
- GET RUTA_BASE/api/recordatorios/pendientes
```bash
curl RUTA_BASE/api/recordatorios/pendientes
```

Listar con Revisión (join recomendado)
- GET RUTA_BASE/api/recordatorios/con-revision?userId=<opcional>
```bash
curl "RUTA_BASE/api/recordatorios/con-revision?userId=7712045261"
```
- Respuesta (200): arreglo de objetos recordatorio con un campo extra `revision` (objeto Revisión o null), por ejemplo:
	{
		"_id": "677000000000000000000001",
		"userId": "7712045261",
		"mensaje": "Revisar entregable",
		"fechaHora": "2025-12-26T21:00:00.000Z",
		"revisionId": "rv123",
		"revision": {
			"id": "rv123",
			"nombre": "Revisión del sprint",
			"dueStart": "2025-12-26T18:00:00.000Z",
			"dueEnd": "2025-12-26T20:00:00.000Z",
			"actividadesRelacionadas": ["abc123", "def456"],
			...otros campos formateados del modelo Revisión...
		}
	}

Actualizar recordatorio (sincroniza con Calendar si hay evento o lo crea si falta)
- PUT RUTA_BASE/api/recordatorios/:id
- Body: cualquier combinación de { mensaje, fechaHora, duracionMinutos, timezone, tipo, activo, enviado, revisionId }
```bash
curl -X PUT RUTA_BASE/api/recordatorios/677000000000000000000001 \
	-H "Content-Type: application/json" \
	-d '{
		"mensaje": "Revisar entregable (nuevo)",
		"fechaHora": "2025-12-26T11:00:00-06:00",
		"duracionMinutos": 45
	}'
```
- Respuesta (200): objeto Recordatorio actualizado. Si se sincronizó con Calendar, `googleHtmlLink` puede actualizarse.

Marcar como enviado
- PATCH RUTA_BASE/api/recordatorios/:id/completar
```bash
curl -X PATCH RUTA_BASE/api/recordatorios/677000000000000000000001/completar
```

Eliminar (también elimina el evento en Calendar si existe)
- DELETE RUTA_BASE/api/recordatorios/:id
```bash
curl -X DELETE RUTA_BASE/api/recordatorios/677000000000000000000001
```
- Respuesta (200): { "message": "Recordatorio eliminado", "recordatorio": { ... } }

Buenas prácticas frontend
- Normaliza `userId` a 10 dígitos; el backend también lo sanitiza (últimos 10).
- Maneja `google.authRequired` mostrando un CTA para conectar Google.
- Usa `googleHtmlLink` para abrir directamente el evento en Calendar.
- Si necesitas listar eventos del Calendar (extra), existen endpoints en `RUTA_BASE/google/events` (POST/PUT/GET/DELETE) que trabajan con los mismos tokens, pero para recordatorios no es necesario usarlos directamente.

Archivos relevantes
- Rutas Google: [src/routes/google.js](src/routes/google.js)
- Controlador Google Calendar: [src/controllers/googleCalendarController.js](src/controllers/googleCalendarController.js)
- Servicio Google Calendar: [src/services/googleCalendarService.js](src/services/googleCalendarService.js)
- Modelo Recordatorio (con revisionId): [src/models/Recordatorio.js](src/models/Recordatorio.js)


**Usuarios (API)**

- Base path: `/api/users`

- Endpoint principal:
	- `GET /api/users/search` — buscar usuarios

- Query params aceptados:
	- `q` (string): búsqueda libre que se aplica a `firstName`, `lastName`, `email`, `phone`, `collaboratorId` (insensible a mayúsculas)
	- `email` (string): filtro por email (regex)
	- `collaboratorId` (string): filtro por collaboratorId (regex)
	- `limit` (number): limitar resultados

- Comportamiento:
	- Si no se envían filtros ni `limit`, devuelve todos los usuarios.
	- El servicio elimina `passwordHash` de la proyección (`.select("-passwordHash")`).
	- Los filtros usan regex case-insensitive (se construyen con `new RegExp(..., "i")`).

- Ejemplo curl y respuestas (texto plano):

	1) Búsqueda por texto libre
	curl -G "https://wlserver-production.up.railway.app/api/users/search" --data-urlencode "q=Juan"
	Respuesta (200):
	{ "items": [ { "_id":"611...","firstName":"Juan","lastName":"Perez","email":"juan@example.com","phone":"+521...","collaboratorId":"c123" } ] }

	2) Buscar por email
	curl -G "https://wlserver-production.up.railway.app/api/users/search" --data-urlencode "email=juan@example.com"
	Respuesta (200):
	{ "items": [ { "_id":"611...","firstName":"Juan","email":"juan@example.com" } ] }

	3) Limitar resultados
	curl -G "https://wlserver-production.up.railway.app/api/users/search" --data-urlencode "q=martin" --data-urlencode "limit=5"
	Respuesta (200):
	{ "items": [ /* hasta 5 usuarios */ ] }

- Errores y códigos:
	- 200: éxito → `{ "items": [...] }`
	- 500: error servidor → `{ "error": "mensaje" }`

Archivos relevantes:
- Rutas: `src/routes/usersRoutes.js`
- Controlador: `src/controllers/usersController.js`
- Servicio: `src/services/usersService.js`

- CRUD Recordatorios (rutas): [src/routes/recordatoriosRoutes.js](src/routes/recordatoriosRoutes.js)
- CRUD Recordatorios (controlador): [src/controllers/recordatoriosController.js](src/controllers/recordatoriosController.js)
- CRUD Recordatorios (servicio): [src/services/recordatoriosService.js](src/services/recordatoriosService.js)
- Modelo Revisión: [src/models/Revision.js](src/models/Revision.js)


**Recordatorios (Resumen de rutas y uso)**

- Base path: `/api/recordatorios`

- Rutas (definidas en `src/routes/recordatoriosRoutes.js`):
	- `POST /` → `postRecordatorio`
	- `POST /actividad/:actividadId` → `postRecordatorioParaActividad`
	- `POST /revision/:revisionId` → `postRecordatorioParaRevision` (recomendado)
	- `GET /` → `getTodosRecordatorios`
	- `GET /usuario/:userId` → `getRecordatoriosPorUserId`
	- `GET /actividad/:actividadId` → `getRecordatoriosPorActividadId`
	- `GET /revision/:revisionId` → `getRecordatoriosPorRevisionId`
	- `GET /pendientes` → `getRecordatoriosPendientes`
	- `GET /con-actividad` → `getRecordatoriosConActividad`
	- `GET /con-revision` → `getRecordatoriosConRevision`
	- `GET /:id` → `getRecordatorioPorId`
	- `PUT /:id` → `putRecordatorio`
	- `PATCH /:id/completar` → `patchMarcarRecordatorioEnviado`
	- `DELETE /:id` → `deleteRecordatorio`

- Headers comunes:
	- `Content-Type`: `application/json` para JSON
	- `x-device-id`: opcional/recomendado para identificar dispositivo

- Campos principales (modelo `Recordatorio`):
	- `userId` (string) — identificador del usuario (teléfono preferible, backend normaliza a últimos 10 dígitos)
	- `mensaje` (string) — título/descripcion del recordatorio
	- `fechaHora` (ISO string) — fecha y hora del recordatorio
	- `duracionMinutos` (number)
	- `tipo` (string) — `unica_vez` | `diaria` | `semanal` | `mensual`
	- `activo` (boolean)
	- `enviado` (boolean)
	- `revisionId` (string) — opcional, para asociar a Revisión
	- `googleEventId`, `googleHtmlLink` — campos que se llenan si se sincroniza con Google Calendar

- Ejemplos rápidos:
	- Crear recordatorio para una revisión (recomendado):
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/recordatorios/revision/rv123" \
	-H "Content-Type: application/json" \
	-d '{"userId":"+5217712045261","mensaje":"Revisar entregable","fechaHora":"2025-12-26T15:00:00-06:00","duracionMinutos":30}'
```
	- Respuesta (200/201):
```json
{
	"recordatorio": { "_id":"677000000000000000000001","userId":"7712045261","mensaje":"Revisar entregable","fechaHora":"2025-12-26T21:00:00.000Z","duracionMinutos":30,"activo":true,"enviado":false,"revisionId":"rv123" },
	"google": { "authRequired": false, "eventId":"abcd1234eventid","link":"https://www.google.com/calendar/event?eid=..." }
}
```

	- Crear recordatorio general (POST `/`):
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/recordatorios" \
	-H "Content-Type: application/json" \
	-d '{"userId":"+5217712045261","mensaje":"Llamar cliente","fechaHora":"2025-12-27T10:00:00-06:00"}'
```

	- Listar pendientes (no enviados o vencidos):
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/recordatorios/pendientes"
```

	- Marcar como enviado (PATCH `/ :id /completar`):
```bash
curl -X PATCH "https://wlserver-production.up.railway.app/api/recordatorios/677000000000000000000001/completar"
```

- Comportamientos importantes:
	- Si el usuario no tiene tokens de Google, la respuesta al crear/actualizar puede incluir `google.authRequired: true` y `google.authUrl` para que el frontend inicie la autorización.
	- Al crear/actualizar un recordatorio vinculado a una `revisionId`, el servicio intentará sincronizar con Google Calendar usando los tokens del `userId`.
	- Las fechas deben enviarse en ISO 8601; si se envía sin zona, el backend normaliza según la zona del servidor.

Archivos relevantes (rápido):
- Rutas: [src/routes/recordatoriosRoutes.js](src/routes/recordatoriosRoutes.js)
- Controlador: [src/controllers/recordatoriosController.js](src/controllers/recordatoriosController.js)
- Servicio: [src/services/recordatoriosService.js](src/services/recordatoriosService.js)


**Dropbox Notion Files (API)**

Base path: `/api/dropbox/notion-files`

- Descripción: endpoints para sincronizar, listar, buscar y manipular archivos/árboles vinculados entre Dropbox y Notion. Usa `upload` middleware para la ruta `/upload` (multipart/form-data, campo `file`).

- Headers comunes:
	- `Content-Type`: `application/json` para JSON; `multipart/form-data` para `/upload`.
	- Autenticación/identificación: depende del controlador; puede usar `x-device-id` u otros headers en funciones internas.

Endpoints:

1) Inicializar sincronización completa
- Método: POST
- Path: `/api/dropbox/notion-files/sync/initial`
- Descripción: inicia un proceso de sincronización inicial desde Dropbox hacia Notion/Mongo.
- Curl:
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/dropbox/notion-files/sync/initial"
```

2) Delta sync (delta incremental)
- Método: POST
- Path: `/api/dropbox/notion-files/sync/delta`
- Descripción: aplicar cambios incrementales (delta) desde Dropbox.

3) Listar nodos/archivos
- Método: GET
- Path: `/api/dropbox/notion-files/list`
- Query: filtros según controlador (path, folderId, page, size). Revisa controlador para opciones exactas.

4) Info (metadatos de un nodo)
- Método: GET
- Path: `/api/dropbox/notion-files/info`
- Query: identificar `id` o `path` para obtener información del nodo.

5) Buscar (sin links)
- Método: GET
- Path: `/api/dropbox/notion-files/search`
- Query: `q=texto`

6) Buscar con links (devuelve recursos y links relacionados)
- Método: GET
- Path: `/api/dropbox/notion-files/buscar`

7) Breadcrumbs (rastro de carpeta)
- Método: GET
- Path: `/api/dropbox/notion-files/breadcrumbs`
- Query: `id` o `path`

8) Árbol / tree
- Método: GET
- Path: `/api/dropbox/notion-files/tree`
- Descripción: devuelve estructura de árbol/árbol de carpetas para un nodo.

9) Estadísticas
- Método: GET
- Path: `/api/dropbox/notion-files/stats`
- Descripción: métricas/estadísticas de sincronización o almacenamiento.

10) Recomputar
- Método: POST
- Path: `/api/dropbox/notion-files/recompute`
- Descripción: fuerza recomputación interna de metadatos/indices.

11) Asegurar link (ensure-link)
- Método: POST
- Path: `/api/dropbox/notion-files/ensure-link`
- Descripción: crea o asegura un link público/compartido para un archivo.

12) Subir archivo (upload)
- Método: POST
- Path: `/api/dropbox/notion-files/upload`
- Content-Type: `multipart/form-data` (campo `file`)
- Curl ejemplo:
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/dropbox/notion-files/upload" \
	-F "file=@/ruta/archivo.pdf"
```

13) Crear carpeta
- Método: POST
- Path: `/api/dropbox/notion-files/folder`
- Body JSON: `{ "path": "/ruta/nueva" }` (según controlador)

14) Eliminar nodo
- Método: DELETE
- Path: `/api/dropbox/notion-files/node`
- Body/Query: identificar `id` o `path` a eliminar

15) Renombrar
- Método: PATCH
- Path: `/api/dropbox/notion-files/rename`
- Body JSON: `{ "id": "...", "name": "nuevoNombre" }`

- Respuestas típicas:
	- Éxito: `{ "success": true, "data": ... }` o `{ "ok": true, ... }` dependiendo del controlador.
	- Error: `{ "success": false, "error": "mensaje" }` o `{ "error": "mensaje" }`.

Notas:
- La ruta `/upload` usa el middleware `upload.single("file")` definido en `src/middlewares/upload.middleware.js`.
- Revisa `src/controllers/dropboxNotionFilesController.js` para detalles de parámetros aceptados en cada endpoint (filtros, paginación, keys esperadas).

**Google (API de Calendar + Auth)**

Base path: `/google` (montado en `app.js` como `/google`)

- Descripción: endpoints para autenticación con Google (OAuth) y operaciones con Calendar (crear/actualizar/borrar eventos, listar). Rutas definidas en `src/routes/google.js`.

- Headers comunes:
	- `Content-Type`: `application/json` para JSON.
	- Para operaciones que modifican datos desde frontend puede requerirse `userId` o `Authorization` dependiendo del flujo (revisa controladores para detalles).

Endpoints:

1) Obtener URL de autorización
- Método: GET
- Path: `/google/auth`
- Query: `userId` (recomendado: teléfono del usuario, e.g. `+5217712045261`)
- Uso: solicitar URL para redirigir al usuario a Google y obtener permisos.
- Curl:
```bash
curl -G "https://wlserver-production.up.railway.app/google/auth" --data-urlencode "userId=+5217712045261"
```
- Respuesta ejemplo:
```json
{ "authUrl": "https://accounts.google.com/o/oauth2/v2/auth?..." }
```

2) Callback de Google (recepción del code)
- Método: GET
- Path: `/google/callback`
- Query: `code`, `state` (state suele ser userId)
- Uso: Google redirige aquí automáticamente tras autorización; el backend guarda tokens y muestra página de éxito.

3) Logout / revocar tokens
- Método: DELETE
- Path: `/google/logout`
- Query: `userId` (o body) para identificar tokens a revocar
- Curl:
```bash
curl -X DELETE "https://wlserver-production.up.railway.app/google/logout?userId=+5217712045261"
```
- Respuesta ejemplo:
```json
{ "success": true, "message": "Tokens revocados y eliminados" }
```

4) Estado de integración (si el usuario tiene tokens)
- Método: GET
- Path: `/google/status`
- Query or headers: `userId` o auth segun implementación
- Curl:
```bash
curl "https://wlserver-production.up.railway.app/google/status?userId=+5217712045261"
```
- Respuesta ejemplo:
```json
{ "ok": true, "connected": true }
```

5) Crear evento en Calendar
- Método: POST
- Path: `/google/events`
- Body JSON ejemplo (mínimo):
```json
{
	"userId": "+5217712045261",
	"summary": "Nombre evento",
	"start": "2026-01-20T10:00:00-06:00",
	"end": "2026-01-20T11:00:00-06:00",
	"description": "Opcional"
}
```
- Curl:
```bash
curl -X POST "https://wlserver-production.up.railway.app/google/events" \
	-H "Content-Type: application/json" \
	-d '{"userId":"+5217712045261","summary":"Reunión","start":"2026-01-20T10:00:00-06:00","end":"2026-01-20T11:00:00-06:00"}'
```
- Respuesta éxito (ejemplo):
```json
{ "success": true, "data": { "eventId": "abcd1234", "htmlLink": "https://www.google.com/calendar/event?eid=..." } }
```

6) Actualizar evento
- Método: PUT
- Path: `/google/events/:eventId`
- Body JSON: campos a actualizar (summary, start, end, description, etc.)
- Curl:
```bash
curl -X PUT "https://wlserver-production.up.railway.app/google/events/abcd1234" \
	-H "Content-Type: application/json" \
	-d '{"summary":"Reunión actualizada"}'
```
- Respuesta éxito:
```json
{ "success": true, "data": { "eventId": "abcd1234" } }
```

7) Listar eventos
- Método: GET
- Path: `/google/events`
- Query opcionales: `userId`, `timeMin`, `timeMax`, `maxResults`
- Curl ejemplo:
```bash
curl -G "https://wlserver-production.up.railway.app/google/events" --data-urlencode "userId=+5217712045261" --data-urlencode "timeMin=2026-01-01T00:00:00Z"
```
- Respuesta ejemplo:
```json
{ "success": true, "data": [ { "eventId":"e1","summary":"...","start":"...","end":"..." }, ... ] }
```

8) Eliminar evento
- Método: DELETE
- Path: `/google/events/:eventId`
- Curl:
```bash
curl -X DELETE "https://wlserver-production.up.railway.app/google/events/abcd1234"
```
- Respuesta ejemplo:
```json
{ "success": true, "data": { "deleted": true, "eventId": "abcd1234" } }
```

Notas y comportamientos:
- `google/callback` es usado por OAuth; el frontend sólo debe solicitar `/google/auth` y redirigir al usuario.
- Algunas respuestas incluyen `googleHtmlLink` o `eventId` cuando el evento se crea en Calendar.
- Si el usuario no tiene tokens, endpoints de Calendar pueden devolver `authRequired: true` y `authUrl` para conectar cuenta.

Archivos relevantes:
- Rutas: `src/routes/google.js`
- Controladores: `src/controllers/googleAuthController.js`, `src/controllers/googleCalendarController.js`

**Opciones (API)**

Ruta base: `/api/opciones`

- Descripción: devuelve opciones/catálagos usados en la UI (prioridades, status, estados de proyecto, contactos, etc.). Los valores se obtienen desde Notion y se cachean en Mongo.

- Método: GET
- Path: `/api/opciones`
- Headers: ninguno especial requerido (puede usarse `x-device-id` si se necesita tracking, pero no es obligatorio)
- Query: ninguna
- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/opciones"
```
- Respuesta éxito (ejemplo):
```json
{
	"success": true,
	"source": "notion", // o "cache" o "fallback"
	"data": {
		"actividades": {
			"prioridad": ["Alta","Media","Baja"],
			"status": ["Por hacer","En progreso","Terminado"]
		},
		"proyectos": {
			"estadoFase": ["Idea","Desarrollo","Producción"],
			"contactos": ["Cliente","Proveedor"]
		}
	}
}
```
- Error (500):
```json
{ "success": false, "error": "mensaje" }
```

Notas:
- El servicio intenta siempre obtener las opciones desde Notion; si falla, devuelve la caché guardada en Mongo (`source: "cache"`). Si no hay caché, devuelve un fallback vacío (`source: "fallback"`).
- Estructura de `data`:
	- `actividades.prioridad`: array de strings
	- `actividades.status`: array de strings
	- `proyectos.estadoFase`: array de strings
	- `proyectos.contactos`: array de strings

Archivos relevantes:
- Ruta: `src/routes/opcionesRoutes.js`
- Controlador: `src/controllers/opcionesController.js`
- Servicio: `src/services/opcionesService.js`


**Reportes (API)**

- Base path: `/api/reportes`

- Descripción: endpoints para generar informes y resúmenes sobre eventos, actividades y revisiones.

- Rutas (definidas en `src/routes/reportsRoutes.js`):
	- `GET /eventos` → lista eventos (`listEventosController`)
	- `GET /resumen` → resumen por periodo (`resumenPorPeriodoController`)
	- `GET /custom` → resumen custom (`resumenCustomController`)
	- `GET /ultimos` → últimos registros (`ultimosController`)
	- `GET /comprobatoria` (o `/comprobatoria/:assignee`) → comprobatoria por colaborador (`comprobatoriaController`)
	- `GET /rezagadas` → tareas rezagadas (`rezagadasController`) (acepta `assignee` y `time` query)
	- `GET /revisiones-por-fecha` → revisiones por fecha y por colaborador (`revisionesPorFechaPorColaboradorController`) (query `date=YYYY-MM-DD`)

- Headers / query comunes:
	- `Content-Type`: `application/json`
	- Query/params específicos según endpoint (ver ejemplos abajo).

- Ejemplos y uso:
	1) Listar eventos
	- GET `/api/reportes/eventos`
	- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/reportes/eventos"
```
	- Respuesta (200):
```json
{ "success": true, "data": [ { "id":"e1","type":"actividad","date":"2026-01-10","summary":"..." }, ... ] }
```

	2) Resumen por periodo
	- GET `/api/reportes/resumen?desde=YYYY-MM-DD&hasta=YYYY-MM-DD` (los nombres de query pueden variar según implementación del controlador; revisar controlador si necesitas queries adicionales)
	- Curl ejemplo:
```bash
curl -G "https://wlserver-production.up.railway.app/api/reportes/resumen" --data-urlencode "desde=2026-01-01" --data-urlencode "hasta=2026-01-31"
```
	- Respuesta (200):
```json
{ "success": true, "data": { "totalActividades": 123, "porAssignee": { "juan@example.com": 34, "maria@example.com": 20 } } }
```

	3) Resumen custom
	- GET `/api/reportes/custom` con querys específicas para construir un resumen a medida (revisa `resumenCustomController` para parámetros aceptados)

	4) Últimos
	- GET `/api/reportes/ultimos` — devuelve los últimos eventos/actividades recientes
	- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/reportes/ultimos"
```

	5) Comprobatoria
	- GET `/api/reportes/comprobatoria?assignee=<email>` o `/api/reportes/comprobatoria/:assignee`
	- Uso: genera un reporte de comprobación para un colaborador (horarios, entregas, etc.)
	- Curl:
```bash
curl -G "https://wlserver-production.up.railway.app/api/reportes/comprobatoria" --data-urlencode "assignee=juan@example.com"
```

	6) Rezagadas
	- GET `/api/reportes/rezagadas?assignee=<email>&time=HH:mm` (time en zona MX) — lista tareas rezagadas para un `assignee` a partir de una hora de referencia
	- Curl ejemplo:
```bash
curl -G "https://wlserver-production.up.railway.app/api/reportes/rezagadas" --data-urlencode "assignee=juan@example.com" --data-urlencode "time=09:00"
```
	- Respuesta (200) ejemplo:
```json
{ "success": true, "data": [ { "id":"act1","titulo":"Tarea vencida","assignee":"juan@example.com","due":"2026-01-09T08:00:00-06:00" } ] }
```

	7) Revisiones por fecha por colaborador
	- GET `/api/reportes/revisiones-por-fecha?date=YYYY-MM-DD`
	- Curl:
```bash
curl -G "https://wlserver-production.up.railway.app/api/reportes/revisiones-por-fecha" --data-urlencode "date=2026-01-10"
```
	- Respuesta (200):
```json
{ "success": true, "data": [ { "revisionId":"rv123","colaborador":"juan@example.com","dueStart":"2026-01-10T10:00:00-06:00" } ] }
```

- Formato general de respuesta: normalmente `{ "success": true, "data": ... }` o `{ "success": false, "error": "..." }`.

Archivos relevantes:
- `src/routes/reportsRoutes.js`
- `src/controllers/reportsController.js`

**Proyectos (API)**

Ruta base: `/api/proyectos`

- Descripción: CRUD de proyectos sincronizados con Notion. Devuelven objetos formateados por `formatProyecto` (id, nombre, url, fechas, contactos, grupos, estadoFase, flags, etc.).

- Headers comunes:
		- `Content-Type`: `application/json` para JSON; `multipart/form-data` si se suben archivos desde el cliente (según flujo).
		- `x-device-id`: recomendado para acciones que modifican datos.

Endpoints principales:

1) Listar proyectos
- Método: GET
- Path: `/api/proyectos`
- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/proyectos"
```
- Respuesta ejemplo:
```json
{ "success": true, "data": [ { "id":"proj123","nombre":"Mi proyecto","estadoFase":"Desarrollo" } ] }
```

2) Obtener proyecto por ID
- Método: GET
- Path: `/api/proyectos/:id`
- Curl:
```bash
curl -X GET "https://wlserver-production.up.railway.app/api/proyectos/proj123"
```
- Respuesta ejemplo:
```json
{ "success": true, "data": { "id":"proj123","nombre":"Mi proyecto","estadoFase":"Desarrollo","contactos":["cliente@example.com"] } }
```

3) Crear proyecto
- Método: POST
- Path: `/api/proyectos`
- Body JSON: campos del proyecto (nombre, fechaInicio, fechaFin, contactos, grupos, estadoFase, url, notas, etc.)
- Headers: `Content-Type: application/json`, `x-device-id` recomendado
- Curl ejemplo:
```bash
curl -X POST "https://wlserver-production.up.railway.app/api/proyectos" \
	-H "Content-Type: application/json" -H "x-device-id: MI_DEVICE_ID" \
	-d '{"nombre":"Nuevo proyecto","estadoFase":"Idea","contactos":["cliente@example.com"]}'
```
- Respuesta éxito (201/200):
```json
{ "success": true, "data": { "id":"proj123","nombre":"Nuevo proyecto","estadoFase":"Idea" } }
```

4) Actualizar proyecto
- Método: PUT
- Path: `/api/proyectos/:id`
- Body JSON: campos a actualizar
- Headers: `Content-Type: application/json`, `x-device-id` recomendado
- Curl:
```bash
curl -X PUT "https://wlserver-production.up.railway.app/api/proyectos/proj123" \
	-H "Content-Type: application/json" -H "x-device-id: MI_DEVICE_ID" \
	-d '{"estadoFase":"Producción","url":"https://repo.example"}'
```
- Respuesta éxito:
```json
{ "success": true, "data": { "id":"proj123","nombre":"Nuevo proyecto","estadoFase":"Producción" } }
```

5) Eliminar proyecto
- Método: DELETE
- Path: `/api/proyectos/:id`
- Headers: `x-device-id` recomendado
- Curl:
```bash
curl -X DELETE "https://wlserver-production.up.railway.app/api/proyectos/proj123" -H "x-device-id: MI_DEVICE_ID"
```
- Respuesta ejemplo:
```json
{ "success": true, "data": { "deleted": true, "id": "proj123" } }
```

Eventos Socket.IO emitidos (útiles para frontend en tiempo real):
- `proyecto_creado` con payload `{ proyecto }`
- `proyecto_actualizado` con payload `{ proyecto }`
- `proyecto_eliminado` con payload `{ id }`

Notas y comportamiento:
- Los proyectos se sincronizan con Notion: las operaciones pueden propagar cambios a Notion y a la colección Mongo local.
- El formato devuelto es el producido por `src/utils/formatProyecto.js`.
- Algunas operaciones pueden emitir eventos Socket.IO para notificar cambios en tiempo real.

Archivos relevantes:
- Rutas: [src/routes/proyectosRoutes.js](src/routes/proyectosRoutes.js)
- Controlador: [src/controllers/proyectosController.js](src/controllers/proyectosController.js)
- Servicio: [src/services/proyectoService.js](src/services/proyectoService.js)
- Formateo/Helpers: [src/utils/formatProyecto.js](src/utils/formatProyecto.js)


**Uploads / Dropbox (servicio `uploadService`)**

- Propósito: subir archivos a Dropbox y devolver enlaces públicos directos (`raw=1`). El servicio principal es `subirArchivosADropbox(nombreCarpeta, archivos)` en `src/services/uploadService.js`.

- Comportamiento principal:
	- Carpeta base en Dropbox: `/CARPETA UNIKA` (constante `CARPETA_UNIKA`).
	- Para cada archivo recibido (objeto Multer), se sube a `/{CARPETA_UNIKA}/{nombreCarpeta}/{originalname}` con modo `overwrite`.
	- Se intenta crear un link compartido; si ya existe, recupera el link existente.
	- El link devuelto es transformado para servir contenido directo (`?dl=0` → `?raw=1`).
	- Devuelve array de objetos `{ nombre, dropboxPath, link }`.
	- Si falla la creación del link para un archivo, el servicio loggea el error y continúa con los demás archivos.

- Uso desde controladores (ejemplos comunes):
	- Revisiones: `POST /api/revisiones` y `PUT /api/revisiones/:id` aceptan `archivos` (multipart) y el service sube estos archivos y añade URLs a la propiedad `Archivos` en Notion.
	- Actividades: endpoints `POST /api/actividades` y `PUT /api/actividades/:id` aceptan `archivos` y usan flujos similares.
	- Dropbox Notion Files: `/api/dropbox/notion-files/upload` también usa middleware `upload.single('file')` y lógica relacionada.

- Requisitos y límites:
	- Los endpoints que usan Multer suelen estar configurados con límite de 10 MB por archivo.
	- `nombreCarpeta` debe ser una cadena simple (habitualmente tipo `REVISIONES`, `ACTIVIDADES` u otro identificador del flujo).

- Ejemplo genérico (curl multipart) — subir un archivo para revisiones:
	curl -X POST "https://wlserver-production.up.railway.app/api/revisiones" \
		-H "x-device-id: MI_DEVICE_ID" \
		-F "nombre=Revisión prueba" \
		-F "mensaje=Detalles..." \
		-F "archivos=@/ruta/adjunto.pdf"

	Respuesta (ejemplo):
	{ "success": true, "data": { "id":"rv999","archivosAdjuntos":[ { "name":"adjunto.pdf","url":"https://...raw=1" } ] } }

- Ejemplo programático (uso de `subirArchivosADropbox`):
	- Entrada: `nombreCarpeta = 'REVISIONES'`, `archivos = req.files` (array de objetos Multer con `originalname` y `buffer`).
	- Salida: `[{ nombre: 'adjunto.pdf', dropboxPath: '/CARPETA UNIKA/REVISIONES/adjunto.pdf', link: 'https://...raw=1' }, ...]`.

- Errores y comportamiento:
	- Si la subida o la creación de link falla para un archivo, el servicio registra el error y continúa; los archivos con fallo se omiten del resultado.
	- Los controladores deben verificar el array devuelto y manejar casos donde faltan archivos esperados.

Archivos relevantes:
- `src/services/uploadService.js`
- Middlewares que usan uploads: `src/routes/revisionsRoutes.js` (usa `upload.array('archivos')`), `src/routes/actividadesRoutes.js`, `src/controllers/dropboxNotionFilesController.js` (ruta `/upload`).

**Revisiones (API)**

- Ruta base: `/api/revisiones`

- Descripción general: CRUD y utilidades para revisiones (sincronizadas con Notion). Acepta uploads de archivos en creación/edición y aplica rate-limit en operaciones de edición.

- Headers comunes:
	- `Content-Type`: `application/json` para JSON; `multipart/form-data` para uploads (campo `archivos`)
	- `x-device-id`: recomendado en endpoints de edición/creación para tracking y rate-limit

- Endpoints (definidos en `src/routes/revisionsRoutes.js`):
	- `GET /api/revisiones` → Obtener todas las revisiones (`getRevisiones`)
	- `GET /api/revisiones/actividad/:actividadId` → Revisiones por actividad (`getRevisionesPorActividad`)
	- `GET /api/revisiones/search/:param` → Buscar por parámetro específico en título (`getRevisionesPorParametro`)
	- `GET /api/revisiones/buscar?q=...` → Buscar texto libre (`buscarRevisionesController`)
	- `GET /api/revisiones/fechas?fechaInicio=...&fechaFin=...` → Revisiones por rango de fechas (`getRevisionesPorFecha`) (fechaInicio requerido)
	- `GET /api/revisiones/dia-actual/resumen` → Resumen del día por colaborador (`getRevisionesResumenDiaPorColaboradorController`)
	- `POST /api/revisiones/mover-fechas` → Mover varias revisiones por horas (body: `{ ids: [...], horas: N }`) (rate-limited)
	- `POST /api/revisiones/:id/confirmacion` → Marcar/desmarcar confirmación (acepta `confirm` en body/query) (rate-limited)
	- `POST /api/revisiones/orden` → Actualizar órdenes (DND) (body: `items` o single `{id, orden}`) (rate-limited)
	- `GET /api/revisiones/:id` → Obtener revisión por id (`getRevision`)
	- `POST /api/revisiones/:id/duplicate` → Duplicar revisión con nuevo título (body: `{ titulo }`) (rate-limited)
	- `POST /api/revisiones/migrate-assignees` → Migrar assignees desde Notion (one-off) (rate-limited)
	- `GET /api/revisiones/:id/preview` → Obtener preview Notion (cached)
	- `POST /api/revisiones/:id/preview` → Forzar refresh de preview (rate-limited)
	- `POST /api/revisiones` → Crear revisión (acepta `archivos` multipart) — `postRevision`
	- `PUT /api/revisiones/:id` → Actualizar revisión (acepta `archivos` multipart) — `putRevision`
	- `DELETE /api/revisiones/:id` → Eliminar revisión — `deleteRevision`

- Comportamiento y notas clave:
	- Límite de archivos: Multer configurado con 10MB por archivo. Use campo `archivos` para uploads.
	- Rate-limit: ediciones (PUT, POST que modifiquen) usan limitador 25 req/min por `x-device-id`.
	- IDs: acepta notionId con o sin guiones y normaliza internamente.
	- Al crear/actualizar se suben archivos a Dropbox y se guardan como `Archivos` en Notion (si aplica).
	- Al actualizar, el controller registra eventos de reporte si una revisión fue marcada como `terminadaPendienteRevision`.
	- Duplicar crea una nueva página en la DB de Revisiones y copia bloques sanitizados (columnas, synced_blocks, etc.)

- Ejemplos curl y respuestas JSON (texto plano):

	1) Listar todas las revisiones
	curl -X GET "https://wlserver-production.up.railway.app/api/revisiones"
	Respuesta (200):
	{ "success": true, "data": [ { "id":"rv123","nombre":"Revision A","dueStart":"2026-01-10T10:00:00-06:00" } ] }

	2) Buscar por texto libre
	curl -G "https://wlserver-production.up.railway.app/api/revisiones/buscar" --data-urlencode "q=texto"
	Respuesta (200):
	{ "success": true, "data": [ { "id":"rv123","nombre":"Revision matching texto" } ] }

	3) Obtener por ID
	curl -X GET "https://wlserver-production.up.railway.app/api/revisiones/rv123"
	Respuesta (200):
	{ "success": true, "data": { "id":"rv123","nombre":"Revision A","archivosAdjuntos":[/*...*/] } }

	4) Crear revisión (multipart con archivos)
	curl -X POST "https://wlserver-production.up.railway.app/api/revisiones" \
		-H "x-device-id: MI_DEVICE_ID" \
		-F "nombre=Revisión prueba" \
		-F "mensaje=Detalles..." \
		-F "archivos=@/ruta/adjunto.pdf"
	Respuesta (200/201):
	{ "success": true, "data": { "id":"rv999","nombre":"Revisión prueba","archivosAdjuntos":[ { "name":"adjunto.pdf","url":"https://..." } ] } }

	5) Actualizar revisión (puede incluir archivos)
	curl -X PUT "https://wlserver-production.up.railway.app/api/revisiones/rv123" \
		-H "x-device-id: MI_DEVICE_ID" \
		-F "mensaje=Actualizado" \
		-F "archivos=@/ruta/nuevo.pdf"
	Respuesta (200):
	{ "success": true, "data": { "id":"rv123","nombre":"Revision A","archivosAdjuntos":[ /* actualizado */ ] } }

	6) Marcar confirmación
	curl -X POST "https://wlserver-production.up.railway.app/api/revisiones/rv123/confirmacion" -H "x-device-id: MI_DEVICE_ID" -d "confirm=true"
	Respuesta (200):
	{ "success": true, "data": { /* revisión actualizada con confirmacion */ } }

	7) Mover varias revisiones por horas
	curl -X POST "https://wlserver-production.up.railway.app/api/revisiones/mover-fechas" \
		-H "Content-Type: application/json" -d '{"ids":["rv1","rv2"],"horas":2}'
	Respuesta (200):
	{ "success": true, "data": [ { "id":"rv1","result":"ok" }, { "id":"rv2","result":"ok" } ] }

	8) Obtener preview de Notion (cached)
	curl -X GET "https://wlserver-production.up.railway.app/api/revisiones/rv123/preview"
	Respuesta (200):
	{ "success": true, "data": { "html":"<html>...preview...","assets":[/*...*/] } }

	9) Duplicar revisión
	curl -X POST "https://wlserver-production.up.railway.app/api/revisiones/rv123/duplicate" \
		-H "Content-Type: application/json" -d '{"titulo":"Copia de RV123","actividadId":"act123"}'
	Respuesta (200):
	{ "success": true, "data": { "id":"rv124","nombre":"Copia de RV123" } }

- Errores y códigos comunes:
	- 200/201: éxito → { "success": true, "data": ... }
	- 400: parámetros faltantes / validación → { "success": false, "error": "mensaje" }
	- 404: no encontrado → { "success": false, "error": "Revisión no encontrada" }
	- 500: error servidor → { "success": false, "error": "mensaje" }

Archivos relevantes:
- Rutas: `src/routes/revisionsRoutes.js`
- Controlador: `src/controllers/revisionsController.js`
- Servicio: `src/services/revisionService.js`
- Utilidades de clonación/props: `src/services/revisionesCommon.js`


**Socket.IO / Tiempo real**

- Conexión y handshake:
	- URL: la misma base del servidor, p. ej. `https://wlserver-production.up.railway.app`.
	- El cliente debe abrir socket pasando `auth` en el handshake con los siguientes campos:
		- `token` (opcional): JWT del usuario; si se envía, el servidor intentará decodificarlo y asociar `userId`.
		- `deviceId` (recomendado): identificador del dispositivo (se registra en `Presence`).
		- `meta` (opcional): objeto con info adicional (p. ej. `{ platform: 'web' }`).
	- Ejemplo de conexión (cliente JS):
		const socket = io('https://wlserver-production.up.railway.app', { auth: { token: '<JWT>', deviceId: 'MI_DEVICE_ID', meta: { platform: 'web' } } });

- Eventos que el servidor emite:
	- `presence:online` — payload: `{ online: [ { name, email, phone, avatar }, ... ] }` cada vez que cambia la lista de conexiones.
	- `chat:pending` — payload: `{ items: [ ...mensajes pendientes... ] }` enviado al socket del usuario cuando se conecta (si tiene mensajes no entregados).
	- `chat:message` — payload: `{ msg }` enviado a destinatarios cuando alguien envía un mensaje por socket o por REST.
	- `actividad_creada`, `actividad_actualizada`, `actividad_eliminada` — payloads: actividad formateada o `{ id }` (emitidos desde controladores de actividades y opciones).
	- `proyecto_creado`, `proyecto_actualizado`, `proyecto_eliminado` — payloads: proyecto formateado o `{ id }`.
	- `revision_creada`, `revision_actualizada`, `revision_eliminada`, `revision_foco_eliminado` — payloads: revisión formateada o `{ id }`.
	- `correo_creado`, `correo_actualizado`, `correo_eliminado` — payloads: correo formateado o `{ id }`.
	- `finanza_creada`, `finanza_actualizada`, `finanza_eliminada` — payloads: finanza formateada o `{ id }`.
	- `interrupcion_creada`, `interrupcion_actualizada`, `interrupcion_cerrada`, `interrupcion_eliminada` — payloads según controlador.
	- `note_created`, `note_updated`, `note_deleted`, `note_seen` — payloads: nota o `{ id }` (emisiones desde `notesController`).

- Eventos que el cliente puede emitir al servidor:
	- En el handshake: enviar `auth: { token, deviceId, meta }`.
	- `presence:ping` — sin payload; actualiza `lastPingAt` en `Presence` para mantener la sesión como activa.
	- `chat:send` — payload: `{ toUserId, text }`; el servidor creará el mensaje y reenviará `chat:message` a sockets del destinatario y marcará como entregado.
	- (No recomendado por defecto) `chat:send-files` — si se implementa, debería enviar archivos codificados; actualmente el flujo recomendado es REST multipart.

- Ejemplo flujo chat en tiempo real:
	1) Cliente A conecta con su `token` y `deviceId`.
	2) Si hay mensajes pendientes, recibe `chat:pending` con `items`.
	3) Cliente A emite `chat:send` con `{ toUserId, text }`.
	4) Servidor guarda el mensaje y emite `chat:message` a sockets del `toUserId`.
	5) Si el destinatario está desconectado, el mensaje queda pendiente y se entregará en la próxima conexión (y se marcará como entregado cuando se envíe `chat:pending`).

- Buenas prácticas:
	- Siempre enviar `deviceId` en el handshake para permitir rate-limits y auditoría por dispositivo.
	- El frontend debe suscribirse a `presence:online` para mostrar usuarios conectados en tiempo real.
	- Preferir uploads por REST multipart en lugar de por socket.
	- Manejar reintentos y reconexión de socket con `reconnectionAttempts` y escucha de `connect_error`.

- Archivos relevantes:
	- `src/server.js` (configuración de Socket.IO y handlers de `connection`)
	- Controladores que emiten eventos: `src/controllers/actividadesController.js`, `src/controllers/proyectosController.js`, `src/controllers/revisionsController.js`, `src/controllers/correosController.js`, `src/controllers/finanzasController.js`, `src/controllers/notesController.js`, `src/controllers/interrupcionesController.js`, etc.



**Feedback / Comentarios (API)**

- Base path: `/api/feedback`

- Propósito: permitir que usuarios (autenticados o anónimos) envíen comentarios y calificaciones (rating) sobre recursos genéricos identificados por `targetType` + `targetId`. Diseñado para escalar a cualquier tipo de "target" (páginas, productos, issues, etc.).

- Modelo (resumen):
	- `author` (opcional): `{ user, name, email }`
	- `targetType` (string) — ejemplo: `page`, `product` (requerido)
	- `targetId` (string) — id del recurso dentro del target. Para `targetType: "page"` este campo es opcional (feedback de la página completa).
	- `comment` (string) — texto del comentario (requerido)
	- `rating` (number) — 1..5 (requerido)
	- `metadata` (object) — datos libres (opcional)
	- `status` — `visible|hidden|archived` (para moderación)
	- `ip`, `device`, `locale` — info de contexto capturada en creación

Rutas y ejemplos (usar `qRUTA_BASE` como host base):

1) Crear feedback
- Método: POST
- Path: `qRUTA_BASE/api/feedback`
- Body (JSON):

```json
{
	"targetType": "page",
	"targetId": "page_abc123",
	"comment": "Muy útil esta página",
	"rating": 5,
	"metadata": { "browser": "Chrome" },
	"author": { "name": "Juan", "email": "juan@example.com" }
}
```

- Headers recomendados:
	- `Content-Type: application/json`
	- `x-device-id`: opcional (identifica dispositivo)

- Respuesta (201):

```json
{ "success": true, "data": { "_id": "...", "targetType": "page", "targetId": "page_abc123", "comment": "Muy útil...", "rating": 5, "createdAt": "..." } }
```

2) Listar feedback (filtros, paginación)
- Método: GET
- Path: `qRUTA_BASE/api/feedback`
- Query params:
	- `targetType` (opcional)
	- `targetId` (opcional)
	- `page` (opcional, default 1)
	- `limit` (opcional, default 20)
	- `minRating` (opcional, filtra >= valor)
	- `sortBy` (opcional, default `createdAt`)
	- `sortDir` (opcional, `asc` o `desc`)

- Ejemplo:
```
GET qRUTA_BASE/api/feedback?targetType=page&targetId=page_abc123&page=1&limit=10
```

- Respuesta (200):

```json
{
	"success": true,
	"data": [ { "_id":"...","comment":"...","rating":5, "author":{...} } ],
	"meta": { "total": 42, "page": 1, "limit": 10 }
}
```

3) Obtener un feedback por id
- Método: GET
- Path: `qRUTA_BASE/api/feedback/:id`
- Respuesta (200): `{ "success": true, "data": { ... } }` o 404 si no existe

4) Actualizar feedback (moderación o corrección)
- Método: PUT
- Path: `qRUTA_BASE/api/feedback/:id`
- Body: campos a actualizar (p. ej. `comment`, `rating`, `status`)
- Respuesta (200): `{ "success": true, "data": { ...updated... } }`

5) Eliminar feedback
- Método: DELETE
- Path: `qRUTA_BASE/api/feedback/:id`
- Respuesta (200): `{ "success": true, "data": { ...deleted... } }`

6) Listar comentarios por usuario (user id)
- Método: GET
- Path: `qRUTA_BASE/api/feedback/user/:userId`
- Query params: `page`, `limit`, `sortBy`, `sortDir`
- Respuesta (200):

```json
{ "success": true, "data": [ /* comentarios del usuario */ ], "meta": { "total": 5, "page": 1, "limit": 20 } }
```

7) Listar comentarios por email de autor
- Método: GET
- Path: `qRUTA_BASE/api/feedback/user-email?email=juan@example.com`
- Respuesta (200): similar a la anterior

Notas de integración y recomendaciones:
- Use `targetType` + `targetId` para poder indexar y escalar a múltiples tipos de recursos sin cambiar schema.
- Para endpoints de listado, el response incluye `meta` con `total`, `page` y `limit` para facilitar paginación en frontend.
- Para moderación, actualizar `status` a `hidden` o `archived` para ocultar sin borrar la evidencia.
- Campo `author.user` puede enlazar a `User` si existe la relación; si es anónimo, enviar sólo `author.name`/`email` opcional.
- Se captura `ip`, `device` y `locale` en la creación para auditoría; `x-device-id` es recomendado en el header.

Ejemplo completo de flujo (curl):

Crear:
```bash
curl -X POST "qRUTA_BASE/api/feedback" -H "Content-Type: application/json" -d '{"targetType":"page","targetId":"page_abc123","comment":"Excelente","rating":5}'
```

Listar:
```bash
curl "qRUTA_BASE/api/feedback?targetType=page&targetId=page_abc123"
```

Archivos/implementación relevantes:
- Modelo: `src/models/Feedback.js`
- Servicio: `src/services/feedbackService.js`
- Controlador: `src/controllers/feedbackController.js`
- Rutas: `src/routes/feedbackRoutes.js`



**WhatsApp: crear grupos y registro de teléfonos**

- Base env: añade `WHATSAPP_BASE_URL` en tu `.env` (ej: `https://mi-whatsapp-service.example.com`).

Endpoints añadidos en esta integración:

- Crear grupo (proxy hacia el servicio externo):
	- Método: POST
	- Path: `/api/whatsapp/create-group`
	- Middlewares: `touchRequest`, `ensureAwake` (implementación ligera)
	- Body JSON mínimo:
		- `name` (string) — obligatorio
		- `participants` (array[string]) — opcional (ej: `"521234567890"` o `"521234567890@s.whatsapp.net"`)
		- `mentionAll` (boolean) — opcional
		- `welcomeMessage` (string) — opcional
		- `notionId` (string) — opcional
	- Respuestas: la API devuelve exactamente lo que el servicio remoto retorne. Errores 429 se manejan y devuelven `rate-overlimit`.

- CRUD de clientes (nueva colección Mongo local `WhatsappClient`):
	- Listar: `GET /api/whatsapp/clients` → devuelve `{ success: true, items: [...] }`
	- Obtener: `GET /api/whatsapp/clients/:id` → `{ success: true, data: {...} }`
	- Crear: `POST /api/whatsapp/clients` → `{ success: true, data: {...} }` (body example: `{ "name":"ACME","phones":["521234567890"] }`)
	- Actualizar: `PUT /api/whatsapp/clients/:id`
	- Eliminar: `DELETE /api/whatsapp/clients/:id`

Notas de implementación:
- `WHATSAPP_BASE_URL` debe apuntar a un servicio que implemente la ruta `POST /create-group` según la documentación de tu integrador.
- El servicio `src/services/whatsappService.js` hace un `POST` a `${WHATSAPP_BASE_URL}/create-group` con el body recibido.
- La colección `WhatsappClient` es independiente de Notion y sirve para almacenar números/metadata localmente; puedes replicar o sincronizar con Notion si lo deseas.

Ejemplo rápido (crear grupo):
```bash
curl -X POST "http://localhost:4000/api/whatsapp/create-group" \
	-H "Content-Type: application/json" \
	-d '{"name":"Grupo de Prueba","participants":["521234567890"],"welcomeMessage":"Bienvenidos"}'
```


**Shared Auth — Uso Detallado**

Esta sección explica cómo usar las rutas compartidas (`/api/shared-auth`) de forma segura desde el frontend o scripts de integración. NO incluyas credenciales reales en repositorios o logs.

- Ruta base: `/api/shared-auth`

- Endpoints principales:
	- `POST /api/shared-auth/login` — Inicia sesión compartido.
		- Qué enviar: `Content-Type: application/json` con body `{ "user": "<usuario>", "pass": "<password>" }`.
		- Respuesta (200):
			```json
			{
				"token": "<jwt_access_token>",
				"refreshToken": "<jwt_refresh_token>"
			}
			```
		- Errores comunes:
			- `400` `{ "message": "Missing credentials" }`
			- `401` `{ "message": "Invalid credentials" }`

	- `POST /api/shared-auth/refresh` — Renueva tokens.
		- Qué enviar: incluir el refresh token en el header `x-shared-refresh`.
		- Header: `x-shared-refresh: <refresh_token>`
		- Respuesta (200): `{ "token": "<new_jwt_access_token>", "refreshToken": "<new_jwt_refresh_token>" }`


- Cómo usar el `token` en peticiones protegidas:
	- Header esperado: `x-shared-token: <jwt_access_token>`
	- Ejemplo curl (usar el token recibido en `/login`):
		```bash
		curl https://RUTA_BASE/api/actividades \
			-H "x-shared-token: <jwt_access_token>"
		```

- Ejemplo paso a paso (curl) con placeholders:
	1) Login (no subir estas credenciales a repositorios):
		```bash
		curl -X POST https://RUTA_BASE/api/shared-auth/login \
			-H "Content-Type: application/json" \
			-d '{"user":"your_shared_user@example.com","pass":"your_shared_password"}'
		```
		- Respuesta: guarda `token` y `refreshToken`.

	2) Llamada a ruta protegida con `x-shared-token`:
		```bash
		curl https://RUTA_BASE/api/actividades -H "x-shared-token: <token_aqui>"
		```

	3) Renovar token cuando expira (usar `x-shared-refresh`):
		```bash
		curl -X POST https://RUTA_BASE/api/shared-auth/refresh -H "x-shared-refresh: <refreshToken_aqui>"
		```

- Uso desde JavaScript (fetch):
	```javascript
	// Login
	const resp = await fetch('/api/shared-auth/login', {
		method: 'POST',
		headers: { 'Content-Type': 'application/json' },
		body: JSON.stringify({ user: 'your_shared_user@example.com', pass: 'your_shared_password' })
	});
	const { token } = await resp.json();

	// Llamada protegida
	await fetch('/api/actividades', { headers: { 'x-shared-token': token } });
	```





