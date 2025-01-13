# Playground Sandbox

[description]

## Features

[list of features]


## How to setup this beauty on local machine

### Install all the dependencies

Nix must be installed before running this command. If you don't want to install nix, take a look at the dependencies from [./devshell.toml](./devshell.toml) and make sure they are available on your host.

```
$ nix-shell
```

### Create a k3d cluster with a local volume

```
$ k3d cluster create playground-sandbox --volume $(pwd)/tmp/k3dvol:/tmp/k3dvol -p"80:80@loadbalancer" -p "443:443@loadbalancer" --k3s-node-label "node=default@server:0"
```

### Configure SOPS locally

First generate a public/private key pair:
```sh
age-keygen -o ./ignore/key.txt
```

Create a K8s secret with the generated key:

```sh
kubectl create namespace sops-operator
kubectl create secret generic sops-age-key -n sops-operator --from-file=./ignore/key.txt
```

### Deploy infra related resources

```sh
kubectl apply -f infra/init-argo-apps.yaml
```

### Others

For opening traefik web UI in the browser:

```

kubectl port-forward -n kube-system "$(kubectl get pods -n kube-system| grep '^traefik-' | awk '{print $1}')" 9000:9000

```

If you run this on something else than k3d, then maybe you need to change the k8s internal ip in nginx.
First get the ip:

```

$ kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
$ kubectl exec -i -t dnsutils -- nslookup kubernetes.default

```

Update nginx-cm.yaml with the new ip;
