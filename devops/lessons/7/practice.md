Отлично! Контейнеризация с Docker - это стандарт де-факто в современной DevOps практике. В этом уроке мы упакуем наше приложение в Docker образ, научимся управлять этими образами и решим важную проблему безопасного хранения и доступа к секретам.

---

**УРОК 7: Docker-контейнеризация и Хранение Данных (Docker, MinIO, Vault)**

**Цель:** Освоить основы работы с Docker для создания, сборки и запуска контейнеров. Познакомиться с объектными хранилищами на примере MinIO. Изучить принципы безопасного управления секретами и интегрировать HashiCorp Vault.

**Результат урока:** Ваше приложение будет запускаться в Docker контейнере. CI/CD пайплайн будет собирать Docker образ и публиковать его в репозиторий образов. Приложение научится получать свои секреты из Vault, а вы сможете использовать объектное хранилище для файлов.

**Предварительные требования:**

*   Успешно завершен Урок 1-6.
*   Рабочий CI/CD пайплайн.
*   Staging VM, настроенная в Уроке 5, теперь должна иметь установленный Docker.
*   DB VM с запущенными PostgreSQL и MongoDB.
*   Дополнительные ресурсы на VM или новых VM для развертывания MinIO и Vault.

**Техническая подготовка:**

1.  **Установите Docker на Staging VM:** Если вы использовали Ansible для настройки VM (Урок 5, Lab 5), добавьте установку Docker в ваш playbook.
    ```yaml
    # В вашем playbook_app_setup.yml или отдельном playbook
    - name: Install Docker
      package:
        name: docker.io # или docker-ce для Docker Inc. репозитория
        state: present

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes

    - name: Ensure app_user is in docker group (if app runs containers)
      # Это нужно, если пользователь app_user будет запускать команды docker
      # В нашем случае Jenkins/Runner будет запускать деплой, но хорошая практика
      user:
        name: "{{ app_user }}"
        groups: docker
        append: yes # Добавить пользователя в группу, а не заменить группы

    # После этой задачи потребуется новое подключение пользователя для применения изменений группы
    # Или можно добавить команду 'newgrp docker' перед docker командами в скрипте деплоя
    ```
    *   Если устанавливаете Docker CE из официального репозитория, следуйте инструкциям на docs.docker.com для вашего дистрибутива.
    *   Перезапустите Staging VM или выйдите/войдите пользователем, который запускает Docker команды, чтобы членство в группе `docker` применилось.

2.  **Установите Docker на Jenkins/Runner VM:** Если вы будете собирать Docker образ на агенте Jenkins или Runner, там тоже нужен Docker.

3.  **Установите Docker Compose:** На VM, где будете развертывать MinIO и Vault (можно использовать DB VM или новую VM).

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **Контейнеризация:** Легковесная альтернатива полной виртуализации. Использует возможности ядра ОС (например, Linux cgroups и namespaces) для изоляции процессов друг от друга и от хост-системы. Контейнер включает приложение и все его зависимости (библиотеки, конфигурационные файлы).
    *   **chroot, jails, LXC:** Исторические шаги к контейнеризации, предоставляли частичную изоляцию. Docker сделал контейнеризацию доступной и удобной.
    *   **Контейнеры vs VM:** Контейнеры запускаются на общем ядре ОС, быстрее стартуют, занимают меньше ресурсов, более портативны (при условии совместимости ОС). VM эмулируют оборудование, имеют свое ядро ОС, лучше изолированы, но требуют больше ресурсов и медленнее стартуют.

*   **Docker:** Самая популярная платформа для работы с контейнерами.
    *   **Демон Docker (dockerd):** Работает на хост-системе, управляет образами, контейнерами, сетями, томами.
    *   **Клиент Docker (docker CLI):** Инструмент командной строки для взаимодействия с демоном.
    *   **Образ (Image):** Шаблон только для чтения, содержащий код приложения, библиотеки, зависимости, конфигурацию. Состоит из слоев (layers), каждый слой - результат инструкции в Dockerfile. Слои кешируются и переиспользуются.
    *   **Контейнер (Container):** Запущенный экземпляр образа. Включает образ и тонкий записываемый слой поверх. Изолирован от хоста и других контейнеров по умолчанию.
    *   **Dockerfile:** Текстовый файл с инструкциями для сборки Docker образа.

*   **Dockerfile Инструкции:**
    *   `FROM`: Базовый образ (операционная система, язык).
    *   `RUN`: Выполнить команду во время сборки образа.
    *   `COPY` / `ADD`: Скопировать файлы/каталоги с хоста в образ.
    *   `WORKDIR`: Установить рабочий каталог для последующих инструкций (`RUN`, `CMD`, `ENTRYPOINT`).
    *   `EXPOSE`: Объявить порт, который приложение слушает внутри контейнера.
    *   `ENV`: Установить переменные окружения во время сборки/выполнения.
    *   `ARG`: Определить переменные сборки (используются только во время `docker build`).
    *   `VOLUME`: Объявить точку монтирования для Volume.
    *   `CMD`: Команда по умолчанию для выполнения при запуске контейнера. Легко переопределяется при `docker run`.
    *   `ENTRYPOINT`: Команда, которая всегда выполняется при запуске контейнера. Аргументы из `docker run` передаются ей как параметры. Часто используется для запуска исполняемого файла приложения.
    *   **Мультистейдж сборка (Multi-stage build):** Использование нескольких инструкций `FROM` в одном Dockerfile. Позволяет использовать один образ для сборки приложения (например, с JDK и Maven), а другой, более легковесный, для запуска готового артефакта (например, с Alpine Linux и JRE). Уменьшает размер финального образа.

*   **Docker Hub и Container Registries:** Централизованные или приватные хранилища для Docker образов. Позволяют делиться образами, контролировать доступ, версионировать образы. Nexus Repository Manager (который у нас уже есть) и GitLab Container Registry могут хостить Docker образы.

*   **Docker Networking:**
    *   `bridge` (по умолчанию): Контейнеры в одной сети могут общаться по имени. Доступ извне хоста через проброс портов (`-p`).
    *   `host`: Контейнер использует сетевой стек хоста. Нет изоляции сети.
    *   `none`: Нет сети.

*   **Docker Volumes:** Рекомендуемый способ управления персистентными данными контейнеров. Данные хранятся вне записываемого слоя контейнера, в специальном каталоге на хосте, управляемом Docker. Данные в Volume сохраняются при удалении контейнера.

*   **Альтернативы Docker:** Podman (аналогичный Docker CLI, без демона, запускает контейнеры как обычные процессы), containerd/CRI-O (низкоуровневые Container Runtimes, используемые в Kubernetes).

*   **Объектные хранилища:** Системы хранения данных, которые управляют данными как дискретными единицами (объектами) с метаданными, доступными через API (часто S3 API). Не имеют иерархии файловой системы в традиционном понимании. Хорошо подходят для хранения неструктурированных данных (документы, медиафайлы, бэкапы, артефакты сборки). Примеры: AWS S3, Google Cloud Storage, Azure Blob Storage, MinIO.

*   **MinIO:** Высокопроизводительное S3-совместимое объектное хранилище с открытым исходным кодом. Легко разворачивается (в т.ч. в Docker).

*   **Безопасное хранение секретов:** Проблема хранения чувствительных данных (пароли к БД, API ключи, приватные ключи) в коде, файлах конфигурации или переменных окружения (которые могут попасть в логи или историю команд).

*   **HashiCorp Vault:** Инструмент для безопасного хранения, управления и предоставления доступа к секретам.
    *   **Архитектура:** Сервер Vault, который взаимодействует с различными Storage Backends (куда физически хранятся зашифрованные секреты) и Secret Engines (интерфейсы для работы с разными типами секретов) и Auth Methods (способы аутентификации клиентов Vault).
    *   **Инициализация (Initialization):** Первый запуск Vault сервера, создает мастер-ключ (Master Key) и ключи разблокировки (Unseal Keys). Сервер после старта находится в "запечатанном" (Sealed) состоянии и не может расшифровать данные, пока не будет "разблокирован" (Unsealed) с помощью части ключей разблокировки. Повышает безопасность.
    *   **Секретные движки (Secret Engines):** Определяют, как Vault управляет секретами (статические KV пары, динамические учетки к БД, сертификаты, ключи шифрования). `kv` - простой движок для хранения Key-Value пар.
    *   **Методы аутентификации (Auth Methods):** Определяют, как клиенты (пользователи, приложения, машины) аутентифицируются в Vault (по логину/паролю, токену, GitHub, Kubernetes Service Account, AWS IAM и др.). После аутентификации клиент получает временный токен доступа к Vault.

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Контейнеризация Приложения с Dockerfile**

*   **Цель:** Создать Docker образ для вашего Java приложения и запустить его в контейнере.
*   **Предварительные требования:**
    *   Работающее приложение, собранное Maven (JAR/WAR).
    *   Установленный Docker на вашей рабочей машине или Jenkins/Runner VM.
    *   Локальный Git репозиторий вашего приложения.
*   **Шаги:**

    1.  **Создайте файл `Dockerfile` в корне вашего Git репозитория:** (там, где лежит `pom.xml` и каталог `my-app`).
        ```bash
        cd my-devops-app
        nano Dockerfile
        ```
    2.  **Напишите Dockerfile (пример с мультистейдж сборкой для Java/Maven):**

        ```dockerfile
        # Dockerfile

        # Этап 1: Сборка приложения (build stage)
        # Используем образ Maven с JDK для сборки
        FROM maven:3.8.6-openjdk-11 AS builder # Присваиваем имя этому этапу 'builder'

        # Устанавливаем рабочий каталог внутри контейнера
        WORKDIR /app

        # Копируем файлы Maven проекта. Сначала pom.xml для кеширования зависимостей.
        COPY my-app/pom.xml .
        # Скачиваем зависимости (если они еще не в кеше). Это отдельный слой, который кешируется.
        RUN mvn dependency:go-offline

        # Копируем остальной исходный код
        COPY my-app/src ./src

        # Собираем приложение. Используем --no-transfer-progress для более чистых логов.
        RUN mvn package -DskipTests --no-transfer-progress

        # Этап 2: Запуск приложения (run stage)
        # Используем более легковесный образ с только JRE
        FROM openjdk:11-jre-slim # Или alpine, если JRE есть

        # Устанавливаем рабочий каталог
        WORKDIR /app

        # Копируем собранный JAR/WAR файл из build stage
        COPY --from=builder /app/target/my-app-1.0-SNAPSHOT.jar myapp.jar # Замените на имя вашего артефакта

        # Объявляем порт, который слушает приложение (если это веб-приложение)
        EXPOSE 8080 # Укажите порт вашего приложения

        # Определяем команду для запуска приложения при старте контейнера
        # ENTRYPOINT предпочтительнее CMD для запуска исполняемых файлов
        ENTRYPOINT ["java", "-jar", "myapp.jar"]

        # CMD может использоваться для передачи аргументов ENTRYPOINT или как команда по умолчанию
        # CMD ["--server.port=8080"] # Пример передачи аргумента Spring Boot
        ```
        *   **Важно:** Убедитесь, что имена файлов и пути (`my-app/pom.xml`, `my-app/src`, `my-app/target/my-app-1.0-SNAPSHOT.jar`) соответствуют структуре вашего проекта. Укажите правильный порт, если приложение его использует.

    3.  **Соберите Docker образ:** Убедитесь, что вы находитесь в корне Git репозитория (`my-devops-app`), где находится `Dockerfile`.
        ```bash
        docker build -t my-java-app:latest . # -t для тега (имя:версия), . для контекста сборки (текущий каталог)
        # Или используйте тег с номером версии/билда, что лучше:
        # docker build -t my-java-app:1.0.0-SNAPSHOT .
        ```
        Процесс сборки покажет выполнение каждой инструкции Dockerfile как отдельного шага, создавая слои.

    4.  **Проверьте собранный образ:**
        ```bash
        docker images
        ```
        Вы должны увидеть образ `my-java-app` с указанным тегом.

    5.  **Запустите контейнер из образа:**
        ```bash
        docker run -d -p 8080:8080 --name my-app-container my-java-app:latest # -d для запуска в фоне, -p для проброса портов
        ```
        *   `-p 8080:8080`: Пробросить порт 8080 из контейнера на порт 8080 на хост-машине.
        *   `--name my-app-container`: Присвоить имя контейнеру.

    6.  **Проверьте запущенный контейнер:**
        ```bash
        docker ps # Посмотреть список запущенных контейнеров
        docker logs my-app-container # Посмотреть логи контейнера
        ```
        Если приложение веб-сервис, попробуйте получить к нему доступ с хост-машины или другой машины в сети хоста (если файрвол позволяет) по адресу `http://localhost:8080` (или IP_ХОСТА:8080).

    7.  **Остановите и удалите контейнер:**
        ```bash
        docker stop my-app-container
        docker rm my-app-container
        ```
        *   *Не удаляйте образ (`docker rmi my-java-app:latest`), он понадобится в следующей Lab.*

**Результат Lab 1:** Вы успешно создали Dockerfile для вашего приложения, собрали из него образ и запустили приложение в изолированном контейнере.

---

**Lab 2: Работа с Container Registry (Nexus/Docker Hub)**

*   **Цель:** Опубликовать собранный Docker образ в репозитории образов.
*   **Предварительные требования:**
    *   Собранный Docker образ (Lab 1).
    *   Доступ к Container Registry (Nexus или Docker Hub).
*   **Использование Nexus как Docker Registry:** Это удобно, т.к. Nexus уже установлен.

    1.  **Настройте Nexus для хостинга Docker образов:**
        *   Перейдите в веб-интерфейс Nexus (Lab 1 Урока 4).
        *   "Server administration" (шестеренка) -> "Repositories".
        *   Нажмите "Create repository".
        *   Выберите тип "docker (hosted)".
        *   Name: `docker-private` (или любое другое имя).
        *   Version Policy: `Release` (для стабильных образов) или `Snapshot` (для разрабатываемых).
        *   Deployment policy: `Allow redeploy` (полезно для SNAPSHOT).
        *   **HTTP Port:** Укажите порт, по которому будет доступен этот репозиторий (например, `8082`). *Важно:* Этот порт должен быть проброшен из Docker контейнера Nexus на хост VM в вашем `docker-compose.yml` (см. Lab 1 Урока 4, закомментированные порты). Остановите и перезапустите Nexus с добавленными портами.
        *   HTTPS Port: (Опционально, требует SSL/TLS).
        *   Allow anonymous docker pull: Снимите галочку (для приватного репозитория).
        *   Нажмите "Create repository".

    2.  **Настройте Docker клиент для работы с приватным реестром Nexus:** Docker клиент на вашей рабочей машине или Jenkins/Runner VM должен доверять вашему Nexus и знать, как аутентифицироваться.
        *   **Логин в Nexus Docker Registry:** Используйте учетные данные пользователя Nexus (например, admin).
            ```bash
            docker login ВАШ_IP_VM_NEXUS:8082 # Укажите IP VM и порт Nexus Docker registry
            Username: admin
            Password: ваш_пароль_nexus
            ```
            Если логин прошел успешно, учетные данные будут сохранены (обычно в `~/.docker/config.json`).

    3.  **Пометьте (tag) ваш локальный образ для публикации в Nexus:** Формат тега: `имя_реестра/имя_образа:тег`.
        ```bash
        docker tag my-java-app:latest ВАШ_IP_VM_NEXUS:8082/my-java-app:1.0.0-SNAPSHOT # Пример для SNAPSHOT версии
        # Или для конкретной сборки/коммита:
        # docker tag my-java-app:latest ВАШ_IP_VM_NEXUS:8082/my-java-app:build-123
        ```

    4.  **Опубликуйте (push) образ в Nexus:**
        ```bash
        docker push ВАШ_IP_VM_NEXUS:8082/my-java-app:1.0.0-SNAPSHOT # Используйте тег, который только что создали
        ```
        Вы увидите процесс загрузки слоев образа.

    5.  **Проверьте образ в Nexus:** Перейдите в веб-интерфейс Nexus, "Browse", найдите ваш новый Docker Hosted репозиторий (`docker-private`). Вы должны увидеть опубликованный образ.

    6.  **Удалите локальный образ и скачайте его из Nexus (для проверки):**
        ```bash
        docker rmi ВАШ_IP_VM_NEXUS:8082/my-java-app:1.0.0-SNAPSHOT # Удалить образ по тегу
        docker pull ВАШ_IP_VM_NEXUS:8082/my-java-app:1.0.0-SNAPSHOT # Скачать из Nexus
        ```
        Если скачивание успешно, значит образ корректно опубликован.

*   **Использование Docker Hub или GitLab Container Registry:**
    *   **Docker Hub:** Потребуется создать аккаунт. Логин: `docker login`. Тег: `your_dockerhub_username/my-java-app:tag`. Пуш: `docker push your_dockerhub_username/my-java-app:tag`.
    *   **GitLab Container Registry:** В каждом проекте GitLab есть встроенный Container Registry. URL: `gitlab.com/your_username/your_project_name/my-java-app:tag`. Логин: `docker login registry.gitlab.com`. Пуш: `docker push registry.gitlab.com/your_username/your_project_name/my-java-app:tag`.

**Результат Lab 2:** Вы научились публиковать Docker образы в приватный репозиторий (Nexus или другой) и скачивать их оттуда.

---

**Lab 3: Практика Docker Volumes**

*   **Цель:** Использовать Docker Volumes для сохранения данных (например, логов приложения) вне контейнера.
*   **Предварительные требования:**
    *   Dockerfile для вашего приложения (Lab 1).
    *   Умение запускать Docker контейнеры.
    *   Ваше приложение должно писать логи в файл в определенном каталоге внутри контейнера.
*   **Шаги:**

    1.  **Убедитесь, что приложение пишет логи в файл:** Модифицируйте конфигурацию логирования вашего Java приложения (например, Logback, Log4j), чтобы оно писало логи в файл в определенном каталоге внутри контейнера (например, `/app/logs/myapp.log`). Убедитесь, что этот каталог существует и доступен для записи пользователю, от имени которого запускается приложение внутри контейнера.

    2.  **Обновите Dockerfile (опционально, но рекомендуется):** Объявите каталог с логами как VOLUME. Это подсказка Docker'у, что этот каталог предназначен для внешнего монтирования.
        ```dockerfile
        # ... (run stage) ...
        WORKDIR /app
        COPY --from=builder /app/target/my-app-1.0-SNAPSHOT.jar myapp.jar
        EXPOSE 8080

        # Объявляем каталог для логов как VOLUME
        VOLUME /app/logs

        ENTRYPOINT ["java", "-jar", "myapp.jar"]
        ```

    3.  **Пересоберите образ с обновленным Dockerfile:**
        ```bash
        docker build -t my-java-app:latest . # Или с новым тегом
        ```

    4.  **Запустите контейнер, смонтировав Volume:**
        *   **Использование именованного Volume (предпочтительно):**
            ```bash
            docker volume create myapp_logs_volume # Создать именованный Volume
            docker run -d -p 8080:8080 --name my-app-container -v myapp_logs_volume:/app/logs my-java-app:latest
            ```
            `myapp_logs_volume` - имя Volume, `/app/logs` - путь внутри контейнера.
        *   **Использование bind mount (монтирование каталога с хоста):**
            ```bash
            mkdir $HOME/app_logs_on_host # Создать каталог на хосте
            docker run -d -p 8080:8080 --name my-app-container -v $HOME/app_logs_on_host:/app/logs my-java-app:latest
            ```
            `$HOME/app_logs_on_host` - путь на хосте, `/app/logs` - путь внутри контейнера.

    5.  **Проверьте, что логи пишутся в Volume:**
        *   Выполните какие-либо действия с приложением, чтобы оно сгенерировало логи.
        *   Если использовали именованный Volume: `docker volume inspect myapp_logs_volume` найдите путь `Mountpoint`. Просмотрите содержимое этого каталога на хосте.
        *   Если использовали bind mount: Просмотрите содержимое каталога `$HOME/app_logs_on_host`. Вы должны увидеть файл логов вашего приложения.

    6.  **Остановите, удалите контейнер и запустите новый:**
        ```bash
        docker stop my-app-container
        docker rm my-app-container
        # Запустите новый контейнер с тем же Volume
        docker run -d -p 8080:8080 --name my-app-container-new -v myapp_logs_volume:/app/logs my-java-app:latest
        ```
        Проверьте логи в смонтированном Volume. Логи из предыдущего запуска контейнера должны быть там, демонстрируя персистентность данных.

**Результат Lab 3:** Умение использовать Docker Volumes для персистентного хранения данных контейнеров, отделяя данные от жизненного цикла самого контейнера.

---

**Lab 4: Установка MinIO**

*   **Цель:** Развернуть S3-совместимое объектное хранилище для практики.
*   **Предварительные требования:**
    *   Установленный Docker Compose на VM (можно использовать DB VM или новую VM).
*   **Шаги:**

    1.  **Создайте каталог для MinIO и файл `docker-compose.yml`:**
        ```bash
        mkdir $HOME/minio
        cd $HOME/minio
        nano docker-compose.yml
        ```
    2.  **Добавьте следующее содержимое:**

        ```yaml
        version: "3.8"

        services:
          minio:
            image: minio/minio:latest # Используйте актуальную версию
            container_name: minio-s3
            ports:
              - "9000:9000" # Консоль MinIO
              - "9001:9001" # S3 API порт (для версии > RELEASE.2020-12-03T00-19-17Z)
            environment:
              MINIO_ROOT_USER: minioadmin # Имя root пользователя S3
              MINIO_ROOT_PASSWORD: minioadminpassword # ПАРОЛЬ! ИЗМЕНИТЕ В PRODUCTION!
              MINIO_DOMAIN: ВАШ_IP_VM_MINIO # IP или домен VM, где запущен MinIO
            command: server /data --console-address ":9001" # Запускаем сервер и указываем порт консоли
            volumes:
              - minio_data:/data # Персистентное хранение данных
            restart: unless-stopped
            healthcheck: # Проверка работоспособности
              test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"] # Проверка порта API
              interval: 30s
              timeout: 20s
              retries: 3

        volumes:
          minio_data:
        ```
        *   Замените `ВАШ_IP_VM_MINIO` на IP адрес VM, где запущен MinIO. Это важно для корректной работы S3 SDK/CLI.

    3.  **Запустите контейнер MinIO:**
        ```bash
        docker-compose up -d
        ```
        Подождите немного.

    4.  **Доступ к консоли MinIO:** Откройте браузер и перейдите по адресу `http://ВАШ_IP_VM_MINIO:9001`. Войдите с `minioadmin`/`minioadminpassword`.

    5.  **Создайте бакет (bucket):** В консоли MinIO нажмите "+" -> "Create Bucket". Дайте бакету имя (например, `app-files`).

    6.  **Установите MinIO Client (mc) или AWS CLI:** Это инструменты командной строки для работы с S3-совместимыми хранилищами.
        *   **MinIO Client (mc):** Следуйте инструкциям на docs.min.io для установки.
        *   **AWS CLI:** Следуйте инструкциям на docs.aws.amazon.com для установки. Настройте его (`aws configure`), но вместо данных AWS введите данные MinIO:
            *   AWS Access Key ID: `minioadmin`
            *   AWS Secret Access Key: `minioadminpassword`
            *   Default region name: `us-east-1` (любой регион)
            *   Default output format: `json`
            *   *Дополнительно для AWS CLI:* Создайте файл настроек MinIO: `nano ~/.aws/config` и добавьте:
                ```ini
                [plugins]
                endpoint = awscli_plugin_endpoint
                [profile minio] # Имя профиля
                region = us-east-1
                output = json
                endpoint_url = http://ВАШ_IP_VM_MINIO:9000 # Порт API MinIO
                s3 =
                  signature_version = s3v4
                ```
                Теперь вы можете использовать AWS CLI с профилем Minio: `aws --profile minio s3 ls`.

    7.  **Проверьте работу с MinIO через CLI:**
        *   **mc:**
            ```bash
            mc alias set myminio http://ВАШ_IP_VM_MINIO:9000 minioadmin minioadminpassword --api s3v4 # Добавить алиас
            mc ls myminio # Посмотреть бакеты
            echo "test object content" > test.txt
            mc cp test.txt myminio/app-files/ # Загрузить объект
            mc ls myminio/app-files/ # Посмотреть объекты в бакете
            mc cat myminio/app-files/test.txt # Скачать и вывести содержимое объекта
            mc rm myminio/app-files/test.txt # Удалить объект
            ```
        *   **aws cli:**
            ```bash
            aws --profile minio s3 ls # Посмотреть бакеты
            echo "test object content" > test.txt
            aws --profile minio s3 cp test.txt s3://app-files/ # Загрузить объект
            aws --profile minio s3 ls s3://app-files/ # Посмотреть объекты в бакете
            aws --profile minio s3 cp s3://app-files/test.txt downloaded_test.txt # Скачать объект
            cat downloaded_test.txt
            aws --profile minio s3 rm s3://app-files/test.txt # Удалить объект
            ```

**Результат Lab 4:** Установленный и работающий MinIO сервер, умение создавать бакеты и взаимодействовать с объектным хранилищем с помощью S3 API через командную строку.

---

**Lab 5: Установка HashiCorp Vault**

*   **Цель:** Развернуть Vault для безопасного хранения секретов.
*   **Предварительные требования:**
    *   Установленный Docker Compose на VM (можно использовать DB VM или новую VM).
*   **Шаги:**

    1.  **Создайте каталог для Vault и файл `docker-compose.yml`:**
        ```bash
        mkdir $HOME/vault
        cd $HOME/vault
        nano docker-compose.yml
        ```
    2.  **Добавьте следующее содержимое (простой вариант с файловым хранилищем для Lab):**

        ```yaml
        version: "3.8"

        services:
          vault:
            image: vault:latest # Используйте актуальную версию
            container_name: vault-server
            ports:
              - "8200:8200" # Порт API и UI Vault
            environment:
              # VAULT_DEV_ROOT_TOKEN_ID: myroot # Для режима разработки, создает root токен
              VAULT_LOCAL_CONFIG: '{
                "storage": {
                  "file": { # Используем файловое хранилище. НЕ ДЛЯ PRODUCTION!
                    "path": "/vault/file"
                  }
                },
                "listener": {
                  "tcp": {
                    "address": "0.0.0.0:8200",
                    "tls_disable": true # Отключаем TLS для простоты. НЕ ДЛЯ PRODUCTION!
                  }
                },
                "api_addr": "http://ВАШ_IP_VM_VAULT:8200", # IP или домен VM, где запущен Vault
                "cluster_addr": "http://ВАШ_IP_VM_VAULT:8201" # IP или домен VM
              }'
            volumes:
              - vault_data:/vault/file # Для файлового хранилища
            restart: unless-stopped
            cap_add: # Требуется для режима production с некоторыми storage backends (например, sealed secrets)
              - IPC_LOCK
            # command: server -dev -dev-root-token-id=myroot # Если хотите запустить в режиме разработки
            command: server # Запускаем в production режиме (требует инициализации и разблокировки)

        volumes:
          vault_data:
        ```
        *   Замените `ВАШ_IP_VM_VAULT` на IP адрес VM, где запущен Vault.
        *   **Важно:** Конфигурация с файловым хранилищем (`file`) и отключенным TLS (`tls_disable: true`) **НЕБЕЗОПАСНА И НЕ ПОДХОДИТ ДЛЯ PRODUCTION!** В production используются зашифрованные storage backends (Consul, ZooKeeper, S3, GCS, Azure Blob Storage) и обязательно TLS.

    3.  **Запустите контейнер Vault:**
        ```bash
        docker-compose up -d
        ```

    4.  **Инициализируйте Vault:** При первом запуске в production режиме, Vault запечатан (sealed). Его нужно инициализировать, чтобы получить ключи разблокировки и начальный root токен.
        *   Подключитесь к контейнеру Vault или выполните команду внутри него:
            ```bash
            docker exec -it vault-server vault operator init
            ```
        *   **ВНИМАТЕЛЬНО ПРОЧТИТЕ ВЫВОД!** Vault выдаст:
            *   **Unseal Keys:** Список ключей разблокировки (обычно 5). Для разблокировки по умолчанию требуется 3 из 5 ключей (это настраивается - Shamir's Secret Sharing). **СОХРАНИТЕ ЭТИ КЛЮЧИ В НАДЕЖНОМ МЕСТЕ!**
            *   **Initial Root Token:** Начальный root токен. Он имеет полные права на Vault. **СОХРАНИТЕ ЭТОТ ТОКЕН В НАДЕЖНОМ МЕСТЕ!** Он нужен для первого входа и настройки других методов аутентификации.
        *   **Это критически важные данные! Потеря ключей разблокировки может сделать ваши секреты недоступными! Потеря root токена может заблокировать доступ к управлению Vault!**

    5.  **Разблокируйте (Unseal) Vault:** После инициализации Vault нужно разблокировать. Это нужно делать каждый раз после старта контейнера или перезагрузки VM (если не настроен авто-разблокировка).
        ```bash
        docker exec -it vault-server vault operator unseal
        # Вас попросят ввести Key 1 (один из ключей из вывода init). Введите.
        docker exec -it vault-server vault operator unseal
        # Вас попросят ввести Key 2. Введите.
        docker exec -it vault-server vault operator unseal
        # Вас попросят ввести Key 3. Введите (нужно 3 из 5 по умолчанию).
        ```
        После ввода достаточного количества ключей, Vault перейдет в состояние `Sealed: false`.

    6.  **Войдите в Vault (с помощью Root Token):**
        *   Установите Vault CLI на вашей VM (см. документацию Vault) или используйте `docker exec`.
        *   Экспортируйте адрес Vault и root токен:
            ```bash
            export VAULT_ADDR="http://ВАШ_IP_VM_VAULT:8200"
            export VAULT_TOKEN="ваш_начальный_root_токен" # Скопируйте из вывода init
            ```
        *   Выполните команду входа:
            ```bash
            vault login $VAULT_TOKEN # Или просто vault login (если VAULT_TOKEN в env)
            # Убедитесь, что вы вошли
            vault status # Должно показать Sealed: false
            vault token lookup # Посмотреть информацию о текущем токене
            ```
        *   *Важно:* Не используйте root токен для повседневных задач или приложений! Создайте более ограниченные токены или используйте другие методы аутентификации.

    7.  **Доступ к UI Vault:** Откройте браузер и перейдите по адресу `http://ВАШ_IP_VM_VAULT:8200`. Войдите, используя ваш Initial Root Token.

**Результат Lab 5:** Установленный и настроенный Vault сервер, инициализирован и разблокирован. У вас есть Root Token для доступа к нему.

---

**Lab 6: Хранение Секретов в Vault**

*   **Цель:** Использовать движок KV Secret Engine для хранения учетных данных к БД и других секретов приложения.
*   **Предварительные требования:**
    *   Работающий и разблокированный Vault (Lab 5).
    *   Доступ к Vault CLI или UI.
    *   Initial Root Token.
*   **Шаги:**

    1.  **Включите KV Secret Engine (если не включен по умолчанию):** По умолчанию KV движок часто включен по пути `secret/`. Если нет, включите его.
        ```bash
        vault secrets enable kv # Включить движок KV по пути secret/
        # Или для версии 2 (с версионированием):
        # vault secrets enable -version=2 kv
        ```
        Проверьте включенные движки: `vault secrets list`.

    2.  **Запишите секреты в KV движок:**
        *   **Используя CLI:**
            ```bash
            vault kv put secret/myapp/config/staging db_url="jdbc:postgresql://IP_DB_VM:5432/myappdb" db_user="myappuser" db_password="mypassword_from_db_lab" mongodb_uri="mongodb://rootuser:rootpassword@IP_DB_VM:27017/myappmongodb?authSource=admin" # Замените данные
            ```
            `secret/myapp/config/staging` - это путь к секрету. Вы можете хранить несколько пар ключ-значение в одном секрете.
        *   **Используя UI:** Перейдите в "Secrets" -> "secret/" -> "Create secret". Укажите путь и добавьте пары ключ-значение.

    3.  **Прочитайте секреты из Vault:**
        *   **Используя CLI:**
            ```bash
            vault kv get secret/myapp/config/staging
            # Прочитать только конкретное поле:
            # vault kv get -field=db_password secret/myapp/config/staging
            ```
        *   **Используя UI:** Перейдите по пути к секрету.

    4.  **(Опционально) Создайте политику (Policy) для доступа к секретам:** Политики определяют, какие пути в Vault доступны для данного токена/сущности и какие операции разрешены (read, write, create, delete, list, sudo).
        *   Создайте файл политики: `nano myapp-staging-policy.hcl`
            ```hcl
            # myapp-staging-policy.hcl
            # Разрешить чтение секретов по пути secret/myapp/config/staging/*
            path "secret/data/myapp/config/staging/*" { # Используйте secret/data/... для KV v2
              capabilities = ["read", "list"] # Разрешить чтение и просмотр списка
            }
            # Если используете KV v1:
            # path "secret/myapp/config/staging/*" {
            #   capabilities = ["read", "list"]
            # }
            ```
        *   Загрузите политику в Vault:
            ```bash
            vault policy write myapp-staging myapp-staging-policy.hcl
            ```

    5.  **(Опционально) Создайте токен с ограниченными правами:** Сгенерируйте новый токен и привяжите к нему созданную политику. Этот токен можно использовать для доступа к секретам из приложения или CI/CD.
        ```bash
        vault token create -policy=myapp-staging -ttl=24h # Токен действителен 24 часа
        ```
        **Сохраните этот токен!** Он нужен будет приложению.

**Результат Lab 6:** Секреты вашего приложения безопасно хранятся в Vault и доступны через API или CLI. Умение создавать политики и ограниченные токены.

---

**Lab 7: Интеграция Приложения с Vault**

*   **Цель:** Изменить приложение, чтобы оно получало свои секреты из Vault при старте.
*   **Предварительные требования:**
    *   Работающий Vault с записанными секретами (Lab 6).
    *   Токен Vault с правами на чтение этих секретов.
    *   Ваше приложение, настроенное на чтение параметров из переменных окружения (Twelve-Factor App, Урок 4).
*   **Шаги:**

    1.  **Выберите способ интеграции:**
        *   **Непосредственно в коде приложения:** Добавить зависимость от Vault Java Client или Vault Spring Boot Starter и читать секреты при старте.
        *   **Скрипт-обертка (Wrapper script):** Написать скрипт, который запускается перед приложением. Скрипт получает секреты из Vault (используя Vault CLI или `curl` к API), экспортирует их как переменные окружения, а затем запускает приложение. **Это более DevOps-подход, т.к. не требует изменений в коде приложения.**

    2.  **Реализация через скрипт-обертку (рекомендуется для Lab):**
        *   Напишите скрипт (например, Bash) на вашей Jenkins/Runner VM или в репозитории инфраструктуры.
        ```bash
        #!/bin/bash

        # wrap_and_run.sh

        VAULT_ADDR="http://ВАШ_IP_VM_VAULT:8200" # Адрес Vault API
        # Vault токен для приложения. Передавать его безопасно - отдельная задача!
        # Для Lab можно захардкодить (НЕБЕЗОПАСНО) или передать через env var пайплайна.
        # В production токен обычно получается через методы аутентификации (Kubernetes auth, AppRole и др.)
        VAULT_TOKEN="${VAULT_TOKEN_VAR}" # Пример: получить из переменной окружения Jenkins/пайплайна

        SECRET_PATH="secret/myapp/config/staging" # Путь к секрету в Vault

        echo "Fetching secrets from Vault at ${VAULT_ADDR}..."

        # Используем curl для вызова Vault API
        # Нужен jq для парсинга JSON (sudo apt install jq -y)
        SECRETS_JSON=$(curl -s -H "X-Vault-Token: ${VAULT_TOKEN}" ${VAULT_ADDR}/v1/${SECRET_PATH})

        if [ \$? -ne 0 ]; then
          echo "Error: Failed to fetch secrets from Vault API."
          exit 1
        fi

        # Проверить на ошибку (например, если токен невалиден или путь неверный)
        if echo "${SECRETS_JSON}" | grep -q '"errors":'; then
           echo "Error fetching secrets from Vault: $(echo "${SECRETS_JSON}" | jq -r '.errors[]')"
           exit 1
        fi

        # Парсим JSON и экспортируем нужные поля как переменные окружения
        # Используем jq для безопасного парсинга
        export SPRING_DATASOURCE_URL=$(echo "${SECRETS_JSON}" | jq -r '.data.data.db_url') # .data.data для KV v2
        export SPRING_DATASOURCE_USERNAME=$(echo "${SECRETS_JSON}" | jq -r '.data.data.db_user')
        export SPRING_DATASOURCE_PASSWORD=$(echo "${SECRETS_JSON}" | jq -r '.data.data.db_password')
        export SPRING_DATA_MONGODB_URI=$(echo "${SECRETS_JSON}" | jq -r '.data.data.mongodb_uri')

        # Проверка, что переменные не пустые
        if [ -z "\$SPRING_DATASOURCE_URL" ] || [ -z "\$SPRING_DATASOURCE_USERNAME" ] || [ -z "\$SPRING_DATASOURCE_PASSWORD" ] || [ -z "\$SPRING_DATA_MONGODB_URI" ]; then
          echo "Error: One or more secrets were not fetched from Vault."
          # Могут быть пустые строки, если секрет не найден, или jq не нашел поле
          exit 1
        fi

        echo "Secrets fetched and exported as environment variables."

        # Выполняем исходную команду запуска приложения
        # Приложение получит переменные окружения, экспортированные выше
        echo "Starting application..."
        exec java -jar myapp.jar # Используйте exec, чтобы заменить текущий процесс скрипта на процесс java
        ```
        *   Сделайте скрипт исполняемым: `chmod +x wrap_and_run.sh`.
        *   Установите `jq` на Staging VM, если его там нет.

    3.  **Обновите Jenkinsfile и скрипт деплоя:**
        *   Добавьте скрипт `wrap_and_run.sh` в ваш репозиторий (лучше в репозиторий инфраструктуры или отдельный репозиторий деплоя).
        *   Скопируйте скрипт на Staging VM как часть Ansible Playbook (Lab 5 Урока 5).
        *   В Jenkinsfile:
            *   Определите переменные окружения `VAULT_ADDR` и `VAULT_TOKEN_VAR` (получите токен из Credential в Jenkins!). **Токен Vault - это СЕКРЕТ! Храните его в Jenkins Credentials!**
            *   Измените этап `Deploy to Staging`: вместо прямого запуска `java -jar ...` запускайте скрипт-обертку.

            ```groovy
            // Jenkinsfile (Declarative Pipeline)

            pipeline {
                agent any
                environment {
                    // ... (DB_URL, DB_USER, MONGODB_URI - эти переменные теперь будут устанавливаться wrap_and_run.sh) ...
                    // Удалите их из env pipeline, если ваше приложение их получало только отсюда
                    // Или оставьте, если нужно для других целей в пайплайне.

                    // Переменные для Vault (ИСПОЛЬЗУЙТЕ JENKINS CREDENTIALS!)
                    VAULT_ADDR = "http://ВАШ_IP_VM_VAULT:8200" // Адрес Vault
                    // Добавьте новый Secret Text Credential в Jenkins для Vault токена (ID: vault-app-token)
                    // VAULT_TOKEN_VAR = credentials('vault-app-token') // Безопасно!
                    VAULT_TOKEN_VAR = 'ваш_токен_из_lab6' // НЕБЕЗОПАСНО ДЛЯ ДЕМО! Используйте credentials!

                    // Путь к скрипту-обертке на Staging VM
                    WRAPPER_SCRIPT_PATH = "/opt/myjavaapp/wrap_and_run.sh" # Или где вы его скопировали
                }

                stages {
                    // ... (CI, Provision, Configure, DB Migrations этапы) ...

                     // СУЩЕСТВУЮЩИЙ ЭТАП: Deploy to Staging
                    stage('Deploy to Staging') {
                        steps {
                            echo "Запуск деплоя версии ${env.APP_VERSION} на Staging (${env.STAGING_SSH_HOST})..."
                            // Скопируйте скрипт-обертку на Staging VM, если еще не сделали это через Ansible
                            // sh "scp your_wrapper_script.sh ${env.STAGING_SSH_USER}@${env.STAGING_SSH_HOST}:/path/on/staging/"

                            sh """
                              ssh ${env.STAGING_SSH_USER}@${env.STAGING_SSH_HOST} << 'EOF'
                                cd ${env.STAGING_APP_DIR}

                                # ... (логика остановки текущего процесса) ...

                                # Запустить скрипт-обертку с переменными окружения
                                # Скрипт сам получит секреты и запустит приложение
                                export VAULT_ADDR="${env.VAULT_ADDR}"
                                export VAULT_TOKEN_VAR="${env.VAULT_TOKEN_VAR}" # ИСПОЛЬЗУЙТЕ БОЛЕЕ БЕЗОПАСНЫЙ СПОСОБ ПЕРЕДАЧИ СЕКРЕТОВ!

                                echo "Starting application using wrapper script..."
                                # nohup не нужен, если wrap_and_run.sh использует exec
                                # nohup ${WRAPPER_SCRIPT_PATH} > ${env.STAGING_APP_DIR}/${env.APP_NAME}.log 2>&1 &
                                ${WRAPPER_SCRIPT_PATH} # Запускаем скрипт-обертку

                                # ... (логика сохранения версии, проверки запуска) ...
                                echo "Application started successfully." # Этот echo не выполнится, если wrap_and_run.sh использует exec
                            EOF
                            """
                            echo 'Деплой на Staging завершен.'
                        }
                    }
                    // ...
                }
                // ...
            }
        }
        ```
        *   **Помните про безопасность передачи Vault токена в SSH сессию!** Это самый сложный момент. В production часто используются более сложные методы (например, Vault Agent с Auto-auth или получение секрета через Kubernetes Service Account Auth Method, если приложение работает в K8s). Для этой Lab мы временно терпим передачу через переменную окружения в SSH, но осознаем риск.

    4.  **Сделайте коммит и пуш:**
        ```bash
        git add Jenkinsfile # Добавьте Jenkinsfile
        # Добавьте скрипт wrap_and_run.sh, если он в репозитории
        # git add wrap_and_run.sh
        # Если вы изменили Ansible Playbook для копирования скрипта, добавьте его тоже
        # git add $HOME/ansible-config/...
        git commit -m "feat: Integrate Vault for secrets management"
        git push origin develop
        ```

    5.  **Запустите пайплайн и проверьте:**
        *   Дождитесь завершения пайплайна.
        *   Проверьте логи этапа "Deploy to Staging". Вы должны увидеть вывод скрипта `wrap_and_run.sh` о получении секретов.
        *   Подключитесь к Staging VM. Убедитесь, что приложение запущено. Проверьте его логи, оно должно успешно подключиться к БД.
        *   Попробуйте изменить пароль БД в Vault UI. Перезапустите приложение на Staging VM вручную (остановите процесс, запустите снова скриптом `wrap_and_run.sh`). Приложение должно использовать новый пароль.

**Результат Lab 7:** Ваше приложение теперь получает параметры подключения к БД и другие секреты из Vault через скрипт-обертку. Вы сделали важный шаг к безопасному управлению секретами в вашем конвейере доставки.

---

**Lab 8: Обновление Pipeline для Сборки и Деплоя Docker Образа**

*   **Цель:** Изменить CI/CD пайплайн, чтобы он собирал Docker образ после Maven сборки, публиковал его, а на этапе деплоя запускал контейнер из этого образа.
*   **Предварительные требования:**
    *   Настроенный Dockerfile (Lab 1).
    *   Настроенный Docker Registry (Nexus или другой, Lab 2).
    *   Работающий Jenkins Pipeline.
    *   Установленный Docker на Jenkins/Runner VM (для сборки) и Staging VM (для запуска).
    *   Настроенный Docker login на Jenkins/Runner VM (для пуша) и Staging VM (для пула, если реестр приватный и требует аутентификации).

*   **Шаги (для Jenkinsfile):**

    1.  **Обновите ваш Jenkinsfile:**

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any # Агент, где будет выполняться пайплайн. Убедитесь, что на нем установлен Docker.

            environment {
                // ... (Infra code dir, SSH user, Vault vars) ...

                // Переменные для Docker
                DOCKER_REGISTRY = 'ВАШ_IP_VM_NEXUS:8082' // Или registry.gitlab.com или Docker Hub
                DOCKER_IMAGE_NAME = "${DOCKER_REGISTRY}/my-java-app"
                // Используйте Jenkins Credentials для Docker Registry (ID: docker-registry-creds)
                // Если Registry приватный и требует логина для пуша/пула
                // DOCKER_REGISTRY_CREDENTIALS = credentials('docker-registry-creds')
            }

            stages {
                // ... (Clone Infra Code, Provision Infrastructure, Wait for SSH, Configure VM, Database Migrations этапы) ...

                // ИЗМЕНЕННЫЙ ЭТАП: Build and Publish Docker Image (вместо Publish Artifact)
                stage('Build & Publish Docker Image') {
                    steps {
                        echo 'Сборка Docker образа...'
                        script {
                            // Получить версию приложения (уже должно быть в env.APP_VERSION из этапа Get App Version)
                            def appVersion = env.APP_VERSION ?: 'latest' // Использовать latest, если версия не определена

                            // Собрать Docker образ
                            dir('my-app') { // Переходим в каталог Maven проекта, где лежит Dockerfile
                                sh "docker build -t ${DOCKER_IMAGE_NAME}:${appVersion} ."
                            }

                            echo 'Публикация Docker образа в Registry...'
                            // Логин в Docker Registry, если требуется
                            // withCredentials([usernamePassword(credentialsId: 'docker-registry-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            //     sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin ${DOCKER_REGISTRY}"
                            // }
                            // Публикация образа
                            sh "docker push ${DOCKER_IMAGE_NAME}:${appVersion}"
                            // Логаут (опционально)
                            // sh "docker logout ${DOCKER_REGISTRY}"
                        }
                        echo 'Сборка и публикация Docker образа завершены.'
                    }
                }

                // ... (Manual Approval stage - перед ним добавим Wait for Health Check) ...

                 // НОВЫЙ ЭТАП: Deploy Docker Container to Staging
                 stage('Deploy Docker Container to Staging') {
                    steps {
                        echo "Запуск деплоя Docker контейнера версии ${env.APP_VERSION} на Staging (${env.STAGING_SSH_HOST})..."
                        script {
                            // Получить версию приложения (уже должно быть в env.APP_VERSION)
                            def appVersion = env.APP_VERSION ?: 'latest'

                            // Команда для запуска контейнера на удаленной VM через SSH
                            def dockerRunCommand = """
                                # Опционально: Логин в Docker Registry на Staging VM, если он приватный
                                # echo ${env.DOCKER_PASS} | docker login -u ${env.DOCKER_USER} --password-stdin ${env.DOCKER_REGISTRY}

                                # Остановить и удалить старый контейнер (простая стратегия Recreate)
                                docker stop ${env.APP_NAME}-container 2>/dev/null || true # Остановить (игнорировать ошибку, если нет контейнера)
                                docker rm ${env.APP_NAME}-container 2>/dev/null || true # Удалить (игнорировать ошибку)

                                # Скачать новый образ (если еще нет)
                                docker pull ${DOCKER_IMAGE_NAME}:${appVersion}

                                # Запустить новый контейнер
                                # Проброс портов (-p)
                                # Передача переменных окружения (-e) - СЕКРЕТЫ ЛУЧШЕ ЧЕРЕЗ VAULT!
                                # Монтирование Volumes (-v) для персистентных данных
                                # Имя контейнера (--name)

                                docker run -d \\
                                  -p 8080:8080 \\
                                  --name ${env.APP_NAME}-container \\
                                  -e VAULT_ADDR="${env.VAULT_ADDR}" \\
                                  -e VAULT_TOKEN_VAR="${env.VAULT_TOKEN_VAR}" \\ # ОПАСНО! Используйте безопасный способ!
                                  -e SECRET_PATH="secret/myapp/config/staging" \\ # Передать путь к секретам
                                  -v myapp_logs_volume:/app/logs \\ # Монтирование Volume (нужно создать его на Staging VM)
                                  ${DOCKER_IMAGE_NAME}:${appVersion}

                                # Опционально: Логаут
                                # docker logout ${env.DOCKER_REGISTRY}
                                """

                            // Выполнить Docker команды на удаленной VM через SSH
                            // Передавать ENV VARs через SSH - ОПАСНО!
                            // Лучше: настроить SSH Agent Forwarding в Jenkins
                            // Или использовать секреты на удаленной стороне (Vault Agent, Kubernetes Secrets)
                            // Для Lab передаем через env vars в скрипте ssh (все еще небезопасно для секретов)
                            sh """
                              ssh ${env.STAGING_SSH_USER}@${env.STAGING_SSH_HOST} << 'EOF'
                                export APP_NAME="my-java-app" # Определить имя приложения
                                export DOCKER_REGISTRY="${env.DOCKER_REGISTRY}" # Передать registry url
                                export DOCKER_IMAGE_NAME="${env.DOCKER_IMAGE_NAME}" # Передать имя образа

                                export VAULT_ADDR="${env.VAULT_ADDR}" # Передать Vault адрес
                                export VAULT_TOKEN_VAR="${env.VAULT_TOKEN_VAR}" # Передать Vault токен (ОПАСНО!)
                                export SECRET_PATH="secret/myapp/config/staging" # Передать путь к секрету

                                # Создать Volume на Staging VM, если он еще не создан
                                docker volume create myapp_logs_volume >/dev/null 2>&1 || true

                                # Выполнить команду запуска контейнера (можно вызвать скрипт, который выполнит эти шаги)
                                # Или просто выполнить команды прямо здесь
                                docker stop \${APP_NAME}-container 2>/dev/null || true
                                docker rm \${APP_NAME}-container 2>/dev/null || true
                                docker pull \${DOCKER_IMAGE_NAME}:${appVersion}

                                # Запустить контейнер, передавая переменные окружения
                                docker run -d \\
                                  -p 8080:8080 \\
                                  --name \${APP_NAME}-container \\
                                  -e VAULT_ADDR="\${VAULT_ADDR}" \\
                                  -e VAULT_TOKEN_VAR="\${VAULT_TOKEN_VAR}" \\ # ОПАСНО!
                                  -e SECRET_PATH="\${SECRET_PATH}" \\
                                  -v myapp_logs_volume:/app/logs \\
                                  \${DOCKER_IMAGE_NAME}:${appVersion}

                                echo "Container started. Check docker logs \${APP_NAME}-container"
                              EOF
                            """
                        }
                        echo 'Деплой Docker контейнера завершен.'
                    }
                }

                // НОВЫЙ ЭТАП: Wait for Container Health / Health Check
                stage('Wait for Health Check') {
                    steps {
                        echo 'Ожидание запуска контейнера и проверки Health Check...'
                        // Здесь нужно добавить логику, которая ждет, пока контейнер станет здоровым
                        // (например, Docker Healthcheck) или пока ваше приложение не станет доступным по HTTP
                        // Пример простого Bash скрипта на агенте Jenkins, который проверяет HTTP endpoint:
                        script {
                            def stagingVmIp = env.STAGING_SSH_HOST // IP Staging VM
                            def appPort = 8080 # Порт приложения
                            def healthCheckUrl = "http://${stagingVmIp}:${appPort}/actuator/health" # Если приложение предоставляет health check endpoint (Spring Boot Actuator)

                            sh """
                              TIMEOUT=300 # 5 минут таймаут
                              INTERVAL=5 # Проверять каждые 5 секунд
                              ELAPSED=0
                              echo "Checking application health at ${healthCheckUrl}"
                              while ! curl -f ${healthCheckUrl} > /dev/null 2>&1; do
                                echo "Health check failed, waiting..."
                                sleep \$INTERVAL
                                ELAPSED=\$((ELAPSED + INTERVAL))
                                if [ \$ELAPSED -ge \$TIMEOUT ]; then
                                  echo "Timeout waiting for health check!"
                                  exit 1
                                fi
                              done
                              echo "Application is healthy."
                            """
                        }
                    }
                }
                 // Переместите этап Manual Approval после Health Check!
                 stage('Manual Approval for Production') {
                     steps {
                         echo 'Ожидание ручного подтверждения для деплоя на Production...'
                         script { input message: 'Проверили на Staging (в контейнере)? Готовы деплоить на Production?', ok: 'Да, деплоить' }
                         echo 'Подтверждение получено.'
                     }
                 }
            }
             post {
                 always { echo 'Пайплайн завершен.' }
                 success { echo 'Пайплайн выполнен успешно! 🎉' ; junit '**/target/surefire-reports/*.xml' }
                 failure { echo 'Пайплайн завершился с ошибкой! 💔' }
             }
        }
        ```
        *   **Важно по секретам:** Передача `VAULT_TOKEN_VAR` как переменной окружения в команде `docker run` через SSH - **ОПАСНО!** Токен будет виден в выводе `docker inspect` или `ps aux` внутри контейнера, если контейнер запущен не от root. Более безопасные методы:
            *   Использовать Docker Secrets (если используете Docker Swarm или Kubernetes).
            *   Использовать Vault Agent на Staging VM: Агент запускается рядом с приложением и может предоставлять секреты через файл или локальный API.
            *   Приложение само аутентифицируется в Vault (например, через AppRole или Instance Identity в облаке).
            *   **Для этой Lab, временно терпим передачу токена как ENV VAR для демонстрации, но осознаем риск.**

    2.  **Настройте Credential для Docker Registry в Jenkins:** Если ваш Registry приватный и требует аутентификации для пуша/пула. Тип: "Username with password". ID: `docker-registry-creds` (или другой). Username: Логин для Registry. Password: Пароль/токен.

    3.  **Обновите ваш скрипт-обертку `wrap_and_run.sh` (если используете):** Если приложение запускается *внутри* контейнера, и вы все еще используете скрипт-обертку для получения секретов, убедитесь, что этот скрипт скопирован в Docker образ и является частью `ENTRYPOINT` или `CMD`. Или перенесите логику получения секретов непосредственно в приложение, используя Vault Java Client. **Проще всего для Lab передать переменные окружения напрямую в `docker run`.**

    4.  **Сделайте коммит и пуш:**
        ```bash
        git add Jenkinsfile Dockerfile # Добавьте измененные файлы
        # Если изменили скрипт wrap_and_run.sh или Ansible playbooks, добавьте их
        git commit -m "ci: Build and deploy Docker container"
        git push origin develop
        ```

    5.  **Запустите пайплайн и проверьте:**
        *   Дождитесь завершения пайплайна.
        *   Просмотрите логи новых/измененных этапов.
        *   Подключитесь к Staging VM. Проверьте, что старый контейнер остановлен, новый запущен (`docker ps`), что логи пишутся в Volume (`docker logs my-app-container`), и что приложение доступно (`curl localhost:8080`).
        *   Проверьте, что приложение успешно подключилось к БД (логи приложения).

**Результат Lab 8:** Ваш Delivery Pipeline теперь полностью основан на Docker контейнерах. Он собирает образ, публикует его и деплоит, запуская контейнер на целевой VM. Приложение использует Vault для получения секретов (хотя метод передачи токена пока не идеален). Вы также добавили Health Check в пайплайн.

---

**Резюме Урока 7:**

Вы совершили переход к контейнеризации, упаковав ваше приложение в Docker образ и научившись управлять этим образом. Вы освоили Dockerfile, сборку и запуск контейнеров, а также работу с Container Registry. Вы познакомились с объектным хранилищем (MinIO) и, что крайне важно, внедрили HashiCorp Vault для безопасного управления секретами, решив проблему хранения учетных данных в явном виде. Ваш CI/CD пайплайн теперь полностью работает с Docker образами.

Контейнеризация и безопасное управление секретами - это маст-хэв для Middle+/Senior DevOps инженера. Теперь вы готовы к оркестрации контейнеров!

**Что дальше?**

В следующем уроке мы перейдем от запуска одного контейнера на VM к управлению несколькими контейнерами и микросервисами с помощью Docker Compose (пока локально) и познакомимся с концепциями балансировки нагрузки, которые будут критически важны для следующего шага - Kubernetes.

Готовьтесь к управлению группами контейнеров!