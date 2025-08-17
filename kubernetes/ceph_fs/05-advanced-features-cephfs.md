# 5. Расширенные возможности: Снэпшоты и расширение томов CephFS

CSI-драйвер для CephFS также поддерживает важные функции для управления данными, такие как расширение томов и создание моментальных снимков.

## Расширение томов (Volume Expansion)

Эта функция позволяет увеличить квоту, выделенную для `subvolume`, без прерывания работы приложения.

**Задача:** Вашему приложению потребовалось больше места, чем изначально выделенные 10Gi.

**Предварительное условие:** В `StorageClass` должен быть установлен параметр `allowVolumeExpansion: true`.

### Как это сделать:

1.  **Отредактируйте PVC:**
    ```bash
    kubectl edit pvc shared-files-pvc
    ```

2.  **Измените размер:**
    Найдите поле `spec.resources.requests.storage` и увеличьте значение, например, с `10Gi` до `20Gi`. Сохраните изменения.

3.  **Проверка:**
    CSI-контроллер обнаружит изменение и выполнит команду `ceph fs subvolume resize`, чтобы увеличить квоту.
    *   **Проверьте события PVC:**
        ```bash
        kubectl describe pvc shared-files-pvc
        ```
        В событиях вы можете увидеть процесс расширения.
    *   **Проверьте квоту в Ceph:**
        ```bash
        # Имя subvolume можно взять из `volumeHandle` связанного PV
        ceph fs subvolume info my-cephfs <subvolume-name>
        ```
        Вы увидите, что `max_bytes` для `subvolume` увеличился до ~20GiB.

## Моментальные снимки (Volume Snapshots)

Снэпшот `subvolume` в CephFS — это мгновенный снимок состояния всех файлов и директорий внутри него.

**Задача:** Создать резервную копию общего хранилища веб-приложения перед обновлением.

### 1. Создание `VolumeSnapshotClass`

```yaml
# cephfs-snapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-cephfs-snapclass
driver: cephfs.csi.ceph.com # Указываем CSI-драйвер для CephFS
deletionPolicy: Delete # Удалять снэпшот в Ceph при удалении объекта VolumeSnapshot
parameters:
  clusterID: b9127830-b0cc-4e34-aa47-9d1a2e9949a8 # FSID кластера

  # Секрет с admin ключом для создания снэпшотов
  csi.storage.k8s.io/snapshotter-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/snapshotter-secret-namespace: default
```
Примените: `kubectl apply -f cephfs-snapshotclass.yaml`.

### 2. Создание `VolumeSnapshot`

Создадим снэпшот с нашего `shared-files-pvc`.

```yaml
# cephfs-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: shared-files-snapshot
spec:
  volumeSnapshotClassName: csi-cephfs-snapclass
  source:
    persistentVolumeClaimName: shared-files-pvc
```
Примените: `kubectl apply -f cephfs-snapshot.yaml`.

Проверьте статус:
```bash
kubectl get volumesnapshot
```
Когда `READYTOUSE` станет `true`, снэпшот создан.

### 3. Восстановление из `VolumeSnapshot`

Восстановление происходит путем создания нового PVC из снэпшота.

```yaml
# pvc-from-cephfs-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-shared-pvc
spec:
  accessModes:
    - ReadWriteMany # Режимы доступа должны совпадать
  storageClassName: cephfs-sc
  resources:
    requests:
      storage: 20Gi # Размер должен быть не меньше исходного
  dataSource:
    name: shared-files-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```
Новый PVC `restored-shared-pvc` будет содержать точную копию данных из `shared-files-pvc` на момент создания снэпшота. Его можно подключить к новому `Deployment` для проверки или восстановления.