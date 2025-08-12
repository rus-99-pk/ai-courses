# Тема 7: Лучшие практики и интеграция с CI/CD

## Какую проблему мы решаем?

Мы научились писать код, управлять состоянием и использовать модули. Но как сделать наш IaC-процесс по-настоящему надежным, безопасным и готовым для командной работы в production?

**Проблемы, которые возникают без лучших практик:**
-   **"Работает на моей машине"**: Запуск `terraform apply` с локального компьютера — это рискованно. У вас могут быть другие версии Terraform, провайдеров или неправильно настроенные переменные окружения.
-   **Отсутствие ревью кода**: Коллега может случайно закоммитить код, который удалит production базу данных. Без проверки (code review) это изменение может быть применено.
-   **Ручное применение**: Процесс, зависящий от человека, который должен не забыть запустить `plan`, проверить его и запустить `apply`, подвержен ошибкам.
-   **Беспорядок в коде**: Без единого стиля и структуры проект быстро становится сложным для понимания и поддержки.

## Решение: Автоматизация и стандартизация

Мы должны перенести выполнение Terraform в автоматизированную среду (CI/CD) и следовать общепринятым стандартам разработки.

### Задача: Настроить простой CI/CD пайплайн для нашего Terraform-кода с использованием GitHub Actions.

**Наш пайплайн будет делать следующее:**
1.  **При создании Pull Request (PR)**:
    -   Проверять форматирование кода (`fmt`).
    -   Инициализировать проект (`init`).
    -   Проверять код на синтаксические ошибки (`validate`).
    -   Генерировать план изменений (`plan`) и публиковать его как комментарий в PR для ревью.
2.  **При слиянии PR в `main` ветку**:
    -   Автоматически применять изменения (`apply`).

## Лучшие практики (Best Practices)

Перед тем как перейти к CI/CD, запомните эти правила:

1.  **Структура проекта**: Используйте логичную структуру с разделением на файлы (`variables.tf`, `outputs.tf`, `providers.tf`) и активно используйте модули для переиспользуемых компонентов.
2.  **Управление состоянием**: **Всегда** используйте удаленный бэкенд с блокировками (S3 + DynamoDB). Локальное состояние — только для песочницы.
3.  **Именование**: Придерживайтесь единого стиля именования ресурсов и переменных (например, `snake_case`).
4.  **Версионирование**: Четко фиксируйте версии Terraform и провайдеров в блоке `terraform {}`. Это предотвратит неожиданное поведение после обновления провайдера.
    ```hcl
    terraform {
      required_version = "~> 1.3" # Разрешает версии 1.3.x, но не 1.4.0
      required_providers {
        aws = {
          source  = "hashicorp/aws"
          version = "~> 4.15" # Разрешает 4.15.x, 4.16.x и т.д., но не 5.0
        }
      }
    }
    ```
5.  **Безопасность**: Никогда не храните "секреты" (ключи, пароли) в коде. Используйте переменные окружения, Vault или секрет-менеджеры вашего облака. В CI/CD для этого есть специальный раздел "Secrets".
6.  **Минимальные изменения**: Старайтесь делать небольшие, атомарные изменения в коде. Один PR — одна логическая задача. Это упрощает ревью и отладку.

## Практика: Настройка GitHub Actions

Для нашего проекта из предыдущего урока (`06_Project_Deploying_a_Web_Server`) создадим CI/CD пайплайн.

1.  **Создайте репозиторий на GitHub** и загрузите туда код проекта.

2.  **Настройте секреты в GitHub**:
    -   В вашем репозитории перейдите в `Settings` -> `Secrets and variables` -> `Actions`.
    -   Нажмите `New repository secret` и создайте два секрета:
        -   `AWS_ACCESS_KEY_ID`: Ваш ключ доступа AWS.
        -   `AWS_SECRET_ACCESS_KEY`: Ваш секретный ключ.
    -   GitHub Actions будет безопасно использовать их для аутентификации в AWS.

3.  **Создайте файл пайплайна**:
    В корне вашего проекта создайте директорию `.github/workflows/` и в ней файл `terraform.yml`.

### `.github/workflows/terraform.yml`
```yaml
name: 'Terraform CI/CD'

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest
    env:
      # Передаем секреты в переменные окружения, которые использует AWS провайдер
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      # Укажите свой IP здесь или в секретах, если он должен быть скрыт
      TF_VAR_my_ip: "YOUR_STATIC_IP/32" 

    steps:
      # Шаг 1: Клонируем репозиторий
      - name: Checkout
        uses: actions/checkout@v3

      # Шаг 2: Устанавливаем Terraform (можно использовать официальный action)
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.3.7 # Укажите версию из вашего кода

      # Шаг 3: Форматирование (выдаст ошибку, если код не отформатирован)
      - name: Terraform Format
        id: fmt
        run: terraform fmt -check
        continue-on-error: true # Не прерывать пайплайн, просто показать ошибку

      # Шаг 4: Инициализация
      # Здесь нужно передать имя S3 бакета. Лучше делать это через переменные, а не хардкодить.
      # Для примера оставим так, но в реальности лучше использовать переменные окружения.
      - name: Terraform Init
        id: init
        run: terraform init -backend-config="bucket=my-awesome-project-tfstate-2023" -backend-config="key=final-project/terraform.tfstate" -backend-config="region=us-east-1" -backend-config="dynamodb_table=terraform-state-locks"

      # Шаг 5: Валидация
      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color

      # Шаг 6: План
      - name: Terraform Plan
        id: plan
        # Запускаем plan только для pull request
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color
        continue-on-error: true # Если в плане ошибка, PR все равно можно будет создать

      # Шаг 7: Обновляем PR с выводом плана (используем сторонний action)
      - uses: actions/github-script@v6
        if: github.event_name == 'pull_request'
        env:
          PLAN: "terraform\n${{ steps.plan.outputs.stdout }}"
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `#### Terraform Format and Style 🖌\`${{ steps.fmt.outcome }}\`
            #### Terraform Initialization ⚙️\`${{ steps.init.outcome }}\`
            #### Terraform Validation 🤖\`${{ steps.validate.outcome }}\`
            <details><summary>Validation Output</summary>

            \`\`\`\n
            ${{ steps.validate.outputs.stdout }}
            \`\`\`

            </details>

            #### Terraform Plan 📖\`${{ steps.plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`\n
            ${process.env.PLAN}
            \`\`\`

            </details>

            *Pushed by: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      # Шаг 8: Применение (только для ветки main)
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```
*Примечание: в `TF_VAR_my_ip` нужно указать ваш IP. Для удаленного бэкенда в `init` нужно передать конфигурацию. В реальных проектах это тоже делается через переменные или файлы конфигурации для разных окружений.*

## Команда-методичка

1.  **Создайте новую ветку** в вашем репозитории, например `feature/add-tags`.
2.  **Внесите небольшое изменение** в код. Например, добавьте новый тег для VPC в `main.tf`.
3.  **Закоммитьте и запушьте** эту ветку.
4.  **Создайте Pull Request** из вашей ветки в `main`.
5.  **Проверьте результат**: перейдите на вкладку `Actions` в вашем репозитории. Вы увидите, как запустился ваш пайплайн. После его завершения в вашем PR появится комментарий от бота с результатами проверок и планом Terraform.
6.  **Ревью**: Теперь ваш коллега может посмотреть на план прямо в PR и убедиться, что изменение безопасно.
7.  **Слияние (Merge)**: После одобрения и слияния PR в `main`, GitHub Actions снова запустит пайплайн. Но на этот раз он дойдет до шага `Terraform Apply` и автоматически применит изменения к вашей реальной инфраструктуре.

---

### Заключение курса

**Поздравляем!** Вы прошли путь от основ IaC до создания полностью автоматизированных, надежных и готовых к командной работе процессов управления инфраструктурой.

**Что дальше?**
-   **Terraform Cloud / Enterprise**: Изучите облачную платформу от HashiCorp, которая предоставляет готовый UI, управление состояниями, приватный реестр модулей и политики (Sentinel).
-   **Сложные провайдеры**: Попробуйте управлять Kubernetes (`kubernetes` provider), DNS-записями (`cloudflare` provider) или пользователями в GitHub (`github` provider).
-   **Terragrunt**: Познакомьтесь с этой популярной "оберткой" для Terraform, которая помогает управлять множеством окружений и состояний, сохраняя ваш код DRY.

Infrastructure as Code — это не просто инструмент, это философия, которая делает вашу работу с инфраструктурой такой же надежной, предсказуемой и эффективной, как современная разработка ПО. Удачи!