


kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment web --port=8080
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.yaml


[root@ccetest-28382 ~]# kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
web-84fb9498c7-hb8wj   1/1     Running   0          81s   172.16.0.9   192.168.0.222   <none>           <none>
[root@ccetest-28382 ~]# curl 172.16.0.9:8080
Hello, world!
Version: 1.0.0
Hostname: web-84fb9498c7-hb8wj


[root@ccetest-28382 ~]# kubectl -n cert-manager get all
NAME                                          READY   STATUS    RESTARTS        AGE
pod/cert-manager-6856c9896b-wtnpn             1/1     Running   1 (4m36s ago)   4m48s
pod/cert-manager-cainjector-fc54fdc88-h848s   1/1     Running   0               4m48s
pod/cert-manager-webhook-68496779c4-mhtg5     1/1     Running   0               4m48s

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.247.114.49   <none>        9402/TCP   4m48s
service/cert-manager-webhook   ClusterIP   10.247.24.88    <none>        443/TCP    4m48s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           4m48s
deployment.apps/cert-manager-cainjector   1/1     1            1           4m48s
deployment.apps/cert-manager-webhook      1/1     1            1           4m48s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-6856c9896b             1         1         1       4m48s
replicaset.apps/cert-manager-cainjector-fc54fdc88   1         1         1       4m48s
replicaset.apps/cert-manager-webhook-68496779c4     1         1         1       4m48s


kubectl explain Certificate
kubectl explain CertificateRequest
kubectl explain Issuer



apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webingress
  namespace: default
  annotations:
    kubernetes.io/elb.port: '80'
    kubernetes.io/elb.class: performance
    kubernetes.io/elb.autocreate: '{"name":"testelb","type":"public","bandwidth_name":"cce-bandwidth-1696357100304","bandwidth_chargemode":"traffic","bandwidth_size":5,"bandwidth_sharetype":"PER","eip_type":"5_bgp","available_zone":["la-north-2a"],"elb_virsubnet_ids":["4d18b0fc-cfff-47e6-8801-bd2595093753"],"ipv6_vip_virsubnet_id":"4d18b0fc-cfff-47e6-8801-bd2595093753","l7_flavor_name":"L7_flavor.elb.s1.small","l4_flavor_name":""}'
spec:
  rules:
    - host: test.361way.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: webnode
                port:
                  number: 8080
            property:
              ingress.beta.kubernetes.io/url-match-mode: STARTS_WITH
            pathType: ImplementationSpecific
  ingressClassName: cce


[root@ccetest-28382 ~]# vim https-issuer.yaml
[root@ccetest-28382 ~]# kubectl apply -f https-issuer.yaml
issuer.cert-manager.io/letsencrypt-staging created
[root@ccetest-28382 ~]# kubectl describe issuers.cert-manager.io letsencrypt-staging
Name:         letsencrypt-staging
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Issuer
Metadata:
  Creation Timestamp:  2023-10-03T18:23:31Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
………………
    Manager:         cert-manager-issuers
    Operation:       Update
    Subresource:     status
    Time:            2023-10-03T18:23:32Z
  Resource Version:  9611
  UID:               aa53aa37-ca02-4844-a771-c58d9209dc99
Spec:
  Acme:
    Email:  itybku@gmail.com
    Private Key Secret Ref:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Name:  webingress
Status:
  Acme:
    Last Private Key Hash:  EjBRE8svSIkKk5DMCCiuEhS4nr2HRJGfktU3Z7j2U8o=
    Last Registered Email:  itybku@gmail.com
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/120612354
  Conditions:
    Last Transition Time:  2023-10-03T18:23:32Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>


# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-ssl
type: kubernetes.io/tls
stringData:
  tls.key: ""
  tls.crt: ""

kubectl apply -f secret.yaml


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingresshttps
  namespace: default
  annotations:
    kubernetes.io/elb.port: '443'
    kubernetes.io/elb.id: 8df86b16-bae3-414b-b0c7-ad4172d80f6e
    kubernetes.io/elb.class: performance
    kubernetes.io/elb.tls-ciphers-policy: tls-1-2
    cert-manager.io/issuer: letsencrypt-staging
spec:
  rules:
    - host: test.361way.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: webnode
                port:
                  number: 8080
            property:
              ingress.beta.kubernetes.io/url-match-mode: STARTS_WITH
            pathType: ImplementationSpecific
  ingressClassName: cce
  tls:
    - secretName: web-ssl



apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpstest
  namespace: default
  annotations:
    kubernetes.io/elb.port: '443'
    kubernetes.io/elb.class: performance
    cert-manager.io/issuer: letsencrypt-staging
    kubernetes.io/elb.autocreate: '{"name":"httpselb","type":"public","bandwidth_name":"cce-bandwidth-1696358134629","bandwidth_chargemode":"traffic","bandwidth_size":5,"bandwidth_sharetype":"PER","eip_type":"5_bgp","available_zone":["la-north-2a"],"elb_virsubnet_ids":["4d18b0fc-cfff-47e6-8801-bd2595093753"],"ipv6_vip_virsubnet_id":"4d18b0fc-cfff-47e6-8801-bd2595093753","l7_flavor_name":"L7_flavor.elb.s1.small","l4_flavor_name":""}'
spec:
  rules:
    - host: test.361way.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: webnode
                port:
                  number: 8080
            property:
              ingress.beta.kubernetes.io/url-match-mode: STARTS_WITH
            pathType: ImplementationSpecific
  ingressClassName: cce
  tls:
    - secretName: web-ssl
