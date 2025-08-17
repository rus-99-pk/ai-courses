# 2. Подготовка и Установка Драйвера Ceph CSI

Прежде чем приложения в Kubernetes смогут использовать хранилище Ceph RBD, необходимо настроить сам Ceph-кластер и установить в Kubernetes специальный драйвер.

## Шаг 1: Настройка на стороне Ceph-кластера

Предполагается, что у вас уже есть работающий Ceph-кластер. [2]

### 1. Создание пула

Для хранения RBD-образов, которые будут использоваться Kubernetes, рекомендуется создать отдельный пул. [1, 20] Это позволяет изолировать данные Kubernetes и применять к ним специфичные настройки (например, уровень репликации).

Выполните на одном из узлов Ceph или на машине с доступом к `ceph` CLI:
```bash
# Создаем пул с именем 'k8s-rbd' и 32 группами размещения (placement groups)
# Подберите количество PG в соответствии с размером вашего кластера
ceph osd pool create k8s-rbd
```

После создания пул необходимо инициализировать для использования с RBD:
```bash
rbd pool init k8s-rbd
```
Эта команда связывает пул с приложением `rbd`. [7]

### 2. Создание пользователя CephX

Из соображений безопасности не следует использовать ключ `client.admin`. Вместо этого создадим специального пользователя для Kubernetes с ограниченными правами доступа только к нужному пулу. [9, 20]

```bash
ceph auth get-or-create client.kube mon 'profile rbd' osd 'profile rbd pool=k8s-rbd' mgr 'profile rbd pool=k8s-rbd'
```

Эта команда создаст пользователя `client.kube` и выведет его ключ. **Сохраните этот ключ**, он понадобится на следующем шаге. Вывод будет выглядеть примерно так:

```
[client.kube]
    key = AQB...long...key...==
```

## Шаг 2: Установка и настройка Ceph CSI в Kubernetes

### 1. Получение информации о кластере Ceph

Для настройки CSI-драйвера нам понадобятся две вещи:
*   **FSID** кластера Ceph.
*   Адреса **мониторов** Ceph.

Получить их можно командой:
```bash
ceph mon dump
```
Вывод будет содержать `fsid` и список IP-адресов мониторов в секции `mons`. [7]

### 2. Создание ConfigMap и Secret в Kubernetes

CSI-драйверу нужно знать, как подключиться к вашему Ceph-кластеру.

**ConfigMap:** Создайте файл `csi-config-map.yaml` с FSID и адресами мониторов. [9, 20]

```yaml
# csi-config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-csi-config
  namespace: default # Укажите namespace, где будет установлен драйвер
data:
  config.json: |
    [
      {
        "clusterID": "b9127830-b0cc-4e34-aa47-9d1a2e9949a8", # <-- ВАШ FSID
        "monitors": [
          "192.168.1.10:6789", # <-- АДРЕСА ВАШИХ МОНИТОРОВ
          "192.168.1.11:6789",
          "192.168.1.12:6789"
        ]
      }
    ]
```

**Secret:** Создайте файл `csi-rbd-secret.yaml` с именем пользователя и ключом, полученным ранее. [7]

```yaml
# csi-rbd-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default # Укажите тот же namespace
stringData:
  userID: kube # Имя пользователя (без 'client.')
  userKey: "AQB...long...key...==" # <-- КЛЮЧ ПОЛЬЗОВАТЕЛЯ
```

Примените эти манифесты:
```bash
kubectl apply -f csi-config-map.yaml
kubectl apply -f csi-rbd-secret.yaml
```

### 3. Установка CSI-драйвера

Самый простой способ установить драйвер — использовать Helm или официальные YAML-манифесты от разработчиков `ceph-csi`. [6]

Клонируем репозиторий `ceph-csi`:
```bash
git clone https://github.com/ceph/ceph-csi.git
cd ceph-csi
```

Далее нужно отредактировать файлы установки, чтобы они использовали наши `ConfigMap` и `Secret`. В файлах, таких как `deploy/rbd/kubernetes/csi-rbd-provisioner.yaml` и `deploy/rbd/kubernetes/csi-rbd-node.yaml`, найдите ссылки на `ceph-conf` и `ceph-keyring` и убедитесь, что они соответствуют созданным вами ресурсам.

Проще всего установить драйвер, применив все необходимые манифесты из директории `deploy/rbd/kubernetes`:
```bash
# Применяем CRD (Custom Resource Definitions) для снэпшотов
kubectl apply -f deploy/rbd/kubernetes/csi-snapshotter-crds.yaml

# Применяем RBAC (роли и привязки)
kubectl apply -f deploy/rbd/kubernetes/csi-rbd-rbac.yaml

# Устанавливаем сам драйвер
kubectl apply -f deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
kubectl apply -f deploy/rbd/kubernetes/csi-rbdplugin.yaml
```

После применения манифестов проверьте, что все поды драйвера успешно запустились в указанном вами namespace (например, `default` или `kube-system`). [2]
```bash
kubectl get pods -l "app=csi-rbdplugin"
kubectl get pods -l "app=csi-rbdplugin-provisioner"
```
Вы должны увидеть несколько запущенных подов `csi-rbdplugin-provisioner` и по одному поду `csi-rbdplugin` на каждом рабочем узле вашего кластера. [7]