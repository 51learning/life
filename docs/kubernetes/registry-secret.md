## 使用私用镜像仓库鉴权

### 方式一

1.登陆私有镜像仓库

```bash
docker login reg.test.com

Username: admin  
Password:   
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.  
Configure a credential helper to remove this warning. See  
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded  
```
 

2.登陆成功之后，查看登陆配置文件，并转码

```bash
cat ~/.docker/config.json | base64 -w 0
```

记录转码后的信息

3.编辑secret的yaml文件

```yaml
apiVersion: v1  
kind: Secret  
metadata:  
  name: registry-pull-secret  
data:  
  .dockerconfigjson: ewoJImF1dGhzIjogewoJCSIxOTIuMTY4LjMxLjYxIjogewoJCQkiYXV0aCI6ICJZV1J0YVc0NlNHRnlZbTl5TVRJek5EVT0iCg6IHsKCQkiVXNlci1BZ2VudCI6ICJEb2NrZXItQ2xpZW50LzE4LjA2LjEtY2UgKGxpbnV4KSIKCX0KfQ==   #转码后的信息  
type: kubernetes.io/dockerconfigjson
```

4.在指定的命名空间创建 registry-pull-secret

```bash
kubectl create -f registry-pull-secret.yaml
```

### 方式二

1.使用kubectl命令创建secret文件

```bash
kubectl create secret docker-registry local-registry   
--docker-server=swr.la-north-2.myhuaweicloud.com   #docker 镜像的服务器地址  
--docker-username=admin   
--docker-password=******   
--docker-email=******   
-n default
```

2.kubernetes资源的引用

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "testapp"
  namespace: default
  labels:
    app: "testapp"
spec:
  containers:
  - name: testapp
    image: "swr.la-north-2.myhuaweicloud.com/hcie/nginx:v1"
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 20Mi
    ports:
    - containerPort:  80
      name:  http
  imagePullSecrets:
  - name: local-registry
```

在所有命名空间创建registry-pull-secret的脚本

```bash
#!/bin/bash  
ns_list=`kubectl get ns | awk '{print $1}' | grep -v NAME`  
for ns in $ns_list;  
do  
kubectl create secret docker-registry imagePullSecret-registry   
--docker-server=服务器地址   
--docker-username=admin   
--docker-password=******   
--docker-email=******   
-n $ns  
done;
```
