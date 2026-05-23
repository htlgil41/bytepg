# 🐘pg_basebackup🐘

pg_basebackup es una utilidad nativa de línea de comandos de PostgreSQL diseñada para crear copias de seguridad físicas (binarias) de todo un clúster de bases de datos en ejecución. Se utiliza principalmente para la recuperación ante desastres o para configurar servidores de réplica (streaming replication).

- Realiza una copia binaria (física) del directorio de datos.
- Puede obtener los WAL necesarios durante la copia (opción -X stream).
- Crea automáticamente archivos de configuración para que el servidor copiado arranque como standby.
- Soporta copia en paralelo, compresión y slots de replicación.

Antes de ejecutar este comando obeservemos una de las configuraciones que debe de tener el nodo (`Permitir la conexión de replicación desde la IP del standby (o desde cualquier IP, según tu entorno)`):

```
# Nivel de escritura de WAL (replica es suficiente; 'logical' también vale)
wal_level = replica

# Número de conexiones de replicación concurrentes (al menos 1)
max_wal_senders = 5

# Número de slots de replicación (si se usan, al menos 1)
max_replication_slots = 5

# (Opcional) Activar sumarios de WAL para hot standby
wal_log_hints = on   # recomendado si usas pg_rewind más adelante

# (Opcional) Configurar retención de WAL en el primario mientras el standby se pone al día
wal_keep_size = 1024   # en MB, o usar replication slots (mejor)
```

Ok ahora veamos la sintaxis basica `pg_basebackup -h <primario> -p <puerto> -U <usuario> -D <destino> [opciones]`

donde la -h es host -p puerto -U usuario -D carpeta donde se copiaran los datos y las opciones son las siguientes:


| Opción | Descripción |
| :--- | :--- |
| **`-D, --pgdata=DIR`** | Directorio donde se escribirá la copia del clúster (debe estar vacío o no existir). |
| **`-F, --format=p\|t`** | Formato: `p` (plain, directorio con archivos, por defecto) o `t` (tar, empaquetado). |
| **`-X, --wal-method=fetch\|stream`** | Cómo obtener los WAL durante la copia. `stream` (recomendado) mantiene una conexión de streaming mientras se copia. `fetch` (por defecto) copia los WAL al final. |
| **`-R, --write-recovery-conf`** | Crea automáticamente el archivo `standby.signal` y añade parámetros de conexión al primario en `postgresql.auto.conf` (PostgreSQL ≥12). En versiones anteriores genera `recovery.conf`. |
| **`-P, --progress`** | Muestra barra de progreso. |
| **`-v, --verbose`** | Salida detallada. |
| **`-C, --create-slot`** | Crea un slot de replicación temporal con el nombre dado por `-S` y lo elimina al finalizar. Muy útil para asegurar que el primario retiene los WAL mientras se copia. |
| **`-S, --slot=SLOTNAME`** | Nombre del slot de replicación a usar o crear (junto con `--create-slot`). |
| **`--checkpoint=fast\|spread`** | Controla el checkpoint inicial: `fast` lo fuerza rápido (más I/O), `spread` distribuye la escritura (por defecto). |
| **`-c, --checkpoint=...`** | Sinónimo de `--checkpoint`. |
| **`-l, --label=LABEL`** | Etiqueta para el backup (visible en vistas como `pg_stat_progress_basebackup`). |
| **`-j, --jobs=N`** | Número de procesos paralelos para copiar tablas/índices (solo con formato tar y directorio, PostgreSQL ≥15). |
| **`-z, --gzip`** | Comprimir salida tar con gzip. |
| **`-Z, --compress=LEVEL`** | Nivel de compresión (0-9) o `none`. |
| **`--no-slot`** | Evita la creación automática de un slot temporal. |
| **`--no-verify-checksums`** | No verificar checksums de datos (si están activados). |
| **`--waldir=DIR`** | Directorio separado para los WAL (si se quiere fuera de `pg_wal`). |
| **`--max-rate=RATE`** | Limita la tasa de transferencia (ej. `10M`). |
| **`--help`** | Muestra todas las opciones. |

Perfecto ahora vamos con un ejemplo teniendo la informacion del primario **`192.168.1.10, puerto 5432, usuario replicador`** y el de replica **`192.168.1.20, directorio de datos /var/lib/postgresql/16/main (vacío)`**

```
pg_basebackup -h 192.168.1.10 -p 5432 -U replicador \
  -D /var/lib/postgresql/16/main \
  -F p -X stream -R -P -v \
  --checkpoint=fast --create-slot -S standby_slot \
  --label="backup_inicial_standby"
```

| Parámetro / Opción | Descripción | Detalle / Nota Clave |
| :--- | :--- | :--- |
| **`-D`** | **Destino.** Especifica el directorio donde se guardará la copia de seguridad. | Obligatorio para definir la ruta de salida. |
| **`-F p`** | **Formato plano.** Copia los archivos directamente en su formato original (sin empaquetar). | Es el formato por defecto y el necesario para levantar un *standby* directamente. |
| **`-X stream`** | **Stream de WAL.** Inicia un flujo de WAL en paralelo durante la copia. | Asegura que no falten segmentos al final del proceso de respaldo. |
| **`-R`** | **Crear Standby.** Genera el archivo `standby.signal` y añade las líneas de conexión a `postgresql.auto.conf` `primary_conninfo = 'host=192.168.1.10 port=5432 user=replicador password=secreto'` `primary_slot_name = 'standby_slot'`. | Automatiza la configuración del nodo como réplica. |
| **`-P -v`** | **Progreso y detalle.** Muestra información en tiempo real sobre el avance y la verbosidad del proceso. | Ideal para monitorear copias de seguridad pesadas. |
| **`--checkpoint=fast`** | **Checkpoint rápido.** Fuerza un checkpoint inmediato en el nodo primario. | Útil para evitar esperas largas antes de que inicie la copia. |
| **`--create-slot -S standby_slot`** | **Slot de replicación permanente.** Crea el slot `standby_slot` en el primario antes de iniciar la copia. | El primario retiene los WAL mientras el *standby* se sincroniza. El slot **no se elimina** al terminar. Si prefieres no usarlo, omítelo y configura `wal_keep_size`. |
| **`--label`** | **Etiqueta.** Asigna un nombre identificativo al backup. | Permite rastrear y reconocer el backup en las estadísticas del nodo primario. |

Veamos estos ajustes recomendados en la replica: 
```
# Activar hot standby (permitir consultas en réplica)
hot_standby = on

# Para permitir retroalimentación al primario (útil para evitar conflictos de limpieza)
hot_standby_feedback = on

# Ajustes de memoria/recursos según el hardware del standby
shared_buffers = ...
```

## Consideraciones importantes y solución de problemas

*   **Permisos y Accesos:** El usuario del sistema operativo que ejecuta `pg_basebackup` debe tener permisos de escritura en el directorio de destino (`-D`). Por el lado de la base de datos, el usuario asignado debe contar con los atributos `REPLICATION` y acceso `LOGIN`.

> [!WARNING]
> **Riesgo crítico con Slots de replicación:** Si dejas un slot creado manualmente y el *standby* deja de consumir los WAL, el nodo primario retendrá estos segmentos indefinidamente, lo que **puede llenar el disco por completo**.
> *   Monitorea constantemente el estado usando la vista `pg_replication_slots`.
> *   Usa la opción `--create-slot` únicamente si tienes la certeza de que mantendrás el *standby* conectado.
> *   Si remueves la réplica, elimina el slot manualmente en el primario ejecutando: `SELECT pg_drop_replication_slot('nombre_del_slot');`.

*   **Seguridad de la contraseña en `primary_conninfo`:** Almacenar credenciales en texto plano dentro de la configuración no es una práctica segura. La alternativa recomendada es utilizar un archivo `~/.pgpass` en el directorio *home* del usuario que ejecuta el servicio de PostgreSQL (comúnmente `postgres`).
    *   **Formato del archivo:** `host:puerto:replication:usuario:contraseña`
    *   **Permisos requeridos:** Debe tener permisos estrictos `0600` (`chmod 600 ~/.pgpass`).
    *   Una vez configurado, puedes omitir de forma segura el parámetro `password=` en el `primary_conninfo`.

*   **Compatibilidad con versiones anteriores (PostgreSQL 11 o inferior):** Ten en cuenta que en versiones anteriores a PostgreSQL 12 no existía el archivo `standby.signal`. En su lugar, la opción `-R` generaba un archivo llamado `recovery.conf` que contenía toda la configuración de recuperación. Si estás migrando a la versión 12 o superior, esta transición de parámetros se gestiona de forma automática al iniciar el servidor.

*   **Impacto en el Rendimiento:** Utilizar la opción `--checkpoint=fast` te permite iniciar el proceso de *backup* de manera casi inmediata, pero genera un pico drástico de Entrada/Salida (E/S o I/O) en el nodo primario debido al volcado de datos. Se sugiere planificar estas operaciones en horarios de baja actividad comercial.

*   **Compresión en el flujo (Stream):** Cuando utilizas el método de *streaming* de WAL, debes saber que `pg_basebackup` no comprime el flujo de estos segmentos por sí mismo (a diferencia de la copia base de archivos, que sí admite compresión si eliges el formato `tar` comprimido). Si necesitas reducir el consumo de ancho de banda en el envío de los WAL, deberás habilitar `sslcompression` en la conexión o canalizar el tráfico a través de un túnel SSH dedicado.

---

# Resumen de comando utiles
```
# Backup completo para crear un standby
pg_basebackup -h primario -U replicador -D /ruta/standby -X stream -R -P -v --checkpoint=fast

# Con slot permanente
pg_basebackup -h primario -U replicador -D /ruta/standby -X stream -R -P -v --create-slot -S mi_slot

# Backup tar comprimido
pg_basebackup -h primario -U replicador -D /backup -F t -z -P

# Probar conexión (sin copiar todo)
pg_basebackup -h primario -U replicador -D /dev/shm/test -R -X stream -v --max-rate=1M --no-slot
```