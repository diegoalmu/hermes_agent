# Fase 7 — Pendientes de trabajo (ZUMMAT) con memoria persistente

## Objetivo

Poder decirle a Leonardo los pendientes de trabajo conforme surgen, que los recuerde de forma persistente (no solo durante una conversación), que los liste bajo pedido o automáticamente cada mañana, y que los elimine cuando Diego confirme que ya los realizó.

## Decisión técnica: memoria persistente, no el tool `todo`

Hermes tiene dos sistemas que parecen similares pero no lo son:
- **`todo`** (Task Planning): scratch-pad **de sesión** — se pierde al terminar la conversación. No sirve para este caso.
- **`memory`** (memoria persistente): sobrevive entre sesiones, días y reinicios del contenedor — es la base correcta para una lista de pendientes real.

Se construyó esta funcionalidad sobre **memoria persistente**.

## 1. Skill creada: `pendientes-trabajo`

Archivo: `~/.hermes/skills/pendientes-trabajo/SKILL.md`. Instruye a Leonardo a:
- Guardar pendientes nuevos en memoria bajo el tema "Pendientes de trabajo (ZUMMAT)".
- Listarlos de forma breve y numerada cuando se le pregunte.
- Eliminar un pendiente de la memoria cuando Diego confirme que ya lo hizo (con reconocimiento flexible, no exacto).
- No mezclar estos pendientes con los de Samate o la escuela.

## 2. Siembra inicial de pendientes — vía chat, no vía terminal

**Decisión de diseño:** la memoria de Hermes la escribe el propio agente al usar su herramienta `memory` — sembrarla directo desde archivos por terminal arriesga un formato distinto al que el agente espera leer después. Por eso la lista inicial se mandó como mensaje normal de Telegram, dejando que Leonardo la guardara él mismo.

**Mensaje real enviado por Diego (8 pendientes, con contexto para diferenciarlos):**
```
Agrega estos pendientes de trabajo a tu memoria, la numeración no es por
importancia, sino para separarlos, tenia que saber cuales son algunos de
mis pendientes de la oficina
1. Recoger llanta montacargas [Cliente A]
2. Entregar luminarias [Cliente B]
3. Regresar la llanta del montacargas a [Proveedor X]
4. Recolectar paquetexpress 4 luminarias para [Cliente C]
5. Comprobantes de pago [Cliente A] de 3 facturas
6. imprimir facturas y OC de [Cliente C]
7. entregar todas las luminarias de [Cliente C]
8. recoger 10 luminarias de [Cliente C] en paquetexpress
```

Respuesta de Leonardo: confirmó el guardado de los 8 pendientes en memoria.

**Nota de rendimiento:** esta ejecución tardó ~3 minutos en responder (hardware con menos RAM/CPU que el equipo anterior, post-migración a WSL2). Se decidió no forzar tiempos de respuesta más rápidos dado que este uso no es de urgencia inmediata — es aceptable para un caso de uso de recordatorio diario, no de alerta en tiempo real.

## 3. Cron diario a las 9:00 am (todos los días)

```bash
docker exec hermes hermes cron create "0 9 * * *" \
  "Revisa tu memoria persistente y busca la lista de 'Pendientes de trabajo (ZUMMAT)' de Diego. Mándale la lista completa, numerada y breve. Si no hay pendientes, dile de forma breve y positiva que está limpio por ahora. No agregues relleno ni mensaje motivacional aquí, solo la lista clara." \
  --deliver telegram --name "Pendientes-Trabajo-Diario"
```

## 4. Uso normal esperado (sin cron, en cualquier momento)

- **Agregar:** "agrega este pendiente: ..." → Leonardo lo guarda en memoria.
- **Consultar:** "¿qué pendientes tengo?" → Leonardo lista lo que tiene guardado.
- **Cerrar uno:** "ya hice X" / "ya entregué X" → Leonardo lo identifica y lo elimina de la memoria, confirmando cuál borró.

## ✅ Checklist de la Fase 7

- [x] Skill `pendientes-trabajo` creada.
- [x] 8 pendientes iniciales sembrados en memoria vía chat (no vía terminal).
- [x] Cron diario de las 9:00 am creado.
- [ ] Pendiente: confirmar con `¿qué pendientes tengo?` que los 8 quedaron guardados correctamente, tal cual se enviaron.
- [ ] Pendiente: probar el flujo de cierre ("ya hice X") con al menos un pendiente real.
- [ ] Pendiente: confirmar que el cron de las 9:00 am se dispara correctamente al día siguiente.

Siguiente: validar el listado completo y probar el ciclo de vida completo (agregar → consultar → cerrar) con un caso real.
