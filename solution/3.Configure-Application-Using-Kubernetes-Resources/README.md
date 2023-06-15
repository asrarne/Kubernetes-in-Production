# Milestone 3

## Steps 1-4

The config objects are all modelled as YAML in the solution folder.

Non-sensitive data is stored in ConfigMaps:

- [products-api-properties.yaml](configMaps/products-api-properties.yaml) for the products API 
- [web-logging.yaml](configMaps/web-logging.yaml) for the web app

And sensitive data is stored in Secrets:

- [products-api-db.yaml](secrets/products-api-db.yaml) for the products API 
- [products-db-password.yaml](secrets/products-db-password.yaml) for the products database 
- [stock-api-connection.yaml](secrets/stock-api-connection.yaml) for the stock API 
- [web-api.yaml](secrets/web-api.yaml) for the web app

The interesting parts of the specs are annotated.

Apply them in the usual way:

```
kubectl apply -f secrets/ -f configMaps/
```

## Steps 1-4: ALTERNATIVE

The original config files in the `Data` folder can be used to create the config objects:

_Products database - using the env file:_

```
kubectl create secret generic products-db-password-2 --from-env-file Data/products-db/password.env
```

_Products API - using the properties files:_

```
kubectl create configmap products-api-properties --from-file Data/products-api/application.properties

kubectl create secret generic products-api-db --from-file Data/products-api/db/application.properties
```

_Stock API - using the env file:_

```
kubectl create secret generic stock-api-connection2 --from-env-file Data/stock-api/connection-string.env
```

_Web app - using the JSON files:_

```
kubectl create configmap web-logging --from-file Data/web/logging.json

kubectl create secret generic web-api --from-file Data/web/api.json
```

## Step 5

The Pod specs need to be updated to load app configuration from the ConfigMaps and Secrets:

- [products-db.yaml](pods/products-db.yaml) loads the Secret as environment variables
- [products-api.yaml](pods/products-api.yaml) loads config objects as volumes and mounts them in the container
- [stock-api.yaml](pods/stock-api.yaml) loads the Secret as environment variables
- [web.yaml](pods/web.yaml) loads config objects as volumes and mounts them in the container, and an extra config setting in an environment variable

Ensure your config objects are all created:

```
kubectl get configmap 

kubectl get secret
```

Then apply the new Pod specs:

```
kubectl apply -f pods/
```

> If your Pods are already running, you will see errors - the volume specification cannot be changed in an existing Pod. 

You'll need to delete the application Pods first - the label comes in handy here:

```
kubectl delete pods -l app=widgetario
```

Then redeploy:

```
kubectl apply -f pods/
```

> Browse to the app again and you'll see the new dark theme, with all the original data loading into the homepage.

**If you think that deleting Pods and recreating them isn't a great approach - you're absolutley right, and we'll address that in the next milestone**

## Troubleshooting

If the web app doesn't load properly, you'll need to check the config settings have been applied correctly.

You can use `kubectl describe` to check the contents of a ConfigMap, printing the raw data:

```
kubectl describe configmap products-api-properties
```

Secrets are encoded as base64 strings, so to view the plaintext of a Secret you need to fetch the data item and decode the output:

```
kubectl get secret stock-api-connection -o jsonpath='{.data.POSTGRES_CONNECTION_STRING}' | base64 -d
```

If the contents of the config objects is correct, then you need to check they're being loaded into the Pod containers.

You can print environment variables like this, which should show the plaintext values:

```
kubectl exec products-db -- printenv POSTGRES_PASSWORD

kubectl exec stock-api -- printenv POSTGRES_CONNECTION_STRING
```

And you can print the contents of files like this:

```
kubectl exec products-api -- cat /app/config/application.properties

kubectl exec products-api -- cat /app/config/db/application.properties

kubectl exec web -- cat /app/secrets/api.json
```

> Those commands use the expected file paths and environment variable names, so if you get errors or unexpected contents - that will be the problem.