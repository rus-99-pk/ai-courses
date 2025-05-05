Отлично! Мы подошли к сердцу современного DevOps - Kubernetes. Это сложная, но невероятно мощная платформа, которая позволяет управлять контейнеризованными приложениями в масштабе. В этом уроке мы развернем локальный кластер, изучим его основы и переведем наше приложение на рельсы Kubernetes.

---

**УРОК 9: Kubernetes: Оркестрация Контейнеров и GitOps**

**Цель:** Освоить фундаментальные концепции и инструменты Kubernetes, научиться деплоить и масштабировать контейнеризованные приложения в кластере, а также познакомиться с GitOps как современным подходом к управлению развертыванием.

**Результат урока:** У вас будет локальный Kubernetes кластер, в котором будет запущено ваше приложение как набор сущностей Kubernetes (Deployment, Service). Вы сможете управлять приложением с помощью `kubectl`, использовать Helm для упаковки и деплоя, и настроить базовый конвейер GitOps с помощью Argo CD.

**Предварительные требования:**

*   Успешно завершен Урок 1-8.
*   Ваше приложение упаковано в Docker образ и опубликовано в Container Registry (Nexus, Docker Hub, GitLab Registry - Урок 7, Lab 2 & 8).
*   Знание основ Docker и контейнеров.
*   Достаточно мощная рабочая машина или VM для запуска локального Kubernetes кластера (Minikube или Kind) - **минимум 8GB RAM, предпочтительно 16GB+**, несколько CPU ядер, достаточно свободного места на диске.

**Техническая подготовка (на Вашей рабочей машине или выделенной VM):**

1.  **Установите Docker:** Если вы работаете на своей машине, а не на VM, убедитесь, что Docker установлен. Minikube и Kind часто используют Docker для запуска нод.
2.  **Установите `kubectl`:** Утилита командной строки для взаимодействия с Kubernetes кластером.
    ```bash
    # Для Ubuntu/Debian
    sudo apt update
    sudo apt install -y apt-transport-https ca-certificates curl
    curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
    echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo apt update
    sudo apt install -y kubectl

    # Для CentOS/RHEL/Fedora
    cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude=kubelet kubeadm kubectl
    EOF
    sudo yum install -y kubectl --disableexcludes=kubernetes

    # Проверьте установку
    kubectl version --client
    ```
3.  **Установите Minikube или Kind:**
    *   **Minikube:** Легковесный Kubernetes, запускающий кластер в VM или непосредственно на хосте (с Docker, Podman и др.). Прост в установке и использовании.
        ```bash
        # Следуйте инструкциям на https://minikube.sigs.k8s.io/docs/start/
        # Пример для Linux:
        curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
        sudo install minikube-linux-amd64 /usr/local/bin/minikube
        ```
    *   **Kind (Kubernetes in Docker):** Запускает Kubernetes кластеры как набор контейнеров Docker. Очень удобен для локального тестирования и разработки CI/CD.
        ```bash
        # Следуйте инструкциям на https://kind.sigs.k8s.io/docs/user/quick-start/
        # Пример для Linux:
        curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64 # Замените версию на актуальную
        chmod +x ./kind
        sudo mv ./kind /usr/local/bin/kind
        ```
    *   **Выберите один** и используйте его для Lab 1. Kind часто быстрее стартует.

4.  **Установите Helm:** Менеджер пакетов для Kubernetes.
    ```bash
    # Следуйте инструкциям на https://helm.sh/docs/intro/install/
    # Пример для Linux:
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    ```
    Проверьте установку: `helm version`.

5.  **Установите Argo CD CLI:** Утилита командной строки для взаимодействия с Argo CD.
    ```bash
    # Следуйте инструкциям на https://argo-cd.readthedocs.io/en/stable/cli_install/
    # Пример для Linux:
    curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
    chmod +x /usr/local/bin/argocd
    ```
    Проверьте установку: `argocd version --client`.

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **Оркестрация контейнеров:** Автоматизированное управление жизненным циклом контейнеров (запуск, остановка, масштабирование), распределение их по хостам, обеспечение сетевого взаимодействия, хранение данных, самовосстановление при сбоях. Зачем нужна: ручное управление десятками/сотнями контейнеров невозможно.
    *   **Kubernetes (K8s):** Открытая система для автоматизации развертывания, масштабирования и управления контейнеризованными приложениями. Стандарт де-факто.
    *   **Docker Swarm:** Встроенный оркестратор в Docker. Проще, но менее функциональный и масштабируемый, чем Kubernetes.
    *   **Mesos/Mesos Marathon:** Более старые универсальные платформы для распределенных приложений, могут запускать и контейнеры.

*   **Архитектура Kubernetes:**
    *   **Control Plane (Master):** Управляет кластером. Включает:
        *   `kube-apiserver`: Фронтенд Kubernetes API. Все взаимодействие с кластером идет через него.
        *   `etcd`: Распределенное key-value хранилище, где хранится состояние кластера.
        *   `kube-scheduler`: Назначает Pods нодам.
        *   `kube-controller-manager`: Запускает различные контроллеры (Replication Controller, Endpoint Controller и др.), которые отслеживают состояние кластера и стремятся привести его к желаемому.
    *   **Worker Nodes:** Запускают контейнеры. Включают:
        *   `kubelet`: Агент на каждой ноде, взаимодействует с Control Plane и Container Runtime.
        *   `kube-proxy`: Обеспечивает сетевые правила и пересылку трафика к Service'ам.
        *   `Container Runtime` (например, containerd, CRI-O, Docker): Запускает и управляет контейнерами.

*   **kubectl:** Инструмент командной строки для взаимодействия с Kubernetes API Server. Позволяет деплоить приложения, проверять состояние кластера, управлять ресурсами.

*   **Основные сущности Kubernetes (Объекты API):**
    *   **Pod:** Наименьшая развертываемая единица в Kubernetes. Группа из одного или более контейнеров с общим сетевым пространством и хранилищем. Контейнеры в поде всегда работают вместе.
    *   **Deployment:** Декларативное описание желаемого состояния для Pods. Управляет созданием, обновлением и масштабированием Pods. Обеспечивает стратегии развертывания (RollingUpdate по умолчанию).
    *   **Service:** Абстракция, которая определяет логическую группу Pods и политику доступа к ним (часто IP и порт). Обеспечивает стабильную точку доступа к нестабильному набору Pods (которые могут пересоздаваться). Типы: `ClusterIP` (доступно только внутри кластера), `NodePort` (доступно по порту на каждой ноде кластера), `LoadBalancer` (интегрируется с внешним облачным балансировщиком), `ExternalName`.
    *   **Namespace:** Виртуальные кластеры внутри физического кластера. Используются для организации ресурсов и изоляции.
    *   **ConfigMap:** Объект для хранения нечувствительных данных конфигурации в виде пар ключ-значение или файлов.
    *   **Secret:** Объект для хранения чувствительных данных (пароли, токены, ключи) в Base64. **Не является надежным шифрованием, просто кодирование.** Требует дополнительных мер безопасности (Vault, External Secrets).

*   **Продвинутые сущности:** DaemonSet (запускает копию пода на каждой или выбранной ноде), StatefulSet (для приложений с состоянием, гарантирует порядок развертывания/удаления и стабильную идентификацию), Job (одноразовая задача), CronJob (задача по расписанию), Ingress (управляет внешним доступом к Service'ам по HTTP/HTTPS, предоставляя маршрутизацию на основе доменного имени/пути).

*   **Persistent Storage:** Поды эфемерны, их данные теряются при пересоздании. Для персистентных данных используются:
    *   `PersistentVolume` (PV): Абстракция физического хранилища в кластере. Управляется администратором кластера.
    *   `PersistentVolumeClaim` (PVC): Запрос пользователя на определенный объем хранилища с определенными характеристиками. Связывается с PV.
    *   `StorageClass`: Определяет "класс" хранилища (тип диска, производитель) и как динамически Provision'ить PV при запросе PVC.

*   **GitOps:** Операционная модель для доставки Cloud Native приложений. Использует Git как единый источник истины для декларативной инфраструктуры и приложений.
    *   **Принципы:**
        *   Декларативность: Все состояние системы описано декларативно (в YAML).
        *   Версионирование и неизменяемость: Желаемое состояние хранится в Git.
        *   Изменения через Pull Requests: Изменения применяются через ревью и слияние Pull/Merge Request'ов в Git.
        *   Автоматическое Pull-модель: Агент в кластере (например, Argo CD) постоянно сравнивает *реальное* состояние кластера с *желаемым* состоянием в Git и автоматически применяет изменения. Сравнение с Push-моделью (Jenkins/GitLab Runner пушит изменения в кластер): Pull-модель безопаснее (агенту не нужны полные права на кластер, ему нужны только права на чтение из Git и применение изменений в своем неймспейсе), более устойчива к сбоям сети, проще масштабируется.
    *   **Инструменты:** Argo CD, Flux CD.

*   **Облачные сервисы:**
    *   **IaaS (Infrastructure as a Service):** Виртуальные ресурсы (VM, хранилище, сеть) - вы управляете ОС и ПО. Пример: AWS EC2, S3, VPC.
    *   **PaaS (Platform as a Service):** Платформа для развертывания приложений, абстрагирующая ОС и нижележащую инфраструктуру. Пример: Heroku, Google App Engine. Managed Kubernetes (EKS, GKE, AKS) часто относят к PaaS или как отдельную категорию CaaS (Container as a Service).
    *   **SaaS (Software as a Service):** Готовое приложение, которым вы пользуетесь через интернет. Пример: Gmail, Slack.

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Развертывание Локального Kubernetes Кластера (Minikube или Kind)**

*   **Цель:** Запустить однонодовый или многонодовый (в контейнерах) локальный кластер Kubernetes.
*   **Предварительные требования:**
    *   Установленные `kubectl` и Minikube или Kind (Техническая подготовка).
    *   Установленный Docker.
*   **Шаги (Выберите один инструмент):**

    *   **Minikube:**
        ```bash
        # Удалить старый кластер, если есть
        minikube delete

        # Запустить новый кластер, используя драйвер Docker (если Minikube на хосте с Docker)
        # Или использовать драйвер virtualbox/vmware/kvm (если Minikube в VM)
        minikube start --driver=docker --memory 8192 --cpus 4 # Укажите ресурсы по вашим возможностям
        ```
        Minikube скачает образ Kubernetes и запустит Control Plane и Worker компоненты в Docker контейнере или VM.

    *   **Kind:**
        ```bash
        # Удалить старый кластер, если есть
        kind delete cluster --name my-devops-cluster

        # Создать новый кластер (по умолчанию single-node cluster в Docker контейнере)
        kind create cluster --name my-devops-cluster
        ```
        Kind создаст несколько Docker контейнеров, один для Control Plane, другие для Worker нод (по умолчанию один Worker).

    2.  **Проверьте статус кластера:**
        ```bash
        kubectl cluster-info
        ```
        Вы должны увидеть адреса API Server и CoreDNS.

    3.  **Проверьте ноды кластера:**
        ```bash
        kubectl get nodes
        ```
        Должна быть хотя бы одна нода в статусе `Ready`.

    4.  **(Для Minikube) Получите доступ к кластеру:** Если вы запускали Minikube в VM, вам может понадобиться настроить доступ к сервисам через `minikube tunnel` или `minikube service`.

**Результат Lab 1:** У вас есть запущенный локальный Kubernetes кластер, и ваш `kubectl` настроен для взаимодействия с ним.

---

**Lab 2: Практика kubectl**

*   **Цель:** Изучить основные команды `kubectl` для просмотра ресурсов и состояния кластера.
*   **Предварительные требования:**
    *   Работающий Kubernetes кластер (Lab 1).
*   **Шаги:**

    1.  **Просмотр ресурсов:**
        *   Посмотреть все Pods во всех неймспейсах: `kubectl get pods --all-namespaces`
        *   Посмотреть Pods в текущем неймспейсе (`default`): `kubectl get pods`
        *   Посмотреть Deployments, Services, ConfigMaps в текущем неймспейсе: `kubectl get deployments`, `kubectl get services`, `kubectl get configmaps`
        *   Посмотреть все типы ресурсов в текущем неймспейсе: `kubectl get all`
        *   Посмотреть ресурсы в конкретном неймспейсе: `kubectl get pods -n kube-system`
        *   Посмотреть список неймспейсов: `kubectl get namespaces`

    2.  **Получение подробной информации:**
        *   Посмотреть подробности о конкретном ресурсе: `kubectl describe pod <имя_пода>` (или `deployment`, `service` и т.д.)
        *   Посмотреть подробности о ноде: `kubectl describe node <имя_ноды>`

    3.  **Работа с логами контейнеров:**
        *   Посмотреть логи контейнера в поде: `kubectl logs <имя_пода>` (если в поде один контейнер)
        *   Посмотреть логи конкретного контейнера в поде: `kubectl logs <имя_пода> -c <имя_контейнера>`
        *   Следить за логами в реальном времени: `kubectl logs -f <имя_пода>`

    4.  **Выполнение команд в контейнере:**
        *   Подключиться к оболочке контейнера (если в образе есть оболочка): `kubectl exec -it <имя_пода> -- /bin/bash` (или `/bin/sh`)
        *   Выполнить одну команду в контейнере: `kubectl exec <имя_пода> -- ls -l /app`

    5.  **Управление ресурсами с помощью YAML:**
        *   Создать файл с определением Pod'а: `nano simple_pod.yaml`
            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: my-simple-pod
            spec:
              containers:
              - name: my-app-container
                image: ВАШ_IP_VM_NEXUS:8082/my-java-app:1.0.0-SNAPSHOT # Или ваш образ из Docker Hub
                ports:
                - containerPort: 8080
            ```
        *   Применить манифест: `kubectl apply -f simple_pod.yaml` (создаст Pod)
        *   Проверить статус Pod'а: `kubectl get pod my-simple-pod`
        *   Удалить Pod: `kubectl delete -f simple_pod.yaml` (или `kubectl delete pod my-simple-pod`)

    6.  **Форматирование вывода:**
        *   Получить информацию в YAML формате: `kubectl get pod my-simple-pod -o yaml`
        *   Получить информацию в JSON формате: `kubectl get pod my-simple-pod -o json`

**Результат Lab 2:** Уверенное использование `kubectl` для просмотра информации о кластере и ресурсах, работы с логами, выполнения команд в контейнерах и базового управления ресурсами с помощью YAML манифестов.

---

**Lab 3: Деплой Приложения с помощью Deployment**

*   **Цель:** Упаковать ваше приложение в Deployment для автоматического управления Pods.
*   **Предварительные требования:**
    *   Работающий Kubernetes кластер (Lab 1).
    *   Ваш Docker образ опубликован в Registry и доступен из кластера.
    *   Умение использовать `kubectl apply`.
*   **Шаги:**

    1.  **Создайте файл манифеста для Deployment:**
        ```bash
        nano myapp_deployment.yaml
        ```
    2.  **Напишите определение Deployment:**

        ```yaml
        # myapp_deployment.yaml

        apiVersion: apps/v1 # API версия для Deployment
        kind: Deployment # Тип ресурса
        metadata:
          name: my-java-app-deployment # Имя Deployment
          labels: # Метки (лейблы) для организации ресурсов
            app: my-java-app
            tier: backend
        spec:
          replicas: 2 # Желаемое количество Pods (экземпляров приложения)
          selector: # Селектор для определения, какие Pods управляются этим Deployment
            matchLabels:
              app: my-java-app # Выбирать Pods с меткой app: my-java-app
          strategy: # Стратегия развертывания (RollingUpdate по умолчанию)
            type: RollingUpdate
            rollingUpdate:
              maxSurge: 1 # На сколько Pods можно превысить желаемое количество во время обновления
              maxUnavailable: 0 # Сколько Pods может быть недоступно во время обновления
          template: # Шаблон для создания Pods
            metadata:
              labels: # Метки, которые будут присвоены Pods, созданным по этому шаблону (должны соответствовать selector)
                app: my-java-app
                tier: backend
            spec:
              containers:
              - name: my-app-container # Имя контейнера (внутри Pod)
                image: ВАШ_IP_VM_NEXUS:8082/my-java-app:1.0.0-SNAPSHOT # Образ контейнера
                ports:
                - containerPort: 8080 # Порт, который слушает контейнер
                  name: http # Имя порта (опционально)
                imagePullPolicy: Always # Всегда скачивать образ (полезно для SNAPSHOT/latest)

                # Опционально: Пробы работоспособности (Liveness/Readiness Probes)
                # livenessProbe: # Перезапустить контейнер, если проба не прошла
                #  httpGet:
                #    path: /actuator/health/liveness # Путь к Liveness endpoint
                #    port: 8080
                #  initialDelaySeconds: 60 # Ждать 60 секунд перед первой проверкой
                #  periodSeconds: 10 # Проверять каждые 10 секунд
                # readinessProbe: # Не направлять трафик на Pod, пока проба не прошла
                #  httpGet:
                #    path: /actuator/health/readiness # Путь к Readiness endpoint
                #    port: 8080
                #  initialDelaySeconds: 60
                #  periodSeconds: 10
                #  timeoutSeconds: 5

                # Опционально: Ограничение ресурсов
                # resources:
                #  requests: # Минимально необходимые ресурсы
                #    memory: "256Mi"
                #    cpu: "200m" # 200 milliCPU (0.2 CPU)
                #  limits: # Максимально разрешенные ресурсы (контейнер будет убит при превышении)
                #    memory: "512Mi"
                #    cpu: "500m"
        ```
        *   Замените путь к образу на ваш образ из Container Registry. Укажите правильный порт приложения.

    3.  **Примените манифест Deployment:**
        ```bash
        kubectl apply -f myapp_deployment.yaml
        ```

    4.  **Проверьте созданные ресурсы:**
        ```bash
        kubectl get deployments # Должен появиться my-java-app-deployment
        kubectl get replicasets # Deployment управляет ReplicaSet'ом
        kubectl get pods -l app=my-java-app # Должны появиться Pods с указанными метками
        ```
        Подождите, пока Pods перейдут в статус `Running`.

    5.  **Просмотрите логи Pods:** Используйте `kubectl logs <имя_пода>`.

    6.  **Потренируйтесь в масштабировании:**
        *   Измените количество реплик в файле `myapp_deployment.yaml` (например, на `replicas: 5`).
        *   Повторно примените манифест: `kubectl apply -f myapp_deployment.yaml`.
        *   Проверьте статус Pods (`kubectl get pods`). Kubernetes Controller Manager создаст новые Pods, чтобы достичь желаемого количества.
        *   Измените количество реплик через команду: `kubectl scale deployment my-java-app-deployment --replicas=1`. Проверьте статус Pods.

    7.  **Потренируйтесь в обновлении (Rolling Update):**
        *   Измените тег образа в файле `myapp_deployment.yaml` на новый (например, если вы опубликовали новую версию образа).
        *   Примените манифест: `kubectl apply -f myapp_deployment.yaml`.
        *   Наблюдайте за процессом обновления (`kubectl get pods -w`). Kubernetes будет постепенно создавать новые Pods с новым образом и удалять старые, обеспечивая обновление без простоя (с учетом `maxSurge` и `maxUnavailable`).

**Результат Lab 3:** Вы успешно развернули ваше приложение в Kubernetes с помощью Deployment, научились управлять его масштабом и выполнять Rolling Updates.

---

**Lab 4: Предоставление Доступа с помощью Service**

*   **Цель:** Создать Service для обеспечения стабильного доступа к вашим Pods.
*   **Предварительные требования:**
    *   Работающий Deployment с Pods (Lab 3).
*   **Шаги:**

    1.  **Создайте файл манифеста для Service:**
        ```bash
        nano myapp_service.yaml
        ```
    2.  **Напишите определение Service (тип NodePort):**

        ```yaml
        # myapp_service.yaml

        apiVersion: v1 # API версия для Service
        kind: Service # Тип ресурса
        metadata:
          name: my-java-app-service # Имя Service
          labels:
            app: my-java-app
            tier: backend
        spec:
          selector: # Селектор для определения, какие Pods относятся к этому Service
            app: my-java-app # Должен соответствовать меткам на Pods (из Deployment template)
          ports:
            - protocol: TCP
              port: 80 # Порт, который слушает Service внутри кластера
              targetPort: 8080 # Порт, который слушает контейнер в Pod'ах (containerPort)
              nodePort: 30000 # Порт на нодах кластера (в диапазоне 30000-32767). Доступно извне.
                               # Для Minikube/Kind это часто единственный способ получить доступ извне без Ingress.
          type: NodePort # Тип Service. Для доступа извне.
          # Другие типы: ClusterIP (только внутри кластера), LoadBalancer (для облачных провайдеров)
        ```

    3.  **Примените манифест Service:**
        ```bash
        kubectl apply -f myapp_service.yaml
        ```

    4.  **Проверьте созданный Service:**
        ```bash
        kubectl get services
        ```
        Вы увидите `my-java-app-service` с ClusterIP, INTERNAL-IP и PORT(S), включая NODEPORT.

    5.  **Получите доступ к приложению извне кластера:**
        *   Если вы используете Minikube: Получите адрес ноды: `minikube ip`. Приложение будет доступно по адресу `IP_НОДЫ:NODEPORT` (например, `192.168.49.2:30000`). Можете использовать `minikube service my-java-app-service`, чтобы Minikube автоматически открыл URL в браузере.
        *   Если вы используете Kind: Получите IP ноды (это IP Docker контейнера): `docker inspect -f '{{.NetworkSettings.IPAddress}}' <имя_контейнера_ноды_kind>`. Приложение будет доступно по адресу `IP_НОДЫ:NODEPORT`.
        *   Если вы используете облачный Managed K8s: Тип `LoadBalancer` создаст внешний балансировщик с публичным IP/hostname.

**Результат Lab 4:** Ваше приложение теперь доступно извне Kubernetes кластера через Service типа NodePort.

---

**Lab 5: Управление Конфигурацией и Секретами с ConfigMaps и Secrets**

*   **Цель:** Перенести конфигурацию приложения (нечувствительные данные) в ConfigMap, а секреты (учетные данные к БД, токен Vault) в Secret Kubernetes.
*   **Предварительные требования:**
    *   Работающий Deployment (Lab 3).
    *   Понимание, какие параметры приложения являются конфигом, а какие секретами.
    *   Знание учетных данных к БД и Vault токена.
    *   Ваше приложение должно уметь читать конфигурацию из переменных окружения (Twelve-Factor App).
*   **Шаги:**

    1.  **Создайте файл манифеста для ConfigMap:**
        ```bash
        nano myapp_configmap.yaml
        ```
    2.  **Напишите определение ConfigMap:**

        ```yaml
        # myapp_configmap.yaml

        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: my-java-app-config # Имя ConfigMap
          labels:
            app: my-java-app
        data: # Данные конфигурации (ключ-значение)
          application.properties: | # Пример: содержимое файла конфигурации
            server.port=8080
            app.greeting=Hello from ConfigMap!
          # Или просто пары ключ-значение
          # app_port: "8080"
          # app_greeting: "Hello from ConfigMap!"
        ```

    3.  **Создайте файл манифеста для Secret:** **Не храните явные секреты в Git!** Создайте файл манифеста, но сами значения секретов добавите через команду `kubectl` или Helm (в следующей Lab).

        ```yaml
        # myapp_secret.yaml (не храните его с реальными значениями в Git!)

        apiVersion: v1
        kind: Secret
        metadata:
          name: my-java-app-secrets # Имя Secret
          labels:
            app: my-java-app
        type: Opaque # Тип секрета (произвольные данные)
        # Данные секретов в Base64. НЕ ДОБАВЛЯЙТЕ ИХ СЮДА ВРУЧНУЮ, используйте kubectl!
        # data:
        #   db_password: <ваш_пароль_к_БД_в_base64>
        #   vault_token: <ваш_vault_токен_в_base64>
        ```

    4.  **Примените манифест ConfigMap:**
        ```bash
        kubectl apply -f myapp_configmap.yaml
        ```

    5.  **Создайте Secret с использованием `kubectl` (безопасно):**
        ```bash
        # Используем echo и base64 для кодирования пароля и токена
        kubectl create secret generic my-java-app-secrets \
          --from-literal=db_password='ваше_значение_пароля_к_БД' \
          --from-literal=vault_token='ваше_значение_vault_токена' \
          --dry-run=client -o yaml > myapp_secret.yaml # Создать файл манифеста (без применения)

        # Или сразу применить (если вы не храните myapp_secret.yaml в Git)
        kubectl create secret generic my-java-app-secrets \
          --from-literal=db_password='ваше_значение_пароля_к_БД' \
          --from-literal=vault_token='ваше_значение_vault_токена'
        ```
        **Если вы хотите хранить `myapp_secret.yaml` в Git (что не рекомендуется для реальных секретов), используйте внешний инструмент, который будет инжектировать секреты в кластер из Vault или другого безопасного хранилища.** Для этой Lab мы можем просто создать Secret вручную один раз или использовать Base64 значения в файле (помня о риске).

        ```bash
        # Альтернативно, если готовы временно захардкодить Base64 в файле (ТОЛЬКО ДЛЯ LAB!):
        # echo -n 'ваше_значение_пароля_к_БД' | base64
        # echo -n 'ваше_значение_vault_токена' | base64
        # Вставьте полученные Base64 значения в myapp_secret.yaml в секцию data
        # Затем примените: kubectl apply -f myapp_secret.yaml
        ```

    6.  **Проверьте созданные ConfigMap и Secret:**
        ```bash
        kubectl get configmaps
        kubectl get secrets
        kubectl describe configmap my-java-app-config
        kubectl describe secret my-java-app-secrets # Значения секретов будут скрыты
        # Чтобы посмотреть значения секретов (опасная команда!):
        # kubectl get secret my-java-app-secrets -o jsonpath='{.data.db_password}' | base64 --decode
        ```

    7.  **Измените манифест Deployment (`myapp_deployment.yaml`) для использования ConfigMap и Secret как переменных окружения:**

        ```yaml
        # myapp_deployment.yaml (обновленный)
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-java-app-deployment
          labels:
            app: my-java-app
            tier: backend
        spec:
          # ... (replicas, selector, strategy) ...
          template:
            metadata:
              labels:
                app: my-java-app
                tier: backend
            spec:
              containers:
              - name: my-app-container
                image: ВАШ_IP_VM_NEXUS:8082/my-java-app:1.0.0-SNAPSHOT
                ports:
                - containerPort: 8080
                  name: http
                imagePullPolicy: Always

                # Инжекция переменных окружения из ConfigMap и Secret
                env:
                - name: SPRING_DATASOURCE_URL # Имя переменной окружения в контейнере
                  value: "jdbc:postgresql://IP_DB_VM:5432/myappdb" # Пока укажем IP DB VM явно, лучше вынести в ConfigMap или Value в Helm
                - name: SPRING_DATA_MONGODB_URI # Имя переменной окружения
                  value: "mongodb://rootuser:rootpassword@IP_DB_VM:27017/myappmongodb?authSource=admin" # Пока явно

                # Получаем пароль БД из Secret
                - name: SPRING_DATASOURCE_PASSWORD # Имя переменной окружения в контейнере
                  valueFrom: # Получить значение из другого ресурса
                    secretKeyRef: # Из Secret по ключу
                      name: my-java-app-secrets # Имя Secret ресурса
                      key: db_password # Ключ внутри Secret

                # Получаем Vault Token из Secret
                - name: VAULT_TOKEN_VAR # Имя переменной окружения в контейнере (если приложение/обертка использует токен)
                  valueFrom:
                    secretKeyRef:
                      name: my-java-app-secrets
                      key: vault_token

                # Получаем адрес Vault из ConfigMap (или просто указываем явно, если Vault статический)
                # Если Vault запущен в K8s, можно использовать его Service Name
                - name: VAULT_ADDR
                  value: "http://IP_VAULT_VM:8200" # Укажите IP Vault VM

                # Получаем другие конфигурационные параметры из ConfigMap
                - name: APP_GREETING # Имя переменной окружения
                  valueFrom:
                    configMapKeyRef: # Из ConfigMap по ключу
                      name: my-java-app-config # Имя ConfigMap ресурса
                      key: app_greeting # Ключ внутри ConfigMap

                # Опционально: монтирование ConfigMap как файла
                # volumeMounts:
                # - name: config-volume
                #  mountPath: /app/config # Путь внутри контейнера
                # readOnly: true
                # volumes:
                # - name: config-volume
                #  configMap:
                #    name: my-java-app-config
                #    items:
                #    - key: application.properties # Имя ключа в ConfigMap
                #      path: application.properties # Имя файла внутри контейнера


                # ... (livenessProbe, readinessProbe, resources) ...
        ```
        *   **Важно:** Убедитесь, что имена переменных окружения (`SPRING_DATASOURCE_PASSWORD`, `VAULT_TOKEN_VAR` и т.д.) совпадают с теми, которые ожидает ваше приложение или скрипт-обертка (если вы его используете внутри контейнера). Убедитесь, что имена `ConfigMapKeyRef` и `SecretKeyRef` совпадают с именами ваших ресурсов и ключей в них.

    8.  **Примените обновленный манифест Deployment:**
        ```bash
        kubectl apply -f myapp_deployment.yaml
        ```
        Kubernetes выполнит Rolling Update, создавая новые Pods с обновленной конфигурацией переменных окружения.

    9.  **Проверьте Pods и логи:** Убедитесь, что новые Pods запущены. Посмотрите их логи, чтобы проверить, что приложение успешно стартовало и, например, подключилось к БД. Проверьте, что переменные окружения в контейнере установлены правильно: `kubectl exec <имя_пода> -- env`.

**Результат Lab 5:** Вы научились использовать ConfigMaps и Secrets для централизованного управления конфигурацией и секретами вашего приложения в Kubernetes, инжектируя их как переменные окружения в Pods.

---

**Lab 6: Настройка Persistent Storage**

*   **Цель:** Использовать PersistentVolumeClaim для предоставления персистентного хранилища для Pods, если это необходимо (например, для логов, данных БД - хотя БД лучше запускать вне K8s или использовать операторы StatefulSets).
*   **Предварительные требования:**
    *   Работающий Kubernetes кластер (Lab 1).
    *   Понимание концепций PV, PVC, StorageClass.
    *   Ваше приложение или другой компонент, который пишет данные на диск и нуждается в их сохранении.
*   **Шаги:**

    1.  **Проверьте доступные StorageClass:**
        ```bash
        kubectl get storageclass
        ```
        Minikube и Kind обычно предоставляют StorageClass по умолчанию. Если его нет, вам может потребоваться настроить Provisioner (например, `hostpath` для Minikube, но это только для локальной разработки, не для production).

    2.  **Создайте файл манифеста для PersistentVolumeClaim:**
        ```bash
        nano myapp_pvc.yaml
        ```
    3.  **Напишите определение PVC:**

        ```yaml
        # myapp_pvc.yaml

        apiVersion: v1
        kind: PersistentVolumeClaim # Запрос на хранилище
        metadata:
          name: myapp-logs-pvc # Имя PVC
          labels:
            app: my-java-app
        spec:
          accessModes: # Режимы доступа к хранилищу (один или несколько)
            - ReadWriteOnce # Может быть примонтирован как чтение/запись одной нодой
            # - ReadOnlyMany # Может быть примонтирован как только чтение многими нодами
            # - ReadWriteMany # Может быть примонтирован как чтение/запись многими нодами
          resources: # Запрашиваемые ресурсы
            requests:
              storage: 1Gi # Запросить 1 ГиБ хранилища
          storageClassName: standard # Имя StorageClass для динамического Provisioning (проверьте доступные в вашем кластере)
          # Если нет StorageClass по умолчанию или вы хотите использовать конкретный
        ```

    4.  **Примените манифест PVC:**
        ```bash
        kubectl apply -f myapp_pvc.yaml
        ```
        Kubernetes попытается найти или создать PersistentVolume, соответствующий запросу.

    5.  **Проверьте статус PVC и PV:**
        ```bash
        kubectl get pvc
        kubectl get pv
        ```
        PVC должен перейти в статус `Bound`, связанный с созданным или найденным PV.

    6.  **Измените манифест Deployment (`myapp_deployment.yaml`) для использования PVC:** Смонтируйте PVC в каталог внутри контейнера.

        ```yaml
        # myapp_deployment.yaml (обновленный)
        apiVersion: apps/v1
        kind: Deployment
        # ...
        spec:
          template:
            # ...
            spec:
              containers:
              - name: my-app-container
                # ... (image, ports, env, probes, resources) ...

                # Монтирование Volume, связанного с PVC
                volumeMounts:
                - name: logs-storage # Имя VolumeMount
                  mountPath: /app/logs # Путь внутри контейнера, куда монтировать хранилище
                  # readOnly: true # Если только чтение

              # Определение Volumes для Pod
              volumes:
              - name: logs-storage # Имя Volume (должно совпадать с volumeMounts)
                persistentVolumeClaim: # Ссылка на PVC
                  claimName: myapp-logs-pvc # Имя PVC, созданного ранее

        ```
        *   **Важно:** Убедитесь, что `mountPath` соответствует каталогу, куда ваше приложение пишет логи или другие данные.

    7.  **Примените обновленный манифест Deployment:**
        ```bash
        kubectl apply -f myapp_deployment.yaml
        ```
        Kubernetes выполнит Rolling Update. Новые Pods будут созданы с примонтированным хранилищем из PVC.

    8.  **Проверьте Pods:** Убедитесь, что новые Pods запущены и не имеют ошибок, связанных с монтированием Volume.

    9.  **Проверьте персистентность:** Сгенерируйте логи в приложении. Удалите Pod (`kubectl delete pod <имя_пода>`). Deployment создаст новый Pod на его месте. Проверьте логи в новом Pod'е - старые логи должны сохраниться, т.к. Volume персистентен.

**Результат Lab 6:** Вы научились использовать PersistentVolumeClaims для предоставления персистентного хранилища вашим Pods, что важно для приложений, работающих с состоянием.

---

**Lab 7: Упаковка Приложения с Helm**

*   **Цель:** Создать Helm Chart для вашего приложения, который упрощает его деплой и настройку в Kubernetes.
*   **Предварительные требования:**
    *   Установленный Helm (Техническая подготовка).
    *   Манифесты K8s для вашего приложения (Deployment, Service, ConfigMap, PVC - Lab 3, 4, 5, 6).
    *   Локальный Git репозиторий вашего приложения.
*   **Шаги:**

    1.  **Создайте базовый Helm Chart:** Убедитесь, что вы находитесь вне каталога `my-app`, но в корне Git репозитория `my-devops-app`.
        ```bash
        cd my-devops-app
        helm create my-app-chart # Создаст каталог my-app-chart
        ```
        Helm создаст скелет Chart'а с примерами манифестов в каталоге `my-app-chart/templates`.

    2.  **Изучите структуру Helm Chart:**
        *   `my-app-chart/Chart.yaml`: Метаданные Chart'а.
        *   `my-app-chart/values.yaml`: Значения по умолчанию для параметров Chart'а.
        *   `my-app-chart/templates/`: Каталог с шаблонами манифестов Kubernetes (используется синтаксис шаблонов Go и функции Sprig).
        *   `my-app-chart/templates/NOTES.txt`: Текст, который выводится после успешного деплоя.
        *   `my-app-chart/.helmignore`: Файлы, которые игнорировать при упаковке Chart'а.

    3.  **Замените примеры манифестов своими манифестами:**
        *   Удалите примеры манифестов из `my-app-chart/templates/` (кроме `NOTES.txt`, `_helpers.tpl` если есть).
        *   Скопируйте ваши файлы `myapp_deployment.yaml`, `myapp_service.yaml`, `myapp_configmap.yaml`, `myapp_pvc.yaml` в каталог `my-app-chart/templates/`.

    4.  **Параметризуйте манифесты с помощью шаблонов Helm:** Замените жестко заданные значения (например, имя образа, количество реплик, порты, значения конфигов/секретов) на ссылки на переменные из `values.yaml`.

        *   **В `my-app-chart/values.yaml`:** Добавьте переменные с значениями по умолчанию:
            ```yaml
            # my-app-chart/values.yaml

            replicaCount: 2 # Количество Pods

            image: # Настройки образа
              repository: ВАШ_IP_VM_NEXUS:8082/my-java-app
              pullPolicy: Always
              tag: "1.0.0-SNAPSHOT" # Тег образа по умолчанию

            imagePullSecrets: [] # Секрет для скачивания образа из приватного Registry (если нужен)
            # name: "my-registry-secret"

            nameOverride: "" # Переопределить имя Chart'а в релизе
            fullnameOverride: ""

            service: # Настройки Service
              type: NodePort
              port: 80
              targetPort: 8080
              nodePort: 30000 # Для NodePort

            ingress: # Настройки Ingress (позже)
              enabled: false
              className: ""
              annotations: {}
              hosts:
                - host: chart-example.local
                  paths:
                    - path: /
                      pathType: ImplementationSpecific
              tls: []

            resources: {} # Настройки ресурсов (CPU/RAM)

            # ... (добавьте переменные для ConfigMap, Secret, PVC) ...
            config: # Переменные для ConfigMap
              appGreeting: "Hello from ConfigMap (default)!"
              # applicationProperties: | # Если монтируете файл
              #  server.port=8080
              #  app.greeting=${appGreeting}

            secrets: # Переменные для Secret. В values.yaml не храните реальные секреты!
                     # Это только пример, как они будут использоваться в шаблонах.
                     # Реальные значения будут передаваться при установке/обновлении Helm.
                     # Или используйте внешние способы управления секретами (Vault, ExternalSecrets)
              dbPassword: "default_db_password" # ТОЛЬКО ДЛЯ СИНТАКСИСА, НЕ РЕАЛЬНЫЙ ПАРОЛЬ
              vaultToken: "default_vault_token" # ТОЛЬКО ДЛЯ СИНТАКСИСА

            database: # Переменные для подключения к БД (если не из Vault)
              url: "jdbc:postgresql://IP_DB_VM:5432/myappdb"
              user: "myappuser"
              # password: см. secrets.dbPassword

            vault: # Переменные для Vault (если приложение использует его)
              address: "http://IP_VAULT_VM:8200"
              # token: см. secrets.vaultToken
              secretPath: "secret/myapp/config/staging"

            persistentVolumeClaim: # Переменные для PVC
              enabled: true
              size: 1Gi
              storageClassName: "standard"

            # Другие параметры (liveness/readiness probes, tolerations, nodeSelector и т.д.)
            livenessProbe:
              enabled: false # Включить/отключить пробу
              initialDelaySeconds: 60
              periodSeconds: 10
              timeoutSeconds: 5
              path: "/actuator/health/liveness"
              port: 8080 # Или http

            readinessProbe:
              enabled: false
              initialDelaySeconds: 60
              periodSeconds: 10
              timeoutSeconds: 5
              path: "/actuator/health/readiness"
              port: 8080 # Или http
            ```

        *   **В манифестах (`.yaml` в `templates/`):** Используйте синтаксис шаблонов для ссылки на переменные из `values.yaml`.

            ```yaml
            # my-app-chart/templates/myapp_deployment.yaml (шаблон)

            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: {{ include "my-app.fullname" . }} # Имя релиза Helm + имя chart'а (или nameOverride/fullnameOverride)
              labels:
                {{ include "my-app.labels" . | nindent 4 }} # Использование хелпера для меток
            spec:
              {{ if not .Values.autoscaling.enabled }}
              replicas: {{ .Values.replicaCount }} # Количество реплик из values.yaml
              {{ end }}
              selector:
                matchLabels:
                  {{ include "my-app.selectorLabels" . | nindent 6 }} # Использование хелпера для селектора
              strategy:
                type: RollingUpdate
                # ...
              template:
                metadata:
                  labels:
                    {{ include "my-app.selectorLabels" . | nindent 8 }}
                spec:
                  {{ if .Values.imagePullSecrets }} # Если определены секреты для скачивания образа
                  imagePullSecrets: {{ toYaml .Values.imagePullSecrets | nindent 8 }}
                  {{ end }}
                  containers:
                  - name: {{ .Chart.Name }} # Имя контейнера
                    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}" # Сборка имени образа из values.yaml
                    imagePullPolicy: {{ .Values.image.pullPolicy }}
                    ports:
                    - containerPort: {{ .Values.service.targetPort }}
                      name: http
                    {{ if .Values.livenessProbe.enabled }} # Условное включение пробы
                    livenessProbe:
                      httpGet:
                        path: {{ .Values.livenessProbe.path }}
                        port: {{ .Values.livenessProbe.port }}
                      initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
                      periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
                      timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
                    {{ end }}
                    {{ if .Values.readinessProbe.enabled }} # Условное включение пробы
                    readinessProbe:
                      httpGet:
                        path: {{ .Values.readinessProbe.path }}
                        port: {{ .Values.readinessProbe.port }}
                      initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
                      periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
                      timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds }}
                    {{ end }}
                    {{ if .Values.resources }} # Условное включение ресурсов
                    resources:
                      {{ toYaml .Values.resources | nindent 10 }}
                    {{ end }}
                    env: # Переменные окружения
                    - name: SPRING_DATASOURCE_URL
                      value: {{ .Values.database.url | quote }} # Значение из values.yaml
                    - name: SPRING_DATASOURCE_USERNAME
                      value: {{ .Values.database.user | quote }}
                    - name: SPRING_DATASOURCE_PASSWORD
                      valueFrom:
                        secretKeyRef:
                          name: {{ include "my-app.fullname" . }}-secrets # Ссылка на Secret, который должен быть создан
                          key: db_password

                    - name: VAULT_ADDR
                      value: {{ .Values.vault.address | quote }}
                    - name: VAULT_TOKEN_VAR # Имя переменной для токена
                      valueFrom:
                        secretKeyRef:
                          name: {{ include "my-app.fullname" . }}-secrets # Ссылка на Secret
                          key: vault_token # Ключ в Secret

                    - name: SECRET_PATH
                      value: {{ .Values.vault.secretPath | quote }}


                  {{ if .Values.persistentVolumeClaim.enabled }} # Условное включение монтирования PVC
                  volumeMounts:
                  - name: logs-storage
                    mountPath: /app/logs
                  {{ end }}
                  # ...

              volumes:
              {{ if .Values.persistentVolumeClaim.enabled }} # Условное включение Volumes
              - name: logs-storage
                persistentVolumeClaim:
                  claimName: {{ include "my-app.fullname" . }}-pvc # Ссылка на PVC, который должен быть создан
              {{ end }}

            ```
        *   **Важно:** Синтаксис шаблонов Helm специфичен. `{{ .Values.someKey }}` ссылается на значение в `values.yaml`. `{{ include "my-app.fullname" . }}` использует хелпер для генерации имени ресурса. `| nindent N` добавляет отступ. `| quote` добавляет кавычки.
        *   Создайте также шаблоны для Service, ConfigMap, PVC (если они включены в `values.yaml`), используя параметризацию.

    5.  **Проверьте синтаксис Chart'а:**
        ```bash
        cd my-app-chart
        helm lint . # Проверить Chart на ошибки
        helm template . --debug # Сгенерировать финальные манифесты с значениями по умолчанию и вывести их
        # Проверить с переопределением значений:
        # helm template . --debug --set image.tag=1.0.1 --set replicaCount=3
        ```

    6.  **Упакуйте Chart (опционально):**
        ```bash
        helm package .
        # Создаст файл my-app-chart-X.Y.Z.tgz
        ```

    7.  **Установите Chart в Kubernetes кластер:**
        ```bash
        helm install my-app-release . # Установить Chart из текущего каталога, создать релиз с именем my-app-release
        # Или установить упакованный Chart:
        # helm install my-app-release my-app-chart-X.Y.Z.tgz
        # Передать значения, отличающиеся от values.yaml:
        # helm install my-app-release . --set image.tag=latest --set replicaCount=1
        # Передать секреты (лучше через файл значений или другие методы, не в командной строке):
        # helm install my-app-release . --set secrets.dbPassword="мой_пароль" --set secrets.vaultToken="мой_токен" # НЕБЕЗОПАСНО! Используйте --values файл.
        ```
        Helm создаст все ресурсы, определенные в Chart'е (Deployment, Service, ConfigMap, PVC).

    8.  **Проверьте установленный релиз и ресурсы:**
        ```bash
        helm list # Список установленных релизов
        kubectl get all -l app=my-java-app # Проверить ресурсы по метке (если метки заданы в Chart'е)
        kubectl get pods -l app=my-java-app # Убедиться, что Pods запущены
        ```

    9.  **Обновите релиз:**
        *   Измените значения в `values.yaml` (например, `replicaCount: 3`) или создайте файл с новыми значениями (`new-values.yaml`).
        *   Обновите релиз:
            ```bash
            helm upgrade my-app-release . # Обновить с values.yaml из Chart'а
            # Или с файлом новых значений:
            # helm upgrade my-app-release . -f new-values.yaml
            # Передать только отдельные значения:
            # helm upgrade my-app-release . --set image.tag=новый_тег
            ```
        *   Helm определит изменения и применит их к ресурсам кластера (например, обновит Deployment, который запустит Rolling Update).

    10. **Удалите релиз:**
        ```bash
        helm uninstall my-app-release
        ```
        Helm удалит все ресурсы, созданные этим релизом.

**Результат Lab 7:** Умение создавать, параметризовать и использовать Helm Charts для управления развертыванием вашего приложения в Kubernetes. Это делает процесс деплоя и обновления гораздо более удобным и версионируемым.

---

**Lab 8: Настройка GitOps с Argo CD**

*   **Цель:** Установить Argo CD и настроить его для автоматической синхронизации состояния кластера с K8s манифестами/Helm Chart'ами, хранящимися в Git.
*   **Предварительные требования:**
    *   Работающий Kubernetes кластер (Minikube/Kind, Lab 1).
    *   Установленный Argo CD CLI (Техническая подготовка).
    *   Отдельный Git репозиторий для K8s манифестов/Helm Chart'ов. Создайте его в Gitea или GitLab (например, `k8s-manifests`).
    *   Ваш Helm Chart (Lab 7) запушен в этот новый репозиторий.

*   **Шаги:**

    1.  **Установите Argo CD в ваш Kubernetes кластер:** Argo CD устанавливается в своем неймспейсе.
        ```bash
        # Создайте неймспейс для Argo CD
        kubectl create namespace argocd

        # Установите Argo CD (используя манифест из официального репозитория)
        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
        ```
        Подождите несколько минут, пока все Pods в неймспейсе `argocd` запустятся (`kubectl get pods -n argocd`).

    2.  **Получите доступ к веб-интерфейсу Argo CD:**
        *   По умолчанию Argo CD API Server Service имеет тип ClusterIP (доступен только внутри кластера). Для доступа извне можно пробросить порт.
        *   **Port Forwarding (для локальной работы):**
            ```bash
            kubectl port-forward service/argocd-server -n argocd 8080:443 # Пробросить локальный порт 8080 на порт 443 в контейнере argocd-server
            ```
            Теперь UI доступен по адресу `http://localhost:8080` (или `https://localhost:8080`, если используете HTTPS).
        *   *Альтернативно:* Изменить тип Service `argocd-server` на NodePort или LoadBalancer (для Production).

    3.  **Получите начальный пароль администратора Argo CD:**
        ```bash
        kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
        ```
        Это начальный пароль для пользователя `admin`. **Смените его после первого входа!**

    4.  **Войдите в Argo CD CLI:**
        ```bash
        argocd login localhost:8080 # Используйте адрес и порт, на который вы пробросили
        # Логин: admin
        # Пароль: ваш_начальный_пароль
        ```

    5.  **Добавьте ваш Git репозиторий с манифестами/Helm Chart'ами в Argo CD:**
        *   **Используя CLI:**
            ```bash
            argocd repo add http://ВАШ_IP_VM_GITEA:3000/MyDevOpsOrg/k8s-manifests.git --username your_gitea_user --password your_gitea_password # Для приватного репозитория Gitea
            # Или для GitLab: argocd repo add https://gitlab.com/your_user/k8s-manifests.git --username your_gitlab_user --password your_gitlab_password
            # Для публичного репозитория логин/пароль не нужны
            ```
        *   **Используя UI:** В левом меню "Settings" -> "Repositories". Нажмите "Connect Repo". Заполните данные репозитория.

    6.  **Создайте новое приложение в Argo CD, которое будет отслеживать ваш Helm Chart:**
        *   **Используя CLI:**
            ```bash
            argocd app create my-app \
              --repo http://ВАШ_IP_VM_GITEA:3000/MyDevOpsOrg/k8s-manifests.git \
              --path my-app-chart # Путь к Helm Chart в репозитории
              --dest-server https://kubernetes.default.svc # Сервер назначения (ваш K8s кластер)
              --dest-namespace default # Неймспейс, куда деплоить
              --helm-values-file my-app-chart/values.yaml # Путь к файлу значений по умолчанию
              # Опционально: --values <(cat values-override.yaml) # Передача переопределяющих значений
            ```
        *   **Используя UI:** На главной странице Argo CD нажмите "+ New App". Заполните:
            *   Application Name: `my-app`
            *   Project: `default`
            *   Sync Policy: `Automatic` (с опциями Prune Resources и Self Heal) - для автоматической синхронизации. Или `Manual`.
            *   Sync Options: (Опционально).
            *   Source -> Repository URL: URL вашего репозитория.
            *   Source -> Revision: `HEAD` или имя ветки (`main`, `develop`).
            *   Source -> Path: Путь к вашему Chart'у (`my-app-chart`).
            *   Destination -> Cluster URL: `https://kubernetes.default.svc` (или выберите из списка).
            *   Destination -> Namespace: `default`.
            *   Helm -> Values Files: Укажите путь к `values.yaml` в репозитории (`my-app-chart/values.yaml`).
            *   Нажмите "Create".

    7.  **Проверьте статус приложения в Argo CD:**
        *   **CLI:** `argocd app list`, `argocd app get my-app`, `argocd app history my-app`.
        *   **UI:** На главной странице вы увидите ваше приложение. Argo CD начнет процесс синхронизации. Он сравнит состояние кластера с желаемым состоянием в Git и применит изменения (деплоит Helm Chart). Статус должен измениться на `Synced` и `Healthy`.

    8.  **Проверьте ресурсы в Kubernetes:**
        ```bash
        kubectl get all -l app=my-java-app # Убедитесь, что ресурсы созданы Argo CD
        ```

    9.  **Попробуйте изменить количество реплик через Git:**
        *   Отредактируйте файл `my-app-chart/values.yaml` в вашем Git репозитории K8s манифестов (`k8s-manifests`). Измените `replicaCount` (например, на 3).
        *   Сделайте коммит и пуш в Git.
        *   Наблюдайте за Argo CD UI. Он должен обнаружить изменение в Git (состояние `OutOfSync`). Если настроена автоматическая синхронизация, он автоматически применит изменение к кластеру (статус `Syncing`), и Kubernetes масштабирует Deployment.

**Результат Lab 8:** Установлен и настроен Argo CD, который автоматически отслеживает изменения в вашем Git репозитории с Helm Chart'ом и поддерживает состояние вашего приложения в Kubernetes кластере в актуальном (синхронизированном) состоянии. Это базовый GitOps конвейер.

---

**Lab 9: Интеграция GitOps в CI/CD Pipeline (Обновление образа)**

*   **Цель:** Изменить Jenkins/GitLab CI пайплайн так, чтобы после сборки и публикации нового Docker образа он обновлял тег образа в файле `values.yaml` в Git репозитории с K8s манифестами. Argo CD автоматически подхватит это изменение и выполнит деплой.
*   **Предварительные требования:**
    *   Рабочий CI/CD пайплайн (Jenkinsfile или `.gitlab-ci.yml`) из Урока 7, который собирает и публикует Docker образ (Lab 8).
    *   Рабочий GitOps конвейер с Argo CD, отслеживающий ваш Helm Chart из отдельного Git репозитория (Lab 8).
    *   Ваш Helm Chart использует переменную в `values.yaml` для тега образа (например, `.Values.image.tag`).
    *   Jenkins/Runner VM должен иметь доступ к Git репозиторию с K8s манифестами по SSH или HTTPS с учетными данными.

*   **Шаги (для Jenkinsfile):**

    1.  **Обновите ваш Jenkinsfile:** Добавьте новый этап *после* этапа `Build & Publish Docker Image`.

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any

            environment {
                // ... (существующие переменные) ...

                // Информация о репозитории K8s манифестов для GitOps
                K8S_MANIFESTS_REPO = 'http://ВАШ_IP_VM_GITEA:3000/MyDevOpsOrg/k8s-manifests.git'
                K8S_MANIFESTS_BRANCH = 'main' # Ветка, которую отслеживает Argo CD
                # Учетные данные для пуша в репозиторий манифестов (ИСПОЛЬЗУЙТЕ JENKINS CREDENTIALS!)
                // K8S_REPO_CREDS = credentials('gitea-user-pass') # Пример: те же, что для репо кода приложения
            }

            stages {
                // ... (CI этапы: Checkout, Build & Publish Docker Image) ...

                // НОВЫЙ ЭТАП: Update K8s Manifests in Git (GitOps)
                stage('Update K8s Manifests') {
                    steps {
                        echo 'Обновление тега образа в репозитории K8s манифестов...'
                        script {
                            // Получить тег образа из текущей сборки
                            def newImageTag = env.APP_VERSION ?: 'latest' // Используйте тот же тег, что и для Docker образа

                            // Каталог для клонирования репозитория манифестов
                            def manifestsRepoDir = "k8s-manifests-repo"

                            // Клонировать репозиторий K8s манифестов
                            sh "git clone ${K8S_MANIFESTS_REPO} ${manifestsRepoDir}"
                            dir(manifestsRepoDir) { // Переходим в каталог репозитория манифестов
                                sh "git checkout ${K8S_MANIFESTS_BRANCH}"

                                // Путь к values.yaml в репозитории
                                def valuesFile = "my-app-chart/values.yaml"

                                // Обновить тег образа в values.yaml
                                // Используем sed или схожий инструмент для редактирования файла.
                                // ЭТО УПРОЩЕННЫЙ МЕТОД! В production лучше использовать инструменты
                                # для редактирования YAML/JSON с сохранением структуры и комментариев (например, yq, jsonnet, kustomize)
                                # Или использовать Helm Post Renderer.
                                sh "sed -i 's|tag: \"\(.*\)\"|tag: \"${newImageTag}\"|g' ${valuesFile}"
                                # Проверка: убедиться, что изменение применено
                                # sh "cat ${valuesFile}"

                                # Добавить измененный файл, сделать коммит и пуш
                                sh "git add ${valuesFile}"
                                sh "git commit -m 'ci: Update my-app image tag to ${newImageTag} [skip ci]'" # [skip ci] чтобы избежать бесконечного цикла, если этот репо тоже имеет CI

                                # Настроить Git для пуша (если еще не настроено)
                                sh 'git config user.email "jenkins@yourcompany.com"'
                                sh 'git config user.name "Jenkins CI"'

                                # Пушнуть изменения
                                // Требуется SSH-доступ по ключу Jenkins к Git серверу ИЛИ HTTPS с учетными данными.
                                // Если используете SSH, убедитесь, что публичный ключ Jenkins добавлен в Gitea/GitLab.
                                // Если HTTPS, используйте withCredentials для инжекции логина/пароля в env или .git-credentials
                                sh "git push origin ${K8S_MANIFESTS_BRANCH}"
                            }
                        }
                        echo 'Репозиторий K8s манифестов обновлен.'
                    }
                }

                // Удаляем этапы ручного деплоя на Staging, Wait for Health Check, Manual Approval из Jenkinsfile!
                // Это теперь делает Argo CD. Оставляем только сборку и обновление манифестов.
                // Если вам нужен Manual Approval, настройте его в Argo CD (Manual Sync Policy) или используйте Promotion в Argo CD.

                // Оставляем только CI-связанные этапы
                // stage('Provision Infrastructure') { ... } // Этот этап теперь можно вынести в отдельный IaC пайплайн, запускаемый реже
                // stage('Wait for SSH') { ... }
                // stage('Configure VM') { ... }
                // stage('Database Migrations') { ... } // Этот этап тоже может быть частью GitOps (например, с помощью Helm) или отдельным пайплайном DBOps

                // СУЩЕСТВУЮЩИЙ ЭТАП: Deploy to Staging - УДАЛЕН или перенесен в Argo CD

                // СУЩЕСТВУЮЩИЙ ЭТАП: Manual Approval for Production - УДАЛЕН
            }

            post {
                 always { echo 'Пайплайн завершен.' }
                 success { echo 'Пайплайн выполнен успешно! 🎉' ; junit '**/target/surefire-reports/*.xml' }
                 failure { echo 'Пайплайн завершился с ошибкой! 💔' }
            }
        }
        ```

    2.  **Настройте Git на Jenkins/Runner VM для пуша:**
        *   Если пушите по SSH: Сгенерируйте SSH ключ на Jenkins/Runner VM (как в Уроке 3, Lab 4). Скопируйте публичный ключ в настройки вашего пользователя/Deploy Keys в Gitea/GitLab для репозитория `k8s-manifests`. Убедитесь, что Jenkins user может использовать этот ключ.
        *   Если пушите по HTTPS: Создайте Credential в Jenkins с логином и паролем/токеном для Git, и используйте `withCredentials` для инжекции в переменные окружения, которые будут использоваться `git push`.

    3.  **Сделайте коммит и пуш Jenkinsfile:**
        ```bash
        git add Jenkinsfile
        git commit -m "ci: Transition to GitOps - update k8s manifests after build"
        git push origin develop
        ```

    4.  **Запустите обновленный CI пайплайн:**
        *   Сделайте новый коммит в коде приложения (в ветке `develop`).
        *   Пайплайн запустится. Он соберет приложение, создаст и опубликует Docker образ с новым тегом (например, номером сборки).
        *   Затем он клонирует репозиторий K8s манифестов, обновит файл `values.yaml`, сделает коммит и пуш этого изменения.

    5.  **Наблюдайте за Argo CD:**
        *   Argo CD обнаружит новый коммит в репозитории K8s манифестов.
        *   Состояние приложения в Argo CD изменится на `OutOfSync`.
        *   Если синхронизация автоматическая, Argo CD автоматически применит изменение к кластеру (обновит Helm Release), который запустит Rolling Update Deployment с новым тегом образа.
        *   Если синхронизация ручная, вам нужно будет зайти в UI Argo CD и нажать кнопку "Sync".

**Результат Lab 9:** Вы успешно интегрировали ваш CI пайплайн с GitOps конвейером. CI отвечает за сборку и тестирование артефакта, а GitOps (Argo CD) отвечает за его автоматическую доставку в Kubernetes кластер на основе изменений в Git репозитории, который является единым источником истины для состояния вашей инфраструктуры и приложений.

---

**Резюме Урока 9:**

Вы сделали огромный шаг вперед, освоив основы Kubernetes - лидирующей платформы для оркестрации контейнеров. Вы научились работать с `kubectl`, деплоить приложения с помощью Deployments и Services, управлять конфигурацией и хранилищем. Упаковав приложение в Helm Chart, вы упростили процесс его развертывания. Наконец, внедрив GitOps с Argo CD, вы перевели процесс доставки на новый, более безопасный и надежный уровень, где Git становится центром управления вашими развертываниями.

Теперь вы обладаете навыками, которые являются ключевыми для Middle+/Senior DevOps инженера, работающего с контейнеризованными приложениями.

**Что дальше?**

В последних уроках мы сосредоточимся на мониторинге, логировании и алертинге, чтобы научиться видеть, что происходит с нашим приложением и инфраструктурой в Kubernetes, и получать уведомления о проблемах. Это завершающие компоненты зрелого DevOps конвейера.

Готовьтесь сделать вашу систему наблюдаемой!