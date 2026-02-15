# API Reservaciones POS Restaurant - Odoo 19 JSON-RPC 2.0

> Referencia para implementar un MCP Streamable HTTP que gestione reservaciones del POS Restaurant.

## Configuración Base

| Parámetro | Valor |
|-----------|-------|
| **Base URL** | `https://odoo.cocinandosonrisas.co` |
| **Endpoint pattern** | `POST /json/2/{modelo}/{metodo}` |
| **Auth** | `Authorization: Bearer <API_KEY>` |
| **Content-Type** | `application/json` |
| **Database Header** | `X-Odoo-Database: cocson_2026` |
| **Appointment Type ID** | `1` (nombre: "Tabla", schedule: resources) |

### Headers comunes

```
Authorization: Bearer <API_KEY>
Content-Type: application/json
X-Odoo-Database: cocson_2026
```

---

## Endpoints

### 1. Crear Reservación

**`POST /json/2/calendar.event/create`**

Crea una reservación vinculada a una mesa del restaurante.

**Request Body:**
```json
{
  "vals": {
    "name": "Juan Pérez - Mesa 5",
    "phone_number": "+573113396736",
    "start": "2026-02-15 19:00:00",
    "stop": "2026-02-15 21:00:00",
    "duration": 2.0,
    "appointment_type_id": 1,
    "appointment_status": "booked",
    "booking_line_ids": [[0, 0, {
      "appointment_resource_id": 44,
      "capacity_reserved": 2
    }]]
  }
}
```

**Campos:**

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `name` | string | sí | Nombre del cliente o descripción |
| `phone_number` | string | no | Teléfono del cliente |
| `start` | datetime | sí | Inicio en UTC `"YYYY-MM-DD HH:MM:SS"` |
| `stop` | datetime | sí | Fin en UTC `"YYYY-MM-DD HH:MM:SS"` |
| `duration` | float | no | Duración en horas (se calcula de start/stop si omite) |
| `appointment_type_id` | int | sí | Siempre `1` para reservas de restaurante |
| `appointment_status` | string | no | `"booked"` (default), `"attended"`, `"no_show"`, `"cancelled"` |
| `booking_line_ids` | array | sí | Comando ORM para vincular mesa(s) |
| `partner_ids` | array | no | `[[4, PARTNER_ID]]` para vincular cliente existente |
| `description` | string | no | Notas adicionales |

**booking_line_ids** usa comandos ORM:
- `[0, 0, {...}]` = crear nueva línea
- `appointment_resource_id` = ID del recurso (NO es el ID de la mesa, ver tabla de mesas abajo)
- `capacity_reserved` = número de personas

**Response:**
```json
[5]
```
Retorna array con el ID del `calendar.event` creado.

**Efecto**: El POS recibe notificación `TABLE_BOOKING` → `ADDED` automáticamente.

---

### 2. Buscar Reservación por Teléfono

**`POST /json/2/calendar.event/search_read`**

**Request Body (búsqueda exacta):**
```json
{
  "domain": [
    ["phone_number", "=", "+573113396736"],
    ["appointment_type_id", "=", 1],
    ["appointment_status", "=", "booked"]
  ],
  "fields": ["id", "name", "start", "stop", "phone_number", "appointment_status", "appointment_resource_ids", "booking_line_ids"]
}
```

**Request Body (búsqueda parcial):**
```json
{
  "domain": [
    ["phone_number", "ilike", "3113396736"],
    ["appointment_type_id", "!=", false]
  ],
  "fields": ["id", "name", "start", "stop", "phone_number", "appointment_status", "appointment_resource_ids"]
}
```

**Response:**
```json
[
  {
    "id": 5,
    "name": "Juan Pérez - Mesa 5",
    "start": "2026-02-15 19:00:00",
    "stop": "2026-02-15 21:00:00",
    "phone_number": "+573113396736",
    "appointment_status": "booked",
    "appointment_resource_ids": [44]
  }
]
```

---

### 3. Consultar Reservaciones de Hoy

**`POST /json/2/calendar.event/search_read`**

**Request Body:**
```json
{
  "domain": [
    ["appointment_type_id", "=", 1],
    ["start", ">=", "2026-02-15 00:00:00"],
    ["start", "<=", "2026-02-15 23:59:59"]
  ],
  "fields": ["id", "name", "start", "stop", "phone_number", "appointment_status", "appointment_resource_ids", "booking_line_ids"]
}
```

---

### 4. Cancelar Reservación por ID

**`POST /json/2/calendar.event/set_cancelled`**

**Request Body:**
```json
{
  "ids": [5]
}
```

**Response:**
```json
true
```

**Efecto**: Cambia `appointment_status` → `"cancelled"` y `active` → `false`. El POS recibe notificación `TABLE_BOOKING` → `REMOVED`.

---

### 5. Cancelar Reservación por Teléfono (2 pasos)

**Paso 1: Obtener IDs**

**`POST /json/2/calendar.event/search`**

```json
{
  "domain": [
    ["phone_number", "ilike", "3113396736"],
    ["appointment_status", "=", "booked"]
  ]
}
```

**Response:** `[5, 12, 18]`

**Paso 2: Cancelar**

**`POST /json/2/calendar.event/set_cancelled`**

```json
{
  "ids": [5, 12, 18]
}
```

---

### 6. Marcar Reservación como "Asistió"

**`POST /json/2/calendar.event/set_attended`**

**Request Body:**
```json
{
  "ids": [5]
}
```

---

### 7. Modificar Reservación

**`POST /json/2/calendar.event/write`**

**Request Body:**
```json
{
  "ids": [5],
  "vals": {
    "name": "Juan Pérez - Mesa 10",
    "start": "2026-02-15 20:00:00",
    "stop": "2026-02-15 22:00:00",
    "phone_number": "+573001234567",
    "description": "Cumpleaños, pedir torta"
  }
}
```

Para cambiar la mesa, hay que reemplazar las booking_lines:
```json
{
  "ids": [5],
  "vals": {
    "booking_line_ids": [
      [5, 0, 0],
      [0, 0, {
        "appointment_resource_id": 49,
        "capacity_reserved": 4
      }]
    ]
  }
}
```
- `[5, 0, 0]` = eliminar todas las líneas existentes (comando ORM "delete all")
- `[0, 0, {...}]` = crear nueva línea con la nueva mesa

---

### 8. Obtener Mesas Disponibles (por Floor)

**`POST /json/2/restaurant.table/search_read`**

**Request Body:**
```json
{
  "domain": [["floor_id", "=", 17]],
  "fields": ["id", "table_number", "appointment_resource_id", "seats", "floor_id"]
}
```

**Response:**
```json
[
  {"id": 52, "table_number": 1, "appointment_resource_id": 40, "seats": 2, "floor_id": 17},
  {"id": 53, "table_number": 2, "appointment_resource_id": 41, "seats": 2, "floor_id": 17}
]
```

---

### 9. Obtener Floors Disponibles

**`POST /json/2/restaurant.floor/search_read`**

**Request Body:**
```json
{
  "domain": [],
  "fields": ["id", "name", "pos_config_ids"]
}
```

---

### 10. Leer Detalle de una Reservación

**`POST /json/2/calendar.event/read`**

**Request Body:**
```json
{
  "ids": [5],
  "fields": ["id", "name", "start", "stop", "duration", "phone_number", "appointment_status", "appointment_type_id", "appointment_resource_ids", "booking_line_ids", "partner_ids", "description"]
}
```

---

## Tabla de Mesas - Salón Principal (Floor ID: 17)

| Mesa # | table_id | resource_id | Capacidad |
|--------|----------|-------------|-----------|
| 1 | 52 | 40 | 2 |
| 2 | 53 | 41 | 2 |
| 3 | 54 | 42 | 2 |
| 4 | 55 | 43 | 2 |
| 5 | 56 | 44 | 2 |
| 6 | 57 | 45 | 2 |
| 7 | 58 | 46 | 2 |
| 8 | 59 | 47 | 2 |
| 9 | 60 | 48 | 2 |
| 10 | 61 | 49 | 2 |
| 11 | 63 | 51 | 2 |
| 12 | 64 | 52 | 2 |
| 13 | 65 | 53 | 2 |
| 14 | 66 | 54 | 2 |
| 15 | 67 | 55 | 2 |
| 16 | 68 | 56 | 2 |

> **Nota**: Para crear una reservación se usa `resource_id` (NO `table_id`) en `booking_line_ids.appointment_resource_id`.

---

## Operadores de Dominio Útiles

| Operador | Ejemplo | Descripción |
|----------|---------|-------------|
| `=` | `["phone_number", "=", "+57311"]` | Exacto |
| `!=` | `["appointment_status", "!=", "cancelled"]` | Diferente |
| `ilike` | `["phone_number", "ilike", "311"]` | Contiene (case insensitive) |
| `>=` | `["start", ">=", "2026-02-15 00:00:00"]` | Mayor o igual |
| `<=` | `["start", "<=", "2026-02-15 23:59:59"]` | Menor o igual |
| `in` | `["appointment_status", "in", ["booked", "attended"]]` | En lista |
| `not in` | `["appointment_status", "not in", ["cancelled"]]` | No en lista |

---

## Valores de appointment_status

| Valor | Significado |
|-------|-------------|
| `"booked"` | Reservado (activo) |
| `"attended"` | Checked-in / Asistió |
| `"no_show"` | No se presentó |
| `"cancelled"` | Cancelado (inactivo) |
| `"request"` | Solicitud pendiente |

---

## Comandos ORM para campos One2many/Many2many

Usados en `booking_line_ids`, `partner_ids`, etc:

| Comando | Formato | Descripción |
|---------|---------|-------------|
| CREATE | `[0, 0, {vals}]` | Crear nuevo registro vinculado |
| UPDATE | `[1, ID, {vals}]` | Actualizar registro existente |
| DELETE | `[2, ID, 0]` | Eliminar registro |
| LINK | `[4, ID, 0]` | Vincular registro existente |
| UNLINK | `[3, ID, 0]` | Desvincular sin eliminar |
| REPLACE | `[6, 0, [IDs]]` | Reemplazar todos los vínculos |
| CLEAR | `[5, 0, 0]` | Eliminar todos los vínculos |

---

## Autenticación - Generar API Key

### Opción 1: Odoo Shell
```bash
source /opt/odoo/odoo-venv/bin/activate
sudo -u odoo /opt/odoo/odoo-venv/bin/python /opt/odoo/odoo-server/odoo-bin shell \
  -d cocson_2026 -c /etc/odoo/odoo.conf --no-http
```
```python
key = env['res.users.apikeys'].with_user(2)._generate(scope='rpc', name='MCP Reservaciones')
print(f"API Key: {key}")
env.cr.commit()
# Guardar el key, no se puede recuperar después (se guarda hasheado)
```

### Opción 2: Backend Odoo
Ajustes → Usuarios → (seleccionar usuario) → Pestaña "Claves API" → "Nueva Clave API"

---

## Comportamiento de Reservas Futuras (sin sesión POS abierta)

### Crear reserva para otro día: FUNCIONA sin problema

La reserva se guarda en `calendar.event` directamente en BD. **No requiere sesión POS abierta.**

### Qué pasa con las notificaciones

| Escenario | Notificación al POS |
|-----------|-------------------|
| Reserva para **hoy** + sesión abierta | SI - notificación `TABLE_BOOKING` → `ADDED` en tiempo real |
| Reserva para **hoy** + sesión cerrada | NO - se carga cuando abran sesión |
| Reserva para **mañana/futuro** | NO - filtrado en línea 17: `if event.start.date() != today: continue` |
| Modificar reserva de **hoy** | SI - envía `REMOVED` + `ADDED` |
| Cancelar reserva de **hoy** | SI - envía `REMOVED` |

### Cómo aparece en el POS al día siguiente

Cuando el POS abre sesión, carga reservas del día vía `_load_pos_data_domain`:
```python
# Carga eventos donde: (start >= ahora AND start <= mañana) OR (stop >= ahora AND stop <= mañana)
now = fields.Datetime.now()
day_after = fields.Date.today() + timedelta(days=1)
```

Es decir: **las reservas futuras aparecen automáticamente cuando abren sesión ese día.**

### Flujo típico de reserva futura vía API

```
1. Cliente llama/escribe por WhatsApp → "Quiero reservar mesa para el viernes 7pm, 4 personas"
2. MCP llama POST /json/2/calendar.event/create con start="2026-02-20 00:00:00" (7PM COT = 00:00 UTC+1)
3. calendar.event se guarda en BD → NO hay notificación al POS (es día futuro)
4. El viernes, el restaurante abre sesión POS
5. POS carga datos → _load_pos_data_domain trae la reserva → aparece en pantalla de mesas
```

---

## Notas para el MCP

1. **Notificaciones en tiempo real**: Cada create/write/unlink en `calendar.event` con `appointment_type_id` envía automáticamente notificación `TABLE_BOOKING` al POS frontend vía bus. **Solo para reservas del día de hoy y con sesión abierta.**

2. **Timezone**: Las fechas `start`/`stop` se almacenan en **UTC**. Colombia es UTC-5, así que `7:00 PM COT` = `00:00:00 UTC del día siguiente`. Ejemplo: viernes 7PM Colombia = `"2026-02-21 00:00:00"` UTC.

3. **Reservaciones canceladas**: `set_cancelled` pone `active = false`. Para buscar canceladas hay que agregar `["active", "=", false]` al domain (por defecto solo se retornan activos).

4. **Modelo principal**: `calendar.event` — este es el único modelo que necesitas para CRUD de reservaciones. Las `booking_line_ids` se crean inline con comandos ORM.

5. **resource_id vs table_id**: Siempre usar `appointment_resource_id` del recurso, nunca el `id` de la mesa directamente. Obtener el mapeo con endpoint #8.

6. **Reservas futuras**: Se pueden crear sin problema, no requieren sesión POS abierta. Se cargan automáticamente cuando el POS abre sesión ese día.

7. **Actualizar reserva (write)**: Envía primero `REMOVED` y luego `ADDED` al POS, así la mesa se actualiza visualmente en tiempo real (solo si es reserva de hoy con sesión abierta).
