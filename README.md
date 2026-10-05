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

## Parte 3 - Visualización y topología
