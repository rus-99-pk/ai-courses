Отлично! Переходим к одному из самых мощных и важных аспектов DevOps — Infrastructure as Code (IaC) и Configuration Management. Это позволит вам управлять вашей инфраструктурой так же, как вы управляете кодом вашего приложения.

---

**УРОК 5: Infrastructure as Code и Системы Управления Конфигурацией**

**Цель:** Научиться описывать, развертывать и управлять вашей серверной инфраструктурой и конфигурацией операционной системы с помощью кода, используя Terraform и Ansible.

**Результат урока:** У вас будет код (в Git) для создания и базовой настройки виртуальных машин. Ваш Delivery Pipeline будет расширен, чтобы автоматически создавать или обновлять Staging VM перед деплоем приложения.

**Предварительные требования:**

*   Успешно завершен Урок 1-4.
*   Рабочий CI/CD пайплайн до этапа "Deploy to Staging" (или хотя бы "Publish Artifact") из Урока 4.
*   Доступ к гипервизору или облачному провайдеру, где вы можете создавать VM программно (через API).
*   SSH доступ по ключу с вашей рабочей машины к VM, на которой будет запускаться Terraform и Ansible (например, Jenkins VM или отдельная управляющая VM).

**Выбор Провайдера для Terraform:**

Для Lab 2 (Provision VMs with Terraform) вам понадобится провайдер. Выберите один из вариантов:

1.  **Облачный провайдер (AWS, GCP, Azure, DigitalOcean, Yandex.Cloud и др.):** **Рекомендуется**, так как это наиболее реалистичный сценарий использования Terraform. Потребуется создать аккаунт (используйте бесплатные кредиты или пробные периоды) и настроить учетные данные (ключи доступа/токены) на VM, откуда будете запускать Terraform.
2.  **Локальный гипервизор с API:**
    *   **VirtualBox:** Требует плагина Vagrant с провайдером VirtualBox или прямой работы через VBoxManage (менее типично для Terraform). Terraform имеет провайдер `virtualbox`, но его использование может быть сложным.
    *   **VMware vSphere/ESXi:** Если у вас есть доступ к такой инфраструктуре, Terraform провайдер `vsphere` очень функционален.
    *   **KVM/libvirt:** Terraform провайдер `libvirt` позволяет управлять VM на хостах с KVM. Хороший вариант, если вы используете Linux хост как гипервизор.
3.  **Vagrant:** Vagrant (с Vagrantfile) является инструментом для создания и управления локальными средами разработки (часто на VirtualBox/VMware/Libvirt). Хотя это не чистый IaC в классическом понимании (скорее Environment Management), он может выполнять похожие задачи. Terraform может интегрироваться с Vagrant.

*Для простоты в этой методичке будут даны общие шаги, которые применимы к большинству провайдеров, но вам нужно будет обратиться к документации конкретного провайдера Terraform для точных названий ресурсов и параметров.* Мы будем использовать облачный провайдер в примерах команд, как наиболее распространенный сценарий.

**Техническая подготовка (на Вашей VM, с которой будете управлять инфраструктурой - например, Jenkins VM):**

1.  **Установите Terraform:**
    ```bash
    # Для Ubuntu/Debian (рекомендуемый способ через репозиторий HashiCorp)
    sudo apt update && sudo apt install -y gnupg software-properties-common
    wget -O- https://apt.releases.hashicorp.com/gpg | \
        gpg --dearmor | \
        sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
    gpg --no-default-keyring \
        --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
        --fingerprint
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
        https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
        sudo tee /etc/apt/sources.list.d/hashicorp.list
    sudo apt update
    sudo apt install terraform -y

    # Для CentOS/RHEL/Fedora (через репозиторий HashiCorp)
    sudo dnf install -y dnf-plugins-core
    sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
    sudo dnf install terraform

    # Проверьте установку
    terraform version
    ```

2.  **Настройте учетные данные для выбранного провайдера Terraform:**
    *   **AWS:** Установите AWS CLI (`sudo apt install awscli -y` или `sudo yum install awscli -y`), настройте его (`aws configure`) или установите переменные окружения (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`).
    *   **GCP:** Установите gcloud CLI и настройте аутентификацию.
    *   **Azure:** Установите Azure CLI (`az login`).
    *   **DigitalOcean:** Создайте токен API и используйте его как переменную окружения или в конфигурации провайдера Terraform.
    *   **Libvirt:** Убедитесь, что пользователь, запускающий Terraform, имеет доступ к сокету libvirt.

3.  **Установите Ansible:** Ansible работает по SSH, поэтому устанавливается только на *управляющей* машине (например, вашей Jenkins VM или отдельной VM).
    ```bash
    # Для Ubuntu/Debian
    sudo apt update
    sudo apt install ansible -y

    # Для CentOS/RHEL/Fedora
    sudo dnf install ansible -y

    # Проверьте установку
    ansible --version
    ```
    Убедитесь, что с этой VM вы можете подключиться по SSH без пароля к Staging VM, которую создадите с помощью Terraform (т.е. публичный ключ пользователя Ansible на этой VM должен быть в `~/.ssh/authorized_keys` на Staging VM).

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **Infrastructure as Code (IaC):** Управление и Provisioning инфраструктуры (серверов, сетей, баз данных, балансировщиков) с помощью машиночитаемых файлов определения, а не интерактивных инструментов или ручной настройки. Позволяет версионировать инфраструктуру, автоматизировать ее развертывание, использовать практики CI/CD для инфраструктуры.
    *   **Идемпотентность:** Ключевой принцип IaC и Configuration Management. Применение операции несколько раз приводит к тому же результату, что и однократное применение. Важно для надежности автоматизации.
    *   **Декларативный vs Императивный:**
        *   **Декларативный (Terraform):** Вы описываете *желаемое конечное состояние* инфраструктуры. Инструмент сам определяет, какие шаги нужно выполнить, чтобы достичь этого состояния. Легче понять общую картину, но сложнее контролировать пошаговое выполнение.
        *   **Императивный (Ansible Playbooks, Bash скрипты):** Вы описываете *последовательность шагов*, которые нужно выполнить для достижения состояния. Полный контроль над выполнением, но сложнее понять конечное состояние, просто глядя на скрипт.

*   **Terraform:** Популярный декларативный IaC инструмент от HashiCorp.
    *   **HCL (HashiCorp Configuration Language):** Язык для написания Terraform кода.
    *   **Провайдеры (Providers):** Плагины, которые взаимодействуют с API различных сервисов (облака, гипервизоры, Kubernetes, DNS и т.д.). Каждый ресурс управляется провайдером.
    *   **Ресурсы (Resources):** Основные строительные блоки Terraform. Представляют компоненты инфраструктуры (виртуальная машина, сеть, правило файрвола).
    *   **Модули (Modules):** Переиспользуемые блоки Terraform кода для организации и абстракции.
    *   **Переменные (Variables):** Позволяют параметризовать конфигурацию.
    *   **State-файл (`.tfstate`):** JSON-файл, в котором Terraform хранит информацию о состоянии развернутой инфраструктуры. Это критически важно для работы Terraform (`plan`, `apply`, `destroy`). **Никогда не теряйте state-файл и не храните его локально для командной работы!** Используйте Remote State (например, с S3, GCS, Azure Blob Storage, Terraform Cloud).

*   **Системы управления конфигурацией (Configuration Management):** Инструменты для установки и настройки ПО, управления файлами, запуска сервисов на *уже существующих* серверах. Примеры: Ansible, Chef, Puppet, SaltStack.
    *   **Pull vs Push:**
        *   **Pull (Chef Client, Puppet Agent):** Агент устанавливается на управляемый сервер, периодически подключается к центральному серверу CM, скачивает свою конфигурацию и применяет ее.
        *   **Push (Ansible):** Управляющая машина подключается по SSH к управляемым серверам и выполняет команды. Не требует агента на управляемых серверах. Ansible - Push-инструмент.

*   **Ansible:** Популярный Push-инструмент управления конфигурацией.
    *   **Inventory:** Файл (INI или YAML), содержащий список управляемых серверов (хостов), сгруппированных по ролям.
    *   **Playbooks:** YAML-файлы, описывающие задачи (Tasks) для выполнения на хостах из Inventory. Задачи выполняются с помощью модулей Ansible.
    *   **Модули (Modules):** Скрипты, которые Ansible запускает на управляемых хостах для выполнения конкретных действий (установка пакета, копирование файла, запуск сервиса). Ansible имеет огромное количество встроенных модулей.
    *   **Роли (Roles):** Способ организации Playbooks и связанных файлов (переменные, шаблоны, файлы) в переиспользуемую структуру.
    *   **Vault:** Инструмент Ansible для шифрования чувствительных данных (пароли, ключи) в Playbooks или переменных.

*   **Сравнение Terraform и Ansible:**
    *   **Terraform:** Лучше подходит для Provisioning *инфраструктуры* (создание/удаление VM, сетей, дисков). Декларативный. Знает состояние инфраструктуры (`.tfstate`).
    *   **Ansible:** Лучше подходит для *конфигурации* *уже созданных* серверов (установка ПО, настройка сервисов). Императивный (обычно). Не хранит состояние инфраструктуры по умолчанию.
    *   **Часто используются вместе:** Terraform создает VM, Ansible их настраивает.

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Установка Terraform (Выполнено в Технической подготовке)**

*   Убедитесь, что Terraform установлен на вашей управляющей VM и настроены учетные данные для выбранного облачного провайдера или гипервизора.

**Lab 2: Создание VM с помощью Terraform**

*   **Цель:** Написать Terraform код для создания 1-2 виртуальных машин и научиться применять и удалять эту инфраструктуру.
*   **Предварительные требования:**
    *   Установленный Terraform.
    *   Настроенные учетные данные для провайдера.
*   **Шаги:**

    1.  **Создайте каталог для Terraform кода:**
        ```bash
        mkdir $HOME/terraform-infra
        cd $HOME/terraform-infra
        ```

    2.  **Создайте файл `main.tf`:** Этот файл будет содержать основную конфигурацию.

        ```terraform
        # main.tf

        # Определение провайдера. Замените "aws" на имя вашего провайдера (gcp, azurerm, digitalocean, libvirt и т.д.)
        # и укажите требуемые параметры (region, project, features и т.д.)
        terraform {
          required_providers {
            aws = {
              source  = "hashicorp/aws"
              version = "~> 5.0" # Используйте актуальную версию провайдера
            }
            # Пример для DigitalOcean:
            # digitalocean = {
            #  source = "digitalocean/digitalocean"
            #  version = "~> 2.0"
            # }
            # Пример для libvirt (KVM):
            # libvirt = {
            #  source = "dmacvicar/libvirt"
            #  version = "~> 0.6.0"
            # }
          }
        }

        # Конфигурация провайдера. Замените на вашу конфигурацию.
        # Для AWS, регион может быть указан здесь или через переменную окружения AWS_REGION.
        provider "aws" {
          region = "us-east-1" # Пример для AWS
        }
        # provider "digitalocean" { token = var.do_token } # Пример для DigitalOcean
        # provider "libvirt" { uri = "qemu:///system" } # Пример для libvirt

        # Определите ресурс - виртуальную машину.
        # Замените "aws_instance" на тип ресурса вашего провайдера (например, digitalocean_droplet, libvirt_domain, google_compute_instance, azurerm_linux_virtual_machine).
        # Замените параметры (ami, instance_type, key_name, tags) на соответствующие для вашего провайдера.
        resource "aws_instance" "staging_vm" {
          # Пример для AWS:
          ami           = "ami-0abcdef1234567890" # Замените на актуальный AMI ID для вашей ОС и региона
          instance_type = "t2.micro" # Выберите подходящий тип
          key_name      = "my-ssh-key" # Имя вашего SSH ключа, который должен быть загружен в облачный провайдер

          tags = {
            Name    = "staging-vm"
            Project = "my-devops-app"
          }

          # Опционально: настройка сети, файрвола (security groups в AWS)
          # vpc_security_group_ids = [aws_security_group.ssh_http_https.id]
          # subnet_id              = aws_subnet.my_subnet.id
        }

        # Пример создания второго ресурса (если нужно)
        # resource "aws_instance" "another_vm" { ... }

        # Пример создания ресурса Security Group (для AWS) для разрешения SSH/HTTP/HTTPS
        # resource "aws_security_group" "ssh_http_https" {
        #  name        = "allow_ssh_http_https"
        #  description = "Allow SSH, HTTP, HTTPS inbound traffic"
        #  vpc_id      = "vpc-abcdef" # ID вашего VPC

        #  ingress {
        #    description = "SSH from anywhere"
        #    from_port   = 22
        #    to_port     = 22
        #    protocol    = "tcp"
        #    cidr_blocks = ["0.0.0.0/0"] # ОЧЕНЬ НЕБЕЗОПАСНО! Ограничьте до своего IP или VPN.
        #  }

        #  ingress {
        #    description = "HTTP from anywhere"
        #    from_port   = 80
        #    to_port     = 80
        #    protocol    = "tcp"
        #    cidr_blocks = ["0.0.0.0/0"] # ОЧЕНЬ НЕБЕЗОПАСНО!
        #  }

        #  ingress {
        #    description = "HTTPS from anywhere"
        #    from_port   = 443
        #    to_port     = 443
        #    protocol    = "tcp"
        #    cidr_blocks = ["0.0.0.0/0"] # ОЧЕНЬ НЕБЕЗОПАСНО!
        #  }

        #  egress {
        #    from_port   = 0
        #    to_port     = 0
        #    protocol    = "-1" # All protocols
        #    cidr_blocks = ["0.0.0.0/0"]
        #  }

        #  tags = { Name = "allow_ssh_http_https" }
        # }

        # Вывод полезной информации после создания инфраструктуры
        output "staging_vm_public_ip" {
          # Замените "aws_instance.staging_vm.public_ip" на соответствующий атрибут вашего ресурса VM
          value = aws_instance.staging_vm.public_ip
        }

        output "staging_vm_private_ip" {
          # Замените "aws_instance.staging_vm.private_ip" на соответствующий атрибут вашего ресурса VM
          value = aws_instance.staging_vm.private_ip
        }
        ```
        *   **Важно:** Найдите актуальный `ami` (или аналог) для вашего провайдера и региона. Найдите, как загрузить ваш публичный SSH ключ в провайдер и как сослаться на него в конфигурации VM.

    3.  **Инициализируйте Terraform:**
        ```bash
        terraform init
        ```
        Эта команда скачает необходимые плагины провайдеров. Выполняется один раз для нового каталога или при добавлении/изменении провайдеров.

    4.  **Спланируйте изменения:**
        ```bash
        terraform plan
        ```
        Terraform проанализирует вашу конфигурацию и текущее состояние инфраструктуры (если оно есть) и покажет, какие изменения будут применены (что будет создано, изменено или удалено). Внимательно изучите вывод!

    5.  **Примените изменения для создания VM:**
        ```bash
        terraform apply
        ```
        Terraform еще раз покажет план и запросит подтверждение. Введите `yes` для выполнения. Дождитесь завершения. Terraform создаст VM и сохранит информацию о созданном ресурсе в файл `terraform.tfstate`.
        *   Если вы видите ошибки, внимательно читайте вывод. Это может быть связано с учетными данными, неправильным AMI, нехваткой ресурсов и т.д.

    6.  **Проверьте созданную VM:**
        *   Перейдите в веб-консоль вашего облачного провайдера/гипервизора. Убедитесь, что VM создана, запущена и имеет IP-адрес.
        *   Используйте выводы Terraform (`output`) для получения IP-адреса: `terraform output staging_vm_public_ip`.
        *   Попробуйте подключиться к новой VM по SSH, используя ключ, который вы указали в Terraform.

    7.  **Потренируйтесь в изменении конфигурации:**
        *   Отредактируйте `main.tf`, например, измените тег VM или тип инстанса (если провайдер позволяет менять тип без пересоздания).
        *   Снова выполните `terraform plan`. Terraform покажет, какие изменения он планирует применить к *существующему* ресурсу.
        *   Выполните `terraform apply`.

    8.  **Удалите инфраструктуру:**
        ```bash
        terraform destroy
        ```
        Terraform покажет план удаления. Введите `yes` для подтверждения. Terraform удалит VM и все связанные ресурсы, определенные в `main.tf`.

    9.  **(Важно) Настройте Remote State:** **Не оставляйте state-файл `.tfstate` локально!** Это опасно. Настройте его хранение в надежном месте (например, S3 бакет в AWS, GCS бакет в GCP, Terraform Cloud). Инструкции зависят от провайдера Backend. Пример для S3:

        *   Создайте S3 бакет (вручную или другим скриптом Terraform).
        *   Добавьте секцию `backend` в ваш `main.tf` (внутри блока `terraform`):
            ```terraform
            terraform {
              required_providers {
                # ...
              }
              backend "s3" {
                bucket = "my-terraform-state-bucket-unique-name" # Уникальное имя вашего S3 бакета
                key    = "my-devops-app/staging/terraform.tfstate" # Путь к файлу состояния в бакете
                region = "us-east-1" # Регион бакета
                # Опционально: dynamodb_table для блокировки состояния при параллельных запусках
              }
            }
            ```
        *   Выполните `terraform init`. Terraform предложит смигрировать существующий локальный state в новый backend. Согласитесь (`yes`). Теперь state-файл будет храниться удаленно.

**Результат Lab 2:** Умение описывать базовую инфраструктуру (VM) с помощью Terraform, применять эти определения для создания/изменения/удаления ресурсов и управлять состоянием инфраструктуры.

---

**Lab 3: Установка Ansible и Inventory**

*   **Цель:** Установить Ansible и создать список серверов для управления.
*   **Предварительные требования:**
    *   Установленный Ansible (Выполнено в Технической подготовке).
    *   Хотя бы одна Linux VM, к которой можно подключиться по SSH по ключу с управляющей машины (можно использовать Staging VM, созданную вручную или с помощью Terraform, или любую другую VM).

*   **Шаги:**

    1.  **Создайте каталог для Ansible Playbooks:**
        ```bash
        mkdir $HOME/ansible-config
        cd $HOME/ansible-config
        ```

    2.  **Создайте файл Inventory:** По умолчанию Ansible ищет `/etc/ansible/hosts` или вы можете указать свой файл с помощью опции `-i`. Создадим свой файл в формате INI.
        ```bash
        nano inventory
        ```
        ```ini
        [staging] # Имя группы серверов
        IP_АДРЕС_STAGING_VM ansible_user=ваше_имя_пользователя_на_staging_vm ansible_ssh_private_key_file=~/.ssh/id_rsa # Пример для SSH ключа

        [all:vars] # Переменные, применимые ко всем хостам
        ansible_python_interpreter=/usr/bin/python3 # Убедитесь, что путь к Python3 правильный на ваших VM
        ```
        *   Замените `IP_АДРЕС_STAGING_VM` и `ваше_имя_пользователя_на_staging_vm` на реальные данные.
        *   Убедитесь, что `ansible_ssh_private_key_file` указывает на правильный путь к приватному ключу на вашей управляющей машине. Если ваш SSH клиент настроен использовать ключ по умолчанию (`~/.ssh/id_rsa`) и агент SSH запущен, эту строку можно опустить.

    3.  **Проверьте связь с хостами:** Используйте модуль `ping`.
        ```bash
        ansible -i inventory staging -m ping
        ```
        Вы должны увидеть ответ `pong` от вашей Staging VM. Если нет, проверьте SSH доступ, IP-адрес, имя пользователя, файрвол на Staging VM (должен разрешать SSH).

**Результат Lab 3:** Установлен Ansible и создан базовый Inventory файл, позволяющий управлять вашей Staging VM.

---

**Lab 4: Написание Базового Ansible Playbook**

*   **Цель:** Написать Playbook для выполнения стандартных задач по настройке сервера.
*   **Предварительные требования:**
    *   Работающий Ansible с Inventory (Lab 3).
    *   Staging VM.
*   **Шаги:**

    1.  **Создайте первый Playbook файл:**
        ```bash
        nano playbook_basics.yml
        ```

    2.  **Напишите Playbook:**

        ```yaml
        # playbook_basics.yml

        - name: Basic server setup and package management
          hosts: staging # Применить этот playbook к группе хостов 'staging' из inventory
          become: yes # Выполнять задачи с повышенными привилегиями (как root, используя sudo)
          vars:
            java_version: 11 # Пример переменной

          tasks:
            - name: Update all packages # Задача: Обновить пакеты
              # Используем модуль package, который работает с apt/yum/dnf в зависимости от дистрибутива
              package:
                name: "*" # Обновить все установленные пакеты
                state: latest # Установить последнюю версию
                update_cache: yes # Обновить кэш репозиториев перед обновлением

            - name: Install Java Development Kit (JDK) # Задача: Установить JDK
              package:
                name: "openjdk-{{ java_version }}-jdk" # Имя пакета, используем переменную
                state: present # Убедиться, что пакет установлен

            - name: Install Git # Задача: Установить Git
              package:
                name: git
                state: present

            - name: Install Nginx # Задача: Установить Nginx (веб-сервер, пригодится позже)
              package:
                name: nginx # Или nginx-full, httpd для CentOS
                state: present

            - name: Copy a simple file # Задача: Скопировать файл на управляемый хост
              copy:
                src: files/hello.txt # Локальный файл на управляющей машине Ansible
                dest: /tmp/hello_ansible.txt # Путь на управляемом хосте
                owner: ваше_имя_пользователя_на_staging_vm # Убедитесь, что пользователь существует
                group: ваше_имя_пользователя_на_staging_vm
                mode: '0644' # Права доступа

            - name: Ensure Nginx service is running and enabled # Задача: Убедиться, что сервис Nginx запущен и стартует при загрузке
              service:
                name: nginx # Имя сервиса (или httpd для CentOS/RHEL)
                state: started # Убедиться, что он запущен
                enabled: yes # Убедиться, что он стартует при загрузке

            # Пример задачи для остановки сервиса
            # - name: Ensure Nginx service is stopped
            #  service:
            #    name: nginx
            #    state: stopped
        ```
        *   **Важно:** Замените `ваше_имя_пользователя_на_staging_vm` на реальное имя пользователя. Убедитесь, что имя пакета для JDK и Nginx соответствует вашему дистрибутиву Linux на Staging VM.
        *   Создайте каталог `files` рядом с playbook и положите туда файл `hello.txt`.
            ```bash
            mkdir files
            echo "Hello from Ansible!" > files/hello.txt
            ```

    3.  **Выполните Playbook:** Используйте опцию `-i` для указания вашего inventory файла.
        ```bash
        ansible-playbook -i inventory playbook_basics.yml
        ```
        Ansible подключится к Staging VM, выполнит все задачи по порядку. Вы увидите вывод о каждом шаге (changed, ok, failed). Идемпотентность Ansible означает, что повторный запуск playbook не будет выполнять действия, если желаемое состояние уже достигнуто (например, пакет уже установлен).

    4.  **Проверьте изменения на Staging VM:** Подключитесь по SSH к Staging VM и проверьте:
        *   Обновлены ли пакеты?
        *   Установлены ли Java, Git, Nginx?
        *   Есть ли файл `/tmp/hello_ansible.txt` с правильным содержимым и правами?
        *   Запущен ли процесс Nginx (`sudo systemctl status nginx` или `ps aux | grep nginx`) и включен ли автозапуск (`sudo systemctl is-enabled nginx`)?

**Результат Lab 4:** Умение писать базовые Ansible Playbooks для выполнения типовых задач по управлению конфигурацией серверов.

---

**Lab 5: Полная Настройка VM с Ansible для Приложения**

*   **Цель:** Написать Playbook, который полностью готовит Staging VM для деплоя вашего Java приложения.
*   **Предварительные требования:**
    *   Работающий Ansible с Inventory (Lab 3).
    *   Staging VM.
    *   Понимание, какие зависимости нужны вашему приложению (JDK, возможно, другие библиотеки), какой пользователь его запускает, куда нужно скопировать файлы, какие каталоги создать.
*   **Шаги:**

    1.  **Создайте новый Playbook для настройки приложения:**
        ```bash
        nano playbook_app_setup.yml
        ```

    2.  **Напишите Playbook:**

        ```yaml
        # playbook_app_setup.yml

        - name: Setup Staging VM for Java Application Deployment
          hosts: staging
          become: yes
          vars:
            app_user: devopsuser # Пользователь, от имени которого будет работать приложение
            app_group: devopsuser
            app_base_dir: /opt/myjavaapp # Базовый каталог для приложения
            app_log_dir: /var/log/myjavaapp # Каталог для логов

          tasks:
            - name: Ensure application user and group exist
              user:
                name: "{{ app_user }}"
                group: "{{ app_group }}"
                state: present # Убедиться, что пользователь и группа существуют
                createhome: no # Не создавать домашний каталог, если не нужно

            - name: Ensure application directories exist
              file:
                path: "{{ item }}"
                state: directory # Убедиться, что это каталог
                owner: "{{ app_user }}"
                group: "{{ app_group }}"
                mode: '0755' # Права на каталоги
              loop:
                - "{{ app_base_dir }}"
                - "{{ app_log_dir }}"
                # Добавьте другие необходимые каталоги (например, для конфигов)

            - name: Install necessary dependencies (JDK)
              package:
                name: "openjdk-11-jre" # Установите только JRE, если не нужна сборка на сервере
                state: present

            # Опционально: скопировать файл сервиса systemd для запуска приложения
            # Это более правильный способ запуска приложения, чем nohup &
            # - name: Copy systemd service file
            #  copy:
            #    src: files/myjavaapp.service # Файл сервиса на управляющей машине Ansible
            #    dest: /etc/systemd/system/myjavaapp.service
            #    owner: root
            #    group: root
            #    mode: '0644'
            #  notify: # Уведомить хэндлер после изменения файла
            #    - reload systemd daemon
            #    - restart myjavaapp service # Если сервис уже был запущен

            # Опционально: настроить логирование (например, установить rsyslog/journald forwarder)
            # - name: Setup log forwarding
            #  ...

            # Опционально: скопировать файлы конфигурации приложения
            # - name: Copy application configuration files
            #  copy:
            #    src: files/application.properties.j2 # Используем шаблон Jinja2
            #    dest: "{{ app_base_dir }}/application.properties"
            #    owner: "{{ app_user }}"
            #    group: "{{ app_group }}"
            #    mode: '0644'
            #    # Переменные для шаблона можно определить в vars или host_vars/group_vars
            #    # vars:
            #    #  db_host: 192.168.1.100
            #    #  db_port: 5432
            #    #  db_name: appdb

          # Хэндлеры (выполняются, если какая-либо задача с notify их вызвала)
          # handlers:
          #  - name: reload systemd daemon
          #    systemd:
          #      daemon_reload: yes
          #  - name: restart myjavaapp service
          #    service:
          #      name: myjavaapp
          #      state: restarted
        ```
        *   **Важно:** Определите все зависимости, которые нужны *на сервере* для запуска вашего приложения. Возможно, не только JDK, но и какие-то нативные библиотеки.
        *   Если вы решили использовать service unit для systemd, создайте файл `files/myjavaapp.service` на вашей управляющей машине Ansible. Пример содержимого `myjavaapp.service`:

            ```ini
            [Unit]
            Description=My Java Application
            After=network.target

            [Service]
            User=devopsuser
            Group=devopsuser
            WorkingDirectory=/opt/myjavaapp
            ExecStart=/usr/bin/java -jar /opt/myjavaapp/my-app-1.0-SNAPSHOT.jar # Укажите полный путь к java и вашему jar
            Restart=on-failure
            # Переменные окружения для приложения, если оно их использует (Twelve-Factor App!)
            # Environment="SPRING_DATASOURCE_URL=jdbc:postgresql://dbhost:5432/appdb"
            # Environment="APP_SECRET=..."

            [Install]
            WantedBy=multi-user.target
            ```
        *   Если вы используете файлы конфигурации, которые зависят от окружения (Staging/Prod), используйте шаблоны Jinja2 (`.j2`) и передавайте переменные из Ansible.

    3.  **Выполните Playbook:**
        ```bash
        ansible-playbook -i inventory playbook_app_setup.yml
        ```

    4.  **Проверьте настройку на Staging VM:** Подключитесь по SSH и убедитесь, что пользователь/группа созданы, каталоги существуют с правильными правами, JDK установлен. Если использовали service unit, проверьте его наличие (`ls /etc/systemd/system/myjavaapp.service`) и попробуйте включить его (`sudo systemctl enable myjavaapp`) (он пока не запустится, т.к. нет JAR файла).

**Результат Lab 5:** Умение использовать Ansible для автоматической подготовки сервера под конкретное приложение, включая создание пользователей, каталогов и установку зависимостей.

---

**Lab 6: Интеграция IaC и Configuration Management в Pipeline**

*   **Цель:** Добавить в Jenkins Pipeline (или GitLab CI) этапы создания и настройки Staging VM с помощью Terraform и Ansible *перед* деплоем приложения.
*   **Предварительные требования:**
    *   Работающий CI/CD пайплайн (Jenkinsfile или `.gitlab-ci.yml`).
    *   Terraform код для создания VM (Lab 2).
    *   Ansible Inventory и Playbook для настройки VM (Lab 3 и 5).
    *   Настроенный SSH доступ по ключу с Jenkins VM (или Runner VM) к *будущей* Staging VM (на основе ключа, указанного в Terraform).
    *   Каталоги `$HOME/terraform-infra` и `$HOME/ansible-config` (или где у вас лежат эти файлы) доступны пользователю Jenkins/Runner. **Лучше хранить этот код в отдельном Git репозитории (например, `infrastructure-repo`) и клонировать его в начале пайплайна.**

*   **Шаги (для Jenkinsfile, адаптация для GitLab CI аналогична):**

    1.  **Создайте или используйте отдельный Git репозиторий для Infrastructure Code:** Это хорошая практика.
        *   Создайте новый репозиторий в Gitea/GitLab (например, `infrastructure-repo`).
        *   Переместите каталоги `terraform-infra` и `ansible-config` туда.
        *   Запушьте их в новый репозиторий.

    2.  **Обновите ваш Jenkinsfile:** Вам нужно добавить новый этап в начале пайплайна.

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any

            // Переменные окружения (если нужны для доступа к провайдеру)
            environment {
                // Пример для AWS Credentials ID из Jenkins
                // AWS_ACCESS_KEY_ID = credentials('aws-key-id')
                // AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
                // AWS_REGION = 'us-east-1'

                // Путь к вашим Infra-кодам в рабочем пространстве Jenkins после клонирования
                INFRA_CODE_DIR = 'infrastructure-repo'
            }

            stages {

                // НОВЫЙ ЭТАП: Clone Infrastructure Code
                stage('Clone Infra Code') {
                    steps {
                        echo 'Клонирование репозитория инфраструктуры...'
                        // Убедитесь, что у Jenkins есть доступ к этому репозиторию (Credentials)
                        git url: 'http://ВАШ_IP_VM_GITEA:3000/MyDevOpsOrg/infrastructure-repo.git',
                            branch: 'main' // Или develop, или ветка, где хранится ваш IaC/CM код
                        // Рабочее пространство будет содержать два репозитория: my-app и infrastructure-repo
                    }
                }

                // НОВЫЙ ЭТАП: Provision Infrastructure (Terraform)
                stage('Provision Infrastructure') {
                    steps {
                        echo 'Запуск Terraform для создания/обновления Staging VM...'
                        dir("${INFRA_CODE_DIR}/terraform-infra") { // Переходим в каталог с кодом Terraform
                            sh 'terraform init' // Инициализация, если не было Remote State
                            sh 'terraform apply -auto-approve' // Применение изменений без запроса подтверждения
                            // -auto-approve ОПАСНО! В production лучше использовать terraform plan > plan.out и terraform apply plan.out
                        }
                        echo 'Terraform завершен.'
                    }
                }

                // НОВЫЙ ЭТАП: Wait for SSH (Убедиться, что VM доступна по SSH)
                stage('Wait for SSH') {
                    steps {
                        echo 'Ожидание доступности Staging VM по SSH...'
                        script {
                            // Получить IP-адрес из output Terraform
                            def stagingVmIp = sh(returnStdout: true, script: "cd ${INFRA_CODE_DIR}/terraform-infra && terraform output -raw staging_vm_public_ip").trim()
                            env.STAGING_SSH_HOST = stagingVmIp // Сохранить IP в переменной окружения

                            // Подождать, пока SSH порт станет доступным.
                            // Есть плагины Jenkins для этого (например, SSH Steps), или использовать bash скрипт.
                            // Пример простого bash скрипта (убедитесь, что ssh клиент установлен на агенте Jenkins):
                            sh """
                              TIMEOUT=300 # 5 минут таймаут
                              INTERVAL=5 # Проверять каждые 5 секунд
                              ELAPSED=0
                              echo "Waiting for SSH on ${STAGING_SSH_HOST}:22"
                              while ! ssh -o ConnectTimeout=5 -o BatchMode=yes -o StrictHostKeyChecking=no ${env.STAGING_SSH_USER}@${STAGING_SSH_HOST} 'exit 0'; do
                                echo "SSH not ready yet, waiting..."
                                sleep \$INTERVAL
                                ELAPSED=\$((ELAPSED + INTERVAL))
                                if [ \$ELAPSED -ge \$TIMEOUT ]; then
                                  echo "Timeout waiting for SSH!"
                                  exit 1
                                fi
                              done
                              echo "SSH is ready."
                            """
                            // env.STAGING_SSH_USER должен быть определен (например, как глобальная переменная Jenkins)
                        }
                    }
                }

                // НОВЫЙ ЭТАП: Configure VM (Ansible)
                stage('Configure VM') {
                    steps {
                        echo 'Запуск Ansible для настройки Staging VM...'
                        dir("${INFRA_CODE_DIR}/ansible-config") { // Переходим в каталог с кодом Ansible
                            // Обновить inventory с IP-адресом новой VM
                            // В реальной жизни, inventory может генерироваться динамически (например, из облачного провайдера)
                            // Для этой lab, можно просто создать временный inventory файл или передать хост как параметр
                            // Простой вариант: передать IP как переменную --extra-vars и указать hosts: all в playbook
                            // Или создать временный inventory файл
                            sh """
                              echo "[staging]" > inventory_temp
                              echo "${env.STAGING_SSH_HOST} ansible_user=${env.STAGING_SSH_USER}" >> inventory_temp
                              ansible-playbook -i inventory_temp playbook_app_setup.yml
                            """
                            # env.STAGING_SSH_USER должен быть определен (например, как глобальная переменная Jenkins)
                        }
                        echo 'Ansible завершен.'
                    }
                }

                // СУЩЕСТВУЮЩИЙ ЭТАП: Deploy to Staging
                stage('Deploy to Staging') {
                    steps {
                        echo "Запуск деплоя версии ${env.APP_VERSION} на Staging (${env.STAGING_SSH_HOST})..."
                        // Теперь скрипт деплоя может использовать env.STAGING_SSH_HOST
                        sh "$HOME/deployment_scripts/deploy_app_to_staging.sh ${env.APP_VERSION}"
                        echo 'Деплой на Staging завершен.'
                    }
                }

                // ... (Manual Approval, Rollback Job - не часть основного пайплайна) ...
            }

            post {
                always { echo 'Пайплайн завершен.' }
                success { echo 'Пайплайн выполнен успешно! 🎉' ; junit '**/target/surefire-reports/*.xml' }
                failure { echo 'Пайплайн завершился с ошибкой! 💔' }
            }
        }
        ```

    3.  **Настройте Credential для Infrastructure Repo в Jenkins:** Если вы вынесли Infra-код в отдельный приватный репозиторий, Jenkins должен иметь Credential для его клонирования (аналогично Credential для репозитория с кодом приложения).
    4.  **Настройте глобальные переменные в Jenkins:** Перейдите "Manage Jenkins" -> "Configure System" -> "Global properties" -> "Environment variables". Добавьте переменные типа `STAGING_SSH_USER` с именем пользователя на Staging VM.
    5.  **Сделайте коммит и пуш Jenkinsfile:**
        ```bash
        git add Jenkinsfile
        git commit -m "ci: Add infrastructure provisioning and configuration stages"
        git push origin develop
        ```
    6.  **Запустите пайплайн и проверьте выполнение:**
        *   Запустите пайплайн в Jenkins.
        *   Смотрите логи новых этапов: "Clone Infra Code", "Provision Infrastructure", "Wait for SSH", "Configure VM", "Deploy to Staging".
        *   Убедитесь, что Terraform создает VM, что Ansible успешно настраивает ее, и что затем приложение деплоится на эту новую VM.
        *   После успешного выполнения пайплайна, проверьте созданную VM в облаке/гипервизоре и подключитесь к ней, чтобы убедиться, что она настроена правильно и приложение запущено.
        *   Попробуйте запустить пайплайн еще раз, не удаляя VM. Terraform должен определить, что VM уже существует, и не пересоздавать ее (`0 added, 0 changed, 0 destroyed`). Ansible также должен быть идемпотентным.

**Результат Lab 6:** Ваш Delivery Pipeline стал гораздо более мощным! Он теперь полностью автоматизирует не только сборку и деплой приложения, но и подготовку инфраструктуры, на которую оно деплоится. Это ключевой шаг к полному Continuous Delivery.

---

**Резюме Урока 5:**

Вы освоили принципы Infrastructure as Code и Configuration Management, научились использовать Terraform для декларативного описания и управления инфраструктурой, и Ansible для императивной настройки серверов. Вы успешно интегрировали эти инструменты в ваш CI/CD пайплайн, создав автоматизированный процесс подготовки окружения и деплоя на него.

Теперь вы можете разворачивать не только приложение, но и серверы для него по требованию, что является важным навыком Middle+/Senior DevOps инженера.

**Что дальше?**

Следующий урок будет посвящен DBOps — применению принципов DevOps к базам данных, управлению миграциями и работе с различными типами БД (реляционными и нереляционными). Это закроет еще один важный пробел в автоматизации конвейера доставки.

Готовьтесь к работе с данными!