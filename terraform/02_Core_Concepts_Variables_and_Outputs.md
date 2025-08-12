# Тема 2: Основные концепции: Провайдеры, Переменные и Выводы

## Какую проблему мы решаем?

В прошлом уроке мы жестко "зашили" все значения в коде. Это неудобно. Что, если мы хотим легко менять имя файла или его содержимое? Что, если мы хотим получить информацию о созданном ресурсе, например, его ID или IP-адрес?

### Задача: Создать EC2-инстанс (виртуальный сервер) на AWS с настраиваемыми параметрами и получить его публичный IP-адрес после создания.

## Концепция 1: Провайдеры (Providers)

Провайдер — это плагин, который учит Terraform общаться с API конкретного сервиса (AWS, Azure, Yandex.Cloud и т.д.).

Мы должны настроить провайдер, чтобы Terraform знал, куда подключаться и какие учетные данные использовать.

**Настройка AWS Provider:**
Предполагается, что у вас уже настроены учетные данные AWS через переменные окружения или файл `~/.aws/credentials`. Это самый безопасный способ.
```bash
export AWS_ACCESS_KEY_ID="YOUR_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_SECRET_KEY"
export AWS_REGION="us-east-1" # Например
```

## Концепция 2: Переменные (Variables)

Переменные позволяют сделать ваш код гибким и переиспользуемым. Они объявляются в блоке `variable`.

**Типы переменных:**
-   `string`: `"текст"`
-   `number`: `123`
-   `bool`: `true`/`false`
-   `list(...)`: `["a", "b"]`
-   `map(...)`: `{ key = "value" }`

## Концепция 3: Выводы (Outputs)

Выводы используются, чтобы извлечь информацию о вашей инфраструктуре после ее создания. Например, IP-адрес сервера или DNS-имя базы данных. Они объявляются в блоке `output`.

## Практика: Создаем EC2-инстанс

Создайте 3 файла в новой директории: `main.tf`, `variables.tf`, `outputs.tf`.

### `variables.tf` — Описание наших переменных
```hcl
# variables.tf

variable "instance_type" {
  description = "Тип EC2 инстанса"
  type        = string
  default     = "t2.micro"
}

variable "instance_name" {
  description = "Имя для тега Name EC2 инстанса"
  type        = string
  default     = "MyFirstTerraformServer"
}
```

### `main.tf` — Основной код
```hcl
# main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

# Настройка AWS провайдера
provider "aws" {
  region = "us-east-1" # Выберите ваш регион
}

# Ищем самый свежий образ Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Создаем EC2 инстанс
resource "aws_instance" "web_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type # Используем переменную

  tags = {
    Name = var.instance_name # Используем переменную
  }
}
```

### `outputs.tf` — Что мы хотим узнать
```hcl
# outputs.tf

output "instance_id" {
  description = "ID созданного EC2 инстанса"
  value       = aws_instance.web_server.id
}

output "instance_public_ip" {
  description = "Публичный IP-адрес EC2 инстанса"
  value       = aws_instance.web_server.public_ip
}
```
*Примечание: `data "aws_ami"` — это специальный тип ресурса "data source", который не создает, а читает информацию из AWS.*

## Команда-методичка

1.  **Инициализация**:
    ```bash
    terraform init
    ```

2.  **Планирование**:
    ```bash
    terraform plan
    ```

3.  **Применение**:
    ```bash
    terraform apply
    ```
    Введите `yes`. После выполнения Terraform создаст EC2-инстанс и в конце выведет блок `Outputs` с его ID и публичным IP.

4.  **Передача переменных**:
    Вы можете переопределить значения по умолчанию при запуске. Создайте файл `terraform.tfvars`:
    ```hcl
    # terraform.tfvars
    instance_type = "t3.small"
    instance_name = "MyStagingServer"
    ```
    Terraform автоматически подхватит значения из этого файла. Запустите `terraform apply` еще раз, и вы увидите, что Terraform планирует *изменить* тип инстанса.

    Либо можно передать через командную строку:
    ```bash
    terraform apply -var="instance_type=t2.nano"
    ```

5.  **Уничтожение**:
    ```bash
    terraform destroy
    ```

Теперь ваш код стал гибким, и вы можете легко получать нужную информацию о созданных ресурсах.