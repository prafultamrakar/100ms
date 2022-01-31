
# DevOps Assignment || 100ms || Praful

The idea of the Assignment is to test the ability to run databases and microservices applications in kubernetes.
## Assignment Requirements
- Create an application that connects to a database, reads some data and returns this data upon HTTP request.
- This can be a simple web app which reads a 'hello world' string from mysql database.
- Run a database app. Data volume should be persistent.
- Application from step 1 needs to discover a database from step 2 using kubernetes native features.
- Database credentials should NOT be hardcoded in application or helm chart code.
- Application should be accessible from the outside of kubernetes.
- Create a helm chart which implements all these steps and deploy via blue/green deployment.

### Pre-requisites
- kubectl: https://kubernetes.io/docs/tasks/tools/#kubectl
- Minikube installed : https://minikube.sigs.k8s.io/docs/start/
- System Memory minimum 8GiB
- Helm v3 : https://helm.sh/docs/intro/install/

### Start Minikube
```bash
minikube start --memory=4g --bootstrapper=kubeadm --extra-config=kubelet.authentication-token-webhook=true --extra-config=kubelet.authorization-mode=Webhook --extra-config=scheduler.bind-address=0.0.0.0 --extra-config=controller-manager.bind-address=0.0.0.0
```


## Nodejs Application

I have used the following references to create a sample CRUD NodeJS application wich communicates to MySQL.

- https://dev.to/tienbku/node-js-crud-operation-with-mysql-example-1gme
- https://www.bezkoder.com/node-js-rest-api-express-mysql/#Source_code

## Installation Steps

Clone the repository
```bash
git clone https://github.com/prafultamrakar/100ms.git 100ms
```

### Create a Hashicorp vault

Ref: https://www.vaultproject.io/docs/platform/k8s/helm/run
```bash
helm install vault hashicorp/vault

helm repo add hashicorp https://helm.releases.hashicorp.com

helm install vault hashicorp/vault

kubectl port-forward vault-0 8200:8200

## Initialize value
kubectl exec -ti vault-0 -- vault operator init

## Unseal the first vault server until it reaches the key threshold
kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 1
kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 2
kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 3
```
Copy the vault “Initial Root Token” as its very imp for authentication and authorizations



### Create a Mysql Statefulsets app using bitnami chart
``` bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm template mysql \
  --set image.tag=5.7,auth.rootPassword=secretpassword,auth.database=nodejs,auth.username=nodejs,auth.password=nodejs-123 \
bitnami/mysql

helm install mysql \
  --set image.tag=5.7,auth.rootPassword=secretpassword,auth.database=nodejs,auth.username=nodejs,auth.password=nodejs-123 \
bitnami/mysql
##The VolumeClaimtemplate is responsible of provisioning or claming dynamic PV 

##Get mysql password 
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)

###Run a pod that you can use as a client:
kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:5.7 --namespace default --command -- bash

mysql -h mysql.default.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"


## NOW Create a Table , using nodejs database
use nodejs;
CREATE TABLE IF NOT EXISTS `tutorials` (
id int(11) NOT NULL PRIMARY KEY AUTO_INCREMENT,
title varchar(255) NOT NULL,
description varchar(255),
published BOOLEAN DEFAULT false
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### Configure Kubernetes authentication
Ref: https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar

Note: Replace [Initial_Root_Token] with the real initial token in previous step.

Please execute these command one-by-one
``` bash
kubectl exec -it vault-0 -- /bin/sh

VAULT_TOKEN="[Initial_Root_Token]" vault auth enable kubernetes

VAULT_TOKEN="[Initial_Root_Token]" vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer="https://kubernetes.default.svc.cluster.local"

VAULT_TOKEN="[Initial_Root_Token]" vault secrets enable kv-v2

VAULT_TOKEN="[Initial_Root_Token]" vault secrets enable -path=dev/ kv

VAULT_TOKEN=”[Initial_Root_Token]” vault kv put dev/db/nodejs username='nodejs' password='nodejs-123
```

For a client to read the secret data defined at dev/db/nodejs
, requires that the read capability be granted for the path dev/data/db/nodejs
```bash
VAULT_TOKEN="[Initial_Root_Token]"  vault policy write internal-app - <<EOF
path "dev/db/nodejs" {
  capabilities = ["read"]
}
EOF
VAULT_TOKEN="[Initial_Root_Token]" vault write auth/kubernetes/role/internal-app \
bound_service_account_names=internal-app \
bound_service_account_namespaces=default \
policies=internal-app \
ttl=24h

VAULT_TOKEN="[Initial_Root_Token]" vault kv get dev/db/nodejs
```

### Create a Service Account you wish to run the application
```bash
kubectl create sa internal-app
```

### Now lets build and push the image to the local repository 
Go to the root folder of the repository and build the nodejs app docker image
```bash
docker login 
docker build -t <your-dockerhub-username>/nodejsapp:test .
docker push <your-dockerhub-username>/nodejsapp:test .
```

### Now lets deploy the using Helm app to check 
We are doing the Canary deployment of nodejs application which consist of two values files.
- stable-value.yaml : for deploying stable release of the application.
- canary-value.yaml : for deploying canary release of the application.

To be more precise, there are two different kind of canary releases.
- A weight-based canary release that routes a certain percentage of the traffic to the new release
- user-based routing where a certain Request Header or value in the Cookies decides which version is being addressed

here we have used cookie based routing to decide canary version of the release.
![Alt text](images/canary-ingress.png?raw=true "Canary Release")
Note: Canary rules are evaluated in order of precedence. Precedence is as follows: canary-by-header -> canary-by-cookie -> canary-weight

Now lets deploy the stable version first
```bash
## go to nodeapp-helm directory
cd nodeapp-helm

### Enable ingress controller in minikube
minikube addons enable ingress

### install the helm chart
helm template stable . -f stable-value.yaml
helm upgrade  --install stable  . -f stable-value.yaml

```
Now lets deploy the Canary version
```bash
helm template canary . -f canary-value.yaml
helm upgrade  --install canary  . -f canary-value.yaml
```

### Testing Canary and Stable traffic 
Expose the Ingress controller
```bash
minikube service ingress-nginx-controller  -n ingress-nginx --url
```

Open another bash terminal to test
Testing Actual traffic -> here 10% traffic goes to Canary and 90% to stable
```bash
while sleep 0.1; do curl "http://127.0.0.1:[port]/api/tutorials" -H "Host: chart-example.local"; done
```
Testing Canary or Test traffic -> Here all traffic goes to Canary for testing purposes
```bash
curl -s --cookie "canary=always" "http://127.0.0.1:[port]/api/tutorials" -H "Host: chart-example.local"
```

    