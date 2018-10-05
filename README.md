# Instalación de proyecto django usando proxy reverso de nginx

## Instalación de  virtualenv

```sh
pip install --upgrade virtualenv
```

## Creación del entorno

```sh
virtualenv env
```

## Activación de entorno

```sh
source env/bin/activate
```

## Creación del proyecto django

```sh
django-admin startproject mysitex
```

## Instalación de gunicorn en django

```sh
pip install gunicorn
```

## Probamos que el gunicorn funcione como servidor de la aplicación

```sh
gunicorn mysitex.wsgi
```

- debería habilitarse el proyecto para ser accedido vía localhost:8000

## Creación de archivo gunicorn.service

```sh
sudo nano /etc/systemd/system/gunicorn.service
```

### Contenido del archivo

```sh
[Unit]
Description=gunicorn daemon
After=network.target
[Service]
#aquí se define el nombre del usuario
User=ubuntu
Group=www-data
#aquí se define en donde está ubicado el proyecto
WorkingDirectory=/home/ubuntu/mysitex
#aquí se define en donde está ubicada la carpeta del ambiente de python
#y que está instalado gunicorn y donde estará ubicado el archivo 
#que usará nginx para hacer la publicación
ExecStart=/home/ubuntu/mysitex/env/bin/gunicorn --workers 3 --bind unix:/home/ubuntu/mysitex/mysitex.sock mysitex.wsgi:application

[Install]
WantedBy=multi-user.target
```

### Se inicia el servicio

```sh
sudo systemctl start gunicorn
```

### Se habilita el servicio

```sh
sudo systemctl enable gunicorn
```

### Se revisa que el estado sea running

```sh
sudo systemctl status gunicorn
```

### El resultado es debe ser:

```sh
 gunicorn.service - gunicorn daemon
   Loaded: loaded (/etc/systemd/system/gunicorn.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-10-05 10:49:12 -05; 54min ago
 Main PID: 2118 (gunicorn)
    Tasks: 4
   Memory: 55.7M
      CPU: 1.957s
   CGroup: /system.slice/gunicorn.service
           ├─2118 /home/ubuntu/mysitex/env/bin/python2 /home/ubuntu/mysitex/env/bin/gunicorn --workers 3 --bind unix:/home/ubuntu           ├─2125 /home/ubuntu/mysitex/env/bin/python2 /home/ubuntu/mysitex/env/bin/gunicorn --workers 3 --bind unix:/home/ubuntu           ├─2127 /home/ubuntu/mysitex/env/bin/python2 /home/ubuntu/mysitex/env/bin/gunicorn --workers 3 --bind unix:/home/ubuntu           └─2129 /home/ubuntu/mysitex/env/bin/python2 /home/ubuntu/mysitex/env/bin/gunicorn --workers 3 --bind unix:/home/ubuntu
Oct 05 10:49:12 zeus gunicorn[2118]: [2018-10-05 10:49:12 +0000] [2127] [INFO] Booting worker with pid: 2127
```

## Instalación de nginx

```sh
sudo apt-get install nginx
```

### Configuración de Nginx

```sh
nano /etc/nginx/sites-available/default
```

### Contenido del archivo default

```sh
server {
    listen 80;
    server_name 127.0.0.1;

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    # aquí se le configura la dirección de los archivos
    # estáticos a los que accederá la aplicación de django
    location /static/ {
        root /home/ubuntu/mysitex;
    }

    location / {
        include proxy_params;
        # aquí se le configura la ruta en la cual está ubicado el archivo sock que se comunicará con nginx para publicar la aplicación
        proxy_pass http://unix:/home/ubuntu/mysitex/mysitex.sock;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        #proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        error_page 404 /var/www/html/http_errors/404.html;
        error_page 502 /var/www/html/http_errors/502.html;
    }
}
```

### Se reinicia nginx

```sh
sudo systemctl restart nginx
```

### Se valida el estado del servicio de nginx

```sh
sudo systemctl status nginx
```

### Debe responder algo así

```sh
nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-10-05 11:10:24 -05; 35min ago
  Process: 2636 ExecStop=/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid (code=exited, status=2)
  Process: 2646 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 2641 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 2649 (nginx)
    Tasks: 2
   Memory: 1.2M
      CPU: 26ms
   CGroup: /system.slice/nginx.service
           ├─2649 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
           └─2650 nginx: worker process

```

## Acceder vía localhost, debería responder django a la petición.