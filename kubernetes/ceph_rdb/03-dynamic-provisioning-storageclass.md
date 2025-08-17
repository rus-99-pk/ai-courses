# 3. Динамическое выделение томов с помощью StorageClass

`StorageClass` — это объект Kubernetes, который позволяет администраторам определять "классы" хранилищ. [9] Пользователи могут запрашивать тома, ссылаясь на `StorageClass`, а Kubernetes, через CSI-драйвер, автоматически создает (`provision`) соответствующий том. [16]

## Разбор манифеста StorageClass для Ceph RBD

Создадим файл `rbd-storageclass.yaml`.

```yaml
# rbd-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-sc
provisioner: rbd.csi.ceph.com # <-- Имя прови저нера CSI-драйвера
parameters:
  clusterID: b9127830-b0cc-4e34-aa47-9d1a2e9949a8 # <-- FSID вашего Ceph кластера
  pool: k8s-rbd # <-- Имя пула в Ceph, созданного ранее

  # RBD image features. Доступные: layering, journaling, exclusive-lock, object-map, fast-diff
  # 'layering' необходим для создания клонов
  imageFeatures: layering

  # Секреты для доступа к Ceph, которые мы создали
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default

  # Файловая система по умолчанию для томов
  csi.storage.k8s.io/fstype: ext4

reclaimPolicy: Delete # Политика возврата. 'Delete' - удалять том в Ceph при удалении PVC. 'Retain' - оставлять.
allowVolumeExpansion: true # Разрешить расширение томов "на лету"
mountOptions:
  - discard
```

### Ключевые параметры:

*   `provisioner`: Всегда `rbd.csi.ceph.com` для RBD-драйвера Ceph CSI. [1]
*   `clusterID`: FSID вашего Ceph-кластера. Должен совпадать с тем, что указан в `ConfigMap`. [1]
*   `pool`: Имя пула, в котором CSI-драйвер будет создавать RBD-образы. [1, 3]
*   `imageFeatures`: Возможности, которые будут включены для создаваемых RBD-образов. `layering` является обязательным для поддержки клонирования томов (создания PVC из другого PVC). [19]
*   `csi.storage.k8s.io/...-secret-name/namespace`: Указывают на `Secret`, содержащий ключ пользователя `client.kube`. [1] Эти секреты используются разными компонентами CSI (прови저нером, контроллером расширения, плагином на узле) для взаимодействия с Ceph.
*   `reclaimPolicy`: Определяет, что произойдет с нижележащим RBD-образом, когда `PersistentVolumeClaim` (PVC), который его использует, будет удален.
    *   `Delete`: RBD-образ будет удален из Ceph. Это поведение по умолчанию и наиболее распространенное для динамически создаваемых томов.
    *   `Retain`: RBD-образ останется в Ceph. Администратору придется удалять его вручную. Полезно для критически важных данных, чтобы избежать случайного удаления.
*   `allowVolumeExpansion`: Установка в `true` разрешает пользователям увеличивать размер своих томов после их создания. [1, 5]
*   `mountOptions`: Опции, которые будут использоваться при монтировании файловой системы на узле. `discard` включает поддержку TRIM/DISCARD.

Примените манифест для создания `StorageClass`:
```bash
kubectl apply -f rbd-storageclass.yaml
```

Проверьте, что класс хранилища создан:
```bash
kubectl get sc
```
Вы должны увидеть `ceph-rbd-sc` в списке.

## Лучшие и плохие практики

### Хорошие практики:

*   **Создавайте несколько `StorageClass`**: Для разных нужд создавайте разные классы. Например, `sc-ssd` для пула на SSD-дисках и `sc-hdd` для пула на HDD. [1] Это позволяет приложениям запрашивать хранилище с нужным уровнем производительности.
*   **Используйте `reclaimPolicy: Delete` для временных или легко восстанавливаемых данных**, чтобы избежать накопления "мусорных" томов в Ceph.
*   **Используйте `reclaimPolicy: Retain` для продуктивных баз данных**, чтобы защититься от случайного удаления PVC.
*   **Всегда включайте `allowVolumeExpansion: true`**: Эта опция дает гибкость и редко имеет негативные последствия. [9]

### Плохие практики:

*   **Использовать один `StorageClass` для всех нужд**: Это лишает вас гибкости в управлении производительностью и стоимостью хранения.
*   **Не указывать `pool`**: Драйвер может попытаться использовать пул по умолчанию (`rbd`), что может быть нежелательно. [20]
*   **Использовать `client.admin` в секретах**: Это серьезная угроза безопасности. Всегда создавайте выделенного пользователя с минимально необходимыми правами.