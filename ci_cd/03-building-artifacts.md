# Тема 3: Сборка артефактов

## Какую проблему решаем?

После того как наш код прошел тесты, мы хотим получить результат, который можно будет куда-то установить или запустить. Это может быть:
*   Архив с файлами для веб-сервера.
*   Скомпилированный бинарный файл.
*   **Docker-образ.**

Сейчас все, что происходит на раннере, исчезает после завершения задачи. **Артефакты** — это способ сохранить файлы и передать их между задачами или скачать себе на компьютер.

### Задача 1: Сборка статического сайта

Предположим, у нас есть папка `build` с готовым сайтом, и мы хотим сохранить ее как `.zip` архив.

### Задача 2: Сборка Docker-образа

Это самый популярный способ упаковки приложений. Мы напишем `Dockerfile` и настроим CI для автоматической сборки образа и его отправки в репозиторий образов (Registry).

---

### GitLab CI

В GitLab артефакты — это встроенная и очень простая в использовании функциональность.

**Подготовка к Docker:**
1.  Создайте в корне проекта файл `Dockerfile`:
    ```dockerfile
    # Используем официальный образ Python
    FROM python:3.9-slim
    
    # Устанавливаем рабочую директорию
    WORKDIR /app
    
    # Копируем файлы зависимостей и приложения
    COPY requirements.txt .
    COPY app.py .
    
    # Устанавливаем зависимости
    RUN pip install --no-cache-dir -r requirements.txt
    
    # Команда для запуска приложения (пример)
    CMD ["python", "app.py"]
    ```

**Конфигурация `.gitlab-ci.yml`:**

```yml
# .gitlab-ci.yml

stages:
  - build
  - push_image

# Задача для сборки статического архива
build_static_job:
  stage: build
  script:
    - mkdir build # Создаем папку для нашего "сайта"
    - echo "Это наш будущий сайт" > build/index.html
  artifacts:
    name: "static-site-$CI_COMMIT_SHORT_SHA" # Имя артефакта
    paths:
      - build/ # Указываем, какую папку сохранить
    expire_in: 1 week # Сколько хранить артефакт

# Задача для сборки и отправки Docker-образа
# Эта задача требует, чтобы GitLab Runner мог работать с Docker (Docker-in-Docker)
# Встроенные раннеры GitLab.com это умеют.
build_and_push_docker:
  stage: push_image
  image: docker:latest # Используем специальный образ с Docker
  services:
    - docker:dind # Запускаем сервис Docker-in-Docker
  before_script:
    # $CI_REGISTRY_USER, $CI_REGISTRY_PASSWORD, $CI_REGISTRY - это предопределенные переменные GitLab
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    # $CI_COMMIT_TAG или $CI_COMMIT_SHORT_SHA - отличные кандидаты для тега образа
    - IMAGE_TAG="$CI_REGISTRY_IMAGE:${CI_COMMIT_REF_SLUG:-latest}"
    - docker build -t "$IMAGE_TAG" .
    - docker push "$IMAGE_TAG"
  rules:
    # Запускать эту задачу только для тегов или для ветки main
    - if: '$CI_COMMIT_TAG || $CI_COMMIT_BRANCH == "main"'
```

**Разбор команд:**
*   `artifacts`: Ключевое слово для создания артефактов.
*   `paths`: Список файлов и директорий для сохранения.
*   `expire_in`: Устанавливает срок жизни артефакта. Очень полезно, чтобы не засорять хранилище.
*   `image: docker:latest` и `services: - docker:dind`: Специальная конфигурация, чтобы внутри вашего CI-задания можно было выполнять команды `docker`.
*   `docker login ...`: Авторизация во встроенном в GitLab репозитории образов. Переменные (`$CI_REGISTRY_USER` и т.д.) GitLab подставляет автоматически.
*   `$CI_REGISTRY_IMAGE`: Предопределенная переменная, содержащая путь к вашему реестру образов.
*   `rules`: Позволяет гибко управлять, когда задача должна выполняться.

---

### GitHub Actions

В GitHub Actions для работы с артефактами и Docker используются готовые экшены из Marketplace.

**Конфигурация `.github/workflows/main.yml`:**

```yml
# .github/workflows/main.yml
name: Build and Push

on:
  push:
    branches: [ "main" ]
    tags: [ 'v*.*.*' ] # Запускать также для тегов вида v1.0.0

jobs:
  build-static:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build static site
        run: |
          mkdir build
          echo "Это наш будущий сайт" > build/index.html

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: static-site
          path: build/

  build-and-push-docker:
    runs-on: ubuntu-latest
    # Эта задача зависит от успешного выполнения build-static
    needs: build-static
    steps:
      - uses: actions/checkout@v3

      # Экшен для входа в Docker Hub (или другой registry)
      # Требует создания секретов DOCKERHUB_USERNAME и DOCKERHUB_TOKEN в настройках репозитория
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # Экшен, который собирает и пушит образ
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: . # Где искать Dockerfile
          push: true # Говорим, что нужно пушить
          # Генерируем теги для образа
          tags: |
            your_dockerhub_username/my-app:latest
            your_dockerhub_username/my-app:${{ github.sha }}
```

**Разбор команд:**
*   `uses: actions/upload-artifact@v3`: Экшен для загрузки артефактов.
*   `with: name: ... path: ...`: Параметры для экшена, указывающие имя артефакта и путь к файлам.
*   `needs: build-static`: Указывает, что эта задача (`build-and-push-docker`) начнется только после успешного завершения задачи `build-static`.
*   `uses: docker/login-action@v2`: Экшен для безопасной авторизации в реестре Docker.
*   `${{ secrets.DOCKERHUB_USERNAME }}`: Так в GitHub Actions используются секреты. Мы разберем их в следующей теме.
*   `uses: docker/build-push-action@v4`: Мощный экшен, который делает за вас всю работу по сборке и отправке образа.
*   `tags: ...`: Здесь мы указываем, с какими тегами будет отправлен образ. `github.sha` — это хэш коммита, что очень удобно для версионирования.