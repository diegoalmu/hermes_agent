# Proyecto: Agente Hermes con Sub-Agentes (Docker)

**Autor:** _(tu nombre aquí)_
**Inicio del proyecto:** 30 de agosto de 2026
**Objetivo:** Desplegar el agente de IA open-source **Hermes Agent** (Nous Research) en Docker, sobre una PC con Linux, configurando dos sub-agentes especializados (perfiles):

- 🧑‍💻 **`coder`** — programación, git, debugging, deploys.
- 🗂️ **`oficina`** — documentos, reportes, correo, tareas administrativas.

Este repositorio sirve como **bitácora de portafolio** (evidencia de trabajo para escuela/portafolio profesional). Cada paso ejecutado, cada comando y cada decisión quedan documentados en los archivos Markdown numerados en la raíz de este repositorio.

## Fuentes oficiales usadas

> ⚠️ Nota importante: **no** usamos `hermes-ai.net` como fuente de comandos porque es un sitio comunitario **no oficial** (lo dice su propio aviso legal). Todos los comandos de este proyecto están verificados contra la documentación oficial:

- Documentación oficial: https://hermes-agent.nousresearch.com/docs/
- Repositorio oficial (GitHub, licencia MIT): https://github.com/NousResearch/hermes-agent

## Entorno de este proyecto

| Ítem | Valor |
|---|---|
| Máquina | PC con **Windows 11 + WSL2** (Ubuntu real, disco `D:`) — migrado desde Linux Mint en la Fase 5 |
| Método de despliegue | Docker (dentro de la Ubuntu de WSL2), imagen oficial `nousresearch/hermes-agent` |
| Proveedor de modelo (LLM) | NVIDIA NIM (`nvidia/nemotron-3-ultra-550b-a55b`) |
| Canal de comunicación | Telegram (bot "Leonardo", `@hermes_asiste_bot`) |
| Sub-agentes (skills) | `coder`, `oficina` (delegación vía `delegate_task` desde `SOUL.md`) |
| Dashboard web | Puerto 9119, accesible por WiFi local y por Tailscale |
| Automatizaciones activas | 3 recordatorios de clase (Lun/Mar/Mié) + 2 de tutoría Infotec |

## Índice de fases

- [`00-plan-proyecto.md`](00-plan-proyecto.md) — Plan general y arquitectura.
- [`01-prerrequisitos-y-docker.md`](01-prerrequisitos-y-docker.md) — Prerrequisitos e instalación de Docker (Linux Mint, fase histórica).
- [`02-instalar-hermes-openrouter.md`](02-instalar-hermes-openrouter.md) — Instalación de Hermes, elección de NVIDIA NIM, incidentes de red resueltos.
- [`03-subagentes-y-dashboard.md`](03-subagentes-y-dashboard.md) — Skills `coder`/`oficina` y Dashboard web (WiFi + Tailscale).
- [`04-recordatorios-clases.md`](04-recordatorios-clases.md) — Cron jobs de recordatorios de clase.
- [`05-migracion-windows-wsl2.md`](05-migracion-windows-wsl2.md) — Migración de Linux Mint a Windows/WSL2, reconstrucción completa de Hermes, bug de permisos por UID.
- [`06-persistencia-wsl2-hermes.md`](06-persistencia-wsl2-hermes.md) — Solución para mantener WSL2/Docker/Hermes vivos sin depender de una terminal abierta.
- [`07-pendientes-trabajo.md`](07-pendientes-trabajo.md) — Lista de pendientes de trabajo con memoria persistente y aviso diario.
- [`08-whatsapp-y-notificaciones.md`](08-whatsapp-y-notificaciones.md) — Conexión de WhatsApp como canal adicional y diagnóstico de notificaciones.
- [`09-incidente-proveedor-y-memoria.md`](09-incidente-proveedor-y-memoria.md) — Caída del proveedor de LLM, migración a otro proveedor y limpieza de memoria.

> ⚠️ **Nota histórica:** las Fases 0-4 se hicieron originalmente sobre Linux Mint. La Fase 5 documenta la migración completa a Windows + WSL2, incluyendo la reconstrucción de Hermes desde cero (se perdió la memoria/configuración anterior por un backup mal ejecutado — lección documentada en esa fase). El entorno **actual y vigente** del proyecto es Windows + WSL2.

## Cómo instalarlo

Este repositorio es una bitácora en Markdown, no una aplicación por sí misma. Para reproducir el entorno que documenta:

1. Sigue [`01-prerrequisitos-y-docker.md`](01-prerrequisitos-y-docker.md) para instalar Docker sobre Linux o WSL2.
2. Sigue [`02-instalar-hermes-openrouter.md`](02-instalar-hermes-openrouter.md) para instalar el [Hermes Agent](https://github.com/NousResearch/hermes-agent) oficial de Nous Research dentro de Docker.
3. Continúa con las fases siguientes en orden según lo que quieras reproducir (sub-agentes, cron jobs, WhatsApp, persistencia en WSL2, etc.).

## Cómo configurarlo

La configuración de Hermes (proveedor de modelo, tokens de Telegram/WhatsApp, usuarios permitidos) vive en `~/.hermes/.env` dentro del contenedor/host donde corre Hermes, **no en este repositorio**. Ningún archivo de este repo contiene credenciales reales; los IDs, tokens y direcciones IP que aparecían en los ejemplos fueron reemplazados por placeholders. Para tu propia instalación:

1. Copia la estructura de variables mencionada en cada fase (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, `TELEGRAM_HOME_CHANNEL`, etc.) a tu propio `~/.hermes/.env`.
2. Sustituye cada placeholder por tus propios valores (obtenidos de `@BotFather`, `@userinfobot`, tu propia red de Tailscale, etc.).

## Cómo subir esta carpeta a GitHub

```bash
cd bitacora-hermes
git init
git add .
git commit -m "Inicio del proyecto: bitácora Hermes Agent"
git branch -M main
git remote add origin https://github.com/TU_USUARIO/bitacora-hermes.git
git push -u origin main
```
