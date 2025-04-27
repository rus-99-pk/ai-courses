Отлично! Приступаем к Модулю 4, который посвящен расширению принципов GitOps на управление инфраструктурой. В этом модуле три домашних задания.

Начнем с Terraform и интеграции его в GitOps.

---

### Методичка по ДЗ: Модуль 4, Тема 1

**Название ДЗ:** Управление простым ресурсом с помощью Terraform через Git

**Цель ДЗ:** Научиться описывать инфраструктуру с помощью Terraform и использовать Git в качестве источника истины для управления этой инфраструктурой, интегрировав процесс применения Terraform в CI/CD или локальный Git-хук/скрипт.

**Необходимые условия/Инструменты:**

1.  Установленный Git.
2.  Аккаунт на Git-хостинге (GitHub, GitLab, Bitbucket).
3.  Установленный Terraform CLI ([https://developer.hashicorp.com/terraform/downloads](https://developer.hashicorp.com/terraform/downloads)).
4.  **Выберите провайдера:**
    *   Аккаунт в облачном провайдере (AWS, Azure, GCP) и настроенные учетные данные для Terraform. **ИЛИ**
    *   Установленный Docker (для локальных провайдеров, если нужно).

**Подробные шаги (Выбираем реализацию: облачный ресурс с CI или локальный ресурс с локальным скриптом):**

**Вариант 1 (Предпочтительный для реальных сценариев): Облачный ресурс и CI**

Этот вариант ближе к реальной практике "Push-based GitOps" для инфраструктуры.

**Шаг 1.1: Выберите облачный ресурс и настройте провайдера**

Выберите простой ресурс для создания, например:
*   AWS: S3 Bucket
*   Azure: Resource Group
*   GCP: Storage Bucket

Убедитесь, что у вас есть учетные данные для доступа к облаку, настроенные для Terraform (через переменные окружения, файл credentials, или другим способом, специфичным для провайдера).

**Шаг 1.2: Создайте новый Git-репозиторий для Terraform кода**

Создайте новый **приватный** репозиторий на вашем Git-хостинге. Назовите его, например, `my-terraform-infra`.

**Шаг 1.3: Клонируйте репозиторий и напишите Terraform код**

Клонируйте репозиторий и создайте `.tf` файл (например, `main.tf`) для вашего ресурса.

```bash
git clone <URL_вашего_нового_репозитория>
cd my-terraform-infra
```

Пример `main.tf` для AWS S3 Bucket:

```terraform
# main.tf (Пример для AWS)
provider "aws" {
  region = "us-east-1" # Замените на ваш регион
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-gitops-bucket-$(uuid())" # S3 bucket имена должны быть уникальными. Используйте что-то уникальное или генератор.
  acl    = "private"

  tags = {
    Environment = "Dev"
    ManagedBy   = "Terraform-GitOps-Course"
  }
}

output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.my_bucket.id
}
```

*   **Важно:** Имена S3 бакетов глобально уникальны. Замените `"my-unique-gitops-bucket-$(uuid())"` на что-то гарантированно уникальное, например, добавив свое имя или случайный суффикс. Или используйте `prefix` вместо `bucket`.
*   Не забудьте инициализировать Terraform, когда будете готовы к применению (`terraform init`).

**Шаг 1.4: Настройте удаленное хранение state-файла (рекомендуется)**

В реальных проектах state-файл Terraform (`terraform.tfstate`) никогда не хранится локально или в Git. Его хранят удаленно (S3, GCS, Azure Blob, Terraform Cloud). Настройте backend в вашем `main.tf` или отдельном файле (например, `backend.tf`).

Пример backend.tf для AWS S3:

```terraform
# backend.tf (Пример для AWS)
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket" # Замените на имя реального S3 бакета для state
    key    = "terraform/my-terraform-infra/terraform.tfstate"
    region = "us-east-1" # Должен совпадать с регионом провайдера
    # dynamodb_table = "my-terraform-lock-table" # Рекомендуется для блокировки состояния
  }
}
```

*   **Внимание:** Вам потребуется заранее создать S3 bucket (и опционально DynamoDB таблицу для блокировки) для хранения state. Это можно сделать вручную или отдельным Terraform скриптом.

**Шаг 1.5: Настройте CI-пайплайн**

Используйте встроенные CI/CD инструменты вашего Git-хостинга (GitHub Actions, GitLab CI, Bitbucket Pipelines) или внешний инструмент (Jenkins, CircleCI).

Создайте файл конфигурации для CI в корне репозитория.

Пример `.github/workflows/terraform.yaml` для GitHub Actions:

```yaml
# .github/workflows/terraform.yaml
name: Terraform Apply on Push

on:
  push:
    branches:
      - main # Или master

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        # Optional: terraform_version: 1.5.0 # Укажите конкретную версию
        cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }} # Если используете Terraform Cloud

    - name: Terraform Init
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan -out=tfplan.binary # Сохраняем план в бинарный файл

    - name: Terraform Apply
      # Только применяем изменения после merge в основную ветку
      # В реальных сценариях apply часто делается вручную или после одобрения,
      # но для ДЗ мы автоматизируем его на push в main.
      run: terraform apply tfplan.binary

    # Настройка аутентификации для облачного провайдера
    # Это может быть через переменные окружения (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY),
    # файлы credentials, или Identity Provider (OIDC для GitHub Actions)
    # Пример для AWS через переменные окружения:
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: us-east-1 # Замените на ваш регион
```

*   **Важно:** Вам нужно будет настроить секреты в вашем Git-репозитории (например, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) с учетными данными для доступа к облаку. **Никогда не храните учетные данные в открытом виде в Git!**
*   Эта конфигурация выполняет `init`, `validate`, `plan` и `apply` при каждом пуше в ветку `main`. В реальной жизни `apply` часто делается в отдельном шаге после ручного одобрения или мерджа Pull Request-а.

**Шаг 1.6: Добавьте, закоммитьте и отправьте все файлы в Git**

```bash
git add .
git commit -m "Add initial Terraform code and CI pipeline"
git push origin main # Или master
```

*   **Наблюдайте за CI:** Перейдите в раздел CI/CD вашего Git-хостинга и наблюдайте за выполнением пайплайна. Он должен пройти все шаги, включая `terraform apply`.

**Шаг 1.7: Проверьте создание ресурса в облаке**

После успешного выполнения пайплайна, зайдите в консоль вашего облачного провайдера и убедитесь, что ресурс (например, S3 bucket) был создан.

**Шаг 1.8: Измените параметр ресурса в Terraform коде**

Отредактируйте ваш `main.tf` файл. Например, измените тег у вашего S3 бакета.

```terraform
# main.tf (фрагмент)
resource "aws_s3_bucket" "my_bucket" {
  ...
  tags = {
    Environment = "Production" # Изменили значение тега
    ManagedBy   = "Terraform-GitOps-Course"
    NewTag      = "AppliedViaGitOps" # Добавили новый тег
  }
}
...
```

**Шаг 1.9: Закоммитьте и отправьте изменение в Git**

```bash
git add main.tf
git commit -m "Update tags for S3 bucket"
git push origin main # Или master
```

*   **Наблюдайте за CI:** Снова наблюдайте за выполнением пайплайна. Он должен обнаружить изменение, выполнить `plan`, показывая, что будет изменено, и затем выполнить `apply`, применив изменение тегов.

**Шаг 1.10: Проверьте изменение ресурса в облаке**

Зайдите в консоль облачного провайдера и убедитесь, что теги вашего ресурса обновились.

**Проверка результатов (Измеряемость для Варианта 1):**

1.  Предоставьте ссылку на ваш Git-репозиторий (`my-terraform-infra`).
2.  Предоставьте содержимое `.tf` файлов (например, `main.tf` и `backend.tf`, если использовали).
3.  Опишите, какой CI/CD инструмент вы использовали (GitHub Actions, GitLab CI и т.д.) и предоставьте файл конфигурации пайплайна (например, `.github/workflows/terraform.yaml`). Объясните, как настроена аутентификация в облаке (использование секретов).
4.  Предоставьте ссылку на выполнение CI-пайплайна, которое создало ресурс. Покажите логи выполнения (`terraform apply`).
5.  Предоставьте ссылку на коммит в Git, который изменил параметр ресурса.
6.  Предоставьте ссылку на выполнение CI-пайплайна, которое применило это изменение. Покажите логи выполнения (`terraform apply`), где видно, что Terraform обновил ресурс.
7.  Предоставьте скриншот из консоли облачного провайдера или вывод команды облачного CLI (например, `aws s3api get-bucket-tagging --bucket <имя_бакета> --region <регион>`), подтверждающий, что ресурс был создан и его параметры (например, теги) были изменены.

**Вариант 2 (Упрощенный для локального выполнения): Локальный ресурс с локальным скриптом**

Этот вариант не использует облако или полноценный CI, но демонстрирует идею триггера из Git.

**Шаг 2.1: Установите Terraform и Git**

Убедитесь, что Terraform и Git установлены.

**Шаг 2.2: Создайте новый Git-репозиторий**

Создайте новый локальный Git-репозиторий.

```bash
mkdir my-local-terraform
cd my-local-terraform
git init
```

**Шаг 2.3: Напишите Terraform код с локальным провайдером**

Создайте файл `main.tf`. Используем `local_file` ресурс, который создает файл на диске.

```terraform
# main.tf (Пример с local_file)
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
      version = "~> 2.1"
    }
  }
}

resource "local_file" "hello_file" {
  filename = "hello.txt"
  content  = "Hello, GitOps!"
}
```

*   Этот код создаст файл `hello.txt` с указанным содержимым.

**Шаг 2.4: Напишите простой скрипт для запуска Terraform**

Создайте скрипт (например, `apply.sh` для Linux/macOS или `apply.bat` для Windows), который будет выполнять `terraform init` и `terraform apply`.

Пример `apply.sh`:

```bash
#!/bin/bash
set -e # Выход при ошибке

echo "Running terraform init..."
terraform init

echo "Running terraform apply..."
terraform apply -auto-approve # Используем -auto-approve для автоматического применения
```

*   Сделайте скрипт исполняемым: `chmod +x apply.sh`.
*   **Внимание:** `-auto-approve` используется здесь для автоматизации в рамках ДЗ. **В реальных проектах всегда выполняйте `terraform plan` и `terraform apply` отдельно с ручным подтверждением!**

**Шаг 2.5: Инициализируйте Terraform и выполните первый apply**

Выполните скрипт первый раз, чтобы создать ресурс.

```bash
./apply.sh
```

*   **Проверка:** Убедитесь, что файл `hello.txt` создан в той же директории.

**Шаг 2.6: Добавьте, закоммитьте файлы в Git**

```bash
git add .
git commit -m "Add initial local Terraform code and apply script"
# В этом варианте мы работаем локально, push не требуется, если вы не используете удаленный репозиторий для отчетности.
```

**Шаг 2.7: Измените параметр ресурса в Terraform коде**

Отредактируйте `main.tf` и измените содержимое файла `hello.txt`.

```terraform
# main.tf (фрагмент)
...
resource "local_file" "hello_file" {
  filename = "hello.txt"
  content  = "Hello from updated GitOps!" # Изменили содержимое
}
...
```

**Шаг 2.8: Закоммитьте изменение в Git**

```bash
git add main.tf
git commit -m "Update content of hello.txt"
```

**Шаг 2.9: Вручную запустите скрипт для применения изменения**

Это имитация "триггера" из Git. В реальной CI системе это бы произошло автоматически при пуше/мердже.

```bash
./apply.sh
```

*   **Проверка:** Убедитесь, что содержимое файла `hello.txt` обновилось.

**Проверка результатов (Измеряемость для Варианта 2):**

1.  Предоставьте ссылку на Git-репозиторий (если вы его создали на хостинге) или опишите, что используете локальный репозиторий.
2.  Предоставьте содержимое `.tf` файла (`main.tf`).
3.  Предоставьте содержимое скрипта `apply.sh` (или `.bat`).
4.  Предоставьте вывод первого запуска скрипта `./apply.sh`, где видно, что Terraform добавил ресурс (`+ local_file.hello_file`).
5.  Предоставьте коммит в Git, который изменил содержимое файла.
6.  Предоставьте вывод второго запуска скрипта `./apply.sh`, где видно, что Terraform изменил ресурс (`~ local_file.hello_file`).
7.  Предоставьте содержимое файла `hello.txt` после второго запуска скрипта, показывающее обновленное содержимое.

---

Какой бы вариант вы ни выбрали, главное - продемонстрировать, что Git является источником истины, и изменения в нем приводят к изменениям в инфраструктуре (автоматически через CI или через скрипт, который "слушает" Git).

Приступаем к следующему ДЗ по Crossplane?