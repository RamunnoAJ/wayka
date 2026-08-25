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
| Documento único | Tutor, Veterinario | número_documento no puede repetirse entre personas del mismo tipo_documento. En Tutor la validación aplica solo a las fichas que ya tienen documento cargado: el auto-registro (4.9) no lo exige. Se valida tanto en el alta como en la edición de la ficha, y el tipo y el número se cargan o se limpian juntos: una ficha nunca tiene uno sin el otro. |
| Email único | Usuario | email no puede repetirse en todo el sistema, independientemente del tipo_usuario. |
| Un solo rol activo por cuenta | Usuario | Exactamente una de tutor_id / veterinario_id / clínica_id debe estar completa, según tipo_usuario. Las otras dos deben ser NULL. |
| Al menos un método de autenticación | Usuario | password_hash y google_id no pueden ser ambos NULL — un Usuario necesita al menos una forma de autenticarse. |
| Google ID único | Usuario | google_id no puede repetirse — una cuenta de Google solo puede estar vinculada a un Usuario de Wayka. |
| Política de contraseña | Usuario | Mínimo 8 caracteres, con al menos una letra minúscula, una mayúscula y un dígito. Rige tanto el alta como el cambio de contraseña. |
| Clínica de pertenencia según rol | Usuario | clínica_de_pertenencia_id debe ser NULL en las cuentas de tutor, estar completa en las de veterinario, y coincidir con clínica_id en las de clínica_admin. |
| Rol y referencias inmutables | Usuario | tipo_usuario y las tres FK de rol no son editables después del alta: permitir cambiarlas habilitaría una escalada de privilegios sobre una cuenta ya creada. |
| Email reutilizable tras la baja | Usuario | La unicidad de email rige entre cuentas vigentes. El email de una cuenta dada de baja lógicamente puede reutilizarse en un alta posterior. |
| Matrícula obligatoria para escribir | Veterinario | Un veterinario sin matrícula cargada no puede crear ni editar Eventos clínicos ni Medicación (puede existir de alta, pero queda en modo restringido). |

### 2.2 Datos clínicos

| Regla | Aplica a | Validación |
|---|---|---|
| Fecha no futura | Evento clínico | fecha no puede ser posterior a la fecha actual (un evento clínico registra algo que ya ocurrió; lo futuro es una Cita). |
| Paciente vigente para cargar | Evento clínico | No se crean Eventos clínicos sobre un Paciente con `deleted_at`. El historial existente sigue consultable y editable (regla 4.5), pero no se le agregan episodios nuevos. |
| Campo estructurado según el tipo | Evento clínico | `campo_estructurado` se valida contra el esquema fijo del `tipo` del evento (Modelo de Datos, 4.5): obligatorio y con forma exigida para vacuna, medicación y alergia, y NULL para consulta, cirugía, control y urgencia. Un JSON con claves ajenas al esquema, con faltantes, o presente en un tipo que no lo admite, se rechaza. |
| Una medicación activa por droga | Medicación | No se permite crear una nueva Medicación con fecha_fin NULL para una droga que el paciente ya tiene activa. Debe cerrarse la anterior (fecha_fin) antes de abrir una nueva. |
| Paciente fijo a una clínica | Paciente | clínica_id no es editable una vez creado el paciente en el MVP. Reasignar de clínica está fuera de alcance hasta Fase 2. |
| Consentimiento previo a alta de paciente | Tutor, Paciente | No se puede dar de alta un Paciente si el Tutor asociado no tiene consentimiento_datos = true. |
| Fecha no pasada | Cita | fecha_programada no puede ser anterior a la fecha actual, ni al crear ni al reagendar. Es la regla espejo del Evento clínico: lo que ya ocurrió se registra como evento, lo que va a ocurrir se agenda como cita. Una cita creada en el pasado nacería vencida. |
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

### 2.3 Bloqueo de canal

| Regla | Aplica a | Validación |
|---|---|---|
| Tutor solo por móvil | Usuario (tipo_usuario = tutor) | Todo intento de autenticación desde el canal web se rechaza, independientemente de que el email y la contraseña sean correctos. |
| Clínica_admin y Veterinario, web habilitada | Usuario (tipo_usuario = clínica_admin, veterinario) | Ambos roles pueden autenticarse desde web. El Veterinario además puede autenticarse desde móvil (paridad); Clínica_admin no tiene acceso a móvil. |

> Esta regla se valida en el backend al momento de autenticar, no solo ocultando opciones en la interfaz — mismo criterio que el resto del motor de permisos (sección 3). Ver Alcance de Plataformas para la matriz completa de plataforma por rol.
>
> El canal lo declara el cliente en el pedido de autenticación (campo `canal`, ver Arquitectura 4.4). No es una barrera contra un request armado a mano: es el único lugar donde el backend puede aplicar la regla, y así queda registrado explícitamente en vez de darse por supuesto. La renovación de token no vuelve a declararlo — lo hereda de la sesión — para que refrescar no sea una vía de evadirlo.

### 2.4 Borrado

| Regla | Aplica a | Validación |
|---|---|---|
| Nunca borrado físico | Todas las entidades clínicas | Ninguna operación de la aplicación ejecuta DELETE físico sobre Evento clínico, Medicación, Cita, Adjuntos o Paciente. Toda baja es lógica (deleted_at). |
| Quién puede borrar (lógicamente) | Evento clínico, Medicación | Solo el veterinario autor del registro, o un veterinario de la misma clínica con rol habilitado. El tutor nunca puede borrar datos clínicos. |
| Quién puede borrar (lógicamente) | Adjuntos | Cada rol retira los adjuntos que subió: el veterinario los suyos, el tutor los suyos. Un tutor no borra el estudio que cargó la clínica, ni la clínica la ficha histórica que subió el tutor. La baja es lógica y **no borra el objeto del bucket**: "nunca borrado físico" alcanza también al archivo. |
| Quién puede borrar (lógicamente) | Cita | Un veterinario de la clínica del paciente, o el tutor de la mascota. Es la excepción a "el tutor nunca borra datos clínicos": la Cita es agenda, no historial — el tutor que sabe que no va a llevar a su mascota es quien mejor puede retirarla del calendario, y la baja no destruye nada que se haya registrado sobre la atención. |
| Quién puede borrar (lógicamente) | Tutor | Solo un veterinario de una clínica vinculada a esa ficha. La baja marca deleted_at y no cascadea sobre la cuenta de Usuario del tutor. **Se rechaza mientras la ficha tenga Pacientes vigentes**: dar de baja al tutor dejaría mascotas activas sin nadie a quien contactar. La clínica tiene que resolver primero qué pasa con ellas. |
| Quién puede borrar (lógicamente) | Paciente | Solo un veterinario de la clínica del paciente. El tutor nunca da de baja una mascota: el alta la inicia la clínica y la baja también. No cascadea sobre Eventos clínicos, Medicación, Citas ni Adjuntos (regla 4.5). |
| Quién puede borrar (lógicamente) | Veterinario | Solo el clínica_admin de su misma clínica. La baja marca deleted_at sobre la ficha **y desactiva su cuenta de acceso en la misma transacción**: separarlas dejaría a un ex empleado con acceso cada vez que alguien olvidara el segundo paso. No cascadea sobre lo que ese veterinario escribió — los Eventos clínicos y las Medicaciones conservan su autoría. |

> Toda regla de esta sección que se viole debe rechazar la operación antes de escribir en base — no como validación posterior ni advertencia silenciosa.

### 2.5 Quién da de alta cada tipo de cuenta

| Tipo de cuenta | Quién la crea | Por qué |
|---|---|---|
| Tutor | El propio tutor, por auto-registro público (proceso 4.9) | Es el único punto de entrada abierto del sistema: el tutor debe poder registrarse y empezar a usar la aplicación sin intervención de una clínica. |
| Veterinario | El clínica_admin de la clínica a la que pertenece | Es la gestión del plantel de la clínica (Alcance de Plataformas, 3.2). La cuenta queda con clínica_de_pertenencia_id = la clínica del administrador que la da de alta; no es un dato que el cliente pueda elegir. |
| Clínica_admin | El administrador de la plataforma, fuera de la API (proceso 4.10) | No existe ningún rol dentro del sistema con permiso para crearla: exponerla por HTTP convertiría a cualquier clínica_admin en una vía de escalada de privilegios. |

> En consecuencia, el endpoint de alta de usuarios de la API solo acepta cuentas de tipo veterinario. Las de tutor entran por el registro público y las administrativas por fuera de la API. Esta misma restricción aplica al alta vía Google (4.7): solo el flujo de tutor puede originar una cuenta nueva.

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
| Veterinario | Solo accede a Pacientes cuyo clínica_id coincide con la clínica_id del propio Veterinario. |
| Tutor | Solo accede a Pacientes cuyo tutor_id coincide con el tutor_id asociado a su Usuario. |
| Clínica_admin | Solo accede a datos administrativos (Veterinarios, Clínica) cuya clínica_id coincide con la propia. Sin acceso a Evento clínico ni Medicación de ningún paciente (ver Modelo de Datos, sección 5). |

Sobre la ficha de Tutor (distinta de la cuenta de Usuario del tutor), el alcance se resuelve así:

| Rol | Regla de alcance sobre Tutor |
|---|---|
| Veterinario | **Busca** sin acotar (ver nota abajo). Para **leer, editar o dar de baja** una ficha concreta necesita un vínculo con su clínica: que el tutor tenga al menos un Paciente vigente ahí, o que la ficha la haya creado esa misma clínica (`clínica_de_origen_id`). |
| Tutor | Solo alcanza su propia ficha, vía el tutor_id de su Usuario: puede leerla y editar nombre, contacto, dirección y documento. No puede darla de baja — la baja la decide la clínica, que es quien tiene Pacientes vinculados a ella — ni revocar su consentimiento por esta vía. |
| Clínica_admin | Sin acceso a fichas de Tutor: su rol alcanza datos administrativos (Veterinarios y Clínica), no las personas atendidas. |

Sobre el Paciente, el alcance se resuelve así:

| Rol | Regla de alcance sobre Paciente |
|---|---|
| Veterinario | La cartera de su propia clínica: da de alta, lista, lee, edita cualquier dato básico y da de baja. La clínica del paciente queda fija desde el alta (regla 2.2). |
| Tutor | Solo sus propias mascotas, estén atendidas en la clínica que sea. Lee y edita **únicamente el peso** — el dato no clínico que puede medir en su casa. No da de alta ni de baja. |
| Clínica_admin | Sin acceso: su rol alcanza datos administrativos, no las mascotas atendidas ni su historial (Modelo de Datos, sección 5). |

Sobre los Adjuntos, el alcance se resuelve así:

| Rol | Regla de alcance sobre Adjuntos |
|---|---|
| Veterinario | Los archivos de los pacientes de su clínica: sube, lista, descarga y retira los que subió él. |
| Tutor | Los archivos de sus propias mascotas: sube, lista, descarga y retira los que subió él. |
| Clínica_admin | Sin acceso: los adjuntos cuelgan del Paciente, y su rol no alcanza las mascotas atendidas. |

> Un Adjunto no se edita. Corregir una carga errónea es retirarla y subir otra: la alternativa sería permitir reemplazar el archivo debajo de un registro que ya se referenció desde un evento clínico.

Sobre la Cita, el alcance se resuelve así:

| Rol | Regla de alcance sobre Cita |
|---|---|
| Veterinario | El calendario de los pacientes de su clínica: agenda, lista, lee, reagenda y da de baja. |
| Tutor | Solo las citas de sus propias mascotas. Lee, reagenda (fecha_programada) y decide si quiere que le avisen (notificar_tutor), y puede darlas de baja. No agenda citas nuevas ni cambia el tipo: qué control corresponde es criterio clínico. |
| Clínica_admin | Sin acceso: el calendario cuelga del Paciente, y su rol no alcanza las mascotas atendidas. |

> El listado de pacientes es un endpoint con dos alcances: cuál aplica lo decide el rol del token, nunca un parámetro del cliente. El veterinario ve la cartera de su clínica; el tutor, sus mascotas.

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

Sobre el Evento clínico, el alcance se resuelve así:

| Rol | Regla de alcance sobre Evento clínico |
|---|---|
| Veterinario | Sobre los Pacientes de su propia clínica: carga, lista, lee, **edita y da de baja cualquier evento de esos pacientes, sea suyo o de un colega**. `veterinario_id` registra al autor original y no cambia nunca, ni siquiera al editarlo. |
| Tutor | Solo lectura, y solo sobre los eventos de sus propias mascotas. Nunca escribe ni da de baja datos clínicos (regla 2.4). |
| Clínica_admin | Sin acceso, ni de lectura ni de escritura (Modelo de Datos, sección 5). |

> Que un veterinario pueda editar el evento de un colega es el mismo criterio que la regla 2.4 ya fija para el borrado lógico ("el autor, o un veterinario de la misma clínica"). La alternativa era reservar la edición al autor, pero deja dos criterios distintos sobre la misma entidad: un colega podría borrar el registro entero y no corregirle una fecha. La trazabilidad no se pierde: `veterinario_id` conserva la autoría y la Auditoría registra quién editó qué y cuándo.

Sobre la entidad Usuario, el alcance se resuelve así:

| Rol | Regla de alcance sobre Usuario |
|---|---|
| Clínica_admin | Alcanza las cuentas cuya clínica_de_pertenencia_id coincide con la propia. No alcanza cuentas de tutor (no tienen pertenencia) ni cuentas de otra clínica. No puede darse de baja a sí mismo: dejaría a la clínica sin ninguna cuenta capaz de administrar usuarios. |
| Veterinario, Tutor | Solo alcanzan su propia cuenta: pueden leerla, cambiar su email y su contraseña. |
| Cualquier rol | El campo `activo` solo lo modifica un clínica_admin dentro de su alcance. Permitir que el propio usuario lo cambie vaciaría de sentido la suspensión. |

### 3.3 Dónde se aplica

El motor de permisos se evalúa en la capa de aplicación (servicio/backend), no únicamente en la interfaz. Ninguna pantalla debe ser el único punto de control de acceso: toda operación sobre datos clínicos pasa por esta validación aunque la request no provenga de la interfaz oficial.

> Esto es una mitigación directa del riesgo de acceso indebido a historial clínico identificado en el análisis de viabilidad inicial del proyecto.

## 4. Procesos de negocio

Secuencias de pasos que involucran más de una entidad o más de una validación encadenada.

### 4.1 Alta de paciente

1. El Veterinario (o clínica_admin, solo para datos administrativos) inicia el alta.
2. El Veterinario busca la ficha del Tutor antes de crearla: por el par (tipo_documento, número_documento) si lo conoce, o por coincidencia parcial de nombre o contacto. Buscar por contacto es lo que permite encontrar la ficha de un tutor que ya se auto-registró, porque esa ficha se crea sin documento (4.9) y su contacto es el email del registro.
3. Si el Tutor no existe en el sistema, se crea primero — valida número_documento único (regla 2.1) y exige consentimiento_datos = true. La ficha queda con `clínica_de_origen_id` = la clínica del veterinario, que es lo que la pone a su alcance antes de que exista el Paciente.
4. Se valida consentimiento_datos = true del Tutor (regla 2.2). Si no existe, se solicita antes de continuar.
5. Se crea el Paciente con clínica_id = clínica del Veterinario que realiza el alta. Este valor queda fijo (regla 2.2).
6. Se registra la creación en Auditoría automáticamente (entidad_tipo = "Paciente", acción = "creación").

> Si el Tutor se había auto-registrado (4.9), su ficha no tiene clínica de origen y por lo tanto no está al alcance del veterinario hasta este punto: primero se crea el Paciente y recién después se completan documento y dirección de la ficha.

### 4.2 Carga de un evento clínico

1. El Veterinario autenticado selecciona un Paciente dentro del alcance de su clínica (regla 3.2).
2. Se valida que el Paciente esté vigente: sobre una mascota dada de baja no se cargan episodios nuevos (regla 2.2).
3. Se valida fecha no futura (regla 2.2).
4. Se valida `campo_estructurado` contra el esquema fijo del tipo (regla 2.2 y Modelo de Datos, 4.5): vacuna, medicación y alergia lo exigen completo; los demás tipos lo exigen NULL.
5. Se persiste el Evento clínico con veterinario_id = usuario autenticado (trazabilidad automática, no editable).
6. Se registra la creación en Auditoría con valor_nuevo = contenido del evento.
7. Un evento ya cargado lo puede editar o dar de baja cualquier Veterinario de la clínica del Paciente, no solo su autor (regla 3.2). La edición nunca reasigna `veterinario_id`, y tanto la edición como la baja quedan auditadas.
8. Editar el `tipo` de un evento **descarta el `campo_estructurado` anterior**: el esquema que rige es el del tipo nuevo, y quien cambia el tipo manda los campos que ese tipo exige. Si el tipo nuevo no admite campo estructurado, alcanza con omitirlo.

> Los dos puntos anteriores resuelven casos que los documentos no cubrían y se decidieron al implementar la entidad.
>
> **Evento sobre un Paciente dado de baja**: la regla 4.5 deja el historial de una mascota dada de baja consultable, y también editable — corregir un typo en un registro clínico viejo no depende de que la mascota siga en la cartera. Lo que se rechaza es *agregar* episodios nuevos: no hay atención que registrar sobre una ficha que la clínica cerró. **Pendiente**: el MVP no tiene forma de revertir la baja de un Paciente, así que una ficha cerrada por error deja su historial congelado para altas nuevas. Hay que decidir si la reactivación entra al alcance o si el caso se resuelve dando de alta una ficha nueva.
>
> **Cambio de tipo**: arrastrar el campo estructurado del tipo anterior obligaría a mandar un `null` explícito solo para corregir una carga con el tipo equivocado, que es el caso más común de edición del tipo. El costo asumido es que un cambio de tipo *entre* dos tipos estructurados (de vacuna a medicación, por ejemplo) pierde lo cargado si el cliente no reenvía los campos nuevos — pero esos campos no son traducibles entre esquemas de todos modos.

### 4.3 Ciclo de vida de una medicación

1. Alta: el Veterinario crea un registro de Medicación con fecha_fin = NULL. Se valida que no exista otra medicación activa con la misma droga para ese paciente (regla 2.2).
2. Cierre: para registrar el fin de un tratamiento, el Veterinario actualiza fecha_fin de la medicación existente — no se crea un nuevo registro para "cerrar" el anterior.
3. Cambio de dosis: se interpreta como cierre de la medicación activa + alta de una nueva (pasos 1 y 2 en secuencia), preservando el historial de dosis anteriores en vez de sobrescribir.
4. Cada cambio de estado dispara un registro en Auditoría (acción = "edición", con valor_anterior y valor_nuevo).
5. Reapertura: enviar fecha_fin = NULL sobre una medicación cerrada la vuelve activa. Es cómo se deshace un cierre cargado por error, y se rechaza si dejara dos activas de la misma droga — la regla 2.2 mirada desde el otro lado.
6. Corrección de una carga errónea: `fecha_fin` es lo único editable de una Medicación. Corregir la droga, la dosis o la frecuencia es dar de baja lógica el registro y cargar uno nuevo. Exponer una edición libre de esos campos permitiría sobrescribir una dosis en silencio, que es exactamente lo que el paso 3 viene a evitar; la baja lógica deja el registro errado en el rastro de Auditoría y libera la droga para la medicación activa que corresponde.

### 4.4 Ciclo de vida de una cita

1. Creación: el Veterinario programa una Cita con estado = "pendiente".
2. Confirmación o reagenda: el Tutor puede modificar fecha_programada y notificar_tutor dentro de los permisos definidos (sección 3.2), pero no puede cambiar estado directamente. Solo se reagenda una cita pendiente (regla 2.2).
3. Transición a "cumplido": ocurre cuando el Veterinario carga un Evento clínico con `cita_id` apuntando a esa Cita (Modelo de Datos, 4.5). Se valida que la Cita sea del mismo Paciente que el evento y que no esté ya cumplida. Una Cita **vencida** también pasa a cumplido por esta vía: la mascota llegó tarde y se la atendió igual, y dejarla vencida para siempre falsearía el historial.
4. Transición a "vencido": la cita cuya fecha_programada ya pasó sin haber sido marcada como cumplida cambia de estado automáticamente. Este es un proceso batch/programado, no una acción de usuario.
5. Baja: retirar del calendario una cita que no va a ocurrir es una baja lógica, no un estado. La puede hacer el veterinario de la clínica o el tutor de la mascota (regla 2.4).

> El mecanismo elegido es un job programado (ver sección 4.6) — lo que se fija acá es la regla de negocio: una cita vencida nunca queda indefinidamente en estado "pendiente".

### 4.5 Borrado lógico en cascada de negocio

Cuando se da de baja lógica un Paciente, sus Eventos clínicos, Medicación, Citas y Adjuntos NO se borran automáticamente en cascada. Quedan visibles y consultables desde el Paciente (aunque el Paciente ya no aparezca en listados activos), preservando la trazabilidad completa del historial. Solo un borrado lógico explícito sobre cada entidad hija la retira de las vistas activas de esa entidad en particular.

> Esta regla prioriza no perder trazabilidad clínica por sobre la simplicidad de un borrado en cascada — es consistente con el criterio de trazabilidad y no destrucción definido en el Modelo de Datos.

### 4.6 Transición automática de citas vencidas (job programado)

1. Un proceso interno del backend corre en forma periódica y consulta las Citas con estado = "pendiente" cuya fecha_programada ya pasó. La frecuencia es **cada hora**, configurable por entorno (`CITAS_VENCIDAS_INTERVAL`), y arranca con una corrida inmediata al levantar el proceso. Con una corrida diaria alcanzaría —`fecha_programada` es una fecha, no un instante—; se corre cada hora porque acota la ventana en que una cita ya pasada sigue figurando pendiente y porque hace que el job se repare solo después de una caída, a costo de una consulta indexada por hora.
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

> Esta regla reemplaza al criterio anterior, según el cual el tutor solo podía crear su cuenta si una clínica ya había registrado su ficha. Se priorizó que el tutor pueda empezar a usar la aplicación sin depender de una clínica. La contrapartida es la deduplicación: si una clínica carga una ficha de Tutor para alguien que ya se auto-registró (o viceversa), quedan dos fichas para la misma persona. El criterio de fusión queda pendiente de definición — ver Modelo de Datos, 4.1.
>
> Este es el único punto del sistema donde el paso 3 de 4.7 puede crear un Usuario nuevo. Para cualquier otro tipo_usuario, Google solo vincula cuentas ya existentes (2.5, 4.7).

### 4.10 Alta de una clínica y de su cuenta administrativa

1. El administrador de la plataforma da de alta la Clínica y su cuenta clínica_admin en una sola operación, por fuera de la API HTTP (una herramienta de línea de comandos que invoca la misma capa de negocio).
2. La cuenta se crea con clínica_id y clínica_de_pertenencia_id apuntando a la clínica recién creada, y con una contraseña inicial que el administrador entrega a la clínica.
3. A partir de ahí, ese clínica_admin es responsable de dar de alta las cuentas de los veterinarios de su clínica (regla 2.5).

> El administrador de la plataforma no es un tipo_usuario del sistema: no tiene cuenta, no se autentica y no aparece en el motor de permisos. Es un operador con acceso al despliegue. Si en algún momento esa responsabilidad se delega o necesita ser auditada, habrá que decidir explícitamente la creación de un cuarto rol.

### 4.12 Alta de un veterinario

1. El clínica_admin da de alta la ficha de Veterinario y su cuenta de acceso **en una única transacción**: una ficha sin cuenta dejaría a alguien en el plantel sin poder entrar al sistema, y una cuenta sin ficha violaría la regla de integridad 2.1.
2. La clínica no se envía en el pedido: es siempre la del administrador que da el alta (regla 2.5). Tanto la ficha como la cuenta quedan en esa clínica.
3. Se valida el par (tipo_documento, número_documento) único entre fichas vigentes (regla 2.1), el email libre en todo el sistema y la política de contraseña.
4. La matrícula es opcional. Sin ella la ficha existe en modo restringido: no habilita a crear ni editar Eventos clínicos ni Medicación (regla 2.2).

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

### 4.15 Recordatorios de una Cita al tutor

El calendario existe para que la mascota llegue a su control; el recordatorio es lo que convierte una fecha anotada en una consulta hecha. Por eso los dos únicos avisos definidos son los que reducen el ausentismo, y no los de confirmación ni los de cita vencida: un aviso posterior al hecho no cambia nada, y uno por cada escritura enseña al tutor a ignorarlos.

1. **Canal**: notificación push al móvil del tutor. Es la única plataforma desde la que el tutor accede (Alcance de Plataformas, sección 2), así que el email sería un segundo canal para el mismo destinatario. La cuenta registra sus dispositivos al iniciar sesión desde la app; una cuenta sin dispositivo registrado no recibe avisos.
2. **Momentos**: dos recordatorios por Cita — uno el día anterior a `fecha_programada` y otro el mismo día. `fecha_programada` es una fecha sin hora, así que ambos se envían a una hora fija del día que corresponde (configurable por entorno), y no a una distancia exacta de la cita.
3. **Encolado**: un proceso periódico busca las Citas pendientes con `notificar_tutor = true` que entran en la ventana de recordatorio y encola las notificaciones que falten. El encolado es un barrido y no un efecto de la escritura de la Cita: así una cita reagendada, una notificación perdida o un backend que estuvo caído se resuelven solos en la corrida siguiente, sin depender de que nadie vuelva a tocar la cita. Encolar es idempotente: existe una sola notificación de cada tipo por Cita, sostenido por un índice único. Reagendar una cita no duplica su recordatorio.
4. **Envío**: el mismo proceso despacha las notificaciones cuya hora ya llegó, reintentando las que fallan hasta un techo de intentos. Una notificación que agota los intentos queda registrada como fallida y no se reintenta más: un aviso de una cita que ya pasó no sirve.
5. **Teléfono rechazado**: cuando el proveedor informa que un aparato dejó de existir (la app se desinstaló, el token caducó), ese Dispositivo se da de baja en vez de reintentarse. Reintentar contra un teléfono que ya no existe gasta los intentos de la notificación sin ninguna chance de éxito. Un aviso que no llegó a ningún aparato no se da por enviado.
6. **Destinatario**: la cuenta del tutor de la mascota, resuelta al encolar. Una ficha de tutor sin cuenta de Usuario no recibe avisos: no hay a dónde mandarlos. Los **dispositivos** de esa cuenta, en cambio, se resuelven recién al despachar, porque entre encolar y enviar pasan horas y el tutor puede haber registrado su primer teléfono en el medio.
7. Las notificaciones **no se auditan**: la Auditoría registra cambios sobre entidades clínicas (Modelo de Datos, 4.10), y un aviso no modifica el historial. Su rastro es la propia tabla, que guarda qué se envió, cuándo y con qué resultado.

## 5. Fuera de alcance de este documento

- **Deduplicación de fichas de Tutor** — cómo se fusionan la ficha de un tutor auto-registrado y la que una clínica pudo haber creado para la misma persona (ver 4.9).
- **Auditoría de la entidad Usuario** — el alta, la edición, la suspensión y la baja de cuentas no generan registro de Auditoría hoy: la sección 4.10 del Modelo de Datos la limita a Evento clínico, Medicación, Cita y Paciente. Queda por decidir si las operaciones sobre cuentas deben auditarse.
- **Recuperación de contraseña, verificación de email y bloqueo por intentos fallidos** — no definidos.
- **Vista de paciente derivado en urgencia** — priorización de datos mostrados, tiempos de carga aceptables, y su relación con la Fase 2 (matching geolocalizado).

Ambas quedan pendientes de definición en una etapa posterior del proyecto.
