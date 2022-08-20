## Postgres Cluster Setup

## Storage Class
Set local storage
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
```sh
# check it
kubectl get sc
```

OpenEBS  
pre-requisites for Ubuntu  
```sh
sudo cat /etc/iscsi/initiatorname.iscsi
systemctl status iscsid 
# if the service status is shown as Inactive, use the following command
sudo systemctl enable --now iscsid
# check status again
systemctl status iscsid

# if iSCSI initiator is not installed on your node
sudo apt-get update
sudo apt-get install open-iscsi
sudo systemctl enable --now iscsid
```
installation through kubectl  
```sh
# install
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml

# verify
kubectl get pods -n openebs
```
set openebs-hostpath as defalut storage class
```sh
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```


## Postgres Operator

Create postgres operator  

```sh
# First, clone the repository and change to the directory
git clone https://github.com/zalando/postgres-operator.git
cd postgres-operator

# apply the manifests in the following order
kubectl create -f manifests/configmap.yaml  # configuration
kubectl create -f manifests/operator-service-account-rbac.yaml  # identity and permissions
kubectl create -f manifests/postgres-operator.yaml  # deployment
kubectl create -f manifests/api-service.yaml  # operator API to be used by UI
```

Check if Postgres Operator is running  
```sh
kubectl get pod -l name=postgres-operator
```
If the operator doesn't get into Running state, either check the latest K8s events of the deployment or pod with kubectl describe or inspect the operator logs:
```sh
kubectl logs "$(kubectl get pod -l name=postgres-operator --output='name')"
```

## Postgres Cluster

Create postgres cluster
postgres_cluster.yaml
```yaml
kind: "postgresql"
apiVersion: "acid.zalan.do/v1"

metadata:
  name: "postgres-cluster-test"
  namespace: "default"
  labels:
    team: acid

spec:
  teamId: "acid"
  postgresql:
    version: "14"
  numberOfInstances: 4
  enableMasterLoadBalancer: true
  enableReplicaLoadBalancer: true
  enableConnectionPooler: true
  volume:
    size: "5Gi"
  users:
    admin:
      - superuser
      - createdb
  databases:
    test: admin
    solr: admin
  allowedSourceRanges:
    # IP ranges to access your cluster go here


  resources:
    requests:
      cpu: 100m
      memory: 100Mi
    limits:
      cpu: 500m
      memory: 500Mi
```

```sh
kubectl create -f postgres_cluster.yaml

# check the deployed cluster
kubectl get postgresql

# check created database pods
kubectl get pods -l application=spilo -L spilo-role

# check created service resources
kubectl get svc -l application=spilo -L spilo-role
```
Connect to postgres database via psql  
```sh
# get secret name
kubectl get secret -A

# get secret
kubectl get secret postgres.acid-postgrescluster.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d

# set password and sslmode
export PGPASSWORD=$(kubectl get secret postgres.acid-postgrescluster.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d)
export PGSSLMODE=require

# install psql
sudo apt install postgresql-client-12

# connect
# check postgres-master distribued in which node in dashboard
psql -U postgres -h 129.69.209.194 -p 30430

# change password in postgres
ALTER USER postgres WITH PASSWORD 'new-password';

```
