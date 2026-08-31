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

> El tutor es el caso barato y el que más lo necesita: su copia local son sus propias mascotas, su superficie de escritura es de cinco campos, y es el único usuario que abre la app en la calle o en el campo. El veterinario tiene señal en la clínica y un caso mucho más caro. Empezar por el difícil habría sido empezar por el que menos paga.

## 2. Qué se replica: partición por audiencia

El conjunto replicable de un tutor es: **sus Pacientes** (`Paciente.tutor_id`), y colgando de cada uno sus Eventos clínicos, Medicaciones, Citas y metadatos de Adjuntos. Más su propia ficha de Tutor.

Ese conjunto **no está acotado por clínica**: un tutor puede tener mascotas atendidas en clínicas distintas (Reglas de Negocio, 3.2 — "sus propias mascotas, estén atendidas en la clínica que sea"). La partición de replicación de este documento es por **audiencia**, no por tenant.

> Vale la pena decirlo explícitamente porque es contraintuitivo: acceso offline y multi-clínica parecen el mismo problema y no lo son. El aislamiento por clínica es una regla del motor de permisos que acota al veterinario y al clínica_admin (Modelo de Datos, sección 5); la replicación es un subconjunto de lo que una audiencia ya puede leer. Para el tutor esas dos cosas no coinciden, y modelar la copia local por `clinica_id` le escondería mascotas que sí son suyas.

La invariante que sí las une, y que no se negocia: **lo replicable es siempre un subconjunto de lo que el motor de permisos ya autoriza**. Ningún dispositivo guarda un registro que el backend le negaría en una lectura online.

`Paciente.tutor_id` no es editable (Modelo de Datos, 4.2), así que el conjunto de un tutor solo crece o decrece por altas y bajas de sus propias mascotas. No existe el traspaso de una mascota a otra persona, y por lo tanto tampoco el problema de un registro que deba desaparecer del dispositivo del dueño anterior.

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
- **La audiencia se deriva al leer, no se resuelve al escribir.** La bitácora guarda la clave del padre —`paciente_id`, que ya viene en la fila que se está escribiendo— y el delta llega al tutor por un join contra Paciente. La alternativa era que el disparador resolviera el `tutor_id` con una consulta por cada escritura, o denormalizarlo en las cuatro tablas hijas. Dos motivos para no hacerlo: lo derivado al leer siempre refleja el vínculo vigente, mientras que lo resuelto al escribir queda congelado en la fila —hoy da igual porque `tutor_id` y `clinica_id` son inmutables, pero en Fase 2 la clínica de un paciente deja de serlo, y sería en ese escenario, el que motivó la columna, donde toda la bitácora histórica pasaría a estar mal—; y porque un disparador que no lee no tiene modos de falla que pensar. El costo es un join por bajada en vez de una consulta por escritura, que es el intercambio correcto: la escritura clínica es el camino permanente y la bajada ya está paginada.
- **Con `paciente_id` alcanza también para el veterinario.** Su audiencia sale del mismo join, por `Paciente.clinica_id`. Una sola columna de partición en lugar de dos, y es la que no puede quedar desactualizada.
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
- **Un registro que sale del conjunto viaja como baja.** No hay un caso de "dejó de ser tuyo" distinto del de "se dio de baja" (sección 2).
- **Una mascota dada de baja no es una lápida.** La ficha de un Paciente con `deleted_at` se sigue leyendo, con su historial completo (Modelo de Datos, 4.2), así que viaja como un registro más y se guarda en el dispositivo. Las lápidas reales son las de Evento clínico, Medicación, Cita y Adjunto: esos sí salen de la copia local, porque su baja no se expone en ninguna lectura.
- **La página del delta es más grande que la de un listado de pantalla.** Hasta 500 registros por pedido, 200 por defecto, contra los 200/50 del resto de la API: acá no hay nadie mirando la respuesta, y una carga inicial paginada de a 50 son decenas de viajes contra una conexión que ya demostró ser mala.

## 5. Subida de mutaciones

El cliente acumula lo que el tutor hizo sin conexión y lo envía en lote al reconectar.

- **Se envía la intención, no el registro completo.** "Reagendó esta cita para tal hora" y no "la cita ahora es este objeto". Un estado final pisa los campos que el tutor no tocó con los valores que su copia tenía, que pueden ser viejos; una intención dice exactamente qué se quiso cambiar y deja intacto todo lo demás.
- **Cada mutación lleva un identificador propio, generado por el cliente.** Es lo que hace que reenviar un lote cuya respuesta se perdió no duplique nada. El backend guarda los identificadores ya aplicados y responde lo mismo que la primera vez.
- **Cada mutación se valida entera, como si llegara online.** El motor de permisos y las reglas de negocio corren igual, sin ningún camino especial para lo que viene de la cola. Esta es la aplicación directa del principio de que el backend es la única barrera de seguridad: una cola de sincronización que se aplicara con validación relajada sería una API paralela sin permisos.
- **Se valida con el rol y la vigencia actuales, no con los del momento en que se encoló.** Un usuario desactivado, o una mascota dada de baja mientras el dispositivo estaba sin señal, rechazan la mutación. Lo contrario dejaría que una cola vieja escriba después de que el permiso se retiró.
- **El lote no es una transacción.** Cada mutación se acepta o se rechaza por separado y la respuesta lo dice una por una. Un lote parcialmente rechazado es el caso normal, no un error: rechazar las diez porque la séptima no era válida haría perder nueve cambios buenos.
- **El lote tiene un tope de 50 mutaciones.** No es una defensa contra abuso —cincuenta pedidos de cincuenta mutaciones no son más caros que uno de dos mil— sino un límite al costo de un solo pedido: la subida se valida entera regla por regla y un lote sin techo puede tardar más que el tiempo de espera del cliente, que entonces lo reenvía y duplica el trabajo. Una cola más larga se envía en lotes sucesivos, en orden. Con la superficie de escritura del tutor, cincuenta mutaciones acumuladas ya son varios días sin señal.
- **La dirección se edita sin conexión, pero sin mapa.** El autocompletado de direcciones es una llamada de red a un servicio externo (Arquitectura, 3.6): sin señal no hay sugerencias. El campo degrada a texto libre y la mutación viaja con la dirección escrita y sin coordenadas, que es exactamente lo que la regla 2.6 de Reglas de Negocio ya admite online. Bloquear la edición hasta tener señal sería hacer que un dato de contacto dependa de un proveedor que no es nuestro; guardar el punto viejo junto al texto nuevo dejaría el pin en la casa anterior. El tutor puede volver a confirmarla en el mapa cuando recupere conexión.
- **Auditoría registra la escritura cuando ocurre en el servidor**, con el usuario que la originó. El momento en que el tutor la hizo en su teléfono viaja como dato informativo de la mutación y se guarda, pero no reemplaza la fecha del asiento: es un dato que declara el cliente y no se puede verificar.

## 6. Conflictos

La superficie de escritura del tutor es corta (Reglas de Negocio, 3.2), y eso es lo que hace este diseño viable:

| Mutación | Conflicto posible | Resolución |
|---|---|---|
| `Paciente.peso_actual` | El veterinario edita el mismo campo | Concurrencia optimista (abajo) |
| Ficha propia de Tutor, salvo `consentimiento_datos` | El veterinario completa documento o dirección | Concurrencia optimista |

> La dirección son cuatro campos que se escriben como un bloque (Modelo de Datos, 3.1), así que en la cola es **una sola mutación** y no cuatro. Partirla dejaría que se aplique el texto y se rechace el punto, o al revés, y el resultado sería una ficha con la dirección de una casa y las coordenadas de otra — justo el estado que la regla 2.6 existe para evitar.
| `Cita.notificar_tutor` | El veterinario lo cambia | Concurrencia optimista |
| `Cita.fecha_programada` (reagenda) | **Las reglas de agenda del servidor** | Solo el servidor decide |
| Baja lógica de una Cita | La cita dejó de estar pendiente | Solo el servidor decide |

**Concurrencia optimista, no "gana el último".** La mutación viaja con el `updated_at` que el registro tenía en la copia local. Si el del servidor es otro, alguien lo modificó mientras tanto: la mutación se rechaza por desactualizada y el cliente resuelve mostrando el valor del servidor.

> El testigo de versión es `updated_at` y no la `seq` de la bitácora, aunque la bitácora sea lo que ordena el resto de este documento. Son dos usos distintos del mismo cambio: la `seq` **ordena** —para eso existe— y `updated_at` acá solo se **compara por igualdad**, que es todo lo que la concurrencia optimista necesita. Un timestamp no sirve como cursor (sección 3) y sí sirve como testigo, porque nada depende de cuál de los dos valores es mayor. La alternativa era exponer la `seq` de cada registro en todas las lecturas online, que es filtrar un contador interno de replicación a un cliente que no sincroniza nada.

Justificación:

- **No hay ningún reloj de cliente en la decisión.** "Gana el último" necesita comparar el momento de dos escrituras, y una de ellas la fecha un teléfono cuyo reloj el backend no controla. Un dispositivo adelantado dos días pisaría en silencio todo lo que el veterinario escriba en ese lapso.
- **En una historia clínica, perder una escritura en silencio es peor que pedirla de nuevo.** Un rechazo es visible y el tutor vuelve a cargar el peso en diez segundos. Una resolución automática que descarta el dato correcto no deja rastro en la pantalla de nadie.
- **El costo es una pantalla de conflicto, que este diseño necesita igual** por la reagenda (abajo). Resolver los campos benignos automáticamente no habría ahorrado esa pantalla, solo habría agregado un segundo mecanismo al lado.

La granularidad es **por registro, no por campo**. Con cinco campos escribibles repartidos en tres entidades, el conflicto falso —dos personas editando campos distintos del mismo registro— es raro, y afinar a nivel de campo obligaría a llevar una marca por columna en toda la copia local para ganar muy poco.

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

> Esto amplía la ventana de revocación de Arquitectura, 4.3, y conviene decirlo sin eufemismo. Ahí la exposición tras desactivar a un usuario eran los minutos del token de acceso. Con copia local, ese usuario sigue **leyendo** lo que ya tenía en el teléfono hasta que la app intente refrescar y sea rechazada. Escribir no puede: sus mutaciones se validan con el estado actual (sección 5). Para el tutor —cuya copia son datos de sus propias mascotas— el balance es aceptable. Para el veterinario, cuya copia sería de pacientes ajenos, **no lo es**, y es una de las razones por las que la sección 9 no es un simple ensanchamiento del alcance.

## 9. Previsto para el veterinario, no implementado

El diseño deja lugar para incorporar al veterinario sin migrar el esquema: la bitácora de cambios parte por `paciente_id` (sección 3), y la clínica sale del mismo join que el tutor. No hay ninguna columna que agregar.

Lo que **falta decidir** antes de habilitarlo, y que este documento no resuelve:

- **Qué se replica.** No toda la cartera de la clínica: una ventana acotada (pacientes con cita próxima, más los atendidos recientemente). Eso implica que la copia local tiene huecos y que la interfaz tiene que saber decir "este paciente no está en tu copia", que es una pantalla que hoy no existe.
- **La escritura clínica offline.** Los Eventos clínicos son en la práctica de solo agregado, así que casi no tienen conflicto — pero sí tienen reglas de validación contra estado del servidor (paciente vigente, una medicación activa por droga) que se rechazan igual que una reagenda.
- **La exposición del dispositivo**, según la nota de la sección 8. Es el punto que decide si esto se hace, no un detalle de implementación.

## 10. Fuera de alcance de este documento

- **Multi-clínica y tabla intermedia Paciente–Clínica** — sigue siendo Fase 2 (Modelo de Datos, sección 1). Este documento no la adelanta ni la necesita.
- **Detección de conectividad y política de reintento** (cuándo se dispara una sincronización, con qué espera entre intentos) — es decisión del cliente, va en los estándares de desarrollo de frontend.
- **Esquema de la base local** (tablas, índices, migraciones del SQLite del dispositivo) — implementación del frontend, no contrato.
- **Sincronización de los bytes de Adjuntos** — ver sección 1.
