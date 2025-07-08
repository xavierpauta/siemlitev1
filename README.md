# Documentación del Proyecto: SIEMLite v1

Este documento describe cómo la implementación actual del proyecto SIEMLite cumple con todos los requisitos funcionales y técnicos especificados en el enunciado del proyecto parcial.

### Requerimiento (1): Manejo de Parámetros

#### a. Nombres de los servicios a monitorear (Mínimo 2)
El programa está diseñado para aceptar una lista variable de nombres de servicios como argumentos de línea de comandos. Esta lógica se encuentra en `main.c`, donde `argc` y `argv` son procesados. Los argumentos a partir de la cuarta posición (`argv[3]`, `argv[4]`, etc.) son interpretados como los nombres de los servicios a monitorear. Se realiza una validación para asegurar que se proporcionen al menos 2 servicios.

#### b. El tiempo de actualización de la información
El tiempo de actualización se proporciona como el primer argumento en la línea de comandos (`argv[1]`). En `main.c`, este valor es convertido a un entero y almacenado en la variable `update_time`. Esta variable se utiliza en dos lugares clave:
1.  **En `parent_logic.c`:** Como el valor de `timeout` para la llamada al sistema `select()`. Esto asegura que el dashboard se refresque periódicamente, incluso si no se reciben nuevos datos.
2.  **En `child_logic.c`:** Como el argumento para la función `sleep()`, pausando cada proceso hijo entre ciclos de recolección de logs.

### Requerimiento (2): Manejo de Niveles de Prioridad
El sistema considera los niveles de prioridad `alert`, `err`, `notice`, `info`, y `debug`. La implementación se encuentra en la función `parse_journal_line` dentro de `child_logic.c`. Esta función recibe cada línea de log, la convierte a minúsculas, y utiliza `strstr()` para buscar palabras clave asociadas a cada nivel de prioridad. De esta forma, cada log es clasificado y su contador correspondiente en la estructura `ServiceStats` es incrementado.

### Requerimiento (3): Dashboard en Modo Texto
Se ha implementado un dashboard dinámico en la terminal. La función `render_dashboard` en `parent_logic.c` es responsable de esta tarea. En cada ciclo del bucle principal del padre, esta función:
1.  Limpia la pantalla de la terminal usando secuencias de escape ANSI (`\033[H\033[J`).
2.  Imprime una cabecera fija y una tabla formateada con los contadores más recientes (`Errores`, `Alertas`, `Info`) para cada servicio.
3.  Muestra un `timestamp` con la hora de la última actualización.
4.  La llamada a `fflush(stdout)` asegura que la actualización del dashboard sea visible de inmediato.

### Requerimiento (4): Uso de `journalctl` y `exec`
Para obtener los logs de manera eficiente, cada proceso hijo (`child_logic.c`) crea a su vez un proceso nieto. Este proceso nieto tiene la única responsabilidad de ejecutar el comando `journalctl` mediante la llamada al sistema `execvp()`. Los argumentos para `journalctl` se construyen dinámicamente para filtrar por unidad de servicio (`-u`) y por tiempo (`--since`). La salida estándar del nieto es redirigida a un pipe para que el proceso hijo pueda leerla y procesarla sin bloquearse.

### Requerimiento (5): Sistema de Alertas y Threshold
El sistema de alertas se activa cuando el número de errores supera un umbral (`threshold`) configurable.
- **Threshold Configurable:** El umbral se pasa como el segundo argumento (`argv[2]`) y se almacena en la variable `error_threshold`.
- **Detección:** En el bucle principal de `parent_logic.c`, después de recibir nuevos datos de un hijo, se comprueba si `current_stats[i].err_count >= error_threshold`.
- **Notificación:** Si se supera el umbral, el proceso padre crea un nuevo proceso "notificador" efímero. Este proceso ejecuta la lógica en `alerter_logic.c`, que utiliza `curl` para enviar una petición `POST` con un payload JSON a un servicio de webhooks, simulando una integración con una API de mensajería como Twilio para WhatsApp.

### Consideración Técnica: Procesos Hijos y Comunicación entre Procesos (IPC)
La arquitectura del programa se basa en el uso de multiprocesamiento para el monitoreo concurrente.
- **Procesos Hijos:** En `main.c`, se crea un proceso hijo por cada servicio a monitorear mediante un bucle que llama a `fork()`.
- **Comunicación (IPC):** El mecanismo de comunicación elegido es `pipes`. Antes de cada `fork()`, se crea un pipe con `pipe()`. El proceso hijo utiliza el descriptor de escritura para enviar la estructura `ServiceStats` llena, y el proceso padre utiliza el descriptor de lectura correspondiente junto con `select()` para recibir los datos de todos los hijos de forma asíncrona.
