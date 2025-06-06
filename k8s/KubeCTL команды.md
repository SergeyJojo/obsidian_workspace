
Вот список полезных команд для работы с `kubectl`. Они покрывают основные задачи управления кластерами Kubernetes.

---

### **1. Базовые команды**

1. **Проверка версии клиента и сервера Kubernetes:**
   ```bash
   kubectl version --client
   kubectl version --short
   ```

2. **Проверка текущего контекста и переключение между контекстами:**
   ```bash
   kubectl config current-context
   kubectl config use-context <context-name>
   ```

3. **Получение информации о кластере:**
   ```bash
   kubectl cluster-info
   ```

4. **Список всех доступных ресурсов:**
   ```bash
   kubectl api-resources
   ```

5. **Список всех объектов в кластере:**
   ```bash
   kubectl get all --all-namespaces
   ```

---

### **2. Работа с объектами Kubernetes**

#### **Получение информации**
1. **Получить все объекты определённого типа:**
   ```bash
   kubectl get pods
   kubectl get deployments
   kubectl get services
   ```

2. **Получить объекты в определённом namespace:**
   ```bash
   kubectl get pods -n <namespace>
   ```

3. **Получить подробную информацию о ресурсе:**
   ```bash
   kubectl describe pod <pod-name>
   kubectl describe deployment <deployment-name>
   ```

4. **Фильтрация вывода (JSONPath):**
   ```bash
   kubectl get pods -o jsonpath='{.items[*].metadata.name}'
   ```

#### **Создание, обновление и удаление**
1. **Создать объекты из YAML/JSON файла:**
   ```bash
   kubectl apply -f <file.yaml>
   ```

2. **Удалить объект:**
   ```bash
   kubectl delete pod <pod-name>
   kubectl delete -f <file.yaml>
   ```

3. **Редактирование объекта в кластере:**
   ```bash
   kubectl edit deployment <deployment-name>
   ```

4. **Обновить объект с помощью `apply`:**
   ```bash
   kubectl apply -f <file.yaml>
   ```

5. **Удалить все объекты определённого типа:**
   ```bash
   kubectl delete pods --all
   ```

---

### **3. Работа с Pod'ами**

1. **Логи Pod'а:**
   ```bash
   kubectl logs <pod-name>
   kubectl logs <pod-name> -c <container-name> # Если под содержит несколько контейнеров
   ```

2. **Просмотр логов в реальном времени:**
   ```bash
   kubectl logs -f <pod-name>
   ```

3. **Войти внутрь Pod'а (запуск интерактивной сессии):**
   ```bash
   kubectl exec -it <pod-name> -- /bin/bash
   kubectl exec -it <pod-name> -- /bin/sh
   ```

4. **Список Pod'ов с дополнительной информацией:**
   ```bash
   kubectl get pods -o wide
   ```

---

### **4. Отладка и диагностика**

1. **Проверка статуса Pod'а:**
   ```bash
   kubectl get pod <pod-name> -o yaml
   ```

2. **Проверка событий в namespace:**
   ```bash
   kubectl get events -n <namespace>
   ```

3. **Диагностика проблем с Pod'ом или Deployment'ом:**
   ```bash
   kubectl describe pod <pod-name>
   kubectl describe deployment <deployment-name>
   ```

4. **Сравнение текущего состояния с YAML-манифестом:**
   ```bash
   kubectl diff -f <file.yaml>
   ```

5. **Проверка доступности узлов:**
   ```bash
   kubectl get nodes
   kubectl describe nodes
   ```

---

### **5. Работа с Namespace**

1. **Список всех namespace:**
   ```bash
   kubectl get namespaces
   ```

2. **Создание namespace:**
   ```bash
   kubectl create namespace <namespace-name>
   ```

3. **Удаление namespace:**
   ```bash
   kubectl delete namespace <namespace-name>
   ```

4. **Работа в конкретном namespace:**
   ```bash
   kubectl get pods -n <namespace>
   ```

---

### **6. Работа с Deployment**

1. **Скалирование Deployment'а:**
   ```bash
   kubectl scale deployment <deployment-name> --replicas=<number>
   ```

2. **Обновление Deployment:**
   ```bash
   kubectl rollout restart deployment <deployment-name>
   ```

3. **Отмена последнего обновления Deployment:**
   ```bash
   kubectl rollout undo deployment <deployment-name>
   ```

4. **Статус обновления Deployment:**
   ```bash
   kubectl rollout status deployment <deployment-name>
   ```

---

### **7. Слежение за изменениями**

1. **Слежение за объектами в namespace:**
   ```bash
   kubectl get pods -n <namespace> -w
   ```

2. **Подписка на изменения ключей в etcd:**
   ```bash
   kubectl get configmap <configmap-name> -o yaml --watch
   ```

---

### **8. Генерация YAML-манифеста**

1. **Экспорт существующего объекта в YAML:**
   ```bash
   kubectl get pod <pod-name> -o yaml
   ```

2. **Создание шаблона для нового объекта:**
   ```bash
   kubectl create deployment <deployment-name> --image=<image-name> --dry-run=client -o yaml
   ```

---

### **9. Автодополнение команд**

Для включения автодополнения команд `kubectl`:
```bash
source <(kubectl completion bash) # Для bash
source <(kubectl completion zsh) # Для zsh
```

---

### **10. Полезные алиасы**

Для упрощения работы с `kubectl` можно настроить алиасы:
```bash
alias k="kubectl"
alias kgp="kubectl get pods"
alias kga="kubectl get all"
alias kdel="kubectl delete"
alias kaf="kubectl apply -f"
```

---

Эти команды помогут вам эффективно управлять кластерами Kubernetes и решать повседневные задачи.