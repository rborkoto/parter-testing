# Datadog Observability Setup for Example Voting App

This guide walks you through integrating Datadog observability (Infrastructure, APM, Logs, DBM, RUM, Synthetics, Security, Network Monitoring) into the Example Voting App running on Kubernetes.

---

## Step 3 — Deploy the Datadog Agent

1. **Add the Datadog Helm repository**  
   ```bash
   helm repo add datadog https://helm.datadoghq.com
   helm repo update
   ```

2. **Create `datadog-values.yaml`** in this folder with at least the following (replace `<DATADOG_API_KEY>`):  
   ```yaml
   datadog:
     apiKey: "<DATADOG_API_KEY>"
     site: datadoghq.com
     logs:
       enabled: true
     apm:
       enabled: true
     processAgent:
       enabled: true
     clusterAgent:
       enabled: true
     orchestratorExplorer:
       enabled: true
   agents:
     containerLogs:
       containerCollectAll: true
   ```

3. **Install / upgrade the agent**  
   ```bash
   helm upgrade --install datadog datadog/datadog      -f datadog-values.yaml --namespace datadog --create-namespace
   ```

4. **Verify the deployment**  
   ```bash
   kubectl get pods -n datadog
   kubectl exec -it "$(kubectl get pod -n datadog -l app=datadog -o jsonpath='{.items[0].metadata.name}')" -n datadog -- agent status
   ```

---

## Step 4 — Enable the Orchestration Explorer

- The Orchestration Explorer visualizes your pods, deployments, and services automatically once `orchestratorExplorer.enabled = true` is set in your Helm values.  
- In the Datadog UI → **Explorer** → **Orchestration**, you should see your `voting-app` namespace resources.

---

## Step 5 — Configure Integrations

### Redis Integration

Annotate the Redis service for autodiscovery by adding to `k8s-specifications/redis-deployment.yaml` under metadata:

```yaml
annotations:
  ad.datadoghq.com/redis.check_names: '["redis"]'
  ad.datadoghq.com/redis.init_configs: '[{}]'
  ad.datadoghq.com/redis.instances: |
    [{
      "host": "%%host%%",
      "port": 6379
    }]
```

### Postgres (DBM) Integration

In your `datadog-values.yaml`, enable the Postgres check:

```yaml
datadog:
  databaseMonitoring:
    enabled: true

agents:
  containers:
    postgres:
      check_names: ["postgres"]
      init_configs: [{}]
      instances:
        - host: "db.voting-app.svc.cluster.local"
          port: 5432
          username: "postgres"
          password: "docker"
          dbname: "postgres"
```

- **Validate** in Datadog UI → **Databases** → **Postgres** Dashboard.

---

## Step 6 — Kubernetes Control Plane Integrations

Use any integration method (Annotations, ConfigMap, or values.yaml) different from what you used for Redis/Postgres.

Enable:
- **CoreDNS**  
- **etcd**  
- **Kube Scheduler**  
- **Kube Controller Manager**  
- **Kube API Server**  

For example, via ConfigMap in your `datadog-values.yaml`:

```yaml
datadog:
  confd:
    kube_coredns.yaml: |-
      init_config: {}
      instances:
        - host: "%%host%%"
          port: 9153
    kube_apiserver.yaml: |-
      init_config: {}
      instances:
        - host: "%%host%%"
          port: 443
    # ... add others similarly
```

---

## Step 7 — Enable Log Collection

With `logs.enabled: true`, the agent tails container logs automatically.  

- **Verify** in Datadog UI → **Logs** → see log streams from `voting-app` pods.

---

## Step 8 — Enable Security Monitoring

1. **Logs-based Security**  
   - In Datadog UI → **Security** → **Policies**, enable logs-based detection.
2. **Runtime Security**  
   - Follow in-app instructions under **Security** → **Runtime** → **Kubernetes** → deploy the `security-agent` via Helm addon.

---

## Step 9 — Configure APM

### Flask Frontend (`vote`)

In your Flask `index.html` template, add before `</head>`:

```html
<script src="https://www.datadoghq-browser-agent.com/datadog-rum-v4.js"></script>
<script>
  window.DD_RUM && window.DD_RUM.init({
    clientToken: "<DATADOG_CLIENT_TOKEN>",
    applicationId: "<DATADOG_APP_ID>",
    site: "datadoghq.com",
    service:"vote-frontend",
    env:"sandbox",
    version:"1.0.0",
    sampleRate:100,
    trackResources:true,
    trackLongTasks:true,
    trackInteractions:true
  });
</script>
```

Instrument the Python backend with `ddtrace`:

```bash
pip install ddtrace
```

Wrap your `app.py` startup:

```python
from ddtrace import patch_all, tracer
patch_all()
# existing imports and Flask app...
```

### .NET Worker

Add the Datadog .NET tracer:

- In your Dockerfile for `worker`, download and inject the tracer, and set environment variables:
  ```dockerfile
  RUN wget -O /worker/dd-trace.dll https://github.com/DataDog/dd-trace-dotnet/releases/latest/download/datadog-agent.zip
  ENV DD_SERVICE="voting-worker"       DD_ENV="sandbox"       DD_TRACE_ENABLED=true
  ENTRYPOINT ["dotnet", "voting-worker.dll"]
  ```

### Node.js Result App

Install and configure `dd-trace`:

```bash
npm install dd-trace
```

At the top of your `app.js`:

```js
const tracer = require('dd-trace').init({
  service: 'result-frontend',
  env: 'sandbox'
});
// existing express setup...
```

- **Restart** pods: `kubectl rollout restart deployment -n voting-app`

- **Verify** in Datadog UI → **APM** → look for `vote-frontend`, `voting-worker`, `result-frontend`.

---

## Step 10 — Configure Synthetics

1. In Datadog UI → **Synthetics** → create new **API Test** for your `vote` endpoint.
2. Create a **Browser Test** pointing to the Vote UI URL.
3. (If using Minikube) get URL via:
   ```bash
   minikube service vote -n voting-app --url
   ```

---

## Step 11 — Configure Real User Monitoring (RUM)

Front-end RUM snippet already added in Step 9 for `vote`. For deeper session replay:

- In `datadog-values.yaml`, enable session replay:
  ```yaml
  datadog:
    rum:
      enabled: true
      sessionReplay:
        enabled: true
  ```

- **Upgrade** the agent:
  ```bash
  helm upgrade --install datadog datadog/datadog -f datadog-values.yaml -n datadog
  ```

---

## Step 12 — Network Performance Monitoring (NPM)

In `datadog-values.yaml`, enable NPM:

```yaml
datadog:
  networkMonitoring:
    enabled: true
agents:
  networkMonitoring:
    enabled: true
```

- **Upgrade** the agent as above and view **Network Map** in Datadog UI → **Network**.

---

## Appendix — Troubleshooting

- **Agent status**:  
  ```bash
  kubectl exec -n datadog -it $(kubectl get pods -n datadog -l app=datadog -o jsonpath='{.items[0].metadata.name}') -- agent status
  ```
- **Logs**: `kubectl logs -n datadog <agent-pod>`
- **Documentation**: Check Datadog’s official docs and FAQs.  
- **Support**: Use Zendesk if required.
