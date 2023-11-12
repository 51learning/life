# kubectl 命令的使用

k8s命令自动补全:
 
```bash
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```  

应用部署

```bash
[root@testcce-26771 ~]# kubectl create deployment web --image=nginx --replicas=1
deployment.apps/web created

[root@testcce-26771 ~]# kubectl expose deployment web --port=8080 --target-port=80
service/web exposed
[root@testcce-26771 ~]# kubectl get  svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.247.0.1       <none>        443/TCP    59m
web          ClusterIP   10.247.145.241   <none>        8080/TCP   6s

[root@testcce-26771 ~]# kubectl get  svc web -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2023-09-29T03:43:43Z"
  labels:
    app: web
  name: web
  namespace: default
  resourceVersion: "15468"
  uid: 7e3bd0d8-fc7d-48d6-8ddb-16959c1f791b
spec:
  clusterIP: 10.247.145.241
  clusterIPs:
  - 10.247.145.241
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    app: web
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}

[root@testcce-26771 ~]# kubectl exec -it web-8667899c97-r7nmq  -- /bin/bash

root@web-8667899c97-r7nmq:/# curl 10.247.145.241:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

