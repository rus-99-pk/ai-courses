Отлично! Продолжаем детализировать курс. Вот подробная методичка для Урока 2.

---

**УРОК 2: Гибкие Методологии и Continuous Integration (Продвинутый CI)**

**Цель:** Углубить понимание принципов DevOps и CI, перейти от простых Jenkins задач к определению конвейера как кода, интегрировать инструменты для анализа качества и безопасности кода, а также рассмотреть альтернативные CI системы.

**Результат урока:** У вас будет CI пайплайн, определенный в файле (`Jenkinsfile`), который хранится в Git. Этот пайплайн будет автоматически собирать проект, запускать тесты и анализировать код на качество и базовую безопасность с помощью SonarQube. Вы также получите опыт работы с GitLab CI.

**Предварительные требования:**

*   Успешно завершен Урок 1 (настроен Git, Gitea, Jenkins, Maven проект).
*   Доступ к VM с установленными сервисами.
*   Понимание основ работы с файлами в Linux (редактирование, создание).

**Техническая подготовка (на Вашей VM, если еще не установлено/настроено):**

1.  **Docker Compose:** Если вы не устанавливали Docker Compose в Уроке 1 для Gitea, установите его. Это удобно для развертывания стека SonarQube.
    ```bash
    # Для Ubuntu (замените версию на последнюю):
    sudo curl -L "https://github.com/docker/compose/releases/download/v2.18.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    docker-compose --version # Проверьте установку
    ```
    (Для других ОС см. официальную документацию Docker Compose).

2.  **Установите SonarQube Scanner для Jenkins:** В Jenkins UI перейдите "Manage Jenkins" -> "Manage Plugins" -> вкладка "Available plugins". Найдите "SonarQube Scanner for Jenkins" и установите его.

3.  **Настройте SonarQube Scanner в Jenkins:** Перейдите "Manage Jenkins" -> "Global Tool Configuration". Найдите секцию "SonarQube Scanner". Нажмите "Add SonarQube Scanner". Назовите его (например, `sonar-scanner`). Выберите "Install automatically" и предпочтительный способ установки (рекомендуется "Install from maven.org"). Нажмите "Save".

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **DevOps-культура: 3 пути:**
    1.  **Первый путь (Flow):** Ускорение потока ценности от идеи до пользователя. Включает CI, CD, автоматизацию, уменьшение размера изменений, быстрое обнаружение проблем.
    2.  **Второй путь (Feedback):** Создание коротких и быстрых петель обратной связи. Раннее обнаружение проблем (тесты, мониторинг, алерты), возможность быстро учиться на ошибках, телеметрия, общение между командами. SonarQube - яркий пример Feedback Loop для разработчиков.
    3.  **Третий путь (Experimentation & Learning):** Культура непрерывного экспериментирования, готовность рисковать (контролируемо), учиться на ошибках и делиться знаниями. Позволяет быстро адаптироваться к изменениям.

*   **Гибкие методологии (Agile):** Kanban и Scrum - популярные фреймворки для управления разработкой. Agile подчеркивает итеративную разработку, сотрудничество с заказчиком, готовность к изменениям. CI/CD критически важны для Agile, так как позволяют быстро доставлять инкременты продукта.

*   **Continuous Integration (CI):** Практика, при которой разработчики часто (несколько раз в день) коммитят изменения в общую ветку репозитория. Каждое изменение автоматически проверяется (собирается, тестируется). Цель - раннее обнаружение и устранение конфликтов интеграции и ошибок.

*   **CI/CD Pipeline:** Автоматизированный конвейер, который проходит код после коммита:
    *   `Source`: Получение исходного кода (например, из Git).
    *   `Build`: Компиляция кода, сборка артефактов (JAR, WAR, Docker Image).
    *   `Test`: Запуск автоматических тестов (Unit, Integration).
    *   `Analyze`: Статический анализ кода (качество, безопасность).
    *   `Package`: Упаковка для деплоя (если не сделано на этапе Build).
    *   ... и далее идут этапы Continuous Delivery/Deployment (которые будут в следующих уроках).

*   **Pipeline as Code (Jenkinsfile):** Определение конвейера сборки/тестирования/деплоя в текстовом файле (`Jenkinsfile`), который хранится вместе с исходным кодом в системе контроля версий. Преимущества: версионирование пайплайна, ревью изменений пайплайна, единый источник истины.
    *   **Jenkins Declarative Pipeline:** Более простой и структурированный синтаксис Jenkinsfile по сравнению со Scripted Pipeline. Использует блоки (`pipeline`, `agent`, `stages`, `stage`, `steps`, `post`, `environment`, `input` и др.).

*   **Альтернативные CI системы:** GitLab CI, GitHub Actions, CircleCI, Travis CI, Bamboo. У всех схожая концепция: определение пайплайна в YAML-файле в корне репозитория. Различаются синтаксисом, возможностями интеграции, моделью выполнения (агенты/раннеры).

*   **SonarQube:** Платформа для статического анализа кода. Проверяет на:
    *   Баги (Bugs): Очевидные ошибки в логике.
    *   Уязвимости (Vulnerabilities): Проблемы безопасности (например, SQL Injection, XSS).
    *   Code Smells: Проблемы с поддерживаемостью, читаемостью, сложностью кода.
    *   **Quality Gate:** Набор пороговых значений метрик (например, 0 новых багов, покрытие кода тестами > 80% на новых строках), который определяет, может ли новая версия кода "пройти" и быть допущена к дальнейшим этапам конвейера. Если Quality Gate не пройден, сборка в CI должна падать.
    *   **SAST (Static Application Security Testing):** Анализ исходного кода или бинарных файлов на наличие уязвимостей без фактического выполнения кода. SonarQube включает SAST в свои возможности анализа.

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Перевод Jenkins Job в Declarative Pipeline**

*   **Цель:** Преобразовать конфигурацию сборки из веб-интерфейса Jenkins в файл `Jenkinsfile`, хранящийся в Git.
*   **Шаги:**

    1.  **Откройте ваш Maven проект локально:** Перейдите в каталог `my-devops-app/my-app` (или где у вас находится `pom.xml`).

    2.  **Создайте файл `Jenkinsfile` в корне *Git репозитория*:** Не в каталоге Maven проекта, а там, где `.git` и каталог `my-app`.
        ```bash
        cd ../.. # Перейдите в корень my-devops-app, если вы в my-app
        nano Jenkinsfile # Или используйте ваш любимый редактор
        ```

    3.  **Напишите базовый Declarative Pipeline:**

        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            // Агент (agent) определяет, где будет выполняться весь пайплайн или отдельные этапы.
            // 'any' означает, что Jenkins выберет любой доступный агент (включая мастер, если разрешено).
            agent any

            // Секция stages определяет последовательность этапов выполнения.
            stages {
                // Этап 'Checkout' - получение исходного кода.
                // В Declarative Pipeline, если agent определен на уровне pipeline или stage,
                // получение кода из SCM (Git) выполняется автоматически по умолчанию
                // на агенте, связанном с этим stage/pipeline.
                stage('Checkout') {
                    steps {
                        // Можно явно указать checkout, но обычно не нужно, если настроено в SCM
                        // checkout scm
                        echo 'Код получен (автоматически).'
                    }
                }

                // Этап 'Build' - сборка проекта с помощью Maven.
                stage('Build') {
                    steps {
                        echo 'Запуск сборки Maven...'
                        // Выполняем команду Maven. Убедитесь, что Maven и JDK доступны на агенте.
                        // Путь до pom.xml может потребоваться, если он не в корне репозитория.
                        // tool 'Maven_Local' - если вы настроили Maven в Global Tool Config и хотите использовать его.
                        // В простых случаях, если Maven в PATH агента, можно просто sh 'mvn...'
                        sh 'mvn clean package -f my-app/pom.xml' // -f указывает путь к pom.xml
                        echo 'Сборка завершена.'
                    }
                }

                // Этап 'Test' - запуск тестов. Обычно mvn package включает тесты,
                // но можно выделить в отдельный этап или явно запустить mvn test.
                stage('Test') {
                     steps {
                         echo 'Запуск тестов...'
                         // Если тесты уже запускались в 'package', этот этап может быть опциональным
                         // или использоваться для запуска других видов тестов (например, интеграционных).
                         //sh 'mvn test -f my-app/pom.xml'
                         echo 'Тесты выполнены (как часть package или опционально).'
                     }
                }
            }

            // Секция post определяет действия, выполняемые после завершения пайплайна
            // (независимо от его статуса или только при успехе/неудаче).
            post {
                // always: выполняется всегда
                // success: выполняется только при успешном завершении пайплайна
                // failure: выполняется только при завершении пайплайна с ошибкой
                // changed: выполняется, если статус пайплайна изменился по сравнению с предыдущим
                // aborted: выполняется, если пайплайн был прерван вручную
                always {
                    echo 'Пайплайн завершен.'
                }
                success {
                    echo 'Пайплайн выполнен успешно! 🎉'
                    // Пример публикации результатов тестов JUnit
                    // Если тесты Maven генерируют Surefire/Failsafe отчеты в target/surefire-reports/*.xml
                    junit '**/target/surefire-reports/*.xml'
                }
                failure {
                    echo 'Пайплайн завершился с ошибкой! 💔'
                }
            }
        }
        ```

    4.  **Добавьте `Jenkinsfile` в Git, сделайте коммит и запушьте:**
        ```bash
        git add Jenkinsfile
        git commit -m "ci: Add initial Jenkinsfile (Declarative Pipeline)"
        git push origin develop
        ```

    5.  **Настройте Jenkins Job для использования `Jenkinsfile`:**
        *   Перейдите в веб-интерфейс Jenkins.
        *   Откройте ваш Job (`my-java-app-build-freestyle`).
        *   Нажмите "Configure".
        *   В секции "General", измените тип проекта на "Pipeline". (В некоторых версиях Jenkins, возможно, придется создать новый Job типа "Pipeline" и скопировать настройки SCM).
        *   Удалите секцию "Build" с вызовом Maven (или убедитесь, что ее нет, если создали новый Pipeline Job).
        *   Прокрутите вниз до секции "Pipeline".
        *   Definition: Выберите "Pipeline script from SCM".
        *   SCM: Выберите "Git".
        *   Repository URL: Укажите URL вашего репозитория Gitea (тот же, что и был).
        *   Credentials: Выберите ваши учетные данные для Gitea.
        *   Branches to build: Укажите `develop`.
        *   Script Path: Укажите имя файла пайплайна относительно корня репозитория: `Jenkinsfile`.
        *   Нажмите "Save".

    6.  **Запустите пайплайн и проверьте результаты:**
        *   Запустите сборку вручную ("Build Now").
        *   Или сделайте еще один тестовый коммит и пуш в ветку `develop`.
        *   На странице Job'а вы увидите "Pipeline Steps". Нажмите на номер сборки -> "Console Output".
        *   Проверьте, что пайплайн прошел по этапам (Checkout, Build, Test) и успешно завершился.
        *   Если в проекте были JUnit тесты, вы должны увидеть график "Latest Test Results" на странице Job'а.

**Результат Lab 1:** Jenkins Job теперь запускает пайплайн, определенный в `Jenkinsfile` из вашего Git репозитория. Пайплайн выполняет шаги получения кода, сборки и (опционально) запуска тестов, с публикацией отчетов JUnit.

---

**Lab 2: Установка и Настройка SonarQube**

*   **Цель:** Развернуть SonarQube сервер для анализа кода.
*   **Шаги:**

    1.  **Выберите место для установки SonarQube:** Рекомендуется отдельная VM или контейнер, отличный от Jenkins/Gitea, если возможно. Для практики можно использовать ту же VM, если ресурсов достаточно.
    2.  **Подготовьте среду для SonarQube:** SonarQube требует базу данных (PostgreSQL, MySQL, MS SQL). SQLite3 не поддерживается для production. Для простоты, используем Docker Compose с PostgreSQL.
    3.  **Создайте каталог для SonarQube и файл `docker-compose.yml`:**
        ```bash
        mkdir $HOME/sonarqube
        cd $HOME/sonarqube
        nano docker-compose.yml
        ```
    4.  **Добавьте следующее содержимое в `docker-compose.yml`:**
        ```yaml
        version: "2"

        services:
          sonarqube:
            image: sonarqube:latest # Используйте последнюю версию SonarQube
            ports:
              - "9000:9000" # Веб-интерфейс SonarQube
              - "9092:9092" # Порт для SonarQube Scanner (не всегда нужен напрямую)
            networks:
              - sonarnet
            environment:
              - sonar.jdbc.url=jdbc:postgresql://db:5432/sonar
              - sonar.jdbc.username=sonar
              - sonar.jdbc.password=sonar # ИЗМЕНИТЕ ЭТОТ ПАРОЛЬ В PRODUCTION!
              - sonar.search.javaOpts=-Xmx512m -Xms512m # Настройте под свои ресурсы
            volumes:
              - sonarqube_data:/opt/sonarqube/data # Данные SonarQube
              - sonarqube_extensions:/opt/sonarqube/extensions # Плагины
              - sonarqube_logs:/opt/sonarqube/logs # Логи
          db:
            image: postgres:13-alpine # Используйте LTS версию PostgreSQL
            networks:
              - sonarnet
            environment:
              - POSTGRES_USER=sonar
              - POSTGRES_PASSWORD=sonar # ИЗМЕНИТЕ ЭТОТ ПАРОЛЬ В PRODUCTION!
              - POSTGRES_DB=sonar
            volumes:
              - postgresql_data:/var/lib/postgresql/data # Данные базы данных
            # ports: # Не нужно пробрасывать порт БД наружу, SonarQube обращается к ней по имени сервиса 'db' внутри сети Docker Compose
            #  - "5432:5432"

        networks:
          sonarnet:
            driver: bridge

        volumes:
          sonarqube_data:
          sonarqube_extensions:
          sonarqube_logs:
          postgresql_data:
        ```
    5.  **Запустите SonarQube и базу данных:**
        ```bash
        docker-compose up -d
        ```
        Подождите несколько минут, пока контейнеры скачаются и запустятся.
    6.  **Проверьте статус контейнеров:**
        ```bash
        docker-compose ps
        ```
        Оба контейнера (`sonarqube` и `db`) должны быть в статусе `Up`.
    7.  **Выполните первичную настройку SonarQube через веб-интерфейс:**
        *   Откройте браузер и перейдите по адресу вашей VM и порту 9000 (например, `http://ВАШ_IP_VM:9000`).
        *   Войдите с логином `admin` и паролем `admin`.
        *   Система попросит вас сменить пароль администратора. Сделайте это и запомните новый пароль.
        *   Нажмите на иконку "+" в правом верхнем углу -> "Create Project".
        *   Выберите "Manually".
        *   **Project Key:** Укажите уникальный ключ для вашего проекта (например, `my-java-app`).
        *   **Display Name:** Укажите отображаемое имя (например, `My Java Application`).
        *   Нажмите "Create".
        *   Выберите способ анализа "With Jenkins".
        *   **Provide a token:** Выберите "Generate a token". Введите имя токена (например, `jenkins-token`). Нажмите "Generate".
        *   **Скопируйте сгенерированный токен!** Он будет показан только один раз. Нажмите "Continue".
        *   На следующем шаге SonarQube покажет инструкции по настройке Jenkins. Не закрывайте эту страницу, она пригодится в следующей Lab. Выберите тип проекта "Maven".

**Результат Lab 2:** Установленный и работающий SonarQube сервер с базой данных. Создан проект в SonarQube, и вы получили токен для подключения Jenkins.

---

**Lab 3: Интеграция SonarQube в Jenkins Pipeline**

*   **Цель:** Добавить этап статического анализа кода с помощью SonarQube в ваш Jenkins пайплайн.
*   **Шаги:**

    1.  **Настройте подключение к SonarQube в Jenkins:**
        *   В Jenkins UI перейдите "Manage Jenkins" -> "Configure System".
        *   Прокрутите до секции "SonarQube servers".
        *   Поставьте галочку "Enable injection of SonarQube server configurations as environment variables".
        *   Нажмите "Add new SonarQube".
            *   **Name:** Укажите имя сервера (например, `MySonarQube`).
            *   **Server URL:** Укажите URL вашего SonarQube (например, `http://ВАШ_IP_VM:9000`).
            *   **Server authentication token:** Нажмите "Add" -> "Jenkins".
                *   Тип учетных данных: "Secret text".
                *   Secret: Вставьте токен, который вы сгенерировали в SonarQube (Lab 2, шаг 7).
                *   ID: Придумайте ID (например, `sonarqube-jenkins-token`).
                *   Description: Краткое описание.
                *   Нажмите "Add".
                *   Выберите созданный токен из выпадающего списка.
        *   Нажмите "Save" внизу страницы.

    2.  **Добавьте Maven SonarQube Plugin в `pom.xml`:**
        *   Откройте файл `my-app/pom.xml` в вашем проекте.
        *   Добавьте секцию `<plugin>` для `sonar-maven-plugin` внутри секции `<build><plugins>`. Если секций `<build>` или `<plugins>` нет, создайте их.
        ```xml
        <build>
            <plugins>
                <plugin>
                    <groupId>org.sonarsource.scanner</groupId>
                    <artifactId>sonar-maven-plugin</artifactId>
                    <version>3.9.1.2184</version> <!-- Используйте актуальную версию -->
                </plugin>
            </plugins>
        </build>
        ```
        *   *Опционально, но рекомендуется:* Добавьте свойства SonarQube в секцию `<properties>` для более чистого вызова:
        ```xml
        <properties>
            <sonar.projectKey>my-java-app</sonar.projectKey> <!-- Ключ проекта из SonarQube -->
            <sonar.organization>default-organization</sonar.organization> <!-- Если используете SonarCloud или Organization в SonarQube -->
            <!-- sonar.host.url и sonar.login не указываем здесь, их предоставит Jenkins -->
        </properties>
        ```
        *   Сохраните `pom.xml`.

    3.  **Обновите `Jenkinsfile` для запуска анализа SonarQube:**
        *   Откройте ваш `Jenkinsfile`.
        *   Добавьте новый этап *после* этапа `Build` (и `Test`, если он у вас был):
        ```groovy
        // Jenkinsfile (Declarative Pipeline)

        pipeline {
            agent any

            stages {
                stage('Checkout') {
                   steps { echo 'Код получен (автоматически).' }
                }

                stage('Build') {
                    steps {
                        echo 'Запуск сборки Maven...'
                        sh 'mvn clean package -f my-app/pom.xml'
                        echo 'Сборка завершена.'
                    }
                }

                // Опциональный этап тестов, если они не включены в package
                stage('Test') {
                     steps { echo 'Тесты выполнены.' }
                }

                // НОВЫЙ ЭТАП: Анализ SonarQube
                stage('SonarQube Analysis') {
                    steps {
                        echo 'Запуск анализа SonarQube...'
                        // withSonarQubeEnv инжектирует переменные окружения с настройками SonarQube
                        // 'MySonarQube' - это имя сервера, которое вы указали в Global Configuration
                        withSonarQubeEnv('MySonarQube') {
                            // Вызываем цель sonar:sonar Maven плагина
                            // Передаем projectKey, который может быть переопределен переменной окружения
                            // или уже указан в pom.xml (<sonar.projectKey>)
                            // Если projectKey указан в pom.xml, можно просто sh 'mvn sonar:sonar -f my-app/pom.xml'
                            // Если нет, нужно передать явно:
                            // sh 'mvn sonar:sonar -Dsonar.projectKey=my-java-app -f my-app/pom.xml'
                            sh 'mvn sonar:sonar -f my-app/pom.xml'
                        }
                        echo 'Анализ SonarQube завершен.'
                    }
                }

                // НОВЫЙ ЭТАП: Ожидание и проверка Quality Gate
                stage('Quality Gate Check') {
                     steps {
                         echo 'Ожидание результатов SonarQube Quality Gate...'
                         // waitForQualityGate step: ждет, пока анализ SonarQube завершится
                         // и проверяет статус Quality Gate. Пайплайн будет ждать.
                         // Если Quality Gate не пройден, этот шаг приведет к ошибке пайплайна.
                         waitForQualityGate abortPipeline: true
                         echo 'Проверка Quality Gate завершена.'
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

    4.  **Настройте Quality Gate в SonarQube:**
        *   Перейдите в веб-интерфейс SonarQube.
        *   Нажмите "Administration" -> "Configuration" -> "Quality Gates".
        *   Нажмите "Create" или выберите существующий Quality Gate (например, "Sonar way").
        *   Рекомендуется создать новый Quality Gate для вашего проекта. Назовите его (например, `My App Quality Gate`).
        *   Нажмите "Add Condition". Выберите "on New Code" (это важно!).
        *   Добавьте условия, которые должны быть соблюдены для нового кода:
            *   `Bugs`: `is greater than` `0` (цель: 0 новых багов).
            *   `Vulnerabilities`: `is greater than` `0` (цель: 0 новых уязвимостей).
            *   `Code Smells`: `is greater than` `X` (например, 5 или 10, или 0 для начала).
            *   `Coverage`: `is less than` `Y %` (например, 80% - т.е. покрытие *не должно быть* меньше 80% на *новых* строках).
        *   Вернитесь на страницу "Quality Gates". Напротив вашего проекта (`My Java Application`) в колонке "Associated Projects" выберите ваш новый Quality Gate (`My App Quality Gate`).

    5.  **Сделайте коммит и пуш, чтобы запустить обновленный пайплайн:**
        ```bash
        git add Jenkinsfile my-app/pom.xml
        git commit -m "ci: Add SonarQube analysis and Quality Gate check"
        git push origin develop
        ```

    6.  **Проверьте выполнение пайплайна:**
        *   Смотрите Jenkins Job. Должны появиться новые этапы.
        *   Этап `SonarQube Analysis` должен запустить анализ.
        *   Этап `Quality Gate Check` будет ожидать результатов от SonarQube.
        *   Перейдите в SonarQube UI, найдите ваш проект. Посмотрите отчет об анализе. Убедитесь, что Quality Gate пройден (Pass) или не пройден (Fail).
        *   В Jenkins, если Quality Gate пройден, этап `Quality Gate Check` завершится успешно. Если не пройден, этап `Quality Gate Check` упадет, и весь пайплайн будет помечен как `FAILURE`.

**Результат Lab 3:** Ваш Jenkins пайплайн теперь включает автоматический анализ кода SonarQube и проверяет статус Quality Gate, что может привести к падению сборки, если код не соответствует заданным метрикам качества/безопасности. Это мощная петля обратной связи (Feedback Loop).

---

**Lab 4: Знакомство с GitLab CI**

*   **Цель:** Понять основы GitLab CI и создать аналогичный пайплайн, определенный в файле `.gitlab-ci.yml`.
*   **Предварительные требования:**
    *   Аккаунт на GitLab.com (бесплатный тариф).
    *   Возможность клонировать репозиторий из Gitea и запушить его в GitLab.
*   **Шаги:**

    1.  **Импортируйте ваш проект из Gitea в GitLab:**
        *   Залогиньтесь на GitLab.com.
        *   Нажмите "+" в верхнем меню -> "New project/repository".
        *   Выберите "Import project" -> "Repo by URL".
        *   Git Repository URL: Вставьте URL вашего репозитория в Gitea (например, `http://ВАШ_IP_VM:3000/MyDevOpsOrg/my-java-app.git`).
        *   (Опционально) Private token: Если репозиторий в Gitea приватный, вам понадобится токен доступа из настроек вашего пользователя Gitea.
        *   Project name: `my-java-app` (или другое имя).
        *   Project slug: `my-java-app`.
        *   Visibility Level: Public или Private.
        *   Нажмите "Create project". GitLab импортирует ваш репозиторий со всей историей и ветками.

    2.  **Создайте файл `.gitlab-ci.yml` в корне репозитория GitLab:**
        *   В веб-интерфейсе GitLab перейдите в ваш репозиторий.
        *   Нажмите кнопку "+" -> "New file".
        *   Имя файла: `.gitlab-ci.yml`.

    3.  **Напишите базовый GitLab CI пайплайн:**

        ```yaml
        # .gitlab-ci.yml

        # Определяем этапы пайплайна. Jobs в одном этапе выполняются параллельно,
        # этапы выполняются последовательно.
        stages:
          - build
          - test
          - analyze

        # ----- Job для сборки (Build Stage) -----
        build_job:
          # Определяем, на каком этапе выполняется этот job
          stage: build
          # Указываем Docker образ, в котором будет выполняться job.
          # Ищем образ с предустановленным Maven и JDK.
          image: maven:3.8.6-openjdk-11 # Используйте актуальную версию, совместимую с вашим проектом
          # Список команд для выполнения в этом job
          script:
            # mvn package включает компиляцию, ресурсы, тесты и упаковку
            - cd my-app # Переходим в каталог с pom.xml
            - mvn clean package -DskipTests=true # Пропускаем тесты здесь, запустим их в отдельном job
          # Сохраняем артефакты (JAR/WAR) и потенциально кэш Maven
          artifacts:
            paths:
              - my-app/target/*.jar # Или *.war
            expire_in: 1 week # Как долго хранить артефакт
          # Кэшируем зависимости Maven для ускорения последующих сборок
          cache:
            key: "$CI_COMMIT_REF_SLUG" # Ключ кэша, уникальный для каждой ветки/тега
            paths:
              - ~/.m2/repository # Путь к локальному репозиторию Maven
            policy: pull-push # Скачивать и загружать кэш

        # ----- Job для тестов (Test Stage) -----
        test_job:
          stage: test
          image: maven:3.8.6-openjdk-11
          # Этот job зависит от успешного завершения build_job
          dependencies:
            - build_job
          script:
            - cd my-app
            - mvn test # Запускаем только тесты
          # Сохраняем отчеты о тестах как артефакты для отображения в GitLab UI
          artifacts:
            reports:
              junit:
                - my-app/target/surefire-reports/TEST-*.xml # Путь к отчетам JUnit
            paths:
              - my-app/target/surefire-reports/ # Сохраняем каталог с отчетами
            expire_in: 1 week
          cache:
            key: "$CI_COMMIT_REF_SLUG"
            paths:
              - ~/.m2/repository
            policy: pull # Только скачивать кэш, так как зависимости должны быть в кэше после сборки

        # ----- Job для SonarQube анализа (Analyze Stage) -----
        sonarqube_analyze_job:
          stage: analyze
          image: maven:3.8.6-openjdk-11
          # Этот job зависит от успешного завершения build_job
          dependencies:
            - build_job
          script:
            - cd my-app
            # Вызываем цель sonar:sonar. Передаем URL SonarQube и токен через переменные.
            # SONAR_HOST_URL и SONAR_TOKEN будут настроены в настройках CI/CD GitLab.
            - mvn verify sonar:sonar -Dsonar.projectKey=my-java-app
          # Правила, когда запускать этот job
          rules:
            # Запускать только для коммитов в ветке develop
            - if: '$CI_COMMIT_BRANCH == "develop"'

        # Правила, когда запускать пайплайн целиком (опционально, можно использовать rules в jobs)
        #workflow:
        # rules:
        #  - if: '$CI_COMMIT_BRANCH == "develop"'
        ```

    4.  **Настройте переменные CI/CD в GitLab:**
        *   В вашем проекте GitLab перейдите "Settings" -> "CI/CD".
        *   Разверните секцию "Variables".
        *   Нажмите "Add variable".
        *   Key: `SONAR_HOST_URL`
        *   Value: URL вашего SonarQube (например, `http://ВАШ_IP_VM:9000`)
        *   Type: Variable
        *   Environment scope: All (или ограничьте, если нужно).
        *   Mask variable: Да (скрыть значение в логах, если возможно).
        *   Нажмите "Add variable".
        *   Повторите шаг для `SONAR_TOKEN`:
        *   Key: `SONAR_TOKEN`
        *   Value: Токен, который вы сгенерировали в SonarQube (Lab 2, шаг 7).
        *   Type: Variable
        *   Mask variable: Да.
        *   Нажмите "Add variable".

    5.  **Сделайте коммит `.gitlab-ci.yml` и проверьте пайплайн:**
        *   После сохранения `.gitlab-ci.yml` в GitLab UI, GitLab автоматически создаст коммит и запустит пайплайн.
        *   Перейдите в "CI/CD" -> "Pipelines".
        *   Вы должны увидеть запущенный пайплайн с этапами `build`, `test`, `analyze`.
        *   Нажмите на пайплайн или отдельный job, чтобы увидеть его статус и логи.
        *   Проверьте логи `sonarqube_analyze_job`. Он должен успешно выполнить анализ.
        *   Перейдите в SonarQube UI, обновите страницу вашего проекта. Вы должны увидеть результаты анализа, запущенного из GitLab CI.

    6.  **Сравните Jenkinsfile и `.gitlab-ci.yml`:**
        *   Посмотрите на синтаксис: Groovy (Jenkinsfile) против YAML (GitLab CI).
        *   Как определяются этапы (`stages`).
        *   Как определяются отдельные задачи/шаги (Steps в Jenkins, Scripts в GitLab CI Jobs).
        *   Как указываются зависимости между этапами/job'ами (неявная последовательность этапов в обоих, `dependencies` в GitLab CI для job'ов).
        *   Как управляются артефакты и кэш.
        *   Как используются переменные окружения и секреты (в Jenkins через Credentials и `withSonarQubeEnv` или env vars, в GitLab CI через Variables).

**Результат Lab 4:** Вы успешно настроили CI пайплайн в GitLab CI, который собирает, тестирует и анализирует ваш проект. Вы увидели, как определяются пайплайны в другой популярной CI системе и сравнили подходы с Jenkinsfile.

---

**Lab 5: Практика SAST (Static Application Security Testing)**

*   **Цель:** Убедиться, что SonarQube (или GitLab SAST, если доступен) находит простые уязвимости в коде.
*   **Шаги:**

    1.  **Добавьте простую уязвимость в код вашего приложения:**
        *   Откройте исходный код вашего Java приложения (например, `my-app/src/main/java/com/mycompany/app/App.java`).
        *   Добавьте код, который SonarQube должен пометить как уязвимость или Security Hotspot. Примеры:
            *   **Хардкод пароля/ключа:**
                ```java
                public class App {
                    private static final String DB_PASSWORD = "my_super_secret_password"; // SonarQube должен найти
                    // ... ваш остальной код ...
                }
                ```
            *   **Простой пример SQL Injection (только для демонстрации, не используйте так в реальном коде!):**
                ```java
                import java.sql.*;

                public class App {
                    // ...
                    public static void unsafeSql(String userId) throws SQLException {
                        Connection conn = null; // Получите соединение
                        Statement stmt = conn.createStatement();
                        // Уязвимость: конкатенация строки из внешнего ввода напрямую в SQL
                        String sql = "SELECT * FROM users WHERE id = " + userId;
                        ResultSet rs = stmt.executeQuery(sql); // SonarQube должен найти
                        // ... обработка результатов ...
                    }
                    // ...
                }
                ```
        *   Сохраните измененный файл(ы).

    2.  **Сделайте коммит и пуш в ветку `develop`:**
        ```bash
        git add my-app/src/main/java/com/mycompany/app/App.java # Или путь к вашему файлу
        git commit -m "feat: Add intentional vulnerability for SAST demo"
        git push origin develop
        ```
        Если вы используете GitLab, пушьте в ваш GitLab репозиторий.

    3.  **Проверьте результаты анализа в SonarQube:**
        *   Дождитесь завершения пайплайна в Jenkins (или GitLab CI).
        *   Перейдите в SonarQube UI, откройте ваш проект.
        *   Посмотрите на вкладки "Vulnerabilities" и "Security Hotspots".
        *   SonarQube должен обнаружить добавленную вами "уязвимость" или "горячую точку безопасности". Изучите детали, почему SonarQube считает этот код проблемным.
        *   *Обратите внимание:* Если вы настроили Quality Gate на 0 новых уязвимостей, пайплайн в Jenkins должен упасть на этапе "Quality Gate Check".

    4.  **Проверьте GitLab SAST (если применимо и настроено):**
        *   В GitLab UI перейдите в "Security & Compliance" -> "Vulnerability Report". Если SAST был настроен и запущен, вы можете увидеть результаты здесь.
        *   Если вы создали Merge Request с этим изменением, результаты SAST часто отображаются прямо в виджете Merge Request.

**Результат Lab 5:** Вы успешно продемонстрировали, как инструменты статического анализа (SonarQube) могут автоматически находить потенциальные проблемы безопасности в вашем коде как часть CI пайплайна, реализуя часть концепции DevSecOps.

---

**Резюме Урока 2:**

Вы углубились в принципы CI и DevOps, научились определять ваш конвейер сборки и тестирования как код в `Jenkinsfile`, что дает множество преимуществ по сравнению с конфигурацией через UI. Вы успешно интегрировали SonarQube для автоматического анализа качества и безопасности кода, настроив Quality Gate для контроля. Также вы получили базовое представление и опыт работы с альтернативной CI системой - GitLab CI.

Теперь ваш CI пайплайн не только собирает и тестирует код, но и предоставляет критически важную обратную связь о его состоянии с точки зрения качества и безопасности.

**Что дальше?**

В следующем уроке мы сделаем шаг назад и сосредоточимся на основах инфраструктуры - работе с сетями и операционной системой Linux на серверах, что является фундаментом для дальнейших шагов по Continuous Delivery и развертыванию приложений.

Готовьтесь к следующему погружению!