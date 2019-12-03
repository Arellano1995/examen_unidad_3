## Examen de Cómputo distribuido Unidad 3
- Desarrollar una aplicación la cual utilice Django, Celery y Redis, esta aplicación tiene la funcionalidad de ajustar el tamaño de imagenes y guardarla en una ruta específica.
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
1. Crear una carpeta llamada image_parroter donde se localizará el proyecto
```
(venv) $ django-admin startproject image_parroter
(venv) $ cd image_parroter
(venv) $ python manage.py startapp thumbnailer
```
2. Para integrar Celery en este proyecto de Django, agrego un nuevo módulo 
```
 image_parroter / image_parrroter / celery.py
 ```
3. Dentro de este nuevo módulo de Python, importamos el paquete os y la clase Celery del paquete Celery.
```python
# image_parroter/image_parroter/celery.py

import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'image_parroter.settings')

celery_app = Celery('image_parroter')
celery_app.config_from_object('django.conf:settings', namespace='CELERY')
celery_app.autodiscover_tasks()
```
4. Ahora, en el módulo settings.py del proyecto, en la parte inferior, definimos una sección para la configuración de Celery y agregamos la configuración que de a continuación.
```python
# image_parroter/image_parroter/settings.py

... skipping to the bottom

# celery
CELERY_BROKER_URL = 'redis://localhost:6379'
CELERY_RESULT_BACKEND = 'redis://localhost:6379'
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TASK_SERIALIZER = 'json'
```
5. A continuación, asegurarse de que la aplicación de Celery previamente creada y configurada se inyecte en la aplicación Django cuando se ejecuta.
```python
# image_parroter/image_parroter/__init__.py

from .celery import celery_app

__all__ = ('celery_app',)

```
6. Agregamos un nuevo módulo llamado task.py dentro de la aplicación "thumbnailer".
```python
# image_parroter/thumbnailer/tasks.py

from celery import shared_task

@shared_task
def adding_task(x, y):
    return x + y
```
7. Por último, agregamos la aplicación de miniaturas a la lista de INSTALLED_APPS en el módulo settings.py del proyecto image_parroter.
```python
# image_parroter/image_parroter/settings.py

... skipping to the INSTALLED_APPS

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'thumbnailer.apps.ThumbnailerConfig',
    'widget_tweaks',
]
```
## Crear Image Thumbnails con Celery
1. En el modulo de tasks.py, importamos la clase Image del paquete PIL, luego agregamos una nueva tarea llamada make_thumbnails, que acepta una ruta de archivo de imagen y una lista de dimensiones de ancho y alto de 2 tuplas para crear miniaturas.
```python
# image_parroter/thumbnailer/tasks.py
import os
from zipfile import ZipFile

from celery import shared_task
from PIL import Image

from django.conf import settings

@shared_task
def make_thumbnails(file_path, thumbnails=[]):
    os.chdir(settings.IMAGES_DIR)
    path, file = os.path.split(file_path)
    file_name, ext = os.path.splitext(file)

    zip_file = f"{file_name}.zip"
    results = {'archive_path': f"{settings.MEDIA_URL}images/{zip_file}"}
    try:
        img = Image.open(file_path)
        zipper = ZipFile(zip_file, 'w')
        zipper.write(file)
        os.remove(file_path)
        for w, h in thumbnails:
            img_copy = img.copy()
            img_copy.thumbnail((w, h))
            thumbnail_file = f'{file_name}_{w}x{h}.{ext}'
            img_copy.save(thumbnail_file)
            zipper.write(thumbnail_file)
            os.remove(thumbnail_file)

        img.close()
        zipper.close()
    except IOError as e:
        print(e)

    return results
```
2. Agregamos al proyecto Django una ubicación MEDIA_ROOT donde puedan residir los archivos de imagen y los archivos zip  y especificamos MEDIA_URL donde se puede servir el contenido.
```python
# image_parroter/settings.py

... skipping down to the static files section

# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/2.2/howto/static-files/

STATIC_URL = '/static/'
MEDIA_URL = '/media/'

MEDIA_ROOT = os.path.abspath(os.path.join(BASE_DIR, 'media'))
IMAGES_DIR = os.path.join(MEDIA_ROOT, 'images')

if not os.path.exists(MEDIA_ROOT) or not os.path.exists(IMAGES_DIR):
    os.makedirs(IMAGES_DIR)
```
3. Dentro del módulo thumbnailer / views.py
```python
# thumbnailer/views.py

import os

from celery import current_app

from django import forms
from django.conf import settings
from django.http import JsonResponse
from django.shortcuts import render
from django.views import View

from .tasks import make_thumbnails

class FileUploadForm(forms.Form):
    image_file = forms.ImageField(required=True)

class HomeView(View):
    def get(self, request):
        form = FileUploadForm()
        return render(request, 'thumbnailer/home.html', { 'form': form })
    
    def post(self, request):
        form = FileUploadForm(request.POST, request.FILES)
        context = {}

        if form.is_valid():
            file_path = os.path.join(settings.IMAGES_DIR, request.FILES['image_file'].name)

            with open(file_path, 'wb+') as fp:
                for chunk in request.FILES['image_file']:
                    fp.write(chunk)

            task = make_thumbnails.delay(file_path, thumbnails=[(128, 128)])

            context['task_id'] = task.id
            context['task_status'] = task.status

            return render(request, 'thumbnailer/home.html', context)

        context['form'] = form

        return render(request, 'thumbnailer/home.html', context)


class TaskView(View):
    def get(self, request, task_id):
        task = current_app.AsyncResult(task_id)
        response_data = {'task_status': task.status, 'task_id': task.id}

        if task.status == 'SUCCESS':
            response_data['results'] = task.get()

        return JsonResponse(response_data)
```
4. Agregamos un módulo urls.py dentro de la aplicación de miniaturas y definimos las siguientes URL:
```python
# thumbnailer/urls.py

from django.urls import path

from . import views

urlpatterns = [
  path('', views.HomeView.as_view(), name='home'),
  path('task/<str:task_id>/', views.TaskView.as_view(), name='task'),
]
```
5. Luego, en la configuración de la URL principal del proyecto, incluimos las URL de nivel de aplicación, así como informarle sobre la URL de los medios
```python
# image_parroter/urls.py

from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('thumbnailer.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```
6. A continuación, creamos una vista de plantilla simple para que un usuario envíe un archivo de imagen, así como para verificar el estado de las tareas de make_thumbnails enviadas e iniciar una descarga de las miniaturas resultantes.
```
(venv) $ mkdir -p thumbnailer/templates/thumbnailer
```
7. Luego, dentro de este directorio de plantillas / miniaturas, agregamos una plantilla llamada home.html.
```html
<!-- templates/thumbnailer/home.html -->
{% load widget_tweaks %}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Thumbnailer</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.7.5/css/bulma.min.css">
  <script src="https://cdn.jsdelivr.net/npm/vue"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/axios/0.18.0/axios.min.js"></script>
  <script defer src="https://use.fontawesome.com/releases/v5.0.7/js/all.js"></script>
</head>
<body>
  <nav class="navbar" role="navigation" aria-label="main navigation">
    <div class="navbar-brand">
      <a class="navbar-item" href="/">
        Thumbnailer
      </a>
    </div>
  </nav>
  <section class="hero is-primary is-fullheight-with-navbar">
    <div class="hero-body">
      <div class="container">
        <h1 class="title is-size-1 has-text-centered">Thumbnail Generator</h1>
        <p class="subtitle has-text-centered" id="progress-title"></p>
        <div class="columns is-centered">
          <div class="column is-8">
            <form action="{% url 'home' %}" method="POST" enctype="multipart/form-data">
              {% csrf_token %}
              <div class="file is-large has-name">
                <label class="file-label">
                  {{ form.image_file|add_class:"file-input" }}
                  <span class="file-cta">
                    <span class="file-icon"><i class="fas fa-upload"></i></span>
                    <span class="file-label">Browse image</span>
                  </span>
                  <span id="file-name" class="file-name" 
                    style="background-color: white; color: black; min-width: 450px;">
                  </span>
                </label>
                <input class="button is-link is-large" type="submit" value="Submit">
              </div>
              
            </form>
          </div>
        </div>
      </div>
    </div>
  </section>
  <script>
  var file = document.getElementById('{{form.image_file.id_for_label}}');
  file.onchange = function() {
    if(file.files.length > 0) {
      document.getElementById('file-name').innerHTML = file.files[0].name;
    }
  };
  </script>

  {% if task_id %}
  <script>
  var taskUrl = "{% url 'task' task_id=task_id %}";
  var dots = 1;
  var progressTitle = document.getElementById('progress-title');
  updateProgressTitle();
  var timer = setInterval(function() {
    updateProgressTitle();
    axios.get(taskUrl)
      .then(function(response){
        var taskStatus = response.data.task_status
        if (taskStatus === 'SUCCESS') {
          clearTimer('Check downloads for results');
          var url = window.location.protocol + '//' + window.location.host + response.data.results.archive_path;
          var a = document.createElement("a");
          a.target = '_BLANK';
          document.body.appendChild(a);
          a.style = "display: none";
          a.href = url;
          a.download = 'results.zip';
          a.click();
          document.body.removeChild(a);
        } else if (taskStatus === 'FAILURE') {
          clearTimer('An error occurred');
        }
      })
      .catch(function(err){
        console.log('err', err);
        clearTimer('An error occurred');
      });
  }, 800);

  function updateProgressTitle() {
    dots++;
    if (dots > 3) {
      dots = 1;
    }
    progressTitle.innerHTML = 'processing images ';
    for (var i = 0; i < dots; i++) {
      progressTitle.innerHTML += '.';
    }
  }
  function clearTimer(message) {
    clearInterval(timer);
    progressTitle.innerHTML = message;
  }
  </script> 
  {% endif %}
</body>
</html>
```

### Para probarlo usamos tres terminales

- Iniciar redis-server en su propia ventana de terminal
```
~/redis-5.0.6/src$ redis-server
```
- Iniciar el trabajador de Celery en su propia ventana de terminal con el entorno virtual de python activado y en el mismo directorio que el script manage.py
```
(venv)~/image_parroter/image_parroter$ celery worker -A image_parroter --loglevel=info
```
- Ejecutar el servidor Django, nuevamente en su propia ventana de terminal con el entorno virtual de python activado y en el mismo directorio que el script manage.py
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

