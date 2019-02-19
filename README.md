# (Deploy) Django / Ubuntu 18.04.1 LTS

Este documento pretende ser un **tutorial** completo de cómo llevar a cabo la subida o deploy de un proyecto Django a un ambiente real de producción. En este ejemplo estamos utilizando un servidor ubuntu hospedado en Amazon Web Services, pero se puede utilizar cualquier servicio de hospedaje como DigitalOcean, Heroku o PythonAnywhere.

>
En el servidor configuramos `nginx` y `supervisor`, y en el proyecto Django configuramos `gunicorn`. Estas 3 tecnologías sirven para poder levantar y tener corriendo nuestro proyecto.

>
* [Nginx](https://en.wikipedia.org/wiki/Nginx) es un servidor web que también se puede utilizar como proxy inverso, equilibrador de carga, proxy de correo y caché HTTP.
* [Supervisor](http://supervisord.org/introduction.html) es un sistema cliente/servidor que permite a sus usuarios controlar varios procesos en sistemas operativos similares a UNIX. 
* [Gunicorn](https://gunicorn.org/) es un Servidor HTTP de Python WSGI para UNIX. El servidor Gunicorn es ampliamente compatible con varios marcos web, simplemente implementado, ligero en cuanto a recursos del servidor y bastante rápido.

***
 
#### Ambiente de trabajo

* Server: `t3.nano (EC2 AWS) - Ubuntu 18.04.1 LTS`
* Python: `3.7.1 (forzar instalación)`
* Django: `2.1.5`

#### Datos de prueba

* User en el servidor: `ubuntu`
* App de Supervisor: `WEB_PROJECT_DJANGO`
* Repositorio GitHub: `NuestroRepositorio`

#### Estructura de carpetas en el Servidor

![Img - Estructura de carpetas](https://raw.githubusercontent.com/antonioxtasis/DeployDjangoProduction-ES/master/imgs/estructura-carpetas.png)


## Paso 1: LogIn en servidor

**Permisos a llave SSH**

Dar permisos al archivo `.pem` que descargaste en tu ordenador

```
$ chmod 400 /ruta/en/tu/ordenador/YOUR_PRIVATE_KEY_FILE.pem
```

**Login SSH**

Conectarse desde una ventana de Terminal al servidor de AWS

```
$ ssh -o TCPKeepAlive=yes -o ServerAliveInterval=100 -i /ruta/en/tu/ordenador/PRIVATE_KEY_FILE.pem ubuntu@example.com
```

## Paso 2: Crear directorio del proyecto
**Crear directorio en el que trabajaremos y entrar**

Este directorio es el que utilizaremos como app en Supervisor

```
$ /home/ubuntu$ mkdir WEB_PROJECT_DJANGO
$ cd WEB_PROJECT_DJANGO
```

## Paso 3: Instalar paquetes necesarios
**Actualizar caché de paquetes (descarga la lista de actualizaciones disponibles)**

```
$ sudo apt update
$ sudo apt-get update
```

**Instala las actualizaciones disponibles**

```
$ sudo apt-get upgrade
```

**Instalar paquetes necesarios**

```
$ sudo apt-get install nginx supervisor git
```


## Paso 4: Python3, Pip3 y VirtualEnviroment
**Instalar Python3**

```
$ sudo apt install python3.7-minimal
$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1 
$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.7 2
$ sudo update-alternatives --config python3
```

**Instalar pip3**

```
$ sudo apt-get install python3-pip
```

**Instalar virtual environment**

```
$ sudo pip3 install virtualenv
```

**Crear entorno virtual llamado _`venv`_**

```
/WEB_PROJECT_DJANGO/$ virtualenv --python=python3.7 venv
```

**Activar entorno virtual**

```
/WEB_PROJECT_DJANGO$ source venv/bin/activate
(venv)$ …

Para desactivarlo
(venv)/WEB_PROJECT_DJANGO$ deactivate
$ …
```

**Dar permisos a la carpeta `/venv`** 

```
$ sudo chown -R $USER venv/
```

## Paso 5: Instalar MySQL

```
$ sudo apt-get install mysql-server
```

**Para evitar problemas con Python3.7 (Error: missing mysql_config)**

```
$ sudo apt-get install python3.7-dev libmysqlclient-dev
$ sudo apt-get install -y python3-mysqldb
```

**Configurar MySQL**

```
$ sudo mysql_secure_installation
```

## Paso 6: GitHub (descargar nuestro proyecto)
Para este paso es necesario que tengas tu repositorio en GitHub

**Clonar el repositorio**

```
(venv)/WEB_PROJECT_DJANGO$ git clone https://github.com/usuario-github/NuestroRepositorio.git
```

**Instalar dependencias del proyecto Django**

Nuestro archivo `requirements.txt` debe de por lo menos tener estas dependencias

![Img - Estructura de carpetas](https://raw.githubusercontent.com/antonioxtasis/DeployDjangoProduction-ES/master/imgs/requirements.png)

```
(venv)/WEB_PROJECT_DJANGO/NuestroRepositorio/web_project$ pip3 install --no-cache-dir -r requirements.txt
```


## Paso 7: Gunicorn (configuración)

**Crear dentro de la carpeta `/venv` las subcarpetas `/run` y `/logs`**

```
/WEB_PROJECT_DJANGO/venv/$ mkdir run        	<-- sin escribir sudo
/WEB_PROJECT_DJANGO/venv/$ sudo mkdir logs
```

**Crear script llamado `start.sh` dentro de `/bin`**

```
/WEB_PROJECT_DJANGO/venv/$ cd bin
/WEB_PROJECT_DJANGO/venv/bin/$ sudo nano start.sh
```

```php
#!/bin/bash

NAME="WEB_PROJECT_DJANGO" # Nombre dela aplicación
DJANGODIR=/home/ubuntu/WEB_PROJECT_DJANGO/NuestroRepositorio/web_project # Ruta de la carpeta (nuestro proyecto de Django)
SOCKFILE=/home/ubuntu/WEB_PROJECT_DJANGO/venv/run/gunicorn.sock # Ruta donde se creará el archivo de socket de unix para comunicarnos
USER=ubuntu # Usuario con el que vamos a correr la app
GROUP=ubuntu # Grupo con el que se va a correr la app
NUM_WORKERS=3 # Número de workers quese van a utilizar para correr la aplicación
DJANGO_SETTINGS_MODULE=web_project.settings # ruta de los settings
DJANGO_WSGI_MODULE=web_project.wsgi # Nombre del módulo wsgi

echo "Starting $NAME as `whoami`"

# Activar el entorno virtual
cd $DJANGODIR
source ../../venv/bin/activate 
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DJANGODIR:$PYTHONPATH

# Crear la carpeta run si no existe para guardar el socket linux
RUNDIR=$(dirname $SOCKFILE)
test -d $RUNDIR || mkdir -p $RUNDIR

# Iniciar la aplicación django por medio de Gunicorn
# Los programas que son corridos por Supervisor no se deben demonizar (no usar --daemon)
exec ../../venv/bin/gunicorn ${DJANGO_WSGI_MODULE}:application \
  --name $NAME \
  --workers $NUM_WORKERS \
  --user=$USER --group=$GROUP \
  --bind=unix:$SOCKFILE \
```


**Permitir la ejecución del archivo**

Este script nos permite levantar nuestra aplicación django sin usar `$ manage.py runserver`

```
/WEB_PROJECT_DJANGO/bin/$ sudo chmod +x start.sh
```

## Paso 8: Supervisor (configuración)

**Supervisor nos ayuda a mantener nuestra aplicación Django corriendo. La configuración consiste en crear el script `myapp.conf` dentro de la carpeta `/etc/supervisor/conf.d/`**

```
$ sudo touch /etc/supervisor/conf.d/myapp.conf
$ sudo nano /etc/supervisor/conf.d/myapp.conf
```

```php
[program:WEB_PROJECT_DJANGO]
command = /home/ubuntu/WEB_PROJECT_DJANGO/venv/bin/start.sh ; Comando para correr la app
user = ubuntu ; Usuario con el que vamos a correr la app
stdout_logfile = /home/ubuntu/WEB_PROJECT_DJANGO/venv/logs/gunicorn_supervisor.log ; Donde vamos a guardar los logs
redirect_stderr = true ; Guardar los errores en el log
```


**Indicar a Supervisor que lea las nuevas configuraciones y que se actualice**

```
$ sudo supervisorctl reread
$ sudo supervisorctl update
```

```
Comandos para realizar acciones en la “aplicación”
$ sudo supervisorctl status WEB_PROJECT_DJANGO      <-- ver status                      
$ sudo supervisorctl stop WEB_PROJECT_DJANGO        <-- pararla
$ sudo supervisorctl restart WEB_PROJECT_DJANGO     <-- reiniciarla
$ sudo supervisorctl start WEB_PROJECT_DJANGO       <-- correrla
```


## Paso 9: NGINX (configuración para desplegar el sitio web)

**Configurar Web Server (NGINX) para conectarse con el Socket que está corriendo con Gunicorn.**

NGINX cuenta con 2 carpetas:

- `/etc/nginx/sites-available/` <-- aquí se almacenan las configuraciones de los sitios disponibles
- `/etc/nginx/sites-enabled/` <-- aquí se almacenan los sitios activos

**Creamos un archivo llamado `myapp.conf`**

```
$ sudo nano /etc/nginx/sites-available/myapp.conf
```

Dentro del archivo escribrimos lo siguiente de acuerdo a nuestra IP o dominio web


```php
upstream myapp_server {
  server unix:/home/ubuntu/WEB_PROJECT_DJANGO/venv/run/gunicorn.sock fail_timeout=0;
}

server {

  listen 80;
  server_name example.com www.example.com;

  client_max_body_size 4G;

  access_log /home/ubuntu/WEB_PROJECT_DJANGO/venv/logs/nginx-access.log;
  error_log /home/ubuntu/WEB_PROJECT_DJANGO/venv/logs/nginx-error.log;

  location /static/ {
    alias /home/ubuntu/WEB_PROJECT_DJANGO/NuestroRepositorio/web_project/static/;
  }

  location /media/ {
    alias /home/ubuntu/WEB_PROJECT_DJANGO/NuestroRepositorio/web_project/media/;
  }
  
  location / {
          # an HTTP header important enough to have its own Wikipedia entry:
          # http://en.wikipedia.org/wiki/X-Forwarded-For
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

          # enable this if and only if you use HTTPS, this helps Rack
          # set the proper protocol for doing redirects:
          # proxy_set_header X-Forwarded-Proto https;

          # pass the Host: header from the client right along so redirects
          # can be set properly within the Rack application
          proxy_set_header Host $http_host;

          # we don't want nginx trying to do something clever with
          # redirects, we set the Host: header above already.
          proxy_redirect off;
          
          # set "proxy_buffering off" *only* for Rainbows! when doing
          # Comet/long-poll stuff. It's also safe to set if you're
          # using only serving fast clients with Unicorn + nginx.
          # Otherwise you _want_ nginx to buffer responses to slow
          # clients, really.
          # proxy_buffering off;

          # Try to serve static files from nginx, no point in making an
          # *application* server like Unicorn/Rainbows! serve static files.
          if (!-f $request_filename) {
                  proxy_pass http://myapp_server;
                  break;
          }
  }
  
  # Error pages
  error_page 500 502 503 504 /500.html;
  location = /500.html {
    root /home/ubuntu/WEB_PROJECT_DJANGO/NuestroRepositorio/web_project/static/;
  }
}

```

**Dar de alta la configuración de nuestro sitio web**

```
$ sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/myapp.conf
```

**Reiniciar nginx**

```
$ sudo service nginx restart
```

### ¡Listo, ya tenemos nuestro proyecto en producción!

😎





***

### Extra 1: Actualizar código del proyecto desde GitHub

Una vez que ya se tiene corriendo correctamente el proyecto en nuestro servidor,
llega el momento de actualizar el código de nuestro proyecto.


**Actualizar código**

- Eliminar primero cualquier cambio que hayamos hecho en el proyecto

```
WEB_PROJECT_DJANGO/NuestroRepositorio$ git reset --hard HEAD
```

- Jalar desde GitHub los cambios 

```
WEB_PROJECT_DJANGO/NuestroRepositorio$ git pull origin master
```

**Reiniciar la Aplicación con Supervisor**

```
$ sudo supervisorctl restart WEB_PROJECT_DJANGO
``` 


***

### Extra 2: Comandos útiles

Resetear nuestro virtualenv (venv) para asegurarnos que requirements.txt se actualice

```
$ sudo virtualenv --clear venv
```

Eliminar directorio

```
$ sudo rm -r folderName
```

Mover directorio

```
$ sudo mv fromPath/ toPath/
```

Si hay varias versiones de python (ver todas las versiones):

```
$ apt list --installed | grep pytho
```

Remover completamente los paquetes python

```
$ sudo apt purge python-pip python-dev
```

Remover todos los paquestes instalados por pip

```
$ pip uninstall -r requirements.txt -y      (método 1)
$ pip freeze | xargs pip uninstall -y       (método 2)
```







## Referencias
* [Platzi - ¿Cómo llevar Django a producción?](https://platzi.com/blog/llevar-django-a-produccion/)

* [Linux For Windows Mac User - How to Install Python 3.7 and make default in Ubuntu 18.04 LTS](https://www.youtube.com/watch?v=a4XDW3czH0c)

* [WebForefront - Django settings.py for the real world](https://www.webforefront.com/django/configuredjangosettings.html)