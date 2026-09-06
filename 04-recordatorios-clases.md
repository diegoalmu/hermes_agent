# Fase 4 — Recordatorios automáticos de clases (Cron Jobs)

## Objetivo

Recibir por Telegram un mensaje motivacional 15 minutos antes de cada clase de la carrera, generado dinámicamente por Leonardo (no un texto fijo repetido), reconociendo la desmotivación real del usuario ante preguntas repetitivas/irrelevantes en clase.

## Horario cubierto

| Día | Materia | Hora de clase | Aviso (15 min antes) |
|---|---|---|---|
| Lunes | Programación Estructurada | 19:00 hrs | 18:45 hrs |
| Martes | Cálculo Diferencial | 19:00 hrs | 18:45 hrs |
| Miércoles | Desarrollo Sostenible, Equidad y Responsabilidad Social | 20:30 hrs | 20:15 hrs |

Todos los horarios en zona horaria de Ciudad de México.

## 1. Configuración de zona horaria

Por defecto Hermes corre en UTC dentro del contenedor. Se configuró la zona horaria global vía variable de entorno:

```bash
docker exec -i hermes tee -a /opt/data/.env > /dev/null << 'EOF'

# Zona horaria para cron jobs
HERMES_TIMEZONE=America/Mexico_City
EOF
```

Y se recreó el contenedor agregando `-e HERMES_TIMEZONE=America/Mexico_City` al `docker run` (junto con las variables ya existentes: DNS, IPv6 deshabilitado, dashboard habilitado).

## 2. Creación de los 3 cron jobs

Usando `hermes cron create <expresión_cron> "<prompt>" --deliver telegram --name "<nombre>"`:

```bash
docker exec hermes hermes cron create "45 18 * * 1" \
  "[prompt motivacional para Programación Estructurada]" \
  --deliver telegram --name "Recordatorio-ProgramacionEstructurada"

docker exec hermes hermes cron create "45 18 * * 2" \
  "[prompt motivacional para Cálculo Diferencial]" \
  --deliver telegram --name "Recordatorio-CalculoDiferencial"

docker exec hermes hermes cron create "15 20 * * 3" \
  "[prompt motivacional para Desarrollo Sostenible]" \
  --deliver telegram --name "Recordatorio-DesarrolloSostenible"
```

**Nota de diseño:** en vez de un mensaje fijo copiado y pegado, cada job usa un **prompt** que le pide a Leonardo (con el modelo Nemotron 3 Ultra) redactar un mensaje motivacional nuevo cada semana — mencionando la materia, la hora, y reconociendo con empatía la pereza que generan las preguntas repetitivas/irrelevantes de otros compañeros en clase, sin reforzar la evitación, animando a entrar con buena actitud.

## 3. Verificación

```bash
docker exec hermes hermes cron list
```

Resultado confirmado:

```
58ad836387dd [active] Recordatorio-ProgramacionEstructurada — 45 18 * * 1 — next: 2026-09-07T18:45:00-06:00 — deliver: telegram
ef19c3e322e4 [active] Recordatorio-CalculoDiferencial       — 45 18 * * 2 — next: 2026-09-08T18:45:00-06:00 — deliver: telegram
9c6bd0f3e09b [active] Recordatorio-DesarrolloSostenible     — 15 20 * * 3 — next: 2026-09-09T20:15:00-06:00 — deliver: telegram
```

El offset `-06:00` confirma que la zona horaria de Ciudad de México quedó bien aplicada (Hermes ya no usa UTC internamente para este cálculo).

## 4. Prueba manual (sin esperar al próximo lunes)

```bash
docker exec hermes hermes cron run Recordatorio-ProgramacionEstructurada
```

`cron run` dispara el job de inmediato en segundo plano; el resultado llega como un mensaje nuevo de Telegram cuando termina (no bloquea la terminal).

## ✅ Checklist de la Fase 4

- [x] Zona horaria `America/Mexico_City` configurada globalmente.
- [x] 3 cron jobs creados y activos (lunes, martes, miércoles).
- [x] `hermes cron list` confirma horarios y zona horaria correctos.
- [x] Mensaje de prueba confirmado — tono correcto (corto, empático, motivador, sin reforzar la evitación).

## Resultado de la prueba manual

```
Cronjob Response: Recordatorio-ProgramacionEstructurada
(job_id: 58ad836387dd)
-------------
¡Ánimo, Diego! En 15 min arranca tu clase de Programación Estructurada a las 19:00.
Sé que a veces dan pereza las preguntas repetitivas o irrelevantes de algunos compañeros,
pero no dejes que eso te quite las ganas de aprender. Entra con buena actitud, filtra lo
que te sirve y sácale provecho al contenido — tú vas por tu crecimiento, no por ellos.
¡Dale con todo! 💪
```

✅ **Fase 4 completada.** Los 3 recordatorios quedan corriendo automáticamente cada semana sin intervención adicional.
