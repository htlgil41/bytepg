# 🤖Chat_1🤖

Ok vamos con todo y la maldita IA que solo hace mas estupidos a los programadores pero esta vez es nuestra mayor fuente de investigacion.

Esta documentacion nacio de una simple pregunta al a la IA y fue sencilla literlamente le pase **que es un failover en bases de datos?**. 

Y pues en bases de datos, un failover (o conmutación por error) es el proceso automático que transfiere el control de una base de datos principal a una de respaldo (secundaria) cuando la primera sufre una falla, garantizando la continuidad del servicio sin que los usuarios finales lo noten. 

Y de alli nacio la cuestion de **Publication/Subscription** en PostgreSQL en conjunto con el sentimiento de vacio de ser un **"Programador promedio"**

Ahora vamos a la respuesta exacta de dicha pregunta y las demas relacionada a todas las respuestas entre todo el chat y estudio por esta parte:

## 💡¿Que es failover?

### ¿Cómo funciona?

- **Monitore constante**: El sistema verifica continuamente el estado del servidor principal.
- **Deteccion de fallas**: Si el servidor principal deja de responder (Por problemas de hardware o red), se actica la alerta.
- **Conmutacion**: El servidor secundario, que contiene una copia de los datos en tiempo real se promueve automaticamente como el nuevo servidor principal.
- **Redireccion**: El trafico de las aplicaciones se redirige al nuevo servidor para mantener la base de datos operativa.

### Ventajas principales

- **Alta disponibilidad**: Evita la caida de los servicios y minimiza el tiempo de inactividad.
- **Proteccion de datos**: Evita la perdida de informacion gracias a la sincronizacion constante entre servidores.

### Conceptos clave relacionados
- **Failback (Recuperacion)**: Es poco proceso similar al failover, pero se realiza de forma manual (por mantenimiento o actualizacion) en lugar de ser activo por un fallo imprevisto.

## 💡¿Ejemplos de failover, te refieres transportar el control a que? es decir de que?

Cuando se habla de "transportar el control", significa cambiar qué computadora (servidor) se encarga de recibir, guardar y procesar los datos de tus aplicaciones.

En un sistema normal, tienes un Servidor A (Principal) que trabaja activamente y un Servidor B (Respaldo) que solo copia los datos en silencio. Si el Servidor A se rompe, el control pasa al Servidor B, que se "despierta" y toma el puesto del A.

📍Carrito de compras de una tienda en linea:

- **El escenario**: Estás comprando ropa en una página web. El Servidor Principal de la base de datos registra tu usuario y los artículos de tu carrito.
- **La falla**: A mitad de tu compra, el centro de datos del Servidor Principal sufre un apagón físico.
- **El Failover**: En milisegundos, el sistema detecta el apagón y pasa el control al Servidor de Respaldo (ubicado en otra ciudad).
- **El resutlado**: Tu compra no se cancela, tu carrito sigue intacto y ni siquiera notas que la base de datos que te atiende ahora es una máquina totalmente distinta.

📍Una aplicacion de banco (Transacciones financieras):

- **El escenario**: Una aplicación bancaria procesa miles de transferencias por segundo en su base de datos principal.
- **La falla**: El disco duro del servidor principal se llena por completo y bloquea el sistema.
- **El Failover**: El software de monitoreo corta la conexión con el servidor bloqueado y activa el Servidor Secundario, que tiene una copia idéntica de los saldos de las cuentas.
- **El resutlado**: El banco no duplica los cobros ni pierde registros de dinero, porque el segundo servidor retoma el trabajo exactamente en el último milisegundo antes del fallo.

## 💡Estoy utilizando postgres y estoy aprendiendo acerca de las subscripciones en en posgtres entre bases de datos y hay temas como estos o bases primarias nodos primarios y mas

¡Qué gran paso! Estás entrando en el terreno de la replicación lógica de PostgreSQL, un sistema potente para mover datos entre diferentes servidores mediante el modelo de Publicaciones y Suscripciones. Cuando conectas varias bases de datos en PostgreSQL, el vocabulario cambia un poco. Aquí te explico de forma sencilla qué es cada pieza de este rompecabezas:

📍Los roles en la suscripcion de PostgreSQL:
- **Nodo primario (Publicador)**: Es la base de datos de origen. Es la dueña original de los datos donde tus usuarios hacen cambios (INSERT, UPDATE, DELETE). Este nodo crea una Publicación para definir qué tablas quiere compartir.
- **Nodo secundario (Suscriptor)**: Es la base de datos de destino. Se conecta al nodo primario y crea una Suscripción para recibir esos cambios automáticamente en tiempo real.

📍¿Como se aplica el Failopver en este escenario?
En un entorno de producción, confiar en un solo Nodo Primario es peligroso. Si el Primario muere, tus Suscriptores se quedan sin actualizaciones. Aquí es donde el Failover salva el día:

``
[ Nodo Primario Activo ] ---- (Envía datos) ----> [ Nodo Suscriptor ]
         │ (Falla el Primario)
         ▼
[ Nodo Réplica Física ] -- (Se convierte en Primario) -- (Retoma el envío) --►
``

Ejemplos practicos de la suscripcion:

- Tienes una base de datos central en Bogotá (Nodo Primario) y bases de datos locales en Medellín y Cali (Nodos Suscriptores).
- Medellín y Cali se suscriben a Bogotá para recibir la lista actualizada de precios y productos.
- Para evitar que el sistema caiga, creas una Réplica Física de Respaldo de la base de datos de Bogotá en la nube de Amazon Web Services (AWS) o Azure.
- **Ocurre el fallo**: Un corte de energía desconecta el servidor físico de Bogotá.
- **Se ejecuta el Failover**: El sistema de respaldo en la nube detecta la caída y se transforma automáticamente en el nuevo Nodo Primario.
- **La magia del Failover**: Las suscripciones de Medellín y Cali se redirigen (usando herramientas como PgBouncer o balanceadores de carga) para conectarse al nuevo Primario en la nube. Los datos siguen fluyendo sin perder información.

📍Conceptos claves a encontrar en PostgreSQL
Para que no te pierdas en la documentación oficial de PostgreSQL Logical Replication, ten en mente estos tres términos:

- **Slot de Replicación (Replication Slot)**: Es un identificador en el Nodo Primario que asegura que este no borre los archivos de historial (WAL) hasta que el Suscriptor los haya recibido de forma segura. Si el Suscriptor se apaga por dos horas, el slot guarda la posición para retomar el trabajo exactamente donde quedó.
- **Replica fisica vs Replica logica**: La Física duplica byte por byte todo el servidor (se usa para Failover puro). La Lógica (Suscripciones) te permite elegir selectivamente qué tablas enviar a otros nodos, ideal para analítica de datos o conectar sistemas distintos.
- **Conflictos de Replicación**: Ocurren si intentas modificar manualmente en el Suscriptor una tabla que viene del Primario. Si modificas la misma fila en ambos lados, la suscripción se detendrá por seguridad hasta que resuelvas cuál dato es el correcto.

# Hagamos un ejemplo en un entorno local

Vamos a levantar dos instancias de PostgreSQL en cualquier maquina/servidor.

En el servidor primario es decir en una de las instancias creadas configuramos `wal_level = logical` en `postgresql.conf`

Luego creamos una publicacion en el servidor 1(Servidor primario) a una o mas tablas segun la seleciconada a replicar para luego crear la suscripcion en la segunda tabla.

**NOTA_**Este ejemplo de sencillo podria ser menos efectivo o podria esta de la mano con el de la investigacion del blog 1 (**[BLOG_1.md](https://github.com/htlgil41/bytepg/blob/main/BLOG_1.md)**)

# Vamos con algunos datos proporcionados por la investigacion dentro del chat

📍 ¿Cómo se sincronizan y saben dónde quedaron tras el Failover?

Cuando la DB primaria se cae, las sucursales locales quedan "huérfanas". Al conectarse al nuevo servidor en la nube, el misterio es: ¿Cómo sabe el nuevo servidor qué datos enviarle a cada sucursal si la Db primaria ya no existe para contarlo?. El secreto se divide en dos partes: el Lado del Suscriptor y el Lado del Nuevo Primario.

- **A. El lado del suscriptor**: Cada base de datos local lleva su propio registro interno. No depende de que Bogotá se lo recuerde.

Postgres usa un número de secuencia llamado LSN (Log Sequence Number). Es como un número de página de un libro.

Si Medellín (suscripto logico) quedó en el cambio LSN: 0/19A2340, ese número está guardado en el disco duro de Medellín (suscripto logico).

Cuando ocurre el Failover y Medellín (suscripto logico) se conecta al servidor de la nube, lo primero que hace es saludar y decir: "Hola, soy Medellín (suscripto logico). Mi último cambio registrado es el LSN 0/19A2340. Dame lo que siga".

- **B. El Lado del Nuevo Primario (El Respaldo promovido)**: Aquí viene el verdadero truco. Para que el servidor de la nube pueda responderle a Medellín (suscripto logico), necesita tener guardado el Slot de Replicación que la Db Primaria anterior usaba para Medellín. El Slot de Replicación es como el separador de hojas del libro que el Primario mantiene por cada suscriptor para saber qué datos no debe borrar del historial.

- Antes de la caída: Bogotá actualizaba constantemente los Slots de Replicación de las sucursales.
- La sincronización de Slots: El servidor de la nube (mientras era físico) clonaba esos Slots en tiempo real desde Bogotá (en PostgreSQL moderno esto se configura con el parámetro failover = true en las ranuras de replicación).
- La sincronización de Slots: El servidor de la nube (mientras era físico) clonaba esos Slots en tiempo real desde Bogotá (en PostgreSQL moderno esto se configura con el parámetro failover = true en las ranuras de replicación).
- Después del Failover: Como el servidor de la nube ya tenía guardada la posición de los Slots, cuando Medellín llega preguntando por el LSN 0/19A2340, la nube busca en su historial, encuentra la posición exacta y le empieza a transmitir los datos lógicos desde el punto 0/19A2341 en adelante.

Gracias a este apretón de manos matemático (LSN del suscriptor + Slot sincronizado en el primario), no se pierde ni se duplica una sola transacción durante la conmutación por error

📍El problema de la "Asincronía" (La velocidad de la red)

Hay un pequeño detalle técnico en el funcionamiento interno de Postgres que debes tener en cuenta, y es la razón por la cual los ingenieros de bases de datos se preocupan tanto por esto durante la reconexión.

Aunque la réplica física copia todo, la transmisión a través de la red toma milisegundos. Esto genera un riesgo en el momento exacto del daño:
- **La fila de espera**: Bogotá (Db primaria) escribe un cambio en su disco duro. Inmediatamente se lo envía a Medellín (Lógico) y al Respaldo en la nube (Físico).
- **El desfase**: Por velocidad de red, puede pasar que Medellín (suscriptor logico) reciba el cambio un milisegundo antes de que el Respaldo en la nube llegue a guardarlo en su propio disco duro.

Si el Respaldo en la nube se promueve a Primario, técnicamente le falta el último milisegundo de datos que Bogotá no alcanzó a enviarle antes de morir. Pero Medellín (el suscriptor lógico) sí alcanzó a recibirlo directamente de Bogotá.

📍¿Qué pasa en la reconexión?

Cuando Medellín (suscriptor logico) intente conectarse al nuevo Primario en la nube, le dirá: "Hola, mi último cambio es el LSN 100". Pero el nuevo Primario revisará su disco y dirá: "Espera... mi propio historial físico solo llegó hasta el LSN 99 porque Bogotá murió antes de mandarme el 100".

A esto se le conoce como un conflicto de línea de tiempo (Timeline conflict). Como el suscriptor lógico va "más adelante" en el futuro que el nuevo primario, la replicación se detiene y arroja un error porque los historiales no encajan perfectamente.Para evitar este dolor de cabeza en la reconexión, se usan dos soluciones:
- **Replicación Física Síncrona**: Configuras a Bogotá para que no le confirme a ninguna aplicación que un dato fue guardado hasta que el Respaldo en la nube confirme que también recibió el byte. Esto asegura que la nube NUNCA se quede atrás de nadie, pero vuelve el sistema un poco más lento.
- **Herramientas de Failover (como Patroni o repmgr)**: Estos programas se aseguran de revisar los historiales de todos los nodos antes de decidir la reconexión y ajustan los punteros automáticamente.


💡Entonces como se programa esto? es decir la subscripcoin o replica fisica y la logica con esto de que a las logicas no se le envia cambios hasta que en la fisica confirme que tiene hasta el ultimo byte

Para lograr que las bases de datos locales (Suscripciones Lógicas) no reciban ningún dato hasta que el respaldo en la nube (Réplica Física) confirme que tiene guardado hasta el último byte, debes configurar Replicación Síncrona en PostgreSQL.

Esto se programa modificando los archivos de configuración nativos de Postgres `postgresql.conf` y ejecutando sentencias SQL específicas.

- **Paso 1: Configurar la Réplica Física Síncrona**: 
Primero debemos asegurar el canal físico entre Bogotá y la Nube. El Servidor Principal no dará por terminada ninguna transacción hasta que la nube responda "ya lo guardé".
En el servidor de Bogota (Primario). Debes activar el nivel lógico (necesario para convivir con las sucursales) y definir el nombre de la réplica física que será obligatoria:

```
# Habilita el historial detallado para ambos tipos de replicación
wal_level = logical

# Define cuántos servidores físicos deben confirmar el dato. 
# 'FIRST 1' significa que el primero de la lista que responda bloquea el commit exitoso.
synchronous_commit = on
synchronous_standby_names = 'FIRST 1 (replica_nube)'

# CRUCIAL: Bloquea los slots lógicos para que no avancen más allá de este slot físico
synchronized_standby_slots = 'slot_fisico_nube'
```

En el servidor de respaldo (nube). Cuando configures la réplica física (usando la herramienta nativa pg_basebackup), asegúrate de que el parámetro de conexión identifique al servidor con el nombre exacto que pusiste en Bogotá: `pg_basebackup -h IP_BOGOTA -D /var/lib/postgresql/data -U replicador -P -R --application-name=replica_nube
` (Comando para clonar la base de datos físicamente por primera vez)

- **Paso 2: Crear el Slot de Replicación con Failover para las Locales**:
Para que los Slots de Replicación de tus sucursales lógicas se copien físicamente a la nube y estén listos en caso de una caída, debes crearlos en Bogotá indicando explícitamente que soportan la conmutación por error (disponible a partir de PostgreSQL 17 17 Documentation).
```
-- Creamos un slot lógico para Medellín que sea compatible con Failover físico
SELECT pg_create_logical_replication_slot(
    'slot_medellin', 
    'pgoutput', 
    false, 
    true -- Este último parámetro activa el 'failover = true'
);

-- Creamos la publicación normal
CREATE PUBLICATION pub_bogota FOR ALL TABLES;

```

- **Paso 3: Configurar las Sucursales Locales (Lógicas)**: Ahora configuras las bases de datos de tus ciudades. Al conectarse a Bogotá, deben usar obligatoriamente el Slot que acabas de preparar.

```
CREATE SUBSCRIPTION sub_medellin
CONNECTION 'host=IP_BOGOTA port=5432 user=postgres password=clave dbname=bogota_db'
PUBLICATION pub_bogota
WITH (
    create_slot = false,           -- No crees un slot nuevo
    slot_name = 'slot_medellin'     -- Usa el slot síncronizado que preparamos
);
```
Cuando pones ese último parámetro en true (que activa la propiedad de failover), le estás ordenando a la base de datos de Bogotá (Primario): «Por favor, permite que este slot lógico sea copiado byte a byte por la réplica física de la nube».


📍¿Cómo fluyen los datos ahora con esta programación?
```
[Cliente] -> Envía INSERT a Bogotá (Primario)
                │
                ▼
[Bogotá (Primario)]  -> Escribe el cambio en su disco duro local (WAL)
                │
                ├──> ENVIAR A LA NUBE (Físico) ──> [Nube] Guarda en disco y responde "OK"
                │                                    │
                ▼ (Solo después del OK de la nube)   ▼
[Bogotá (Primario)]  -> Hace el "Commit" de la transacción (El dato ya es real y visible)
                │
                ▼
[Bogotá (Primario)]  -> Libera el dato para el Slot Lógico de Medellín
                │
                ▼
[Medellín (Suscriptor)]-> Recibe el cambio lógico
```

Si Bogotá explota exactamente después de escribir el cambio en su propio disco pero antes de que la nube responda, la transacción se cancela (se hace Rollback). 

Medellín nunca llegará a ver ese dato.Cuando la Nube tome el control tras el Failover, su base de datos estará limpia, los Slots de replicación locales estarán en perfecta sincronía posicional y las sucursales se reconectarán sin toparse jamás con un conflicto de historial.

Para implementar esto de forma robusta y automatizar el cambio de IP de las sucursales, te recomiendo revisar la documentación de Patroni, que es la herramienta estándar de la industria para gestionar este flujo en PostgreSQL. (El slot de primario que se creo se relaciona a la suscripcion del las replicas ya sea logica o fisica)

**SUPER NOTA POR SI A ESTE PUNTO NO SE ENTIENDE -> EN LAS SUSCRIPCIONES SE MARCA EL FILOVER  PARA QUE REPLIQUE EN EL RESPALDO FISICO** es decir la replica fisica tendria los slots habilitados cuando este pase a ser primario.


# 💡Entiendo pero igualmente me pregunto porque si la replica en la nube debe recibir byte a byte es decir un slot fisico pero se creo logico

¡Ah, excelente! Ya veo exactamente dónde está la confusión. Estás tocando el punto más profundo de cómo cooperan estos dos mecanismos por dentro.

Tu duda es totalmente lógica, si la nube es una réplica física y recibe bytes, ¿por qué el comando para crear el slot de Medellín es un comando lógico (`pg_create_logical_replication_slot`). La respuesta es que ese comando no configura la conexión de la nube. 

Ese comando es exclusivo para Medellín. Lo que pasa es que PostgreSQL guarda la existencia y el progreso de ese slot lógico dentro de los mismos archivos de datos del sistema.

El Slot Físico para la Nube. (Este se crea tras bambalinas o con pg_create_physical_replication_slot). Este slot le dice a Bogotá: "Mantén un canal de bytes abierto para el servidor de la nube".

El Slot Lógico para Medellín. (El comando que seleccionaste). Este slot le dice a Bogotá: "Mantén un traductor SQL abierto para la base de datos de Medellín".

**`(Peronalmente a este punto creo que esta Doc es quizas entendible unicamente para mi "Lo siento")`**

# 🧩Vamos con todo el paso

📍**Paso 1 Configurar el servidor principal**

Este servidor es el que manda al principio. Debe estar preparado para enviar datos binarios (a la Nube) y datos traducidos (a Medellín) al mismo tiempo.

```
# Cambiar el nivel de historial a lógico (sirve para físico y lógico a la vez)
wal_level = logical

# Activar la replicación síncrona obligatoria para la Nube
synchronous_commit = on
synchronous_standby_names = 'FIRST 1 (replica_nube)'

# Permitir que los slots lógicos se puedan copiar físicamente a la nube (Esta regla es concepto de investigacion)
replication_slots_development_mode = true # (Nota: En PG17+ esto habilita el failover de slots)
```

Creamos los Slots y publiaciones en el primario:
```
-- 1. Creamos la tabla real de nuestro negocio
CREATE TABLE productos (id SERIAL PRIMARY KEY, nombre TEXT, precio NUMERIC);

-- 2. Creamos la PUBLICACIÓN lógica para las sucursales
CREATE PUBLICATION pub_bogota FOR TABLE productos;

-- 3. Creamos el SLOT LÓGICO para Medellín con la etiqueta 'failover = true'
SELECT pg_create_logical_replication_slot(
    'slot_medellin', 
    'pgoutput', 
    false, 
    true -- <-- Aquí le ordenamos empacarse en el viaje físico hacia la nube
);

-- 4. Creamos el SLOT FÍSICO que usará la Nube para no perderse ningún byte
SELECT pg_create_physical_replication_slot('slot_fisico_nube');
```

Ahora vamos a configurar l respaldo (Alternativa como primario)

Este servidor empieza completamente vacío porque clonará el disco duro de Bogotá byte a byte. No tienes que crear tablas ni bases de datos aquí; Postgres lo hará por ti.

-  **Clonar el servidor de Bogotá por primera vez (Línea de comandos)**:Con el servidor del puerto 5433 apagado, ejecuta este comando en tu terminal para traer una copia idéntica del disco duro de Bogotá: `pg_basebackup -h 127.0.0.1 -p 5432 -U postgres -D /ruta/de/tu/carpeta_puerto_5433 -Fp -Xs -P -R --slot=slot_fisico_nube --application-name=replica_nube
` (Para mas informacion de `pg_basebackup` se documentara)

Ahora vamos con las replicas logicas que basicamente seria el mismo proceso e incluso los paso en **[CHAT_1.md](https://github.com/htlgil41/bytepg/blob/main/CHAT_1.md)**