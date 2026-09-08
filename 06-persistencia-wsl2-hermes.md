# Fase 6 — Persistencia de WSL2/Hermes al cerrar la terminal

## Problema

Cada vez que se cerraba la ventana de Windows Terminal (pestaña de Ubuntu), Telegram avisaba de inmediato:

```
⚠️ Gateway shutting down — Your current task will be interrupted.
```

Confirmado con `wsl -l -v` desde PowerShell: la distro Ubuntu pasaba a `Stopped` en cuanto se cerraba la ventana, llevándose a Docker (y por lo tanto a Hermes) con ella — aunque el contenedor tuviera `--restart unless-stopped`, eso solo protege contra caídas del propio contenedor, no contra que la VM de WSL2 completa se apague.

## Intentos que NO funcionaron (en orden)

1. **`vmIdleTimeout=-1` en `.wslconfig`** — pensado para casos de apagado por inactividad tras varios minutos. No aplicaba aquí: el apagado ocurría de inmediato al cerrar la ventana, no por timeout, así que esta opción nunca fue la causa real.

2. **Revisar perfiles "fantasma" de Ubuntu en Windows Terminal** — se encontró un perfil viejo marcado como "ya no detectado" (de una distro anterior, reimportada a `D:`), pero era basura inofensiva, no la causa del apagado. Se eliminó por limpieza.

3. **Tarea programada de Windows con la cuenta del propio usuario (`diego`)** — falló porque la cuenta es de tipo Microsoft en línea (`diego1384@hotmail.com`), y el modo "ejecutar tanto si el usuario inició sesión como si no" requiere su contraseña real de Microsoft (no el PIN de Windows Hello), la cual no se tenía a la mano.

4. **Tarea programada con la cuenta `NT AUTHORITY\SYSTEM`** — evitó el problema de la contraseña, pero la tarea fallaba con código `0xFFFFFFFF` al ejecutarse. Causa: **WSL2 depende de un servicio (`LxssManager`) ligado a una sesión de usuario real con escritorio — no funciona correctamente lanzado desde la cuenta `SYSTEM`.**

## Solución que sí funcionó: acceso directo en la carpeta de Inicio de Windows

En vez de pelear con el Programador de tareas, se creó un acceso directo simple en la carpeta de inicio de sesión de Windows:

**Paso 1 — Abrir la carpeta de Inicio del usuario**

```
Win + R → shell:startup
```

**Paso 2 — Crear un acceso directo nuevo** (clic derecho → Nuevo → Acceso directo) con el destino:

```
wsl.exe -d Ubuntu -- sleep infinity
```

**Paso 3 — En Propiedades del acceso directo, cambiar "Ejecutar:" de "Ventana normal" a "Minimizada"**

Esto hace que, cada vez que Windows inicia sesión con el usuario `diego` (proceso normal, sin necesidad de contraseñas adicionales ni cuentas especiales), se lance una sesión de WSL2 que se queda viva ejecutando `sleep infinity` — manteniendo la VM completa (y con ella Docker y Hermes) encendida indefinidamente, sin importar cuántas ventanas de terminal se abran o cierren después.

**Resultado visual:** aparece un ícono de pingüino de Linux en la barra de tareas (minimizado). Mientras ese ícono siga ahí, WSL2/Docker/Hermes se mantienen vivos.

## Verificación

```powershell
wsl -l -v
# Ubuntu   Running   2   (antes y después de cerrar todas las demás terminales)
```

```bash
docker ps
# hermes sigue "Up", sin reiniciarse
```

Confirmado en producción: el cron de las 18:45 (recordatorio de clase) llegó puntual por Telegram con todas las terminales cerradas excepto el ícono de pingüino.

## 📌 Regla para el futuro

- **Nunca cerrar el ícono de pingüino de la barra de tareas** — es el que mantiene viva toda la infraestructura (Hermes, Docker, WSL2).
- Si se cierra por accidente (clic derecho → cerrar ventana, o similar), basta con volver a hacer doble clic en el acceso directo `Mantener WSL activo` dentro de `shell:startup` para relanzarlo manualmente sin reiniciar la PC.
- El acceso directo se ejecuta solo en cada inicio de sesión de Windows — no requiere configuración adicional después de reiniciar la PC.
- **WSL2 no es compatible con ejecución bajo la cuenta `SYSTEM`** — cualquier automatización futura relacionada con WSL2 debe correr bajo un usuario real con sesión iniciada, nunca bajo `SYSTEM` ni tareas programadas "sin sesión".

**Fase 6 completada el 7 de septiembre de 2026.**
