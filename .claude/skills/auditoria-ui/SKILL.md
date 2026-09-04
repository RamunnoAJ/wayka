---
name: auditoria-ui
description: Auditoría de los diseños UI y los flujos de UX de Wayka contra el contrato del proyecto (docs/ + design-system), con la app corriendo en web y en el emulador. Produce un informe priorizado por severidad y aplica los arreglos triviales. Usar antes de salir a producción y en cada pasada de pulido. Acepta un alcance opcional: todo, tutor, veterinario, clinica-admin, o una ruta concreta.
user-invocable: true
---

# Auditoría de UI y UX — Wayka

Alcance pedido: **el argumento con el que se invocó la skill** (vacío = todo: los tres roles
y todas sus pantallas).

Sos el auditor de diseño de Wayka antes de salir a producción con la clínica piloto. Tu
trabajo es comparar **lo que está construido** contra **lo que está contratado**, con
evidencia de la app corriendo, y no contra tu propio gusto.

> **El rubro no lo ponés vos.** Cada hallazgo tiene que citar la regla que viola y dónde
> está escrita. Si algo te parece feo pero ninguna regla lo prohíbe, no es un hallazgo:
> como mucho es una nota al pie del informe. Si una pantalla necesita una regla que no
> está en `docs/`, eso **sí** es un hallazgo — el vacío del contrato — y se reporta como tal.

---

## Fase 0 — Entorno y rubro

### 0.1 Levantar el entorno (en este orden; si algo falla, pará y decilo)

1. **Base y datos.** Postgres arriba (`backend/docker-compose.yml`), migraciones al día
   (goose), y el lote de desarrollo:
   `psql "$DATABASE_URL" -f backend/scripts/seed-desarrollo.sql`
   Deja tres cuentas, todas con contraseña `Test1234`:

   | Cuenta | Rol | Notas |
   |---|---|---|
   | `clinica@wayka.test` | clínica_admin de Wayka Centro | solo web |
   | `veterinario@wayka.test` | Lucía Ferreyra, con matrícula | web + móvil |
   | `tutor@wayka.test` | Valentina Ríos | tres mascotas, solo app |

   Sin este lote vas a auditar estados vacíos y nada más. Hay una segunda clínica
   (Wayka Norte) que existe para probar el alcance de permisos: nada suyo debería verse
   desde una cuenta de la primera.

2. **API.** `backend/cmd/api` corriendo, en background.

3. **Web** (clínica_admin y veterinario): `cd frontend && npm run web`, en background.
   Navegá con las herramientas de Chrome. Anchos a probar: **1440, 1024, 900, 768, 390**.

4. **Móvil** (tutor y veterinario): es la **única** forma de ver al tutor —
   `app/(tutor)/_layout.tsx` usa `alcanzableEnPlataforma={esNativo}` y, aun salteando el
   guard, el backend rechaza el canal `web` para un tutor (regla 2.3). No lo intentes por
   Chrome.

   ```
   ~/Android/Sdk/emulator/emulator -avd Pixel_7 &
   cd frontend && npm run android
   adb exec-out screencap -p > /tmp/auditoria-ui/<pantalla>.png
   adb shell input tap <x> <y> / adb shell input text "..."
   ```

   Si el emulador no arranca o el build nativo falla, **no simules el móvil en Chrome**:
   decilo, seguí con la parte web, y marcá los ejes móviles como no auditados.

5. **Chequeos mecánicos**, antes de abrir ninguna pantalla — son baratos y encuadran todo
   lo demás:

   ```
   cd frontend
   npm run typecheck
   npm run lint
   npm test
   ```

   `npm run lint` ya incluye las reglas de adherencia al design system: `eslint.config.js`
   lee los selectores de sintaxis de `design-system/_adherence.oxlintrc.json` —hex crudos,
   `px` crudos, fuentes fuera de Satoshi— y los aplica sobre `src/` y `app/`. Quedan afuera
   a propósito los selectores que listan los props válidos de cada componente: describen los
   originales React DOM y nuestros ports aceptan además los props de React Native. Eso el
   linter no lo cubre y **lo revisa el eje 1 leyendo**.

### 0.2 Leer el rubro (obligatorio, antes de juzgar nada)

- **`frontend/design-system/readme.md`** — completo. Es el rubro visual y de contenido:
  las tres reglas que ordenan el sistema, tono y copy, jerarquía tipográfica, escala de
  espaciado, estados obligatorios, patrones prohibidos, variantes deprecadas.
- **`docs/03-alcance-de-plataformas.md`** — la matriz rol × pantalla × plataforma y unos
  cuarenta requisitos de copy y de interacción redactados como obligación.
- **`docs/02-reglas-de-negocio.md` §4** — los procesos 4.1 a 4.23, que son los flujos.
- **`docs/11-sincronizacion-offline.md`** — estados offline del tutor.
- **`docs/12-telemetria-de-producto.md` §5** — los eventos que emite el cliente.
- **`frontend/CLAUDE.md`** y **`frontend/docs/09-design-system-integracion.md`**.

### 0.3 Dónde vive la UI

Las rutas de `app/` son cascarones finos (mediana ~20 LOC). **La UI real está en
`src/features/<dominio>/`**, y los componentes compartidos en `src/components/` (ports a
React Native de los 39 componentes DOM de `design-system/components/`, que ya no se
importan desde ninguna pantalla). Auditá ahí.

### 0.4 Deuda ya declarada — no la reportes como hallazgo nuevo

- `--color-primary-fill` del tema tutor (`#F6A56C`) con blanco encima da **2.0:1**, y
  `--color-primary-strong` (`#B85F2E`) da **4.45:1**. Los dos están bajo AA y los dos
  están aceptados por escrito (`frontend/docs/09` §7, `docs/04:127`). **Sí es hallazgo**
  usar ese fill en caption o en cuerpo chico, que el propio doc prohíbe: sobre el naranja
  el texto va en cuerpo grande y semibold.
- Iconos: `Icon` usa Lucide como sustitución declarada, no hay set propio.
- No hay Storybook, ni pruebas visuales, ni accesibilidad automatizada — es una decisión
  escrita en `frontend/docs/10` §6, no un olvido.

---

## Fase 1 — Seis ejes en paralelo

Lanzá **seis subagentes en paralelo, uno por eje** (una sola tanda de llamadas). A cada
uno pasale: el alcance pedido, las credenciales del lote, cómo sacar evidencia
(Chrome para web, `adb` para móvil), la sección 0.4 completa, su rubro, y el formato de
hallazgo de la Fase 3. **Ningún agente de esta fase edita código**: devuelven hallazgos.

Si el alcance es un solo rol, cada agente se limita a las pantallas de ese rol; los ejes
que no aplican devuelven lista vacía en vez de inventar trabajo.

### Eje 1 — Adherencia al design system
Rubro: `design-system/readme.md` §Visual foundations.
- Todo color y espaciado sale de `useTheme()` de `src/theme` — nunca un hex, un `px`
  suelto ni una custom property escrita a mano en un componente de `src/`.
- Jerarquía: H1 30/700 · sección 20/600 · título de card 17/600 · body 15/400 (17 en el
  móvil del tutor) · metadata 13/400 · label 12/500 · overline 11/700 (**la única
  versalita del sistema**). Mínimos: 12px web, 13px móvil.
- Escala base-4; padding de card 24; gutter de página 32; 40 entre secciones; contenido
  máximo 1160 centrado; sidebar 248.
- Radios de la escala (4/8/10/12/16/22/999). **Nunca esquina viva.** Toda card lleva
  borde de 1px: no flota solo con sombra.
- Sombras bajas y frías de la escala. **Nada de sombras interiores.**
- **Patrones prohibidos**: tarjeta con ícono y franja de acento a la izquierda (la
  excepción única es `CalendarWeek`); fondo tintado que no sea la alergia; más de dos
  fondos neutros por pantalla; más de un color de acción por pantalla; el primario claro
  como fondo de navegación (va `--surface-nav`).
- **Variantes deprecadas** que siguen funcionando como alias y no deberían usarse:
  `Button variant="accent"` → `primary`; `Badge tone="brand"|"accent"` → `primary`;
  `Card tone="clinical"|"owner"` → `default`.
- Estados: hover cambia el fondo, no la opacidad; press sin escala en web (`.97` en
  nativo); foco = borde + anillo; disabled = `--surface-disabled` + `--text-subtle`.
- El relleno sólido lleva `--color-primary-fill-fg`, nunca un blanco escrito a mano.
- Movimiento: un solo easing en web (140/220/340 ms); en nativo los tres resortes
  `snap`/`default`/`gentle`, sin rebote; un solo desplazamiento de entrada de 6px.

### Eje 2 — Estados obligatorios
Rubro: `design-system/readme.md` §Carga, error y vacío + `frontend/docs/10`.
Los tres son **obligatorios en todo bloque que dependa de datos**. Recorré cada pantalla
del alcance y verificá que existan y sean los correctos:
- **Carga**: `Skeleton`/`SkeletonText` con las medidas del contenido real. **Un spinner
  centrado es un hallazgo**, siempre. (Ojo: `GuardDeRol` usa `ActivityIndicator` —
  evaluá si corresponde ahí o si es el patrón filtrándose.)
- **Error**: `InlineError` dentro del bloque que falló, con reintento si la acción se
  puede repetir, y el resto de la pantalla usable. `Toast` es solo para avisos efímeros.
  El texto describe el problema y la salida, nunca "Error 500" ni culpa al usuario.
- **Vacío**: `EmptyState` con una acción que resuelva el vacío.
- **Sin permiso** y **sin conexión**: ver Eje 3.
Hoy `InlineError` aparece en 55 archivos, `Skeleton` en 29, `EmptyState` en 14 — buscá los
bloques con datos que no tienen ninguno de los tres.

### Eje 3 — Flujos por rol (recorrido real)
Rubro: `docs/03` §3–5 (matriz de pantallas) y `docs/02` §4 (procesos 4.1–4.23).
Recorré **en la app corriendo** cada pantalla del alcance, siguiendo el proceso de negocio
que la sostiene. Buscá: pantalla que existe y no está en el contrato; pantalla del
contrato que falta; paso que se desvía de lo escrito. Chequeos con nombre propio:
- Clínica_admin: cuatro secciones (Panel, Agenda, Plantel, Ajustes) y cerrar sesión en la
  barra, en ninguna sección. **No** da de baja una cita, **no** asienta la atención, **no**
  ve historial ni el tipo de cita en el tablero, y el número de documento no aparece en
  los resultados de la búsqueda de padrón. Las ausencias van en el menú de la fila del
  Plantel, no en sección propia.
- Veterinario: **paridad total** web/móvil, feature por feature. El asiento de la atención
  (3.3.1) es el caso principal en teléfono. "Mi cuenta" va en el menú, no colgada de un
  avatar.
- Tutor: tres pestañas (Mascotas, Calendario, Ajustes). Notificaciones **no** es pantalla
  (es un interruptor en Ajustes) y adjuntos **no** es pestaña (cuelga de la ficha). Las
  invitaciones pendientes van **arriba del listado de Mascotas**, con contador en esa
  pestaña. Compartir es solo del dueño.
- **Nivel del tutor** (dueño / co-tutor edición / co-tutor lectura, `docs/02:213-222`): la
  matriz que el doc de pantallas no tabula. El de lectura no ve ni el peso, y en "Quién la
  ve" ve la lista sin acciones.
- **Offline del tutor** (`docs/11`): "sincronizado" **no se muestra** — un cartel verde
  permanente es un hallazgo. Pendiente se muestra aplicado y marcado como no confirmado.
  Rechazado muestra el motivo **de negocio** y el valor del servidor. Compartir y revocar
  **no se ofrecen** sin señal. El paso de antecedentes desde el onboarding avisa que
  necesita conexión **antes** de que el tutor escriba. La dirección degrada a texto libre.
  El aviso de mascotas compartidas caducadas **no las nombra**.
- Interacciones especificadas que suelen romperse: las horas inválidas **no se ofrecen**;
  no se ofrecen profesionales ocupados ni ausentes; el selector de antecedente son cuatro
  tarjetas con vacuna encabezando y **sin preselección**; la fecha se muestra como se
  declaró ("marzo de 2023"), nunca un 1 de enero; la barra de progreso del alta no arranca
  en cero; guardar un antecedente vuelve al selector, no a la ficha; marcar una foto de
  perfil desmarca la anterior sin preguntar; toque y toque largo hacen lo mismo.

### Eje 4 — Copy y contenido
Rubro: `design-system/readme.md` §Content fundamentals + la lista de `docs/03`.
- Español rioplatense, voseo ("Cargá", nunca "Carga" ni "Cargue"). Wayka no habla en
  primera persona ni se nombra en la interfaz.
- Mayúscula solo inicial, siempre, botones incluidos. Overline es la única versalita.
- Coma decimal, unidad separada por espacio, cifras en `tabular-nums`. Fechas cortas en
  minúscula ("12 mar 2026").
- Emoji: prácticamente no. Nunca en la web, nunca en un dato clínico, nunca en un botón.
- Registro por audiencia: al veterinario se le habla en **datos**; al tutor en
  **consecuencias**.
- **Nunca "eliminar"** cuando la baja es lógica — el dato sigue existiendo.
- **Mensajes que deben ser indistinguibles** (anti-enumeración): recuperar contraseña
  responde lo mismo exista o no la cuenta; token inexistente/vencido/usado, un mismo
  mensaje genérico (activación, recuperación, confirmación de correo, invitación); en el
  login, email sin cuenta / contraseña incorrecta / email mal formado / cuenta solo-Google
  dan **el mismo rechazo** (la cuenta inactiva sí se distingue); permiso denegado, genérico.
- **Copy que debe decir algo concreto**: el alta de veterinario dice por qué no pide
  contraseña; antes de guardar una ausencia se dice cuántas citas caen en el rango, y que
  darla de baja no las devuelve; un día sin franjas dice **"día cerrado"** con esas
  palabras; la previsualización de la grilla dice qué citas quedarían afuera **antes** del
  intento; compartir con una clínica avisa que verá el historial completo, incluido lo de
  otras; cada nivel de co-tutor se explica en una línea debajo de su nombre, **no en un
  tooltip**; revocar dice la verdad sobre el efecto; salir del paso de antecedentes dice
  qué queda sin resolver; si falla la foto, la pantalla dice que la mascota quedó creada
  igual; el resumen dice "la ficha de [nombre]"; la ficha de ejemplo dice que es un
  ejemplo; la política de contraseña está a la vista mientras se escribe.
- **Nunca**: "no encontramos ese correo"; el documento en resultados de búsqueda; el token
  de invitación en un listado; pedir motivo de ausencia; "desde siempre" en el tablero;
  descargar o compartir desde el visor de adjuntos; "editar" en el resumen de antecedentes
  (solo quitar); mostrar el estado "sincronizado".

### Eje 5 — Accesibilidad y contraste
Rubro: `frontend/docs/09` §7 + `readme.md` §Foco/Estados. Ojo: **el contrato de producto
casi no habla de accesibilidad** (solo movimiento reducido en el visor y el contraste del
correo) — ese vacío es en sí un hallazgo a reportar una sola vez, no por pantalla.
- Nombre accesible en todo control: hoy hay 73 `accessibilityLabel`, 47
  `accessibilityRole`, 28 `accessibilityState` y **solo 3 `accessibilityHint`** repartidos
  en 50 archivos. Buscá los controles sin nombre — la regla de `docs/10` dice que un
  `testID` casi siempre delata un elemento sin nombre accesible.
- Objetivo táctil ≥ `--control-h-touch` (52px) en móvil.
- Foco: un solo anillo, aplicado desde `styles.css`; los componentes no lo repiten. Sobre
  cualquier contenedor `data-surface="dark"` el anillo pasa a blanco. En nativo el foco es
  borde de 2px en `--border-focus`. Verificá el recorrido de tabulación completo en web.
- Movimiento reducido: `useReducedMotion()` en **todo** hook de animación, y el visor de
  adjuntos abre sin animación con la imagen fija (`docs/03:238`).
- Contraste: medí las combinaciones reales de cada pantalla, con el tema del rol puesto
  (`data-theme="tutor"` invierte primario y acento). Aplicá la sección 0.4.

### Eje 6 — Paridad, responsive y telemetría
- **Paridad web/móvil del veterinario**, feature por feature. Es el único rol con dos
  canales, y una paridad rota es un hallazgo bloqueante.
- **Guards**: `(clinica-admin)` solo web, `(tutor)` solo nativo, `(veterinario)` ambos.
  El guard es de navegación, no de seguridad — verificá que no muestre una pantalla a
  medias antes de rebotar.
- **Responsive**: el corte es por **ancho, no por plataforma** — un solo breakpoint,
  `ANCHO_PARA_BARRA_LATERAL = 900` en `src/features/navegacion/Shell.tsx`, sidebar arriba
  y `MobileTabBar` abajo. Probá 1440/1024/**901**/**899**/768/390 y buscá lo que se rompe
  en el medio. Prestá atención al fallback `ancho === 0` (export estático / SSR), que
  asume sidebar.
- **Telemetría** (`docs/12` §5): el cliente emite exactamente siete eventos —
  `carga_evento_abierta`, `carga_evento_abandonada` (con `duración_ms`, el único
  cronómetro del catálogo), `pantalla_vista`, `app_abierta_desde_push`,
  `notificaciones_desactivadas`, `paso_de_antecedentes_resuelto`, `sesion_servida_offline`.
  Verificá que estén instrumentados, que **no** emita ninguno de los del backend, que
  ninguna propiedad lleve `paciente_id`, nombre ni texto libre, y que la ingesta nunca
  bloquee al usuario.

---

## Fase 2 — Verificación adversarial

Todo hallazgo marcado **bloqueante** pasa por un agente verificador que abre el archivo y
la pantalla e intenta refutarlo. Se descarta el hallazgo que:
- no cita `archivo:línea`, o
- no cita la regla del rubro que viola y dónde está escrita, o
- no tiene captura que lo muestre (cuando es visual), o
- resulta ser deuda de la sección 0.4.

Consolidá los seis informes en uno: mismo hallazgo reportado por dos ejes se fusiona.

---

## Fase 3 — Informe y arreglos triviales

### Formato de cada hallazgo

```
[BLOQUEANTE|DEBERÍA|PULIDO] eje N — <título en una línea>
  Dónde:  frontend/src/features/.../Archivo.tsx:123  ·  pantalla: <rol> / <ruta>
  Regla:  <la regla, citada> — <archivo del rubro>:<sección>
  Qué pasa: <lo observado, con la captura>
  Arreglo: <concreto>
```

Severidades: **Bloqueante** = impide salir a producción (flujo roto, dato del contrato que
no se ve, copy que muestra algo que el contrato prohíbe mostrar, paridad rota).
**Debería** = rompe el rubro sin romper la función. **Pulido** = cosmético.

### Arreglos que aplicás vos, en el momento

Solo los triviales y sin riesgo, y los marcás como aplicados en el informe:
- hex o `px` crudo → el token correspondiente de `useTheme()`;
- `accessibilityLabel` / `accessibilityRole` faltante;
- variante deprecada → la vigente;
- copy que viola una regla textual (casing, voseo, "eliminar" por una baja lógica, punto
  decimal en vez de coma).

**Todo lo demás queda como propuesta**, sin tocar. Un cambio de layout, de flujo o de
componente no es trivial aunque parezca chico.

Cerrá corriendo `npm run typecheck && npm run lint && npm test` en `frontend/`, y reportá
el resultado tal cual. Si algo se pone rojo, revertí el arreglo que lo rompió y anotalo.
`npm run lint` arranca con warnings de adherencia preexistentes: contalos antes de tocar
nada y reportá el delta, no el total.

### Cierre

Terminá con un resumen de una pantalla: cuántos hallazgos por severidad y por eje, qué se
aplicó, qué falta, y **una línea con lo que falta para poder salir**. Si algún eje no se
pudo auditar (emulador caído, backend abajo), decilo ahí en vez de dejarlo implícito.
