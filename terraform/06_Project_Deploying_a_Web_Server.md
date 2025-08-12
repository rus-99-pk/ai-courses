# Тема 6: Проект - Развертывание полноценного веб-сервера

## Какую проблему мы решаем?

До сих пор мы создавали отдельные, изолированные ресурсы. В реальном мире инфраструктура — это взаимосвязанная система. Веб-сервер бесполезен без сети, в которой он находится, и правил брандмауэра (групп безопасности), которые разрешают к нему доступ.

### Задача: Развернуть в AWS полноценную, но минималистичную веб-инфраструктуру, состоящую из:
1.  **VPC (Virtual Private Cloud)**: Наша изолированная частная сеть.
2.  **Subnet**: Подсеть внутри VPC, где будут жить наши ресурсы.
3.  **Internet Gateway**: Шлюз для доступа в интернет.
4.  **Route Table**: Таблица маршрутизации, направляющая трафик из подсети в интернет-шлюз.
5.  **Security Group**: Виртуальный брандмауэр, разрешающий входящий трафик на порты 80 (HTTP) и 22 (SSH).
6.  **EC2 Instance**: Наш веб-сервер, который будет использовать все созданные сетевые ресурсы и запускать простой веб-сервер при старте.

## Структура проекта

Мы будем использовать модули для лучшей организации.

```
terraform-final-project/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    └── web-server/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```
*Модуль `web-server` будет содержать EC2-инстанс и Security Group, так как они тесно связаны.*
*Сетевые ресурсы (VPC, Subnet и т.д.) мы оставим в корневом модуле для простоты.*

## Шаг 1: Написание кода

### `variables.tf` (корневой)
```hcl
variable "aws_region" {
  description = "Регион AWS для развертывания"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Имя проекта, используется для тегирования"
  type        = string
  default     = "TerraformCourse"
}

variable "my_ip" {
  description = "Ваш IP-адрес для доступа по SSH. Узнать можно на 2ip.ru"
  type        = string
  # ВАЖНО: Укажите свой IP, иначе не сможете подключиться к серверу!
  # default     = "0.0.0.0/0" # Небезопасно!
}

variable "instance_type" {
  description = "Тип EC2 инстанса"
  type        = string
  default     = "t2.micro"
}
```

### `main.tf` (корневой) - Сеть и вызов модуля
```hcl
provider "aws" {
  region = var.aws_region
}

# 1. Создаем VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "${var.project_name}-vpc" }
}

# 2. Создаем подсеть
resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  map_public_ip_on_launch = true # Автоматически назначать публичный IP
  tags = { Name = "${var.project_name}-subnet" }
}

# 3. Создаем интернет-шлюз
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "${var.project_name}-igw" }
}

# 4. Создаем таблицу маршрутизации
resource "aws_route_table" "main" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0" # Весь трафик
    gateway_id = aws_internet_gateway.gw.id
  }
  tags = { Name = "${var.project_name}-rt" }
}

# 5. Привязываем таблицу маршрутизации к подсети
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.main.id
  route_table_id = aws_route_table.main.id
}

# 6. Вызываем наш модуль для создания сервера
module "web_server" {
  source = "./modules/web-server"

  # Передаем нужные параметры
  vpc_id        = aws_vpc.main.id
  subnet_id     = aws_subnet.main.id
  instance_type = var.instance_type
  project_name  = var.project_name
  my_ip         = var.my_ip
}
```

### `outputs.tf` (корневой)
```hcl
output "server_public_ip" {
  description = "Публичный IP-адрес созданного веб-сервера"
  value       = module.web_server.public_ip
}
```

---

### Код для модуля `modules/web-server/`

### `variables.tf` (модуль)
```hcl
variable "vpc_id" { type = string }
variable "subnet_id" { type = string }
variable "instance_type" { type = string }
variable "project_name" { type = string }
variable "my_ip" { type = string }
```

### `main.tf` (модуль) - Security Group и EC2
```hcl
# 1. Создаем Security Group
resource "aws_security_group" "web_sg" {
  name        = "${var.project_name}-sg"
  description = "Allow HTTP and SSH traffic"
  vpc_id      = var.vpc_id

  # Разрешаем входящий HTTP трафик отовсюду
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Разрешаем входящий SSH трафик только с вашего IP
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["${var.my_ip}/32"]
  }

  # Разрешаем весь исходящий трафик
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Ищем свежий AMI Amazon Linux 2
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# 2. Создаем EC2 инстанс
resource "aws_instance" "main" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  # user_data - это скрипт, который выполняется при первом запуске сервера
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Deployed via Terraform!</h1>" > /var/www/html/index.html
              EOF

  tags = {
    Name = "${var.project_name}-server"
  }
}
```
*`user_data` — мощный инструмент для первоначальной настройки сервера.*

### `outputs.tf` (модуль)
```hcl
output "public_ip" {
  value = aws_instance.main.public_ip
}
```

## Команда-методичка

1.  **Заполните переменную `my_ip`!**
    Создайте файл `terraform.tfvars` в корне проекта и добавьте в него свой IP:
    ```
    # terraform.tfvars
    my_ip = "ВАШ_IP_АДРЕС"
    ```
    Это критически важно для безопасности.

2.  **Инициализация**:
    ```bash
    terraform init
    ```

3.  **Применение**:
    ```bash
    terraform apply
    ```
    Просмотрите план, который создаст VPC, подсеть, шлюз, маршруты, группу безопасности и EC2-инстанс. Введите `yes`.

4.  **Проверка**:
    После завершения `apply` Terraform выведет IP-адрес вашего сервера. Скопируйте его и вставьте в адресную строку браузера. Вы должны увидеть страницу с надписью "Deployed via Terraform!".

5.  **Уничтожение**:
    Когда закончите, не забудьте удалить все ресурсы, чтобы не платить за них.
    ```bash
    terraform destroy
    ```

**Поздравляем!** Вы только что развернули с нуля полноценную облачную инфраструктуру с помощью кода, объединив все концепции, которые мы изучили.