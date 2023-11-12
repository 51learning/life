使用kubectl scale 命令可以来实现 Pod 的扩缩容功能，但是这个毕竟是完全手动操作的，通过 kubectl autoscale 命令来创建一个 HPA 资源对象，HPA Controller默认30s轮询一次（可通过 kube-controller-manager 的--horizontal-pod-autoscaler-sync-period 参数进行设置）。

###　Metrics Server

在 HPA 的第一个版本中，我们需要 Heapster 提供 CPU 和内存指标，在 HPA v2 过后就需要安装 Metrcis Server 了，Metrics Server 可以通过标准的 Kubernetes API 把监控数据暴露出来，有了 Metrics Server 之后，我们就完全可以通过标准的 Kubernetes API 来访问我们想要获取的监控数据了：

`https://10.96.0.1/apis/metrics.k8s.io/v1beta1/namespaces/<namespace-name>/pods/<pod-name>`

当我们访问上面的 API 的时候，我们就可以获取到该 Pod 的资源数据，这些数据其实是来自于 kubelet 的 Summary API 采集而来的。不过需要说明的是我们这里可以通过标准的 API 来获取资源监控数据，并不是因为 Metrics Server 就是 APIServer 的一部分，而是通过 Kubernetes 提供的 Aggregator 汇聚插件来实现的，是独立于 APIServer 之外运行的。

![testpic](https://www.qikqiak.com/k8strain/assets/img/controller/k8s-hpa-ms.png)
```
[root@testcce-47617-dk54v ~]#  kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
{"kind":"NodeMetricsList","apiVersion":"metrics.k8s.io/v1beta1","metadata":{"selfLink":"/apis/metrics.k8s.io/v1beta1/nodes"},"items":[{"metadata":{"name":"192.168.0.111","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/192.168.0.111","creationTimestamp":"2022-04-07T03:17:11Z"},"timestamp":"2022-04-07T03:17:04Z","window":"30s","usage":{"cpu":"144172468n","memory":"1413452Ki"}},{"metadata":{"name":"192.168.0.114","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/192.168.0.114","creationTimestamp":"2022-04-07T03:17:11Z"},"timestamp":"2022-04-07T03:16:59Z","window":"30s","usage":{"cpu":"82688241n","memory":"1412200Ki"}}]}
You have new mail in /var/spool/mail/root
[root@testcce-47617-dk54v ~]# kubectl top nodes
NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
192.168.0.111   145m         1%     1380Mi          10%
192.168.0.114   83m          1%     1379Mi          10%
```

可以通过 kubectl top 命令来获取到资源数据了，证明 Metrics Server 已经安装成功了。

###　基于 CPU 的HPA

用 Deployment 来创建一个 Nginx Pod，然后利用 HPA 来进行自动扩缩容。资源清单如下所示：（hpa-demo.yaml）

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
然后直接创建 Deployment：

```
$ kubectl apply -f hpa-demo.yaml
deployment.apps/hpa-demo created
$ kubectl get pods -l app=nginx
NAME                        READY   STATUS    RESTARTS   AGE
hpa-demo-85ff79dd56-pz8th   1/1     Running   0          21s
```
现在我们来创建一个 HPA 资源对象，可以使用kubectl autoscale命令来创建：

```
$ kubectl autoscale deployment hpa-demo --cpu-percent=10 --min=1 --max=10
horizontalpodautoscaler.autoscaling/hpa-demo autoscaled
$ kubectl get hpa
NAME       REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   <unknown>/10%   1         10        1          16s
```

此命令创建了一个关联资源 hpa-demo 的 HPA，最小的 Pod 副本数为1，最大为10。HPA 会根据设定的 cpu 使用率（10%）动态的增加或者减少 Pod 数量。

当然我们依然还是可以通过创建 YAML 文件的形式来创建 HPA 资源对象。如果我们不知道怎么编写的话，可以查看上面命令行创建的HPA的YAML文件：

```
$ kubectl get hpa hpa-demo -o yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  annotations:
    autoscaling.alpha.kubernetes.io/conditions: '[{"type":"AbleToScale","status":"True","lastTransitionTime":"2019-11-19T09:15:12Z","reason":"SucceededGetScale","message":"the
      HPA controller was able to get the target''s current scale"},{"type":"ScalingActive","status":"False","lastTransitionTime":"2019-11-19T09:15:12Z","reason":"FailedGetResourceMetric","message":"the
      HPA was unable to compute the replica count: missing request for cpu"}]'
  creationTimestamp: "2019-11-19T09:14:56Z"
  name: hpa-demo
  namespace: default
  resourceVersion: "3094084"
  selfLink: /apis/autoscaling/v1/namespaces/default/horizontalpodautoscalers/hpa-demo
  uid: b84d79f1-75b0-46e0-95b5-4cbe3509233b
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  targetCPUUtilizationPercentage: 10
status:
  currentReplicas: 1
  desiredReplicas: 0
```

然后我们可以根据上面的 YAML 文件就可以自己来创建一个基于 YAML 的 HPA 描述文件了。但是我们发现上面信息里面出现了一些 Fail 信息，我们来查看下这个 HPA 对象的信息：
```
$ kubectl describe hpa hpa-demo
Name:                                                  hpa-demo
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 19 Nov 2019 17:14:56 +0800
Reference:                                             Deployment/hpa-demo
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  <unknown> / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       1 current / 0 desired
Conditions:
  Type           Status  Reason                   Message
  ----           ------  ------                   -------
  AbleToScale    True    SucceededGetScale        the HPA controller was able to get the target's current scale
  ScalingActive  False   FailedGetResourceMetric  the HPA was unable to compute the replica count: missing request for cpu
Events:
  Type     Reason                        Age                From                       Message
  ----     ------                        ----               ----                       -------
  Warning  FailedGetResourceMetric       14s (x4 over 60s)  horizontal-pod-autoscaler  missing request for cpu
  Warning  FailedComputeMetricsReplicas  14s (x4 over 60s)  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: missing request for cpu
```

我们可以看到上面的事件信息里面出现了 failed to get cpu utilization: missing request for cpu 这样的错误信息。这是因为我们上面创建的 Pod 对象没有添加 request 资源声明，这样导致 HPA 读取不到 CPU 指标信息，所以如果要想让 HPA 生效，对应的 Pod 资源必须添加 requests 资源声明，更新我们的资源清单文件：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: 50Mi
            cpu: 50m
```

然后重新更新 Deployment，重新创建 HPA 对象：

```
$ kubectl apply -f hpa.yaml 
deployment.apps/hpa-demo configured
$ kubectl get pods -o wide -l app=nginx
NAME                        READY   STATUS    RESTARTS   AGE     IP            NODE         NOMINATED NODE   READINESS GATES
hpa-demo-69968bb59f-twtdp   1/1     Running   0          4m11s   10.244.4.97   ydzs-node4   <none>           <none>
$ kubectl delete hpa hpa-demo
horizontalpodautoscaler.autoscaling "hpa-demo" deleted
$ kubectl autoscale deployment hpa-demo --cpu-percent=10 --min=1 --max=10
horizontalpodautoscaler.autoscaling/hpa-demo autoscaled
$ kubectl describe hpa hpa-demo                                          
Name:                                                  hpa-demo
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 19 Nov 2019 17:23:49 +0800
Reference:                                             Deployment/hpa-demo
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       1 current / 1 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange   the desired count is within the acceptable range
Events:           <none>
$ kubectl get hpa                                                        
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   0%/10%    1         10        1          52s
```

现在可以看到 HPA 资源对象已经正常了，现在我们来增大负载进行测试，我们来创建一个 busybox 的 Pod，并且循环访问上面创建的 Pod：

```
$ kubectl run -it --image busybox test-hpa --restart=Never --rm /bin/sh
If you don't see a command prompt, try pressing enter.
/ # while true; do wget -q -O- http://10.244.4.97; done
```

另开一个窗口，查看：

```
$  kubectl get hpa
NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   338%/10%   1         10        1          5m15s
$ kubectl get pods -l app=nginx --watch 
NAME                        READY   STATUS              RESTARTS   AGE
hpa-demo-69968bb59f-8hjnn   1/1     Running             0          22s
hpa-demo-69968bb59f-9ss9f   1/1     Running             0          22s
hpa-demo-69968bb59f-bllsd   1/1     Running             0          22s
hpa-demo-69968bb59f-lnh8k   1/1     Running             0          37s
hpa-demo-69968bb59f-r8zfh   1/1     Running             0          22s
hpa-demo-69968bb59f-twtdp   1/1     Running             0          6m43s
hpa-demo-69968bb59f-w792g   1/1     Running             0          37s
hpa-demo-69968bb59f-zlxkp   1/1     Running             0          37s
hpa-demo-69968bb59f-znp6q   0/1     ContainerCreating   0          6s
hpa-demo-69968bb59f-ztnvx   1/1     Running             0          6s
```

我们可以看到已经自动拉起了很多新的 Pod，最后定格在了我们上面设置的 10 个 Pod，同时查看资源 hpa-demo 的副本数量，副本数量已经从原来的1变成了10个：

```
$ kubectl get deployment hpa-demo
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
hpa-demo   10/10   10           10          17m
```

查看 HPA 资源的对象了解工作过程：

```
$ kubectl describe hpa hpa-demo
Name:                                                  hpa-demo
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 19 Nov 2019 17:23:49 +0800
Reference:                                             Deployment/hpa-demo
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       10 current / 10 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooManyReplicas      the desired replica count is more than the maximum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  5m45s  horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  5m30s  horizontal-pod-autoscaler  New size: 8; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  5m14s  horizontal-pod-autoscaler  New size: 10; reason: cpu resource utilization (percentage of request) above target
```

同样的这个时候我们来关掉 busybox 来减少负载，然后等待一段时间观察下 HPA 和 Deployment 对象：

```
$ kubectl get hpa
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   0%/10%    1         10        1          14m
$ kubectl get deployment hpa-demo
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
hpa-demo   1/1     1            1           24m
```

从 Kubernetes v1.12 版本开始我们可以通过设置 kube-controller-manager 组件的--horizontal-pod-autoscaler-downscale-stabilization 参数来设置一个持续时间，用于指定在当前操作完成后，HPA 必须等待多长时间才能执行另一次缩放操作。默认为5分钟，也就是默认需要等待5分钟后才会开始自动缩放。

可以看到副本数量已经由 10 变为 1，当前我们只是演示了 CPU 使用率这一个指标，在后面的课程中我们还会学习到根据自定义的监控指标来自动对 Pod 进行扩缩容。
