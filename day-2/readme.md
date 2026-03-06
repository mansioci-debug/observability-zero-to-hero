# three pillars of observability with EFK:

| Pillar      | What it does                        | Example (EFK stack)                                                                                          |
| ----------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Logs**    | Record events from apps and systems | **Elasticsearch + Fluentd + Kibana (EFK)**: Fluentd collects logs → Elasticsearch stores → Kibana visualizes |
| **Metrics** | Numeric measurements over time      | **Prometheus + Node Exporter / App metrics**: CPU, memory, request counts, error rates                       |
| **Traces**  | Track a request across services     | **Jaeger / OpenTelemetry**: Shows request flow in microservices, latency per service                         |
Simple example:

App throws an error → log collected by Fluentd → visualized in Kibana
CPU spike → metric scraped by Node Exporter → Prometheus alert
User request slow → trace collected by Jaeger → shows which service caused delay

==============================================================================================================

🎤 5-Minute Java Observability Script with Examples

*"For observability in a Java microservice, I instrumented the app using the Prometheus Java client. I exposed both JVM metrics and custom application metrics.

Example 1 – JVM Metrics: Using simpleclient_hotspot, Prometheus automatically collects heap usage, garbage collection stats, and thread counts. For instance, jvm_memory_used_bytes shows the memory currently used by the application.

Example 2 – Custom Metrics: Developers added a counter for HTTP requests and a histogram for request durations:

``` static final Counter httpRequests = Counter.build()
    .name("http_requests_total")
    .help("Total HTTP requests")
    .register();

static final Histogram requestDuration = Histogram.build()
    .name("http_request_duration_seconds")
    .help("Request duration in seconds")
    .register();
```

These metrics are exposed at the /metrics endpoint on port 8080.

Next, I containerized the service using Docker:

``` docker build -t myrepo/java-app:latest .
docker run -p 8080:8080 myrepo/java-app:latest
```

and deployed it to Kubernetes using a Deployment and Service.

Prometheus is configured to scrape metrics from the Java app, along with Node Exporter for system metrics (port 9100) and MySQL Exporter for database metrics (port 9104). For example, the Prometheus scrape_configs for the Java app:

scrape_configs:
  - job_name: 'java-app'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: java-app

This ensures Prometheus dynamically discovers all pods running the Java service.

In production, I integrate HashiCorp Vault with Alertmanager to manage Slack webhooks securely. The webhook is stored in Vault, injected into the Alertmanager pod, and referenced in the AlertmanagerConfig. Alerts like High CPU usage or Pod restarts are routed to the Slack receiver with repeat intervals, and all sensitive data remains secure and auditable. 
``` # Enable KV engine if not already
vault secrets enable -path=secret kv-v2

# Store Slack webhook in Vault
vault kv put secret/alertmanager slack-webhook=https://hooks.slack.com/services/TOKEN
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: slack-alert-config
  namespace: monitoring
spec:
  route:
    receiver: 'slack-notifications'
    repeatInterval: 10m
    routes:
      - matchers:
          - name: alertname
            value: HighCpuUsage
        receiver: 'slack-notifications'
        repeatInterval: 5m
      - matchers:
          - name: alertname
            value: PodRestart
        receiver: 'slack-notifications'
        repeatInterval: 1m

  receivers:
    - name: 'slack-notifications'
      slackConfigs:
        - sendResolved: true
          channel: '#alerts'
          text: "🚨 Alert: {{ .CommonAnnotations.summary }}\nSeverity: {{ .CommonLabels.severity }}\nInstance: {{ .CommonLabels.instance }}"
          apiURL:
            file: /vault/secrets/slack  # Vault-injected secret
    - name: 'null'  # Catch-all receiver
```

Visualization: Grafana dashboards (port 3000) display JVM metrics, HTTP request counts, response times, and database metrics.

Logging & Tracing: I implemented structured logging using the EFK stack (Elasticsearch, FluentBit, Kibana) to collect logs from both the app and nodes. For distributed tracing, I used Jaeger to track requests across microservices, enabling easy debugging of performance issues."*

| Component           | Port | Notes                            |
| ------------------- | ---- | -------------------------------- |
| Java App `/metrics` | 8080 | Scraped by Prometheus            |
| Prometheus          | 9090 | Main UI and API                  |
| Alertmanager        | 9093 | Sends notifications              |
| Node Exporter       | 9100 | Server metrics                   |
| Grafana             | 3000 | Dashboards                       |
| kube-state-metrics  | 8080 | Kubernetes cluster state metrics |


```
prometheus.yaml
 # Global settings applied to all scrape jobs
global:
  scrape_interval: 15s         # How often Prometheus scrapes metrics
  evaluation_interval: 15s     # How often rules are evaluated

MySQL Exporter is running in pods that can restart or scale, you don’t use static targets. Instead, you use Kubernetes service discovery:

- job_name: 'mysql'
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_label_app]
      action: keep
      regex: mysqld-exporter

Prometheus automatically discovers all pods with label app=mysqld-exporter

No need to manually update targets if pods move or restart
```

# Monitoring

## Metrics vs Monitoring

Metrics are measurements or data points that tell you what is happening. For example, the number of steps you walk each day, your heart rate, or the temperature outside—these are all metrics.

Monitoring is the process of keeping an eye on these metrics over time to understand what’s normal, identify changes, and detect problems. It's like watching your step count daily to see if you're meeting your fitness goal or checking your heart rate to make sure it's in a healthy range.

| Exporter               | What it monitors         | Example Metrics                                                        | Notes                               |
| ---------------------- | ------------------------ | ---------------------------------------------------------------------- | ----------------------------------- |
| **Node Exporter**      | Server / hardware        | `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`             | Installed on each server            |
| **MySQL Exporter**     | MySQL database           | `mysql_global_status_queries`, `mysql_global_status_threads_connected` | Monitors DB performance             |
| **kube-state-metrics** | Kubernetes cluster state | `kube_pod_status_phase`, `kube_deployment_status_replicas`             | Does **not** collect CPU/memory     |
| **Custom App Metrics** | Application-specific     | `http_requests_total`, `http_request_duration_seconds`                 | Exposed by developers at `/metrics` |


## 🚀 Prometheus
- Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud.
- It is known for its robust data model, powerful query language (PromQL), and the ability to generate alerts based on the collected time-series data.
- It can be configured and set up on both bare-metal servers and container environments like Kubernetes.

## 🏠 Prometheus Architecture
- The architecture of Prometheus is designed to be highly flexible, scalable, and modular.
- It consists of several core components, each responsible for a specific aspect of the monitoring process.

![Prometheus Architecture](images/prometheus-architecture.gif)

### 🔥 Prometheus Server
- Prometheus server is the core of the monitoring system. It is responsible for scraping metrics from various configured targets, storing them in its time-series database (TSDB), and serving queries through its HTTP API.
- Components:
    - **Retrieval**: This module handles the scraping of metrics from endpoints, which are discovered either through static configurations or dynamic service discovery methods.
    - **TSDB (Time Series Database)**: The data scraped from targets is stored in the TSDB, which is designed to handle high volumes of time-series data efficiently.
    - **HTTP Server**: This provides an API for querying data using PromQL, retrieving metadata, and interacting with other components of the Prometheus ecosystem.
- **Storage**: The scraped data is stored on local disk (HDD/SSD) in a format optimized for time-series data.

### 🌐 Service Discovery
- Service discovery automatically identifies and manages the list of scrape targets (i.e., services or applications) that Prometheus monitors.
- This is crucial in dynamic environments like Kubernetes where services are constantly being created and destroyed.
- Components:
    - **Kubernetes**: In Kubernetes environments, Prometheus can automatically discover services, pods, and nodes using Kubernetes API, ensuring it monitors the most up-to-date list of targets.
    - **File SD (Service Discovery)**: Prometheus can also read static target configurations from files, allowing for flexibility in environments where dynamic service discovery is not used.

### 📤 Pushgateway
- The Pushgateway is used to expose metrics from short-lived jobs or applications that cannot be scraped directly by Prometheus.
- These jobs push their metrics to the Pushgateway, which then makes them available for Prometheus to scrape(pull).
- Use Case:
    - It's particularly useful for batch jobs or tasks that have a limited lifespan and would otherwise not have their metrics collected.

### 🚨 Alertmanager
- The Alertmanager is responsible for managing alerts generated by the Prometheus server.
- It takes care of deduplicating, grouping, and routing alerts to the appropriate notification channels such as PagerDuty, email, or Slack.

### 🧲 Exporters
- Exporters are small applications that collect metrics from various third-party systems and expose them in a format Prometheus can scrape. They are essential for monitoring systems that do not natively support Prometheus.
- Types of Exporters:
    - Common exporters include the Node Exporter (for hardware metrics), the MySQL Exporter (for database metrics), and various other application-specific exporters.

### 🖥️ Prometheus Web UI
- The Prometheus Web UI allows users to explore the collected metrics data, run ad-hoc PromQL queries, and visualize the results directly within Prometheus.

### 📊 Grafana
- Grafana is a powerful dashboard and visualization tool that integrates with Prometheus to provide rich, customizable visualizations of the metrics data.

### 🔌 API Clients
- API clients interact with Prometheus through its HTTP API to fetch data, query metrics, and integrate Prometheus with other systems or custom applications.

# 🛠️  Installation & Configurations
## 📦 Step 1: Create EKS Cluster

### Prerequisites
- Download and Install AWS Cli - Please Refer [this]("https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html") link.
- Setup and configure AWS CLI using the `aws configure` command.
- Install and configure eksctl using the steps mentioned [here]("https://eksctl.io/installation/").
- Install and configure kubectl as mentioned [here]("https://kubernetes.io/docs/tasks/tools/").


```bash
eksctl create cluster --name=observability \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
```
```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster observability \
    --approve
```
```bash
eksctl create nodegroup --cluster=observability \
                        --region=us-east-1 \
                        --name=observability-ng-private \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=20 \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking

# Update ./kube/config file
aws eks update-kubeconfig --name observability
```

### 🧰 Step 2: Install kube-prometheus-stack
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 🚀 Step 3: Deploy the chart into a new namespace "monitoring"
```bash
kubectl create ns monitoring
```
```bash
cd day-2

helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
-f ./custom_kube_prometheus_stack.yml
```

### ✅ Step 4: Verify the Installation
```bash
kubectl get all -n monitoring
```
- **Prometheus UI**:
```bash
kubectl port-forward service/prometheus-operated -n monitoring 9090:9090
```

**NOTE:** If you are using an EC2 Instance or Cloud VM, you need to pass `--address 0.0.0.0` to the above command. Then you can access the UI on <instance-ip:port>

- **Grafana UI**: password is `prom-operator`
```bash
kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
```
- **Alertmanager UI**:
```bash
kubectl port-forward service/alertmanager-operated -n monitoring 9093:9093
```

### 🧼 Step 5: Clean UP
- **Uninstall helm chart**:
```bash
helm uninstall monitoring --namespace monitoring
```
- **Delete namespace**:
```bash
kubectl delete ns monitoring
```
- **Delete Cluster & everything else**:
```bash
eksctl delete cluster --name observability
```
