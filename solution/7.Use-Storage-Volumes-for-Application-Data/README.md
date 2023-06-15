# Milestone 3 Solution

## Step 1

The sample solution uses an emptyDir volume in the Pod spec, with an init container to create the log file and a sidecar container to relay the logs out from the app container:

- [deployments/web.yaml](deployments/web.yaml)

Before you make the change, confirm that logs are being written by the app inside the container filesystem:

```
kubectl exec deploy/web -- cat /logs/app.log
```

You can deploy this update independently:

```
kubectl apply -f ./deployments/web.yaml
```

> Browse to the app at http://localhost:8080 or http://localhost:30080

Check the logs:

```
kubectl logs -l component=web -c logger
```

You'll see application logs in this format:

```
2021-06-23 13:03:22.592 +00:00 [INF] Request finished in 0.2983ms 200
```

## Step 2

The Stock API writes files to a local cache directory, to save database calls for every request. Using a read-only filesystem means the cache write fails; the solution uses a memory-backed emptyDir for the cache location:

- [deployments/stock-api.yaml](deployments/stock-api.yaml)

Check the Stock API logs to see the error:

```
kubectl logs -l component=stock-api
```

Update the API Deployment:

```
kubectl apply -f ./deployments/stock-api.yaml
```

Verify you can write files to the cache directory, but the rest of the filesystem is read-only:

```
# this should fail:
kubectl exec deploy/stock-api -- touch /app/newfile

# this should work:
kubectl exec deploy/stock-api -- touch /cache/newfile
```

> Refresh your browser when the new Pods are ready.

Check the logs again - you'll see the API is caching responses:

```
kubectl logs -l component=stock-api
```

## Step 3

Nothing new here. The sample solution includes a LoadBalancer and a NodePort Service, so it works across different Kubernetes distributions.

- [services/stock-api.yaml](services/stock-api.yaml)

Deploy the update:

```
kubectl apply -f ./services/stock-api.yaml
```

Now you can call the API directly:

```
curl localhost:8088/stock/1  

# OR - using the NodePort
curl localhost:30088/stock/1  
```

And make a PUT request to update the stock level:

```
# Linux or MacOS
curl -XPUT http://localhost:8088/stock/1 -d '{"stock" : 0}'

# OR on Windows
curl -XPUT http://localhost:8088/stock/1 -d '{\"stock\" : 0}'
```

> Refresh the browser a few times - when all the Pod caches have cleared, the Arm64 SoC will show as out of stock

## Step 4

A simple PVC will do, together with an updated Pod spec to load the PVC into the container filesystem at the Postgres data directory:

- [pvcs/db-data-pvc.yaml](pvcs/db-data-pvc.yaml)
- [deployments/products-db.yaml](deployments/products-db.yaml)

Before you make the change, verify that the database changes you made in Step 3 don't persist through a Pod replacement:

```
kubectl rollout restart deploy/products-db
```

> Refresh the web page and when the caches have been emptied, you'll see the Arm64 product is back in stock - the new database Pod loads the original data

For this update, the PVC should be created before the updated Pod deployment:

```
kubectl apply -f pvcs -f deployments

kubectl get po -l app=widgetario --watch
```

When the new Pods are online, update the stock again:

```
# Linux or MacOS
curl -XPUT http://localhost:8088/stock/1 -d '{"stock" : 0}'

# OR on Windows
curl -XPUT http://localhost:8088/stock/1 -d '{\"stock\" : 0}'
```

> Refresh the site to confirm the product is out of stock again

Now repeat the rollout:

```
kubectl rollout restart deploy/products-db
```

> Refresh the site and confirm the product is still out of stock - the new database Pod is using the data written by the previous Pod
