Вот подробный гайд по работе со стеком **ELK** — Elasticsearch, Logstash, Kibana — с упором на практическое применение в продакшене.

---

## 🧱 Общая структура ELK стека

- **Elasticsearch** — поисковый движок и хранилище данных.
    
- **Logstash** — инструмент для сбора, трансформации и отправки логов.
    
- **Kibana** — визуализация и дашборды на основе данных из Elasticsearch.
    

Иногда стек называют **Elastic Stack**, особенно если используется **Beats** (агенты сбора логов).

---

## 📦 Установка компонентов

### Elasticsearch

```bash
docker run -d --name elasticsearch -p 9200:9200 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:8.13.4
```

### Kibana

```bash
docker run -d --name kibana -p 5601:5601 --link elasticsearch:elasticsearch docker.elastic.co/kibana/kibana:8.13.4
```

### Logstash (пример с конфигом)

```bash
docker run -d --name logstash \
  -v "$PWD/logstash.conf":/usr/share/logstash/pipeline/logstash.conf \
  --link elasticsearch:elasticsearch \
  docker.elastic.co/logstash/logstash:8.13.4
```

---

## 🧰 Основные конфиги

### Пример `logstash.conf`

```plaintext
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}" }
  }
  date {
    match => ["timestamp", "ISO8601"]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

---

## 📡 Сбор логов с серверов

- Используется **Filebeat** — лёгкий агент от Elastic.
    
- Устанавливается на каждый сервер.
    

### Пример `filebeat.yml`:

```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/nginx/*.log

output.logstash:
  hosts: ["logstash:5044"]
```

---

## 📊 Kibana

- Интерфейс по адресу: `http://localhost:5601`
    
- Не забудь настроить **Index Pattern** (`logs-*`) для работы с логами.
    
- Используй:
    
    - Discover — просмотр логов
        
    - Dashboard — сводки
        
    - Visualize — графики
        
    - Alerting — оповещения
        

---

## 🧪 Лучшие практики

1. **Схема логов**: логируй в JSON, например:
    
    ```json
    {"timestamp":"2025-08-07T20:00:00Z","level":"INFO","msg":"User logged in","user_id":42}
    ```
    
2. **Grok или JSON?**
    
    - Если сервис пишет в JSON — обрабатывай через `json` плагин, это быстрее и надёжнее.
        
    - Grok — только для обычного текста.
        
3. **Индексация**
    
    - Индексы по дню: `logs-2025.08.07`
        
    - Настрой политики ILM (Index Lifecycle Management) для удаления старых логов.
        
4. **Безопасность**
    
    - Используй HTTPS, логин/пароль, TLS между компонентами.
        
    - Если в облаке — Elastic Cloud предлагает out-of-box безопасный стек.
        

---

## 🏗 Расширения

- **Beats**: Filebeat, Metricbeat, Packetbeat и др.
    
- **Fleet + Elastic Agent** — современный способ централизованного управления агентами.
    
- **Logstash plugins**: [официальный список](https://www.elastic.co/guide/en/logstash/current/input-plugins.html)
    

---

## 🚀 Продакшн-заметки

|Задача|Инструмент|
|---|---|
|Централизованный сбор|Filebeat + Logstash|
|Хранилище логов|Elasticsearch|
|Визуализация|Kibana|
|Алерты|Kibana Alerting|
|Безопасность|TLS, RBAC|
|Архивация|S3, ILM policies|

---

## 🔗 Полезные ссылки

- [Документация Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
    
- [Документация Logstash](https://www.elastic.co/guide/en/logstash/current/index.html)
    
- [Документация Kibana](https://www.elastic.co/guide/en/kibana/current/index.html)
    
- [Filebeat docs](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
    

---

Если хочешь, могу дополнительно:

- Нарисовать схему компонентов ELK;
    
- Добавить раздел по OpenSearch как опенсорс-альтернативе;
    
- Разобрать типовой use case — например, сбор логов с Kubernetes.