### Методичка по ДЗ: Модуль 1, Тема 4

**Название ДЗ:** Использование постоянного хранилища

**Цель ДЗ:** Научиться использовать PersistentVolumeClaims (PVC) для запроса постоянного хранилища и подключать его к подам для сохранения данных.

**Необходимые условия/Инструменты:**

1.  Работающий кластер Kubernetes с установленным `kubectl`.
2.  **Важно:** В кластере должен быть настроен StorageClass или быть доступен статический PersistentVolume (PV). Без этого динамическое выделение хранилища через PVC не сработает.
    *   **Для локальных кластеров:**
        *   **Minikube:** Имеет встроенный `standard` StorageClass, который работает "из коробки".
        *   **Kind:** Требует установки провайдера локального хранилища, например `local-path-provisioner`. Или вы можете вручную создать `hostPath` PV и StorageClass. Для простоты ДЗ можно использовать `hostPath` PV.
        *   **Docker Desktop (Kubernetes):** Имеет встроенный StorageClass.
    *   **Для облачных кластеров:** У облачных провайдеров всегда есть настроенные StorageClass (например, `gp2`/`gp3` в AWS, `standard` в GCP/Azure).
3.  Текстовый редактор.

**Подробные шаги:**

**Шаг 1: Создайте директорию для манифестов**

```bash
mkdir k8s-pv-app
cd k8s-pv-app
```

**Шаг 2: Напишите манифест PersistentVolumeClaim (PVC)**

PVC - это запрос на хранилище. Kubernetes найдет подходящий PV или Provisioner (через StorageClass) для удовлетворения этого запроса.

Создайте файл `my-pvc.yaml`.

```yaml
# my-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data-claim # Имя PVC
spec:
  accessModes:
    - ReadWriteOnce # Режим доступа: может быть смонтирован как ReadWrite только одной нодой
  resources:
    requests:
      storage: 1Gi # Запрашиваемый размер хранилища (1 Гигабайт)
  # storageClassName: standard # Опционально: указать конкретный StorageClass, если их несколько.
                               # Если не указан, используется StorageClass по умолчанию, если он есть.
```

*   **Пояснения:**
    *   `spec.accessModes`: Указывает режимы доступа. `ReadWriteOnce` - самый распространенный режим для большинства файловых систем. Другие: `ReadOnlyMany`, `ReadWriteMany`.
    *   `spec.resources.requests.storage`: Запрашиваемый объем хранилища.

**Шаг 3: Напишите манифест Deployment, использующий PVC**

Теперь создадим Deployment, который смонтирует запрошенное хранилище. Используем образ Nginx, который будет писать логи в файл на монтированном Volume.

Создайте файл `app-with-pv.yaml`.

```yaml
# app-with-pv.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-log-writer-app
spec:
  replicas: 1 # Для простоты используем одну реплику
  selector:
    matchLabels:
      app: log-writer
  template:
    metadata:
      labels:
        app: log-writer
    spec:
      containers:
      - name: writer-container
        image: busybox:latest # Используем busybox для записи файла
        command: ["/bin/sh", "-c"]
        args: ["while true; do echo $(date -u) 'Hello from pod' >> /app-data/log.txt; sleep 5; done"] # Пишем в файл каждые 5 секунд
        volumeMounts:
        - name: data-volume # Имя VolumeMount, должно соответствовать имени Volume ниже
          mountPath: /app-data # Путь внутри контейнера, куда монтируется Volume
      volumes:
      - name: data-volume # Имя Volume
        persistentVolumeClaim:
          claimName: my-data-claim # Ссылка на имя вашего PVC
```

*   **Пояснения:**
    *   `spec.template.spec.volumes`: Определяет Volume, который будет доступен подам.
    *   `name: data-volume`: Имя Volume, которое используется в `volumeMounts`.
    *   `persistentVolumeClaim.claimName`: Ссылка на PVC, который мы хотим использовать.
    *   `spec.template.spec.containers.volumeMounts`: Определяет, куда Volume монтируется внутри контейнера.
    *   Используем образ `busybox` с командой, которая бесконечно пишет строки в файл `/app-data/log.txt`. `/app-data` - это наша точка монтирования.

**Шаг 4: Примените манифесты к кластеру**

```bash
kubectl apply -f my-pvc.yaml
kubectl apply -f app-with-pv.yaml
```

**Шаг 5: Проверьте состояние PVC и Deployment**

Убедитесь, что PVC находится в состоянии `Bound` (связан с PV) и под Deployment запущен.

```bash
kubectl get pvc my-data-claim
kubectl get deploy my-log-writer-app
kubectl get pods -l app=log-writer
```

*   **Ожидаемый вывод:**
    *   `kubectl get pvc`: `STATUS` для `my-data-claim` должен быть `Bound`. В колонке `VOLUME` будет указано имя связанного PV.
    *   `kubectl get deploy`: Должен показать `AVAILABLE` равным `1`.
    *   `kubectl get pods`: Должен показать под со статусом `Running`.

**Шаг 6: Проверьте запись данных в смонтированный Volume**

С помощью `kubectl exec` выполните команду внутри пода, чтобы посмотреть содержимое файла, который пишется в смонтированный Volume.

Найдите имя пода: `kubectl get pods -l app=log-writer`

Прочитайте файл:

```bash
# Замените <log-writer-pod-name> на имя пода
kubectl exec <log-writer-pod-name> -- cat /app-data/log.txt
```

*   **Ожидаемый результат:** Вы должны увидеть строки, которые пишет busybox (даты и "Hello from pod").

**Шаг 7: Продемонстрируйте сохранение данных после удаления пода**

Удалите текущий под Deployment. Deployment автоматически создаст новый под.

```bash
# Замените <log-writer-pod-name> на имя текущего пода
kubectl delete pod <log-writer-pod-name>
```

Дождитесь, пока новый под поднимется. Проверьте это с помощью `kubectl get pods -l app=log-writer`. У него будет другое имя.

Теперь снова прочитайте содержимое файла на **новом** поде.

```bash
# Замените <new-log-writer-pod-name> на имя нового пода
kubectl exec <new-log-writer-pod-name> -- cat /app-data/log.txt
```

*   **Ожидаемый результат:** Вы должны увидеть не только новые строки, но и те строки, которые были записаны первым подом до его удаления. Это подтверждает, что данные сохранились на постоянном хранилище, привязанном к PVC, и новый под успешно смонтировал тот же Volume.

**Проверка результатов (Измеряемость):**

1.  Предоставьте YAML-файлы: `my-pvc.yaml`, `app-with-pv.yaml`.
2.  Предоставьте вывод команды `kubectl get pvc my-data-claim`, показывающий статус `Bound`.
3.  Предоставьте вывод команды `kubectl get pods -l app=log-writer` после того, как новый под поднялся.
4.  Предоставьте вывод команды `kubectl exec <new-pod-name> -- cat /app-data/log.txt`, показывающий содержимое файла, включающее записи от предыдущего пода.

---

Продолжайте, когда будете готовы перейти к следующему ДЗ из Модуля 1 (Тема 7).