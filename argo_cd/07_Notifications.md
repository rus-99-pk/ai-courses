# 07 - Настройка уведомлений

### Цель урока
Научиться настраивать компонент Argo CD Notifications для отправки уведомлений о состоянии приложений в Slack, Email или другие сервисы.

### Какую проблему решаем?
ArgoCD отлично справляется с синхронизацией приложений, но как узнать, что:
*   Синхронизация прошла успешно?
*   Синхронизация завершилась с ошибкой?
*   Приложение перешло в состояние `Degraded` после развертывания?
*   Кто-то вручную изменил состояние, и ArgoCD выполнил `self-heal`?

Постоянно смотреть в UI ArgoCD — не вариант. Нам нужны автоматические уведомления, которые приходят туда, где мы работаем (например, в командный чат Slack).

**Важно:** Контроллер `argocd-notifications-controller` уже установлен вместе с ArgoCD, если вы использовали стандартный манифест `install.yaml`. Нам нужно только его настроить.

### Теория: Как работают уведомления

Система уведомлений состоит из трех основных компонентов, которые настраиваются в ConfigMap `argocd-notifications-cm`:

1.  **Сервисы (Services):** Описывают, *куда* отправлять уведомления. Это конфигурация для подключения к внешним системам, таким как Slack, Email, Telegram, Microsoft Teams и т.д. Вы можете определить несколько сервисов.
    *   **Пример:** Сервис для отправки сообщений в Slack с использованием `token`.

2.  **Триггеры (Triggers):** Описывают, *когда* отправлять уведомления. Триггер — это условие, которое срабатывает при определенном событии в жизненном цикле приложения.
    *   **Пример:** Триггер `on-sync-failed` срабатывает, когда синхронизация приложения завершается с ошибкой.

3.  **Шаблоны (Templates):** Описывают, *что* отправлять в уведомлении. Это шаблоны текста сообщения, которые могут содержать переменные (имя приложения, коммит, автор и т.д.).
    *   **Пример:** Шаблон сообщения для Slack, который форматирует информацию о сбое в удобном виде.

Связь между ними простая: **Триггер** срабатывает и использует **Шаблон** для формирования сообщения, которое затем отправляется через указанный **Сервис**.

### Практика: Настройка уведомлений в Slack

Давайте настроим отправку уведомлений в Slack о статусе синхронизации.

#### Шаг 1: Получение Slack Bot Token
1.  Перейдите на `https://api.slack.com/apps`.
2.  Создайте новое приложение (`Create New App` -> `From scratch`).
3.  В разделе `Features` -> `OAuth & Permissions`, в секции `Scopes` -> `Bot Token Scopes`, добавьте право `chat:write`.
4.  Вверху страницы нажмите `Install to Workspace` и разрешите установку.
5.  Скопируйте `Bot User OAuth Token`. Он выглядит как `xoxb-...`. **Это секрет!**

#### Шаг 2: Сохранение токена в Kubernetes Secret
Мы не будем хранить токен в ConfigMap в открытом виде.

```bash
kubectl create secret generic argocd-notifications-secret -n argocd --from-literal=slack-token=<ВАШ_SLACK_TOKEN>
```

#### Шаг 3: Настройка ConfigMap `argocd-notifications-cm`
Теперь отредактируем ConfigMap, чтобы описать сервис, триггеры и шаблоны.

```bash
# Открываем ConfigMap для редактирования
kubectl edit configmap argocd-notifications-cm -n argocd
```

Вставьте в поле `data:` следующий YAML. Если `data:` уже существует, объедините содержимое.

```yaml
data:
  # 1. Описываем сервис для подключения к Slack
  service.slack: |
    token: $slack-token # Ссылка на наш секрет

  # 2. Определяем шаблоны сообщений
  template.app-sync-status: |
    message: |
      {{if .application.status.operationState.phase == "Succeeded"}}
      ✅ Приложение `{{.app.metadata.name}}` успешно синхронизировано.
      {{else}}
      ❌ Не удалось синхронизировать приложение `{{.app.metadata.name}}`.
      {{end}}
      *Проект:* `{{.app.spec.project}}`
      *Репозиторий:* `{{.app.spec.source.repoURL}}`
      *Ревизия:* `{{.app.status.sync.revision}}`
      <{{.context.argocdUrl}}/applications/{{.app.metadata.name}}|Открыть в ArgoCD>
  
  # 3. Определяем триггеры, которые будут использовать шаблоны и сервис
  trigger.on-sync-status-changed: |
    - when: app.status.operationState.phase in ['Succeeded', 'Failed', 'Error']
      send:
      - app-sync-status # Имя нашего шаблона
      to:
      - slack: # Имя нашего сервиса
          channel: '#argo-alerts' # Укажите ваш канал в Slack

  # Также можно добавить подписку по умолчанию для всех приложений
  # context: |
  #   defaultTriggers: |
  #     - on-sync-status-changed
```

**Что мы сделали:**
*   `service.slack`: Определили сервис `slack` и указали, что токен нужно брать из переменной окружения `$slack-token` (контроллер сам подтянет ее из секрета `argocd-notifications-secret`).
*   `template.app-sync-status`: Создали шаблон `app-sync-status`, который генерирует разный текст в зависимости от успеха или провала операции.
*   `trigger.on-sync-status-changed`: Создали триггер, который срабатывает, когда фаза операции меняется на `Succeeded`, `Failed` или `Error`, и отправляет сообщение по шаблону `app-sync-status` в канал `#argo-alerts`.

#### Шаг 4: Привязка уведомлений к приложению

Теперь нужно "сказать" нашему приложению, что оно должно использовать эти уведомления. Это делается через аннотацию в манифесте `Application`.

Отредактируйте ваше приложение (например, `guestbook`):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  annotations:
    # Включаем подписку на триггер
    notifications.argoproj.io/subscribe.on-sync-status-changed.slack: '#argo-alerts'
spec:
# ... остальная часть манифеста
```

**Что делает эта аннотация:**
`notifications.argoproj.io/subscribe.<имя-триггера>.<имя-сервиса>: '<куда-отправлять>'`

Она подписывает это конкретное приложение на триггер `on-sync-status-changed` и указывает, что уведомление нужно отправить через сервис `slack` в канал `#argo-alerts`.

**Примените изменения:**
Обновите манифест вашего приложения в Git (если вы управляете им через GitOps) или через `kubectl apply`.

#### Шаг 5: Проверка
1.  Добавьте вашего Slack-бота в канал `#argo-alerts`.
2.  Запустите синхронизацию приложения `guestbook` в ArgoCD.
3.  После завершения синхронизации вы должны получить уведомление в Slack!

### Другие полезные триггеры

Вы можете настроить множество других полезных уведомлений:

*   **`on-health-degraded`**: Срабатывает, когда приложение становится `Degraded`.
    ```yaml
    # В ConfigMap
    trigger.on-health-degraded: |
      - when: app.status.health.status == 'Degraded'
        send: [app-health-degraded-template]
        to: [slack: { channel: '#argo-alerts' }]
    
    # В аннотации Application
    notifications.argoproj.io/subscribe.on-health-degraded.slack: '#argo-alerts'
    ```

*   **`on-sync-running`**: Срабатывает, когда синхронизация задерживается дольше определенного времени.
    ```yaml
    # В ConfigMap
    trigger.on-sync-running: |
      - when: app.status.operationState.phase == 'Running' and time.Now().Sub(time.Parse(app.status.operationState.startedAt)) > duration("10m")
        oncePer: app.status.operationState.syncResult.revision
        send: [app-sync-running-template]
        to: [slack: { channel: '#argo-alerts' }]
    ```

Система уведомлений очень гибкая и позволяет точно настроить, какую информацию и при каких условиях вы хотите получать, делая ваш GitOps-процесс прозрачным и управляемым.