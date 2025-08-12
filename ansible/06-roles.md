# Тема 6: Роли: Структурирование и переиспользование кода

## Задача, которую мы решаем

Наш плейбук `install_nginx.yml` разросся. В нем перемешаны:
*   Переменные (`vars`).
*   Задачи (`tasks`).
*   Шаблоны (файл `index.html.j2`).
*   Конфигурационные файлы (`my_nginx.conf`).
*   Обработчики (`handlers`).

**Проблема:**
1.  **Сложность поддержки:** В одном большом файле легко запутаться.
2.  **Невозможность переиспользования:** Мы настроили Nginx. Теперь мы хотим настроить базу данных PostgreSQL в другом проекте. Копировать куски из старого плейбука? Это плохой путь, ведущий к дублированию и ошибкам.

**Решение (Роли):**
**Роль** — это стандартизированный способ организации контента Ansible (задач, переменных, файлов, шаблонов, хендлеров) в единую, переиспользуемую структуру. Роль инкапсулирует в себе всё необходимое для настройки определенного компонента, например, "веб-сервер", "база данных", "мониторинг".

---

### Стандартная структура директорий для Роли

Ansible ожидает, что роль будет иметь определенную структуру. Создадим роль с именем `webserver`.

```
roles/
└── webserver/
    ├── tasks/
    │   └── main.yml      # Основной файл с задачами для этой роли
    ├── handlers/
    │   └── main.yml      # Хендлеры для этой роли
    ├── templates/
    │   └── index.html.j2 # Шаблоны, используемые ролью
    ├── files/
    │   └── my_nginx.conf # Статические файлы для копирования
    ├── vars/
    │   └── main.yml      # Переменные, относящиеся к этой роли
    └── defaults/
        └── main.yml      # Переменные по умолчанию (низкий приоритет)
```
*   `tasks/main.yml`: Сюда мы перенесем все наши задачи из плейбука.
*   `handlers/main.yml`: Сюда — хендлеры.
*   `templates/`: Каталог для шаблонов Jinja2.
*   `files/`: Каталог для статичных файлов (модуль `copy`).
*   `vars/main.yml`: Переменные, специфичные для роли.
*   `defaults/main.yml`: Переменные по умолчанию. Их легко переопределить.

---

### Методичка: Создаем и используем роль `webserver`

1.  **Создайте структуру директорий:**
    В корне вашего проекта создайте директорию `roles`, а внутри нее — `webserver` со всей структурой выше.

    ```bash
    mkdir -p roles/webserver/{tasks,handlers,templates,files,vars}
    touch roles/webserver/tasks/main.yml
    touch roles/webserver/handlers/main.yml
    touch roles/webserver/vars/main.yml
    ```

2.  **"Разнесите" наш старый плейбук по файлам роли:**

    *   **`roles/webserver/vars/main.yml`** (Переменные):
        ```yaml
        ---
        web_package: nginx
        ```

    *   **`roles/webserver/tasks/main.yml`** (Задачи):
        ```yaml
        ---
        # tasks file for webserver role
        - name: Install web package
          ansible.builtin.apt:
            name: "{{ web_package }}"
            state: present
            update_cache: yes

        - name: Ensure web service is started and enabled
          ansible.builtin.service:
            name: "{{ web_package }}"
            state: started
            enabled: yes
        
        - name: Copy custom Nginx configuration
          ansible.builtin.copy:
            src: my_nginx.conf
            dest: /etc/nginx/conf.d/my_site.conf
          notify: Restart Webserver

        - name: Deploy custom index.html from template
          ansible.builtin.template:
            src: index.html.j2
            dest: /var/www/html/index.html
        ```
        *Обратите внимание: `src` в модулях `copy` и `template` теперь ищет файлы относительно каталогов `files` и `templates` внутри роли.*

    *   **`roles/webserver/handlers/main.yml`** (Хендлеры):
        ```yaml
        ---
        # handlers file for webserver role
        - name: Restart Webserver
          ansible.builtin.service:
            name: "{{ web_package }}"
            state: restarted
        ```

    *   **Переместите файлы:**
        *   `my_nginx.conf` -> `roles/webserver/files/my_nginx.conf`
        *   `index.html.j2` -> `roles/webserver/templates/index.html.j2`

3.  **Создайте новый, "чистый" плейбук `site.yml`:**
    Теперь наш основной плейбук становится очень простым. Он просто вызывает нужные роли. Это называется "композиция".

    ```yaml
    ---
    - name: Configure Webservers
      hosts: webservers
      become: yes
      
      roles:
        - webserver
        # Если бы у нас была роль для базы данных, мы бы добавили её сюда:
        # - role: database
        #   db_user: myuser
        #   db_pass: mypass
    ```

4.  **Запустите новый плейбук:**
    Убедитесь, что ваш `inventory.ini` на месте.

    ```bash
    ansible-playbook -i inventory.ini site.yml
    ```

Результат будет точно таким же, как и раньше, но теперь наш код организован, структурирован и готов к переиспользованию. Мы можем взять директорию `roles/webserver` и перенести её в любой другой проект Ansible!

**Далее:** Мы почти у цели. В последней теме мы разберем, как безопасно хранить секреты и как использовать чужие роли из сообщества.
**[Перейти к Теме 7: Продвинутые темы](./07-vault-and-galaxy.md)**