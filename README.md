# Explicación de docker-compose.yml — Stack de monitoreo

## Parte 1 — Persistencia de datos

### 1. Servicio MySQL

MySQL es el servicio encargado de proporcionar la base de datos que utiliza el stack de monitoreo. En este caso, se ejecuta dentro de un contenedor Docker a partir de una imagen de MySQL.

La configuración del servicio en `docker-compose.yml` es:

```yaml
mysql:
  image: mysql:latest
  restart: always
  environment:
    MYSQL_ROOT_PASSWORD: mysecretpassword
  networks:
    - mynetwork
```

#### ¿Qué hace cada elemento?

| Elemento | Explicación |
|---|---|
| `mysql:` | Es el nombre que identifica al servicio dentro del archivo `docker-compose.yml`. |
| `image: mysql:latest` | Indica que Docker debe utilizar la imagen de MySQL como base para crear el contenedor donde se ejecutará el servidor MySQL. `latest` indica la versión marcada como más reciente de esa imagen. |
| `restart: always` | Indica que Docker debe intentar reiniciar automáticamente el contenedor si este se detiene. |
| `environment:` | Permite definir variables de entorno que serán utilizadas por el contenedor. |
| `MYSQL_ROOT_PASSWORD: mysecretpassword` | Establece la contraseña del usuario `root` de MySQL mediante una variable de entorno. |
| `networks:` | Indica a qué red o redes de Docker pertenece el servicio. |
| `- mynetwork` | Conecta el contenedor de MySQL a la red `mynetwork`, permitiendo que se comunique con los demás servicios conectados a esa red. |

#### Imagen y contenedor

La **imagen** de Docker funciona como una plantilla que contiene lo necesario para ejecutar un programa. El **contenedor** es una instancia creada a partir de esa imagen que se encuentra ejecutándose.

En este caso:

```text
Imagen mysql:latest
        ↓
   crea el contenedor
        ↓
   MySQL funcionando
```

#### ¿Por qué se utiliza una variable de entorno para la contraseña?

La variable `MYSQL_ROOT_PASSWORD` permite proporcionar la contraseña que MySQL necesita durante su configuración inicial.

En este `docker-compose.yml`, la contraseña se define mediante una variable de entorno para que Docker pueda entregársela al contenedor al momento de iniciarlo, sin tener que modificar la imagen de MySQL.

### 2. Servicio dbexporter

`dbexporter` es el servicio encargado de obtener métricas de MySQL y exponerlas en un formato que Prometheus pueda recolectar. Funciona como intermediario entre MySQL y Prometheus: consulta información de MySQL, obtiene sus métricas y las pone a disposición de Prometheus.

La configuración del servicio en `docker-compose.yml` es:

```yaml
dbexporter:
  image: prom/mysqld-exporter:latest
  restart: always
  volumes:
    - ./my.cnf:/.my.cnf:ro
  command:
    - --config.my-cnf=/.my.cnf
  depends_on:
    - mysql
  networks:
    - mynetwork
```

#### ¿Qué hace cada elemento?

| Elemento | Explicación |
|---|---|
| `dbexporter:` | Es el nombre que identifica al servicio dentro del archivo `docker-compose.yml`. |
| `image: prom/mysqld-exporter:latest` | Indica que Docker debe utilizar la imagen de `mysqld-exporter` para crear el contenedor donde se ejecutará el exporter. |
| `restart: always` | Indica que Docker debe intentar reiniciar automáticamente el contenedor si este se detiene. |
| `volumes:` | Permite montar archivos o directorios del equipo donde se ejecuta Docker dentro del contenedor. |
| `./my.cnf` | `./` indica el directorio actual, por lo que se utiliza el archivo `my.cnf` ubicado en la misma carpeta que el `docker-compose.yml`. |
| `/.my.cnf` | Es la ubicación donde el archivo `my.cnf` estará disponible dentro del contenedor. |
| `:ro` | Significa `read only` (solo lectura). Permite que el contenedor lea el archivo, pero no que lo modifique. |
| `command:` | Permite indicar un comando o parámetros adicionales que se ejecutarán al iniciar el contenedor. |
| `--config.my-cnf=/.my.cnf` | Indica al exporter que debe utilizar el archivo `/.my.cnf` como archivo de configuración para conectarse a MySQL. |
| `depends_on:` | Establece una dependencia entre servicios. |
| `- mysql` | Indica que `dbexporter` depende del servicio `mysql`, ya que necesita conectarse a MySQL para obtener sus métricas. |
| `networks:` | Indica a qué red de Docker pertenece el servicio. |
| `- mynetwork` | Conecta `dbexporter` a la red `mynetwork`, permitiendo que se comunique con los demás servicios conectados a esa red. |

#### Relación entre `my.cnf` y `dbexporter`

El archivo `my.cnf` contiene la información de configuración que necesita `dbexporter` para conectarse a MySQL.

El volumen:

```yaml
- ./my.cnf:/.my.cnf:ro
```

hace que el archivo que se encuentra junto al `docker-compose.yml` esté disponible dentro del contenedor.

El parámetro:

```yaml
--config.my-cnf=/.my.cnf
```

le indica al exporter dónde encontrar ese archivo.

De esta manera, `dbexporter` puede leer la configuración necesaria sin modificar el archivo original.

#### Relación entre `dbexporter` y MySQL

La configuración:

```yaml
depends_on:
  - mysql
```

indica que `dbexporter` depende del servicio `mysql`. Esto establece el orden de inicio de los servicios: primero se inicia `mysql` y luego `dbexporter`.

Esta relación tiene sentido porque `dbexporter` necesita conectarse a MySQL para obtener sus métricas.


## Parte 2 — Recolección de métricas (`cadvisor` y `prometheus`)

### Qué hace cada servicio

**`cadvisor`** mide el consumo de recursos (CPU, memoria, disco, red) de todos los
contenedores que corren en el host y lo expone como métricas en su puerto 8080.

**`prometheus`** es el recolector: cada 5 segundos (`scrape_interval: 5s` en
`prometheus.yml`) consulta a cada target, y guarda las métricas como series de
tiempo. Tiene cuatro jobs: él mismo, `cadvisor`, `mysql` (vía `dbexporter`) y
`crud-app`.

### Por qué cAdvisor monta rutas del host en solo lectura

```yaml
volumes:
  - /:/rootfs:ro
  - /var/run:/var/run:ro
  - /sys:/sys:ro
  - /var/lib/docker/:/var/lib/docker:ro
  - /dev/disk/:/dev/disk:ro
```

Un contenedor está aislado y por defecto no ve a los demás. Para medirlos,
cAdvisor necesita mirar el host: `/sys` tiene los cgroups (donde el kernel lleva
la cuenta de CPU y memoria por contenedor), `/var/run` el socket de Docker,
`/var/lib/docker` el estado de los contenedores y `/dev/disk` los discos.
Todo va con `:ro` porque cAdvisor solo observa: se le da mucha visibilidad sobre
el host, así que se le quita la posibilidad de escribir.

### Qué puerto se publica y por qué

```yaml
ports:
  - "9090:9090"
```

Solo Prometheus publica un puerto (formato `host:contenedor`), porque hay que
entrar a su interfaz web desde el navegador para ver los targets y hacer consultas
PromQL. cAdvisor no publica nada: su único consumidor es Prometheus, que lo
alcanza por la red interna como `cadvisor:8080`.

### Qué resuelve `extra_hosts`

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

La app Flask del CRUD no corre en un contenedor: el `setup.sh` la levanta
directamente en el host (`nohup python3 app.py`) en el puerto 8888. Al no estar
en `mynetwork`, Prometheus no puede encontrarla por nombre de servicio.
`extra_hosts` agrega una entrada al `/etc/hosts` del contenedor para que
`host.docker.internal` apunte a la IP del host (`host-gateway`). Así funciona el
target `host.docker.internal:8888` del job `crud-app`. En Linux ese nombre no
existe por defecto, por eso hay que declararlo.

### Sintaxis YAML de esta sección

- **Strings entre comillas**: `"9090:9090"` y `"host.docker.internal:host-gateway"`
  van entre comillas para que YAML los tome como texto y no intente interpretar
  los dos puntos de otra forma.
- **Listas de strings**: `volumes`, `ports` y `extra_hosts` son listas donde cada
  item sigue el patrón `origen:destino`.
- **Versión fijada**: `cadvisor:v0.47.2` usa un tag concreto, a diferencia de
  `prometheus:latest`.
- **Rutas relativas vs. absolutas**: `./prometheus.yml` es relativa a la carpeta
  del compose; las de cAdvisor (`/sys`, `/var/run`) son absolutas del host.

## Parte 3 - Visualización y topología

### 1. Servicio Grafana

Grafana es el servicio encargado de mostrar de forma gráfica los datos que recolecta Prometheus. Se ejecuta dentro de un contenedor Docker a partir de una imagen de Grafana subida por la cuenta oficial creadora.

La configuración del servicio en `docker-compose.yml` es:

```yaml
grafana:
  image: grafana/grafana:13.2.2
  restart: always
  ports:
    - ¨3000:3000¨
  environment:
    - GF_SECURITY_ADMIN_PASSWORD=admin
  depends_on:
    - prometheus
  networks:
    - mynetwork
```

#### ¿Qué hace cada elemento?

| Elemento | Explicación |
|---|---|
| `grafana:` | Es el nombre que identifica al servicio dentro del archivo `docker-compose.yml`. |
| `image: grafana/grafana:13.2.2` | Indica que Docker debe utilizar la imagen de Grafana como base para crear el contenedor donde se ejecutará el servidor Grafana. `13.2.2` indica la versión marcada que se tiene que utilizar.De esta forma se garantiza la inmutabilidad y estabilidad del entorno |
| `restart: always` | Indica que Docker debe intentar reiniciar automáticamente el contenedor si este se detiene. |
| `ports: ¨3000:3000¨` | Establece un mapeo de puertos entre el host y el contenedor. Define que cualquier tráfico que ingrese al puerto 3000 del host será redirigido automáticamente al puerto 3000 interno donde escucha Grafana. |
| `environment:` | Permite definir variables de entorno que serán utilizadas por el contenedor. |
| `GF_SECURITY_ADMIN_PASSWORD: admin` | Establece la contraseña del usuario `admin` de Grafana mediante una variable de entorno. |
| `depends_on: prometheus` | Establece el orden de inicio de los servicios. Le indica a Docker que debe arrancar el contenedor Prometheus antes de iniciar el de Grafana, asegurando que la base de datos de métricas estén disponibles en la red desde el principio. |
| `networks:` | Indica a qué red o redes de Docker pertenece el servicio. |
| `- mynetwork` | Conecta el contenedor de Grafana a la red `mynetwork`, permitiendo que se comunique con los demás servicios conectados a esa red. |

### Grafana y la red mynetwork: por qué Grafana depende de Prometheus, y cómo la red común permite resolver por nombre entre los cinco servicios

#### 1. Relación de dependencia entre Grafana y Prometheus
En este stack de monitoreo, **Grafana depende estrictamente de Prometheus porque actúa únicamente como la capa de interfaz y visualización**, lo que significa que no recolecta, genera ni almacena ningún tipo de métrica por sí mismo. 

Su función es conectarse a Prometheus (que funciona como la base de datos y motor de recolección) para realizar consultas utilizando el lenguaje PromQL. Sin Prometheus recopilando el estado de los contenedores (vía cAdvisor), de la base de datos (vía dbexporter) y de la app CRUD, Grafana no tendría datos que transformar en gráficos o paneles de control. En el archivo `docker-compose.yml`, la directiva `depends_on: - prometheus` refleja esta jerarquía estructural, asegurando que Docker inicie primero el contenedor del motor de métricas antes de levantar la interfaz de usuario.

#### 2. Resolución por nombre en la red común `mynetwork`
Al declarar la red virtualizada `mynetwork` y conectar los cinco servicios a ella (`mysql`, `dbexporter`, `cadvisor`, `prometheus` y `grafana`), Docker Compose habilita de forma automática un **servidor DNS interno integrado**. 

Este mecanismo resuelve la comunicación del stack de la siguiente manera:
* **Registro de alias:** Docker asocia el nombre de cada servicio definido en el archivo YAML con la dirección IP privada y dinámica que le asigna a su respectivo contenedor dentro de la red aislada.
* **Resolución automática:** Los servicios no necesitan conocer las direcciones IP de los demás para comunicarse. Cuando Grafana necesita conectarse a su fuente de datos, utiliza la URL `http://prometheus:9090`. El DNS interno de Docker intercepta ese nombre ("prometheus") y lo traduce instantáneamente a la IP interna correcta del contenedor de métricas. Lo mismo ocurre cuando Prometheus busca el exportador de la base de datos usando el nombre `dbexporter`.

Esta topología permite un entorno seguro y aislado, donde los servicios dialogan entre sí por nombre de forma transparente sin necesidad de exponer todos sus puertos hacia la máquina host.
