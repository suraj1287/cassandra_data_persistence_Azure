# Cassandra Azure setup with data persistence using azure file storage

## Setup cassandra environment:

1. az login --service-principal -u  xxxx00000-0000-000-xxxx-000000000000 --password  pass@123 --tenant xxxx00000-0000-000-xxxx-000xxxxxxxx

2. az group create -l eastus -n myapp-dl

3. sudo az aks create --resource-group myapp-dl --name myapp-cass-aks --node-count 1 --nodepool-name myappcassaks --node-vm-size Standard_A4_v2 --enable-addons monitoring --service-principal xxxx00000-0000-000-xxxx-000000000000 --client-secret pass@123

4. az aks get-credentials --resource-group myapp-dl --name myapp-cass-aks --overwrite-existing

5. Create a fileshare:
az storage share create --name cassandra-data --account-name myapptest

6)Base64 encode the connection details for use in the Kubernetes Secret:(this has to be done in local machine using windows powershell)

```
$accountName = "myapptest"
$accountNameBytes = [System.Text.Encoding]::UTF8.GetBytes($accountName)
$accountNameBase64 = [Convert]::ToBase64String($accountNameBytes)
Write-Host "Account Name Base 64: " $accountNameBase64

$accountKey = "xxxx00000-0000-000-xxxx-000xxxxxxxx=="
$accountKeyBytes = [System.Text.Encoding]::UTF8.GetBytes($accountKey)
$accountKeyBase64 = [Convert]::ToBase64String($accountKeyBytes)
Write-Host "Account Name Key 64: " $accountKeyBase64

```

8. kubectl create -f cassandra-deployment.yaml

9. kubectl get svc

10. kubectl get pod

11. kubectl exec -it cassandra-56dc588f4-khtrg -- cqlsh

## Backup:

1. az aks get-credentials --resource-group myapp-test --name myapp-test-aks --overwrite-existing

2. kubectl get pod

3. kubectl exec -it myapp-backend-cassandra-6845d7597b-fhpbt -- /bin/bash

cd /home

## list snapshot

nodetool listsnapshots

## clear old snapshots
nodetool clearsnapshot -t 1552478139150 -- myapp

## back up schema
cqlsh -e "describe keyspace myapp" > myapp.cql

## snapshot keyspace to /var/lib/cassandra/data/myapp/*/snapshots/mybackup1
nodetool snapshot -t mybackup1 -- myapp

## tar gz
#FILES=$(find /var/lib/cassandra/data/ path /var/lib/cassandra/data/myapp/*/snapshots/mybackup1)
tar czf backup.tar.gz $(find /var/lib/cassandra/data/ path /var/lib/cassandra/data/myapp/*/snapshots/mybackup1) ./myapp.cql

## delete snapshot - we don't need it anymore
nodetool clearsnapshot -t mybackup1 -- myapp

## exit from pod

kubectl cp myapp-backend-cassandra-6845d7597b-fhpbt:/home/backup.tar.gz /home/dspg/myapp-test-data/

# Restore:

1. az aks get-credentials --resource-group myapp-dl --name myapp-cass-aks --overwrite-existing

2. kubectl get pod

3. kubectl cp /home/dspg/myapp-test-data/ myapp-backend-cassandra-6bffcb6c4-2fkc2:/home/
4. kubectl exec -it myapp-backend-cassandra-6bffcb6c4-2fkc2 -- /bin/bash
5. cd /home/
6. tar -xvf backup.tar.gz
7. cd /home/myapp-test-data/var/lib/cassandra/data/myapp

#to verify snapshots for single table one by one:

ls -d /home/myapp-test-data/var/lib/cassandra/data/myapp/*/snapshots/mybackup1
/home/myapp-test-data/var/lib/cassandra/data/myapp/abc-64cffd101a7c11e9ab97ef8d12218189/snapshots/mybackup1

## Restore keyspace and blank table structure:

cqlsh -e "SOURCE 'myapp.cql'";

## Copy files into original location:

cp /home/myapp-test-data/var/lib/cassandra/data/myapp/def-6815d9401a7c11e9ab97ef8d12218189/snapshots/mybackup1/* /var/lib/cassandra/data/myapp/def-a6bab2c0461d11e9985cf9f82ea3ea1e/
cp /home/myapp-test-data/var/lib/cassandra/data/myapp/ghi-75152e201a7c11e9ab97ef8d12218189/snapshots/mybackup1/* /var/lib/cassandra/data/myapp/ghi-ac7b2be0461d11e9985cf9f82ea3ea1e/
cp /home/myapp-test-data/var/lib/cassandra/data/myapp/emp-3a74a2a01a7c11e9ab97ef8d12218189/snapshots/mybackup1/* /var/lib/cassandra/data/myapp/emp-ba38c7b0461d11e9985cf9f82ea3ea1e/


## Import using sstableloader utility

nodetool status

replace ip in below command

sstableloader --nodes 10.244.2.3 --verbose /var/lib/cassandra/data/myapp/abc-c37c9b30461d11e9985cf9f82ea3ea1e/
sstableloader --nodes 10.244.2.3 --verbose /var/lib/cassandra/data/myapp/def-a6bab2c0461d11e9985cf9f82ea3ea1e/
sstableloader --nodes 10.244.2.3 --verbose /var/lib/cassandra/data/myapp/ghi-ac7b2be0461d11e9985cf9f82ea3ea1e/
sstableloader --nodes 10.244.2.3 --verbose /var/lib/cassandra/data/myapp/emp-ba38c7b0461d11e9985cf9f82ea3ea1e/


## Verify count of all tables:

1. select count(*) from abc;
2. select count(*) from def;
3. select count(*) from ghi;
4. select count(*) from emp;


