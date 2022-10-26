# OSM Edge Ingress Test

## 1. Download and install the `osm-edge` command line tool

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.2.0
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

## 2. Install `osm-edge` and `FSM Ingress`

```bash
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.certificateProvider.kind=tresor \
    --set=osm.image.registry=flomesh \
    --set=osm.image.tag=1.2.0 \
    --set=osm.image.pullPolicy=Always \
    --set=osm.enableEgress=false \
    --set=osm.sidecarLogLevel=error \
    --set=osm.controllerLogLevel=warn \
    --set=fsm.enabled=true \
    --timeout=900s
```

## 3. Ingress Test

### 3.1 Deploy business POD

```bash
#Simulate business service
kubectl create namespace httpbin
osm namespace add httpbin
kubectl apply -n httpbin -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/ingress-fsm/httpbin.yaml

#Simulate external client
kubectl create namespace ext-curl
kubectl apply -n ext-curl -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress-gateway/curl.yaml

#Wait for the dependent POD to start normally
kubectl wait --for=condition=ready pod -n httpbin -l app=httpbin --timeout=180s
kubectl wait --for=condition=ready pod -n ext-curl -l app=curl --timeout=180s
```

### 3.2 Setting up Ingress Rules

```bash
export osm_namespace=osm-system
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: httpbin
spec:
  ingressClassName: pipy
  rules:
  - host: httpbin.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 14001      
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin
  namespace: httpbin
spec:
  backends:
  - name: httpbin
    port:
      number: 14001 # targetPort of httpbin service
      protocol: http
  sources:
  - kind: Service
    namespace: "$osm_namespace"
    name: fsm-ingress-pipy-controller
EOF
```

### 3.3 Test commands

```bash
kubectl exec "$(kubectl get pod -n ext-curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n ext-curl -- curl -sI http://fsm-ingress-pipy-controller.osm-system:80/get -H "Host: httpbin.org"
```

### 3.4 Test Results

The correct return result might look similar to :

```bash
HTTP/1.1 200 OK
server: gunicorn/19.9.0
date: Fri, 16 Sep 2022 10:44:13 GMT
content-type: application/json
content-length: 247
access-control-allow-origin: *
access-control-allow-credentials: true
osm-stats-namespace: httpbin
osm-stats-kind: Deployment
osm-stats-name: httpbin
osm-stats-pod: httpbin-7c6464475-pz5gf
connection: keep-alive
```
