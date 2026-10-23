# PetGuardian — Guía de Demostración

Guion reproducible para mostrar el sistema completo con **datos reales** (no
inventados) usando el *seed* de demostración. Cubre los cuatro roles.

- **Contraseña demo por defecto:** `demo-password-123`
  (configurable con `SEED_DEMO_PASSWORD`).
- **Seed idempotente y no destructivo:** se puede ejecutar varias veces sin
  duplicar ni borrar nada; se desactiva con `SEED_DEMO=0`.

---

## 1. Puesta en marcha

```powershell
# 1. Dependencias (solo la primera vez)
npm run install:all

# 2. (Opcional) variables de entorno
Copy-Item backend\.env.example backend\.env

# 3. Datos de demostración
npm --prefix backend run seed:demo

# 4. Arrancar backend + frontend
npm run dev
```

- Frontend: `http://localhost:5173`
- Backend: `http://localhost:3000`

Salida esperada del seed:

```
[seed-demo] Listo.
[seed-demo] Mascotas creadas: 6
[seed-demo] Fichas clinicas con datos: 4
[seed-demo] Solicitudes de adopcion creadas: 1
[seed-demo] Cuentas demo (contrasena: demo-password-123): ...
```

> Si vuelves a ejecutarlo, los contadores serán `0` (ya existe todo).

---

## 2. Cuentas de demostración

| Rol | Correo | Contraseña |
| --- | --- | --- |
| Administrador | `demo.admin@petguardian.test` | `demo-password-123` |
| Fundación | `demo.fundacion@petguardian.test` | `demo-password-123` |
| Veterinario aliado | `demo.vet@petguardian.test` | `demo-password-123` |
| Adoptante | `demo.adoptante@petguardian.test` | `demo-password-123` |

> El correo del veterinario es `demo.vet@...` (no `demo.veterinario@...`).

---

## 3. Datos sembrados

### 3.1 Mascotas (dueño: `Fundacion Patitas Demo`)

| Nombre | Especie | Estado | Ficha clínica |
| --- | --- | --- | --- |
| Luna | gato | `bajo_cuidado` | Vacunas, desparasitación, procedimiento y cita |
| Rocky | perro | `disponible` | Vacuna y consulta |
| Milo | perro | `en_proceso` | Cita |
| Nala | gato | `adoptada` | — |
| Toby | perro | `disponible` | Vacuna, procedimiento, cita y consulta |
| Michi | gato | `disponible` | — |

### 3.2 Solicitud de adopción

- **Solicitante:** `Carlos Adoptante Demo` (adoptante).
- **Mascota:** Rocky (estaba `disponible` → pasa a `en_proceso`).
- **Estado:** `pendiente`.

### 3.3 Escenarios de alerta (reales)

| Mascota | Alerta | Datos que la generan |
| --- | --- | --- |
| **Luna** | **Crítica** | Vacuna vencida (Moquillo, −5 días) + vacuna próxima (Rabia, +20 d) + desparasitación vencida (−3 d) + próxima (+10 d) + cirugía programada + cita próxima (+3 d, 09:30) |
| **Toby** | Media | Vacuna próxima (+12 d) + cirugía pendiente (Castración, +9 d) + cita (+9 d, 10:30) |
| **Milo** | Baja | Cita próxima (+7 d, 11:00) |
| Rocky / Nala / Michi | Sin alertas | — |

Los offsets son relativos a la fecha en que se ejecuta el seed.

---

## 4. Recorrido por rol

### 4.1 Adoptante — `demo.adoptante@petguardian.test`

1. Inicia sesión → pantalla de **Inicio / Panel** con "Mascotas disponibles" y
   "Mis solicitudes".
2. Pestaña **Mascotas disponibles**: se listan las mascotas (Rocky, Toby, Michi,
   Luna, Milo) — **sin** el botón de registrar.
3. Abre el detalle de Rocky → ficha **pública**: **no** aparece el expediente
   clínico.
4. Pestaña **Mis solicitudes**: verás la solicitud **pendiente** sobre Rocky
   creada por el seed, con su historial (`created`).
5. Crea una nueva solicitud sobre **Toby** con una observación. La mascota pasa a
   `en_proceso`.
6. Intenta crear una **segunda** solicitud sobre Rocky → `409 duplicate_request`.

**Qué NO puede:** ver el historial clínico, ver usuarios, aprobar/rechazar, editar
mascotas.

### 4.2 Fundación — `demo.fundacion@petguardian.test`

1. Inicia sesión → **Centro de operaciones** con hero "Centro de operaciones" y
   tarjetas de **Adopciones**.
2. **Atención requerida**: aparecen **Luna** (crítica) y **Toby**; cada fila abre
   su ficha.
3. **Estado de mascotas**: distribución por estado (6 mascotas).
4. Pestaña **Alertas**: alertas agregadas de vacunas/desparasitación/cirugías/citas.
5. Pestaña **Adopciones** (variante *inbox*): filtra `pendiente`; abre la
   solicitud de Rocky y pulsa **Aprobar**.
   - Rocky pasa a `adoptada`.
   - La solicitud queda `aprobada` con evento `approved`.
6. Pestaña **Mascotas**: abre Luna → **Expediente clínico** con sus 5 registros y
   comprueba los estados calculados (vencida/próxima).
7. Comprueba que **no** ve la pestaña Usuarios.

**Detalle de alertas:** al aprobar una mascota, las demás solicitudes pendientes
de esa mascota se **rechazan automáticamente** (aquí solo hay una).

### 4.3 Veterinario aliado — `demo.vet@petguardian.test`

1. Inicia sesión → **Centro clínico** con hero clínico.
2. **Atención pendiente** y **Estado clínico** (vacunas próximas/vencidas, etc.).
3. Pestaña **Pacientes**: solo sus **pacientes asignados** (el seed lo asigna a
   6 mascotas; no ve la séptima, no asignada).
4. Abre el **Expediente clínico** de Luna:
   - Pestaña **Vacunacion**: Rabia (próxima) y Moquillo (vencida).
   - **Tratamientos y controles**: desparasitaciones (vencida/próxima) y los
     **controles próximos** (citas programadas + `followUpDate` de cirugías).
   - **Cirugias y procedimientos**: Esterilización (programado). Prueba a marcar
     `realizado` **sin** fecha → `409 procedure_not_completed`; y con
     posoperatorio + próximo control.
   - Añade una **consulta** con **signos vitales** (peso/temperatura/FC/FR);
     ciérrala; intenta editarla → se bloquea; reábrela para corregir.
   - Registra un **tratamiento** (medicamento con dosis/frecuencia).
5. Pestaña **Calendario clínico**: alterna **Mes/Semana/Día**, agenda una cita,
   reprograma, cancela y **reactiva**; intenta duplicar el mismo horario para el
   mismo paciente → se rechaza (`409 schedule_conflict`).
6. Pestaña **Alertas clinicas**.
7. Comprueba que **no** ve Adopciones ni Usuarios.

### 4.4 Administrador — `demo.admin@petguardian.test`

1. Inicia sesión → **Panel** global.
2. Pestaña **Usuarios**: lista de los 4 usuarios demo (sin `password_hash`).
3. Ve **todas** las mascotas, adopciones (global) e historial clínico.
4. Puede revertir una mascota `adoptada` a otro estado (transición reservada al
   admin).

---

## 5. Verificación por API (sin navegador)

Útil para evidenciar resultados de forma objetiva. Con PowerShell:

```powershell
$s = New-Object Microsoft.PowerShell.Commands.WebRequestSession

# Login
Invoke-RestMethod -Method Post -Uri "http://localhost:3000/api/auth/login" `
  -ContentType "application/json" -WebSession $s `
  -Body '{"email":"demo.fundacion@petguardian.test","password":"demo-password-123"}'

# Panel de la fundación (métricas reales)
Invoke-RestMethod -Uri "http://localhost:3000/api/dashboard" -WebSession $s | ConvertTo-Json -Depth 6

# Mascotas
Invoke-RestMethod -Uri "http://localhost:3000/api/pets" -WebSession $s | ConvertTo-Json -Depth 5

# Solicitudes de adopción visibles para la fundación
Invoke-RestMethod -Uri "http://localhost:3000/api/adoptions" -WebSession $s | ConvertTo-Json -Depth 6

# Expediente clínico (vacunación) de Luna (id 1 habitualmente)
Invoke-RestMethod -Uri "http://localhost:3000/api/pets/1/vaccinations" -WebSession $s | ConvertTo-Json -Depth 5

# Controles próximos y tratamientos vigentes del paciente
Invoke-RestMethod -Uri "http://localhost:3000/api/pets/1/clinical-controls" -WebSession $s | ConvertTo-Json -Depth 6
```

Prueba de alcance clínico del veterinario (login como vet; solo ve sus pacientes
asignados; una mascota no asignada responde `403`):

```powershell
$v = New-Object Microsoft.PowerShell.Commands.WebRequestSession
Invoke-RestMethod -Method Post -Uri "http://localhost:3000/api/auth/login" `
  -ContentType "application/json" -WebSession $v `
  -Body '{"email":"demo.vet@petguardian.test","password":"demo-password-123"}'
Invoke-RestMethod -Uri "http://localhost:3000/api/clinical/agenda" -WebSession $v | ConvertTo-Json -Depth 6
```

Prueba de privacidad (debe dar `403`):

```powershell
$a = New-Object Microsoft.PowerShell.Commands.WebRequestSession
Invoke-RestMethod -Method Post -Uri "http://localhost:3000/api/auth/login" `
  -ContentType "application/json" -WebSession $a `
  -Body '{"email":"demo.adoptante@petguardian.test","password":"demo-password-123"}'
try {
  Invoke-RestMethod -Uri "http://localhost:3000/api/pets/1/vaccinations" -WebSession $a
} catch {
  $_.Exception.Response.StatusCode.value__   # 403
}
```

Prueba de concurrencia (dos aprobaciones): lanza dos `POST /approve` sobre la
misma solicitud; una responde `200` y la otra `409`
(`invalid_status_transition` o `pet_not_available`).

---

## 6. Valores esperados del panel (fundación, recién sembrado)

Según los datos del seed y las reglas de cálculo:

| Métrica | Valor esperado |
| --- | --- |
| `pets.total` | 6 |
| `pets.perros` / `pets.gatos` | 3 / 3 |
| `pets.disponibles` | 3 (Rocky, Toby, Michi) |
| `pets.enProceso` | 1 (Milo; Rocky tras la solicitud) |
| `pets.adoptadas` | 1 (Nala) |
| `pets.bajoCuidado` | 1 (Luna) |
| `adoptions.pendientes` | 1 (Rocky) |
| `adoptions.total` | 1 |
| `medical.vacunasVencidas` | 1 (Luna/Moquillo) |
| `medical.vacunasProximas` | 2 (Luna/Rabia +20 d, Toby/Rabia +12 d) |
| `medical.desparasitacionesVencidas` | 1 (Luna) |
| `medical.desparasitacionesProximas` | 1 (Luna) |
| `medical.cirugiasPendientes` | 2 (Luna programada, Toby pendiente) |
| `medical.proximasCitas` | 3 (Luna, Milo, Toby) |
| `attention` | Luna y Toby (y Milo por la cita) |

> Los valores de "próximas/vencidas" dependen de la fecha de ejecución del seed;
> los offsets están descritos en la sección 3.3.

---

## 7. Reejecución, reinicio y limpieza

```powershell
# Reejecutar el seed (idempotente): no duplica
npm --prefix backend run seed:demo

# Desactivar explícitamente
$env:SEED_DEMO=0; npm --prefix backend run seed:demo

# Cambiar la contraseña demo
$env:SEED_DEMO_PASSWORD="otra-clave"; npm --prefix backend run seed:demo
```

Para **empezar de cero**, detén los procesos y elimina la base de datos:

```powershell
Remove-Item backend\data\petguardian.db* -Force
npm --prefix backend run seed:demo
```

> Eliminar `petguardian.db*` borra **todos** los datos (reales y demo). Hazlo solo
> en un entorno de práctica.

---

## 8. Ejecutar pruebas y build (evidencia)

```powershell
# Pruebas del backend (BD temporal, no toca datos reales)
npm test
# -> 132 tests, 40 suites, pass 132, fail 0

# Build del frontend + PWA
npm run build
# -> frontend/dist con sw.js y manifest.webmanifest
```

---

## 9. Preguntas frecuentes

- **No aparecen datos en el panel:** ejecuta el seed y recarga; recuerda que la
  fundación solo ve **sus** mascotas y el veterinario solo **sus pacientes
  asignados** (`vet_pet_assignments`).
- **El veterinario no ve una mascota:** probablemente no está asignada a él; el
  admin o la fundación propietaria deben asignársela (o el ejemplo usa una
  mascota sin asignar).
- **El adoptante no ve el expediente clínico:** es el comportamiento esperado
  (privacidad); el backend responde `403`.
- **`demo.veterinario@...` no funciona:** el correo correcto es
  `demo.vet@petguardian.test`.
- **Al aprobar una solicitud, otras se rechazan solas:** es correcto; al aprobar
  se rechazan automáticamente las demás pendientes de la misma mascota.
- **La cita de "consulta pendiente" no existe:** el modelo de alertas no incluye
  consultas; solo vacunas, desparasitaciones, procedimientos y citas.
