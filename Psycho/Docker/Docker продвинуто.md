

---

# **Мастер-класс: Docker и Kubernetes для экспертов**

---

## **1. Docker на пределе возможностей**

### **1.1. Контейнеризация ядра: безопасность и изоляция**

Docker использует механизмы Linux на низком уровне для обеспечения безопасности и изоляции. Мы копнём глубже в детали.

#### **Namespaces: полная изоляция контейнера**

1. **PID namespace**:
    
    - Каждый контейнер видит только свои процессы, начиная с `PID=1`.
    - Вы можете отключить эту изоляцию:
        
        ```bash
        docker run --pid=host my-container
        ```
        
2. **Network namespace**:
    
    - Полностью изолирует сетевые интерфейсы контейнера.
    - Каждый контейнер получает свой `eth0`, но вы можете подключить его напрямую к сети:
        
        ```bash
        docker run --network=host my-container
        ```
        
3. **Mount namespace**:
    
    - Каждый контейнер видит только свою файловую систему.
    - Проверить монтированные ресурсы:
        
        ```bash
        docker inspect <container_id> | jq .Mounts
        ```
        

---

#### **Cgroups: управление ресурсами**

Cgroups контролируют использование ресурсов, чтобы контейнеры не "съели" всю машину.

1. **Лимит по CPU**:
    
    - Задайте долю процессорного времени:
        
        ```bash
        docker run --cpus=1.5 my-container
        ```
        
2. **Лимит по памяти**:
    
    - Укажите жёсткий лимит:
        
        ```bash
        docker run --memory=512m my-container
        ```
        
3. **Disk I/O**:
    
    - Ограничение скорости записи:
        
        ```bash
        docker run --device-write-bps /dev/sda:1mb my-container
        ```
        

#### **Проверка лимитов в реальном времени**

```bash
cat /sys/fs/cgroup/cpu/docker/<container_id>/cpu.stat
```

---

### **1.2. Сетевые магии Docker**

#### **Overlay-сети**

Overlay-сети соединяют контейнеры на разных хостах, используя VXLAN.

- **Создание overlay-сети**:
    
    ```bash
    docker network create --driver overlay my-overlay
    ```
    
- **Подключение сервисов к overlay**:
    
    ```bash
    docker service create --name my-service --network my-overlay my-image
    ```
    

#### **Трюки с DNS**

Docker автоматически добавляет контейнеры в DNS-сервис. Проверить DNS:

```bash
docker exec <container_id> cat /etc/resolv.conf
```

---

### **1.3. Multi-arch образы**

Собирайте образы для разных архитектур (например, `amd64`, `arm64`):

```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t my-app .
```

---

### **1.4. Модульность через плагины**

Docker поддерживает плагины для расширения возможностей:

1. **Сетевые плагины** (например, Calico, Flannel).
2. **Плагины для хранения** (например, Portworx, RexRay).

Установка плагина:

```bash
docker plugin install rexray/rexray
docker volume create -d rexray my-volume
```

---

## **2. Kubernetes на уровне сверхразумов**

---

### **2.1. Многокластерное управление Kubernetes**

#### **Федерация кластеров**

Федерация Kubernetes позволяет управлять несколькими кластерами как одним.

1. **Установка федерации**:
    
    - Используйте KubeFed:
        
        ```bash
        kubefedctl join my-cluster --host-cluster-context=my-context
        ```
        
2. **Пример декларации ресурса в федерации**:
    
    ```yaml
    apiVersion: types.kubefed.io/v1beta1
    kind: FederatedDeployment
    metadata:
      name: my-app
    spec:
      template:
        spec:
          replicas: 3
          template:
            spec:
              containers:
                - name: my-container
                  image: nginx
      placement:
        clusters:
          - name: cluster-1
          - name: cluster-2
    ```
    

---

### **2.2. Глубокая оптимизация Ingress**

#### **NGINX Ingress Controller**

1. Установка через Helm:
    
    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm install ingress-nginx ingress-nginx/ingress-nginx
    ```
    
2. Настройка баланса нагрузки:
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-ingress
    spec:
      rules:
        - host: my-app.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: my-service
                    port:
                      number: 80
    ```
    

---

### **2.3. CI/CD с Kubernetes**

#### **ArgoCD: управление GitOps**

1. Установка ArgoCD:
    
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
    
2. Пример синхронизации приложения:
    
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: my-app
    spec:
      project: default
      source:
        repoURL: https://github.com/my-repo.git
        targetRevision: HEAD
        path: deploy
      destination:
        server: https://kubernetes.default.svc
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
    ```
    

---

### **2.4. Мониторинг на продвинутом уровне**

#### **Prometheus Operator**

1. Установка через Helm:
    
    ```bash
    helm install prometheus-operator prometheus-community/kube-prometheus-stack
    ```
    
2. Создание ServiceMonitor:
    
    ```yaml
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: my-service-monitor
    spec:
      selector:
        matchLabels:
          app: my-app
      endpoints:
        - port: http
          interval: 30s
    ```
    

---

### **2.5. Глубокая работа с сетями**

#### **Network Policies**

1. Блокировка всего трафика, кроме разрешённого:
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: deny-all
    spec:
      podSelector: {}
      policyTypes:
        - Ingress
    ```
    
2. Разрешение трафика от определённых подов:
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-nginx
    spec:
      podSelector:
        matchLabels:
          app: nginx
      ingress:
        - from:
            - podSelector:
                matchLabels:
                  app: backend
    ```
    

---

### **2.6. Безопасность Kubernetes**

#### **Pod Security Policies (PSP)**

PSP ограничивают привилегии подов.

Пример запрета запуска с root:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
```

---

---


---

