## Модуль 6: Управление трафиком (Traffic Management)

До этого момента мы плыли по течению: Kubernetes сам решал, на какой под отправить запрос (стандартный Round Robin). Но в микросервисном мире этого мало. Нам нужно уметь перенаправлять пользователей, тестировать гипотезы и безопасно выкатывать обновления.

Istio разделяет управление трафиком на две концепции, которые часто путают новички. Давайте разберем их на аналогии с такси:

1.  **DestinationRule (Правило назначения):** Это **список доступных машин в таксопарке**, сгруппированных по классам (Эконом, Комфорт, Бизнес). Мы просто описываем, *какие бывают версии* (subsets).
2.  **VirtualService (Виртуальный сервис):** Это **навигатор**, который решает, какую машину подать клиенту. "Клиент Вася хочет машину, отправь ему 'Бизнес'".

### 6.1 DestinationRule: Определение версий
Прежде чем направлять трафик, Istio должен знать, чем отличаются ваши поды друг от друга. Мы используем Kubernetes Labels (`version: v1`) для создания именованных групп (subsets).

Создайте (или убедитесь, что применили) правила для всех сервисов Bookinfo:

```bash
kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/destination-rule-all.yaml
```

**Давайте заглянем "под капот" одного из правил:**
Вот как выглядит конфиг для сервиса `reviews` (выдержка):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews   # Применяется к сервису 'reviews'
  subsets:
  - name: v1      # Имя подгруппы для Istio
    labels:
      version: v1 # Искать поды в K8s с этим лейблом
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```
*Суть:* Теперь Istio знает, что у сервиса `reviews` есть три подмножества: `v1`, `v2`, `v3`.

### 6.2 Сценарий 1: Жесткая фиксация версии (Blue/Green)
Представьте, что v2 и v3 работают нестабильно. Мы хотим принудительно завернуть 100% пользователей на старую добрую v1 (без звезд).

Создайте файл `reviews-v1.yaml`:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews # Перехватываем запросы к сервису reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1 # Отправляем ВСЕХ сюда
```

Применяем:
```bash
kubectl apply -f reviews-v1.yaml
```

**Как проверить:**
1.  Откройте [http://localhost:8080/productpage](http://localhost:8080/productpage).
2.  Обновите страницу 10 раз. Вы увидите, что **звезды исчезли навсегда**. Ни черных, ни красных.
3.  **В Kiali:** Зайдите в Graph, выберите `Display -> Traffic Animation`. Вы увидите, что поток от `productpage` идет исключительно к `reviews-v1`. Узлы v2 и v3 станут серыми (неактивными).

### 6.3 Сценарий 2: Canary Deployment (Канареечный релиз)
Мы починили v2 (черные звезды) и хотим начать плавную выкатку. Рискованно переключать всех сразу. Давайте направим 90% людей на старую версию, а 10% — на новую.

Создайте файл `reviews-90-10.yaml`:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90 # 90% запросов (важно: сумма весов должна быть 100)
    - destination:
        host: reviews
        subset: v2
      weight: 10 # 10% запросов
```

Применяем:
```bash
kubectl apply -f reviews-90-10.yaml
```

**Как проверить:**
1.  Вернитесь в браузер и начните яростно обновлять страницу (F5).
2.  Примерно 9 раз из 10 звезд не будет.
3.  1 раз из 10 вы увидите черные звезды.
4.  **В Kiali:** В меню `Display` включите галочку **Traffic Distribution**. На линиях графа появятся проценты. Вы увидите реальное распределение (оно может колебаться, например, 88% / 12%, это нормально для малых выборок).

### 6.4 Сценарий 3: А/B Тестирование (Умная маршрутизация)
Маркетинг хочет показать красные звезды (v3) только VIP-пользователям. В нашем случае VIP — это пользователь `jason`.
Istio умеет читать HTTP-заголовки и принимать решения на уровне L7.

Создайте файл `reviews-jason.yaml`:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  # Правило 1: Проверяем, Джейсон ли это?
  - match:
    - headers:
        end-user:      # Кастомный заголовок, который приложение Bookinfo пробрасывает само
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v3     # Джейсону — красные звезды (v3)
  
  # Правило 2: Дефолтное (для всех остальных)
  - route:
    - destination:
        host: reviews
        subset: v1     # Остальным — ничего (v1)
```

Применяем:
```bash
kubectl apply -f reviews-jason.yaml
```

**Как проверить:**
1.  Откройте сайт. Звезд нет (вы аноним, попадаете в правило 2).
2.  Нажмите **Sign in** (справа сверху).
3.  Введите логин `jason` (пароль любой).
4.  Вуаля! Красные звезды.
5.  Попробуйте зайти под логином `admin` — звезд снова нет (так как имя не jason).

**Резюме модуля:**
Вы только что реализовали сложные сценарии деплоя (Canary, A/B), не изменив ни строчки кода в самих микросервисах `reviews`. Вся логика вынесена в инфраструктуру.