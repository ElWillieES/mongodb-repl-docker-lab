# MongoDB Replication Docker Lab

![Docker](https://img.shields.io/badge/Docker-2496ED?&style=flat&logo=docker&logoColor=ffffff)&nbsp;
![MongoDB](https://img.shields.io/badge/MongoDB-4EA94B?style=flat&logo=mongodb&logoColor=white)

Más ejemplos de Badges en [markdown-badges](https://ileriayo.github.io/markdown-badges/), [Badges 4 README.md Profile](https://github.com/alexandresanlim/Badges4-README.md-Profile) y [Awesome Badges](https://github.com/Envoy-VC/awesome-badges)

## Introducción

La replicación asíncrona de MongoDB permite que nuestros datos sean más durables y proporciona alta disponibilidad (HA). Para este fin, MongoDB utiliza Nodos adicionales que mantienen una copia de los datos, que se sincroniza de forma asíncrona.

Una aplicación que usa un Replica Set de MongoDB como base de datos, tiene varias ventajas en pro de una alta disponibilidad:

* Desde el lado de la aplicación, es posible y sencillo conectarse a los Secundarios o a los Primarios, según la necesidad que tengamos (ej: lecturas o escrituras), descargando al Primario todo lo posible, incluso podemos hacer escrituras y que nos devuelva el OK cuando han sido replicadas al menos a un Secundario o a la mayoría de los Secundarios, etc. Este tipo de técnicas (ej: Write Concers, Read Concers, Read Preferences) son muy útiles en aplicaciones operacionales.
* En caso de Failover, la aplicación es capaz de reconectar con los nuevos Primarios y/o Secundarios, de forma automática, ya sea por un crash, o por una operación de mantenimiento planificada.

**Este repo se ha creado para compartir un laboratorio de Replicación de MongoDB utilizando Docker Compose**, de forma que sea posible arrancar y probar de forma rápida un Replica Set. Dicho Docker Compose:

* Define una docker-net.
* Define un contenedor **mongo000** que usaremos como cliente, para pruebas, con una IP fija.
* Define tres contenedores (**mongo001**, **mongo002**, **mongo003**) que seran los Nodos (Primario y Secundarios) de nuestro Replica Set, con las siguientes peculiaridades:
  * Tienen un volumen persistente, para los datos.
  * Tienen una IP fija.
  * MongoDB escucha en el puerto por defecto (tcp-27017), pero en el Docker Compose lo mapeamos al exterior en puertos distintos, ya que el mismo puerto no lo podríamos mapear a los tres Mongos.
  * Pasamos como variables de entorno del contenedor, el usuario y contraseña, para el usuario root de MongoDB.
  * Mapeamos el fichero del Key File utilizado para la autenticación y comunicación entre los Mongos.
  * Personalizamos el entrypoint para ajustar los permisos del Key File (es requisito, debe tener un 400)
  * Personalizamos el comando de inicio de MongoDB, indicando el nombre del Replica Set (todos los Nodos que forman parte del mismo Replica Set, deben indicar aquí el mismo valor), el fichero de KeyFile, e indicando que escuche en todas las IPs.

Para la autenticación y comunicación entre los Nodos de un Replica Set, es necesario crear el fichero Key File, que mapearemos en el Docker File. En el repo compartimos uno, pero es posible crear uno nuevo (recomendable) con un comando como el siguiente.

```shell
openssl rand -base64 741 > keyfile
```

El Docker Compose utiliza un fichero de configuración .env, que define variables para:

* Time Zone
* Usuario y Password de MongoDB. Es recomendable ajustar el valor de la contraseña.

**Puedes apoyar mi trabajo siguiéndome, haciendo "☆ Star" en el repo, o nominarme a "GitHub Star"**. Muchas gracias :-) 

[![GitHub Star](https://img.shields.io/badge/GitHub-Nominar_a_star-yellow?style=for-the-badge&logo=github&logoColor=white&labelColor=101010)](https://stars.github.com/nominate/)

En mi Blog personal ([El Willie - The Geeks invaders](https://elwillie.es)) y en mi perfil de GitHub, encontrarás más información sobre mi, y sobre los contenidos de tecnología que comparto con la comunidad.

[![Web](https://img.shields.io/badge/GitHub-ElWillieES-14a1f0?style=for-the-badge&logo=github&logoColor=white&labelColor=101010)](https://github.com/ElWillieES)

# Configuración de un Replica Set con MongoDB

Lo primero, arrancar nuestro Docker Compose, comprobar el estado de nuestros contenedores (que estén up & running), y abrir una sesión bash contra el cliente mongo000.

```shell
docker compose up -d
docker ps
docker exec -it mongo000 bash
```

Desde la sesión bash de mongo000, nos conectamos con MongoDB Shell (mongosh) a Nodo mongo001, especificando el host, usuario, contraseña, y la base de datos a la que nos queremos conectar (admin).

```shell
mongosh --host mongo001 --username root --password password admin
```

A continuación iniciamos la replicación en el Nodo mongo001, añadimos los nodos mongo002 y mongo003, para tener un Replica Set con tres nodos, y comprobamos la topología de nuestra réplica. Todo esto, ejecutando los siguientes comandos.

```
rs.initiate()
rs.add("mongo002")
rs.add("mongo003")
rs.isMaster()
```

Si deseamos tener una información más detallada de la topología y de la saluda de los Nodos de nuestro Replica Set, podemos ejecutar el siguiente comando.

```
rs.status()
```

También es muy útil el comando rs.conf() o su alias rs.config(), que nos devuelven un documento con la configuración del Replica Set. Algunos detalles, sobre el documento que nos devuelve:

* El campo _id es el nombre de nuestro Replica Set.
* Los campos version y term se incrementan en cada cambio de configuración o de la topología.
* El array members contiene un subdocumento por cada Nodo del Replica Set, el cual, almacena su configuración (ej: el nombre de host, una prioridad entre 1 y 1000 para condicionar que Nodo debería ser elegido Primario más frecuentemente, si es árbitro en cuyo caso la prioridad debe ser 0, si es hidden en cuyo caso la prioridad debe ser 0, si tiene delay, si tiene derecho a voto, etc).

```
rs.config()
```

Otros comandos con los que podemos obtener información del estado de la replicación.

```
db.serverStatus()['repl']
rs.printReplicationInfo()
```

Nos salimos de la MongoDB Shell, para volver a conectarnos, pero esta vez, **en la conexión vamos a especificar como Host, el nombre del Replica Set y el nombre del Nodo**, de tal modo, que si el Nodo indicado es un Secundario, la MongoDB Shell sea capaz de **descubrir cuál es el Primario y conectarse a él**. Para probarlo, vamos a conectarnos al Nodo mongo002, que ahora es Secundario, y comprobamos la topología de la Réplica.

```shell
mongosh --host mongo-repl/mongo002 --username root --password password admin
```

Comprobamos la réplica.

```
rs.isMaster()
```

Lo siguiente que vamos a hacer es **forzar un failover**, y ver cómo nos reconectamos al nuevo primario automáticamente.

```
rs.stepDown()
rs.isMaster()
```

Por defecto los Secundarios no están disponibles para lecturas, como se demuestra al crer una conexión con un Secundario (se puede apreciar en la Shell que indica que estamos conectados a un Secundario) para ejecutar una consulta, obteniendo un mensaje de error.

```
rs.isMaster().ismaster
rs.isMaster().me
rs.isMaster().primary
show collections
db.sports.find()
```

**Para permitir las lecturas en el Secundario deberemos indicarlo explícitamente con un comando**. Esta es una configuración que podemos habilitar para la conexión actual, es decir, si nos desconectamos y nos volvemos a conectar después al mismo u otro Secundario, deberemos de utilizar de nuevo dicho comando para indicar explícitamente a MongoDB que deseamos poder leer del Secundario, y listo.

```
db.getMongo().setReadPref('secondary')
db.sports.findOne()
```

Otro detalle a tener en cuenta, es que si tenemos **un Replica Set con un Primario y dos Secundarios, si dos de los nodos se apagan, el único nodo restante, al no tener mayoría, seguirá levantado, pero sólo podrá actuar como Secundario y aceptar lecturas**, en ningún caso escrituras.