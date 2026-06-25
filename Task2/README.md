
# Как запустить этот шаг

## 1. Запустить minikube

```
minikube delete
minikube start
```

## 2. Установить инструменты

```
minikube addons enable metrics-server


kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --kubeconfig ~/.kube/config \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false


helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --kubeconfig ~/.kube/config \
  --values adapter-values.yaml

```

## 3. Деплой

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

kubectl apply -f servicemonitor.yaml
kubectl apply -f horizontal-pod-autoscaler-rps.yaml

kubectl port-forward service/scaletestapp-service 8080:8080

python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt

python -m locust -f locustfile.py
```
