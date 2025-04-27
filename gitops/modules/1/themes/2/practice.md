### Методичка по ДЗ: Модуль 1, Тема 2

**Название ДЗ:** Развертывание простого приложения

**Цель ДЗ:** Научиться описывать базовые объекты Kubernetes (Deployment, Service) с помощью YAML-манифестов и развертывать их в кластере.

**Необходимые условия/Инструменты:**

1.  Доступ к работающему кластеру Kubernetes. Это может быть локальный кластер (Minikube, Kind, Docker Desktop) или облачный кластер.
2.  Установленный и настроенный инструмент командной строки `kubectl`, который может взаимодействовать с вашим кластером.
3.  Текстовый редактор для написания YAML-файлов (VS Code, Sublime Text, Nano, Vim и т.д.).

**Подробные шаги:**

**Шаг 1: Создайте директорию для манифестов**

Создайте на своем компьютере директорию, где будут храниться YAML-файлы для этого ДЗ.

```bash
mkdir k8s-app-manifests
cd k8s-app-manifests
```

**Шаг 2: Напишите манифест Deployment**

Deployment отвечает за управление подами, гарантируя, что определенное количество реплик вашего приложения запущено.

Создайте файл с именем `app-deployment.yaml` в созданной директории. Вставьте в него следующий код (используем простой Nginx образ):

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-app
  labels:
    app: nginx
spec:
  replicas: 3 # Желаемое количество реплик подов
  selector:
    matchLabels:
      app: nginx # Селектор, который Deployment использует для поиска подов для управления
  template:
    metadata:
      labels:
        app: nginx # Лейбл для подов, который соответствует селектору Deployment
    spec:
      containers:
      - name: nginx # Имя контейнера
        image: nginx:latest # Docker образ для контейнера (можно использовать k8s.gcr.io/echoserver:1.10 тоже)
        ports:
        - containerPort: 80 # Порт, который приложение слушает внутри контейнера
```

*   **Пояснения:**
    *   `apiVersion`, `kind`, `metadata`: Стандартные поля для любого объекта Kubernetes.
    *   `spec.replicas`: Указывает, сколько подов должно быть запущено. Ставим `3`.
    *   `spec.selector.matchLabels`: Определяет, какие поды управляются этим Deployment. Он ищет поды с лейблом `app: nginx`.
    *   `spec.template`: Шаблон для создания новых подов.
    *   `spec.template.metadata.labels`: Лейблы, которые будут присвоены подам, созданным по этому шаблону. **Они должны совпадать с `spec.selector.matchLabels`!**
    *   `spec.template.spec.containers`: Список контейнеров в поде. Здесь у нас один контейнер `nginx`.
    *   `image`: Указывает Docker-образ. `nginx:latest` по умолчанию слушает на 80 порту.
    *   `containerPort`: Информационное поле, указывающее, какой порт открыт внутри контейнера.

**Шаг 3: Напишите манифест Service (ClusterIP)**

Service типа ClusterIP создает виртуальный IP-адрес внутри кластера, через который другие поды могут обращаться к вашему приложению по имени Service.

Создайте файл с именем `app-service-clusterip.yaml` в той же директории. Вставьте в него следующий код:

```yaml
# app-service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service-internal
spec:
  selector:
    app: nginx # Селектор, который Service использует для поиска подов, на которые будет направлять трафик
  ports:
  - protocol: TCP
    port: 80 # Порт Service (на который будут обращаться другие поды)
    targetPort: 80 # Порт на подах (containerPort)
  type: ClusterIP # Тип Service
```

*   **Пояснения:**
    *   `spec.selector`: Должен совпадать с лейблами подов вашего Deployment (`app: nginx`). Service будет направлять трафик на поды с этим лейблом.
    *   `spec.ports.port`: Порт, на который Service будет "слушать" внутри кластера.
    *   `spec.ports.targetPort`: Порт на поде, куда Service будет направлять трафик (совпадает с `containerPort` в Deployment).
    *   `spec.type: ClusterIP`: Указывает тип Service.

**Шаг 4: Напишите манифест Service (NodePort)**

Service типа NodePort открывает порт на каждой рабочей ноде кластера, который перенаправляет трафик на Service. Это один из способов получить доступ к приложению извне кластера (особенно удобен для тестирования на локальных кластерах).

Создайте файл с именем `app-service-nodeport.yaml`. Вставьте в него следующий код:

```yaml
# app-service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service-external
spec:
  selector:
    app: nginx # Селектор, должен совпадать с лейблами подов
  ports:
  - protocol: TCP
    port: 80 # Порт Service (на который будут обращаться другие поды)
    targetPort: 80 # Порт на подах (containerPort)
    # nodePort: 30000-32767 # Опционально: можно указать конкретный порт ноды, иначе он будет выбран автоматически
  type: NodePort # Тип Service
```

*   **Пояснения:**
    *   Этот Service также использует селектор `app: nginx` для выбора подов.
    *   `spec.type: NodePort`: Указывает тип Service. Kubernetes выберет доступный порт в диапазоне 30000-32767 на каждой ноде и свяжет его с этим Service.
    *   `nodePort`: Вы можете явно указать порт ноды (в диапазоне 30000-32767), если хотите, но обычно лучше позволить Kubernetes выбрать его автоматически.

**Шаг 5: Примените манифесты к кластеру**

Используйте `kubectl apply` для создания объектов в кластере.

```bash
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service-clusterip.yaml
kubectl apply -f app-service-nodeport.yaml
```

*   `kubectl apply -f <файл>`: Команда применяет конфигурацию из файла к кластеру. Она и создает объекты, и обновляет их, если файл уже применялся.

**Шаг 6: Проверьте состояние развернутых объектов**

Используйте команды `kubectl get` для проверки состояния Deployment, подов и Services.

```bash
kubectl get deploy my-nginx-app
kubectl get pods -l app=nginx # Получить поды с лейблом app=nginx
kubectl get svc my-nginx-service-internal my-nginx-service-external
```

*   **Ожидаемый вывод:**
    *   `kubectl get deploy`: Должен показать Deployment `my-nginx-app` с желаемым количеством реплик (`DESIRED`, `CURRENT`, `UP-TO-DATE`, `AVAILABLE` должны быть равны `3`).
    *   `kubectl get pods`: Должен показать 3 пода с именами, начинающимися на `my-nginx-app-...`, и статусом `Running`.
    *   `kubectl get svc`: Должен показать оба Service (`my-nginx-service-internal` и `my-nginx-service-external`) с соответствующими типами (`ClusterIP` и `NodePort`) и IP-адресами. У NodePort Service также будет указан порт ноды в колонке `PORT(S)`.

**Шаг 7: Проверьте доступность приложения (внешний доступ)**

Найдите IP-адрес одной из нод вашего кластера и NodePort вашего Service.

*   Если используете **Minikube**: `minikube ip` даст IP ноды. NodePort вы увидите в выводе `kubectl get svc my-nginx-service-external`.
*   Если используете **Kind**: Kind запускает кластер в Docker-контейнере. IP ноды обычно `127.0.0.1`. NodePort виден в `kubectl get svc`. Вам нужно будет пробросить порт контейнера Docker на хост, если он не проброшен по умолчанию. Или получить доступ через `kubectl port-forward service/my-nginx-service-external 8080:80` и обращаться к `localhost:8080`. **Для простоты и соответствия ДЗ, NodePort доступ через IP ноды+порт ноды - это основной метод проверки.** IP ноды можно получить, например, `kubectl get nodes -o wide`.
*   Если используете **Docker Desktop (Kubernetes)**: IP ноды обычно `127.0.0.1`. NodePort виден в `kubectl get svc`. Доступ будет по `127.0.0.1:<NodePort>`.
*   Если используете **облачный кластер (GKE, EKS, AKS)**: NodePort Service может быть доступен, но часто в облаке для внешнего доступа используются Service типа `LoadBalancer`. Если вы использовали тип `LoadBalancer` вместо `NodePort` (как опция в ДЗ), то `kubectl get svc my-nginx-service-external` покажет внешний IP в колонке `EXTERNAL-IP`.

Выполните команду `curl` или откройте браузер:

```bash
# Замените <Node-IP> на IP одной из нод, а <NodePort> на порт из вывода 'kubectl get svc' для NodePort Service
curl http://<Node-IP>:<NodePort>
```

или если у вас LoadBalancer Service:

```bash
# Замените <LoadBalancer-IP> на внешний IP LoadBalancer из вывода 'kubectl get svc'
curl http://<LoadBalancer-IP>
```

*   **Ожидаемый результат:** Вы должны получить HTML-код страницы приветствия Nginx.

**Проверка результатов (Измеряемость):**

1.  Предоставьте файлы `app-deployment.yaml`, `app-service-clusterip.yaml`, `app-service-nodeport.yaml`.
2.  Предоставьте вывод команд:
    ```bash
    kubectl get deploy my-nginx-app
    kubectl get pods -l app=nginx
    kubectl get svc my-nginx-service-internal my-nginx-service-external
    ```
3.  Предоставьте вывод команды `curl http://<IP>:<Port>` или скриншот браузера, показывающий страницу приветствия Nginx. Укажите IP и порт, который вы использовали.