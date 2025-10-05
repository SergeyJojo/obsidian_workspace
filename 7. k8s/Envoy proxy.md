
Конечно, Сергей. Давай подробно разберём, как устроен **Envoy Proxy** — компонент, который стал де-факто стандартом в облачной и сервис-меш архитектуре.

---

## 📌 Что такое Envoy?

**Envoy Proxy** — это современный, высокопроизводительный, _L3/L4 и L7_ прокси-сервер с поддержкой сервис-меша, написанный на C++ и изначально созданный в Lyft.

Он используется как **data plane** (передача данных) в системах вроде Istio, Consul Connect, AWS App Mesh и т.д. Но также может использоваться **самостоятельно** как edge proxy или internal proxy между сервисами.

---

## 🧱 Архитектура Envoy — основные компоненты

### 1. **Listener**

- Определяет, **на каком IP и порту Envoy принимает входящие соединения**.
    
- Можно настроить несколько листенеров (например, HTTP на 8080, HTTPS на 8443).
    
- Каждый Listener подключен к **filter chain**.
    

### 2. **Filter Chain (цепочка фильтров)**

- Сердце обработки запроса.
    
- Каждый Listener имеет список фильтров, которые работают **по порядку**, обрабатывая соединение:
    
    - L4-фильтры: TLS termination, TCP proxy.
        
    - L7-фильтры: HTTP connection manager, routing, rate limiting, ext_authz и т.д.
        

Пример цепочки для HTTP:

```
TLS termination → HTTP connection manager → Router
```

### 3. **Route Configuration**

- HTTP фильтр подключает **HTTP маршруты**, описывающие:
    
    - какой URL → в какой **Cluster** направить;
        
    - использовать ли rewrite, retry, redirect;
        
    - применять ли rate limiting, tracing и т.д.
        

### 4. **Cluster**

- Представляет собой **набор upstream-серверов**, куда Envoy может направлять запросы.
    
- Cluster'ы бывают:
    
    - `STATIC`: задан список IP/портов;
        
    - `EDS` (Endpoint Discovery Service): динамически получаемые адреса;
        
    - `STRICT_DNS`: DNS-сервис с автообновлением.
        

### 5. **Endpoint**

- Это **один экземпляр** сервиса в кластере (IP:port).
    
- Envoy может балансировать нагрузку между endpoint’ами (round robin, least request и т.д.).
    
- Может автоматически **health-check**-ать их.
    

### 6. **Admin интерфейс**

- Доступен по HTTP (`localhost:9901`).
    
- Показывает метрики, конфигурацию, лог маршрутов, профайлинг и т.д.
    

---

## ⚙️ Пример потока запроса (HTTP)

1. Клиент подключается к `Listener` на порту 8080.
    
2. TLS фильтр дешифрует трафик (если нужен).
    
3. HTTP Connection Manager разбирает HTTP-запрос.
    
4. Срабатывает routing filter:
    
    - Запрос `GET /users/123` → route rule.
        
    - Rule направляет в Cluster `users_backend`.
        
5. Внутри `users_backend` Envoy выбирает один из endpoint'ов (например, `10.0.1.5:5000`) и делает proxy-переход.
    
6. Получив ответ — обратный путь: response → фильтры → клиент.
    

---

## 🔁 Поддержка динамической конфигурации (xDS API)

Envoy поддерживает _control plane_, который может **динамически управлять** конфигурацией прокси:

|Тип|Назначение|
|---|---|
|ADS|Aggregated Discovery Service|
|CDS|Cluster Discovery Service|
|EDS|Endpoint Discovery Service|
|LDS|Listener Discovery Service|
|RDS|Route Discovery Service|
|SDS|Secret Discovery Service (TLS)|

Таким образом, **Envoy сам не хранит конфиг статично**, а может получать всё "на лету" от control-plane (например, Istio Pilot).

---

## 🧰 Возможности Envoy:

- ✅ HTTP/2 и gRPC support
    
- ✅ Load balancing (RR, least request, ring hash, магистрально-конфигурируемый)
    
- ✅ Circuit breaking
    
- ✅ Retry logic
    
- ✅ Rate limiting
    
- ✅ Tracing (Zipkin, Jaeger, Datadog)
    
- ✅ Prometheus метрики
    
- ✅ WebSocket и raw TCP проксирование
    
- ✅ TLS termination и mTLS между сервисами
    
- ✅ Ext_authz (авторизация внешним сервисом)
    

---

## 📦 Пример конфигурации (упрощённо)

```yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 10000
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/...
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: my_service
                http_filters:
                  - name: envoy.filters.http.router
  clusters:
    - name: my_service
      connect_timeout: 0.25s
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: my_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: my-service.internal
                      port_value: 8080
```

---

## 📌 Envoy как sidecar

В архитектуре service mesh (например, **Istio**) Envoy **запускается рядом с каждым сервисом** (sidecar):

```
[App Container] ⇄ [Envoy Sidecar] ⇄ [Mesh]
```

Он:

- перехватывает исходящий трафик,
    
- оборачивает его в TLS (mTLS),
    
- добавляет трассировку,
    
- шлёт логи, метрики,
    
- применяет политики доступа (RBAC),
    
- может остановить "зловредный" трафик.
    

---

## 🔍 Как думать о Envoy

- Это **kernel** сетевого уровня для микросервисов;
    
- **Интерпретатор политик и маршрутов**, заданных через xDS;
    
- **Платформа**, а не просто прокси.
    

---

Хочешь, могу:

- разобрать конкретный YAML конфиг;
    
- объяснить, как работает маршрутизация с `prefix_match`, `regex_match`;
    
- рассказать, как подключить Jaeger/Prometheus;
    
- показать разницу между Envoy и NGINX (в L7 проксировании).
    

Что интересно глубже?