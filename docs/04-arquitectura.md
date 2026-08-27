# Wayka — Arquitectura del Sistema

MVP — Backend, plataformas y autenticación
Versión 1.0 · Complementa a Modelo de Datos, Reglas de Negocio y Alcance de Plataformas

## 1. Alcance

Este documento define cómo se organizan los componentes técnicos de Wayka y cómo se comunican entre sí, para el MVP con una clínica piloto. No define tecnologías ni frameworks específicos — esa decisión queda pendiente para una etapa posterior.

## 2. Estilo arquitectónico: monolito modular

Wayka usa un único backend compartido por la web y la aplicación móvil, organizado en capas internas claramente separadas, en lugar de microservicios independientes.

Justificación:

- **Paridad funcional del veterinario.** Web y móvil ofrecen el mismo alcance para ese rol (Alcance de Plataformas, sección 2). Si cada plataforma tuviera su propio backend, el motor de permisos y las reglas de negocio se duplicarían y podrían desincronizarse con el tiempo.
- **Escala del equipo y del piloto.** Un equipo reducido desarrollando para una única clínica no se beneficia de la complejidad operativa de microservicios (despliegues independientes, comunicación entre servicios, observabilidad distribuida).
- **Modularidad interna preservada.** La separación en capas (sección 3) permite, si el proyecto crece en Fase 2, extraer un módulo a un servicio independiente sin rediseñar el sistema desde cero.

## 3. Componentes

| Componente | Responsabilidad |
|---|---|
| Cliente Web | Interfaz de uso exclusivo de la clínica (Veterinario, Clínica_admin). Sin lógica de negocio ni de permisos — toda decisión se valida en el backend. |
| Cliente Móvil | Interfaz para Veterinario (paridad con web) y Tutor (acceso exclusivo). Misma regla: sin lógica de negocio propia. |
| Backend — Capa de presentación / API | Recibe requests, autentica (emite y valida tokens), aplica el bloqueo de canal (Reglas de Negocio, 2.3). |
| Backend — Capa de negocio | Implementa el motor de permisos, los servicios de dominio y la auditoría automática — es la capa que aplica el documento de Reglas de Negocio. |
| Backend — Job de notificaciones | Proceso interno que encola los recordatorios de Citas próximas y despacha los que ya corresponden, contra el proveedor de push (Reglas de Negocio, 4.15). Como el de citas vencidas, entra a la capa de negocio directamente. |
| Proveedor de notificaciones push | Expo Push, alcanzado por HTTP desde el backend. El envío es responsabilidad del servidor: la app Expo solo obtiene el token del aparato y lo registra contra la API. Hay dos adaptadores detrás de la misma interfaz de negocio y se eligen por configuración (`PUSH_PROVEEDOR`): el de Expo y uno que registra el envío en el log, que es el default para que un despliegue mal configurado deje rastro en vez de avisar a los teléfonos equivocados. |
| Backend — Job programado | Proceso interno que transiciona Citas vencidas, invocando la capa de negocio directamente (Reglas de Negocio, 4.6). Corre dentro del mismo proceso que la API, en una goroutine que vive lo que vive el proceso. En el MVP hay una sola instancia del backend; si se despliega más de una, dos jobs pueden solaparse — la transición es idempotente (vencer una cita ya vencida no la cambia), pero repartir el trabajo o tomar un lock queda por decidir en ese momento. |
| Backend — Herramienta de administración | Comando de línea de comandos que da de alta una Clínica y su cuenta clínica_admin (Reglas de Negocio, 4.10). Como el job, entra a la capa de negocio directamente, sin pasar por la API: no hay sesión de usuario que autenticar y la operación no debe quedar expuesta por HTTP. |
| Backend — Capa de datos | Repositorios que implementan el Modelo de Datos — una entidad por repositorio. |
| Base de datos relacional | Persiste todas las entidades del Modelo de Datos. |
| Almacenamiento de archivos | Bucket privado compatible con S3. Persiste los Adjuntos (fotos, PDFs, estudios) por separado de la base relacional. El cliente nunca le habla directo para escribir: sube al backend y el backend al bucket (sección 3.2). |

### 3.1 Diagrama de componentes

```
Web (clínica) ──────┐
                    ├──> Backend (monolito modular)
Móvil (vet + tutor) ┘         │
                               ├─ Capa de presentación / API
                               │     (auth, bloqueo de canal)
                               │
                               ├─ Capa de negocio
                               │     (permisos, servicios, auditoría)
                               │     ├─ Job programado
                               │     │     (citas vencidas, recordatorios)
                               │     └─ Herramienta de administración
                               │           (alta de clínica + clínica_admin)
                               │
                               └─ Capa de datos (repositorios)
                                      │
                        ┌─────────────┴─────────────┐
                        ▼                           ▼
          Base de datos relacional      Almacenamiento de archivos
```

### 3.2 Acceso al almacenamiento de archivos

El bucket es privado y el backend es el único que tiene credenciales sobre él. De ahí salen las dos mitades del flujo:

- **Subida por el backend.** El cliente manda el archivo al backend (multipart) y el backend lo sube al bucket. La alternativa —URL prefirmada de subida, con el cliente escribiendo directo en S3— evita que los bytes pasen por el backend, pero parte el alta en dos pasos que pueden desincronizarse (un objeto subido que nadie confirma, o un registro confirmado sin objeto) y deja al backend validando un archivo que nunca vio. Con el volumen de una clínica piloto, esa complejidad no se paga.
- **Descarga por URL prefirmada.** El backend evalúa los permisos y devuelve una URL de vida corta contra el bucket, en vez de hacer stream del archivo. Acá sí conviene: la lectura es lo frecuente (una ficha con fotos las pide todas juntas) y la URL vive minutos. La contrapartida asumida es que, dentro de esa ventana, la URL funciona aunque al usuario se le revoque el acceso.

La vida de la URL prefirmada (`ARCHIVOS_URL_TTL`), el tamaño máximo por archivo (`ARCHIVOS_TAMANO_MAXIMO`) y las credenciales del bucket son configuración de entorno, no constantes del código.

### 3.3 Envío de notificaciones push

La API de Expo (`POST /--/api/v2/push/send`) acepta hasta 100 mensajes por pedido y contesta un resultado **por mensaje**, no por pedido. De ahí salen tres decisiones del adaptador:

- **Un lote que falla no cancela a los demás.** El aviso que ya salió a un teléfono no se puede deshacer; devolver un error haría que la notificación se reintente entera y duplique el aviso donde sí llegó. Solo se reporta el fallo cuando no salió ningún lote.
- **`DeviceNotRegistered` da de baja el Dispositivo.** Es el único error de Expo que significa que el aparato dejó de existir; el resto son transitorios o de configuración y sí ameritan reintento.
- **La autenticación es opcional.** El endpoint acepta pedidos sin credenciales; el token de acceso (`PUSH_EXPO_ACCESS_TOKEN`) solo hace falta si la cuenta de Expo tiene activada la seguridad reforzada. Por eso el adaptador se puede desplegar y probar sin secretos.

Los *receipts* de Expo —los errores que el proveedor publica después de aceptar el mensaje— quedan fuera del MVP: el backend se entera de un teléfono muerto en el envío siguiente, no en el mismo.

### 3.4 Rutas y versionado de la API

Todo el contrato cuelga de `/api/v1`. Dos consecuencias deliberadas:

- **El prefijo lo aplica el router, no cada operación.** El YAML declara las rutas relativas y el prefijo va en su bloque `servers`, en un solo lugar del código. Repetirlo en cada path del contrato es la forma de que, tarde o temprano, un endpoint nuevo se lo olvide.
- **Convivencia de versiones.** Cuando exista una v2, ambas se montan sobre el mismo backend con prefijos distintos. Es lo que permite que la app móvil instalada —que no se actualiza cuando nosotros desplegamos— siga hablando con la v1 mientras la web ya usa la v2.

Quedan fuera del prefijo, en la raíz, las rutas que no son recursos de negocio:

| Ruta | Por qué en la raíz |
|---|---|
| `/health` | Lo consulta el balanceador o el orquestador, no un cliente. Su ruta es parte del despliegue, y no tiene por qué mudarse cuando cambie la versión de la API. También responde dentro del prefijo, porque está declarado en el contrato. |
| `/openapi.yaml`, `/docs` | Son el contrato y su visor, no algo que el contrato describa. |

### 3.5 CORS

El cliente web corre en otro origen que el backend, así que el navegador exige una autorización explícita. Las decisiones:

- **Fuera de producción se autoriza cualquier origen; en producción, solo los de `CORS_ORIGENES`.** Un entorno de producción sin la variable configurada no autoriza a ninguno: la falla tiene que ser cerrada, no permisiva.
- **No se habilitan credenciales.** El token de acceso viaja en `Authorization` y no en una cookie, así que permitirlas solo agregaría superficie de CSRF sin que nada lo necesite. Si alguna vez aparece una cookie de sesión, esta decisión hay que revisarla junto con el resto.
- **Un pedido de un origen no autorizado no se rechaza en el servidor.** CORS lo bloquea en el navegador; cortarlo acá daría la falsa impresión de ser una barrera de seguridad, cuando un cliente que no sea un navegador lo ignora por completo. La barrera real sigue siendo el motor de permisos.
- **El preflight lo contesta el middleware**, antes de la autenticación y sin llegar al router: viaja sin token y ninguna ruta del contrato declara `OPTIONS`, así que el router lo rechazaría con un 405 que el navegador reporta como un CORS mal configurado.
- `Authorization` se declara explícitamente en `Access-Control-Allow-Headers`: el comodín `*` de la especificación no la cubre, y sin ella el token nunca sale del navegador.

## 4. Autenticación

Wayka usa autenticación basada en tokens, con un token de acceso de vida corta y un token de refresco de vida más larga — el estándar para sistemas que sirven a web y móvil desde un mismo backend.

### 4.1 Token de acceso
- Vida corta: **15 minutos** por defecto (`ACCESS_TOKEN_TTL`).
- Se envía en cada request en el header `Authorization: Bearer <token>` y se valida por firma — el backend no consulta la base de datos para verificarlo.
- Es un JWT firmado con HS256. El algoritmo esperado se fija al validar: un token que declare otro algoritmo (incluido `none`) se rechaza sin mirar su contenido.
- Contiene la información necesaria para el motor de permisos: tipo_usuario y el id de la entidad asociada (tutor_id / veterinario_id / clínica_id), más el canal desde el que se emitió.
- Un request sin header `Authorization` sigue su curso sin usuario autenticado y es la capa de negocio la que lo rechaza; uno con un token presente pero inválido se corta con 401 en la capa de presentación. Así las rutas públicas no tienen que enumerarse en el middleware, y un token vencido no se confunde con un permiso denegado.

### 4.2 Token de refresco
- Vida más larga: **30 días** por defecto (`REFRESH_TOKEN_TTL`).
- Es un valor opaco (256 bits de aleatoriedad), no un JWT: no transporta información, solo sirve para encontrar su sesión.
- Queda registrado en el backend — es el único punto con estado en este esquema. Se persiste **el hash SHA-256 del token, nunca el token**: leer la tabla no debe alcanzar para hacerse pasar por un usuario. El hash es además la clave de búsqueda de la sesión, y por eso no se usa bcrypt: el token es aleatorio, no una contraseña elegida por una persona, así que no hay diccionario que un hash lento pueda encarecer.
- Cada vez que se usa para pedir un nuevo token de acceso, se valida Usuario.activo. Si está en false, se rechaza, se revoca la cadena (4.2.1) y el acceso se corta ahí.
- El canal no se vuelve a declarar al refrescar: se hereda de la sesión, para que renovar no sea una vía de mudar una sesión de móvil a web y evadir el bloqueo de canal.

#### 4.2.1 Rotación y detección de reuso

Cada refresco emite un token nuevo e invalida el usado — un token de refresco es de un solo uso. Las sesiones que se suceden por rotación comparten un identificador de cadena.

Si llega un token de refresco **ya canjeado**, es el indicio de que alguien tiene una copia: no hay forma de saber si quien lo presenta es el usuario legítimo o quien se lo robó, así que se revoca la cadena entera y ambos deben volver a autenticarse. Cerrar sesión revoca la cadena completa por el mismo motivo: revocar solo el último token dejaría vivos los anteriores.

La contrapartida asumida es que un cliente que pierda la respuesta de un refresco (por un corte de red en el momento exacto del canje) queda sin sesión y debe volver a autenticarse. Es preferible a no poder distinguir un token robado de uno legítimo.

### 4.2.2 Token de activación

Credencial de un solo uso con la que una cuenta de clínica_admin recién creada define su primera contraseña (Reglas de Negocio, 4.16). Como la tabla de sesión, **no es una entidad de dominio**: no la lee ni la escribe ninguna regla fuera de este proceso, no se audita y no aparece en el motor de permisos.

- Valor opaco de 256 bits de aleatoriedad, generado por la línea de comandos que da de alta la clínica. Se muestra una sola vez y no se puede volver a consultar.
- Se persiste **el hash SHA-256, nunca el token**, con el mismo criterio que el token de refresco: leer la tabla no puede alcanzar para activar una cuenta ajena. Y por el mismo motivo no es bcrypt — el valor es aleatorio, no una contraseña elegida por una persona, así que no hay diccionario que encarecer.
- Guarda a qué `usuario_id` pertenece, cuándo vence y cuándo se usó. Vida por defecto: **7 días** (`ACTIVACION_TOKEN_TTL`), holgada para el traspaso manual entre el administrador de la plataforma y la clínica, y corta frente a un token que quede olvidado en un chat.
- Un solo uso: el canje marca el token como usado en la misma transacción en que escribe el `password_hash`. Las dos escrituras van juntas o no va ninguna — un token consumido sin contraseña escrita deja la cuenta inaccesible para siempre.
- Un token usado o vencido **no se borra**: queda como rastro de que la activación ocurrió y cuándo.

> Todas las condiciones de rechazo (inexistente, vencido, ya usado, cuenta inactiva) devuelven el mismo error genérico. Distinguirlas le diría a quien esté probando valores al azar cuál de ellos existió alguna vez.

### 4.3 Ventana de revocación

Con este esquema, desactivar a un Usuario (por ejemplo, un Veterinario que deja la clínica) no corta el acceso de forma instantánea: sigue siendo válido hasta que expira su token de acceso vigente. La ventana de exposición queda acotada a la vida del token de acceso (minutos, no días) — es un balance consciente entre seguridad y simplicidad, no un descuido.

### 4.4 Relación con el bloqueo de canal

El bloqueo de canal (Reglas de Negocio, sección 2.3 — el tutor no puede autenticarse desde web, el clínica_admin no desde móvil) se valida en el mismo paso donde se emite el token: si el tipo_usuario no coincide con el canal declarado, no se emite el token y el login se rechaza.

El canal viaja como un campo `canal` (`web` | `movil`) en el cuerpo del login. **Lo declara el cliente**: un atacante que arme el request a mano puede afirmar cualquier canal, así que esto no es una barrera de seguridad sino la aplicación de una regla de producto en el único lugar donde el backend puede aplicarla. Convertirlo en una barrera real exigiría credenciales de cliente por canal (un client_id con secreto por aplicación), y quedó fuera del alcance del MVP: el secreto de una app móvil es extraíble de todos modos, y el riesgo que cubriría — un tutor entrando por web a datos que ya son suyos — no lo justifica todavía.

### 4.5 Rutas públicas

Casi toda la API exige un token de acceso válido. Las excepciones son el login (con contraseña o con Google), la renovación de token, el cierre de sesión y el **registro de tutor** (Reglas de Negocio, 4.9), que por definición ocurren antes de que exista un token. El registro de tutor es el único alta de cuenta que no requiere un usuario autenticado; el resto de las altas pasan por el motor de permisos como cualquier otra escritura.

### 4.6 Autenticación con Google (OAuth)

Google es un método de autenticación alternativo a email + contraseña, no un canal distinto: un Usuario autenticado vía Google recibe el mismo token de acceso + refresco que uno autenticado por contraseña, y queda sujeto exactamente al mismo bloqueo de canal (sección 4.4) según su tipo_usuario.

El backend verifica el ID token de Google (firma, expiración, audiencia) antes de resolver a qué Usuario corresponde — nunca confía en un email enviado sin esa verificación. El proceso completo de alta/vinculación de cuenta está definido en Reglas de Negocio, sección 4.7.

## 5. Entornos

Dos entornos para el MVP: desarrollo y producción. Cada uno con su propia base de datos y configuración, sin entornos intermedios (staging) por ahora — apropiado para el alcance de un piloto de una clínica, sin necesidad de infraestructura adicional hasta validar el producto.

## 6. Fuera de alcance de este documento

- **Stack tecnológico** — lenguajes, frameworks y librerías específicas (pendiente, a definir por separado).
- **Infraestructura de despliegue y CI/CD** — cómo se publican los cambios a cada entorno.
- **Escalabilidad más allá del piloto** — esta arquitectura está pensada para una clínica; no contempla múltiples instancias ni balanceo de carga.
- **Vista de paciente derivado en urgencia y matching geolocalizado** — pertenecen a la Fase 2 del proyecto.
