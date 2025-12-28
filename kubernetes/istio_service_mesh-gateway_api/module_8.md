## Модуль 8: Безопасность (Security) и Zero Trust

Обычно в кластере Kubernetes сеть "плоская": любой под может отправить запрос любому поду. Если хакер взломает один слабый микросервис, он получает доступ ко всему кластеру.
Istio реализует концепцию **Zero Trust** (Никому не доверяй).

### 8.1 Mutual TLS (mTLS) — Шифрование и Аутентификация
mTLS решает две задачи:
1.  **Шифрование:** Трафик между сервисами нельзя прослушать (tcpdump покажет мусор).
2.  **Идентификация:** Сервис А криптографически доказывает сервису Б, что он действительно Сервис А.

По умолчанию Istio работает в режиме **Permissive** (Разрешающий). Это значит: "Если у соседа есть сертификат — шифруем, если нет (например, это старый сервис без Istio) — общаемся открытым текстом".

#### Включаем режим STRICT (Строгий)
Мы запретим любое общение без сертификатов в неймспейсе `bookinfo`.

Файл `strict-mtls.yaml`:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT # Только mTLS! Никакого открытого текста.
```

Применяем:
```bash
kubectl apply -f strict-mtls.yaml
```

**Как проверить, что это работает?**

Наши текущие сервисы (reviews, productpage) продолжат работать, так как у них у всех есть sidecar-proxy, которые автоматически обмениваются сертификатами.
Чтобы увидеть блокировку, нам нужен "чужак".

1.  Создайте "голый" под (без Istio) в другом неймспейсе:
    ```bash
    kubectl create ns legacy
    kubectl run sleep --image=curlimages/curl -n legacy --command -- /bin/sleep 3600
    ```
2.  Попробуйте постучаться из него в наш защищенный сервис:
    ```bash
    # Ждем запуска пода
    kubectl wait --for=condition=ready pod/sleep -n legacy
    
    # Пытаемся сделать запрос
    kubectl exec -n legacy sleep -- curl -v http://productpage.bookinfo.svc.cluster.local:9080
    ```
3.  **Результат:** Вы увидите ошибку соединения `Recv failure: Connection reset by peer` или зависание. Envoy на стороне `productpage` отверг подключение, потому что у `sleep` нет сертификата Istio. Замок закрыт!

### 8.2 AuthorizationPolicy — Кто куда может ходить?
Теперь, когда мы знаем, что все личности подтверждены (аутентификация), настроим права доступа (авторизация).

**Задача:** Мы хотим, чтобы к сервису `ratings` (база оценок) мог обращаться **только** сервис `reviews`. Фронтенд `productpage` не должен лазить в базу оценок напрямую.

#### Шаг 1: Запрет по умолчанию (опционально, но рекомендуется)
В Zero Trust мы сначала все запрещаем, потом разрешаем нужное. Но для простоты примера мы сразу напишем "Allow Policy".

#### Шаг 2: Разрешаем доступ только для Reviews
Istio использует **SPIFFE ID** для идентификации. Формат такой:
`cluster.local/ns/<NAMESPACE>/sa/<SERVICE_ACCOUNT>`

Файл `allow-reviews-only.yaml`:
```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: ratings-policy
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings # Политика вешается на сервис Ratings
  action: ALLOW
  rules:
  - from:
    - source:
        # Разрешаем доступ ТОЛЬКО если запрос пришел от ServiceAccount 'bookinfo-reviews'
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-reviews"]
    to:
    - operation:
        methods: ["GET"] # Разрешаем только GET запросы
```

Применяем:
```bash
kubectl apply -f allow-reviews-only.yaml
```

**Проверка:**
1.  Откройте сайт. Всё должно работать как обычно (так как `reviews` имеет право ходить в `ratings`).
2.  Теперь давайте притворимся хакером, который захватил один из сервисов и хочет украсть оценки напрямую.
```bash
kubectl run curl-test --image=curlimages/curl -n bookinfo --restart=Never --command -- sleep 3600
```
3. Подождите пару секунд, пока он запустится:
```bash
kubectl wait --for=condition=ready pod/curl-test -n bookinfo
```
4. Сделайте запрос из этого пода
```bash
kubectl exec curl-test -n bookinfo -- curl -sS -o /dev/null -w "HTTP Code: %{http_code}\n" http://ratings:9080/ratings/1
```

### Что вы должны увидеть?

**Вариант А: Вы получите код 403 Forbidden**

```text
HTTP Code: 403
```

**Вариант Б: Вы получите код 200 OK**

Это тоже успех с точки зрения mTLS, но значит, что вы забыли применить или удалили политику авторизации (файл `allow-reviews-only.yaml`).