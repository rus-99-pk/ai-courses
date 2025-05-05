*   **Урок 9: Kubernetes: Оркестрация Контейнеров и GitOps**
    *   **Цель:** Освоить Kubernetes как основную платформу для оркестрации контейнеров и познакомиться с принципами GitOps.
    *   **Ключевые Темы:**
        *   Оркестрация контейнеров: Зачем нужна, Kubernetes vs Docker Swarm vs Mesos.
        *   Архитектура Kubernetes: Master/Control Plane (API Server, Scheduler, Controller Manager, etcd) и Worker Nodes (Kubelet, Kube-proxy, Container Runtime).
        *   Запуск K8s кластера (варианты): Minikube/Kind (локально), kops/kubeadm (self-hosted), Managed K8s (GKE, EKS, AKS).
        *   kubectl: Основные команды (`get`, `describe`, `apply`, `delete`, `exec`, `logs`).
        *   Основные сущности K8s: Pods, Deployments, Services (ClusterIP, NodePort, LoadBalancer), Namespaces, ConfigMaps, Secrets.
        *   Продвинутые сущности: DaemonSets, StatefulSets, Jobs, CronJobs, Ingress.
        *   Persistent Storage в K8s: PersistentVolumes (PV), PersistentVolumeClaims (PVC), StorageClasses.
        *   Стратегии деплоя в K8s: RollingUpdate (по умолчанию), Recreate, Blue/Green (с Services/Ingress), Canary (с Ingress/Service Mesh).
        *   Twelve-Factor App и Kubernetes.
        *   Пакетные менеджеры для K8s: Helm (Charts, Releases, Values, Templates).
        *   GitOps: Принципы, сравнение Push vs Pull моделей.
        *   Инструменты GitOps: Argo CD (или Flux).
        *   Работа с облачными сервисами (обзор): IaaS, PaaS, SaaS. Популярные облачные провайдеры и их K8s сервисы.
    *   **Практические Задачи (Hands-on Labs):**
        1.  **Minikube/Kind Setup:** Развернуть локальный Kubernetes кластер с помощью Minikube или Kind.
        2.  **kubectl Practice:** Изучить основные команды `kubectl`. Получить информацию о кластере, нодах, неймспейсах.
        3.  **Deploy Pods/Deployments:** Написать YAML-манифесты для деплоя вашего Docker-контейнеризованного приложения в K8s в виде Deployment. Применить манифест (`kubectl apply`). Проверить статус Pods, Logs.
        4.  **Expose with Services:** Создать Service (например, NodePort), чтобы получить доступ к приложению извне кластера.
        5.  **ConfigMaps and Secrets:** Перенести конфигурацию приложения и секреты (из Vault или напрямую) в K8s ConfigMaps и Secrets. Изменить Deployment, чтобы использовать их.
        6.  **Persistent Storage:** Настроить Persistent Volume Claim для БД или файлового хранилища, если они работают в K8s.
        7.  **Helm Chart:** Создать Helm Chart для вашего приложения. Параметризовать его (настройки приложения, ресурсы). Развернуть приложение с помощью Helm (`helm install`). Обновить Chart и сделать апгрейд релиза (`helm upgrade`). Удалить релиз (`helm uninstall`).
        8.  **GitOps with Argo CD:** Развернуть Argo CD в Minikube. Создать новый Git-репозиторий для K8s-манифестов/Helm-чартов. Сделать синхронизацию Argo CD с этим репозиторием, чтобы Argo CD автоматически деплоил ваше приложение в кластер при изменении манифестов в Git.
        9.  **Update Pipeline (GitOps):** Изменить CI/CD Pipeline:
            *   После сборки и публикации Docker образа, обновить тег образа в YAML-манифесте или Values-файле Helm Chart'а.
            *   Закоммитить и отправить изменения в репозиторий с K8s-манифестами, который отслеживает Argo CD.
            *   Проверить, как Argo CD автоматически подхватывает изменение и обновляет приложение в K8s.
    *   **Результат:** Умение деплоить и управлять контейнеризованными приложениями в Kubernetes с помощью YAML и Helm. Понимание и базовый опыт работы с GitOps инструментами для автоматической поставки приложений в K8s.