# Файл 7: Безопасность Подов и Сетевые Политики

Два фундаментальных аспекта безопасности во время выполнения в Kubernetes — это контроль над тем, **что может делать под** (`Pod Security`), и контроль над тем, **с кем он может общаться** (`Network Policies`).

## 1. Безопасность Подов (Pod Security)

### 1.1. Контекст Безопасности (Security Context)

`SecurityContext` — это набор полей в манифесте пода или контейнера, который позволяет вам определять привилегии и настройки контроля доступа на уровне ядра Linux. [1, 4] Это Kubernetes-аналог флагов `--cap-drop`, `--security-opt` и других из `docker run`.

**Ключевые задачи, которые мы решаем:**

*   Запуск контейнеров от имени непривилегированного пользователя.
*   Предотвращение эскалации привилегий.
*   Ограничение доступа к файловой системе хоста.
*   Применение кастомных профилей Seccomp и AppArmor.

**Пример манифеста с `SecurityContext`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod-example
spec:
  # --- Pod-level Security Context ---
  # Эти настройки применяются ко всем контейнерам в поде
  securityContext:
    runAsUser: 1000            # Запускать все процессы с UID 1000
    runAsGroup: 3000           # Запускать все процессы с GID 3000
    runAsNonRoot: true         # Обязательно запускать от non-root пользователя
    fsGroup: 2000              # GID, который будет владеть смонтированными томами
    seccompProfile:
      type: RuntimeDefault     # Использовать стандартный seccomp профиль среды выполнения

  containers:
  - name: my-secure-container
    image: nginx:alpine
    # --- Container-level Security Context ---
    # Эти настройки могут переопределять или дополнять pod-level
    securityContext:
      # Запрещаем эскалацию привилегий (например, через suid биты)
      allowPrivilegeEscalation: false
      # Отключаем ВСЕ capabilities, так как для nginx они не нужны
      capabilities:
        drop:
        - "ALL"
      # Делаем корневую файловую систему контейнера read-only
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: nginx-cache
      mountPath: /var/cache/nginx
      # Том для записи остается доступным, даже если root ФС read-only
  volumes:
  - name: nginx-cache
    emptyDir: {}
```

### 1.2. Стандарты Безопасности Подов (Pod Security Standards)

`Pod Security Standards` (PSS) — это встроенный в Kubernetes механизм, который определяет три уровня безопасности для подов, чтобы предотвратить запуск слишком привилегированных рабочих нагрузок. [8, 9] PSS пришли на смену устаревшим `PodSecurityPolicy` (PSP). [15]

**Три уровня PSS:**

1.  **Privileged (Привилегированный):**
    *   **Описание:** Полностью открытая политика. Не накладывает никаких ограничений. [8]
    *   **Использование:** Только для доверенных системных компонентов, которым необходим доступ к хосту (например, CNI-плагины, агенты мониторинга).

2.  **Baseline (Базовый):**
    *   **Описание:** Минимально ограничительная политика, которая предотвращает известные пути эскалации привилегий. [9]
    *   **Использование:** Стандартный уровень для большинства обычных приложений. Запрещает `hostPath`, привилегированные контейнеры, `hostNetwork` и т.д.

3.  **Restricted (Ограниченный):**
    *   **Описание:** Очень строгая политика, следующая лучшим практикам по усилению безопасности.
    *   **Использование:** Для приложений, работающих с особо чувствительными данными. Требует явного указания `runAsNonRoot`, `seccompProfile` и отключения `allowPrivilegeEscalation`. [9]

#### Применение PSS на уровне неймспейса

PSS применяются с помощью **лейблов** на неймспейсах. Это делается с помощью встроенного механизма **Pod Security Admission (PSA)**. [3]

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-restricted-app
  labels:
    # enforce: Блокировать создание подов, не соответствующих политике
    pod-security.kubernetes.io/enforce: restricted
    # warn: Разрешить, но выдать предупреждение пользователю
    pod-security.kubernetes.io/warn: restricted
    # audit: Разрешить, но записать событие в аудит-лог
    pod-security.kubernetes.io/audit: restricted
    # Указываем, на какую версию PSS ориентироваться
    pod-security.kubernetes.io/enforce-version: v1.30
```

**Нюанс для Middle DevOps:** Начните с применения `baseline` политики в режиме `warn` или `audit` на существующих неймспейсах, чтобы понять, какие рабочие нагрузки ее нарушают, и постепенно исправляйте их, прежде чем переходить в режим `enforce`. [11]

---

## 2. Сетевые Политики (Network Policies)

**Задача, которую мы решаем:** Контроль трафика на 3 и 4 уровнях (IP/Port) между подами в кластере. [17, 18] Это основной инструмент для реализации **сетевой сегментации** и принципа **Zero Trust**. [13, 23]

**Важный нюанс:** По умолчанию в Kubernetes **весь трафик разрешен**. Любой под может соединиться с любым другим подом в любом неймспейсе. [21] `NetworkPolicy` — это ресурс, который позволяет создавать правила "файрвола" для подов.

### Требования

Сетевые политики не работают "из коробки". Вам необходим CNI-плагин (Container Network Interface), который их поддерживает. Популярные варианты: **Calico**, Cilium, Weave Net. [12, 25]

### Как работают Network Policies?

*   Политики привязаны к неймспейсу.
*   Они выбирают поды, к которым применяются, с помощью `podSelector`.
*   Правила `ingress` (входящий трафик) и `egress` (исходящий трафик) определяют, какой трафик разрешен.
*   Как только к поду применяется хотя бы одна политика, **весь трафик, не разрешенный явно, блокируется**.

### Практические примеры

**1. "Default Deny" — лучшая практика для начала**
Создайте эту политику в каждом неймспейсе, чтобы заблокировать весь трафик. После этого вы будете явно разрешать только необходимые коммуникации. [7, 14, 17]

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {} # Выбирает все поды в неймспейсе
  policyTypes:
  - Ingress
  - Egress
```

**2. Разрешить frontend -> backend**
Представим, что у нас есть frontend (`app: frontend`) и backend (`app: backend`). Мы хотим разрешить входящий трафик к backend только от frontend.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-from-frontend
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: backend  # Применяем эту политику к подам backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend # Разрешаем трафик от подов frontend
    ports:
    - protocol: TCP
      port: 8080 # на порт 8080
```

**3. Разрешить исходящий трафик к внешнему DNS**
Приложениям часто нужен доступ к DNS.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-dns
  namespace: my-app
spec:
  podSelector: {} # Применяем ко всем подам
  policyTypes:
  - Egress
  egress:
  - to:
    # Это правило разрешает доступ к DNS подам kube-system (CoreDNS)
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### Команды-методички

**Проверить, поддерживаются ли сетевые политики:**
Убедитесь, что у вас установлен CNI-плагин, который их реализует (например, Calico). [10, 22]

```bash
# Проверяем наличие подов CNI-плагина в kube-system
kubectl get pods -n kube-system | grep -E "calico|cilium|weave"
```

**Посмотреть существующие политики в неймспейсе:**
```bash
kubectl get networkpolicies -n my-app
```

**Отладить сетевую политику:**
Самый простой способ — запустить временный под с `netshoot` или `busybox` и попытаться установить соединение с помощью `wget` или `nc`.

```bash
# Запускаем под 'debugger' в том же неймспейсе
kubectl run debugger --image=nicolaka/netshoot -it --rm

# Внутри пода debugger:
# Пытаемся подключиться к вашему сервису. Если соединение не проходит (timeout),
# значит, сетевая политика его блокирует.
wget -T 2 -O - http://my-backend-service:8080
```