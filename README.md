<div>


::: {.page-header-icon .page-header-icon-with-cover}
[🌐]{.icon}
:::

# Documentación Paso a Paso para el Parcial 3 de ST {#documentación-paso-a-paso-para-el-parcial-3-de-st .page-title}

</div>

:::::::::::: page-body
### [**Primera Parte:**]{.mark .highlight-red} {#201dab02-9ae2-81ef-92ed-ff9aedd79423}

1: **Clonar el repositorio base (MiniWebApp)**

Este repositorio será la base en la cual trabajaremos.

``` {#201dab02-9ae2-8176-a7d6-e7af2c7dc817 .code}
git clone https://github.com/usuario/MiniWebApp.git
cd MiniWebApp
```

2: **Elegir un servidor web**

::: indented
Elegimos **Nginx** como proxy inverso. Se encargará de manejar el
tráfico HTTPS y redirigirlo a nuestra app interna.
:::

**3: Creación de los certificados SSL**

::: indented
Usaremos el siguiente comando el cual nos creara dos archivos, uno será
el certificado con la información proporcionada al momento de correr el
comando, y el segundo será la key de seguridad que usara Nginx mas
adelante:
:::

``` {#201dab02-9ae2-8140-8b04-e06bf191a64a .code}
openssl req -x509 -newkey rsa:2048 -nodes -keyout selfsigned.key -out selfsigned.crt -days 365
```

### 🧾 Archivos generados: {#201dab02-9ae2-81fa-8ed5-ce6ea3f52be7}

`selfsigned.key` → 🔐 **Clave privada**

- Es un archivo secreto que **debes proteger muy bien**.

<!-- -->

- Lo usa el servidor (como Nginx o Apache) para **desencriptar la
  información cifrada** que recibe.

<!-- -->

- No se comparte **nunca** con los clientes o el público.

<!-- -->

- Contiene la **private key RSA**.

`selfsigned.crt` → 📢 **Certificado público**

- Contiene la **clave pública** y metadatos como:
  - Nombre común (CN)

  <!-- -->

  - Organización

  <!-- -->

  - País

  <!-- -->

  - Validez (365 días)

<!-- -->

- Es lo que se entrega al navegador para **verificar la identidad del
  servidor**.

<!-- -->

- El navegador usa la **clave pública** para cifrar los datos que solo
  el servidor (con la clave privada) podrá leer.

### 🔁 Relación entre ellos {#201dab02-9ae2-8106-a9e5-eab16e3603ba}

- El `.crt` **está firmado** con el `.key`.

<!-- -->

- Cuando un cliente se conecta por HTTPS, el navegador recibe el `.crt`,
  lo valida, y **usa la clave pública** para cifrar los datos.

<!-- -->

- El servidor los desencripta usando la `.key`.

4: Configurar Nginx para servir por HTTPS

Creamos un archivo `nginx/nginx.conf` con esta configuración:

``` {#201dab02-9ae2-81bd-aff0-c920525342e5 .code}
server {
    listen 80;
    server_name localhost;
    #Redireccionamiento de HTTP -> HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    #Rutas de los archivos para el uso de SSL
    ssl_certificate     /etc/nginx/certs/selfsigned.crt;
    ssl_certificate_key /etc/nginx/certs/selfsigned.key;

    location / {
        proxy_pass http://webapp:5000; #Flask que esta en este contenedor
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Explicacion:

1.  **`location / {`**
    - Esta línea define un bloque de configuración para manejar las
      solicitudes que llegan a la raíz (`/`) del servidor. Todo lo que
      se encuentre bajo este bloque se aplicará a las solicitudes que
      coincidan con esta ubicación.

<!-- -->

2.  **`proxy_pass http://webapp:5000;`**
    - Este comando indica que las solicitudes que lleguen a esta
      ubicación deben ser redirigidas al servidor backend que se
      encuentra en `http://webapp:5000`. Aquí, `webapp` es el nombre del
      contenedor o servicio que ejecuta la aplicación Flask, y `5000` es
      el puerto en el que está escuchando.

<!-- -->

3.  **`proxy_set_header Host $host;`**
    - Esta línea establece el encabezado `Host` de la solicitud que se
      envía al servidor backend. `$host` se reemplaza con el nombre del
      host de la solicitud original. Esto es útil para que el backend
      sepa cuál es el dominio que se utilizó para acceder a la
      aplicación.

<!-- -->

4.  **`proxy_set_header X-Real-IP $remote_addr;`**
    - Aquí se establece el encabezado `X-Real-IP`, que contiene la
      dirección IP del cliente que hizo la solicitud original. Esto
      permite que el servidor backend conozca la IP real del usuario, en
      lugar de la IP del servidor proxy.

<!-- -->

5.  **`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`**
    - Este encabezado se utiliza para mantener un registro de las
      direcciones IP de todos los proxies que han manejado la
      solicitud. `$proxy_add_x_forwarded_for` agrega la dirección IP del
      cliente a la lista de direcciones IP que ya están en el
      encabezado `X-Forwarded-For`.

<!-- -->

6.  **`proxy_set_header X-Forwarded-Proto $scheme;`**
    - Esta línea establece el encabezado `X-Forwarded-Proto`, que indica
      el protocolo utilizado por el cliente para hacer la solicitud (ya
      sea `http` o `https`). `$scheme` se reemplaza automáticamente con
      el esquema de la solicitud original.

5: Crear el Dockerfile de la aplicación

En este caso creamos dos archivos `Dockerfile` ,uno para nuestra bd
mysql y otro para la app web con Flask

Dockerfile dentro de `mysql/`

``` {#201dab02-9ae2-8126-8e75-c34275cffee0 .code}
# Imagen oficial de MySQL
FROM mysql:latest

# Pasa el script del inicializador init a el contenedor
COPY init.sql /docker-entrypoint-initdb.d/

# Port defecto de MySQL
EXPOSE 3306
```

Dockerfile dentro de `webapp/`

``` {#201dab02-9ae2-819a-af72-feab59a0b827 .code}
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
```

6: creación del archivo `docker-compose.yml` para poder orquestar los
servicios y montar toda la app funcional

Este archivo orquesta Nginx que expone el servicio en HTTPS, la app como
tal construida en Flask y el acceso a los datos de la bd MySQL:

``` {#201dab02-9ae2-8180-a9ac-dd17ac11042e .code}
version: '3.8'

services:
  webapp:
    build:
      context: ./webapp
      dockerfile: Dockerfile
    container_name: webapp
    restart: always
    expose:
      - "5000"
    environment:
      - FLASK_ENV=production
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=myflaskapp
    depends_on:
      - mysql
  nginx:
    build:
      context: ./nginx
      dockerfile: Dockerfile
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - webapp
    volumes:
      - ./nginx/certs:/etc/nginx/certs:ro
  mysql:
    build:
      context: ./mysql
      dockerfile: Dockerfile
    container_name: mysql
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=root
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

7: Probar que todo funcione localmente

> ⚠️ Antes de esto y si deseas iniciar o correr comando de Docker sin
> sudo, podemos agregar un usuario al grupo de Docker con el siguiente
> comando:

`sudo usermod -aG docker $USER` ,luego cerramos sesión con `exit`y
ingresamos de nuevo con ssh *(Puede aplicar para probar localmente los
contenedores y en AWS)*

Construimos y levantamos los contenedores con:

``` {#201dab02-9ae2-815f-9a1c-f72dd39e350e .code}
docker-compose up --build
```

Luego entramos al navegador por medio de `https://localhost` o
`https://192.168.70.3`

> ⚠️ Si vemos un aviso de \"sitio no seguro\", es normal con
> certificados autofirmados. Solo haz clic en \"avanzado\" y continúa.

### [**Segunda Parte:**]{.mark .highlight-red} {#201dab02-9ae2-8132-afea-d4a3b64ff73a}

- Nos conectamos al lab de AWS Academy

<figure id="201dab02-9ae2-81a9-88b8-db27672aa38c" class="image">
<a href="image.png"><img src="image.png"
style="width:681.984375px" /></a>
</figure>

- seleccionamos lanzar instancia EC2 y asignamos nombre,cantidad de
  instancias,imagen o SIO

<figure id="201dab02-9ae2-8124-89a7-c75f004a1b1c" class="image">
<a href="image%201.png"><img src="image%201.png"
style="width:681.984375px" /></a>
</figure>

- luego creamos un par de claves para la nueva instancia,esto nos creara
  un archivo el cual le daremos permisos mas adelante para conectarnos

<figure id="201dab02-9ae2-8111-8d0f-db054c4009e1" class="image">
<a href="image%202.png"><img src="image%202.png"
style="width:681.984375px" /></a>
</figure>

- ahora entramos al grupo de seguridad de una de las instancias que será
  nuestro server donde tendremos la API Rest para activar reglas del
  firewall y que permita usar puertos como el del HTTP,HTTPS y SSH

<figure id="201dab02-9ae2-81e4-88ba-fc0529d5370b" class="image">
<a href="image%203.png"><img src="image%203.png"
style="width:681.96875px" /></a>
</figure>

- Ahora vamos a conectarnos por medio del client SSH, para eso vamos a
  darles permisos a el par de claves y luego nos conectamos con los
  siguientes comando via terminal
  - para la MV AWS:
    `ssh -i "punto2Keys.pem" `[`ubuntu@ec2-52-91-98-150.compute-1.amazonaws.com`](mailto:ubuntu@ec2-52-91-98-150.compute-1.amazonaws.com)

2: instalación de Docker y Docker compose para ejecutar el contenedor de
la app de la parte 1

Instalar Docker:

``` {#201dab02-9ae2-81fa-b203-c5035f868a18 .code}
sudo apt update && sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

Instalar Docker Compose:

``` {#201dab02-9ae2-811f-97fd-edffdb06d5a7 .code}
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

3: Despliegue de la app web en AWS (en mi caso lo hice mediante la
clonación de mi repositorio GitHub)

``` {#201dab02-9ae2-813a-8e82-fb19f999fd1d .code}
git clone https://github.com/semaxyD/Parcial-3
cd Parcial-3/Parcial3-punto1
docker-compose up --build -d
```

y verificamos su funcionamiento desde la ip https://\<IP-PÚBLICA-EC2\>

### [**Tercera Parte:**]{.mark .highlight-red} {#201dab02-9ae2-819a-8701-d549d0c0dd7f}

**1.1: Creación de usuario para el uso de prometheus y directorios
necesarios**

Crear usuario:
`sudo useradd --no-create-home --shell /bin/false prometheus`

Crear directorios:

`sudo mkdir /etc/prometheus`\
`sudo mkdir /var/lib/prometheus`\

Explicación:

- **`useradd --no-create-home --shell /bin/false prometheus`**

  Se crea un usuario del sistema llamado `prometheus`, **sin directorio
  personal** y con el Shell deshabilitado (`/bin/false`).

  Esto se hace por seguridad: este usuario será usado únicamente por el
  servicio Prometheus, sin necesidad de iniciar sesión.

  ⚠️ *Buena práctica de*
  *[hardening]{style="border-bottom:0.05em solid"}* *para servicios de
  red.*

<!-- -->

- **`/etc/prometheus`**

  Se crea este directorio para guardar archivos de configuración, como
  `prometheus.yml`, `rules.yml`, etc.

  📁 *Es el equivalente a lo que sería* *`/etc/nginx/`* *para*
  *[Nginx]{style="border-bottom:0.05em solid"}, o* *`/etc/apache2/`*
  *para* *[Apache]{style="border-bottom:0.05em solid"}.*

<!-- -->

- **`/var/lib/prometheus`**

  Este directorio será usado por Prometheus para almacenar sus **series
  temporales (time series data)** y **metadatos de métricas**.

  🧠 *Es la base de datos local de Prometheus: guarda métricas
  recolectadas.*

**1.2: Instalación de prometheus desde su repositorio en GitHub (versión
2.45.0)**

`cd /tmp`\
`wget`\
[`https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz`](https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz)\
`tar xvf prometheus-2.45.0.linux-amd64.tar.gz`\
`cd prometheus-2.45.0.linux-amd64`\

<figure id="201dab02-9ae2-81b2-baa8-d4fcdbaca640" class="image">
<a href="image%204.png"><img src="image%204.png"
style="width:681.984375px" /></a>
</figure>

Explicación:

- **`wget ...`**

  Se descarga directamente desde GitHub la versión oficial **2.45.0** de
  Prometheus en formato `.tar.gz`, que es el recomendado para
  instalaciones manuales.

  🌐 *Descarga segura y reproducible desde el repositorio oficial.*

<!-- -->

- **`tar xvf ...`**

  Se descomprime el archivo `.tar.gz`, lo cual genera una carpeta
  llamada `prometheus-2.45.0.linux-amd64` con todos los binarios y
  archivos necesarios.

  📦 *Formato estándar para software distribuido como binario en Linux.*

<!-- -->

- **`cd prometheus-2.45.0.linux-amd64`**

  Se accede al directorio descomprimido para proceder con la instalación
  y configuración.

**1.3: movemos y damos permisos a los archivos binarios de prometheus**

copiar:

`sudo cp prometheus /usr/local/bin/`\
`sudo cp promtool /usr/local/bin/`\

dar permisos:\
\
`sudo chown prometheus:prometheus /usr/local/bin/prometheus`\
`sudo chown prometheus:prometheus /usr/local/bin/promtool`\

Explicación:

- **`cp prometheus /usr/local/bin/`**

  Copia el binario principal de Prometheus al directorio
  `/usr/local/bin/`, el cual es una ruta estándar para binarios de
  software instalado manualmente.

  Esto permite ejecutar `prometheus` desde cualquier lugar del sistema
  sin necesidad de rutas absolutas.

  📌 *Buena práctica: mantener separados los binarios del sistema (en*
  *`/bin`* *o* *`/usr/bin`) de los instalados por el usuario.*

<!-- -->

- **`cp promtool /usr/local/bin/`**

  De igual forma, se copia `promtool`, una herramienta de validación y
  prueba para configuraciones y reglas de Prometheus.

  🛠️ *Herramienta útil para verificar archivos* *`prometheus.yml`* *o
  reglas de alertas antes de aplicarlas.*

<!-- -->

- Asignar propiedad **(`chown`)**

  Se cambia el propietario de ambos binarios al usuario `prometheus`,
  que fue creado anteriormente con acceso limitado.

  Esto garantiza que solo ese usuario pueda interactuar directamente con
  los binarios, reduciendo riesgos de seguridad.

  🔐 *Principio de privilegios mínimos: solo el usuario que ejecutará el
  servicio necesita tener control sobre los binarios.*

comprobamos permisos con el comando →
`ls -l /usr/local/bin | grep prometheus`

<figure id="201dab02-9ae2-81f6-be14-dda9948bac8a" class="image">
<a href="image%205.png"><img src="image%205.png"
style="width:681.921875px" /></a>
</figure>

**1.4: Mover configuración y consolas**

`sudo cp -r consoles /etc/prometheus`\
`sudo cp -r console_libraries /etc/prometheus`\
`sudo cp prometheus.yml /etc/prometheus/`\
`sudo chown -R prometheus:prometheus /etc/prometheus`\
`sudo chown -R prometheus:prometheus /var/lib/prometheus`\

Explicación:

- Copiar archivos de configuración **(`cp -r`)**

  - `consoles` y `console_libraries`: carpetas utilizadas por la
    interfaz web de Prometheus para mostrar dashboards predeterminados.

  <!-- -->

  - `prometheus.yml`: archivo principal de configuración de Prometheus,
    donde se definen los targets a monitorear, reglas, y scrape
    intervals.

  📁 Todos estos archivos se ubican en `/etc/prometheus`, una ubicación
  estándar para configuraciones persistentes de servicios en Linux.

<!-- -->

- Asignar propiedad **(`chown -R`)**

  - Se otorga la propiedad completa de `/etc/prometheus` y
    `/var/lib/prometheus` al usuario `prometheus`.

  <!-- -->

  - Esto asegura que el servicio pueda leer sus configuraciones y
    almacenar datos de forma segura sin intervención de root.

  🔒 *Esto aísla la ejecución del servicio y mejora la seguridad al
  evitar accesos innecesarios del sistema o de otros usuarios.*

**1.5: Creamos un servicio systemd**

primero un archivo que llamaremos prometheus.service dentro de
`/etc/systemd/system/`

dentro de el ponemos la siguiente configuración:

::: indented
`[Unit]`\
`Description=Prometheus`\
`Wants=network-online.target`\
`After=network-online.target`\

`[Service]`\
`User=prometheus`\
`Group=prometheus`\
`Type=simple`\
`ExecStart=/usr/local/bin/prometheus \`\
`--config.file=/etc/prometheus/prometheus.yml \`\
`--storage.tsdb.path=/var/lib/prometheus/ \`\
`--web.console.templates=/etc/prometheus/consoles \`\
`--web.console.libraries=/etc/prometheus/console_libraries`\

`[Install]`\
`WantedBy=multi-user.target`\
:::

Explicación:

- **\[Unit\]**:
  - Se asegura de que Prometheus se inicie *después de que la red esté
    disponible* (`network-online.target`), lo cual es crucial para
    monitorear otros servicios.

<!-- -->

- **\[Service\]**:
  - Ejecuta Prometheus como el usuario restringido `prometheus`, lo que
    evita riesgos de seguridad.

  <!-- -->

  - `Type=simple`: el servicio se considera activo mientras el proceso
    principal esté corriendo.

  <!-- -->

  - `ExecStart`: se definen los argumentos de ejecución:
    - `-config.file`: ruta al archivo de configuración principal.

    <!-- -->

    - `-storage.tsdb.path`: base de datos de series temporales.

    <!-- -->

    - `-web.console.*`: define las rutas para las consolas web de
      Prometheus.

<!-- -->

- **\[Install\]**:
  - Hace que el servicio se inicie automáticamente al arrancar el
    sistema (modo multiusuario).

1.6: **Activamos y comprobamos para poder acceder desde**
**`http://<IP-SERVIDOR>:9090`**

`sudo systemctl daemon-reexec`\
`sudo systemctl daemon-reload`\
`sudo systemctl start prometheus`\
`sudo systemctl enable prometheus`\

Explicación:

- `daemon-reexec`:

  Reinicia el proceso del sistema `systemd`. Es útil (aunque no siempre
  necesario) después de instalar nuevos binarios de systemd o cuando hay
  dudas con la configuración. Se puede omitir en algunos casos, pero lo
  incluimos por buena práctica.

<!-- -->

- `daemon-reload`:

  Informa a `systemd` que hay nuevos archivos de unidad (como
  `prometheus.service`) para que los recargue.

<!-- -->

- `start prometheus`:

  Inicia el servicio de Prometheus inmediatamente.

<!-- -->

- `enable prometheus`:

  Hace que Prometheus se inicie automáticamente en cada arranque del
  sistema.

Asi debe de aparecernos:

<figure id="201dab02-9ae2-81a6-a2a3-c7708744fa8a" class="image">
<a href="image%206.png"><img src="image%206.png"
style="width:682px" /></a>
</figure>

1.7: **Modificación del archivo prometheus.yml para el monitoreo de la
maquina local**

Cuando instalamos prometheus en nuestras maquinas por lo defecto ya
viene con una configuración básica en el archivo prometheus.yml,
nosotros agregaremos el siguiente código para la habilitación del
monitoreo.

Ruta: `/etc/prometheus`

<figure id="201dab02-9ae2-81d7-ac72-ea192ccf34ac" class="image">
<a href="image%207.png"><img src="image%207.png"
style="width:681.984375px" /></a>
</figure>

comprobamos que este bien usando promtool

`promtool check config /etc/prometheus/prometheus.yml`

Explicación:

- `global`

  Este bloque define valores generales para Prometheus:

  - **`scrape_interval: 15s`**

    Intervalo con el que Prometheus recopila (scrapea) las métricas de
    todos los endpoints configurados. Aquí está ajustado a 15 segundos,
    lo que significa que cada 15 segundos Prometheus hace una petición
    para obtener datos nuevos.

    *(Valor por defecto es 1 minuto, 15s es más frecuente para monitoreo
    más fino).*

  <!-- -->

  - **`evaluation_interval: 15s`**

    Intervalo con el que Prometheus evalúa las reglas de alerta y
    grabación (aunque en esta configuración no hay reglas activas, se
    deja preparado para que en el futuro pueda evaluarlas cada 15
    segundos).

<!-- -->

- `alerting`
  - Aquí se configura la comunicación con Alertmanager (que se encarga
    de gestionar alertas y notificaciones).

  <!-- -->

  - En este caso, el bloque está vacío (`targets: []`), ya que no hemos
    configurado alertas aún.

  <!-- -->

  - Esto es común en configuraciones iniciales para ir agregando alertas
    más adelante.

<!-- -->

- `rule_files`
  - Aquí se listan archivos con reglas para evaluar condiciones
    específicas y generar alertas o métricas calculadas.

  <!-- -->

  - Está vacío ahora porque no hemos definido ninguna regla
    personalizada.

<!-- -->

- `scrape_configs`

Es el corazón de la configuración: define qué endpoints Prometheus debe
consultar para obtener métricas.

::: indented
- **Job** **`"prometheus"`**:
  - Prometheus se auto-monitorea a sí mismo.

  <!-- -->

  - `targets: ["localhost:9090"]` significa que Prometheus consulta su
    propio endpoint de métricas (por defecto en el puerto 9090) para
    saber su estado y desempeño.

<!-- -->

- **Job** **`"node_exporter"`**:
  - Aquí agregamos el endpoint de Node Exporter, que es el agente que
    corre en la máquina y expone métricas del sistema operativo (CPU,
    memoria, disco, red, etc).

  <!-- -->

  - `targets: ["localhost:9100"]` indica que se conecta al Node Exporter
    en el puerto 9100, escuchando en localhost.
:::

1.8: **Documentación de rutas y archivos configurados para prometheus
(Resumen por que ya se ha explicado antes)**

  ------------------------------------------ --------------------------------------------------------------------------------------------------------------------------------------------------
  Ruta/Archivo                               Descripción
  `/etc/prometheus/prometheus.yml`           **Archivo principal de configuración de Prometheus.** Aquí se definen los endpoints a scrapear, las reglas, alertas y otros parámetros globales.
  `/etc/prometheus/consoles/`                Contiene las plantillas HTML para la interfaz web de Prometheus. Se usan para mostrar métricas en la consola web.
  `/etc/prometheus/console_libraries/`       Bibliotecas compartidas utilizadas por las consolas.
  `/var/lib/prometheus/`                     Directorio de almacenamiento de datos de series temporales. Prometheus guarda aquí las métricas que recolecta.
  `/etc/systemd/system/prometheus.service`   Archivo de definición del servicio en systemd. Permite iniciar, detener y habilitar Prometheus como un servicio del sistema.
  `/usr/local/bin/prometheus`                Binario principal que ejecuta Prometheus.
  `/usr/local/bin/promtool`                  Herramienta CLI para hacer validaciones, pruebas y otras tareas de administración.
  ------------------------------------------ --------------------------------------------------------------------------------------------------------------------------------------------------

2.1: **Instalación y configuración de Node Exporter**

para la descarga lo haremos desde un repositorio del GitHub y en el
directorio tmp

`cd /tmp`\
`wget`\
[`https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz`](https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz)\
`tar xvf node_exporter-1.8.0.linux-amd64.tar.gz`\
`cd node_exporter-1.8.0.linux-amd64`\

<figure id="201dab02-9ae2-81d9-879e-c9b20c991210" class="image">
<a href="image%208.png"><img src="image%208.png"
style="width:709.984375px" /></a>
</figure>

2.2: **Crearemos un usuario y moveremos los binarios necesario para su
ejecución correcta**

`sudo useradd --no-create-home --shell /bin/false node_exporter`\
`sudo cp node_exporter /usr/local/bin/`\
`sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter`\

2.3: **Creación del Service Node Exporter dentro del directorio**
**`/etc/systemd/system/`** **con el siguiente contenido**

`[Unit]`\
`Description=Node Exporter`\
`Wants=network-online.target`\
`After=network-online.target`\

`[Service]`\
`User=node_exporter`\
`Group=node_exporter`\
`Type=simple`\
`ExecStart=/usr/local/bin/node_exporter`\

`[Install]`\
`WantedBy=multi-user.target`\

2.4: **Activamos y ejecutamos Node Exporter**

`sudo systemctl daemon-reexec`\
`sudo systemctl daemon-reload`\
`sudo systemctl start node_exporter`\
`sudo systemctl enable node_exporter`\

<figure id="201dab02-9ae2-8195-89a3-fd5b6cbf2924" class="image">
<a href="image%209.png"><img src="image%209.png"
style="width:709.921875px" /></a>
</figure>

por defecto se puede verificar en el puerto 9100 ene l navegador o
atravez de un comando Curl:

`curl `[`http://localhost:9100/metrics`](http://localhost:9100/metrics)

<figure id="201dab02-9ae2-81ca-a2d7-d2337cdcd48f" class="image">
<a href="image%2010.png"><img src="image%2010.png"
style="width:709.953125px" /></a>
</figure>

Dentro de la ruta metrics veremos algo como:

<figure id="201dab02-9ae2-81e1-9711-db4c6b318561" class="image">
<a href="image%2011.png"><img src="image%2011.png"
style="width:709.984375px" /></a>
</figure>

\
expone un\
**montón de métricas del sistema en formato Prometheus**, es decir, una
colección de datos crudos listos para ser recolectados (scrapeados) por
el propio Prometheus.\
\

Cada métrica tiene:

::: indented
  ------------ ------------------------------------------------------------------------------
  Elemento     Qué es
  `# HELP`     Una descripción corta de la métrica.
  `# TYPE`     El tipo de métrica (counter, gauge, histogram, etc.).
  `nombre`     El nombre de la métrica, como `node_cpu_seconds_total`.
  `{labels}`   Etiquetas que describen el contexto (por ejemplo: CPU 0, modo `idle`, etc.).
  valor        El valor actual de esa métrica.
  ------------ ------------------------------------------------------------------------------
:::

Node Exporter recolecta métricas de:

- 🔧 CPU (`node_cpu_seconds_total`)

<!-- -->

- 💾 Memoria (`node_memory_*`)

<!-- -->

- 📀 Disco (`node_disk_*`, `node_filesystem_*`)

<!-- -->

- 🌐 Red (`node_network_*`)

<!-- -->

- 🔌 Sensores, carga (`node_load1`, `node_load5`, etc.)

<!-- -->

- ⏱️ Tiempo de actividad (`node_time_seconds`, `node_boot_time_seconds`)

<!-- -->

- Y muchos más...

📌 Todas estas métricas están **formateadas para Prometheus**, por eso
se ven así planas y textuales. No es algo "bonito" para humanos, sino
óptimo para máquinas.

2.5: **Configuración de prometheus para monitorear Node Exporter**

Esta parte ya la habíamos dejado lista desde el momento que modificamos
y habilitaos el uso del Node Exporter dentro del archivo prometheus.yml,
pero por si algo, verificar que este bien escrito el puerto del Node
Exporter dentro de `-targets: [”localhost:9100”]`

2.6: **Verificación del funcionamiento de las métricas expuestas por
Node Exporte en Prometheus**

Corremos nuestro `http://<IP-SERVIDOR>:9090` y buscamos en un panel la
métrica que nos interesa, recuerden que podemos buscarlas dentro del
Node Exporter del punto 2.4.

En mi caso cree tres paneles y busque las siguientes métricas

- node_cpu_seconds_total

<figure id="201dab02-9ae2-819c-9e1d-fb9954ab78a5" class="image">
<a href="image%2012.png"><img src="image%2012.png"
style="width:685.984375px" /></a>
</figure>

- node_memory_MemAvailable_bytes

<figure id="201dab02-9ae2-81ae-8f79-fd7b8619a4eb" class="image">
<a href="image%2013.png"><img src="image%2013.png"
style="width:685.984375px" /></a>
</figure>

- node_filesystem_avail_bytes

<figure id="201dab02-9ae2-81eb-a040-df95d4062767" class="image">
<a href="image%2014.png"><img src="image%2014.png"
style="width:685.953125px" /></a>
</figure>

2.7: Documentación de las tres métricas elegidas:

1\. `node_cpu_seconds_total`

- **Tipo:** `counter`

<!-- -->

- **Descripción:**

  Representa la cantidad total de segundos que cada CPU ha pasado en
  diferentes modos (user, system, idle, iowait, etc.) desde el arranque
  del sistema.

<!-- -->

- **Ejemplo de métricas:**

  ``` {#201dab02-9ae2-8175-a214-f7cd9843c3ca .code}
  node_cpu_seconds_total{cpu="0", mode="idle"}     4950.09
  node_cpu_seconds_total{cpu="0", mode="user"}     18.41
  node_cpu_seconds_total{cpu="1", mode="system"}   53.78
  ```

<!-- -->

- **Modos comunes:**
  - `user`: tiempo ejecutando procesos de usuario.

  <!-- -->

  - `system`: tiempo en procesos del sistema/kernel.

  <!-- -->

  - `idle`: tiempo en que la CPU no hace nada.

  <!-- -->

  - `iowait`: esperando operaciones de disco.

  <!-- -->

  - `irq` y `softirq`: interrupciones de hardware y software.

<!-- -->

- **¿Cómo ayuda?**

  Esta métrica permite calcular la **utilización real de la CPU**. Por
  ejemplo, si el tiempo en `idle` es bajo y el de `system` o `iowait` es
  alto, puede indicar **cuellos de botella en disco o sobrecarga del
  sistema**. Es esencial para detectar problemas de rendimiento y
  balanceo de carga.

2\. `node_memory_MemAvailable_bytes`

- **Tipo:** `gauge`

<!-- -->

- **Descripción:**

  Indica cuántos **bytes de memoria RAM están disponibles para ser
  utilizados** por aplicaciones, sin necesidad de hacer swap. Es más
  preciso que simplemente \"libre\", ya que considera cachés que pueden
  ser liberadas si es necesario.

<!-- -->

- **Ejemplo:**

  ``` {#201dab02-9ae2-81c6-b232-c013f9e7ed99 .code}
  node_memory_MemAvailable_bytes{instance="localhost:9100"} 1234567890
  ```

<!-- -->

- **¿Cómo ayuda?**

  Esta métrica es clave para saber si **hay suficiente RAM disponible**
  o si el sistema está en riesgo de usar **memoria swap**, lo que
  degradaría el rendimiento. Un descenso sostenido en esta métrica puede
  indicar **fugas de memoria** o procesos que consumen más de lo
  esperado.

3\. `node_filesystem_avail_bytes`

- **Tipo:** `gauge`

<!-- -->

- **Descripción:**

  Reporta la cantidad de **espacio disponible (en bytes)** en cada
  sistema de archivos montado en el sistema.

<!-- -->

- **Ejemplo:**

  ``` {#201dab02-9ae2-812f-bdd5-d197fe3af294 .code}
  node_filesystem_avail_bytes{device="/dev/mapper/ubuntu--vg-ubuntu--lv", mountpoint="/"} 19699867648
  node_filesystem_avail_bytes{device="/dev/sda2", mountpoint="/boot"} 1667715072
  node_filesystem_avail_bytes{device="home_vagrant_miproyectoClient", mountpoint="/home/vagrant/miproyectoClient"} 6221398016
  ```

<!-- -->

- **¿Cómo ayuda?**

  Es fundamental para evitar que el sistema se quede sin espacio. Un
  volumen lleno puede **romper servicios, impedir escrituras o incluso
  causar caídas**. Esta métrica permite configurar alertas antes de que
  el disco llegue a niveles críticos de uso.

### [**Cuarta Parte:**]{.mark .highlight-red} {#201dab02-9ae2-81ad-b123-d796dcd6df32}

1.1: **Habilitación del repositorio de grafana**

usamos los comandos:

`sudo apt-get install -y software-properties-common`\
`sudo add-apt-repository "deb`\
[`https://packages.grafana.com/oss/deb`](https://packages.grafana.com/oss/deb)` stable main"`

<figure id="201dab02-9ae2-8184-8fca-ca9b287b5969" class="image">
<a href="image%2015.png"><img src="image%2015.png"
style="width:685.984375px" /></a>
</figure>

1.2: **Añadimos la clave GPG de Grafana**

agregamos con el siguiente comando

`wget -q -O - `[`https://packages.grafana.com/gpg.key`](https://packages.grafana.com/gpg.key)` | sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg`

y verificamos la referencia de esa key en el archivo de repos

`echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] `[`https://packages.grafana.com/oss/deb`](https://packages.grafana.com/oss/deb)` stable main" | sudo tee /etc/apt/sources.list.d/grafana.list`

<figure id="201dab02-9ae2-8195-a318-f9bbffa3ddd6" class="image">
<a href="image%2016.png"><img src="image%2016.png"
style="width:685.9375px" /></a>
</figure>

1.3: **Actualizamos paquetes y instalamos Grafana**

`sudo apt-get update`\
`sudo apt-get install -y grafana`\

<figure id="201dab02-9ae2-8147-b488-f550b7f2fd09" class="image">
<a href="image%2017.png"><img src="image%2017.png"
style="width:685.984375px" /></a>
</figure>

1.4: **Habilitamos e iniciamos el servicio**

`sudo systemctl start grafana-server`\
`sudo systemctl enable grafana-server`\

verificamos su funcionamiento con el comando

`sudo systemctl status grafana-server`

<figure id="201dab02-9ae2-8193-8e8e-c9c3854e4a42" class="image">
<a href="image%2018.png"><img src="image%2018.png"
style="width:685.96875px" /></a>
</figure>

1.5: **probamos ingresando a** **`http://<IP-SERVIDOR>:3000`**

por defecto las credenciales son:

::: indented
Usuario: admin

contraseña: admin (nos pedirá cambiarla)
:::

<figure id="201dab02-9ae2-81ae-9427-e0036f902f4b" class="image">
<a href="image%2019.png"><img src="image%2019.png"
style="width:685.96875px" /></a>
</figure>

luego de cambiar podremos entrar a home de grafana donde seguiremos con
el siguiente punto

<figure id="201dab02-9ae2-8195-bce1-df0f4c5b30fa" class="image">
<a href="image%2020.png"><img src="image%2020.png"
style="width:685.984375px" /></a>
</figure>

1.6: **información adicional sobre el servicio de grafana**

::: indented
  ----------------------------- ----------------------------------------------------------------------------------------------------
  Concepto                      Detalle / Comando
  **Servicio principal**        `grafana-server` (control con `systemctl start/stop/status grafana-server`)
  **Ruta del binario**          `/usr/sbin/grafana-server`
  **Configuración principal**   `/etc/grafana/grafana.ini`
  **Puerto por defecto**        `3000` (HTTP)
  **Panel Web**                 `http://localhost:3000`
  **Credenciales iniciales**    Usuario: `admin` / Contraseña: `admin` (se cambia al primer inicio)
  **Logs del servicio**         `journalctl -u grafana-server -f`
  **Directorio de datos**       `/var/lib/grafana/` (base de datos interna SQLite, dashboards, plugins, etc.)
  **Plugins instalados**        `/var/lib/grafana/plugins/`
  **Dashboards**                Se configuran desde la web o importan desde [Grafana Labs](https://grafana.com/grafana/dashboards)
  **Fuentes de datos**          Se añaden desde la interfaz web (`Configuration → Data Sources`)
  **Formato de dashboards**     JSON (exportable/importable)
  ----------------------------- ----------------------------------------------------------------------------------------------------
:::

2: **conexión de Grafana con prometheus**

1.  Ingresamos dentro de grafana a **\"Connections\"** → **\"Data
    Sources\"**

<!-- -->

2.  presionamos en **"Add data source"**

<!-- -->

3.  Seleccionamos **Prometheus** de la lista de fuentes y ponemos la URL
    correspondiente a nuestro servicio de prometheus, el resto de campos
    los podemos dejar así como están.

¿Dónde se guarda esta conexión?

Aunque se configura desde la interfaz, se guarda internamente en la base
de datos de Grafana (por defecto es SQLite en
`/var/lib/grafana/grafana.db`.

3.1: **Crearemos un panel simple personalizado con las tres métricas ya
escogidas**

::: indented
En el panel lateral, haz clic en el **+ (plus)** → **Dashboard**.

Clic en **\"Add a new panel\" o "Add Visualization"**.
:::

1.  Métrica `node_cpu_seconds_total` en el dashboard personalizado :
    - En el campo de búsqueda de métricas (parte inferior, en la opción
      Run Queries: Code), escribimos:

      `rate(node_cpu_seconds_total{mode="user"}[1m])`

    <!-- -->

    - Este mide el **uso de CPU en modo usuario** en un rango de 1
      minuto.

    <!-- -->

    - Automáticamente vas a ver una **gráfica**.

    <!-- -->

    - Podes cambiar el nombre del panel haciendo clic en el título
      (arriba a la izquierda del panel).

    <!-- -->

    - Clic en **\"Apply\"** (esquina superior derecha del panel).

<!-- -->

2.  Métrica `node_memory_MemAvailable_bytes` en el dashboard
    personalizado :
    - En la búsqueda de métrica, pegamos:

      `node_memory_MemAvailable_bytes`

    <!-- -->

    - Si queremos que sea más legible (por ejemplo en GB), vamos a la
      pestaña derecha **\"Field\" \> \"Unit\"** y seleccionamos:\
      \
      `Data → bytes (IEC)`

    <!-- -->

    - Podemos elegir visualización tipo:
      - **Time series** (gráfica de línea)

      <!-- -->

      - **Gauge** (medidor)

      <!-- -->

      - **Stat** (número grande)

    <!-- -->

    - Clic en **\"Apply\"** para guardar el panel.

<!-- -->

3.  Métrica `node_filesystem_avail_bytes` en el dashboard personalizado
    (con medidor de estado (gauge):
    - En la consulta, escribimos esta métrica:

      `node_filesystem_avail_bytes{mountpoint="/", fstype!~"tmpfs|overlay"}`

      Esto te da el **espacio realmente disponible** (no reservado) en
      el sistema de archivos raíz `/`, ignorando sistemas temporales o
      sobre montados.

    <!-- -->

    - En la parte inferior, seleccionamos la pestaña
      **\"Visualization\"** y elegimos **\"Gauge\"**

    <!-- -->

    - A la derecha, en el panel de configuración:
      - Podemos ajustar los **valores mínimos** y **máximos**. Por
        ejemplo, si sabes que tu disco tiene 20 GB, podemos poner:
        - Min: `0`

        <!-- -->

        - Max: `21474836480` (20 GB en bytes)

      <!-- -->

      - También podemos activar colores de estado:
        - Verde si \> 10 GB

        <!-- -->

        - Amarillo si entre 5 y 10 GB

        <!-- -->

        - Rojo si \< 5 GB (esto lo haces desde \"Thresholds\")

    <!-- -->

    - Clic en **\"Apply\"** para guardar el panel.

4\. Guardar el dashboard completo

::: indented
- En la parte superior del dashboard, clic en el **ícono de disquete 💾
  o \"Save dashboard\"**.

<!-- -->

- Ponemos un nombre, como por ejemplo:
  `Node Exporter - Métricas básicas`.

<!-- -->

- Confirmamos con **Save**.
:::

Y este seria el resultado final de un dashboard personalizado básico:

<figure id="201dab02-9ae2-8152-88da-f0f68e7403b1" class="image">
<a href="image%2021.png"><img src="image%2021.png"
style="width:709.96875px" /></a>
</figure>

3.2: **Selección de un dashboard de Grafana preconfigurado para
Prometheus y Node Exporter**

¿Qué es esto?

Grafana tiene una librería de dashboards listos para usar. Para
Prometheus + Node Exporter, hay uno supercompleto que visualiza todo:
CPU, RAM, red, disco, temperatura, etc.

Pasos para importar un dashboard desde la librería oficial:

1.  En el menú lateral izquierdo, hacemos clic en:

    **\"+\" → \"Import\"**.

<!-- -->

2.  En el campo **\"Import via grafana.com\"**, podemos usar el ID de
    los dashboard que encontremos en
    [`https://grafana.com/grafana/dashboards/`](https://grafana.com/grafana/dashboards/),
    en mi caso escogí el dashboard con ID` 1860`

<!-- -->

3.  Hacemos clic en **\"Load\"**.

<!-- -->

4.  En el siguiente paso, seleccionamos la **fuente de datos
    Prometheus** (la que configurada en el paso anterior).

<!-- -->

5.  Clic en **\"Import\"**.

Ahora podemos verificar que funcione

Una vez importado:

- Vas a ver un **dashboard lleno de paneles** (CPU por core, RAM,
  filesystem, carga promedio, temperatura si aplica, etc.).

<!-- -->

- Todos los datos deberían actualizarse en tiempo real.

<!-- -->

- Si algo aparece en gris o vacío, asegurarse de que:
  - El servicio de Prometheus está corriendo.

  <!-- -->

  - Node Exporter sigue exponiendo métricas en el puerto `9100`.

<figure id="201dab02-9ae2-81e4-b41e-f516b79479c0" class="image">
<a href="image%2022.png"><img src="image%2022.png"
style="width:709.96875px" /></a>
</figure>

------------------------------------------------------------------------
::::::::::::

[]{.sans style="font-size:14px;padding-top:2em"}
