### Методичка по ДЗ: Модуль 4, Тема 2

**Название ДЗ:** Управление облачным ресурсом через Crossplane и GitOps

**Цель ДЗ:** Научиться использовать Crossplane для управления внешними (облачными) ресурсами через Kubernetes API и интегрировать этот процесс с вашим GitOps инструментом (Argo CD или Flux).

**Необходимые условия/Инструменты:**

1.  Работающий кластер Kubernetes с установленным `kubectl` (после ДЗ Модуля 1, Тема 7).
2.  Установленный Git.
3.  Аккаунт на Git-хостинге (GitHub, GitLab, Bitbucket) с существующим GitOps-репозиторием, который синхронизируется с кластером с помощью Argo CD или Flux (после ДЗ Модуля 3, Тема 2 или Тема 4).
4.  Аккаунт в облачном провайдере (AWS, Azure, GCP) с правами на создание выбранного ресурса (например, S3 bucket, PostgreSQL instance, ServiceAccount).
5.  Настроенные учетные данные для доступа к облаку (Access Key/Secret Key для AWS, Service Principal для Azure, Service Account Key для GCP).

**Подробные шаги:**

**Шаг 1: Установите Crossplane**

Установите базовые компоненты Crossplane в ваш кластер.

```bash
# Добавьте репозиторий Crossplane Helm
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Установите Crossplane (рекомендуется в отдельный неймспейс crossplane-system)
kubectl create namespace crossplane-system
helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --wait
```

*   **Проверка:** Убедитесь, что поды Crossplane запущены:
    ```bash
    kubectl get pods -n crossplane-system
    ```

**Шаг 2: Установите Провайдер Crossplane для вашего облака**

Выберите провайдер для вашего облака и установите его.

*   **Для AWS:**
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
      name: provider-aws
    spec:
      package: xpkg.upbound.io/crossplane/provider-aws:v0.41.0 # Уточните актуальную версию
    EOF
    ```
*   **Для Azure:**
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
      name: provider-azure
    spec:
      package: xpkg.upbound.io/crossplane/provider-azure:v0.40.0 # Уточните актуальную версию
    EOF
    ```
*   **Для GCP:**
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
      name: provider-gcp
    spec:
      package: xpkg.upbound.io/crossplane/provider-gcp:v0.40.0 # Уточните актуальную версию
    EOF
    ```
*   **Проверка:** Убедитесь, что поды провайдера запущены:
    ```bash
    kubectl get pods -n crossplane-system -l xpkg.crossplane.io/pkg=provider-aws # или provider-azure, provider-gcp
    ```
    Убедитесь, что статус провайдера `INSTALLED` и `HEALTHY`:
    ```bash
    kubectl get providers
    ```

**Шаг 3: Настройте учетные данные облака как Secret в Kubernetes**

Crossplane провайдеру нужны учетные данные для взаимодействия с облачным API. Храните их в Secret.

*   **Для AWS (с Access Key/Secret Key):**
    ```bash
    # Создайте файл aws-creds.conf с вашими ключами
    cat <<EOF > aws-creds.conf
    [default]
    aws_access_key_id = <ВАШ_ACCESS_KEY_ID>
    aws_secret_access_key = <ВАШ_SECRET_ACCESS_KEY>
    EOF

    # Создайте Secret из файла
    kubectl create secret generic aws-creds -n crossplane-system --from-file=credentials=./aws-creds.conf

    # Удалите временный файл
    rm aws-creds.conf
    ```
*   **Для Azure (с Service Principal):** Создайте JSON-файл с учетными данными Service Principal и создайте Secret типа `Opaque`. Подробности в документации `provider-azure`.
*   **Для GCP (с Service Account Key JSON):** Создайте JSON-файл с ключом Service Account и создайте Secret типа `Opaque`. Подробности в документации `provider-gcp`.

**Шаг 4: Создайте ресурс ProviderConfig, ссылающийся на Secret**

ProviderConfig сообщает провайдеру, какие учетные данные использовать.

*   **Для AWS:**
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: aws.upbound.io/v1beta1 # Уточните apiVersion для вашей версии provider-aws
    kind: ProviderConfig
    metadata:
      name: default # Имя ProviderConfig (можно использовать другое, но 'default' часто удобно)
    spec:
      credentials:
        source: Secret # Указываем, что учетные данные в Secret
        secretRef:
          namespace: crossplane-system # Неймспейс, где находится Secret
          name: aws-creds # Имя Secret
          key: credentials # Ключ в Secret, содержащий учетные данные
      region: us-east-1 # Укажите регион по умолчанию
    EOF
    ```
*   **Для Azure/GCP:** См. документацию соответствующих провайдеров для структуры ProviderConfig.

*   **Проверка:** Убедитесь, что ProviderConfig готов:
    ```bash
    kubectl get providerconfigs
    ```
    Статус должен быть `Ready`.

**Шаг 5: Клонируйте ваш GitOps репозиторий (Argo CD или Flux)**

Используйте репозиторий, который уже синхронизируется с вашим кластером из ДЗ Модуля 3.

```bash
git clone <URL_вашего_gitops_репозитория>
cd <название_репозитория>
```

**Шаг 6: Создайте директорию для манифестов Crossplane и напишите YAML для облачного ресурса**

В вашем GitOps репозитории создайте директорию для инфраструктуры, например `infra/crossplane`.

```bash
mkdir -p infra/crossplane
```

Теперь напишите YAML-манифест для создания простого облачного ресурса, используя CRD вашего провайдера.

Пример `infra/crossplane/s3-bucket.yaml` для AWS S3 Bucket:

```yaml
# infra/crossplane/s3-bucket.yaml
apiVersion: s3.aws.upbound.io/v1beta1 # API-версия для S3 Bucket в provider-aws (может отличаться)
kind: Bucket # Тип ресурса в Crossplane
metadata:
  name: my-gitops-crossplane-bucket # Имя Kubernetes объекта, который представляет S3 Bucket
spec:
  forProvider:
    region: us-east-1 # Регион для S3 bucket (переопределяет ProviderConfig)
    acl: private # ACL для S3 bucket
  providerConfigRef:
    name: default # Ссылка на ProviderConfig ресурс, созданный ранее
```

*   **Важно:** Убедитесь, что вы используете правильную `apiVersion` и `kind` для ресурса в вашем провайдере. Их можно найти в документации провайдера или с помощью команды `kubectl api-resources | grep <имя_провайдера>`. Имя S3 бакета (`spec.forProvider.bucket`) можно добавить, если нужно, но Crossplane часто генерирует уникальное имя по умолчанию на основе `metadata.name`.

**Шаг 7: Настройте ваш GitOps инструмент для синхронизации новой директории**

Теперь нужно сообщить Argo CD или Flux, чтобы они синхронизировали папку `infra/crossplane`.

*   **Если используете Argo CD:** Отредактируйте ваш корневой Application (root.yaml) или создайте новый дочерний Application, который указывает на папку `infra/crossplane`.
    *   *Вариант 1 (Редактировать Root App):* В `root.yaml` добавьте еще один `path` в `source`.
        ```yaml
        # root.yaml (фрагмент)
        spec:
          source:
            repoURL: <URL_вашего_gitops_репозитория>
            targetRevision: HEAD
            path: . # Синхронизировать все из корня репозитория
          destination:
            server: https://kubernetes.default.svc.cluster.local
            namespace: argocd # Корневой App of Apps смотрит в неймспейс Argo CD
          # ... другие настройки ...
        ```
        И убедитесь, что Argo CD синхронизирует корневой Application (он должен быть создан командой `kubectl apply -n argocd -f root.yaml`).
    *   *Вариант 2 (Новый дочерний App):* Создайте новый файл `infra/crossplane/crossplane-app.yaml` с Argo CD Application ресурсом, который ссылается на `path: infra/crossplane`. Затем убедитесь, что ваш корневой Application (root.yaml) синхронизирует директорию `infra/`.
        ```yaml
        # infra/crossplane/crossplane-app.yaml
        apiVersion: argoproj.io/v1alpha1
        kind: Application
        metadata:
          name: crossplane-infra
          namespace: argocd
        spec:
          project: default
          source:
            repoURL: <URL_вашего_gitops_репозитория>
            targetRevision: HEAD
            path: infra/crossplane # Путь к манифестам Crossplane
          destination:
            server: https://kubernetes.default.svc.cluster.local
            namespace: default # Или crossplane-system, куда вы хотите создать ресурсы
          syncPolicy:
            automated:
              prune: true
              selfHeal: true
            syncOptions:
            - CreateNamespace=true
        ```
        Убедитесь, что ваш корневой Application в `root.yaml` указывает на папку `infra`, например `path: .` или `path: infra`.

*   **Если используете Flux CD:** Создайте новый Kustomization ресурс в папке, которую синхронизирует Flux (например, в `clusters/my-cluster`).
    Создайте файл `clusters/my-cluster/crossplane-kustomization.yaml`:
    ```yaml
    # clusters/my-cluster/crossplane-kustomization.yaml
    apiVersion: kustomize.toolkit.fluxcd.io/v1
    kind: Kustomization
    metadata:
      name: crossplane-infra
      namespace: flux-system
    spec:
      interval: 1m
      sourceRef:
        kind: GitRepository
        name: flux-system # Ссылка на ваш GitRepository ресурс Flux
      path: ./infra/crossplane # Путь в Git к манифестам Crossplane
      prune: true
      timeout: 2m
      # targetNamespace: default # Или crossplane-system, куда вы хотите создать ресурсы
    ```

**Шаг 8: Добавьте, закоммитьте и отправьте все новые файлы в Git**

```bash
git add .
git commit -m "Add Crossplane infra manifests and GitOps config"
git push origin main # Или master
```

**Шаг 9: Наблюдайте за синхронизацией GitOps инструмента и Crossplane**

Ваш GitOps инструмент (Argo CD или Flux) увидит новые файлы и применит их к кластеру.
*   Если вы добавили новый Argo CD Application, он создастся.
*   Если вы добавили новый Flux Kustomization, он создастся.

Затем GitOps инструмент применит YAML-файл вашего облачного ресурса (`s3-bucket.yaml`). Kubernetes API получит этот ресурс, а контроллер Crossplane для S3 (или универсальный контроллер) увидит его и начнет взаимодействовать с облачным API для создания реального ресурса.

*   **Для Argo CD:** Смотрите веб-интерфейс. Должен появиться новый Application (если создавали отдельный) или корневой Application покажет новый ресурс. Дождитесь, пока ресурс перейдет в статус `Healthy` и `Synced`.
*   **Для Flux CD:** Смотрите логи Kustomize Controller (`kubectl logs -n flux-system -l app=kustomize-controller -f`). Или используйте `flux reconcile kustomization crossplane-infra --namespace=flux-system` и `flux get kustomizations -n flux-system`. Убедитесь, что Kustomization `crossplane-infra` синхронизирован.

**Шаг 10: Проверьте создание ресурса в кластере (через CRD) и в облаке**

*   **В кластере:** Проверьте, что Crossplane ресурс был создан и находится в статусе `Ready`.
    ```bash
    kubectl get Bucket my-gitops-crossplane-bucket # Замените Bucket на тип вашего ресурса и имя
    # Или более общий способ:
    kubectl get <тип_вашего_crossplane_ресурса> <имя_ресурса> -n default # Укажите правильный неймспейс
    ```
    *   **Ожидаемый результат:** Ресурс должен появиться и через некоторое время (когда Crossplane создаст его в облаке) его статус должен быть `Ready`, а в полях `STATUS` или `EVENTS` должно быть указание на успешное создание.

*   **В облаке:** Зайдите в консоль вашего облачного провайдера или используйте его CLI, чтобы убедиться, что реальный ресурс был создан.
    ```bash
    # Пример для AWS S3:
    aws s3 ls | grep my-gitops-crossplane-bucket # Или часть имени
    # Пример для AWS S3 Tagging:
    aws s3api get-bucket-tagging --bucket my-gitops-crossplane-bucket --region us-east-1
    ```

**Шаг 11: Измените параметр ресурса в Git**

Отредактируйте YAML-файл вашего Crossplane ресурса (`infra/crossplane/s3-bucket.yaml`) в локальной копии репозитория. Например, добавьте тег.

```yaml
# infra/crossplane/s3-bucket.yaml (фрагмент)
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-gitops-crossplane-bucket
spec:
  forProvider:
    region: us-east-1
    acl: private
    tags: # Добавляем теги
      Environment: GitOpsManaged
      Tool: Crossplane
  providerConfigRef:
    name: default
```

**Шаг 12: Закоммитьте и отправьте изменение в Git**

```bash
git add infra/crossplane/s3-bucket.yaml
git commit -m "Add tags to S3 bucket via Crossplane"
git push origin main # Или master
```

**Шаг 13: Наблюдайте за автоматическим применением изменения**

Ваш GitOps инструмент увидит новый коммит, применит обновленный YAML к кластеру. Контроллер Crossplane увидит изменение в Kubernetes объекте `Bucket` и вызовет облачной API для обновления реального S3 бакета.

*   **Для Argo CD:** Смотрите статус Application в веб-интерфейсе. Он должен показать, что ресурс `Bucket` находится в процессе синхронизации, затем вернется в `Synced` и `Healthy`.
*   **Для Flux CD:** Смотрите логи Kustomize Controller.

**Шаг 14: Проверьте изменение ресурса в облаке**

Снова проверьте ресурс в консоли облака или через CLI, чтобы убедиться, что изменение (например, теги) было применено.

**Проверка результатов (Измеряемость):**

1.  Предоставьте ссылку на ваш GitOps-репозиторий.
2.  Предоставьте YAML-файлы:
    *   Манифест вашего Crossplane ProviderConfig (если не использовали ProviderConfig по умолчанию).
    *   Манифест вашего Crossplane ресурса (например, `infra/crossplane/s3-bucket.yaml`).
    *   Конфигурацию вашего GitOps инструмента, который синхронизирует эту папку (Argo CD Application/AppSet или Flux Kustomization).
3.  Предоставьте вывод команд, подтверждающих установку Crossplane и провайдера: `kubectl get pods -n crossplane-system` и `kubectl get providers`.
4.  Предоставьте вывод команды `kubectl get providerconfigs`, подтверждающий создание ProviderConfig.
5.  Предоставьте вывод команды `kubectl get <тип_вашего_crossplane_ресурса> <имя_ресурса> -n <неймспейс>`, показывающий, что ресурс создан и находится в статусе `Ready`.
6.  Предоставьте скриншот из консоли облачного провайдера или вывод команды облачного CLI, подтверждающий создание реального облачного ресурса.
7.  Предоставьте ссылку на коммит в Git, который изменил параметр ресурса (например, добавил теги).
8.  Предоставьте скриншот из консоли облачного провайдера или вывод команды облачного CLI, подтверждающий, что параметр ресурса (например, теги) был обновлен.
9.  (Опционально) Предоставьте скриншоты из веб-интерфейса Argo CD или логи контроллеров Flux, показывающие процесс синхронизации и применения Crossplane ресурсов.

---

Это ДЗ демонстрирует мощь Crossplane в управлении облачной инфраструктурой через знакомый Kubernetes API, полностью вписываясь в парадигму GitOps.

Теперь у нас осталось только одно ДЗ в Модуле 4, связанное с обновлением приложений через GitOps, и затем Проектная работа.

Готовы к последнему ДЗ из Модуля 4?