# Wayka — Sincronización sin conexión

MVP — Acceso offline del tutor y reconciliación
Versión 1.0 · Complementa a Modelo de Datos, Reglas de Negocio, Alcance de Plataformas y Arquitectura del Sistema

## 1. Alcance

Este documento define cómo un cliente de Wayka lee y escribe sin conexión, y cómo esos cambios se reconcilian con el backend al volver online. Es contrato de producto: vale igual para el backend y para el frontend.

Entra al MVP **únicamente el tutor en móvil**. Queda fuera, deliberadamente:

| Fuera de alcance | Por qué |
|---|---|
| Veterinario offline | Su conjunto de datos es toda la cartera de la clínica: volumen alto, escritura clínica y, sobre todo, un dispositivo perdido que expone el historial de pacientes que no son del portador. El diseño para incorporarlo está previsto en la sección 9, pero no se implementa acá. |
| Clínica_admin offline | Gestiona plantel y datos administrativos desde la web, sentado en la clínica. No hay caso de uso. |
| Web offline | La web no tiene el almacenamiento local cifrado del que depende la sección 8, y el rol que la usa sin señal no existe (el tutor no accede por web — Alcance de Plataformas, sección 2). |
| Adjuntos (bytes) | Solo se replica su metadato. Los binarios multiplican el tamaño de la copia local y su caso de uso offline —mirar una radiografía sin señal— no justifica todavía ni el espacio ni la exposición. |
| Notificaciones | Un recordatorio es un push que el servidor encola (Reglas de Negocio, 4.15). Sin red no llega, y programarlo localmente duplicaría la lógica de encolado en el cliente con otro reloj. Un dispositivo que estuvo offline recibe sus avisos al reconectar, o no los recibe. |

> El tutor es el caso barato y el que más lo necesita: su copia local son las pocas mascotas que alcanza, su superficie de escritura es de unos pocos campos, y es el único usuario que abre la app en la calle o en el campo. El veterinario tiene señal en la clínica y un caso mucho más caro. Empezar por el difícil habría sido empezar por el que menos paga.

## 2. Qué se replica: partición por audiencia

El conjunto replicable de un tutor es: **las mascotas que alcanza** —las suyas (`Paciente.tutor_id`) y aquellas sobre las que tiene un acceso de co-tutor vigente, en cualquier nivel—, y colgando de cada una sus Eventos clínicos, Medicaciones, Citas y metadatos de Adjuntos. Más su propia ficha de Tutor.

Ese conjunto **no está acotado por clínica**: una mascota puede estar atendida por varias clínicas o por ninguna, y eso no cambia quién la ve. La partición de replicación de este documento es por **audiencia**, no por tenant.

> Vale la pena decirlo explícitamente porque es contraintuitivo: acceso offline y multi-clínica parecen el mismo problema y no lo son. El aislamiento por clínica es una regla del motor de permisos que acota al veterinario y al clínica_admin (Modelo de Datos, sección 5); la replicación es un subconjunto de lo que una audiencia ya puede leer. Para el tutor esas dos cosas no coinciden, y modelar la copia local por clínica le escondería mascotas que sí alcanza.

La invariante que sí las une, y que no se negocia: **lo replicable es siempre un subconjunto de lo que el motor de permisos ya autoriza**. Ningún dispositivo guarda un registro que el backend le negaría en una lectura online.

El co-tutor con nivel de **solo lectura** también tiene copia local, y no es una concesión: la invariante de arriba dice que lo replicable es un subconjunto de lo que el motor ya autoriza, y ese tutor está autorizado a leer esa mascota. Lo que su nivel acota es qué puede escribir, y de eso se ocupa la sección 5.

**El conjunto ahora puede perder una mascota que nadie dio de baja.** En la primera versión de este documento no podía: `Paciente.tutor_id` es inmutable, así que el conjunto de un tutor solo crecía o decrecía por altas y bajas de sus propias mascotas, y este documento afirmaba —correctamente, entonces— que no existía el problema de un registro que debiera desaparecer del dispositivo de alguien.

Con la mascota compartible eso deja de ser cierto por los dos lados:

- **Otorgar** un acceso incorpora una mascota entera, con todo su historial, a un dispositivo que no tenía nada de ella.
- **Revocar** un acceso, o bajarlo de edición a lectura, la saca o la degrada.

Ninguna de las dos cosas modifica un solo registro de la mascota, así que ninguna deja rastro por sí sola en la bitácora de la sección 3. Es el problema central que resuelven las secciones 4 y 8.

## 3. El cursor de sincronización

Cada cambio sobre una entidad replicable deja una fila en una bitácora de cambios:

| Campo | Descripción |
|---|---|
| seq | Entero creciente. Es el cursor, y su orden es el orden de *commit* — ver abajo. |
| entidad, entidad_id | Qué registro cambió. |
| op | `alta` \| `modificación` \| `baja`. |
| paciente_id | Mascota de la que cuelga el cambio. Es la clave de partición de las cuatro entidades hijas. |
| tutor_id | Solo para la ficha del propio Tutor, que no cuelga de ninguna mascota. |
| ocurrido_en | Momento del cambio en el servidor. Informativo: no se usa para ordenar ni para resolver. |

Decisiones:

- **El cursor es `seq`, no `updated_at`.** Un timestamp no sirve por dos motivos independientes: dos transacciones concurrentes pueden escribir timestamps en un orden y hacer *commit* en el otro, dejando un cambio permanentemente invisible para un cliente que ya avanzó su marca; y cualquier corrección de reloj del servidor reordena la bitácora hacia atrás. Un entero no tiene el segundo problema — pero el primero **tampoco lo arregla solo**, y de eso se ocupa la decisión de más abajo.
- **La audiencia se deriva al leer, no se resuelve al escribir.** La bitácora guarda la clave del padre —`paciente_id`, que ya viene en la fila que se está escribiendo— y el delta llega al tutor por un join contra Paciente. La alternativa era que el disparador resolviera el `tutor_id` con una consulta por cada escritura, o denormalizarlo en las cuatro tablas hijas. Dos motivos para no hacerlo: lo derivado al leer siempre refleja el vínculo vigente, mientras que lo resuelto al escribir queda congelado en la fila; y porque un disparador que no lee no tiene modos de falla que pensar.
  - **Esa decisión ya se cobró.** Se tomó anticipando que "en Fase 2 la clínica de un paciente deja de ser inmutable", y eso ocurrió: hoy una mascota puede ser atendida por varias clínicas y vista por varios tutores. La bitácora no cambió de forma — sigue guardando `paciente_id` y derivando la audiencia con un join, que ahora es contra las tablas de vínculo en vez de contra dos columnas. Si el `tutor_id` se hubiera denormalizado en las cuatro tablas hijas, toda la bitácora histórica habría quedado mal. El costo es un join por bajada en vez de una consulta por escritura, que es el intercambio correcto: la escritura clínica es el camino permanente y la bajada ya está paginada.
- **Con `paciente_id` alcanza también para el veterinario.** Su audiencia sale del mismo join, ahora contra el vínculo de su clínica con el paciente. Una sola columna de partición en lugar de dos, y es la que no puede quedar desactualizada.
- **La excepción es el Acceso de co-tutor**, que particiona por `tutor_id`: es el único cambio replicable cuyo destinatario es una persona y no una mascota. La ficha del propio Tutor ya usaba esa misma clave por el mismo motivo.
- **El número lo entrega un contador en una fila, no una secuencia.** Un `bigserial` ordena por el momento en que se pide el número, no por el commit: dos transacciones pueden tomar 5 y 6, la de 6 commitear primero, y un cliente que lea justo ahí guarda la marca 6 y **nunca va a ver el cambio 5**. Es el mismo defecto que descarta a `updated_at` como cursor, y una secuencial no lo arregla. Incrementar una fila sí: el bloqueo se sostiene hasta el commit, así que nadie puede obtener un número mayor antes de que el menor haya terminado, y el orden de numeración es el de commit por construcción. El precio es que toda escritura replicable se serializa en esa fila, que con una clínica piloto no se nota.
- **La bitácora la escribe un disparador de base de datos, no la capa de datos.** Es el único punto por el que pasan todas las escrituras. Poblarla desde el código de negocio significa que la próxima escritura que alguien agregue sin acordarse deja un cambio que ningún cliente va a ver nunca — y ese error es silencioso, que es la peor clase.
- **La baja lógica ya existente alcanza como lápida.** Como ningún dato se borra físicamente (Modelo de Datos, sección 3), un `deleted_at` es un `UPDATE` y viaja por el mismo camino que cualquier otro cambio. No hace falta una tabla de borrados.
- **La bitácora se purga.** Retención por defecto: **90 días** (`BITACORA_CAMBIOS_RETENCION_DIAS`). Es lo que define cuánto puede estar un dispositivo sin sincronizar antes de tener que rehacer su copia entera (sección 4).

> Si alguna vez el contador serializado llegara a molestar —no con una clínica—, la salida conocida es guardar `pg_current_xact_id()` en cada fila y que el lector nunca devuelva más allá de `pg_snapshot_xmin(pg_current_snapshot())`, o sea que excluya la cola donde todavía hay transacciones en vuelo. No serializa nada, a cambio de una columna, un cálculo en cada lectura y un retraso de visibilidad que dura lo que la transacción abierta más vieja. Se deja anotada y no se implementa: la versión con contador es correcta de leer de un vistazo y esta hay que pensarla dos veces para creerle.

> `ocurrido_en` está en la tabla y no se usa para nada operativo. Es a propósito: es el campo que se mira cuando hay que entender qué pasó, y ponerlo a decidir el orden es exactamente lo que este diseño evita. Quién cambió qué y cuándo se sigue consultando en Auditoría (Modelo de Datos, 4.10), que no se reemplaza ni se duplica acá — la bitácora de cambios es un índice de replicación, no un registro de responsabilidad.

## 4. Bajada de cambios

Un cliente guarda la última `seq` que aplicó y pide lo posterior. La respuesta trae los cambios, la nueva marca, y si el pedido cabía completo.

Reglas:

- **La audiencia la resuelve el token, nunca el pedido.** El cliente pide "desde tal marca"; a qué tutor corresponde lo decide la sesión. Un parámetro de audiencia en el pedido sería una vía de leer el conjunto de otra persona.
- **Un cambio se envía con el estado actual del registro, no con el delta del campo.** Si un registro cambió cinco veces desde la última sincronización, viaja una sola vez y con su valor final. La bitácora dice *qué* mirar; el contenido sale de la tabla.
- **Respuesta paginada, con la marca al final.** El cliente solo avanza su marca cuando aplicó el lote entero, en una sola transacción local. Avanzarla por adelantado deja huecos permanentes si la aplicación se interrumpe.
- **La carga inicial es la excepción: no se pagina.** Viaja entera, con `hay_mas` siempre en false, y el límite pedido no se aplica. Paginar una respuesta que cruza seis entidades a la vez pide un cursor compuesto que hay que mantener para siempre, y lo que acota el tamaño acá no es la paginación sino el conjunto: las mascotas de una persona y su historial. Donde esta excepción no se sostiene es en el veterinario, cuyo conjunto es la cartera de una clínica — y es la misma razón por la que la sección 9 le exige una ventana acotada antes de habilitarlo. Si esa ventana llega, esta decisión se revisa con ella.
- **Marca fuera de retención: se rehace la copia.** Si la marca del cliente es anterior a lo que la bitácora conserva, no hay forma de saber qué se perdió. La respuesta lo indica y el cliente descarta su copia local y hace una carga inicial completa. Es el mismo camino que el de la primera vez que se instala la app.
- **Un cambio de acceso obliga a rehacer la copia.** El Acceso de co-tutor es una entidad replicable más, pero particionada **por tutor** y no por mascota: a quien le importa que le hayan otorgado, revocado o cambiado de nivel un acceso es a la persona, no al animal. Recibir una fila suya no se aplica como las demás — el cliente descarta su copia y hace una carga inicial, por el mismo camino que ya existe para la marca fuera de retención.
  - Es lo que resuelve el problema de la sección 2 en los tres casos de una sola vez: el acceso otorgado, el revocado y el nivel que cambió. Ninguno toca un registro de la mascota, así que ninguno tiene delta que enviar.
  - Para el acceso **otorgado** no hay alternativa barata: un delta no puede traer el historial de una mascota cuyas filas son todas anteriores a la marca del cliente. Lo único que lo evitaría sería sembrar la bitácora con un alta por cada registro de ese historial — miles de filas en una cola que todas las demás audiencias tienen que saltear, y que además mienten, porque no cambió nada.
  - El precio es que también se tiran las mascotas propias, que no cambiaron. Con las pocas mascotas que tiene un tutor no se nota. Si alguna vez se nota, la salida es incorporar y purgar **por mascota** en vez de rehacer todo, y para eso hacen falta un número de secuencia en el vínculo y la lista completa de vínculos en cada respuesta. Está pensado y no implementado: no hace falta todavía.
- **Un registro que sale del conjunto por baja viaja como baja.** Lo que ya no viaja es el registro que sale porque el acceso se revocó: ahí no hay baja que enviar, y de eso se ocupa la regla anterior.
- **Una mascota dada de baja no es una lápida.** La ficha de un Paciente con `deleted_at` se sigue leyendo, con su historial completo (Modelo de Datos, 4.2), así que viaja como un registro más y se guarda en el dispositivo. Las lápidas reales son las de Evento clínico, Medicación, Cita y Adjunto: esos sí salen de la copia local, porque su baja no se expone en ninguna lectura.
- **La página del delta es más grande que la de un listado de pantalla.** Hasta 500 registros por pedido, 200 por defecto, contra los 200/50 del resto de la API: acá no hay nadie mirando la respuesta, y una carga inicial paginada de a 50 son decenas de viajes contra una conexión que ya demostró ser mala.

## 5. Subida de mutaciones

El cliente acumula lo que el tutor hizo sin conexión y lo envía en lote al reconectar.

- **Se envía la intención, no el registro completo.** "Reagendó esta cita para tal hora" y no "la cita ahora es este objeto". Un estado final pisa los campos que el tutor no tocó con los valores que su copia tenía, que pueden ser viejos; una intención dice exactamente qué se quiso cambiar y deja intacto todo lo demás.
- **Cada mutación lleva un identificador propio, generado por el cliente.** Es lo que hace que reenviar un lote cuya respuesta se perdió no duplique nada. El backend guarda los identificadores ya aplicados y responde lo mismo que la primera vez.
- **Cada mutación se valida entera, como si llegara online.** El motor de permisos y las reglas de negocio corren igual, sin ningún camino especial para lo que viene de la cola. Esta es la aplicación directa del principio de que el backend es la única barrera de seguridad: una cola de sincronización que se aplicara con validación relajada sería una API paralela sin permisos.
- **Se valida con el rol, el vínculo y la vigencia actuales, no con los del momento en que se encoló.** Un usuario desactivado, una mascota dada de baja, o **un acceso de co-tutor revocado o bajado a solo lectura** mientras el dispositivo estaba sin señal, rechazan la mutación. Lo contrario dejaría que una cola vieja escriba después de que el permiso se retiró — y con la mascota compartible ese ya no es un caso teórico, es lo que pasa cada vez que alguien revoca un acceso.
- **Compartir y revocar no pasan por la cola.** Son las únicas escrituras del tutor que quedan afuera, y a propósito: encolar una revocación es revocar en diferido, que es la peor versión del problema que la sección 8 describe. Sin señal, la aplicación lo dice y no ofrece la acción. Por el mismo motivo queda afuera el alta de una mascota, aunque el argumento ahí es otro: encolarla obligaría a inventar identificadores locales y a reconciliarlos después, un mecanismo entero para una pantalla que se usa dos veces en la vida.
- **El lote no es una transacción.** Cada mutación se acepta o se rechaza por separado y la respuesta lo dice una por una. Un lote parcialmente rechazado es el caso normal, no un error: rechazar las diez porque la séptima no era válida haría perder nueve cambios buenos.
- **El lote tiene un tope de 50 mutaciones.** No es una defensa contra abuso —cincuenta pedidos de cincuenta mutaciones no son más caros que uno de dos mil— sino un límite al costo de un solo pedido: la subida se valida entera regla por regla y un lote sin techo puede tardar más que el tiempo de espera del cliente, que entonces lo reenvía y duplica el trabajo. Una cola más larga se envía en lotes sucesivos, en orden. Con la superficie de escritura del tutor, cincuenta mutaciones acumuladas ya son varios días sin señal.
- **La dirección se edita sin conexión, pero sin mapa.** El autocompletado de direcciones es una llamada de red a un servicio externo (Arquitectura, 3.6): sin señal no hay sugerencias. El campo degrada a texto libre y la mutación viaja con la dirección escrita y sin coordenadas, que es exactamente lo que la regla 2.6 de Reglas de Negocio ya admite online. Bloquear la edición hasta tener señal sería hacer que un dato de contacto dependa de un proveedor que no es nuestro; guardar el punto viejo junto al texto nuevo dejaría el pin en la casa anterior. El tutor puede volver a confirmarla en el mapa cuando recupere conexión.
- **Auditoría registra la escritura cuando ocurre en el servidor**, con el usuario que la originó. El momento en que el tutor la hizo en su teléfono viaja como dato informativo de la mutación y se guarda, pero no reemplaza la fecha del asiento: es un dato que declara el cliente y no se puede verificar.

## 6. Conflictos

La superficie de escritura del tutor sigue siendo corta (Reglas de Negocio, 3.2), y eso es lo que hace este diseño viable. Lo que cambió es que ahora puede haber **más de una persona escribiendo** sobre la misma mascota:

| Mutación | Conflicto posible | Resolución |
|---|---|---|
| Datos no clínicos del Paciente (nombre, especie, raza, fecha de nacimiento, sexo, peso) | El veterinario, o **el otro tutor**, editan el mismo registro | Concurrencia optimista (abajo) |
| Ficha propia de Tutor, salvo `consentimiento_datos` | El veterinario completa documento o dirección | Concurrencia optimista |

> La dirección son cuatro campos que se escriben como un bloque (Modelo de Datos, 3.1), así que en la cola es **una sola mutación** y no cuatro. Partirla dejaría que se aplique el texto y se rechace el punto, o al revés, y el resultado sería una ficha con la dirección de una casa y las coordenadas de otra — justo el estado que la regla 2.6 existe para evitar.
| `Cita.notificar_tutor` | El veterinario lo cambia | Concurrencia optimista |
| `Cita.fecha_programada` (reagenda) | **Las reglas de agenda del servidor** | Solo el servidor decide |
| Baja lógica de una Cita | La cita dejó de estar pendiente | Solo el servidor decide |
| **Cualquiera, sobre una mascota ajena** | El dueño revocó el acceso, o lo bajó a solo lectura, mientras el teléfono estaba sin señal | Rechazo por permiso. **No hay valor del servidor que mostrar** |

**Concurrencia optimista, no "gana el último".** La mutación viaja con el `updated_at` que el registro tenía en la copia local. Si el del servidor es otro, alguien lo modificó mientras tanto: la mutación se rechaza por desactualizada y el cliente resuelve mostrando el valor del servidor.

> El testigo de versión es `updated_at` y no la `seq` de la bitácora, aunque la bitácora sea lo que ordena el resto de este documento. Son dos usos distintos del mismo cambio: la `seq` **ordena** —para eso existe— y `updated_at` acá solo se **compara por igualdad**, que es todo lo que la concurrencia optimista necesita. Un timestamp no sirve como cursor (sección 3) y sí sirve como testigo, porque nada depende de cuál de los dos valores es mayor. La alternativa era exponer la `seq` de cada registro en todas las lecturas online, que es filtrar un contador interno de replicación a un cliente que no sincroniza nada.

Justificación:

- **No hay ningún reloj de cliente en la decisión.** "Gana el último" necesita comparar el momento de dos escrituras, y una de ellas la fecha un teléfono cuyo reloj el backend no controla. Un dispositivo adelantado dos días pisaría en silencio todo lo que el veterinario escriba en ese lapso.
- **En una historia clínica, perder una escritura en silencio es peor que pedirla de nuevo.** Un rechazo es visible y el tutor vuelve a cargar el peso en diez segundos. Una resolución automática que descarta el dato correcto no deja rastro en la pantalla de nadie.
- **El costo es una pantalla de conflicto, que este diseño necesita igual** por la reagenda (abajo). Resolver los campos benignos automáticamente no habría ahorrado esa pantalla, solo habría agregado un segundo mecanismo al lado.

La granularidad es **por registro, no por campo**. Con la mascota compartida hay más de un tutor escribiendo, así que el conflicto falso —dos personas editando campos distintos del mismo registro— pasó de raro a posible; sigue siendo la granularidad correcta igual, porque un conflicto falso se resuelve reintentando en diez segundos y una marca por columna en toda la copia local es una complejidad permanente.

**El rechazo por acceso revocado es distinto de todos los demás** y la pantalla lo tiene que tratar distinto: no hay un valor del servidor que mostrar ni un horario alternativo que ofrecer, porque la mutación no perdió una carrera — dejó de estar permitida. Lo único que se ofrece es descartarla. Y si la mascota ya salió de la copia local, la fila muestra el nombre que quedó guardado en la cola, en vez de buscar una ficha que ya no está.

**La reagenda de una Cita no se puede resolver en el cliente por definición.** Cambiar `fecha_programada` exige que la hora caiga en el horario de atención de la clínica, esté alineada a la grilla de turnos, no le solape otra cita al profesional asignado y la cita siga pendiente (Reglas de Negocio, 2.2 y 4.6). Las tres primeras dependen de datos que pudieron cambiar mientras el dispositivo no tenía señal: el turno que el tutor eligió puede haberse ocupado. El cliente valida lo que puede para no ofrecer horarios obviamente inválidos, pero eso es UX; la respuesta real llega al sincronizar.

**Un rechazo por agenda devuelve hasta tres horarios alternativos**, los libres más cercanos al que se pidió, ya validados contra el horario de atención, la grilla de turnos y la ocupación del profesional asignado.

> Es la diferencia entre "no se pudo" y "no se pudo, pero tenés estos tres". Sin las alternativas, el tutor que reagendó sin señal vuelve a una pantalla que le pide adivinar, y adivina contra un calendario que su dispositivo tiene desactualizado — que es exactamente la situación que produjo el rechazo. El cálculo lo hace el servidor porque es el único que conoce la agenda real, y devolverlo en la misma respuesta evita un segundo viaje en el peor momento posible para pedirlo.
>
> **La misma respuesta la da la reagenda online** (`PATCH /citas/{citaId}`). El problema es idéntico y el cálculo es el mismo; tener dos formas de contestar "ese turno no se puede" según por dónde entró el pedido sería una diferencia sin causa. Ahí viaja en dos códigos distintos —400 para la hora fuera de la grilla o del horario, que es una validación previa, y 409 para el turno ocupado, que es un choque con el estado actual— con el mismo cuerpo en los dos: lo que el cliente hace al recibirlo es lo mismo, elegir otra hora.
>
> Las alternativas se devuelven solo cuando el rechazo lo admite: turno ocupado, fuera de horario o fuera de la grilla. Una cita que ya no está pendiente no se reagenda a ninguna hora, y ofrecer alternativas ahí sería mentir sobre lo que se puede hacer.

## 7. Rechazos en la interfaz

Un cambio del tutor tiene tres estados en su dispositivo: **sincronizado**, **pendiente** y **rechazado**. Los tres son visibles.

- Lo pendiente se muestra aplicado, marcado como no confirmado. Ocultarlo hasta que sincronice haría que la app pareciera haber perdido lo que el tutor acaba de escribir.
- Lo rechazado se muestra con el motivo y con el valor que quedó en el servidor. La app **no reintenta sola** una mutación rechazada: el rechazo significa que las condiciones cambiaron, y reintentar es pedirle al servidor la misma respuesta otra vez.
- Una mutación rechazada sale de la cola cuando el tutor la resuelve o la descarta. Una cola que acumula rechazos indefinidamente reenvía basura en cada sincronización.

> Los motivos de rechazo que el tutor ve son de negocio ("ese turno ya está ocupado", "el peso lo actualizó la clínica"), no técnicos. Un rechazo por permiso denegado se muestra genérico, por el mismo criterio que el resto de la API (Arquitectura, 4.2.2).

## 8. Los datos en el dispositivo

Una copia local del historial clínico de una mascota es un dato personal de salud fuera del servidor, y es la parte de este diseño con consecuencias legales (Ley 25.326, criterio rector del Modelo de Datos, sección 1).

- **La base local va cifrada en reposo**, con una clave guardada en el almacenamiento seguro del sistema operativo. Un teléfono perdido no entrega el historial por copiar un archivo.
- **Cerrar sesión destruye la copia local**, no la conserva por si el usuario vuelve. Es la misma decisión que revocar la cadena de sesión entera al cerrar (Arquitectura, 4.2.1): el estado que sobrevive a un cierre de sesión es estado que alguien más puede leer.
- **La copia local no sobrevive a un token de refresco rechazado.** Si el backend corta la sesión —cuenta desactivada, reuso detectado—, el cliente descarta la copia en el mismo paso en que vuelve a la pantalla de ingreso.
- **Sin sesión válida no hay lectura offline.** El vencimiento del token de acceso no bloquea la lectura local (sería inutilizable: quince minutos sin señal y la app deja de servir), pero el vencimiento del **refresco** sí. Eso acota cuánto tiempo puede un dispositivo mostrar datos sin volver a demostrar que la sesión sigue viva: 30 días por defecto (`REFRESH_TOKEN_TTL`).

- **Revocar un acceso saca la mascota del dispositivo en la próxima sincronización.** El acceso revocado llega como un cambio más y dispara la carga inicial de la sección 4, que rehace la copia sin esa mascota. Para que eso ocurra en segundos y no en la próxima vez que el tutor abra la app, revocar además **empuja un aviso silencioso** a los dispositivos de esa persona, que dispara la sincronización sin mostrar nada en pantalla.

> Esto amplía la ventana de revocación de Arquitectura, 4.3, y conviene decirlo sin eufemismo. Ahí la exposición tras desactivar a un usuario eran los minutos del token de acceso. Con copia local, ese usuario sigue **leyendo** lo que ya tenía en el teléfono hasta que la app intente refrescar y sea rechazada. Escribir no puede: sus mutaciones se validan con el estado actual (sección 5).

> **Con la mascota compartible, el balance de ese párrafo cambia y hay que decirlo.** La primera versión de este documento lo cerraba diciendo que para el tutor era aceptable porque su copia son datos de sus propias mascotas. Con co-tutores eso deja de ser cierto: parte de la copia son datos de una mascota **ajena**, y a quien se le revoca el acceso le queda en el teléfono hasta que el aparato se conecte. El empujón silencioso cubre el caso normal —un teléfono prendido y con señal purga en segundos— y no cubre el que importa: el teléfono apagado, en modo avión, o de alguien que desinstaló la app sin cerrar sesión. Ahí el límite superior sigue siendo el del refresco, 30 días.
>
> Es una fuga de **lectura**, sobre datos que esa persona ya había visto legítimamente, y no de escritura: cualquier mutación suya rebota contra el vínculo actual. Aun así es una fuga, y es el punto más débil de este diseño.
>
> **Pendiente, con la decisión de producto sin tomar**: acotar por antigüedad la copia de las mascotas **ajenas** —que dejen de abrirse si hace más de N días que no se confirma el acceso, con N bastante menor que 30— manteniendo el límite del refresco para las propias, que nadie puede revocar. Cerraría el hueco exactamente sobre el conjunto que lo produce, sin castigar al tutor que solo tiene mascotas suyas. Lo que falta no es el mecanismo, es el número.
>
> Para el veterinario, cuya copia sería de pacientes ajenos por definición y en volumen, **el balance sigue sin cerrar**, y es una de las razones por las que la sección 9 no es un simple ensanchamiento del alcance.

## 9. Previsto para el veterinario, no implementado

El diseño deja lugar para incorporar al veterinario sin migrar el esquema: la bitácora de cambios parte por `paciente_id` (sección 3), y la clínica sale del mismo join que el tutor. No hay ninguna columna que agregar.

Lo que **falta decidir** antes de habilitarlo, y que este documento no resuelve:

- **Qué se replica.** No toda la cartera de la clínica: una ventana acotada (pacientes con cita próxima, más los atendidos recientemente). Eso implica que la copia local tiene huecos y que la interfaz tiene que saber decir "este paciente no está en tu copia", que es una pantalla que hoy no existe.
- **La escritura clínica offline.** Los Eventos clínicos son en la práctica de solo agregado, así que casi no tienen conflicto — pero sí tienen reglas de validación contra estado del servidor (paciente vigente, una medicación activa por droga) que se rechazan igual que una reagenda.
- **La exposición del dispositivo**, según la nota de la sección 8. Es el punto que decide si esto se hace, no un detalle de implementación.

## 10. Fuera de alcance de este documento

- **Detección de conectividad y política de reintento** (cuándo se dispara una sincronización, con qué espera entre intentos) — es decisión del cliente, va en los estándares de desarrollo de frontend.
- **Esquema de la base local** (tablas, índices, migraciones del SQLite del dispositivo) — implementación del frontend, no contrato.
- **Sincronización de los bytes de Adjuntos** — ver sección 1.
