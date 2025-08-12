# Тема 8: Матричные сборки

## Какую проблему решаем?

Ваше приложение должно поддерживать несколько версий интерпретатора/компилятора (например, Python 3.8, 3.9, 3.10) или работать на разных операционных системах (Linux, Windows, macOS). Запускать тесты для каждой комбинации по очереди долго и неудобно.

**Матричная сборка** позволяет определить набор переменных и автоматически создать параллельные задачи для каждой их комбинации.

### Задача

Автоматически запустить тесты для нашего Python-приложения на версиях Python 3.8, 3.9 и 3.10 одновременно.

---

### GitLab CI

Для создания матрицы используется директива `parallel: matrix`.

```yml
# .gitlab-ci.yml
stages:
  - test

python-tests:
  stage: test
  # Определяем матрицу
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.8", "3.9", "3.10"]
        # Можно добавлять и другие переменные, они будут комбинироваться
        # REPORT_NAME: ["py38-report", "py39-report", "py40-report"]
  
  # Используем переменную из матрицы, чтобы указать нужный Docker-образ
  image: python:$PYTHON_VERSION
  
  script:
    - python -V # Показать версию Python, чтобы убедиться
    - pip install pytest
    - pytest # Запуск ваших тестов
```

**Разбор команд:**
*   `parallel: matrix:`: Указывает GitLab, что нужно создать несколько параллельных задач.
*   `- PYTHON_VERSION: ["3.8", "3.9", "3.10"]`: Мы определяем переменную `PYTHON_VERSION` и ее возможные значения. GitLab создаст 3 задачи, и в каждой из них переменная `$PYTHON_VERSION` будет иметь одно из этих значений (`3.8`, `3.9` или `3.10`).
*   `image: python:$PYTHON_VERSION`: Мы используем эту переменную для динамического выбора Docker-образа.

**Как это выглядит:**
В разделе CI/CD -> Jobs вы увидите не одну задачу `python-tests`, а три:
*   `python-tests: [3.8]`
*   `python-tests: [3.9]`
*   `python-tests: [3.10]`
Все они будут выполняться одновременно (если у вас достаточно свободных раннеров).

---

### GitHub Actions

В GitHub Actions матрица настраивается с помощью `strategy: matrix`.

```yml
# .github/workflows/main.yml
name: Matrix Build Demo

on:
  push:
    branches: [ "main" ]

jobs:
  test-python:
    runs-on: ubuntu-latest
    # Определяем стратегию и матрицу
    strategy:
      matrix:
        # Имя переменной и ее значения
        python-version: ['3.8', '3.9', '3.10']

    steps:
      - uses: actions/checkout@v3

      # Используем переменную из матрицы для настройки Python
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pytest

      - name: Run tests
        run: pytest
```

**Разбор команд:**
*   `strategy: matrix:`: Аналог `parallel: matrix` в GitLab.
*   `python-version: ['3.8', '3.9', '3.10']`: Определяем переменную `python-version` и ее значения.
*   `name: Set up Python ${{ matrix.python-version }}`: Мы можем использовать переменную из матрицы в любом месте, где поддерживаются выражения. Здесь мы используем ее в названии шага для наглядности.
*   `python-version: ${{ matrix.python-version }}`: Передаем значение из матрицы в экшен `setup-python`, который установит нужную версию.

**Как это выглядит:**
В интерфейсе Actions вы увидите одну задачу `test-python`, которая развернется в три параллельных выполнения, каждое для своей версии Python.