# EFK (Elasticsearch, Fluent-bit and Kibana) for Log Management



## Create Namespace
```
# kubectl create -f namespace.yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: efk-logging
---
# kubectl get namespace
```
# Setup Elasticsearch

## Creating Headless Service

```
# kubectl create -f elastic-service.yml
---
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: efk-logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```
Label **app: elasticsearch** which will be used when we define the statefulset for Elasticsearch. Also, we have kept the clusterIP as None as this is requried for making it a "headless" service.
Ports 9200 and 9300 specified for REST API access and inter-node communication service
```
# kubectl get svc -n efk-logging
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None         <none>        9200/TCP,9300/TCP   11m
```
Now that we’ve set up our headless service and a stable .elasticsearch.efk-logging.svc.cluster.local domain for our Pods, we can go ahead and create the StatefulSet.

## Creating the StatefulSet
```
# kubectl create -f elasticsearch_statefulset.yml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: efk-logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.10.1
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts
            value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0,es-cluster-1,es-cluster-2"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 10Gi
```
```
# kubectl get sts -n efk-logging
NAME         READY   AGE
es-cluster   3/3     21h

# kubectl rollout status sts/es-cluster -n efk-logging
partitioned roll out complete: 3 new pods have been updated...

# kubectl get pod -n efk-logging
NAME           READY   STATUS    RESTARTS   AGE
es-cluster-0   1/1     Running   0          14m
es-cluster-1   1/1     Running   0          14m
es-cluster-2   1/1     Running   0          13m
```
### We can do a CURL request to the REST API, but for that we need the IP address of the pod, to get that, we need to run the following command:

```
# kubectl get pod -n efk-logging -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP               NODE                       NOMINATED NODE   READINESS GATES
es-cluster-0   1/1     Running   0          15m   192.168.248.58   worker2.yourdomain.com   <none>           <none>
es-cluster-1   1/1     Running   0          14m   192.168.24.248   worker1.yourdomain.com   <none>           <none>
es-cluster-2   1/1     Running   0          14m   192.168.24.249   worker1.yourdomain.com   <none>           <none>

# curl http://192.168.248.58:9200/_cluster/state?pretty
```
## Elasticsearch Ingress Rule
```
# kubectl create -f elastic-ingress.yml
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: efk-elastic-ingress
  namespace: efk-logging
spec:
  tls:
  - hosts:
      - efk-log.yourdomain.com
    secretName: efk-ingress
  rules:
  - host: efk-log.yourdomain.com
    http:
      paths:
      - path: /
        backend:
          serviceName: elasticsearch
          servicePort: 9200
```
```
root@master:~# kubectl get ing -n efk-logging
NAME                  CLASS    HOSTS                           ADDRESS        PORTS     AGE
efk-elastic-ingress   <none>   efk-log.yourdomain.com    172.28.28.14   80, 443   57s
```
# Setup Kibana
## Kibana Service Create:
```
# kubectl create -f kibana-service.yml
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: efk-logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
  selector:
    app: kibana
```
```
# kubectl get svc -n efk-logging
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None           <none>        9200/TCP,9300/TCP   6h20m
kibana          ClusterIP   10.111.87.12   <none>        5601/TCP            2s
```
## Kibana Deployment
```
# kubectl create -f kibana-deployment.yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: efk-logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: kibana:7.10.1
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
```
## Kibana Ingress Rule:
```
# kubectl create -f kibana-ingress.yml
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: efk-kibana-ingress
  namespace: efk-logging
spec:
  tls:
  - hosts:
      - efk.yourdomain.com
    secretName: efk-ingress
  rules:
  - host: efk.yourdomain.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kibana
          servicePort: 5601
```
```
# kubectl get ing -n efk-logging
NAME                 CLASS    HOSTS                       ADDRESS        PORTS     AGE
efk-kibana-ingress   <none>   efk.yourdomain.com   172.28.28.14   80, 443   15h
```

### Nginx ingress controller:
```
# kubectl get svc -n kube-system
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
heapster                        ClusterIP   10.106.73.173    <none>        80/TCP                       216d
kube-dns                        ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP       216d
nginx-ingress-controller        NodePort    10.105.205.110   <none>        80:31133/TCP,443:32167/TCP   110d
nginx-ingress-default-backend   ClusterIP   10.97.222.2      <none>        80/TCP                       110d
tiller-deploy                   ClusterIP   10.111.250.206   <none>        44134/TCP                    110d
```
# Then you can access the Kibana UI using the following URL: 
```
https://efk.yourdomain.com:32167/
```
# Now if you want to configure authentication for Kibana UI access:
## First login to a elastic pod and generate certificate
```
# kubectl -n efk-logging exec es-cluster-0 -it bash
[root@es-cluster-0 elasticsearch]# bin/elasticsearch-certutil cert -out config/elastic-certificates.p12 -pass ""

>>> Certificates written to /usr/share/elasticsearch/config/elastic-certificates.p12 <<<

>>> copy certificate file from pod to master machine <<<
```
### Copy the certificate from pod to master node:
```
# kubectl -n efk-logging cp es-cluster-0:/usr/share/elasticsearch/config/elastic-certificates.p12 /root/efk/elastic-certificates.p12
```
#### Now create secret with certificate:
```
# kubectl create secret generic es-cert --from-file=elastic-certificates.p12 --namespace efk-logging

or

# kubectl create -f elastic-secret.yml
---
apiVersion: v1
data:
  elastic-certificates.p12: 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
kind: Secret
metadata:
  creationTimestamp: null
  name: es-cert
  namespace: efk-logging
```
```
# kubectl get secret -n efk-logging
NAME                  TYPE                                  DATA   AGE
es-cert               Opaque                                1      3s
```
#### Next, create configmap for elasticsearch.yaml

```
# kubectl create -f elastic-configmap.yml
---
apiVersion: v1
data:
  elasticsearch.yml: |
    cluster.name: "es-cluster"
    network.host: 0.0.0.0
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/elastic-certificates.p12
kind: ConfigMap
metadata:
  labels:
    app: elasticsearch
  name: elasticsearch-config
  namespace: efk-logging
```
```
# kubectl get configmap -n efk-logging
NAME                   DATA   AGE
elasticsearch-config   1      3s
```
## Re-deploy the StatefulSet with configmap & secret
```
# kubectl create -f 2-elasticsearch_statefulset.yml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: efk-logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.10.1
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        - name: elasticsearch-configuration
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          subPath: elasticsearch.yml
        - name: elasticsearch-certificate
          mountPath: /usr/share/elasticsearch/config/elastic-certificates.p12
          subPath: elastic-certificates.p12
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts
            value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0,es-cluster-1,es-cluster-2"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
      volumes:
        - name: elasticsearch-certificate
          secret:
            secretName: es-cert
        - name: elasticsearch-configuration
          configMap:
            name: elasticsearch-config
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: managed-nfs-storage
      resources:
        requests:
          storage: 10Gi
EOF
```
## Generate password for Elasticsearch cluster, later we shall use this credentials to access kibana UI.
### Login to a elastic pod and generate password:
```
# kubectl -n efk-logging exec es-cluster-0 -it bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
[root@es-cluster-0 elasticsearch]#
[root@es-cluster-0 elasticsearch]# bin/elasticsearch-setup-passwords auto
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
The passwords will be randomly generated and printed to the console.
Please confirm that you would like to continue [y/N]y

Changed password for user apm_system
PASSWORD apm_system = VpoyOJFhgdjkNhgy7xPpM7

Changed password for user kibana_system
PASSWORD kibana_system = FQ2L65NWE7vghrrcrrwz

Changed password for user kibana
PASSWORD kibana = FQ2L65NWE7vghrrcrrwz

Changed password for user logstash_system
PASSWORD logstash_system = DPRWBj76gfdwerzvnX

Changed password for user beats_system
PASSWORD beats_system = nb7Mf32zc9kggtgcfjJYoO

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = ugGPwhgftyvgdfhlVyiFOF

Changed password for user elastic
PASSWORD elastic = Hh5de3gmj8f543dgss5s2r
```
## Now create config map and deploy kibana
### Create configmap file for user kibana & insert password in kibana.yml file:
```
# kubectl create -f kibana-configmap.yml
---
apiVersion: v1
data:
  kibana.yml: |
    server.name: kibana
    server.host: 0.0.0.0
    elasticsearch.hosts: [ "http://elasticsearch:9200" ]
    monitoring.ui.container.elasticsearch.enabled: true
    elasticsearch.username: "kibana"
    elasticsearch.password: "FQ2L65NWE7vghrrcrrwz"
kind: ConfigMap
metadata:
  labels:
    app: kibana
  name: kibana-config
  namespace: efk-logging
EOF
```
```
# kubectl get cm -n efk-logging
NAME                   DATA   AGE
elasticsearch-config   1      32h
kibana-config          1      11h
```
## Run deployment of Kibana with configmap
```
# kubectl create -f 2-kibana-deployment.yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: efk-logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: kibana:7.10.1
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
        volumeMounts:
          - name:  kibana-config
            mountPath: /usr/share/kibana/config/kibana.yml
            subPath: kibana.yml
      volumes:
        - name:  kibana-config
          configMap:
            name: kibana-config
```
```
# kubectl get deployment -n efk-logging
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
kibana   1/1     1            1           11h
```
```
# kubectl get pod -n efk-logging
NAME                    READY   STATUS    RESTARTS   AGE
es-cluster-0            1/1     Running   0          32h
es-cluster-1            1/1     Running   0          32h
es-cluster-2            1/1     Running   0          32h
kibana-dd478474-pr8v2   1/1     Running   0          11h
```
# Now you can access the Kibana UI using the following URL with user elastic:
https://efk.yourdomain.com:32167/
user: elastic
Password: Hh5de3gmj8f543dgss5s2r

# Reference link:
```
https://www.digitalocean.com/community/tutorials/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes
https://www.studytonight.com/post/efk-stack-setup-elasticsearch-fluentbit-and-kibana-for-kubernetes-log-management#
https://mherman.org/blog/logging-in-kubernetes-with-elasticsearch-Kibana-fluentd/
https://www.elastic.co/blog/getting-started-with-elasticsearch-security
```
