# Milestone 1 

## Step 1

Get the app running from the YAML in the `Data` folder:

```
kubectl apply -f ./widgetario/
```

> Check the app at http://localhost:8080 or http://localhost:30080

You should see this:

![](../Images/step-1.png)

## Steps 2-5

All the healthchecks are modelled as readiness probes and liveness probes in the Deployment specs:

- [products-api.yaml](deployments/products-api.yaml) - uses HTTP probes pointing to the root path
- [stock-api.yaml](deployments/stock-api.yaml) - uses HTTP probes pointing to `/healthz`
- [web.yaml](deployments/web.yaml) - HTTP probes pointing to the root path
- [products-db.yaml](deployments/products-db.yaml) - uses a TCP probe over the Postgres port 5432 and an exec probe to run the `pg_isready` command


Deploy these updates and you'll see new Pods created - the old ones won't be removed until the container probes in the new Pods pass:

```
kubectl apply -f deployments

kubectl get po --watch
```

Describe the Pods to confirm the probes are configured:

```
# e.g. for the database Pod
kubectl describe po -l app=widgetario,component=products-db
```

In the output you'll see readiness and liveness probes described, like this:

```
    Ready:          True
    Restart Count:  0
    Liveness:       exec [pg_isready -h localhost] delay=20s timeout=1s period=60s #success=1 #failure=2
    Readiness:      tcp-socket :postgres delay=10s timeout=1s period=10s #success=1 #failure=1
```

> Note that the numbers in the output are not the current count of success or failure - these are the confgiured thresholds. The Pod events, status and restart count show the probes in action.

## Step 6

The updated Deployment specs provided named ports for all the containers, and use the names in the probes. The Service specs should also be updated to use the port names:

- [products-api-service.yaml](services/products-api-service.yaml) 
- [products-db-service.yaml](services/products-db-service.yaml) 
- [stock-api-service.yaml](services/stock-api-service.yaml) 
- [web-service.yaml](services/web-service.yaml) 

> You can choose your own names. This solution uses the network protocol, e.g. `http` or `postgres`.

The port numbers haven't changed, so the app still works in the same way with the updated Deployments, but changing the Services will make the configuration more flexible:

```
kubectl apply -f services

kubectl get endpoints -l app=widgetario
```

> You'll see the endpoint list includes the target port number. If the named port in the Service doesn't exist in the container, you won't get an error - but there will be no endpoints for the Service.