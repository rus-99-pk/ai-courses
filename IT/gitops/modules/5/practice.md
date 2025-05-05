Отлично! Мы подошли к финалу курса - Проектной работе. Это задача, которая позволит собрать воедино все полученные знания и навыки.

Вот подробная методичка, как подойти к выполнению этой проектной задачи. Она будет включать шаги по выбору и настройке компонентов, организации репозитория и демонстрации работы GitOps flow.

---

### Методичка: Проектная работа

**Название:** Развертывание и управление веб-приложением с БД с помощью GitOps

**Цель:** Реализовать комплексный сценарий GitOps, управляя как приложением (Frontend, Backend), так и его инфраструктурой (База данных) из Git, с использованием Kubernetes, Crossplane и Argo CD/Flux.

**Необходимые условия/Инструменты:**

1.  Работающий кластер Kubernetes с установленным `kubectl` (рекомендуется облачный кластер для реальной БД и Ingress/LoadBalancer, но можно использовать локальный с `provider-kubernetes` для БД и Ingress-контроллером).
2.  Установленный Git.
3.  Аккаунт на Git-хостинге.
4.  Аккаунт в облачном провайдере (если управляете облачной БД через Crossplane).
5.  Установленный Crossplane и соответствующий провайдер для вашего облака (или `provider-kubernetes`) в кластере (после ДЗ Модуля 4, Тема 2).
6.  Установленный **Argo CD** или **Flux CD** в кластере (после ДЗ Модуля 3, Тема 2 или Тема 4). **Выберите ОДИН инструмент для управления приложениями.**
7.  Базовое веб-приложение "Гостевая книга":
    *   Backend-образ, который может подключаться к PostgreSQL и имеет HTTP API (например, на Go, Python, Node.js). **Вам понадобится найти или написать простое приложение.**
    *   Frontend-образ (например, на React, Vue, простом HTML/JS), который может взаимодействовать с Backend API. **Также понадобится найти или написать.**
    *   У вас должны быть Docker-образы для Frontend и Backend, доступные в публичном или приватном репозитории Docker.
8.  PostgreSQL доступен для управления через выбранный провайдер Crossplane (например, RDSInstance для AWS, Database.azure.upbound.io для Azure, SQLInstance для GCP). **ИЛИ** вы можете использовать `provider-kubernetes` для развертывания PostgreSQL внутри кластера, но это менее типичный GitOps-сценарий для БД продакшен уровня. **Мы сосредоточимся на управлении внешней БД через облачный провайдер с помощью Crossplane.**

**Подробные шаги:**

**Шаг 1: Спроектируйте структуру Git-репозитория(ев)**

Решите, будете ли вы использовать один репозиторий для всего (приложения + инфраструктура + GitOps config) или разделите их (например, репо для кода приложений, репо для K8s манифестов и репо для Crossplane/Terraform). Для этого проекта один репозиторий часто проще.

Предлагаемая структура одного репозитория `guestbook-gitops`:

```
guestbook-gitops/
├── README.md
├── kubernetes/
│   ├── apps/
│   │   ├── backend/
│   │   │   ├── deployment.yaml
│   │   │   └── service.yaml
│   │   └── frontend/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── ingress.yaml # Или NodePort service, LoadBalancer service
│   └── environments/ # Опционально: для конфигов, зависящих от среды
│       └── dev/
│           └── kustomization.yaml # или helmrelease.yaml, если используете kustomize/helm
└── infra/
    └── crossplane/
        ├── providerconfig.yaml # Если не используете ProviderConfig по умолчанию
        └── postgresql.yaml     # Манифест Crossplane для БД
└── gitops/
    ├── argocd/             # Если используете Argo CD
    │   └── applications/   # App of Apps структура
    │       ├── root.yaml   # Корневой Application
    │       └── guestbook/  # Дочерний Application или ApplicationSet
    │           ├── backend-app.yaml # Argo CD App для backend
    │           ├── frontend-app.yaml # Argo CD App для frontend
    │           └── infra-app.yaml    # Argo CD App для Crossplane infra
    └── flux/               # Если используете Flux CD
        └── clusters/
            └── my-cluster/
                ├── flux-system.yaml # Базовая Kustomization Flux
                ├── backend-kustomization.yaml # Flux Kustomization для backend
                ├── frontend-kustomization.yaml # Flux Kustomization для frontend
                └── crossplane-kustomization.yaml # Flux Kustomization для Crossplane infra
```

**Шаг 2: Создайте Git-репозиторий и структуру папок**

Создайте новый репозиторий `guestbook-gitops` на вашем Git-хостинге и склонируйте его. Создайте выбранную структуру папок.

**Шаг 3: Подготовьте или найдите манифесты приложений (Frontend, Backend)**

Создайте базовые манифесты Deployment и Service для вашего Frontend и Backend. Поместите их в соответствующие папки в `kubernetes/apps/`.

*   **Deployment:** Укажите ваши Docker-образы. Настройте селекторы и лейблы.
*   **Service:** Используйте тип `ClusterIP` для Backend (для связи с Frontend) и для Frontend (для доступа через Ingress).
*   **Ingress:** Создайте Ingress-ресурс для Frontend, чтобы сделать его доступным извне кластера (как в ДЗ Модуля 1, Тема 3). Убедитесь, что у вас установлен Ingress-контроллер.

**Шаг 4: Напишите манифест Crossplane для Базы данных (PostgreSQL)**

Определите тип ресурса PostgreSQL для вашего облачного провайдера (например, `RDSInstance` для AWS). Создайте YAML-файл в `infra/crossplane/postgresql.yaml`.

*   **Важно:** Манифест должен включать спецификацию инстанса (размер, версию БД, storage) и ссылку на ваш ProviderConfig (см. ДЗ Модуля 4, Тема 2).
*   Пример (обобщенный):
    ```yaml
    apiVersion: <api_версия_вашего_провайдера>/v1beta1
    kind: <Тип_ресурса_PostgreSQL> # Например, RDSInstance, SQLInstance, Database
    metadata:
      name: guestbook-db
    spec:
      forProvider:
        # Специфичные для провайдера поля (размер инстанса, версия БД, настройки сети и т.д.)
        # Например: engine: postgres, engineVersion: "14", instanceClass: db.t3.micro, allocatedStorage: 20
        # Укажите необходимые параметры региона, группы безопасности и т.д.
      writeConnectionSecretToRef:
        namespace: default # Или неймспейс, где будут поды приложения
        name: guestbook-db-conn # Имя Secret, куда Crossplane запишет учетные данные для подключения
      providerConfigRef:
        name: default # Ссылка на ваш ProviderConfig
    ```
*   Crossplane автоматически создаст Secret (`guestbook-db-conn` в неймспейсе `default` в примере) с учетными данными для подключения к созданной БД.

**Шаг 5: Настройте Deployment Backend для использования Secret с учетными данными БД**

Измените манифест Deployment Backend (`kubernetes/apps/backend/deployment.yaml`). Смонтируйте Secret `guestbook-db-conn` как переменные окружения или файл, чтобы Backend мог получить данные для подключения к БД.

```yaml
# kubernetes/apps/backend/deployment.yaml (фрагмент)
...
spec:
  template:
    ...
    spec:
      containers:
      - name: backend
        image: <ваш_образ_backend_api>
        ports:
        - containerPort: <порт_backend>
        envFrom: # Читать переменные окружения из Secret
        - secretRef:
            name: guestbook-db-conn # Имя Secret, созданного Crossplane
        # Или через volumeMounts, если приложение читает конфиг из файла
        # volumeMounts:
        # - name: db-creds-volume
        #   mountPath: /etc/db-creds # Путь в контейнере
        #   readOnly: true
      # volumes:
      # - name: db-creds-volume
      #   secret:
      #     secretName: guestbook-db-conn
...
```
*   **Важно:** Убедитесь, что ваш Backend-образ знает, как читать учетные данные из указанных переменных окружения или файла.

**Шаг 6: Настройте GitOps инструмент (Argo CD или Flux) для синхронизации репозитория**

*   **Если используете Argo CD:** Создайте Application для всего репозитория или используйте паттерн App of Apps. Создайте корневой Application (`gitops/argocd/applications/root.yaml`) и, возможно, дочерние Application для backend (`gitops/argocd/applications/guestbook/backend-app.yaml`), frontend (`gitops/argocd/applications/guestbook/frontend-app.yaml`) и инфраструктуры (`gitops/argocd/applications/guestbook/infra-app.yaml`), указывающие на соответствующие папки с манифестами. Убедитесь, что корневой Application синхронизируется с кластером (как в ДЗ Модуля 3, Тема 2, Шаг 9).
*   **Если используете Flux CD:** Модифицируйте ваш основной Kustomization или создайте новые Kustomization ресурсы (`gitops/flux/clusters/my-cluster/*.yaml`) для синхронизации папок с манифестами приложений (`kubernetes/apps/`) и Crossplane (`infra/crossplane/`). Убедитесь, что основной Flux Kustomization синхронизируется с кластером (как в ДЗ Модуля 3, Тема 4, Шаг 2).

**Шаг 7: Добавьте, закоммитьте и отправьте все файлы в Git**

```bash
git add .
git commit -m "Initial commit for guestbook application and infrastructure"
git push origin main # Или master
```

**Шаг 8: Наблюдайте за процессом развертывания**

*   **Для Argo CD:** Смотрите веб-интерфейс. Должны появиться новые Application(ы). Сначала Crossplane Application начнет синхронизироваться, создавая CRD для БД. Затем контроллер Crossplane создаст реальную БД в облаке и Secret с учетными данными. После этого Application(ы) для Backend и Frontend увидят свои манифесты, Backend увидит созданный Secret и сможет запуститься, подключившись к БД.
*   **Для Flux CD:** Смотрите логи контроллеров Flux. Убедитесь, что Kustomization для Crossplane успешно синхронизирован (создал CRD БД), затем Kustomization для приложений синхронизируется.

Это самый долгий этап, так как создание инстанса базы данных в облаке через Crossplane может занять 5-15 минут или более, в зависимости от провайдера.

**Шаг 9: Проверьте состояние развернутых ресурсов**

Используйте `kubectl` для проверки:
*   Состояние Crossplane ресурса БД: `kubectl get <тип_вашего_crossplane_бд_ресурса> guestbook-db -n default` (или неймспейс, куда деплоите). Убедитесь, что статус `Ready`.
*   Создан ли Secret с учетными данными: `kubectl get secret guestbook-db-conn -n default`.
*   Состояние подов Frontend и Backend: `kubectl get pods -l app=frontend`, `kubectl get pods -l app=backend`. Убедитесь, что они `Running`. Если Backend падает, проверьте его логи (`kubectl logs <backend-pod-name>`) - возможно, он не смог подключиться к БД (Secret еще не создан или некорректен, или настройки подключения в коде неверны).
*   Состояние Services: `kubectl get svc frontend-service backend-service`.
*   Состояние Ingress: `kubectl get ingress frontend-ingress` (или имя вашего Ingress). Найдите внешний IP/Hostname.

**Шаг 10: Проверьте функциональность приложения**

Откройте Frontend в браузере по адресу Ingress (или Service NodePort/LoadBalancer). Попробуйте добавить запись в гостевую книгу. Убедитесь, что записи отображаются. Это подтвердит, что Frontend общается с Backend, а Backend успешно подключается к БД и сохраняет/читает данные.

**Шаб 11: Продемонстрируйте процесс обновления приложения через GitOps**

Как вы делали в ДЗ Модуля 4, Тема 6:
1.  Найдите второй Docker-образ для Frontend или Backend с видимым отличием.
2.  Измените тег образа в манифесте Deployment этого приложения в Git-репозитории.
3.  Закоммитьте и отправьте изменение в Git.
4.  Наблюдайте, как ваш GitOps инструмент обнаруживает изменение и автоматически обновляет Deployment в кластере.
5.  После завершения обновления, получите доступ к приложению и продемонстрируйте, что запущена новая версия.

**Шаг 12: Продемонстрируйте процесс изменения инфраструктуры через GitOps (Опционально, но приветствуется)**

*   Измените какой-либо параметр в YAML-манифесте Crossplane для БД в Git (например, увеличьте размер хранилища, измените версию БД, добавьте тег).
*   Закоммитьте и отправьте изменение.
*   Наблюдайте, как GitOps инструмент применяет изменение CRD, а контроллер Crossplane инициирует обновление реальной БД в облаке.
*   Проверьте в консоли облака, что изменение применено (это также может занять время и может привести к временной недоступности БД в зависимости от типа изменения и облачного провайдера).

**Измеряемость (Сбор результатов для демонстрации):**

Соберите все артефакты и выводы, запрошенные в описании Проектной работы:

1.  **Ссылка на Git-репозиторий:** Предоставьте URL вашего репозитория `guestbook-gitops`. Убедитесь, что он содержит все необходимые файлы:
    *   Манифесты K8s для Frontend и Backend (`kubernetes/apps/...`).
    *   Манифест Crossplane для БД (`infra/crossplane/postgresql.yaml`).
    *   Конфигурация Argo CD или Flux (`gitops/...`).
    *   (Опционально) README файл с кратким описанием проекта и инструкциями по развертыванию.
2.  **Состояние кластера и инструментов:**
    *   Укажите, какой тип кластера вы использовали (Kind, Minikube, облачный).
    *   Предоставьте вывод команд, подтверждающих установку и работу GitOps инструмента и Crossplane:
        *   `kubectl get pods -n <неймспейс_argocd_или_flux-system>`
        *   `kubectl get pods -n crossplane-system`
        *   `kubectl get providers`
        *   `kubectl get providerconfigs` (если ProviderConfig не default)
3.  **Развернутая инфраструктура (БД):**
    *   Предоставьте вывод команды `kubectl get <тип_вашего_crossplane_бд_ресурса> guestbook-db -n <неймспейс>`, показывающий, что ресурс создан и в статусе `Ready`.
    *   Предоставьте скриншот из консоли облачного провайдера или вывод команды облачного CLI, подтверждающий создание реального инстанса БД.
4.  **Развернутое приложение (Frontend/Backend):**
    *   Предоставьте вывод команд:
        *   `kubectl get deploy -n <неймспейс_приложения>`
        *   `kubectl get svc -n <неймспейс_приложения>`
        *   `kubectl get ingress -n <неймспейс_приложения>` (если используете Ingress)
        *   `kubectl get pods -l app=frontend -n <неймспейс_приложения>`
        *   `kubectl get pods -l app=backend -n <неймспейс_приложения>`
5.  **Функциональность приложения:**
    *   Предоставьте URL для доступа к Frontend.
    *   Предоставьте скриншот веб-страницы гостевой книги, показывающий добавленные записи.
6.  **Процесс обновления:**
    *   Предоставьте ссылку на коммит в Git, который изменил тег образа Frontend или Backend.
    *   Предоставьте вывод команды `kubectl get deploy <имя_деплоймента> -n <неймспейс> -o yaml | grep image` после обновления, показывающий новый тег образа.
    *   Предоставьте скриншот веб-страницы или вывод команды (например, curl), демонстрирующий, что запущена новая версия приложения.
    *   (Опционально) Предоставьте короткое видео или скриншоты из веб-интерфейса Argo CD / логов Flux, демонстрирующие процесс синхронизации обновления.
7.  **(Опционально) Процесс изменения инфраструктуры:**
    *   Предоставьте ссылку на коммит в Git, который изменил параметр Crossplane ресурса БД.
    *   Предоставьте скриншот из консоли облачного провайдера или вывод команды облачного CLI, подтверждающий, что параметр БД был изменен.

**Советы по выполнению:**

*   Начните с самого простого варианта Backend/Frontend приложения (возможно, используйте готовые примеры, если написание кода занимает много времени). Главное - Docker-образы.
*   Тестируйте каждый шаг. Сначала убедитесь, что Crossplane может создать БД вручную (применив манифест `postgresql.yaml` через `kubectl apply`). Затем добавьте его в GitOps. Аналогично с приложениями.
*   Если у вас нет аккаунта в облаке или вы не хотите тратить деньги, используйте `provider-kubernetes` для Crossplane и разверните PostgreSQL внутри кластера (например, используя Helm-чарт PostgreSQL). Тогда Crossplane будет управлять Deployment, StatefulSet, Service и т.д. для БД. Это также допустимый сценарий GitOps.
*   Уделяйте внимание неймспейсам. Инструменты GitOps и Crossplane обычно устанавливаются в свои неймспейсы (argocd, flux-system, crossplane-system), а приложения и Secret для БД могут быть в неймспейсе `default` или отдельном неймспейсе приложения.
*   Автоматизация подключения Frontend к Backend и Backend к БД: Frontend должен знать имя Service Backend (`backend-service`). Backend должен уметь читать учетные данные из Secret, созданного Crossplane.

Успехов в выполнении проектной работы! Это отличная возможность закрепить знания и увидеть GitOps в действии на связке "приложение + инфраструктура". Не стесняйтесь обращаться за помощью, если столкнетесь с конкретными проблемами на каком-либо шаге.