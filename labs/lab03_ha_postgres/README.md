# ЛР 1. HA Postgres Cluster



**Задача:** Развернуть и настроить высокодоступный кластер PostgreSQL.



Кластеризация выполнена при помощи \*\*Patroni\*\*,  

**ZooKeeper** используется для управления состоянием кластера и выбора лидера,  

**HAProxy** обеспечивает единую точку входа и высокую доступность.



---



## Часть 1. Поднимаем Postgres



Подготавливаем Dockerfile, включающий PostgreSQL и Patroni.



```Dockerfile

FROM postgres:15


RUN apt-get update \&\& apt-get install -y \\
&nbsp;   python3 python3-pip python3-venv \\
&nbsp;   libpq-dev gcc curl netcat-openbsd


RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"


RUN pip install --upgrade pip
RUN pip install patroni\[zookeeper] psycopg2-binary


COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml

```



Создаём docker-compose.yml:



```yaml

services:



&nbsp; pg-master:
&nbsp;   build: .
&nbsp;   container\_name: pg-master
&nbsp;   user: "postgres"
&nbsp;   ports:
&nbsp;     - "5433:5432"
&nbsp;   expose:
&nbsp;     - "8008"
&nbsp;   command: patroni /postgres0.yml
&nbsp;   depends\_on:
&nbsp;     - zoo



&nbsp; pg-slave:
&nbsp;   build: .
&nbsp;   container\_name: pg-slave
&nbsp;   user: "postgres"
&nbsp;   ports:
&nbsp;     - "5434:5432"
&nbsp;   expose:
&nbsp;     - "8008"
&nbsp;   command: patroni /postgres1.yml
&nbsp;   depends\_on:
&nbsp;     - zoo



&nbsp; zoo:
&nbsp;   image: zookeeper:3.9
&nbsp;   container\_name: zoo
&nbsp;   ports:
&nbsp;     - "2181:2181"

```



Деплоим кластер:



```bash

docker compose up -d --build

```



!\[](./screenshots/1.png)



Проверяем статус контейнеров:



```bash

docker compose ps

```



!\[](labs/lab03_ha_postgres/screenshots/screenshots/2.png)



Контейнеры pg-master, pg-slave и zoo успешно запущены.



---



## Часть 2. Проверяем репликацию



Подключаемся через pgAdmin:



\- pg-master — localhost:5433  

\- pg-slave — localhost:5434  



Проверяем роли.



Попытка записи на реплике:



```sql

INSERT INTO test\_replication VALUES (2, 'should fail on replica');

```



!\[](labs/lab03_ha_postgres/screenshots/screenshots/3.png)



Получаем ошибку read-only, что подтверждает корректную работу реплики.



Запись на мастере выполняется успешно:



!\[](labs/lab03_ha_postgres/screenshots/screenshots/4.png)



Репликация работает корректно.



---



## Часть 3. Добавляем HAProxy



Добавляем сервис HAProxy в docker-compose.yml:



```yaml

haproxy:
&nbsp; image: haproxy:3.0
&nbsp; container\_name: postgres\_entrypoint
&nbsp; ports:
&nbsp;   - "15432:5432"
&nbsp;   - "7000:7000"
&nbsp; depends\_on:
&nbsp;   - pg-master
&nbsp;   - pg-slave
&nbsp;   - zoo
&nbsp; volumes:
&nbsp;   - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg

```



Создаём haproxy.cfg:



```cfg

global
&nbsp;   maxconn 100


defaults
&nbsp;   log global
&nbsp;   mode tcp
&nbsp;   timeout connect 10s
&nbsp;   timeout client 1m
&nbsp;   timeout server 1m



listen stats
&nbsp;   mode http
&nbsp;   bind \*:7000
&nbsp;   stats enable
&nbsp;   stats uri /



listen postgres
&nbsp;   bind \*:5432
&nbsp;   option httpchk GET /master
&nbsp;   http-check expect status 200
&nbsp;   default-server inter 2s fall 3 rise 2 on-marked-down shutdown-sessions
&nbsp;   server pg-master pg-master:5432 check port 8008
&nbsp;   server pg-slave pg-slave:5432 check port 8008

```



Перезапускаем проект:



```bash

docker compose up -d --build

```



Открываем страницу статистики HAProxy:



http://127.0.0.1:7000



!\[](labs/lab03_ha_postgres/screenshots/screenshots/5.png)



На момент тестирования мастером являлся pg-slave.



---



## Подключение через единую точку входа



Подключаемся к базе через HAProxy:



\- localhost:15432  



Создаём таблицу (если необходимо):



```sql

CREATE TABLE test\_replication (

&nbsp; id INT PRIMARY KEY,

&nbsp; msg TEXT

);

```



Выполняем запись:



```sql

INSERT INTO test\_replication VALUES (400, 'before failover via haproxy');

SELECT \* FROM test\_replication;

```



!\[](labs/lab03_ha_postgres/screenshots/screenshots/6.png)



Запись проходит успешно.



---



## Часть 4. Тестирование failover



Отключаем текущего мастера:



```bash

docker stop pg-slave

```



!\[](labs/lab03_ha_postgres/screenshots/screenshots/7.png)



Обновляем страницу статистики HAProxy.



!\[](labs/lab03_ha_postgres/screenshots/screenshots/8.png)



pg-master автоматически становится новым мастером.



---



## Проверка сохранности данных



Пытаемся вставить существующий ключ:



```sql

INSERT INTO test\_replication VALUES (500, 'after failover');

```



!\[](labs/lab03_ha_postgres/screenshots/screenshots/9.png)



Получаем ошибку duplicate key, что подтверждает сохранность данных.



Добавляем новую запись:



```sql

INSERT INTO test\_replication VALUES (501, 'after failover confirmed');

SELECT \* FROM test\_replication ORDER BY id;

```



!\[](labs/lab03_ha_postgres/screenshots/screenshots/10.png)



Все записи присутствуют.



---



## Возникшие трудности



1\. Ошибка `initdb: cannot be run as root`  

&nbsp;  Решено добавлением `user: "postgres"`.



2\. Ошибки YAML (tab character violates indentation)  

&nbsp;  Причина — использование TAB вместо пробелов.



3\. Ошибка HAProxy (Missing LF on last line)  

&nbsp;  Решено добавлением перевода строки в конце haproxy.cfg.



4\. Разрыв соединения pgAdmin после failover  

&nbsp;  Решено переподключением к серверу.



---



## Вывод



Развернут отказоустойчивый кластер PostgreSQL с использованием Patroni и ZooKeeper.

HAProxy обеспечивает единую точку входа.

При отключении мастера система автоматически выполняет failover, а данные полностью сохраняются.



Лабораторная работа выполнена успешно.



