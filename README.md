# envoy-spire-opa-service-mesh

Demo to build Service Mesh on Kubernetese using Envoy as data plane and SPIRE and OPA as control plane. This demo is [zlabjp/spiffejp-demo](https://github.com/zlabjp/spiffejp-demo) with OPA added.

- Use Kubernetes 1.17.0
- Use Envoy 1.12.2, SPIRE 0.9.0 and OPA 0.15.1
- Envoy uses SPIRE as SDS Server to obtain TLS certificate
- Envoy uses OPA as External Authorization Server to check if the incoming request is authorized or not

Blog is [here](https://qiita.com/ryysud). (Japanese)

## Policy Overview

![policy-overview](/img/policy-overview.png)

Four services are running in Kubernetes Cluster.

- ec-web（ `spiffe://example.org/ec-web` ）
  - `ec-web` **ONLY** accept requests with custom header "X-Opa-Secret" containing "ec-web-secret" value
- ec-backend（ `spiffe://example.org/ec-backend` ）
  - `ec-backend` can **ONLY** be accessed from `ec-web`
- news-web（ `spiffe://example.org/news-web` ）
  - `news-web` **ONLY** accept requests with custom header "X-Opa-Secret" containing "news-web-secret" value
- news-backend（ `spiffe://example.org/news-backend` ）
  - `news-backend` can **ONLY** be accessed from `news-web`

## Architecture

![architecture](/img/architecture.png)

- Authentication using SPIRE
- Authorization using OPA

## 1. Create Kubernetes Cluster

Create a Kubernetes cluster with the required flags to run NodeAttestor "k8s_psat" in SPIRE.

```bash
minikube start \
    --kubernetes-version v1.17.0 \
    --extra-config=apiserver.service-account-signing-key-file=/var/lib/minikube/certs/sa.key \
    --extra-config=apiserver.service-account-key-file=/var/lib/minikube/certs/sa.pub \
    --extra-config=apiserver.service-account-issuer=api \
    --extra-config=apiserver.service-account-api-audiences=api,spire-server
```

## 2. Deploy SPIRE Server

Deploy SPIRE Server as StatefulSet.

```bash
kubectl apply -f spire-server.yaml
```

## 3. Deploy SPIRE Agent

Deploy SPIRE Agent as DaemonSet.

```bash
kubectl apply -f spire-agent.yaml
```

## 4. Create Registration Entries in SPIRE Server

Create node entry.

```bash
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/node \
    -selector k8s_psat:cluster:demo-cluster \
    -node
```

Create workload entries.

```bash
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -parentID spiffe://example.org/node \
    -spiffeID spiffe://example.org/ec-web \
    -selector k8s:pod-label:app:ec-web

kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -parentID spiffe://example.org/node \
    -spiffeID spiffe://example.org/ec-backend \
    -selector k8s:pod-label:app:ec-backend

kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -parentID spiffe://example.org/node \
    -spiffeID spiffe://example.org/news-web \
    -selector k8s:pod-label:app:news-web

kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry create \
    -parentID spiffe://example.org/node \
    -spiffeID spiffe://example.org/news-backend \
    -selector k8s:pod-label:app:news-backend
```

Confirm the created registration entries.

```bash
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry show
```

## 5. Deploy four services

Deploy four services. SPIRE Agent (Unix Domain Socket) mounts on Envoy container and run Envoy and OPA as sidecar.

```bash
kubectl apply -f ec-backend.yaml
kubectl apply -f ec-web.yaml
kubectl apply -f news-backend.yaml
kubectl apply -f news-web.yaml
```

## 6. Check policy

### ✅ Request to ec-web and news-web Envoy with custom header "X-Opa-Secret" via Ingress

Expect `200 OK` response.

```bash
curl -i -H 'Host: ec-web.example.org' -H 'X-Opa-Secret: ec-web-secret' http://$(minikube ip)/noproxy
curl -i -H 'Host: news-web.example.org' -H 'X-Opa-Secret: news-web-secret' http://$(minikube ip)/noproxy
```

### ❌ Request to ec-web and news-web Envoy without custom header "X-Opa-Secret" via Ingress

Expect `403 Forbidden` response.

```bash
curl -i -H 'Host: ec-web.example.org' http://$(minikube ip)/noproxy
curl -i -H 'Host: news-web.example.org' http://$(minikube ip)/noproxy
```

### ✅ Request to /data of (ec|news)-backend from (ec|news)-web

Expect `200 OK` response.

```bash
curl -i -H 'Host: ec-web.example.org' -H 'X-Opa-Secret: ec-web-secret' http://$(minikube ip)/data
curl -i -H 'Host: news-web.example.org' -H 'X-Opa-Secret: news-web-secret' http://$(minikube ip)/data
```

### ❌ Request to /admin of (ec|news)-backend from (ec|news)-web

Expect `403 Forbidden` response.

```bash
curl -i -H 'Host: ec-web.example.org' -H 'X-Opa-Secret: ec-web-secret' http://$(minikube ip)/admin
curl -i -H 'Host: news-web.example.org' -H 'X-Opa-Secret: news-web-secret' http://$(minikube ip)/admin
```

### ❌ Request to /data of ec-backend from news-web

Expect `403 Forbidden` response.

```bash
curl -i -H 'Host: news-web.example.org' -H 'X-Opa-Secret: news-web-secret' http://$(minikube ip)/ec-backend-data
```

## 7. Cleaning Up

```bash
kubectl delete -f spire-server.yaml
kubectl delete -f spire-agent.yaml
kubectl delete -f ec-backend.yaml
kubectl delete -f ec-web.yaml
kubectl delete -f news-backend.yaml
kubectl delete -f news-web.yaml
```

## References

- [Envoy External Authorization with OPA](https://blog.openpolicyagent.org/envoy-external-authorization-with-opa-578213ed567c)

## License

MIT
