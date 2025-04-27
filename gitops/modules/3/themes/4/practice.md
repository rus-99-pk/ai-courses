### Методичка по ДЗ: Модуль 3, Тема 4

**Название ДЗ:** Интеграция инструмента Flux

**Цель ДЗ:** Научиться устанавливать Flux CD и использовать его для синхронизации состояния кластера с Git-репозиторием, демонстрируя автоматическое применение изменений.

**Необходимые условия/Инструменты:**

1.  Работающий кластер Kubernetes с установленным `kubectl` (после ДЗ Модуля 1, Тема 7).
2.  Установленный Git.
3.  Аккаунт на Git-хостинге (GitHub, GitLab, Bitbucket).
4.  YAML-манифесты простого приложения (Deployment, Service). Можно использовать манифесты из ДЗ Модуля 2, Тема 3, или Модуля 3, Тема 2.
5.  Установленный Flux CLI (`flux`). Следуйте инструкции по установке: [https://fluxcd.io/docs/installation/](https://fluxcd.io/docs/installation/)

**Подробные шаги:**

**Шаг 1: Создайте новый Git-репозиторий для Flux**

Создайте новый **приватный** или **публичный** репозиторий на вашем Git-хостинге. Назовите его, например, `my-gitops-flux`.

**Шаг 2: Установите Flux CD в кластер и настройте его на ваш репозиторий**

Flux CLI упрощает процесс установки и первоначальной настройки.

*   **Проверьте готовность кластера и учетные данные Git:**
    ```bash
    flux check --pre
    ```
    Эта команда проверит, что у вас установлен `kubectl`, доступен кластер и что у вас настроен доступ к Git (через переменные окружения или SSH-ключ). Вам может понадобиться настроить SSH-доступ из кластера к вашему Git-репозиторию, если он приватный. Инструкции есть в документации Flux. Для публичных репозиториев это не требуется.
*   **Загрузите SSH-ключ в кластер (если репозиторий приватный):** Сгенерируйте новую пару SSH-ключей (`ssh-keygen -t ed25519 -C "your_email@example.com"`), добавьте публичный ключ в настройки вашего Git-репозитория (Deploy keys или аналогично), а приватный ключ добавьте в кластер с помощью Flux CLI:
    ```bash
    # Замените <path/to/your/private/key> на путь к вашему приватному ключу
    flux create secret git flux-system --url ssh://git@github.com/your_username/my-gitops-flux --private-key-file=<path/to/your/private/key> # Пример для GitHub
    ```
    *   *Пропустите этот шаг, если репозиторий публичный.*

*   **Установите Flux в кластер и свяжите его с репозиторием:**

    ```bash
    flux bootstrap gitlab \ # Или github, bitbucket, generic-git, и т.д. в зависимости от вашего хостинга
        --owner=<ваш_логин_на_git_хостинге> \ # Имя пользователя или организации
        --repository=my-gitops-flux \ # Имя вашего репозитория
        --branch=main \ # Имя основной ветки (или master)
        --path=./clusters/my-cluster \ # Путь в репозитории, куда Flux положит свою конфигурацию (создаст сам)
        --namespace=flux-system # Неймспейс для установки Flux
    ```
    *   **Пояснения:**
        *   Эта команда устанавливает контроллеры Flux в неймспейс `flux-system`.
        *   Она создает необходимую структуру директорий (`clusters/my-cluster`) в вашем удаленном репозитории.
        *   Она создает ресурсы `GitRepository` и `Kustomization`, которые указывают Flux на ваш репозиторий и путь к конфигурации (`clusters/my-cluster`).
        *   Flux автоматически клонирует ваш репозиторий и начинает мониторить указанный путь.

*   **Проверка:** Убедитесь, что поды Flux запущены и ресурсы созданы.
    ```bash
    kubectl get pods -n flux-system
    kubectl get gitrepositories -n flux-system
    kubectl get kustomizations -n flux-system
    ```
    Поды должны быть в статусе `Running`, а ресурсы `GitRepository` и `Kustomization` в статусе `Ready`.

**Шаг 3: Клонируйте репозиторий Flux на свой компьютер**

Flux bootstrap уже создал базовую структуру в вашем удаленном репозитории. Теперь клонируйте его:

```bash
git clone <URL_вашего_flux_репозитория>
cd my-gitops-flux
```

**Шаг 4: Поместите манифесты вашего приложения в репозиторий**

Внутри вашего склонированного репозитория, в папке `clusters/my-cluster` (или той, которую вы указали в `--path`), вы увидите файлы, созданные Flux. Создайте новую папку для ваших приложений.

```bash
mkdir apps
mkdir apps/my-app
```

Скопируйте манифесты простого приложения (Deployment, Service) в эту папку `apps/my-app`.

```bash
# Замените <путь_к_манифестам>
cp <путь_к_манифестам>/app-deployment.yaml apps/my-app/
cp <путь_к_манифестам>/app-service.yaml apps/my-app/ # Используйте Service ClusterIP или NodePort
```

**Шаг 5: Создайте Kustomization-ресурс для приложения**

Flux использует Kustomization ресурсы (не путать с одноименным инструментом kustomize, хотя он его и использует) для определения того, какую директорию в Git синхронизировать.

Создайте файл `clusters/my-cluster/my-app-kustomization.yaml`.

```yaml
# clusters/my-cluster/my-app-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app # Имя Kustomization ресурса
  namespace: flux-system # Где будет создан ресурс Kustomization (в неймспейсе Flux)
spec:
  interval: 1m # Как часто проверять Git на изменения (например, каждые 1 минуту)
  sourceRef:
    kind: GitRepository
    name: flux-system # Ссылка на GitRepository ресурс, созданный при bootstrap
  path: ./apps/my-app # Путь внутри Git-репозитория к манифестам этого приложения
  prune: true # Удалять ресурсы из кластера, которых нет в Git в этой директории
  timeout: 2m # Таймаут для применения изменений
  # targetNamespace: default # Опционально: указать неймспейс, куда деплоить ресурсы, если они не имеют metadata.namespace
```

*   **Пояснения:**
    *   `spec.interval`: Как часто Flux должен опрашивать Git-репозиторий.
    *   `spec.sourceRef`: Ссылка на GitRepository ресурс, который Flux уже создал. Его имя по умолчанию совпадает с неймспейсом установки Flux (`flux-system`).
    *   `spec.path`: Путь в Git-репозитории, где находятся манифесты для этого Kustomization.

**Шаг 6: Добавьте, закоммитьте и отправьте новые файлы в Git**

```bash
git add .
git commit -m "Add my-app manifests and Kustomization"
git push origin main # Или master
```

**Шаг 7: Наблюдайте за синхронизацией Flux**

Flux, согласно `interval: 1m` в Kustomization, в течение минуты-двух увидит новые файлы в репозитории. Он обнаружит новый Kustomization ресурс (`my-app`) и начнет его синхронизировать. Kustomization укажет Flux на папку `apps/my-app`, и Flux применит все манифесты из этой папки к кластеру.

Вы можете следить за логами контроллера Flux Kustomize:

```bash
kubectl logs -n flux-system -l app=kustomize-controller -f
```

Вы также можете вручную триггернуть синхронизацию, если не хотите ждать:

```bash
flux reconcile kustomization my-app --namespace=flux-system
```

**Шаг 8: Проверьте развернутое приложение в кластере**

Убедитесь, что Deployment и Service вашего приложения созданы.

```bash
kubectl get deploy my-nginx-app # Замените на имя вашего Deployment
kubectl get svc my-nginx-service # Замените на имя вашего Service ClusterIP/NodePort
kubectl get pods -l app=nginx # Замените app=nginx на лейбл вашего приложения
```

*   **Ожидаемый результат:** Deployment и Service должны быть созданы, поды должны быть в статусе `Running`.

**Шаг 9: Измените манифест приложения в Git**

Отредактируйте файл манифеста Deployment вашего приложения в локальной копии репозитория (`apps/my-app/app-deployment.yaml`). Измените количество реплик или версию образа.

```yaml
# apps/my-app/app-deployment.yaml (фрагмент)
...
spec:
  replicas: 3 # Измените количество реплик
  selector:
    matchLabels:
      app: my-app-label # Убедитесь, что лейбл корректный
  template:
    metadata:
      labels:
        app: my-app-label # Убедитесь, что лейбл корректный
    spec:
      containers:
      - name: my-app-container
        image: nginx:1.23.0 # Или измените версию образа
...
```

**Шаг 10: Закоммитьте и отправьте изменение в Git**

```bash
git add apps/my-app/app-deployment.yaml
git commit -m "Update replica count/image version for my-app"
git push origin main # Или master
```

**Шаг 11: Наблюдайте за автоматическим применением изменения**

Flux увидит новый коммит в репозитории (в течение `interval` или после ручного `flux reconcile`). Он обнаружит изменение в файле `app-deployment.yaml` внутри пути, который отслеживает Kustomization `my-app`. Flux автоматически применит это изменение к кластеру.

Вы можете снова смотреть логи Kustomize Controller или использовать `flux reconcile` для ускорения процесса.

**Шаг 12: Проверьте, что изменение применено в кластере**

Проверьте количество запущенных реплик или версию образа Deployment.

```bash
kubectl get deploy my-nginx-app # Замените на имя вашего Deployment
# Или для проверки образа:
kubectl get deploy my-nginx-app -o yaml | grep image
```

*   **Ожидаемый результат:** Количество реплик или версия образа в кластере должны обновиться до значения, которое вы указали в Git.

**Проверка результатов (Измеряемость):**

1.  Предоставьте ссылку на ваш Git-репозиторий (`my-gitops-flux`).
2.  Предоставьте YAML-файлы: Kustomization для вашего приложения (например, `clusters/my-cluster/my-app-kustomization.yaml`) и манифесты приложения (`apps/my-app/app-deployment.yaml`, `apps/my-app/app-service.yaml`).
3.  Предоставьте вывод команд `kubectl get gitrepositories -n flux-system` и `kubectl get kustomizations -n flux-system`, показывающий, что ресурсы созданы и готовы (`Ready`).
4.  Предоставьте коммит в Git, где вы изменили манифест приложения (количество реплик или версию образа).
5.  Предоставьте вывод команды `kubectl get deploy <имя_деплоймента>` (после того, как Flux применит изменение), демонстрирующий, что изменение было успешно применено в кластере. (Или `kubectl get deploy <имя> -o yaml | grep image`).

---

Теперь вы освоили базовые принципы работы с двумя ведущими инструментами GitOps: Argo CD (с App of Apps) и Flux CD.