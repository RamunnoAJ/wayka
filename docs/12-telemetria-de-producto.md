# Wayka — Telemetría de producto

MVP — Qué se mide del uso, cómo se registra y qué preguntas responde
Versión 1.0 · Complementa a Modelo de Datos, Reglas de Negocio, Alcance de Plataformas, Arquitectura del Sistema y Sincronización sin conexión

## 1. Alcance

Este documento define **qué hechos de uso registra Wayka, con qué forma, quién los escribe y qué métrica sale de cada uno**. Es contrato de producto: vale igual para el backend y para el frontend.

Existe porque el piloto tiene una sola pregunta que contestar —si la clínica reemplazó a su sistema anterior— y esa respuesta no está en ninguna de las entidades del Modelo de Datos. La Auditoría (Modelo de Datos, 4.10) registra qué cambió un dato clínico y quién, para responder ante un reclamo; no responde cuánto tardó el veterinario en cargarlo, cuántas veces abandonó el formulario a la mitad, ni si el tutor abrió la app alguna vez sin que se lo pidiera un push. Son dos registros con dos propósitos y dos ciclos de vida distintos, y mezclarlos degrada a los dos.

Entra al MVP el registro de eventos de uso y su consulta directa contra la base. Queda fuera, deliberadamente:

| Fuera de alcance | Por qué |
|---|---|
| Proveedor externo de analítica | Mandar el uso a un tercero (Firebase, Amplitude, PostHog alojado) exporta un flujo de datos de personas identificables desde el que se infiere información de salud —quién tiene una mascota con tratamiento activo se lee del ritmo de aperturas— y abre un tratamiento de datos por un tercero que la Ley 25.326 obliga a declarar y a contratar. El volumen de un piloto de una clínica cabe entero en una tabla de Postgres. |
| Panel de métricas en la aplicación | En el MVP las métricas se consultan por SQL contra la base. Ninguna cuenta las lee por la API (sección 6). Construir un panel antes de saber qué números se miran todos los días es construirlo dos veces. |
| Grabación de sesión, mapas de calor, replay | Capturan la pantalla de una ficha clínica. No entran, ni con consentimiento. |
| Telemetría técnica de infraestructura | Latencias, tasas de error y trazas son observabilidad del backend, no uso del producto. Viven en el log estructurado (`backend/docs/07`), no acá. La sección 5.4 define las únicas dos medidas de rendimiento que sí son de producto porque decidieron un abandono. |
| Eventos antes del login | La ingesta exige token (sección 4). Medir el embudo de registro exigiría una ruta pública que acepta escrituras anónimas, y el dato que daría —cuántos abandonan el alta— hoy se responde mirando invitaciones canjeadas contra enviadas. |

## 2. Principios

- **El backend es la fuente de verdad también acá.** Todo hecho que el backend ya ve —se creó un evento clínico, se canjeó una invitación, se despachó un push— lo emite el backend, no el cliente. Un cliente que emite lo que el servidor ya sabe genera dos cifras que no coinciden y una discusión sobre cuál vale.
- **El cliente emite solo lo que el backend no puede ver**: qué pantalla se abrió, cuánto tardó un formulario entre que se abrió y se guardó, si se abandonó, si la sesión se sirvió de la copia local, si el usuario llegó desde un push. Es la lista cerrada de la sección 5.
- **Ningún evento lleva dato clínico ni texto libre.** Ni diagnóstico, ni nombre de mascota, ni de persona, ni el `paciente_id`. La regla está en Reglas de Negocio, 2.7, y la sección 3 define la forma que la hace cumplible.
- **La telemetría nunca bloquea al usuario.** Si la ingesta falla, la acción del usuario sigue adelante. Un evento perdido es un dato menos; una carga clínica que no se guarda porque falló la métrica es un incidente.
- **Un evento que nadie mira se saca.** El catálogo de la sección 5 nombra la métrica que sale de cada evento. Si no hay métrica, no hay evento.

## 3. Qué se registra: Evento de telemetría

La entidad está definida en Modelo de Datos, 4.17. Lo que importa acá es por qué tiene esa forma:

- **`nombre` es un enum cerrado**, no un string libre. El catálogo de la sección 5 es el contrato: un nombre que no está en el enum se rechaza. Es lo que evita que dentro de seis meses convivan `evento_guardado`, `evento_cargado` y `clinical_event_saved` midiendo lo mismo.
- **`propiedades` es un JSON con claves permitidas por evento**, declaradas en el catálogo. El backend descarta las que no están en la lista en vez de guardarlas. Un JSON abierto es por dónde termina entrando el nombre de la mascota.
- **Dos relojes: `ocurrido_at` y `registrado_at`.** El primero es cuándo pasó según el cliente; el segundo, cuándo lo recibió el servidor. Se separan porque el tutor genera eventos sin conexión y los sube horas después (sección 7), y porque el reloj de un teléfono se puede correr. Toda métrica se calcula sobre `ocurrido_at`, con `registrado_at` como control: un `ocurrido_at` posterior al `registrado_at`, o más de 7 días anterior, se guarda igual pero queda marcado como sospechoso y se excluye de las series.
- **`plataforma`, `usuario_id`, `rol` y `clínica_id` son columnas del evento, no propiedades.** Las tres últimas salen del token. La plataforma viaja una vez por lote cuando la emite el cliente —un cliente es una sola plataforma—, y cuando la emite el servidor sale del **canal** con el que esa sesión se autenticó (`web` o `movil`, regla 2.3): el backend sabe desde qué canal le hablan, no qué sistema operativo tiene el aparato. Por eso la columna admite `web`, `ios`, `android` y `movil`, y la lectura de paridad suma los tres últimos del mismo lado. Queda en NULL en los eventos que emite un job, donde no hubo ningún cliente.
- **`sesión_id` es un identificador efímero que genera el cliente** al abrir la app y descarta al cerrarla. No es el token, no es el refresh, no sobrevive al reinicio: sirve para agrupar los eventos de un mismo uso y para contar sesiones, y para nada más.
- **`usuario_id` se guarda, y eso es dato personal.** Con una única clínica piloto, seudonimizarlo sería teatro: el plantel son unas pocas personas y cualquier corte por rol y fecha las identifica igual. Se asume que es dato personal, se lo trata como tal —queda alcanzado por el aviso de privacidad y por el plazo de retención de la sección 8— y se lo conserva porque sin él no hay retención por usuario ni cohortes, que es la mitad de lo que este documento existe para medir.
- **No tiene `deleted_at` ni auditoría.** Es un registro operativo, como Notificación (Modelo de Datos, 4.12): no es una entidad del dominio clínico que alguien dé de baja. Nadie lo edita nunca; la única baja es la del plazo de retención.

## 4. Cómo llega el evento

Un único punto de entrada: `POST /api/v1/telemetria`, autenticado, que acepta un **lote** de eventos y no uno solo. El proceso está en Reglas de Negocio, 4.20. Las decisiones:

- **Por lote, no por evento.** El cliente acumula y despacha; el móvil, además, tiene que poder subir lo que juntó offline. Un request por pantalla vista es un costo de red desproporcionado para el valor del dato.
- **La respuesta es siempre exitosa si el lote está bien formado**, aunque adentro haya eventos descartados por nombre desconocido o por propiedad no permitida. El cliente no reintenta ni corrige: descartar en silencio es correcto acá, y devolverle un error por un evento mal armado lo empujaría a reintentar en loop lo que nunca va a ser aceptado. Lo descartado se cuenta en el log del backend, que es donde se ve si un cliente nuevo está emitiendo mal.
- **El backend completa lo que no puede venir del cliente**: `usuario_id`, `rol` y `clínica_id` salen del token, nunca del cuerpo. Un cliente que afirma su propio rol es un cliente que puede ensuciar toda la serie con un typo.
- **Hay techo de eventos por lote y por usuario y ventana de tiempo.** Es la misma clase de límite que cualquier escritura de la API, y acá importa más porque el volumen lo decide un bucle del cliente, no una persona.

## 5. Catálogo de eventos

Cada fila es un evento del enum, con quién lo emite y la métrica que sostiene. Las propiedades listadas son la lista permitida completa: cualquier otra clave se descarta.

### 5.1 Veterinario — la métrica norte

| Evento | Emisor | Propiedades | Para qué |
|---|---|---|---|
| `evento_clinico_creado` | Backend | `tipo`, `con_cita` | **Eventos clínicos por veterinario por semana.** Es la métrica norte: si el veterinario no carga, no hay historial, y sin historial el tutor no tiene nada que ver. Todo lo demás de este documento es secundario. |
| `carga_evento_abierta` | Cliente | `plataforma` | Denominador del abandono y arranque del cronómetro. |
| `carga_evento_abandonada` | Cliente | `plataforma`, `duración_ms` | **Tasa de abandono del formulario clínico.** Un abandono alto con duración alta es un formulario largo; con duración baja, es un formulario que no se entiende. |
| `consulta_atendida_asentada` | Backend | `origen` (agendada / espontánea / urgencia), `automatica` (bool) | **El denominador de la cobertura.** Es el hecho asistencial (Modelo de Datos, 4.16), asentado al atender e independiente de que se cargue el historial. `automática` separa las que asentó una persona de las que dedujo el sistema de un evento con cita: solo las primeras informan algo. |
| `cita_creada` | Backend | `tipo`, `con_profesional` | Cuánto del calendario se agenda en Wayka, y cuánto se reparte al agendar. |
| `medicacion_creada`, `medicacion_finalizada` | Backend | — | Si la medicación se usa como registro vivo o se carga una vez y se abandona. |

`duración_ms` del abandono y del guardado se mide entre `carga_evento_abierta` y el cierre, en el cliente. Es el único cronómetro del catálogo y existe porque el tiempo de carga es lo que decide si el veterinario vuelve al papel.

### 5.2 Adopción de la clínica

| Evento | Emisor | Propiedades | Para qué |
|---|---|---|---|
| `sesion_iniciada` | Backend | `metodo` (contraseña / Google) | **Veterinarios activos por semana** sobre el plantel dado de alta. Con una clínica, se mira por persona: el promedio de cinco personas no dice nada, el día que una dejó de entrar sí. |
| `pantalla_vista` | Cliente | `pantalla` (enum del Alcance de Plataformas) | Qué partes del producto se usan y cuáles no llegó a abrir nadie. |
| `paciente_creado` | Backend | `origen` (clínica / tutor) | Crecimiento de la cartera y por qué vía entra. |

El corte `plataforma` de todos estos eventos es lo que valida la **paridad web/móvil** que exige el Alcance de Plataformas, sección 2: si el veterinario nunca carga desde el teléfono, la paridad se está pagando sin que nadie la use.

### 5.3 Tutor y efecto red

| Evento | Emisor | Propiedades | Para qué |
|---|---|---|---|
| `sesion_iniciada` | Backend | `metodo` | **Retención D7 / D30 del tutor**, cortada por si la sesión vino de un push o no (`app_abierta_desde_push`). Un tutor que solo entra cuando le avisan valora el aviso, no la app. |
| `app_abierta_desde_push` | Cliente | `notificacion_tipo` | Numerador de la tasa de apertura de push. El denominador es `notificacion_despachada`. |
| `notificacion_despachada` | Backend | `notificacion_tipo`, `resultado` (enviada / fallida) | Entregabilidad del canal. Sale del proceso de Reglas de Negocio, 4.15. |
| `notificaciones_desactivadas` | Cliente | — | **El mejor aviso temprano de fatiga.** Un tutor que apaga los avisos en Ajustes no se va todavía, pero se fue del único canal que lo traía de vuelta. |
| `adjunto_subido` | Backend | `tipo`, `rol_del_autor` | **Fichas con al menos un adjunto**: señal de que el historial se percibe como el archivo real de la mascota y no como una copia del de la clínica. |
| `invitacion_cotutor_enviada` | Backend | `nivel` | Denominador del embudo de co-tutor. |
| `invitacion_cotutor_canjeada` | Backend | `nivel`, `horas_hasta_canje` | **Embudo de co-tutor**: enviadas → canjeadas → vencidas. Es el único canal de crecimiento orgánico del MVP y ya está modelado (Modelo de Datos, 4.15). `cuenta_nueva` separa al que ya era usuario del que llegó por la invitación. |
| `mascota_compartida_con_clinica` | Backend | `origen` | Si el tutor usa el compartir con clínica o solo lo usa la clínica. |

### 5.4 Salud del offline y del rendimiento percibido

Las dos únicas medidas técnicas que entran acá, porque las dos deciden abandono y ninguna se ve desde el log del servidor.

| Evento | Emisor | Propiedades | Para qué |
|---|---|---|---|
| `sesion_servida_offline` | Cliente | `copia_caducada` (bool) | **Porcentaje de sesiones móviles servidas desde la copia local.** Es lo que dice si el offline resuelve algo real o si solo agrega complejidad. Con `copia_caducada`, además: cuántas veces el tutor llegó a la copia vencida a los 7 días (Sincronización Offline, 8). |
| `sincronizacion_conflicto` | Backend | `entidad`, `motivo` | **Conflictos por cada 100 sincronizaciones.** Es la parte más frágil del diseño (Sincronización Offline, 6) y la que más barato sale medir antes de que un usuario la sufra. Hoy el servidor siempre gana y el cliente vuelve a bajar el registro, así que no hay una resolución que registrar: lo que se mide es cuántas veces pasa. |
| `rechazo_de_mutacion` | Backend | `entidad`, `motivo` | Escrituras offline que el servidor rechazó al subir (Sincronización Offline, 7). Un motivo que se repite es una regla que el cliente no está aplicando y debería. |

## 6. Quién escribe y quién lee

| | |
|---|---|
| **Escribe** | Cada usuario, solo sus propios eventos, por la ruta de ingesta. El backend, en la capa de negocio, para todo lo que emite él. |
| **Lee** | Nadie por la API en el MVP. Se consulta por SQL contra la base, como el resto del análisis del piloto. |

Va también a la matriz de permisos del Modelo de Datos, sección 5. La ausencia de lectura por API es deliberada y no un pendiente: en cuanto exista un endpoint que devuelva telemetría, hay que decidir quién la ve, y la respuesta obvia —el clínica_admin, sobre su clínica— choca con que ese rol no tiene acceso al historial clínico (Modelo de Datos, sección 5). El día que haya panel, esa decisión se toma explícitamente.

## 7. Telemetría y offline

Los eventos que emite el cliente **se encolan en el dispositivo y suben con el mismo ciclo de sincronización** que las mutaciones (Sincronización Offline, 4.1 y 5), pero por su propia ruta y con dos diferencias:

- **La cola tiene techo y descarta lo más viejo.** Si el tutor pasó una semana sin señal, se pierden los eventos más antiguos antes que llenar el almacenamiento del teléfono. Una mutación no se descarta nunca; un evento de telemetría sí, y esa es la diferencia entre los dos.
- **Nunca reintenta indefinidamente ni bloquea la subida de mutaciones.** Si la ingesta falla, la cola se descarta al llegar al techo de intentos. El dato clínico va primero.

Los eventos que emite el backend no participan de esto: se escriben en el mismo request que atendió la acción.

## 8. Retención y privacidad

- **13 meses de detalle.** Alcanza para comparar contra el mismo mes del año anterior y para todas las cohortes que este documento define. Pasado el plazo, un job borra las filas y conserva únicamente agregados sin `usuario_id`.
- **El borrado es físico, y es la única excepción al "nunca borrado físico" del proyecto** (Reglas de Negocio, 2.4). La regla protege al dato clínico de una pérdida irreversible; acá el objetivo es el opuesto: guardar el rastro de uso de una persona más tiempo del que sirve es exactamente lo que la Ley 25.326 no quiere, y un borrado lógico no borra nada.
- **La telemetría queda cubierta por el consentimiento y por el aviso de privacidad** que ya asienta el Tutor (Modelo de Datos, 4.1). No hay un consentimiento separado para métricas en el MVP: sin proveedor externo, sin dato clínico en el evento y con plazo acotado, es tratamiento propio con finalidad declarada. Si alguna vez entra un tercero, esta decisión hay que rehacerla antes, no después.
- **Un pedido de supresión de datos borra también su telemetría.** Es dato personal del titular y ninguna obligación legal la retiene, a diferencia del historial clínico.

## 9. Las métricas que salen de esto

Definidas con numerador y denominador, porque una métrica sin denominador es una anécdota.

| Métrica | Cálculo | Salud esperada en el piloto |
|---|---|---|
| **Eventos clínicos por veterinario por semana** | `evento_clinico_creado` agrupado por `usuario_id` y semana | Serie estable o creciente, mirada por persona |
| **Veterinarios activos** | vets con ≥1 `sesion_iniciada` en la semana / plantel activo | > 70 % |
| **Cobertura** | consultas atendidas de la semana con ≥1 Evento clínico vinculado / consultas atendidas de la semana, contando solo las asentadas por una persona (`automática = false`) | Acercarse a 1,0 — es la forma de leer "documenta lo que atiende" |
| **Adopción del asiento** | consultas asentadas por una persona / total de consultas asentadas, y eventos clínicos con `consulta_id` NULL sobre el total | La lectura inversa: dice si la clínica está usando el asiento o si la cobertura se está midiendo sobre una muestra chica |
| **Tiempo de carga** | mediana de `duración_ms` de las cargas guardadas | < 60–90 s |
| **Abandono del formulario** | `carga_evento_abandonada` / `carga_evento_abierta` | Bajo y estable; un salto señala un cambio de UI que salió mal |
| **Activación del tutor** | tutores con ≥1 `pantalla_vista` de ficha dentro de los 7 días del alta / tutores dados de alta | — |
| **Retención D7 / D30 del tutor** | vuelve a haber `sesion_iniciada`, separando las precedidas por `app_abierta_desde_push` | La serie sin push es la que dice si el producto vale |
| **Apertura de push** | `app_abierta_desde_push` / `notificacion_despachada` con `resultado = enviada` | — |
| **Fuga de notificaciones** | `notificaciones_desactivadas` acumulado / tutores activos | Cualquier tendencia al alza es una alarma |
| **Embudo de co-tutor** | canjeadas / enviadas. Las vencidas no salen de un evento: se cuentan sobre `invitacion_de_co_tutor` (`expira_at` pasado, sin `usado_at`) | — |
| **Fichas con adjunto** | pacientes con ≥1 `adjunto_subido` / pacientes activos | — |
| **Sesiones offline** | `sesion_servida_offline` / sesiones móviles del tutor | Si es ~0, el offline no está resolviendo nada |
| **Conflictos** | `sincronizacion_conflicto` × 100 / sincronizaciones, y `rechazo_de_mutacion` aparte | Un motivo de rechazo que se repite es una regla que el cliente no está aplicando |

**La cobertura es la métrica que decide el lanzamiento**, y ahora se apoya en un hecho propio y no en un proxy: la **Consulta atendida** (Modelo de Datos, 4.16) se asienta al atender, en un toque, y no depende de que la atención estuviera agendada ni de que el historial se cargue el mismo día. Eso es lo que la hace un denominador y no una derivación del numerador.

Queda una limitación honesta y hay que leerla junto con la métrica: **el denominador sigue siendo lo que la clínica asienta, no lo que atiende**. Un profesional que no asienta nada tampoco aparece en el numerador, así que la cobertura no lo penaliza: lo que lo delata es la *adopción del asiento*, y por eso las dos filas van juntas. En el piloto, además, el número se contrasta una vez por semana contra el conteo que lleva la propia clínica — un control que sirve mientras haya una sola y que no escala, y por eso es del piloto y no del producto.

## 10. Fuera de alcance de este documento

- **Cómo se consultan y se grafican las métricas** — herramienta de análisis, panel, alertas.
- **Sala de espera y estado de la atención** — el asiento registra que se atendió, no un ciclo de admisión, en curso y alta. Modelar el flujo de la recepción es otro producto.
- **Métricas de negocio que no son de uso** — facturación, costo por clínica, conversión comercial.
- **Observabilidad técnica** — latencias, errores y trazas del backend (`backend/docs/07`).
- **Experimentación A/B** — no hay motor de experimentos ni asignación de variantes en el MVP.
