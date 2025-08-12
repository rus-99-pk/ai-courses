# Тема 7: Ручные триггеры и утверждения

## Какую проблему решаем?

Полностью автоматический деплой на `production` после каждого коммита в `main` может быть рискованным. Часто требуется, чтобы ответственный сотрудник (например, тимлид или DevOps-инженер) вручную подтвердил развертывание после того, как все тесты и сборки прошли успешно.

Нам нужен "ручной тормоз" или "кнопка подтверждения" перед критически важным шагом.

### Задача

Настроить пайплайн так, чтобы задача деплоя на `production` запускалась не автоматически, а только после нажатия кнопки в интерфейсе GitLab или GitHub.

---

### GitLab CI

В GitLab для этого есть очень простое и элегантное решение: `when: manual`.

```yml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build_job:
  stage: build
  script:
    - echo "Building the app..."
    - mkdir app && echo "my_app_binary" > app/run

test_job:
  stage: test
  script:
    - echo "Testing the app..."

deploy_to_production:
  stage: deploy
  # Вот и вся магия:
  when: manual
  script:
    - echo "Deploying to PRODUCTION!"
  # Добавим привязку к окружению для наглядности
  environment:
    name: production
  rules:
    # Показывать эту задачу только для ветки main
    - if: '$CI_COMMIT_BRANCH == "main"'
```

**Разбор команд:**
*   `when: manual`: Эта директива указывает GitLab, что задачу не нужно запускать автоматически. Вместо этого в интерфейсе пайплайна напротив этой задачи появится кнопка "play" (▶️). Пайплайн будет ждать, пока кто-нибудь не нажмет ее.
*   `environment`: Когда вы используете окружения, GitLab ведет историю деплоев, что очень удобно.

**Как это выглядит:**
Пайплайн успешно пройдет этапы `build` и `test`, а затем остановится. В UI вы увидите, что этап `deploy` ожидает ручного запуска.

---

### GitHub Actions

В GitHub есть два основных способа добиться похожего поведения: `workflow_dispatch` и `environments` с ревьюерами. Второй способ наиболее точно соответствует задаче.

#### Способ 2: Environments с обязательным утверждением (рекомендуемый)

Это самый правильный способ создать "ворота" для деплоя.

**Подготовка (в интерфейсе GitHub):**
1.  Зайдите в репозиторий на GitHub -> `Settings` -> `Environments`.
2.  Нажмите `New environment`. Назовите его `production`.
3.  В настройках этого окружения найдите секцию `Required reviewers` и добавьте себя (или вашего тимлида) в качестве ревьюера.
4.  Сохраните.

**Конфигурация `.github/workflows/main.yml`:**

```yml
# .github/workflows/main.yml
name: Manual Approval Demo

on:
  push:
    branches: [ "main" ]

jobs:
  build_and_test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building and testing..."

  deploy_to_production:
    runs-on: ubuntu-latest
    # Эта задача зависит от успешной сборки
    needs: build_and_test
    # Привязываем задачу к нашему защищенному окружению
    environment: production
    steps:
      - name: Deploy
        run: echo "Deploying to PRODUCTION!"
```

**Разбор команд:**
*   `environment: production`: Мы указываем, что эта задача относится к окружению `production`.
*   Поскольку в настройках этого окружения мы указали обязательное утверждение, GitHub Actions автоматически поставит workflow на паузу перед выполнением этой задачи.
*   Человек, указанный как ревьюер, получит уведомление. Ему нужно будет зайти в запуск workflow и нажать кнопку `Review deployments`, а затем `Approve and deploy`.

#### Способ 1: Ручной запуск всего workflow (`workflow_dispatch`)

Этот способ позволяет запустить весь пайплайн вручную, а не только один шаг.

```yml
# .github/workflows/manual-deploy.yml
name: Manual Deploy to Production

on:
  # Этот триггер добавляет кнопку "Run workflow" в UI
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (e.g., 1.2.3)'
        required: true
        default: 'latest'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ github.event.inputs.version }} to production..."
```
**Разбор команд:**
* `on: workflow_dispatch`: Добавляет кнопку "Run workflow" во вкладке `Actions`.
* `inputs`: Позволяет определить параметры, которые пользователь должен будет ввести перед запуском.