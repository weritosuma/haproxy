### HAProxy: Расширенное Руководство с Реальными Примерами

---

#### **1. Алгоритмы балансировки (подробное описание)**

HAProxy поддерживает гибкие алгоритмы для распределения трафика. Вот ключевые из них:

1. **Round Robin (`balance roundrobin`)**  
   - Запросы распределяются последовательно между серверами.  
   - **Пример**:  
     Серверы `app-server-1:8080`, `app-server-2:8080`, `app-server-3:8080`.  
     Запросы идут в порядке: 1 → 2 → 3 → 1 → 2 и т.д.

2. **Least Connections (`balance leastconn`)**  
   - Запрос отправляется на сервер с наименьшим числом активных соединений.  
   - **Используется для**: Динамических нагрузок (например, API с долгими запросами).

3. **Source IP Hash (`balance source`)**  
   - Хеширует IP-адрес клиента для закрепления сессий.  
   - **Пример**:  
     Клиент с IP `192.168.1.100` всегда направляется на `app-server-1`.

4. **URI Hash (`balance uri`)**  
   - **Как работает**:  
     HAProxy хеширует часть или весь URL (например, `/blog/post/123`) и использует это значение для выбора сервера. Запросы с одинаковым URI всегда направляются на один и тот же сервер.  
   - **Зачем нужно**:  
     - Для **кэширования**: Если серверы имеют локальный кэш, повторные запросы к `/images/logo.png` попадут на тот же сервер, где кэш уже загружен.  
     - Для **сеансов**: Например, все запросы к `/user/123/profile` будут обрабатываться одним сервером.  
   - **Настройка**:  
     ```cfg
     backend static_servers
         balance uri  # Хеширует полный URI
         server static1 10.0.0.151:80 check
         server static2 10.0.0.152:80 check
     ```
   - **Пример**:  
     - Запросы:  
       - `GET /images/cat.jpg` → Хэш = 123 → Сервер `static1`.  
       - `GET /images/dog.jpg` → Хэш = 456 → Сервер `static2`.  
       - Повторный запрос к `/images/cat.jpg` снова пойдет на `static1`.  
   - **Дополнительные опции**:  
     - `balance uri whole`: Хешировать весь URI (по умолчанию).  
     - `balance uri path-only`: Игнорировать параметры запроса (`?id=1`).  
     - `balance uri len 10`: Хешировать только первые 10 символов URI.  

5. **Weighted Round Robin**  
   - Учитывает вес серверов.  
   - **Пример**:  
     ```cfg
     server app1 10.0.0.1:8080 weight 3  # Получает 3/4 трафика
     server app2 10.0.0.2:8080 weight 1  # Получает 1/4 трафика
     ```

6. **Random**  
   - Случайный выбор сервера.  
   - **Пример**:  
     Используется для тестирования отказоустойчивости.

---

#### **2. ACL (Access Control Lists) для Маршрутизации**
ACL позволяют направлять трафик на основе условий. Примеры:

- **Маршрутизация по домену**:
  ```cfg
  acl is_domain_api hdr(host) -i api.example.com
  use_backend api_servers if is_domain_api
  ```

- **Маршрутизация по пути**:
  ```cfg
  acl is_images_path path_beg /images
  use_backend image_servers if is_images_path
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

#### **3. Пример Конфигурации HAProxy с Реальными Серверами**
**Сценарий**:  
- Балансировка между 3 веб-серверами (`app-server-1`, `app-server-2`, `app-server-3`).  
- Отдельный бэкенд для статики (`static-server`).  
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

#### **4. Запуск через Docker Compose**
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

#### **5. Проверка Работы**
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
HAProxy позволяет строить сложные схемы балансировки с учетом реальных сценариев:  
- **URI Hash** идеален для закрепления запросов к статическим ресурсам или API-методам.  
- **ACL** обеспечивают гибкую маршрутизацию (например, направление `/api` на отдельный кластер).  
- **Алгоритмы балансировки** выбираются под задачу: Round Robin, Least Connections, Weighted и др.  

Примеры конфигов демонстрируют работу с реальными серверами (веб, API, статика).
