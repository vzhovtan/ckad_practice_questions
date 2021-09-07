### Practice questions for CKAD exam
These questions were collected and used to prepare for CKAD exam.

#### - Create a Persistent Volume called ```log-volume```. It should make use of a storage class name ```manual```. It should use RWX as the access mode and have a size of 1Gi. The volume should use the hostPath ```/opt/volume/nginx```. Next, create a PVC called ```log-claim``` requesting a minimum of 200Mi of storage. This PVC should bind to log-volume. Mount this in a pod called ```logger``` at the location ```/var/www/nginx```. This pod should use the image ```nginx:alpine```.
<details><summary>show</summary>
<p>

```
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: manual

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: log-volume
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  storageClassName: manual
  hostPath:
    path: /opt/volume/nginx

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi

>>> kubectl run logger --image=nginx:alpine -o yaml

---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: logger
  name: logger
spec:
  volumes:
    - name: pod-logger
      persistentVolumeClaim:
        claimName: log-claim
  containers:
  - image: nginx:alpine
    name: logger
    resources: {}
    volumeMounts:
      - mountPath: /var/www/nginx
        name: pod-logger
```
</p>
</details>

#### - There are deployed pod called ```secure-pod``` and a service called ```secure-service```. Incoming or Outgoing connections to this pod are not working.Troubleshoot why this is happening. Make sure that incoming connection from the pod ```webapp``` are successful.
<details><summary>show</summary>
<p>

```
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector:
    matchLabels:
      tier: frontend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: secret-db
  policyTypes:
  - Ingress
```
</p>
</details>

#### - Create a pod called time-check in the ```1987``` namespace. This pod should run a container called ```time-check``` that uses the ```busybox``` image.Create a config map called ```time-config``` with the data TIME_FREQ=10 in the same namespace. The time-check container should run the command: ```while true; do date; sleep $TIME_FREQ;done``` and write the result to the location ```/opt/time/time-check.log```. The path ```/opt/time``` on the pod should mount a volume that lasts the lifetime of this pod.
<details><summary>show</summary>
<p>

```
>>> kubectl create ns 1987
>>> kubectl get ns 
>>> kubectl config set-context --current --namespace=1987
>>> kubectl config view | grep -i namespace
>>> kubectl create cm time-config --from-literal=TIME_FREQ=10
>>> kubectl run time-check --image=busybox -o yaml --dry-run=client -- /bin/sh -c "while true; do date; sleep $TIME_FREQ; done >> /opt/time/time-check/log"

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: time-check
  name: time-check
  namespace: 1987
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - while true;do date;sleep $TIME_FREQ;done >> /opt/time/time-check.log
    image: busybox
    envFrom:
    - configMapRef:
        name: time-config
    imagePullPolicy: Always
    name: time-check
    volumeMounts:
    - mountPath: /opt/time/
      name: time-checklog
  volumes:
  - name: time-checklog
    emptyDir: {}
```
</p>
</details>

#### - Create a new deployment called ```nginx-deploy```, with one signle container called ```nginx```, image ```nginx:1.16``` and 4 replicas. The deployment should use RollingUpdate strategy with maxSurge=1, and maxUnavailable=2. Next upgrade the deployment to version 1.17 using rolling update. Finally, once all pods are updated, undo the update and go back to the previous version.
<details><summary>show</summary>
<p>

```
>>> kubectl create deploy nginx-deploy --image=nginx:1.16 --replicas=4

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
  namespace: default
spec:
  replicas: 4
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 50%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

>>> kubectl set image deploy/nginx-deploy nginx=nginx:1.17
>>> kubectl rollout undo deploy nginx-deploy
```
</p>
</details>

#### - Create a ```redis``` deployment with the following parameters:

```
Name of the deployment should be redis using the redis:alpine image. It should have exactly 1 replica.
The container should request for .2 CPU. It should use the label app=redis.
It should mount exactly 2 volumes:
* An Empty directory volume called data at path /redis-master-data.
* A configmap volume called redis-config at path /redis-master.
* The container should expose the port 6379
```
#### - Make sure that the pod is scheduled on master node.
<details><summary>show</summary>
<p>

```
>>> kubectl label node minikube type=redis
>>> kubectl taint node control plane <taint>-
>>> kubectl create deploy Redis --image=redis:alpine --replicas=1

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: redis
    spec:
      containers:
      - image: redis:alpine
        name: redis
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: 200m
        volumeMounts:
        - name: data
          mountPath: /redis-master-data
        - name: redis-config
          mountPath: /redis-master
      volumes:
      - name: data
        emptyDir: {}
      - name: edis-config
        configMap:
          name: redis-config
      nodeSelector:
        type: redis
```
</p>
</details>

#### - There are few pods deployed in the cluster in various namespaces. Inspect them and identify the pod which is not in a Ready state. Troubleshoot and fix the issue. Next, add a check to restart the container on the same pod if the command ```ls /var/www/html/file_check``` fails. This check should start after a delay of 10 seconds and run every 60 seconds.
<details><summary>show</summary>
<p>

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx1401
  namespace: dev1401
spec:
  containers:
  - image: kodekloud/nginx
    name: nginx
    ports:
    - containerPort: 9080
      protocol: TCP
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /
        port: 9080
        scheme: HTTP
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 1
    livenessProbe:
      exec:
        command:
        - ls
        - /var/www/html/file_check
      initialDelaySeconds: 10
      periodSeconds: 60
```
</p>
</details>

#### - Create a cronjob called ```dice``` that runs every one minute. Use the given Pod template located at ```/root/tdice```. The image ```dice``` randomly returns a value between 1 and 6. The result of 6 is considered success and all others are failure. The job should be non-parallel and complete the task once. Use a backoffLimit of 25. If the task is not completed within 20 seconds the job should fail and pods should be terminated.
<details><summary>show</summary>
<p>

```
>>> kubectl create cronjob dice --image=dice --schedule="*/1 * * * *" -o yaml --dry-run=client

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  creationTimestamp: null
  name: dice
spec:
  jobTemplate:
    metadata:
      name: dice
    spec:
      completions: 1
      backoffLimit: 25
      activeDeadlineSeconds: 20
      template:
        spec:
          containers:
          - image: dice
            name: dice
            resources: {}
          restartPolicy: OnFailure
  schedule: '*/1 * * * *'
```
</p>
</details>

#### - Create a pod called ```my-busybox``` in the ```dev``` namespace using the ```busybox``` image. The container should be called ```secret``` and should sleep for 3600 seconds. The container should mount a read-only secret volume called ```secret-volume``` at the path ```/etc/secret-volume```. The secret being mounted has already been created for you and is called ```dotfile-secret```. Make sure that the pod is scheduled on master and no other node in the cluster.
<details><summary>show</summary>
<p>

```
>>> kubectl -n dev2406 -run my-busybox --image=busybox -o yaml --dry-run=client

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: my-busybox
  name: my-busybox
  namespace: dev
spec:
  nodeName: master
  containers:
  - command: ["sleep", "3600"]
    image: busybox
    name: secret
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: dotfile-secret
```
</p>
</details>

#### - Create a single ingress resource called ```ingress-vh-routing```. The resource should route HTTP traffic to multiple hostnames as specified below:

```
* The service video-service should be accessible on http://watch.ecom-store.com:30093/video
* The service apparels-service should be accessible on http://apparels.ecom-store.com:30093/wear
```
<details><summary>show</summary>
<p>

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-vh-routing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: watch.ecom-store.com
    http:
      paths:
      - path: /video
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 8080
  - host: apparels.ecom-store.com
    http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: apparels-service
            port:
              number: 8080
```
</p>
</details>

#### - A pod called ```dev-pod-dind-878516``` has been deployed in the default namespace. Inspect the logs for the container called ```log-x``` and redirect the warnings to ```/opt/dind-878516_logs.txt``` on the master node.
<details><summary>show</summary>
<p>

```
>>> kubectl logs dev-pod-dind-878516 -c log-x | grep -i warning > /opt/dind-878516_logs.txt
```
</p>
</details>

#### - Deploy a pod named ```nginx-448839``` using the ```nginx:alpine``` image.
<details><summary>show</summary>
<p>

```
>>> kubectl run nginx-448839  --image=nginx:alpine

>>> kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
nginx-448839   1/1     Running   0          33s

```
</p>
</details>

#### - Create a namespace named ```apx-z993845```.
<details><summary>show</summary>
<p>

```
>>> kubectl create ns apx-z993845
namespace/apx-z993845 created
>>> kubectl get ns
NAME              STATUS   AGE
apx-z993845       Active   4s
```
</p>
</details>

#### - Create a new Deployment named ```httpd-frontend``` with 3 replicas using image ```httpd:2.4-alpine```.
<details><summary>show</summary>
<p>

```
>>> kubectl create deploy httpd-frontend --image=httpd:2.4-alpine --replicas=3
deployment.apps/httpd-frontend created
>>> kubectl get deploy
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
httpd-frontend   0/3     3            0           4s
```
</p>
</details>

#### - Deploy a ```messaging``` pod using the ```redis:alpine``` image with the labels set to ```tier=msg```.
<details><summary>show</summary>
<p>

```
>>> kubectl run messaging --image=redis:alpine --labels=tier=msg
pod/messaging created
>>> kubectl get pod --show-labels
NAME                              READY   STATUS    RESTARTS   AGE     LABELS
messaging                         1/1     Running   0          11s     tier=msg
```
</p>
</details>

#### - A replicaset ```rs-d33393``` is created. However the pods are not coming up. Identify and fix the issue. Once fixed, ensure the ReplicaSet has 4 Ready replicas.
<details><summary>show</summary>
<p>

```
spec:
  replicas: 4
  selector:
    matchLabels:
      name: busybox-pod
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: busybox-pod
    spec:
      containers:
      - command:
        - sh
        - -c
        - echo Hello Kubernetes! && sleep 3600
        image: busyboxXXXXXXX

>>> kubectl get rs rs-d33393 -o yaml> rs.yaml
>>> vim rs.yaml 
>>> kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
rs-d33393                   4         4         0       2m16s
>>> kubectl delete rs rs-d33393 --force
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
replicaset.apps "rs-d33393" force deleted
>>> kubectl create -f rs.yaml 
>>> kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
rs-d33393-9xptx                   1/1     Running   0          36s
rs-d33393-dc9jd                   1/1     Running   0          36s
rs-d33393-mfbwg                   1/1     Running   0          36s
rs-d33393-rrlxt                   1/1     Running   0          36s
```
</p>
</details>

#### - Create a service ```messaging-service``` to expose the ```redis``` deployment in the ```marketing``` namespace within the cluster on port ```6379```. Use imperative commands.
<details><summary>show</summary>
<p>

```
>>> kubectl -n marketing expose deploy redis --name=messaging-service --port=6379
service/messaging-service exposed
>>> kubectl -n marketing describe svc messaging-service
Name:              messaging-service
Namespace:         marketing
Labels:            <none>
Annotations:       <none>
Selector:          name=redis-pod
Type:              ClusterIP
IP:                10.100.15.148
Port:              <unset>  6379/TCP
TargetPort:        6379/TCP
Endpoints:         10.32.0.5:6379
Session Affinity:  None
Events:            <none>
```
</p>
</details>

#### - Update the environment variable on the pod ```webapp-color``` to use a green background.
<details><summary>show</summary>
<p>

```
>>> kubectl get pod webapp-color -o yaml > webapp.yaml
>>> vim webapp.yaml 
>>> kubectl delete pod webapp-color --force
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "webapp-color" force deleted
>>> kubectl create -f webapp.yaml 
pod/webapp-color created
>>> kubectl describe pod webapp-color | grep -i -C2 env
    Restart Count:  0
    Environment:
      APP_COLOR:  green
```
</p>
</details>

#### - Create a new ConfigMap named ```cm-3392845```. Use the spec given:

```
* ConfigName Name: cm-3392845
* Data: DB_NAME=SQL3322
* Data: DB_HOST=sql322.mycompany.com
* Data: DB_PORT=3306
```
<details><summary>show</summary>
<p>

```
>>> kubectl create cm cm-3392845 --from-literal=DB_NAME=SQL3322 --from-literal=DB_HOST=sql322.mycompany.com --from-literal=DB_PORT=3306
configmap/cm-3392845 created
>>> kubectl describe cm cm-3392845
Name:         cm-3392845
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
DB_PORT:
----
3306
DB_HOST:
----
sql322.mycompany.com
DB_NAME:
----
SQL3322
Events:  <none>
```
</p>
</details>

#### - Create a new Secret named ```db-secret-xxdf``` with the data given:

```
Secret Name: db-secret-xxdf
Secret 1: DB_Host=sql01
Secret 2: DB_User=root
Secret 3: DB_Password=password123
```
<details><summary>show</summary>
<p>

```
>>> kubectl create secret generic db-secret-xxdf --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123
secret/db-secret-xxdf created
>>> kubectl describe secret/db-secret-xxdf 
Name:         db-secret-xxdf
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
DB_Host:      5 bytes
DB_Password:  11 bytes
DB_User:      4 bytes
```
</p>
</details>

#### - Update pod ```app-sec``` to run as Root user and with the ```SYS_TIME``` capability.
<details><summary>show</summary>
<p>

```
>>> kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
app-sec.                          1/1     Running   0          27s
>>> kubectl get pod app-sec -o yaml > app.yaml

spec:
  securityContext:
    runAsUser: 0
  containers:
  - command:
    - sleep
    - "4800"
    image: ubuntu
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
    imagePullPolicy: Always
    name: ubuntu

>>> kubectl delete pod app-sec5 --force
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "app-sec" force deleted
>>> kubectl create -f app.yaml 
>>> kubectl exec app-sec -- whoami
root
```
</p>
</details>

#### - Export the logs of the ```e-com-1123``` pod to the file ```/opt/outputs/e-com-1123.logs```. It is in a different namespace. Identify the namespace first.
<details><summary>show</summary>
<p>

```
>>> kubectl get all -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
e-commerce    pod/e-com-1123                             1/1     Running   1          27m

>>> kubectl -n e-commerce logs e-com-1123 > /opt/outputs/e-com-1123.logs
>>> head /opt/outputs/e-com-1123.logs
[2021-06-28 16:28:18,035] INFO in event-simulator: USER1 is viewing page2
[2021-06-28 16:28:19,036] INFO in event-simulator: USER1 is viewing page3
[2021-06-28 16:28:20,038] INFO in event-simulator: USER1 is viewing page1
[2021-06-28 16:28:21,039] INFO in event-simulator: USER1 is viewing page2
[2021-06-28 16:28:22,041] INFO in event-simulator: USER2 is viewing page3
[2021-06-28 16:28:23,042] WARNING in event-simulator: USER5 Failed to Login as the account is locked due to MANY FAILED ATTEMPTS.
```
</p>
</details>

#### - Create a Persistent Volume with the given specification:

```
* Volume Name: pv-analytics
* Storage: 100Mi
* Access modes: ReadWriteMany
* Host Path: /pv/data-analytics
```
<details><summary>show</summary>
<p>

```
>>> kubectl create -f pv.yaml 
persistentvolume/pv-analytics created
>>> cat pv.yaml 
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/data-analytics
>>> kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-analytics   100Mi      RWX            Retain           Available           manual                  7s
```
</p>
</details>

#### - Create a ```redis``` deployment using the image ```redis:alpine``` with 1 replica and label ```app=redis```. Expose it via a ClusterIP service called ```redis``` on port ```6379```. Create a new Ingress Type NetworkPolicy called ```redis-access``` which allows only the pods with label ```access=redis``` to access the deployment.
<details><summary>show</summary>
<p>

```
>>> kubectl create deploy redis --image=redis:alpine --replicas=1
deployment.apps/redis created
>>> kubectl describe deploy redis
Name:                   redis
Namespace:              default
Labels:                 app=redis
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=redis
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=redis
  Containers:
   redis:
    Image:        redis:alpine
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   redis-647d66978 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  13s   deployment-controller  Scaled up replica set redis-647d66978 to 1

>>> kubectl expose deploy redis --name=redis --port=6379 --type=ClusterIP
service/redis exposed
>>> kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    35m
redis        ClusterIP   10.97.87.130   <none>        6379/TCP   5s

>>> kubectl create -f netpol.yaml 
networkpolicy.networking.k8s.io/redis-access created
>>> kubectl describe netpol redis-access
Name:         redis-access
Namespace:    default
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=redis
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From:
      PodSelector: access=redis
  Not affecting egress traffic
  Policy Types: Ingress
```
</p>
</details>

#### - Create a Pod called sega with two containers:

```
* Container 1: Name tails with image busybox and command: sleep 3600.
* Container 2: Name sonic with image nginx and Environment variable: NGINX_PORT with the value 8080.
```
<details><summary>show</summary>
<p>

```
>>> kubectl run sega --image=busybox -o yaml --dry-run=client -- "sleep 3600" > sega.yaml
>>> kubectl create -f sega.yaml 
pod/sega created
>>> cat sega.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sega
  name: sega
spec:
  containers:
  - command: ["sleep", "3600"]
    image: busybox
    name: tails
  - image: nginx
    name: sonic
    env:
    - name: NGINX_PORT
      value: "8080" 
```
</p>
</details>

#### - Create a deployment called ```my-webapp``` with image ```nginx```, label ```tier:frontend``` and 2 replicas. Expose the deployment as a NodePort service with name ```front-end-service``` , port ```80``` and NodePort ```30083```
<details><summary>show</summary>
<p>

```
>>> kubectl create deploy  my-webapp --image=nginx -o yaml --dry-run=client > deploy-ng.yaml
>>> vim deploy-ng.yaml 
>>> kubectl create -f deploy-ng.yaml 
>>> kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
my-webapp   2/2     2            2           44s

>>> kubectl create -f svc.yaml 
service/front-end-service created
>>> kubectl get svc
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
front-end-service   NodePort    10.97.65.157   <none>        80:30083/TCP   5s
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP        10m

>>> cat deploy-ng.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    tier: frontend
  name: my-webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-webapp
  strategy: {}
  template:
    metadata:
      labels:
        app: my-webapp
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
>>> cat svc.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    tier: frontend
  name: front-end-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30083
  selector:
    app: my-webapp
  type: NodePort
status:
  loadBalancer: {}
```
</p>
</details>

#### - Add a taint to the node worker of the cluster. Use the specification below:

```
key:app_type
value:alpha
effect:NoSchedule
```
#### - Create a pod called ```alpha```, image ```redis``` with toleration to ```worker``` node.
<details><summary>show</summary>
<p>

```
>>> kubectl taint node worker app_type=alpha:NoSchedule
node/worker tainted
>>> kubectl describe node worker | grep -i taint
Taints:             app_type=alpha:NoSchedule

>>> kubectl run alpha --image=redis -o yaml --dry-run=client > alpha.yaml
>>> vim alpha.yaml 
>>> kubectl create -f alpha.yaml 
pod/alpha created

>>> cat alpha.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: alpha
  name: alpha
spec:
  containers:
  - image: redis
    name: alpha
    resources: {}
  dnsPolicy: ClusterFirst
  tolerations:
  - key: app_type
    operator: "Equal"
    value: alpha
    effect: "NoSchedule"
  restartPolicy: Always
status: {}
>>> kubectl describe pod alpha | grep -i tole
Tolerations:     app_type=alpha:NoSchedule
>>> kubectl get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
alpha                        1/1     Running   0          51s   10.244.5.4   worker   <none>          <none>
```
</p>
</details>

#### - Apply a label ```app_type=beta``` to node ```node02```. Create a new deployment called ```beta-apps``` with image ```nginx``` and 3 replicas. Set Node Affinity to the deployment to place the PODs on ```node02``` only 
<details><summary>show</summary>
<p>

```
>>> kubectl label node node02 app_type=beta
node/node02 labeled
>>> kubectl get node --show-labels
NAME           STATUS   ROLES    AGE   VERSION   LABELS
controlplane   Ready    master   16m   v1.19.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=controlplane,kubernetes.io/os=linux,node-role.kubernetes.io/master=
node01         Ready    <none>   15m   v1.19.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01,kubernetes.io/os=linux
node02         Ready    <none>   15m   v1.19.0   app_type=beta,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node02,kubernetes.io/os=linux
node03         Ready    <none>   15m   v1.19.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node03,kubernetes.io/os=linux

>>> kubectl create deploy beta-apps --image=nginx --replicas=3 -o yaml --dry-run=client > beta.yaml

deployment.apps/beta-apps created
>>> kubectl get pod -o wide
NAME                         READY   STATUS              RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
beta-apps-6f69666d6-64f69    1/1     Running             0          9s      10.244.5.5   node02   <none>           <none>
beta-apps-6f69666d6-lzhx9    1/1     Running             0          9s      10.244.5.6   node02   <none>           <none>
beta-apps-6f69666d6-z9xjq    0/1     ContainerCreating   0          9s      <none>       node02   <none>           <none>

>>> cat beta.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: beta-apps
  name: beta-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: beta-apps
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: beta-apps
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: app_type
                operator: In
                values:
                - beta
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```
</p>
</details>

#### - Create a new Ingress Resource for the service ```my-video-service``` to be made available at the URL ```http://exam-solution.com:30093/video```.
<details><summary>show</summary>
<p>

```
>>> kubectl create -f ingress.yaml 
ingress.networking.k8s.io/my-ingress created
>>> kubectl get ingress
>>> 
NAME         CLASS    HOSTS                         ADDRESS   PORTS   AGE
my-ingress   <none>   exam-solution.com             80      4s

>>> cat ingress.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: exam-solution.com
    http:
      paths:
      - path: /video
        pathType: Prefix
        backend:
          service:
            name: my-video-service
            port:
              number: 8080

>>> curl -v http://exam-solution.com:30093/video
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to ckad-mock-exam-solution.com (127.0.0.1) port 30093 (#0)
> GET /video HTTP/1.1
> Host: ckad-mock-exam-solution.com:30093
> User-Agent: curl/7.58.0
> Accept: */*
> 
< HTTP/1.1 200 OK
```
</p>
</details>

#### - There is new pod deployed called ```pod-with-rprobe```. This Pod has an initial delay before it is Ready. Update the newly created pod ```pod-with-rprobe``` with a readinessProbe using the given spec

```
* httpGet path: /ready
* httpGet port: 8080
```
<details><summary>show</summary>
<p>

```
containers:
  - env:
    - name: APP_START_DELAY
      value: "180"
    image: /webapp-delayed-start
    imagePullPolicy: Always
    name: pod-with-rprobe
    ports:
    - containerPort: 8080
      protocol: TCP
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
        
>>> kubectl create -f rprobe.yaml 
pod/pod-with-rprobe created

>>> kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
pod-with-rprobe                 0/1     Running   0          14s

>>> kubectl describe pod pod-with-rprobe
Name:         pod-with-rprobe
Namespace:    default
Priority:     0
Node:         node03/172.17.0.39
Start Time:   Mon, 28 Jun 2021 17:52:08 +0000
Labels:       name=pod-with-rprobe
Annotations:  <none>
Status:       Running
IP:           10.244.6.6
IPs:
  IP:  10.244.6.6
Containers:
  pod-with-rprobe:
    Container ID:   docker://f43b3d37e325294671e51bb4d0b4d00c5055b228031a12a9e19a3f335d56ded5
    Image:          webapp-delayed-start
    Image ID:       docker-pullable://webapp-delayed-start@sha256:666b95c2ef8e00a74fa0ea96def8d9d69104ec5d9b9b0f49d8a76bd4c94ad343
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 28 Jun 2021 17:52:10 +0000
    Ready:          False
    Restart Count:  0
    Readiness:      http-get http://:8080/ready delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:
      APP_START_DELAY:  180        
```
</p>
</details>

#### - Create a new pod called ```nginx1401``` in the default namespace with the image ```nginx```. Add a livenessProbe to the container to restart it if the command ls ```/var/www/html/probe``` fails. This check should start after a delay of 10 seconds and run every 60 seconds.
<details><summary>show</summary>
<p>

```
>>> kubectl run nginx1401 --image=nginx -o yaml --dry-run=client > nginx.yaml

>>> kubectl create -f nginx.yaml 

>>> cat nginx.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx1401
  name: nginx1401
spec:
  containers:
  - image: nginx
    name: nginx1401
    livenessProbe:
      exec:
        command:
        - ls
        - /var/www/html/probe
      initialDelaySeconds: 10
      periodSeconds: 60
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

pod/nginx1401 created
>>> kubectl describe pod nginx 
Name:         nginx1401
Namespace:    default
Priority:     0
Node:         node03/172.17.0.39
Start Time:   Mon, 28 Jun 2021 17:55:21 +0000
Labels:       run=nginx1401
Annotations:  <none>
Status:       Running
IP:           10.244.6.7
IPs:
  IP:  10.244.6.7
Containers:
  nginx1401:
    Container ID:   docker://8ed1fde3c758c62c6652740a32ba8aec137dbddaaaa66cdcaf826de2c9f2664c
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:47ae43cdfc7064d28800bc42e79a429540c7c80168e8c8952778c0d5af1c09db
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 28 Jun 2021 17:55:23 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       exec [ls /var/www/html/probe] delay=10s timeout=1s period=60s #success=1 #failure=3
    Environment:    <none>
```
</p>
</details>

#### - Create a job called ```whalesay``` with image ```docker/whalesay``` and command "cowsay I am going to ace exam!".

```
* completions: 10
* backoffLimit: 6
* restartPolicy: Never
```
<details><summary>show</summary>
<p>

```
kubectl create job whalesay --image=docker/whalesay -o yaml --dry-run=client -- "cowsay I am going to ace exam!" > say.yaml

>>> kubectl create -f say.yaml 
job.batch/whalesay created

>>> kubectl get job
NAME       COMPLETIONS   DURATION   AGE
whalesay   0/10          4s         4s

>>> cat say.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: whalesay
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - cowsay I am going to ace CKAD!
        image: docker/whalesay
        name: whalesay
        resources: {}
      restartPolicy: Never
  backoffLimit: 6
  completions: 10
status: {}
```
</p>
</details>

#### - Create a pod called multi-pod with two containers:

```
* Container 1: name: jupiter, image: nginx
* Container 2: europa, image: busybox
* command: sleep 4800
Environment Variables: 
* Container 1: type: planet
* Container 2: type: moon
```
<details><summary>show</summary>
<p>

```
>>> cat multi.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi-pod
  name: multi-pod
spec:
  containers:
  - image: nginx
    name: jupiter
    env:
    - name: type
      value: planet
  
  - image: busybox
    name: europa 
    command: ["sleep", "4800"]
    env:
    - name: type
      value: moon
```
</p>
</details>

#### - Create a PersistentVolume called ```custom-volume``` with size ```50MiB``` reclaim policy ```Retain```, Access Modes: ```ReadWriteMany``` and hostPath ```/opt/data```.
<details><summary>show</summary>
<p>

```
>>> kubectl create -f pv.yaml 
persistentvolume/custom-volume created

>>> kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
custom-volume   50Mi       RWX            Retain           Available           manual                  4s

>>> cat pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: custom-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 50Mi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /opt/data
```
</p>
</details>
