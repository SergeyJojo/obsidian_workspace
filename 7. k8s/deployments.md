
### **Разбираем Kubernetes Deployments: Полное Руководство**  

**Deployment** — это ключевой объект Kubernetes, который управляет жизненным циклом ваших приложений, обеспечивая:  
- ✅ **Развертывание** и обновление подов (Pods).  
- ✅ **Масштабирование** (увеличение/уменьшение числа реплик).  
- ✅ **Откат** к предыдущей версии при проблемах.  
- ✅ **Самовосстановление** (если под падает, Deployment создает новый).  

---

## **1. Как работает Deployment?**  
Deployment контролирует **ReplicaSet**, который, в свою очередь, управляет подами.  

```
+---------------------+
|     Deployment      |  # Описывает желаемое состояние (образ, кол-во реплик)
+----------+----------+
           |
           v
+---------------------+
|      ReplicaSet     |  # Обеспечивает, что запущено нужное кол-во подов
+----------+----------+
           |
           v
+---------------------+
|        Pods         |  # Фактические контейнеры с приложением
+---------------------+
```

---

## **2. Основные функции Deployment**  

### **A. Создание и управление подами**  
**Пример `deployment.yaml`:**  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3  # Количество подов
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Что здесь важно:**  
- **`replicas: 3`** — Deployment создаст 3 идентичных пода.  
- **`selector.matchLabels`** — связывает Deployment с подами через метки (labels).  
- **`template`** — шаблон для создания подов.  

**Применяем:**  
```bash
kubectl apply -f deployment.yaml
```

---

### **B. Масштабирование**  
Увеличим число подов до 5:  
```bash
kubectl scale deployment nginx-deployment --replicas=5
```  
Или через редактирование YAML:  
```yaml
spec:
  replicas: 5
```

---

### **C. Обновление образа (Rolling Update)**  
Deployment обновляет поды **постепенно**, чтобы избежать downtime.  

**Обновляем версию nginx:**  
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.26
```  
**Что происходит:**  
1. Создается новый ReplicaSet с `nginx:1.26`.  
2. Постепенно запускаются новые поды, а старые удаляются.  
3. Если что-то идет не так — автоматический откат.  

**Стратегия обновления (strategy):**  
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # На сколько подов можно временно превысить replicas
      maxUnavailable: 0   # Сколько подов могут быть недоступны во время обновления
```

---

### **D. Откат (Rollback)**  
Если новая версия сломалась, вернем предыдущую:  
```bash
kubectl rollout undo deployment/nginx-deployment
```  
**Просмотр истории изменений:**  
```bash
kubectl rollout history deployment/nginx-deployment
```

---

### **E. Проверка состояния**  
**Статус Deployment:**  
```bash
kubectl get deployments
```  
**Вывод:**  
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           5m
```  
- **READY** — сколько подов запущено (3 из 3).  
- **UP-TO-DATE** — сколько подов обновлено до последней версии.  
- **AVAILABLE** — сколько подов доступно для трафика.  

**Детальный статус:**  
```bash
kubectl describe deployment nginx-deployment
```

---

## **3. Продвинутые сценарии**  

### **A. Ленивые обновления (Paused)**  
Приостанавливаем обновления для ручной проверки:  
```bash
kubectl rollout pause deployment/nginx-deployment
```  
Возобновляем:  
```bash
kubectl rollout resume deployment/nginx-deployment
```

### **B. Blue-Green Deployment**  
1. Создаем 2 Deployment с разными версиями.  
2. Переключаем трафик через Service.  

**Пример:**  
```bash
kubectl apply -f deployment-blue.yaml
kubectl apply -f deployment-green.yaml
# Переключаем Service на "green":
kubectl patch service my-service -p '{"spec":{"selector":{"version":"green"}}}'
```

---

## **4. Ограничения и лучшие практики**  

### **Чего не может Deployment?**  
- ❌ Управление stateful-приложениями (нужен **StatefulSet**).  
- ❌ Запуск одного пода на каждой ноде (нужен **DaemonSet**).  

### **Советы:**  
1. **Всегда указывайте `resources` (CPU/RAM):**  
   ```yaml
   resources:
     requests:
       cpu: "100m"
       memory: "256Mi"
     limits:
       cpu: "500m"
       memory: "512Mi"
   ```  
2. **Используйте `readinessProbe` и `livenessProbe`:**  
   ```yaml
   livenessProbe:
     httpGet:
       path: /health
       port: 80
     initialDelaySeconds: 5
     periodSeconds: 10
   ```  
3. **Тегируйте образы явно (`nginx:1.25`, а не `nginx:latest`).**  

---

## **5. Полезные команды**  
| Команда | Описание |  
|---------|----------|  
| `kubectl get deployments` | Список Deployment |  
| `kubectl rollout status deployment/nginx` | Статус обновления |  
| `kubectl edit deployment nginx` | Редактирование в реальном времени |  
| `kubectl delete deployment nginx` | Удаление Deployment |  

---

## **Итог**  
Deployment — это **основной инструмент** для управления stateless-приложениями в Kubernetes. Он обеспечивает:  
- 🔄 **Бесшовные обновления** (rolling updates).  
- 🔢 **Масштабирование** (replicas).  
- ⏮ **Откаты** при ошибках.  

**Пример полного манифеста:** [Официальная документация](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).