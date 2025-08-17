# 4. Работа с PVC (RWX) и использование в Подах

Главное преимущество CephFS — это возможность совместного доступа к данным несколькими подами одновременно. Это достигается с помощью режима доступа `ReadWriteMany` (RWX).

## Запрос хранилища: PVC с `ReadWriteMany`

Создадим `PersistentVolumeClaim` (PVC), который запрашивает хранилище CephFS.

```yaml
# cephfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-files-pvc
spec:
  accessModes:
    - ReadWriteMany # <-- Ключевой режим для совместного доступа
  resources:
    requests:
      storage: 10Gi # Запрошенный размер станет квотой на subvolume
  storageClassName: cephfs-sc # Указываем наш StorageClass для CephFS
```

*   `accessModes: ReadWriteMany` (RWX): Означает, что этот том может быть смонтирован для чтения и записи несколькими узлами (и, соответственно, подами на этих узлах) одновременно. [2]

Создайте PVC:
```bash
kubectl apply -f cephfs-pvc.yaml
```

Проверьте его статус. Он должен быстро перейти в `Bound`:
```bash
kubectl get pvc shared-files-pvc
```

## Использование общего тома в `Deployment`

Теперь создадим `Deployment` с несколькими репликами (например, 2), где каждый под будет монтировать один и тот же PVC.

**Задача:** Создать два пода веб-сервера, которые совместно используют одну директорию для записи логов или контента.

```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: writer-container
        image: busybox
        # Эта команда будет каждые 5 секунд писать имя пода в общий файл
        command: ["/bin/sh", "-c"]
        args:
        - >
          while true; do
            echo "Hello from $(hostname)" >> /mnt/shared/data.txt;
            sleep 5;
          done
        volumeMounts:
        - name: shared-storage
          mountPath: /mnt/shared # Монтируем общий том
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: shared-files-pvc # Ссылаемся на наш RWX PVC
```

Примените манифест:
```bash
kubectl apply -f web-deployment.yaml
```

### Проверка совместной работы

1.  **Убедитесь, что оба пода запущены:**
    ```bash
    kubectl get pods -l app=web-server
    ```

2.  **Зайдите в один из подов и посмотрите содержимое файла:**
    ```bash
    # Получаем имя первого пода
    POD1_NAME=$(kubectl get pods -l app=web-server -o jsonpath='{.items.metadata.name}')

    # Читаем файл из первого пода
    kubectl exec $POD1_NAME -- tail -f /mnt/shared/data.txt
    ```

Вы увидите, что в файл `data.txt` пишут оба пода, так как строки будут содержать разные `hostname`. Это наглядно демонстрирует работу режима `ReadWriteMany`.

### Команды-методички

*   **Посмотреть все PVC:**
    ```bash
    kubectl get pvc
    ```
*   **Посмотреть детали PVC и связанного с ним PV:**
    ```bash
    kubectl describe pvc <pvc-name>
    ```
*   **Посмотреть, какому `subvolume` в CephFS соответствует PV:**
    ```bash
    kubectl describe pv <pv-name>
    ```
    В выводе в поле `volumeHandle` будет содержаться идентификатор `subvolume`.
*   **Проверить квоту на `subvolume` в Ceph:**
    Зайдите на узел с доступом к Ceph CLI и выполните:
    ```bash
    # Имя subvolume можно взять из 'volumeHandle'
    ceph fs subvolume getpath my-cephfs <subvolume-name>
    ceph fs subvolume info my-cephfs <subvolume-name>
    ```