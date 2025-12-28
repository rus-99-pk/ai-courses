## Модуль 1: Подготовка окружения и теория

### 1.1 Что такое Service Mesh и зачем он нужен?
В микросервисной архитектуре сервисов становится много. Возникают проблемы:
*   Как шифровать трафик между ними (mTLS)?
*   Как перенаправлять трафик (canary deploy, A/B тесты)?
*   Как понять, кто кого вызывает и где ошибки?

Istio решает эти проблемы, внедряя прокси-сервер (Envoy) к каждому вашему сервису. Весь трафик идет через эти прокси.

### 1.2 Установка инструментов
Убедитесь, что у вас установлены:
1.  **Docker**
2.  **Kind** (Kubernetes in Docker)
3.  **Kubectl**
4.  **Istioctl** (CLI для Istio)

### 1.3 Поднятие кластера Kind
Нам нужен кластер с пробросом портов, чтобы мы могли открывать веб-интерфейсы локально.

Создайте файл `kind-config.yaml`:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000 # Для Kiali/Grafana (упрощенный доступ)
    hostPort: 30000
    protocol: TCP
```

Запустите кластер:
```bash
kind create cluster --config kind-config.yaml --name istio-lab
kubectl cluster-info --context kind-istio-lab
```