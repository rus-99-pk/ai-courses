# 04 - ApplicationSet: Управление в масштабе

### Цель урока
Научиться использовать контроллер `ApplicationSet` для автоматического создания и управления множеством ArgoCD `Application` на основе различных источников (Git, списки, генераторы).

### Какую проблему решаем?
Представьте, что у вас есть десятки или сотни микросервисов. Или вы хотите развернуть одно и то же приложение (например, Prometheus) в нескольких кластерах (dev, staging, prod).

Без `ApplicationSet` вам пришлось бы:
1.  Скопировать и вставить YAML-манифест `Application` для каждого микросервиса или кластера.
2.  Изменить в каждом файле имя, неймспейс, возможно, версию или другие параметры.
3.  Поддерживать эту гору однотипных YAML-файлов.

Это скучно, подвержено ошибкам и плохо масштабируется. **`ApplicationSet` решает проблему "копипасты"**, позволяя вам определить шаблон `Application` и автоматически генерировать конкретные экземпляры на основе данных из "генераторов".

**Важно:** `ApplicationSet` контроллер уже установлен вместе с ArgoCD, если вы использовали стандартный манифест `install.yaml` (начиная с ArgoCD v2.3).

### Теория: Как работает ApplicationSet

`ApplicationSet` — это Kubernetes CRD, который состоит из двух основных частей:
1.  **Генераторы (Generators):** Они определяют, *откуда* брать параметры для создания приложений. Например, список кластеров, папки в Git-репозитории и т.д. Генератор создает список параметров (например, `{ cluster: 'prod', url: 'https://prod-k8s.example.com' }`).
2.  **Шаблон (Template):** Это шаблон ArgoCD `Application`, в котором используются "переменные", подставляемые из генератора. Например, `name: 'my-app-{{cluster}}'`.

`ApplicationSet` контроллер "слушает" генераторы. Для каждого набора параметров, который возвращает генератор, он рендерит шаблон и создает (или обновляет) соответствующий `Application`.

![ApplicationSet Workflow](https://argo-cd.readthedocs.io/en/stable/assets/applicationset-overview.png)

### Практика: Использование генераторов

Давайте рассмотрим самые популярные генераторы на практике.

#### 1. Генератор `List`
Самый простой генератор. Вы просто перечисляете наборы параметров прямо в манифесте `ApplicationSet`.

**Задача:** Развернуть два экземпляра приложения `guestbook` в разных неймспейсах: `guestbook-dev` и `guestbook-staging`.

**Манифест `applicationset-list.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-set
  namespace: argocd
spec:
  # Используем List генератор
  generators:
  - list:
    elements:
    - environment: dev
      namespace: guestbook-dev
    - environment: staging
      namespace: guestbook-staging
  
  # Шаблон для создания Application
  template:
    metadata:
      # Имя приложения будет сгенерировано: guestbook-dev, guestbook-staging
      name: 'guestbook-{{environment}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
      destination:
        server: https://kubernetes.default.svc
        # Неймспейс будет взят из параметров генератора
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```
**Применение:**
```bash
kubectl apply -f applicationset-list.yaml
```
**Результат:**
Зайдите в UI ArgoCD или выполните `argocd app list`. Вы увидите два новых приложения: `guestbook-dev` и `guestbook-staging`, каждое из которых развернуто в свой неймспейс.

---

#### 2. Генератор `Git` (Directory)
Это самый мощный и распространенный генератор. Он сканирует структуру папок в Git-репозитории и создает приложения для каждой найденной папки.

**Задача:** У нас есть "репозиторий приложений", где каждая папка — это отдельный микросервис. Мы хотим, чтобы ArgoCD автоматически разворачивал любой новый микросервис, который появляется в этом репозитории.

**Структура репозитория `my-apps-repo`:**
```
my-apps-repo/
├── apps/
│   ├── app1/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── app2/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── app3/
│       ├── statefulset.yaml
│       └── service.yaml
└── ...
```

**Манифест `applicationset-git.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-microservices
  namespace: argocd
spec:
  # Используем Git генератор
  generators:
  - git:
      repoURL: https://github.com/my-org/my-apps-repo.git # Замените на ваш репозиторий
      revision: HEAD
      # Указываем, что нужно сканировать папки по этому пути
      directories:
      - path: apps/*
  
  template:
    metadata:
      # Имя приложения будет равно имени папки: app1, app2, app3
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/my-apps-repo.git # Тот же репозиторий
        targetRevision: HEAD
        # Путь к манифестам будет равен пути к найденной папке
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        # Разворачиваем в неймспейс с таким же именем, как у приложения
        namespace: '{{path.basename}}'
```
**Применение и результат:**
После применения этого манифеста ArgoCD создаст три приложения: `app1` (в неймспейсе `app1`), `app2` (в неймспейсе `app2`) и `app3` (в неймспейсе `app3`). Если вы добавите папку `app4` в Git и запушите изменения, `ApplicationSet` автоматически создаст для нее новое `Application`!

---

#### 3. Генератор `Cluster`
Этот генератор получает список кластеров, которые зарегистрированы в ArgoCD, и позволяет разворачивать приложения в каждый из них. Это идеальное решение для управления "флотом" кластеров.

**Задача:** Развернуть оператор мониторинга `node-exporter` во все зарегистрированные кластеры, кроме локального.

**Предварительное условие:** У вас должно быть зарегистрировано несколько кластеров в ArgoCD (через `argocd cluster add`).

**Манифест `applicationset-cluster.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: node-exporter-all-clusters
  namespace: argocd
spec:
  generators:
  - clusters:
      # Можно фильтровать кластеры по лейблам
      selector:
        matchLabels:
          # Например, разворачивать только в 'production' кластеры
          # argocd.argoproj.io/cluster.label: 'production'

  template:
    metadata:
      # Имя: node-exporter-prod-cluster, node-exporter-staging-cluster
      name: 'node-exporter-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/prometheus-community/helm-charts.git
        targetRevision: 4.23.0 # Пример для Helm-чарта
        chart: prometheus-node-exporter
        helm:
          values: |
            # Здесь можно передать специфичные для кластера значения
            extraArgs:
              - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
      destination:
        # Имя и сервер берутся из параметров кластера
        server: '{{server}}'
        namespace: 'monitoring'
```
**Результат:** ArgoCD создаст по одному `Application` для каждого зарегистрированного кластера, обеспечивая консистентность конфигурации мониторинга во всей вашей инфраструктуре.

### Выводы

`ApplicationSet` — это необходимый инструмент для работы с ArgoCD в любой среде, кроме самой маленькой. Он позволяет перейти от ручного управления к автоматизированному и декларативному управлению целыми группами приложений.

**Ключевая идея:** Вместо того чтобы управлять `Application`, вы управляете `ApplicationSet`.

В следующем уроке мы затронем важную тему безопасности: как управлять секретами в GitOps-подходе, не компрометируя их.