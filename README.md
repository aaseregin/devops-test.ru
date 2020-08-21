# devops-test.ru
Данный репозиторий - решение решение тестового задания для DevOps

Тестовое задание (DevOps)
Общая информация
Задания можно отнести к уровню Middle, но без ограничения времени и при наличии доступа к сети Интернет, оно вполне выполнимо для молодого
специалиста.
Оно позволит нам оценить Ваши возможности при решении реальных задач, а Вы сможете понять насколько подобные задачи Вам интересны.
Стек используемых технологий вполне современный и будет крайне полезен для повышения профессиональных знаний.
Исходные данные
Для выполнения задания необходим виртуальная машина (ВМ) с ОС Ubuntu 18.04 и с глобальным IP адресом. 
Задание
Необходимо развернуть web-сервер с двумя сайтами работающими по протоколу HTTPS. Раздаваемый контент очень простой, например, в index.html
поместить текст: "Welcome to site 1" и "Welcome to site 2", для первого и второго сайта соответственно.
Запросы, пришедшие по протоколу HTTP, должны перенаправляться на HTTPS.
Технические условия
В качестве web-серверов для сайтов используется Nginx в docker контейнере (каждый сайт в своем контейнере).
В качество обратного прокси с терминированием SSL необходимо использовать Traefik ( https://docs.traefik.io/ ) в контейнере. Именно он должен
слушать порты 80 и 443 ВМ. SSL сертификаты Traefik должен самостоятельно получать в центре сертификации Let's Encrypt.
Задание не предполагает сборку своих Docker образов, можно и нужно использовать официальные образы от Traefik и Nginx.
Для оркестрации этих трех контейнеров предлагается использовать Docker Compose ( https://docs.docker.com/compose/ ). Docker Compose позволит
организовать старт и остановку всех трех контейнеров одной командой.
Docker и Docker Compose необходимо установить на сервер самостоятельно.
Результат
Результат выполнения должен представлять из себя набор из файлов:
1. Файлы конфигурации Traefik
2. Файлы конфигурации Docker Compose
3. Инструкция по запуска этого набора контейнеров
4. Инструкция по добавлению site3 в эту конфигурацию
Инструкции можно писать в простом текстовом файле с использованием языка Markdown.
Будет плюсом размещение всех результатов в github репозитории ( https://github.com/ )
