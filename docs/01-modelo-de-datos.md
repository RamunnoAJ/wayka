# Wayka — Modelo de Datos

Historial clínico colaborativo y calendario — MVP
Versión 1.0 · Documento técnico de referencia

## 1. Alcance y criterios de diseño

Este documento define el modelo de datos para la fase de MVP de Wayka: historial clínico colaborativo y calendario de eventos, con un piloto de una única clínica veterinaria. El modelo se diseñó bajo tres criterios rectores:

- **Portabilidad**: los registros deben poder ser leídos por un veterinario ajeno a la clínica de origen en una futura Fase 2 (matching geolocalizado de urgencias).
- **Permisos claros**: el veterinario es la fuente de verdad clínica; el tutor tiene lectura y puede editar únicamente datos no clínicos.
- **Trazabilidad y no destrucción**: ningún dato clínico se borra físicamente; todo cambio queda auditado. Esto responde al riesgo legal de responsabilidad ante errores identificado en la etapa de análisis de viabilidad (Ley 25.326 de Protección de Datos Personales).

Alcance del MVP: un paciente pertenece a una única clínica durante toda esta fase (relación fija, sin tabla intermedia). La multi-clínica queda reservada para la Fase 2.

## 2. Diagrama de relaciones

```
Tutor 1───N Paciente N───1 Clínica
                │
                ├──N Evento clínico ──N Adjuntos
                │         └── veterinario_id (trazabilidad)
                │
                ├──N Medicación (activa si fecha_fin IS NULL)
                └──N Cita (calendario) ──N Notificación

Veterinario N───1 Clínica

Usuario N───1 Tutor  (si tipo_usuario = tutor)
Usuario N───1 Veterinario  (si tipo_usuario = veterinario)
Usuario N───1 Clínica  (si tipo_usuario = clínica_admin)
Usuario N───1 Clínica  (clínica_de_pertenencia_id: veterinario y clínica_admin)

Auditoría ──registra cambios de──> Evento clínico, Medicación, Cita, Paciente, Adjuntos
```

## 3. Campos transversales

Los siguientes tres campos están presentes en todas las entidades principales (Tutor, Paciente, Clínica, Veterinario, Usuario, Evento clínico, Medicación, Cita, Adjuntos):

| Campo | Tipo | Descripción |
|---|---|---|
| created_at | timestamp | Fecha y hora de creación del registro. Obligatorio. |
| updated_at | timestamp | Fecha y hora de la última modificación. Se actualiza en cada UPDATE. |
| deleted_at | timestamp (nullable) | NULL = registro activo. Con fecha = borrado lógico. Nunca se elimina físicamente. |

> El borrado lógico evita eliminar información clínica de forma irreversible. Las vistas activas del sistema filtran siempre por `deleted_at IS NULL`.
>
> `created_at` y `updated_at` se exponen en la API de todas las entidades. `deleted_at` **no**: un registro dado de baja simplemente no aparece en los listados, y devolver el campo en todas partes sugeriría que el cliente puede pedir los borrados, que no es el caso. La única excepción es **Paciente**, donde sí se expone — ver la nota de 4.2.

## 4. Entidades

### 4.1 Tutor

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| nombre | string | Nombre completo del tutor. |
| tipo_documento | enum (nullable) | DNI / Pasaporte u otro documento de identidad extranjero. NULL hasta que se complete la ficha. |
| número_documento | string (nullable) | Único junto con tipo_documento. DNI para residentes argentinos; pasaporte u otro documento equivalente para no residentes. NULL hasta que se complete la ficha. |
| contacto | string | Teléfono y/o email. |
| dirección | string (nullable) | Útil para Fase 2 (geolocalización). |
| clínica_de_origen_id | UUID / FK (nullable) | Clínica que dio de alta la ficha. NULL si el tutor se auto-registró. Sostiene el alcance de esa clínica sobre la ficha entre el momento en que la crea y el momento en que le da de alta un Paciente (Reglas de Negocio, 3.2). |
| consentimiento_datos | boolean + fecha | Consentimiento explícito de uso de datos (Ley 25.326). | No se revoca por la API: la edición de la ficha no expone el campo, porque la ley exige rastro del otorgamiento y no una baja silenciosa.

> El par `(tipo_documento, número_documento)` es único entre fichas vigentes: evita duplicar tutores y sirve como criterio de búsqueda/verificación en Fase 2 (matching geolocalizado). La unicidad es por par y no sobre el número solo, porque un DNI y un pasaporte pueden compartir numeración sin ser la misma persona.
>
> Los campos de documento y dirección son nullables porque el auto-registro de tutor (Reglas de Negocio, 4.9) solo pide nombre, contacto y consentimiento: exigir el documento en ese punto agregaría fricción a un alta que debe ser inmediata. Se completan cuando una clínica atiende al tutor por primera vez. La consecuencia es que dos fichas sin documento cargado no colisionan entre sí — la deduplicación al vincular un tutor auto-registrado con la ficha que una clínica podría haber creado en paralelo queda pendiente de definición.

### 4.2 Paciente (mascota)

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| nombre | string | Nombre de la mascota. |
| especie | string | Ej: canino, felino. |
| raza | string | — |
| fecha_nacimiento | date | — |
| sexo | string | — |
| peso_actual | decimal | Editable por tutor o veterinario. |
| tutor_id | UUID / FK | Referencia a Tutor. |
| clínica_id | UUID / FK | Clínica a la que pertenece (fija en el MVP). |
| identificador_externo | string (nullable) | Número de chip/microchip. Clave para portabilidad en Fase 2. |

> `peso_actual` se persiste como NUMERIC, no como punto flotante binario: el peso se compara y se muestra al gramo, y un `double` redondea de formas que en una historia clínica se notan. Debe ser mayor a cero.
>
> `identificador_externo` es único entre fichas vigentes cuando está cargado: dos mascotas no pueden compartir número de chip.
>
> La baja del Paciente es lógica y no cascadea (Reglas de Negocio, 4.5). Ni `clínica_id` ni `tutor_id` son editables: la primera es fija en el MVP (regla 2.2) y cambiar la segunda sería transferir la mascota a otra persona sin dejar rastro de la transferencia.
>
> **`deleted_at` se expone en la API de Paciente**, a diferencia del resto de las entidades. Es la consecuencia directa de la regla 4.5: la ficha de una mascota dada de baja se sigue leyendo, con su historial, su medicación, sus citas y sus adjuntos completos — pero no admite escrituras nuevas. Sin el campo en la respuesta, el cliente no tiene forma de distinguir esa ficha de una vigente y le ofrecería al veterinario acciones que el backend va a rechazar. Es un dato de presentación, no un permiso: quién puede leerla lo sigue decidiendo el motor de permisos.

### 4.3 Clínica

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| nombre | string | — |
| dirección | string | — |
| contacto | string | — |
| hora_apertura | time | Hora a la que la clínica empieza a atender. |
| hora_cierre | time | Hora a la que deja de atender. Posterior a `hora_apertura`. |
| duración_turno_minutos | int | Largo del turno con el que se agenda. Define la grilla del calendario. |

> Los tres campos de agenda entraron después de la primera versión de esta tabla, junto con la decisión de darle hora a la Cita (4.7): sin ellos, una cita con hora se puede agendar a cualquier minuto de cualquier momento del día, y el calendario deja de ser una grilla para pasar a ser una lista de instantes sueltos.
>
> `hora_apertura` y `hora_cierre` son un **único intervalo para toda la semana**, no un horario por día ni con corte de mediodía. Es la simplificación deliberada del MVP: un horario por día multiplica por siete la configuración, y con una sola clínica piloto todavía no hay evidencia de que haga falta. El día que la haya, este es el campo que se promueve a una tabla de franjas — no un campo al que se le agregan casos especiales.
>
> `duración_turno_minutos` es de la Clínica y no del Veterinario porque en el MVP la agenda es de la clínica: no hay agenda por profesional (ver la nota de 4.7 sobre por qué la Cita no lleva `veterinario_id`). Debe ser mayor a cero y dividir de forma exacta el intervalo de atención, o la grilla deja un hueco al final del día.
>
> No se modelan **especialidades**, ni de la clínica ni del veterinario. Ninguna regla de negocio ni pantalla del MVP las consume: serían un dato que se carga y nadie lee. Cuando exista el criterio que las use (derivar una urgencia, filtrar el plantel), ahí se decide si son un enum fijo o una entidad propia.

### 4.4 Veterinario

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| nombre | string | — |
| tipo_documento | enum | DNI / Pasaporte u otro documento de identidad extranjero. |
| número_documento | string | Único (constraint UNIQUE). DNI para residentes argentinos; pasaporte u otro documento equivalente para no residentes. |
| matrícula | string | Habilitación profesional. Preparado para validación automática en Fase 2. |
| clínica_id | UUID / FK | Clínica a la que pertenece. |

> `número_documento` es independiente de `matrícula`: identifica a la persona, mientras que `matrícula` identifica su habilitación profesional. Ambos se validan por separado.
>
> `matrícula` es nullable: la ficha puede existir sin ella, en modo restringido (Reglas de Negocio, 2.2). `número_documento` no lo es — a diferencia del Tutor, que puede auto-registrarse sin documento, el Veterinario siempre lo da de alta la clínica, que tiene el dato a mano. La unicidad es por el par (tipo_documento, número_documento) y entre fichas vigentes, con el mismo criterio que Tutor.
>
> La ficha y la cuenta de Usuario del veterinario se crean juntas y se dan de baja juntas (procesos 4.12 y 4.13). La baja es lógica (`deleted_at`) aunque Veterinario no sea una entidad clínica: lo que escribió conserva su autoría.

### 4.5 Evento clínico

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | — |
| veterinario_id | UUID / FK | Quién cargó el evento (trazabilidad). |
| tipo | enum | Consulta / vacuna / cirugía / control / urgencia / medicación / alergia. |
| fecha | date | — |
| descripción | text | Texto libre del veterinario. |
| diagnóstico | text (nullable) | — |
| campo_estructurado | JSON (nullable) | Datos estructurados para tipos críticos (ver nota). |
| cita_id | UUID / FK (nullable) | Cita que esta atención vino a cumplir. NULL cuando la atención no estaba agendada. |

> `cita_id` es el vínculo entre Evento clínico y Cita que la regla 4.4 de Reglas de Negocio dejaba pendiente del diseño técnico. La FK vive del lado del Evento clínico, no de la Cita, por dos motivos: la mayoría de las atenciones no nacen de una cita (la FK es nullable en el lado que efectivamente puede estar vacío), y una misma Cita no se cumple dos veces, así que no hace falta la colección que la dirección inversa habilitaría. Cargar un evento con `cita_id` es lo que transiciona esa Cita a `cumplido` (Reglas de Negocio, 4.4).

> Se recomienda estructurar como campos fijos (no texto libre) al menos: vacunas, medicación activa y alergias — son los datos que un veterinario ajeno necesita leer en segundos durante una urgencia.

El esquema de `campo_estructurado` es fijo y lo valida el backend según el `tipo` del evento. No es un JSON libre: si no cumple la forma de su tipo, la operación se rechaza antes de escribir (regla 2.2).

| tipo | Forma exigida de campo_estructurado |
|---|---|
| vacuna | `{ nombre_vacuna: string (requerido), lote: string (requerido), fecha_proxima_dosis: date (opcional) }` |
| medicación | `{ nombre_droga: string (requerido), dosis: string (requerido), frecuencia: string (requerido) }` |
| alergia | `{ alergeno: string (requerido), severidad: enum leve/moderada/severa (requerido), reaccion: string (opcional) }`. `fecha` es la fecha de detección. |
| consulta, cirugía, control, urgencia | Debe ser NULL. Enviar un valor se rechaza. |

> El campo se valida contra un esquema fijo en vez de aceptarse como JSON libre porque el motivo entero de tenerlo es la lectura en urgencia: un dato que cada clínica escribe con las claves que quiera no es consultable, y equivale a texto libre en otro envoltorio. La contrapartida asumida es que agregar un tipo estructurado nuevo pide tocar el backend, no solo cargar datos.
>
> Las claves de `tipo = medicación` replican los campos tipados de la entidad Medicación (4.6) a propósito: el Evento clínico registra **que se indicó** una medicación en una fecha (hecho histórico, inmutable), mientras que la entidad Medicación lleva el **tratamiento vigente** y su ciclo de vida (`fecha_fin`, regla de una activa por droga). Son dos preguntas distintas y la duplicación es el precio de poder responder ambas.
>
> Las alergias se cubren como un `tipo` más de Evento clínico, no como entidad propia. La alternativa era una tabla espejo de Medicación (4.6), que modela mejor un estado vigente que un hecho histórico; se descartó para el MVP porque duplicaría alcance, auditoría y baja lógica que el Evento clínico ya resuelve. La consecuencia asumida es que una alergia no tiene estado *vigente / superada*: solo está registrada o dada de baja (`deleted_at`), y descartar un diagnóstico errado es darla de baja. Si aparece la necesidad de distinguir una alergia superada de una mal diagnosticada, ese es el punto donde conviene promoverla a entidad.
>
> La vista de urgencia de un paciente se arma con sus Eventos clínicos de `tipo = alergia` sin `deleted_at`, junto con sus Medicaciones activas (`fecha_fin` NULL) y sus vacunas.

### 4.6 Medicación

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | — |
| veterinario_id | UUID / FK | Quién indicó el tratamiento. No se reasigna al cerrarlo ni al corregirlo. |
| nombre_droga | string | — |
| dosis | string | — |
| frecuencia | string | — |
| fecha_inicio | date | — |
| fecha_fin | date (nullable) | NULL indica medicación activa. Filtro clave para vista de urgencia. |

> `veterinario_id` no figuraba en la primera versión de esta tabla, pero las Reglas de Negocio ya lo exigían en dos lugares: la baja lógica está reservada al *veterinario autor del registro* (regla 2.9) y la baja de un Veterinario "no cascadea sobre lo que ese veterinario escribió: los Eventos clínicos y las Medicaciones conservan su autoría" (proceso 4.7). Sin el campo, ninguna de las dos reglas era evaluable.

> `fecha_fin` registra el **cierre efectivo** del tratamiento, no su duración planificada: una fecha futura dejaría de contar como activa a una medicación que el paciente todavía está recibiendo, y "activa" está definido acá como `fecha_fin IS NULL`. Prescribir "por 7 días" se registra hoy en `frecuencia` y se cierra el día que termina. Modelar la duración planificada exigiría separar *fin previsto* de *fin efectivo*, y queda fuera del MVP.

### 4.7 Cita (calendario)

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | — |
| tipo | enum | Próxima vacuna / control / cirugía programada. |
| fecha_programada | timestamp | Momento de la cita, con hora. Cae dentro del horario de atención de la clínica y sobre la grilla de turnos (4.3). |
| estado | enum | Pendiente / cumplido / vencido. |
| notificar_tutor | boolean | Dispara notificación al tutor. |

> `estado` no es un campo que el cliente escriba: nace en `pendiente`, pasa a `cumplido` cuando se carga el Evento clínico que la referencia por `cita_id` (4.5) y a `vencido` por el job programado (Reglas de Negocio, 4.6). No hay endpoint que lo reciba — exponerlo dejaría marcar como cumplida una cita que nadie atendió.

> `fecha_programada` era una fecha sin hora en la primera versión de esta tabla. Pasó a timestamp porque una agenda de veterinaria sin hora no es una agenda: dos cirugías el mismo martes no son intercambiables, y el calendario no podía mostrar más que "hay algo ese día". El costo asumido es que ahora hay una zona horaria en juego — se persiste en UTC y se presenta en `America/Argentina/Buenos_Aires`, que es la zona de operación del MVP y la misma que ya usan los logs (Backend, doc 07).
>
> La Cita **sigue sin llevar `veterinario_id`**, aun con hora. A diferencia de Evento clínico y Medicación, ninguna regla se apoya en su autor: la baja no está reservada a quien la creó y el alcance se resuelve contra la mascota. Quién la programó y quién la reagendó queda en Auditoría, que es donde se lo consulta.
>
> La consecuencia de esa ausencia es que la agenda es **de la clínica, no de cada profesional**: dos citas del mismo horario no colisionan, porque no hay a quién asignárselas. Es una limitación conocida y no un descuido — asignar profesional cambia el motor de permisos (el alcance de la Cita pasaría a tener un segundo eje) y es el paso siguiente, no este.

### 4.8 Adjuntos

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | Mascota a la que pertenece el archivo. Siempre presente. |
| evento_id | UUID / FK (nullable) | Completo cuando el adjunto documenta un evento clínico concreto; NULL cuando es un adjunto general del paciente (ej. ficha histórica escaneada). |
| subido_por_usuario_id | UUID / FK | Cuenta que subió el archivo. Referencia a Usuario y no a Veterinario o Tutor, porque suben los dos roles. |
| tipo | enum | Foto / PDF / estudio. |
| clave_de_archivo | string | Ruta del objeto dentro del bucket. Es opaca para el cliente y no se expone en la API. |
| nombre_archivo | string | Nombre original con el que se subió, para que la descarga no se llame como la clave. |
| content_type | string | Tipo MIME verificado contra el `tipo` declarado. |
| tamano_bytes | bigint | Tamaño del objeto, validado contra el máximo permitido antes de escribir. |

> `paciente_id` pasó a ser obligatorio y `evento_id` quedó como el único nullable de los dos. La primera versión los tenía a ambos nullable y mutuamente excluyentes; con esa forma, resolver el alcance de un adjunto colgado de un evento exigía ir a buscar el evento para llegar a la mascota, y nada impedía una fila con las dos FK vacías. Todo adjunto pertenece a una mascota; que además documente un evento es información adicional.

> `subido_por_usuario_id` no figuraba en la primera versión de esta tabla. Lo exige la regla de baja (Reglas de Negocio, 2.4): cada rol retira los adjuntos que subió, y sin el campo esa regla no es evaluable.

> `archivo_url` se reemplazó por `clave_de_archivo`. El bucket es privado: no existe una URL estable que guardar. La API expone en cada lectura una URL prefirmada de vida corta, que se calcula en el momento y no se persiste — guardar una URL sería guardar un permiso vencido (Arquitectura, 3.4).

### 4.9 Usuario

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| email | string | Único (constraint UNIQUE). Usado para login. |
| password_hash | string (nullable) | Contraseña hasheada. NULL si el usuario solo se autentica vía Google y nunca configuró contraseña propia. |
| google_id | string (nullable) | Único (constraint UNIQUE). ID de cuenta de Google vinculada. NULL si el usuario nunca vinculó Google. |
| avatar_url | string (nullable) | URL de avatar, obtenido de Google al vincular la cuenta. |
| tipo_usuario | enum | Tutor / veterinario / clínica_admin. |
| tutor_id | UUID / FK (nullable) | Completo solo si tipo_usuario = tutor. |
| veterinario_id | UUID / FK (nullable) | Completo solo si tipo_usuario = veterinario. |
| clínica_id | UUID / FK (nullable) | Completo solo si tipo_usuario = clínica_admin. |
| clínica_de_pertenencia_id | UUID / FK (nullable) | Clínica a la que pertenece la cuenta, independiente de la FK de rol. NULL para tutor; obligatoria para veterinario; igual a clínica_id para clínica_admin. |
| activación_pendiente | boolean | `true` entre que la línea de comandos crea la cuenta de clínica_admin y que alguien la estrena. Solo mientras vale `true` la cuenta puede no tener ningún método de autenticación. |
| último_acceso | timestamp (nullable) | Fecha del último login exitoso. |
| activo | boolean | Permite desactivar la cuenta sin afectar el historial que generó. |

> Exactamente una de las tres FK (tutor_id, veterinario_id, clínica_id) debe estar completa según tipo_usuario — regla de integridad a validar en la aplicación (o vía CHECK constraint si el motor lo soporta). Separar Usuario de Tutor/Veterinario permite que una misma persona tenga distintos roles a futuro y centraliza la lógica de autenticación en un solo lugar.
>
> `clínica_de_pertenencia_id` existe porque el motor de permisos necesita acotar el alcance de un clínica_admin sobre las cuentas de su clínica, y `clínica_id` solo lo llevan las cuentas administrativas: la clínica de un veterinario vive en la tabla Veterinario. Sin este campo el alcance no sería evaluable sobre una cuenta de veterinario y quedaría abierto — un clínica_admin podría administrar cuentas de otra clínica. Es una denormalización deliberada al servicio del motor de permisos, no un reemplazo de `Veterinario.clínica_id`. Una cuenta sin pertenencia (tutor) queda fuera del alcance de cualquier clínica.
>
> Un Usuario debe tener al menos uno de password_hash o google_id no nulo — nunca ambos NULL, o no tendría forma de autenticarse. Ver Reglas de Negocio, sección 2.1, para el detalle de esta validación y el proceso de vinculación de cuenta Google.
>
> `activación_pendiente` es lo que hace expresable la única excepción a esa regla. Existe como campo y no como "deducilo de que no tiene ninguno de los dos" porque la restricción vive en la base, y una restricción no puede consultar otra tabla para saber si hay un token de activación sin usar. El campo pasa a `false` en la misma transacción en que se escribe la contraseña (proceso 4.16), así que la ventana en que la excepción aplica es exactamente la que dura la activación.
>
> No se expone en la API. `tiene_contrasena` y `tiene_google_vinculado` en `false` a la vez ya dicen lo mismo para quien lo necesite, y agregar el campo invitaría a que un cliente decida algo a partir de él.
>
> Los tokens de refresco viven en una tabla `sesión` aparte, que **no es una entidad de dominio**: no la lee ni la escribe ninguna regla de negocio fuera de la autenticación, no se audita y no aparece en el motor de permisos. Guarda el hash del token (nunca el token), el canal de origen, la cadena de rotación y su vencimiento. Su diseño está en Arquitectura, sección 4.2.

### 4.10 Auditoría

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| entidad_tipo | string | Ej: "Evento clínico", "Medicación". |
| entidad_id | UUID | Registro específico afectado. |
| acción | enum | Creación / edición / borrado_lógico. |
| usuario_id | UUID (nullable) | Quién realizó la acción. NULL si fue una acción automática (usuario_tipo = sistema). |
| usuario_tipo | enum | Veterinario / tutor / clínica_admin / sistema. |
| fecha_hora | timestamp | — |
| valor_anterior | JSON (nullable) | Estado previo del registro. |
| valor_nuevo | JSON | Estado posterior del registro. |

> Se completa automáticamente vía lógica de aplicación o triggers en cada creación, edición o borrado lógico de las entidades clínicas. No depende de que el usuario registre nada manualmente.
>
> Adjuntos entró a la Auditoría después de la primera versión de esta sección: una radiografía o la foto de una herida son dato clínico, y que aparezcan o desaparezcan del historial tiene que dejar rastro igual que el evento del que cuelgan. El rastro guarda los metadatos del archivo, nunca su contenido.

> `usuario_tipo = sistema` cubre acciones ejecutadas por procesos automáticos del backend (ej. el job programado que transiciona Citas vencidas), no por una persona autenticada. Ver Reglas de Negocio, sección 4.6.

### 4.11 Dispositivo

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| usuario_id | UUID / FK | Cuenta a la que pertenece el teléfono. |
| token_push | string | Único (constraint UNIQUE). Token que el proveedor de push asigna a la instalación de la app. |
| plataforma | enum | iOS / Android. |

> Un token identifica un teléfono, no una persona: si en el mismo aparato entra otra cuenta, el token se reasigna a la nueva (Reglas de Negocio, 2.1). La baja es lógica como en el resto del modelo, y es lo que hace el cierre de sesión.

> El Dispositivo existe solo para el canal push. No es una entidad clínica: no se audita y su baja no afecta a nada del historial.

### 4.12 Notificación

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| usuario_id | UUID / FK | Destinatario. Hoy siempre la cuenta de un tutor. |
| cita_id | UUID / FK | Cita que origina el recordatorio. |
| tipo | enum | Recordatorio del día anterior / recordatorio del mismo día. |
| titulo | string | Título del aviso, ya redactado al encolar. |
| cuerpo | string | Texto del aviso. No lleva diagnóstico ni contenido clínico: dice qué mascota y qué día. |
| estado | enum | Pendiente / enviada / fallida / descartada. |
| intentos | int | Envíos intentados. Al llegar al techo, la notificación queda fallida. |
| programada_para | timestamp | Momento a partir del cual corresponde enviarla. |
| enviada_at | timestamp (nullable) | Cuándo se envió efectivamente. |
| ultimo_error | text (nullable) | Motivo del último fallo, para diagnóstico. |

> Es una tabla de salida: el recordatorio se encola y un proceso aparte lo despacha (Reglas de Negocio, 4.15). No lleva `deleted_at` — no es una entidad del dominio clínico que se dé de baja, sino el registro operativo de un envío. `descartada` cubre el aviso que dejó de tener sentido antes de salir.

> `cita_id` es una FK concreta y no un par entidad_tipo/entidad_id: hoy el único origen de una notificación es una Cita. Si aparece un segundo origen, ese es el momento de generalizarlo, no antes.

> El par (`cita_id`, `tipo`) es único: es lo que hace idempotente al encolado y evita que reagendar una cita duplique su recordatorio.

## 5. Matriz de permisos

| Entidad | Quién escribe | Quién lee |
|---|---|---|
| Evento clínico, Medicación | Solo veterinario, sobre los pacientes de su clínica: cualquiera del plantel edita y da de baja, no solo el autor (Reglas de Negocio, 3.2) | Veterinario (todo) + Tutor (solo lectura) |
| Cita / Calendario | Veterinario crea; Tutor confirma/reagenda | Ambos |
| Adjuntos | Ambos | Ambos |
| Dispositivo | Cada usuario los suyos (los registra al entrar y los da de baja al salir) | Cada usuario los suyos |
| Notificación | Nadie: las escribe el proceso que las encola y las despacha | Nadie por API en el MVP: llegan como push, no se listan |
| Paciente (datos básicos) | Veterinario de su clínica (alta, edición, baja); Tutor edita solo el peso | Veterinario de la clínica + el tutor dueño. Clínica_admin sin acceso |
| Tutor (ficha) | Veterinario de una clínica vinculada a esa ficha (alta y edición, incluido completar documento y dirección); el propio tutor sobre la suya, salvo el consentimiento | Búsqueda: cualquier veterinario. Ficha concreta: veterinario vinculado + el propio tutor. Clínica_admin sin acceso |
| Evento clínico, Medicación (acceso de clínica_admin) | Sin acceso | Sin acceso — reservado a veterinario y tutor |
| Veterinario (ficha del plantel) | Clínica_admin, sobre el plantel de su propia clínica | Clínica_admin (edita) + Veterinario de la misma clínica (solo lectura). Tutor sin acceso |
| Usuario (cuentas de veterinario) | Clínica_admin, sobre las cuentas de su propia clínica | Clínica_admin (las de su clínica) + el propio usuario sobre su cuenta |
| Usuario (cuenta propia) | Cada usuario sobre su email y su contraseña | Cada usuario sobre su cuenta |
| Usuario (cuentas de tutor) | Solo el propio tutor (auto-registro) | Solo el propio tutor |
| Clínica (datos administrativos y horario de atención) | Alta: administrador de la plataforma, fuera de la API. Edición: clínica_admin sobre su propia clínica | Clínica_admin sobre su propia clínica. Veterinario de esa clínica, solo lectura: necesita el horario de atención para agendar |
| Usuario (cuentas de clínica_admin) | Administrador de la plataforma, fuera de la API | Clínica_admin sobre su propia cuenta |

> El alcance del veterinario sobre la ficha de Tutor no está acotado a su clínica, a diferencia del que tiene sobre Paciente. El motivo es el proceso de alta de paciente (Reglas de Negocio, 4.1): la ficha del tutor se busca y se completa antes de que exista ningún Paciente que vincule a esa persona con una clínica, así que no hay dato sobre el cual acotarla. Es una excepción deliberada, a revisar cuando exista la entidad Paciente.

> El rol clínica_admin gestiona veterinarios y datos administrativos de la clínica, pero no tiene acceso al historial clínico de pacientes por defecto — ese acceso permanece exclusivo del veterinario que atiende al paciente y del tutor. Si el negocio requiere lo contrario, debe decidirse explícitamente antes de implementarlo.

## 6. Preparado para Fase 2

Los siguientes campos no se utilizan activamente en el MVP pero se incluyen desde ahora para evitar una migración de esquema costosa cuando se incorpore el matching geolocalizado de urgencias:

- **Paciente.identificador_externo** — permite localizar el historial vía chip/microchip sin depender del nombre de la clínica de origen.
- **Veterinario.matrícula** — base para validación profesional automática (tipo KYC).
- **Paciente.clínica_id** — modelado como FK simple en el MVP; migrará a tabla intermedia N:N cuando un paciente pueda ser atendido por más de una clínica.
- **Evento clínico.campo_estructurado** — permite sumar campos de triaje estructurado sin romper el esquema existente.
