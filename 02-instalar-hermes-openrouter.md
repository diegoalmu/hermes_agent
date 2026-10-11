# Fase 2 — Obtener clave de OpenRouter y levantar Hermes en Docker

> ⚠️ **Actualización tras la ejecución real:** finalmente **no usamos OpenRouter** como proveedor de modelo. Se investigó en vivo durante el proyecto y se decidió usar **NVIDIA NIM** en su lugar, por ofrecer límites más generosos para uso personal (sin cuota diaria dura, ~40 solicitudes/minuto vs. las 20/min y 50/día de OpenRouter `:free`). Ver la sección "Decisión final de proveedor" más abajo. Los pasos originales de OpenRouter se dejan documentados abajo como referencia/alternativa, pero el proveedor realmente configurado es NVIDIA NIM.

## 1. Crear cuenta y obtener tu API key de OpenRouter

1. Entra a **https://openrouter.ai** y crea una cuenta (con Google/GitHub o correo). No pide tarjeta de crédito para usar modelos `:free`.
2. Ve a **https://openrouter.ai/keys** y crea una nueva clave (botón "Create Key").
3. Copia la clave, se ve algo así: `sk-or-v1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.

⚠️ **Nunca pegues esta clave en el chat conmigo ni la subas a GitHub.** Solo la vas a pegar dentro del asistente `hermes setup` que corre en tu propia terminal (paso 4).

## 2. Confirmar el modelo gratuito a usar

1. Ve a **https://openrouter.ai/models** y en el filtro de la izquierda activa **"Free"**.
2. Busca algo como `nvidia/nemotron-3-ultra:free` (contexto grande, buen razonamiento) o cualquier otro modelo marcado `:free` que tenga al menos 64,000 tokens de contexto (casi todos los `:free` actuales cumplen esto de sobra).
3. Anota el **ID exacto del modelo** (aparece debajo del nombre, formato `proveedor/nombre-modelo:free`) — lo necesitarás en el paso 4.

> La lista de modelos gratis cambia seguido. Si el que anotamos en la Fase 0 (`nvidia/nemotron-3-ultra:free`) ya no existe cuando tú revises, usa cualquier otro que veas marcado como Free — el proceso de configuración es idéntico.

## 3. Crear la carpeta de datos de Hermes en el host

```bash
mkdir -p ~/.hermes
```

**Qué hace:** crea la carpeta en tu PC donde Hermes va a guardar TODA su configuración, memoria, sesiones y claves (`~/.hermes`). Esta carpeta se "monta" dentro del contenedor Docker, así que aunque borres o actualices el contenedor, tus datos no se pierden.

## 4. Descargar la imagen oficial y correr el asistente de configuración

```bash
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  nousresearch/hermes-agent setup
```

**Qué hace cada parte:**
- `docker run -it --rm` → corre un contenedor **interactivo** (`-it`, para poder escribir respuestas) y lo **borra automáticamente** al salir (`--rm`) — el contenedor en sí es desechable, tus datos NO se borran porque viven en `~/.hermes` en tu PC.
- `-v ~/.hermes:/opt/data` → conecta tu carpeta `~/.hermes` (host) con `/opt/data` (dentro del contenedor), que es donde Hermes espera encontrar su configuración.
- `nousresearch/hermes-agent setup` → usa la imagen oficial de Hermes y ejecuta el comando `setup` (el asistente interactivo).

**Qué te va a preguntar el asistente (responde así):**
1. **Proveedor de modelo (LLM):** elige **OpenRouter**.
2. **API Key:** pega la clave que copiaste en el paso 1 (`sk-or-v1-...`).
3. **Modelo:** escribe el ID exacto que anotaste en el paso 2 (ej. `nvidia/nemotron-3-ultra:free`).
4. **Herramientas (tools):** puedes dejar las que vienen activadas por defecto; las ajustamos más adelante según el sub-agente.
5. **Canal de mensajería (Telegram, etc.):** puedes elegir **"configurar después"** — lo haremos en la Fase 3, dedicada solo a Telegram.

Al terminar, el asistente guarda todo en `~/.hermes/.env` y `~/.hermes/config.yaml`.

## Nota: incidencia de red durante la descarga de la imagen

Al primer intento de `docker run ... setup`, Docker falló con `no route to host` al resolver una de las IPs de Docker Hub. Diagnóstico: internet, DNS general y conexión TLS directa (`curl`) funcionaban bien — era una IP puntual de Docker Hub temporalmente inalcanzable desde esta red. Solución aplicada:

```bash
sudo mkdir -p /etc/docker
echo '{
  "dns": ["8.8.8.8", "1.1.1.1"]
}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
docker pull nousresearch/hermes-agent
```

Forzar a Docker a usar DNS de Google/Cloudflare resolvió el problema. **Imagen descargada exitosamente** (`sha256:7515a2c7...`).



```bash
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  nousresearch/hermes-agent
```

**Qué hace:** abre un chat interactivo directo con Hermes usando la configuración que acabas de guardar. Escríbele algo simple como `hola, preséntate` para confirmar que responde usando el modelo gratuito de OpenRouter. Para salir, `Ctrl+D` o escribe `/quit`.

## 6. Verificación de salud del sistema

Con el mismo comando de arriba abierto, o en un `docker run` nuevo, corre dentro del chat:

```
/help
```

o, desde tu terminal normal (fuera del chat):

```bash
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  nousresearch/hermes-agent doctor
```

**Qué hace `doctor`:** revisa que la configuración, las claves y las dependencias estén correctas, y te dice exactamente qué falta si algo está mal (equivalente al `hermes doctor` de tu curso original).

---

## ✅ Checklist de la Fase 2

- [ ] Tengo cuenta en OpenRouter y una API key generada.
- [ ] Anoté el ID exacto de un modelo `:free` con ≥64k contexto.
- [ ] Existe la carpeta `~/.hermes` en mi PC.
- [ ] Corrí `hermes-agent setup` dentro de Docker y configuré OpenRouter + el modelo.
- [ ] Tuve una conversación de prueba exitosa con Hermes.
- [ ] `hermes-agent doctor` no marca errores críticos.

## Decisión final de proveedor: NVIDIA NIM en vez de OpenRouter

Investigación comparativa realizada en vivo (30-31 ago 2026):

| | NVIDIA NIM (build.nvidia.com) | OpenRouter `:free` |
|---|---|---|
| Límite de velocidad | ~40 solicitudes/min (mejor esfuerzo, no garantizado) | 20 solicitudes/min |
| Límite diario | Sin cuota diaria documentada | 50/día (1,000/día solo con $10 USD de crédito) |
| Fuente | Directa (NVIDIA es dueño de Nemotron) | Reventa vía router |
| Registro | Solo correo, sin tarjeta | Sin tarjeta para modelos `:free` |

**Modelo elegido:** `nvidia/nemotron-3-ultra-550b-a55b` (550B total / 55B activos, contexto de 1,000,000 tokens). Es el modelo insignia de Nemotron 3 para razonamiento y trabajo de agente con herramientas.

**Cómo se obtuvo la clave:**
1. Cuenta creada en https://build.nvidia.com (solo correo, sin tarjeta).
2. Clave generada desde la página de cualquier modelo Nemotron (botón "Get API Key"), con formato `nvapi-...`.
3. Base URL usada internamente por Hermes: `https://integrate.api.nvidia.com/v1`.

**Modelo de respaldo anotado para el futuro:** si NVIDIA NIM llegara a fallar o cambiar sus condiciones, ya tenemos también configurada una clave de **Google Gemini** (usada por ahora solo para TTS, ver Fase 3), que también sirve como proveedor de modelo de texto alterno.

## Resultado real del asistente `hermes setup` (ejecutado en Docker)

Se usó el modo **"Full setup"** del asistente (no "Quick Setup / Nous Portal" ni "Blank Slate"). Configuración resultante:

| Paso del asistente | Elección |
|---|---|
| Terminal backend | **Local** (dentro del contenedor Docker que ya aísla todo — no se anidó un segundo Docker) |
| Plataformas de mensajería | **Telegram** y **WhatsApp** |
| Proveedor de modelo | **NVIDIA NIM** — `nvidia/nemotron-3-ultra-550b-a55b` |
| Herramientas (CLI y Telegram) | Selección por defecto completa: Web Search, Browser Automation, Terminal, File Ops, Code Execution, Vision, Image Generation, TTS, Skills, Task Planning, Memory, Session Search, Clarifying Questions, Task Delegation, Cron Jobs, Computer Use. Desactivadas: Video Analysis/Generation, X Search, Context Engine, Home Assistant, Spotify, Yuanbao, A2A |
| Proveedor de navegador | **Local Browser** (Chromium headless, sin clave) |
| Proveedor de generación de imágenes | **Skip** (se puede activar después) |
| Proveedor de Text-to-Speech | **Google Gemini TTS** (el usuario ya contaba con API key de Gemini) |
| Backend de visión | **Auto** (usa el modelo principal / fallback automático) |
| Proveedor de búsqueda web | **Tavily** (funciona sin clave — "keyless") |

## Incidente 1: error de red al descargar la imagen de Docker

Al primer intento de `docker run ... setup`, Docker falló con `no route to host` al resolver una de las IPs de Docker Hub. Diagnóstico: internet, DNS general y conexión TLS directa (`curl`) funcionaban bien — era una IP puntual de Docker Hub temporalmente inalcanzable desde esta red. Solución aplicada:

```bash
sudo mkdir -p /etc/docker
echo '{
  "dns": ["8.8.8.8", "1.1.1.1"]
}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
docker pull nousresearch/hermes-agent
```

Forzar a Docker a usar DNS de Google/Cloudflare resolvió el problema. **Imagen descargada exitosamente.**

## Configuración de Telegram con BotFather

Se creó el bot manualmente vía `@BotFather`:
- Nombre visible: **Leonardo**
- Username: **`@hermes_asiste_bot`**
- Token guardado en `~/.hermes/.env` como `TELEGRAM_BOT_TOKEN`.
- Acceso restringido solo al ID de Telegram del propietario: `TELEGRAM_ALLOWED_USERS=tu_id_de_telegram_aqui` (obtenido con `@userinfobot`).
- `TELEGRAM_HOME_CHANNEL=tu_id_de_telegram_aqui` (chat personal como canal por defecto de notificaciones).

`hermes setup` terminó exitosamente. `hermes doctor` inicial reportó **1 solo issue real** (`~/.local/bin/hermes` symlink faltante), corregido con `hermes doctor --fix`. El resto de advertencias (`⚠`) correspondían a proveedores/plataformas que no se activaron a propósito (OpenRouter, Discord, Spotify, X, Home Assistant, etc.) — no requieren acción.

## Prueba 1 exitosa: chat CLI directo

```bash
docker run -it --rm -v ~/.hermes:/opt/data nousresearch/hermes-agent
```

Respuesta confirmada del agente:
> "Hola. Soy Hermes Agent, desarrollado por Nous Research. Estoy corriendo con el modelo nvidia/nemotron-3-ultra-550b-a55b a través del proveedor NVIDIA"

Modelo y proveedor confirmados end-to-end. ✅

## Incidente 2: `docker exec` con "permission denied"

Al ejecutar `docker exec hermes hermes doctor --fix` en una terminal nueva, apareció `permission denied while trying to connect to the docker API`. Causa: el usuario `endoralex` sí estaba en el grupo `docker` a nivel de sistema (confirmado con `getent group docker`), pero la sesión de terminal abierta no había recargado los grupos. Solución: `newgrp docker` (o abrir una terminal nueva) aplicó el grupo correctamente; `groups` confirmó `docker` en la lista, y el comando funcionó sin `sudo`.

## Incidente 3 (en curso): el bot no responde en modo gateway (Telegram)

Al levantar el gateway 24/7 con:
```bash
docker run -d --name hermes --restart unless-stopped \
  -v ~/.hermes:/opt/data -p 8642:8642 \
  nousresearch/hermes-agent gateway run
```
el log se quedaba congelado en `[Telegram] Connecting to Telegram (attempt 1/8)…` y el bot no respondía en Telegram (aunque el modo CLI directo sí funcionaba).

**Diagnóstico realizado:**
1. `curl` a `https://api.telegram.org` desde dentro y fuera del contenedor: **funciona perfecto** (HTTP 302 normal). Pero se observó que la resolución **IPv6** de Telegram falla ("Network is unreachable"); `curl` cae a IPv4 casi instantáneo, sin problema.
2. Se verificó Tailscale y las rutas de red (`ip route show`) — sin conflictos.
3. Se confirmó que el **token de Telegram es válido** llamando directamente a la API:
   ```bash
   docker exec hermes sh -c 'curl -s https://api.telegram.org/bot$(grep "^TELEGRAM_BOT_TOKEN=" /opt/data/.env | cut -d= -f2)/getMe'
   ```
   Respuesta: `{"ok":true,"result":{...,"username":"hermes_asiste_bot",...}}` ✅ — el problema **no es de credenciales**.
4. El log mostró que, a diferencia de `curl`, la librería interna (`python-telegram-bot`) tarda **~20 segundos en fallar** cada intento de conexión (timeout largo) antes de caer a IPv4 — probablemente porque intenta primero la ruta IPv6 rota y espera el timeout completo en vez de fallar rápido como `curl`.

**Solución aplicada:** recrear el contenedor forzando DNS explícito y desactivando IPv6 a nivel de kernel dentro del contenedor:
```bash
docker rm -f hermes
docker run -d \
  --name hermes \
  --restart unless-stopped \
  --dns=8.8.8.8 \
  --dns=1.1.1.1 \
  --sysctl net.ipv6.conf.all.disable_ipv6=1 \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  nousresearch/hermes-agent gateway run
```

**Estado al momento de esta actualización de bitácora:** contenedor recreado con IPv6 desactivado, esperando confirmación de que el gateway conecta correctamente a Telegram.

## Incidente 3 — Resolución final: causa raíz doble

El incidente de Telegram no fue causado por un solo factor, sino por **dos problemas superpuestos** que se fueron descubriendo en capas:

**Causa A — IPv6 roto en el contenedor.** Confirmada y resuelta con `--dns=8.8.8.8 --dns=1.1.1.1 --sysctl net.ipv6.conf.all.disable_ipv6=1` en el `docker run`.

**Causa B — Contenedores duplicados compitiendo por el mismo token.** Después de varias recreaciones del contenedor (`docker rm -f hermes` + `docker run` de nuevo) sin limpiar antes los contenedores viejos, terminaron **dos procesos de Hermes corriendo al mismo tiempo**, ambos intentando hacer `getUpdates` con el mismo `TELEGRAM_BOT_TOKEN`. Telegram solo permite un proceso por token, así que el segundo (o ambos) entraban en un ciclo de:
```
[Telegram] Telegram polling conflict (1/5) — previous session still held open on Telegram's servers...
```
Esto a veces coincidía además con un corte de DNS puntual dentro del contenedor (`Temporary failure in name resolution`), lo que hacía más confuso el diagnóstico.

**Lección aprendida (clave para la bitácora/portafolio):** al recrear un contenedor con `docker run --name hermes`, si el nombre ya existe Docker da error y no lo deja crear — pero si en algún punto se usó un nombre distinto o el contenedor viejo no se eliminó bien, pueden quedar **instancias huérfanas** corriendo en segundo plano sin darse cuenta. Antes de cualquier troubleshooting de conexión a una plataforma de mensajería, **siempre verificar primero cuántos contenedores de la imagen existen**:
```bash
docker ps -a --filter ancestor=nousresearch/hermes-agent
```
y limpiar todos antes de levantar uno nuevo:
```bash
docker rm -f $(docker ps -a --filter ancestor=nousresearch/hermes-agent -q)
```

**Regla operativa adoptada:** nunca abrir un `docker run -it` o `docker exec -it hermes hermes` interactivo mientras el contenedor `gateway run` esté activo — eso crea un segundo proceso compitiendo por el mismo token. Para chatear con el agente, usar siempre Telegram/WhatsApp; para depurar, usar solo `docker logs -f hermes` (solo lectura, no abre sesión nueva).

**Confirmación final (31 de agosto de 2026, sesión nueva del día siguiente):**
```bash
newgrp docker   # la terminal nueva no heredaba el grupo docker; se reactivó
docker rm -f $(docker ps -a --filter ancestor=nousresearch/hermes-agent -q)   # se encontraron y eliminaron 2 contenedores viejos
docker run -d --name hermes --restart unless-stopped \
  --dns=8.8.8.8 --dns=1.1.1.1 \
  --sysctl net.ipv6.conf.all.disable_ipv6=1 \
  -v ~/.hermes:/opt/data -p 8642:8642 \
  nousresearch/hermes-agent gateway run
```
Resultado: log limpio (sin `Conflict`, sin errores de DNS), un solo contenedor activo (`docker ps -a` confirma únicamente `hermes`, `Up`), y respuesta exitosa del bot en Telegram:
> Diego: "¿ya estás funcionando bien ahora?"
> Leonardo: "Sí, funcionando con normalidad. ¿Hay algo en lo que te pueda ayudar?"

✅ **Incidente cerrado.**

### Checklist actualizado de la Fase 2

- [x] Cuenta y API key de NVIDIA NIM (build.nvidia.com) obtenida.
- [x] Modelo elegido: `nvidia/nemotron-3-ultra-550b-a55b`.
- [x] Imagen `nousresearch/hermes-agent` descargada (tras resolver incidente de DNS de Docker Hub).
- [x] `hermes setup` completado en modo Full Setup.
- [x] Bot de Telegram creado (`@hermes_asiste_bot`), acceso restringido al ID del propietario.
- [x] WhatsApp seleccionado como plataforma adicional.
- [x] Proveedor de búsqueda (Tavily), navegador (Local Browser), TTS (Gemini) y visión (Auto) configurados.
- [x] `hermes doctor` sin issues pendientes (symlink corregido).
- [x] Prueba de chat CLI directo exitosa, modelo confirmado.
- [x] Gateway 24/7 confirmado funcionando correctamente en Telegram (incidente de conexión + contenedores duplicados resuelto).
- [ ] Pendiente: confirmar respuesta en WhatsApp.

Siguiente: **Fase 3 — Crear los sub-agentes (perfiles) `coder` y `oficina`.**
