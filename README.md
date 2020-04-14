# Deploying nodejs function to OpenFaas

## Installing OpenFaas to minikube

Create a service account for Helm’s server component (tiller):

```code
kubectl -n kube-system create sa tiller && kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```

Install tiller which is Helm’s server-side component:

```code
helm init --skip-refresh --upgrade --service-account tiller
```

Create namespaces for OpenFaaS core components and OpenFaaS Functions:

```
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```

Add the OpenFaaS helm repository

```
helm repo add openfaas https://openfaas.github.io/faas-netes/
```

Update all the charts for helm

```
helm repo update
```

Generate a random password

```
export PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)
```

Create a secret for the password

```
kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user=admin --from-literal=basic-auth-password="$PASSWORD"
```

Install OpenFaaS using the chart

```
helm upgrade openfaas --install openfaas/openfaas --namespace openfaas --set functionNamespace=openfaas-fn --set basic_auth=true
```

Set the OPENFAAS_URL env-var

```
export OPENFAAS_URL=$(minikube ip):31112
```

Login using the CLI

```
echo -n $PASSWORD | faas-cli login -g http://$OPENFAAS_URL -u admin — password-stdin
```

## Creating node function

Login to docker if using docker hub

```code
docker login
```

Create node function using the faas cli

```
faas-cli new callme --lang node
```

Build, Push Docker image and deploy to openFaas

```
faas-cli build -f callme.yml
faas-cli push -f callme.yml
faas-cli deploy -f callme.yml --gateway $OPENFAAS_URL
```

or

```
faas-cli up  -f callme.yml
```

Invoke function for tests using the cli

```
faas-cli invoke callme --gateway $OPENFAAS_URL
```

Invoke function for tests using curl command

```
curl -XGET http://$OPENFAAS_URL/function/callme
```

# Now let's connect to kafka

Install kafka connector
Specify topics separated by a comma and set the borker host according to your kafka broker endpoint

```
helm repo add openfaas https://openfaas.github.io/faas-netes/

helm upgrade kafka-connector openfaas/kafka-connector \
    --install \
    --namespace openfaas \
    --set topics="mytopic" \
    --set broker_host="192.168.99.101:29092" \
    --set print_response="true" \
    --set print_response_body="true"
```

Modify callme.yml by adding annotation to function

```
 annotations:
      topic: mytopic
```

Visualize logs

```
kubectl logs -n openfaas deploy/kafka-connector -f
```

Send messages to topic in order to validate trigger

```
.\kafka-console-producer.bat --broker-list 192.168.99.101:29092 --topic mytopic
```

# Ressources

https://medium.com/faun/getting-started-with-openfaas-on-minikube-634502c7acdf
https://blog.alexellis.io/quickstart-openfaas-cli/
https://www.openfaas.com/blog/kafka-connector/
