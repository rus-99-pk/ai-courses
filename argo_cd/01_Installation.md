# 01 - Установка и базовая настройка ArgoCD

### Цель урока
Установить ArgoCD в Kubernetes-кластер, получить доступ к его веб-интерфейсу (UI) и интерфейсу командной строки (CLI).

### Какую проблему решаем?
У нас есть пустой кластер, и нам нужен инструмент для реализации GitOps. На этом шаге мы подготовим рабочее окружение.

### Практика: Установка

#### Шаг 1: Создание неймспейса
ArgoCD принято устанавливать в отдельный неймспейс `argocd`.

```bash
kubectl create namespace argocd
```

#### Шаг 2: Применение манифестов установки
ArgoCD устанавливается как набор Kubernetes-ресурсов (Deployments, Services, CRDs и т.д.). Разработчики предоставляют готовый манифест для быстрой установки.

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Эта команда скачает последнюю стабильную версию манифеста и применит ее в неймспейсе `argocd`.

#### Шаг 3: Проверка статуса установки
Убедимся, что все поды ArgoCD запустились и находятся в состоянии `Running`.

```bash
kubectl get pods -n argocd
```
Вы должны увидеть примерно следующий вывод:
```
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          98s
argocd-applicationset-controller-57c9f87c9-z8gkh   1/1     Running   0          98s
argocd-dex-server-7568c8d8b4-v6h6v                  1/1     Running   0          98s
argocd-notifications-controller-6d6bb985cd-z75k9   1/1     Running   0          98s
argocd-redis-ha-haproxy-84b958f698-qgsjw            1/1     Running   0          98s
argocd-repo-server-5f5994895c-kpxwx                 1/1     Running   0          98s
argocd-server-5878b688d6-gknxf                      1/1     Running   0          98s
```

### Практика: Доступ к ArgoCD

#### Доступ к веб-интерфейсу (UI)

По умолчанию `argocd-server` доступен только внутри кластера. Для доступа с локальной машины мы используем `port-forward`.

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Теперь UI доступен в браузере по адресу `https://localhost:8080`. Браузер будет ругаться на самоподписанный сертификат — это нормально, просто примите его.

#### Получение пароля администратора

Начальный пароль для пользователя `admin` генерируется автоматически и хранится в Kubernetes-секрете.

```bash
# Для Linux/macOS
argocd admin initial-password -n argocd

# Или через kubectl, если CLI еще не установлен
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Используйте логин `admin` и полученный пароль для входа в UI.

#### Установка и использование CLI
ArgoCD CLI — мощный инструмент для управления приложениями из командной строки.

**Установка (macOS с Homebrew):**
```bash
brew install argocd
```
**Установка (Linux):**
```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

**Логин через CLI:**
Теперь, когда CLI установлен, можно залогиниться.

```bash
# Указываем адрес нашего сервера (который мы пробросили через port-forward)
# --insecure нужен из-за самоподписанного сертификата
argocd login localhost:8080 --username admin --password <ВАШ_ПАРОЛЬ> --insecure
```

После успешного входа вы готовы к созданию своего первого приложения!

### Команды-методички

```bash
# Создать неймспейс для ArgoCD
kubectl create namespace argocd

# Установить ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Проверить статус подов
kubectl get pods -n argocd

# Пробросить порт для доступа к UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Получить начальный пароль администратора
argocd admin initial-password -n argocd

# Установить CLI (macOS)
brew install argocd

# Залогиниться через CLI
argocd login localhost:8080 --insecure
```

**Рекомендация**: После первого входа смените пароль администратора через команду `argocd account update-password` или в настройках UI.