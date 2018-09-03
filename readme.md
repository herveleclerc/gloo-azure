# AZURE FUNCTIONS WITH GLOO

### What you'll need

- `Kubernetes` v1.8+ deployed somewhere. Minikube is a great way to get a cluster up quickly.
- `kubectl` to interact with kubernetes.
- `glooctl` to interact with gloo.
- `azure` account

# Setup the environment

## Install Kubernetes

```shell
minikube start --extra-config=apiserver.Authorization.Mode=RBAC --cpus 4 --memory 4096
kubectl create clusterrolebinding permissive-binding \
         --clusterrole=cluster-admin \
         --user=admin \
         --user=kubelet \
         --group=system:serviceaccounts
```

## Install Gloo
```shell
glooctl install kube
```

Wait \ Verify that all the pods are in Running status:
```
kubectl get pods --all-namespaces
```

## Get the url of the ingress
If you installed kubernetes using minikube as mentioned above, you can use this command:
```shell
export GATEWAY_URL=http://$(minikube ip):$(kubectl get svc ingress -n gloo-system -o 'jsonpath={.spec.ports[?(@.name=="http")].nodePort}')
```


## Steps

### Create a directory for your project

```shell
mkdir joke-gen
cd joke-gen
```

### Create an azure resource group via azure portal

![create-resource-group.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/AF9E4FC0CC209E81AB244E1010E39DC6.png)

### Create an azure function app in the resource group

![create-functions-app.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/94E4477C7D4FAD123BD5A6D3B0E67CA3.png)

Select functions app from list

![functions-app.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/08A7FF7654A4E158E0EE45BCD326F2E6.png)

* `Click create`


![create-function-app-form.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/FCECB2E9BE95FB554EA0DF93E92476ED.png)

- Give an application name
- Select your resource group previously created
- Give a name for storage account

* `Click create`

### Get publish profile

![publish-profile.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/8CC9BC5B600DAE6E85A1BFFF3DFF7A0A.png)


* `Click Get publish profile`
* `Save it in previously created directory`

You should have file with a xml content looking like this

![publish-setings.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/A2D3A1FD2D3B29DA8F559E413B5B2BC4.png)

Create a very simple script to create kubernetes secret containing the publish profile data

(adjust the name of the secret in metadata)

```shell
#!bin/bash
DATA=$(cat $1 | base64)

tee $2 <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: joke-gen
data:
  publish_profile: $DATA
EOF
```

### Generate the secret yaml file
```
bash gen-secret.sh publish-profile-filename yaml-secret-file-name
# eg: bash gen-secret.sh gloo-azure-app.PublishSettings secret.yaml
```

You should get something like this

![secret.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/58D493AE4C751A5C55D56E1C5B219DEF.png)


### Create the secret

Note - Don't forget to create secret in `gloo-system` namespace

```shell
# create
kubectl apply -f secret.yaml -n gloo-system

# check kubernetes
kubectl get secret -n gloo-system
+--------------------+------------------------------------+-------+-------+
|  NAME              |   TYPE                             | DATA  |   AGE |
+--------------------+------------------------------------+-------+-------+
|default-token-jrhbj | kubernetes.io/service-account-token|   3   |   3d  |
|joke-gen   Opaque   |                                    |   1   |   1m  |
+--------------------+------------------------------------+-------+-------+

# check gloo 
glooctl secret get

+---------------------+---------+-----------+
|        NAME         |  TYPE   | IN USE BY |
+---------------------+---------+-----------+
| default-token-jrhbj | Unknown |           |
| joke-gen            | Unknown |           |
+---------------------+---------+-----------+
```


### Create an azure function 

For example a function that retrieve Chuck Norris jokes from an api

![function-editor.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/59CD9F026AC71A57C50C83A964B6D627.png)

## Save and test the function

![test-func.png](https://github.com/herveleclerc/gloo-azure/blob/master/resources/95C7F2297825B092BE39FE84EE29F276.png)


---


### Create an upstream 

* Create a file (upstream.yaml) with this content 

```yaml
name: joke-gen-upstream
type: azure
spec:
  secret_ref: joke-gen
  function_app_name: gloo-azure-app
metadata:
  annotations:
    gloo.solo.io/azure_publish_profile: joke-gen
```

where 

- `name` is the name of the upstream
- `secret_ref`is the name of the secret
- `function_app_name`is the name of the Azure Functions App
- `gloo.solo.io/azure_publish_profile` is the name of the secret. this annotation is mandatory


* Create the upstream 

```shell
glooctl upstream create -f upstream.yaml

+-------------------+-------+--------+----------+
|       NAME        | TYPE  | STATUS | FUNCTION |
+-------------------+-------+--------+----------+
| joke-gen-upstream | azure |        |          |
+-------------------+-------+--------+----------+
```

* Check functions discovery is ok

```shell
glooctl upstream get joke-gen-upstream
+-------------------+-------+--------+----------+
|       NAME        | TYPE  | STATUS | FUNCTION |
+-------------------+-------+--------+----------+
| joke-gen-upstream | azure |        | joke-gen |
+-------------------+-------+--------+----------+
```

You should see in column the names of your functions in azure function app

### Create a route to the function

```shell
glooctl route create --sort \
   --upstream joke-gen-upstream \
   --function 'joke-gen' \
   --path-exact /joke

+----+------------+-------------+------+--------+-----------------------+----------+-----------+
| ID |  MATCHER   |    TYPE     | VERB | HEADER |         UPSTREAM      | FUNCTION | EXTENSION |
+----+------------+-------------+------+--------+-----------------------+----------+-----------+
| 1  | /joke      | Exact Path  | *    |        | joke-gen-upstream     | joke-gen |           |
+----+------------+-------------+------+--------+-----------------------+----------+-----------+
   
```

### Get the url of the ingress

If you installed kubernetes using minikube, you can use this command:

```shell
export GATEWAY_URL=http://$(minikube ip):$(kubectl get svc ingress -n gloo-system -o 'jsonpath={.spec.ports[?(@.name=="http")].nodePort}')
```

### Test 

Try out the route using curl:

```shell
curl $GATEWAY_URL/joke

{"category":["dev"],"icon_url":"https://assets.chucknorris.host/img/avatar/chuck-norris.png","id":"ag_6paerrkg-mxfjjqw4ba","url":"https://api.chucknorris.io/jokes/ag_6paerrkg-mxfjjqw4ba","value":"Chuck Norris's beard can type 140 wpm."}
```
