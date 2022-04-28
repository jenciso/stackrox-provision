# StackRox Provision

## Intro

This is a simple document based on the [Oficial Documentation](https://github.com/stackrox/helm-charts/tree/main/3.69.1/central-services). The ideia is create a local environment with multiple kubernetes clusters provisioned via kind.

![](https://i.imgur.com/B9zXKFA.png)

## Central services (Central)

### Prerequisites:

* A kind cluster deployed the install the stackrox central.
* To have a free node port in your cluster. E.g 32444.
* A Red Hat Account (needed to download the container images).
* A domain name. E.g. stackrox.iplanet.site
* A SSL Certificate files for the domain stackrox.iplanet.site (cert.crt and cert.key).

### Steps:

* Create a `.env` file based on `.env.sample` with the username and password of your Red Hat account.

* Running the helm install:

```shell
export $(cat .env | xargs)

helm repo add stackrox https://charts.stackrox.io
helm repo update

helm install -n stackrox stackrox-central-services rhacs/central-services \
  --create-namespace \
  --set-file central.defaultTLS.cert=./cert.crt \
  --set-file central.defaultTLS.key=./cert.key \
  --set imagePullSecrets.username=$RH_USERNAME \
  --set imagePullSecrets.password=$RH_PASSWORD \
  --set central.exposure.nodePort.enabled=true \
  --set central.exposure.nodePort.port=32444
```

* [Optional] If you want to save this deployment configuration. Save the `generated-values.yaml` file created by this below command.

```shell
kubectl -n stackrox get secret stackrox-generated-vmxhju \
      -o go-template='{{ index .data "generated-values.yaml" }}' | \
      base64 --decode > generated-values.yaml
```

## Secured cluster services (Kubernetes Clusters Clients)

### Prerequisites:

* Create another cluster with Kind.
* Create a token with "admin role" from the central services.
* Download the same version of `roxctl` CLI from Central. 

### Steps:

* To create a token, go to this URL: https://stackrox.iplanet.site:32444/main/integrations/authProviders/apitoken/create

* Generate a token: (Save it as `register.token`)
![](https://i.imgur.com/p9WqHFz.png)

* Download the CLI from the central UI
![](https://imgur.com/9zdzlAx.png)
```shell
export ROX_API_TOKEN="$(cat ./register.token)"
export ROX_CENTRAL_ADDRESS=stackrox.iplanet.site:32444

export CLUSTER_NAME=local-standard
roxctl -e $ROX_CENTRAL_ADDRESS central \
  init-bundles generate cluster-init-$CLUSTER_NAME \
  --output cluster-init-bundle-$CLUSTER_NAME.yaml
```

* Install via helm
 
```shell
helm upgrade -n stackrox \
  stackrox-secured-cluster-services rhacs/secured-cluster-services \
  --create-namespace \
  --set clusterName=$CLUSTER_NAME \
  --set imagePullSecrets.username=$RH_USERNAME \
  --set imagePullSecrets.password=$RH_PASSWORD \
  --set centralEndpoint=$ROX_CENTRAL_ADDRESS \
  --set clusterLabels.env=local \
  --set collector.collectionMethod=NO_COLLECTION \
  -f cluster-init-bundle-$CLUSTER_NAME.yaml
```

* [Optional] Permit the network communication with the central kind network (172.28.1.0/24). 
> It is only if you are using isolated docker network configuration for your kind clusters

```shell
sudo iptables -I FORWARD -s 172.28.1.0/24 -d 0/0 -j ACCEPT
sudo iptables -I FORWARD -s 0/0 -d 172.28.1.0/24 -j ACCEPT
```

