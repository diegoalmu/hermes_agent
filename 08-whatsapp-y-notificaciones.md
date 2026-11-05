# Fase 8 — Conexión de WhatsApp y diagnóstico de notificaciones (MIUI)

## Contexto: por qué se evaluó agregar WhatsApp

Diego reportó que las notificaciones de Telegram no le llegaban a menos que abriera la app manualmente, a pesar de tener la app "trabajando en segundo plano" — sospechando que Telegram fallaba, se consideró WhatsApp como alternativa o respaldo.

## Parte A: Conexión de WhatsApp (Baileys bridge — gratuito)

### Aclaración de costos

Hermes soporta 2 formas de conectar WhatsApp:

| | Baileys (WhatsApp Web bridge) — **usada aquí** | WhatsApp Business Cloud API (Meta) |
|---|---|---|
| Costo | Gratis | Requiere cuenta Meta Business, verificación, posibles costos por volumen |
| Conexión | Código QR (como WhatsApp Web) | Número de negocio dedicado + aprobación de Meta |
| Uso ideal | Personal (este caso) | Bots de atención a clientes a gran escala |

Se usó **Baileys**, sin ningún costo.

### Decisión: self-chat vs. número separado

El wizard (`hermes whatsapp`) pregunta entre "Separate bot number" (número dedicado exclusivo para el bot) o "Personal number (self-chat)" (hablarle al bot en el chat de "Mensaje a mí mismo").

Diego tiene 2 números, pero **ambos en uso activo** (uno personal con familia/amigos, otro de trabajo que se apaga fuera de horario laboral) — ninguno estaba libre para dedicarlo 100% al bot, y el de trabajo no puede quedar siempre encendido (requisito para mantener viva la sesión de WhatsApp). **Se eligió self-chat con el número personal** (el que siempre está encendido).

### Comandos usados

```bash
docker exec -it hermes hermes gateway setup
# → seleccionar WhatsApp → configurar allowed users y home channel

docker exec -it hermes hermes whatsapp
# → flujo de emparejamiento dedicado → elegir opción 2 (Personal number / self-chat)
# → escanear QR con WhatsApp del número personal
```

### Incidencia: código de emparejamiento (pairing code)

Al escribirse a sí mismo por primera vez, WhatsApp respondió pidiendo aprobación adicional:
```
Hi~ I don't recognize you yet!
Here's your pairing code: GFETA5L5
Ask the bot owner to run: hermes pairing approve whatsapp GFETA5L5
```
Causa probable: el número no coincidía exactamente con el formato guardado en la lista de permitidos (se había preparado inicialmente para el escenario de "número separado" con 2 números, antes de decidir el self-chat). Resuelto con:
```bash
docker exec hermes hermes pairing approve whatsapp GFETA5L5
```
Después de esto, el self-chat respondió con normalidad, con el prefijo `⚕ Hermes Agent`.

### Confirmación en logs

```
[Whatsapp] Dependencies installed
[Whatsapp] Bridge ready (status: connected)
[Whatsapp] Bridge started on port 3000
```
Sin conflicto con Telegram (`[Telegram] Connected to Telegram (polling mode)` sigue sano en paralelo).

### Nota importante sobre entrega de cron jobs

Se confirmó (vía `docker exec hermes hermes cron create --help`) que el flag `--deliver` **no soporta múltiples plataformas en un solo job** — solo una por job (`telegram`, `discord`, `signal`, `platform:chat_id`, etc.). Si en el futuro se quiere redundancia real (mismo aviso por Telegram y WhatsApp), la única forma es **duplicar el cron job**, uno por plataforma. No se implementó esta duplicación porque la causa raíz del problema de notificaciones (ver Parte B) no era de Telegram ni de Hermes.

## Parte B: Diagnóstico real — notificaciones de Telegram no llegaban en segundo plano

### Causa raíz encontrada

El dispositivo de Diego es un **Xiaomi Redmi 10c**, con **MIUI** — una capa de Android conocida por matar agresivamente apps en segundo plano para "ahorrar batería", mucho más agresiva que Android puro. Esto explicaba por qué Telegram no mostraba notificaciones push a menos que se abriera manualmente la app — **no era un problema de Hermes, Docker, ni de Telegram como servicio.**

### Ajustes aplicados en el Redmi 10c (MIUI)

1. **Batería y rendimiento → Uso de batería de apps → Telegram → "Sin restricciones"**
2. **Aplicaciones → Permisos → Inicio automático (Autostart) → activar Telegram**
3. **Batería y rendimiento → Iniciar apps → Telegram → "Administrar manualmente"** con: Inicio automático, Inicio secundario, y Ejecutar en segundo plano — las 3 activadas
4. **Apps → Permisos → Notificaciones → Telegram** → activadas, prioridad alta (no silenciosa)
5. Confirmar que el modo **"Ultra ahorro de batería"** del sistema no esté activo de forma permanente

### Prueba de confirmación

Con Telegram cerrado por completo (no solo en segundo plano, deslizado fuera de apps recientes) y sin abrirlo:
```bash
docker exec hermes hermes cron run Recordatorio-ProgramacionEstructurada
```
**Resultado: el mensaje llegó correctamente a Telegram sin haber abierto la app.** Confirma que el ajuste de MIUI resolvió el problema de raíz.

### Decisión final

Se mantiene **Telegram como canal principal** para todos los avisos (clases, tutorías, pendientes de trabajo) — es la plataforma con mejor soporte y estabilidad dentro de Hermes ("más noble", en palabras de Diego). **WhatsApp queda conectado y funcional como canal adicional/de respaldo**, disponible para usar en cualquier momento (self-chat), pero no se duplicaron los cron jobs hacia él dado que el problema real ya quedó resuelto en el propio Telegram.

## ✅ Checklist de la Fase 8

- [x] WhatsApp conectado vía Baileys (gratuito), modo self-chat con número personal.
- [x] Incidencia de pairing code resuelta (`hermes pairing approve`).
- [x] Confirmado funcionando: respuesta de prueba recibida en WhatsApp.
- [x] Causa raíz de notificaciones perdidas identificada: restricciones agresivas de batería de MIUI (Xiaomi Redmi 10c), no un problema de Hermes/Telegram.
- [x] Ajustes de batería/autostart aplicados en el dispositivo.
- [x] Confirmado con prueba real: notificación de cron job recibida en Telegram sin abrir la app manualmente.
- [ ] Pendiente (opcional, baja prioridad): simplificar `WHATSAPP_ALLOWED_USERS` en el `.env` a un solo número, ya que el plan original de "2 números" quedó descartado a favor de self-chat con uno solo.

**Fase 8 completada el 8 de septiembre de 2026.**
