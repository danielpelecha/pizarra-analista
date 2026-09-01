# PIZARRA ANALISTA — Documento maestro de la aplicación web

**Última actualización:** 31/08/2026
**Propietario:** Daniel Pelechá (D. Pelechá C.F. / entrenador AFV Valencia CF — Cadete)
**Propósito de este documento:** referencia técnica completa de cómo está construida y cómo funciona la app PIZARRA ANALISTA, para poder reconstruirla, depurarla o ampliarla en cualquier momento sin depender del historial de conversación.

> **Cómo mantener este documento vivo:** cada vez que se añade o modifica una función de la app, este documento debe actualizarse en la misma conversación (sección afectada + apartado 10, "Historial de cambios") y volver a subirse al Proyecto de Claude, sustituyendo la versión anterior.

---

## 1. Qué es PIZARRA ANALISTA

Aplicación web para entrenadores de fútbol, pensada para gestionar el club, la plantilla, las alineaciones y los partidos desde un único archivo HTML, con los datos guardados en la nube (Supabase) para poder acceder desde cualquier dispositivo con la misma cuenta.

Es multi-club: cada entrenador que se registra tiene su propio club, su propia plantilla y sus propios partidos, completamente aislados de los demás usuarios (ver apartado 4.4, seguridad).

El logo/marca de la propia app ("D. Pelechá C.F.", figura azul) es distinto del escudo de cada club: el escudo de club es la identidad que ve cada entrenador y sus jugadores; el logo PIZARRA es la marca de la herramienta en sí (aparece pequeño, p. ej. en la cabecera y en "hecho con PIZARRA" del pie de página).

---

## 2. Arquitectura técnica

- **Frontend:** un único archivo HTML autocontenido, `pizarra-app.html` (HTML + CSS + JavaScript en un solo archivo, sin build ni instalación). Se abre localmente o se sirve con `npx serve .` desde la carpeta donde está guardado.
- **Backend / base de datos:** [Supabase](https://supabase.com) (Postgres en la nube + autenticación + almacenamiento de archivos), proyecto llamado **"DATOS EQUIPO FUTBOL"**, dentro de la organización **"D.PELECHA C.F."** (plan Free).
- **Conexión:** el HTML se conecta a Supabase mediante el SDK JS (`@supabase/supabase-js@2`, importado vía `esm.sh` como módulo ES, sin necesidad de `npm install`).
- **Conector MCP de Supabase — CONECTADO desde el 31/08/2026.** A partir de esta fecha, los cambios de base de datos (tablas, columnas, políticas, buckets) los ejecuta Claude directamente sobre el proyecto real mediante este conector (`apply_migration` para cambios de estructura, `execute_sql` para consultas), sin que el usuario tenga que copiar/pegar SQL en el panel de Supabase. El proyecto identificado es **`dhberjeufirasvhtvahj`** ("DATOS EQUIPO FUTBOL", organización `aprwwaqpsxmnqaghphem`). El código de la app (`pizarra-app.html`) se sigue entregando como archivo para que el usuario lo descargue y sobrescriba; solo la parte de base de datos se aplica de forma directa.
- **Autenticación:** Supabase Auth con email + contraseña (registro/login propios, sin login social).
- **Seguridad de datos:** Row Level Security (RLS) de Postgres — cada tabla tiene políticas que aseguran que cada usuario solo puede leer/escribir los datos de su propio club.

---

## 3. Estructura del archivo `pizarra-app.html`

El archivo se divide en tres bloques, en este orden:

1. **`<style>`** — todo el CSS de la app: variables de color, pantallas de login/registro, barra de pestañas, tarjetas, tablas, campo de fútbol, modo en vivo, buscador de jugadores.
2. **HTML de las 4 pantallas principales** (una dentro de la otra, mostrando solo una cada vez mediante la clase `hidden`):
   - `#screenLoading` — pantalla de carga inicial
   - `#screenAuth` — login / registro
   - `#screenCreateClub` — alta del club (primera vez que se entra)
   - `#screenApp` — la aplicación en sí, con la barra de pestañas y el contenido de cada una
3. **`<script type="module">`** — toda la lógica JavaScript, organizada por bloques (ver apartado 6).

### Variables de color (CSS `:root`)
```css
--navy:#0d1b3e;      /* azul marino, color principal de marca */
--blue:#2563eb;      /* azul de acción (botones) */
--lightblue:#ebf2fa; /* fondo suave */
--grey:#787878;
--border:#c7d2e0;
--danger:#b91c1c;    /* rojo, alertas y borrar */
--success:#15803d;
```

---

## 4. Base de datos (Supabase)

### 4.1 Tablas

**`clubs`** — un club por usuario registrado
| columna | tipo | notas |
|---|---|---|
| id | uuid | clave primaria |
| owner_id | uuid | referencia al usuario de Supabase Auth (`auth.uid()`) |
| name | text | nombre del club |
| crest_url | text | URL pública del escudo (añadida más adelante, ver 4.3) |

**`players`** — jugadores de la plantilla (sin límite de cantidad)
| columna | tipo | notas |
|---|---|---|
| id | uuid | clave primaria |
| club_id | uuid | referencia a `clubs.id` |
| name | text | nombre completo |
| alias | text | nombre corto usado en la pizarra táctica; **si no se indica, se recomienda que sea la primera palabra del nombre** (criterio heredado de la herramienta HTML anterior) |
| num | text | dorsal |
| pos | text | posición |
| created_at | timestamptz | |

**`stat_columns`** — columnas de estadísticas personalizadas, **a nivel de club** (no por partido)
| columna | tipo | notas |
|---|---|---|
| id | uuid | clave primaria |
| club_id | uuid | referencia a `clubs.id` |
| key | text | identificador interno (slug único, p. ej. `regates_a1b2`) |
| label | text | nombre visible (p. ej. "Regates") |
| color | text | color del botón/columna en formato hexadecimal (p. ej. `#2563eb`); por defecto `#2563eb` |
| orden | integer | posición de la estadística, tanto en los botones del modo en vivo como en las columnas de la tabla; se reordena arrastrando desde "🎨 Organizar estadísticas" |
| created_at | timestamptz | |

Columnas por defecto que se crean automáticamente la primera vez que un club abre la pestaña Partidos: `minutos`, `goles`, `asistencias`, `ocasiones` (ocasiones de gol), `amarillas`, `rojas`.

**`matches`** — partidos
| columna | tipo | notas |
|---|---|---|
| id | uuid | clave primaria |
| club_id | uuid | referencia a `clubs.id` |
| date | date | |
| rival | text | |
| fase | text | `pretemporada` o `temporada` |
| condicion | text | `LOCAL` o `VISITANTE` |
| sistema_partido | text | disposición táctica usada (informativo) |
| stats | jsonb | **not null, default `{}`** — estadísticas por jugador, ver formato abajo |
| live | jsonb | **not null, default `{}`** — estado del modo en vivo, ver formato abajo |
| last_modified | timestamptz | |
| created_at | timestamptz | |

> ⚠️ **Importante para no repetir un error ya corregido:** las columnas `stats` y `live` son `NOT NULL`. Al crear un partido nuevo desde la app hay que enviar `{}` (objeto vacío), **nunca `null`** — enviar `null` explícito rompe la inserción sin avisar visiblemente. Ver apartado 7.

**Formato del campo `stats` (jsonb):**
```json
{
  "<player_id>": { "minutos": 0, "goles": 0, "asistencias": 0, "...": "..." },
  "<player_id>": { ... }
}
```
Una entrada por jugador de la plantilla, con una clave por cada columna de `stat_columns`.

**Formato del campo `live` (jsonb):**
```json
{
  "startersIds": ["<player_id>", "..."],
  "started": false,
  "finished": false,
  "events": [
    { "type": "gol", "minute": 23, "playerId": "<id>" },
    { "type": "cambio", "minute": 60, "outId": "<id>", "inId": "<id>" }
  ],
  "accSeconds": 0,
  "runStartedAt": null,
  "running": false,
  "half": 1,
  "finalMinute": null
}
```
Tipos de evento posibles: `gol`, `asistencia`, `ocasion`, `amarilla`, `roja` (con `playerId`), y `cambio` (con `outId`/`inId`).

### 4.2 Políticas de seguridad (RLS)

Cada tabla tiene una política que restringe todas las operaciones (`for all`) a las filas cuyo `club_id` pertenece a un club cuyo `owner_id` coincide con el usuario autenticado:
```sql
create policy "jugadores de mi club" on players
  for all using ( club_id in (select id from clubs where owner_id = auth.uid()) );
```
(Mismo patrón para `stat_columns` y `matches`; en `clubs` la política compara directamente `owner_id = auth.uid()`.)

### 4.3 Almacenamiento (Storage) — escudo del club

- Bucket llamado **`crests`**, público para lectura.
- Creado con:
```sql
insert into storage.buckets (id, name, public)
values ('crests', 'crests', true)
on conflict (id) do nothing;
```
- Políticas: lectura pública de todo el bucket; subida/actualización solo permitida sobre carpetas cuyo nombre coincide con un `club_id` propiedad del usuario.
- La columna `clubs.crest_url` guarda la URL pública devuelta tras la subida.

### 4.4 Seguridad y control de acceso — estado actual

- Cualquier persona puede registrarse libremente desde la pantalla de login (autoservicio, sin aprobación).
- Los datos de cada club están completamente aislados gracias a RLS (un usuario nunca puede ver datos de otro club, aunque use la misma app).
- **No existe (todavía) un sistema de aprobación, invitación o cuotas de pago dentro de la app.**
- Si hace falta bloquear a un usuario concreto, se hace **manualmente** desde el panel de Supabase: *Authentication → Users → (seleccionar usuario) → desactivar o eliminar*. No requiere cambios en el código.
- Decisión consciente (30/08/2026): se deja así a propósito mientras la app está en fase de prueba abierta con otros entrenadores/profesionales. Un sistema de aprobación o pago se añadirá más adelante si hace falta.

---

## 5. Flujo de pantallas

```
screenLoading (comprobando sesión)
    │
    ├── sin sesión ──────────────► screenAuth (login / registro)
    │                                   │
    │                                   ▼ (login correcto)
    ├── con sesión, sin club ────► screenCreateClub (alta del club)
    │                                   │
    │                                   ▼
    └── con sesión y con club ──► screenApp (pestañas)
```
La función `afterAuthChange()` es el punto de entrada que decide a qué pantalla ir cada vez que cambia el estado de sesión.

Dentro de `screenApp`, la barra de pestañas (`.tab-btn`) alterna la visibilidad de los `.tab-panel` correspondientes. Al entrar en una pestaña por primera vez se disparan las cargas de datos necesarias (jugadores, columnas de estadísticas, partidos).

---

## 6. Funcionalidades por pestaña

### 6.1 Club — ✅ implementado
- Editar el nombre del club.
- Subir/cambiar el escudo (PNG, JPG o SVG; se recomienda fondo transparente o blanco).
- El escudo se usa después en: cabecera de la Alineación descargada y cabecera de la ficha del partido.

### 6.2 Plantilla — ✅ implementado
- Tabla de jugadores: dorsal, nombre completo, alias, posición.
- Añadir, editar (en línea, con botones de guardar/cancelar) y eliminar jugadores.
- Aviso visual en rojo si dos jugadores comparten el mismo alias (no bloquea, solo avisa).
- **Sin límite de jugadores** registrables en la plantilla.

### 6.3 Alineación — ✅ implementado
- Selector de disposición con dos grupos:
  - **Sistemas de juego:** 1-4-1-4-1, 1-4-3-3, 1-4-4-2, 1-4-2-3-1, 1-3-5-2, 1-5-3-2.
  - **Jugadas a balón parado (ABP):** córner izquierdo, córner derecho, falta frontal, falta lateral izq., falta lateral der., saque de banda ofensivo.
- Datos del partido para la imagen exportada: fecha, hora, rival, condición (local/visitante).
- **11 puestos titulares** + **19 suplentes** (hasta 30 convocados en total, ajustado tras detectar que en amistosos se convocan más de 22).
- **Buscador de jugadores con autocompletado**, igual en titulares y en suplentes:
  - Al escribir una letra o parte del texto se filtra en tiempo real por **nombre completo o alias** de la plantilla.
  - **La lista desplegable de resultados muestra el nombre COMPLETO de cada jugador** (dorsal + nombre completo, con el alias entre paréntesis) — esto es intencional, para poder distinguir entre jugadores que compartan alias o nombre de pila. **Una vez seleccionado**, el hueco se queda con el **alias únicamente** (dorsal + alias), que es lo mismo que se verá en el campo, en el modo en vivo de Partidos y en cualquier otro sitio de la app. Son dos funciones de etiquetado distintas en el código: `searchResultLabel()` (nombre completo, para buscar) y `playerLabel()`/`displayAlias()` (alias, para todo lo demás).
  - Un jugador que **ya está puesto en cualquier otro puesto** (sea titular o suplente) aparece en la lista **en gris, con el texto "· ya está en la alineación", y sin poder hacer clic en él** — el bloqueo es real, no solo un aviso de color.
  - Los jugadores libres aparecen en negro normal y sí se pueden seleccionar.
  - Opción "✕ Vaciar este puesto" para dejarlo sin jugador.
  - El **alias** que se usa para filtrar y para mostrar sobre el campo es el guardado en la Plantilla; si un jugador no tiene alias propio, se usa automáticamente su primer nombre (mismo criterio que la herramienta HTML anterior — se aplica tanto al mostrar como al guardar un jugador nuevo/editado sin alias explícito, ver 4.1).
- Cambiar de disposición **no borra** a los jugadores ya elegidos: se conservan y se recolocan en los nuevos puestos.
- **Arrastre libre**: cada jugador titular se puede arrastrar con el ratón o el dedo sobre el campo para ajustar su posición exacta; botón "Restablecer posiciones" para volver a la disposición estándar.
- Descarga de la alineación como imagen PNG, con escudo del club, datos del partido, campo, jugadores y lista de suplentes.

### 6.4 Partidos — ✅ implementado
- Lista de partidos del club (fecha, rival, etiqueta pretemporada/temporada), con botón para crear uno nuevo.
- Ficha del partido: fecha, rival, condición, fase.
- **Modo en vivo:**
  - Selección de quiénes empiezan en el campo.
  - Cronómetro con iniciar / pausar / reanudar, cambio de parte (1ª/2ª), ajuste manual del minuto, finalizar (y reabrir) partido.
  - Registro de eventos con flujo de dos toques (jugador → acción): ⚽ Gol, 🅰️ Asistencia, 🎯 Ocasión, 🟨 Amarilla, 🟥 Roja, 🔄 Cambio (sustituciones ilimitadas, con seguimiento de los minutos jugados por cada jugador según sus entradas/salidas).
  - Registro cronológico de eventos, con opción de borrar un evento suelto o deshacer el último.
- **Estadísticas por jugador:** tabla con una fila por jugador de la plantilla y una columna por cada estadística. Minutos/goles/asistencias/ocasiones/tarjetas se calculan y bloquean automáticamente mientras el modo en vivo está activo; el resto de columnas se rellenan a mano.
- Botón para añadir estadísticas personalizadas nuevas (a nivel de club, se aplican a todos los partidos). **Cualquier estadística nueva creada genera automáticamente su propio botón de un toque en el modo en vivo** (icono 📌 + nombre de la estadística), con el mismo comportamiento que Gol/Asistencia/Ocasión/Amarilla/Roja: se selecciona al jugador, se toca el botón, y queda registrada en el minuto correspondiente y en el registro del partido. Su casilla en la tabla se bloquea igual que las demás mientras el partido está en marcha.
- **Organizar estadísticas** ("🎨 Organizar estadísticas"): panel con todas las estadísticas del club (incluidas las 6 de siempre), donde cada una tiene:
  - **Color propio**, editable con un selector de color; se aplica al fondo del botón en el modo en vivo y a la cabecera de su columna en la tabla.
  - **Orden personalizado**, reordenable arrastrando (ratón o dedo) con desplazamiento en bloque del resto de filas mientras se arrastra (no un simple intercambio de posiciones). El orden se guarda automáticamente en Supabase (campo `orden`) y se refleja tanto en los botones del modo en vivo como en las columnas de la tabla, siempre sincronizados.
- Guardado automático en Supabase tras cada acción relevante (sin botón de guardar manual).
- Eliminar partido (con confirmación).

### 6.5 Ficha de jugador — ⏳ pendiente (diseño en 5 fases, ver apartado 9)

### 6.6 Equipo — ⏳ pendiente

### 6.7 Barra de navegación fija y botón de actualización — ✅ implementado (01/09/2026)
- La cabecera (`app-header`) y la barra de pestañas (`tabbar`) están envueltas en `#appTopBar`, con `position:sticky; top:0;` — quedan siempre visibles arriba de la pantalla, en cualquier pestaña y con cualquier posición de scroll, para poder cambiar de departamento sin perder de vista dónde se está.
- **Continuidad de estado al cambiar de pestaña:** como la app es una sola página que nunca se recarga, cambiar de pestaña (p. ej. de Partidos a Ficha de jugador y volver) **no reinicia nada** — el cronómetro del modo en vivo, los eventos registrados y los datos del formulario del partido se conservan tal cual, porque solo se oculta/muestra el panel con CSS (`display:none`), sin destruir las variables de JavaScript ni el intervalo del cronómetro. El cronómetro usa una marca de tiempo real (`runStartedAt`) en vez de contar "ticks", así que el tiempo mostrado es siempre exacto aunque haya estado en segundo plano.
- **Botón "🔄 Actualizar"** (junto a "Cerrar sesión"): pide al servidor una copia fresca de `index.html` (sin caché), compara su etiqueta de versión (`<meta name="app-version">`) con la que tiene cargada la página actual, y si son distintas recarga automáticamente; si son iguales, avisa de que ya está actualizado sin recargar. Se combina con cabeceras `Cache-Control: no-cache` en el propio HTML para que incluso un F5 normal traiga siempre la versión más reciente.
- La constante `CURRENT_APP_VERSION` se lee del `<meta name="app-version" content="...">` del `<head>`; **cada despliegue nuevo debe actualizar este valor** para que la comprobación de versión funcione.

### 6.8 Pantalla siempre encendida (Wake Lock) — ✅ implementado (01/09/2026)
Mientras la app está abierta con sesión iniciada, se usa la API estándar del navegador `navigator.wakeLock` para evitar que la pantalla se apague sola en móvil/tablet. Se solicita al entrar en la app (login, registro de club, `afterAuthChange`) y se libera al cerrar sesión. El sistema operativo libera el wake lock automáticamente si la pestaña deja de estar visible (cambias de app); se vuelve a pedir solo con que la pantalla vuelva a estar visible, sin que el usuario tenga que hacer nada. En navegadores sin soporte, no hace nada (sin errores).

### 6.9 Sincronización en tiempo real entre dispositivos — ✅ implementado (01/09/2026)
Si dos dispositivos tienen abierto el **mismo partido** a la vez (p. ej. dos entrenadores, o el mismo usuario en móvil y tablet), los cambios de uno (goles, minutos, estadísticas, eventos) aparecen automáticamente en el otro, sin recargar la página. Se usa Supabase Realtime (`supabase.channel(...).on("postgres_changes", ...)`) suscrito a la fila concreta del partido abierto (`matches` con `filter: "id=eq."+matchId`). Requiere que la tabla `matches` tenga activada la replicación (`alter publication supabase_realtime add table matches;`, ya aplicado). La suscripción se abre al seleccionar/crear un partido y se cierra al borrar un partido o cerrar sesión, para no acumular canales abiertos.

### 6.10 Alineación vinculada a un Partido — ✅ implementado (01/09/2026)
En la pestaña Alineación hay un selector **"Vincular esta alineación a un partido"**: se puede elegir "sin vincular" (comportamiento de siempre, solo para descargar la imagen), un partido ya existente de Partidos, o crear uno nuevo directamente desde ahí. Al pulsar **"💾 Guardar alineación para este partido"**, el once titular elegido se guarda en el campo `live.startersIds` de ese partido en Supabase (haciendo antes un `select` fresco del partido para no pisar datos si ya se hubiera empezado a jugar). Al abrir después ese mismo partido en Partidos, los jugadores ya aparecen marcados en la pantalla de "quién empieza en el campo", sin tener que volver a elegirlos uno a uno.
*Limitación conocida:* solo se guardan los **titulares**; los suplentes elegidos en Alineación no se restringen en Partidos (en Partidos, "banquillo" sigue siendo automáticamente cualquier jugador de la plantilla que no sea titular).

### 6.11 Acceso rápido a "Organizar estadísticas" desde el partido — ✅ implementado (01/09/2026)
Los botones "➕ Añadir estadística" y "🎨 Organizar estadísticas" (con su panel de color/orden) se movieron de la parte inferior de la página a justo encima del modo en vivo, para poder añadir o reordenar estadísticas rápidamente antes o durante un partido sin tener que desplazarse por toda la pantalla.

### 6.12 Estadísticas de equipo y de portero automático — ✅ implementado (01/09/2026)
La tabla `stat_columns` tiene ahora una columna **`scope`** (`'player'` | `'goalkeeper'` | `'team'`, por defecto `'player'`) que define cómo se anota cada estadística en el modo en vivo:
- **`player`** (normal, como siempre): hay que tocar antes a un jugador en "En el campo" y luego el botón de la estadística.
- **`goalkeeper`**: de un solo toque, sin elegir jugador — la app identifica sola quién juega de portero en ese momento (el jugador que está en el campo cuya posición incluye "portero", vía `findCurrentGoalkeeper()`) y anota el evento a su nombre. Usado en **"Parada nuestro portero"**.
- **`team`**: de un solo toque, sin ningún jugador asociado — solo cuenta el suceso y el minuto. Usado en **"Gol en contra"** y **"Ocasión en contra"** (no se pueden atribuir a un jugador propio porque son del rival). Se guardan como eventos `{type:"team_stat", key, minute}` (sin `playerId`) dentro de `live.events`, y su recuento se persiste en `stats.__team__[key]` (un "jugador" ficticio que no es ningún jugador real de la plantilla). Se muestran en un panel aparte, **"Estadísticas de equipo"**, con el contador y los minutos en los que ha ocurrido cada una — no aparecen como columna en la tabla por jugador, ya que no tendría sentido ahí.

Al crear una estadística nueva con "➕ Añadir estadística", ahora se pregunta qué tipo es (jugador / portero / equipo), para que esto funcione igual con cualquier estadística que se añada en el futuro, no solo con las tres ya creadas.

---

## 7. Incidencias ya resueltas (para no repetirlas)

| Fecha | Problema | Causa | Solución |
|---|---|---|---|
| 30/08/2026 | "No se ha podido subir el escudo: Bucket not found" | El bucket `crests` no se había creado en Supabase Storage | Ejecutar el SQL que crea el bucket + políticas (apartado 4.3) |
| 30/08/2026 | "Could not find the 'crest_url' column of 'clubs'" | Faltaba la columna en la tabla `clubs` | `alter table clubs add column if not exists crest_url text;` |
| 30/08/2026 | Importación de jugadores fallaba: "Too small: expected string to have >=1 characters" (caso 1) | Se insertaban cadenas vacías `''` en la columna `pos` | Usar `null` en vez de `''` cuando no hay valor |
| 30/08/2026 | Mismo error, persistía tras el cambio anterior | El texto se estaba escribiendo dentro del cuadro flotante de sugerencias de IA del editor SQL de Supabase, no en el editor real (la consulta enviada estaba vacía) | Cerrar el cuadro de IA (Esc/Cancel) y escribir directamente en el área de código |
| 30/08/2026 | Botón "Nuevo partido" no abría nada | El código enviaba `live: null` al crear el partido, pero esa columna es `NOT NULL` en la base de datos | Enviar `live: {}` en vez de `null`; además `ensureLive()` se corrigió para que también inicialice la estructura interna cuando `live` llega como objeto vacío `{}`, no solo cuando es `null`/`undefined` |
| 30/08/2026 | Alias de jugadores importados por SQL quedó vacío | El script de importación no incluía el campo `alias` | `update players set alias = split_part(name, ' ', 1) where alias is null or alias = '';` (mismo criterio que usaba la herramienta HTML anterior: alias = primer nombre) |
| 31/08/2026 | Las estadísticas personalizadas nuevas no se podían anotar con botón de un toque en el modo en vivo, solo a mano en la tabla | Los botones de acción del modo en vivo estaban fijados a solo 5 tipos (gol/asistencia/ocasión/amarilla/roja); `recomputeStatsFromLive` solo contaba esos mismos tipos | Se generalizó el sistema: cualquier columna de `stat_columns` que no sea una de las 6 fijas obtiene automáticamente su propio botón (evento `type:"custom"` con `key` = clave de la columna); el contador de estadísticas ahora es genérico por clave, no una lista fija |

---

## 8. Cómo reconstruir la app desde cero (resumen exprés)

1. Crear un proyecto en Supabase.
2. Ejecutar el SQL de creación de tablas (`clubs`, `players`, `stat_columns`, `matches`) con sus columnas exactas del apartado 4.1.
3. Activar RLS y crear las políticas del apartado 4.2 para las 4 tablas.
4. Crear el bucket `crests` y sus políticas (apartado 4.3).
5. Añadir la columna `crest_url` a `clubs` si no se creó en el paso 2.
6. Copiar las claves del proyecto (`SUPABASE_URL` y `SUPABASE_ANON_KEY`) dentro del `<script>` de `pizarra-app.html`.
7. Abrir `pizarra-app.html` (local o con `npx serve .`), registrar el usuario, crear el club, y volver a dar de alta la plantilla (a mano o con un script `insert into players (...)` como los ya usados).

---

## 9. Hoja de ruta / pendiente

- **Ficha de jugador**, en 5 fases (acordadas el 30/08/2026):
  1. Datos base del jugador (edad, salud, académico, pierna dominante… — ampliar tabla `players` o tabla nueva 1-a-1).
  2. Estadísticas automáticas agregadas desde `matches` (histórico de minutos/goles/etc. por temporada).
  3. Seguimiento/evaluaciones: tabla nueva tipo histórico, entradas fechadas por jugador (qué mejorar, objetivos).
  4. Fotos y vídeos adjuntos a cada evaluación (nuevo bucket de Storage; vigilar límite de almacenamiento del plan gratuito, especialmente con vídeo).
  5. Informe en PDF por jugador para enviar a las familias.
- **Equipo:** estadísticas agregadas de toda la plantilla/temporada.
- **Control de acceso avanzado** (aprobación de nuevos usuarios, cuotas de pago): aparcado a propósito, se retoma cuando el entrenador lo decida.

---

## 10. Alojamiento en producción y protocolo de actualización

**Desde el 31/08/2026, la app ya NO se distribuye como archivo descargable.** Vive en una dirección web fija, accesible para el usuario y para cualquier otro entrenador que la use:

- **URL pública:** https://pizarra-analista.onrender.com
- **Repositorio de código:** https://github.com/danielpelecha/pizarra-analista (archivo `index.html` en la raíz — es el mismo contenido que `pizarra-app.html`, solo renombrado porque Render sirve `index.html` por defecto)
- **Servicio en Render:** `pizarra-analista` (tipo *Static Site*, workspace `tea-daanrumgekts73eluaog`), configurado con auto-deploy activado sobre la rama `main`

### Cómo funciona una actualización, paso a paso

1. El usuario pide un cambio o mejora en la conversación con Claude.
2. Claude edita el código igual que siempre.
3. En vez de entregar el archivo para descargar, Claude lo sube directamente al repositorio de GitHub usando `git` desde su entorno de trabajo (clona el repo, sobrescribe `index.html`, hace commit y `git push`), autenticándose con un **Personal Access Token (classic)** de GitHub que el usuario generó una vez y que Claude conserva para las siguientes sesiones.
4. Render detecta el cambio en `main` automáticamente (auto-deploy) y publica la nueva versión en un plazo de segundos, sin intervención humana.
5. El usuario (o cualquier entrenador) simplemente recarga la página en `https://pizarra-analista.onrender.com` y ve la versión más reciente.

**El usuario no tiene que descargar, sustituir ni subir ningún archivo nunca más.**

### Por qué se llegó a esta solución (para no repetir el proceso de prueba y error)

- **Netlify** fue el primer intento: su conector MCP falló de forma repetida en la autorización (`error_code=mcp_token_exchange_failed`), incluso después de verificar que la cuenta de Netlify del usuario funcionaba perfectamente por otras vías (proyectos ya publicados con "Netlify Drop"). Se descartó tras varios reintentos infructuosos.
- **Vercel** se descartó sin llegar a conectarlo: su conector MCP solo expone herramientas de lectura/depuración de despliegues ya existentes (`list_projects`, `get_deployment`...), no tiene ninguna herramienta para crear o publicar un sitio nuevo.
- **Render** sí tiene una herramienta real de despliegue (`create_static_site`), pero esta *exige* un repositorio Git de origen — no admite subir un archivo suelto directamente.
- El conector MCP de **"Integración con GitHub"** de Claude se conecta correctamente (aparece con ✓) pero, comprobado con múltiples búsquedas de herramientas, no expone ninguna función de escritura (crear repositorio, subir/editar archivos) — solo parece pensado para otros usos dentro de Claude, no para automatizar despliegues.
- **Solución final:** el usuario generó manualmente un token de acceso personal de GitHub (Settings → Developer settings → Personal access tokens → Tokens classic → scope `repo`), y Claude lo usa a través de comandos `git`/`curl` en su propio entorno de terminal para crear el repositorio y publicar cambios — sin pasar por el conector MCP de GitHub, que no sirve para esto.

### Si el token de GitHub caduca o se revoca

Si en el futuro Claude no puede subir cambios (error de autenticación al hacer `git push`), el usuario debe generar un nuevo token classic con scope `repo` en https://github.com/settings/tokens/new y pasárselo a Claude en la conversación. Claude lo guardará de nuevo para las siguientes sesiones.

### Nota sobre el auto-deploy de Render (01/09/2026)

El auto-deploy de Render (publicar solo al detectar un `git push`) **no se disparó automáticamente** las dos primeras veces que se probó, probablemente porque el repositorio se conectó a Render pasando la URL directamente (vía API), sin pasar por el flujo propio de autorización de la GitHub App de Render, que es quien instala el aviso automático (webhook) en el repositorio.
**Solución adoptada:** tras cada `git push`, Claude llama también a `Render:trigger_deploy` para forzar la publicación manualmente. Esto es transparente para el usuario (no requiere ninguna acción suya) y forma parte del protocolo estándar de actualización descrito arriba — simplemente hay un paso técnico más en el lado de Claude.

## 11. Historial de cambios de este documento

- **30/08/2026** — Creación del documento maestro, con el estado de la app tras completar Club, Plantilla, Alineación (con ABPs y arrastre) y Partidos (modo en vivo completo), más el buscador de jugadores por autocompletado.
- **30/08/2026** — Ampliado el apartado 6.3 (Alineación) para detallar explícitamente el comportamiento del buscador de jugadores: filtrado por nombre/alias, bloqueo real (no solo visual) de jugadores ya usados en otro puesto mostrados en gris, y opción de vaciar un puesto.
- **31/08/2026** — Corregido Partidos: las estadísticas personalizadas ahora generan automáticamente su propio botón de un toque en el modo en vivo (antes solo las 6 fijas eran "ágiles"; las nuevas creadas por el entrenador solo se podían rellenar a mano). Añadido apartado a la tabla de incidencias.
- **31/08/2026** — Añadido "🎨 Organizar estadísticas": color y orden personalizables por estadística (columnas `color` y `orden` nuevas en `stat_columns`), con reordenación por arrastre (desplazamiento en bloque) sincronizada entre los botones del modo en vivo y las columnas de la tabla.
- **31/08/2026** — Renombrada la app de "PIZARRA" a **"PIZARRA ANALISTA"** en todos los textos visibles (título de pestaña, login, registro, cabecera, pie de página). Documentada la existencia de un conector MCP de Supabase disponible para ejecutar cambios en la base de datos sin SQL manual (no conectado por defecto).
- **31/08/2026** — Conectado el conector MCP de Supabase. Aplicada mediante `apply_migration` la migración pendiente de color/orden en `stat_columns` (antes preparada como script SQL manual). A partir de ahora los cambios de base de datos se ejecutan directamente, sin pasos manuales del usuario.
- **31/08/2026** — **Alojamiento en producción configurado.** La app ya no se distribuye como archivo descargable: vive en **https://pizarra-analista.onrender.com**, servida por Render a partir del repositorio de GitHub **`danielpelecha/pizarra-analista`** (archivo `index.html` en la raíz). El conector MCP de Netlify falló repetidamente en la autorización (error `mcp_token_exchange_failed`) y se descartó; el conector MCP de Vercel se descartó por no tener herramientas de despliegue (solo lectura/depuración). El conector MCP de "Integración con GitHub" de Claude aparece conectado pero no expone herramientas de escritura (crear repos, subir archivos) — para poder publicar de verdad, el usuario generó un **Personal Access Token (classic)** de GitHub con permiso `repo`, que Claude usa desde la terminal (bash + git) para crear el repositorio y subir cambios directamente. Ver apartado 10 para el protocolo de actualización completo.
- **01/09/2026** — Documento maestro conectado a la memoria del Proyecto vía integración GitHub de Claude Projects (sincronización con botón "Sync now", no automática por push — limitación conocida de Anthropic, ver apartado 10).
- **01/09/2026** — Barra de navegación (`#appTopBar`) ahora fija arriba de la pantalla (`position:sticky`) en cualquier pestaña y posición de scroll. Confirmado y documentado que cambiar de pestaña nunca interrumpe el modo en vivo de Partidos (cronómetro, eventos) gracias a la arquitectura de una sola página. Añadido botón "🔄 Actualizar" con comprobación de versión (`<meta name="app-version">`) y cabeceras no-cache. Detectado que el auto-deploy de Render no se dispara solo tras un `git push` (probablemente por no haber pasado por el flujo de autorización de su GitHub App) — Claude ahora llama a `Render:trigger_deploy` tras cada push, de forma transparente para el usuario.
- **01/09/2026** — Corregido el alias de jugadores: los 16 jugadores tenían `alias = null` en la base de datos real (el script de relleno de una sesión anterior nunca llegó a ejecutarse) — corregido con `execute_sql` vía el conector de Supabase. Corregido también el código: el fallback cuando no hay alias ahora usa el primer nombre (no el nombre completo) en los 5 sitios donde se mostraba (Alineación, canvas del campo, chips del modo en vivo, registro de eventos, tabla de estadísticas), centralizado en una función `displayAlias()`. Añadir/editar un jugador sin poner alias ahora lo rellena automáticamente con el primer nombre.
- **01/09/2026** — Separado el etiquetado del buscador de jugadores en dos funciones: la lista desplegable de resultados muestra el **nombre completo** (para distinguir jugadores con alias parecido), y el resultado ya seleccionado (y todo lo demás: campo, partido, tabla) sigue mostrando solo el **alias**.
- **01/09/2026** — 5 mejoras grandes: (1) pantalla siempre encendida en móvil/tablet (Wake Lock); (2) sincronización en tiempo real entre dos dispositivos mirando el mismo partido (Supabase Realtime); (3) la Alineación se puede vincular y guardar directamente sobre un Partido, precargando el once titular; (4) acceso a "Organizar estadísticas" reubicado junto al modo en vivo; (5) nuevo sistema de estadísticas por `scope` (`player`/`goalkeeper`/`team`) — "Parada nuestro portero" ahora es de un toque y se anota sola en el portero en el campo, "Gol en contra" y "Ocasión en contra" son de equipo (sin jugador), con panel propio de contadores y minutos.
