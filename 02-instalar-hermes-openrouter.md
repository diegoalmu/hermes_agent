# Fase 2 — Obtener clave de OpenRouter y levantar Hermes en Docker

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

## 5. Primera conversación de prueba (modo CLI, sin gateway)

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

Cuando termines, dime "listo fase 2" (o pégame cualquier error) y seguimos con la **Fase 3: configurar Telegram como canal de comunicación con BotFather**.
