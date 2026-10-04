# Explicación de docker-compose.yml — Stack de monitoreo

## Parte 1 — Persistencia de datos

mysql:
  image: mysql:latest
  restart: always
  environment:
    MYSQL_ROOT_PASSWORD: mysecretpassword
  networks:
    - mynetwork


mysql: es el nombre del servicio dentro de Docker Compose.
image: mysql:latest indica qué imagen va a utilizar Docker para crear el contenedor. En este caso, la imagen de MySQL, usando la etiqueta latest.

Prueba de mi primer cambio en la documentación.

## Parte 3 - Visualización y topoligía
