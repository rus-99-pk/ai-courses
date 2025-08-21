# Файл 9: Усиление безопасности кластера и Аудит (Hardening & Auditing)

Обеспечение безопасности — это непрерывный процесс. После первоначальной настройки необходимо регулярно проверять конфигурацию кластера на соответствие лучшим практикам и отслеживать все происходящие в нем действия.

## 1. Усиление безопасности (Hardening) с помощью CIS Benchmarks

**Задача, которую мы решаем:** Систематически проверять конфигурацию всех компонентов Kubernetes (API Server, etcd, Kubelet и т.д.) на соответствие общепринятым стандартам безопасности, чтобы минимизировать поверхность атаки. [1, 6, 19]

**Center for Internet Security (CIS) Benchmarks** — это набор рекомендаций, который де-факто является золотым стандартом для "закалки" (hardening) IT-систем, включая Kubernetes. [1] Рекомендации разделены на уровни (Level 1 - базовые, Level 2 - для сред с повышенными требованиями) и охватывают все компоненты кластера. [12]

### Инструмент: `kube-bench`

`kube-bench` — это open-source инструмент от Aqua Security, который автоматизирует проверку вашего кластера на соответствие CIS Kubernetes Benchmark. [7, 14, 15] Он запускается на ваших нодах (master и worker) и выдает отчет с результатами `[PASS]`, `[FAIL]` и `[WARN]`, а также рекомендации по исправлению. [16]

#### Как запустить `kube-bench`?

`kube-bench` проще всего запустить как `Job` внутри самого Kubernetes. Он требует доступа к файловой системе хоста для проверки конфигурационных файлов.

**Пример манифеста `job.yaml` для запуска `kube-bench`:**
(Взят с официального GitHub репозитория `kube-bench` и может потребовать адаптации)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          command: ["kube-bench", "run", "--targets=master,node"]
          volumeMounts:
            - name: var-lib-etcd
              mountPath: /var/lib/etcd
              readOnly: true
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-systemd
              mountPath: /etc/systemd
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
            - name: usr-bin
              mountPath: /usr/local/mount-from-host/bin
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-etcd
          hostPath:
            path: "/var/lib/etcd"
        - name: var-lib-kubelet
          hostPath:
            path: "/var/lib/kubelet"
        - name: etc-systemd
          hostPath:
            path: "/etc/systemd"
        - name: etc-kubernetes
          hostPath:
            path: "/etc/kubernetes"
        - name: usr-bin
          hostPath:
            path: "/usr/bin"
```

### Команды-методички

**1. Запустить проверку:**
```bash
kubectl apply -f job.yaml
```

**2. Подождать завершения Job:**
```bash
kubectl get pods -l job-name=kube-bench --watch
```

**3. Посмотреть результаты:**
Когда под перейдет в состояние `Completed`, можно посмотреть логи, которые и будут являться отчетом. [25]

```bash
# Замените <pod-name> на имя пода, полученное из предыдущей команды
kubectl logs <pod-name>
```

**Пример вывода `kube-bench`:**

```
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 Master Node Configuration Files
[FAIL] 1.1.1 Ensure that the API server pod specification file permissions are set to 644 or more restrictive (Automated)
...
== Remediations ==
1.1.1 Run the following command:
chmod 644 /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Нюанс для Middle DevOps:** Регулярно запускайте `kube-bench` (например, раз в неделю с помощью `CronJob`) и интегрируйте его отчеты в вашу систему мониторинга, чтобы отслеживать соответствие стандартам во времени. [7]

---

## 2. Аудит и Мониторинг

**Задача, которую мы решаем:** Ответить на вопросы "кто, что, когда и откуда сделал" в кластере. Это критически важно для расследования инцидентов, обнаружения аномалий и соответствия требованиям регуляторов. [8, 11]

### Логи Аудита Kubernetes (Audit Logs)

API Server может генерировать подробные логи всех поступающих к нему запросов. [2] Эти логи содержат информацию о:
*   **`who`**: Пользователь или ServiceAccount, совершивший действие.
*   **`what`**: Какой ресурс был затронут (`pods`, `secrets`) и какое действие было выполнено (`create`, `delete`).
*   **`when`**: Временная метка.
*   **`where`**: С какого IP-адреса пришел запрос.
*   **`responseStatus`**: Был ли запрос успешным.

**Важный нюанс:** Аудит по умолчанию **не включен** в большинстве self-hosted кластеров. [2] В managed-сервисах (EKS, GKE, AKS) его обычно можно включить одной кнопкой, и логи будут отправляться в облачный сервис логирования (CloudWatch, Google Cloud Logging и т.д.). [5]

#### Настройка политики аудита

Чтобы включить аудит, необходимо передать API Server'у два флага:
*   `--audit-log-path`: Куда сохранять файл с логами.
*   `--audit-policy-file`: Путь к YAML-файлу, который описывает, **что именно** логировать.

**Пример `audit-policy.yaml`:**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Не логировать "шумные" запросы, которые происходят постоянно
omitStages:
  - "RequestReceived"
rules:
  # Логировать все запросы на чтение и запись к secrets и configmaps на уровне метаданных
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]

  # Логировать тело запроса и ответа для изменений в deployments
  - level: RequestResponse
    resources:
    - group: "apps"
      resources: ["deployments"]
    verbs: ["create", "update", "patch", "delete"]

  # Логировать все остальные запросы на уровне метаданных
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

**Уровни логирования (`level`):** [13]
*   `None`: Не логировать.
*   `Metadata`: Логировать метаданные запроса (кто, что, когда), но не тело запроса/ответа.
*   `Request`: Логировать метаданные и тело запроса.
*   `RequestResponse`: Логировать все: метаданные, тело запроса и тело ответа. **Используйте с осторожностью**, так как может содержать чувствительные данные и занимать много места.

**Лучшая практика:** Отправляйте логи аудита во внешнюю, защищенную SIEM-систему (Splunk, ELK Stack, etc.) для анализа, корреляции событий и настройки алертов на подозрительные действия (например, "пользователь X удалил 10 `secrets` за 1 минуту"). [6]

### Другие инструменты для аудита

Помимо `kube-bench`, существует экосистема инструментов для статического анализа и поиска угроз: [4, 10, 18, 20]
*   **Kubescape:** Сканирует YAML-файлы, Helm-чарты и работающие кластеры на предмет мисконфигураций, уязвимостей и соответствия различным фреймворкам (NSA, MITRE ATT&CK).
*   **kube-hunter:** Запускает "пентест" вашего кластера, активно ища в нем дыры в безопасности.
*   **Checkov:** Статический анализатор для Infrastructure as Code, который поддерживает проверку манифестов Kubernetes.

## Заключение

Вы завершили курс по безопасности контейнеров в Docker и Kubernetes!

Надеюсь, этот материал оказался для вас полезным и структурированным. Помните, что безопасность — это не конечная цель, а непрерывный процесс, требующий постоянного внимания и совершенствования.

### Следующие шаги

*   **Практика:** Лучший способ закрепить знания — это применить их на практике. Попробуйте развернуть тестовый кластер и настроить все описанные механизмы: `kube-bench`, `Falco`, `Network Policies`, `Pod Security Standards`.
*   **Изучение инструментов:** Глубже изучите инструменты, которые были упомянуты: OPA/Gatekeeper, Kyverno, Istio, Linkerd. Service Mesh — это отдельная большая и важная тема в безопасности cloud-native.
*   **Сертификация:** Рассмотрите возможность сдачи экзамена **Certified Kubernetes Security Specialist (CKS)**. Подготовка к нему отлично систематизирует и углубит ваши знания.
*   **Следите за новостями:** Мир Kubernetes и безопасности постоянно меняется. Подпишитесь на рассылки от CNCF, блоги по безопасности (например, от Aqua Security, Sysdig, Palo Alto Networks) и следите за новыми CVE.

Удачи в построении безопасных и надежных систем.