ConfigMap用于保存配置。既可以保存为键值对格式，也可是保存为配置文件。可以在Pod的环境变量配置直接引用config map的值。或者是在数据卷(volume)中引用config map，把config map中的内容以文件形式放在volume中。

上一篇中我们通过`from-literal=key1=config1`的方式创建configmap，并在容器中通过变量映射的方式使用。本篇讲下`--from-file`和`--from-env-file`的用法，并通过volume的方式映射为容器内的configmap文件。

### 一、帮助信息

通过 `kubectl create configmap -h` 的帮助信息，我们可以获取到以下configmap的配置示例：

```bash
Examples:
  # Create a new configmap named my-config based on folder bar
  kubectl create configmap my-config --from-file=path/to/bar

  # Create a new configmap named my-config with specified keys instead of file basenames on disk
  kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt

  # Create a new configmap named my-config with key1=config1 and key2=config2
  kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2

  # Create a new configmap named my-config from the key=value pairs in the file
  kubectl create configmap my-config --from-file=path/to/bar

  # Create a new configmap named my-config from an env file
  kubectl create configmap my-config --from-env-file=path/to/bar.env
```

从这里可以看到`--from-file` 可以指定一个有多个文件的目录，也可以是多个单文件，通过指定不同的key来映射对应的文件。除此之外，其还有`--from-env-file`一样的用法，对应的文件里使用k/v 健值对的方式存相关变量信息。

### 二、volume方式挂载单个配置文件

**1\. 创建配置文件**

```bash
#创建一个配置文件 （假设是redis）
cat > redis.properties <<EOF
redis.host=127.0.0.1
redis.port=6379
redis.password=123456
EOF
```

**2\. 创建confimap**

```bash
#获取yaml
[root@test-64626 ~]# kubectl create configmap redis-config --from-file=redis.properties -o yaml --dry-run=client > redis-config-configmap.yaml
[root@test-64626 ~]# more redis-config-configmap.yaml
apiVersion: v1
data:
  redis.properties: |
    redis.host=127.0.0.1
    redis.port=6379
    redis.password=123456
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: redis-config
```

**3\. 部署**

```bash
kubectl create -f redis-config-configmap.yaml
```

**4\. 查看详细信息**

```bash
[root@test-64626 ~]# kubectl describe configmap redis-config
Name:         redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis.properties:
----
redis.host=127.0.0.1
redis.port=6379
redis.password=123456

Events:  <none>
```

**5\. 创建pod并添加挂载**

```bash
# 创建pod，并打印volume挂载后的文件信息
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: busybox
      image: busybox
      command: [ "/bin/sh","-c","cat /etc/config/redis.properties" ]
           #执行命令,将挂载的文件信息输出

      volumeMounts:        #挂载到容器
      - name: config-volume
        mountPath: /etc/config    #挂载的目录
  volumes:
    - name: config-volume
      configMap:              #指定为configmap文件
        name: redis-config      #刚才创建的redis-config的cm文件
  restartPolicy: Never

#部署
kubectl create -f cm.yaml
```

**6\. 查看容器信息**

```
# Completed表示完成并退出，因为这里只执行了cat文件,所以完成就退出了
[root@test-64626 ~]# kubectl get pods
NAME                 READY   STATUS      RESTARTS   AGE
mypod                0/1     Completed   0          22s
[root@test-64626 ~]# kubectl logs mypod
redis.host=127.0.0.1
redis.port=6379
redis.password=123456
```

同样的方法，还可以用于更新/etc/nginx/conf.d目录下的虚拟配置文件。
