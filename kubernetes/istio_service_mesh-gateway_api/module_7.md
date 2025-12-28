## Модуль 7: Fault Injection (Chaos Engineering)

В распределенных системах сеть ненадежна. Сервисы падают, тормозят и теряют пакеты.
**Chaos Engineering** — это практика намеренного создания сбоев, чтобы проверить, как приложение их переживет.

Istio позволяет внедрять сбои (Fault Injection) прямо в трафик, не убивая поды по-настоящему.

### 7.1 Задача: Эмуляция медленного бэкенда
У сервиса `productpage` есть жесткий тайм-аут ожидания ответа от других сервисов (например, 3 секунды). Что будет, если сервис `reviews` начнет отвечать за 7 секунд?

Мы применим это правило только для пользователя `jason`, чтобы не сломать прод для всех.

Создайте файл `test-delay.yaml`:
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
  - match:
    - headers:
        end-user:
          exact: jason
    fault: # Блок внедрения сбоев
      delay:
        percentage:
          value: 100.0 # Ломаем 100% запросов от Джейсона
        fixedDelay: 7s # Задержка 7 секунд
    route:
    - destination:
        host: reviews
        subset: v1
  - route: # Для остальных всё работает быстро
    - destination:
        host: reviews
        subset: v1
```

Применяем:
```bash
kubectl apply -f test-delay.yaml
```

### 7.2 Наблюдаем за падением
Это нужно увидеть своими глазами.

1.  Откройте [http://localhost:8080/productpage](http://localhost:8080/productpage).
2.  Залогиньтесь как `jason`.
3.  Нажмите F5. Засекайте время.
    *   Браузер "висит" и крутит спиннер загрузки.
    *   Проходит около 6-7 секунд...
    *   Страница загружается, но блок **Book Reviews** содержит сообщение об ошибке: *"Sorry, product reviews are currently unavailable for this book."*

### 7.3 Анализ (Root Cause Analysis)
Почему мы видим ошибку?
1.  `productpage` отправил запрос в `reviews`.
2.  Envoy Proxy перехватил запрос и "придержал" его на 7 секунд (наша инъекция).
3.  Код `productpage` (Python) имеет внутренний тайм-аут (допустим, 3 сек). Он устал ждать и разорвал соединение, выдав ошибку пользователю.

**В Kiali:**
1.  Зайдите в **Graph**.
2.  Нажмите на стрелку между `productpage` и `reviews`.
3.  Справа в панели деталей вы увидите вкладку **Flags**. Там появится флажок `DI,DC` (Delayed via Fault Injection). Istio честно помечает такой трафик.

> **Вывод:** С помощью этого теста мы узнали, что наш фронтенд умеет (хоть и с ошибкой) обрабатывать тормоза бэкенда, не падая целиком с "белым экраном смерти".

**Очистка эксперимента:**
Вернем нормальную работу сервиса, удалив правило задержки:
```bash
kubectl delete -f test-delay.yaml 
# Возвращаем простое правило (без звезд)
kubectl apply -f reviews-v1.yaml
```