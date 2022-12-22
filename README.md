# Sharded / Tenanted Gateways with Gloo Edge and Gloo Portal

This project is for demonstrating sharded gateways with Gloo Edge and Gloo Portal.
This deploys two individual gateways for applications running in namespaces `tenant1` and `tenant2`. 
Each tenant in the namespace has a logical separation with its own gateway.

## Prerequisites

1. Create required env vars

    ```
    export PROJECT="ge-sharded-gateways"

    export DOMAIN_NAME=testing.development.internal

    export GLOO_EDGE_HELM_VERSION=1.12.40
    export GLOO_EDGE_VERSION=v${GLOO_EDGE_HELM_VERSION}

    export GLOO_PORTAL_HELM_VERSION=1.3.0-beta17
    export GLOO_PORTAL_VERSION=v${GLOO_PORTAL_HELM_VERSION}
    ```

2. Provision cluster
    ```
    colima start --runtime containerd --kubernetes --kubernetes-disable-servicelb -p $PROJECT -c 4 -m 8 -d 20 --network-address --install-metallb --metallb-address-pool "192.168.106.230/29" --kubernetes-version v1.23.14-rc3+k3s1
    ```

## Deployment

### Setup Gloo Edge & Gloo Portal

```
helm repo add gloo-ee https://storage.googleapis.com/gloo-ee-helm
helm repo update

helm install gloo-ee gloo-ee/gloo-ee -n gloo-system \
  --version ${GLOO_EDGE_HELM_VERSION} \
  --create-namespace \
  --set-string license_key=${GLOO_EDGE_LICENSE_KEY} \
  -f gloo-edge-helm-values.yaml

helm repo add gloo-portal https://storage.googleapis.com/dev-portal-helm
helm repo update

helm install gloo-portal gloo-portal/gloo-portal -n gloo-portal \
  --version ${GLOO_PORTAL_HELM_VERSION} \
  --create-namespace \
  -f gloo-portal-helm-values.yaml
```

## Setup Application and Configuration

```
kubectl create ns tenant1
kubectl create ns tenant1-configuration

kubectl create ns tenant2
kubectl create ns tenant2-configuration

kubectl apply -n tenant1 -f apps/httpbin/deployment.yaml
kubectl apply -n tenant2 -f apps/petstore/deployment.yaml

kubectl apply -n tenant1-configuration -f configuration/httpbin/upstream.yaml
kubectl apply -n tenant2-configuration -f configuration/petstore/upstream.yaml

envsubst < <(cat configuration/httpbin/vs.yaml) | kubectl apply -n tenant1-configuration -f -

envsubst < <(cat configuration/petstore/env.yaml) | kubectl apply -n tenant2-configuration -f -
envsubst < <(cat configuration/petstore/portal.yaml) | kubectl apply -n tenant2-configuration -f -
kubectl apply -n tenant2-configuration -f configuration/petstore/product.yaml
kubectl apply -n tenant2-configuration -f configuration/petstore/schema.yaml
```