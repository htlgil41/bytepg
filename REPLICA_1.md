# 💻Replica step_1💻

Vamos con algunos pasos para lograr la replicacion de los ejemplos de la documentacion del primer chat **[CHAT_1.md](https://github.com/htlgil41/bytepg/blob/main/CHAT_1.md)**

Vamos con la configuracion del archivo `postgresql.conf` para el servidor principal que sera primario:

```
# Habilita el historial necesario para réplicas lógicas y físicas
wal_level = logical

# Activa la confirmación síncrona básica
synchronous_commit = on

# Nombre de la réplica física que debe confirmar antes de cerrar transacciones
synchronous_standby_names = 'FIRST 1 (replica_nube)'

# CRUCIAL: Bloquea los slots lógicos para que no avancen más allá de este slot físico
synchronized_standby_slots = 'slot_fisico_nube'
```

ahora vamos con el servidor de respaldo en la nube (candidato para ser primario)

```
# Cambia el puerto para evitar conflictos en tu localhost
port = 5433

# Requisito obligatorio para que las réplicas físicas acepten sincronizar slots lógicos
hot_standby_feedback = on

# CRUCIAL: Activa el trabajador interno que copia los slots del servidor primario
sync_replication_slots = on

# Datos para conectarse a Bogotá (Se configuran automáticamente con pg_basebackup)
primary_conninfo = 'host=127.0.0.1 port=5432 user=postgres password=tu_clave application_name=replica_nube'
primary_slot_name = 'slot_fisico_nube'
```

Creamos los Slots dentro del primario:
```
-- 1. Creamos el slot físico para la Nube
SELECT pg_create_physical_replication_slot('slot_fisico_nube');

-- 2. Creamos el slot lógico para Medellín con el failover habilitado
-- Parámetros: slot_name, plugin, temporary, wal_status, failover
SELECT pg_create_logical_replication_slot('slot_medellin', 'pgoutput', false, false, true);
```

Ahora biem tenemos ambas configuradas y los slots estan creados y verificamos la sincronizacion de slots funcione:

```
SELECT slot_name, failover, synced, active 
FROM pg_replication_slots;
```

En el servodor de replica fisica se debe instalar PostgreSQL mas no crear ningun schema ni nada porque esta seria igual byteXbyte para iniciar la replica fisica y asi ambas db estar sincronizadas.

**Sitaxis basica:** `pg_basebackup -h [IP_DEL_PRIMARIO] -p [PUERTO] -U [USUARIO] -D /ruta/donde/guardar/la/copia -Fp -P
`

**Opciones basicas**:
- **-h y -p**: La dirección IP y el puerto del servidor que vas a copiar.
- **-U**: El usuario con permisos de replicación (usualmente postgres).
- **-D**: La carpeta de tu computadora donde quieres que se cree la base de datos clonada. Esta carpeta debe estar vacía o no existir.
- **-Fp**: Formato plano (Plain). Copia los archivos tal cual están en el disco duro original.
- **-P**: Muestra una barra de progreso en la terminal mientras se descargan los datos.

`pg_basebackup -h 127.0.0.1 -p 5432 -U postgres -D /tu/carpeta_puerto_5433 -Fp -P -R --slot=slot_fisico_nube --application-name=replica_nube --wal-method=stream
`

`pg_basebackup -h 127.0.0.1 -p 5432 -U postgres -D /tu/carpeta_puerto_5433 -Fp -P -R --slot=slot_fisico_nube --application-name=replica_nube --wal-method=stream
` (`Comando exacto de replicacion fisica para la nube`)

**-R (Oopción Crucial)**: Significa Write Recovery Conf. Crea automáticamente el archivo standby.signal (para que el clon sepa que es un esclavo de solo lectura) y escribe de forma automática las variables primary_conninfo y primary_slot_name al final del archivo postgresql.conf de la copia. Te ahorra hacer toda la configuración manual.

**--slot=slot_fisico_nube**: Le ordena a la copia amarrarse de inmediato al slot físico que creamos en Bogotá. Esto evita que Bogotá borre los archivos del historial (WAL) mientras la copia se está descargando.

**--application-name=replica_nube**: Le da un "nombre de pila" a la conexión de la nube. Así, cuando Bogotá revise su parámetro synchronous_standby_names = 'FIRST 1 (replica_nube)', reconocerá a este clon de inmediato y activará el bloqueo síncrono.

**--wal-method=stream**: Transmite el historial de transacciones en tiempo real en un canal paralelo mientras se realiza la copia general, asegurando que no se pierda un solo byte del disco duro.

En la configuracion una vez ejecutado el comando debemos colocar en el archivo:

```
sync_replication_slots = on
hot_standby_feedback = on
```

- **sync_replication_slots**: Evita que las consultas en la réplica se cancelen abruptamente. El proceso VACUUM del servidor principal borra filas muertas, pero si una réplica las necesita, la consulta falla. Esta propiedad avisa al principal qué transacción está activa en la réplica para que retenga esas filas.
- **hot_standby_feedback**: En escenarios de alta disponibilidad, asegura que si el servidor principal cae y una réplica es promovida como principal, los suscriptores (por ejemplo, herramientas de replicación lógica) no pierdan eventos ni generen datos duplicados tras el cambio. Sincroniza las ranuras lógicas del primario hacia el standby.

# 💡Entonces tu dices que esta base de datos es de solo lectura pero cuando se vuelve principal como se realiza el cambio? es decir porque deja de ser solo lectura

Cuando la base de datos de respaldo detecta la orden de promoción, realiza una metamorfosis interna inmediata: borra el archivo bandera standby.signal de su disco duro, cambia su línea de tiempo interna (Timeline) y se declara a sí misma como un nodo listo para recibir escrituras.

## ¿Cómo se realiza el cambio? (Los Comandos)

En tu laboratorio local, cuando decidas que el servidor principal de Bogotá (puerto 5432) "murió" (puedes apagarlo a la fuerza), debes ir a la terminal de tu computadora y ejecutar un solo comando apuntando a la carpeta de datos del servidor de respaldo (puerto 5433).

- **Opción A: Usando la herramienta nativa pg_ctl**: `pg_ctl promote -D /home/usuario/postgres_respaldo
`

# ¿Cómo automatizar esto en la vida real?

Hacer pg_ctl promote a mano es excelente para tu laboratorio de aprendizaje. Sin embargo, en un entorno de producción real en la nube, los ingenieros no esperan a que un humano se despierte a las 3:00 AM a escribir un comando.Se utilizan herramientas de automatización como Patroni. Patroni es un programa que corre al lado de Postgres, envía señales de vida ("Heartbeats") entre Bogotá y la Nube cada segundo y, si detecta que Bogotá no responde en un lapso de (por ejemplo) 10 segundos, él mismo ejecuta el comando de promoción automáticamente sin intervención humana.