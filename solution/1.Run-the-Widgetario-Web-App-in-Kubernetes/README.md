# Milestone 1 Solution

## Step 1

All the application components are modelled as Pods in the `pods` folder:

- [products-db.yaml](pods/products-db.yaml) is the products database 
- [products-api.yaml](pods/products-api.yaml) is the products API
- [stock-api.yaml](pods/stock-api.yaml) is the stock API  
- [web.yaml](pods/web.yaml) is the website

They're all very similar - [products-api.yaml](pods/products-api.yaml) is annotated so you can see why the values are used.

## Step 2

Deploy the Pods to your cluster, using Kubectl to apply all the YAML files in the folder:

```
kubectl apply -f ./pods/
```

> You can also apply files independently, with a command like `kubectl apply -f pods/products-dbs.yaml -f pods/products-api.yaml...`. Or you can put multiple resources into a single YAML file, separating them with `---`.

## Step 3

Nice and easy, you can list all the Pods with:

```
kubectl get pods
```

Or you can have Kubectl monitor the Pods and update you with status changes using the `watch` flag:

```
kubectl get pods --watch
```

> Kubernetes has short aliases for common resource types, so instead of `get pods` you can also use `get po`.

## Step 4

The Pod name in the YAML metadata is how you refer to the object in Kubectl, so you can get the logs for the stock API Pod with:

```
kubectl logs stock-api
```

My Pod definitions also include a label for every Pod, identifying them all as part of the Widgetario app. You can print the logs for every Pod with a matching label like this:

```
kubectl logs -l app=widgetario
```

> You'll see lots of error from the Java products API, telling you it can't reach the database. That's because we haven't set up networking between components yet. The web application doesn't print any logs - that's something we can deal with later.

## Step 5

Stopping the application process cause a container to exit. When that container is running in a Pod, the Pod sees it has exited and restarts it.

This is how to stop the stock API application:

```
kubectl exec stock-api -- kill 1 
```

Now list the Pods again:

```
kubectl get po
```

> You'll see the stock API Pod has been restarted - that's because you stopped the container process. The products API has been failing too because it can't reach the database, and it's probably in the status CrashLoopBackOff - which means Kubernetes will keep trying to restart it, but with an increasing pause between each attempt.
