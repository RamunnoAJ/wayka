# Wayka

Historial clínico colaborativo y calendario para veterinarias. MVP con una clínica piloto, modelo B2B. Tres tipos de usuario: **Clínica_admin** (web), **Veterinario** (web y móvil, con paridad total) y **Tutor** (solo móvil).

## Cómo está organizado este directorio

No es un monorepo con un solo repo git: cada subproyecto tiene su propio historial y su propio remote. Este repo raíz versiona únicamente `/docs` y este archivo.

| Carpeta | Qué es | Git |
|---|---|---|
| `backend/` | API en Go (chi, sqlc, goose, oapi-codegen, Postgres) | repo propio → `RamunnoAJ/wayka-backend` |
| `frontend/` | Cliente único web + móvil (Expo, React Native Web, Expo Router) | repo propio, todavía sin remote |
| `docs/` | Contrato de producto compartido por los dos | este repo |

**Cada subproyecto tiene su propio `CLAUDE.md`** con su stack, su arquitectura y sus principios no negociables — es la fuente de verdad de cómo se trabaja ahí:

- [backend/CLAUDE.md](backend/CLAUDE.md)
- [frontend/CLAUDE.md](frontend/CLAUDE.md)

Leerlo antes de tocar código de ese subproyecto. Lo de acá no lo reemplaza.

## Documentos de referencia

Los cinco son **contrato de producto**, no de implementación: valen igual para el backend y para el frontend, y por eso viven en una sola copia acá. Estuvieron duplicados en los dos repos y divergieron; no volver a copiarlos.

1. **[docs/01-modelo-de-datos.md](docs/01-modelo-de-datos.md)** — entidades, campos, relaciones, campos transversales
2. **[docs/02-reglas-de-negocio.md](docs/02-reglas-de-negocio.md)** — validaciones, motor de permisos, procesos de negocio
3. **[docs/03-alcance-de-plataformas.md](docs/03-alcance-de-plataformas.md)** — qué rol accede a qué pantalla, desde qué plataforma
4. **[docs/04-arquitectura.md](docs/04-arquitectura.md)** — capas, autenticación, entornos, rutas y CORS
5. **[docs/11-sincronizacion-offline.md](docs/11-sincronizacion-offline.md)** — acceso sin conexión del tutor en móvil, bitácora de cambios, resolución de conflictos y datos en el dispositivo

Propios de cada subproyecto: `backend/docs/` (05 stack, 06 estándares, 07 logging) y `frontend/docs/` (08 arquitectura frontend, 09 design system).

## Al trabajar desde esta raíz

- **El backend es la única barrera de seguridad.** El frontend valida por UX; el motor de permisos (rol + alcance) y las reglas de negocio se aplican solo en el backend.
- **Un cambio de contrato toca tres lugares y no es atómico**: el doc en `docs/`, el backend y el frontend viven en repos separados. Si una tarea cambia una regla de negocio o el modelo de datos, actualizar el doc en el mismo pasaje, no después.
- **Si una funcionalidad requiere una regla no contemplada en `docs/`, señalarlo explícitamente** en vez de asumir un comportamiento — los documentos son el contrato del proyecto, no una sugerencia. Mismo criterio en ambos subproyectos.
- El orden de trabajo de una feature completa es contrato primero: `docs/` → `backend/openapi/openapi.yaml` → migración goose → backend (rojo/verde/refactor) → frontend.
