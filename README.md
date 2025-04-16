# 🚀 K8s Sandbox

A GitOps repository with Kubernetes tools simulating a real enterprise project.

---

## 🔥 Features

✅ ***Implemented:***
- **Development & Cluster Setup:**
  - Devbox
  - K3d with two clusters (control and apps)
  - Sops secrets for secrets management
  - Ingress Nginx
- **GitOps & Deployment:**
  - Argocd with SSO
  - ApplicationSet for preview PR
  - Argo Rollouts (Blue-Green & Canary Deployments)
  - Kargo for progressive rollouts between environments
  - Postgres DB operator
- **Monitoring & Logging:**
  - VM Metrics
  - Alert Manager
  - Grafana (Dashboards & Alerts as Code)
  - Metrics Collectors:
      - Grafana Alloy (k8s-monitoring-stack)
      - kube-prometheus-stack
      - Opentelemetry kube stack
  - Grafana Tempo (Tracing)
  - Logs:
      - Grafana Loki
      - VM Logs
  - Kubernetes Events
- **Automation & Security:**
  - Renovate
  - Keda
  - Windmill just for test
  - Trivy
  - Teleport

⏳ ***To-do:***
- [vmalertmanagerconfig](https://docs.victoriametrics.com/operator/resources/vmalertmanagerconfig/)
- [victoria-metrics-anomaly](https://github.com/VictoriaMetrics/helm-charts/tree/master/charts/victoria-metrics-anomaly)
- Gvizor
- Atlas schema migration
- Service mesh
- Crossplane
- Github action runner

---

## 🏗️How to set this beauty on the local machine

### Install Dependencies

Ensure Nix and Devbox are installed. If you prefer not to install Nix, manually install the dependencies listed in [./devbox.json](./devbox.json)
```sh
$ devbox shell
```

### Create a K3d Cluster

```sh
$ k3d cluster create --config k3d-control-config.yaml
```

### Configure SOPS Locally

1. Generate an encryption key pair:
  ```sh
  age-keygen -o ./ignore/key.txt
  ```
2. Copy the **public key** into `.sops.yaml` under the `age:` attribute.
3. Create a K8s secret with the generated key for the sops-operator to be able to decrypt the secrets:
  ```sh
  kubectl create namespace sops-operator
  kubectl create secret generic sops-age-key -n sops-operator --from-file=./ignore/key.txt
  ```

### Deploy Infrastructure Resources

1. Create a **GitHub OAuth Application** if you want to use [SSO with ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#dex). Paste the client ID and secret into the [argocd-secrets.yaml](infra/argocd/argocd-secrets.yaml) encryptedfile.

2. Deploy ArgoCD:
  ```sh
  kubectl apply -k infra/argocd
  kubectl apply -f infra/init-argo-apps.yaml
  ```

Access the ArgoCD UI at [https://argocd.127.0.0.1.nip.io](https://argocd.127.0.0.1.nip.io)

### Add the Second Application Cluster

```sh
k3d cluster create --config k3d-apps-config.yaml --volume $(pwd)/tmp:/tmp/k3dvol
```

Create a K8s secret with the generated key for the sops-operator to be able to decrypt the secrets:

```sh
kubectl create namespace sops-operator
kubectl create secret generic sops-age-key -n sops-operator --from-file=./ignore/key.txt
```

Add the SA, Role and, RB to the target cluster
```sh
argocd cluster add k3d-sandbox-apps --name k3d-sandbox-apps --kube-context k3d-sandbox-control
```


Get the token from the argocd-manager-token secret:
```sh
kubectl get secret argocd-manager-token-* -n kube-system -o yaml --context k3d-sandbox-apps
```

Create a new secret in [infra/argocd/clusters/k3d-apps-secret.yaml](infra/argocd/clusters/k3d-apps-secret.yaml) following this [example](infra/argocd/clusters/example-secret.yaml):
```
bearerToken ⇒ token from argocd-manager-token secret. You must base64 decode it. `echo <token> | base64 -d`
caData ⇒ ca.crt from argocd-manager-token secret.

# not needed for this example
certData ⇒ client-certificate-data from kube context
keyData ⇒ client-key-data from kube context
```
Encrypt the secret:
```sh
sops encrypt --in-place infra/argocd/clusters/k3d-apps-secret.yaml
```

### Deploy Applications & Monitoring Components

Deploy all applications:
```sh
kubectl apply -f apps/init-argo-apps.yaml --context k3d-sandbox-apps
```

Deploy individual applications:
```sh
kubectl apply -f apps/argo-apps/<app-name> --context k3d-sandbox-apps
```

## 📂 Project Structure

### **Apps** - Our Own Applications

We use this space for defining applications developed by us. Applications are deployed to `k3d-sandbox-apps` cluster. We have defined here, `production`, `staging`, `dev` as well as `ephemeral environments` created from PR environments.

The application deployed here is just an example of a basic backend/frontend service. The code and helm chart can be found on the [playground-sandbox repo](https://github.com/Utwo/playground-sandbox).

Available URLs:
* http://dev.127.0.0.1.nip.io:8000
* http://staging.127.0.0.1.nip.io:8000
* http://127.0.0.1.nip.io:8000

### **Infra** - Infrastructure Services

#### ArgoCD

[ArgoCD](./infra/argo-apps/argocd.yaml) is a continuous deployment tool for gitOps workflows.

SSO with Github Login is enabled for the ArgoCD web UI using the Github OAuth App.
Another Github APP is used to:
* read the content of the playground-sandbox repo and bypass the rate limit rule.
* send argocd notifications on open pull requests when we deploy an ephemeral environment

It uses one repo to deploy to multiple clusters. Registered clusters can be found in the [clusters](./infra/argocd/clusters/) folder. For each cluster, we define a couple of labels that are later used to deploy or fill information in the CD pipeline.

Visit ArgoCD UI at https://argocd.127.0.0.1.nip.io

#### SOPS operator

[SOPS operator](./infra/argo-apps/sops-secret-operator.yaml) is used to decrypt the sops secrets stored in the git repository and transform them into Kubernetes secrets

Below is an example of how to create a secret with Sops to safely store it on git.

Create a yaml SopsSecret:

```yaml
apiVersion: isindir.github.com/v1alpha3
kind: SopsSecret
metadata:
  name: sopssecret-sample
spec:
  secretTemplates:
    - name: sopssecret-sample
      labels:
        label0: value0
        labelK: valueK
      annotations:
        key0: value0
        keyN: valueN
      stringData:
        data-name0: data-value0
        data-nameL: data-valueL
```

Encrypt the secret:
```sh
sops encrypt test.yaml > test.enc.yaml
```

Decrypt the secret:
```sh
SOPS_AGE_KEY_FILE=./ignore/key.txt sops test.enc.yaml
```

#### Argo Rollouts
[Argo Rollouts](./infra/argo-apps/argo-rollouts.yaml) is a Kubernetes controller and set of CRDs that provide advanced deployment capabilities such as blue-green, canary, canary analysis, experimentation, and progressive delivery features to Kubernetes.

We use Argo Rollouts to enable canary deployments for the `playground-sandbox` app using `ingress-ngnix`.
To expose the Argo Rollouts web UI locally run:
```sh
kubectl port-forward services/argo-rollouts-k3d-apps-dashboard 3100:3100 -n argo-rollouts --context k3d-sandbox-apps
```

#### Kargo
[Kargo](./infra/argo-apps/kargo.yaml) is a continuous promotion orchestration layer, that complements Argo CD for Kubernetes. With Kargo we can define and promote the steps necessary to deploy to `dev`, `staging`, and `production`. The project is defined [here](./infra/kargo/projects/playground-sandbox/). It listens for both this repo and the application repo and when there is a new change, it generates the plain manifests from the app helm chart. The output is then pushed to [stage/dev branch](https://github.com/Utwo/k8s-playground/tree/stage/dev), [stage/staging branch](https://github.com/Utwo/k8s-playground/tree/stage/staging) or [stage/production branch](https://github.com/Utwo/k8s-playground/tree/stage/production) and applied to ArgoCD.

To expose the Kargo web UI locally run:
```sh
kubectl port-forward services/kargo-api 4000:443 -n kargo --context k3d-sandbox-control
```

#### Keda
[KEDA](./infra/argo-apps/keda.yaml) is a Kubernetes-based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.

We use Keda to scale ephemeral environments to 0 when they receive no traffic.

### Monitoring - observability related services

Services related to monitoring. Most of them should be deployed to a dedicated monitoring cluster.

#### Grafana
Grafana is used for visualizing metrics, aggregated logs, and tracings. It has SSO with Github enabled.
To expose the Grafana web UI locally run:
```sh
kubectl port-forward services/grafana 3000:3000 -n grafana --context k3d-sandbox-control
```

#### Victoria Metrics
[Victoria Metrics](./monitoring/argo-apps/victoria-metrics.yaml) is used as a drop-in replacement for Prometheus with long-time storage. This service is exposed in order for other clusters to be able to write prometheus metrics. All prometheus instances/alloy are sending data to vm-single, from there vm-alert is reading and sending data to alertmanager and back to vm-single. We enable victoria metrics operator just for the rules.

https://docs.victoriametrics.com/guides/multi-regional-setup-dedicated-regions/

#### Grafana Tempo
[Tempo](./monitoring/argo-apps/tempo.yaml) is used for storing traces.

#### Grafana Loki
[Loki](./monitoring/argo-apps/loki.yaml) is used for storing logs. This service is exposed in order for other clusters to be able to send logs here.

#### Victoria Metrics logs
[vmlogs](./monitoring/argo-apps/victoria-metrics.yaml) is used for collecting logs. It was added just as a proof of concept. It does not have backup/recovery solutions and it cannot offload old logs to a bucket. For now, we will rely on Loki for storing logs.

#### Kube Prometheus Stack
[kube-prometheus-stack](./monitoring/argo-apps/kube-prometheus-stack.yaml) is used just for deploying Prometheus rules, alerts and Grafana dashboards. Everything else is disabled because Grafana Alloy is used for collecting metrics and ServiceMonitors.

#### Grafana Alloy
[Alloy](./monitoring/argo-apps/alloy.yaml) is an open-telemetry collector distribution, used to collect metrics, logs, traces and, profiles. It is installed on every cluster that has the label `alloy: true`. Alloy also installs Prometheus CRDs to collect metrics from `ServiceMonitor`. It is also used to collect Kubernetes events. Logs are sent to Loki, traces to Tempo and, metrics to Victoria Metrics. All the data can be visualized in Grafana.

#### Opentelemetry Kube Stack
[Opentelemetry Kube Stack](./monitoring/argo-apps/opentelemetry-kube-stack.yaml) is an open-telemetry collector distribution, used to collect metrics, logs and traces. It is similar to Grafana Alloy but I couldn't make it work with the kube-prometheus-stack dashboards. We miss the `job` label and because of that the dashboards are not populated. Open [issue](https://github.com/open-telemetry/opentelemetry-helm-charts/issues/1545#issuecomment-2694671722).

#### Alertmanager
Alerts defined in kube-prometheus-stack are sent to alert-manager. From there we can define multiple routes to send them to external services like Pagerduty.
Alerts from Grafana can be sent directly to external services like Pagerduty or to our own Alertmanager. One option would be to have a contact point defined in Grafana for every application/service, and every contact point to be mapped to a different PagerDuty service.

---
Alert Manager and Grafana can be installed using kube-prometheus-stack, but I prefer to handle them as a separate Argo application. This approach makes it easier to swap or update components.
In Grafana, datasources, providers, dashboards, alerting configurations, and plugins can be loaded through the Helm values file or from ConfigMaps using a sidecar. We can place the ConfigMaps in any namespace; for example, we can store the Victoria Metrics datasource in the Victoria Metrics folder.
Note that Promtail has been discontinued, and Alloy is now the recommended option for collecting logs.
Kubernetes Event Exporter is no longer necessary, as Alloy also collects Kubernetes events.

### Services - other services and deployments

Services that cannot be added to any other category.

[Trivy](./services/argo-apps/trivy.yaml) is a security scanner that finds vulnerabilities, misconfigurations, secrets, and SBOM in containers.

[Windmill](./services/argo-apps/windmill.yaml) is a developer platform and workflow engine. Turn scripts into auto-generated UIs, APIs and, cron jobs. Compose them as workflows or data pipelines.
