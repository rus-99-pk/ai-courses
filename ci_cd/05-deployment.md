# Тема 5: Непрерывная доставка (Continuous Deployment)

## Какую проблему решаем?

Мы уже умеем тестировать код, собирать артефакты и работать с секретами. Финальный шаг — доставить наше приложение пользователю. Мы автоматизируем процесс развертывания (деплоя) приложения на сервер.

### Пример 1: Деплой статического сайта на GitLab/GitHub Pages

Это самый простой способ развернуть статический сайт (HTML/CSS/JS). Обе платформы предоставляют бесплатный хостинг для таких сайтов прямо из вашего репозитория.

### Пример 2: Деплой на удаленный сервер по SSH

Классический сценарий: у вас есть виртуальный сервер (VPS/VDS), и вам нужно скопировать туда файлы приложения и перезапустить сервис.

---

### GitLab CI

#### Пример 1: Деплой на GitLab Pages

GitLab Pages имеет строгие соглашения. Чтобы деплой сработал, задача должна:
1.  Называться `pages`.
2.  Создавать артефакт в виде папки `public`.

```yml
# .gitlab-ci.yml
stages:
  - deploy

# Специальное имя задачи для GitLab Pages
pages:
  stage: deploy
  script:
    # GitLab Pages ищет контент в папке 'public'
    - mkdir public
    - echo "Hello from GitLab Pages!" > public/index.html
    - cp -r build/* public/ # Копируем наш сайт из артефактов предыдущей задачи
  artifacts:
    paths:
      - public # Обязательно указываем эту папку
  rules:
    # Деплоить только из ветки main
    - if: '$CI_COMMIT_BRANCH == "main"'
```
После выполнения этой задачи ваш сайт будет доступен по адресу `https://<your-username>.gitlab.io/<your-project-name>`.

#### Пример 2: Деплой по SSH

**Подготовка:**
1.  Сгенерируйте SSH-ключ (`ssh-keygen -t ed25519 -C "gitlab-ci"`).
2.  Добавьте **публичный** ключ (`.pub`) в файл `~/.ssh/authorized_keys` на вашем сервере.
3.  Добавьте **приватный** ключ в GitLab (`Settings -> CI/CD -> Variables`) как **File-type** переменную с именем `SSH_PRIVATE_KEY`.

```yml
# .gitlab-ci.yml
deploy_to_server:
  stage: deploy
  image: alpine # Легковесный образ
  before_script:
    - apk add openssh-client rsync # Устанавливаем SSH-клиент и rsync
    - eval $(ssh-agent -s)
    # Добавляем приватный ключ в ssh-agent
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    # Создаем директорию .ssh и добавляем хост сервера в known_hosts
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "SERVER_IP_ADDRESS ecdsa-sha2-nistp256 AAAA..." >> ~/.ssh/known_hosts
  script:
    - echo "Deploying to server..."
    # Копируем файлы на сервер с помощью rsync (эффективнее, чем scp)
    - rsync -rav build/ user@SERVER_IP_ADDRESS:/var/www/my-app/
    # Выполняем команду на сервере для перезапуска сервиса
    - ssh user@SERVER_IP_ADDRESS "systemctl restart my-app.service"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```
**Разбор команд:**
*   `before_script` готовит окружение для SSH-соединения: устанавливает клиент, запускает ssh-agent, добавляет ключ и настраивает `known_hosts` для безопасности.
*   `rsync`: Копирует файлы. `user@SERVER_IP_ADDRESS` и путь `/var/www/my-app/` замените на свои.
*   `ssh user@...`: Выполняет удаленную команду на сервере.

---

### GitHub Actions

#### Пример 1: Деплой на GitHub Pages

**Подготовка:**
В настройках репозитория (`Settings -> Pages`) выберите `Source: GitHub Actions`.

```yml
# .github/workflows/deploy-pages.yml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ "main" ]

# Даем права на запись для деплоя
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v3

      - name: Build
        run: |
          mkdir public
          echo "Hello from GitHub Pages" > public/index.html

      - name: Setup Pages
        uses: actions/configure-pages@v3

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: './public'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```
После выполнения сайт будет доступен по адресу `https://<your-username>.github.io/<your-project-name>`.

#### Пример 2: Деплой по SSH
Используем готовый экшен для простоты.

**Подготовка:**
Добавьте в Secrets репозитория:
*   `SSH_PRIVATE_KEY`: приватный SSH-ключ.
*   `SSH_HOST`: IP-адрес или домен вашего сервера.
*   `SSH_USER`: имя пользователя на сервере.

```yml
# .github/workflows/deploy-ssh.yml
name: Deploy to Server via SSH

on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build project (example)
        run: echo "Building..." # Здесь может быть ваша команда сборки

      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/my-app
            git pull
            # Здесь могут быть команды копирования файлов, перезапуска сервиса и т.д.
            # Пример с копированием:
            # scp -r ./build/* ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}:/var/www/my-app/
            echo "Restarting service..."
            systemctl restart my-app.service
```
**Разбор команд:**
*   `appleboy/ssh-action`: Популярный экшен, который берет на себя всю сложную настройку SSH.
*   `with`: Мы передаем ему хост, пользователя и ключ из секретов.
*   `script`: Команды, которые будут выполнены на удаленном сервере.