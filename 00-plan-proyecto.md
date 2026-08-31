# Fase 0 — Plan General del Proyecto

## Objetivo
Levantar Hermes Agent en Docker sobre Linux local, con sub-agentes (perfiles) especializados en programación y en oficina, usando un modelo LLM gratuito vía OpenRouter, y documentar todo para portafolio.

## ¿Qué es Hermes Agent? (resumen técnico verificado)

Hermes Agent es un agente de IA autónomo y auto-mejorable, creado por **Nous Research**, licencia **MIT**. A diferencia de un chatbot, tiene:

- **Loop de razonamiento (agent loop):** itera hasta resolver la tarea.
- **Memoria persistente** entre sesiones (carpeta `memories/`).
- **Skills auto-generadas:** el agente crea sus propios manuales de procedimiento (SOPs) según patrones de uso.
- **Cron jobs** integrados (automatizaciones programadas).
- **Soporte MCP** (Model Context Protocol) para conectar herramientas externas.
- **Perfiles / Bot Mode:** cada perfil es un sub-agente independiente (modelo propio, memoria propia, skills propias, personalidad propia vía `SOUL.md`). Esto es justo lo que usaremos para tener un sub-agente de programación y uno de oficina.
- **7 backends de terminal:** local, Docker, SSH, Daytona, Singularity, Modal, Vercel Sandbox — nosotros correremos Hermes DENTRO de Docker.

## Arquitectura elegida para este proyecto

```
PC Linux (host)
 └── Docker
      └── Contenedor "hermes" (imagen nousresearch/hermes-agent)
           ├── Perfil "default"   → agente orquestador / uso general
           ├── Perfil "coder"     → sub-agente de programación
           └── Perfil "oficina"   → sub-agente de tareas de oficina
      Volumen persistente: ~/.hermes  →  /opt/data (dentro del contenedor)
```

Un solo contenedor puede supervisar varios perfiles a la vez (gracias a `s6-overlay` como proceso principal), así que **no necesitamos un contenedor por sub-agente** — es más simple y más liviano.

## Proveedor de modelo (LLM)

- **Proveedor:** OpenRouter (https://openrouter.ai)
- **Modelo inicial:** un modelo con sufijo `:free` (ej. `nvidia/nemotron-3-ultra:free` — sujeto a cambio, hay que confirmar el nombre exacto en https://openrouter.ai/models filtrando por "Free" el día que lo configuremos).
- **Límites del nivel gratis:** ~20 solicitudes/minuto, 50/día sin tarjeta (1,000/día si se agregan $10 USD de crédito, opcional).
- **Por qué OpenRouter y no Grok/DeepSeek/Gemini directos:** a la fecha de este proyecto, Grok no ofrece API gratuita, y DeepSeek/Gemini no tienen variantes `:free` directas en OpenRouter. Los modelos `:free` de OpenRouter (Nemotron, Gemma, etc.) sí cumplen el requisito mínimo de Hermes (64,000 tokens de contexto) de sobra.
- **Ventaja:** cambiar de modelo después es un solo comando (`hermes model`), sin tocar el resto de la configuración.

## Fases del proyecto

1. **Fase 0** — Prerrequisitos y Docker (este documento + `01-prerrequisitos-y-docker.md`).
2. **Fase 1** — Repositorio de GitHub para la bitácora (hecho, ver `README.md`).
3. **Fase 2** — Levantar el contenedor de Hermes y correr el asistente de configuración (`setup`), incluyendo la clave de OpenRouter.
4. **Fase 3** — Configurar Telegram como canal de comunicación (BotFather).
5. **Fase 4** — Crear el sub-agente `coder` (perfil con skills de programación).
6. **Fase 5** — Crear el sub-agente `oficina` (perfil con skills de documentos/administración).
7. **Fase 6** — `docker-compose.yaml` definitivo con límites de recursos y arranque automático.
8. **Fase 7** — Pruebas: comunicación entre sub-agentes (`message_agent`), cron jobs, backup a GitHub.
9. **Fase 8** — Documentación final y checklist de seguridad (credenciales, `.env`, backups).

## Buenas prácticas de seguridad que seguiremos

- Nunca escribir API keys directamente en el chat del agente ni en commits de GitHub.
- Todas las claves van en `~/.hermes/.env` (fuera del repositorio de bitácora).
- El repositorio de GitHub de la bitácora **no** debe incluir la carpeta `~/.hermes` real, solo documentación y, si acaso, plantillas de configuración sin datos sensibles.
