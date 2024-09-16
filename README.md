## О проекте
Домашнее задание по балансировке.  
Проект состоит из следующих компонентов:  
* Приложение .NET WebApi в папке ./server, которое собирается в образ server:local и контейнеры server1 и server2  
* Бэкап postgres в папке ./db, который собирается в образ db:local. На его основе создаются контейнеры pg_master, pg_slave1, pg_slave2  
* Контейниризованный ansible в папке ./ansible. В папке ./ansible/playbooks находится плейбук main.yml, необходимый для настройки стриминговой репликации pg_master -> pg_slave1, pg_slave2.
* Nginx и HAProxy, подключающиеся в Docker Compose. Конфигурационные файлы в корневой папке репозитория.
* Redis, RabbitMQ, подключающиеся в Docker Compose.  
### Начало работы
Склонировать проект, сделать cd в корень репозитория и запустить Docker Compose.  
```bash
https://github.com/npctheory/highload-loadbalancer.git
cd highload-loadbalancer
```
Создать docker-сеть:  
```bash
docker network create highload_net
```
Находясь в папке highload-loadbalancer запустить docker-compose:  
```bash
docker compose up --build -d
```
Когда все контейнеры будут запущены, для того чтобы репликация работала, выполнить плейбук main.yml:  
```bash
docker exec -it ansible bash
ansible-playbook playbooks/main.yml
```
Как это всё вместе должно работать:  

[setup.webm](https://github.com/user-attachments/assets/77300490-5382-4f02-aa14-2d342c9c911b)

### Nginx  
Docker Compose запустит два инстанса бэкенда: server1 и server2.  
В контейнере nginx будет запущен реверс-прокси, который будет распределять между ними запросы.  
Конфигурация Nginx:  
```
http {
    upstream backend_servers {
        server server1:80;
        server server2:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend_servers;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
            proxy_connect_timeout 60s;
        }
    }

}
events {
    worker_connections 1024;
}
```

Пример того как работает приложение при отключении инстансов бэкенда:  

[nginx.webm](https://github.com/user-attachments/assets/bf1b3746-1e33-4ece-94f9-0db4ff3fc42f)


### HAProxy
Docker Compose запустит три сервера Postgres: pg_master, pg_slave1, pg_slave2. Если плейбук main.yml, отработал то включится primary-secondary репликация, в которой pg_master будет primary, а pg_slave1 и pg_slave2 - secondary.    
В контейнере haproxy будет запущен TCP-балансировщик, который будет направлять запросы на порт 5432 на pg_master, а запросы на порт 5433 распределять между pg_slave1 и pg_slave2. В .NET приложении sql-запросы на запись используют в соединении порт 5432, а sql-запросы на чтение порт 5433.    
Конфигурация HAProxy:  
```
global
    log stdout format raw local0

defaults
    log global
    option tcplog
    timeout connect 5s
    timeout client 1m
    timeout server 1m

frontend pg_write_front
    bind *:5432
    mode tcp
    default_backend pg_write_back

frontend pg_read_front
    bind *:5433
    mode tcp
    default_backend pg_read_back

backend pg_write_back
    mode tcp
    balance roundrobin
    option tcp-check
    server pg_master pg_master:5432 check

backend pg_read_back
    mode tcp
    balance roundrobin
    option tcp-check
    server pg_slave1 pg_slave1:5432 check
    server pg_slave2 pg_slave2:5432 check
```
Пример того как работает приложение при отключении слейвов Postgres:  

[Postgres.webm](https://github.com/user-attachments/assets/6285c773-0396-4f0d-aee1-21474567063b)
