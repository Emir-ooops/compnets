# ЛР 2. Loki + Zabbix + Grafana

**Задача:** Подключить к сервису Nextcloud мониторинг и логирование. Выполнить визуализацию данных через Grafana.

---

## Часть 1. Логирование (Loki + Promtail)

Собираем `docker-compose.yml` и `promtail_config.yml` согласно методическим указаниям.

Запускаем стек:

```
docker compose up --build -d
docker ps
```

Проверяем, что поднялись контейнеры:

- nextcloud  
- loki  
- promtail  
- grafana  
- zabbix-front  
- zabbix-back  
- postgres-zabbix  

![1](https://github.com/user-attachments/assets/1ec76106-34da-4eeb-a5ef-97029229b6c7)


---

Далее инициализируем Nextcloud.

Переходим в браузере по адресу:

```
http://localhost:8080
```

Создаем учетную запись администратора.

![2](https://github.com/user-attachments/assets/321b7fc7-5ee5-4c09-9949-94fdc8e67962)



Проверяем работу Promtail:

```
docker logs promtail
```

В логах должны присутствовать строки вида:

```
msg="Seeked /opt/nc_data/nextcloud.log"
```

Это означает, что promtail корректно читает лог-файл.

<img width="1002" height="16" alt="3" src="https://github.com/user-attachments/assets/168ee8f2-0ed3-4a25-8be9-c42b2a7139e4" />


## Часть 2. Мониторинг (Zabbix)

Переходим в Zabbix:

```
http://localhost:8082
```

Авторизуемся (Admin / zabbix).

---

### Исправление ошибки trusted_domains

При обращении к Nextcloud возникала ошибка:

> Host localhost was not connected because it violates local access rules

Исправляем в контейнере Nextcloud:

```
docker exec -u 33 -it nextcloud php occ config:system:set trusted_domains 1 --value="nextcloud"
```

После выполнения получаем:

```
System config value trusted_domains => 1 set to string nextcloud
```

### Импорт шаблона

Переходим:

```
Data collection → Templates → Import
```

Загружаем подготовленный template.yml.

![4](https://github.com/user-attachments/assets/910f2620-e061-409e-9412-9761e22a6d81)


### Добавление хоста

Переходим:

```
Data collection → Hosts → Create host
```

Создаем хост:

- Host name: nextcloud  
- Template: Test ping template

![5](https://github.com/user-attachments/assets/b466f5bb-63b2-46c6-b310-3f23e1e5b3c9)

---

### Проверка мониторинга

Переходим:

```
Monitoring → Latest data
```

Проверяем:

- Last check обновляется  
- Last value = healthy  

Это подтверждает работу health-check.

![8](https://github.com/user-attachments/assets/5c144aa6-7afe-4bf1-9e9d-0a0d29d2b60f)


---

### Настройка Preprocessing

В Item добавлены шаги:

1. JSONPath → `$.body.maintenance`  
2. Replace → false → healthy  
3. Replace → true → unhealthy  

Тип данных: Text.

![7](https://github.com/user-attachments/assets/da17d2c6-0d2c-4f9b-8175-e0d824ffd3db)


---

## Часть 3. Визуализация (Grafana)

Для интеграции с Zabbix устанавливаем плагин:

```
docker exec -it grafana bash -c "grafana cli plugins install alexanderzobnin-zabbix-app"
docker restart grafana
```

![9](https://github.com/user-attachments/assets/80ddc20e-7bca-4de3-9eca-ddf212cd7fe0)


---

Заходим в Grafana:

```
http://localhost:3000
```

---

### Подключение Loki

Переходим:

```
Connections → Data sources → Add data source → Loki
```

URL:

```
http://loki:3100/
```

После сохранения появляется сообщение:

> Data source successfully connected

![12](https://github.com/user-attachments/assets/2894a666-8f26-4602-b661-6a6793ccf8fa)

---

### Подключение Zabbix

Переходим:

```
Connections → Data sources → Add data source → Zabbix
```

URL:

```
http://zabbix-front:8080/api_jsonrpc.php
```

После сохранения отображается:

> Data source successfully connected  
> Zabbix API version 6.4.21

![11](https://github.com/user-attachments/assets/5e2d65ff-3669-4dc9-9334-23024b79fc5d)


---

### Просмотр логов в Grafana (Loki)

В разделе Explore выполняем запрос:

```
{job="nextcloud_logs"} != "violates local access rules"
```

Отображаются логи Nextcloud.

![13](https://github.com/user-attachments/assets/aaf1b50e-3010-4e0d-9e02-3ead824301bc)

---

### Визуализация

Создаем Dashboard в Grafana:

- статус health-check из Zabbix  

<img width="1470" height="920" alt="14" src="https://github.com/user-attachments/assets/7aad5334-0d1d-4f13-af44-194c1502fe26" />


---

# Итог

В ходе лабораторной работы:

- Развернут стек Loki + Promtail + Zabbix + Grafana  
- Настроен мониторинг Nextcloud  
- Реализован health-check через Zabbix  
- Настроен сбор логов через Promtail  
- Подключены источники данных в Grafana  
- Выполнена визуализация  

Система мониторинга и логирования функционирует корректно.

