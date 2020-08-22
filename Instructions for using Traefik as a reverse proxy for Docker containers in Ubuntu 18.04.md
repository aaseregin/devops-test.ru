# Введение.
Для запуска нескольких приложений на одном хосте Docker нужно настроить обратный прокси, чтобы оставить открытыми только порты `80` и `443`.
Traefik - обратный прокси с поддержкой Docker, имеет собственную панель мониторинга. В данной инструкции Traefik будет использоваться для перенаправления запросов двух разных контейнеров NGINX. Также traefik будет настроен для обслуживание соединений через HTTPS с помощью Let's Encrypt.

Начальные условия:
- Сервер Ubuntu, развернутый на облачной платформе. Например, [Яндекс.Облако](https://cloud.yandex.ru/)
- Docker, установленный согласно официальной инструкции: <https://docs.docker.com/engine/install/ubuntu/>
- Docker Compose, установленный согласно официальной инструкции: <https://docs.docker.com/compose/install/>
- Домен и три записи A, test1, test2, и monitor, каждая из которых указывает на IP-адрес вашего сервера.

Для начала настроим шифрованный пароль для доступа к информационной панели мониторинга с помощью htpasswd.
Устанавливаем пакет:
```sh
$ sudo apt-get install apache2-utils
```
Генерируем пароль:
Вместо `secure_password` вставьте свой пароль
```sh
$ htpasswd -nb admin secure_password
```
Программа выдаст результат:
`admin:$apr1$CCrW2yx7$M49EG0SKai3t5G8N..g631`

Используйте этот результат в файле конфигурации Traefik для настройки базовой аутентификации HTTP для проверки состояния Traefik и информационной панели мониторинга. Скопируйте всю строку результатов, чтобы ее можно было вставить.

Создаем папку проекта `devops-test`
Внутри папки с проектом создаем 3 подпапки:
`data` - для хранения файлов конфигурации traefik
`site1` и `site2` - для хранения конфигураций наших приложений на базу nginx

В папке `data` создаем пустой файл, где будет храниться информация Let’s Encrypt. Мы передадим ее в контейнер, чтобы Traefik мог ее использовать:
```sh
$ touch acme.json
```
Traefik сможет использовать этот файл, только если пользователь root внутри контейнера будет иметь уникальный доступ к этому файлу для чтения и записи. Для этого нам нужно будет заблокировать разрешения файла acme.json, чтобы права записи и чтения были только у владельца файла.
```sh
$ chmod 600 acme.json
```
После передачи файла в Docker владелец автоматически сменяется на пользователя root внутри контейнера.

Также в папке `data` создаём файл `traefik.toml`
```sh
$ vim traefik.toml
```yml
#Добавляем две точки входа с именами http и https, которые будут по умолчанию доступны всем серверным компонентам
defaultEntryPoints = ["http", "https"]

# Настраиваем провайдер api, который дает доступ к интерфейсу информационной панели. Здесь вам нужно вставить результат выполнения команды htpasswd:
# В своей конфигурации я использую нестандартный порт 9090, если вы используете файерволл, не забудьте добавить его в список разрешенных
[entryPoints]
  [entryPoints.monitor]
    address = ":9090"
    [entryPoints.monitor.auth]
      [entryPoints.monitor.auth.basic]
        users = ["admin:$apr1$CCrW2yx7$M49EG0SKai3t5G8N..g631"]

#Настраиваем точки входа "http", "https"
[entryPoints.http]
    address = ":80"
      [entryPoints.http.redirect]
        entryPoint = "https"
[entryPoints.https]
    address = ":443"
      [entryPoints.https.tls]

#включаем api
[api]
entrypoint="monitor"
dashboard = true
debug = true

#Раздел для настройки поддержки сертификата Let’s Encrypt для Traefik, пропишите в него свой e-mail
[acme]
email = "a.a.seregin@yandex.ru"
storage = "acme.json"
entryPoint = "https"
onHostRule = true
  [acme.httpChallenge]
  entryPoint = "http"

#Провайдер docker позволяет Traefik выступать в качестве прокси для контейнеров Docker.
#Данные настройки позволяют отслеживать новые контейнеры в сети web (которую мы вскоре создадим) и для предоставляют доступ к ним как к субдоменам домена devops-test.ru.
[docker]
domain = "devops-test.ru"
watch = true
network = "web"
```
Далее, переходим в папку `~/devops-test/site1` и  создаем в ней 3 подпапки:
`hosts` - папка для хранения конфигурации nginx
`logs` - папка для логов
`www` - папка для файлов сайта.

В папке `hosts` создаем файл `default.conf` следующего содержания:
```conf
server {
	listen 80;

	server_name site1.devops-test.ru www.site1.devops-test.ru;
	access_log /var/log/nginx/site1.devops-test.ru.access.log main;
	root   /var/www/;

	location / {
		index  index.html index.htm index.php;
	}
	
}
```
Переходим в папку `www`. Создаем файл `index.html` с текстом `Welcome to site 1`
```sh
$ echo "Welcome to site 1" > index.html
```
Переходим в папку `~/devop-test/site2` и проделываем все те же манипуляции, заменяя текст `site1` на `site2` во всех конфигурациях.

Возвращаемся в корень проекта ~/devops-test и создаем файл `docker-compose.yml`. Используйте имена своих поддоменов вместо <https://monitor.devops-test.ru>, <https://site1.devops-test.ru>, <https://site2.devops-test.ru>.
```yml
#Docker Compose версии 3
version: "3"

networks:
  web:
    external: true
  internal:
    external: false

services:
  traefik:
    image: traefik:1.7.2-alpine 
    ports:
      - 80:80
      - 443:443
  
    volumes:
      - /etc/localtime:/etc/localtime
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data/traefik.toml:/traefik.toml
      - ./data/acme.json:/acme.json
    labels:
      - traefik.enable=true
      - traefik.frontend.rule=Host:monitor.devops-test.ru
      - traefik.port=9090
    networks:
      - internal
      - web

  site1:
    image: nginx:alpine
    restart: always

    labels:
      - traefik.backend=site1
      - traefik.frontend.rule=Host:site1.devops-test.ru
      - traefik.docker.network=web
      - traefik.port=80

    volumes:
      - ./site1/hosts:/etc/nginx/conf.d
      - ./site1/www:/var/www
      - ./site1/logs:/var/log/nginx
    networks:
      - internal
      - web
    depends_on:
      - traefik

  site2:
    image: nginx:alpine
    restart: always

    labels:
      - traefik.backend=site2
      - traefik.frontend.rule=Host:site2.devops-test.ru
      - traefik.docker.network=web
      - traefik.port=80

    volumes:
      - ./site2/hosts:/etc/nginx/conf.d
      - ./site2/www:/var/www
      - ./site2/logs:/var/log/nginx
    networks:
      - internal
      - web
    depends_on:
      - traefik
```
Создаём сеть web:
```sh
$ docker network create web
```
Всё готово. Запускаем контейнеры с помощью docker-compose:
```sh
$ docker-compose up -d
```
Для запуска набора контейнеров в автоматическом режиме, сразу после старта сервера, необходимо создать службу в директории: `/etc/systemd/system/docker-compose-app.service`
```
[Unit]
Description=Docker Compose Application Service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/aseregin/devops-test
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```
Включить службу для запуска автоматически:
```sh
$ systemctl enable docker-compose-app
```

