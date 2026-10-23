# PetGuardian

Aplicación web adaptable a celulares y computadores, instalable como **PWA**.
Incluye el esqueleto (frontend, backend y base de datos), el módulo de
**usuarios, autenticación y autorización por roles**, el módulo de
**gestión de mascotas** (CRUD, búsqueda, filtros y estados), el módulo de
**adopciones** (solicitudes, revisión y decisión por parte de la fundación), el
**panel con métricas reales** y el **expediente clínico** (carnet de vacunación,
procedimientos/cirugías, citas, consultas y desparasitación).

PetGuardian admite **exclusivamente perros y gatos**; la especie se valida en
el frontend y en el backend. Los registros existentes de otras especies **no se
eliminan ni se modifican automáticamente**.

## Tecnologías

| Capa            | Tecnología                                  |
| --------------- | ------------------------------------------- |
| Frontend        | React + Vite (JavaScript)                   |
| PWA             | `vite-plugin-pwa` (Workbox, `generateSW`)   |
| Backend         | Node.js + Express                           |
| Base de datos   | SQLite (`better-sqlite3`)                   |
| Orquestación    | `concurrently` (scripts raíz)               |

## Requisitos

Comprobados en este equipo:

- Node.js **v20.20.1** (mínimo `>=20.19.0`)
- npm **10.8.2**
- Git (opcional, para control de versiones)

## Estructura de carpetas

```
PETGUARDIAN/
├─ backend/
│  ├─ src/
│  │  ├─ config/db.js            Conexión + migraciones versionadas SQLite
│  │  ├─ config/roles.js         Roles y roles permitidos en registro público
│  │  ├─ config/auth.js          Cookie y duración de sesión
│  │  ├─ config/pets.js          Especies, sexos, estados y transiciones
│  │  ├─ config/adoptions.js     Estados, transiciones y roles de adopciones
│  │  ├─ config/medical.js       Tipos de registro, estados clínicos y permisos
│  │  ├─ data/                   Acceso a datos (users, sessions, pets, adoptions, medical, dashboard, health)
│  │  ├─ services/               Lógica de negocio (auth, password, pets, adoptions, medical, dashboard)
│  │  ├─ controllers/            Manejo de peticiones/respuestas
│  │  ├─ routes/                 Definición de rutas /api
│  │  ├─ validations/            Validación de entradas (auth, pets, adoptions, medical)
│  │  ├─ middlewares/            Cookies, auth/roles, 404 y errores
│  │  ├─ utils/httpError.js      Errores HTTP con código y detalles
│  │  ├─ app.js                  Configuración de Express
│  │  └─ server.js               Arranque del servidor HTTP
│  ├─ scripts/create-admin.mjs   Creación del primer administrador (desde .env)
│  ├─ scripts/seed-demo.mjs      Datos de demostración idempotentes (opcional)
│  ├─ test/health.test.js        Pruebas de /api/health
│  ├─ test/auth.test.js          Pruebas de auth, roles y contraseñas
│  ├─ test/pets.test.js          Pruebas de mascotas (CRUD, permisos, estados)
│  ├─ test/adoptions.test.js     Pruebas de adopciones (permisos, transiciones, concurrencia)
│  ├─ test/medical.test.js       Pruebas de salud veterinaria y dashboard
│  ├─ data/                      Archivo SQLite (generado, no versionado)
│  ├─ .env                       Variables locales + secretos (no versionado)
│  ├─ .env.example               Plantilla de variables (sin secretos)
│  └─ package.json
├─ frontend/
│  ├─ public/                    Iconos PWA, favicon, robots.txt
│  ├─ src/
│  │  ├─ context/AuthContext.jsx Estado de sesión (login/registro/logout)
│  │  ├─ components/             LoginForm, RegisterForm, AppShell, Dashboard,
│  │  │                          PetsSection, PetCard, PetDetail, PetForm,
│  │  │                          PetMedicalPanel, MedicalRecordForm,
│  │  │                          MedicalRecordList, AdoptionsSection,
│  │  │                          AdoptionCard, AdoptionDetail, AdoptionForm,
│  │  │                          ConfirmDialog, FlashMessage, Icon, Field
│  │  ├─ validations/auth.js     Validación de formularios en el cliente
│  │  ├─ constants/roles.js      Etiquetas de roles
│  │  ├─ constants/pets.js       Etiquetas de especies, sexos y estados
│  │  ├─ constants/adoptions.js  Etiquetas y reglas de adopciones en el cliente
│  │  ├─ constants/medical.js    Etiquetas, estados y formularios clínicos
│  │  ├─ services/api.js         Cliente HTTP del frontend
│  │  ├─ App.jsx                 Navegación según estado de sesión
│  │  ├─ main.jsx                Punto de entrada de React
│  │  └─ index.css               Estilos (responsive)
│  ├─ index.html
│  ├─ vite.config.js             Vite + configuración PWA + proxy /api
│  ├─ .npmrc                     legacy-peer-deps (ver nota)
│  └─ package.json
├─ package.json                  Scripts raíz (dev con concurrently)
├─ .gitignore
└─ README.md
```

## Instalación

Abre **Windows PowerShell** en la carpeta raíz del proyecto.

```powershell
# 1. Dependencias de la raíz (concurrently)
npm install

# 2. Dependencias del backend
npm --prefix backend install

# 3. Dependencias del frontend
npm --prefix frontend install
```

O en un solo comando:

```powershell
npm run install:all
```

> Nota: la instalación del frontend usa `legacy-peer-deps=true` (definido en
> `frontend/.npmrc`). Evita un error conocido de npm 10
> (`Cannot read properties of null (reading 'edgesOut')`) al resolver
> peer-dependencies de las herramientas de PWA.

### Variables de entorno

El backend carga `.env` de forma opcional. Si no existe, se usan los valores por
defecto. Para personalizarlos:

```powershell
Copy-Item backend\.env.example backend\.env
```

| Variable        | Por defecto                    | Descripción                    |
| --------------- | ------------------------------ | ------------------------------ |
| `PORT`          | `3000`                         | Puerto del backend             |
| `DATABASE_FILE` | `./data/petguardian.db`        | Ruta del archivo SQLite        |
| `CORS_ORIGIN`   | `http://localhost:5173`        | Origen permitido por CORS      |
| `SESSION_COOKIE_NAME` | `pg_session`             | Nombre de la cookie de sesión  |
| `SESSION_TTL_HOURS`   | `168` (7 días)           | Duración de la sesión          |
| `COOKIE_SECURE` | `false` (usar `true` en HTTPS) | Marca la cookie como `Secure`  |

> Los secretos y credenciales **nunca** se versionan: `.env` está en
> `.gitignore`. Usa `backend/.env.example` como plantilla (sin valores reales).

## Iniciar la aplicación en desarrollo

```powershell
# Backend y frontend a la vez
npm run dev
```

- Frontend: http://localhost:5173 (Vite redirige `/api` al backend)
- Backend:  http://localhost:3000

También puedes arrancarlos por separado:

```powershell
npm run dev:backend
npm run dev:frontend
```

## Endpoint de salud

```powershell
# Respuesta resumida
Invoke-RestMethod http://localhost:3000/api/health

# Informe completo
Invoke-RestMethod "http://localhost:3000/api/health?verbose=true"
```

Ejemplo de respuesta resumida:

```json
{
  "status": "ok",
  "service": "petguardian-backend",
  "version": "0.1.0",
  "timestamp": "2026-01-01T00:00:00.000Z",
  "database": true
}
```

## Autenticación, sesiones y roles

- **Contraseñas:** se guardan como hash **scrypt** (`node:crypto`, sin
  dependencias extra). Nunca en texto plano.
- **Sesiones:** token opaco aleatorio almacenado en SQLite (se guarda solo su
  hash SHA-256) y enviado en una cookie `HttpOnly`, `SameSite=Lax`.
- **Roles:** `admin`, `fundacion`, `adoptante`, `veterinario`.
- **Registro público:** solo permite `adoptante`, `fundacion` y `veterinario`.
  El rol `admin` **no** puede elegirse desde el formulario; se valida en el
  servidor. El rol siempre se verifica en el backend, nunca se confía en lo que
  envíe el navegador.

### Endpoints

| Método | Ruta                 | Acceso          | Descripción                        |
| ------ | -------------------- | --------------- | ---------------------------------- |
| POST   | `/api/auth/register` | Público         | Crea cuenta (roles no admin)       |
| POST   | `/api/auth/login`    | Público         | Inicia sesión, emite cookie        |
| POST   | `/api/auth/logout`   | Autenticado     | Cierra sesión e invalida el token  |
| GET    | `/api/auth/me`       | Autenticado     | Devuelve el usuario de la sesión   |
| GET    | `/api/auth/roles`    | Público         | Roles disponibles para el registro |
| GET    | `/api/users`         | Solo `admin`    | Lista de usuarios                  |
| GET    | `/api/users/veterinarians` | `admin`, `fundacion`, `veterinario` | Directorio de veterinarios (id y nombre) |
| GET    | `/api/dashboard`     | Autenticado     | Métricas reales según el rol       |
| GET    | `/api/pets`          | Autenticado     | Lista/busca/filtra mascotas        |
| GET    | `/api/pets/:id`      | Autenticado     | Detalle de una mascota             |
| POST   | `/api/pets`          | `admin`, `fundacion` | Crea una mascota              |
| PUT    | `/api/pets/:id`      | Dueño o `admin` | Edita una mascota                  |
| DELETE | `/api/pets/:id`      | Dueño o `admin` | Elimina una mascota                |
| GET    | `/api/adoptions`     | Autenticado     | Lista solicitudes (según el rol)   |
| POST   | `/api/adoptions`     | Solo `adoptante`| Crea una solicitud de adopción     |
| GET    | `/api/adoptions/:id` | Solicitante, fundación dueña o `admin` | Detalle de la solicitud |
| GET    | `/api/adoptions/:id/events` | Solicitante, fundación dueña o `admin` | Historial de cambios |
| PUT    | `/api/adoptions/:id` | Solicitante o `admin` | Edita observaciones (solo si pendiente) |
| POST   | `/api/adoptions/:id/approve` | Fundación dueña o `admin` | Aprueba la adopción |
| POST   | `/api/adoptions/:id/reject`  | Fundación dueña o `admin` | Rechaza la solicitud |
| POST   | `/api/adoptions/:id/cancel`  | Solicitante o `admin` | Cancela una solicitud pendiente |
| GET    | `/api/pets/:id/:type`        | `admin`, `veterinario` o fundación dueña | Historial clínico de una mascota |
| POST   | `/api/pets/:id/:type`        | `admin`, `veterinario` o fundación dueña | Crea un registro clínico |
| PUT    | `/api/pets/:id/:type/:recordId` | `admin`, `veterinario` o fundación dueña | Edita un registro clínico |
| DELETE | `/api/pets/:id/:type/:recordId` | `admin`, `veterinario` o fundación dueña | Elimina un registro clínico |

`type` es uno de: `vaccinations`, `procedures`, `appointments`, `consultations`
o `deworming`.

### Crear el primer administrador

El rol `admin` se crea por consola, no desde el formulario público. Añade las
credenciales en `backend/.env` (no versionado) y ejecuta:

```powershell
# En backend\.env
# ADMIN_EMAIL=tu-correo@example.com
# ADMIN_PASSWORD=una-contrasena-larga-y-unica
# ADMIN_NAME=Administrador PetGuardian

npm --prefix backend run create:admin
```

### Datos de demostración (opcional)

Para explorar la aplicación con contenido realista sin inventarlo en la interfaz,
hay un *seed* idempotente que crea una cuenta demo por rol, seis mascotas
(perro/gato, en distintos estados), sus fichas clínicas (vacunas,
desparasitación, procedimiento y citas) y una solicitud de adopción pendiente, de
modo que el Panel y el flujo de adopción muestren datos reales:

```powershell
npm --prefix backend run seed:demo
```

Cuentas demo (contraseña por defecto `demo-password-123`):

| Rol | Correo |
| --- | --- |
| Administrador | `demo.admin@petguardian.test` |
| Fundación | `demo.fundacion@petguardian.test` |
| Veterinario aliado | `demo.vet@petguardian.test` |
| Adoptante | `demo.adoptante@petguardian.test` |

- Es **idempotente**: volver a ejecutarlo no duplica nada.
- **No** elimina ni modifica datos existentes; solo agrega lo que falta.
- Se puede desactivar con `SEED_DEMO=0`. La contraseña de las cuentas demo se
  puede cambiar con `SEED_DEMO_PASSWORD` (por defecto `demo-password-123`).
- No se ejecuta solo: hay que invocarlo a propósito.

#### Demo por roles

Con `npm --prefix backend run seed:demo` (contraseña `demo-password-123`) puedes
entrar con cada rol y comprobar la experiencia completa:

| Rol | Usuario | Qué probar |
| --- | --- | --- |
| **Administrador** | `demo.admin@petguardian.test` | Panel global, **Usuarios y roles**, todas las mascotas, adopciones e historial clínico completo (lectura/escritura). |
| **Fundación** | `demo.fundacion@petguardian.test` | Gestión de sus mascotas, aprobación/rechazo de solicitudes, expediente clínico y alertas **solo de sus mascotas**. No ve Usuarios. |
| **Veterinario aliado** | `demo.vet@petguardian.test` | Panel clínico, historial de cada mascota (vacunas, desparasitación, procedimientos, citas, consultas) con escritura. **No** ve adopciones ni usuarios. |
| **Adoptante** | `demo.adoptante@petguardian.test` | **Mascotas disponibles**, ficha pública y **Mis solicitudes** (crear solicitud, estado e historial). **No** ve historial clínico, usuarios ni herramientas de gestión. |

Escenarios de alerta demostrables con el seed (datos reales, sin inventar campos):

- **Luna** (gato): vacuna **vencida** y **próxima**, desparasitación **vencida** y **próxima**, cirugía **programada** y cita **próxima** → alerta **crítica**.
- **Toby** (perro): vacuna **próxima**, cirugía **pendiente** y cita **programada**.
- **Milo** (perro): cita **programada** (próxima).
- **Rocky/Nala/Michi**: sin alertas (el modelo solo genera alertas a partir de
  fechas de próxima dosis, procedimientos activos y citas programadas).

### Probar en el navegador

1. `npm run dev` y abre http://localhost:5173
2. Pulsa **Registrate**, completa el formulario y elige un rol (no aparece
   administrador). Al crearse la cuenta, la sesión se inicia automáticamente.
3. **Cerrar sesión** desde el panel y volver a entrar con **Iniciar sesión**.
4. Si entras con un usuario `admin` o `fundacion`, verás además **Usuarios**
   (solo `admin`) y el historial clínico editable; con `adoptante` la sección de
   adopciones pasa a ser **Mis solicitudes**.

## Tema claro/oscuro

La aplicación incluye un selector de tema (**claro**, **oscuro** o **sistema**)
en el panel lateral y en la sección **Perfil**. La preferencia se guarda en el
navegador (`localStorage`) y se aplica antes del primer pintado para evitar
parpadeos. Si eliges *sistema*, sigue automáticamente el tema del sistema
operativo.

## Módulo de mascotas

La navegación principal (tras iniciar sesión) se adapta al rol: **Panel**,
**Mascotas**, **Adopciones** y **Perfil** para todos; **Usuarios** solo para
`admin`. En **Mascotas** se listan las mascotas en tarjetas, con búsqueda por
nombre, filtros por especie y estado y ordenación (más recientes, nombre o
estado), vista de detalle y formulario de alta/edición (solo `admin` y
`fundacion`). La **barra de búsqueda** del panel lateral es global: al escribir,
salta a **Mascotas** y filtra por nombre. El **Panel** muestra, con datos reales,
una barra de distribución por estado, las mascotas que requieren atención, la
actividad reciente y las alertas calculadas.

### Campos

Nombre, especie (`perro` o `gato`), raza, fecha de nacimiento **o** edad
aproximada en meses, sexo (`macho`, `hembra`, `desconocido`), peso en kg
(opcional), color, descripción, fotografía opcional, estado y fecha de registro
(automática). La especie se limita a perro y gato tanto en el formulario como en
la API.

### Estados y transiciones

Estados: `disponible`, `en_proceso`, `adoptada`, `bajo_cuidado`. Se validan en
el servidor:

- Entre `disponible`, `en_proceso` y `bajo_cuidado` se puede cambiar libremente
  (por el dueño o un admin).
- `adoptada` es un estado **terminal**: solo un `admin` puede revertirlo. Así
  una mascota adoptada no vuelve a `disponible` sin una operación autorizada.
- Si al editar no se envía el estado, se conserva el actual.

### Permisos

- **Listar y ver detalle:** cualquier usuario autenticado.
- **Crear:** `admin` y `fundacion`. El propietario de la mascota es el usuario
  que la crea.
- **Editar y eliminar:** el **dueño** o un `admin`. Otra fundación recibe `403`.
- Toda la autorización se decide en el backend; el rol, el identificador y la
  propiedad **nunca** se confían al frontend.

### Fotografías

La imagen es opcional y se valida tipo y tamaño en cliente y servidor:

- Formatos permitidos: **PNG, JPEG o WEBP**.
- Tamaño máximo: **2 MB**.
- Se comprueba el tipo declarado y los **bytes mágicos** del archivo, además de
  aceptar una URL `http(s)`.
- El cuerpo JSON acepta hasta 6 MB (para la imagen en base64).

### Duplicados

Si el mismo dueño intenta registrar dos mascotas con **el mismo nombre, especie
y fecha de nacimiento**, se responde `409 possible_duplicate`. Tener el mismo
nombre en otra fundación (o sin coincidir fecha y especie) está permitido.

### Probar el módulo en el navegador

1. `npm run dev` y abre http://localhost:5173.
2. Inicia sesión con una cuenta **fundación** (o crea una en **Registrate**).
3. Pestaña **Mascotas** → **Registrar mascota**; completa el formulario y
   guarda. Aparecerá en el listado.
4. Usa la **búsqueda** por nombre y los **filtros** por especie/estado; pulsa
   una tarjeta para ver el **detalle**.
5. En el detalle, **Editar** y **Eliminar** (con confirmación). Cambia el estado
   a `adoptada` y comprueba que el botón de volver a `disponible` da error.
6. Entra con una cuenta **adoptante**: verás el listado pero **no** el botón de
   registrar ni de editar/eliminar (el backend también lo impediría).

## Módulo de adopciones

La pestaña **Adopciones** lista las solicitudes en tarjetas, con filtros por
estado y por mascota, vista de detalle e historial de cambios. El rol determina
qué se ve y qué se puede hacer.

### Flujo

- Un usuario **`adoptante`** crea una solicitud eligiendo una mascota (con
  observaciones opcionales). Puede editar sus observaciones o cancelarla
  mientras siga **pendiente**.
- La **fundación responsable de la mascota** (o un `admin`) **aprueba** o
  **rechaza** la solicitud, con una nota opcional.
- Al aprobar, la mascota pasa a `adoptada` y las demás solicitudes pendientes de
  esa mascota se **rechazan automáticamente**: nunca quedan dos adopciones
  activas incompatibles para la misma mascota.
- Si se rechazan o cancelan todas las solicitudes pendientes de una mascota que
  estaba `en_proceso`, vuelve a `disponible`.

### Estados y transiciones

Estados: `pendiente`, `aprobada`, `rechazada`, `cancelada`. Solo `pendiente`
admite cambios; `aprobada`, `rechazada` y `cancelada` son finales. Cualquier
operación incompatible (por ejemplo, aprobar una solicitud ya rechazada) responde
`409 invalid_status_transition`.

### Permisos

- **Crear:** solo `adoptante`.
- **Listar y ver detalle:** `adoptante` (solo las suyas), `fundacion` (solo las de
  sus mascotas) y `admin` (todas). El rol `veterinario` recibe `403`.
- **Editar observaciones / cancelar:** el solicitante o un `admin`.
- **Aprobar / rechazar:** la fundación dueña de la mascota o un `admin`. Un
  adoptante **nunca** puede aprobar ni modificar solicitudes de otros.
- Toda la autorización se decide en el backend.

### Consistencia y concurrencia

Aprobar una adopción y actualizar el estado de la mascota ocurren en **una sola
transacción** `BEGIN IMMEDIATE` (con `busy_timeout`): dos aprobaciones
simultáneas se serializan y solo una gana. Además, un **índice único parcial**
garantiza a nivel de base de datos que una mascota tenga como máximo una
adopción aprobada.

### Historial

Cada solicitud registra su **fecha, estado, observaciones y decisiones**, y
guarda un historial de eventos (`created`, `edited`, `approved`, `rejected`,
`cancelled`) consultable en `GET /api/adoptions/:id/events`.

### Probar el módulo en el navegador

1. `npm run dev` y abre http://localhost:5173.
2. Con una cuenta **fundación**, registra una mascota (pestaña **Mascotas**).
3. Con una cuenta **adoptante**, pestaña **Adopciones** → **Solicitar adopción**
   y elige esa mascota.
4. Vuelve a la cuenta **fundación**: en **Adopciones**, abre la solicitud y
   pulsa **Aprobar** (o **Rechazar**). Verás la mascota como `adoptada` en
   **Mascotas** y el historial de la solicitud actualizado.

## Panel con métricas reales

El **Panel** consulta `GET /api/dashboard`, que ejecuta agregaciones SQL sobre la
base de datos (nunca cifras inventadas). Si aún no hay datos, se muestran `0` y
estados vacíos:

- **Mascotas:** total, perros, gatos, disponibles, en proceso, adoptadas y bajo
  cuidado.
- **Salud** (solo roles con acceso clínico): vacunas próximas y vencidas,
  desparasitación próxima y vencida, cirugías pendientes y próximas citas.
- **Adopciones:** solicitudes pendientes y adopciones aprobadas.

Además, el Panel funciona como **centro de operaciones**:

- **Accesos rápidos** a Mascotas, Adopciones y al expediente clínico.
- **Mascotas que requieren atención**: lista real (no inventada) de las mascotas
  con vacunas/desparasitaciones vencidas o próximas, cirugías pendientes o citas
  programadas, ordenadas por prioridad. Cada fila abre directamente la ficha de la
  mascota.
- **Alertas** agregadas a partir de las mismas métricas reales.
- **Actividad reciente**: últimos movimientos reales (mascotas registradas y
  solicitudes de adopción), construidos a partir de los datos existentes. Si no
  hay registros, la sección no se muestra.
- **Estados de carga**: las tarjetas de métricas usan *skeletons* mientras se
  consultan los datos (nunca cifras provisionales inventadas).

El alcance depende del rol: una **fundación** solo ve sus propias mascotas; un
**adoptante** no recibe la sección de salud; un **veterinario** no recibe la de
adopciones; un **admin** ve todo.

## Módulo de salud veterinaria (expediente clínico)

La ficha de cada mascota (en **Mascotas** → detalle) se organiza en **pestañas**:
**Informacion** (datos generales), **Vacunacion** (carnet e historial, con fechas
y dosis proximas/vencidas), **Consultas** (historial clinico), **Cirugias y
procedimientos** (con su estado) y **Tratamientos y controles** (desparasitacion
y proximos controles/citas). La pestaña **Informacion** la ve cualquier usuario
autenticado; el resto solo los roles con acceso clinico, que cuentan con CRUD
completo. El historial medico es **privado**: nunca se expone a un adoptante. No
hay una seccion de navegacion aparte: todo vive dentro del detalle de la mascota.

### Tipos de registro

| Tipo            | Contenido                                             | Estado                                        |
| --------------- | ----------------------------------------------------- | --------------------------------------------- |
| `vaccinations`  | Vacunas (nombre, lote, dosis, fecha de aplicación)    | Calculado: aplicada / próxima / vencida       |
| `procedures`    | Procedimientos y cirugías (fecha programada y de realización) | pendiente / programado / realizado / cancelado |
| `appointments`  | Citas veterinarias (fecha y hora, motivo)             | programada / completada / cancelada           |
| `consultations` | Consultas (fecha, diagnóstico, tratamiento)           | —                                             |
| `deworming`     | Desparasitación (interna / externa / mixta, próxima dosis) | Calculado: aplicada / próxima / vencida  |

### Reglas de negocio

- Los **estados calculados** derivan de las fechas: *aplicada* si la fecha ya
  pasó, *próxima* si vence en los siguientes 30 días y *vencida* si la fecha de
  próxima dosis quedó atrás. No se almacenan a mano.
- Un **procedimiento** no puede marcarse como `realizado` sin fecha de
  realización ni con una fecha futura (`409 procedure_not_completed` /
  `invalid_performed_date`). `realizado` es **terminal**; las demás transiciones
  se validan en el servidor (`409 invalid_status_transition`).
- Fechas de aplicación y de consulta **no** pueden ser futuras.
- Si se indica un veterinario, debe ser un usuario con ese rol (`400`).

### Permisos

- **Ver y gestionar:** `admin`, `veterinario` (cualquier mascota) y `fundacion`
  (solo sus mascotas). Un `adoptante` recibe `403`.
- El directorio `GET /api/users/veterinarians` (solo `admin`, `fundacion`,
  `veterinario`) devuelve únicamente **id y nombre**, sin datos personales.

### Probar el módulo en el navegador

1. `npm run dev` y abre http://localhost:5173.
2. Con una cuenta **fundación**, registra una mascota y abre su **detalle**.
3. En el **expediente clínico**, añade una **vacuna** con fecha de próxima dosis
   y comprueba el estado calculado; registra una **cirugía** y prueba a marcarla
   como realizada con y sin fecha.
4. Vuelve al **Panel**: verás las métricas reales actualizadas.
5. Entra con una cuenta **adoptante**: no aparece el expediente clínico.

## Accesibilidad y experiencia de uso

- **Estados de interfaz** en cada vista: cargando (*skeletons*), error, vacío,
  éxito y deshabilitado (mientras hay operaciones en curso).
- **Feedback** tras crear/editar/eliminar (mascotas, adopciones y registros
  clínicos) con avisos de éxito auto-descartables.
- **Navegación por teclado**: los diálogos de confirmación se cierran con `Esc`,
  enfocan el botón seguro al abrirse y usan `role="dialog"` + `aria-modal`.
  Los campos obligatorios se marcan visualmente y con `aria-required`.
- **Diseño responsive**: rail lateral en escritorio, barra superior con
  navegación y buscador en móvil; rejillas fluidas de tarjetas.
- **Iconografía** propia en SVG (sin emojis) y color de tema coherente.

## Pruebas

```powershell
npm test
```

Usa el runner integrado de Node (`node --test`), sin dependencias adicionales.
Cubre `GET /api/health`, el módulo de autenticación (registro, login,
credenciales incorrectas, rutas protegidas por rol y almacenamiento del hash), el
módulo de mascotas (CRUD, validación de datos y fotografías, búsqueda, filtros,
permisos por rol y propiedad, transiciones de estado y persistencia), el módulo
de adopciones (creación, permisos y visibilidad por rol, edición, aprobación y
rechazo, transiciones y cancelación, concurrencia de aprobaciones y persistencia)
y el módulo de salud veterinaria y dashboard (permisos y privacidad, estados
calculados, reglas de procedimientos, métricas por rol y persistencia).
Las pruebas se ejecutan sobre una base de datos SQLite temporal, por lo que no
tocan tus datos.

## Build de producción (frontend + PWA)

```powershell
npm run build
```

Genera `frontend/dist` con el service worker (`sw.js`) y el
`manifest.webmanifest`. Para previsualizarlo:

```powershell
npm --prefix frontend run preview
```

> La PWA requiere HTTPS o `localhost` para instalarse. En red local usar
> `vite --host` y servir por HTTPS.

## Notas de compatibilidad

- Node.js `>=20.19.0` (requerido por Vite 8 y `@vitejs/plugin-react` 6).
- `better-sqlite3` se instala con binarios precompilados; si no hubiera
  binario para tu versión de Node, compilaría desde código fuente
  (requiere las Build Tools de Visual Studio).

## Qué falta para los módulos funcionales

1. Notificaciones y recordatorios de vacunas y citas (por correo o PWA).
2. Edición de la ficha clínica desde el rol `veterinario` en móvil.
3. Exportar el expediente clínico (PDF) y compartirlo con el adoptante.
4. Recuperación de contraseña y verificación de correo.
5. Ampliar los tests end-to-end en el navegador.
6. Inicializar el repositorio Git si se desea control de versiones.

> La versión actual del esquema de base de datos es la **4** (migración
> `pet_health_records`). Las migraciones son **aditivas**: al arrancar se aplican
> sobre la base existente sin borrar ni reinicializar datos.
