# Тема 4: Модули для переиспользования кода

## Какую проблему мы решаем?

Наш код для создания EC2-инстанса становится все сложнее. Мы добавили AMI, тип инстанса, теги. А если нам понадобится добавить группу безопасности (Security Group), эластичный IP-адрес (Elastic IP) и еще что-то? Конфигурация одного сервера может занять десятки строк.

Теперь представьте, что вам нужно создать три таких сервера: для разработки (`dev`), тестирования (`staging`) и продакшена (`prod`). Копировать и вставлять этот большой блок кода три раза — ужасная идея. Если понадобится внести изменение (например, обновить AMI), придется менять его в трёх местах.

## Решение: Модули (Modules)

**Модуль** в Terraform — это набор `.tf` файлов в отдельной директории, который представляет собой логически сгруппированный набор ресурсов. Модули — это основной способ создания переиспользуемых и компонуемых блоков инфраструктуры.

**Преимущества модулей:**
-   **DRY (Don't Repeat Yourself)**: Вы пишете код один раз и используете его многократно.
-   **Инкапсуляция**: Модуль скрывает сложность реализации. Вы просто передаете ему нужные параметры (как в функцию), а он создает все необходимые ресурсы.
-   **Организация кода**: Код становится чище и структурированнее.

### Задача: Превратить наш код для создания веб-сервера в переиспользуемый модуль и с его помощью создать два разных EC2-инстанса: `staging-server` и `prod-server`.

## Шаг 1: Создание структуры проекта

Создайте следующую структуру директорий и файлов:

```
terraform-modules-project/
├── main.tf              # "Корневой" модуль, который будет вызывать наш модуль
├── outputs.tf
└── modules/
    └── web-server/
        ├── main.tf      # Основная логика модуля (создание EC2, SG и т.д.)
        ├── variables.tf # Входные переменные для модуля
        └── outputs.tf   # Выходные данные модуля (IP-адрес, ID инстанса)
```

## Шаг 2: Написание кода модуля `web-server`

Перенесем нашу логику создания сервера в папку `modules/web-server/`.

### `modules/web-server/variables.tf`
Здесь мы описываем, какие параметры можно передавать в наш модуль.
```hcl
variable "instance_type" {
  description = "Тип EC2 инстанса"
  type        = string
  default     = "t2.micro"
}

variable "server_name" {
  description = "Имя для тега Name EC2 инстанса"
  type        = string
}
```

### `modules/web-server/main.tf`
Основная логика. Обратите внимание, что мы используем переменные, объявленные выше.
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  tags = {
    Name = var.server_name
  }
}
```

### `modules/web-server/outputs.tf`
Что наш модуль будет "возвращать" наружу.
```hcl
output "instance_id" {
  description = "ID созданного EC2 инстанса"
  value       = aws_instance.app_server.id
}

output "public_ip" {
  description = "Публичный IP-адрес EC2 инстанса"
  value       = aws_instance.app_server.public_ip
}
```

## Шаг 3: Использование модуля в корневом `main.tf`

Теперь в корневом файле `main.tf` мы можем вызывать наш модуль, как будто это обычный ресурс.

### `terraform-modules-project/main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Вызываем наш модуль для создания staging-сервера
module "staging_server" {
  source = "./modules/web-server" # Путь к модулю

  # Передаем переменные в модуль
  instance_type = "t2.micro"
  server_name   = "Staging Web Server"
}

# Вызываем этот же модуль второй раз для prod-сервера
module "prod_server" {
  source = "./modules/web-server"

  instance_type = "t3.small"
  server_name   = "Production Web Server"
}
```

### `terraform-modules-project/outputs.tf`
Мы можем использовать выводы из наших модулей.
```hcl
output "staging_server_ip" {
  description = "IP адрес staging сервера"
  value       = module.staging_server.public_ip
}

output "prod_server_ip" {
  description = "IP адрес production сервера"
  value       = module.prod_server.public_ip
}
```
Синтаксис: `module.<имя_модуля>.<имя_вывода>`.

## Команда-методичка

1.  **Инициализация**:
    Когда вы добавляете новые модули, `init` обязателен.
    ```bash
    terraform init
    ```
    *Ожидаемый результат*: Terraform обнаружит и "установит" локальные модули. Вы увидите сообщения о `staging_server` и `prod_server`.

2.  **Планирование**:
    ```bash
    terraform plan
    ```
    *Ожидаемый результат*: План покажет создание **двух** EC2-инстансов (`module.staging_server.aws_instance.app_server` и `module.prod_server.aws_instance.app_server`) с разными параметрами. `Plan: 2 to add, 0 to change, 0 to destroy.`

3.  **Применение**:
    ```bash
    terraform apply
    ```
    *Ожидаемый результат*: Будут созданы два сервера, и в конце вы увидите `Outputs` с IP-адресами обоих.

Теперь вы можете легко создавать любое количество серверов с одинаковой конфигурацией, просто добавляя новый блок `module` и меняя параметры. Ваш код стал чистым, масштабируемым и простым в поддержке.