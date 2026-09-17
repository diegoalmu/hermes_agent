# Fase 9 — Caída de NVIDIA NIM, migración a Gemini y limpieza de memoria

## Contexto

En un mismo día se presentaron 3 incidentes relacionados entre sí: un pendiente que Leonardo repetía aunque ya se le había pedido borrarlo, seguido de fallas de cron jobs con errores de "Function id not found" al intentar diagnosticarlo.

## Incidente 1 — NVIDIA NIM caído (lado del proveedor, no de Hermes)

**Síntoma:** cron jobs fallando con:
```
HTTP 404: {"status":404,"title":"Not Found","detail":"Function id '948fe171-...' version 'null':
Specified function in account '...' is not found"}
```

**Diagnóstico:** se confirmó llamando directo a la API de NVIDIA sin pasar por Hermes:
```bash
docker exec hermes sh -c 'curl -s -X POST https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Authorization: Bearer $(grep NVIDIA_API_KEY /opt/data/.env | cut -d= -f2)" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"nvidia/nemotron-3-ultra-550b-a55b\",\"messages\":[{\"role\":\"user\",\"content\":\"hola\"}],\"max_tokens\":10}"'
```
Mismo error reproducido fuera de Hermes → confirma que es una falla del lado de NVIDIA, no de la configuración. Se encontraron múltiples reportes idénticos en el foro oficial de desarrolladores de NVIDIA (afectando a distintos modelos NIM, no solo Nemotron), algunos resueltos por NVIDIA re-desplegando el modelo, sin ETA garantizado.

**Decisión:** migrar temporalmente el proveedor principal a **Gemini** (Diego ya tenía una API key de Google AI Studio configurada desde el TTS).

## Incidente 2 — Modelo de Gemini deprecado el mismo día

Al cambiar a Gemini con `hermes model`, se seleccionó `gemini-2.5-flash-lite`. Los cron jobs fallaron con:
```
Gemini HTTP 404 (NOT_FOUND): This model models/gemini-2.5-flash-lite is no longer available
to new users. Please update your code to use models/gemini-3.5-flash-lite...
```
Google deprecó ese modelo específico para cuentas nuevas justo el mismo día. Se corrigió a `gemini-3.5-flash-lite`:
```bash
docker exec hermes hermes config set model.default gemini-3.5-flash-lite
```

### Mecanismo de seguridad descubierto: "drift_skip"

Al cambiar el modelo global, los cron jobs ya creados (que no tenían modelo "fijado") se negaron a ejecutarse con un aviso explícito:
```
RuntimeError: Skipped to prevent unintended spend: global inference config drifted since
this job was created (provider 'nvidia' -> 'gemini'; ...), and this job is unpinned.
```
Esto es una protección intencional de Hermes contra gasto no planeado al cambiar de proveedor — no es un bug. Se resolvió fijando explícitamente el proveedor/modelo en cada uno de los 7 cron jobs activos:
```bash
docker exec hermes hermes cron edit <job_id> --provider gemini --model gemini-3.5-flash-lite
```
(repetido para los 7 jobs: 3 recordatorios de clase, 2 de Infotec, Pendientes-Trabajo-Diario, y un recordatorio puntual de "Cálculo - Sesión 17 sep").

**Lección para el futuro:** cada vez que se cambie el modelo/proveedor global, hay que re-fijar todos los cron jobs existentes con `cron edit --provider --model`, o se quedan "huérfanos" esperando confirmación.

## Incidente 3 — Memoria persistente con contenido de prueba mezclado

Al revisar por qué la lista de pendientes de trabajo se comportaba de forma inconsistente, se encontró que `MEMORY.md` y `USER.md` tenían líneas de prueba/depuración mezcladas con el contenido real (`test`, `dummy`, `dummy2`, `[SILENT]`, `read`, `check`, `read_memory`, `list_pendientes`), probablemente residuo de cuando se armó la skill de pendientes inicialmente.

**Solución:** reescritura limpia de ambos archivos, conservando solo el contenido real:
```bash
docker exec -i hermes tee /opt/data/memories/MEMORY.md > /dev/null << 'EOF'
Pendientes de trabajo (ZUMMAT):

EOF

docker exec -i hermes tee /opt/data/memories/USER.md > /dev/null << 'EOF'
Pendientes de oficina de Diego (lista no priorizada): (ninguno — todos completados)
§
Usuario: Diego (Endor_alexander). Estudiante TIC en UNRC. Trabaja en ZUMMAT. Usa Telegram
como canal principal. Prefiere respuestas directas y concisas. Quiere gestión persistente
de pendientes de oficina y recordatorios de clases vía cronjobs.
§
Estilo de trabajo: lista de tareas no priorizada por número, elimina items completados en
bloque, usa recordatorios 30 min antes de eventos (clases).
EOF

docker restart hermes
```

## Mejora adicional: quitar el envoltorio "Cronjob Response / To stop or manage" de los mensajes

Los avisos de cron llegaban con un encabezado (`Cronjob Response: {nombre} (job_id: ...) -------------`) y un pie fijo (`To stop or manage this job, send me a new message...`) que no se veían limpios para el uso diario. Se confirmó (vía documentación e issues abiertos del proyecto) que esto es controlable con un solo interruptor global (todo o nada, sin edición de plantilla fina disponible aún):

```bash
docker exec hermes hermes config set cron.wrap_response false
docker restart hermes
```

**Resultado confirmado:** los mensajes de cron ahora llegan limpios, solo con el contenido redactado por Leonardo (ej. *"¡Estás al día por ahora! No hay pendientes en ZUMMAT."*).

## ✅ Checklist de la Fase 9

- [x] Confirmado: caída de NVIDIA NIM fue del lado del proveedor, no de la configuración de Diego.
- [x] Migrado a Gemini (`gemini-3.5-flash-lite`) como proveedor temporal.
- [x] Los 7 cron jobs re-fijados al nuevo proveedor/modelo.
- [x] `MEMORY.md` y `USER.md` limpiados de contenido de prueba.
- [x] Envoltorio de mensajes de cron desactivado (`cron.wrap_response: false`) — mensajes limpios confirmados.
- [ ] Pendiente: monitorear si NVIDIA NIM se restablece, para decidir si se regresa a Nemotron o se mantiene Gemini de forma permanente.

**Fase 9 completada el 16 de septiembre de 2026.**
