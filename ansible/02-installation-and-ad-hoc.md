# Тема 2: Установка и Ad-Hoc команды

## Задача, которую мы решаем

Прежде чем писать сложные сценарии, нам нужно проверить, что Ansible работает и может связываться с нашими серверами.

**Проблема:** Как быстро проверить доступность всех серверов из группы `webservers`? Как узнать, сколько на них свободного места, не заходя на каждый по SSH?

**Решение (Ansible Ad-Hoc):**
Использовать короткие одноразовые команды (ad-hoc) для выполнения простых задач без написания плейбуков.

---

### Шаг 1: Подготовка окружения

**На Control Node (ваш компьютер):**

1.  **Установка Ansible.**
    *   Для Ubuntu/Debian:
        ```bash
        sudo apt update
        sudo apt install software-properties-common
        sudo add-apt-repository --yes --update ppa:ansible/ansible
        sudo apt install ansible
        ```
    *   Для CentOS/RHEL:
        ```bash
        sudo yum install epel-release
        sudo yum install ansible
        ```
    *   Проверяем установку:
        ```bash
        ansible --version
        ```

2.  **Настройка SSH-ключа.**
    Ansible по умолчанию использует SSH-ключи для аутентификации. Это безопаснее и удобнее паролей.
    *   Если у вас еще нет ключа, создайте его:
        ```bash
        ssh-keygen -t rsa -b 4096
        ```
    *   Скопируйте ваш публичный ключ на все **Managed Nodes**:
        ```bash
        # Замените user и managed-node-ip на ваши данные
        ssh-copy-id user@managed-node-ip
        ```
    *   Проверьте, что можете зайти на сервер без пароля:
        ```bash
        ssh user@managed-node-ip
        ```

---

### Шаг 2: Создание файла Inventory

Inventory — это список ваших серверов.

1.  Создайте файл `inventory.ini` в вашей рабочей директории:
    ```ini
    [webservers]
    server1 ansible_host=192.168.1.10
    server2 ansible_host=192.168.1.11

    [dbservers]
    db1 ansible_host=192.168.1.20

    [all:vars]
    ansible_user=ubuntu
    ansible_python_interpreter=/usr/bin/python3
    ```
    *   `[webservers]` — это имя группы.
    *   `server1` — это псевдоним хоста.
    *   `ansible_host` — его IP-адрес.
    *   `[all:vars]` — переменные, общие для всех хостов. `ansible_user` — имя пользователя для SSH-подключения.

---

### Шаг 3: Выполнение Ad-Hoc команд

Теперь самое интересное. Будем давать команды Ansible прямо из терминала.

**Синтаксис:** `ansible <хост-паттерн> -i <файл-inventory> -m <модуль> -a "<аргументы модуля>"`

**Пример 1: Проверить доступность всех серверов**
Используем модуль `ping`. Он не имеет отношения к ICMP ping, а просто проверяет, может ли Ansible подключиться к хосту и выполнить модуль.

```bash
# Методичка
ansible all -i inventory.ini -m ping
```

**Ожидаемый вывод:**
```
server1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
server2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
# ... и так для всех хостов
```
Если вы видите `SUCCESS` и `"ping": "pong"`, значит, все настроено верно!

**Пример 2: Узнать время работы (uptime) серверов группы `webservers`**
Используем модуль `command` для выполнения любой shell-команды.

```bash
# Методичка
ansible webservers -i inventory.ini -m command -a "uptime -p"
```

**Ожидаемый вывод:**
```
server1 | CHANGED | rc=0 >>
up 2 weeks, 3 days, 15 hours, 3 minutes
server2 | CHANGED | rc=0 >>
up 1 day, 5 hours, 22 minutes
```

**Пример 3: Узнать свободное место на диске на всех серверах**
Используем модуль `shell` (он, в отличие от `command`, поддерживает пайпы `|` и перенаправления `>`).

```bash
# Методичка
ansible all -i inventory.ini -m shell -a "df -h | grep '/dev/sda1'"
```

**Пример 4: Установить пакет `mc` на серверы `webservers`**
Для этого нужно повышение прав (`sudo`). Добавим флаг `-b` (become).

```bash
# Методичка
ansible webservers -i inventory.ini -m apt -a "name=mc state=present" -b
```
*   `-m apt`: используем модуль для управления пакетами в Debian/Ubuntu. Для CentOS/RHEL будет `-m yum`.
*   `-a "name=mc state=present"`: аргументы модуля. `name` — имя пакета, `state=present` — убедиться, что он установлен.
*   `-b`: `become` — выполнить задачу от имени `sudo`.

Ad-hoc команды отлично подходят для быстрых проверок и одноразовых действий. Но для сложных, повторяемых задач нам понадобятся плейбуки.

**Далее:** В следующей теме мы напишем наш первый плейбук для автоматизации установки Nginx.
**[Перейти к Теме 3: Основы Playbooks](./03-playbooks-basics.md)**