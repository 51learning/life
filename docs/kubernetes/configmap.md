 ConfigMap 功能在 Kubernetes1.2 版本的时候就有了，许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。这些配置信息需要与 docker image 解耦，ConfigMap API 给我们提供了向容器中注入配置信息的机制，ConfigMap 可以被用来保存单个属性，也可以用来保存整个配置文件或者 JSON 二进制大对象。

本篇先从环境变量开始。

```bash
[root@test-64626 ~]# kubectl create configmap myconfigmap --from-literal=user.name=itybku --from-literal=user.id=1001 

# 也可以直接使用yaml文件，可以通过如下命令生成myconfigmap的yaml文件
[root@test-64626 ~]# kubectl create configmap myconfigmap --from-literal=user.name=itybku --from-literal=user.id=1001 -o
 yaml --dry-run=client
apiVersion: v1
data:
  user.id: "1001"
  user.name: itybku
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: myconfigmap

[root@test-64626 ~]# kubectl describe configmaps myconfigmap
Name:         myconfigmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
user.id:
----
1001
user.name:
----
itybku
Events:  <none>
```


### 环境变量方式引用方式1：key引用

```bash
[root@test-64626 ~]# cat create-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-test-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      env:
        - name: ENV_VAR_USERNAME
          valueFrom:
            configMapKeyRef:
              name: myconfigmap
              key: user.name
        - name: ENV_VAR_ID
          valueFrom:
            configMapKeyRef:
              name: myconfigmap
              key: user.id
  restartPolicy: Never

[root@test-64626 ~]# kubectl create -f create-pod.yaml
pod/configmap-test-pod created
[root@test-64626 ~]# kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
configmap-test-pod   1/1     Running   0          4s
[root@test-64626 ~]# kubectl exec configmap-test-pod -- env|grep ENV
ENV_VAR_USERNAME=itybku
ENV_VAR_ID=1001
```


### 环境变量方式引用方式2：configmap名称映射

```bash
[root@test-64626 ~]# cat create-pod2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-test-all
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      envFrom:
        - configMapRef:
           name: myconfigmap
  restartPolicy: Never
[root@test-64626 ~]#
[root@test-64626 ~]# kubectl create -f create-pod2.yaml
pod/configmap-test-all created
[root@test-64626 ~]# kubectl exec configmap-test-all -- env|grep ENV
[root@test-64626 ~]# kubectl exec configmap-test-all -- env|grep user
user.id=1001
user.name=itybku
```

注意，在使用configmap名映射方式，不建议以user.id、user.name这样的方式命名变量，因为这样的变量无法正常通过`echo $变量名` 的方式进行输出，因为这不符合shell规范，里面包含了特殊符号点。可以改用下面的方式命名变量：

```bash
[root@test-64626 ~]# kubectl create configmap myconfigmap --from-literal=Name=itybku --from-literal=Id=1001 
```
