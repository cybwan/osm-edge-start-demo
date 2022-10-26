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

## 3. 安装 Nginx Ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo ingress-nginx

kubectl create namespace ingress-nginx
helm install -n ingress-nginx ingress-nginx ingress-nginx/ingress-nginx --version=4.0.18 -f - <<EOF
{
  "controller": {
    "hostPort": {
      "enabled": true,
		},
		"service":  {
		  "type": "NodePort",
		},
	},
}
EOF

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=ingress-nginx \
  --timeout=180s
```

## 4. 双臂 mTLS 测试

### 4.1 技术概念

<img src="https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-nginx/Bidirectional_mTLS.png" alt="Bidirectional_mTLS" style="zoom:80%;" />

### 4.2 部署业务 POD

```bash
#模拟时间服务
kubectl create namespace egress-server
kubectl apply -n egress-server -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-nginx/server.yaml

#模拟中间件服务
kubectl create namespace egress-middle
osm namespace add egress-middle
kubectl apply -n egress-middle -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-nginx/middle.yaml

#模拟外部客户端
kubectl create namespace egress-client
kubectl apply -n egress-client -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/bidirection-mtls-nginx/client.yaml

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n egress-server -l app=server --timeout=180s
kubectl wait --for=condition=ready pod -n egress-middle -l app=middle --timeout=180s
kubectl wait --for=condition=ready pod -n egress-client -l app=client --timeout=180s
```

### 4.3 场景测试一：HTTP Nginx & HTTP Ingress & mTLS Egress

#### 4.3.1 测试指令

流量路径: 

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### 4.3.2 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:36:48 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### 4.3.3 设置 Ingress 策略

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
EOF
```

#### 4.3.4 设置 IngressBackend 策略

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: http
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
EOF
```

#### 4.3.5 测试指令

流量路径: 

Client --**http**--> Nginx Ingress --**http** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### 4.3.6 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:37:18 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 13
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

hello world.
```

#### 4.3.7 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 4.3.8 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 4.3.9 创建Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### 4.3.10 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### 4.3.11 测试指令

流量路径: 

Client --**http**--> Nginx Ingress --**http**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/time
```

#### 4.3.12 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:38:03 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 74
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

The current time: 2022-10-09 08:38:03.03610171 +0000 UTC m=+107.351638075
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete ingress -n egress-middle egress-middle
kubectl delete ingressbackend -n egress-middle egress-middle
kubectl delete egress -n egress-middle server-8443
kubectl delete secrets -n osm-system egress-middle-cert
```

### 4.4 场景测试二：HTTP Nginx & mTLS Ingress & mTLS Egress

#### 4.4.1 测试指令

流量路径: 

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### 4.4.2 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:38:27 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### 4.4.3 设置 Ingress Controller 证书上下文

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":{"secret":{"name":"ingress-controller-cert","namespace":"osm-system"},"subjectAltNames":["ingress-nginx.ingress-nginx.cluster.local"],"validityDuration":"24h"}}}}' --type=merge
```

#### 4.4.4 设置 Ingress 策略

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "middle.egress-middle.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/ingress-controller-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
EOF
```

#### 4.4.5 设置 IngressBackend 策略

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

#### 4.4.6 测试指令

流量路径: 

Client --**http**--> Nginx Ingress --**mtls** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### 4.4.7 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:39:02 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 13
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

hello world.
```

#### 4.4.8 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 4.4.9 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 4.4.10 创建Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### 4.4.11 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### 4.4.12 测试指令

流量路径: 

Client --**http**--> Nginx Ingress --**mtls**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/time
```

#### 4.4.13 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 200 OK
Date: Sun, 09 Oct 2022 08:39:42 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 75
Connection: keep-alive
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc

The current time: 2022-10-09 08:39:42.336230168 +0000 UTC m=+206.716581170
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":null}}}' --type=merge

kubectl delete ingress -n egress-middle egress-middle
kubectl delete ingressbackend -n egress-middle egress-middle
kubectl delete egress -n egress-middle server-8443
kubectl delete secrets -n osm-system egress-middle-cert
```

### 4.5 场景测试三：TLS Nginx & mTLS Ingress & mTLS Egress

#### 4.5.1 测试指令

流量路径: 

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### 4.5.2 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:40:00 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### 4.5.3 设置 Ingress Controller 证书上下文

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":{"secret":{"name":"ingress-controller-cert","namespace":"osm-system"},"subjectAltNames":["ingress-nginx.ingress-nginx.cluster.local"],"validityDuration":"24h"}}}}' --type=merge
```

#### 4.5.4 创建 Nginx TLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/nginx.crt -o nginx.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/nginx.key -o nginx.key

kubectl create secret tls -n egress-middle nginx-cert-secret \
  --cert=./nginx.crt \
  --key=./nginx.key 
```

#### 4.5.5 设置 Ingress 策略

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "middle.egress-middle.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/ingress-controller-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
  tls:
  - hosts:
    - ingress-nginx-controller.ingress-nginx
    secretName: nginx-cert-secret
EOF
```

#### 4.5.6 设置 IngressBackend 策略

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

#### 4.5.7 测试指令

流量路径: 

Client --**tls**--> Nginx Ingress --**mtls** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/hello --key /certs/client.key --cert /certs/client.crt
```

#### 4.5.8 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:40:40 GMT
content-type: text/plain; charset=utf-8
content-length: 13
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

hello world.
```

#### 4.5.9 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 4.5.10 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 4.5.11 创建Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### 4.5.12 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### 4.5.13 测试指令

流量路径: 

Client --**tls**--> Nginx Ingress --**mtls**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/time --key /certs/client.key --cert /certs/client.crt
```

#### 4.5.14 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:41:21 GMT
content-type: text/plain; charset=utf-8
content-length: 75
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

The current time: 2022-10-09 08:41:21.121578846 +0000 UTC m=+305.588353001
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":null}}}' --type=merge

kubectl delete ingress -n egress-middle egress-middle
kubectl delete ingressbackend -n egress-middle egress-middle
kubectl delete egress -n egress-middle server-8443
kubectl delete secrets -n osm-system egress-middle-cert
kubectl delete secrets -n egress-middle nginx-cert-secret
```

### 4.6 场景测试四：mTLS Nginx & mTLS Ingress & mTLS Egress

#### 4.6.1 测试指令

流量路径: 

Client --**http**--> Nginx Ingress Controller

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -si http://ingress-nginx-controller.ingress-nginx/hello
```

#### 4.6.2 测试结果

正确返回结果类似于:

```bash
HTTP/1.1 404 Not Found
Date: Sun, 09 Oct 2022 08:41:52 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

#### 4.6.3 设置 Ingress Controller 证书上下文

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":{"secret":{"name":"ingress-controller-cert","namespace":"osm-system"},"subjectAltNames":["ingress-nginx.ingress-nginx.cluster.local"],"validityDuration":"24h"}}}}' --type=merge
```

#### 4.6.4 创建 Nginx CA Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/ca.crt -o ca.crt

kubectl create secret generic -n egress-middle nginx-ca-secret \
  --from-file=ca.crt=./ca.crt 
```

#### 4.6.5 创建 Nginx TLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/nginx.crt -o nginx.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/nginx.key -o nginx.key

kubectl create secret tls -n egress-middle nginx-cert-secret \
  --cert=./nginx.crt \
  --key=./nginx.key 
```

#### 4.6.6 设置 Ingress 策略

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: egress-middle
  namespace: egress-middle
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "false"
    nginx.ingress.kubernetes.io/auth-tls-secret: egress-middle/nginx-ca-secret
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # proxy_ssl_name for a service is of the form <service-account>.<namespace>.cluster.local
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_ssl_name "middle.egress-middle.cluster.local";
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "osm-system/ingress-controller-cert"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: middle
            port:
              number: 8080
  tls:
  - hosts:
    - ingress-nginx-controller.ingress-nginx
    secretName: nginx-cert-secret
EOF
```

#### 4.6.7 设置 IngressBackend 策略

```bash
kubectl apply -f - <<EOF
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: egress-middle
  namespace: egress-middle
spec:
  backends:
  - name: middle
    port:
      number: 8080 # targetPort of middle service
      protocol: https
    tls:
      skipClientCertValidation: false
  sources:
  - kind: Service
    namespace: ingress-nginx
    name: ingress-nginx-controller
  - kind: AuthenticatedPrincipal
    name: ingress-nginx.ingress-nginx.cluster.local
EOF
```

#### 4.6.8 测试指令

流量路径: 

Client --**mtls**--> Nginx Ingress --**mtls** --> sidecar --> Middle

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/hello  --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### 4.6.9 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:42:37 GMT
content-type: text/plain; charset=utf-8
content-length: 13
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

hello world.
```

#### 4.6.10 禁用Egress目的宽松模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enableEgress":false}}}' --type=merge
```

#### 4.6.11 启用Egress目的策略模式

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"featureFlags":{"enableEgressPolicy":true}}}'  --type=merge
```

#### 4.6.12 创建Egress mTLS Secret

```bash
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/ca.crt -o ca.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.crt -o middle.crt
curl https://raw.githubusercontent.com/cybwan/mtls-time-demo/main/certs/middle.key -o middle.key

kubectl create secret generic -n osm-system egress-middle-cert \
  --from-file=ca.crt=./ca.crt \
  --from-file=tls.crt=./middle.crt \
  --from-file=tls.key=./middle.key 
```

#### 4.6.13 设置Egress目的策略

```bash
kubectl apply -f - <<EOF
kind: Egress
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: server-8443
  namespace: egress-middle
spec:
  sources:
  - kind: ServiceAccount
    name: middle
    namespace: egress-middle
    mtls:
      issuer: other
      cert:
        sn: 1
        expiration: 2030-1-1 00:00:00
        secret:
          name: egress-middle-cert
          namespace: osm-system
  hosts:
  - server.egress-server.svc.cluster.local
  ports:
  - number: 8443
    protocol: http
EOF
```

#### 4.6.14 测试指令

流量路径: 

Client --**mtls**--> Nginx Ingress --**mtls**--> sidecar --> Middle --> sidecar --**egress mtls**--> Server

```bash
kubectl exec "$(kubectl get pod -n egress-client -l app=client -o jsonpath='{.items..metadata.name}')" -n egress-client -- curl -ksi https://ingress-nginx-controller.ingress-nginx/time --cacert /certs/ca.crt --key /certs/client.key --cert /certs/client.crt
```

#### 4.6.15 测试结果

正确返回结果类似于:

```bash
HTTP/2 200 
date: Sun, 09 Oct 2022 08:43:13 GMT
content-type: text/plain; charset=utf-8
content-length: 75
osm-stats-namespace: egress-middle
osm-stats-kind: Deployment
osm-stats-name: middle
osm-stats-pod: middle-5fc9f7b8b5-7txmc
strict-transport-security: max-age=15724800; includeSubDomains

The current time: 2022-10-09 08:43:13.241006879 +0000 UTC m=+417.741653620
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
export osm_namespace=osm-system
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"certificate":{"ingressGateway":null}}}' --type=merge

kubectl delete ingress -n egress-middle egress-middle
kubectl delete ingressbackend -n egress-middle egress-middle
kubectl delete egress -n egress-middle server-8443
kubectl delete secrets -n osm-system egress-middle-cert
kubectl delete secrets -n egress-middle nginx-cert-secret
kubectl delete secrets -n egress-middle nginx-ca-secret
```

