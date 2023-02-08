Docker Buildx
=============

Note: Docker buildx only work with Linux container image builds at this time (Windows support is expected in the near future with BuildKit is updated to support Windows).

Download the latest linux arm64 `docker-buildx` binary release from [GitHub - https://github.com/docker/buildx](https://github.com/docker/buildx/releases/latest).

Rename the file to `docker-buildx` and copy to this repo's local directory.

e.g.

```sh
curl -LO https://github.com/docker/buildx/releases/download/v0.10.2/buildx-v0.10.2.linux-arm64
mv buildx-v0.10.2.linux-arm64 docker-buildx
```

Build the Docker-buildx builder image to run builds from (this could also be connected to a CI server like Azure DevOps or GitHub Actions by installing the corresponding agent script and dependencies):

```sh
# Build the builder image which should have Docker client and Docker-buildx installed
docker build -t dockerbuildx . -f Dockerfile.buildx
docker tag <your-private-reg-fqdn>/dockerbuildx dockerbuildx
docker push <your-private-reg-fqdn>/dockerbuildx
```

or using ACR build tasks:

```sh
ACR_NAME=<your-acr-name>
RG_NAME=<your_acr_rg>

az acr build -r <your-acr-name> -t dockerbuildx -f Dockerfile.buildx .

ACR_LOGIN_SERVER=$ACR_NAME.azurecr.io
# The following command use the admin user on ACR
ACR_USERNAME=$(az acr credential show -n $ACR_NAME -g $RG_NAME --query username -o tsv)
ACR_PASSWORD=$(az acr credential show -n $ACR_NAME -g $RG_NAME --query passwords[0].value -o tsv)
```

Test docker buildx in Kubernetes interactively:

```sh
# Create our interactive pod to test out Docker Buildx
kubectl create ns buildx

# Create a least privileged role that allows the pod to have access to deployments API in group Apps for Kubernetes otherwise you'll get this error:
# --> error: error while calling deploymentClient.Create for "builderxxxx0": deployments.apps is forbidden: User "system:serviceaccount:default:sa-cluster-admin" cannot create resource "deployments" in API group "apps" in the namespace "buildx"

kubectl create clusterrole buildx-runner --verb=get,list,watch,create,delete,patch,update --resource=deployments.apps
kubectl create rolebinding buildx-runner-buildx-ns --clusterrole=deployer --serviceaccount=buildx:default --namespace buildx

kubectl apply -f pod-builder.yaml -n buildx
kubectl get pod -n buildx

# Exec into the demo container
kubectl exec -ti builder -n buildx -- bash

# Inside container - create the buildx instance (3 replicas)
docker buildx create    \
  --driver kubernetes     \
  --driver-opt replicas=3 \
  --use \
  --name builderpool

docker buildx ls
docker buildx inspect builderpool

ls ~/.docker/buildx/

mkdir build
cd build

# Create a test Dockerfile
cat <<EOF>Dockerfile
FROM alpine:latest

LABEL description "Docker Buildx demo"
EOF

# Log into the container registry (requires admin user to be enabled or use azure workload idenity with acrPush role assigned)
docker login $ACR_LOGIN_SERVER -u $ACR_USERNAME -p $ACR_PASSWORD

docker buildx build -t $ACR_LOGIN_SERVER/containerbuild:003 --push .
```

Note: if the docker push doesn't work with azure workload idenity, you can try:

```sh
az login --identity
az acr login -n <acr_name>
```

but you'll need the Azure CLI in the container as well.

Cleanup
-------

```sh
kubectl delete rolebinding buildx-runner-buildx-ns -n buildx
kubectl delete clusterrole buildx-runner

kubectl delete ns buildx
```
