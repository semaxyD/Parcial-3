# Imagen oficial de MySQL
FROM mysql:latest

# Pasa el script del inicializador init a el contenedor
COPY init.sql /docker-entrypoint-initdb.d/

# Port defecto de MySQL
EXPOSE 3306
