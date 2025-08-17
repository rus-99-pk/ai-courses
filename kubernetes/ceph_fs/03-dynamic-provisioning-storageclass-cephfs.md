# 3. Динамическое выделение томов CephFS с помощью StorageClass

`StorageClass` для CephFS определяет, как CSI-драйвер будет создавать `subvolumes` (подкаталоги), которые затем будут использоваться подами в качестве `PersistentVolume`.

## Разбор манифеста StorageClass для CephFS

Создадим файл `cephfs-storageclass.yaml`.

```yaml
# cephfs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs-sc
provisioner: cephfs.csi.ceph.com # <-- Имя прови저нера для CephFS
parameters:
  clusterID: b9127830-b0cc-4e34-aa47-9d1a2e9949a8 # <-- FSID вашего Ceph кластера
  fsName: my-cephfs # <-- Имя файловой системы CephFS

  # Пул, в котором будут создаваться метаданные для subvolumes
  # Обычно это пул данных вашей CephFS
  pool: cephfs_data

  # Секреты для доступа к Ceph
  csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default

reclaimPolicy: Delete # 'Delete' удаляет subvolume при удалении PVC. 'Retain' оставляет его.
allowVolumeExpansion: true # Разрешить расширение томов (увеличение квоты)
mountOptions:
  # Дополнительные опции монтирования, если нужны
  # - mds_namespace=my-cephfs
```

### Ключевые параметры:

*   `provisioner`: Всегда `cephfs.csi.ceph.com` для драйвера CephFS CSI. [1]
*   `fsName`: Имя вашей файловой системы CephFS, которую вы создали ранее (`my-cephfs`). [6]
*   `pool`: Имя пула данных. CSI-драйверу необходимо знать, в каком пуле хранить метаданные для `subvolumes`. [4]
*   `csi.storage.k8s.io/...-secret-name/namespace`: Указывают на `Secret`, содержащий `adminKey` (для создания `subvolumes`) и `userKey` (для монтирования). [1]
*   `reclaimPolicy: Delete`: При удалении `PersistentVolumeClaim` (PVC) соответствующий `subvolume` (директория) в CephFS будет удален вместе со всем содержимым. Используйте `Retain` для критически важных данных.
*   `allowVolumeExpansion: true`: Позволяет увеличивать квоту на `subvolume` после его создания. [4]

Примените манифест:
```bash
kubectl apply -f cephfs-storageclass.yaml
```

Проверьте создание:
```bash
kubectl get sc
```

## Лучшие и плохие практики

### Хорошие практики:

*   **Используйте квоты**: Хотя CephFS представляет собой единое пространство, для каждого `subvolume` можно (и нужно) устанавливать квоту, чтобы один PVC не мог занять все место. Размер, указанный в PVC (`spec.resources.requests.storage`), автоматически превращается в квоту на `subvolume`. [4]
*   **Изоляция через `subvolume` группы**: Для логического разделения (например, по проектам) используйте `subvolume` группы в Ceph. Это упрощает управление снэпшотами и квотами на уровне группы.
*   **Отказоустойчивость MDS**: Всегда запускайте как минимум два MDS-сервера для одной файловой системы в режиме `active/standby`, чтобы избежать единой точки отказа.
*   **Используйте `reclaimPolicy: Retain` для продуктивных данных**, чтобы случайное `kubectl delete pvc` не привело к потере данных.

### Плохие практики:

*   **Не использовать `fsName`**: Если не указать имя файловой системы, драйвер не будет знать, где создавать `subvolumes`.
*   **Неправильные ключи в Secret**: Использование ключа обычного пользователя (`userID`/`userKey`) в качестве `adminID`/`adminKey` приведет к ошибкам прав доступа при попытке создать `subvolume`.
*   **Разрешать безлимитное использование**: Не указывать размер в PVC — плохая практика, так как это не устанавливает квоту, и одно приложение может исчерпать все доступное место в CephFS.