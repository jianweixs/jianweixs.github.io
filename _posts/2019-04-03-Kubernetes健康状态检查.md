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


# Pod健康状态检查


## 存活性探针( livenessProbe)：

判断容器是是否为健康，如果应用程序不能正常响应请求，则标记容器为非健康状态，根据deploy中设置的重启策略进行重启。

![google-kubernetes-probe-livenessae14](img/google-kubernetes-probe-livenessae14.GIF)

## 就绪性探针(readnessProbe)：

​	kubernetes配置一个等待时间，经过等待时间之后才可以执行第一次准备就绪检查。之后会周期性地调用探针，并根据就绪探针的结果采取行动。如果某个pod就绪检查未通过，则会从该服务中删除该pod。如果pod再次准备就绪，则重新添加pod。

![google-kubernetes-probe-readiness6ktf](img/google-kubernetes-probe-readiness6ktf.GIF)

探针有许多字段，您可以使用它们来更精确地控制活动和准备情况检查的行为：

- initialDelaySeconds：启动活动或准备就绪探测之前容器启动后的秒数。
- periodSeconds：执行探测的频率（以秒为单位）。默认为10秒。最小值为1。
- timeoutSeconds：探测超时的秒数。默认为1秒。最小值为1。
- successThreshold：失败后探测成功的最小连续成功次数。默认为1.活跃度必须为1。最小值为1。
- failureThreshold：当Pod启动并且探测失败时，Kubernetes将在放弃之前尝试failureThreshold times。在活动探测的情况下放弃意味着重新启动Pod。如果准备好探测，Pod将被标记为未准备好。默认为3.最小值为1。

HTTP probe 具有可在`httpGet`上设置的其他字段：

- host：要连接的主机名，默认为pod IP。 您可能希望在httpHeaders中设置“主机”。

- scheme：用于连接主机的方案（HTTP或HTTPS）。 默认为HTTP。

- path：HTTP服务器上的访问路径。

- httpHeaders：要在请求中设置的自定义标头。 HTTP允许重复标头。

- port：要在容器上访问的端口的名称或编号。 数字必须在1到65535的范围内。

许多运行很长时间的应用程序最终会变为损坏状态，除非重新启动，否则无法恢复。 Kubernetes提供了健康状态来检测和纠正这种情况。 在本练习中，您将创建一个基于`k8s.gcr.io/busybox`映像运行Container的Pod。 这是Pod的配置文件：

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

## 类型：

### Exec探针：

​	执行命令。容器的状态由命令执行完返回的状态码确定。如果返回的状态码是0，则认为pod是健康的，如果返回的是其他状态码，则认为pod不健康。

### HTTP Get探针

​	任何大于或等于200且小于400的代码表示成功。任何其他代码表示失败。

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


## 参考资料：

[Kubernetes最佳实践：使用准备和活性探针设置健康检查](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes)

[Configure Liveness and Readiness Probes](<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-tcp-liveness-probe>)

