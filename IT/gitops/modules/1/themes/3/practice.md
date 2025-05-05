### Методичка по ДЗ: Модуль 1, Тема 3

**Название ДЗ:** Настройка сетевого взаимодействия и внешнего доступа через Ingress

**Цель ДЗ:** Научиться настраивать взаимодействие между подами внутри кластера через Service ClusterIP и предоставлять внешний доступ к приложению с использованием Ingress.

**Необходимые условия/Инструменты:**

1.  Работающий кластер Kubernetes с установленным `kubectl`.
2.  Установленный Ingress-контроллер в кластере. **Важно:** Ingress-контроллер (например, Nginx Ingress Controller, Traefik, HAProxy Ingress) не входит в стандартную установку Kubernetes и должен быть установлен отдельно. Если в вашем кластере его еще нет, вам нужно его установить. Для локальных кластеров (Minikube, Kind) есть простые способы установки. *В рамках этого ДЗ предполагается, что Ingress-контроллер уже установлен или вы готовы его установить как часть ДЗ.* Мы будем ориентироваться на Nginx Ingress Controller как на один из самых распространенных.
3.  Текстовый редактор.

**Подробные шаги:**

**Шаг 1: Установите Ingress-контроллер (если его нет)**

*   **Для Minikube:** Включите аддон:
    ```bash
    minikube addons enable ingress
    ```
*   **Для Kind:** Следуйте официальной документации Kind для установки Nginx Ingress Controller. Обычно это сводится к применению манифеста из репозитория Nginx Ingress Controller:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/kind/deploy.yaml # Уточните актуальную версию
    ```
    Дождитесь, пока поды Ingress-контроллера будут запущены (`kubectl get pods -n ingress-nginx`).
*   **Для Docker Desktop (Kubernetes):** Включите Ingress в настройках Docker Desktop.
*   **Для облачных кластеров:** Следуйте инструкциям вашего облачного провайдера или документации выбранного Ingress-контроллера.

**Шаг 2: Создайте директорию для манифестов (если еще нет)**

Если вы не используете ту же директорию, что и для предыдущего ДЗ.

```bash
mkdir k8s-ingress-app
cd k8s-ingress-app
```

**Шаг 3: Напишите манифесты для Backend приложения**

Создайте файл `backend.yaml`. Это будет простое приложение, которое можно вызвать из frontend.

```yaml
# backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: k8s.gcr.io/echoserver:1.10 # Простой образ, который отвечает с информацией о поде
        ports:
        - containerPort: 8080 # echserver слушает на 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80 # Service порт
    targetPort: 8080 # Порт на поде
  type: ClusterIP # Internal service
```

*   Здесь мы создаем Deployment и Service типа ClusterIP для backend. Service будет называться `backend-service` и доступен другим подам внутри кластера по этому имени на порту 80.

**Шаг 4: Напишите манифесты для Frontend приложения**

Создайте файл `frontend.yaml`. Это будет приложение, которое будет обращаться к backend. Можно использовать другой echoserver или кастомный образ, если у вас есть. Для простоты возьмем еще один echoserver, но представим, что это frontend.

```yaml
# frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: k8s.gcr.io/echoserver:1.10 # Предположим, что этот frontend может делать запросы
        ports:
        - containerPort: 8080
```

*   Создаем только Deployment для frontend. Service ClusterIP не нужен для frontend, так как мы будем обращаться к нему извне через Ingress.

**Шаг 5: Напишите манифест Ingress**

Создайте файл `ingress.yaml`. Этот ресурс сообщит Ingress-контроллеру, как маршрутизировать внешний трафик.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-frontend-ingress
  annotations:
    # Это специфично для Nginx Ingress Controller, может отличаться для других контроллеров
    # nginx.ingress.kubernetes.io/rewrite-target: /$1 # Пример аннотации для перезаписи пути
spec:
  ingressClassName: nginx # Указывает, какой Ingress-контроллер должен обрабатывать этот Ingress
  rules:
  - http:
      paths:
      - path: /app # Путь URL, по которому будет доступен frontend
        pathType: Prefix # Тип совпадения пути (Prefix, Exact, ImplementationSpecific)
        backend:
          service:
            name: frontend-service # Имя Service, на который перенаправлять трафик
            port:
              number: 8080 # Порт этого Service
```

*   **Пояснения:**
    *   `apiVersion`, `kind`, `metadata`: Стандартные поля.
    *   `spec.ingressClassName`: **Обязательно** указывает имя IngressClass, который должен обработать этот Ingress. Для Nginx Ingress Controller, установленного стандартным способом, это обычно `nginx`. Убедитесь, что такая IngressClass существует в вашем кластере (`kubectl get ingressclass`).
    *   `spec.rules`: Определяет правила маршрутизации.
    *   `http.paths`: Список правил для HTTP-трафика.
    *   `path`: Путь в URL запроса (например, `/app`).
    *   `pathType`: Как сопоставлять путь. `Prefix` означает, что совпадет `/app`, `/app/`, `/app/something`.
    *   `backend`: Определяет, куда направлять трафик, если путь совпадает.
    *   `service.name`: Имя Service, на который направлять трафик. **Здесь есть ошибка в логике ДЗ:** Мы хотим направить трафик на Frontend Deployment, но Ingress направляет трафик ТОЛЬКО на **Service**. Frontend Deployment пока не имеет Service. Исправим это, добавив ClusterIP Service для Frontend.

**Шаг 5 (Исправлено): Напишите манифесты для Frontend приложения и Service**

Модифицируйте `frontend.yaml` или создайте отдельный файл для Service.

```yaml
# frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: k8s.gcr.io/echoserver:1.10 # Или ваш frontend образ
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend # Должен соответствовать лейблу пода Frontend Deployment
  ports:
  - protocol: TCP
    port: 8080 # Service порт (можно взять любой, например 80 или 8080)
    targetPort: 8080 # Порт на поде (containerPort)
  type: ClusterIP # Внешний доступ будет через Ingress, Service нужен только для Ingress
```

*   Теперь у Frontend Deployment есть соответствующий ClusterIP Service (`frontend-service`), на который Ingress сможет направлять трафик.

**Шаг 6: Примените манифесты к кластеру**

```bash
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml # Убедитесь, что он включает Deployment и Service для Frontend
kubectl apply -f ingress.yaml
```

Дождитесь, пока поды поднимутся (`kubectl get pods`).

**Шаг 7: Проверьте сетевое взаимодействие Pod-to-Service**

Выполните команду `kubectl exec` на поде frontend, чтобы сделать curl запрос к backend Service по его имени.

Сначала найдите имя пода frontend: `kubectl get pods -l app=frontend`

Затем выполните запрос:

```bash
# Замените <frontend-pod-name> на имя пода frontend
kubectl exec <frontend-pod-name> -- curl backend-service:80
```

*   **Пояснения:** Мы обращаемся по имени Service (`backend-service`) и порту Service (`80`). Kubernetes Service Discovery и kube-proxy обеспечивают маршрутизацию этого запроса на один из подов backend.
*   **Ожидаемый результат:** Вы должны получить ответ от echoserver-пода backend, содержащий информацию о запросе, включая IP-адрес пода frontend и имя хоста пода backend.

**Шаг 8: Проверьте внешний доступ через Ingress**

Найдите внешний IP-адрес или hostname вашего Ingress-контроллера.

*   **Для Minikube:** `minikube ip` даст IP.
*   **Для Kind:** Используйте `kubectl get services -n ingress-nginx` и посмотрите на Service типа LoadBalancer (если он есть) или NodePort. IP может быть `localhost`. Вам может понадобиться `kubectl port-forward` для доступа к Ingress-контроллеру, если LoadBalancer не работает или нет NodePort. Простейший способ - `kubectl port-forward service/ingress-nginx-controller 8080:80 -n ingress-nginx` и обращаться к `localhost:8080`.
*   **Для Docker Desktop:** IP обычно `127.0.0.1`.
*   **Для облачных кластеров:** `kubectl get ingress my-frontend-ingress` покажет внешний IP/Hostname в колонке `ADDRESS`.

Выполните команду `curl`, используя IP/Hostname Ingress-контроллера и путь, указанный в вашем Ingress-ресурсе (`/app`).

```bash
# Замените <Ingress-IP> на IP/Hostname вашего Ingress-контроллера
curl http://<Ingress-IP>/app
```

*   **Ожидаемый результат:** Вы должны получить ответ от echoserver-пода frontend. Если IngressController правильно настроен и ресурс Ingress применен, трафик с внешнего IP на путь `/app` будет перенаправлен на `frontend-service:8080`.

**Проверка результатов (Измеряемость):**

1.  Предоставьте YAML-файлы: `backend.yaml`, `frontend.yaml`, `ingress.yaml`.
2.  Предоставьте вывод команд:
    ```bash
    kubectl get deploy backend-app frontend-app
    kubectl get svc backend-service frontend-service
    kubectl get ingress my-frontend-ingress
    # Опционально: kubectl get pods -l app=backend,app=frontend
    ```
3.  Предоставьте вывод команды `kubectl exec <frontend-pod-name> -- curl backend-service:80`, демонстрирующий успешное взаимодействие между подами.
4.  Предоставьте вывод команды `curl http://<Ingress-IP>/app`, демонстрирующий успешный внешний доступ через Ingress. Укажите IP, который вы использовали.