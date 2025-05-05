Отлично! Теперь, когда мы умеем автоматизировать инфраструктуру и приложение, пора включить в конвейер еще один критически важный компонент — базы данных. Этот урок посвящен DBOps и управлению изменениями в БД.

---

**УРОК 6: DBOps: Реляционные и Нереляционные Базы Данных**

**Цель:** Понять вызовы, связанные с управлением базами данных в контексте DevOps (DBOps), освоить инструменты для версионирования и автоматического применения изменений схемы БД (миграций), а также получить базовый опыт работы с реляционной (PostgreSQL) и нереляционной (MongoDB) базами данных.

**Результат урока:** Вы сможете разворачивать экземпляры PostgreSQL и MongoDB, понимать их базовые принципы. Ваш Delivery Pipeline будет расширен, чтобы автоматически выполнять миграции схемы базы данных перед каждым деплоем приложения, обеспечивая согласованность кода и структуры данных.

**Предварительные требования:**

*   Успешно завершен Урок 1-5.
*   Рабочий CI/CD пайплайн с автоматической подготовкой Staging VM и деплоем приложения на нее.
*   Две VM: одна для развертывания баз данных (DB VM) и ваша существующая Staging VM. Можно использовать одну VM для обеих БД, или даже разместить их на Staging VM для простоты, но в реальной жизни БД обычно на отдельных серверах. Для этой методички предполагается, что БД будут на отдельной **DB VM**.
*   SSH доступ по ключу с вашей Jenkins/Runner VM к DB VM.

**Техническая подготовка (на Вашей VM или новых VM):**

1.  **Разверните DB VM:** Создайте новую VM Linux (как в Уроке 3). Установите на ней SSH сервер и настройте SSH доступ по ключу с вашей Jenkins/Runner VM.

2.  **Установите Docker Compose:** Если еще не установлено, установите его на DB VM (см. инструкции в Уроке 2). Это самый простой способ развернуть БД.

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **Реляционные Базы Данных (RDBMS):** Данные организованы в таблицы со строками и столбцами. Связи между таблицами устанавливаются с помощью ключей (Primary Keys, Foreign Keys). Примеры: PostgreSQL, MySQL, Oracle, MS SQL Server.
    *   **SQL (Structured Query Language):** Стандартизированный язык для взаимодействия с реляционными БД (создание таблиц, вставка, выборка, обновление, удаление данных).
    *   **Нормализация:** Процесс организации данных в БД для уменьшения избыточности и улучшения целостности.
    *   **Высокая доступность (HA):** Меры для обеспечения непрерывной работы БД при сбоях (репликация, кластеризация, резервное копирование). Кратко: **репликация** (копирование данных на другие серверы), **резервное копирование** (спасение от потери данных, а не от простоя).

*   **Нереляционные Базы Данных (NoSQL):** Хранят данные в форматах, отличных от таблиц (документы, пары ключ-значение, графы, широкие столбцы). Используются, когда реляционная модель неудобна (например, для неструктурированных данных, очень высокой масштабируемости или специфических типов запросов). Примеры: MongoDB (документы), Redis (ключ-значение), Cassandra (широкие столбцы), Neo4j (графы).

*   **DBOps:** Применение принципов и практик DevOps к управлению базами данных. Цель: автоматизировать процессы изменения, тестирования и развертывания схем и данных БД, сделать их частью общего конвейера доставки ПО. Проблемы DBOps:
    *   Изменения в БД часто более рискованны (обратная несовместимость, потеря данных).
    *   Состояние БД является персистентным (долгоживущим).
    *   Схема БД и код приложения должны быть согласованы.
    *   Традиционно управление БД находится в отдельной команде DBA.

*   **Миграции Базы Данных (Database Migrations):** Версионированные, последовательные изменения схемы БД или данных, описанные в виде скриптов. Позволяют отслеживать историю изменений схемы, применять их автоматически, откатывать (если скрипты написаны соответствующим образом). Инструменты миграции (Flyway, Liquibase) отслеживают, какие миграции уже применены к конкретной базе данных, и применяют только новые.

*   **Flyway:** Популярный инструмент миграции с открытым исходным кодом. Использует простые SQL-скрипты (или Java-классы) с именованием по соглашению (например, `V1__create_table_users.sql`). Ведет историю примененных миграций в специальной таблице в базе данных.

*   **MongoDB:** Пример документоориентированной NoSQL БД. Данные хранятся в формате BSON (бинарный JSON) в *документах*, которые объединяются в *коллекции*. Документы в одной коллекции могут иметь разную структуру (схема гибкая).

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Установка и Базовая Работа с PostgreSQL**

*   **Цель:** Развернуть PostgreSQL и научиться выполнять базовые SQL-запросы.
*   **Шаги:**

    1.  **Создайте каталог для PostgreSQL и файл `docker-compose.yml` на DB VM:**
        ```bash
        mkdir $HOME/postgresql
        cd $HOME/postgresql
        nano docker-compose.yml
        ```
    2.  **Добавьте следующее содержимое:**

        ```yaml
        version: "3.8"

        services:
          postgres:
            image: postgres:13-alpine # Используйте актуальную и стабильную версию
            container_name: postgres-db
            ports:
              - "5432:5432" # Проброс порта
            environment:
              POSTGRES_DB: myappdb # Имя базы данных
              POSTGRES_USER: myappuser # Имя пользователя
              POSTGRES_PASSWORD: mypassword # ПАРОЛЬ! ИЗМЕНИТЕ В PRODUCTION!
            volumes:
              - postgres_data:/var/lib/postgresql/data # Персистентное хранение данных
            restart: unless-stopped # Перезапускать контейнер, если он упал
            healthcheck: # Проверка работоспособности
              test: ["CMD-SHELL", "pg_isready -U myappuser -d myappdb"]
              interval: 5s
              timeout: 5s
              retries: 5

        volumes:
          postgres_data:
        ```
    3.  **Запустите контейнер PostgreSQL:**
        ```bash
        docker-compose up -d
        ```
        Подождите минуту или две, пока контейнер запустится и БД инициализируется.

    4.  **Подключитесь к PostgreSQL из командной строки VM:** Установите клиент PostgreSQL на DB VM или на вашей Jenkins VM (если нужно подключаться оттуда).
        ```bash
        # Установите клиент (на DB VM или другой VM с доступом):
        sudo apt install postgresql-client -y # или yum install postgresql -y

        # Подключитесь к вашей БД, запущенной в Docker (через localhost, т.к. порт проброшен)
        # Используйте переменные окружения для логина/пароля или введите их при запросе
        PGPASSWORD=mypassword psql -h localhost -p 5432 -U myappuser -d myappdb
        ```
    5.  **Выполните базовые SQL запросы:**
        *   Посмотреть список таблиц: `\dt`
        *   Создать простую таблицу `users`:
            ```sql
            CREATE TABLE users (
                id SERIAL PRIMARY KEY,
                username VARCHAR(50) UNIQUE NOT NULL,
                email VARCHAR(100) UNIQUE NOT NULL,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            );
            ```
            (Не забудьте точку с запятой в конце).
        *   Проверить, что таблица создана: `\dt`
        *   Посмотреть описание таблицы: `\d users`
        *   Вставить данные:
            ```sql
            INSERT INTO users (username, email) VALUES ('john.doe', 'john.doe@example.com');
            INSERT INTO users (username, email) VALUES ('jane.smith', 'jane.smith@example.com');
            ```
        *   Выбрать данные:
            ```sql
            SELECT * FROM users;
            SELECT id, username FROM users WHERE username = 'john.doe';
            ```
        *   Обновить данные:
            ```sql
            UPDATE users SET email = 'john.doe.new@example.com' WHERE username = 'john.doe';
            ```
        *   Удалить данные:
            ```sql
            DELETE FROM users WHERE username = 'jane.smith';
            ```
        *   Проверить изменения: `SELECT * FROM users;`
        *   Удалить таблицу: `DROP TABLE users;`
        *   Выйти из клиента psql: `\q`

**Результат Lab 1:** Установленный PostgreSQL сервер в Docker, умение подключаться к нему и выполнять базовые операции с данными и схемой через SQL.

---

**Lab 2: Установка и Базовая Работа с MongoDB**

*   **Цель:** Развернуть MongoDB и научиться выполнять базовые операции с документами.
*   **Шаги:**

    1.  **Создайте каталог для MongoDB и файл `docker-compose.yml` на DB VM:**
        ```bash
        mkdir $HOME/mongodb
        cd $HOME/mongodb
        nano docker-compose.yml
        ```
    2.  **Добавьте следующее содержимое:**

        ```yaml
        version: "3.8"

        services:
          mongodb:
            image: mongo:latest # Используйте актуальную версию
            container_name: mongodb-db
            ports:
              - "27017:27017" # Проброс порта по умолчанию
            environment:
              MONGO_INITDB_ROOT_USERNAME: rootuser # Имя root пользователя
              MONGO_INITDB_ROOT_PASSWORD: rootpassword # ПАРОЛЬ! ИЗМЕНИТЕ В PRODUCTION!
              # Можно также настроить создание первой базы данных и пользователя для приложения здесь
            volumes:
              - mongodb_data:/data/db # Персистентное хранение данных
            restart: unless-stopped
            healthcheck: # Проверка работоспособности
              test: echo 'db.runCommand("ping").ok' | mongo localhost:27017/test --quiet
              interval: 5s
              timeout: 5s
              retries: 5

        volumes:
          mongodb_data:
        ```
    3.  **Запустите контейнер MongoDB:**
        ```bash
        docker-compose up -d
        ```
        Подождите немного, пока контейнер запустится.

    4.  **Подключитесь к MongoDB из командной строки VM:** Установите клиент MongoDB (`mongosh`).
        ```bash
        # Установите mongosh (на DB VM или другой VM с доступом):
        # Следуйте официальной документации MongoDB для установки mongosh,
        # т.к. его нет в стандартных репозиториях большинства дистрибутивов.
        # Пример для Ubuntu/Debian (смотрите актуальные инструкции на mongodb.com!):
        # wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add - # Замените версию на актуальную!
        # echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
        # sudo apt update
        # sudo apt install -y mongosh

        # Подключитесь к вашей БД, запущенной в Docker:
        mongosh "mongodb://rootuser:rootpassword@localhost:27017" # Укажите логин/пароль/хост/порт
        # Или проще, если логин/пароль не требуются для localhost (зависит от версии/конфига):
        # mongosh
        ```
        (Может потребоваться аутентификация после подключения: `use admin`, `db.auth('rootuser', 'rootpassword')`).

    5.  **Выполните базовые операции с документами:**
        *   Посмотреть список баз данных: `show dbs`
        *   Переключиться на базу данных (создаст ее, если не существует): `use myappmongodb`
        *   Посмотреть список коллекций: `show collections`
        *   Вставить один документ в коллекцию `products` (создаст коллекцию, если не существует):
            ```javascript
            db.products.insertOne({ name: "Laptop", brand: "Dell", price: 1200, tags: ["electronics", "computer"] });
            ```
        *   Вставить несколько документов:
            ```javascript
            db.products.insertMany([
              { name: "Smartphone", brand: "Samsung", price: 800, tags: ["electronics", "mobile"] },
              { name: "Tablet", brand: "Apple", price: 500, tags: ["electronics", "mobile"] }
            ]);
            ```
        *   Найти все документы в коллекции: `db.products.find()`
        *   Найти документы с условием: `db.products.find({ brand: "Samsung" })`
        *   Найти документы с условием и проекцией (показать только некоторые поля): `db.products.find({ price: { $gt: 700 } }, { name: 1, price: 1, _id: 0 })`
        *   Обновить один документ: `db.products.updateOne({ name: "Laptop" }, { $set: { price: 1150 } })`
        *   Удалить один документ: `db.products.deleteOne({ name: "Tablet" })`
        *   Проверить изменения: `db.products.find()`
        *   Удалить коллекцию: `db.products.drop()`
        *   Удалить базу данных: `db.dropDatabase()`
        *   Выйти из клиента mongosh: `exit`

**Результат Lab 2:** Установленный MongoDB сервер в Docker, умение подключаться к нему и выполнять базовые операции с коллекциями и документами через `mongosh`.

---

**Lab 3: Подключение Приложения к БД**

*   **Цель:** Изменить код вашего тестового приложения, чтобы оно могло подключаться и выполнять базовые операции с PostgreSQL и MongoDB.
*   **Предварительные требования:**
    *   Работающие серверы PostgreSQL и MongoDB (Lab 1 и 2).
    *   Ваше тестовое приложение.
*   **Шаги:**

    1.  **Определите IP-адрес и порт ваших DB серверов, доступные с Staging VM:** Если БД запущены на отдельной DB VM в Docker, используйте IP DB VM и проброшенные порты (5432 для Postgres, 27017 для Mongo). Убедитесь, что файрвол на DB VM разрешает входящие соединения с Staging VM на эти порты.

    2.  **Добавьте зависимости в `pom.xml` вашего приложения:**
        *   Для PostgreSQL: `postgresql` драйвер.
        *   Для MongoDB: `mongodb-driver-sync` или драйвер, соответствующий вашему фреймворку (например, Spring Data MongoDB).

        ```xml
        <dependencies>
            <!-- ... другие зависимости ... -->

            <!-- PostgreSQL Driver -->
            <dependency>
                <groupId>org.postgresql</groupId>
                <artifactId>postgresql</artifactId>
                <version>42.5.0</version> <!-- Используйте актуальную версию -->
            </dependency>

            <!-- MongoDB Driver (Sync) -->
            <dependency>
                <groupId>org.mongodb</groupId>
                <artifactId>mongodb-driver-sync</artifactId>
                <version>4.8.1</version> <!-- Используйте актуальную версию -->
            </dependency>

            <!-- Если используете Spring Boot, добавьте соответствующие стартеры: -->
            <!--
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-data-jpa</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-data-mongodb</artifactId>
            </dependency>
             <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-jdbc</artifactId>
            </dependency>
             -->

        </dependencies>
        ```

    3.  **Измените код приложения для подключения и выполнения базовых операций:**
        *   **Конфигурация:** Следуйте принципам Twelve-Factor App (Урок 4, Lab 6) и настройте приложение так, чтобы оно получало параметры подключения к БД (URL, логин, пароль, имя БД) из **переменных окружения**.
        *   **Код:** Добавьте простой функционал для взаимодействия с каждой БД:
            *   **PostgreSQL:** Например, создать JpaRepository (если используете Spring Data JPA) или использовать JdbcTemplate для сохранения и чтения данных из таблицы `users`, созданной в Lab 1.
            *   **MongoDB:** Например, создать MongoRepository (если используете Spring Data MongoDB) или использовать MongoDB Java Driver для сохранения и чтения документов из коллекции `products`.
        *   Добавьте API endpoint'ы или команды, которые будут вызывать этот функционал, чтобы вы могли проверить его работу.

    4.  **Настройте запуск приложения на Staging VM с переменными окружения для БД:**
        *   Обновите ваш Ansible Playbook (`playbook_app_setup.yml`) или скрипт деплоя (`deploy_app_to_staging.sh`), чтобы передавать переменные окружения при запуске приложения.
        *   Если используете service unit (Lab 5 Урока 5):

            ```ini
            [Service]
            # ...
            Environment="SPRING_DATASOURCE_URL=jdbc:postgresql://IP_DB_VM:5432/myappdb"
            Environment="SPRING_DATASOURCE_USERNAME=myappuser"
            Environment="SPRING_DATASOURCE_PASSWORD=mypassword"
            Environment="SPRING_DATA_MONGODB_URI=mongodb://rootuser:rootpassword@IP_DB_VM:27017/myappmongodb?authSource=admin" # Укажите правильно URI
            # ...
            ```
        *   Если запускаете скриптом:

            ```bash
            # Внутри deploy_app_to_staging.sh, перед java -jar ...
            export SPRING_DATASOURCE_URL="jdbc:postgresql://IP_DB_VM:5432/myappdb"
            export SPRING_DATASOURCE_USERNAME="myappuser"
            export SPRING_DATASOURCE_PASSWORD="mypassword"
            export SPRING_DATA_MONGODB_URI="mongodb://rootuser:rootpassword@IP_DB_VM:27017/myappmongodb?authSource=admin"
            nohup java ...
            ```
            (Замените `IP_DB_VM` на реальный IP вашей DB VM). **ИЗБЕГАЙТЕ ХРАНЕНИЯ ПАРОЛЕЙ В ЯВНОМ ВИДЕ В СКРИПТАХ/PLAYBOOKS!** Используйте Ansible Vault (будет в Уроке 5 Теория) или HashiCorp Vault (Урок 7) для безопасного хранения секретов.

    5.  **Соберите приложение локально (`mvn clean package`), чтобы убедиться, что зависимости добавлены корректно.**

    6.  **Сделайте коммит и пуш измененного кода приложения и конфигурации деплоя:**
        ```bash
        git add my-app/pom.xml my-app/src/main/java/... # Добавьте измененные файлы кода
        git add $HOME/deployment_scripts/deploy_app_to_staging.sh # Если изменили скрипт
        # Или обновите playbook_app_setup.yml и myjavaapp.service, если используете service unit
        # git add $HOME/ansible-config/...
        git commit -m "feat: Connect app to PostgreSQL and MongoDB"
        git push origin develop
        ```

    7.  **Запустите Jenkins пайплайн (или GitLab CI) и проверьте деплой на Staging VM.** После успешного деплоя, проверьте работу функционала, который использует БД (например, через API вашего приложения).

**Результат Lab 3:** Ваше приложение теперь может подключаться и работать с PostgreSQL и MongoDB, получая параметры подключения из переменных окружения.

---

**Lab 4: Интеграция Flyway для Миграций БД**

*   **Цель:** Интегрировать Flyway в Maven проект и написать первую миграцию.
*   **Предварительные требования:**
    *   Работающий PostgreSQL сервер.
    *   Ваш Maven проект.
*   **Шаги:**

    1.  **Добавьте зависимость Flyway в `pom.xml`:**

        ```xml
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
            <version>9.10.0</version> <!-- Используйте актуальную версию -->
        </dependency>
        <!-- Также нужна зависимость драйвера БД, например, postgresql (уже добавлено в Lab 3) -->
        ```

    2.  **Добавьте плагин Flyway Maven Plugin в `pom.xml`:**

        ```xml
        <build>
            <plugins>
                <!-- ... другие плагины ... -->
                <plugin>
                    <groupId>org.flywaydb</groupId>
                    <artifactId>flyway-maven-plugin</artifactId>
                    <version>9.10.0</version> <!-- Используйте ту же версию -->
                    <configuration>
                        <!-- Параметры подключения к БД - ВАЖНО: Не храните здесь секреты! -->
                        <!-- Лучше использовать переменные окружения или параметры Maven,
                             которые будут установлены CI/CD системой -->
                        <url>${env.DB_URL}</url>
                        <user>${env.DB_USER}</user>
                        <password>${env.DB_PASSWORD}</password>
                        <!-- Расположение скриптов миграций -->
                        <locations>
                            <location>classpath:db/migration</location>
                        </locations>
                         <!-- Имя схемы (если используется не по умолчанию) -->
                        <schemas>
                            <schema>public</schema>
                        </schemas>
                    </configuration>
                    <dependencies>
                        <!-- Зависимость от драйвера БД для Flyway плагина -->
                        <dependency>
                            <groupId>org.postgresql</groupId>
                            <artifactId>postgresql</artifactId>
                            <version>42.5.0</version> <!-- Та же версия, что и в dependencies -->
                        </dependency>
                    </dependencies>
                </plugin>
            </plugins>
        </build>
        ```
        *   **Важно:** Обратите внимание на использование `${env.DB_URL}` и т.д. Это переменные окружения, которые мы будем устанавливать в Jenkins/GitLab CI. Не захардкоживайте учетные данные здесь.

    3.  **Создайте каталог для скриптов миграций:** Flyway по умолчанию ищет скрипты в `src/main/resources/db/migration`.
        ```bash
        mkdir -p my-app/src/main/resources/db/migration
        ```

    4.  **Напишите первый скрипт миграции:** Скрипты именуются по соглашению: `V<версия>__<описание>.sql`. Версии должны быть уникальными и следовать порядку.
        ```bash
        nano my-app/src/main/resources/db/migration/V1__create_users_table.sql
        ```
        ```sql
        -- V1__create_users_table.sql
        CREATE TABLE users (
            id SERIAL PRIMARY KEY,
            username VARCHAR(50) UNIQUE NOT NULL,
            email VARCHAR(100) UNIQUE NOT NULL,
            created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
        );
        ```
        (Это тот же запрос, что вы выполняли вручную в Lab 1).

    5.  **Проверьте миграцию локально (опционально):** Создайте временную локальную БД или используйте тестовую БД. Установите переменные окружения (`DB_URL`, `DB_USER`, `DB_PASSWORD`) для подключения к ней и выполните:
        ```bash
        cd my-app
        # Пример для подключения к локальной PostgreSQL (если запущена):
        export DB_URL="jdbc:postgresql://localhost:5432/myappdb"
        export DB_USER="myappuser"
        export DB_PASSWORD="mypassword"
        mvn flyway:info # Показать статус миграций (не применены)
        mvn flyway:migrate # Применить миграции
        mvn flyway:info # Показать статус миграций (применены)
        ```
        Проверьте в БД, что таблица `users` создана и появилась таблица `flyway_schema_history`.

    6.  **Сделайте коммит и пуш измененных файлов в Git:**
        ```bash
        git add my-app/pom.xml my-app/src/main/resources/db/migration/
        git commit -m "feat: Add Flyway and V1 migration for users table"
        git push origin develop
        ```

**Результат Lab 4:** Ваш проект настроен на использование Flyway для миграций БД, и у вас есть первая миграция, которая создает таблицу пользователей.

---

**Lab 5: Автоматизация Миграций БД в Pipeline**

*   **Цель:** Добавить в Delivery Pipeline этап, который автоматически запускает Flyway для применения миграций к базе данных на Staging VM перед деплоем приложения.
*   **Предварительные требования:**
    *   Рабочий пайплайн с этапами Provision и Configure (Урок 5, Lab 6).
    *   Проект с настроенным Flyway (Lab 4 этого урока).
    *   DB VM с запущенным PostgreSQL.
    *   Настроенный SSH доступ с Jenkins/Runner VM к DB VM.
*   **Шаги (для Jenkinsfile, адаптация для GitLab CI аналогична):**

    1.  **Обновите ваш Jenkinsfile:** Добавьте новый этап *после* настройки VM (Configure VM), но *перед* деплоем приложения (Deploy to Staging).

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any
            environment {
                // ... (Infra code dir, SSH user variables) ...

                // Переменные окружения для подключения к БД (для Flyway и приложения)
                // ИСПОЛЬЗУЙТЕ JENKINS CREDENTIALS ДЛЯ ПАРОЛЯ!
                // Добавьте новый Secret Text Credential в Jenkins для пароля БД (ID: db-password)
                DB_URL = "jdbc:postgresql://IP_DB_VM:5432/myappdb" // Замените IP_DB_VM
                DB_USER = "myappuser"
                // DB_PASSWORD = credentials('db-password') // Безопасно!
                DB_PASSWORD = 'mypassword' // НЕБЕЗОПАСНО ДЛЯ ДЕМО! Используйте credentials!

                // Для MongoDB (для приложения)
                MONGODB_URI = "mongodb://rootuser:rootpassword@IP_DB_VM:27017/myappmongodb?authSource=admin" // Замените IP_DB_VM
                // Используйте Jenkins Credentials для rootpassword MongoDB тоже!
            }

            stages {
                // ... (Clone Infra Code, Provision Infrastructure, Wait for SSH, Configure VM этапы) ...

                // НОВЫЙ ЭТАП: Database Migrations (Flyway)
                stage('Database Migrations') {
                    steps {
                        echo 'Применение миграций БД с помощью Flyway...'
                        dir("my-app") { // Переходим в каталог Maven проекта
                            // Запускаем Flyway Maven Plugin.
                            // Параметры подключения берутся из переменных окружения (которые определены выше).
                            // Если вы используете credentials() для пароля, вам нужно передать его явно в вызове sh
                            // или использовать плагины Jenkins, которые инжектят credentials как переменные окружения.
                            // Пример с инжекцией credentials:
                            // withCredentials([string(credentialsId: 'db-password', variable: 'DB_PASSWORD_SECRET')]) {
                            //     sh "mvn flyway:migrate -Dflyway.url=${env.DB_URL} -Dflyway.user=${env.DB_USER} -Dflyway.password=${env.DB_PASSWORD_SECRET} -f pom.xml"
                            // }
                            // Или если используете переменные окружения, настроенные в env:
                            sh "mvn flyway:migrate -f pom.xml"

                        }
                        echo 'Миграции БД применены.'
                    }
                }

                // СУЩЕСТВУЮЩИЙ ЭТАП: Deploy to Staging
                stage('Deploy to Staging') {
                    steps {
                        echo "Запуск деплоя версии ${env.APP_VERSION} на Staging (${env.STAGING_SSH_HOST})..."
                        // Обновите скрипт деплоя или service unit, чтобы он использовал
                        // переменные окружения DB_URL, DB_USER, DB_PASSWORD, MONGODB_URI,
                        // которые теперь определены на уровне пайплайна Jenkins.

                        // Передача переменных окружения при запуске скрипта через ssh
                        sh """
                          ssh ${env.STAGING_SSH_USER}@${env.STAGING_SSH_HOST} << 'EOF' # Используйте одинарные кавычки для EOF, чтобы переменные раскрывались локально на агенте Jenkins
                            export SPRING_DATASOURCE_URL="${env.DB_URL}"
                            export SPRING_DATASOURCE_USERNAME="${env.DB_USER}"
                            export SPRING_DATASOURCE_PASSWORD="${env.DB_PASSWORD}" # Если передаете пароль так - ОПАСНО!
                            export SPRING_DATA_MONGODB_URI="${env.MONGODB_URI}" # Если передаете пароль так - ОПАСНО!
                            # Получить пароли безопасно на удаленной машине сложнее, возможно, через Vault (Урок 7)
                            # или передавая Credential ID и используя скрипт на удаленной машине для получения секрета.

                            cd ${env.STAGING_APP_DIR} # Переходим в каталог приложения на Staging VM

                            # ... (логика остановки текущего процесса из скрипта deploy_app_to_staging.sh) ...

                            # Запустить новую версию приложения в фоне
                            nohup java -jar ${env.STAGING_APP_DIR}/my-app-1.0-SNAPSHOT.jar > ${env.STAGING_APP_DIR}/${env.APP_NAME}.log 2>&1 &
                            # ... (логика сохранения версии, проверки запуска) ...
                            echo "Application started successfully."
                      EOF
                      """
                        echo 'Деплой на Staging завершен.'
                    }
                }

                // ... (Manual Approval stage) ...
            }

            post {
                 // ... (post actions) ...
            }
        }
        ```
        *   **Важно по безопасности:** Передача паролей как переменных окружения через команду `ssh` в скрипте Jenkinsfile **небезопасна**, т.к. пароль будет виден в логах команды. Гораздо лучше:
            *   Использовать Jenkins Credentials Binding Плагин для инжекции секрета как переменной окружения только для конкретного шага/блока.
            *   Использовать HashiCorp Vault (Урок 7) и настроить приложение/скрипт запуска на Staging VM для получения секретов из Vault.
            *   Использовать Ansible Vault для шифрования секретов в Ansible переменных и передавать их при запуске Ansible.

    2.  **Настройте Credential для пароля БД в Jenkins:** Перейдите "Manage Jenkins" -> "Manage Credentials" -> Global -> "Add Credentials". Тип: "Secret text". ID: `db-password` (или другой, который вы используете в `Jenkinsfile`). Secret: пароль пользователя БД.

    3.  **Создайте вторую миграцию Flyway:** Чтобы проверить, что миграции применяются инкрементально.
        *   Создайте файл `V2__add_product_table.sql` в `my-app/src/main/resources/db/migration`.
        ```sql
        -- V2__add_product_table.sql
        CREATE TABLE products (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            price DECIMAL(10, 2) NOT NULL,
            created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
        );
        ```

    4.  **Сделайте коммит и пуш:**
        ```bash
        git add Jenkinsfile my-app/src/main/resources/db/migration/V2__add_product_table.sql
        git commit -m "feat: Add Flyway stage to pipeline and V2 migration"
        git push origin develop
        ```

    5.  **Запустите пайплайн и проверьте выполнение:**
        *   Дождитесь завершения пайплайна.
        *   Просмотрите логи этапа "Database Migrations". Вы должны увидеть вывод Flyway о применении миграции `V1` (при первом запуске) и `V2` (при втором и последующих запусках). При повторном запуске `V1` не должна применяться.
        *   Подключитесь к PostgreSQL на DB VM (`psql`). Проверьте, что таблицы `users`, `products` и `flyway_schema_history` существуют (`\dt`). Посмотрите содержимое `flyway_schema_history`, там должны быть записи о примененных миграциях.

**Результат Lab 5:** Ваш Delivery Pipeline теперь автоматически выполняет миграции схемы PostgreSQL перед деплоем, используя Flyway. Это критически важный компонент DBOps, гарантирующий, что приложение деплоится на базу данных с ожидаемой структурой.

---

**Резюме Урока 6:**

Вы углубили свои знания о базах данных, изучив основы реляционных и нереляционных систем, а главное — освоили принципы DBOps и инструменты миграции БД. Интегрировав Flyway в ваш CI/CD пайплайн, вы автоматизировали процесс управления изменениями схемы базы данных, что значительно повышает надежность и безопасность процесса доставки ПО.

Теперь ваш конвейер включает автоматическую подготовку инфраструктуры, миграцию базы данных, сборку, проверку и деплой приложения.

**Что дальше?**

В следующем уроке мы перейдем к контейнеризации с использованием Docker. Мы упакуем наше приложение в Docker образ, научимся работать с хранилищами образов и, что очень важно, освоим HashiCorp Vault для безопасного централизованного хранения секретов, чтобы избавиться от передачи паролей через переменные окружения в скриптах.

Готовьтесь к миру контейнеров!