Отлично! Пришло время расширить наш конвейер от CI к CD. В этом уроке мы научимся управлять артефактами сборки и автоматизируем процесс их доставки на серверы.

---

**УРОК 4: Continuous Delivery и Continuous Deployment**

**Цель:** Понять разницу между Continuous Delivery и Continuous Deployment, научиться управлять артефактами сборки в централизованном хранилище и автоматизировать процесс развертывания приложения на тестовом окружении.

**Результат урока:** Ваш CI пайплайн превратится в базовый Continuous Delivery Pipeline. Он будет собирать и проверять код, публиковать готовый артефакт в Nexus и автоматически развертывать его на отдельной VM (тестовое окружение). Вы также настроите ручное подтверждение для перехода на следующий этап и реализуете простую возможность отката.

**Предварительные требования:**

*   Успешно завершен Урок 1, 2 и 3.
*   Настроенный CI пайплайн в Jenkins (с `Jenkinsfile`) или GitLab CI.
*   Рабочая VM (на которой уже установлен Jenkins/SonarQube или другая вспомогательная VM).
*   **Новая VM для тестового окружения (Staging VM).** Разверните еще одну чистую VM Linux (как в Уроке 3). Установите на ней JDK и настройте SSH доступ по ключу с вашей Jenkins/GitLab Runner VM (или с вашей рабочей станции, если Jenkins/Runner запускаются там). Убедитесь, что порты приложения открыты в файрволе Staging VM.

**Техническая подготовка (на Вашей VM или новой VM):**

1.  **Установите Docker Compose:** Если вы его еще не устанавливали для SonarQube (Lab 2 Урока 2), сделайте это (см. инструкции в Уроке 2). Это самый простой способ развернуть Nexus.

2.  **Убедитесь, что SSH доступ настроен:** На Jenkins/GitLab Runner VM (или там, где будет запускаться скрипт деплоя) должен быть установлен SSH клиент, и настроен беспарольный доступ по ключу к Staging VM. Вы можете использовать тот же ключ, что генерировали в Уроке 3 Lab 4, добавив публичный ключ на Staging VM в `~/.ssh/authorized_keys`.

3.  **Установите `curl` или `wget` на Jenkins/GitLab Runner VM:** Эти утилиты понадобятся для скачивания артефактов из Nexus. Скорее всего, они уже установлены.

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **Continuous Delivery (CD):** Практика, при которой каждое изменение кода, прошедшее CI (сборку, тестирование, анализ), **готово к релизу в продакшен в любой момент**. Процесс доставки до продакшена полностью автоматизирован, но *решение о релизе* принимает человек.
*   **Continuous Deployment (CD):** Практика, при которой каждое изменение кода, прошедшее все этапы CI и автоматизированного тестирования (включая тесты на стейджинге), **автоматически выпускается в продакшен без участия человека**. Это "золотой стандарт" DevOps, требующий высокой степени автоматизации и уверенности в тестах.
*   **Value Stream Management:** Управление потоком создания ценности. В контексте CD/CD это означает визуализацию и оптимизацию всех шагов от идеи/коммита до работающего ПО у пользователя. Цель - уменьшить время цикла (Lead Time) и повысить пропускную способность (Throughput).
*   **Delivery Pipeline:** Визуализация Value Stream в CI/CD системе. Последовательность этапов (stages), через которые проходит каждое изменение. Включает CI этапы (Build, Test, Analyze) и CD этапы (Deploy to Stage, Run Stage Tests, Deploy to Prod).
*   **Системы хранения артефактов (Artifact Repositories):** Централизованные хранилища для артефактов, созданных в процессе сборки (JAR, WAR, Docker образы, пакеты npm, deb, rpm и т.д.). Примеры: Nexus Repository Manager, JFrog Artifactory. Позволяют управлять версиями артефактов, проксировать внешние репозитории, обеспечивать надежность и безопасность.
*   **Twelve-Factor App:** Методология для создания приложений, которые:
    1.  Легко деплоятся на современные облачные платформы.
    2.  Масштабируются без значительных изменений кода.
    3.  Чисто разделяют код и конфигурацию.
    4.  Имеют сильные контракты с операционной системой.
    *Мы будем использовать ее принципы для нашего приложения и деплоя.*
*   **Стратегии развертывания:** Способы обновления работающего приложения с минимальным простоем и риском:
    *   **Recreate:** Остановка старой версии, развертывание новой. Простой в работе, но есть время простоя.
    *   **Rolling Update:** Постепенная замена экземпляров старой версии на экземпляры новой. Плавный переход, нет простоя, легко откатывать.
    *   **Blue/Green:** Развертывание новой версии ("Green") рядом со старой ("Blue"). Трафик переключается с Blue на Green сразу после проверки Green. Быстрый откат (просто переключить трафик обратно). Требует удвоенных ресурсов.
    *   **Canary:** Развертывание новой версии для небольшого процента пользователей. Если проблем нет, постепенно увеличивается процент нового трафика. Позволяет обнаружить проблемы на небольшой группе пользователей. Требует продвинутого управления трафиком.
    *   *В этом уроке мы начнем с простого Recreate/Rolling Update подхода на одной VM.*
*   **Откаты (Rollbacks):** Возможность быстро вернуться к предыдущей стабильной версии приложения в случае проблем с новым релизом. Критически важны для CD/CD.
*   **Бэкапирование:** Создание резервных копий данных и конфигураций. Хотя напрямую не является частью CI/CD пайплайна приложения, это фундаментальная практика для обеспечения надежности и восстановления после сбоев, особенно для баз данных.

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Установка и Настройка Nexus Repository Manager**

*   **Цель:** Развернуть централизованное хранилище артефактов.
*   **Шаги:**

    1.  **Выберите место для установки Nexus:** Рекомендуется отдельная VM или контейнер. Для практики можно использовать ту же VM, что и SonarQube, или новую.

    2.  **Создайте каталог для Nexus и файл `docker-compose.yml`:**
        ```bash
        mkdir $HOME/nexus
        cd $HOME/nexus
        nano docker-compose.yml
        ```
    3.  **Добавьте следующее содержимое в `docker-compose.yml`:** Nexus довольно ресурсоемкий, ему нужно хотя бы 2GB RAM (рекомендуется 4GB+) и быстрый диск. Убедитесь, что на VM достаточно ресурсов.

        ```yaml
        version: "3"

        services:
          nexus:
            image: sonatype/nexus3:latest # Используйте последнюю версию Nexus 3
            ports:
              - "8081:8081" # Порт веб-интерфейса и репозиториев Nexus
              # Если планируете хостить Docker образы, также нужны порты 8082 и 8083
              # - "8082:8082" # Для Docker (HTTP)
              # - "8083:8083" # Для Docker (HTTPS) - потребуется настройка SSL/TLS
            volumes:
              - nexus_data:/nexus-data # Данные Nexus (репозитории, конфигурация)
            environment:
              # Настройте потребление памяти Nexus под свои ресурсы
              - INSTALL4J_ADD_VMOPTIONS=-Xms1g -Xmx1g # Минимум 1GB, лучше 2GB+
            # Настройте перезапуск при ошибке, если нужно
            # restart: always

        volumes:
          nexus_data:
        ```
    4.  **Запустите Nexus в Docker:**
        ```bash
        docker-compose up -d
        ```
        Nexus может запускаться несколько минут при первом старте.

    5.  **Проверьте статус контейнера:**
        ```bash
        docker-compose ps
        ```
        Контейнер `nexus` должен быть в статусе `Up`.

    6.  **Выполните первичную настройку Nexus через веб-интерфейс:**
        *   Откройте браузер и перейдите по адресу вашей VM и порту 8081 (например, `http://ВАШ_IP_VM:8081`).
        *   Подождите, пока Nexus полностью загрузится (может занять 5-10 минут). Страница входа появится, когда он будет готов.
        *   Нажмите кнопку "Sign in" (человечек вверху справа).
        *   Стандартный логин: `admin`.
        *   Пароль находится в файле внутри контейнера. Выполните команду на VM, где запущен Nexus:
            ```bash
            docker exec -it nexus cat /nexus-data/admin.password
            ```
            Скопируйте этот временный пароль.
        *   Войдите с логином `admin` и временным паролем.
        *   Система попросит вас сменить пароль администратора. Сделайте это и запомните новый пароль.
        *   Система предложит настроить анонимный доступ. Для начала можно отключить его (Disable anonymous access).
        *   Вы попадете в интерфейс администратора Nexus.

    7.  **Ознакомьтесь с репозиториями по умолчанию:**
        *   В левом меню перейдите "Server administration and configuration" (шестеренка) -> "Repositories".
        *   Вы увидите список репозиториев:
            *   `maven-central`: Проксирует центральный репозиторий Maven.
            *   `maven-public`: Группа, объединяющая `maven-central`, `maven-releases`, `maven-snapshots`. Вы будете использовать URL этой группы для скачивания зависимостей и артефактов.
            *   `maven-releases`: Хостит ваши стабильные релизы.
            *   `maven-snapshots`: Хостит ваши SNAPSHOT (нестабильные, разрабатываемые) версии.
        *   *На будущее:* Можете посмотреть репозитории для Docker (`docker-hosted`, `docker-proxy`, `docker-all` - потребуется их включить и настроить порты в `docker-compose.yml`).

**Результат Lab 1:** Установленный и настроенный Nexus Repository Manager, готовый для хостинга ваших Maven артефактов.

---

**Lab 2: Публикация Артефактов в Nexus**

*   **Цель:** Настроить Maven проект и CI пайплайн для автоматической публикации собранного артефакта (JAR/WAR) в Nexus.
*   **Предварительные требования:**
    *   Работающий Nexus (Lab 1 этого урока).
    *   Рабочий Jenkins Pipeline (или GitLab CI Pipeline) из Урока 2.
    *   Локальный Maven проект (или в Git).
*   **Шаги:**

    1.  **Настройте Maven для работы с Nexus:** Maven должен знать, куда публиковать артефакты и откуда скачивать зависимости.
        *   Вам нужно отредактировать файл `settings.xml` Maven. Этот файл обычно находится в `~/.m2/settings.xml` для конкретного пользователя или в `$M2_HOME/conf/settings.xml` для всех пользователей системы. **Для Jenkins/GitLab Runner это должен быть `settings.xml`, доступный пользователю, от имени которого запускается сборка.** Чаще всего это `/var/lib/jenkins/.m2/settings.xml` для Jenkins.
        *   *Создайте или отредактируйте `/var/lib/jenkins/.m2/settings.xml` на вашей Jenkins VM:*
            ```bash
            sudo mkdir /var/lib/jenkins/.m2 # Если не существует
            sudo chown jenkins:jenkins /var/lib/jenkins/.m2 # Убедитесь, что у пользователя jenkins есть права
            sudo nano /var/lib/jenkins/.m2/settings.xml
            ```
        *   Добавьте (или отредактируйте) секции `<servers>` и `<mirrors>`:

            ```xml
            <settings xmlns="http://maven.apache.org/SETTINGS/1.1.0"
                      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                      xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd">

              <servers>
                <!-- Сервер для публикации SNAPSHOT версий -->
                <server>
                  <id>nexus-snapshots</id>
                  <!-- <username>admin</username> --> <!-- Логин для публикации -->
                  <!-- <password>your_nexus_admin_password</password> --> <!-- Пароль для публикации -->
                  <!-- Или используйте секцию <servers> в usersettings.xml с <server><id>...</id><username>...</username><password>...</password></server> -->
                  <!-- ИЛИ используйте Credentials ID из Jenkins - более безопасно -->
                </server>
                <!-- Сервер для публикации RELEASE версий -->
                <server>
                  <id>nexus-releases</id>
                  <!-- <username>admin</username> -->
                  <!-- <password>your_nexus_admin_password</password> -->
                  <!-- ИЛИ используйте Credentials ID из Jenkins -->
                </server>
              </servers>

              <mirrors>
                <!-- Использовать Nexus Public Group для всех запросов к центральным репозиториям -->
                <mirror>
                  <id>nexus</id>
                  <mirrorOf>*</mirrorOf> <!-- Зеркалировать все репозитории -->
                  <url>http://ВАШ_IP_VM_NEXUS:8081/repository/maven-public/</url>
                </mirror>
              </mirrors>

              <!-- Профили (опционально, но полезно) -->
              <!--
              <profiles>
                <profile>
                  <id>nexus</id>
                  <repositories>
                    <repository>
                      <id>central</id>
                      <url>http://central</url>
                      <releases><enabled>true</enabled></releases>
                      <snapshots><enabled>true</enabled></snapshots>
                    </repository>
                  </repositories>
                  <pluginRepositories>
                    <pluginRepository>
                      <id>central</id>
                      <url>http://central</url>
                      <releases><enabled>true</enabled></releases>
                      <snapshots><enabled>true</enabled></snapshots>
                    </pluginRepository>
                  </pluginRepositories>
                </profile>
              </profiles>

              <activeProfiles>
                <activeProfile>nexus</activeProfile>
              </activeProfiles>
              -->

            </settings>
            ```
        *   **Настройка учетных данных для публикации (важно для безопасности):** **Не храните пароль в `settings.xml`!**
            *   В Jenkins, перейдите "Manage Jenkins" -> "Manage Credentials".
            *   Выберите "Jenkins" (или домен) -> "Global credentials (unrestricted)".
            *   Нажмите "Add Credentials".
            *   Тип: "Username with password".
            *   Username: `admin` (или создайте специального пользователя в Nexus для деплоя).
            *   Password: Пароль пользователя Nexus.
            *   ID: **Важно!** ID должен совпадать с `id` серверов в вашем `settings.xml`, т.е., `nexus-snapshots` и `nexus-releases`. Создайте два таких Credential ID с одинаковыми логином/паролем Nexus.
            *   Description: Описание.
            *   Нажмите "Add".
            *   *Jenkins автоматически использует Credentials с совпадающим ID сервера из `settings.xml` при вызове Maven целей `deploy`/`install`, если они определены в `settings.xml` и настроены в Jenkins Credentials.*

    2.  **Настройте `pom.xml` для публикации в Nexus:**
        *   Откройте `my-app/pom.xml`.
        *   Добавьте секцию `<distributionManagement>` в корневой элемент `<project>`:

            ```xml
            <distributionManagement>
                <snapshotRepository>
                    <id>nexus-snapshots</id> <!-- ID должен совпадать с server ID в settings.xml -->
                    <url>http://ВАШ_IP_VM_NEXUS:8081/repository/maven-snapshots/</url>
                </snapshotRepository>
                <repository>
                    <id>nexus-releases</id> <!-- ID должен совпадать с server ID в settings.xml -->
                    <url>http://ВАШ_IP_VM_NEXUS:8081/repository/maven-releases/</url>
                </repository>
            </distributionManagement>
            ```
        *   Сохраните `pom.xml`.

    3.  **Обновите `Jenkinsfile` для публикации артефакта:**
        *   Откройте ваш `Jenkinsfile`.
        *   Добавьте новый этап *после* успешного прохождения Quality Gate:

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any

            stages {
                // ... (Checkout, Build, Test, SonarQube Analysis, Quality Gate Check этапы из Урока 2) ...

                // НОВЫЙ ЭТАП: Publish Artifact to Nexus
                stage('Publish Artifact') {
                    steps {
                        echo 'Публикация артефакта в Nexus...'
                        // Вызываем цель 'deploy' Maven плагина.
                        // Maven автоматически определит, куда публиковать (snapshots или releases)
                        // по суффиксу -SNAPSHOT в <version> в pom.xml.
                        // Maven использует настройки сервера из settings.xml с соответствующим ID.
                        sh 'mvn deploy -f my-app/pom.xml'
                        echo 'Публикация завершена.'
                    }
                }

                // Следующий этап будет Deploy to Staging (добавим позже)
            }

            post {
                always { echo 'Пайплайн завершен.' }
                success { echo 'Пайплайн выполнен успешно! 🎉' ; junit '**/target/surefire-reports/*.xml' }
                failure { echo 'Пайплайн завершился с ошибкой! 💔' }
                // Опционально: Clean up workspace после успешной публикации для экономии места
                // cleanupWs()
            }
        }
        ```
        *   Сохраните `Jenkinsfile`.

    4.  **Сделайте коммит и пуш:**
        ```bash
        git add my-app/pom.xml Jenkinsfile
        git commit -m "ci: Configure Maven deploy to Nexus and update pipeline"
        git push origin develop
        ```

    5.  **Проверьте выполнение пайплайна и артефакт в Nexus:**
        *   Дождитесь завершения пайплайна в Jenkins.
        *   Перейдите в веб-интерфейс Nexus.
        *   В левом меню перейдите "Browse" (иконка с папкой) -> "maven-snapshots" (если версия вашего проекта заканчивается на `-SNAPSHOT`).
        *   Найдите иерархию каталогов, соответствующую `<groupId>`, `<artifactId>` и `<version>` вашего проекта. Вы должны увидеть там ваш опубликованный JAR/WAR файл.

**Результат Lab 2:** Ваш CI пайплайн теперь успешно публикует собранный артефакт в Nexus Repository Manager после прохождения всех проверок качества и тестов.

---

**Lab 3: Автоматизация Развертывания на Staging Окружение**

*   **Цель:** Добавить в пайплайн этап автоматического деплоя последней успешной сборки на Staging VM.
*   **Предварительные требования:**
    *   Работающий пайплайн с публикацией в Nexus (Lab 2 этого урока).
    *   Подготовленная Staging VM с установленным JDK и настроенным SSH доступом по ключу с Jenkins VM (или Runner VM).
    *   Умение выполнять ручной деплой (Lab 5 Урока 3).
*   **Шаги:**

    1.  **Напишите скрипт деплоя:** Создайте простой Bash скрипт на вашей Jenkins VM (или Runner VM), который будет выполнять шаги ручного деплоя.
        ```bash
        # На вашей Jenkins VM (или Runner VM)
        mkdir $HOME/deployment_scripts
        nano $HOME/deployment_scripts/deploy_app_to_staging.sh
        ```
        ```bash
        #!/bin/bash

        # --- Параметры скрипта ---
        APP_NAME="my-java-app"
        ARTIFACT_NAME="my-app-1.0-SNAPSHOT.jar" # Имя вашего JAR/WAR файла
        NEXUS_URL="http://ВАШ_IP_VM_NEXUS:8081/repository/maven-snapshots/" # URL вашего репозитория в Nexus
        # URL для скачивания артефакта в Nexus. Может потребоваться токен, если Nexus приватный.
        # Формат URL: <Nexus_URL>/<groupId>/<artifactId>/<version>/<artifactId>-<version>.<extension>
        ARTIFACT_DOWNLOAD_URL="${NEXUS_URL}/com/mycompany/app/${ARTIFACT_NAME%.*}/${ARTIFACT_NAME}" # Пример URL

        STAGING_SSH_USER="ваше_имя_пользователя_на_staging_vm"
        STAGING_SSH_HOST="IP_АДРЕС_STAGING_VM"
        STAGING_APP_DIR="/opt/${APP_NAME}"

        # --- Начало скрипта ---
        echo "Starting deployment of ${APP_NAME} to Staging (${STAGING_SSH_HOST})..."

        # 1. Скачать последнюю версию артефакта из Nexus
        # Для SNAPSHOT версий может потребоваться более сложный URL или использование Maven Dependency Plugin
        # Простой вариант для SNAPSHOT: использовать Maven Dependency Plugin на Staging VM
        # или если имя файла SNAPSHOT всегда одно и то же, скачиваем напрямую.
        # Для Release версий имя файла стабильно.
        # Пока скачаем локально на Jenkins VM, потом передадим по SCP.
        echo "Downloading artifact from Nexus: ${ARTIFACT_DOWNLOAD_URL}"
        curl -o /tmp/${ARTIFACT_NAME} "${ARTIFACT_DOWNLOAD_URL}"
        if [ $? -ne 0 ]; then
          echo "Error: Failed to download artifact from Nexus."
          exit 1
        fi
        echo "Artifact downloaded to /tmp/${ARTIFACT_NAME}"


        # 2. Скопировать артефакт на Staging VM
        echo "Copying artifact to Staging VM..."
        scp /tmp/${ARTIFACT_NAME} ${STAGING_SSH_USER}@${STAGING_SSH_HOST}:${STAGING_APP_DIR}/
        if [ $? -ne 0 ]; then
          echo "Error: Failed to copy artifact to Staging VM."
          exit 1
        fi
        echo "Artifact copied to ${STAGING_APP_DIR}/ on Staging VM."

        # 3. Подключиться к Staging VM и выполнить шаги деплоя
        echo "Executing deployment steps on Staging VM..."
        ssh ${STAGING_SSH_USER}@${STAGING_SSH_HOST} << EOF
          # Перейти в каталог приложения
          cd ${STAGING_APP_DIR}

          # Найти и остановить текущий процесс приложения
          # Убедитесь, что ваше приложение имеет уникальный идентификатор в процессах (например, по имени JAR файла)
          CURRENT_PID=\$(pgrep -f "${ARTIFACT_NAME}") # Найти PID по имени файла

          if [ -n "\$CURRENT_PID" ]; then
            echo "Stopping current process (PID: \$CURRENT_PID)..."
            kill \$CURRENT_PID
            # Добавить ожидание, пока процесс действительно остановится
            # Пример: wait \$CURRENT_PID 2>/dev/null || echo "Process \$CURRENT_PID stopped."
            sleep 5 # Просто подождем 5 секунд
            # Проверить, остановился ли процесс
            if ps -p \$CURRENT_PID > /dev/null; then
               echo "Warning: Process \$CURRENT_PID did not stop gracefully, killing..."
               kill -9 \$CURRENT_PID
               sleep 2
            else
               echo "Process \$CURRENT_PID stopped successfully."
            fi
          else
            echo "No running process found for ${ARTIFACT_NAME}."
          fi

          # Запустить новую версию приложения в фоне
          echo "Starting new version of the application..."
          # Убедитесь, что JDK установлен на Staging VM!
          nohup java -jar ${ARTIFACT_NAME} > ${APP_NAME}.log 2>&1 &
          NEW_PID=\$!
          echo "New process started with PID: \$NEW_PID"

          # Простая проверка, что процесс запущен (можно добавить проверку порта)
          sleep 5 # Подождем немного, пока приложение стартует
          if ps -p \$NEW_PID > /dev/null; then
            echo "Application started successfully with PID \$NEW_PID."
            # Опционально: curl -I localhost:<PORT> для проверки доступности
          else
            echo "Error: Application failed to start!"
            exit 1 # Выйти с ошибкой из SSH сессии, что вызовет ошибку в Jenkins
          fi
EOF

        # Проверить статус выполнения SSH команды
        if [ $? -ne 0 ]; then
          echo "Error: Deployment script on Staging VM failed."
          exit 1
        fi

        echo "Deployment to Staging completed successfully."

        exit 0 # Успешное завершение скрипта
        ```
        *   Сделайте скрипт исполняемым:
            ```bash
            chmod +x $HOME/deployment_scripts/deploy_app_to_staging.sh
            ```
        *   **Важные замечания:**
            *   Скрипт очень простой. В реальных сценариях деплой сложнее (управление конфигурацией, миграции БД - Урок 6, health checks, управление версиями на сервере).
            *   Скачивание SNAPSHOT версии из Nexus по прямому URL может быть нестабильным, т.к. имя файла SNAPSHOT меняется при каждом перезапуске сборки (`...1.0-SNAPSHOT.jar` становится `...1.0-20231027.123456-789.jar`). В production вы будете деплоить **RELEASE** версии с фиксированным именем файла или использовать Maven Dependency Plugin для скачивания последней SNAPSHOT.
            *   Остановка и запуск приложения методом `kill` и `nohup` - это **самый простой (Recreate)** способ деплоя. Для Rolling Update или Blue/Green нужны более сложные механизмы (например, в K8s или с использованием Process Manager типа `systemd` или `supervisor`).
            *   Учетные данные SSH берутся из `~/.ssh/` пользователя, который запускает скрипт (Jenkins или GitLab Runner).

    2.  **Добавьте этап деплоя в `Jenkinsfile`:**
        *   Откройте ваш `Jenkinsfile`.
        *   Добавьте новый этап *после* этапа `Publish Artifact`:

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any

            stages {
                // ... (Checkout, Build, Test, SonarQube Analysis, Quality Gate Check, Publish Artifact этапы) ...

                // НОВЫЙ ЭТАП: Deploy to Staging
                stage('Deploy to Staging') {
                    steps {
                        echo 'Запуск деплоя на Staging...'
                        // Выполняем наш скрипт деплоя на агенте Jenkins
                        sh "$HOME/deployment_scripts/deploy_app_to_staging.sh" // Укажите полный путь к скрипту
                        echo 'Деплой на Staging завершен.'
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
        *   Сохраните `Jenkinsfile`.

    3.  **Сделайте коммит и пуш:**
        ```bash
        git add Jenkinsfile $HOME/deployment_scripts/deploy_app_to_staging.sh # Добавьте также скрипт в Git, если он не в хом-директории Jenkins
        git commit -m "cd: Add Deploy to Staging stage"
        git push origin develop
        ```
        *Примечание:* Хранение скриптов деплоя в домашней директории Jenkins не является хорошей практикой. Лучше хранить их в том же Git репозитории, где и код приложения, или в отдельном репозитории для деплоя/инфраструктуры. Если вы храните скрипт в репозитории, путь в `Jenkinsfile` будет относительно корня репозитория.

    4.  **Проверьте выполнение пайплайна и деплоя:**
        *   Дождитесь завершения пайплайна в Jenkins.
        *   Если все настроено правильно (SSH доступ, права на скрипт, JDK на Staging), пайплайн должен успешно завершиться.
        *   Подключитесь по SSH к Staging VM. Убедитесь, что приложение запущено (`ps aux | grep my-java-app`). Посмотрите логи (`cat /opt/myjavaapp/my-java-app.log`).

**Результат Lab 3:** Ваш пайплайн теперь автоматически деплоит собранный артефакт на тестовое окружение после успешной сборки и проверок.

---

**Lab 4: Настройка Ручного Подтверждения (Manual Promotion)**

*   **Цель:** Добавить шаг в пайплайн, требующий ручного подтверждения пользователя, прежде чем переходить к следующим этапам (например, деплою на Production).
*   **Шаги:**

    1.  **Обновите `Jenkinsfile` для добавления шага `input`:** Шаг `input` приостанавливает выполнение пайплайна до тех пор, пока пользователь не подтвердит его в Jenkins UI.
        *   Откройте ваш `Jenkinsfile`.
        *   Добавьте новый этап *после* `Deploy to Staging`. Назовите его `Promote to Production` или `Manual Approval`.
        *   Внутри этого этапа используйте шаг `input`:

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any

            stages {
                // ... (CI и Deploy to Staging этапы) ...

                // НОВЫЙ ЭТАП: Manual Approval for Production
                stage('Manual Approval for Production') {
                    steps {
                        echo 'Ожидание ручного подтверждения для деплоя на Production...'
                        script { // Шаг input должен быть внутри блока script в Declarative Pipeline
                            input message: 'Проверили на Staging? Готовы деплоить на Production?', ok: 'Да, деплоить на Production'
                        }
                        echo 'Подтверждение получено. Переход к Production.'
                    }
                }

                // Добавьте сюда следующий этап, который будет выполняться после подтверждения
                // stage('Deploy to Production') { ... }
            }

            post {
                always { echo 'Пайплайн завершен.' }
                success { echo 'Пайплайн выполнен успешно! 🎉' ; junit '**/target/surefire-reports/*.xml' }
                failure { echo 'Пайплайн завершился с ошибкой! 💔' }
            }
        }
        ```
        *   Сохраните `Jenkinsfile`.

    2.  **Сделайте коммит и пуш:**
        ```bash
        git add Jenkinsfile
        git commit -m "cd: Add manual approval stage before Production"
        git push origin develop
        ```

    3.  **Проверьте выполнение пайплайна:**
        *   Дождитесь, пока пайплайн выполнится до этапа `Manual Approval for Production`.
        *   Пайплайн приостановится на этом этапе. На странице выполнения пайплайна появится сообщение с кнопкой (например, "Да, деплоить на Production").
        *   Нажмите на кнопку, чтобы продолжить выполнение пайплайна.

**Результат Lab 4:** Ваш Delivery Pipeline теперь включает ручное подтверждение, имитируя процесс принятия решения о выпуске в продакшен. Это классический пример Continuous Delivery.

---

**Lab 5: Реализация Базового Отката (Basic Rollback)**

*   **Цель:** Добавить возможность откатить приложение на предыдущую версию с помощью скрипта.
*   **Предварительные требования:**
    *   Несколько успешных сборок/деплоев вашего приложения через Jenkins, чтобы в Nexus были разные версии артефактов (хотя бы две).
    *   Настроенный SSH доступ к Staging VM.
*   **Шаги:**

    1.  **Продумайте, как хранить информацию о развернутых версиях:** Чтобы откатить, нужно знать, какая версия была развернута *до* текущей. Простой способ для этой Lab - хранить информацию о последних деплоях на самой Staging VM в файле.

    2.  **Добавьте в скрипт деплоя логику сохранения версии:** Измените скрипт `deploy_app_to_staging.sh` на вашей Jenkins VM.

        ```bash
        #!/bin/bash

        # ... (Параметры скрипта) ...
        APP_VERSION="1.0-SNAPSHOT" # Здесь нужно как-то получить версию из pom.xml или из пайплайна!
        # В Jenkinsfile это можно передать как параметр сборки или прочитать из файла
        # Например, добавить параметр APP_VERSION в скрипт и передавать его из Jenkinsfile
        # Или прочитать из pom.xml на агенте Jenkins:
        # APP_VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout -f my-app/pom.xml)

        # Каталог для хранения информации о версиях на Staging VM
        VERSION_INFO_DIR="${STAGING_APP_DIR}/versions"
        CURRENT_VERSION_FILE="${VERSION_INFO_DIR}/current_version.txt"
        PREVIOUS_VERSION_FILE="${VERSION_INFO_DIR}/previous_version.txt"

        # --- Начало скрипта ---
        echo "Starting deployment of ${APP_NAME} version ${APP_VERSION} to Staging (${STAGING_SSH_HOST})..."

        # 1. Скачать последнюю версию артефакта из Nexus
        # ... (остается без изменений) ...
        # Для SNAPSHOT версий нужно доработать скачивание, чтобы получить конкретный файл.
        # Самый простой вариант: на Staging VM использовать Maven Dependency Plugin для резолвинга последней SNAPSHOT
        # или для этой Lab просто скопируем файл с жестко заданным именем SNAPSHOT.

        # 2. Скопировать артефакт на Staging VM
        echo "Copying artifact to Staging VM..."
        # Убедитесь, что целевой каталог существует на Staging VM
        ssh ${STAGING_SSH_USER}@${STAGING_SSH_HOST} "mkdir -p ${STAGING_APP_DIR}"
        scp /tmp/${ARTIFACT_NAME} ${STAGING_SSH_USER}@${STAGING_SSH_HOST}:${STAGING_APP_DIR}/
        if [ $? -ne 0 ]; then
          echo "Error: Failed to copy artifact to Staging VM."
          exit 1
        fi
        echo "Artifact copied to ${STAGING_APP_DIR}/ on Staging VM."

        # 3. Подключиться к Staging VM и выполнить шаги деплоя + сохранить информацию о версии
        echo "Executing deployment steps on Staging VM..."
        ssh ${STAGING_SSH_USER}@${STAGING_SSH_HOST} << EOF
          # Создать каталог для информации о версиях, если его нет
          mkdir -p ${VERSION_INFO_DIR}

          # Сохранить текущую версию как предыдущую
          if [ -f ${CURRENT_VERSION_FILE} ]; then
            CURRENT_DEPLOYED_VERSION=\$(cat ${CURRENT_VERSION_FILE})
            echo "Saving current version \${CURRENT_DEPLOYED_VERSION} as previous."
            echo "\${CURRENT_DEPLOYED_VERSION}" > ${PREVIOUS_VERSION_FILE}
          else
            echo "No current version file found. This is the first deployment."
          fi

          # Найти и остановить текущий процесс приложения
          # ... (логика остановки процесса из Lab 3) ...

          # Запустить новую версию приложения в фоне
          echo "Starting new version ${APP_VERSION}..."
          # Предполагаем, что артефакт всегда называется ARTIFACT_NAME в каталоге STAGING_APP_DIR
          # Если имя артефакта меняется (для SNAPSHOT), нужно скопировать его с правильным именем.
          nohup java -jar ${STAGING_APP_DIR}/${ARTIFACT_NAME} > ${STAGING_APP_DIR}/${APP_NAME}.log 2>&1 &
          NEW_PID=\$!
          echo "New process started with PID: \$NEW_PID"

          # Простая проверка, что процесс запущен
          sleep 5
          if ps -p \$NEW_PID > /dev/null; then
            echo "Application version ${APP_VERSION} started successfully with PID \$NEW_PID."
            # Сохранить новую версию как текущую
            echo "${APP_VERSION}" > ${CURRENT_VERSION_FILE}
          else
            echo "Error: Application failed to start!"
            exit 1
          fi
EOF

        # ... (проверка статуса SSH команды) ...

        echo "Deployment of version ${APP_VERSION} to Staging completed successfully."

        exit 0
        ```
        *   Сделайте скрипт исполняемым: `chmod +x $HOME/deployment_scripts/deploy_app_to_staging.sh`.
        *   **Как передать версию приложения?** В `Jenkinsfile` перед вызовом скрипта можно получить версию из `pom.xml` с помощью Maven Helper Plugin или просто считать файл `pom.xml` в Groovy, распарсить XML и извлечь версию. Например:

            ```groovy
            // В Jenkinsfile, перед вызовом скрипта деплоя
            // Шаг для получения версии Maven проекта
            stage('Get App Version') {
                steps {
                    script {
                        APP_VERSION = sh(returnStdout: true, script: 'cd my-app && mvn help:evaluate -Dexpression=project.version -q -DforceStdout').trim()
                        echo "Detected application version: ${APP_VERSION}"
                        // Сохранить версию как переменную окружения для последующих шагов
                        env.APP_VERSION = APP_VERSION
                    }
                }
            }

            // Измените этап 'Deploy to Staging' для передачи версии скрипту
            stage('Deploy to Staging') {
                steps {
                    echo "Запуск деплоя версии ${env.APP_VERSION} на Staging..."
                    // Передаем версию как аргумент скрипту
                    sh "$HOME/deployment_scripts/deploy_app_to_staging.sh ${env.APP_VERSION}"
                    echo 'Деплой на Staging завершен.'
                }
            }
            ```
            *Обновите скрипт `deploy_app_to_staging.sh` для приема версии как первого аргумента (`APP_VERSION="$1"`).*

    3.  **Напишите скрипт отката:** Создайте новый скрипт на вашей Jenkins VM.
        ```bash
        nano $HOME/deployment_scripts/rollback_app_on_staging.sh
        ```
        ```bash
        #!/bin/bash

        # --- Параметры скрипта ---
        APP_NAME="my-java-app"
        # Для отката нужна предыдущая версия. Скрипт деплоя должен ее сохранять.
        STAGING_SSH_USER="ваше_имя_пользователя_на_staging_vm"
        STAGING_SSH_HOST="IP_АДРЕС_STAGING_VM"
        STAGING_APP_DIR="/opt/${APP_NAME}"
        VERSION_INFO_DIR="${STAGING_APP_DIR}/versions"
        CURRENT_VERSION_FILE="${VERSION_INFO_DIR}/current_version.txt"
        PREVIOUS_VERSION_FILE="${VERSION_INFO_DIR}/previous_version.txt"

        # --- Начало скрипта ---
        echo "Starting rollback of ${APP_NAME} on Staging (${STAGING_SSH_HOST})..."

        # 1. Получить предыдущую версию с Staging VM
        PREVIOUS_VERSION=\$(ssh ${STAGING_SSH_USER}@${STAGING_SSH_HOST} "cat ${PREVIOUS_VERSION_FILE} 2>/dev/null")

        if [ -z "\$PREVIOUS_VERSION" ]; then
          echo "Error: No previous version information found on Staging VM."
          exit 1
        fi

        echo "Previous version detected: \$PREVIOUS_VERSION."

        # 2. Скачать артефакт предыдущей версии из Nexus
        # Это самый сложный шаг для SNAPSHOT версий. Для RELEASE версий URL будет стабильным.
        # Если вы публикуете только SNAPSHOT, возможно, вам придется хранить несколько последних
        # артефактов на самой Staging VM или в другом месте, чтобы не зависеть от
        # нестабильного URL SNAPSHOT в Nexus.
        # ДЛЯ ПРОСТОТЫ LAB 5, предположим, что у нас есть способ скачать артефакт предыдущей версии.
        # Это может быть:
        # а) Если деплоили RELEASE - URL стабилен.
        # б) Если деплоили SNAPSHOT - может потребоваться использовать Maven Dependency Plugin
        #    с версией <version> (например, 1.0-SNAPSHOT) и указанием <classifier> или <type>
        #    для скачивания конкретного файла из SNAPSHOT репозитория.
        # в) Более надежно: при каждом деплое копировать артефакт в каталог с именем версии
        #    на Staging VM (например, /opt/myjavaapp/versions/1.0-SNAPSHOT-BUILD123/)
        #    и просто переключать симлинк или запускать из нужного каталога при деплое/откате.
        #
        # Давайте для этой Lab *упростим* и предположим, что последняя рабочая версия
        # всегда доступна как ARTIFACT_NAME в каталоге STAGING_APP_DIR на Staging VM.
        # (Это не совсем корректно для полноценного отката на *любую* предыдущую, но иллюстрирует процесс).
        # Реальный откат требует либо стабильных артефактов в репозитории, либо хранения копий на сервере.
        #
        # Пропускаем шаг скачивания из Nexus для этой Lab, используем файл уже на Staging VM.

        # 3. Подключиться к Staging VM и выполнить шаги отката (остановить текущую, запустить предыдущую)
        echo "Executing rollback steps on Staging VM..."
        ssh ${STAGING_SSH_USER}@${STAGING_SSH_HOST} << EOF
          cd ${STAGING_APP_DIR}

          # Найти и остановить текущий процесс приложения (новой версии)
          CURRENT_PID=\$(pgrep -f "${ARTIFACT_NAME}") # Ищем по имени файла, оно может быть одинаковым
          # Более надежно: искать по PID, который был сохранен при последнем деплое
          # Или искать по имени, но убедиться, что запускаем именно предыдущую версию
          # Для этой простой Lab, просто остановим текущую и запустим файл с известным именем

          if [ -n "\$CURRENT_PID" ]; then
            echo "Stopping current process (PID: \$CURRENT_PID)..."
            kill \$CURRENT_PID
            sleep 5
            if ps -p \$CURRENT_PID > /dev/null; then
               echo "Warning: Process \$CURRENT_PID did not stop gracefully, killing..."
               kill -9 \$CURRENT_PID
               sleep 2
            else
               echo "Process \$CURRENT_PID stopped successfully."
            fi
          else
            echo "No running process found for ${ARTIFACT_NAME}."
          fi

          # Запустить предыдущую версию приложения в фоне
          # В этой упрощенной модели, предыдущая версия - это файл с тем же именем.
          # В реальной жизни, мы бы либо скачали файл предыдущей версии из Nexus
          # с конкретным версионированным именем (например, my-app-1.0.0.jar)
          # либо запустили бы из каталога предыдущей версии.
          echo "Starting previous version (\$PREVIOUS_VERSION)..."
          nohup java -jar ${ARTIFACT_NAME} > ${APP_NAME}.log 2>&1 &
          ROLLEDBACK_PID=\$!
          echo "Rolled back process started with PID: \$ROLLEDBACK_PID"

          # Обновить файл current_version.txt на предыдущую версию
          echo "\$PREVIOUS_VERSION" > ${CURRENT_VERSION_FILE}
          # Очистить файл previous_version.txt (или сохранить более старую версию)
          > ${PREVIOUS_VERSION_FILE} # Очищаем, чтобы предотвратить повторный откат на ту же версию

          # Простая проверка, что процесс запущен
          sleep 5
          if ps -p \$ROLLEDBACK_PID > /dev/null; then
            echo "Application version \$PREVIOUS_VERSION started successfully with PID \$ROLLEDBACK_PID (Rollback successful)."
          else
            echo "Error: Rollback failed! Application process did not start!"
            exit 1
          fi
EOF

        # ... (проверка статуса SSH команды) ...

        echo "Rollback to version \$PREVIOUS_VERSION completed successfully."

        exit 0
        ```
        *   Сделайте скрипт исполняемым: `chmod +x $HOME/deployment_scripts/rollback_app_on_staging.sh`.
        *   *Повторяем: этот скрипт отката **очень упрощен** и не отражает полную логику версионирования артефактов на сервере или надежного скачивания предыдущих SNAPSHOT версий из Nexus. Это только для иллюстрации принципа.*

    4.  **Добавьте Job для отката в Jenkins:** Откат обычно не является частью основного CI/CD пайплайна, он запускается вручную при необходимости.
        *   В Jenkins UI, "New Item".
        *   Имя: `my-java-app-rollback-staging`.
        *   Тип: "Freestyle project" (или "Pipeline", если хотите определить его в Jenkinsfile отдельно). Выберем Freestyle для простоты запуска "по требованию".
        *   **General:** Описание "Job для отката приложения на Staging".
        *   **Build:**
            *   "Add build step".
            *   "Execute shell".
            *   Command: `$HOME/deployment_scripts/rollback_app_on_staging.sh`.
        *   Нажмите "Save".

    5.  **Проверьте функцию отката:**
        *   Запустите ваш основной CI/CD пайплайн (Lab 3) несколько раз, чтобы убедиться, что он успешно деплоит последнюю версию и обновляет файл `current_version.txt` на Staging VM, а также сохраняет предыдущую версию в `previous_version.txt`.
        *   Сделайте коммит с ошибкой в приложении или его запуске, который приведет к проблемам на Staging. Запушьте, дождитесь деплоя. Убедитесь, что приложение на Staging не работает или работает неправильно.
        *   Перейдите в Jenkins, откройте Job `my-java-app-rollback-staging`. Нажмите "Build Now".
        *   Проверьте логи выполнения Job'а отката. Он должен определить предыдущую версию, остановить текущее приложение и запустить предыдущую.
        *   Проверьте на Staging VM, что теперь запущена предыдущая версия приложения и файл `current_version.txt` содержит номер предыдущей версии.

**Результат Lab 5:** Вы реализовали простую механику сохранения информации о версиях на сервере и создали скрипт отката, запускаемый вручную. Это показывает базовый подход к обеспечению возможности быстрого восстановления после неудачного деплоя.

---

**Lab 6: Анализ Приложения по Принципам Twelve-Factor App**

*   **Цель:** Оценить, насколько ваше тестовое приложение соответствует принципам Twelve-Factor App, и определить области для улучшения.
*   **Ключевые Принципы Twelve-Factor App (кратко):**
    1.  **Codebase:** Один codebase, отслеживаемый в VCS, множество деплоев.
    2.  **Dependencies:** Явно объявлять и изолировать зависимости (например, через `pom.xml`, `requirements.txt`).
    3.  **Config:** Хранить конфигурацию в среде (переменные окружения), а не в коде.
    4.  **Backing Services:** Рассматривать вспомогательные сервисы (БД, брокеры сообщений) как подключаемые ресурсы.
    5.  **Build, release, run:** Строго разделять стадии сборки, релиза и выполнения.
    6.  **Processes:** Выполнять приложение как один или несколько stateless процессов.
    7.  **Port binding:** Экспортировать сервисы через привязку портов.
    8.  **Concurrency:** Масштабировать через модель процессов.
    9.  **Disposability:** Максимизировать надежность за счет быстрого запуска/грамотного завершения процессов.
    10. **Dev/prod parity:** Минимизировать расхождения между окружениями разработки, стейджинга и продакшена.
    11. **Logs:** Рассматривать логи как потоки событий (Event Streams).
    12. **Admin processes:** Выполнять административные/миграционные задачи как разовые процессы.
*   **Шаги:**

    1.  **Просмотрите код и структуру вашего тестового приложения.**
    2.  **Пройдитесь по каждому из 12 принципов и задайте себе вопросы:**
        *   *Codebase:* Весь код в одном репозитории? Деплои (Staging VM) создаются из этого codebase? (Вероятно, да).
        *   *Dependencies:* Зависимости Maven объявлены в `pom.xml`? (Вероятно, да).
        *   *Config:* Где хранятся настройки подключения к БД, порты, другие параметры? В коде? В файле свойств (`application.properties`)? Передаются через аргументы командной строки или переменные окружения? **Если в коде или файле свойств, это нарушение принципа 3.** Лучший способ - использовать переменные окружения.
        *   *Backing Services:* Подключается ли приложение к БД? Как оно находит ее? (Вероятно, через URL в конфиге). Можно ли легко поменять подключение к БД, не меняя код? (Если конфиг вынесен, то да).
        *   *Build, release, run:* CI пайплайн разделяет сборку (`mvn package`) и деплой (скрипт)? Да. Но артефакт содержит всю информацию о сборке? (Да, JAR). Процесс релиза (сборка + конфигурация для окружения) четко определен? (Пока только сборка и простой деплой).
        *   *Processes:* Приложение stateless? (Для простого REST API, вероятно, да). Хранит ли оно данные в своей файловой системе между запросами? (Не должно).
        *   *Port binding:* Приложение слушает порт? Можно ли указать порт через конфиг/переменную окружения? (Вероятно, да).
        *   *Concurrency:* Как бы вы масштабировали приложение? Запуском нескольких копий? (Если оно stateless, то да).
        *   *Disposability:* Приложение быстро стартует? Корректно завершается при получении сигналов (например, SIGTERM при `kill`)? (Стандартные фреймворки типа Spring Boot обычно корректно обрабатывают сигналы).
        *   *Dev/prod parity:* Насколько сильно отличаются ваше локальное окружение разработки и Staging VM? Используете ли вы одни и те же версии JDK, одни и те же сервисы (БД)? (Вероятно, есть отличия). Использование Docker (будет в Уроке 7) помогает сильно сократить этот разрыв.
        *   *Logs:* Приложение пишет логи в файл? Или в стандартные потоки (stdout/stderr)? **Принцип 11 рекомендует писать в stdout/stderr**, а сбор логов должен заниматься окружение (docker/kubernetes/системы логирования).
        *   *Admin processes:* Как вы выполняете миграции БД или другие разовые задачи? Вручную? (Вероятно, да. DBOps в Уроке 6 автоматизирует миграции).

    3.  **Определите 2-3 ключевых принципа, которые ваше приложение нарушает сильнее всего, и подумайте, как их можно было бы улучшить.** Скорее всего, это будет Principle 3 (Config) и Principle 11 (Logs). Возможно, Principle 2 (Dependencies) если есть неуправляемые зависимости.

    4.  **Обсудите с коллегами (если возможно) или запишите ваши выводы.**

**Результат Lab 6:** Вы провели анализ вашего приложения с точки зрения Twelve-Factor App, определили его сильные и слабые стороны с точки зрения готовности к облачному деплою и масштабированию, и поняли, какие изменения в архитектуре или настройках были бы полезны.

---

**Резюме Урока 4:**

Вы перешли от Continuous Integration к Continuous Delivery. Вы научились использовать систему управления артефактами (Nexus) как центральное хранилище для ваших собранных бинарников. Вы автоматизировали процесс деплоя вашего приложения на тестовое окружение с помощью скрипта, запускаемого из Jenkins пайплайна. Добавив шаг ручного подтверждения, вы реализовали классическую модель CD. Вы также рассмотрели важность откатов и проанализировали ваше приложение с точки зрения Twelve-Factor App.

Теперь ваш конвейер может не только собирать и проверять код, но и автоматически доставлять его на первое окружение.

**Что дальше?**

В следующем уроке мы сфокусируемся на автоматизации развертывания и настройки *инфраструктуры* с помощью Infrastructure as Code (Terraform) и Configuration Management (Ansible). Это позволит нам автоматически создавать и настраивать такие VM, как Staging VM, вместо того, чтобы делать это вручную.

Продолжайте практиковаться и готовиться к новому блоку автоматизации!