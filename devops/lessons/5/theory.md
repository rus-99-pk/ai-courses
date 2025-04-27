*   **Урок 5: Infrastructure as Code и Системы Управления Конфигурацией**
    *   **Цель:** Научиться описывать и управлять инфраструктурой и конфигурацией серверов с помощью кода.
    *   **Ключевые Темы:**
        *   Infrastructure as Code (IaC): Принципы, преимущества, идемпотентность, декларативный vs императивный подход.
        *   Terraform: Язык HCL, провайдеры (AWS, Azure, GCP, vSphere, VirtualBox), ресурсы, модули, переменные, state-файл (`.tfstate`).
        *   Системы управления конфигурацией (Configuration Management): Принципы, pull vs push модели.
        *   Ansible: Архитектура, Inventory, Playbooks (YAML), Модули, Роли, Задачи (Tasks), Переменные, Vault (для секретов в Ansible).
        *   Сравнение IaC и Config Management: Terraform vs Ansible в их типичных ролях.
    *   **Практические Задачи (Hands-on Labs):**
        1.  **Terraform Setup:** Установить Terraform. Выбрать провайдер (например, `local` для файловой системы, `libvirt` для KVM, `vsphere`, или облачный провайдер типа AWS/GCP/Azure).
        2.  **Provision VMs with Terraform:** Написать Terraform код для создания 1-2 виртуальных машин (или инстансов в облаке). Использовать переменные для настройки. Выполнить `terraform init`, `terraform plan`, `terraform apply`. Потренироваться в изменении конфигурации и применении изменений. Удалить инфраструктуру (`terraform destroy`).
        3.  **Ansible Setup:** Установить Ansible. Создать Inventory файл с вашими VMs.
        4.  **Basic Ansible Playbook:** Написать простой Ansible Playbook для:
            *   Обновления пакетов на VM.
            *   Установки нужного ПО (например, Java, Git, Nginx).
            *   Копирования файла на VM.
            *   Запуска/остановки сервиса.
        5.  **Configure Application VM with Ansible:** Написать Ansible Playbook для полной настройки одной из VM, созданных Terraform, для запуска вашего приложения (установка всех зависимостей, создание пользователя, настройка директорий).
        6.  **Integrate IaC/CM into Pipeline:** Добавить в Delivery Pipeline новый Stage (`Provision Infrastructure`). В этом Stage:
            *   Запустить Terraform для создания/обновления VM(s).
            *   Добавить задержку или проверку доступности по SSH.
            *   Запустить Ansible Playbook для настройки VM(s).
            *   *Затем* выполнить деплой приложения на настроенную VM (как в Уроке 4).
    *   **Результат:** Умение описывать и разворачивать инфраструктуру и настраивать сервера с помощью Terraform и Ansible. Пайплайн, который *сначала* готовит окружение, а *затем* деплоит на него приложение.