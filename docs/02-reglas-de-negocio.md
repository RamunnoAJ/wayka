# Wayka — Reglas de Negocio y Lógica de Aplicación

MVP — Historial clínico colaborativo y calendario
Versión 1.0 · Complementa a Wayka — Modelo de Datos

## 1. Alcance

Este documento define el comportamiento del sistema que no queda expresado en el modelo de datos: qué reglas se validan antes de escribir, quién puede hacer qué sobre cada dato, y cómo se encadenan las operaciones dentro de un mismo proceso de negocio.

Cubre tres capas: integridad y validación, motor de permisos, y procesos de negocio. Queda fuera de este documento (a definir en una etapa posterior): la vista específica de paciente derivado en urgencia.

## 2. Reglas de integridad y validación

Reglas que la aplicación debe hacer cumplir antes de persistir un cambio, independientemente de si la base de datos también las restringe.

### 2.1 Identidad y cuentas

| Regla | Aplica a | Validación |
|---|---|---|
| Documento único | Tutor, Veterinario | número_documento no puede repetirse entre fichas vigentes **de la misma entidad** con el mismo tipo_documento. **La unicidad no cruza Tutor y Veterinario**: el mismo documento puede estar en una ficha de cada una, porque es la misma persona en dos papeles y no una ficha duplicada (Modelo de Datos, 4.1 y 4.4). En Tutor la validación aplica solo a las fichas que ya tienen documento cargado: el auto-registro (4.9) no lo exige. Se valida tanto en el alta como en la edición de la ficha, y el tipo y el número se cargan o se limpian juntos: una ficha nunca tiene uno sin el otro. |
| Email único | Usuario | email no puede repetirse en todo el sistema, independientemente del tipo_usuario. |
| Un solo rol activo por cuenta | Usuario | Exactamente una de tutor_id / veterinario_id / clínica_id debe estar completa, según tipo_usuario. Las otras dos deben ser NULL. |
| Al menos un método de autenticación | Usuario | password_hash y google_id no pueden ser ambos NULL — un Usuario necesita al menos una forma de autenticarse. **Única excepción**: la cuenta con `activación_pendiente = true`, entre que la crean —la línea de comandos si es clínica_admin (4.10), el clínica_admin si es veterinario (4.12)— y que alguien la estrena (proceso 4.16). Un tutor nunca está en ese estado: se registra él mismo, y elige su credencial en el mismo acto. Mientras tanto no puede autenticarse por ninguna vía, que es exactamente lo que se busca: la cuenta existe pero todavía no la estrenó nadie. El canje escribe la contraseña y baja la bandera en la misma transacción, y la regla vuelve a regir sin matices. |
| Google ID único | Usuario | google_id no puede repetirse — una cuenta de Google solo puede estar vinculada a un Usuario de Wayka. |
| Política de contraseña | Usuario | Mínimo 8 caracteres, con al menos una letra minúscula, una mayúscula y un dígito. Rige tanto el alta como el cambio de contraseña. |
| Clínica de pertenencia según rol | Usuario | clínica_de_pertenencia_id debe ser NULL en las cuentas de tutor, estar completa en las de veterinario, y coincidir con clínica_id en las de clínica_admin. |
| Rol y referencias inmutables | Usuario | tipo_usuario y las tres FK de rol no son editables después del alta: permitir cambiarlas habilitaría una escalada de privilegios sobre una cuenta ya creada. |
| Email reutilizable tras la baja | Usuario | La unicidad de email rige entre cuentas vigentes. El email de una cuenta dada de baja lógicamente puede reutilizarse en un alta posterior. |
| Matrícula única | Veterinario | La matrícula no se repite entre fichas vigentes que la tengan cargada, **en todo el sistema y no por clínica**: la emite un colegio profesional, no la clínica (Modelo de Datos, 4.4). Se valida en el alta y en la edición, y se compara normalizada —sin espacios al borde y sin distinguir mayúsculas—, porque `mp-4821` y `MP-4821` son la misma habilitación. Dos fichas sin matrícula no colisionan. Limpiarla libera el número para otra ficha, con el mismo criterio que el email tras la baja. |
| Matrícula obligatoria para escribir | Veterinario | Un veterinario sin matrícula cargada no puede crear ni editar Eventos clínicos ni Medicación (puede existir de alta, pero queda en modo restringido). |

### 2.2 Datos clínicos

| Regla | Aplica a | Validación |
|---|---|---|
| Fecha no futura | Evento clínico | fecha no puede ser posterior a la fecha actual (un evento clínico registra algo que ya ocurrió; lo futuro es una Cita). |
| Paciente vigente para cargar | Evento clínico | No se crean Eventos clínicos sobre un Paciente con `deleted_at`. El historial existente sigue consultable y editable (regla 4.5), pero no se le agregan episodios nuevos. |
| Campo estructurado según el tipo **y el origen** | Evento clínico | `campo_estructurado` se valida contra el esquema fijo del `tipo` del evento (Modelo de Datos, 4.5): obligatorio para vacuna, medicación y alergia, y NULL para consulta, cirugía, control y urgencia. **Qué claves son obligatorias depende de `cargado_por`**: con `veterinario` rige la forma completa; con `tutor` quedan opcionales `lote` (vacuna), `severidad` (alergia) y `dosis` y `frecuencia` (medicación). Las claves son las mismas en los dos casos. Un JSON con claves ajenas al esquema, sin las obligatorias de su origen, o presente en un tipo que no lo admite, se rechaza. |
| El origen lo pone el backend | Evento clínico, Medicación | `cargado_por` **no se recibe por la API**: se resuelve contra el `tipo_usuario` del token —veterinario escribe `veterinario`, tutor escribe `tutor`— igual que `usuario_id`. Recibirlo del cliente dejaría que un tutor firme un registro como si lo hubiera escrito un profesional, que es la única cosa que este campo existe para impedir. |
| El origen no se edita | Evento clínico, Medicación | `cargado_por` no cambia en ninguna operación de edición. Un antecedente declarado no se convierte en acto médico, ni al revés. Corregir un origen mal cargado es dar de baja el registro y escribir uno nuevo. |
| Precisión de fecha según el origen | Evento clínico, Medicación | `fecha_precision` distinta de `dia` solo se admite con `cargado_por = tutor`; un registro del veterinario que la declare se rechaza. Los componentes desconocidos de la fecha se guardan en `01` (Modelo de Datos, 4.5), y el backend **normaliza en vez de rechazar**: con precisión `anio` se persiste el 1 de enero de ese año aunque el cliente haya mandado otro día, y con `mes`, el 1 de ese mes. Guardar el día que el cliente mandó junto a una precisión que dice que no se conoce sería guardar dos afirmaciones que se contradicen. |
| Fecha no futura con precisión parcial | Evento clínico, Medicación | La regla de fecha no futura se evalúa sobre el **período** que la fecha declara, no sobre el `DATE` guardado: con precisión `anio`, el año en curso es válido aunque el 1 de enero ya haya pasado; con `mes`, el mes en curso. Rechazar "2026" en marzo de 2026 sería rechazar un dato cierto por culpa del relleno.
| Momento no futuro | Consulta atendida | `fecha_hora` no puede ser posterior al momento actual. Es el mismo criterio que el Evento clínico: lo que ya ocurrió se asienta, lo que va a ocurrir se agenda. |
| Paciente vigente para asentar | Consulta atendida | No se asientan consultas sobre un Paciente con `deleted_at`, igual que no se le cargan eventos. |
| Clínica vinculada | Consulta atendida | La clínica que asienta tiene que tener vínculo vigente con el paciente. Se resuelve contra el actor, no se recibe por la API. |
| El profesional atiende en esa clínica | Consulta atendida | `veterinario_id` tiene que ser de un veterinario de esa clínica, tanto al asentar como al corregir. Un veterinario asienta la atención de un colega, no la de alguien de otra clínica. |
| Una cita se atiende una sola vez | Consulta atendida | `cita_id` es único entre las consultas vigentes. Si viene, la Cita tiene que ser del mismo Paciente, de la misma clínica y no estar dada de baja; y `origen` tiene que ser `agendada`. Una consulta sin `cita_id` no puede declararse `agendada`. |
| Consulta vigente y del mismo paciente | Evento clínico | El evento se guarda con `consulta_id` y ya no con una FK propia a la Cita (Modelo de Datos, 4.5). Si trae `consulta_id`, la consulta tiene que ser del mismo Paciente y estar vigente. Si en cambio declara la cita que cumple, se resuelve a la consulta de esa cita — y si trae las dos, tienen que ser la misma. |
| Una medicación activa por droga | Medicación | No se permite crear una nueva Medicación con fecha_fin NULL para una droga que el paciente ya tiene activa. Debe cerrarse la anterior (fecha_fin) antes de abrir una nueva. |
| Zona horaria válida | Clínica | zona_horaria tiene que ser un nombre IANA que el sistema pueda resolver. Una zona que no existe deja el horario de atención sin significado: "abre 09:00" no dice nada si no se sabe 09:00 de dónde. |
| Franja coherente | Franja de atención | `hora_hasta` tiene que ser posterior a `hora_desde`. No se admite una franja que cruce la medianoche: una guardia nocturna es un caso que el MVP no modela, y aceptarlo acá lo haría parecer soportado. Un día de la semana **sin ninguna franja es un día cerrado**, y es válido: es como se expresa que la clínica no abre el domingo. |
| Franjas que no se solapan ni se tocan | Franja de atención | Dos franjas del mismo día no pueden superponerse, ni terminar donde empieza la siguiente. 09:00–13:00 y 13:00–18:00 son una sola franja escrita en dos filas y se rechazan: lo que hace que dos filas signifiquen algo distinto de una es el hueco entre ellas — el corte de mediodía. |
| El horario se escribe entero | Franja de atención | La escritura reemplaza el conjunto completo de franjas de la clínica en una transacción, y se valida completo antes de aceptarse. No hay alta ni baja de una franja suelta: la grilla es una sola cosa, y aceptarla por partes deja pasar estados intermedios inválidos. |
| Turno que divide cada franja | Clínica, Franja de atención | `duración_turno_minutos` tiene que ser mayor a cero y dividir de forma exacta **cada una** de las franjas de la clínica. Si no divide, el último turno de esa franja queda cortado por su cierre y la grilla miente sobre cuántos turnos entran. Se valida en las dos direcciones: al escribir las franjas y al cambiar la duración del turno. |
| Horario que no invalida lo agendado | Clínica, Franja de atención | Achicar una franja, borrarla (cerrar el día) o cambiar la duración del turno **no reagenda ni cancela** las Citas pendientes que quedan fuera de la grilla nueva. Se rechaza el cambio mientras existan, y el rechazo dice cuáles son: mover la agenda de una mascota es una decisión clínica, no un efecto colateral de editar la configuración de la clínica. Es la diferencia con la Ausencia (4.22), que sí desasigna — ahí lo que se pierde es quién atiende, acá se perdería el turno entero. |
| Consentimiento previo a alta de paciente | Tutor, Paciente | No se puede dar de alta un Paciente si el Tutor asociado no tiene consentimiento_datos = true. |
| Momento no pasado | Cita | fecha_programada no puede ser anterior al momento actual, ni al crear ni al reagendar. Es la regla espejo del Evento clínico: lo que ya ocurrió se registra como evento, lo que va a ocurrir se agenda como cita. Una cita creada en el pasado nacería vencida. Ahora que `fecha_programada` lleva hora, la comparación es contra el instante y no contra el día: agendar hoy a las 09:00 cuando son las 15:00 se rechaza. |
| Dentro de una franja de atención | Cita | `fecha_programada`, leída en la `zona_horaria` de **la clínica de la cita**, tiene que caer dentro de **alguna** franja del día de la semana que le toca, y el turno completo (`duración_turno_minutos`) tiene que terminar antes del cierre de **esa misma** franja. Un turno que empieza a las 12:45 con la franja cerrando 13:00 se rechaza aunque a las 13:00 empiece otra: el corte de mediodía es un hueco, no una pausa que el turno pueda atravesar. Un día sin franjas no admite citas. |
| Un profesional no se duplica | Cita | Un mismo veterinario no puede tener dos Citas **pendientes** en el mismo momento. Es la regla que hace que asignar profesional signifique algo: sin ella la asignación es una etiqueta y la agenda vuelve a mentir sobre quién está libre. Las citas sin asignar no colisionan entre sí — sin profesional no hay a quién solapar. Una cita cumplida o vencida tampoco bloquea: ya pasó. |
| El profesional atiende en esa clínica | Cita | `veterinario_id`, si viene, tiene que ser de un veterinario vigente de **la clínica de la cita**. Asignarle una cita a alguien de otra clínica sería agendarle trabajo a quien no atiende ahí. |
| El profesional está disponible | Cita | No se asigna una Cita a un veterinario que tiene una Ausencia vigente (Modelo de Datos, 4.19) cubriendo ese momento. Es la misma familia que "un profesional no se duplica": asignar trabajo a quien no va a estar hace que la agenda mienta sobre quién está libre. La cita se agenda igual **sin profesional** — que no haya nadie disponible no es motivo para no tomar el turno. |
| Una ausencia por vez | Ausencia del profesional | Dos ausencias vigentes del mismo veterinario no pueden solaparse. Dos filas para el mismo rango no significan nada distinto de una, y dejan sin respuesta cuál se da de baja para volver a estar disponible. |
| Rango de ausencia coherente | Ausencia del profesional | `hasta` tiene que ser posterior a `desde`. Se admite una ausencia que empieza en el pasado: alguien que no vino ayer no vino ayer, y cargarlo hoy es lo normal. |
| Ausencia sobre el propio plantel | Ausencia del profesional | El veterinario tiene que pertenecer a la clínica del clínica_admin que la carga, y estar vigente. Se resuelve contra el actor, no se recibe por la API. |
| Cita en una clínica vinculada | Cita | La clínica de la cita tiene que tener vínculo vigente con el paciente al momento de agendar (Modelo de Datos, 4.13). Se resuelve contra el actor, no se recibe por la API: una cita nace en la clínica del veterinario que la agenda. Una mascota que el tutor todavía no compartió con nadie no admite citas, y el rechazo lo dice así — no es una falta de permiso, es que no hay agenda donde ponerla. |
| Alineada a la grilla de turnos | Cita | La hora de `fecha_programada` tiene que ser múltiplo del `duración_turno_minutos` de **la clínica de la cita**, contado desde el `hora_desde` de **la franja en la que cae**. Con turnos de 30 min y una franja que abre 09:00, valen 09:00, 09:30, 10:00…; 09:17 se rechaza. Cada franja empieza a contar de nuevo desde su propio comienzo — es lo que hace que una franja de tarde que abre a las 15:20 tenga su grilla y no la del turno de la mañana. Sin esta regla los turnos se solapan de a poco y el calendario deja de representar la agenda real. |
| Paciente vigente para agendar | Cita | No se agendan Citas nuevas sobre un Paciente con `deleted_at`. Las ya programadas siguen consultables (regla 4.5). |
| Estado no editable | Cita | `estado` no se recibe por la API en ninguna operación: lo mueve el sistema (Modelo de Datos, 4.7). |
| Un dispositivo pertenece a una sola cuenta | Dispositivo | Un token de push identifica un teléfono, no una persona. Registrar un token ya asociado a otra cuenta lo reasigna a la que lo registra: es el mismo teléfono donde ahora entró otro usuario, y seguir mandándole las notificaciones del anterior sería filtrarle datos de otra mascota. |
| Solo se notifica lo que el tutor habilitó | Notificación | No se encolan recordatorios de una Cita con `notificar_tutor = false`. El flag lo controla tanto el veterinario como el propio tutor (sección 3.2). |
| No se notifica lo que ya no está pendiente | Notificación | Un recordatorio encolado que, llegado el momento de enviarse, corresponde a una Cita cumplida, vencida o dada de baja se descarta sin enviar. Entre que se encola y que se despacha pasan horas: avisar de una cita que ya se cumplió es peor que no avisar. |
| Tipo declarado coherente con el archivo | Adjuntos | El `tipo` (foto / PDF / estudio) tiene que corresponderse con el tipo MIME real del archivo subido. Declarar "PDF" y subir un ejecutable se rechaza antes de escribir en el bucket. |
| Tamaño máximo por archivo | Adjuntos | Un archivo que excede el máximo configurado se rechaza. El límite es del backend, no del cliente. |
| Paciente vigente para adjuntar | Adjuntos | No se suben archivos nuevos a un Paciente con `deleted_at`. Los ya subidos siguen consultables (regla 4.5). |
| Evento del mismo paciente | Adjuntos | Si el adjunto declara `evento_id`, ese evento tiene que pertenecer al mismo Paciente. |
| Solo se reagenda lo pendiente | Cita | Una Cita en estado `cumplido` o `vencido` no admite cambio de fecha_programada ni de tipo: reagendar es mover algo que todavía va a pasar. Lo que se hace con una cita vencida es agendar una nueva. |
| La clínica de la cita no se muda | Cita | `clinica_id` no es editable. Mudar una cita de clínica le cambia la grilla, el huso horario y la validez del profesional asignado, los tres a la vez: eso no es editar una cita, es dar de baja una y agendar otra. |
| Una clínica no se vincula dos veces | Vínculo con clínica | No se crea un vínculo con una clínica que ya tiene uno vigente sobre ese paciente. Revocar y volver a compartir sí se permite, y deja las dos filas: la unicidad es entre los vínculos vigentes, no sobre el historial. |
| Un co-tutor no es el dueño | Acceso de co-tutor | No se otorga acceso al tutor que ya es dueño de esa mascota: ya lo alcanza todo, y la fila lo dejaría con dos títulos distintos sobre el mismo animal. |
| Un acceso por persona y mascota | Acceso de co-tutor | No se otorgan dos accesos vigentes al mismo tutor sobre la misma mascota. Cambiar de nivel es editar el que existe, no agregar otro. |
| Consentimiento previo al acceso | Acceso de co-tutor | No se crea un acceso si quien lo recibe no tiene `consentimiento_datos = true`. Es la regla espejo de la del alta de paciente: recibir el historial de una mascota ajena es recibir dato de salud. |
| Solo se comparte una mascota vigente | Vínculo con clínica, Acceso de co-tutor, Invitación | No se comparte, invita ni revoca sobre un Paciente con `deleted_at` (regla 4.5). |

### 2.3 Bloqueo de canal

| Regla | Aplica a | Validación |
|---|---|---|
| Tutor solo por móvil | Usuario (tipo_usuario = tutor) | Todo intento de autenticación desde el canal web se rechaza, independientemente de que el email y la contraseña sean correctos. |
| Veterinario, los dos canales | Usuario (tipo_usuario = veterinario) | Puede autenticarse desde web y desde móvil. |
| Clínica_admin solo por web | Usuario (tipo_usuario = clínica_admin) | Todo intento de autenticación desde el canal móvil se rechaza, con el mismo criterio. |

> **El tutor estuvo un tiempo habilitado en la web y se lo volvió a sacar.** Se lo había abierto para el que no tiene el teléfono a mano, con el argumento de que ningún dato ni ninguna acción suya cambian con el canal. Sigue siendo cierto y no alcanza: el producto del tutor **es** la aplicación. Los avisos (4.15), la cámara y la copia local que le da acceso sin conexión (Sincronización sin Conexión) dependen del aparato, y una web que le da todo eso a medias es peor que una que lo manda de entrada a donde el producto está entero. Los dos bloqueos que quedan en pie —el del tutor y el del clínica_admin— son de producto y no de dispositivo.

> Esta regla se valida en el backend al momento de autenticar, no solo ocultando opciones en la interfaz — mismo criterio que el resto del motor de permisos (sección 3). Ver Alcance de Plataformas para la matriz completa de plataforma por rol.
>
> El canal lo declara el cliente en el pedido de autenticación (campo `canal`, ver Arquitectura 4.4). No es una barrera contra un request armado a mano: es el único lugar donde el backend puede aplicar la regla, y así queda registrado explícitamente en vez de darse por supuesto. La renovación de token no vuelve a declararlo — lo hereda de la sesión — para que refrescar no sea una vía de evadirlo.

### 2.4 Borrado

| Regla | Aplica a | Validación |
|---|---|---|
| Nunca borrado físico | Todas las entidades clínicas | Ninguna operación de la aplicación ejecuta DELETE físico sobre Evento clínico, Medicación, Cita, Adjuntos o Paciente. Toda baja es lógica (deleted_at). |
| Quién puede borrar (lógicamente) | Evento clínico, Medicación | **Depende del `cargado_por` del registro.** Los de origen `veterinario`: cualquier veterinario de una clínica con vínculo vigente sobre ese paciente, no solo el autor. Los de origen `tutor`: el dueño de la mascota y el co-tutor con edición, y también el veterinario. Lo que ningún tutor puede es dar de baja un registro de origen `veterinario`, sea dueño o co-tutor. |
| Quién puede renombrar | Adjuntos | Solo quien lo subió, mismo criterio que la baja. Cambia `nombre_archivo` y nada más: el archivo no se edita (Modelo de Datos, 4.8 y proceso 4.14). |
| Quién puede borrar (lógicamente) | Adjuntos | Cada rol retira los adjuntos que subió: el veterinario los suyos, el tutor los suyos. Un tutor no borra el estudio que cargó la clínica, ni la clínica la ficha histórica que subió el tutor. La baja es lógica y **no borra el objeto del bucket**: "nunca borrado físico" alcanza también al archivo. |
| Quién puede borrar (lógicamente) | Cita | Un veterinario de la clínica de la cita, el dueño de la mascota o un co-tutor con nivel de edición. Es la excepción a "el tutor solo da de baja lo que él declaró": la Cita es agenda, no historial — el tutor que sabe que no va a llevar a su mascota es quien mejor puede retirarla del calendario, y la baja no destruye nada que se haya registrado sobre la atención. |
| Quién puede borrar (lógicamente) | Tutor | Solo un veterinario de una clínica vinculada a esa ficha. La baja marca deleted_at y no cascadea sobre la cuenta de Usuario del tutor. **Se rechaza mientras la ficha tenga Pacientes vigentes**: dar de baja al tutor dejaría mascotas activas sin nadie a quien contactar. La clínica tiene que resolver primero qué pasa con ellas. |
| Quién puede borrar (lógicamente) | Paciente | **Solo el dueño.** Ni un co-tutor, ni siquiera con nivel de edición, ni un veterinario. No cascadea sobre Eventos clínicos, Medicación, Citas ni Adjuntos (regla 4.5). |
| Quién puede revocar | Vínculo con clínica | El dueño de la mascota, sobre cualquier clínica. Un veterinario, solo sobre su propia clínica — es cómo la saca de su cartera. **Se rechaza mientras esa clínica tenga Citas pendientes de esa mascota**, informando cuántas. |
| Quién puede revocar | Acceso de co-tutor | El dueño, sobre cualquier acceso que otorgó. El propio co-tutor, sobre el suyo — renunciar a ver una mascota ajena no necesita permiso de nadie. |
| Quién puede borrar (lógicamente) | Veterinario | Solo el clínica_admin de su misma clínica. La baja marca deleted_at sobre la ficha **y desactiva su cuenta de acceso en la misma transacción**: separarlas dejaría a un ex empleado con acceso cada vez que alguien olvidara el segundo paso. No cascadea sobre lo que ese veterinario escribió — los Eventos clínicos y las Medicaciones conservan su autoría. |

> **Que la baja del Paciente haya pasado del veterinario al dueño** es el cambio de comportamiento más grande de esta versión, y no es un ajuste de permisos: es la consecuencia de que la mascota deje de pertenecer a una clínica. Con una mascota atendida por tres, dejar que cualquiera de ellas la dé de baja es dejar que una borre el registro de las otras dos. Lo que el veterinario tiene en su lugar es **revocar el vínculo de su clínica**, que es lo que en realidad quería hacer cuando daba de baja una ficha que dejó de atender, y que hasta ahora no tenía cómo expresar.
>
> **Revocar un vínculo o un acceso no borra nada.** Los Eventos clínicos, la Medicación y los Adjuntos que esa clínica o esa persona escribieron quedan donde están, con su autoría intacta: quién escribió qué no depende de quién puede leerlo hoy. Lo que se pierde es el acceso de ahí en adelante.
>
> **Revocar con citas pendientes se rechaza**, con el mismo criterio con el que achicar el horario de atención se rechaza mientras haya citas fuera del nuevo (regla 2.2) y con el que no se da de baja un tutor con pacientes vigentes. Vaciar la agenda de una clínica como efecto colateral de que alguien tocó un botón en su teléfono sería una baja silenciosa: primero se resuelve qué pasa con esos turnos.

> Toda regla de esta sección que se viole debe rechazar la operación antes de escribir en base — no como validación posterior ni advertencia silenciosa.

### 2.5 Quién da de alta cada tipo de cuenta

| Tipo de cuenta | Quién la crea | Por qué |
|---|---|---|
| Tutor | El propio tutor, por auto-registro público (proceso 4.9) | Es el único punto de entrada abierto del sistema: el tutor debe poder registrarse y empezar a usar la aplicación sin intervención de una clínica. |
| Veterinario | El clínica_admin de la clínica a la que pertenece | Es la gestión del plantel de la clínica (Alcance de Plataformas, 3.2). La cuenta queda con clínica_de_pertenencia_id = la clínica del administrador que la da de alta; no es un dato que el cliente pueda elegir. |
| Clínica_admin | El administrador de la plataforma, fuera de la API (proceso 4.10) | No existe ningún rol dentro del sistema con permiso para crearla: exponerla por HTTP convertiría a cualquier clínica_admin en una vía de escalada de privilegios. |

> **El clínica_admin crea la cuenta, pero no la estrena**: la deja con activación pendiente y el veterinario define su propia contraseña con el token que le llega por correo (4.12). Quien administra el plantel decide quién entra, no con qué credencial.

> En consecuencia, el endpoint de alta de usuarios de la API solo acepta cuentas de tipo veterinario. Las de tutor entran por el registro público y las administrativas por fuera de la API. Esta misma restricción aplica al alta vía Google (4.7): solo el flujo de tutor puede originar una cuenta nueva.

### 2.6 Dirección

La dirección de Tutor y de Clínica son cuatro campos que viajan juntos: el texto, el `place_id` del proveedor de mapas y el par lat/lng (Modelo de Datos, 3.1). Confirmar el punto en el mapa es opcional; lo que no es opcional es que los cuatro campos sean coherentes entre sí.

| Regla | Aplica a | Validación |
|---|---|---|
| Las coordenadas van completas o no van | Tutor, Clínica | `dirección_place_id`, `dirección_lat` y `dirección_lng` se cargan y se limpian **los tres juntos**. Una ficha con latitud y sin longitud, o con un punto sin el identificador que permita refrescarlo, es un dato a medias que ningún consumidor puede usar. |
| No hay punto sin dirección | Tutor, Clínica | No se aceptan coordenadas con `dirección` vacía. Un pin que no se puede leer como domicilio no sirve ni para mostrar ni para verificar. |
| La dirección arrastra su punto | Tutor, Clínica | Editar `dirección` sin enviar los otros tres campos **limpia** el `place_id` y las coordenadas. Conservarlos dejaría el pin apuntando a la casa anterior mientras el texto ya dice otra cosa, y esa ficha miente en las dos direcciones: el mapa contradice al texto y nada indica cuál de los dos es el dato bueno. Volver a confirmarla en el mapa es una acción explícita de quien edita. |
| Coordenadas en rango | Tutor, Clínica | Latitud entre -90 y 90, longitud entre -180 y 180. Es la validación mínima contra un cliente que manda cualquier cosa. |
| El backend no verifica contra el proveedor | Tutor, Clínica | El punto y el `place_id` los declara el cliente y se persisten tal como llegan: el backend **no** vuelve a consultar el servicio de mapas para confirmarlos. |

> **Que el backend no re-verifique es una excepción deliberada al principio de que el cliente no decide nada** (Arquitectura, sección 3). Un cliente manipulado puede guardar una dirección con un punto que no le corresponde. Se acepta porque lo que está en juego es un dato declarado por el propio titular sobre sí mismo —el tutor sobre su domicilio, la clínica sobre el suyo—, no un permiso ni un dato clínico: falsearlo no da acceso a nada. Re-verificar cada escritura pondría una dependencia de red externa en el camino de guardar una ficha, y una caída del proveedor pasaría a impedir editar un tutor.
>
> El día que las coordenadas dejen de ser un dato de contacto y pasen a decidir algo —el matching geolocalizado de urgencias de Fase 2 elige a quién derivar— esta excepción hay que revisarla, porque ahí sí el punto empieza a tener consecuencias sobre terceros.
>
> **Quién puede escribir la dirección no cambia con esto**: es exactamente quien ya podía escribir la ficha (sección 3.2). En Tutor, el propio tutor sobre la suya y el veterinario vinculado; en Clínica, el clínica_admin sobre la propia. El campo es parte de la ficha, no un permiso aparte.
>
> La dirección es **dato personal alcanzado por la Ley 25.326**, igual que el documento: queda cubierta por el `consentimiento_datos` que el tutor ya otorga al registrarse (4.9) y por la auditoría de la ficha. No se agrega un consentimiento nuevo por guardar el punto en el mapa — es el mismo domicilio que ya se pedía, con más precisión.

### 2.7 Telemetría de producto

El registro de uso (Modelo de Datos, 4.17) es la única entidad del sistema que existe para ser mirada en masa, y por eso es la que más se equivoca si se la deja crecer sola. El catálogo completo está en Telemetría de Producto, sección 5; las reglas duras son estas cuatro.

| Regla | Validación |
|---|---|
| Nombre del catálogo | `nombre` pertenece al enum del catálogo. Un evento con un nombre desconocido se **descarta en silencio**, no se rechaza el lote. |
| Propiedades por lista permitida | Cada evento declara qué claves acepta en `propiedades`. Las que no están se descartan al recibirlas, antes de persistir. |
| Nunca dato clínico, texto libre ni `paciente_id` | Ninguna propiedad admite texto escrito por una persona, ni identifica a una mascota. Los valores son enums, números y booleanos. |
| El actor sale del token | `usuario_id`, `rol` y `clínica_id` los completa el backend desde la sesión. Si vienen en el cuerpo, se ignoran. |

> **Descartar en silencio es correcto acá y en ningún otro lado de la API.** Un cliente que emite un evento mal armado no tiene nada que corregir ni al usuario a quien avisarle: devolverle un error lo empujaría a reintentar en loop lo que nunca va a ser aceptado. Lo descartado se cuenta en el log del backend, que es donde se ve que una versión nueva del cliente está emitiendo mal.
>
> **La telemetría nunca bloquea al usuario.** Si el registro falla —la ingesta, o la emisión desde la capa de negocio—, la acción sigue adelante y el error queda en el log. Un evento perdido es un dato menos; una carga clínica que no se guarda porque falló una métrica es un incidente.
>
> El `usuario_id` es **dato personal** y queda alcanzado por el consentimiento que el tutor ya otorga (4.9) y por el plazo de retención de 13 meses. Un pedido de supresión borra también la telemetría del titular: ninguna obligación legal la retiene, a diferencia del historial clínico.

## 3. Motor de permisos

La matriz de permisos definida en el Modelo de Datos (sección 5) es la referencia; esta sección describe el servicio que la aplica en tiempo de ejecución, sobre cada lectura y escritura.

### 3.1 Principio general

Todo acceso a un recurso (Paciente, Evento clínico, Medicación, Cita, Adjuntos) se evalúa en dos niveles, en este orden:

1. **Rol** — ¿el tipo_usuario tiene permiso de escribir/leer este tipo de entidad? (según la matriz del Modelo de Datos).
2. **Alcance** — dado que tiene el rol correcto, ¿tiene permiso sobre este registro específico?

El segundo nivel es el que evita que, por ejemplo, un veterinario habilitado para escribir Eventos clínicos pueda editar el historial de un paciente que no es suyo.

### 3.2 Reglas de alcance

| Rol | Regla de alcance |
|---|---|
| Veterinario | Solo accede a Pacientes que tienen un **vínculo vigente** con su clínica (Modelo de Datos, 4.13). Cualquiera del plantel los alcanza: el vínculo es con la clínica, no con la persona. |
| Tutor | Solo accede a Pacientes de los que es **dueño** (`Paciente.tutor_id`) o sobre los que tiene un **acceso de co-tutor vigente** (4.14). Qué puede hacer sobre ellos lo decide el nivel de ese vínculo. |
| Clínica_admin | Solo accede a datos administrativos (Veterinarios, Clínica) cuya clínica_id coincide con la propia. Sin acceso a Evento clínico ni Medicación de ningún paciente (ver Modelo de Datos, sección 5). |

Sobre la ficha de Tutor (distinta de la cuenta de Usuario del tutor), el alcance se resuelve así:

| Rol | Regla de alcance sobre Tutor |
|---|---|
| Veterinario | **Busca** sin acotar (ver nota abajo). Para **leer, editar o dar de baja** una ficha concreta necesita un vínculo con su clínica: que el tutor sea dueño o co-tutor vigente de al menos un Paciente vinculado a esa clínica, o que la ficha la haya creado esa misma clínica (`clínica_de_origen_id`). |
| Tutor | Solo alcanza su propia ficha, vía el tutor_id de su Usuario: puede leerla y editar nombre, contacto, dirección y documento. No puede darla de baja — la baja la decide la clínica, que es quien tiene Pacientes vinculados a ella — ni revocar su consentimiento por esta vía. |
| Clínica_admin | **Busca en el padrón en proyección reducida** —nombre, contacto y si ya tiene documento cargado— y **da de alta una ficha** con nombre, contacto y consentimiento. No abre la ficha completa, no ve documento ni dirección, no edita lo ya cargado y no da de baja. |

> **La búsqueda del padrón no se acota por clínica, y con el clínica_admin pasa lo mismo que con el veterinario**: antes del alta no hay ningún vínculo contra el cual acotarla. Lo que cambia es la proyección — el admin ve lo justo para reconocer a la persona que llama y decidir si ya está: nombre, cómo contactarla, y si la ficha está completa o le falta el documento.
>
> **El documento entra en la búsqueda sin entrar en la respuesta.** El mostrador que tiene el DNI delante lo tipea y encuentra la ficha; lo que vuelve sigue siendo nombre, contacto y si tiene documento cargado, nunca el número. Buscar por un dato y leerlo son dos permisos distintos, y acá se da el primero: sin él, el rol que más veces tiene el documento a mano es el único que no puede usarlo para no crear una ficha repetida.
>
> El costo asumido es que quien tiene este buscador puede **confirmar si un documento está en el padrón**, aunque no pueda leer el de nadie. Es el mismo tipo de filtración que ya tiene la búsqueda por contacto, y se acepta por el mismo motivo: es un rol de la clínica, autenticado y auditado, no una pantalla pública.
>
> Es la proyección reducida que esta misma sección tiene anotada como **pendiente** para la búsqueda del veterinario. Se implementa primero para el rol que se está abriendo ahora; la del veterinario sigue devolviendo la ficha completa y la deuda sigue abierta.
>
> Del alta escribe **nombre, contacto y consentimiento**, lo mismo que pide el auto-registro (4.9). Documento y dirección los completa el veterinario cuando atiende, que es cuando los tiene delante.

Sobre el Paciente, el alcance se resuelve así:

| Rol | Regla de alcance sobre Paciente |
|---|---|
| Veterinario | La cartera de su propia clínica —las mascotas con vínculo vigente—: da de alta a nombre de un tutor, lista, lee, edita los datos básicos y el chip. **No da de baja la mascota**: lo que puede hacer es revocar el vínculo de su clínica (regla 2.4). |
| Tutor **dueño** | Sus mascotas, estén atendidas por las clínicas que sea o por ninguna. Da de alta, lee, edita todos los datos no clínicos, gestiona las citas y los adjuntos, da de baja, y **administra los accesos**: comparte con clínicas, invita co-tutores, les cambia el nivel y los revoca. |
| Tutor **co-tutor con edición** | Todo lo del dueño **salvo administrar**: no invita, no revoca, no cambia niveles y no da de baja la mascota. Sí lee el historial completo, edita los datos no clínicos, reagenda y retira citas, y sube adjuntos. |
| Tutor **co-tutor con lectura** | Mira. El historial, la medicación, las citas y los adjuntos de esa mascota, sin escribir nada — ni el peso. |
| Clínica_admin | **La cartera en proyección reducida** —nombre, especie y a quién llamar— y el **alta a nombre de un tutor**, que vincula su clínica en la misma operación (proceso 4.1). No abre la ficha, no lee el historial ni la medicación, no edita lo ya cargado y no da de baja. El chip sigue siendo del veterinario. |

> **La cartera es una proyección, no la ficha.** Existe porque agendar exige elegir una mascota, y sin poder nombrarla el mostrador no puede tomar un turno. Lo que protege el dato no es el alcance sino la proyección, con el mismo criterio que el directorio de Clínicas que lee el tutor (3.2): salen nombre, especie y a quién llamar, y nada que cuelgue de la mascota — ni fecha de nacimiento, ni sexo, ni peso, ni chip.
>
> Es poco más de lo que la agenda ya le muestra, que trae el nombre y la especie de cada cita. Lo que agrega es poder encontrar a una mascota que **todavía no tiene ninguna**, que es justo el caso del primer turno.
>
> Acotada a la cartera de su propia clínica, no al padrón: a diferencia de la búsqueda de Tutor, que el veterinario hace sin acotar porque necesita encontrar fichas de otras clínicas para el alta, acá no hay ningún proceso que lo justifique.

Sobre los datos de la ficha, la línea no la marca el rol sino qué dato es:

| Dato | Quién lo edita |
|---|---|
| nombre, especie, raza, fecha_nacimiento, sexo, peso_actual | El dueño, el co-tutor con edición y el veterinario de una clínica vinculada |
| identificador_externo (chip) | **Solo el veterinario** |

> Que el tutor edite los datos básicos y no solo el peso es consecuencia directa de que ahora da de alta la mascota: quien carga una ficha tiene que poder corregirle el nombre. Son datos del animal, no del acto médico, y no dejan de serlo porque la mascota entre a una clínica.
>
> El **chip** es la excepción, y no por desconfianza: lo implanta y lo lee el veterinario, es único entre fichas vigentes (Modelo de Datos, 4.2), y dejarlo abierto habilitaría que alguien ocupe el número de otro animal desde su teléfono.

Sobre los Adjuntos, el alcance se resuelve así:

| Rol | Regla de alcance sobre Adjuntos |
|---|---|
| Veterinario | Los archivos de los pacientes vinculados a su clínica: sube, lista, descarga y retira los que subió él. |
| Tutor | Los archivos de las mascotas que alcanza. El dueño y el co-tutor con edición suben y retiran los que subieron; el co-tutor con lectura solo lista y mira. |
| Clínica_admin | Sin acceso: los adjuntos cuelgan del Paciente, y su rol no alcanza las mascotas atendidas. |

> Un Adjunto no se edita. Corregir una carga errónea es retirarla y subir otra: la alternativa sería permitir reemplazar el archivo debajo de un registro que ya se referenció desde un evento clínico.

Sobre la Cita, el alcance se resuelve así:

| Rol | Regla de alcance sobre Cita |
|---|---|
| Veterinario | El calendario de los pacientes vinculados a su clínica: agenda —siempre en su propia clínica—, lista, lee, reagenda, asigna profesional y da de baja. **La asignación no acota el alcance**: un veterinario alcanza esas citas le toquen a él o a un colega. Que le asignen una cita es organización del trabajo, no un permiso. |
| Tutor | Las citas de las mascotas que alcanza. El dueño y el co-tutor con edición leen, reagendan (fecha_programada), deciden si quieren que les avisen (notificar_tutor) y pueden darlas de baja; el co-tutor con lectura solo lee. Ningún tutor agenda citas nuevas, cambia el tipo ni asigna profesional: qué control corresponde y quién lo hace son criterio de la clínica. |
| Clínica_admin | La agenda de su propia clínica: lista, lee, **agenda, reagenda y reparte**, igual que el veterinario. Lo único que no hace es **dar de baja una cita**. El alcance sale de `Cita.clínica_id` y no del Paciente: la cita dice en qué clínica se agenda, y esa es la que el rol administra. |

> **La agenda es el libro de turnos, no el historial.** Que el clínica_admin lea las citas de su clínica no afloja el alcance sobre el Paciente: sigue sin poder abrir una ficha, un historial, una medicación ni un adjunto, y sigue sin asentar la atención. Una Cita no lleva diagnóstico: dice cuándo viene quién, que es la pregunta del mostrador.
>
> Reemplaza al criterio anterior —"sin acceso: el calendario cuelga del Paciente"—, que trataba a la agenda como si fuera dato clínico. El costo asumido es explícito: "Luna, cirugía, martes 10:00" es información de salud de un paciente identificado.
>
> **Escribe todo lo de la agenda menos la baja.** La versión anterior le dejaba solo la asignación, con el argumento de que agendar y mover un turno son criterio clínico. Ese criterio esta misma tabla lo reserva frente al **tutor** —"qué control corresponde y quién lo hace son criterio de la clínica"— y el clínica_admin *es* la clínica: tomar y mover turnos es la tarea del mostrador.
>
> Retirar una cita del calendario sigue siendo del veterinario y del tutor: decir que algo **no va a ocurrir** lo sabe quien atiende o quien lleva a la mascota, no quien administra.
>
> Agendar exige elegir una mascota, y para eso el rol lee la **cartera** de su clínica en proyección reducida — ver el alcance sobre Paciente, más abajo.

> El listado de pacientes es un endpoint con dos alcances: cuál aplica lo decide el rol del token, nunca un parámetro del cliente. El veterinario ve la cartera de su clínica; el tutor, sus mascotas.
>
> El listado de **citas** funciona igual y por el mismo motivo, con **tres** alcances y no dos: el veterinario y el clínica_admin ven la agenda de su clínica, y el tutor las citas de sus mascotas. Existe además del calendario de una mascota porque las dos preguntas son distintas: "qué le toca a Luna" cuelga del Paciente, pero "qué tiene la clínica esta semana" (Alcance de Plataformas, 3.6) no cuelga de ninguna mascota en particular. Sin él, la agenda habría que armarla pidiendo el calendario de cada paciente de la cartera, uno por uno.

Sobre la Clínica, el alcance se resuelve así:

| Rol | Regla de alcance sobre Clínica |
|---|---|
| Clínica_admin | Su propia clínica, la de su `clínica_id`: la lee y edita sus datos administrativos (nombre, dirección, contacto), la duración del turno y sus **Franjas de atención** (Modelo de Datos, 4.18), que escribe como conjunto completo. También carga y da de baja las **Ausencias** de su plantel (4.19). **No la da de alta ni de baja** — eso es del administrador de la plataforma, fuera de la API (proceso 4.10). |
| Veterinario | Solo lectura de la clínica a la que pertenece, vía su `clínica_de_pertenencia_id`, incluidas sus Franjas de atención y las Ausencias del plantel. Las necesita para agendar: sin la grilla no sabe qué horas son válidas, y sin las ausencias ofrecería turnos a quien no va a estar (regla 2.2). No las edita. |
| Tutor | **Lee el directorio**: nombre, dirección y contacto de cualquier clínica, para poder elegir con cuál compartir su mascota. No lee su plantel ni sus ausencias, y no edita nada. Sí alcanza la **grilla de la clínica que atiende a su mascota**, y solo esa: sin ella no podría elegir hora al pedir una reagenda (Alcance de Plataformas, 5.4). |

> Que el clínica_admin **edite** su propia clínica no estaba en la primera versión de este documento: la matriz del Modelo de Datos la daba como escritura exclusiva del administrador de la plataforma, mientras que Alcance de Plataformas 3.2 pedía la pantalla de edición desde la web. Era una contradicción entre dos documentos del mismo contrato. Se resolvió a favor de Alcance de Plataformas: el alta sigue siendo del administrador de la plataforma, la edición pasa a ser del clínica_admin. Que una clínica corrija su propio teléfono no puede depender de abrirle un ticket a Wayka.

Sobre la ficha de Veterinario (el plantel de la clínica), el alcance se resuelve así:

| Rol | Regla de alcance sobre Veterinario |
|---|---|
| Clínica_admin | Alcanza el plantel de su propia clínica: lo da de alta, lo lista, lo lee, lo edita y lo da de baja. Es la gestión de su plantel (Alcance de Plataformas, 3.2). |
| Veterinario | Lee y lista el plantel de su propia clínica, pero no lo modifica. Necesita saber quién más atiende ahí para leer la trazabilidad de lo que otros escribieron; darse de alta o editarse a sí mismo sería administrar la clínica. |
| Tutor | Sin acceso: el plantel es un dato administrativo de la clínica, no del tutor. |

> **La búsqueda queda deliberadamente sin acotar**: es como el veterinario resuelve si el tutor ya existe antes de que exista ningún vínculo (proceso 4.1). Sin ella no habría forma de encontrar la ficha de un tutor que se auto-registró para darle de alta su primera mascota.
>
> La contrapartida asumida es que el listado devuelve la ficha completa, incluidos documento y dirección, para cualquier persona del sistema. Acotar la lectura por id mientras la búsqueda sigue exponiendo el mismo dato deja el acotamiento a medias. **Pendiente**: que el listado devuelva una proyección reducida (id, nombre, contacto parcial, si tiene documento cargado) y que la ficha completa solo salga por el endpoint acotado.
>
> `clínica_de_origen_id` cubre la ventana entre crear la ficha y crear el Paciente que justifica el alcance. Para un tutor **auto-registrado** ese campo es NULL, así que su ficha no está al alcance de ninguna clínica hasta que una le dé de alta una mascota: en ese caso el veterinario crea primero el Paciente y después completa el documento y la dirección de la ficha. Es un cambio de orden respecto de 4.9, que describía la completitud sin decir en qué momento ocurre.

> A diferencia del alcance sobre Tutor, este sí está acotado por clínica: la ficha nace con la clínica que la dio de alta y no se muda de clínica, así que siempre hay dato contra el cual acotarlo.
>
> Que el veterinario pueda **leer** el plantel de su clínica no estaba definido en los documentos: se decidió al implementar la entidad. La alternativa era reservar el plantel entero al clínica_admin, pero eso dejaría al veterinario sin poder resolver quién firmó un Evento clínico de su propia clínica. Si el criterio no es el correcto, es el punto a revisar.

Sobre el Evento clínico y la Medicación, el alcance se resuelve así:

| Rol | Regla de alcance sobre Evento clínico y Medicación |
|---|---|
| Veterinario | Sobre los Pacientes vinculados a su propia clínica: carga, lista, lee, **edita y da de baja cualquier registro de esos pacientes, sea suyo, de un colega o declarado por el tutor**. Escribe siempre con `cargado_por = veterinario`. `usuario_id` registra al autor original y no cambia nunca, ni siquiera al editarlo; `cargado_por` tampoco. |
| Tutor **dueño** y **co-tutor con edición** | Lee el historial completo de las mascotas que alcanza. **Carga antecedentes** (proceso 4.23), que se persisten con `cargado_por = tutor`, y edita y da de baja **únicamente los registros de ese origen**. Sobre un registro de origen `veterinario` no tiene escritura de ninguna clase, en ningún momento. |
| Tutor **co-tutor con lectura** | Lee, y nada más: no carga antecedentes propios ni toca los que cargó otro. |
| Clínica_admin | Sin acceso, ni de lectura ni de escritura (Modelo de Datos, sección 5). |

> **El tutor escribe historial, y lo que lo acota es el `cargado_por` del registro, no un estado ni una ventana de tiempo.** Es el cambio de criterio que el Modelo de Datos explica en su sección 1: la mayoría de las mascotas llega con vacunas, alergias y medicación que solo el dueño puede volcar. La capacidad no se apaga después del alta — se usa desde el onboarding la primera vez y desde la ficha siempre (4.23).
>
> **La asimetría es deliberada**: el veterinario corrige y da de baja lo que declaró el tutor, y el tutor no toca lo del veterinario. No es desconfianza sino competencia. El profesional es la fuente de verdad clínica y tiene que poder arreglar un antecedente mal cargado —una alergia mal escrita en la vista de urgencia es un riesgo, no un detalle—; un tutor editando un diagnóstico ajeno estaría reescribiendo un acto médico. Toda corrección queda en Auditoría con quién la hizo.
>
> **El co-tutor con edición carga antecedentes** con el mismo criterio con el que ya sube adjuntos: alcanza lo mismo que el dueño salvo administrar accesos, y quien cuida al animal la mitad de la semana sabe lo mismo de su historia.

> **El historial ahora tiene autores de otras clínicas.** Una mascota compartida acumula eventos escritos por profesionales que el veterinario que la está leyendo no alcanza —el plantel solo se lee dentro de la propia clínica—, así que la lectura de un Evento clínico devuelve el **nombre del autor y el de su clínica** resueltos por el backend. No es un aflojamiento del alcance sobre la ficha de Veterinario: es el mínimo sin el cual la trazabilidad se vuelve un identificador que nadie puede resolver.

> Que un veterinario pueda editar el evento de un colega es el mismo criterio que la regla 2.4 ya fija para el borrado lógico ("el autor, o un veterinario de la misma clínica"). La alternativa era reservar la edición al autor, pero deja dos criterios distintos sobre la misma entidad: un colega podría borrar el registro entero y no corregirle una fecha. La trazabilidad no se pierde: `usuario_id` conserva la autoría, `cargado_por` conserva de qué clase de registro se trata, y la Auditoría registra quién editó qué y cuándo.

Sobre los **agregados de gestión** de la clínica, el alcance se resuelve así:

| Rol | Regla de alcance sobre los agregados |
|---|---|
| Clínica_admin | Su propia clínica, la de su `clínica_id`. Lee **conteos** de Cita, Consulta atendida y Vínculo con clínica: ocupación de la grilla, citas sin asignar, atenciones por período y por profesional, vínculos vigentes y altas del período. Nunca la fila individual. |
| Veterinario, Tutor | Sin acceso por esta vía. Los dos ya leen los registros que les corresponden, uno por uno, con su propio alcance. |

> **Un conteo sin `paciente_id` ni dato clínico no es historial clínico.** Es la decisión que la matriz del Modelo de Datos (sección 5) dejaba pendiente de tomar explícitamente. El rol que administra la clínica no podía saber si su propia agenda se estaba usando, y negarle un número que no identifica a nadie no protegía a ningún paciente.
>
> **El límite es la forma del dato, no la entidad.** Lo que sale es un número con su corte —período, profesional, `origen`—, nunca una lista de registros. Un listado de "atenciones del martes" con hora y profesional reconstruye quién fue atendido en cuanto se cruza con cualquier otra cosa, aunque no traiga el `paciente_id`. Por eso la regla se aplica sobre la respuesta y no sobre la tabla de origen.
>
> **El Evento clínico y la Medicación quedan afuera incluso agregados.** No es una inconsistencia con lo anterior: el agregado que los necesitaría es la cobertura por profesional —cuántas atenciones terminaron con historial cargado—, y esa es una medición del desempeño de una persona, no de la operación de la clínica. Se decidió dejarla fuera del alcance del rol. Sigue calculándose para el piloto por SQL (Telemetría de Producto, 9), que es donde vivía desde el principio.
>
> **El período es el corte, y es explícito.** Todo agregado se pide por semana o por mes; no hay un "desde siempre" que devuelva un número sin denominador. Las horas se leen en la `zona_horaria` de la clínica (Modelo de Datos, 4.3), igual que la Cita y la Consulta atendida.

Sobre la entidad Usuario, el alcance se resuelve así:

| Rol | Regla de alcance sobre Usuario |
|---|---|
| Clínica_admin | Alcanza las cuentas cuya clínica_de_pertenencia_id coincide con la propia. No alcanza cuentas de tutor (no tienen pertenencia) ni cuentas de otra clínica. No puede darse de baja a sí mismo: dejaría a la clínica sin ninguna cuenta capaz de administrar usuarios. |
| Veterinario, Tutor | Solo alcanzan su propia cuenta: pueden leerla, cambiar su email y su contraseña. |
| Cualquier rol | El campo `activo` solo lo modifica un clínica_admin dentro de su alcance. Permitir que el propio usuario lo cambie vaciaría de sentido la suspensión. |

Sobre los vínculos —con quién se comparte una mascota—, el alcance se resuelve así:

| Rol | Regla de alcance sobre Vínculo con clínica, Acceso de co-tutor e Invitación |
|---|---|
| Tutor **dueño** | Administra: comparte con una clínica, invita a un co-tutor, le cambia el nivel, revoca cualquiera de los dos y anula una invitación que todavía no se canjeó. |
| Tutor **co-tutor** | Lee la lista completa de quién más ve la mascota, en cualquier nivel, y **sin acciones**: no comparte, no invita y no revoca. Lo único que puede hacer es **renunciar a su propio acceso**. |
| Veterinario | Lee los vínculos de los pacientes que alcanza —a qué tutores contactar y qué otras clínicas la atienden— y puede **revocar el vínculo de su propia clínica**. No comparte una mascota ajena con nadie. |
| Clínica_admin | Sin acceso. |

> Que un co-tutor **vea la lista y no pueda tocarla** es deliberado. Saber quién más está mirando el historial de un animal no es una capacidad administrativa: es lo mínimo para entender con quién se está compartiendo. Ocultárselo dejaría a alguien leyendo datos de salud sin saber quién más los lee.
>
> El **dueño no se revoca a sí mismo** ni renuncia a su propia mascota: no hay adónde iría el título. Lo que existe para dejar de tenerla es la baja (regla 2.4), y la transferencia de propiedad sigue fuera de alcance (Modelo de Datos, 4.2).

> **Cómo obtiene el motor estos vínculos.** El token de acceso transporta el rol y los identificadores de la cuenta, pero no puede transportar con qué mascotas hay vínculo: eso cambia sin que la sesión cambie. El servicio los resuelve **antes** de autorizar y se los pasa al motor como un hecho ya calculado — es el mismo patrón que ya se usaba para saber si una clínica atiende a un tutor, y la razón por la que la evaluación de permisos sigue sin consultar la base por su cuenta.
>
> La regla que sostiene el patrón: **la consulta devuelve hechos, no permisos.** Lo que viaja hacia el motor es "el nivel de este tutor sobre esta mascota" y "esta clínica la atiende"; nunca "puede editar". La decisión sigue viviendo en un solo lugar, y una consulta mal escrita no puede otorgar un permiso que la tabla de permisos no otorga.

### 3.3 Dónde se aplica

El motor de permisos se evalúa en la capa de aplicación (servicio/backend), no únicamente en la interfaz. Ninguna pantalla debe ser el único punto de control de acceso: toda operación sobre datos clínicos pasa por esta validación aunque la request no provenga de la interfaz oficial.

> Esto es una mitigación directa del riesgo de acceso indebido a historial clínico identificado en el análisis de viabilidad inicial del proyecto.

## 4. Procesos de negocio

Secuencias de pasos que involucran más de una entidad o más de una validación encadenada.

### 4.1 Alta de paciente

1. El Veterinario **o el clínica_admin** inicia el alta. Los dos pueden: el mostrador toma al cliente nuevo que llama, y esperar al veterinario para poder darle un turno no es una división que se pueda explicar en la recepción.
2. Se busca la ficha del Tutor antes de crearla, y **es una sola búsqueda**: lo que se tipea se compara contra el nombre, el contacto y el número de documento a la vez. Buscar por contacto es lo que permite encontrar la ficha de un tutor que ya se auto-registró, porque esa ficha se crea sin documento (4.9) y su contacto es el email del registro; buscar por documento es lo que resuelve el caso inverso, el de la ficha que una clínica ya completó.
   - Que sea un campo y no tres es deliberado: quien atiende el mostrador tiene delante lo que la persona le dice —un apellido, un teléfono, un DNI— y no debería tener que declarar de antemano cuál de las tres cosas es. El documento se compara **por el comienzo del número**, no por un fragmento suelto en el medio: un documento se tipea desde el principio, y buscar `4821` adentro de cualquier número devuelve un ruido que nadie pidió.
3. Si el Tutor no existe en el sistema, se crea primero — valida número_documento único (regla 2.1) y exige consentimiento_datos = true. La ficha queda con `clínica_de_origen_id` = la clínica de quien da el alta, que es lo que la pone a su alcance antes de que exista el Paciente.
   - El clínica_admin la crea con **nombre, contacto y consentimiento**; documento y dirección los completa el veterinario cuando atiende, que es cuando los tiene delante. Es la misma ficha incompleta con la que nace un auto-registro (4.9).
4. Se valida consentimiento_datos = true del Tutor (regla 2.2). Si no existe, se solicita antes de continuar.
5. Se crea el Paciente con `tutor_id` = el Tutor buscado o creado, que queda como **dueño**, y en la misma transacción el **vínculo con la clínica** de quien realiza el alta (`origen = alta_de_la_clínica`).
   - El **chip** (`identificador_externo`) no entra en el alta del clínica_admin: lo carga el veterinario, que es quien lo implanta y lo lee (3.2).
6. Se registra la creación en Auditoría automáticamente (entidad_tipo = "Paciente", acción = "creación"), y también el vínculo.

> El vínculo se crea en la misma transacción que la ficha, y no como un segundo paso: un Paciente que existe pero que ninguna clínica alcanza es una mascota que el veterinario acaba de cargar y no puede ver.
>
> **Este ya no es el único camino de alta.** El tutor da de alta su propia mascota desde la aplicación (4.17), sin ninguna clínica involucrada, y decide después con quién compartirla. La consecuencia práctica es la deduplicación: si el tutor ya la cargó y no la compartió, el veterinario no la ve y va a darla de alta de nuevo. Cuando el número de chip coincide con una ficha que el veterinario no alcanza, el alta se rechaza sugiriendo pedirle al tutor que la comparta — sin revelar nada de esa ficha.

> Si el Tutor se había auto-registrado (4.9), su ficha no tiene clínica de origen y por lo tanto no está al alcance del veterinario hasta este punto: primero se crea el Paciente y recién después se completan documento y dirección de la ficha.

### 4.2 Carga de un evento clínico

1. El Veterinario autenticado selecciona un Paciente dentro del alcance de su clínica (regla 3.2).
2. Se valida que el Paciente esté vigente: sobre una mascota dada de baja no se cargan episodios nuevos (regla 2.2).
3. Se valida fecha no futura (regla 2.2).
4. Se valida `campo_estructurado` contra el esquema fijo del tipo (regla 2.2 y Modelo de Datos, 4.5): vacuna, medicación y alergia lo exigen completo; los demás tipos lo exigen NULL.
5. Se persiste el Evento clínico con `usuario_id` = usuario autenticado y `cargado_por = veterinario` (trazabilidad automática, ninguno de los dos editable ni recibido por la API), y con `consulta_id`, que es el único vínculo que guarda con la atención: si se carga desde una **Consulta atendida** ya asentada, es la suya; si declara la cita que cumple, es la de esa cita, asentada en la misma transacción si todavía no estaba (4.21). Si no declara ninguna, queda en NULL, que es un caso válido y no un error.
6. Se registra la creación en Auditoría con valor_nuevo = contenido del evento.
7. Un evento ya cargado lo puede editar o dar de baja cualquier Veterinario de la clínica del Paciente, no solo su autor, y eso incluye los que declaró el tutor (regla 3.2). La edición nunca reasigna `usuario_id` ni cambia `cargado_por`, y tanto la edición como la baja quedan auditadas.
8. Editar el `tipo` de un evento **descarta el `campo_estructurado` anterior**: el esquema que rige es el del tipo nuevo, y quien cambia el tipo manda los campos que ese tipo exige. Si el tipo nuevo no admite campo estructurado, alcanza con omitirlo.

> Los dos puntos anteriores resuelven casos que los documentos no cubrían y se decidieron al implementar la entidad.
>
> **Evento sobre un Paciente dado de baja**: la regla 4.5 deja el historial de una mascota dada de baja consultable, y también editable — corregir un typo en un registro clínico viejo no depende de que la mascota siga en la cartera. Lo que se rechaza es *agregar* episodios nuevos: no hay atención que registrar sobre una ficha que la clínica cerró. **Pendiente**: el MVP no tiene forma de revertir la baja de un Paciente, así que una ficha cerrada por error deja su historial congelado para altas nuevas. Hay que decidir si la reactivación entra al alcance o si el caso se resuelve dando de alta una ficha nueva.
>
> **Cambio de tipo**: arrastrar el campo estructurado del tipo anterior obligaría a mandar un `null` explícito solo para corregir una carga con el tipo equivocado, que es el caso más común de edición del tipo. El costo asumido es que un cambio de tipo *entre* dos tipos estructurados (de vacuna a medicación, por ejemplo) pierde lo cargado si el cliente no reenvía los campos nuevos — pero esos campos no son traducibles entre esquemas de todos modos.

### 4.3 Ciclo de vida de una medicación

1. Alta: el Veterinario crea un registro de Medicación con fecha_fin = NULL y `cargado_por = veterinario`. Se valida que no exista otra medicación activa con la misma droga para ese paciente (regla 2.2) — **sin mirar el origen de la que ya está**: si la activa la declaró el tutor, el rechazo lo dice, y lo que corresponde es cerrarla y abrir la propia.
2. Cierre: para registrar el fin de un tratamiento, el Veterinario actualiza fecha_fin de la medicación existente — no se crea un nuevo registro para "cerrar" el anterior.
3. Cambio de dosis: se interpreta como cierre de la medicación activa + alta de una nueva (pasos 1 y 2 en secuencia), preservando el historial de dosis anteriores en vez de sobrescribir.
4. Cada cambio de estado dispara un registro en Auditoría (acción = "edición", con valor_anterior y valor_nuevo).
5. Reapertura: enviar fecha_fin = NULL sobre una medicación cerrada la vuelve activa. Es cómo se deshace un cierre cargado por error, y se rechaza si dejara dos activas de la misma droga — la regla 2.2 mirada desde el otro lado.
6. Corrección de una carga errónea: `fecha_fin` es lo único editable de una Medicación, sea cual sea su origen — un antecedente declarado por el tutor se corrige igual que uno del veterinario: dando de baja y cargando de nuevo. Corregir la droga, la dosis o la frecuencia es dar de baja lógica el registro y cargar uno nuevo. Exponer una edición libre de esos campos permitiría sobrescribir una dosis en silencio, que es exactamente lo que el paso 3 viene a evitar; la baja lógica deja el registro errado en el rastro de Auditoría y libera la droga para la medicación activa que corresponde.

### 4.4 Ciclo de vida de una cita

1. Creación: el Veterinario programa una Cita con fecha y hora, en estado = "pendiente", **en su propia clínica**, que tiene que tener vínculo vigente con el paciente. La hora tiene que caer dentro del horario de atención de esa clínica y sobre su grilla de turnos (regla 2.2). Puede asignarle un profesional o dejarla de la clínica para repartirla después.
2. Asignación o reasignación: el Veterinario asigna, cambia o quita el profesional mientras la cita siga pendiente. Se valida que no le quede otra cita en el mismo momento (regla 2.2). Sacar la asignación es dejarla de nuevo de la clínica, no darla de baja.
3. Confirmación o reagenda: el Tutor puede modificar fecha_programada y notificar_tutor dentro de los permisos definidos (sección 3.2), pero no puede cambiar estado directamente. Solo se reagenda una cita pendiente (regla 2.2).
4. Transición a "cumplido": ocurre cuando se asienta la **Consulta atendida** que referencia a esa Cita (4.21), sea porque el Veterinario la asentó al atender o porque la dedujo la carga de un Evento clínico que declara la cita que cumple. Se valida que la Cita sea del mismo Paciente y que no esté ya cumplida. Una Cita **vencida** también pasa a cumplido por esta vía: la mascota llegó tarde y se la atendió igual, y dejarla vencida para siempre falsearía el historial.
5. Transición a "vencido": la cita cuya fecha_programada ya pasó sin haber sido marcada como cumplida cambia de estado automáticamente. Este es un proceso batch/programado, no una acción de usuario.
6. Baja: retirar del calendario una cita que no va a ocurrir es una baja lógica, no un estado. La puede hacer el veterinario de la clínica o el tutor de la mascota (regla 2.4).

> El mecanismo elegido es un job programado (ver sección 4.6) — lo que se fija acá es la regla de negocio: una cita vencida nunca queda indefinidamente en estado "pendiente".

### 4.4.1 Recuperación de contraseña

1. Quien olvidó su contraseña pide el enlace con su correo. El sistema responde igual exista o no la cuenta: contestar distinto lo convertiría en una forma de recorrer el padrón probando direcciones.
2. Si la cuenta existe, está activa y ya se estrenó, se emite un token de un solo uso con vencimiento corto y se manda por correo. **Pedir uno nuevo invalida los anteriores**: un correo viejo reenviado o filtrado no puede seguir abriendo la cuenta.
3. El canje define la contraseña nueva y **cierra todas las sesiones abiertas** de esa cuenta. No emite sesión: para entrar hay que iniciar sesión, que es donde vive el bloqueo de canal (mismo criterio que la activación, 4.16).
4. Una cuenta que todavía no se estrenó queda fuera: ya tiene su token de activación, y recuperarla sería saltear ese proceso.

> **El enlace del correo lleva a distinto lado según quién lo recibe.** A un tutor o a un veterinario les ofrece abrir la aplicación; a un clínica_admin no, porque el bloqueo de canal (2.3) le impide autenticarse desde móvil y mandarlo ahí sería mandarlo a una pantalla donde no puede entrar. Es el único de los cuatro correos del sistema que alcanza a los tres tipos de cuenta, así que es donde la distinción decide algo — el de confirmación es solo del tutor (4.9.1), el de activación por correo solo del veterinario (4.12) y el de invitación va a alguien que va a ser co-tutor (4.19). El detalle del mecanismo está en Arquitectura, 3.8.

> **Cambiar la contraseña cierra las sesiones abiertas, venga de donde venga el cambio** — el propio usuario, el clínica_admin restableciendo la de su plantel, o la recuperación por correo. Recuperar o restablecer es, casi siempre, sospechar que alguien más la tiene: no cerrar lo abierto dejaría adentro justo a quien se quería echar. Alcanza a la sesión de quien la cambia, que va a tener que volver a entrar con la que acaba de elegir.

### 4.5 Borrado lógico en cascada de negocio

Cuando se da de baja lógica un Paciente, sus Eventos clínicos, Medicación, Citas, Consultas atendidas y Adjuntos NO se borran automáticamente en cascada. Quedan visibles y consultables desde el Paciente (aunque el Paciente ya no aparezca en listados activos), preservando la trazabilidad completa del historial. Solo un borrado lógico explícito sobre cada entidad hija la retira de las vistas activas de esa entidad en particular.

Que el historial siga consultable exige que **la ficha del Paciente también lo esté**: sin poder leer la mascota no hay desde dónde consultar lo que cuelga de ella. La lectura por id de un Paciente dado de baja devuelve la ficha, con `deleted_at` completo (Modelo de Datos, 4.2). Lo que se rechaza es la escritura:

| Operación sobre un Paciente con `deleted_at` | Resultado |
|---|---|
| Leer la ficha por id | Devuelve la ficha, con `deleted_at` completo. |
| Listarla entre los pacientes de la clínica o del tutor | No aparece: los listados filtran por `deleted_at IS NULL`. |
| Leer su historial, medicación, citas, consultas atendidas y adjuntos | Se consultan normalmente. |
| Editar la ficha, o darla de baja otra vez | Se rechaza. |
| Compartirla, invitar a un co-tutor o revocar un acceso | Se rechaza. |
| Cargar Eventos clínicos, Medicación, Citas, Consultas atendidas o Adjuntos nuevos | Se rechaza (reglas 2.2). |
| Editar o dar de baja los registros ya existentes | Se permite: corregir un diagnóstico mal cargado no depende de que la mascota siga en la cartera. |

> Esta regla prioriza no perder trazabilidad clínica por sobre la simplicidad de un borrado en cascada — es consistente con el criterio de trazabilidad y no destrucción definido en el Modelo de Datos.
>
> El detalle de la lectura por id se agregó después de la primera versión de esta sección. La regla ya decía que el historial quedaba consultable, pero no decía explícitamente que la ficha misma se leyera, y la API terminó devolviendo un 404 sobre el Paciente mientras sus entidades hijas seguían respondiendo. Era una contradicción del contrato consigo mismo, no una decisión.

### 4.6 Transición automática de citas vencidas (job programado)

1. Un proceso interno del backend corre en forma periódica y consulta las Citas con estado = "pendiente" cuya fecha_programada ya pasó. La frecuencia es **cada hora**, configurable por entorno (`CITAS_VENCIDAS_INTERVAL`), y arranca con una corrida inmediata al levantar el proceso. La comparación es contra el instante actual: desde que `fecha_programada` lleva hora (Modelo de Datos, 4.7), una cita de las 09:00 vence esa misma mañana y no a la medianoche. La frecuencia horaria, que antes era holgada para una fecha sin hora, ahora es la que define la precisión del vencimiento — una cita queda hasta una hora figurando pendiente después de pasar. Es aceptable para un estado que nadie mira en tiempo real; si deja de serlo, se baja el intervalo, no se cambia el mecanismo.
2. El job invoca el mismo servicio de la capa de negocio que usaría cualquier otro cambio de estado de Cita — no escribe directo en la base de datos, para no crear un camino que evada las validaciones ya definidas.
3. Cada Cita transicionada a "vencido" se registra en Auditoría con usuario_tipo = "sistema" y usuario_id = NULL (ver Modelo de Datos, sección 4.10).
4. Cada Cita se vence en su propia transacción y cada corrida tiene un techo de citas: que una falle no puede dejar sin vencer a las demás, y una mora acumulada grande no se procesa en una transacción única. Lo que queda afuera del techo lo toma la corrida siguiente.

> El job es un componente interno del backend, no un cliente externo — arquitectónicamente entra a la capa de negocio directamente, sin pasar por la capa de presentación/API (no hay sesión de usuario que autenticar).

### 4.7 Alta o vinculación de cuenta vía Google

1. El cliente (web o móvil) obtiene un ID token de Google y lo envía al backend.
2. El backend verifica ese token contra Google (firma, expiración, audiencia) — nunca confía en el email que el cliente afirma tener sin esa verificación.
3. Se busca un Usuario existente por email:
   - **Si existe** (por ejemplo, fue creado antes con email + contraseña), se vincula google_id y avatar_url a ese Usuario existente. No se crea una cuenta duplicada.
   - **Si no existe**, la creación de una cuenta nueva vía Google solo está permitida dentro del flujo de auto-registro de tutor (4.9) — ahí Google es un método alternativo a la contraseña, no una vía de alta distinta. Para veterinario y clínica_admin, Google únicamente puede vincularse a una cuenta ya existente (caso anterior); si no hay ninguna, el login se rechaza — esas cuentas solo se crean por los procesos definidos en 2.5.
4. El bloqueo de canal (regla 2.3) se valida igual que con cualquier otro método de autenticación — Google no es una vía para evadirlo.
5. Se emite el token de acceso + refresco igual que en el flujo de autenticación por contraseña (Arquitectura, sección 4).

> La vinculación automática por email asume que el email que devuelve Google ya está verificado por ellos — es una garantía del proveedor, no una suposición de Wayka. La restricción del paso 3 evita que este proceso se convierta en una puerta de alta paralela a la sección 2.5: sin ella, cualquiera podría crear una cuenta con un tipo_usuario indefinido, violando la regla de integridad "un solo rol activo por cuenta" (2.1).

### 4.8 Configurar contraseña propia

1. Un Usuario autenticado (por cualquier método) que tiene password_hash = NULL puede configurar una contraseña propia.
2. A partir de ese momento, puede autenticarse tanto con Google como con email + contraseña — ambos métodos coexisten, no se reemplazan.
3. Un Usuario que ya tiene contraseña configurada puede cambiarla, pero nunca puede quedar con password_hash = NULL y google_id = NULL simultáneamente (regla 2.1).

### 4.9 Auto-registro de tutor

1. El tutor abre la aplicación móvil y se registra con nombre, contacto y el consentimiento de uso de datos, autenticándose con contraseña o vinculando directamente su cuenta de Google (4.7) — ambos caminos completan la misma alta. No necesita estar registrado previamente por ninguna clínica ni tener un usuario autenticado: es la única alta de cuenta abierta del sistema.
2. Se valida el consentimiento_datos = true (Ley 25.326). Sin consentimiento explícito el registro se rechaza — no se crea la ficha ni la cuenta.
3. Si se registra con contraseña, se valida la política de contraseña (regla 2.1). En ambos casos (contraseña o Google) se valida que el email no esté en uso por ninguna cuenta vigente.
4. Se crean la ficha de Tutor y su Usuario **en una única transacción**: una ficha sin cuenta, o una cuenta sin ficha, dejaría al tutor sin poder entrar y sin forma de recuperar el registro a medias.
5. La ficha se crea con nombre, contacto = el email del registro y el consentimiento fechado. Los campos de documento y dirección quedan vacíos hasta que una clínica complete la ficha. Esa completitud la hace un veterinario editando la ficha (alcance en 3.2), o el propio tutor sobre la suya.
6. La cuenta queda activa y sin clínica de pertenencia: el tutor entra de inmediato y ve su sección vacía hasta que una clínica le vincule Pacientes.
7. Si el alta fue con contraseña, se emite un **token de confirmación del correo** y se manda a esa dirección (4.9.1). El alta con Google no lo emite: el email lo aporta el proveedor ya verificado (4.7), así que esa cuenta nace con el correo confirmado. El envío queda **fuera de la transacción del alta**: si el correo no sale, la cuenta se creó igual y el tutor puede pedir otro.

> Esta regla reemplaza al criterio anterior, según el cual el tutor solo podía crear su cuenta si una clínica ya había registrado su ficha. Se priorizó que el tutor pueda empezar a usar la aplicación sin depender de una clínica. La contrapartida es la deduplicación: si una clínica carga una ficha de Tutor para alguien que ya se auto-registró (o viceversa), quedan dos fichas para la misma persona. El criterio de fusión queda pendiente de definición — ver Modelo de Datos, 4.1.
>
> **La cuenta queda operativa sin confirmar el correo, y ahora eso es una decisión tomada.** El envío existe (4.9.1) y la confirmación **no es condición de nada**: el tutor entra, da de alta su primera mascota (4.17) y le carga antecedentes sin esperar el correo. Exigirla cortaría en seco el único momento en que el tutor está dispuesto a cargar todo de una vez, y compraría poco: una cuenta recién registrada no alcanza ningún dato que no haya cargado ella misma, así que un correo falso no expone a nadie más que a quien lo escribió.
>
> Lo que la confirmación sí hace es **habilitar el canal de salida**. Es lo que distingue una dirección tipeada mal de una real, y de eso depende que la recuperación de contraseña (4.4.1) no sea un callejón sin salida: un tutor que se registró con un correo con un error de tipeo y olvidó su contraseña no tiene a nadie que se la restablezca. Por eso se manda, y por eso el estado se guarda.

> Este es el único punto del sistema donde el paso 3 de 4.7 puede crear un Usuario nuevo. Para cualquier otro tipo_usuario, Google solo vincula cuentas ya existentes (2.5, 4.7).

### 4.9.1 Confirmación del correo

1. Al registrarse con contraseña (4.9) se emite un token de un solo uso, se guarda **hasheado** y se manda por correo dentro de un enlace. Pedir uno nuevo invalida los anteriores, mismo criterio que la recuperación (4.4.1).
2. El canje marca la fecha de confirmación en la cuenta y consume el token. **No emite sesión, no cambia permisos y no habilita nada**: lo único que cambia es que el sistema ahora sabe que esa dirección existe y que su titular la lee.
3. Si la cuenta ya estaba confirmada, el canje responde igual que si hubiera confirmado. El segundo clic sobre el mismo enlace es un accidente corriente —un cliente de correo que precarga el link, alguien que vuelve atrás en el navegador— y contestarle un error a quien ya hizo bien las cosas no protege de nada.
4. El **reenvío exige sesión**: lo pide la propia cuenta desde adentro de la aplicación. No hay un endpoint público que acepte un correo cualquiera, porque sería exactamente el padrón de direcciones registradas que 4.4.1 se cuida de no revelar — y acá, a diferencia de la recuperación, quien necesita el reenvío ya está adentro.
5. Un token vencido, inexistente o ya usado de otra cuenta se rechaza con **el mismo error genérico**, como la activación y la recuperación.

> **Vive mucho más que los otros dos tokens** (48 horas por defecto, contra una hora la recuperación). La diferencia no es un descuido: la activación y la recuperación **abren una cuenta** —quien las tenga define la contraseña—, y este no abre nada. Lo peor que puede hacer quien intercepte el enlace es marcar como confirmada una dirección que ya era de otro. Estirarlo es lo que evita que un correo leído al día siguiente llegue muerto.
>
> **Solo alcanza a las cuentas con contraseña.** La cuenta creada con Google nace confirmada porque el proveedor ya verificó el email (4.7), y a la de veterinario o clínica_admin la confirma su propia activación: el token de activación viajó a esa dirección y alguien lo canjeó, que es la misma prueba (4.12, 4.16).

### 4.10 Alta de una clínica y de su cuenta administrativa

1. El administrador de la plataforma da de alta la Clínica y su cuenta clínica_admin en una sola operación, por fuera de la API HTTP (una herramienta de línea de comandos que invoca la misma capa de negocio).
2. La cuenta se crea con clínica_id y clínica_de_pertenencia_id apuntando a la clínica recién creada, y **sin contraseña**: la define la propia clínica al estrenarla (proceso 4.16). La clínica nace con **Franjas de atención por defecto de lunes a viernes** y una duración de turno por defecto. Nace con horario y no vacía porque una clínica sin ninguna franja no admite ninguna cita, y la primera pantalla que vería el veterinario sería un calendario que rechaza todo.

> **El horario por defecto es un punto de partida, no una configuración fija.** La clínica lo edita entero desde la web (Alcance de Plataformas, 3.2.4), con las mismas reglas que cualquier otro cambio de grilla: puede abrir el sábado, cerrar el miércoles, partir el mediodía o cambiar las horas de cualquier día. Lo que el alta elige es solo qué ve la clínica la primera vez que entra, y se eligió lunes a viernes porque es lo que más veterinarias hacen — no porque el sistema suponga que no se atiende el fin de semana.
3. La herramienta emite un **token de activación de un solo uso** y lo imprime una única vez. El administrador se lo entrega a la clínica por el canal que haya acordado con ella; el sistema no lo envía por email ni lo vuelve a mostrar.
4. A partir de ahí, ese clínica_admin es responsable de dar de alta las cuentas de los veterinarios de su clínica (regla 2.5).

> El administrador de la plataforma no es un tipo_usuario del sistema: no tiene cuenta, no se autentica y no aparece en el motor de permisos. Es un operador con acceso al despliegue. Si en algún momento esa responsabilidad se delega o necesita ser auditada, habrá que decidir explícitamente la creación de un cuarto rol.
>
> La contraseña inicial la fijaba el administrador en la primera versión de este proceso. Se cambió por el token porque una contraseña que el operador conoce, escribe en algún lado y transmite por un canal cualquiera es una contraseña compartida desde el minuto cero, y la cuenta que protege administra el plantel entero de una clínica. Con el token, nadie fuera de la clínica llega a conocer la credencial con la que se entra.

### 4.12 Alta de un veterinario

1. El clínica_admin da de alta la ficha de Veterinario y su cuenta de acceso **en una única transacción**: una ficha sin cuenta dejaría a alguien en el plantel sin poder entrar al sistema, y una cuenta sin ficha violaría la regla de integridad 2.1.
2. La clínica no se envía en el pedido: es siempre la del administrador que da el alta (regla 2.5). Tanto la ficha como la cuenta quedan en esa clínica.
3. Se valida el par (tipo_documento, número_documento) único entre fichas vigentes **de Veterinario** (regla 2.1) —un documento que ya figure en una ficha de Tutor no obsta: son padrones distintos—, la matrícula libre si viene cargada y el email libre en todo el sistema.
4. La matrícula es opcional. Sin ella la ficha existe en modo restringido: no habilita a crear ni editar Eventos clínicos ni Medicación (regla 2.2). Cargada, tiene que estar libre: la habilitación es de una persona y no de dos (regla 2.1).
5. **El clínica_admin no elige la contraseña.** La cuenta nace con `activación_pendiente = true` y sin ningún método de autenticación, se emite un token de activación y se manda por correo a la dirección del veterinario. Él define su contraseña canjeándolo, con el mismo proceso con que un clínica_admin estrena la suya (4.16).
6. El envío del correo queda **fuera de la transacción del alta**, igual que en el auto-registro: si el correo no sale, la ficha y la cuenta se crearon igual y quedan esperando una reemisión del token. Una ficha a medias sería peor que un correo que hay que volver a mandar.

> **Que el administrador no elija la contraseña es el punto del cambio, no un efecto lateral.** Cuando la elegía él, la credencial de un profesional pasaba por un tercero y quedaba escrita en algún lado —un chat, un papel, la cabeza de quien la tipeó— antes de llegar a su dueño; y como el veterinario podía no cambiarla nunca, cualquier acto médico firmado con esa cuenta era, en rigor, atribuible a dos personas. Con el token, **el único que llega a saber la contraseña es su titular**. La contrapartida es que el alta ya no deja al veterinario adentro en el acto: hay un correo de por medio.

### 4.13 Baja de un veterinario

1. El clínica_admin da de baja lógica la ficha. En la misma transacción se desactiva su cuenta de acceso (regla 2.4).
2. El token de acceso vigente sigue valiendo hasta expirar — minutos, no días. La renovación se rechaza porque valida `Usuario.activo` (Arquitectura, 4.2), y un nuevo login también. Es la ventana de revocación ya asumida por el esquema (Arquitectura, 4.3).
3. La baja no cascadea sobre lo que ese veterinario escribió: los Eventos clínicos y las Medicaciones conservan su autoría, que es justamente el registro que la historia clínica necesita preservar.

### 4.11 Autenticación e inicio de sesión

1. El cliente envía email + contraseña, o un idtoken de Google (4.7), junto con el canal desde el que se autentica.
2. Se acredita la identidad. Un email sin cuenta, una contraseña incorrecta, un email mal formado o una cuenta que solo tiene Google configurado devuelven **el mismo rechazo**: distinguirlos le informaría a un atacante qué direcciones están registradas.
3. Se valida `Usuario.activo`. Este caso sí se distingue del anterior: quien llega acá ya acreditó su identidad, así que decirle que su cuenta está inhabilitada no revela nada que no sepa, y evita que siga probando contraseñas en vez de hablar con su clínica.
4. Se valida el bloqueo de canal (regla 2.3). Recién si los tres pasos anteriores pasan se emite el par de tokens (Arquitectura, sección 4).
5. Se registra el `ultimo_acceso` del Usuario. Que ese registro falle no niega el ingreso: es un dato informativo, no parte de la acreditación.

> El registro de tutor (4.9) no emite tokens: crea la cuenta y el cliente autentica a continuación por este proceso. Son dos pasos deliberadamente separados — el alta es pública y la emisión de tokens es donde se aplica el bloqueo de canal.

### 4.14 Subida y descarga de un Adjunto

1. Subida: el cliente envía el archivo al backend (multipart), que valida rol, alcance, tipo declarado contra el MIME real y tamaño **antes** de tocar el bucket.
2. El backend sube el objeto al bucket con una clave que él genera, y recién después inserta la fila con sus metadatos y su registro de Auditoría, en la misma transacción.
3. El orden importa: si la escritura en base falla, el backend intenta borrar el objeto recién subido y, si no puede, lo deja registrado en el log. La consecuencia asumida es que puede quedar un objeto huérfano en el bucket, nunca una fila que apunte a un archivo inexistente — el error visible para el usuario tiene que ser "no se subió", no "se subió y no se puede abrir".
4. Descarga: el bucket es privado. Cada lectura devuelve una URL prefirmada de vida corta, generada en el momento tras evaluar los permisos. La URL no se persiste ni se reutiliza.
5. El cliente no elige la clave del objeto ni la ve: la genera el backend. Una clave elegida por el cliente sería una vía para escribir sobre el archivo de otra mascota.
6. **Foto de perfil.** Un adjunto se puede marcar como foto de la mascota (`es_foto_perfil`, Modelo de Datos, 4.8), al subirlo o después, y solo si es de tipo **foto**. Marcar uno **desmarca al que estuviera marcado, en la misma operación**: la mascota tiene como máximo una foto de perfil vigente. La desmarcada no se da de baja — queda como un adjunto más, que es lo que siempre fue. Dar de baja el adjunto marcado deja a la mascota sin foto de perfil: no se elige una sucesora sola.
7. Quién puede marcarla es quien puede subir adjuntos de esa mascota (regla 2.4 y motor de permisos): no es una operación aparte con su propio alcance.
8. **Renombrar.** `nombre_archivo` es el nombre con el que el archivo se ve y se baja, y se puede elegir **al subirlo** —junto con el archivo— o **después**, sobre uno ya cargado. Lo demás del adjunto sigue sin editarse: renombrar no toca los bytes, ni la clave, ni el tipo, ni el peso.
9. **Renombra quien lo subió**, con el mismo criterio que retirar (regla 2.4): el archivo es de quien lo cargó, y el nombre es cómo lo van a leer todos los demás. Marcar la foto de perfil sigue siendo la excepción — ahí lo que se decide es qué muestra la ficha, no qué dice el archivo de otro.
10. El nombre se valida: no puede quedar vacío, no lleva rutas ni saltos de línea, tiene un techo de largo, y **conserva la extensión del archivo real** — si el nombre elegido no la trae, el backend se la agrega (Modelo de Datos, 4.8). Un nombre sin extensión baja como un archivo que el sistema no sabe abrir.

> **Desmarcar la anterior lo hace el backend, no el cliente.** Resolverlo en dos pedidos —desmarcar una, marcar la otra— abre la ventana en la que la mascota tiene dos fotos de perfil o ninguna, y obliga a cada cliente a recordar cuál era la vigente para poder apagarla. Es una sola operación y el estado intermedio no existe.

### 4.15 Recordatorios de una Cita al tutor

El calendario existe para que la mascota llegue a su control; el recordatorio es lo que convierte una fecha anotada en una consulta hecha. Por eso los dos únicos avisos definidos son los que reducen el ausentismo, y no los de confirmación ni los de cita vencida: un aviso posterior al hecho no cambia nada, y uno por cada escritura enseña al tutor a ignorarlos.

1. **Canal**: notificación push al móvil del tutor. La cuenta registra sus dispositivos al iniciar sesión **desde la app**, que es el único lugar desde el que el tutor entra (regla 2.3); una cuenta sin dispositivo registrado —la del tutor que se dio de alta y todavía no abrió la aplicación en su teléfono— no recibe avisos. El email queda descartado como segundo canal: no se agrega un medio nuevo para el mismo destinatario mientras el push cubra el caso.
2. **Momentos**: dos recordatorios por Cita — uno el día anterior a `fecha_programada` y otro el mismo día. El del día anterior se sigue enviando a una hora fija (configurable por entorno): a las 19:00 del día previo no importa si la cita es a las 09:00 o a las 17:00, el tutor necesita el mismo aviso. El del mismo día pasó a enviarse **a una distancia fija de la cita** (`RECORDATORIO_HORAS_ANTES`, por defecto 2 h), que es lo que la hora en `fecha_programada` hizo posible: avisar a las 08:00 de un turno de las 17:00 es un aviso que se olvida antes de servir. Los dos se miden distinto al encolar, y no es una inconsistencia. El del **día anterior se tolera tarde**: se encola mientras su día siga siendo hoy o el futuro, porque una cita que se agenda hoy para mañana genera su aviso a una hora que en general ya pasó, y mandarlo a la tarde sigue siendo el aviso del día anterior — descartarlo dejaría a casi toda cita con un solo recordatorio. El del **mismo día no se tolera tarde**: está atado a la hora del turno, y mandarlo después de que el turno pasó no avisa nada. Una cita agendada para dentro de una hora se queda sin él.
3. **Encolado**: un proceso periódico busca las Citas pendientes con `notificar_tutor = true` que entran en la ventana de recordatorio y encola las notificaciones que falten. El encolado es un barrido y no un efecto de la escritura de la Cita: así una cita reagendada, una notificación perdida o un backend que estuvo caído se resuelven solos en la corrida siguiente, sin depender de que nadie vuelva a tocar la cita. Encolar es idempotente: existe una sola notificación de cada tipo por Cita, sostenido por un índice único. Reagendar una cita no duplica su recordatorio.
4. **Envío**: el mismo proceso despacha las notificaciones cuya hora ya llegó, reintentando las que fallan hasta un techo de intentos. Una notificación que agota los intentos queda registrada como fallida y no se reintenta más: un aviso de una cita que ya pasó no sirve.
5. **Teléfono rechazado**: cuando el proveedor informa que un aparato dejó de existir (la app se desinstaló, el token caducó), ese Dispositivo se da de baja en vez de reintentarse. Reintentar contra un teléfono que ya no existe gasta los intentos de la notificación sin ninguna chance de éxito. Un aviso que no llegó a ningún aparato no se da por enviado.
6. **Destinatario**: la cuenta del tutor de la mascota, resuelta al encolar. Una ficha de tutor sin cuenta de Usuario no recibe avisos: no hay a dónde mandarlos. Los **dispositivos** de esa cuenta, en cambio, se resuelven recién al despachar, porque entre encolar y enviar pasan horas y el tutor puede haber registrado su primer teléfono en el medio.
7. Las notificaciones **no se auditan**: la Auditoría registra cambios sobre entidades clínicas (Modelo de Datos, 4.10), y un aviso no modifica el historial. Su rastro es la propia tabla, que guarda qué se envió, cuándo y con qué resultado.

### 4.16 Activación de una cuenta

1. Quien la estrena abre el enlace de activación con el token que le llegó: al clínica_admin se lo entrega el administrador de la plataforma en mano (4.10), al veterinario le llega por correo cuando su clínica lo da de alta (4.12). El proceso es el mismo de los dos lados — lo único que cambia es por dónde viajó el token.
2. El backend valida el token: que exista, que no esté usado, que no esté vencido y que su cuenta siga activa. Cualquiera de esas condiciones que falle devuelve **el mismo error genérico**: distinguir "vencido" de "inexistente" le dice a quien esté probando tokens al azar cuál acertó a medias.
3. La persona define su contraseña, que se valida contra la política de la regla 2.1 (mínimo 8 caracteres, una minúscula, una mayúscula y un dígito).
4. En una sola transacción se escribe el `password_hash` de la cuenta y se marca el token como usado. Es de un solo uso: presentarlo de nuevo falla, aunque quien lo presente sea la misma persona.
5. El canje **no autentica**: devuelve el resultado de la operación, no un par de tokens. Para entrar hay que iniciar sesión con la contraseña recién definida, que pasa por el bloqueo de canal como cualquier otro login (regla 2.3).

> **Es un solo proceso para dos orígenes, y eso es deliberado.** El problema es idéntico —una cuenta que existe y que nadie estrenó todavía— y duplicarlo por tipo de usuario habría dejado dos caminos hacia una contraseña, cada uno con su propia forma de equivocarse.
>
> Es el segundo endpoint público del sistema, junto con el auto-registro de tutor (4.9), y el único que escribe sobre una cuenta ya existente sin autenticación previa. Por eso el token es la credencial completa —no un dato que acompaña a un email o a un identificador de cuenta—: pedir además el email haría que la protección real dependa de un dato que circula en cualquier lado, y no agrega nada que el token no dé.
>
> El canje no emite sesión a propósito. Emitirla ahorraría un paso, pero convertiría este endpoint en una vía de autenticación paralela al login, con su propio bloqueo de canal que mantener sincronizado. Un formulario de login extra es más barato que dos caminos hacia una sesión.
>
> **Pendiente**: qué pasa si el token se vence o se pierde antes de usarse. Hoy la salida es que el administrador de la plataforma emita otro por línea de comandos, lo cual funciona pero no está definido como proceso ni deja rastro de cuántas veces se reemitió.

### 4.17 Alta de una mascota por el tutor

1. El tutor autenticado carga nombre, especie, raza, fecha de nacimiento, sexo y peso, y **opcionalmente una foto de la mascota**. **No elige clínica**: el formulario no la ofrece, porque en este camino no hay ninguna.
2. Se valida el consentimiento de datos del tutor. Ya está otorgado desde su registro (4.9), pero se verifica igual: es la misma condición que exige el alta por la clínica (regla 2.2), y el alta no puede depender de por dónde entró.
3. Se crea el Paciente con `tutor_id` = el tutor del token, que queda como dueño. Nunca se toma del cuerpo del pedido: sería dar de alta una mascota a nombre de otro.
4. No se crea ningún vínculo con clínica. La mascota existe, es legible por su dueño y no la ve nadie más.
5. Se registra la creación en Auditoría.
6. Si el tutor eligió una foto, se sube con el Paciente ya creado por el proceso de Adjuntos (4.14), marcada como foto de perfil. **No es parte de la transacción del alta**, por el mismo motivo que los antecedentes: cuelga de un `paciente_id` que antes no existe. Si la subida falla, la mascota queda dada de alta sin foto y la aplicación lo dice — el alta no se revierte por eso.
7. Con el Paciente ya creado, la aplicación le ofrece **cargar los antecedentes** que la mascota trae de antes (4.23). Es un paso ofrecido y no un paso del alta: se puede saltear entero, y la mascota queda dada de alta igual.

> El número de chip no se carga acá: lo pone el veterinario (3.2).

> **El paso 7 no es parte de esta transacción y no puede serlo.** Los antecedentes cuelgan de un `paciente_id` que todavía no existe mientras el alta no terminó, así que primero se crea la mascota y después se cargan — y si el tutor abandona la aplicación en el medio, lo que queda es una mascota sin antecedentes, que es un estado válido, y no un alta a medias.
>
> Es también el motivo por el que **este camino exige conexión de punta a punta** (Sincronización sin Conexión, 5).
>
> **El alta no exige tener el correo confirmado, y se decidió que siga así.** Ahora el envío existe (4.9.1) y la pregunta ya no está abierta: la mascota se crea con la cuenta recién registrada en 4.9, sin esperar el correo. Se eligió sostenerlo a propósito, porque el alta de la primera mascota es el único momento en que el tutor está dispuesto a cargar todo de una vez, y porque lo que escribe acá es historial **de su propia mascota**, que nadie más lee hasta que él comparta (4.18): una dirección de correo sin confirmar no pone ese dato al alcance de nadie.
>
> Una mascota sin clínica **no admite citas** (regla 2.2), y es lo único que le falta hasta que el tutor comparta. Todo lo demás —la ficha, el peso, los adjuntos que él suba— funciona desde el primer minuto. Es deliberado que la aplicación sirva para algo antes de que exista una veterinaria de por medio: esa era exactamente la fricción que el modelo anterior imponía.

### 4.18 Compartir una mascota con una clínica

1. El **dueño** busca la clínica en el directorio, por nombre. El directorio devuelve nombre, dirección y contacto — lo suficiente para no confundir dos sucursales, y nada más.
2. Se valida que el Paciente esté vigente y que esa clínica no tenga ya un vínculo vigente con él.
3. Se crea el vínculo (`origen = compartido_por_el_tutor`, con el usuario que lo otorgó) y se audita.
4. A partir de ese momento, **todo el plantel vigente de esa clínica** alcanza la mascota: lee su historial completo, escribe eventos y medicación, y agenda citas en su propia agenda.

**Revocarlo** es el camino inverso, y lo puede hacer el dueño sobre cualquier clínica o un veterinario sobre la suya. Se rechaza mientras esa clínica tenga citas pendientes de esa mascota (regla 2.4). Lo que la clínica escribió se queda con su autoría; lo que pierde es el acceso de ahí en adelante.

> Compartir da acceso **al historial completo, incluido lo que escribieron otras clínicas**, y no solo a lo que esa clínica vaya a escribir. Es el punto entero de la funcionalidad —un profesional que no ve las vacunas que puso otro no puede atender bien— y conviene decirlo explícito en la pantalla antes de confirmar, no después.

### 4.19 Invitar a un co-tutor

1. El **dueño** ingresa el correo de la persona y elige el nivel: edición o lectura.
2. Se valida que el Paciente esté vigente, que no se esté invitando al propio dueño y que esa persona no tenga ya un acceso vigente sobre esa mascota — cambiar de nivel es editar el acceso que existe, no emitir otra invitación.
3. Se emite un código de un solo uso, se guarda **hasheado** y se envía por correo. Si había otra invitación pendiente para ese mismo correo y esa misma mascota, queda anulada: mismo criterio que la recuperación de contraseña (4.4.1).
4. Quien recibe el enlace ve, **sin autenticarse**, qué mascota es, con qué nivel y quién lo invita. Nada del historial, y nunca si ese correo tiene o no una cuenta en Wayka.
5. Para canjearlo tiene que estar autenticado como tutor. Si no tiene cuenta, se registra primero (4.9) y vuelve. Quien **ya tiene cuenta** no depende del correo: la invitación le aparece en la aplicación, resuelta por su dirección, y la acepta desde ahí sin el enlace. El token es la credencial de quien llega sin sesión, y por eso no se devuelve en ningún listado.
6. El canje valida que el código esté vigente, sin usar y sin anular, **que el correo de la cuenta sea el invitado**, que el Paciente siga vigente y que el consentimiento de datos esté otorgado. En una sola transacción marca el código como usado, crea el acceso con su `consentimiento_at` y audita.

> El canje exige que el correo coincida porque, si no, un enlace reenviado lo estrena cualquiera y la invitación deja de estar dirigida a alguien.
>
> Los errores del canje son **indistinguibles entre sí**: inválido, vencido y ya usado devuelven lo mismo, con el mismo criterio que la recuperación de contraseña.
>
> **Rechazar** es del invitado y anula la invitación sin dar acceso; el enlace del correo deja de servir. Se distingue de **anular**, que es del dueño arrepintiéndose de haberla mandado: son la misma escritura con dos motivos distintos, y quién puede hacerla no es el mismo.

> **Revocar un acceso** lo hace el dueño, o el propio co-tutor sobre el suyo. El efecto en el servidor es inmediato; en un teléfono sin señal, no — ver Sincronización sin Conexión, 8, que explica hasta cuándo puede seguir leyéndose una copia ya descargada y qué se hace al respecto. La interfaz se lo dice al dueño en el momento de revocar, en vez de prometerle algo que el sistema no puede cumplir.

### 4.20 Registro de un evento de telemetría

Dos caminos, según quién lo emita (Telemetría de Producto, 2).

**Lo que emite el backend** — todo hecho que el servidor ya ve: se creó un evento clínico, se canjeó una invitación, se despachó un push. Lo escribe la capa de negocio en el mismo request o job que atendió la acción, y no depende de que ningún cliente lo mande. Es lo que evita tener dos cifras del mismo hecho y una discusión sobre cuál vale.

**Lo que emite el cliente** — solo lo que el backend no puede ver: qué pantalla se abrió, cuánto tardó un formulario, si se abandonó, si la sesión salió de la copia local, si el usuario llegó desde un push.

1. El cliente acumula eventos y los despacha **en lote** a la ruta de ingesta, autenticado. No hay un request por evento.
2. El backend valida la forma del lote y su techo de tamaño. Un lote mal formado se rechaza; un lote válido se acepta entero.
3. Evento por evento: descarta los de nombre desconocido, poda las propiedades fuera de la lista permitida (2.7), completa `usuario_id`, `rol` y `clínica_id` desde el token, sella `registrado_at` y marca `reloj_sospechoso` si el `ocurrido_at` no es creíble contra él.
4. Persiste lo que quedó. La respuesta es exitosa aunque adentro se haya descartado todo.

> **En móvil la cola vive en el dispositivo y sube con el ciclo de sincronización** (Sincronización sin Conexión, 4.1), por su propia ruta. Con dos diferencias respecto de las mutaciones: la cola **tiene techo y descarta lo más viejo** —una mutación no se descarta nunca, un evento sí— y nunca demora ni bloquea la subida de mutaciones. El dato clínico va primero.

> **El plazo de retención lo aplica un job**: a los 13 meses borra las filas de detalle y conserva agregados sin `usuario_id`. Es un borrado **físico**, la única excepción del proyecto a la regla de 2.4, y está argumentada en Telemetría de Producto, 8.

### 4.21 Registro de una consulta atendida

El hecho asistencial (Modelo de Datos, 4.16), separado de lo que se escriba sobre él. Dos caminos.

**Asentada por el veterinario** — el camino que importa:

1. Desde la agenda del día, sobre una cita: un toque. Desde la ficha del paciente, para la atención que nadie agendó: un toque y elegir el origen (espontánea o urgencia).
2. Se validan paciente vigente, clínica vinculada, profesional de esa clínica, momento no futuro y cita no atendida (regla 2.2). `clínica_id` sale del actor y `veterinario_id` es por defecto quien asienta, cambiable a otro del plantel.
3. Se persiste con `asentada_automáticamente = false` y, si trae `cita_id`, se transiciona esa Cita a `cumplido` en la misma transacción (4.4).
4. Se registra en Auditoría, como cualquier escritura asistencial.

**Deducida por el sistema** — cuando se carga un Evento clínico que declara la cita que cumple y esa cita no tiene consulta asentada, se asienta una en la misma transacción, con `origen = agendada`, `veterinario_id` = el autor del evento, `fecha_hora` = la fecha del evento y `asentada_automáticamente = true`. El evento queda vinculado a ella.

> **Se deduce sola porque perder el hecho sería absurdo**: un evento clínico que dice qué cita cumplió es prueba de que la atención ocurrió. Lo que no hace es agregar información a la métrica —esas filas entraron por el mismo camino que el numerador—, y por eso quedan marcadas y la cobertura se lee sobre las asentadas por una persona (Telemetría de Producto, 9).
>
> **El evento ya no guarda su propia FK a la Cita** (Modelo de Datos, 4.5): declarar la cita al cargar es un atajo de entrada que resuelve la consulta, y lo que queda persistido es `consulta_id`. Así de una misma atención pueden colgar varios eventos sin que haya que elegir cuál se queda con la cita.
>
> **No se deduce del evento sin cita.** Ahí no hay nada que confirme que la atención fue hoy y no una carga histórica, y asentarla igual haría que toda atención documentada tuviera su asiento: la cobertura daría 1 siempre y no mediría nada. Un evento sin consulta queda con `consulta_id` NULL, y cuántos hay es la lectura inversa — cuánto se está escribiendo sin asentar.
>
> **Corrección y baja**: se corrigen la fecha, el profesional y el origen; se da de baja lógica el asiento cargado por error. Dar de baja la consulta de una cita **no revierte** la Cita a pendiente si ya hay un Evento clínico colgado de ella: lo que se atendió y se documentó ocurrió, más allá de que el asiento estuviera mal. Si no hay ningún evento, la Cita vuelve a `pendiente` o a `vencido` según su fecha.
>
> **La baja del Paciente no arrastra las consultas**, con el mismo criterio que el resto del historial (4.5): quedan consultables desde la ficha y siguen contando para las métricas del período en que ocurrieron. Lo que se rechaza es asentar consultas nuevas.

### 4.22 Carga de una ausencia del profesional

Que un veterinario no va a estar disponible (Modelo de Datos, 4.19).

1. El clínica_admin carga el rango —`desde` y `hasta`, con hora— sobre alguien de su propio plantel. Se validan rango coherente, veterinario vigente de su clínica y no solapamiento con otra ausencia vigente (regla 2.2).
2. Antes de guardar, el sistema resuelve **qué Citas pendientes asignadas a ese veterinario caen dentro del rango**, y se lo dice a quien está cargando: cuántas son y cuáles.
3. La ausencia se guarda, y en la **misma transacción** esas citas quedan con `veterinario_id` en NULL. No se cancelan, no se mueven de hora y no cambian de estado: siguen pendientes, sin profesional.
4. Cada desasignación queda en la Auditoría de su Cita, con `usuario_tipo = clínica_admin`. La ausencia en sí no se audita: `registrada_por_usuario_id` ya dice quién la cargó.

> **Se guarda siempre.** No se rechaza por las citas que toca, a diferencia de achicar el horario (regla 2.2). La asimetría es deliberada y las dos cosas que se pierden no son la misma: achicar el horario deja la cita **sin turno donde existir**, y eso hay que resolverlo antes; una ausencia deja la cita **sin quién la atienda**, que es un estado que la agenda ya sabe representar. Además, el horario se planifica y una ausencia muchas veces se entera: impedir registrar que alguien no vino no hace que haya venido, y el día que hay que cargarla rápido es justo el día en que nadie tiene tiempo de reasignar seis turnos primero.

> **Desasignar no es perder trabajo.** La cita cae en la lista de "sin asignar" (Alcance de Plataformas, 3.6), que es exactamente la cola de lo que hay que repartir, y el panel del clínica_admin la cuenta. Sin la desasignación, la cita seguiría figurando a nombre de alguien que no va a estar, y nadie se enteraría hasta el día del turno.

> **No mira hacia atrás.** No toca Consultas atendidas ya asentadas ni Citas cumplidas: si el sistema dice que esa persona atendió el martes, una ausencia cargada después no lo desmiente. Ahí hay un error de carga en uno de los dos y se corrige el que esté mal.

> **La baja de la ausencia no reasigna nada.** Las citas desasignadas siguen sin profesional y se reparten como cualquier otra. Devolverlas a quien las tenía sería adivinar que nadie las movió mientras tanto, y pisaría una asignación que alguien pudo haber hecho a propósito.

### 4.23 Carga de antecedentes por el tutor

La mayoría de las mascotas que se dan de alta ya tienen historia: vacunas puestas, alergias conocidas, una medicación en curso. Este proceso es cómo eso entra a Wayka, y **no está atado al alta**: se llega desde el onboarding la primera vez (4.17) y desde la ficha de la mascota siempre.

1. El tutor —dueño o co-tutor con edición— elige una mascota que alcanza y vigente (regla 2.2), y elige **qué tipo de antecedente** carga: vacuna, alergia, medicación o "otra cosa que pasó". La interfaz llega con **vacuna destacada**, que es el antecedente que más tutores tienen a mano; elegir otro es un toque y no un camino distinto.
2. Completa el formulario del tipo, que pide menos que el del veterinario: obligatorio es lo que identifica al antecedente —el nombre de la vacuna, el alérgeno, la droga— y la fecha; el resto es opcional (regla 2.2 y Modelo de Datos, 4.5).
3. La fecha se declara **con su precisión**: día, mes o solo año. La interfaz no obliga a bajar de nivel — "en 2023" es una respuesta completa — y **arranca en año**, que es la precisión del caso normal cuando el dato sale de una libreta vieja. Precisar el mes o el día es la excepción y la decide el tutor. Es la única diferencia con el formulario del veterinario, que sí arranca en día porque escribe lo que está pasando hoy.
4. Se persiste según el tipo:
   - **vacuna**, **alergia** y "otra cosa que pasó" son un **Evento clínico** con `usuario_id` = el usuario autenticado, `cargado_por = tutor` y `consulta_id` en NULL, que es lo que el modelo ya llamaba una carga histórica (Modelo de Datos, 4.5). "Otra cosa que pasó" se guarda con el `tipo` que el tutor elija entre consulta, cirugía, control y urgencia, y su contenido es la descripción libre.
   - **medicación en curso** es una **Medicación** con `fecha_fin` NULL y `cargado_por = tutor` (4.3). Si el paciente ya tiene esa droga activa, se rechaza por la regla 2.2 y el rechazo dice cuál es la que ya está y quién la cargó.
5. Se registra la creación en Auditoría, con el usuario que la originó, igual que cualquier otra escritura del historial.
6. El tutor puede cargar varios, uno atrás del otro. Cada uno es su propia operación: que el tercero se rechace no descarta los dos anteriores.
7. **Documentos de la libreta sanitaria**: la subida de fotos o PDF es el proceso de Adjuntos que ya existe (4.14), a nivel `paciente_id` y sin `evento_id`. No hace falta asociar cada foto a un antecedente puntual — pedirlo convertiría fotografiar una libreta en una tarea de clasificación.
8. Un antecedente ya cargado lo edita y lo da de baja el tutor, en cualquier momento, **siempre que sea de origen `tutor`** (regla 3.2). El veterinario también puede corregirlo.

> **No hay ningún paso de verificación, y es deliberado** (Modelo de Datos, sección 1). El antecedente queda cargado y visible desde que se guarda, marcado como declarado por el tutor. Nadie lo aprueba.
>
> **La vista de urgencia lo incluye** y lo distingue de lo que escribió un profesional de forma imposible de pasar por alto (Modelo de Datos, 4.5). Es la consecuencia más delicada de todo este proceso y por eso la marca es parte del contrato y no una decisión de cada pantalla.
>
> **Qué hace el veterinario con esto.** Lo lee como contexto al atender: no tiene que aprobarlo, copiarlo ni convertirlo. Si confirma clínicamente una alergia declarada, lo que carga es su propio registro con su propio origen — y la declaración del tutor se queda donde está, porque también es cierta.

## 5. Fuera de alcance de este documento

- **Deduplicación de fichas de Tutor** — cómo se fusionan la ficha de un tutor auto-registrado y la que una clínica pudo haber creado para la misma persona (ver 4.9).
- **Auditoría de la entidad Usuario** — el alta, la edición, la suspensión y la baja de cuentas no generan registro de Auditoría hoy: la sección 4.10 del Modelo de Datos la limita a Evento clínico, Medicación, Cita y Paciente. Queda por decidir si las operaciones sobre cuentas deben auditarse.
- **Verificación de email y bloqueo por intentos fallidos** — no definidos. La recuperación de contraseña sí está definida (proceso 4.4.1).
- **Vista de paciente derivado en urgencia** — priorización de datos mostrados, tiempos de carga aceptables, y su relación con la Fase 2 (matching geolocalizado).

Ambas quedan pendientes de definición en una etapa posterior del proyecto.
