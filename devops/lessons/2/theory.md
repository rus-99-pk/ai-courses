*   **Урок 2: Гибкие Методологии и Continuous Integration**
    *   **Цель:** Погрузиться в принципы DevOps и CI, научиться использовать пайплайны и интегрировать инструменты анализа качества/безопасности кода.
    *   **Ключевые Темы:**
        *   DevOps-культура: 3 пути DevOps (Value Stream, Feedback Loops, Experimentation/Learning).
        *   Гибкие методологии (Agile): Kanban, Scrum (краткий обзор в контексте DevOps).
        *   Continuous Integration (CI): Принципы, лучшие практики, преимущества.
        *   Введение в CI/CD Pipeline: Этапы (Source, Build, Test, Deploy...).
        *   Jenkins Declarative Pipeline: Синтаксис, Stages, Steps, Agents, Post actions.
        *   Альтернативные CI системы: GitLab CI (введение в синтаксис `.gitlab-ci.yml`), GitHub Actions, CircleCI (обзор).
        *   Анализ качества кода: SonarQube. Правила, Quality Gates, метрики.
        *   Анализ безопасности кода (SAST - Static Application Security Testing): SonarQube SAST, GitLab SAST.
    *   **Практические Задачи (Hands-on Labs):**
        1.  **Convert to Declarative Pipeline:** Переписать Jenkins Freestyle Job из Урока 1 в Jenkins Declarative Pipeline (создать `Jenkinsfile` в репозитории). Добавить шаги сборки, тестирования (если есть тесты в проекте).
        2.  **SonarQube Setup:** Развернуть SonarQube. Настроить проект в SonarQube.
        3.  **SonarQube Integration (Jenkins):** Интегрировать SonarQube в Jenkins Pipeline: добавить шаг анализа кода после сборки. Настроить Quality Gate в SonarQube и сделать его частью пайплайна (например, с помощью SonarQube Scanner for Jenkins), чтобы сборка падала, если качество/безопасность не соответствуют порогу.
        4.  **Explore GitLab CI:** Создать аккаунт на GitLab.com (или использовать локальный Gitea с поддержкой CI, если есть). Настроить `.gitlab-ci.yml` для того же проекта, replicating Jenkins Pipeline (build, test, sonar analysis). Сравнить подходы Jenkinsfile и `.gitlab-ci.yml`.
        5.  **SAST Practice:** Включить SAST в SonarQube анализ. Если возможно, добавить уязвимость в код приложения и посмотреть, как SonarQube ее обнаружит. Изучить отчеты. Если используете GitLab, настроить GitLab SAST.
    *   **Результат:** Пайплайн CI, определенный как код (`Jenkinsfile`), который автоматически собирает, тестирует и проверяет качество/безопасность кода при каждом изменении, давая быструю обратную связь. Понимание принципов CI и различий между CI системами.