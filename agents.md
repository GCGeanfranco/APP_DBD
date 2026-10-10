# agents.md — Constitution del proyecto APP DBD

Reglas permanentes para cualquier agente (Coordinator, Planner, Implementor, Reviewer) que trabaje en este repositorio. Se escribe una sola vez, antes de la primera `/spec`, y se actualiza rara vez y a mano — no por un comando automático (ver `02_metodologia_sdd.md`, Fase 0).

Ninguna spec, diseño o implementación puede contradecir este archivo. Si una ambigüedad reaparece varias veces para el mismo tema, se resuelve una vez y se agrega aquí como regla, en vez de repetirse en cada `/spec`.

## 1. Qué es este proyecto

Réplica profesional de **DBD Streamer**: panel de control + overlays en tiempo real (OBS) para streamers de Dead by Daylight. Se parte de una aplicación existente (ver referencia completa en `07_app_dbd_streamer_referencia.md`) con deuda técnica conocida, que se corrige durante la réplica, no se hereda.

## 2. Alcance de la v1

### Rutas
- `#/` — panel de control del streamer (con login)
- `#/overlay` — overlay principal para OBS (sin login, token por variable de build)
- `#/chat-window` — ventana de chat independiente, **solo Twitch por ahora** (YouTube/TikTok/Kick queda fuera de la v1)

**Fuera de alcance de la v1:** `#/alerts-overlay`.

### Widgets (7)
`stream-timer`, `killer-roulette`, `perk-roulette`, `match-stats`, `twitch-chat-queue`, `palomo-chase`, `scrolling-ticker`.

**Fuera de alcance de la v1:** `multiplier-wheel` (ruleta de multiplicador), `twitch-events` (alertas de Twitch: follow/sub/prime_sub/resub/bits/gift_sub/raid), bot de chat (comandos, mensajes programados, respuestas automáticas), `unified-chat` (chat multiplataforma).

### Reglas de widgets para la v1
- `stream-timer` mantiene su mecanismo original: cuenta regresiva calculada contra un timestamp objetivo (no restando segundos), extendida automáticamente por eventos de Twitch vía EventSub. Esta extensión es parte del widget, no una funcionalidad separada.
- EventSub de Twitch (bits, follow, gift, raid) sigue activo igual que en el original, aunque se quite el widget `twitch-events` — solo deja de alimentar popups de alerta.
- Redimensionables en la v1: `stream-timer`, `killer-roulette`, `perk-roulette`, `match-stats`, `twitch-chat-queue`, `palomo-chase`. (`match-stats`, `twitch-chat-queue` y `palomo-chase` no eran redimensionables en el original; en la réplica sí deben serlo.)
- No redimensionable: `scrolling-ticker`.
- Patrón `isController` se mantiene: el panel pasa `true` (corre timers/efectos), el overlay pasa `false` (solo renderiza). Ningún widget corre su lógica dos veces.

El alcance puede ampliarse en cualquier momento (ver `/projects/.../areas/alcance-replica.md` para el detalle y el historial de recortes). Cada ampliación pasa por su propio ciclo `/spec` → `/design` → `/plan` → `/implement`, nunca se agrega directo en código.

## 3. Stack permitido

| Capa | Servicio |
|---|---|
| Base de datos principal | Neon (Postgres) |
| Base de datos secundaria (solo si se necesita Storage/Realtime) | Supabase |
| Caché / estado en vivo / tokens | Upstash Redis |
| Frontend | Vercel |
| Backend | Render (free, con estado externalizado) |
| DNS / protección / túnel | Cloudflare (nunca para el servidor en sí) |
| Imágenes/assets | Cloudinary |
| Autenticación del panel | Clerk (separado del OAuth de Twitch) |
| Monitoreo | Sentry |
| Testing E2E | Playwright MCP oficial (Microsoft) |
| Testing unitario | Vitest |

No se introduce ningún servicio fuera de esta lista sin registrar la decisión en `08_decisiones.md`. Detalle completo y motivos en `01_stack_y_conectores.md`.

## 4. Convenciones de código

- **Idioma:** código (nombres de variables, funciones, clases, tipos) y mensajes de commit en **inglés**. Specs, diseños, comentarios en el código y toda la documentación del proyecto en **español**.
- Backend y frontend en TypeScript. El backend corre en Node sin build step directo para `server.js`-equivalente; cualquier módulo compartido con el frontend debe ser ejecutable por Node sin build (ver lección aprendida en la prueba T3: verificar siempre qué copia realmente el Dockerfile/build a producción, no asumirlo).
- Dividir el backend por dominios (auth, twitch, chat, estado, uploads) desde el diseño inicial — no repetir el monolito de ~2000 líneas del original.
- TypeScript estricto (`strict: true`) en todo el proyecto, a diferencia del original.
- Toda funcionalidad nueva o modificada lleva pruebas unitarias (Vitest) antes de pasar a revisión.

## 5. Reglas de seguridad

- Nunca commitear secretos ni pegarlos en documentación, specs o chats de agentes. Usar `.env` + el gestor de secretos de cada plataforma.
- No compartir tokens en texto plano entre herramientas (Claude Desktop, OpenCode, MCPs).
- El streamer nunca realiza acciones relacionadas con el hosting (keep-alive, reconexión, persistencia): todo eso es responsabilidad del sistema.

## 6. Agentes: quién hace qué

| Agente | Herramienta / modelo | Hace | No hace |
|---|---|---|---|
| Coordinator | OpenCode, agente primario | Decide la fase SDD, delega, consolida, único que habla con el usuario | No escribe código de producción directamente |
| Planner | Claude Desktop, Sonnet 5.5 (o Sonnet 5 para specs simples) | Convierte spec aprobada en diseño técnico + tareas | No escribe código de producción |
| Implementor | OpenCode, subagente. Principal: **Big Pickle** (OpenCode Zen). Respaldo: **DeepSeek V4.1 Flash** (NVIDIA) | Ejecuta una tarea a la vez en rama propia | No modifica la spec; si hay contradicción, la reporta al coordinator |
| Reviewer | Claude Desktop, Sonnet 5 (rutina) o Sonnet 5.5 (crítico/arquitectura) | Revisa diff contra spec y diseño, clasifica hallazgos | No corrige código directamente |

**Regla fija:** implementor y reviewer nunca son el mismo modelo.

**Mitigación activa (no opcional):** Big Pickle tuvo dos reportes falsos durante la prueba comparativa (afirmó cambios aplicados que no existían, con referencias a código inventado, antes de un tercer intento correcto). El reviewer debe verificar sus entregas contra el diff real del repo con más rigor que al resto — nunca aceptar "criterios cumplidos" solo por el reporte del propio modelo.

Límite de iteraciones implementor↔reviewer: **3**. Si no se cumple la spec tras 3 iteraciones, el caso sube al usuario — nunca se cambia la spec para ajustarla al código.

## 7. Metodología SDD

Fases, en orden, cada una con su comando: `/spec` → `/design` → `/plan` → `/implement` → `/review` → `/test` → `/close`. Detalle completo de cada fase en `02_metodologia_sdd.md`.

- Ninguna fase de implementación empieza sin spec y diseño aprobados explícitamente por el usuario.
- Cada tarea cabe en una sesión de trabajo y tiene criterios de aceptación verificables.
- Todo cambio se trabaja en rama; el reviewer valida antes de fusionar.
- Las decisiones importantes se registran en `memory.md` del repo, no solo en el chat.
- Clarify (resolución de ambigüedades) ocurre dentro de `/spec` y `/design`, en el chat, antes de la aprobación — no es una fase ni un documento aparte.

## 8. Gestión del proyecto

- **Trello** (Kanban) es el tablero único. Columnas: Backlog → Spec → Diseño → Por hacer → En progreso → En revisión → Testeo → Hecho. Límite de trabajo en progreso: 2 tarjetas.
- **Miro** solo para diagramas de arquitectura/flujos, nunca para gestionar tareas.
- Jira descartado.
- Cada tarjeta se vincula con su rama y commit.

## 9. Reglas aprendidas (de la prueba comparativa de implementores)

- Nunca registrar un resultado o una entrega solo por el reporte del modelo que la hizo: verificar siempre con Desktop Commander contra el diff real y los comandos ejecutados.
- Antes de verificar una entrega del implementor, confirmar explícitamente dónde vive y qué copia el Dockerfile/proceso de build a producción — una solución que pasa tests en local puede no sobrevivir en el contenedor real.
- Ignorar diffs de fin de línea (CRLF/LF) al comparar ramas si el repo usa `core.autocrlf=true`.

## 10. Render free — reglas de operación

- Estado externalizado en Redis (Upstash) y Neon — nada crítico en disco local.
- Assets (GIF, sonidos, avatar) en Cloudinary, nunca en disco del servidor.
- Keep-alive externo (UptimeRobot o cron-job.org) contra `/health`, configurado una sola vez por el desarrollador.
- Reconexión automática con backoff exponencial para IRC, EventSub y WebSocket.
- `/health` debe reportar el estado real de cada conexión (Twitch, EventSub, chat).

Detalle completo y plan de pruebas en `01_stack_y_conectores.md`.
