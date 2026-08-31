# Proyecto: Agente Hermes con Sub-Agentes (Docker)

**Autor:** _(tu nombre aquí)_
**Inicio del proyecto:** 30 de agosto de 2026
**Objetivo:** Desplegar el agente de IA open-source **Hermes Agent** (Nous Research) en Docker, sobre una PC con Linux, configurando dos sub-agentes especializados (perfiles):

- 🧑‍💻 **`coder`** — programación, git, debugging, deploys.
- 🗂️ **`oficina`** — documentos, reportes, correo, tareas administrativas.

Este repositorio sirve como **bitácora de portafolio** (evidencia de trabajo para escuela/portafolio profesional). Cada paso ejecutado, cada comando y cada decisión quedan documentados en Markdown dentro de `fases/`.

## Fuentes oficiales usadas

> ⚠️ Nota importante: **no** usamos `hermes-ai.net` como fuente de comandos porque es un sitio comunitario **no oficial** (lo dice su propio aviso legal). Todos los comandos de este proyecto están verificados contra la documentación oficial:

- Documentación oficial: https://hermes-agent.nousresearch.com/docs/
- Repositorio oficial (GitHub, licencia MIT): https://github.com/NousResearch/hermes-agent

## Entorno de este proyecto

| Ítem | Valor |
|---|---|
| Máquina | PC local con Linux |
| Método de despliegue | Docker (imagen oficial `nousresearch/hermes-agent`) |
| Proveedor de modelo (LLM) | OpenRouter, modelo gratuito `:free` (ej. NVIDIA Nemotron 3 Ultra) |
| Canal de comunicación | Telegram (a configurar en fase posterior) |
| Sub-agentes (perfiles) | `coder`, `oficina` |

## Índice de fases

- [`fases/00-plan-proyecto.md`](fases/00-plan-proyecto.md) — Plan general y arquitectura.
- [`fases/01-prerrequisitos-y-docker.md`](fases/01-prerrequisitos-y-docker.md) — Fase 0: prerrequisitos e instalación de Docker.
- _(Las siguientes fases se irán agregando aquí conforme avancemos: instalación de Hermes, configuración de Telegram, creación de sub-agentes, docker-compose final, pruebas.)_

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
