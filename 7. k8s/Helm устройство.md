
### **Что такое Helm, Deployment, Service в Kubernetes и как они связаны**

---

## **1. Helm: пакетный менеджер для Kubernetes**
**Определение:**  
Helm — это инструмент для управления приложениями в Kubernetes через **чарты (charts)** — упакованные наборы YAML-файлов.  

**Зачем нужен?**  
- Упрощает деплой сложных приложений (например, Elasticsearch с 10+ компонентами).  
- Позволяет параметризировать конфиги (как шаблонизатор).  
- Даёт версионность и возможность отката (`helm rollback`).  

**Основные понятия:**  
| Термин       | Описание                                                                 |
|--------------|--------------------------------------------------------------------------|
| **Chart**    | Пакет с описанием приложения (аналог DEB/RPM).                          |
| **Release**  | Экземпляр развернутого чарта (можно иметь несколько версий одного чарта). |
| **Values**   | Переменные для кастомизации (`values.yaml` или `--set key=value`).      |

**Пример:**  
```bash
helm install my-app ./chart --set replicaCount=3
```

---

## **2. Deployment: управление подами**
**Определение:**  
Deployment — это объект Kubernetes, который:  
- Описывает **желаемое состояние** приложения (образ, реплики, стратегия обновления).  
- Управляет **Pod'ами** (создает/удаляет/обновляет).  
- Обеспечивает **отказоустойчивость** (автовосстановление после падения).  

**Что внутри `deployment.yaml`?**  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3  # Количество Pod'ов
  selector:
    matchLabels:
      app: nginx
  template:    # Шаблон Pod'а
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
```

**Как Helm использует Deployment?**  
В чарте Helm это выглядит так (`templates/deployment.yaml`):  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - image: {{ .Values.image }}
```

---

## **3. Service: доступ к подам**
**Определение:**  
Service — это абстракция, которая:  
- Дает **постоянный IP/DNS-имя** для набора Pod'ов (даже если те пересоздаются).  
- Определяет **способ доступа** (ClusterIP, NodePort, LoadBalancer).  

**Типы Service:**  
| Тип            | Доступность               | Пример использования               |
|----------------|--------------------------|------------------------------------|
| **ClusterIP**  | Только внутри кластера    | Внутренний API                     |
| **NodePort**   | Порт на всех нодах        | Тестирование без Ingress           |
| **LoadBalancer**| Внешний IP (облако)      | Публичный веб-сервис               |

**Пример `service.yaml`:**  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx  # Связь с Pod'ами через labels
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**В Helm:**  
```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
```

---

## **4. Как это работает вместе?**
1. **Helm-чарт** содержит:  
   - `templates/deployment.yaml` → Создает Pod'ы.  
   - `templates/service.yaml` → Дает доступ к Pod'ам.  
   - `values.yaml` → Параметры (образ, порты, реплики).  

2. **Деплой:**  
   ```bash
   helm install my-app ./chart --set replicaCount=3
   ```
   - Создается **Deployment** с 3 Pod'ами.  
   - Создается **Service** для доступа к ним.  

3. **Доступ:**  
   - Внутри кластера: `curl my-app-svc:80`.  
   - Снаружи: Если Service типа `LoadBalancer`, используем внешний IP.  

---

## **5. Продвинутые фичи**
### **5.1. Ingress + Helm**
Для маршрутизации HTTP-трафика:  
1. Добавляем `templates/ingress.yaml`:  
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: {{ .Release.Name }}-ingress
   spec:
     rules:
     - host: {{ .Values.ingress.host }}
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: {{ .Release.Name }}-svc
               port:
                 number: 80
   ```
2. Устанавливаем контроллер Ingress (например, Nginx):  
   ```bash
   helm install ingress-nginx bitnami/nginx-ingress-controller
   ```

### **5.2. HPA (Horizontal Pod Autoscaler)**
Автомасштабирование через Helm:  
```yaml
# templates/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Release.Name }}-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Release.Name }}-deployment
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## **6. Проблемы и решения**
| Проблема                          | Решение                              |
|-----------------------------------|--------------------------------------|
| Под не запускается                | `kubectl describe pod <имя>` + логи  |
| Service не доступен               | Проверить `selector` и labels Pod'ов |
| Helm не обновляет релиз           | `helm upgrade --install`             |
| Ingress возвращает 404            | Проверить `host` и `path` в правилах |

---

## **Вывод**
1. **Helm** — это «пакетный менеджер» для Kubernetes, который управляет:  
   - **Deployment** → Кто и как запускает Pod'ы.  
   - **Service** → Как до них достучаться.  
2. **Deployment** гарантирует, что нужное количество Pod'ов работает.  
3. **Service** обеспечивает стабильный доступ к Pod'ам.  
4. **Ingress** добавляет маршрутизацию HTTP/HTTPS.  

**Команды для проверки:**  
```bash
kubectl get pods,svc,deploy,ingress  # Список ресурсов
helm list                            # Список релизов
helm status my-app                   # Статус конкретного релиза
```