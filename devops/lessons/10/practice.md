Отлично! У нас есть автоматизированный конвейер доставки с инфраструктурой как кодом, базами данных и оркестрацией контейнеров. Теперь пришло время убедиться, что мы знаем, что происходит с нашей системой после деплоя. Мониторинг, логирование и алертинг - это глаза и уши DevOps инженера.

---

**УРОК 10: Логирование и Мониторинг Ошибок (Observability Stack)**

**Цель:** Понять важность наблюдаемости (Observability), освоить инструменты для сбора, хранения, анализа и визуализации логов и метрик, а также настроить систему оповещений о проблемах.

**Результат урока:** У вас будет развернут стек для логирования (Promtail, Loki) и мониторинга (Prometheus, Grafana, Alertmanager). Ваше приложение в Kubernetes будет настроено для предоставления метрик, и вы сможете просматривать логи и метрики, а также получать уведомления о возникших проблемах.

**Предварительные требования:**

*   Успешно завершен Урок 1-9.
*   Работающий Kubernetes кластер (Minikube/Kind, Lab 1 Урока 9) с задеплоенным приложением (через Helm/Argo CD, Урок 9).
*   Дополнительные ресурсы в кластере или на отдельной VM/VMs для развертывания стека наблюдаемости.

**Техническая подготовка:**

1.  **Выберите место для стека наблюдаемости:**
    *   **В Kubernetes кластере (рекомендуется):** Развернуть все компоненты (Promtail, Loki, Prometheus, Grafana, Alertmanager) непосредственно в вашем Minikube/Kind кластере, используя Helm Chart'ы. Это самый "клауд-нативный" подход. Потребуется выделить достаточно ресурсов для кластера (это может сильно увеличить потребление RAM и CPU).
    *   **На отдельных VM:** Развернуть Prometheus, Grafana, Loki, Alertmanager на одной или нескольких VM. Promtail (лог-агент) и Node Exporter (для метрик VM) будут устанавливаться на Staging VM и Worker ноды K8s.

2.  **Установите Helm:** Если вы еще не устанавливали Helm (например, для деплоя приложения в Уроке 9), установите его (см. Урок 9, Техническая подготовка). Мы будем использовать Helm для развертывания стека наблюдаемости в K8s.

3.  **(Если разворачиваете в K8s) Изучите Helm Chart'ы для стека:** Популярные Helm Chart'ы для стека наблюдаемости:
    *   [Prometheus Community Helm Charts](https://prometheus-community.github.io/helm-charts) (содержит charts для Prometheus, Alertmanager, Node Exporter, Kube-state-metrics и др.)
    *   [Grafana Labs Helm Charts](https://grafana.github.io/helm-charts/) (содержит charts для Grafana, Loki, Promtail)

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **Наблюдаемость (Observability):** Способность системы предоставлять информацию о своем внутреннем состоянии, чтобы пользователи могли задавать вопросы о том, что происходит, без необходимости заранее знать, что искать. Достигается за счет сбора и анализа трех типов данных: Logs, Metrics, Traces.
*   **Логирование (Logging):** Запись событий, происходящих в системе. unstructured или semi-structured текст. Полезны для отладки конкретных инцидентов, понимания последовательности событий. Проблемы: большой объем, сложность поиска, отсутствие агрегации по умолчанию.
*   **Метрики (Metrics):** Числовые значения, агрегированные по времени. Полезны для понимания общего состояния системы, трендов, выявления аномалий. Типы: Counter (только увеличивается), Gauge (произвольное значение), Histogram/Summary (распределение значений).
*   **Трассировка (Tracing):** Отслеживание выполнения одного запроса или операции через все компоненты распределенной системы. Полезно для анализа производительности, выявления узких мест в микросервисных архитектурах. (Обзорно в этом уроке).

*   **Централизованное логирование:** Сбор логов со всех компонентов системы (приложений, серверов, K8s) и отправка их в централизованное хранилище для индексации, поиска и анализа.
    *   **ELK Stack:** Elasticsearch (хранилище и поиск), Logstash (сбор и обработка), Kibana (визуализация). Классический, но ресурсоемкий стек.
    *   **PLG Stack:** Promtail (сбор логов с таргетов и отправка в Loki), Loki (хранилище логов с индексацией по меткам), Grafana (визуализация и анализ логов с помощью LogQL). Легковеснее ELK, использует метки вместо полного текстового индекса.

*   **Loki:** Система агрегации логов от Grafana Labs. Индексирует логи только по набору *меток* (например, `job="myapp"`, `namespace="default"`, `pod="myapp-..."`). Это делает его быстрее и дешевле, чем системы с полным текстовым индексом, но поиск по тексту внутри лога без меток может быть медленнее.
    *   **LogQL:** Язык запросов Loki.

*   **Мониторинг:** Процесс сбора, обработки, агрегации и анализа метрик для оценки производительности и доступности системы.
*   **Prometheus:** Популярная система мониторинга временных рядов (time-series data). Использует модель `pull` (скрапит метрики с экспортеров по расписанию).
    *   **Prometheus Server:** Собирает и хранит метрики.
    *   **Exporters:** Агенты, которые собирают метрики с определенного сервиса (Node Exporter для хоста, JMX Exporter для Java приложений, exporters для баз данных, веб-серверов и т.д.) и предоставляют их в формате, понятном Prometheus.
    *   **Pushgateway:** Компонент для кратковременных задач, которые не могут быть скраплены (например, скрипты).
    *   **Service Discovery:** Механизмы для автоматического обнаружения таргетов для скрапинга (например, интеграция с Kubernetes API).
    *   **PromQL:** Язык запросов Prometheus.

*   **Grafana:** Популярная платформа для визуализации данных. Поддерживает множество источников данных (Prometheus, Loki, Elasticsearch, БД). Позволяет создавать дашборды с графиками, таблицами, алертами.

*   **Ключевые метрики (RED, USE методы):**
    *   **RED (для сервисов):** Rate (частота запросов), Errors (частота ошибок), Duration (время ответа).
    *   **USE (для ресурсов: CPU, Memory, Disk, Network):** Utilization (загрузка), Saturation (насыщенность/очереди), Errors (ошибки).

*   **SLA, SLO, SLI:**
    *   **SLI (Service Level Indicator):** Метрика, измеряющая аспект предоставляемого сервиса (например, процент успешных HTTP запросов, среднее время ответа).
    *   **SLO (Service Level Objective):** Целевое значение для SLI за определенный период (например, 99% запросов должны быть успешными за месяц).
    *   **SLA (Service Level Agreement):** Официальный договор с клиентом, включающий SLO и последствия их невыполнения.

*   **Алертинг (Alerting):** Уведомление о наступлении определенного условия (например, метрика превысила порог, Pod упал).
    *   **Alertmanager (для Prometheus):** Получает алерты от Prometheus, группирует их, подавляет дублирующиеся, маршрутизирует уведомления в различные системы (Email, Slack, PagerDuty).
    *   **Принцип "Alert on symptoms, not causes":** Лучше получать алерты о том, что *пользователи ощущают* (симптомы, например, высокая задержка, ошибки), чем о внутренних причинах (например, высокая загрузка CPU - это может быть нормально).

*   **C.A.L.M.S. Framework (DevOps):** Повторение пройденного: Culture (сотрудничество), Automation (автоматизация всего), Lean (бережливое производство, Value Stream), Measurement (измерения - мониторинг, логирование, метрики), Sharing (обмен знаниями). Мониторинг и логирование напрямую относятся к Measurement.

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Развертывание PLG Stack (Promtail, Loki, Grafana) в K8s**

*   **Цель:** Установить централизованную систему сбора и хранения логов в вашем кластере.
*   **Предварительные требования:**
    *   Работающий Kubernetes кластер (Lab 1 Урока 9).
    *   Установленный Helm (Техническая подготовка Урока 9).
*   **Шаги:**

    1.  **Добавьте репозиторий Helm Chart'ов Grafana Labs:**
        ```bash
        helm repo add grafana https://grafana.github.io/helm-charts
        helm repo update
        ```

    2.  **Создайте неймспейс для стека наблюдаемости (опционально):** Хорошая практика - разворачивать компоненты инфраструктуры в отдельном неймспейсе.
        ```bash
        kubectl create namespace monitoring
        ```

    3.  **Разверните Loki и Promtail с помощью Helm:** Chart `loki` включает в себя Loki (хранилище логов) и Promtail (агент сбора логов), который будет установлен на каждой ноде как DaemonSet.
        ```bash
        helm install loki grafana/loki-stack --namespace monitoring --set promtail.enabled=true --set prometheus.enabled=false --set grafana.enabled=false
        ```
        *   `loki`: имя релиза Helm.
        *   `grafana/loki-stack`: имя Chart'а.
        *   `--namespace monitoring`: установить в неймспейс `monitoring`.
        *   `--set promtail.enabled=true`: явно включить установку Promtail (по умолчанию в этом чарте).
        *   `--set prometheus.enabled=false --set grafana.enabled=false`: отключить установку Prometheus и Grafana из этого чарта (мы установим их отдельно или позже).

    4.  **Проверьте статус развернутых компонентов:**
        ```bash
        kubectl get pods -n monitoring
        kubectl get daemonsets -n monitoring # Должен появиться DaemonSet promtail
        ```
        Подождите, пока все Pods будут в статусе `Running`. Promtail должен запуститься на каждой Worker ноде.

**Результат Lab 1:** Установлены Loki и Promtail в вашем кластере Kubernetes. Promtail уже начал собирать логи со всех Pods и отправлять их в Loki.

---

**Lab 2: Развертывание Prometheus, Grafana и Alertmanager в K8s**

*   **Цель:** Установить компоненты для сбора метрик, визуализации и алертинга в вашем кластере.
*   **Предварительные требования:**
    *   Работающий Kubernetes кластер (Lab 1 Урока 9).
    *   Установленный Helm (Техническая подготовка Урока 9).
    *   Неймспейс `monitoring` (Lab 1).
*   **Шаги:**

    1.  **Добавьте репозиторий Helm Chart'ов Prometheus Community:**
        ```bash
        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
        helm repo update
        ```

    2.  **Разверните Prometheus с помощью Helm:** Chart `kube-prometheus-stack` - это комплексное решение, включающее Prometheus Server, Alertmanager, Grafana, Kube-state-metrics (метрики K8s объектов), Node Exporter (метрики нод) и Prometheus Operator. Это удобнее, чем устанавливать все по отдельности.

        ```bash
        # Нам уже установлен Promtail, Loki, Grafana из чарта loki-stack.
        # Вместо kube-prometheus-stack, который дублирует многое, установим компоненты по отдельности.

        # Развернуть Prometheus Server, Kube-state-metrics, Node Exporter
        # Chart prometheus-community/prometheus - это сам сервер и экспортеры
        helm install prometheus prometheus-community/prometheus --namespace monitoring --set alertmanager.enabled=false --set grafana.enabled=false
        ```
        *   `prometheus`: имя релиза.
        *   `prometheus-community/prometheus`: имя Chart'а.
        *   `--set alertmanager.enabled=false --set grafana.enabled=false`: отключаем Alertmanager и Grafana из этого Chart'а, т.к. мы их установим отдельно (или уже установили Grafana из `loki-stack`, хотя лучше использовать одну установку Grafana).

    3.  **Разверните Grafana с помощью Helm:** Используем Chart от Grafana Labs для установки Grafana.

        ```bash
        helm install grafana grafana/grafana --namespace monitoring --set persistence.enabled=true --set persistence.size=5Gi --set adminPassword='your_grafana_admin_password' --set service.type=NodePort # Укажите свои настройки
        ```
        *   `grafana`: имя релиза.
        *   `grafana/grafana`: имя Chart'а.
        *   `--set persistence.enabled=true --set persistence.size=5Gi`: включить персистентное хранилище для данных Grafana (графики, дашборды), запросить 5GB. Потребуется, чтобы ваш StorageClass (Урок 9, Lab 6) поддерживал динамическое Provisioning.
        *   `--set adminPassword='your_grafana_admin_password'`: установить пароль администратора. **Используйте Kubernetes Secret для пароля в production!**
        *   `--set service.type=NodePort`: сделать Grafana доступной извне по NodePort (для локальной Lab).

    4.  **Разверните Alertmanager с помощью Helm:**

        ```bash
        helm install alertmanager prometheus-community/alertmanager --namespace monitoring --set service.type=NodePort # Сделать доступным извне для проверки
        ```
        *   `alertmanager`: имя релиза.
        *   `prometheus-community/alertmanager`: имя Chart'а.
        *   `--set service.type=NodePort`: сделать доступным извне (для Lab).

    5.  **Проверьте статус развернутых компонентов:**
        ```bash
        kubectl get pods -n monitoring
        kubectl get deployments -n monitoring # prometheus-server, grafana, alertmanager
        kubectl get daemonsets -n monitoring # node-exporter
        kubectl get statefulsets -n monitoring # может быть для prometheus/alertmanager HA
        kubectl get services -n monitoring # prometheus-server, grafana, alertmanager
        ```
        Подождите, пока все Pods будут в статусе `Running`.

    6.  **Получите доступ к веб-интерфейсам Grafana и Alertmanager:**
        *   Найдите сервисы (`kubectl get services -n monitoring`).
        *   Для сервисов типа NodePort, найдите их NodePort (например, 3xxxx).
        *   Найдите IP ноды K8s (`minikube ip` или IP Docker контейнера).
        *   Grafana UI: `http://IP_НОДЫ:NODEPORT_GRAFANA`. Войдите с `admin` и паролем, который вы установили.
        *   Alertmanager UI: `http://IP_НОДЫ:NODEPORT_ALERTMANAGER`.

**Результат Lab 2:** Развернуты Prometheus Server, Grafana и Alertmanager в вашем кластере. У вас есть доступ к их веб-интерфейсам.

---

**Lab 3: Интеграция Логирования и Анализ Логов в Grafana**

*   **Цель:** Настроить Grafana для использования Loki как источника данных и визуализировать логи вашего приложения.
*   **Предварительные требования:**
    *   Работающие Loki, Promtail и Grafana (Lab 1 и 2).
    *   Работающее приложение в K8s (Урок 9), которое пишет логи в stdout/stderr контейнера.
*   **Шаги:**

    1.  **Настройте Loki как Data Source в Grafana:**
        *   Войдите в Grafana UI.
        *   В левом меню "Connections" -> "Data sources".
        *   Нажмите "Add data source".
        *   Выберите "Loki".
        *   Name: `Loki` (или любое имя).
        *   HTTP -> URL: Укажите URL Loki Query Frontend или Loki Service в вашем кластере. Если Loki установлен в том же кластере и неймспейсе `monitoring`, URL будет `http://loki.monitoring.svc.cluster.local:3100` (имя сервиса Loki по умолчанию из helm chart'а `loki-stack` + неймспейс + суффикс кластера + порт). Если вы пробрасывали порт Loki на хост, используйте `http://localhost:ПОРТ`.
        *   Auth: Оставьте по умолчанию (если нет аутентификации).
        *   Нажмите "Save & test". Должно появиться сообщение "Data source is working".

    2.  **Исследуйте логи вашего приложения с помощью Explore в Grafana:**
        *   В левом меню "Explore" (иконка с компасом).
        *   Выберите ваш Data Source "Loki".
        *   Откройте вкладку "Logs".
        *   Используйте LogQL для поиска логов. Promtail автоматически добавляет метки Pod'ам, с которых собирает логи. Основные метки: `namespace`, `pod`, `container`.
        *   Найдите логи вашего приложения:
            ```LogQL
            {namespace="default", app="my-java-app"} # Если в Deployment есть метка app: my-java-app
            # Или по имени пода:
            # {namespace="default", pod="<начало_имени_пода_вашего_приложения>"}
            ```
        *   Нажмите "Run query". Вы должны увидеть список логов вашего приложения.
        *   Попробуйте фильтровать по тексту:
            ```LogQL
            {namespace="default", app="my-java-app"} |= "error" # Найти логи с ошибками
            ```

    3.  **Создайте простой дашборд для логов:**
        *   В левом меню "Dashboards" -> "New Dashboard".
        *   Нажмите "Add visualization".
        *   Выберите ваш Data Source "Loki".
        *   В Query Builder используйте LogQL запрос для ваших логов (как в шаге 2).
        *   Выберите тип визуализации (например, "Logs").
        *   Сохраните панель на дашборде.

**Результат Lab 3:** Grafana настроена для работы с Loki. Вы можете просматривать, фильтровать и анализировать логи всех Pods в кластере, включая логи вашего приложения, с помощью LogQL и в интерфейсе Grafana.

---

**Lab 4: Интеграция Мониторинга (Prometheus Exporter) в Приложение**

*   **Цель:** Настроить ваше Java приложение на предоставление метрик в формате, понятном Prometheus.
*   **Предварительные требования:**
    *   Работающее приложение в K8s.
    *   Prometheus Server запущен в K8s.
*   **Шаги:**

    1.  **Добавьте зависимости Prometheus Exporter в ваш Maven проект:** Если вы используете Spring Boot, Spring Boot Actuator и Micrometer значительно упрощают этот процесс.
        *   Добавьте Spring Boot Actuator и Micrometer Prometheus Registry в `pom.xml`:
            ```xml
            <dependencies>
                <!-- ... другие зависимости ... -->

                <!-- Spring Boot Actuator -->
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-actuator</artifactId>
                </dependency>

                <!-- Micrometer Registry for Prometheus -->
                <dependency>
                    <groupId>io.micrometer</groupId>
                    <artifactId>micrometer-registry-prometheus</artifactId>
                </dependency>

                <!-- ... другие зависимости ... -->
            </dependencies>
            ```

    2.  **Настройте приложение для предоставления метрик:** В Spring Boot Actuator метрики Prometheus доступны по умолчанию на endpoint'е `/actuator/prometheus`. Убедитесь, что этот endpoint включен.
        *   В `application.properties` (или другом файле конфигурации) вашего приложения:
            ```properties
            management.endpoints.web.exposure.include=health,prometheus # Включить health check и prometheus endpoints
            management.endpoint.health.show-details=always # Показать детали health check (опционально)
            ```
        *   *Важно:* Убедитесь, что порт Actuator endpoint'а доступен *внутри контейнера*. По умолчанию это тот же порт, что и у приложения (8080).

    3.  **Обновите Dockerfile (опционально):** Убедитесь, что порт Actuator (если он отличается от основного порта приложения) тоже EXPOSEd в Dockerfile.

    4.  **Обновите Helm Chart вашего приложения (`my-app-chart`):**
        *   **Deployment Template:** Убедитесь, что порты контейнера объявлены (включая порт Actuator, если он другой).
        *   **Service Template:** Убедитесь, что Service направляет трафик на нужный порт контейнера (обычно на основной порт приложения). Prometheus будет скрапить метрики напрямую с Pod'ов по другому порту, если нужно.
        *   **Добавьте аннотации Kubernetes для Service Discovery Prometheus:** Prometheus может автоматически обнаруживать цели для скрапинга по аннотациям. Аннотации добавляются к Pod'ам или Service'ам. Проще добавить аннотации к Pod Template в Deployment.

            ```yaml
            # my-app-chart/templates/myapp_deployment.yaml (обновленный)
            apiVersion: apps/v1
            kind: Deployment
            # ...
            spec:
              template:
                metadata:
                  labels:
                    app: my-java-app
                    tier: backend
                  annotations: # АННОТАЦИИ для Service Discovery Prometheus
                    prometheus.io/scrape: 'true' # Включить скрапинг
                    prometheus.io/path: '/actuator/prometheus' # Путь к endpoint'у метрик
                    prometheus.io/port: '8080' # Порт endpoint'а метрик (порт контейнера)
                spec:
                  containers:
                  - name: my-app-container
                    # ...
                    ports:
                    - containerPort: 8080
                      name: http
                    # ...
            ```
        *   **Обновите `values.yaml` Chart'а:** Добавьте переменные для аннотаций, если хотите сделать их настраиваемыми.

    5.  **Пересоберите Docker образ приложения (если меняли код/зависимости):** Сделайте коммит, запушьте. CI пайплайн должен пересобрать образ и опубликовать его.
    6.  **Обновите Helm Chart в Git репозитории K8s манифестов:** Добавьте изменения в `pom.xml` и `Dockerfile` (если есть), обновите манифесты и `values.yaml` Helm Chart'а (Lab 7 Урока 9).
    7.  **Обновите релиз Helm в K8s через GitOps (Argo CD):** Сделайте коммит и пуш в репозиторий K8s манифестов. Argo CD должен автоматически обновить Deployment в K8s. Дождитесь, пока новые Pods запустятся.

    8.  **Настройте Prometheus для скрапинга метрик вашего приложения:** Prometheus, развернутый Chart'ом, должен быть настроен на автоматическое обнаружение Pod'ов с аннотациями `prometheus.io/scrape: 'true'`. Это стандартная конфигурация для Prometheus, работающего в K8s. Вам не нужно вручную добавлять цели скрапинга.

    9.  **Проверьте, что Prometheus скрапит метрики:**
        *   Получите доступ к веб-интерфейсу Prometheus. Найдите его Service (`kubectl get services -n monitoring`) и пробросьте порт или используйте NodePort.
        *   В UI Prometheus перейдите в "Status" -> "Targets".
        *   Найдите ваши Pods приложения (по IP или имени). Убедитесь, что статус `State` для endpoint'а `/actuator/prometheus` на порту 8080 - `UP`.
        *   В меню "Graph" или "Explore" введите метрику, которую предоставляет ваше приложение (например, `jvm_memory_used_bytes` или `http_server_requests_seconds_count` если используете Spring Boot). Вы должны увидеть график.

**Результат Lab 4:** Ваше приложение предоставляет метрики, и Prometheus автоматически их скрапит из Kubernetes кластера.

---

**Lab 5: Визуализация Метрик в Grafana**

*   **Цель:** Настроить Grafana для использования Prometheus как источника данных и создать дашборд для метрик вашего приложения.
*   **Предварительные требования:**
    *   Работающие Prometheus и Grafana (Lab 2).
    *   Prometheus успешно скрапит метрики вашего приложения (Lab 4).
*   **Шаги:**

    1.  **Настройте Prometheus как Data Source в Grafana:**
        *   Войдите в Grafana UI.
        *   "Connections" -> "Data sources" -> "Add data source".
        *   Выберите "Prometheus".
        *   Name: `Prometheus` (или любое имя).
        *   HTTP -> URL: Укажите URL Prometheus Server в вашем кластере. Если Prometheus установлен в том же кластере и неймспейсе `monitoring`, URL будет `http://prometheus-server.monitoring.svc.cluster.local:80` (или 9090, в зависимости от Service). Если вы пробрасывали порт Prometheus на хост, используйте `http://localhost:ПОРТ`.
        *   Нажмите "Save & test". Должно появиться сообщение "Data source is working".

    2.  **Создайте дашборд для метрик приложения:**
        *   "Dashboards" -> "New Dashboard".
        *   Нажмите "Add visualization".
        *   Выберите ваш Data Source "Prometheus".
        *   Используйте PromQL для запроса метрик.
            *   Пример: График количества запросов в секунду (для Spring Boot):
                ```PromQL
                rate(http_server_requests_seconds_count{job="kubernetes-pods", namespace="default", app="my-java-app"}[5m])
                ```
                (Убедитесь, что метки `job`, `namespace`, `app` соответствуют вашим).
            *   Пример: Среднее время ответа (для Spring Boot):
                ```PromQL
                rate(http_server_requests_seconds_sum{job="kubernetes-pods", namespace="default", app="my-java-app"}[5m]) / rate(http_server_requests_seconds_count{job="kubernetes-pods", namespace="default", app="my-java-app"}[5m])
                ```
            *   Пример: Загрузка CPU контейнеров:
                ```PromQL
                sum(rate(container_cpu_usage_seconds_total{image!="", container!="POD", pod=~"my-java-app-deployment-.*", namespace="default"}[5m])) by (pod)
                ```
        *   Выберите тип визуализации (например, "Graph" или "Stat").
        *   Настройте внешний вид (названия осей, легенда и т.д.).
        *   Повторите, чтобы добавить несколько панелей на дашборд (например, количество запросов, время ответа, ошибки, использование CPU/RAM).
        *   Сохраните дашборд.

    3.  **(Опционально) Импортируйте готовый дашборд:** Grafana Sharing (grafana.com/grafana/dashboards) содержит множество готовых дашбордов для стандартных технологий (JVM, Spring Boot, Node Exporter, Kubernetes Cluster). Найдите подходящий, скопируйте его ID или JSON и импортируйте в свою Grafana ("Dashboards" -> "Import").

**Результат Lab 5:** Grafana настроена для работы с Prometheus. Вы можете визуализировать метрики вашего приложения и инфраструктуры на дашбордах, получая представление о состоянии системы.

---

**Lab 6: Настройка Базового Алертинга с Prometheus и Alertmanager**

*   **Цель:** Создать правила алертинга в Prometheus и настроить Alertmanager для отправки уведомлений.
*   **Предварительные требования:**
    *   Работающие Prometheus и Alertmanager (Lab 2).
    *   Метрики вашего приложения скрапятся Prometheus (Lab 4).
    *   Доступ к конфигурации Prometheus и Alertmanager.
*   **Шаги:**

    1.  **Добавьте правила алертинга в конфигурацию Prometheus:**
        *   Если вы разворачивали Prometheus с помощью Helm Chart'а `prometheus-community/prometheus`, конфигурация обычно находится в ConfigMap. Найдите его: `kubectl get configmaps -n monitoring | grep prometheus-server`. Отредактируйте этот ConfigMap (`kubectl edit configmap <имя_configmap> -n monitoring`).
        *   В секции `data` -> `alerting_rules.yml` (или похожем ключе) добавьте правила алертинга в формате Prometheus.

            ```yaml
            # Пример правил алертинга (внутри alertmanager_rules.yml в ConfigMap Prometheus)

            groups:
            - name: my-app-alerts # Имя группы правил
              rules:
              - alert: HighRequestLatency # Имя алерта
                expr: | # PromQL запрос, который возвращает 1, если условие выполняется, и 0, если нет
                  (
                    sum(rate(http_server_requests_seconds_sum{job="kubernetes-pods", namespace="default", app="my-java-app", outcome="SUCCESS"}[5m]))
                    /
                    sum(rate(http_server_requests_seconds_count{job="kubernetes-pods", namespace="default", app="my-java-app", outcome="SUCCESS"}[5m]))
                  ) > 0.5 # Среднее время ответа > 0.5 секунд за последние 5 минут
                for: 2m # Ждать 2 минуты, прежде чем отправить алерт (чтобы избежать ложных срабатываний)
                labels: # Метки для алерта
                  severity: warning
                  tier: backend
                annotations: # Аннотации для алерта (информация для человека)
                  summary: "Высокая задержка запросов к приложению {{ \$labels.app }} в неймспейсе {{ \$labels.namespace }}"
                  description: "Среднее время ответа для успешных запросов > 0.5s за последние 2 минуты."

              - alert: HighErrorRate # Имя алерта
                expr: |
                  sum(rate(http_server_requests_seconds_count{job="kubernetes-pods", namespace="default", app="my-java-app", outcome="CLIENT_ERROR"}[5m])) by (app, namespace) / sum(rate(http_server_requests_seconds_count{job="kubernetes-pods", namespace="default", app="my-java-app"}[5m])) by (app, namespace)
                  > 0.1 # Более 10% запросов завершаются с ошибкой клиента за последние 5 минут
                for: 1m
                labels:
                  severity: critical
                  tier: backend
                annotations:
                  summary: "Высокая частота ошибок клиента (4xx) в приложении {{ \$labels.app }} в неймспейсе {{ \$labels.namespace }}"
                  description: "Более 10% запросов завершились с ошибкой клиента за последнюю минуту."

              - alert: AppPodDown # Алерт при падении пода приложения
                expr: kube_deployment_status_replicas_available{deployment="my-java-app-deployment", namespace="default"} < 1 # Если доступно меньше 1 реплики
                for: 1m
                labels:
                  severity: critical
                  tier: backend
                annotations:
                  summary: "Под приложения {{ \$labels.deployment }} недоступен в неймспейсе {{ \$labels.namespace }}"
                  description: "Количество доступных реплик {{ \$value }} меньше 1."
            ```
        *   Сохраните ConfigMap. Prometheus Server должен автоматически подхватить изменения (проверьте его логи, если нет).

    2.  **Настройте Alertmanager для получения алертов от Prometheus:** Prometheus по умолчанию отправляет алерты на адрес Alertmanager. Убедитесь, что в конфигурации Prometheus указан правильный адрес Alertmanager Service в K8s кластере (например, `alertmanager.monitoring.svc.cluster.local:9093`). Этот адрес также настраивается через Helm Chart Prometheus.

    3.  **Настройте Alertmanager для отправки уведомлений:** Конфигурация Alertmanager также находится в ConfigMap. Найдите его: `kubectl get configmaps -n monitoring | grep alertmanager`. Отредактируйте этот ConfigMap (`kubectl edit configmap <имя_configmap> -n monitoring`).
        *   В секции `data` -> `alertmanager.yml` добавьте конфигурацию для отправки уведомлений.

            ```yaml
            # Пример конфигурации alertmanager.yml (внутри ConfigMap Alertmanager)

            global:
              resolve_timeout: 5m

            route: # Корневой маршрут
              group_by: ['alertname', 'cluster', 'service'] # Группировать алерты по этим меткам
              group_wait: 30s # Ждать 30 секунд, прежде чем отправить первое уведомление
              group_interval: 5m # Отправлять повторные уведомления для сгруппированных алертов каждые 5 минут
              repeat_interval: 1h # Отправлять повторные уведомления для разрешенных алертов каждые 1 час
              receiver: 'default-receiver' # Куда отправлять алерты по умолчанию

              routes: # Дополнительные маршруты (например, для разных команд или важности)
              - match:
                  severity: critical # Если алерт имеет метку severity: critical
                receiver: 'critical-receiver' # Отправить на другой приемник
                group_wait: 10s # Ждать меньше

            receivers: # Определения приемников
            - name: 'default-receiver'
              # Пример: отправка в Slack (вам нужно создать Slack Webhook URL)
              # slack_configs:
              # - api_url: 'https://hooks.slack.com/services/...' # Ваш Slack Webhook URL
              #  channel: '#general' # Канал
              #  text: '{{ template "slack.default.text" . }}' # Шаблон сообщения

              # Пример: отправка на Email (вам нужно настроить SMTP сервер)
              # email_configs:
              # - to: 'devops@yourcompany.com'
              #  from: 'alertmanager@yourcompany.com'
              #  smarthost: 'smtp.yourcompany.com:587'
              #  auth_username: 'alertmanager@yourcompany.com'
              #  auth_password: 'your_smtp_password'
              #  require_tls: true

              # Пример: запись в файл (для Lab)
              webhook_configs: # Используем webhook для отправки в файл или mock-сервис
              - url: 'http://localhost:5001/' # URL mock-сервиса или скрипта, который запишет в файл

            - name: 'critical-receiver'
              # Настроить другой канал Slack или email для критических алертов
              # ...

            # Пример mock-сервиса для webhook (не нужно настраивать, просто для понимания URL)
            # Можно использовать маленький скрипт на Python с Flask/HTTP server,
            # который принимает POST запросы и пишет их в файл.

            # На VM или в отдельном контейнере:
            # from flask import Flask, request, json
            # app = Flask(__name__)
            # @app.route('/', methods=['POST'])
            # def webhook():
            #  data = request.json
            #  with open('/tmp/alerts.log', 'a') as f:
            #    f.write(json.dumps(data) + '\n')
            #  return '', 200
            # if __name__ == '__main__':
            #  app.run(host='0.0.0.0', port=5001)
            ```
        *   **Для этой Lab, настройте простой приемник webhook, указывающий на URL, который вы можете контролировать (например, локальный скрипт или mock-сервис), чтобы видеть, что алерты приходят.** Настройка реального Slack или Email требует внешних сервисов.
        *   Сохраните ConfigMap Alertmanager. Alertmanager должен автоматически подхватить изменения.

    4.  **Проверьте алертинг:**
        *   Перейдите в веб-интерфейс Prometheus, меню "Alerts". Вы должны увидеть ваши правила алертинга.
        *   Вызовите ситуацию, которая должна вызвать алерт. Например, остановите Pod вашего приложения (`kubectl delete pod <имя_пода>`) - это должно вызвать алерт `AppPodDown`. Или отправьте много запросов, чтобы увеличить задержку/количество ошибок.
        *   Перейдите в веб-интерфейс Alertmanager. Вы должны увидеть появившийся алерт.
        *   Проверьте, отправилось ли уведомление на настроенный приемник (например, в файл, если вы настроили webhook на локальный скрипт).

**Результат Lab 6:** Вы настроили правила алертинга в Prometheus для отслеживания состояния вашего приложения и его метрик. Alertmanager получает эти алерты и может отправлять уведомления по настроенным каналам.

---

**Lab 7: Анализ CALMS Framework**

*   **Цель:** Подвести итоги курса, проанализировав, как пройденные уроки охватили принципы фреймворка CALMS.
*   **Шаги:**

    1.  **Culture:** Как курс способствовал пониманию важности сотрудничества и общих целей между Dev и Ops? (CI, DevOps-культура, общие пайплайны, GitOps как совместная работа над конфигурацией).
    2.  **Automation:** Какие аспекты жизненного цикла ПО мы автоматизировали? (Сборка, тестирование, анализ кода, миграции БД, создание инфраструктуры, настройка серверов, деплой, обновление - почти все!)
    3.  **Lean:** Как концепции бережливого производства (Value Stream, Flow) были отражены в курсе? (Визуализация пайплайна, ускорение цикла доставки, уменьшение размера изменений, раннее обнаружение проблем - CI, тесты, SonarQube).
    4.  **Measurement:** Какие системы измерения и мониторинга мы настроили? (Логирование с Loki, сбор метрик с Prometheus, визуализация с Grafana, алертинг с Alertmanager). Как эти измерения помогают видеть состояние системы и принимать решения?
    5.  **Sharing:** Как инструменты и практики способствуют обмену знаниями и прозрачности? (Код в Git, Jenkinsfile/`.gitlab-ci.yml` в Git, Dockerfile, K8s манифесты, Helm Charts, Playbooks - все как код и доступно для просмотра/ревью; дашборды мониторинга доступны всем).

    2.  **Составьте краткий список достижений по каждому пункту CALMS на основе вашего опыта прохождения курса.**

**Результат Lab 7:** Вы закрепили понимание фреймворка CALMS, увидев, как каждый его компонент был реализован на практике в ходе курса, и как инструменты и практики DevOps способствуют его достижению.

---

**Резюме Урока 10:**

Вы успешно развернули и настроили комплексный стек для логирования (Promtail, Loki), мониторинга (Prometheus), визуализации (Grafana) и алертинга (Alertmanager) в вашем Kubernetes кластере. Вы научились настраивать приложение для предоставления метрик и использовать эти данные для понимания состояния вашей системы. Настроив алерты, вы гарантировали, что будете уведомлены о критических проблемах. Наконец, вы увидели, как все пройденные темы укладываются в общую картину принципов CALMS.

Стек наблюдаемости является неотъемлемой частью Production-ready приложений и позволяет вам эффективно управлять вашей системой и быстро реагировать на инциденты.

**Что дальше?**

Поздравляю! Вы прошли через очень насыщенный и глубокий курс. Вы охватили все основные области DevOps, от контроля версий и CI до IaC, DBOps, контейнеризации, оркестрации и наблюдаемости.

В финальном этапе вас ждет **Итоговый Проект (Урок 11)**. Это будет возможность объединить все полученные знания и построить полный, end-to-end DevOps конвейер для приложения (возможно, более сложного, чем использовалось в Labs). Это ваша возможность закрепить навыки, заполнить пробелы и создать мощное портфолио, демонстрирующее ваш уровень Middle+/Senior DevOps инженера.

Удачи на финальном проекте! Вы хорошо поработали!