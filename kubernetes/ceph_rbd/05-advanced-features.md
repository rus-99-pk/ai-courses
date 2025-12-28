# 5. Расширенные возможности: Снэпшоты, клонирование и расширение томов

Ceph RBD и CSI-драйвер предоставляют мощные инструменты для управления жизненным циклом данных, выходящие за рамки простого создания и подключения томов.

## Расширение томов (Volume Expansion)

Это возможность увеличить размер `PersistentVolumeClaim` "на лету", без остановки приложения. [24]

**Задача:** Ваше приложение заполнило выделенные 5Gi дискового пространства, и вам нужно срочно его увеличить.

**Предварительное условие:** `StorageClass`, который использовался для создания тома, должен содержать параметр `allowVolumeExpansion: true`. [1, 17]

### Как это сделать:

1.  **Отредактируйте PVC:**
    Самый простой способ — использовать команду `kubectl edit`.

    ```bash
    kubectl edit pvc my-rbd-claim
    ```

2.  **Измените размер:**
    В открывшемся редакторе найдите секцию `spec.resources.requests.storage` и измените значение, например, с `5Gi` на `10Gi`. Сохраните и закройте файл.

3.  **Проверьте результат:**
    Kubernetes и CSI-драйвер начнут процесс расширения.

    *   **Проверьте статус PVC:**
        ```bash
        kubectl describe pvc my-rbd-claim
        ```
        В секции `Conditions` вы можете увидеть статус `FileSystemResizePending`. [17]

    *   **Проверьте размер внутри пода:**
        После завершения процесса, который обычно занимает несколько секунд, вы можете проверить новый размер прямо внутри контейнера.
        ```bash
        kubectl exec -it web-server -- df -h /usr/share/nginx/html
        ```
        Вы должны увидеть, что размер файловой системы увеличился до ~10Gi. [24]

## Моментальные снимки (Volume Snapshots)

Снэпшот — это "фотография" состояния вашего тома в определенный момент времени. [23] Это невероятно полезно для резервного копирования и восстановления. [18]

**Задача:** Создать резервную копию базы данных перед крупным обновлением.

**Предварительные условия:**

1.  В кластере должны быть установлены `CustomResourceDefinitions` (CRD) для снэпшотов (`volumesnapshots.snapshot.storage.k8s.io` и др.). Обычно они устанавливаются вместе с CSI-драйвером. [27]
2.  В кластере должен быть запущен `snapshot-controller`. [25]
3.  Необходимо создать `VolumeSnapshotClass`.

### 1. Создание `VolumeSnapshotClass`

Это аналог `StorageClass`, но для снэпшотов. Он указывает, какой CSI-драйвер должен обрабатывать создание снэпшотов.

```yaml
# rbd-snapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbd-snapclass
driver: rbd.csi.ceph.com # Указываем наш CSI-драйвер
deletionPolicy: Delete # 'Delete' - удалять снэпшот в Ceph при удалении объекта VolumeSnapshot. 'Retain' - оставлять.
parameters:
  # clusterID должен совпадать с ID кластера в StorageClass
  clusterID: b9127830-b0cc-4e34-aa47-9d1a2e9949a8

  # Секреты для доступа к Ceph
  csi.storage.k8s.io/snapshotter-secret-name: csi-rbd-secret
  csi.storage.k8s.io/snapshotter-secret-namespace: default
```
Примените его: `kubectl apply -f rbd-snapshotclass.yaml`.

### 2. Создание `VolumeSnapshot`

Теперь можно сделать снэпшот с существующего PVC (`my-rbd-claim`).

```yaml
# rbd-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-rbd-claim-snapshot
spec:
  volumeSnapshotClassName: csi-rbd-snapclass
  source:
    persistentVolumeClaimName: my-rbd-claim```
Примените: `kubectl apply -f rbd-snapshot.yaml`.

Проверить статус снэпшота:
```bash
kubectl get volumesnapshot
# или подробнее
kubectl describe volumesnapshot my-rbd-claim-snapshot
```
Когда `READYTOUSE` станет `true`, снэпшот готов.

### 3. Восстановление из `VolumeSnapshot`

Для восстановления данных мы создаем **новый** PVC, указывая снэпшот в качестве источника данных. [18]

```yaml
# pvc-from-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-restored-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-rbd-sc
  resources:
    requests:
      storage: 10Gi # Размер должен быть не меньше, чем у исходного тома
  dataSource:
    name: my-rbd-claim-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```
После создания этого PVC, он будет содержать точную копию данных из `my-rbd-claim` на момент создания снэпшота.

## Клонирование томов (Volume Cloning)

Клонирование — это создание нового PVC из уже существующего PVC, без промежуточного создания снэпшота.

**Задача:** Быстро создать тестовое окружение с копией продуктивных данных.

```yaml
# pvc-clone.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-cloned-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-rbd-sc
  resources:
    requests:
      storage: 5Gi # Размер должен быть равен исходному
  dataSource:
    name: my-rbd-claim # Имя исходного PVC
    kind: PersistentVolumeClaim
```
Это создаст новый PVC `my-cloned-pvc`, который будет точной копией `my-rbd-claim`. Под капотом Ceph использует ту же технологию `snapshot` + `clone`, но Kubernetes абстрагирует это для пользователя.