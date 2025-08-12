# 05 - Управление секретами в GitOps

### Цель урока
Понять, почему нельзя хранить секреты в Git в открытом виде, и изучить два популярных и надежных способа безопасного управления секретами с ArgoCD: **Sealed Secrets** и **External Secrets Operator**.

### Какую проблему решаем?
Наши приложения требуют для работы чувствительные данные: пароли от баз данных, API-ключи, TLS-сертификаты. Принцип GitOps гласит, что *всё* должно быть в Git. Но если мы просто закодируем секрет в Base64 и положим `Secret.yaml` в репозиторий, любой, у кого есть доступ на чтение, сможет его расшифровать.

```yaml
# ❌ ПЛОХОЙ ПРИМЕР - НИКОГДА ТАК НЕ ДЕЛАЙТЕ!
apiVersion: v1
kind: Secret
metadata:
  name: my-db-secret
type: Opaque
data:
  # echo -n 'SuperSecretPassword' | base64
  # Этот пароль легко декодировать!
  password: U3VwZXJTZWNyZXRQYXNzd29yZA== 
```

Это огромная дыра в безопасности. Нам нужен способ хранить секреты в Git в **зашифрованном** виде, чтобы только наш Kubernetes-кластер мог их расшифровать.

---

### Подход 1: Sealed Secrets (шифрование в Git)

**Идея:** Вы шифруете ваш секрет с помощью публичного ключа, который принадлежит вашему кластеру. Зашифрованный секрет (`SealedSecret`) можно безопасно хранить в Git. Внутри кластера работает контроллер `sealed-secrets`, у которого есть приватный ключ. Этот контроллер "видит" новый `SealedSecret`, расшифровывает его и создает обычный Kubernetes `Secret`.

![Sealed Secrets Workflow](https://github.com/bitnami-labs/sealed-secrets/raw/main/img/sealed-secrets.png)

#### Практика с Sealed Secrets

**Шаг 1: Установка контроллера Sealed Secrets**
```bash
# Устанавливаем контроллер в кластер
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
```
Контроллер будет запущен в неймспейсе `kube-system` и сгенерирует пару ключей для шифрования.

**Шаг 2: Установка CLI `kubeseal`**
`kubeseal` — это утилита командной строки для шифрования секретов.
```bash
# macOS
brew install kubeseal

# Linux
wget https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/kubeseal-linux-amd64 -O kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

**Шаг 3: Создание и шифрование секрета**

1.  Создайте обычный Kubernetes `Secret` в виде YAML-файла. **Не применяйте его в кластер!**

    `my-secret.yaml`:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: my-db-secret
      namespace: my-app # Важно указать неймспейс, секрет нельзя будет расшифровать в другом!
    type: Opaque
    data:
      password: U3VwZXJTZWNyZXRQYXNzd29yZA== # echo -n 'SuperSecretPassword' | base64
    ```

2.  Используйте `kubeseal` для шифрования этого файла. `kubeseal` сам найдет публичный ключ в вашем кластере (нужен настроенный `kubeconfig`).

    ```bash
    kubeseal < my-secret.yaml > sealed-my-secret.yaml
    ```
    Или, если нужно указать путь к kubeconfig или контекст:
    ```bash
    kubeseal --controller-namespace kube-system --controller-name sealed-secrets-controller < my-secret.yaml > sealed-my-secret.yaml
    ```

3.  Взгляните на созданный `sealed-my-secret.yaml`:

    ```yaml
    apiVersion: bitnami.com/v1alpha1
    kind: SealedSecret
    metadata:
      name: my-db-secret
      namespace: my-app
    spec:
      encryptedData:
        # Вот наш зашифрованный пароль. Расшифровать его без приватного ключа невозможно.
        password: AgBy3i4OJSWK+Pi4w+...
      template:
        metadata:
          name: my-db-secret
          namespace: my-app
        type: Opaque
    ```

**Шаг 4: Коммит и развертывание через ArgoCD**

Теперь вы можете безопасно закоммитить `sealed-my-secret.yaml` в ваш Git-репозиторий. ArgoCD развернет этот `SealedSecret` в кластер, контроллер `sealed-secrets` его расшифрует, и ваше приложение увидит обычный `Secret` с именем `my-db-secret`.

*   **Плюсы:** Полностью вписывается в GitOps. Всё, что нужно, хранится в Git.
*   **Минусы:** Требуется CLI `kubeseal`. Если кластер "умрет" вместе с приватным ключом, все зашифрованные секреты станут бесполезны (нужно делать бэкап ключа!).

---

### Подход 2: External Secrets Operator (интеграция с внешними хранилищами)

**Идея:** Вместо того чтобы хранить зашифрованные секреты в Git, мы храним только *ссылки* на секреты, которые лежат в специализированном хранилище (Vault, AWS Secrets Manager, Google Secret Manager и т.д.). В кластере работает `External Secrets Operator` (ESO), который читает эти ссылки, подключается к внешнему хранилищу, забирает оттуда секреты и создает обычные Kubernetes `Secret`.

![External Secrets Workflow](https://external-secrets.io/latest/images/eso-flow.png)

#### Практика с External Secrets Operator

**Шаг 1: Установка ESO**
Установка производится через Helm, что хорошо согласуется с GitOps.
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

**Шаг 2: Настройка доступа к хранилищу (`SecretStore`)**
Вы должны создать ресурс `SecretStore` (или `ClusterSecretStore` для всего кластера), который "научит" ESO, как подключаться к вашему хранилищу.

**Пример для AWS Secrets Manager:**
```yaml
# secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: my-app
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      # Настройка аутентификации (например, через IRSA для EKS)
      auth:
        jwt:
          serviceAccountRef:
            name: my-app-sa
```
Этот манифест также коммитится в Git.

**Шаг 3: Создание `ExternalSecret`**
Теперь, вместо `Secret` или `SealedSecret`, вы создаете `ExternalSecret`, который ссылается на `SecretStore` и на конкретный секрет во внешнем хранилище.

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-db-external-secret
  namespace: my-app
spec:
  # Ссылка на наш SecretStore
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore

  # Имя секрета, который будет создан в Kubernetes
  target:
    name: my-db-secret # Приложение будет использовать этот секрет
    creationPolicy: Owner

  # Описываем, какие данные забрать из внешнего хранилища
  data:
  - secretKey: password # Ключ в Kubernetes Secret
    remoteRef:
      key: /production/myapp/database # Путь к секрету в AWS SM
      property: password # Поле внутри секрета в AWS SM
```

**Шаг 4: Коммит и развертывание через ArgoCD**
Вы коммитите `external-secret.yaml` в Git. ArgoCD его применяет. ESO "видит" новый `ExternalSecret`, идет в AWS Secrets Manager, забирает значение и создает `Secret` с именем `my-db-secret` в Kubernetes.

*   **Плюсы:** Централизованное управление секретами. Используются мощные, аудируемые хранилища. Не нужно шифровать/расшифровывать файлы вручную.
*   **Минусы:** Добавляется внешняя зависимость (Vault, AWS SM и т.д.). Требуется настройка аутентификации между кластером и хранилищем.

### Какой подход выбрать?

*   **Sealed Secrets:** Отличный выбор для небольших и средних проектов, когда вы хотите простое, самодостаточное решение, полностью работающее в рамках Git и Kubernetes.
*   **External Secrets Operator:** Стандарт де-факто для крупных enterprise-систем, где уже есть централизованное хранилище секретов (Vault, AWS/GCP/Azure-аналоги), и требуется строгий аудит и ротация ключей.

Оба подхода прекрасно интегрируются с ArgoCD и позволяют безопасно управлять секретами в GitOps.