# Тема 2: Линтинг и Тестирование

## Какую проблему решаем?

Код должен быть не только рабочим, но и качественным: соответствовать стандартам стиля, не содержать синтаксических ошибок и проходить тесты. Делать это вручную — ненадежно. Мы автоматизируем эти проверки.

**Пример:** Будем использовать Python. Нам нужно, чтобы CI:
1.  Проверил стиль кода с помощью `flake8`.
2.  Запустил тесты с помощью `pytest`.

### Подготовка

Создадим три файла в нашем репозитории:

`app.py`
```python
def add(a, b):
    # Эта функция написана хорошо
    return a + b

def bad_formatted_function():
    # Эта функция нарушает стиль (слишком длинная строка)
    print("This is a very very very very very very very very very very very long line that will fail flake8 check")
```

`test_app.py`
```python
from app import add

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
```

`requirements.txt`
```
pytest
flake8
```
---
### GitLab CI

```yml
# .gitlab-ci.yml
stages:
  - test

# Используем готовый образ Python
default:
  image: python:3.9

lint-and-test:
  stage: test
  before_script:
    - pip install -r requirements.txt
  script:
    - echo "Running linter..."
    - flake8 . --count --show-source --statistics
    - echo "Running tests..."
    - pytest
```
**Разбор команд:**
*   `default: image: python:3.9`: Устанавливает Docker-образ `python:3.9` по умолчанию для всех `jobs`. Теперь все команды будут выполняться в контейнере с установленным Python.
*   `before_script`: Команды, которые выполняются *перед* основным `script`. Идеально для установки зависимостей.
*   `flake8 .`: Запускает линтер. Если он найдет ошибки, команда завершится с ненулевым кодом, и `job` провалится.
*   `pytest`: Запускает тесты. Если тест не пройдет, `job` также провалится.

---
### GitHub Actions

```yml
# .github/workflows/main.yml
name: Linting and Testing

on:
  push:
    branches: [ "main" ]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      # Шаг 1: Получаем код из репозитория
      - name: Check out repository code
        uses: actions/checkout@v3

      # Шаг 2: Настраиваем Python
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      # Шаг 3: Устанавливаем зависимости
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      # Шаг 4: Запускаем линтер
      - name: Lint with flake8
        run: |
          flake8 . --count --show-source --statistics

      # Шаг 5: Запускаем тесты
      - name: Test with pytest
        run: pytest
```
**Разбор команд:**
*   `uses: actions/checkout@v3`: Это **Action**. Он скачивает код вашего репозитория на раннер. Это почти всегда первый шаг.
*   `uses: actions/setup-python@v4`: Еще один Action из Marketplace. Он настраивает нужную версию Python для дальнейших шагов.
*   `with: python-version: '3.9'`: Параметры для Action.
*   Каждая `run` команда — логически отделенный шаг. Если любой из них провалится, вся `job` будет помечена как проваленная.