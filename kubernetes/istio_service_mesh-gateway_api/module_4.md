## Модуль 4: Gateway API (Входной трафик)

Это поворотный момент в курсе. Мы настроим доступ к нашему приложению из "внешнего мира" (с вашего ноутбука).

**Немного истории:**
Раньше в Kubernetes был ресурс `Ingress`. Он был слишком простым и не умел делать сложные вещи (канареечные релизы, разделение трафика по весам).
Istio придумал свои ресурсы: `Gateway` + `VirtualService`. Они мощные, но работали только в Istio.
Сейчас индустрия (Google, RedHat и др.) создала новый стандарт — **Kubernetes Gateway API**. Это "убийца" старых Ingress. Он вобрал в себя мощь Istio, но стал нативным для Kubernetes.

**Архитектура Gateway API:**
Вместо одного "жирного" конфига, у нас разделение ответственности:
1.  **Gateway (Инфраструктура):** Создается админом кластера. Это "железная" (или виртуальная) дверь. Она открывает порт (80/443).
2.  **HTTPRoute (Приложение):** Создается разработчиком. Это карта, которая говорит: "Если пришли по пути `/login`, иди в сервис А".

### 4.1 Установка Gateway API CRDs
Kubernetes по умолчанию не знает, что такое `Gateway` или `HTTPRoute`. Нам нужно добавить эти определения (Custom Resource Definitions).

```bash
# Проверяем, есть ли CRD, и если нет — устанавливаем из официального репозитория
kubectl get crd gateways.gateway.networking.k8s.io || \
  kubectl apply -k "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0"
```

### 4.2 Создание Gateway (Дверь)
Мы создадим ресурс `Gateway`.

> **Важная магия Istio:** Как только вы создадите этот ресурс с классом `istio`, Istio **автоматически** развернет под и сервис LoadBalancer специально для этого шлюза. Вам не нужно вручную создавать Deployment для nginx или haproxy.

Файл `gateway.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  gatewayClassName: istio  # Говорим Istio: "Это твой шлюз, управляй им"
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same # Разрешаем подключать маршруты только из этого же namespace
```

### 4.3 Создание HTTPRoute (Карта)
Теперь свяжем входящий трафик с нашим микросервисом `productpage`.
Обратите внимание на `parentRefs` — так маршрут "пристегивается" к конкретному шлюзу.

Файл `httproute.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-route
  namespace: bookinfo
spec:
  parentRefs:
  - name: bookinfo-gateway # Ссылка на Gateway из шага 4.2
  rules:
  - matches:
    # Перечисляем все пути, которые использует наше веб-приложение
    - path:
        type: PathPrefix
        value: /productpage
    - path:
        type: PathPrefix
        value: /static
    - path:
        type: PathPrefix
        value: /login
    - path:
        type: PathPrefix
        value: /logout
    - path:
        type: PathPrefix
        value: /api/v1/products
    backendRefs:
    - name: productpage # Имя Kubernetes Service
      port: 9080        # Порт сервиса
```

Применяем конфигурацию:
```bash
kubectl apply -f gateway.yaml -f httproute.yaml
```

### 4.4 Проверка и доступ (Важно!)

После применения `gateway.yaml` Istio создал новый Service типа `LoadBalancer` в неймспейсе `bookinfo`. Давайте найдем его.

```bash
kubectl get svc -n bookinfo
```
Вы увидите новый сервис с именем `bookinfo-gateway-istio` (или просто `bookinfo-gateway`).
Так как мы используем **Kind** (Kubernetes in Docker), у нас нет настоящего облачного балансировщика, и `EXTERNAL-IP` будет висеть в статусе `<pending>`. Это нормально.

Чтобы попасть на сайт, мы пробросим порт этого сервиса на ваш компьютер.

```bash
# ВНИМАНИЕ: Замените 'bookinfo-gateway' на точное имя сервиса из предыдущей команды, если оно отличается.
# Обычно Istio создает сервис с именем, совпадающим с именем Gateway ресурса.
kubectl port-forward -n bookinfo svc/bookinfo-gateway-istio 8080:80
```
*Держите этот терминал открытым. Команда блокирует ввод, пока туннель работает.*

### 4.5 Тестирование
Откройте браузер по адресу: [http://localhost:8080/productpage](http://localhost:8080/productpage).

**Что вы должны увидеть:**
1.  Страницу "Simple Bookstore App".
2.  Справа колонку "Book Reviews".
3.  **Пообновляйте страницу (F5)** 5-10 раз.
    *   Иногда звезд нет (отвечает `reviews-v1`).
    *   Иногда звезды черные (отвечает `reviews-v2`).
    *   Иногда звезды красные (отвечает `reviews-v3`).

**Что происходит?**
Istio по умолчанию использует балансировку **Round Robin**. Так как мы не написали никаких правил маршрутизации внутри Mesh, трафик равномерно размазывается по всем трем версиям сервиса `reviews`. Это подтверждает, что Mesh работает!