# Fase 1 — Prerrequisitos e instalación de Docker (Linux local)

> Ejecuta estos comandos en la terminal de tu PC Linux. Si tu distro **no** es Ubuntu/Debian (por ejemplo Fedora, Arch), avísame para darte los comandos equivalentes — el resto del proyecto (Hermes en Docker) es idéntico sin importar la distro.

## 1. Verificar qué tienes instalado

```bash
git --version
curl --version
docker --version
```

**Qué hace cada comando:**
- `git --version` → confirma si Git está instalado (lo necesitas para el repo de bitácora y porque el instalador de Hermes lo requiere).
- `curl --version` → confirma si `curl` está instalado (se usa para descargar cosas por línea de comandos).
- `docker --version` → confirma si Docker ya está instalado. Si te dice `command not found`, seguimos con el paso 2.

## 2. Instalar Docker Engine (si no lo tienes) — Ubuntu/Debian

```bash
# 2.1 Actualizar el índice de paquetes
sudo apt-get update

# 2.2 Instalar paquetes necesarios para agregar el repositorio de Docker
sudo apt-get install -y ca-certificates curl gnupg

# 2.3 Crear la carpeta para las llaves GPG de Docker
sudo install -m 0755 -d /etc/apt/keyrings

# 2.4 Descargar la llave GPG oficial de Docker
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 2.5 Agregar el repositorio oficial de Docker a las fuentes de apt
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 2.6 Actualizar de nuevo el índice (ahora incluye el repo de Docker)
sudo apt-get update

# 2.7 Instalar Docker Engine, CLI, containerd y el plugin de Compose
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Explicación breve:** estos pasos son la instalación oficial de Docker desde su propio repositorio APT (en vez de la versión vieja que trae Ubuntu por defecto), e incluyen `docker-compose-plugin`, que es lo que nos permitirá usar `docker compose` más adelante para el despliegue final de Hermes.

## 3. Permitir usar Docker sin `sudo` (recomendado)

```bash
sudo usermod -aG docker $USER
```

**Qué hace:** agrega tu usuario actual al grupo `docker`, para no tener que escribir `sudo` antes de cada comando `docker`.

⚠️ **Importante:** después de este comando debes **cerrar sesión y volver a entrar** (o reiniciar la PC) para que el cambio de grupo surta efecto. Alternativa rápida sin reiniciar:

```bash
newgrp docker
```

## 4. Verificar que Docker funciona correctamente

```bash
docker run hello-world
```

**Qué hace:** descarga una imagen mínima de prueba y la ejecuta. Si ves un mensaje que empieza con `"Hello from Docker!"`, todo quedó instalado correctamente.

## 5. Verificar Docker Compose

```bash
docker compose version
```

**Qué hace:** confirma que el plugin de Compose (`docker compose`, sin guion) está disponible — lo usaremos en la Fase de despliegue final con `docker-compose.yaml`.

---

## ✅ Checklist de la Fase 1

- [x] `git --version` responde correctamente (2.43.0).
- [x] `docker --version` responde correctamente (Docker CE 29.7.2).
- [x] `docker run hello-world` se ejecuta sin errores.
- [x] `docker compose version` responde correctamente (v5.5.0).
- [x] Puedo correr `docker` sin escribir `sudo`.
- [x] `docker.service` está `active (running)` y habilitado en el arranque.

## Nota especial de este entorno: Linux Mint

Este equipo corre **Linux Mint** (basado en Ubuntu 24.04 "noble"), no Ubuntu puro. Docker **no publica** repositorios para los codenames de Mint (ej. `zena`), así que el repositorio de APT se configuró usando `$UBUNTU_CODENAME` (`noble`) en vez de `$VERSION_CODENAME` (`zena`). Si en el futuro hay que tocar `/etc/apt/sources.list.d/docker.list`, recordar usar siempre `noble`, no el codename de Mint.

**Fase 1 completada el 30 de agosto de 2026.**

Siguiente: **Fase 2 — Obtener clave de OpenRouter y levantar el contenedor de Hermes con el asistente de configuración (`hermes setup`)**. Ver `02-instalar-hermes-openrouter.md`.
