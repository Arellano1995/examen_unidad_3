# examen_unidad_3
Examen de computo distribuido unidad 3
Desarrollar una aplicación la cual utilice Django, Celery y Redis, esta aplicación tiene la funcionalidad de ajustar el tamaño de imagenes y guardarla en una ruta específica.
# Contenido
- [Instalación de Redis](#primero-instalar-redis)
- [Instalacion de Python Virtual Env](#instalar-python-virtual-env-y-dependencias)
- [Configuración del proyecto Django](#configurar-el-projecto-de-django)
- [Configuración de Celery](#crear-image-thumbnails-con-celery)
- [Referencia](#referencia)
- [Autores](#autores)
- [License](#license)

## Primero instalar Redis 
Descargar, descomprimir y compilar Redis
```
$ wget http://download.redis.io/releases/redis-5.0.7.tar.gz
$ tar xzf redis-5.0.7.tar.gz
$ cd redis-5.0.7
$ make
```

## Instalar Python virtual env y dependencias	
Se crea un directorio llamado image_parroter donde se creará el entorno virtual	
```
$ mkdir image_parroter
$ cd image_parroter
$ python3 -m venv venv (*)
```
Al tener Python 3.7 y aparecia un error al crear el entorno virtual venv. Para evitar esto, use el comando virtualenv en su lugar.
```
$ sudo apt-get install python-virtualenv
$ virtualenv --python=python3.7 venv
$ source venv/bin/activate
```

Después de activar venv, se instalan los paquetes de Python 
```
(venv) $ pip3 install Django Celery redis Pillow django-widget-tweaks
(venv) $ pip3 freeze > requirements.txt
```
## Configurar el projecto de Django
Crear una carpeta llamada image_parroter donde se localizará el proyecto
```
(venv) $ django-admin startproject image_parroter
(venv) $ cd image_parroter
(venv) $ python manage.py startapp thumbnailer
```
## Crear Image Thumbnails con Celery
Iniciar redis-server en su propia ventana de terminal
```
~/redis-5.0.6/src$ redis-server
```
Iniciar el trabajador de Celery en su propia ventana de terminal con el entorno virtual de python activado y en el mismo directorio que el script manage.py
```
(venv)~/image_parroter/image_parroter$ celery worker -A image_parroter --loglevel=info
```
Ejecutar el servidor Django, nuevamente en su propia ventana de terminal con el entorno virtual de python activado y en el mismo directorio que el script manage.py
```
(venv) ~/image_parroter/image_parroter$ python manage.py runserver
```
En caso de error:
You have 17 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Ejecute 'python manage.py migrate' para aplicarlos.
```
python manage.py migrate
```
## Referencia
<a href="https://stackabuse.com/asynchronous-tasks-in-django-with-redis-and-celery/" target="_blank">**Stackabuse**</a>

## Autores
* **Jose Arellano** - [Github](https://github.com/Arellano1995)
* **Lilia Rosales** - [Github](https://github.com/liliarsis)

