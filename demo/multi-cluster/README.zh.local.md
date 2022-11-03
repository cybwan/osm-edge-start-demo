# OSM Edge 多集群测试

## 1. 下载并安装 osm-edge 命令行工具

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.3.0-alpha.1
curl -L https://github.com/cybwan/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

## 2. 下载并安装 kubecm 命令行工具

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v0.21.0
curl -L https://github.com/sunny0826/kubecm/releases/download/${release}/kubecm_${release}_${system}_${arch}.tar.gz | tar -vxzf -
cp ./kubecm/kubecm /usr/local/bin/
```

## 3. 部署多集群环境

### 3.1 部署控制平面集群和两个业务集群

```bash
curl -o kind-with-registry.sh https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/scripts/kind-with-registry.sh
chmod u+x kind-with-registry.sh

#调整为你的 Host 的 IP 地址
MY_HOST_IP=192.168.127.91
export API_SERVER_ADDR=${MY_HOST_IP}

KIND_CLUSTER_NAME=control-plane MAPPING_HOST_PORT=8090 API_SERVER_PORT=6445 ./kind-with-registry.sh
KIND_CLUSTER_NAME=cluster1 MAPPING_HOST_PORT=8091 API_SERVER_PORT=6446 ./kind-with-registry.sh
KIND_CLUSTER_NAME=cluster2 MAPPING_HOST_PORT=8092 API_SERVER_PORT=6447 ./kind-with-registry.sh
```

### 3.2 编译 FSM 控制平面组件

```bash
git clone -b feature/service-export-n-import https://github.com/flomesh-io/fsm.git
cd fsm
make dev
```

### 3.3 集群 control-plane 部署 FSM 控制平面

```bash
kubecm switch kind-control-plane
helm install --namespace flomesh --create-namespace --set fsm.version=0.2.0-alpha.1-dev --set fsm.logLevel=5 --set fsm.serviceLB.enabled=true fsm charts/fsm/

#等待 FSM 相关组件启动完成
kubectl get pods -A -o wide
```

### 3.4 集群 cluster1 部署 FSM 控制平面

```bash
kubecm switch kind-cluster1
helm install --namespace flomesh --create-namespace --set fsm.version=0.2.0-alpha.1-dev --set fsm.logLevel=5 --set fsm.serviceLB.enabled=true fsm charts/fsm/

#等待 FSM 相关组件启动完成
kubectl get pods -A -o wide
```

### 3.5 集群 cluster2 部署 FSM 控制平面

```bash
kubecm switch kind-cluster2
helm install --namespace flomesh --create-namespace --set fsm.version=0.2.0-alpha.1-dev --set fsm.logLevel=5 --set fsm.serviceLB.enabled=true fsm charts/fsm/

#等待 FSM 相关组件启动完成
kubectl get pods -A -o wide
```

### 3.6 集群 cluster1 加入集群 control-plane FSM 控制平面纳管

```bash
kubecm switch kind-control-plane
cat <<EOF | kubectl apply -f -
apiVersion: flomesh.io/v1alpha1
kind: Cluster
metadata:
  name: cluster1
spec:
  gatewayHost: ${API_SERVER_ADDR}
  gatewayPort: 8091
  kubeconfig: |+
`kind get kubeconfig --name cluster1 | sed 's/^/    /g'`
EOF
```

### 3.7 集群 cluster2 加入集群 control-plane FSM 控制平面纳管

```bash
kubecm switch kind-control-plane
cat <<EOF | kubectl apply -f -
apiVersion: flomesh.io/v1alpha1
kind: Cluster
metadata:
  name: cluster2
spec:
  gatewayHost: ${API_SERVER_ADDR}
  gatewayPort: 8092
  kubeconfig: |+
`kind get kubeconfig --name cluster2 | sed 's/^/    /g'`
EOF
```

## 4. 安装 osm-edge

### 4.1 集群 Cluster1 安装 osm-edge

```bash
kubecm switch kind-cluster1
export osm_namespace=osm-system
export osm_mesh_name=osm
dns_svc_ip="$(kubectl get svc -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[0].spec.clusterIP}')"
osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.certificateProvider.kind=tresor \
    --set=osm.image.registry=localhost:5000/flomesh \
    --set=osm.image.tag=latest \
    --set=osm.image.pullPolicy=Always \
    --set=osm.sidecarLogLevel=error \
    --set=osm.controllerLogLevel=warn \
    --timeout=900s \
    --set=osm.localDNSProxy.enable=true \
    --set=osm.localDNSProxy.primaryUpstreamDNSServerIPAddr="${dns_svc_ip}"
```

### 4.2 集群 Cluster2 安装 osm-edge

```bash
kubecm switch kind-cluster2
export osm_namespace=osm-system
export osm_mesh_name=osm
dns_svc_ip="$(kubectl get svc -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[0].spec.clusterIP}')"
osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.certificateProvider.kind=tresor \
    --set=osm.image.registry=localhost:5000/flomesh \
    --set=osm.image.tag=latest \
    --set=osm.image.pullPolicy=Always \
    --set=osm.sidecarLogLevel=error \
    --set=osm.controllerLogLevel=warn \
    --timeout=900s \
    --set=osm.localDNSProxy.enable=true \
    --set=osm.localDNSProxy.primaryUpstreamDNSServerIPAddr="${dns_svc_ip}"
```

## 5. 多集群测试

### 5.1 部署模拟业务服务

#### 5.1.1 集群 cluster1 部署不被 osm edge 纳管的业务服务

```bash
kubecm switch kind-cluster1
kubectl create namespace pipy
cat <<EOF | kubectl apply -n pipy -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipy-ok
  labels:
    app: pipy-ok
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy-ok
  template:
    metadata:
      labels:
        app: pipy-ok
    spec:
      containers:
        - name: pipy-ok
          image: flomesh/pipy:0.50.0-146
          ports:
            - name: pipy
              containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am from Cluster1 !'))
---
apiVersion: v1
kind: Service
metadata:
  name: pipy-ok
spec:
  ports:
    - name: pipy
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy-ok
EOF

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n pipy -l app=pipy-ok --timeout=180s
```

#### 5.1.2 集群 cluster1 部署被 osm edge 纳管的业务服务

```bash
kubecm switch kind-cluster1
kubectl create namespace pipy-osm
osm namespace add pipy-osm
cat <<EOF | kubectl apply -n pipy-osm -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipy-ok
  labels:
    app: pipy-ok
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy-ok
  template:
    metadata:
      labels:
        app: pipy-ok
    spec:
      containers:
        - name: pipy-ok
          image: flomesh/pipy:0.50.0-146
          ports:
            - name: pipy
              containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am from Cluster1 and controlled by OSM !'))
---
apiVersion: v1
kind: Service
metadata:
  name: pipy-ok
spec:
  ports:
    - name: pipy
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy-ok
EOF

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n pipy-osm -l app=pipy-ok --timeout=180s
```

#### 5.1.3 集群 cluster2 部署被 osm edge 纳管的客户端服务

```bash
kubecm switch kind-cluster2
kubectl create namespace curl
osm namespace add curl
kubectl apply -n curl -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/demo/multi-cluster/curl.yaml

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=180s
```

#### 5.1.4 集群 cluster1 导出业务服务

```bash
kubecm switch kind-cluster1

#kubectl delete serviceexports.flomesh.io -n pipy pipy-ok
cat <<EOF | kubectl apply -f -
apiVersion: flomesh.io/v1alpha1
kind: ServiceExport
metadata:
  namespace: pipy
  name: pipy-ok
spec:
  rules:
    - portNumber: 8080
      path: "/ok"
      pathType: Prefix
EOF

#kubectl delete serviceexports.flomesh.io -n pipy-osm pipy-ok
cat <<EOF | kubectl apply -f -
apiVersion: flomesh.io/v1alpha1
kind: ServiceExport
metadata:
  namespace: pipy-osm
  name: pipy-ok
spec:
  rules:
    - portNumber: 8080
      path: "/ok-osm"
      pathType: Prefix
EOF

kubectl get serviceexports.flomesh.io -A
```

#### 3.1.5 集群 cluster2 导入业务服务

```bash
kubecm switch kind-cluster2

kubectl create namespace pipy
kubectl create namespace pipy-osm
osm namespace add pipy-osm

#创建完 Namespace, 补偿创建ServiceImporI,有延迟,需等待
kubectl get serviceimports.flomesh.io -A
kubectl get serviceimports.flomesh.io -n pipy pipy-ok -o yaml
kubectl get serviceimports.flomesh.io -n pipy-osm pipy-ok -o yaml

curl http://$API_SERVER_ADDR:8091/mesh/repo/default/default/default/local/services/config/registry.json | jq
curl -si http://$API_SERVER_ADDR:8091/ok
curl -si http://$API_SERVER_ADDR:8091/ok-osm/
```

### 5.2 场景测试一：导入集群不存在同质服务

#### 5.2.1 测试指令

```bash
kubecm switch kind-cluster2
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "${curl_client}" -n curl -c curl -- curl -si http://pipy-ok.pipy:8080/
```

#### 5.2.2 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
server: pipy
x-pipy-upstream-service-time: 3
content-length: 24
connection: keep-alive

Hi, I am from Cluster1 !
```

#### 5.2.3 测试指令

```bash
kubecm switch kind-cluster2
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "${curl_client}" -n curl -c curl -- curl -si http://pipy-ok.pipy-osm:8080/
```

#### 5.2.4 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
server: pipy
x-pipy-upstream-service-time: 3
content-length: 46
connection: keep-alive

Hi, I am from Cluster1 and controlled by OSM !
```

### 5.3 场景测试二：导入集群存在同质无 SA 服务

#### 5.3.1 部署无SA业务服务

```bash
kubecm switch kind-cluster2
osm namespace add pipy
cat <<EOF | kubectl apply -n pipy -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipy-ok
  labels:
    app: pipy-ok
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy-ok
  template:
    metadata:
      labels:
        app: pipy-ok
    spec:
      containers:
        - name: pipy-ok
          image: flomesh/pipy:0.50.0-146
          ports:
            - name: pipy
              containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am from Cluster2 !'))
---
apiVersion: v1
kind: Service
metadata:
  name: pipy-ok
spec:
  ports:
    - name: pipy
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy-ok
EOF
```

#### 5.3.2 测试指令

连续执行两次:

```bash
kubecm switch kind-cluster2
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "${curl_client}" -n curl -c curl -- curl -si http://pipy-ok.pipy:8080/
```

#### 5.3.3 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
server: pipy
x-pipy-upstream-service-time: 3
content-length: 24
connection: keep-alive

Hi, I am from Cluster1 !

HTTP/1.1 200 OK
server: pipy
x-pipy-upstream-service-time: 6
content-length: 24
connection: keep-alive

Hi, I am from Cluster2 !
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubecm switch kind-cluster2
kubectl delete deployments -n pipy pipy-ok
kubectl delete service -n pipy pipy-ok
```

### 5.4 场景测试三：导入集群存在同质有 SA 服务

#### 5.4.1 部署有SA业务服务

```bash
kubecm switch kind-cluster2
osm namespace add pipy
cat <<EOF | kubectl apply -n pipy -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipy
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipy-ok
  labels:
    app: pipy-ok
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy-ok
  template:
    metadata:
      labels:
        app: pipy-ok
    spec:
      serviceAccountName: pipy
      containers:
        - name: pipy-ok
          image: flomesh/pipy:0.50.0-146
          ports:
            - name: pipy
              containerPort: 8080
          command:
            - pipy
            - -e
            - |
              pipy()
              .listen(8080)
              .serveHTTP(new Message('Hi, I am from Cluster2 !'))
---
apiVersion: v1
kind: Service
metadata:
  name: pipy-ok
spec:
  ports:
    - name: pipy
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: pipy-ok
EOF
```

#### 5.4.2 测试指令

连续执行两次:

```bash
kubecm switch kind-cluster2
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "${curl_client}" -n curl -c curl -- curl -si http://pipy-ok.pipy:8080/
```

#### 5.4.3 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
server: pipy
x-pipy-upstream-service-time: 4
content-length: 24
connection: keep-alive

Hi, I am from Cluster2 !

HTTP/1.1 200 OK
server: pipy
x-pipy-upstream-service-time: 3
content-length: 24
connection: keep-alive

Hi, I am from Cluster1 !
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubecm switch kind-cluster2
kubectl delete deployments -n pipy pipy-ok
kubectl delete service -n pipy pipy-ok
kubectl delete serviceaccount -n pipy pipy
```

## 6. 多集群卸载

```
kind delete cluster --name control-plane
kind delete cluster --name cluster1
kind delete cluster --name cluster2
```
