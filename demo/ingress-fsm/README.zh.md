# OSM Edge Ingress 测试

## 1. 下载并安装 osm-edge 命令行工具

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.2.0
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

## 2. 安装 osm-edge&FSM Ingress

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

## 3. Pipy Ingress 测试

### 3.1 部署业务 POD

```bash
#模拟业务服务
kubectl create namespace httpbin
osm namespace add httpbin
kubectl apply -n httpbin -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/ingress-fsm/httpbin.yaml

#模拟外部客户端
kubectl create namespace ext-curl
kubectl apply -n ext-curl -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress-fsm/curl.yaml

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n httpbin -l app=httpbin --timeout=180s
kubectl wait --for=condition=ready pod -n ext-curl -l app=curl --timeout=180s
```

### 3.2 设置 Ingress 策略

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

### 3.3 测试指令

```bash
kubectl exec "$(kubectl get pod -n ext-curl -l app=curl -o jsonpath='{.items..metadata.name}')" -n ext-curl -- curl -sI http://fsm-ingress-pipy-controller.osm-system:80/get -H "Host: httpbin.org"
```

### 3.4 测试结果

正确返回结果类似于:

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
