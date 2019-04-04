---
layout:     post
title:      Kubernetes健康状态检查
subtitle:   Pod健康状态检查
date:       2019-04-03
author:     jianweixs
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - kubernetes
    - pod
    - 容器
    - 健康状态
---



[TOC]

# Pod健康状态检查



## 存活性探针( livenessProbe)

判断容器是是否为健康，如果应用程序不能正常响应请求，则标记容器为非健康状态，根据deploy中设置的重启策略进行重启。

![google-kubernetes-probe-readiness6ktf.GIF](https://i.loli.net/2019/04/03/5ca4a713bc1da.gif)

​	*Picture From - Configure Liveness and Readiness Probes*



## 就绪性探针(readnessProbe)

​	kubernetes配置一个等待时间，经过等待时间之后才可以执行第一次准备就绪检查。之后会周期性地调用探针，并根据就绪探针的结果采取行动。如果某个pod就绪检查未通过，则会从该服务中删除该pod。如果pod再次准备就绪，则重新添加pod。

![google-kubernetes-probe-livenessae14.GIF](https://i.loli.net/2019/04/03/5ca4a713bf0a6.gif)

​	*Picture From - Configure Liveness and Readiness Probes*



## 探针类型

### Exec探针

​	执行命令。容器的状态由命令执行完返回的状态码确定。如果返回的状态码是0，则认为pod是健康的，如果返回的是其他状态码，则认为pod不健康。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

在配置文件中，您可以看到Pod具有单个Container。 `periodSeconds`字段指定kubelet应每5秒执行一次活跃度探测。 `initialDelaySeconds`字段告诉kubelet它应该在执行第一个探测之前等待5秒。 要执行探测，kubelet将在Container中执行命令`cat /tmp/healthy`。 如果命令成功，则返回0，并且kubelet认为Container是活动且健康的。 如果该命令返回非零值，则kubelet会终止容器并重新启动它。

当Container启动时，它会执行以下命令：

```shell
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```

对于Container生命的前30秒，有一个`/tmp/health`文件。 因此，在前30秒，命令`cat /tmp/healthy`返回成功代码。 30秒后，`cat /tmp/healthy`返回失败代码。

创建Pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/exec-liveness.yaml
```

在30秒内，查看Pod事件：

```shell
FirstSeen    LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
24s       24s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "k8s.gcr.io/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "k8s.gcr.io/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
```

35秒后再次查看pod事件：

```shell
kubectl describe pod liveness-exec
```

在输出的底部，有消息指示活动探测失败，并且容器已被杀死并重新创建。

```shell
FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
37s       37s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "k8s.gcr.io/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "k8s.gcr.io/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

再等30秒，确认Container已重新启动

```shell
kubectl get pod liveness-exec
```

输出显示`RESTARTS`已递增1

```shell
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   1          1m
```

### HTTP Get探针

  任何大于或等于200且小于400的代码表示成功。任何其他代码表示失败。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```
该配置文件只定义了一个容器，`livenessProbe` 指定kubelet需要每隔3秒执行一次liveness probe。`initialDelaySeconds` 指定kubelet在该执行第一次探测之前需要等待3秒钟。该探针将向容器中的server的8080端口发送一个HTTP GET请求。如果server的`/healthz`路径的handler返回一个成功的返回码，kubelet就会认定该容器是活着的并且很健康。如果返回失败的返回码，kubelet将杀掉该容器并重启它。

任何大于200小于400的返回码都会认定是成功的返回码。其他返回码都会被认为是失败的返回码。

查看该server的源码：[server.go](https://github.com/kubernetes/kubernetes/blob/master/test/images/liveness/server.go).

最开始的10秒该容器是活着的， `/healthz` handler返回200的状态码。这之后将返回500的返回码。

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

容器启动3秒后，kubelet开始执行健康检查。第一次健康监测会成功，但是10秒后，健康检查将失败，kubelet将杀掉和重启容器。

创建一个Pod来测试一下HTTP liveness检测：

```bash
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/http-liveness.yaml
```

After 10 seconds, view Pod events to verify that liveness probes have failed and the Container has been restarted:

10秒后，查看Pod的event，确认liveness probe失败并重启了容器。

```bash
kubectl describe pod liveness-http
```

### TCP socket探针

​	它打开一个tcp链接到容器的指定端口，Kubernetes在指定端口上建立tcp连接。如果可以建立连接，容器被认为是健康的，如果不能建立，会被认为不健康。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

如您所见，TCP检查的配置与HTTP检查非常相似。 此示例同时使用了readiness和liveness probe。 容器启动后5秒钟，kubelet将发送第一个readiness probe。 这将尝试连接到端口8080上的goproxy容器。如果探测成功，则该pod将被标记为就绪。Kubelet将每隔10秒钟执行一次该检查。
除了readiness probe之外，该配置还包括liveness probe。 容器启动15秒后，kubelet将运行第一个liveness probe。 就像readiness probe一样，这将尝试连接到goproxy容器上的8080端口。如果liveness probe失败，容器将重新启动。
使用命名的端口
可以使用命名的ContainerPort作为HTTP或TCP liveness检查：

```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
  path: /healthz
  port: liveness-port
```

定义readiness探针
有时，应用程序暂时无法对外部流量提供服务。 例如，应用程序可能需要在启动期间加载大量数据或配置文件。 在这种情况下，你不想杀死应用程序，但你也不想发送请求。 Kubernetes提供了readiness probe来检测和减轻这些情况。 Pod中的容器可以报告自己还没有准备，不能处理Kubernetes服务发送过来的流量。
Readiness probe的配置跟liveness probe很像。唯一的不同是使用 readinessProbe而不是livenessProbe。

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

Readiness probe的HTTP和TCP的探测器配置跟liveness probe一样。
Readiness和livenss probe可以并行用于同一容器。 使用两者可以确保流量无法到达未准备好的容器，并且容器在失败时重新启动。



## 参考资料：

[Kubernetes最佳实践：使用准备和活性探针设置健康检查](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes)

[Configure Liveness and Readiness Probes](<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-tcp-liveness-probe>)

