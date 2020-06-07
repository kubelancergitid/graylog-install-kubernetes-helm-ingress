# Graylog Installation on Kubernetes using Helm on Rancher with Ingress

## Steps followed here:
  0. Add Catalog
  1. Install Nginx Ingress
  2. Install Cert- Manager
  3. Install  mongodb-replicaset
  4. Install elasticsearch
  5. Install graylog
  6. Install fluent-bit
  7. Install Issuer
  8. DNS Records ADD
  9. Create Ingress for graylog
  10. Access graylog web ui
  11. Configure Graylog input
  12. Deploy sample application (nginx web server)
  13. Validate graylog lmessages/metrics
 


## 0. Add Catalog

  1. Go to Rancher Console > Cluster -> Default -> Apps -> Manage Catalogs > Add Catalog
Name: google-repo
Catalog URL: https://kubernetes-charts.storage.googleapis.com/

  2. Go to Rancher Console > Cluster -> System -> Apps -> Manage Catalogs > Add Catalog
Name: cert-manager
Catalog URL: https://charts.jetstack.io

## 1. Install Nginx Ingress

  1. Go to Rancher Console > Cluster -> Default -> Apps -> Launch
  2. Search "Nginx ingress"
  3. Select "Nginx ingress" and Launch.

## 2. Install Cert- Manager

  1. Go to Rancher Console > Cluster -> System -> Apps -> Launch
  2. Search "cert manager"
  3. Select "cert manager"
  4. Select Namespace "kube-system"
  5. Add Variable
```
installCRDs=true
```
  6. Launch.

## 3. Install  mongodb-replicaset

  1. Go to Rancher Console > Cluster -> Default -> Apps -> Launch
  2. Search "mongodb-replicaset"
  3. Select "mongodb-replicaset"
  4. Select Namespace "graylog" and Launch.

## 4. Install elasticsearch

  1. Go to Rancher Console > Cluster -> Default -> Apps -> Launch
  2. Search "elasticsearch"
  3. Select "elasticsearch"
  4. Select Namespace "graylog" and Launch.

## 5. Install graylog

  1. Go to Rancher Console > Cluster -> Default -> Apps -> Launch
  2. Search "graylog"
  3. Select "graylog"
  4. Select Namespace "graylog"
  5. Set Variables

```bash
graylog.elasticsearch.hosts=http://elasticsearch-client.graylog.svc.cluster.local:9200
graylog.externalUri=https://graylog.kubelancer.net
graylog.mongodb.uri=mongodb://mongodb-replicaset.graylog.svc.cluster.local:27017/graylog?replicaSet=rs0
tags.install-elasticsearch=false
tags.install-mongodb=false
```
  6. Launch
  7. Go to Rancher Console > Cluster -> Default -> Resources -> Config
    Select graylog
    Click Edit
    Modify parameter http_external_uri as like
```bash
http_external_uri = https://graylog.kubelancer.net/
```
  8. To Re-deploy pod to take configmap, Go to Rancher Console > Cluster -> Default -> Resources -> Workloads
  Click graylog
  Select graylog-0
  Click Delete

  Once POD up and running, then repeat same

  Click graylog
  Select graylog-1
  Click Delete

## 6. Install fluent-bit

  1. Go to Rancher Console > Cluster -> Default -> Apps -> Launch
  2. Search "fluent-bit"
  3. Select "fluent-bit"
  4. Select Namespace "graylog"
  5. Launch
  6. Go to Rancher Console > Cluster -> Default -> Resources -> Config
Select fluent-bit-config
Click Edit
Modify parameter fluent-bit-output.conf as like
```yaml
[OUTPUT]
Name gelf
Match *
Host graylog-0.graylog.graylog.svc.cluster.local
Port 12201
Mode tcp
Gelf_Short_Message_Key log
Retry_Limit False
tls off
tls.verify on
tls.debug 1
[OUTPUT]
Name gelf
Match *
Host graylog-1.graylog.graylog.svc.cluster.local
Port 12201
Mode tcp
Gelf_Short_Message_Key log
Retry_Limit False
tls off
tls.verify on
tls.debug 1
```
  7. To Re-deploy pod to take configmap, Go to Rancher Console > Cluster -> Default -> Resources -> Workloads
Click fluent-bit
Click Redeploy

## 7. Install Issuer

  1. create letsencrypt-prod-graylog.yaml
```bash
vi letsencrypt-prod-graylog.yaml
```
```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: graylog
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: kubernetio@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
       ingress:
         class: nginx
```
```bash
kubectl apply -f letsencrypt-prod-graylog.yaml
```
  2. To check Issuer status
```bash
kubectl get issuer -n graylog
```

## 8. DNS Records ADD
  1. To get ingress External IP
```bash
kubectl get svc -n nginx-ingress
```
  2. Add A record on DNS controller
```
Ex:
graylog.kubelancer.net A 35.184.146.232
```

## 9. Create Ingress for graylog

  1. create graylog-ingress.yaml
```bash
vi graylog-ingress.yaml
```
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: graylog-ingress-tls
  namespace: graylog
  annotations:
    fluentbit.io/parser: "k8s-nginx-ingress"
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: letsencrypt-prod
spec:
  tls:
  - secretName: graylog-ingress-tls
    hosts:
    - graylog.kubelancer.net
  rules:
  - host: graylog.kubelancer.net
    http:
      paths:
      - path: /
        backend:
          serviceName: graylog-web
          servicePort: 9000
```
  2. Create ingress for Graylog
```bash
kubectl apply -f graylog-ingress.yaml
```
  3. To check certs status
```bash
kubectl get certs -n graylog
```

## 10. Access graylog web ui
https://graylog.kubelancer.net

Credentials:
user: admin
password:

To get password, run below command
```bash
kubectl get secret/graylog -n graylog -o yaml | grep graylog-password-secret: | awk {'print $2'} | base64 -d ; echo
```

## 11. Configure Graylog input

  1. Goto graylog Console -> system/inputs -> inputs
  2. Select `GELF TCP` -> Launch new input
  3. Select Global
  4. Title GELF TCP
  5. Click Save

## 12. Deploy sample application (nginx web server)

  1. create deployment file nginx.yaml
```bash
vi nginx.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: graylog
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
status:
  loadBalancer: {}
---
---
apiVersion: v1
kind: Pod
metadata:
  namespace: graylog
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  annotations:
    fluentbit.io/parser: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
  2. Apply the deployment
```bash
kubectl apply -f nginx.yaml
```

  3. Create DNS record.
web1.kubelancer.net A 35.225.208.53

  4.  create ingress for web application

```bash
vi  web1-ingress.yaml
```
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web1-ingress-tls
  namespace: graylog
  annotations:
    fluentbit.io/parser: "k8s-nginx-ingress"
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: letsencrypt-prod
spec:
  tls:
  - secretName: web1-ingress-tls
    hosts:
    - web1.kubelancer.net
  rules:
  - host: web1.kubelancer.net
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
```
  5. Apply the web Ingress
```bash
kubectl apply -f web1-ingress.yaml
```

## 13. Validate graylog messages/metrics

  1. Goto graylog Console -> Search

```
```
#####  Now Messages will stream
#       *  *  *  *  *  

# * * * * Thank You * * * * *
