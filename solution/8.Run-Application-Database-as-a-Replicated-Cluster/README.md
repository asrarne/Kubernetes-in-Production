# Milestone 4 Solution

## Steps 1 & 2

The sample solution uses a StatefulSet as the database Pod controller:

- [statefulset/products-db-service.yaml](deployments/products-db-service.yaml) - headless Service for the database, using the DNS name `products-db`

- [statefulset/products-db-statefulset.yaml](deployments/products-db-statefulset.yaml) - the completed StatefulSet, with volume claim templates for Pod PVCs

Remove the existing database controller and Service:

```
kubectl delete svc products-db

kubectl delete deploy products-db
```

Deploy the new StatefulSet and Service:

```
kubectl apply -f ./statefulset/
```

Confirm the Pods are running and Pod 0 is the database primary:

```
kubectl get po -l component=products-db

kubectl logs products-db-0 | grep 'system is ready'
```

> You'll see output like this (among various other messages):

```
** Postgres primary **
...
2021-03-30 14:49:22.236 UTC [12] LOG:  database system is ready to accept connections
```

Confirm that Pod 1 is running as the secondary:

```
kubectl logs products-db-1 | grep 'from primary'
```

> You'll see output like this:

```
** Postgres standby - initializing replication**
...
2021-03-30 14:49:24.585 UTC [35] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```

Confirm that each Pod has its own PVC:

```
kubectl get pvc

kubectl describe pod products-db-0
```

You can test the database with a port-forward to the primary Pod:

```
kubectl port-forward pod/products-db-0 5532:5432
```

With your SQL client you can connect and confirm you can make changes:

```
UPDATE "public"."products"
SET stock = 1
WHERE id = 4
```

Try a port-forward to the secondary Pod:

```
kubectl port-forward pod/products-db-1 5532:5432
```

> Your changes have been replicated, but you can't update data with this connection

## Steps 3 & 4

Using configuration objects with a version number (or date) in the name lets you rollback to previous configuration settings:

- [secrets/products-api-v2.yaml](secrets/products-api-v2.yaml) - creates a v2 Secret which uses Pod 1 - the database replica - for the data connection

- [deployments/products-api.yaml](deployments/products-api.yaml) - updates the Pod spec to use the v2 Secret

- [secrets/stock-api-v2.yaml](secrets/stock-api-v2.yaml) - creates a v2 Secret which uses Pod 0 - the database primary - for the data connection

- [deployments/stock-api.yaml](deployments/stock-api.yaml) - updates the Pod spec to use the v2 Secret

Deploy the new configuration objects and update the Deployments:

```
kubectl apply -f ./secrets/ -f ./deployments/
```

Confirm you can use the Stock API to get data:

```
curl http://localhost:8088/stock/2
```

And to update data:

```
# Linux or MacOS
curl -XPUT http://localhost:8088/stock/2 -d '{"stock" : 0}'

# OR on Windows
curl -XPUT http://localhost:8088/stock/2 -d '{\"stock\" : 0}'
```

> Try the app to confirm both APIs can read data and present it to the website

## Step 5

Scaling up the database is now as simple as increasing the replica count. Load-balancing across specific Pods in a StatefulSet isn't possible, so the Products API needs to be explicitly configured to use the two replicas:

- [replicated-secondaries/products-db-statefulset.yaml](replicated-secondaries/products-db-statefulset.yaml) - uses 3 replicas for the database

- [replicated-secondaries/products-api-v3.yaml](replicated-secondaries/products-api-v3.yaml) - creates a v3 Secret which uses Pod 1 and Pod 2 in the connection string, together with the`loadBalanceHosts` flag

- [replicated-secondaries/products-api.yaml](deployments/products-api.yaml) - updates the Pod spec to use the v3 Secret

Deploy the changes:

```
kubectl apply -f ./replicated-secondaries/
```

Confirm Pod 2 starts as a database secondary:

```
kubectl logs products-db-2 | grep 'from primary'
```

> You'll see output like this:

```
** Postgres standby - initializing replication**
...
2021-03-30 14:49:24.585 UTC [35] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```

Confirm that the new Pod has its own PVC:

```
kubectl get pvc

kubectl describe pod products-db-2
```

> Test the website with lots of refreshes, to confirm the Products API Pod can read data from both secondaries