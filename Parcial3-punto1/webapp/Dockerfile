#Usamos la verison liviana de Python 3
FROM python:3.11-slim

#Trabajamos sobre el directorio /app
WORKDIR /app

#Copiamos el codigo para ponerlo dentro del contenedor
COPY . /app

# Instalamos las dependencias del sistema necesarias para mysqlclient
RUN apt-get update && \
    apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir Flask flask-cors Flask-MySQLdb Flask-SQLAlchemy

#Exponemos nuestra app web en el puerto 5000
EXPOSE 5000

CMD ["python", "run.py"]
