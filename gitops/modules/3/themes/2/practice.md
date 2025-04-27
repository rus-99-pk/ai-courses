Отлично! Переходим к Модулю 3. В нем есть два домашних задания, связанных с основными инструментами GitOps: Argo CD и Flux. Начнем с Argo CD и паттерна App of Apps.

---

### Методичка по ДЗ: Модуль 3, Тема 2

**Название ДЗ:** Развертывание нескольких приложений с помощью App of Apps (с Argo CD)

**Цель ДЗ:** Научиться устанавливать Argo CD и использовать паттерн "App of Apps" для централизованного управления развертыванием нескольких приложений из Git-репозитория.

**Необходимые условия/Инструменты:**

1.  Работающий кластер Kubernetes с установленным `kubectl` (после ДЗ Модуля 1, Тема 7).
2.  Установленный Git.
3.  Аккаунт на Git-хостинге (GitHub, GitLab, Bitbucket).
4.  YAML-манифесты двух разных простых приложений (Deployment, Service). Можно взять манифесты из ДЗ Модуля 1, Тема 2 и Тема 3, или создать новые простые.

**Подробные шаги:**

**Шаг 1: Установите Argo CD**

Установка Argo CD включает создание неймспейса и применение его установочного манифеста.

```bash
# Создайте неймспейс для Argo CD
kubectl create namespace argocd

# Установите Argo CD (используйте последнюю стабильную версию манифеста)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.0/manifests/install.yaml # Уточните актуальную версию!
```

*   **Пояснения:** Эта команда развернет все необходимые компоненты Argo CD (контроллеры, API Server, UI) в неймспейсе `argocd`.
*   **Проверка:** Подождите несколько минут и убедитесь, что все поды в неймспейсе `argocd` перешли в состояние `Running`.
    ```bash
    kubectl get pods -n argocd
    ```

**Шаг 2: Получите доступ к веб-интерфейсу Argo CD**

Веб-интерфейс удобен для визуализации состояния.

*   **Получите пароль администратора:**
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```
    Сохраните этот пароль. Имя пользователя по умолчанию: `admin`.
*   **Перенаправьте порт для доступа к API Server:**
    ```bash
    kubectl port-forward service/argocd-server -n argocd 8080:443
    ```
    Теперь веб-интерфейс будет доступен по адресу `https://localhost:8080`. Возможно, потребуется принять сертификат безопасности.
*   **Войдите в веб-интерфейс:** Используйте логин `admin` и полученный пароль.

**Шаг 3: Создайте новый Git-репозиторий для App of Apps**

Создайте новый **приватный** или **публичный** репозиторий на вашем Git-хостинге. Назовите его, например, `my-gitops-apps`.

**Шаг 4: Клонируйте репозиторий и создайте структуру папок**

Клонируйте репозиторий на свой компьютер и создайте структуру:

```bash
git clone <URL_вашего_нового_репозитория>
cd my-gitops-apps
mkdir apps # Папка для дочерних приложений
mkdir apps/app1
mkdir apps/app2
```

**Шаг 5: Подготовьте манифесты для двух простых приложений**

Выберите два простых приложения. Например:

*   `app1`: Nginx (из ДЗ Модуля 1, Тема 2 - Deployment, Service ClusterIP).
*   `app2`: Apache HTTP Server (аналогично: Deployment с образом `httpd:latest`, Service ClusterIP).

Создайте YAML-файлы манифестов для каждого приложения и поместите их в соответствующие директории:

*   Создайте `apps/app1/deployment.yaml` и `apps/app1/service.yaml` (для Nginx).
*   Создайте `apps/app2/deployment.yaml` и `apps/app2/service.yaml` (для Apache).

**Пример `apps/app1/deployment.yaml` (Nginx):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-nginx-deployment
spec:
  replicas: 2 # Например, 2 реплики
  selector:
    matchLabels:
      app: app1-nginx
  template:
    metadata:
      labels:
        app: app1-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**Пример `apps/app1/service.yaml` (Nginx):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-nginx-service
spec:
  selector:
    app: app1-nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

**Пример `apps/app2/deployment.yaml` (Apache):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-apache-deployment
spec:
  replicas: 1 # Например, 1 реплика
  selector:
    matchLabels:
      app: app2-apache
  template:
    metadata:
      labels:
        app: app2-apache
    spec:
      containers:
      - name: apache
        image: httpd:latest # Образ Apache
        ports:
        - containerPort: 80
```

**Пример `apps/app2/service.yaml` (Apache):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app2-apache-service
spec:
  selector:
    app: app2-apache
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

**Шаг 6: Создайте Application-ресурсы для дочерних приложений**

Внутри каждой папки приложения создайте Argo CD `Application` ресурс, который будет указывать на манифесты в этой же папке.

Создайте файл `apps/app1/app1-argocd-app.yaml`:

```yaml
# apps/app1/app1-argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app1-nginx # Имя Argo CD Application
  namespace: argocd # Где будет создан Application ресурс (в неймспейсе Argo CD)
spec:
  project: default # Проект Argo CD
  source:
    repoURL: <URL_вашего_git_репозитория> # URL вашего репозитория my-gitops-apps
    targetRevision: HEAD # Или master/main, конкретный тег или хеш
    path: apps/app1 # Путь внутри репозитория к манифестам этого приложения
  destination:
    server: https://kubernetes.default.svc.cluster.local # Адрес кластера (стандартный)
    namespace: default # Неймспейс в кластере, куда будут деплоиться манифесты приложения (можно создать новый)
  syncPolicy:
    automated:
      prune: true # Удалять ресурсы, которых нет в Git
      selfHeal: true # Автоматически синхронизировать при обнаружении дрифта
    syncOptions:
    - CreateNamespace=true # Создать неймспейс назначения, если он не существует
```

Создайте файл `apps/app2/app2-argocd-app.yaml`:

```yaml
# apps/app2/app2-argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app2-apache # Имя Argo CD Application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <URL_вашего_git_репозитория>
    targetRevision: HEAD
    path: apps/app2 # Путь к манифестам app2
  destination:
    server: https://kubernetes.default.svc.cluster.local
    namespace: default # Или другой неймспейс
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Важно:** Замените `<URL_вашего_git_репозитория>` на актуальный URL вашего репозитория `my-gitops-apps`.

**Шаг 7: Создайте корневой Application-ресурс (App of Apps)**

Создайте файл `root.yaml` в корне вашего репозитория `my-gitops-apps`. Этот Application будет следить за папкой `apps/` и применять все `Application`-ресурсы, которые в ней найдет.

```yaml
# root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-apps-root # Имя корневого Argo CD Application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <URL_вашего_git_reпозитория>
    targetRevision: HEAD
    path: apps # Путь к директории, содержащей дочерние Application-ресурсы
  destination:
    server: https://kubernetes.default.svc.cluster.local
    namespace: argocd # Дочерние Application-ресурсы будут созданы в неймспейсе Argo CD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Важно:** Снова замените `<URL_вашего_git_репозитория>`. `destination.namespace` для корневого App of Apps должен быть `argocd`, потому что именно там должны создаваться дочерние ресурсы `Application`.

**Шаг 8: Добавьте, закоммитьте и отправьте все новые файлы в Git**

```bash
git add . # Добавляем все файлы в текущей директории и поддиректориях
git commit -m "Add App of Apps structure and initial apps"
git push origin main # Или master
```

**Шаг 9: Сообщите Argo CD о корневом Application**

Argo CD по умолчанию не знает о вашем репозитории. Вы должны создать корневой Application-ресурс вручную в кластере один раз.

```bash
# Убедитесь, что вы находитесь в директории my-gitops-apps
kubectl apply -n argocd -f root.yaml
```

*   **Пояснения:** Мы применяем манифест `root.yaml` в неймспейсе `argocd`. Argo CD увидит этот ресурс Application, клонирует указанный репозиторий, найдет в папке `apps` другие `Application`-ресурсы и начнет их создавать и синхронизировать.

**Шаг 10: Наблюдайте за синхронизацией в Argo CD**

*   Вернитесь в веб-интерфейс Argo CD (`https://localhost:8080`).
*   Вы должны увидеть корневой Application (`my-apps-root`). Кликните на него.
*   Он должен показать другие Application (`app1-nginx`, `app2-apache`), которые он управляет.
*   Argo CD начнет автоматически синхронизировать эти дочерние Application, которые, в свою очередь, развернут Deployment и Service для Nginx и Apache.
*   Дождитесь, пока все Application перейдут в статус `Synced` и `Healthy`.

**Шаг 11: Проверьте развернутые приложения в кластере**

Используйте `kubectl get` для проверки подов и Services в неймспейсе, куда вы их деплоили (например, `default`).

```bash
kubectl get deploy -n default # Убедитесь, что app1-nginx-deployment и app2-apache-deployment запущены
kubectl get svc -n default # Убедитесь, что app1-nginx-service и app2-apache-service существуют
kubectl get pods -l app=app1-nginx -n default # Убедитесь, что поды Nginx запущены
kubectl get pods -l app=app2-apache -n default # Убедитесь, что поды Apache запущены
```

**Проверка результатов (Измеряемость):**

1.  Предоставьте ссылку на ваш Git-репозиторий (`my-gitops-apps`) с настроенной структурой App of Apps.
2.  Предоставьте вывод команды `kubectl get applications -n argocd`, показывающий корневое (`my-apps-root`) и дочерние (`app1-nginx`, `app2-apache`) Application в неймспейсе `argocd`.
3.  Предоставьте скриншоты из веб-интерфейса Argo CD:
    *   Общий вид с тремя Application (корневым и двумя дочерними).
    *   Детализация корневого Application, показывающая, что он управляет дочерними.
    *   Детализация одного из дочерних Application (например, `app1-nginx`), показывающая его ресурсы (Deployment, Service) в статусе `Synced` и `Healthy`.

---

Это ДЗ демонстрирует, как Argo CD может управлять сложными структурами приложений из Git с помощью паттерна App of Apps, обеспечивая централизованный контроль и автоматическую синхронизацию.

Готовы к следующему ДЗ по Flux?