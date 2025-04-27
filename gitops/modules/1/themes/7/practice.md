### Методичка по ДЗ: Модуль 1, Тема 7

**Название ДЗ:** Установка локального кластера Kubernetes

**Цель ДЗ:** Научиться использовать простой инструмент (Kind или Minikube) для развертывания рабочего локального кластера Kubernetes на своей машине для разработки и тестирования.

**Необходимые условия/Инструменты:**

1.  Компьютер (Linux, macOS, Windows).
2.  Права администратора для установки программ.
3.  **Для Kind:** Установленный Docker.
4.  **Для Minikube:** Установленный Docker, или другой драйвер виртуализации (VirtualBox, Hyper-V). Docker Desktop с включенным Kubernetes также может быть альтернативой, но ДЗ просит использовать Kind или Minikube как отдельные инструменты.

**Подробные шаги (Выберите один из инструментов: Kind или Minikube):**

**Вариант 1: Использование Kind**

Kind (Kubernetes in Docker) запускает кластеры Kubernetes как контейнеры Docker. Это быстро и легко.

**Шаг 1.1: Установите Docker**

Если у вас еще нет Docker, установите его с официального сайта (Docker Desktop для Windows/macOS или Docker Engine для Linux). Убедитесь, что демон Docker запущен.

**Шаг 1.2: Установите Kind**

*   **Linux:**
    ```bash
    curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64 # Уточните актуальную версию
    chmod +x ./kind
    sudo mv ./kind /usr/local/bin/kind
    ```
*   **macOS (с Homebrew):**
    ```bash
    brew install kind
    ```
*   **Windows (с Chocolatey):**
    ```bash
    choco install kind
    ```
*   **Windows (с Scoop):**
    ```bash
    scoop install kind
    ```
*   **Убедитесь, что `kind` установлен:**
    ```bash
    kind version
    ```

**Шаг 1.3: Установите `kubectl`**

Если `kubectl` не установлен, установите его, следуя официальной документации Kubernetes: [https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.ks.io/docs/tasks/tools/install-kubectl/)

*   **Убедитесь, что `kubectl` установлен:**
    ```bash
    kubectl version --client
    ```

**Шаг 1.4: Создайте кластер Kind**

Это самая простая команда для создания кластера с настройками по умолчанию.

```bash
kind create cluster --name my-kind-cluster # Можно не указывать --name, тогда будет 'kind'
```

*   **Пояснения:** Kind создаст контейнер Docker, скачает образы Control Plane и Worker Node, запустит их и настроит `kubectl` для взаимодействия с новым кластером. Это может занять несколько минут.

**Шаг 1.5: Настройте `kubectl` (если не настроилось автоматически)**

Обычно Kind автоматически обновляет ваш файл `$HOME/.kube/config`, чтобы указать на новый кластер. Если нет, или у вас много кластеров, вы можете вручную выбрать контекст:

```bash
kubectl config use-context kind-my-kind-cluster # Замените 'my-kind-cluster' если назвали его иначе
```

**Вариант 2: Использование Minikube**

Minikube запускает однонодовый кластер Kubernetes внутри виртуальной машины или контейнера.

**Шаг 2.1: Установите виртуализацию или Docker**

Minikube может использовать разные "драйверы". Наиболее популярные: Docker, VirtualBox, Hyper-V. Убедитесь, что у вас установлен один из них и выбран в качестве драйвера для Minikube. Docker - хороший выбор, так как он часто уже установлен.

**Шаг 2.2: Установите Minikube**

Следуйте официальной документации Minikube: [https://minikube.sigs.k8s.io/docs/start/](https://minikube.sigs.k8s.io/docs/start/)

*   **Linux:**
    ```bash
    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 # Уточните актуальную версию и архитектуру
    chmod +x minikube
    sudo mv minikube /usr/local/bin/
    ```
*   **macOS (с Homebrew):**
    ```bash
    brew install minikube
    ```
*   **Windows (с Chocolatey):**
    ```bash
    choco install minikube
    ```
*   **Убедитесь, что `minikube` установлен:**
    ```bash
    minikube version
    ```

**Шаг 2.3: Установите `kubectl`**

Как и для Kind, если `kubectl` не установлен, установите его по официальной документации.

*   **Убедитесь, что `kubectl` установлен:**
    ```bash
    kubectl version --client
    ```

**Шаг 2.4: Запустите кластер Minikube**

Выберите драйвер (например, `docker`).

```bash
minikube start --driver=docker # Или --driver=virtualbox, --driver=hyperv и т.д.
```

*   **Пояснения:** Minikube скачает необходимые образы, создаст виртуальную машину или контейнер (в зависимости от драйвера) и запустит в нем однонодовый кластер Kubernetes. Это может занять несколько минут.

**Шаг 2.5: Настройте `kubectl`**

Minikube автоматически настраивает `kubectl` для взаимодействия с запущенным кластером. Если у вас несколько кластеров или контекстов, вы можете явно переключиться:

```bash
kubectl config use-context minikube
```

**Шаги для проверки (едины для Kind и Minikube):**

**Шаг 3: Проверьте информацию о кластере**

Используйте команду `kubectl cluster-info`, чтобы получить информацию о мастере и других компонентах кластера.

```bash
kubectl cluster-info
```

*   **Ожидаемый результат:** Должен показать адреса Kubernetes master и CoreDNS (или kube-dns).

**Шаг 4: Проверьте список узлов (нод)**

Используйте команду `kubectl get nodes`, чтобы увидеть рабочие ноды в кластере.

```bash
kubectl get nodes
```

*   **Ожидаемый результат:** Должен показать одну ноду (по умолчанию для Kind и Minikube) со статусом `Ready`.

**Шаг 5: Проверьте взаимодействие с кластером**

Попробуйте получить список подов во всех неймспейсах.

```bash
kubectl get pods --all-namespaces
```

*   **Ожидаемый результат:** Должен показать список системных подов (контроллеры, DNS и т.д.) в `kube-system` и других системных неймспейсах. Это подтверждает, что `kubectl` успешно взаимодействует с кластером.

**Проверка результатов (Измеряемость):**

1.  Опишите, какой инструмент (Kind или Minikube) вы выбрали и какие шаги выполнили для его установки.
2.  Предоставьте вывод команды `kubectl cluster-info`.
3.  Предоставьте вывод команды `kubectl get nodes`.
4.  (Опционально, но желательно) Предоставьте вывод команды `kubectl get pods --all-namespaces`.