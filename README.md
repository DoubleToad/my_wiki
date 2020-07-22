# Настройка сервера

Ubuntu 19.04  
Nginx + gunicorn + django

Создаем ссылку *python* на *python3*  
`sudo ln -s /usr/bin/python3.7 /usr/bin/python`

## Установка пакетов
`sudo apt-get install python3-distutils python3-dev python3-pip python3-venv libpq-dev nginx curl postgresql postgresql-contrib`

## Postgres
```
sudo -u postgres psql
create database testweb;
create user testwebuser with password 'testwebpass';
alter role testwebuser set timezone to 'Europe/Moscow';
alter role testwebuser set client_encoding to 'UTF8';
alter role testwebuser set default_transaction_isolation to 'read committed';
grant all privileges on database testweb to testwebuser;
\q
```

---

> **Это уже есть в проекте. В этом блоке описано как оно создавалось**

## Virtual Environment
```
mkdir ~/testweb
cd ~/testweb
python -m venv web_env

```

## Django + gunicorn

Активируем venv  
`source ~/testweb/web_env/bin/activate`

Pip upgrade  
`pip install --upgrade pip`

Устанавливаем зависимости  
`pip install -r requirements.pip`

Или  
`pip install django gunicorn psycopg2-binary`

## Создаем django проект website
`django-admin.py startproject website ~/testweb`

## Настройка проекта
`nano ~/testweb/website/settings.py`

```python
ALLOWED_HOSTS = ['my.host.com', 'localhost']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'testweb',
        'USER': 'testwebuser',
        'PASSWORD': 'testwebpass',
        'HOST': 'localhost',
        'PORT': '',
    }
}

LANGUAGE_CODE = 'ru-ru'
TIME_ZONE = 'Europe/Moscow'

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

`~/testweb/manage.py makemigrations`  
`~/testweb/manage.py migrate`

Создаем административного пользователя  
`~/testweb/manage.py createsuperuser`

Собираем статику  
`~/testweb/manage.py collectstatic`

---

## Настройка gunicorn

Проверяем, что gunicorn работает
```
cd ~/testweb
gunicorn --bind 0.0.0.0:8000 website.wsgi
```

Выходим из *venv* и оздаем сокет  
`deactivate`  
`sudo nano /etc/systemd/system/gunicorn.socket`

Внутри *gunicorn.socket*
```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

Создаем службу systemd для gunicorn  
`sudo nano /etc/systemd/system/gunicorn.service`
```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=mdolmanov
Group=www-data
WorkingDirectory=/home/mdolmanov/testweb
ExecStart=/home/mdolmanov/testweb/web_env/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          website.wsgi:application

[Install]
WantedBy=multi-user.target
```

Стартуем и включаем *gunicorn.socket*
```
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

Проверяем сокет  
`sudo systemctl status gunicorn.socket`

Проверяем наличие *gunicorn.sock*  
`file /run/gunicorn.sock`

Если что то пошло не туда, смотрим логи  
`sudo journalctl -u gunicorn.socket`

Проверяем службу gunicorn (она не активна, пока нет подключений)  
`sudo systemctl status gunicorn`

Делаем подключение к сокету (должны увидеть html код)  
`curl --unix-socket /run/gunicorn.sock localhost`

Проверяем, что служба запустилась  
`sudo systemctl status gunicorn`

Если что то пошло не туда, смотрим логи  
`sudo journalctl -u gunicorn`

---

## Настройка Nginx

Создаем сайт  
`sudo nano /etc/nginx/sites-available/website.conf`
```
server {
    listen 80;
    server_name server.name.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/mdolmanov/testweb;
    }

location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

Активируем сайт, создав линк  
`sudo ln -s /etc/nginx/sites-available/website.conf /etc/nginx/sites-enabled`

Тестим конфигурацию сайта  
`sudo nginx -t`

Рестартуем Nginx  
`sudo systemctl restart nginx`

Заходим на сайт по *server_name* адресу

---

## Git

Устанавливаем git  
`sudo apt-get install git`

Настройки пользователя  
`git config --global user.name "User Name"`  
`git config --global user.email mail@example.com`

Выбираем текстовый редактор  
`git config --global core.editor nano`

Задаем алиасы  
`nano ~/.gitconfig`
```
[alias]
  co = checkout
  ci = commit
  st = status
  br = branch
  hist = log --pretty=format:\"%h %ad | %s%d [%an]\" --graph --date=short
  type = cat-file -t
  dump = cat-file -p
```

Создание репозитория в существующем каталоге
```
cd ~/testweb
git init
```

Добавляем файлы под версионный контроль
```
git add *
git commit -m 'initial project version'
```

Удаляем файлы из контроля  
`git rm --cached readme.txt`

Чтобы забрать репозеторий отсюда  
`git clone https://doubletoad@bitbucket.org/doubletoad/testweb.git`
