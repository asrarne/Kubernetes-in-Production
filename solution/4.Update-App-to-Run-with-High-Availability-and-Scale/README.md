# Milestone 4

## Steps 1-3

The Deployment objects are all modelled as YAML in the `Deployments` folder:

- [products-db.yaml](deployments/products-db.yaml) for the products database 
- [products-api.yaml](deployments/products-api.yaml) for the products API 
- [stock-api.yaml](deployments/stock-api.yaml) for the stock API 
- [web.yaml](deployments/web.yaml) for the web app

My solution copies the Pod spec for each component from milestone 3 and uses it as the template in the Deployment spec. 

The only changes are to remove the Pod name - controllers manage the names of their object - and increase the indentation.

## Step 4

You can create your Deployments if you already have the app running, but that could confuse things. You'll have several Pods matching the label selectors for your Services, and you won't know which responses are coming from the new Deployment Pods.

Better to delete all the existing Pods, identifying them with the app selector:

```
kubectl delete pods -l app=widgetario
```

Now you can create the Deployments:

```
kubectl apply -f deployments/
```

This will creat new Pods, and if you kept your specs consistent they'll have the same labels so you can manage them in the same way:

```
kubectl get pods -l app=widgetario
```

Pods created by a controllaer have random names, so you'll use labels to work with individual Pods:

```
# print the logs of the products API:
kubectl logs -l component=products-api

# delete the stock API:
kubectl delete pod -l component=stock-api

# verify the Deployment creates a replacement stock API Pod:
kubectl get pods -l app=widgetario
```

> Browse to the app at the usual endpoint for your web Service, and you should see the same output from milestone 3.

# Step 5

The YAML specs in the `scale` folder add the replica settings. You wouldn't normally have copies of the YAML files like this, you would have one set of manifest files which live in source control and evolve over time.

- [products-api.yaml](scale/products-api.yaml) - runs two replicas
- [stock-api.yaml](scale/stock-api.yaml) - runs two replicas
- [web.yaml](scale/web.yaml) - runs three replicas

This is an update to the existing Deployments. Kubernetes can manage the change for the APIs without replacing Pods, because only the scale setting has changed. For the web component there's also a new environment variable - so that's a change to the Pod spec and the web Pod will be replaced.

Make the update in the usual way, and watch the Pods to see the new resources being created:

```
kubectl apply -f scale/

kubectl get po -l app=widgetario --watch
```

> Browse to the site again and you'll see the Pod name shown in the homepage. Refresh your browser and you'll see the same website but with a different Pod name - the requests are load-balanced between the web Pods. (Refresh without the cache if you don't see that e.g. with Ctrl-F5 with Firefox on Windows).

## Step 5: ALTERNATIVE

You can also scale Deployments directly with Kubectl:

```
kubectl scale deployment products-api --replicas 2

kubectl scale deployment stock-api --replicas 2

kubectl scale deployment web --replicas 3
```

The result is the same, but now your YAML is out of sync with the Deployments in your cluster.

It's good to know about this as a fast way of scaling components, but a YAML update is a better process.

## Step 6

Releasing an update to an app component is as simple as changing the image in the Pod spec (assuming you have a new container image which is tested and ready to deploy). My update is in the `v2` folder:

- [web.yaml](v2/web.yaml) - updates to a new image version

Deploy the change in the usual way and watch the Pod rollout:

```
kubectl apply -f v2/

kubectl get po -l component=web --watch
```

You'll see the Pods get replaced with new ones - a change to the container image changes the Pod spec which means the Deployment replaces them.

> Browse to the site now and you'll see there's a "buy" button for every product that's in stock. The button doesn't do anything, but it's a step closer to us turning a profit.

## Step 6: ALTERNATIVE

You can also update the container image using `kubectl set`, specifying the name of the Deployment, and the new image to use for the Pod container (I typically call my containers `app`): 

```
kubectl set image deploy/web app=widgetario/web:21.03-v2
```

It's useful to know you can do this, but again it causes drift between the running application and the YAML model. It's better if the YAML is the golden source so you should use `set` sparingly.

## Troubleshooting

The main thing here is to get the label selector correct in your Deployment manifests. If the Pod spec doesn't contain the labels you're using in the selector, you'll get an error like this from `kubectl apply`:

```
The Deployment ... is invalid: ... `selector` does not match template `labels`
```

The labels in your Pod spec need to exactly match (or be a superset of) the labels in the Deployment selector.

If your app deploys correctly but you can't see how Deployments and Pods are related, you can track them using Kubectl:

```
# the deployment shows the number of ready Pods:
kubectl get deployment web

# the ReplicaSets show the number of desired and ready Pods:
kubectl get replicaset -l component=web
```

And if the app doesn't function correctly, it's likely to be an issue with your Services - maybe the label selectors in the Service don't match the ones in your new Pods.

You can check the Service endpoints, and you should see your Pod IP addresses:

```
kubectl get po -l component=web -o wide

kubectl get endpoints web
```