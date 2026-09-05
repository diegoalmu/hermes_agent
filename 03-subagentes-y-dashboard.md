# Fase 3 — Sub-agentes (Skills) y Dashboard Web

## Parte A: Sub-agentes "coder" y "oficina" vía Skills + delegate_task

### Enfoque elegido

En vez de crear perfiles completamente separados (cada uno con su propio bot de Telegram/WhatsApp — limitación real de Hermes: cada perfil requiere su propia sesión de WhatsApp y su propio token de Telegram), se optó por **un solo bot ("Leonardo") que delega internamente** usando la herramienta `delegate_task` combinada con dos **Skills** especializadas:

```
Telegram/WhatsApp → Leonardo (perfil "default")
                       ├── detecta tarea de programación → delegate_task con skill "coder"
                       └── detecta tarea de oficina       → delegate_task con skill "oficina"
```

### Skill "coder" creada

Archivo: `~/.hermes/skills/coder/SKILL.md`. Cubre: escritura/revisión/depuración de código, Git, terminal, despliegues. Instruye pedir confirmación antes de comandos destructivos (`git push --force`, borrados, etc.).

### Skill "oficina" creada

Archivo: `~/.hermes/skills/oficina/SKILL.md`. Cubre: redacción de correos, documentos, reportes, hojas de cálculo, organización de agenda/tareas. Tono profesional adaptable según destinatario.

### Comandos usados para crear ambas skills

```bash
docker exec hermes mkdir -p /opt/data/skills/coder
docker exec -i hermes tee /opt/data/skills/coder/SKILL.md > /dev/null << 'EOF'
[contenido del SKILL.md de coder]
EOF

docker exec hermes mkdir -p /opt/data/skills/oficina
docker exec -i hermes tee /opt/data/skills/oficina/SKILL.md > /dev/null << 'EOF'
[contenido del SKILL.md de oficina]
EOF
```

### Instrucción de delegación agregada a SOUL.md

Se agregó (con `tee -a`, sin sobreescribir el archivo existente) una sección a `~/.hermes/SOUL.md` indicando a Leonardo cuándo usar `delegate_task` con cada skill, y que pregunte si la tarea es ambigua.

### Verificación

```bash
docker exec hermes hermes skills list   # confirmar que "coder" y "oficina" aparecen junto a las 58 skills incluidas
docker restart hermes                    # para que relea SOUL.md y las skills nuevas
```

**Estado:** skills creadas y SOUL.md actualizado. Pendiente de una ronda de pruebas formales en Telegram con casos reales de programación vs. oficina para confirmar el comportamiento de delegación en la práctica.

---

## Parte B: Habilitar el Dashboard Web (puerto 9119)

Hermes incluye un panel web (similar en concepto a n8n/OpenClaw) para gestionar sesiones, configuración, API keys y servidores MCP desde el navegador, sin usar la terminal.

### Seguridad: por qué no se dejó sin contraseña

Hermes **bloquea por diseño** el dashboard sin autenticación cuando se expone más allá de `localhost` (`HERMES_DASHBOARD_INSECURE=1` es un no-op obsoleto). Además, la pestaña "Files" del dashboard puede exponer el `.env` completo (incluye API keys) a quien tenga acceso — bug conocido reportado en el repo oficial. Por eso se configuró autenticación básica (usuario/contraseña) desde el inicio.

### Pasos realizados

1. Generar credenciales aleatorias:
   ```bash
   openssl rand -hex 16   # contraseña
   openssl rand -hex 32   # secret de sesión
   ```
2. Agregarlas al `.env` (sin borrar el contenido existente):
   ```bash
   docker exec -i hermes tee -a /opt/data/.env > /dev/null << EOF

   # Web Dashboard
   HERMES_DASHBOARD_BASIC_AUTH_USERNAME=endoralex
   HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=<generada>
   HERMES_DASHBOARD_BASIC_AUTH_SECRET=<generada>
   EOF
   ```
3. Recrear el contenedor agregando `-e HERMES_DASHBOARD=1` y exponiendo el puerto `9119`:
   ```bash
   docker rm -f hermes
   docker run -d \
     --name hermes \
     --restart unless-stopped \
     --dns=8.8.8.8 \
     --dns=1.1.1.1 \
     --sysctl net.ipv6.conf.all.disable_ipv6=1 \
     -e HERMES_DASHBOARD=1 \
     -v ~/.hermes:/opt/data \
     -p 8642:8642 \
     -p 9119:9119 \
     nousresearch/hermes-agent gateway run
   ```
4. Confirmación en logs:
   ```
   Hermes Web UI → http://0.0.0.0:9119
   HERMES_DASHBOARD_READY port=9119
   ```

### Resultado

✅ **Dashboard funcionando** en `http://192.168.1.12:9119` (IP local de la PC en la red doméstica), protegido con usuario/contraseña. Login confirmado exitoso.

### Alcance actual y nota de seguridad

- Acceso **solo dentro de la red WiFi doméstica** — el puerto 9119 **no** se abrió en el router hacia internet, para no exponer el panel (y por extensión el `.env`) a escaneos públicos.
- Alternativa ya identificada para acceso remoto seguro sin abrir puertos: usar la IP de **Tailscale** de esta PC (`100.78.159.127`) en vez de la IP local — pendiente de implementar si se necesita acceso fuera de casa.

---

## ✅ Checklist de la Fase 3

- [x] Skill `coder` creada (`~/.hermes/skills/coder/SKILL.md`).
- [x] Skill `oficina` creada (`~/.hermes/skills/oficina/SKILL.md`).
- [x] `SOUL.md` actualizado con reglas de delegación.
- [ ] Pendiente: pruebas formales de delegación en Telegram (casos de programación y de oficina).
- [x] Dashboard web habilitado en puerto 9119 con autenticación básica.
- [x] Login al dashboard confirmado exitoso desde la red local.
- [ ] Pendiente (opcional): exponer el dashboard vía Tailscale para acceso remoto sin abrir puertos.

Siguiente: probar la delegación coder/oficina con casos reales, y decidir si se implementa acceso remoto vía Tailscale al dashboard.
