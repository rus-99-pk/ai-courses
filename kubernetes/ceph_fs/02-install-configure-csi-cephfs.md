# 2. Подготовка Ceph и Установка Драйвера CephFS CSI

Процесс установки драйвера для CephFS похож на установку для RBD, но требует специфичной настройки на стороне Ceph-кластера.

## Шаг 1: Настройка на стороне Ceph-кластера

Предполагается, что у вас есть работающий Ceph-кластер с запущенными сервисами мониторов (MON) и менеджеров (MGR).

### 1. Создание файловой системы CephFS

Если у вас еще нет файловой системы, ее необходимо создать. Это требует как минимум одного активного Metadata Server (MDS).

```bash
# Создаем два пула: один для данных, другой для метаданных
ceph osd pool create cephfs_data
ceph osd pool create cephfs_metadata

# Создаем саму файловую систему
ceph fs new my-cephfs cephfs_metadata cephfs_data
```

Проверьте статус файловой системы:
```bash
ceph fs status
```
Вы должны увидеть, что MDS находятся в состоянии `active`.

### 2. Создание пользователя CephX

Создадим специального пользователя для Kubernetes, который будет иметь доступ к CephFS.

```bash
# Для Ceph Nautilus и новее
ceph fs authorize my-cephfs client.kube / rw
```

Эта команда создает пользователя `client.kube`, выдает ему права на чтение и запись (`rw`) в корневой директории (`/`) файловой системы `my-cephfs` и выводит его ключ. **Сохраните этот ключ**.

Вывод будет выглядеть так:
```
[client.kube]
    key = AQD...another...long...key...==
```

## Шаг 2: Установка и настройка CephFS CSI в Kubernetes

### 1. Получение информации о кластере Ceph

Нам нужны **FSID** кластера и адреса **мониторов**.
```bash
ceph mon dump
```
Сохраните `fsid` и IP-адреса мониторов.

Также нам нужен **admin ID**. Обычно это `admin`. Получить ключ администратора можно командой `ceph auth get-key client.admin`. Этот ключ нужен CSI-драйверу для выполнения административных операций, таких как создание `subvolumes`.

### 2. Создание ConfigMap и Secret в Kubernetes

**ConfigMap**: Содержит информацию о подключении к кластеру.
```yaml
# csi-config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-csi-config
  namespace: default # Укажите namespace для установки драйвера
data:
  config.json: |
    [
      {
        "clusterID": "b9127830-b0cc-4e34-aa47-9d1a2e9949a8", # <-- ВАШ FSID
        "monitors": [
          "192.168.1.10:6789", # <-- АДРЕСА ВАШИХ МОНИТОРОВ
          "192.168.1.11:6789"
        ]
      }
    ]
```

**Secret**: Содержит ключи для пользователя `client.kube` и `client.admin`.
```yaml
# csi-cephfs-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-cephfs-secret
  namespace: default # Тот же namespace
stringData:
  # Ключ администратора для создания subvolumes
  adminID: admin
  adminKey: "AQB...admin...key...=="

  # Ключ пользователя для монтирования
  userID: kube
  userKey: "AQD...another...long...key...=="
```
**Важное замечание**: CSI-драйверу для CephFS требуется `adminKey` для создания `subvolumes` (поддиректорий). Это отличается от RBD, где можно было обойтись правами только на пул.

Примените эти манифесты:
```bash
kubectl apply -f csi-config-map.yaml
kubectl apply -f csi-cephfs-secret.yaml
```

### 3. Установка CSI-драйвера

Используем официальные манифесты из репозитория `ceph-csi`.
```bash
git clone https://github.com/ceph/ceph-csi.git
cd ceph-csi
```

Применяем необходимые компоненты для CephFS:
```bash
# Применяем CRD (Custom Resource Definitions)
kubectl apply -f deploy/cephfs/kubernetes/csi-snapshotter-crds.yaml

# Применяем RBAC (роли и привязки)
kubectl apply -f deploy/cephfs/kubernetes/csi-cephfs-rbac.yaml

# Устанавливаем сам драйвер
kubectl apply -f deploy/cephfs/kubernetes/csi-cephfsplugin-provisioner.yaml
kubectl apply -f deploy/cephfs/kubernetes/csi-cephfsplugin.yaml
```

Проверьте, что все поды драйвера успешно запустились:
```bash
kubectl get pods -l "app=csi-cephfsplugin"
kubectl get pods -l "app=csi-cephfsplugin-provisioner"
```