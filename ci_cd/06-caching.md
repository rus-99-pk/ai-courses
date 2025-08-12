# Тема 6: Оптимизация с помощью кэширования

## Какую проблему решаем?

Наши CI/CD пайплайны при каждом запуске начинают с чистого листа. Это значит, что они каждый раз скачивают все зависимости (`npm install`, `pip install`, `mvn install` и т.д.). На больших проектах это может занимать несколько минут и является пустой тратой времени и ресурсов, если зависимости не менялись.

**Кэширование** позволяет сохранять файлы и папки (например, папку с загруженными зависимостями) между запусками пайплайнов, что значительно их ускоряет.

### Задача

Ускорить установку зависимостей для Node.js проекта. Мы будем кэшировать папку `node_modules`.

**Подготовка:**
Создайте в корне проекта файл `package.json` и `package-lock.json`:
`package.json`
```json
{
  "name": "my-cool-app",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```
Запустите `npm install` локально, чтобы сгенерировался `package-lock.json`, и закоммитьте оба файла.

---

### GitLab CI

В GitLab кэш — это встроенная функция, которая настраивается в `.gitlab-ci.yml`.

```yml
# .gitlab-ci.yml

stages:
  - test

# Используем образ с Node.js
default:
  image: node:18

test_job:
  stage: test
  cache:
    # Ключ кэша. Если файл package-lock.json изменится, GitLab создаст новый кэш.
    key:
      files:
        - package-lock.json
    # Что именно кэшировать.
    paths:
      - node_modules/
    # Политика: скачивать кэш в начале и обновлять в конце.
    policy: pull-push
  script:
    - echo "Installing dependencies..."
    # npm ci быстрее и надежнее для CI, чем npm install
    - npm ci
    - echo "Dependencies installed!"
    - echo "Running some tasks..."
```

**Разбор команд:**
*   `cache`: Главное слово для настройки кэширования.
*   `key: files: - package-lock.json`: Это **самая важная** часть. Мы говорим GitLab: "Создай уникальный ключ для кэша на основе хеш-суммы файла `package-lock.json`". Если этот файл не меняется, ключ остается прежним, и GitLab использует старый кэш. Если вы обновили зависимость, `package-lock.json` изменится, ключ станет другим, и GitLab создаст новый кэш.
*   `paths: - node_modules/`: Указывает, какую папку нужно сохранить в кэше.
*   `policy: pull-push`: `pull` — скачать кэш в начале задачи. `push` — обновить кэш в конце, если задача успешна. `pull-push` делает и то, и другое.
*   `npm ci`: Эта команда использует `package-lock.json` для быстрой и точной установки зависимостей. Она идеально подходит для CI.

---

### GitHub Actions

В GitHub Actions для кэширования используется специальный экшен `actions/cache`.

```yml
# .github/workflows/main.yml
name: Caching Demo

on:
  push:
    branches: [ "main" ]

jobs:
  test_job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Cache node modules
        id: cache-npm # Даем id, чтобы ссылаться на результат шага
        uses: actions/cache@v3
        with:
          # Путь, который мы кэшируем
          path: node_modules
          # Ключ кэша. Используем ОС и хеш файла package-lock.json
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          # Ключ для восстановления, если точного совпадения не найдено
          restore-keys: |
            ${{ runner.os }}-npm-

      # Запускаем npm ci только если кэш не был найден
      - name: Install dependencies
        if: steps.cache-npm.outputs.cache-hit != 'true'
        run: npm ci

      - name: Run some tasks
        run: echo "Dependencies are ready!"
```

**Разбор команд:**
*   `uses: actions/cache@v3`: Экшен, который управляет кэшем.
*   `id: cache-npm`: Мы даем этому шагу идентификатор, чтобы потом проверить результат его работы.
*   `path: node_modules`: Что кэшировать.
*   `key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}`: Очень похожая на GitLab логика. Ключ состоит из имени ОС (`runner.os`) и хеша файла `package-lock.json`. `hashFiles` — встроенная функция для вычисления хеша.
*   `if: steps.cache-npm.outputs.cache-hit != 'true'`: Это главная "магия". Экшен `actions/cache` возвращает результат `cache-hit: 'true'`, если он нашел и восстановил кэш. Мы используем это условие, чтобы **пропустить** шаг `npm ci`, если зависимости уже есть.