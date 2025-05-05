**Модуль 1: Основы Kubernetes**

*   **Цель модуля:** Получить необходимые базовые знания о ключевых компонентах Kubernetes, их декларативном управлении и сетевых аспектах для дальнейшего применения GitOps.
*   **Тема 1: Введение в Kubernetes. Основные понятия и архитектура**
    *   **Теория:** Что такое Kubernetes, зачем он нужен. Декларативный vs Императивный подход. Архитектура кластера (Control Plane, Worker Nodes). Ключевые компоненты Control Plane (kube-apiserver, etcd, kube-scheduler, kube-controller-manager, cloud-controller-manager). Ключевые компоненты Worker Node (kubelet, kube-proxy, Container Runtime). Namespaces. kubectl - основной инструмент взаимодействия.
    *   **ДЗ:** Нет. (Эта тема вводная, фокусируемся на понимании).
*   **Тема 2: Объекты в Kubernetes**
    *   **Теория:** Подробное изучение ключевых объектов: Pods (жизненный цикл), Deployments (управление ReplicaSets, стратегии обновления - Rolling Update, Recreate), Services (типы: ClusterIP, NodePort, LoadBalancer, ExternalName). YAML-манифесты для описания объектов. Селекторы и лейблы.
    *   **ДЗ:** Развертывание простого приложения.
        *   **Задача:** Написать YAML-манифесты для:
            *   Deployment, запускающего 3 реплики простого веб-приложения (например, Nginx или `k8s.gcr.io/echoserver:1.10`).
            *   Service типа ClusterIP для доступа к этому Deployment.
            *   Service типа NodePort для внешнего доступа к этому Deployment (или LoadBalancer, если работаете в облаке).
        *   **Измеряемость:**
            1.  Предоставьте YAML-файлы манифестов.
            2.  Примените манифесты к кластеру (`kubectl apply -f <файл>`).
            3.  Предоставьте вывод команд `kubectl get pods`, `kubectl get deploy`, `kubectl get svc`.
            4.  Продемонстрируйте доступность приложения: `curl <IP:NodePort>` или доступ через LoadBalancer IP.
*   **Тема 3: Сетевая подсистема в Kubernetes**
    *   **Теория:** Модель сети Kubernetes (каждый под имеет свой IP). Взаимодействие Pod-to-Pod, Pod-to-Service. Роль kube-proxy. CNI (Container Network Interface) - концепция и примеры (Calico, Flannel). Введение в Ingress (контроллеры Ingress, Ingress-ресурсы) как способ управления внешним доступом с учетом правил маршрутизации по доменному имени/пути.
    *   **ДЗ:** Настройка сетевого взаимодействия и внешнего доступа через Ingress.
        *   **Задача:**
            *   Развернуть два различных приложения (например, frontend и backend).
            *   Настроить Service ClusterIP для backend и продемонстрировать доступ frontend к backend по имени Service.
            *   Установить Ingress-контроллер (например, Nginx Ingress Controller, если его нет в кластере).
            *   Написать Ingress-ресурс для доступа к frontend по определенному URL (например, `/app`).
        *   **Измеряемость:**
            1.  Предоставьте YAML-файлы для Deployment и Service обоих приложений, а также Ingress-ресурса.
            2.  Предоставьте вывод команд `kubectl get pods`, `kubectl get svc`, `kubectl get ingress`.
            3.  Продемонстрируйте успешный curl с пода frontend на Service backend.
            4.  Продемонстрируйте успешный внешний доступ к frontend через IP Ingress-контроллера и настроенный путь (например, `curl http://<Ingress-IP>/app`).
*   **Тема 4: Хранение данных в Kubernetes**
    *   **Теория:** Проблема эфемерности данных подов. Понятие Volume. Типы Volume (emptyDir, hostPath, анонс CSI). PersistentVolumes (PV) и PersistentVolumeClaims (PVC). StorageClasses как способ динамического выделения PV. Привязка PVC к поду. Жизненный цикл PV/PVC.
    *   **ДЗ:** Использование постоянного хранилища.
        *   **Задача:**
            *   Написать YAML-манифесты для PersistentVolumeClaim (PVC).
            *   Написать Deployment для приложения, которое сохраняет данные на Volume (например, простой веб-сервер, пишущий логи в файл, или база данных вроде SQLite).
            *   Подключить PVC к поду через VolumeMounts.
            *   Удалить под и убедиться, что новый под (созданный Deployment-ом) имеет доступ к тем же данным.
        *   **Измеряемость:**
            1.  Предоставьте YAML-файлы для PVC и Deployment.
            2.  Предоставьте вывод команд `kubectl get pvc`.
            3.  Продемонстрируйте сохранение данных: например, запишите файл из одного пода (`kubectl exec <pod-name> -- sh -c 'echo "test" > /path/to/volume/test.txt'`), затем удалите этот под (`kubectl delete pod <pod-name>`). После создания нового пода Deployment-ом, проверьте наличие файла (`kubectl exec <new-pod-name> -- cat /path/to/volume/test.txt`).
            *   *Примечание:* Для этого ДЗ потребуется настроенный StorageClass в кластере или ручное создание PV. Для простоты локально можно использовать `hostPath` PV/StorageClass.
*   **Тема 5: Requests, limits и load balancing в Kubernetes**
    *   **Теория:** Управление ресурсами (CPU, Memory) для контейнеров через `requests` и `limits`. Quality of Service (QoS) классов (Guaranteed, Burstable, BestEffort). Балансировка нагрузки в Service ClusterIP/NodePort/LoadBalancer. Введение в Horizontal Pod Autoscaler (HPA) - автоматическое масштабирование подов по метрикам (CPU, Memory, кастомные).
    *   **ДЗ:** Нет. (Теория важна для понимания надежности, но сложно измерить без инструментов мониторинга и генерации нагрузки).
*   **Тема 6: Архитектурный подход Service Mesh**
    *   **Теория:** Что такое Service Mesh (Istio, Linkerd). Зачем нужен: обнаружение сервисов, трафик-менеджмент (A/B тестирование, канареечные релизы), безопасность (mTLS), наблюдаемость (метрики, логи, трассировка). Sidecar-контейнеры. Место Service Mesh в архитектуре микросервисов.
    *   **ДЗ:** Нет. (Вводная тема, сложно реализовать в рамках базового курса).
*   **Тема 7: Развертывание кластера Kubernetes**
    *   **Теория:** Обзор методов развертывания: локальные (Minikube, Kind, Docker Desktop), self-managed (kubeadm, kops, Kubespray), cloud-managed (GKE, EKS, AKS). Плюсы и минусы каждого подхода. Фокус на одном простом методе для локальной разработки/тестирования (например, Kind или Minikube).
    *   **ДЗ:** Установка локального кластера Kubernetes.
        *   **Задача:** Используя инструмент Kind или Minikube, развернуть локальный кластер Kubernetes на своей машине.
        *   **Измеряемость:**
            1.  Предоставьте шаги, которые вы выполнили для установки инструмента и развертывания кластера.
            2.  Предоставьте вывод команды `kubectl cluster-info`.
            3.  Предоставьте вывод команды `kubectl get nodes`.
            4.  Убедитесь, что вы можете взаимодействовать с кластером с помощью `kubectl`.