

# OSM Edge 限速测试

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
    --set=osm.enableEgress=false \
    --set=osm.sidecarLogLevel=debug \
    --set=osm.controllerLogLevel=warn \
    --timeout=900s
```

## 3. 限速策略测试

### 3.1 技术概念

在 OSM Edge 中，支持本地限速：

- 4层TCP限速：
  - 触发条件
    - 统计时间窗口单位：
      - 秒
      - 分
      - 小时
    - 统计时间窗口内连接的数量
    - 统计时间窗口内连接的波动峰值
- 7层HTTP限速：
  - 限速粒度
    - 虚拟主机层级
    - 请求路径层级
    - 请求头层级
  
  - 触发条件
    - 统计时间窗口内请求的数量
    - 统计时间窗口内请求的波动峰值
  
  - 限速响应
    - 回写状态码：状态码取值范围 400~599，缺省 429
    - 回写响应头
  

### 3.2 部署业务 POD

```bash
#设置流量宽松模式
export osm_namespace=osm-system 
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge

#模拟业务服务
kubectl create namespace ratelimit
osm namespace add ratelimit
kubectl apply -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/rate-limit/fortio.yaml -n ratelimit

#模拟客户端
kubectl apply -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/rate-limit/fortio-client.yaml -n ratelimit
kubectl apply -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/osm-edge-v1.2.0/demo/rate-limit/curl.yaml -n ratelimit

#等待依赖的 POD 正常启动
kubectl wait --for=condition=ready pod -n ratelimit -l app=fortio --timeout=180s
kubectl wait --for=condition=ready pod -n ratelimit -l app=fortio-client --timeout=180s
kubectl wait --for=condition=ready pod -n ratelimit -l app=curl --timeout=180s
```

## 4 场景测试一：4层TCP限速

### 4.1 无限速设置，100%通过率

##### 3.3.1.1 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -qps -1 -c 3 -n 10 tcp://fortio.ratelimit.svc.cluster.local:8078
```

##### 3.3.1.2 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at -1 queries per second, 8->8 procs, for 10 calls: tcp://fortio.ratelimit.svc.cluster.local:8078
09:57:10 I tcprunner.go:238> Starting tcp test for tcp://fortio.ratelimit.svc.cluster.local:8078 with 3 threads at -1.0 qps
Starting at max qps with 3 thread(s) [gomax 8] for exactly 10 calls (3 per thread + 1)
09:57:10 I periodic.go:721> T002 ended after 16.205043ms : 3 calls. qps=185.127555662765
09:57:10 I periodic.go:721> T001 ended after 16.501644ms : 3 calls. qps=181.80006792050537
09:57:10 I periodic.go:721> T000 ended after 17.322609ms : 4 calls. qps=230.91209874909723
Ended after 17.396684ms : 10 calls. qps=574.82
Aggregated Function Time : count 10 avg 0.0049640116 +/- 0.006207 min 0.000433138 max 0.015211916 sum 0.049640116
# range, mid point, percentile, count
>= 0.000433138 <= 0.001 , 0.000716569 , 60.00, 6
> 0.002 <= 0.003 , 0.0025 , 70.00, 1
> 0.012 <= 0.014 , 0.013 , 80.00, 1
> 0.014 <= 0.0152119 , 0.014606 , 100.00, 2
# target 50% 0.000886628
# target 75% 0.013
# target 90% 0.014606
# target 99% 0.0151513
# target 99.9% 0.0152059
Error cases : no data
Sockets used: 3 (for perfect no error run, would be 3)
Total Bytes sent: 240, received: 240
tcp OK : 10 (100.0 %)
All done 10 calls (plus 0 warmup) 4.964 ms avg, 574.8 qps
```

正如上面的测试结果所示，所有的请求都成功执行

```bash
Total Bytes sent: 240, received: 240
tcp OK : 10 (100.0 %)
All done 10 calls (plus 0 warmup) 4.964 ms avg, 574.8 qps
```

### 4.2 每分钟 1 个连接，30%通过率

##### 3.3.2.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: tcp-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  rateLimit:
    local:
      tcp:
        connections: 1
        unit: minute
EOF
```

##### 3.3.2.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -qps -1 -c 3 -n 10 tcp://fortio.ratelimit.svc.cluster.local:8078
```

##### 3.3.2.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at -1 queries per second, 8->8 procs, for 10 calls: tcp://fortio.ratelimit.svc.cluster.local:8078
09:59:36 I tcprunner.go:238> Starting tcp test for tcp://fortio.ratelimit.svc.cluster.local:8078 with 3 threads at -1.0 qps
Starting at max qps with 3 thread(s) [gomax 8] for exactly 10 calls (3 per thread + 1)
09:59:36 E tcprunner.go:203> [0] Unable to read: EOF
09:59:36 E tcprunner.go:203> [1] Unable to read: EOF
09:59:36 E tcprunner.go:203> [0] Unable to read: EOF
09:59:36 E tcprunner.go:203> [1] Unable to read: EOF
09:59:36 E tcprunner.go:203> [0] Unable to read: EOF
09:59:36 E tcprunner.go:203> [1] Unable to read: EOF
09:59:36 I periodic.go:721> T001 ended after 11.017611ms : 3 calls. qps=272.29133430105674
09:59:36 E tcprunner.go:203> [0] Unable to read: EOF
09:59:36 I periodic.go:721> T000 ended after 11.174798ms : 4 calls. qps=357.94830474788
09:59:36 I periodic.go:721> T002 ended after 11.636863ms : 3 calls. qps=257.80143669303317
Ended after 11.745703ms : 10 calls. qps=851.38
Aggregated Function Time : count 10 avg 0.0033618082 +/- 0.001662 min 0.001739169 max 0.007397066 sum 0.033618082
# range, mid point, percentile, count
>= 0.00173917 <= 0.002 , 0.00186958 , 10.00, 1
> 0.002 <= 0.003 , 0.0025 , 60.00, 5
> 0.003 <= 0.004 , 0.0035 , 80.00, 2
> 0.005 <= 0.006 , 0.0055 , 90.00, 1
> 0.007 <= 0.00739707 , 0.00719853 , 100.00, 1
# target 50% 0.0028
# target 75% 0.00375
# target 90% 0.006
# target 99% 0.00735736
# target 99.9% 0.0073931
Error cases : count 7 avg 0.0031465759 +/- 0.001039 min 0.002091518 max 0.00551779 sum 0.022026031
# range, mid point, percentile, count
>= 0.00209152 <= 0.003 , 0.00254576 , 57.14, 4
> 0.003 <= 0.004 , 0.0035 , 85.71, 2
> 0.005 <= 0.00551779 , 0.00525889 , 100.00, 1
# target 50% 0.00284859
# target 75% 0.003625
# target 90% 0.00515534
# target 99% 0.00548154
# target 99.9% 0.00551417
Sockets used: 8 (for perfect no error run, would be 3)
Total Bytes sent: 240, received: 72
tcp OK : 3 (30.0 %)
tcp short read : 7 (70.0 %)
All done 10 calls (plus 0 warmup) 3.362 ms avg, 851.4 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Total Bytes sent: 240, received: 72
tcp OK : 3 (30.0 %)
tcp short read : 7 (70.0 %)
All done 10 calls (plus 0 warmup) 3.362 ms avg, 851.4 qps
```

##### 3.3.2.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep fortio_8078_tcp.rate_limited
```

查询结果:

```bash
local_rate_limit.inbound_ratelimit/fortio_8078_tcp.rate_limited: 7
```

### 4.3 每分钟 1 个连接, 波动峰值为 10，100%通过率

##### 3.3.3.1 调整限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: tcp-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  rateLimit:
    local:
      tcp:
        connections: 1
        unit: minute
        burst: 10
EOF
```

##### 3.3.3.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -qps -1 -c 3 -n 10 tcp://fortio.ratelimit.svc.cluster.local:8078
```

##### 3.3.3.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at -1 queries per second, 8->8 procs, for 10 calls: tcp://fortio.ratelimit.svc.cluster.local:8078
10:22:14 I tcprunner.go:238> Starting tcp test for tcp://fortio.ratelimit.svc.cluster.local:8078 with 3 threads at -1.0 qps
Starting at max qps with 3 thread(s) [gomax 8] for exactly 10 calls (3 per thread + 1)
10:22:14 I periodic.go:721> T002 ended after 10.275656ms : 3 calls. qps=291.952163443385
10:22:14 I periodic.go:721> T001 ended after 11.19298ms : 3 calls. qps=268.02513718419937
10:22:14 I periodic.go:721> T000 ended after 11.251359ms : 4 calls. qps=355.512609632312
Ended after 11.298995ms : 10 calls. qps=885.03
Aggregated Function Time : count 10 avg 0.0032481685 +/- 0.003836 min 0.000587106 max 0.00943663 sum 0.032481685
# range, mid point, percentile, count
>= 0.000587106 <= 0.001 , 0.000793553 , 70.00, 7
> 0.008 <= 0.009 , 0.0085 , 90.00, 2
> 0.009 <= 0.00943663 , 0.00921832 , 100.00, 1
# target 50% 0.000862369
# target 75% 0.00825
# target 90% 0.009
# target 99% 0.00939297
# target 99.9% 0.00943226
Error cases : no data
Sockets used: 3 (for perfect no error run, would be 3)
Total Bytes sent: 240, received: 240
tcp OK : 10 (100.0 %)
All done 10 calls (plus 0 warmup) 3.248 ms avg, 885.0 qps
```

正如上面的测试结果所示，所有的请求都成功执行

```bash
Total Bytes sent: 240, received: 0
tcp short read : 10 (100.0 %)
All done 10 calls (plus 0 warmup) 2.364 ms avg, 1137.8 qps
```

##### 3.3.3.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep fortio_8078_tcp.rate_limited
```

查询结果:

```bash
local_rate_limit.inbound_ratelimit/fortio_8078_tcp.rate_limited: 10
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete upstreamtrafficsettings -n ratelimit tcp-rate-limit
```

## 5 场景测试二：7层HTTP限速

### 5.1 无限速设置，100%通过率

#### 5.1.1 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080
```

#### 5.1.2 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
06:05:00 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
06:05:01 I periodic.go:721> T002 ended after 1.128258011s : 3 calls. qps=2.6589662743374043
06:05:01 I periodic.go:721> T001 ended after 1.128512582s : 3 calls. qps=2.6583664620586394
06:05:01 I periodic.go:721> T000 ended after 1.503203697s : 4 calls. qps=2.660983343763024
Ended after 1.503353227s : 10 calls. qps=6.6518
Sleep times : count 7 avg 0.5325257 +/- 0.03084 min 0.496371521 max 0.559626374 sum 3.72767993
Aggregated Function Time : count 10 avg 0.0027281626 +/- 0.0006551 min 0.001730995 max 0.003678273 sum 0.027281626
# range, mid point, percentile, count
>= 0.00173099 <= 0.002 , 0.0018655 , 10.00, 1
> 0.002 <= 0.003 , 0.0025 , 60.00, 5
> 0.003 <= 0.00367827 , 0.00333914 , 100.00, 4
# target 50% 0.0028
# target 75% 0.00325435
# target 90% 0.0035087
# target 99% 0.00366132
# target 99.9% 0.00367658
Error cases : no data
06:05:01 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.15.74:8080
06:05:01 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.15.74:8080
06:05:01 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.15.74:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.15.74:8080: 3
Code 200 : 10 (100.0 %)
Response Header Sizes : count 10 avg 62 +/- 0 min 62 max 62 sum 620
Response Body/Total Sizes : count 10 avg 62 +/- 0 min 62 max 62 sum 620
All done 10 calls (plus 0 warmup) 2.728 ms avg, 6.7 qps
```

正如上面的测试结果所示，所有的请求都成功执行

```bash
Code 200 : 10 (100.0 %)
```

### 5.2 虚拟主机层级限速

#### 5.2.1 每分钟 3 个请求，30%通过率

##### 5.2.1.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  rateLimit:
    local:
      http:
        requests: 3
        unit: minute
EOF
```

##### 5.2.1.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.2.1.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
10:30:21 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T001 ended after 1.129974506s : 3 calls. qps=2.654927154613168
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T002 ended after 1.131041289s : 3 calls. qps=2.652423062868397
10:30:23 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:23 I periodic.go:721> T000 ended after 1.503440771s : 4 calls. qps=2.660563739627359
Ended after 1.503678339s : 10 calls. qps=6.6504
Sleep times : count 7 avg 0.52937663 +/- 0.03061 min 0.488825155 max 0.560046385 sum 3.70563638
Aggregated Function Time : count 10 avg 0.0052598876 +/- 0.003915 min 0.00164245 max 0.011292554 sum 0.052598876
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 20.00, 2
> 0.002 <= 0.003 , 0.0025 , 50.00, 3
> 0.003 <= 0.004 , 0.0035 , 60.00, 1
> 0.004 <= 0.005 , 0.0045 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 80.00, 1
> 0.011 <= 0.0112926 , 0.0111463 , 100.00, 2
# target 50% 0.003
# target 75% 0.0105
# target 90% 0.0111463
# target 99% 0.0112779
# target 99.9% 0.0112911
Error cases : count 7 avg 0.0027715733 +/- 0.001114 min 0.00164245 max 0.004884773 sum 0.019401013
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 28.57, 2
> 0.002 <= 0.003 , 0.0025 , 71.43, 3
> 0.003 <= 0.004 , 0.0035 , 85.71, 1
> 0.004 <= 0.00488477 , 0.00444239 , 100.00, 1
# target 50% 0.0025
# target 75% 0.00325
# target 90% 0.00426543
# target 99% 0.00482284
# target 99.9% 0.00487858
10:30:23 I httprunner.go:197> [0]   3 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [1]   2 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [2]   2 socket used, resolved to 10.96.56.21:8080
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.56.21:8080: 3
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 86.4 +/- 8.249 min 81 max 99 sum 864
All done 10 calls (plus 0 warmup) 5.260 ms avg, 6.7 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
```

##### 5.2.1.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

#### 5.2.2 每分钟 3 个请求, 波动峰值为 10，100%通过率

##### 5.2.2.1 调整限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  rateLimit:
    local:
      http:
        requests: 3
        unit: minute
        burst: 10
EOF
```

##### 5.2.2.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.2.2.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
01:15:26 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
01:15:27 I periodic.go:721> T001 ended after 1.128083378s : 3 calls. qps=2.6593778957356466
01:15:27 I periodic.go:721> T002 ended after 1.128407363s : 3 calls. qps=2.65861434298351
01:15:28 I periodic.go:721> T000 ended after 1.504108753s : 4 calls. qps=2.6593821703529437
Ended after 1.504282705s : 10 calls. qps=6.6477
Sleep times : count 7 avg 0.52865926 +/- 0.03053 min 0.488882417 max 0.55887838 sum 3.70061482
Aggregated Function Time : count 10 avg 0.0053517901 +/- 0.003907 min 0.00200691 max 0.011444314 sum 0.053517901
# range, mid point, percentile, count
>= 0.00200691 <= 0.003 , 0.00250345 , 30.00, 3
> 0.003 <= 0.004 , 0.0035 , 70.00, 4
> 0.011 <= 0.0114443 , 0.0112222 , 100.00, 3
# target 50% 0.0035
# target 75% 0.0110741
# target 90% 0.0112962
# target 99% 0.0114295
# target 99.9% 0.0114428
Error cases : no data
01:15:28 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.249.207:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.249.207:8080: 3
Code 200 : 10 (100.0 %)
Response Header Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
Response Body/Total Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
All done 10 calls (plus 0 warmup) 5.352 ms avg, 6.6 qps
```

正如上面的测试结果所示，所有的请求都成功执行

```bash
Code 200 : 10 (100.0 %)
```

##### 5.2.2.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
local_rate_limit.inbound_ratelimit/fortio_8078_tcp.rate_limited: 10
```

#### 5.2.3 每分钟 3 个请求，30%通过率，回写状态码 509

##### 5.2.3.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  rateLimit:
    local:
      http:
        requests: 3
        unit: minute
        responseStatusCode: 509
        responseHeadersToAdd:
          - name: hello
            value: world
EOF
```

##### 5.2.3.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.2.3.3 测试结果

返回结果类似于:

```bash
Fortio 1.38.0 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
09:55:52 I httprunner.go:102> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
09:55:53 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
09:55:53 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
09:55:53 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
09:55:53 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
09:55:53 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
09:55:53 I periodic.go:809> T001 ended after 1.130335981s : 3 calls. qps=2.654078124051153
09:55:53 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
09:55:53 I periodic.go:809> T002 ended after 1.13143543s : 3 calls. qps=2.6514990784759145
09:55:54 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
09:55:54 I periodic.go:809> T000 ended after 1.50652638s : 4 calls. qps=2.6551144759907888
Ended after 1.506576576s : 10 calls. qps=6.6376
Sleep times : count 7 avg 0.52863099 +/- 0.03064 min 0.488977925 max 0.55902312 sum 3.70041693
Aggregated Function Time : count 10 avg 0.0061497978 +/- 0.00338 min 0.002247739 max 0.011437815 sum 0.061497978
# range, mid point, percentile, count
>= 0.00224774 <= 0.003 , 0.00262387 , 20.00, 2
> 0.003 <= 0.004 , 0.0035 , 30.00, 1
> 0.004 <= 0.005 , 0.0045 , 50.00, 2
> 0.005 <= 0.006 , 0.0055 , 60.00, 1
> 0.006 <= 0.007 , 0.0065 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 90.00, 2
> 0.011 <= 0.0114378 , 0.0112189 , 100.00, 1
# target 50% 0.005
# target 75% 0.01025
# target 90% 0.011
# target 99% 0.011394
# target 99.9% 0.0114334
Error cases : count 7 avg 0.0040566104 +/- 0.001295 min 0.002247739 max 0.006127756 sum 0.028396273
# range, mid point, percentile, count
>= 0.00224774 <= 0.003 , 0.00262387 , 28.57, 2
> 0.003 <= 0.004 , 0.0035 , 42.86, 1
> 0.004 <= 0.005 , 0.0045 , 71.43, 2
> 0.005 <= 0.006 , 0.0055 , 85.71, 1
> 0.006 <= 0.00612776 , 0.00606388 , 100.00, 1
# target 50% 0.00425
# target 75% 0.00525
# target 90% 0.00603833
# target 99% 0.00611881
# target 99.9% 0.00612686
# Socket and IP used for each connection:
[0]   3 socket used, resolved to [10.96.137.219:8080] connection timing : count 3 avg 0.00032004133 +/- 0.0001848 min 0.000160741 max 0.000579103 sum 0.000960124
[1]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.0003931895 +/- 0.0002296 min 0.000163557 max 0.000622822 sum 0.000786379
[2]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.0004590685 +/- 0.0002689 min 0.000190136 max 0.000728001 sum 0.000918137
Connection time histogram (s) : count 7 avg 0.00038066286 +/- 0.0002318 min 0.000160741 max 0.000728001 sum 0.00266464
# range, mid point, percentile, count
>= 0.000160741 <= 0.000728001 , 0.000444371 , 100.00, 7
# target 50% 0.000397099
# target 75% 0.00056255
# target 90% 0.000661821
# target 99% 0.000721383
# target 99.9% 0.000727339
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.137.219:8080: 7
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 101.1 +/- 1.375 min 99 max 102 sum 1011
All done 10 calls (plus 0 warmup) 6.150 ms avg, 6.6 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
```

##### 5.2.3.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

##### 5.2.3.5 测试指令

多次执行，触发限流

```bash
curl="$(kubectl get pod -n ratelimit -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl" -n ratelimit -c curl -- curl -sI http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.2.3.6 测试结果

返回结果类似于:

```bash
HTTP/1.1 509 Unassigned
hello: world
content-length: 17
connection: keep-alive
```

返回响应头:

```bash
hello: world
```

### 5.3 请求路径层级限速

#### 5.3.1 宽松流量模式

##### 5.3.1.1 设置流量宽松模式

```bash
export osm_namespace=osm-system 
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
```

##### 5.3.1.2 每分钟 3 个请求，30%通过率

###### 5.3.1.2.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpRoutes:
    - path: .*
      rateLimit:
        local:
          requests: 3
          unit: minute
EOF
```

###### 5.3.1.2.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080
```

###### 5.3.1.2.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
10:30:21 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T001 ended after 1.129974506s : 3 calls. qps=2.654927154613168
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T002 ended after 1.131041289s : 3 calls. qps=2.652423062868397
10:30:23 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:23 I periodic.go:721> T000 ended after 1.503440771s : 4 calls. qps=2.660563739627359
Ended after 1.503678339s : 10 calls. qps=6.6504
Sleep times : count 7 avg 0.52937663 +/- 0.03061 min 0.488825155 max 0.560046385 sum 3.70563638
Aggregated Function Time : count 10 avg 0.0052598876 +/- 0.003915 min 0.00164245 max 0.011292554 sum 0.052598876
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 20.00, 2
> 0.002 <= 0.003 , 0.0025 , 50.00, 3
> 0.003 <= 0.004 , 0.0035 , 60.00, 1
> 0.004 <= 0.005 , 0.0045 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 80.00, 1
> 0.011 <= 0.0112926 , 0.0111463 , 100.00, 2
# target 50% 0.003
# target 75% 0.0105
# target 90% 0.0111463
# target 99% 0.0112779
# target 99.9% 0.0112911
Error cases : count 7 avg 0.0027715733 +/- 0.001114 min 0.00164245 max 0.004884773 sum 0.019401013
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 28.57, 2
> 0.002 <= 0.003 , 0.0025 , 71.43, 3
> 0.003 <= 0.004 , 0.0035 , 85.71, 1
> 0.004 <= 0.00488477 , 0.00444239 , 100.00, 1
# target 50% 0.0025
# target 75% 0.00325
# target 90% 0.00426543
# target 99% 0.00482284
# target 99.9% 0.00487858
10:30:23 I httprunner.go:197> [0]   3 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [1]   2 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [2]   2 socket used, resolved to 10.96.56.21:8080
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.56.21:8080: 3
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 86.4 +/- 8.249 min 81 max 99 sum 864
All done 10 calls (plus 0 warmup) 5.260 ms avg, 6.7 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
```

###### 5.3.1.2.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

##### 5.3.1.3 每分钟 3 个请求, 波动峰值为 10，100%通过率

###### 5.3.1.3.1 调整限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpRoutes:
    - path: .*
      rateLimit:
        local:
          requests: 3
          unit: minute
          burst: 10
EOF
```

###### 5.3.1.3.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080
```

###### 5.3.1.3.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
01:15:26 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
01:15:27 I periodic.go:721> T001 ended after 1.128083378s : 3 calls. qps=2.6593778957356466
01:15:27 I periodic.go:721> T002 ended after 1.128407363s : 3 calls. qps=2.65861434298351
01:15:28 I periodic.go:721> T000 ended after 1.504108753s : 4 calls. qps=2.6593821703529437
Ended after 1.504282705s : 10 calls. qps=6.6477
Sleep times : count 7 avg 0.52865926 +/- 0.03053 min 0.488882417 max 0.55887838 sum 3.70061482
Aggregated Function Time : count 10 avg 0.0053517901 +/- 0.003907 min 0.00200691 max 0.011444314 sum 0.053517901
# range, mid point, percentile, count
>= 0.00200691 <= 0.003 , 0.00250345 , 30.00, 3
> 0.003 <= 0.004 , 0.0035 , 70.00, 4
> 0.011 <= 0.0114443 , 0.0112222 , 100.00, 3
# target 50% 0.0035
# target 75% 0.0110741
# target 90% 0.0112962
# target 99% 0.0114295
# target 99.9% 0.0114428
Error cases : no data
01:15:28 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.249.207:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.249.207:8080: 3
Code 200 : 10 (100.0 %)
Response Header Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
Response Body/Total Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
All done 10 calls (plus 0 warmup) 5.352 ms avg, 6.6 qps
```

正如上面的测试结果所示，所有的请求都成功执行

```bash
Code 200 : 10 (100.0 %)
```

###### 5.3.1.3.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
local_rate_limit.inbound_ratelimit/fortio_8078_tcp.rate_limited: 10
```

##### 5.3.1.4 每分钟 4 个请求，30%通过率，回写状态码 509

###### 5.3.1.4.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpRoutes:
    - path: .*
      rateLimit:
        local:
          requests: 3
          unit: minute
          responseStatusCode: 509
          responseHeadersToAdd:
            - name: hello
              value: world
EOF
```

###### 5.3.1.4.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080
```

###### 5.3.1.4.3 测试结果

返回结果类似于:

```bash
10:04:43 I httprunner.go:102> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
10:04:43 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:04:43 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
10:04:43 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
10:04:44 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:04:44 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
10:04:44 I periodic.go:809> T002 ended after 1.131536359s : 3 calls. qps=2.651262574232491
10:04:44 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
10:04:44 I periodic.go:809> T001 ended after 1.131883596s : 3 calls. qps=2.6504492251692637
10:04:44 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:04:44 I periodic.go:809> T000 ended after 1.506504165s : 4 calls. qps=2.655153628466736
Ended after 1.506533766s : 10 calls. qps=6.6378
Sleep times : count 7 avg 0.52863491 +/- 0.03069 min 0.489218406 max 0.559163296 sum 3.70044435
Aggregated Function Time : count 10 avg 0.0063686645 +/- 0.003339 min 0.002259049 max 0.011597923 sum 0.063686645
# range, mid point, percentile, count
>= 0.00225905 <= 0.003 , 0.00262952 , 20.00, 2
> 0.003 <= 0.004 , 0.0035 , 30.00, 1
> 0.004 <= 0.005 , 0.0045 , 40.00, 1
> 0.005 <= 0.006 , 0.0055 , 60.00, 2
> 0.006 <= 0.007 , 0.0065 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 90.00, 2
> 0.011 <= 0.0115979 , 0.011299 , 100.00, 1
# target 50% 0.0055
# target 75% 0.01025
# target 90% 0.011
# target 99% 0.0115381
# target 99.9% 0.0115919
Error cases : count 7 avg 0.004355754 +/- 0.001534 min 0.002259049 max 0.006051199 sum 0.030490278
# range, mid point, percentile, count
>= 0.00225905 <= 0.003 , 0.00262952 , 28.57, 2
> 0.003 <= 0.004 , 0.0035 , 42.86, 1
> 0.004 <= 0.005 , 0.0045 , 57.14, 1
> 0.005 <= 0.006 , 0.0055 , 85.71, 2
> 0.006 <= 0.0060512 , 0.0060256 , 100.00, 1
# target 50% 0.0045
# target 75% 0.005625
# target 90% 0.00601536
# target 99% 0.00604762
# target 99.9% 0.00605084
# Socket and IP used for each connection:
[0]   3 socket used, resolved to [10.96.137.219:8080] connection timing : count 3 avg 0.00024358433 +/- 6.69e-05 min 0.000172271 max 0.000333078 sum 0.000730753
[1]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.0001659035 +/- 2.797e-05 min 0.000137933 max 0.000193874 sum 0.000331807
[2]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.000200063 +/- 7.513e-05 min 0.000124936 max 0.00027519 sum 0.000400126
Connection time histogram (s) : count 7 avg 0.00020895514 +/- 6.943e-05 min 0.000124936 max 0.000333078 sum 0.001462686
# range, mid point, percentile, count
>= 0.000124936 <= 0.000333078 , 0.000229007 , 100.00, 7
# target 50% 0.000211662
# target 75% 0.00027237
# target 90% 0.000308795
# target 99% 0.00033065
# target 99.9% 0.000332835
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.137.219:8080: 7
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 101.1 +/- 1.375 min 99 max 102 sum 1011
All done 10 calls (plus 0 warmup) 6.369 ms avg, 6.6 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
```

###### 5.3.1.4.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

###### 5.3.1.4.5 测试指令

多次执行，触发限流

```bash
curl="$(kubectl get pod -n ratelimit -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl" -n ratelimit -c curl -- curl -sI http://fortio.ratelimit.svc.cluster.local:8080
```

###### 5.3.1.4.6 测试结果

返回结果类似于:

```bash
HTTP/1.1 509 Unassigned
hello: world
content-length: 17
connection: keep-alive
```

返回响应头:

```bash
hello: world
```

#### 5.3.2 非宽松流量模式

##### 5.3.2.1 设置非流量宽松模式

```bash
export osm_namespace=osm-system 
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

##### 5.3.2.2 设置流量策略

```bash
kubectl apply -f - <<EOF
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: fortio-service-routes
  namespace: ratelimit
spec:
  matches:
  - name: debug
    pathRegex: /debug
    methods:
    - GET
  - name: fortio
    pathRegex: /fortio
    methods:
    - GET
EOF

kubectl apply -f - <<EOF
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: curl-client-access-service
  namespace: ratelimit
spec:
  destination:
    kind: ServiceAccount
    name: server
    namespace: ratelimit
  rules:
  - kind: HTTPRouteGroup
    name: fortio-service-routes
    matches:
    - debug
    - fortio
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: ratelimit
EOF

kubectl apply -f - <<EOF
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: fortio-client-access-service
  namespace: ratelimit
spec:
  destination:
    kind: ServiceAccount
    name: server
    namespace: ratelimit
  rules:
  - kind: HTTPRouteGroup
    name: fortio-service-routes
    matches:
    - debug
    - fortio
  sources:
  - kind: ServiceAccount
    name: client
    namespace: ratelimit
EOF

curl_client="$(kubectl get pod -n ratelimit -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl_client" -n ratelimit -c curl -- curl -si http://fortio.ratelimit.svc.cluster.local:8080/debug

curl_client="$(kubectl get pod -n ratelimit -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl_client" -n ratelimit -c curl -- curl -si http://fortio.ratelimit.svc.cluster.local:8080/fortio
```

##### 5.3.2.3 每分钟 3 个请求，30%通过率

###### 5.3.2.3.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpRoutes:
    - path: /debug
      rateLimit:
        local:
          requests: 3
          unit: minute
EOF
```

###### 5.3.2.3.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080/debug
```

###### 5.3.2.3.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
10:30:21 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T001 ended after 1.129974506s : 3 calls. qps=2.654927154613168
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T002 ended after 1.131041289s : 3 calls. qps=2.652423062868397
10:30:23 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:23 I periodic.go:721> T000 ended after 1.503440771s : 4 calls. qps=2.660563739627359
Ended after 1.503678339s : 10 calls. qps=6.6504
Sleep times : count 7 avg 0.52937663 +/- 0.03061 min 0.488825155 max 0.560046385 sum 3.70563638
Aggregated Function Time : count 10 avg 0.0052598876 +/- 0.003915 min 0.00164245 max 0.011292554 sum 0.052598876
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 20.00, 2
> 0.002 <= 0.003 , 0.0025 , 50.00, 3
> 0.003 <= 0.004 , 0.0035 , 60.00, 1
> 0.004 <= 0.005 , 0.0045 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 80.00, 1
> 0.011 <= 0.0112926 , 0.0111463 , 100.00, 2
# target 50% 0.003
# target 75% 0.0105
# target 90% 0.0111463
# target 99% 0.0112779
# target 99.9% 0.0112911
Error cases : count 7 avg 0.0027715733 +/- 0.001114 min 0.00164245 max 0.004884773 sum 0.019401013
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 28.57, 2
> 0.002 <= 0.003 , 0.0025 , 71.43, 3
> 0.003 <= 0.004 , 0.0035 , 85.71, 1
> 0.004 <= 0.00488477 , 0.00444239 , 100.00, 1
# target 50% 0.0025
# target 75% 0.00325
# target 90% 0.00426543
# target 99% 0.00482284
# target 99.9% 0.00487858
10:30:23 I httprunner.go:197> [0]   3 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [1]   2 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [2]   2 socket used, resolved to 10.96.56.21:8080
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.56.21:8080: 3
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 86.4 +/- 8.249 min 81 max 99 sum 864
All done 10 calls (plus 0 warmup) 5.260 ms avg, 6.7 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
```

###### 5.3.2.3.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

##### 5.3.2.4 每分钟 3 个请求, 波动峰值为 10，100%通过率

###### 5.3.2.4.1 调整限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpRoutes:
    - path: /debug
      rateLimit:
        local:
          requests: 3
          unit: minute
          burst: 10
EOF
```

###### 5.3.2.4.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080/debug
```

###### 5.3.2.4.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
01:15:26 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
01:15:27 I periodic.go:721> T001 ended after 1.128083378s : 3 calls. qps=2.6593778957356466
01:15:27 I periodic.go:721> T002 ended after 1.128407363s : 3 calls. qps=2.65861434298351
01:15:28 I periodic.go:721> T000 ended after 1.504108753s : 4 calls. qps=2.6593821703529437
Ended after 1.504282705s : 10 calls. qps=6.6477
Sleep times : count 7 avg 0.52865926 +/- 0.03053 min 0.488882417 max 0.55887838 sum 3.70061482
Aggregated Function Time : count 10 avg 0.0053517901 +/- 0.003907 min 0.00200691 max 0.011444314 sum 0.053517901
# range, mid point, percentile, count
>= 0.00200691 <= 0.003 , 0.00250345 , 30.00, 3
> 0.003 <= 0.004 , 0.0035 , 70.00, 4
> 0.011 <= 0.0114443 , 0.0112222 , 100.00, 3
# target 50% 0.0035
# target 75% 0.0110741
# target 90% 0.0112962
# target 99% 0.0114295
# target 99.9% 0.0114428
Error cases : no data
01:15:28 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.249.207:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.249.207:8080: 3
Code 200 : 10 (100.0 %)
Response Header Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
Response Body/Total Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
All done 10 calls (plus 0 warmup) 5.352 ms avg, 6.6 qps
```

正如上面的测试结果所示，所有的请求都成功执行

```bash
Code 200 : 10 (100.0 %)
```

###### 5.3.2.4.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
local_rate_limit.inbound_ratelimit/fortio_8078_tcp.rate_limited: 10
```

##### 5.3.2.5 每分钟 3 个请求，30%通过率，回写状态码 509

###### 5.3.2.5.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpRoutes:
    - path: /debug
      rateLimit:
        local:
          requests: 3
          unit: minute
          responseStatusCode: 509
          responseHeadersToAdd:
            - name: hello
              value: world
EOF
```

###### 5.3.2.5.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 http://fortio.ratelimit.svc.cluster.local:8080/debug
```

###### 5.3.2.5.3 测试结果

返回结果类似于:

```bash
10:04:43 I httprunner.go:102> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
10:04:43 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:04:43 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
10:04:43 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
10:04:44 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:04:44 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
10:04:44 I periodic.go:809> T002 ended after 1.131536359s : 3 calls. qps=2.651262574232491
10:04:44 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
10:04:44 I periodic.go:809> T001 ended after 1.131883596s : 3 calls. qps=2.6504492251692637
10:04:44 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:04:44 I periodic.go:809> T000 ended after 1.506504165s : 4 calls. qps=2.655153628466736
Ended after 1.506533766s : 10 calls. qps=6.6378
Sleep times : count 7 avg 0.52863491 +/- 0.03069 min 0.489218406 max 0.559163296 sum 3.70044435
Aggregated Function Time : count 10 avg 0.0063686645 +/- 0.003339 min 0.002259049 max 0.011597923 sum 0.063686645
# range, mid point, percentile, count
>= 0.00225905 <= 0.003 , 0.00262952 , 20.00, 2
> 0.003 <= 0.004 , 0.0035 , 30.00, 1
> 0.004 <= 0.005 , 0.0045 , 40.00, 1
> 0.005 <= 0.006 , 0.0055 , 60.00, 2
> 0.006 <= 0.007 , 0.0065 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 90.00, 2
> 0.011 <= 0.0115979 , 0.011299 , 100.00, 1
# target 50% 0.0055
# target 75% 0.01025
# target 90% 0.011
# target 99% 0.0115381
# target 99.9% 0.0115919
Error cases : count 7 avg 0.004355754 +/- 0.001534 min 0.002259049 max 0.006051199 sum 0.030490278
# range, mid point, percentile, count
>= 0.00225905 <= 0.003 , 0.00262952 , 28.57, 2
> 0.003 <= 0.004 , 0.0035 , 42.86, 1
> 0.004 <= 0.005 , 0.0045 , 57.14, 1
> 0.005 <= 0.006 , 0.0055 , 85.71, 2
> 0.006 <= 0.0060512 , 0.0060256 , 100.00, 1
# target 50% 0.0045
# target 75% 0.005625
# target 90% 0.00601536
# target 99% 0.00604762
# target 99.9% 0.00605084
# Socket and IP used for each connection:
[0]   3 socket used, resolved to [10.96.137.219:8080] connection timing : count 3 avg 0.00024358433 +/- 6.69e-05 min 0.000172271 max 0.000333078 sum 0.000730753
[1]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.0001659035 +/- 2.797e-05 min 0.000137933 max 0.000193874 sum 0.000331807
[2]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.000200063 +/- 7.513e-05 min 0.000124936 max 0.00027519 sum 0.000400126
Connection time histogram (s) : count 7 avg 0.00020895514 +/- 6.943e-05 min 0.000124936 max 0.000333078 sum 0.001462686
# range, mid point, percentile, count
>= 0.000124936 <= 0.000333078 , 0.000229007 , 100.00, 7
# target 50% 0.000211662
# target 75% 0.00027237
# target 90% 0.000308795
# target 99% 0.00033065
# target 99.9% 0.000332835
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.137.219:8080: 7
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 101.1 +/- 1.375 min 99 max 102 sum 1011
All done 10 calls (plus 0 warmup) 6.369 ms avg, 6.6 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
```

###### 5.3.2.5.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

###### 5.3.2.5.5 测试指令

多次执行，触发限流

```bash
curl="$(kubectl get pod -n ratelimit -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl" -n ratelimit -c curl -- curl -si http://fortio.ratelimit.svc.cluster.local:8080/debug
```

###### 5.3.2.5.6 测试结果

返回结果类似于:

```bash
HTTP/1.1 509 Unassigned
hello: world
content-length: 17
connection: keep-alive
```

返回响应头:

```bash
hello: world
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
export osm_namespace=osm-system 
kubectl patch meshconfig osm-mesh-config -n "$osm_namespace" -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
```

### 5.4 请求头层级限速

#### 5.4.1 每分钟 3 个请求，30%通过率

##### 5.4.1.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpHeaders:
    - headers:
        - name: hello
          value: world
      rateLimit:
        local:
          requests: 3
          unit: minute
EOF
```

##### 5.4.1.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 -H "hello:world" http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.4.1.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
10:30:21 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:22 W http_client.go:889> [1] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T001 ended after 1.129974506s : 3 calls. qps=2.654927154613168
10:30:22 W http_client.go:889> [2] Non ok http code 429 (HTTP/1.1 429)
10:30:22 I periodic.go:721> T002 ended after 1.131041289s : 3 calls. qps=2.652423062868397
10:30:23 W http_client.go:889> [0] Non ok http code 429 (HTTP/1.1 429)
10:30:23 I periodic.go:721> T000 ended after 1.503440771s : 4 calls. qps=2.660563739627359
Ended after 1.503678339s : 10 calls. qps=6.6504
Sleep times : count 7 avg 0.52937663 +/- 0.03061 min 0.488825155 max 0.560046385 sum 3.70563638
Aggregated Function Time : count 10 avg 0.0052598876 +/- 0.003915 min 0.00164245 max 0.011292554 sum 0.052598876
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 20.00, 2
> 0.002 <= 0.003 , 0.0025 , 50.00, 3
> 0.003 <= 0.004 , 0.0035 , 60.00, 1
> 0.004 <= 0.005 , 0.0045 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 80.00, 1
> 0.011 <= 0.0112926 , 0.0111463 , 100.00, 2
# target 50% 0.003
# target 75% 0.0105
# target 90% 0.0111463
# target 99% 0.0112779
# target 99.9% 0.0112911
Error cases : count 7 avg 0.0027715733 +/- 0.001114 min 0.00164245 max 0.004884773 sum 0.019401013
# range, mid point, percentile, count
>= 0.00164245 <= 0.002 , 0.00182123 , 28.57, 2
> 0.002 <= 0.003 , 0.0025 , 71.43, 3
> 0.003 <= 0.004 , 0.0035 , 85.71, 1
> 0.004 <= 0.00488477 , 0.00444239 , 100.00, 1
# target 50% 0.0025
# target 75% 0.00325
# target 90% 0.00426543
# target 99% 0.00482284
# target 99.9% 0.00487858
10:30:23 I httprunner.go:197> [0]   3 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [1]   2 socket used, resolved to 10.96.56.21:8080
10:30:23 I httprunner.go:197> [2]   2 socket used, resolved to 10.96.56.21:8080
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.56.21:8080: 3
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 86.4 +/- 8.249 min 81 max 99 sum 864
All done 10 calls (plus 0 warmup) 5.260 ms avg, 6.7 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 429 : 7 (70.0 %)
```

##### 5.4.1.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

#### 5.4.2 每分钟 3 个请求, 波动峰值为 10，100%通过率

##### 5.4.2.1 调整限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpHeaders:
    - headers:
        - name: hello
          value: world
      rateLimit:
        local:
          requests: 3
          unit: minute
          burst: 10
EOF
```

##### 5.4.2.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 -H "hello:world" http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.4.2.3 测试结果

返回结果类似于:

```bash
Fortio 1.34.1 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
01:15:26 I httprunner.go:98> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
01:15:27 I periodic.go:721> T001 ended after 1.128083378s : 3 calls. qps=2.6593778957356466
01:15:27 I periodic.go:721> T002 ended after 1.128407363s : 3 calls. qps=2.65861434298351
01:15:28 I periodic.go:721> T000 ended after 1.504108753s : 4 calls. qps=2.6593821703529437
Ended after 1.504282705s : 10 calls. qps=6.6477
Sleep times : count 7 avg 0.52865926 +/- 0.03053 min 0.488882417 max 0.55887838 sum 3.70061482
Aggregated Function Time : count 10 avg 0.0053517901 +/- 0.003907 min 0.00200691 max 0.011444314 sum 0.053517901
# range, mid point, percentile, count
>= 0.00200691 <= 0.003 , 0.00250345 , 30.00, 3
> 0.003 <= 0.004 , 0.0035 , 70.00, 4
> 0.011 <= 0.0114443 , 0.0112222 , 100.00, 3
# target 50% 0.0035
# target 75% 0.0110741
# target 90% 0.0112962
# target 99% 0.0114295
# target 99.9% 0.0114428
Error cases : no data
01:15:28 I httprunner.go:197> [0]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [1]   1 socket used, resolved to 10.96.249.207:8080
01:15:28 I httprunner.go:197> [2]   1 socket used, resolved to 10.96.249.207:8080
Sockets used: 3 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.249.207:8080: 3
Code 200 : 10 (100.0 %)
Response Header Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
Response Body/Total Sizes : count 10 avg 99 +/- 0 min 99 max 99 sum 990
All done 10 calls (plus 0 warmup) 5.352 ms avg, 6.6 qps
```

正如上面的测试结果所示，所有的请求都成功执行

```bash
Code 200 : 10 (100.0 %)
```

##### 5.4.2.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
local_rate_limit.inbound_ratelimit/fortio_8078_tcp.rate_limited: 10
```

#### 5.4.3 每分钟 3 个请求，30%通过率，回写状态码 509

##### 5.4.3.1 设置限速策略

```bash
kubectl apply -f - <<EOF
apiVersion: policy.openservicemesh.io/v1alpha1
kind: UpstreamTrafficSetting
metadata:
  name: http-rate-limit
  namespace: ratelimit
spec:
  host: fortio.ratelimit.svc.cluster.local
  httpHeaders:
    - headers:
        - name: hello
          value: world
      rateLimit:
        local:
          requests: 3
          unit: minute
          responseStatusCode: 509
          responseHeadersToAdd:
            - name: hello
              value: world
EOF
```

##### 5.4.3.2 测试指令

```bash
fortio_client="$(kubectl get pod -n ratelimit -l app=fortio-client -o jsonpath='{.items[0].metadata.name}')"

kubectl exec "$fortio_client" -n ratelimit -c fortio-client -- fortio load -c 3 -n 10 -H "hello:world" http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.4.3.3 测试结果

返回结果类似于:

```bash
Fortio 1.38.0 running at 8 queries per second, 8->8 procs, for 10 calls: http://fortio.ratelimit.svc.cluster.local:8080
10:13:19 I httprunner.go:102> Starting http test for http://fortio.ratelimit.svc.cluster.local:8080 with 3 threads at 8.0 qps and parallel warmup
Starting at 8 qps with 3 thread(s) [gomax 8] : exactly 10, 3 calls each (total 9 + 1)
10:13:19 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:13:19 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
10:13:19 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
10:13:20 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:13:20 W http_client.go:922> [2] Non ok http code 509 (HTTP/1.1 509)
10:13:20 I periodic.go:809> T002 ended after 1.131283315s : 3 calls. qps=2.651855605242441
10:13:20 W http_client.go:922> [1] Non ok http code 509 (HTTP/1.1 509)
10:13:20 I periodic.go:809> T001 ended after 1.132550849s : 3 calls. qps=2.6488876880441063
10:13:20 W http_client.go:922> [0] Non ok http code 509 (HTTP/1.1 509)
10:13:20 I periodic.go:809> T000 ended after 1.50629727s : 4 calls. qps=2.655518322754445
Ended after 1.506371683s : 10 calls. qps=6.6385
Sleep times : count 7 avg 0.52789827 +/- 0.03063 min 0.488691677 max 0.557950572 sum 3.69528787
Aggregated Function Time : count 10 avg 0.0070139603 +/- 0.002997 min 0.003365999 max 0.011462095 sum 0.070139603
# range, mid point, percentile, count
>= 0.003366 <= 0.004 , 0.003683 , 20.00, 2
> 0.004 <= 0.005 , 0.0045 , 30.00, 1
> 0.005 <= 0.006 , 0.0055 , 40.00, 1
> 0.006 <= 0.007 , 0.0065 , 60.00, 2
> 0.007 <= 0.008 , 0.0075 , 70.00, 1
> 0.01 <= 0.011 , 0.0105 , 80.00, 1
> 0.011 <= 0.0114621 , 0.011231 , 100.00, 2
# target 50% 0.0065
# target 75% 0.0105
# target 90% 0.011231
# target 99% 0.011439
# target 99.9% 0.0114598
Error cases : count 7 avg 0.0052136809 +/- 0.001418 min 0.003365999 max 0.0074324 sum 0.036495766
# range, mid point, percentile, count
>= 0.003366 <= 0.004 , 0.003683 , 28.57, 2
> 0.004 <= 0.005 , 0.0045 , 42.86, 1
> 0.005 <= 0.006 , 0.0055 , 57.14, 1
> 0.006 <= 0.007 , 0.0065 , 85.71, 2
> 0.007 <= 0.0074324 , 0.0072162 , 100.00, 1
# target 50% 0.0055
# target 75% 0.006625
# target 90% 0.00712972
# target 99% 0.00740213
# target 99.9% 0.00742937
# Socket and IP used for each connection:
[0]   3 socket used, resolved to [10.96.137.219:8080] connection timing : count 3 avg 0.00025921467 +/- 7.802e-05 min 0.000149211 max 0.000321564 sum 0.000777644
[1]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.0001199535 +/- 2.047e-05 min 9.9479e-05 max 0.000140428 sum 0.000239907
[2]   2 socket used, resolved to [10.96.137.219:8080] connection timing : count 2 avg 0.0001547225 +/- 1.082e-05 min 0.000143902 max 0.000165543 sum 0.000309445
Connection time histogram (s) : count 7 avg 0.00018957086 +/- 8.107e-05 min 9.9479e-05 max 0.000321564 sum 0.001326996
# range, mid point, percentile, count
>= 9.9479e-05 <= 0.000321564 , 0.000210521 , 100.00, 7
# target 50% 0.000192014
# target 75% 0.000256789
# target 90% 0.000295654
# target 99% 0.000318973
# target 99.9% 0.000321305
Sockets used: 7 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false
IP addresses distribution:
10.96.137.219:8080: 7
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
Response Header Sizes : count 10 avg 29.7 +/- 45.37 min 0 max 99 sum 297
Response Body/Total Sizes : count 10 avg 101.1 +/- 1.375 min 99 max 102 sum 1011
All done 10 calls (plus 0 warmup) 7.014 ms avg, 6.6 qps
```

正如上面的测试结果所示，30%的请求成功执行

```bash
Code 200 : 3 (30.0 %)
Code 509 : 7 (70.0 %)
```

##### 5.4.3.4 指标数据

```bash
fortio_server="$(kubectl get pod -n ratelimit -l app=fortio -o jsonpath='{.items[0].metadata.name}')"
osm proxy get stats "$fortio_server" -n ratelimit | grep http_local_rate_limiter.http_local_rate_limit
```

查询结果:

```bash
http_local_rate_limiter.http_local_rate_limit.rate_limited: 7
```

##### 5.4.3.5 测试指令

多次执行，触发限流

```bash
curl="$(kubectl get pod -n ratelimit -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl" -n ratelimit -c curl -- curl -sI -H "hello:world" http://fortio.ratelimit.svc.cluster.local:8080
```

##### 5.4.3.6 测试结果

返回结果类似于:

```bash
HTTP/1.1 509 Unassigned
hello: world
content-length: 17
connection: keep-alive
```

返回响应头:

```bash
hello: world
```

本业务场景测试完毕，清理策略，以避免影响后续测试

```bash
kubectl delete upstreamtrafficsettings -n ratelimit http-rate-limit
```

