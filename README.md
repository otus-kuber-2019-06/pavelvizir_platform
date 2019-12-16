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
- [Homework-10 aka 'kubernetes-templating'](#homework-10-aka-kubernetes-templating)  
- [Homework-11 aka 'kubernetes-vault'](#homework-11-aka-kubernetes-vault)  

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

## Homework-10 aka 'kubernetes-templating'  
[.history-10](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10)  
### Task \#1:  
#### Install nginx-ingress + helm 2 + tiller + cluster-admin  

[Log Task1](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L51-L56)  

### Task \#2:  
#### Install cert-manager + helm 2 + tiller(ns) + role(ns)  

[Log Task2](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L57-L147)  

### Task \#3:  
#### Finally install cert-manager  

[Log Task3](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L149-L198)  

### Task \#4:  
#### Install chartmuseum + helm 2 + helm-tiller  

[Log Task4](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L199-L239)  

### Task \#5:  
#### Install harbor + helm3  

[Log Task5](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L241-L287)  

### Task \#6:  
#### Create helm chart  

[Log Task6](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L289-L325)  

### Task \#7:  
#### Kubecfg  

[Log Task7](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L327-L342)  

### Task \#8:  
#### Kustomize  

[Log Task8](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-templating/.history-10#L344-L354)  

## Homework-11 aka 'kubernetes-vault'  
[.history-11](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-vault/.history-11)  
### Task \#1:  
#### Install vault  

```sh
# clone consul and vault repos with helm
git clone https://github.com/hashicorp/consul-helm.git
cp consul-helm/values.yaml consul-helm-values.yaml
./helm install --name=consul consul-helm -f consul-helm-values.yaml
git clone https://github.com/hashicorp/vault-helm.git
cp vault-helm/values.yaml vault-helm-values.yaml
# change values to mode, add ui
./helm install --name=vault vault-helm -f vault-helm-values.yaml
echo -e 'consul-helm\nhelm\nvault-helm' > .gitignore
# now check vault release status
helm status vault
LAST DEPLOYED: Fri Dec 13 17:28:25 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME          AGE
vault-config  11m

==> v1/Pod(related)
NAME     AGE
vault-0  11m
vault-1  11m
vault-2  11m

==> v1/Service
NAME      AGE
vault     11m
vault-ui  11m

==> v1/ServiceAccount
NAME   AGE
vault  11m

==> v1/StatefulSet
NAME   AGE
vault  11m

==> v1beta1/PodDisruptionBudget
NAME   AGE
vault  11m


NOTES:

Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get vault
```

### Task \#2:  
#### Init vault  

```sh
kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: ILnFVuaYPH3fJnoLbfMickokYoqyy/smQ1TPwQlcmFo=

Initial Root Token: s.6LFRpZuwwY9d3t9fVqfVIj1M

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

### Task \#3:  
#### Unseal vault  

```sh
for i in $(seq 0 2); do kubectl exec -it vault-$i -- vault operator unseal 'ILnFVuaYPH3fJnoLbfMickokYoqyy/smQ1TPwQlcmFo='; done
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.3.0
Cluster Name           vault-cluster-89c489dc
Cluster ID             9822c28c-9430-32d0-9d14-2576b8ab5d5e
HA Enabled             true
HA Cluster             n/a
HA Mode                standby
Active Node Address    <none>
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.3.0
Cluster Name           vault-cluster-89c489dc
Cluster ID             9822c28c-9430-32d0-9d14-2576b8ab5d5e
HA Enabled             true
HA Cluster             https://10.16.2.7:8201
HA Mode                standby
Active Node Address    http://10.16.2.7:8200
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.3.0
Cluster Name           vault-cluster-89c489dc
Cluster ID             9822c28c-9430-32d0-9d14-2576b8ab5d5e
HA Enabled             true
HA Cluster             https://10.16.2.7:8201
HA Mode                standby
Active Node Address    http://10.16.2.7:8200
```

### Task \#4:  
#### Login into vault  

```sh
kubectl exec -it vault-0 -- vault auth list
Error listing enabled authentications: Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/auth
Code: 400. Errors:

* missing client token
command terminated with exit code 2

kubectl exec -it vault-0 -- vault login 's.6LFRpZuwwY9d3t9fVqfVIj1M'
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.6LFRpZuwwY9d3t9fVqfVIj1M
token_accessor       UPoOIGT3CrskTJdgPTM0s8KQ
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]

kubectl exec -it vault-0 -- vault auth list
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_86d61e5a    token based credentials
```

### Task \#5:  
#### Create secrets  

```sh
kubectl exec -it vault-0 -- vault secrets enable --path=otus kv
kubectl exec -it vault-0 -- vault secrets list --detailed
kubectl exec -it vault-0 -- vault kv put otus/otus-ro/config username='otus' password='asajkjkahs'
kubectl exec -it vault-0 -- vault kv put otus/otus-rw/config username='otus' password='asajkjkahs'
kubectl exec -it vault-0 -- vault read otus/otus-ro/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus
kubectl exec -it vault-0 -- vault kv get otus/otus-rw/config
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
```

### Task \#6:  
#### Enable kubernetes auth  

```sh
kubectl exec -it vault-0 -- vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
kubectl exec -it vault-0 -- vault auth list
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_6a2fa02e    n/a
token/         token         auth_token_86d61e5a         token based credentials
```

### Task \#7:  
#### Create ClusterRoleBinding  

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: default
```

```sh
kubectl create serviceaccount vault-auth
kubectl apply -f vault-auth-service-account.yaml
```

### Task \#8:  
#### Prepare authorization, create policy and check it  

```sh
# Prepare variables for kubernetes auth config
export VAULT_SA_NAME=$(kubectl get sa vault-auth -o jsonpath="{.secrets[*]['name']}")
export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
export K8S_HOST=$(more ~/.kube/config | grep server |awk '/http/ {print $NF}')
### alternative way
export K8S_HOST=$(kubectl cluster-info | grep 'Kubernetes master' | awk '/https/ {print $NF}' | sed 's/\x1b\[[0-9;]*m//g' )

# what's the purpose of sed 's/\x1b\[[0-9;]*m//g' ?
# That's to remove control characters and colour codes from coloured output.
# macos here, no gnu sed, so switch to 
export K8S_HOST=$(kubectl cluster-info | grep 'Kubernetes master' | awk '/https/ {print $NF}' | sed $'s,\x1b\\[[0-9;]*[a-zA-Z],,g' )


# Now write config to vault
kubectl exec -it vault-0 -- vault write auth/kubernetes/config \
token_reviewer_jwt="$SA_JWT_TOKEN" \
kubernetes_host="$K8S_HOST" \
kubernetes_ca_cert="$SA_CA_CRT"
Success! Data written to: auth/kubernetes/config

# create policy file
tee otus-policy.hcl <<EOF
path "otus/otus-ro/*" {
    capabilities = ["read", "list"]
}
path "otus/otus-rw/*" {
    capabilities = ["read", "create", "list"]
}
EOF

# create policy and role
kubectl cp otus-policy.hcl vault-0:vault/
kubectl exec -it vault-0 -- vault policy write otus-policy /vault/otus-policy.hcl
kubectl exec -it vault-0 -- vault write auth/kubernetes/role/otus  \
bound_service_account_names=vault-auth         \
bound_service_account_namespaces=default policies=otus-policy  ttl=24h

# let's check authorization
# login and get client token
kubectl run alpine --serviceaccount=vault-auth --image=alpine --tty --rm -i
apk add curl jq
VAULT_ADDR=http://vault:8200
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST  --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq
TOKEN=`curl -s --request POST  --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | tr -d '"'`
# added tr to strip double

# read and right then
curl --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config
{"request_id":"f1428dd1-65ca-0b74-4c4a-ba3c3d949aa9","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"password":"asajkjkahs","username":"otus"},"wrap_info":null,"warnings":null,"auth":null}
curl --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
{"request_id":"4265ddf0-a314-8bb4-06bd-37ebeaae1835","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"password":"asajkjkahs","username":"otus"},"wrap_info":null,"warnings":null,"auth":null
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config
{"errors":["1 error occurred:\n\t* permission denied\n\n"]}
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
{"errors":["1 error occurred:\n\t* permission denied\n\n"]}
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config1
```

### Task \#9:  
#### Why do we have errors? Fix it  

We don't have rights to modify records  

```sh
# Q: why do we have errors?
# A: no rights to modify records
# let's change policy
cat otus-policy.hcl
path "otus/otus-ro/*" {
    capabilities = ["read", "list"]
}
path "otus/otus-rw/*" {
    capabilities = ["read", "create", "list", "update"]
}

kubectl cp otus-policy.hcl vault-0:vault/
kubectl exec -it vault-0 -- vault policy write otus-policy /vault/otus-policy.hcl
# repeat write
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
# now good
```

### Task \#10:  
#### Use case: authorization with exapmle  

```sh
# Use case: authorization via kubernetes with example
git clone https://github.com/hashicorp/vault-guides.git
cd vault-guides/identity/vault-agent-k8s-demo
# now change configs 
# change role name in vault-agent-config.hcl
vim configs-k8s/vault-agent-config.hcl
# Uncomment this to have Agent run once (e.g. when running as an initContainer)
exit_after_auth = true
pid_file = "/home/vault/pidfile"

auto_auth {
    method "kubernetes" {
        mount_path = "auth/kubernetes"
        config = {
            role = "otus"
        }
    }

    sink "file" {
        config = {
            path = "/home/vault/.vault-token"
        }
    }
}

# change secret name in consul-template-config.hcl
vim configs-k8s/consul-template-config.hcl
vault {
  renew_token = false
  vault_agent_token_file = "/home/vault/.vault-token"
  retry {
    backoff = "1s"
  }
}

template {
  destination = "/etc/secrets/index.html"
  contents = <<EOH
  <html>
  <body>
  <p>Some secrets:</p>
  {{- with secret "otus/otus-ro/config" }}
  <ul>
  <li><pre>username: {{ .Data.username }}</pre></li>
  <li><pre>password: {{ .Data.password }}</pre></li>
  </ul>
  {{ end }}
  </body>
  </html>
  EOH
}

# now run it
# Create a ConfigMap, example-vault-agent-config
kubectl create configmap example-vault-agent-config --from-file=./configs-k8s/
# View the created ConfigMap
kubectl get configmap example-vault-agent-config -o yaml
# Finally, create vault-agent-example Pod
kubectl apply -f example-k8s-spec.yml --record
# and we get error...
2019-12-16T07:48:48.782Z [ERROR] auth.handler: error authenticating: error="Put http://10.0.2.2:8200/v1/auth/kubernetes/login: dial tcp 10.0.2.2:8200: i/o timeout" backoff=1.737779293
# now change vault address in spec and rerun
# and finally works
kubectl exec -it vault-agent-example -c consul-template -- /bin/cat "/home/vault/.vault-token"
s.zUgWRYgDFeHp6LIuvp10ZXT
kubectl cp vault-agent-example:etc/secrets/index.html index.html
cat index.html
  <html>
  <body>
  <p>Some secrets:</p>
  <ul>
  <li><pre>username: otus</pre></li>
  <li><pre>password: asajkjkahs</pre></li>
  </ul>

  </body>
  </html>
```

```html
  <html>
  <body>
  <p>Some secrets:</p>
  <ul>
  <li><pre>username: otus</pre></li>
  <li><pre>password: asajkjkahs</pre></li>
  </ul>

  </body>
  </html>
```

### Task \#11:  
#### Create CA with vault's help  

```sh
# turn on pki secrets
kubectl exec -it vault-0 -- vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki
Success! Tuned the secrets engine at: pki/
kubectl exec -it vault-0 -- vault write -field=certificate pki/root/generate/internal common_name="exmaple.ru"  ttl=87600h > CA_cert.crt
# now input urls for SA and crl
kubectl exec -it vault-0 -- vault write pki/config/urls issuing_certificates="http://vault:8200/v1/pki/ca" crl_distribution_points="http://vault:8200/v1/pki/crl"
Success! Data written to: pki/config/urls
# create intermediate certificate
kubectl exec -it vault-0 -- vault secrets enable --path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki_int
Success! Tuned the secrets engine at: pki_int/
kubectl exec -it vault-0 -- vault write -format=json \
pki_int/intermediate/generate/internal common_name="example.ru Intermediate Authority" | jq -r '.data.csr' > pki_intermediate.csr
# now input intermediate cert into vault
kubectl cp pki_intermediate.csr vault-0:vault/ 
kubectl exec -it vault-0 -- vault write -format=json pki/root/sign-intermediate \
csr=@vault/pki_intermediate.csr format=pem_bundle ttl="43800h" |  jq -r '.data.certificate' > intermediate.cert.pem
kubectl cp intermediate.cert.pem vault-0:vault/
kubectl exec -it vault-0 -- vault write pki_int/intermediate/set-signed certificate=@vault/intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed
# now let's create and revoke new certs
# create role to make certs
kubectl exec -it vault-0 -- vault write pki_int/roles/example-dot-ru allowed_domains="example.ru" allow_subdomains=true   max_ttl="720h"
Success! Data written to: pki_int/roles/example-dot-ru
# create and revoke cert
kubectl exec -it vault-0 -- vault write pki_int/issue/example-dot-ru common_name="gitlab.example.ru" ttl="24h"
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUbDof6RR17sC6DT83TftPzbMdJZgwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0xOTEyMTYxMjQxMTdaFw0yNDEy
MTQxMjQxNDdaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPNs5JAWgygs
useLIUdSd9Cgh4KDOsj6hPXCg95sKaGbAkdJ8YCSwOFD092qvOH/A/pklUmNZEu9
dE++2QqmDHC1jQ7Yb+pnSs/9aaXoUgM8rmkP/jLwtGQygGZvQTysnfcuC9tTtI/D
CTs+jsZuhUCLRjhaR/sS9jbzrjTWLXTnllQYAuW04o1pNF/gqdqlJWXEgd35NiAi
VDjlwTlGYXb6bXqhcDtjFq4cNo863o5WyBXfXrcAiPeLx4OvbuatjQjs5AzvJN7h
yLVUNv/Qiqo9n3DvqrtyGpWT3oJDmHLrA8Wj+VKva1e5qctu8nt1enRZ0rbHA4CZ
eTfSTz3jwlECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUbBYpYimeHM0e5rn5aXdXC8jX3/YwHwYDVR0jBBgwFoAU
2Aaqapp3FS7ZSKOCxheEIA6FYF8wNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
iAfRnRfsJo9MKAcj3KyW4EBh6zNy3gVzQVltfipRWleAzp+ayQ9c9fu42NejTpg0
WGGFwm6+PcxI4HHiItt2wkC6lyaq25cVPbC1BS9G5HEFFbT2sIg3+lLRO2H3gtqA
zDCNxoFGN8dgWECHBjQK6mKMESCpMlgbFQ4q3HQBSUiHVeW5UtgTTX6O7BbouJgb
VCBdR2t12AVHRCz/yyEy+gDgl4EDBklALf31cwin+6hG3W/Mdcx7mYvfLujkxifi
uHg+uPbxutP6TdCCCtMffSBF9l9U86BsP3ly/zL7Ga9TdFaix0pjFx9+IwwpFd/h
HUiVCwhcqPpbEdTmuoQvVA==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDZzCCAk+gAwIBAgIUcZ0TBO3ZIkSBoT8hHBDxKgs5/b8wDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTE5MTIxNjEyNDYyNloXDTE5MTIxNzEyNDY1NlowHDEaMBgGA1UEAxMRZ2l0
bGFiLmV4YW1wbGUucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDA
gmlum0IWco1nLIKxW1+ejI9iSXqcR9FGmcP1GwsjURyMof1LyN/pGrOsaQ8OWeDw
Uk9cbYmnuRmQPW8oqkQobIMsa54Es7TkZMYeWf+0Tt2AJW4X2AkO5i89P8Zr7Q1y
Xuc2wbbaQQRjWvDE1BapGsi84WzSCppPGV8KULswnl8yuyJnrAJtOONJ0tmNr+KP
KNkqGgxlaf9qFLo1AmLjAnqXknYO9+Z6ctZ4UVJ4Hl2DSk7BtukBJAJOEI5Dt9zZ
5BTF76Knr6CRGLKf4mYOA6WGBTlRdII4ULgSHxbUwE97+COmP2KqV33Vc/0D5NdH
Meom/SDb7ns6JwKUBzpBAgMBAAGjgZAwgY0wDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQU5HNUpyxila5vXMf+
Nxy0ltlzB4owHwYDVR0jBBgwFoAUbBYpYimeHM0e5rn5aXdXC8jX3/YwHAYDVR0R
BBUwE4IRZ2l0bGFiLmV4YW1wbGUucnUwDQYJKoZIhvcNAQELBQADggEBAH61zTFY
2rjJJ7Gv+EC5eo01T0oT0ureCOCjrewVDMDdrF0HW+3xINzt1B11kNdFxjMwv/Q/
JE1JgK5fvy5f6Vum9Gl2Swwb57ivv3VGrfJW8ap2ZmvHptyO1YPUX//tFQKP3OJQ
UHNIVvTtxN3NGq9PwR8Mj7ZbYqK0FiQnIYd7RT1jQ5MauytMgGOrCzSIvQRMvrak
PhVtY1ShQaf5Bug8GGzSocCVTPW2NyQdr7JyP6RdaN7+qa7OkqFl4UypKvoGQXEA
0f93QZXaSoVq2/fHXbY4vnBnQ8wQCJRJeA2HcIs4lGwPzgtr/Y2czUmlboi0mK39
aDnX83nr91BcIVw=
-----END CERTIFICATE-----
expiration          1576586816
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUbDof6RR17sC6DT83TftPzbMdJZgwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0xOTEyMTYxMjQxMTdaFw0yNDEy
MTQxMjQxNDdaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPNs5JAWgygs
useLIUdSd9Cgh4KDOsj6hPXCg95sKaGbAkdJ8YCSwOFD092qvOH/A/pklUmNZEu9
dE++2QqmDHC1jQ7Yb+pnSs/9aaXoUgM8rmkP/jLwtGQygGZvQTysnfcuC9tTtI/D
CTs+jsZuhUCLRjhaR/sS9jbzrjTWLXTnllQYAuW04o1pNF/gqdqlJWXEgd35NiAi
VDjlwTlGYXb6bXqhcDtjFq4cNo863o5WyBXfXrcAiPeLx4OvbuatjQjs5AzvJN7h
yLVUNv/Qiqo9n3DvqrtyGpWT3oJDmHLrA8Wj+VKva1e5qctu8nt1enRZ0rbHA4CZ
eTfSTz3jwlECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUbBYpYimeHM0e5rn5aXdXC8jX3/YwHwYDVR0jBBgwFoAU
2Aaqapp3FS7ZSKOCxheEIA6FYF8wNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
iAfRnRfsJo9MKAcj3KyW4EBh6zNy3gVzQVltfipRWleAzp+ayQ9c9fu42NejTpg0
WGGFwm6+PcxI4HHiItt2wkC6lyaq25cVPbC1BS9G5HEFFbT2sIg3+lLRO2H3gtqA
zDCNxoFGN8dgWECHBjQK6mKMESCpMlgbFQ4q3HQBSUiHVeW5UtgTTX6O7BbouJgb
VCBdR2t12AVHRCz/yyEy+gDgl4EDBklALf31cwin+6hG3W/Mdcx7mYvfLujkxifi
uHg+uPbxutP6TdCCCtMffSBF9l9U86BsP3ly/zL7Ga9TdFaix0pjFx9+IwwpFd/h
HUiVCwhcqPpbEdTmuoQvVA==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAwIJpbptCFnKNZyyCsVtfnoyPYkl6nEfRRpnD9RsLI1EcjKH9
S8jf6RqzrGkPDlng8FJPXG2Jp7kZkD1vKKpEKGyDLGueBLO05GTGHln/tE7dgCVu
F9gJDuYvPT/Ga+0Ncl7nNsG22kEEY1rwxNQWqRrIvOFs0gqaTxlfClC7MJ5fMrsi
Z6wCbTjjSdLZja/ijyjZKhoMZWn/ahS6NQJi4wJ6l5J2DvfmenLWeFFSeB5dg0pO
wbbpASQCThCOQ7fc2eQUxe+ip6+gkRiyn+JmDgOlhgU5UXSCOFC4Eh8W1MBPe/gj
pj9iqld91XP9A+TXRzHqJv0g2+57OicClAc6QQIDAQABAoIBAByN+4eNfgMIYNMR
9hzKmedRoB8LGSW/PVqEil178m39pQdzK7gnBpdz/3yuZK5TRJtBCkaCdO2s9g7A
HhHhF5ULa3WWTO0TntxV2lE8NkKPhCly496ji8xq9kzWfd8aXWk+jHtBxpafGECI
h7gaYXYZ4/aoVVTef78F22QTT4DJaycotli4ulKy8FrY1e2HgpqQfzRsVEqHNBpj
9VzyCAaJnVuYMMySvMvbsggGHxmV9SEUFk0sLZbiFrmCWrEQ3f22l2yvmBZZ1rl3
jckbT+q1vit38p325D7V4O+GGxtWphjozWtgoTWNlj3OeEI3uaO2GY73Q5gG8ZDg
3xMr47ECgYEA4B36nyG/KFgBwF95Fk7m/3+1RR6y6wbLSsCNbRx9L0hClR+IadTN
eK8sbWGd2yFqtGSkmGMQxXOz540GsSMeBTs9Us7re2hP4cUUlu/PWLyon3zBUdHN
DuYEGWl8BP5ri8cjL+FGlrf3+OI0GEh6WuPk4zXQ3uMHaL4/BIBRv6UCgYEA2+VT
obRlwOVHfSTVx2WC9verJpK5F3H2g/QiGm4mh2m5SgJp8tIGJ/3FqbH3dY49gRBK
B/ln1k+jhS7uau3JWUAYJsRZAoMVIxJSjX04LvJZSeUu/gRk5aXUbx0qyEXJQgT4
mc4ikw8VVvxXuYZnkQXYg99SlhqmTKmBqCwETW0CgYBUqx6+taoZHL50pd0CD4b3
aZDa7xEa93Mf54TGfufQUBVPbx1DFjEV8d/v5twTKBm+0vLX2z0/y0lhJgcsLp8t
zMaLHT8bXTooiiMQLsL/vC5cKm6CcadthHpx+0buQAvzP6VMdmgLkq7s6NBTiDYp
VkVnjTI+sjhfWthF5BB+PQKBgEZpKxtXQVG/2OFIfy+G4KWl7ma+iofoVPAxpw3h
gXLQtqTtGvHGsHPzvWw18S/yKN1/0sS05rvn6ktGGM+iblumu1UGgB3ezVDamBZ4
JxpZPZ/8w8xQqeIi9F/T7hQMzIHYR6YwLD/8j2+4A3sDf3wfbBHl23L2+5MGn96y
oXoNAoGAT5X7muIqiJbiUG7yfPdkRxP04mM559YXYr2H4cX1so44iGWM5ZAYx1uA
dyfMtxE9tUOQBuPAsX0bFSbluEO8CrKWXVT9K3s4TFKEvemVmszmI1BqDaoHm3jU
Hl0+06RWhH1Xh5mCDRToOq+re/a4Jajlc1zSAhb0fHjD5bYkrHw=
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       71:9d:13:04:ed:d9:22:44:81:a1:3f:21:1c:10:f1:2a:0b:39:fd:bf


kubectl exec -it vault-0 -- vault write pki_int/revoke serial_number="71:9d:13:04:ed:d9:22:44:81:a1:3f:21:1c:10:f1:2a:0b:39:fd:bf"
Key                        Value
---                        -----
revocation_time            1576500518
revocation_time_rfc3339    2019-12-16T12:48:38.010849494Z
```
