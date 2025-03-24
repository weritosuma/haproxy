### HAProxy: Полное Руководство с Примерами

---

#### **1. Что такое HAProxy?**
HAProxy (High Availability Proxy) — это высокопроизводительный балансировщик нагрузки и прокси-сервер для TCP и HTTP. Используется для:
- Распределения трафика между серверами.
- Обеспечения отказоустойчивости.
- Маршрутизации запросов на основе правил (ACL).

**Сетевые уровни**:
- **4-й уровень (Transport)**: Работает с TCP/UDP, балансирует по IP и портам (например, базы данных).
- **7-й уровень (Application)**: Анализирует HTTP-заголовки, URL, cookies (например, маршрутизация по домену).

---

#### **2. Алгоритмы балансировки**

1. **Round Robin (`balance roundrobin`)**  
   - Запросы распределяются последовательно между серверами.  
   - **Пример**:  
     Серверы `app1:8080`, `app2:8080`, `app3:8080`.  
     Порядок: app1 → app2 → app3 → app1...

2. **Least Connections (`balance leastconn`)**  
   - Запрос отправляется на сервер с наименьшим числом активных соединений.  
   - **Используется для**: Динамических нагрузок (например, API с долгими запросами).

3. **Source IP Hash (`balance source`)**  
   - Хеширует IP-адрес клиента для закрепления сессий.  
   - **Пример**:  
     Клиент `192.168.1.100` всегда направляется на `app1`.

4. **URI Hash (`balance uri`)**  
   - **Как работает**:  
     HAProxy хеширует часть или весь URL (например, `/images/logo.png`) и направляет запросы с одинаковым URI на один сервер.  
   - **Зачем нужно**:  
     - Кэширование статики (повторные запросы попадают на сервер с закэшированными данными).  
     - Закрепление сессий (например, все запросы к `/user/123` идут на один сервер).  
   - **Настройка**:  
     ```cfg
     backend static_servers
         balance uri
         server static1 10.0.0.151:80 check
         server static2 10.0.0.152:80 check
     ```
   - **Дополнительные опции**:  
     - `balance uri whole`: Хешировать весь URI (по умолчанию).  
     - `balance uri path-only`: Игнорировать параметры запроса (`?id=1`).  
     - `balance uri len 10`: Хешировать первые 10 символов URI.  

5. **Weighted Round Robin**  
   - Учитывает вес серверов.  
   - **Пример**:  
     ```cfg
     server app1 10.0.0.1:8080 weight 3  # 75% трафика
     server app2 10.0.0.2:8080 weight 1  # 25% трафика
     ```

6. **Random**  
   - Случайный выбор сервера.  
   - **Используется для**: Тестирования отказоустойчивости.

---

#### **3. ACL (Access Control Lists) для Маршрутизации**
ACL позволяют направлять трафик на основе условий. Примеры:

- **Маршрутизация по домену**:
  ```cfg
  acl is_api hdr(host) -i api.example.com
  use_backend api_servers if is_api
  ```

- **Маршрутизация по пути**:
  ```cfg
  acl is_images path_beg /images
  use_backend image_servers if is_images
  ```

- **Маршрутизация по HTTP-методу**:
  ```cfg
  acl is_post method POST
  use_backend post_servers if is_post
  ```

- **Маршрутизация по заголовку**:
  ```cfg
  acl is_mobile hdr(User-Agent) -i iPhone Android
  use_backend mobile_servers if is_mobile
  ```

- **Комбинация условий**:
  ```cfg
  acl is_admin hdr(Cookie) admin=1
  acl is_backend path_beg /admin
  use_backend admin_servers if is_admin is_backend
  ```

---

#### **4. Пример Конфигурации HAProxy**
**Сценарий**:  
- Балансировка между 3 веб-серверами.  
- Отдельный бэкенд для API и статики.  
- Маршрутизация по домену и пути.

```cfg
global
    log stdout format raw local0
    maxconn 2048

defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend http_front
    bind *:80
    # ACL для домена и пути
    acl is_api hdr(host) -i api.example.com
    acl is_static path_beg /static
    # Маршрутизация
    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend app_servers

backend app_servers
    balance roundrobin
    server app1 10.0.0.101:8080 check
    server app2 10.0.0.102:8080 check
    server app3 10.0.0.103:8080 check

backend api_servers
    balance leastconn
    server api1 10.0.0.201:8080 check
    server api2 10.0.0.202:8080 check

backend static_servers
    balance uri
    server static1 10.0.0.151:80 check
    server static2 10.0.0.152:80 check
```

---

#### **5. Настройка через Docker Compose**
```yaml
version: '3'
services:
  haproxy:
    image: haproxy:latest
    ports:
      - "80:80"
      - "8404:8404"  # Порт для статистики
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    depends_on:
      - app1
      - app2
      - app3
      - api1
      - static1
      - static2

  app1:
    image: your-web-app  # Замените на ваш образ
    ports:
      - "8080"

  app2:
    image: your-web-app
    ports:
      - "8080"

  app3:
    image: your-web-app
    ports:
      - "8080"

  api1:
    image: your-api-app  # Замените на ваш образ
    ports:
      - "8080"

  static1:
    image: nginx
    volumes:
      - ./static1:/usr/share/nginx/html

  static2:
    image: nginx
    volumes:
      - ./static2:/usr/share/nginx/html
```

---

#### **6. Проверка Работы**
- **Статус HAProxy**:  
  Добавьте в конфиг:
  ```cfg
  listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats auth admin:password
  ```
  Доступ: `http://localhost:8404/stats` (логин: `admin`, пароль: `password`).

- **Тестирование ACL**:  
  ```bash
  curl -H "Host: api.example.com" http://localhost  
  curl http://localhost/static/logo.png
  ```

- **Проверка URI Hash**:  
  ```bash
  # Первый запрос к /images/cat.jpg
  curl http://localhost/images/cat.jpg  # Уйдет на static1
  # Второй запрос к тому же URI
  curl http://localhost/images/cat.jpg  # Снова static1
  ```

---

### **Итог**
HAProxy — мощный инструмент для балансировки нагрузки, поддерживающий:
- **Гибкие алгоритмы**: Round Robin, Least Connections, URI Hash и др.  
- **Маршрутизацию через ACL**: По доменам, путям, заголовкам.  
- **Интеграцию с Docker**: Легкое развертывание и масштабирование.  

Примеры конфигураций и настроек демонстрируют работу с реальными сценариями (веб-серверы, API, статика).
