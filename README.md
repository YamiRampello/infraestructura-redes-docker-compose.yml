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


## Parte 2


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

