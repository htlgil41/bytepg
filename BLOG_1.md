# 📘Blog_1📘

En esta documentacion estaremos explicando de alguna mejor forma la **Publication/Subscription** en PostgreSQL.

La replicación lógica de PostgreSQL permite replicar datos de forma selectiva entre bases de datos en tiempo real. A diferencia de la replicación física, que copia todo el clúster, la publicación/suscripción ofrece un control preciso sobre qué tablas se replican y dónde.

Los escenarios mas reales a nivel productivo son: Leer replicas de informes, Migracion de de bases de datos, Despliegues mutirregional, Intercambio selectivo de datos y por ultimo la Actualizacion de versiones. Prodrian existir mas casos al respecto pero nacen de una nececidad corporativa o una idea de proyecto pero en cuestion estods pueden ser un gran ejemplo.

## 1Prerrequisitos

Para comenzar debemos modificar parte del archivo de configuracion de PostgreSQL **`postgresql.conf`** en ambos lados tanto para el publicador como para el subscriptor.

**`postgresql.conf:`** 
```
    wal_level = logical

    max_replication_slots = 10
    max_wal_senders = 10
```
Para ver los cambios debemos reinciar el servidor o servicio de PostgreSQL. Ahora veamos que significa esta configuracion:

📍 wal_level: Esta determina la cantidad de información (y detalle) que el motor escribe en los Write-Ahead Logs (WAL). Write-Ahead Logs (WAL). Esto define el nivel de recuperación ante fallos y habilita características avanzadas como respaldos continuos o replicación

Esta acepta valores (minimal, replica, logical) que para la `minimal` es el nivel predeterminado original y el mas rapido. Para la `replica` esta agrega la informacion necesaria para el archivo WAL y para habilitar servidores que esperan (Standby) o replicas fisicas en tiempo real. Por otro lado la `logical` es el mas completo ya que este incluye toda la informacion de replica y agrega los datos necesarios para extrael conjuntos de cambios logicos.

📍 max_replication_slots: Esta define el numero maximo de ranuras (Slots -> Objeto de PostgreSQL) de replicacion que pueden existir simultaneamente en un servidor. Estas reanuras garantizan que el nodo primario conserve los registros de transacciones WAL hasta que sean consumidos y confirmados por todas las replicas o suscriptores. (En resumen sirve para proteccion del WAL, Replicacion continua y Replicacion logica)

El valor de `max_replication_slots` debe ser igual a la suma de todos los consumidores de datos simultáneos, incluyendo un margen para el crecimiento futuro o procesos paralelos. La fórmula recomendada es: `max_replication_slots = Replicas fisicas + suscriptores logicos + Reservas de contingencia`

Existen parametros que debemos considerar la respecto del uso o declaracion de esta configuracion:

`wal_level = replica` para replicacion fisica o `wal_level = logical` obligatorio para replicacion logica.

`max_wal_senders` (Numero maximo de conexiones de red simultaneas que el servidor primario puede abrir para enviar datos) debe configurarse como minimo, con el miso valor que `max_replication_slots` mas el numero de replicas fisicas que se conecter al mismo tiempo.

Y para evitar desbordamiento  de disto a causa del WAL lleno porque el slot lo mantiene bloqueado por tener confirmaciones debemos definir que tanta cantidad de informacion debe guardar para mantener la integridad de esta misma `max_slot_wal_keep_size = 100GB`

**Nota_**`Herramientas de respaldo (pgBackRest o Barman)`

## 2Usuario de replicacion
Desde el servidor primario o instancia primaria de PostgreSQL el usuario que se usara para esta configuracion debe tener permisos de replicacion: `CREATE USER replication_user WITH REPLICATION PASSWORD 'secure_password_here';`

## 3Configurar acceso
Permitir que los subscriptores se conecten editando el archivo `pg_hba.conf:`
```
# For replication connections
host    replication    replication_user    10.0.0.50/32    scram-sha-256
```
Para ver los cambios debemos reinciar el servidor o servicio de PostgreSQL.

## 4Preparar la BD de suscriptores

El subscriptor necesita estructuras de datos coincidentes es decir (el mismo schema de la principal p primaria). Mas simple, Creamos la tablas que tiene el primario fin.

## 5Suscripcion

En la base de datos de suscriptores, creamos una suscripción que apunte al editor: `CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replication_user password=secure_password_here'
    PUBLICATION my_publication;` 
La suscripcion realiza inmediatamente una sincronizacion de datos iniciales y luego cambia a cambios de transmision.

este podria ser un ejemplo de las opciones de la suscripcion al crearla:
```
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host port=5432 dbname=mydb user=replication_user password=secure_password_here'
    PUBLICATION my_publication
    WITH (
        copy_data = true,           -- Initial data copy (default: true)
        create_slot = true,         -- Create replication slot (default: true)
        enabled = true,             -- Start replication immediately (default: true)
        synchronous_commit = off,   -- Async for better performance
        binary = true               -- Use binary format (PostgreSQL 14+)
    );
```

Hasta este punto no es mas que cambiar valores estaticos sobre una configuracion en este caso de PostgreSQL y tratar de mantener una estructura en ambos casos a nivel de esquema referente a las bases de datos ya sea master o secundaria es decir uns subscritor. Ahora vamos a pasar a un tema sobre gestiones y codigo SQL->PostgreSQL para mantener y visualizar estas subscripciones y publicadores.

# 🚧 Gestion de Publicacion y suscripcion

Ok, vamos a esto de una, para ver las subscripciones existentes ejecutamos la sentencia `SELECT * FROM pg_publication` y para ver que tablas estan siendo publicadas `SELECT * FROM pg_publication_tables;`.

Ahora para las subscripciones existentes ejecutamos `SELECT * FROM pg_subscription` y para ver el status y demas datos ya sea por seleccion de columna usamo ``
SELECT
    subname,
    received_lsn,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;
``

Vamos ahora a como agregar una tabla a una publicacion existente con `ALTER PUBLICATION my_publication ADD TABLE new_table;` DESPUES DE AGREGAR TABLAS A LAS SUBSCRIPCIONES DEBEMOS ACTUALIZAR DICHA SUSCRIPCION `ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;`

Supongamos que un dia decimos "Quiero eliminar esta tabla de X suscripcion" pues listo ejecuta `ALTER PUBLICATION my_publication DROP TABLE old_table;`

Para desabilitar y reanudar la replicacion `ALTER SUBSCRIPTION my_subscription DISABLE;` -> `ALTER SUBSCRIPTION my_subscription ENABLE;`

Eliminar publicacion y suscripcion `DROP SUBSCRIPTION my_subscription;` -> `DROP PUBLICATION my_publication;`

# 🚥 Monitoreo del retraso en la replicacion

El retraso en la replicacion indica que tan atrasado esta el suscriptor por lo que se puede vigilar, ahora para veriricar el estado del slot ``
SELECT
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_size
FROM pg_replication_slots;
`` y para verificar el estado de la suscripcion ``
SELECT
    subname,
    pid,
    received_lsn,
    latest_end_lsn,
    latest_end_time,
    pg_size_pretty(pg_wal_lsn_diff(latest_end_lsn, received_lsn)) AS pending
FROM pg_stat_subscription;
``

# 🚨 Manejo de conflictos

A diferencia de las replicas fisicas, la replicacion logica puede encontrar conflictos cuando el suscriptor tiene datos contradictorios. 

Cuando existen conflictos la replicacion se detiene por lo que hay que validar los registros del suscriptor y resolverlos manualmente por ejemplo: Se dispara un dato con un id que se repite en la repliac que recibe esa nueva informacion.

# ✅ Mejores practicas

📍 Utilice usuarios de replicación dedicados con privilegios mínimos requeridos.

📍 Monitorizar el retraso en la replicación y alerta cuando supere los umbrales.

📍 Probar procedimientos de conmutación por error antes de que los necesites.

📍 Mantenga los esquemas sincronizados entre editor y suscriptor.

📍 Utilice filtros de fila/columna para minimizar el tráfico de replicación.

📍 Limpie las ranuras de replicación no utilizadas para evitar la acumulación de WAL.

📍 Documente su topología de replicación para mayor claridad operativa.

# 👾 Soluciones de problemas comunes

📍 **Problema**: La suscripcion no recibe datos. **Causa**: Ranura de replicacion inactiva. **Solucion**: Check `pg_replication_slots` y conectividad de red.

📍 **Problema**: Acumulación de WAL en el editor. **Causa**: Suscriptor rezagado o desconectado. **Solucion**: Elimina espacios no utilizados o arregla el suscriptor.

📍 **Problema**: Permiso denegado. **Causa**: Subvenciones faltantes. **Solucion**: Otorgar privilegios de SELECCIÓN y REPLICACIÓN.

📍 **Problema**: Permiso denegado. **Causa**: Subvenciones faltantes. **Solucion**: Otorgar privilegios de SELECCIÓN y REPLICACIÓN.

📍 **Problema**: La tabla no existe. **Causa**: Desajuste de esquemas. **Solucion**: Crear una estructura de tabla coincidente en el suscriptor.

📍 **Problema**: Error de conflicto. **Causa**: Clave duplicada. **Solucion**: Eliminar fila conflictiva u omitir transacción.

Y pues es esta la documentacion que lei en un blog, pienso que es mas a una introduccion que a un ejemplo claro de como se deben de realizar las cosa pero es interesante los conceptos que manejan porque son la base 🙄.