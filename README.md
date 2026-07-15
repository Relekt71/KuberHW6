# Домашнее задание 2.3 - Настройка приложений и управление доступом в Kubernetes

## Задание 1: Работа с ConfigMap

### Описание
Развернуто приложение (nginx + multitool) с подключением веб-страницы через ConfigMap.

### Манифесты

#### configmap-web.yaml

    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: web-content
    namespace: default
    data:
    index.html: |
        <!DOCTYPE html>
        <html>
        <head>
        <title>Моя страница из ConfigMap</title>
        </head>
        <body>
        <h1>Привет от Kubernetes на Minikube!</h1>
        <p>Эта страница загружена из ConfigMap</p>
        <p>Время выполнения: 15.07.2026</p>
        </body>
        </html>

####  deployment.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: web-app
    labels:
        app: web-app
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: web-app
    template:
        metadata:
        labels:
            app: web-app
        spec:
        containers:
        - name: nginx
            image: nginx:alpine
            ports:
            - containerPort: 80
            volumeMounts:
            - name: web-content
            mountPath: /usr/share/nginx/html
        - name: multitool
            image: wbitt/network-multitool
            env:
            - name: HTTP_PORT
            value: "8080"
            ports:
            - containerPort: 8080
        volumes:
        - name: web-content
            configMap:
            name: web-content
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: web-app-service
    spec:
    selector:
        app: web-app
    ports:
    - name: nginx
        port: 80
        targetPort: 80
        protocol: TCP
    - name: multitool
        port: 8080
        targetPort: 8080
        protocol: TCP
    type: NodePort

Команды для развертывания

# Применение манифестов
    kubectl apply -f configmap-web.yaml
    kubectl apply -f deployment.yaml

# Проверка статуса
    kubectl get pods -l app=web-app
    kubectl get configmap web-content
    kubectl get service web-app-service

## Проверка работы

1. Проверка через curl внутри кластера:

        kubectl run test-pod --rm -it --image=busybox --restart=Never -- /bin/sh -c "wget -qO- http://web-app-service"

2. Проверка через NodePort:

# Получение URL для доступа

    minikube service web-app-service --url

# Или напрямую через curl

    curl http://$(minikube ip):$(kubectl get service web-app-service -o jsonpath='{.spec.ports[0].nodePort}')

3. Проверка через exec внутри пода:

        POD_NAME=$(kubectl get pods -l app=web-app -o jsonpath='{.items[0].metadata.name}')
        kubectl exec -it $POD_NAME -c nginx -- cat /usr/share/nginx/html/index.html

# Результат

<img width="933" height="247" alt="image" src="https://github.com/user-attachments/assets/09bf0957-a382-4991-bdb5-6d39c9a39d84" />

Рисунок 1 - Веб-страница, загруженная из ConfigMap

<img width="933" height="247" alt="image" src="https://github.com/user-attachments/assets/f76986c5-31a8-41ef-af7c-07341e3a1912" />

Рисунок 2 - Проверка через curl внутри кластера

# Задание 2: Настройка HTTPS с Secrets
##Описание
    Развернуто приложение с доступом по HTTPS с использованием самоподписанного сертификата.
    Команды генерации сертификатов

# Генерация самоподписанного SSL-сертификата
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout tls.key -out tls.crt -subj "/CN=myapp.example.com"

# Проверка созданных файлов
    ls -la tls.*

# Манифесты
## Создание Secret (через команду)

    kubectl create secret tls tls-secret --key tls.key --cert tls.crt

#### ingress-tls.yaml

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    name: web-app-ingress
    annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
    tls:
    - hosts:
        - myapp.example.com
        secretName: tls-secret
    rules:
    - host: myapp.example.com
        http:
        paths:
        - path: /
            pathType: Prefix
            backend:
            service:
                name: web-app-service
                port:
                number: 80

#### secret-tls.yaml (альтернативный способ)

    apiVersion: v1
    kind: Secret
    metadata:
    name: tls-secret
    namespace: default
    type: kubernetes.io/tls
    data:
    tls.crt: <base64-encoded-cert>
    tls.key: <base64-encoded-key>

## Команды для развертывания


# Применение манифестов
    kubectl apply -f ingress-tls.yaml

# Проверка статуса
    kubectl get ingress
    kubectl get secret tls-secret
    kubectl describe ingress web-app-ingress

# Настройка доступа

## Добавление хоста в /etc/hosts:

# Получение IP Minikube
    minikube ip

# Добавление записи (Linux/Mac)
    echo "$(minikube ip) myapp.example.com" | sudo tee -a /etc/hosts

# Для Windows (в PowerShell от администратора)
# Add-Content C:\Windows\System32\drivers\etc\hosts "$(minikube ip) myapp.example.com"

## Запуск туннеля (если ingress не отвечает):

# В отдельном терминале
    minikube tunnel

### Проверка HTTPS доступа

# Проверка через curl с игнорированием проверки сертификата
    curl -k https://myapp.example.com

# Проверка с выводом заголовков
    curl -k -v https://myapp.example.com

# Проверка через IP с указанием Host
    curl -k https://$(minikube ip) -H "Host: myapp.example.com"

## Результат

<img width="885" height="502" alt="image" src="https://github.com/user-attachments/assets/2a2738f9-db5d-4352-a0b9-b2a78aeff09b" />

Рисунок 3 - Проверка HTTPS доступа через curl

<img width="675" height="368" alt="image" src="https://github.com/user-attachments/assets/f5148f7f-fe95-41bf-9a84-c557b21a4d53" />

Рисунок 4 - Статус Ingress ресурса


# Задание 3: Настройка RBAC
## Описание

Создан пользователь developer с ограниченными правами (только просмотр логов и описания подов).
### Команды генерации сертификатов для пользователя

# 1. Генерация приватного ключа
    openssl genrsa -out developer.key 2048

# 2. Создание CSR (Certificate Signing Request)
    openssl req -new -key developer.key -out developer.csr -subj "/CN=developer"

# 3. Подпись сертификата с использованием CA Minikube
    openssl x509 -req -in developer.csr \
    -CA ~/.minikube/ca.crt \
    -CAkey ~/.minikube/ca.key \
    -CAcreateserial -out developer.crt -days 365

# 4. Проверка созданных файлов
    ls -la developer.*

### Манифесты
#### role-pod-reader.yaml

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
    name: pod-viewer
    namespace: default
    rules:
    - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

#### rolebinding-developer.yaml

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    name: developer-pod-viewer
    namespace: default
    subjects:
    - kind: User
    name: developer
    apiGroup: rbac.authorization.k8s.io
    roleRef:
    kind: Role
    name: pod-viewer
    apiGroup: rbac.authorization.k8s.io

### Команды для развертывания


# Применение манифестов
    kubectl apply -f role-pod-reader.yaml
    kubectl apply -f rolebinding-developer.yaml

# Проверка созданных ресурсов
    kubectl get role pod-viewer
    kubectl get rolebinding developer-pod-viewer

### Настройка контекста для пользователя


# Добавление пользователя в kubeconfig
    kubectl config set-credentials developer \
    --client-certificate=./developer.crt \
    --client-key=./developer.key

# Создание контекста
    kubectl config set-context developer-context \
    --cluster=minikube \
    --namespace=default \
    --user=developer

# Просмотр всех контекстов
    kubectl config get-contexts

# Переключение на контекст developer (опционально)
# kubectl config use-context developer-context

### Проверка прав
1. Просмотр подов (РАЗРЕШЕНО)

    kubectl get pods --as=developer

#### Результат:
    NAME                       READY   STATUS    RESTARTS   AGE
    web-app-79555c9bf5-nbs9f   2/2     Running   0          16m
    web-app-79555c9bf5-pz7k6   2/2     Running   0          16m

<img width="513" height="64" alt="image" src="https://github.com/user-attachments/assets/060d147f-a7a5-45e8-9246-d663b98f63aa" />

Рисунок 5 - Просмотр подов пользователем developer
2. Просмотр логов (РАЗРЕШЕНО)

# Просмотр логов nginx
    kubectl logs web-app-79555c9bf5-nbs9f -c nginx --as=developer --tail=5

# Просмотр логов multitool
    kubectl logs web-app-79555c9bf5-nbs9f -c multitool --as=developer --tail=5

# Просмотр логов всех подов с лейблом
    kubectl logs -l app=web-app --as=developer --tail=5

#### Результат:

    /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
    /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
    /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
    10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
    10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf

<img width="939" height="247" alt="image" src="https://github.com/user-attachments/assets/d2a2edb2-e247-4b43-8abf-7550037dfbc8" />

Рисунок 6 - Просмотр логов пользователем developer

3. Создание deployment (ЗАПРЕЩЕНО)

    kubectl create deployment test-deployment --image=nginx --as=developer

#### Результат:

    error: failed to create deployment: deployments.apps is forbidden: User "developer" cannot create resource "deployments" in API group "apps" in the namespace "default"

<img width="936" height="51" alt="image" src="https://github.com/user-attachments/assets/d6457723-4deb-4de8-a89d-61289243a8d9" />

Рисунок 7 - Ошибка при попытке создания deployment

4. Просмотр secrets (ЗАПРЕЩЕНО)

    kubectl get secrets --as=developer

#### Результат:

    Error from server (Forbidden): secrets is forbidden: User "developer" cannot list resource "secrets" in API group "" in the namespace "default"

<img width="936" height="51" alt="image" src="https://github.com/user-attachments/assets/3b626f3d-a31d-4db3-8d8e-8b39cc3e779a" />

Рисунок 8 - Ошибка при попытке просмотра secrets

5. Просмотр всех ресурсов (ЗАПРЕЩЕНО ❌)

    kubectl get all --as=developer

#### Результат:

    Error from server (Forbidden): replicationcontrollers is forbidden: User "developer" cannot list resource "replicationcontrollers" in API group "" in the namespace "default"
    Error from server (Forbidden): services is forbidden: User "developer" cannot list resource "services" in API group "" in the namespace "default"
    Error from server (Forbidden): daemonsets.apps is forbidden: User "developer" cannot list resource "daemonsets" in API group "apps" in the namespace "default"
    Error from server (Forbidden): deployments.apps is forbidden: User "developer" cannot list resource "deployments" in API group "apps" in the namespace "default"
    Error from server (Forbidden): replicasets.apps is forbidden: User "developer" cannot list resource "replicasets" in API group "apps" in the namespace "default"
    Error from server (Forbidden): statefulsets.apps is forbidden: User "developer" cannot list resource "statefulsets" in API group "apps" in the namespace "default"
    Error from server (Forbidden): horizontalpodautoscalers.autoscaling is forbidden: User "developer" cannot list resource "horizontalpodautoscalers" in API group "autoscaling" in the namespace "default"
    Error from server (Forbidden): cronjobs.batch is forbidden: User "developer" cannot list resource "cronjobs" in API group "batch" in the namespace "default"
    Error from server (Forbidden): jobs.batch is forbidden: User "developer" cannot list resource "jobs" in API group "batch" in the namespace "default"

<img width="933" height="367" alt="image" src="https://github.com/user-attachments/assets/8ef03b0f-808c-4cf4-a95b-5478324e2f6f" />

Рисунок 9 - Ошибки при попытке просмотра всех ресурсов
