# K8s Sandbox

[description]

## Features

- Devshell
- K3d with three clusters (control, apps eu and apps us)
- Sops secrets for secrets management
- Ingress Nginx
- Argocd with SSO
- ApplicationSet for preview PR
- Gvizor
- Monitoring
    - Grafana with kube-prom-stack
    - Custom dashboards and alerts created as code
    - OtelCollector
    - VM Metrics
    - VM Logs/Loki
    - Alert manager
    - Grafana Tempo
    - K8s events
- Renovate
- Teleport
- Argo rollouts
- Keda
- Argo Rollouts
- Kargo for progressive rollouts between envs
- Windmill just for test
- Trivy or Kiverno

- Postgres DB operator
- Atlas schema migration
- Service mesh?
- Crossplane
- Github action runner


## How to setup this beauty on local machine

### Install all the dependencies

Nix must be installed before running this command. If you don't want to install nix, take a look at the dependencies from [./devshell.toml](./devshell.toml) and make sure they are available on your host.

```
$ nix-shell
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

### Add the applications

```sh
kubectl apply -f apps/init-argo-apps.yaml --context k3d-sandbox-control
```


### Monitoring

Services related to monitoring, most of them deployed to the monitoring cluster.
Monitoring
We use victoria-metrics as a drop-in replacement for Prometheus with long-time storage. Tempo for traces, Loki and Promtail for logging, opentelemetry-collector for tracing proxy, and kube-prometheus-stack for k8s monitoring.

kubeRBACProxy is enabled for node-exporter because there is a port exposed by node-exporter Daemonset that should not be open to the internet without auth.

kube-prometheus-stack is installed on every cluster that has the label monitoring: true. Data is sent to the central Victoria Metrics store(monitoring cluster) using Prometheus remoteWrite.

kube-prometheus-extra is installed just on the monitoring cluster and is a subset of kube-prometheus-stack with just Grafana dashboards and Prometheus rules and alerts enabled.

Because remoteWrite is used for aggregating all the data in VictoriaMetrics, Prometheus rules and alerts can be deployed just on the monitoring cluster, where all the data resides.
Victoria Metrics
In order to add a new scrape job for prometheus, use kind: ServiceMonitor CRD with label "kube-prometheus-stack-{{ .Values.cluster }}". For external services, use the scrape_config attribute from the victoria-metrics helm value files.
Logging
Promtail for logging is installed on every cluster that has the label logging: true. Logs are sent to Loki in the monitoring cluster and can be visualized in Grafana.
Alerts
Alerts from Grafana are sent directly to PagerDuty. There is a contact point defined in Grafana for every application/service, and every contact point is mapped to a different PagerDuty service.

Alerts defined in kube-prometheus-stack are sent to alert-manager that in turn sends them to PagerDuty fc-cluster service.


All prometheus instances are sending data to vm-single, from there vm-alert is reading and sending data to alertmanager and back to vm-single. Because we use prometheus in agent-mode, only victoria-metrics operator will read and ingest PrometheusRules. In non-agent mode, prometheus would also read and ingest PrometheusRules. Victoria metrics collects servicemonitor from all namespaces and prometheus operator is doing the same thing. We enable victoria metrics operator just for the rules. Prometheus is not connected to any Alertmanagers, because Prometheus from the control node is in agent mode and does not have all the necessary data.

https://docs.victoriametrics.com/guides/multi-regional-setup-dedicated-regions/


### Things to improve

- it should be easier to just read the hostname and append the api subdomain in front on frontend
- improve cluster setup
    - in monitoring/victoria-metrics/vm-auth.yaml and monitoring/promtail/helm-values-files/clusters/all/values.json we have [collector.172.20.0.5.nip.io](http://metrics-collector.172.20.0.5.nip.io/), it would be nice to not have the IP hardcoded
- dashboards grafana as json files
- service monitor targets both alertmanager and alertmanager-headless
- alert manager (and grafana) can be installed with kube-prom-stack
- vmlogs does not have backup/recovery solutions and it cannot offload old logs to a bucket (alternative, loki provides a link to the branch)
- datasources, providers, dashboards, alerting, plugins can be loaded in grafana from the helm value file or from configmaps using sidecare (we can place the configmaps in any namespace, for example, we can put datasources victoria metrics configmap in the victoria metrics folder)


### Alternatives

- Grafana Alloy is compatible with both the OpenTelemetry Collector and Prometheus Agent which means it can be used as an alternative to either of these solutions https://grafana.com/blog/2025/01/03/how-to-send-otlp-or-prometheus-metrics-and-logs-to-grafana-cloud-with-grafana-alloy/
- https://vector.dev/
- https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-kube-stack
- https://docs.victoriametrics.com/operator/resources/vmalertmanagerconfig/
- https://github.com/VictoriaMetrics/helm-charts/tree/master/charts/victoria-metrics-anomaly


The current kargo example can be greatly simplified if all the necessary rollout manifests would be moved in the playground-sandbox github helm chart repo. Basically we should have just the values.yaml files and the secrets in the [./apps/playground-sandbox/envs](./apps/playground-sandbox/envs) folder. I'm choosing to keep them in this repo for now because it is easier from a learning perspective to discover all the files.
