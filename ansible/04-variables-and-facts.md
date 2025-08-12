# Тема 4: Переменные и факты: Делаем плейбуки гибкими

## Задача, которую мы решаем

Наш плейбук для установки Nginx работает отлично. Но есть проблема:
1.  Имя пакета "nginx" повторяется в двух задачах. Если мы захотим поменять его на "apache2", придется править несколько строк.
2.  Плейбук привязан к `apt`. На CentOS/RHEL имя пакета — `httpd`, а пакетный менеджер — `yum`. Плейбук не будет работать.
3.  Мы хотим создать на сервере приветственную страницу, которая будет показывать информацию о самом сервере (например, его имя хоста и ОС).

**Решение (Переменные и Факты):**
*   Использовать **переменные** для хранения значений (имя пакета, имя сервиса).
*   Использовать **факты** (автоматически собираемые переменные) для получения информации о системе.
*   Использовать **шаблоны (Jinja2)** для создания динамических файлов.

---

## Переменные в Playbook

Переменные позволяют вынести изменяемые значения из логики задач. Самый простой способ определить их — прямо в плейбуке в секции `vars`.

**Синтаксис:** Переменные используются внутри двойных фигурных скобок `{{ variable_name }}`.

### Методичка: Рефакторинг плейбука с переменными

1.  **Обновите файл `install_nginx.yml`:**
    Добавим секцию `vars` и заменим "nginx" на переменную `{{ web_package }}`.

    ```yaml
    ---
    - name: Install and configure Webserver
      hosts: webservers
      become: yes

      vars:
        web_package: nginx  # Определяем переменную

      tasks:
        - name: Update apt cache and install web package
          ansible.builtin.apt:
            name: "{{ web_package }}" # Используем переменную
            state: present
            update_cache: yes

        - name: Ensure web service is started and enabled
          ansible.builtin.service:
            name: "{{ web_package }}" # Используем переменную
            state: started
            enabled: yes
    ```
    Теперь, чтобы установить Apache, достаточно поменять значение `web_package` в одном месте. Плейбук стал чище и проще в поддержке.

---

## Факты Ansible и шаблоны Jinja2

При каждом запуске Ansible собирает информацию о системе — **факты**. Это сотни готовых переменных: `ansible_os_family`, `ansible_hostname`, `ansible_default_ipv4.address` и т.д.

Давайте используем их, чтобы создать динамическую главную страницу для нашего веб-сервера. Для этого используется модуль `template`.

### Методичка: Создание динамической страницы

1.  **Создайте файл шаблона `index.html.j2`:**
    В той же директории создайте файл с именем `index.html.j2`. Расширение `.j2` говорит о том, что это шаблон Jinja2.

    ```html
    <!DOCTYPE html>
    <html>
    <head>
        <title>Welcome to {{ ansible_hostname }}</title>
    </head>
    <body>
        <h1>This is {{ ansible_hostname }}!</h1>
        <p>
            It is running {{ ansible_distribution }} {{ ansible_distribution_version }}.
        </p>
        <p>
            Its IP address is {{ ansible_default_ipv4.address }}.
        </p>
    </body>
    </html>
    ```
    Здесь `{{ ansible_hostname }}` и другие — это факты, которые Ansible подставит автоматически для каждого хоста.

2.  **Добавьте новую задачу в плейбук `install_nginx.yml`:**
    Используем модуль `template` для копирования файла с заменой переменных.

    ```yaml
    ---
    - name: Install and configure Webserver
      hosts: webservers
      become: yes

      vars:
        web_package: nginx

      tasks:
        # ... (предыдущие две задачи остаются без изменений)
        - name: Update apt cache and install web package
          # ...
        - name: Ensure web service is started and enabled
          # ...
        
        - name: Deploy custom index.html from template
          ansible.builtin.template:
            src: index.html.j2  # Исходный файл шаблона на Control Node
            dest: /var/www/html/index.html # Путь назначения на Managed Node
            owner: root
            group: root
            mode: '0644'
    ```

3.  **Запустите плейбук и проверьте результат:**
    ```bash
    ansible-playbook -i inventory.ini install_nginx.yml
    ```
    После успешного выполнения откройте в браузере IP-адрес одного из ваших серверов (`http://192.168.1.10`) или воспользуйтесь `curl` из терминала:
    ```bash
    curl http://192.168.1.10
    ```

    Вы должны увидеть HTML-страницу с информацией, уникальной для этого сервера!

**Чтобы увидеть все доступные факты для хоста**, выполните ad-hoc команду:
```bash
ansible server1 -i inventory.ini -m setup
```

**Далее:** Мы научились делать плейбуки гибкими. Теперь пора добавить логику: выполнять задачи по условию и в циклах.
**[Перейти к Теме 5: Управление потоком и обработчики](./05-flow-control-and-handlers.md)**