# Milestone 2

## Steps 1-4

All the Deployments set the Pod spec with `automountServiceAccountToken: false`. That stops the API token being mounted, so attackers need to work harder to authenticate with the Kubernetes API server.

Each Pod spec adds the extra security specified for that component - they all use different application platforms with different capabilities, so we can't use a the same security profile for all of them.

- [products-db.yaml](solution\deployments\products-db.yaml) - just removes the API token mount

- [products-api.yaml](deployments\products-api.yaml) - sets CPU and memory limits for the container

- [stock-api.yaml](deployments\stock-api.yaml) - runs the container with a read-only filesystem

- [web.yaml](deployments\web.yaml) - runs as a non-root user with all Linux capabilities dropped, configures the app to listen on port 5001 and changes the port number in the container spec.

> This is why port names are a good idea - the application port changes for the web container, but all other components reference the port name so they don't need to change.

Deploy the updates and watch the Pods, checking that the health probes pass in the new Pods so they enter the ready state and the old Pods are removed:

```
kubectl apply -f deployments/

kubectl get po -l app=widgetario --watch
```

> Try the app - it's working, but there's a problem...

Check the logs for the Stock API:

```
kubectl logs -l component=stock-api
```

> The API seems to be doing something, it's fetching information from the database - but it's not able to cache the response. Perhaps this component doesn't work so well with a read-only filesystem. We'll fix that in the next milestone.