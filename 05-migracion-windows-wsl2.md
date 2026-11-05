# Fase 5 — Migración de Linux Mint a Windows (WSL2) y Reconstrucción de Hermes

## Contexto y motivo del cambio

Se decidió migrar `endor-pc` de Linux Mint de vuelta a Windows 11 (25H2), principalmente por friccion acumulada de mantenimiento en Mint (ej. compilar `v4l2loopback` desde código fuente para la webcam por incompatibilidad de kernel) y por preferencia de estabilidad para el uso diario. Se evaluaron alternativas (dual-boot, VirtualBox) antes de decidir por **Ubuntu real dentro de WSL2**, que da rendimiento casi nativo y un entorno Linux completo para seguir programando, sin las limitaciones de VirtualBox (carpetas compartidas lentas) ni de Docker Desktop solo (sin shell Linux completa).

## Inventario de lo que había que migrar

Antes de migrar se hizo un inventario real de los sistemas que corrían en el `endor-pc` viejo:

| Sistema | Ubicación real | Decisión |
|---|---|---|
| **Valeria** (n8n) | Oracle Cloud (`aztlan`), Docker, **no** en endor-pc | Sin acción — no depende de esta máquina |
| **Demian** (OpenClaw) | endor-pc, proceso npm nativo (no Docker) | Desinstalado — Hermes cubre la misma función |
| **Leonardo** (Hermes) | endor-pc, Docker (`nousresearch/hermes-agent`) | Único sistema migrado |

**Lección:** antes de asumir que algo "vive en esta PC", vale la pena confirmarlo con `docker ps -a` en cada máquina. Valeria llevaba semanas corriendo en Oracle Cloud sin que se recordara con certeza.

## Intento de respaldo fallido — advertencia importante

Se intentó respaldar el volumen de datos de Hermes con:

```bash
docker exec hermes tar czf /tmp/hermes-backup.tar.gz /ruta/a/su/config
```

Sin sustituir `/ruta/a/su/config` por la ruta real (nunca se confirmó con `docker inspect hermes | grep -A 5 Mounts` antes de correr el comando). El resultado fue un archivo de **45 bytes**, prácticamente vacío. Para cuando se detectó el problema, Linux Mint ya había sido formateado — **la memoria y configuración anteriores de Hermes se perdieron por completo.**

**Lección para la próxima vez:** siempre confirmar la ruta real del volumen con `docker inspect <container> --format='{{json .Mounts}}'` ANTES de armar cualquier comando de backup, y verificar el tamaño del archivo resultante (`ls -lh`) antes de dar por bueno un respaldo.

**Mitigación:** por suerte, todo el despliegue de Hermes ya estaba documentado paso a paso en fases anteriores de esta misma bitácora (`00` a `04`), así que se pudo reconstruir la configuración completa sin depender de memoria acumulada — Hermes solo tenía los 3 cron jobs de clases en ese momento, sin memoria conversacional significativa que lamentar.

## Configuración de red: WSL2 en modo "mirrored"

Para que Tailscale (y cualquier servicio publicado desde dentro de WSL2) fuera alcanzable desde otros dispositivos de la red, se activó el modo de red espejo de WSL2, editando `%USERPROFILE%\.wslconfig`:

```
[wsl2]
networkingMode=mirrored
```

Con esto, la Ubuntu de WSL2 comparte la misma IP que Windows (incluida la IP de Tailscale), eliminando el problema típico de NAT donde servicios dentro de WSL2 no son alcanzables desde fuera de la PC.

**Nota:** se detectó que Tailscale ya estaba instalado nativamente en Windows (nodo `endorpc`) además de instalarse por error también dentro de la Ubuntu de WSL2 (nodo `endorpc-wsl`), generando un duplicado. Con el modo mirrored activo, el de Windows es suficiente — el de WSL2 debe desinstalarse:

```bash
sudo tailscale down
sudo apt remove tailscale -y
```

## Mudanza de la distro WSL2 al segundo disco (D:)

Para no llenar el disco de sistema (C:), se exportó e importó la distro completa a `D:\WSL`:

```powershell
wsl --shutdown
wsl --export Ubuntu D:\WSL\Ubuntu-backup.tar
wsl --unregister Ubuntu
wsl --import Ubuntu D:\WSL\Ubuntu D:\WSL\Ubuntu-backup.tar --version 2
```

**Cuidado:** `wsl --import` reinicia el usuario por defecto a `root`. Se corrigió agregando a `/etc/wsl.conf`:

```
[user]
default=endoralex
```

**Bug encontrado:** el `.tar` exportado ya traía esta configuración desde antes (de cuando la distro vivía en C:), así que al agregarla de nuevo quedó **duplicada** en el archivo, generando una advertencia (no un error real) cada vez que arrancaba WSL. Se corrigió editando `/etc/wsl.conf` manualmente y dejando un solo bloque `[user]`.

## Reconstrucción de Hermes desde cero

Se recreó el contenedor con la configuración completa desde el inicio (a diferencia del despliegue original, que se fue armando por partes):

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  --dns=8.8.8.8 \
  --dns=1.1.1.1 \
  --sysctl net.ipv6.conf.all.disable_ipv6=1 \
  -e HERMES_DASHBOARD=1 \
  -e HERMES_TIMEZONE=America/Mexico_City \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  -p 9119:9119 \
  nousresearch/hermes-agent gateway run
```

Se corrió `hermes setup` (asistente interactivo) para reconfigurar: modelo (NVIDIA NIM, `nvidia/nemotron-3-ultra-550b-a55b`), backend de terminal (Local — corre dentro del propio contenedor, no confundir con el modo "Docker" del wizard que requiere montar el socket de Docker del host), buscador web (Tavily, modo sin clave), y herramientas de Telegram — **desmarcando explícitamente Computer Use e Image Generation** por ser las de mayor riesgo/menor necesidad para este proyecto.

**Bug del wizard:** al intentar desmarcar herramientas en la pantalla de selección múltiple, el cambio no se guardó correctamente (posible Enter prematuro cerrando la lista antes de confirmar todos los checks). Se verificó y corrigió editando `/opt/data/config.yaml` directamente, quitando `computer_use` e `image_gen` de ambos bloques `platform_toolsets` (`cli` y `telegram`).

## 🔴 El bug más importante: permisos por UID, no por "root vs usuario"

Este fue el problema que más tiempo tomó resolver, y vale la pena documentarlo con detalle porque no es intuitivo.

### Síntoma inicial
Al intentar editar `~/.hermes/config.yaml` desde el host, la carpeta daba "Permission denied" incluso para `ls`.

### Diagnóstico incorrecto (primer intento)
Se asumió que el proceso dentro del contenedor corría como **root**, y que por eso los archivos quedaban con dueño root en el host. Se corrigió con:

```bash
sudo chown -R endoralex:endoralex /home/endoralex/.hermes
```

Esto **rompió Hermes por completo** — empezó a fallar `hermes skills list`, `hermes doctor`, y cualquier comando que necesitara leer `/opt/data/.env`, con `PermissionError: [Errno 13] Permission denied`.

### Causa real
El proceso de Hermes dentro del contenedor **no corre como root** — corre como un usuario dedicado:

```
uid=10000(hermes) gid=10000(hermes) groups=10000(hermes)
```

Al hacer `chown` a `endoralex` (o después, por error, de vuelta a `root:root`), ninguno de los dos coincidía con el UID 10000 real del proceso, así que el proceso perdía acceso de lectura/escritura a su propia carpeta de datos — incluyendo `.env`, `config.yaml`, y los archivos de lock (`kanban.db.init.lock`, `state.db.auto-maintenance.lock`, `gateway-locks/telegram-bot-token-*.lock`), lo cual en cascada rompía: el dashboard (rehusaba bind por no poder leer la config de auth), el dispatcher de kanban (fallaba cada minuto en logs), y **la conexión de Telegram** (no podía crear el archivo de lock del token, así que el bot nunca llegaba a conectar aunque el token fuera válido).

### Solución definitiva

```bash
docker exec hermes id hermes
# uid=10000(hermes) gid=10000(hermes)

sudo chown -R 10000:10000 /home/endoralex/.hermes
docker restart hermes
```

Confirmado en logs tras el fix: `[Telegram] Connected to Telegram (polling mode)`, sin ningún `PermissionError` adicional.

### 📌 Regla general para el futuro
**Nunca asumir que un contenedor corre como root.** Antes de tocar permisos de un volumen montado desde el host, confirmar el UID/GID real del usuario del proceso dentro del contenedor con `docker exec <container> id <usuario>` (o `id` a secas si se desconoce el nombre del usuario), y aplicar ESE UID/GID exacto al `chown` del host — nunca el propio usuario del host ni `root:root` a ciegas.

## Otros hallazgos menores

- **Hairpin NAT / loopback:** la propia PC no puede conectarse a sí misma usando su IP pública/de Tailscale (`Test-NetConnection` fallaba desde `endor-pc` hacia su propia IP de Tailscale). Esto es normal y no indica un problema real — la prueba válida es siempre desde OTRO dispositivo de la red. Desde `endor-pc`, usar `localhost` en vez de la IP de Tailscale.
- **Hermes cambió su mecanismo de autenticación del dashboard** desde que se armó el proyecto original: ya no usa variables de entorno (`HERMES_DASHBOARD_BASIC_AUTH_*`), ahora requiere un hash de contraseña (`scrypt`) directo en `config.yaml`, generado con:
  ```bash
  docker exec hermes python -c "from plugins.dashboard_auth.basic import hash_password; print(hash_password('TuContraseña'))"
  ```
- **Cuidado con pegar secretos en el chat con el asistente:** durante esta sesión se expuso el token de Telegram más de una vez al copiar el `.env` completo para depurar. Buena práctica: pedir solo el resultado de `grep NOMBRE_VAR archivo` en vez de volcar el archivo entero, y regenerar cualquier secreto que se haya compartido así.

## ✅ Estado final

- [x] Ubuntu real corriendo en WSL2 (modo mirrored), en disco `D:`
- [x] Docker instalado dentro de esa Ubuntu (no Docker Desktop de Windows)
- [x] Hermes reconstruido: NVIDIA NIM + Tavily, dashboard con auth por hash, `computer_use`/`image_gen` desactivados
- [x] Skills `coder` y `oficina` recreadas, con delegación configurada en `SOUL.md`
- [x] 5 cron jobs activos: 3 recordatorios de clase (Lunes/Martes/Miércoles) + 2 de tutoría Infotec (Lunes-Miércoles/Sábado)
- [x] Acceso remoto al dashboard confirmado vía Tailscale desde un dispositivo externo (celular, datos móviles)
- [x] Telegram conectado y respondiendo
- [ ] Pendiente: eliminar nodo duplicado `endorpc-wsl` de Tailscale
- [ ] Pendiente: regenerar token de Telegram (expuesto en el proceso de depuración)
- [ ] Pendiente: guardar el contenido real de los `SKILL.md` de `coder`/`oficina` en este repo (hoy solo viven dentro del volumen de Docker)

**Fase 5 completada el 7 de septiembre de 2026.**
