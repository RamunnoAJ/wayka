# Wayka — Alcance de Plataformas

MVP — Web de gestión + Aplicación móvil
Versión 1.0 · Complementa a Modelo de Datos y Reglas de Negocio

## 1. Definición de plataformas

Wayka se compone de dos productos para el MVP: una aplicación web de gestión, de uso exclusivo de la clínica, y una aplicación móvil que sirve tanto al veterinario como al tutor, con permisos distintos según el rol autenticado.

| Plataforma | Quién accede | Propósito |
|---|---|---|
| Web | Veterinario, Clínica_admin | Gestión completa del día a día clínico y administrativo. |
| Móvil | Veterinario (paridad con web), Tutor (acceso exclusivo) | Para el veterinario: misma herramienta que la web, disponible fuera de la clínica. Para el tutor: su único acceso al producto, y el lugar donde recibe avisos, saca fotos y lee sin conexión. |
| Línea de comandos | Administrador de la plataforma | Alta de una Clínica junto con su cuenta clínica_admin, y emisión del token de activación con el que esa cuenta define su contraseña (Reglas de Negocio, 4.10). No es una interfaz de usuario del producto: es una herramienta de operación, fuera de la API HTTP. |

> **El clínica_admin no tiene acceso a la aplicación móvil**, y esa restricción se valida en el backend, no solo ocultando la opción en la interfaz — mismo criterio aplicado al motor de permisos en Reglas de Negocio.
>
> **El tutor tampoco entra por la web**, y la restricción se valida en el mismo lugar y con el mismo criterio. Estuvo habilitado un tiempo para el que no tiene el teléfono a mano; se volvió atrás porque su producto **es** la aplicación: los avisos (5.5), la cámara (5.6) y la copia local que le da acceso sin conexión (Sincronización sin Conexión) dependen del aparato, y una web que se los da a medias no es un canal alternativo sino una versión peor de lo mismo.

## 2. Matriz de plataforma por rol

| Rol | Web | Móvil |
|---|---|---|
| Clínica_admin | Acceso completo | Sin acceso |
| Veterinario | Acceso completo | Acceso completo (paridad total con la web) |
| Tutor | Sin acceso | Acceso completo |

> La paridad entre web y móvil implica que ambas plataformas comparten el mismo conjunto de funcionalidades para ese rol — la diferencia entre plataformas es de dispositivo/contexto de uso, no de permisos ni de features disponibles.

## 3. Pantallas mínimas — Web

### 3.1 Login
- Autenticación por email + contraseña.
- Rechaza el acceso si el usuario autenticado es de tipo tutor (regla de canal).

### 3.1.2 Recuperar la contraseña (sin sesión)
- Se alcanza desde el ingreso, y también desde el enlace que llega por correo.
- **Pedir el enlace**: se escribe el correo y el sistema manda un token de un solo uso. La pantalla responde **lo mismo exista o no la cuenta** — decir "no encontramos ese correo" convertiría la pantalla en una forma de averiguar qué direcciones están registradas, que es justo lo que un padrón de tutores no debería revelar.
- **Definir la nueva**: se abre con el token del enlace, pide la contraseña nueva con la política de la regla 2.1 a la vista, y **cierra todas las sesiones abiertas** de esa cuenta.
- Al canjear **no se entra**: el canje no emite sesión, igual que la activación (3.1.1). La pantalla lleva al ingreso.
- Un token inexistente, vencido o ya usado se rechaza con un mismo mensaje genérico.
- **Si quien llegó es tutor o veterinario, la pantalla ofrece abrir la app** con una tira arriba, y sigue funcionando entera para quien no la tenga o esté en la computadora (Arquitectura, 3.8). Al clínica_admin no se la muestra: no puede entrar desde móvil.
- Es, junto con el ingreso y la activación (3.1.1), una de las pantallas de la web alcanzables sin sesión. El registro de tutor (5.1) no está en esa lista: es de la app.

### 3.1.1 Activación de la cuenta (sin sesión)
- Se abre con el token de activación: al clínica_admin se lo entrega el administrador de la plataforma en mano, al veterinario le llega por correo cuando su clínica lo da de alta (procesos 4.12 y 4.16 de Reglas de Negocio). La pantalla es la misma para los dos y no distingue de dónde vino el token.
- Pide únicamente la contraseña nueva, con la política de la regla 2.1 a la vista mientras se escribe.
- Un token inexistente, vencido o ya usado se rechaza con un mismo mensaje genérico: la pantalla no ayuda a distinguir cuál de los tres fue.
- Al activar **no se entra**: el canje no emite sesión. La pantalla lleva al login, donde se estrena la contraseña recién definida.
- Al veterinario, que llega desde el correo, la pantalla le **ofrece abrir la app**; al clínica_admin, que pega el token a mano en la web, no (Arquitectura, 3.8).
- Es, junto con el ingreso y la recuperación de contraseña (3.1.2), una de las pantallas de la web alcanzables sin sesión.

### 3.2 Panel de clínica (rol: Clínica_admin)

**Son cuatro secciones de la barra de navegación, no una sola pantalla**: Panel, Agenda, Plantel y Ajustes. La primera versión de este documento decía lo contrario —"la pantalla entera del rol, un tablero arriba y la gestión abajo"— y valía cuando el rol solo tenía tres formularios de configuración.

El corte es por **frecuencia de uso** y no por afinidad temática. El panel y la agenda se miran todos los días; el plantel, cada tanto; y **Ajustes** junta lo que se configura una vez —los datos de la clínica y el horario de atención— más la cuenta propia.

> **Las ausencias no tienen sección**, y no es que no quepan: **una ausencia es de una persona**, y el lugar donde se la busca es su fila. Se cargan desde el menú de esa fila en el Plantel y se listan en su ficha. Lo que se perdía con eso es la mirada transversal —quién falta hoy—, y la repone la fila del listado, que dice "ausente hasta el 14" con el mismo criterio con el que ya avisa quién no tiene matrícula.

> **Cerrar sesión no está en ninguna de las cinco.** Vive en la barra de navegación, al lado del nombre de quien está adentro: es donde se lo busca, y donde ya estaba para los otros roles.

> **La barra lateral no se cambia por la inferior, por angosta que quede la ventana.** El resto de la web corta por ancho —el veterinario tiene paridad con el móvil y una ventana chica es un teléfono—, pero este rol no tiene canal móvil (sección 2) y su salida de sesión vive solo en esa barra: cambiarla por la inferior lo dejaba adentro sin forma de salir, y no hace falta un teléfono para llegar ahí —un zoom al 150 % en un portátil de 1366 px ya da 911—. Por debajo del corte la página **se desplaza en horizontal** en vez de recomponerse: para el rol que no tiene una versión angosta, desplazar deja los datos alcanzables y recortarlos no.

#### 3.2.1 Panel — el tablero

Cuatro bloques de **conteos**. Ninguno lista registros ni nombra una mascota: el rol no alcanza el historial clínico y un listado de atenciones con hora y profesional lo reconstruye por el costado (Modelo de Datos, 5).

- **Ocupación de la grilla** — turnos ocupados sobre turnos disponibles del período, y el mismo par por profesional. Los disponibles salen de las Franjas de atención y de la duración del turno (3.2.4); los ocupados, de las Citas pendientes. Es lo único del producto que cambia todos los días para este rol, y la razón por la que el tablero existe.
- **Sin asignar** — cuántas Citas del período no tienen profesional. Es la cola de lo que hay que repartir (3.6), y crece sola cuando se carga una ausencia (3.2.3).
- **Atenciones** — cuántas Consultas atendidas se asentaron en el período, por profesional y por origen (agendada / espontánea / urgencia). Es el volumen de trabajo real, que no coincide con la agenda: la mayoría de las atenciones de una veterinaria no estaban agendadas.
- **Cartera** — pacientes con vínculo vigente, y cuántos entraron en el período separados por vía (alta de la clínica / compartido por el tutor).

Es lo único de la sección que se mira todos los días, y por eso es lo único que hay en esta pantalla.

> **Toggle semana / mes, en todo el tablero a la vez.** Un control único arriba y no uno por bloque: comparar la ocupación de la semana contra las atenciones del mes es leer dos cosas que no se corresponden, y dos controles invitan justo a eso. El período se lee en la `zona_horaria` de la clínica (Modelo de Datos, 4.3).
>
> **No hay "desde siempre".** Todo conteo va contra un período, porque un número sin denominador no se puede interpretar: "412 atenciones" no dice nada, "412 este mes contra 380 el anterior" sí.
>
> **No se muestra el tipo de la Cita** (vacuna, control, cirugía). Es un conteo de ocupación, no de qué se hace: el tipo por paciente empieza a parecerse al historial, y para saber si la agenda se está usando alcanza con cuántos turnos están tomados.
>
> **No hay cobertura de historial** —qué porcentaje de las atenciones terminó con un evento clínico cargado— y no es un olvido: eso mide el desempeño de una persona, no la operación de la clínica, y se decidió dejarlo fuera del alcance de este rol. Se sigue calculando para el piloto por SQL (Telemetría de Producto, 9).

#### 3.2.2 Agenda

- El calendario de la clínica, **el mismo que ve el veterinario** (3.6): por día, semana y mes, con la mascota, el tipo de cita y quién atiende.
- **Agendar un turno nuevo**, sobre la grilla de la clínica (3.2.4) y con las mismas reglas que usa el veterinario. Para elegir la mascota hay una **búsqueda acotada a la cartera**: nombre, especie y a qué tutor llamar, sin abrir ninguna ficha.
- **Dar de alta la mascota que todavía no está**, desde la misma búsqueda: si nadie coincide, se carga ahí. Es el cliente nuevo que llama para pedir el primer turno, y hasta acá el mostrador tenía que esperar al veterinario para poder tomárselo.
  - Antes de darlo de alta hay que no encontrarlo, y el padrón se busca **también por documento**: es el dato que el mostrador tiene más a mano cuando alguien llama. El número no se muestra en los resultados —el rol no lo lee (Reglas de Negocio, 3.2)—; sirve para encontrar la ficha, no para leerla.
  - Si el tutor no está en el padrón, se lo da de alta con **nombre, contacto y consentimiento**. Documento y dirección los completa el veterinario cuando atiende (proceso 4.1).
  - El **número de chip no se carga acá**: lo pone el veterinario, que es quien lo implanta y lo lee.
- **Reagendar** una cita pendiente: fecha, hora y si se le avisa al tutor.
- **Asignar y desasignar profesional**, que es la tarea que el tablero ya señalaba sin dejar hacer: el contador de "sin asignar" (3.2.1) es la cola de lo que hay que repartir. No se ofrece a quien ya tiene otra cita a esa hora ni a quien tiene una ausencia cargada (3.2.3).
- **No da de baja una cita**: retirarla del calendario es decir que no va a ocurrir, y eso lo sabe quien atiende o quien lleva a la mascota.
- **No asienta la atención.** Eso es afirmación asistencial y la hace quien atendió (3.3.1).

> La primera versión de este documento no le daba agenda a este rol, con el criterio de que el calendario cuelga del Paciente. Se cambió porque una Cita no lleva diagnóstico: dice cuándo viene quién, que es la pregunta del mostrador y la tarea de quien administra la clínica. Lo que sigue afuera es el historial, la medicación, los adjuntos y las fichas.
>
> La segunda versión le daba **solo la asignación**, con el argumento de que agendar y mover un turno son criterio clínico. Ese criterio está reservado frente al **tutor**, y el clínica_admin es la clínica: recepción toma y mueve turnos todo el día. Lo que quedó afuera es solo la baja.

#### 3.2.3 Plantel

- Alta, edición y baja lógica de Veterinarios asociados a la clínica. La ficha y la cuenta de acceso se crean juntas, en una sola operación (proceso 4.12 de Reglas de Negocio), y la baja de la ficha desactiva la cuenta (4.13).
- **El formulario de alta no pide una contraseña**, y tiene que decir por qué: al veterinario le llega un correo con un enlace para definir la suya (4.12). Sin esa línea, quien completa el alta se queda esperando un campo que no está, o peor, cree que la cuenta ya quedó lista para usar.
- Desde la ficha de quien todavía no estrenó su cuenta se **reenvía el correo de activación**. Es la salida del token que se venció o del correo que no llegó, y sin ella la única sería que el administrador de la plataforma emita otro por línea de comandos. Una cuenta sin estrenar se reconoce porque no tiene ningún método de autenticación configurado.
- **Buscador por nombre, documento o matrícula**, con el mismo criterio de campo único que la búsqueda de tutores (3.3). No está por el tamaño del plantel —una clínica entra en una pantalla— sino porque es donde se responde "¿este número ya está cargado?" antes de que el alta lo rechace: la matrícula es única en todo el sistema (Reglas de Negocio, 2.1) y el error de guardado no dice de quién es la que colisiona.
- El listado avisa **quién no tiene matrícula cargada**. Sin matrícula el veterinario queda en modo restringido y no puede escribir historial (Reglas de Negocio, 2.2) — hoy eso se descubre cuando la persona intenta cargar un evento y no puede, que es el peor momento y el peor lugar para enterarse.
- Estado de cada cuenta: activa, suspendida o dada de baja.
- **Cargar una ausencia**, desde el menú de la fila de esa persona (Modelo de Datos, 4.19): un rango con fecha y hora. Sirve para que la grilla no le ofrezca turnos a quien no va a estar, y va acá y no en una sección aparte porque **una ausencia es de alguien**. La ficha lista las suyas y ahí se dan de baja.
  - **No se pide un motivo**, y la pantalla no tiene dónde escribirlo. Es deliberado: el motivo de la ausencia de un empleado puede ser un dato de salud, y para que la agenda funcione alcanza con el rango.
  - Antes de guardar, la pantalla dice **cuántas citas asignadas caen adentro del rango**. Al guardar, esas citas quedan **sin profesional** —no se cancelan ni se mueven de hora— y pasan a la cola de sin asignar (3.6). La ausencia se guarda siempre: el diálogo informa el efecto, no pide permiso para dejar de bloquear.
  - Dar de baja una ausencia **no devuelve las citas** a quien las tenía. La pantalla lo dice, porque lo contrario es lo que cualquiera esperaría.
- El listado dice **quién no está hoy y hasta cuándo**. Es la mirada transversal que la ficha no da: sin eso, saber quién falta obligaría a abrir el plantel entero ficha por ficha.
- **Restablecer la contraseña de una cuenta del plantel**, desde la ficha de esa persona. No exige conocer la anterior. No es la única salida del veterinario que olvidó la suya —para eso está la recuperación sin sesión (3.1.2)—, sino la que no depende del correo: sirve cuando el enlace no llega o la persona ya no tiene acceso a esa casilla. La contraseña nueva se la comunica el administrador por un medio propio, fuera del sistema.

#### 3.2.4 Ajustes

Los datos de la clínica, el horario de atención y la cuenta propia. Van juntos porque comparten frecuencia: se configuran una vez y casi no se vuelven a tocar.

- Edición de datos administrativos de la Clínica: nombre, dirección con confirmación en el mapa, contacto.
- Cambio de la contraseña de la propia cuenta.
- La clínica y su propia cuenta de administrador **no se crean desde acá**: las da de alta el administrador de la plataforma (proceso 4.10 de Reglas de Negocio).
- Sin acceso a historial clínico, medicación, adjuntos, fichas de paciente ni fichas de tutor.

**Horario de atención**

- Edición de las **Franjas de atención** de la clínica (Modelo de Datos, 4.18) y de la duración del turno. Los dos definen la grilla con la que el veterinario agenda, así que un cambio acá cambia qué horas son válidas en el calendario de toda la clínica.
- **La duración del turno se edita acá y no con los datos administrativos**, aunque sea un campo de la Clínica y no de la Franja: se valida contra las franjas —un turno que no divide un tramo se rechaza— y tener un control en una pantalla y el otro en otra dejaría un error que no se puede corregir sin cambiar de pantalla.
- Se edita **la semana entera y se guarda de una vez**: la grilla se valida completa (que ninguna franja se solape, que el turno divida a cada una) antes de aceptarse.
- Un día **sin ninguna franja es un día cerrado**, y la pantalla lo dice con esas palabras en vez de dejar el día vacío y ambiguo. Dos franjas el mismo día son el **corte de mediodía**: el hueco entre las dos es lo que las hace dos y no una.
- **Previsualización antes de guardar**: cuántos turnos por día produce la grilla nueva, y qué Citas pendientes quedarían afuera. El horario no se puede achicar mientras esas citas existan (regla 2.2), y la pantalla tiene que decir cuáles son **antes** del intento, no como el texto de un error — corregir la grilla a ciegas hasta que el guardado deje de fallar no es editar un horario.

### 3.3 Gestión de pacientes (rol: Veterinario)
- Lectura del plantel de la propia clínica (sin poder modificarlo), para resolver quién firmó cada registro clínico.
- Alta de paciente (con alta de tutor si no existe, según proceso 4.1 de Reglas de Negocio).
- Búsqueda de tutores por documento, nombre o contacto, **en un solo campo**: se tipea lo que la persona diga —un apellido, un teléfono, un DNI— y el buscador resuelve contra los tres (Reglas de Negocio, 4.1). Alta y edición de la ficha de un tutor, incluido completar documento y dirección — la dirección con el mismo autocompletado y mapa que usa el tutor en su ficha propia (5.8). El listado se pagina siempre: no está acotado a la propia clínica.
- Baja lógica de una ficha de tutor.
- Búsqueda y listado de los pacientes vinculados a la propia clínica.
- Ficha de paciente: datos básicos, historial de eventos clínicos, medicación activa e histórica. **Cada registro dice quién lo declaró**: los que cargó el tutor como antecedente van marcados de forma inequívoca junto a los del profesional, en la misma lista y no en una solapa aparte (Modelo de Datos, 4.5). El veterinario los edita y los da de baja como a los propios.
- **Quiénes son sus tutores**: el dueño y los co-tutores con acceso, con su contacto. Antes había uno solo y era implícito; ahora hay que saber a quién llamar.
- **Qué otras clínicas la atienden**, solo el nombre. Es continuidad de cuidado —no repetir una vacuna que puso otra—, no una ventana a la cartera ajena.
- **Dejar de atenderla**: revoca el vínculo de la propia clínica y la saca de la cartera, sin tocar el registro. Se rechaza si quedan citas pendientes. El veterinario **no da de baja la mascota**: eso es del dueño (Reglas de Negocio, 2.4).

### 3.3.1 Registro de la atención (rol: Veterinario)
- **Asentar que se atendió**, en un toque: desde la agenda del día sobre la cita, o desde la ficha del paciente cuando nadie la agendó (ahí se elige el origen: espontánea o urgencia). Es el hecho asistencial, y es independiente de cargar el historial — se asienta al atender, aunque el evento clínico se escriba a la noche (Reglas de Negocio, 4.21).
- Por defecto atiende quien asienta; se puede indicar a otro profesional del plantel.
- **Atendidas de hoy, y cuáles no tienen historial cargado**: la lista de trabajo pendiente del día. Es lo único que hace que asentar valga la pena para quien lo hace, y no solo para quien mira las métricas.
- Corregir o dar de baja un asiento cargado por error.

### 3.4 Carga de evento clínico (rol: Veterinario)
- Formulario por tipo de evento (consulta, vacuna, cirugía, control, urgencia). Si se entra desde una atención asentada (3.3.1), el evento queda vinculado a ella sin que haya que elegir nada.
- Campos estructurados obligatorios para vacunas, medicación y alergias (según nota 4.5 del Modelo de Datos). Para el veterinario rige la forma completa —el lote y la severidad incluidos—: lo que se afloja con el origen `tutor` no se afloja acá.
- Adjuntar archivos (foto, PDF, estudio) al evento.

### 3.5 Gestión de medicación (rol: Veterinario)
- Alta de medicación activa, con bloqueo si ya existe una activa de la misma droga (regla 2.2).
- Cierre de medicación (fecha_fin).
- Vista rápida de medicación activa por paciente.

### 3.6 Calendario / Citas (rol: Veterinario)
- Creación de citas (vacuna, control, cirugía programada) con **fecha y hora**, sobre la grilla que definen las franjas de atención y la duración del turno de la clínica (3.2.4). Las horas fuera de la grilla no se ofrecen: el backend las rechaza igual, pero ofrecerlas y después fallar es un error que la interfaz puede evitar.
- La cita atendida se marca como cumplida desde la agenda, asentando la atención (3.3.1). No hay un control de "cumplida" aparte: cumplir una cita es haber atendido.
- Vista de citas pendientes y vencidas de la clínica, por día, semana y mes, con quién atiende cada una. Son las citas agendadas **en esta clínica**: una mascota atendida también en otra tiene allá su propia agenda, que acá no se ve.
- **Asignación de profesional**, opcional: una cita puede quedar de la clínica y repartirse después. Al asignar, no se ofrecen los profesionales que ya tienen otra cita en ese momento **ni los que tienen una ausencia cargada** (3.2.3) — el backend lo rechaza igual, pero ofrecerlo y después fallar es un error que la interfaz puede evitar. Si no queda ninguno disponible, la cita se agenda igual sin profesional: que no haya quién la atienda todavía no es motivo para no tomar el turno.
- Filtrar la agenda por profesional, incluyendo "sin asignar": es la lista de lo que todavía hay que repartir.

### 3.7 Mi cuenta (rol: Veterinario)
- Lectura de los propios datos: nombre, matrícula y correo. **No son editables desde acá** — los carga el clínica_admin al dar de alta la cuenta (proceso 4.12), y la matrícula además decide si el veterinario puede escribir historial (regla 2.1): cambiársela a sí mismo sería cambiarse los permisos.
- Cambio de la contraseña de la propia cuenta.
- Existe porque el veterinario era el único rol sin ningún lugar donde vivieran sus datos: el tutor los tiene en "Mis datos" (5.8) y el clínica_admin en el panel (3.2). Va en el menú y no colgada de un avatar porque el rol tiene paridad entre web y móvil, y en el teléfono no hay avatar donde colgarla.

## 4. Pantallas mínimas — Móvil (Veterinario)

Mismo conjunto funcional que las secciones 3.3 a 3.6 —incluido el asiento de la atención (3.3.1), que en el teléfono es el caso principal: se atiende parado al lado de la mesa, no sentado frente a la computadora—, adaptado a formato móvil. No se listan de nuevo por tratarse de las mismas funcionalidades con paridad total.

> Nota de diseño: al tener paridad completa, conviene evaluar en la etapa técnica si conviene un codebase compartido (ej. framework multiplataforma) para el veterinario, en vez de mantener dos implementaciones separadas del mismo alcance funcional.

## 5. Pantallas mínimas — Tutor

Son de la aplicación y de ningún otro lado (sección 2). Están diseñadas para teléfono: una columna y barra inferior de pestañas.

**La barra tiene tres pestañas: Mascotas, Citas y Ajustes.** Es la lista entera de lo que el tutor abre por sí mismo; todo lo demás es una pantalla de detalle a la que se llega desde una de las tres.

- Los **adjuntos no son pestaña**: acompañan a una mascota y no viven por su cuenta. Un listado global de archivos sueltos, sin la ficha que les da sentido, no responde ninguna pregunta que el tutor se haga. Se entra a ellos desde la ficha (5.6).
- La **ficha de paciente**, **compartir**, **quién la ve** y **cargar un antecedente** cuelgan de Mascotas: todas piden una mascota elegida antes de existir.
- Los **avisos no son pestaña**: son un interruptor por teléfono, no una bandeja (5.5), y viven adentro de Ajustes junto con la ficha propia del tutor (5.8).

### 5.1 Login / Registro
- Autenticación por email + contraseña, o con Google.
- **Registro abierto**: el tutor crea su cuenta desde la app sin intervención de una clínica, indicando nombre, email, contraseña y el consentimiento de uso de datos (proceso 4.9 de Reglas de Negocio). El registro crea su ficha de Tutor junto con la cuenta.
- **La web no ofrece el registro.** El alta es pública y no declara canal —la barrera está en el login que viene inmediatamente después (Arquitectura, 4.5)—, así que un formulario en el navegador terminaría en una cuenta creada y un ingreso rechazado. Quien llegue al ingreso web con una cuenta de tutor, o a la ruta del registro a mano, encuentra en su lugar que Wayka para tutores está en la aplicación.
- Queda operativo de inmediato: entra y ve sus secciones vacías hasta que una clínica le vincule Pacientes.
- **Al registrarse con contraseña le sale un correo con un enlace de confirmación** (proceso 4.9.1). La pantalla lo avisa y sigue de largo: confirmar **no es un paso del registro** y no bloquea nada — ni entrar, ni dar de alta la primera mascota (5.2). Registrarse con Google no manda nada, porque el correo ya viene verificado por el proveedor.
- La pantalla de confirmación se abre desde el enlace del correo, **con o sin sesión**, y lo único que hace es avisar que quedó confirmado. Ofrece abrir la app, sin obligar: el correo llega tanto al teléfono como a la computadora. Un enlace ya usado dice lo mismo: quien vuelve a hacer clic hizo bien las cosas dos veces, no algo mal una.

### 5.2 Mis mascotas
- Listado de las mascotas del tutor autenticado: las suyas y las que otra persona le compartió, con una etiqueta que distingue unas de otras y con qué nivel.
- **Alta de una mascota** (Reglas de Negocio, 4.17). No pide clínica: la mascota nace del tutor y se comparte después. La pantalla vacía ofrece cargar la primera, en vez de pedir que espere a que una clínica lo haga.
- **Antes de pedir, se muestra.** El estado vacío no arranca con el formulario: muestra primero una **ficha de ejemplo** —nombre, foto y antecedentes ficticios— para que el tutor vea qué va a tener antes de escribir nada. Es un **mock del cliente**: no persiste, no crea una mascota y no toca el backend. No se puede compartir, no se puede editar y la pantalla dice que es un ejemplo; una ficha de demostración que se confunda con una real es peor que no tenerla.
- **El alta pide una foto de la mascota**, opcional, y la pone primero en la pantalla y no como un campo más abajo del peso. Se sube por el camino de adjuntos (5.6), marcada como foto de perfil, y **con la mascota ya creada**: si la subida falla, la mascota queda dada de alta igual y la pantalla lo dice, en vez de perder el formulario entero por una foto.
- **El progreso del alta no arranca en cero.** El tutor ya se registró, y la barra lo reconoce: empieza el formulario en un tercio, llega a dos tercios en el paso de antecedentes y cierra en el resumen. El resumen dice qué armó el tutor —"la ficha de [nombre]"— y no que se guardaron unos datos.
- Terminada el alta, la aplicación pregunta **si tiene antecedentes para cargar** y lleva a 5.12. Es una pantalla con salida directa: saltearla deja la mascota dada de alta igual, y la ficha ofrece cargarlos más adelante. La salida **dice qué queda sin resolver** en vez de un "lo hago después" seco: que se puede cargar cuando quiera, pero que si hay una urgencia antes el veterinario arranca sin saber nada de esa mascota. Es cierto, y es lo que el tutor necesita para decidir. **Este camino pide conexión** de punta a punta (Sincronización sin Conexión, 5) y la pantalla lo dice cuando no la hay, en vez de dejar escribir algo que no se va a poder guardar.
- Las invitaciones pendientes aparecen arriba del listado, que es donde el tutor mira. No suman un ítem a la barra de pestañas: aceptar una deja la mascota en este mismo listado, así que la acción ya está en su lugar y una pestaña propia solo agregaría un desvío para volver acá.

### 5.3 Ficha de paciente
- Historial de eventos clínicos: fecha, tipo, descripción, diagnóstico, y **quién lo escribió** — el profesional y su clínica, que con una mascota compartida ya no son siempre los mismos, o el propio tutor cuando el registro es un antecedente que él cargó.
- **Foto y nombre de la mascota encabezan la ficha**, cuando hay foto de perfil cargada (5.6). Sin foto la ficha funciona igual; no se rellena con un ícono de especie que finja ser una.
- **La foto se cambia tocando el avatar**, ahí mismo en la ficha: la cámara de la app o el carrete (5.6). Es donde el tutor la está mirando. No se elige de entre los adjuntos ya cargados — eso obligaba a saber que la foto de perfil es un adjunto marcado, que es cómo se guarda y no cómo se piensa. La foto nueva entra por el camino de adjuntos marcada como foto de perfil, así que la anterior no se borra: deja de ser la que se muestra (Reglas de Negocio, 4.14).
- Medicación activa e histórica.
- Qué clínicas la atienden hoy.
- **Agregar un antecedente** (5.12), desde la ficha y en cualquier momento, no solo al dar de alta.
- **Sobre lo que escribió un profesional no hay edición de ninguna clase**, en ningún nivel. Lo que el tutor edita y da de baja son **sus propios antecedentes** (Reglas de Negocio, 3.2), más los datos no clínicos de la mascota según el nivel (5.7). La ficha tiene que hacer evidente cuál es cuál: una fila con acciones al lado de otra sin ellas, sin decir por qué, se lee como una falla.
- Una fecha declarada con precisión de mes o de año **se muestra como se declaró** —"marzo de 2023", "2023"— y nunca como un 1 de enero. El día rellenado es un detalle de cómo se guarda, no algo que el tutor haya dicho.
- La entrada a "Quién la ve" (5.10) **cambia según el estado**: mientras no la vea nadie es la invitación a compartir, en un toque; ya compartida es la fila de gestión, que dice con quién. Una mascota recién cargada no la ve nadie, y ahí lo que hace falta no es la entrada a una lista vacía sino la acción — sin una clínica que la atienda, nadie va a escribirle el historial.

### 5.4 Calendario
- Vista de citas pendientes con su fecha **y hora**, y con quién la va a atender si ya está asignada.
- Confirmar o solicitar reagenda de una cita (sin poder cambiar el estado directamente, según proceso 4.4 de Reglas de Negocio). Al reagendar, el tutor elige entre las horas válidas de la clínica que atiende a su mascota, igual que el veterinario.

### 5.5 Notificaciones

No es una pantalla de la barra: lo que el tutor toca de todo esto es un interruptor, y vive en Ajustes (5.8). La sección define qué se envía y bajo qué reglas.

- Recordatorio push el día anterior a una cita —a una hora fija— y otro un par de horas antes del turno, sobre las citas con `notificar_tutor` habilitado (Reglas de Negocio, 4.15).
- La app registra el dispositivo al iniciar sesión y lo da de baja al cerrarla: una cuenta sin dispositivo registrado no recibe avisos. **El tutor que se registró y todavía no abrió la app en su teléfono no recibe ninguno**, y la pantalla se lo dice ahí mismo en vez de dejarlo esperando un aviso que no va a llegar.
- **El tutor prende y apaga los avisos desde la app**, en Ajustes (5.8). Apagarlos da de baja este dispositivo; prenderlos lo da de alta de nuevo.
  - Es **por teléfono, no por cuenta**: el modelo registra Dispositivos y no una preferencia del Usuario (Modelo de Datos, sección 5). Apagarlos en un aparato no los apaga en el otro del mismo tutor — el que molesta es el que se tiene en la mano.
  - La decisión **sobrevive al cierre de sesión**. El registro automático del login la respeta; si no, el próximo ingreso volvería a prenderlos y el control no serviría de nada. Cerrar sesión sigue dando de baja el aparato por seguridad, pero eso no es apagarlos: al volver a entrar quedan como el tutor los dejó.
  - **No reemplaza al permiso del sistema operativo**, que es otra cosa y solo se revierte desde los ajustes del teléfono. Mientras el permiso no esté concedido no se ofrece el interruptor: sería un control que el sistema ya bloqueó. Conceder el permiso los deja prendidos, porque conceder ya fue decir que sí.
- El aviso dice qué mascota, qué día y a qué hora; nunca contenido clínico. Una notificación se lee en la pantalla bloqueada del teléfono.

### 5.6 Adjuntos
- Subida de archivos (ej. ficha histórica en papel, foto de una herida) asociados al paciente. Es también por donde entran las **fotos de la libreta sanitaria** en la carga de antecedentes (5.12): quedan colgadas de la mascota, sin obligar a asociar cada una a un antecedente puntual.
- **La foto de perfil de la mascota es un adjunto marcado, pero acá no se marca**: se cambia desde el avatar de la ficha (5.3). La tarjeta del adjunto no ofrece "usar como foto" — una foto de perfil que se elige en la lista de archivos hace que el tutor tenga que entender el modelo de datos para cambiarla. La tarjeta sí distingue cuál es la vigente.
- **Sacar la foto en el momento** con la cámara de la app, con guía de encuadre y revisión antes de subir, además de elegir un archivo que ya esté en el teléfono.
- **Mirar el adjunto sin salir de la ficha**, tocando la tarjeta (o el chip, en el historial). Un listado que solo muestra nombre y peso obliga a retirar y volver a subir para saber si la foto que se cargó era la correcta.
  - La tarjeta de un adjunto que es imagen **muestra la imagen**, no un icono genérico: reconocer cuál es cuál sin abrir ninguno es la mitad del problema.
  - Las **imágenes se ven dentro de la aplicación**, con acercamiento por pinch y doble toque, y arrastre para recorrerlas cuando están acercadas — una herida o el renglón de una ficha en papel se miran de cerca. El resto (un PDF, un formato que el aparato no dibuja) se abre con el visor del sistema: la aplicación no incorpora un motor de renderizado propio, y la URL prefirmada ya es lo que ese visor necesita.
  - El archivo **no se descarga al dispositivo**. Se mira contra la URL prefirmada del momento (Reglas de Negocio, 4.14.4), que vence en minutos: cada apertura pide una nueva en vez de reusar la del listado. Una copia local sería historial clínico fuera del alcance del motor de permisos.
  - El gesto es el mismo en las dos plataformas: el toque abre, y mantener apretado también — un adjunto no tiene una segunda acción que reclame el toque largo, así que los dos hacen lo mismo en vez de dejar uno sin respuesta. No hay acción de descarga ni de compartir: sacar el archivo del sistema es una decisión de producto que este documento no toma.
  - Con **movimiento reducido** activado, el visor abre sin animación y la imagen queda fija: el acercamiento continuo bajo el dedo es justo lo que ese ajuste pide evitar.

### 5.7 Datos básicos del paciente
- Edición de los campos no clínicos: nombre, especie, raza, fecha de nacimiento, sexo y peso actual. La hace el dueño y el co-tutor con nivel de edición; el de lectura los ve y no los toca.
- El **número de chip no se edita desde acá**: lo carga el veterinario, que es quien lo implanta y lo lee (Reglas de Negocio, 3.2).

### 5.8 Ajustes
Una sola pestaña para todo lo que es del tutor y no de una mascota: su ficha, su cuenta y el interruptor de avisos. Estaban separadas en dos pestañas y no lo justificaban — la de avisos era un único control, y un control solo no sostiene una entrada en la barra.

**Ficha propia del tutor**
- Lectura y edición de la ficha propia: nombre, contacto, dirección y documento.
- La dirección se escribe con autocompletado y se confirma sobre un mapa, que es lo que permite ver que el punto es el correcto antes de guardarlo. **Confirmarla no es obligatorio**: se puede guardar una dirección escrita a mano que el mapa no reconoce, y sin conexión el campo es texto libre (Arquitectura, 3.6). La pantalla lo tiene que dejar claro — un campo que parece exigir la sugerencia deja trabada a la persona que vive en una calle mal mapeada.
- El consentimiento de uso de datos no se edita desde acá: se otorga en el registro y no se revoca por la aplicación.
- El tutor no ve ni busca fichas de otros tutores, y no puede dar de baja la suya.

**Cuenta**
- Cambio de la contraseña de la propia cuenta.
- **Si el correo está sin confirmar, se ve acá y desde acá se reenvía el enlace.** Es el único lugar donde el tutor puede arreglar un correo mal tipeado antes de necesitarlo, que es siempre el peor momento: el día que olvide la contraseña, la recuperación va a mandar el enlace a esa dirección (3.1.2). El aviso no bloquea nada y no vuelve a aparecer una vez confirmado.
- Cerrar sesión, que además da de baja este dispositivo (5.5).

**Avisos**
- El interruptor de recordatorios push, con las reglas de 5.5: es por teléfono y no por cuenta, sobrevive al cierre de sesión, y no aparece mientras el permiso del sistema operativo no esté concedido.

### 5.9 Compartir una mascota
Solo el dueño. Dos caminos en la misma pantalla:

- **Con una veterinaria**: se busca por nombre en el directorio y se elige de una lista que muestra la dirección, para no confundir dos sucursales. Antes de confirmar, la pantalla dice que la clínica va a ver **el historial completo**, incluido lo que escribieron otras — no solo lo que escriba de ahí en adelante.
- **Con otra persona**: se ingresa su correo y se elige el nivel. Cada opción explica en una línea qué habilita, abajo de su nombre y no en un globo de ayuda: la diferencia entre "puede editar" y "solo mirar" es la decisión entera de esta pantalla.

Quien recibe la invitación no necesita tener cuenta: el correo trae un enlace, y si hace falta se registra antes de aceptar.

- **Necesita conexión.** Compartir es una decisión de seguridad y no entra a la cola de cambios sin conexión: encolarla sería compartir en diferido. Sin señal, la pantalla lo dice y no ofrece el botón.

### 5.10 Quién la ve
- Tres bloques: el dueño, las personas con acceso y las veterinarias vinculadas.
- El dueño revoca cualquiera y cambia el nivel de un co-tutor. Un co-tutor ve la misma lista sin acciones —saber quién más está mirando el historial no es administrar— y puede renunciar al suyo.
- Al revocar, el diálogo **dice la verdad sobre el efecto**: en el servidor es inmediato, pero un teléfono sin señal puede seguir mostrando lo que ya descargó hasta que se conecte (Sincronización sin Conexión, 8). Prometer un corte instantáneo sería mentir sobre algo que el sistema no controla.
- También necesita conexión, por el mismo motivo que 5.9.

### 5.11 Invitaciones recibidas
- Las que están pendientes, con qué mascota es, quién invita y con qué nivel. **Nada del historial**: aceptar es justamente lo que da acceso a él.
- Aparecen **arriba de Mis mascotas**, con un contador en esa misma pestaña. Es donde el tutor mira, y una invitación que vive detrás de un menú es una que nadie encuentra.
  - El contador va en **Mascotas y no en Ajustes**: la acción de aceptar vive ahí, y un contador que lleva a una pantalla sin nada que hacer es peor que ninguno. Los avisos son el interruptor del push del teléfono, no una bandeja.
- El **enlace del correo sigue siendo el otro camino**, y no es redundante: quien todavía no tiene cuenta no puede recibir un aviso dentro de una app en la que no está. Si el enlace se abre sin sesión, la pantalla de ingreso vuelve a la invitación después de entrar o de registrarse.
- Aceptar o rechazar. Aceptar deja la mascota en el listado con su etiqueta de nivel; rechazar la anula, y el enlace del correo deja de servir.
  - Desde la app se acepta **por identificador y no con el token**: el token es la credencial y no vuelve en ningún listado, porque devolverlo lo convertiría en algo que se puede reenviar. Lo que autoriza en los dos casos es lo mismo — que la cuenta tenga el correo al que se dirigió la invitación.

### 5.12 Cargar un antecedente
- Selector del tipo: **vacuna**, **alergia**, **medicación que está tomando** y **otra cosa que pasó** (una cirugía vieja, una consulta en otra veterinaria, un diagnóstico). Son cuatro tarjetas y no un desplegable de siete tipos: el tutor elige entre las cosas que él sabe nombrar, no entre las categorías del historial (Reglas de Negocio, 4.23). **Vacuna viene destacada y encabeza la lista**: es lo que más gente tiene a mano. En un menú de cuatro tarjetas el valor por defecto es eso —destacar— y no una selección previa: abrir el formulario de vacuna sin que el tutor lo haya pedido le cobraría un paso al que viene con una alergia.
- Formulario corto por tipo. Lo único obligatorio es qué es —el nombre de la vacuna, el alérgeno, la droga— y la fecha; el resto se puede dejar vacío y la pantalla lo dice así, en vez de marcar en rojo lo que el tutor no tiene cómo saber.
- **La fecha se declara con la precisión que se tenga**: día exacto, mes, o solo el año. El control ofrece los tres niveles sin castigar al que elige el más grueso, y no exige bajar a día. Es el caso normal, no el degradado: una libreta de hace cinco años dice el año — y por eso el control **arranca en año** en esta pantalla, al revés que en la del veterinario, que escribe lo de hoy.
- Se cargan **varios seguidos**: guardar uno devuelve al selector con lo ya cargado a la vista, y no a la ficha. Vaciar una libreta son seis o siete entradas, y volver al principio en cada una convierte diez minutos en veinte.
- **Fotos de la libreta sanitaria u otros papeles**, por el mismo camino de adjuntos de 5.6 y con la cámara en móvil. También salteable.
- **Resumen antes de terminar**: qué quedó cargado, con la opción de **quitar** cualquiera de las entradas y cargarla de nuevo. No hay "editar" acá, y no es una simplificación: la Medicación solo admite corregir su cierre —un cambio de dosis se registra cerrando la activa y abriendo otra, para no perder la anterior (Reglas de Negocio, 4.3)—, así que ofrecer editar en un tipo de antecedente y no en el otro sería una diferencia sin causa a los ojos del tutor. Quitar es baja lógica, como cualquier otra del historial.
- Sin nada cargado el resumen no aparece: salir es salir. Pararlo en una pantalla que dice "no cargaste nada" es cobrarle un toque más a quien ya dijo que no tenía nada.
- Se llega desde dos lados y es la misma pantalla: desde el alta de la mascota (5.2), envuelta en el onboarding, y desde la ficha (5.3), sin envoltorio. Lo único que cambia entre los dos caminos es que el primero exige conexión.
- **Lo cargado se ve como propio.** En el historial aparece marcado como declarado por el tutor, y con las acciones de editar y dar de baja que los registros del veterinario no tienen. Es donde el tutor entiende, sin que nadie se lo explique, cuál es la diferencia entre las dos clases de registro.

## 6. Fuera de alcance de este documento

- **Vista específica de paciente derivado en urgencia** — pendiente de definición, relacionada con Fase 2.
- **Elección de stack técnico** — este documento define funcionalidad, no tecnología.
