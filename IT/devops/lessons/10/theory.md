*   **Урок 10: Логирование и Мониторинг Ошибок**
    *   **Цель:** Настроить сбор и анализ логов, метрик, визуализацию состояния системы и оповещения о проблемах.
    *   **Ключевые Темы:**
        *   Важность логирования, мониторинга и алертинга.
        *   Типы данных для мониторинга: Logs, Metrics, Traces (обзор).
        *   Логирование: Сбор логов (агенты), централизованное хранение и анализ. ELK Stack (Elasticsearch, Logstash, Kibana) - обзор, PLG Stack (Promtail, Loki, Grafana) - фокус.
        *   Loki: Архитектура, Label-based indexing, LogQL.
        *   Мониторинг: Типы метрик (счетчики, таймеры, гистограммы, gauges), сбор метрик (pull vs push).
        *   Prometheus: Архитектура (Server, Exporters, Pushgateway, Alertmanager), Service Discovery, PromQL.
        *   Grafana: Визуализация данных из Prometheus и Loki, создание дашбордов.
        *   Ключевые метрики для приложений и инфраструктуры (RED, USE методы).
        *   SLA, SLO, SLI: Определение и мониторинг.
        *   Алертинг: Принципы, "alert on symptoms, not causes".
        *   Prometheus Alertmanager: Получение алертов, группировка, маршрутизация, интеграции (Slack, Email).
        *   C.A.L.M.S. Framework (Culture, Automation, Lean, Measurement, Sharing): Повторение и закрепление принципов на практике.
    *   **Практические Задачи (Hands-on Labs):**
        1.  **PLG Stack Setup:** Развернуть Loki, Promtail (или другой лог-агент), Prometheus, Grafana и Alertmanager (можно использовать Docker Compose или Helm Chart для K8s).
        2.  **Logging Integration:** Настроить лог-агент (например, Promtail как DaemonSet в K8s), чтобы он собирал логи из контейнеров вашего приложения и отправлял их в Loki.
        3.  **Log Analysis in Grafana:** Настроить Grafana Data Source для Loki. Создать дашборд для просмотра и фильтрации логов вашего приложения с помощью LogQL.
        4.  **Metrics Integration:** Интегрировать Prometheus Exporter в ваше приложение (например, Spring Boot Actuator с Prometheus). Настроить Prometheus для скрапинга метрик с вашего приложения (через Service Discovery в K8s или static config).
        5.  **Monitoring Dashboard:** Настроить Grafana Data Source для Prometheus. Создать дашборд для визуализации ключевых метрик вашего приложения (запросы в секунду, время ответа, ошибки, загрузка ресурсов).
        6.  **Basic Alerting:** Написать правила алертинга для Prometheus (например, высокая частота ошибок, долгое время ответа). Настроить Alertmanager для отправки уведомлений (например, в файл, консоль или mock-сервис).
        7.  **Review CALMS:** Проанализировать, как пройденный курс покрыл каждый аспект фреймворка CALMS.
    *   **Результат:** Полностью настроенный стек мониторинга и логирования, который собирает данные о работе вашего приложения, визуализирует их и оповещает о проблемах. Глубокое понимание важности измеримости (Measurement) в DevOps.