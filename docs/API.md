# PetGuardian — Referencia de la API

API REST en JSON servida por el backend Express bajo el prefijo `/api`.

- **Base de desarrollo:** `http://localhost:3000`
- **En el frontend:** el proxy de Vite redirige `/api` al backend, por lo que el
  cliente usa rutas relativas (`/api/...`).
- **Autenticación:** cookie de sesión `pg_session` (por defecto), `HttpOnly` y
  `SameSite=Lax`. El navegador la envía automáticamente con
  `credentials: "include"`.
- **Formato:** todas las respuestas son `application/json`. Los cuerpos de
  solicitud con datos se envían como JSON (límite `6mb`, por la foto en base64).

---

## 1. Convenciones

### 1.1 Envoltorios de respuesta

Cada recurso se devuelve bajo una clave con su nombre:

| Recurso | Clave |
| --- | --- |
| Mascota(s) | `pet` / `pets` |
| Solicitud(es) de adopción | `adoption` / `adoptions` |
| Evento(s) | `events` |
| Registro(s) clínico(s) | `record` / `records` (+ `type`) |
| Expediente clínico | `profile` / `entries` / `tratamientosVigentes`+`controlesProximos` |
| Agenda / calendario clínico | `agenda` / `calendar` |
| Vínculo veterinario | `veterinarians` / `assignment` |
| Usuario(s) | `user` / `users` |
| Veterinarios | `veterinarians` |
| Roles públicos | `roles` |
| Panel | `dashboard` |
| Entidad(es) / asignación vet | `entities` / `assignment` |
| Inventario: item/lote/movimiento | `item`/`items`, `batch`/`batches`, `movement`/`movements` |
| Alertas de inventario | `lowStock` / `expiringSoon` / `expired` |
| Sala(s) / reserva(s) | `room`/`rooms` / `reservation`/`reservations` |
| Equipo(s) / mantenimiento | `equipment` / `record` |

Ejemplo: `GET /api/pets` → `{ "pets": [ ... ] }`.

### 1.2 Errores

Formato uniforme (`middlewares/errorHandler.js`):

```json
{
  "error": "codigo_maquina",
  "message": "Mensaje legible en español.",
  "details": { "campo": "descripción del problema" }
}
```

`details` solo está presente cuando existe (habitualmente en `400`).

### 1.3 Códigos de estado

| Estado | Significado | Códigos típicos |
| --- | --- | --- |
| `200` | OK | lectura/actualización |
| `201` | Creado | alta de recursos |
| `400` | Validación | `validation_error`, `invalid_id` |
| `401` | Sin sesión | `unauthorized`, `invalid_credentials` |
| `403` | Sin permiso | `forbidden` |
| `404` | No encontrado | `pet_not_found`, `adoption_not_found`, `record_not_found`, `medical_type_not_found`, `not_found` |
| `409` | Conflicto | `email_taken`, `possible_duplicate`, `duplicate_request`, `pet_not_available`, `invalid_status_transition`, `procedure_not_completed`, `invalid_performed_date`, `invalid_follow_up_date`, `appointment_in_past`, `schedule_conflict`, `consultation_closed` |
| `500` | Error interno | `internal_error` |

---

## 2. Salud

### `GET /api/health`

Público. Parámetro opcional `verbose`.

- `verbose` ausente o `false` → respuesta resumida.
- `verbose=true` → informe completo.
- Cualquier otro valor → `400`.

**Resumida**

```json
{
  "status": "ok",
  "service": "petguardian-backend",
  "version": "0.1.0",
  "timestamp": "2026-01-01T00:00:00.000Z",
  "database": true
}
```

**Completa** (`?verbose=true`)

```json
{
  "status": "ok",
  "service": "petguardian-backend",
  "version": "0.1.0",
  "timestamp": "2026-01-01T00:00:00.000Z",
  "uptimeSeconds": 12.34,
  "database": { "connected": true, "schemaVersion": "8" }
}
```

`status` es `ok` si la BD responde, o `degraded` en caso contrario.

---

## 3. Autenticación

### `POST /api/auth/register`

Público. Crea una cuenta (no admin) e inicia sesión emitiendo la cookie.

**Cuerpo**

```json
{
  "email": "ana@example.com",
  "password": "contrasena-segura",
  "fullName": "Ana Torres",
  "role": "adoptante"
}
```

**Reglas:** email válido y único; `password` ≥ 8 caracteres; `fullName` ≥ 3;
`role` ∈ `{adoptante, fundacion, veterinario}` (por defecto `adoptante`).

**Respuesta `201`**

```json
{ "user": { "id": 1, "email": "ana@example.com", "fullName": "Ana Torres", "role": "adoptante", "createdAt": "..." } }
```

**Errores:** `400 validation_error` (con `details`), `409 email_taken`.

### `POST /api/auth/login`

Público.

**Cuerpo:** `{ "email": "...", "password": "..." }`

**Respuesta `200`:** `{ "user": { ... } }` + `Set-Cookie: pg_session=...`

**Errores:** `400 validation_error`, `401 invalid_credentials`.

### `POST /api/auth/logout`

Requiere sesión. Invalida el token y limpia la cookie.

**Respuesta `200`:** `{ "ok": true }`

### `GET /api/auth/me`

Requiere sesión.

**Respuesta `200`:** `{ "user": { ... } }`

**Errores:** `401 unauthorized`.

### `GET /api/auth/roles`

Público. Roles disponibles para el registro público.

**Respuesta `200`**

```json
{ "roles": [ { "value": "adoptante", "label": "Adoptante" }, { "value": "fundacion", "label": "Fundacion" }, { "value": "veterinario", "label": "Veterinario aliado" } ] }
```

---

## 4. Usuarios

### `GET /api/users`

Solo `admin`. Lista de usuarios (sin `passwordHash`).

**Respuesta `200`:** `{ "users": [ { "id", "email", "fullName", "role", "createdAt" } ] }`

**Errores:** `401`, `403`.

### `GET /api/users/veterinarians`

Roles `admin`, `fundacion`, `veterinario` (`MEDICAL_READ_ROLES`). Directorio
reducido para asignar veterinario responsable.

**Respuesta `200`:** `{ "veterinarians": [ { "id": 3, "fullName": "Dra. Valentina Rios" } ] }`

> Solo expone `id` y `fullName`, nunca email ni otros datos.

**Errores:** `401`, `403`.

---

## 5. Mascotas

### `GET /api/pets`

Cualquier usuario autenticado. Filtros por query string.

| Parámetro | Valores | Descripción |
| --- | --- | --- |
| `search` | texto (≤ 80) | Búsqueda por nombre (case-insensitive) |
| `species` | `perro`, `gato` | Filtro por especie |
| `status` | `disponible`, `en_proceso`, `adoptada`, `bajo_cuidado` | Filtro por estado |

Orden: más recientes primero (`created_at DESC, id DESC`).

**Respuesta `200`**

```json
{
  "pets": [
    {
      "id": 1, "name": "Luna", "species": "gato", "breed": "Comun europeo",
      "sex": "hembra", "birthDate": "2021-05-10", "approximateAgeMonths": null,
      "ageMonths": 55, "ageLabel": "4 años", "weightKg": 3.4, "color": "Atigrado",
      "description": "...", "photoUrl": null, "status": "bajo_cuidado",
      "ownerId": 2, "ownerName": "Fundacion Patitas Demo", "entityId": 2,
      "createdAt": "2026-...", "updatedAt": "2026-..."
    }
  ]
}
```

**Errores:** `400 validation_error` (filtro inválido), `401`.

> El frontend, para el rol `adoptante`, filtra localmente a `disponible` y
> `en_proceso`; la API no oculta otros estados en el listado.

### `GET /api/pets/:id`

Cualquier autenticado.

**Respuesta `200`:** `{ "pet": { ... } }`
**Errores:** `400 invalid_id`, `404 pet_not_found`, `401`.

### `POST /api/pets`

Roles `admin`, `fundacion` (`PET_WRITE_ROLES`).

**Cuerpo**

```json
{
  "name": "Firulais", "species": "perro", "breed": "Mestizo", "sex": "macho",
  "birthDate": "2022-03-01", "approximateAgeMonths": null, "weightKg": 12.5,
  "color": "Marron", "description": "Muy jugueton.", "status": "disponible",
  "photoUrl": "data:image/png;base64,..."
}
```

- `name`: 2–80 caracteres. `species`: `perro`|`gato`. `sex`: `macho`|`hembra`|`desconocido`.
- `birthDate`: `AAAA-MM-DD`, no futura. `approximateAgeMonths`: entero 0–600.
- `weightKg`: 0.01–200. `status`: estado válido (por defecto `disponible`).
- `photoUrl`: `http(s)://...` (≤ 2048) o data URL `image/png|jpeg|webp`
  (≤ 2 MB, con verificación de bytes mágicos).

**Respuesta `201`:** `{ "pet": { ... } }`
**Errores:** `400 validation_error`, `403 forbidden`, `409 possible_duplicate`.

### `PUT /api/pets/:id`

Dueño o `admin` (`PET_WRITE_ROLES` + propiedad en el servicio). Mismos campos
que el alta. Si se omite `status`, se conserva el actual.

**Respuesta `200`:** `{ "pet": { ... } }`
**Errores:** `400`, `403`, `404`, `409 invalid_status_transition`.

### `DELETE /api/pets/:id`

Dueño o `admin`.

**Respuesta `200`:** `{ "id": 1 }`
**Errores:** `400 invalid_id`, `403`, `404`.

---

## 6. Salud veterinaria (expediente clínico)

Rutas anidadas: `/api/pets/:petId/<tipo>[/:recordId]`.

`type` ∈ `vaccinations`, `procedures`, `appointments`, `consultations`,
`deworming`, `treatments`. Un `type` desconocido responde `404
medical_type_not_found`.

**Acceso:** lectura y escritura para `MEDICAL_READ_ROLES` / `MEDICAL_WRITE_ROLES`
(`admin`, `fundacion`, `veterinario`), con regla de propiedad para `fundacion`.
El `adoptante` recibe `403`. La restricción de lectura también se aplica en el
servicio (no solo por rol de ruta): el `veterinario` solo accede al expediente de
sus **pacientes asignados** (`vet_pet_assignments.is_active = 1`).

### 6.1 Expediente: perfil clínico, auditoría y controles

Rutas específicas del expediente (declaradas **antes** de `/:petId/:type`):

| Método | Ruta | Descripción |
| --- | --- | --- |
| GET | `/api/pets/:petId/clinical-profile` | Antecedentes/alergias/enfermedades previas |
| PUT | `/api/pets/:petId/clinical-profile` | Crea/actualiza antecedentes (auditado) |
| GET | `/api/pets/:petId/clinical-audit` | Bitácora clínica del paciente |
| GET | `/api/pets/:petId/clinical-controls` | Tratamientos vigentes + próximos controles |

**`GET .../clinical-profile`** → `{ "profile": { petId, allergies, chronicConditions, clinicalHistory, notes, updatedBy, createdAt, updatedAt } }` (o `null`).

**`PUT .../clinical-profile`** — cuerpo con `allergies`, `chronicConditions`,
`clinicalHistory`, `notes` (texto ≤ 2000). Devuelve `{ "profile": { ... } }` y
registra `profile_updated` en la auditoría.

**`GET .../clinical-audit`** → `{ "entries": [ { id, recordType, recordId, action,
actionLabel, actorId, actorName, changes, createdAt } ] }`. Acciones: `created`,
`updated`, `closed`, `reopened`, `voided`, `profile_updated`.

**`GET .../clinical-controls`** → `{ "tratamientosVigentes": [...], "controlesProximos": [ { kind: 'appointment'|'procedure', id, date, title, status } ] }`. Los controles próximos combinan citas programadas futuras y los `followUpDate` de procedimientos, ordenados por fecha.

> Las lecturas de estas rutas usan `MEDICAL_READ_ROLES`; la escritura del perfil
> usa `MEDICAL_WRITE_ROLES`. El `adoptante` recibe `403`.

### `GET /api/pets/:petId/:type`

**Respuesta `200`:** `{ "type": "vaccinations", "records": [ ... ] }`

Cada registro incluye `id`, `petId`, campos del tipo, `veterinarianId`,
`veterinarianName`, `createdBy`, `createdAt`, `updatedAt` y, cuando aplica,
`status` + `statusLabel` (calculado o almacenado).

### `GET /api/pets/:petId/:type/:recordId`

**Respuesta `200`:** `{ "type": "...", "record": { ... } }`
**Errores:** `400 invalid_id`, `403`, `404 record_not_found`.

### `POST /api/pets/:petId/:type`

**Cuerpo por tipo**

| Tipo | Campos |
| --- | --- |
| `vaccinations` | `vaccineName`\*, `appliedDate`\*, `nextDoseDate`, `veterinarianId`, `batchNumber`, `notes` |
| `deworming` | `productName`\*, `dewormingType` (`interna`/`externa`/`mixta`), `appliedDate`\*, `nextDoseDate`, `veterinarianId`, `notes` |
| `procedures` | `procedureType`\*, `status` (`pendiente`/`programado`/`realizado`/`cancelado`), `scheduledDate`, `performedDate`, `preOpInstructions`, `postOpNotes`, `followUpDate`, `veterinarianId`, `notes` |
| `appointments` | `reason`\*, `scheduledAt`\* (`AAAA-MM-DD HH:MM`), `status` (`programada`/`completada`/`cancelada`), `veterinarianId`, `notes` |
| `consultations` | `reason`\*, `consultationDate`\*, `status` (`abierta`/`cerrada`), `weightKg`, `temperatureC`, `heartRateBpm`, `respiratoryRateRpm`, `examination`, `findings`, `diagnosis`, `treatment`, `evolution`, `veterinarianId`, `notes` |
| `treatments` | `medication`\*, `startDate`\*, `dose`, `route`, `frequency`, `duration`, `endDate`, `instructions`, `veterinarianId`, `notes` |

`*` = obligatorio. Fechas de aplicación/consulta (`appliedDate`,
`consultationDate`) **no** pueden ser futuras. `veterinarianId`, si se indica,
debe existir **y tener rol `veterinario`** (`400`). Los signos vitales se validan
por rango (`weightKg` 0.1–200; `temperatureC` 25–45; `heartRateBpm` 20–400 entero;
`respiratoryRateRpm` 5–200 entero). Al crear/editar citas se valida que no
colisione el mismo horario con otra cita activa del **mismo paciente o del mismo
veterinario**; una cita `programada` no puede quedar en fecha pasada.

**Respuesta `201`:** `{ "type": "...", "record": { ... } }`

### `PUT /api/pets/:petId/:type/:recordId`

Actualización parcial: solo se modifican los campos enviados. Aplica:

- `procedures`: transición de estado e integridad (`followUpDate` no anterior a
  `performedDate`).
- `appointments`: transición de estado, no-fecha-pasada (solo si se envía
  `scheduledAt`) y conflicto de horario.
- `consultations`: una consulta **cerrada** es inmutable; editarla sin reabrir →
  `409 consultation_closed`. La reapertura explícita (`{ "status": "abierta" }`
  sin otros campos) queda auditada (`reopened`).

**Respuesta `200`:** `{ "type": "...", "record": { ... } }`
**Errores:** `400`, `403`, `404`, `409 invalid_status_transition`,
`409 procedure_not_completed`, `409 invalid_performed_date`,
`409 invalid_follow_up_date`, `409 appointment_in_past`, `409 schedule_conflict`,
`409 consultation_closed`.

### `DELETE /api/pets/:petId/:type/:recordId`

**Anulación lógica** (auditada): marca el registro como anulado, nunca lo borra
físicamente.

**Respuesta `200`:** `{ "id": 17, "voided": true }`
**Errores:** `400`, `403`, `404`.

---

### 6.2 Vínculo veterinario ↔ mascota (asignaciones)

| Método | Ruta | Acceso |
| --- | --- | --- |
| GET | `/api/pets/:petId/veterinarians` | admin, fundación propietaria, veterinario asignado |
| POST | `/api/pets/:petId/veterinarians` | admin, fundación propietaria |
| DELETE | `/api/pets/:petId/veterinarians/:veterinarianId` | admin, fundación propietaria |

**GET** → `{ "petId": 7, "veterinarians": [ { id, fullName, assignedAt } ] }`.
**POST** — cuerpo `{ "veterinarianId": 3 }` (debe existir y tener rol
`veterinario`); devuelve `201 { assignment: { petId, id, fullName, assignedAt } }`.
**DELETE** → `200 { petId, veterinarianId, removed }` (desasignación lógica).

**Errores:** `400` (veterinario inválido), `403`, `404 pet_not_found`.

### 6.3 Agenda y calendario clínico

Solo `admin` y `veterinario` (`authorize(admin, veterinario)`). El veterinario se
acota en el servicio a sus pacientes asignados.

### `GET /api/clinical/agenda`

**Respuesta `200`:** `{ "agenda": { citas, procedimientos, consultasRecientes, tratamientosVigentes, controlesProximos } }`.

### `GET /api/clinical/appointments`

Bandeja de consultas (citas) del veterinario/admin. El veterinario se acota a sus
pacientes asignados; el admin ve todas. Filtros opcionales por query string:

| Parametro | Valores |
| --- | --- |
| `status` | `programada`, `completada`, `cancelada` |
| `from` / `to` | fecha `AAAA-MM-DD` (rango inclusivo) |
| `petId` | identificador numerico del paciente |
| `search` | texto libre (motivo o nombre del paciente) |

**Respuesta `200`:** `{ "appointments": [ { id, petId, petName, species, photoUrl,
reason, status, scheduledAt, veterinarianId, veterinarianName } ] }`.
**Errores:** `400 validation_error` (filtros invalidos), `401`, `403`.

### `GET /api/clinical/calendar?from=AAAA-MM-DD&to=AAAA-MM-DD`

Rango inclusivo (parámetros obligatorios).

**Respuesta `200`:** `{ "calendar": { "from", "to", "citas", "procedimientos" } }`.
**Errores:** `400 validation_error` (rango inválido), `401`, `403`.

---

## 7. Adopciones

### `GET /api/adoptions`

Roles `admin`, `fundacion`, `adoptante` (`ADOPTION_READ_ROLES`). El `veterinario`
recibe `403`. El alcance depende del rol:

- `adoptante` → solo sus solicitudes (`applicant_id`).
- `fundacion` → solo solicitudes de sus mascotas (`pets.owner_id`).
- `admin` → todas.

| Parámetro | Valores |
| --- | --- |
| `status` | `pendiente`, `aprobada`, `rechazada`, `cancelada` |
| `petId` | entero > 0 |

**Respuesta `200`:** `{ "adoptions": [ { ... } ] }`

### `POST /api/adoptions`

Solo `adoptante`. La mascota no puede estar `adoptada` y no puede existir otra
solicitud pendiente del mismo solicitante para esa mascota.

**Cuerpo:** `{ "petId": 4, "message": "Vivo en un piso con patio..." }`
(`message` ≤ 500, opcional).

**Respuesta `201`:** `{ "adoption": { ... } }` con `status: "pendiente"`. Si la
mascota estaba `disponible`, pasa a `en_proceso`.

**Errores:** `400 validation_error`, `403`, `404 pet_not_found`,
`409 pet_not_available`, `409 duplicate_request`.

### `GET /api/adoptions/:id`

Solicitante, fundación dueña o `admin`.

**Respuesta `200`:** `{ "adoption": { ... } }`
**Errores:** `400`, `403`, `404 adoption_not_found`.

### `GET /api/adoptions/:id/events`

Mismo acceso. Historial (`created`, `edited`, `approved`, `rejected`,
`cancelled`).

**Respuesta `200`:** `{ "events": [ { "id", "type", "typeLabel", "fromStatus", "fromStatusLabel", "toStatus", "toStatusLabel", "actorId", "actorName", "note", "createdAt" } ] }`

### `PUT /api/adoptions/:id`

Solicitante o `admin`, solo si la solicitud está `pendiente`. Edita
`message`.

**Cuerpo:** `{ "message": "..." }`
**Respuesta `200`:** `{ "adoption": { ... } }`
**Errores:** `400`, `403`, `409 invalid_status_transition` (no pendiente).

### `POST /api/adoptions/:id/approve`

Fundación dueña o `admin` (`ADOPTION_DECISION_ROLES` + propiedad). Transacción
`BEGIN IMMEDIATE`.

**Cuerpo:** `{ "note": "..." }` (opcional, ≤ 500).

**Respuesta `200`:** `{ "adoption": { ... status: "aprobada" } }`. La mascota
pasa a `adoptada` y las demás pendientes se rechazan automáticamente.

**Errores:** `403`, `404`, `409 pet_not_available`, `409 invalid_status_transition`.

### `POST /api/adoptions/:id/reject`

Fundación dueña o `admin`.

**Cuerpo:** `{ "note": "..." }` (opcional).

**Respuesta `200`:** `{ "adoption": { ... status: "rechazada" } }`. Libera la
mascota a `disponible` si corresponde.

### `POST /api/adoptions/:id/cancel`

Solicitante o `admin`, solo si `pendiente`.

**Respuesta `200`:** `{ "adoption": { ... status: "cancelada" } }`. Libera la
mascota a `disponible` si corresponde.

**Errores transversales de decisión:** `403`, `404`, `409 invalid_status_transition`.

---

## 8. Panel

### `GET /api/dashboard`

Cualquier usuario autenticado. El contenido depende del rol.

**Respuesta `200`**

```json
{
  "dashboard": {
    "pets": {
      "total": 6, "perros": 3, "gatos": 3,
      "disponibles": 3, "enProceso": 1, "adoptadas": 1, "bajoCuidado": 1
    },
    "adoptions": { "pendientes": 1, "aprobadas": 0, "total": 1 },
    "medical": {
      "vacunasProximas": 1, "vacunasVencidas": 1,
      "desparasitacionesProximas": 1, "desparasitacionesVencidas": 1,
      "cirugiasPendientes": 2, "proximasCitas": 3
    },
    "attention": [
      {
        "id": 1, "name": "Luna", "species": "gato", "photoUrl": null,
        "status": "bajo_cuidado",
        "vacunasVencidas": 1, "vacunasProximas": 1,
        "desparasitacionesVencidas": 1, "desparasitacionesProximas": 1,
        "cirugiasPendientes": 1, "citasProximas": 1,
        "proximaFecha": "2026-...", "vencidaFecha": "2026-..."
      }
    ]
  }
}
```

**Presencia de secciones por rol**

| Sección | admin | fundacion | veterinario | adoptante |
| --- | :---: | :---: | :---: | :---: |
| `pets` | ✅ (global) | ✅ (propias) | ✅ (asignadas) | ✅ (global) |
| `adoptions` | ✅ (global) | ✅ (propias) | — | ✅ (propias) |
| `medical` + `attention` | ✅ (global) | ✅ (propias) | ✅ (asignados) | — |

`attention` se limita a **6** mascotas y solo incluye las que tienen al menos una
alerta.

---

## 9. Entidades aliadas (v8)

Introducidas en la migración **v8** (`entities`, `entity_users`,
`vet_entity_assignments`). Una entidad representa una **fundación** o **clínica**.
Cada cuenta de fundación posee una entidad (creada de forma atómica en el
registro); las mascotas de esa fundación quedan vinculadas a su `entity_id`. Las
mascotas con dueño `admin`, `adoptante` u otro rol permanecen con `entity_id`
nulo.

Las entidades **no se crean por API**: se provisionan al registrar la cuenta de
fundación. Estos endpoints listan entidades y gestionan las asignaciones
veterinario ↔ entidad.

| Método | Ruta | Acceso |
| --- | --- | --- |
| GET | `/api/entities` | admin (todas), fundacion y veterinario (sus entidades) |
| GET | `/api/entities/:entityId/veterinarians` | admin, fundacion (propia), veterinario (asignado) |
| POST | `/api/entities/:entityId/veterinarians` | admin, fundacion (propia) |
| DELETE | `/api/entities/:entityId/veterinarians/:veterinarianId` | admin, fundacion (propia) |

### `GET /api/entities`

**Respuesta `200`**

```json
{
  "entities": [
    {
      "id": 1, "name": "Fundacion Patitas", "kind": "fundacion",
      "taxId": null, "contactEmail": "contacto@patitas.org",
      "phone": null, "address": null, "isActive": true,
      "originUserId": 3, "createdBy": null,
      "createdAt": "2026-...", "updatedAt": "2026-..."
    }
  ]
}
```

### `GET /api/entities/:entityId/veterinarians`

Devuelve las asignaciones **activas** de la entidad.

```json
{
  "veterinarians": [
    {
      "id": 7, "entityId": 1, "veterinarianId": 7,
      "veterinarianName": "Dra. Valentina Rios",
      "veterinarianEmail": "vet@example.test",
      "isActive": true, "createdAt": "2026-..."
    }
  ]
}
```

### `POST /api/entities/:entityId/veterinarians`

Cuerpo: `{ "veterinarianId": 7 }`. Reactiva una asignación previa si existía.
Valida que el usuario destino tenga rol `veterinario` (si no, `400`).

**Respuesta `201`**: `{ "assignment": { "entityId": 1, "veterinarianId": 7, "isActive": true } }`

### `DELETE /api/entities/:entityId/veterinarians/:veterinarianId`

Marca la asignación como inactiva (`is_active = 0`). `404` si no existía o ya
estaba inactiva.

**Respuesta `200`**: `{ "assignment": { "entityId": 1, "veterinarianId": 7, "isActive": false } }`

---

## 10. Inventario (v8)

Inventario de medicamentos, vacunas e insumos por entidad. Unidades
controladas: `unidad`, `ml`, `mg`, `g`, `dosis` (sin conversiones automáticas).
Categorías: `medicamento`, `vacuna`, `insumo`.

Capacidades: **lectura** y **consumo** para `admin`, `fundacion` y `veterinario`
(el vet solo en sus entidades asignadas); **administración** del catálogo, lotes
y movimientos de ajuste para `admin` y `fundacion`.

| Método | Ruta | Acceso |
| --- | --- | --- |
| GET | `/api/inventory/items` | admin, fundacion, veterinario |
| GET | `/api/inventory/items/:itemId` | admin, fundacion, veterinario |
| POST | `/api/inventory/items` | admin, fundacion |
| PUT | `/api/inventory/items/:itemId` | admin, fundacion |
| DELETE | `/api/inventory/items/:itemId` | admin, fundacion |
| GET | `/api/inventory/batches` | admin, fundacion, veterinario |
| POST | `/api/inventory/items/:itemId/batches` | admin, fundacion |
| PUT | `/api/inventory/batches/:batchId` | admin, fundacion |
| DELETE | `/api/inventory/batches/:batchId` | admin, fundacion |
| GET | `/api/inventory/movements` | admin, fundacion, veterinario |
| POST | `/api/inventory/movements` | admin, fundacion, veterinario |
| GET | `/api/inventory/alerts` | admin, fundacion, veterinario |

Filtros de consulta:

- `GET /items`: `?search=&category=&includeInactive=true`
- `GET /batches`: `?itemId=&includeEmpty=true`
- `GET /movements`: `?itemId=&batchId=&petId=&limit=` (límite por defecto 100, máximo 500)

### `POST /api/inventory/items`

Cuerpo:

```json
{
  "entityId": 1, "name": "Amoxicilina 500mg", "category": "medicamento",
  "unit": "mg", "description": null, "minStock": 10, "isActive": true
}
```

**Respuesta `201`**: `{ "item": { "id", "entityId", "entityName", "name", "category", "unit", "minStock", "isActive", "createdAt", "updatedAt" } }`

### `POST /api/inventory/items/:itemId/batches`

Crea un lote y registra automáticamente el movimiento de **entrada**.

```json
{
  "batchNumber": "L-2026-01", "expiresOn": "2027-06-30",
  "quantityReceived": 50, "receivedOn": "2026-05-01",
  "supplier": "Provet", "notes": null
}
```

**Respuesta `201`**: `{ "batch": {...}, "movement": {...} }`

### `POST /api/inventory/movements`

Cuerpo: `{ "batchId": 3, "movementType": "consumo", "quantity": 2, "petId": 5, "reason": null }`

Tipos de movimiento: `entrada`, `consumo`, `ajuste_positivo`, `ajuste_negativo`,
`merma`, `devolucion`. Los tipos de **ajuste** y **merma** están restringidos a
`admin`/`fundacion` (el veterinario recibe `403`). No se permiten existencias
negativas salvo *override* explícito (`overrideReason`) autorizado por
administración.

**Respuesta `201`**: `{ "movement": {...} }`

### `GET /api/inventory/alerts`

**Respuesta `200`**

```json
{
  "lowStock": [ /* items con existencias <= min_stock */ ],
  "expiringSoon": [ /* lotes que vencen en los próximos 60 días */ ],
  "expired": [ /* lotes ya vencidos */ ]
}
```

Ventanas configurables: vencimiento **60 días**, mantenimiento **30 días**.

---

## 11. Salas de operación y reservas (v8)

| Método | Ruta | Acceso |
| --- | --- | --- |
| GET | `/api/rooms` | admin, fundacion, veterinario |
| POST | `/api/rooms` | admin, fundacion |
| GET | `/api/rooms/:roomId` | admin, fundacion, veterinario |
| PUT | `/api/rooms/:roomId` | admin, fundacion |
| DELETE | `/api/rooms/:roomId` | admin, fundacion |
| GET | `/api/rooms/reservations` | admin, fundacion, veterinario |
| POST | `/api/rooms/:roomId/reservations` | admin, fundacion, veterinario |
| PUT | `/api/rooms/reservations/:reservationId` | admin, fundacion, veterinario |
| DELETE | `/api/rooms/reservations/:reservationId` | admin, fundacion, veterinario |
| GET | `/api/rooms/:roomId/equipment` | admin, fundacion, veterinario |
| POST | `/api/rooms/:roomId/equipment` | admin, fundacion |
| DELETE | `/api/rooms/:roomId/equipment/:equipmentId` | admin, fundacion |

Estados de sala: `disponible`, `mantenimiento`, `fuera_de_servicio`.
Estados de reserva y transiciones válidas:

- `programada` → `en_curso` | `cancelada`
- `en_curso` → `finalizada` | `cancelada`
- `finalizada` y `cancelada` son terminales.

Se rechaza el **solapamiento** de reservas activas en la misma sala (`409`); las
reservas contiguas (fin == inicio) sí se permiten.

### `POST /api/rooms`

```json
{ "entityId": 1, "name": "Quirófano A", "location": "Planta 1", "capacity": 2, "status": "disponible", "notes": null }
```

**Respuesta `201`**: `{ "room": { "id", "entityId", "entityName", "name", "location", "capacity", "status", "notes", "createdAt", "updatedAt" } }`

### `POST /api/rooms/:roomId/reservations`

```json
{ "startsAt": "2026-06-01 09:00", "endsAt": "2026-06-01 10:30", "petId": 5, "notes": null }
```

`startsAt`/`endsAt` en formato `AAAA-MM-DD HH:MM`. `petId` es opcional; si se
indica, debe pertenecer a la misma entidad (`403`/`400` si no).

**Respuesta `201`**: `{ "reservation": { "id", "roomId", "roomName", "entityId", "petId", "petName", "startsAt", "endsAt", "status", "notes", ... } }`

### `PUT /api/rooms/reservations/:reservationId`

Cuerpo parcial: `{ "status": "en_curso" }` (o `startsAt`, `endsAt`, `notes`,
`petId`, `procedureId`, `veterinarianId`). Se validan las transiciones de estado.

### `POST /api/rooms/:roomId/equipment`

Cuerpo: `{ "equipmentId": 4 }`. Asocia un equipo a la sala (idempotente).

---

## 12. Equipos quirúrgicos y mantenimiento (v8)

| Método | Ruta | Acceso |
| --- | --- | --- |
| GET | `/api/equipment` | admin, fundacion, veterinario |
| POST | `/api/equipment` | admin, fundacion |
| GET | `/api/equipment/:equipmentId` | admin, fundacion, veterinario |
| PUT | `/api/equipment/:equipmentId` | admin, fundacion |
| DELETE | `/api/equipment/:equipmentId` | admin, fundacion |
| GET | `/api/equipment/:equipmentId/maintenance` | admin, fundacion, veterinario |
| POST | `/api/equipment/:equipmentId/maintenance` | admin, fundacion |

Estados de equipo: `operativo`, `en_mantenimiento`, `fuera_de_servicio`, `baja`.
Tipos de mantenimiento: `preventivo`, `correctivo`, `calibracion`, `inspeccion`.

Filtros: `GET /api/equipment?status=&needsMaintenance=true&includeInactive=true`.

### `POST /api/equipment`

```json
{
  "entityId": 1, "name": "Monitor de signos vitales", "category": "monitorizacion",
  "serialNumber": "SN-123", "status": "operativo",
  "acquiredOn": "2025-01-15", "lastMaintenanceOn": null,
  "nextMaintenanceOn": "2026-08-01", "notes": null
}
```

**Respuesta `201`**: `{ "equipment": { "id", "entityId", "entityName", "name", "category", "serialNumber", "status", "acquiredOn", "lastMaintenanceOn", "nextMaintenanceOn", "notes", ... } }`

### `POST /api/equipment/:equipmentId/maintenance`

```json
{
  "performedOn": "2026-05-10", "kind": "preventivo",
  "description": "Calibración anual", "performedBy": "Tec. Gomez",
  "cost": 120.5, "nextDueOn": "2026-11-10"
}
```

**Respuesta `201`**: `{ "record": { "id", "equipmentId", "performedOn", "kind", "description", "cost", "performedBy", "nextDueOn", "recordedBy", "createdAt" }, "equipment": {...} }`

Al registrar mantenimiento se actualiza `last_maintenance_on` del equipo y, si se
envía `nextDueOn`, `next_maintenance_on`.

---

## 13. Resumen de rutas

| Método | Ruta | Acceso |
| --- | --- | --- |
| GET | `/api/health` | Público |
| POST | `/api/auth/register` | Público |
| POST | `/api/auth/login` | Público |
| POST | `/api/auth/logout` | Autenticado |
| GET | `/api/auth/me` | Autenticado |
| GET | `/api/auth/roles` | Público |
| GET | `/api/users` | admin |
| GET | `/api/users/veterinarians` | admin, fundacion, veterinario |
| GET | `/api/dashboard` | Autenticado |
| GET | `/api/pets` | Autenticado |
| GET | `/api/pets/:id` | Autenticado |
| POST | `/api/pets` | admin, fundacion |
| PUT | `/api/pets/:id` | dueño o admin |
| DELETE | `/api/pets/:id` | dueño o admin |
| GET | `/api/pets/:petId/veterinarians` | admin, fundación propietaria, vet asignado |
| POST | `/api/pets/:petId/veterinarians` | admin, fundación propietaria |
| DELETE | `/api/pets/:petId/veterinarians/:veterinarianId` | admin, fundación propietaria |
| GET | `/api/pets/:petId/clinical-profile` | admin, fundacion (propias), veterinario (asignado) |
| PUT | `/api/pets/:petId/clinical-profile` | admin, fundacion (propias), veterinario (asignado) |
| GET | `/api/pets/:petId/clinical-audit` | admin, fundacion (propias), veterinario (asignado) |
| GET | `/api/pets/:petId/clinical-controls` | admin, fundacion (propias), veterinario (asignado) |
| GET | `/api/pets/:petId/:type` | admin, fundacion (propias), veterinario (asignado) |
| GET | `/api/pets/:petId/:type/:recordId` | admin, fundacion (propias), veterinario (asignado) |
| POST | `/api/pets/:petId/:type` | admin, fundacion (propias), veterinario (asignado) |
| PUT | `/api/pets/:petId/:type/:recordId` | admin, fundacion (propias), veterinario (asignado) |
| DELETE | `/api/pets/:petId/:type/:recordId` | admin, fundacion (propias), veterinario (asignado) |
| GET | `/api/clinical/agenda` | admin, veterinario |
| GET | `/api/clinical/appointments` | admin, veterinario |
| GET | `/api/clinical/calendar` | admin, veterinario |
| GET | `/api/entities` | admin, fundacion, veterinario |
| GET | `/api/entities/:entityId/veterinarians` | admin, fundacion, veterinario |
| POST | `/api/entities/:entityId/veterinarians` | admin, fundacion |
| DELETE | `/api/entities/:entityId/veterinarians/:veterinarianId` | admin, fundacion |
| GET | `/api/inventory/items` | admin, fundacion, veterinario |
| GET | `/api/inventory/items/:itemId` | admin, fundacion, veterinario |
| POST | `/api/inventory/items` | admin, fundacion |
| PUT | `/api/inventory/items/:itemId` | admin, fundacion |
| DELETE | `/api/inventory/items/:itemId` | admin, fundacion |
| GET | `/api/inventory/batches` | admin, fundacion, veterinario |
| POST | `/api/inventory/items/:itemId/batches` | admin, fundacion |
| PUT | `/api/inventory/batches/:batchId` | admin, fundacion |
| DELETE | `/api/inventory/batches/:batchId` | admin, fundacion |
| GET | `/api/inventory/movements` | admin, fundacion, veterinario |
| POST | `/api/inventory/movements` | admin, fundacion, veterinario |
| GET | `/api/inventory/alerts` | admin, fundacion, veterinario |
| GET | `/api/rooms` | admin, fundacion, veterinario |
| POST | `/api/rooms` | admin, fundacion |
| GET | `/api/rooms/:roomId` | admin, fundacion, veterinario |
| PUT | `/api/rooms/:roomId` | admin, fundacion |
| DELETE | `/api/rooms/:roomId` | admin, fundacion |
| GET | `/api/rooms/reservations` | admin, fundacion, veterinario |
| POST | `/api/rooms/:roomId/reservations` | admin, fundacion, veterinario |
| PUT | `/api/rooms/reservations/:reservationId` | admin, fundacion, veterinario |
| DELETE | `/api/rooms/reservations/:reservationId` | admin, fundacion, veterinario |
| GET | `/api/rooms/:roomId/equipment` | admin, fundacion, veterinario |
| POST | `/api/rooms/:roomId/equipment` | admin, fundacion |
| DELETE | `/api/rooms/:roomId/equipment/:equipmentId` | admin, fundacion |
| GET | `/api/equipment` | admin, fundacion, veterinario |
| POST | `/api/equipment` | admin, fundacion |
| GET | `/api/equipment/:equipmentId` | admin, fundacion, veterinario |
| PUT | `/api/equipment/:equipmentId` | admin, fundacion |
| DELETE | `/api/equipment/:equipmentId` | admin, fundacion |
| GET | `/api/equipment/:equipmentId/maintenance` | admin, fundacion, veterinario |
| POST | `/api/equipment/:equipmentId/maintenance` | admin, fundacion |
| GET | `/api/adoptions` | admin, fundacion, adoptante |
| POST | `/api/adoptions` | adoptante |
| GET | `/api/adoptions/:id` | solicitante, fundación dueña, admin |
| GET | `/api/adoptions/:id/events` | solicitante, fundación dueña, admin |
| PUT | `/api/adoptions/:id` | solicitante, admin (pendiente) |
| POST | `/api/adoptions/:id/approve` | fundación dueña, admin |
| POST | `/api/adoptions/:id/reject` | fundación dueña, admin |
| POST | `/api/adoptions/:id/cancel` | solicitante, admin |

---

## 14. Ejemplos con `curl` / PowerShell

```powershell
# Estado del servicio (informe completo)
Invoke-RestMethod "http://localhost:3000/api/health?verbose=true"

# Login y guardado de cookie de sesión
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
Invoke-RestMethod -Method Post -Uri "http://localhost:3000/api/auth/login" `
  -ContentType "application/json" -WebSession $session `
  -Body '{"email":"demo.admin@petguardian.test","password":"demo-password-123"}'

# Listar mascotas
Invoke-RestMethod -Uri "http://localhost:3000/api/pets" -WebSession $session

# Panel
Invoke-RestMethod -Uri "http://localhost:3000/api/dashboard" -WebSession $session
```
