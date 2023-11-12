为k8s容器配置环境变量，在[官方文档](https://kubernetes.io/zh-cn/docs/tasks/inject-data-application/define-environment-variable-container/ "官方文档")中有相关说明。本篇就是基于原内容的理解再加工。这里使用华为CCE集群我已准备好了k8s环境。

## 一、为容器设置一个环境变量

创建 Pod 时，可以为其下的容器设置环境变量。通过配置文件的 env 或者 envFrom 字段来设置环境变量。

基于以下yaml文件可以创建一个pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/node-hello:1.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
```

也可以使用`kubectl apply -f https://k8s.io/examples/pods/inject/envars.yaml`命令直接创建。
pod创建完成后，可以通过printenv命令打印输出：

```bash
[root@testcce-38939 ~]# kubectl exec envar-demo -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=envar-demo
DEMO_GREETING=Hello from the environment
DEMO_FAREWELL=Such a sweet sorrow
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.247.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.247.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.247.0.1
KUBERNETES_SERVICE_HOST=10.247.0.1
KUBERNETES_SERVICE_PORT=443
PAAS_POD_ID=460dedcc-d764-4042-afae-75f50df5cce4
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=4.4.2
HOME=/root
```

注：如果Dockerfile中配置的变量和k8s yaml文件中指定的变量重叠，并且赋值不同，通过Dockerfile的 env 或 envFrom 字段设置的环境变量将覆盖容器镜像中指定的所有环境变量。

## 二、环境变量的使用

环境变量的配置是为了使用，不然配置也没有用，其主要用于pod启动时的传参。比如开发环境、生产环境传入的参数就是不一样的，对应pod启动时，内部执行的脚本动作也有区别。这里来两个比较容易理解的示例。

### 示例1：echo输出

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-greeting
spec:
  containers:
  - name: env-print-demo
    image: bash
    env:
    - name: GREETING
      value: "Warm greetings to"
    - name: HONORIFIC
      value: "The Most Honorable"
    - name: NAME
      value: "Kubernetes"
    command: ["echo"]
    args: ["$(GREETING) $(HONORIFIC) $(NAME)"]
```

以上这个pod执行后，会执行命令`echo Warm greetings to The Most Honorable Kubernetes`。

### 示例2：mysql服务密码设置

这个不是官方文档示例，这个是在CCE k8s集群上创建mariadb应用的StatefulSet有状态应用的示例。这里重点关注下env部分，通过env部分，我们传了一个变量` MARIADB_ROOT_PASSWORD`，也就是数据库的root密码。

```yaml
[root@testcce-38939 ~]# kubectl apply -f mysql-statefulset.yaml
statefulset.apps/mariadb created
[root@testcce-38939 ~]# cat mysql-statefulset.yaml
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    annotations:
      description: ""
    generation: 1
    labels:
      appgroup: ""
      version: v1
    name: mariadb
    namespace: default
  spec:
    podManagementPolicy: OrderedReady
    replicas: 1
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        app: mariadb
        version: v1
    serviceName: mariadb-headless
    template:
      metadata:
        labels:
          app: mariadb
          version: v1
      spec:
        affinity: {}
        containers:
        - env:
          - name: MARIADB_ROOT_PASSWORD
            value: test@123
          image: mariadb:latest
          imagePullPolicy: Always
          name: container-0
          resources:
            limits:
              cpu: "2"
              memory: 2Gi
            requests:
              cpu: 250m
              memory: 512Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
          - mountPath: /var/lib/mysql
            name: pvc-mariadb-storage
        dnsConfig:
          options:
          - name: timeout
            value: ""
          - name: ndots
            value: "5"
          - name: single-request-reopen
        dnsPolicy: ClusterFirst
        imagePullSecrets:
        - name: default-secret
        - name: default-secret
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
        tolerations:
        - effect: NoExecute
          key: node.kubernetes.io/not-ready
          operator: Exists
          tolerationSeconds: 300
        - effect: NoExecute
          key: node.kubernetes.io/unreachable
          operator: Exists
          tolerationSeconds: 300
    updateStrategy:
      type: RollingUpdate
    volumeClaimTemplates:
    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        annotations:
          everest.io/disk-volume-type: SAS
        creationTimestamp: null
        name: pvc-mariadb-storage
        namespace: default
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
        storageClassName: csi-disk
        volumeMode: Filesystem
kind: List
```
注：
1. 这里的设置比较多，因为涉及到数据持久化保存的问题，这里配置了PVC和PV，如果仅出于测试目的，这里部分可以省略。
2. `everest.io/disk-volume-type` 这个类型是华为云CCE里特有的，每家云厂商对存储部分的接口是不相同的，在不同的云厂商上要根据云厂商公布的存储接口情况进行修改该项和 `storageClassName` 项。

接下来我们使用环境变量创建的密码可以连接数据库看下：

[![k8s-env-mysql-password](http://www.361way.com/wp-content/uploads/2021/09/k8s-env-mysql-password.png "k8s-env-mysql-password")](http://www.361way.com/wp-content/uploads/2021/09/k8s-env-mysql-password.png "k8s-env-mysql-password")
