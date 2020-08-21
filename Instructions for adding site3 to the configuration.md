Для добавления site3 в текущую конфигурацию необходимо сделать следущее:
1. Создать А-запись для поддомена site3.devops-test.ru
2. В папке с проектом devops-test создать папку для нового сайта "site3".
3. В папке site3 необходимо создать 3 подпапки: 
- "hosts" - для хранения конфигураций nginx
- "logs" - для хранения логов
- "www" - папка, в которой будут размещаться файлы сайта
4. В папке hosts создаём файл default.conf следующего содержания:

server {
	listen 80;

	server_name site3.devops-test.ru www.site3.devops-test.ru;
	access_log /var/log/nginx/site3.devops-test.ru.access.log main;
	root   /var/www/;

	location / {
		index  index.html index.htm index.php;
	}
	
}

5. В папке "www" создаем файл index.html, например, с текстом "welcome to site3"
6. Теперь необходимо отредактировать файл docker-compose.yml, находящийся в корне проекта.
Необходимо добавить следующий код

site3:
    image: nginx:alpine
    restart: always

    labels:
      - traefik.backend=site3
      - traefik.frontend.rule=Host:site3.devops-test.ru
      - traefik.docker.network=web
      - traefik.port=80

    volumes:
      - ./site3/hosts:/etc/nginx/conf.d
      - ./site3/www:/var/www
      - ./site3/logs:/var/log/nginx
    networks:
      - internal
      - web
    depends_on:
      - traefik
      
7. Запускаем команду из папки с проектом ~/devops-test

docker-compose up -d
