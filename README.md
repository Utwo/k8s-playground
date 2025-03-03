# 🚀 K8s Sandbox

Gitops repo with k8s tools simulating a real enterprise project

## 🔥 Features

✅ ***Implemented:***
- Devbox
- K3d with two clusters (control and apps)
- Sops secrets for secrets management
- Ingress Nginx
- Argocd with SSO
- ApplicationSet for preview PR
- Monitoring
    - VM Metrics
    - Alert Manager
    - Grafana
        - Custom dashboards and alerts created as code
    - Collectors:
        - Grafana Alloy (k8s-monitoring-stack)
        - kube-prometheus-stack
        - Opentelemetry kube stack
    - Grafana Tempo
    - Logs:
        - Grafana Loki
        - VM Logs
    - K8s events
- Renovate
- Argo rollouts
- Keda
- Argo Rollouts
- Kargo for progressive rollouts between envs
- Windmill just for test
- Trivy
- Teleport

⏳ ***To-do:***
- [vmalertmanagerconfig](https://docs.victoriametrics.com/operator/resources/vmalertmanagerconfig/)
- [victoria-metrics-anomaly](https://github.com/VictoriaMetrics/helm-charts/tree/master/charts/victoria-metrics-anomaly)
- Gvizor
- Postgres DB operator
- Atlas schema migration
- Service mesh
- Crossplane
- Github action runner


## 🏗️How to setup this beauty on local machine

### Install all the dependencies

Nix and devbox must be installed before running this command. If you don't want to install nix, take a look at the dependencies from [./devbox.json](./devbox.json) and make sure they are available on your host.

```sh
$ devbox shell
```

### Create a k3d cluster with a local volume

```sh
$ k3d cluster create --config k3d-control-config.yaml
```

### Configure SOPS locally

First generate a public/private key pair:
```sh
age-keygen -o ./ignore/key.txt
```

Copy the public generate key and paste it in the [.sops.yaml](.sops.yaml) file on the `age:` attribute.

Next, create a K8s secret with the generated key for the sops-operator to be able to decrypt the secrets:

```sh
kubectl create namespace sops-operator
kubectl create secret generic sops-age-key -n sops-operator --from-file=./ignore/key.txt
```

### Deploy infra related resources

Create a new Github OAuth application if you want to use [SSO with ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#dex).
Paste the client id and secret in to the [argocd-secrets.yaml](infra/argocd/argocd-secrets.yaml) encryptedfile.

```sh
kubectl apply -k infra/argocd
kubectl apply -f infra/init-argo-apps.yaml
```

Visit ArgoCD UI at https://argocd.127.0.0.1.nip.io

### Add the second application cluster

Create the k3d cluster:
```sh
k3d cluster create --config k3d-apps-config.yaml --volume $(pwd)/tmp:/tmp/k3dvol
```

Create a K8s secret with the generated key for the sops-operator to be able to decrypt the secrets:

```sh
kubectl create namespace sops-operator
kubectl create secret generic sops-age-key -n sops-operator --from-file=./ignore/key.txt
```

Add the SA, Role and RB to the target cluster
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

### Add the rest of the components from monitoring or apps folders

Add everything with one command:
```sh
kubectl apply -f apps/init-argo-apps.yaml --context k3d-sandbox-apps
```

Add one by one:
```sh
kubectl apply -f apps/argo-apps/<app-name> --context k3d-sandbox-apps
```

## Projects

### Apps - our own develop applications

We use this space for defining applications developed by us. Applications are deployed to k3d-sandbox-apps cluster. We have defined here, `production`, `staging`, `dev` as well as `ephemeral environments` created from PR environemnts.

The application deployed here is just an example of a basic backend/frontend service. The code and helm chart can be found on the [playground-sandbox repo](https://github.com/Utwo/playground-sandbox).

### Infra - infrastructure related services

#### ArgoCD
[ArgoCD](./infra/argo-apps/argocd.yaml) is a continuous deployment tool for gitOps workflows.

SSO with Github Login is enabled for the ArgoCD web UI using Github OAuth App.
Another Github APP is used to:
* read the content of the playground-sandbox repo and to bypass the rate limit rule.
* send argocd notifications on open pull requests when we deploy an ephemeral environment

We use one repo to deploy to multiple clusters. Registered clusters can be found on the [clusters](./infra/argocd/clusters/) folder. For each cluster, we define a couple of labels that are later used to deploy or fill information in the CD pipeline.

Visit ArgoCD UI at https://argocd.127.0.0.1.nip.io

#### SOPS operator

[SOPS operator](./infra/argo-apps/sops-secret-operator.yaml) is used to decrypt the sops secrets stored in the git repository and transform them into kubernetes secrets

Below is an example of how to create a secret with sops to safely store it on git.

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
[Argo Rollouts](./infra/argo-apps/argo-rollouts.yaml) is a Kubernetes controller and set of CRDs which provide advanced deployment capabilities such as blue-green, canary, canary analysis, experimentation, and progressive delivery features to Kubernetes.

We use Argo Rollouts to enable canary deployments for the `playground-sandbox` app using `ingress-ngnix`.
To expose the Argo Rollouts web UI locally run:
```sh
kubectl port-forward services/argo-rollouts-k3d-apps-dashboard 3100:3100 -n argo-rollouts --context k3d-sandbox-apps
```

#### Kargo
[Kargo](./infra/argo-apps/kargo.yaml) is a continuous promotion orchestration layer, that complements Argo CD for Kubernetes. With Kargo we can define and promote the steps necesary to deploy to `dev`, `staging` and `production`. The project is defined [here](./infra/kargo/projects/playground-sandbox/). It listens for both this repo and the application repo and when there is a new change, it generates the plain manifests from the app helm chart. The output is then pushed to [stage/dev branch](https://github.com/Utwo/k8s-playground/tree/stage/dev), [stage/staging branch](https://github.com/Utwo/k8s-playground/tree/stage/staging) or [stage/production branch](https://github.com/Utwo/k8s-playground/tree/stage/production) and applied to ArgoCD.

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
[vmlogs](./monitoring/argo-apps/victoria-metrics.yaml) is used for collecting logs. It was added just as a prof of concept. It does not have backup/recovery solutions and it cannot offload old logs to a bucket

#### Kube Prometheus Stack
[kube-prometheus-stack](./monitoring/argo-apps/kube-prometheus-stack.yaml) is used just for deploying Prometheus rules, alerts and Grafana dashboards. Everything else is disabled because, Grafana Alloy is used for collecting metrics and ServiceMonitors.

#### Grafana Alloy
[Alloy](./monitoring/argo-apps/alloy.yaml) is an open-telemetry collector distribution, used to collect metrics, logs, traces and profiles. It is installed on every cluster that has the label `alloy: true`. Alloy also installs Prometheus CRDs to collect metrics from `ServiceMonitor`. It is also used to collect Kubernetes events. Logs are sent to Loki, traces to Tempo and metrics to Victoria Metrics. All the data can be visualized in Grafana.

#### Alertmanager
Alerts defined in kube-prometheus-stack are sent to alert-manager. From there we can define multiple routes to send them to external services like Pagerduty.
Alerts from Grafana can be sent directly to external services like Pagerduty or to our own Alertmanager. One option would be to have a contact point defined in Grafana for every application/service, and every contact point to be mapped to a different PagerDuty service.

---
Alert Manager and Grafana can be installed with kube-prometheus-stack but I prefer to have it over separate argo app to swap/update components easier.
Datasources, providers, dashboards, alerting, plugins can be loaded in grafana from the helm value file or from configmaps using sidecare (we can place the configmaps in any namespace, for example, we can put datasources victoria metrics configmap in the victoria metrics folder).
Promtail is discontinued and Alloy is the new recommanded option for collecting logs.
Kubernetes Event Exporter is not needed anymore since Alloy is also collecting Kubernetes events.

### Services - other services and deployments

Services that cannot be added to any other category.

[Trivy](./services/argo-apps/trivy.yaml) is a security scanner that finds vulnerabilities, misconfigurations, secrets, SBOM in containers.

[Windmill](./services/argo-apps/windmill.yaml) is a developer platform and workflow engine. Turn scripts into auto-generated UIs, APIs and cron jobs. Compose them as workflows or data pipelines.
