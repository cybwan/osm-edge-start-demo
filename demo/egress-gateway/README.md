# OSM Edge Egress Gateway Test

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

## 3. Egress Gateway Test

### 3.1 Technical concepts

There are two types of policies for Egress in OSM Edge:

- Destination Control Policies
  - **Permissive Mode**: Release all external traffic
  - **Policy Mode**: Release explicitly configured external traffic
- Egress Control Policies
  - **Side car pass-through**: external traffic goes directly to the external destination via the side car
  - **Gateway Proxy**: External traffic is sent to the egress gateway via the side car and then to the external destination
    - **Global egress proxy gateway**

A combination of policies that can satisfy four business scenarios.

- Destination permissive mode + edge sidecar pass-through
- Destination policy mode + edge sidecar pass-through
- Destination Loose Mode + Global Egress Proxy Gateway
- Destination policy mode + global egress proxy gateway

### 3.2 Testing Environment Requirements

#### 3.2.1 Deployment Operations POD

```bash
kubectl create namespace curl
osm namespace add curl
kubectl apply -n curl -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress-gateway/curl.yaml
```

#### 3.2.2 Deploying the Global Egress Gateway

```bash
# Ignore possible duplicate namespace creation errors
kubectl create namespace egress-gateway
kubectl apply -n egress-gateway -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress-gateway/global-egress-gateway-rbac.yaml
kubectl apply -n egress-gateway -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress-gateway/global-egress-gateway-service.yaml
kubectl apply -n egress-gateway -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress-gateway/global-egress-gateway-configmap.yaml
kubectl apply -n egress-gateway -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress-gateway/global-egress-gateway-deployment.yaml
```

#### 3.2.3 Wait for dependent PODs to start properly

```bash
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=180s
kubectl wait --for=condition=ready pod -n egress-gateway -l app=global-egress-gateway --timeout=180s
```

### 3.3 Scenario test case#1: destination permissive mode + side car pass-through

#### 3.3.1 Disabling Egress destination permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.3.2 Test commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://httpbin.org:80/get
```

#### 3.3.3 Test Results

Returns the correct result:

```bash
command terminated with exit code 52
```

#### 3.3.4  Enabling Egress permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}' --type=merge
```

#### 3.3.5 Test Commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://httpbin.org:80/get
```

#### 3.3.6 Test Results

he correct return result might look similar to :

```bash
HTTP/1.1 200 OK
Date: Fri, 16 Sep 2022 07:50:40 GMT
Content-Type: application/json
Content-Length: 257
Connection: keep-alive
Server: gunicorn/19.9.0
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

This business scenario is tested and the strategy is cleaned up to avoid affecting subsequent tests

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

### 3.4 Scenario test case#2: Destination policy mode + side car pass-through

#### 3.4.1 Enable Egress destination policy mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.4.2 Test Commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://httpbin.org:80/get
```

#### 3.4.3 Test Results

Returns the correct result:

```bash
command terminated with exit code 52
```

#### 3.4.4 Setting Egress Destination Policy

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin-80
  namespace: curl
spec:
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
  hosts:
  - httpbin.org
  ports:
  - number: 80
    protocol: http
EOF
```

#### 3.4.5 Test Commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://httpbin.org:80/get
```

#### 3.4.6 Test Results

The correct return result might look similar to :

```bash
HTTP/1.1 200 OK
date: Sat, 17 Sep 2022 12:35:42 GMT
content-type: application/json
content-length: 572
server: gunicorn/19.9.0
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```

#### 3.4.7 Test Commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://edition.cnn.com?test=3.3.7
```

#### 3.4.8 Test Results

The request does not have a corresponding destination policy, and the side car will simply reject it, returning a correct result similar to :

```bash
HTTP/1.1 403 Forbidden
content-length: 13
connection: keep-alive
```

This business scenario is tested and the strategy is cleaned up to avoid affecting subsequent tests

```bash
kubectl delete egress -n curl httpbin-80
```

### 3.5 Scenario test case#3: Destination Permissive Mode + Global Egress Proxy Gateway

#### 3.5.1 Enabling Egress destination permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":true}}}' --type=merge
```

#### 3.5.2 Setting the Global Egress Proxy Gateway Policy

```bash
kubectl apply -f - <<EOF
kind: EgressGateway
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: global-egress-gateway
  namespace: egress-gateway
spec:
  global:
    - service: global-egress-gateway
      namespace: egress-gateway
EOF
```

#### 3.5.3 Test Commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://edition.cnn.com?test=3.5.3
```

#### 3.5.4 Test Results

The correct return result might look similar to :

```bash
HTTP/1.1 301 Moved Permanently
Server: Varnish
Retry-After: 0
Content-Length: 0
Cache-Control: public, max-age=300
Location: https://edition.cnn.com/?test=3.5.3
Accept-Ranges: bytes
Date: Sat, 17 Sep 2022 11:23:24 GMT
Via: 1.1 varnish
Connection: close
Set-Cookie: countryCode=US; Domain=.cnn.com; Path=/; SameSite=Lax
Set-Cookie: stateCode=CA; Domain=.cnn.com; Path=/; SameSite=Lax
Set-Cookie: geoData=fremont|CA|94539|US|NA|-700|broadband|37.520|-121.930|807; Domain=.cnn.com; Path=/; SameSite=Lax
X-Served-By: cache-pao17462-PAO
X-Cache: HIT
X-Cache-Hits: 0
```

Global egress proxy gateway traffic log:

```bash
#Log output might have some latency (max 15 seconds)
kubectl logs -n egress-gateway "$(kubectl get pod -n egress-gateway -l app=global-egress-gateway -o jsonpath='{.items..metadata.name}')" | grep edition.cnn.com | jq
#Traffic log return
{
  "connection_id": "b2316fbb5ab14034",
  "request_time": "2022-09-17T11:51:33.075Z",
  "source_address": "10.243.1.6",
  "source_port": 42316,
  "host": "edition.cnn.com",
  "path": "/?test=3.5.3",
  "method": "HEAD"
}
{
  "id": "b2316fbb5ab14034",
  "start_time": "2022-09-17T11:51:33.074Z",
  "source_address": "10.243.1.6",
  "source_port": 42316,
  "destination_address": "edition.cnn.com",
  "destination_port": 80,
  "end_time": "2022-09-17T11:51:33.603Z"
}
```

This business scenario is tested and the strategy is cleaned up to avoid affecting subsequent tests

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge

kubectl delete egressgateways -n egress-gateway global-egress-gateway
```

### 3.6 Scenario Test case#4: Destination Policy Mode + Global Egress Proxy Gateway

#### 3.6.1 Disabling Egress destination permissive mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.6.2 Enabling Egress Destination Policy Mode

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.6.3 Setting the Egress destination policy

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin-80
  namespace: curl
spec:
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
  hosts:
  - httpbin.org
  ports:
  - number: 80
    protocol: http
EOF
```

#### 3.6.4 Setting the Global Egress Proxy Gateway Policy

```bash
kubectl apply -f - <<EOF
kind: EgressGateway
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: global-egress-gateway
  namespace: egress-gateway
spec:
  global:
    - service: global-egress-gateway
      namespace: egress-gateway
EOF
```

#### 3.6.5 Test commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://httpbin.org:80/get?test=3.6.5
```

#### 3.6.6 Test Results

The correct return result might look similar to :

```bash
HTTP/1.1 200 OK
date: Sat, 17 Sep 2022 11:55:20 GMT
content-type: application/json
content-length: 606
server: gunicorn/19.9.0
access-control-allow-origin: *
access-control-allow-credentials: true
connection: keep-alive
```

Global egress proxy gateway traffic log:

```bash
#Log output might have some latency (max 15 seconds)
kubectl logs -n egress-gateway "$(kubectl get pod -n egress-gateway -l app=global-egress-gateway -o jsonpath='{.items..metadata.name}')" | grep httpbin.org | jq
#Traffic log returns
{
  "id": "62445ee7ebeb455e",
  "start_time": "2022-09-17T11:46:19.920Z",
  "source_address": "10.243.1.6",
  "source_port": 47026,
  "destination_address": "httpbin.org",
  "destination_port": 80,
  "end_time": "2022-09-17T11:46:25.686Z"
}
{
  "connection_id": "b7afbcab6f284c5c",
  "request_time": "2022-09-17T11:55:20.278Z",
  "source_address": "10.243.1.6",
  "source_port": 41160,
  "host": "httpbin.org",
  "path": "/get?test=3.6.5",
  "method": "HEAD"
}
```

#### 3.6.7 Test Commands

```bash
kubectl exec "$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n curl -c curl -- curl -sI http://edition.cnn.com?test=3.5.3
```

#### 3.6.8 Test Results

The request does not have a corresponding destination policy, and the side car will simply reject it, returning a correct result similar to :

```bash
HTTP/1.1 403 Forbidden
content-length: 13
connection: keep-alive
```

This business scenario is tested and the strategy is cleaned up to avoid affecting subsequent tests

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge

kubectl delete egress -n curl httpbin-80
kubectl delete egressgateways -n egress-gateway global-egress-gateway
```