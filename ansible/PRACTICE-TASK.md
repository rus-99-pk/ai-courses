# Практическое задание: Установка стека LEMP

Поздравляем! Вы прошли теоретическую часть курса. Теперь настало время собрать все знания воедино и выполнить комплексную задачу — автоматизировать развертывание стека **LEMP** (Linux, Nginx, MySQL/MariaDB, PHP) на сервере.

## Цель

Написать Ansible-проект, который с нуля настраивает сервер для работы простого PHP-сайта с базой данных. Проект должен быть структурирован с использованием ролей.

## Требования к конечному результату

1.  **Структура проекта:**
    *   Проект должен использовать роли. Как минимум, должны быть созданы роли `nginx`, `mariadb`, `php`.
    *   Чувствительные данные (пароль root для MariaDB) должны храниться в зашифрованном файле `vault.yml`.

2.  **Функционал:**
    *   **Nginx:**
        *   Устанавливается Nginx.
        *   Настраивается `server block` (виртуальный хост) для обработки PHP-скриптов через PHP-FPM.
        *   Создается тестовая страница `info.php`, которая выводит результат `phpinfo()`.
    *   **MariaDB (аналог MySQL):**
        *   Устанавливается сервер MariaDB.
        *   Задается пароль для пользователя `root` (пароль берется из Vault).
        *   Создается отдельная база данных (например, `webapp_db`).
        *   Создается отдельный пользователь базы данных (например, `webapp_user`) с паролем (тоже из Vault), которому даны все права на `webapp_db`.
    *   **PHP:**
        *   Устанавливается PHP-FPM и модуль для работы с MySQL (`php-mysql`).
        *   Служба PHP-FPM запущена и добавлена в автозагрузку.

## План выполнения (Шаги-методичка)

### Шаг 1: Подготовка проекта

1.  Создайте новую директорию для проекта, например, `ansible-lemp`.
2.  Создайте базовые файлы: `inventory.ini` (с одним вашим сервером в группе `[lemp]`), `site.yml` (главный плейбук).
3.  Создайте файл `.vault_pass` с вашим паролем для Vault и добавьте его в `.gitignore`.
4.  Создайте и зашифруйте файл `vault.yml` (`ansible-vault create vault.yml`). Добавьте в него переменные для MariaDB:
    ```yaml
    mariadb_root_password: "SomeVeryStrongRootPassword"
    webapp_db_name: "webapp_db"
    webapp_db_user: "webapp_user"
    webapp_db_password: "AnotherStrongPassword"
    ```

### Шаг 2: Создание ролей

1.  Используя `ansible-galaxy init`, создайте заготовки для трех ролей:
    ```bash
    ansible-galaxy init roles/nginx
    ansible-galaxy init roles/mariadb
    ansible-galaxy init roles/php
    ```

### Шаг 3: Наполнение роли `php`

*   **`roles/php/tasks/main.yml`:**
    *   Задача на установку пакетов `php-fpm` и `php-mysql` с помощью модуля `apt` или `yum`. Используйте цикл `loop`.
    *   Задача на запуск и включение сервиса `php-fpm` (имя сервиса может отличаться в разных ОС, например `php7.4-fpm`).

### Шаг 4: Наполнение роли `mariadb`

Это самая сложная роль, так как требует работы с базой данных.

*   **`roles/mariadb/tasks/main.yml`:**
    *   Задача на установку пакета `mariadb-server` и `python3-mysqldb` (нужен Ansible для работы с MySQL).
    *   Задача на запуск сервиса `mariadb`.
    *   Задача на установку пароля для `root` с помощью модуля `community.mysql.mysql_user`. Используйте `check_implicit_admin=yes`.
    *   Задача на создание базы данных `webapp_db` (`community.mysql.mysql_db`).
    *   Задача на создание пользователя `webapp_user` с паролем и правами на `webapp_db` (снова `mysql_user`).

*   **Подсказка:** Переменные берите из `vault.yml`. Например, `{{ mariadb_root_password }}`.

### Шаг 5: Наполнение роли `nginx`

*   **`roles/nginx/tasks/main.yml`:**
    *   Задача на установку `nginx`.
    *   Задача для копирования конфигурации виртуального хоста с помощью модуля `template`. Конфигурация должна перенаправлять `.php` запросы на сокет PHP-FPM.
    *   Задача на создание тестового файла `/var/www/html/info.php` с содержимым `<?php phpinfo(); ?>` с помощью модуля `copy`.
*   **`roles/nginx/templates/default.conf.j2`:** Создайте шаблон конфигурации для Nginx. Примерный блок `location` для PHP:
    ```nginx
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock; # Путь может отличаться!
    }
    ```
*   **`roles/nginx/handlers/main.yml`:** Добавьте хендлер для перезапуска Nginx, который будет вызываться после изменения конфигурации.

### Шаг 6: Сборка и запуск

1.  **Отредактируйте `site.yml`:**
    Он должен подключать Vault-файл и вызывать все три роли последовательно.

    ```yaml
    ---
    - hosts: lemp
      become: yes
      vars_files:
        - vault.yml

      roles:
        - php
        - mariadb
        - nginx
    ```

2.  **Запустите плейбук:**
    ```bash
    ansible-playbook -i inventory.ini site.yml --vault-password-file .vault_pass
    ```

### Шаг 7: Проверка

1.  Откройте в браузере `http://<your_server_ip>/info.php`. Вы должны увидеть страницу с информацией о PHP.
2.  (Опционально) Подключитесь к серверу и проверьте, что можете зайти в MySQL под пользователем `webapp_user`:
    ```bash
    mysql -u webapp_user -p
    ```
    Введите пароль из вашего Vault-файла. Затем выполните `SHOW DATABASES;` — вы должны увидеть `webapp_db`.

Удачи!