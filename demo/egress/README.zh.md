# OSM Edge Egress 测试

## 1. 下载并安装 osm-edge 命令行工具

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.2.0
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

## 2. 安装 osm-edge

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
    --set=osm.sidecarLogLevel=error \
    --set=osm.controllerLogLevel=warn \
    --timeout=900s
```

## 3. Egress策略测试

### 3.1 技术概念

在 OSM Edge 中Egress策略：

- 支持的源类型：
  - ServiceAccount
- 支持的目的类型：
  - 基于域名，支持的协议：
    - http
    - tcp
  - 基于IP范围，支持的协议：
    - tcp
- 支持的传输类型
  - 明文传输，支持的出口控制策略：
    - 边车透传
    - 网关代理
  - mTLS加密传输，支持的出口控制策略：
    - 采用边车透传模式，自动忽略网关代理


### 3.2 部署业务 POD

```bash
#模拟外部服务
kubectl create namespace egress-server
kubectl apply -n egress-server -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/egress/httpbin.yaml
kubectl apply -n egress-server -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-nginx/server.yaml

#模拟业务服务
kubectl create namespace egress-client
osm namespace add egress-client
kubectl apply -n egress-client -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-nginx/client.yaml

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n egress-server -l app=httpbin --timeout=180s
kubectl wait --for=condition=ready pod -n egress-server -l app=server --timeout=180s
kubectl wait --for=condition=ready pod -n egress-client -l app=client --timeout=180s
```

### 3.3 场景测试一：基于域名的外部访问，明文传输

#### 3.3.1 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.3.2 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.3.3 测试指令

```bash
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -sI httpbin.egress-server.svc.cluster.local:14001
```

#### 3.4.4 测试结果

正确返回结果类似于:

```bash
command terminated with exit code 52
```

#### 3.4.5 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin-14001
  namespace: egress-client
spec:
  sources:
  - kind: ServiceAccount
    name: client
    namespace: egress-client
  hosts:
  - httpbin.egress-server.svc.cluster.local
  ports:
  - number: 14001
    protocol: http
EOF
```

#### 3.3.6 测试指令

```bash
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -sI httpbin.egress-server.svc.cluster.local:14001
```

#### 3.3.7 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
server: pipy
date: Sat, 08 Oct 2022 23:38:14 GMT
content-type: text/html; charset=utf-8
content-length: 9593
access-control-allow-origin: *
access-control-allow-credentials: true
x-pipy-upstream-service-time: 3
connection: keep-alive
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete egress -n egress-client httpbin-14001
```

### 3.4 场景测试二：基于 IP 范围的外部访问，明文传输

#### 3.4.1 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.4.2 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.4.3 测试指令

```bash
httpbin_pod_ip="$(kubectl get pod -n egress-server -l app=httpbin -o jsonpath='{.items[0].status.podIP}')"
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -sI ${httpbin_pod_ip}:14001
```

#### 3.4.4 测试结果

正确返回结果类似于:

```bash
command terminated with exit code 52
```

#### 3.4.5 设置Egress目的策略

```bash
httpbin_pod_ip="$(kubectl get pod -n egress-server -l app=httpbin -o jsonpath='{.items[0].status.podIP}')"
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: httpbin-14001
  namespace: egress-client
spec:
  sources:
  - kind: ServiceAccount
    name: client
    namespace: egress-client
  ipAddresses:
  - ${httpbin_pod_ip}/32
  ports:
  - number: 14001
    protocol: tcp
EOF
```

#### 3.4.6 测试指令

```bash
httpbin_pod_ip="$(kubectl get pod -n egress-server -l app=httpbin -o jsonpath='{.items[0].status.podIP}')"
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -sI ${httpbin_pod_ip}:14001
```

#### 3.4.7 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
Server: gunicorn/19.9.0
Date: Sat, 08 Oct 2022 23:41:39 GMT
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Content-Length: 9593
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete egress -n egress-client httpbin-14001
```

### 3.5 场景测试三：基于域名的外部访问，业务自实现 mTLS

#### 3.5.1 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.5.2 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.5.3 测试指令

```bash
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -ksi https://server.egress-server.svc.cluster.local:8443/time --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### 3.5.4 测试结果

正确返回结果类似于:

```bash
command terminated with exit code 35
```

#### 3.5.5 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-client
spec:
  sources:
  - kind: ServiceAccount
    name: client
    namespace: egress-client
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: https
EOF
```

#### 3.5.6 测试指令

```bash
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -ksi https://server.egress-server.svc.cluster.local:8443/time --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### 3.5.7 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
content-type: text/plain; charset=utf-8
content-length: 75
date: Sun, 09 Oct 2022 00:24:03 GMT

The current time: 2022-10-09 00:24:03.830082283 +0000 UTC m=+2154.622989759
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete egress -n egress-client server-8443
```

### 3.6 场景测试四：基于域名的外部访问，边车mTLS

#### 3.6.1 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.6.2 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.6.3 测试指令

```bash
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -ksi http://server.egress-server.svc.cluster.local:8443/time
```

#### 3.6.4 测试结果

正确返回结果类似于:

```bash
command terminated with exit code 52
```

#### 3.6.5 创建Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/client.crt -o client.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/client.key -o client.key

kubectl create secret generic -n osm-system egress-client-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./client.crt \
  --from-file=tls.key=./client.key 
```

#### 3.6.6 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-client
spec:
  sources:
  - kind: ServiceAccount
    name: client
    namespace: egress-client
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-client-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### 3.6.7 测试指令

```bash
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -ksi http://server.egress-server.svc.cluster.local:8443/time
```

#### 3.6.8 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
date: Sun, 09 Oct 2022 07:48:15 GMT
content-type: text/plain; charset=utf-8
server: pipy
x-pipy-upstream-service-time: 32
content-length: 74
connection: keep-alive

The current time: 2022-10-09 07:48:15.069213069 +0000 UTC m=+269.209148917
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete egress -n egress-client server-8443
kubectl delete secrets -n osm-system egress-client-cert
```

### 3.7 场景测试五：基于 IP 范围的外部访问，业务自实现 mTLS

#### 3.7.1 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.7.2 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.7.3 测试指令

```bash
server_pod_ip="$(kubectl get pod -n egress-server -l app=server -o jsonpath='{.items[0].status.podIP}')"
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -ksi https://$server_pod_ip:8443/time -H "Host: server.egress-server.svc.cluster.local" --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### 3.7.4 测试结果

正确返回结果类似于:

```bash
command terminated with exit code 35
```

#### 3.7.5 设置Egress目的策略

```bash
server_pod_ip="$(kubectl get pod -n egress-server -l app=server -o jsonpath='{.items[0].status.podIP}')"
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-client
spec:
  sources:
  - kind: ServiceAccount
    name: client
    namespace: egress-client
  ipAddresses:
  - ${server_pod_ip}/32
  ports:
  - number: 8443
    protocol: tcp
EOF
```

#### 3.7.6 测试指令

```bash
server_pod_ip="$(kubectl get pod -n egress-server -l app=server -o jsonpath='{.items[0].status.podIP}')"
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -ksi https://$server_pod_ip:8443/time -H "Host: server.egress-server.svc.cluster.local" --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### 3.7.7 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
content-type: text/plain; charset=utf-8
content-length: 75
date: Sun, 09 Oct 2022 00:48:33 GMT

The current time: 2022-10-09 00:48:33.580642547 +0000 UTC m=+3625.463281090
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete egress -n egress-client server-8443
```

### 3.8 场景测试六：基于 IP 范围的外部访问，边车mTLS

#### 3.8.1 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 3.8.2 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 3.8.3 测试指令

```bash
server_pod_ip="$(kubectl get pod -n egress-server -l app=server -o jsonpath='{.items[0].status.podIP}')"
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -si http://$server_pod_ip:8443/time -H "Host: server.egress-server.svc.cluster.local"
```

#### 3.8.4 测试结果

正确返回结果类似于:

```bash
command terminated with exit code 52
```

#### 3.8.5 创建Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/client.crt -o client.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/client.key -o client.key

kubectl create secret generic -n osm-system egress-client-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./client.crt \
  --from-file=tls.key=./client.key 
```

#### 3.8.6 设置Egress目的策略

```bash
server_pod_ip="$(kubectl get pod -n egress-server -l app=server -o jsonpath='{.items[0].status.podIP}')"
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-client
spec:
  sources:
  - kind: ServiceAccount
    name: client
    namespace: egress-client
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-client-cert
          namespace: osm-system
  ipAddresses:
  - ${server_pod_ip}/32
  ports:
  - number: 8443
    protocol: tcp
EOF
```

#### 3.8.7 测试指令

```bash
server_pod_ip="$(kubectl get pod -n egress-server -l app=server -o jsonpath='{.items[0].status.podIP}')"
curl_client="$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items[0].metadata.name}')"
kubectl exec ${curl_client} -n egress-client -c client -- curl -ksi http://$server_pod_ip:8443/time -H "Host: server.egress-server.svc.cluster.local"
```

#### 3.8.8 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
content-type: text/plain; charset=utf-8
content-length: 75
date: Sun, 09 Oct 2022 00:48:33 GMT

The current time: 2022-10-09 00:48:33.580642547 +0000 UTC m=+3625.463281090
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete egress -n egress-client server-8443
kubectl delete secrets -n osm-system egress-client-cert
```
