## Модуль 9: Istio Ambient Mesh (Будущее без Sidecar)

До 2022 года Istio работал исключительно по модели **Sidecar**: в *каждый* под внедрялся прокси-контейнер. Это надежно, но дорого по ресурсам (память, CPU) и сложно в администрировании (нужно перезапускать приложение, чтобы обновить Istio).

**Istio Ambient Mesh** — это новая архитектура ("Dataplane v2"), которая разделяет обязанности:
1.  **Ztunnel (Zero Trust Tunnel):** Один агент на **весь узел (Node)** Kubernetes. Он занимается шифрованием (mTLS) и аутентификацией (L4). Это замена "черновой работы" сайдкаров.
2.  **Waypoint Proxy:** Полноценный Envoy-прокси, который запускается как **обычный Deployment** (не sidecar). Он нужен только тогда, когда вам требуются функции L7 (парсинг HTTP, сложные маршруты, разделение трафика).

> **Суть:** Вы получаете mTLS бесплатно и автоматически, не внедряя ничего в поды. А тяжелую обработку L7 включаете только по требованию.

### 9.1 Установка Ambient Profile
Для работы Ambient нам нужно обновить конфигурацию Istio в кластере, установив компонент CNI (Container Network Interface) и Ztunnel.

*Примечание: Если вы используете Kind, убедитесь, что ваш Docker имеет доступ к ресурсам, так как Ambient требует специфических настроек сети.*

```bash
# Устанавливаем профиль ambient
# В Kind иногда требуются специфические настройки CNI, но мы попробуем стандартный путь:
istioctl install --set profile=ambient --set "components.cni.enabled=true" -y
```

Проверьте, что запустились демоны `ztunnel` (по одному на каждую ноду) и `istio-cni-node`:
```bash
kubectl get daemonset -n istio-system
```
Вы должны увидеть `ztunnel` и `istio-cni-node`.

### 9.2 Создание пространства Ambient
Давайте развернем приложение `curl` и `httpbin` в новом неймспейсе, чтобы протестировать Ambient режим. Мы НЕ будем использовать метку `istio-injection=enabled`.

```bash
kubectl create ns ambient-demo

# Главная магия: включаем Ambient режим
kubectl label namespace ambient-demo istio.io/dataplane-mode=ambient
```

Развернем тестовые сервисы:
```bash
# Httpbin - сервис, который умеет эхо-отвечать
kubectl apply -n ambient-demo -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml

# Sleep - клиент для проверки (просто curl)
kubectl apply -n ambient-demo -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml
```

### 9.3 Проверка "Без Sidecar"
Подождите запуска подов.
```bash
kubectl get pods -n ambient-demo
```

**Обратите внимание на столбец READY:**
В классическом Istio мы видели `2/2`. Здесь вы увидите **`1/1`**.
```text
NAME                      READY   STATUS    RESTARTS   AGE
httpbin-xxxx-xxxx         1/1     Running   0          40s
sleep-xxxx-xxxx           1/1     Running   0          30s
```
Внутри подов **нет** Envoy Proxy. Но при этом они **уже** находятся в меше! Трафик перехватывается на уровне ядра Linux и направляется в `ztunnel` на ноде.

### 9.4 Добавление L7 функций (Waypoint Proxy)
Сейчас у нас работает mTLS и TCP-метрики (L4). Но если мы хотим использовать `VirtualService` (канареечные релизы, как в Модуле 6), нам нужен L7 процессинг.

В Ambient это делается явным созданием прокси-шлюза для неймспейса (или сервиса).

```bash
# Создаем Waypoint Proxy для всего неймспейса
istioctl waypoint apply -n ambient-demo --enroll-namespace
```

Посмотрите на поды:
```bash
kubectl get pods -n ambient-demo
```
Вы увидите новый под `waypoint-xxxx`. Весь L7 трафик для этого неймспейса теперь проходит через него.

### 9.5 Тестирование маршрутизации
Теперь, когда есть Waypoint, мы можем применить L7 политики. Давайте запретим запросы к `httpbin` по пути `/status/500`.

Создайте `deny-500.yaml`:
```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  namespace: ambient-demo
  name: deny-500
spec:
  targetRef:
    kind: Service
    group: ""
    name: httpbin
  action: DENY
  rules:
  - to:
    - operation:
        methods: ["GET"]
        paths: ["/status/500"]
```

Применяем:
```bash
kubectl apply -f deny-500.yaml
```

Проверяем:
```bash
# Обычный запрос (должен работать)
kubectl exec deploy/sleep -n ambient-demo -- curl -s -o /dev/null -w "%{http_code}\n" http://httpbin:8000/status/200
# Ожидаем: 200

# Запрещенный запрос
kubectl exec deploy/sleep -n ambient-demo -- curl -s -o /dev/null -w "%{http_code}\n" http://httpbin:8000/status/500
# Ожидаем: 403 (Forbidden)
```

**Вывод:** Мы получили весь функционал Istio (безопасность и управление трафиком), но наши приложения остались чистыми, без лишних контейнеров внутри.

> **P.S.** Не забудьте снести кластер Kind: `kind delete cluster -n istio-lab`.