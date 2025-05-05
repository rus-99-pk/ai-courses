**УРОК 1: Системы Контроля Версий и Автоматизация Сборки**

**Цель:** Понять фундаментальные принципы организации разработки в современном мире (DevOps), освоить продвинутые техники работы с Git и настроить базовый конвейер автоматической сборки проекта с использованием Maven и Jenkins, связанный с системой контроля версий Gitea.

**Результат урока:** У вас будет настроен локальный Git-репозиторий, связанный с удаленным репозиторием в Gitea. Проект на Maven будет собираться локально, а затем автоматически (при каждом изменении кода в Git) собираться Jenkins'ом.

**Предварительные требования:**

*   Базовое понимание работы с командной строкой Linux.
*   Установленный Git на вашей рабочей машине.
*   Понимание основ Java и Maven (как работает `pom.xml`, что такое зависимости).
*   Виртуальная машина (или две, если хотите разделить сервисы) с установленной ОС Linux (например, Ubuntu Server 20.04/22.04 или CentOS Stream 8/9). Для практики подойдет VM с 4GB RAM и 2 ядрами CPU, 40GB диска.
*   Доступ в интернет с VM для скачивания ПО.

**Техническая подготовка (на Вашей VM):**

1.  **Обновление системы:**
    ```bash
    sudo apt update
    sudo apt upgrade -y
    ```
    (Используйте `yum update -y` для CentOS/RHEL).

2.  **Установка необходимого ПО:**
    *   **Git:**
        ```bash
        sudo apt install git -y
        ```
        (Используйте `yum install git -y` для CentOS/RHEL).
    *   **Java Development Kit (JDK):** Maven требует Java. Установите OpenJDK 11 или выше.
        ```bash
        sudo apt install openjdk-11-jdk -y
        # Или для последней LTS:
        # sudo apt install openjdk-17-jdk -y
        ```
        (Используйте `yum install java-11-openjdk-devel -y` для CentOS/RHEL).
        Проверьте установку: `java -version`, `javac -version`.
    *   **Maven:**
        ```bash
        sudo apt install maven -y
        ```
        (Используйте `yum install maven -y` для CentOS/RHEL).
        Проверьте установку: `mvn -v`.

---

**ТЕОРЕТИЧЕСКИЙ БЛОК (Краткий обзор)**

*   **Жизненный цикл ПО (SDLC):**
    *   **Классический (Waterfall):** Последовательные этапы (анализ, проектирование, разработка, тестирование, внедрение, поддержка). Медленно, высокий риск ошибок, трудно менять требования.
    *   **Agile:** Инкрементальная и итеративная разработка. Короткие циклы (спринты). Частая обратная связь.
    *   **DevOps:** Культурное и техническое движение, направленное на улучшение взаимодействия между разработкой (Dev) и эксплуатацией (Ops). Основная цель - сокращение цикла поставки ценности, повышение частоты и надежности релизов.
*   **Проблематика "функциональных колодцев":** Ситуация, когда команды (Dev, Ops, QA) работают изолированно, передавая результаты "через стену". Приводит к замедлению, недопониманию, конфликтам и снижению качества. DevOps стремится сломать эти "колодцы".
*   **Бережливое производство (Lean):** Принципы, применимые к IT:
    *   Определение ценности с точки зрения клиента.
    *   Построение потока создания ценности (Value Stream). Визуализация и оптимизация всех шагов от идеи до получения ценности пользователем.
    *   Обеспечение непрерывного потока (Flow).
    *   Вытягивание (Pull) вместо выталкивания (Push).
    *   Стремление к совершенству.
    *   DevOps берет многие идеи из Lean для оптимизации Value Stream.
*   **Git Workflows:**
    *   **Feature Branch Workflow:** Простой и распространенный. Для каждой новой фичи/исправления создается отдельная ветка от `develop` (или `main`). Работа идет в этой ветке. По завершении делается Pull/Merge Request для ревью и слияния обратно в `develop`.
    *   **GitFlow:** Более формализованный. Использует долгоживущие ветки (`main` / `master` для продакшена, `develop` для текущей разработки) и короткоживущие (`feature` для фич, `release` для подготовки релиза, `hotfix` для быстрых исправлений на продакшене). Хорош для проектов с четким циклом релизов, но может быть избыточным.
    *   *В этом уроке сосредоточимся на Feature Branch Workflow как более простом для старта.*

---

**ПРАКТИЧЕСКИЙ БЛОК (Hands-on Labs)**

**Lab 1: Продвинутая Практика Git и Работа с Ветками**

*   **Цель:** Закрепить навыки работы с ветками, слиянием и разрешением конфликтов.
*   **Шаги:**

    1.  **Создайте новый каталог и инициализируйте Git репозиторий:**
        ```bash
        mkdir my-devops-app
        cd my-devops-app
        git init
        ```

    2.  **Создайте первый файл и сделайте начальный коммит:**
        ```bash
        echo "Initial project setup" > README.md
        git add README.md
        git commit -m "feat: Initial project setup"
        ```

    3.  **Создайте ветку `develop` и переключитесь на нее:**
        ```bash
        git branch develop
        git checkout develop
        ```
        Теперь ветка `develop` является основной для текущей разработки.

    4.  **Создайте ветку для новой фичи (Feature Branch) и переключитесь на нее:**
        ```bash
        git checkout -b feature/add-greeting-message develop
        ```
        Вы создали ветку `feature/add-greeting-message` на основе ветки `develop` и сразу переключились на нее.

    5.  **Внесите изменения в Feature Branch:**
        ```bash
        echo "Hello, DevOps World!" >> README.md
        git add README.md
        git commit -m "feat: Add greeting message"
        ```

    6.  **Переключитесь обратно на ветку `develop`:**
        ```bash
        git checkout develop
        ```

    7.  **Внесите *конфликтующие* изменения в ветку `develop`:**
        ```bash
        echo "Project documentation" > README.md # Перезаписываем файл
        echo "Added setup instructions." >> README.md
        git add README.md
        git commit -m "docs: Add setup instructions"
        ```
        Теперь в `develop` `README.md` содержит другой текст, чем в `feature/add-greeting-message`.

    8.  **Попробуйте слить ветку `feature/add-greeting-message` в `develop` (метод `merge`):**
        ```bash
        git merge feature/add-greeting-message
        ```
        Вы увидите сообщение о конфликте слияния (`CONFLICT (content): Merge conflict in README.md`).

    9.  **Разрешите конфликт слияния:**
        *   Откройте `README.md` в текстовом редакторе.
        *   Вы увидите маркеры конфликта (`<<<<<<<`, `=======`, `>>>>>>>`).
        *   Вручную отредактируйте файл так, чтобы он содержал желаемый итоговый текст (например, обе строки или новую объединенную версию).
        *   Пример желаемого содержимого:
            ```
            Project documentation
            Added setup instructions.
            Hello, DevOps World!
            ```
        *   Сохраните файл.
        *   Добавьте разрешенный файл в staging:
            ```bash
            git add README.md
            ```
        *   Завершите слияние (Git автоматически создаст коммит слияния):
            ```bash
            git commit
            ```
            (Редактор откроется для ввода сообщения коммита слияния. Сохраните стандартное или отредактируйте).

    10. **(Опционально) Попробуйте метод `rebase` вместо `merge`:**
        *   *Сбросьте ветки к состоянию до попытки слияния:* **Будьте осторожны, это сбросит историю!**
            ```bash
            git reset --hard HEAD~1 # Сбросить develop на коммит до слияния
            git checkout feature/add-greeting-message
            git reset --hard HEAD~1 # Сбросить фичу на коммит до ее создания (или до коммита фичи) - убедитесь, что вы на правильном коммите! Проще удалить и создать заново ветку фичи от того же коммита develop, откуда она начиналась.
            # Альтернативно, если коммит на develop был один, можно просто git reset --hard origin/develop (если есть origin) или HEAD~1
            # Самый надежный способ для практики: удалить ветку feature и создать ее заново от текущего develop
            git branch -D feature/add-greeting-message # Принудительно удалить
            git checkout develop
            git checkout -b feature/add-greeting-message # Создать заново
            # Снова добавьте "Hello, DevOps World!" и сделайте коммит в новой ветке feature/add-greeting-message
            echo "Hello, DevOps World!" >> README.md
            git add README.md
            git commit -m "feat: Add greeting message (rebased)"
            ```
        *   *Теперь выполните rebase:* Переключитесь на Feature Branch и выполните rebase на `develop`:
            ```bash
            git checkout feature/add-greeting-message
            git rebase develop
            ```
        *   Вы снова увидите сообщение о конфликте. Разрешите его так же, как и при слиянии.
        *   Продолжите rebase:
            ```bash
            git add README.md
            git rebase --continue
            ```
        *   *Важно:* После rebase ветка `feature/add-greeting-message` будет "поверх" ветки `develop`, как если бы ваши изменения вносились *после* изменений в `develop`. История будет более линейной. *Не используйте `git rebase` для веток, которые уже были опубликованы (pushed в удаленный репозиторий), так как это переписывает историю!*
        *   Переключитесь на `develop` и выполните быстрое слияние (fast-forward merge), если `develop` не имеет новых коммитов:
            ```bash
            git checkout develop
            git merge feature/add-greeting-message
            ```

    11. **Просмотрите историю коммитов:**
        ```bash
        git log --oneline --graph --all
        ```
        Сравните историю после `merge` и после `rebase`.

**Результат Lab 1:** У вас есть локальный репозиторий с несколькими ветками, вы умеете создавать ветки фич, делать коммиты, сливать изменения (`merge`) и переосновывать их (`rebase`), а также разрешать конфликты.

---

**Lab 2: Установка и Настройка Gitea**

*   **Цель:** Развернуть централизованный Git-репозиторий для командной работы.
*   **Шаги:**

    1.  **Выберите метод установки Gitea:** Самый простой способ для быстрой установки - использовать Docker. Если у вас нет Docker на VM, установите его:
        ```bash
        # Для Ubuntu:
        sudo apt update
        sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        sudo apt update
        sudo apt install docker-ce docker-ce-cli containerd.io -y
        sudo usermod -aG docker $USER # Добавьте текущего пользователя в группу docker
        # Выйдите и войдите снова или выполните newgrp docker для применения изменений группы
        docker run hello-world # Проверьте установку
        ```
        (Для CentOS/RHEL смотрите документацию Docker Compose).

    2.  **Запустите Gitea в Docker:** Используйте официальный образ. Простой вариант с сохранением данных в Docker Volume и использованием встроенной базы данных SQLite:
        ```bash
        mkdir $HOME/gitea
        docker run -d --name=gitea -p 3000:3000 -p 2222:22 \
          -v $HOME/gitea:/data \
          gitea/gitea:latest
        ```
        *   `-d`: Запуск в фоновом режиме.
        *   `--name=gitea`: Имя контейнера.
        *   `-p 3000:3000`: Проброс HTTP/HTTPS порта Gitea.
        *   `-p 2222:22`: Проброс SSH порта Gitea (изменяем внешний порт на 2222, чтобы не конфликтовать с системным SSH).
        *   `-v $HOME/gitea:/data`: Монтирование локального каталога для сохранения данных Gitea.
        *   `gitea/gitea:latest`: Имя образа Docker.

    3.  **Выполните первичную настройку Gitea через веб-интерфейс:**
        *   Откройте браузер и перейдите по адресу вашей VM и порту 3000 (например, `http://ВАШ_IP_VM:3000`).
        *   Нажмите кнопку "Зарегистрироваться".
        *   На экране настройки Gitea укажите:
            *   Тип базы данных: SQLite3 (для простоты в рамках урока).
            *   Путь к базе данных: `/data/gitea/gitea.db` (по умолчанию).
            *   Путь к корневой директории Gitea: `/data/gitea` (по умолчанию).
            *   Путь к корневой директории репозиториев: `/data/git/repositories` (по умолчанию).
            *   **Базовый URL приложения:** Укажите протокол и IP адрес/домен вашей VM, с портом 3000 (например, `http://ВАШ_IP_VM:3000/`).
            *   **Порт для SSH сервера домена Gitea:** 2222 (тот, который мы пробросили).
            *   **Домен сервера Gitea:** IP адрес или домен вашей VM (например, `ВАШ_IP_VM`).
            *   Оставьте остальные настройки по умолчанию или настройте по желанию.
        *   Нажмите "Установить Gitea".

    4.  **Зарегистрируйте нового пользователя (первый пользователь станет администратором):**
        *   После установки вы попадете на страницу входа/регистрации.
        *   Зарегистрируйте пользователя и запомните логин/пароль. Этот пользователь будет администратором и вашим основным пользователем.

    5.  **Создайте новую организацию/группу (опционально, но хорошая практика):**
        *   После входа, нажмите "+" в верхнем меню -> "Новая организация".
        *   Введите имя организации (например, `MyDevOpsOrg`).
        *   Нажмите "Создать организацию".

    6.  **Создайте новый репозиторий для вашего тестового приложения:**
        *   Перейдите на страницу вашей организации (или оставайтесь на своей личной странице).
        *   Нажмите "+" в верхнем меню -> "Новый репозиторий".
        *   Владелец: Выберите вашу организацию (или себя).
        *   Название репозитория: `my-java-app` (или другое название для вашего проекта).
        *   Описание: Краткое описание.
        *   Выберите видимость (Public или Private). Для практики Public проще.
        *   Снимите галочки "Инициализировать репозиторий..." (мы запушим существующий локальный репозиторий).
        *   Нажмите "Создать репозиторий".

    7.  **Свяжите локальный репозиторий с удаленным в Gitea:**
        *   Перейдите в каталог вашего локального Git репозитория (`cd my-devops-app`).
        *   На странице созданного репозитория в Gitea скопируйте URL для клонирования (HTTP или SSH). Используйте HTTP пока, это проще: `http://ВАШ_IP_VM:3000/MyDevOpsOrg/my-java-app.git` (замените данные на свои).
        *   Добавьте удаленный репозиторий:
            ```bash
            git remote add origin http://ВАШ_IP_VM:3000/MyDevOpsOrg/my-java-app.git
            ```
        *   Проверьте удаленные репозитории:
            ```bash
            git remote -v
            ```
        *   Запушьте ваши локальные ветки (`develop`, `feature/add-greeting-message` и `main` / `master`) в удаленный репозиторий:
            ```bash
            git push -u origin main # Или master, если так называется ваша основная ветка
            git push -u origin develop
            git push -u origin feature/add-greeting-message
            ```
            (-u устанавливает вышестоящую ветку).

    8.  **Проверьте Gitea:** Обновите страницу репозитория в Gitea. Вы увидите свои файлы, ветки, коммиты.

**Результат Lab 2:** Установленный и настроенный Gitea, в который запушен код вашего проекта. Умение работать с удаленными репозиториями.

---

**Lab 3: Подготовка и Практика с Maven Проектом**

*   **Цель:** Иметь готовый Maven проект и уметь выполнять его сборку локально.
*   **Шаги:**

    1.  **Если у вас уже есть Maven проект:**
        *   Скопируйте его файлы в каталог `my-devops-app`, созданный в Lab 1.
        *   Убедитесь, что файл `pom.xml` находится в корне репозитория.
        *   Добавьте новые файлы в Git:
            ```bash
            git status # Посмотреть добавленные файлы
            git add .
            git commit -m "feat: Add initial Maven project code"
            git push origin develop # Или ветку, куда хотите добавить
            ```

    2.  **Если у вас нет Maven проекта, создайте простой:**
        *   Убедитесь, что находитесь в каталоге `my-devops-app`.
        *   Используйте Maven Archetype для создания простого Java проекта:
            ```bash
            mvn archetype:generate \
              -DgroupId=com.mycompany.app \
              -DartifactId=my-app \
              -DarchetypeArtifactId=maven-archetype-quickstart \
              -DarchetypeVersion=1.4 \
              -DinteractiveMode=false
            ```
            Эта команда создаст каталог `my-app` внутри `my-devops-app` с базовым Java классом и тестом.
        *   Перейдите в созданный каталог проекта:
            ```bash
            cd my-app
            ```
        *   Добавьте файлы проекта в Git (вы находитесь внутри `my-devops-app`, где инициализирован Git):
            ```bash
            cd .. # Вернитесь в корень my-devops-app
            git status
            git add my-app/
            git commit -m "feat: Add simple Maven quickstart project"
            git push origin develop
            ```

    3.  **Поймите структуру Maven проекта:**
        *   Откройте файл `my-app/pom.xml`. Изучите секции `<groupId>`, `<artifactId>`, `<version>`, `<properties>`, `<dependencies>`, `<build>`. Это "координаты" вашего проекта, его зависимости и настройки сборки.
        *   Посмотрите структуру каталогов: `src/main/java` (основной код), `src/test/java` (тесты), `target` (результаты сборки).

    4.  **Выполните различные фазы сборки Maven из командной строки:**
        *   Перейдите в корневой каталог Maven проекта (где находится `pom.xml`):
            ```bash
            cd my-devops-app/my-app # Если вы создали my-app внутри my-devops-app
            ```
        *   Очистка каталога `target`:
            ```bash
            mvn clean
            ```
        *   Компиляция исходного кода:
            ```bash
            mvn compile
            ```
        *   Запуск тестов:
            ```bash
            mvn test
            ```
        *   Упаковка артефакта (JAR или WAR файла):
            ```bash
            mvn package
            ```
            После этой команды в каталоге `target` должен появиться ваш собранный JAR (например, `my-app-1.0-SNAPSHOT.jar`).
        *   Установка артефакта в локальный репозиторий Maven (`~/.m2/repository`):
            ```bash
            mvn install
            ```
        *   *Важно:* Помните о Maven Wrapper (`mvnw` / `mvnw.cmd`) как лучшей практике для обеспечения одинаковой версии Maven у всех разработчиков и CI/CD. В реальном проекте его стоит использовать.

**Результат Lab 3:** У вас есть Maven проект в репозитории Gitea, и вы можете успешно собирать его локально с помощью `mvn package`.

---

**Lab 4: Базовая Установка и Настройка Jenkins**

*   **Цель:** Развернуть Jenkins для автоматизации задач.
*   **Шаги:**

    1.  **Выберите место для установки Jenkins:** Можно установить на ту же VM, что и Gitea, если ресурсов достаточно. Для Middle+/Senior уровня лучше разнести сервисы на разные VM (или контейнеры), но для первого урока сойдет и одна VM, если она достаточно мощная.

    2.  **Установите Jenkins:** Следуйте официальной документации Jenkins. Для Ubuntu:
        ```bash
        sudo apt update
        sudo apt install fontconfig openjdk-11-jre -y # Jenkins требует Java JRE
        curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
          /usr/share/keyrings/jenkins-archive-keyring.gpg > /dev/null
        echo "deb [signed-by=/usr/share/keyrings/jenkins-archive-keyring.gpg] \
          https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
          /etc/apt/sources.list.d/jenkins.list > /dev/null
        sudo apt update
        sudo apt install jenkins -y
        ```
        (Для CentOS/RHEL смотрите документацию Jenkins).

    3.  **Запустите Jenkins и проверьте его статус:**
        ```bash
        sudo systemctl start jenkins
        sudo systemctl status jenkins
        ```
        Jenkins должен слушать порт 8080 по умолчанию.

    4.  **Выполните первичную настройку Jenkins через веб-интерфейс:**
        *   Откройте браузер и перейдите по адресу вашей VM и порту 8080 (например, `http://ВАШ_IP_VM:8080`).
        *   Вам будет предложено разблокировать Jenkins. Начальный пароль находится в файле на сервере. Выполните команду на VM:
            ```bash
            sudo cat /var/lib/jenkins/secrets/initialAdminPassword
            ```
            Скопируйте этот пароль и вставьте на странице Jenkins.
        *   Нажмите "Continue".
        *   Вам будет предложено установить плагины. Для начала выберите "Install suggested plugins". Дождитесь завершения установки.
        *   Создайте первого пользователя-администратора. Заполните данные и нажмите "Save and Finish".
        *   Настройте URL Jenkins (если требуется, обычно автоопределяется). Нажмите "Save and Finish".
        *   Вы увидите страницу "Welcome to Jenkins!".

    5.  **Установите необходимые плагины вручную (если они не были установлены):**
        *   Перейдите в "Manage Jenkins" -> "Manage Plugins".
        *   Перейдите на вкладку "Available plugins".
        *   В поле фильтра введите "Git". Выберите "Git plugin" и "Git Client Plugin".
        *   В поле фильтра введите "Maven". Выберите "Maven Integration plugin".
        *   Выберите "Install without restart" или "Download now and install after restart". Если требуется перезапуск, Jenkins предложит это сделать.

    6.  **Настройте глобальные инструменты (JDK и Maven):** Jenkins должен знать, где найти Java и Maven на агенте, где будет выполняться сборка (в данном случае, прямо на мастере Jenkins).
        *   Перейдите в "Manage Jenkins" -> "Global Tool Configuration".
        *   Найдите секцию JDK. Нажмите "Add JDK".
            *   Снимите галочку "Install automatically".
            *   Укажите имя (например, `JDK_11_Local`).
            *   Укажите `JAVA_HOME` путь до вашей установки JDK на VM (например, `/usr/lib/jvm/java-1.11.0-openjdk-amd64`). Этот путь можно найти командой `readlink -f $(which java)`.
        *   Найдите секцию Maven. Нажмите "Add Maven".
            *   Снимите галочку "Install automatically".
            *   Укажите имя (например, `Maven_Local`).
            *   Укажите `MAVEN_HOME` путь до вашей установки Maven на VM (например, `/usr/share/maven`). Этот путь можно найти командой `which mvn`, а затем посмотреть симлинки, если есть.
        *   Нажмите "Save" внизу страницы.

**Результат Lab 4:** Установленный и настроенный Jenkins с необходимыми плагинами и путями к JDK/Maven.

---

**Lab 5: Создание Автоматизированного Build Job в Jenkins**

*   **Цель:** Настроить Jenkins на автоматическую сборку проекта из Gitea при каждом коммите.
*   **Шаги:**

    1.  **Создайте новый Jenkins Job:**
        *   На главной странице Jenkins нажмите "New Item".
        *   Введите имя элемента (например, `my-java-app-build-freestyle`).
        *   Выберите тип "Freestyle project".
        *   Нажмите "OK".

    2.  **Настройте Job:** Вы попадете на страницу конфигурации Job'а.

        *   **General:** (Опционально) Добавьте описание.
        *   **Source Code Management:**
            *   Выберите "Git".
            *   **Repository URL:** Вставьте URL вашего репозитория из Gitea (например, `http://ВАШ_IP_VM:3000/MyDevOpsOrg/my-java-app.git`).
            *   **Credentials:** Здесь нужно добавить учетные данные для доступа Jenkins к Gitea.
                *   Нажмите кнопку "Add" -> "Jenkins".
                *   Выберите тип учетных данных (например, "Username with password").
                *   Scope: Global.
                *   Username: Ваш логин в Gitea.
                *   Password: Ваш пароль в Gitea.
                *   ID: Придумайте уникальный ID (например, `gitea-user-pass`).
                *   Description: Краткое описание.
                *   Нажмите "Add".
                *   Выберите только что созданные учетные данные из выпадающего списка "Credentials".
            *   **Branches to build:** Укажите ветку, которую Jenkins будет отслеживать и собирать. Нажмите "Add Branch", выберите "Branch Specifier (blank for 'any')". В поле "Name" введите `develop` (или другую ветку, которую хотите собирать).

        *   **Build Triggers:** Настройте автоматический запуск сборки при изменении в Git.
            *   Поставьте галочку "Generic Webhook Trigger".
            *   *Опционально:* Можно настроить токен для безопасности, но для первого урока пропустим этот шаг. Запишите URL, который предоставляет Jenkins (он будет отображен после сохранения настроек Job'а). Пример: `http://ВАШ_IP_VM:8080/generic-webhook-trigger/invoke`.

        *   **Build Environment:** Оставьте по умолчанию для начала.
        *   **Build:**
            *   Нажмите "Add build step".
            *   Выберите "Invoke top-level Maven targets".
            *   Maven Version: Выберите имя вашей глобальной настройки Maven (например, `Maven_Local`).
            *   Goals: Введите цели Maven, которые нужно выполнить. Для сборки обычно используют `clean package`.
            *   (Опционально) Advanced -> POM: Если ваш файл `pom.xml` находится не в корне репозитория (например, в подкаталоге `my-app`), укажите путь к нему относительно корня репозитория (например, `my-app/pom.xml`).

        *   **Post-build Actions:** Оставьте по умолчанию для начала.

    3.  **Сохраните Job:** Нажмите "Save".

    4.  **Настройте Webhook в Gitea:**
        *   Перейдите в ваш репозиторий в Gitea (`MyDevOpsOrg/my-java-app`).
        *   Настройки репозитория (Settings) -> Webhooks.
        *   Нажмите "Add Webhook" -> "Gitea".
        *   **Target URL:** Вставьте URL, который предоставил Jenkins в настройках Job'а для Generic Webhook Trigger (например, `http://ВАШ_IP_VM:8080/generic-webhook-trigger/invoke`).
        *   **HTTP Method:** POST.
        *   **ContentType:** application/json.
        *   **Secret:** Оставьте пустым (если не настраивали токен в Jenkins).
        *   **Trigger On:** Выберите события, которые должны запускать webhook. Оставьте "Push Events" (сборка при коммите). Можно убрать остальные (Tag Push, Pull Request и т.д.) пока.
        *   **Active:** Поставьте галочку.
        *   Нажмите "Add Webhook".
        *   *Важно:* После создания webhook, Gitea предложит "Test Delivery". Нажмите ее. Jenkins должен получить запрос (хоть и пустой) и, возможно, запустить сборку или показать ошибку (если триггер настроен строго на push).

    5.  **Проверьте автоматическую сборку:**
        *   Вернитесь в каталог вашего локального Git репозитория (`my-devops-app`).
        *   Внесите небольшое изменение в любой файл (например, добавьте строку в `README.md`).
        *   Сделайте коммит и запушьте изменение в ветку `develop` в Gitea:
            ```bash
            git add .
            git commit -m "test: Trigger Jenkins build"
            git push origin develop
            ```
        *   Перейдите в веб-интерфейс Jenkins. На главной странице или на странице вашего Job'а (`my-java-app-build-freestyle`), вы должны увидеть, что запустилась новая сборка.
        *   Нажмите на номер сборки (например, `#1`).
        *   Нажмите "Console Output". Внимательно просмотрите логи сборки. Вы должны увидеть, как Jenkins клонирует репозиторий, запускает Maven (`mvn clean package`) и результаты сборки. Убедитесь, что сборка завершилась успешно (BUILD SUCCESS).

    6.  **Попробуйте сломать сборку:** Внесите ошибку в код или `pom.xml`, запушьте. Убедитесь, что Jenkins запускает сборку и она завершается с ошибкой (BUILD FAILURE). Проанализируйте логи, чтобы понять, почему сборка упала.

**Результат Lab 5:** У вас есть Jenkins Job, который автоматически запускается при каждом коммите в Gitea, клонирует ваш проект, собирает его с помощью Maven и показывает результат сборки (успех/неудача) в логах. Это ваш первый работающий CI pipeline!

---

**Резюме Урока 1:**

Вы познакомились с базовыми концепциями DevOps, научились эффективно работать с ветками в Git, развернули свой сервер Git (Gitea) и сервер автоматизации (Jenkins). Самое главное — вы настроили автоматическую сборку вашего Maven проекта при каждом изменении кода, что является первым шагом к Continuous Integration.

**Что дальше?**

В следующем уроке мы углубимся в Continuous Integration, переведем наш Jenkins Job в формат "Pipeline as Code" (Jenkinsfile), познакомимся с GitLab CI как альтернативой и интегрируем инструменты для анализа качества и безопасности кода (SonarQube).

Готовьтесь к следующему уроку! У вас уже есть отличная база.