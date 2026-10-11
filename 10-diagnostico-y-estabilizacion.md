# Fase 10 — Diagnóstico y estabilización: Nemotron de vuelta, puertos cerrados y retiro de WhatsApp

## Contexto

Tras la Fase 9 (caída de NVIDIA NIM y migración temporal a Gemini), Leonardo parecía "fallar a cada rato" con Nemotron. La sospecha inicial era que el nivel gratuito de NVIDIA seguía siendo inestable. La sesión empezó evaluando si convenía cambiar de modelo (Ollama local, modelos gratuitos de OpenCode Zen) y terminó en una revisión completa de la configuración, que encontró 6 problemas independientes.

## Evaluación previa: ¿Ollama local en lugar de NVIDIA?

**Descartado.** `endor-pc` (Lenovo M720q, Intel Core i5-8400T, 32 GB RAM) no tiene GPU dedicada, solo gráfica integrada Intel UHD 630. Ollama correría únicamente en CPU:

- Los modelos que caben en memoria (8B–30B) son lentos y bastante menos capaces que los de nube.
- Un agente envía entre 10,000 y 30,000 tokens de contexto por turno; sin GPU, solo leerlos puede tardar varios minutos.
- La RAM se comparte con WSL2, Docker, el hub de ZUMMAT y el propio Hermes.

**Lección:** sin GPU dedicada, un modelo local sirve para pruebas sin internet, no para un agente de uso diario.

## Incidente 1 — Nemotron nunca respondía: la `base_url` apuntaba a Google

**Síntoma:** errores en cada ejecución de la tarea programada de las 9:00 am:

```
openai.BadRequestError: Error code: 400 - Please pass a valid API key (INVALID_ARGUMENT)
Model fallback: nvidia/nemotron-3-ultra-550b-a55b via nvidia unavailable (provider failure);
using gemini-3.5-flash-lite via gemini.
```

**Diagnóstico incorrecto (primer intento):** se asumió que el endpoint gratuito de NVIDIA era inestable (límite de ~40 solicitudes por minuto, caídas como la de la Fase 9).

**Causa real:** la línea del log que lo delataba:

```
provider=nvidia base_url=https://generativelanguage.googleapis.com/v1beta model=nvidia/nemotron-3-ultra-550b-a55b
```

El bloque del modelo principal en `config.yaml` tenía la URL de **Google Gemini**. Hermes enviaba las peticiones de Nemotron, con la clave de NVIDIA, al servidor de Google, que las rechazaba. **Las peticiones nunca llegaron a NVIDIA.** El formato del error (`INVALID_ARGUMENT`) era de Google, no de NVIDIA.

**Origen probable:** en la Fase 9 se cambió el proveedor principal a Gemini. Al regresar a Nemotron se actualizaron el proveedor y el modelo, pero la `base_url` de Google se quedó.

**Impacto:** el panel web mostró que, de 62 sesiones en 30 días, 43 corrieron en Gemini 3.5 Flash Lite (el modelo de respaldo), no en Nemotron.

**Ubicación real de la configuración:** la carpeta `~/hermes-data` que existía en el home estaba vacía. La configuración real se confirmó con:

```bash
docker inspect hermes --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

Resultado: `/home/endoralex/.hermes -> /opt/data`. Como la carpeta pertenece al UID 10000 del contenedor (Fase 5), todos los comandos de lectura requieren `sudo`.

**Solución:** primero un respaldo verificado (ver Incidente 6), luego la corrección solo de la línea 4:

```bash
sudo sed -i '4s|https://generativelanguage.googleapis.com/v1beta|https://integrate.api.nvidia.com/v1|' /home/endoralex/.hermes/config.yaml
```

Configuración resultante:

```yaml
model:
  default: nvidia/nemotron-3-ultra-550b-a55b
  provider: nvidia
  base_url: https://integrate.api.nvidia.com/v1
fallback_providers:
  - provider: gemini
    model: gemini-3.5-flash-lite
    base_url: https://generativelanguage.googleapis.com/v1beta
```

```bash
docker restart hermes
```

**Confirmación:** tras un mensaje de prueba por Telegram, los logs ya no mostraron errores de NVIDIA ni la línea `Model fallback`.

### 📌 Regla general

Cuando un proveedor "falla siempre", revisar primero la `base_url` en el log antes de culpar al proveedor. Y cada vez que se cambie de proveedor (como en la Fase 9), revisar las **tres** claves del bloque `model`: `default`, `provider` **y** `base_url`.

## Incidente 2 — Errores de OpenRouter y Nous sin tener cuentas ahí

**Síntoma:** en cada mensaje aparecían:

```
Auxiliary: marking openrouter unhealthy for 60s (payment / credit error)
Auxiliary: marking nous unhealthy for 60s (payment / credit error)
```

**Diagnóstico incorrecto (primer intento):** se pensó que había una clave de OpenRouter sin crédito en el `.env`. No existía ninguna.

**Causa real:** las 11 tareas auxiliares de Hermes (resúmenes, títulos, visión, compresión, aprobaciones, etc.) estaban en modo `auto`. En ese modo, Hermes recorre una lista fija de proveedores (OpenRouter → Nous → local → api-key) aunque no estén configurados. "Payment / credit error" significaba "no tengo con qué autenticarme ahí". Los logs de septiembre lo confirmaron:

```
Auxiliary approval: connection error on auto and no fallback available (tried: openrouter, nous, local/custom, api-key)
```

**Solución:** desde el panel web (Models → Auxiliary tasks → Configure) se asignó `gemini-3.5-flash-lite` a las 11 tareas. Se eligió Gemini y no Nemotron porque:

- Ya estaba probado como modelo de respaldo.
- Tiene visión (puede analizar imágenes).
- Las tareas auxiliares se disparan en cada mensaje; si fueran a Nemotron, consumirían el doble del límite gratuito de NVIDIA.

Verificación:

```bash
sudo sed -n '/^auxiliary:/,/^[a-z_]*:/p' /home/endoralex/.hermes/config.yaml
```

### 📌 Regla general

"Auto" no significa "usa lo que tengo configurado", sino "prueba una lista fija". Asignar modelos explícitos a las tareas auxiliares.

## Incidente 3 — Mixture of Agents activado con modelos de pago sin acceso

**Síntoma:** el panel web mostraba "Mixture of Agents: 2 references · openrouter/claude-opus-4.8".

**Causa real:** la función Mixture of Agents (MoA) estaba activada (`enabled: true`) con `fanout: user_turn`; es decir, en **cada mensaje** intentaba consultar a GPT-5.5 (`openai-codex`) y a Claude Opus 4.8 (`openrouter`) como "segunda opinión". Ninguno de los dos proveedores estaba configurado, y ambos son de pago.

**Problema adicional:** usar el botón **"Remove"** del panel no quitó la referencia; reescribió el bloque `moa` completo en `config.yaml` con valores por defecto.

**Diagnóstico incorrecto (al editar):** se calculó el número de línea a partir de una salida filtrada con `grep -v "key|token|secret"`. El filtro escondió una línea del bloque y el número quedó desfasado por uno. El `sed` no se ejecutó porque primero se verificó la línea con `sed -n '156p'`, que mostró otra cosa.

**Solución:** ubicar la línea exacta sobre el archivo real (solo la de 2 espacios de sangría dentro del bloque):

```bash
sudo awk 'NR>=138 && NR<=165 && /^  enabled:/ {print NR": "$0}' /home/endoralex/.hermes/config.yaml
```

```bash
sudo sed -i '157s/^  enabled: true$/  enabled: false/' /home/endoralex/.hermes/config.yaml
```

### 📌 Regla general

- Antes de un `sed` por número de línea, **imprimir esa línea primero** para confirmar.
- Nunca calcular números de línea a partir de una salida filtrada.
- Para cambios de configuración, editar el archivo directamente fue más predecible que el panel web.

## Incidente 4 — Panel y API expuestos a toda la red local

**Síntoma:** `docker ps -a` mostraba:

```
0.0.0.0:8642->8642/tcp, 0.0.0.0:9119->9119/tcp
```

Todos los demás contenedores del equipo (hub de ZUMMAT, n8n, Postgres, Nginx) escuchan en `127.0.0.1`. Hermes era el único publicado hacia la red.

**Causa real:** el `docker run` de la Fase 5 usaba `-p 8642:8642 -p 9119:9119`, que publica en todas las interfaces. Esto fue intencional en la Fase 3 (acceso por WiFi y por Tailscale). El panel ya estaba protegido con contraseña (hash scrypt, Fase 5), pero el API server (8642) también quedaba alcanzable desde la red.

**Solución:** recrear el contenedor publicando solo en localhost. Los datos viven en `/home/endoralex/.hermes`, fuera del contenedor, así que no se pierden.

Verificar primero que no haya contenedores duplicados (regla de la Fase 2):

```bash
docker ps -a --filter ancestor=nousresearch/hermes-agent
```

```bash
docker stop hermes
```

```bash
docker rm hermes
```

**Nuevo comando canónico:**

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  --dns=8.8.8.8 \
  --dns=1.1.1.1 \
  --sysctl net.ipv6.conf.all.disable_ipv6=1 \
  -e HERMES_DASHBOARD=1 \
  -e HERMES_TIMEZONE=America/Mexico_City \
  -v /home/endoralex/.hermes:/opt/data \
  -p 127.0.0.1:8642:8642 \
  -p 127.0.0.1:9119:9119 \
  nousresearch/hermes-agent gateway run
```

Verificación:

```bash
docker ps --filter name=hermes --format "{{.Names}}  {{.Status}}  {{.Ports}}"
```

Resultado: `127.0.0.1:8642->8642/tcp, 127.0.0.1:9119->9119/tcp`. Telegram, CLI y panel web (`http://localhost:9119`) confirmados funcionando.

**Importante:** `API_SERVER_HOST=0.0.0.0` se mantiene en el `.env`. Dentro del contenedor, Hermes debe escuchar en todas sus interfaces para que Docker pueda conectarle el puerto; la restricción se hace en el `-p` del `docker run`.

**Consecuencia no prevista:** este cambio eliminó el acceso al panel por Tailscale desde otros dispositivos, que en la Fase 3 se había decidido conservar. Queda como decisión pendiente.

### 📌 Regla general

Publicar puertos como `-p 127.0.0.1:PUERTO:PUERTO` salvo que algo externo los necesite. Antes de cambiar la forma de acceso, revisar en la bitácora si hubo una decisión previa al respecto.

## Incidente 5 — Retiro de WhatsApp: vulnerabilidad crítica e inestabilidad

**Síntoma:**

- `hermes doctor` reportó: `WhatsApp bridge deps (1 critical, 1 high, 4 moderate)`.
- El puente de WhatsApp se caía en cada reinicio (`WhatsApp bridge process exited unexpectedly (code -15)`) y daba errores de conexión al puerto 3000.

**Contexto:** en la Fase 8, WhatsApp se conectó en modo self-chat con el número personal. Eso convierte a Hermes en un **dispositivo vinculado** de la cuenta personal: el puente recibe todos los mensajes de esa cuenta (familia, amigos), y las llaves de la sesión quedan en `.hermes/whatsapp/session`, al alcance de la herramienta de terminal del agente.

**Decisión:** retirar WhatsApp. En la práctica no se estaba usando. Canales finales: **Telegram, CLI y panel web.**

```bash
sudo sed -i 's/^WHATSAPP_ENABLED=true$/WHATSAPP_ENABLED=false/' /home/endoralex/.hermes/.env
```

Además, en el teléfono: **WhatsApp → Ajustes → Dispositivos vinculados** → cerrar la sesión de Hermes. Desactivarlo en Hermes no cierra la sesión vinculada.

**Sobre `hermes doctor --fix`:** no se usó para las vulnerabilidades de npm. Hermes vive en un contenedor; lo que se arregle dentro de `/opt/hermes` se pierde al actualizar la imagen. Las demás advertencias (`web workspace`, `ui-tui`) son herramientas de construcción que no se usan al correr Hermes.

### 📌 Regla general

Un agente con acceso a terminal no debe tener vinculada una cuenta personal de mensajería. Sin un número dedicado, usar un canal de bot (Telegram) restringido al ID propio.

## Incidente 6 — El respaldo existente estaba vacío (otra vez)

**Síntoma:**

```bash
tar -tzf /home/endoralex/hermes-backup.tar.gz | head -20
```

No listó ningún archivo.

**Causa real:** el respaldo se hizo de la carpeta equivocada (`~/hermes-data`, vacía), no de `~/.hermes`. Es la misma lección de la Fase 5, repetida.

**Solución:**

```bash
sudo tar -czf /home/endoralex/respaldo-hermes-2026-10-10.tar.gz -C /home/endoralex .hermes
```

```bash
sudo tar -tzf /home/endoralex/respaldo-hermes-2026-10-10.tar.gz | head -20
```

Los avisos `socket ignored` y `file changed as we read it` son normales con el contenedor encendido.

### 📌 Regla general

Un respaldo no existe hasta que se verifica su contenido con `tar -tzf`. Respaldar siempre la ruta que muestra `docker inspect`.

## Hallazgos menores

- **Archivos basura por comandos mal formados:** `~/API_SERVER_ENABLED=true\nAPI_SERVER_HOST=0.0.0.0\n` (en el home) y `~/.hermes/.envprintf` (dueño `root`) eran resultado de `echo`/`printf` que crearon archivos nuevos en vez de agregar líneas al `.env`. Las variables ya estaban en el `.env` real.
- **Skill `pendientes-trabajo` con dueño `root`:** Hermes (UID 10000) puede leerla, pero no modificarla.
- **Herramienta de memoria:** tras el arreglo, Nemotron llamó dos veces la herramienta `memory` sin indicar la acción (`Unknown action 'None'. Use: add, replace, remove`). Es un error del modelo, no de la herramienta (`hermes doctor` la marca disponible). Queda en observación.
- **Hora de los logs:** los logs de Hermes están en UTC. Los errores "de las 15:00" correspondían a la tarea programada de las 9:00 am hora de Ciudad de México.
- **`hermes doctor`:** sin avisos de seguridad ni servidores MCP sospechosos; NVIDIA NIM y Gemini conectan correctamente. `MEMORY.md` con 95 caracteres, consistente con la limpieza de la Fase 9.

## Acceso a la CLI desde WSL2

Hermes vive dentro del contenedor, así que el comando `hermes` no existe directamente en Ubuntu. Se agregó un alias:

```bash
echo "alias hermes='docker exec -it hermes hermes'" >> /home/endoralex/.bashrc
```

```bash
source /home/endoralex/.bashrc
```

Uso: `hermes` (chat), `hermes doctor`, `hermes cron list`, `hermes gateway status`.

**Nota:** la regla operativa de la Fase 2 prohibía abrir `docker exec -it hermes hermes` con el gateway activo. En esta sesión se usó y Telegram siguió respondiendo con normalidad; en la Fase 2 la causa real del conflicto fueron contenedores duplicados. Queda en observación (ver pendientes).

## ✅ Checklist de la Fase 10

- [x] Ollama local evaluado y descartado por falta de GPU.
- [x] `base_url` del modelo principal corregida a NVIDIA; Nemotron respondiendo.
- [x] 11 tareas auxiliares asignadas a `gemini-3.5-flash-lite`.
- [x] Mixture of Agents desactivado.
- [x] Contenedor recreado con puertos solo en localhost (`127.0.0.1`).
- [x] Telegram, CLI y panel web confirmados funcionando.
- [x] Telegram restringido al ID del propietario (verificado).
- [x] WhatsApp desactivado.
- [x] Respaldo real y verificado de `~/.hermes`.
- [ ] Pendiente urgente: revisar con `hermes cron list` si las tareas programadas siguen fijadas a Gemini (Fase 9) o quedaron sin fijar tras el cambio de `base_url` (riesgo de `drift_skip`).
- [ ] Pendiente: cerrar la sesión de WhatsApp en "Dispositivos vinculados" del teléfono.
- [ ] Pendiente: prueba de memoria (agregar pendientes por texto y por nota de voz, verificar en `~/.hermes/memories/` y en el aviso de las 9:00 am).
- [ ] Pendiente: decidir el acceso remoto al panel (`tailscale serve`, o `-p 100.x.x.x:9119:9119` además de `127.0.0.1`, como se anotó en la Fase 3).
- [ ] Pendiente: verificar que `computer_use` e `image_gen` sigan fuera de `platform_toolsets` (decisión de la Fase 5).
- [ ] Pendiente: revisar en los logs que la CLI interactiva no provoque `Telegram polling conflict`.
- [ ] Pendiente: medir durante 2–3 días los errores de herramientas con Nemotron:
  ```bash
  docker logs --since 24h hermes 2>&1 | grep "returned error" | grep -oE "Tool [a-z_]+" | sort | uniq -c | sort -rn
  ```
- [ ] Pendiente: borrar `~/.hermes/.envprintf`, el archivo basura del home y `~/hermes-backup.tar.gz` (vacío).
- [ ] Pendiente: devolver la skill `pendientes-trabajo` al UID 10000.

**Fase 10 completada el 10 de octubre de 2026.**
