# Explicación de docker-compose.yml — Stack de monitoreo

## Introducción

En este trabajo se analiza la configuración de un stack de monitoreo implementado mediante Docker Compose. El objetivo es comprender cómo se organizan y se comunican los distintos servicios que lo componen, y qué función cumplen las principales directivas utilizadas en los archivos de configuración.

El stack está integrado por `mysql`, que proporciona la base de datos; `dbexporter`, que obtiene y expone sus métricas; `cadvisor`, que permite monitorear los recursos utilizados por los contenedores; `prometheus`, que recolecta y almacena las métricas; y `grafana`, que permite visualizarlas mediante gráficos y paneles.

Para explicar su funcionamiento, dividimos el análisis en cuatro partes. Primero, abordamos la persistencia de datos y la configuración de MySQL y `dbexporter`. Luego, analizamos la recolección de métricas mediante cAdvisor y Prometheus. En tercer lugar, explicamos la visualización de los datos con Grafana y la comunicación entre los servicios a través de la red `mynetwork`. Por último, estudiamos el archivo `prometheus.yml`, que define cómo Prometheus identifica y consulta los distintos objetivos de recolección de métricas (*targets*).

A lo largo del trabajo, explicamos tanto la función de cada componente como la sintaxis de las configuraciones utilizadas, con el propósito de comprender cómo se integra el stack y cómo interactúan sus servicios.

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

Definir la contraseña mediante una variable de entorno facilita la configuración del servicio y permite cambiar el valor sin modificar el resto de la configuración. Sin embargo, esto no garantiza que la contraseña sea secreta, ya que puede quedar expuesta en el archivo docker-compose.yml o en la configuración del contenedor. Para entornos reales, es preferible utilizar mecanismos específicos para gestionar secretos.

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

El modo de solo lectura (`:ro`) se utiliza como medida de protección para evitar que el contenedor modifique el archivo de configuración original. De esta manera, `dbexporter` puede consultar los datos necesarios para conectarse a MySQL sin alterar el archivo `my.cnf` del host.

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

## Parte 4 — Configuración de Prometheus (`prometheus.yml`)

El archivo `prometheus.yml` es el encargado de definir el comportamiento del motor de monitoreo. Establece la frecuencia de recolección de datos y los objetivos específicos a los que debe consultar en el entorno.

La configuración del archivo es:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'mysql'
    static_configs:
      - targets: ['dbexporter:9104']

  - job_name: 'crud-app'
    static_configs:
      - targets: ['host.docker.internal:8888']
```

#### ¿Qué hace cada elemento?

| Elemento | Explicación |
|---|---|
| `global:` | Permite definir directivas de configuración generales que se aplicarán por defecto a todas las tareas de monitoreo si no se especifica lo contrario. |
| `scrape_interval: 5s` | Establece el intervalo de tiempo en el que Prometheus realiza la recolección de métricas. Define que cada 5 segundos se consultará el estado de todos los objetivos configurados. |
| `scrape_configs:` | Estructura la lista de tareas de recolección, organizando bajo bloques individuales cada uno de los sistemas o aplicaciones que se van a monitorear. |
| `job_name: 'prometheus'` | Prometheus se monitorea a sí mismo de forma nativa. Al apuntar a su propia dirección local y puerto, recolecta métricas de su propio rendimiento interno (uso de memoria, consultas procesadas, tiempos de almacenamiento, etc). |
| `targets: ['localhost:9090']` | Define el endpoint interno de destino para el primer trabajo, apuntando a su propio puerto local. |
| `job_name: 'cadvisor'` | Asigna un nombre identificatorio a la tarea encargada de registrar el consumo de hardware de los contenedores Docker en ejecución. |
| `targets: ['cadvisor:8080']` | Define la dirección de destino para cAdvisor, aprovechando el puerto 8080 dentro de la red del escenario. |
| `job_name: 'mysql'` | Asigna un nombre identificatorio a la tarea dedicada a capturar las estadísticas de la base de datos MySQL. |
| `targets: ['dbexporter:9104']` | Prometheus no puede extraer estadísticas directamente desde el servidor MySQL porque no entiende el lenguaje nativo de las bases de datos. Por este motivo, se utiliza `dbexporter` como un intermediario o traductor. Este componente se encarga de consultar los datos internos de MySQL, transformarlos al formato estándar de métricas y dejarlos listos en el puerto 9104 para que Prometheus los pueda recolectar.
| `job_name: 'crud-app'` | Asigna un nombre identificatorio a la tarea orientada a medir el comportamiento y las peticiones de la aplicación web Flask. |
| `targets: ['host.docker.internal:8888']` | Realiza el scrape a la aplicación desarrollada en Flask. Como la app de Python corre directamente sobre el sistema operativo del host (no en un contenedor) en el puerto `8888`, se utiliza la dirección especial `host.docker.internal` para que el contenedor de Prometheus pueda "salir" hacia la máquina real y encontrar el endpoint `/metrics`.


## Conclusión

Al analizar las cuatro partes, comprendimos que el `docker-compose.yml` no define
cinco contenedores independientes, sino una cadena en la que cada servicio cumple
un rol dentro del flujo de datos. MySQL genera el estado que queremos observar;
`dbexporter` y cAdvisor lo transforman en métricas; Prometheus las recolecta y
almacena según lo definido en `prometheus.yml`; y Grafana las presenta en forma de
gráficos y paneles. Si alguno de estos eslabones deja de funcionar, los servicios
que dependen de él se quedan sin datos.

A lo largo del trabajo identificamos algunas decisiones de diseño que se repiten:

- **Mínimo privilegio:** los archivos y directorios que solo necesitan leerse se
  montan con `:ro`, como `my.cnf` en `dbexporter` y las rutas del host en cAdvisor.
- **Mínima exposición:** de los cinco servicios, solo Prometheus (9090) y Grafana
  (3000) publican puertos hacia el host, porque son los que accedemos desde el
  navegador. El resto se comunica únicamente a través de `mynetwork`.
- **Configuración fuera de la imagen:** las contraseñas de MySQL y Grafana se
  definen mediante variables de entorno, y los archivos `my.cnf` y `prometheus.yml`
  se montan desde el host, de modo que podemos cambiar el comportamiento de cada
  servicio sin modificar sus imágenes.
- **Orden de inicio con `depends_on`:** aparece en `dbexporter` y en `grafana`, y en
  ambos casos establece el orden en que arrancan los servicios, pero no garantiza
  que el servicio del que dependen ya esté listo para recibir conexiones.

Encontramos que el archivo `prometheus.yml` funciona como el punto donde se
integran todas las partes. Sus *targets* reflejan las dos formas de comunicación
del stack: los servicios conectados a `mynetwork` se consultan por su nombre
(`cadvisor:8080`, `dbexporter:9104`) gracias al DNS interno de Docker, mientras
que la aplicación Flask, que se ejecuta directamente en el host y fuera de la red,
solo puede alcanzarse mediante `host.docker.internal:8888`, nombre que existe
dentro del contenedor de Prometheus gracias a la directiva `extra_hosts`. Este
contraste fue lo que mejor nos permitió entender la función de la red común:
cuando un componente queda fuera de ella, es necesario configurar explícitamente
cómo encontrarlo.

En cuanto a la sintaxis, observamos que tanto `docker-compose.yml` como
`prometheus.yml` se construyen con los mismos elementos de YAML: mapas
(`clave: valor`), listas (`- elemento`) e indentación con espacios para expresar
qué elementos están contenidos dentro de otros. Comprender estas tres estructuras
nos permitió leer e interpretar cualquier sección de ambos archivos.