These notes are made mostly for myself, for didactical purposes. Pull requests are welcome.

# Create a k3s cluster

## ...on Civo

```
civo k3s create test --remove-applications=traefik --nodes=1 --size=g2.xsmall --wait --save --switch
```

Traefik 2 is not supported on k3s; valid alternatives exist: https://github.com/rancher/k3s/issues/817

## ...on VPS or bare metal

See: https://github.com/alexellis/k3sup

# Deploy something on k3s

## Preparation

Please be sure that your DNS is configured properly and install Helm.

## Setup nginx

This step is required if Traefik has been removed.

```
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm install nginx nginx-stable/nginx-ingress
```

Reference: https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/#installing-the-chart

## Deploy a service

References:
- https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/basic-configuration/

```
kubectl apply -f is-osm-uptodate.yaml
```

Test if the service is reachable over HTTP.

## Setup SSL

```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.14.1
```

References:
- https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm

An alternative way to create a namespace is to apply the following YAML file:
```
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
```

### Setup letsencrypt


References:
- https://cert-manager.io/docs/configuration/acme/#creating-a-basic-acme-issuer
- https://letsencrypt.org/docs/acme-protocol-updates/#acme-v2-rfc-8555

```
kubectl apply -f letsencrypt-staging.yaml
kubectl apply -f letsencrypt.yaml
```

## Use SSL

Uncomment the TLS section of `is-osm-uptodate.yaml` and the line referring to `letsencrypt-staging`, then update the deployment.

```
kubectl apply -f is-osm-uptodate.yaml
```

Try to connect over HTTPS and check if the fake certificate has been generated:

```
kubectl get secrets is-osm-uptodate-tls -o yaml | grep tls\.crt | awk '{ print $2 }' | base64 -d > cert
openssl x509 -in cert -text
```

Switch to the real letsencrypt certificate by commenting the line with `letsencrypt-staging` and uncommenting the following one, then update the deployment:

```
kubectl apply -f is-osm-uptodate.yaml
```



