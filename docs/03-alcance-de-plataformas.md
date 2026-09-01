# Wayka — Alcance de Plataformas

MVP — Web de gestión + Aplicación móvil
Versión 1.0 · Complementa a Modelo de Datos y Reglas de Negocio

## 1. Definición de plataformas

Wayka se compone de dos productos para el MVP: una aplicación web de gestión, de uso exclusivo de la clínica, y una aplicación móvil que sirve tanto al veterinario como al tutor, con permisos distintos según el rol autenticado.

| Plataforma | Quién accede | Propósito |
|---|---|---|
| Web | Veterinario, Clínica_admin, Tutor | Gestión completa del día a día clínico y administrativo. Para el tutor: la misma consulta que hace desde el teléfono, cuando no lo tiene a mano. |
| Móvil | Veterinario (paridad con web), Tutor (paridad con web) | Para el veterinario: misma herramienta que la web, disponible fuera de la clínica. Para el tutor: el punto de entrada habitual, y el único donde recibe avisos y puede sacar una foto. |
| Línea de comandos | Administrador de la plataforma | Alta de una Clínica junto con su cuenta clínica_admin, y emisión del token de activación con el que esa cuenta define su contraseña (Reglas de Negocio, 4.10). No es una interfaz de usuario del producto: es una herramienta de operación, fuera de la API HTTP. |

> **El clínica_admin no tiene acceso a la aplicación móvil**, y esa restricción se valida en el backend, no solo ocultando la opción en la interfaz — mismo criterio aplicado al motor de permisos en Reglas de Negocio.
>
> El tutor **sí** entra por web. La primera versión de este documento decía que no accedía bajo ninguna circunstancia; se cambió porque dejaba afuera al tutor sin el teléfono a mano, y porque ningún dato ni ninguna acción del tutor dependen del canal. Lo que la app le da y la web no son las **dos funciones que dependen del aparato**: las notificaciones push (5.5) y sacar una foto con la cámara (5.6). Las dos degradan solas — sin push no hay recordatorio, y sin cámara se adjunta un archivo que ya esté en la computadora.

## 2. Matriz de plataforma por rol

| Rol | Web | Móvil |
|---|---|---|
| Clínica_admin | Acceso completo | Sin acceso |
| Veterinario | Acceso completo | Acceso completo (paridad total con la web) |
| Tutor | Acceso completo, menos push y cámara | Acceso completo |

> La paridad entre web y móvil implica que ambas plataformas comparten el mismo conjunto de funcionalidades para ese rol — la diferencia entre plataformas es de dispositivo/contexto de uso, no de permisos ni de features disponibles.
>
> Las dos excepciones del tutor no son de permisos: **push** (5.5) necesita un dispositivo registrado, que solo existe entrando desde la app, y **la cámara** (5.6) necesita una. En web, la subida de adjuntos sigue funcionando con un archivo del disco.

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
- Es, junto con el ingreso, el registro de tutor (5.1) y la activación (3.1.1), una de las pantallas alcanzables sin sesión.

### 3.1.1 Activación de la cuenta de clínica_admin (sin sesión)
- Se abre con el token de activación que el administrador de la plataforma le entregó a la clínica (proceso 4.16 de Reglas de Negocio).
- Pide únicamente la contraseña nueva, con la política de la regla 2.1 a la vista mientras se escribe.
- Un token inexistente, vencido o ya usado se rechaza con un mismo mensaje genérico: la pantalla no ayuda a distinguir cuál de los tres fue.
- Al activar **no se entra**: el canje no emite sesión. La pantalla lleva al login, donde se estrena la contraseña recién definida.
- Es, junto con el registro de tutor en móvil (5.1), una de las dos pantallas del sistema alcanzables sin sesión.

### 3.2 Panel de clínica (rol: Clínica_admin)
- Alta, edición y baja lógica de Veterinarios asociados a la clínica. La ficha y la cuenta de acceso se crean juntas, en una sola operación (proceso 4.12 de Reglas de Negocio), y la baja de la ficha desactiva la cuenta (4.13).
- La clínica y su propia cuenta de administrador no se crean desde acá: las da de alta el administrador de la plataforma (proceso 4.10 de Reglas de Negocio).
- Edición de datos administrativos de la Clínica (nombre, dirección con confirmación en el mapa, contacto) y de su **horario de atención**: hora de apertura, hora de cierre y duración del turno. Los tres definen la grilla con la que el veterinario agenda (Modelo de Datos, 4.3), así que un cambio acá cambia qué horas son válidas en el calendario de toda la clínica.
- El horario no se puede achicar mientras existan Citas pendientes que queden afuera del horario nuevo (regla 2.2): la pantalla tiene que decir cuáles son, no solo que la operación falló.
- Cambio de la contraseña de la propia cuenta.
- **Restablecer la contraseña de una cuenta del plantel**, desde la ficha de esa persona. No exige conocer la anterior. Es la única salida que tiene hoy un veterinario que olvidó la suya: no hay recuperación sin sesión (sección 6), y el correo con el que se avisaría tampoco existe. La contraseña nueva se la comunica el administrador por un medio propio, fuera del sistema.
- Sin acceso a historial clínico ni medicación de pacientes (regla de alcance ya definida).

### 3.3 Gestión de pacientes (rol: Veterinario)
- Lectura del plantel de la propia clínica (sin poder modificarlo), para resolver quién firmó cada registro clínico.
- Alta de paciente (con alta de tutor si no existe, según proceso 4.1 de Reglas de Negocio).
- Búsqueda de tutores por documento, nombre o contacto; alta y edición de la ficha de un tutor, incluido completar documento y dirección — la dirección con el mismo autocompletado y mapa que usa el tutor en su ficha propia (5.8). El listado se pagina siempre: no está acotado a la propia clínica.
- Baja lógica de una ficha de tutor.
- Búsqueda y listado de los pacientes vinculados a la propia clínica.
- Ficha de paciente: datos básicos, historial de eventos clínicos, medicación activa e histórica.
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
- Campos estructurados obligatorios para vacunas y alergias (según nota 4.5 del Modelo de Datos).
- Adjuntar archivos (foto, PDF, estudio) al evento.

### 3.5 Gestión de medicación (rol: Veterinario)
- Alta de medicación activa, con bloqueo si ya existe una activa de la misma droga (regla 2.2).
- Cierre de medicación (fecha_fin).
- Vista rápida de medicación activa por paciente.

### 3.6 Calendario / Citas (rol: Veterinario)
- Creación de citas (vacuna, control, cirugía programada) con **fecha y hora**, sobre la grilla que definen el horario de atención y la duración del turno de la clínica (3.2). Las horas fuera de la grilla no se ofrecen: el backend las rechaza igual, pero ofrecerlas y después fallar es un error que la interfaz puede evitar.
- La cita atendida se marca como cumplida desde la agenda, asentando la atención (3.3.1). No hay un control de "cumplida" aparte: cumplir una cita es haber atendido.
- Vista de citas pendientes y vencidas de la clínica, por día, semana y mes, con quién atiende cada una. Son las citas agendadas **en esta clínica**: una mascota atendida también en otra tiene allá su propia agenda, que acá no se ve.
- **Asignación de profesional**, opcional: una cita puede quedar de la clínica y repartirse después. Al asignar, no se ofrecen los profesionales que ya tienen otra cita en ese momento — el backend lo rechaza igual, pero ofrecerlo y después fallar es un error que la interfaz puede evitar.
- Filtrar la agenda por profesional, incluyendo "sin asignar": es la lista de lo que todavía hay que repartir.

### 3.7 Mi cuenta (rol: Veterinario)
- Lectura de los propios datos: nombre, matrícula y correo. **No son editables desde acá** — los carga el clínica_admin al dar de alta la cuenta (proceso 4.12), y la matrícula además decide si el veterinario puede escribir historial (regla 2.1): cambiársela a sí mismo sería cambiarse los permisos.
- Cambio de la contraseña de la propia cuenta.
- Existe porque el veterinario era el único rol sin ningún lugar donde vivieran sus datos: el tutor los tiene en "Mis datos" (5.8) y el clínica_admin en el panel (3.2). Va en el menú y no colgada de un avatar porque el rol tiene paridad entre web y móvil, y en el teléfono no hay avatar donde colgarla.

## 4. Pantallas mínimas — Móvil (Veterinario)

Mismo conjunto funcional que las secciones 3.3 a 3.6 —incluido el asiento de la atención (3.3.1), que en el teléfono es el caso principal: se atiende parado al lado de la mesa, no sentado frente a la computadora—, adaptado a formato móvil. No se listan de nuevo por tratarse de las mismas funcionalidades con paridad total.

> Nota de diseño: al tener paridad completa, conviene evaluar en la etapa técnica si conviene un codebase compartido (ej. framework multiplataforma) para el veterinario, en vez de mantener dos implementaciones separadas del mismo alcance funcional.

## 5. Pantallas mínimas — Tutor

Son las mismas en las dos plataformas. Están diseñadas para teléfono —una columna, barra inferior de pestañas— y en el navegador se muestran en esa misma composición, centradas: es una decisión de alcance, no un pendiente olvidado. Un kit propio de tutor-web se puede pedir más adelante sin rehacer nada de esto.

**La barra tiene tres pestañas: Mascotas, Citas y Ajustes.** Es la lista entera de lo que el tutor abre por sí mismo; todo lo demás es una pantalla de detalle a la que se llega desde una de las tres.

- Los **adjuntos no son pestaña**: acompañan a una mascota y no viven por su cuenta. Un listado global de archivos sueltos, sin la ficha que les da sentido, no responde ninguna pregunta que el tutor se haga. Se entra a ellos desde la ficha (5.6).
- La **ficha de paciente**, **compartir** y **quién la ve** cuelgan de Mascotas: todas piden una mascota elegida antes de existir.
- Los **avisos no son pestaña**: son un interruptor por teléfono, no una bandeja (5.5), y viven adentro de Ajustes junto con la ficha propia del tutor (5.8).

### 5.1 Login / Registro
- Autenticación por email + contraseña, o con Google.
- **Registro abierto**: el tutor crea su cuenta desde la app sin intervención de una clínica, indicando nombre, email, contraseña y el consentimiento de uso de datos (proceso 4.9 de Reglas de Negocio). El registro crea su ficha de Tutor junto con la cuenta.
- Queda operativo de inmediato: entra y ve sus secciones vacías hasta que una clínica le vincule Pacientes.

### 5.2 Mis mascotas
- Listado de las mascotas del tutor autenticado: las suyas y las que otra persona le compartió, con una etiqueta que distingue unas de otras y con qué nivel.
- **Alta de una mascota** (Reglas de Negocio, 4.17). No pide clínica: la mascota nace del tutor y se comparte después. La pantalla vacía ofrece cargar la primera, en vez de pedir que espere a que una clínica lo haga.
- Las invitaciones pendientes aparecen arriba del listado, que es donde el tutor mira. No suman un ítem a la barra de pestañas: aceptar una deja la mascota en este mismo listado, así que la acción ya está en su lugar y una pestaña propia solo agregaría un desvío para volver acá.

### 5.3 Ficha de paciente
- Historial de eventos clínicos: fecha, tipo, descripción, diagnóstico, y **quién lo escribió** — el profesional y su clínica, que con una mascota compartida ya no son siempre los mismos.
- Medicación activa e histórica.
- Qué clínicas la atienden hoy.
- **Sin permisos de edición sobre ningún dato clínico**, en ningún nivel (regla de la matriz de permisos). Lo que sí se edita son los datos de la mascota, y depende del nivel (5.7).
- La entrada a "Quién la ve" (5.10) **cambia según el estado**: mientras no la vea nadie es la invitación a compartir, en un toque; ya compartida es la fila de gestión, que dice con quién. Una mascota recién cargada no la ve nadie, y ahí lo que hace falta no es la entrada a una lista vacía sino la acción — sin una clínica que la atienda, nadie va a escribirle el historial.

### 5.4 Calendario
- Vista de citas pendientes con su fecha **y hora**, y con quién la va a atender si ya está asignada.
- Confirmar o solicitar reagenda de una cita (sin poder cambiar el estado directamente, según proceso 4.4 de Reglas de Negocio). Al reagendar, el tutor elige entre las horas válidas de la clínica que atiende a su mascota, igual que el veterinario.

### 5.5 Notificaciones — solo móvil

No es una pantalla de la barra: lo que el tutor toca de todo esto es un interruptor, y vive en Ajustes (5.8). La sección define qué se envía y bajo qué reglas.

- Recordatorio push el día anterior a una cita —a una hora fija— y otro un par de horas antes del turno, sobre las citas con `notificar_tutor` habilitado (Reglas de Negocio, 4.15).
- La app registra el dispositivo al iniciar sesión y lo da de baja al cerrarla: una cuenta sin dispositivo registrado no recibe avisos. **El tutor que solo entra por web no recibe ninguno**, y la pantalla se lo dice ahí mismo en vez de dejarlo esperando un aviso que no va a llegar.
- **El tutor prende y apaga los avisos desde la app**, en Ajustes (5.8). Apagarlos da de baja este dispositivo; prenderlos lo da de alta de nuevo.
  - Es **por teléfono, no por cuenta**: el modelo registra Dispositivos y no una preferencia del Usuario (Modelo de Datos, sección 5). Apagarlos en un aparato no los apaga en el otro del mismo tutor — el que molesta es el que se tiene en la mano.
  - La decisión **sobrevive al cierre de sesión**. El registro automático del login la respeta; si no, el próximo ingreso volvería a prenderlos y el control no serviría de nada. Cerrar sesión sigue dando de baja el aparato por seguridad, pero eso no es apagarlos: al volver a entrar quedan como el tutor los dejó.
  - **No reemplaza al permiso del sistema operativo**, que es otra cosa y solo se revierte desde los ajustes del teléfono. Mientras el permiso no esté concedido no se ofrece el interruptor: sería un control que el sistema ya bloqueó. Conceder el permiso los deja prendidos, porque conceder ya fue decir que sí.
- El aviso dice qué mascota, qué día y a qué hora; nunca contenido clínico. Una notificación se lee en la pantalla bloqueada del teléfono.

### 5.6 Adjuntos
- Subida de archivos (ej. ficha histórica en papel, foto de una herida) asociados al paciente.
- En móvil, además, **sacar la foto en el momento** con la cámara de la app, con guía de encuadre y revisión antes de subir. En web solo se adjunta un archivo que ya exista.
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
- Cerrar sesión, que además da de baja este dispositivo (5.5).

**Avisos — solo móvil**
- El interruptor de recordatorios push, con las reglas de 5.5: es por teléfono y no por cuenta, sobrevive al cierre de sesión, y no aparece mientras el permiso del sistema operativo no esté concedido.
- En web el bloque no se muestra como un control apagado sino como la explicación de que los avisos llegan al teléfono: un interruptor que la plataforma no puede honrar es peor que su ausencia.

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

## 6. Fuera de alcance de este documento

- **Recuperación de contraseña sin sesión ("olvidé mi contraseña").** Hoy no existe ni como pantalla ni como endpoint. El veterinario tiene salida por el restablecimiento que hace su clínica_admin (3.2), pero **el tutor no tiene ninguna**: ninguna clínica alcanza su cuenta, porque el alcance de un administrador se resuelve contra la clínica de pertenencia y un tutor no tiene. Un tutor que olvida su contraseña hoy pierde el acceso. Cerrar el hueco pide backend nuevo (token de recuperación de un solo uso con vencimiento, y un proveedor de correo, que el sistema no tiene: hoy solo manda push).
- **Vista específica de paciente derivado en urgencia** — pendiente de definición, relacionada con Fase 2.
- **Elección de stack técnico** — este documento define funcionalidad, no tecnología.
