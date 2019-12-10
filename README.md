# pavelvizir_platform
pavelvizir Platform repository  

[![Build Status](https://travis-ci.com/otus-kuber-2019-06/pavelvizir_platform.svg?branch=master)](https://travis-ci.com/otus-kuber-2019-06/pavelvizir_platform)

[Link to docker hub](https://hub.docker.com/r/pavelvizir)

# Table of contents:  
- [Homework-01 aka 'kubernetes-intro'](#homework-01-aka-kubernetes-intro)  
- [Homework-02 aka 'kubernetes-security'](#homework-02-aka-kubernetes-security)  
- [Homework-03 aka 'kubernetes-networks'](#homework-03-aka-kubernetes-networks)  
- [Homework-04 aka 'kubernetes-volumes'](#homework-04-aka-kubernetes-volumes)  
- [Homework-05 aka 'kubernetes-storage'](#homework-05-aka-kubernetes-storage)  
- [Homework-06 aka 'kubernetes-debug'](#homework-06-aka-kubernetes-debug)  
- [Homework-07 aka 'kubernetes-operators'](#homework-07-aka-kubernetes-operators)  

## Homework-01 aka 'kubernetes-intro'  
[.history-01](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-intro/.history-01)  
### Task \#1:  
#### How pods get recreated in kube-system

`kubectl delete pod --all -n kube-system`

> kube-apiserver = static pod, contolled by kubelet directly (/etc/kubernetes/manifests/kube-apiserver.yaml)  
> core-dns = normal pod in deployment, deployment controller, replicas > 0  

### Task \#2:  
#### Make everything in homework

[.history-01](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-intro/.history-01)  

## Homework-02 aka 'kubernetes-security'  
[.history-02](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-security/.history-02)  
### Tasks \#01, \#02, \#03:  
#### Play with service accounts, roles, (cluster)RoleBindings  

[.history-02](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-security/.history-02)  

## Homework-03 aka 'kubernetes-networks'  
[.history-03](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-networks/.history-03)  
### Task \#1:  
#### What's wrong with the following probe? Any chance to use it with purpose?  
```sh
livenessProbe:
  exec:
    command:
      - 'sh'
      - '-c'
      - 'ps aux | grep my_web_server_process'
```
> 1. Checking for main process is strange. You should aim to make it exit on errors.  
> 2. Checking for secondary process i suppose.  

### Task \#2:  
#### Play with strategy  
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 100%
```
kubectl get events --watch 
> brew install kubespy
> Disappointment: kubespy uses deprecated API :-(

### Task \#3:
#### Play with services

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc-cip
spec:
  selector:
    app: web
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

> Interesting dive into how it works and games with IPVS  

### Task \#4:
#### Play with Metallb

[.history-03](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-networks/.history-03#L142-L185)  

### Task \#5:
#### Play with ingress

[.history-03](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-networks/.history-03#L190-L247)  

## Homework-04 aka 'kubernetes-volumes'  
[.history-04](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-volumes/.history-04)  
### Task \#1:  
#### Install kind and MinIO  

```sh 
brew install kind minio/stable/mc
kind create cluster
kubectl apply -f . --recursive
```
Test it:  
```sh
mc ls
kubectl get statefulsets
kubectl get pods
kugectl get pvc
kubectl get pv
```
Clean up:
```sh
kind delete cluster
```

## Homework-05 aka 'kubernetes-storage'  
[.history-05](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-storage/.history-05)  
### Task \#1:  
#### Prepare cluster.yaml (feature gates to enable CSI snapshots)  

```yaml
---
apiVersion: kind.sigs.k8s.io/v1alpha3
kind: Cluster
kubeadmConfigPatches:
  - |
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: ClusterConfiguration
    metadata:
      name: config
    apiServer:
      extraArgs:
        "feature-gates": "VolumeSnapshotDataSource=true"
    scheduler:
      extraArgs:
        "feature-gates": "VolumeSnapshotDataSource=true"
    controllerManager:
      extraArgs:
        "feature-gates": "VolumeSnapshotDataSource=true"
  - |
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: InitConfiguration
    metadata:
      name: config
    nodeRegistration:
      kubeletExtraArgs:
        "feature-gates": "VolumeSnapshotDataSource=true"
```

### Task \#2:  
#### Create StorageClass for CSI Host Path Driver  

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-storageclass
provisioner: hostpath.csi.k8s.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

### Task \#3:  
#### Create PVC with name storage-pvc  

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: my-storageclass
```

### Task \#4:  
#### Create Pod with name storage-pod  
> Mount storage into /data  

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  restartPolicy: Always
  containers:
    - image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c"]
      args: ["tail -f /dev/null"]
      name: busybox
      volumeMounts:
        - name: storage-volume
          mountPath: /data
  volumes:
    - name: storage-volume
      persistentVolumeClaim:
        claimName: storage-pvc
```

### Task \#5:  
#### Test snapshots finally  
> Snapshot and restore manifests further down, first goes shell test commands  

```sh
# write some data
kubectl exec -it storage-pod -- /bin/sh
 cat /dev/urandom | env LC_CTYPE=C tr -dc A-Za-z0-9 | head -c $(expr 79) | tee /data/storage-file
 exit
# create snapshot
kubectl apply -f 04_Snapshot.yaml
# delete data
kubectl exec -it storage-pod -- /bin/sh -c 'rm -f /data/storage-file'
kubectl apply -f 05_RestoreSnapshot.yaml
# The PersistentVolumeClaim "storage-pvc" is invalid: spec: Forbidden: is immutable after creation except resources.requests for bound claims  
# :-(
# delete pvc
kubectl delete pods storage-pod
kubectl delete pvc storage-pvc
# restore snapshot
kubectl apply -f 05_RestoreSnapshot.yaml
# recrete pod and read data
kubectl exec -it storage-pod -- /bin/sh -c 'cat /data/storage-file'
7cq6RytcqmsSEFYQHpKQ3V5Uemg8RQCTuBQ1XTlPBSu0Bv3rZfi1vVhjCOa0x30G7HKLr84arYgBPtz
# HOORAY! OK
```

```yaml
---
apiVersion: snapshot.storage.k8s.io/v1alpha1
kind: VolumeSnapshot
metadata:
  name: storage-snapshot
spec:
  snapshotClassName: csi-hostpath-snapclass
  source:
    name: storage-pvc
    kind: PersistentVolumeClaim
```

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-pvc
spec:
  storageClassName: my-storageclass
  dataSource:
    name: storage-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### Task \#X:  
#### Overcoming tests  
> 1. Travis tests expect *bash* to be available,  
> so had to change image and command  

> 2. Travies tests expect storage class name = 'csi-hostpath-sc'

From:  
```
- image: gcr.io/google_containers/busybox
  command: ["/bin/sh", "-c"]
```
To:  
```
- image: ubuntu:18.04
  command: ["/bin/bash", "-ec"]
```
Perform snapshots test again:  
[Updated test run from .history-05](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-storage/.history-05#L212-L234)  

Update (change storage class name):
```sh
sed -i '' 's/my-storageclass/csi-hostpath-sc/' $(rg storageclass . -l)       
```
Perform snapshots test again:  
[Updated test run from .history-05](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-storage/.history-05#L241-L256)  

## Homework-06 aka 'kubernetes-debug'  
[.history-06](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-debug/.history-06)  
### Task \#1:  
#### Make strace work in kubectl debug  
[.history-06-kubectl-debug](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-debug/.history-06#L1-L31)  

> It works now

Because of:
```
docker inspect b9168fb3402f:
...
            "CapAdd": [
                "SYS_PTRACE",
                "SYS_ADMIN"
            ],
```
 because of:
[kubectl debug source code](https://github.com/aylei/kubectl-debug/blob/ca81bf784bc6570fb14b7c3ac4004d5b70853515/pkg/agent/runtime.go#L211)
```
CapAdd:      strslice.StrSlice([]string{"SYS_PTRACE", "SYS_ADMIN"}),
```

The easiest way to fail it is to empty CapAdd string and recompile

### Task \#2:  
#### Practice with iptables-tailer  
[.history-06-iptables-tailer](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-debug/.history-06#L33-L108)  

Install test app - netperf-operator:  
```sh
for i in crd rbac operator cr
do
  curl -O https://raw.githubusercontent.com/piontec/netperf-operator/master/deploy/$i.yaml
  kubectl apply -f $i.yaml
done        
```

Now create calico policy:
```yaml
---
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: netperf-calico-policy
  labels:
spec:
  order: 10
  selector: app == "netperf-operator"
  ingress:
    - action: Allow
      source:
        selector: netperf-role == "netperf-client"
    - action: Log
    - action: Deny
  egress:
    - action: Allow
      destination:
        selector: netperf-role == "netperf-client"
    - action: Log
    - action: Deny
```

Create service account, role, rolebinding for iptables-tailer.  

Install iptables-tailer.  

Fix all errors to finally get result with:  
`kubectl get events -A`  

## Homework-07 aka 'kubernetes-operators'  
[.history-07](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-operators/.history-07)  
### Task \#1:  
#### Create CR and CRD, make validations, adopt to 1.16

deploy/cr.yaml:
```yaml
---
apiVersion: otus.homework/v1
kind: MySQL
metadata:
  name: mysql-instance
spec:
  image: mysql:5.7
  database: otus-database
  password: otuspassword
  storage_size: 1Gi
# useless_data: "useless info
```
 
deploy/crd.yaml:
```yaml
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mysqls.otus.homework
spec:
  group: otus.homework
  preserveUnknownFields: false
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            # x-kubernetes-preserve-unknown-fields: false
            apiVersion:
              type: string
            kind:
              type: string
            metadata:
              type: object
              properties:
                name:
                  type: string
            spec:
              type: object
              properties:
                image:
                  type: string
                database:
                  type: string
                password:
                  type: string
                storage_size:
                  type: string
              required: ["image", "database", "password", "storage_size"]
          required: ["apiVersion", "kind", "metadata", "spec"]
  scope: Namespaced
  names:
    kind: MySQL
    plural: mysqls
    singular: mysql
    shortNames:
      - ms
```

### Task \#2:  
#### Deploy and test operator  

```sh
# deploy
for i in crd cr service-account role role-binding deploy-operator; do kubectl apply -f deploy/$i.yml; done
# check pvc
kubectl get pvc
  backup-mysql-instance-pvc   Bound    pvc-fa31bd64-474a-479e-a46c-790d7ce6c46a   1Gi        RWO            standard       98s
  mysql-instance-pvc          Bound    pvc-530e2339-f65a-437b-ba72-04c9fee45cb4   1Gi        RWO            standard       99s
# fill database with data
export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -u root  -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword  -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data-2' );" otus-database
# check table's contents
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
  +----+-------------+
  | id | name        |
  +----+-------------+
  |  1 | some data   |
  |  2 | some data-2 |
  +----+-------------+
# remove instance
kubectl delete mysqls.otus.homework mysql-instance
# check pv 
kubectl get pv # there is no more mysql-instance-pv
# check jobs
kubectl get jobs.batch
  backup-mysql-instance-job    1/1           2s         2m27s
# recreate instance
kubectl apply -f deploy/cr.yml
# wait a bit and check for completed job
kubectl get jobs
  backup-mysql-instance-job    1/1           2s         92s
  restore-mysql-instance-job   1/1           3m17s      3m20s
# now check data
export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
  +----+-------------+
  | id | name        |
  +----+-------------+
  |  1 | some data   |
  |  2 | some data-2 |
  +----+-------------+
```

### Task \#3:  
#### Adopt to 1.15

crd.yml:
```yaml
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: mysqls.otus.homework
spec:
  group: otus.homework
  preserveUnknownFields: false
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    kind: MySQL
    plural: mysqls
    singular: mysql
    shortNames:
      - ms
  validation:
    openAPIV3Schema:
      type: object
      properties:
        # x-kubernetes-preserve-unknown-fields: false
        apiVersion:
          type: string
        kind:
          type: string
        metadata:
          type: object
          properties:
            name:
              type: string
        spec:
          type: object
          properties:
            image:
              type: string
            database:
              type: string
            password:
              type: string
            storage_size:
              type: string
          required: ["image", "database", "password", "storage_size"]
      required: ["apiVersion", "kind", "metadata", "spec"]
```
