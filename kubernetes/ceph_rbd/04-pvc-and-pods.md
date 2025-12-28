# 4. Работа с PersistentVolumeClaims и использование томов в Подах

После того как `StorageClass` создан, разработчики и приложения могут запрашивать хранилище с помощью `PersistentVolumeClaim` (PVC) и использовать его в своих подах.

## Запрос хранилища: PersistentVolumeClaim (PVC)

`PersistentVolumeClaim` — это запрос на хранилище со стороны пользователя. [1, 21] Он похож на запрос CPU или памяти для пода.

### Пример 1: PVC с файловой системой (Filesystem)

Это наиболее частый сценарий использования, когда приложению нужна директория для хранения файлов.

Создадим файл `rbd-pvc-fs.yaml`:```yaml
# rbd-pvc-fs.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-rbd-claim
spec:
  accessModes:
    - ReadWriteOnce # Том может быть смонтирован для чтения/записи одним узлом
  volumeMode: Filesystem # Режим тома: файловая система (по умолчанию)
  resources:
    requests:
      storage: 5Gi # Запрашиваем 5 ГиБ
  storageClassName: ceph-rbd-sc # Указываем наш StorageClass
```

*   `accessModes`:
    *   `ReadWriteOnce` (RWO): Том может быть смонтирован в режиме чтения-записи только одним узлом (но может использоваться несколькими подами на этом узле). Для Ceph RBD с `volumeMode: Filesystem` это основной режим доступа. [20]
*   `volumeMode: Filesystem`: Указывает, что на блочном устройстве будет создана файловая система (та, что указана в `StorageClass`, например, `ext4`), и в под будет смонтирована именно она. [1]
*   `resources.requests.storage`: Размер запрашиваемого тома.

Создайте PVC:
```bash
kubectl apply -f rbd-pvc-fs.yaml
```

Проверьте статус PVC:
```bash
kubectl get pvc my-rbd-claim
```
Сначала статус будет `Pending`. Через несколько секунд CSI-драйвер создаст RBD-образ и PV, и статус изменится на `Bound` (Связан). [2]

### Пример 2: PVC с блочным устройством (Block)

Некоторым приложениям, например, базам данных, может быть полезнее получить "сырое" блочное устройство без файловой системы для лучшей производительности.

Создадим файл `rbd-pvc-block.yaml`:
```yaml
# rbd-pvc-block.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-raw-block-claim
spec:
  accessModes:
    - ReadWriteMany # Том может быть смонтирован для чтения/записи несколькими узлами
  volumeMode: Block # Режим тома: блочное устройство
  resources:
    requests:
      storage: 10Gi
  storageClassName: ceph-rbd-sc
```
*   `accessModes`:
    *   В режиме `Block` Ceph RBD поддерживает `ReadWriteMany` (RWX), что позволяет подключать одно и то же блочное устройство к нескольким подам на разных узлах. [20] Это продвинутый сценарий, требующий, чтобы само приложение умело работать с общим блочным устройством (например, было кластеризованным).
*   `volumeMode: Block`: Указывает, что под получит не файловую систему, а само устройство (например, `/dev/sdb`). [1]

## Использование PVC в Подах

Чтобы под мог использовать том, PVC монтируется в его файловую систему (для `volumeMode: Filesystem`) или передается как блочное устройство (для `volumeMode: Block`).

### Пример 1: Монтирование файловой системы

Создадим простой Nginx-под, который будет использовать наш PVC `my-rbd-claim`.

```yaml
# pod-fs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  containers:
    - name: web-server
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - name: my-storage
          mountPath: /usr/share/nginx/html # Монтируем том в директорию Nginx
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: my-rbd-claim # Ссылаемся на наш PVC
```
Здесь мы определяем том `my-storage`, который ссылается на PVC `my-rbd-claim`, и затем монтируем его в контейнер с помощью `volumeMounts`.

### Пример 2: Использование блочного устройства

```yaml
# pod-block.yaml
apiVersion: v1
kind: Pod
metadata:
  name: block-app
spec:
  containers:
    - name: block-app
      image: busybox
      command: ["/bin/sh", "-c", "sleep 3600"]
      volumeDevices:
        - name: my-raw-block-storage
          devicePath: /dev/xvd # Устройство появится внутри пода по этому пути
  volumes:
    - name: my-raw-block-storage
      persistentVolumeClaim:
        claimName: my-raw-block-claim # Ссылаемся на наш блочный PVC
```
Вместо `volumeMounts` здесь используется `volumeDevices`, которое передает блочное устройство напрямую в контейнер. [13]

### Команды-методички

*   **Посмотреть все PVC:**
    ```bash
    kubectl get pvc
    ```
*   **Посмотреть детали PVC и связанного с ним PV:**
    ```bash
    kubectl describe pvc <pvc-name>
    ```
*   **Посмотреть, какой RBD-образ соответствует PV (в выводе `describe pv`):**
    ```bash
    kubectl describe pv <pv-name>
    ```
    В секции `Source` -> `CSI` вы увидите `volumeHandle`, который обычно содержит имя RBD-образа.
*   **Зайти в под и проверить смонтированный том:**
    ```bash
    kubectl exec -it web-server -- /bin/bash
    df -h # Показать смонтированные файловые системы
    ```