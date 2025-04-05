Deploying Prometheus, Grafana and Loki in a Kubernetes Cluster
This projects shows the steps taken to install and configure monitoring tools. I will be deploying Prometheus for collecting metrics, Loki for logs and Grafana for visualizing the data.

Prerequisites

Create a K8s Cluster using [minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Fx86-64%2Fstable%2Fbinary+download)

Install Helm.

Step 1: Installing Prometheus

  Create a namespace called Prometheus.

```bash
kubectl create ns prometheus
```

  Add the Prometheus Helm chart repository.
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

  Update the local Helm chart repository cache.
```bash
helm repo update
```

  Install the Prometheus chart in the prometheus namespace.
```bash
helm install prometheus prometheus-community/prometheus -n prometheus
```

  Verify that Prometheus is running.
```bash
kubectl get pods -n prometheus
```

  Change the Service Type of the Prometheus-Server service from ClusterIP to NodePort.
```bash
kubectl edit svc/prometheus-server -n prometheus
```
![pro2](https://github.com/user-attachments/assets/8b8894bf-3273-4853-8680-7fa425bac035)

Step 2: Access the Prometheus server from a Web Browser

  Run the following command to get the value of NodePort:

```bash
kubectl get svc/prometheus-server -n prometheus
```
  Get the IP address of the minikube cluster.
```bash
minikube ip
```
  Go to your web browser and paste the following url:

```bash
http://<minikube_ip>:<prometheus-server-nodeport>
```
![pro4](https://github.com/user-attachments/assets/0469c6e4-c2c4-4aa9-a72f-2175e0991f8c)
![pro5](https://github.com/user-attachments/assets/657fb95f-213b-40f1-875e-aa4f1f0fe22f)

Step 3: Installing Loki

  Create a namespace called loki.
```bash
kubectl create ns loki
```

  Add the Bitnami Helm chart repository.
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

  Update the local Helm chart repository cache.
```bash
helm repo update
```
  Install Loki chart in the loki namespace.
```bash
helm install loki bitnami/grafana-loki -n loki
```
  Change the Service Type of the loki-grafana-loki-gateway service from ClusterIP to NodePort.
```bash
kubectl edit svc/loki-grafana-loki-gateway -n loki
```
![pro6](https://github.com/user-attachments/assets/29c82c4f-450b-4198-b849-c3c313a8d16e)

  Run the following command to get the value of the NodePort:
```bash
kubectl get svc/loki-grafana-loki-gateway -n loki
```
Step 4: Installing Grafana
  Create a namespace called grafana.
```bash
kubectl create ns grafana
```
  Add the Grafana Helm chart repository.
```bash
helm repo add grafana https://grafana.github.io/helm-charts
```
  Update the local Helm chart repository cache.
```bash
helm repo update
```
  Install the Grafana chart in the Grafana namespace.
```bash
helm install grafana grafana/grafana -n grafana
```
  Copy and run this following command to get the default of the password:
```bash
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base 64 --decode ; echo
```
  Change the Service Type of the grafana service from ClusterIP to NodePort.
```bash
kubectl edit svc/grafana -n grafana
```
![pro7](https://github.com/user-attachments/assets/bd4ed079-1cac-4082-b4c6-b2d385599a31)

Step 5: Access the Grafana server from a Web Browser
  Run the following command to get the value of the NodePort:
```bash
kubectl get svc -n grafana
```
  Get the IP address of the minikube cluster.
```bash
minikube ip
```
  Go to your web browser and paste the following url:
```bash
http://<minikube_ip>:<grafana-nodeport>
```
<img width="1024" alt="5 grafana url" src="https://github.com/user-attachments/assets/2d09b1cf-a0a2-408a-aacb-24c9f949c92c" />

Step 6: Configuring Prometheus and Loki as Data Sources to Grafana

  On the Grafana webpage, click on the Open menu tab then click on the Connections dropdown and Add New Connections.

  Search for the Prometheus Data Source and click on it.
![pro8](https://github.com/user-attachments/assets/71c74ca3-6ab1-400a-a087-1b9fa6cd2a25)

  Click on Add new data source.

  Paste the ```bash http://<minikube_ip>:<prometheus-server-nodeport>```into the Prometheus server URL box.
![pro9](https://github.com/user-attachments/assets/0c28ff22-b0a8-4a49-9b2a-7384c3928d91)

  Then click Save & test.

Step 7: Visualizing The K8 Cluster on Grafana using Dashboards

  On Grafana Server homepage, click on Open menu --> Dasboard --> Create Dashboard --> Import dashboard

  Go to https://grafana.com/grafana/dashboards and search for K8s and click on it.
![pro10](https://github.com/user-attachments/assets/63d5f149-8eeb-4721-a6c6-f5c9abf6609c)

  Click on Copy ID to clipboard
![pro11](https://github.com/user-attachments/assets/e44d81e0-2496-4458-852e-a0186a24f603)

  Go back to the Grafana server, paste the GUID and click on Load.
![pro12](https://github.com/user-attachments/assets/d615b773-25db-4ec1-8192-554426479cc4)

  Click on the Prometheus tab, select Prometheus as the Data source and click on Import.

Step 8: Setting Up Alerts on Prometheus

  Create a prometheus.yaml file in your directory and add an alert rule.
```bash
serverFiles:
  alerting_rules.yml:
    groups:
    - name: Pod_Status_Alerts
      rules:
      - alert: PodNotReady
        expr: count(kube_pod_status_phase{phase=~"Pending|Failed|Uknown"}) > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is not ready"
          description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} has been in Pending/Failed state for more than 2 minutes"

    - name: CPU_USAGE_ALERT
      rules:
      - alert: HighCPUUsage
        expr: 100 * (1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) > 80
        for: 5m
        labels:
          severity: critical
        annotations:
           summary: "High CPU usage detected on {{ $labels.instance }}"
           description: "CPU usage on instance {{ $labels.instance }} has been above 80% for the last 5 minutes."

    - name: Memory_Usage_Alerts
      rules:
      - alert: HighMemoryUsage
        expr: (node_memory_Active_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Memory Usage on {{ $labels.instance }}"
          description: "Memory usage on instance {{ $labels.instance }} has exceeded 90% for more than 5 minutes."
  
```
  Upgrade the Prometheus Helm release with the prometheus.yaml configuration.
```bash
helm upgrade prometheus prometheus-community/prometheus -f prometheus.yaml -n prometheus
```
Note: After upgrade the prometheus helm, nodeport of the Prometheus-Server will change

  Go your web browser and access the Prometheus-Server with the new NodePort and click on Alert.
```bash
http://<minikube_ip>:<new-prometheus-server-nodeport>
```
![pro14](https://github.com/user-attachments/assets/fec92e3d-0045-42a3-a438-741ee774fc14)

Step 9: Examining and Visualizing the etcd Pod Log Entries with Loki

  Access the Grafana server, click on Open menu --> Explore --> Select Pod and etcd-minikube as the label filters respectively then click on Run query.
![pro15](https://github.com/user-attachments/assets/f93c1967-e80b-4e44-aada-35d4fa09f439)




