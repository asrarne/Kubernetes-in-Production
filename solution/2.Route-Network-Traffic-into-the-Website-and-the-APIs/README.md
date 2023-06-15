# Milestone 2 Solution

## Steps 1-4

All the network components are modelled as Services in the `services` folder.

The internal components all use ClusterIP Services:

- [products-db.yaml](services/products-db.yaml) for the products database 
- [products-api.yaml](services/products-api.yaml) for the products API
- [stock-api.yaml](services/stock-api.yaml) forthe stock API  

The external web component uses a LoadBalancer Service:

- [web.yaml](services/web.yaml) is the website

But if you can't use LoadBalancer in your environment, there's also a NodePort alternative:

- [web.yaml](services/web-nodePort.yaml) is the website using a NodePort

The interesting parts of the spec are annotated.

## Step 5

You should have the Pods running from milestone 1.

Now deploy all the Services:

```
kubectl apply -f ./services/
```

Print the Service list to see all the cluster Services:

```
kubectl get services
```

> You'll see the Kubernetes API server also has a Service - that's so applications running in Pods can use the Kubernetes REST API if they need to.

Print the Services - `svc` is the alias - filtering on the Widgetario app label:

```
kubectl get svc -l app=widgetario
```

> You'll see the external IP address for the Web Service if you use a LoadBalancer. Browse to the port on that IP (or if you used NodePort browse to the node port on your machine's IP) to see the web app.

## Troubleshooting

Check all the Pods are running, and print the IP address for each Pod:

```
kubectl get po products-api -o wide
```

> If the products API Pod is in the CrashLoopBackOff state, use `kubectl delete pod` to remove it and `kubectl apply` to recreate it.

Check the endpoints in each Service, to make sure the Service is linked to the correct Pod - e.g. for the products API:

```
kubectl get endpoints products-api
```

> There should be one endpoint for each Service, matching the IP address of the Pod for that component.
