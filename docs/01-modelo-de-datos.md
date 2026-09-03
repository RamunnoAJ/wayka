# Wayka — Modelo de Datos

Historial clínico colaborativo y calendario — MVP
Versión 1.0 · Documento técnico de referencia

## 1. Alcance y criterios de diseño

Este documento define el modelo de datos para la fase de MVP de Wayka: historial clínico colaborativo y calendario de eventos, con un piloto de una única clínica veterinaria. El modelo se diseñó bajo tres criterios rectores:

- **Portabilidad**: los registros deben poder ser leídos por un veterinario ajeno a la clínica de origen en una futura Fase 2 (matching geolocalizado de urgencias).
- **Permisos claros**: el veterinario es la fuente de verdad clínica; el tutor es el dueño de la ficha de su mascota y de quién la ve. El tutor **escribe los antecedentes de su propia mascota** —lo que sabe o tiene anotado de antes de Wayka—, y todo lo que escribe queda marcado como suyo (`cargado_por = tutor`, 4.5): nunca crea ni edita un registro del veterinario, y lo que carga no se confunde con lo que escribió un profesional. Además edita los datos no clínicos de la mascota y decide con qué clínicas y con qué otras personas la comparte.
- **Trazabilidad y no destrucción**: ningún dato clínico se borra físicamente; todo cambio queda auditado. Esto responde al riesgo legal de responsabilidad ante errores identificado en la etapa de análisis de viabilidad (Ley 25.326 de Protección de Datos Personales).

> **Hasta esta versión el tutor no escribía ni una línea de dato clínico**, y el criterio decía exactamente eso. Cambió porque casi ninguna mascota llega a Wayka sin historia: las vacunas puestas, las alergias conocidas y la medicación que está tomando existen, solo que en una libreta de papel o en la memoria del dueño, y el día del alta la única persona que puede volcarlas es él. Una ficha que arranca vacía y espera a la próxima consulta para tener algo adentro no sirve para lo que la app promete.
>
> La regla que reemplaza al "nunca escribe" no es "ahora escribe". Es que **quién declara cada registro es un dato del registro** (`cargado_por`) y no se pierde nunca. El veterinario sigue siendo la fuente de verdad: lo que carga el tutor es información declarada, se lee como contexto, se muestra siempre marcada como tal, y el tutor sigue sin poder tocar una línea de lo que escribió un profesional.
>
> Lo que **no** se agrega es un proceso de verificación: un antecedente del tutor no queda pendiente de que alguien lo apruebe. Un estado así le impondría al veterinario una tarea de curaduría que nadie pidió, y dejaría en un limbo —ni dato ni ausencia de dato— justo la información que hay que leer en una urgencia. Se marca y se muestra; el profesional la pondera como pondera lo que le cuenta el dueño en el mostrador.

Alcance del MVP: **un paciente puede ser atendido por varias clínicas y visto por varios tutores**, y las dos cosas se resuelven con tablas de vínculo revocables. La mascota tiene siempre un único dueño —el tutor que la dio de alta, o aquel a cuyo nombre la dio de alta la clínica— y es él quien otorga y retira todo acceso.

> Esto reemplaza al criterio de la primera versión, que fijaba un paciente a una única clínica por medio de una FK y reservaba la multi-clínica para la Fase 2 (ver sección 6). Se adelantó porque la relación fija no solo impedía la segunda clínica: obligaba a que toda mascota naciera de una, y dejaba afuera al tutor que quiere tener el historial de su animal antes de pisar una veterinaria, y a la familia que comparte el cuidado del mismo animal.

## 2. Diagrama de relaciones

```
Tutor 1───N Paciente          (tutor_id: el dueño)
Tutor N───N Paciente          (Acceso de co-tutor: edición o lectura, revocable)
Clínica N───N Paciente        (Vínculo con clínica, revocable)

Invitación de co-tutor ──canje──> Acceso de co-tutor

Paciente
                │
                ├──N Consulta atendida  (el hecho asistencial)
                │         ├── cita_id (nullable: la agendada que vino a cumplir)
                │         └── veterinario_id (quién atendió)
                │
                ├──N Evento clínico ──N Adjuntos
                │         ├── usuario_id (autoría) + cargado_por (veterinario | tutor)
                │         └── consulta_id (nullable: en qué atención se escribió)
                │
                ├──N Medicación (activa si fecha_fin IS NULL)
                │         └── usuario_id (autoría) + cargado_por (veterinario | tutor)
                └──N Cita (calendario) ──N Notificación
                          └── clínica_id (quién la atiende)

Veterinario N───1 Clínica
Veterinario 1───N Ausencia del profesional  (cuándo no está disponible)

Clínica 1───N Franja de atención  (la grilla, por día de la semana)

Usuario N───1 Tutor  (si tipo_usuario = tutor)
Usuario N───1 Veterinario  (si tipo_usuario = veterinario)
Usuario N───1 Clínica  (si tipo_usuario = clínica_admin)
Usuario N───1 Clínica  (clínica_de_pertenencia_id: veterinario y clínica_admin)

Auditoría ──registra cambios de──> Evento clínico, Medicación, Cita, Consulta atendida,
                                   Paciente, Adjuntos, Vínculo con clínica, Acceso de co-tutor

Cita N───1 Veterinario  (veterinario_id: a quién le toca atender, opcional)

Evento de telemetría N───1 Usuario  (nullable: los que emite el sistema no tienen actor)
```

## 3. Campos transversales

Los siguientes tres campos están presentes en todas las entidades principales (Tutor, Paciente, Clínica, Veterinario, Usuario, Evento clínico, Medicación, Cita, Consulta atendida, Adjuntos, Ausencia del profesional):

| Campo | Tipo | Descripción |
|---|---|---|
| created_at | timestamp | Fecha y hora de creación del registro. Obligatorio. |
| updated_at | timestamp | Fecha y hora de la última modificación. Se actualiza en cada UPDATE. |
| deleted_at | timestamp (nullable) | NULL = registro activo. Con fecha = borrado lógico. Nunca se elimina físicamente. |

> El borrado lógico evita eliminar información clínica de forma irreversible. Las vistas activas del sistema filtran siempre por `deleted_at IS NULL`.
>
> `created_at` y `updated_at` se exponen en la API de todas las entidades. `deleted_at` **no**: un registro dado de baja simplemente no aparece en los listados, y devolver el campo en todas partes sugeriría que el cliente puede pedir los borrados, que no es el caso. La única excepción es **Paciente**, donde sí se expone — ver la nota de 4.2.

### 3.1 Dirección

Tutor y Clínica guardan la dirección con el mismo grupo de cuatro campos. No es un dato de texto suelto: es una dirección legible con su punto en el mapa al lado.

| Campo | Tipo | Descripción |
|---|---|---|
| dirección | string | La dirección tal como se muestra. Si salió de una sugerencia del mapa, es el texto normalizado que devolvió el proveedor; si la escribió una persona a mano, es lo que escribió. |
| dirección_place_id | string (nullable) | Identificador que el proveedor de mapas le asigna al lugar. Permite volver a consultarlo más adelante sin depender de que el texto siga escribiéndose igual. |
| dirección_lat | decimal (nullable) | Latitud del punto confirmado. |
| dirección_lng | decimal (nullable) | Longitud del punto confirmado. |

> **Confirmar en el mapa no es obligatorio.** El cliente ofrece autocompletado (Arquitectura, 3.6) y, cuando quien carga elige una de las sugerencias, los cuatro campos se guardan juntos. Cuando escribe una dirección que el proveedor no conoce —un barrio nuevo, un paraje rural, una casa sin altura— se guarda igual: `dirección` cargada y los otros tres en NULL. Exigir la confirmación convertiría una zona mal mapeada en una ficha imposible de completar, y en el móvil sin conexión dejaría la dirección sin poder editarse.
>
> La contrapartida asumida es que conviven dos calidades de dirección en el mismo padrón. Es explícito y consultable: la que tiene coordenadas es la que el matching geolocalizado de Fase 2 va a poder usar tal cual; la que no las tiene habrá que geocodificarla o volver a preguntarla en ese momento.
>
> **Los cuatro campos se escriben como un bloque.** Cambiar el texto sin cambiar el punto dejaría el pin apuntando a la casa anterior, que es peor que no tener pin (Reglas de Negocio, 2.6).
>
> Se guardan **lat/lng además del place_id**, y no solo el identificador. Resolver un place_id a un punto es una llamada a un servicio externo, y las consultas geográficas de Fase 2 no pueden depender de que ese servicio esté arriba ni de re-geocodificar el padrón entero cada vez. El place_id sirve para refrescar el dato cuando una calle se renombra; el punto, para consultarlo.
>
> **Veterinario no tiene dirección**, y no es un olvido: la dirección del veterinario es la de la clínica donde atiende, que ya vive en `Clínica`. Un domicilio particular del profesional no lo necesita ningún proceso del MVP, y guardarlo sería pedir un dato personal que nadie va a leer.

## 4. Entidades

### 4.1 Tutor

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| nombre | string | Nombre completo del tutor. |
| tipo_documento | enum (nullable) | DNI / Pasaporte u otro documento de identidad extranjero. NULL hasta que se complete la ficha. |
| número_documento | string (nullable) | Único junto con tipo_documento. DNI para residentes argentinos; pasaporte u otro documento equivalente para no residentes. NULL hasta que se complete la ficha. |
| contacto | string | Teléfono y/o email. |
| dirección + dirección_place_id + dirección_lat + dirección_lng | ver 3.1 | Domicilio del tutor, opcionalmente confirmado en el mapa. `dirección` es nullable acá: NULL hasta que se complete la ficha. Base del matching geolocalizado de Fase 2. |
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
| tutor_id | UUID / FK | El **dueño** de la mascota. Quien la dio de alta, o aquel a cuyo nombre la dio de alta la clínica. |
| identificador_externo | string (nullable) | Número de chip/microchip. Clave para portabilidad en Fase 2. |

> `peso_actual` se persiste como NUMERIC, no como punto flotante binario: el peso se guarda y se compara al gramo, y un `double` redondea de formas que en una historia clínica se notan. Debe ser mayor a cero.
>
> **Guardar al gramo no es mostrar al gramo.** La interfaz redondea a un decimal y no arrastra el cero final —`2,6 kg`, `12 kg`—, porque en un perro de 31 kg el gramo es ruido. La excepción, y lo que termina de justificar el NUMERIC, es **por debajo del kilo**: ahí se muestra el gramo, porque en una calopsita de 95 gramos un decimal es `0,1 kg` y el redondeo se come el dato entero.
>
> `identificador_externo` es único entre fichas vigentes cuando está cargado: dos mascotas no pueden compartir número de chip.
>
> La baja del Paciente es lógica y no cascadea (Reglas de Negocio, 4.5). `tutor_id` no es editable: cambiarlo sería transferir la mascota a otra persona sin dejar rastro de la transferencia. La transferencia de propiedad queda fuera de alcance; lo que sí existe es dar acceso a otro tutor, que es otra cosa y se modela aparte (4.14).
>
> **La clínica no vive en esta tabla.** La llevaba en la primera versión, como FK fija desde el alta, y esa columna se eliminó: una mascota puede ser atendida por varias clínicas, por ninguna —la que el tutor todavía no compartió— y puede dejar de serlo. Todo eso es un vínculo con estado y no un campo de la ficha, y vive en 4.13. Dejarla como columna nullable habría sido peor que sacarla: una columna que ya no decide ni el alcance ni la agenda pero sigue estando es la que alguien va a volver a leer por error.
>
> **El dueño sí es un campo y no una fila de 4.13**, y la asimetría es deliberada: es lo que garantiza por construcción que toda mascota tenga exactamente una persona responsable. Modelarlo como un vínculo más obligaría a sostener con un índice único parcial algo que la columna `NOT NULL` ya sostiene sola.
>
> **`deleted_at` se expone en la API de Paciente**, a diferencia del resto de las entidades. Es la consecuencia directa de la regla 4.5: la ficha de una mascota dada de baja se sigue leyendo, con su historial, su medicación, sus citas y sus adjuntos completos — pero no admite escrituras nuevas. Sin el campo en la respuesta, el cliente no tiene forma de distinguir esa ficha de una vigente y le ofrecería al veterinario acciones que el backend va a rechazar. Es un dato de presentación, no un permiso: quién puede leerla lo sigue decidiendo el motor de permisos.

### 4.3 Clínica

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| nombre | string | — |
| dirección + dirección_place_id + dirección_lat + dirección_lng | ver 3.1 | Domicilio de la clínica, opcionalmente confirmado en el mapa. `dirección` es obligatoria: una clínica sin dirección no se puede visitar. |
| contacto | string | — |
| duración_turno_minutos | int | Largo del turno con el que se agenda. Junto con las Franjas de atención (4.18) define la grilla del calendario. |
| zona_horaria | string | Zona IANA en la que se interpretan el horario de atención y la hora de las Citas. Por defecto `America/Argentina/Buenos_Aires`. |

> Los campos de agenda entraron después de la primera versión de esta tabla, junto con la decisión de darle hora a la Cita (4.7): sin ellos, una cita con hora se puede agendar a cualquier minuto de cualquier momento del día, y el calendario deja de ser una grilla para pasar a ser una lista de instantes sueltos.
>
> **`hora_apertura` y `hora_cierre` ya no están acá**: se promovieron a la Franja de atención (4.18). Eran un único intervalo para toda la semana —la simplificación deliberada de la primera versión—, y esta tabla anticipaba que el día que hiciera falta un horario por día "este es el campo que se promueve a una tabla de franjas, no un campo al que se le agregan casos especiales". Eso es lo que pasó: la promoción es lo que permite el **día cerrado** (un día sin ninguna franja) y el **corte de mediodía** (dos franjas el mismo día), que con un intervalo único no se podían expresar de ninguna manera.
>
> `duración_turno_minutos` **se queda en la Clínica**, y no bajó a la franja. El turno es cuánto dura atender a un paciente en esa clínica, y eso no cambia porque sea martes a la mañana o jueves a la tarde. Es de la Clínica y no del Veterinario porque la agenda es de la clínica: no hay agenda por profesional (ver la nota de 4.7 sobre qué significa `veterinario_id` en la Cita). Debe ser mayor a cero y dividir de forma exacta **cada franja**, no el día entero — una franja que no cierra en un múltiplo del turno deja un hueco al final que no se puede agendar.
>
> `zona_horaria` entró después de que el cliente y el backend tuvieran la zona escrita a mano cada uno por su lado. Es un dato de la clínica y no una constante del sistema: `hora_apertura` dice "09:00" sin decir 09:00 de dónde, y esa pregunta la contesta la clínica, no el despliegue. Mientras estuvo hardcodeada, además, no había forma de que el cliente supiera qué zona usar salvo repetir la constante y confiar en que nadie la cambiara de un solo lado.
>
> Se guarda el **nombre IANA** (`America/Argentina/Buenos_Aires`) y no un desfasaje fijo: un desfasaje queda viejo cuando el país cambia de huso o adopta horario de verano, y convierte una decisión de política pública en un dato a corregir a mano en cada clínica.
>
> El valor por defecto cubre al piloto y a cualquier clínica argentina, que es el alcance del MVP. Que exista el campo es lo que hace que la segunda clínica, en otro huso, sea una fila más y no una migración.

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
| usuario_id | UUID / FK | Cuenta que cargó el evento (trazabilidad). Referencia a **Usuario** y no a Veterinario, porque escriben los dos roles. |
| cargado_por | enum | `veterinario` \| `tutor`. Quién declara el registro. NOT NULL, default `veterinario`. |
| tipo | enum | Consulta / vacuna / cirugía / control / urgencia / medicación / alergia. |
| fecha | date | — |
| fecha_precision | enum | `dia` \| `mes` \| `anio`. Con qué precisión se conoce `fecha`. NOT NULL, default `dia`. |
| descripción | text | Texto libre de quien lo carga. |
| diagnóstico | text (nullable) | — |
| campo_estructurado | JSON (nullable) | Datos estructurados para tipos críticos (ver nota). |
| consulta_id | UUID / FK (nullable) | Consulta atendida (4.16) en la que se escribió. NULL cuando no hay asiento — una carga histórica, o una atención que nadie asentó. |

> **`veterinario_id` se llamaba así y era una FK a Veterinario**; pasó a `usuario_id` cuando el tutor empezó a cargar antecedentes (sección 1). Un registro con `cargado_por = tutor` no tiene veterinario, y dejar el campo nullable habría dejado sin autoría justo a las filas que más necesitan decir de quién son. La forma no es nueva en el modelo: es la misma que `Adjuntos.subido_por_usuario_id` (4.8), que ya resolvía este problema porque los adjuntos los suben los dos roles. Sigue siendo **autoría y no permiso**: no se reasigna nunca, y de él cuelgan la regla de baja (Reglas de Negocio, 2.4) y el hecho de que la baja de un Veterinario no borre lo que escribió (Reglas de Negocio, 4.13).
>
> Quién es el veterinario y de qué clínica se sigue leyendo entero, por el Usuario. Lo que se perdió es poder llegar al Veterinario con un JOIN directo, y es el precio de que la tabla admita dos tipos de autor.

> **`cargado_por` no es lo mismo que mirar el tipo de usuario del autor**, aunque hoy coincidan. Es la afirmación de qué clase de registro es esto —un acto médico documentado o un antecedente declarado por el dueño— y esa afirmación se congela al escribir. Es lo que hace que el marcado de la vista de urgencia no dependa de ir a buscar la cuenta del autor, que puede haberse dado de baja.
>
> **No es editable.** Ni el tutor ni el veterinario cambian el origen de un registro ya cargado: un antecedente declarado no se convierte en acto médico porque alguien toque un campo. Corregir un origen mal cargado es dar de baja el registro y escribirlo de nuevo.
>
> El nombre no es `origen` a propósito: `origen` ya significa otras dos cosas en este modelo —el de Consulta atendida (4.16) y el del Vínculo con clínica (4.13)—, y un tercero con un significado distinto en las dos tablas centrales del historial sería ambigüedad gratuita.

> **`fecha_precision` existe porque el tutor rara vez tiene el día exacto.** Una libreta vieja dice "2023" o "en marzo", y obligar a inventar un día convierte un dato cierto en uno falso con apariencia de preciso. La fecha se sigue guardando como un `DATE` completo, rellenando con `01` lo que no se sabe —"fue en 2023" se guarda `2023-01-01` con `fecha_precision = anio`—, y la interfaz muestra solo lo que el campo dice que se conoce.
>
> Se eligió esto por sobre partir la fecha en componentes nullables (año, mes, día por separado). Un `DATE` con un enum al lado se sigue ordenando, filtrando por rango e indexando con las mismas consultas que ya existen; con componentes sueltos, todo filtro por período y la vista de urgencia habría que reescribirlos. El costo asumido es que el `DATE` guardado no es exactamente lo que la persona declaró, y por eso **nunca se muestra sin leer la precisión al lado**.
>
> Para `cargado_por = veterinario` siempre es `dia`, y se valida (Reglas de Negocio, 2.2): el profesional carga en el momento, con la fecha delante.

> **`cita_id` ya no existe en esta entidad**: la reemplazó `consulta_id`. La FK a Cita vivía acá desde antes de que existiera la Consulta atendida, y sostenía la regla "una cita se cumple con un solo evento" con un índice único. Esa regla dejó de ser cierta: de una misma atención cuelgan varios eventos —se pone una vacuna y se registra una alergia—, y con las dos FK al lado había que elegir arbitrariamente cuál de los eventos se quedaba con la cita. La cadena es una sola y no se duplica: **Cita** (lo que se planeó) → **Consulta atendida** (lo que ocurrió) → **Evento clínico** (lo que se escribió). Qué cita cumplió un evento se sigue leyendo entero, por su consulta.
>
> La API sí recibe `cita_id` al cargar un evento, y no es una contradicción: es un atajo de entrada. Declarar qué cita cumple esta carga asienta la Consulta atendida de esa cita si todavía no estaba, y el evento queda vinculado a ella (Reglas de Negocio, 4.21). Lo que se persiste es la consulta.

> Una atención espontánea tiene consulta y no tiene cita; una carga histórica no tiene ninguna de las dos, y por eso `consulta_id` es nullable.
>
> Varios eventos pueden colgar de la misma consulta: en una atención se pone una vacuna y se registra una alergia, y son dos eventos de un solo hecho asistencial. Por eso la FK vive del lado del Evento clínico, que es el lado que puede estar vacío.
>
> Los eventos con `consulta_id` NULL no son un error a corregir: son la medida de cuánto se está escribiendo sin asentar, que es la lectura inversa de la cobertura.

> Se recomienda estructurar como campos fijos (no texto libre) al menos: vacunas, medicación activa y alergias — son los datos que un veterinario ajeno necesita leer en segundos durante una urgencia.

El esquema de `campo_estructurado` es fijo y lo valida el backend según el `tipo` del evento. No es un JSON libre: si no cumple la forma de su tipo, la operación se rechaza antes de escribir (regla 2.2).

Las **claves son las mismas para los dos orígenes**; lo que cambia es qué es obligatorio. Un antecedente del tutor no puede exigir los campos que solo tiene el profesional que atendió.

| tipo | Con `cargado_por = veterinario` | Con `cargado_por = tutor` |
|---|---|---|
| vacuna | `{ nombre_vacuna: string (requerido), lote: string (requerido), fecha_proxima_dosis: date (opcional) }` | Las mismas claves, con **`lote` opcional**. |
| medicación | `{ nombre_droga: string (requerido), dosis: string (requerido), frecuencia: string (requerido) }` | Las mismas claves, con **`dosis` y `frecuencia` opcionales**. |
| alergia | `{ alergeno: string (requerido), severidad: enum leve/moderada/severa (requerido), reaccion: string (opcional) }`. `fecha` es la fecha de detección. | Las mismas claves, con **`severidad` opcional**. |
| consulta, cirugía, control, urgencia | Debe ser NULL. Enviar un valor se rechaza. | Igual: NULL. La descripción libre y la fecha alcanzan. |

> **Qué se afloja y por qué, uno por uno.** El **lote** está impreso en el frasco y lo copia quien vacuna; el dueño tiene la libreta con el nombre de la vacuna y ningún número. La **severidad** de una alergia es un juicio clínico graduado, y pedirle al tutor que elija entre leve, moderada y severa es pedirle que emita ese juicio: si no la sabe, el campo queda vacío y el veterinario la gradúa cuando la vea. La **dosis** y la **frecuencia** de lo que el animal está tomando el dueño las conoce a veces con precisión y a veces como "media pastilla a la mañana" — el campo es texto libre justamente para eso, pero exigirlo dejaría afuera al que solo sabe la droga.
>
> Lo que **no** se afloja es la clave que identifica de qué se está hablando: `nombre_vacuna`, `alergeno` y `nombre_droga` siguen siendo obligatorios en los dos orígenes. Un antecedente que no dice de qué es no es un antecedente.
>
> **`fecha_proxima_dosis` es el campo que más rinde en la carga del tutor**, aunque sea opcional: es lo que está escrito en la libreta al lado de la vacuna, y es lo que permite ofrecerle agendar el turno en vez de esperar a que se acuerde.

> **No hace falta un tipo "evento general"** para lo que el tutor quiera contar y no encaje en vacuna, alergia o medicación —una cirugía vieja, una consulta en otra veterinaria, un diagnóstico—. Eso ya es un evento de `tipo = consulta`, `cirugía`, `control` o `urgencia`, que son exactamente los tipos que no llevan campo estructurado: descripción libre y fecha. Agregar un tipo más habría duplicado los que ya existen con otro nombre.

> El campo se valida contra un esquema fijo en vez de aceptarse como JSON libre porque el motivo entero de tenerlo es la lectura en urgencia: un dato que cada clínica escribe con las claves que quiera no es consultable, y equivale a texto libre en otro envoltorio. La contrapartida asumida es que agregar un tipo estructurado nuevo pide tocar el backend, no solo cargar datos.
>
> Las claves de `tipo = medicación` replican los campos tipados de la entidad Medicación (4.6) a propósito: el Evento clínico registra **que se indicó** una medicación en una fecha (hecho histórico, inmutable), mientras que la entidad Medicación lleva el **tratamiento vigente** y su ciclo de vida (`fecha_fin`, regla de una activa por droga). Son dos preguntas distintas y la duplicación es el precio de poder responder ambas.
>
> Las alergias se cubren como un `tipo` más de Evento clínico, no como entidad propia. La alternativa era una tabla espejo de Medicación (4.6), que modela mejor un estado vigente que un hecho histórico; se descartó para el MVP porque duplicaría alcance, auditoría y baja lógica que el Evento clínico ya resuelve. La consecuencia asumida es que una alergia no tiene estado *vigente / superada*: solo está registrada o dada de baja (`deleted_at`), y descartar un diagnóstico errado es darla de baja. Si aparece la necesidad de distinguir una alergia superada de una mal diagnosticada, ese es el punto donde conviene promoverla a entidad.
>
> La vista de urgencia de un paciente se arma con sus Eventos clínicos de `tipo = alergia` sin `deleted_at`, junto con sus Medicaciones activas (`fecha_fin` NULL) y sus vacunas.

> **Lo que declaró el tutor entra en la vista de urgencia, y se marca de forma imposible de pasar por alto.** No se filtra: una alergia que el dueño conoce y ninguna clínica registró es exactamente el dato que salva a un animal en una urgencia, y esconderla por no venir de un profesional sería tirar el motivo entero de esta pantalla.
>
> Pero un veterinario ajeno que la lee en treinta segundos tiene que saber, sin buscarlo, qué está mirando. La marca es del **contrato, no del frontend**: la API devuelve `cargado_por` en cada registro de la vista, y las dos plataformas están obligadas a distinguir los dos orígenes con algo que se lea de un vistazo —no un icono chico ni una diferencia de color sola (Design System)—, y a nombrar al autor: "lo declaró el tutor", no "sin verificar". El registro no está mal: está declarado por otra persona.
>
> Los dos grupos **se muestran juntos y no en dos listas separadas**. Partir la vista obligaría a leer dos veces para responder una sola pregunta —"¿a qué es alérgico este animal?"— y la segunda lista es la que se saltea el que va apurado, que es el caso para el que la pantalla existe.

### 4.6 Medicación

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | — |
| usuario_id | UUID / FK | Cuenta que cargó el tratamiento. Referencia a **Usuario** y no a Veterinario, por el mismo motivo que en 4.5. No se reasigna al cerrarlo ni al corregirlo. |
| cargado_por | enum | `veterinario` \| `tutor`. NOT NULL, default `veterinario`. Con las mismas reglas que en 4.5: no es editable y se congela al escribir. |
| nombre_droga | string | — |
| dosis | string (nullable) | Obligatoria con `cargado_por = veterinario`; opcional cuando la declara el tutor. |
| frecuencia | string (nullable) | Misma regla que `dosis`. |
| fecha_inicio | date | — |
| fecha_precision | enum | `dia` \| `mes` \| `anio`. Aplica a `fecha_inicio`, con el mismo criterio que en 4.5. NOT NULL, default `dia`. |
| fecha_fin | date (nullable) | NULL indica medicación activa. Filtro clave para vista de urgencia. |

> La autoría no figuraba en la primera versión de esta tabla, pero las Reglas de Negocio ya la exigían en dos lugares: la baja lógica está reservada al autor del registro (regla 2.4) y la baja de un Veterinario "no cascadea sobre lo que ese veterinario escribió: los Eventos clínicos y las Medicaciones conservan su autoría" (proceso 4.13). Sin el campo, ninguna de las dos reglas era evaluable. Entró como `veterinario_id` y pasó a `usuario_id` al abrirse la carga del tutor, por el motivo explicado en 4.5.

> **La medicación que declara el tutor es la que el animal está tomando ahora**, y por eso nace activa (`fecha_fin` NULL). Un tratamiento que ya terminó no es medicación vigente: es un hecho del pasado y se carga como Evento clínico de `tipo = medicación`, que es la distinción que estas dos entidades ya sostenían (4.5).
>
> La regla de **una activa por droga** (Reglas de Negocio, 2.2) se evalúa sobre las activas del paciente **sin mirar el origen**, y por lo tanto una medicación declarada por el tutor bloquea la carga de la misma droga por el veterinario. Es lo correcto: dos filas activas de la misma droga son exactamente la ambigüedad que la regla existe para evitar, y no deja de serlo porque una la haya escrito el dueño. Lo que el veterinario hace en ese caso es cerrar la declarada y abrir la suya —el mismo gesto con el que corrige una dosis (4.3)—, y el rechazo tiene que decirle que la activa que le estorba la cargó el tutor, o va a parecer un error del sistema.

> `fecha_fin` registra el **cierre efectivo** del tratamiento, no su duración planificada: una fecha futura dejaría de contar como activa a una medicación que el paciente todavía está recibiendo, y "activa" está definido acá como `fecha_fin IS NULL`. Prescribir "por 7 días" se registra hoy en `frecuencia` y se cierra el día que termina. Modelar la duración planificada exigiría separar *fin previsto* de *fin efectivo*, y queda fuera del MVP.

### 4.7 Cita (calendario)

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | — |
| clínica_id | UUID / FK | La clínica que atiende esta cita. Fija desde el alta. |
| tipo | enum | Próxima vacuna / control / cirugía programada. |
| fecha_programada | timestamp | Momento de la cita, con hora. Cae dentro de alguna Franja de atención de la clínica (4.18) y sobre la grilla de turnos que esa franja define con `duración_turno_minutos` (4.3). |
| veterinario_id | UUID / FK (nullable) | Profesional que va a atender. NULL cuando la cita es de la clínica y todavía no se repartió. |
| estado | enum | Pendiente / cumplido / vencido. |
| notificar_tutor | boolean | Dispara notificación al tutor. |

> `estado` no es un campo que el cliente escriba: nace en `pendiente`, pasa a `cumplido` cuando se asienta la **Consulta atendida** que la referencia (4.16) y a `vencido` por el job programado (Reglas de Negocio, 4.6). No hay endpoint que lo reciba — exponerlo dejaría marcar como cumplida una cita que nadie atendió.
>
> El disparador era antes la carga de un Evento clínico que la referenciaba por una FK propia, y cambió al aparecer la Consulta atendida — que además se quedó con esa FK (4.5). El motivo es que cumplir una cita es **haber atendido**, no haber escrito: con el disparador viejo, la cita de un paciente atendido el martes y documentado el jueves figuraba vencida dos días. El comportamiento no se pierde, porque cargar ese evento declarando su cita asienta la consulta (Reglas de Negocio, 4.21) y la transición ocurre igual.

> `fecha_programada` era una fecha sin hora en la primera versión de esta tabla. Pasó a timestamp porque una agenda de veterinaria sin hora no es una agenda: dos cirugías el mismo martes no son intercambiables, y el calendario no podía mostrar más que "hay algo ese día". El costo asumido es que ahora hay una zona horaria en juego — se persiste en UTC y se presenta en la `zona_horaria` de la clínica que atiende a la mascota (4.3).
>
> `veterinario_id` entró después de la primera versión de esta tabla, y no significa lo mismo que el `usuario_id` de Evento clínico o Medicación. Allá el campo es **autoría** —quién escribió el registro, dato que no se reasigna nunca—; acá es **asignación**: a quién le toca atender, y se puede cambiar mientras la cita siga pendiente. Quién la programó y quién la reagendó sigue estando en Auditoría, que es donde se lo consulta.
>
> Es **nullable a propósito**. Una cita puede nacer "de la clínica" y repartirse después, y una clínica chica que no divide agenda no tiene por qué elegir profesional en cada turno. Las citas ya cargadas cuando se agregó el campo quedaron sin asignar, que es lo que efectivamente eran: inventarles un profesional habría falseado el dato.
>
> Un veterinario **no puede tener dos citas pendientes en el mismo momento** (Reglas de Negocio, 2.2), y eso es lo que hace que la asignación signifique algo. Las citas sin asignar no colisionan entre sí: sin profesional no hay a quién solapar.
>
> El **alcance no cambia**: se sigue resolviendo contra la mascota. Que a un veterinario le toque una cita no le da acceso a esa mascota, ni que no le toque se lo quita — cualquiera del plantel atiende a los pacientes que su clínica tiene vinculados. La asignación es organización del trabajo, no un permiso.
>
> **`clínica_id` entró cuando el paciente dejó de tener clínica propia** (4.2), y no es una denormalización: es el dato que se perdió. Todas las reglas de agenda —el horario de atención, la `zona_horaria` en la que se lee la hora, la grilla de turnos y "el profesional atiende en esa clínica" (Reglas de Negocio, 2.2)— hablaban de "la clínica del paciente", y con una mascota atendida en tres esa expresión dejó de tener referente único: las 09:00 son válidas o no según de cuál se hable. La cita dice en cuál se agenda, y con eso las cuatro reglas vuelven a significar lo mismo que significaban.
>
> Se resuelve contra el actor y **no se recibe por la API**: una cita nace en la clínica del veterinario que la agenda. Recibirla del cliente sería dejar agendar en la agenda de otro.
>
> **Es fija.** Mudar una cita de clínica le cambiaría la grilla, el huso horario y la validez del profesional asignado, los tres a la vez: eso no es editar una cita, es dar de baja una y agendar otra. La clínica tiene que tener vínculo vigente con el paciente al momento del alta.

### 4.8 Adjuntos

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | Mascota a la que pertenece el archivo. Siempre presente. |
| evento_id | UUID / FK (nullable) | Completo cuando el adjunto documenta un evento clínico concreto; NULL cuando es un adjunto general del paciente (ej. ficha histórica escaneada). |
| subido_por_usuario_id | UUID / FK | Cuenta que subió el archivo. Referencia a Usuario y no a Veterinario o Tutor, porque suben los dos roles. |
| tipo | enum | Foto / PDF / estudio. |
| clave_de_archivo | string | Ruta del objeto dentro del bucket. Es opaca para el cliente y no se expone en la API. Guarda **el archivo original, tal como se subió**. |
| clave_de_vista_previa | string (nullable) | Ruta de un JPEG derivado, para los formatos que el navegador no sabe mostrar. NULL cuando el original ya se muestra —que es el caso de casi todos los adjuntos. Tampoco se expone en la API. |
| nombre_archivo | string | Nombre original con el que se subió, para que la descarga no se llame como la clave. |
| content_type | string | Tipo MIME de **lo que sirve la URL de descarga**, que es la vista previa cuando existe. |
| tamano_bytes | bigint | Tamaño del **original**, validado contra el máximo permitido antes de escribir. |
| es_foto_perfil | boolean | Marca al adjunto que la aplicación muestra como foto de la mascota. `NOT NULL`, default `false`. Como máximo uno vigente por Paciente (Reglas de Negocio, 4.14). |

> `paciente_id` pasó a ser obligatorio y `evento_id` quedó como el único nullable de los dos. La primera versión los tenía a ambos nullable y mutuamente excluyentes; con esa forma, resolver el alcance de un adjunto colgado de un evento exigía ir a buscar el evento para llegar a la mascota, y nada impedía una fila con las dos FK vacías. Todo adjunto pertenece a una mascota; que además documente un evento es información adicional.

> `subido_por_usuario_id` no figuraba en la primera versión de esta tabla. Lo exige la regla de baja (Reglas de Negocio, 2.4): cada rol retira los adjuntos que subió, y sin el campo esa regla no es evaluable.

> **La foto de perfil es un adjunto marcado y no un campo del Paciente.** Un `foto_url` colgado de la mascota habría sido una segunda ruta de subida —con su propia validación de MIME, su propio techo de tamaño y su propio objeto en el bucket— para un archivo que es exactamente lo mismo que ya sube el proceso de Adjuntos. Con el flag, la foto entra por el camino de siempre y hereda el alcance, la Auditoría y la baja lógica de cualquier adjunto; lo único que agrega es cuál de todos se muestra.
>
> **La unicidad la sostiene la capa de negocio**: marcar una foto desmarca a la anterior en la misma operación (Reglas de Negocio, 4.14). Un índice único parcial sobre `paciente_id` para las filas marcadas y vigentes es igual de válido como red de seguridad, y queda a criterio de la implementación — el contrato es "como máximo una", no cómo se hace cumplir.
>
> **La lectura de la foto no pasa por Adjuntos.** Toda lectura de Paciente lleva un `foto_perfil_url` —la URL prefirmada de la foto marcada, o null si la mascota no tiene—, y lo mismo la fila de agenda que ya viaja con el nombre y la especie de la mascota. No es una columna: se resuelve por pedido, con el mismo criterio que `nivel_de_acceso` (4.14). Sin eso, una lista de mascotas necesita un pedido de adjuntos por fila para dibujar el avatar de cada una, que es exactamente lo que la proyección de agenda evita para el nombre.
>
> **No se persiste ni se replica**: es una URL prefirmada de vida corta, igual que el `archivo_url` del adjunto, así que no viaja en el delta de sincronización (Sincronización sin Conexión, 2) y sin conexión el avatar vuelve al ícono de la especie.

> `archivo_url` se reemplazó por `clave_de_archivo`. El bucket es privado: no existe una URL estable que guardar. La API expone en cada lectura una URL prefirmada de vida corta, que se calcula en el momento y no se persiste — guardar una URL sería guardar un permiso vencido (Arquitectura, 3.4).

**Límite y formatos.** El techo por archivo son **10 MiB** (10485760 bytes). El número está declarado en el contrato, en el `maxLength` de `SubirAdjuntoRequest.archivo`, y no solo en el código: la interfaz tiene que mostrar el límite **antes** de que el usuario elija el archivo, y para eso necesita leerlo de algún lado — descubrirlo con un 413 después de haber subido por red móvil es el viaje perdido que la interfaz puede evitar. El backend lo aplica igual.

Los formatos admitidos dependen del `tipo` declarado: **foto** acepta cualquier imagen, **pdf** solo `application/pdf`, y **estudio** los dos, porque puede ser una placa o el informe que la acompaña.

> El tipo MIME **se determina leyendo el contenido**, nunca la extensión ni lo que declare el cliente — el tipo declarado es un dato del cliente y el backend es la única barrera. Eso incluye a la familia **HEIF (HEIC/HEIF/AVIF)**, que es el formato con el que la cámara de un iPhone saca fotos por defecto: no la reconoce la detección estándar de la librería de Go, así que el backend la resuelve leyendo la marca de la caja `ftyp`. Sin eso, la foto de una herida sacada desde un iPhone quedaba rechazada por "no es una imagen".
>
> Se mira solo la marca **principal** y no las compatibles: el contenedor es el mismo que el de un mp4, y aceptar por marca compatible haría pasar un video por foto.

**Vista previa de los formatos que el navegador no muestra.** Aceptar un HEIC no alcanza: fuera de Safari, ningún navegador lo dibuja, así que el veterinario que abría la foto desde la web se encontraba con una descarga. Al subir un HEIC o un HEIF, el backend **deriva un JPEG equivalente y guarda los dos objetos**: `clave_de_archivo` sigue apuntando al original intacto y `clave_de_vista_previa` al derivado. El `archivo_url` que devuelve la API —que sigue siendo uno solo— firma la vista previa cuando existe.

> **El original no se pisa ni se descarta.** Convertir y tirar el archivo que subió el usuario sería destruir material clínico, y en este sistema ni siquiera la baja de un adjunto borra el objeto del bucket (Reglas de Negocio, 2.4). Guardar los dos cuesta almacenamiento y deja el original disponible para exponerlo más adelante sin haberlo perdido.
>
> **AVIF queda afuera a propósito**: es de la misma familia y tampoco lo reconoce la detección estándar, pero lo muestran Chrome, Firefox y Safari. Derivarle una copia sería duplicar el bucket sin que nadie la mire.
>
> **Si la conversión falla, el archivo se sube igual**, sin vista previa: quedarse sin la copia mostrable es no poder mirar el archivo en el navegador, y abortar la subida es quedarse sin el archivo. Lo segundo es peor.
>
> La conversión **no redimensiona ni recorta** — la vista previa se mira para decidir algo clínico, y achicarla tiraría justo el detalle por el que se abre la imagen. Lo único que cambia es el formato.

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
> Consulta atendida se audita con el mismo criterio que el Evento clínico: es una afirmación asistencial sobre un paciente, y darla de baja o corregirle la fecha tiene que dejar rastro.

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

### 4.13 Vínculo con clínica

Qué clínicas atienden a una mascota. Es lo que reemplaza a la vieja FK `Paciente.clínica_id`.

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | La mascota. |
| clínica_id | UUID / FK | La clínica que la atiende. |
| origen | enum | Alta de la clínica / compartido por el tutor / migración. |
| otorgado_por_usuario_id | UUID / FK (nullable) | Quién lo otorgó. |
| otorgado_at | timestamp | Desde cuándo la atiende. |
| revocado_at | timestamp (nullable) | NULL = vigente. |
| revocado_por_usuario_id | UUID / FK (nullable) | Quién lo revocó. |

> **Revocar no es un borrado lógico**, y por eso el campo no se llama `deleted_at`. Que una clínica haya atendido a una mascota entre marzo y agosto es un hecho del negocio que hay que poder leer después: la fila revocada se conserva entera y no se toca nunca más. Es la misma familia de decisiones que "nunca borrado físico" (Reglas de Negocio, 2.4), aplicada a un vínculo en vez de a un registro clínico.
>
> Una clínica no se vincula dos veces a la misma mascota **entre los vínculos vigentes**. La unicidad es parcial y no total a propósito: revocar y volver a compartir tiene que poder dejar una fila nueva sin pisar el rastro de la anterior.
>
> `otorgado_por_usuario_id` es nullable **solo para `origen = migración`**: los vínculos que existían antes de que compartir fuera una acción no tienen autor real, e inventarle uno falsearía el dato. Es el mismo criterio con el que la Auditoría deja `usuario_id` en NULL para las acciones del sistema (4.10).
>
> **Revocar no toca el historial.** Los Eventos clínicos, la Medicación y los Adjuntos que esa clínica escribió quedan donde están, con su autoría: `usuario_id` identifica a quien los escribió y no depende de que el vínculo siga vivo. Lo que la clínica pierde es poder leer y escribir de ahí en adelante, y que la mascota figure en su cartera.

### 4.14 Acceso de co-tutor

Qué otras personas ven una mascota que no es suya. El dueño no aparece acá: el dueño es `Paciente.tutor_id`, y a él no le otorga acceso nadie.

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | La mascota. |
| tutor_id | UUID / FK | Quien recibe el acceso. |
| nivel | enum | Edición / lectura. |
| otorgado_por_usuario_id | UUID / FK | El dueño. |
| otorgado_at | timestamp | — |
| consentimiento_at | timestamp | Cuándo aceptó recibir estos datos. Obligatorio. |
| invitación_id | UUID / FK (nullable) | La invitación que se canjeó (4.15). |
| revocado_at | timestamp (nullable) | NULL = vigente. |
| revocado_por_usuario_id | UUID / FK (nullable) | — |

> **`edición` hace lo mismo que el dueño salvo administrar**: lee el historial completo, edita los datos no clínicos de la ficha, gestiona las citas y sube adjuntos, pero no invita, no revoca, no cambia niveles y no da de baja la mascota. `lectura` mira. La lista completa está en la matriz de la sección 5 y en Reglas de Negocio, 3.2.
>
> `consentimiento_at` es **obligatorio y se asienta al canjear la invitación**. Un co-tutor recibe datos de salud de un animal que no es suyo, y la Ley 25.326 exige rastro del otorgamiento: un acceso sin ese asiento no tendría con qué demostrarlo. No se revoca por la API, con el mismo criterio que el consentimiento del Tutor (4.1) — lo que corta el tratamiento es revocar el acceso, y borrar el asiento borraría la evidencia de que alguna vez se otorgó.
>
> **Un acceso nunca es el del propio dueño.** La base lo sostiene: la fila guarda además el `tutor_id` del dueño de esa mascota, para poder exigir que sea distinto del que recibe el acceso. Es redundante a propósito, porque una restricción de tabla no puede consultar otra tabla, y no puede quedar desincronizado porque `Paciente.tutor_id` es inmutable.

### 4.15 Invitación de co-tutor

Cómo se le da acceso a alguien que puede todavía no tener cuenta.

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | La mascota que se comparte. |
| invitado_por_usuario_id | UUID / FK | El dueño. |
| email | string | A quién va dirigida. Normalizado en minúsculas. |
| nivel | enum | Edición / lectura. |
| token_hash | string | Hash del código. Nunca el código. |
| expira_at | timestamp | — |
| usado_at | timestamp (nullable) | Cuándo se canjeó. |
| aceptado_por_usuario_id | UUID / FK (nullable) | Quién la canjeó. |
| revocado_at | timestamp (nullable) | Anulada por el dueño antes de usarse. |

> Es la misma forma que el token de activación (Reglas de Negocio, 4.16) y el de recuperación (4.4.1): un secreto de un solo uso, guardado hasheado, con vencimiento, que no se borra al usarse. Lo que cambia es que **no cuelga de un `usuario_id`** — el invitado puede no tener cuenta todavía, y ese es justamente el caso que esta entidad existe para resolver. La identidad del destinatario la lleva el email.
>
> **El canje exige que la cuenta que lo ejecuta tenga ese email.** Sin esa verificación, un enlace reenviado lo canjea cualquiera y la invitación deja de ser dirigida.
>
> Hay **una sola invitación pendiente por mascota y destinatario**: reinvitar al mismo correo anula la anterior, en vez de dejar dos enlaces vivos. Mismo criterio que la recuperación de contraseña.
>
> Vence a los **7 días**, mucho más que la activación y la recuperación, que se miden en horas. La diferencia es deliberada: esos dos son credenciales de una cuenta que ya existe y que su titular está mirando en ese preciso momento; este viaja entre dos personas, y quien lo recibe puede tener que crearse una cuenta antes de poder canjearlo.

### 4.16 Consulta atendida

Que la clínica atendió a esta mascota este día. Es el hecho asistencial, separado de lo que se haya escrito después sobre él.

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| paciente_id | UUID / FK | A quién se atendió. |
| clínica_id | UUID / FK | Dónde. Fija desde el alta, como en Cita (4.7). |
| veterinario_id | UUID / FK | Quién atendió. Obligatorio: una atención sin profesional no es un hecho asistencial. |
| cita_id | UUID / FK (nullable) | La cita que esta atención vino a cumplir. NULL cuando nadie la agendó. Única: una cita no se atiende dos veces. |
| origen | enum | Agendada / espontánea / urgencia. |
| fecha_hora | timestamp | Cuándo se atendió. Se persiste en UTC y se lee en la `zona_horaria` de la clínica, igual que la Cita. |
| registrada_por_usuario_id | UUID / FK | Quién la asentó. Puede no ser quien atendió: un veterinario asienta la del colega que ya se fue. |
| asentada_automáticamente | boolean | True cuando la creó el sistema al cargarse un Evento clínico que declara la cita que cumple, y no un profesional al atender. Es lo que permite separar lo asentado de lo deducido al leer las métricas. |

> **Por qué es una entidad y no un estado de la Cita.** La mayoría de las atenciones de una veterinaria no estaban agendadas: entra alguien con el perro que se cortó. Colgar el hecho de la Cita dejaría afuera justo al caso más frecuente, y obligaría a inventar una cita retroactiva para poder registrar que se atendió — una agenda que se completa hacia atrás deja de servir como agenda.

> **Por qué no alcanza con el Evento clínico.** El Evento clínico es lo que el veterinario *escribió*; esto es lo que *pasó*. Son la misma cosa solo si se escribe siempre, que es precisamente lo que hay que medir: la cobertura del piloto es cuántas de las atenciones asentadas terminaron con historial cargado (Telemetría de Producto, 9). Derivar el denominador del numerador no mide nada.

> **`asentada_automáticamente` existe para no mentir la métrica.** Cargar un Evento clínico declarando la cita que cumple asienta la consulta si no estaba (Reglas de Negocio, 4.21), porque una atención documentada es una atención que ocurrió y perder ese hecho sería absurdo. Pero esas filas no informan nada nuevo: entraron por el mismo camino que el numerador. La cobertura se lee sobre las asentadas por una persona, y el resto se cuenta aparte.

> **La asienta el veterinario, no la recepción.** Sería más barato que la asiente quien recibe al paciente, pero eso exigiría darle al clínica_admin la información de qué mascota fue atendida y cuándo, que es exactamente lo que la matriz de la sección 5 le niega. El asiento vale un toque desde la agenda o desde la ficha, y esa es toda la fricción que puede tener para que se use.

> **Es dato asistencial: se audita y su baja es lógica**, como el Evento clínico. Un asiento cargado por error se da de baja, no se borra: "esta mascota fue atendida el martes" es afirmación que dejó de ser cierta, y por qué dejó de serlo tiene que quedar en el rastro.

> **No la escribe el tutor ni la lee como tal.** El tutor ve el historial, que es lo que le sirve; una lista de "fuiste atendido" sin nada escrito al lado solo comunicaría que la clínica no cargó, y eso es un problema entre nosotros y la clínica, no un dato para el dueño de la mascota.

### 4.17 Evento de telemetría

Qué hace la gente con el producto. Es el registro que sostiene las métricas del piloto (Telemetría de Producto).

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| nombre | enum | Qué pasó. El catálogo cerrado está en Telemetría de Producto, sección 5. Un nombre fuera del enum se descarta. |
| usuario_id | UUID / FK (nullable) | Quién. NULL solo para los eventos que emite un proceso del backend sin usuario detrás (ej. el despacho de una notificación). |
| rol | enum (nullable) | Veterinario / tutor / clínica_admin / sistema. Sale del token, nunca del cuerpo del request. |
| clínica_id | UUID / FK (nullable) | Clínica de pertenencia del actor, si tiene. Es el corte por cuenta B2B. |
| plataforma | enum | Web / iOS / Android. |
| app_versión | string (nullable) | Versión del cliente que lo emitió. Es lo que permite leer una serie que cambia después de un despliegue. |
| sesión_id | UUID (nullable) | Agrupa los eventos de un mismo uso de la app. Lo genera el cliente al abrir y lo descarta al cerrar. No es el token ni deriva de él. |
| propiedades | JSON | Claves permitidas por evento, declaradas en el catálogo. Las demás se descartan al recibirlas. |
| ocurrido_at | timestamp | Cuándo pasó, según el reloj de quien lo emitió. |
| registrado_at | timestamp | Cuándo lo recibió el servidor. |
| reloj_sospechoso | bool | Marca los eventos cuyo `ocurrido_at` no es creíble contra `registrado_at`. Se guardan igual, pero quedan fuera de las series. |

> **No es la Auditoría y no la reemplaza.** La Auditoría (4.10) registra qué cambió un dato clínico y quién, para responder ante un reclamo: es prueba, se conserva, y no lleva nada que no sea el cambio. Esto registra cuánto tardó, desde qué pantalla y cuántas veces se abandonó a la mitad: es producto, se agrega y se borra a los 13 meses. Tienen dos propósitos, dos ciclos de vida y dos plazos de retención distintos.

> **Ninguna fila lleva dato clínico, texto libre ni `paciente_id`** (Reglas de Negocio, 2.7). El `propiedades` con claves permitidas por evento es lo que hace cumplible la regla: un JSON abierto es por dónde termina entrando el nombre de la mascota.

> Los dos relojes existen porque el tutor genera eventos sin conexión y los sube horas después (Sincronización Offline, 4.1). Toda métrica se calcula sobre `ocurrido_at`.

> No lleva `deleted_at` ni se audita, con el mismo criterio que Notificación (4.12): es un registro operativo, nadie lo edita nunca y su única baja es la del plazo de retención — que es física, y es la única excepción del proyecto al "nunca borrado físico" (Telemetría de Producto, 8).

### 4.18 Franja de atención

Cuándo atiende la clínica. Es lo que, junto con `duración_turno_minutos` (4.3), produce la grilla sobre la que se agenda.

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| clínica_id | UUID / FK | La clínica que atiende en esta franja. |
| día_semana | int (0–6) | Día al que aplica. 0 = lunes. |
| hora_desde | time | Hora a la que empieza a atender. |
| hora_hasta | time | Hora a la que deja de atender. Posterior a `hora_desde`. |

> **Un día sin ninguna franja es un día cerrado.** No hay un campo `cerrado`: la ausencia de filas ya lo dice, y un booleano que pudiera contradecir a las franjas del mismo día sería un dato que hay que mantener coherente a mano. Es también la única forma de expresar que la clínica no abre el domingo, que con el intervalo único de la primera versión era imposible.

> **Dos franjas el mismo día son el corte de mediodía.** No se solapan entre sí ni se tocan: dos franjas contiguas —09:00–13:00 y 13:00–18:00— son una sola franja escrita en dos filas, y se rechazan al guardar. Lo que hace que dos filas signifiquen algo distinto de una es el hueco entre ellas.

> **Se escriben como un conjunto, no de a una.** Guardar el horario reemplaza las franjas de la clínica entera en una transacción: la grilla es una sola cosa y hay que poder validarla completa —que ninguna se solape, que `duración_turno_minutos` divida a cada una— antes de aceptarla. Una API que agregue y borre franjas sueltas deja pasar estados intermedios inválidos, y obliga a decidir qué hacer con las Citas en cada paso en vez de una vez.

> **No lleva `deleted_at` ni se audita.** Es configuración de la clínica, no un registro del dominio clínico: nadie consulta cuál era el horario en marzo, y su baja es que la próxima escritura del conjunto no la incluya. Lo que sí queda auditado es el efecto — achicar el horario no puede dejar Citas pendientes afuera (Reglas de Negocio, 2.2), así que no hay Citas silenciosamente huérfanas de grilla.

> **Las horas se leen en la `zona_horaria` de la clínica** (4.3), igual que la Cita y la Consulta atendida. `hora_desde` dice "09:00" sin decir 09:00 de dónde, y esa pregunta la sigue contestando la clínica.

> **La migración desde el intervalo único escribe siete franjas idénticas por clínica**, una por día, con la `hora_apertura` y la `hora_cierre` que esa clínica tenía. Es la lectura literal de lo que el modelo viejo afirmaba —atiende todos los días en ese horario— y deja la grilla y las Citas existentes exactamente donde estaban. Cerrar el domingo es una edición que la clínica hace después, no algo que la migración pueda adivinar por ella.

### 4.19 Ausencia del profesional

Cuándo un veterinario no está disponible para atender. Existe para que la grilla no ofrezca a quien no va a estar.

| Campo | Tipo | Descripción |
|---|---|---|
| id | UUID / PK | Identificador único. |
| veterinario_id | UUID / FK | Quién no está. |
| desde | timestamp | Comienzo de la ausencia. |
| hasta | timestamp | Fin de la ausencia. Posterior a `desde`. |
| registrada_por_usuario_id | UUID / FK | El clínica_admin que la cargó. |

> **No tiene campo `motivo`, ni libre ni enum.** "Licencia médica" sobre una persona identificada es dato de salud de un empleado, y la Ley 25.326 lo trata como dato sensible: guardarlo abriría una obligación que esta funcionalidad no necesita para nada. Para que la grilla no ofrezca a quien no está alcanza con el rango de fechas. El motivo lo sabe quien lo tiene que saber, por el canal por el que siempre lo supo.

> **Es un rango con hora, no un día.** Una ausencia de media tarde y una de dos semanas son la misma fila con distintos extremos. Un campo de fecha suelta obligaría a inventar un booleano de "día completo" y a decidir qué significa media jornada.

> **La carga la hace el clínica_admin**, sobre el plantel de su propia clínica (Reglas de Negocio, 3.2). El veterinario las lee —son su propia agenda— y no las escribe, con el mismo criterio por el que no se edita su ficha ni se da de alta: eso es administrar la clínica. Por `registrada_por_usuario_id` se sabe quién la cargó sin necesidad de una entrada de Auditoría.

> **Cargarla desasigna las Citas que caen adentro** (Reglas de Negocio, 4.22): quedan con `veterinario_id` en NULL, que es un estado que la Cita ya tenía (4.7), y aparecen en la lista de lo que hay que repartir. La ausencia se guarda siempre. Impedir registrar que alguien no vino a trabajar no hace que haya venido, y el día que hace falta cargarla rápido es justo el día en que nadie tiene tiempo de reasignar seis turnos primero.

> **No cascadea ni se propaga hacia atrás.** No toca las Consultas atendidas ya asentadas: si el sistema dice que esa persona atendió el martes, la ausencia cargada después no lo desmiente — lo que hay ahí es un error de carga en uno de los dos, y se corrige el que esté mal.

> **Su baja es lógica** (`deleted_at`), a diferencia de la Franja: una ausencia cargada por error tuvo un efecto sobre las Citas, y ese efecto sí quedó en la Auditoría de cada Cita desasignada. Dar de baja la ausencia **no reasigna nada**: las citas siguen sin profesional y se reparten como cualquier otra. Devolverlas automáticamente a quien las tenía sería adivinar que nadie las movió mientras tanto.

## 5. Matriz de permisos

| Entidad | Quién escribe | Quién lee |
|---|---|---|
| Evento clínico, Medicación | **Veterinario**, sobre los pacientes vinculados a su clínica: escribe con `cargado_por = veterinario` y cualquiera del plantel edita y da de baja, no solo el autor (Reglas de Negocio, 3.2). **Tutor dueño y co-tutor con edición**, solo antecedentes propios: escribe con `cargado_por = tutor` y edita o da de baja **únicamente los registros de ese origen**. Ningún rol cambia el `cargado_por` de un registro existente | Veterinario (todo) + el dueño y sus co-tutores, en cualquier nivel |
| Cita / Calendario | Veterinario y clínica_admin, en su propia clínica: agendan, reagendan y reparten. El dueño y el co-tutor con edición confirman y reagendan. **La baja es del veterinario y del tutor**, no del clínica_admin | Veterinario + el dueño y sus co-tutores + clínica_admin, sobre las citas de su propia clínica |
| Adjuntos | Veterinario + el dueño y el co-tutor con edición. Cada uno retira los que subió | Veterinario + el dueño y sus co-tutores |
| Dispositivo | Cada usuario los suyos (los registra al entrar y los da de baja al salir) | Cada usuario los suyos |
| Notificación | Nadie: las escribe el proceso que las encola y las despacha | Nadie por API en el MVP: llegan como push, no se listan |
| Paciente (datos básicos) | Alta: el dueño, o un veterinario **o el clínica_admin** a nombre de un tutor. Edición de los datos no clínicos: el dueño, el co-tutor con edición y el veterinario de una clínica vinculada. `identificador_externo` (chip): solo el veterinario. Baja: solo el dueño | Veterinario de una clínica vinculada + el dueño y sus co-tutores. Clínica_admin: **solo la proyección de la cartera** (ver la nota) |
| Vínculo con clínica | Otorga y revoca: solo el dueño. Un veterinario puede desvincular su propia clínica | El dueño, sus co-tutores y los veterinarios de las clínicas vinculadas. Clínica_admin, **solo agregados** de su propia clínica |
| Acceso de co-tutor, Invitación | Solo el dueño. El co-tutor puede renunciar al suyo | El dueño y sus co-tutores (los co-tutores, sin acciones) |
| Tutor (ficha) | Veterinario de una clínica vinculada a esa ficha (alta y edición, incluido completar documento y dirección); el propio tutor sobre la suya, salvo el consentimiento. **Clínica_admin: solo el alta**, con nombre, contacto y consentimiento | Búsqueda: cualquier veterinario, con la ficha completa; el clínica_admin, **solo la proyección del padrón**. Ficha concreta: veterinario vinculado + el propio tutor |
| Clínica (directorio) | Nadie por esta vía | Cualquier cuenta autenticada, con proyección reducida: es cómo el tutor elige con quién compartir |
| Evento clínico, Medicación (acceso de clínica_admin) | Sin acceso | Sin acceso — reservado a veterinario y tutor |
| Veterinario (ficha del plantel) | Clínica_admin, sobre el plantel de su propia clínica | Clínica_admin (edita) + Veterinario de la misma clínica (solo lectura). Tutor sin acceso |
| Usuario (cuentas de veterinario) | Clínica_admin, sobre las cuentas de su propia clínica | Clínica_admin (las de su clínica) + el propio usuario sobre su cuenta |
| Usuario (cuenta propia) | Cada usuario sobre su email y su contraseña | Cada usuario sobre su cuenta |
| Usuario (cuentas de tutor) | Solo el propio tutor (auto-registro) | Solo el propio tutor |
| Clínica (datos administrativos) | Alta: administrador de la plataforma, fuera de la API. Edición: clínica_admin sobre su propia clínica | Clínica_admin sobre su propia clínica. Veterinario de esa clínica, solo lectura |
| Franja de atención | Clínica_admin sobre su propia clínica, escribiendo el conjunto completo | Clínica_admin sobre su propia clínica. Veterinario de esa clínica, solo lectura: sin la grilla no puede saber qué horas son válidas |
| Ausencia del profesional | Clínica_admin, sobre el plantel de su propia clínica | Clínica_admin (las de su clínica) + Veterinario de esa clínica, solo lectura |
| Usuario (cuentas de clínica_admin) | Administrador de la plataforma, fuera de la API | Clínica_admin sobre su propia cuenta |
| Consulta atendida | Solo el veterinario, sobre los pacientes vinculados a su clínica: asienta, corrige y da de baja. El sistema, cuando la deduce de un Evento clínico con cita | Veterinario de la clínica que atendió. Clínica_admin, **solo agregados** de su propia clínica. Tutor y co-tutores sin acceso |
| Evento de telemetría | Cada usuario, solo los suyos, por la ruta de ingesta; y el backend, para lo que emite él | Nadie por API en el MVP: se consulta por SQL (Telemetría de Producto, 6) |

> El alcance del veterinario sobre la ficha de Tutor no está acotado a su clínica, a diferencia del que tiene sobre Paciente. El motivo es el proceso de alta de paciente (Reglas de Negocio, 4.1): la ficha del tutor se busca y se completa antes de que exista ningún Paciente que vincule a esa persona con una clínica, así que no hay dato sobre el cual acotarla. Es una excepción deliberada, a revisar cuando exista la entidad Paciente.

> **El tutor escribe en las dos entidades del historial, y esa es la novedad de esta versión** (sección 1). Lo que lo acota no es un permiso temporal ni un estado de la ficha: es el `cargado_por` del registro. Escribe filas con `cargado_por = tutor`, y edita y da de baja esas y ninguna otra — sobre un registro del veterinario no tiene ni edición ni baja, en ningún nivel y en ningún momento. El veterinario, al revés, alcanza todo el historial del paciente, incluido lo que declaró el tutor: es dato de la mascota que atiende.
>
> **No está atado al alta.** La capacidad existe siempre, desde el primer día de la mascota y desde el año que viene. El onboarding es una pantalla que la usa, no la condición que la habilita: modelarla como un permiso que se apaga después habría dejado sin cargar la libreta al tutor que la encuentra en un cajón dos meses más tarde.
>
> **El co-tutor con edición también carga antecedentes**, con el mismo criterio con el que ya sube adjuntos: alcanza los mismos datos que el dueño salvo administrar accesos, y quien cuida al animal la mitad de la semana sabe lo mismo de su historia. El de lectura no escribe nada.

> El rol clínica_admin gestiona veterinarios y datos administrativos de la clínica, y **no tiene acceso al historial clínico de ningún paciente**: ni al Evento clínico, ni a la Medicación, ni a los Adjuntos, ni a la ficha de un Paciente o de un Tutor. Ese acceso es del veterinario que atiende y del tutor.

> **El padrón también es una proyección.** Dar de alta una mascota exige encontrar a su tutor o crearlo (Reglas de Negocio, 4.1), y para eso el clínica_admin busca en el padrón entero —igual que el veterinario, y por el mismo motivo: antes del alta no hay vínculo contra el cual acotar—. Lo que ve de cada persona es **nombre, contacto y si ya tiene documento cargado**; no el documento ni la dirección.
>
> Es la proyección reducida que esta sección ya tenía anotada como pendiente para la búsqueda del veterinario. Se implementa primero acá porque es el rol que se está abriendo ahora; la del veterinario sigue devolviendo la ficha completa y sigue siendo deuda.
>
> Del alta escribe **nombre, contacto y consentimiento**, que es lo que el auto-registro también pide. Documento y dirección los completa el veterinario cuando atiende (3.2).

> **La cartera es una proyección, no la ficha.** Para agendar hay que elegir una mascota, y el clínica_admin no lee Paciente. Lo que sí alcanza es una lista acotada a la cartera de su propia clínica con **nombre, especie, y el nombre y contacto de su tutor** — lo justo para tomar un turno por teléfono y saber a quién llamar. Ni fecha de nacimiento, ni sexo, ni peso, ni chip, ni nada que cuelgue de la mascota.
>
> Es el mismo criterio con el que el tutor lee el directorio de Clínicas (4.3): lo que protege el dato no es el alcance sino **la proyección**. Y es poco más de lo que la agenda ya le muestra, que trae el nombre y la especie de cada cita.

> **La agenda es del libro de turnos, no del historial.** El clínica_admin lee las Citas de su clínica fila por fila —con la mascota, el tipo y quién atiende— y reparte las que no tienen profesional. Una Cita no lleva diagnóstico ni evolución: es cuándo viene quién, y atender esa pregunta es la tarea de quien administra la clínica.
>
> Es un cambio respecto de la versión anterior de esta sección, que le daba **solo conteos** de Cita y dejaba escrito que "un endpoint que devuelva Citas fila por fila sigue estando fuera del alcance de este rol". Ese criterio trataba a la agenda como si fuera historial. El costo asumido y explícito es que "Luna, cirugía, martes 10:00" es información de salud de un paciente identificado, y queda alcanzada por el aviso de privacidad como el resto del padrón.
>
> Lo que **no** cambió: sigue sin alcanzar Evento clínico, Medicación, Adjuntos, ficha de Paciente ni ficha de Tutor. Y **no asienta la atención** (4.16), que es afirmación asistencial y la hace quien atendió.
>
> **Agenda, reagenda y reparte; no da de baja.** La versión anterior de esta sección le dejaba solo la asignación, con el argumento de que agendar y mover un turno son decisiones clínicas. El argumento estaba mal aplicado: lo que Reglas de Negocio reserva como criterio clínico es frente al **tutor** —"qué control corresponde y quién lo hace son criterio de la clínica"— y el clínica_admin *es* la clínica. Tomar y mover turnos es la tarea del mostrador.
>
> Lo que sigue siendo del veterinario y del tutor es **dar de baja una cita**: retirarla del calendario es decir que no va a ocurrir, y eso lo sabe quien atiende o quien lleva a la mascota.

> **Agregados de gestión.** Además de la agenda, el clínica_admin lee **conteos** sobre su propia clínica: cuántas Consultas atendidas hubo por semana y por profesional, cuántos Vínculos con clínica están vigentes. Se decidió que **un conteo sin `paciente_id` ni dato clínico no es historial clínico**, y negárselo dejaba al rol que administra la clínica sin poder saber si su propia agenda se está usando.
>
> Para Consulta atendida y Vínculo el límite sigue siendo la forma del dato: la fila es un número y un corte (semana o mes, profesional, `origen`), nunca una lista de registros ni un identificador de mascota. Un listado de "atenciones del martes" con hora y profesional reconstruye quién fue atendido en cuanto se cruza con cualquier otra cosa, y eso sigue afuera.
>
> Alcanza **solo a su propia clínica**, por `clínica_id`, como todo lo demás de este rol.

## 6. Preparado para Fase 2

Los siguientes campos no se utilizan activamente en el MVP pero se incluyen desde ahora para evitar una migración de esquema costosa cuando se incorpore el matching geolocalizado de urgencias:

- **Paciente.identificador_externo** — permite localizar el historial vía chip/microchip sin depender del nombre de la clínica de origen.
- **Veterinario.matrícula** — base para validación profesional automática (tipo KYC).
- **Evento clínico.campo_estructurado** — permite sumar campos de triaje estructurado sin romper el esquema existente.
- **Tutor.dirección_lat / dirección_lng y Clínica.dirección_lat / dirección_lng** — el punto en el mapa sobre el que va a correr la búsqueda por cercanía. Se capturan desde el MVP porque geocodificar hacia atrás un padrón de direcciones escritas a mano años después es reconstruir el dato, no migrarlo.
