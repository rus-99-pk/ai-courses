# Тема 3: Основы Playbooks: Автоматизируем по-взрослому

## Задача, которую мы решаем

С помощью Ad-hoc команд мы установили пакет `mc`. Но для полноценной настройки веб-сервера нам нужно выполнить несколько шагов:
1.  Обновить кэш пакетов.
2.  Установить пакет Nginx.
3.  Запустить службу Nginx.
4.  Включить автозапуск службы Nginx при старте системы.

**Проблема:** Выполнять 4 ad-hoc команды подряд — неудобно. Нет единого сценария, который можно было бы сохранить, версионировать (в Git) и переиспользовать.

**Решение (Ansible Playbook):**
Создать один YAML-файл (плейбук), который описывает все эти шаги. Этот файл станет нашим "рецептом" для настройки веб-сервера.

---

## Что такое Playbook?

**Playbook** — это сердце Ansible. Это файл в формате YAML, который содержит список задач для выполнения.

### Структура Playbook

```yaml
---
- name: Название нашего плейбука (например, Configure Webserver)
  hosts: webservers  # На какой группе хостов из inventory.ini выполнять?
  become: yes        # Выполнять задачи с правами sudo? (Аналог флага -b)

  tasks:
    - name: Описание первой задачи (например, Install Nginx)
      ansible.builtin.apt: # Имя модуля
        name: nginx
        state: present
        update_cache: yes

    - name: Описание второй задачи (например, Start Nginx service)
      ansible.builtin.service: # Другой модуль
        name: nginx
        state: started
        enabled: yes
```

**Ключевые моменты:**
*   `---`: Так начинается YAML-файл.
*   **Отступы:** YAML критичен к отступам! Используйте 2 пробела. Не используйте табы. Отступ определяет вложенность.
*   `hosts`: Указывает, на какую группу из `inventory.ini` нацелен плейбук. Можно указать `all` для всех.
*   `become: yes`: Говорит Ansible, что для выполнения задач в этом плее нужно повысить привилегии (стать `root`).
*   `tasks`: Список задач. Каждая задача начинается с дефиса `-`.
*   `name`: **Очень важно!** Это описание задачи, которое вы увидите в консоли при запуске. Всегда пишите понятные `name`.
*   `ansible.builtin.apt` / `ansible.builtin.service`: Полное имя модуля (FQCN). Можно использовать и короткие имена (`apt`, `service`), но полные считаются лучшей практикой.

---

### Методичка: Создаем и запускаем наш первый плейбук

1.  **Создайте файл `install_nginx.yml`:**
    В той же директории, где лежит `inventory.ini`, создайте файл `install_nginx.yml` со следующим содержимым:

    ```yaml
    ---
    - name: Install and configure Nginx
      hosts: webservers
      become: yes

      tasks:
        - name: Update apt cache and install nginx
          ansible.builtin.apt:
            name: nginx
            state: present
            update_cache: yes

        - name: Ensure Nginx is started and enabled on boot
          ansible.builtin.service:
            name: nginx
            state: started
            enabled: yes
    ```

2.  **Запустите плейбук:**
    Команда для запуска отличается от ad-hoc. Используется `ansible-playbook`.

    ```bash
    # Методичка
    ansible-playbook -i inventory.ini install_nginx.yml
    ```

3.  **Анализируем вывод:**
    Вы увидите что-то вроде этого:

    ```
    PLAY [Install and configure Nginx] **************************************

    TASK [Gathering Facts] **************************************************
    ok: [server1]
    ok: [server2]

    TASK [Update apt cache and install nginx] *******************************
    changed: [server1]
    changed: [server2]

    TASK [Ensure Nginx is started and enabled on boot] **********************
    changed: [server1]
    changed: [server2]

    PLAY RECAP **************************************************************
    server1                    : ok=3    changed=2    unreachable=0    failed=0
    server2                    : ok=3    changed=2    unreachable=0    failed=0
    ```
    *   `TASK [Gathering Facts]`: Ansible по умолчанию собирает информацию о хостах.
    *   `changed`: Означает, что Ansible внес изменения в систему (установил пакет, запустил сервис).
    *   `ok`: Задача выполнена успешно.
    *   `PLAY RECAP`: Итог. `changed=2` означает, что две задачи изменили состояние системы.

4.  **Проверяем идемпотентность!**
    Запустите ту же самую команду еще раз:
    ```bash
    ansible-playbook -i inventory.ini install_nginx.yml
    ```
    Теперь вывод будет другим:

    ```
    ...
    TASK [Update apt cache and install nginx] *******************************
    ok: [server1]
    ok: [server2]

    TASK [Ensure Nginx is started and enabled on boot] **********************
    ok: [server1]
    ok: [server2]

    PLAY RECAP **************************************************************
    server1                    : ok=3    changed=0    unreachable=0    failed=0
    server2                    : ok=3    changed=0    unreachable=0    failed=0
    ```
    Обратите внимание: `changed=0`. Ansible проверил, что Nginx уже установлен и запущен, и ничего не делал. В этом его сила!

**Далее:** Наш плейбук работает, но он негибкий. Имя пакета "nginx" зашито прямо в коде. В следующей теме мы научимся использовать переменные.
**[Перейти к Теме 4: Переменные и факты](./04-variables-and-facts.md)**