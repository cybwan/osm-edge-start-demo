# OSM Edge 双臂 mTLS 测试

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

## 3. 双臂 mTLS 测试

### 3.1 技术概念

<img src="https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-access-control/Bidirectional_mTLS.png" alt="Bidirectional_mTLS" style="zoom:80%;" />

### 3.2 部署业务 POD

```bash
#模拟时间服务
kubectl create namespace egress-server
kubectl apply -n egress-server -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-access-control/server.yaml

#模拟中间件服务
kubectl create namespace bidi-mtls-middle
osm namespace add bidi-mtls-middle
kubectl apply -n bidi-mtls-middle -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-access-control/middle.yaml

#模拟外部客户端
kubectl create namespace bidi-mtls-client
kubectl apply -n bidi-mtls-client -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-access-control/client.yaml

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n egress-server -l app=server --timeout=180s
kubectl wait --for=condition=ready pod -n bidi-mtls-middle -l app=middle --timeout=180s
kubectl wait --for=condition=ready pod -n bidi-mtls-client -l app=client --timeout=180s
```

### 3.3 场景测试

#### 3.3.1 启用访问控制策略

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableAccessControlPolicy":true}}}'  --type=merge
```

#### 3.3.2 启用证书颁发策略

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableAccessCertPolicy":true}}}'  --type=merge
```

#### 3.3.3 为客户端创建证书 Secret

```bash
kubectl apply -f - <<EOF
kind: AccessCert
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: client-mtls-cert
  namespace: bidi-mtls-middle
spec:
  subjectAltNames:
  - client.bidi-mtls-client.cluster.local
  secret:
    name: client-mtls-secret
    namespace: bidi-mtls-client
EOF
```

#### 3.3.4 客户端挂在证书 Secret

```bash
#模拟外部客户端
kubectl apply -n bidi-mtls-client -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-access-control/client-mtls.yaml

#等待依赖的 POD 正常启动
```

#### 3.3.5 设置基于服务的访问控制策略

```bash
export osm_namespace=osm-system
kubectl apply -f - <<EOF
kind: AccessControl
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: client2middle
  namespace: bidi-mtls-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of httpbin service
      protocol: http
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    namespace: bidi-mtls-client
    name: client
  - kind: AuthenticatedPrincipal
    name: client.bidi-mtls-client.cluster.local
EOF
```

#### 3.3.6 测试指令

流量路径: 

Client --**mtls**--> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n bidi-mtls-client -l app=client -o jsonpath='{.items..metadata.name}')" -n bidi-mtls-client -- curl -ksi https://middle.bidi-mtls-middle:8080/hello --cacert /certs/ca.crt --key /certs/tls.key --cert /certs/tls.crt
```

#### 3.3.7 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
date: Tue, 11 Oct 2022 13:55:28 GMT
content-length: 13
content-type: text/plain; charset=utf-8
osm-stats-namespace: bidi-mtls-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-rwlr8

hello world.
```

#### 4.3.8 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 4.3.9 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 4.3.10 创建Egress mTLS Secret

```bash
kubectl apply -f - <<EOF
kind: AccessCert
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-mtls-cert
  namespace: bidi-mtls-middle
spec:
  subjectAltNames:
  - server.egress-server.svc.cluster.local
  secret:
    name: server-mtls-secret
    namespace: egress-server
EOF
```

#### 3.3.11 服务端挂在证书 Secret

```bash
#模拟时间服务
kubectl apply -n egress-server -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-access-control/server-mtls.yaml

#等待依赖的 POD 正常启动
```

#### 3.3.12 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: bidi-mtls-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: bidi-mtls-middle
    mtls:
      issuer: osm
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### 3.3.13 测试指令

流量路径: 

Client --**mtls**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n bidi-mtls-client -l app=client -o jsonpath='{.items..metadata.name}')" -n bidi-mtls-client -- curl -ksi https://middle.bidi-mtls-middle:8080/time --cacert /certs/ca.crt --key /certs/tls.key --cert /certs/tls.crt
```

#### 3.3.14 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
date: Tue, 11 Oct 2022 13:56:26 GMT
content-length: 74
content-type: text/plain; charset=utf-8
osm-stats-namespace: bidi-mtls-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-rwlr8

The current time: 2022-10-11 13:56:26.616686218 +0000 UTC m=+16.808331102
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableAccessCertPolicy":false}}}'  --type=merge

kubectl delete AccessCert -n bidi-mtls-middle client-mtls-cert
kubectl delete AccessControl -n bidi-mtls-middle client2middle
kubectl delete AccessCert -n bidi-mtls-middle server-mtls-cert
kubectl delete Egress -n bidi-mtls-middle server-8443
```

